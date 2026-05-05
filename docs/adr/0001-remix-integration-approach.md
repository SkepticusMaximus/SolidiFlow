# ADR-0001: Remix Integration Approach — Plugin vs. Fork Modification

- **Status:** Accepted (2026-05-05) — Option C, plugin first, fork later if needed
- **Date:** 2026-05-05
- **Deciders:** Stevo
- **Context source:** [`SEED.md`](../../SEED.md), Open Question #1
- **Supporting evidence:** [`docs/research/remix-plugin-api-spike.md`](../research/remix-plugin-api-spike.md)

## Context

SolidiFlow v0.1 is committed to building on top of [Remix IDE](https://remix.ethereum.org/)
rather than from scratch. That commitment is made in `SEED.md` and is not
reopened here. What is open is *how* the SolidiFlow visual layer integrates
with Remix.

Two integration strategies are on the table:

1. **Plugin** — implement SolidiFlow's visual flowchart layer as a Remix plugin
   that loads into an unmodified upstream Remix. Distribution is a separate
   plugin package. Users install Remix and add the SolidiFlow plugin.
2. **Fork modification** — fork the Remix repository, modify it in place to
   embed SolidiFlow as a first-class panel, and distribute the resulting fork
   as the SolidiFlow IDE. Users install SolidiFlow as a single product.

This decision shapes everything downstream — repo layout, build tooling,
upgrade discipline, distribution story, and how disruptive future Remix
versions are. Picking the wrong path and reversing it is expensive.

## Decision Drivers

- **Integration depth** — SolidiFlow needs to surface a flowchart panel,
  inject generated Solidity into Remix's file workspace, and react to
  compile/deploy events. How much of that is reachable through Remix's
  plugin API surface?
- **Upstream upgrade discipline** — Remix releases regularly. We want to
  inherit those upgrades, not fight them.
- **Distribution simplicity** — SolidiFlow's target users are not Solidity
  developers; "install Remix, then install plugin X" is friction we'd rather
  avoid for v1.0.
- **Commercial variant optionality** — the seed doc preserves the option of
  a proprietary commercial variant. The integration choice should not
  foreclose that option.
- **Build complexity** — a fork inherits Remix's full build system; a plugin
  is a smaller, independent package.
- **Reversibility** — at this stage we want the cheapest path to a working
  end-to-end demonstration, with the option to deepen integration later.

## Options Considered

### Option A — Plugin

Build SolidiFlow as a Remix plugin loaded by an unmodified upstream Remix.

**Pros**
- Smallest codebase. SolidiFlow owns only the visual-layer code.
- Trivial to inherit Remix upgrades.
- Cleanest separation of concerns; the visual-layer/code-generator pipeline
  is decoupled from the IDE.
- Lowest barrier for Solidity-developer collaborators who already know Remix.

**Cons**
- Bound by whatever Remix's plugin API exposes. If we need to influence the
  file workspace layout, the editor chrome, or compile lifecycle in ways the
  plugin API doesn't support, we are stuck.
- Distribution is two-step ("install Remix, then add this plugin").
- Plugin sandboxing may impose performance or UX constraints (iframe
  isolation, message-passing latency for large diagrams).

### Option B — Fork modification

Fork Remix; embed SolidiFlow as a first-class panel; distribute the resulting
fork as a single product.

**Pros**
- Unlimited integration depth. Anywhere in the IDE is reachable.
- One-step install for end users.
- Strong product identity ("SolidiFlow" rather than "Remix + plugin X").
- Easier to host a proprietary commercial variant later.

**Cons**
- We own a fork of a non-trivial codebase. Upstream merges become recurring
  ongoing work.
- Build complexity inherits Remix's full toolchain.
- Higher day-one engineering cost; less of the early effort is on
  SolidiFlow's actual contribution.

### Option C — Plugin first, fork later if needed

Start as a plugin. If and when the plugin API proves insufficient, migrate
to a fork — keeping the plugin's visual-layer code as a self-contained
React module that can be embedded either way.

**Pros**
- Cheapest path to an end-to-end demonstration.
- Defers the fork-maintenance cost until it's earned.
- Forces the visual-layer code to stay decoupled from Remix internals
  (good architectural discipline regardless of eventual path).
- Keeps reversibility cheap; the visual layer is a portable React module
  in either world.

**Cons**
- We may pay a migration cost later if we hit plugin-API limits.
- Mitigated if we keep the visual layer as a standalone React module
  with a thin Remix-plugin wrapper from day one.

## Spike findings

A desk-research spike against Remix's current plugin engine (canonical repo:
`remix-project-org/remix-plugin`; client lib: `@remixproject/plugin-webview`)
confirms all four capability questions:

- **Custom panel rendering:** plugins set `location: 'sidePanel' | 'mainPanel'`
  in their profile and are loaded into a host iframe — any HTML/CSS/JS,
  including a full React app. Cross-iframe comms via `postMessage`. Modal /
  popout / chrome-injection are *not* part of the public surface, so
  SolidiFlow's UI must fit within sidePanel or mainPanel.
- **File workspace I/O:** the `fileManager` API exposes `readFile`, `writeFile`,
  `rename`, `copyFile`, `mkdir`, `readdir`, `getCurrentFile`, `open`, plus
  events on file changes (`currentFileChanged`, `fileSaved`, `fileAdded`,
  `fileRemoved`, `fileRenamed`). No documented size limits; binary content
  needs base64.
- **Trigger compilation:** the `solidity` plugin exposes `compile(fileName)`,
  `compileWithParameters(sources, settings)`, `setCompilerConfig(settings)`,
  and `getCompilationResult()`. Result matches the Solidity standard JSON
  output (ABI, bytecode, errors, sources).
- **Compile / deploy events:** `client.solidity.on('compilationFinished', …)`
  fires for both success and failure (errors are part of the same payload —
  no separate `compilationFailed` event). Deploy events via
  `client.udapp.on('newTransaction', …)`; receipts returned synchronously by
  `udapp.sendTransaction`.

Caveats worth carrying forward:
- The plugin docs site (`remix-plugin-docs.readthedocs.io`) is stamped 2020;
  master spec on the `remix-project-org/remix-plugin` repo adds methods and
  events not present on the docs site. **Code against master, not readthedocs.**
- First-time cross-plugin calls (e.g. `solidity.compile`) trigger the Plugin
  Manager permission modal — first-run UX cost to plan for.
- `postMessage` size limits and binary-file handling are undocumented; risk
  for very large source maps but not for v0.1 use cases.

Full notes including source URLs in
[`docs/research/remix-plugin-api-spike.md`](../research/remix-plugin-api-spike.md).

## Decision

**Option C — plugin first, fork later if needed.**

All four MVP capabilities are first-class API. A vanilla
`@remixproject/plugin-webview`-based plugin is shippable to any
`remix.ethereum.org` user, avoids maintaining a Remix fork, inherits upstream
upgrades for free, and keeps the visual-layer code as a portable React module
that can be re-embedded into a fork later if the plugin path hits a wall.

A fork is reserved for any of these triggers, not committed to in advance:
modal/chrome integration beyond sidePanel/mainPanel, same-origin React
integration, compile-pipeline hooks not exposed to plugins, or a
single-product distribution story for v1.0+ ("install SolidiFlow" rather than
"install Remix, add this plugin").

## Consequences

Following from Option C:

- The SolidiFlow visual layer is implemented as a standalone React module
  with a clean external interface (props / events for IR in/out, Solidity
  out, file-write requests). This keeps the plugin-vs-fork decision
  reversible.
- We adopt `@remixproject/plugin-webview` as the host-comms client library.
- A plugin manifest (profile name, displayName, location, methods, events)
  is established early.
- Upstream Remix becomes a versioned dependency we test against, but is
  not vendored. Compatibility breaks observed against new Remix versions
  are documented as ADR follow-ups, not silently absorbed.
- We code against the master spec of `remix-project-org/remix-plugin`,
  not the 2020-stamped readthedocs site.
- The intermediate representation (SEED Open Question #2) is designed
  in parallel; it is the boundary between the visual layer and the
  Solidity generator and is unaffected by this decision.

Forking is not currently planned. If any of the trigger conditions in
the Decision section above are hit, we open an ADR-0002 to revisit.

## Follow-ups

- **IR design:** initial draft at
  [`docs/ir/0001-ir-schema-v0.md`](../ir/0001-ir-schema-v0.md).
- **Plugin scaffold:** stand up a minimal `@remixproject/plugin-webview`
  React app that loads into Remix's sidePanel and round-trips a single
  Function Node + State Node through the IR to deployable Solidity
  (the SEED Immediate Next Action #5 prototype).
- **Manifest / profile design:** name, displayName, methods, events, and
  permission scope for the SolidiFlow plugin. Captured in a future ADR.
