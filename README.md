# Audit Notes

My personal notes for smart contract audit learning.

## Structure
- `/low` - Low severity vulnerabilities
- `/medium` - Medium severity vulnerabilities

## Progress
- 2026-05-12: Created repository, start learning.
- 2026-05-14: finished Uniswap-ERC20ETH-audit-notes.
- 2026-05-14: finished BARD-Token-Audit-notes.
- 2026-05-25：finished EIP-712. 深入理解 EIP-712，并完成审计视角的知识笔记
- 2026-05-26: finished eip712-version-consistency. 审计 Note：EIP-712 版本一致性（ANVL Token 新旧部署）

## Total numbers of notes：5
## Current stage：First stage(Zero business logic vulnerabilities)





---
type: audit-note
project: [项目名称]
severity: [Critical/High/Medium/Low/Informational]
tags: [tag1, tag2]
date: YYYY-MM-DD
status: [in-progress/completed]
---

## [漏洞编号] [漏洞名称]

**Severity**: [等级]

**Location**: [合约文件:行号 或 函数名]

**Description**: 
[用自己的话，2-3句话说清楚漏洞是什么]

**Impact**: 
[如果被利用，会造成什么后果？]

**Root Cause**: 
[一句话说清楚：为什么会产生这个漏洞？]

**Fix**: 
[如何修复？]

**English Takeaway**: 
[1句英文总结]

**Code (Vulnerable & Fixed)**:

```solidity
// Vulnerable
[漏洞代码]

// Fixed
[修复代码]
```





---
type: project-note
project: [项目名称]
audit-source: [Code4rena / OpenZeppelin / 其他]
date: YYYY-MM-DD
tags: [tag1, tag2]
status: completed
---

# [Project Name] Audit Learning Notes

## Project Brief
[用一两句话说清楚这个项目是做什么的]

## High Risk Findings

### [H-01]: swapYieldSource 权限过大导致资金被 rug

**Location**: [合约文件:行号 或 函数名]

**Description**: [用自己的话描述]

**Impact**: [后果]

**Root Cause**: [一句话原因]

**Fix**: [修复方式]

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
```

**My POC Walkthrough (optional)**：[我的POC思路]

**English Takeaway**: [1句英文总结]

### [H-02]: redeemToken 错误使用 transferFrom
...

### [H-03]: transferERC20 允许转走 depositToken（独立发现）
...

## Medium Risk Findings（仅记录新模式）

### [M-01]: 滑点攻击
（如果之前没见过，记下来；否则不记）

## Low Risk Findings（仅记录从未见过的）

### [L-01]: 使用 block.timestamp 作为随机数源
...

## Summary & Takeaways
- [总结和收获]
- 任何允许管理员单方面转移用户资产的函数都应视为高危，除非有强力缓冲机制。
- 资金流转时，区分 transfer 和 transferFrom 的使用场景至关重要。
- 审计不能假设管理员诚实，应设计最小权限原则。

## Further Thoughts
- [延伸思考]
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
