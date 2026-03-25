# XRPL Amendment Security Governance

**Version:** 2.0
**Date:** 2026-03-25
**Author:** Chris Lethaby (Vinylwasp), with Claude Opus
**Status:** Community proposal
**Scope:** Amendment lifecycle from specification to validator activation

---

## Why This Exists

On 19 February 2026, an external researcher found a critical authentication bypass in the Batch amendment (XLS-0056) during the validator voting window, one vote from irreversible mainnet activation. The bug was a `return tesSUCCESS` instead of `continue` in a signer validation loop, a textbook CWE-305 that had survived 8 months in the codebase, two PRs, and functional testing on devnet. No threat model existed. No adversarial tests were written. No independent security reviewer looked at it. The amendment nearly activated on a ledger securing $80B+ in assets.

The [detailed analysis](https://github.com/vinylwasp/XRPL-contribs/blob/main/XLS-0056-secure-dev.md) is published separately. This document proposes what should change.

This is a community proposal, not a mandate. XRPL is an open-source project with diverse contributors: Ripple engineers, external developers like @dangell7 (Transia-RnD), community validators (Anodos Labs, Alloy Networks), the XRPL Foundation, wallet teams, and independent researchers. Any governance framework must work for all of them.

---

## What Already Works

Before proposing changes, acknowledge what exists:

- **CONTRIBUTING.md** defines a clear branch model (develop/release/master), PR expectations, XLS requirements for amendments, and code style standards
- **SECURITY.md** defines a responsible disclosure process with 24-hour triage SLA and a Ripple-sponsored bug bounty
- **clang-tidy** runs on every PR and daily against develop/release branches
- **QE labels** (`QE test required`, `QE test desired`, `QE signed off`) exist for quality engineering triage
- **Discussion #5379** (bthomee, April 2025) proposes formalising the release process into quarterly releases with frequent patches
- **Amendment gating** (`Supported::no` / `Supported::yes`) provides a mechanism to include code without activating it
- **Code coverage** is tracked on PRs via Codecov

The gap is not tooling or process structure. It is security-specific process: no threat modelling, no adversarial testing requirement, no security review role, no structured evidence for validators, and no gate between "code merged" and "amendment enters voting."

---

## The XRPL Amendment Problem

XRPL amendments are unique in software:

1. **Irreversible.** Once 80% of validators vote YES for two consecutive weeks, the amendment activates permanently. There is no rollback. The only remedy is a new amendment that undoes the change (requiring another 80% vote).

2. **Decentralised activation.** No single entity controls activation. Each validator operator makes a sovereign voting decision. This framework cannot mandate validator behaviour; it can only ensure they have the evidence to make informed decisions.

3. **Two-phase release.** Code is built centrally (maintainer team merges PRs, builds binaries). Activation is decentralised (validators vote). The security gap sits between these two phases: code ships in a binary, validators see it's available, but no structured security evidence tells them whether it's safe to activate.

4. **Diverse contributors.** The Batch amendment's primary author (@dangell7) is an external contributor from Transia-RnD. The fix PR was by a Ripple core developer (@ximinez). The spec update was by another core developer (@mvadari). The bug was found by an external researcher (Pranamya Keshkamat) and an AI tool (Cantina Apex). The development community is not a single team with a single process.

Any governance framework must respect these realities.

---

## Proposal: Six Gates for Amendments

Six checkpoints from specification to validator voting. Each gate defines what evidence must exist. The gates apply to **amendment code** (consensus-affecting changes). Non-amendment changes (bug fixes, refactors, dependency updates) continue with the existing PR review process.

```
XLS Spec ──> Threat ──> Code ──> Verification ──> Testnet ──> Validator
  Draft       Model     Review                    Validation   Evidence
 (Gate 0)   (Gate 1)  (Gate 2)    (Gate 3)       (Gate 4)    (Gate 5)
```

### Gate 0: Specification includes security considerations

**What changes:** XLS specifications for amendments must include a "Security Considerations" section before the spec leaves Draft status. This section identifies:

- Trust boundaries (which components trust which inputs)
- Authentication and authorisation model (who can do what, and how is it verified)
- Abuse scenarios the author has considered

**What this would have caught:** The Batch spec defined ~15 preflight failure conditions but had no security considerations section. The inner/outer transaction authentication boundary (who validates inner transaction signers, and what happens when validation short-circuits) was never formally described.

**How to implement:** Add a section template to the XLS specification format in XRPL-Standards. Spec reviewers check for it before approving.

**Effort:** Low. Template addition + reviewer awareness.

### Gate 1: Threat model exists and is open to community contribution

**What changes:** Before implementation begins on a Class A amendment (consensus/cryptographic/authentication), a structured threat analysis is published as a public document (GitHub issue, spec addendum, or dedicated file). The threat model is open for community contribution for a minimum of 14 days.

Anyone can contribute: security researchers, competing implementation teams, academics, community members. Contributions are publicly attributed.

The threat model uses any recognised methodology (STRIDE, attack trees, misuse cases) applied to the trust boundaries identified in Gate 0. It must include "evil user stories" written from the attacker's perspective.

**What this would have caught:** "As an attacker, I submit a Batch where the first BatchSigner controls a non-existent account, followed by signers for victim accounts with forged signatures." This is the exact vulnerability that shipped. It is a natural output of STRIDE spoofing analysis on the signer validation loop.

**How to implement:** Publish a threat model template. The spec author or a designated facilitator initiates the process. Community contributions are tracked in the public document. The XRPL Foundation or Ripple could offer small grants ($500-$1,000) for substantive threat model contributions.

**Effort:** Medium. Process design, template, community coordination.

### Gate 2: Code review with separation of duties

**What changes:** For amendment PRs:

1. **No self-merge.** The PR author cannot merge their own PR. *(XLS-0056: PR #6069 was authored and merged by @ximinez.)*
2. **At least one independent reviewer.** For consensus-affecting code, at least one reviewer must be outside the immediate feature team. *(XLS-0056: all Batch PRs were reviewed within a closed loop of three developers.)*
3. **Formal approval required.** GitHub "Approved" status (not just "Commented") before merge. *(XLS-0056: PR #5060 was merged with all 4 reviewers in COMMENTED status.)*
4. **Reviewer concerns block merge.** Unresolved concerns must be documented and addressed. *(XLS-0056: @dangell7's "This doesn't seem right" was not resolved before merge.)*

**How to implement:** GitHub branch protection rules on `develop`. These are repository settings, not code changes. Discussion #5379 already proposes branch model changes; this adds review requirements.

**Effort:** Low. GitHub configuration + maintainer policy.

### Gate 3: Adversarial testing, not just functional testing

**What changes:** Amendment PRs must include:

- **Adversarial test cases** derived from the threat model (Gate 1). These test what an attacker would try, not what the feature should do.
- **Static analysis** using security-focused tooling (Semgrep, CodeQL, or equivalent) in addition to the existing clang-tidy. Custom rules for authentication/authorisation patterns are particularly valuable.
- **Combinatorial edge-case coverage** for state spaces identified in the threat model.

Code coverage alone is insufficient. PR #5060 had 94% patch coverage. The vulnerable code was executed. The logic was wrong.

**What this would have caught:** A test matrix of {existent account, non-existent account} x {valid key, invalid key} x {first signer, middle signer, last signer} produces 18 cases. The bug triggers on any test with a non-existent matching-key signer followed by any other signer.

**How to implement:**
- Add a security SAST tool to CI (CodeQL via GitHub Actions is free for open-source). This complements clang-tidy, which catches code quality issues but not authentication logic flaws.
- Publish an adversarial test template for amendment PRs.
- The existing `QE test required` label could be extended to `Security test required` for amendment PRs.

**Effort:** Medium. CI configuration, initial triage, template, reviewer training.

### Gate 4: Testnet validation with independent scrutiny

**What changes:** Before the binary containing an amendment is released to production validators, the amendment must be:

1. **Deployed to a test network** with the amendment force-enabled
2. **Subjected to a bug bounty** specifically scoped to the amendment's attack surface
3. **Tested by independent parties** (not just the feature team)
4. **Soaked for a minimum period** (28 days for consensus-affecting amendments)

**The testnet problem:** XRPL's current test network architecture has a gap:

- **Testnet** mirrors mainnet amendment status (follows, does not lead). It cannot serve as a pre-activation validation environment.
- **Devnet** is a development environment with force-enabled amendments and periodic resets. It is not structured for sustained security testing.

**Proposed solution:** Use devnet as the testnet validation environment for amendments, with the following additions:

- Stabilise devnet resets during the soak period (no resets while a bounty is active)
- Publish clear "how to test this amendment on devnet" guides for researchers
- Run amendment-scoped bug bounties on devnet before releasing the binary

Alternatively, create a dedicated "staging net" (similar to Ethereum's Sepolia) for pre-activation testing. This is higher effort but provides a clean separation.

**What this would have caught:** The Batch vulnerability was exploitable on devnet for 4-8 months. A single adversarial transaction would have triggered it. The Anodos Labs testing on devnet was functional only (and declared the amendment "technically sound and ready for mainnet"). No adversarial transactions were submitted by anyone.

**Evidence of independent engagement is critical.** "We ran a bounty and nobody submitted" is not evidence of security. It is evidence of non-engagement. The gate should require affirmative evidence that independent parties actively tested the amendment:

- Minimum 5 unique researchers registered or submitted reports
- At least 2 independent testing reports
- At least 1 security researcher attestation

When engagement thresholds are not met, extend the bounty period, increase rewards, and document the outcome honestly.

**Bug bounty economics:** The existing Ripple-sponsored bug bounty (SECURITY.md) covers rippled and client libraries. For amendment-scoped bounties:

- Scope derived from the Gate 1 threat model
- Reward tiers from informational ($500) to critical ($100,000+)
- An "informational / participation" tier incentivises researchers to engage and document their methodology even when they find nothing
- Funding source: Ripple, XRPL Foundation, or community grants (this needs ecosystem buy-in)

**Effort:** High. Process design, bounty programme, community coordination, infrastructure.

### Gate 5: Structured evidence for validators

**What changes:** When an amendment enters the validator voting window, a structured evidence pack is published alongside it:

- **Security attestation summary:** What the amendment does, what security analysis was performed, what threats were identified and mitigated, what residual risks exist
- **Threat model:** The full community-contributed threat model from Gate 1
- **Testnet validation summary:** Soak period, bounty engagement, findings, resolution status
- **Validator guidance:** What to test on your own infrastructure, what to monitor after activation, how to disable if issues are found

This is not a centralised approval gate. Validator voting is sovereign. This gate ensures validators have the information they need to make an informed decision. When a validator votes YES, they can point to the evidence pack and say "I reviewed this." When they vote NO, they can point to specific gaps.

**What this would have changed:** Validators voting on the Batch amendment had no structured security information. No threat model. No testnet validation summary. No bounty results. They were voting based on the spec, the code, and community sentiment. The evidence pack gives them something concrete.

**How to implement:** Publish evidence packs in the XRPL-Standards repository alongside XLS specs. Announce through validator communication channels.

**Effort:** Low per amendment (once the process exists). Medium to establish the process.

---

## Change Classification

Not every change needs all six gates. Amendments get the full treatment. Other changes get proportional scrutiny:

| Class | Description | Required Gates |
|-------|-------------|---------------|
| **A. Consensus / Cryptographic** | Transaction validation, signing, key handling, amendment activation | All 6 |
| **B. Protocol / Ledger State** | New ledger objects, protocol fields, serialisation, network messages | 0, 1 (scoped), 2, 3, 4, 5 |
| **C. Client / Integration** | SDK changes, tooling, documentation | 0, 2, 3 |
| **D. Maintenance** | Bug fixes, dependency updates, refactoring | 2, 3 |

The PR author proposes a classification. A maintainer (not the author) confirms it. If a Class C/D change touches consensus-critical paths (`src/ripple/app/tx/`, `src/ripple/consensus/`, `src/ripple/protocol/`), it escalates to Class B minimum. Any change modifying authentication/signing logic is Class A.

---

## Separation of Duties

Four rules for amendment PRs. Non-negotiable.

| Rule | What it prevents | XLS-0056 violation |
|------|-----------------|-------------------|
| No self-merge | Author bias | PR #6069: authored and merged by @ximinez |
| Independent reviewer | Closed review loops | All Batch PRs reviewed by same 3 developers |
| Formal approval before merge | Implicit sign-off | PR #5060: 29 reviews, all COMMENTED, zero APPROVED |
| Unresolved concerns block merge | Dismissed warnings | @dangell7's "This doesn't seem right" not resolved |

These are GitHub branch protection settings. They cost nothing to enable and would have prevented three of the four process failures in XLS-0056.

---

## XLS-0056 Gap Analysis

Every control in this framework assessed against the Batch amendment's actual development lifecycle:

| Control | Gate | Evidence in XLS-0056 | Severity |
|---------|------|---------------------|----------|
| Security considerations in spec | 0 | Absent | Critical |
| Threat model | 1 | Absent | Critical |
| Community threat contribution window | 1 | Absent | Critical |
| Evil user stories | 1 | Absent | Critical |
| Independent code reviewer | 2 | Absent (closed 3-person loop) | Critical |
| Formal GitHub approval | 2 | Absent (PR #5060: all COMMENTED) | High |
| No self-merge | 2 | Violated (PR #6069, PR #452) | Critical |
| Reviewer concern resolution | 2 | Violated ("This doesn't seem right") | High |
| Security-focused SAST | 3 | Absent (clang-tidy only) | High |
| Adversarial test matrix | 3 | Absent | Critical |
| Testnet validation with soak period | 4 | Absent | Critical |
| Amendment-scoped bug bounty | 4 | Absent (bounty ran in voting window) | Critical |
| Independent tester engagement | 4 | Absent | Critical |
| Security attestation for validators | 5 | Absent | Critical |
| Validator guidance document | 5 | Absent | High |

**12 Critical, 3 High.** The amendment reached validator voting (one vote from irreversible activation) without passing any gate.

---

## Adoption Roadmap

### Phase 1: Now (0-3 months)

Things that can be done with GitHub settings and maintainer agreement:

| Action | Gate | Effort |
|--------|------|--------|
| Enable branch protection on `develop`: require PR, 2 approvers for amendment paths, no force-push | 2 | GitHub settings |
| Enforce APPROVED status (not COMMENTED) as merge requirement | 2 | GitHub settings |
| Require classification label on amendment PRs | 2 | PR template |
| Add security considerations template to XLS spec format | 0 | Template addition |
| Document no-self-merge policy | 2 | Policy document |
| Publish this framework for community review | All | Community discussion |

### Phase 2: Near-term (3-9 months)

Process additions requiring community coordination:

| Action | Gate | Effort |
|--------|------|--------|
| Community threat modelling process: template, open contribution window, attribution | 1 | Medium |
| Add CodeQL or equivalent to CI alongside clang-tidy | 3 | Medium |
| Adversarial test matrix template for amendment PRs | 3 | Medium |
| Recruit 2+ Security Reviewers independent of core feature team | 2, 3 | Medium |
| Define devnet soak process for amendments | 4 | Medium |
| Amendment-scoped bug bounty template and first pilot | 4 | Medium |
| Validator evidence pack template | 5 | Low |

### Phase 3: Target state (9-18 months)

Full process maturity:

| Action | Gate | Effort |
|--------|------|--------|
| SLSA Build Level 2 for rippled binaries | 5 | High |
| Formal evidence packs for all amendment releases | 5 | Medium |
| Continuous fuzzing for consensus-critical paths | 3 | High |
| Pilot full Gate 4 on next Class A amendment | 4 | High |
| OpenSSF Scorecard targets | All | Medium |
| Annual security contributor recognition | All | Low |

---

## How This Relates to Discussion #5379

@bthomee's release process proposal (April 2025) addresses release mechanics: quarterly versions, patch releases, automated tagging, branch management. This framework addresses security process: what evidence must exist before code reaches validators. They are complementary:

- #5379 defines **how** code moves through branches to become a release
- This framework defines **what security evidence** must accompany that code

The two can be adopted independently or together. Phase 1 of this framework (branch protection, review requirements) aligns naturally with #5379's branch management proposals.

---

## Standards Alignment

| Control | NIST SSDF (SP 800-218) | OWASP SAMM | OpenSSF Scorecard |
|---------|----------------------|------------|-------------------|
| Security in spec | PW.1.1 | Design:Security Architecture | N/A |
| Threat modelling | PW.1.1, PW.4.1 | Design:Threat Assessment | N/A |
| Independent review | PW.7.1 | Implementation:Secure Build | Branch-Protection |
| No self-merge | PW.7.1, PW.7.2 | Implementation:Secure Build | Branch-Protection |
| SAST | PW.7.1, PW.8.1 | Verification:Security Testing | SAST |
| Adversarial testing | PW.8.2 | Verification:Security Testing | N/A |
| Fuzz testing | PW.8.2 | Verification:Security Testing | Fuzzing |
| Signed releases | PS.2.1 | Implementation:Secure Build | Signed-Releases |

---

## Sources

### XRPL
- [XLS-0056 Security SDLC Analysis](https://github.com/vinylwasp/XRPL-contribs/blob/main/XLS-0056-secure-dev.md)
- [XRPL Vulnerability Disclosure Report (Feb 2026)](https://xrpl.org/blog/2026/vulnerabilitydisclosurereport-bug-feb2026)
- [rippled CONTRIBUTING.md](https://github.com/XRPLF/rippled/blob/develop/CONTRIBUTING.md)
- [rippled SECURITY.md](https://github.com/XRPLF/rippled/blob/develop/SECURITY.md)
- [Discussion #5379: Revisiting the release process](https://github.com/XRPLF/rippled/discussions/5379)
- [PR #5060, Batch (XLS-56)](https://github.com/XRPLF/rippled/pull/5060)
- [PR #6069, fixBatchInnerSigs](https://github.com/XRPLF/rippled/pull/6069)

### Standards
- [NIST SP 800-218: SSDF v1.1](https://csrc.nist.gov/publications/detail/sp/800-218/final)
- [OWASP SAMM v2.0](https://owaspsamm.org/)
- [OpenSSF Scorecard](https://securityscorecards.dev/)
- [SLSA v1.0](https://slsa.dev/spec/v1.0/)
