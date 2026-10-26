# 1WIN — Key Separation Violation: Shared Upgrade/Mint Authority + Active Freeze Authority Enables Permanent Freezing of User Funds

**Program:** [1win](https://hackerone.com/1win_com) (HackerOne)
**Campaign:** $1WIN Security Campaign
**Severity:** High · 8.3 (CVSS 4.0) — self-assessed, pending triage
**Status:** 🟡 Submitted 2026-10-26
**Assets:** Solana mainnet-beta
- Sale: `GWvQinGBRa7ZUzhXY23MrLjdsx3KfRGxH8yXURBwrz1R`
- Vesting: `sxzuA7TuRvjVbrXkJUgrYq2hfjVf3RFd8WFaSxt2rYm`
- Burn: `B2RFzpckwjtHXHMrPMKarg6Cj6xkSdT6mru4gJiJE87L`
- Token mint: `FjPKUGnF5zBpdEv18gT8FJCtxh3MKrFkoeYekgYr1win`

**Source:** https://github.com/1win-token/1win-token-solana-programs @ `68c301d`

---

## Summary

The $1WIN token ecosystem concentrates critical authorities in single externally-owned keys with no multisig or timelock:

1. **One key (`4K5xxirCw8WD5uKGcu5kP3qdh5ZkBanNbm6BVoKD8vcf`) holds both the program upgrade authority (all three programs) and the token mint authority.**
2. **A second key (`G68zSDguycN8BfZLtpQRBDbkBaw62aaMV5EpdoaqsEVm`) holds the SPL freeze authority**, which can freeze any token account of the mint — including the program vaults that hold user funds.
3. The mint authority is still active (supply is currently 0, pre-TGE), so the litepaper's "fixed total supply of one billion $1WIN, no additional minting after TGE" is not enforced on-chain.

The freeze authority alone enables a **permanent freeze of all user funds**: freezing the sale token vault and refund vault, plus the six vesting vaults, makes every claim and refund instruction fail permanently (SPL Token rejects transfers from frozen accounts).

## Approach

1. **Source audit** — cloned the full repo (127 Rust files, 7,216 LOC across Sale/Vesting/Burn) and reviewed every instruction, the math cores, the state machines, and the access controls. Result: no exploitable code findings (consistent with the prior Halborn + CertiK audits; the 9 post-audit commits of 2026-09-16 were reviewed individually).
2. **On-chain verification** — queried Solana mainnet-beta via public JSON-RPC (`getAccountInfo`, `getProgramAccounts`, `getTokenAccountBalance`) to establish the live authority configuration and fund exposure.

## Evidence

### Upgrade authority (all three programs, same key)

| Program | ProgramData | Upgrade authority | Last upgrade slot |
|---|---|---|---|
| Sale | `FAbd3kdHEw4SKgXgsLhRP7bdwN8V5e7WhpWGakSoGGGX` | `4K5xxirCw8WD5uKGcu5kP3qdh5ZkBanNbm6BVoKD8vcf` | 456397295 |
| Vesting | `7rwZryLt3iL7EkeyYt1GQHerJaHWA4MmMQ4sgYisRSeA` | `4K5xxirCw8WD5uKGcu5kP3qdh5ZkBanNbm6BVoKD8vcf` | 456424390 |
| Burn | `7JB334nNKZmwCLp2xPBgwspNeLFWaaAM3G8mwVtTE9fg` | `4K5xxirCw8WD5uKGcu5kP3qdh5ZkBanNbm6BVoKD8vcf` | 456421370 |

### Token mint

```
mint:             FjPKUGnF5zBpdEv18gT8FJCtxh3MKrFkoeYekgYr1win
decimals:         8
supply:           0
mint_authority:   4K5xxirCw8WD5uKGcu5kP3qdh5ZkBanNbm6BVoKD8vcf   (SAME as upgrade authority)
freeze_authority: G68zSDguycN8BfZLtpQRBDbkBaw62aaMV5EpdoaqsEVm  (SAME as admin role)
```

### Sale state (live)

```
SaleState:   E9Sdba46tH8KD9PPwZngGwkHVnmrbKL64Qz8PnBMiiAL
deposit_vault (HciAS69vfU1ZzfWEczqwkAUfLodpbGFVALLpA1wtA5W2): 6,284,101.69 USDC actual
bookkeeping deposits_usdc_contributed: 6,284,080.95 USDC (2,633 participants)
token_vault (7zWFaaSu9VgAoWPgXYmgMiS4tNG1gDv4kzHUkbiK4y98): 0 $1WIN (pre-TGE)
```

### Vesting (6 activities, all claimed = 0, TGE 2026-10-30T10:00Z)

| Activity | Reference | TGE bps | Monthly bps |
|---|---|---|---|
| 1 | 80,000,000 | 100% | 0 |
| 2 | 250,000,000 | 10% | 5% |
| 3 | 200,000,000 | 10% | 7.5% |
| 4 | 180,000,000 | 0% | 8.34% |
| 5 | 60,000,000 | 20% | 8% |

## Attack Paths

### Path 1 — Permanent freeze of all user funds (freeze authority)

1. The freeze authority calls SPL Token `freeze` on the program vault token accounts: the sale `token_vault`, the sale `refund_vault` (created at `activate_claim`), and the six vesting vaults (seeds `[b"vault", activity_id]`).
2. SPL Token rejects every transfer from a frozen account.
3. Consequences, all permanent until unfrozen:
   - Sale `claim` (`programs/sale/src/instructions/claim.rs:112`): `transfer_from_vault(token_vault → wallet_token_ata)` fails → no buyer can ever receive purchased tokens.
   - Sale refund leg (`claim.rs:134`): `transfer_from_vault(refund_vault → wallet_usdc_ata)` fails → oversubscribed buyers cannot recover unused USDC.
   - Vesting `claim` (`programs/vesting/src/instructions/claim.rs:94`): `token::transfer(vault → recipient_ata)` fails → all six vesting recipients are permanently blocked.
4. No instruction in either program detects or mitigates a frozen vault; the failure surfaces as an opaque SPL error on every claim.

### Path 2 — Single-key total compromise (upgrade + mint)

Compromise of `4K5xx...` yields simultaneously:
- **Program replacement**: upgrade all three programs — e.g. an upgraded Sale routing the 6.28M USDC deposit vault to an attacker address, or an upgraded Vesting reassigning every `recipient`.
- **Unlimited minting**: mint arbitrary $1WIN to any address — the supply is not capped on-chain.

### Path 3 — Mint authority never renounced

Supply is currently 0 and the mint authority is active. If not renounced after the initial mint, the "fixed 1B supply" claim is unenforceable.

## Impact

- **User funds at risk:** 6,284,101.69 USDC (sale deposits, 2,633 participants) + 770M+ $1WIN in vesting allocations for user-facing buckets (games 250M, liquidity 200M, wager rewards 180M, TMA airdrop 80M, marketing 60M).
- **Failure mode:** permanent, unilateral, and silent — claims simply fail with an SPL error; users have no on-chain recourse.
- **Trigger:** compromise (or misuse) of a single EOA. No multisig, no timelock, no on-chain constraint.

## Remediation

1. **Separate the authorities.** Move program upgrade authority to a multisig (e.g. Squads) with a timelock; never share it with the mint authority.
2. **Renounce the freeze authority** (`spl-token authorize --disable` for freeze) before TGE, or move it to the same multisig. Vesting/sale vaults must never be freezable by a hot key.
3. **Renounce the mint authority** immediately after minting the fixed 1B supply, so the supply cap is enforced on-chain rather than by policy.
4. **Fail-closed claim handling:** explicit frozen-vault check with a distinct error code (defense in depth; the primary fix is 1–3).

## Notes

- All evidence obtained via public Solana JSON-RPC; read-only, no transactions sent.
- The source audit found no exploitable code defects — the finding is architectural (authority concentration), not a code bug.
- CVSS 4.0: `AV:N/AC:L/AT:N/PR:H/UI:N/VC:N/VI:H/VA:H/SC:N/SI:N/SA:H` = 8.3 High (self-assessed).
