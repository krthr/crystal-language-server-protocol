# Proposal

## Why

The project needs a usable Crystal language server in Zed: completion, hover, navigation, and current diagnostics while editing real Kemal code. Compiler feasibility alone does not deliver that experience; this change connects a resilient interactive engine, a standards-based server, and verified editor setup.

## What Changes

- Ship a local `crystal-lsp` executable speaking LSP 3.17 over stdio, with lifecycle handling, cancellation, incremental document synchronization, and correct Unicode positions.
- Provide completion, hover, and go-to-definition from current buffers, including recoverable incomplete code and navigation into dependencies and the standard library.
- Make a cold start on an already-incomplete Kemal route an acceptance case; generated declarations and block parameter types must not depend on a previous successful application compile.
- Integrate the compiler worker from `validate-kemal-analysis` for debounced diagnostics and semantic refinement, preserving workspace generations and reporting unavailable analysis honestly.
- Discover simple project targets and compiler paths, provide an explicit override for ambiguous application/library/spec contexts, and retain syntax features when compiler setup is unavailable.
- Deliver documented Zed setup using its existing Crystal extension's binary override, plus an end-to-end Kemal smoke procedure and measured LSP interaction traces.

### First milestone and non-goals

The first supported environment is local macOS arm64 with Crystal 1.21.1, one project root and selected semantic target per server process. The generic LSP interface remains usable by other clients, but Zed is the editor acceptance gate.

This milestone does not include safe project-wide rename/references, broad refactoring, signature help, semantic tokens, custom macro-expansion UI, formatting, multi-root target orchestration, remote/Windows packaging, marketplace publication, or automatic dependency installation. Unsaved ECR and arbitrary macro subprocess inputs retain the compiler experiment's explicit limits. Syntax-level work on a new file is supported; compiler analysis of a nonexistent file is reported unavailable until saved.

### Dependency and completion boundary

`validate-kemal-analysis` owns the underlying `compiler-analysis` capability and its correctness/performance experiment. Read its [proposal](../validate-kemal-analysis/proposal.md), [design](../validate-kemal-analysis/design.md), [spec](../validate-kemal-analysis/specs/compiler-analysis/spec.md), and [tasks](../validate-kemal-analysis/tasks.md) before compiler integration. Its required correctness checks and an accepted integration recommendation are prerequisites to completing this milestone; transport, document handling, and recovery parsing can proceed independently.

This change is complete only when the new server runs in Zed and passes the Kemal editing scenarios through actual LSP messages. A working compiler worker, parser, or initialize handshake alone does not meet that boundary. Prerequisite links describe an implementation dependency, not an OpenSpec scheduling feature.

## Capabilities

### New Capabilities

- `lsp-session`: Standards-compliant stdio sessions, document synchronization, positions, cancellation, and process lifecycle.
- `editor-language-features`: Current-buffer completion, hover, and definition navigation with recovery and explicit semantic uncertainty.
- `project-diagnostics`: Target selection and compiler-backed diagnostics for versioned project contents, with clear setup and analysis availability.
- `zed-integration`: Reproducible local editor setup and a verified Kemal development workflow using the new server.

### Modified Capabilities

None. `compiler-analysis` is introduced by the prerequisite change and is consumed here without duplicating its requirements.

## Impact

- Future implementation adds the server executable, document/index state, LSP handlers, project configuration, compiler scheduling, focused Crystal specs, and editor setup/acceptance documentation.
- Reuse the compiler worker and native Crystal facilities. Add a pinned Tree-sitter runtime and Crystal grammar for error recovery; justify the binding and build integration in the design.
- Reuse Zed's installed Crystal grammar and language registration. The first milestone requires neither a Rust extension project nor a published extension.
- Compiler execution remains explicit per trusted project because Crystal macros can execute programs during analysis. The lightweight syntax path remains available without compiler execution.
