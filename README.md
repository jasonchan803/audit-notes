# Audit Notes

My personal notes for smart contract audit learning.

## Structure
- `/low` - Low severity vulnerabilities
- `/medium` - Medium severity vulnerabilities

## Progress
- 2026-05-12: Created repository, starting with zero-address check.


# 漏洞：缺少零地址检查

**报告来源**：OpenZeppelin Sonic Gateway 审计报告  
**漏洞位置**：`setTrustedForwarder` 函数  
**危害**：管理员可能误设零地址，导致元交易功能永久失效  
**修复方式**：增加 `require(forwarder != address(0))`  
**我的英语总结**：  
`Missing zero-address check in setTrustedForwarder allows admin to set forwarder to address(0), which would disable all meta-transactions.`

**代码示例（漏洞版本 vs 修复版本）**：

\`\`\`solidity
// 漏洞版本
function setTrustedForwarder(address forwarder) external onlyOwner {
    trustedForwarder = forwarder;
}

// 修复版本
function setTrustedForwarder(address forwarder) external onlyOwner {
    require(forwarder != address(0), "Invalid forwarder");
    trustedForwarder = forwarder;
}
\`\`\`
