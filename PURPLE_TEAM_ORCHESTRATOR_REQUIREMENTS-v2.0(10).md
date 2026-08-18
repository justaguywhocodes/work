# Purple Team Workflow Orchestrator — Requirements Document v2.0

**PowerShell Automation for Cymulate, Jira, and VECTR**

Version 2.0 — Cymulate OpenAPI 2.0.62 Detection Contract + Implementation Phases | August 18, 2026

> Workstation-hosted orchestration for standing Purple Team assessments. Cymulate owns execution and raw detection results; Jira owns workflow; VECTR owns technical test history.

> **Swagger-reviewed revision.** This revision supersedes the findings-first detection design in v1.9. The supplied OpenAPI 3.1.1 document, Cymulate API version 2.0.62, explicitly defines a per-scenario `detection` result and a parent assessment integration-query lifecycle. The specification now treats the launched-assessment scenario result `detection` field as the primary documented scenario-level detection source, while `GET /v2/assessments/launched/{id}/findings` remains the required supporting-evidence source for prevention/control outcomes, AV evidence, EDR evidence, TheHive use-case evidence, consistency checking, and VECTR reporting. The reporting framework is intentionally limited to the approved launched-assessment/scenario GET endpoints and scenario metadata.

**Terminology note:** In this specification, **scenario result `detection` field** means the `detection` property on each item returned by `GET /v2/assessments/launched/{id}/scenarios`, i.e. `data[].detection`. This is a response field on the launched-assessment scenarios response, not a separate endpoint or standalone JSON path.
> **Reporting-endpoint scope revised.** The orchestrator uses the launched-assessment/scenario GET APIs for reporting evidence. Alternate Cymulate findings/search/detail APIs are not part of the product; unresolved or contradictory AV/EDR/TheHive evidence enters `MANUAL_REVIEW_REQUIRED` and routes Jira to `Analyze` rather than triggering another Cymulate evidence query or fabricating a `Blue team detection` verdict.
> **Verdict model revised.** The active Jira result model is vendor-agnostic for prevention: any proven AV/EDR prevention maps to `Blocked`. AV, EDR, and TheHive use-case evidence are retained as separate source dimensions and combined to render the final `Blue team detection` verdict.
> **VECTR outcome assignment revised.** The same normalized AV/EDR/TheHive evidence that renders the Jira verdict must also render the VECTR Test Case's native overall defense outcome. New Test Cases must not be created without an explicit VECTR outcome. In per-tool outcome mode, attributable per-tool outcomes are written as well; the overall Test Case outcome is verified and overridden only when VECTR's priority-derived outcome would not faithfully represent the organizational verdict.

---

# PART I: REQUIREMENTS & ARCHITECTURE

## 1. Executive Summary

The Purple Team Workflow Orchestrator is a PowerShell 7 automation package that runs unattended from Windows Task Scheduler on a Purple Team workstation.

Its purpose is to eliminate repetitive reporting work around standing Cymulate assessments while preserving the team's existing systems of record:

- **Cymulate** executes scheduled BAS assessments and produces assessment/scenario results, a documented per-scenario detection result, and supporting findings/evidence.
- **Jira** remains the workflow, assignment, commentary, and operational result system of record.
- **VECTR** remains the technical Purple Team test repository for ATT&CK mapping, evidence, native defense outcomes, and historical results.

Version 2.0 remains a **scheduled batch job**, not a polling service. Windows Task Scheduler starts the PowerShell package at the configured schedule. Each invocation performs one bounded discovery pass against Cymulate, processes newly eligible terminal scenario results, synchronizes Jira and VECTR, reconciles prior incomplete work, writes its audit/state records, and exits.

The process must contain no long-running polling loop, sleep/retry loop used to wait for new assessments, daemon, Windows service, or continuously running listener. Until/unless Cymulate provides an officially supported webhook mechanism that the organization chooses to adopt, this system remains Task Scheduler-driven. A future webhook design would be event-driven rather than a reason to introduce a poller.

The core reporting unit is **one Cymulate scenario result**. Discovery is assessment-driven: the orchestrator queries launched assessments in the configured N-hour/N-day lookback window and evaluates every scenario result returned for each in-scope terminal assessment. If one completed launched assessment contains ten scenario results, all ten are reporting candidates and may produce ten independent Jira/VECTR record pairs. The orchestrator does not require a local allowlist of scenario IDs.

The system is expected to process **dozens to hundreds of routine scenario results in a reporting period**. Routine volume must not create external communications or require analyst interaction unless a result itself enters an explicit Jira manual-analysis workflow.

The design has one overriding integrity rule:

> A run is not complete merely because Cymulate completed. Jira and VECTR automation-owned state must be written, read back, and verified.

---

## 2. Project Goals & Non-Goals

### 2.1 Goals

- On each scheduled invocation, perform one bounded Cymulate discovery pass for terminal launched assessments and retain non-terminal integration-processing observations without provisioning result records prematurely.
- Fully enumerate scenario results from each relevant reportable terminal assessment.
- Use `GET /v2/assessments/launched/{id}/scenarios` as the primary per-scenario result source, including `scenarioResultID`, `scenarioID`, `status`, `statusReason`, and the documented `detection` value.
- Retrieve and fully page `/findings` for each relevant assessment as supporting evidence for prevention/control outcomes, AV evidence, EDR evidence, TheHive use-case evidence, technical evidence, and consistency checks.
- Normalize AV, EDR, and TheHive use-case evidence only from fields whose semantics have been validated against actual responses from `GET /v2/assessments/launched/{id}/findings` in the organization's Cymulate environment, and use the combined evidence to render the Jira `Blue team detection` verdict.
- Create exactly one Jira issue and exactly one VECTR Test Case for each in-scope scenario result.
- Use a dedicated Jira service account for all machine-generated Jira activity.
- Preserve individual operator accountability for human Jira comments, analysis, and overrides.
- Populate the existing Jira **`Blue team detection`** field using only its exact approved values.
- Assign the newly created VECTR Test Case an approved native overall defense outcome derived from the same normalized evidence used for Jira, and write attributable per-tool defense outcomes plus compact evidence notes.
- Never use VECTR tags as an automation mechanism.
- Survive workstation shutdown, missed Task Scheduler runs, API outages, and lost API responses without duplicate Jira/VECTR records.
- Maintain a durable local state and audit trail.
- Support Purple Team stakeholders including DLP, CSIRT, and Use Case Generation.
- Be deployable as a self-contained PowerShell folder on the existing Purple Team workstation.
- Be testable phase-by-phase with Pester and synthetic API responses before production writes are enabled.

### 2.2 Non-Goals (v2.0)

- Launch Cymulate assessments.
- Create, edit, delete, replay, stop, or relaunch Cymulate assessments or scenarios.
- Replace Cymulate scheduling.
- Replace Jira or VECTR.
- Use VECTR tags for orchestration, disposition, review routing, or synchronization.
- Copy VECTR tags into Jira labels.
- Query AV, EDR, TheHive, Elastic, Tanium, Trellix, or other controls directly when Cymulate already exposes sufficient result evidence.
- Automatically remediate failed controls.
- Automatically override human analyst decisions.
- Create arbitrary Jira custom fields.
- Create arbitrary VECTR Assessments, Campaigns, or Content Library objects.
- Invent a finding-to-scenario correlation rule that has not been proven against the live Cymulate tenant.
- Infer `NotDetected` merely because `/findings` returned no correlated finding records or because the launched-assessment scenario result `detection` field is empty.
- Use any alternate Cymulate findings/search/detail API for reporting, enrichment, detection, correlation, or evidence resolution outside the approved reporting endpoint set.
- Call an API operation that is represented only as an unbound component schema in the Swagger and is not present under `paths`.
- Invent scenario timestamps that the Cymulate API does not provide.
- Copy unrestricted Cymulate payload/output content into Jira, VECTR, or normal logs.
- Build a large dashboard or replace stakeholder SLA reporting.
- Introduce Python, Docker, FastAPI, PostgreSQL, or a continuously running application server.

---

## 3. System Architecture

### 3.1 Component Overview

| Component | Runtime | Responsibility |
|-----------|---------|----------------|
| **Invoke-PurpleTeamOrchestrator.ps1** | PowerShell / scheduled | One-shot scheduled batch entry point. Acquires lock, performs one Cymulate completed-result discovery pass, synchronizes Jira/VECTR, reconciles incomplete runs, writes audit/logs, exits. |
| **PurpleTeam.Cymulate.psm1** | PowerShell module | Read-only Cymulate API client. Terminal-assessment discovery, scenario-result enumeration, findings retrieval, and scenario metadata. |
| **PurpleTeam.Jira.psm1** | PowerShell module | Jira service-account authentication, issue creation, custom-field updates, comments, workflow transitions, assignment, read-back verification. |
| **PurpleTeam.Vectr.psm1** | PowerShell module | VECTR GraphQL client. Environment/Campaign/library mapping, Test Case creation, overall Test Case outcome assignment, per-tool defense outcomes, evidence notes, status/timeline, read-back verification. |
| **PurpleTeam.State.psm1** | PowerShell module | Durable local state, idempotency keys, discovery checkpoint, retries, audit events, reconciliation state. |
| **Local state store** | SQLite preferred | Persists ScenarioRun state, processed keys, retries, discovery checkpoint, optional scenario-override state, and audit records. |
| **Windows Task Scheduler** | Windows | Runs the orchestrator periodically and prevents unattended workflow drift. |

### 3.2 Execution Model

Scheduled task:

```text
Task Name: PurpleTeam-Orchestrator
Schedule:  <configured operational schedule>
Command:   pwsh.exe -NoProfile -NonInteractive -File C:\PurpleTeamAutomation\scripts\Invoke-PurpleTeamOrchestrator.ps1
```

The requirements document does not prescribe a five-minute or other pseudo-continuous cadence. The organization configures the Task Scheduler trigger(s) for completed-result processing.

Each invocation performs the following finite sequence:

1. Acquire the named mutex.
2. Load configuration and non-interactive secrets.
3. Read the last successful discovery checkpoint.
4. Perform one Cymulate terminal-assessment discovery pass.
5. Retrieve and fully page the required scenario results and launched-assessment findings.
6. Process new in-scope scenario results.
7. Create, update, and verify Jira state.
8. Create, update, and verify VECTR state.
9. Route Jira to `Analyze` when explicit analyst work is required; do not fabricate a `Blue team detection` value solely to represent uncertainty.
10. Reconcile incomplete prior records.
11. Write audit and operational log state.
12. Advance the discovery checkpoint only after the discovery cycle is safely persisted.
13. Release the mutex and exit.

The task must:

- Execute one finite batch and exit; it must not wait in a loop for a future Cymulate result.
- Run whether the interactive user is logged on or not.
- Run as soon as practical after a missed scheduled invocation.
- Prevent overlapping instances.
- Use a named mutex in addition to Task Scheduler overlap protection.
- Use only non-interactive credentials.
- Leave recoverable state if the workstation reboots mid-run.

### 3.3 System-of-Record Boundaries

| System | Authoritative For |
|--------|-------------------|
| **Cymulate** | Assessment lifecycle, scenario execution result, the raw per-scenario detection value, findings/evidence, configured integrations, and scenario metadata used to derive Jira `Blue team detection`. |
| **Jira** | Operational workflow, `Blue team detection`, assignee, analyst comments, human decisions, workflow transitions. |
| **VECTR** | Technical test history, ATT&CK mapping, evidence notes, native defense-tool outcomes, technical status/timeline. |
| **Local state** | Orchestration correlation, idempotency, retry/sync state, discovery checkpoint, audit trail. |

VECTR tags are **not** a system-of-record input or output for the orchestrator.

### 3.4 Correlation, Jira Key Generation & Idempotency Protocol

Every processed scenario result has two identifiers available **before Jira creation**:

**External idempotency key**

```text
<assessmentId>:<scenarioResultID>
```

This is derived from Cymulate and prevents duplicate processing when scheduled discovery windows overlap.

**Internal run ID**

```text
UUID v4
```

This is generated and persisted by the orchestrator before any external create request.

The Jira issue key does **not** exist before Jira creates the issue. Jira generates the human-facing key (for example `CIBETHICAL-1842`) as part of the issue-creation POST.

Required creation order:

1. Discover an in-scope Cymulate scenario result.
2. Create or load its internal `ScenarioRun` record.
3. Generate and persist `run_id` before any external create request.
4. Create the Jira issue with `run_id`, `assessmentId`, `scenarioResultID`, and `scenarioID`.
5. Persist the Jira-generated `CIBETHICAL-####` key immediately after the create or recovery succeeds.
6. Create or recover the mapped VECTR Test Case using the persisted Jira key.
7. Persist the VECTR Test Case ID and write the approved VECTR reference back to Jira.
8. Read back all automation-owned state and verify correlation.

The final correlation record must retain `run_id`, Cymulate `assessmentId`, Cymulate `scenarioResultID`, Cymulate `scenarioID`, the Jira key, and the VECTR Test Case ID.

### Jira key rules

- Never precompute, predict, reserve, or synthesize a `CIBETHICAL-####` value.
- Never require the Jira key in data that must exist before the Jira create POST.
- Persist `run_id` and Cymulate correlation identifiers before sending the Jira create request.
- Persist the Jira key returned by Jira immediately after a successful create response.
- Do not create the VECTR Test Case until the Jira key has been obtained or safely recovered.
- Any VECTR naming convention containing `[CIBETHICAL-####]` is therefore a post-Jira-create operation.
- Jira comments or VECTR records that require the Jira key must occur only after the key is known.

### Lost Jira create-response recovery

If Jira commits the issue but the create response is lost, the orchestrator does not yet know the generated Jira key.

Recovery must search Jira using pre-create correlation data, preferably the unique `Purple Run ID` (`run_id`) custom field and, as secondary validation, the Cymulate identifiers.

Recovery procedure:

1. Search Jira for the persisted `run_id`.
2. If exactly one matching issue is found, validate its Cymulate correlation fields and recover the Jira-generated key.
3. Persist the recovered Jira key before any VECTR provisioning.
4. If no issue is found, retry creation only after recovery logic has established that Jira did not commit the original request.
5. If multiple issues match, stop the unsafe create path, record a synchronization conflict, and require reconciliation.


The Jira summary is not sufficient as the sole recovery key because names/dates can collide and the Jira key was not known when the create request was issued.

Before creating any VECTR record, the orchestrator must have a persisted Jira key.

A lost create response must never be recovered by blindly creating another Jira issue.


### 3.5 Reporting Framework

The reporting framework is defined by the normative requirements in this specification rather than by a separate diagram set.

For each in-scope Cymulate scenario result, the orchestrator must:

1. Use `assessmentId:scenarioResultID` as the external idempotency key.
2. Maintain one durable internal `ScenarioRun` record. `ScenarioRun` is an orchestrator-defined persistence object, not a Cymulate, Jira, or VECTR API entity.
3. Preserve Cymulate assessment lifecycle, scenario execution status, scenario detection, and supporting findings as separate source facts.
4. Resolve prevention/control outcome before evaluating detection disposition.
5. Use only `GET /v2/assessments/launched/{id}/findings` for Cymulate findings evidence.
6. Normalize AV outcome, EDR outcome, and TheHive use-case outcome only from exact launched-assessment findings fields validated against actual Cymulate API responses from the organization's environment.
7. For each deterministic terminal run, produce exactly one approved Jira `Blue team detection` value. For an unresolved terminal run, use `MANUAL_REVIEW_REQUIRED` and Jira `Analyze` without fabricating a field value; re-evaluate it on later invocations.
8. From the same normalized evidence vector, derive the required VECTR overall Test Case outcome and any attributable per-tool Defense Tool outcomes. The VECTR outcome mapping is related to, but not a string copy of, the Jira verdict.
9. Keep Jira `Blue team detection` separate from Jira workflow/swimlane state. Jira Automation is preferred for ordinary field-driven workflow movement after a substantive field write is verified; verdict-less manual-review cases may be transitioned directly to `Analyze`.
10. Create or recover exactly one Jira issue and one VECTR Test Case per in-scope scenario result. The VECTR Test Case must receive an explicit overall outcome at creation or in the same provisioning transaction sequence.
11. Preserve VECTR as the technical reporting repository and never use VECTR tags as orchestrator workflow state.
12. Read back automation-owned Jira and VECTR state, including VECTR outcome state, before marking the internal record complete.
12. Retain unresolved evidence, synchronization failures, and manual-analysis requirements in durable state for reconciliation on a later scheduled invocation.

The detailed decision precedence, field mappings, retry rules, evidence requirements, and completion barriers in Sections 4 through 6 are authoritative.

---

## 4. Integration Component Requirements

### 4.1 Cymulate Read Adapter

**Reporting API boundary:** detection/evidence reporting is built from the launched-assessment family (`/v2/assessments/launched...`) plus `/v2/scenarios/{scenarioID}` metadata where required. No alternate Cymulate findings/search/detail API is part of the product.

| Aspect | Requirement |
|--------|-------------|
| **Reviewed API contract** | OpenAPI `3.1.1`, Cymulate API version `2.0.62`. Preserve the reviewed version in configuration/audit output so schema drift can be identified. |
| **Authentication** | The Swagger defines both `x-token` API-key authentication and OAuth2 client-credentials authentication. Version 2.0 production must support the approved `x-token` path; OAuth2 may be enabled only by configuration after a contract test. Credentials are retrieved non-interactively from approved secret storage. |
| **Discovery endpoint** | `GET /v2/assessments/launched`. Query the result-bearing terminal statuses `completed` and `partiallyCompleted`; also inspect `failed` and `stopped` for assessment-level failure handling. Non-terminal states may be read for health/settling observations, but the normal result processor does not provision Jira/VECTR records from them. The status query parameter accepts an array. |
| **Lifecycle normalization** | Store the raw assessment status. Normalize the filter values and response display values separately. While the assessment is in `queryingIntegration`, `finalizingQueryingIntegration`, `inProgress`, `inQueue`, `preparing`, `finalizing`, or another non-terminal state, persist an internal pending observation and revisit it on a later scheduled invocation. The normal result processor does not create a new Jira/VECTR pair or finalize detection from those states. |
| **Discovery window** | Each scheduled invocation queries launched assessments over the configured N-hour/N-day lookback window, using the persisted UTC discovery checkpoint plus a configurable overlap for restart/missed-run safety. Optional scope filters, if used, apply to launched-assessment attributes such as assessment/template ID, name, tag, or environment—not to a local scenario-ID allowlist. The process performs one finite discovery cycle per invocation, follows pagination/retries needed to complete that cycle, persists state, and exits. Do not limit recovery to only "today." |
| **Assessment detail** | Call `GET /v2/assessments/launched/{id}` to capture the current raw status, `integrations[]`, `startedAt`, `updatedAt`, environment, targets, and controls before deriving a result. |
| **Scenario results** | For each relevant assessment, call and fully page `GET /v2/assessments/launched/{id}/scenarios`. This endpoint is the authoritative per-scenario correlation source because it returns `scenarioResultID` and `scenarioID`. |
| **Primary scenario detection source** | Preserve the launched-assessment scenario result `detection` field exactly. The documented values are `Detected`, `Not Detected`, and an empty string. Empty means detection was not evaluated because no detection integration was configured or detection was not applicable; it does **not** mean `Not Detected`. Any other value is contract drift and routes to manual analysis. |
| **Scenario execution source** | Use `/scenarios.status` and `/scenarios.statusReason` to normalize `NOT_STARTED`, `RUNNING`, `SUCCESS`, `PARTIAL`, `FAILED`, or `UNKNOWN`. The Swagger does not enumerate scenario-status values, so the environment-validated allowlist and normalization table must be captured during Phase 0. |
| **Findings endpoint** | For each relevant terminal assessment, call and fully page `GET /v2/assessments/launched/{id}/findings`. The endpoint returns up to 100 findings per page and exposes `nextCursor`. This is the only Cymulate findings API used by the orchestrator. Findings support prevention/control outcomes, AV evidence, EDR evidence, TheHive use-case evidence, technical evidence, consistency checks, and VECTR notes. |
| **Finding correlation limitation** | The launched-finding schema exposes fields such as `assessmentId`, `scenarioName`, `attackId`, `testCaseKey`, `status`, `detection`, and controls, but it does not expose `scenarioResultID` or `scenarioID`. Therefore findings must not be the sole source of a per-scenario detection result, and a join by `scenarioName` alone is prohibited unless actual Cymulate API responses from the organization's environment prove it unique for the applicable assessment/test-point model. |
| **Outcome model** | Normalize `assessment_lifecycle`, `scenario_execution`, `prevention_outcome`, `scenario_detection`, `av_outcome`, `edr_outcome`, and `thehive_use_case_outcome` from the approved launched-assessment APIs. AV/EDR/TheHive values may be populated only from exact fields validated against actual Cymulate API responses from the organization's environment in the launched-assessment findings response. Unproven or contradictory evidence remains `UNKNOWN`/`NOT_AVAILABLE` and routes according to the Jira decision policy; do not query another Cymulate findings API to fill the gap. |
| **Zero findings** | An empty/unmatched findings set never creates a detection result by itself. A clear value in the launched-assessment scenario result `detection` field may still be used according to the approved decision policy; if the substantive verdict still cannot be resolved after bounded retries, enter `MANUAL_REVIEW_REQUIRED` and route Jira to `Analyze`. |
| **Multiple findings** | Aggregate findings only for evidence, prevention, and consistency using Phase-0-proven rules. Never use a generic `any Detected = Detected` shortcut to replace the scenario-level detection field. |
| **Consistency** | Compare findings-derived evidence to scenario `status`, `statusReason`, and `detection`. A material contradiction produces `EVIDENCE_CONFLICT`, enters `MANUAL_REVIEW_REQUIRED`, and routes Jira to `Analyze`; no contradictory automated `Blue team detection` conclusion is permitted. |
| **Pagination** | Implement one reusable pager. Prefer `nextCursor` when present; otherwise follow a usable `next` value or safe page progression. Stop on repeated cursors/tokens. Persist enough state to resume safely. |
| **Rate limit** | Client-side throttle must remain below the documented limit of 30 requests per 10 seconds per IP. Honor `Retry-After` when supplied; otherwise use bounded exponential backoff with jitter. |
| **Reporting endpoint boundary** | The reporting path uses the approved launched-assessment/scenario read endpoints only. Alternate Cymulate findings/search/detail APIs are prohibited. No Cymulate POST/PATCH/PUT/DELETE operation is permitted in the normal reporting path. |

#### 4.1.1 Evidence Resolution Requirement

The orchestrator uses one evidence-resolution path. It does not have alternate detection modes.

For every terminal scenario, the orchestrator must preserve the documented launched-assessment scenario result `detection` value and resolve the supporting control evidence needed by the Jira business mapping. The verdict model is based on the combination of:

- **AV outcome:** whether antivirus prevented or detected the activity.
- **EDR outcome:** whether endpoint controls prevented, detected, or observed the activity.
- **TheHive use-case outcome:** whether the expected Blue Team use case fired.

These dimensions answer different questions and must remain separate in local state even though they converge into one Jira `Blue team detection` value. Prevention is evaluated before detection. TheHive directly answers whether the expected Blue Team use case fired, while AV and EDR establish the prevention/detection context; the verdict engine uses all three together rather than treating any one source as sufficient in isolation.

The exact mapping from AV + EDR + TheHive evidence to the substantive Jira verdicts must be proven during Phase 0 using real/sanitized tenant fixtures. A confirmed prevention by AV or EDR maps to `Blocked` regardless of vendor. A confirmed expected TheHive use-case fire on a successfully executed, non-blocked scenario is a `Detected` candidate. A confirmed non-fire, when the applicable default evidence policy (plus any optional scenario-specific override) is satisfied and the AV/EDR/TheHive evidence is complete and consistent, is a `NotDetected` candidate. Missing, contradictory, or uncorrelatable required evidence does not produce a `Blue team detection` value; it enters `MANUAL_REVIEW_REQUIRED` and routes Jira to `Analyze`.

No alternate Cymulate findings/search/detail endpoint may be introduced to resolve the verdict. If required evidence cannot be resolved reliably from the approved launched-assessment/scenario APIs after bounded retries, the orchestrator must enter `MANUAL_REVIEW_REQUIRED`, route Jira to `Analyze`, leave `Blue team detection` unset until a substantive verdict is supported, and record the evidence limitation.

### 4.2 Jira Adapter

#### 4.2.1 Service Account

Production Jira automation must use a dedicated non-person account.

Recommended display name:

```text
Purple Team Orchestrator
```

The scheduled task must never use a Red/Purple Team operator's personal Jira credentials.

The service account owns machine actions:

- Issue creation.
- Automation-owned field updates.
- Automated comments.
- Approved workflow transitions.
- Correlation-field updates.
- Assignment changes when configured.

Human operators continue to use personal accounts for:

- Analyst comments.
- Investigation.
- Manual result changes.
- Risk/remediation decisions.
- Manual workflow overrides.

The service account must use least privilege and must not receive Jira administrative rights merely to simplify automation.

#### 4.2.2 Jira Project & Issue Identity

```text
Project: CIBETHICAL
Key:     CIBETHICAL-####   # generated by Jira during issue-creation POST
```

The orchestrator must not know or attempt to construct the Jira issue key before creation.

The create request is correlated using the already-persisted `run_id` and Cymulate identifiers. The Jira key returned by the create POST becomes the canonical human-facing identifier for all downstream work.

Recommended automated summary:

```text
[Purple Team] <Scenario Name> - <YYYY-MM-DD>
```

The summary is human-readable and must **not** be treated as the primary idempotency/recovery key.

Recommended correlation fields:

```text
Purple Run ID
Cymulate Assessment ID
Cymulate Scenario Result ID
Cymulate Scenario ID
VECTR Test Case ID
Stakeholder
Run Type
Target
```

Actual custom-field IDs must be configuration values discovered in Phase 0.

#### 4.2.3 `Blue team detection`

The existing Jira field:

```text
Blue team detection
```

is the authoritative Jira operational result/disposition field.

Allowed values are exactly:

```text
NotApplicable
NotDetected
Detected
NotStarted
Blocked
AttackFailed
PartialTesting
TestingInProgress
```

Requirements:

- Preserve capitalization and spelling exactly.
- Reject any generated value outside the approved list.
- The Swagger defines Cymulate source values, not the organization's Jira business semantics. Every Cymulate-to-Jira mapping remains configuration-driven and must be approved against the organization's actual Jira configuration and team practice during Phase 0.
- Store the Jira custom-field ID and option IDs in configuration after live metadata discovery.
- When the orchestrator has sufficient evidence for a substantive verdict, it must write exactly one approved `Blue team detection` value and read it back. An issue routed to manual analysis may temporarily leave the field unset until the script or an analyst can support a substantive verdict.
- The normal result processor creates Jira/VECTR records only after a reportable terminal scenario result is available. `TestingInProgress` remains an approved value for a separately approved pre-provisioned/active-ticket workflow. Terminal ambiguity is represented by internal `MANUAL_REVIEW_REQUIRED` plus Jira `Analyze`, not by a synthetic `Blue team detection` value.
- For deterministic cases, include the substantive value in Jira issue creation when create metadata permits it; otherwise write and read it back immediately after creation. For `MANUAL_REVIEW_REQUIRED`, create or recover the issue without fabricating a field value, route it to `Analyze`, and populate the field later when the evidence supports one of the approved outcomes.
- Do not infer `NotDetected` from an empty findings set or an empty scenario detection value.
- Do not use `NotApplicable` to hide a missing required integration.
- Preserve human overrides according to the field-ownership policy.

##### Source hierarchy

The decision engine consumes evidence in this order:

```text
1. Parent assessment lifecycle and configured integrations
2. Scenario status and statusReason
3. Scenario detection (Detected / Not Detected / empty / unknown)
4. Launched-assessment findings for prevention/control evidence, AV outcome, EDR outcome, TheHive use-case outcome, and consistency
5. Default evidence policy plus any optional scenario-specific override for detection applicability, expected use case, and evidence requirements
```

The launched-assessment scenario result `detection` field is authoritative for Cymulate's documented scenario-level detection result. The final Jira verdict additionally uses the environment-validated AV, EDR, and TheHive use-case evidence required by the default evidence policy and any applicable optional scenario override.

##### Normalized outcome vector

For each ScenarioRun, the Cymulate adapter must expose:

```text
assessment_lifecycle
    NOT_STARTED
    RUNNING
    QUERYING_INTEGRATION
    TERMINAL
    UNKNOWN

scenario_execution
    NOT_STARTED
    RUNNING
    SUCCESS
    PARTIAL
    FAILED
    UNKNOWN

prevention_outcome
    BLOCKED
    NOT_BLOCKED
    NOT_APPLICABLE
    UNKNOWN

scenario_detection
    DETECTED
    NOT_DETECTED
    NOT_EVALUATED
    UNKNOWN

av_outcome
    BLOCKED
    DETECTED
    NOT_DETECTED
    NOT_APPLICABLE
    UNKNOWN
    NOT_AVAILABLE

edr_outcome
    BLOCKED
    DETECTED
    NOT_DETECTED
    NOT_APPLICABLE
    UNKNOWN
    NOT_AVAILABLE

thehive_use_case_outcome
    FIRED
    NOT_FIRED
    NOT_APPLICABLE
    UNKNOWN
    NOT_AVAILABLE

evidence_consistency
    CONSISTENT
    PARTIAL
    CONFLICT
    UNKNOWN
```

The raw values, source endpoint, source field, observation timestamp, and normalizer version must be retained with each normalized value.

##### Decision precedence

Before applying the table, the parent assessment and child scenario must be reportable/terminal. If Cymulate is still preparing, running, querying integrations, or finalizing integration results, the normal result processor stores pending state locally and provisions no new Jira/VECTR result pair. `TestingInProgress` is reserved for a separately approved pre-provisioned-ticket workflow.

Higher-priority terminal outcomes stop evaluation of lower-priority outcomes.

| Priority | Condition | Jira `Blue team detection` | Required handling |
|----------|-----------|----------------------------|-------------------|
| 1 | Terminal scenario explicitly has not started or was not tested | `NotStarted` | Use only for a terminal scenario result whose status semantics have been validated against actual Cymulate API responses from the organization's environment, or a separately approved pre-provisioned workflow record. |
| 2 | Scenario execution failed | `AttackFailed` | No prevention/detection conclusion may override execution failure. |
| 3 | Scenario execution was partial | `PartialTesting` | Use only for a partial status whose semantics have been validated against actual Cymulate API responses from the organization's environment. |
| 4 | Scenario ran and AV or EDR positively proves that the activity was prevented/blocked | `Blocked` | Vendor attribution may be retained as evidence but does not change the Jira value. Preserve detection/use-case evidence separately if it also exists. |
| 5 | Scenario ran, was not blocked, and the expected TheHive use case positively fired with consistent supporting evidence | `Detected` | Preserve the Cymulate scenario `detection` value and AV/EDR evidence as supporting source facts. |
| 6 | Scenario ran, was not blocked, the expected TheHive use case is confirmed not to have fired, and the applicable required AV/EDR/TheHive evidence is complete and consistent | `NotDetected` | This is a confirmed detection/use-case miss, not an inference from absent findings or empty source fields. |
| 7 | Detection is empty and the applicable default policy or optional scenario override says detection is genuinely not applicable | `NotApplicable` | Requires `detection_applicability = NOT_APPLICABLE`; absence of an integration alone is not enough. |


##### Manual-review routing (no `Blue team detection` verdict)

The following conditions do **not** produce a `Blue team detection` value. They set `processing_status = MANUAL_REVIEW_REQUIRED`, record `manual_review_reason`, and route the Jira issue to `Analyze`:

- Required AV/EDR/TheHive evidence is missing, contradictory, ambiguous, or cannot be correlated after bounded retries.
- The scenario result `detection` field is empty even though detection is required, an expected integration/use case is missing, or applicability is unknown.
- Cymulate returns an undocumented detection value or unrecognized terminal scenario status.
- A material evidence conflict or schema/contract issue prevents a substantive verdict.

The orchestrator must re-evaluate these runs on later scheduled invocations. If the evidence later supports a substantive outcome, the script writes the correct `Blue team detection` value and read-backs it before moving the issue out of `Analyze`, unless a human-authoritative value has already been recorded.

##### Key semantic rules

**`AttackFailed` is not `NotDetected`.** A test that did not execute correctly did not fairly test the defensive stack.

**`NotDetected` is never inferred from absence.** It requires a successfully executed, non-blocked scenario plus a complete and consistent environment-validated AV/EDR/TheHive evidence set proving that the expected Blue Team use case did not fire. Empty findings and empty detection are not negative evidence.

**Manual review is workflow state, not a detection verdict.** Missing, contradictory, ambiguous, or uncorrelatable required AV/EDR/TheHive evidence after bounded retries produces internal `MANUAL_REVIEW_REQUIRED` and Jira `Analyze`. `Blue team detection` remains unset until the evidence supports a substantive approved value. Non-terminal API or integration processing remains internal pending state. A separate pre-provisioned ticket workflow may use `TestingInProgress`.

**`NotApplicable` is policy-driven.** A required detection integration that was omitted or unhealthy is a review condition, not `NotApplicable`.

**Prevention has precedence over detection.** If AV or EDR blocked the scenario and detection/use-case evidence also exists, Jira records `Blocked`; the additional evidence remains in the evidence summary and VECTR outcome details.

##### Evidence provenance

For every `Blue team detection` write, local state and the Jira automated comment must retain:

```text
assessment id and raw assessment status
configured integrations
scenarioResultID and scenarioID
raw scenario status and statusReason
raw scenario detection value
normalized execution/prevention/detection values
finding IDs and correlation status
AV outcome and source, when available
EDR outcome and source, when available
TheHive use-case outcome and source, when available
evidence consistency result
selected Jira value
mapping rule/version
evidence resolution status and source coverage
```

Example:

```text
CYMULATE OUTCOME DECISION
--------------------------------------------------
Assessment Status:        Completed
Scenario Execution:       SUCCESS
Prevention:               NOT_BLOCKED
Scenario Detection:       NOT_DETECTED
AV Outcome:               NOT_DETECTED
EDR Outcome:              DETECTED
TheHive Use Case:         NOT_FIRED
Evidence Consistency:     CONSISTENT
Evidence Resolution:      COMPLETE

Blue team detection:      NotDetected
Decision Rule:            BTDMAP-2.0-06
```

Sensitive or unbounded payload content must not be copied into Jira, VECTR, or normal logs.

#### 4.2.4 Jira Workflow

Jira issue status/swim lane and `Blue team detection` are separate. The orchestrator owns the technical field value; Jira Automation is the preferred mechanism for applying ordinary status transitions after that value is written and verified.

Recommended status policy for the existing board columns:

| Verdict / routing condition | Jira column/status target | Policy |
|-----------------------------|---------------------------|--------|
| `NotStarted` | `Schedule` | Scenario has not started or was not tested. |
| `TestingInProgress` | `In Progress` | Reserved for a separately approved pre-provisioned/active-ticket workflow; the normal terminal-result processor does not create a ticket in this state. |
| `MANUAL_REVIEW_REQUIRED` with `Blue team detection` unset | `Analyze` | Human Blue Team review is required before a substantive verdict can be assigned. |
| `AttackFailed`, `PartialTesting` | `Analyze` | Determine cause and whether a rerun is required. |
| `Detected`, `NotDetected`, `Blocked`, `NotApplicable` | `Validation` | Machine result is available and awaits the team's normal validation step. |
| Any value | `On Hold` | Manual-only exception state; never selected merely because an API retry is pending. |
| Validated result | `Done` | Reached only after the approved validation/closure rule; the reporting orchestrator does not bypass validation. |

Requirements:

- Use the exact board label `Analyze`, not `Analysis`.
- For substantive verdicts, the orchestrator must write and read back `Blue team detection` before the dependent Jira Automation transition is expected. For `MANUAL_REVIEW_REQUIRED`, the orchestrator may transition the issue directly to `Analyze` because there is intentionally no verdict value to trigger a field-based Jira Automation rule.
- Jira Automation rules should be idempotent and should not transition an issue backward after a human has advanced or overridden it, unless an approved reconciliation rule explicitly allows that action.
- The PowerShell adapter must query available Jira transitions and map configured transition names; it must not hard-code transition IDs.
- Human changes to shared fields must not be blindly overwritten. If an operator intentionally changes `Blue team detection`, record the conflict or accept the configured human-authoritative value.

### 4.3 VECTR GraphQL Adapter

| Aspect | Requirement |
|--------|-------------|
| **Endpoint** | `POST https://<host>/sra-purpletools-rest/graphql`. |
| **Authentication** | `Authorization: VEC1 <key_id>:<key_secret>`. |
| **Environment** | Use configured VECTR environment name as GraphQL `db`. |
| **Campaign** | Every reportable scenario result must resolve to a pre-approved Campaign ID. Resolution may use a configured default, assessment/template-level mapping, or optional scenario-specific override. Lack of a per-scenario override must never suppress discovery or processing. Version 2.0 does not create Campaigns dynamically. |
| **Library mapping** | Prefer `libraryTestCaseId`; do not build new automation around deprecated `templateId`. Resolve the library mapping from configured defaults/assessment-level mappings and optional scenario overrides. If no deterministic VECTR mapping can be resolved, keep the scenario result in synchronization/configuration error state rather than ignoring it or creating a guessed Test Case. |
| **Creation** | Prefer `testCase.createWithTemplateMatchByLibraryId`. Set `createNewIfNotExists=false`. |
| **Create correlation** | Set create `clientId = run_id`. |
| **Jira dependency** | VECTR Test Case creation occurs only after Jira create/recovery has produced a real `CIBETHICAL-####`. |
| **Test Case name** | Build the deterministic VECTR name only after `jira_key` is known, e.g. `[CIBETHICAL-1842] <Scenario Name> - <YYYY-MM-DD>`. |
| **Lost response recovery** | Search deterministic Test Case name; verify Campaign + `libraryTestCaseId` before reusing. |
| **Pagination** | Implement GraphQL cursor pagination (`first`, `after`, `pageInfo.endCursor`, `hasNextPage`). |
| **GraphQL errors** | HTTP success is insufficient. Non-empty top-level `errors` is a failed/partial operation. |
| **Outcome catalog** | Query VECTR `outcomes` and/or `outcomesTree` and resolve approved outcome `path`/`id` values at startup or configuration validation. Do not hard-code database-specific outcome IDs. |
| **Overall Test Case outcome** | Every newly created local Test Case must receive an explicit approved overall defense outcome from the same normalized AV/EDR/TheHive evidence used by the Jira verdict engine. For creation, use `CreateTestCaseDataInput.outcomePath`; do not build new logic around the deprecated create-time `outcome` field. |
| **Per-tool outcome mode** | Prefer per-tool defense outcomes when the Assessment/Test Case uses `dataVer=2`. Resolve each `defenseToolId` and `outcomeId`; populate `defenseToolOutcomes` only when the evidence is attributable to that tool. |
| **Overall override** | In per-tool mode, VECTR normally derives the overall Test Case outcome from tool outcomes. If that priority-derived result would not faithfully represent the organizational verdict, set the approved overall `outcomePath` with `overrideOutcome=true` using behavior proven during Phase 0. |
| **Evidence attribution** | Only set a per-tool outcome when Cymulate evidence is attributable to that specific Defense Tool. |
| **Evidence notes** | Write compact approved finding/evidence summaries to `outcomeNotes`, including the Jira verdict (when substantive), the VECTR outcome rule used, and why an overall override was or was not required. |
| **Status** | Use only native VECTR statuses supported by the target VECTR GraphQL schema. |
| **Timestamps** | Phase 0 must approve one historical timestamp strategy: Test Case fields or explicit timeline events. Never use orchestration time as attack time. |
| **Deprecated fields** | Do not depend on deprecated `GoldStandard`, primary `templateId`, `activityLogged`, `alertSeverity`, or the deprecated create-time `outcome` field. Use `libraryTestCaseId`, `outcomePath`, and per-tool outcome structures where supported. |

#### 4.3.1 VECTR Overall Blue Team / Test Case Outcome Assignment

VECTR documentation calls this value the **Test Case outcome** in the Defense Activity model. In this product specification, the phrase **VECTR Blue Team outcome** refers to that native overall Test Case outcome; it is not a new custom field.

The VECTR outcome is rendered from the **same normalized evidence vector** used for Jira:

- scenario execution;
- prevention outcome;
- AV outcome;
- EDR outcome;
- TheHive use-case outcome;
- evidence consistency.

There must not be a second independent VECTR verdict engine. One evidence-resolution decision produces both the Jira operational verdict and the VECTR technical outcome target. The labels do not need to be identical because Jira and VECTR model different concepts.

##### Semantic mapping

Exact VECTR outcome paths are environment-specific and must be resolved and approved during Phase 0 using `outcomes` / `outcomesTree`. The following table defines the required **semantic target**, not a hard-coded VECTR ID or path:

| Result/evidence condition | Jira `Blue team detection` | Required VECTR overall outcome semantics |
|---|---|---|
| AV or EDR positively prevented the scenario | `Blocked` | A configured outcome under the VECTR `Blocked` branch that most accurately reflects the evidence. |
| Scenario ran, was not blocked, and the expected TheHive use case fired | `Detected` | A configured `Alerted` outcome path representing the successful Blue Team detection/use-case result. |
| Scenario ran, was not blocked, and the expected TheHive use case did not fire | `NotDetected` | A configured unsuccessful-defense outcome that preserves the actual evidence. Use `None` only when there was genuinely no relevant defensive observation; if AV/EDR produced local telemetry or another partial defensive result, select the approved `Logged`/local-telemetry or other unsuccessful path instead. |
| Detection is genuinely not applicable | `NotApplicable` | The configured VECTR `N/A` outcome. |
| Scenario did not start | `NotStarted` | The configured VECTR `TBD` outcome because no valid defensive conclusion was produced. |
| Scenario execution failed | `AttackFailed` | The configured VECTR `TBD` outcome; the attack failure is explained in `outcomeNotes` and must not be converted into a defense success/failure. |
| Scenario execution was partial | `PartialTesting` | The configured VECTR `TBD` outcome pending rerun/analysis. |
| Separately approved active/pre-provisioned workflow | `TestingInProgress` | The configured VECTR `TBD` outcome until the test reaches a substantive result. |
| Terminal evidence is missing, contradictory, ambiguous, or uncorrelatable | field unset; `MANUAL_REVIEW_REQUIRED` | The configured VECTR `TBD` outcome. When the case is later resolved, the orchestrator must update both the Jira verdict (if still automation-owned) and the VECTR overall outcome. |

`TBD`, `N/A`, `Blocked`, `Alerted`, `Logged`, and `None` above refer to VECTR outcome **semantics/categories**. The implementation must resolve the exact selectable `Outcome.path` and `Outcome.id` values from the target VECTR instance rather than assuming a literal path.

##### Per-tool outcomes and overall outcome

For `dataVer=2` Test Cases, AV, EDR, and TheHive should be represented as individual Defense Tools when those tools exist in the VECTR environment and Cymulate evidence can be attributed to them. `DefenseToolOutcomeInput` requires both a `defenseToolId` and an `outcomeId`.

VECTR derives the overall Test Case outcome from per-tool outcomes using its native priority behavior. The orchestrator must compare the returned/derived overall outcome with the outcome required by the verdict mapping above. If they differ materially, the orchestrator must use an approved overall override rather than allow VECTR's priority ordering to contradict the Purple Team verdict.

A key example is `NotDetected`: an EDR may have local telemetry or an alert while the expected TheHive use case does not fire. In that case, the EDR's per-tool outcome must still be preserved, while the overall Test Case outcome must represent the unsuccessful end-to-end detection result using the approved VECTR path rather than being automatically promoted to a successful `Alerted` outcome.

##### Creation, update, and verification contract

1. Before Test Case creation, resolve the desired overall outcome path and any attributable `defenseToolOutcomes` from the normalized result.
2. Create the Test Case with the approved `outcomePath`, `outcomeNotes`, `dataVer` policy, and per-tool outcomes where supported. Set `overrideOutcome=true` only when required by the approved mapping.
3. Read the created Test Case back and verify at minimum `outcome { id name path }`, `overrideOutcome`, `dataVer`, and `defenseToolOutcomes` against the desired state.
4. A deterministic run cannot become `COMPLETE` if the VECTR overall outcome is missing or does not match the approved mapping.
5. For `MANUAL_REVIEW_REQUIRED` or other non-substantive states, VECTR receives the approved `TBD` outcome rather than a fabricated defense conclusion.
6. If a later scheduled invocation or human-authoritative resolution changes the substantive result, update and read back the VECTR outcome as well as the Jira verdict where automation still owns the field. Phase 0 must prove the exact `testCase.update` contract used for post-create overall-outcome changes before production automation relies on it.

#### 4.3.2 VECTR Tag Non-Interference

The orchestrator must perform **zero VECTR tag operations**.

It must not:

```text
add tags
remove tags
create automation tags
read tags to decide workflow
read tags to decide Jira outcome
mirror tags into Jira
reconcile tag differences
```

This includes:

```text
BLUE_TEAM_ANALYZE
DONE
TODO_PCSIRT
TODO_PURPLE
TODO_PCSIRT_LATER
```

and any future human-owned workflow/remediation tag.

Humans may continue to use VECTR tags manually for legitimate cases. Automation must ignore them completely.

### 4.4 State, Retry & Reconciliation Engine

| Aspect | Requirement |
|--------|-------------|
| **Durability** | Persist state across process exit and workstation reboot. SQLite is preferred. |
| **Run uniqueness** | Unique constraint on external idempotency key `<assessmentId>:<scenarioResultID>`. |
| **Processed events** | Duplicate discovery must be safe and produce no duplicate side effects. |
| **Retries** | Retry transient transport/API failures with exponential backoff + jitter. |
| **No blind retries** | Do not blindly retry schema errors, invalid transitions, permission errors, or authentication failures. |
| **Read-after-write** | Critical Jira/VECTR writes must be read back and compared to expected state. |
| **Reconciliation** | Every invocation must revisit incomplete runs and repair safe automation-owned drift. |
| **Human conflict** | Human-modified shared Jira fields are not overwritten blindly. |
| **VECTR tags** | Excluded from reconciliation entirely. |
| **Discovery checkpoint** | Advance only after the scheduled discovery cycle has completed safely. |
| **Offline recovery** | On restart, process the persisted lookback/window and incomplete runs before declaring healthy. |
| **Kill switch** | `AUTOMATION_WRITE_ENABLED=false` computes desired state but performs no Jira/VECTR mutations. |

### 4.5 Secrets & Security

- No API tokens in source code.
- No API tokens in `settings.psd1` committed to Git.
- No authorization headers in logs.
- Scheduled-task identity must retrieve secrets non-interactively.
- Supported mechanisms may include approved enterprise secret management, Windows Credential Manager, or DPAPI-protected local secrets.
- Credentials must be independently rotatable for Cymulate, Jira, and VECTR.
- Jira automation identity must be non-personal.
- VECTR API identity/key must be least privilege.
- Cymulate token should be read-only for Version 2.0 wherever permission controls allow it.

---

## 5. Core Data Model

### 5.1 Optional Scenario Overrides

Scenario-specific configuration is optional and exists only to override a default or assessment/template-level mapping. Every scenario result returned by an in-scope launched assessment is processed whether or not an override exists. In the absence of an override, the orchestrator uses the configured default/assessment-level evidence, Jira, and VECTR policies.

| Field | Purpose |
|-------|---------|
| `cymulate_scenario_id` | Stable Cymulate scenario ID used only to match an optional override. |
| `stakeholder_override` | Optional DLP / CSIRT / Use Case Generation / other owner override. |
| `detection_applicability` | Optional `REQUIRED` or `NOT_APPLICABLE` override used to resolve an empty Cymulate detection value. |
| `expected_detection_integrations` | Optional override for expected AV/EDR/TheHive integration/control evidence. |
| `expected_thehive_use_case` | Optional expected TheHive use case/rule identity or approved correlation selector. |
| `required_verdict_evidence` | Optional override declaring which AV, EDR, and TheHive evidence dimensions must be resolved before an automated `Detected` or `NotDetected` verdict is allowed. |
| `jira_issue_type_override` | Optional Jira issue-type override. |
| `vectr_environment_name_override` | Optional VECTR GraphQL `db` override. |
| `vectr_assessment_id_override` | Optional parent Assessment override. |
| `vectr_campaign_id_override` | Optional Campaign override. |
| `vectr_library_test_case_id_override` | Optional Content Library Test Case override. |

There is no scenario eligibility flag or scenario-ID allowlist. Absence of an override must never cause a discovered scenario result to be ignored. If a required downstream mapping cannot be resolved from defaults, assessment/template mappings, or an optional override, the scenario remains a tracked ScenarioRun and enters the appropriate synchronization/configuration-error path.

### 5.2 ScenarioRun

One record per processed Cymulate scenario result.

| Field | Purpose |
|-------|---------|
| `run_id` | Internal UUID. |
| `external_key` | `<assessmentId>:<scenarioResultID>`. |
| `assessment_id` | Cymulate launched-assessment ID. |
| `assessment_status_raw` | Exact status returned by Cymulate. |
| `assessment_integrations` | Exact configured integration values returned for the assessment. |
| `assessment_updated_at` | Source `updatedAt`, retained for evidence freshness and settling analysis. |
| `scenario_result_id` | Cymulate scenario-result ID. |
| `scenario_id` | Cymulate stable scenario ID. |
| `scenario_status_raw` | Exact scenario `status`. |
| `scenario_status_reason` | Exact scenario `statusReason`, subject to output minimization. |
| `scenario_detection_raw` | Exact launched-assessment scenario result `detection` value, including empty string. |
| `assessment_lifecycle` | Normalized lifecycle state. |
| `scenario_execution` | Normalized `NOT_STARTED/RUNNING/SUCCESS/PARTIAL/FAILED/UNKNOWN`. |
| `prevention_outcome` | Normalized `BLOCKED/NOT_BLOCKED/NOT_APPLICABLE/UNKNOWN`. |
| `scenario_detection` | Normalized `DETECTED/NOT_DETECTED/NOT_EVALUATED/UNKNOWN`. |
| `av_outcome` | Normalized `BLOCKED/DETECTED/NOT_DETECTED/NOT_APPLICABLE/UNKNOWN/NOT_AVAILABLE`. |
| `edr_outcome` | Normalized `BLOCKED/DETECTED/NOT_DETECTED/NOT_APPLICABLE/UNKNOWN/NOT_AVAILABLE`. |
| `thehive_use_case_outcome` | Normalized `FIRED/NOT_FIRED/NOT_APPLICABLE/UNKNOWN/NOT_AVAILABLE`. |
| `evidence_consistency` | `CONSISTENT/PARTIAL/CONFLICT/UNKNOWN`. |
| `finding_ids` | Finding IDs used as supporting evidence. |
| `findings_retrieval_status` | Paging/retrieval state for the launched findings endpoint. |
| `findings_correlation_status` | `NOT_REQUIRED/PROVEN/PARTIAL/AMBIGUOUS/FAILED`. |
| `blue_team_detection_candidate` | Nullable until a substantive verdict is supported. When non-null, it must be one exact approved Jira value. |
| `blue_team_detection_rule` | Mapping rule/version that produced the value. |
| `manual_review_reason` | Required when `processing_status = MANUAL_REVIEW_REQUIRED`; also recommended for `AttackFailed` and `PartialTesting` to explain the cause/rerun requirement. |
| `jira_key` | Nullable until Jira create/recovery succeeds; then stores Jira-generated `CIBETHICAL-####`. Never preallocated locally. |
| `vectr_test_case_id` | VECTR local Test Case ID. |
| `vectr_overall_outcome_path` | Desired/verified VECTR overall Test Case `Outcome.path`, derived from the same normalized evidence as the Jira verdict. |
| `vectr_overall_outcome_id` | Read-back VECTR overall `Outcome.id`. |
| `vectr_override_outcome` | Whether the overall Test Case outcome is explicitly overridden because the per-tool priority-derived outcome would misrepresent the verdict. |
| `vectr_defense_tool_outcomes_json` | Attributable per-tool `defenseToolId`/`outcomeId` assignments used for AV, EDR, TheHive, or other approved Defense Tools. |
| `vectr_outcome_rule` | Versioned mapping rule that converted normalized evidence/Jira verdict state into the VECTR semantic outcome target. |
| `processing_status` | Orchestrator state. |
| `jira_sync_status` | Jira synchronization state. |
| `vectr_sync_status` | VECTR synchronization state. |
| `retry_count` | Retry tracking. |
| `last_error` | Last classified failure. |
| `created_at/updated_at/completed_at` | Audit timestamps. |

### 5.3 Processing States

```text
DISCOVERED
WAITING_FOR_RESULT
WAITING_FOR_INTEGRATION
CORRELATED
EVIDENCE_PENDING
PROVISIONING
REPORTING
MANUAL_REVIEW_REQUIRED
SYNC_PENDING
COMPLETE
FAILED
```

A scheduled invocation never waits in a resident loop. `WAITING_FOR_RESULT`, `WAITING_FOR_INTEGRATION`, and `EVIDENCE_PENDING` are persisted states revisited by the next Task Scheduler invocation.

A ScenarioRun cannot be marked `COMPLETE` merely because the parent assessment is terminal. The required Jira/VECTR automation-owned state must be written and read back. A run routed to `MANUAL_REVIEW_REQUIRED` is automation-synchronized but remains human-workflow open.

---

## 6. End-to-End Integration Test Plan

| Test | Event | Expected Orchestrator Response |
|------|-------|--------------------------------|
| Assessment lifecycle | Assessment is `queryingIntegration` or `finalizingQueryingIntegration` | Persist an internal pending observation; create no new Jira/VECTR result pair; revisit on the next invocation. |
| Terminal discovery | Assessment reaches `completed` or `partiallyCompleted` | Scenario pages are fully enumerated; terminal decision evaluation begins. |
| Scenario enumeration | Assessment contains N scenario results | Every page is retrieved; all N scenario results from the in-scope assessment become ScenarioRuns; no scenario-ID allowlist exists; idempotency is `assessmentId:scenarioResultID`. |
| Confirmed detection | Valid, unblocked scenario has consistent evidence and the expected TheHive use case fires | Jira `Detected`; VECTR receives the approved `Alerted`-semantics overall outcome; raw Cymulate detection plus AV/EDR/TheHive evidence retained. |
| Confirmed detection miss | Valid, unblocked scenario has complete, consistent AV/EDR/TheHive evidence and the expected TheHive use case is confirmed `NOT_FIRED` | Jira `NotDetected`; VECTR receives the approved unsuccessful-defense overall outcome path that preserves any local AV/EDR visibility rather than blindly mapping to `None`. |
| Incomplete verdict evidence | Valid, unblocked scenario lacks one or more required AV/EDR/TheHive evidence dimensions after bounded retries | Leave `Blue team detection` unset; set `MANUAL_REVIEW_REQUIRED`; route Jira to `Analyze`; assign VECTR `TBD` semantics. |
| Conflicting verdict evidence | AV, EDR, TheHive, scenario detection, or findings evidence materially conflicts | Leave `Blue team detection` unset; set `MANUAL_REVIEW_REQUIRED`; route Jira to `Analyze`; assign VECTR `TBD` semantics. |
| Empty detection, N/A | Scenario detection is empty and the applicable default policy or optional scenario override says `NOT_APPLICABLE` | Jira `NotApplicable`. |
| Empty detection, required | Scenario detection is empty after terminal status and detection is required | Leave `Blue team detection` unset; route to `Analyze`; comment identifies missing/not-evaluated detection. |
| Missing required integration/use case | A configured AV/EDR/TheHive evidence source or expected TheHive use case is absent | Leave `Blue team detection` unset; route to `Analyze`; configuration failure is recorded. |
| Scenario execution failure | Scenario status map identifies failure | Jira `AttackFailed`; VECTR overall outcome uses approved `TBD` semantics; no defense success/failure is inferred. |
| Partial execution | Scenario status map identifies partial execution | Jira `PartialTesting`; VECTR overall outcome uses approved `TBD` semantics. |
| Prevention | Scenario ran and AV or EDR positively proves prevention | Jira `Blocked`; VECTR receives the approved `Blocked`-semantics overall outcome and attributable per-tool outcome(s); vendor attribution is retained as evidence. |
| Prevention plus detection | Scenario was blocked and detection/TheHive evidence also indicates detection | Jira `Blocked` wins; detection/use-case evidence is preserved. |
| Empty findings with clear scenario result | Findings count is zero but scenario detection is explicit | Scenario detection remains usable, but zero findings contributes no evidence. If required AV/EDR/TheHive evidence is unresolved, leave the verdict unset and route to `Analyze`. |
| Empty findings with empty detection | Findings count is zero and detection is empty | `NotApplicable` only by policy; otherwise leave the verdict unset and route to `Analyze`. |
| Evidence conflict | Findings evidence materially contradicts status/detection | Leave `Blue team detection` unset; set `EVIDENCE_CONFLICT` / `MANUAL_REVIEW_REQUIRED`; route to `Analyze`. |
| Unknown detection value | Cymulate returns an undocumented detection string | Leave `Blue team detection` unset; route to `Analyze`; raise schema-drift alert; retain the raw value. |
| Launched-findings verdict evidence | `/v2/assessments/launched/{id}/findings` exposes environment-validated AV, EDR, and TheHive evidence | Normalize only the proven fields; if required verdict evidence cannot be resolved, enter `MANUAL_REVIEW_REQUIRED` and route to `Analyze`. |
| Prohibited Cymulate endpoint guard | Code attempts to call an alternate Cymulate findings/search/detail API or another non-allowlisted reporting endpoint | Test fails before any HTTP request is issued. |
| Jira creation | New ScenarioRun | Deterministic cases include/write the exact substantive value. Manual-review cases may be created with `Blue team detection` unset and are routed to `Analyze`. |
| Jira create response lost | Jira committed but response was lost | Recover by `run_id`; no duplicate issue. |
| VECTR creation | New ScenarioRun | Exactly one Test Case in the configured Campaign mapped to approved `libraryTestCaseId`, with a non-null approved overall Test Case outcome (`outcomePath`) and compact `outcomeNotes`. |
| VECTR evidence | Attributable evidence exists | Corresponding per-tool Defense Tool outcomes are assigned; the overall Test Case outcome is verified against the verdict mapping; compact evidence summary added. |
| Human override | Operator changes shared Jira field | Orchestrator does not blindly overwrite; conflict policy applies. |
| Completion | Jira/VECTR writes succeed | Both systems read back correctly, including the VECTR overall outcome and per-tool outcomes, before ScenarioRun becomes `COMPLETE`; manual-review runs remain workflow-open. |

---

## 7. Feature Priority Matrix

| Feature | Priority | Phase | Notes |
|---------|----------|-------|-------|
| PowerShell repository/config scaffolding | P0 — Critical | Phase 0 | Foundation |
| Durable ScenarioRun state + idempotency | P0 — Critical | Phase 1 | Prevents duplicates |
| Jira service-account authentication | P0 — Critical | Phase 2 | Required for all machine Jira writes |
| Jira issue create/recover | P0 — Critical | Phase 2 | Reporting system of record |
| Exact-value `Blue team detection` decision engine | P0 — Critical | Phase 2/5 | Every deterministic verdict receives one approved value; unresolved cases remain unset only while in manual review |
| VECTR GraphQL auth/environment/campaign validation | P0 — Critical | Phase 3 | Technical record integrity |
| VECTR create/recover by library mapping | P0 — Critical | Phase 3 | Prevents duplicate/mismapped tests |
| Terminal assessment lifecycle discovery | P0 — Critical | Phase 4 | Includes integration-query settling and partial/failure monitoring |
| Scenario result/detection retrieval | P0 — Critical | Phase 4 | Primary per-scenario status and documented detection source |
| Findings retrieval + supporting evidence | P0 — Critical | Phase 4 | Prevention, attribution, consistency, VECTR evidence |
| End-to-end Jira + VECTR reporting | P0 — Critical | Phase 5 | Core product |
| Read-after-write verification | P0 — Critical | Phase 5 | Completion barrier |
| Jira Automation + manual-analysis transitions | P1 — High | Phase 5 | Field-driven Jira Automation handles substantive verdicts; orchestrator routes verdict-less manual-review cases to `Analyze` |
| Reconciliation worker | P1 — High | Phase 6 | Detects/repairs drift |
| Workstation outage catch-up | P1 — High | Phase 6 | Required for scheduled-task hosting |
| VECTR native overall Test Case outcome | P0 — Critical | Phase 3/5 | Every created Test Case receives the Blue Team/defense outcome derived from the verdict model |
| VECTR native per-tool outcomes | P1 — High | Phase 3/5 | Preserve AV/EDR/TheHive tool-specific evidence and support correct overall outcome derivation/override |
| Shadow mode / kill switch | P1 — High | Phase 7 | Safe rollout |
| Cymulate webhook ingestion | P3 — Conditional | future only | Only if Cymulate provides an official webhook mechanism |
| Web dashboard | P3 — Low | future | Not required for workflow integrity |

---

## 8. Build & Development Environment

### 8.1 Toolchain

- **Language:** PowerShell 7+
- **Host OS:** Windows 10/11 enterprise workstation
- **Scheduling:** Windows Task Scheduler
- **HTTP:** `Invoke-RestMethod` / `Invoke-WebRequest`
- **Testing:** Pester
- **Configuration:** `.psd1` + JSON
- **State:** SQLite preferred; Phase 0-approved atomic per-run JSON fallback permitted if SQLite cannot be approved
- **Logging:** JSON Lines + human-readable operational log
- **Source control:** Git
- **No persistent service:** scripts start, process, persist, and exit

### 8.2 Test Environment

Minimum non-production access:

- Jira test issue context in `CIBETHICAL` or approved equivalent.
- VECTR non-production/test Campaign.
- Cymulate API access with at least one known completed standing assessment.
- Representative DLP, CSIRT, and Use Case Generation scenario results where possible.
- Ability to simulate workstation restart and API failure.

### 8.3 Repository Structure

| Path | Purpose |
|------|---------|
| `scripts/Invoke-PurpleTeamOrchestrator.ps1` | Scheduled batch entry point |
| `modules/PurpleTeam.Common.psm1` | Shared configuration, HTTP, logging, validation helpers |
| `modules/PurpleTeam.Cymulate.psm1` | Read-only Cymulate adapter |
| `modules/PurpleTeam.Jira.psm1` | Jira adapter |
| `modules/PurpleTeam.Vectr.psm1` | VECTR adapter |
| `modules/PurpleTeam.State.psm1` | Durable state and idempotency |
| `config/` | Environment-specific discovery scope, default Jira/VECTR/evidence policy, and optional scenario-specific overrides |
| `state/` | Local durable runtime state |
| `logs/` | Structured operational logs |
| `tests/Unit/` | Pester unit tests |
| `tests/Integration/` | Explicit integration tests |
| `tests/EndToEnd/` | End-to-end and failure-injection tests |
| `docs/` | API contracts, mappings, runbooks, and UAT evidence |

---

## 9. Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Duplicate Jira/VECTR records after lost create response | High | Persist `run_id` before Jira POST; recover Jira by run ID; persist Jira-generated key before VECTR creation; deterministic VECTR names; unique local constraints. |
| Downstream logic assumes Jira key before POST | High | Jira key is nullable pre-create; prohibit local key generation; enforce Jira-create/recover gate before VECTR operations requiring the key. |
| Scenario `Not Detected` is confused with empty/not-evaluated detection | High | Preserve raw detection; exact normalizer; empty is `NOT_EVALUATED`; use policy-driven `NotApplicable` or manual review, never an inferred `NotDetected`. |
| Empty findings are misclassified as `NotDetected` | High | Findings absence never creates a detection result. Use explicit scenario detection plus approved evidence policy. |
| Scenario detection alone is insufficient for the organizational verdict | High | Resolve the required AV, EDR, and TheHive use-case evidence. Missing or contradictory required evidence enters `MANUAL_REVIEW_REQUIRED` and routes to `Analyze`. |
| An alternate Cymulate findings/search/detail API is accidentally introduced as an enrichment dependency | High | Explicit endpoint allowlist tests; reporting evidence is limited to approved launched-assessment/scenario APIs. |
| Integration results are still being queried/finalized | High | Normalize lifecycle; retain internal pending state; do not provision or finalize a result ticket; revisit later. |
| Scenario execution failure is misclassified as a security-control miss | High | Evaluate execution before prevention/detection; `FAILED -> AttackFailed`. |
| Finding cannot be deterministically assigned to a scenario result | High | Findings are supporting evidence; use `scenarioResultID` from `/scenarios` for identity; ambiguous evidence enters `MANUAL_REVIEW_REQUIRED`. |
| Findings or detailed evidence settle after parent completion | High | Measure tenant behavior; use bounded evidence-settling retries and persist `EVIDENCE_PENDING`. |
| Jira issue remains without a substantive `Blue team detection` verdict | Medium | Blank is permitted only while `MANUAL_REVIEW_REQUIRED` is active. Re-evaluate unresolved runs on later invocations; once evidence supports a verdict, the script must write/read back the correct approved value. Preserve deliberate human assignments. |
| Wrong Jira value or workflow transition | High | Exact allowlist, live option validation, decision tests, read-back, transition-by-name, Jira Automation idempotency. |
| Human edits are overwritten | Medium | Field ownership model + conflict detection. |
| Personal Jira account is used by automation | High | Dedicated service account; production validation; no fallback to personal credentials. |
| VECTR tags are polluted by automated tests | High | Absolute prohibition on VECTR tag reads/writes/reconciliation. |
| VECTR Test Case is created against the wrong Campaign/template | High | Preconfigured IDs + read-back verification of Campaign and `libraryTestCaseId`. |
| VECTR Test Case has no overall Blue Team/defense outcome | High | Derive a VECTR semantic outcome for every provisioned Test Case; use `outcomePath` at creation; manual-review/non-substantive cases use approved `TBD`; read back `outcome { id name path }`. |
| VECTR per-tool priority produces an overall outcome that contradicts the organizational verdict | High | Compare derived VECTR overall outcome to the verdict mapping; preserve per-tool outcomes but use an approved `overrideOutcome` + overall path when required. |
| Workstation sleep/reboot misses scheduled executions | Medium | Persisted checkpoint + overlap/lookback + missed-run recovery. |
| API throttling / temporary outage | Medium | Client throttling, bounded backoff, durable retries, no false completion. |
| Secrets are exposed in source/logs | High | Approved secret store, log redaction, non-interactive service identity. |
| Cymulate schema changes | High | Record reviewed API version, contract fixtures, preserve unknown values, and route schema drift to `MANUAL_REVIEW_REQUIRED` / Jira `Analyze`. |

---

## 10. References

- Supplied Cymulate OpenAPI Swagger (`swagger.json`), OpenAPI 3.1.1, Cymulate API version 2.0.62, reviewed August 18, 2026. This is the primary source for overlapping endpoint/schema contracts in this revision.
- Supplied Cymulate API Postman collection, retained as a secondary historical source where it does not conflict with the Swagger.
- Actual Cymulate API responses from the organization's environment, captured and sanitized during Phase 0.
- Official VECTR **Recording Defense Outcomes** documentation, including per-tool outcome mode and overall Test Case outcome behavior.
- Official VECTR GraphQL **Test Cases** documentation and schema reference for `CreateTestCaseDataInput`, `UpdateTestCaseDataInput`, `DefenseToolOutcomeInput`, `TestCase`, `Outcome`, `outcomes`, and `outcomesTree`.
- Actual VECTR schema/introspection and outcome tree from the organization's target VECTR environment, captured during Phase 0.
- Jira REST API documentation and live project field/workflow metadata.
- Existing Purple Team Jira workflow and `CIBETHICAL` project configuration.
- Existing manual VECTR operating practices. VECTR tags remain human-owned and out of orchestrator scope.

### 10.1 Swagger-Derived Detection Conclusions

1. `/v2/assessments/launched/{id}/scenarios` is the only documented result endpoint that directly combines `scenarioResultID`, `scenarioID`, scenario execution fields, and a defined scenario-level detection value.
2. The documented scenario detection domain is `Detected`, `Not Detected`, or empty/not evaluated. Unknown strings must be preserved and treated as contract drift.
3. Parent assessment lifecycle explicitly contains integration-query states; detection is not finalized while those states are active.
4. The launched findings schema is rich evidence but lacks a top-level `scenarioResultID` and `scenarioID`.
5. Product decision: alternate Cymulate findings/search/detail APIs are out of scope; the reporting framework uses only the approved launched-assessment/scenario endpoints.
6. The organizational Jira verdict depends on the combined AV, EDR, and TheHive evidence contract proven from `GET /v2/assessments/launched/{id}/findings`; missing or contradictory required evidence enters `MANUAL_REVIEW_REQUIRED` and routes Jira to `Analyze` without fabricating a verdict.

---

# PART II: IMPLEMENTATION PHASES

## 11. How To Use Part II With Codex

Part II breaks Part I into discrete, dependency-ordered tasks sized for iterative Codex sessions.

### Workflow

1. Start a Codex session with this document and one task ID, for example: `Implement P2-T3`.
2. Codex implements only that task and its acceptance tests.
3. Run Pester/build/test.
4. Review the output against the acceptance criteria.
5. If the criteria pass, commit before starting the next dependent task.
6. If the criteria fail, fix the current task before proceeding.

Do not ask Codex to implement all phases in one pass.

### Complexity Ratings

- **S (Small):** one focused script/config/test change.
- **M (Medium):** one module capability with mocks and validation.
- **L (Large):** multi-function adapter flow or stateful integration.
- **XL (Extra Large):** cross-system end-to-end workflow requiring multiple sessions.

### Session Constraints

- PowerShell 7 only for production code.
- Pester tests accompany every behavior change.
- No production Cymulate mutation calls.
- No VECTR tag reads/writes.
- No personal Jira credentials.
- No hard-coded Jira custom-field IDs or transition IDs.
- No hard-coded VECTR outcome IDs unless Phase 0 explicitly approves them.
- No silent fallback that can create duplicate records.
- Do not infer `NotDetected` from absence of finding records or from an empty scenario detection value.
- Treat the launched-assessment scenario result `detection` field as the primary documented scenario-level detection source and preserve the raw value.
- Evaluate assessment lifecycle and `scenario_execution` before prevention or detection.
- Keep prevention, AV outcome, EDR outcome, and TheHive use-case outcome as separate normalized dimensions.
- Keep transient/pending integration processing in durable local state; use `TestingInProgress` only in a separately approved pre-provisioned ticket workflow. Use `MANUAL_REVIEW_REQUIRED` plus Jira `Analyze` for terminal ambiguity or evidence conflict; leave `Blue team detection` unset until a substantive verdict is supported.
- Ensure every deterministic verdict uses one exact allowed `Blue team detection` value. Manual-review issues may temporarily leave the field unset and must be re-evaluated until a substantive verdict is assigned or a human-authoritative value is recorded.
- Never implement an endpoint from an unbound component schema.
- Never generate or predict Jira issue numbers locally; `CIBETHICAL-####` must come from Jira create/recovery.
- Persist `run_id` before Jira POST and persist returned/recovered `jira_key` before VECTR creation.
- No `COMPLETE` state until Jira and VECTR automation-owned state is read back and verified.

---

## Phase 0: Contract Discovery & Project Scaffolding

**Goal:** Prove the live Cymulate/Jira/VECTR contract and create the PowerShell repository without production automation writes.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P0-T1 | Initialize repository, module/script/test folders, `.gitignore`, README, sample config. | Top-level + `config/` | Import all empty modules; sample config parses; no secrets committed. | S |
| P0-T2 | Create configuration loader/validator for Cymulate auth, API version, Jira, VECTR, state, launched-assessment discovery window/scope, default evidence policy, optional scenario overrides, and kill switch. | `modules/PurpleTeam.Common.psm1`, `config/settings.example.psd1` | Invalid/missing required config fails before any API call; no scenario-ID allowlist is required. | M |
| P0-T3 | Capture the Swagger/live contract for launched assessments, lifecycle statuses, scenario results, scenario detection, launched-assessment findings, scenario metadata, pagination, and rate limiting. Explicitly document the reporting endpoint allowlist and that alternate Cymulate findings/search/detail APIs are excluded. | `docs/cymulate-contract.md`, sanitized fixtures | Exact request/response fields, status aliases, paging shapes, reviewed API version, and reporting endpoint allowlist are documented. | L |
| P0-T4 | Prove scenario-status normalization and the three documented scenario-detection cases: `Detected`, `Not Detected`, and empty/not evaluated. Validate when detection becomes stable relative to integration-query statuses. | `docs/cymulate-detection-contract.md`, fixtures | Each case has a real/sanitized fixture; unknown value test routes to contract review; no empty value becomes `NotDetected`. | L |
| P0-T5 | Prove the AV, EDR, and TheHive evidence contract from `GET /v2/assessments/launched/{id}/findings`, including prevention, detection/observation, expected use-case fire/non-fire, correlation, multiple findings, zero findings, and settling behavior. | `docs/cymulate-findings-correlation.md`, `docs/cymulate-outcome-mapping.md` | Exact `/findings` JSON fields, semantics, correlation rules, and the AV+EDR+TheHive verdict matrix are documented from real/sanitized fixtures. Missing/contradictory required evidence is explicitly mapped to `MANUAL_REVIEW_REQUIRED` and Jira `Analyze` with no fabricated verdict. | XL |
| P0-T6 | Capture Jira contract: service account, custom fields/options, create-time field support, workflow transitions, Jira Automation rules, assignment, comments, and human ownership. | `docs/jira-mapping.md` | All eight active values round-trip; deterministic issues receive the correct value; manual-review issues may be created with the field unset; `Analyze` transition is validated. | M |
| P0-T7 | Capture the VECTR Test Case outcome contract and define default/assessment-level Jira/VECTR/evidence mappings plus the optional scenario-override format. Query the actual `outcomes`/`outcomesTree`, prove `outcomePath` creation, `defenseToolOutcomes`, `dataVer`, `overrideOutcome`, read-back, and the post-create `testCase.update` behavior needed when a result changes. Prove the mapping with representative DLP, CSIRT, and Use Case Generation results. | `docs/vectr-mapping.md`, `docs/vectr-outcome-mapping.md`, `config/scenario-overrides.example.json` | Representative Jira verdict/evidence cases map to approved VECTR overall paths; every created non-prod Test Case has a verified overall outcome; per-tool outcomes and override behavior are proven; zero tag mutation; no scenario-ID allowlist is required. | XL |

**Exit gate:** No unresolved detection-source, evidence-requirement, Jira-value, or correlation assumption remains for the pilot scenarios.

---

## Phase 1: Durable State, Idempotency & Audit

**Goal:** Make every future API operation restart-safe before adding write integrations.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P1-T1 | Implement local state initialization/schema for discovery checkpoint, ScenarioRun, retries, processed keys, audit events. | `modules/PurpleTeam.State.psm1` | State survives PowerShell process restart; schema can initialize on clean workstation. | L |
| P1-T2 | Implement external idempotency key `<assessmentId>:<scenarioResultID>` with uniqueness enforcement. | `PurpleTeam.State.psm1`, tests | 100 duplicate insert attempts result in exactly one ScenarioRun. | M |
| P1-T3 | Implement ScenarioRun state machine and legal transitions. | `PurpleTeam.State.psm1` | Illegal transitions fail; Pester covers every state transition. | M |
| P1-T4 | Implement structured audit events and JSONL operational logging with secret redaction. | `PurpleTeam.Common.psm1`, `PurpleTeam.State.psm1` | Every state-changing test emits an audit record; auth headers/tokens never appear. | M |
| P1-T5 | Implement named mutex/process lock and clean abandoned-lock recovery. | `PurpleTeam.Common.psm1` | Two concurrent orchestrator processes cannot mutate state simultaneously. | S |

**Exit gate:** Restart/duplicate/concurrency tests pass with no external APIs.

---

## Phase 2: Jira Adapter

**Goal:** Prove machine-owned Jira operations using the dedicated service account.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P2-T1 | Implement Jira service-account authentication and connectivity test. | `modules/PurpleTeam.Jira.psm1` | Request is visibly made as approved automation identity; 401/403 classified explicitly. | M |
| P2-T2 | Implement Jira field metadata/config validation including exact `Blue team detection` values. | `PurpleTeam.Jira.psm1`, tests | Any value outside exact allowlist is rejected locally; custom-field ID is config-driven. | M |
| P2-T3 | Implement create issue using persisted `run_id`/Cymulate correlation; capture Jira-generated issue key from POST response and persist it. | `PurpleTeam.Jira.psm1`, `PurpleTeam.State.psm1` | Before POST, `jira_key` is null. After successful POST, returned `CIBETHICAL-####` is stored and read back exactly. No local Jira-key generation exists. | L |
| P2-T4 | Implement lost-create-response recovery by `Purple Run ID` (`run_id`) plus Cymulate validation. | `PurpleTeam.Jira.psm1`, mocks | Simulated Jira commit + lost POST response across 100 trials recovers the generated Jira key and creates zero duplicate issues. | L |
| P2-T5 | Implement comments and automation attribution. | `PurpleTeam.Jira.psm1` | Automated comment appears under service account; no operator credential used. | S |
| P2-T6 | Implement `Blue team detection` update/read-back and candidate mapping interface. | `PurpleTeam.Jira.psm1`, config, tests | All eight active `Blue team detection` values round-trip unchanged; null/unset is accepted only for a run in `MANUAL_REVIEW_REQUIRED`. | M |
| P2-T7 | Implement transition discovery by name and approved assignment behavior. | `PurpleTeam.Jira.psm1` | No hard-coded transition ID; invalid transition fails without changing issue. | M |
| P2-T8 | Implement human-change conflict protection for shared Jira fields. | `PurpleTeam.Jira.psm1`, state tests | Human-modified shared value is not blindly overwritten. | M |

**Exit gate:** Jira adapter has zero duplicate issues and zero use of personal operator credentials.

---

## Phase 3: VECTR GraphQL Adapter

**Goal:** Create and update technical VECTR records with a verified overall Blue Team/Test Case outcome and attributable per-tool outcomes, without touching tags.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P3-T1 | Implement GraphQL client, VEC1 authentication, top-level `errors` handling, variables. | `modules/PurpleTeam.Vectr.psm1` | HTTP 200 + GraphQL error is treated as failure. | M |
| P3-T2 | Implement cursor pagination helper and Environment/Assessment/Campaign validation. | `PurpleTeam.Vectr.psm1` | Multi-page fixture returns complete set; wrong Campaign fails startup validation. | M |
| P3-T3 | Implement Content Library lookup by `libraryTestCaseId`. | `PurpleTeam.Vectr.psm1` | Pilot template resolves exactly; deprecated `templateId` not used as primary key. | M |
| P3-T4 | Implement VECTR outcome catalog discovery using `outcomes` / `outcomesTree`, including `Outcome.id`, `name`, `path`, `userSelectable`, and related metadata used for mapping validation. | `PurpleTeam.Vectr.psm1`, config | Configured semantic targets resolve to exactly one approved selectable path/ID; missing/ambiguous mappings fail before Test Case creation. | M |
| P3-T5 | Implement local Test Case creation by library ID **after `jira_key` exists**, using `[CIBETHICAL-####] ...` deterministic name and `clientId=run_id`, and supplying the required overall `outcomePath`, `outcomeNotes`, `dataVer` policy, per-tool outcomes when available, and `overrideOutcome` when required. | `PurpleTeam.Vectr.psm1` | Creation is rejected when `jira_key` or the desired VECTR overall outcome is unresolved. After Jira creation/recovery, one non-prod Test Case is created in the configured Campaign with the returned Jira key in its name and a verified non-null outcome. | XL |
| P3-T6 | Implement lost-create-response recovery by deterministic identity + Campaign/library verification. | `PurpleTeam.Vectr.psm1`, mocks | 100 lost-response tests create zero duplicates and recover the existing Test Case outcome state. | L |
| P3-T7 | Implement `outcomeNotes` plus overall outcome read-back verification (`outcome { id name path }`, `overrideOutcome`, `dataVer`). | `PurpleTeam.Vectr.psm1` | Approved compact evidence summary and expected overall outcome round-trip correctly. | M |
| P3-T8 | Implement Defense Tool discovery and per-tool `DefenseToolOutcomeInput` assignment (`defenseToolId` + `outcomeId`). | `PurpleTeam.Vectr.psm1` | Only evidence-attributable Defense Tools receive outcomes; AV/EDR/TheHive tool outcomes read back exactly. | L |
| P3-T9 | Implement overall-outcome reconciliation and override logic for per-tool mode. | `PurpleTeam.Vectr.psm1`, tests | When VECTR's priority-derived outcome matches the desired verdict mapping, no override is used; when it conflicts, the approved overall outcome is applied with `overrideOutcome=true` and verified. | L |
| P3-T10 | Implement tested post-create outcome update path for `TBD`/manual-review cases that later become substantive, plus approved historical timestamp/timeline strategy. | `PurpleTeam.Vectr.psm1` | A synthetic `TBD` case can be updated to its final approved outcome and read back; known historical timestamps read back unchanged; no current-time fabrication. | L |
| P3-T11 | Add negative tag and deprecated-outcome tests. | `tests/Unit/Vectr.Tags.Tests.ps1`, outcome tests | Production module performs zero VECTR tag mutation and never uses deprecated create-time `outcome` as the primary outcome writer. | S |

**Exit gate:** VECTR adapter creates/recovers technical records with the correct verified overall Test Case outcome, preserves attributable per-tool outcomes, supports approved outcome updates, performs zero tag operations, and creates zero duplicates.

---

## Phase 4: Cymulate Read Adapter

**Goal:** Reliably discover terminal scenario results, track integration processing, and collect documented scenario detection plus supporting evidence with no Cymulate data mutations.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P4-T1 | Implement `x-token` client, optional configured OAuth2 client-credentials path, rate limiter, and transient retry policy. | `modules/PurpleTeam.Cymulate.psm1` | Synthetic burst stays under configured rate; 429/5xx retried safely; auth modes are explicit. | M |
| P4-T2 | Implement one-shot launched-assessment discovery with terminal/non-terminal status normalization, UTC checkpoint, and overlap/lookback. | `PurpleTeam.Cymulate.psm1` | `queryingIntegration` cases persist as pending; completed/partial/failure cases are revisited safely; no wait/sleep polling loop. | L |
| P4-T3 | Implement launched-assessment pagination and detail retrieval. | `PurpleTeam.Cymulate.psm1` | All fixture pages returned exactly once; raw status, integrations, and timestamps retained. | M |
| P4-T4 | Implement scenario-result retrieval/pagination and exact detection normalizer. | `PurpleTeam.Cymulate.psm1` | All IDs returned; `Detected`, `Not Detected`, empty, and unknown fixtures map to `DETECTED`, `NOT_DETECTED`, `NOT_EVALUATED`, and `UNKNOWN`. | L |
| P4-T5 | Implement an environment-validated scenario status/statusReason normalizer for execution and prevention. | `PurpleTeam.Cymulate.psm1` | Not-started, running, success, partial, failed, prevented, and unknown fixtures behave exactly as approved. | L |
| P4-T6 | Implement launched findings retrieval/pagination once per parent assessment per cycle. | `PurpleTeam.Cymulate.psm1` | All pages up to the endpoint's 100-row page contract are retrieved; zero findings remains evidence-neutral. | M |
| P4-T7 | Implement supporting finding correlation and prevention/control evidence extraction. | `PurpleTeam.Cymulate.psm1` | Known evidence attaches correctly; ambiguous evidence is marked `AMBIGUOUS`; no join relies on scenario name alone unless fixture-approved. | L |
| P4-T8 | Implement AV/EDR/TheHive evidence extraction exclusively from fields returned by `GET /v2/assessments/launched/{id}/findings` whose semantics have been validated against actual Cymulate API responses from the organization's environment. | `PurpleTeam.Cymulate.psm1` | Proven fields normalize to the separate AV, EDR, and TheHive dimensions; missing/unproven fields remain `UNKNOWN`/`NOT_AVAILABLE` and never trigger another Cymulate findings endpoint. | L |
| P4-T9 | Implement evidence consistency resolver and scenario metadata cache. | `PurpleTeam.Cymulate.psm1` | Material contradiction yields `EVIDENCE_CONFLICT`; repeated metadata lookup hits cache. | M |
| P4-T10 | Add guard tests proving no normal-path Cymulate mutation, alternate findings/search/detail request, or non-allowlisted reporting endpoint can be invoked. | tests | POST/PATCH/PUT/DELETE reporting calls fail; every non-allowlisted findings/search/detail request fails before HTTP; only approved GET reporting endpoints are reachable. | S |

**Exit gate:** A real pilot assessment can be read into the normalized result using only the approved launched-assessment/scenario GET endpoints. No alternate Cymulate findings/search/detail API is invoked.

---

## Phase 5: End-to-End Orchestration & Result Reporting

**Goal:** Convert one normalized Cymulate scenario result into verified Jira + VECTR records, assigning an exact substantive Jira result whenever the evidence supports one and routing unresolved cases to manual analysis without fabricating a verdict.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P5-T1 | Implement ScenarioRun creation for every scenario result returned by an in-scope terminal launched assessment. Apply configured defaults/assessment-level mappings and then any optional scenario override. | orchestrator/services | Every discovered scenario result creates exactly one run; absence of a scenario override never suppresses processing. | M |
| P5-T2 | Implement strict creation sequence: persist terminal run → Jira create/recover with the substantive verdict when available, otherwise manual-review state with the field unset → persist Jira key → derive required VECTR overall outcome/per-tool outcomes from the same normalized result → VECTR create/recover with outcome → write VECTR ID back. | orchestrator + adapters | No normal-path ticket is provisioned from a non-terminal assessment; VECTR creation is impossible before `jira_key` or before a VECTR semantic outcome target exists; deterministic Jira cases and `TBD` manual-review cases receive the correct VECTR outcome; lost response creates no duplicate. | XL |
| P5-T3 | Implement the precedence-based terminal decision engine. | config + orchestrator | Tests cover `NotStarted`, `AttackFailed`, `PartialTesting`, `Blocked`, `Detected`, confirmed `NotDetected`, `NotApplicable`, and unresolved/conflicting AV/EDR/TheHive evidence. Deterministic cases return one approved value; unresolved terminal cases return no verdict and enter `MANUAL_REVIEW_REQUIRED`; non-terminal observations provision nothing. | XL |
| P5-T4 | Implement VECTR overall Test Case outcome rendering, `outcomeNotes`, attributable per-tool outcomes, and override selection from the same normalized result used by Jira. | orchestrator + VECTR | Every provisioned VECTR Test Case has the approved overall outcome; per-tool outcomes preserve AV/EDR/TheHive evidence; `NotDetected` does not get incorrectly promoted by per-tool priority; unresolved evidence uses approved `TBD`; no VECTR tag changes. | XL |
| P5-T5 | Implement Jira Automation integration/transition expectations and manual-analysis route. | orchestrator + Jira/config | `MANUAL_REVIEW_REQUIRED`, `AttackFailed`, and `PartialTesting` route to `Analyze`; substantive deterministic results route according to policy, normally `Validation`; no auto-bypass to `Done`. | M |
| P5-T6 | Implement completion barrier, evidence-pending policy, and read-after-write verification. | orchestrator + state | Pending integrations do not provision/finalize result records; Jira state and VECTR overall/per-tool outcome state read back for terminal runs; no deterministic run completes with a missing/mismatched VECTR outcome; manual-review runs use `TBD` and remain workflow-open. | L |
| P5-T7 | Implement parent assessment containing multiple scenario results and repeated scenario templates/test points. | EndToEnd tests | An in-scope assessment with N scenario results produces exactly N tracked ScenarioRuns and, when downstream mappings are resolvable, N Jira + N VECTR record pairs; ambiguous finding correlation cannot merge runs. | XL |

**Exit gate:** Deterministic cases complete without manual reporting, and every non-deterministic terminal case is safely routed to `Analyze` without a fabricated `Blue team detection` value.

---

## Phase 6: Reconciliation & Operational Resilience

**Goal:** Make the workstation-hosted automation reliable under outages, missed schedules, and partial failures.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P6-T1 | Implement incomplete-run reconciliation on every invocation. | `scripts/Invoke-PurpleTeamReconciliation.ps1`, state/adapters | Induced stale Jira/VECTR automation-owned field is detected. | L |
| P6-T2 | Implement safe automatic repair + conflict classification. | reconciliation | Safe drift repaired; human-modified shared Jira field becomes conflict instead. | L |
| P6-T3 | Implement workstation-offline catch-up and discovery-checkpoint advancement rules. | orchestrator/state | After missed Task Scheduler executions, the next scheduled invocation processes every missed reportable terminal result once and then exits. | M |
| P6-T4 | Implement operational metrics and health summary. | Common/State | Counts include runs, sync failures, retries, findings failures, duplicate suppressions; VECTR tag mutation metric remains zero. | M |
| P6-T5 | Implement admin commands: show run, retry Jira, retry VECTR, reconcile run, test config. | `scripts/Repair-PurpleTeamRun.ps1`, `Test-PurpleTeamConfiguration.ps1` | Operator can inspect/repair a synthetic failed run without editing state files manually. | M |
| P6-T6 | Implement Task Scheduler installer/uninstaller and recommended settings. | `scripts/Install-PurpleTeamScheduledTask.ps1` | Fresh workstation can register task; task runs non-interactively and prevents overlap. | M |

**Exit gate:** Known outages/mismatches are detected; no run silently disappears or falsely completes.

---

## Phase 7: Shadow Mode, UAT & Controlled Production Pilot

**Goal:** Prove the automation against the existing human process before broadening production discovery scope.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P7-T1 | Implement `AUTOMATION_WRITE_ENABLED=false` shadow mode. | config/orchestrator | Reads/calculates/logs desired changes; Jira/VECTR receive zero mutations. | M |
| P7-T2 | Shadow one representative DLP scenario result from an in-scope launched assessment. | UAT docs | Desired Jira/VECTR state matches human-created reference record. | M |
| P7-T3 | Shadow one representative CSIRT scenario result from an in-scope launched assessment. | UAT docs | Same as above; scenario detection, integration lifecycle, and approved evidence requirements are verified. | M |
| P7-T4 | Shadow one representative Use Case Generation scenario result from an in-scope launched assessment. | UAT docs | Same as above; defense evidence mapping verified. | M |
| P7-T5 | Run failure-injection and Jira/VECTR outcome-matrix UAT suite. | `tests/EndToEnd/` | Lost responses, 401/403, 429/5xx, integration-query pending with no premature ticket, `Detected`, confirmed and unresolved `NotDetected` evidence paths, empty detection, missing integration/use case, unknown value, ambiguous findings, execution failure, AV/EDR block, evidence conflict, VECTR per-tool priority/overall override, `TBD`→final VECTR update, workstation restart, and human override all pass. | L |
| P7-T6 | Enable writes for a narrowly defined pilot launched-assessment scope (for example approved assessment/template IDs, tags, names, or environments). Process every scenario result inside that scope. | production config | ≥20 successful pilot ScenarioRuns; 0 scenario results silently skipped because an override is absent; 0 duplicate Jira; 0 duplicate VECTR; 0 VECTR tag mutations. | XL |
| P7-T7 | Production readiness review. | `docs/production-readiness.md` | All P0/P1 reliability targets met; rollback/credential rotation/operator runbook approved. | M |

**Exit gate:** Controlled pilot has no unexplained discrepancy, duplicate, lost execution, incorrect Jira outcome, or VECTR tag mutation.

---

## Phase 8: Production Scope Expansion

**Goal:** Expand the proven assessment-driven workflow into normal production operation without introducing a scenario-ID allowlist.

| ID | Task | Files | Acceptance Criteria | Est. |
|----|------|-------|---------------------|------|
| P8-T1 | Validate the production launched-assessment discovery query, lookback/overlap window, and any approved assessment-level filters. | config validator | The query returns the intended assessment population; no scenario-ID allowlist is required. | M |
| P8-T2 | Expand from pilot assessment/template scope to the remaining approved production discovery scope in controlled batches. | production config | Every scenario result inside each enabled assessment scope is discovered and tracked exactly once. | M |
| P8-T3 | Add optional scenario override entries only where the default or assessment-level evidence/Jira/VECTR mapping is insufficient. | `config/scenario-overrides.json` | Overrides change only the specified mapping/policy; scenarios with no override continue to process normally. | M |
| P8-T4 | Run a full configured-lookback shadow/catch-up validation before each material scope expansion, then enable writes. | UAT/operations | No scenario result within the target assessment scope is silently skipped; duplicate suppression remains correct. | M |
| P8-T5 | Produce monthly operational validation checklist. | `docs/monthly-operations.md` | Operators can confirm discovery coverage, schedules, sync failures, credentials, unmatched findings, optional override drift, and scenario/API drift. | S |

**Exit gate:** The approved launched-assessment discovery scope is processed comprehensively; optional scenario overrides exist only where defaults cannot resolve policy/mapping, and no scenario-ID allowlist exists.

---

## Phase Summary & Dependency Map

| Phase | Name | Tasks | Depends On | Primary Outcome |
|-------|------|-------|------------|-----------------|
| P0 | Contract Discovery & Scaffolding | 7 | — | Proven live API/workflow mappings |
| P1 | State, Idempotency & Audit | 5 | P0 | Restart-safe local orchestration |
| P2 | Jira Adapter | 8 | P0, P1 | Reliable service-account Jira writes |
| P3 | VECTR Adapter | 11 | P0, P1 | Reliable Test Case creation with verified overall/per-tool outcomes and zero tag use |
| P4 | Cymulate Read Adapter | 10 | P0, P1 | Lifecycle + scenario detection + supporting evidence normalization |
| P5 | End-to-End Orchestration | 7 | P2, P3, P4 | Scenario result → verified Jira/VECTR |
| P6 | Resilience & Operations | 6 | P5 | Reconciliation, catch-up, operational controls |
| P7 | Shadow/UAT/Pilot | 7 | P6 | Production confidence |
| P8 | Production Scope Expansion | 5 | P7 | Broad assessment-driven rollout |

**Total: 66 tasks across 9 phases.**

---

## Code Conventions & Constraints

### PowerShell

- PowerShell 7+.
- `Set-StrictMode -Version Latest`.
- Production functions use approved PowerShell verb-noun naming.
- External API failures use terminating errors and explicit `try/catch`.
- Do not rely on `$?` for workflow correctness.
- Return structured objects, not formatted strings, from module functions.
- Business logic must not call Jira/VECTR/Cymulate HTTP endpoints directly.
- All vendor HTTP/GraphQL calls remain inside their adapter modules.
- No secrets in logs.
- No personal Jira credentials.
- No VECTR tag operations.
- No Cymulate mutations.
- No long-running Cymulate polling/wait loop; every Task Scheduler invocation performs a finite discovery pass and exits.
- All timestamps stored internally in UTC.

### Testing

- Pester for unit/contract/integration/end-to-end tests.
- Unit tests use mocked HTTP/GraphQL responses.
- Real API tests run only under explicit integration/UAT switches.
- Every create mutation has a lost-response/idempotency test.
- Every critical write has a read-back verification test.
- Every human-owned/shared field has a conflict test.
- The complete Jira `Blue team detection` and VECTR overall/per-tool outcome matrix has deterministic Pester tests with provenance assertions.
- Contract tests prove that unbound component schemas cannot create callable client methods.
- Detection parser tests preserve unknown raw values and never coerce empty to `NotDetected`.

### Reliability Targets

```text
Duplicate Jira issues:                         0
Duplicate VECTR Test Cases:                    0
Completed runs with unresolved sync mismatch: 0
Known reconciliation mismatch detection:      100%
External mutation audit coverage:              100%
Automation-created VECTR tags:                0
Jira writes using personal operator account:  0
Deterministic Jira issues with blank `Blue team detection`: 0
NotDetected values inferred from empty data:       0
Unknown detection values silently coerced:        0
Unsupported/unbound Cymulate endpoint calls:      0
Automated Jira outcomes without required evidence:         0
Provisioned VECTR Test Cases with missing overall outcome:   0
Completed runs with incorrect VECTR overall outcome:         0
Cymulate data-mutation calls from normal path:     0
Long-running polling loops/resident pollers:       0
```

---

## Future Roadmap

Candidates only after Version 2.0 workflow integrity is proven:

- Server-hosted orchestrator instead of workstation Task Scheduler.
- Optional event-driven Cymulate webhook ingestion only if Cymulate introduces an officially supported webhook mechanism and the organization elects to use it. Scheduled Task execution remains the baseline design otherwise.
- Central multi-workstation orchestration.
- Web operational dashboard.
- Optional UI for discovery-scope and scenario-override management.
- Additional stakeholder SLA/report generation.
- Optional Cymulate assessment launch/schedule management under a separately approved write-capable design.
