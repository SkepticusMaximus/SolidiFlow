# ADR-0002: Authoring-time constraint, not output-time validation

- **Status:** Accepted (2026-05-07)
- **Date:** 2026-05-07
- **Deciders:** Stevo
- **Context source:** Design philosophy articulated in conversation following [ADR-0001](./0001-remix-integration-approach.md) and the [IR v0 draft](../ir/0001-ir-schema-v0.md). Captured here so it's a touchstone for future UX and architecture decisions, not a piece of folklore.

## Context

The seed document names SolidiFlow's target user as someone who *"understands the logic of what they want a contract to do but is not a Solidity developer."* That commitment carries a hidden cost the seed doc implies but does not state: the burden of *knowing Solidity* — its keywords, its conventions, the standards (ERC-20, etc.) layered on top — must be borne by the tool, not by the user. Anywhere that burden leaks out onto the user, SolidiFlow has failed its own brief.

Three classes of "hidden Solidity knowledge" are particularly dangerous if surfaced to the user:

1. **Language keywords and syntax.** Already covered by the visual layer's existence — the user drags nodes, the generator emits keywords.
2. **Domain conventions.** Names like `oracle`, `owner`, `treasury`, `paused` aren't enforced by the compiler but are universally understood by Solidity developers. A non-developer doesn't know these are conventional and may reach for less-conventional names that compile fine but read as foreign to anyone reviewing the contract.
3. **External standards.** ERC-20 requires functions named exactly `balanceOf`, `totalSupply`, `transfer`, `transferFrom`, `approve`, `allowance` with exact signatures. A non-developer asked to "make an ERC-20 token" cannot reasonably be expected to know this list, let alone the exact signatures.

If the visual layer surfaces these as choices the user has to navigate, the tool is no friendlier than writing Solidity by hand.

## Decision

SolidiFlow commits to the following principle:

> **Any error or non-compliance catchable in the visual layer is caught there. The generator never emits code that requires the user to know Solidity, its conventions, or its external standards in order to author, read, or fix.**

The user's experience is *"draw the logic; the rest is taken care of."* Where that's not literally possible — some choices genuinely need the user's input — the tool surfaces a small set of well-explained options with a defensible default already selected.

## Corollaries

The principle has four concrete consequences for SolidiFlow's design.

### C-1. Templates / scaffolds for known shapes are first-class

SolidiFlow ships built-in templates for common contract patterns. *"New ERC-20 token"* drops a full ERC-20-compatible scaffold onto the canvas with all six required functions pre-named, pre-typed, and pre-wired. The user fills in the token's display name, symbol, and any custom logic (mint/burn rules, fee structure, etc.) — never the `balanceOf` boilerplate. Other initial templates: ownable contract, basic governance / vesting, multisig wallet. Templates are entries in the visual toolbox alongside the nine primitive node types.

### C-2. Sane defaults at every property panel

Wherever the user has to choose — visibility, mutability, type, storage modifiers — the panel pre-selects the most common-and-defensible value:

- New Function Node: `external` visibility, `nonpayable` mutability.
- New State Node: `internal` visibility, no `constant` or `immutable`.
- Visibility / mutability selectors list options in order of common use, not alphabetical.
- Each option carries a one-line tooltip explaining what flipping it changes.

The user can override every default. Defaults are not constraints; they're the rest position.

### C-3. Live constraint checking as the diagram is drawn

The visual layer surfaces problems the moment they exist, not at compile time:

- A Token Flow Arrow with no token selected — highlight.
- A Modifier Node attached to a Function Node but referencing a Condition Node that doesn't exist — highlight.
- A Phase transition with no guard condition — warning (not error; sometimes intentional).
- A Function Node attached to a Phase Node but missing a state variable the auto-guard requires — highlight.
- A `payable` Function with no use of `msg.value` — warning.

Errors caught at authoring time are vastly cheaper to fix than errors caught at compile time, and radically cheaper than errors caught after deployment.

### C-4. Canonical generator output

Statement ordering and formatting in the emitted Solidity follow strict canonical conventions, regardless of the order in which nodes were dragged onto the canvas. Every contract emits in this order: licence comment, pragma, imports, NatSpec block, contract declaration with inheritance, then within the contract — type declarations (enums, structs), state variables, events, modifiers, constructor, then functions ordered by visibility (`external` &rarr; `public` &rarr; `internal` &rarr; `private`). The user never has to know this convention exists. The output is always reviewable and diff-friendly.

## Resolution of previously deferred IR open questions

The [IR v0 draft](../ir/0001-ir-schema-v0.md) named three questions as deferred pending more design grounding. This ADR resolves all three, in the same direction.

**Q-A: Self-token Token Flow auto-detection.** *Resolved: auto-detect with override.* When the user draws a Token Flow Arrow whose source/target is the contract's own ERC-20 balance, the generator emits the internal form (`_mint` / `_burn` / `_transfer`). A property-panel toggle lets the user force the external form when needed — a rare case that should never be the default.

**Q-B: Phase-scoped Function auto-guard.** *Resolved: auto-inject with override.* When a Function Node is attached to a Phase Node, the generator auto-injects a `require(currentPhase == Phase.X, "...")` at the top of the function body. A property-panel flag (`disablePhaseGuard: true`) lets the user opt out for cross-phase functions.

**Q-C: Constructor as its own node type.** *Resolved: dedicated node type.* The IR gains a Constructor Node primitive in v0.1. Modelling it as a Function Node named `"constructor"` would force the user to know that constructors are special; a dedicated type lets the editor surface immutables, inherited constructor calls, and one-shot semantics naturally.

## Consequences

- The plugin scaffold (SEED Immediate Action #5) starts with the principle in place. Defaults, validation hooks, and template support are wired into the foundation rather than retrofitted later.
- The IR schema v0.1 grows a Constructor Node primitive (Q-C). The IR doc receives a follow-up update to reflect this.
- Templates require their own JSON shape — likely a top-level `templates[]` collection in a SolidiFlow installation, distinct from project documents. Specifics deferred to a future ADR when the template authoring story is in scope.
- Live constraint checking implies a separate validation layer between the IR and the generator. A future ADR will define its scope and the validation rules' canonical list.
- Documentation gains a new audience — *"what error did SolidiFlow's validator catch and what should I do about it?"* — that the primer will eventually need to address.

## Follow-ups

- **IR v0.1 update:** add Constructor Node primitive; remove the three resolved questions from the open-questions list, replaced by pointers to this ADR.
- **ADR-0001 follow-ups section:** add a reference to this ADR.
- **Primer update:** the "Three deferred IR design questions, restated" section becomes "previously deferred, now resolved", with each question's "Recommendation" rewritten as a "Decision (ADR-0002)".
- **Plugin scaffold:** the validation seam and template entry-point shape are designed in early so the foundation supports them by construction.
