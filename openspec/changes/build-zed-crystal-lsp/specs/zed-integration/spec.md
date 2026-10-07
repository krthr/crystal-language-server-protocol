# Zed Integration

## Purpose

Make the new Crystal language server usable in the developer's daily Zed workflow through reproducible local setup and verified editing behavior on Kemal.

## Integration References

Use Zed's documented [language-server binary overrides and selection](https://zed.dev/docs/configuring-languages#configuring-language-servers), [environment handling](https://zed.dev/docs/environment), and existing [Crystal language extension](https://github.com/crystal-lang-tools/zed-crystal). The server identity and advertised capabilities follow LSP 3.17 [InitializeResult](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initializeResult); feature behavior is defined in the sibling capability specs.

## ADDED Requirements

### Requirement: Reproducible local launch

The project SHALL provide build and Zed configuration instructions for local macOS arm64 with the supported Crystal toolchain. Setup SHALL launch the new server through a documented binary path, preserve existing Crystal syntax support, and require only one selected semantic server during acceptance testing. A compatibility alias MUST be explained and MUST NOT change the actual server identity.

#### Scenario: Existing extension launches the new server

- **WHEN** the user installs the documented Crystal extension, builds crystal-lsp, and applies the documented binary override and server selection
- **THEN** Zed connects to the new binary, which identifies itself as crystal-lsp and supplies the advertised features
- **AND** the user does not need to install the Crystalline executable or build a new Rust extension

#### Scenario: Paths and launch environment

- **WHEN** the workspace or binary path contains spaces and Zed is launched from either a terminal or the Dock
- **THEN** the configured binary and explicit compiler path are honored without shell-quoting errors
- **AND** unavailable paths produce actionable setup information rather than silent failure

### Requirement: End-to-end Kemal editing acceptance

The milestone SHALL include a recorded smoke test in Zed using the pinned Kemal consumer and focused library/spec workloads. It MUST verify completion, hover, definition navigation, current unsaved diagnostics, and recovery through actual editor/server interaction. Protocol-only or compiler-only test results SHALL NOT substitute for this editor check.

#### Scenario: Daily editing workflow

- **WHEN** the tester edits an unfinished Kemal route, navigates a dependency symbol, and introduces then fixes a cross-file unsaved error
- **THEN** the corresponding editor-language-features and project-diagnostics requirements hold in Zed with only the new server providing semantic results
- **AND** the record includes server, editor, extension, compiler, fixture, and platform versions and any remaining limitations

#### Scenario: Restart with dirty buffers

- **WHEN** the language server restarts while the route is already incomplete and dependent buffers are unsaved
- **THEN** replayed document contents restore useful queries and current diagnostics without requiring the user to save or first repair the application

### Requirement: Troubleshooting is discoverable

The setup documentation SHALL explain target selection, compiler enablement, supported analysis coverage, server restart, and where to read logs. The server SHALL report resolved configuration and analysis failures without contaminating stdout or logging full source buffers by default.

#### Scenario: Diagnose a setup problem

- **WHEN** a compiler version mismatch or missing dependency prevents semantic analysis
- **THEN** the user can identify the selected project context, understand the next corrective action, and continue using available syntax features
