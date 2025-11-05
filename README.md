## 一、项目整体说明

**项目名（示例）**：FairPhoenix Launch

**核心流程：**

1. 部署代币合约 `FairPhoenixToken`（ERC20，带税费、黑白名单、限转、交易阶段等）。
2. 部署 Fair Launch 合约 `FairPhoenixFairLaunch`：

   * 用户在一段时间内，用原生币（ETH/BNB）参与筹资；
   * 没有固定代币价格，**所有参与者按出资占比瓜分一部分代币**（Fair Launch 精髓）。
3. Fair Launch 结束：

   * 合约自动 / 半自动调用 UniswapV2/PancakeSwap Router：

     * 用 **募集到的全部/大部分 ETH/BNB + 一部分代币** 创建初始流动性池（加池）；
   * 根据「代币投入 LP / 资金总额」的比例，自然形成开盘价；
   * Token 合约 `enableTrading()`，正式开盘；
   * 参与者调用 `claim()` 领取自己的代币份额。
4. 项目方可以选择：

   * 把 LP Token 锁仓 / 销毁；
   * 锁定税率、关闭增发等，模拟「放权」。

---

## 二、合约模块与角色

### 2.1 模块

1. `FairPhoenixToken`

   * ERC20 代币 + 税费 + 黑白名单 + 多种转账限制 + 交易阶段。
2. `FairPhoenixFairLaunch`

   * Fair Launch 募资合约：只收一种币（ETH/BNB），按比例分配代币；
   * 负责调用 Router 添加 LP、触发开盘。
3. （可选）`LPLocker`

   * 简单 LP 锁仓合约，锁定一定时间，练习锁仓逻辑。

> 你可以把 Launch 的逻辑直接写在 `FairPhoenixFairLaunch` 里，这样模块更少。

### 2.2 主要角色

* `owner`：项目方钱包，初期持有代币合约和 FairLaunch 的控制权。
* `router`：UniswapV2 或 PancakeSwap Router 地址，通过构造函数指定：

  * 例如 BSC：Pancake Router v2。
* `WETH` / `WBNB`：从 Router 读取或构造时传入。

---

## 三、FairPhoenixToken（代币合约）需求

### 3.1 基础参数

* 标准：ERC20（OpenZeppelin 风格）。
* 参数：

  * `name`, `symbol`, `decimals=18`；
  * `maxTotalSupply`：例如 `1_000_000_000 * 1e18`。
* 供应策略：

  * 合约支持 `mint`，但总供应量不能超过 `maxTotalSupply`；
  * 在部署后：

    * mint 一部分给团队 / 运营；
    * mint 一部分给 `FairLaunch` 合约用于：

      * 一部分作为 Fair Launch 分配给参与者；
      * 一部分用于 Dex 加池；
    * 其余作为预留（营销、空投等）。

### 3.2 权限与状态

* `Ownable`：

  * `transferOwnership`, `renounceOwnership`；
* 关键不可逆开关（用来练）：

  * `bool mintingDisabled;`

    * `disableMintingForever()`：调用后再也不能 `mint`。
  * `bool taxesLocked;`

    * `lockTaxesForever()`：调用后不可再更改税率。
  * `bool limitsDisabled;`

    * `disableAllLimits()`：关闭所有限购/限卖/冷却等。

### 3.3 税费机制

**税率：**

* `uint16 buyTax;`
* `uint16 sellTax;`
* `uint16 transferTax;`
* 单位：100 = 1%，最大 ≤ 2000（20%）。

**税收归集与分配：**

* 税费在 `_transfer` 中计算并扣除，累计到合约余额。
* 税费分配比例：

  ```solidity
  struct FeeDistribution {
      uint16 marketing;
      uint16 lp;
      uint16 treasury;
  }
  FeeDistribution public feeDistribution;
  ```
* 提供 `processFees()`，将累计税费：

  * 部分换成 ETH/BNB；
  * 部分加池；
  * 部分转到营销地址。

**买卖识别：**

* `mapping(address => bool) automatedMarketMakerPairs;`
* 规则：

  * `from` 是 pair → 买入；
  * `to` 是 pair → 卖出；
  * 两者都不是 → 普通转账。

**税率管理：**

* `updateBuyTax`, `updateSellTax`, `updateTransferTax`：

  * `onlyOwner`；
  * 不能超过上限；
  * 若 `taxesLocked == true` 则 `revert`。

---

### 3.4 交易阶段 + 全局开关

**阶段：**

```solidity
enum TradingPhase { NotStarted, FairLaunch, Launch, Normal }
TradingPhase public tradingPhase;
```

* `NotStarted`：仅 owner / 特定白名单可转；
* `FairLaunch`：Token 正在为 Fair Launch 准备/锁仓，普通交易依然关闭；
* `Launch`：刚加池那段「上线初期」，有强限制；
* `Normal`：进入正常交易状态，多数限制关闭或放松。

**交易开关：**

* `bool tradingEnabled;`
* `_transfer` 中：

  * 若 `!tradingEnabled` 且双方都不是豁免地址 → `revert`。
* `enableTrading()`：

  * 仅允许 `FairPhoenixFairLaunch` 调用；
  * 只允许执行一次；
  * 设置 `tradingEnabled = true; tradingPhase = TradingPhase.Launch;`
  * 记录 `launchBlock = block.number;`

---

### 3.5 转账 / 交易限制（现实里常见的都给上）

#### 3.5.1 最大交易额度 / 最大持仓

* `uint256 maxTxAmount;`
* `uint256 maxWalletAmount;`
* `_transfer` 中：

  * 若 `from`/`to` 不是豁免地址 (`isExcludedFromLimits`)：

    * `amount <= maxTxAmount`；
    * `balanceOf(to) + amount <= maxWalletAmount`；
* 提供：

  * `setMaxTxAmount`, `setMaxWalletAmount`；
  * `disableMaxLimits()`：将两者设为 `type(uint256).max`，并标记 `limitsDisabled = true`。

#### 3.5.2 P2P 转账限制（常见防场外 OTC）

* 在 `Launch` 阶段：

  * 对于非白名单：

    * 若 `from` & `to` 都不是 AMM pair，且都不是豁免地址 → 禁止（防止私下大量转移）。
* 在 `Normal` 阶段：

  * P2P 限制可以关闭（配置开关 `p2pRestrictionEnabled`），只保留黑名单等基本风控。

#### 3.5.3 冷却时间（Cooldown）

* 变量：

  * `uint256 tradeCooldown;` （如 30 秒）
  * `mapping(address => uint256) lastTradeTime;`
* 规则（对普通地址）：

  * 每次买入/卖出/转账时：

    * `require(block.timestamp >= lastTradeTime[addr] + tradeCooldown);`
    * 更新 `lastTradeTime[addr]`。
* 只在 `Launch` 阶段启用，`Normal` 阶段可关闭。

#### 3.5.4 每周期卖出限制（类似线性解锁）

* 可选功能（进阶练习）：

  * 按「周期」限制卖出的总量：

    * `periodDuration`（如 1 天）；
    * `maxSellPercentPerPeriod`（如 20% 持仓）。
  * 对「卖出」(`to` 是 AMM pair)：

    1. 若 `block.timestamp > periodStart[addr] + periodDuration` → 重置计数；
    2. 计算 `soldInPeriod[addr] + amount <= balanceOf(addr) * maxSellPercentPerPeriod / 100`；
    3. 更新 `soldInPeriod`。

---

### 3.6 黑名单 / 白名单 / 费用豁免

* 黑名单：

  * `mapping(address => bool) blacklist;`
  * `_beforeTokenTransfer` / `_transfer` 检查，黑名单地址直接 `revert`。
* 白名单：

  * `mapping(address => bool) tradingWhitelist;` 用于：

    * 在 `tradingEnabled == false` 时允许少数地址操作（如 FairLaunch 合约、owner）。
  * `mapping(address => bool) p2pWhitelist;` 用于 P2P 限制例外。
* 费用豁免：

  * `mapping(address => bool) isExcludedFromFees;`
  * 如：FairLaunch 合约、Router、LP 对等。

---

### 3.7 增发与销毁

* `mint(address to, uint256 amount) onlyOwner`：

  * 确保 `totalSupply + amount <= maxTotalSupply`；
  * 若 `mintingDisabled` 为 true，则禁止。
* `disableMintingForever()`：设置 `mintingDisabled = true`。
* `burn`, `burnFrom`：标准实现。

---

## 四、FairPhoenixFairLaunch（Fair Launch 合约）需求

### 4.1 Fair Launch 基本逻辑

**特点：**

* 筹资期间没有代币固定价格；
* 所有参与者按出资比例瓜分**预留的 Fair Launch 代币池**；
* 所有/大部分筹资资金用于**创建 LP**，而不是让项目方提走（这也是很多 fair launch 卖点）。

**参数：**

* `address token;` → `FairPhoenixToken`
* `address router;` → UniswapV2 / PancakeSwap Router
* `uint256 fairLaunchTokenPool;` → Fair Launch 分配给用户的代币量
* `uint256 liquidityTokenPool;` → 用于加池的代币量
* 时间：

  * `startTime`, `endTime`
* 参与限制：

  * `minContribution`, `maxContribution`（每地址）
  * `hardCap`（总募集上限，可选）
* 是否有软顶 `softCap`：

  * 可选：

    * 有软顶：未达软顶 → 退款；
    * 没软顶：只要 > 0 就成功，以筹到多少为准。

### 4.2 参与规则（Fair）

* 用户在 `[startTime, endTime]` 期间用 ETH/BNB 调用 `contribute()`。
* 合约记录：

  * `mapping(address => uint256) contributions;`
  * `uint256 totalRaised;`
* 限制：

  * 单地址 `contributions[addr] + msg.value` 在 `[minContribution, maxContribution]` 范围内；
  * 若 `hardCap > 0`，`totalRaised + msg.value <= hardCap`；
* 可筛掉合约地址（只允许 EOA），作为简单反 Bot（`require(tx.origin == msg.sender)`）。

> 这里的「fair」主要体现在：**最后按比例瓜分代币，没有早鸟价格优势**。

### 4.3 Fair Launch 结束与加池（DEX）

结束后，Owner 或指定角色调用 `finalizeFairLaunch()`：

1. 检查时间：

   * `block.timestamp >= endTime`

2. 判断是否成功：

   * 若设置了 `softCap` 且 `totalRaised < softCap` → 标记失败，开启退款；
   * 否则 → 成功。

3. 成功路径：

   * 从 Token 合约 转/铸 `liquidityTokenPool` + `fairLaunchTokenPool` 到本合约；
   * 调用 Router 的 `addLiquidityETH`：

     * 传入：

       * `token = FairPhoenixToken`；
       * `amountTokenDesired = liquidityTokenPool`；
       * `amountETHDesired = totalRaised * liquidityETHPercent`（通常 100%）；
   * 获得 LP Token：

     * 可选：立即发送到 `LPLocker` 锁定一定时间；
     * 或发送给 owner（用来练 LP 风险）。
   * 剩余 ETH/BNB（若有）：

     * 按预设比例转给项目方 / 运营 / 库房。
   * 通知 Token 合约：

     * 调用 `token.setAutomatedMarketMakerPair(pair, true)`；
     * 调用 `token.enableTrading()` 开盘；
     * 设置 Token 阶段为 `Launch`，进入「高限制阶段」。

4. 失败路径：

   * 标记 `failed = true`；
   * 用户可调用 `refund()` 取回原始出资。

### 4.4 代币领取（按比例）

在 `finalizeFairLaunch()` 成功后：

* 每个地址可领取的代币数：

  ```text
  userShare = contributions[user] / totalRaised
  userTokens = fairLaunchTokenPool * userShare
  ```
* 用户通过 `claim()` 领取，合约：

  * 从自身余额转 `userTokens` 给用户；
  * 记录 `claimed[user] = true` 或已领取数量。

> 这样：
>
> * 价格由：`liquidityTokenPool : LP 中 ETH/BNB` 决定；
> * 用户实际「买入成本」取决于他在 totalRaised 里的占比和最终流动池价格，完全由市场自行决定。

### 4.5 与 Token 限制的联动

* 在 Fair Launch 阶段：

  * Token 的 `tradingPhase == FairLaunch`；
  * `tradingEnabled == false`，禁止公开交易；
* `finalizeFairLaunch()` 成功后：

  * 调 Token：`enableTrading()` → 进入 `Launch` 阶段；
  * 此时：

    * 各种限购 / 冷却 / P2P 限制生效；
    * 例如前 `botBlockCount` 个区块对非白名单地址高税 / 黑名单；
  * 一段时间后（由 owner 调），可以：

    * 调 Token：`setTradingPhase(TradingPhase.Normal)`；
    * 关闭部分限制（如 `disableMaxLimits()`、`setCooldownEnabled(false)`）；
    * 锁定税率、锁定 mint 等，模拟对外宣称的「已放权」。

---

## 五、LPLocker（可选，简单锁仓合约）

* 提供 `lock(address token, address lpToken, uint256 amount, uint256 unlockTime)`；
* 存储：

  * LP Token 持有地址、解锁时间；
* 仅在 `block.timestamp >= unlockTime` 时允许 `withdraw()`；
* 这个合约只是为了练你「LP 锁仓」（也是你工作中会分析的点）。

---

## 六、你可以按什么顺序来实现

1. **第一步：**

   * 实现 `FairPhoenixToken` 的基础 ERC20 + Ownable + 简单税费（不分买卖）+ `enableTrading()`；
2. **第二步：**

   * 实现 `FairPhoenixFairLaunch` 的基本 fair launch（筹资 + finalize + 按比例 claim），先不用 Router，加池逻辑可以写死或先空函数。
3. **第三步：**

   * 接入 UniswapV2/PancakeSwap Router，完成 `addLiquidityETH`，实盘加池；
   * 在 Token 里支持 `automatedMarketMakerPairs` + 区分买卖税；
4. **第四步：**

   * 把所有转账限制补齐：maxTx、maxWallet、P2P 限制、冷却时间、botBlock 高税等；
5. **第五步：**

   * 做 LP 锁仓、税率/增发永久锁死、阶段切换等细节。

这样一轮下来，你对 **发行盘 + Fair Launch + 加池 + 开盘限制** 的完整套路就会从「看别人」变成「自己写过一遍」。
如果你接下来想，我可以直接把 **Token 和 FairLaunch 的函数清单 / 事件名草稿** 列出来，方便你贴着写 Solidity。
