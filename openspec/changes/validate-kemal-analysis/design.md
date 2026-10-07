# Design

## Context

See [proposal.md](proposal.md) for motivation and [the capability spec](specs/compiler-analysis/spec.md) for the behavior contract. The repository currently contains a Crystal library scaffold and a deliberately failing placeholder spec. There is no server implementation to preserve.

Crystal 1.21.1 provides in-memory entrypoint sources, semantic analysis without application code generation, and compiler data for declarations and instantiated methods. Required sources are normally read from disk. Its context tool can report multiple type contexts or no context for an uncalled method. [Compiler](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/compiler.cr), [require loader](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/semantic/semantic_visitor.cr), [context tool](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/tools/context.cr).

Crystalline 0.20.0 contains a source-override map at its compiler require boundary, but source inspection found no production caller supplying that map to workspace compilation. Its approach is useful prior art; runtime comparisons still need to verify observable behavior. [Compiler integration](https://github.com/elbywan/crystalline/blob/a5f6f1bb9d55decf507138da7b2c47628ee15328/src/crystalline/ext/compiler.cr), [workspace](https://github.com/elbywan/crystalline/blob/a5f6f1bb9d55decf507138da7b2c47628ee15328/src/crystalline/workspace.cr).

This design is warranted by compiler compatibility, subprocess isolation, source consistency, and performance uncertainty. It specifies an experiment, not measured results.

## Goals / Non-Goals

**Goals:**

- Prove supported unsaved analysis agrees with equivalent saved analysis, without changing the project's sources.
- Expose enough compiler data to evaluate future generated-method navigation and typed completion.
- Establish reproducible costs and a clear decision about this compiler integration.

**Non-Goals:**

- No general analysis backend framework, persistent semantic database, file-watcher service, or worker pool.
- No claim that a complete compiler AST can be produced from incomplete code. Recovery parsing remains a separate investigation.
- No transparent virtual filesystem for arbitrary macro subprocesses. External inputs remain fixed within each controlled benchmark run.

## Decisions

### 1. One pinned compiler worker per analysis job

Use a Crystal executable with a worker mode and a small driver. Send one JSON request through stdin and return one JSON result through stdout; send logs to stderr. This is an internal experiment format, not JSON-RPC or LSP. Use Crystal's standard JSON, process, timing, and digest facilities rather than a protocol framework or new runtime shard.

Build against Crystal 1.21.1 compiler sources and document the required LLVM setup. Identify the embedded compiler version in every result and reject a request for incompatible semantics. A different compiler found on PATH does not change the embedded worker. One fresh process and `Program` per job makes failure recovery and memory lifetime explicit. Keep semantic analysis off the driver process and disable application code generation; macro generators can still compile and execute.

Alternative: a stock compiler subprocess needs less integration code but lacks a multi-file buffer overlay and direct typed-AST access. A shadow checkout can make relative reads see edited files, but changes absolute source identity and may affect path-sensitive macros. Neither is the preferred starting point for this experiment. A long-lived compiler service is deferred until measurements justify its additional lifecycle and invalidation work.

### 2. Preserve the require graph while substituting source contents

A request contains an explicit workspace and entrypoint, generation number, compile flags, dependency-lock fingerprint, and a frozen map of original absolute source paths to contents and buffer versions. Resolve existing paths consistently at the boundary; validate their existence and valid UTF-8 contents. Dependencies outside the workspace are allowed when they are part of the selected target.

Provide the entrypoint through `Compiler::Source`. At the shared required-file read, consult the map before disk, preserving normal lookup, require order, and once-only inclusion. Reuse the upstream loader and Crystalline's override idea with the smallest version-specific adaptation; preserve applicable attribution if copying upstream code. Do not pass every dirty buffer as an extra compiler entrypoint: that changes inclusion and ordering.

Record a content fingerprint for each saved Crystal source actually consumed. Before accepting a result, the driver checks those fingerprints, the current generation, flags, and dependency lock. A change invalidates the result; it does not get presented as a current successful analysis. This provides a bounded consistency check without building a watcher or persistent cache. External macro inputs and environment are fixed benchmark prerequisites, not covered by this saved-source check.

Ordinary Crystal macros defined in the supported source graph are included. The boundary does not cover new unsaved files, require-glob membership changes, dirty macro-run generator sources, or dirty data read by macro file helpers/subprocesses. Reject nonexistent and non-`.cr` overlay inputs up front. At known macro file-read and generator-entry boundaries, detect dirty inputs and return unsupported coverage instead of using disk silently. Arbitrary child-program I/O remains outside the contract and must be stated in every benchmark configuration.

ECR is a concrete follow-up: `ECR.embed` runs a separate processor which reads the template file. A source override inside the compiler cannot change that child process's read. [Macro execution](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/macros/macros.cr), [macro file helpers](https://github.com/crystal-lang/crystal/blob/1.21.1/src/compiler/crystal/macros/methods.cr), [ECR macro](https://github.com/crystal-lang/crystal/blob/1.21.1/src/ecr/macros.cr), [ECR processor](https://github.com/crystal-lang/crystal/blob/1.21.1/src/ecr/processor.cr).

### 3. Keep result provenance and failure states explicit

Return the request context, outcome, structured compiler diagnostics, consumed-source fingerprints, requested semantic facts, and measurements. Distinguish completed analysis, source-error diagnostics, unsupported or incomplete analysis, failed analysis, and superseded results. Do not turn a crash, timeout, missing dependency, or malformed worker response into an empty successful diagnostic list.

Preserve the compiler's original filenames, native line/column information, and expansion trace where available. The prototype result must identify its coordinate convention and test a non-ASCII location; conversion to negotiated LSP positions belongs in a later protocol change. Build any source excerpt from the analyzed buffer. Do not reuse human-readable compiler excerpts that reread the saved file, and do not fabricate end ranges or editable locations for generated code.

The driver accepts only its current generation and context. A dependency edit increments the workspace generation even when the entrypoint did not change. On timeout or cancellation, terminate and reap the worker and its owned child processes; a late response cannot be accepted. Start the next job normally after a failure. Exercise out-of-order result rejection with controlled test completions rather than adding parallel workers to production behavior.

### 4. Query a small set of real semantic facts

On a valid Kemal application, retrieve the generated `get` declaration and the compiler context at a selected route-block `env` variable. Use compiler objects and tools; do not recognize Kemal identifiers through a curated list. Keep the declaration's macro/source provenance. If several contexts exist, return them rather than selecting an arbitrary one.

Compare the same library method under `src/kemal.cr` and a focused Kemal spec that exercises it. An uninstantiated method has missing typed context, not proof of an error-free body. Reuse the compiler's top-level declaration pass where useful, but do not make a second pass mandatory when full semantic results already contain the requested data. On invalid source, return diagnostics and explicit unavailable facts; partial typed recovery is outside this change.

### 5. Use a small, repeatable Kemal experiment

Pin the following inputs in the implementation's benchmark metadata:

| Input | Selection |
| --- | --- |
| Compiler | Crystal 1.21.1; record executable/source identity and LLVM version |
| Kemal | Commit `80777f648d03b67e0467dfd43d2e74ebfc1c9595`; verify it when preparing the fixture |
| Baseline | Crystalline 0.20.0, commit `a5f6f1bb9d55decf507138da7b2c47628ee15328`; record the actual binary/build identity |
| Dependencies | Resolve once, retain the resulting lockfile and immutable revisions, then reuse them |
| Target A | A small Kemal consumer with an entrypoint and two transitively required helper/route files |
| Target B | Kemal's `spec/context_spec.cr`, with `src/kemal.cr` used separately to demonstrate coverage differences |

Keep dependency preparation separate from timed runs. Use disposable benchmark checkouts; never save buffers into a user's project to obtain the oracle. Compare the overlaid result with a separate fixture containing those exact saved contents, normalizing only the known fixture root when comparing paths. Include saved-valid/unsaved-invalid and saved-invalid/unsaved-valid cases, shifted diagnostic lines, an unrelated dirty buffer, and a changed macro-generated context-storage type. Kemal already supplies the relevant route and storage mechanisms. [Routing DSL](https://github.com/kemalcr/kemal/blob/80777f648d03b67e0467dfd43d2e74ebfc1c9595/src/kemal/dsl.cr), [context spec](https://github.com/kemalcr/kemal/blob/80777f648d03b67e0467dfd43d2e74ebfc1c9595/spec/context_spec.cr).

For each valid target, record five fresh-process samples using an isolated empty compiler cache per sample, then five fresh-process samples reusing a populated experiment cache. Here, warm means disk caches are populated, not that typed compiler state is reused. Never clear the user's global compiler cache. Record total launch-to-result time, semantic-analysis time, and other available compiler stages without inventing unavailable timing boundaries.

On the initial macOS host, use the platform's process resource measurement to record peak worker resident memory, with units and whether child processes are included. Record driver overhead separately if measured; do not compare one process's memory to another system's total. Preserve raw samples and report median and maximum; ten samples are not a defensible tail-latency study. Record CPU, OS, architecture, flags, environment inputs, and exact workload contents or hashes.

First verify correctness against the saved compiler oracle. Compare Crystalline on equivalent observable queries when possible, recording its own cold/warm conditions. Mark inaccessible or unsupported unsaved queries explicitly; no protocol implementation is required in this change just to automate that baseline. Existing clients/tools or a documented manual procedure suffice. Do not equate a CLI compiler timing with end-to-end LSP request latency.

Finish with a concise result record recommending retain, revise, or reject this integration, backed by measured results and remaining limits. Slow but correct analysis is a useful result. A failed required correctness case remains a blocker; do not weaken the spec to produce a favorable verdict.

## Risks / Trade-offs

- Compiler internals and LLVM increase build coupling -> Pin one compiler version, isolate adaptations, and document the build used. Additional versions require separate verification.
- A fresh compile per job can be expensive -> Measure it before introducing reuse; future interactive requests must remain independent of compiler completion.
- File overlays do not virtualize all macro I/O -> Declare the supported graph, reject known dirty external inputs, and keep other external inputs fixed for this experiment.
- The strict parser still rejects unfinished code -> Return honest diagnostics; evaluate the recovering syntax layer in a later change, including cold starts on already-broken projects.
- A worker process is isolation, not a filesystem sandbox -> Limit this experiment to the selected controlled fixtures; macro execution remains a documented input to analysis.
- Kemal does not establish large-workspace scalability -> Treat it as the first baseline and avoid extrapolating measured results to larger projects.

## Migration Plan

No production deployment or data migration is involved. Add the prototype and focused verification alongside the scaffold, document a reproducible invocation, and leave the normal user project sources untouched. If the experiment fails, retain its evidence and revise the integration before building server features; the prototype can be removed without changing an editor integration or persistent user data.
