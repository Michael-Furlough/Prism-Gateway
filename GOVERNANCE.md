# Prism Gateway Governance

This document describes how Prism Gateway is led, how decisions are made, and how contributors move into positions of greater responsibility. It exists so that "how does this project actually work" has a single, stable answer that doesn't depend on who you happen to ask.

Governance evolves as the project grows — this document is expected to change, particularly around the maintainer/voting sections, as the contributor base expands beyond its founding group. Changes to governance itself follow the same RFC process described in `CONTRIBUTING.md` and are called out explicitly in §5 below.

---

## Table of Contents

1. [Project Leadership](#1-project-leadership)
2. [Roles](#2-roles)
3. [Decision-Making Model](#3-decision-making-model)
4. [RFC Process Ownership](#4-rfc-process-ownership)
5. [Changing This Document](#5-changing-this-document)
6. [Voting](#6-voting)
7. [Release Process](#7-release-process)
8. [Conflict Resolution](#8-conflict-resolution)
9. [Becoming a Maintainer](#9-becoming-a-maintainer)
10. [Security Team](#10-security-team)
11. [Working Groups](#11-working-groups)
12. [Community Meetings](#12-community-meetings)
13. [Long-Term Sustainability](#13-long-term-sustainability)

---

## 1. Project Leadership

Prism Gateway is currently led by its founding maintainer group. There is no single "benevolent dictator" role by design — decisions of consequence go through the consensus and, where necessary, voting processes described below, even in the project's early stage, because we want the *habit* of shared decision-making established before it becomes strictly necessary, not retrofitted after governance disputes have already happened.

Project leadership is responsible for:

- Setting overall technical direction, in coordination with the roadmap process (`ROADMAP.md`)
- Maintaining the integrity of the security response process (`SECURITY.md`)
- Approving new maintainers and working group leads
- Serving as the final decision-making body when consensus cannot be reached (§3, §6)
- Stewarding project resources (repository, domain, funding if/when applicable) in the community's interest, not any single party's

---

## 2. Roles

| Role | Responsibilities | How you get there |
|---|---|---|
| **Contributor** | Anyone who has had a PR, documentation fix, triage contribution, or RFC discussion accepted | Submit a contribution |
| **Reviewer** | Trusted to review PRs in a specific area; review approval counts toward merge requirements for non-security-critical crates | Sustained, high-quality contribution in an area; nominated by a maintainer (§9) |
| **Maintainer** | Merge access, RFC decision authority, release authority | Sustained reviewer contribution + maintainer vote (§9) |
| **Security Team Member** | Handles private vulnerability reports, holds authority on `prism-security`/`prism-authn`/`prism-authz`/parser review | Maintainer, additionally vetted for security-response responsibility (§10) |
| **Working Group Lead** | Coordinates a specific working group's scope | Established by maintainer consensus when a working group forms (§11) |

Roles are additive, not exclusive — a maintainer can also be a working group lead and a security team member.

---

## 3. Decision-Making Model

Prism Gateway uses a **consensus-seeking model**, with voting as a fallback, not a first resort.

### 3.1 Day-to-Day Decisions

Most decisions — accepting a PR, triaging an issue, answering a design question in review — are made by whichever reviewer or maintainer is handling that piece of work, using the standards in `CONTRIBUTING.md`. No vote is needed for routine work; requiring one would make the project impossible to operate.

### 3.2 Significant Decisions

Decisions with broader impact — accepting an RFC, changing governance, admitting a new maintainer, a major architectural shift — follow this sequence:

1. **Proposal**, typically via the RFC process (`CONTRIBUTING.md` §13) or a governance-specific discussion for non-RFC matters (e.g., a maintainer nomination).
2. **Discussion period**, open to the whole community, not just maintainers — anyone can raise concerns, and concerns are expected to be substantively addressed, not just acknowledged.
3. **Consensus check** among maintainers: does anyone have a standing objection, as opposed to a preference for a different approach? A standing objection must be grounded in a concrete technical, security, or project-health concern — not simply "I'd have done it differently."
4. **If consensus is reached**, the decision is recorded (RFC merged, decision documented in the relevant issue/discussion) and moves forward.
5. **If consensus cannot be reached** after reasonable discussion (see §8 for what "reasonable" means in practice), the decision moves to a formal vote among maintainers (§6).

We prefer consensus because a decision reached by convincing people tends to be a better decision, and tends to have fewer people quietly working around it afterward. Voting exists as a real fallback, not a rubber stamp, but is deliberately the second resort.

---

## 4. RFC Process Ownership

The RFC process itself is described procedurally in `CONTRIBUTING.md` §13. From a governance perspective:

- Any maintainer can shepherd an RFC (help the author refine it, drive discussion toward resolution) but shepherding does not imply approval authority.
- RFC acceptance requires maintainer consensus per §3, or a vote if consensus fails.
- RFCs affecting the security engine, authentication, authorization, or parser trust boundary additionally require security team sign-off per §10, regardless of general maintainer consensus — a security-relevant RFC cannot be accepted over a standing security team objection through a general maintainer vote. This is a deliberate override of the normal voting fallback, because security-critical design decisions need domain-specific authority, not just numerical majority.

---

## 5. Changing This Document

Changes to `GOVERNANCE.md` itself follow the significant-decision process in §3, with two additional requirements:

- A minimum discussion period (published in the repository's contribution calendar / pinned discussion, currently two weeks) to ensure the community has real opportunity to weigh in on changes to how decisions get made.
- Explicit maintainer consensus (or vote, if consensus fails) is required — governance changes cannot be merged as a routine documentation PR, even though they are technically just a markdown file change.

---

## 6. Voting

When consensus fails per §3.2, a formal vote among current maintainers is held:

- **Eligible voters**: all current maintainers (§2), excluding anyone with a disclosed conflict of interest on the specific matter (per `CODE_OF_CONDUCT.md` §2's disclosure expectation).
- **Threshold**: simple majority for most decisions; **two-thirds majority** for governance changes (§5), maintainer removal (§9.3), and any decision that would materially weaken a security guarantee described in `ARCHITECTURE.md` §2 or `SECURITY.md` — the higher bar reflects that these categories of decisions are harder to reverse and higher-consequence if wrong.
- **Quorum**: a majority of current maintainers must participate for a vote to be binding; if quorum isn't reached within the discussion window, the vote period extends rather than passing by default.
- **Transparency**: vote outcomes (not necessarily individual votes, if privacy is a concern for a sensitive matter) are recorded publicly, alongside the reasoning, so the community can see not just what was decided but why.

---

## 7. Release Process

- Release readiness is assessed against the exit criteria published in `ROADMAP.md` for the relevant milestone — a release does not ship because a date arrived, it ships because its scope is actually done, per the philosophy in `ROADMAP.md` §3.
- Any maintainer can propose cutting a release once they believe exit criteria are met; release approval requires sign-off from at least two maintainers, including explicit security team sign-off (§10) for any release including changes to `prism-security`, `prism-authn`, `prism-authz`, or a parser.
- Release notes are required for every tagged release and must call out `BREAKING` changes explicitly, per `ROADMAP.md` §3.3's pre-1.0 compatibility policy.
- Post-`v1.0`, release process will additionally follow the semantic versioning commitments in `ROADMAP.md` §4, and this section will be expanded to cover backport/patch release procedures for supported prior major versions.

---

## 8. Conflict Resolution

Technical and process disagreements are a normal, healthy part of building infrastructure software — this section is about handling them productively, not avoiding them.

1. **Discuss in the open venue where the disagreement arose** (the PR, the RFC thread, the issue) first. Most disagreements resolve here once both sides' underlying concerns — not just their proposed solutions — are actually understood.
2. **If discussion stalls**, either party can ask a maintainer not otherwise involved to weigh in as a neutral third opinion. This is a normal, non-escalatory step, not a signal that something has gone wrong.
3. **If the disagreement is about project direction rather than a specific technical point**, it's a candidate for the RFC process (`CONTRIBUTING.md` §13), which gives it a structured venue with a defined resolution path.
4. **If the disagreement involves conduct**, not just substance, it goes through `CODE_OF_CONDUCT.md`'s reporting and enforcement process instead — conflating a technical disagreement with a conduct issue helps no one, and we try to be careful to keep these tracks separate.
5. **As a last resort for a stalled significant decision**, it goes to a maintainer vote per §6. We treat reaching this step as acceptable, not a failure — some decisions genuinely have no consensus answer, and a clear, transparent vote is better than a disagreement that never gets resolved.

---

## 9. Becoming a Maintainer

### 9.1 Path

There is no fixed tenure requirement, but in practice, maintainer status follows sustained demonstration of:

- **Technical judgment**, evidenced by review comments and contributions that consistently reflect the principles in `ARCHITECTURE.md` and the standards in `CONTRIBUTING.md`.
- **Reliability**, evidenced by following through on review commitments and being a predictable, trustworthy presence in the areas they're active in.
- **Alignment with project values**, evidenced by how they engage in disagreement — per `CODE_OF_CONDUCT.md`, this matters as much as technical skill.
- **Breadth beyond a single PR** — active review of others' contributions, participation in RFC discussion, and issue triage, not just authoring code.

### 9.2 Nomination and Approval

Any current maintainer can nominate a contributor for maintainer status, with a brief written rationale posted for maintainer discussion. Approval follows the significant-decision process in §3 — consensus among current maintainers, with a vote (§6) as fallback if consensus isn't reached.

### 9.3 Stepping Back and Removal

- **Stepping back** is always available, no justification required — maintaining open-source infrastructure is unpaid, volunteer effort for most contributors, and life circumstances change. A maintainer who steps back is welcome to return later without re-running the full nomination process, at the discretion of current maintainers.
- **Inactivity**: a maintainer who has been unreachable and inactive for an extended period (a de-facto threshold discussed case-by-case rather than a rigid cutoff, out of respect for the fact that people have lives outside this project) may be moved to emeritus status by maintainer consensus, freeing up review-load expectations without it being framed as a punitive action.
- **Removal for cause** (conduct violations, sustained failure to meet the standards in §9.1) follows the conflict-resolution process in §8 and, if unresolved, the two-thirds voting threshold in §6 — a deliberately high bar, reflecting that removal is a serious, largely irreversible action toward a person who has typically invested significant volunteer effort in the project.

---

## 10. Security Team

The security team is a subset of maintainers additionally responsible for:

- Triaging and responding to reports through the process in `SECURITY.md`
- Holding review authority on `prism-security`, `prism-authn`, `prism-authz`, and all parser crates, per `CONTRIBUTING.md` §16
- Sign-off authority on RFCs and releases touching those areas, per §4 and §7 of this document
- Maintaining the `UNSAFE.md` inventory described in `SECURITY.md` §8

Security team membership is intentionally more selective than general maintainer status — it requires demonstrated security-specific judgment (not just general engineering skill), and is approved by existing security team members' consensus rather than the general maintainer body, though the security team itself remains accountable to overall project governance and can be expanded or restructured through the standard significant-decision process.

We keep the security team deliberately small and high-trust rather than large — the value of the role is specifically in consistent, deep judgment on a narrow, high-stakes surface, not broad coverage.

---

## 11. Working Groups

As the project grows beyond what the core maintainer group can cover directly, we expect to form working groups scoped to specific domains — for example, a Dashboard/UX working group, or a Format Support working group focused on new parser/encoder implementations.

- Working groups are formed by maintainer consensus when a domain has sustained, distinct activity that benefits from dedicated coordination.
- Each working group has a lead (§2), responsible for coordinating scope and reporting status to the broader maintainer group, particularly for community meetings (§12).
- Working groups do not have independent decision-making authority beyond their delegated scope — significant decisions within a working group's domain still follow §3, with the working group's recommendation carrying real weight but not final authority, except where explicitly delegated by maintainer consensus.
- Working groups are a `Planned` structure as of this writing — see `ROADMAP.md` §7 for the community milestone tracking the formation of a second working group beyond the initial core/security groups.

---

## 12. Community Meetings

We intend to hold periodic, open community meetings once project activity justifies the coordination overhead (tracked as a milestone rather than committed to a fixed cadence prematurely). When active, community meetings will:

- Be open to anyone, not maintainers-only
- Have published notes for anyone who couldn't attend synchronously — we treat "you had to be in the room" as a governance failure mode to actively avoid
- Cover roadmap status, open RFC discussion, and a standing "raise anything" segment

Until this is formally established, roadmap and design discussion happens asynchronously through GitHub Discussions and RFC threads, which remain the primary venue for anyone unable to attend a live meeting even after one exists.

---

## 13. Long-Term Sustainability

Governance exists to keep the project healthy beyond any single contributor's availability. Concretely, this means:

- **No single point of failure in maintainer access** — repository administration, release signing, and domain/infrastructure credentials are held by more than one person, with documented recovery procedures.
- **Deliberate investment in maintainer pipeline** — the reviewer → maintainer path in §9 exists specifically so the project isn't structurally dependent on its founding group indefinitely.
- **Willingness to formalize further** as the project grows — a fiscal sponsorship or foundation relationship (similar in spirit to CNCF- or Rust-Foundation-adjacent structures) is a plausible future step if project scale and funding needs justify it, and would itself go through the significant-decision governance process rather than being decided unilaterally.
- **Honesty about capacity** — if the maintainer group is ever genuinely unable to sustain the response-time commitments in `SECURITY.md` or the review standards in `CONTRIBUTING.md`, we consider that a governance problem to surface and address openly, not one to quietly let slide.

A project that handles security-sensitive traffic has an obligation to its users to be honest about its own sustainability, not just its code quality. This section is our commitment to taking that obligation seriously.

---

*This document is versioned alongside the project and changes through the process described in §5. If something in this document doesn't match how the project actually operates, that mismatch is itself worth raising — governance documentation that drifts from practice is a problem regardless of which one is "wrong."*
