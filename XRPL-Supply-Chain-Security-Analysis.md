# XRPL Supply Chain Security Analysis

**Date:** 2026-03-25
**Author:** Chris Lethaby (Vinylwasp), with Claude Opus
**Audience:** XRPL validators, developers, wallet builders, ecosystem integrators
**Status:** Discussion, open for community contribution
**Companion:** [XLS-0056 Security SDLC Analysis](XLS-0056-secure-dev.md), [XRPL STRIDE Threat Model](XRPL-STRIDE-Threat-Model.md)

---

## The Problem

The XRPL ecosystem has a software supply chain that runs from developer workstations through GitHub to npm registries to validator nodes to $90B+ in on-ledger assets. In April 2025, an attacker compromised one link in that chain (a single npm maintainer's credentials) and injected a private key exfiltration backdoor into xrpl.js, the official JavaScript library with 135,000+ weekly downloads. The backdoor shipped in 5 npm versions over 12 hours before it was caught by an external security researcher.

That incident was the client library chain. The consensus node chain (rippled) has different but equally significant supply chain risks that have not been publicly analysed. This document examines both.

---

## The XRPL Supply Chain Map

### Chain 1: Consensus Node (rippled)

```
Developer workstation
  │
  ├── External contributor forks on GitHub (e.g., Transia-RnD/rippled)
  │   └── Code developed on personal infrastructure
  │
  └── Core team develops on XRPLF/rippled
      │
      v
GitHub (XRPLF/rippled)
  │
  ├── PR reviewed and merged to develop (default branch)
  ├── CI: clang-tidy, build, test (GitHub Actions)
  │
  v
C++ Dependencies (Conan package manager)
  │
  ├── openssl/3.5.5       (cryptographic foundation)
  ├── grpc/1.72.0          (inter-node communication)
  ├── secp256k1/0.7.1      (ECDSA signatures)
  ├── ed25519/2015.03      (EdDSA signatures)
  ├── libarchive/3.8.1     (data handling)
  ├── nudb/2.0.9           (database)
  ├── soci/4.0.3           (database abstraction)
  ├── zlib/1.3.1           (compression)
  ├── protobuf/6.32.1      (serialisation, build tool)
  │
  v
Binary build
  │
  ├── No pre-built binaries on GitHub releases (0 assets on 3.1.2, 3.1.1, 3.1.1-rc1)
  ├── Validators build from source or use package repositories
  │
  v
Validator operators download/build and run
```

### Chain 2: Client Libraries

```
Developer workstation
  │
  v
GitHub (XRPLF/xrpl.js, xrpl-py, xrpl4j)
  │
  v
Package registry (npm, PyPI, Maven Central)
  │
  ├── xrpl.js: 6 direct dependencies (monorepo packages)
  │   └── 39 dev dependencies
  │   └── Transitive dependency tree: hundreds of npm packages
  │
  v
Wallets, exchanges, dApps, payment integrators
  │
  v
End users' private keys and funds
```

### Chain 3: Specification

```
Spec author (anyone)
  │
  v
XRPL-Standards repository (GitHub)
  │
  ├── PR reviewed and merged
  ├── Spec defines protocol behaviour that validators implement
  │
  v
rippled implementation follows spec
  │
  v
Amendment activates on mainnet (irreversible)
```

---

## What Has Already Gone Wrong

### xrpl.js npm Compromise (April 2025)

| Aspect | Detail |
|--------|--------|
| **Attack vector** | Phishing of npm maintainer credentials (Ripple employee) |
| **Impact** | Private key exfiltration backdoor in xrpl.js versions 4.2.1-4.2.4 and 2.14.2 |
| **Detection** | External researcher (Aikido Security), 12 hours after first malicious publish |
| **Scope** | 135,000+ weekly downloads; all wallets and applications using affected versions |
| **GitHub compromised?** | No. Attack bypassed GitHub entirely, published directly to npm |
| **CVE** | CVE-2025-32965, CVSS 9.3 |
| **Root cause** | Single npm maintainer account with publish rights, no 2FA at time of compromise |

**Key lesson:** The attack targeted the weakest link (npm credentials) not the strongest (GitHub code review). The code on GitHub was never compromised. The backdoor existed only in npm, where it was published directly, bypassing all PR review processes.

**Post-incident remediation (per XRPL disclosure):**
- 2FA enabled for all npm maintainers
- Compromised user removed from all XRPL npm packages
- Review of collaborator/publish access underway
- "Strategic prevention measures" announced (automation, verification, monitoring)

### Bybit Hack (February 2025)

Not XRPL, but directly relevant: $1.5 billion stolen via a supply chain compromise of a developer workstation that led to a JavaScript injection in the signing UI. The Lazarus Group compromised a single developer's machine and used that access to modify the transaction signing interface. The attack vector is identical to what could target an xrpl.js or rippled contributor.

---

## Risk Analysis: The Single-Platform Concentration

### GitHub as Single Point of Trust

The entire XRPL development lifecycle runs through GitHub:

- **Code:** XRPLF/rippled, XRPLF/xrpl.js, XRPLF/xrpl-py, XRPLF/xrpl4j
- **Specifications:** XRPLF/XRPL-Standards
- **CI/CD:** GitHub Actions
- **Code review:** GitHub PR review
- **Issue tracking:** GitHub Issues and Discussions
- **Release tags:** GitHub Releases

This means:

1. **A GitHub account compromise gives access to everything.** A maintainer's GitHub account with write access to rippled could push directly to develop (if branch protection is not enforced), approve their own PRs, or modify CI workflows to inject build-time backdoors.

2. **GitHub Actions are the build chain.** CI workflows run on GitHub-hosted runners. A compromised workflow file could exfiltrate signing keys, inject code during build, or modify test results to suppress failures.

3. **No independent build verification.** Releases have zero binary assets (verified: 3.1.2, 3.1.1, 3.1.1-rc1 all show 0 assets on GitHub). Validators build from source, which is good for auditability but means each validator trusts their own build environment, their Conan dependency resolution, and the state of the GitHub repository at the time they pull.

4. **The XRPL-Standards repository is a trust anchor.** Specs define what the code should do. A compromised spec (a subtle change to authentication semantics, for example) could legitimise a malicious implementation that reviewers accept because "it matches the spec."

### Developer Workstation Risk

The XRPL contributor base includes:

- **Ripple employees** (presumably on corporate-managed devices with endpoint security)
- **External contributors** like @dangell7 (Transia-RnD), developing on personal infrastructure
- **Community developers** contributing patches, tests, documentation
- **Validators** who may also contribute code

There is no way to mandate endpoint security for external contributors. You cannot require a community developer to run EDR on their laptop. This is fundamental to open source.

What you can do:

- **Verify commits are signed.** CONTRIBUTING.md already requires this: "The tip of each branch must be signed." GPG or SSH signatures provide non-repudiation.
- **Review at the PR boundary, not the workstation.** The trust boundary is the PR review, not the developer's machine. A compromised workstation produces compromised code, but the code still has to survive review.
- **Never trust a single reviewer.** The xrpl.js attack bypassed GitHub entirely. The Batch vulnerability survived review. Multiple independent reviewers are the defence.

---

## Dependency Chain Risk

### rippled (C++ / Conan)

8 runtime dependencies. The critical ones:

| Dependency | Risk | Mitigation |
|-----------|------|------------|
| **openssl/3.5.5** | Cryptographic foundation. A compromised OpenSSL would undermine all XRPL security. | OpenSSL is heavily audited, widely deployed. Risk is low but impact is existential. Pin version, track CVEs. |
| **secp256k1/0.7.1** | ECDSA signature verification. A backdoored secp256k1 could forge signatures. | Bitcoin Core library, heavily reviewed. Pin version. |
| **ed25519/2015.03** | EdDSA signatures. Same risk as secp256k1. | Version from 2015 (old but stable). Consider updating to a more actively maintained implementation. |
| **grpc/1.72.0** | Inter-node communication. A compromised gRPC could enable MITM between peers. | Google-maintained, widely deployed. Pin version, track CVEs. |
| **protobuf/6.32.1** | Serialisation. Malformed protobuf handling could enable RCE. | Build tool only, not runtime. Lower risk. |

**Conan lock file:** `conan.lock` exists in the repository. This pins transitive dependencies, preventing silent dependency resolution changes. Good practice.

**No SBOM published.** Validators building from source get whatever Conan resolves. A published SBOM per release would let operators verify they are running the same dependency set.

### Client Libraries (npm / PyPI / Maven)

The xrpl.js monorepo has 6 direct dependencies (all internal packages) and 39 dev dependencies. But the transitive dependency tree (dependencies of dependencies) runs into hundreds of npm packages, each a potential attack vector.

**The npm trust problem:** npm packages are published by maintainers, not by repository owners. The xrpl.js attack proved this: GitHub was never compromised, but npm was. Any dependency in the transitive tree with a single maintainer and no 2FA is an attack vector.

---

## The Version Pinning Question

### Bleeding Edge vs Known Good

A common tension in dependency management: do you run the latest version (with the latest security patches but also the latest untested code), or do you trail by a version or two (missing patches but with more community vetting)?

For a financial protocol, the answer should be deliberate:

**For cryptographic dependencies (openssl, secp256k1, ed25519):** Pin to a specific version. Update only after the new version has been deployed in comparable production environments (Bitcoin Core, Ethereum, major TLS deployments) for a sufficient period. These libraries are the foundation. A regression in OpenSSL signature verification would be catastrophic.

**For infrastructure dependencies (grpc, protobuf, zlib):** Track security patches promptly. These libraries have large attack surfaces (network parsing, data handling) and actively discovered vulnerabilities. Trailing by more than one minor version risks running with known CVEs.

**For client library dependencies (npm ecosystem):** This is where the risk concentrates. The transitive dependency tree is large and opaque. Practical mitigations:

1. **Lock files committed to the repository.** (xrpl.js uses package-lock.json; this is correct.)
2. **Regular `npm audit` in CI.** (Check if this runs; the rippled CI has no visible equivalent for xrpl.js.)
3. **Maintain a known-good baseline.** Before updating any dependency, verify the update against the current test suite AND review the changelog for unexpected changes.
4. **Consider vendoring critical dependencies.** For cryptographic libraries, copy the source into the repository rather than relying on a package registry. This eliminates the registry as an attack vector.
5. **Establish an internal review of major dependency updates.** Not every `npm update` needs a security review, but any update to a cryptographic library, a serialisation library, or a network library should be reviewed as a Class B change minimum.

---

## What Validators Should Verify

Validators are the last line of defence. When they build and run a new rippled version:

| Check | Why | How |
|-------|-----|-----|
| **Verify the release tag signature** | Ensures the release was created by a maintainer, not an attacker | `git verify-tag 3.1.2` |
| **Build from a clean checkout** | Prevents local modifications from contaminating the build | Fresh clone, not an existing working directory |
| **Compare dependency versions** | Ensures your Conan resolution matches the lock file | `conan lock` comparison |
| **Check for new or changed dependencies** | A new dependency in a patch release is a red flag | Diff `conanfile.py` between versions |
| **Verify the binary hash matches peers** | If your build produces a different hash, something is wrong | Compare SHA-256 with other trusted validators |
| **Monitor for unexpected network behaviour** | A compromised binary may behave differently under load | Consensus metrics, error rates, resource usage |

Most validators do not do all of these. There is no published validator security checklist. This is a gap the XRPL Foundation could close with a simple document.

---

## Recommendations

### For the rippled build chain

| Action | Effort | Impact |
|--------|--------|--------|
| Publish SBOM with each release (SPDX or CycloneDX) | Low | Validators can verify dependency set |
| Publish reproducible build instructions | Medium | Independent parties can verify binary integrity |
| Publish pre-built signed binaries on GitHub Releases | Medium | Reduces variance in validator build environments |
| Add Dependabot or Renovate for automated dependency CVE alerts | Low | Already flagged: 6 vulnerabilities on osa-website, likely similar on rippled |
| Target SLSA Build Level 2 (scripted build, build service provenance) | High | Verifiable build chain |

### For the client library chain

| Action | Effort | Impact |
|--------|--------|--------|
| Enforce 2FA on all npm/PyPI/Maven maintainer accounts | Low | **Done post-xrpl.js incident** |
| Require at least 2 maintainers for any publish action | Low | Eliminates single-account compromise |
| Run `npm audit` in CI and block on critical/high findings | Low | Catches known vulnerable dependencies |
| Vendor cryptographic dependencies into the repository | Medium | Eliminates registry as attack vector for crypto |
| Publish dependency attestations per release | Medium | Consumers can verify what they are running |

### For validators

| Action | Effort | Impact |
|--------|--------|--------|
| Publish a validator security checklist | Low | Standardises build verification practices |
| Establish a binary hash comparison channel | Low | Validators can verify build consistency |
| Trail production updates by 1-2 weeks unless security-critical | Low | Lets early adopters catch issues first |
| Maintain a known-good binary alongside the latest | Low | Enables rapid rollback |

### For ecosystem integrators (wallets, exchanges, dApps)

| Action | Effort | Impact |
|--------|--------|--------|
| Pin xrpl.js/xrpl-py versions, do not auto-update | Low | Prevents supply chain compromise from propagating |
| Subscribe to xrpl-announce for security notifications | Low | Early warning |
| Verify npm package integrity before deploying | Low | `npm pack --dry-run` + hash comparison |
| Consider trailing production xrpl.js by one minor version | Low | Lets the community vet new releases first |

---

## The Bigger Picture

The XRPL's supply chain security is not worse than most open-source projects of its scale. It is comparable to Ethereum's geth, Bitcoin Core, and Solana's validator client. None of these projects have perfect supply chain security.

What distinguishes the XRPL is the combination of:

1. **Irreversible amendments** (a supply chain compromise that ships a malicious amendment cannot be rolled back)
2. **Single implementation** (there is no alternative rippled implementation to compare against)
3. **Recent supply chain incident** (xrpl.js, April 2025, CVE-2025-32965)
4. **Growing institutional dependency** (Ripple Payments, Hidden Road, RLUSD, tokenised treasuries)

The xrpl.js incident was handled well: fast detection (12 hours), transparent disclosure, concrete remediation (2FA, access review, CVE). The question is whether the same discipline extends to the consensus node chain, the dependency chain, and the specification chain, where a compromise would be harder to detect and impossible to reverse once an amendment activates.

---

## Sources

- [XRPL Vulnerability Disclosure: xrpl.js npm Compromise (April 2025)](https://xrpl.org/blog/2025/vulnerabilitydisclosurereport-bug-apr2025)
- [CVE-2025-32965: xrpl.js Malicious Code Injection](https://nvd.nist.gov/vuln/detail/CVE-2025-32965)
- [rippled conanfile.py (dependency list)](https://github.com/XRPLF/rippled/blob/develop/conanfile.py)
- [rippled CONTRIBUTING.md](https://github.com/XRPLF/rippled/blob/develop/CONTRIBUTING.md)
- [rippled SECURITY.md](https://github.com/XRPLF/rippled/blob/develop/SECURITY.md)
- [Bybit Hack Technical Analysis, NCC Group](https://www.nccgroup.com/research-blog/in-depth-technical-analysis-of-the-bybit-hack/)
- [SLSA: Supply-chain Levels for Software Artifacts](https://slsa.dev/)
- [OpenSSF Scorecard](https://securityscorecards.dev/)
