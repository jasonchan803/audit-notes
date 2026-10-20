---
type: project-note
project: "Folks Finance: Staking Contracts"
audit-source: immunefi
date: 2026-10-15
tags: [staking, permit, rounding, low]
status: completed
---

## Project Brief
A fixed-APR staking protocol where users stake ERC-20 tokens for a fixed duration, then withdraw principal + reward linearly over an unlock period. Supports migration and role-based administration.

## Low Risk Findings

### [L-01]: `stakeWithPermit()` will Swallow all the permit errors

**Severity**: Low

**Location**: `Staking.sol` - `stakeWithPermit()`

**Description**: Using try ... catch {} would causes permit failures to be silently ignored

**Impact**: 
- When allowance is insufficient, the transaction reverts with a misleading error (e.g., "insufficient allowance") instead of indicating that the permit has failed.
- the transaction is successful but the permit actually fails when the allowance is enough.

**Root Cause**: 
try ... catch {} do not distinguish error types and emit the event.

**My POC Walkthrough (optional)**：
```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.30;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {IERC165} from "openzeppelin-contracts/contracts/utils/introspection/IERC165.sol";
import {IAccessControl} from "openzeppelin-contracts/contracts/access/IAccessControl.sol";
import {Pausable} from "openzeppelin-contracts/contracts/utils/Pausable.sol";
import {IERC20Errors} from "openzeppelin-contracts/contracts/interfaces/draft-IERC6093.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {IMigratorV1} from "../src/interfaces/IMigratorV1.sol";
import {IMigratorReceiverV2} from "../src/interfaces/IMigratorReceiverV2.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("TestToken") ERC20("TestToken", "TTKN") {}
}

contract UnsupportedToken is ERC20Permit {
    constructor() ERC20Permit("UnsupportedToken") ERC20("UnsupportedToken", "UTKN") {}
}

contract Mytest is Test {
    Staking public staking;

    address public admin = address(2);
    address public manager = address(3);
    address public migrator = address(4);
    address public pauser = address(5);
    address public alice = address(6);
    address public bob = address(7);
    uint256 public charliePrivateKey = 0xca51;
    address public charlie = vm.addr(charliePrivateKey);

    Token public token;
    UnsupportedToken public unsupportedToken;

    function setUp() public {
        token = new Token();
        unsupportedToken = new UnsupportedToken();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migrator);
    }

    function test_Staking_RevertWhen_DifferentPersonSignature() public {
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

    function test_Staking_When_PermitExpired() public {
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

    // Helpers

    function addStakingPeriodByManager(
        uint256 cap,
        uint64 stakingDurationSeconds,
        uint64 unlockDurationSeconds,
        uint32 aprBps,
        bool isActive
    ) internal returns (uint8 periodIndex) {
        vm.prank(manager);
        periodIndex = staking.addStakingPeriod(cap, stakingDurationSeconds, unlockDurationSeconds, aprBps, isActive);
    }

    function approveAndStake(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer
    ) internal returns (uint8 stakeIndex) {
        vm.startPrank(user);
        token.approve(address(staking), amount);
        stakeIndex = staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            })
        );
        vm.stopPrank();
    }

    function stake(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer
    ) internal returns (uint8 stakeIndex) {
        vm.prank(user);
        stakeIndex = staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            })
        );
    }

    function stakeWithPermit(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) internal returns (uint8 stakeIndex) {
        vm.prank(user);
        stakeIndex = staking.stakeWithPermit(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            }),
            deadline,
            v,
            r,
            s
        );
    }

    function signCharliePermit(uint256 amount, uint256 deadline) internal view returns (uint8 v, bytes32 r, bytes32 s) {
        return vm.sign(
            charliePrivateKey,
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    token.DOMAIN_SEPARATOR(),
                    keccak256(
                        abi.encode(
                            keccak256(
                                "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                            ),
                            charlie,
                            address(staking),
                            amount,
                            token.nonces(charlie),
                            deadline
                        )
                    )
                )
            )
        );
    }

    function calculateReward(uint256 stakingAmount, uint256 stakingTime, uint256 stakingApyBps)
        internal
        pure
        returns (uint256)
    {
        return (stakingAmount * stakingTime * stakingApyBps) / (365 days * 10000);
    }
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

**Severity**: Low

**Location**: `Staking.sol` - `_getAccrued()`

**Description**: Due to integer division rounding down, users staking very small amounts receive zero reward despite locking their tokens for the full duration.

**Impact**: Users cannot receive any reward despite locking their tokens

**Root Cause**: The calculation is (stakingAmount * stakingTime * stakingApyBps) / (365 days * 10000)

**My POC Walkthrough (optional)**：
```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.30;

import {ERC20Permit, ERC20} from "openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";
import {IERC165} from "openzeppelin-contracts/contracts/utils/introspection/IERC165.sol";
import {IAccessControl} from "openzeppelin-contracts/contracts/access/IAccessControl.sol";
import {Pausable} from "openzeppelin-contracts/contracts/utils/Pausable.sol";
import {IERC20Errors} from "openzeppelin-contracts/contracts/interfaces/draft-IERC6093.sol";
import {Test} from "forge-std/Test.sol";
import {Staking} from "../src/Staking.sol";
import {IStakingV1} from "../src/interfaces/IStakingV1.sol";
import {IMigratorV1} from "../src/interfaces/IMigratorV1.sol";
import {IMigratorReceiverV2} from "../src/interfaces/IMigratorReceiverV2.sol";

contract Token is ERC20Permit {
    constructor() ERC20Permit("TestToken") ERC20("TestToken", "TTKN") {}
}

contract UnsupportedToken is ERC20Permit {
    constructor() ERC20Permit("UnsupportedToken") ERC20("UnsupportedToken", "UTKN") {}
}

contract Mytest is Test {
    Staking public staking;

    address public admin = address(2);
    address public manager = address(3);
    address public migrator = address(4);
    address public pauser = address(5);
    address public alice = address(6);
    address public bob = address(7);
    uint256 public charliePrivateKey = 0xca51;
    address public charlie = vm.addr(charliePrivateKey);

    Token public token;
    UnsupportedToken public unsupportedToken;

    function setUp() public {
        token = new Token();
        unsupportedToken = new UnsupportedToken();
        staking = new Staking(admin, manager, pauser, address(token));

        vm.prank(admin);
        staking.grantRole(keccak256("MIGRATOR"), migrator);
    }

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

    // Helpers

    function addStakingPeriodByManager(
        uint256 cap,
        uint64 stakingDurationSeconds,
        uint64 unlockDurationSeconds,
        uint32 aprBps,
        bool isActive
    ) internal returns (uint8 periodIndex) {
        vm.prank(manager);
        periodIndex = staking.addStakingPeriod(cap, stakingDurationSeconds, unlockDurationSeconds, aprBps, isActive);
    }

    function approveAndStake(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer
    ) internal returns (uint8 stakeIndex) {
        vm.startPrank(user);
        token.approve(address(staking), amount);
        stakeIndex = staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            })
        );
        vm.stopPrank();
    }

    function stake(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer
    ) internal returns (uint8 stakeIndex) {
        vm.prank(user);
        stakeIndex = staking.stake(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            })
        );
    }

    function stakeWithPermit(
        address user,
        uint8 periodIndex,
        uint256 amount,
        uint64 maxStakingDurationSeconds,
        uint64 maxUnlockDurationSeconds,
        uint32 minAprBps,
        address referrer,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) internal returns (uint8 stakeIndex) {
        vm.prank(user);
        stakeIndex = staking.stakeWithPermit(
            periodIndex,
            amount,
            IStakingV1.StakeParams({
                maxStakingDurationSeconds: maxStakingDurationSeconds,
                maxUnlockDurationSeconds: maxUnlockDurationSeconds,
                minAprBps: minAprBps,
                referrer: referrer
            }),
            deadline,
            v,
            r,
            s
        );
    }

    function signCharliePermit(uint256 amount, uint256 deadline) internal view returns (uint8 v, bytes32 r, bytes32 s) {
        return vm.sign(
            charliePrivateKey,
            keccak256(
                abi.encodePacked(
                    "\x19\x01",
                    token.DOMAIN_SEPARATOR(),
                    keccak256(
                        abi.encode(
                            keccak256(
                                "Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"
                            ),
                            charlie,
                            address(staking),
                            amount,
                            token.nonces(charlie),
                            deadline
                        )
                    )
                )
            )
        );
    }

    function calculateReward(uint256 stakingAmount, uint256 stakingTime, uint256 stakingApyBps)
        internal
        pure
        returns (uint256)
    {
        return (stakingAmount * stakingTime * stakingApyBps) / (365 days * 10000);
    }
}
```

**Fix**: Revert when the reward equal to 0. Alternatively, enforce a minimum stake amount in _stake to prevent dust-level stakes.

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;

// Fixed
uint256 reward = (amount * stakingPeriod.aprBps * stakingPeriod.stakingDurationSeconds) / rewardBpsDenominator;
if (reward == 0) revert RewardTooSmallToBeNonZero();
```

### [L-02]: Stale Migration Permit Allows Asset Theft After Role Restoration
**Severity**: Medium / High (取决于平台对资产转移的定级)

**Location**: Staking.sol - setMigrationPermit()

**Description**:
The setMigrationPermit function includes an unconditional check for the MIGRATOR_ROLE:

```solidity
if (!hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
When a migrator loses this role, users who had previously granted them permission are unable to revoke it (setMigrationPermit(..., false) will revert). This creates a permanent "stale state" where migrationPermits[migrator][user] remains true.
```
**Impact**:

Loss of User Control (Low/Griefing): Users cannot revoke permissions for deprecated migrators, violating the documented functionality.

Unauthorized Asset Migration (Medium/High): If the admin later re-grants the MIGRATOR_ROLE to this address, the migrator can immediately call migratePositionsFrom(user) and transfer the user's staked assets without their renewed consent.

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
Fix:
Restrict the role check to only when granting permissions (as seen in the patched version), and/or remove redundant state updates to avoid gas waste:

```solidity
if (migrationPermits[_migrator][msg.sender] == _isMigrationPermitted) return;
if (_isMigrationPermitted && !hasRole(MIGRATOR_ROLE, _migrator)) revert MigratorNotFound(_migrator);
migrationPermits[_migrator][msg.sender] = _isMigrationPermitted;
```
**My Biggest Takeaway**:

Version Control is Critical: Testing against the fixed main branch made me think there was no bug. I had to use git checkout to revert to the audited commit to make the POC work.

Push the Impact: The official report stopped at "User cannot revoke". By extending the POC, I proved "User's funds can be stolen". This is the difference between a Low and a High severity.
