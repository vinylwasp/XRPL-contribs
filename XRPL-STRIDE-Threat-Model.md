# XRPL Protocol — STRIDE Threat Model

**Date:** 2026-03-05
**Author:** Vinylwasp
**Scope:** XRP Ledger protocol and `rippled` reference implementation
**Methodology:** STRIDE (Microsoft), mapped to MITRE ATT&CK, MITRE AADAPT, and MITRE ATLAS
**Audience:** XRPL validators, XRP community, ecosystem builders
**Status:** Discussion
**Companion:** [How Secure Should the XRPL Be?](XRPL-How-secure-should-it-be.md)

---

## 1. System Decomposition

### 1.1 Components

| ID | Component | Description |
|---|---|---|
| **C1** | Consensus Engine | XRPL Consensus Protocol (RPCA): proposal, voting, validation, ledger close |
| **C2** | Transaction Engine | Preflight, preclaim, apply, invariant checks for all transaction types |
| **C3** | Cryptographic Subsystem | Key derivation, signature verification (secp256k1, Ed25519), multi-sign, hashing (SHA-512Half) |
| **C4** | Peer Network | P2P protocol, node discovery, transaction relay, proposal propagation |
| **C5** | Client API | WebSocket, JSON-RPC, gRPC interfaces for transaction submission and queries |
| **C6** | Ledger State | SHAMap state tree, ledger objects, account state, amendment status |
| **C7** | Amendment Governance | Amendment proposal, validator signalling, activation (80% / 2 weeks), irreversibility |
| **C8** | Issued Token Layer | Trust lines, MPTs, freeze/clawback, authorisation, rippling |
| **C9** | DEX and AMM | Order book, offer matching, AMM constant-product pools, auto-bridging |
| **C10** | Cross-Chain Bridge | Door accounts, witness servers, attestation quorum, claim processing |

### 1.2 Trust Boundaries

| ID | Boundary | Entities | What Crosses It |
|---|---|---|---|
| **TB1** | Client → Node | External clients → `rippled` node | Unsigned/signed transactions via WebSocket/JSON-RPC; no authentication required |
| **TB2** | Node ↔ Node | Peer `rippled` instances | Peer protocol messages (transactions, proposals, validations); TLS + node key signing |
| **TB3** | Validator ↔ Validator | Trusted validators in UNL | Consensus proposals and validation votes; trust based on UNL membership |
| **TB4** | Amendment → Protocol | Validator votes → Protocol rules | New transaction processing logic; irreversible once activated |
| **TB5** | Issuer → Holder | Token issuer → Token holder | Freeze, clawback, authorisation; issuer has power over holder balances |
| **TB6** | Signer → Account | Multi-sign members → Target account | Multi-sign authority; weight and quorum determine threshold |
| **TB7** | Witness → Bridge | Witness servers → Bridge door account | Cross-chain attestations; quorum of witnesses required for claim validation |
| **TB8** | Oracle → Consumer | Oracle provider → On-chain consumers | Price data; no on-chain accuracy verification |
| **TB9** | Delegate → Delegator | Delegate account → Delegating account | Granular permission execution; delegate acts on behalf of delegator |

### 1.3 Data Flows

```
[External Client] --TX submission--> [Public Node (C5)]
                                        |
                                   [TX Validation (C2/C3)]
                                        |
                                   [P2P Relay (C4)]
                                        |
                              [Consensus Engine (C1)]
                               /         |         \
                    [Validator A]  [Validator B]  [Validator N]
                               \         |         /
                              [Validated Ledger (C6)]
                                        |
                              [State Update Applied]
```

---

## 2. Threat Actor Profiles

Threats to the XRPL come from adversaries with varying levels of sophistication, resources, and motivation. The following tiers reflect the observed capability spectrum in the blockchain threat landscape, calibrated against real-world incidents and MITRE ATT&CK threat actor data.

### Tier 1 — Opportunistic

**Profile:** Script kiddies, low-skill criminals, automated scanners.
**Resources:** Individual or small group; public tools and exploits.
**Motivation:** Financial gain, notoriety.
**Blockchain examples:** Nomad Bridge mass exploitation (August 2022) — once the initial exploit was public, hundreds of opportunistic actors copied the transaction pattern.
**Relevant techniques:** [T1190](https://attack.mitre.org/techniques/T1190/) (public-facing app exploits), [T1078](https://attack.mitre.org/techniques/T1078/) (credential stuffing), [T1059](https://attack.mitre.org/techniques/T1059/) (scripting), [T1566](https://attack.mitre.org/techniques/T1566/) (mass phishing).
**XRPL threat level:** Low for consensus; moderate for individual account compromise and ecosystem tooling.

### Tier 2 — Targeted Criminal

**Profile:** Organised cybercrime groups with specific targets and developed tradecraft.
**Resources:** Funded teams; custom tooling; targeted reconnaissance.
**Motivation:** Financial gain at scale.
**Blockchain examples:** FIN-series groups, ransomware operators pivoting to crypto.
**Relevant techniques:** [T1566.002](https://attack.mitre.org/techniques/T1566/002/) (spear-phishing), [T1078.004](https://attack.mitre.org/techniques/T1078/004/) (cloud account compromise), [T1195](https://attack.mitre.org/techniques/T1195/) (supply chain), [T1021](https://attack.mitre.org/techniques/T1021/) (remote services).
**XRPL threat level:** Moderate. Targeted attacks on validator operators, ecosystem developers, or high-value accounts.

### Tier 3 — Organised / State-Proxy

**Profile:** Sophisticated criminal organisations or state-proxied groups with operational security discipline.
**Resources:** Custom malware, counter-forensics, long-term campaigns.
**Motivation:** Large-scale financial theft, strategic disruption.
**Blockchain examples:** Lazarus Group / TraderTraitor (DPRK) — $6.75B cumulative theft; responsible for Ronin ($625M), Bybit ($1.5B), and numerous supply chain attacks.
**Relevant techniques:** [T1195.001](https://attack.mitre.org/techniques/T1195/001/)/[.002](https://attack.mitre.org/techniques/T1195/002/) (supply chain), [T1556](https://attack.mitre.org/techniques/T1556/) (modify auth), [T1027](https://attack.mitre.org/techniques/T1027/) (obfuscation), [T1562](https://attack.mitre.org/techniques/T1562/) (impair defenses), [T1557](https://attack.mitre.org/techniques/T1557/) (adversary-in-the-middle).
**XRPL threat level:** High. The XRPL's $90B+ asset base makes it a prime target. The xrpl.js supply chain attack (April 2025) demonstrated this threat actor tier's interest in the XRPL ecosystem.

### Tier 4 — State-Directed

**Profile:** Nation-state intelligence agencies with strategic objectives.
**Resources:** Zero-day capabilities, firmware/hardware implants, BGP-level network control, long-term persistent access.
**Motivation:** Strategic intelligence, economic disruption, sanctions evasion.
**Blockchain examples:** Equation Group (NSA/TAO) capabilities; theoretical application to blockchain infrastructure.
**Relevant techniques:** [T1557](https://attack.mitre.org/techniques/T1557/) (BGP hijacking), [T1195.002](https://attack.mitre.org/techniques/T1195/002/) (supply chain at build level), [T1542](https://attack.mitre.org/techniques/T1542/) (firmware persistence), [T1556](https://attack.mitre.org/techniques/T1556/) (modify authentication).
**XRPL threat level:** Moderate-High. Direct consensus attacks are costly, but infrastructure-level attacks (BGP, supply chain, validator host compromise) are within capability.

### AI-Augmented Threats

MITRE ATLAS documents the growing intersection of AI/ML capabilities with cyber operations:

- **[AML.T0040](https://atlas.mitre.org/techniques/AML.T0040) (ML Model Inference API Access):** AI-powered vulnerability discovery in consensus code. The tool that found one major XRPL vulnerability (Cantina AI Apex) demonstrates the same capability available to adversaries.
- **[AML.T0048](https://atlas.mitre.org/techniques/AML.T0048) (Social Engineering):** Deepfake-enhanced social engineering targeting validator operators and key holders. Deepfake incidents surged 179 in Q1 2025 alone, with creation costs averaging $1.33 versus average business losses of $450K-$600K per incident.
- **[AML.T0047](https://atlas.mitre.org/techniques/AML.T0047) (ML-Assisted Cyber Intrusion):** AI-accelerated exploitation of node software vulnerabilities, including automated fuzzing and exploit generation.

---

## 3. STRIDE Analysis

### 3.1 Spoofing

Spoofing threats involve an attacker impersonating a legitimate entity to gain unauthorised access or influence.

#### S1: Validator Impersonation

| Field | Detail |
|---|---|
| **Target** | C1 (Consensus Engine), TB3 (Validator ↔ Validator) |
| **Description** | An attacker compromises a trusted validator's private key and publishes proposals or validations as that validator, influencing consensus outcomes. |
| **ATT&CK** | [T1078](https://attack.mitre.org/techniques/T1078/) (Valid Accounts), [T1552](https://attack.mitre.org/techniques/T1552/) (Unsecured Credentials) |
| **AADAPT** | [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) (Exploit Blockchain Technology Specific Vulnerabilities) |
| **Preconditions** | Validator key stored insecurely; validator host compromised |
| **Impact** | Consensus influence proportional to the compromised validator's weight in UNLs; potential to block or approve transactions |
| **Existing mitigations** | UNL trust model limits influence to trusted validators; validator keys can be rotated; ephemeral keys separate node identity from validation |
| **Gaps** | No mandatory key management standards for validators; no hardware security module (HSM) requirement; no monitoring for anomalous validation patterns |

#### S2: Account Impersonation via Key Theft

| Field | Detail |
|---|---|
| **Target** | C3 (Cryptographic Subsystem), C8 (Issued Token Layer) |
| **Description** | Attacker steals a private key or seed phrase through phishing, malware, password manager compromise, or supply chain attack, and signs transactions as the victim. |
| **ATT&CK** | [T1566](https://attack.mitre.org/techniques/T1566/) (Phishing), [T1552](https://attack.mitre.org/techniques/T1552/) (Unsecured Credentials), [T1195.001](https://attack.mitre.org/techniques/T1195/001/) (Supply Chain: Software Dependencies) |
| **ATLAS** | [AML.T0048](https://atlas.mitre.org/techniques/AML.T0048) (AI-enhanced Social Engineering) |
| **Preconditions** | Victim stores keys insecurely; victim's wallet software or dependencies compromised |
| **Impact** | Complete account takeover; fund drainage; persistent control via SetRegularKey |
| **Existing mitigations** | Multi-sign requires multiple keys; regular key rotation possible; DepositAuth limits incoming |
| **Gaps** | No ecosystem-wide key management guidance; no mandatory MFA for high-value operations; supply chain attacks on client libraries (xrpl.js, April 2025) bypass all on-chain protections |

#### S3: UNL Publisher Compromise

| Field | Detail |
|---|---|
| **Target** | C1 (Consensus Engine), C7 (Amendment Governance) |
| **Description** | Attacker compromises a UNL publisher (Ripple or XRPLF) to inject malicious validators into the recommended UNL. |
| **ATT&CK** | [T1199](https://attack.mitre.org/techniques/T1199/) (Trusted Relationship), [T1078](https://attack.mitre.org/techniques/T1078/) (Valid Accounts) |
| **Preconditions** | Access to UNL publisher's signing infrastructure |
| **Impact** | Gradual consensus influence; potential to manipulate amendment voting; network fork if UNL overlap degrades below 90% |
| **Existing mitigations** | UNL lists are signed; operators can choose custom UNLs; multiple UNL publishers exist |
| **Gaps** | High concentration of trust in two publishers (Ripple and XRPLF); no formal vetting process for validator additions is publicly documented |

#### S4: Peer Node Impersonation

| Field | Detail |
|---|---|
| **Target** | C4 (Peer Network), TB2 (Node ↔ Node) |
| **Description** | Attacker operates nodes that impersonate legitimate peers to intercept or manipulate transaction relay and consensus messages. |
| **ATT&CK** | [T1557](https://attack.mitre.org/techniques/T1557/) (Adversary-in-the-Middle), [T1584](https://attack.mitre.org/techniques/T1584/) (Compromise Infrastructure) |
| **Preconditions** | Network-level access; ability to reach target nodes |
| **Impact** | Eclipse attacks isolating nodes; selective transaction censorship; delayed transaction visibility |
| **Existing mitigations** | TLS on peer connections; node key signing; fixed peer configuration option |
| **Gaps** | Default peer discovery relies on gossip and is susceptible to Sybil flooding; no peer reputation or scoring system |

---

### 3.2 Tampering

Tampering threats involve unauthorised modification of data, code, or protocol behaviour.

#### T1: Amendment Logic Vulnerabilities

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), C7 (Amendment Governance), TB4 (Amendment → Protocol) |
| **Description** | A vulnerability in new amendment code alters transaction processing in unintended ways — authentication bypasses, incorrect state transitions, or logic errors. |
| **ATT&CK** | [T1195.002](https://attack.mitre.org/techniques/T1195/002/) (Supply Chain: Software Supply Chain), [T1556](https://attack.mitre.org/techniques/T1556/) (Modify Authentication Process) |
| **AADAPT** | [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) (Exploit Blockchain Technology Specific Vulnerabilities) |
| **Preconditions** | Vulnerable code passes review and testing; amendment activates on mainnet |
| **Impact** | Protocol-wide: every node runs the same code. Impact ranges from individual transaction failures to universal authentication bypass. |
| **Existing mitigations** | Code review by maintainers; CI test suite; 2-week voting window; invariant checks post-transaction |
| **Gaps** | No mandatory threat model before amendments; no required adversarial test matrix; no independent security review gate; no structured testnet validation phase; amendment activation is irreversible |

#### T2: Supply Chain Compromise of rippled

| Field | Detail |
|---|---|
| **Target** | All components; TB4 (Amendment → Protocol) |
| **Description** | Attacker compromises the `rippled` build pipeline, dependencies, or release process to inject malicious code into the binary distributed to validators. |
| **ATT&CK** | [T1195.001](https://attack.mitre.org/techniques/T1195/001/) (Compromise Software Dependencies), [T1195.002](https://attack.mitre.org/techniques/T1195/002/) (Compromise Software Supply Chain) |
| **Preconditions** | Access to CI/CD infrastructure, dependency repositories, or maintainer accounts |
| **Impact** | Every validator running the compromised binary is affected. Could enable consensus manipulation, key exfiltration, or backdoor access. |
| **Existing mitigations** | Open-source code (auditable); GitHub Actions CI with visible checks |
| **Gaps** | No reproducible builds; no SLSA build provenance; no SBOM published with releases; the xrpl.js incident (April 2025) demonstrates the feasibility of dependency compromise in the XRPL ecosystem |

#### T3: Transaction Data Manipulation

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), C9 (DEX and AMM) |
| **Description** | Attacker crafts transactions that exploit parsing, validation, or execution logic to achieve unintended outcomes — partial payment confusion, path manipulation, DEX front-running. |
| **ATT&CK** | [T1190](https://attack.mitre.org/techniques/T1190/) (Exploit Public-Facing Application), [T1565](https://attack.mitre.org/techniques/T1565/) (Data Manipulation) |
| **AADAPT** | [ADT3021.003](https://aadapt.mitre.org/techniques/ADT3021/003/) (Market Manipulation: Wash Trading), [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) |
| **Preconditions** | Knowledge of protocol internals; ability to submit transactions |
| **Impact** | Financial loss for individual victims; DEX price manipulation; trust line balance manipulation via rippling |
| **Existing mitigations** | Preflight validation; invariant checks; partial payment protections (`delivered_amount` field); deterministic transaction ordering within ledger |
| **Gaps** | Low-liquidity DEX pairs are vulnerable to price manipulation; no circuit breakers equivalent to TradFi exchanges; AMM sandwich attack vectors exist |

#### T4: Cross-Chain Bridge Attestation Forgery

| Field | Detail |
|---|---|
| **Target** | C10 (Cross-Chain Bridge), TB7 (Witness → Bridge) |
| **Description** | Attacker compromises sufficient witness servers to forge attestations, enabling unauthorised cross-chain transfers. |
| **ATT&CK** | [T1078](https://attack.mitre.org/techniques/T1078/) (Valid Accounts), [T1098](https://attack.mitre.org/techniques/T1098/) (Account Manipulation) |
| **Preconditions** | Compromise of witness quorum threshold |
| **Impact** | Unauthorised minting or release of bridged assets; bridge draining. Historical precedent: Ronin Bridge ($625M), Wormhole ($326M). |
| **Existing mitigations** | Quorum requirement for attestations; door account controls |
| **Gaps** | Witness server security depends on operator practices with no mandatory standards; bridge security model is only as strong as its weakest witness |

#### T5: BGP Hijacking of Validator Traffic

| Field | Detail |
|---|---|
| **Target** | C4 (Peer Network), TB2/TB3 |
| **Description** | AS-level adversary hijacks BGP routes to intercept, delay, or manipulate validator-to-validator communication. |
| **ATT&CK** | [T1557](https://attack.mitre.org/techniques/T1557/) (Adversary-in-the-Middle) |
| **Preconditions** | BGP-level network access (ISP, IX, or nation-state capability) |
| **Impact** | Consensus delays; network partitioning; targeted double-spend against isolated nodes. Documented in Bitcoin (2014), MyEtherWallet (2018), Celer Bridge (2022). |
| **Existing mitigations** | TLS on peer connections; multiple network paths |
| **Gaps** | No RPKI/ROV adoption requirements for validators; no detection of BGP anomalies affecting consensus timing |

---

### 3.3 Repudiation

Repudiation threats involve an actor denying responsibility for an action or claiming an event did not occur.

#### R1: Amendment Voting Without Accountability

| Field | Detail |
|---|---|
| **Target** | C7 (Amendment Governance) |
| **Description** | Validators vote for or against amendments without publicly documenting their rationale or the security evidence they considered. If an amendment introduces a vulnerability, no mechanism exists to determine whether validators assessed the risk or voted on trust. |
| **ATT&CK** | N/A (governance process gap) |
| **Impact** | Inability to attribute governance failures; no feedback loop for improving amendment quality |
| **Existing mitigations** | Validator votes are on-chain and attributable |
| **Gaps** | Votes are visible but rationale is not; no structured evidence pack for validators; no post-activation accountability mechanism |

#### R2: Transaction Signing Disputes

| Field | Detail |
|---|---|
| **Target** | C3 (Cryptographic Subsystem), TB6 (Signer → Account) |
| **Description** | A multi-sign participant disputes that they authorised a transaction. With shared or poorly managed keys, attribution may be contested. |
| **ATT&CK** | [T1078](https://attack.mitre.org/techniques/T1078/) (Valid Accounts) |
| **Impact** | Legal and operational disputes over transaction authorisation, particularly for institutional accounts |
| **Existing mitigations** | Cryptographic signatures are non-repudiable if keys are properly managed; on-chain transaction records are immutable |
| **Gaps** | No on-chain timestamped attestation of signer intent; no standard for proving key custody at time of signing |

---

### 3.4 Information Disclosure

Information disclosure threats involve exposure of sensitive data to unauthorised parties.

#### I1: Private Key Exfiltration via Supply Chain

| Field | Detail |
|---|---|
| **Target** | C3 (Cryptographic Subsystem) |
| **Description** | Compromised client libraries, wallet software, or dependencies exfiltrate private keys to attacker-controlled infrastructure. |
| **ATT&CK** | [T1195.001](https://attack.mitre.org/techniques/T1195/001/) (Compromise Software Dependencies), [T1552](https://attack.mitre.org/techniques/T1552/) (Unsecured Credentials) |
| **Preconditions** | Victim uses compromised library version |
| **Impact** | Mass key compromise across all users of the affected library. The xrpl.js backdoor (April 2025, CVE-2025-32965, CVSS 9.3) exfiltrated seed phrases via a `checkValidityOfSeed` function. |
| **Existing mitigations** | Rapid detection and response (xrpl.js was patched same day); npm security advisories |
| **Gaps** | No code signing for official XRPL libraries; no dependency pinning guidance; 135,000+ weekly downloads of xrpl.js means a brief compromise window affects many users |

#### I2: On-Chain Reconnaissance

| Field | Detail |
|---|---|
| **Target** | C6 (Ledger State) |
| **Description** | Blockchain data is public by design. Attackers scrape account balances, signer list configurations, trust line relationships, and transaction patterns to identify high-value targets and security weaknesses. |
| **AADAPT** | [ADT3025](https://aadapt.mitre.org/techniques/ADT3025/) (Scrape Blockchain Data) |
| **Impact** | Targeted attacks on high-value accounts; signer list analysis to identify quorum weaknesses; transaction pattern analysis for social engineering |
| **Existing mitigations** | Pseudonymity (account addresses are not directly linked to identities) |
| **Gaps** | Inherent to public ledger design; no privacy features for transaction amounts or participants; pseudonymity is weakened by exchange KYC linkages |

#### I3: Validator Infrastructure Information Leakage

| Field | Detail |
|---|---|
| **Target** | C1 (Consensus Engine), C4 (Peer Network) |
| **Description** | Validators leak infrastructure details (IP addresses, hosting providers, software versions, peer connections) that enable targeted attacks. |
| **ATT&CK** | [T1592](https://attack.mitre.org/techniques/T1592/) (Gather Victim Host Information), [T1590](https://attack.mitre.org/techniques/T1590/) (Gather Victim Network Information) |
| **Impact** | Enables targeted DoS, eclipse attacks, or infrastructure compromise |
| **Existing mitigations** | Validators can use proxy layers; peer connections use TLS |
| **Gaps** | Validator IP addresses are often discoverable via peer protocol; no standard for validator operational security |

---

### 3.5 Denial of Service

Denial of service threats aim to disrupt the availability of the XRPL network or specific accounts.

#### D1: Consensus Disruption

| Field | Detail |
|---|---|
| **Target** | C1 (Consensus Engine), TB3 (Validator ↔ Validator) |
| **Description** | Attacker disrupts consensus by taking >20% of UNL validators offline (liveness attack) or causing validation drift. |
| **ATT&CK** | [T1499](https://attack.mitre.org/techniques/T1499/) (Endpoint Denial of Service), [T1498](https://attack.mitre.org/techniques/T1498/) (Network Denial of Service) |
| **Preconditions** | Ability to disrupt validator connectivity or crash validator processes |
| **Impact** | Network halt (fail-safe — no invalid transactions). The XRPL experienced a 64-minute halt in February 2025 due to validation drift. |
| **Existing mitigations** | Negative UNL mechanism excludes offline validators from quorum; network fails safe (halts rather than processing invalid transactions) |
| **Gaps** | No coordinated resilience testing programme; no defined recovery playbook; manual validator intervention required during February 2025 halt |

#### D2: Transaction Flooding

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), C5 (Client API), TB1 (Client → Node) |
| **Description** | Attacker floods the network with valid or near-valid transactions to exhaust node resources and delay legitimate transactions. |
| **ATT&CK** | [T1499](https://attack.mitre.org/techniques/T1499/) (Endpoint DoS) |
| **Preconditions** | Sufficient XRP to pay transaction fees; multiple accounts |
| **Impact** | Degraded performance; increased latency; legitimate transaction delays |
| **Existing mitigations** | Transaction fees (destroyed, not redistributed); reserve requirements; fee escalation under load; per-account sequence numbers prevent replay |
| **Gaps** | Fee levels may be insufficient to deter well-funded attackers; no rate limiting at the protocol level beyond fee mechanics |

#### D3: Eclipse Attack on Individual Nodes

| Field | Detail |
|---|---|
| **Target** | C4 (Peer Network), TB2 (Node ↔ Node) |
| **Description** | Attacker monopolises all peer connections of a target node, controlling its view of the network. |
| **ATT&CK** | [T1557](https://attack.mitre.org/techniques/T1557/) (Adversary-in-the-Middle), [T1584](https://attack.mitre.org/techniques/T1584/) (Compromise Infrastructure) |
| **Preconditions** | Network proximity; knowledge of target's peer configuration |
| **Impact** | Targeted node sees attacker's version of the ledger; potential for localised double-spend; delayed transaction visibility for exchanges or payment processors relying on the eclipsed node |
| **Existing mitigations** | Fixed peer configuration; validator-specific connections; UNL validation provides a cross-check |
| **Gaps** | Default peer discovery is gossip-based and vulnerable; no peer diversity enforcement; no eclipse detection mechanism |

#### D4: Node Crash via Malformed Input

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), C5 (Client API) |
| **Description** | Attacker submits specially crafted transactions or peer messages that trigger crashes in `rippled`. |
| **ATT&CK** | [T1190](https://attack.mitre.org/techniques/T1190/) (Exploit Public-Facing Application) |
| **AADAPT** | [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) (Exploit Blockchain Technology Specific Vulnerabilities) |
| **Preconditions** | Knowledge of parser or validation bugs |
| **Impact** | Individual node crashes; if widespread, consensus disruption. The November 2024 PaymentChannelClaim crash affected nodes with cached objects for approximately 10 minutes. |
| **Existing mitigations** | Preflight validation; invariant checks; CI test suite |
| **Gaps** | No public fuzzing programme; no continuous fuzzing of transaction parsing and validation paths; the November 2024 crash passed all existing tests |

---

### 3.6 Elevation of Privilege

Elevation of privilege threats involve an attacker gaining capabilities beyond what they are authorised to perform.

#### E1: Authentication Bypass in Transaction Validation

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), C3 (Cryptographic Subsystem) |
| **Description** | A flaw in transaction authentication logic allows an attacker to execute transactions on behalf of accounts they do not control. |
| **ATT&CK** | [T1556](https://attack.mitre.org/techniques/T1556/) (Modify Authentication Process) |
| **AADAPT** | [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) (Exploit Blockchain Technology Specific Vulnerabilities) |
| **Preconditions** | Authentication logic bug in consensus-critical code |
| **Impact** | Universal account takeover; fund drainage; persistent compromise via SetRegularKey. CWE-305 (Authentication Bypass by Primary Weakness). |
| **Existing mitigations** | Code review; invariant checks (but invariants verify state consistency, not authentication) |
| **Gaps** | No mandatory security SAST; no adversarial test matrix for authentication paths; no independent security review gate for authentication code |

#### E2: Amendment Governance Manipulation

| Field | Detail |
|---|---|
| **Target** | C7 (Amendment Governance), TB4 (Amendment → Protocol) |
| **Description** | Attacker influences the amendment process to activate malicious or vulnerable code — through social engineering of validators, compromise of validator keys, or introduction of subtle vulnerabilities in amendment code. |
| **ATT&CK** | [T1199](https://attack.mitre.org/techniques/T1199/) (Trusted Relationship), [T1566](https://attack.mitre.org/techniques/T1566/) (Phishing) |
| **ATLAS** | [AML.T0048](https://atlas.mitre.org/techniques/AML.T0048) (AI-enhanced Social Engineering) |
| **Preconditions** | Ability to influence >80% of UNL validators' votes, or ability to introduce vulnerable code that passes review |
| **Impact** | Irreversible protocol change; cannot be rolled back without a new corrective amendment and another 2-week activation cycle |
| **Existing mitigations** | 2-week sustained supermajority required; validators can independently evaluate code; open-source codebase |
| **Gaps** | No structured security evidence provided to validators; no threat model requirement; no independent testing requirement; validators vote on trust rather than evidence |

#### E3: Issuer Privilege Abuse

| Field | Detail |
|---|---|
| **Target** | C8 (Issued Token Layer), TB5 (Issuer → Holder) |
| **Description** | A token issuer uses freeze, clawback, or authorisation controls to seize or restrict assets beyond what holders expected. |
| **ATT&CK** | [T1098](https://attack.mitre.org/techniques/T1098/) (Account Manipulation) |
| **Preconditions** | Issuer account with freeze or clawback enabled |
| **Impact** | Holder assets frozen or clawed back; trust erosion in XRPL-issued assets |
| **Existing mitigations** | NoFreeze flag (irreversible); clawback must be enabled before issuance (AllowTrustLineClawback is irreversible); holders can set NoRipple |
| **Gaps** | Holders may not understand issuer capabilities at time of acquiring tokens; no on-chain disclosure of issuer risk |

#### E4: Delegate Permission Escalation

| Field | Detail |
|---|---|
| **Target** | C2 (Transaction Engine), TB9 (Delegate → Delegator) |
| **Description** | A delegate exploits overly broad permissions or a bug in the delegation system to perform actions beyond their intended scope. |
| **ATT&CK** | [T1098](https://attack.mitre.org/techniques/T1098/) (Account Manipulation) |
| **Preconditions** | Delegation granted with excessive permissions; or bug in delegation enforcement |
| **Impact** | Unauthorised transactions from delegator's account |
| **Existing mitigations** | Granular permission types; sub-permissions limit scope; delegator can revoke |
| **Gaps** | Delegation is a new feature; no long-term adversarial testing history; no automatic expiry mechanism |

#### E5: Multi-Sign Quorum Manipulation

| Field | Detail |
|---|---|
| **Target** | C3 (Cryptographic Subsystem), TB6 (Signer → Account) |
| **Description** | Attacker exploits signer list configuration — misconfigured weights, compromised subset of signers, or (in nested multi-sign) quorum relaxation — to sign transactions with fewer authorisations than the account holder intended. |
| **ATT&CK** | [T1098](https://attack.mitre.org/techniques/T1098/) (Account Manipulation), [T1078](https://attack.mitre.org/techniques/T1078/) (Valid Accounts) |
| **Preconditions** | Quorum misconfiguration or compromise of sufficient signer keys |
| **Impact** | Transactions signed below intended security threshold; account compromise |
| **Existing mitigations** | Explicit quorum and weight configuration; maximum 32 signers; hash prefix separation between single-sign and multi-sign |
| **Gaps** | Nested multi-sign (proposed) introduces recursive validation with quorum relaxation when cycles make configured quorum unachievable — the effective quorum is silently lowered without account holder consent |

---

## 4. Threat Summary Matrix

| ID | STRIDE | Threat | Actors | Severity | ATT&CK / AADAPT |
|---|---|---|---|---|---|
| S1 | Spoofing | Validator key compromise | T3, T4 | Critical | [T1078](https://attack.mitre.org/techniques/T1078/), [T1552](https://attack.mitre.org/techniques/T1552/) |
| S2 | Spoofing | Account key theft | T1-T4 | High | [T1566](https://attack.mitre.org/techniques/T1566/), [T1552](https://attack.mitre.org/techniques/T1552/), [T1195.001](https://attack.mitre.org/techniques/T1195/001/) |
| S3 | Spoofing | UNL publisher compromise | T3, T4 | Critical | [T1199](https://attack.mitre.org/techniques/T1199/), [T1078](https://attack.mitre.org/techniques/T1078/) |
| S4 | Spoofing | Peer node impersonation | T2-T4 | Moderate | [T1557](https://attack.mitre.org/techniques/T1557/), [T1584](https://attack.mitre.org/techniques/T1584/) |
| T1 | Tampering | Amendment logic vulnerability | T2-T4 | Critical | [T1195.002](https://attack.mitre.org/techniques/T1195/002/), [T1556](https://attack.mitre.org/techniques/T1556/), [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) |
| T2 | Tampering | Supply chain compromise of rippled | T3, T4 | Critical | [T1195.001](https://attack.mitre.org/techniques/T1195/001/)/[.002](https://attack.mitre.org/techniques/T1195/002/) |
| T3 | Tampering | Transaction data manipulation | T1-T3 | Moderate-High | [T1190](https://attack.mitre.org/techniques/T1190/), [T1565](https://attack.mitre.org/techniques/T1565/), [ADT3021.003](https://aadapt.mitre.org/techniques/ADT3021/003/) |
| T4 | Tampering | Bridge attestation forgery | T2-T4 | High | [T1078](https://attack.mitre.org/techniques/T1078/), [T1098](https://attack.mitre.org/techniques/T1098/) |
| T5 | Tampering | BGP hijacking of validator traffic | T4 | High | [T1557](https://attack.mitre.org/techniques/T1557/) |
| R1 | Repudiation | Amendment voting without accountability | N/A | Moderate | N/A |
| R2 | Repudiation | Transaction signing disputes | N/A | Low-Moderate | [T1078](https://attack.mitre.org/techniques/T1078/) |
| I1 | Info Disclosure | Key exfiltration via supply chain | T2-T4 | Critical | [T1195.001](https://attack.mitre.org/techniques/T1195/001/), [T1552](https://attack.mitre.org/techniques/T1552/) |
| I2 | Info Disclosure | On-chain reconnaissance | T1-T4 | Low | [ADT3025](https://aadapt.mitre.org/techniques/ADT3025/) |
| I3 | Info Disclosure | Validator infrastructure leakage | T2-T4 | Moderate | [T1592](https://attack.mitre.org/techniques/T1592/), [T1590](https://attack.mitre.org/techniques/T1590/) |
| D1 | DoS | Consensus disruption | T3, T4 | High | [T1499](https://attack.mitre.org/techniques/T1499/) |
| D2 | DoS | Transaction flooding | T1-T3 | Moderate | [T1499](https://attack.mitre.org/techniques/T1499/) |
| D3 | DoS | Eclipse attack | T2-T4 | Moderate-High | [T1557](https://attack.mitre.org/techniques/T1557/), [T1584](https://attack.mitre.org/techniques/T1584/) |
| D4 | DoS | Node crash via malformed input | T1-T3 | Moderate-High | [T1190](https://attack.mitre.org/techniques/T1190/), [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) |
| E1 | EoP | Authentication bypass | T2-T4 | Critical | [T1556](https://attack.mitre.org/techniques/T1556/), [ADT3013](https://aadapt.mitre.org/techniques/ADT3013/) |
| E2 | EoP | Amendment governance manipulation | T3, T4 | Critical | [T1199](https://attack.mitre.org/techniques/T1199/), [T1566](https://attack.mitre.org/techniques/T1566/), [AML.T0048](https://atlas.mitre.org/techniques/AML.T0048) |
| E3 | EoP | Issuer privilege abuse | T1-T2 | Moderate | [T1098](https://attack.mitre.org/techniques/T1098/) |
| E4 | EoP | Delegate permission escalation | T1-T3 | Moderate | [T1098](https://attack.mitre.org/techniques/T1098/) |
| E5 | EoP | Multi-sign quorum manipulation | T2-T3 | High | [T1098](https://attack.mitre.org/techniques/T1098/), [T1078](https://attack.mitre.org/techniques/T1078/) |

---

## 5. Critical Threat Chains

The following attack chains combine multiple STRIDE threats into realistic end-to-end scenarios.

### Chain 1: Supply Chain → Universal Account Compromise

```
[T3 Actor: Lazarus Group]
    |
    T1566.002 (Spear-phish npm maintainer)             ATT&CK: https://attack.mitre.org/techniques/T1566/002/
    |
    T1195.001 (Compromise xrpl.js dependency)           ATT&CK: https://attack.mitre.org/techniques/T1195/001/
    |
    T1552 (Exfiltrate private keys via backdoor)    ← I1  ATT&CK: https://attack.mitre.org/techniques/T1552/
    |
    T1078 (Use stolen keys as valid accounts)        ← S2  ATT&CK: https://attack.mitre.org/techniques/T1078/
    |
    T1565 (Execute unauthorised transactions)        ← E1  ATT&CK: https://attack.mitre.org/techniques/T1565/
    |
    ADT3028.005 (Peel chain fund laundering)              AADAPT: https://aadapt.mitre.org/techniques/ADT3028/005/
```

**Precedent:** xrpl.js CVE-2025-32965 (April 2025). This chain was partially executed in the wild.

### Chain 2: Amendment Vulnerability → Protocol-Wide Exploitation

```
[T2-T3 Actor]
    |
    T1195.002 (Introduce vulnerable amendment code)       ATT&CK: https://attack.mitre.org/techniques/T1195/002/
    |
    Social engineering of validators (no evidence pack)  ← E2
    |
    Amendment activates (irreversible)                   ← T1
    |
    T1556 (Authentication bypass via logic flaw)         ← E1  ATT&CK: https://attack.mitre.org/techniques/T1556/
    |
    T1190 (Exploit via public transaction API)           ← S2  ATT&CK: https://attack.mitre.org/techniques/T1190/
    |
    Universal account compromise
```

**Precedent:** A major XRPL amendment vulnerability reached the validator voting stage — one vote from irreversible mainnet activation.

### Chain 3: Validator Infrastructure Compromise → Consensus Manipulation

```
[T4 Actor: State-directed]
    |
    T1590/T1592 (Gather validator infrastructure info)   ← I3  ATT&CK: https://attack.mitre.org/techniques/T1590/ , https://attack.mitre.org/techniques/T1592/
    |
    T1557 (BGP hijack validator communications)          ← T5  ATT&CK: https://attack.mitre.org/techniques/T1557/
    |
    OR T1078.004 (Compromise cloud hosting accounts)           ATT&CK: https://attack.mitre.org/techniques/T1078/004/
    |
    T1098 (Manipulate validator configuration)           ← S1  ATT&CK: https://attack.mitre.org/techniques/T1098/
    |
    Consensus influence / amendment voting manipulation  ← E2
```

**Precedent:** No direct XRPL precedent, but the Ronin Bridge hack ($625M) demonstrated compromise of 5 of 9 validator keys. BGP hijacking attacks against blockchain infrastructure are documented across Bitcoin and Ethereum ecosystems.

### Chain 4: Social Engineering → Persistent Account Takeover

```
[T2-T3 Actor]
    |
    AML.T0048 (Deepfake-enhanced social engineering)           ATLAS: https://atlas.mitre.org/techniques/AML.T0048
    |
    T1566 (Spear-phish key holder)                       ← S2  ATT&CK: https://attack.mitre.org/techniques/T1566/
    |
    T1552 (Obtain private key / seed)                          ATT&CK: https://attack.mitre.org/techniques/T1552/
    |
    SetRegularKey (set attacker's key on victim account) ← E1
    |
    Persistent access (survives key rotation unless
    victim specifically removes the regular key)
```

**Precedent:** Chris Larsen (Ripple chairman) lost $150 million via key compromise linked to the LastPass breach. Deepfake-enhanced social engineering incidents reached 179 in Q1 2025 alone.

---

## 6. Gaps and Recommendations Summary

| Priority | Gap | Recommendation | STRIDE Coverage |
|---|---|---|---|
| **Critical** | No mandatory security evidence for amendments | Require threat model, adversarial test matrix, and independent review before amendment enters voting | T1, E1, E2, R1 |
| **Critical** | No independent security review of consensus code | Establish independent security reviewer role outside the feature development team | T1, E1 |
| **Critical** | No supply chain security for rippled builds | Implement reproducible builds, SLSA provenance, SBOM, code signing | T2, I1 |
| **High** | No structured testnet validation | Require amendment-scoped bug bounties and soak periods on devnet before mainnet voting | T1, D4, E1 |
| **High** | No continuous fuzzing | Establish fuzzing programme for transaction parsing and validation paths | D4, T3 |
| **High** | No validator operational security standards | Publish guidance on key management, HSM usage, network security, and peer configuration | S1, I3, D1 |
| **High** | No coordinated resilience testing | Establish periodic, coordinated resilience tests for the validator network | D1, D3 |
| **Moderate** | No incident reporting framework | Define mandatory disclosure timelines and formats | R1 |
| **Moderate** | No peer diversity enforcement | Implement peer scoring and diversity requirements to resist eclipse attacks | S4, D3 |
| **Moderate** | No circuit breakers for DEX | Consider halt mechanisms for extreme price movements in low-liquidity pairs | T3 |

---

## Sources

### MITRE Frameworks
- [MITRE ATT&CK Enterprise](https://attack.mitre.org/)
- [MITRE AADAPT — Digital Asset Threat Framework](https://aadapt.mitre.org/)
- [MITRE ATLAS — AI/ML Threat Framework](https://atlas.mitre.org/)

### XRPL Protocol Documentation
- [XRPL Consensus Protocol](https://xrpl.org/docs/concepts/consensus-protocol)
- [XRPL Consensus Protections](https://xrpl.org/docs/concepts/consensus-protocol/consensus-protections)
- [XRPL Amendments](https://xrpl.org/docs/concepts/networks-and-servers/amendments)
- [XRPL Transaction Types](https://xrpl.org/docs/references/protocol/transactions/types)
- [XRPL Cryptographic Keys](https://xrpl.org/docs/concepts/accounts/cryptographic-keys)
- [XRPL Multi-Signing](https://xrpl.org/docs/concepts/accounts/multi-signing)
- [XRPL Peer Protocol](https://xrpl.org/docs/concepts/networks-and-servers/peer-protocol)
- [XRPL Invariant Checking](https://xrpl.org/docs/concepts/consensus-protocol/invariant-checking)

### Incident Data
- [xrpl.js Supply Chain Attack Disclosure (April 2025)](https://xrpl.org/blog/2025/vulnerabilitydisclosurereport-bug-apr2025)
- [Aikido Security — XRP Supply Chain Attack Analysis](https://www.aikido.dev/blog/xrp-supplychain-attack-official-npm-package-infected-with-crypto-stealing-backdoor)
- [XRPL PaymentChannelClaim Crash Disclosure (Nov 2024)](https://xrpl.org/blog/2025/vulnerabilitydisclosurereport-bug-nov2024)
- [XRPL Network Halt (Feb 2025)](https://www.theblock.co/post/338980/xrp-ledger-halt)
- [Bybit Hack Technical Analysis — NCC Group](https://www.nccgroup.com/research-blog/in-depth-technical-analysis-of-the-bybit-hack/)
- [Ronin Bridge Hack (2022)](https://limechain.tech/blog/biggest-blockchain-bridge-hacks-2022)
- [Lazarus Group Analysis — Hacken](https://hacken.io/discover/lazarus-group/)
- [North Korea $2.02B Crypto Theft 2025](https://thehackernews.com/2025/12/north-korea-linked-hackers-steal-202.html)
- [BGP Hijacking for Cryptocurrency — Secureworks](https://www.secureworks.com/research/bgp-hijacking-for-cryptocurrency-profit)
- [Eclipse Attacks on Ethereum P2P (2026)](https://arxiv.org/html/2601.16560v1)
- [Chris Larsen $150M Loss — LastPass Link](https://thehackernews.com/2025/04/ripples-xrpljs-npm-package-backdoored.html)
- [Deepfake Statistics 2025](https://deepstrike.io/blog/deepfake-statistics-2025)
- [MITRE ATLAS Deepfake KYC Case Study (Nov 2025)](https://atlas.mitre.org/)

### TradFi Security Comparison
- [Visa FY2025 Earnings](https://s1.q4cdn.com/050606653/files/doc_financials/2025/q4/Q4-2025-Earnings-Release_vF.pdf)
- [SWIFT Customer Security Programme](https://www.swift.com/myswift/customer-security-programme-csp/security-controls)
- [DTCC Operational Resilience](https://www.dtcc.com/operational-resilience)
- [SEC — Regulation SCI](https://www.sec.gov/rules-regulations/2015/12/regulation-systems-compliance-integrity)
- [SEC — ICE $10M Penalty](https://www.sec.gov/newsroom/press-releases/2024-63)
