# Architecture

Stashd keeps the Vault canonical. Broadcasts are disposable, rebuildable views
over preserved assets. Core owns persistence, provenance, fixity, filesystem
authority, jobs, and generic plugin lifecycle. Providers own provider protocols.

Native PHP plugins are production. The language-neutral contract is maintained
in [`plugin-api`](https://github.com/Lost-and-Fonds/plugin-api), the authoring
surface in [`plugin-sdk`](https://github.com/Lost-and-Fonds/plugin-sdk), and
provider implementations in their own repositories.
