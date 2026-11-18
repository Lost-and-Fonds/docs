# Packages, components and process execution

A Stashd plugin **package** is one deployable identity. It contains one or more **components**, each implementing one of the four canonical WIT worlds.

The **0.18.0 contract** defines how an already-unpacked package is validated and launched. It does **not** prescribe an OCI installer, Composer layout, portable interpreter, or user-facing install command. Current Core's older installer must not be mistaken for a 0.18.0 implementation.

## The one authoritative manifest

Place `stashd-plugin.json` at the **root of the unpacked package**:

~~~text
example-package/
├── stashd-plugin.json
└── bin/
    ├── archive-input
    └── archive-broadcast
~~~

Example:

~~~json
{
  "id": "org.example.archive",
  "version": "0.1.0",
  "contract": "stashd:plugin@0.18.0",
  "components": {
    "source": {
      "world": "input-world",
      "artifact": "bin/archive-input",
      "credential_slots": {
        "account": {
          "label": "Source account",
          "description": "Only needed for private collections",
          "required": false
        }
      }
    },
    "publisher": {
      "world": "broadcast-world",
      "artifact": "bin/archive-broadcast"
    }
  }
}
~~~

A manifest does **not** declare any raw password or stored credential reference. The slot declares a requirement; Core chooses and grants an opaque reference for an invocation.

The accepted top-level fields are exactly `id`, `version`, `contract` and `components`. The new canonical schema does **not** use the old `plugin.json`, `runtime`, `api_version`, `entrypoint`, or `kind` fields.

The component worlds are exactly `input-world`, `broadcast-world`, `enrichment-world` and `collection-export-world`. Each declares `world` and `artifact`, with optional `credential_slots`. Components may use one program if their world/lifecycle identity keeps them unambiguous. The manifest's package `version` is independent of its exact `contract` identity.

See [generated schema](https://github.com/Lost-and-Fonds/plugin-api/blob/main/schema/plugin-package.schema.json) and [normative package contract](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/plugin-package.md).

## What happens before anything executes?

The host validates the immutable unpacked package in order:

1. Read the **exact** root manifest filename.
2. Parse UTF-8 JSON; reject duplicate keys at **any** nesting level.
3. Validate schema and package semantics.
4. Check support for the **exact** declared contract identity.
5. Resolve every component artifact against the **same package root**, checking regular-file type, symlinks and containment.
6. Complete component structural checks before launch.

An invalid package is rejected **before** any lifecycle method. The host does not guess another filename, search for an entrypoint or use an old SDK convention to rescue it.

Component artifact paths are portable logical paths using `/`. Absolute paths, backslashes, colons and traversal above the root are invalid. Lexically normalisable `.` and `..` segments may be accepted when they remain contained; symlinks are allowed **only** when resolution stays within the immutable package. Prefer simple relative paths anyway.

See [artifact resolution](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/package-artifact.md).

## How a native component starts

The host launches the resolved artifact as a program with the package root as working directory (or a sandbox-equivalent path). A component must not depend on absolute host paths, inherited environment variables or Core-specific command-line arguments.

For the canonical native RPC v1 binding:

- **stdin**: only host-to-plugin framed RPC traffic. Never interactive input.
- **stdout**: only plugin-to-host framed RPC traffic. No banners, warnings, debug prints or whitespace.
- **stderr**: ordinary diagnostics; can be captured or discarded.
- **first stdout frame**: plugin's RPC `hello` request. Only after successful exchange may a lifecycle begin.

A frame is a four-byte unsigned big-endian payload length followed by UTF-8 JSON. `hello` exchanges the supported RPC version and each side's maximum **receivable encoded payload size**. It does **not** negotiate the WIT package version. One lifecycle invocation is active per process; imported host calls can occur during that invocation with correlated request/response IDs.

Processes **may** be reused, so correctness cannot depend on previous invocation-local state. Host resource handles, credentials, streams and grants expire with their invocation even in a reused process. Channel corruption or process exit before accepted lifecycle completion is **transport failure**, not an ordinary plugin error.

[Component execution](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/component-execution.md) · [RPC v1](https://github.com/Lost-and-Fonds/plugin-api/blob/main/protocol/rpc-v1.md)

## Versioning and installation caution

A plugin package can declare `contract: "stashd:plugin@0.18.0"` even if the host doesn't support it; **schema validity is not executable compatibility**. The contract requires hosts to reject unsupported identities rather than quietly downgrade.

As of the [checked implementation snapshot](implementation-status.md), Core still reads the older packaging form. Do not follow its existing installation commands with a new-style manifest until host support has been verified at the exact branch/commit.

[Back to plugin development](README.md)
