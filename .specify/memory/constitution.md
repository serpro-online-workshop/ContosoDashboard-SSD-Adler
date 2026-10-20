<!--
Sync Impact Report
- Version change: template -> 1.0.0
- Modified principles: placeholders -> five project principles
- Added sections: Technical Constraints; Development Workflow
- Removed sections: none
- Follow-up TODOs: repair the missing Resolve-TemplateContent function in common.ps1
-->

# ContosoDashboard Constitution

## Core Principles

### I. Layered Architecture and Abstractions
Features MUST preserve the separation between pages, services, data, and models. Infrastructure
dependencies MUST be accessed through interfaces when a local implementation may later be replaced
by a cloud implementation. Business logic MUST remain independent of storage-provider details so
the application can continue to run offline and retain a practical migration path.

### II. Security and Authorization by Default
Every protected page and operation MUST enforce authentication and authorization. Service-layer
checks MUST protect against unauthorized data access and IDOR even when a page or endpoint already
performs authorization. User-supplied filenames, paths, and metadata MUST be validated and MUST NOT
be used to construct trusted storage paths directly. Security-sensitive behavior MUST fail closed.

### III. Requirements Before Implementation
Each feature MUST have a clear user-facing requirement, acceptance criteria, and an implementation
plan before code changes begin. Changes MUST remain within the existing application architecture
unless a documented constraint requires otherwise. Ambiguous requirements MUST be recorded as
clarifications rather than resolved implicitly in code.

### IV. Verification at the Risk Boundary
New behavior MUST include focused verification for its primary acceptance criteria. Authorization,
storage, persistence, and cross-service changes MUST include integration coverage or an equivalent
executable check. A change is not complete until the relevant build, tests, or validation command
has passed, or the remaining limitation is explicitly documented.

### V. Simplicity, Auditability, and Offline Operation
The simplest design that satisfies the requirements MUST be preferred. The application MUST remain
usable without cloud services in its training configuration. Operations that affect documents,
permissions, or other security-relevant state MUST produce sufficient audit information for the
authorized audience. Performance targets and known training-only limitations MUST be documented
when they affect design decisions.

## Technical Constraints

The project targets ASP.NET Core with Blazor Server, Entity Framework Core, SQL Server LocalDB,
Bootstrap, and the existing mock authentication system. Training deployments MUST work offline
and use local filesystem storage for uploaded files. Files MUST be stored outside `wwwroot`, use
GUID-based paths, enforce an allowlist and the 25 MB per-file limit, and follow the sequence:
validate, authorize, generate a unique path, save the file, then persist metadata. Document storage
MUST be exposed through `IFileStorageService` so Azure Blob Storage can be added without business
logic changes. Existing integer identifiers and text category values MUST be preserved for document
data unless an approved migration changes that contract.

## Development Workflow

Work MUST proceed from specification to plan to tasks to implementation. Reviews MUST check
constitution compliance, authorization boundaries, data-validation behavior, and regression risk.
Before completion, the implementer MUST run the narrowest relevant executable validation and then
the broader project build or test command when available. Unrelated refactors MUST be excluded.

## Governance
<!-- Example: Constitution supersedes all other practices; Amendments require documentation, approval, migration plan -->

This constitution governs feature specifications, plans, tasks, implementation, and reviews for
ContosoDashboard. Amendments MUST state the motivation, affected principles, compatibility impact,
and migration or follow-up work. Every amendment MUST update the semantic version and amendment
date. MAJOR versions remove or redefine obligations, MINOR versions add or materially expand
obligations, and PATCH versions clarify wording without changing intent. Reviews MUST verify that
new work follows these principles; deviations require explicit justification in the relevant plan.

**Version**: 1.0.0 | **Ratified**: 2026-10-20 | **Last Amended**: 2026-10-20
