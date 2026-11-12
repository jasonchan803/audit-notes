# Incorrect Controller Assignment Can Permanently Lock Deposit Assets

**Severity:** Medium

**Affected Contract:** `TrancheVault.sol`

**Affected Location:** `/home/jason/audit-comp-plether/packages/perps/src/TrancheVault.sol`

## Summary

The `TrancheVault` allows an asset owner to submit an asynchronous deposit request by specifying a separate controller address. Upon submission, the owner's USDG is transferred into the vault, while the resulting deposit request and its associated claim rights are recorded under the specified controller.

However, the original asset owner has no independent mechanism to cancel the request, recover the deposited assets, or claim the resulting shares. Both cancellation and deposit claims require authorization from the designated controller or its approved operators.

If the depositor mistakenly specifies an incorrect controller address that is not controlled by any available party, neither the original owner nor the intended controller can access the deposit entitlement. Since the contract provides no owner-authorized recovery or controller reassignment mechanism, the deposited value may become permanently inaccessible.

## Root Cause

TrancheVault accepts deposit assets from an owner while assigning the resulting deposit request exclusively to the supplied controller.

Subsequent request cancellation and deposit claims are authorized based on the controller address and its approved operators. The original asset owner has no independent authorization path to cancel the request, claim the resulting shares, or change the controller associated with the request.

## Preconditions

 - A depositor submits a deposit request with a controller address different from the intended controller.

 - The depositor does not control the specified controller and cannot obtain its cooperation.

 - No authorized operator or alternative recovery mechanism is available.

## Vulnerability Details

If a depositor supplies an unintended controller address, the deposit request is permanently associated with that address.

The intended controller and the original asset owner cannot claim the resulting shares. The request can only be managed and claimed by the mistakenly specified controller or its authorized operators.

If the specified address is not controlled by any available party, the deposited value may become permanently inaccessible because the contract provides no owner recovery or controller migration mechanism.

If the address is controlled by another party, that party may obtain the ability to claim the shares despite not having supplied the original deposit assets.


**Affected Code:**

```solidity
// Vulnerable
 function _claimDeposit(
        uint256 requestId,
        uint256 assets,
        address receiver,
        address controller
    ) internal returns (uint256 shares) {
        // Only controller can claim the shares
        _requireControllerAuth(controller);
        if (receiver == address(0)) {
            revert TrancheVault__ZeroAddress();
        }

function cancelPendingDeposit(
        uint256 requestId,
        address receiver,
        address controller
    ) public returns (uint256 assets) {
        // Only controller can cancel the request
        _requireControllerAuth(controller);
        if (receiver == address(0)) {
            revert TrancheVault__ZeroAddress();
        }
```

## Impact

A depositor may permanently lose access to the value of their deposited USDG if an incorrect and unavailable controller address is specified when submitting a deposit request.

The deposited assets are transferred into the `TrancheVault` immediately, but all subsequent cancellation and claim operations are controlled by the designated controller.

As a result:

- The original asset owner cannot cancel the pending deposit request.
- The original asset owner cannot claim the shares after the deposit becomes claimable.
- The intended controller cannot claim the deposit because the request is associated with a different address.
- No owner-authorized recovery or controller reassignment mechanism is available.

If the designated controller is an address for which no party has control or access, the deposit entitlement may remain inaccessible indefinitely, resulting in a permanent loss of access to the deposited value.

## Proof of Concept

**Test File:** `/home/jason/audit-comp-plether/packages/perps/test/perps/spec/lp/IncorrectControllerAssignment.t.sol`

**Test Function:** `test_OwnerCannotCancelDepositControlledByAnotherAddress()`、`test_OwnerCannotClaimDepositControlledByAnotherAddress()`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CooldownBypassReceiver, TrancheCooldownBypassReceiver} from "../../support/BehaviorScenarioHelpers.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

error ERC4626ExceededMaxDeposit(address receiver, uint256 assets, uint256 max);
contract InputWrongControllerAddress is BasePerpTest {

    address bob = address(0xB0B);
    address alice = address(0xA11CE);
    address wrongController = address(0xA12CE);


    /// @notice Verifies that the original asset owner cannot cancel a pending
    ///         deposit request assigned to a different controller.
    /// @dev The test demonstrates that:
    ///      1. Bob submits a deposit request and transfers USDG into the vault.
    ///      2. The deposit assets are held by the vault and the request is
    ///         associated with `wrongController`.
    ///      3. Bob, as the original asset owner, attempts to cancel the request
    ///         but the call reverts because he is not the controller or its operator.
    ///      4. The deposit request remains pending and the escrowed assets
    ///         cannot be recovered by Bob through the cancellation mechanism.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function test_OwnerCannotCancelDepositControlledByAnotherAddress() public {
        settlementToken.mint(bob, 100_000e6);
        uint256 bobBefore = settlementToken.balanceOf(bob);
        uint256 vaultBefore = settlementToken.balanceOf(address(juniorVault));

        vm.startPrank(bob);
        settlementToken.approve(address(juniorVault), 100_000e6);

        uint256 requestId = juniorVault.requestDeposit(
            100_000e6,
            wrongController,
            bob
        );
        vm.stopPrank();

        assertEq(
            settlementToken.balanceOf(bob),
            bobBefore - 100_000e6
        );
        assertEq(
            settlementToken.balanceOf(address(juniorVault)),
            vaultBefore + 100_000e6
        );

        // Request has been submitted and assets are already escrowed.
        vm.expectRevert();
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

        vm.expectRevert();
        vm.prank(bob);
        juniorVault.claimDeposit(
            requestId,
            100_000e6,
            bob,
            bob
        );
    }
    
    /// @notice Verifies that the original asset owner cannot claim shares from
    ///         a finalized deposit request assigned to a different controller.
    /// @dev The test demonstrates that:
    ///      1. Bob submits a deposit request and transfers USDG into the vault.
    ///      2. The LP epoch is finalized and the request becomes claimable
    ///         for `wrongController`.
    ///      3. Bob, as the original asset owner, attempts to claim the deposit
    ///         but the call reverts.
    ///      4. The intended controller (Alice) also cannot claim the deposit
    ///         because the request is associated with `wrongController`.
    ///      5. `wrongController` successfully claims the deposit shares.
    ///      6. The resulting shares are credited to `wrongController`, while
    ///         Bob receives no shares.
    ///      7. The claimable deposit balance is fully consumed.
    ///
    ///      This demonstrates that deposit ownership and claim authorization
    ///      are determined by the specified controller rather than the
    ///      original asset owner. If the specified controller is unavailable
    ///      or uncontrolled, the original owner has no independent recovery
    ///      path through the deposit claim mechanism.
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
                address(0xA11cE),     // receiver
                100000000000,         // assets (1e11)
                0                     // max
            )
        );
        // Alice: intended controller cannot claim.
        vm.prank(alice);
        juniorVault.claimDeposit(
            requestId, 
            100_000e6, 
            alice, 
            alice);

        vm.expectRevert(
            abi.encodeWithSelector(
                ERC4626ExceededMaxDeposit.selector,
                address(0xB0B),     // receiver
                100000000000,         // assets (1e11)
                0                     // max
            )
        );
        // Bob: asset owner cannot claim either.
        vm.prank(bob);
        juniorVault.claimDeposit(
            requestId, 
            100_000e6, 
            bob, 
            bob);

        uint256 bobSharesBefore = juniorVault.balanceOf(bob);
        uint256 controllerSharesBefore = juniorVault.balanceOf(wrongController);
        uint256 controllerAssetsBefore = settlementToken.balanceOf(wrongController);

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
            juniorVault.claimableDepositRequest(requestId, wrongController),
            0
        );
        
    }

}

```

**Expected Result:**

The original asset owner should retain a recovery path when a deposit request is associated with an incorrect or unavailable controller.

At a minimum, the protocol should provide a secure mechanism that allows the original owner to cancel an unclaimed deposit request or reassign its controller, subject to appropriate state and accounting restrictions.

If the request has already been finalized, the recovery mechanism should preserve the original deposit entitlement and prevent duplicate claims or unauthorized transfers.

The original owner should not permanently lose access to deposited assets solely because an incorrect controller address was supplied when submitting the request.

**Actual Result:**

<img width="1390" height="141" alt="image" src="https://github.com/user-attachments/assets/42a26039-66c6-4d05-a4ab-094fd2f91617" />

## Recommendation

Introduce an owner-authorized recovery mechanism for unclaimed deposit requests, allowing the original deposit owner to cancel or reassign a request when the designated controller is incorrect or unavailable.

Any reassignment mechanism must atomically update all controller-indexed request records, queue states, and pending or claimable accounting to prevent duplicate claims or inconsistent state.

Alternatively, require explicit controller acceptance before permanently associating a deposit request with the designated controller. Until acceptance, the original owner should retain the ability to cancel or update the request.
