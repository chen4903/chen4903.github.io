---
layout: post
category: blockchain
---

## 1.简介

### 1.1组件

#### 1.1.1Agent Commerce Protocol(ACP)

> 一种新的开放标准，支持自主代理之间安全、可验证且高效的商业交易。

<img src="../assets/images/virtuals-acp.png" style="zoom: 33%;" />

#### 1.1.2Tokenization Platform

> 提供机制用于发行Agent Tokens和Business Tokens，内置激励对齐、流动性提供和公平发行原则。

**工作原理：**

1. 代币创建：一个创造者决定在Virtuals平台上启动一个新的AI Agent。
2. Bonding曲线设置：创建者支付100个$VIRTUAL代币，将为新Agent的代币创建一个与$VIRTUAL配对的bonding曲线。
3. 流动性池创建：一旦达到bonding曲线阈值（在bonding曲线中累积了42k个$VIRTUAL），Agent完成使命，并创建一个与$VIRTUAL代币配对的Agent代币流动性池
4. 流动性锁定：流动性池被锁定十年。每个Agent代币的固定供应量为10亿代币。

**Trading Fees：**所有交易将产生1%的fee。交易费用分配如下：

- 使命完成前（Prototype Agent）：1%的fee将进入协议金库
- 使命完成后（Sentient Agent）：
  - 70%进入agent创建者的钱包
  - 30%进入ACP

**Genesis Launch 采用贡献加权模型：**

- 每个积分奖励的有效期为 30 天
- 释放逻辑是实时的，并在滚动的24小时基础上重新计算
- 使用动态、点加权分配模型。
- 参与者承诺使用积分来竞争预售代币分配的37.5%份额。承诺的积分越多，最高分配潜力就越高——每位用户最多可获得代币总供应量的0.5%。
- 每项最大虚拟承诺金额：566 $VIRTUAL（含 1% 税）
- 未使用的 $VIRTUAL 和积分将自动退还
- 如果用户承诺的积分比其他用户多，他们将获得按比例增加的 Genesis 供应份额——最高可达每个用户的上限。
- 用户必须在质押积分的同时，提交计算出的$VIRTUAL金额。若有任何一项不足，系统将不予接受。
- 如果用户的最终分配金额小于他们承诺的 $VIRTUAL 金额，则超出的部分将被退还。

**每次Genesis Launch都遵循标准化的代币分配框架：**

- 预售：占代币总供应量的 37.5%
- 流动资金池（LP）：总供应量的 12.5%（用于 Sentient Agent）
- 开发 / 财务 / 营销 /：50%

**Genesis退款政策：**

- 承诺绝不会带有惩罚性。该协议确保每个参与者获得与其投入成比例的代币分配，所有未使用的代币将自动返还。
- 如果Genesis Launches不成功：
  - 所有承诺积分将全额退还
  - 所有已提交的 $VIRTUAL 均已完全返还
  - 如果承诺的总 $VIRTUAL低于 42,425 $VIRTUAL（实例化结合曲线和触发 Sentient 激活所需的最低限度），则启动被视为失败。
- 如果Genesis Launches成功：
  - 任何未使用的 $VIRTUAL（超出最终分配）将全额退还
  - 任何未使用的积分（超出最终分配逻辑）将全额退还

#### 1.1.3GAME Framework

> 由基础模型驱动的模块化决策引擎。它接收上下文、目标、个性以及可用工具，生成跨环境的智能自主行动。

没啥用，噱头

### 1.2主要流程

**VIRTUAL Genesis**
1. 在AgentFactory提交新申请，将VIRTUAL转移到AgentFactory
2. 在VirtualGenesisDAO上提出提案（动作：VirtualFactory.executeApplication）
3. 在VirtualGenesisDAO开始投票
4. 在VirtualGenesisDAO执行提案，将执行以下操作：
   1. 克隆AgentToken
   2. 克隆AgentDAO
   3. 铸造AgentNft
   4. 质押VIRTUAL -> $PERSONA（取决于申请中发送的符号）
   5. 使用AgentNft创建TBA


**提交贡献**
1. 在AgentDAO创建提案（动作 = ServiceNft.mint）
2. 铸造ContributionNft，通过检查发送者是否为提案的提议者进行身份验证

**升级Core**
1. 验证者在AgentDAO上对贡献提案进行投票
2. 在AgentDAO执行提案，将铸造ServiceNft，并触发以下动作：
   1. 更新成熟度评分
   2. 更新VIRTUAL核心服务ID


**分配奖励**
1. 每日，协议后端将每日利润汇总为单一金额
2. 协议后端调用AgentReward.distributeRewards，触发以下操作：
   1. 将VIRTUAL转移到AgentReward
   2. 核算并更新可领取金额，涉及：协议、质押者、验证者、数据集贡献者、模型贡献者


**领取奖励**
1. 协议调用AgentReward.withdrawProtocolRewards
2. 质押者、验证者、数据集贡献者、模型贡献者调用AgentReward.claimAllRewards

**质押VIRTUAL**
1. 调用AgentToken.stake，传入您希望委托投票权的验证者。它将接受sVIRTUAL并为您铸造$PERSONA
2. 调用AgentToken.withdraw进行提取，将销毁您的$PERSONA并将sVIRTUAL返还给您

### 1.3合约组件

| 合约               | 用途                                                         | 访问控制                                                    | 可升级 |
| ------------------ | ------------------------------------------------------------ | ----------------------------------------------------------- | ------ |
| veVirtualToken     | 不可转让的投票代币，用于在Virtual Protocol DAO和Virtual Genesis DAO上投票 | 可拥有（Ownable）                                           | 否     |
| VirtualProtocolDAO | 维护VIRTUAL生态系统的常规DAO                                 | -                                                           | 否     |
| VirtualGenesisDAO  | 用于投票决定VIRTUAL的实例化。此DAO允许提案在达到法定人数（10k票）后立即执行 | -                                                           | 否     |
| AgentFactory       | 处理新VIRTUAL的申请和实例化。存储对TBA注册表、VIRTUAL DAO/代币实现和Persona NFT vault合约的引用 | 角色：DEFAULT_ADMIN_ROLE, WITHDRAW_ROLE                     | 是     |
| AgentNft           | Persona、Core和Validator的主要注册表，用于生成ICV钱包地址    | 角色：DEFAULT_ADMIN_ROLE, VALIDATOR_ADMIN_ROLE, MINTER_ROLE | 是     |
| ContributionNft    | 每次贡献都会铸造一个新的ContributionNft。任何人都可以在VIRTUAL DAO上提出新贡献并使用提案ID铸造代币 | -                                                           | 是     |
| ServiceNft         | 接受的贡献将铸造ServiceNft，仅限VIRTUAL DAO可以铸造。用户可以查询VIRTUAL CORE的最新服务NFT | -                                                           | 是     |
| AgentToken         | VIRTUAL质押的实现合约。AgentFactory将在VIRTUAL实例化时克隆此合约。质押代币不可转让 | -                                                           | 否     |
| AgentDAO           | VIRTUAL特定DAO的实现合约。AgentFactory将在VIRTUAL实例化时克隆此合约，持有每个核心服务的成熟度评分 | -                                                           | 否     |
| AgentReward        | 奖励分配中心                                                 | 角色：GOV_ROLE, TOKEN_SAVER_ROLE                            | 是     |
| TimeLockStaking    | 允许用户质押VIRTUAL以换取sVIRTUAL                            | 角色：GOV_ROLE, TOKEN_SAVER_ROLE                            | 否     |
| Virtual            | VIRTUAL代币                                                  | 可拥有（Ownable）                                           | 否     |
| Airdrop            | 向持有者空投代币                                             | -                                                           | 否     |

## 2.代码

### 2.1lib

#### 2.1.1 ELO

这个项目写了一个ELO库来评估两个选手之间的ELO变化。

```solidity
/// @notice 计算给定结果后ELO评分的变化。
/// @param ratingA 玩家A的ELO评分
/// @param ratingB 玩家B的ELO评分
/// @param score 玩家A的得分，按100缩放。100 = 胜利，50 = 平局，0 = 失败
/// @param kFactor k因子或发展乘数，用于计算ELO评分的变化。20是典型值
/// @return change 玩家A的ELO评分变化，精确到小数点后两位。1501 = 15.01 ELO变化
/// @return negative 玩家A的ELO变化方向。玩家B的符号相反
function ratingChange(
    uint256 ratingA,
    uint256 ratingB,
    uint256 score,
    uint256 kFactor
) internal pure returns (uint256 change, bool negative) {
```

对于玩家 A 和玩家 B，玩家 A 的预期得分`E_A`定义为下面的公式。`R_A`和`R_B`分别是玩家 A 和玩家 B 的当前 ELO 评分。这个公式表示根据两名玩家的评分差异，预测玩家 A 击败玩家 B 的概率。

$$
E_A = \frac{1}{1 + 10^{(R_B - R_A)/400}}
$$

玩家 A 的 ELO 评分变化 `ΔR_A` 计算为：`S_A`是玩家 A 的实际得分：胜利得 1 分，平局得 0.5 分，失败得 0 分。K 是因子，一个常数，用于控制评分变化的幅度（例如，K=20 表示变化较平稳）。

$$
\Delta R_A = K \times (S_A - E_A)
$$

`ratingChange()` 函数的目标是根据这些原理，计算玩家 A 的 ELO 变化`ΔR_A` 的绝对值，并以放大 100 倍的形式返回（例如，返回 1501 表示 ELO 变化 15.01）。

1.输入处理与缩放：输入的 score 是以 100 为单位的，因此：`S_A` = score / 100。为了处理小数精度，函数将 kFactor 放大 10,000 倍，因此，最终返回的变化值是`ΔR_A` ×100。

```solidity
uint256 _kFactor; // scaled up `kFactor` by 100
bool _negative = ratingB < ratingA;
uint256 ratingDiff; // absolute value difference between `ratingA` and `ratingB`

unchecked {
    // scale up the inputs by a factor of 100
    // since our elo math is scaled up by 100 (to avoid low precision integer division)
    _kFactor = kFactor * 10_000;
    ratingDiff = _negative ? ratingA - ratingB : ratingB - ratingA;
}
```

2.计算评分差异：计算两名玩家的评分差ratingDiff

3.计算预期得分的指数部分：为了高效计算`E_A`预期得分的公式（`10^(ratingDeff/400的部分)`），使用了下面的数学转换：

$$
10^{x/400} = \left(10^{x/25}\right)^{1/16}
$$

$$
\_powered = 10^{n/25}
$$

$$
\text{powered} = (\_powered)^{1/16} = 10^{(n/25)/16} = 10^{n/400}
$$

这里的 800 是一个缩放基准，因为 800/400=2，对应 10^2 = 100，使得后续计算与 100 倍缩放一致。然后计算预期得分（缩放版本）：令：

$$
\text{powered} \approx 100 \times 10^{(\text{ratingDiff}/400)}
$$

预期得分的缩放版本：

$$
\text{kExpectedScore} = \frac{\_kFactor}{100 + \text{powered}}
$$

```solidity
unchecked {
    // apply offset of 800 to scale the result by 100
    n = _negative ? 800 - ratingDiff : 800 + ratingDiff;

    // (x / 400) is the same as ((x / 25) / 16))
    _powered = fp.rpow(10, n / 25, 1); // divide by 25 to avoid reach uint256 max
    powered = sixteenthRoot(_powered); // x ^ (1 / 16) is the same as 16th root of x

    // given `change = kFactor * (score - expectedScore)` we can distribute kFactor to both terms
    kExpectedScore = _kFactor / (100 + powered); // both numerator and denominator scaled up by 100
    kScore = kFactor * score; // input score is already scaled up by 100

    // determines the sign of the ELO change
    negative = kScore < kExpectedScore;
    change = negative ? kExpectedScore - kScore : kScore - kExpectedScore;
}
```

4.实际得分的缩放版本：

$$
\text{kScore} = \text{kFactor} \times \text{score} = \text{kFactor} \times (S_A \times 100)
$$

$$
\text{change} = |\text{kScore} - \text{kExpectedScore}|
$$

转化之后变成：

$$
\text{change} = |\Delta R_A| \times 100
$$

因此最终返回的change是放大了100倍的，比如1022表示10.22。

最终被调用的地方：

```solidity
// Get winner elo rating
function battleElo(uint256 currentRating, uint8[] memory battles) public view returns (uint256) {
    uint256 eloA = 1000;
    uint256 eloB = 1000;
    for (uint256 i = 0; i < battles.length; i++) {
        uint256 result = mapBattleResultToGameResult(battles[i]);
        (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k);
        change = _roundUp(change, 100);
        if (negative) {
            eloA -= change;
            eloB += change;
        } else {
            eloA += change;
            eloB -= change;
        }
    }
    return currentRating + eloA - 1000;
}
```

> 上述代码均使用[审计比赛版本](https://github.com/code-423n4/2025-04-virtuals-protocol)，非最终版本





