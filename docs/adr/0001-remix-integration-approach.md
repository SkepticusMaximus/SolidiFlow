# ADR-0001: Remix Integration Approach — Plugin vs. Fork Modification

- **Status:** Proposed (decision deferred pending plugin-API spike)
- **Date:** 2026-05-05
- **Deciders:** Stevo
- **Context source:** [`SEED.md`](../../SEED.md), Open Question #1

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

## Decision

**Deferred.** Recommendation pending a time-boxed spike (≤ 2 days) into
Remix's current plugin API, with the following question:

> Can a Remix plugin (a) render a custom panel of arbitrary size,
> (b) read and write files in the Remix workspace, (c) trigger compilation
> of generated Solidity, and (d) listen for compile/deploy events?

If all four are yes with acceptable performance, **Option C (plugin first,
fork later if needed)** is the leading recommendation. If one or more is no
or pluginAPI-restricted, **Option B (fork modification)** becomes the
recommendation.

## Consequences

Independent of the eventual choice, the following are required:

- The SolidiFlow visual layer is implemented as a standalone React module
  with a clean external interface (props/events for IR in/out, Solidity
  out, file-write requests). This keeps the plugin-vs-fork decision
  reversible.
- The intermediate representation (SEED Open Question #2) is designed
  before the visual layer code lands. The IR is the boundary between
  the visual layer and the Solidity generator, and is unaffected by the
  Remix integration choice.

If Option C / plugin path is taken:
- Plugin metadata, manifest, and packaging conventions need to be
  established early.
- Upstream Remix becomes a versioned dependency we test against.

If Option B / fork path is taken:
- A `upstream` git remote tracks Remix's main branch; merges happen on a
  cadence (proposal: monthly).
- The SolidiFlow product README documents which Remix version the fork
  is currently rebased onto.

## Follow-ups

- **Spike task:** evaluate the four plugin-API questions above against
  current Remix. Result captured in a follow-up note appended to this
  ADR or in a new ADR-0002 if the conclusion is non-obvious.
- **IR design:** open ADR-0003 once the plugin/fork decision is made;
  the IR design proceeds in parallel and does not block on this ADR.
