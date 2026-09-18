# ASK-POST-CORE-ASKS - POST /api/v1/core-asks

| Field | Value |
|---|---|
| API ID | ASK-POST-CORE-ASKS |
| Operation | POST /api/v1/core-asks - operationId createCoreAsk |
| Module | ASK |
| Purpose | Create a new Core ASK and either save it as a draft or submit it for review, based on buttonValue. Exit makes no change. |
| Complexity | Complex - sets the workflow status, creates a task and writes several entities in one request |
| Sources | US-ASK-001; ADO 1857; FR1-FR12; BR1-BR26; AC1-AC7; Master LLD 7.4 Core ASK routing |
| Standards | ARCH 002, ARCH 004, API 001, API 002, SEC 001, SEC 002, SEC 003, DATA 001, DATA 003, WF 001, WF 003, REL 001, OBS 001, TEST 001 |
| Status | Approved |

## Settled decisions

These five items were differences between the specification and the Master LLD. They have been
decided and are recorded here so that no later stage has to reopen them. Decision reference
ADR-DRT-API-001.

| Item | Spec said | Master LLD said | Settled |
|---|---|---|---|
| Route | `POST /core-asks` | `POST /api/v1/asks` | `POST /api/v1/core-asks` - the standards require the `/api/v1/` prefix and a plural kebab-case collection, so the spec's resource name keeps the Master LLD's prefix |
| Error code for rule violations | 422 | 400 validation_failed | **400** - the Master LLD error catalogue (12.2) outranks the story text under the source precedence table, and 422 is not in the catalogue |
| Workflow statuses | Numeric IDs 123, 127, 145 | Named states only | Numeric IDs travel in the contract as integers. The ID-to-name mapping is reference data, not contract surface |
| Leadership routing shortcut | Leadership submitter goes straight to 145 | Referenced but unconfirmed | Server-side behaviour with no contract surface. The response returns whichever statusId resulted |
| `buttonValue = Exit` | Accepted as a no-op | Not mentioned | Stays in the enum, returns 200 with no change. The Master LLD is silent, not contradictory |

## Contract summary

| Field | Value |
|---|---|
| Path and query | None |
| Request schema | CreateCoreAskRequest: coreAskDetails, comment, documents, buttonValue (Save & Exit, SUBMIT, Exit) |
| Response schema | CreateCoreAskResponse: askId, askDetailId, coreAskDetails, version, task (taskId, statusId, assigneeId), comment.commentId, auditHistory.auditId |
| Success status | **201 Created** for Save & Exit and SUBMIT. **200 OK** for Exit, which saves nothing (DRT API Design Standards 6) |
| Idempotency | `Idempotency-Key` header required (DRT API Design Standards 6, REL 001) |
| Concurrency | Not applicable - new record created at version 1 |

## Authorization

| Control | Value |
|---|---|
| Authentication | SEC 001 - Entra ID, OAuth 2.0 Authorization Code with PKCE |
| Permission | Create New ASK - exact permission code still to be supplied (OI-3, non-blocking) |
| Role condition | Leadership-group membership decides SUBMIT routing (BR24, BR25); taken from the caller profile, never from the request (SEC 003) |
| Record scope | Business-unit scope not stated in the story (OI-3, non-blocking) |
| Workflow scope | New record; no existing task |
| Audit | Creator, create action, submitted values and resulting status (FR7) |

## Processing flow

1. If buttonValue is Exit, return 200 and save nothing (BR23, AC4).
2. Resolve the current DRT actor and check the Create New ASK permission (SEC 002); otherwise return 403.
3. Validate required fields, lengths and formats (BR1-BR5, BR9, BR10, BR15, BR18-BR22); otherwise return 400.
4. Check that reference IDs are active (BR2) and apply conditional rules BR6-BR8, BR11 and BR17; otherwise return 400.
5. Apply rules BR12, BR13 and BR16; otherwise return **400 validation_failed** (settled - see Settled decisions).
6. Derive outgoingResource from employeeId (BR14) and ignore any value sent by the caller.
7. Set the status: Save & Exit gives 123 In Progress; SUBMIT gives 127 PPL Review, or 145 DPP Ops Review for a leadership-group submitter (BR24-BR26).
8. In one transaction, save the ASK, version 1, comment, attachment links, workflow instance, task and audit record (DATA 003, WF 003).
9. Return 201 with the created identifiers, version 1 and task status (FR7, FR12).

## Business and workflow rules

| Rule ID | Condition | Result |
|---|---|---|
| BR6 | needReasonId is the first option | Clear outgoingResource, employeeId, projectedStartDate and endDate |
| BR7, BR8 | generalSpecialityNeedId is the first option | Clear generalSpecialityNeedComment; for any other option the comment is required |
| BR11 | needReasonId is retirement | endDate is not required |
| BR12, BR13 | endDate is provided | Must be after projectedStartDate and not before today; otherwise 400 |
| BR16 | fteAmount is greater than headCountAmount | Reject with 400 |
| BR17 | levelNeedId is in the top-3 set | rolePostingId is required |
| BR9, BR18, BR19 | New-version fields | pml and transitionalCoach optional, max 99 characters; titlingCategory required |
| BR24-BR26 | buttonValue and submitter group | Save & Exit to 123; SUBMIT to 127, or to 145 for the leadership group |

## Data impact

| Operation | Entity or table | Purpose |
|---|---|---|
| Insert | Ask | New Core ASK record |
| Insert | AskVersion | Version 1 holding coreAskDetails |
| Insert | AskComment | Only when a comment is provided |
| Insert | AskAttachmentLink | Only when documents are provided |
| Insert | WorkflowInstance | Links the ASK to status 123, 127 or 145 |
| Insert | WorkflowTask | Task and assignee returned in the response |
| Insert | AuditHistory | Creation event returned as auditId |
| Read | Reference data and employee | Validate option IDs and derive outgoingResource |

## Events and integrations

| Item | Specification |
|---|---|
| Outbox event | Not applicable - notifications are out of scope in the story |
| Email | Not applicable - out of scope |
| External integration | Attachment upload: contract is a file upload array (OI-6, non-blocking) |

## Errors and tests

| Area | Content |
|---|---|
| Errors | 400 validation_failed - missing required field, inactive reference ID, date rule broken, or fteAmount above headCountAmount; 403 access_denied - no create permission; 409 conflict - duplicate Idempotency-Key with a different body; 503 dependency_unavailable - reference, employee or attachment service down. Every non-2xx returns ProblemDetails |
| Tests | Save & Exit gives 201 and status 123; SUBMIT by non-leadership gives 201 and status 127; SUBMIT by leadership gives 201 and status 145; Exit gives 200 and saves nothing; missing coreAskName gives 400; inactive dppGroupId gives 400; fteAmount above headCountAmount gives 400; endDate not after projectedStartDate gives 400; endDate before today gives 400; missing rolePostingId for a top-3 level gives 400; caller without permission gives 403; repeated Idempotency-Key returns the original result; failure during save leaves no records |

## Open items - none blocking

All of these can be settled after the contract is published. None of them changes the shape of a
request or a response, so none of them blocks this stage.

| ID | Item | Owner | Decision required |
|---|---|---|---|
| OI-3 | Permission code, business-unit scope and leadership-group source are not defined | Security | Provide the RBAC rows |
| OI-6 | Attachment upload contract is only "file upload array" | API Contract owner | Define the upload contract |
| OI-7 | Single transaction across writes is not confirmed in the story | Solution Architecture | Confirm the transaction boundary |
| OI-8 | Values for "first option", "retirement", the "top-3 set" and the status ID to name mapping are not given | Business Analysis | Provide the reference values |
| OI-9 | "New-version only" semantics for pml, titlingCategory and transitionalCoach | Business Analysis | Confirm they apply to version 1 |