# Project Diagnostics

## Purpose

Select an understandable Crystal analysis context and deliver current editor diagnostics while preserving useful syntax features when semantic analysis is unavailable.

## Protocol References

This capability uses LSP 3.17 [InitializeParams.initializationOptions](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize), [PublishDiagnostics](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_publishDiagnostics), [DidSave](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didSave), [DidChangeWatchedFiles](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeWatchedFiles), and [LogMessage](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#window_logMessage). Compiler behavior is inherited from the prerequisite [compiler-analysis spec](../../../validate-kemal-analysis/specs/compiler-analysis/spec.md).

## ADDED Requirements

### Requirement: Explicit and discoverable target selection

The server SHALL resolve one project root and selected analysis target, honoring an explicit entrypoint before automatic discovery. It SHALL expose the selected target, compiler identity, flags, dependency state, and analysis coverage. Ambiguous targets, missing dependencies, or incompatible compilers MUST produce actionable setup information while preserving syntax features.

#### Scenario: Simple application

- **WHEN** a project contains one valid executable target in shard.yml and a compatible compiler is available
- **THEN** the server discovers that target without an extra entrypoint setting and reports the resolved context

#### Scenario: Library or ambiguous project

- **WHEN** Kemal itself is opened without an executable target, or several executable targets are possible
- **THEN** the server explains declaration-only or unavailable semantic coverage and how to select a spec/application entrypoint
- **AND** an explicit focused spec target selects that context after restart instead of silently analyzing the currently edited file as a standalone program

### Requirement: Compiler execution is explicitly enabled

Compiler and dependency-semantic analysis SHALL remain disabled until the server is explicitly configured to allow project-code execution for the workspace. With it disabled, source parsing and supported lexical features SHALL remain available. Enabling it MUST clearly identify that Crystal macros can execute programs; process isolation MUST NOT be described as a sandbox.

#### Scenario: Syntax-only startup

- **WHEN** a project containing a macro with an observable external side effect is opened without compiler enablement
- **THEN** the macro does not execute, lexical features work, and the editor receives a clear explanation of semantic availability

#### Scenario: Enabled trusted project

- **WHEN** the documented compiler option is enabled for the selected Kemal project and the server restarts
- **THEN** dependency indexing and compiler diagnostics become eligible to run under the resolved analysis context

### Requirement: Current diagnostic publication

The server SHALL publish current syntax feedback and debounced compiler diagnostics for the selected target, using all supported open-buffer overlays. Diagnostic locations SHALL use the session encoding. Old compiler results MUST NOT be published for a newer workspace generation, and corrected diagnostics SHALL be removed through replacement publication.

#### Scenario: Fix across unsaved files

- **WHEN** an error is introduced and then corrected by edits to two existing dependent files without saving
- **THEN** diagnostics first reflect the error and then remove it after analysis of the corrected combined snapshot
- **AND** published versions match open documents when the client supports diagnostic versions

#### Scenario: Old compile finishes last

- **WHEN** an older compile finishes after an affected buffer or target configuration changes
- **THEN** its diagnostics are discarded and cannot restore errors from the old snapshot

#### Scenario: Current syntax error

- **WHEN** an incomplete expression prevents successful compiler analysis
- **THEN** the server provides current syntax or compiler-error feedback, keeps interactive features available, and does not label the target fully type-checked

### Requirement: Analysis availability is observable

The server SHALL distinguish checking, completed target analysis, syntax-only operation, and failed or unsupported semantic analysis through standard editor messages or logs. Compiler failure MUST NOT masquerade as successful validation. Diagnostic replacement and invalidation SHALL avoid leaving stale errors presented as current results.

#### Scenario: Worker fails

- **WHEN** analysis times out, crashes, or encounters unsupported dirty external inputs
- **THEN** the editor receives the reason and a useful next action, current syntax feedback remains available, and a later valid analysis can recover

#### Scenario: New unsaved file

- **WHEN** a file with no disk representation is opened and edited
- **THEN** lexical features and syntax feedback operate on its buffer while compiler coverage is explicitly unavailable until the file is saved

### Requirement: Relevant project changes invalidate analysis

Changes to consumed sources, require membership, shard metadata, dependency state, or analysis configuration SHALL invalidate affected indexes and compiler results. Closing a document SHALL release its overlay and invalidate dependent results. Unsupported multi-root semantics MUST NOT silently mix projects in one target.

#### Scenario: Dependency or require tree changes

- **WHEN** a saved required file changes, a file is added to a require glob, or shard.lock changes
- **THEN** subsequent features and diagnostics use refreshed project facts rather than the previous dependency index

#### Scenario: Close an unsaved dependency

- **WHEN** a changed dependency buffer closes and its saved contents differ
- **THEN** dependent analysis reverts to the saved contents and previously cached unsaved facts are no longer presented as current

### Requirement: Background analysis is bounded

The server SHALL coalesce rapid edits, bound concurrent compiler work, and retain only current scheduled work. Compilation MUST NOT block document synchronization or interactive queries. Shutdown, cancellation, and timeout SHALL release owned compiler processes, including their children.

#### Scenario: Rapid editing

- **WHEN** many edits arrive while a compiler job is running
- **THEN** the session continues applying edits, obsolete jobs cannot accumulate without bound, and the newest eligible snapshot is analyzed after editing settles
