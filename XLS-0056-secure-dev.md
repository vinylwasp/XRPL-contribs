# XLS-0056 Batch Amendment: Security SDLC Analysis

**Date:** 2026-03-02
**Author:** Legwork by Claude (Anthropic), adult supervision by Chris Lethaby (Vinylwasp)
**Subject:** XRPL Batch Transactions (XLS-0056), assessment of security design validation and testing prior to the February 2026 vulnerability disclosure

## Background

The XRPL Batch amendment (XLS-0056) introduces atomic multi-transaction execution on the XRP Ledger. It allows a single outer transaction to contain multiple inner transactions across one or more accounts, with four execution modes (ALLORNOTHING, ONLYONE, UNTILFAILURE, INDEPENDENT).

On 19 February 2026, external researcher Pranamya Keshkamat and Cantina AI identified a critical flaw in the signature validation logic that allowed an attacker to execute inner transactions on behalf of arbitrary victim accounts without their private keys. The amendment was in its validator voting phase, one vote from mainnet activation. rippled 3.1.1 was published on 23 February 2026 to disable the amendment.

This analysis examines the security practices visible in the feature's development lifecycle *before* the bug was discovered.

## What is present in XLS-0056

### Functional validation logic

The specification defines approximately 15 preflight failure conditions with specific error codes:

- `temINVALID_FLAG`: invalid flags on outer or inner transactions
- `temARRAY_EMPTY` / `temARRAY_TOO_LARGE`: bounds checking on RawTransactions
- `tefBAD_AUTH` / `tefMASTER_DISABLED`: authentication failures
- `tefBAD_QUORUM`: multi-sign quorum failures
- `temREDUNDANT`: duplicate inner transaction detection
- `temINVALID_INNER_BATCH`: inner transaction preflight failures

This demonstrates defensive programming at the specification level.

### Some abuse-case thinking

The spec addresses several misuse scenarios:

- Duplicate transaction prevention
- Nested batch prohibition (no batch-within-batch)
- Fee consolidation (inner transactions must have zero fees) to prevent fee manipulation
- Mandatory `tfInnerBatchTxn` flag on inner transactions
- BatchSigners must not include the outer transaction's account or contain duplicates

### Execution mode semantics

Four modes with defined failure behaviour represent significant combinatorial complexity. The spec attempts to define each mode's semantics, though the interaction between modes, multi-account signing, and edge cases in signer validation was not systematically explored.

## What is absent

| SDLC Security Activity | Evidence in XLS-0056 | Observation |
|---|---|---|
| Threat model (STRIDE, PASTA, attack trees) | None visible | No "Threats" section, no attacker personas, no structured abuse-case enumeration |
| Evil user stories | None | "As an attacker, I submit a Batch where one BatchSigner controls a non-existent account to bypass validation of subsequent signers". This is the exact bug that shipped. |
| Trust boundary analysis | None | The outer/inner transaction boundary is a classic trust boundary; who authenticates what, when, and what happens when validation short-circuits is never formally modelled |
| Security-focused SAST / DAST | Absent (see [Static Analysis Tooling](#static-analysis-tooling) below) | The CI pipeline runs **clang-tidy** (a general C++ linter) on PRs and daily, but no security-focused SAST (Semgrep, CodeQL, Coverity) is configured. No fuzz testing or dynamic scanning is visible. clang-tidy's bugprone checks do not cover authentication-logic flaws like CWE-305. |
| Formal security review or audit | None before voting began | The bug was found by an external researcher during the voting window, not during development or a pre-release review gate |
| Testnet/devnet security validation | Absent (see [Test Network Deployment and Validation](#test-network-deployment-and-validation) below) | The Batch amendment was force-enabled on devnet for months and the vulnerability was exploitable there throughout. All testing on devnet (Anodos Labs, performance testing) was functional or performance-focused, with zero adversarial testing. No bug bounty targeted devnet. XRPL Testnet mirrors mainnet by policy and played no role. |
| Security-focused test cases | Not visible | The spec defines happy-path failure conditions but no adversarial test matrix; no test for "what if the signer account doesn't exist on-ledger?" |
| Combinatorial edge-case analysis | Insufficient | 4 batch modes x multi-account signing x non-existent accounts x key-matching conditions produces a large state space with no visible systematic coverage |
| Independent security review gate | None | The feature reached the validator voting stage without evidence of a dedicated security review before entering the activation pipeline |

### Static Analysis Tooling

A review of the `XRPLF/rippled` repository's CI pipeline (March 2026) reveals the following static analysis posture:

**What is present:**

- **clang-tidy** runs on every PR (changed files only) and on a daily schedule against `develop` and `release` branches. The `.clang-tidy` configuration enables extensive `bugprone-*` checks. This is the repository's only automated static analysis tool.
- **Pre-commit hooks** (`.pre-commit-config.yaml`) enforce formatting (clang-format, prettier, black, cmake-format), spell-checking (cspell), and basic hygiene (trailing whitespace, merge conflict markers). None are security-focused.

**What is absent:**

- No **Semgrep** configuration (`.semgrep.yml`, `.semgrep/` rules directory) exists anywhere in the repository. A code search for "semgrep" across the entire `XRPLF` GitHub organisation returns zero results. The only reference is a single commit (PR #5903, October 2025) that cites Semgrep's *published guidance* on GitHub Actions input sanitisation. The developer followed Semgrep's advice, but Semgrep does not run as a tool in the pipeline.
- No **CodeQL**, **Coverity**, **Snyk**, or other security-focused SAST tooling is configured.
- No **fuzz testing** infrastructure (libFuzzer, AFL, OSS-Fuzz integration) is visible.
- The GitHub code-scanning API returns HTTP 403 (not publicly accessible), so a Semgrep GitHub App integration cannot be conclusively ruled out. However, the complete absence of configuration files, rule definitions, and any textual reference to Semgrep in the codebase makes this unlikely.

**Could SAST run privately at Ripple?**

It is reasonable to assume that Ripple, as a regulated fintech company, may operate internal security tooling that is not visible in the public `XRPLF/rippled` repository. Enterprise SAST platforms (including Semgrep AppSec Platform) can run against private mirrors or internal repos without requiring configuration files in the public codebase. This analysis does not claim that Ripple has no security tooling; it notes only that none is visible in the public development pipeline where this vulnerability was introduced.

However, several observations constrain how effective any private SAST could have been for XLS-0056:

1. **Development happened in public.** PRs #5060 and #6069 were authored, reviewed, and merged on `XRPLF/rippled`, not mirrored from an internal repository. If SAST runs only on a private mirror, it would catch issues *after* code lands on the public default branch, not at PR review time when it matters most.
2. **All CI checks are visible.** PR #5060 showed 26 passing checks; PR #6069 showed 24. None reference security scanning. If a private Semgrep integration ran as a PR check, it would typically appear in the checks list even if detailed results were gated behind authentication.
3. **No custom rules checked in.** Even organisations running Semgrep privately typically maintain `.semgrep/` rule directories or reference custom rule packs in their configuration. The absence of any such artifacts suggests that if SAST runs privately, it uses only generic rulesets, not custom rules tailored to rippled's authentication patterns. Custom rules are where the real value lies for catching defects like CWE-305.
4. **The outcome speaks for itself.** The vulnerability survived two PRs and 8 months of development. If security SAST was running at any stage of the pipeline (public or private), it either did not flag the early-return pattern in `checkBatchSign` or the finding was not actioned. Neither outcome supports the claim that SAST provided meaningful security coverage for this feature.

This analysis gives Ripple the benefit of the doubt that internal tooling may exist. But the relevant question is not whether a tool is *installed*; it is whether it *prevented the defect*. On that measure, the evidence is clear: no automated security analysis, public or private, caught this bug before an external researcher did.

**Why this matters for XLS-0056:**

clang-tidy is a valuable general-purpose C++ linter, but its `bugprone-*` checks operate at the syntactic and type-safety level. It does not model authentication semantics or control-flow implications of early returns in validation loops. The `checkBatchSign` bug (`return tesSUCCESS` instead of `continue` inside a signer-validation loop) is syntactically valid C++ and would not trigger clang-tidy warnings.

Security-focused SAST tools (Semgrep, CodeQL) support custom rules that flag exactly this class of defect: early exits from authentication/authorisation loops, `return` where `continue` was likely intended in validation contexts, and success-path short-circuits in multi-entity verification. A Semgrep rule matching `return $SUCCESS` inside `for` loops within functions named `*Sign*` or `*Auth*` would be straightforward to write and would have flagged this code at PR time.

The absence of security-focused SAST in the public pipeline, and the absence of any observable effect from private tooling (if it exists), means no automated analysis validated code *security* for this feature. For a financial protocol where authentication bypass has catastrophic consequences, this is a material gap.

## PR Trail: Who Built, Reviewed, and Merged What

### PR #5060, Original Batch Implementation (23 May 2025)

The foundational implementation of the Batch transaction type. **This PR contains the vulnerable code.**

| Field | Detail |
|---|---|
| **Author** | Denis Angell (@dangell7), external contributor, from `Transia-RnD/rippled` fork |
| **Merged by** | Bart (@bthomee) |
| **Commits** | 114 |
| **Files changed** | 69 (+6,400 / −744) |
| **Reviewers** | @mvadari, @shawnxie999, @cindyyan317, @ximinez (all COMMENTED, no formal GitHub approval recorded) |
| **Requested but not completed** | @ximinez (requested review), @ckeshava |
| **CI checks** | 26 passed |
| **Code coverage** | 94.05% patch, 32 lines uncovered |
| **Test plan** | External gist (dangell7 personal account), not a formal test specification |

**The vulnerable code** is in the `checkBatchSign` function added to `Transactor.cpp`:

```cpp
// From PR #5060, Transactor::checkBatchSign
for (auto const& signer : signers)
{
    auto const idAccount = signer.getAccountID(sfAccount);
    // ...
    auto const sleAccount = ctx.view.read(keylet::account(idAccount));

    // A batch can include transactions from an un-created account ONLY
    // when the account master key is the signer
    if (!sleAccount)
    {
        if (idAccount != idSigner)
            return tefBAD_AUTH;

        return tesSUCCESS;  // ← BUG: exits the ENTIRE loop,
                            //   skipping all remaining signers
    }
    // ...
}
```

When a BatchSigner's account does not exist on-ledger and the signing key derives to the same account ID, the function returns `tesSUCCESS` immediately, exiting the `for` loop. Every subsequent signer in the array is never validated. An attacker can place a signer for a non-existent account first in the array, followed by signers for victim accounts with forged or missing signatures, and the loop will never reach them.

The correct code should have used `continue` (proceed to next signer) rather than `return tesSUCCESS` (declare all signers valid). This is a classic instance of [CWE-305: Authentication Bypass by Primary Weakness](https://cwe.mitre.org/data/definitions/305.html).

**However, this bug was dormant at merge time.** PR #5060 also contained an inverted condition in `Transactor::preflight2`:

```cpp
// From PR #5060, Transactor::preflight2
if (ctx.tx.isFlag(tfInnerBatchTxn) && !ctx.rules.enabled(featureBatch))
    return tesSUCCESS;
```

This says "skip signature checks on inner batch transactions when Batch is NOT enabled", which is logically backwards and effectively dead code (the flag can't exist if the feature isn't enabled). This meant inner transactions still went through normal signature validation, which masked the broken `checkBatchSign` loop.

**Observations:**

- 114 commits and 69 files (+6,400 / −744 lines) is a substantial changeset. No reviewer recorded a formal GitHub approval; all reviews were COMMENTED status only, meaning the PR was merged without explicit sign-off through GitHub's approval mechanism.
- No security-specific reviewer. The reviewers are protocol/transaction developers, not security engineers. No evidence of a dedicated security review or sign-off.
- 94% patch coverage sounds strong, but coverage measures code execution, not adversarial path exploration. The vulnerable code path was *covered*; it simply had wrong logic.
- The test plan is a personal gist, not a formal test specification with negative/adversarial cases.
- @ximinez was requested to review but did not approve; he later authored fix PRs post-merge, suggesting issues were found only after the code landed.

### PR #6069, fixBatchInnerSigs (10 January 2026)

**This PR activated the dormant vulnerability** by fixing the inverted condition and adding a new amendment.

| Field | Detail |
|---|---|
| **Author** | Ed Hennis (@ximinez), Ripple core developer |
| **Merged by** | Ed Hennis (@ximinez), **same person who authored it** |
| **Approvers** | @mvadari, @dangell7 |
| **Commits** | 22 |
| **CI checks** | 24 passed |
| **Code coverage** | 79.2% overall, all modified lines covered |
| **Labels** | "Amendment", "Ready to merge" |
| **Amendment name** | `fixBatchInnerSigs` |

The key change in `Transactor::preflight2`:

```diff
- if (ctx.tx.isFlag(tfInnerBatchTxn) && !ctx.rules.enabled(featureBatch))
+ if (ctx.tx.isFlag(tfInnerBatchTxn) && ctx.rules.enabled(featureBatch))
      return tesSUCCESS;
```

This flipped the condition to skip signature checks on inner batch transactions when Batch IS enabled (the intended behaviour). The design assumption is sound: inner transactions don't carry their own signatures; authentication is delegated to the outer transaction's `checkBatchSign`. But `checkBatchSign` had the early-exit bug from PR #5060, so removing the (accidentally protective) redundant signature check exposed the broken loop.

The PR description stated: *"I don't think this is exploitable as such, because there are additional checks before an inner transaction is applied to the ledger, but why take that chance?"*

**Observations:**

- **Self-merge.** The author merged his own PR. This bypasses the fundamental principle of independent review: the person who wrote the code should not be the person who approves it into the codebase.
- **Two approvers, both from the same feature team.** @dangell7 authored the original Batch implementation (and the vulnerable `checkBatchSign` code); @mvadari approved both PRs. Neither is a security reviewer. This creates a closed review loop within the feature team with no external security perspective.
- **The author acknowledged uncertainty about exploitability** ("I don't think this is exploitable") but merged anyway without seeking a security-focused review. The phrase "why take that chance?" is ironic in hindsight, as the PR removed the last layer of defence hiding the bug.
- **@dangell7 flagged concern during review.** On 2 December 2025, his [comment on the `Transactor.cpp` change](https://github.com/XRPLF/rippled/pull/6069#discussion_r2580751864), *"This doesnt seem right. Can't you just check for the parentBatchId?"*, was a red flag. @ximinez [responded 13 days later](https://github.com/XRPLF/rippled/pull/6069#discussion_r2620120779): *"I ended up fixing this because of test failures that started after I merged Lending code in from `develop`"*, citing a commit hash. No further discussion. @dangell7 approved the PR on 9 January without revisiting his concern.
- **The author explicitly acknowledged limited adversarial reasoning.** On the `apply.cpp` thread about missing code coverage, @ximinez [wrote](https://github.com/XRPLF/rippled/pull/6069#discussion_r2621516482): *"It also shows that a transaction can't succeed in any of the ways I can think of to submit it. So Batch is likely to be safe with or without this amendment."* The qualifier "ways I can think of" is exactly the gap that structured threat modelling fills; he tested the paths he imagined, not the paths an attacker would try.
- **No adversarial test cases.** Tests verified pre- and post-amendment behaviour but did not include scenarios with non-existent signer accounts or early-exit validation paths.
- **Approval before clarification.** @mvadari [approved the PR](https://github.com/XRPLF/rippled/pull/6069#pullrequestreview-3641905497) at 00:52 UTC on 9 January with no comments. Two minutes later, at 00:54, she [posted](https://github.com/XRPLF/rippled/pull/6069#discussion_r2674408854): *"Just to clarify, this isn't consensus-breaking because you can't have tfInnerBatchTxn before featureBatch is enabled anyways, right?"* @ximinez replied *"Correct."* The approval was not conditional on the answer. The question itself is optimistic developer thinking: it asks "this won't break existing behaviour?" rather than "could this new behaviour be exploited?" The framing assumes safety and seeks confirmation, rather than assuming risk and seeking evidence. This is the opposite of a security review posture.
- **All three self-merges went to the default branch.** PR #5060 and #6069 merged to `develop` (rippled's default), XRPL-Standards #452 merged to `master`. These are not intermediate feature branches with a further review gate; they are the main development line that feeds releases and amendment voting.

### XRPL-Standards PR #452, Integration Considerations (17 February 2026)

Two days before the vulnerability was disclosed, the specification was still being updated.

| Field | Detail |
|---|---|
| **Author** | @mvadari |
| **Merged by** | @mvadari, **self-merge** |
| **Approver** | @sappenin (single reviewer, approved same day) |
| **Merged** | 17 February 2026 |
| **Content** | Adds Section 6 (Integration Considerations) for client libraries, wallets, and explorers |

**Observations:**

- The spec was still being written while the amendment was in validator voting. Integration guidance for client libraries, wallets, and explorers was added on 17 Feb, but the amendment was already being voted on. This is backwards: integration guidance should exist before implementation, not after.
- Self-merge on the spec too. @mvadari authored and merged this PR herself, with a single same-day approval.
- The added content is functional, not security. There's a brief mention of fraud detection ("if it is not in the same ledger, then it is likely a fraud attempt") but no section on security considerations for integrators, no guidance on validating BatchSigner authenticity, no warning about the trust model.

### Post-merge fix PRs

| PR | Title | Date | Author | Observation |
|---|---|---|---|---|
| #6176 | Reorder Batch Preflight Errors | 8 Jan 2026 | @dangell7 | Error ordering cleanup |
| #6207 | Suppress 'parse failed' message in Batch tests | 14 Jan 2026 | @ximinez | Test noise suppression |
| #6402 | Disable support for Batch and fixBatchInnerSigs | 21 Feb 2026 | @vlntb | Emergency disable after vulnerability disclosure |
| #6410 | Set version to 3.1.1 | 23 Feb 2026 | @ximinez | Emergency release |

The pattern of post-merge fix PRs (#6069, #6176, #6207) before the vulnerability was even discovered suggests the original implementation was not stable at merge time.

## Test Network Deployment and Validation

The XLS-0056 secure development analysis above covers the PR trail, code review, and static analysis. This section examines the role (or absence) of test network validation in the Batch amendment's release lifecycle.

### XRPL network topology

XRPL operates three public networks relevant to amendment testing:

| Network | Purpose | Amendment policy |
|---|---|---|
| **Mainnet** | Production financial network | Amendments activate via 80% validator consensus over two weeks. Irreversible. |
| **Testnet** | Mirrors mainnet amendment status | Per XRPL policy (established after the [2020 testnet amendments incident](https://xrpl.org/blog/2020/testnet-amendments-rippled-1.5.0)), new amendments should be **vetoed on Testnet until they gain majority on Mainnet**. Testnet is not a pre-production validation environment; it follows mainnet, not the other way around. |
| **Devnet** | Experimental feature preview | Amendments that are finished but awaiting release or voting are force-enabled. Based on the `develop` branch. Periodically reset. |

In addition, @dangell7 (the Batch feature author) operated an ad-hoc **"BatchNet"** on personal infrastructure (`nerdnest.xyz`) from approximately August 2024, providing early access to the Batch transaction type before it entered the official release pipeline.

### What testing occurred on each network

#### BatchNet (August 2024 onwards)

- **Operated by:** @dangell7, the same developer who authored the vulnerable `checkBatchSign` code in PR #5060
- **Infrastructure:** Community/personal servers at `batch.rpc.nerdnest.xyz` and `batch.nerdnest.xyz`
- **Purpose:** Developer preview for Batch transaction integration
- **Security testing:** None visible. BatchNet was a feature development environment, not a security validation platform.
- **Independence:** Zero. The network was operated by the feature author, using his fork's code. No independent parties are known to have conducted adversarial testing against it.

#### Devnet (Batch force-enabled from approximately mid-2025)

Devnet had the Batch amendment force-enabled. This is confirmed by the [devnet reset scheduled for March 3, 2026](https://u.today/xrp-ledger-devnet-reboot-scheduled-for-march-3-as-devs-prepare-for-update), which was required specifically because Batch was set to "unsupported" in rippled 3.1.1 and validators running the new version on devnet would become amendment-blocked.

**The vulnerability was exploitable on devnet for months.** From the time the Batch amendment was force-enabled through February 19, 2026, anyone submitting a Batch transaction with a non-existent signer account followed by victim account signers would have triggered the `checkBatchSign` early-return bug. No one did, because no adversarial testing was conducted.

Two testing efforts occurred on devnet:

1. **Anodos Labs functional testing (October 2025).** [Published report](https://dev.to/anodos/xrpl-batch-transactions-testing-report-3p1n) validated functional behaviour: bundling Payment and TrustSet operations, ALLORNOTHING mode, fee abstraction, and sequence numbering. **No signature validation testing. No authentication bypass testing. No adversarial inputs.** The report concluded the amendment was *"technically sound, functions as designed, and is ready for mainnet activation."* This conclusion was based entirely on happy-path functional testing. Anodos Labs is an XRPL UNL validator, and their endorsement carried weight with other validators considering their votes.

2. **Performance testing (August 2025).** [Published report](https://dev.to/ripplexdev/batch-transaction-performance-testing-report-13j8) by Qi Zhao and Luc des Trois Maisons measured throughput and latency on a private 9-node simulation (5 validators, 4 P2P gateways on AWS). **Performance-only scope. No security testing of any kind.** The report confirmed Batch transactions did not impair ledger throughput but did not examine whether inner transaction signers were correctly validated.

#### Testnet

Per XRPL's own policy, testnet mirrors mainnet amendment status. The Batch amendment was not activated on mainnet, so it would not have been active on testnet. **Testnet played no role in Batch validation.**

### What was missing

| Test Network Activity | Status | Observation |
|---|---|---|
| Adversarial transaction testing on devnet | **Absent** | No test submitted Batch transactions with non-existent signer accounts, mismatched signatures, or signer-ordering attacks. The bug was one adversarial transaction away from discovery. |
| Independent security testing on any network | **Absent** | All testing was conducted by the feature team or aligned community members. No external security researchers tested Batch on devnet or BatchNet. |
| Bug bounty against devnet-enabled amendment | **Absent** | No amendment-scoped bug bounty was established for Batch on any test network. The bug bounty that ultimately caught the vulnerability (Cantina/Immunefi) ran against code already in the mainnet voting window. |
| Structured testnet validation phase | **Absent** | No defined period between "amendment available on devnet" and "amendment enters mainnet voting" was reserved for independent security testing. |
| Soak period with monitoring | **Absent** | No evidence of systematic monitoring of devnet consensus metrics, error rates, or anomalous transaction patterns during the months Batch was force-enabled. |

### The structural problem

The XRPL test network architecture has a design-level gap for pre-activation security validation:

1. **Testnet cannot serve as a pre-production gate.** XRPL policy explicitly states testnet should mirror mainnet; amendments are vetoed on testnet until they gain majority on mainnet. This means testnet follows mainnet activation, not the other way around. There is no equivalent of Ethereum's Sepolia/Holesky deployment phase where amendments run on a live test network for weeks to months before mainnet activation, with expanded bug bounties targeting the new features.

2. **Devnet is a development environment, not a security validation environment.** Devnet has amendments force-enabled and is periodically reset. It is not structured for sustained adversarial testing, independent security engagement, or soak-period monitoring. No bug bounty programme targets devnet-specific amendment testing.

3. **BatchNet was author-operated.** Having the feature author operate the only dedicated test network creates the same closed-loop problem seen in the code review: the person who wrote the code is also controlling the environment where it is tested. Independent testers need infrastructure they don't control and didn't build.

4. **No transition gate exists between devnet availability and mainnet voting.** The Batch amendment went from devnet (force-enabled, functional testing only) directly to mainnet voting (validator signalling, one vote from irreversible activation) with no structured security validation phase in between.

### The missed opportunity

The vulnerability was exploitable on devnet for approximately 4–8 months before it was discovered on February 19, 2026. During this window:

- Anodos Labs tested on devnet and declared it ready for mainnet, without testing authentication paths
- Performance testing ran on a private simulation, without testing security
- The amendment entered mainnet voting, without any adversarial testing on any network
- Validators began signalling support, based on functional endorsements, not security evidence

A single adversarial Batch transaction (constructed using the procedure described in the [Impact Analysis](#why-this-matters--impact-analysis)), submitted to devnet at any point during this window would have exposed the `checkBatchSign` bypass. The test networks existed. The amendment was running. No one tested what an attacker would do.

### Comparison with peer protocols

| Protocol | Pre-activation testing practice |
|---|---|
| **Ethereum** | Amendments deployed to Sepolia and Holesky testnets weeks to months before mainnet hard forks. Continuous bug bounties with scope expansions targeting upcoming upgrades. |
| **Polkadot** | Kusama canary network (a production-grade testnet with real economic value) validates runtime upgrades before Polkadot mainnet. |
| **Cosmos** | Incentivised testnets with real rewards for finding issues in major upgrades. |
| **Solana** | Feature gates activated on testnet and devnet before mainnet, with extended observation periods for consensus changes. |
| **XRPL** | Devnet force-enables amendments. Testnet mirrors mainnet (follows, does not lead). No structured security testing phase. No amendment-scoped bounties on test networks. Bug bounty ran against code in the mainnet voting window. |

XRPL's test network infrastructure exists but is not positioned as a security validation gate. The gap is process discipline, not technical capability.

---

## The vulnerability: two PRs, one bug

The vulnerability is the product of two PRs interacting:

1. **PR #5060** (May 2025) introduced `checkBatchSign` with an early-return bug: when a signer's account doesn't exist on-ledger and the key matches, it returns `tesSUCCESS` immediately, skipping all remaining signers. The same PR also had an inverted condition in `preflight2` that accidentally kept redundant signature checks on inner transactions, masking the broken loop.

2. **PR #6069** (January 2026) correctly fixed the inverted `preflight2` condition, removing the redundant signature checks (the intended design). But this exposed the `checkBatchSign` early-exit bug that had been dormant for 8 months.

Neither PR introduced the vulnerability alone. The root cause is `return tesSUCCESS` instead of `continue` in an authentication loop, a textbook instance of [CWE-305: Authentication Bypass by Primary Weakness](https://cwe.mitre.org/data/definitions/305.html). It sat undetected for 8 months, was activated by a well-intentioned fix, and nearly reached mainnet.

### Why standard security practices would have caught this

1. **Threat modelling.** The inner/outer transaction authentication boundary is a classic trust boundary. STRIDE analysis on the signing flow would identify "spoofing" as a threat and ask: "What happens if a signer can't be validated? Does the loop continue or exit?" That question leads directly to the bug.

2. **Evil user stories.** "As an attacker, I submit a Batch where the first BatchSigner controls a non-existent account, followed by signers for victim accounts with forged signatures." Structured misuse-case analysis would surface this scenario.

3. **Security-focused SAST.** The repository runs clang-tidy, but this is a general C++ linter, not a security SAST tool. Security-focused static analysis (Semgrep, CodeQL) flags early-return patterns in authentication loops as high-risk. The `return tesSUCCESS` inside a `for` loop that validates multiple signers is a well-known anti-pattern (CWE-305). A Semgrep rule targeting `return` statements inside signer/auth validation loops would have flagged this at PR time. No such tooling is configured in the rippled CI pipeline despite claims to the contrary (see [Static Analysis Tooling](#static-analysis-tooling)).

4. **Adversarial test cases.** A test matrix combining {existent account, non-existent account} x {valid key, invalid key} x {first signer, middle signer, last signer} would have 18 cases. The bug is triggered by any test with a non-existent matching-key signer followed by any other signer.

5. **Independent security review.** A reviewer outside the feature team, looking at the code with an adversarial mindset, would ask why an authentication loop returns success mid-iteration rather than continuing. This is Security Code Review 101.

6. **Adversarial testnet validation.** The vulnerability was exploitable on devnet for months. A single adversarial Batch transaction (with a non-existent signer account placed first in the `BatchSigners` array, followed by a victim account signer) would have triggered the bypass. An amendment-scoped bug bounty on devnet, or structured adversarial testing by independent parties, would have found this before the amendment entered mainnet voting. Peer protocols (Ethereum, Polkadot, Cosmos, Solana) all deploy to test networks with security-focused testing phases before production activation.

## Assessment

The Batch feature demonstrates **functional design rigour**: detailed failure conditions, defined execution semantics, and defensive checks at the spec level. However, there is **near-zero evidence of security design validation** in the SDLC.

Key process failures:

- **No threat modelling** at any stage (spec, implementation, or amendment)
- **No security-specific reviewers** on the original PR, the amendment that activated the vulnerability, or the spec updates
- **Self-merge on security-sensitive code** (PR #6069) and on the specification (XRPL-Standards PR #452)
- **Closed review loop.** The same small group (@dangell7, @mvadari, @ximinez) authored, reviewed, and approved all Batch PRs with no external security perspective
- **Dismissed reviewer concern.** @dangell7's "This doesn't seem right" comment on PR #6069 was not escalated or investigated
- **No adversarial test cases.** High code coverage but zero authentication bypass scenarios
- **Static analysis is quality-focused, not security-focused.** clang-tidy runs in CI but does not detect authentication-logic flaws; no security SAST (Semgrep, CodeQL, Coverity) is configured despite the repository handling cryptographic authentication for a financial protocol
- **No test network security validation.** The vulnerability was exploitable on devnet for months while all testing (Anodos Labs functional, Ripple performance) examined only happy-path behaviour. No adversarial transactions were submitted to any test network. No bug bounty targeted the devnet-enabled amendment. XRPL Testnet mirrors mainnet by policy and was structurally unable to serve as a pre-activation validation gate.
- **Spec still being written during voting.** Integration considerations added 2 days before the vulnerability disclosure
- **No security gate before the amendment entered validator voting.** This was the last chance to catch issues before an irreversible mainnet activation

For a financial ledger where the amendment process is effectively irreversible once activated, these gaps are significant. The vulnerability was caught by luck (an external researcher and AI tooling during the voting window) rather than by a security gate built into the development process.

## Why This Matters: Impact Analysis

Had the Batch amendment activated on mainnet, this vulnerability would have allowed **universal, unauthenticated account takeover and fund drainage** across the entire XRP Ledger. This section quantifies the impact.

### Attack construction

The exploit is trivial to construct:

1. The attacker generates a fresh keypair. The derived account (B) does not exist on-ledger.
2. The attacker builds a Batch transaction containing inner transactions from the **victim's account** (e.g., a Payment draining funds to the attacker).
3. The attacker places account B's `BatchSigner` entry **first** in the `BatchSigners` array, followed by the victim's signer entry with a forged or missing signature.
4. The attacker signs the outer Batch transaction only with account B's key.
5. `checkBatchSign` processes account B first: account does not exist on-ledger, signing key matches account ID → `return tesSUCCESS`. The loop exits. The victim's signer entry is **never validated**.
6. Inner transactions execute as if the victim authorised them.

No interaction with the victim is required. No brute-forcing. No social engineering. One keypair, one Batch transaction.

### Exploitable transaction types

The XLS-0056 specification places almost no restrictions on inner transaction types. The only prohibited types are `Batch` itself (no nesting) and disabled lending/vault types. **All standard XRPL transaction types were allowed**, including:

| Transaction Type | What an attacker could do |
|---|---|
| **Payment** | Drain XRP and issued tokens from any account down to the reserve floor (~10 XRP) |
| **SetRegularKey** | Set the attacker's key as the victim's regular key, enabling **persistent account takeover that survives a code fix** |
| **SignerListSet** | Replace the victim's multisig signer list, taking over governance entirely |
| **AccountSet** | Modify account flags (disable master key, change settings) |
| **OfferCreate / OfferCancel** | Manipulate DEX order books, place or cancel orders from victim accounts |
| **AMMDeposit / AMMWithdraw** | Drain AMM liquidity positions |
| **NFTokenCreateOffer / NFTokenAcceptOffer** | Steal NFTs |
| **TrustSet** | Create trust lines from victim accounts to attacker-controlled issuers |
| **EscrowCreate / EscrowFinish** | Manipulate escrows |
| **PaymentChannelCreate / PaymentChannelFund** | Manipulate payment channels |
| **CheckCreate / CheckCash** | Create checks payable to attacker from victim accounts |
| **AccountDelete** | Delete victim accounts (subject to sequence age preconditions) |

### No secondary defences

Inner Batch transactions carry **no signatures of their own** by design: `SigningPubKey` must be empty and `TxnSignature` must be omitted. The entire security model delegates authentication to the outer transaction's `checkBatchSign` function. Once `checkBatchSign` returned `tesSUCCESS` prematurely, there was **zero fallback authorisation**.

- **Preflight checks** validate structural correctness (valid fields, amounts), not authorisation.
- **Preclaim checks** validate ledger state (sufficient balance, account exists), but run *after* the signature bypass. They assume the transaction is already authorised.
- **Invariant checks** enforce per-transaction constraints (e.g., account cannot go below reserve), but do not verify whether the transaction was legitimately signed.
- **DepositAuth** blocks *incoming* payments; the attack uses *outgoing* payments from the victim.
- **Multisig, regular keys, disabled master keys.** None of these account-level security configurations would have helped, because the bypass occurred at the Batch signer level, not individual account authentication.

### Value at risk

| Metric | Approximate value (mid-February 2026) | Source |
|---|---|---|
| XRP market capitalisation | ~$80–96 billion | CoinMarketCap, multiple media reports |
| XRPL DeFi TVL | ~$51–55 million | DefiLlama |
| XRPL total RWA value | ~$1.96–2.3 billion | Ripple exec confirmation (27 Feb), 247WallSt, BitcoinEthereumNews |
| RLUSD stablecoin (total) | ~$1.56 billion | U.Today, Phemex, CoinPaprika (multiple independent) |
| RLUSD on XRPL specifically | ~$348–360 million | ~23% of total supply on-chain (CryptoSlate chain-split reporting) |
| U.S. Treasury tokens (Ondo OUSG, Guggenheim, OpenEden) | ~$300 million | The Crypto Basic (23 Feb 2026) |
| Justoken JMWH energy tokens | ~$861 million | Largest single RWA on XRPL; launched mid-January 2026 |
| Ctrl Alt diamond tokens | ~$280 million | The Coin Republic; Ripple/Billiton Diamond deal confirmed 28 Feb |

Additional issued assets at risk include Société Générale's EURCV euro stablecoin (MiCA-compliant, launched on XRPL February 2026), Circle's USDC (natively available on XRPL), and multiple additional stablecoins (EUROP, USDB, XSGD). Individual on-chain supply figures for these are not publicly reported, but they contribute to the aggregate exposure.

The vulnerability was **universal**: every funded account on the XRPL was equally vulnerable regardless of its security configuration. An attacker could have automated mass drainage limited only by XRPL's transaction throughput (~1,500 TPS). `SetRegularKey` takeovers would have been **persistent**, surviving the emergency code fix and requiring individual account-by-account remediation by every affected user.

The combined exposure ($80B+ in XRP, $2.3B in tokenised RWAs, $1.56B in RLUSD, and all DEX/AMM positions) represents the total blast radius. The media framing of "$80 billion at risk" captures the XRP market cap but understates the full scope of issued assets, stablecoins, and tokenised real-world assets that were equally exposed. The exploit was mechanically simple, universally applicable, and had no mitigating controls. The only thing between a working exploit and mainnet activation was one more validator vote and an external researcher who happened to be analysing the right code at the right time.

### Severity scoring, CWSS v1.0.1

The vulnerability is classified as **[CWE-305: Authentication Bypass by Primary Weakness](https://cwe.mitre.org/data/definitions/305.html)**. The following CWSS (Common Weakness Scoring System) assessment uses MITRE's v1.0.1 methodology, the current version as of March 2026. CWSS was chosen over CVSS because this analysis assesses a design-level weakness class (CWE-305) in the development lifecycle, not a specific CVE instance in a deployed system. CWSS scores weaknesses at design time; CVSS scores vulnerabilities at disclosure time. CWSS has not been updated since September 2014; the version numbers sometimes confused with "CWSS 4" refer to the CWE *list* versioning (currently v4.17–4.19), not the scoring system itself. CWSS v1.0.1 remains the canonical weakness scoring framework published by MITRE ([source](https://cwe.mitre.org/cwss/cwss_v1.0.1.html)).

**Vector:**

```
TI:C,1.0/AP:A,1.0/AL:A,1.0/IC:N,1.0/FC:T,1.0/RP:N,1.0/RL:A,1.0/AV:I,1.0/AS:N,1.0/IN:A,1.0/SC:A,1.0/BI:C,1.0/DI:H,1.0/EX:H,1.0/EC:N,1.0/P:W,1.0
```

#### Base Finding (technical severity)

| Metric | Value | Weight | Justification |
|---|---|---|---|
| Technical Impact (TI) | Critical | 1.0 | Complete authentication bypass; arbitrary transactions executed without victim's private key |
| Acquired Privilege (AP) | Administrator | 1.0 | Full account control: drain funds, set keys, delete account, modify governance |
| Acquired Privilege Layer (AL) | Application | 1.0 | Protocol-level compromise; the authentication mechanism itself is bypassed |
| Internal Control Effectiveness (IC) | None | 1.0 | No secondary authorisation checks exist for inner Batch transactions once `checkBatchSign` returns success |
| Finding Confidence (FC) | Proven True | 1.0 | Independently verified by Ripple engineering with PoC and unit test reproduction on 19 February 2026 |

**Base Finding subscore:** [(10×1.0 + 5×(1.0+1.0) + 5×1.0) × f(TI) × 1.0] × 4.0 = **100.0**

#### Attack Surface (exploitability)

| Metric | Value | Weight | Justification |
|---|---|---|---|
| Required Privilege (RP) | None | 1.0 | Attacker needs only a self-generated keypair; no existing account, no special access |
| Required Privilege Layer (RL) | Application | 1.0 | Exploit operates at the XRPL protocol layer |
| Access Vector (AV) | Internet | 1.0 | Batch transactions can be submitted to any public XRPL node via WebSocket or JSON-RPC |
| Authentication Strength (AS) | None | 1.0 | No authentication is required to submit transactions to XRPL nodes; the attacker authenticates only with their own key |
| Level of Interaction (IN) | Automated | 1.0 | No victim interaction required; fully automatable, no social engineering, no timing dependency |
| Deployment Scope (SC) | All | 1.0 | Every XRPL validator and full-history node runs the same `rippled` binary; the amendment activates network-wide simultaneously |

**Attack Surface subscore:** [20×(1.0+1.0+1.0) + 20×1.0 + 15×1.0 + 5×1.0] / 100.0 = **1.0**

#### Environmental (real-world context)

| Metric | Value | Weight | Justification |
|---|---|---|---|
| Business Impact (BI) | Critical | 1.0 | Financial protocol securing $80B+ in assets; exploitation would constitute the largest theft in cryptocurrency history |
| Likelihood of Discovery (DI) | High | 1.0 | Simple logic flaw (`return` vs `continue`) discoverable by code review or automated static analysis; was in fact found by AI tooling |
| Likelihood of Exploit (EX) | High | 1.0 | Trivial exploit construction: single keypair, single transaction, no prerequisites beyond the amendment being active |
| External Control Effectiveness (EC) | None | 1.0 | No WAF, rate limiter, or external security control sits between a transaction submission and consensus processing |
| Prevalence (P) | Widespread | 1.0 | Every node in the XRPL network runs the same code; 100% of deployments are affected once the amendment activates |

**Environmental subscore:** [(10×1.0 + 3×1.0 + 4×1.0 + 3×1.0) × f(BI) × 1.0] / 20.0 = **1.0**

#### Final CWSS score

```
CWSS = BaseFinding × AttackSurface × Environmental
CWSS = 100.0 × 1.0 × 1.0 = 100.0 / 100
```

**CWSS Score: 100.0**, the theoretical maximum.

Every metric scores at its highest severity value. This is consistent with the nature of the weakness: a complete authentication bypass in a financial protocol with no mitigating controls, no privilege requirements, internet-accessible attack surface, trivially automatable exploitation, and universal deployment scope. The score reflects a vulnerability that, had it activated, would have placed the entire value of the XRP Ledger at the disposal of any attacker who understood the `checkBatchSign` loop.

### What actually happened

The amendment was one validator vote from mainnet activation when Pranamya Keshkamat and Cantina's AI tool (Apex) identified the flaw on 19 February 2026. Ripple engineering validated the report with an independent proof-of-concept the same evening. Emergency release `rippled 3.1.1` shipped on 23 February 2026, marking both `featureBatch` and `fixBatchInnerSigs` as unsupported to prevent validator votes from activating the amendment. **No funds were lost.** The full logic fix is under development as the `BatchV1_1` amendment, with no release date set.

## Sources

- [XLS-0056 Specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0056-batch)
- [XRPL-Standards PR #452, Integration Considerations](https://github.com/XRPLF/XRPL-Standards/pull/452)
- [PR #5060, Batch (XLS-56)](https://github.com/XRPLF/rippled/pull/5060)
- [PR #6069, fix: Inner batch transactions never have valid signatures](https://github.com/XRPLF/rippled/pull/6069)
- [PR #5903, Sanitize GitHub Actions inputs (sole Semgrep reference)](https://github.com/XRPLF/rippled/pull/5903)
- [XRPL Vulnerability Disclosure Report (Feb 2026)](https://xrpl.org/blog/2026/vulnerabilitydisclosurereport-bug-feb2026)
- [XRPL batch amendment security patch blocks mainnet risk](https://en.cryptonomist.ch/2026/02/27/xrpl-batch-amendment-security/)
- [Batch (XLS-56) is Now Available for Testing and Development, @dangell7 (Aug 2024)](https://dev.to/dangell7/batch-xls-56-is-now-available-for-testing-and-development-18bk)
- [XRPL Batch Transactions Testing Report, Anodos Labs (Oct 2025)](https://dev.to/anodos/xrpl-batch-transactions-testing-report-3p1n)
- [Batch Transaction Performance Testing Report, Qi Zhao, Luc des Trois Maisons (Aug 2025)](https://dev.to/ripplexdev/batch-transaction-performance-testing-report-13j8)
- [XRP Ledger Devnet Reboot Scheduled for March 3, U.Today](https://u.today/xrp-ledger-devnet-reboot-scheduled-for-march-3-as-devs-prepare-for-update)
- [Postmortem: Testnet Amendments from rippled 1.5.0 (2020)](https://xrpl.org/blog/2020/testnet-amendments-rippled-1.5.0)
- [XRPL Amendments, Network and Server Concepts](https://xrpl.org/docs/concepts/networks-and-servers/amendments)

### Verification note (2026-03-25, accuracy review)

All factual claims in this report were re-verified against primary sources on 2026-03-25 using the GitHub API (PR metadata, review status, merge history), the official XRPL vulnerability disclosure blog post, and repository content checks. All 15 PR trail claims, all 4 static analysis claims, and all 7 vulnerability disclosure claims were confirmed accurate against their primary sources. Market capitalisation and asset value figures are date-sensitive and cited with appropriate ranges and multiple independent sources.

External sources hosted on dev.to (Anodos Labs testing report, BatchNet announcement, performance testing report) are third-party content that may be modified or removed. The key conclusions from these sources (functional-only testing scope, no adversarial testing, no authentication validation) are quoted in this analysis and would survive source removal.

### Verification note (2026-03-04)

The SAST assessment was revalidated against the `XRPLF/rippled` public repository in response to a claim that Semgrep is used upstream before binary builds. Verification included: enumeration of all 14 GitHub Actions workflow files, inspection of `.pre-commit-config.yaml`, repository-wide code search for "semgrep" (zero results), org-wide code search across `XRPLF` (zero results), and API checks for `.semgrep.yml` and `.semgrep/` directory (404). The only Semgrep reference found is a commit message in PR #5903 citing Semgrep's published security guidance, not evidence of Semgrep as a pipeline tool. The GitHub code-scanning API returned 403 (not public), so a private GitHub App integration cannot be conclusively excluded, but no supporting artifacts exist in the public repository.

The possibility that Ripple runs SAST internally on a private repository or mirror was considered and addressed in the "Could SAST run privately at Ripple?" section. The analysis acknowledges this possibility but notes that (a) all Batch development occurred on the public repo, (b) no CI checks reference security scanning, (c) no custom SAST rules are checked in, and (d) the vulnerability was not caught by any automated tooling, public or private, over an 8-month window. The conclusion (that no effective security SAST covered this feature) holds regardless of whether private tooling exists.
