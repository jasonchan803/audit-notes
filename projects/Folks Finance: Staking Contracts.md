---
type: project-note
project: Folks Finance: Staking Contracts
audit-source: immunefi
date: 2026-10-15
tags: [tag1, tag2]
status: [in-progress/completed]
---

## Project Brief
A fixed-APR staking protocol where users stake ERC-20 tokens for a fixed duration, then withdraw principal + reward linearly over an unlock period. Supports migration and role-based administration.

## Low Risk Findings

### [L-01]: `stakeWithPermit()` will Swallow all the permit errors

**Severity**: Low

**Location**: `Staking.sol` - `stakeWithPermit()`

**Description**: Using try ... catch {} would cause the failure of permit being silently ignored

**Impact**: 
- When allowance is insufficient, the transaction reverts with a misleading error (e.g., "insufficient allowance") instead of indicating that the permit has failed.
- the transaction is successful but the permit actually fail when the allowance is enough.

**Root Cause**: 
try ... catch {} do not distinguish error types and emit the event.

**My POC Walkthrough (optional)**：
```solidity
    function test_1() public {
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), charlie, 100 ether);

        uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);

        uint256 deadline = block.timestamp + 1 hours;
        (uint8 v, bytes32 r, bytes32 s) = signCharliePermit(10 ether, deadline);

        vm.expectRevert(
            abi.encodeWithSelector(IERC20Errors.ERC20InsufficientAllowance.selector, address(staking), 0, 10 ether)
        );
        stakeWithPermit(alice, periodIndex, 10 ether, 20, 10, 5000, address(0), deadline, v, r, s);
    }

    function test_2() public {
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), charlie, 100 ether);

        uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);

        vm.prank(charlie);
        token.approve(address(staking), 50 ether);

        uint256 deadline = block.timestamp + 1 hours;
        (uint8 v, bytes32 r, bytes32 s) = signCharliePermit(10 ether, deadline);

        vm.warp(block.timestamp + 2 hours);

        assertEq(staking.getUserStakes(charlie).length, 0);
        stakeWithPermit(charlie, periodIndex, 10 ether, 20, 10, 5000, address(0), deadline, v, r, s);

        assertEq(staking.getUserStakes(charlie).length, 1);
        assertEq(staking.getUserStake(charlie, periodIndex).amount, 10 ether);
    }
```

**Fix**: Distinguish the error type in the catch block, or emit at least one event to indicate that permit has failed

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
try IERC20Permit(address(TOKEN)).permit(msg.sender, address(this), amount, deadline, v, r, s) {} catch {}

// Fixed
try IERC20Permit(address(TOKEN)).permit(msg.sender, address(this), amount, deadline, v, r, s) {} 
catch (bytes memory reason) {
    emit PermitFailed(reason);
}
```

### [L-02]: The small pledge reward is rounded to 0 due to integer division

**Severity**: Low/Informational 

**Location**: `Staking.sol` - `_getAccrued()`

**Description**: Due to integer division rounding down, users staking very small amounts receive zero reward despite locking their tokens for the full duration.

**Impact**: User can not get any reward throuth they locking their tokens for the full duration.

**Root Cause**: The calculate is (stakingAmount * stakingTime * stakingApyBps) / (365 days * 10000)

**My POC Walkthrough (optional)**：
```solidity
    function test_Staking_less_Withdrawnone() public {
        deal(address(token), address(staking), 1000 ether);
        deal(address(token), alice, 100 ether);

        uint8 periodIndex = addStakingPeriodByManager(50 ether, 365 days, 10, 1000, true);

        vm.prank(alice);
        token.approve(address(staking), 50 ether);

        vm.warp(0);
        uint256 initialamount = 1;
        stake(alice, periodIndex, initialamount, 365 days, 10, 1000, address(0));

        vm.warp(366 days);

        uint256 aliceReward = calculateReward(1, 365 days, 1000);
        assertEq(aliceReward, 0);
        assertEq(staking.getClaimable(alice, periodIndex), initialamount);
    }
```

**Fix**: Revert when the reward equal to 0.

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;

// Fixed
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;
if (reward == 0) revert RewardTooSmallToBeNonZero();
```
