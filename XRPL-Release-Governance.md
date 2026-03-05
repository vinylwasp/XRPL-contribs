# XRPL Release Governance Framework

**Version:** 1.1 draft
**Date:** 2026-03-05
**Author:** Vinylwasp
**Status:** Proposal
**Scope:** rippled (consensus node), XRPL-Standards (specifications), client libraries (xrpl.js, xrpl-py, xrpl4j)

---

## Executive Summary

The XLS-0056 Batch amendment vulnerability — an authentication bypass that reached the validator voting stage one vote from mainnet activation — exposed systemic gaps in XRPL's development lifecycle: no threat modelling, self-merges on security-critical code, closed review loops, no structured security evidence available to validators before voting, and no requirement for live-network testing before amendments reach production. This framework defines a gate-based release governance model that prescribes **what evidence** must exist at each stage, **who** is responsible, and **how** decisions are made — while respecting XRPL's architecture where code is built by a core maintainer team but amendment activation is governed by decentralised validator voting. The framework is tool-agnostic, open to community participation at every stage, and designed for phased adoption.

---

## Core Principles

Three principles underpin this framework and apply to every gate, role, and process defined below:

### Open Participation

Anyone can contribute to any stage of the XRPL development lifecycle — specification, threat modelling, code, review, testing, and security analysis. Contributions are governed by structure and quality gates, not by access restrictions. A security researcher challenging a spec design is as valuable as the developer who wrote it.

> **Gatekeeping is applied to the quality of evidence, never to who may provide it.**

### Transparency

All security-relevant decisions, evidence, and rationale are public by default. Review status, threat models, test results, and evidence packs are visible to the community and to validators. No closed review loops. No hidden gates. When a validator votes on an amendment, they can see exactly what security evidence exists — or doesn't.

### Recognition

Contributions are attributed and rewarded. Threat model contributions, adversarial test cases, security reviews, and design-stage flaw discoveries are credited to their authors. Recognition mechanisms may include public attribution, bug bounty payments, XRP grants, or other community-determined incentives. Finding a design flaw before code is written is at least as valuable as finding an implementation bug after release.

---

## Normative References

| Standard | Relevance |
|---|---|
| NIST SSDF v1.1 (SP 800-218) | Secure software development practices |
| OWASP SAMM v2.0 | Software assurance maturity model |
| SLSA v1.0 | Supply-chain levels for software artefacts |
| PCI DSS v4.0 | Payment security requirements (§6 secure development) |
| NIST CSF 2.0 | Cybersecurity framework — governance functions |
| OpenSSF Scorecard | Open-source project security health metrics |

---

## §1 — Scope and Applicability

### What This Framework Covers

| Component | Repository | Nature |
|---|---|---|
| **rippled** | `XRPLF/rippled` | Consensus node — C++ implementation |
| **XRPL-Standards** | `XRPLF/XRPL-Standards` | XLS specifications defining protocol behaviour |
| **Client libraries** | `XRPLF/xrpl.js`, `XRPLF/xrpl-py`, `XRPLF/xrpl4j` | SDK/library implementations |

### The XRPL Release Architecture

The XRPL has a two-phase release model that this framework must respect:

**Phase A — Centralised Build Pipeline (maintainer-controlled)**

Code is proposed (by anyone — core team or external contributor), reviewed, merged, and compiled into a `rippled` binary. This phase is a traditional software development process governed by the maintainer team. The gates in this framework (Gates 0–4) apply here and can be enforced through repository policy, CI/CD, and maintainer discipline.

**Phase B — Decentralised Amendment Activation (validator-governed)**

New protocol features are gated behind amendments. Validators independently decide whether to vote YES on an amendment. When ≥80% of validators vote YES for two consecutive weeks, the amendment activates irreversibly on mainnet. No single entity controls this — it is a sovereign decision by each validator operator.

**Implication:** This framework cannot mandate validator behaviour. Gate 6 (Amendment Activation) provides validators with structured security evidence and guidance to support informed voting decisions. The evidence pack is an **input to decentralised governance**, not a centralised approval gate.

### Amendment Lifecycle Mapping

| Lifecycle Stage | Framework Gate |
|---|---|
| XLS specification draft | Gate 0 — Specification Review |
| Security analysis of proposed design | Gate 1 — Threat Model |
| Code implementation and PR review | Gate 2 — Implementation Review |
| Testing, static analysis, fuzzing | Gate 3 — Verification |
| Testnet deployment, bug bounty, independent testing | Gate 4 — Testnet Validation |
| Binary build and release | Gate 5 — Release Candidate |
| Validator voting on amendment | Gate 6 — Amendment Activation Evidence |

---

## §2 — Change Classification

Not every change requires a full threat model. Classification determines which gates are mandatory, reducing friction for low-risk changes while enforcing rigour where it matters.

### Classification Ontology

| Class | Description | Examples | Mandatory Gates |
|---|---|---|---|
| **A — Consensus / Cryptographic** | Changes to consensus logic, transaction validation, signing, key derivation, or amendment activation | Batch transactions, signature validation changes, new transaction types that modify ledger state, cryptographic primitive changes | 0, 1, 2, 3, 4, 5, 6 |
| **B — Protocol / Ledger State** | Changes to ledger objects, protocol fields, serialisation, or network message handling that do not alter consensus rules | New ledger entry types, RPC method additions, peer protocol changes | 0, 1 (scoped), 2, 3, 4, 5, 6 |
| **C — Client / Integration** | SDK changes, client library updates, tooling | New xrpl.js methods, documentation, client-side validation | 0, 2, 3, 5 |
| **D — Maintenance** | Bug fixes, dependency updates, refactoring, CI changes, documentation-only | Typo fixes, dependency bumps, test improvements | 2, 3 |

### Classification Criteria

The change author proposes a classification in the PR description. A maintainer (not the author) confirms or escalates the classification before review proceeds.

**Escalation rule:** If a Class C or D change touches files in consensus-critical paths (`src/ripple/app/tx/`, `src/ripple/consensus/`, `src/ripple/protocol/`), it is automatically escalated to Class B minimum, regardless of the stated intent. Any change that modifies signature validation, key handling, or authentication logic is Class A regardless of scope.

---

## §3 — Roles and Responsibilities

### Defined Roles

| Role | Responsibility | Separation Requirements |
|---|---|---|
| **Spec Author** | Writes the XLS specification, including security considerations section | — |
| **Threat Model Facilitator** | Initiates and coordinates the threat modelling process; ensures community contributions are incorporated | Must not be the sole author of the spec under review |
| **Community Threat Contributor** | Any individual who contributes attack scenarios, abuse cases, or security analysis to a threat model | Open to all; contributions are publicly attributed |
| **Implementer** | Writes the code (PR author) | — |
| **Code Reviewer** | Reviews implementation for correctness and security | Must not be the PR author; for Class A/B, at least one reviewer must be independent of the feature team |
| **Security Reviewer** | Reviews for adversarial resilience, authentication/authorisation correctness, and attack surface | Must not be the PR author or implementer; for Class A, must not be from the same feature team |
| **Testnet Validation Coordinator** | Manages testnet deployment, bug bounty programme, engagement tracking, and testnet evidence collection | Must not be the sole implementer of the amendment under test |
| **Release Manager** | Assembles evidence pack, verifies gate compliance, coordinates binary release | Must not be the sole implementer of any Class A feature in the release |
| **Validator Operator** | Makes sovereign voting decision on amendment activation using available evidence | Independent; not governed by this framework |

### Separation of Duties

The following constraints are non-negotiable for Class A and B changes:

1. **No self-merge.** The person who authored a PR must not be the person who merges it. *(XLS-0056 finding: PR #6069 was authored and merged by the same developer.)*
2. **Independent review.** At least one reviewer must be outside the immediate feature team. *(XLS-0056 finding: all Batch PRs were reviewed within a closed loop of three developers.)*
3. **Approval before merge, not after.** GitHub approval status must be "Approved" (not merely "Commented") before merge is permitted. *(XLS-0056 finding: PR #5060 was merged with all reviewers in COMMENTED status only.)*
4. **Escalated concerns block merge.** If a reviewer raises a concern, the PR cannot be merged until the concern is resolved with a documented response. *(XLS-0056 finding: @dangell7's "This doesn't seem right" comment on PR #6069 was not resolved before merge.)*

### Community Participation in Roles

The Security Reviewer, Community Threat Contributor, and bug bounty participant roles are explicitly open to the community. The XRPL Foundation or maintainer team should:

- Maintain a public list of recognised security contributors
- Provide structured templates for community threat contributions (see Gate 1)
- Ensure community-submitted adversarial test cases are evaluated and, if valid, integrated
- Attribute all accepted contributions publicly in the evidence pack

---

## §4 — Release Gates

Each gate defines: purpose, required evidence, who approves, and pass/fail criteria. Gates are sequential — a gate must be passed before work dependent on it proceeds. Evidence is **outcome-based, not tool-specific**: any methodology or tool that produces conforming evidence is acceptable.

---

### Gate 0 — Specification Review

**Purpose:** Ensure the XLS specification is complete, includes security considerations, and has been subject to open review.

**Applies to:** Class A, B, C

| Evidence Required | Criteria |
|---|---|
| XLS specification document | Published in XRPL-Standards repository as a PR, open for public comment |
| Security considerations section | Present in the spec; identifies trust boundaries, authentication/authorisation model, and data validation requirements |
| Trust boundary diagram | For Class A/B: documents which components trust which inputs, and where validation occurs |
| Public review period | Minimum 14 days for Class A, 7 days for Class B, from the date the spec PR is opened |
| Reviewer sign-off | At least two reviewers (not the author) have approved the spec, with at least one providing security-focused feedback |

**Who approves:** Spec reviewers (community or maintainer)

**Failure criteria:**
- No security considerations section
- Trust boundaries not identified for features involving multi-party authentication or cross-boundary data flow
- Review period not met
- Spec merged by author (self-merge)

**XLS-0056 gap:** The Batch specification had no security considerations section, no trust boundary analysis of the inner/outer transaction authentication model, and the spec was still being updated (PR #452, self-merged) two days before the vulnerability was disclosed while the amendment was already in validator voting.

---

### Gate 1 — Threat Model

**Purpose:** Systematically identify what could go wrong, who could abuse the feature, and what the blast radius is — before code is written.

**Applies to:** Class A (full), Class B (scoped to changed attack surface)

**This gate is explicitly open to community participation.** The threat model is a living document, initiated by the spec author or threat model facilitator and open for contribution by anyone — security researchers, competing implementation teams, academics, or any community member.

| Evidence Required | Criteria |
|---|---|
| Structured threat analysis | Uses any recognised methodology (STRIDE, PASTA, attack trees, misuse cases, or equivalent) applied to the feature's trust boundaries |
| Attacker personas | At least: (1) external attacker with no keys, (2) authenticated attacker abusing their own access, (3) colluding parties |
| Evil user stories | Adversarial scenarios written from the attacker's perspective, covering at minimum: authentication bypass, authorisation escalation, input manipulation, state confusion |
| Community contribution window | Threat model published as a public document (GitHub issue, spec addendum, or dedicated file) with a minimum 14-day open contribution period for Class A |
| Contributor attribution | All community threat contributions are publicly credited by name/handle |
| Risk acceptance | Any identified threats not mitigated must have documented risk acceptance with rationale, signed off by a maintainer who is not the spec author |

**Who approves:** Threat Model Facilitator + at least one Security Reviewer

**Failure criteria:**
- No structured threat analysis exists
- Threat model was produced solely by the feature author with no independent review or community input
- Identified threats have no documented mitigation or risk acceptance
- Community contribution window not provided for Class A changes

**Methodology note:** This framework does not mandate a specific threat modelling tool or notation. STRIDE worksheets, attack trees, tabular misuse cases, or narrative adversarial analysis are all acceptable provided they systematically cover the feature's trust boundaries and produce actionable findings.

**XLS-0056 gap:** No threat model existed at any stage. The inner/outer transaction authentication boundary — a textbook trust boundary — was never formally analysed. The exact attack scenario that was exploited ("attacker places a signer for a non-existent account first in the array, skipping validation of subsequent signers") would be a natural output of STRIDE spoofing analysis or an evil user story exercise.

---

### Gate 2 — Implementation Review

**Purpose:** Ensure code is reviewed by someone other than the author, with security-aware scrutiny proportional to the change classification.

**Applies to:** All classes

| Evidence Required | Criteria |
|---|---|
| PR with classification label | Author has proposed a classification (A/B/C/D); a maintainer (not the author) has confirmed it |
| Code review — correctness | At least one reviewer (not the author) has approved the PR via GitHub's approval mechanism (not just COMMENTED) |
| Code review — security (Class A/B) | At least one reviewer independent of the feature team has reviewed with explicit attention to: authentication logic, authorisation checks, input validation at trust boundaries, loop/exit behaviour in security-critical paths |
| Reviewer concern resolution | All reviewer comments marked as concerns or questions are resolved with documented responses before merge |
| No self-merge | The person who clicks "Merge" is not the PR author |
| Branch protection | Target branch requires PR, required reviewers, and no force-push |

**Who approves:** Code Reviewer(s) + Security Reviewer (Class A/B)

**Failure criteria:**
- PR merged with only COMMENTED reviews (no formal approval)
- Self-merge
- Unresolved reviewer concerns at time of merge
- Class A/B PR merged without an independent (non-feature-team) reviewer
- Branch protection not enforced on target branch

**XLS-0056 gap:** PR #5060 (69 files, +6,400 lines) was merged with all reviewers in COMMENTED status — no formal GitHub approval. PR #6069 was self-merged by the author. @dangell7's concern ("This doesn't seem right") was not resolved before merge. All reviews were within a closed loop of three developers with no independent security perspective.

---

### Gate 3 — Verification

**Purpose:** Demonstrate through testing and analysis that the implementation is resilient to adversarial inputs, not just functionally correct.

**Applies to:** All classes (scope varies)

| Evidence Required | Criteria |
|---|---|
| Static analysis (SAST) | Scan completed using any tool capable of detecting the relevant vulnerability classes (e.g., authentication bypass, injection, resource exhaustion). No unresolved critical or high findings in changed code. Tool and version documented. |
| Adversarial test matrix (Class A/B) | Documented matrix of adversarial test cases derived from the threat model. Must cover trust boundary violations, authentication bypass scenarios, and input manipulation. Matrix must include edge cases at boundaries (empty arrays, maximum sizes, non-existent entities, malformed inputs). |
| Fuzz targets (Class A) | Fuzz targets covering security-critical parsing and validation paths. Minimum runtime and corpus size documented. |
| Code coverage | Coverage report for changed code. Coverage alone is not sufficient — the adversarial test matrix must demonstrate that security-critical paths are exercised with adversarial inputs, not just reached. |
| Regression tests | Tests that specifically reproduce any threats identified in Gate 1, confirming mitigations work |

**Who approves:** Code Reviewer(s) confirm test matrix coverage; Security Reviewer (Class A/B) confirms adversarial adequacy

**Failure criteria:**
- No static analysis results
- Class A/B with no adversarial test matrix
- Adversarial test matrix does not cover scenarios from the threat model
- Critical/high SAST findings unresolved
- Class A with no fuzz targets for security-critical paths

**Tool-agnostic note:** Any SAST tool (CodeQL, Semgrep, Coverity, Clang Static Analyzer, or equivalent) is acceptable. Any fuzzing framework (libFuzzer, AFL, Honggfuzz, or equivalent) is acceptable. Any test framework is acceptable. The repository may provide CI-integrated open-source tooling as a default baseline, but contributors are not required to use it — only to provide equivalent evidence.

**XLS-0056 gap:** No evidence of static analysis. No adversarial test cases — the test suite verified functional behaviour but included no scenarios with non-existent signer accounts, no early-exit validation paths, and no authentication bypass attempts. 94% patch coverage on PR #5060 covered the vulnerable code but with correct-path inputs only. The combinatorial state space (4 batch modes × multi-account signing × existent/non-existent accounts × valid/invalid keys × signer position) was not systematically explored.

---

### Gate 4 — Testnet Validation

**Purpose:** Validate the amendment under live network conditions with independent adversarial testing before the binary is released to production validators. Produce measurable evidence that the amendment has been subjected to real-world scrutiny beyond the feature team's automated tests.

**Applies to:** Class A (full), Class B (scoped)

#### Why Automated Testing Alone Is Insufficient

Automated tests — even adversarial ones — operate within the test author's imagination. They validate scenarios someone thought to write. A testnet deployment creates conditions that CI cannot replicate:

- **Multi-party interaction**: Different wallets, SDKs, and tools submit transactions independently, creating interaction patterns the feature team didn't anticipate.
- **Adversarial exploration**: Security researchers probe a running system, not a test harness. They try things the spec doesn't describe.
- **Integration surface**: Client libraries, wallets, explorers, and DEX interfaces interact with the amendment through their own implementations, exposing integration assumptions that unit tests don't cover.
- **Time under load**: A soak period reveals stability issues, memory leaks, performance degradation, and state-dependent bugs that only manifest over sustained operation.
- **Real protocol behaviour**: Consensus, peer propagation, ledger close timing, and validator agreement interact with the amendment in ways that a single-node test environment cannot reproduce.

#### Evidence Requirements

| Evidence Required | Criteria | Class A | Class B |
|---|---|---|---|
| **Testnet deployment record** | Amendment force-enabled on testnet with confirmed activation. Deployment date, rippled version, and amendment hash documented. | Required | Required |
| **Dedicated bug bounty period** | Bug bounty programme specifically scoped to the amendment's attack surface, with clear scope definition, reward tiers, and submission process. Published on recognised platforms and XRPL community channels. | Required (28 days minimum) | Recommended (14 days) |
| **Evidence of independent tester engagement** | Documented evidence that independent parties (not the feature team) actively tested the amendment. See engagement criteria below. | Required — minimum thresholds defined | Required — qualitative evidence |
| **Integration test results** | Client libraries (xrpl.js, xrpl-py, xrpl4j) tested against the amendment on testnet. Wallet and explorer compatibility confirmed. Integration test results published. | Required | Required |
| **Testnet soak period** | Amendment running on testnet for the minimum soak period without consensus failures, crashes, or degraded performance. Monitoring data published. | 28 days | 14 days |
| **Bug bounty engagement report** | Summary of bounty programme: number of participants, number of submissions, valid findings, findings by severity, resolution status. Published as part of the evidence pack. | Required | Recommended |
| **Testnet adversarial transaction log** | Sample of adversarial transactions submitted during the bounty period demonstrating the scope of testing performed. Sanitised to remove researcher-identifying information if requested. | Required | Recommended |

#### Evidence of Independent Tester Engagement

This is the critical requirement that prevents the testnet phase from becoming a checkbox exercise. "We ran a bug bounty and nobody submitted anything" is not evidence of security — it is evidence of non-engagement. The gate requires **affirmative evidence that independent parties actively tested the amendment**.

**Engagement criteria for Class A amendments** — at least **three** of the following must be demonstrated:

1. **Bounty programme participation**: A minimum of 5 unique researchers registered or submitted reports (valid or invalid) against the amendment-scoped bounty. A submission of "no issues found after X hours of testing" with a described methodology counts.

2. **Community testing reports**: At least 2 independent testing reports published by parties outside the feature team, describing what was tested, how, and what was found (including null findings). Reports are attributed (or pseudonymously attributed) and published in the XRPL-Standards repository or linked from the evidence pack.

3. **SDK/wallet integration testing**: At least 2 independent wallet or SDK teams have tested their integration against the testnet-enabled amendment and published results.

4. **Adversarial transaction evidence**: Transaction logs on testnet demonstrating that adversarial inputs (malformed transactions, boundary violations, authentication bypass attempts, state confusion scenarios) were submitted by parties other than the feature team.

5. **Security researcher attestation**: At least 1 recognised security researcher or firm has reviewed the amendment on testnet and provided a written attestation (positive, negative, or qualified). "We spent 40 hours on this and found no issues" is valid. "We spent 2 hours and ran out of time" is valid and honestly useful — it tells validators the depth of review.

#### Bug Bounty Design

The amendment-scoped bug bounty should be distinct from the standing XRPL bug bounty programme:

**Scope**: Derived from the Gate 1 threat model. In scope: all identified attack scenarios plus any additional adversarial interactions with the amendment's trust boundaries, integration failures, and denial-of-service vectors specific to the amendment. Out of scope: issues unrelated to the amendment, known issues documented in the threat model's risk acceptance records.

**Reward structure** (Class A amendments):

| Severity | Description | Suggested minimum reward |
|---|---|---|
| Critical | Authentication bypass, unauthorised fund movement, consensus failure | $100,000+ |
| High | Authorisation escalation, significant state corruption, denial of service to validators | $25,000–$100,000 |
| Medium | Limited state corruption, information disclosure, integration failures causing user-facing risk | $5,000–$25,000 |
| Low | Edge-case misbehaviour, cosmetic issues, minor integration inconsistencies | $1,000–$5,000 |
| Informational / participation | Documented null finding with methodology description | $500–$1,000 |

The **informational / participation tier** is deliberate. It incentivises researchers to engage and document their testing methodology even when they find no issues. Five researchers spending 40 hours each and finding nothing is meaningful assurance. Zero researchers engaging is not.

**Timing**: The bug bounty closes **before** the release candidate is built. All critical and high findings must be resolved. The resolution status of all findings is published in the evidence pack.

```
                                                    Bug bounty findings
                                                    must be resolved
                                                    BEFORE this point
                                                              |
  Gate 3        Testnet         Bug bounty          Gate 5    v    Gate 6
  (Verify) --> Deployment --> Period (28d) --> RC Build --> Validator
                    |                                         Voting
                    |
            Amendment force-enabled
            on testnet
```

#### Testnet Monitoring

During the soak period, the following metrics should be collected and published:

| Metric | Purpose |
|---|---|
| Consensus round completion rate | Detect consensus disruption caused by the amendment |
| Ledger close time distribution | Detect performance degradation |
| Validator agreement rate | Detect divergence caused by amendment edge cases |
| Transaction success/failure rate for amendment-related transaction types | Detect unexpected failures |
| Memory and CPU utilisation on testnet validators | Detect resource exhaustion |
| Peer connection stability | Detect network-level issues |
| Error log analysis | Detect unexpected error conditions |

#### When Engagement Thresholds Are Not Met

If the minimum engagement criteria are not met after the initial bounty period:

1. **Do not waive the gate.** Low engagement is not evidence of security — it is evidence of insufficient scrutiny.
2. **Diagnose the cause.** Is it rewards? Scope clarity? Infrastructure accessibility? Community awareness?
3. **Extend and remediate.** Extend the bounty period by 14–28 days while addressing engagement barriers. Increase rewards if necessary. Publish improved testing documentation. Actively recruit through additional channels.
4. **Escalate if persistent.** If engagement remains below thresholds after extension, raise this as a risk factor in the Gate 6 evidence pack — validators should know the amendment received limited independent scrutiny despite efforts to attract it.
5. **Document the outcome.** Whether engagement is high, low, or zero, the result is documented honestly in the evidence pack.

**Who approves:** Testnet Validation Coordinator confirms deployment, soak completion, and engagement evidence; Security Reviewer confirms adversarial adequacy of testnet testing

**Failure criteria:**
- Amendment not deployed to testnet
- Soak period not met
- No bug bounty programme for Class A amendment
- Engagement thresholds not met and not escalated with documented remediation attempts
- Unresolved critical/high bug bounty findings
- No integration test results

**XLS-0056 gap:** No testnet validation phase existed. The bug bounty that caught the authentication bypass was running against code already deployed to production validators, in an active voting window, one vote from irreversible activation. The security researcher found the vulnerability in production's anteroom — not in a controlled testing environment with time to respond. A 28-day testnet bounty period would have caught the same vulnerability weeks earlier, on testnet, with zero financial exposure, and the development team would have had a full soak period to fix and re-test instead of a 4-day emergency response.

---

### Gate 5 — Release Candidate

**Purpose:** Ensure the binary release is reproducible, attributable, and accompanied by a complete evidence trail.

**Applies to:** Class A, B, C (Class D if included in a release containing A/B/C changes)

| Evidence Required | Criteria |
|---|---|
| Reproducible build | Build process documented and independently reproducible. Target: SLSA Build Level 2 minimum (scripted build, version-controlled build config, build service generates provenance). |
| Signed artefacts | Binary and source artefacts are cryptographically signed. Signing key management documented. |
| Software Bill of Materials (SBOM) | SBOM in SPDX or CycloneDX format listing all dependencies |
| Evidence pack | Compiled evidence from Gates 0–4 for all changes in the release, with SHA-256 hash of the pack published as a release artefact |
| Release notes | Changelog identifying all amendments included, their classifications, and links to evidence |

**Who approves:** Release Manager (must not be the sole implementer of any Class A feature in the release)

**Failure criteria:**
- Build not reproducible by an independent party
- No signed artefacts
- Evidence pack incomplete (missing gate evidence for any Class A/B change)
- Release notes do not identify included amendments

**XLS-0056 gap:** No evidence pack existed. No structured release documentation tying the Batch amendment to security evidence. Validators had no consolidated security information to reference when deciding whether to vote for the amendment.

---

### Gate 6 — Amendment Activation Evidence

**Purpose:** Provide validators and the community with structured security evidence to support informed, sovereign voting decisions on amendment activation.

**Applies to:** Class A, B (any change gated behind an amendment)

**This is not an approval gate.** Validator voting is decentralised and sovereign. This gate ensures that the evidence validators need to make an informed decision is available, structured, and transparent.

| Evidence Provided to Validators | Contents |
|---|---|
| Security attestation summary | One-page document summarising: what the amendment does, what security analysis was performed, what threats were identified and how they were mitigated, what residual risks exist, and what monitoring is recommended post-activation |
| Link to full evidence pack | Gates 0–5 evidence, publicly accessible |
| Testnet validation summary | Soak period duration, bug bounty engagement metrics, number and severity of findings, resolution status, independent tester engagement evidence |
| Validator guidance document | Amendment-specific guidance: what validators should test on their own infrastructure, what to monitor after activation, and how to disable the amendment if issues are discovered post-activation |
| Rollback/disable strategy | Documented procedure for disabling the amendment if a critical issue is discovered after activation, including the expected timeline and coordination mechanism |
| Community threat model | The full community-contributed threat model from Gate 1, including any unresolved or contested findings |

**Who produces:** Release Manager, with Security Reviewer sign-off on the attestation summary

**Published:** In the XRPL-Standards repository alongside the XLS specification, and announced through validator communication channels

**XLS-0056 gap:** No security attestation existed. No validator guidance document was published. Validators had no structured information about the security analysis (or lack thereof) performed on the Batch amendment. The amendment reached the voting stage without any public security evidence. Validators were being asked to make an irreversible activation decision with no security information beyond the spec and the code itself.

---

## §5 — Evidence Pack

### Per-Gate Evidence Requirements

| Gate | Evidence Artefact | Format | Retention |
|---|---|---|---|
| 0 — Specification | Approved XLS spec with security considerations | Markdown in XRPL-Standards | Permanent |
| 0 — Specification | Trust boundary diagram (Class A/B) | Diagram (any format) in spec repository | Permanent |
| 1 — Threat Model | Structured threat analysis with community contributions | Public document (GitHub issue, repo file, or spec addendum) | Permanent |
| 1 — Threat Model | Risk acceptance records | Signed-off document in spec repository | Permanent |
| 2 — Implementation | PR review record with approval status | GitHub PR (immutable) | Permanent (GitHub) |
| 2 — Implementation | Reviewer concern resolution log | GitHub PR comments (immutable) | Permanent (GitHub) |
| 3 — Verification | SAST report summary | Tool output (tool and version documented) | Per release cycle + 2 years |
| 3 — Verification | Adversarial test matrix and results | Markdown or structured format in repository | Permanent |
| 3 — Verification | Fuzz target definitions and run summary (Class A) | Repository files + summary document | Permanent |
| 3 — Verification | Code coverage report | Tool output | Per release cycle |
| 4 — Testnet Validation | Testnet deployment record (date, version, amendment hash) | Structured document | Permanent |
| 4 — Testnet Validation | Bug bounty programme specification (scope, rewards, timeline) | Markdown | Permanent |
| 4 — Testnet Validation | Bug bounty engagement report (participants, submissions, findings, resolutions) | Markdown | Permanent |
| 4 — Testnet Validation | Independent testing reports | Markdown or PDF | Permanent |
| 4 — Testnet Validation | Integration test results (SDK/wallet compatibility) | Structured document | Permanent |
| 4 — Testnet Validation | Testnet monitoring data (soak period metrics) | Structured data + summary | Per release cycle + 2 years |
| 4 — Testnet Validation | Adversarial transaction log (sanitised) | Transaction hashes + descriptions | Per release cycle + 2 years |
| 5 — Release Candidate | SBOM | SPDX or CycloneDX | Per release + 2 years |
| 5 — Release Candidate | Build provenance record | SLSA provenance format | Per release + 2 years |
| 6 — Activation Evidence | Security attestation summary | Markdown in XRPL-Standards | Permanent |
| 6 — Activation Evidence | Testnet validation summary | Markdown in XRPL-Standards | Permanent |
| 6 — Activation Evidence | Validator guidance document | Markdown in XRPL-Standards | Permanent |

### Evidence Pack Integrity

Each release evidence pack is hashed (SHA-256) and the hash is published as a release artefact. This allows anyone to verify that the evidence pack has not been modified after release.

### Contributor Attribution in Evidence

All evidence artefacts must credit their contributors. Threat model contributors, security reviewers, adversarial test case authors, bug bounty participants, independent testnet testers, and anyone who provided security-relevant input during the development process is named in the evidence pack. This serves both transparency (the community can see who vouched for what) and recognition (contributors receive public credit).

---

## §6 — Gap Analysis: XLS-0056 as Case Study

The following table assesses each governance control against the evidence visible in the XLS-0056 Batch amendment development lifecycle, referencing specific PRs and behaviours.

| Control | Required by Gate | Current XRPL Evidence (XLS-0056) | Gap Severity |
|---|---|---|---|
| **Security considerations in XLS spec** | Gate 0 | Absent. The Batch spec defines ~15 preflight failure conditions but has no security considerations section, no trust boundary analysis. PR #452 added integration guidance (not security guidance) 2 days before disclosure, while already in validator voting. | **Critical** |
| **Public review period for spec** | Gate 0 | Not enforced. PR #452 was merged same-day by author. | **High** |
| **No self-merge on spec** | Gate 0 | Violated. PR #452 was authored and merged by @mvadari. | **High** |
| **Structured threat model** | Gate 1 | Absent. No threat model at any stage — spec, implementation, or amendment voting. The inner/outer transaction authentication boundary was never formally analysed. | **Critical** |
| **Community threat modelling window** | Gate 1 | Absent. No mechanism existed for community security contributions to proposed amendments. | **Critical** |
| **Evil user stories / adversarial scenarios** | Gate 1 | Absent. The exact attack — "submit a Batch where the first BatchSigner controls a non-existent account, skipping validation of subsequent signers" — is a natural output of structured misuse-case analysis. | **Critical** |
| **Independent code review (not feature team)** | Gate 2 | Absent. All Batch PRs were reviewed within a closed loop of three developers (@dangell7, @mvadari, @ximinez). No external security perspective. | **Critical** |
| **Formal GitHub approval before merge** | Gate 2 | Absent for PR #5060. All reviewers had COMMENTED status only — no formal approval recorded. PR was merged without explicit sign-off through GitHub's approval mechanism. | **High** |
| **No self-merge** | Gate 2 | Violated. PR #6069 was authored and merged by @ximinez. PR #452 was authored and merged by @mvadari. | **Critical** |
| **Reviewer concern resolution before merge** | Gate 2 | Violated. @dangell7 raised "This doesn't seem right" on PR #6069's `Transactor.cpp` change. @ximinez responded 13 days later citing a commit hash. @dangell7 approved without revisiting his concern. The concern was not formally resolved. | **High** |
| **Branch protection (no force-push, required reviewers)** | Gate 2 | Not evidenced. PRs #5060 and #6069 merged to `develop` (default branch). PR #452 merged to `master`. Whether branch protection was enabled is not publicly visible, but the self-merge pattern suggests required-reviewer enforcement was not active. | **High** |
| **Static analysis (SAST)** | Gate 3 | No evidence. No mention of SAST in PRs or CI output. | **High** |
| **Adversarial test matrix** | Gate 3 | Absent. Tests verified functional behaviour. No scenarios with non-existent signer accounts, no early-exit validation path testing, no authentication bypass attempts. The combinatorial state space was not systematically explored. | **Critical** |
| **Fuzz testing** | Gate 3 | No evidence of fuzz targets for Batch transaction parsing or validation. | **Medium** |
| **Coverage with adversarial adequacy** | Gate 3 | Misleading. PR #5060 reported 94% patch coverage — the vulnerable code was executed but only with correct-path inputs. PR #6069 reported 79.2% with "all modified lines covered." Coverage measured execution, not adversarial exploration. | **High** |
| **Testnet validation with soak period** | Gate 4 | Absent. No structured testnet validation phase existed. The amendment was not subjected to a dedicated testnet soak period before entering validator voting. | **Critical** |
| **Amendment-scoped bug bounty on testnet** | Gate 4 | Absent. The bug bounty that caught the vulnerability was running against code already in the validator voting window — one vote from irreversible mainnet activation. No dedicated pre-release bounty period existed. | **Critical** |
| **Evidence of independent tester engagement** | Gate 4 | Absent. No independent parties tested the amendment on testnet before it reached production. The vulnerability was found by an external researcher during the production voting window, not during a structured pre-release testing phase. | **Critical** |
| **Integration testing on testnet** | Gate 4 | Absent. No evidence of SDK or wallet integration testing against the amendment before it entered validator voting. | **High** |
| **Reproducible build / signed artefacts** | Gate 5 | Not assessed (build infrastructure not in scope of XLS-0056 analysis). | **Medium** |
| **SBOM** | Gate 5 | Not assessed. | **Low** |
| **Evidence pack** | Gate 5 | Absent. No compiled evidence pack for the Batch amendment. | **High** |
| **Security attestation for validators** | Gate 6 | Absent. Validators had no structured security information when voting on the Batch amendment. | **Critical** |
| **Testnet validation summary for validators** | Gate 6 | Absent. No testnet validation data was available to validators. | **Critical** |
| **Validator guidance document** | Gate 6 | Absent. No amendment-specific guidance for validators. | **High** |
| **Rollback/disable strategy** | Gate 6 | Reactive only. PR #6402 (emergency disable) was published 2 days after disclosure. No pre-planned rollback procedure existed. | **High** |

### Summary

| Severity | Count |
|---|---|
| Critical | 12 |
| High | 10 |
| Medium | 2 |
| Low | 1 |

Of the 25 controls assessed, 12 are rated Critical (control entirely absent for a consensus-level feature) and 10 are rated High (control absent or violated). The XLS-0056 Batch amendment reached the validator voting stage — one vote from irreversible mainnet activation — without passing any of the seven gates defined in this framework.

---

## §7 — Adoption Roadmap

This roadmap recognises that XRPL is an open-source project. Changes to the maintainer-controlled build pipeline (Gates 0–5) can be implemented through repository policy. Changes to the validator-side process (Gate 6) require community adoption and cannot be mandated.

### Phase 1 — Immediate (0–3 months)

**Goal:** Eliminate the highest-severity gaps with controls that GitHub and existing tooling already support.

| Action | Gate | Effort | Impact |
|---|---|---|---|
| Enable branch protection on `develop` and `master` with required reviewers (≥2 for Class A/B paths) and no force-push | Gate 2 | Low — GitHub repository settings | Prevents self-merge, enforces approval |
| Enforce GitHub approval status (not just COMMENTED) as a merge requirement | Gate 2 | Low — branch protection rule | Ensures explicit sign-off |
| Require classification label on all PRs before review begins | Gate 2 | Low — PR template + label | Routes changes to appropriate review depth |
| Add security considerations section template to XLS specification format | Gate 0 | Low — template addition | Forces spec authors to address trust boundaries |
| Document no-self-merge policy and communicate to all maintainers | Gate 2 | Low — policy document | Sets expectations |
| Publish this governance framework for community review | All | Low | Establishes the target state transparently |

### Phase 2 — Near-term (3–9 months)

**Goal:** Introduce structured security analysis, community participation in threat modelling, and testnet validation processes.

| Action | Gate | Effort | Impact |
|---|---|---|---|
| Establish community threat modelling process: public threat model template, open contribution window (14 days for Class A), contributor attribution | Gate 1 | Medium — process design, template creation, community coordination | Shifts security analysis left; leverages community expertise |
| Integrate SAST into CI pipeline using open-source tooling (e.g., CodeQL via GitHub Actions) as a default baseline | Gate 3 | Medium — CI configuration, initial triage of existing findings | Catches known vulnerability patterns automatically |
| Define adversarial test matrix template and require it for Class A/B changes | Gate 3 | Medium — template, reviewer training | Ensures adversarial path coverage |
| Create Security Reviewer role with at least 2 recognised individuals independent of the core feature team | Gate 2, 3 | Medium — community recruitment, role definition | Breaks closed review loops |
| Define testnet deployment process for amendments, including soak period requirements and monitoring metrics | Gate 4 | Medium — process design, monitoring infrastructure | Establishes the testnet validation phase |
| Create amendment-scoped bug bounty template: scope definition derived from threat model, reward tiers, engagement criteria, reporting process | Gate 4 | Medium — programme design, platform selection | Enables structured pre-release adversarial testing |
| Establish engagement tracking and reporting: participation dashboard, triage SLA, evidence collection | Gate 4 | Medium — tooling, process design | Ensures engagement thresholds are measurable and auditable |
| Publish testnet testing guides and adversarial transaction examples for first Class A amendment | Gate 4 | Medium — documentation, tooling | Lowers barrier to independent tester engagement |
| Extend bug bounty programme (or create one) to cover design-stage findings: threat model contributions, spec-level flaws, adversarial scenario discoveries | Gate 1, 4 | Medium — programme design, funding | Incentivises pre-code security analysis and testnet participation |
| Publish security attestation template for amendment voting, including testnet validation summary | Gate 6 | Low — document template | Gives validators structured information |

### Phase 3 — Target State (9–18 months)

**Goal:** Achieve full framework compliance with reproducible builds, formal evidence packs, mature community security participation, and proven testnet validation processes.

| Action | Gate | Effort | Impact |
|---|---|---|---|
| Achieve SLSA Build Level 2 for rippled binary releases (scripted build, build service provenance) | Gate 5 | High — build infrastructure changes | Verifiable build provenance |
| Publish formal evidence packs for all releases containing Class A/B amendments, including testnet validation evidence | Gate 5, 6 | Medium — process discipline, Release Manager role | Complete evidence trail for validators and community |
| Establish fuzz testing programme for consensus-critical code paths with continuous fuzzing infrastructure | Gate 3 | High — infrastructure, corpus development | Discovers edge cases that unit tests miss |
| Pilot Gate 4 (testnet validation) on next Class A amendment with full bug bounty, engagement tracking, and soak period | Gate 4 | High — full process execution, community coordination, bounty funding | Validates the testnet quality gate in practice |
| Achieve OpenSSF Scorecard targets (branch protection, CI, SAST, signed releases, dependency management) | All | Medium — incremental improvements | Third-party-verifiable security posture |
| Implement cryptographic review process for any changes to signing, key derivation, or authentication | Gate 2, 3 | Medium — specialist reviewer recruitment | Expert review for the highest-risk code |
| Publish validator guidance documents (including testnet validation summaries) for all new amendments entering voting | Gate 6 | Low per amendment — requires process discipline | Informed validator decision-making |
| Recognise and publicly attribute community security contributors annually (or per-release) | All | Low — community communication | Sustains contributor engagement |

### Adoption Metrics

Progress can be measured against these indicators:

| Metric | Phase 1 Target | Phase 3 Target |
|---|---|---|
| Self-merge rate on Class A/B PRs | 0% | 0% |
| PRs merged with COMMENTED-only reviews | 0% | 0% |
| Class A amendments with threat model | — | 100% |
| Class A amendments with adversarial test matrix | — | 100% |
| Class A amendments with completed testnet soak period | — | 100% |
| Class A amendments with bug bounty meeting engagement thresholds | — | 100% |
| Amendments with published security attestation (including testnet summary) | — | 100% |
| Community threat model contributions per Class A amendment | — | ≥ 3 contributors |
| Independent testnet testers per Class A amendment | — | ≥ 5 participants |
| OpenSSF Scorecard score | — | ≥ 7/10 |

---

## §8 — Standards Cross-Reference Matrix

| Governance Control | NIST SSDF | OWASP SAMM | SLSA | PCI DSS v4.0 | NIST CSF 2.0 | OpenSSF Scorecard |
|---|---|---|---|---|---|---|
| Security considerations in spec | PW.1.1 | Design:Security Architecture | — | 6.2.2 | ID.RA | — |
| Threat modelling | PW.1.1, PW.4.1 | Design:Threat Assessment | — | 6.2.2, 6.5.1 | ID.RA-1, ID.RA-2 | — |
| Community threat contribution | PW.1.1 | Governance:Education & Guidance | — | — | GV.RR | — |
| Independent code review | PW.7.1 | Implementation:Secure Build | — | 6.2.3.1 | PR.AA | Branch-Protection |
| No self-merge | PW.7.1, PW.7.2 | Implementation:Secure Build | — | 6.2.3.1, 6.5.4 | GV.RR | Branch-Protection |
| Formal approval before merge | PW.7.2 | Implementation:Secure Build | — | 6.5.4 | GV.RR | Branch-Protection |
| Reviewer concern resolution | PW.7.2 | Implementation:Defect Management | — | 6.5.4 | GV.RR | — |
| Static analysis (SAST) | PW.7.1, PW.8.1 | Verification:Security Testing | — | 6.2.4, 6.5.5 | DE.CM | SAST |
| Adversarial test matrix | PW.8.2 | Verification:Security Testing | — | 6.2.4, 6.5.6 | DE.CM | — |
| Fuzz testing | PW.8.2 | Verification:Security Testing | — | 6.2.4 | DE.CM | Fuzzing |
| Testnet soak period | PW.8.1 | Verification:Security Testing L3 | — | 6.2.3 | DE.CM | — |
| Amendment-scoped bug bounty | PW.8.2 | Verification:Security Testing L3 | — | 6.2.4, 11.3.1 | ID.RA-1 | Vulnerabilities |
| Independent tester engagement | PW.8.2 | Verification:Security Testing L3 | — | 6.2.4 | ID.RA-1 | Vulnerabilities |
| Integration testing on testnet | PW.8.2 | Verification:Security Testing | — | 6.2.3 | DE.CM | — |
| Reproducible build | PO.3.1, PO.3.2 | Implementation:Secure Build | Build L2 | 6.2.3 | PR.DS | — |
| Signed artefacts | PS.2.1 | Implementation:Secure Build | Build L1 | 6.2.3 | PR.DS | Signed-Releases |
| SBOM | PS.3.1 | Implementation:Secure Build | — | 6.3.2 | ID.AM | — |
| Evidence pack | PO.1.1, PW.7.2 | Governance:Strategy & Metrics | Build L2 | 6.5.1 | GV.OC | — |
| Security attestation for validators | PO.1.1 | Governance:Strategy & Metrics | — | — | GV.OC | — |
| Testnet validation summary for validators | PO.1.1 | Governance:Strategy & Metrics | — | — | GV.OC | — |
| Validator guidance | — | Governance:Education & Guidance | — | — | GV.OC | — |
| Rollback/disable strategy | PW.9.1, PW.9.2 | Operations:Incident Management | — | 6.3.3 | RS.MI | — |
| Contributor attribution/recognition | — | Governance:Education & Guidance | — | — | GV.RR | Contributors |

---

## §9 — Sources

### XRPL-Specific

- [XLS-0056 Batch Amendment — Security SDLC Analysis (Vitruvius, 2026-03-02)](https://github.com/vinylwasp/XRPL-contribs/blob/main/XLS-0056-secure-dev.md) — companion gap analysis document
- [XLS-0056 Specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0056-batch)
- [XRPL-Standards PR #452 — Integration Considerations](https://github.com/XRPLF/XRPL-Standards/pull/452)
- [rippled PR #5060 — Batch (XLS-56)](https://github.com/XRPLF/rippled/pull/5060)
- [rippled PR #6069 — fix: Inner batch transactions never have valid signatures](https://github.com/XRPLF/rippled/pull/6069)
- [XRPL Vulnerability Disclosure Report (Feb 2026)](https://xrpl.org/blog/2026/vulnerabilitydisclosurereport-bug-feb2026)

### Standards and Frameworks

- [NIST SP 800-218: Secure Software Development Framework (SSDF) v1.1](https://csrc.nist.gov/publications/detail/sp/800-218/final)
- [OWASP Software Assurance Maturity Model (SAMM) v2.0](https://owaspsamm.org/)
- [SLSA: Supply-chain Levels for Software Artifacts v1.0](https://slsa.dev/spec/v1.0/)
- [PCI DSS v4.0](https://www.pcisecuritystandards.org/document_library/)
- [NIST Cybersecurity Framework (CSF) 2.0](https://www.nist.gov/cyberframework)
- [OpenSSF Scorecard](https://securityscorecards.dev/)

### Vulnerability Classification

- [CWE-305: Authentication Bypass by Primary Weakness](https://cwe.mitre.org/data/definitions/305.html)
