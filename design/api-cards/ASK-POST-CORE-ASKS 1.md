# ASK-POST-CORE-ASKS - POST /core-asks

| Field | Value |
|---|---|
| API ID | ASK-POST-CORE-ASKS |
| Operation | POST /core-asks - operationId createCoreAsk (proposed) |
| Module | ASK |
| Purpose | Create a new Core ASK record in draft or submitted state, routing to the appropriate workflow status based on buttonValue and the caller's leadership group membership. |
| Complexity | Complex - SUBMIT action changes workflow state, creates a human task, branches routing on leadership group membership, and writes multiple aggregates (Ask, AskVersion, WorkflowTask, WorkflowDecision, WorkflowHistory, OutboxMessage, Comment, Audit) atomically. |
| Sources | US-ASK-001; ADO 1857; FR1; FR2; FR3; FR4; FR5; FR6; FR7; FR8; FR9; FR10; FR11; FR12; BR1–BR26; AC1–AC7 |
| Standards | ARCH 001; ARCH 002; ARCH 003; ARCH 004; DATA 001; DATA 003; DATA 004; API 001; API 002; SEC 001; SEC 002; SEC 003; WF 001; WF 003; REL 001; REL 002; OBS 001; TEST 001 |
| Status | Ready for Review |

## Contract summary

| Row | Detail |
|---|---|
| Path and query | POST /core-asks — no path or query parameters |
| Request schema | CreateCoreAskRequest: `coreAskDetails` object (FR10), `comment` string (FR8), `documents` file-upload array (FR9, Assumption 3), `buttonValue` enum ["Save & Exit", "SUBMIT", "Exit"] (BR23) |
| Response schema | CreateCoreAskResponse: `askId`, `askDetailId`, `taskId`, `statusId`, `assigneeId`, `commentId` (nullable), `auditId`, `version` (integer, 1 for new), `coreAskDetails` echo (FR7, FR12) |
| Success status | TBD (OI-7) — spec states 200 or 201 but does not confirm which applies for a create operation |
| Idempotency | TBD (OI-8) — idempotency key requirement and scope not specified in spec |
| Concurrency | Not specified in spec |

## Authorization

| Row | Detail |
|---|---|
| Authentication | Caller must be authenticated per SEC 001; mechanism TBD (OI-4) |
| Permission | Caller must hold "Create New ASK" permission (spec Business Constraints); exact permission code TBD (OI-6) |
| Role condition | Leadership group membership is resolved server-side from the authenticated caller's profile and must not be supplied in the request body (SEC 003, BR25, AC3) |
| Record scope | No existing record scope — this is a create operation |
| Workflow scope | Not applicable at entry; workflow state is set by this operation |
| Audit | Creation event recorded as auditId in response (FR7); governed by DATA 004 and OBS 001 |

## Processing flow

1. Authenticate caller and verify "Create New ASK" permission; return 403 if absent (SEC 001, SEC 002, spec Error Behaviour).
2. If `buttonValue` is "Exit", return success immediately without persisting any data (FR6, BR23, AC4).
3. Validate all required fields and character-length constraints (BR1–BR5, BR8, BR10, BR15, BR16, BR18, BR20–BR22, AC6); return 400 listing missing or invalid fields.
4. Validate reference data IDs (`dppGroupId`, `needReasonId`, `generalSpecialityNeedId`, `levelNeedId`, `rolePostingId` when required) are active via reference data services (FR3, BR2, BR17, spec Dependencies); return 400 if any ID is inactive or unknown.
5. Validate `employeeId` via employee service and derive `outgoingResource` as read-only (FR11, BR14); return 400 if `employeeId` is invalid.
6. Apply conditional field clearing rules: if `needReasonId` is the first-option value, clear `outgoingResource`, `employeeId`, `projectedStartDate`, and `endDate` (BR6, OI-3); if `generalSpecialityNeedId` is the first-option value, clear `generalSpecialityNeedComment` (BR7, OI-3).
7. Validate date and FTE business constraints (BR11–BR13, BR16, AC5, AC7); return 422 for date ordering or FTE violations.
8. Determine target `statusId`: if `buttonValue` is "Save & Exit" set status 123 In Progress (BR26, AC1); if "SUBMIT" and caller is not in leadership group set status 127 PPL Review (BR24, AC2); if "SUBMIT" and caller is in leadership group set status 145 DPP Ops Review (BR25, AC3). Leadership group membership sourced from caller's server-side profile (OI-5).
9. Atomically persist Ask, AskVersion, WorkflowTask, WorkflowDecision, WorkflowHistory, OutboxMessage, Comment (if provided), Audit, and Attachments (if provided) — transaction scope TBD (OI-9); governed by ARCH 003, DATA 001, REL 001.
10. Return CreateCoreAskResponse with all created identifiers, `statusId`, `version` = 1, and echoed `coreAskDetails` (FR7, FR12).

## Business and workflow rules

| Rule ID | Condition | Result |
|---|---|---|
| BR1 | `coreAskName` is absent or exceeds 200 characters | 400 validation error |
| BR2 | `dppGroupId` is absent or not an active option | 400 validation error |
| BR3 | `needReasonId` is absent | 400 validation error |
| BR4 | `generalSpecialityNeedId` is absent | 400 validation error |
| BR5 | `levelNeedId` is absent | 400 validation error |
| BR6 | `needReasonId` equals first-option value (OI-3) | Clear `outgoingResource`, `employeeId`, `projectedStartDate`, `endDate` |
| BR7 | `generalSpecialityNeedId` equals first-option value (OI-3) | Clear `generalSpecialityNeedComment` |
| BR8 | `generalSpecialityNeedId` is not the first option and `generalSpecialityNeedComment` is absent | 400 validation error |
| BR9 | `pml` is provided and exceeds 99 characters | 400 validation error; new-version-only semantics TBD (OI-10) |
| BR10 | `projectedStartDate` is absent | 400 validation error |
| BR11 | `endDate` is absent and `needReasonId` is not the retirement value (OI-4) | 400 validation error |
| BR12 | `endDate` is not after `projectedStartDate` | 422 business rule violation |
| BR13 | `endDate` is earlier than today | 422 business rule violation |
| BR14 | `outgoingResource` is supplied in request body | Ignored; value derived from `employeeId` via employee service |
| BR15 | `headCountAmount` is absent | 400 validation error |
| BR16 | `fteAmount` is absent or exceeds `headCountAmount` | 400 if absent; 422 if exceeds (AC5) |
| BR17 | `levelNeedId` is in the top-3 set (OI-2) and `rolePostingId` is absent | 400 validation error |
| BR18 | `titlingCategory` is absent; new-version-only semantics TBD (OI-10) | 400 validation error |
| BR19 | `transitionalCoach` is provided and exceeds 99 characters; new-version-only semantics TBD (OI-10) | 400 validation error |
| BR20 | `roleSummary` is absent | 400 validation error |
| BR21 | `roleResponsibility` is absent | 400 validation error |
| BR22 | `roleQualification` is absent | 400 validation error |
| BR23 | `buttonValue` is "Exit" | No data persisted; success response returned |
| BR24 | `buttonValue` is "SUBMIT" and caller is not in leadership group | Route to status 127 PPL Review |
| BR25 | `buttonValue` is "SUBMIT" and caller is in leadership group | Route to status 145 DPP Ops Review |
| BR26 | `buttonValue` is "Save & Exit" | Route to status 123 In Progress |

## Data impact

| Operation | Entity or table | Purpose |
|---|---|---|
| INSERT | Ask | New Core ASK master record (FR1, FR4, FR5) |
| INSERT | AskVersion | Version 1 detail record with all coreAskDetails fields (FR10, FR12) |
| INSERT | WorkflowTask | Human task created and assigned per routing outcome (FR7) |
| INSERT | WorkflowDecision | Records the buttonValue routing decision (WF 001) |
| INSERT | WorkflowHistory | Initial state transition entry (WF 003) |
| INSERT | OutboxMessage | Outbox event for downstream consumers (ARCH 004, REL 002) |
| INSERT | Comment | Persisted when `comment` is provided in request (FR8) |
| INSERT | Audit | Creation audit record; auditId returned in response (FR7, DATA 004) |
| INSERT | Attachment | One record per entry in `documents` array when provided (FR9) |
| READ | Reference data | Validate dppGroupId, needReasonId, generalSpecialityNeedId, levelNeedId, rolePostingId (FR3) |
| READ | Employee | Validate employeeId and derive outgoingResource (FR11, BR14) |

## Errors and tests

| Area | Content |
|---|---|
| Errors | 400 validation_failed - missing required field (AC6, BR1–BR5, BR8, BR10, BR15, BR16, BR20–BR22); 400 validation_failed - inactive or unknown reference data ID (BR2, BR17); 400 validation_failed - invalid employeeId (BR14); 422 business_rule_violation - fteAmount exceeds headCountAmount (AC5, BR16); 422 business_rule_violation - endDate not after projectedStartDate (AC7, BR12); 422 business_rule_violation - endDate earlier than today (BR13); 403 forbidden - caller lacks create permission (SEC 002, spec Error Behaviour); 503 service_unavailable - upstream dependency unavailable (spec Error Behaviour) |
| Tests | AC1: Save & Exit stores draft at status 123 and returns all identifiers; AC2: SUBMIT as non-leadership routes to status 127 PPL Review; AC3: SUBMIT as leadership routes to status 145 DPP Ops Review; AC4: Exit returns success with no persisted data; AC5: fteAmount exceeds headCountAmount returns 422; AC6: missing required field returns 400; AC7: endDate not after projectedStartDate returns 422; endDate earlier than today returns 422; inactive dppGroupId returns 400; invalid employeeId returns 400; caller without create permission returns 403; upstream service unavailable returns 503 |

## Assumptions and open items

| ID | Item | Owner | Decision required |
|---|---|---|---|
| OI-1 | API path `/core-asks` vs proposed `/api/v1/asks` in Master LLD catalogue (MLLD §15) | Solution Architecture | Confirm canonical path before contract freeze |
| OI-2 | "Top-3 set" for `levelNeedId` referenced in BR17 — specific reference data IDs not defined | Business Analysis | Provide the exact levelNeedId values that constitute the top-3 set |
| OI-3 | "First option" ID for `needReasonId` (BR6) and `generalSpecialityNeedId` (BR7) — specific reference data IDs not defined | Business Analysis | Provide the exact option IDs for the first-option clearing rule |
| OI-4 | "Retirement" value for `needReasonId` referenced in BR11 — specific reference data ID not defined | Business Analysis | Provide the exact needReasonId value for retirement |
| OI-5 | Leadership group membership rule and attribute source for routing (BR24, BR25, AC2, AC3) — not present in RBAC matrix reference | Security | Define the attribute name and source used to determine leadership group membership |
| OI-6 | Exact permission code for "Create New ASK" — permission name not specified in spec or RBAC matrix | Security | Confirm the permission code string |
| OI-7 | Success HTTP status code — spec states 200 or 201 but does not confirm which applies for a create operation | API Contract owner | Confirm 200 or 201 |
| OI-8 | Idempotency key requirement and scope for the create command — not specified in spec | Solution Architecture | Confirm whether an idempotency key header is required and its scope |
| OI-9 | Transaction scope across Ask, AskVersion, Task, Comment, Audit, and Attachment entities — explicitly unconfirmed in spec (Assumption 1) | Solution Architecture | Confirm atomic transaction boundary |
| OI-10 | "New-version only" field semantics for `pml`, `titlingCategory`, `transitionalCoach` (BR9, BR18, BR19) — not clarified whether this create operation is the new-version context | Business Analysis | Clarify whether new-version-only fields are required or optional on initial create |
| OI-11 | HTTP 422 for business rule violations vs Master LLD error catalogue (MLLD §12.2) which defines only 400 for validation_failed — 422 not listed | Solution Architecture | Confirm 422 is an approved status code for business rule violations in this platform |
| OI-12 | Leadership submitter routing shortcut detail unconfirmed in Master LLD (MLLD §7.4, open item LLD T03) | Solution Architecture | Confirm routing shortcut design is approved |
| OI-13 | `numberofresources` field appears twice in the ADO API Details section — assumed to be a documentation error and the field appears once | Business Analysis | Confirm field appears once and clarify intended semantics |
