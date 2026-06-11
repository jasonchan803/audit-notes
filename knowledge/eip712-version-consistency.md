---
type: knowledge-note
topic: EIP-712
date: 2026-05-25
status: completed
---

## 审计 Note：EIP-712 版本一致性（ANVL Token 新旧部署）

**来源**：Anvil Protocol Diff Audit

**业务场景**：ANVL 代币计划部署到新地址，与旧代币合约分开。

**审计发现**：新旧合约的 EIP-712 `version` 字段相同（例如都是 `"1"`）。

**潜在风险**：非安全漏洞（因 `verifyingContract` 已变），但会造成接入方混淆，难以区分新旧合约的签名来源。

**我的理解**：`version` 是 `domain separator` 的一部分，升级后可彻底从签名层隔离新旧合约。

**审计建议的权衡**：
升级 `version` 从 `"1"` 到 `"2"` 可以明确区分新旧合约的签名域，但其可行性取决于前端集成方式：

- 若前端通过合约的 `eip712Domain()` 动态获取 domain 数据：
升级 `version` 后，链上返回的新 `domain separator` 会自动生效，签名验证无缝切换，无中断风险。

- 若前端硬编码了 `version` 或整个 `domain separator`（当前约 90% 项目的实践）：
升级 `version` 会导致所有基于旧 `domain separator` 的签名立即失效，用户必须重新签名。此时必须同步更新前端的硬编码值，否则新签名无法通过验证。

**为什么值得记录**：真实案例将抽象标准转化为可权衡的业务决策。
