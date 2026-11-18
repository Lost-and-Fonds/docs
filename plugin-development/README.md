# Develop a Stashd plugin

**Bring something into Stashd, do something useful with preserved material, or send it somewhere new.** A plugin is where that integration-specific knowledge belongs. Core owns the archive, its integrity, the work queue, and the permissions; your plugin owns the peculiarities of the source or destination.

This is the approachable guide to the **vNext plugin contract, `stashd:plugin@0.18.0`**. You do not need to understand the RPC wire protocol before understanding how a plugin works.

> **Rebuild in progress (checked 18 November 2026).** The canonical contract has advanced to 0.18.0, but the checked Core and PHP SDK still describe the earlier 0.2-era interface. Examples here illustrate the **new contract** and are **not a claim that a 0.18.0 plugin can currently be installed or run**. Start with [implementation status](implementation-status.md) before using this as an installation tutorial.

## Choose where to start

| You want to… | Read |
| --- | --- |
| Understand the system and sketch a first plugin | [Your first plugin](first-plugin.md) |
| Package a plugin and understand process launch | [Packages and execution](packages-and-execution.md) |
| Fetch/discover and preserve external material | [Input plugins](input.md) |
| Publish Vault content to a feed, folder, or service | [Broadcast plugins](broadcast.md) |
| Add transcripts, OCR, artwork, or derived representations | [Enrichment plugins](enrichment.md) |
| Export a small list of references in a portable format | [Collection Export plugins](collection-export.md) |
| Use HTTP, credentials, staging, streams, and helpers safely | [Host capabilities](host-capabilities.md) |
| Test difficult cases, failures, and compatibility | [Testing and troubleshooting](testing.md) |
| Find out what actually runs during the rebuild | [Implementation status](implementation-status.md) |

## The mental model

~~~text
External source → Input → Stashd Vault → Broadcast → destination
                               |
                               └→ Enrichment → metadata or derived Assets

Small list of references → Collection Export → interchange file
~~~

Those arrows describe *responsibilities*, not a universal automatic workflow or an implemented UI. An Input discovers Items and stages acquired bytes; **Core** decides what becomes authoritative Vault material. A Broadcast sees caller-selected preserved Items through a bounded reader. Enrichment works on supplied Item/Asset context. Collection Export is a separate, bounded one-shot interchange facility.

A **package** can contain several named **components**, and each component implements exactly one of the four WIT worlds. It is normal to combine related Input and Broadcast components in one package.

## Trust boundaries, in one paragraph

Plugins don't own the Vault. They get invocation-scoped host capabilities to access only the bytes, credentials, HTTP destinations, and helper tools the host grants. Finished staged outputs are not preserved merely because the plugin wrote them; only valid outputs from successful lifecycle results can be adopted. Plugin-specific metadata stays plugin-specific rather than becoming a new universal Core schema.

## Where the truth lives

- [`plugin-api` WIT](https://github.com/Lost-and-Fonds/plugin-api/tree/main/wit) — **normative** worlds, types, functions and host imports.
- [`plugin-api` protocol documents](https://github.com/Lost-and-Fonds/plugin-api/tree/main/protocol) — **normative** lifecycle and transport semantics.
- [Generated package schema](https://github.com/Lost-and-Fonds/plugin-api/blob/main/schema/plugin-package.schema.json) — manifest shape for the contract.
- [Stashd Core](https://github.com/Lost-and-Fonds/stashd) — host implementation, installation and application integration.
- [PHP SDK](https://github.com/Lost-and-Fonds/stashd-php-sdk) — language-specific authoring API, which is **currently behind the vNext contract**.

If these guides disagree with WIT or a normative protocol document, **the contract wins**. If the contract describes something the current host does not implement, **the feature is specified, not shipped**. Help us keep that distinction clear.
