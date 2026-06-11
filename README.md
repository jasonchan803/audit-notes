# Audit Notes

My personal notes for smart contract audit learning.

## Structure
- `/low` - Low severity vulnerabilities
- `/medium` - Medium severity vulnerabilities

## Progress
- 2026-05-12: Created repository, start learning.
- 2026-05-14: finished Uniswap-ERC20ETH audit-notes.
- 2026-05-14: finished BARD-Token audit-notes.
- 2026-05-25：finished EIP-712 knowledage-note.
- 2026-05-26: finished eip712-version-consistency knowledage-note.
- 2026-06-09: finished high risk findings in PoolTogether Micro Contest #1 project-notes.
- 2026-06-11: finished safeApprove-vs-safeIncreaseAllowance knowledage-note.


## Total numbers of notes：6
## Current stage：First stage(Zero business logic vulnerabilities)





---
type: audit-note
project: [项目名称]
audit-source: [Code4rena / OpenZeppelin / 其他]
tags: [tag1, tag2]
date: YYYY-MM-DD
status: [in-progress/completed]
---

## [漏洞编号] [漏洞名称]

**Severity**: [Critical/High/Medium/Low/Informational]

**Location**: [合约文件:行号 或 函数名]

**Description**: 
[用自己的话，2-3句话说清楚漏洞是什么]

**Impact**: 
[如果被利用，会造成什么后果？]

**Root Cause**: 
[一句话说清楚：为什么会产生这个漏洞？]

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 
[如何修复？]

**Code (Vulnerable & Fixed)**:

```solidity
// Vulnerable
[漏洞代码]

// Fixed
[修复代码]
```

**English Takeaway**: 
[1句英文总结]





---
type: project-note
project: [Project Name]
audit-source: [Code4rena / OpenZeppelin / Other]
date: YYYY-MM-DD
tags: [tag1, tag2]
status: [in-progress/completed]
---

## Project Brief
[用一两句话说清楚这个项目是做什么的]

## High Risk Findings

### [H-01]: 

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






---
type: knowledge-note
topic: [主题名称，如 safeApprove-vs-safeIncreaseAllowance]
date: YYYY-MM-DD
status: completed
---

## [主题名称]：[一句话概括核心问题]

**来源**：[可选] 哪份审计报告/哪个项目中学到的（如 PoolTogether Micro Contest）

**背景/场景**：
[用一两句话描述在什么业务场景下会遇到这个问题]

**问题/错误模式**：
[清楚地说明错误写法是什么、为什么错]

**影响**：
[如果写错会有什么后果？—— 功能失效？资金损失？]

**正确做法**：
[应该如何正确实现？给出代码示例或原则]

**代码对比（Vulnerable vs Fixed）**：

```solidity
// ❌ 错误写法
...

// ✅ 正确写法
...
```

**延伸思考（可选）**：
[如果有历史版本、相关标准、或你个人的独立见解，可以写在这里]

**为什么值得记录**：
[你从中学到了什么可以迁移到其他审计场景的原则或思维模式]
