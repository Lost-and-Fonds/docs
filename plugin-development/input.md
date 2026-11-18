# Input plugins: find material and preserve it honestly

An **Input** understands some external source — a podcast feed, a video channel, a web collection — and tells Stashd which Items exist and what bytes it can acquire.

The core distinction is between **enumeration** and **custody**. Discovering a URL is not preserving its content; fetching a file into temporary staging is not a committed Vault Asset.

Contract: [`input.wit`](https://github.com/Lost-and-Fonds/plugin-api/blob/main/wit/input.wit).

## Lifecycle at a glance

| Method | Responsibility | Return shape |
| --- | --- | --- |
| `resolve(source, credentials)` | Turn typed user source values into a stable Input identity | `resolved-input` |
| `resolve-delegation(delegation, credentials)` | Accept an opaque handoff from another Input | `resolved-input` or typed unsupported error |
| `discover(request, credentials)` | Enumerate with **host-acknowledged batches** | Unit success after acknowledged terminal batch |
| `acquire(item, options)` | Obtain representations of one discovered Item | Staged artifacts plus complete/partial preservation outcome |

All methods have typed Input plugin errors. Input source/option values are tagged **boolean**, **number** (`s64`) or **text**; their keys are plugin-owned, not universal fields.

### Resolve without acquiring

`resolve` identifies the logical Input. It returns a stable `id`, optional opaque `canonical-reference`, optional item-count and size estimates, and metadata facets. Source identity must be enough for later independent discovery: don't rely on a prior process-local cache.

`resolve-delegation` accepts another Input's opaque delegation reference. This is useful when a website Input finds a video for a video-specific Input. The receiving plugin may return `unsupported`; Core may try other installed Inputs. The delegation's string is not automatically a URL, and Core does not interpret its format.

## Discovery is a durable conversation, not an array

`discover` receives:

- Stable `input-id` and intent (`refresh` or `complete`).
- Plugin-owned typed options.
- Optional run `continuation`, and an optional *completed* `refresh-state` baseline.
- A positive `maximum-items-per-batch` imposed by the host.

The plugin submits each `discovery-batch` through **`input-host.commit-discovery-batch`**. Each batch carries Items plus **either**:

- `more(continuation)` — a resumable checkpoint, or
- `finished(exhaustive(refresh-state))`, `finished(partial(deficiencies))`, or `finished(indeterminate(diagnostic))` — a terminal coverage statement.

This is the important bit: **the host commits Items and the associated progress transition atomically**. An accepted `more` batch survives a later crash; an accepted terminal batch is the durable completion point. The plugin must not send another batch after a terminal acknowledgment.

The batch cap and the negotiated RPC frame size are **independent limits**. Send smaller batches when a large Item would make the encoded message too big; don't probe by sending oversized frames.

An empty terminal batch is permitted. Discovery success **without** an acknowledged terminal batch is a protocol violation. If the acknowledgment was lost, the plugin must not guess whether the host committed it — host durable state wins.

See [Input discovery semantics](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/input-discovery.md).

## What goes inside a discovered Item?

The current `discovered-item` contains:

- `id`: stable Item identity.
- `reference`: opaque and sufficient for a later, independent acquire.
- Optional `delegation` to another Input.
- Optional `size-bytes` with a separate `size-estimated` flag.
- `metadata`: plugin-owned, versioned facets (title, episode date, provider-specific type, etc.).

Do **not** read the old SDK's "required title / role list / upstream state" examples into this record. The new shared contract deliberately leaves domain metadata out of universal fields.

## Exhaustive, partial or indeterminate?

Coverage means *what was enumerated*:

- **Exhaustive:** the plugin can truthfully claim it enumerated the logical Input under the run's intent, options and credentials. Only this terminal outcome may replace/clear the completed refresh baseline.
- **Partial:** the plugin knows some material was missed; include at least one deficiency with diagnostic evidence.
- **Indeterminate:** the plugin cannot honestly assert either exhaustive coverage or a known definite gap.

For a transient failure that should **resume**, retain previously accepted continuation and return a typed execution error. Don't relabel an interrupted run as terminal partial merely because the network failed today.

A deficiency has a disposition: `retryable`, `terminal` or `unknown`. That disposition is a hint about the *gap*, not the same as the plugin error's `retryable` flag for a failed invocation.

## Acquisition: successful and incomplete is possible

`acquire(discovered-item, acquisition-options)` returns an `acquisition-result` containing:

- A list of **completed, host-issued staged-artifact descriptors**.
- A `preservation-outcome`: `complete`, or `partial(deficiencies)` with one or more deficiencies.

Consider a podcast episode whose audio is available but accompanying transcript has vanished. The plugin may return valid staged audio *and* a partial outcome describing the missing transcript. Core must preserve the partial claim alongside adopted material.

If the invocation fails with a plugin error, the host does **not** adopt output simply because some bytes were staged before failure. A partial successful result must never contain a half-written or corrupt temporary artifact.

`complete` is scoped to **this Item and these acquisition options**. It makes no claim that the entire podcast, provider or historical catalogue is complete.

## Example: a feed with a special announcement

A podcast feed might contain 51 numbered episodes and one promotional announcement. An Input should preserve source facts and its classification as **plugin-owned metadata**, not force every RSS item into a generic "episode" label.

An illustrative item value (not literal live-feed data):

~~~json
{
  "id": "feed-item:example-001",
  "reference": "feed-entry:example-001",
  "delegation": null,
  "size-bytes": null,
  "size-estimated": false,
  "metadata": [
    {
      "schema": "org.example.podcast.item@1",
      "json": "{\"kind\":\"announcement\",\"title\":\"Another show is coming\"}"
    }
  ]
}
~~~

The original feed XML itself may be worth preserving as a separate representation/evidence object. Exactly how that maps into a plugin's Item and the Vault is **domain design**, not a universal WIT rule.

## Tests worth writing

- Resume after two acknowledged `more` batches and a process crash.
- A terminal exhaustive batch with zero Items; a partial batch with a deficiency; an indeterminate result.
- Duplicate/unstable source IDs and absent/estimated sizes.
- Lost acknowledgment (do not assume either acceptance or rejection).
- A coherent partial acquisition with one good staged artifact.
- An execution error after staging (no adoption).
- Delegation unsupported by a target Input; stale or missing credentials.
- Frame-limited discovery batches; errors without invented completeness.

[Preservation outcomes](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/input-preservation.md) · [Host capabilities](host-capabilities.md) · [Back to guide](README.md)
