# Arc (Circle) — ValidatorRegistry `_nextRegistrationId` defaults to 0, causing first validator to receive unusable registration ID

| Field | Value |
|---|---|
| **Program** | [Arc (Circle)](https://hackerone.com/arc-bbp) — https://hackerone.com/arc-bbp |
| **Asset** | `https://github.com/circlefin/arc-node` (Source code, in scope) |
| **Component** | `contracts/src/validator-manager/ValidatorRegistry.sol` |
| **Versions tested** | `circlefin/arc-node` @ `6e764023ee6515fe70573e123ed2db912a7207b4` (2026-09-25); Foundry 1.8.5, solc 0.8.29 |
| **Weakness** | CWE-16: Misconfiguration (missing on-chain initialization of `_nextRegistrationId` creates a fragile dependency on off-chain genesis configuration) |
| **Severity** | Low · CVSS 3.1 `AV:N/AC:L/PR:H/UI:N/S:U/C:N/I:L/A:L` = 3.8 |
| **Status** | 🟡 Submitted 2026-10-23 · Report [#4141500](https://hackerone.com/arc-bbp/reports/4141500) |

---

## Summary

`ValidatorRegistry.registerValidator()` assigns registration IDs using a post-increment counter:

```solidity
// contracts/src/validator-manager/ValidatorRegistry.sol:110
registrationId = $._nextRegistrationId++;
```

The contract's own documentation states the counter starts at 1 (`ValidatorRegistry.sol:59-60`: *"Counter tracking the next available registration ID for new validator registrations, starting from 1"*), and the rest of the contract suite depends on that invariant — but **no on-chain code enforces it**.

`ValidatorRegistry` has no `initialize()` function (it is a genesis-configured predeploy at `0x3600...0002`), so `_nextRegistrationId` defaults to `0` unless the genesis/deployment process explicitly pokes the storage slot. If that off-chain step is missed on any chain instance, the first registered validator receives `registrationId = 0` — and ID 0 is treated as an invalid sentinel everywhere else in the system:

- `Controller.configureController()` reverts with `RegistrationIdIsZero()` for ID 0 (`Controller.sol:122`)
- `Controller._onlyController()` uses `registrationOf[msg.sender] != 0` as the "is a controller" check (`Controller.sol:113`)
- The management tooling also rejects ID 0 (`ValidatorManagement.s.sol:273`: `require(_registrationId != 0, "RegistrationId not found")`)

The result is a validator that is registered but can never be assigned a controller, and therefore can never have its voting power raised above `DEFAULT_VOTING_POWER = 0`. Per the contract's own documentation, a validator with 0 voting power *"will have no influence in consensus"* (`ValidatorRegistry.sol:167-170`).

## Steps to reproduce

1. Clone the in-scope repository at the tested commit:

```bash
git clone https://github.com/circlefin/arc-node.git
cd arc-node
git checkout 6e764023ee6515fe70573e123ed2db912a7207b4
forge install
```

2. Add the following Foundry test to `contracts/test/` (adapted from the project's own `ValidatorRegistryProxy.t.sol` — the only difference is the omission of the manual `_nextRegistrationId = 1` storage poke, simulating a genesis that misses that off-chain step):

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity ^0.8.29;

import {Test} from "forge-std/Test.sol";
import {ValidatorRegistry, ValidatorStatus} from "src/validator-manager/ValidatorRegistry.sol";
import {PermissionedValidatorManager} from "src/validator-manager/PermissionedValidatorManager.sol";
import {IValidatorRegistry} from "src/validator-manager/interfaces/IValidatorRegistry.sol";
import {AdminUpgradeableProxy} from "src/proxy/AdminUpgradeableProxy.sol";

contract RegistrationIdZeroPoC is Test {
    ValidatorRegistry registry;
    PermissionedValidatorManager pvm;

    address controller = address(0xC0FFEE);
    bytes validatorPubKey;

    function setUp() public {
        validatorPubKey = new bytes(32);
        for (uint256 i = 0; i < 32; i++) {
            validatorPubKey[i] = bytes1(uint8(i + 1));
        }

        // Deploy registry behind a proxy with EMPTY init data.
        // ValidatorRegistry has no initialize() function, so nothing on-chain
        // sets _nextRegistrationId. This mirrors a genesis that does not poke
        // the storage slot off-chain.
        ValidatorRegistry impl = new ValidatorRegistry();
        AdminUpgradeableProxy proxy = new AdminUpgradeableProxy(
            address(impl),
            address(this),
            ""
        );
        registry = ValidatorRegistry(address(proxy));

        // Set owner directly (in production this is done via genesis storage;
        // here we simulate it to reach registerValidator).
        // OZ v5 Ownable _owner slot = 0
        vm.store(
            address(registry),
            bytes32(uint256(0)),
            bytes32(uint256(uint160(address(this))))
        );

        // NOTE: unlike ValidatorRegistryProxy.t.sol setUp (line ~101:
        // "Initialize _nextRegistrationId to 1 (slot offset 4 in
        // ValidatorRegistryStorage)"), we deliberately DO NOT poke
        // _nextRegistrationId here.
    }

    /// 1. The counter starts at 0, contradicting the documented invariant.
    function test_counterStartsAtZero() public {
        assertEq(
            registry.getNextRegistrationId(),
            0,
            "counter starts at 0 despite comment claiming 'starting from 1'"
        );
    }

    /// 2. The first registration returns ID 0.
    function test_firstValidatorReceivesIdZero() public {
        uint256 id = registry.registerValidator(validatorPubKey, 0);
        assertEq(id, 0, "first validator gets registrationId 0");
    }

    /// 3. ID 0 can never be assigned a controller.
    function test_idZeroCanNeverGetController() public {
        uint256 id = registry.registerValidator(validatorPubKey, 0);
        assertEq(id, 0);

        PermissionedValidatorManager pvmImpl =
            new PermissionedValidatorManager(IValidatorRegistry(address(registry)));
        AdminUpgradeableProxy pvmProxy = new AdminUpgradeableProxy(
            address(pvmImpl),
            address(this),
            abi.encodeWithSignature(
                "initialize(address,address)", address(this), address(this)
            )
        );
        pvm = PermissionedValidatorManager(address(pvmProxy));

        // Owner call passes onlyOwner, then hits the ID-0 guard:
        vm.expectRevert(PermissionedValidatorManager.RegistrationIdIsZero.selector);
        pvm.configureController(controller, 0, 100);
    }

    /// 4. The validator is locked at zero voting power with no recovery path.
    function test_idZeroValidatorLockedAtZeroPower() public {
        uint256 id = registry.registerValidator(validatorPubKey, 0);
        assertEq(id, 0);

        Validator memory v = registry.getValidator(id);
        assertEq(v.votingPower, 0, "locked at DEFAULT_VOTING_POWER");
        assertEq(
            uint256(v.status),
            uint256(ValidatorStatus.Registered),
            "stuck in Registered status forever"
        );
    }
}
```

3. Run:

```bash
forge test --match-path contracts/test/RegistrationIdZeroPoC.t.sol -vvv
```

4. All four tests pass, demonstrating that a deployment which misses the off-chain `_nextRegistrationId = 1` initialization produces a permanently unusable first validator registration.

## Evidence

**The counter is never initialized on-chain** — `contracts/src/validator-manager/ValidatorRegistry.sol`:

- Lines 59-60 — comment claims an invariant the code does not enforce:
  ```solidity
  /// @notice Counter tracking the next available registration ID for new
  ///         validator registrations, starting from 1
  uint256 _nextRegistrationId;
  ```
- Line 110 — post-increment from whatever storage holds: `registrationId = $._nextRegistrationId++;`
- Lines 167-170 — zero voting power means no consensus influence:
  ```solidity
  /// @dev Setting voting power to 0 effectively deactivates the validator
  ///      without removing it from storage. The validator remains registered
  ///      but will have no influence in consensus.
  ```
- No `initialize()` function exists anywhere in the file (verified: the only `initialize()` functions in `contracts/src/` are in `Denylist.sol`, `PermissionedValidatorManager.sol`, and the `TestImplementation` mock)

**ID 0 is a reserved sentinel in the controller layer** — `contracts/src/validator-manager/roles/Controller.sol`:

- Line 113 — 0 means "not a controller": `require($.registrationOf[msg.sender] != 0, CallerIsNotController());`
- Line 122 — ID 0 explicitly rejected: `require(registrationId != 0, RegistrationIdIsZero());`

**Only controllers can change voting power** — `contracts/src/validator-manager/PermissionedValidatorManager.sol`:

- Line 47 — new validators registered with zero power: `uint64 public constant DEFAULT_VOTING_POWER = 0;`
- Line 119 — onlyController gate: `function updateValidatorVotingPower(uint64 newVotingPower) external onlyController whenNotPaused`

**The project's own tests prove the invariant is off-chain only** — `contracts/test/validator-manager/ValidatorRegistryProxy.t.sol`:

- Line ~101 — the test must manually poke storage because no on-chain mechanism exists: *"Initialize _nextRegistrationId to 1 (slot offset 4 in ValidatorRegistryStorage)"*
- Line ~136 — confirms the raw default: *"Implementation should have default nextRegistrationId (0, uninitialized)"*
- Line ~258 — the intended invariant, enforced only by the test setup: `assertEq(registry.getNextRegistrationId(), 1, "Next registration ID should start at 1");`

**The management tooling treats ID 0 as invalid** — `contracts/scripts/ValidatorManagement.s.sol`:

- Line 273: `require(_registrationId != 0, "RegistrationId not found");`

## Impact

1. **Burned first registration.** On any chain instance whose genesis does not set `_nextRegistrationId = 1`, the first registered validator receives ID 0 and can never be assigned a controller. Its voting power is locked at `DEFAULT_VOTING_POWER = 0`, meaning it has no influence in consensus. No function in the contract suite can assign a controller to registration ID 0 or reassign a validator's ID — that specific registration is unrecoverable without a contract upgrade and state migration.

2. **Partial recovery available.** The operator can mitigate by registering a second validator: after the burned ID-0 registration, `_nextRegistrationId` has incremented to 1, so the next registration receives a valid ID and can be configured normally. The cost is one wasted validator public key — plus any node, stake, or infrastructure already bound to it — not permanent exclusion from the validator set. This bounds the impact to a single wasted registration per affected deployment.

3. **Silent failure mode.** Registration itself succeeds without error. The failure only surfaces later when the operator attempts `configureController()`, by which point the registration is already broken and the validator's public key is burned.

4. **Fragile, undocumented dependency.** The safety of the entire validator registration flow depends on an off-chain genesis storage poke that is invisible to the contract, unenforced on-chain, and easy to miss on new deployments (devnet, testnet resets, future chain launches). The codebase's own comments, tests, and scripts all assume the invariant holds — but nothing guarantees it.

## Remediation

1. **Enforce the counter on-chain (preferred):** add an `initialize()` function to `ValidatorRegistry` that sets `_nextRegistrationId = 1` and call it during proxy deployment, consistent with the pattern used by `PermissionedValidatorManager` and `Denylist`.
2. **Change the increment pattern:** use pre-increment (`registrationId = ++$._nextRegistrationId;`) so the first ID is 1 even when storage defaults to 0 — no initialization required.
3. **Change the sentinel:** replace the `!= 0` checks in `Controller.sol` with a dedicated boolean mapping (or `type(uint256).max` sentinel) so registration ID 0 is a valid, fully usable ID.
4. **Add a regression test:** deploy the registry without the manual storage poke and assert the first registration ID is non-zero.

## Process notes

- Source-code finding in the in-scope `circlefin/arc-node` repository. No live system touched — no testnet RPC calls, no transactions broadcast, no accounts created.
- No scanners used: manual review of the validator-manager contract suite; PoC is a hand-written Foundry test adapted from the project's own test file.
- Testnet-only policy respected: the PoC runs entirely against a local Foundry EVM.
