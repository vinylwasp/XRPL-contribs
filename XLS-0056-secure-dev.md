# XLS-0056 Batch Amendment — Security SDLC Analysis

**Date:** 2026-03-02
**Author:** Vinylwasp
**Subject:** XRPL Batch Transactions (XLS-0056) — assessment of security design validation and testing prior to the February 2026 vulnerability disclosure

## Background

The XRPL Batch amendment (XLS-0056) introduces atomic multi-transaction execution on the XRP Ledger. It allows a single outer transaction to contain multiple inner transactions across one or more accounts, with four execution modes (ALLORNOTHING, ONLYONE, UNTILFAILURE, INDEPENDENT).

On 19 February 2026, external researcher Pranamya Keshkamat and Cantina AI identified a critical flaw in the signature validation logic that allowed an attacker to execute inner transactions on behalf of arbitrary victim accounts without their private keys. The amendment was in its validator voting phase — one vote from mainnet activation. rippled 3.1.1 was published on 23 February 2026 to disable the amendment.

This analysis examines the security practices visible in the feature's development lifecycle *before* the bug was discovered.

## What is present in XLS-0056

### Functional validation logic

The specification defines approximately 15 preflight failure conditions with specific error codes:

- `temINVALID_FLAG` — invalid flags on outer or inner transactions
- `temARRAY_EMPTY` / `temARRAY_TOO_LARGE` — bounds checking on RawTransactions
- `tefBAD_AUTH` / `tefMASTER_DISABLED` — authentication failures
- `tefBAD_QUORUM` — multi-sign quorum failures
- `temREDUNDANT` — duplicate inner transaction detection
- `temINVALID_INNER_BATCH` — inner transaction preflight failures

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
| Evil user stories | None | "As an attacker, I submit a Batch where one BatchSigner controls a non-existent account to bypass validation of subsequent signers" — this is the exact bug that shipped |
| Trust boundary analysis | None | The outer/inner transaction boundary is a classic trust boundary; who authenticates what, when, and what happens when validation short-circuits is never formally modelled |
| SAST / DAST | No evidence | No mention of static analysis tooling, fuzz testing, or dynamic scanning in the specification or visible PR process |
| Formal security review or audit | None before voting began | The bug was found by an external researcher during the voting window, not during development or a pre-release review gate |
| Security-focused test cases | Not visible | The spec defines happy-path failure conditions but no adversarial test matrix; no test for "what if the signer account doesn't exist on-ledger?" |
| Combinatorial edge-case analysis | Insufficient | 4 batch modes x multi-account signing x non-existent accounts x key-matching conditions produces a large state space with no visible systematic coverage |
| Independent security review gate | None | The feature reached the validator voting stage without evidence of a dedicated security review before entering the activation pipeline |

## PR Trail — Who Built, Reviewed, and Merged What

### PR #5060 — Original Batch Implementation (23 May 2025)

The foundational implementation of the Batch transaction type. **This PR contains the vulnerable code.**

| Field | Detail |
|---|---|
| **Author** | Denis Angell (@dangell7) — external contributor, from `Transia-RnD/rippled` fork |
| **Merged by** | Bart (@bthomee) |
| **Commits** | 114 |
| **Files changed** | 69 (+6,400 / −744) |
| **Reviewers** | @mvadari, @shawnxie999, @cindyyan317, @ximinez (all COMMENTED — no formal GitHub approval recorded) |
| **Requested but not completed** | @ximinez (requested review), @ckeshava |
| **CI checks** | 26 passed |
| **Code coverage** | 94.05% patch, 32 lines uncovered |
| **Test plan** | External gist (dangell7 personal account), not a formal test specification |

**The vulnerable code** is in the `checkBatchSign` function added to `Transactor.cpp`:

```cpp
// From PR #5060 — Transactor::checkBatchSign
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

When a BatchSigner's account does not exist on-ledger and the signing key derives to the same account ID, the function returns `tesSUCCESS` immediately — exiting the `for` loop. Every subsequent signer in the array is never validated. An attacker can place a signer for a non-existent account first in the array, followed by signers for victim accounts with forged or missing signatures, and the loop will never reach them.

The correct code should have used `continue` (proceed to next signer) rather than `return tesSUCCESS` (declare all signers valid). This is a classic instance of CWE-305 (Authentication Bypass by Primary Weakness).

**However, this bug was dormant at merge time.** PR #5060 also contained an inverted condition in `Transactor::preflight2`:

```cpp
// From PR #5060 — Transactor::preflight2
if (ctx.tx.isFlag(tfInnerBatchTxn) && !ctx.rules.enabled(featureBatch))
    return tesSUCCESS;
```

This says "skip signature checks on inner batch transactions when Batch is NOT enabled" — logically backwards and effectively dead code (the flag can't exist if the feature isn't enabled). This meant inner transactions still went through normal signature validation, which masked the broken `checkBatchSign` loop.

**Observations:**

- 114 commits and 69 files (+6,400 / −744 lines) is a substantial changeset. No reviewer recorded a formal GitHub approval — all reviews were COMMENTED status only, meaning the PR was merged without explicit sign-off through GitHub's approval mechanism.
- No security-specific reviewer — the reviewers are protocol/transaction developers, not security engineers. No evidence of a dedicated security review or sign-off.
- 94% patch coverage sounds strong, but coverage measures code execution, not adversarial path exploration. The vulnerable code path was *covered* — it simply had wrong logic.
- The test plan is a personal gist, not a formal test specification with negative/adversarial cases.
- @ximinez was requested to review but did not approve — he later authored fix PRs post-merge, suggesting issues were found only after the code landed.

### PR #6069 — fixBatchInnerSigs (10 January 2026)

**This PR activated the dormant vulnerability** by fixing the inverted condition and adding a new amendment.

| Field | Detail |
|---|---|
| **Author** | Ed Hennis (@ximinez) — Ripple core developer |
| **Merged by** | Ed Hennis (@ximinez) — **same person who authored it** |
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

This flipped the condition to skip signature checks on inner batch transactions when Batch IS enabled — the intended behaviour. The design assumption is sound: inner transactions don't carry their own signatures; authentication is delegated to the outer transaction's `checkBatchSign`. But `checkBatchSign` had the early-exit bug from PR #5060, so removing the (accidentally protective) redundant signature check exposed the broken loop.

The PR description stated: *"I don't think this is exploitable as such, because there are additional checks before an inner transaction is applied to the ledger, but why take that chance?"*

**Observations:**

- **Self-merge.** The author merged his own PR. This bypasses the fundamental principle of independent review — the person who wrote the code should not be the person who approves it into the codebase.
- **Two approvers, both from the same feature team.** @dangell7 authored the original Batch implementation (and the vulnerable `checkBatchSign` code); @mvadari approved both PRs. Neither is a security reviewer. This creates a closed review loop within the feature team with no external security perspective.
- **The author acknowledged uncertainty about exploitability** ("I don't think this is exploitable") but merged anyway without seeking a security-focused review. The phrase "why take that chance?" is ironic in hindsight — the PR removed the last layer of defence hiding the bug.
- **@dangell7 flagged concern during review.** On 2 December 2025, his [comment on the `Transactor.cpp` change](https://github.com/XRPLF/rippled/pull/6069#discussion_r2580751864) — *"This doesnt seem right. Can't you just check for the parentBatchId?"* — was a red flag. @ximinez [responded 13 days later](https://github.com/XRPLF/rippled/pull/6069#discussion_r2620120779): *"I ended up fixing this because of test failures that started after I merged Lending code in from `develop`"* — citing a commit hash. No further discussion. @dangell7 approved the PR on 9 January without revisiting his concern.
- **The author explicitly acknowledged limited adversarial reasoning.** On the `apply.cpp` thread about missing code coverage, @ximinez [wrote](https://github.com/XRPLF/rippled/pull/6069#discussion_r2621516482): *"It also shows that a transaction can't succeed in any of the ways I can think of to submit it. So Batch is likely to be safe with or without this amendment."* The qualifier "ways I can think of" is exactly the gap that structured threat modelling fills — he tested the paths he imagined, not the paths an attacker would try.
- **No adversarial test cases.** Tests verified pre- and post-amendment behaviour but did not include scenarios with non-existent signer accounts or early-exit validation paths.
- **Approval before clarification.** @mvadari [approved the PR](https://github.com/XRPLF/rippled/pull/6069#pullrequestreview-3641905497) at 00:52 UTC on 9 January with no comments. Two minutes later, at 00:54, she [posted](https://github.com/XRPLF/rippled/pull/6069#discussion_r2674408854): *"Just to clarify, this isn't consensus-breaking because you can't have tfInnerBatchTxn before featureBatch is enabled anyways, right?"* — @ximinez replied *"Correct."* The approval was not conditional on the answer. The question itself is optimistic developer thinking: it asks "this won't break existing behaviour?" rather than "could this new behaviour be exploited?" The framing assumes safety and seeks confirmation, rather than assuming risk and seeking evidence. This is the opposite of a security review posture.
- **All three self-merges went to the default branch.** PR #5060 and #6069 merged to `develop` (rippled's default), XRPL-Standards #452 merged to `master`. These are not intermediate feature branches with a further review gate — they are the main development line that feeds releases and amendment voting.

### XRPL-Standards PR #452 — Integration Considerations (17 February 2026)

Two days before the vulnerability was disclosed, the specification was still being updated.

| Field | Detail |
|---|---|
| **Author** | @mvadari |
| **Merged by** | @mvadari — **self-merge** |
| **Approver** | @sappenin (single reviewer, approved same day) |
| **Merged** | 17 February 2026 |
| **Content** | Adds Section 6 (Integration Considerations) for client libraries, wallets, and explorers |

**Observations:**

- The spec was still being written while the amendment was in validator voting. Integration guidance for client libraries, wallets, and explorers was added on 17 Feb — the amendment was already being voted on. This is backwards: integration guidance should exist before implementation, not after.
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

## The vulnerability — two PRs, one bug

The vulnerability is the product of two PRs interacting:

1. **PR #5060** (May 2025) introduced `checkBatchSign` with an early-return bug: when a signer's account doesn't exist on-ledger and the key matches, it returns `tesSUCCESS` immediately, skipping all remaining signers. The same PR also had an inverted condition in `preflight2` that accidentally kept redundant signature checks on inner transactions, masking the broken loop.

2. **PR #6069** (January 2026) correctly fixed the inverted `preflight2` condition, removing the redundant signature checks — the intended design. But this exposed the `checkBatchSign` early-exit bug that had been dormant for 8 months.

Neither PR introduced the vulnerability alone. The root cause is `return tesSUCCESS` instead of `continue` in an authentication loop — a textbook instance of CWE-305 (Authentication Bypass by Primary Weakness). It sat undetected for 8 months, was activated by a well-intentioned fix, and nearly reached mainnet.

### Why standard security practices would have caught this

1. **Threat modelling.** The inner/outer transaction authentication boundary is a classic trust boundary. STRIDE analysis on the signing flow would identify "spoofing" as a threat and ask: "What happens if a signer can't be validated? Does the loop continue or exit?" That question leads directly to the bug.

2. **Evil user stories.** "As an attacker, I submit a Batch where the first BatchSigner controls a non-existent account, followed by signers for victim accounts with forged signatures." Structured misuse-case analysis would surface this scenario.

3. **SAST.** Static analysis tools flag early-return patterns in authentication loops as high-risk. The `return tesSUCCESS` inside a `for` loop that validates multiple signers is a well-known anti-pattern.

4. **Adversarial test cases.** A test matrix combining {existent account, non-existent account} x {valid key, invalid key} x {first signer, middle signer, last signer} would have 18 cases. The bug is triggered by any test with a non-existent matching-key signer followed by any other signer.

5. **Independent security review.** A reviewer outside the feature team, looking at the code with an adversarial mindset, would ask why an authentication loop returns success mid-iteration rather than continuing. This is Security Code Review 101.

## Assessment

The Batch feature demonstrates **functional design rigour**: detailed failure conditions, defined execution semantics, and defensive checks at the spec level. However, there is **near-zero evidence of security design validation** in the SDLC.

Key process failures:

- **No threat modelling** at any stage — spec, implementation, or amendment
- **No security-specific reviewers** on the original PR, the amendment that activated the vulnerability, or the spec updates
- **Self-merge on security-sensitive code** (PR #6069) and on the specification (XRPL-Standards PR #452)
- **Closed review loop** — the same small group (@dangell7, @mvadari, @ximinez) authored, reviewed, and approved all Batch PRs with no external security perspective
- **Dismissed reviewer concern** — @dangell7's "This doesn't seem right" comment on PR #6069 was not escalated or investigated
- **No adversarial test cases** — high code coverage but zero authentication bypass scenarios
- **Spec still being written during voting** — integration considerations added 2 days before the vulnerability disclosure
- **No security gate before the amendment entered validator voting** — the last chance to catch issues before an irreversible mainnet activation

For a financial ledger where the amendment process is effectively irreversible once activated, these gaps are significant. The vulnerability was caught by luck — an external researcher and AI tooling during the voting window — rather than by a security gate built into the development process.

## Sources

- [XLS-0056 Specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0056-batch)
- [XRPL-Standards PR #452 — Integration Considerations](https://github.com/XRPLF/XRPL-Standards/pull/452)
- [PR #5060 — Batch (XLS-56)](https://github.com/XRPLF/rippled/pull/5060)
- [PR #6069 — fix: Inner batch transactions never have valid signatures](https://github.com/XRPLF/rippled/pull/6069)
- [XRPL Vulnerability Disclosure Report (Feb 2026)](https://xrpl.org/blog/2026/vulnerabilitydisclosurereport-bug-feb2026)
- [XRPL batch amendment security patch blocks mainnet risk](https://en.cryptonomist.ch/2026/02/27/xrpl-batch-amendment-security/)
