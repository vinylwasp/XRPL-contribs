# Governance Paradigm Comparison: Traditional Financial Services vs. XRPL

**Date:** 2026-03-05 (updated 2026-03-25)
**Author:** Chris Lethaby (Vinylwasp), with Claude Opus
**Status:** Discussion
**Companion to:** [XRPL Amendment Security Governance](XRPL-Release-Governance.md)

**Note (2026-03-25):** The rippled development process was independently analysed against the public repository activity (PRs, labels, milestones, discussions). The observable process is informal Git Flow with quarterly release aspirations (see [Discussion #5379](https://github.com/XRPLF/rippled/discussions/5379)) but no sprint cadence, no security labels, no security reviewer role, and no gate between "code merged to develop" and "amendment enters voting." The existing CONTRIBUTING.md and SECURITY.md provide structure for contributions and vulnerability reporting respectively, but neither addresses amendment security governance. This comparison should be read with that context.

---

## Executive Summary

The XRP Ledger is becoming a production financial services platform. It hosts a native asset ($XRP) with an $80B+ market capitalisation, a regulated stablecoin ($RLUSD) with $1.56B supply, a decentralised exchange (DEX), automated market makers (AMMs), and a growing portfolio of real-world assets (RWAs) including U.S. Treasury tokens, diamond-backed tokens, and MiCA-compliant euro stablecoins. The transaction types, economic activity, and risk profile are functionally equivalent to those found in traditional payments networks, trading platforms, and central securities depositories.

Yet the governance model that oversees software changes to this platform bears no structural resemblance to the governance applied to equivalent traditional financial infrastructure. In a regulated financial institution, the path from development to production traverses multiple independent control functions (security teams, change advisory boards, operational risk management, compliance, internal audit, external audit, and regulatory oversight), each with defined accountability, authority to block, and a mandate to challenge. On the XRPL, every one of these functions collapses into a single mechanism: the validator amendment vote.

This document provides a structured comparison of the two paradigms. The purpose is not to argue that the XRPL should replicate a bank's governance structure (it cannot, and in many respects should not), but to make visible the control functions that exist in traditional financial services, identify which of those functions are absent or vestigial in the XRPL model, and assess the resulting risk posture for a platform that increasingly carries financial-services-grade obligations.

---

## 1. The Traditional Financial Services Governance Model

### 1.1 The Three Lines of Defence

The dominant governance model in financial services is the Three Lines of Defence (3LoD), codified by the Institute of Internal Auditors (IIA) and adopted by virtually every financial regulator globally. The IIA updated this to the "Three Lines Model" in 2020, but the structural logic remains the same. Every regulated financial institution, from retail banks to commodities exchanges to payment networks, organises its risk governance around three independent layers of accountability.

The model's core principle is **separation of accountability**: the people who do the work, the people who oversee the work, and the people who independently verify the oversight are different people with different reporting lines, different incentives, and independent authority.

#### First Line: Operational Risk Ownership

The first line owns and manages risk. These are the teams that build, run, and operate systems. In the context of a financial platform's software lifecycle:

| Function | Responsibility | Authority |
|---|---|---|
| **Development / Engineering** | Write code, implement features, fix bugs | Propose changes; cannot approve their own work into production |
| **IT Operations** | Run production infrastructure, deploy releases, monitor systems | Execute approved changes; authority to halt deployments |
| **Security Operations (SecOps)** | Monitor for threats, respond to incidents, manage vulnerabilities | Authority to block releases with unresolved critical/high findings |
| **Infrastructure & Networks** | Manage compute, storage, networking, capacity | Authority to reject deployments that exceed capacity or violate architecture standards |
| **Change Management** | Coordinate release scheduling, impact assessment, rollback planning | Authority to approve or reject change requests based on evidence of readiness |

In a well-run financial institution, a software change cannot reach production without traversing multiple first-line gates:

1. **Peer code review**: another developer reviews the code
2. **Security review**: the security team reviews for vulnerabilities
3. **Change Advisory Board (CAB)**: a cross-functional body assesses readiness, risk, impact, and rollback capability
4. **Release management**: the release manager verifies test evidence, environment readiness, and deployment procedures
5. **Operations acceptance**: the operations team confirms they can support, monitor, and roll back the change

Each of these is a potential **block point**. Any function can halt the release with documented rationale. The developer cannot override the security team. The security team cannot override the change advisory board. The release manager cannot deploy without operations acceptance.

> **From experience:** I've sat in CAB meetings at the London Metal Exchange, a $16 trillion commodities exchange classified as a Systemically Important Financial Institution, where a single unresolved penetration test finding held up a trading platform release for weeks. I've seen the same dynamic at Macquarie Bank and Visa Europe. Nobody enjoyed it. The project sponsors were furious. But nobody questioned whether the process was right, because everyone in the room had seen what happens when you skip it. The process exists because someone, somewhere, at some point, learned the hard way.

#### Second Line: Risk Oversight and Governance

The second line provides oversight, challenge, and governance. These functions do not build or operate systems. They set standards, monitor compliance, and challenge the first line's risk decisions:

| Function | Responsibility | Authority |
|---|---|---|
| **IT Risk Management** | Define IT risk appetite, maintain risk registers, assess technology risks | Authority to escalate unaccepted risks to senior management |
| **Operational Risk Management** | Assess operational risks across the business, including technology failure, process failure, and third-party risk | Authority to mandate controls and risk mitigations |
| **Compliance** | Ensure adherence to regulatory requirements, internal policies, and industry standards | Authority to block activities that breach regulatory obligations |
| **Information Security Governance** | Set security policies, standards, and control frameworks; assess control effectiveness | Authority to mandate security controls and reject non-compliant systems |
| **Enterprise Risk Management (ERM)** | Aggregate risks across the organisation, report to the board, maintain the risk appetite framework | Sets the risk framework within which the first line operates |
| **Business Management** | Own the business outcomes; accountable for risk decisions within their domain | Accept or reject residual risk for their business area |

The second line's defining characteristic is **independence from execution**. The risk management function does not write code, does not deploy systems, and does not operate infrastructure. It assesses whether the first line's risk management is adequate and challenges it when it is not.

> **From experience:** I've worked in organisations where the second line was genuinely independent and organisations where it was a rubber stamp. The difference is night and day. When the risk team reports to the same executive as the delivery team, "challenge" becomes "negotiation" and deadlines always win. When they have a separate reporting line and genuine authority, uncomfortable questions get asked and answered before production, not after an incident. The XRPL has neither version. It has nothing.

In the context of a software release, the second line provides:

- **Risk assessment**: Is the residual risk of this change within the organisation's risk appetite?
- **Policy compliance**: Does the change process comply with internal policies and regulatory requirements?
- **Control adequacy**: Are the first-line controls (code review, security testing, change management) operating effectively?
- **Escalation**: If the first line accepts a risk the second line considers unacceptable, it escalates to senior management or the board.

#### Third Line: Independent Assurance

The third line provides independent, objective assurance that the first and second lines are functioning as intended:

| Function | Responsibility | Authority |
|---|---|---|
| **Internal Audit** | Independently assess the effectiveness of governance, risk management, and controls across all three lines | Reports directly to the Board Audit Committee; authority to examine any process, system, or decision |
| **External Audit** | Provide independent assurance on financial statements and, increasingly, on operational controls (SOC 1/2, ISAE 3402) | Reports to shareholders/regulators; authority to qualify opinions |
| **Board / Audit Committee** | Oversee the risk governance framework, challenge management, set risk appetite | Ultimate accountability for governance effectiveness |

#### External Oversight

Beyond the three lines, regulated financial institutions operate under:

| Function | Responsibility | Authority |
|---|---|---|
| **Financial regulators** (FCA, PRA, SEC, OCC, MAS, etc.) | Set prudential and conduct rules; supervise compliance; enforce sanctions | Authority to impose requirements, restrict activities, levy fines, revoke licenses |
| **Payment scheme regulators** (e.g., Bank of England for CHAPS, ECB for TARGET2) | Set operational and security standards for payment systems | Authority to mandate specific controls, audit processes, require incident reporting |
| **Industry standards bodies** | Define control frameworks (PCI DSS, SWIFT CSP, ISO 27001) | Compliance often mandated by regulators or business partners |
| **Legal frameworks** | Data protection (GDPR), operational resilience (DORA), cybersecurity (NIS2, NYDFS) | Statutory obligations with penalties for non-compliance |

### 1.2 How This Applies to Software Releases

When a traditional financial institution deploys a change to its payments engine, trading platform, or depository system, the release traverses a control chain roughly equivalent to:

```
Developer writes code
        |
        v
Peer code review (1st line - Engineering)
        |
        v
Security review / SAST / penetration test (1st line - Security)
        |
        v
Test evidence compiled (1st line - QA)
        |
        v
Change Advisory Board review (1st line - Change Management)
        |
        v
Risk assessment (2nd line - IT Risk / Operational Risk)
        |
        v
Compliance review if regulatory impact (2nd line - Compliance)
        |
        v
Release management approval (1st line - Release Manager)
        |
        v
Operations acceptance (1st line - IT Ops)
        |
        v
Production deployment
        |
        v
Post-implementation review (1st line + 2nd line)
        |
        v
Internal Audit review (3rd line - periodic)
        |
        v
Regulatory examination (External - periodic)
```

At every stage, an independent function has the authority to challenge, request evidence, or block the change. The developer cannot push to production. The security team can halt the release. The CAB can defer it. Risk management can escalate it. Compliance can block it. Internal audit can retrospectively flag control failures. The regulator can sanction the institution.

**The critical structural property is not that any one control is perfect. It is that multiple independent parties, with different incentives and different perspectives, each have the opportunity to catch what the others missed.**

> **From experience:** I've delivered security infrastructure (EDR, PAM, SIEM) into commodities exchanges and payment networks, and I've been embedded in project teams building eBanking and trading platforms. In every one of those environments, the thing that actually catches bugs is not any single brilliant review. It is the sheer number of independent eyes. The developer misses something, the security reviewer catches it. The security reviewer misses something, the pen tester finds it. The pen tester misses something, the CAB asks an awkward question. Layers are not bureaucracy. They are how you stop the thing that everyone individually missed.

---

## 2. The XRPL Governance Model

### 2.1 Architecture

The XRPL operates under a two-phase model:

**Phase A: Open-Source Development (community + maintainers)**

The XRPL is an open-source protocol. Anyone can propose an amendment, write an XLS specification, submit code, or contribute to review. The development process is, by design, permissionless: external contributors, independent researchers, competing implementation teams, and individual developers all have the right to participate.

In practice, the `XRPLF/rippled` repository is maintained by a small team, currently dominated by Ripple employees, who hold merge authority over the codebase. This creates a structural tension: the development model is architecturally open but operationally concentrated. The XLS-0056 Batch amendment illustrates this. While the original implementation was contributed by an external developer (@dangell7 from Transia-RnD), the review loop, merge decisions, and fix PRs were controlled by a small group of Ripple-affiliated maintainers. The amendment that activated the dormant vulnerability (PR #6069) was authored, reviewed, and self-merged entirely within this group.

This concentration is not inherent to the protocol's design; it reflects the current state of contributor participation and repository governance. As the contributor base grows and governance matures, the maintainer role should diversify. The Release Governance Framework is designed to support this: its gates are tool-agnostic and open to any contributor, and its separation-of-duties requirements (no self-merge, independent review) are structurally designed to prevent closed review loops regardless of who holds merge authority.

**Phase B: Decentralised Amendment Activation (Validator-governed)**

New protocol features are gated behind amendments. UNL (Unique Node List) validator operators independently decide whether to vote YES on each amendment. When ≥80% of validators vote YES continuously for two weeks, the amendment activates **irreversibly** on mainnet. This phase is genuinely decentralised; no single entity, including Ripple, controls the activation decision.

### 2.2 The Development Model: Open by Design, Concentrated in Practice

The XRPL's development model is fundamentally different from a traditional FiServ vendor relationship. In a bank, software is either built internally (proprietary) or procured from a vendor under contract (third-party). In both cases, the institution has formal governance authority over the software lifecycle, through employment relationships, contractual SLAs, vendor due diligence, and regulatory oversight of outsourcing arrangements.

The XRPL is none of these. It is an open-source protocol where:

- **Anyone can propose an amendment.** The XLS specification process is open. Any community member can draft a specification, submit it for review, and advocate for its adoption.
- **Anyone can write code.** The `XRPLF/rippled` repository accepts pull requests from any contributor. The original Batch implementation (PR #5060) was authored by an external contributor (@dangell7 from Transia-RnD), not a Ripple employee.
- **Anyone can review.** Code review, threat modelling, security analysis, and testing are open to all participants. The community-produced threat model for the NestedMultiSign amendment demonstrates that external security contributions are both possible and valuable.
- **Anyone can run a validator.** The UNL is curated, but the validator software is open-source and any operator can participate in the network.

This open model is a structural advantage over traditional FiServ development: it eliminates vendor lock-in, enables global participation, and makes the entire codebase auditable by anyone. The security researcher who found the XLS-0056 vulnerability (Pranamya Keshkamat) was not an employee, contractor, or even a known community member. They were an independent researcher who chose to examine the code. This kind of ad-hoc external scrutiny is impossible in a proprietary financial system.

**However, the current reality diverges from the architectural ideal.** While development is open in principle, it is concentrated in practice:

| Aspect | Architectural Design | Current Reality |
|---|---|---|
| **Code contribution** | Open to all | Dominated by a small group of Ripple-affiliated developers. The XLS-0056 Batch amendment involved contributions from one external developer and two Ripple engineers. |
| **Merge authority** | Repository maintainers (could be diverse) | Concentrated among Ripple employees who control the `XRPLF/rippled` repository. |
| **Code review** | Open to all contributors | In practice, reviews are conducted within a small maintainer group. XLS-0056: all Batch PRs were reviewed within a closed loop of three developers. |
| **Specification authorship** | Open via XLS process | Many XLS specifications are authored by Ripple employees. |
| **Release coordination** | Could be community-driven | Ripple coordinates binary releases. |
| **Security review** | Open to all | No formal security review function exists at all, neither from Ripple nor from the community. |
| **Threat modelling** | Open to all | Did not occur for any amendment until community members began producing them post-XLS-0056. |

This concentration is not evidence of bad faith; it reflects the historical reality that Ripple has invested the most engineering resources in the protocol. But it creates two risks:

1. **Single-team dependency.** If the security culture, review discipline, or process rigour of one organisation is inadequate, the entire protocol inherits those gaps. XLS-0056 demonstrated this: the self-merge pattern, closed review loop, and absent threat modelling were characteristics of the maintainer team's practices, and they propagated directly to the protocol.

2. **Decentralisation theatre.** If development is architecturally open but operationally closed, the governance benefits of open-source (diverse perspectives, independent challenge, adversarial scrutiny) are not realised. The protocol gets the risks of open-source (anyone can see the code, including attackers) without the full benefits (broad independent review).

The Release Governance Framework is designed to address both risks. Its gates are open to any contributor, its evidence requirements are tool-agnostic, and its separation-of-duties rules (no self-merge, independent review) are structurally designed to break closed review loops regardless of the contributor's organisational affiliation. As the contributor base diversifies and more community members participate in security review, threat modelling, and testnet testing, the concentration risk should diminish. But the framework must work in the current state, where a small maintainer team holds outsized influence, as well as in the target state.

### 2.3 The Validator as Universal Control Function

In the traditional model, at least eight independent control functions stand between development and production (code review, security, QA, change management, risk, compliance, release management, operations). On the XRPL, **all of these functions collapse into one**: the validator operator's amendment vote.

| Traditional FiServ Function | XRPL Equivalent | Assessment |
|---|---|---|
| **Peer code review** | GitHub PR review (open to any contributor, but in practice concentrated among maintainers) | Exists, but with evidenced gaps: self-merges, closed review loops, COMMENTED-only approvals (XLS-0056) |
| **Security review** | No formal equivalent | No dedicated security review function. No security team. No mandatory security gate before amendment voting. |
| **Security testing (SAST, DAST, pen test)** | No formal equivalent | No evidence of security-focused static analysis, dynamic analysis, or penetration testing in the public pipeline (XLS-0056 analysis) |
| **QA / Test evidence** | CI-based automated tests | Exists, but limited to functional testing. No adversarial test matrix, no testnet validation requirement (pre-Gate 4 framework) |
| **Change Advisory Board** | None | No cross-functional body assesses readiness, risk, or rollback capability before an amendment enters voting |
| **IT Risk Management** | None | No risk management function. No risk appetite framework. No risk register for amendments. |
| **Operational Risk Management** | None | No operational risk assessment. No impact analysis. No business continuity assessment for amendment failures. |
| **Compliance** | None | No compliance function. No regulatory mapping. No assessment of legal or regulatory implications of protocol changes. |
| **Information Security Governance** | None | No security policy framework. No control standards. No security architecture review board. |
| **Enterprise Risk Management** | None | No aggregated risk view. No risk appetite statement. No board-level risk governance. |
| **Release Management** | Informal: maintainers coordinate releases | No formal release manager role. No evidence pack requirement (pre-framework). No structured release criteria. |
| **Operations Acceptance** | **Validator amendment vote** | This is where everything lands. The validator vote is simultaneously the security review, the risk assessment, the compliance check, the operations acceptance, and the go/no-go decision. |
| **Post-implementation review** | None | No structured post-activation review. No lessons-learned process (until XLS-0056 forced one). |
| **Internal Audit** | None | No independent assurance function. |
| **External Audit** | None | No external audit of protocol governance or control effectiveness. |
| **Board / Audit Committee** | None | No governing body with oversight accountability. XRPL Foundation exists but has no governance authority over the protocol. |
| **Financial regulator** | None (for the protocol itself) | Individual validators or businesses built on XRPL may be regulated, but the protocol itself has no regulatory oversight. |
| **Industry standards compliance** | None | No PCI DSS, ISO 27001, SOC 2, or equivalent certification for the protocol or its development process. |

### 2.4 What the Validator Vote Actually Is

In the traditional model, the operations acceptance gate is the **last** control in a long chain. By the time a change reaches the operations team, it has already been:

- Code-reviewed by peers
- Security-reviewed by a dedicated team
- Tested against an adversarial matrix
- Assessed by the change advisory board
- Risk-assessed by the risk management function
- Compliance-checked
- Approved by the release manager

The operations team's acceptance is a final confirmation that they are ready to deploy and support something that has already been validated through multiple independent lenses.

On the XRPL, the validator vote is the **only** control. It is the first, last, and sole gate between development and irreversible production activation. The validator operator must simultaneously be:

- The **code reviewer**, assessing whether the code is correct and secure
- The **security reviewer**, assessing whether the amendment introduces vulnerabilities
- The **risk manager**, assessing whether the residual risk is acceptable
- The **compliance officer**, assessing whether the amendment has regulatory implications
- The **change manager**, assessing whether the amendment is ready for production
- The **operations manager**, assessing whether they can support and monitor the amendment
- The **auditor**, assessing whether the development process was adequate

No individual or organisation can perform all of these functions competently for every amendment. It is unreasonable to expect a validator operator, who may be running a validator as one responsibility among many, to independently reproduce the work of eight specialised teams.

> **From experience:** I've held GCIA, GCFW, GSNA, CISSP, and SABSA certifications. I've spent 20 years doing this. And I would not trust myself to perform all eight of those functions competently on a single amendment, under time pressure, with no structured evidence to work from. That is not false modesty. It is the reason these functions are separated in every serious financial institution on the planet. The people who designed the three lines of defence model were not being precious. They understood that no one person can hold all the context.

### 2.5 The Information Asymmetry

The problem is compounded by an information asymmetry. In a traditional financial institution, each control function receives **structured evidence** appropriate to its role:

- The security team receives SAST results, penetration test reports, and threat models
- The CAB receives change impact assessments, test summaries, and rollback plans
- Risk management receives risk assessments with likelihood and impact ratings
- Compliance receives regulatory impact assessments

On the XRPL, validators receive:

- The XLS specification (functional description: "how does this work?")
- The source code (if they choose to read it)
- Nothing else

There is no threat model. No adversarial test results. No security attestation. No risk assessment. No rollback plan. No operational guidance. Validators are asked to make an irreversible decision about activating a consensus-level change on a production financial network with no structured security evidence.

As the Release Governance Framework documents, the XLS-0056 Batch amendment reached the voting stage, one vote from irreversible activation, with zero security evidence across all six control domains. Validators were voting on trust, not evidence.

### 2.6 The Irreversibility Problem

In traditional financial services, production deployments are **reversible**. If a payments engine change causes issues, the operations team executes a rollback plan. The system returns to its previous state. Customers may experience temporary disruption, but the change is undone.

On the XRPL, amendment activation is **irreversible**. Once ≥80% of validators vote YES for two consecutive weeks, the amendment is permanently part of the protocol. There is no rollback. There is no "undo." If a critical vulnerability activates on mainnet, the only response is an emergency release that marks the amendment as unsupported and hopes enough validators upgrade quickly enough. This is a reactive, uncoordinated process that depends on the same validator operators who activated the amendment in the first place.

This irreversibility transforms the validator vote from a deployment decision into a **permanent architectural commitment**. In traditional FiServ terms, it is not analogous to a release approval; it is analogous to a board decision to permanently adopt a new technology standard with no ability to reverse the decision if it proves flawed.

Board decisions of this magnitude in a regulated institution would be supported by:

- A full risk assessment with quantified impact analysis
- Independent security assurance
- Legal and compliance review
- A documented risk appetite statement
- Regulatory notification or approval (for material changes)
- Board-level discussion with documented minutes

Validator operators make equivalent decisions with none of this support.

---

## 3. The Structural Comparison

### 3.1 Control Function Mapping

The following maps each traditional FiServ control function to its XRPL equivalent, with an assessment of the gap:

| Control Domain | Traditional FiServ | XRPL | Gap |
|---|---|---|---|
| **Development Controls** | | | |
| Secure coding standards | Defined and enforced | Informal; no published secure coding standard for rippled | Moderate |
| Code review (correctness) | Mandatory peer review with formal approval | Exists but with evidenced failures (self-merge, COMMENTED-only) | High |
| Code review (security) | Dedicated security reviewer, independent of feature team | Absent; no security review function | Critical |
| Separation of duties | Enforced by tooling and policy; author cannot approve/merge own code | Not enforced; self-merges documented on security-critical code | Critical |
| **Testing Controls** | | | |
| Functional testing | Mandatory with coverage requirements | Exists; CI-based automated tests | Low |
| Security testing (SAST) | Mandatory for security-sensitive changes | Absent in public pipeline; clang-tidy runs but is not security SAST | High |
| Adversarial / negative testing | Required for security-critical paths | Absent; no adversarial test matrix requirement | Critical |
| Penetration testing | Required before major releases | Absent; no pen testing programme | High |
| Pre-production environment testing | Mandatory staging/UAT environment with soak period | Testnet exists but no governance process requires its use (pre-Gate 4) | High |
| **Change Management Controls** | | | |
| Change classification | Formal classification by risk/impact | Absent; no classification system (pre-framework) | High |
| Change Advisory Board | Cross-functional review and approval body | Absent; no equivalent | Critical |
| Impact assessment | Required for all changes; detailed for high-risk | Absent; no impact assessment process | Critical |
| Rollback planning | Mandatory documented rollback procedure | Absent; rollback is reactive (XLS-0056: emergency PR 2 days after disclosure) | Critical |
| Release scheduling | Coordinated with operations, business, and risk | Informal; maintainer discretion | Moderate |
| **Risk Management Controls** | | | |
| IT Risk Management | Dedicated function, risk register, risk appetite framework | Absent | Critical |
| Operational Risk Management | Dedicated function, scenario analysis, loss event tracking | Absent | Critical |
| Risk appetite statement | Board-approved statement defining acceptable risk levels | Absent; no risk appetite defined for protocol changes | Critical |
| Threat modelling | Required for security-sensitive changes | Absent; no threat model for any amendment (pre-framework) | Critical |
| **Oversight Controls** | | | |
| Information Security Governance | CISO / security governance function with policy authority | Absent; no security governance function | Critical |
| Compliance | Dedicated function assessing regulatory adherence | Absent; no compliance function for protocol changes | High |
| Internal Audit | Independent assurance function reporting to board/audit committee | Absent | Critical |
| External Audit | Independent third-party assurance (SOC 2, ISAE 3402) | Absent | High |
| Board / governance body oversight | Board/committee with defined accountability for technology risk | XRPL Foundation exists but has no governance authority over protocol | Critical |
| **Regulatory Controls** | | | |
| Regulatory oversight | Prudential and conduct supervision by financial regulators | Absent for the protocol itself | Structural: inherent to decentralisation |
| Regulatory reporting | Mandatory incident reporting, material change notification | Absent | Structural |
| Industry standards compliance | PCI DSS, ISO 27001, SWIFT CSP, SOC 2, etc. | Absent | High |

### 3.2 Summary of Gaps by Severity

| Severity | Count | Characterisation |
|---|---|---|
| **Critical** | 13 | Control function entirely absent; no equivalent exists in the XRPL model |
| **High** | 7 | Control function absent or present only in a degraded form |
| **Moderate** | 2 | Control function exists informally but without governance rigour |
| **Low** | 1 | Control function exists and is reasonably effective |
| **Structural** | 2 | Gap is inherent to the decentralised model and cannot be closed by process alone |

Of the 25 control functions assessed, 13 are **entirely absent** in the XRPL model. The XRPL has one control that is reasonably effective (functional testing via CI), two that exist informally, and seven that are present only in a degraded or unevidenced form.

---

## 4. Why the Traditional Model Cannot Be Directly Replicated

The comparison above is not an argument that the XRPL should adopt a bank's governance structure. Several structural properties of the XRPL make direct replication impossible and, in some respects, undesirable:

### 4.1 No Employment Relationship or Hierarchical Authority

In a traditional institution, control functions have authority because they operate within an employment hierarchy. The security team can block a release because the CISO has authority delegated from the board, and the developer's employment depends on following the process.

On the XRPL, there is no employment relationship between the developer community, the repository maintainers, the XRPL Foundation, and validator operators. Anyone can propose an amendment. Anyone can write code. Anyone can review. No one can compel a validator to vote a particular way. No one can compel a developer to submit to a security review. Authority must come from **community norms and evidence quality**, not from hierarchy.

This is both a strength and a vulnerability. It means no single entity can capture the governance process, but it also means no single entity can enforce governance discipline. When Ripple-affiliated maintainers self-merged security-critical code (XLS-0056), there was no authority that could have prevented it. Equally, when community members raised concerns (@dangell7's "This doesn't seem right"), there was no authority to ensure those concerns were resolved before merge. The governance framework must work through community adoption and validator expectations, not through organisational hierarchy.

### 4.2 No Central Governing Body

The XRPL has no board of directors with fiduciary accountability for protocol governance. The XRPL Foundation serves an administrative and community coordination role but explicitly does not govern protocol decisions. There is no body that can define risk appetite, mandate controls, or hold individuals accountable for governance failures.

This means the second line (risk oversight) and third line (independent assurance) have no institutional home. Risk management, compliance, and audit functions require a governing body to report to and derive authority from. Without one, these functions cannot exist in their traditional form.

### 4.3 Censorship Resistance as a Design Goal

Traditional governance relies on central authority to enforce controls. A regulator can compel a bank to implement controls. A CISO can block a release. A board can dismiss a CEO.

The XRPL's decentralisation is a feature, not a bug. The absence of central authority is what makes the network censorship-resistant and sovereign. Any governance framework must operate within this constraint, providing validators with **better information and evidence** to support their decisions, not replacing their sovereign authority with centralised approval gates.

### 4.4 The Strengths of the Decentralised Model

The XRPL model has properties that traditional FiServ governance lacks:

| Property | XRPL Advantage |
|---|---|
| **No single point of failure in governance** | No individual, team, or regulator can unilaterally approve or block a protocol change. A compromised or captured governance function cannot force through a malicious change. |
| **Sovereign decision-making** | Each validator makes an independent assessment. Groupthink is possible but not structurally enforced. |
| **Public transparency** | All code, specifications, and (ideally) evidence are public. Traditional FiServ governance is opaque (internal audit reports, risk assessments, and regulatory findings are confidential). |
| **Open participation** | Anyone can propose amendments, write specifications, contribute code, review, test, and threat-model. The XLS-0056 vulnerability was found by an independent researcher, a category of contributor that cannot exist in proprietary financial systems. Traditional FiServ restricts these functions to employees and contracted auditors. |
| **No regulatory capture** | No regulator can mandate changes to the protocol for political or jurisdictional reasons. The protocol serves all users equally. |

These are real strengths. The goal is not to eliminate them but to complement them with the control functions that the decentralised model lacks.

---

## 5. The Emerging XRPL Risk Profile

### 5.1 From Cryptocurrency to Financial Infrastructure

The XRPL's risk profile has fundamentally changed. It is no longer just a cryptocurrency network; it is becoming a multi-asset financial platform:

| Asset / Function | Description | Traditional FiServ Equivalent | Approximate Value (Feb 2026) |
|---|---|---|---|
| **$XRP** | Native asset | Settlement asset (like cash in a payment system) | $80B+ market cap |
| **$RLUSD** | Ripple-issued USD stablecoin, NYDFS-regulated | E-money / payment token (like a prepaid balance) | $1.56B supply |
| **EURCV** | Société Générale EUR stablecoin, MiCA-compliant | Regulated e-money | Active on XRPL |
| **DEX** | On-ledger order book exchange | Trading venue / exchange | Active |
| **AMMs** | Automated market makers | Liquidity pools / market-making service | Active |
| **U.S. Treasury tokens** | Ondo OUSG, Guggenheim, OpenEden tokenised treasuries | Central securities depository / custodian | ~$300M |
| **Diamond-backed tokens** | Ctrl Alt / Billiton Diamond | Commodity-backed securities | ~$280M |
| **Energy tokens** | Justoken JMWH | Commodity tokens | ~$861M |
| **Payment channels** | Streaming micropayments | Payment rails | Active |
| **Escrows** | Time-locked / conditional payments | Escrow / trust services | Active |
| **Multi-signature accounts** | Shared custody with quorum requirements | Joint accounts / corporate treasury controls | Active |
| **NFTs** | Non-fungible tokens | Unique digital assets / collectibles | Active |

The platform now carries the functional characteristics of:

- A **payment network** (XRP payments, RLUSD transfers, cross-border settlement)
- A **trading venue** (DEX order books, AMM pools)
- A **central securities depository** (tokenised treasuries, RWAs)
- A **custody service** (multi-sig accounts, escrows)
- An **asset issuance platform** (stablecoins, commodity tokens)

Every one of these functions, if operated by a traditional financial institution, would be subject to dedicated regulatory oversight, mandatory security controls, independent audit, and board-level governance.

### 5.2 The Regulatory Trajectory

The regulatory environment is converging on digital assets:

| Regulation | Jurisdiction | Relevance to XRPL |
|---|---|---|
| **MiCA** (Markets in Crypto-Assets Regulation) | EU | Regulates stablecoin issuers (RLUSD, EURCV) and crypto-asset service providers. Does not regulate the protocol, but regulated entities building on XRPL will face questions about protocol governance. |
| **DORA** (Digital Operational Resilience Act) | EU | Requires financial entities to manage ICT risk, including third-party technology risk. Entities dependent on XRPL may need to demonstrate that the protocol has adequate governance. |
| **NYDFS Virtual Currency Regulations** | New York, USA | RLUSD is NYDFS-regulated. The regulator will expect operational resilience of the underlying infrastructure. |
| **Basel III / IV Crypto Asset Standards** | Global (BCBS) | Banks holding crypto assets must assess infrastructure risk. Protocol governance quality affects capital treatment. |
| **FCA Crypto Roadmap** | UK | Developing regulatory framework for crypto trading venues and stablecoin issuers. Protocol governance will be a supervisory consideration. |

The regulatory trajectory creates **second-order governance pressure**: even if no regulator directly supervises the XRPL protocol, regulated entities building on the XRPL will be required to demonstrate that the infrastructure they depend on has adequate governance. If the protocol's governance cannot satisfy regulatory scrutiny, regulated entities will face barriers to building on it.

### 5.3 The Accountability Vacuum

In a traditional FiServ institution, accountability for governance failures is clearly defined:

- The **board** is accountable to shareholders and regulators for governance effectiveness
- The **CISO** is accountable for security control adequacy
- The **CRO** is accountable for risk management effectiveness
- **Individual managers** are accountable through the Senior Managers and Certification Regime (SM&CR in the UK) or equivalent

On the XRPL:

- **Who is accountable if a vulnerable amendment activates?** No one, formally. The developers who wrote the code, whether Ripple employees, external contributors, or independent researchers, are not accountable to validators. Validators are not accountable to each other. The XRPL Foundation is not accountable for protocol governance. The repository maintainers who merged the code are not accountable to the community beyond reputational consequences.
- **Who is accountable if the development process is inadequate?** No one. There is no governance body that defines development standards or holds any contributor or maintainer to them.
- **Who is accountable for risk management?** No one. There is no risk management function and no one with a mandate to create one.

This accountability vacuum is the structural root cause of the XLS-0056 failure. The vulnerability was not caused by incompetent individuals. It was caused by the absence of governance structures that would have required threat modelling, security review, and adversarial testing. No one was responsible for ensuring these controls existed, so no one ensured they existed. The open-source development model means anyone *can* contribute these controls, but the absence of governance norms means no one *must*.

> **From experience:** I have been the person who got the phone call when something went wrong in production. I have also been the person who blocked a release and got shouted at for it, only to watch a competitor's platform go down the following month because they shipped the same class of bug. Accountability without authority is just blame. Authority without accountability is just power. The XRPL currently has neither, and that is what makes XLS-0056 a systemic issue rather than a one-off mistake.

---

## 6. Bridging the Gap: What the XRPL Can Adopt

The XRPL cannot replicate the three lines of defence model, but it can adopt specific control functions that address the most critical gaps while preserving decentralisation. The Release Governance Framework proposes mechanisms for several of these. This section maps them to the traditional control functions they substitute for.

### 6.1 First-Line Equivalents (Development Controls)

These can be implemented through repository policy, CI/CD, and maintainer discipline. They do not require centralised governance:

| Traditional Control | XRPL Mechanism | Framework Gate |
|---|---|---|
| Peer code review with formal approval | Branch protection: required reviewers, formal GitHub approval, no self-merge | Gate 2 |
| Security code review | Independent security reviewer role (community or contracted) | Gate 2 |
| Security testing (SAST) | CI-integrated SAST (CodeQL, Semgrep) | Gate 3 |
| Adversarial testing | Adversarial test matrix derived from threat model | Gate 3 |
| Pre-production environment testing | Testnet validation with soak period and bug bounty | Gate 4 |
| Change classification | PR classification system (Class A/B/C/D) with escalation rules | Gate 2 |
| Rollback planning | Documented rollback/disable strategy before amendment enters voting | Gate 6 |

### 6.2 Second-Line Equivalents (Risk Oversight)

These are harder to implement without a governing body, but can be partially addressed through structured processes and community participation:

| Traditional Control | XRPL Mechanism | Framework Gate |
|---|---|---|
| Threat modelling / risk assessment | Community threat modelling process with open contribution window | Gate 1 |
| Risk acceptance | Documented risk acceptance for unmitigated threats, signed off by maintainer who is not the spec author | Gate 1 |
| Security governance | Published security policy and control standards for the rippled development process | Framework-wide |
| Independent challenge | Community threat contributors and security reviewers with public attribution | Gates 1, 2, 3, 4 |
| Evidence-based decision-making | Evidence pack compiled from all gates, published for validator review | Gates 5, 6 |

### 6.3 Third-Line Equivalents (Independent Assurance)

Traditional third-line functions (internal audit, external audit, board oversight) have no direct equivalent in the XRPL model. However, partial substitutes exist:

| Traditional Control | XRPL Mechanism | Effectiveness |
|---|---|---|
| Internal audit | Post-incident retrospectives (reactive, not proactive); community gap analysis (e.g., the XLS-0056 SDLC analysis that produced these governance documents) | Weak; depends on community initiative, not institutional mandate |
| External audit | OpenSSF Scorecard provides automated third-party assessment of repository security practices | Partial; covers repository hygiene but not governance process effectiveness |
| Board oversight | No equivalent. The XRPL Foundation could potentially serve a governance coordination role (not authority) if mandated by the community. | Absent |

### 6.4 What Cannot Be Bridged

Some gaps are inherent to the decentralised model and cannot be closed by governance process alone:

| Gap | Reason | Risk Implication |
|---|---|---|
| **Regulatory oversight of the protocol** | No jurisdiction governs the protocol. Regulators govern entities, not decentralised networks. | The protocol has no external accountability mechanism. Governance quality depends entirely on community self-discipline. |
| **Binding accountability for governance failures** | No employment relationship, no fiduciary duty, no contractual obligation between developers, validators, and users. | When governance fails (as in XLS-0056), there is no formal accountability. Reputational incentives exist but are weaker than legal or regulatory accountability. |
| **Mandatory compliance** | No authority can compel adoption of governance controls. | The framework is voluntary. Validators can choose to ignore it. Developers can choose not to produce evidence. |
| **Enterprise Risk Management** | ERM requires a single entity with visibility across all risk domains. The XRPL has no such entity. | Risks are assessed in isolation (if at all). Interaction effects between amendments, cumulative attack surface growth, and systemic risk are not managed. |

---

## 7. The Validator as Informed Decision-Maker

### 7.1 Reframing the Validator Role

The comparison reveals that expecting validators to replicate the functions of eight traditional control teams is structurally unreasonable. The Release Governance Framework reframes the validator's role from **universal control function** to **informed decision-maker**.

In this model:

- **Gates 0-5** (Specification through Release Candidate) are the responsibility of the development ecosystem: maintainers, security reviewers, community threat contributors, testnet testers, and the release manager. These gates produce the evidence.
- **Gate 6** (Amendment Activation Evidence) provides validators with the compiled evidence pack, security attestation, testnet validation summary, and validator guidance.
- **The validator vote** becomes an informed decision based on structured evidence, not a blind trust judgment.

The validator does not need to be a security expert, a risk manager, and an auditor. They need to be able to read an evidence pack and make a judgment: *"Has this amendment been subjected to adequate scrutiny? Do I trust the evidence? Are there unresolved concerns?"*

### 7.2 The Validator Decision Framework

The Release Discussion document proposes a decision framework that converts the traditional multi-function control chain into a checklist that validators can evaluate:

```
IF (no threat model exists) THEN reject
IF (no adversarial test matrix) THEN reject
IF (self-merge on Class A/B PRs) THEN reject
IF (unresolved reviewer concerns) THEN reject
IF (no security attestation) THEN reject
IF (no testnet validation period completed) THEN reject
IF (bug bounty engagement below minimum thresholds) THEN evaluate risk
IF (unresolved critical/high bug bounty findings) THEN reject
IF (all gates passed with credible evidence) THEN evaluate technical merit
```

This is a simplified proxy for the multi-layered control chain that exists in traditional FiServ. It is not as robust, but it transforms the validator vote from an uninformed trust decision into an evidence-based governance decision.

### 7.3 Default Position

In traditional FiServ, the default position is **do not deploy** until all gates are passed. The burden of proof lies with the change proposer.

The same principle should apply to the XRPL: **in the absence of security evidence, validators should abstain or vote against.** The burden of proof lies with the amendment proposer to demonstrate that adequate scrutiny has occurred. This is not obstructionism; it is the standard that every regulated financial institution applies to production deployments.

> **From experience:** Every payments platform I've worked on had a standing rule: if the evidence pack is incomplete, the release does not ship. Full stop. No exceptions, no "we'll fix it in the next release," no "the business needs it by Friday." The rule existed because someone, years earlier, had shipped without evidence and it went badly. The XRPL hasn't had its "badly" moment yet. XLS-0056 came within one validator vote of providing one.

---

## 8. Conclusion

The XRPL is functionally equivalent to a payments network, trading venue, and securities depository. Its governance model bears no structural resemblance to the governance applied to any of those systems.

In traditional financial services, the path from development to production traverses multiple independent control functions (security, risk, compliance, change management, audit), each with defined accountability and authority to block. On the XRPL, every one of these functions collapses into a single mechanism: the validator amendment vote.

This is not a failure of validators. It is a structural design gap. Validators cannot reasonably be expected to replicate the work of eight specialised teams. The solution is not to centralise governance but to **produce the evidence that validators need to make informed decisions**, and to establish community norms that treat missing evidence as grounds for rejection.

The Release Governance Framework addresses this by:

1. **Creating the control functions** (Gates 0-4) that produce evidence analogous to what traditional first-line and second-line functions produce
2. **Packaging the evidence** (Gate 5) into a structured artefact that validators can assess
3. **Reframing the validator vote** (Gate 6) from a universal control function to an informed governance decision

The XRPL cannot have a CISO, a risk committee, or a regulator. But it can have threat models, security reviews, adversarial testing, testnet validation, and evidence packs. The three lines of defence cannot be replicated, but the **evidence those lines produce** can be, through open community participation, structured processes, and a governance framework that makes the quality of evidence visible.

The XRPL's open development model is a structural advantage that traditional FiServ cannot match: anyone can contribute, anyone can challenge, and the entire codebase is publicly auditable. But this advantage is only realised when participation is broad, review is independent, and governance norms prevent concentration from recreating the closed loops that open-source is designed to break. The current concentration of maintainer authority is a transitional state, not a permanent architecture. The governance framework is designed to work in both states, ensuring that whether the contributor base is small or large, the evidence of security scrutiny is produced, published, and available to validators.

**The question is not whether the XRPL should be governed like a bank. It is whether a platform securing $80B+ in assets, regulated stablecoins, and tokenised real-world assets should have governance that produces zero structured security evidence before irreversible protocol changes. The answer, on the evidence of XLS-0056, is clearly no.**

---

## Sources

### Governance Frameworks

- [IIA Three Lines Model (2020)](https://www.theiia.org/en/content/articles/iia-blog/2020/july/the-iias-three-lines-model-an-update-of-the-three-lines-of-defense/)
- [Basel Committee: Principles for the Sound Management of Operational Risk (BCBS 195)](https://www.bis.org/publ/bcbs195.htm)
- [FCA Senior Managers and Certification Regime (SM&CR)](https://www.fca.org.uk/firms/senior-managers-certification-regime)

### Regulatory

- [EU Markets in Crypto-Assets Regulation (MiCA)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32023R1114)
- [EU Digital Operational Resilience Act (DORA)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32022R2554)
- [NYDFS Virtual Currency Regulations (23 NYCRR Part 200)](https://www.dfs.ny.gov/virtual_currency_businesses)
- [Basel Committee: Prudential Treatment of Cryptoasset Exposures](https://www.bis.org/bcbs/publ/d545.htm)

### XRPL-Specific

- [XRPL Release Governance Framework (Vinylwasp, 2026-03-05)](XRPL-Release-Governance.md)
- [XLS-0056 Batch Amendment: Security SDLC Analysis (Vinylwasp, 2026-03-02)](XLS-0056-secure-dev.md)
- [XRPL Vulnerability Disclosure Report (Feb 2026)](https://xrpl.org/blog/2026/vulnerabilitydisclosurereport-bug-feb2026)

### Standards

- [NIST SP 800-218: Secure Software Development Framework (SSDF) v1.1](https://csrc.nist.gov/publications/detail/sp/800-218/final)
- [OWASP Software Assurance Maturity Model (SAMM) v2.0](https://owaspsamm.org/)
- [PCI DSS v4.0](https://www.pcisecuritystandards.org/document_library/)
- [ISO/IEC 27001:2022](https://www.iso.org/standard/27001)
- [OpenSSF Scorecard](https://securityscorecards.dev/)
