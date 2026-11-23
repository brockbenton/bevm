# Milestone 4: Gas

**Goal:** charge every operation exactly.
**Deliverable:** execution gas for your ERC-20 calls matches a Foundry trace to the unit.
**Size:** 450–650 source lines, 350–500 test lines. Dense; go slowly.

Values below are Cancun.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Gas counter | `gas.rs` | 40–60 |
| 2. Static costs | `gas.rs` / `opcode.rs` | 70–100 |
| 3. Memory expansion | `memory.rs` / `gas.rs` | 45–65 |
| 4. Dynamic costs | `interpreter/` | 75–105 |
| 5. Warm/cold tracking | `state.rs` | 70–100 |
| 6. SSTORE and refunds | `interpreter/storage.rs` | 110–160 |
| 7. GAS opcode and wiring | `interpreter/` | 40–60 |

## Step 1: Counter

Remaining `u64`, refund `i64` (nested frames can lower it). `charge` returns OutOfGas on shortfall. An exceptional halt consumes everything left; STOP, RETURN and REVERT don't.

**Charge before doing the work,** including before allocating memory.

## Step 2: Static costs

| Gas | Ops |
|---:|---|
| 0 | STOP, RETURN, REVERT (plus memory) |
| 1 | JUMPDEST |
| 2 | PUSH0, POP, PC, MSIZE, GAS, ADDRESS, ORIGIN, CALLER, CALLVALUE, CALLDATASIZE, CODESIZE, GASPRICE, RETURNDATASIZE, COINBASE, TIMESTAMP, NUMBER, PREVRANDAO, GASLIMIT, CHAINID, BASEFEE, BLOBBASEFEE |
| 3 | ADD, SUB, NOT, comparisons, bitwise, shifts, BYTE, PUSH, DUP, SWAP, MLOAD, MSTORE, MSTORE8, CALLDATALOAD, BLOBHASH, copy base |
| 5 | MUL, DIV, SDIV, MOD, SMOD, SIGNEXTEND, SELFBALANCE |
| 8 | ADDMOD, MULMOD, JUMP |
| 10 | JUMPI, EXP base |
| 20 | BLOCKHASH |
| 30 | KECCAK256 base |
| 100 | TLOAD, TSTORE |
| 375 | LOG base |

## Step 3: Memory

With `w` words, total cost is `3w + floor(w² / 512)`. Charge new total minus old. 1 word costs 3; 32 words cost 98. A size that doesn't fit is OutOfGas. You can now remove M2's temporary cap.

## Step 4: Dynamic costs

| Op | Extra |
|---|---|
| EXP | 50 per byte of the exponent |
| KECCAK256 | 6 per word + memory |
| CALLDATACOPY, CODECOPY, MCOPY, RETURNDATACOPY | 3 per word + memory |
| EXTCODECOPY | account access + 3 per word + memory |
| LOGn | 375 per topic + 8 per byte + memory |

Words round up: 33 bytes is 2 words.

## Step 5: Warm and cold (EIP-2929)

| Access | Cold | Warm |
|---|---:|---:|
| Account: BALANCE, EXTCODE* | 2600 | 100 |
| SLOAD | 2100 | 100 |

The set lasts for a transaction. Start warm: sender, recipient, precompiles, coinbase (EIP-3651). Those and SELFBALANCE matter when matching traces.

## Step 6: SSTORE

Track three values: **original** (at transaction start), **current**, **new**.

1. Fail with OutOfGas if remaining gas ≤ 2300.
2. Add 2100 if the slot is cold.
3. Base cost:

| Case | Gas |
|---|---:|
| new == current | 100 |
| current == original, original == 0 | 20000 |
| current == original, original != 0 | 2900 |
| current != original | 100 |

4. Refunds (only when new != current):

| Case | Refund |
|---|---:|
| current == original, original != 0, new == 0 | +4800 |
| current != original, original != 0, current == 0 | −4800 |
| current != original, original != 0, new == 0 | +4800 |
| current != original, new == original, original == 0 | +19900 |
| current != original, new == original, original != 0 | +2800 |

The rows for current != original are cumulative. Refunds are capped at `gas_used / 5`, **at transaction level** (M6); now you only count them.

## Tests to write

| Bytecode | Gas | Refund |
|---|---:|---:|
| `6002600301` | 9 | 0 |
| `5f5f52` (MSTORE at 0) | 10 | 0 |
| M2's sum loop | 400 | 0 |
| `60015f55`, empty slot | 22105 | 0 |
| `60015f555f5f55`, empty slot | 22209 | 19900 |

Also: every cost with exactly enough gas and with one less; every SSTORE row (the EIP-2200 test table); cold then warm; a huge memory offset is OutOfGas without allocating; EXP with exponents 0, 255 and 256.

Derive the 400 by hand: 5 to start, 38 per iteration × 10, 15 to finish.

## Matching Foundry

Receipts include 21,000 plus calldata costs, which don't exist until M6. Compare the opcode trace:

```sh
anvil --hardfork cancun
cast rpc debug_traceTransaction <txhash>
```

Compare the first step's `gas` minus gas remaining at the end, **before refunds**. If totals differ, print your own per-opcode gas and find the **first** difference; that's the bug.

<details><summary>Hint: where to hold original values</summary>

Per slot, keep `original` and `current` together in the overlay. Set both when the slot first loads, and change `original` only at transaction boundaries.
</details>

## Done when

- [ ] ERC-20 transfer gas matches for a new recipient, an existing one, and a sender emptying their balance.
- [ ] The table above passes.
- [ ] Infinite loops end with OutOfGas and the test step limit is gone.

**Check yourself:** why is memory quadratic? Why does SSTORE need the original value? How would you make a transfer cheaper?

Next: [Milestone 5](05-calls-creation.md)
