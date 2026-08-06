---
type: project-note
project: Yield
audit-source: Code4rena
date: 2026-07-31
tags: [tag1, tag2]
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

## Medium Risk Findings（仅记录新模式）

### [M-01]: 

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
