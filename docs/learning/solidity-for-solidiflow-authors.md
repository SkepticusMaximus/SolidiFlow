# Solidity for SolidiFlow Authors

- **Audience:** anyone who understands the *logic* of what they want a smart contract to do but is not a Solidity developer — including SolidiFlow's primary author.
- **Goal:** enough grounding that you can read what SolidiFlow generates, sanity-check it, and make informed calls on the design questions that shape the visual layer.
- **Not a Solidity reference.** It's a tour of the language framed around SolidiFlow's IR primitives. The authoritative reference is <https://docs.soliditylang.org>.
- **Companion:** [`docs/ir/0001-ir-schema-v0.md`](../ir/0001-ir-schema-v0.md).

---

## A mental model for smart contracts

A smart contract is a small program permanently deployed to a blockchain. Once
deployed, its code cannot be changed (without specific upgrade patterns), and
anyone can call its public functions by sending a transaction. Each call costs
gas, paid by whoever sent the transaction. Calls run deterministically — given
the same state and inputs, every node in the network must agree on the result —
which is why contracts cannot read random numbers, query the time precisely, or
make HTTP calls. The contract sees only what's already on-chain plus the
arguments of the current call.

A contract has *state* that persists between calls (a small, expensive
database) and *functions* that read or modify it. There is no "main" loop —
the contract simply waits to be called. When called, a function runs to
completion, every state change is committed to the chain, and the contract
goes back to sleep until the next call. If the function reverts (for any
reason), all its state changes are rolled back as if the call never happened.

This last point matters. Reverts are how Solidity contracts enforce rules.
If a function does `require(condition, "message")` and the condition is false,
execution stops, all changes roll back, the user sees the error message, and
they pay gas for the work the EVM did before the revert. Most of the
"safety" patterns in Solidity are some flavour of `require(...)`.

A few global variables are always available inside a function:

- `msg.sender` — the address that called this function. *Never trust the
  user input; trust `msg.sender`.* This is how access control works.
- `msg.value` — wei of native ETH sent along with the call.
- `block.timestamp` — the timestamp of the block the call is mined in.
  Coarse and miner-influenceable; never use for fine timing or as a random
  seed, but fine for "after this deadline" logic.
- `address(this)` — the contract's own address.

That's the minimum mental model. Everything below is how SolidiFlow's visual
primitives map onto Solidity language constructs.

---

## State Node → state variable

A State Node becomes a state variable: data stored permanently on-chain. The
shape:

```solidity
address public immutable oracle;
uint256 public totalSupply;
mapping(address => uint256) private balances;
```

Three things shape a state variable: its **type** (`address`, `uint256`,
`mapping`, struct, etc.), its **visibility** (`public`, `internal`,
`private`), and one of two optional modifiers — `constant` (set at compile
time, never changes) or `immutable` (set once in the constructor, never
changes after).

`public` on a state variable auto-generates a getter function with the same
name — so `address public oracle;` gives you a free `oracle()` external
function that returns the address. This is why public state variables
appear "callable" from outside the contract.

In the IR, `solidityType` is a free-form string that the generator emits
verbatim. This is intentionally pragmatic — SolidiFlow doesn't try to
re-implement Solidity's type system; it just lets the user say
"`mapping(address => uint256)`" and trusts them. Validation is the
compiler's job.

---

## Function Node → function

A Function Node becomes a Solidity function:

```solidity
function mint(address contributor, uint256 amount)
    external
    onlyOracle
{
    _mint(contributor, amount);
    emit Mint(contributor, amount);
}
```

The function declaration has, in this order: name, parameters, **visibility**
(`public` / `external` / `internal` / `private`), **mutability**
(`view` / `pure` / `payable` / unspecified-is-nonpayable), modifiers,
returns, then the body in `{ ... }`.

Visibility means:

- `external` — only callable from outside the contract. Cheaper for large
  argument lists. Most user-facing functions are `external`.
- `public` — callable from outside *and* from other functions within the
  contract. Slightly more flexible, slightly less efficient.
- `internal` — only callable from this contract or contracts that inherit
  it. Used for shared helpers and for the underscore-prefixed primitives
  like `_mint`, `_burn`, `_transfer` that ERC-20 exposes for subclasses.
- `private` — only callable from this exact contract. Rare in practice;
  `internal` is more common.

Mutability means:

- `view` — promises not to modify state (only reads). Free to call from
  off-chain readers.
- `pure` — promises not to even read state. Pure computation.
- `payable` — willing to receive ETH. Without this, a function that's sent
  ETH reverts.
- *(unspecified)* — `nonpayable` by default; can read and write state but
  rejects ETH.

The body of the function is what control-flow edges in the IR generate. A
Function Node connected to a Condition Node, then an `_mint` external-call
or sideEffect to an Event, is the visual analogue of the function body
above.

---

## Modifier Node → modifier

A modifier is a reusable pre-condition that wraps a function:

```solidity
modifier onlyOracle() {
    require(msg.sender == oracle, "Not oracle");
    _;
}
```

The `_;` is the magic. It's where the wrapped function's body gets
inserted. So `function mint(...) external onlyOracle { ... }` becomes
roughly:

```solidity
function mint(...) external {
    require(msg.sender == oracle, "Not oracle");
    // <body of mint>
}
```

If the require fails, the function never runs. Order matters when multiple
modifiers stack — `function foo() onlyA onlyB onlyC { ... }` means A's
require runs first, then B's, then C's, then the body, then C's
post-`_;` cleanup, etc. The IR preserves this ordering via the `modifiers[]`
array on the Function Node.

This is why we don't model "modifier attached to function" as a graph edge —
it's a property of the function declaration, not a flow.

---

## Condition Node → require / if-else

Condition Nodes have two flavours, captured by their `style`:

`style: "require"` becomes:

```solidity
require(msg.sender == oracle, "Not oracle");
```

If the expression is true, execution continues. If false, the entire
transaction reverts and the user pays gas up to that point.

`style: "ifElse"` becomes a normal if-else block:

```solidity
if (block.timestamp >= deadline) {
    // pass branch
} else {
    // fail branch
}
```

The IR's `pass` / `fail` source handles map directly to which branch the
downstream nodes belong to. For `require` style, only the `pass` branch
continues — the `fail` handle is a placeholder and no edge should leave it.

---

## Event Node → event + emit

Events are how a contract writes to the chain log — a special append-only
list visible to off-chain readers (block explorers, frontends, indexers)
but not readable from inside Solidity. Events are cheap and non-blocking
and exist mostly so user interfaces can find out that something happened
without having to poll the contract's state.

The declaration:

```solidity
event Mint(address indexed to, uint256 amount);
```

The `indexed` keyword makes that parameter searchable in the log
(up to three indexed params per event). Indexed addresses are the common
case so you can filter "show me every Mint to address X."

The emit:

```solidity
emit Mint(contributor, amount);
```

In the IR, the `event` *declaration* lives as a node attached to the
contract; the `emit` *call* is a `sideEffect` edge from a Function Node
to the Event Node. So the same Event Node can be emitted from many
different functions, each one is just another sideEffect edge.

---

## Time Trigger Node → block.timestamp comparison

A Time Trigger Node is a specialised Condition Node — semantically the same
shape, but flagged differently in the IR so the editor can render a clock
icon and the generator can recognise time-decay patterns. It compiles to:

```solidity
require(block.timestamp >= deadline, "Too early");
```

or in the if-else form:

```solidity
if (block.timestamp >= deadline) { /* ... */ }
```

Worth knowing: `block.timestamp` is the timestamp of the block the
transaction is mined in, in Unix seconds. Miners can fudge it by a few
seconds, so don't use it for fine-grained timing or randomness. For
"deadline passed" logic at minute-or-coarser granularity, it's fine — and
that's most of what CGP needs.

---

## External Call Node → interface call

When your contract wants to call a *different* deployed contract, you do it
through that contract's interface:

```solidity
interface IERC20 {
    function transferFrom(address from, address to, uint256 amount)
        external
        returns (bool);
}

contract Example {
    IERC20 public usdc;

    function deposit(uint256 amount) external {
        bool ok = usdc.transferFrom(msg.sender, address(this), amount);
        require(ok, "USDC transfer failed");
    }
}
```

`IERC20(usdcAddress).transferFrom(...)` is shorthand for "call the
`transferFrom` function at `usdcAddress`, treating that contract as if it
implements `IERC20`'s function signatures." If the target address doesn't
actually implement them, you get a runtime revert.

External calls are how oracles, AMMs, and cross-contract composition all
work. They're more expensive than internal calls and they're a security
risk surface — a malicious external contract could try to re-enter your
function before it finishes (the reentrancy attack). For SolidiFlow's
target use cases, follow the standard pattern: do all your state changes
*before* you make external calls, and `require` the boolean return for
ERC-20 transfers.

---

## Phase Node → enum + state variable + guards

Phase Nodes compile to three things working together: an `enum` declaring
the phase names, a state variable holding the current phase, and a `require`
in each phase-scoped function checking that the current phase is correct.

```solidity
enum Phase { PoD, PoM, PoR }
Phase public currentPhase;

function submitDeliverable() external {
    require(currentPhase == Phase.PoD, "Not in PoD");
    // ...
}

function advanceToPoM() external {
    require(currentPhase == Phase.PoD, "Not in PoD");
    require(block.timestamp >= deliveryDeadline, "PoD not over");
    currentPhase = Phase.PoM;
}
```

`enum` is just a named integer — `Phase.PoD` is `0`, `Phase.PoM` is `1`,
`Phase.PoR` is `2`. The `enumIndex` on the IR's Phase Node controls this
ordering. `Phase public currentPhase;` auto-generates a getter so frontends
can read which phase the contract is in.

Each phase transition (an edge between two Phase Nodes in the IR) becomes
an advance function: enforce we're in the source phase, evaluate the
guard condition (often a Time Trigger), then write the new phase. SolidiFlow
generates these advance functions automatically from the `phaseTransition`
edges; you don't have to author them by hand.

---

## Token Flow Arrow → transfer / transferFrom / internal _transfer

A Token Flow Arrow is the only IR primitive that has multiple Solidity
translations depending on context. Three cases:

**Native ETH transfer** — when `tokenStandard: "native"`:

```solidity
(bool ok,) = recipient.call{value: amount}("");
require(ok, "ETH transfer failed");
```

**External ERC-20 transfer** — when the token is a *different* contract
than the one you're authoring:

```solidity
bool ok = IERC20(tokenAddress).transferFrom(from, to, amount);
require(ok, "Transfer failed");
```

**Self-token transfer** — when the contract you're authoring *is* the
ERC-20 (CompuToken's case), you can manipulate balances internally without
making an external call to yourself:

```solidity
_transfer(from, to, amount);   // moves tokens
_mint(to, amount);              // creates new tokens
_burn(from, amount);            // destroys tokens
```

These `_`-prefixed functions are inherited from OpenZeppelin's `ERC20.sol`
when you do `contract CompuToken is ERC20`. They're cheaper, they don't
trigger reentrancy concerns, and they're idiomatic. **This is the case
that drives one of the IR open questions** — does SolidiFlow auto-detect
"this Token Flow operates on the contract's own ERC-20 balance" and emit
the internal version, or does it require the user to declare which?

---

## Worked example: CompuToken end-to-end

Putting all the pieces together, here's what SolidiFlow's generator should
emit for the CompuToken IR in
[`docs/ir/0001-ir-schema-v0.md`](../ir/0001-ir-schema-v0.md):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/// @title CompuToken — the P2PCP token
/// @notice Minted when verified compute is contributed; burned when consumed.
contract CompuToken is ERC20 {
    address public immutable oracle;

    event Mint(address indexed to, uint256 amount);
    event Burn(address indexed from, uint256 amount);

    modifier onlyOracle() {
        require(msg.sender == oracle, "Not oracle");
        _;
    }

    constructor(address _oracle) ERC20("CompuToken", "CMPT") {
        oracle = _oracle;
    }

    function mint(address contributor, uint256 computeUnits)
        external
        onlyOracle
    {
        _mint(contributor, computeUnits);
        emit Mint(contributor, computeUnits);
    }

    function burn(address consumer, uint256 computeUnits)
        external
        onlyOracle
    {
        _burn(consumer, computeUnits);
        emit Burn(consumer, computeUnits);
    }
}
```

Tour through what came from where:

- `pragma solidity ^0.8.20;` from `project.solidityVersion`.
- `import "@openzeppelin/.../ERC20.sol";` from `contracts[0].imports`.
- `contract CompuToken is ERC20` from `contracts[0].name` and
  `contracts[0].inherits`.
- `address public immutable oracle;` from the `state-oracle` State Node.
- `event Mint(...)` from the `evt-Mint` Event Node's declaration shape.
- `modifier onlyOracle() { require(...); _; }` from the `mod-onlyOracle`
  Modifier Node and its referenced Condition Node.
- `constructor(address _oracle) ERC20("CompuToken", "CMPT") { oracle = _oracle; }`
  is *not* in the v0 IR yet — this is one of the open questions. v0 either
  treats it as an implicit Function Node named `constructor` or gets its
  own node type.
- `function mint(...)` from the `fn-mint` Function Node, with `_mint`
  emitted because the visual layer represents this as a Token Flow on the
  contract's own ERC-20 balance. This is the auto-detect case that needs
  resolving.
- `emit Mint(contributor, computeUnits);` from the `sideEffect` edge from
  `fn-mint` to `evt-Mint`.

Saving and re-loading the IR should produce byte-identical Solidity. That's
the round-trip property the IR is designed for.

---

## Worked example: CGP three-phase advance functions

The CGP partial example becomes:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract CGPAgent {
    enum Phase { PoD, PoM, PoR }
    Phase public currentPhase;

    uint256 public deliveryDeadline;
    uint256 public measureDeadline;

    function advanceToPoM() external {
        require(currentPhase == Phase.PoD, "Not in PoD");
        require(block.timestamp >= deliveryDeadline, "PoD not over");
        currentPhase = Phase.PoM;
    }

    function advanceToPoR() external {
        require(currentPhase == Phase.PoM, "Not in PoM");
        require(block.timestamp >= measureDeadline, "PoM not over");
        currentPhase = Phase.PoR;
    }

    // submitDeliverable, postMeasurement, resolve to be added —
    // each begins with require(currentPhase == Phase.<correct>, "...")
}
```

The advance functions are generated entirely from the `phaseTransition`
edges. The phase guards on `submitDeliverable`, `postMeasurement`, and
`resolve` are the ones the generator would auto-inject under Open Question
#2's "auto-inject yes" interpretation. If we go "explicit only", the user
has to wire each Function Node to a Condition Node themselves — more work,
but more honest.

---

## Now you can answer the three deferred questions

With the above grounding, the three IR open questions are no longer
abstract:

**1. Self-token Token Flow auto-detection.** When `tokenFlow` source/target
references the contract's own ERC-20 balance, do we emit `_mint` / `_burn` /
`_transfer` automatically, or require the user to declare an "internal"
flag? Auto-detect is fewer clicks but can be surprising the first time it
happens; explicit is more honest but adds friction. The sympathetic pattern
is: auto-detect by default, surface the choice in the property panel so
users can override.

**2. Phase-scoped Function auto-guard.** When a Function Node is
`attachedFunctionIds` of a Phase Node, do we auto-emit
`require(currentPhase == Phase.X)` at the top of the function body?
Auto-inject matches user intuition (drawing a function inside a phase
*means* it's only callable in that phase). Explicit-only forces the user
to wire a Condition Node, which is verbose but visible. Auto-inject with
an opt-out flag in the Function Node's data is probably the right balance.

**3. Constructor as a Function Node or its own type.** Constructors are
genuinely different from regular functions — one-shot, no `external`/
`public` distinction (they're called by the deployment process), can set
`immutable` state, and can take parameters that influence the deployed
contract's permanent configuration. Modelling them as a regular Function
Node with name `"constructor"` works but conflates them with normal
methods. A dedicated Constructor Node type is more honest, gives the
visual layer room to surface immutables and inherited constructor calls
clearly, but adds a primitive. The CGP and P2PCP contracts both need
constructors (oracle address, ERC20 name/symbol, initial deadlines), so
this *will* show up in v0.1.

---

## What to read when you want to go deeper

In rough order of usefulness for SolidiFlow's design space:

The Solidity language tour at <https://docs.soliditylang.org/en/latest/solidity-by-example.html>
is the most economical way to internalise the language's idioms. Plain
prose, complete examples, no fluff.

OpenZeppelin's `ERC20.sol` source at
<https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol>
is what `_mint` / `_burn` / `_transfer` actually do. Reading this once
will demystify the "self-token internal call" pattern.

The Solidity security considerations page at
<https://docs.soliditylang.org/en/latest/security-considerations.html>
covers reentrancy, integer overflow, the dangers of `block.timestamp`,
and other footguns. SolidiFlow's job is to make these footguns harder
to step on; knowing what they are makes the tool's design choices land.

Remix IDE itself at <https://remix.ethereum.org> is the best playground
for poking at small contracts. Paste the CompuToken example above into
the file editor, hit Compile, and you'll have working bytecode in seconds.
Deploy to Remix's in-memory VM, call `mint` from the deployer address
(which is the oracle in the constructor), and watch the balance update
in the Deploy & Run panel. This is the loop SolidiFlow ultimately
inherits.

---

## What this primer doesn't cover

Things that exist in real Solidity contracts but that SolidiFlow's v0
deliberately doesn't try to express:

- Inline assembly (`assembly { ... }`) — bypasses Solidity's safety net,
  not in scope.
- Custom errors (`error InsufficientBalance(...)` instead of
  `require(..., "string")`) — gas-efficient, will appear in v0.1 once the
  Condition Node grows a richer error mode.
- Libraries (`library SafeMath`) — the patterns SolidiFlow needs are mostly
  covered by 0.8's built-in overflow checks plus OpenZeppelin's contracts.
- Upgrade patterns (proxies, UUPS, transparent) — explicitly out of scope
  for v0 per the seed doc; immutable is fine for the protocols SolidiFlow
  serves.
- Yul, low-level `call`/`delegatecall` semantics, fallback/receive
  functions — not needed for the visual layer's target contracts.

When SolidiFlow eventually outgrows what's covered here, this doc grows
with it.
