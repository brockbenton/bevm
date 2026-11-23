# Milestone 7: The official tests

**Goal:** measure correctness against Ethereum's own suite.
**Deliverable:** a runner reporting 90%+ on Cancun state tests, with a useful diff for each failure.
**Size:** 900–1,400 source lines, 150–300 test lines. Most time goes into debugging.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Fork parameter | `fork.rs` | 100–150 |
| 2. Fixture parsing | `bin/statetest.rs` | 150–220 |
| 3. Runner, diff, report | `bin/statetest.rs` | 200–300 |
| 4. Logs hash | `bin/statetest.rs` | 40–60 |
| 5. Merkle Patricia Trie | `trie.rs` | 200–320 |
| 6. Fixes across the codebase | everywhere | 210–350 |

Prague and Osaka support adds 400–650 lines (not counted above).

## Get the fixtures

Releases are on [ethereum/execution-specs](https://github.com/ethereum/execution-specs/releases), tagged `tests@vX.Y.Z`. This was current on 2026-11-22:

```sh
mkdir -p fixtures
curl -fL -o fixtures/fixtures.tar.gz \
  'https://github.com/ethereum/execution-specs/releases/download/tests%40v21.0.3/fixtures.tar.gz'
tar -xzf fixtures/fixtures.tar.gz -C fixtures --wildcards 'fixtures/state_tests/for_cancun/*'
```

The archive is 945 MB compressed; Cancun state tests take about 376 MB extracted. Keep `fixtures/` in `.gitignore` and pin the version so your numbers stay comparable.

| Fork (v21.0.3) | Files | Cases |
|---|---:|---:|
| `for_cancun` | 2,089 | 11,569 |
| `for_prague` | 2,228 | 13,919 |
| `for_osaka` | 2,192 | 14,664 |

1,870 of the Cancun files are in `ported_static`, the older broad suite.

## Step 1: Forks

Pass a `Fork` enum (ordered, so `fork >= Fork::Prague` works) explicitly. Only support Cancun and later.

## Step 2: Format

Each file maps test names to cases:

```text
config        chainid, blobSchedule
env           currentNumber, currentTimestamp, currentCoinbase, currentBaseFee,
              currentGasLimit, currentRandom, currentExcessBlobGas, ...
pre           address → { nonce, balance, code, storage }
transaction   gasLimit[], value[], data[], accessLists[], sender, to, nonce, ...
post.<Fork>[] { indexes, hash, logs, state, receipt, txbytes, expectException? }
```

`indexes` selects elements from `data`, `gasLimit` and `value`; the access list uses the data index. An empty `to` is a creation. Use `serde`, `serde_json` and `sender` as given.

If `expectException` is present (592 Cancun cases), the transaction must be **rejected** and state stay unchanged. Blob base fee derives from excess blob gas, using the config's update fraction. You need that math from EIP-4844.

## Step 3: Runner

Compare against **`post.state`** first: it gives the full expected accounts, so you can report "slot 3: expected 5, got 0". A root only says something is wrong.

- Catch panics and count them separately, in a thread with a big stack.
- Filter by path or test name; print pass, fail, panic per directory.
- Add a threaded mode (`rayon`) and a timeout.
- Write results to a file so a later run shows what newly broke.

Also check `receipt` gas used, and `logs`, which is `keccak256(rlp(logs))`.

## Step 5: The trie

After most cases pass. Keys are `keccak256(address)` or `keccak256(slot)`; nodes are branch, extension and leaf; children whose RLP is under 32 bytes are inlined. Omit zero slots and empty accounts. Empty root is `keccak256(rlp(""))`: `56e81f17…b421`.

Building a root from a complete sorted map is far simpler than an updatable trie, and is all you need. Compare to `alloy-trie` in a test.

## Work order

Run in this order, fixing one group before the next:

1. `frontier/opcodes`, `constantinople/eip145_bitwise_shift`
2. `berlin`, `istanbul/eip2200_net_gas_metering`
3. `cancun/eip1153_tstore`, `cancun/eip5656_mcopy`, `constantinople/eip1014_create2`
4. `cancun/eip6780_selfdestruct`, `shanghai`, `byzantium`
5. `cancun/eip4844_blobs`
6. `ported_static`

Fix the failure with the smallest gas limit and fewest accounts first. If only the coinbase and sender balances differ, it's a gas bug: the gap divided by the gas price is the exact error, and numbers like 2500, 2300 or 4800 identify it. For the rest, diff traces with geth's `evm statetest --trace` (M8 tracing helps here too).

## Next forks

**Prague:** EIP-7702 set-code transactions, EIP-7623 calldata floor, EIP-2537 BLS precompiles (0x0b–0x11, `blst`), EIP-2935 history contract.

**Osaka:** EIP-7939 CLZ (`1e`, 5 gas), EIP-7951 P256VERIFY (0x100, 6900 gas), EIP-7823 and EIP-7883 MODEXP changes, EIP-7825 gas limit cap of 2^24.

With EIP-7623, apply refunds **first**, then take the maximum with the floor.

## Done when

- [ ] No panics on any Cancun case.
- [ ] 90%+ of Cancun passes by post-state, gas and logs. Then push for 100%; the rest is finite.
- [ ] State roots match on everything that passes.
- [ ] CI runs a small fixture subset.
- [ ] Stretch: Osaka at 90%+, which M8 wants.

Keep a log as you go:

| Date | Fork | Passed / total | What you fixed |
|---|---|---|---|

Next: [Milestone 8](08-base.md)
