# SolidiFlow IR v0 — JSON Schema Design

- **Author:** SolidiFlow design notes
- **Status:** v0 draft, not yet frozen
- **Scope:** intermediate representation consumed by the Solidity generator
- **Companion:** [`docs/adr/0001-remix-integration-approach.md`](../adr/0001-remix-integration-approach.md) (independent of the integration choice)

---

## 1. Design rationale

The IR follows ReactFlow's `nodes[]` + `edges[]` convention because (a) the
visual layer is the source of truth and we want lossless round-trips, and
(b) it keeps layout concerns (`position`, `width`, `handles`) cleanly
separable from semantic concerns. Each node carries a `data` object that
holds Solidity-specific configuration (visibility, mutability, type, etc.);
the generator reads only `data`, while the editor reads the full record.
Top-level the document splits into `diagram` (per-canvas layout + node/edge
arrays) and `contracts` (the logical contract artefacts that the generator
emits). A project is `contracts: [...]` even though v0 typically has one —
this lets us introduce libraries, interfaces, and multi-file projects later
without a breaking migration.

Modifiers are *defined* as their own nodes (so the visual canvas can show
them) but *attached* to Function Nodes by listing the modifier node's `id`
in the function's `data.modifiers[]`. We deliberately did not model the
attachment as a graph edge: attachment is a property of the function, not a
flow, and modelling it as an edge would force the generator to traverse
edges to figure out a function signature. Conversely, control flow inside a
function (`Condition`, `Event`, `External Call`, `Token Flow`) *is*
edge-based, because that's a flow. Phase transitions are also edges, between
Phase Nodes, gated by an optional Condition Node referenced in edge `data`.

Two ReactFlow conventions we honour: (1) Condition Nodes expose two named
source handles, `"pass"` and `"fail"`, and edges out of them must set
`sourceHandle` to one of those values; (2) every node and edge has a stable
string `id` (UUID or slug) that the generator uses for cross-references.

---

## 2. Top-level structure

```json
{
  "version": "0.1.0",
  "project": {
    "name": "string",
    "createdAt": "ISO-8601",
    "updatedAt": "ISO-8601",
    "solidityVersion": "^0.8.20"
  },
  "diagram": {
    "viewport": { "x": 0, "y": 0, "zoom": 1 },
    "nodes": [ /* Node */ ],
    "edges": [ /* Edge */ ]
  },
  "contracts": [
    {
      "id": "string",
      "name": "string",
      "kind": "contract | interface | library | abstract",
      "inherits": ["ERC20"],
      "imports": ["@openzeppelin/contracts/token/ERC20/ERC20.sol"],
      "nodeIds": ["state-1", "fn-mint", "..."]
    }
  ]
}
```

Each node belongs to exactly one contract via `contracts[].nodeIds`.
JSON-Schema fragment for the root:

```json
{
  "$id": "https://solidiflow.dev/schema/ir-0.1.json",
  "type": "object",
  "required": ["version", "diagram", "contracts"],
  "properties": {
    "version": { "type": "string", "pattern": "^0\\.\\d+\\.\\d+$" },
    "project": { "type": "object" },
    "diagram": {
      "type": "object",
      "required": ["nodes", "edges"],
      "properties": {
        "nodes": { "type": "array", "items": { "$ref": "#/$defs/Node" } },
        "edges": { "type": "array", "items": { "$ref": "#/$defs/Edge" } }
      }
    },
    "contracts": { "type": "array", "items": { "$ref": "#/$defs/Contract" } }
  }
}
```

---

## 3. Node type definitions

All nodes share a base shape:

```json
{
  "id": "string",
  "type": "state | function | condition | modifier | event | timeTrigger | externalCall | phase",
  "position": { "x": 120, "y": 240 },
  "width": 180, "height": 80,
  "data": { /* type-specific */ }
}
```

### State Node

```json
{ "type": "state",
  "data": {
    "name": "balances",
    "solidityType": "mapping(address => uint256)",
    "visibility": "public | internal | private",
    "constant": false,
    "immutable": false,
    "initialValue": null
  } }
```

`solidityType` is a string that the generator emits verbatim (mapping,
struct ref, primitive). `visibility` defaults to `internal`.

### Function Node

```json
{ "type": "function",
  "data": {
    "name": "mint",
    "visibility": "public | external | internal | private",
    "mutability": "view | pure | payable | nonpayable",
    "params": [{ "name": "to", "type": "address" }],
    "returns": [{ "name": "", "type": "bool" }],
    "modifiers": ["mod-onlyOracle"],
    "bodyEntryEdgeId": "edge-fn-mint-entry"
  } }
```

`modifiers[]` references modifier node IDs. `bodyEntryEdgeId` (optional)
names the edge that begins the function body's control flow.

### Condition Node

```json
{ "type": "condition",
  "data": {
    "expression": "msg.sender == oracle",
    "style": "require | ifElse",
    "revertMessage": "Not authorised"
  } }
```

Source handles are fixed: `"pass"` and `"fail"`. If `style == "require"`
only `"pass"` continues, `"fail"` triggers the revert and is not expected to
be wired.

### Modifier Node

```json
{ "type": "modifier",
  "data": {
    "name": "onlyOracle",
    "params": [],
    "conditionNodeIds": ["cond-isOracle"]
  } }
```

The modifier's body is the chain of Condition Nodes it references; the
generator emits `_;` after them.

### Event Node

```json
{ "type": "event",
  "data": {
    "name": "Mint",
    "params": [
      { "name": "to", "type": "address", "indexed": true },
      { "name": "amount", "type": "uint256", "indexed": false }
    ],
    "emitArgs": ["to", "computeUnits"]
  } }
```

`emitArgs` are the expressions used at the emit site; if the event is
*declared* but not emitted in this position, omit `emitArgs`.

### Time Trigger Node

```json
{ "type": "timeTrigger",
  "data": {
    "kind": "deadline | interval | timestampGte | timestampLte",
    "expression": "block.timestamp >= deadline",
    "boundVariable": "deadline"
  } }
```

Semantically a specialised Condition Node; kept distinct so the editor can
render a clock icon and so the generator can recognise time-decay patterns.

### External Call Node

```json
{ "type": "externalCall",
  "data": {
    "interfaceName": "IERC20",
    "targetExpression": "token",
    "method": "transferFrom",
    "args": ["msg.sender", "address(this)", "amount"],
    "returnsTo": "ok",
    "checkReturn": true
  } }
```

### Phase Node

```json
{ "type": "phase",
  "data": {
    "name": "PoD",
    "enumIndex": 0,
    "attachedFunctionIds": ["fn-submitDeliverable"],
    "onEnter": [],
    "onExit": []
  } }
```

The generator emits a `Phase` enum from all phase nodes (ordered by
`enumIndex`) and a `currentPhase` state variable.

---

## 4. Edge / connection definitions

Base edge:

```json
{ "id": "string",
  "type": "control | tokenFlow | phaseTransition | sideEffect",
  "source": "node-id", "target": "node-id",
  "sourceHandle": "pass | fail | out | null",
  "targetHandle": null,
  "data": { /* type-specific */ } }
```

- **`control`** — sequence inside a function body. `data` may carry
  `{ "label": "..." }`.
- **`sideEffect`** — function → event or function → external call. Generator
  emits the call/emit at the corresponding point in the function body,
  ordered by `data.order`.
- **`tokenFlow`** — directed token transfer:

  ```json
  { "type": "tokenFlow",
    "data": {
      "tokenStandard": "native | ERC20 | ERC721 | ERC1155",
      "tokenAddress": "address(this)",
      "amountExpression": "amount",
      "from": "msg.sender",
      "to": "address(this)"
    } }
  ```

  `source`/`target` may be Function or Phase nodes; the generator translates
  to `transfer`/`transferFrom`/`call{value:}`.

- **`phaseTransition`** — between two Phase Nodes:

  ```json
  { "type": "phaseTransition",
    "data": {
      "guardConditionId": "cond-deadline-passed",
      "triggerFunctionId": "fn-advancePhase"
    } }
  ```

**Condition handle convention:** edges leaving a `condition` node MUST set
`sourceHandle` to `"pass"` or `"fail"`. Validators reject anything else.

---

## 5. Worked example A — CompuToken

```json
{
  "version": "0.1.0",
  "project": { "name": "CompuToken", "solidityVersion": "^0.8.20" },
  "diagram": {
    "viewport": { "x": 0, "y": 0, "zoom": 1 },
    "nodes": [
      { "id": "state-oracle", "type": "state", "position": {"x":40,"y":40},
        "data": { "name": "oracle", "solidityType": "address",
                  "visibility": "public", "immutable": true } },
      { "id": "mod-onlyOracle", "type": "modifier", "position": {"x":40,"y":140},
        "data": { "name": "onlyOracle", "params": [],
                  "conditionNodeIds": ["cond-isOracle"] } },
      { "id": "cond-isOracle", "type": "condition", "position": {"x":40,"y":220},
        "data": { "expression": "msg.sender == oracle",
                  "style": "require", "revertMessage": "Not oracle" } },
      { "id": "fn-mint", "type": "function", "position": {"x":300,"y":40},
        "data": { "name": "mint", "visibility": "external",
                  "mutability": "nonpayable",
                  "params": [
                    { "name": "contributor", "type": "address" },
                    { "name": "computeUnits", "type": "uint256" }
                  ],
                  "returns": [],
                  "modifiers": ["mod-onlyOracle"] } },
      { "id": "evt-Mint", "type": "event", "position": {"x":540,"y":40},
        "data": { "name": "Mint",
                  "params": [
                    { "name": "to", "type": "address", "indexed": true },
                    { "name": "amount", "type": "uint256", "indexed": false }
                  ],
                  "emitArgs": ["contributor", "computeUnits"] } },
      { "id": "fn-burn", "type": "function", "position": {"x":300,"y":160},
        "data": { "name": "burn", "visibility": "external",
                  "mutability": "nonpayable",
                  "params": [
                    { "name": "consumer", "type": "address" },
                    { "name": "computeUnits", "type": "uint256" }
                  ],
                  "returns": [],
                  "modifiers": ["mod-onlyOracle"] } },
      { "id": "evt-Burn", "type": "event", "position": {"x":540,"y":160},
        "data": { "name": "Burn",
                  "params": [
                    { "name": "from", "type": "address", "indexed": true },
                    { "name": "amount", "type": "uint256", "indexed": false }
                  ],
                  "emitArgs": ["consumer", "computeUnits"] } }
    ],
    "edges": [
      { "id": "e1", "type": "sideEffect", "source": "fn-mint", "target": "evt-Mint",
        "sourceHandle": "out", "data": { "order": 1 } },
      { "id": "e2", "type": "sideEffect", "source": "fn-burn", "target": "evt-Burn",
        "sourceHandle": "out", "data": { "order": 1 } }
    ]
  },
  "contracts": [
    { "id": "c-CompuToken", "name": "CompuToken", "kind": "contract",
      "inherits": ["ERC20"],
      "imports": ["@openzeppelin/contracts/token/ERC20/ERC20.sol"],
      "nodeIds": ["state-oracle", "mod-onlyOracle", "cond-isOracle",
                  "fn-mint", "evt-Mint", "fn-burn", "evt-Burn"] }
  ]
}
```

The generator emits: `_mint(contributor, computeUnits); emit Mint(...)` for
`mint` (and the symmetric `_burn` for `burn`), gated by `onlyOracle`.

---

## 6. Worked example B — CGP three-phase state machine (partial)

```json
{
  "version": "0.1.0",
  "project": { "name": "CGPAgent", "solidityVersion": "^0.8.20" },
  "diagram": {
    "viewport": { "x": 0, "y": 0, "zoom": 1 },
    "nodes": [
      { "id": "ph-PoD", "type": "phase", "position": {"x":40,"y":40},
        "data": { "name": "PoD", "enumIndex": 0,
                  "attachedFunctionIds": ["fn-submitDeliverable"] } },
      { "id": "ph-PoM", "type": "phase", "position": {"x":260,"y":40},
        "data": { "name": "PoM", "enumIndex": 1,
                  "attachedFunctionIds": ["fn-postMeasurement"] } },
      { "id": "ph-PoR", "type": "phase", "position": {"x":480,"y":40},
        "data": { "name": "PoR", "enumIndex": 2,
                  "attachedFunctionIds": ["fn-resolve"] } },
      { "id": "state-deadlineDelivery", "type": "state", "position":{"x":40,"y":160},
        "data": { "name": "deliveryDeadline", "solidityType": "uint256",
                  "visibility": "public" } },
      { "id": "tt-deliveryOver", "type": "timeTrigger", "position":{"x":150,"y":160},
        "data": { "kind": "timestampGte",
                  "expression": "block.timestamp >= deliveryDeadline",
                  "boundVariable": "deliveryDeadline" } },
      { "id": "cond-deliveryOver", "type": "condition", "position":{"x":150,"y":240},
        "data": { "expression": "block.timestamp >= deliveryDeadline",
                  "style": "require", "revertMessage": "PoD not over" } },
      { "id": "tt-measureOver", "type": "timeTrigger", "position":{"x":370,"y":160},
        "data": { "kind": "timestampGte",
                  "expression": "block.timestamp >= measureDeadline",
                  "boundVariable": "measureDeadline" } },
      { "id": "cond-measureOver", "type": "condition", "position":{"x":370,"y":240},
        "data": { "expression": "block.timestamp >= measureDeadline",
                  "style": "require", "revertMessage": "PoM not over" } }
    ],
    "edges": [
      { "id": "t1", "type": "phaseTransition",
        "source": "ph-PoD", "target": "ph-PoM",
        "sourceHandle": "out",
        "data": { "guardConditionId": "cond-deliveryOver",
                  "triggerFunctionId": "fn-advanceToPoM" } },
      { "id": "t2", "type": "phaseTransition",
        "source": "ph-PoM", "target": "ph-PoR",
        "sourceHandle": "out",
        "data": { "guardConditionId": "cond-measureOver",
                  "triggerFunctionId": "fn-advanceToPoR" } }
    ]
  },
  "contracts": [
    { "id": "c-CGPAgent", "name": "CGPAgent", "kind": "contract",
      "inherits": [], "imports": [],
      "nodeIds": ["ph-PoD","ph-PoM","ph-PoR",
                  "state-deadlineDelivery",
                  "tt-deliveryOver","cond-deliveryOver",
                  "tt-measureOver","cond-measureOver"] } ]
}
```

The generator emits `enum Phase { PoD, PoM, PoR }`,
`Phase public currentPhase;`, plus advance functions whose bodies inline the
guard `Condition` and assign `currentPhase = Phase.PoM` (etc.).

---

## 7. Open questions

- **Function bodies — graph or AST?** v0 leans graph (control edges +
  side-effect edges). Complex arithmetic inside a function still has to be a
  string expression on a Condition or Event node. v0.2 should consider a
  lightweight expression AST to avoid string-juggling.
- **Struct definitions.** Currently `solidityType` is a free-form string.
  We should add a top-level `types[]` for struct/enum declarations so the
  editor can offer autocomplete and the generator can validate references.
- **Inheritance and overrides.** `contracts[].inherits` is a flat list; we
  haven't modelled `override` markers on Function Nodes. Likely a
  `data.override: true` flag plus a `virtual` flag in v0.1.
- **Constructor.** Implicit for now (a Function Node named `constructor`?).
  Probably needs its own node type in v0.2 for clarity.
- **Token Flow vs ERC-20 internals.** A `tokenFlow` arrow on an ERC-20
  contract that operates on its *own* balances should compile to internal
  `_transfer`, not to an external call. Disambiguating "self-token" vs
  "external token" needs Stevo's call.
- **Modifier composition.** Multiple modifiers on one function — order
  matters in Solidity. We preserve order via the `modifiers[]` array, but
  should we surface that in the visual layer (drag-to-reorder) or treat it
  as a property panel concern?
- **Phase-scoped functions.** `attachedFunctionIds` says "this function is
  callable in this phase" — should the generator auto-inject a
  `require(currentPhase == Phase.X)` guard, or must the user wire it
  explicitly via a Condition Node? Lean toward auto-inject with an opt-out
  flag, but flagging for review.
- **Layout metadata granularity.** We store `position`, `width`, `height`
  but not edge waypoints, node colour, or collapsed/expanded state.
  Round-trip fidelity is therefore "good enough" but not perfect; v0.2
  should add an `editorMeta` blob per node/edge.
- **Validation seam.** JSON Schema covers shape; semantic validation
  (e.g. "every Condition reachable from a Function must terminate in a
  return or revert") is generator-side. Whether to publish this as a
  separate linter is an open question.
