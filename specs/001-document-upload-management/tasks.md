# Tasks: Document Upload and Management

**Input**: Design documents from `/specs/001-document-upload-management/`
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/virus-scan-queue.md](contracts/virus-scan-queue.md), [quickstart.md](quickstart.md)

**Tests**: Included because the constitution requires executable verification for authorization, storage, persistence, and cross-service boundaries.

**Organization**: Tasks are grouped by user story so each story can be implemented and tested as an incremental slice.

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Establish the application and test surfaces needed by every story.

- [ ] T001 Create the focused test project `ContosoDashboard.Tests/ContosoDashboard.Tests.csproj` targeting `net10.0`, referencing `ContosoDashboard/ContosoDashboard.csproj`, and using the repository's compatible test framework.
- [ ] T002 [P] Create the optional Azure Functions scan worker project `ContosoDashboard.ScanFunction/ContosoDashboard.ScanFunction.csproj` targeting the supported .NET isolated-worker runtime and Queue Storage trigger packages; keep it deployable separately from the offline web app.
- [ ] T003 [P] Add shared test helpers and isolated SQLite database fixtures in `ContosoDashboard.Tests/Infrastructure/DatabaseFixture.cs` for repeatable offline service tests.
- [ ] T004 [P] Add scan-related configuration sections and safe local defaults to `ContosoDashboard/appsettings.json` and `ContosoDashboard/appsettings.Development.json`; do not store secrets or require Azure in local development.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Build the storage, scan, persistence, authorization, and audit boundaries required before any user story can be completed.

**Checkpoint**: Foundation ready when metadata, file bytes, scan states, queue dispatch, authorized result updates, and audit events can be exercised offline.

- [ ] T005 Create the `Document` entity in `ContosoDashboard/Models/Document.cs` with integer `DocumentId`, required `Title` and `Category`, optional `Description`, `Tags`, `ProjectId`, and `TaskId`, required uploader and file metadata, generated `StoredFilePath`, `SecurityStatus`, `ScanVersion`, `ScanAttemptCount`, `ScanUpdatedDate`, and `ScanFailureReason`; preserve the data-model rule that `FileType` has a maximum of 255 characters and file size is limited to 25 MB.
- [ ] T006 [P] Create the `DocumentShare` entity in `ContosoDashboard/Models/DocumentShare.cs` with integer identity, document, sharer, optional user/team recipient, `SharedDate`, and nullable `RevokedDate`; enforce exactly one recipient target and no duplicate active recipient share.
- [ ] T007 [P] Create the `DocumentAuditEvent` entity in `ContosoDashboard/Models/DocumentAuditEvent.cs` with document, actor, action, outcome, UTC timestamp, and non-sensitive context fields.
- [ ] T008 Update `ContosoDashboard/Data/ApplicationDbContext.cs` with document DbSets, relationships to `User`, `Project`, and `TaskItem`, restrictive delete behavior where required, uniqueness for generated storage paths, and indexes for uploader/date, project/status, category/date/size, active shares, and audit queries.
- [ ] T009 Create document lifecycle and scan-state constants in `ContosoDashboard/Models/DocumentSecurityStatus.cs` for `Pending`, `Scanning`, `Available`, `Rejected`, and `ScanFailed`, ensuring only `Available` records are returned to content-consuming queries.
- [ ] T010 Create the storage boundary `IFileStorageService` and safe local implementation in `ContosoDashboard/Services/FileStorageService.cs`; store outside `ContosoDashboard/wwwroot`, resolve only generated relative paths below the configured root, reject traversal, stream bytes, and delete safely.
- [ ] T011 Create the scanner boundary `IMalwareScanner` and offline implementation in `ContosoDashboard/Services/MalwareScanningService.cs`; make unavailable scanning fail closed and return non-sensitive result classifications.
- [ ] T012 Create `IVirusScanDispatcher` and the local background adapter in `ContosoDashboard/Services/VirusScanDispatcher.cs`; enqueue work only after file and metadata persistence, use `DocumentId` plus `ScanVersion` for idempotency, and keep pending documents inaccessible.
- [ ] T013 Create the Azure Queue Storage dispatcher implementation in `ContosoDashboard/Services/AzureQueueVirusScanDispatcher.cs`; serialize only the fields defined by `specs/001-document-upload-management/contracts/virus-scan-queue.md`, use configured queue credentials, and avoid placing file bytes or secrets in messages.
- [ ] T014 Create the authenticated scan-result application boundary in `ContosoDashboard/Endpoints/VirusScanResultEndpoint.cs`; accept results only when `DocumentId` and `ScanVersion` match the current pending scan, prevent stale or duplicate terminal updates, and write audit events.
- [ ] T015 Create the Azure Function Queue Storage trigger in `ContosoDashboard.ScanFunction/VirusScanFunction.cs`; load the generated storage identity, scan content, submit `Clean`, `MalwareDetected`, `ScannerUnavailable`, `PermanentFailure`, or `StaleVersion` results, and never grant access directly.
- [ ] T016 Configure bounded retry and poison-message handling in `ContosoDashboard.ScanFunction/host.json` and `ContosoDashboard.ScanFunction/VirusScanFunction.cs`; terminal poison handling must leave the document inaccessible as `ScanFailed` and support authorized retry through a new scan version.
- [ ] T017 Add the queue message and result DTOs in `ContosoDashboard.ScanFunction/Contracts/ScanWorkItem.cs` and `ContosoDashboard.ScanFunction/Contracts/ScanResult.cs` matching the documented contract, including correlation and version fields.
- [ ] T018 Create shared document authorization and audit helpers in `ContosoDashboard/Services/DocumentAuthorizationService.cs` and `ContosoDashboard/Services/DocumentAuditService.cs`; re-check owner, administrator, project membership, active share, and revoked share state for every operation.
- [ ] T019 Register the document, storage, scanner, dispatcher, authorization, audit, and hosted local worker services in `ContosoDashboard/Program.cs` without enabling cloud dependencies by default.
- [ ] T020 Add foundational persistence, storage-path, scan-state, idempotency, and authorization tests in `ContosoDashboard.Tests/DocumentFoundationTests.cs`, including failed file-save and failed metadata-save cleanup behavior.

---

## Phase 3: User Story 1 - Upload and Organize a Document (Priority: P1) 🎯 MVP

**Goal**: Authenticated employees can upload supported files, enter required metadata, see scan progress/status, and access only clean files.

**Independent Test**: Upload a valid file as a seeded employee, observe `Pending` followed by `Available`, then verify invalid type, size, malware, unavailable scanner, and partial multi-file outcomes are rejected or inaccessible with clear messages.

### Tests for User Story 1

- [ ] T021 [P] [US1] Add upload validation tests in `ContosoDashboard.Tests/UserStory1UploadTests.cs` for required title/category, supported PDF/Office/text/image types, 25 MB limit, 255-character MIME storage, and per-file multi-upload results.
- [ ] T022 [P] [US1] Add asynchronous scan workflow tests in `ContosoDashboard.Tests/UserStory1ScanTests.cs` for Pending-to-Available, malware rejection, scanner unavailable, duplicate queue delivery, stale `ScanVersion`, bounded retry, and `ScanFailed` behavior.

### Implementation for User Story 1

- [ ] T023 [US1] Implement upload orchestration in `ContosoDashboard/Services/DocumentService.cs` with validate → authorize → generate unique path → save bytes → save metadata as `Pending` → dispatch scan ordering, including cleanup when any persistence step fails.
- [ ] T024 [US1] Implement upload DTOs and validation in `ContosoDashboard/Services/DocumentUploadModels.cs` for title, description, category, project, tags, original filename, content type, and stream size.
- [ ] T025 [US1] Create the authenticated upload page in `ContosoDashboard/Pages/Documents.razor` with multi-file selection, required metadata, progress display, per-file success/error states, Pending scan status, and no display of trusted storage paths.
- [ ] T026 [US1] Add upload navigation and authorized route registration in `ContosoDashboard/Shared/NavMenu.razor` and `ContosoDashboard/Pages/_Imports.razor` as needed for the Documents page.
- [ ] T027 [US1] Add notification and audit events for accepted uploads, rejected uploads, malware results, scanner failures, and scan completion through `ContosoDashboard/Services/NotificationService.cs` and `ContosoDashboard/Services/DocumentAuditService.cs`.
- [ ] T028 [US1] Add the clean-file and rejected-file UI states plus accessible error messaging in `ContosoDashboard/wwwroot/css/site.css` and `ContosoDashboard/Pages/Documents.razor`.

**Checkpoint**: User Story 1 is independently demonstrable when clean files become available only after background scanning and all invalid files remain inaccessible.

---

## Phase 4: User Story 2 - Find and Use Accessible Documents (Priority: P1)

**Goal**: Users can browse, sort, filter, search, preview, and download only documents they currently may access.

**Independent Test**: Seed documents across users, categories, projects, dates, and scan states; verify authorized results, sorting/filtering/search latency, preview, download, and complete exclusion of inaccessible or non-available documents.

### Tests for User Story 2

- [ ] T029 [P] [US2] Add authorized query tests in `ContosoDashboard.Tests/UserStory2QueryTests.cs` for sorting, category/project/date filters, title/description/tag/uploader/project search, 500-document list behavior, and exclusion of Pending, Rejected, ScanFailed, and unauthorized records.
- [ ] T030 [P] [US2] Add download and preview authorization tests in `ContosoDashboard.Tests/UserStory2ContentAccessTests.cs` for owner, project member, shared recipient, revoked recipient, administrator, and IDOR attempts.

### Implementation for User Story 2

- [ ] T031 [US2] Implement paged document browsing, filtering, sorting, and search methods in `ContosoDashboard/Services/DocumentService.cs`; apply authorization and `SecurityStatus == Available` before projecting results and use the data-model indexes.
- [ ] T032 [US2] Implement authorized content response handling in `ContosoDashboard/Endpoints/DocumentContentEndpoints.cs`; resolve every request through `DocumentService`, stream previews/downloads from `IFileStorageService`, set safe content headers, and audit preview/download/denial outcomes.
- [ ] T033 [US2] Complete the browse/search/filter/sort UI in `ContosoDashboard/Pages/Documents.razor` with loading, empty, error, and inaccessible-result states and metadata columns required by FR-011 and FR-012.
- [ ] T034 [US2] Add PDF/image preview and download actions to `ContosoDashboard/Pages/DocumentDetails.razor`, ensuring Pending, Rejected, and ScanFailed documents cannot expose content.
- [ ] T035 [US2] Add accessible pagination, table, filter, and preview styling in `ContosoDashboard/wwwroot/css/site.css` without exposing file paths or sensitive failure details.

**Checkpoint**: User Stories 1 and 2 work independently; users can locate and consume only available documents within authorization and performance targets.

---

## Phase 5: User Story 3 - Manage Owned and Project Documents (Priority: P2)

**Goal**: Owners, project managers, and administrators can safely edit, replace, delete, and retry documents within their authority.

**Independent Test**: The owner edits metadata, replaces a file, retries a failed scan, and deletes a document; project-manager and administrator permissions work; unauthorized mutations are denied and audited.

### Tests for User Story 3

- [ ] T036 [P] [US3] Add metadata and lifecycle authorization tests in `ContosoDashboard.Tests/UserStory3ManagementTests.cs` for owner, project-manager, administrator, unauthorized user, stale identifier, replacement failure, permanent deletion, and scan retry cases.

### Implementation for User Story 3

- [ ] T037 [US3] Implement metadata update, replacement, deletion, and authorized scan-retry methods in `ContosoDashboard/Services/DocumentService.cs`; preserve metadata on valid replacement, clean old/new files safely, create a new scan version on retry, and require confirmation before permanent deletion.
- [ ] T038 [US3] Add management DTOs and validation in `ContosoDashboard/Services/DocumentManagementModels.cs`, preserving required title/category, supported file allowlist, 25 MB limit, and safe generated paths.
- [ ] T039 [US3] Add edit, replace, delete, and retry controls to `ContosoDashboard/Pages/DocumentDetails.razor` with confirmation, progress, failure, and terminal scan states.
- [ ] T040 [US3] Add administrator audit filtering/report query support in `ContosoDashboard/Services/DocumentAuditService.cs` for upload, download, preview, update, replace, delete, share, revoke, denied, and scan outcomes.
- [ ] T041 [US3] Add management result messages and confirmation styling in `ContosoDashboard/wwwroot/css/site.css` without exposing filesystem or scanner internals.

**Checkpoint**: User Story 3 is independently demonstrable when authorized lifecycle operations are safe, auditable, and cannot affect a different document through stale identifiers.

---

## Phase 6: User Story 4 - Share Documents with People and Teams (Priority: P2)

**Goal**: Owners can grant and revoke explicit access for selected users or teams, with notifications and auditable permission changes.

**Independent Test**: An owner shares a document with a selected recipient, the recipient sees Shared with Me and a notification, an unselected user does not, and revocation immediately removes recipient access.

### Tests for User Story 4

- [ ] T042 [P] [US4] Add share and revocation tests in `ContosoDashboard.Tests/UserStory4SharingTests.cs` for selected user/team recipients, duplicate active shares, owner/admin authority, unselected users, revocation, project-member access, notifications, and audit outcomes.

### Implementation for User Story 4

- [ ] T043 [US4] Implement share, list-shared, revoke, and Shared with Me queries in `ContosoDashboard/Services/DocumentService.cs`; enforce exactly one recipient target, active-share uniqueness, owner/admin mutation authority, and current-access checks.
- [ ] T044 [US4] Add share and revoke UI to `ContosoDashboard/Pages/DocumentDetails.razor` with recipient selection, active-share list, confirmation, and immediate revoked state.
- [ ] T045 [US4] Add Shared with Me view and notification navigation in `ContosoDashboard/Pages/Documents.razor` and `ContosoDashboard/Pages/Notifications.razor`.
- [ ] T046 [US4] Extend `ContosoDashboard/Services/NotificationService.cs` and `ContosoDashboard/Models/Notification.cs` for document-share and project-document notification types while preserving existing user-specific read/unread behavior.
- [ ] T047 [US4] Create the administrator document-audit page in `ContosoDashboard/Pages/DocumentAudit.razor` with date, actor, action, outcome, and document filters, without exposing document content.

**Checkpoint**: User Story 4 is independently demonstrable when explicit access is narrow, revocable, notified, and cannot bypass project or document authorization.

---

## Phase 7: User Story 5 - Use Documents in Projects, Tasks, and the Dashboard (Priority: P3)

**Goal**: Documents are available in authorized project and task contexts and appear in dashboard summaries without leaving normal workflows.

**Independent Test**: A project member views project documents, attaches/uploads a task document, sees project association, and sees five recent documents plus document count on the dashboard.

### Tests for User Story 5

- [ ] T048 [P] [US5] Add project and task association tests in `ContosoDashboard.Tests/UserStory5ContextTests.cs` for project-manager/member access, task-project inheritance, unauthorized project/task access, and notification recipients.
- [ ] T049 [P] [US5] Add dashboard query tests in `ContosoDashboard.Tests/UserStory5DashboardTests.cs` for five recent uploads, document count, empty state, and current-user isolation.

### Implementation for User Story 5

- [ ] T050 [US5] Add project and task document query/association methods in `ContosoDashboard/Services/DocumentService.cs`, enforcing that task attachments inherit the task's project and that project membership is re-checked.
- [ ] T051 [US5] Add the project document section, upload entry point, and authorized download actions to `ContosoDashboard/Pages/ProjectDetails.razor`.
- [ ] T052 [US5] Add task document attachment and upload actions to the existing `ContosoDashboard/Pages/Tasks.razor` task workflow, without bypassing `DocumentService`.
- [ ] T053 [US5] Extend `ContosoDashboard/Services/DashboardService.cs` and its `DashboardSummary` type with current-user document count and five most recent available uploads.
- [ ] T054 [US5] Add Recent Documents and document-count summary UI to `ContosoDashboard/Pages/Index.razor` with loading and empty states.
- [ ] T055 [US5] Notify authorized project members after a new available project document in `ContosoDashboard/Services/DocumentService.cs` and `ContosoDashboard/Services/NotificationService.cs`; do not notify unauthorized users.

**Checkpoint**: User Story 5 is independently demonstrable when project, task, dashboard, and notification integrations preserve document authorization.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Complete security, performance, documentation, and end-to-end validation across all stories.

- [ ] T056 [P] Add end-to-end authorization and IDOR regression coverage in `ContosoDashboard.Tests/DocumentAuthorizationRegressionTests.cs` for every read, preview, download, edit, replace, delete, share, revoke, retry, and scan-result operation.
- [ ] T057 [P] Add performance fixtures and query timing checks in `ContosoDashboard.Tests/DocumentPerformanceTests.cs` for 500-document lists, 2-second searches, 30-second 25 MB uploads, and 3-second previews.
- [ ] T058 [P] Add Azure Function contract tests in `ContosoDashboard.Tests/VirusScanQueueContractTests.cs` for JSON shape, missing fields, duplicate delivery, stale versions, retry limits, and poison-message outcomes defined in `contracts/virus-scan-queue.md`.
- [ ] T059 [P] Add structured, non-sensitive logging and operational counters for queue depth, oldest message age, retries, poison messages, scan outcomes, and denied access in `ContosoDashboard/Services/DocumentAuditService.cs` and `ContosoDashboard.ScanFunction/VirusScanFunction.cs`.
- [ ] T060 Update `README.md` with offline local-worker setup, optional Azure Queue/Function configuration, security limitations, and the feature validation link to `specs/001-document-upload-management/quickstart.md`.
- [ ] T061 Run the complete quickstart and focused test suite from `specs/001-document-upload-management/quickstart.md`, then run `dotnet build ContosoDashboard/ContosoDashboard.csproj` and the test project; record any environment-only limitation in the implementation review.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No feature dependencies; T001-T004 can begin immediately, with T002 optional for cloud-enabled deployment.
- **Foundational (Phase 2)**: Depends on T001-T004; blocks all user stories. T005-T009 establish persistence, while T010-T019 establish storage, scanning, queue, authorization, and audit boundaries. T020 validates the foundation.
- **User Stories (Phases 3-7)**: Depend on Phase 2 completion. US1 is the MVP; US2 depends on available documents from US1; US3 and US4 depend on the document service from US1 and can proceed in parallel after it; US5 depends on the document service and existing project/task/dashboard surfaces.
- **Polish (Phase 8)**: Depends on the desired user stories and all foundational scan behavior being complete.

### User Story Dependencies

- **US1 (P1)**: Starts after Phase 2; no dependency on another user story. MVP.
- **US2 (P1)**: Starts after Phase 2; consumes documents created by US1 for realistic validation, but its queries and content authorization are independently testable with seeded data.
- **US3 (P2)**: Starts after US1's document lifecycle service exists; can proceed in parallel with US2 after foundational validation.
- **US4 (P2)**: Starts after US1's document lifecycle and authorization service exists; can proceed in parallel with US2 and US3 where files do not overlap.
- **US5 (P3)**: Starts after US1 and the existing project/task/dashboard integration points are available; integrates US2 content access.

### Within Each User Story

- Write focused tests before implementation where practical and keep them failing until the behavior exists.
- Implement or extend models before service methods, service methods before pages/endpoints, and core behavior before notifications and styling.
- Re-run the story's independent test criteria at each checkpoint.

## Parallel Execution Examples

### Foundation after setup

```text
Task T005: Document model and validation in ContosoDashboard/Models/Document.cs
Task T006: DocumentShare model in ContosoDashboard/Models/DocumentShare.cs
Task T007: DocumentAuditEvent model in ContosoDashboard/Models/DocumentAuditEvent.cs
Task T010: Local file storage boundary in ContosoDashboard/Services/FileStorageService.cs
Task T011: Malware scanner boundary in ContosoDashboard/Services/MalwareScanningService.cs
```

### User Story 1 after foundation

```text
Task T021: Upload validation tests in ContosoDashboard.Tests/UserStory1UploadTests.cs
Task T022: Async scan workflow tests in ContosoDashboard.Tests/UserStory1ScanTests.cs
Task T024: Upload DTOs and validation in ContosoDashboard/Services/DocumentUploadModels.cs
Task T025: Upload page in ContosoDashboard/Pages/Documents.razor
```

### P2 stories after US1 service boundary

```text
Task T029: Browse/search query tests in ContosoDashboard.Tests/UserStory2QueryTests.cs
Task T036: Management lifecycle tests in ContosoDashboard.Tests/UserStory3ManagementTests.cs
Task T042: Sharing/revocation tests in ContosoDashboard.Tests/UserStory4SharingTests.cs
Task T033: Browse/search UI in ContosoDashboard/Pages/Documents.razor (after T031)
Task T039: Management UI in ContosoDashboard/Pages/DocumentDetails.razor (after T037)
Task T045: Shared with Me view in ContosoDashboard/Pages/Documents.razor (after T043)
```

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1 setup and Phase 2 foundation, including the local background scan adapter.
2. Complete Phase 3 User Story 1.
3. Run the upload, scan, failure, authorization, and cleanup tests plus the first quickstart flow.
4. Stop and validate that a clean file becomes available only after scanning; demonstrate the MVP before adding browse and management features.

### Incremental Delivery

1. Foundation plus US1: secure asynchronous upload MVP.
2. Add US2: find, preview, and download authorized documents.
3. Add US3 and US4 in parallel where possible: lifecycle management and sharing/revocation.
4. Add US5: project, task, dashboard, and notification integration.
5. Complete Phase 8 and validate the optional Azure Function deployment separately from offline training.

### Parallel Team Strategy

1. One developer owns models, EF configuration, and tests T005-T009.
2. One developer owns storage/scanning/queue boundaries and the Function contract T010-T017.
3. One developer owns authorization/audit and registration T018-T020.
4. After the foundation checkpoint, separate developers can own US2, US3, and US4; US5 follows the established document service.

## Notes

- `[P]` tasks use different files and have no dependency on incomplete work.
- `[US1]` through `[US5]` map directly to the five prioritized stories in `spec.md`.
- Every task includes an exact repository path and a concrete outcome.
- The cloud scan worker is optional for offline training, but its contract and failure behavior are planned and testable.
