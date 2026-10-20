# Research: Document Upload and Management

**Feature**: [spec.md](spec.md)  
**Date**: 2026-10-20

## Decision 1: Keep document behavior behind storage and scanning boundaries

**Decision**: Define application-level interfaces for file storage and malware validation, with offline implementations used by the training deployment. The document service owns validation, authorization, persistence ordering, and audit events; storage implementations only handle file bytes and generated relative paths.

**Rationale**: The constitution requires layered architecture, offline operation, and a future cloud migration path. Separating storage and scanning keeps provider details out of pages and business rules, and makes fail-closed behavior testable.

**Alternatives considered**:
- Writing files directly from Blazor pages: rejected because it couples UI code to filesystem behavior and weakens authorization and cleanup guarantees.
- Requiring an online malware provider: rejected because core training workflows must work without cloud services.

## Decision 2: Use generated relative paths outside the web root

**Decision**: Store files below a configured application-data directory outside `wwwroot`. Generate a GUID-based relative path using the uploader and project-or-personal scope before saving metadata. Never use the submitted filename as a trusted path component.

**Rationale**: This preserves the existing offline constraint, prevents direct static access and path traversal, and allows the same relative storage identity to represent a future blob name.

**Alternatives considered**:
- Saving under `wwwroot/uploads`: rejected because it bypasses authorization on static-file requests.
- Using the original filename as the path: rejected because names are untrusted and can collide or contain traversal sequences.

## Decision 3: Process virus scans asynchronously and fail closed

**Decision**: After the file and metadata are safely persisted, the application creates a `Pending` scan state and publishes an idempotent work item to Azure Queue Storage. An Azure Function with a Queue Storage trigger performs the virus scan and updates the document to `Available`, `Rejected`, or `ScanFailed`. If the queue or scanner is unavailable, the document remains inaccessible, retries are bounded, and poison messages become an auditable terminal failure. The offline training deployment uses a local dispatcher and background worker with the same state and idempotency contract.

**Rationale**: This preserves responsive uploads while honoring the accepted clarification and the constitution's security-first and fail-closed requirements. Extension and size checks are necessary but are not a substitute for content validation. Keeping the queue and worker behind an application boundary avoids making Azure a requirement for local training.

**Alternatives considered**:
- Allowing the file after extension and size checks: rejected because it can expose malicious content.
- Scanning synchronously during the web request: rejected because it makes upload latency depend on scanner duration and reduces resilience to transient scanner failures.
- Allowing the file and scanning later while it is accessible: rejected because the requirement prohibits availability before scanning.

## Decision 4: Use Azure Queue Storage and a Queue-triggered Function for cloud scanning

**Decision**: Cloud-enabled deployments use Azure Queue Storage as the handoff between the web application and a stateless Azure Function. The queue message contains the document identifier, document/version token, generated storage identity, and an attempt-safe correlation identifier. The Function obtains the file through the storage boundary, scans it, and calls an authenticated application update boundary or durable result handler. It never updates access by trusting queue payload metadata alone.

**Rationale**: Queue Storage triggers provide retry behavior and decouple upload availability from scan duration. The generated storage identity is portable to blob storage, and the version token prevents a late result from changing a replaced or deleted document.

**Alternatives considered**:
- Calling the Function synchronously from the upload request: rejected because it loses the latency and resilience benefits of background processing.
- Putting full file bytes in the queue: rejected because queue messages should contain small work metadata, not document content.
- Letting the Function write the database directly without an application boundary: rejected because it duplicates authorization and persistence rules.

## Decision 5: Resolve access as project membership plus explicit sharing

**Decision**: A project-associated document is visible to authorized project members by default. Owners may additionally share it with selected users or teams outside the project. Owners may revoke explicit shares; administrators may intervene. Every read or mutation re-checks current authorization.

**Rationale**: This matches the clarified user experience and existing project service model, while keeping explicit sharing narrow and revocable.

**Alternatives considered**:
- Requiring an explicit share for every project member: rejected because it conflicts with project-document expectations and adds unnecessary administration.
- Making project documents visible to all authenticated users: rejected because it violates least privilege.

## Decision 6: Use focused service and integration verification

**Decision**: Add focused automated coverage for document validation, authorization, storage cleanup/order, sharing/revocation, and project/task integration. Use the existing SQLite-backed application context for repeatable offline checks, plus manual browser validation for upload progress, preview, and responsive flows.

**Rationale**: The repository currently has no test project, but the constitution requires executable verification at authorization, storage, persistence, and cross-service boundaries. SQLite keeps the core checks local and repeatable.

**Alternatives considered**:
- Relying only on manual browser testing: rejected because IDOR, cleanup, and failure-ordering cases need repeatable assertions.
- Depending on cloud storage or a production malware service in tests: rejected because tests must run offline.

## Resolved Planning Inputs

- Target runtime is the actual project target: .NET 10 with ASP.NET Core Blazor Server.
- Existing persistence is Entity Framework Core with SQLite in the project file; the feature preserves integer document identifiers and text categories.
- Existing authentication, project membership, task, dashboard, and notification services remain the integration boundaries.
- The queue-triggered Function is an external integration boundary, so its message and result expectations are documented in [contracts/virus-scan-queue.md](contracts/virus-scan-queue.md).
