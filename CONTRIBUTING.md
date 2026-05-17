# Contributing to TrustLink — Soroban Escrow Contract

Thank you for your interest in contributing to TrustLink! This repository is the trustless core of the TrustLink protocol — a Soroban smart contract written in Rust that powers secure escrow for social commerce on Stellar.

We welcome contributions of all kinds: bug fixes, new features, tests, documentation improvements, and security reviews. Every merged contribution moves us closer to making fraud-free social commerce a reality for millions of people.

---

## 📋 Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Stellar Wave Program](#stellar-wave-program)
- [Before You Start](#before-you-start)
- [Development Setup](#development-setup)
- [Project Structure](#project-structure)
- [Making Changes](#making-changes)
- [Commit Convention](#commit-convention)
- [Pull Request Process](#pull-request-process)
- [Writing Tests](#writing-tests)
- [Security Vulnerabilities](#security-vulnerabilities)
- [Getting Help](#getting-help)

---

## Code of Conduct

This project follows a simple rule: **be respectful, be constructive, be helpful**. We're building open infrastructure for underserved commerce communities — everyone who contributes deserves a welcoming environment regardless of experience level.

Harassment, gatekeeping, or dismissive behaviour will not be tolerated. Report issues to the maintainers via the contact below.

---

## 🌊 Stellar Wave Program

This repository participates in the **[Stellar Wave Program](https://www.drips.network/wave/stellar)** — a funded, sprint-based contribution initiative by the Stellar Development Foundation. During active Wave cycles, contributors can earn real rewards for resolving labelled issues.

### How Wave Contributions Work

1. Browse issues tagged [`Stellar Wave`](../../issues?q=label%3A%22Stellar+Wave%22)
2. Sign in at [drips.network/wave](https://www.drips.network/wave) with your GitHub account
3. Apply to the issue you want to work on
4. Wait to be assigned by a maintainer (we review applications promptly)
5. Submit a Pull Request before the Wave cycle ends
6. Earn Points that translate to XLM rewards

### Issue Point Values

| Complexity Label | Points | Typical Scope |
|---|---|---|
| `complexity: trivial` | 100 pts | Typo, comment fix, minor error code addition |
| `complexity: medium` | 150 pts | New test case, view function, bug fix |
| `complexity: high` | 200 pts | New contract function, refactor, security improvement |

> ⚡ **Speed matters.** Maintainers assign contributors quickly during active Waves — apply early and have your dev environment ready before you apply.

---

## Before You Start

### Find Something to Work On

- **New to Soroban?** → Start with [`good first issue`](../../issues?q=label%3A%22good+first+issue%22) labels. These are deliberately scoped and well-documented.
- **Experienced with Rust/Soroban?** → Look at [`complexity: high`](../../issues?q=label%3A%22complexity%3A+high%22) issues or check the roadmap section of the README.
- **Have an idea?** → Open a [GitHub Discussion](../../discussions) first before building. This prevents duplicate effort and ensures your PR gets merged.

### Check Before You Build

- Is there already an open PR for this issue? Check the issue's linked PRs.
- Is there a comment from a maintainer saying the approach has changed? Read the full thread.
- For anything beyond a trivial fix — comment on the issue to briefly describe your intended approach. A maintainer will confirm before you spend time building.

---

## Development Setup

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Rust | `1.75+` | [rustup.rs](https://rustup.rs) |
| Stellar CLI | `21+` | [Stellar Docs](https://developers.stellar.org/docs/tools/stellar-cli) |
| wasm32 target | latest | `rustup target add wasm32-unknown-unknown` |

### First-Time Setup

```bash
# 1. Fork this repo on GitHub, then clone your fork
git clone https://github.com/YOUR_USERNAME/trustlink-contract
cd trustlink-contract

# 2. Add the upstream remote
git remote add upstream https://github.com/your-org/trustlink-contract

# 3. Install Rust and the wasm target
rustup update stable
rustup target add wasm32-unknown-unknown

# 4. Verify everything builds cleanly
cargo build --target wasm32-unknown-unknown --release

# 5. Run the full test suite — all tests must pass on a clean checkout
cargo test
```

### Staying Up to Date

```bash
git fetch upstream
git rebase upstream/main
```

Always rebase onto `main` before opening a PR.

---

## Project Structure

```
trustlink-contract/
├── src/
│   ├── lib.rs          # Contract entry point — public-facing interface only
│   ├── escrow.rs       # State machine logic — the heart of the contract
│   ├── storage.rs      # Persistent storage read/write helpers
│   ├── events.rs       # On-chain event definitions and emitters
│   ├── errors.rs       # All custom ContractError codes live here
│   └── types.rs        # Shared structs (EscrowData, EscrowState, etc.)
├── tests/
│   ├── happy_path.rs   # Full end-to-end flow tests
│   ├── dispute_flow.rs # Dispute and resolution scenario tests
│   ├── edge_cases.rs   # Boundary conditions and attack vectors
│   └── helpers.rs      # Reusable test fixtures and mock setup
└── Cargo.toml
```

**Where to make changes:**

- New contract features → `escrow.rs` + `lib.rs` (expose the function) + `events.rs` (emit an event)
- New error codes → `errors.rs` only — never use raw `panic!()` in contract code
- New data fields → `types.rs` — be mindful of storage cost on Stellar
- New tests → add to the appropriate file in `tests/`, or create a new file for a new scenario group

---

## Making Changes

### Branching

Always work on a feature branch off `main`:

```bash
git checkout main
git pull upstream main
git checkout -b feat/your-feature-name
```

Branch naming conventions:

| Type | Pattern | Example |
|---|---|---|
| New feature | `feat/short-description` | `feat/multi-asset-support` |
| Bug fix | `fix/short-description` | `fix/auto-release-double-sign` |
| Test addition | `test/short-description` | `test/dispute-edge-cases` |
| Documentation | `docs/short-description` | `docs/improve-storage-comments` |
| Refactor | `refactor/short-description` | `refactor/storage-key-naming` |

### Coding Standards

**General Rust**
- Run `cargo fmt` before every commit — the CI will reject unformatted code
- Run `cargo clippy -- -D warnings` — fix all warnings, never suppress them without a comment
- Prefer explicit error returns over `unwrap()` — use `ContractError` variants from `errors.rs`
- Comment non-obvious logic. If you had to think about it for more than 30 seconds, leave a comment

**Soroban-specific**
- Every state-mutating function must call `require_auth()` on the appropriate address before any logic
- Use `env.storage().instance()` for contract-level data and `env.storage().persistent()` for per-escrow data — understand the cost difference
- Emit an event in `events.rs` for every meaningful state transition — the backend oracle depends on these
- Never use `env.storage().temporary()` for data that must survive ledger expiry
- Storage keys must be defined as constants in `storage.rs` — no raw strings inline

**Error Handling**
```rust
// ✅ Correct
if escrow.state != EscrowState::Funded {
    return Err(ContractError::InvalidState);
}

// ❌ Wrong — panics are not catchable and give no useful error code
if escrow.state != EscrowState::Funded {
    panic!("wrong state");
}
```

---

## Commit Convention

This repo uses [Conventional Commits](https://www.conventionalcommits.org/). PRs with non-conventional commit messages will be asked to squash/rebase.

```
<type>(<scope>): <short imperative description>

[optional body]

[optional footer: closes #123]
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | New contract function or capability |
| `fix` | Bug fix |
| `test` | Adding or improving tests |
| `docs` | Comments, README, or documentation changes |
| `refactor` | Code restructuring with no behaviour change |
| `chore` | Dependency updates, CI config, tooling |
| `security` | Security hardening or vulnerability fix |

**Examples:**

```bash
git commit -m "feat(escrow): add multi-asset support for SEP-41 tokens"
git commit -m "fix(auto-release): prevent double-signing when delivery timestamp races"
git commit -m "test(dispute): add edge case for expired dispute window"
git commit -m "docs(storage): clarify TTL behaviour for instance storage keys"
```

---

## Pull Request Process

### Before Opening a PR

```bash
# Format
cargo fmt

# Lint — zero warnings allowed
cargo clippy -- -D warnings

# Build
cargo build --target wasm32-unknown-unknown --release

# Test — all must pass
cargo test

# Check WASM binary size hasn't ballooned unexpectedly
ls -lh target/wasm32-unknown-unknown/release/trustlink_escrow.wasm
```

### PR Checklist

When you open a PR, the description must include:

- [ ] **What** — A clear description of what changed and why
- [ ] **How** — Brief explanation of your approach (especially for non-obvious choices)
- [ ] **Tests** — What test cases did you add or modify?
- [ ] **Breaking changes** — Does this change the contract ABI? (requires extra review)
- [ ] **Issue reference** — `Closes #123` or `Relates to #123`

### PR Template

```markdown
## Summary
<!-- What does this PR do? -->

## Motivation
<!-- Why is this change needed? Link to the issue. -->

## Changes
<!-- List the key changes -->
- 

## Test Coverage
<!-- What tests were added/modified? -->
- 

## Notes for Reviewers
<!-- Anything the reviewer should pay special attention to? -->

Closes #
```

### Review Process

- A maintainer will review your PR within **48 hours** during active Wave cycles, and within **5 business days** otherwise
- At least **1 approving review** is required to merge
- For changes touching the release logic, dispute resolution, or fee calculation — **2 approving reviews** are required
- The CI pipeline must be green (build + tests + clippy + fmt) before merge
- Maintainers may request changes — please respond within 5 days or the PR may be closed

### What Maintainers Look For

- Does the code actually solve the stated problem?
- Are the auth checks correct and in the right order?
- Is the new code covered by tests?
- Does the storage model make sense (cost, TTL)?
- Are events emitted for state changes the backend oracle needs to observe?
- Is the WASM binary size increase justified?

---

## Writing Tests

All Soroban contract tests live in the `tests/` directory and use the `soroban_sdk::testutils` environment.

### Test Structure

```rust
// tests/happy_path.rs

#[cfg(test)]
mod tests {
    use soroban_sdk::{testutils::Address as _, Address, Env};
    use crate::helpers::{setup_contract, mint_usdc};

    #[test]
    fn test_full_escrow_flow() {
        let env = Env::default();
        let (contract_id, vendor, buyer) = setup_contract(&env);

        // 1. Create escrow
        let escrow_id = client.create_escrow(&vendor, &buyer, &token, &amount, &window);

        // 2. Buyer funds
        mint_usdc(&env, &buyer, amount);
        client.fund_escrow(&escrow_id);

        // 3. Vendor ships
        client.mark_shipped(&escrow_id, &String::from_str(&env, "TRK123"));

        // 4. Buyer confirms
        client.confirm_delivery(&escrow_id);

        // Assert vendor received funds minus fee
        assert_eq!(token_client.balance(&vendor), expected_payout);
    }
}
```

### Test Coverage Requirements

New features must include tests for:
1. The happy path (expected usage)
2. At least one unauthorized access attempt (wrong caller)
3. Invalid state transitions (calling a function out of sequence)
4. Edge case inputs (zero amounts, expired windows, etc.)

### Running Specific Tests

```bash
# Run a single test
cargo test test_full_escrow_flow

# Run all tests in a module
cargo test dispute_flow

# Run with output (useful for debugging)
cargo test -- --nocapture
```

---

## Security Vulnerabilities

**Do not open a public GitHub issue for security vulnerabilities.**

If you discover a security issue in the contract — especially anything related to fund drainage, unauthorized release, or state manipulation — please report it privately:

📧 **security@trustlink.xyz** (or the contact listed in [SECURITY.md](SECURITY.md))

Include:
- A description of the vulnerability
- Steps to reproduce or a proof-of-concept test
- Your assessment of the severity and impact

We will acknowledge within 48 hours and aim to patch within 7 days for critical issues.

---

## Getting Help

Stuck on the codebase? Have a question before diving in?

- 💬 **GitHub Discussions** → [Ask a question](../../discussions/categories/q-a) — preferred for anything technical
- 🐛 **GitHub Issues** → For confirmed bugs only — include reproduction steps
- 🌊 **Stellar Wave Discord** → Join the Stellar developer community at [discord.gg/stellardev](https://discord.gg/stellardev) for real-time help

If you're new to Soroban, these resources will get you up to speed quickly:
- [Soroban Documentation](https://developers.stellar.org/docs/build/smart-contracts/overview)
- [Soroban by Example](https://soroban.stellar.org/docs/learn/getting-started)
- [Stellar Developers Discord](https://discord.gg/stellardev)

---

> We appreciate every contribution — from a one-line comment fix to a full feature implementation. TrustLink is open infrastructure for real people. Thank you for helping build it.
