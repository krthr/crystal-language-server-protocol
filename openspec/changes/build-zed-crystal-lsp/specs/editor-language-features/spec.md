# Editor Language Features

## Purpose

Provide useful Crystal completion, hover, and definition navigation while developers edit unfinished source, with explicit boundaries between known declarations and confirmed semantic information.

## Protocol References

Responses follow LSP 3.17 [Completion](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_completion), [Hover](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover), and [Go to Definition](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_definition). Only client-supported response forms and markup are used. The internal compiler contract is supplied by the prerequisite [compiler-analysis spec](../../../validate-kemal-analysis/specs/compiler-analysis/spec.md).

## ADDED Requirements

### Requirement: Features survive recoverable syntax damage

Completion, hover, and definition queries SHALL use current buffer contents and continue serving recoverable scopes when other source regions are incomplete. Supported results SHALL include lexical bindings, declarations, and members of resolvable receiver types. An unsupported or ambiguous receiver MUST yield explicit uncertainty or a valid empty result rather than a fabricated exact answer.

#### Scenario: Unfinished receiver expression

- **WHEN** a configured Kemal route is edited to end in `env.` or `env.params.`
- **THEN** completion includes context members such as `request`, `response`, and `params`, or parameter-parser members such as `url`, respectively, once the required dependency facts are available
- **AND** hover on the route's `env` and navigation on the preceding `params` expression remain usable

#### Scenario: Unrelated missing end

- **WHEN** a block terminator is removed outside an otherwise recoverable queried scope
- **THEN** that scope's local completion, hover, and definition queries still return applicable results
- **AND** the broken region is not treated as evidence that a different scope owns those bindings

### Requirement: Cold startup supports the Kemal editing case

With compiler analysis enabled and valid installed dependencies, a fresh server SHALL provide the required Kemal route queries for an already-incomplete application after bounded initial analysis, without any prior successful application snapshot. The initial indexing state and an inability to obtain required dependency facts SHALL be observable.

#### Scenario: Broken application is the first document

- **WHEN** a fresh session opens a Kemal route containing a trailing receiver dot, a missing closing end, or both, and has no cached application analysis
- **THEN** the required route completion, parameter hover, and definition navigation become available within the configured initial-analysis deadline
- **AND** the user does not need to repair or save the application first

### Requirement: Inference follows source evidence

Receiver and block-parameter information SHALL be derived from lexical scope, current source, and compatible declaration or compiler facts. The server MUST NOT hard-code Kemal route names or variable names as semantic rules. Reassignment, shadowing, conflicting overloads, and dependency changes SHALL invalidate unsupported assumptions.

#### Scenario: Equivalent user-defined block API

- **WHEN** a user-defined method has the same explicit block restriction as the route API, the block variable has another name, and the application is incomplete
- **THEN** the same resolution rules provide the corresponding parameter information without framework-specific handling

#### Scenario: Shadowing or ambiguity

- **WHEN** a local is shadowed or reassigned, or candidate call declarations disagree about the block parameter type
- **THEN** the server stops presenting the previous single type as certain and avoids receiver-incompatible member suggestions marked as exact

#### Scenario: Application reopens a dependency class

- **WHEN** current application source reopens HTTP::Server::Context and overrides params while the application remains incomplete
- **THEN** dependency-only ParamParser members and the old dependency definition target are not presented as exact answers merely because env still has the same receiver type
- **AND** the server uses sufficient current source evidence or reports uncertainty until current full-target analysis resolves the override

### Requirement: Navigation includes dependencies and generated declarations

Definition queries SHALL resolve supported lexical references and known declarations across application, dependency, and standard-library sources. Macro-generated symbols SHALL lead to a meaningful original generator or invocation location when available, with honest hover provenance. Ambiguous targets MUST be returned as candidates or left unresolved rather than selected by name alone.

#### Scenario: Kemal dependency and standard library

- **WHEN** the user navigates from `env.params` or a known standard-library type in the benchmark application
- **THEN** the result identifies its real declaration in the matching dependency or standard-library source

#### Scenario: Generated route method

- **WHEN** the user requests a definition or hover for the generated `get` method
- **THEN** the server provides an available original generator/invocation location and describes the generated declaration's provenance
- **AND** it does not return an invented editable file or a stale range from another source generation

### Requirement: Protocol results respect current scope and client capabilities

Completion edits SHALL replace only the intended token range. Hover SHALL use supported markup and source ranges. Empty or unresolved queries SHALL return valid LSP results. A source-derived fallback or dependency-only type result MUST NOT be presented as successful analysis of the entire application.

#### Scenario: Apply a completion

- **WHEN** the client accepts a member completion after text containing Unicode characters
- **THEN** the edit changes only the intended member token and preserves surrounding text

#### Scenario: Plaintext client and missing information

- **WHEN** a client supports only plaintext hover and a query has partial or no type information
- **THEN** the response uses plaintext, states relevant uncertainty for a partial answer, or returns null for no answer

### Requirement: Interactive queries remain independent of compilation

The server SHALL answer from available current syntax and compatible semantic facts without waiting for a running full-project compilation. Missing refinements SHALL be scheduled separately. On the pinned Kemal traces after initial indexing, completion, hover, and definition requests SHALL each meet a measured p95 response time of at most 100 ms on the recorded reference machine.

#### Scenario: Compiler remains busy

- **WHEN** compiler work is deliberately delayed while at least 100 measured requests of each interactive feature run against the indexed Kemal workload
- **THEN** those requests complete independently, satisfy the response-time budget, and do not wait for the delayed compiler job
- **AND** raw samples, hardware, build mode, cache state, and result correctness are recorded
