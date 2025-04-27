---
layout: post
category: blockchain
---

## 简介

- 介绍：SushiSwap的[MasterChef](https://github.com/sushiswap/masterchef/blob/master/contracts/MasterChefV2.sol)是一个质押协议，一种流动性挖矿（Liquidity Mining）机制，也常被称为收益耕作（Yield Farming）
- V1版本铸造奖励的SuShi代币，V2的奖励来自V1版本。V2 通过质押 Dummy Token 到 V1 的特定池，间接获取 SUSHI 奖励，避免重复造币。我们在分析的时候简化其经济模型，尽量不分开V1与V2，看成整体。
- 模型：用户可以存入不同的代币（协议中是LP）到不同的池子，然后在每一个block中获取奖励（协议中是Sushi Token），管理者可以控制不同池子瓜分奖励的百分比。这是一个非常经典的模型
- 核心公式：

$$
\text{Pool Rewards} = \text{sushiPerBlock} \times \frac{\text{Pool's allocPoint}}{\text{totalAllocPoint}}
$$

$$
\text{User Rewards} = \text{User's Collateral} \times \text{accSushiPerShare} - \text{rewardDebt}
$$

## 例子

假设当前 SushiSwap 的 MasterChef 配置如下：

- `sushiPerBlock` = 100 SUSHI（每个区块产出 100 SUSHI）。
- 有 3 个质押池：
  - 池子 0（ETH/USDT）：`allocPoint` = 400，总质押量 1000 LP。
  - 池子 1（SUSHI/ETH）：`allocPoint` = 300，总质押量 500 LP。
  - 池子 2（DAI/ETH）：`allocPoint` = 200，总质押量 200 LP。
- `totalAllocPoint` = 400 + 300 + 200 = 900。

### 第一步：计算每个池子的奖励

每个区块的 SUSHI 奖励按 `allocPoint` 比例分配：

- 池子 0 奖励 = 100 × (400 / 900) ≈ 44.44 SUSHI/区块。
- 池子 1 奖励 = 100 × (300 / 900) ≈ 33.33 SUSHI/区块。
- 池子 2 奖励 = 100 × (200 / 900) ≈ 22.22 SUSHI/区块。

### 第二步：用户质押与奖励计算

假设用户 A 在池子 0（ETH/USDT）质押了 100 LP 代币，持续 10 个区块。

- 池子 0 每区块奖励 44.44 SUSHI，总质押量 1000 LP。
- 每 LP 的奖励（`accSushiPerShare` 增量）= 44.44 / 1000 = 0.04444 SUSHI/区块。
- 10 个区块后，`accSushiPerShare` 增加 = 0.04444 × 10 = 0.4444 SUSHI/LP（实际乘以 1e12 精度）。
- 用户 A 的奖励 = 100 × 0.4444 = 44.44 SUSHI。

### 第三步：用户提取

用户 A 调用 withdraw，提取 100 LP 代币，同时领取 44.44 SUSHI 奖励。其 `rewardDebt` 更新为 100 × 0.4444 = 44.44，确保不会重复领取。

### 多池子影响

如果管理员将池子 1 的` allocPoint` 调整为 500（`totalAllocPoint` 变为 1100），则：

- 池子 0 奖励变为 100 × (400 / 1100) ≈ 36.36 SUSHI/区块。
- 池子 1 奖励变为 100 × (500 / 1100) ≈ 45.45 SUSHI/区块。
- 池子 2 奖励变为 100 × (200 / 1100) ≈ 18.18 SUSHI/区块。 这说明多池子之间通过 `allocPoint` 竞争奖励，管理员可以灵活调整激励方向。

## 数学原理

- 每区块SUSHI奖励（sushiPerBlock）：
  - 每个池子都有一个份额，每个区块有奖励数额，各个池子根据自己的份额占比瓜分这个奖励数额

```solidity
function sushiPerBlock() public view returns (uint256 amount) {
    amount = uint256(MASTERCHEF_SUSHI_PER_BLOCK)
        .mul(MASTER_CHEF.poolInfo(MASTER_PID).allocPoint) / MASTER_CHEF.totalAllocPoint();
}
```

$$
\text{sushiPerBlock} = \text{sushiPerBlock} \times \frac{\text{Pool's allocPoint}}{\text{totalAllocPoint}}
$$

- 池的奖励累积（accSushiPerShare）：
  - 每个池根据其allocPoint占比获得SUSHI奖励。奖励按LP代币总量平摊，更新accSushiPerShare，表示每单位LP代币的累积奖励。
  - 这里的pid是每个池子的编号
  - 这里更新池子的行为，就是更新奖励的“累积量”，后续领取的时候，用”累积量“减去之前的领取时的”债务“，就是可领取的值。这种模式很常见。

```solidity
function updatePool(uint256 pid) public returns (PoolInfo memory pool) {
    pool = poolInfo[pid];
    if (block.number > pool.lastRewardBlock) {
        uint256 lpSupply = lpToken[pid].balanceOf(address(this));
        if (lpSupply > 0) {
            uint256 blocks = block.number.sub(pool.lastRewardBlock);
            uint256 sushiReward = blocks.mul(sushiPerBlock()).mul(pool.allocPoint) / totalAllocPoint;
            pool.accSushiPerShare = pool.accSushiPerShare.add((sushiReward.mul(ACC_SUSHI_PRECISION) / lpSupply).to128());
        }
        pool.lastRewardBlock = block.number.to64();
        poolInfo[pid] = pool;
    }
}
```

$$
\text{sushiReward} = \text{blocks} \times \text{sushiPerBlock} \times \frac{\text{Pool's allocPoint}}{\text{totalAllocPoint}}
$$

$$
\text{accSushiPerShare} += \frac{ \text{sushiReward} \times ACC\_SUSHI\_PRECISION}{ lpSupply}
$$

- 用户未领取的奖励（pendingSushi）：
  - 基于其质押的LP代币数量和池的accSushiPerShare，通过“奖励债务”机制计算。这个是一个查看函数。
  - 核心：当前累积的奖励减去之前领取时的债务，差值就是可领取的数量

```solidity
function pendingSushi(uint256 _pid, address _user) external view returns (uint256 pending) {
    PoolInfo memory pool = poolInfo[_pid];
    UserInfo storage user = userInfo[_pid][_user];
    uint256 accSushiPerShare = pool.accSushiPerShare;
    uint256 lpSupply = lpToken[_pid].balanceOf(address(this));
    if (block.number > pool.lastRewardBlock && lpSupply != 0) {
        uint256 blocks = block.number.sub(pool.lastRewardBlock);
        uint256 sushiReward = blocks.mul(sushiPerBlock()).mul(pool.allocPoint) / totalAllocPoint;
        vouaccSushiPerShare = accSushiPerShare.add(sushiReward.mul(ACC_SUSHI_PRECISION) / lpSupply);
    }
    pending = int256(user.amount.mul(accSushiPerShare) / ACC_SUSHI_PRECISION).sub(user.rewardDebt).toUInt256();
}
```

$$
\text{pending} = \left( \text{user.amount} \times \frac{accSushiPerShare}{ACC\_SUSHI\_PRECISION} \right) - \text{user.rewardDebt}
$$

每次用户存款或提取时，更新rewardDebt（债务）：

$$
\text{rewardDebt} += \text{amount} \times \frac{\text{accSushiPerShare}}{ACC\_SUSHI\_PRECISION}
$$

收获奖励时，rewardDebt设置为当前累积奖励，确保用户只领取新产生的奖励。

## 其他

管理

- add()：添加新的质押池

- set()：调整质押池的分配比例和奖励分配合约

用户

- deposit()：用户质押，函数执行之前会更新池子的状态

- withdraw()：用户取回资产，函数执行之前会更新池子的状态

- harvest()：用户收获已经产生的代币奖励，函数执行之前会更新池子的状态

























