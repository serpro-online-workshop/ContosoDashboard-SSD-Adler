# Implementation Plan: Document Upload and Management

**Branch**: `main` | **Date**: 2026-10-20 | **Spec**: [spec.md](spec.md)  
**Input**: Feature specification from [spec.md](spec.md)

## Summary

Add secure, searchable document management to the existing ContosoDashboard application. The feature will introduce document metadata, project/task relationships, explicit sharing with revocation, audit events, and local file storage behind replaceable interfaces. After upload, virus scanning runs asynchronously: the application publishes an idempotent scan message to Azure Queue Storage, an Azure Function with a Queue Storage trigger scans the file and updates the document security state. The offline training deployment uses local equivalents without requiring Azure.

## Technical Context

**Language/Version**: C# on .NET 10 (`ContosoDashboard.csproj`)  
**Primary Dependencies**: ASP.NET Core Blazor Server, Entity Framework Core 10, SQLite, existing cookie authentication and authorization services; optional Azure Storage Queues and Azure Functions for asynchronous virus scanning  
**Storage**: SQLite for metadata; local filesystem outside `wwwroot` for training file bytes; generated relative paths; Azure Queue Storage for scan work items in cloud-enabled deployments  
**Testing**: Add focused automated coverage for services, persistence, authorization, storage cleanup, and integrations; use `dotnet build` plus the browser and performance flows in [quickstart.md](quickstart.md)  
**Target Platform**: Windows-friendly local/offline training deployment; web browser client  
**Project Type**: Single web application with Blazor Server pages and application services  
**Performance Goals**: 30-second valid upload target for files up to 25 MB; 2-second list/search targets; 3-second preview target, as defined by SC-001 through SC-004  
**Constraints**: Files outside `wwwroot`; generated unique storage paths; 25 MB limit; supported-type allowlist; fail-closed asynchronous malware validation; idempotent queue processing; service-layer authorization on every document operation; integer identifiers and text categories; no cloud dependency for training  
**Scale/Scope**: Existing employee/project/task application; document lists up to 500 accessible records; one feature slice spanning models, DbContext, services, pages, dashboard, task/project views, notifications, and audit reporting

## Constitution Check

*GATE: Must pass before Phase 0 research and again after Phase 1 design.*

| Principle / gate | Status | Evidence in this plan |
|---|---|---|
| Layered architecture and abstractions | PASS | Models and EF configuration remain in `Models/` and `Data/`; document orchestration stays in services; filesystem and malware checks are interface boundaries. |
| Security and authorization by default | PASS | All document reads and mutations re-check owner, project, share, revoked-share, and administrator permissions in the service layer; files are not static web assets. |
| Requirements before implementation | PASS | Clarified [spec.md](spec.md) supplies prioritized stories, 25 functional requirements, acceptance scenarios, edge cases, and measurable outcomes. |
| Verification at the risk boundary | PASS | [quickstart.md](quickstart.md) defines focused automated and browser checks for validation, IDOR, persistence ordering, sharing, and integrations. |
| Simplicity, auditability, and offline operation | PASS | Local SQLite/filesystem implementations are the default; audit events cover security-sensitive operations; no cloud service is required. |
| Existing technical constraints | PASS | Existing `ApplicationDbContext`, integer keys, text categories, mock roles, and service registration remain the integration points. |

No constitution violations require complexity tracking or exception approval.

## Phase 0: Research Summary

Research is complete in [research.md](research.md). The key resolved decisions are:

1. Use replaceable storage, scan-dispatch, and malware-validation boundaries with offline implementations.
2. Store generated relative paths outside `wwwroot`, with safe save ordering and cleanup.
3. Process virus scans asynchronously through Azure Queue Storage and a Queue-triggered Azure Function in cloud-enabled deployments, while failing closed when scanning is unavailable.
4. Grant project members automatic access to project documents and use explicit, revocable shares for additional recipients.
5. Add focused offline automated checks because the repository currently has no test project.

No external research dependency remains unresolved, and no `NEEDS CLARIFICATION` marker remains in the technical context.

## Phase 1: Design Summary

The design artifacts are complete:

- [data-model.md](data-model.md) defines `Document`, `DocumentShare`, `DocumentAuditEvent`, notification integration, relationships, validation, indexes, access evaluation, and state transitions.
- [quickstart.md](quickstart.md) defines build/start prerequisites, browser acceptance flows, focused automated checks, and performance checks.
- [contracts/virus-scan-queue.md](contracts/virus-scan-queue.md) defines the queue message, idempotency, retry, poison-message, and result-update contract between the application and the Azure Function.

## Project Structure

### Documentation (this feature)

```text
specs/001-document-upload-management/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── checklists/
│   └── requirements.md
└── tasks.md                 # Created by /speckit-tasks
```

### Source Code (repository root)

```text
ContosoDashboard/
├── Data/
│   └── ApplicationDbContext.cs       # New document DbSets, relationships, indexes
├── Models/
│   ├── Document.cs                   # Document metadata and security state
│   ├── DocumentShare.cs              # Explicit user/team shares and revocation
│   └── DocumentAuditEvent.cs         # Audit record for document operations
├── Services/
│   ├── DocumentService.cs            # Validation, authorization, queries, lifecycle
│   ├── FileStorageService.cs         # IFileStorageService and local implementation
│   ├── MalwareScanningService.cs     # IMalwareScanner and offline fail-closed behavior
│   ├── VirusScanDispatcher.cs        # Queue publisher and local background adapter
│   └── NotificationService.cs        # Document notification support
├── Pages/
│   ├── Documents.razor               # My Documents, Shared with Me, search/filter/manage
│   ├── DocumentDetails.razor         # Metadata, preview, download, share, revoke
│   ├── ProjectDetails.razor          # Project document section
│   ├── Tasks.razor                   # Task document attachment entry point
│   └── Index.razor                   # Recent documents and count
├── Shared/
│   └── NavMenu.razor                 # Documents navigation
├── Program.cs                        # Service registration and authorized file routes
└── wwwroot/css/site.css              # Focused document UI styling
```

A focused test project or equivalent test directory will cover the service and persistence boundaries; its exact location is finalized by `/speckit-tasks` based on the repository's available test tooling.

**Structure Decision**: Extend the existing single Blazor Server project and its established `Models` / `Data` / `Services` / `Pages` boundaries. Do not introduce a separate frontend, API, repository layer, or cloud provider for the training implementation.

## Implementation Boundaries

- Pages collect user input and present loading, progress, empty, success, and failure states; they do not construct trusted paths or bypass services.
- `DocumentService` is the only application boundary for document authorization, metadata queries, lifecycle mutations, shares, notifications, and audit events.
- `IFileStorageService` owns byte persistence and deletion only; it receives generated relative paths and never decides user authorization.
- `IMalwareScanner` owns content validation and must fail closed when unavailable.
- `IVirusScanDispatcher` publishes a scan work item after the file and metadata are safely persisted. The cloud implementation writes to Azure Queue Storage; the offline implementation executes an equivalent local background job.
- The Azure Function uses a Queue Storage trigger, reads only the generated storage identity, scans the file, and updates the document security state through an authenticated application boundary. It must not trust a user-supplied filename, grant access directly, or emit duplicate terminal updates.
- Upload and document views expose `Pending`, `Available`, `Rejected`, and `ScanFailed` states. Only `Available` files can be previewed, downloaded, searched, shared, or attached.
- Queue retries use bounded attempts with poison-message handling and an auditable terminal failure. Repeated delivery is safe through document/version idempotency checks.
- Authorized download/preview handling must resolve a document through `DocumentService` before returning a stream; no direct static-file exposure is allowed.
- Persistence uses integer document keys, text categories, relationships to existing users/projects/tasks, and indexes needed by the stated list/search limits.

## Complexity Tracking

No constitution violations. No complexity exceptions requested.
