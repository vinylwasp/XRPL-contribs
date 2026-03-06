# XRPL Contributions: Vinylwasp

## Why These Documents Exist

I've held XRP since 2017 and have been a quiet member of the community ever since. The kind who reads the occasional amendment debate, nods along to validator discussions, and never posts. Professionally, I've spent 20-odd years as a security architect across commodities exchanges, payment networks, investment banks, retail banks, telecoms, and government, being involved (badly at times), in the kind of secure SDLC, threat modelling, and architecture governance work that keeps trading systems from having very bad days.

Some of you will be wondering who the hell I am to come wading in here. Let me be clear: I'm not a dev, don't pretend to be. I'm a security architect who started out in operations and has worked across a wide spectrum of roles. I've spent almost my entire career in project delivery roles: either standing up new cybersecurity capabilities (EDR, PAM, SIEM, the usual alphabet soup) or embedded in project teams alongside the devs building eBanking platforms, trading systems, and payment infrastructure.

Along the way I've picked up some skills and knowledge: CISSP, SABSA Chartered Practitioner (Security Architecture), SABSA Chartered Foundation, Certificate of Cloud Security Knowledge (CSA), PCI Internal Security Assessor, and a stack of SANS/GIAC certifications (GCIA Gold with Honours, GCFW Gold, GSNA Gold, GAWN, and Security Policy and Awareness).

I normally stay out of the XRPL amendment process and leave it to the pros in this community, but when the XLS-0056 Batch vulnerability dropped in February 2026 (an authentication bypass one validator vote from going live on mainnet), I read the PRs, recognised the patterns, and thought: *"I know exactly how this happens, and I know what stops it."* Self-merges on security-critical code, closed review loops, no threat model, no adversarial test cases. These are solved problems in every regulated exchange I've worked at. The XRPL just hasn't adopted them yet.

So after years of quietly watching from the sidelines, I decided it was time to contribute something useful rather than just shouting into the void on X. These documents are that contribution:

### Case Study

- **[XLS-0056 Batch Amendment, Security SDLC Analysis](XLS-0056-secure-dev.md):** A detailed walkthrough of what went wrong in the Batch amendment development process, citing specific PRs and review behaviours, and explaining why standard security practices would have caught the bug before it shipped.

### Security Posture and Threat Analysis

- **[How Secure Should the XRPL Be?](XRPL-How-secure-should-it-be.md):** A narrative executive summary comparing the XRPL's security posture to TradFi infrastructure (Visa, SWIFT, DTCC, NYSE), examining the assets, businesses, and ambitions the protocol protects and the governance gap between where it stands and where it needs to be.

- **[XRPL STRIDE Threat Model](XRPL-STRIDE-Threat-Model.md):** A structured STRIDE analysis of the XRPL protocol and `rippled` implementation, with threat actor profiles, 22 threats mapped to MITRE ATT&CK, AADAPT, and ATLAS frameworks, critical attack chains, and a prioritised recommendations summary.

### Governance

- **[XRPL Release Governance Framework](XRPL-Release-Governance.md):** A gate-based governance model designed for XRPL's unique architecture (maintainer-built code, validator-governed activation), with a gap analysis, phased adoption roadmap, and cross-references to NIST, OWASP, SLSA, and PCI DSS standards.

- **[XRPL vs TradFi Governance Comparison](XRPL-vs-TradFi-Governance-Comparison.md):** A structured comparison of XRPL amendment governance against release and change management practices at Visa, SWIFT, the DTCC, and NYSE/ICE.

These are a mix of analysis, threat modelling, and one concrete governance proposal. None are mandates. They represent my personal assessment based on publicly available information (PRs, amendment discussions, vulnerability disclosures, and protocol documentation). I am not a `rippled` contributor, I do not have access to internal processes or communications, and there may well be context I am missing that would change parts of this analysis. If something here is wrong, I would genuinely like to know.

These documents are offered in the spirit of the XRPL itself: open, transparent, and tool-agnostic. If anyone can write an amendment, the security process shouldn't be locked behind a paywall. Fork them, challenge them, improve them. If even one self-merge gets blocked or one threat model gets written because of this, it was worth the effort.

If you want to discuss any of this, disagree with something, or point out where I've got it wrong, DM me on X: [@vinylwasp](https://x.com/vinylwasp). You can also connect with me on [LinkedIn](https://www.linkedin.com/in/chris-lethaby-3875251/).

*Chris Lethaby (Vinylwasp), March 2026*
