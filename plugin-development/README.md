# Plugin development

Start with the [`plugin-api`](https://github.com/Lost-and-Fonds/plugin-api)
contract and [`plugin-sdk`](https://github.com/Lost-and-Fonds/plugin-sdk) PHP
8.5 SDK. A provider package declares its manifest and entrypoint under
`stashd-plugin/`, depends on the SDK, and never imports core source.

Provider repositories contain provider fixtures and focused tests. PostgreSQL
lifecycle and native-runtime integration tests remain in core.
