# SolidiFlow
## Project Seed Document

**Status:** Pre-inception. No code. No repository. This document is the founding artefact.
**Date:** May 2026
**Author:** Stevo (SkepticusMaximus)
**Repository:** TBD — GitHub repo setup is the immediate next action from this document.

---

## What SolidiFlow Is

SolidiFlow is a visual, codeless IDE for writing Ethereum smart contracts. It targets
people who understand the *logic* of what they want a contract to do but are not
Solidity developers. The user draws their contract logic as a flowchart; SolidiFlow
generates valid, deployable Solidity source code from the visual representation.

It is a domain-specific tool. Its domain is Ethereum smart contracts. It is not a
general-purpose visual programming environment — that is a separate, more ambitious
project (FlowCode) of which SolidiFlow serves as a proof of concept and precursor.

The one-directional principle is foundational: visual representation is the source
of truth. Solidity is the output. The user authors in the visual layer; the generated
code is not expected to be hand-edited. If a developer wishes to hand-edit the Solidity
output, they may do so, but at that point they have left the SolidiFlow environment.
This keeps the tool honest about what it is.

---

## Why SolidiFlow Exists

Two specific protocols — CGP (Consensus Governance Protocol) and P2PCP (P2P Compute
Protocol) — require non-trivial Ethereum smart contracts to function. Their authors
are not Solidity developers. The contracts need to be correct, auditable, and
maintainable by people who understand governance and compute protocol logic rather
than EVM bytecode.

SolidiFlow exists to bridge that gap. CGP and P2PCP are its primary design use cases.
The visual primitives it supports are chosen to be sufficient to express the contract
logic both protocols require. Generality beyond that is a bonus, not a requirement for
v1.0.

A secondary motivation: SolidiFlow demonstrates the core thesis of FlowCode — that
complex technical artefacts can be authored visually by domain experts without
programming knowledge — in a bounded, shippable scope. When FlowCode development
eventually begins, SolidiFlow is exhibit A.

A tertiary motivation: a well-executed SolidiFlow has independent commercial value.
The codebase is maintained separately from FlowCode, preserving the option to develop
a proprietary commercial variant without entangling it with FlowCode's fully open
codebase.

---

## Fast-Track Strategy: The Remix Fork

Remix IDE (remix.ethereum.org) is the industry-standard open-source browser-based
Solidity IDE. It is Apache 2.0 licensed, actively maintained, and provides:

- In-browser Solidity compiler
- Contract deployment to testnets and mainnet
- Debugger and transaction inspector
- Plugin architecture
- File management and workspace

SolidiFlow v0.1 is a fork of Remix with a visual flowchart layer added on top. Rather
than building a Solidity IDE from scratch, the Remix fork inherits all of the above
and the development effort focuses entirely on the visual authoring layer that is
SolidiFlow's actual contribution.

This fast-track fork is the immediate implementation target. A ground-up SolidiFlow
— built cleanly to specification without Remix's accumulated complexity — is a later
milestone, once the concept is validated in working software and the CGP/P2PCP
contracts have been designed using the tool.

### Proposed Stack for the Visual Layer

The flowchart component will be built using **ReactFlow** (MIT licensed), a
React-based library for node-and-edge diagram interfaces. Remix is already React-based,
making integration straightforward. ReactFlow provides:

- Draggable, connectable nodes
- Custom node types (each maps to a contract primitive)
- Edge routing and labelling
- Serialisation to JSON (which becomes the intermediate representation the
  Solidity generator consumes)

A spreadsheet component for managing contract state variables and mappings is planned
for a subsequent milestone. Candidate libraries include AG Grid Community (MIT) and
React Data Grid.

---

## Smart Contract Concepts for Reference

*This section is included for the benefit of collaborators who are new to Solidity
and smart contracts. Experienced Solidity developers may skip it.*

A smart contract is a program deployed permanently to a blockchain. It executes
automatically when its conditions are met, with no human or corporate intermediary.
Once deployed it is immutable (absent specific upgrade patterns) and its execution
is deterministic and publicly auditable.

The vocabulary of a smart contract is small:

- **State variables** — data stored permanently on-chain (balances, ownership,
  status flags, counters)
- **Functions** — callable actions that read or modify state
- **Modifiers** — reusable conditions that gate function execution
  (e.g. `onlyOwner`, `onlyAfterDeadline`)
- **Events** — records emitted to the chain log when significant things happen
- **Mappings** — key/value stores (e.g. address → token balance)
- **Structs** — grouped data types
- **Interfaces** — the public-facing function signatures of a contract,
  allowing other contracts to call it

ERC-20 is the Ethereum community's standard interface for fungible tokens.
Any contract that implements ERC-20 is automatically compatible with wallets,
exchanges, and other contracts that know how to talk to ERC-20 tokens.
CompuToken (P2PCP) and CitiCoin (CGP) are both ERC-20 tokens with additional
logic layered on top.

---

## Visual Primitives

The following node types form SolidiFlow's visual vocabulary. Together they are
sufficient to express the contract logic required by CGP and P2PCP:

**State Node** — represents a named state variable or mapping. Configurable type
(uint, address, bool, mapping, struct). Displayed as a data rectangle.

**Function Node** — represents a callable function. Configurable visibility
(public, external, internal, private), mutability (view, pure, payable, nonpayable),
and parameter list.

**Condition Node** — a decision diamond. Represents a require() statement or
an if/else branch. Has two output edges: pass and fail.

**Modifier Node** — a named reusable condition that can be attached to any
Function Node as a gate.

**Token Flow Arrow** — a directed edge representing a token transfer between
addresses. Annotated with token type and amount expression.

**Event Node** — represents an emitted event. Connected as a side-effect of
a Function Node execution.

**Time Trigger Node** — represents a block.timestamp condition. Used for
deadline enforcement and time-decay logic (e.g. CGP's active-pool decay).

**External Call Node** — represents a call to another deployed contract via
its interface. Used for oracle queries, AMM interactions, and cross-contract
composition.

**Phase/State Machine Node** — represents a named protocol phase (e.g. PoD,
PoM, PoR in CGP). Transitions between Phase Nodes are gated by Condition Nodes.
This is the primitive that makes CGP's three-phase consensus expressible visually.

---

## Primary Design Use Cases

### CompuToken (P2PCP)

CompuToken is an ERC-20 token with two custom functions:

`mint(address contributor, uint256 computeUnits)` — issues tokens when verified
compute contribution is recorded. Callable only by the oracle/verifier address.

`burn(address consumer, uint256 computeUnits)` — destroys tokens when compute
is consumed. Called automatically when a node draws on the network.

The open design question — how does the contract verify that real compute was
contributed before minting? — is a P2PCP protocol question, not a SolidiFlow
question. SolidiFlow needs to be able to express "only callable by the verified
oracle address" as a Modifier Node.

### CGP Agent Contract

The agent contract is a state machine with three phases (PoD, PoM, PoR) managing:

- CitiCoin bid pool intake and active-pool liquidity
- Mandate lifecycle tracking
- PoR settlement: residual pool valuation and PoliCoin minting
- Fiat on/off-ramp interface (AgentCoin bridge)

This is the contract that most fully exercises SolidiFlow's Phase Node and
Token Flow Arrow primitives. Its design will drive the v1.0 feature set more
than any other single use case.

---

## Relationship to Other Projects

**FlowCode** — SolidiFlow is a domain-specific precursor and proof of concept.
FlowCode is a general-purpose visual programming IDE on the back burner. The two
projects have separate codebases by design. SolidiFlow does not wait for FlowCode,
nor does FlowCode subsume SolidiFlow. When FlowCode development begins, SolidiFlow
exists as a working demonstration of the core thesis.

**CGP (Consensus Governance Protocol)** — Primary consumer. SolidiFlow is the
intended authoring tool for all CGP smart contracts. CGP's contract requirements
drive SolidiFlow's primitive vocabulary. See CGP Tokenomics v0.5 Position Paper
for the contract architecture SolidiFlow must be able to express.

**P2PCP (P2P Compute Protocol)** — Primary consumer. CompuToken and any
protocol-level P2PCP contracts will be authored in SolidiFlow. See P2PCP Project
Seed Document for the protocol architecture.

**Remix IDE** — Upstream fork source for SolidiFlow v0.1. Apache 2.0 licensed.
SolidiFlow inherits Remix's compiler, deployer, and debugger tooling.

---

## Immediate Next Actions

1. **Create the GitHub repository** — `SolidiFlow` under SkepticusMaximus.
   Initialise with this seed document as `README.md` or `SEED.md`,
   a `LICENSE` file (MIT or Apache 2.0 recommended for FOSS credibility
   and commercial variant optionality), and a `.gitignore` for Node/React projects.

2. **Fork Remix** — Fork the official Remix IDE repository into the SolidiFlow
   GitHub organisation or personal account. Establish the fork as the working
   codebase for SolidiFlow v0.1.

3. **Dependency audit** — Review Remix's current dependency tree and identify
   the insertion points for a ReactFlow-based visual panel. Remix's plugin
   architecture may provide a clean integration path without requiring deep
   modification of the core IDE.

4. **Define the intermediate representation** — Before any visual layer code
   is written, specify the JSON schema that the visual diagram serialises to
   and that the Solidity generator consumes. This schema is the contract between
   the visual layer and the code generation layer. Getting it right early avoids
   architectural refactoring later.

5. **Prototype a single primitive** — Implement one Function Node connected to
   one State Node, generating a minimal valid Solidity contract. This end-to-end
   proof establishes that the pipeline works before the full primitive vocabulary
   is built.

---

## Open Questions

1. **Plugin vs. fork modification** — Does Remix's plugin API provide sufficient
   integration depth for the visual layer, or does SolidiFlow require deeper
   fork-level modification? Answering this requires a spike into Remix's plugin
   architecture.

2. **Intermediate representation schema** — What is the canonical JSON structure
   that the visual diagram serialises to? This needs design before implementation.

3. **Solidity version targeting** — Which Solidity compiler version(s) should
   SolidiFlow target? Ethereum tooling moves fast; the choice affects which
   language features are available.

4. **Commercial variant boundary** — What is the precise feature boundary between
   the FOSS SolidiFlow and a hypothetical proprietary commercial variant?
   Establishing this early informs licence choice and API design.

5. **Audit and verification** — Smart contract bugs are expensive and irreversible.
   Does SolidiFlow v1.0 include any static analysis or formal verification layer,
   or is that deferred to a later milestone?

6. **Multi-contract composition** — CGP requires multiple interacting contracts
   (CitiCoin, Agent, PoliCoin, oracle interfaces). Does v1.0 support multi-contract
   projects, or does it target single-contract authoring initially?

---

## Milestone Target

**Functional CGP smart contract skeleton on Ethereum testnet before December 2026.**

This requires: SolidiFlow v0.1 fork working, CompuToken deployed on testnet,
CGP agent contract state machine expressible in the visual layer and deployable
from it.

This is the definition of "functional progress on CGP before Christmas."

---

*This is a founding seed document. It captures intent and architecture at the
point of project inception. It is not a specification. Specifications follow
once the repository is established and early implementation work clarifies
the design questions listed above.*

*Companion documents: P2PCP Project Seed, CGP Tokenomics v0.5 Position Paper.*
