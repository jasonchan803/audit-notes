---
type: project-note
project: Yield
audit-source: Code4rena
date: 2026-07-31
tags: [atomicity, state-exposure]
status: in-progress
---

## Project Brief
Yield是一个固定利率借贷协议。用户存入抵押资产后可以借出fyToken（代表未来固定收益的债权），也可以通过存入base资产买入fyToken，到期后获得固定利息收益。协议集成了借贷、兑换、清算等核心功能。

## High Risk Findings

## [H-01]: `stir` 函数允许自我转账导致余额重复覆盖

**Severity**: High

**Location**: `Cauldron.sol` - `stir()` : L267-295

**Description**: 用户可以传入相同的参数到`from`和`to`中，此时 `balances[from]`和`balances[to]`指向同一个存储槽，当进行ink和art的单独计算后， `balances[from] = balancesFrom`和`balances[to] = balancesTo`等同于同一个存储槽赋值两遍，最后的`balancesTo`的值会覆盖前面`balancesFrom`的值

**Impact**: 攻击者可以通过自我转账凭空增加的抵押物（ink）或债务（art），破坏协议会计系统，可能导致无限增发或坏账。

**Root Cause**:  由于`balancesTo`才是这个用户最后覆盖的值，当执行`balancesFrom.ink -= ink`和` balancesTo.ink += ink`时，`balancesFrom`的值最后会被`balancesTo`的值覆盖，等同于这个value凭空增加了ink

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 需要检查`from`和`to`是否相同

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
    function stir(bytes12 from, bytes12 to, uint128 ink, uint128 art)
        external
        auth
        returns (DataTypes.Balances memory, DataTypes.Balances memory)
    {
        (DataTypes.Vault memory vaultFrom, , DataTypes.Balances memory balancesFrom) = vaultData(from, false);
        (DataTypes.Vault memory vaultTo, , DataTypes.Balances memory balancesTo) = vaultData(to, false);

        if (ink > 0) {
            require (vaultFrom.ilkId == vaultTo.ilkId, "Different collateral");
            balancesFrom.ink -= ink;
            balancesTo.ink += ink;
        }
        if (art > 0) {
            require (vaultFrom.seriesId == vaultTo.seriesId, "Different series");
            balancesFrom.art -= art;
            balancesTo.art += art;
        }

        balances[from] = balancesFrom;
        balances[to] = balancesTo;

        if (ink > 0) require(_level(vaultFrom, balancesFrom, series[vaultFrom.seriesId]) >= 0, "Undercollateralized at origin");
        if (art > 0) require(_level(vaultTo, balancesTo, series[vaultTo.seriesId]) >= 0, "Undercollateralized at destination");

        emit VaultStirred(from, to, ink, art);
        return (balancesFrom, balancesTo);
    }

// Fixed
    function stir(bytes12 from, bytes12 to, uint128 ink, uint128 art)
        external
        auth
        returns (DataTypes.Balances memory, DataTypes.Balances memory)
    {
        require(from != to, "Cauldron/self-stir-not-allowed");

        (DataTypes.Vault memory vaultFrom, , DataTypes.Balances memory balancesFrom) = vaultData(from, false);
        (DataTypes.Vault memory vaultTo, , DataTypes.Balances memory balancesTo) = vaultData(to, false);

        if (ink > 0) {
            require (vaultFrom.ilkId == vaultTo.ilkId, "Different collateral");
            balancesFrom.ink -= ink;
            balancesTo.ink += ink;
        }
        if (art > 0) {
            require (vaultFrom.seriesId == vaultTo.seriesId, "Different series");
            balancesFrom.art -= art;
            balancesTo.art += art;
        }

        balances[from] = balancesFrom;
        balances[to] = balancesTo;

        if (ink > 0) require(_level(vaultFrom, balancesFrom, series[vaultFrom.seriesId]) >= 0, "Undercollateralized at origin");
        if (art > 0) require(_level(vaultTo, balancesTo, series[vaultTo.seriesId]) >= 0, "Undercollateralized at destination");

        emit VaultStirred(from, to, ink, art);
        return (balancesFrom, balancesTo);
    }
```

**English Takeaway**: Always validate that from and to are distinct when a function assumes they refer to different entities. Failing to do so can cause accounting errors and asset duplication.

## [H-02]: 权限标识符ROOT与函数签名共享命名空间（0x00000000 碰撞）

**Severity**: High

**Location**: `AccessControl.sol` - `auth()`: L90-93、`Ladle.sol` - `_moduleCall`: L588-596

**Description**: 权限标识符ROOT的函数选择器为0x00000000，如果新模块的函数选择器也刚好是0x00000000，那么在管理员不知情的情况下把这个新模块函数授权给某些地址，相当于让这些地址意外的获取到ROOT管理员权限

**Impact**: 有可能在管理员不知情的情况下把ROOT权限授权给其他地址，导致这些地址获得完整的系统控制权

**Root Cause**: 权限标识符 ROOT 与函数签名共享 bytes4 命名空间。当新增模块的函数选择器恰好为 0x00000000 时，对 0x00000000 这个 role 的授权，会同时作用于该函数，导致意外获得 ROOT 权限。

**Fix**:
- 代码层面：使用独立的命名空间（如 `bytes32` 哈希）作为权限标识符，或限制 `grantRole` 不能授予 `ROOT`。
- 流程层面：通过 CI 自动化检查新增模块的函数签名，避免与 `ROOT` 碰撞。

**English Takeaway**: Never use function selectors as permission identifiers; they share a namespace with any future function that may be added to the system.


## [H-03] `log_2` 函数中 `>=` 与 `>` 的差异导致精度误差

**Severity**: High

**Location**: `Exp64x64.sol` – `log_2()` 函数中的位运算分支

**Description**: `log_2` 函数用于 YieldMath 核心定价计算。对比 V1 和 V2 版本发现，在边界条件判断中，V1 使用 `>=`，V2 使用 `>`，导致当 `b` 恰好等于 `0x100000000000000000000000000000000` 时，两个版本行为不一致。该偏差会传播到所有依赖 `log_2` 的定价计算中。

**Impact**: 定价偏差可能导致套利、资金缓慢流失。

**Root Cause**: 版本迭代中无意识修改了比较符号（`>=` → `>`），破坏了数学一致性。

**Fix**: 统一使用正确的比较符号（根据数学定义确定，建议使用 V1 的 `>=` 或重新验证）。

**English Takeaway**: Even a single-character difference in core math library can cause severe pricing errors; always compare code versions for unintended changes.

## [H-04]: `Ladle._redeem` 通过 `batch` 暴露，允许盗取 Ladle 持有的 fyToken

**Severity**: High

**Location**: `Ladle.sol` - `_redeem()`函数（通过`batch`调用）: L211-214

**Description**: Ladle._redeem 是 private 函数，但通过 batch 操作暴露给任何外部调用者。该函数没有进行权限检查，允许任何人调用 batch 并传入 Operation.REDEEM，将 Ladle 持有的所有 fyToken 赎回并转出底层资产。当 wad = 0 时，且fytoken系列到期后，调用 fyToken.redeem(to, 0) 会赎回 fyToken.balanceOf(address(this)))，即 Ladle 持有的全部 fyToken。

**Impact**: 攻击者可以一次性盗取 Ladle 持有的所有 fyToken，将其转换为底层资产（如 DAI）并转给自己。如果 Ladle 中有大量 fyToken，会导致协议资金损失。

**Root Cause**: batch 允许公开调用内部操作，且 _redeem 缺少对调用者或意图的验证。

**My POC Walkthrough (optional)**：
```solidity
// 攻击者调用 Ladle.batch
bytes12 vaultId = bytes12(0); // 任意值
bytes[] memory data = new bytes[](1);
data[0] = abi.encode(
    Ladle.Operation.REDEEM,
    abi.encode(
        seriesId,       // 任意有效 seriesId
        attacker,       // 攻击者地址
        0               // wad = 0，意味着赎回全部
    )
);
ladle.batch(operations, data);
// Ladle 持有的 fyToken 被赎回，底层资产转给攻击者
```

**Fix**: 
- 为 _redeem 添加权限检查，仅允许 auth 角色（如治理或协议自身）调用。
- 或者移除 Operation.REDEEM 分支，限制通过 batch 调用。
- 确保 Ladle 不会长期持有用户资产，或对持有的资产进行保护。
- 其实以上方法都不太好，限制了这个功能，等于用户也没法赎回他们的资产，其实最重要的是要确保这个函数不能滥用，必须把这个函数打包到其他功能中一起使用，不要让资金停留在Ladle当中，保持原子性这才最优解，batch虽然自由度很高，但同时也带了很多隐患

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable (batch 允许任何人调用 _redeem)
function _redeem(IFYToken fyToken, address to, uint256 wad) private returns (uint256) {
    return fyToken.redeem(to, wad != 0 ? wad : fyToken.balanceOf(address(this)));
}

// Fixed (添加权限修饰符)
function _redeem(IFYToken fyToken, address to, uint256 wad) private auth returns (uint256) {
    // 或者移除该操作，或仅允许特定模块调用
}
```

**English Takeaway**: Any public function that allows arbitrary calls to internal logic must enforce proper access control, especially when handling assets held by the contract.

## [H-05]: `batch` 操作非原子性，允许攻击者抢跑盗取用户存入 Ladle 的 fyToken

**Severity**: High

**Location**: `Ladle.sol` – `batch()` 函数及 `Operation.REDEEM` 分支

**Description**: `Ladle.batch` 允许用户将多个操作打包执行，但**不强制原子性**。用户可能先调用 `_transferToFYToken` 将 fyToken 转入 Ladle，再调用 `_redeem` 赎回。在两笔交易之间，用户的 fyToken 停留在 Ladle 中，攻击者可通过监控 mempool 发现用户的存入操作，并在用户执行赎回之前调用 `batch` + `_redeem(fyToken, attacker, 0)` 将资产盗走。

**Impact**:
- 用户预期通过两步操作完成赎回，但资产在中间状态被攻击者拦截。
- 攻击者无需依赖 Ladle 意外持有资产，只需等待正常用户操作即可触发。
- 用户资金直接损失，且难以追回。

**Root Cause**: 
1. `batch` 中的 `REDEEM` 操作没有与“资产存入”绑定，允许非原子执行。
2. `_redeem` 缺少权限检查，允许任何人消耗 Ladle 持有的 fyToken。
3. 协议没有为“中间状态”提供保护机制（如锁定或权限隔离）。

**My POC Walkthrough (optional)**：
1. 用户调用 `_transferToFYToken(fyToken, 1000)`，将 1000 fyToken 转入 Ladle。
2. 攻击者监控 mempool，发现该交易。
3. 攻击者在用户执行赎回之前，调用 `batch` + `_redeem(fyToken, attacker, 0)`。
4. Ladle 中的 1000 fyToken 被赎回为底层资产（如 DAI），转给攻击者。
5. 用户后续的赎回交易失败（Ladle 中已无余额），资金被盗。

**Fix**: 
1. **强制原子性**：要求存入和赎回必须在同一 `batch` 中完成，或设计为单笔交易。
2. **权限隔离**：为 `_redeem` 添加权限控制，仅允许原存入者或 `auth` 角色调用。
3. **临时锁定**：用户存入 fyToken 时，可以将其锁定，仅允许该用户赎回。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable (任何人都可消耗 Ladle 持有的 fyToken)
function _redeem(IFYToken fyToken, address to, uint256 wad) private returns (uint256) {
    return fyToken.redeem(to, wad != 0 ? wad : fyToken.balanceOf(address(this)));
}

// Fixed (限制调用者)
mapping(bytes6 => address) public fyTokenDepositor;
function _transferToFYToken(bytes6 seriesId, uint256 wad) private {
    fyTokenDepositor[seriesId] = msg.sender;  // 记录存款人
    IERC20(fyToken).safeTransferFrom(msg.sender, address(this), wad);
}
function _redeem(IFYToken fyToken, address to, uint256 wad) private {
    require(msg.sender == fyTokenDepositor[seriesId], "Only depositor");
    // ...
}
```

**English Takeaway**: Batch operations must enforce atomicity or isolate intermediate states. Any gap between two dependent transactions is a window for front-running attacks.

## Medium Risk Findings

### [M-01]: `vaultID` 可被抢注导致用户创建 `vault` 失败

**Severity**: Medium

**Location**: `Cauldron.sol` – `build()` 函数，`Ladle.sol` – `batch()` 中的 `BUILD` 操作

**Description**: 用户创建 vault 时需要指定 `vaultId`。攻击者可监控 mempool，在用户交易之前使用相同的 `vaultId` 调用 `batch` + `build` 抢先创建。由于 `Cauldron.build()` 会检查 `vaultId` 是否已被使用（L180），用户交易将失败。

**Impact**: 攻击者可无限干扰协议的正常用户操作，导致协议可用性下降，影响用户信任。虽不涉及资金损失，但属于有效的 DoS 攻击。

**Root Cause**: `vaultId` 由用户指定而非协议分配，导致存在“命名空间抢注”风险。

**My POC Walkthrough (optional)**：
1. Alice 提交 batch 交易创建 vault，`vaultId = 0x123...`
2. Eve 监控 mempool，用更高 gas 提交仅包含 `build(0x123...)` 的 batch 交易
3. Eve 抢跑成功，Alice 的交易因 `vaultId` 已存在而回滚

**Fix**:
1. 由协议自动分配 `vaultId`（需要调整 batch/caching 逻辑）。
2. 在 `batch` 中检测异常操作模式（如仅包含 `build` 的单操作批次），自动回滚。

**English Takeaway**: When users can choose their own resource identifiers, attackers can front-run and reserve them, causing denial of service.

### [M-02]: `Witch.grab()` 二次调用会导致金库原有的所有权丢失

**Severity**: Medium

**Location**: `Witch.sol` - `grab()`函数 : L49-54

**Description**: 由于第一次调用`Witch.grab()`时，金库的所有权已经转移给 Witch，再次调用`Witch.grab()`时，`vaultOwners[vaultId] = vault.owner` 会把原有的 owner 覆盖，那么即使债务完全还清，原 owner 也无法取回剩余的抵押物。

**Impact**: 当金库债务被全额清偿后，剩余的抵押物因原所有者信息丢失而无法退还给原用户。

**Root Cause**: `vaultOwners[vaultId] = vault.owner` 二次调用导致原数据被覆盖。

**My POC Walkthrough (optional)**：
1. Alice的金库被第一次扣押，但超过时间后仍未结清债务。
2. Alice的金库被二次扣押，再次调用 `Witch.grab()` 后，`vaultOwners[vaultId]` 变成 `Witch` 合约本身。
3. Alice的金库债务完全结清，Alice无法取回金库中剩余的抵押物。

**Fix**: 
在 `Witch.grab()` 添加额外的检查，当 `vaultOwners[vaultId]` 已存在数据时，不再重复调用 `vaultOwners[vaultId] = vault.owner` 。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
    function grab(bytes12 vaultId) public {
        DataTypes.Vault memory vault = cauldron.vaults(vaultId);
        vaultOwners[vaultId] = vault.owner;
        cauldron.grab(vaultId, address(this));
    }

// Fixed
    function grab(bytes12 vaultId) public {
        DataTypes.Vault memory vault = cauldron.vaults(vaultId);
        if (vaultOwners[vaultId] != address(this)){
            vaultOwners[vaultId] = vault.owner;
        }
        cauldron.grab(vaultId, address(this));
    }
```

**English Takeaway**: The original owner record is overwritten when grab() is called a second time, preventing the vault from being returned after debt is cleared.


## Low Risk Findings（仅记录从未见过的）

### [L-01]: 

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
[漏洞代码]

// Fixed
[修复代码]
```

**English Takeaway**: [1句英文总结]

## Summary & Takeaways
- [总结和收获]
- 任何允许管理员单方面转移用户资产的函数都应视为高危，除非有强力缓冲机制。
- 资金流转时，区分 transfer 和 transferFrom 的使用场景至关重要。
- 审计不能假设管理员诚实，应设计最小权限原则。

## Further Thoughts
- [延伸思考]
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
