# Methodology — How I Find Bugs

## 1. Program Selection

Not all programs are worth the time. My filter, in order:

| Criterion | Why |
|---|---|
| **Newly launched** (< 90 days) | Low competition, triagers hungry, scope fresh |
| **Source code in scope** | Logic bugs pay 10–100x more than config issues |
| **Crypto / L1 / L2 / infra** | My strongest domain; consensus-critical logic is under-audited |
| **Responsive triage** | Check average time-to-first-response and bounty stats before investing |

Web-only programs with thousands of reports per quarter get deprioritized — the easy XSS and IDOR are long gone.

## 2. Source-Code-First Auditing

Before any live testing:

1. **Clone and inventory** — count files, identify the core modules (contracts, signing, validator management, RPC handlers)
2. **Read the threat model** — README, SECURITY.md, docs. What does the system claim to guarantee? Every claim is a testable invariant.
3. **Read end to end** — the core paths, not just the entry points. Bugs live in the assumptions between components.

## 3. Sink Identification

Map trust boundaries first: who supplies what data to whom. Then audit the sinks where attacker-influenced data meets security decisions:

- **Signature construction** — what exactly gets signed? Is chain ID committed? Is the domain separator present?
- **Access-control sentinels** — what values mean "none"? Does a default state collide with a sentinel?
- **Initialization paths** — what happens on a fresh deployment? Which invariants are enforced on-chain vs. assumed?
- **Schema/data validators** — what happens when the validator itself receives hostile input?
- **Economic flows** — what is announced vs. what is actually paid/executed?

## 4. PoC Before Claim

Every finding gets a runnable proof executed locally before it goes into a report:

- **Solidity** — Foundry test, adapted from the project's own test harness where possible (proves the bug against their setup, not mine)
- **Node/TS** — standalone script against the SDK's public API
- **State verification** — anvil for on-chain state transitions, real tx hashes, real balance diffs

If it doesn't run, it doesn't get reported. A PoC that passes is the difference between "possible issue" and "demonstrated vulnerability."

## 5. Honest Severity Calibration

Before claiming impact, check:

- **Recovery paths** — can the victim work around it? (e.g., re-register after a burned ID)
- **Compensating controls** — is there an off-chain process that prevents the bad state?
- **Blast radius** — one component or the whole system?
- **Adversary model** — is there actually an attacker, or is this a latent defect?

An accurate Low with a thorough writeup builds Signal and reputation. An inflated Medium that gets downgraded burns both. Let the triager upgrade you — that's the win.

## 6. Report Craft

- **One vulnerability per report** — always
- **Exact versions/commits** — programs need reproducibility
- **Evidence with line numbers** — quote the code, cite the lines
- **Adapted PoCs** — build on the project's own tests to show the bug in their harness
- **Process notes** — state what you did and didn't touch (scope compliance builds trust)
