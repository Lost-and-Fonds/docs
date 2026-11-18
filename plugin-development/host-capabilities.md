# Host capabilities: ask for access, don't reach around the host

The plugin process isn't an extension of Core's memory or filesystem. The **host mediates** HTTP, credentials, preserved bytes, staging and helper processes through WIT imports.

This is where much of Stashd's safety — and many of its surprising edge cases — lives.

Source: [`io.wit`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/wit/io.wit) and [shared semantics](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/shared-values.md).

## Capabilities by world

| Capability | Input | Broadcast | Enrichment | Collection Export |
| --- | :---: | :---: | :---: | :---: |
| HTTP via `http-host` | Yes | Yes | Yes | No |
| Staged writers / shared I/O via `io-host` | Yes | Yes | Yes | No |
| Approved helpers via `io-host.start-helper` | Yes | Yes | Yes | No |
| Progress via `progress-host` | Yes | Yes | Yes | No |
| Logging via `logging-host` | Yes | Yes | Yes | Yes |
| Read already-preserved Assets | Not through an Input Asset-open API | Yes, delivered Items only | Yes, supplied context only | No |
| Raw credential access | Input-only exceptional route | No | No | No |

This lists **contract imports**, not a guarantee the current host implements every world.

## Byte streams and staging

Use **`io-host.open-staging-area`** to create a staged writer with optional media type and metadata. Write bytes in bounded chunks, then call **`finish()`**. Only `finish()` creates an immutable host-issued **`staged-artifact`** descriptor.

A staged descriptor contains an opaque `reference`, optional media type, host-computed `size-bytes`, and metadata. **It is not a filename**. A completed artifact is visible only for the invocation and is adopted only if returned by an accepted successful lifecycle result. Dropping an unfinished/poisoned writer discards its partial output; a write error poisons the writer permanently.

A completed artifact can be reopened during its own invocation using **`open-staged-artifact(artifact, offset, length)`**. Host-validated exact descriptor equality matters: don't reconstruct altered descriptors.

Readers return `some(non-empty bytes)` for a chunk and **`none` at sticky EOF**. `some([])` is invalid protocol behaviour. A byte range starts at offset, may specify an optional length, and clamps beyond EOF. Offset equal to size is valid and returns immediate EOF; offset greater than size is denied. The logical byte range is independent of RPC frame chunk size.

[Staged output rules](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/staging.md) · [Byte ranges](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/byte-ranges.md)

## HTTP without unlimited networking

Open the host HTTP client, then submit a request with method, URL, headers, optional **credential-reference**, and optional owned **byte-stream** body. The response has status, headers and a byte-stream body.

Hosts apply runtime URL and redirect policy and re-check granted credentials for each use. A request may be rejected as denied, credential-unavailable, authentication-rejected, rate-limited, body-failed, unavailable or failed. Preserve the distinction when mapping to a lifecycle result.

Do not assume the plugin process has direct networking, that redirects automatically inherit authentication, or that a streamed request body is always replayable on redirect. Credential material must never be written to URLs, metadata, settings or logs.

The HTTP contract accepts syntactically valid method tokens; it is **not** restricted to the small hardcoded method list from the older API.

## Credentials: references are not secrets

A package component may declare named **credential slots**. For an invocation, the host supplies a list of bindings: stable slot name plus opaque credential reference.

- A reference alone is **not** authorization.
- Grants and availability are checked at use time, including after rotation/revocation.
- HTTP and approved helpers mediate ordinary credential use.
- **Only Input** has `input-host.open-credential` for protocols the standard mediation cannot express.
- Broadcast and Enrichment do **not** receive raw-secret access.
- Never put secrets or credential references into metadata facets, source values, destination configuration or result receipts.

Slot identity is **(package ID, component ID, slot name)**. Renaming a slot isn't the same as changing its label; don't silently borrow the old credential binding.

[Package credential contract](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/plugin-package.md).

## Helper processes

For established tools such as a media converter, use **`io-host.start-helper(name, args, input, output, credentials)`**. The host decides which named helpers are approved and creates an invocation-scoped `helper-process` resource.

- `input` is an optional transferred byte stream for stdin.
- `output` is an optional transferred staged writer for helper stdout.
- `next-event()` returns arbitrary output bytes, cumulative stdout activity (when an output writer is attached), and exactly one terminal event.
- Terminal may be normal exit (including **non-zero** code), cancelled, timed-out or host/runtime failed.
- On normal exit, a valid owned writer is returned to the plugin for `finish()`. On cancellation/failure/drop, it is discarded.
- Stderr output remains available as events. Chunks are **bytes, not lines**.
- The helper is stopped and reaped at invocation end; do not depend on a detached long-running process.

Do not invent a universal helper executable path, package lockfile format or arbitrary shell command interface: those are **host packaging policy**, not promised by the 0.18.0 WIT.

[Helper process contract](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/helper-process.md).

## Metadata, logging and progress

A metadata facet has exactly **`schema: string`** and **`json: string`**. The JSON string must represent an object, without duplicate nested keys. Core validates that structure and a non-empty schema string; it does **not** interpret your custom fields. As the producer, use a stable, versioned schema identity and publish your facet semantics.

Logging takes a diagnostic message. Progress has `stage` and `fraction` (none for indeterminate; otherwise finite in **[0, 1]**). Invalid progress isn't clamped; it fails the invocation. Don't infer durable business state from progress text.

## Typed failure versus broken protocol

A normal provider problem should become a **typed lifecycle error** (such as `rate-limited`, `authentication`, `invalid-data`, `unavailable` or `failed`). Input additionally distinguishes denied/unavailable credential cases; Enrichment adds `invalid-configuration`.

A malformed RPC message, invalid resource handle, forged staged descriptor or invalid metadata structure is different: it is **contract/protocol failure**, not an opportunity to return a friendly provider error and continue. Treat the channel as unusable.

On the wire, `s64`/`u64` values are **canonical decimal JSON strings**, `list<u8>` values are arrays of byte integers (not base64), and WIT variants use their defined tag/value shape. Let a conforming SDK handle this where possible; avoid hand-rolled RPC framing without conformance tests.

[RPC v1](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/rpc-v1.md) · [Back to plugin development](README.md)
