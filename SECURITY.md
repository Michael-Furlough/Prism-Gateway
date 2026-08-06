# Security Policy

Prism Gateway is designed to sit on the trust boundary between untrusted clients and internal services. Its security posture is not a feature — it is the reason the project exists (see `ARCHITECTURE.md` §2.1). This document describes how we handle vulnerabilities, what our threat model is, and the engineering standards we hold the codebase to.

---

## Table of Contents

1. [Supported Versions](#1-supported-versions)
2. [Reporting a Vulnerability](#2-reporting-a-vulnerability)
3. [Responsible Disclosure](#3-responsible-disclosure)
4. [PGP](#4-pgp)
5. [Security Response Timeline](#5-security-response-timeline)
6. [Threat Model](#6-threat-model)
7. [Secure Coding Standards](#7-secure-coding-standards)
8. [Unsafe Rust Policy](#8-unsafe-rust-policy)
9. [Cryptography Policy](#9-cryptography-policy)
10. [OWASP and NIST Alignment](#10-owasp-and-nist-alignment)
11. [Dependency Management](#11-dependency-management)
12. [Supply Chain Security and SBOM](#12-supply-chain-security-and-sbom)
13. [Security Testing](#13-security-testing)
14. [Fuzz Testing](#14-fuzz-testing)
15. [Continuous Security Scanning](#15-continuous-security-scanning)

---

## 1. Supported Versions

Prism Gateway is currently in pre-1.0 development. Until `v1.0`, only the **most recent tagged release** receives security fixes; there is no long-term support branch yet, because maintaining backport branches for a moving pre-1.0 architecture would spread review capacity thin in exactly the areas that need it most concentrated.

| Version | Supported |
|---|---|
| Latest tagged `v0.x` release | ✅ |
| Prior `v0.x` releases | ❌ (upgrade to latest) |
| `main` branch (unreleased) | Best-effort; fixes land here first |

Starting at `v1.0.0`, this policy will expand to support the current major version and the immediately preceding major version for security fixes, for a minimum window published in `ROADMAP.md` at that time. This document will be updated when that policy takes effect.

---

## 2. Reporting a Vulnerability

**Do not open a public GitHub issue for a security vulnerability.** Public issues are appropriate for functional bugs; a public report of an exploitable vulnerability puts every current deployment at risk before a fix exists.

Report privately through one of:

1. **GitHub Security Advisories** — use the "Report a vulnerability" button under the repository's Security tab. This is the preferred channel; it creates a private advisory thread with maintainers and supports coordinated disclosure timing directly in GitHub.
2. **Email** — `security@prism-gateway.dev`, optionally encrypted with the PGP key in §4.

### What to Include

A useful report includes as much of the following as you have:

- Affected version/commit
- Affected component (crate name if known — see `ARCHITECTURE.md` §31 for the crate map)
- Description of the vulnerability and its impact
- Reproduction steps or proof-of-concept (a minimal one is more useful than an elaborate one)
- Any suggested remediation, if you have one

You do not need to have a fix, or even a full understanding of root cause, to report — a well-described symptom is enough to start an investigation. We would much rather receive a report with 60% confidence than have a real issue go unreported because a researcher wasn't fully certain.

---

## 3. Responsible Disclosure

We follow a coordinated disclosure model:

- We ask that reporters give us the response window described in §5 to investigate, develop, and release a fix before any public disclosure.
- We will keep the reporter informed of progress throughout — you should never be left wondering whether a report was received or is being worked on.
- We credit reporters (by name, handle, or organization, per their preference — including anonymously if requested) in the security advisory and release notes, unless the reporter asks not to be credited at all.
- If a reporter and the maintainers disagree on disclosure timing, we default to erring toward the reporter's preference for an earlier public disclosure once a fix is available, rather than indefinitely delaying — an unpatched, undisclosed vulnerability that a reporter already knows about is not meaningfully more "secure" than a disclosed one, once a fix exists.
- If we determine a report does not describe an exploitable vulnerability, we'll explain our reasoning and, if the reporter disagrees, are open to revisiting — security disagreements are handled as engineering discussions, not dismissals.

---

## 4. PGP

For reporters who want to encrypt vulnerability details before a private advisory thread is established (for example, when initiating first contact by email):

```
Key ID: (published in SECURITY.md on the main branch and on the project website)
Fingerprint: (published alongside the key)
```

> The actual key material and fingerprint are published and kept current in the repository's root `SECURITY.md` on `main` rather than duplicated here, so that a compromised or rotated key cannot go stale in copies of this document. Always verify against the version on `main` before relying on it.

---

## 5. Security Response Timeline

| Stage | Target |
|---|---|
| Acknowledgment of report received | Within 3 business days |
| Initial assessment (valid/invalid, severity estimate) | Within 7 business days |
| Regular status updates to reporter, if fix is non-trivial | At least every 14 days until resolution |
| Fix developed and privately validated | Timeline depends on severity — see below |
| Coordinated public disclosure + release | Promptly after fix validation, coordinated with reporter |

**Severity-based fix targets** (using a CVSS-like informal scale, formalized further as the project matures):

- **Critical** (remote, unauthenticated, no user interaction, e.g. a parser RCE or auth bypass): fix targeted within days, not weeks; may result in an out-of-band patch release.
- **High** (requires authentication or specific configuration, significant impact): fix targeted within the current release cycle.
- **Medium/Low**: fix targeted for the next regular release, documented in the advisory with any interim mitigation guidance.

These are targets, not guarantees — some vulnerabilities are genuinely harder to fix correctly than others, and we will not ship a rushed, incorrect fix just to hit a timeline. We will communicate honestly if a target is slipping and why.

---

## 6. Threat Model

This section summarizes the threat model; the full architectural trust-boundary discussion is in `ARCHITECTURE.md` §35.

### 6.1 In Scope

- **Untrusted network clients** sending arbitrary bytes to any listener, including malformed, oversized, deeply nested, or adversarially crafted payloads targeting parser vulnerabilities (memory exhaustion, algorithmic complexity attacks, XXE, deserialization gadget-style issues).
- **Authenticated-but-malicious clients** attempting to exceed their authorized scope (authorization bypass, privilege escalation via crafted requests).
- **Malicious or compromised plugins**, bounded by the WASM sandbox described in `ARCHITECTURE.md` §17–18 — we treat the sandbox boundary itself as a security-critical component subject to the same standards as the parser boundary.
- **Supply-chain compromise** of a direct or transitive dependency (see §11–12).
- **Credential and secret exposure** via logging, error messages, or telemetry (see `ARCHITECTURE.md` §26).
- **Timing side channels** in authentication comparisons (see `ARCHITECTURE.md` §13).
- **Configuration-level misconfiguration risk** — where a plausible but incorrect configuration would silently produce an insecure state; we treat "insecure by confusing default" as a class of vulnerability, not just "insecure by bug."

### 6.2 Explicitly Out of Scope

- **Physical access to the host** running Prism — host-level security (disk encryption, physical access control) is the deployer's responsibility.
- **Compromise of the underlying OS or container runtime** — Prism assumes a reasonably trustworthy execution environment; kernel-level sandboxing is complementary to, not a substitute for, Prism's own boundaries.
- **Denial of service via sheer volume** exceeding what rate limiting and infrastructure-level DDoS protection are designed to absorb — Prism's rate limiter (`ARCHITECTURE.md` §12.1) protects against abusive per-identity behavior, not a volumetric attack that should be handled upstream (CDN, load balancer, cloud provider DDoS protection).
- **Vulnerabilities in upstream services** Prism forwards requests to — Prism's job is to not make things worse (e.g., not forwarding a payload it should have rejected), not to compensate for an insecure upstream.
- **Local AI subsystem model weights' training-data provenance** — this is tracked as a supply-chain and licensing question, not a security vulnerability class, though we take it seriously in `ARCHITECTURE.md` §16's design constraints.

### 6.3 Key Assumptions

- TLS termination, when used, relies on `rustls` and correctly configured certificates — misconfigured TLS (e.g., disabled verification) is a deployment error we will document prominently but cannot prevent architecturally.
- The security engine's ordering guarantee (screening before parsing, before routing) is the primary structural defense — see `ARCHITECTURE.md` §5.1 and §12. A vulnerability that violates this ordering is treated as critical regardless of its individual exploitability, because it undermines the architectural claim the rest of the threat model depends on.

---

## 7. Secure Coding Standards

- **No `unwrap()`/`expect()` on network-derived data**, enforced via `clippy::unwrap_used`/`clippy::expect_used` denial in security-relevant crates — see `CONTRIBUTING.md` §6.
- **All parser limits are enforced during parsing, not after** — see `ARCHITECTURE.md` §8. A parser that allocates before checking limits is a defect, not a style issue.
- **Credential and secret fields use the `Redact` wrapper type** at the type-system level, so logging a credential accidentally is a compile-time-visible pattern rather than a silent runtime leak — see `ARCHITECTURE.md` §26.
- **Constant-time comparison for any secret-equality check** (API keys, HMAC verification), using `subtle`-style constant-time primitives rather than `==`.
- **Fixed minimum latency for authentication failure paths** to reduce timing side-channel value — see `ARCHITECTURE.md` §13.
- **Input validation happens before trust is extended**, never the reverse — this is the entire premise of the request lifecycle ordering in `ARCHITECTURE.md` §5.
- **No dynamic code evaluation in the core transformation engine** — see `ARCHITECTURE.md` §10.2's rationale for rejecting an embedded scripting language in favor of a declarative operation set plus sandboxed plugins.

---

## 8. Unsafe Rust Policy

`unsafe_code` is denied by default at the workspace level (`CONTRIBUTING.md` §6). Exceptions require:

1. **A documented, narrow justification** in the PR description explaining why safe Rust cannot achieve the required behavior or performance characteristic.
2. **A `// SAFETY:` comment immediately preceding every `unsafe` block**, explaining the specific invariants being relied upon and why they hold.
3. **Two maintainer approvals**, at least one from a maintainer with security-team standing (see `GOVERNANCE.md`).
4. **A corresponding test** that would fail if the relied-upon invariant were violated, where feasible.

As of this writing, no `unsafe` block exists in `prism-core`, `prism-security`, `prism-authn`, or `prism-authz`. Where `unsafe` is required elsewhere (primarily to interoperate with zero-copy parsing internals of vetted third-party crates), it is tracked in an `UNSAFE.md` inventory file listing every occurrence, its justification, and its last security review date. This inventory is reviewed as part of every security audit.

We prefer to eliminate the need for `unsafe` over auditing it more carefully — if a safe alternative exists at acceptable performance cost, it is preferred, and we periodically revisit existing `unsafe` blocks to see if upstream crates or newer Rust features have closed the gap.

---

## 9. Cryptography Policy

- **We do not implement our own cryptographic primitives.** All cryptography goes through well-reviewed crates (`rustls` for TLS, `ring`/`RustCrypto`-family crates for hashing, HMAC, and signature verification, depending on the specific need).
- **`rustls` over OpenSSL bindings**, keeping TLS handling within memory-safe Rust rather than linking against a C TLS implementation — see `ARCHITECTURE.md` §32.
- **No custom key derivation, no custom random number generation** — we use the operating system's CSPRNG via the standard `getrandom`/`rand` ecosystem, never a hand-rolled PRNG, even for non-cryptographic-seeming purposes like generating request correlation IDs, since we've seen "not really security-sensitive" randomness misused as security-sensitive randomness elsewhere in the industry.
- **Secrets at rest** (API keys, if stored by Prism rather than an external secret store) are hashed, never stored in plaintext, using a modern password-hashing-style KDF where appropriate rather than a fast general-purpose hash.
- **Algorithm agility is limited deliberately** — we support a small, current, well-regarded set of algorithms rather than maximal configurability, since a large algorithm menu is itself a historical source of downgrade-attack vulnerabilities in other systems.

---

## 10. OWASP and NIST Alignment

Prism's security engine (`ARCHITECTURE.md` §12) is designed with explicit reference to:

- **OWASP API Security Top 10** — the security engine's layered checks map directly onto categories including broken object-level authorization (addressed by route-attached authorization, §14 of `ARCHITECTURE.md`), excessive data exposure (addressed by the `drop_field` transformation operation and schema-driven response validation), and lack of resources/rate limiting (addressed by the sharded rate limiter).
- **OWASP ASVS** (Application Security Verification Standard) — used as an internal review checklist for authentication and session-handling-adjacent code, even though Prism's "sessions" are typically short-lived, per-request authentication contexts rather than traditional web sessions.
- **NIST SP 800-53 / SP 800-63** — referenced for authentication assurance level concepts (particularly relevant to the JWT and forthcoming OIDC authenticators) and for general secure-development lifecycle practices reflected in this document's testing and review requirements.

We do not currently claim formal certification against any of these frameworks — alignment here means "designed with reference to and reviewed against," not "certified compliant." Organizations requiring formal compliance attestation should treat this section as a starting point for their own assessment, not a substitute for it.

---

## 11. Dependency Management

- Every new dependency requires justification in its introducing PR, per `CONTRIBUTING.md` §16.
- `cargo deny check` runs in CI on every PR, enforcing:
  - No dependencies with known advisories (via the [RustSec Advisory Database](https://rustsec.org/))
  - License compatibility (MIT/Apache-2.0/BSD-family only in the core binary; copyleft licenses are rejected)
  - No duplicate major-version dependency graphs where avoidable, to limit total dependency surface
  - No dependencies from unregistered/unusual sources — crates.io only for release builds, no git dependencies in the published crate graph
- Dependency updates are reviewed, not auto-merged blindly — a version bump PR still gets a human look at the changelog/diff for anything security- or behavior-relevant, particularly for crates in the parsing and cryptography paths.

---

## 12. Supply Chain Security and SBOM

- **Software Bill of Materials**: generated per release using `cargo-cyclonedx` (or equivalent), published alongside release artifacts in CycloneDX format, covering the full transitive dependency graph.
- **Reproducible builds**: `In Progress` — we are working toward build reproducibility for release binaries so that a published SBOM and published binary can be independently verified to correspond to each other.
- **Signed releases**: release artifacts are signed; signature verification instructions are published with each release.
- **Dependency pinning**: `Cargo.lock` is committed and used for all release builds, so the exact dependency graph of any given release is reproducible and auditable independent of what `crates.io` serves at build time in the future.
- **Provenance**: CI build artifacts are built from a clean, isolated CI environment with no access to maintainer credentials beyond what's needed for the specific pipeline step, minimizing the blast radius of a compromised CI dependency.

| Capability | Status |
|---|---|
| `cargo deny` in CI | Implemented |
| SBOM generation per release | Implemented |
| Signed release artifacts | Implemented |
| Reproducible builds | In Progress |
| Full build provenance attestation (SLSA-style) | Planned |

---

## 13. Security Testing

- **Security regression suite**: `tests/security_regressions/` — every confirmed vulnerability, once fixed, gets a permanent test that fails against the pre-fix code and passes against the fix. This suite runs on every PR, not just periodically.
- **Manual security review**: PRs touching `prism-security`, `prism-authn`, `prism-authz`, or any parser require two maintainer approvals, at least one from security-team standing (`GOVERNANCE.md`), with explicit threat-model consideration documented in the PR discussion.
- **Periodic third-party audit**: planned ahead of `v1.0` (see `ROADMAP.md`), and periodically thereafter, scoped to the areas of highest risk identified in §6.1.
- **Bug bounty**: not currently operated, pending project maturity; tracked as a `Planned` item alongside the `v1.0` audit in `ROADMAP.md`. We will announce clearly if and when this changes.

---

## 14. Fuzz Testing

Every parser implementation (`ARCHITECTURE.md` §8) has a corresponding `cargo-fuzz` harness under `fuzz/fuzz_targets/`:

```
fuzz/
├── fuzz_targets/
│   ├── parse_json.rs
│   ├── parse_yaml.rs
│   ├── parse_toml.rs
│   ├── parse_msgpack.rs
│   ├── parse_cbor.rs
│   ├── parse_xml.rs           # in progress alongside the parser itself
│   └── parse_protobuf.rs      # in progress alongside the parser itself
└── corpus/                     # seed corpora, checked in per-target
```

- **CI-triggered fuzzing**: a short fuzz run (bounded time budget) executes on every PR touching a parser, as a smoke check.
- **Scheduled extended fuzzing**: longer, corpus-accumulating fuzz sessions run on a recurring schedule independent of PR activity, with any crash automatically filed as a high-priority internal issue (not public, until triaged per §2–3, since a crash may indicate an exploitable vulnerability).
- **Corpus persistence**: crash-inducing and coverage-expanding inputs are retained and checked into the seed corpus, so fuzzing effort compounds over time rather than restarting from scratch each run.
- **New parsers are not merged as `Implemented`** in `ARCHITECTURE.md`'s status tables until they have a fuzz harness passing a minimum sustained fuzzing duration with no unresolved crashes — this is a hard gate, not a suggestion, consistent with `ROADMAP.md`'s v0.1 exit criteria.

---

## 15. Continuous Security Scanning

| Tool/Check | Frequency | Purpose |
|---|---|---|
| `cargo deny` | Every PR | License, advisory, and dependency source checks |
| `cargo audit` | Every PR + daily scheduled | RustSec advisory database checks against the current lockfile |
| `clippy` security-relevant lints | Every PR | `unwrap_used`, `expect_used`, `unsafe_code` gating, and other static analysis |
| CI fuzz smoke run | Every PR touching parsers | Fast regression check against the existing corpus |
| Scheduled extended fuzzing | Recurring (see §14) | Deep coverage over time |
| Dependency update monitoring | Automated PRs (Dependabot-equivalent) | Keeps the dependency graph current, reviewed per §11 |
| SBOM diff review | Every release | Confirms no unexpected dependency graph changes shipped |

If a scheduled scan surfaces a new advisory affecting a dependency already in use, it is triaged with the same severity framework as §5, even though the report originated internally rather than from an external researcher.

---

*Questions about this policy that are not themselves vulnerability reports are welcome as a GitHub Discussion. If you're ever unsure whether something is sensitive enough to report privately, err toward the private channel in §2 — we would rather triage a false alarm than miss a real report.*
