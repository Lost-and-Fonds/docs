# Collection Export: a small portable file

Not every integration needs to download media or publish a Vault. Sometimes the user has a selection of references and wants one small portable list: an OPML file, a text playlist, or another interchange format.

That's the purpose of **Collection Export**.

Contract: [`collection-export.wit`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/wit/collection-export.wit).

## One call, bounded input and output

The world exports one method:

~~~wit
export-collection: func(
    exporter: string,
    collection: collection,
    options: list<setting>
) -> result<exported-artifact, plugin-error>;
~~~

The collection contains an optional title and a list of entries; each entry has a `reference` and optional `title`. The plugin chooses what `exporter` and option keys mean.

The result has:

- `filename`
- `media-type`
- inline `contents` as a list of bytes.

Unlike Input, Broadcast or Enrichment, this contract does not provide staged byte streams. The **entire encoded request and result must fit their receiver's advertised RPC frame maximum**. A host may impose an explicit Collection Export artifact size limit, reported via its reserved `limit-exceeded` error. That error is **not** a generic name for exceeding the transport frame size.

## Example: export a few feed links

~~~text
exporter = "opml"
collection:
  title = "Podcasts to follow"
  entries:
    - reference: "https://example.org/show-a.xml"
      title: "Show A"
    - reference: "https://example.org/show-b.xml"
      title: "Show B"
result:
  filename: "podcasts.opml"
  media-type: "text/x-opml"
  contents: (generated OPML bytes)
~~~

This is an illustrative use case, not a built-in exporter or a real executable request. The component does not automatically resolve, fetch, preserve, publish or broadcast those references.

## When *not* to choose Collection Export

- **Lots of media or arbitrarily large results?** Use an appropriate streaming/staging lifecycle instead.
- **Need to discover or download external files?** Use Input.
- **Need access to selected preserved Assets?** Use Broadcast or Enrichment as appropriate.
- **Need an indefinitely resumable multi-step export?** No continuation is specified here.

The Collection Export world imports **`logging-host`** and `plugin-types` only. Do not assume HTTP, staging, helper, credentials or Asset-read capabilities are available.

## Tests

Check empty collections, special characters and escaping in generated formats, correctly sized inline byte output, bad option values, invalid references for your exporter, and deterministic results. Exercise frame-bound input/output without sending deliberately oversized frames.

[Back to plugin development](README.md)
