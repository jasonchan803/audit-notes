---
type: audit-note
project: Uniswap ERC20ETH
severity: Informational
tags: 
  - floating pragma
  - incomplete Docstrings
date: 2026-05-14
status: completed
---

## L-01: Floating Pragma

**Severity**: Informational

**Location**: `ERC20Eth.sol:2`

**Description**: 
这个合约使用了浮动的Pragma版本，solidity ^0.8.13也就是0.8.13以上的版本都兼容

**Impact**: 
如果不使用固定的版本，有可能会出现部署失败或者运行异常

**Root Cause**: 
当选择较新版本（如0.8.20），需要注意0.8.20引入了PUSH0操作码，某些链（如Arbitrum、Optimism的旧版本、Polygon zkEVM等）未必支持PUSH0，从而导致部署失败或者运行异常

**Fix**: 
使用固定的pragma版本，如solidity 0.8.13

**English Takeaway**: 
The contract may fail to deploy or run due to the use of a floating pragma, which may not support the PUSH0 opcode.

**Code (Vulnerable & Fixed)**:

```solidity
// Vulnerable
pragma solidity ^0.8.13;

// Fixed
pragma solidity 0.8.13;
