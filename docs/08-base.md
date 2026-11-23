# Milestone 8: Make it real

**Goal:** match a live chain.
**Deliverable:** replay a Base transaction and reproduce its receipt: status, gas used, logs and fees.
**Size:** 1,350–2,100 source lines, 250–400 test lines, for tracks A–C.

| Track | Source lines | Needed for the goal? |
|---|---:|---|
| A. Forking state over JSON-RPC | 600–900 | yes |
| B. Base execution rules | 500–800 | yes |
| C. Tracing | 250–400 | strongly advised; do it first |
| D. Performance | open | no |

Order: **C → A → B → D.**

## Track C: Tracing

Add a hook with a default no-op: before each opcode, after it, on frame enter and exit. Emit `debug_traceTransaction` struct logs: `pc, op, gas, gasCost, depth, stack` (memory and storage optional). Write a differ that stops at the first mismatch and shows ten preceding steps.

Done when a trace of an ERC-20 transfer matches Anvil step for step.

## Track A: Forking

Implement M3's `Database` trait over RPC: `eth_getBalance`, `eth_getTransactionCount`, `eth_getCode`, `eth_getStorageAt`, `eth_getBlockByNumber`.

- Query at the **parent block**, always pinned, never `latest`.
- Cache to disk as JSON. Tests run offline from that cache.
- Batch requests; retry on rate limits.
- Old blocks need an archive endpoint.
- The trait's error path matters: network failures abort the replay, and never read as zero.

The parent block's state isn't your transaction's starting state: earlier transactions in the block ran first. Either execute them all, or get the exact pre-state with `debug_traceTransaction` and `prestateTracer`. Use the latter first; replay a whole block as the stretch goal.

Warm up on **Ethereum mainnet** with a recent transaction, so Base rules aren't involved. Check which fork is active for that block: Amsterdam has gas rules you don't support.

## Track B: Base rules

Start from Osaka and read the [Base upgrades list](https://docs.base.org/upgrades/overview). Relevant history as of 2026-11-22:

| Upgrade | Mainnet | Execution effect |
|---|---|---|
| Fjord | 2024-07-10 | FastLZ L1 fee, P256 precompile |
| Granite | 2024-09-11 | bn254 pairing input limit |
| Isthmus | 2025-05-09 | Pectra EIPs, operator fee |
| Jovian | 2025-12-02 | operator fee formula, tighter precompile input limits |
| Azul | 2026-05-28 | Osaka execution EIPs |
| Beryl | 2026-06-25 | B20 tokens as **native precompiles** |
| Cobalt | 2026-09-30 | B20 changes, validity transactions |
| Denim | 2026-11-05 | 200 ms blocks, BaseTime predeploy |

**Choose a transaction between Azul and Beryl (2026-05-28 to 2026-06-25)** for your first replay. That gives Osaka rules without B20. This table comes from the upgrade overviews; read the exec-engine page for the fork you pick.

### Deposit transactions (type 0x7E)

Mint first, so it survives failure. No signature, fee or nonce check, no L1 or operator fee, no gas refund. The sender nonce still increments. The first transaction in every block is a deposit that updates the L1Block contract.

### L1 data fee (Fjord onward)

```text
scaled = l1BaseFeeScalar * l1BaseFee * 16 + l1BlobFeeScalar * l1BlobBaseFee
size   = max(100 * 1e6, -42_585_600 + 836_500 * fastlzSize)
l1Fee  = size * scaled / 1e12
```

`fastlzSize` is the FastLZ-compressed length of the full signed transaction. You need only the length, about 70 lines, but it must match the reference compressor exactly. The subtraction can go negative. Parameters come from L1Block (`0x4200000000000000000000000000000000000015`).

### Operator fee (Jovian onward)

`gas × operatorFeeScalar × 100 + operatorFeeConstant`. Charge with the gas limit up front and refund down to gas used. The scalar (`u32`) and constant (`u64`) are in L1Block slot 8, at byte offsets 0 and 4.

### Fee recipients

| Fee | Goes to |
|---|---|
| Priority | block coinbase (the sequencer fee vault, `0x4200…0011`) |
| Base | base fee vault, `0x4200…0019` (**not burned**) |
| L1 data | L1 fee vault, `0x4200…001A` |
| Operator | operator fee vault, `0x4200…001B` |

Both extra fees count in the sender's balance check.

### Precompile limits (Jovian)

bn254 pairing 81,984 bytes; BLS G1 MSM 288,960; G2 MSM 278,784; BLS pairing 156,672. P256VERIFY costs 6900 from Azul.

### Replay ladder

1. Plain ETH transfer.
2. ERC-20 transfer (an ordinary bytecode token).
3. A swap through several contracts.
4. A deposit.
5. A reverted transaction.
6. Stretch: every transaction in a block.

For each, compare status, gas used, logs, `l1Fee`, and sender and vault balances.

## Track D: Performance

Benchmark before changing anything (`criterion`; compare with revm on the same bytecode): the sum loop, an ERC-20 transfer, repeated KECCAK256, deep calls. Profile with `cargo flamegraph`. Usual wins: caching jump analysis by code hash, avoiding code clones, a fixed-capacity stack, reusing memory buffers, table dispatch. Rerun the M7 suite after each change.

## Done when

- [ ] Steps 1–3 of the ladder match exactly and run offline from cached state.
- [ ] Traces can be diffed against a real node.
- [ ] The README shows your pass rate and the replayed transaction hash.

You've built a working EVM.
