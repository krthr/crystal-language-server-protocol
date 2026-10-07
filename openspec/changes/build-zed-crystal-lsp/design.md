# Design

## Context

See [proposal.md](proposal.md) for the delivery boundary and the four [capability specs](specs) for observable behavior. The repository is a Crystal scaffold. The planned [compiler experiment](../validate-kemal-analysis/design.md) supplies a version-pinned worker, source overlays, structured diagnostics, and evidence about compiler integration; it does not supply a language server or recovery parser.

The first supported configuration is one local macOS arm64 project using Crystal 1.21.1. LSP 3.17 is the compatibility baseline for the implemented subset; capabilities determine interoperability. No newer protocol-only feature is needed for this milestone.

The design is needed because error recovery, compiler isolation, source versions, and editor responses cross several boundaries. A syntax tree alone cannot infer Kemal's generated API, and a previously successful full compile cannot satisfy cold startup with broken application source.

## Goals / Non-Goals

**Goals:**

- Run a real server in Zed and complete the Kemal editing acceptance matrix.
- Keep interactive queries independent of full compilation while preserving honest semantic provenance.
- Reuse the validated compiler worker and existing grammar work with a small implementation surface.

**Non-Goals:**

- No plugin framework, database, general type checker, new macro interpreter, or persistent index format.
- No comprehensive Crystal semantic coverage on arbitrary broken code. Supported recovery cases and uncertain cases are tested explicitly.
- No new Zed extension or publisher infrastructure for the initial binary-override setup.

## Decisions

### 1. Separate the editor loop from compiler jobs

```text
Zed + installed Crystal extension
               |
          LSP over stdio
               |
 session + current document store
        /                  \
 recovering syntax      one compiler job at a time
 + declaration index    (prerequisite worker)
        |                  |
 completion/hover/      diagnostics + dependency facts
 definition             + typed query results
```

Use Crystal for the server. Reuse standard JSON, URI, YAML, Process, and timing facilities. Keep protocol framing/dispatch, source state, interactive queries, and worker scheduling as a few direct modules; do not introduce one implementation behind an extensible backend interface. The existing worker remains a separate process, with its internal JSON format separate from editor JSON-RPC.

The server owns document mutations in arrival order and captures the appropriate document state when dispatching a query. Compiler work and dependency scanning must not monopolize the editor loop. Use bounded scan batches/yields for index construction and the worker for CPU-heavy compiler work. One serialized protocol writer prevents interleaved frames.

### 2. Implement a deliberate LSP subset

Expose `crystal-lsp --stdio` and `--version`. The initial server capabilities are incremental open/change/save/close synchronization, completion triggered by `.`, hover, and definition. Use UTF-16 positions even when the client offers additional encodings. No unsupported feature is advertised.

Framing uses UTF-8 byte lengths, a 16 KiB header limit, and a 16 MiB message-body limit, documented as implementation bounds. Parse complete frames and validate request shapes; preserve integer/string IDs. Framing failures with no trustworthy boundary close the connection with a stderr explanation. Valid framed JSON errors use the applicable JSON-RPC response. Ignore unknown notifications and return MethodNotFound for unknown requests. Keep compiler and macro output away from the protocol writer.

Track uninitialized, running, and shutdown states. A second initialize is InvalidRequest; ordinary pre-initialize requests receive ServerNotInitialized. Shutdown returns null after cancelling owned work; exit then terminates. Handle EOF and death of the non-null initialize.processId as cleanup paths; do not assume a shell launcher is the editor process. Cancellation completes an outstanding request once with RequestCancelled when acknowledged; an unknown cancellation is harmless. Do not use ContentModified merely because another didChange is queued: ordinary requests are answered for their captured state, and client cancellation determines whether that response is still wanted. [LSP framing, ordering, and cancellation](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#baseProtocol), [initialization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize).

Initially use plaintext hover and plain-text completion insertion, which avoid optional snippet/markup behavior. Return standard Location values for definitions. URI decoding and UTF-16 conversion are centralized, tested with emoji, combining characters, CRLF, and paths containing spaces. Compiler-native columns and Tree-sitter byte offsets are converted using the exact relevant text, never a freshly reread different version.

### 3. Use current buffers and a recovering parser

The document store maps file identity to URI, version, UTF-8 contents, line offsets, syntax tree, and current declarations. Apply a notification's edits to a temporary next state in order, then commit atomically. Older versions are ignored with a log; an invalid range marks the document desynchronized until a full update or reopen. Clamp overlong character positions as the protocol requires. A closed file falls back to disk and invalidates dependent state. New unsaved files work in the syntax path, with semantic unavailability stated until saved.

Use the C Tree-sitter runtime and the existing Crystal grammar, including its external scanner. Compile pinned generated parser/scanner sources so running the server does not need Node.js or a grammar generator. This dependency is necessary because the compiler parser aborts on syntax errors; Crystal's standard library has no equivalent recovering parser. [Tree-sitter recovery nodes](https://tree-sitter.github.io/tree-sitter/using-parsers/queries/1-syntax.html#special-nodes), [Crystal grammar](https://github.com/crystal-lang-tools/tree-sitter-crystal).

Use a narrow Crystal C binding for the parser/tree/node APIs required here, reusing upstream binding definitions and license notices where appropriate. The existing [Crystal binding](https://github.com/crystal-lang-tools/crystal-tree-sitter) is useful prior art, but its high-level setup discovers grammars from a user's tree-sitter CLI configuration. Choose explicit linking/loading of this project's pinned grammar instead of inheriting that runtime setup. Record runtime, grammar, and binding revisions in build metadata, validate ABI compatibility, and keep native ownership/release tests next to the binding. Do not build a generic language-loader API.

Before building feature handlers, test incremental edits and fresh parses of the exact missing-end and trailing-dot cases. Extract only declarations/scopes that the recovered structure supports; do not stretch an uncertain error node across unrelated scopes. If the upstream grammar cannot pass a required case, fix the smallest grammar issue and retain the regression, rather than silently dropping cold recovery from the milestone.

### 4. Bootstrap semantics without a valid application body

When compiler execution is enabled, use the worker's declaration pass on resolvable valid dependency roots found in current literal requires. For the Kemal consumer this includes the `kemal` dependency and its transitive standard-library/dependency sources, without compiling the incomplete application body. Keep application declarations from current recoverable syntax separately. This does not pretend to expand arbitrary macros from an incomplete application. [Compiler declaration pass](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/compiler.cr).

Index declaration ownership, ordinary and generated method names, argument/block restrictions, documentation, inheritance/include relationships supplied by the compiler, and source/expansion provenance. Do not exclude `lib/` or the standard library. Dependency facts are keyed by toolchain, flags, dependency state, consumed-source fingerprints, and bootstrap roots. Dirty dependency edits invalidate affected facts; unresolved replacements remain unknown.

Resolve lexical bindings, annotations, literals, simple assignments, and block restrictions in current source. Propagate a block parameter's type only when applicable declarations agree, and stop propagation after unsupported reassignment or ambiguity. Kemal's generated route methods explicitly restrict their block parameter to HTTP::Server::Context; this is source metadata, not a special rule for `get` or `env`. [Kemal route declarations](https://github.com/kemalcr/kemal/blob/80777f648d03b67e0467dfd43d2e74ebfc1c9595/src/kemal/dsl.cr).

For an unannotated dependency method such as Context#params, use a targeted compiler query instead of introducing a return-flow type checker. Build a compiler AST or internal synthetic source containing the valid dependency roots, a typed receiver derived from declaration metadata, and the supported member call. A typed uninitialized receiver is usable for semantic analysis; generated application code is never emitted or run. Restrict the initial query forms to known receiver types and zero-argument member chains, and return unknown for unsupported forms or failed probes. [Compiler handling of typed uninitialized locals](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/semantic/main_visitor.cr), [unannotated params implementation](https://github.com/kemalcr/kemal/blob/80777f648d03b67e0467dfd43d2e74ebfc1c9595/src/kemal/ext/context.cr).

Run these queries in the background and cache facts by dependency context plus resolved receiver/query, checking compatibility with the current lexical binding and application declarations before reuse. A class reopening, method override, or changed ancestor can alter lookup without changing the receiver type: suppress dependency-only exact members and definition targets when current application declarations may affect that owner/method/ancestor, returning uncertainty until current full-target facts or sufficient current source evidence resolve it. Unknown application macro effects are not evidence that a dependency answer remains exact. Probe results establish facts about that dependency context; they never certify the broken application's semantics. Never publish probe diagnostics as application diagnostics or expose the synthetic source as an editable definition target. Ordinary source locations and available macro generator/invocation locations remain the navigation targets.

An initial query may return current lexical candidates while the missing dependency refinement is queued. Use CompletionList.isIncomplete for an incomplete list and report indexing through standard log messages. Subsequent queries must produce the required Kemal answers within the initial-analysis deadline, without repairing or saving the application. The cold-start test includes this refinement phase; it cannot be passed using a warmed application snapshot. Exact full-target worker facts refine later queries only while their source generation remains compatible.

### 5. Make project configuration small and predictable

Use initialization options as the first configuration mechanism; changes require a server restart. The options are:

| Option | Behavior |
| --- | --- |
| `entrypoint` | Optional path relative to the workspace or absolute path; takes precedence over discovery |
| `compiler.path` | Optional explicit Crystal executable used to verify compatibility and discover its environment; otherwise resolve from the server environment |
| `compiler.enabled` | Default false; explicitly enables analysis that can execute project macros |
| `compiler.flags` | Extra compile flags, default empty |
| `compiler.timeoutMs` | Per-job bound, default 30000 |
| `initialAnalysisTimeoutMs` | Cold dependency indexing/refinement deadline, default 60000 |

Select one workspace folder, otherwise rootUri. Without a root, allow open-document lexical operation. Reject a multi-folder semantic setup with a clear explanation rather than mixing projects; multiple independent server instances can be used outside this milestone.

Target discovery uses the sole shard.yml executable target when present. With no executable target, an existing `src/<shard-name>.cr` supplies library declarations with limited body coverage; it is not a claim that all methods were checked. Multiple targets or an absent conventional library entrypoint require the explicit setting. In the Kemal library acceptance workflow, select `spec/context_spec.cr`. Do not select every edited file as its own implicit program.

Read shard metadata with the standard YAML library. Resolve compiler version and CRYSTAL_PATH on the server's host and compare against the worker's embedded 1.21.1 semantics. Record the resolved context. Do not run shards install automatically or silently substitute an incompatible compiler. Configuration errors leave the protocol session and syntax queries available.

Compiler execution opt-in is needed because even dependency declaration analysis can execute macros. Zed's native project trust remains useful but is not a portable authorization signal delivered through standard LSP. Explain the option once per session, and run no compiler/bootstrap/probe job while disabled. This is distinct from worker timeouts and does not claim sandboxing. [Crystal macro execution](https://crystal-lang.org/api/1.21.0/Crystal/Macros.html), [Zed worktree trust](https://zed.dev/docs/worktree-trust).

### 6. Publish diagnostics only for their source context

Reuse the prerequisite worker's snapshots and validity checks. After an edit, publish bounded local syntax feedback promptly, invalidate prior compiler diagnostics for affected context, and debounce full-target work by 300 ms. Keep one active compiler job and coalesce pending work by purpose: newest full snapshot, newest dependency bootstrap, and newest interactive probe. Prioritize bootstrap/probe work needed for interactive readiness, then the latest diagnostic job; bounded queues replace obsolete work instead of growing with keystrokes. No worker pool is needed.

Before accepting a full-target result, compare the workspace generation, target options, dependency lock, consumed saved sources, and require membership. Match each open file's diagnostic version when supported. Deduplicate overlapping local/parser and compiler diagnostics. Publish replacements, including empty lists where errors disappeared, and clear diagnostics for files no longer in the target. While clearing stale compiler errors, retain current syntax feedback and report checking/unavailable status so an empty list does not imply completed type checking. A failed worker cannot be translated into a successful clean compile.

Register watched-source/shard changes when the client supports dynamic watched-file registration. If it does not, refresh the relevant source/dependency inventory before queries or scheduled analysis, in bounded batches. Recheck consumed file fingerprints and require-glob membership before accepting compiler results. Invalidate on didSave and close as well as didChange. This extends the experiment's fixed-input conditions to ordinary editor activity; arbitrary external macro inputs remain a declared limitation. [LSP diagnostic replacement](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_publishDiagnostics), [watched files](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#workspace_didChangeWatchedFiles).

Timeout/cancellation kills and reaps the worker's owned process group and cannot end unrelated processes. Log stage boundaries, duration, result generation, resolved target, and failure reason. Do not log full source buffers by default. On shutdown or editor loss, cancel timers and workers before exit. A later job can recover from a worker failure without restarting the LSP session.

### 7. Reuse Zed's existing launch adapter

Use the installed Crystal extension and its `crystalline` registration as a documented temporary compatibility alias. Zed's configured binary override bypasses that extension's default executable lookup, so the Crystalline executable itself is unnecessary. `initialize.serverInfo.name` stays `crystal-lsp`. A first-party registration can follow when distribution is in scope. [Zed language-server configuration](https://zed.dev/docs/configuring-languages#configuring-language-servers), [binary override implementation](https://github.com/zed-industries/zed/blob/main/crates/project/src/lsp_store.rs).

The setup guide will provide a concrete configuration using the built binary's actual absolute path:

```json
{
  "languages": {
    "Crystal": { "language_servers": ["crystalline"] }
  },
  "lsp": {
    "crystalline": {
      "binary": {
        "path": "/absolute/path/to/crystal-lsp",
        "arguments": ["--stdio"]
      },
      "initialization_options": {
        "compiler": { "enabled": true }
      }
    }
  }
}
```

The example's compiler enablement is for the chosen trusted workspace. Show how to add an explicit compiler path and focused spec entrypoint. Do not include `...` in the acceptance server list: it would enable other language servers and make attribution unclear. Existing grammar highlighting and editor behavior remain supplied by the extension. Document how to restore the user's prior language-server settings after the disposable smoke test.

### 8. Measure the editor-facing behavior

Use the prerequisite's pinned Kemal revisions and locks. Add a protocol replay harness that launches the real server and sends framed initialize/open/change/query/cancel/shutdown messages. Keep focused tests near each implementation slice; the final replay is an integration check, not the first test of synchronization or recovery.

The acceptance matrix includes cold broken startup; trailing dot and missing end separately and together; renamed block variables and a user-defined equivalent block API; shadowing/ambiguous declarations; dependency/stdlib navigation; generated route provenance; two unsaved-file diagnostics; delayed stale results; Unicode positions; setup failure; and restart with dirty buffers. Use native Zed on a disposable workspace for the final smoke test and record its version, extension version, server identity, and visible outcomes. Protocol traces do not replace that UI evidence.

The first interactive budget is p95 <= 100 ms for each of completion, hover, and definition after required indexing, measured over at least 100 correct requests per feature in a release build on the recorded reference machine. It is an acceptance target, not an achieved performance claim. Measure cold readiness and compiler diagnostic latency separately, including the 300 ms debounce and prerequisite compiler costs. A worker held artificially pending must not delay interactive replies. Record peak server/worker memory and individual samples; compare the pinned Crystalline baseline only under equivalent conditions.

## Risks / Trade-offs

- Compiler experiment fails its correctness gate -> Keep independent session/parser work, but do not mark the LSP milestone complete until the integration is corrected; do not silently remove diagnostics from scope.
- Tree-sitter recovery chooses an unexpected scope -> Keep focused grammar/query regressions and decline uncertain scope resolution.
- Dependency bootstrap omits application-defined macro effects -> Label dependency-only facts, invalidate them when their assumptions change, and rely on current full-target results for application-specific effects.
- Typed probes need compiler adaptation and add cold latency -> Implement the narrow member-chain path early, bound it, and keep the cold Kemal gate mandatory; a failing approach requires a design revision before claiming completion.
- Binding/runtime/grammar ABI drift -> Pin and test the native combination and document reproducible build inputs.
- Cold compile or fallback disk refresh is slow -> Measure those stages, keep them off the query path where possible, and never hide them in warmed latency claims.
- Setup uses the crystalline adapter name -> Explain the compatibility alias and verify the actual process/serverInfo during acceptance.
- Unsaved ECR and arbitrary macro child I/O remain unsupported -> State coverage in setup and failure messages rather than implying that every project file is overlaid.

## Migration Plan

There is no existing server to migrate. Implement the protocol and syntax path first while the compiler prerequisite is completed, then add semantic bootstrap and diagnostics, then verify Zed. Keep the initial setup local to a disposable benchmark workspace. Rollback is stopping this process and restoring the previous Zed language-server configuration; no project source or persistent semantic database is rewritten.

Future changes can add safe references/rename, broader recovery/type coverage, template overlays, richer LSP features, and an independently registered/distributed Zed backend after this milestone is measured and usable.
