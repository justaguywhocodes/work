# Purple Team Workflow Orchestrator

**Draft POC Requirements & Implementation Plan**  
PowerShell automation for Cymulate, Jira, and VECTR

| **API baseline** | **Runtime** | **Updated** |
|---|---|---|
| Cymulate OpenAPI 2.0.62 | PowerShell 7 + Windows Task Scheduler | August 19, 2026 |

> **In one sentence:** Cymulate supplies execution results and evidence; Jira records workflow and `Blue team detection`; VECTR records the technical test outcome; local state keeps the three systems synchronized.

## Part I — Requirements

### 1. Overview

The POC automates reporting for Cymulate Purple Team assessments that already run on schedule.

| System | Role |
|---|---|
| **Cymulate** | Source of assessment status, scenario results, scenario detection, and findings/evidence. |
| **Jira** | Operational workflow, `Blue team detection`, comments, assignment, and human decisions. |
| **VECTR** | Technical test history, ATT&CK mapping, overall Test Case outcome, per-tool outcomes, and evidence notes. |
| **Local state** | Correlation, idempotency, retries, recovery, and audit. |

The orchestrator is a **finite scheduled batch job**. Windows Task Scheduler starts it, the script performs one discovery and synchronization cycle, persists state, and exits. It is not a resident polling service.

The reporting unit is **one Cymulate scenario result**. If one launched assessment contains ten scenario results, the orchestrator processes all ten and may create ten Jira/VECTR record pairs. There is no local scenario-ID enrollment list.

A run is complete only when the required Jira and VECTR state has been written and read back successfully.

### 2. Scope

#### Goals

- Query launched assessments in a configurable N-hour/N-day lookback window.
- Process every scenario result in each in-scope terminal assessment.
- Use `GET /v2/assessments/launched/{id}/scenarios` for scenario identity, execution status, and documented scenario `detection`.
- Use `GET /v2/assessments/launched/{id}/findings` for supporting AV, EDR, TheHive, prevention, and technical evidence.
- Create exactly one Jira issue and one VECTR Test Case per scenario result.
- Render Jira `Blue team detection` and the VECTR overall outcome from the same normalized evidence.
- Use attributable VECTR per-tool outcomes where supported.
- Survive duplicate discovery, missed schedules, workstation restarts, API outages, and lost create responses without duplicate records.
- Use a dedicated Jira service account and non-interactive credentials.
- Preserve human Jira decisions and keep VECTR tags human-owned.

#### POC Non-Goals

- **Do not mutate Cymulate.** No launch, create, edit, stop, replay, delete, or relaunch operations.
- **Do not build a resident poller.** Each Task Scheduler run performs finite work and exits.
- **Do not use VECTR tags or create VECTR structure dynamically.** Use approved Environment/Assessment/Campaign/library mappings.
- **Do not bypass Cymulate to fill evidence gaps.** Do not directly query AV, EDR, TheHive, Elastic, Tanium, Trellix, or other controls for verdict evidence.
- **Do not fabricate result semantics.** Never infer `NotDetected` from zero findings or an empty `detection` value, guess ambiguous correlation, invent timestamps, or use alternate Cymulate findings APIs to force a verdict.
- **Do not overwrite human-authoritative decisions or copy unrestricted payloads.**

### 3. Architecture

#### 3.1 Components

| Component | Responsibility |
|---|---|
| `Invoke-PurpleTeamOrchestrator.ps1` | Scheduled entry point; discovery, processing, reconciliation, exit. |
| `PurpleTeam.Cymulate.psm1` | Read-only Cymulate API client. |
| `PurpleTeam.Jira.psm1` | Jira create/update/read-back/transition operations. |
| `PurpleTeam.Vectr.psm1` | VECTR GraphQL create/update/outcome operations. |
| `PurpleTeam.State.psm1` | Durable state, idempotency, retries, audit, checkpoint. |
| Local state store | SQLite preferred; approved atomic per-run JSON fallback if needed. |

#### 3.2 Scheduled Run

Each invocation:

1. Acquires a named mutex.
2. Loads configuration and secrets.
3. Reads the last successful discovery checkpoint.
4. Queries launched assessments in the configured lookback window.
5. Records non-terminal assessments for later review; it does not create final result records for them.
6. Fully retrieves scenario results and findings for reportable terminal assessments.
7. Processes each scenario result once.
8. Creates/updates and verifies Jira.
9. Creates/updates and verifies VECTR.
10. Reconciles incomplete prior runs.
11. Writes audit/log state and advances the checkpoint only after safe persistence.
12. Releases the mutex and exits.

Task Scheduler must prevent overlapping runs and run non-interactively. The script must also use its own mutex so duplicate task starts cannot modify the same run concurrently.

#### 3.3 Correlation and Idempotency

**External idempotency key:**

`<assessmentId>:<scenarioResultID>`

**Internal run ID:**

`UUID v4`

The Jira issue key is generated by Jira and must never be predicted locally.

Creation order:

1. Persist the ScenarioRun and `run_id`.
2. Create or recover Jira using `run_id` and Cymulate identifiers.
3. Persist the Jira-generated `CIBETHICAL-####` key.
4. Create or recover the VECTR Test Case.
5. Persist the VECTR Test Case ID.
6. Read back both systems before marking the run complete.

If a create response is lost, recover the existing object before retrying creation. Jira recovery uses `run_id`; VECTR recovery uses deterministic Test Case identity plus Campaign/library verification.

### 4. Integration Requirements

#### 4.1 Cymulate

**Approved reporting endpoints**

- `GET /v2/assessments/launched`
- `GET /v2/assessments/launched/{id}`
- `GET /v2/assessments/launched/{id}/scenarios`
- `GET /v2/assessments/launched/{id}/findings`
- `GET /v2/scenarios/{scenarioID}` when metadata is required

No alternate Cymulate findings/search/detail API is part of the reporting path.

| Requirement | Rule |
|---|---|
| **Authentication** | Support the approved `x-token` path. OAuth2 may be enabled only after contract testing. |
| **Discovery** | Query terminal result-bearing assessments and retain relevant failed/stopped cases for exception handling. |
| **Non-terminal states** | Persist pending state and revisit on a later scheduled run. Do not create final Jira/VECTR result records while integrations are still processing. |
| **Scenario identity** | `/scenarios` is authoritative for `scenarioResultID` and `scenarioID`. |
| **Scenario detection** | Preserve raw `detection`: documented values are `Detected`, `Not Detected`, and empty/not evaluated. Empty is never treated as `Not Detected`. |
| **Scenario execution** | Normalize `status` and `statusReason` using values validated against actual Cymulate responses. |
| **Findings** | Fully page `/findings`; use findings as supporting evidence, not scenario identity. |
| **Finding correlation** | Do not join by `scenarioName` alone unless actual Cymulate responses prove it safe for the relevant assessment model. |
| **Zero findings** | Evidence-neutral. Zero findings do not create a detection conclusion. |
| **Conflicts/unknowns** | Preserve the raw values and route unresolved terminal cases to manual review. |
| **Pagination/rate limit** | Implement safe paging, stop repeated cursors, honor `Retry-After`, and throttle below 30 requests/10 seconds/IP. |
| **Mutations** | No Cymulate POST/PATCH/PUT/DELETE calls in the POC reporting path. |

#### 4.2 Evidence and Jira Verdict

The decision engine keeps these evidence dimensions separate:

| Dimension | Normalized values |
|---|---|
| `assessment_lifecycle` | `NOT_STARTED`, `RUNNING`, `QUERYING_INTEGRATION`, `TERMINAL`, `UNKNOWN` |
| `scenario_execution` | `NOT_STARTED`, `RUNNING`, `SUCCESS`, `PARTIAL`, `FAILED`, `UNKNOWN` |
| `prevention_outcome` | `BLOCKED`, `NOT_BLOCKED`, `NOT_APPLICABLE`, `UNKNOWN` |
| `scenario_detection` | `DETECTED`, `NOT_DETECTED`, `NOT_EVALUATED`, `UNKNOWN` |
| `av_outcome` | `BLOCKED`, `DETECTED`, `NOT_DETECTED`, `NOT_APPLICABLE`, `UNKNOWN`, `NOT_AVAILABLE` |
| `edr_outcome` | `BLOCKED`, `DETECTED`, `NOT_DETECTED`, `NOT_APPLICABLE`, `UNKNOWN`, `NOT_AVAILABLE` |
| `thehive_use_case_outcome` | `FIRED`, `NOT_FIRED`, `NOT_APPLICABLE`, `UNKNOWN`, `NOT_AVAILABLE` |
| `evidence_consistency` | `CONSISTENT`, `PARTIAL`, `CONFLICT`, `UNKNOWN` |

AV/EDR/TheHive values may only come from `/findings` fields whose meaning has been validated against actual responses from the organization's Cymulate environment.

##### Jira `Blue team detection`

Allowed values are exactly:

`NotApplicable` · `NotDetected` · `Detected` · `NotStarted` · `Blocked` · `AttackFailed` · `PartialTesting` · `TestingInProgress`

**Decision precedence**

| Priority | Condition | Jira value |
|---|---|---|
| 1 | Terminal scenario did not start / was not tested | `NotStarted` |
| 2 | Execution failed | `AttackFailed` |
| 3 | Execution was partial | `PartialTesting` |
| 4 | AV or EDR proves prevention/block | `Blocked` |
| 5 | Scenario ran, was not blocked, and expected TheHive use case fired | `Detected` |
| 6 | Scenario ran, was not blocked, expected TheHive use case did not fire, and required evidence is complete and consistent | `NotDetected` |
| 7 | Detection is genuinely not applicable by approved policy | `NotApplicable` |

`TestingInProgress` is reserved for a separately approved pre-provisioned/active-ticket workflow. The normal terminal-result processor does not create tickets in that state.

##### Manual Review

If required evidence is missing, contradictory, ambiguous, uncorrelatable, or uses an unknown schema value:

- set internal `processing_status = MANUAL_REVIEW_REQUIRED`;
- leave `Blue team detection` unset;
- route Jira to `Analyze`;
- record the reason;
- re-evaluate on later scheduled runs;
- do not overwrite a later human-authoritative value.

##### Semantic Rules

- `AttackFailed` is not `NotDetected`.
- `NotDetected` requires a successful, non-blocked run plus complete, consistent evidence that the expected use case did not fire.
- `NotApplicable` is policy-driven, not a fallback for missing integrations.
- Prevention takes precedence over detection: if AV/EDR blocked the activity, Jira records `Blocked` while detection evidence is preserved.
- Manual review is workflow state, not another `Blue team detection` value.

##### Jira Workflow

| Result / routing condition | Target column |
|---|---|
| `NotStarted` | `Schedule` |
| `TestingInProgress` | `In Progress` |
| `MANUAL_REVIEW_REQUIRED` | `Analyze` |
| `AttackFailed`, `PartialTesting` | `Analyze` |
| `Detected`, `NotDetected`, `Blocked`, `NotApplicable` | `Validation` |
| Manual exception | `On Hold` |
| Validated result | `Done` |

Jira Automation is preferred for normal field-driven transitions. Verdict-less manual-review cases may be transitioned directly to `Analyze` by the orchestrator.

Any automation-owned Jira write must be read back. Human changes to shared fields must not be blindly overwritten.

#### 4.3 Jira Service Account and Identity

- Project: `CIBETHICAL`
- Machine writes use a dedicated non-person Jira account.
- Human comments, analysis, and overrides remain attributable to individual operators.
- Jira custom-field IDs, option IDs, and transition IDs are discovered/configured; they are not hard-coded.
- Suggested summary: `[Purple Team] <Scenario Name> - <YYYY-MM-DD>`
- The summary is not an idempotency key.

Recommended correlation fields include `Purple Run ID`, Cymulate assessment/scenario identifiers, VECTR Test Case ID, stakeholder, run type, and target.

#### 4.4 VECTR

**Endpoint:** `POST https://<host>/sra-purpletools-rest/graphql`  
**Authentication:** `Authorization: VEC1 <key_id>:<key_secret>`

| Requirement | Rule |
|---|---|
| **Environment** | Use the configured VECTR environment as GraphQL `db`. |
| **Assessment/Campaign** | Resolve to approved existing IDs. The POC does not create them dynamically. |
| **Library mapping** | Prefer `libraryTestCaseId`; do not build new logic around deprecated `templateId`. |
| **Creation** | Prefer `testCase.createWithTemplateMatchByLibraryId` with `createNewIfNotExists=false`. |
| **Correlation** | Set `clientId = run_id`; create only after `jira_key` exists. |
| **Name** | `[CIBETHICAL-####] <Scenario Name> - <YYYY-MM-DD>` |
| **Lost response** | Recover by deterministic identity and verify Campaign + library mapping before reuse. |
| **GraphQL errors** | Treat non-empty top-level `errors` as failure even when HTTP status is 200. |
| **Outcome catalog** | Resolve selectable paths/IDs from `outcomes` / `outcomesTree`; do not hard-code database-specific IDs. |
| **Overall outcome** | Every created Test Case receives an explicit approved `outcomePath` derived from the same evidence used for Jira. |
| **Per-tool outcomes** | For `dataVer=2`, write attributable AV/EDR/TheHive Defense Tool outcomes where supported. |
| **Override** | If VECTR's per-tool priority would produce an overall outcome that contradicts the approved verdict, use the approved `overrideOutcome` behavior. |
| **Evidence notes** | Write compact evidence/outcome notes only. |
| **Timestamps** | Use validated source timestamps only; never use orchestration time as attack time. |
| **Tags** | Perform zero tag reads/writes for workflow or verdict logic. |

##### VECTR Outcome Mapping

Exact paths are environment-specific and must be resolved during Phase 0.

| Jira result / state | VECTR overall outcome intent |
|---|---|
| `Blocked` | Approved `Blocked` path |
| `Detected` | Approved successful `Alerted` path |
| `NotDetected` | Approved unsuccessful outcome reflecting actual evidence (`None`, `Logged`, or other approved path) |
| `NotApplicable` | Approved `N/A` path |
| `NotStarted` | `TBD` |
| `AttackFailed` | `TBD` |
| `PartialTesting` | `TBD` |
| `TestingInProgress` | `TBD` |
| `MANUAL_REVIEW_REQUIRED` | `TBD`, updated after resolution |

The VECTR outcome is not a string copy of the Jira value. It expresses the same evidence using VECTR's native outcome model.

For `NotDetected`, preserve any local AV/EDR visibility in per-tool outcomes even when the overall result remains an unsuccessful end-to-end detection result.

Every create/update must be read back and verified. A deterministic run cannot complete with a missing or mismatched VECTR overall outcome.

#### 4.5 State, Retry, Reconciliation, and Security

| Requirement | Rule |
|---|---|
| **Durability** | Persist state across script exit and workstation reboot. |
| **Uniqueness** | Unique constraint on `<assessmentId>:<scenarioResultID>`. |
| **Retries** | Retry transient transport/API failures with bounded backoff and jitter. |
| **No blind retries** | Do not blindly retry auth, permission, schema, invalid-transition, or ambiguous create states. |
| **Read-back** | Verify critical Jira/VECTR writes. |
| **Reconciliation** | Revisit incomplete runs on every scheduled invocation. |
| **Human conflict** | Do not overwrite deliberate human changes to shared fields. |
| **Checkpoint** | Advance only after the discovery cycle is safely persisted. |
| **Kill switch** | `AUTOMATION_WRITE_ENABLED=false` computes desired state but makes no Jira/VECTR writes. |
| **Secrets** | No tokens in source/config committed to Git or normal logs. Use approved non-interactive secret storage. |
| **Least privilege** | Jira automation account is non-personal; VECTR key is least privilege; Cymulate should be read-only where permissions allow. |

### 5. Core Data Model

#### 5.1 Optional Scenario Overrides

Scenario-specific configuration is optional. It may override default or assessment/template-level policy, but it never controls whether a discovered scenario is processed.

Possible overrides:

- stakeholder;
- detection applicability;
- expected integrations;
- expected TheHive use case;
- required verdict evidence;
- Jira issue type;
- VECTR environment/assessment/campaign/library mapping.

If no deterministic downstream mapping can be resolved, keep the scenario as a tracked run and raise a configuration/synchronization error; do not ignore it or guess.

#### 5.2 ScenarioRun

`ScenarioRun` is an orchestrator-owned persistence record, not a Cymulate/Jira/VECTR API object.

It must retain at minimum:

| Category | Required data |
|---|---|
| **Identity** | `run_id`, external idempotency key, assessment ID, scenario result ID, scenario ID |
| **Raw Cymulate state** | assessment status/integrations, scenario status/statusReason/detection |
| **Normalized evidence** | lifecycle, execution, prevention, scenario detection, AV, EDR, TheHive, consistency |
| **Evidence references** | finding IDs, retrieval/correlation status |
| **Jira** | candidate/final `Blue team detection`, rule, manual-review reason, Jira key, sync state |
| **VECTR** | Test Case ID, overall outcome path/ID, override flag, per-tool outcomes, mapping rule, sync state |
| **Operations** | processing state, retry count, last error, created/updated/completed timestamps |

Processing states:

`DISCOVERED` · `WAITING_FOR_RESULT` · `WAITING_FOR_INTEGRATION` · `CORRELATED` · `EVIDENCE_PENDING` · `PROVISIONING` · `REPORTING` · `MANUAL_REVIEW_REQUIRED` · `SYNC_PENDING` · `COMPLETE` · `FAILED`

Pending states are persisted and revisited on the next scheduled run; the script never waits in a resident loop.

### 6. POC Acceptance Tests

| Test | Expected result |
|---|---|
| Non-terminal assessment | Persist pending state; create no final Jira/VECTR pair. |
| Assessment with N scenario results | Create N ScenarioRuns; no scenario allowlist. |
| Confirmed detection | Jira `Detected`; approved VECTR successful outcome. |
| Confirmed miss | Jira `NotDetected`; approved unsuccessful VECTR outcome preserving local telemetry. |
| AV/EDR prevention | Jira `Blocked`; VECTR blocked outcome and attributable tool results. |
| Execution failure | Jira `AttackFailed`; VECTR `TBD`. |
| Partial execution | Jira `PartialTesting`; VECTR `TBD`. |
| Detection genuinely N/A | Jira `NotApplicable`; VECTR `N/A`. |
| Missing/conflicting/ambiguous evidence | Jira field unset; `MANUAL_REVIEW_REQUIRED`; Jira `Analyze`; VECTR `TBD`. |
| Zero findings | No automatic negative conclusion. |
| Unknown Cymulate value | Preserve raw value; route to manual review. |
| Lost Jira create response | Recover existing issue; no duplicate. |
| Lost VECTR create response | Recover existing Test Case; no duplicate. |
| Human Jira override | Do not blindly overwrite. |
| Read-back mismatch | Keep run open for retry/reconciliation. |
| Workstation missed schedule | Catch up from checkpoint/lookback without duplicates. |
| VECTR tag guard | Zero automation tag reads/writes. |
| Cymulate mutation guard | Zero normal-path mutation calls. |

### 7. Key Risks

| Risk | Mitigation |
|---|---|
| Duplicate Jira/VECTR records | Persist correlation before writes; recover lost responses before retrying create. |
| Empty/unknown evidence becomes a false verdict | Preserve raw values; use manual review rather than inference. |
| Findings cannot be tied safely to a scenario | Use `/scenarios` for identity; ambiguous evidence goes to manual review. |
| Integration results settle late | Persist pending/evidence-pending state and revisit later. |
| Wrong Jira/VECTR mapping | Validate configuration and read back written state. |
| Human decisions are overwritten | Field ownership and conflict protection. |
| Workstation/API outage | Durable state, checkpoint overlap, retries, reconciliation. |
| Secrets exposed | Non-interactive secret store and log redaction. |
| Cymulate schema changes | Contract fixtures, preserve unknowns, fail safe to manual review. |
| VECTR per-tool outcome conflicts with overall verdict | Apply approved overall override and verify stored outcome. |

### 8. References

- Supplied Cymulate OpenAPI Swagger (`swagger.json`), OpenAPI 3.1.1, Cymulate API version 2.0.62.
- Supplied Cymulate API Postman collection, secondary where it does not conflict with Swagger.
- Sanitized actual Cymulate responses captured during Phase 0.
- Official VECTR outcome and GraphQL Test Case documentation.
- Actual VECTR schema/outcome tree from the target environment.
- Jira REST documentation and actual `CIBETHICAL` field/workflow metadata.
- Existing Purple Team Jira and VECTR operating practices.

## Part II — Implementation Plan

### 9. Codex Working Rules

Use this document with one task ID at a time. Implement the task, run Pester/tests, verify the acceptance criteria, and commit before starting a dependent task.

Implementation constraints:

- PowerShell 7 only.
- Pester tests for behavior changes.
- No Cymulate mutation calls.
- No VECTR tag reads/writes.
- No personal Jira credentials.
- No hard-coded Jira custom-field/transition IDs or VECTR outcome IDs unless explicitly approved in Phase 0.
- Never infer `NotDetected` from absence or empty detection.
- Never predict Jira issue numbers locally.
- Persist `run_id` before Jira create and `jira_key` before VECTR create.
- No `COMPLETE` state until Jira/VECTR automation-owned state is verified.

### Phase 0: Contract Discovery & Scaffolding

**Goal:** Prove the actual Cymulate/Jira/VECTR contracts before write automation.

| ID | Task | Acceptance Criteria |
|---|---|---|
| P0-T1 | Initialize repository, modules, scripts, tests, sample config. | Modules import; config parses; no secrets committed. |
| P0-T2 | Build configuration loader for auth, discovery window, mappings, overrides, state, and kill switch. | Invalid required config fails before API calls; no scenario allowlist. |
| P0-T3 | Capture launched-assessment/scenario/findings contracts, paging, statuses, rate limits, and endpoint allowlist. | Exact request/response contract documented with fixtures. |
| P0-T4 | Prove scenario status/detection normalization and integration-settling behavior. | `Detected`, `Not Detected`, empty, unknown, and status cases are fixture-backed. |
| P0-T5 | Prove AV/EDR/TheHive evidence fields, correlation, zero/multiple findings, and verdict matrix. | Missing/conflicting evidence maps to manual review, never fabricated verdict. |
| P0-T6 | Capture Jira service account, fields/options, create/update behavior, transitions, Jira Automation, and human ownership. | All eight active values round-trip; `Analyze` route validated. |
| P0-T7 | Capture VECTR mappings/outcome tree, Test Case create/update, per-tool outcomes, override behavior, and zero-tag rule. | Representative records receive correct verified overall/per-tool outcomes. |

**Exit gate:** No unresolved API, verdict, correlation, or mapping assumption for the POC pilot.

### Phase 1: Durable State & Idempotency

| ID | Task | Acceptance Criteria |
|---|---|---|
| P1-T1 | Implement local schema for checkpoint, ScenarioRun, retries, audit. | State survives restart. |
| P1-T2 | Enforce unique `<assessmentId>:<scenarioResultID>`. | 100 duplicate attempts create one run. |
| P1-T3 | Implement processing states/transitions. | Illegal transitions fail; states are tested. |
| P1-T4 | Implement structured audit/logging with secret redaction. | State-changing tests emit audit records; secrets never appear. |
| P1-T5 | Implement named mutex/abandoned-lock recovery. | Concurrent runs cannot mutate state together. |

**Exit gate:** Restart, duplicate, and concurrency tests pass without external writes.

### Phase 2: Jira Adapter

| ID | Task | Acceptance Criteria |
|---|---|---|
| P2-T1 | Implement service-account auth/connectivity. | Approved automation identity is used; 401/403 classified. |
| P2-T2 | Validate field metadata and exact `Blue team detection` values. | Invalid values rejected locally. |
| P2-T3 | Create issue from persisted correlation and store Jira-generated key. | `jira_key` is null before POST and stored after success. |
| P2-T4 | Recover lost Jira create response by `run_id`. | 100 lost-response trials create zero duplicates. |
| P2-T5 | Implement automated comments/attribution. | Comment appears under service account. |
| P2-T6 | Implement `Blue team detection` write/read-back. | All eight values round-trip; unset only for manual review. |
| P2-T7 | Discover transitions by name and apply approved assignment behavior. | No hard-coded transition IDs. |
| P2-T8 | Protect human changes to shared fields. | Human-modified value is not blindly overwritten. |

**Exit gate:** Zero duplicate issues and zero personal-account automation writes.

### Phase 3: VECTR Adapter

| ID | Task | Acceptance Criteria |
|---|---|---|
| P3-T1 | Implement GraphQL client, VEC1 auth, variables, top-level error handling. | HTTP 200 + GraphQL error is treated as failure. |
| P3-T2 | Implement cursor paging and Environment/Assessment/Campaign validation. | Complete paging; invalid mapping fails validation. |
| P3-T3 | Resolve Content Library Test Case by `libraryTestCaseId`. | Approved template resolves exactly. |
| P3-T4 | Discover/validate VECTR outcome catalog. | Semantic targets resolve to one approved selectable path/ID. |
| P3-T5 | Create Test Case after `jira_key` exists with deterministic name, `clientId`, overall outcome, notes, and per-tool outcomes. | One mapped Test Case is created with verified non-null outcome. |
| P3-T6 | Recover lost create response safely. | 100 lost-response trials create zero duplicates. |
| P3-T7 | Read back and verify overall outcome, override flag, `dataVer`, and notes. | Stored state matches desired state. |
| P3-T8 | Implement attributable Defense Tool outcomes. | Only supported/attributable tools receive outcomes. |
| P3-T9 | Implement overall-outcome override when per-tool priority conflicts. | Correct overall result is stored and verified. |
| P3-T10 | Implement `TBD` → final outcome update and approved historical timestamp strategy. | Outcome update and timestamps read back correctly. |
| P3-T11 | Add negative tests for tags and deprecated outcome fields. | Zero tag mutation; deprecated create-time `outcome` not used as primary writer. |

**Exit gate:** Correct VECTR records/outcomes, zero duplicates, zero tag operations.

### Phase 4: Cymulate Read Adapter

| ID | Task | Acceptance Criteria |
|---|---|---|
| P4-T1 | Implement `x-token`, optional approved OAuth2, throttling, transient retries. | Rate/retry behavior is tested. |
| P4-T2 | Implement one-shot launched-assessment discovery with checkpoint/lookback and lifecycle normalization. | Non-terminal cases persist; no polling loop. |
| P4-T3 | Implement assessment paging/detail retrieval. | All pages and required raw fields retained. |
| P4-T4 | Implement scenario retrieval and detection normalizer. | `Detected`, `Not Detected`, empty, unknown map correctly. |
| P4-T5 | Implement validated scenario status/statusReason normalizer. | Execution states behave as approved. |
| P4-T6 | Implement launched findings retrieval/paging once per parent/cycle. | All pages retrieved; zero findings neutral. |
| P4-T7 | Implement finding correlation and prevention/control evidence extraction. | Ambiguity stays explicit; no unsafe name-only join. |
| P4-T8 | Implement AV/EDR/TheHive extraction from validated `/findings` fields only. | Missing/unproven fields stay unknown/not available. |
| P4-T9 | Implement evidence-consistency resolver and metadata cache. | Material conflict becomes `EVIDENCE_CONFLICT`. |
| P4-T10 | Guard against Cymulate mutations and alternate/non-allowlisted reporting endpoints. | Prohibited calls fail before HTTP. |

**Exit gate:** Pilot assessment normalizes correctly using only approved read endpoints.

### Phase 5: End-to-End Reporting

| ID | Task | Acceptance Criteria |
|---|---|---|
| P5-T1 | Create one ScenarioRun for every scenario result in an in-scope terminal assessment; apply defaults and optional overrides. | No scenario is skipped because no override exists. |
| P5-T2 | Implement create sequence: persist run → Jira create/recover → persist key → derive VECTR target → VECTR create/recover → persist VECTR ID. | No premature records; no duplicate lost-response creates. |
| P5-T3 | Implement terminal verdict precedence. | All approved outcomes and manual-review cases are tested. |
| P5-T4 | Render VECTR overall/per-tool outcomes and notes from the same evidence. | Outcome matches Jira semantics; unresolved cases use `TBD`. |
| P5-T5 | Implement Jira routing/Jira Automation expectations. | Manual/failure cases go `Analyze`; substantive results normally go `Validation`. |
| P5-T6 | Implement completion barrier and read-back verification. | No run completes with pending integration or mismatched Jira/VECTR state. |
| P5-T7 | Test multi-scenario assessments and repeated templates/test points. | N results produce N runs and, when mappings resolve, N Jira/VECTR pairs. |

**Exit gate:** Deterministic results synchronize automatically; unresolved cases route safely to `Analyze`.

### Phase 6: Reconciliation & Resilience

| ID | Task | Acceptance Criteria |
|---|---|---|
| P6-T1 | Reconcile incomplete runs on each invocation. | Stale automation-owned state is detected. |
| P6-T2 | Implement safe repair and human-conflict classification. | Safe drift repaired; human changes protected. |
| P6-T3 | Implement missed-schedule catch-up/checkpoint rules. | Missed terminal results are processed once. |
| P6-T4 | Implement operational health metrics. | Runs, failures, retries, and duplicate suppression are visible. |
| P6-T5 | Implement operator repair/test-config commands. | Failed synthetic run can be inspected/repaired without editing state files. |
| P6-T6 | Implement Task Scheduler installer/settings. | Fresh workstation can register a non-overlapping, non-interactive task. |

**Exit gate:** Known outages/mismatches are visible and recoverable; no run silently disappears or falsely completes.

### Phase 7: Shadow Mode & Controlled POC Pilot

| ID | Task | Acceptance Criteria |
|---|---|---|
| P7-T1 | Implement `AUTOMATION_WRITE_ENABLED=false` shadow mode. | Desired changes are calculated; Jira/VECTR receive no writes. |
| P7-T2 | Shadow representative DLP result. | Desired Jira/VECTR state matches human reference. |
| P7-T3 | Shadow representative CSIRT result. | Detection/integration/evidence behavior matches human reference. |
| P7-T4 | Shadow representative Use Case Generation result. | Defense evidence mapping matches human reference. |
| P7-T5 | Run end-to-end failure and outcome matrix. | Lost responses, API errors, pending integrations, verdict cases, evidence conflicts, VECTR overrides, restart, and human override pass. |
| P7-T6 | Enable writes for a narrow assessment/template/tag/name/environment scope. | ≥20 successful runs; zero skipped scenarios, duplicate Jira/VECTR, or VECTR tag mutations. |
| P7-T7 | POC acceptance review. | Reliability, rollback, credential rotation, and operator runbook reviewed; go/no-go recorded. |

**Exit gate:** Controlled pilot has no unexplained discrepancy, duplicate, lost execution, incorrect Jira/VECTR outcome, or tag mutation.

### Phase 8: Post-POC Rollout (Future)

If the POC is accepted:

1. Validate production discovery scope and lookback.
2. Expand assessment-level scope in controlled batches.
3. Add scenario overrides only when default mappings are insufficient.
4. Run shadow/catch-up validation before material expansions.
5. Maintain a lightweight monthly discovery/sync/configuration check.

This phase is outside POC acceptance.

### Implementation Baseline

- PowerShell 7+
- Windows 10/11 enterprise workstation
- Windows Task Scheduler
- `Invoke-RestMethod` / `Invoke-WebRequest`
- Pester
- `.psd1` + JSON configuration
- SQLite preferred local state
- JSONL + human-readable logs
- Git source control

Suggested repository:

| Path | Purpose |
|---|---|
| `scripts/Invoke-PurpleTeamOrchestrator.ps1` | Scheduled entry point |
| `modules/PurpleTeam.Common.psm1` | Config, HTTP, logging, validation |
| `modules/PurpleTeam.Cymulate.psm1` | Cymulate read adapter |
| `modules/PurpleTeam.Jira.psm1` | Jira adapter |
| `modules/PurpleTeam.Vectr.psm1` | VECTR adapter |
| `modules/PurpleTeam.State.psm1` | State/idempotency |
| `config/` | Discovery scope, mappings, optional overrides |
| `state/` | Durable runtime state |
| `logs/` | Operational logs |
| `tests/` | Unit/integration/end-to-end tests |
| `docs/` | Contracts, mappings, runbooks, UAT evidence |

### Reliability Targets

- Duplicate Jira issues: **0**
- Duplicate VECTR Test Cases: **0**
- Completed runs with unresolved Jira/VECTR mismatch: **0**
- Automation-created VECTR tags: **0**
- Jira writes using personal accounts: **0**
- `NotDetected` values inferred from empty data: **0**
- Unsupported Cymulate reporting calls: **0**
- Cymulate mutation calls from the POC reporting path: **0**
- Provisioned VECTR Test Cases without an overall outcome: **0**
- Resident Cymulate polling loops: **0**

### Future Roadmap

- Server-hosted orchestration instead of workstation Task Scheduler.
- Event-driven Cymulate webhook ingestion if officially supported and adopted.
- Centralized operations/dashboarding.
- UI for scope/override management.
- Additional stakeholder reporting.
- Cymulate launch/schedule management only under a separately approved write-capable design.

## Addendum A — Architecture Diagrams

This addendum is explanatory. The requirements above are authoritative.

### A.1 End-to-End Architecture

```text
Windows Task Scheduler
        |
        v
PowerShell Orchestrator
        |
        | read-only
        v
     Cymulate
        |
        +-- launched assessments
        +-- scenario results
        +-- findings / AV / EDR / TheHive evidence
        |
        v
Normalize evidence + render result
        |
        +------------------+
        |                  |
        v                  v
      Jira               VECTR
 workflow/status      technical history
 Blue team detection  overall outcome
 comments             per-tool outcomes
        |                  |
        +---------+--------+
                  v
           Local durable state
      idempotency / retry / audit
```

### A.2 Scheduled Discovery

```text
last successful checkpoint
        |
        +-- apply configured overlap
        v
GET /v2/assessments/launched
        |
        +-- non-terminal --> persist pending --> next scheduled run
        |
        +-- terminal ------> enumerate all scenario results
                                  |
                                  v
                             process each once
                                  |
                                  v
                           persist + advance checkpoint
```

The overlap is intentional; duplicate safety comes from the idempotency key.

### A.3 Reporting Cardinality

```text
1 launched assessment
        |
        +-- scenario 1 --> Jira 1 + VECTR 1
        +-- scenario 2 --> Jira 2 + VECTR 2
        +-- scenario 3 --> Jira 3 + VECTR 3
        +-- ...
```

One Cymulate scenario result is one reporting unit.

### A.4 Evidence to Verdict

```text
scenario result
   |
   +-- execution
   +-- scenario detection
   +-- AV
   +-- EDR
   +-- TheHive use case
   |
   v
normalize evidence
   |
   v
apply precedence
   |
   +-- failed -----------------> AttackFailed
   +-- partial ----------------> PartialTesting
   +-- AV/EDR blocked ---------> Blocked
   +-- use case fired ---------> Detected
   +-- confirmed use-case miss -> NotDetected
   +-- detection N/A ----------> NotApplicable
   +-- unresolved/conflicting -> MANUAL_REVIEW_REQUIRED
```

The same normalized evidence drives both Jira and VECTR.

### A.5 Jira Result vs. Workflow

```text
Can evidence support a substantive verdict?
            /            \
          yes             no
           |               |
           v               v
set Blue team detection   leave field unset
           |               |
           v               v
verify field          MANUAL_REVIEW_REQUIRED
           |               |
           v               v
normal routing           Analyze
```

Manual review is workflow state, not a detection verdict.

### A.6 VECTR Organization

```text
VECTR Environment / GraphQL db
└-- Assessment: Cymulate Continuous Validation - 2026 Q3
    ├-- Campaign: Endpoint / EDR
    │   ├-- [CIBETHICAL-1842] Scenario A
    │   └-- [CIBETHICAL-1843] Scenario B
    └-- Campaign: Detection Engineering
        └-- [CIBETHICAL-1844] Scenario C
```

Cymulate-originated Test Cases normally stay in the VECTR Environment representing the defensive environment being tested. Use a separate Environment only for a real isolation boundary such as infrastructure, Defense Tools, access, governance, or reporting ownership.

The POC uses existing approved Assessments and Campaigns; it does not create them dynamically.

### A.7 VECTR Outcomes

```text
one scenario result
        |
        v
one normalized evidence set
        |
        v
one VECTR Test Case
        |
        +-- overall outcome
        +-- AV per-tool outcome       (if attributable)
        +-- EDR per-tool outcome      (if attributable)
        +-- TheHive per-tool outcome  (if attributable)
        +-- compact outcomeNotes
```

Cymulate findings support the Test Case; they do not become separate VECTR Test Cases.

### A.8 Correlation, Recovery, and Completion

```text
assessmentId + scenarioResultID
        |
        v
external_key + local run_id
        |
        v
create/recover Jira
        |
        v
Jira returns CIBETHICAL-####
        |
        v
create/recover VECTR
        |
        v
read back Jira + VECTR
        |
        +-- match --> COMPLETE
        |
        +-- mismatch/error --> SYNC_PENDING --> next scheduled run
```

Lost create responses are recovered before retrying creation. Cymulate completion alone does not mark the orchestration run complete.
