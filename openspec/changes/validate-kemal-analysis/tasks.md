# Tasks

## 1. Reproducible inputs and compiler worker

- [ ] 1.1 Prepare disposable Kemal consumer and library/spec workloads at the design's pinned commit, lock dependency revisions, and record compiler/LLVM and baseline identities; verify preparation can be repeated without selecting newer revisions.
- [ ] 1.2 Add a worker build target using Crystal 1.21.1 compiler sources and a one-request JSON input/output contract with logs on stderr; verify a valid small target returns its analysis context and no application binary is generated.
- [ ] 1.3 Validate request contents, explicit target settings, compiler compatibility, and dependency prerequisites; replace the scaffold's failing placeholder with focused Crystal specs that verify malformed inputs, incompatible versions, and missing dependencies produce explicit non-success outcomes.
- [ ] 1.4 Document the build and one-worker invocation, including supported source coverage and macro execution; verify the documented commands run with the pinned toolchain and preserve the fixture's source files.

## 2. Unsaved source consistency and diagnostics

- [ ] 2.1 Add a frozen entrypoint/required-source overlay at the narrow compiler loader boundary, preserving require order and original paths; verify an entrypoint plus two transitively required unsaved files is analyzed together and an unrelated invalid buffer is not included.
- [ ] 2.2 Add saved-source fingerprints and explicit unsupported-input handling for nonexistent files, ECR buffers, and dirty inputs encountered at known macro file-read/generator boundaries; verify focused regression cases and document the remaining arbitrary-subprocess limitation.
- [ ] 2.3 Return structured syntax/type diagnostics with snapshot provenance and native coordinate conventions; verify shifted unsaved lines, a non-ASCII location, and original paths against equivalent saved fixtures, without rereading stale disk text for excerpts.
- [ ] 2.4 Add end-to-end overlay regression coverage for saved-valid/unsaved-invalid, saved-invalid/unsaved-valid, and an edited macro-generated context-storage type; verify results agree with the saved-content oracle and hashes of original fixture sources remain unchanged.

## 3. Semantic facts and target coverage

- [ ] 3.1 Expose the generated `get` declaration and compiler-resolved context of a selected route-block `env` parameter; verify `HTTP::Server::Context`, preserve expansion/source provenance, and assert no Kemal-specific semantic lookup table is involved.
- [ ] 3.2 Exercise a library method under `src/kemal.cr` and an appropriate focused Kemal spec; verify missing versus instantiated type contexts are reported accurately, and invalid source returns diagnostics without claiming unavailable facts.
- [ ] 3.3 Document the semantic query invocation, result provenance, and target coverage limits; verify both documented target examples and keep the executable assertions with the relevant Crystal specs.

## 4. Driver freshness and worker lifecycle

- [ ] 4.1 Add a small driver that accepts only the current generation, target settings, dependency state, and consumed saved-source fingerprints; verify an old completion cannot replace a newer result after an unsaved dependency edit, saved-source edit, or configuration change.
- [ ] 4.2 Bound each job with a timeout and cancellation, terminate/reap its owned processes, and handle malformed output or abnormal exit explicitly; verify a cancelled result is discarded, owned children do not remain running, and the next valid job succeeds.
- [ ] 4.3 Document driver outcomes and timing boundaries; verify the example distinguishes source errors, incomplete/unsupported inputs, worker failure, and superseded results rather than reporting them as a clean check.

## 5. Feasibility measurements and integration verification

- [ ] 5.1 Run the consumer and focused spec workloads with the design's five empty-cache and five populated-cache fresh-process samples per target; retain individual times, peak worker memory with units/process coverage, input hashes, and machine/toolchain metadata, and verify user-global caches were not cleared.
- [ ] 5.2 Compare supported observable queries with the pinned Crystalline baseline using an existing client/tool or documented manual procedure; record equivalent conditions and explicitly mark unsupported unsaved workloads or unavailable measurements instead of substituting incomparable timings.
- [ ] 5.3 Write the feasibility result record with correctness outcomes, measured costs, a retain/revise/reject recommendation, and the next investigations for incomplete-code recovery, new files, and ECR; verify every claim points to a recorded sample or reproducible case and leave failed required cases visibly unresolved.
- [ ] 5.4 Run `crystal tool format --check src spec`, `crystal spec`, the documented Kemal acceptance procedure, and `openspec validate validate-kemal-analysis --strict`; verify each applicable check passes and separately report pre-existing failures or checks that could not run.
