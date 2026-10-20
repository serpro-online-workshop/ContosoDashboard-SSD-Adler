# Quickstart Validation: Document Upload and Management

**Feature**: [spec.md](spec.md)  
**Plan**: [plan.md](plan.md)  
**Data model**: [data-model.md](data-model.md)

## Prerequisites

- .NET 10 SDK.
- PowerShell on Windows.
- A clean local SQLite database created by the application, or an isolated test database.
- A writable application-data directory outside `ContosoDashboard/wwwroot`.
- The repository checked out at `ContosoDashboard-SSD-Adler`.
- Azure Storage Queue and an Azure Function are optional for cloud-enabled validation; the offline adapter is the default training path.

## Build and start

From the repository root:

```powershell
cd ContosoDashboard
 dotnet restore
 dotnet build
 dotnet run
```

Open the HTTPS URL printed by `dotnet run`, then use the mock login page to select one of the seeded users.

## Browser acceptance flows

### 1. Upload validation

1. Log in as the seeded employee.
2. Upload a PDF smaller than 25 MB with a title and category.
3. Confirm progress, success feedback, metadata, and the new document in My Documents.
4. Verify that the upload first shows a pending-scan state and becomes usable only after the background worker reports a clean result.
5. Repeat with a supported image and verify browser preview after scanning completes.
6. Try an unsupported extension and a file larger than 25 MB; verify each is rejected with no accessible record.
7. Simulate malware scanning unavailable or rejected; verify the file remains inaccessible and the user receives a non-sensitive status.

### 2. Authorization and project access

1. Log in as the seeded project manager and upload a project document.
2. Log in as a seeded project member; verify the document appears in the project view and can be downloaded.
3. Log in as a user outside the project; verify the document is absent from browse and search results.
4. Attempt direct access using another document identifier; verify the operation is denied without metadata or content disclosure.

### 3. Sharing and revocation

1. As the owner, share a document with a selected user or team.
2. Log in as the recipient; verify Shared with Me and an in-app notification.
3. Log in as an unselected user; verify the document is not visible.
4. Revoke the share as the owner; verify the recipient can no longer preview or download it.
5. Verify the share, revocation, and denied access events are visible to an administrator audit view.

### 4. Replacement, deletion, and failure cleanup

1. Replace an available document with a valid supported file and verify metadata association is preserved.
2. Attempt replacement with an invalid file and verify the existing available file remains usable.
3. Delete an owned document after confirmation and verify it disappears from lists, search, project, and task contexts.
4. Force a storage or metadata failure in a focused test; verify no accessible orphaned file or metadata record remains.

### 5. Task and dashboard integration

1. Open a task with a project and attach an existing document or upload a new related document.
2. Verify the document is associated with the task's project and visible from the task.
3. Return to the dashboard and verify the five most recent user uploads and document count are shown.
4. Add a project document and verify authorized project members receive in-app notifications.

### 6. Azure background scan integration (cloud-enabled deployment)

1. Upload a valid file and verify the application publishes a small JSON scan work item containing the document identifier, scan version, generated storage identity, and correlation identifier.
2. Verify the Queue Storage-triggered Azure Function processes the item without receiving file bytes in the message.
3. Verify a clean result changes only the matching document version to `Available`.
4. Deliver the same message twice and verify no duplicate terminal state, access grant, or audit corruption occurs.
5. Force transient scanner failures and verify bounded retries; exceed the retry limit and verify poison-message handling leaves the document inaccessible as `ScanFailed`.
6. Replace or delete a document before delivering an old result and verify the stale result cannot change the newer document state.

## Focused automated checks

The implementation should add a test project or equivalent repeatable test harness and cover at least:

- Allowlist, 25 MB limit, required metadata, MIME length, and malware failure states.
- Pending-to-available scan transitions, duplicate queue delivery, stale scan versions, bounded retries, and poison-message handling.
- Generated relative paths and rejection of traversal or user-controlled storage identities.
- Save-order cleanup when file storage or metadata persistence fails.
- Owner, project-member, project-manager, recipient, revoked-recipient, and administrator authorization cases.
- Search, filtering, sorting, and result exclusion for inaccessible or non-available documents.
- Share/revoke idempotency and audit event creation.
- Project, task, dashboard, and notification integration.

## Expected performance checks

Using representative local data:

- At least 90% of valid uploads up to 25 MB finish within 30 seconds on a typical network.
- At least 95% of lists with up to 500 accessible documents become usable within 2 seconds.
- At least 95% of searches return a result or empty state within 2 seconds.
- At least 95% of PDF/image previews become usable within 3 seconds.
