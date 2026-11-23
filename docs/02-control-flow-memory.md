# Milestone 2: Control flow and memory

**Goal:** loops, memory, inputs and outputs.
**Deliverable:** a Solidity `add(uint256,uint256)` compiled by `solc` returns the right answer.
**Size:** 350–500 source lines, 300–400 test lines.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Jump analysis | `code.rs` | 40–60 |
| 2. JUMP, JUMPI, JUMPDEST, PC | `interpreter/` | 45–65 |
| 3. Memory + MLOAD, MSTORE, MSTORE8, MSIZE, MCOPY | `memory.rs` | 90–130 |
| 4. CALLDATALOAD, CALLDATASIZE, CALLDATACOPY | `interpreter/` | 60–85 |
| 5. RETURN, REVERT, outcome type | `interpreter/mod.rs` | 65–90 |
| 6. Signed ops + CALLVALUE | `arithmetic.rs` | 50–70 |

## Step 1: Jump analysis

Scan once from byte 0. On `0x5b`, mark the position. On a PUSHn, skip its n data bytes. Use a `Vec<bool>` first.

A `0x5b` inside PUSH data is not a destination. Otherwise an attacker could hide code inside constants.

## Step 2: Control flow

| Hex | Op | Behaviour |
|---|---|---|
| 56 | JUMP | `pc = a`; error unless valid |
| 57 | JUMPI | jump to `a` if `b != 0` |
| 58 | PC | position of this instruction |
| 5b | JUMPDEST | nothing |

JUMPI validates its destination **only when taken**. Destinations are `U256`; don't truncate them.

## Step 3: Memory

Memory grows in 32-byte words and starts zeroed. Reads expand it too. `MSIZE` is the active size in bytes, always a multiple of 32.

| Hex | Op | Operands |
|---|---|---|
| 51 | MLOAD | offset |
| 52 | MSTORE | offset, value |
| 53 | MSTORE8 | offset, value (lowest byte) |
| 59 | MSIZE | |
| 5e | MCOPY | dest, src, length |

- **Length zero never expands memory**, whatever the offset.
- MCOPY handles overlap, and expands for the larger of both ranges.
- Until M4 adds gas, cap memory (say 16 MiB) and return an error, or a bad offset will exhaust your RAM.
- Compute `offset + length` with checked arithmetic.

## Step 4: Calldata

Reads beyond the end give zeroes. A huge offset is legal and returns zero. CALLDATACOPY takes `dest, offset, length`.

## Step 5: Outcomes

Separate three things, because M5 depends on it:

```rust
// Success(output)     STOP or RETURN
// Revert(output)      undo state, keep remaining gas, return data
// Halt(error)         undo state, consume all gas, no data
```

`REVERT` is normal program behaviour. RETURN (`f3`) and REVERT (`fd`) take `offset, length`.

## Step 6: Signed operations

| Hex | Op | Watch out |
|---|---|---|
| 05 | SDIV | `MIN / -1 = MIN`; divide by zero = 0; truncates toward zero |
| 07 | SMOD | sign follows the dividend |
| 0b | SIGNEXTEND | `a` is the byte index of the sign byte, `b` the value; unchanged if `a >= 31` |
| 12, 13 | SLT, SGT | two's complement |
| 1d | SAR | a shift of 256 or more gives 0 or −1 |

For `CALLVALUE` (`34`), push a value passed in by the caller (zero for now). Solidity checks it on non-payable functions.

<details><summary>Hint: signed arithmetic</summary>

Bit 255 is the sign. Two's-complement negation is NOT then wrapping add 1. Work on absolute values, then reapply the sign. Comparisons only need special handling when the signs differ.
</details>

## Tests to write

**Sum 1 to 10.** This should return the 32-byte word `0x37` (55):

```text
5f600a5b80910190600190038060035750 5f5260205ff3      (remove the space)
```

Before running it, disassemble it by hand and trace the stack for two iterations. Counter and accumulator trade places with SWAPs.

Also: a jump into PUSH data fails; an untaken JUMPI to nowhere succeeds; MLOAD at 1 gives MSIZE 64; overlapping MCOPY both ways; RETURN of length 0 at offset MAX; calldata shorter than 32 bytes; SDIV, SMOD, SAR and SIGNEXTEND edges.

**Solidity.** [Install Foundry](https://getfoundry.sh/) and solc, then compile:

```solidity
pragma solidity ^0.8.20;
contract Adder {
    function add(uint256 a, uint256 b) external pure returns (uint256) { return a + b; }
}
```

```sh
solc --evm-version cancun --optimize --bin-runtime Adder.sol
cast sig 'add(uint256,uint256)'     # 0x771602f7
```

Calldata is the selector followed by two 32-byte big-endian arguments: 68 bytes. Use **runtime** bytecode; constructors come in M3. Keep the hex in the test so it doesn't depend on solc.

## Done when

- [ ] The loop returns 55.
- [ ] Solidity `add(2, 3)` returns 5.
- [ ] `add(MAX, 1)` **reverts** with 36 bytes: `Panic(uint256)` selector `4e487b71` plus code `0x11`.
- [ ] An unknown selector reverts with empty data.
- [ ] Random bytecode still never panics. Add a step limit, since loops can now be infinite.

**Check yourself:** where does Solidity keep its free memory pointer? What do the first instructions of your runtime code do?

Next: [Milestone 3](03-state-environment.md)
