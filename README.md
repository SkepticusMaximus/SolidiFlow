# SolidiFlow

A visual, codeless IDE for authoring Ethereum smart contracts. The user draws
contract logic as a flowchart; SolidiFlow generates valid, deployable Solidity.

**Status:** Pre-inception. v0.1 is planned as a fork of [Remix IDE](https://remix.ethereum.org/)
with a [ReactFlow](https://reactflow.dev/)-based visual layer added on top.

The visual representation is the source of truth. Solidity is the output. Authoring
happens in the visual layer; the generated code is not expected to be hand-edited.

## Where to start reading

- [`SEED.md`](./SEED.md) — the founding project document. Intent, architecture,
  visual primitives, primary design use cases (CompuToken, CGP Agent), open
  questions, and the milestone target.
- [`docs/adr/`](./docs/adr/) — architecture decision records. Start with
  `0001-remix-integration-approach.md` once it lands.

## Milestone

Functional CGP smart contract skeleton on Ethereum testnet before December 2026.

## License

MIT — see [`LICENSE`](./LICENSE).

## Maintainer

Stevo (SkepticusMaximus).
