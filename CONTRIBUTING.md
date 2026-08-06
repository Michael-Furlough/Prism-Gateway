# Contributing to Prism Gateway

Thank you for considering a contribution to Prism Gateway. This document explains how the project works, what we expect from contributions, and how to get from "I want to help" to a merged pull request with as little friction as possible.

Read this document before your first PR. Most review delays come from PRs that skip a step described here, not from disagreements about the code itself.

---

## Table of Contents

1. [Project Philosophy](#1-project-philosophy)
2. [Ways to Contribute](#2-ways-to-contribute)
3. [Getting Started](#3-getting-started)
4. [Development Environment](#4-development-environment)
5. [Running Tests](#5-running-tests)
6. [Formatting and Linting](#6-formatting-and-linting)
7. [Coding Style](#7-coding-style)
8. [Documentation Standards](#8-documentation-standards)
9. [Commit Message Conventions](#9-commit-message-conventions)
10. [Branch Strategy](#10-branch-strategy)
11. [Pull Request Workflow](#11-pull-request-workflow)
12. [Issue Reporting](#12-issue-reporting)
13. [RFC Process](#13-rfc-process)
14. [Review Expectations](#14-review-expectations)
15. [Performance Expectations](#15-performance-expectations)
16. [Security Expectations](#16-security-expectations)
17. [Testing Requirements](#17-testing-requirements)
18. [Community Values](#18-community-values)

---

## 1. Project Philosophy

Prism Gateway is security-sensitive infrastructure software. That shapes how we accept contributions in ways that differ from a typical application project:

- **Correctness and clarity beat cleverness.** A PR that is 20% slower but obviously correct will usually beat a PR that is faster but requires the reviewer to trust a subtle invariant.
- **Every change is reviewed as if it will run in production handling untrusted traffic**, because it will. We do not have a "just a prototype" exception to review standards for code that lands on `main`.
- **We explain *why*, not just *what*.** PR descriptions, commit messages, and code comments should give a reviewer (or a future contributor) enough context to understand the reasoning, not just the diff.
- **Determinism is sacred.** Any change that makes the gateway's behavior depend on something other than its explicit input and configuration (hidden caching, time-of-day behavior, non-deterministic ordering) needs a very strong justification and maintainer sign-off before it is even worth writing a full PR for.

If you're unsure whether an idea fits the project before investing significant time, open a discussion issue first. We would much rather talk through an approach for free than review a large PR that turns out to be off-direction.

---

## 2. Ways to Contribute

Code is only one form of contribution we value. Equally valuable:

| Contribution type | Where it goes |
|---|---|
| Bug reports | GitHub Issues, using the bug report template |
| Security vulnerability reports | **Not** GitHub Issues — see `SECURITY.md` |
| Feature proposals | GitHub Discussions first, then an RFC if it gains traction (§13) |
| Documentation improvements | PRs directly, no RFC needed for clarity/accuracy fixes |
| New format parsers/encoders | RFC required (touches the trusted parsing surface — see §16) |
| New transformation operations | RFC required for core operations; plugins for anything niche |
| Plugin examples | PRs to `examples/plugins/`, no RFC needed |
| Performance improvements | PR with before/after `criterion` benchmark data |
| Fuzz harness improvements | PRs directly, always welcome, never need an RFC |
| Triage help on issues | Just start doing it — no permission needed |

---

## 3. Getting Started

### 3.1 Installing Rust

Prism Gateway targets the Rust version pinned in `rust-toolchain.toml` at the repository root. Install via [rustup](https://rustup.rs/):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Once installed, `rustup` will automatically pick up the pinned toolchain when you run any Cargo command inside the repository — you do not need to manually switch toolchains.

Verify your setup:

```bash
rustc --version
cargo --version
```

### 3.2 Repository Setup

```bash
git clone https://github.com/prism-gateway/prism-gateway.git
cd prism-gateway
git remote add upstream https://github.com/prism-gateway/prism-gateway.git   # if you forked
cargo build --workspace
```

The workspace is organized as described in `ARCHITECTURE.md` §31. Familiarize yourself with the crate boundaries before making a change that spans multiple crates — a change that needs to cross the `prism-schema-intel` → `prism-policy-store` boundary, for example, is almost certainly wrong (see `ARCHITECTURE.md` §15) and should be raised as a discussion before being written.

### 3.3 Recommended First Contributions

If this is your first PR, look for issues labeled `good-first-issue`. These are scoped to a single crate, have a clear acceptance criterion, and do not require an RFC. Avoid starting with a new parser or a security-engine change as a first contribution — not because you're not capable, but because the review bar for those areas is deliberately high, and it's a frustrating place to learn our conventions for the first time.

---

## 4. Development Environment

### 4.1 Required Tooling

```bash
rustup component add rustfmt clippy
cargo install cargo-nextest    # test runner, faster and clearer output than `cargo test`
cargo install cargo-deny       # license/advisory/dependency checks
cargo install cargo-fuzz       # required only if working on parsers
```

### 4.2 Editor Setup

We don't mandate an editor, but `rust-analyzer` is the reference LSP implementation the maintainers use, and CI's clippy configuration is exposed via `.clippy.toml` and workspace lint tables in `Cargo.toml` so your editor's diagnostics should match CI's, if `rust-analyzer` is configured to use the workspace's `clippy` target.

### 4.3 Workspace Feature Flags

Several crates are gated behind Cargo features to keep the default build minimal:

```bash
cargo build --workspace --all-features        # everything, including local-ai
cargo build --workspace --features local-ai    # opt into the optional AI subsystem
cargo build -p prism-parsers --features xml    # single-crate, single-feature builds for focused work
```

When contributing to an optional feature, make sure your change also builds cleanly *without* the feature enabled — CI checks both.

---

## 5. Running Tests

```bash
cargo nextest run --workspace                    # unit + integration tests
cargo nextest run --workspace --all-features
cargo test --doc --workspace                      # doctests (nextest does not run these yet)
```

### 5.1 Fuzz Testing

Fuzz targets live under `fuzz/fuzz_targets/`. If you're modifying a parser:

```bash
cd fuzz
cargo fuzz run parse_json -- -max_total_time=60    # quick local smoke run
```

CI runs a longer fuzz session against every parser on a schedule, independent of PR checks — see `SECURITY.md` §"Fuzz Testing" for the full policy. A PR that changes parsing logic should include a note in the description confirming a local fuzz smoke run was performed, and should not need to modify existing fuzz targets unless it changes the parser's public interface.

### 5.2 Integration Tests

Full-pipeline integration tests live under `tests/integration/` and spin up a real `prism-core` instance against a mock upstream (`tests/support/mock_upstream.rs`). These are slower than unit tests; run them explicitly when your change touches pipeline ordering, routing, or cross-crate behavior:

```bash
cargo nextest run --workspace -- integration::
```

### 5.3 Benchmarks

```bash
cargo bench -p prism-parsers
cargo bench -p prism-transform
```

Benchmark results are not required for every PR, only for PRs claiming a performance improvement or touching a hot path identified in `ARCHITECTURE.md` §22–24 (see §15 below).

---

## 6. Formatting and Linting

```bash
cargo fmt --all                                   # format
cargo fmt --all -- --check                        # CI check, non-mutating
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

CI runs both `rustfmt --check` and `clippy -D warnings`. Neither is optional, and neither has a "just this once" override — if `clippy` flags something you believe is a false positive, either restructure the code or add a narrowly-scoped `#[allow(...)]` with a comment explaining why, rather than disabling the lint workspace-wide.

Notable workspace-level lint configuration contributors should be aware of:

- `clippy::unwrap_used` and `clippy::expect_used` are **denied** in `prism-core`, `prism-security`, `prism-authn`, `prism-authz`, `prism-parsers`, and `prism-encoders` — any value derived from network input must be handled via `Result`, never unwrapped. See `ARCHITECTURE.md` §25.
- `unsafe_code` is **denied** at the workspace level by default; crates that need a narrow, audited exception opt back in per-module with `#[allow(unsafe_code)]` and an accompanying safety comment, per the unsafe Rust policy in `SECURITY.md`.

---

## 7. Coding Style

Beyond what `rustfmt`/`clippy` enforce mechanically:

- **Trait-first design at crate boundaries.** Public crate interfaces should be expressed as traits (see `FormatParser`, `SecurityCheck`, `Authenticator` in `ARCHITECTURE.md`) so alternative implementations — including test doubles — don't require modifying the consuming crate.
- **No stringly-typed state.** Use enums for anything with a closed set of variants (formats, decision outcomes, error kinds). If you find yourself matching on a `&str`, that's a signal to introduce a type.
- **Error types are specific, not `anyhow`-everywhere.** Library crates (anything under `crates/`) should use `thiserror`-defined, structured error enums so callers can match on failure kind. `anyhow` is acceptable in `prism-cli` and test code, where the consumer is a human reading a message, not another crate making a decision.
- **Prefer explicit over implicit conversions.** `From`/`TryFrom` implementations are welcome; blanket, surprising `Into` chains that obscure a lossy conversion are not — this matters especially in `prism-cir` and `prism-encoders`, given the lossy-conversion handling described in `ARCHITECTURE.md` §7.3.
- **Keep the hot path allocation-conscious**, but don't pre-optimize outside of `prism-parsers`, `prism-encoders`, `prism-transform`, and `prism-routing`. Elsewhere, clarity wins by default.

---

## 8. Documentation Standards

- Every public item (`pub fn`, `pub struct`, `pub trait`, `pub enum`) requires a doc comment explaining *purpose*, not just restating the signature. `cargo doc --workspace` should produce output a new contributor could navigate without reading the source.
- Doc comments for anything implementing a core trait (`FormatParser`, `SecurityCheck`, etc.) should note any deviation from the trait's documented contract, if any exists (it usually shouldn't).
- User-facing behavior changes require a corresponding update to `docs/` in the same PR — not a follow-up. A feature without documentation is not considered done for review purposes.
- Architectural changes — anything altering pipeline ordering, trust boundaries, or a subsystem's guarantees — require an update to `ARCHITECTURE.md` in the same PR, and likely require an RFC first (§13).

---

## 9. Commit Message Conventions

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

<optional body — explain why, not just what>

<optional footer — e.g. "Closes #123", "BREAKING CHANGE: ...">
```

**Types:** `feat`, `fix`, `docs`, `test`, `perf`, `refactor`, `chore`, `security`

**Scope** is typically the crate name without the `prism-` prefix (e.g. `parsers`, `security`, `routing`).

Examples:

```
feat(parsers): add streaming depth limit to XML parser

Enforces ParserLimits.max_depth during parse rather than after,
closing a resource-exhaustion vector for deeply nested XML.

security(authn): use constant-time comparison for API key verification

Previous implementation used slice equality, which is not
constant-time and introduced a timing side channel.
```

We squash-merge PRs, and the squashed commit message is drawn from the PR title/description — so a well-formed PR title in Conventional Commits style matters more than perfecting every intermediate commit on your branch.

---

## 10. Branch Strategy

- `main` is always releasable in the sense that it passes CI and does not knowingly contain a security regression — it is not required to be feature-complete for any given roadmap milestone.
- Feature work happens on branches named `<type>/<short-description>` (e.g. `feat/xml-parser-limits`, `fix/rate-limiter-shard-race`).
- We do not use long-lived release branches for pre-1.0 development; releases are tagged directly from `main`. This will change at `v1.0` to support backporting security fixes to prior major versions — see `ROADMAP.md` and `SECURITY.md` for the supported-versions policy that will take effect then.
- Rebase your branch on `main` before requesting review if it has drifted significantly; we prefer a clean, linear history for review purposes, but we won't block a PR over this alone if the diff is otherwise easy to review.

---

## 11. Pull Request Workflow

1. **Open an issue or discussion first** for anything beyond a small fix — this avoids wasted work on PRs that don't fit project direction.
2. **Fork and branch**, following the naming convention in §10.
3. **Write the change, with tests**, per §17.
4. **Run the full local check suite** before opening the PR:
   ```bash
   cargo fmt --all -- --check
   cargo clippy --workspace --all-targets --all-features -- -D warnings
   cargo nextest run --workspace --all-features
   ```
5. **Open the PR** with:
   - A description of *what* and *why*, not just a restatement of the diff.
   - A note on which `ARCHITECTURE.md`/`docs/` sections were updated, if any.
   - For performance-sensitive changes, benchmark output (§15).
   - For anything touching parsing, a note confirming a local fuzz smoke run (§5.1).
6. **CI runs automatically**: format check, clippy, full test suite, `cargo-deny`, and (for parser changes) an extended fuzz run.
7. **Review** proceeds per §14. Expect at least one round of feedback on anything non-trivial — this is normal, not a sign something is wrong with your PR.
8. **Merge**: squash-merged by a maintainer once approved and green. Contributors do not self-merge, even with approval, except for maintainers merging their own documentation-only fixes.

---

## 12. Issue Reporting

**Security vulnerabilities do not go here — see `SECURITY.md`.**

For everything else:

- **Bug reports** should include: Prism version/commit, configuration relevant to the bug (redacted of secrets), expected vs. actual behavior, and reproduction steps. A bug report without reproduction steps will usually get a "can you provide a minimal repro?" response before triage can proceed.
- **Feature requests** should describe the problem being solved, not just the desired implementation — see §13 for when this needs to become a full RFC.
- **Questions** are welcome as GitHub Discussions rather than Issues, so they're easier to search and don't clutter the issue tracker used for actionable work.

We label issues by area (matching crate names), by kind (`bug`, `feature`, `docs`, `question`), and by status (`good-first-issue`, `help-wanted`, `blocked`, `needs-rfc`).

---

## 13. RFC Process

Some changes are too significant to design in a PR review thread. An RFC is required for:

- New format parsers/encoders (they expand the trusted parsing surface described in `ARCHITECTURE.md` §35)
- New core transformation operations (as opposed to plugin-based extensions)
- Any change to pipeline stage ordering
- Any change to the plugin ABI or capability model
- Any change affecting the advisory-only guarantees of the Schema Intelligence Engine or local AI subsystem
- Authorization/authentication model changes
- Anything the maintainers explicitly label `needs-rfc` on an issue

### 13.1 Process

1. Copy `rfcs/0000-template.md` to `rfcs/0000-your-proposal.md` (the number is assigned at merge time).
2. Fill in: motivation, detailed design, drawbacks, alternatives considered, and unresolved questions.
3. Open a PR with the RFC file. This PR is for discussing the *design*, not implementation — code should not be part of this PR.
4. The RFC is discussed publicly. Maintainers aim to reach a decision (accept, reject, or request revision) within the timeframe published in `GOVERNANCE.md`.
5. Once accepted, the RFC is merged, gets a permanent number, and implementation PRs reference it (`Implements rfcs/0007-xml-parser.md`).

RFCs are not bureaucracy for its own sake — they exist because the areas listed above are exactly the areas where a change that seems reasonable in isolation can quietly undermine one of the guarantees in `ARCHITECTURE.md` §2. Design review before implementation is cheaper than review-driven redesign after the fact.

---

## 14. Review Expectations

- **Every PR needs at least one maintainer approval** before merge; PRs touching `prism-security`, `prism-authn`, `prism-authz`, or any parser need two.
- Reviewers focus on: correctness against the trait/module contract, adherence to the principles in `ARCHITECTURE.md` §2–3, test coverage per §17, and whether the change is the smallest reasonable diff that solves the stated problem.
- We review code, not people. Feedback is about the change, phrased constructively — see `CODE_OF_CONDUCT.md`. If a reviewer's comment reads harshly, that's worth flagging to a maintainer, not absorbing.
- If a PR sits without response for more than a week, it's fair to leave a polite ping. Maintainers are volunteers with day jobs (see `GOVERNANCE.md`), and things do fall through the cracks.

---

## 15. Performance Expectations

Correctness comes first, but Prism is infrastructure software sitting on a request hot path, so performance is a real requirement, not an afterthought:

- Changes to `prism-parsers`, `prism-encoders`, `prism-transform`, or `prism-routing` that could plausibly affect throughput must include `cargo bench` before/after numbers in the PR description.
- A performance regression greater than 5% on an existing `criterion` benchmark requires either a justification (e.g., a necessary security fix that has an inherent cost) or a follow-up plan to recover the regression.
- New hot-path allocations should be justified in the PR description — see `ARCHITECTURE.md` §24 for the existing allocation strategy (buffer pooling, pre-sized collections).
- We do not accept performance optimizations that introduce `unsafe` code without a corresponding SECURITY.md-compliant justification, safety comment, and — for anything non-trivial — a maintainer discussion before the PR is written.

---

## 16. Security Expectations

- **Do not open a public PR or issue for a security vulnerability.** Follow the process in `SECURITY.md`.
- Any code touching the parser boundary, security engine, authentication, or authorization is held to the highest review bar in the project (two maintainer approvals, mandatory fuzzing where applicable, explicit threat-model consideration in the PR description).
- New dependencies go through the check described in `ARCHITECTURE.md` §32 and `SECURITY.md`'s supply-chain section — `cargo deny check` must pass, and the PR description should state why the dependency is needed and why an existing dependency couldn't cover it.
- If, while working on something unrelated, you notice what might be a security issue in existing code, stop and report it per `SECURITY.md` rather than fixing it inline in an unrelated PR — this keeps the disclosure and fix-verification process intact.

---

## 17. Testing Requirements

Minimum bar for any non-trivial PR, mapped to the testing architecture in `ARCHITECTURE.md` §30:

| Change type | Required tests |
|---|---|
| New pure function / small logic change | Unit test(s) covering normal + edge cases |
| New parser or encoder | Unit tests, round-trip property test (`proptest`), fuzz harness, integration test through the full pipeline |
| New transformation operation | Unit tests, integration test exercising it via a route config |
| Security engine change | Unit tests, a regression test added to `tests/security_regressions/` if fixing a specific issue |
| New authenticator/authorization behavior | Unit tests, integration test covering both allow and deny paths |
| Bug fix | A regression test that fails on `main` before the fix and passes after |
| Documentation-only change | No test required, but examples in docs should be verified to actually work |

PRs without adequate test coverage will be asked to add it before review proceeds — this isn't a formality, it's how we keep the guarantees in `ARCHITECTURE.md` actually true over time rather than aspirational.

---

## 18. Community Values

- **We assume good faith.** Most disagreements in review are about tradeoffs, not about someone being wrong on purpose.
- **We value clear communication over impressive-sounding communication.** A PR description that plainly states "I wasn't sure about X, here's what I chose and why" is more useful to a reviewer than one that projects false confidence.
- **We credit contributions accurately.** Co-authored commits, `Suggested-by:` trailers, and changelog credit are used deliberately so that contributions — including non-code ones like triage, RFC discussion, and documentation review — are visible.
- **We don't gatekeep based on experience level.** A well-reasoned, well-tested first PR from someone new to Rust gets the same consideration as one from a long-time contributor. Review feedback should teach, not just reject.
- **We take the project's security-critical nature seriously without being precious about it.** High review standards exist to protect users, not to make contributing feel unwelcoming. If our process ever feels like the latter, please tell us — via an issue, a discussion, or directly to a maintainer.

Full behavioral expectations are in `CODE_OF_CONDUCT.md`, which applies to every space associated with this project: issues, PRs, discussions, and any official community channels.

---

Thank you again for reading this far, and for contributing. Infrastructure software is a long game, and every well-reviewed PR, every clear bug report, and every thoughtful RFC comment is what makes Prism Gateway something people can actually trust in production over the long term.
