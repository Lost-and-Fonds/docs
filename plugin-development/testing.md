# Testing, compatibility and troubleshooting

A plugin that works against one happy-path fixture is a promising experiment. A plugin that **fails accurately** when a source changes or a worker crashes is a preservation tool.

Because vNext is being rebuilt, separate **contract tests** from **host integration tests**; don't report the latter as passing just because a WIT schema validates.

## A useful test pyramid

1. **Plugin-owned unit tests:** parsing, provider identity, metadata facets, selection and error mapping. Deterministic fixtures, no live services.
2. **Contract/value tests:** encode and decode exact WIT types and variants; validate discovery/result shapes, staging lifetimes and frame bounds.
3. **Host interaction tests:** fake or test host capabilities (HTTP, credentials, stream reads, staging, helper cancellation), including denial and failures.
4. **Process/packaging tests:** immutable root manifest, component artifact resolution, RPC `hello`, stdout discipline and failure cleanup.
5. **End-to-end tests:** a supported host + SDK, actual job, staged output, Core adoption and durable Vault evidence or a real destination publication.
6. **Live provider smoke tests:** opt-in only, rate-conscious, resilient to source changes and outages. Never make public CI depend on a third-party feed being live.

## Contract verification tools

The [`plugin-api` repository](https://github.com/Lost-and-Fonds/plugin-api) provides:

- [`bin/verify-contract`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/bin/verify-contract) — generates/extracts WIT-derived artefacts twice and compares them to retained schema, then runs contract sensitivity and verification scripts. Its first step requires `wasm-tools`, and its Python-based verification scripts must be available.
- [RPC v1 vectors](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/rpc-v1-vectors.json) — framing and typed-value examples.
- [Package/contract vectors](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/package-contract-vectors.json) and [artifact vectors](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/package-artifact-vectors.json).
- [Helper process vectors](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/helper-process-vectors.json).
- Byte-range and preserved-Asset grant verifiers linked from [`bin/verify-contract`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/bin/verify-contract).

Run the repository's own verifier when changing the **contract**. When changing a **provider**, run provider fixtures, the relevant SDK conformance suite, and an integration test against a compatible host. Merely running Core's old 0.2 suite does not validate 0.18.0.

## Behavioural tests by component

### Input

- An acknowledged `more` batch survives a crash, with Items and continuation accepted atomically.
- Acknowledged terminal batch commits completion; success before terminal acceptance is rejected.
- Exhaustive may replace refresh state; partial and indeterminate don't.
- A transient error retains an earlier durable continuation instead of inventing terminal completeness.
- Partial acquisition returns valid staged bytes **and** explicit deficiencies.
- Plugin error after staging means **no new adoption**.
- Independent `acquire` works after a new process launch.

### Broadcast

- Empty collection, partial last batch and mandatory EOF before success.
- Large result reports are split into accepted non-empty bounded batches.
- Duplicate destination file path is rejected; `not-applicable` means zero file records.
- Final failure discards provisional reporter state, even if remote side effects occurred.
- Multiple Items and Asset reads work without relying on provider ordering or Vault paths.

### Enrichment

- `capabilities` is deterministic and makes **no** remote/Asset-byte reads.
- Duplicate capability IDs or option keys are rejected.
- Changed accepted choices/semantics increment capability revision.
- Stale revision → `unsupported`; invalid caller choices → `invalid-configuration`.
- Derived artifacts carry valid source Asset IDs and activity/version provenance.

### Collection Export

- Small empty or nonempty collections, correct escaping and file type.
- Explicit host artifact-size limit vs RPC framing failure are distinct.
- Export does not depend on HTTP, helper or staging privileges.

## Failure modes authors should recognise

| Symptom | Likely boundary |
| --- | --- |
| Package rejected before process launch | Missing root manifest, bad schema/duplicate keys, unsupported exact contract, invalid artifact |
| Plugin prints debug text and fails startup | stdout is reserved for framed RPC; use stderr or invocation logging |
| Plugin receives an unknown handle after retry | Resources are **invocation-scoped**; don't cache handles between invocations |
| Asset read denied | The Item/Asset wasn't granted for this invocation, or the requested range is invalid |
| Staged bytes disappear after error | Unfinished writers or unreturned completed artifacts are not adopted |
| A discovered item vanished from a refresh | Don't infer successful exhaustive discovery if the provider was incomplete |
| Remote files exist but publication is failed | Reporter state is provisional; remote side effects aren't automatically rolled back |
| An enrichment operation is advertised but fails | Applicability isn't remote availability; execution has its own typed failures |
| "Works with SDK" but fails against host | Check **exact contract identity**, SDK binding and host implementation status |

## Review checklist

Before proposing a plugin as ready:

- [ ] Package declares an exact contract supported by the target host.
- [ ] Every exported lifecycle and imported capability uses the correct **current WIT** shape.
- [ ] No credentials or opaque resource handles leak into durable metadata.
- [ ] No invented Vault paths, arbitrary host shell access or direct database writes.
- [ ] Partial source coverage and partial preservation are honestly reported.
- [ ] Results are bounded and frame-aware; EOF and final commit rules are observed.
- [ ] Retries don't silently duplicate or misreport external effects.
- [ ] Test outputs distinguish **not run**, **failed** and **passed**.
- [ ] A real integration proves host acceptance and actual Vault custody/publication if such claims are made.

[Implementation status](implementation-status.md) · [Back to plugin development](README.md)
