# Milestone 5: Calls and contract creation

**Goal:** contracts that call and create other contracts, with exact rollback.
**Deliverable:** proxies and CREATE2 factories work, and a failed inner call leaves outer state intact.
**Size:** 700–1,000 source lines, 500–700 test lines. This is the hard one.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Journaled state | `journal.rs` | 170–240 |
| 2. Message and result types | `evm.rs` | 80–110 |
| 3. Frame driver | `evm.rs` | 150–210 |
| 4. CALL, CALLCODE, DELEGATECALL, STATICCALL | `interpreter/call.rs` | 130–190 |
| 5. CREATE, CREATE2 | `interpreter/create.rs` | 110–160 |
| 6. RETURNDATASIZE, RETURNDATACOPY | `interpreter/` | 25–40 |
| 7. SELFDESTRUCT | `interpreter/` | 35–50 |

Build and test the journal completely before touching calls.

## Step 1: Journal

Append one undo entry per change. A snapshot is the journal length; reverting pops entries and undoes them in reverse order; committing does nothing.

Journal all of: balance, nonce, code, storage, **transient storage**, account creation, touches, **warm accounts and slots**, logs, selfdestructs, and refunds (or restore those per frame).

Test with random operations: snapshot, change, revert, and compare against a clone taken at the snapshot.

## Step 2–3: Messages and frames

Keep one owned EVM object and pass `&mut` down the recursion. Frames own stack, memory, pc and gas, but **never hold references to state**.

```text
call(message):
  depth > 1024, or value > balance  → fail, return all gas
  snapshot
  transfer value
  run precompile (M6) or code
  Success → keep     Revert → undo, return remaining gas + data
  Halt    → undo, return no gas, no data
```

Recursing 1024 deep can overflow the 2 MiB stack Rust gives test threads. Run those in `std::thread::Builder::new().stack_size(...)`, and keep big structs off the frame.

## Step 4: Calls

| | CALL `f1` | CALLCODE `f2` | DELEGATECALL `f4` | STATICCALL `fa` |
|---|---|---|---|---|
| Code from | target | target | target | target |
| `address` (storage) | target | self | self | target |
| `caller` | self | self | parent's caller | self |
| `value` | given | given | parent's | 0 |
| Moves ETH | yes | no (balance still checked) | no | no |

Operands: `gas, address, [value,] in_offset, in_len, out_offset, out_len`; only CALL and CALLCODE take value.

Gas, in order:

1. Memory expansion for input and output.
2. Account access: 2600 cold, 100 warm.
3. +9000 if value ≠ 0 (CALL, CALLCODE).
4. +25000 if value ≠ 0 and the target is empty (CALL only).
5. Forward at most `remaining − floor(remaining / 64)`, or the requested gas if smaller.
6. Add a 2300 stipend to the callee if value ≠ 0, free to the caller.

Push 1 for success, 0 otherwise. Copy `min(out_len, returndata length)` bytes; leave the rest of the output area alone. Failing for depth or balance returns the forwarded gas, stipend included.

In a static frame, these halt: SSTORE, TSTORE, LOG*, CREATE*, SELFDESTRUCT, and CALL **with value**. The flag is inherited by all descendants.

## Step 5: Creation

- CREATE `f0`: `keccak256(rlp([sender, nonce]))[12..]`
- CREATE2 `f5`: `keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))[12..]`

CREATE takes `value, offset, length`; CREATE2 adds `salt`. A tiny RLP encoder for `[address, u64]` is about 20 lines.

| Rule | Value |
|---|---|
| Base | 32000 |
| Init code | 2 per word (CREATE2: +6 more for hashing) |
| Max init code | 49152 bytes; larger halts |
| Deposit | 200 per byte of runtime code |
| Max runtime code | 24576 bytes |
| Runtime starting `0xef` | rejected |

Sequence: charge → depth, balance, nonce-overflow checks (push 0, no gas lost) → bump creator nonce → warm new address → collision check (nonce ≠ 0, or code, or storage; consumes all forwarded gas) → snapshot → new nonce = 1 → transfer → run → validate and pay deposit → store.

The nonce bump and warming happen **before** the snapshot, so they survive failure. Creation forwards 63/64 of gas. Failing to pay the deposit is an exceptional halt. Push the address or 0.

## Step 6: Return data

A per-frame buffer replaced by each call. `3d` gives its size, `3e` copies it (`dest, offset, length`). **Reading out of bounds halts**, unlike calldata. Successful creation leaves it empty; a reverted creation leaves revert data.

## Step 7: SELFDESTRUCT (EIP-6780)

`ff` costs 5000, +2600 if the beneficiary is cold (no warm charge), +25000 if it's empty and balance ≠ 0. It always sends the balance. It deletes the account at transaction end **only if created in this same transaction**. If the beneficiary is itself, the ETH burns only in that case.

## Tests to write

- Reverted inner call: inner changes gone, outer kept; also a success inside a failing parent.
- Logs, transient storage, refunds and warmth roll back too.
- Each of the four calls: storage owner, caller and value.
- STATICCALL → CALL → SSTORE fails.
- Value to empty, existing and poor accounts; stipend.
- 63/64 at the exact boundary; depth 1024.
- Reentrancy: a vulnerable bank is drained, a guarded one isn't.
- RETURNDATACOPY one past the end halts.
- CREATE2: deployer zero, salt zero, init code `00` → `0x4D1A2e2bB4F88F0250f26Ffff098B0b30B26BF38`.
- Collision, oversized code, `0xef` code, reverting constructor.

EIP-1167 proxy runtime code:

```text
363d3d373d3d3d363d73<20-byte address>5af43d82803e903d91602b57fd5bf3
```

<details><summary>Hint: borrow checker fights</summary>

Copy what the child needs (calldata, code as reference-counted `Bytes`) into an owned message. If you feel you need `RefCell` or lifetimes on the frame, look for a field that should be a function parameter instead.
</details>

<details><summary>Hint: keeping opcodes simple</summary>

The opcode can just build a message and hand it to the driver, then apply the result to its own stack and memory. That keeps it testable and lets you switch to an explicit frame stack later.
</details>

## Done when

- [ ] Nested calls, proxy and factory all work, with storage in the right account.
- [ ] Failed calls roll back exactly their own subtree.
- [ ] Factory addresses match `cast create2`.
- [ ] Gas for a nested call matches a Foundry trace.
- [ ] 1024 nested calls don't crash.

**Check yourself:** why is an uninitialised proxy dangerous? What survives a failed CREATE, and why?

Next: [Milestone 6](06-transactions-precompiles.md)
