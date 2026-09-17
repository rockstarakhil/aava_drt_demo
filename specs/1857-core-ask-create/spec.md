# Specification: 1857 Core ASK API – Create Core ASK (US-ASK-001)

## Feature Overview
This feature provides an API endpoint to create new Core ASK records through a POST operation. The endpoint supports three submission modes (Save & Exit, SUBMIT, Exit), validates business rules and field constraints, and routes the created ASK to the appropriate workflow status based on the submitter's leadership group membership and the selected button action.

## Business Objective
Enable authorized users to create new Core resource request records with validated business data and proper routing based on submission intent, ensuring that Core ASK records are captured accurately for further processing through the organization's resource planning workflow.

## Functional Requirements

| ID | Requirement | Source |
|----|-------------|--------|
| FR1 | Expose POST /core-asks endpoint to create new Core ASK records | ADO Description - API Details |
| FR2 | Accept request body containing coreAskDetails, comment, documents, and buttonValue | ADO Description - API Details |
| FR3 | Validate all required and conditional fields according to business rules | ADO Description - Business Rules |
| FR4 | Support Save & Exit button behavior to persist draft with status 123 In Progress | ADO Description - Scope, Business Rules |
| FR5 | Support SUBMIT button behavior with conditional routing to status 127 PPL Review or 145 DPP Ops Review based on leadership group membership | ADO Description - Business Rules |
| FR6 | Support Exit button behavior as no-op without persisting changes | ADO Description - Assumptions |
| FR7 | Return created identifiers including askId, askDetailId, taskId, statusId, assigneeId, commentId, and auditId | ADO Description - API Details Response |
| FR8 | Accept and process comment payload when provided | ADO Description - API Details |
| FR9 | Accept and process documents payload when provided | ADO Description - API Details |
| FR10 | Support all specified coreAskDetails fields: coreAskName, dppGroupId, needReasonId, generalSpecialityNeedId, generalSpecialityNeedComment, levelNeedId, pml, projectedStartDate, endDate, outgoingResource, employeeId, headCountAmount, fteAmount, rolePostingId, numberofresources, titlingCategory, transitionalCoach, roleSummary, roleResponsibility, roleQualification | ADO Description - API Details |
| FR11 | Derive outgoingResource as read-only from selected employeeId | ADO Description - Business Rules |
| FR12 | Return version number in response | ADO Description - API Details Response |

## Expected Behaviour

**Scenario: Save Core ASK draft successfully**
- When a caller provides valid Core ASK data with buttonValue set to "Save & Exit"
- The API validates all required and conditional fields
- The API persists the ASK record as a draft
- The API sets the status to 123 In Progress
- The API returns HTTP 200/201 with the created identifiers (askId, askDetailId, taskId, statusId, assigneeId, commentId, auditId) and version 1

**Scenario: Submit Core ASK as non-leadership user**
- When a caller who is not in the leadership group provides valid Core ASK data with buttonValue set to "SUBMIT"
- The API validates all required and conditional fields
- The API creates the ASK record
- The API routes the ASK to status 127 PPL Review
- The API returns HTTP 200/201 with the created identifiers and status 127

**Scenario: Submit Core ASK as leadership user**
- When a caller who is in the leadership group provides valid Core ASK data with buttonValue set to "SUBMIT"
- The API validates all required and conditional fields
- The API creates the ASK record
- The API routes the ASK to status 145 DPP Ops Review
- The API returns HTTP 200/201 with the created identifiers and status 145

**Scenario: Exit without save**
- When a caller sends buttonValue as "Exit"
- The API does not persist any data
- The API returns a success response indicating no changes were made

**Scenario: Validation failure**
- When a caller provides invalid data (missing required fields, invalid reference IDs, constraint violations)
- The API does not persist any data
- The API returns the appropriate error response (400 or 422) with details of the validation failure

## Business Rules

| ID | Rule | Source |
|----|------|--------|
| BR1 | coreAskName is required and max 200 characters | ADO Description - Business Rules |
| BR2 | dppGroupId is required and must be an active option | ADO Description - Business Rules |
| BR3 | needReasonId is required | ADO Description - Business Rules |
| BR4 | generalSpecialityNeedId is required | ADO Description - Business Rules |
| BR5 | levelNeedId is required | ADO Description - Business Rules |
| BR6 | First needReasonId option clears outgoingResource, employeeId, projectedStartDate, and endDate | ADO Description - Business Rules |
| BR7 | First generalSpecialityNeedId option clears generalSpecialityNeedComment | ADO Description - Business Rules |
| BR8 | generalSpecialityNeedComment is required unless generalSpecialityNeedId is the first option | ADO Description - Business Rules |
| BR9 | pml is optional, max 99 characters, new-version only | ADO Description - Business Rules |
| BR10 | projectedStartDate is required | ADO Description - Business Rules |
| BR11 | endDate is required unless needReasonId is retirement | ADO Description - Business Rules |
| BR12 | endDate must be greater than projectedStartDate | ADO Description - Business Rules |
| BR13 | endDate must not be earlier than today | ADO Description - Business Rules |
| BR14 | outgoingResource is derived and read-only from selected employeeId | ADO Description - Business Rules |
| BR15 | headCountAmount is required | ADO Description - Business Rules |
| BR16 | fteAmount is required and must not exceed headCountAmount | ADO Description - Business Rules |
| BR17 | rolePostingId is required only when levelNeedId is in the top-3 set | ADO Description - Business Rules |
| BR18 | titlingCategory is required, new-version only | ADO Description - Business Rules |
| BR19 | transitionalCoach is optional, max 99 characters, new-version only | ADO Description - Business Rules |
| BR20 | roleSummary is required | ADO Description - Business Rules |
| BR21 | roleResponsibility is required | ADO Description - Business Rules |
| BR22 | roleQualification is required | ADO Description - Business Rules |
| BR23 | buttonValue controls save/submit/exit behavior | ADO Description - Business Rules |
| BR24 | SUBMIT with non-leadership submitter routes to status 127 PPL Review | ADO Description - Business Rules |
| BR25 | SUBMIT with leadership-group submitter routes to status 145 DPP Ops Review | ADO Description - Business Rules |
| BR26 | Save & Exit routes to status 123 In Progress | ADO Description - Business Rules |

## Acceptance Criteria

| ID | Criterion | Traces to |
|----|-----------|----------|
| AC1 | Given valid Core ASK data and buttonValue is Save & Exit, when POST /core-asks is called, then the API saves the ASK as draft with status 123 In Progress and returns created identifiers | FR4, BR26 |
| AC2 | Given valid Core ASK data, caller is not in leadership group, and buttonValue is SUBMIT, when the create endpoint is called, then the API creates the ASK and routes it to status 127 PPL Review | FR5, BR24 |
| AC3 | Given valid Core ASK data, caller is in leadership group, and buttonValue is SUBMIT, when the create endpoint is called, then the API creates the ASK and routes it to status 145 DPP Ops Review | FR5, BR25 |
| AC4 | Given buttonValue is Exit, when the create endpoint is called, then no data is saved | FR6 |
| AC5 | Given fteAmount greater than headCountAmount, when the create endpoint is called, then the API returns 422 | FR3, BR16 |
| AC6 | Given a mandatory field is omitted, when the create endpoint is called, then the API returns 400 | FR3 |
| AC7 | Given endDate is not after projectedStartDate, when the create endpoint is called, then the API returns 422 | FR3, BR12 |

## Dependencies

**Upstream Dependencies:**
- Reference data services for DPP Group, ASK Reason, Level, General/Specialty, and Role Posting to validate option IDs
- Employee search/picker service to validate employeeId and derive outgoingResource
- Authorization service to verify caller has permission to create new ASK
- Authorization service to determine leadership group membership for routing logic

**Downstream Dependencies:**
- Workflow status routing service to transition ASK to the appropriate status (123, 127, or 145)
- Attachment upload handling service to process documents payload
- Audit history service to record creation event
- Task assignment service to create and assign task entity

## Constraints

**Business Constraints:**
- Only users with "Create New ASK" permission may invoke this endpoint
- Reference data values must be active at the time of submission
- Leadership group membership determines routing path and cannot be overridden by the caller

**Data Constraints:**
- coreAskName maximum 200 characters
- pml maximum 99 characters
- transitionalCoach maximum 99 characters
- Date fields must be valid ISO date format
- Numeric fields must be valid integers

**Timing Constraints:**
- endDate must not be earlier than the current date (today)
- endDate must be after projectedStartDate

## API and Integration Considerations

**Endpoint Contract:**
- Method: POST
- Path: /core-asks
- Content-Type: application/json (assumed)
- Request body structure as defined in ADO API Details section
- Response body structure as defined in ADO API Details section

**Integration Points:**
- Reference data validation services must be called to verify dppGroupId, needReasonId, generalSpecialityNeedId, levelNeedId, and rolePostingId are valid and active
- Employee service must be called to validate employeeId and retrieve outgoingResource value
- Authorization service must be called to verify create permission and determine leadership group membership
- Workflow service must be called to set initial status based on buttonValue and leadership group
- Attachment service must be called when documents array is provided
- Comment service must be called when comment is provided
- Audit service must be called to record creation event

**Error Response Contract:**
- 400 for missing required fields or invalid active option selections
- 422 for business rule violations (date constraints, FTE exceeds headcount)
- Standard error response format should include field-level validation details

## Data Considerations

**Data Consumed:**
- All fields in coreAskDetails object as specified in the request schema
- comment text
- documents array (file upload payload)
- buttonValue enumeration
- Caller identity and authorization context (implicit)

**Data Produced:**
- askId (new identifier for the ASK entity)
- askDetailId (new identifier for the ASK detail entity)
- taskId (new identifier for the associated task)
- statusId (workflow status: 123, 127, or 145)
- assigneeId (task assignee identifier)
- commentId (identifier for the saved comment, if provided)
- auditId (identifier for the audit history record)
- version (set to 1 for new ASK)
- Complete coreAskDetails object echoed back in response

**Data Ownership:**
- The created ASK record is owned by the creating user
- Task assignment follows workflow routing rules

**Data Sensitivity:**
- Employee identifiers and resource names may be considered PII
- Role descriptions and qualifications may contain sensitive business information
- Access control must be enforced based on caller permissions

**Data Retention:**
- Not specified in the ticket or available Knowledge Base sources

## Non Functional Requirements

| Category | Requirement | Source |
|----------|-------------|--------|
| Performance | None identified | Not specified in ADO ticket |
| Availability | None identified | Not specified in ADO ticket |
| Scalability | None identified | Not specified in ADO ticket |
| Observability | None identified | Not specified in ADO ticket |

## Security Considerations

**Authentication:**
- Caller must be authenticated to invoke the endpoint
- Authentication mechanism not specified in the ticket

**Authorization:**
- Caller must have permission to create new ASK records
- Authorization check must occur before any data validation or persistence
- Leadership group membership must be determined from the authenticated caller's profile

**Data Protection:**
- Employee identifiers and resource names should be treated as PII
- Role descriptions may contain sensitive business information
- Attachment content should be scanned and validated

**Compliance:**
- Not specified in the ticket or available Knowledge Base sources

## Error Behaviour

**When required fields are missing:**
- The API must not persist any data
- The API must return HTTP 400
- The response must indicate which required fields are missing

**When reference data IDs are invalid or inactive:**
- The API must not persist any data
- The API must return HTTP 400
- The response must indicate which reference IDs are invalid

**When endDate is earlier than today:**
- The API must not persist any data
- The API must return HTTP 422
- The response must indicate the date constraint violation

**When endDate is not after projectedStartDate:**
- The API must not persist any data
- The API must return HTTP 422
- The response must indicate the date ordering constraint violation

**When fteAmount exceeds headCountAmount:**
- The API must not persist any data
- The API must return HTTP 422
- The response must indicate the FTE constraint violation

**When caller lacks create permission:**
- The API must not persist any data
- The API must return HTTP 403 (assumed, not specified in ticket)
- The response must indicate insufficient permissions

**When upstream dependency services are unavailable:**
- The API should return HTTP 503 (assumed, not specified in ticket)
- The response should indicate which service is unavailable
- No partial data should be persisted

## Assumptions

1. **Transaction behavior across write operations is not confirmed** (Unconfirmed) - The ticket explicitly states this is not confirmed in the current export. The specification assumes atomic transaction behavior where all entities (ASK, task, comment, audit, attachments) are created together or none are created.

2. **buttonValue=Exit is treated as no-op** (Confirmed) - The ticket explicitly states this in the Assumptions section.

3. **Attachment file-upload contract is represented as "file upload array"** (Confirmed) - The ticket explicitly states this in the Assumptions section.

4. **Authentication mechanism exists** (Unconfirmed) - The ticket references "permitted user" and "caller has permission" but does not specify the authentication mechanism.

5. **Leadership group membership is determinable from caller context** (Unconfirmed) - The ticket requires routing based on leadership group but does not specify how this attribute is obtained.

6. **"Top-3 set" for levelNeedId is defined in reference data** (Unconfirmed) - BR17 references "top-3 set" but does not define which level IDs are included.

7. **"First option" for needReasonId and generalSpecialityNeedId is defined in reference data** (Unconfirmed) - BR6 and BR7 reference "first option" but do not specify the option ID or value.

8. **"Retirement" value for needReasonId is defined in reference data** (Unconfirmed) - BR11 references retirement but does not specify the option ID.

9. **"New-version only" fields are applicable to this implementation** (Unconfirmed) - BR9, BR18, BR19 reference "new-version only" but do not clarify whether this implementation is the new version.

10. **Response HTTP status code is 200 or 201** (Unconfirmed) - The ticket does not specify the success status code.

11. **Request Content-Type is application/json** (Unconfirmed) - The ticket shows JSON structure but does not explicitly state Content-Type.

12. **numberofresources field appears twice in the request schema** (Unconfirmed) - The API Details section shows this field twice with different values; the specification assumes this is a documentation error and the field appears once.

## Out of Scope

- Amend existing Core ASK (explicitly excluded in ADO Description - Scope)
- Rotational ASK creation (explicitly excluded in ADO Description - Scope)
- UI presentation and tooltip behavior (explicitly excluded in ADO Description - Scope)
- GET, PUT, PATCH, DELETE operations on Core ASK resources
- Bulk creation of multiple Core ASK records in a single request
- Workflow state transitions after initial creation
- Notification or email triggers upon ASK creation
- Reporting or analytics on created ASK records
- Historical version retrieval beyond the initial version 1

## Open Questions

None identified. All questions that would block specification have been captured in Assumptions.

## Traceability

| Artifact | Reference |
|----------|----------|
| ADO ticket | 1857 |
| ADO fields used | Title, Description (User Story, Business Objective, Feature Description, Scope, Preconditions, Postconditions, Business Rules, Functional Requirements, Validation Rules, Error Handling, API Details, Dependencies, Assumptions, Story Points, Definition of Done), Acceptance Criteria |
| Knowledge Base sources | None - Knowledge Base was not available or did not contain relevant sources for this feature |
| Repository | solutionlab-11 |
| Branch | feature/devtesting |
| Generated by | Specification stage of the AI DLC pipeline |
| Generated at | 2025-01-10T08:15:00Z |