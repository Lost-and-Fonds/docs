# Your first plugin: from idea to contract-shaped design

Imagine you want to preserve an old podcast. The feed is a valuable historical source, its episodes are separate preserved works, and you may later want to make your archived copies playable again.

That's a useful way to learn Stashd's component model without mistaking "I fetched some bytes" for "I preserved an archive".

> **Before coding:** the 0.18.0 contract is specified but is not yet proven runnable through the checked PHP SDK/Core combination. This is a **conceptual walkthrough with real contract shapes**, not a copy/paste executable project. See [implementation status](implementation-status.md).

## 1. Pick a world

| Requirement | Component world | Important question |
| --- | --- | --- |
| Read a podcast feed and acquire episodes | `input-world` | What can I discover, and what can I actually preserve? |
| Generate a new feed over Vault material | `broadcast-world` | What Items did Core select, and how will this destination publish them? |
| Generate a transcript for an existing Asset | `enrichment-world` | Does this capability apply, and which new Asset/metadata did it produce? |
| Export a small list of episode links | `collection-export-world` | What one bounded interchange file should be returned? |

You can implement several components in one package. But don't put preservation, publication and enrichment into one vague "run" method; their failure and provenance rules are different.

## 2. Describe your package

A minimal **illustrative** package for both podcast Input and Broadcast might be:

~~~json
{
  "id": "org.example.podcast",
  "version": "0.1.0",
  "contract": "stashd:plugin@0.18.0",
  "components": {
    "podcast-input": {
      "world": "input-world",
      "artifact": "bin/podcast-input"
    },
    "podcast-broadcast": {
      "world": "broadcast-world",
      "artifact": "bin/podcast-broadcast"
    }
  }
}
~~~

The file is at the **unpacked package root**, named **`stashd-plugin.json`**. The two `artifact` values are hypothetical package-relative programs; this manifest by itself is **not an installable plugin**. See [packages and execution](packages-and-execution.md) for loading and launch rules.

## 3. Choose stable identity and metadata

Consider a feed with one ordinary episode and one announcement. Don't assume each feed item is an episode just because an RSS parser returned it. A discovered Item has a **stable ID**, an **opaque reference** that a later independent `acquire` can understand, optional size information, and plugin-owned metadata facets.

One illustrative facet is:

~~~json
{
  "schema": "org.example.podcast.item@1",
  "json": "{\"title\":\"Morning Powder\",\"entry_type\":\"episode\",\"guid\":\"example-guid\"}"
}
~~~

The outer `json` field is itself a **string containing a JSON object**, not an arbitrary nested object in the WIT value. The schema name belongs to the plugin; Core validates structure but does not infer the meaning of `entry_type` or `guid`. The contents above are illustrative, not a real YLNT catalogue record.

Use references durable across invocation restarts. Never make later acquisition depend on an in-memory index created by a previous `discover` call.

## 4. Sketch the Input lifecycle

~~~text
source fields
  └─ resolve → stable Input identity
       └─ discover(request)
            ├─ commit batch + continuation → acknowledged by host
            ├─ commit batch + continuation → acknowledged by host
            └─ commit final batch + coverage → acknowledged by host
                 └─ acquire(discovered Item, options)
                      └─ staged artifact(s) + complete/partial outcome
                           └─ Core validates and adopts into Vault
~~~

The plugin does **not** return the discovery catalogue as one giant response, and it does **not** write directly into the Vault. [Input guide](input.md) covers interrupted batches and partial preservation.

## 5. Sketch the Broadcast lifecycle

~~~text
publish(request, destination configuration, credentials)
  ├─ read caller-selected Items in bounded batches
  ├─ open granted Asset bytes only when needed
  ├─ stage generated RSS (or publish to destination)
  ├─ report filesystem paths / destination receipts in bounded batches
  ├─ consume Item collection to EOF (required for success)
  └─ return final publication (one commit point)
~~~

This is not a round trip through the original feed: Broadcast produces **a view of what the Vault actually holds**. A source can contain a bad date or unavailable enclosure; the preserved source evidence and any cleaned publication are different things.

[Broadcast guide](broadcast.md) explains publication reporting and failure recovery.

## 6. Decide what failure looks like

Ask four separate questions:

1. **Did the invocation execute coherently?** If not, return the relevant typed plugin error.
2. **Was discovery exhaustive?** If not, report partial or indeterminate coverage without silently claiming a complete catalogue.
3. **Did acquisition preserve everything intended for this Item?** Partial acquisition can still return valid, adoptable artifacts *and* honest deficiencies.
4. **Did publication finish over the complete selected collection?** Broadcast success requires the collection's EOF and an accepted final result.

These distinctions are not bureaucracy; they're how archives avoid claiming to possess things that were never captured.

## 7. Build tests before a full integration

Start with small deterministic fixtures: one source, two discovered Items, an interrupted discovery continuation, one intact media Asset, and one missing supplementary representation. Test typed errors, incomplete outcomes and host-grant denial.

When the host and SDK support the target contract, add the actual package, process, RPC, helper and Vault-adoption tests. [Testing guide](testing.md) contains the checklist and conformance references.

**Next:** [Input plugins](input.md), [host capabilities](host-capabilities.md), then [packaging](packages-and-execution.md).

[Back to plugin development](README.md)
