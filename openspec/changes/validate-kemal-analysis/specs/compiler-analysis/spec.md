# Compiler Analysis

## Purpose

Provide compiler-grounded diagnostics and semantic facts for a selected Crystal target using versioned source contents, with reproducible evidence of correctness and analysis cost.

## ADDED Requirements

### Requirement: Explicit analysis context

An analysis request SHALL identify its workspace, entrypoint, source generation, compiler compatibility, dependency state, and compile flags. The result SHALL identify the context actually used. Unsupported compiler versions or unavailable analysis prerequisites MUST produce an explicit failure or incomplete outcome.

#### Scenario: Compatible Kemal target
- **WHEN** a request selects a pinned Kemal application or focused spec target with a compatible compiler and installed locked dependencies
- **THEN** the result identifies that target, compiler version, dependency state, flags, and source generation

#### Scenario: Missing prerequisite
- **WHEN** the selected compiler is incompatible or a required dependency is unavailable
- **THEN** the result explains the unavailable prerequisite and does not claim a successful check with no diagnostics

### Requirement: Consistent source overlays

Analysis SHALL use one immutable set of supplied contents for the existing entrypoint and normally required `.cr` files. It MUST preserve normal require resolution and order, leave unrelated buffers outside the target, and leave original project sources unchanged. Coverage SHALL explicitly exclude unsaved macro external inputs, ECR templates, and nonexistent files in this feasibility stage.

#### Scenario: Unsaved contents introduce an error
- **WHEN** the entrypoint and two transitively required files have unsaved contents whose combination introduces a type error absent from disk
- **THEN** analysis reports the error from that combination without saving any source file

#### Scenario: Unsaved contents fix an error
- **WHEN** the corresponding saved sources contain an error that the supplied contents fix
- **THEN** analysis no longer reports that error and agrees with analysis of the equivalent saved fixture

#### Scenario: Unrelated open source
- **WHEN** an unrelated file outside the selected require graph has an invalid unsaved buffer
- **THEN** opening that file does not include it in the target or alter the target's diagnostics

#### Scenario: Unsupported overlay coverage
- **WHEN** a request includes a nonexistent source file or an unsaved ECR template, or analysis encounters a dirty input read through a known unsupported macro path
- **THEN** the result explicitly identifies unsupported coverage instead of silently treating saved contents as the supplied contents

### Requirement: Diagnostic provenance

Diagnostics SHALL refer to the original source identity, the analyzed generation, and positions in the analyzed contents. Any displayed excerpt MUST use those same contents. Syntax and type errors SHALL be diagnostic outcomes; worker crashes, invalid output, and timeouts MUST remain distinguishable from a completed check with no diagnostics.

#### Scenario: Error moves in an unsaved file
- **WHEN** unsaved lines are inserted before an error in a required file
- **THEN** the diagnostic names the original file and locates the error in the updated contents, matching the equivalent saved fixture

#### Scenario: Invalid Crystal source
- **WHEN** a supported source snapshot has a syntax or type error
- **THEN** analysis returns a diagnostic outcome without asserting that all target code was analyzed or returning unconfirmed semantic facts

#### Scenario: Worker failure and recovery
- **WHEN** an analysis worker crashes, produces malformed output, or exceeds its time limit
- **THEN** the driver reports an analysis failure rather than clearing diagnostics as if analysis succeeded
- **AND** a subsequent valid request can complete without restarting the driver

### Requirement: Compiler-grounded semantic facts

For supported Kemal snapshots that complete semantic analysis successfully, analysis SHALL expose a generated route method declaration and the compiler-resolved type of a selected route-block parameter. Each fact SHALL carry its analysis context and distinguish ordinary source locations from macro expansion locations. Missing or ambiguous information MUST be represented explicitly, without hard-coded Kemal semantics.

#### Scenario: Generated route and block parameter
- **WHEN** a supported Kemal application defines a `get` route with an `env` block parameter and completes semantic analysis successfully
- **THEN** compiler analysis exposes the generated `get` declaration and resolves the selected `env` to `HTTP::Server::Context`
- **AND** any location for the generated declaration retains its source or expansion provenance without inventing an editable source range

#### Scenario: Library coverage depends on its target
- **WHEN** a method lacks a typed context under the library entrypoint but is exercised by a selected Kemal spec
- **THEN** the first result reports the missing context explicitly and the spec-target result reports only the contexts confirmed by that analysis

### Requirement: Superseded results are not accepted

The driver SHALL accept a result only for the still-current analysis context and source generation. Changes to an overlaid dependency, a consumed saved source, target settings, or dependency state MUST invalidate an older result even if the entrypoint text is unchanged. Cancelled work MUST NOT become the current result.

#### Scenario: Dependency edit overtakes an older job
- **WHEN** job A starts, an unsaved dependency changes, and job B becomes current before A completes
- **THEN** A cannot replace B or publish diagnostics for the current generation

#### Scenario: Saved input or configuration changes
- **WHEN** a consumed saved source, selected target, compile flags, or dependency lock changes during analysis
- **THEN** the old result is marked superseded and is not accepted as current

### Requirement: Reproducible feasibility evidence

The feasibility run SHALL record immutable workload and toolchain identifiers, correctness outcomes, elapsed analysis times, and peak worker memory with measurement coverage. It SHALL distinguish startup and cache conditions, preserve individual samples, and state unsupported comparisons or missing measurements. No performance improvement SHALL be claimed without comparable evidence.

#### Scenario: Repeated Kemal workloads
- **WHEN** the application and focused spec workloads are evaluated
- **THEN** the report records Kemal and dependency revisions, source contents or hashes, compiler and baseline versions, machine information, flags, cache conditions, and individual timing and memory samples
- **AND** correctness is checked against equivalent saved contents before interpreting performance

#### Scenario: Baseline cannot express the workload
- **WHEN** the pinned baseline cannot perform an equivalent simultaneous unsaved-buffer query
- **THEN** the report identifies that coverage difference and does not present saved-only analysis timing as an equivalent unsaved-analysis result
