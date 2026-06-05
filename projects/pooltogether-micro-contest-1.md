---
type: project-note
project: PoolTogether Micro Contest #1
date: 2021-07
audit-source: Code4rena
tags: [yield-source, permission-control, transfer]
---

# PoolTogether Micro Contest #1 审计学习笔记

## 项目简述
[用一两句话说清楚这个项目是做什么的]

## 高危漏洞

### H-01: swapYieldSource 权限过大导致资金被 rug
**位置**：`SwappableYieldSource.swapYieldSource()`
**描述**：...
**影响**：...
**修复**：...
**我的POC思路**：...

### H-02: redeemToken 错误使用 transferFrom
...

### H-03: transferERC20 允许转走 depositToken（独立发现）
...

## 中危漏洞（仅记录新模式）

### M-01: 滑点攻击
（如果之前没见过，记下来；否则不记）

## 低危漏洞（仅记录从未见过的）

### L-01: 使用 block.timestamp 作为随机数源
...

## 总结与收获
- 权限控制必须假设管理员不可信，应使用多签+Timelock
- 资金流转时要区分 `transfer` 和 `transferFrom` 的使用场景
- ...

## 延伸思考
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
