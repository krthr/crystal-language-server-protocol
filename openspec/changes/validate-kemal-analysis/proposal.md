# Proposal

## Why

The first useful Crystal language server milestone is resilient completion and navigation together with timely, trustworthy diagnostics in Zed. Before choosing its architecture, we need evidence that Crystal's compiler can analyze multiple unsaved files consistently and expose the macro-generated declarations and types used by Kemal.

## What Changes

- Add a focused, executable compiler-analysis prototype using Crystal 1.21.1 in a separate worker process.
- Analyze an explicit entrypoint against one versioned set of unsaved contents for the existing entrypoint and normally required Crystal files, preserving their original paths and require order.
- Return structured diagnostics and selected semantic facts, including Kemal's generated route methods and the type of a route-block parameter.
- Reject superseded results and distinguish source errors from worker failures or unsupported inputs.
- Establish reproducible Kemal application and library/spec workloads with pinned source and dependencies. Compare unsaved analysis with equivalent saved contents and measure elapsed time and worker memory.
- Record a source-backed comparison with Crystalline 0.20.0; use equivalent runtime measurements only where its exposed behavior supports the workload.

### Non-goals

This change does not implement LSP or JSON-RPC, a Zed extension, a recovering parser, a new type checker, incremental semantic compilation, or production completion/navigation handlers. It does not promise overlays for macro-run generator programs, macro file helpers or subprocess inputs, unsaved ECR templates, new files that do not exist on disk, or general project autodetection. Those remain explicit follow-up investigations; this prototype must not silently report them as supported.

## Capabilities

### New Capabilities

- `compiler-analysis`: Analyze a selected Crystal target against versioned source contents, preserve diagnostic and semantic provenance, and expose reproducible correctness and performance evidence.

### Modified Capabilities

None. The project has no existing capability specifications.

## Impact

- Future implementation will add a small Crystal worker, a driver for the feasibility cases, focused Crystal specs, and reproducible benchmark inputs/instructions. It will replace the scaffold's deliberately failing placeholder spec when adding real coverage.
- The worker depends on compiler internals from Crystal 1.21.1 and their LLVM build requirements. The experiment must document this compatibility boundary; discovering another `crystal` executable must not silently substitute its semantics.
- Kemal and its dependencies are benchmark inputs, not runtime dependencies of the language server. No editor changes or user project source rewrites are required.
- Results will determine whether to retain this integration, revise it, or pursue an upstream compiler seam before implementing the broader editor experience.
