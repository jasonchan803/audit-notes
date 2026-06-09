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

## [H-02]: redeemToken 赎回功能失效

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
删除 `setYieldSource()` 外部调用 ，只通过 `_setYieldSource()` 内部调用设置新的收益源地址

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function setYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager returns (bool) {
    _setYieldSource(_newYieldSource);
    return true;
  }

// Fixed

```

**English Takeaway**:
The swap of the YieldSource cannot be separated into two steps. Keep it as an atomic operation to prevent users from depositing before the funds are transferred from the old YieldSource to the new one.


## [H-04]: transferFunds 有被套利的风险

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `transferFunds()` : L296-L300

**Description**: 
`transferFunds()` 里面只有检查收益源地址和原来的是否不一致 `_requireDifferentYieldSource(_yieldSource)`，并没有检查收益源的代币是否和原来一致，假设新收益源地址和旧的收益源地址的代币不一致，且价值也不一致，转移相同数量的代币后仍有剩余价值停留在 `SwappableYieldSource` 中， 那么 Owner 或者 AssetManager 可以通过 `transferERC20()` 把剩余在 `SwappableYieldSource` 的余额转走

**Impact**: 
用户资金丢失，部分资金可以被 Owner 或者 AssetManager 转走

**Root Cause**: 
`transferFunds()` 缺少收益源代币检查，当代币价值不一致时，剩余资金可以停留在 `SwappableYieldSource` 中

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
删除 `transferFunds()` 外部调用 ，先通过 `_setYieldSource()` 内部调用检查收益源代币地址一致并且设置新的收益源地址，然后通过 `_transferFunds()` 内部调用转移资金到新的收益源地址。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function transferFunds(IYieldSource _yieldSource, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    _requireDifferentYieldSource(_yieldSource);
    _transferFunds(_yieldSource, amount);
    return true;
  }

// Fixed

```

**English Takeaway**:
Keep checking that the deposit token is the same when the owner or the asset manager calls _transferFunds().


## [H-05-self]: transferERC20 可以转走包括收益源代币在内的所有代币


**Severity**: High


**Location**: `SwappableYieldSource.sol` 中的  `transferERC20()`: L323-L329


**Description**: 
 `transferERC20()` 只检查了转账的代币是否与收益源地址不一致，并没有检查是否和收益源代币的地址不一致，当收益源代币地址一致，且出现收益源代币停留在 `SwappableYieldSource` 中的情况（比如由于gas不足，设置完新收益源地址后，不能及时的把资金转移到新的收益源地址），那么 Owner 或者 AssetManager 可以转走停留在 `SwappableYieldSource` 中的收益源代币

**Impact**: 
用户资金丢失，资金可以被 Owner 或者 AssetManager 转走

**Root Cause**: 
`transferERC20()` 缺少收益源代币检查

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
`transferERC20()` 添加收益源代币检查，确保 Owner 或者 AssetManager 不能转走收益源代币，只能转移与收益源不相关的代币

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function transferERC20(IERC20Upgradeable erc20Token, address to, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    require(address(erc20Token) != address(yieldSource), "SwappableYieldSource/yield-source-token-transfer-not-allowed");
    erc20Token.safeTransfer(to, amount);
    emit TransferredERC20(msg.sender, to, amount, erc20Token);
    return true;
  }
}

// Fixed
  function transferERC20(IERC20Upgradeable erc20Token, address to, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    require(address(erc20Token) != address(yieldSource), "SwappableYieldSource/yield-source-token-transfer-not-allowed");
    require(address(erc20Token) != yieldSource.depositToken(), "SwappableYieldSource/different-deposit-token");
    erc20Token.safeTransfer(to, amount);
    emit TransferredERC20(msg.sender, to, amount, erc20Token);
    return true;
  }
}
```

**English Takeaway**:
The erc20Token token must be verified to be different from the deposit token, which can prevent the owner or asset manager from transferring funds by calling transferERC20().


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
