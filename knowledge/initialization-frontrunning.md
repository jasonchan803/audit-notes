---
type: knowledge-note
topic: initialization-frontrunning
date: 2026-06-13
status:	completed
---

## 初始化函数能被抢跑

**来源**：PoolTogether Micro Contest – SwappableYieldSource 审计 (L-01)

**背景**：
某些合约使用可升级模式或分离式部署，将初始化逻辑放在一个独立的 `initialize()` 函数中（而非构造函数）。如果该函数是 `public` 或 `external`，且部署后没有立即原子调用，攻击者可以抢跑（front-run），用恶意参数初始化合约，从而获得控制权。

**风险场景**：
1. 合约部署交易发送后，初始化交易在另一个区块中执行。
2. 攻击者观察到部署交易，立即发出更高的 gas 调用 `initialize()`，抢先设置参数。
3. 合法的初始化交易失败，合约已被攻击者控制。

**问题/错误模式**：
部署和初始化缺少原子性，中间存在抢跑风险

**影响**：
Owner所有权丢失，所有onlyOwner函数无法调用，收益源被恶意设置，导致逻辑混乱甚至用户的资产受到损失

**正确做法**：
- **原子部署**：使用工厂合约，在创建合约后立即在同一交易中调用 `initialize()`。
- **构造函数初始化**：如果不需要可升级，直接在构造函数中完成初始化。
- **初始化保护**：使用 OpenZeppelin 的 `Initializable` 库，防止重复初始化（但不能防止抢跑）。
- **部署脚本**：确保部署脚本在同一交易中完成合约创建和初始化。

**代码对比（Vulnerable vs Fixed）**：

```solidity
// ❌ 危险：独立的 public 初始化函数
contract Vulnerable {
    address public owner;
    function initialize(address _owner) public {
        owner = _owner;
    }
}

// ✅ 安全：使用工厂原子部署
contract Factory {
    function create() external returns (address) {
        Vulnerable v = new Vulnerable();
        v.initialize(msg.sender);
        return address(v);
    }
}
```

**延伸思考（可选）**：
- 该模式同样适用于可升级合约（UUPS / Transparent Proxy）。initialize 函数必须保护且与部署原子执行。
- 审计中如果发现合约存在 initialize 函数但未在构造函数中调用，且未使用工厂模式，应标记为 Low/Medium（取决于权限影响）。

**为什么值得记录**：
这是一个反复出现的初始化安全模式，理解它可以帮助你在审计中快速识别部署阶段的风险。
