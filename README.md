StellarVault Soroban contracts workspace

This workspace contains scaffolded Soroban contract templates for:

- access_roles
- transaction_proposal
- treasury
- delay_time

Build instructions (requires Rust + Cargo):

1. Install Rust (https://rustup.rs) and ensure `cargo` is on PATH.
2. From the repository root run:

```bash
cargo build --workspace
```

Note: Contracts depend on `soroban-sdk`. Ensure internet access to download crates. For Soroban-specific toolchains or versions, adjust `Cargo.toml` files inside `contracts/*` crates.
