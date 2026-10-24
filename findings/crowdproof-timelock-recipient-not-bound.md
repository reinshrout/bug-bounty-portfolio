# CrowdProof Protocol — PaymentEscrow timelock withdrawal does not bind recipient or funding source

| Field | Value |
|---|---|
| **Program** | CrowdProof Protocol (private engagement) |
| **Asset** | `github.com/crowdpay/crowdproof-protocol` @ `92a4498` |
| **Component** | `contracts/src/PaymentEscrow.sol` (treasury timelock + escrow subject paths) |
| **Versions tested** | Foundry 1.8.5, solc 0.8.24, OpenZeppelin v5.1.0; verified on local anvil (chain-id 31337) |
| **Weakness** | CWE-20: Improper Input Validation (execution-time parameters not bound to queue-time commitments) |
| **Severity** | Medium |
| **Status** | ✅ Verified on-chain · disclosure channel inactive (security contact has no MX, private reporting disabled, bounty "coming soon") |

---

## Summary

The treasury timelock announces an amount and a recipient at **queue time**, but `executeTreasuryWithdrawal` pays whatever recipient is passed at **execution time**. The announced recipient is never bound to the execution — the queue record is decorative. Any party able to trigger execution can redirect the payment to an arbitrary address.

The same flaw exists on the escrow subject path: a queued payment for one subject can be executed to a different sink while the treasury balance stays intact, making the redirection invisible to balance-based monitoring.

## Steps to reproduce

1. Deploy the protocol contracts to a local anvil instance (chain-id 31337) using the project's own deploy scripts (`DeployAll.s.sol`, `PostDeploy.s.sol`, `TransferOwnership.s.sol`).
2. Queue a treasury withdrawal announcing recipient **A** for 9e15 wei:
   ```
   queueTreasuryWithdrawal(A, 9e15)   → tx 0xcce32871…dae11
   ```
3. Execute the withdrawal passing recipient **B**:
   ```
   executeTreasuryWithdrawal(1, B)    → tx 0xed004d1d…2b853
   ```
4. Observe the result:
   - balance of **A** (the announced recipient): **0** — never paid
   - balance of **B** (the arbitrary sink): **9000000000000000 wei** — received the full amount

5. Repeat the pattern on the escrow subject path: a queued 4e15 payment for the subject was executed to sink **C**; subject balance went 4e15 → 0, sink C received 4e15, treasury balance unchanged.

## Evidence

Raw on-chain state from the anvil verification run:

```
queueTreasuryWithdrawal(A, 9e15)      tx 0xcce32871…dae11
executeTreasuryWithdrawal(1, B)       tx 0xed004d1d…2b853

saldo A (diumumkan):  0
saldo B (dibayar):    9000000000000000 wei

escrow subject path:
subject 4e15 → 0
sink C:      4e15
treasury:    utuh
```

## Impact

- **Payment redirection.** Funds announced for one recipient can be executed to any address. The queue record provides no protection.
- **Invisible to balance monitoring.** On the escrow path, the treasury balance is unchanged by the redirection — only per-account balances reveal the swap, and only if someone is watching the right accounts.
- **Trust breakdown.** The timelock's entire purpose — giving observers time to react to an announced payment — is defeated when the announced parameters are not the executed parameters.

## Remediation

1. **Bind the recipient at queue time:** store the recipient in the timelock record and have `executeTreasuryWithdrawal` pay the stored recipient, ignoring any execution-time parameter (or remove the parameter entirely).
2. **Bind the funding source:** record which treasury/escrow pool funds the withdrawal at queue time and validate at execution.
3. **Emit the full commitment:** include the bound recipient in the queued event so off-chain monitoring can verify execution matches announcement.

## Process notes

- All verification performed on a local anvil instance against the project's own deploy scripts. No mainnet or public testnet interaction.
- Finding verified with real transaction hashes and balance diffs, not static analysis alone.
- Report was prepared for submission but the program's disclosure channel was inactive at the time (no MX on the security contact, private reporting disabled, bounty program "coming soon"). Held pending a working channel.
