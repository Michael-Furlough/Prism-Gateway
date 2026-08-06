# Prism Gateway — Roadmap

**Status:** Living document, updated at the start of each release cycle.
**Purpose:** Communicate where the project is, where it is going, and how we decide what to build next.

This roadmap is a planning tool, not a contract. Dates are intentionally omitted in favor of milestone sequencing — infrastructure software that handles security-sensitive traffic should ship when it is ready, not when a calendar says it should. Where we give estimates, they are ranges based on current maintainer capacity, and they will move if quality would otherwise be compromised.

---

## Table of Contents

1. [Vision](#1-vision)
2. [Current State](#2-current-state)
3. [Release Strategy](#3-release-strategy)
4. [Semantic Versioning Policy](#4-semantic-versioning-policy)
5. [Version Roadmap](#5-version-roadmap)
6. [Success Metrics](#6-success-metrics)
7. [Community Milestones](#7-community-milestones)
8. [Stretch Goals](#8-stretch-goals)
9. [Long-Term Vision](#9-long-term-vision)
10. [How This Roadmap Is Maintained](#10-how-this-roadmap-is-maintained)

---

## 1. Vision

Prism Gateway exists to make secure, format-agnostic API integration a solved problem rather than a recurring engineering cost. We want a team standing up a new service integration to spend their time on the business logic of that integration, not on writing another one-off serializer, another ad-hoc validation layer, or another bespoke rate limiter.

We are building toward a gateway that:

- Understands the *shape* of the data flowing through it, not just its route.
- Enforces security policy as a structural property of the pipeline, not an optional plugin.
- Gives operators genuine visibility into what the gateway is doing, without requiring a separate observability stack to get basic answers.
- Works fully offline, so that teams operating in regulated, air-gapped, or bandwidth-constrained environments are not second-class users.
- Treats AI as a tool that proposes and explains, never one that acts unilaterally on production traffic.

We are explicitly *not* trying to build a service mesh, a general-purpose ESB, or a low-code integration platform. Scope discipline is part of the roadmap, not just the feature list.

---

## 2. Current State

As of this writing, Prism Gateway is in **active pre-1.0 development**. The core pipeline architecture described in `ARCHITECTURE.md` is implemented and exercised by the integration test suite; the project is not yet recommended for production traffic.

**Implemented today:**

- Canonical Intermediate Representation and the parser/encoder trait framework
- JSON, YAML, TOML, MessagePack, and CBOR parsers and encoders
- Fixed-order request lifecycle (security → authn → authz → routing → parse → validate → transform → encode)
- Declarative transformation operation set (`rename_field`, `drop_field`, `cast`, `default_value`, `flatten`/`nest`, `map_array`, `conditional`)
- Security engine layers: transport limits, content-type policy, rate limiting, header validation
- API key, mTLS, and local-JWT authentication
- Route-attached authorization policy
- Schema Intelligence Engine (read-only, suggestion-queue based)
- WASM plugin runtime with fuel metering and capability-scoped host imports
- Structured logging, Prometheus metrics, and `tracing` span instrumentation across pipeline stages

**In progress:**

- XML parser (DTD/XXE-hardened) and Protocol Buffers parser (descriptor-driven)
- OAuth2/OIDC token introspection authenticator
- OpenTelemetry OTLP trace export
- Kubernetes manifests and Helm chart
- WASM Component Model migration for the plugin host interface

**Planned but not started:**

- Avro support (pending schema registry integration design)
- Distributed multi-node clustering with consensus-based policy sync
- Formal authorization policy language
- Plugin marketplace with signed distribution
- Local AI inference subsystem (design largely settled per `ARCHITECTURE.md` §16; implementation not yet begun)

This roadmap tracks the path from this state to a `v1.0` we are willing to recommend for production use, and beyond.

---

## 3. Release Strategy

### 3.1 Pre-1.0 Releases (`v0.x`)

Pre-1.0 releases are cut when a coherent, independently useful set of capabilities is ready — not on a fixed cadence. Each `v0.x` release is expected to be usable and documented for the capabilities it claims to support, even though the overall API surface is not yet stable. We do not ship known-broken features labeled as "implemented"; anything not ready is labeled `In Progress` or `Planned` in both this document and `ARCHITECTURE.md`.

### 3.2 Release Channels

| Channel | Purpose | Stability Expectation |
|---|---|---|
| `main` branch builds | Continuous integration target, nightly artifacts | May contain incomplete features behind flags; not for production |
| Tagged `v0.x.y` releases | Milestone snapshots | Stable for the features documented as `Implemented` in that release's changelog |
| `v1.0` and beyond | Production-track releases | Full semantic versioning guarantees (see below) |

### 3.3 Pre-1.0 Compatibility Expectations

Before `v1.0`, breaking changes to configuration format, plugin ABI, or the management API are permitted between minor versions but **must** be documented in the release's `CHANGELOG.md` under a `BREAKING` heading, with a migration note. We treat this as a hard requirement, not a courtesy — pre-1.0 does not mean undocumented churn.

---

## 4. Semantic Versioning Policy

Starting at `v1.0.0`, Prism Gateway follows [Semantic Versioning 2.0.0](https://semver.org/) applied across three distinct surfaces, versioned together but reasoned about separately:

| Surface | What a breaking change means |
|---|---|
| **Configuration format** | A previously valid `prism.toml` (or route/policy definition) no longer parses, or changes behavior without a config-level opt-in |
| **Plugin ABI** | A plugin built against a given `prism-plugin` interface version fails to load or behaves differently against a new gateway version |
| **Management/Dashboard API** | A previously valid API request against the management API returns a different shape or status code for equivalent input |

A change is considered breaking, and therefore requires a major version bump post-1.0, if it breaks *any* of the three surfaces above. Data-plane behavior changes that only affect newly-configured routes (e.g., a new transformation operation) are additive (minor version). Security fixes that necessarily change behavior (e.g., closing a validation bypass) are documented as an exception in `SECURITY.md`'s policy and may ship as a patch release even if technically behavior-changing, because a security fix that waits for the next major version is not a fix.

---

## 5. Version Roadmap

### v0.1 — Foundation *(current focus)*

**Theme:** Prove the core architecture end-to-end with a minimal but real format set.

- [x] CIR design and implementation
- [x] Parser/encoder framework and trait definitions
- [x] JSON, YAML, TOML, MessagePack, CBOR support
- [x] Fixed-order request lifecycle
- [x] Core security engine (transport limits, content-type policy, rate limiting)
- [x] API key and mTLS authentication
- [x] Route-attached authorization
- [x] Declarative transformation engine (initial operation set)
- [x] Structured logging and Prometheus metrics
- [ ] XML parser (XXE-hardened) — **in progress**
- [ ] Integration test suite covering full pipeline for all `v0.1`-scoped formats
- [ ] Reference deployment: single static binary + Docker image, documented

**Exit criteria for v0.1:** every format and security control listed above has a passing integration test, a fuzz harness (where applicable), and documentation in `docs/`. No component may be marked `Implemented` in `ARCHITECTURE.md` without meeting this bar.

### v0.2 — Visibility and Extensibility

**Theme:** Give operators a way to see what the gateway is doing, and give integrators a safe way to extend it.

- WASM plugin runtime (fuel metering, capability-scoped imports) — carried over from initial design work
- Plugin extension points: pre-authn, post-validation, pre/post-transform
- `tracing`-based span instrumentation across all pipeline stages
- OpenTelemetry OTLP export
- Protocol Buffers parser/encoder (descriptor-driven)
- Dashboard v1: read-only pipeline visualization, live request inspection
- Management API v1 (versioned, authenticated, separate listener from the data plane)

### v0.5 — Intelligence and Policy Maturity

**Theme:** Introduce advisory intelligence and mature the authorization model, without compromising determinism.

- Schema Intelligence Engine: field inference, enumeration detection, drift alerts
- Suggestion queue and dashboard Approve/Reject workflow
- OAuth2/OIDC token introspection authenticator
- SPIFFE/SPIRE workload identity (evaluation)
- Expanded authorization policy expressiveness (still declarative, not a general policy language)
- Live transformation preview in the dashboard, isolated from production upstream dispatch
- Embedded code editor for transformation pipelines and plugin manifests
- Hardened, fuzz-tested XML and Protocol Buffers parsers promoted from `In Progress` to `Implemented`
- Formal security audit of `v0.5` scope (see `SECURITY.md`) prior to promotion toward `v1.0` release candidates

### v1.0 — Production Readiness

**Theme:** A gateway we are willing to recommend for production traffic, with a stable, versioned contract.

- Semantic versioning guarantees begin (§4)
- All `v0.1`–`v0.5` scoped formats and features hardened, documented, and covered by the full testing architecture described in `ARCHITECTURE.md` §30
- Third-party security audit completed and findings resolved or explicitly accepted-and-documented
- Kubernetes manifests and Helm chart, with documented production topology guidance
- Documented upgrade path and configuration migration tooling for every prior `v0.x` release
- Performance baseline published (throughput/latency benchmarks across the supported format matrix, on documented reference hardware)
- Local AI subsystem, if ready, ships as an explicitly optional, off-by-default feature (`--features local-ai`); if not ready, it remains `Planned` for a `v1.x` release rather than delaying `v1.0`

**We will not ship `v1.0` with any core pipeline component still labeled `In Progress`.** This is the single hardest gate in this roadmap and takes priority over any date.

### v2.0 — Distributed and Ecosystem Maturity

**Theme:** Scale beyond a single node and mature the plugin ecosystem.

- Distributed multi-node clustering with consensus-based policy synchronization
- Plugin marketplace with signed, versioned distribution independent of core release cadence
- WASM Component Model fully adopted for the plugin host interface
- Avro support with schema registry integration
- Zero-copy transformation pipeline for high-throughput pass-through routes
- Managed/hosted deployment option evaluated (self-hosting remains the primary supported model regardless of outcome)
- Formal policy language for authorization, if community demand and design maturity justify the added complexity over the `v1.0` declarative model

---

## 6. Success Metrics

We track project health across four dimensions. These are the metrics maintainers actually look at when deciding whether the project is on track — not vanity metrics.

| Dimension | Metric | Why it matters |
|---|---|---|
| **Correctness** | Fuzz corpus coverage per parser; zero open critical/high security findings older than the response SLA in `SECURITY.md` | A gateway that mishandles input is worse than no gateway |
| **Performance** | p50/p99 latency and throughput per format, tracked in CI via `criterion` benchmarks, regression-gated | Determinism and safety must not come at an unbounded performance cost |
| **Adoption** | Non-maintainer-authored issues, PRs, and plugins; production deployments reported via voluntary community showcase | Signals whether the project is solving a real problem for people outside the core team |
| **Contributor health** | Time-to-first-response on issues/PRs; ratio of first-time to repeat contributors; number of active maintainers | A project with a single maintainer is a project with a single point of failure |

We deliberately do not track GitHub stars as a success metric. It is easy to game, poorly correlated with production usage, and encourages the wrong incentives for a security-critical project.

---

## 7. Community Milestones

Alongside the version roadmap, we track milestones that are about the *project* rather than the *software*:

- **First external security disclosure handled end-to-end** through the process in `SECURITY.md`, validating the process itself rather than just the policy document.
- **First non-maintainer-authored plugin** published and documented, validating the plugin SDK's real-world usability.
- **First production deployment case study** contributed by an adopter (opt-in, no pressure — see `GOVERNANCE.md` on community values).
- **First community-proposed RFC** accepted and implemented, validating that the RFC process in `CONTRIBUTING.md` works for people outside the founding maintainer group.
- **Formation of a second working group** (beyond the initial core/security working groups described in `GOVERNANCE.md`), signaling the project has grown beyond what a single group can effectively cover.

---

## 8. Stretch Goals

These are capabilities we find genuinely interesting but are not committed to, and are sequenced explicitly *after* `v2.0` scope in priority:

- **Format-aware diffing and changelog generation** for schema evolution — given two versions of an inferred schema, produce a human-readable summary of what changed and its likely compatibility impact.
- **Deterministic replay tooling** — capture a request (with consent/redaction policy applied) and replay it against a modified pipeline configuration offline, to let operators validate a configuration change against real traffic shapes before promoting it.
- **Native SDKs** for common languages that wrap the management API and plugin authoring workflow, reducing the friction of writing a first plugin.
- **Formal verification of the security engine's ordering guarantees**, likely via a model-checked specification of the pipeline stage ordering described in `ARCHITECTURE.md` §5, to move the "provable by inspection" claim to "provable by tooling."

Stretch goals are revisited at the start of each major version's planning cycle and may be promoted into the version roadmap, deferred further, or dropped, based on community input gathered through the RFC process.

---

## 9. Long-Term Vision

Five-plus years out, we want Prism Gateway to be the kind of project referenced the way Envoy or Kafka are referenced today — not because it is the biggest or most feature-complete option, but because it is the option people reach for when correctness and security actually matter more than time-to-first-demo.

Concretely, that means:

- A plugin ecosystem broad enough that most format and policy needs are covered without touching the core codebase.
- A track record of handled security disclosures that demonstrates the process in `SECURITY.md` is not theater.
- A governance structure (see `GOVERNANCE.md`) that has successfully transferred meaningful decision-making authority beyond the founding maintainer(s), because a project that cannot outlive its founder's availability is not sustainable infrastructure.
- A local AI assistance layer that has earned operator trust specifically *because* of its architectural inability to act unilaterally — we would rather be the gateway that people trust with AI assistance because it structurally cannot betray that trust, than one that is merely well-behaved by convention.

We are building for the long term. Features that would compromise the core principles in `ARCHITECTURE.md` §2 for short-term adoption gains are out of scope, permanently, regardless of how this roadmap otherwise evolves.

---

## 10. How This Roadmap Is Maintained

This document is updated:

- At the start of each `v0.x`/`v1.x` planning cycle, by the maintainers, with community input solicited via a pinned discussion issue.
- Whenever a milestone's exit criteria are met or substantively changed, to keep the "current state" section accurate rather than aspirational.
- Whenever an RFC (see `CONTRIBUTING.md`) materially changes planned scope for an upcoming version.

If you believe this roadmap is out of date relative to the actual state of the codebase, that is itself a bug — please open an issue. A roadmap that doesn't reflect reality is worse than no roadmap.
