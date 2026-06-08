---
type: project-note
project: PoolTogether Micro Contest #1
audit-source: Code4rena
date: 2026-06-05
tags: [yield-source, permission, transfer, rug-pull]
status: in-progress
---

# PoolTogether Micro Contest #1 Audit Learning Notes

## Project Brief
PoolTogether 是一个无损彩票协议。用户存入 DAI 等资产，利息汇入奖池，每周开奖一次。本审计针对其收益源可切换模块 `SwappableYieldSource`。

## High Risk Findings

## [H-01]: swapYieldSource 权限过大

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` : L307-L315

**Description**: 
Owner 或 AssetManager 可以随时将收益源地址切换到自己控制的恶意合约，只要该合约实现了 `IYieldSource` 接口。切换后，所有资金会通过内部逻辑转移到新收益源，导致 rug pull。

**Impact**: 
资金完全丢失。即使用户不作恶，私钥泄露同样导致毁灭性后果。

**Root Cause**: 
缺少对收益源合约的信任校验，且切换操作无延迟和多签控制。

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 
使用多签治理 + Timelock 控制该权限，或完全移除该功能。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
function swapYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager {
    _setYieldSource(_newYieldSource);
    // 没有校验新源的安全性，也没有延迟
}

// Fixed
function swapYieldSource(IYieldSource _newYieldSource) external onlyTimelock {
    require(_newYieldSource.isTrusted(), "Not trusted");
    // 或者只允许切换到预先注册的白名单地址
}
```

**English Takeaway**:
Administrative roles with the power to migrate user funds must be protected by timelocks and multi-signature wallets.

## [H-02]: redeemToken 功能失效

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `redeemToken()`: L240

**Description**: 
当用户赎回存款时，`_burnShares(amount)` 会先销毁用户在`SwappableYieldSource` 中的份额 ，然后`yieldSource.redeemToken()` 会将存款从 `YieldSource` 收益源合约中赎回到本合约 `SwappableYieldSource` 中，然后再通过 `_depositToken.safeTransferFrom()` 将存款转回给用户，问题就是 `safeTransferFrom()` 是一个授权转账函数，本来就是自己转账出去，直接转就行了,不应该自己授权自己再转账，这样不但耗费更多gas，而且还可能导致失败，因为 `safeTransferFrom()` 有可能需要授权额度才能转账成功，而代码中并没有自己给自己授权额度，所以会导致转账失败

**Impact**: 
用户提现有可能会失败，导致无法取回资金

**Root Cause**: 
`safeTransferFrom()`的错误使用，由于 `supplyTokenTo()`存款的时候使用了，所以开发者认为赎回也是同样的函数，实际他并没有了解清楚整个资金的流动，赎回是直接转回用户，和存款不一样，不需要授权

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 
把 `safeTransferFrom()` 修改为  `safeTransfer()` 

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
_depositToken.safeTransferFrom(address(this), msg.sender, redeemableBalance);

// Fixed
_depositToken.safeTransfer(msg.sender, redeemableBalance);
```

**English Takeaway**:
Prioritize using safeTransfer()to transfer funds when the funds are in the contract. This directly sends the funds to the user and does not require any approval of allowance.


## [H-03]: setYieldSource 会导致份额结果出错

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `setYieldSource()`: L268-L271

**Description**: 
当使用  `setYieldSource()` 单独切换收益源地址时，在旧收益源地址的资金没有转移到新的收益源地址之前，会出现一个真空期，此时存款到 `SwappableYieldSource` 中，获得的份额会比实际份额要多，用户可以在这个真空期使用少量的资金获取大量的份额，等资金全部转移到新收益源地址后直接提现获利

**Impact**: 
可造成合约的资金损失

**Root Cause**: 
在这个真空期，新收益源地址的一开始的供应和余额为0，此时存款和份额是以当前收益源地址的供应 `totalSupply()` 和余额 `yieldSource.balanceOfToken(address(this)` 计算的,而不是旧的收益源，所以当前存款会获取到更多的份额

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
删除 `setYieldSource()` 外部调用 ，只通过 `_setYieldSource()` 内部调用切换收益源地址

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  /// @notice Set new yield source.
  /// @dev This function is only callable by the owner or asset manager.
  /// @param _newYieldSource New yield source address to set.
  /// @return true if operation is successful.
  function setYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager returns (bool) {
    _setYieldSource(_newYieldSource);
    return true;
  }

// Fixed

```

**English Takeaway**:
The swap of the YieldSource cannot be separated into two steps. Keep it as an atomic operation to prevent users from depositing before the funds are transferred from the old YieldSource to the new one.


## [H-04]: 

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` 和 `_setYieldSource()`

**Description**: 


**Impact**: 


**Root Cause**: 


**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 


**Code (Vulnerable & Fixed)**:
```solidity

```

**English Takeaway**:



## [H-05]: 


**Severity**: High


**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` 和 `_setYieldSource()`


**Description**: 


**Impact**: 


**Root Cause**: 


**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 


**Code (Vulnerable & Fixed)**:
```solidity

```

**English Takeaway**:



## Medium Risk Findings

### [M-01]: swapYieldSource 权限过大导致资金被 rug

**Severity**: [Critical/High/Medium/Low/Informational]

**Location**: [合约文件:行号 或 函数名]

**Description**: [用自己的话描述]

**Impact**: [后果]

**Root Cause**: [一句话原因]

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: [修复方式]

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
```

**English Takeaway**: [1句英文总结]

## Low Risk Findings

### [L-01]: swapYieldSource 权限过大导致资金被 rug

**Severity**: [Critical/High/Medium/Low/Informational]

**Location**: [合约文件:行号 或 函数名]

**Description**: [用自己的话描述]

**Impact**: [后果]

**Root Cause**: [一句话原因]

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: [修复方式]

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
```
### [H-04]: swapYieldSource 权限过大

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` 和 `_setYieldSource()`

**Description**: 


**Impact**: 


**Root Cause**: 


**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 


**Code (Vulnerable & Fixed)**:
```solidity

```

**English Takeaway**:

### [H-02]: swapYieldSource 权限过大

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` 和 `_setYieldSource()`

**Description**: 


**Impact**: 


**Root Cause**: 


**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 


**Code (Vulnerable & Fixed)**:
```solidity

```

**English Takeaway**:


## Summary & Takeaways
- 任何允许管理员单方面转移用户资产的函数都应视为高危，除非有强力缓冲机制。
- 资金流转时，区分 transfer 和 transferFrom 的使用场景至关重要。
- 审计不能假设管理员诚实，应设计最小权限原则。

## Further Thoughts
- [延伸思考]
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
