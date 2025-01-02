---
title: DEX 开发笔记 - Raydium 恒定乘积交换合约源码阅读
date: 2024-12-26 20:30:56
tags: ['solana', 'web3', 'dex', 'amm']
mathjax: true
---

Raydium 是一个 Solana 平台上的 DEX，提供了各种代币的流动性池，为用户提供流动性挖矿、购买代币的功能。为了给 DEX 项目引入流动性池、对接 Raydium 的交换协议，对 raydium 恒定乘积交换合约的源码进行深入学习，这也是 Raydium 上最新的标准 AMM。

> Repository: https://github.com/raydium-io/raydium-cp-swap

## 流动性池状态及创建时的基本流程

流动性池状态定义如下：

```rust
#[account(zero_copy(unsafe))]
#[repr(packed)]
#[derive(Default, Debug)]
pub struct PoolState {
    /// 使用的配置，主要包括各种费率，控制标志，拥有者 / 受益者 等信息
    pub amm_config: Pubkey,
    /// 流动性池创建者
    pub pool_creator: Pubkey,
    /// Token A 的金库，用来存储流动性 Token 及收益
    pub token_0_vault: Pubkey,
    /// Token B 的金库，用来存储流动性 Token 及收益
    pub token_1_vault: Pubkey,

    /// 流动性池 token 会在存入 A 或 B 时发放
    /// 流动性池 token 可以被提取为对应的 A 或 B
    pub lp_mint: Pubkey,
    /// A 的铸币厂
    pub token_0_mint: Pubkey,
    /// B 的铸币厂
    pub token_1_mint: Pubkey,

    /// A 使用的 Token 程序，比如旧版本或者 2022
    pub token_0_program: Pubkey,
    /// B 使用的 Token 程序
    pub token_1_program: Pubkey,

    /// TWAP 计价账户的地址
    pub observation_key: Pubkey,

    /// 用于 PDA 签名时使用的 bump
    pub auth_bump: u8,
    /// Bitwise representation of the state of the pool
    /// bit0, 1: disable deposit(vaule is 1), 0: normal
    /// bit1, 1: disable withdraw(vaule is 2), 0: normal
    /// bit2, 1: disable swap(vaule is 4), 0: normal
    pub status: u8,

    pub lp_mint_decimals: u8,
    /// mint0 and mint1 decimals
    pub mint_0_decimals: u8,
    pub mint_1_decimals: u8,

    /// True circulating supply without burns and lock ups
    pub lp_supply: u64,
    /// 金库中欠流动性提供者的 A 和 B 的数额.
    pub protocol_fees_token_0: u64,
    pub protocol_fees_token_1: u64,

    pub fund_fees_token_0: u64,
    pub fund_fees_token_1: u64,

    /// The timestamp allowed for swap in the pool.
    pub open_time: u64,
    /// recent epoch
    pub recent_epoch: u64,
    /// padding for future updates
    pub padding: [u64; 31],
}
```

初始化流动性池的指令函数签名如下：

```rust
pub fn initialize(
    ctx: Context<Initialize>, // 上下文
    init_amount_0: u64,       // A 资产初始数额
    init_amount_1: u64,       // B 资产初始数额
    mut open_time: u64,       // 开始交易时间
) -> Result<()>;
```

初始化流动性池账户定义如下：

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    /// Address paying to create the pool. Can be anyone
    #[account(mut)]
    pub creator: Signer<'info>,

    /// Which config the pool belongs to.
    pub amm_config: Box<Account<'info, AmmConfig>>,

    /// CHECK: pool vault and lp mint authority
    #[account(
        seeds = [
            crate::AUTH_SEED.as_bytes(),
        ],
        bump,
    )]
    pub authority: UncheckedAccount<'info>,

    /// CHECK: Initialize an account to store the pool state
    /// PDA account:
    /// seeds = [
    ///     POOL_SEED.as_bytes(),
    ///     amm_config.key().as_ref(),
    ///     token_0_mint.key().as_ref(),
    ///     token_1_mint.key().as_ref(),
    /// ],
    ///
    /// Or random account: must be signed by cli
    #[account(mut)]
    pub pool_state: UncheckedAccount<'info>,

    /// Token_0 mint, the key must smaller then token_1 mint.
    #[account(
        constraint = token_0_mint.key() < token_1_mint.key(),
        mint::token_program = token_0_program,
    )]
    pub token_0_mint: Box<InterfaceAccount<'info, Mint>>,

    /// Token_1 mint, the key must grater then token_0 mint.
    #[account(
        mint::token_program = token_1_program,
    )]
    pub token_1_mint: Box<InterfaceAccount<'info, Mint>>,

    /// pool lp mint
    #[account(
        init,
        seeds = [
            POOL_LP_MINT_SEED.as_bytes(),
            pool_state.key().as_ref(),
        ],
        bump,
        mint::decimals = 9,
        mint::authority = authority,
        payer = creator,
        mint::token_program = token_program,
    )]
    pub lp_mint: Box<InterfaceAccount<'info, Mint>>,

    /// payer token0 account
    #[account(
        mut,
        token::mint = token_0_mint,
        token::authority = creator,
    )]
    pub creator_token_0: Box<InterfaceAccount<'info, TokenAccount>>,

    /// creator token1 account
    #[account(
        mut,
        token::mint = token_1_mint,
        token::authority = creator,
    )]
    pub creator_token_1: Box<InterfaceAccount<'info, TokenAccount>>,

    /// creator lp token account
    #[account(
        init,
        associated_token::mint = lp_mint,
        associated_token::authority = creator,
        payer = creator,
        token::token_program = token_program,
    )]
    pub creator_lp_token: Box<InterfaceAccount<'info, TokenAccount>>,

    /// CHECK: Token_0 vault for the pool, create by contract
    #[account(
        mut,
        seeds = [
            POOL_VAULT_SEED.as_bytes(),
            pool_state.key().as_ref(),
            token_0_mint.key().as_ref()
        ],
        bump,
    )]
    pub token_0_vault: UncheckedAccount<'info>,

    /// CHECK: Token_1 vault for the pool, create by contract
    #[account(
        mut,
        seeds = [
            POOL_VAULT_SEED.as_bytes(),
            pool_state.key().as_ref(),
            token_1_mint.key().as_ref()
        ],
        bump,
    )]
    pub token_1_vault: UncheckedAccount<'info>,

    /// create pool fee account
    #[account(
        mut,
        address= crate::create_pool_fee_reveiver::id(),
    )]
    pub create_pool_fee: Box<InterfaceAccount<'info, TokenAccount>>,

    /// an account to store oracle observations
    #[account(
        init,
        seeds = [
            OBSERVATION_SEED.as_bytes(),
            pool_state.key().as_ref(),
        ],
        bump,
        payer = creator,
        space = ObservationState::LEN
    )]
    pub observation_state: AccountLoader<'info, ObservationState>,

    /// Program to create mint account and mint tokens
    pub token_program: Program<'info, Token>,
    /// Spl token program or token program 2022
    pub token_0_program: Interface<'info, TokenInterface>,
    /// Spl token program or token program 2022
    pub token_1_program: Interface<'info, TokenInterface>,
    /// Program to create an ATA for receiving position NFT
    pub associated_token_program: Program<'info, AssociatedToken>,
    /// To create a new program account
    pub system_program: Program<'info, System>,
    /// Sysvar for program account
    pub rent: Sysvar<'info, Rent>,
}
```

初始化流动性池流程如下：

1. 判断两种资产的 Mint 账户是否合法，判断 AMM 配置中是否关闭创建 Pool。
2. 设定开始交易时间。
3. 创建两种资产的金库。
4. 创建 PoolState 数据账户。
5. 创建 ObservationState 数据账户。
6. 将两种初始资产从创建者账户转账到金库账户。
7. 判断两个金库账户的数额是否合法（实际上只要大于 0 就合法）。
8. 计算流动性值: $liquidity = \sqrt(amount0 * amount1)$。
9. 固定锁定 100 个流动性值: `let lock_lp_amount = 100`
10. 发放 `liquidity - lock_lp_amount` 个 LP token 给创建者。
11. 从创建者账户里收取创建费（lamports）到一个专门存放创建费的账户中（地址硬编码在合约里）。
12. 初始化流动性池数据的各个字段。

## 资产交换

资产交换分为两类：

1. 基于输入资产: 输入资产额度固定，一部分会作为手续费，一部分作为购买资金输入。
2. 基于输出资产: 输出资产额度固定，需要额外购买一部分输出资产作为手续费，其他作为购买到的资产输出。

### 基于输入资产

指令函数签名：

```rust
pub fn swap_base_input(
    ctx: Context<Swap>,      // 上下文
    amount_in: u64,          // 输入资产
    minimum_amount_out: u64  // 最小输出资产，由调用者按照当前价格及滑点计算得出
) -> Result<()>;

#[derive(Accounts)]
pub struct Swap<'info> {
    // 交换者
    pub payer: Signer<'info>,

    // 流动性池的金库和流动性 token mint 的权限账户
    /// CHECK: pool vault and lp mint authority
    #[account(
        seeds = [
            crate::AUTH_SEED.as_bytes(),
        ],
        bump,
    )]
    pub authority: UncheckedAccount<'info>,

    /// 用于读取协议费用
    #[account(address = pool_state.load()?.amm_config)]
    pub amm_config: Box<Account<'info, AmmConfig>>,

    /// 流动性池数据账户
    #[account(mut)]
    pub pool_state: AccountLoader<'info, PoolState>,

    /// 用户购买使用的 token ATA，即输入 token 的 ATA
    #[account(mut)]
    pub input_token_account: Box<InterfaceAccount<'info, TokenAccount>>,

    /// 用户希望购买到的 token ATA，即输出 token 的 ATA
    #[account(mut)]
    pub output_token_account: Box<InterfaceAccount<'info, TokenAccount>>,

    /// 接收用户购买使用的 token 的金库
    #[account(
        mut,
        constraint = input_vault.key() == pool_state.load()?.token_0_vault || input_vault.key() == pool_state.load()?.token_1_vault
    )]
    pub input_vault: Box<InterfaceAccount<'info, TokenAccount>>,

    /// 输出用户希望购买到的 token 的金库
    #[account(
        mut,
        constraint = output_vault.key() == pool_state.load()?.token_0_vault || output_vault.key() == pool_state.load()?.token_1_vault
    )]
    pub output_vault: Box<InterfaceAccount<'info, TokenAccount>>,

    /// 输入 token 的程序（可能是 2022）
    pub input_token_program: Interface<'info, TokenInterface>,

    /// 输出 token 的程序（可能是 2022）
    pub output_token_program: Interface<'info, TokenInterface>,

    /// 输入 token 的铸币厂
    #[account(
        address = input_vault.mint
    )]
    pub input_token_mint: Box<InterfaceAccount<'info, Mint>>,

    /// 输出 token 的铸币厂
    #[account(
        address = output_vault.mint
    )]
    pub output_token_mint: Box<InterfaceAccount<'info, Mint>>,

    /// 记录价格的数据账户，用于计算 TWAP 价格 
    #[account(mut, address = pool_state.load()?.observation_key)]
    pub observation_state: AccountLoader<'info, ObservationState>,
}
```

基于输入的资产交换流程如下：

1. 检查时间、状态等是否允许交换。
2. 计算转账（应该指的是向金库中转账）费用，从总的输入费用中减去这部分，将剩余部分 `actual_amount_in` 作为实际购买资金。
3. 计算两个金库扣除协议费用和资金费用之后剩余的部分，即实际上提供流动性的两种资金。
4. 按照上一步计算得到了两种资金，计算两种资金置换另一种的价格，$A / B$ 和 $B / A$，同时转换为 `u128` 左移 32 位使用定点数来保存精度。
5. 将第 3 步得到的两种资金额度相乘，得到恒定乘积 $A * B$。
6. 计算交换结果，包括各种额度、费用。
7. 将上一步得到的计算结果中的交换后的的源资产额度减去交易费用，再乘以这个结果中交换后的目标资产额度，得到交换后的恒定乘积 $A' * B'$。
8. 验证第 6 步计算结果中交换的源资产额度是否等于 `actual_amount_in`。
9. 检查第 6 步计算结果中交换得到的目标资产额度减去转账费用之后，是否仍然大于 0 ；同时检查，是否大于等于 `minimum_amount_out`，如果不满足表示超过滑点限制。
10. 更新流动性池状态的协议费用及资金费用，用于记录金库中非流动性的部分的额度。
11. 发送一个 `SwapEvent` 事件。（为什么？怎么利用？）
12. 验证交换后的恒定乘积大于等于交换前的恒定乘积，即 $A' * B' \geq A * B$。
13. 根据计算后的结果，将相应额度的输入资产从用户账户转账到金库账户，将相应额度的输出资产从金库账户转账到用户账户。
14. 更新观测状态（`ObservationState`）的值。

### 基于输出资产

TODO

### 观测状态 / TWAP 计价器

观测值及观测状态结构定义如下：

```rust
#[zero_copy(unsafe)]
#[repr(packed)]
#[derive(Default, Debug)]
pub struct Observation {
    /// The block timestamp of the observation
    pub block_timestamp: u64,
    /// the cumulative of token0 price during the duration time, Q32.32, the remaining 64 bit for overflow
    pub cumulative_token_0_price_x32: u128,
    /// the cumulative of token1 price during the duration time, Q32.32, the remaining 64 bit for overflow
    pub cumulative_token_1_price_x32: u128,
}

#[account(zero_copy(unsafe))]
#[repr(packed)]
#[cfg_attr(feature = "client", derive(Debug))]
pub struct ObservationState {
    /// Whether the ObservationState is initialized
    pub initialized: bool,
    /// the most-recently updated index of the observations array
    pub observation_index: u16,
    pub pool_id: Pubkey,
    /// 固定长度的环形缓冲区，用于循环写入观测记录
    pub observations: [Observation; OBSERVATION_NUM], // OBSERVATION_NUM == 100
    /// padding for feature update
    pub padding: [u64; 4],
}
```

其中最重要的函数是更新：

```rust
pub fn update(
    &mut self,
    block_timestamp: u64,
    token_0_price_x32: u128,
    token_1_price_x32: u128,
);
```

1. 该函数将一个 Oracle 观测值写入到当前状态中，并将当前索引自增一模 `OBSERVATION_NUM`，即循环写入。
2. 该函数一秒钟最多执行一次。
3. 将两种资产的当前价格与跟上一个记录的时间差值相乘，即: $Price * \Delta T$。
4. 然后将上一步求出的两个结果与上一个记录的两个值累加起来 (`wrapping_add`)，作为新记录的两个值。
5. 更新索引。

### 原理

TWAP ，即时间加权平均价格(Time Weighted Average Price)；用来平滑价格波动，减少大宗交易对市场价格的冲击。

计算公式为：

$$
\text{TWAP} = \frac{\sum_{i=1}^{n} \left( P_i \times \Delta t_i \right)}{\sum_{i=1}^{n} \Delta t_i}
$$

即某个特定时间内的 **TWAP** 为：每小段时间乘以当时的价格求和，除以总时间。


### 设计理由（猜测）

我能想到的另外一种方案是：

1. 观测状态中的每个观测记录，只记录 $Price * \Delta T$ 和时间戳即可。
2. 当要求某段时间的 TWAP 时，将这段时间的所有记录累加，除以总时长即可，时间复杂度 $O(n)$。
3. 这样看起来好像可以避免 `wrapping_add`，源码中不断累加更可能遇到这种情况。

而源码在计算某段时间的 TWAP 时，只需要将最后一个记录的值和第一个记录的值的差除以总时间即可，即这种方案时间复杂度只有 $O(1)$ 。而且实际上 `wrapping_add` 得到的累加值在相减的时候仍然可以得到正确的结果，只要在这段时间内没有溢出两次就行了。