# LSP Session

## Purpose

Maintain a reliable editor-to-server session whose messages, document contents, source positions, and lifecycle provide a correct foundation for Crystal language features.

## Protocol References

This capability targets LSP 3.17: [Base Protocol](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#baseProtocol), [Initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize), [Shutdown](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#shutdown), [Exit](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#exit), [Cancellation](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#cancelRequest), [Document Synchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization), and [Position](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#position). Message outcomes follow [JSON-RPC 2.0 sections 4 and 5](https://www.jsonrpc.org/specification#request_object).

## ADDED Requirements

### Requirement: Framed stdio messages

The executable SHALL accept LSP 3.17 messages over stdin and emit responses and notifications over stdout using byte-counted Content-Length framing and UTF-8 JSON. It MUST preserve request IDs, reserve stdout for protocol traffic, and handle partial reads without losing message boundaries.

#### Scenario: Fragmented and adjacent messages

- **WHEN** a multibyte JSON message arrives in several reads followed immediately by another message
- **THEN** both messages are decoded once, with correct byte lengths and matching response IDs
- **AND** diagnostic logs do not appear in protocol output

#### Scenario: Invalid input

- **WHEN** a complete frame contains malformed JSON, an invalid request, unknown request method, or invalid parameters
- **THEN** the server returns the applicable JSON-RPC error without replying to valid notifications
- **AND** an invalid or excessive frame length causes a bounded, logged transport failure rather than unbounded allocation or guessed resynchronization

### Requirement: Truthful initialization and shutdown

The server SHALL implement initialize, initialized, shutdown, and exit according to LSP 3.17. It SHALL identify itself as crystal-lsp, advertise only supported features, reject premature requests, and clean up owned analysis processes when the session ends. Missing compiler setup MUST NOT prevent initialization of available syntax features.

#### Scenario: Session lifecycle

- **WHEN** the client initializes and then sends shutdown followed by exit
- **THEN** initialization returns implemented capabilities, shutdown returns a null result, and exit terminates successfully with no owned workers left running

#### Scenario: Invalid lifecycle order

- **WHEN** a request arrives before initialization, a second initialize arrives, or an ordinary request arrives after shutdown
- **THEN** the server returns the applicable lifecycle error and does not start analysis for that request
- **AND** exit without prior shutdown terminates with a nonzero status

#### Scenario: Editor disappears

- **WHEN** the input stream closes or the known parent process exits
- **THEN** the server releases its session and owned workers without waiting indefinitely for compiler output

### Requirement: Versioned document contents

The server SHALL support didOpen, incremental and full-content didChange events, didSave, and didClose. Ordered changes within a notification SHALL apply to successive document states. Open-buffer contents MUST take precedence over disk, and each accepted change SHALL invalidate affected workspace results.

#### Scenario: Several edits and a close

- **WHEN** an open document receives several ordered range edits, then a full replacement, then closes
- **THEN** queries observe each resulting version correctly and the closed document stops overriding its saved contents

#### Scenario: Invalid synchronization

- **WHEN** a duplicate or older version, an unopened-document change, or an invalid edit range is received
- **THEN** the server logs the synchronization error, avoids partial application of that notification, and avoids returning results from known-desynchronized text
- **AND** a subsequent full-content update or close/open sequence can restore synchronization

### Requirement: Consistent Unicode positions and file identities

The server SHALL use UTF-16 LSP positions, explicitly selecting that supported encoding at initialization, and convert all parser/compiler positions against the corresponding source contents. File URIs SHALL preserve the correct file identity through decoding, lookup, and returned locations.

#### Scenario: Multibyte and escaped paths

- **WHEN** a file path contains spaces or non-ASCII characters and its text contains emoji, accented characters, and CRLF lines before a queried symbol
- **THEN** edits, hover ranges, definition locations, and diagnostics refer to the expected characters and file
- **AND** a character offset beyond the line length is handled according to the LSP Position definition

### Requirement: Cancellation completes the request

The server SHALL process cancellation without blocking behind compiler work. A cancelled outstanding request SHALL receive one terminal response, using RequestCancelled when cancellation is acknowledged. Unknown or already-completed cancellation IDs MUST NOT disrupt other requests.

#### Scenario: Cancellation during analysis

- **WHEN** the client cancels an outstanding request while background analysis is running
- **THEN** the request completes once, unrelated syntax requests continue, and any later completion cannot produce a second response for that ID
