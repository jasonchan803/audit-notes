---
type: project-note
project: PoolTogether Micro Contest #1
audit-source: Code4rena
date: 2026-06-05
tags: [yield-source, permission, transfer, rug-pull]
status: in-progress
---

## Project Brief
PoolTogether 是一个无损彩票协议。用户存入 DAI 等资产，利息汇入奖池，每周开奖一次。本审计针对其收益源可切换模块 `SwappableYieldSource`。

## High Risk Findings

## [H-01]: swapYieldSource 权限过大

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `swapYieldSource()` : L307-L315

**Description**: 
Owner 或 AssetManager 可以随时将收益源地址切换到自己控制的恶意合约，只要该合约实现了 `IYieldSource` 接口。切换后，所有资金会通过内部逻辑转移到新收益源，导致 rug pull。

**Impact**: 
资金完全丢失。即使用户不作恶，私钥泄露同样导致毁灭性后果。

**Root Cause**: 
缺少对收益源合约的信任校验，且切换操作无延迟和多签控制。

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 
使用多签治理 + Timelock 控制该权限，或完全移除该功能。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
function swapYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager {
    _setYieldSource(_newYieldSource);
    // 没有校验新源的安全性，也没有延迟
}

// Fixed
function swapYieldSource(IYieldSource _newYieldSource) external onlyTimelock {
    require(_newYieldSource.isTrusted(), "Not trusted");
    // 或者只允许切换到预先注册的白名单地址
}
```

**English Takeaway**:
Administrative roles with the power to migrate user funds must be protected by timelocks and multi-signature wallets.

## [H-02]: redeemToken 赎回功能失效

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `redeemToken()`: L240

**Description**: 
当用户赎回存款时，`_burnShares(amount)` 会先销毁用户在`SwappableYieldSource` 中的份额 ，然后`yieldSource.redeemToken()` 会将存款从 `YieldSource` 收益源合约中赎回到本合约 `SwappableYieldSource` 中，然后再通过 `_depositToken.safeTransferFrom()` 将存款转回给用户，问题就是 `safeTransferFrom()` 是一个授权转账函数，本来就是自己转账出去，直接转就行了,不应该自己授权自己再转账，这样不但耗费更多gas，而且还可能导致失败，因为 `safeTransferFrom()` 有可能需要授权额度才能转账成功，而代码中并没有自己给自己授权额度，所以会导致转账失败

**Impact**: 
用户提现有可能会失败，导致无法取回资金

**Root Cause**: 
`safeTransferFrom()`的错误使用，由于 `supplyTokenTo()`存款的时候使用了，所以开发者认为赎回也是同样的函数，实际他并没有了解清楚整个资金的流动，赎回是直接转回用户，和存款不一样，不需要授权

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 
把 `safeTransferFrom()` 修改为  `safeTransfer()` 

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
_depositToken.safeTransferFrom(address(this), msg.sender, redeemableBalance);

// Fixed
_depositToken.safeTransfer(msg.sender, redeemableBalance);
```

**English Takeaway**:
Prioritize using safeTransfer()to transfer funds when the funds are in the contract. This directly sends the funds to the user and does not require any approval of allowance.


## [H-03]: setYieldSource 会导致份额结果出错

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `setYieldSource()`: L268-L271

**Description**: 
当使用  `setYieldSource()` 单独切换收益源地址时，在旧收益源地址的资金没有转移到新的收益源地址之前，会出现一个真空期，此时存款到 `SwappableYieldSource` 中，获得的份额会比实际份额要多，用户可以在这个真空期使用少量的资金获取大量的份额，等资金全部转移到新收益源地址后直接提现获利

**Impact**: 
可造成合约的资金损失

**Root Cause**: 
在这个真空期，新收益源地址的一开始的供应和余额为0，此时存款和份额是以当前收益源地址的供应 `totalSupply()` 和余额 `yieldSource.balanceOfToken(address(this)` 计算的,而不是旧的收益源，所以当前存款会获取到更多的份额

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
删除 `setYieldSource()` 外部调用 ，只通过 `_setYieldSource()` 内部调用设置新的收益源地址

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function setYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager returns (bool) {
    _setYieldSource(_newYieldSource);
    return true;
  }

// Fixed

```

**English Takeaway**:
The swap of the YieldSource cannot be separated into two steps. Keep it as an atomic operation to prevent users from depositing before the funds are transferred from the old YieldSource to the new one.


## [H-04]: transferFunds 有被套利的风险

**Severity**: High

**Location**: `SwappableYieldSource.sol` 中的 `transferFunds()` : L296-L300

**Description**: 
`transferFunds()` 里面只有检查收益源地址和原来的是否不一致 `_requireDifferentYieldSource(_yieldSource)`，并没有检查收益源的代币是否和原来一致，假设新收益源地址和旧的收益源地址的代币不一致，且价值也不一致，转移相同数量的代币后仍有剩余价值停留在 `SwappableYieldSource` 中， 那么 Owner 或者 AssetManager 可以通过 `transferERC20()` 把剩余在 `SwappableYieldSource` 的余额转走

**Impact**: 
用户资金丢失，部分资金可以被 Owner 或者 AssetManager 转走

**Root Cause**: 
`transferFunds()` 缺少收益源代币检查，当代币价值不一致时，剩余资金可以停留在 `SwappableYieldSource` 中

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
删除 `transferFunds()` 外部调用 ，先通过 `_setYieldSource()` 内部调用检查收益源代币地址一致并且设置新的收益源地址，然后通过 `_transferFunds()` 内部调用转移资金到新的收益源地址。

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function transferFunds(IYieldSource _yieldSource, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    _requireDifferentYieldSource(_yieldSource);
    _transferFunds(_yieldSource, amount);
    return true;
  }

// Fixed

```

**English Takeaway**:
Keep checking that the deposit token is the same when the owner or the asset manager calls _transferFunds().


## [H-05-self]: transferERC20 可以转走包括收益源代币在内的所有代币


**Severity**: High


**Location**: `SwappableYieldSource.sol` 中的  `transferERC20()`: L323-L329


**Description**: 
 `transferERC20()` 只检查了转账的代币是否与收益源地址不一致，并没有检查是否和收益源代币的地址不一致，当收益源代币地址一致，且出现收益源代币停留在 `SwappableYieldSource` 中的情况（比如由于gas不足，设置完新收益源地址后，不能及时的把资金转移到新的收益源地址），那么 Owner 或者 AssetManager 可以转走停留在 `SwappableYieldSource` 中的收益源代币

**Impact**: 
用户资金丢失，资金可以被 Owner 或者 AssetManager 转走

**Root Cause**: 
`transferERC20()` 缺少收益源代币检查

**My POC Walkthrough (optional)**：[我的POC思路]


**Fix**: 
`transferERC20()` 添加收益源代币检查，确保 Owner 或者 AssetManager 不能转走收益源代币，只能转移与收益源不相关的代币

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function transferERC20(IERC20Upgradeable erc20Token, address to, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    require(address(erc20Token) != address(yieldSource), "SwappableYieldSource/yield-source-token-transfer-not-allowed");
    erc20Token.safeTransfer(to, amount);
    emit TransferredERC20(msg.sender, to, amount, erc20Token);
    return true;
  }
}

// Fixed
  function transferERC20(IERC20Upgradeable erc20Token, address to, uint256 amount) external onlyOwnerOrAssetManager returns (bool) {
    require(address(erc20Token) != address(yieldSource), "SwappableYieldSource/yield-source-token-transfer-not-allowed");
    require(address(erc20Token) != address(yieldSource).depositToken(), "SwappableYieldSource/different-deposit-token");
    erc20Token.safeTransfer(to, amount);
    emit TransferredERC20(msg.sender, to, amount, erc20Token);
    return true;
  }
}
```

**English Takeaway**:
The erc20Token token must be verified to be different from the deposit token, which can prevent the owner or asset manager from transferring funds by calling transferERC20().


## Medium Risk Findings

### [M-01]: Ownership 转移不够严谨

**Severity**: Medium

**Location**: `OwnableUpgradeable.sol` 中的 `transferOwnership()`

**Description**: 在 `@openzeppelin/contracts-upgradeable": "3.4.0"` 版本中的 `OwnableUpgradeable.sol`，转移所有权只需要一步，Owner通过调用 `transferOwnership()` 传入一个新地址，就会直接更换到新的Owner，它这个函数只检查 `newOwner != address(0)` ,但不会检查这地址是否是正确的，这实际非常危险，因为Owner的权限非常大，可以设置新的收益源地址、可以转移收益收益源的资金到新的收益源、可以随意转移额外收入的代币、同时还可以转移新的Owner，以上所有的功能的都必须通过Owner才能操作，假设在转移新Owner的过程中，输入了错误的地址，或者Owner无意调用了 `renounceOwnership()` 那么相当于Owner的所有权权限丢失，上述的所有功能都失效了，只能通过重新部署合约才能恢复，并且所有用户都需要同步转移，这会导致用户对此项目失去信心
在OpenZeppelin v4.8.0版本之后，首次新增​ `Ownable2Step` 及 `Ownable2StepUpgradeable`，直接使用该库就可以实现两步双重确认，安全性更高

**Impact**: 所有权丢失，所有onlyOwner修饰符的函数全部无法调用

**Root Cause**: Owner的所有权转移只有一步，且还可以放弃所有权

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: [修复方式]
删除 `renounceOwnership()` 函数，避免出现错误的调用丢失所有权
使用新版的 `Ownable2StepUpgradeable.sol` 实现双重确认

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable

// Fixed
```

**English Takeaway**: It is very risky when you transfer ownership in one step. If an incorrect address is used accidentally, that will make all the onlyOwner functions unusable forever.

### [M-02] 非零授权额度使用safeApprove去增加授权额度会导致revert

See detailed note: [`knowledge/safeApprove-vs-safeIncreaseAllowance.md`](../knowledge/safeApprove-vs-safeIncreaseAllowance.md)

**English Takeaway**: The `safeApprove()` function increases the allowance of an address that already has some allowance, which will cause it to revert.

### [M-03] 收费/通缩的代币在转账过程中会导致实际金额丢失

See detailed note: [`knowledge/handle-fee-on-transfer-tokens.md`](../knowledge/handle-fee-on-transfer-tokens.md)

**English Takeaway**: The `supplyTokenTo()` function assumes the transferred amount equals the `amount` parameter, which fails for tokens with transfer fees or rebasing mechanisms. The fix is to compute actual received balance.

### [M-04] 切换新YieldSource后，旧YieldSource仍有无限授权

**Severity**: Medium

**Location**: `SwappableYieldSource.sol`

**Description**: 在切换新的收益源之后，旧的收益源由于没有取消授权，仍拥有 `SwappableYieldSource.sol` 无限的授权额度，假设旧的收益源出现漏洞被恶意利用或者出现私钥被盗等情况，资金就有可能出现严重的损失

**Impact**: 资金被旧的收益源全部转走

**Root Cause**: 由于旧的收益源在切换新的收益源后没有取消授权

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 在切换新收益源的时候同时取消旧收益源所有的授权

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function swapYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager returns (bool) {
    IYieldSource _currentYieldSource = yieldSource;
    uint256 balance = _currentYieldSource.balanceOfToken(address(this));

    _setYieldSource(_newYieldSource);
    _transferFunds(_currentYieldSource, balance);

    return true;
  }

// Fixed
  function swapYieldSource(IYieldSource _newYieldSource) external onlyOwnerOrAssetManager returns (bool) {
    IYieldSource _currentYieldSource = yieldSource;
    uint256 balance = _currentYieldSource.balanceOfToken(address(this));

    _setYieldSource(_newYieldSource);
    _transferFunds(_currentYieldSource, balance);  // 先转移资金
    // 转移成功后，撤销对旧收益源的授权
    IERC20Upgradeable(_currentYieldSource.depositToken()).safeApprove(address(_currentYieldSource), 0);   
    return true;
  }
```

**English Takeaway**: Recommend decreasing approval after swapping the yield source.

## Low Risk Findings

### [L-01] 初始化函数可以被抢跑

See detailed note: [`knowledge/initialization-frontrunning.md`](../knowledge/initialization-frontrunning.md)

**English Takeaway**: The `initialize()` function can be front-run if that function uses a public modifier and is not done atomically with creation.


### [L-02]: 缺少0地址检查(部分已修复/误报)

####1. `_newYieldSource` 零地址检查（实际已存在）

- **报告原称**：缺少对 `_newYieldSource` 的零地址检查。
- **实际代码**：在 `_requireDifferentYieldSource()` 函数（L251-L253）中已包含：
  ```solidity
  require(address(_yieldSource) != address(0), "...");

####2. `MStableYieldSource.sol` L85的零地址检查

**Severity**: Low

**Location**: `MStableYieldSource.sol` 中的 `supplyTokenTo()`: L82-L88

**Description**: 在 `supplyTokenTo` 函数中，用户调用时指定 `to` 地址，表示将存款产生的份额记入该地址的余额（imBalances）。由于未对 `to` 地址进行零地址检查，攻击者或用户可能误将 `to` 设为 `address(0)`，导致份额被记录在零地址上。虽然用户的代币已正常存入协议，但对应份额永久无法赎回，造成用户资金损失。

**Impact**: 用户存入的代币无法赎回（因为份额归属 address(0)），导致资产永久锁定在协议中。

**Root Cause**: 缺少0地址检查

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**: 增加0地址检查

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
    function supplyTokenTo(uint256 mAssetAmount, address to) external override nonReentrant {
        mAsset.safeTransferFrom(msg.sender, address(this), mAssetAmount);
        uint256 creditsIssued = savings.depositSavings(mAssetAmount);
        imBalances[to] += creditsIssued;

        emit Supplied(msg.sender, to, mAssetAmount);
    }

// Fixed
    function supplyTokenTo(uint256 mAssetAmount, address to) external override nonReentrant {
        require(address(to) != address(0)," MStableYieldSource: to address cannot be zero");
        mAsset.safeTransferFrom(msg.sender, address(this), mAssetAmount);
        uint256 creditsIssued = savings.depositSavings(mAssetAmount);
        imBalances[to] += creditsIssued;

        emit Supplied(msg.sender, to, mAssetAmount);
    }    
```

**English Takeaway**: Missing zero-address checks may lead to a later revert, gas wastage or even token burn.

### [L-07]: `_requireYieldSource` 函数没有检查返回值

**Severity**: Low

**Location**: `SwappableYieldSource.sol` 中的 `_requireYieldSource`: L74-L88

**Description**: `_requireYieldSource` 函数使用了 `staticcall`低级调用的方法去检查地址是否有 `depositToken()` 这个函数，实际上只要该地址有 `fallback()` 函数能正常调用且有返回的数据，那么 `staticcall` 就会返回true和返回fallback非零的数据，即使它没有 `depositToken()` 这个函数，在这样的情况下，就可以通过所有的检查

**Impact**: 恶意的地址可以通过所有的检查，完全依靠多签治理地址确保地址的安全

**Root Cause**: 使用了 `staticcall` 低级调用的方法去检查收益源地址的合法性

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**:  使用高级调用的方法确保收益源地址的合法性

**Code (Vulnerable & Fixed)**:
```solidity
// Vulnerable
  function _requireYieldSource(IYieldSource _yieldSource) internal view {
    require(address(_yieldSource) != address(0), "SwappableYieldSource/yieldSource-not-zero-address");

    (, bytes memory depositTokenAddressData) = address(_yieldSource).staticcall(abi.encode(_yieldSource.depositToken.selector));

    bool isInvalidYieldSource;

    if (depositTokenAddressData.length > 0) {
      (address depositTokenAddress) = abi.decode(depositTokenAddressData, (address));

      isInvalidYieldSource = depositTokenAddress != address(0);
    }

    require(isInvalidYieldSource, "SwappableYieldSource/invalid-yield-source");
  }
// Fixed
  function _requireYieldSource(IYieldSource _yieldSource) internal view {
    require(address(_yieldSource) != address(0), "SwappableYieldSource/yieldSource-not-zero-address");

    address depositTokenAddress = IYieldSource(_yieldSource).depositToken();

    require(depositTokenAddress != address(0), "SwappableYieldSource/invalid-yield-source");
  }  
```

**English Takeaway**: `staticcall` is a low-level opcode which cannot ensure that the contract actually has that function. Even if the success flag is checked, a malicious contract with a fallback() can still pass the validation. Always use high-level calls for interface checks.

### [L-10] `FundsTransferred` 事件中的 `amount` 返回的数据可能不准确

**Severity**: Low

**Location**: `SwappableYieldSource.sol` 中的 `_transferFunds()` :L288

**Description**:  `_transferFunds` 函数中的 `FundsTransferred(_yieldSource, _amount)` 使用的是 `_amount` ，但是实际转移的金额是 `currentBalance` ，假设出现 `currentBalance` >= `_amount` 的情况，那么实际转移的金额就应该是 `currentBalance` 而不是 `_amount`

**Impact**:  事件返回的金额数据有误，可能实际转移的金额比事件返回的数据要多

**Root Cause**: `FundsTransferred` 使用了 `_amount`

**My POC Walkthrough (optional)**：[我的POC思路]

**Fix**:  
使用 `currentBalance` 代替`_amount` 

**Code (Vulnerable & Fixed)**:
```solidity
// Before
emit FundsTransferred(_yieldSource, _amount);
// After
emit FundsTransferred(_yieldSource, currentBalance);
```

**English Takeaway**: Always ensure the logs return exactly correct data.

## Discussion & Takeaways

### L-03 (informational): `onlyOwner` for `approveMaxAmount()` – a design choice, not a bug

**Report claim**: 这个特权函数使用 `onlyOwner` 的修饰符但其他特权函数却使用 `onlyOwnerOrAssetManager` ，假设 owner 在添加 asset managers 后放弃了合约的所有权那么这个函数也将无法被调用。

**Project's response**:
- 最初项目方的想法是给函数额外一层安全保护
- 但经过后面的讨论，他们最终把函数改为 `public` ，因为这个函数只能增加当前收益源地址的授权额度，并不会造成资金的损失，所以可以让任何人在收益源授权额度过低的时候紧急调用，避免其他功能的调用出现问题

**My takeaway**:
- 不是说每一个特权函数设计不一致就是有风险有漏洞，某些函数就是为了权衡业务与理论风险而有意设计的
- 最终修改为使用 `public` 展示出来，有时候最简单的解决方案反而最安全（兼顾了业务与安全）
- 这个案例告诉我： **审计不只是指出理论风险，同时也要考虑业务逻辑与信任假设**

### L-05 (informational): `_requireYieldSource` does not guarantee valid yield source

**Report claim**: `_requireYieldSource` 这个函数只检查这个收益源是否是0地址、是否有depositToken这个函数不够安全，这样恶意的收益源地址（只要包含depositToken这个函数的）都可以通过这个检查

**Project's response**:项目方也很明确反驳了，本来owner就是治理多签地址，以后也是通过社区投票后确认后，owner就可以直接执行了，因为单一个人也不可能通过地址，需要多人同意后才可以执行切换收益源，在这个过程等于这个地址需要通过多个人的验证通过才去执行通过的，所以其实已经足够安全，不需要额外增加一个白名单验证，这反而增加检查的复杂性同时增加gas的消耗

**My takeaway**:
- 单独的技术检查不能代替治理和人为检查
- 在一些极其重要的操作中，真正的安全依赖于（多签治理）而不是代码本身
- 这个案例告诉我：**审计不只是了解代码本身，而是要明白整个系统是如何运作的**

### L-12 (informational): `MStableYieldSource` 增加一个转出其他代币函数

**Report claim**: 当有其他代币意外的转入 `MStableYieldSource` 合约时，这些代币需要一个函数转出，否则永远卡在合约当中无法取出

**My analysis**:
- `MStableYieldSource` 合约并没有像 `SwappableYieldSource` 合约中 `transferERC20` 类似的函数，所以意外转入的代币会卡在合约当中
- 但是在审计报告中的 `Balance of mAsset - total deposited amount of mAsset` 这个建议未必适合，因为加入计算差额的方法会增加复杂性和潜在的漏洞面
- 所以我会建议 `MStableYieldSource` 合约参考 `SwappableYieldSource` 中的 `transferERC20` 函数，仅允许转出非 `mAsset` 的代币，避免触及用户存款

**My takeaway**: The safest design is often the simplest. Not every "missing function" is worth adding. Some may introduce more risk than they solve.

## Summary & Takeaways
- 任何允许管理员单方面转移用户资产的函数都应视为高危，除非有强力缓冲机制。
- 资金流转时，区分 transfer 和 transferFrom 的使用场景至关重要。
- 审计不能假设管理员诚实，应设计最小权限原则。

## Further Thoughts
- [延伸思考]
- 如果使用 OpenZeppelin 的最新库，哪些问题会自动避免？
