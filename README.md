# XRPL Contributions — Vinylwasp

## Why These Documents Exist

I've held XRP since 2017 and have been a quiet member of the community ever since — the kind who reads the occasional amendment debate, nods along to validator discussions, and never posts. Professionally, I've spent 20-odd years as a security architect across commodities exchanges, payment networks, investment banks, retail banks, telecoms, and government — being involved in (badly at times) the kind of secure SDLC, threat modelling, and architecture governance work that keeps trading systems from having very bad days.

I'm not a dev, I'm a security infrastructure guy who played around with pen testing tools and built a lot of active security defenses so I normally stay out of the XRPL amendment process and leave it to the pros in this community, but when the XLS-0056 Batch vulnerability dropped in February 2026 — an authentication bypass one validator vote from going live on mainnet — I read the PRs, recognised the patterns, and thought: *"I know exactly how this happens, and I know what stops it."* Self-merges on security-critical code, closed review loops, no threat model, no adversarial test cases — these are solved problems in every regulated exchange I've worked at. The XRPL just hasn't adopted them yet.

So after years of quietly watching from the sidelines, I decided it was time to contribute something useful rather than just shouting into the void on X. These two documents are that contribution:

- **[XLS-0056 Batch Amendment — Security SDLC Analysis](https://github.com/vinylwasp/XRPL-contribs/blob/main/XLS-0056-secure-dev.md)** — A detailed walkthrough of what went wrong in the Batch amendment development process, citing specific PRs and review behaviours, and explaining why standard security practices would have caught the bug before it shipped.

- **[XRPL Release Governance Framework](https://github.com/vinylwasp/XRPL-contribs/blob/main/XRPL-Release-Governance.md)** — A gate-based governance model designed for XRPL's unique architecture (maintainer-built code, validator-governed activation), with a gap analysis, phased adoption roadmap, and cross-references to NIST, OWASP, SLSA, and PCI DSS standards. Open, transparent, and tool-agnostic — because if anyone can write an amendment, the security process shouldn't be locked behind a paywall.

Both are proposals, not mandates. The XRPL is open-source and decentralised — these documents are offered in that spirit. Fork them, challenge them, improve them. If even one self-merge gets blocked or one threat model gets written because of this, it was worth the effort.

*— Vinylwasp, March 2026*
