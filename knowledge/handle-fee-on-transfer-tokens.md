---
type: knowledge-note
topic: handle-fee-on-transfer-tokens
date: 2026-06-12
status: completed
---

## 处理转账即燃烧/收费代币：实际到账金额 ≠ 参数金额

**来源**：PoolTogether Micro Contest – SwappableYieldSource 审计 (M-03)

**背景/场景**：
某些代币（如 USDT 在部分链上、SAITAMA、AKRO 等）在转账时会自动扣除一定比例作为手续费或燃烧（通缩）。当合约调用 `token.safeTransferFrom(from, to, amount)` 后，`to` 地址实际增加的余额可能小于 `amount`。

**问题/错误模式**：
许多协议（包括 SwappableYieldSource 的原始代码）直接使用 `amount` 作为用户存款的记账数值：
```solidity
_depositToken.safeTransferFrom(msg.sender, address(this), amount);
// 然后按 amount 铸造 shares 或增加用户余额
```
如果代币有手续费，用户实际只存入了 amount - fee，但合约却记为 amount。这会导致：
- 用户份额被高估，可以从金库提取多于实际存入的资产。
- 金库的总账面余额与真实余额永久不符，最终可能被攻击者套利。

**影响**：
轻则会计错误，重则资金损失（如果用户利用高估的份额提现）。

**正确做法**：
获取转账前后合约地址的代币余额差值，作为实际到账金额：
```solidity
uint256 balanceBefore = _depositToken.balanceOf(address(this));
_depositToken.safeTransferFrom(msg.sender, address(this), amount);
uint256 actualReceived = _depositToken.balanceOf(address(this)) - balanceBefore;
// 后续使用 actualReceived 记账
```

**代码对比（Vulnerable vs Fixed）**：

```solidity
// ❌ 错误：直接使用 amount，忽略手续费
function supplyTokenTo(uint256 amount, address to) external {
    _depositToken.safeTransferFrom(msg.sender, address(this), amount);
    _mintShares(amount, to);
}

// ✅ 正确：计算实际到账金额
function supplyTokenTo(uint256 amount, address to) external {
    uint256 before = _depositToken.balanceOf(address(this));
    _depositToken.safeTransferFrom(msg.sender, address(this), amount);
    uint256 actual = _depositToken.balanceOf(address(this)) - before;
    _mintShares(actual, to);
}
```

**延伸思考（可选）**：
- 同样的模式也适用于** rebasing 代币**（如 aToken, stETH），因为余额会随时间变化。通常审计中会建议：涉及此类代币时，采用“份额（shares）”模型，或者在每次交互时实时计算余额。
- 对于 transfer 操作（比如提现），也需要考虑类似的逻辑吗？提现时一般是从合约转给用户，如果代币有转账费，用户实际收到会少于合约转出的数量，这需要项目方明确设计（通常认为用户承担费用）。

**为什么值得记录**：
这是审计中一个经典且容易遗漏的非标准 ERC20 行为。任何与代币交互的协议（存款、取款、奖励分发）都应注意不能盲目信任 transfer/transferFrom 的参数值。学会这个模式后，你将来审计借贷协议、收益聚合器等时，会本能地检查是否处理了 fee-on-transfer 代币。
