# Tasks

The [validate-kemal-analysis tasks](../validate-kemal-analysis/tasks.md) own the compiler prerequisite. Groups 1-3 can proceed independently; compiler integration and final completion require its correctness evidence and usable integration recommendation. Do not duplicate or mark its work complete from this checklist.

## 1. Executable and LSP session

- [ ] 1.1 Add the crystal-lsp executable target with `--stdio` and `--version`, framed JSON-RPC input/output, and stderr logging; verify fragmented/adjacent UTF-8 frames, request IDs, invalid JSON, invalid lengths, and the documented size limits through focused Crystal specs.
- [ ] 1.2 Implement initialization/capability reporting, initialized, shutdown, exit, EOF/parent cleanup, and cancellation; verify a real subprocess session follows protocol lifecycle errors and produces exactly one terminal response per request.
- [ ] 1.3 Add the small reusable protocol replay helper and document the implemented LSP subset and launch command; verify the documented initialize-to-exit exchange against the built binary and remove any remaining scaffold placeholder failure.

## 2. Documents, positions, and project context

- [ ] 2.1 Implement atomic versioned open/change/save/close state, incremental and full changes, and workspace generations; verify ordered edits, stale versions, invalid-range recovery, close/reopen, and a dependency edit with unchanged entrypoint text.
- [ ] 2.2 Centralize file URI identity and UTF-16/UTF-8/compiler position conversion; verify emoji, combining characters, CRLF, overlong character offsets, percent-encoded paths, and definition/diagnostic round trips.
- [ ] 2.3 Implement initialization options, single-target discovery, compiler identity/environment resolution, and explicit compiler enablement; verify a sole shard target, Kemal library fallback, ambiguous targets, an explicit spec target, incompatible setup, and a sentinel macro that never executes when disabled.
- [ ] 2.4 Document resolved-context logs, configuration defaults, restart requirements, and syntax-only operation; verify the examples work with missing semantic prerequisites without preventing LSP initialization.

## 3. Recovering syntax and local features

- [ ] 3.1 Pin and build the Tree-sitter runtime, Crystal grammar/external scanner, and narrow native binding; verify a reproducible macOS arm64 build, ABI compatibility, and native tree/parser ownership, and document exact revisions/build inputs.
- [ ] 3.2 Apply incremental tree edits and extract recoverable scopes/declarations from current buffers; verify fresh and incremental parses of a trailing dot, missing end, combined damage, and unaffected scopes, adding minimal grammar regressions if needed.
- [ ] 3.3 Add lexical completion, plaintext hover, and definition handlers with valid empty/uncertain results; verify shadowing, reassignment, completion replacement ranges, unsupported expressions, and Unicode edits through LSP messages.
- [ ] 3.4 Add prompt bounded syntax feedback and document its coverage separately from compiler checking; verify incomplete code produces current feedback without disabling unrelated local queries.

## 4. Compiler-backed declarations and cold recovery

- [ ] 4.1 Verify the compiler prerequisite's supported-overlay cases, lifecycle behavior, measurements, and integration recommendation are complete; record the evidence used and leave this gate incomplete if required correctness remains unresolved.
- [ ] 4.2 Reuse the validated worker for dependency-only declaration bootstrap, including generated methods, ownership, documentation, and stdlib/dependency locations; verify a cold broken application obtains Kemal's route declarations without a successful application compile, and document dependency-only provenance.
- [ ] 4.3 Resolve lexical receiver facts and explicit block restrictions against current declarations; verify renamed block parameters, a user-defined equivalent block API, conflicting overloads, shadowing/reassignment, and application-side class reopening/overrides without hard-coded Kemal names or stale dependency-only exact answers.
- [ ] 4.4 Implement background typed-receiver queries for zero-argument member chains, with context-keyed caching and unknown outcomes for unsupported cases; verify `env.params.` exposes parameter-parser members, probe code is never run, and probe diagnostics/synthetic locations never leak into application results.
- [ ] 4.5 Combine current syntax, compatible dependency facts, and current full-target facts in completion/hover/definition; verify cold trailing-dot/missing-end cases within the configured initial-analysis deadline, real dependency/stdlib navigation, and honest generated-method locations.
- [ ] 4.6 Document semantic coverage, bootstrap/refinement states, cache validity, and cold-start behavior; verify the documented Kemal examples in the protocol replay without first saving or repairing the application.

## 5. Current diagnostics and bounded scheduling

- [ ] 5.1 Connect worker snapshots to all supported open documents and add the debounced one-worker scheduler with bounded coalesced job types; verify rapid edits do not accumulate jobs or delay document synchronization and current interactive requests.
- [ ] 5.2 Publish merged current syntax/compiler diagnostics with appropriate versions, replacement clearing, and observable analysis states; verify both saved-valid/unsaved-invalid and saved-invalid/unsaved-valid cross-file edits, delayed obsolete results, errors that move, worker failures, and recovery.
- [ ] 5.3 Add watched-source/shard invalidation with a bounded refresh fallback and pre-publication checks; verify saved dependency changes, require-glob additions/removals, lockfile changes, closing dirty dependencies, and noncurrent facts being discarded.
- [ ] 5.4 Integrate timeout/cancellation/shutdown cleanup of owned workers and children; verify no orphaned owned processes, no double responses, and successful subsequent work after a worker failure, then document relevant logs and unsupported external-input coverage.

## 6. Zed setup and editor acceptance

- [ ] 6.1 Write build/setup instructions using the existing Crystal extension's crystalline binary override, one selected server, explicit compiler enablement, optional compiler path, and Kemal spec entrypoint; verify the real binary path/serverInfo and explain the compatibility alias and rollback steps.
- [ ] 6.2 Run the documented setup in a disposable Zed workspace from both terminal and Dock launch paths, including paths with spaces; verify only crystal-lsp supplies semantic results and record Zed/extension/toolchain versions and resolved configuration.
- [ ] 6.3 Exercise incomplete route completion, hover, dependency/generated-symbol navigation, two-unsaved-file diagnostics, and restart with dirty buffers in Zed; record visible outcomes and logs, including actionable behavior for missing compiler/dependencies and syntax-only mode.

## 7. End-to-end performance and delivery gate

- [ ] 7.1 Run the complete framed LSP replay matrix against a release build, including a fresh broken application and deliberately delayed compiler work; verify all four capability specs and retain reproducible inputs/traces.
- [ ] 7.2 Measure at least 100 correct warm completion, hover, and definition requests per feature on the recorded reference machine; verify each p95 is at most 100 ms and report cold readiness, diagnostic latency, raw samples, and server/worker memory separately.
- [ ] 7.3 Compare the pinned Crystalline baseline only on equivalent editor workloads and document remaining limitations and supported setup; verify claims distinguish measured improvements, unsupported comparisons, and future features.
- [ ] 7.4 Run `crystal tool format --check src spec`, `crystal spec`, reproducible native build checks, and `openspec validate build-zed-crystal-lsp --strict`; verify the compiler prerequisite, protocol replay, performance budget, and recorded Zed smoke all pass before declaring the server milestone complete, with unavailable checks or pre-existing failures reported separately.
