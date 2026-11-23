# Milestone 1: The stack machine

**Goal:** execute straight-line bytecode.
**Deliverable:** `cargo run -- 6002600301` prints a final stack containing `5`.
**Size:** 450–650 source lines, 300–400 test lines.

## What you're building

| Step | File | Source lines |
|---|---|---:|
| 1. Error enum | `error.rs` | 20–30 |
| 2. Stack | `stack.rs` | 80–110 |
| 3. Opcode constants | `opcode.rs` | 60–90 |
| 4. Interpreter loop, PUSH/DUP/SWAP/POP/STOP | `interpreter/mod.rs` | 90–130 |
| 5. Arithmetic, comparison, bitwise | `interpreter/arithmetic.rs` | 150–220 |
| 6. CLI | `main.rs` | 50–70 |

## Step 1: Errors

Start with stack underflow, stack overflow and invalid opcode (carry the byte). More arrive each milestone.

## Step 2: Stack

```rust
pub struct Stack { data: Vec<U256> }

// push(U256) -> Result<(), Error>      fails at 1024 items
// pop() -> Result<U256, Error>
// peek(depth) -> Result<U256, Error>   0 is the top
// dup(n), swap(n)                      n is 1..=16
```

`DUPn` copies the n-th item **counting from 1**. `SWAPn` exchanges the top with the item **n below it**, so it needs n + 1 items. `swap(16)` on a 16-item stack underflows.

## Step 3: Opcode constants

Plain `u8` constants work well in `match`. An enum also works but makes PUSH/DUP/SWAP ranges clumsy.

## Step 4: The loop

- Running off the end of the code is `STOP`, not an error.
- `PUSHn` reads n big-endian bytes and moves `pc` by n + 1.
- If the code ends mid-PUSH, missing bytes are zero **on the right**: `61ff` pushes `0xff00`.
- `0xfe` (INVALID) and unassigned bytes are errors.

## Step 5: Opcodes

`a` is the top of the stack, `b` the next, `c` the third. All arithmetic is modulo 2^256.

| Hex | Op | Result |
|---|---|---|
| 01, 02, 03 | ADD, MUL, SUB | `a + b`, `a * b`, `a - b` |
| 04, 06 | DIV, MOD | 0 when `b == 0` |
| 08, 09 | ADDMOD, MULMOD | 0 when `c == 0`; intermediate is **not** truncated to 256 bits |
| 0a | EXP | `a` to the power `b`; `0^0 = 1` |
| 10, 11, 14 | LT, GT, EQ | 1 or 0 |
| 15 | ISZERO | one operand |
| 16, 17, 18, 19 | AND, OR, XOR, NOT | |
| 1a | BYTE | byte `a` of `b`, 0 = most significant; 0 when `a >= 32` |
| 1b, 1c | SHL, SHR | shift **`b` by `a`**; 0 when `a >= 256` |
| 50 | POP | |
| 5f, 60–7f | PUSH0, PUSH1–32 | |
| 80–8f, 90–9f | DUP1–16, SWAP1–16 | |

Check a `U256` fits before converting it to `usize`. A silent truncation turns a shift of 2^64 into a shift of 0.

Defer the signed ops to M2.

## Step 6: CLI

Accept hex with or without `0x`, print the stack top first, print errors to stderr and exit nonzero.

## Tests to write

At least one normal and one edge case per opcode, plus:

| Bytecode | Result |
|---|---|
| `6002600301` | `[5]` |
| `6003600a03` | `[7]` (a huge number means reversed operands) |
| `5f600a04` | `[0]` (10 / 0) |
| `61ff` | `[0xff00]` |
| `01` | stack underflow |
| 1024 × `5f` | succeeds |
| 1025 × `5f` | stack overflow |

Also: `MAX + 1 = 0`; `0 - 1 = MAX`; `ADDMOD(MAX, MAX, 7)`; `MULMOD(MAX, MAX, 12)`; shifts by 255, 256 and 2^64; BYTE at 0, 31 and 32. Take expected values from the evm.codes playground.

<details><summary>Hint: repetitive opcodes</summary>

After writing three binary opcodes by hand, extract a helper that pops two values, takes a closure and pushes the result. `FnOnce(U256, U256) -> U256` is enough.
</details>

<details><summary>Hint: reading PUSH data</summary>

Fill a zeroed `[u8; 32]`. Work out where an n-byte value must start for a big-endian word, then copy however many bytes are available.
</details>

## Rust you'll use

Structs, `impl`, `Vec`, range patterns (`0x60..=0x7f`), `Result`, `?`, enums with data, `#[test]`, closures.

## Done when

- [ ] Every opcode above works and is tested.
- [ ] The CLI handles the sample programs.
- [ ] Random bytes never panic. Loop over a few thousand random inputs in a test.

**Check yourself:** why does the top of the stack come first in SUB? Why must ADDMOD use a wider intermediate?

Next: [Milestone 2](02-control-flow-memory.md)
