---
type: audit-note
project: Uniswap ERC20ETH
severity: Informational
tags: 
- floating pragma
- incomplete docstrings
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
```

## L-02: Incomplete Docstrings

**Severity**: [Informational]

**Location**: `ERC20Eth.sol`: `L27-L27`, `L32-L32`, `L36-L38`, `L45-L45`

**Description**: 
1. `name()`(L27), `symbol()`(L32)和`totalSupply()`(L36-L38)，这3个函数都缺失了@return的标签去分别记录他们的返回值
2. `balanceOf(address)`(L45)函数里面address后面并没有使用一个变量名参数，没有见名知意
3. `totalSupply()`(L36-L38)函数上面多了一行空格，导致这个函数和它的注释分开了

**Impact**: 
不会有严重的后果，但是对于开发和协助来说，规范的注释有助于理解代码

**Root Cause**: 
代码书写不规范

**Fix**: 
重新规范书写，如添加@return标签、增加参数等

**English Takeaway**: 
The completed docstrings help us understand the code more clearly.

**Code (Vulnerable & Fixed)**:

```solidity
// Vulnerable
 function balanceOf(address) public pure override returns (uint256)

// Fixed
/// @return the name of the token.
/// @return the symbol of the token.
/// @return the amount of tokens in existence.
/// @param account The address to query the balance for
function balanceOf(address account) public pure override returns (uint256)
```
