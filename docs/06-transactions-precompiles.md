# Milestone 6: Transactions and precompiles

**Goal:** execute a whole transaction, not just a call.
**Deliverable:** a signed raw transaction gives correct balances, nonces, logs and gas.
**Size:** 900–1,300 source lines, 400–600 test lines.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Transaction types, fee math | `tx.rs` | 130–180 |
| 2. RLP decoding, sender recovery | `tx.rs` | 120–180 |
| 3. Validation, intrinsic gas | `tx.rs` | 130–190 |
| 4. Execute and settle | `evm.rs` | 150–210 |
| 5. Precompile dispatch + simple ones | `precompiles/` | 110–160 |
| 6. MODEXP | `precompiles/modexp.rs` | 100–140 |
| 7. bn254, KZG | `precompiles/` | 100–150 |
| 8. BLAKE2F | `precompiles/blake2f.rs` | 60–90 |

## Step 1–2: Types

Legacy (with and without EIP-155), type 1 (access list), 2 (EIP-1559), 3 (blob). Type 4 (EIP-7702) waits for Prague. Use `alloy-rlp` and `k256`, or write the RLP yourself (about 150 extra lines, good practice, reusable in M7). Reject high-s signatures.

`effective = min(max_fee, base_fee + max_priority)`.

## Step 3: Validation

An invalid transaction changes **nothing** and differs from a failed one, which still pays. Check:

- chain ID; nonce equals the account's; nonce below 2^64 − 1
- `max_fee ≥ base_fee` and `max_fee ≥ max_priority`
- sender has no code (EIP-3607)
- balance ≥ `gas_limit × max_fee + value` (+ max blob fee)
- gas limit ≥ intrinsic, and ≤ block gas limit
- creation init code ≤ 49152 bytes
- blobs: at least 1, at most 6 on Cancun, valid version byte, max blob fee ≥ blob base fee, `to` present

Intrinsic gas:

| Part | Gas |
|---|---:|
| Base | 21000 |
| Calldata | 4 per zero byte, 16 per nonzero |
| Creation | 32000 + 2 per init code word |
| Access list | 2400 per address, 1900 per key |

## Step 4: Execute and settle

1. Validate.
2. Deduct `gas_limit × effective` (and blob fee: 131072 per blob × blob base fee, burned).
3. Bump sender nonce. A creation address uses the **old** nonce.
4. Warm sender, target, coinbase, precompiles, access list.
5. Run the top-level call or creation with `gas_limit − intrinsic`.
6. Cap refund at `gas_used / 5`. Refunds survive only if the top level succeeded.
7. Return unused gas to the sender at `effective`.
8. Pay coinbase `gas_used × (effective − base_fee)`. The base fee is burned.
9. Delete same-transaction selfdestructs and touched empty accounts (EIP-161).
10. Clear transient storage, warm sets, originals; return a receipt.

The balance **check** uses `max_fee`; the **deduction** uses `effective`. Fees and the nonce bump sit outside the frame snapshot, so they survive failure.

## Steps 5–8: Precompiles

| Addr | Name | Gas | Suggested crate |
|---|---|---|---|
| 01 | ecrecover | 3000 | `k256` |
| 02 | SHA-256 | 60 + 12/word | `sha2` |
| 03 | RIPEMD-160 | 600 + 120/word | `ripemd` |
| 04 | identity | 15 + 3/word | none |
| 05 | MODEXP | EIP-2565, min 200 | `num-bigint` |
| 06 | bn254 add | 150 | `ark-bn254` |
| 07 | bn254 mul | 6000 | `ark-bn254` |
| 08 | bn254 pairing | 45000 + 34000/pair | `ark-bn254` |
| 09 | BLAKE2F | rounds | write it |
| 0a | KZG point evaluation | 50000 | `c-kzg` |

- Two failure modes: ecrecover with a bad signature **succeeds with empty output**; bn254 with bad points **fails and consumes all forwarded gas**.
- Pad short input with zeroes for ecrecover, MODEXP and bn254 add/mul; pairing needs a multiple of 192, BLAKE2F exactly 213, KZG exactly 192.
- ecrecover accepts v of 27 or 28 only, as a full 32-byte word, and **does** accept high s.
- bn254 coordinates must be below the field modulus, on the curve, and for G2 in the subgroup. Infinity is (0, 0). G2 encodes the imaginary part first.
- MODEXP length fields may be enormous: charge before allocating.
- DELEGATECALL to a precompile runs it normally.

## Tests to write

Each invalid-transaction rule leaves state untouched; a reverting transaction still pays and bumps its nonce; creation success and revert; refund cap hit and not hit; access list effect on gas; each precompile with EIP test vectors, empty input and too little gas.

**Conservation test:** total ETH before equals total after plus burned base and blob fees. Run it after every transaction test.

## Done when

- [ ] A raw transaction from `cast mktx` matches Anvil's receipt: status, gas used, logs, contract address, balances.
- [ ] The same for a failing transaction and a creation.
- [ ] All ten precompiles pass their vectors.
- [ ] Conservation holds everywhere.

**Check yourself:** why does a failed transaction cost anything? Who receives each part of the fee?

Next: [Milestone 7](07-official-tests.md)
