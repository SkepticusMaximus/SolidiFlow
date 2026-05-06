# SolidiFlow

A visual, codeless IDE for authoring Ethereum smart contracts. The user draws
contract logic as a flowchart; SolidiFlow generates valid, deployable Solidity.

**Status:** Pre-inception. v0.1 is planned as a [Remix IDE](https://remix.ethereum.org/) **plugin**
with a [ReactFlow](https://reactflow.dev/)-based visual layer. The integration
approach was resolved by [ADR-0001](./docs/adr/0001-remix-integration-approach.md)
following a plugin-API spike: a Remix plugin can render a custom panel,
read/write the workspace, trigger compilation, and listen for compile/deploy
events &mdash; everything SolidiFlow needs &mdash; so a fork is not required.

The visual representation is the source of truth. Solidity is the output. Authoring
happens in the visual layer; the generated code is not expected to be hand-edited.

## Where to start reading

- [`SEED.md`](./SEED.md) — the founding project document. Intent, architecture,
  visual primitives, primary design use cases (CompuToken, CGP Agent), open
  questions, and the milestone target. Note: the seed doc was written before
  the plugin/fork question was resolved and frames v0.1 as a Remix fork; that
  framing has been superseded by ADR-0001. See the status note at the top of
  `SEED.md`.
- [`docs/adr/0001-remix-integration-approach.md`](./docs/adr/0001-remix-integration-approach.md)
  — the resolved Remix-integration ADR.
- [`docs/learning/solidity-for-solidiflow-authors.html`](./docs/learning/solidity-for-solidiflow-authors.html)
  — Solidity primer for the project's "logic-understands, Solidity-doesn't"
  audience.
- [`docs/ir/0001-ir-schema-v0.md`](./docs/ir/0001-ir-schema-v0.md) — v0
  draft of the JSON intermediate representation between the visual layer
  and the Solidity generator.
- [`docs/research/remix-plugin-api-spike.md`](./docs/research/remix-plugin-api-spike.md)
  — research notes that informed ADR-0001.

## Milestone

Functional CGP smart contract skeleton on Ethereum testnet before December 2026.

## License

MIT — see [`LICENSE`](./LICENSE).

## Maintainer

Stevo (SkepticusMaximus).
