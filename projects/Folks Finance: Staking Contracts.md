---
type: project-note
project: "Folks Finance: Staking Contracts"
audit-source: immunefi
date: 2026-10-15
tags: [staking, permit, rounding, access-control, state-lock]
status: completed
---

## Project Brief
A fixed-APR staking protocol where users stake ERC-20 tokens for a fixed duration, then withdraw principal + reward linearly over an unlock period. Supports migration and role-based administration.

## Low Risk Findings

### [L-01]: `stakeWithPermit()` will swallow all the permit errors

**Severity**: Low

**Location**: `Staking.sol` - `stakeWithPermit()`

**Description**: Using `try ... catch {}` causes permit failures to be silently ignored
```solidity
// Vulnerable
    try IERC20Permit(address(TOKEN)).permit(msg.sender, address(this), amount, deadline, v, r, s) {} catch {}
```

**Impact**: 
- When allowance is insufficient, the transaction reverts with a misleading error (e.g., "insufficient allowance") instead of indicating that the permit has failed.
- the transaction is successful but the permit actually fails when the allowance is enough.

**Root Cause**: 
try ... catch {} do not distinguish error types and emit the event.

**My POC Walkthrough**：
```solidity
function test_Staking_When_PermitExpired() public {
    // 1. Setup: Authorize Charlie and generate a signature in advance
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), charlie, 100 ether);
    
    uint8 periodIndex = addStakingPeriodByManager(50 ether, 20, 10, 5000, true);
    
    vm.prank(charlie);
    token.approve(address(staking), 50 ether);
    
    uint256 deadline = block.timestamp + 1 hours;
    (uint8 v, bytes32 r, bytes32 s) = signCharliePermit(10 ether, deadline);
    
    // 2. Fast forward past the deadline to make permit invalid.
    vm.warp(block.timestamp + 2 hours);
    
    // 3. Execute: Permit fails silently, but stake succeeds due to allowance.
    assertEq(staking.getUserStakes(charlie).length, 0);
    stakeWithPermit(charlie, periodIndex, 10 ether, 20, 10, 5000, address(0), deadline, v, r, s);
    
    // 4. Assertion: User successfully staked despite permit failure.
    assertEq(staking.getUserStakes(charlie).length, 1);
    assertEq(staking.getUserStake(charlie, periodIndex).amount, 10 ether);
}
```

**Fix**: Distinguish the error type in the catch block, or emit at least one event to indicate that permit has failed
```solidity
// Fixed
    try IERC20Permit(address(TOKEN)).permit(msg.sender, address(this), amount, deadline, v, r, s) {} 
    catch (bytes memory reason) {
        emit PermitFailed(reason);
    }
```

### [L-02]: The small pledge reward is rounded to 0 due to integer division

**Severity**: Low

**Location**: `Staking.sol` - `_stake()`

**Description**: Due to integer division rounding down, users staking very small amounts receive zero reward despite locking their tokens for the full duration.
```solidity
// Vulnerable
    uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;
```

**Impact**: Users cannot receive any reward despite locking their tokens

**Root Cause**: The calculation is (stakingAmount * stakingTime * stakingApyBps) / (365 days * 10000)

**My POC Walkthrough (optional)**：
```solidity
function test_Staking_less_Withdrawnone() public {
    // 1. Setup
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);
    
    uint8 periodIndex = addStakingPeriodByManager(50 ether, 365 days, 10, 1000, true);
    
    vm.prank(alice);
    token.approve(address(staking), 50 ether);
    
    vm.warp(0);
    uint256 initialamount = 1;
    stake(alice, periodIndex, initialamount, 365 days, 10, 1000, address(0));

    // 2. Fast forward past the time to make unlockTime maturity.
    vm.warp(366 days);

    // 3. Alice receive zero reward despite locking their tokens for the full duration.
    uint256 aliceReward = calculateReward(1, 365 days, 1000);
    assertEq(aliceReward, 0);
    assertEq(staking.getClaimable(alice, periodIndex), initialamount);
}
```

**Fix**: Revert when the reward equals 0. Alternatively, enforce a minimum stake amount in _stake to prevent dust-level stakes.
```solidity
// Fixed
    uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;
    if (reward == 0) revert RewardTooSmallToBeNonZero();
```

### [L-03]: Stale Migration Permit Allows Asset Theft After Role Restoration

**Severity**: Medium

**Location**: Staking.sol - setMigrationPermit()

**Description**:
The setMigrationPermit function includes an unconditional check for the MIGRATOR_ROLE:
```solidity
// Vulnerable
    if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
```
When a migrator loses this role, users who had previously granted them permission are unable to revoke it (`setMigrationPermit(..., false)` will revert). This creates a permanent "stale state" where `migrationPermits[migrator][user]` remains true.


**Impact**:
- Unauthorized Asset Migration (Medium/High): If the admin later re-grants the MIGRATOR_ROLE to this address, the migrator can immediately call migratePositionsFrom(user) and transfer the user's staked assets without their renewed consent.

- Loss of User Control (Low/Griefing): Users cannot revoke permissions for deprecated migrators, violating the documented functionality.

**Root Cause**:
The hasRole guard is applied unconditionally, instead of being restricted only to the granting action (_isMigrationPermitted == true).

**My POC Walkthrough (Extended)**:

```solidity
function test_exploit_revokePermitBlockedAfterMigratorRoleRevoked() public {
    // 1. Setup
    deal(address(token), address(staking), 1000 ether);
    deal(address(token), alice, 100 ether);
    uint8 periodIndex = addStakingPeriodByManager(50 ether, 365 days, 10, 1000, true);

    // 2. Alice stakes and grants permission
    vm.startPrank(alice);
    token.approve(address(staking), 50 ether);
    staking.setMigrationPermit(migrator, true);
    vm.stopPrank();
    stake(alice, periodIndex, 10 ether, 365 days, 10, 1000, address(0));

    // 3. Admin revokes the role
    vm.prank(admin);
    staking.revokeRole(keccak256("MIGRATOR"), migrator);

    // 4. Alice tries to revoke permission but fails
    vm.prank(alice);
    vm.expectRevert(abi.encodeWithSelector(IStakingV1.MigratorNotFound.selector, migrator));
    staking.setMigrationPermit(migrator, false);  

    // 5. Admin accidentally/somehow restores the role
    vm.prank(admin);
    staking.grantRole(keccak256("MIGRATOR"), migrator);

    // 6. Migrator exploits the stale permit to steal assets
    vm.prank(migrator);
    staking.migratePositionsFrom(alice);

    // 7. Assertion: Alice's stake is gone (transferred to migrator)
    assertEq(staking.getUserStakes(alice).length, 0);
}
```
**Fix**:
Restrict the role check to only when granting permissions (as seen in the patched version), and/or remove redundant state updates to avoid gas waste:

```solidity
// Fixed
    if (migrationPermits[_migrator][msg.sender] == _isMigrationPermitted) return;
    if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
    migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
```
**My Biggest Takeaway**:

Version Control is Critical: Testing against the fixed main branch made me think there was no bug. I had to use git checkout to revert to the audited commit to make the POC work.

Push the Impact: The official report stopped at "User cannot revoke". By extending the POC, I proved "User's funds can be stolen". This is the difference between a Low and a High severity.
