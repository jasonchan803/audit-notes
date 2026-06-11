---
type: knowledge-note
topic: SafeERC20Upgradeable
date: 2026-06-11
status: completed
---

## safeApprove 的误用：为什么不能用来增加授权额度

**来源**：PoolTogether Micro Contest – MStableYieldSource 审计

**背景/场景**：
合约需要保持对某收益池（Savings）的无限授权（`type(uint256).max`），以便随时将用户的代币存入。当授权额度因部分消耗而降低时，需要有一个紧急函数重新授权到最大值。

**问题/错误模式**：
开发者使用了 OpenZeppelin 的 `safeApprove()` 来实现重新授权：`mAsset.safeApprove(address(savings), type(uint256).max);`但 safeApprove() 的设计是：仅当当前授权额度为 0 时才能成功设置非零值；如果当前额度非零，它会直接 revert。这是为了避免 approve 竞态条件（ERC20 标准的历史问题）。

因此，第一次授权后（如构造函数中已设为 max），该紧急函数将永远无法再次执行，形同虚设。

**影响**：
如果授权额度因某些操作意外降低（例如部分代币被转移），后续的存款将因授权不足而失败，导致用户无法存入资金。

**正确做法**：
使用 safeIncreaseAllowance() 来增加授权额度，或者先 safeDecreaseAllowance 到 0 再 safeApprove。推荐使用 safeIncreaseAllowance()。

**代码对比（Vulnerable vs Fixed）**：
```solidity
// ❌ 错误：总是 revert（除非授权额度恰好为 0）
mAsset.safeApprove(address(savings), type(uint256).max);

// ✅ 正确：增加差额到最大值
IERC20 _mAsset = mAsset;
address _savings = address(savings);
uint256 _allowance = _mAsset.allowance(address(this), _savings);
_mAsset.safeIncreaseAllowance(_savings, type(uint256).max - _allowance);
```

**延伸思考**：
OpenZeppelin 自 v4.x 起已经弃用 safeApprove，推荐使用 safeIncreaseAllowance / safeDecreaseAllowance。理解其设计原因（防止 approve 竞态条件）比记忆函数名更重要。

**为什么值得记录**：
1. 这是 ERC20 授权机制中一个经典的安全陷阱，许多老项目或新手仍会踩坑。
2. 学会区分“初始设置授权”和“增加现有授权”的不同安全要求，可迁移到其他需要谨慎处理状态更新的场景。
