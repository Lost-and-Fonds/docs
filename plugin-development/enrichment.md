# Enrichment plugins: add meaning or new representations

An **Enrichment** component looks at already-preserved Item/Asset context and may produce new metadata or derived Assets, such as a transcript, translated subtitles, OCR text or artwork.

Enrichment should not secretly replace the original. Stashd's preservation model needs to know **what was derived from what**, by which plugin activity and version.

Contract: [`enrichment.wit`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/wit/enrichment.wit).

## Two stages: advertise, then execute

### 1. `capabilities(context) -> list<capability>`

This call is **deterministic, local, credentialless applicability discovery**. Given an Item's ID, preserved Asset descriptors and plugin-owned metadata, describe operations the component could *attempt*.

Each capability has:

- stable plugin-owned **`id`**;
- **`revision`** for the accepted configuration/execution meaning;
- zero or more **`configuration-option`** descriptors.

An option has a unique key, label, required flag and **non-empty** list of unique choice values. Selections are **single-choice opaque strings**, not arbitrary form fields. A capability with no options is a fixed operation.

An example capability descriptor (illustrative values):

~~~json
{
  "id": "transcribe",
  "revision": "2",
  "options": [
    {
      "key": "language",
      "label": "Language",
      "required": false,
      "choices": [
        {"value": "en", "label": "English"},
        {"value": "fr", "label": "French"}
      ]
    }
  ]
}
~~~

**Do not** inspect Asset bytes, call the network, consult secrets, check service uptime or run helpers in `capabilities`. An empty list simply means no applicable operation. Being advertised does *not* promise that execution will succeed.

### 2. `enrich(context, capability-id, capability-revision, configuration, credentials)`

This executes one previously advertised operation. Unlike capability discovery, it can use granted HTTP, helpers, staged output and Asset reads.

- Unknown/inapplicable capability or stale revision → **`unsupported`**.
- Duplicate/unknown option keys, missing required selections, unsupported choice → **`invalid-configuration`**.
- Remote failure/authentication/rate limits → the corresponding typed execution error.

If accepted configuration meaning changes, **change the capability revision**. Mere presentation-label changes do not require a revision bump. Don't advertise two descriptors with the same ID and different revisions in one result.

## What an enrichment can return

`enrichment-result` contains plugin-owned **metadata facets** and/or **derived Assets**.

Each derived Asset contains:

- a completed host-issued **`staged-artifact`**;
- `derived-from` — IDs of source Assets supplied in context;
- `activity` — plugin-owned description/identity of derivation operation;
- `activity-version` — that operation's version.

Use `enrichment-host.open-asset(reference, offset, length)` to read granted source bytes; never infer a physical Vault path from a reference.

For example, "transcribe episode audio" might return a VTT transcript staged artifact and a provenance relationship to the original audio Asset. If transcription fails, don't present an empty artifact as a successful transcript.

## Keep discovery lightweight

This split makes enrichment capability listings reliable and cheap. A UI can ask "what operations could apply to this Item?" without triggering model downloads, consuming API quota or revealing sensitive material.

But it also means *remote service-specific* choices must be locally describable in the capability contract; `capabilities` is **not** a live remote model catalogue. If choices really depend on a remote service, solve that through a separate product flow or a stable local selection contract; do not hide a network call inside discovery.

## Tests worth writing

- Same context and component version produce the same capability descriptors.
- No Asset reads, credentials, helpers or network in `capabilities`.
- Empty capability list; duplicate IDs; duplicate option keys or choice values; zero-choice options.
- Revision changes on configuration semantics; stale revision → `unsupported`.
- Invalid selections → `invalid-configuration`, without partial acceptance.
- Derived Asset provenance names only valid source Asset IDs.
- Staging failure does not produce an adopted derived Asset.

[Normative capability rules](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/enrichment-capabilities.md) · [Host capabilities](host-capabilities.md) · [Back to guide](README.md)
