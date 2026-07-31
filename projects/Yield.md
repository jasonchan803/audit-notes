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

### [H-01]: `stir` 函数允许自我转账导致余额重复覆盖

**Severity**: High

**Location**: `Cauldron.sol` 中的 `stir()` : L267-L295

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
