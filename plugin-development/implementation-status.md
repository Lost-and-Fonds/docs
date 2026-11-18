# Implementation status: the rebuild boundary

**Checked against repository default branches on 18 November 2026.** This is a dated snapshot, not an automated compatibility matrix. Re-check versions before beginning implementation.

| Layer | Observed state | What it means |
| --- | --- | --- |
| [`plugin-api`](https://github.com/Lost-and-Fonds/plugin-api) | Canonical WIT package `stashd:plugin@0.18.0` | This documentation's **target contract**. Four worlds, stream-oriented I/O, bounded discovery/publication, exact manifest identity. |
| [PHP SDK](https://github.com/Lost-and-Fonds/stashd-php-sdk/blob/main/composer.json) | Composer package `stashd/php-sdk` version `0.3.7` requires `stashd/plugin-api ^0.2.2`; its README refers to a 0.2.0 binding | Do **not** assume its classes or minimal example implement 0.18.0. |
| [Core package validator](https://github.com/Lost-and-Fonds/stashd/blob/main/packages/plugin-runtime/src/Package/PackageManifest.php) | Uses the older `plugin.json` / `api_version` / PHP-runtime packaging model and defaults to 0.2/0.1 | A compliant **new** root-level `stashd-plugin.json` cannot be assumed to load on this host revision. |
| [Core's older authoring docs](https://github.com/Lost-and-Fonds/stashd/blob/main/docs/plugins/authoring.md) | Document `prepare → publish → finalize` and old manifest fields | Historical implementation reference only for vNext authors. |
| [This documentation](README.md) | Authoring guide for **0.18.0 contract semantics** | Not an assertion of completed runtime/SDK integration. |

## What has changed in the target contract?

**Compared with the older 0.2-era material:**

- Four worlds are defined: **Input, Broadcast, Enrichment, Collection Export**. This does not prove Core currently runs all four.
- A package uses a root-level `stashd-plugin.json` with **id, version, exact contract identity, and named components** — no old `runtime`, `api_version` or `entrypoint` fields in this canonical schema.
- **Input discovery** sends host-acknowledged batches, not one giant returned item list. It explicitly records exhaustive, partial or indeterminate coverage; acquisition likewise distinguishes complete from partial.
- **Broadcast** exposes `publish` and `operation`. It does **not** expose the older `prepare` or `finalize` lifecycle calls. Selected Items arrive through a bounded collection; file and destination receipts are reported in bounded batches.
- **Enrichment** has locally discoverable capabilities with stable IDs/revisions and a separate execution call.
- **Host I/O** uses opaque, invocation-scoped byte streams and staged writers, rather than paths or inline large payloads.
- Credentials are host-owned references supplied in explicit per-invocation grants.

## What should a new contributor do now?

If you want to help implement vNext **today**, start from the [WIT](https://github.com/Lost-and-Fonds/plugin-api/tree/main/wit) and the corresponding normative protocol documents, then inspect the **actual branch and commit** of the host/SDK you intend to work against. Do not backport old PHP signatures by intuition.

Before advertising a runnable 0.18.0 plugin tutorial, we need evidence of:

1. A host that accepts and loads the canonical 0.18.0 package and named components.
2. A language binding/SDK with the 0.18.0 lifecycle shapes and host resources, or a documented direct RPC implementation.
3. Cross-boundary conformance tests for the relevant world(s).
4. At least one **end-to-end** tested example, with exact version/commit details and honest failures.

These are **documentation readiness gates**, not a claim about an approved roadmap or the state of private/uninspected branches. The compatibility status should be updated as the rebuild progresses.

## Versioning: three separate identities

- `stashd:plugin@0.18.0`: exact **contract** package identity. Host support is exact, not assumed semver compatibility.
- A plugin package's own `version`: its release, independent of the contract identity.
- An SDK's Composer/package version: an authoring implementation's release, which must explicitly support the chosen contract.

RPC v1 `hello` negotiates the **wire protocol and frame size**, **not** the Stashd WIT package version.

[Back to plugin development](README.md)
