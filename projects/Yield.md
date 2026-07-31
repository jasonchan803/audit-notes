---
type: project-note
project: Yield
audit-source: Code4rena
date: 2026-07-31
tags: [tag1, tag2]
status: in-progress
---

## Project Brief
Yield是一个固定利息借贷的Defi平台。用户可以存入多种资产作为抵押，借出fytoken并且可以通过池子兑换成对应的base资产，同时用户也可以通过存入对应的base资产买入fytoken，到期后获得固定的利息收益，这个平台集合了借贷、兑换、清算等一系列功能

## High Risk Findings

### [H-01]: `stir` 函数允许自我转账导致余额重复覆盖

**Severity**: [Critical/High/Medium/Low/Informational]

**Location**: `Cauldron.sol` 中的 `stir()` : L267-L295

**Description**: 用户可以传入相同的地址参数到from和to中，使得 balances[from] = balances[to]，当进行ink和art的单独计算后， balances[from] = balancesFrom 和 balances[to] = balancesTo 等同于同一个value赋值两遍，最后的balancesTo的值会覆盖前面balancesFrom的值

**Impact**: 让这个value凭空增加的抵押物资产，通过重复操作从而掏空协议的抵押物资产

**Root Cause**:  由于balancesTo才是这个用户最后覆盖的值，当执行balancesFrom.ink -= ink 和 balancesTo.ink += ink时，balancesFrom的值最后会被balancesTo的值覆盖，等同于这个value凭空增加了ink

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 在获取到From和To地址后，需要检查两个value是否相同

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
        (DataTypes.Vault memory vaultFrom, , DataTypes.Balances memory balancesFrom) = vaultData(from, false);
        (DataTypes.Vault memory vaultTo, , DataTypes.Balances memory balancesTo) = vaultData(to, false);

        require (vaultFrom != vaultTo, "Different vault");

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

**English Takeaway**: [1句英文总结]


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
