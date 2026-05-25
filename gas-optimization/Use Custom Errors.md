## Custom Error vs Require String

```solidity
revert("xxx...");  // 会消耗更多 gas（字符串存储）
error NotOwner();  // 自定义错误
if (msg.sender != owner) revert NotOwner();  // 更省 gas
// 参考：Solidity 0.8.4+ 引入
```
