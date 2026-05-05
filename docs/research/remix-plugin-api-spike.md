# Remix Plugin API Spike — Research Notes

- **Date:** 2026-05-05
- **Question owner:** [`docs/adr/0001-remix-integration-approach.md`](../adr/0001-remix-integration-approach.md)
- **Status:** Complete; informs ADR-0001 Decision (Option C)
- **Method:** desk research; no local Remix install yet

## Headline

The Remix plugin API IS sufficient for SolidiFlow's MVP needs. A plugin loaded
into vanilla `remix.ethereum.org` can render a custom panel, read/write the
workspace, trigger Solidity compilation, and subscribe to compile/deploy
events. A fork is not required to ship v0.

## Per-question findings

### 1. Custom panel rendering — Yes

Plugins set `location: 'sidePanel' | 'mainPanel'` in their profile and are
loaded into a host iframe (any HTML/CSS/JS, including a full React app).
Recommended client lib: `@remixproject/plugin-webview` (`plugin-iframe` works
but is deprecated in favor of webview). Cross-iframe comms via `postMessage`;
same-origin React injection into Remix's tree is not exposed. Modals/popups
outside `sidePanel`/`mainPanel` are not in the public profile surface.

Sources:
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/engine/core/doc/tutorial/3-hosted-plugin.html>
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/plugin/iframe/README.html>
- <https://remix-ide.readthedocs.io/en/latest/plugin_manager.html>

### 2. File workspace I/O — Yes

`fileManager` API: `readFile`, `writeFile`, `rename`, `copyFile`, `mkdir`,
`readdir`, `getCurrentFile`, `open`. Events: `currentFileChanged`,
`fileSaved`, `fileAdded`, `fileRemoved`, `fileRenamed`, `fileClosed`,
`noFileSelected`. Event list is richer in the master branch than in the 2020
docs. Content as string; no documented size limit; binary files would need
base64.

Sources:
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/api/doc/file-system.html>
- <https://raw.githubusercontent.com/remix-project-org/remix-plugin/master/packages/api/doc/file-system.md>

### 3. Trigger compilation — Yes

`solidity` plugin exposes `compile(fileName)`,
`compileWithParameters(sources, settings)`, `setCompilerConfig(settings)`,
`getCompilationResult()`. Result matches Solidity standard JSON output (ABI,
bytecode, errors, sources).

Sources:
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/api/doc/solidity.html>
- <https://raw.githubusercontent.com/remix-project-org/remix-plugin/master/packages/api/doc/solidity.md>

### 4. Compile / deploy events — Yes

`client.solidity.on('compilationFinished', (fileName, source, langVer, data) => …)`
fires for both success and failure — errors are part of the same payload, so
there is no separate `compilationFailed` event.

Deploy lifecycle: `client.udapp.on('newTransaction', tx => …)`. Receipts are
returned by `udapp.sendTransaction`.

Sources:
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/api/doc/solidity.html#events>
- <https://remix-plugin-docs.readthedocs.io/en/latest/plugin/packages/api/doc/udapp.html>

## Caveats / unknowns

- The `remix-ide.readthedocs.io/en/latest/plugin_api.html` URL referenced in
  early scoping notes 404s — that page no longer exists. Remix IDE docs now
  point at the repo README directly.
- Canonical repo is **`remix-project-org/remix-plugin`**, not
  `ethereum/remix-plugin` — the latter appears in older docs but git history
  shows the active org is `remix-project-org`.
- `remix-plugin-docs` site is stamped 2020 / revision `1142413d`. Master spec
  adds events and methods listed above — code against master, not readthedocs.
- Every first-time cross-plugin call triggers a Plugin Manager permission
  modal; first-run UX cost to plan for.
- Panel sizing is host-controlled (no arbitrary popouts); `mainPanel` is the
  closest thing to a full-canvas surface.
- Binary / large-file handling and exact `postMessage` size limits are
  undocumented — a risk for very large source maps but not a v0.1 concern.

## Recommendation

**Option C — plugin first, fork later if needed.** All four MVP capabilities
are first-class API. Building a vanilla `@remixproject/plugin-webview`-based
plugin keeps SolidiFlow shippable to any `remix.ethereum.org` user, avoids
maintaining a Remix fork, and inherits upstream upgrades. Reserve a fork only
if we later need same-origin React integration, custom IDE chrome beyond
sidePanel/mainPanel, or compile-pipeline hooks not exposed to plugins.

This recommendation is adopted in
[`docs/adr/0001-remix-integration-approach.md`](../adr/0001-remix-integration-approach.md).
