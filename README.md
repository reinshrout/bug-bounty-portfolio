# 🔍 Bug Bounty Portfolio — reinshrout

Security researcher focused on **smart contract auditing**, **source-code review**, and **anti-bot protection research**. I find logic flaws that scanners miss, verify every finding on-chain before reporting, and calibrate severity honestly — no over-claiming.

**GitHub:** [reinshrout](https://github.com/reinshrout)
**HackerOne:** [reinshrout22](https://hackerone.com/reinshrout22)
**Focus:** Solidity · Rust · Go · TypeScript
**Method:** source-code-first, PoC-verified, honest severity

---

## 📊 Track Record

| # | Program | Target | Vulnerability | Severity | Status |
|---|---------|--------|---------------|----------|--------|
| 1 | [Arc (Circle)](https://hackerone.com/arc-bbp) | `circlefin/arc-node` | `ValidatorRegistry._nextRegistrationId` defaults to 0 — first validator receives unusable registration ID, locked out of controller assignment and voting power | Low · CVSS 3.8 · CWE-16 | 🟡 Submitted 2026-10-23 · Report #4141500 |
| 2 | [Anthropic](https://hackerone.com/anthropic) | `modelcontextprotocol/typescript-sdk` | `$async` JSON Schema bypasses client output-schema validation and crashes the client via unhandled rejection; ReDoS in peer-supplied `pattern` | Medium · CVSS 5.3 · CWE-20 | 🟡 Submitted |
| 3 | CrowdProof Protocol | `crowdpay/crowdproof-protocol` | PaymentEscrow timelock withdrawal does not bind recipient or funding source — announced recipient never paid, arbitrary sink drained | Medium | ✅ Verified on-chain · disclosure channel inactive |
| 4 | [Circle](https://hackerone.com/circle-bbp) | `circlefin/stablecoin-near` | `remove_controller` revokes the controller's roles and mapping but not the minter's `Role::Minter` / `minter_allowed` — orphaned minters retain minting capability after controller removal or key rotation | Low · CVSS 4.0 2.1 · CWE-459 | 🟡 Submitted 2026-10-24 · pre-submission review passed |
| 5 | Paxos (Cantina) | `paxos-token-contracts` | `_internalUnregisterRewardAddress` uses stale shares after `claimAll` early-return — payout-group shares over-decremented, rewards permanently lost | Medium | ✅ PoC verified 3/3 (Hardhat) |
| 6 | [DataDome (YesWeHack)](https://yeswehack.com/programs/datadome-bot-bounty) | `bounty-nginx.datashield.co` | Anti-bot protection bypass — automated JS-challenge solving via real-browser automation + session rotation; 4,847 verified requests before IP-level volume cap | Research | 🔬 Bypass proven · 4,847 verified |

---

## 📝 Writeups

### 1. Arc (Circle) — ValidatorRegistry registration ID 0 sentinel collision

`registerValidator()` assigns IDs via post-increment from an uninitialized counter. The contract's own docs say the counter starts at 1, but no on-chain code enforces it — the invariant lives in an off-chain genesis storage poke. Miss that step and the first validator gets ID 0, which is the reserved "not a controller" sentinel everywhere else in the suite: permanently locked at zero voting power, no consensus influence, no recovery function. Includes a 4-test Foundry PoC adapted from the project's own test file.

📄 [`findings/arc-validator-registry-id-zero.md`](findings/arc-validator-registry-id-zero.md)

### 2. Anthropic — MCP SDK AJV `$async` validation bypass + client crash

`AjvJsonSchemaValidator` wraps AJV's compiled validator with `valid ? {valid: true} : …`. A schema containing `$async: true` compiles to an async validator returning a **Promise** — always truthy, so every input passes validation, and the rejection that follows is never handled. Because MCP servers supply the output schema, a malicious server can disable the client's only containment check and kill the host process via unhandled rejection. A second class measured in the same path: attacker-supplied `pattern` blocks the event loop for 55 seconds.

📄 [`findings/anthropic-mcp-ajv-async-bypass.md`](findings/anthropic-mcp-ajv-async-bypass.md)

### 3. CrowdProof — Timelock treasury withdrawal does not bind recipient

The treasury timelock announces an amount and a recipient at queue time, but `executeTreasuryWithdrawal` pays whatever recipient is passed at execution time. Verified on a local anvil instance: `queueTreasuryWithdrawal(A, 9e15)` followed by `executeTreasuryWithdrawal(1, B)` paid **B**, not A — announced recipient received 0, arbitrary sink received the full amount. A second test showed the same flaw on the escrow subject path (4e15 redirected to an unannounced sink while the treasury stayed intact).

📄 [`findings/crowdproof-timelock-recipient-not-bound.md`](findings/crowdproof-timelock-recipient-not-bound.md)

### 4. Circle — `remove_controller` leaves orphaned minters able to mint (stablecoin-near)

The doc says `remove_controller` "removes its minter" — the code only deletes the controller mapping. `Role::Minter` and the `minter_allowed` allowance survive, and `mint()` checks nothing else. Net effect: minter key rotation (`M1` → `M2`) and decommissioning are both no-ops on the old minter, which keeps minting up to its remaining allowance. No MasterMinter-reachable recovery exists, because `RemoveMinter` requires the very controller mapping that was just deleted. Found by reading 8 Rust files end to end; classified CWE-459 (Incomplete Cleanup) per HackerOne review, scored 2.1 (Low) on their own CVSS 4.0 calculator.

📄 [`findings/circle-stablecoin-near-minter-revocation.md`](findings/circle-stablecoin-near-minter-revocation.md)

### 5. Paxos — Payout-group share corruption after `claimAll` + `unregisterRewardAddress`

`_internalUnregisterRewardAddress` reads the user's stored shares to decrement the payout group's total. But when `claimAll` takes its early-return path (`rewards == 0`), the adjusted shares are never persisted — so the unregister that follows operates on **stale shares**. The group total is over-decremented, the user under-claims, and repeated cycles clamp group shares to zero: permanent reward loss. Hardhat PoC: 3/3 tests passing — bug scenario lost 0.001999 where the control lost 2 wei of dust, with 25% reward loss at scale.

📄 [`findings/paxos-payout-group-share-corruption.md`](findings/paxos-payout-group-share-corruption.md)

### 6. DataDome — Automated bot-protection challenge bypass + protection profile analysis

DataDome's multi-stage JS challenge (403 → `c.js` → captcha iframe → interstitial fingerprinting) is solvable by an automated real-browser client: Playwright driving system Edge with two stealth patches clears the full chain in ~5-6 seconds with zero human interaction. Session rotation (~90 requests per solved cookie) sustains scraping; the measured IP-level cap is ~1,000 requests per flag-window with escalating cooldowns. 4,847 requests verified with HTTP 200 + `pagehash` flags from a single disclosed IP before the run was stopped at 16% of the 30K target — the IP volume cap, not the bypass, is the binding constraint.

📄 [`findings/datadome-bot-protection-bypass.md`](findings/datadome-bot-protection-bypass.md)

---

## 🛠 Methodology

Full details in [`methodology.md`](methodology.md). The short version:

1. **Program selection** — newly launched or low-competition programs where source code is in scope. Web surfaces get 3,000+ reports per quarter; source repos get a handful. Go where the scanners aren't.
2. **Source-code-first** — clone the repo, read the core contracts/services end to end before touching any live endpoint. The $200K–$1M tier lives in consensus-critical logic, not in missing headers.
3. **Sink identification** — map every trust boundary first (who supplies what data to whom), then audit the sinks: signature construction, access-control sentinels, initialization paths, schema validators.
4. **PoC before claim** — every finding gets a runnable proof (Foundry test, standalone script) executed locally. If it doesn't run, it doesn't get reported.
5. **Honest severity** — check for recovery paths and compensating controls before claiming impact. An accurate Low builds Signal; an inflated Medium burns it.

---

## 📫 Contact

- GitHub: [github.com/reinshrout](https://github.com/reinshrout)
- HackerOne: [hackerone.com/reinshrout22](https://hackerone.com/reinshrout22)
- Reports are submitted through official program channels only.

---

*Portfolio maintained by reinshrout. All testing performed against in-scope assets under program rules; no live systems harmed.*
