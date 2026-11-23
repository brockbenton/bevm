# Milestone 3: State, storage and the environment

**Goal:** contracts that remember things and see the world.
**Deliverable:** an ERC-20 deploys from init code, transfers, and emits events.
**Size:** 600–850 source lines, 300–450 test lines.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Account types, state trait, in-memory state | `state.rs` | 130–180 |
| 2. Block, tx and call context | `context.rs` | 80–110 |
| 3. SLOAD, SSTORE, TLOAD, TSTORE | `interpreter/storage.rs` | 70–100 |
| 4. KECCAK256, LOG0–4 | `interpreter/` | 70–100 |
| 5. Environment opcodes | `interpreter/env.rs` | 130–190 |
| 6. Block opcodes | `interpreter/block.rs` | 60–80 |
| 7. Deploy/call test harness | `tests/common/` | 60–90 |

## Step 1: State

```rust
struct Account { nonce: u64, balance: U256, code_hash: B256 }

trait Database {
    fn account(&mut self, a: Address) -> Result<Option<Account>, DbError>;
    fn storage(&mut self, a: Address, slot: U256) -> Result<U256, DbError>;
    fn code(&mut self, hash: B256) -> Result<Bytes, DbError>;
    fn block_hash(&mut self, number: u64) -> Result<B256, DbError>;
}
```

Reads take `&mut self` and return `Result` because in M8 they become cached network calls. Keep **writes out of the trait**: they go in an overlay you own, which M5 turns into the journal.

A nonexistent account and an existing empty one differ. Never-written slots read as zero.

## Step 2: Context

Three lifetimes, three structs:

- **Block:** number, timestamp, coinbase, gas limit, base fee, prevrandao, chain ID, blob base fee.
- **Transaction:** origin, effective gas price, blob hashes.
- **Call:** address (whose storage), caller, value, calldata, code, static flag.

Keep `address` separate from where the code came from. DELEGATECALL needs that in M5.

## Step 3: Storage

`54` SLOAD(key), `55` SSTORE(key, value), `5c` TLOAD, `5d` TSTORE. Transient storage is keyed by `(address, key)` and cleared after each transaction.

## Step 4: Hashing and logs

`20` KECCAK256(offset, length). Keccak-256 is **not** NIST SHA3-256; use alloy's. `a0`–`a4` LOGn takes `offset, length, topic1..n` and records address, topics and data.

## Step 5: Environment

| Hex | Op | Hex | Op |
|---|---|---|---|
| 30 | ADDRESS | 39 | CODECOPY |
| 31 | BALANCE | 3a | GASPRICE |
| 32 | ORIGIN | 3b | EXTCODESIZE |
| 33 | CALLER | 3c | EXTCODECOPY |
| 34 | CALLVALUE | 3f | EXTCODEHASH |
| 38 | CODESIZE | 47 | SELFBALANCE |

EXTCODEHASH is **0** for a nonexistent or empty account, and `keccak256("")` (`c5d24601…`) for an existing one without code. Addresses come from the low 20 bytes of a word; ignore the top 12.

## Step 6: Block

`40` BLOCKHASH, `41` COINBASE, `42` TIMESTAMP, `43` NUMBER, `44` PREVRANDAO, `45` GASLIMIT, `46` CHAINID, `48` BASEFEE, `49` BLOBHASH, `4a` BLOBBASEFEE.

BLOCKHASH covers only the previous 256 blocks, and is zero for the current block or older ones. BLOBHASH out of range is zero.

## Step 7: Harness

`deploy(init_code)` runs code with empty calldata and stores whatever it **returns** as runtime code. `call(to, data, from)` runs stored code. On Revert or Halt, discard changes; cloning the in-memory state beforehand is fine for now.

## Solidity storage layout

- Variables fill slots from 0, packed when several fit.
- `mapping[k]` at slot p: `keccak256(pad32(k) ++ pad32(p))`.
- Dynamic array: length at slot p, elements from `keccak256(pad32(p))`.

## Tests to write

Round trips and unset reads for storage; isolation between addresses; transient storage cleared between transactions; logs with 0 and 4 topics; CODECOPY past the end gives zeroes; BLOCKHASH boundaries; both EXTCODEHASH empty cases; state discarded on revert.

**ERC-20.** Compile a small one (about 30 lines, or OpenZeppelin's) with `--evm-version cancun --bin`. Constructor arguments are ABI-encoded and **appended to init code**.

## Done when

- [ ] Init code produces runtime code matching `--bin-runtime` (no immutables).
- [ ] Deployer's `balanceOf` (`70a08231`) equals the supply.
- [ ] `transfer` (`a9059cbb`) moves balances and returns 1.
- [ ] One Transfer log: topic 0 is `ddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`, with both addresses in topics.
- [ ] You compute a balance's storage slot by hand and read it directly from state.
- [ ] An excessive transfer reverts and changes nothing.

**Check yourself:** why isn't deployed code the same as the init code? Which context fields change on a nested call?

Next: [Milestone 4](04-gas.md)
