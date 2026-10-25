# Payout Group Shares Accounting Corruption on Unregister After claimAll

**Program:** Paxos Bug Bounty (Cantina) — https://cantina.xyz/bounties/6a6ef71c-383d-4357-85c1-a0d1dbb6659b
**Asset:** Core Smart Contracts — `paxosglobal/paxos-token-contracts` (Claimable On-Chain Rewards, USDG)
**Severity:** Medium (loss of yield; no theft — claim source is under-drained, not over-drained)
**Status:** PoC verified — 3/3 Hardhat tests passing

---

## Summary

`_internalUnregisterRewardAddress` in `contracts/facets/PayoutGroupFacet.sol` removes an account from a payout group using the account's **stored** shares. When the account was affected by a `claimAll` that occurred in the same reward period (no multiplier growth between the claimAll and the unregister), the stored shares are **stale** — they still include the rewards that the claimAll already claimed. The group's aggregate shares are therefore over-decremented, drifting below the true sum of its members' shares. Every subsequent `claimAll` for that group under-claims rewards, and with enough affected unregisters the group's shares are clamped to zero by `_applyDelta`, permanently destroying all remaining members' rewards.

## Root Cause

Two functions interact:

**1. `ClaimableRewardsBase._claimIndividualRewardsWithData` (ClaimableRewardsBase.sol:326–378)**

```solidity
uint256 shares = SharesLib.handleClaimAllDetection(
    uint256(wallet.balance), uint256(wallet.shares), wallet.lastUpdateTime,
    payoutGroup.lastClaimAllTime, uint256(payoutGroup.lastClaimAllMultiplier)
);
uint256 rewards = SharesLib.calcRewards(shares, currentMultiplier, uint256(wallet.balance));

if (rewards == 0) {
    return 0;   // <-- EARLY RETURN: adjusted shares are DISCARDED, never persisted
}

// Only reached when rewards > 0:
uint256 newShares = SharesLib.calcShares(uint256(wallet.balance), currentMultiplier);
_setBalanceData(account, ..., newShares, ...);
```

When a claimAll has zeroed an account's rewards, `calcRewards` returns 0 and the function returns early. The claimAll-adjusted shares (computed in memory by `handleClaimAllDetection`) are never written to storage.

**2. `PayoutGroupFacet._internalUnregisterRewardAddress` (PayoutGroupFacet.sol:768–803)**

```solidity
_claimIndividualRewards(account, destination);          // may early-return (rewards == 0)

TokenAccountData memory updatedWallet = _getBalanceData(account);
int256 sharesDelta = -int256(uint256(updatedWallet.shares));   // <-- STALE shares
_updatePayoutGroupBals(payoutGroupId, payoutGroup, balanceDelta, sharesDelta);
```

The sharesDelta is computed from the wallet's stored shares. If the claim early-returned, those shares are the pre-claimAll (stale) values, which are **larger** than the claimAll-adjusted shares by exactly the rewards the claimAll already paid out.

**3. `_applyDelta` silently clamps underflow (ClaimableRewardsBase.sol:206–213)**

```solidity
return current >= decrease64 ? current - decrease64 : 0;   // <-- clamped to 0
```

Repeated over-decrement drives the group's shares toward zero without reverting.

## Trigger Sequence

1. Payout group `G` has members with balances and shares; multiplier has grown (rewards accrued).
2. `claimAll(G)` — claims **all** group rewards to the configured destination; resets group shares to balance-equivalent; sets `lastClaimAllTime` and `lastClaimAllMultiplier`.
3. `unregisterRewardAddress(G, A)` (or `registrarUnregisterRewardAddress`, batch variant, or a user-initiated payout-group switch via `registerRewardAddress`) **in the same reward period** — no multiplier growth between steps 2 and 3.
4. A's internal claim computes `rewards == 0` (the claimAll already claimed them) → early return → A's shares stay stale.
5. Group shares are decremented by A's stale shares → corrupted.

## Verified PoC

File: `test/PoCUnregisterClaimAllShareCorruption.js` (runs against the repo's own Hardhat setup, `npx hardhat test`).

**Test 1 — BUG (passing):**

```
Phase 1 — rewards accrued:  A: 0.001999  B: 0.003999  C: 0.005999
Phase 2 — claimAll #1 claimed: 0.011999
Phase 3 — unregistered A (same period as claimAll)
Phase 5 — expected rewards for B+C (per-wallet): 0.009996
Phase 6 — claimAll #2 ACTUALLY paid:              0.007997
>>> REWARDS LOST TO REMAINING MEMBERS:            0.001999
```

The loss (0.001999) equals exactly A's pre-claimAll reward — the amount double-counted into the stale shares.

**Test 2 — CONTROL (passing):** identical flow but with multiplier growth between claimAll and unregister (so A's claim returns > 0 and shares are persisted correctly). Difference between expected and actual: **2 base units (2e-6 tokens)** — pure integer-truncation dust. This isolates the bug to the claimAll + same-period-unregister sequence.

**Test 3 — ESCALATION (passing):** batch-unregistering two of three members after a claimAll:

```
C's true rewards:      0.023995
group claimAll paid:   0.017990
loss:                  ~25% of the remaining member's rewards
```

With a large group (the design targets "batch claiming thousands of wallets"), repeated affected unregisters clamp the group's shares to zero and **all** remaining rewards are permanently lost.

## Impact

- **Loss of yield** for remaining payout group members: their accrued rewards are under-claimed on every subsequent `claimAll`.
- **Permanent loss** at scale: enough affected unregistrations zero the group's shares; unclaimed rewards remain locked in the claim source with no path to distribution.
- **No theft**: the claim source is under-drained. The bug destroys value rather than redirecting it.
- **Triggerable by**: `PAYOUT_GROUP_REGISTRAR_ROLE` (batch unregister), payout group claimer/manager (`unregisterRewardAddress`), or any user switching payout groups via signature (`registerRewardAddress` → auto-unregister from the old group). Admin automation that unregisters accounts in the same block/period as a claimAll triggers it unintentionally.

## Affected Code

- `contracts/ClaimableRewardsBase.sol` — `_claimIndividualRewardsWithData` (early return discards adjusted shares), `_applyDelta` (silent underflow clamp)
- `contracts/facets/PayoutGroupFacet.sol` — `_internalUnregisterRewardAddress` (uses stale shares for sharesDelta)
- Feature introduced in commit `74999fc` ("Claimable On-Chain Rewards", 2026-03-04); audited by Zellic + Halborn (reports in `audit-reports/`), but no test combines `claimAll` with unregister — verified by grep over `test/`.

## Recommended Fix

Persist the claimAll-adjusted shares even when `rewards == 0`, so any later reader (including the unregister path) sees consistent state. Minimal patch in `_claimIndividualRewardsWithData`:

```solidity
if (rewards == 0) {
    // Still persist claimAll-adjusted shares so hierarchical accounting stays consistent
    if (shares != uint256(wallet.shares)) {
        _setBalanceData(account, uint256(wallet.balance), shares,
                        uint256(wallet.payoutGroupId), uint40(block.timestamp));
    }
    return 0;
}
```

Alternatively, `_internalUnregisterRewardAddress` can recompute the adjusted shares itself via `SharesLib.handleClaimAllDetection` before computing the sharesDelta.

A regression test should cover: claimAll → unregister (same period) → multiplier growth → claimAll, asserting the group claim equals the sum of remaining members' `availableRewardsOf`.
