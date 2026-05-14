# Audit Notes

My personal notes for smart contract audit learning.

## Structure
- `/low` - Low severity vulnerabilities
- `/medium` - Medium severity vulnerabilities

## Progress
- 2026-05-12: Created repository, starting with zero-address check.



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
