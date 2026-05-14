---
type: audit-note
project: BARD Token
severity: Low
tags: 
 - Duplicate Event
date: 2026-05-14
status: completed
---

## L01：Duplicate Event Emission in mint

**Severity**: Low

**Location**: `BARD.sol`:`L46`

**Description**: 
多余的`Mint(to, amount)`(L46)事件，由于调用`mint()`同时会触发`Transfer(address(0), to, amount)`，这两个事件所触发的内容都是一样的，
同样在`constructor`中也调用了`mint()`，但是却没有使用这个事件，前后的设计不一致。

**Impact**: 
触发多余的事件会增加gas的消耗

**Root Cause**: 
设计冗余了，其实并不需要这个事件，本身合约就自带事件了

**Fix**: 
删除`Mint(to, amount)`事件和相关函数的代码

**English Takeaway**: 
When you call the mint function, it will emit a custom Mint(to, amount)event and a Transfer(address(0), to, amount)event, 
but actually these events convey the same information. Therefore, I suggest removing the redundant Mintevent to simplify the code.

**Code (Vulnerable & Fixed)**:

```solidity
// Vulnerable
emit Mint(to, amount)

// Fixed
// removing the redundant Mint event
```
