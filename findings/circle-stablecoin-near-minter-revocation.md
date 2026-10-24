# Circle — stablecoin-near: `remove_controller` fails to revoke the minter's `Role::Minter` and `minter_allowed`, leaving orphaned minters with retained minting capability

| Field | Value |
|---|---|
| **Program** | [Circle](https://hackerone.com/circle-bbp) — https://hackerone.com/circle-bbp |
| **Asset** | `https://github.com/circlefin/stablecoin-near` (Smart contract, in scope) |
| **Component** | `src/fiat_token.rs`, `src/fiat_token_action.rs` |
| **Versions tested** | `circlefin/stablecoin-near` @ `main` (2026-10-24); Rust / NEAR SDK, `near-sdk-contract-tools` |
| **Weakness** | CWE-459: Incomplete Cleanup (per HackerOne pre-submission review) |
| **Severity** | Low · CVSS 4.0 = 2.1 |
| **Status** | 🟡 Submitted 2026-10-24 · passed HackerOne pre-submission review |

---

## Summary

`FiatToken::remove_controller` documents that it *"Disables the controller by revoking its Controller Role and removing its minter."* The implementation only:

1. revokes `Role::Controller` and `Role::Multisig` from the controller account, and
2. deletes the `controllers` mapping entry.

The mapped minter's `Role::Minter` and its `minter_allowed` entry are **not** touched. Because `mint()` authorizes solely on `Role::Minter` plus a sufficient `minter_allowed` balance, a minter remains fully able to mint after its controller has been removed.

The same gap occurs on controller re-pointing: `configure_controller(C, M2)` overwrites the mapping for controller `C`, but the previous minter `M1` keeps `Role::Minter` and its allowance.

There is no MasterMinter-reachable path to revoke a minter directly: the `RemoveMinter` action requires `Role::Controller` (`role_required()` in `fiat_token_action.rs`), and the controller-ownership check in `approve_multisig_request` / `execute_multisig_request` compares the caller's mapped minter against the request's controller. Once the mapping is gone, no account satisfies that check for the orphaned minter.

## Steps to reproduce

1. Initialize the contract with MasterMinter `MM`, controller `C`, minter `M`.
2. `MM` calls `configure_controller(C, M)`; `C` calls `configure_minter_allowance(1_000_000)` through the multisig request flow.
3. `MM` calls `remove_controller(C)`.
4. Observe: `is_minter(M)` returns `true` and `minter_allowance(M)` returns `1_000_000` — the minter was not disabled, contradicting the documented behavior.
5. `M` calls `mint(recipient, 1_000_000)` — the mint succeeds despite the controller having been removed.

Rotation variant:

1. `MM` calls `configure_controller(C, M1)`; `C` sets `M1`'s allowance to `500_000`.
2. `MM` calls `configure_controller(C, M2)` (minter key rotation).
3. `M1` remains an active minter (`is_minter(M1) == true`, `minter_allowance(M1) == 500_000`) and can still mint.

A runnable Rust test demonstrating both scenarios uses the contract's existing unit-test harness:

```rust
#[test]
fn testPoc_remove_controller_does_not_disable_minter() {
    // Arrange: controller controls minter with a 1,000,000 allowance.
    let mut usdc: Contract = init_contract();
    set_caller(controller());
    usdc.configure_minter_allowance(U128::from(1_000_000));

    // Sanity: minter is active.
    assert!(usdc.is_minter(&minter()));
    assert_eq!(usdc.minter_allowance(&minter()), U128::from(1_000_000));

    // Act: MasterMinter removes the controller.
    set_caller(master_minter());
    usdc.remove_controller(controller());

    // Assert (BUG): minter role and allowance both survive, contradicting the
    // remove_controller doc ("removing its minter").
    assert!(
        usdc.is_minter(&minter()),
        "BUG: minter role survives remove_controller"
    );
    assert_eq!(
        usdc.minter_allowance(&minter()),
        U128::from(1_000_000),
        "BUG: minter allowance survives remove_controller"
    );

    // And the orphaned minter can still mint.
    let recipient: AccountId = "recipient".parse().unwrap();
    init_account(&mut usdc, recipient.clone(), None);
    set_caller(minter());
    usdc.mint(recipient.clone(), U128::from(1_000_000));
    assert_eq!(
        usdc.ft_balance_of(recipient).0,
        1_000_000,
        "BUG: orphaned minter mints after controller removal"
    );
}

#[test]
fn testPoc_configure_controller_rotation_keeps_old_minter() {
    // Arrange.
    let mut usdc: Contract = init_contract();
    let minter1: AccountId = "minter1".parse().unwrap();
    let minter2: AccountId = "minter2".parse().unwrap();

    set_caller(master_minter());
    usdc.configure_controller(controller(), minter1.clone());
    set_caller(controller());
    usdc.configure_minter_allowance(U128::from(500_000));

    // Act: rotate the controller's minter to minter2.
    set_caller(master_minter());
    usdc.configure_controller(controller(), minter2.clone());

    // Assert (BUG): the old minter is orphaned but still active.
    assert!(
        usdc.is_minter(&minter1),
        "BUG: old minter survives controller re-pointing"
    );
    assert_eq!(
        usdc.minter_allowance(&minter1),
        U128::from(500_000),
        "BUG: old minter allowance survives controller re-pointing"
    );
}
```

Run with: `cargo test testPoc_ -- --nocapture`

## Evidence

**`remove_controller` performs no minter teardown** — `src/fiat_token.rs` (lines 221–231):

```rust
/// Disables the controller by revoking its Controller [`Role`] and removing its minter.
/// Only callable by a MasterMinter.
fn remove_controller(&mut self, controller_id: AccountId) {
    require_only(Role::MasterMinter);
    self._revoke_multisig_role(&controller_id, &Role::Controller);

    if self.controllers.contains_key(&controller_id) {
        self.controllers.remove(&controller_id);
        fiat_token_event::ControllerRemoved { controller_id }.emit();
    } else {
        env::panic_str("FiatToken: controller does not exist");
    }
}
```

**`remove_minter` is the only real teardown — and only the controller can call it** — `src/fiat_token.rs` (lines 250–255):

```rust
/// Disables a controller's minter. This function can only be called by a controller.
fn remove_minter(&mut self) {
    let minter_id: AccountId = self.get_minter().clone();
    Rbac::remove_role(self, &minter_id, &Role::Minter);
    self.minter_allowed.remove(&minter_id);
    fiat_token_event::MinterRemoved { minter_id }.emit();
}
```

**`mint()` authorizes on role + allowance only** — `src/fiat_token.rs` (lines 406–435):

```rust
pub fn mint(&mut self, to: AccountId, amount: U128) {
    require!(!self.paused, "FiatToken: paused");
    require_only(Role::Minter);
    // ... blocklist checks, amount > 0 ...
    let new_minter_allowance: u128 = self.minter_allowed
        .get(&env::predecessor_account_id())
        .unwrap_or(&U128::from(0))
        .0
        .checked_sub(amount.0)
        .unwrap_or_else(|| env::panic_str("FiatToken: mint amount exceeds minter allowance"));
    // ... internal_deposit, allowance decrement, FtMint event ...
}
```

**The controller-ownership check locks out recovery** — `src/fiat_token.rs` (lines 648–662) and `src/fiat_token_action.rs` (lines 72–100):

```rust
// approve_multisig_request / execute_multisig_request / remove_multisig_request:
if request.action.requires_controller_check() {
    require!(
        self.controllers.get(request.action.controller()).unwrap()
            == self.controllers.get(&env::predecessor_account_id()).unwrap(),
        "FiatToken: can only approve requests to configure the allowance of your own minter"
    );
}
```

```rust
// role_required():
FiatTokenAction::ConfigureMinterAllowance { .. }
| FiatTokenAction::DecreaseMinterAllowance { .. }
| FiatTokenAction::IncreaseMinterAllowance { .. }
| FiatTokenAction::RemoveMinter { .. } => Role::Controller,
```

**`configure_controller` overwrites without teardown** — `src/fiat_token.rs` (lines 203–216):

```rust
fn configure_controller(&mut self, controller_id: AccountId, minter_id: AccountId) {
    require_only(Role::MasterMinter);
    require_not_blocklisted(&controller_id);
    require_not_blocklisted(&minter_id);
    self._grant_multisig_role(controller_id.clone(), &Role::Controller);
    self.controllers.insert(controller_id.clone(), minter_id.clone());
    // ... event ...
}
```

**Test coverage gap** — the unit tests exercise `remove_controller` (mapping removal + event) and `remove_minter` (controller path) but never assert the minter's role or allowance state after `remove_controller` or controller re-pointing.

## Impact

1. **Minter key rotation is ineffective.** Rotating a controller's minter (`M1` → `M2`) leaves the retired minter `M1` able to mint up to its remaining allowance. If `M1`'s key is retired, leaked, or compromised, the documented control (re-point or remove the controller) does not cut off its minting power.

2. **Decommissioned integrations keep minting.** An automated minter (e.g. a bridge mint leg) continues minting after Circle removes its controller, up to the remaining allowance — unintended USDC supply inflation on NEAR.

3. **Bounded impact.** The orphaned minter cannot exceed its remaining `minter_allowed` and cannot raise it (allowance changes require a live controller mapping). Recovery is possible but non-obvious: re-point a fresh controller at the orphaned minter, then call `remove_minter()`.

## Remediation

In `remove_controller`, tear down the mapped minter before deleting the mapping:

```rust
if let Some(minter_id) = self.controllers.get(&controller_id) {
    Rbac::remove_role(self, &minter_id, &Role::Minter);
    self.minter_allowed.remove(&minter_id);
}
```

Apply the same teardown to the previously mapped minter in `configure_controller` before overwriting the entry.

## Process notes

- Source-code finding in the in-scope `circlefin/stablecoin-near` repository. No live system touched — no testnet RPC calls, no transactions broadcast, no accounts created.
- No scanners used: manual review of `fiat_token.rs` and `fiat_token_action.rs`.
- The PoC test was written against the contract's own unit-test harness but **not executed** in the reporter's environment (no Rust toolchain available at report time); the behavior is derived directly from source review. This limitation was disclosed in the submitted report.
- Severity calibrated Low per HackerOne's own CVSS 4.0 calculator (2.1) and confirmed by pre-submission review; weakness reclassified to CWE-459 per reviewer suggestion.
