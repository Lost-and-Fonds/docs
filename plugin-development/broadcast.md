# Broadcast plugins: make preserved material useful

A **Broadcast** publishes caller-selected preserved Items to a destination: a podcast feed, a filesystem view, a media server, a remote object store, or another service.

The direction is deliberate: **the Vault stays authoritative; a Broadcast is a view or delivery of its selected content**. The plugin never gets the Vault's database or host filesystem path.

Contract: [`broadcast.wit`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/wit/broadcast.wit).

## The two methods (not four)

The **0.18.0** lifecycle contains:

- `publish(request, configuration, credentials) -> publication`
- `operation(request, credentials) -> operation-result`

There is **no** `prepare` or `finalize` method in the current contract. Those appear in old 0.2-era SDK/Core documentation. Transformation, packaging, remote upload, and any destination-specific commit work happen **inside** `publish`. `operation` is a distinct, explicitly requested interactive destination action, not a second publication phase.

## Publish is a bounded stream of selected Items

`publish-request` contains a host-owned:

- **`item-collection`** reader, with `next(max-items)`,
- **`publication-reporter`** for result batches,
- positive **`maximum-report-records-per-batch`**.

`destination-configuration` separately contains non-secret plugin-owned settings for the specific host-selected destination. Per-invocation credential bindings are also separate.

`item-collection.next(...)` returns **non-empty batches** or **`none` at EOF**. Items are stable for that invocation, delivered once, in **host-defined** order; do not assume chronological or provider order. `next(0)` is invalid. The host may send fewer than requested to fit the negotiated frame. Each successfully delivered Item grants access to its listed preserved Assets for **that invocation** only.

A plugin **must consume to EOF before reporting success** — even if it decides not to publish some selected Items. Successful `publish` before EOF is a contract violation.

## Items are deliberately generic

An Item has stable `id`, a list of preserved Assets, and plugin-owned metadata facets. A preserved Asset has stable ID, opaque reference, optional media type, actual byte size, and facets.

The Broadcast can use `broadcast-host.open-asset(reference, offset, length)` to get an invocation-scoped bounded stream over an Asset that was actually delivered. This is **not** a Vault path, public URL or durable bearer token.

Titles, dates, podcast episode numbers and similar fields belong to plugin-owned facets. Core does not impose one universal media schema on books, videos, podcasts and web pages.

## Reporting publication results

A successful Broadcast may create a **staged artifact** (for example, generated RSS) and/or perform remote side effects. It reports outcome through two bounded reporter methods:

- `report-files(files)` — filesystem-relative paths, optionally associated with an Item ID and Asset ID.
- `report-destination-metadata(metadata)` — plugin-owned opaque receipts, destination revisions, object IDs, etc.

Every report batch must contain **1 through the advertised maximum** records and must also fit the RPC frame limit. Reports are **provisional** until the final successful `publish` result is accepted. They do not persist as partial publication success on failure.

The returned `publication` contains:

- optional completed **`artifact`**;
- `files` status: **`complete`** or **`not-applicable`**.

Use `complete` when the accepted file records are an **exhaustive** set of the canonical filesystem-relative outputs (an empty complete set is allowed). Paths must not repeat within the successful publication. Use `not-applicable` for non-filesystem publication; this requires **zero** accepted file records.

Relative paths are in the **destination namespace**, not host or Vault paths. A successful publication can report destination metadata even with `files: not-applicable`.

See [publication results](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/broadcast-publication.md).

## A podcast feed example

~~~text
publish
  ├─ read selected Vault Items in batches
  ├─ inspect podcast metadata facets
  ├─ open granted audio/caption Assets if bytes are required
  ├─ write a standards-compliant RSS representation to staging
  ├─ optionally report destination file mappings / receipts
  ├─ read until the collection returns EOF
  └─ return one completed staged feed artifact + file status
~~~

That's a **conceptual algorithm**, not a claim a particular host can serve those generated enclosure URLs today. Valid RSS alone does not make a functioning podcast: enclosure reachability, authentication, MIME types, and byte-range playback must be verified in real clients.

## Failure and retry: one critical nuance

On failure, Stashd discards provisional reporter state and does not adopt newly staged outputs as a successful publication. **But the protocol cannot roll back external side effects**: a server may already have received some uploaded objects.

Make repeated publication safe where you can: stable external object identities, replacement/upsert semantics, explicit reconciliation and truthful receipts. The contract does **not** define cross-invocation Broadcast continuation or remote rollback.

## Interactive operations

`operation` accepts a plugin-defined operation name and typed settings/payload, and returns choices and values. It can support "list available libraries" or "validate this destination" flows, subject to the host capabilities and credentials granted for that invocation.

An operation is **not** a secretly restarted `publish`. Keep its names stable and its effects clear.

## Test checklist

- Empty collection (`none` on first read), partial final batch and host-defined ordering.
- Failure if `publish` returns success before collection EOF.
- Asset read denied before Item delivery; valid range and EOF cases afterward.
- Reporter batch too large, duplicate file path, and corrected re-report.
- `complete` with exhaustive paths vs `not-applicable` with no paths.
- Remote upload succeeds but final RPC fails; retry without false success.
- No secrets in destination settings or metadata receipts.

[Collection and EOF rules](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/broadcast-collection.md) · [Host capabilities](host-capabilities.md) · [Back to guide](README.md)
