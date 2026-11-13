# Incorrect Controller Assignment Can Make Deposit Entitlements Inaccessible to the Original Owner

**Severity:** Medium

**Affected Contract:** `TrancheVault.sol`

**Affected Location:** `packages/perps/src/TrancheVault.sol`

## Summary

The `TrancheVault` allows an asset owner or an owner-authorized operator to submit an asynchronous deposit request with a separately specified controller address.

Upon submission, the owner's USDG is transferred into the vault, while the request and its claim rights are assigned exclusively to the specified controller.

However, the original asset owner has no independent mechanism to cancel the request, claim the resulting shares, or reassign its controller. Revoking an operator's authorization also does not affect requests already created by that operator.

If an incorrect controller is assigned and is unavailable or uncontrolled, the original owner may permanently lose access to the deposit entitlement.

## Root Cause

The deposit request is indexed by the specified controller rather than the original asset owner. Both cancellation and claiming require authorization from that controller or its approved operators.

Although the owner can revoke an authorized deposit operator, this only prevents future operator actions. It does not restore the owner's control over requests already created or provide an owner-authorized recovery mechanism.

## Preconditions
 - A deposit request is assigned to an unintended controller, either directly by the owner or by an owner-authorized operator.
 - The original owner does not control the assigned controller and cannot obtain its cooperation.
 - No available controller operator or alternative recovery mechanism can access the deposit entitlement.

## Vulnerability Details

When `requestDeposit(assets, controller, owner)` is called, the assets are transferred from `owner` into `TrancheVault`, while the request is recorded under `controller`.

The original owner cannot cancel the request because `cancelPendingDeposit()` requires controller authorization. After epoch finalization, `_claimDeposit()` applies the same controller-based authorization, preventing the owner from claiming the resulting shares.

Revoking an owner-authorized operator does not change the controller assigned to an existing request. Consequently, neither the original owner nor the intended controller can recover the deposit entitlement through the available request-management and claim mechanisms.

The incorrectly assigned controller, however, can claim the resulting shares.

**Affected Code:**

```solidity
// Relevant code excerpts
    function _requestDeposit()
            IERC20(asset()).safeTransferFrom(owner, address(this), assets);
            DepositPosition storage position =
                depositRequests[controller][requestId];

    function _claimDeposit()
            _requireControllerAuth(controller);
            _requireShareReceiver(controller, receiver);
            }

    function cancelPendingDeposit()
            _requireControllerAuth(controller);

    function _requireControllerAuth(
        address controller
    ) internal view {
        if (msg.sender != controller && !isOperator[controller][msg.sender]) {
            revert TrancheVault__NotControllerOrOperator();
        }
    }
```

## Impact

The original asset owner and intended controller may permanently lose access to the deposit entitlement when the assigned controller is unavailable or uncontrolled.

If the assigned controller is controlled by another party, that party may claim the resulting shares despite not providing the original deposit assets.

This can result in permanent inaccessibility of deposited value under the stated preconditions.

## Proof of Concept

**Test File:** `packages/perps/test/perps/spec/lp/IncorrectControllerAssignment.t.sol`

**Test Function:**
 - `test_RevokingOperatorDoesNotRestoreOwnerControl()`
 - `test_OwnerCannotClaimDepositControlledByAnotherAddress()`

**Run:**

`git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8`

`cd packages/perps`

`forge test --match-path test/perps/spec/lp/IncorrectControllerAssignment.t.sol -vvv`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

error TrancheVault__NotControllerOrOperator();
error ERC4626ExceededMaxDeposit(address receiver, uint256 assets, uint256 max);

contract IncorrectControllerAssignment is BasePerpTest {

    address bob = address(0xB0B);
    address eva = address(0xA0A);
    address alice = address(0xA11CE);
    address wrongController = address(0xA12CE);


    /// @notice Verifies that revoking an authorized deposit operator does not
    ///         restore the original asset owner's control over a deposit request
    ///         assigned to a different controller.
    /// @dev The test demonstrates that:
    ///      1. bob authorizes eva as an operator.
    ///      2. eva uses her operator permission to submit a deposit request
    ///         on behalf of bob, assigning it to wrongController.
    ///      3. bob's USDG is transferred into the vault and escrowed under
    ///         the request associated with wrongController.
    ///      4. bob revokes eva's operator permission after the request is created.
    ///      5. Revoking eva's permission does not change the request's controller
    ///         or restore bob's ability to manage the request.
    ///      6. bob cannot cancel the pending request because he is neither
    ///         its controller nor an authorized operator of that controller.
    ///      7. After the LP epoch is finalized, the deposit entitlement remains
    ///         associated with wrongController.
    ///      8. bob cannot claim the finalized deposit entitlement.
    ///
    ///      This demonstrates that revoking an authorized deposit operator
    ///      only prevents future operator actions and does not provide the
    ///      original asset owner with a recovery mechanism for requests
    ///      already assigned to another controller.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_RevokingOperatorDoesNotRestoreOwnerControl() public {
        // bob funds the deposit.
        settlementToken.mint(bob, 100_000e6);

        // bob authorizes eva as an operator.
        vm.prank(bob);
        juniorVault.setOperator(eva, true);

        uint256 bobBefore = settlementToken.balanceOf(bob);
        uint256 vaultBefore = settlementToken.balanceOf(address(juniorVault));

        vm.prank(bob);
        settlementToken.approve(address(juniorVault), 100_000e6);

        // alice submits a deposit request for bob, assigning it to wrongController.
        vm.prank(eva);
        uint256 requestId = juniorVault.requestDeposit(
            100_000e6,
            wrongController,
            bob
        );

        assertEq(
            settlementToken.balanceOf(bob),
            bobBefore - 100_000e6
        );
        assertEq(
            settlementToken.balanceOf(address(juniorVault)),
            vaultBefore + 100_000e6
        );

        // bob revokes eva's operator permission.
        vm.prank(bob);
        juniorVault.setOperator(eva, false);

        // Verify eva's operator permission is revoked.
        assertFalse(juniorVault.isOperator(bob, alice));

        // Verify the request remains assigned to wrongController.
        assertEq(
            juniorVault.pendingDepositRequest(requestId, wrongController),
            100_000e6
        );

        // bob still cannot cancel the request.
        vm.expectRevert(TrancheVault__NotControllerOrOperator.selector);
        vm.prank(bob);
        juniorVault.cancelPendingDeposit(
            requestId,
            bob,
            wrongController
        );
    
        // Then settle the epoch.
        vm.warp(juniorVault.depositEpochStart(requestId));
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
        _settleLpEpochForTest();

        assertEq(
            juniorVault.claimableDepositRequest(requestId, wrongController),
            100_000e6
        );

        vm.expectRevert(
            abi.encodeWithSelector(
                ERC4626ExceededMaxDeposit.selector,
                bob,                  // receiver
                100000000000,         // assets (1e11)
                0                     // max
            )
        );
        vm.prank(bob);
        juniorVault.claimDeposit(
            requestId,
            100_000e6,
            bob,
            bob
        );
    }

    /// @notice Verifies that only the controller assigned to a deposit request
    ///         can claim the resulting deposit shares.
    /// @dev The test demonstrates that:
    ///      1. bob submits a deposit request and transfers USDG into the vault,
    ///         specifying wrongController as the request controller.
    ///      2. The LP epoch is finalized and the deposit entitlement becomes
    ///         claimable under wrongController.
    ///      3. alice, the intended controller, cannot claim the deposit because
    ///         the request is not associated with her address.
    ///      4. bob, the original asset owner, also cannot claim the deposit
    ///         because the request is not associated with his address.
    ///      5. wrongController successfully claims the resulting deposit shares.
    ///      6. The shares are credited to wrongController, while bob and alice
    ///         receive no shares.
    ///      7. The claimable deposit entitlement associated with wrongController
    ///         is fully consumed.
    ///
    ///      This demonstrates that deposit claim rights are determined by the
    ///      controller specified when the request is submitted, rather than
    ///      by the original asset owner or the intended controller.
    ///      If the assigned controller is unavailable or uncontrolled,
    ///      the original asset owner has no independent recovery path through
    ///      the deposit claim mechanism.
    function test_OwnerCannotClaimDepositControlledByAnotherAddress() public {
        settlementToken.mint(bob, 100_000e6);
        vm.startPrank(bob);
        settlementToken.approve(address(juniorVault), 100_000e6);
        uint256 requestId = juniorVault.requestDeposit(
            100_000e6,
            wrongController,
            bob
        );
        vm.stopPrank();

        vm.warp(juniorVault.depositEpochStart(requestId));
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(block.timestamp));
        _settleLpEpochForTest();

        assertGt(
            juniorVault.claimableDepositRequest(requestId, wrongController),
            0
        );

        vm.expectRevert(
            abi.encodeWithSelector(
                ERC4626ExceededMaxDeposit.selector,
                alice,                // receiver
                100000000000,         // assets (1e11)
                0                     // max
            )
        );
        // alice: intended controller cannot claim.
        vm.prank(alice);
        juniorVault.claimDeposit(
            requestId, 
            100_000e6, 
            alice, 
            alice);

        vm.expectRevert(
            abi.encodeWithSelector(
                ERC4626ExceededMaxDeposit.selector,
                bob,                  // receiver
                100000000000,         // assets (1e11)
                0                     // max
            )
        );
        // bob: asset owner cannot claim either.
        vm.prank(bob);
        juniorVault.claimDeposit(
            requestId, 
            100_000e6, 
            bob, 
            bob);

        uint256 bobSharesBefore = juniorVault.balanceOf(bob);
        uint256 aliceSharesBefore = juniorVault.balanceOf(alice);
        uint256 controllerSharesBefore = juniorVault.balanceOf(wrongController);

        // Wrong controller: the request's actual controller can claim.
        vm.prank(wrongController);
        uint256 shares = juniorVault.claimDeposit(
            requestId,
            100_000e6,
            wrongController,
            wrongController
        );

        assertGt(shares, 0);

        assertEq(
            juniorVault.balanceOf(wrongController),
            controllerSharesBefore + shares
        );

        assertEq(
            juniorVault.balanceOf(bob),
            bobSharesBefore
        );

        assertEq(
            juniorVault.balanceOf(alice),
            aliceSharesBefore
        );

        assertEq(
            juniorVault.claimableDepositRequest(requestId, wrongController),
            0
        );
        
    }

}
```

**Test Result:**
```solidity
Ran 2 tests for test/perps/spec/lp/IncorrectControllerAssignment.t.sol:IncorrectControllerAssignment
[PASS] test_OwnerCannotClaimDepositControlledByAnotherAddress() (gas: 904244)
[PASS] test_RevokingOperatorDoesNotRestoreOwnerControl() (gas: 842680)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 38.16ms (8.70ms CPU time)

Ran 1 test suite in 96.82ms (38.16ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Recommendation

Introduce an owner-authorized recovery mechanism allowing the original asset owner to cancel or reassign deposit requests when the designated controller is incorrect or unavailable, including requests submitted by authorized operators.

The mechanism should safely update controller-indexed request records, queue states, and pending or claimable accounting to prevent duplicate claims or inconsistent state.

Alternatively, require controller acceptance before permanently associating a request with the designated controller, while preserving the owner's ability to cancel or update unaccepted requests.
