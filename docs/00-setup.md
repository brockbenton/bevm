# Milestone 0: Setup

**Deliverable:** `cargo test` runs clean in this repository.
**Size:** ~20 lines. **Time:** about a week, mostly Rust learning.

## Steps

- [ ] Run `cargo init --lib --name bevm` in the repo root (`init`, because the directory exists).
- [ ] Run `cargo add alloy-primitives` and commit `Cargo.lock`.
- [ ] Check that `.gitignore` contains `/target`, and add `/fixtures` for M7.
- [ ] Install VS Code with rust-analyzer, and `rustup component add clippy rustfmt`.
- [ ] Read [The Rust Book](https://doc.rust-lang.org/book/) chapters 1–10 alongside [rustlings](https://rustlings.rust-lang.org/). Chapter 11 (testing) is short and needed straight away.
- [ ] Bookmark [evm.codes](https://www.evm.codes/), [execution-specs](https://github.com/ethereum/execution-specs) and the [alloy-primitives docs](https://docs.rs/alloy-primitives).

Foundry isn't needed until M2.

## Warm-up test

Write one test that proves `U256::MAX` plus one wraps to zero. Find out what the `+` operator does on overflow versus `wrapping_add`, and use explicit `wrapping_*` calls in opcodes.

## Every commit

```sh
cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test
```

## Done when

- [ ] That command passes.
- [ ] You can explain `&T` vs `&mut T`, what `?` does, and why `match` must be exhaustive.

Next: [Milestone 1](01-stack-machine.md)
