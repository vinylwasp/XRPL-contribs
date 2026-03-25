# XRPL Security Contributions

## Why These Documents Exist

I have held XRP since 2017 and have been a quiet member of the community ever since. Professionally, I have spent 20+ years as a security architect across commodities exchanges, payment networks, investment banks, retail banks, telecoms, and government: the kind of secure SDLC, threat modelling, and architecture governance work that keeps trading systems from having very bad days.

When the XLS-0056 Batch vulnerability dropped in February 2026 (an authentication bypass one validator vote from going live on mainnet), I read the PRs, recognised the patterns, and decided to contribute rather than just observe. Self-merges on security-critical code, closed review loops, no threat model, no adversarial test cases: these are solved problems in every regulated exchange I have worked at. The XRPL just has not adopted them yet.

These documents are offered in the spirit of the XRPL itself: open, transparent, and available for anyone to fork, challenge, or improve. If even one self-merge gets blocked or one threat model gets written because of this, it was worth the effort.

## Documents

### Case Study

- **[XLS-0056 Batch Amendment: Security SDLC Analysis](XLS-0056-secure-dev.md)** - A detailed walkthrough of what went wrong in the Batch amendment development process, citing specific PRs, review behaviours, and the static analysis posture. Every factual claim has been [independently verified](https://github.com/vinylwasp/XRPL-contribs/blob/main/XLS-0056-secure-dev.md#verification-note-2026-03-25-accuracy-review) against primary sources (GitHub API, official XRPL blog).

### Security Posture and Threat Analysis

- **[How Secure Should the XRPL Be?](XRPL-How-secure-should-it-be.md)** - A narrative executive summary comparing the XRPL's security posture to TradFi infrastructure (Visa, SWIFT, DTCC, NYSE), examining the assets the protocol protects and the governance gap between where it stands and where it needs to be.

- **[XRPL STRIDE Threat Model](XRPL-STRIDE-Threat-Model.md)** - A structured STRIDE analysis of the XRPL protocol and `rippled` implementation, with threat actor profiles, 22 threats mapped to MITRE ATT&CK, AADAPT, and ATLAS frameworks, critical attack chains, and prioritised recommendations. Open for community contribution.

### Governance

- **[XRPL Amendment Security Governance](XRPL-Release-Governance.md)** - Six gates for amendments, from specification to validator voting. Designed for XRPL's unique architecture (maintainer-built code, validator-governed activation). Acknowledges existing processes (CONTRIBUTING.md, SECURITY.md, Discussion #5379) and proposes incremental additions. Community proposal, not a mandate.

- **[XRPL vs TradFi Governance Comparison](XRPL-vs-TradFi-Governance-Comparison.md)** - A structured comparison of XRPL amendment governance against release and change management practices at Visa, SWIFT, the DTCC, and NYSE/ICE.

## Context

These are a mix of analysis, threat modelling, and one concrete governance proposal. None are mandates. They represent my personal assessment based on publicly available information: PRs, amendment discussions, vulnerability disclosures, the `rippled` repository's CI pipeline, GitHub Discussions, and protocol documentation. I am not a `rippled` contributor, I do not have access to internal processes or communications, and there may be context I am missing that would change parts of this analysis. If something here is wrong, I would genuinely like to know.

The XRPL has earned its place in the financial system through technical innovation and a decade of reliable operation. The community of developers, validators, and contributors who built it deserve credit for that. These documents address process gaps, not people.

## Get in Touch

If you want to discuss any of this, disagree with something, or point out where I have got it wrong:

- X: [@vinylwasp](https://x.com/vinylwasp)
- LinkedIn: [Chris Lethaby](https://www.linkedin.com/in/chris-lethaby-3875251/)

*Chris Lethaby (Vinylwasp), March 2026*
