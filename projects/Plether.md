---
type: project-note
project: Plether
audit-source: immunefi
date: 2026-11-12
tags: [tag1, tag2]
status: completed
---

## Project Brief
Plether lets users trade and hedge the strength of the US dollar onchain. Plether Perps is a perpetual market on a dollar index built from a basket of six major currencies: EUR, JPY, GBP, CAD, SEK and CHF. Traders open LONG USD positions, 
which gain when the dollar strengthens against the basket, or SHORT USD positions, which gain when it weakens. Margin and settlement are in USDG, and orders are priced by Pyth through a delayed-order model.

## High Risk Findings

### [H-01]: 

**Severity**: [Critical/High/Medium/Low/Informational]

**Location**: [合约文件:行号 或 函数名]

**Description**: 
**Vulnerability Details**

If a depositor supplies an unintended controller address, the deposit request is permanently associated with that address.

The intended controller and the original asset owner cannot claim the resulting shares. The request can only be managed and claimed by the mistakenly specified controller or its authorized operators.

If the specified address is not controlled by any available party, the deposited value may become permanently inaccessible because the contract provides no owner recovery or controller migration mechanism.

If the address is controlled by another party, that party may obtain the ability to claim the shares despite not having supplied the original deposit assets.

**Preconditions**

 - A depositor submits a deposit request with a controller address different from the intended controller.

 - The depositor does not control the specified controller and cannot obtain its cooperation.

 - No authorized operator or alternative recovery mechanism is available.


```solidity
// Vulnerable
[漏洞代码]
```

**Impact**: [后果]

**Root Cause**: TrancheVault accepts deposit assets from an owner while assigning the resulting deposit request exclusively to the supplied controller.

Subsequent request cancellation and deposit claims are authorized based on the controller address and its approved operators. The original asset owner has no independent authorization path to cancel the request, claim the resulting shares, or change the controller associated with the request.

**My POC Walkthrough**：[我的POC思路]
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

**Fix**: [修复方式]
```solidity
// Fixed
[修复代码]
```

**English Takeaway**: [1句英文总结]
