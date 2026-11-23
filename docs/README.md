# bevm: build guide

**Final goal:** an EVM in Rust that runs real Ethereum bytecode, passes 90%+ of one fork's official state tests, and replays a real Base transaction with the same result as the chain.

You write the EVM by hand. These docs give you scope, interfaces, rules, traps and tests, not finished implementations.

## Roadmap

| # | Milestone | You end up with | Source lines | Test lines |
|---|---|---|---:|---:|
| 0 | [Setup](00-setup.md) | A crate that builds and tests | ~20 | 0 |
| 1 | [Stack machine](01-stack-machine.md) | A CLI that runs arithmetic bytecode | 450–650 | 300–400 |
| 2 | [Control flow and memory](02-control-flow-memory.md) | Compiled Solidity pure functions run | 350–500 | 300–400 |
| 3 | [State and environment](03-state-environment.md) | A real ERC-20 deploys and transfers | 600–850 | 300–450 |
| 4 | [Gas](04-gas.md) | Execution gas matches to the unit | 450–650 | 350–500 |
| 5 | [Calls and creation](05-calls-creation.md) | Proxies, factories, safe reverts | 700–1,000 | 500–700 |
| 6 | [Transactions and precompiles](06-transactions-precompiles.md) | Full signed transactions execute | 900–1,300 | 400–600 |
| 7 | [Official tests](07-official-tests.md) | 90%+ pass rate on Cancun | 900–1,400 | 150–300 |
| | **Milestones 1–7** | | **4,350–6,350** | **2,300–3,350** |
| 8 | [Make it real](08-base.md) | A Base transaction replayed | 1,350–2,100 | 250–400 |

Line counts are estimates, not targets. They cover new lines you write, excluding blanks and comments. Going over by 50% is normal while learning Rust; 3× over usually means you're repeating code that wants a helper. Milestone 8 covers forking, Base rules and tracing; performance work is extra.

## Mental model

| Piece | What it is | Lives for | Added in |
|---|---|---|---|
| Code + pc | Bytecode and the next position in it | one frame | M1 |
| Stack | Up to 1024 words of 256 bits | one frame | M1 |
| Memory | Growable byte array | one frame | M2 |
| Storage | Word → word map per contract | forever | M3 |
| Transient storage | Same shape as storage | one transaction | M3 |
| Context | Caller, value, calldata, block | frame / tx / block | M3 |
| Gas | Execution budget | one frame, returned to parent | M4 |
| Journal | Undo log of state changes | one transaction | M5 |

The loop: read the opcode at `pc`, do it, move `pc`, repeat until something halts.

```text
Transaction  (M6: validate, buy gas, refund, pay fees)
  └─ Call frames  (M5: snapshot, run, commit or roll back)
       └─ Interpreter  (M1–M4: stack, memory, pc, gas)
            └─ State  (M3: accounts, storage, code)
```

Outer layers know about inner ones, never the other way round.

## Suggested source layout

Create each file when its milestone needs it.

```text
src/
  lib.rs
  main.rs                CLI (M1)
  error.rs               halt reasons (M1)
  stack.rs               (M1)
  opcode.rs              constants (M1)
  memory.rs              (M2)
  code.rs                bytecode + jump analysis (M2)
  interpreter/
    mod.rs               the loop (M1)
    arithmetic.rs ...    opcode groups; split when mod.rs hurts (M2–M3)
  context.rs             block, tx and call info (M3)
  state.rs               trait + in-memory state (M3)
  gas.rs                 (M4)
  journal.rs, evm.rs     rollback, frames (M5)
  tx.rs, precompiles/    (M6)
  fork.rs, trie.rs       (M7)
tests/                   integration tests, one file per milestone
```

## Decisions made up front

- **Types:** `alloy-primitives` for `U256`, `Address`, `B256`, `Bytes` and `keccak256`. Its `U256` is `ruint`. The `B256`/`Address`/`U256` distinction lets the compiler catch a class of bugs.
- **First fork: Cancun.** It has the most stable documentation. Then move through Prague to **Osaka**, whose EVM changes Base adopted in its Azul upgrade. Skip Amsterdam: it reworks gas extensively.
- **Gas is `u64`**, balances are `U256`.
- **Guest behaviour never panics.** Any bytecode must give a result, an error or a revert. A panic is always your bug.

## How to work through a milestone

1. Read the whole guide, then the linked opcodes on [evm.codes](https://www.evm.codes/).
2. Build in the listed order. Every step should compile and have tests before the next.
3. Write expected values **before** running your code.
4. Open hints only after about 30 minutes stuck.
5. Tick the deliverables, commit, and tag it (`git tag m1`).

When sources disagree: fixtures, then [execution-specs](https://github.com/ethereum/execution-specs), then the EIP, then evm.codes. Avoid reading revm until you've finished the equivalent milestone.

## Changes from the original plan

- `SLT` and the other signed ops are **required in M2**, not a bonus: Solidity's argument decoding emits `SLT`. `CALLVALUE` also moves to M2, for the non-payable check.
- M4 compares **execution** gas. Receipts include intrinsic gas, which arrives in M6.
- M7 starts without a trie: current fixtures include the full expected post-state.
- M7 uses the `ethereum/execution-specs` releases, where fixtures are now published.

Facts about fixtures and Base were checked on **2026-11-22**. Recheck them when you reach M7 and M8.
