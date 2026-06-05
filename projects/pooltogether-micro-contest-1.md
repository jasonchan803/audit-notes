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

## Vulnerability List

## High Risk Findings

### [H-01]: swapYieldSource 权限过大 (Severity: High)

-**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` 和 `_setYieldSource()`

-**Description**: 
Owner 或 AssetManager 可以随时将收益源地址切换到自己控制的恶意合约，只要该合约实现了 `IYieldSource` 接口。切换后，所有资金会通过内部逻辑转移到新收益源，导致 rug pull。

-**Impact**: 
资金完全丢失。即使用户不作恶，私钥泄露同样导致毁灭性后果。

-**Root Cause**: 
缺少对收益源合约的信任校验，且切换操作无延迟和多签控制。

-**Fix**: 
使用多签治理 + Timelock 控制该权限，或完全移除该功能。

-**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
function swapYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager {
    _setYieldSource(_newYieldSource);
    // 没有校验新源的安全性，也没有延迟
}

// Fixed (示例)
function swapYieldSource(IYieldSource _newYieldSource) external onlyTimelock {
    require(_newYieldSource.isTrusted(), "Not trusted");
    // 或者只允许切换到预先注册的白名单地址
}
```

-**English Takeaway**:
Administrative roles with the power to migrate user funds must be protected by timelocks and multi-signature wallets.
...

（后续漏洞同理）

## Summary & Takeaways
- 任何允许管理员单方面转移用户资产的函数都应视为高危，除非有强力缓冲机制。
- 资金流转时，区分 transfer 和 transferFrom 的使用场景至关重要。
- 审计不能假设管理员诚实，应设计最小权限原则。

## Further Thoughts
- [延伸思考]
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
