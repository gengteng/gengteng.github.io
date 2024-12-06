---
title: Solana / Anchor 学习笔记 - 账户
date: 2024-12-06 14:32:26
tags: ['solana', 'web3', 'anchor']
---

## AccountInfo

> `solana_program::account_info::AccountInfo`

账户信息（AccountInfo），合约中对于账户基本信息的引用（余额和数据可变）。

```rust
/// Account information
#[derive(Clone)]
#[repr(C)]
pub struct AccountInfo<'a> {
    /// Public key of the account
    pub key: &'a Pubkey,
    /// The lamports in the account.  Modifiable by programs.
    pub lamports: Rc<RefCell<&'a mut u64>>,
    /// The data held in this account.  Modifiable by programs.
    pub data: Rc<RefCell<&'a mut [u8]>>,
    /// Program that owns this account
    pub owner: &'a Pubkey,
    /// The epoch at which this account will next owe rent
    pub rent_epoch: Epoch,
    /// Was the transaction signed by this account's public key?
    pub is_signer: bool,
    /// Is the account writable?
    pub is_writable: bool,
    /// This account's data contains a loaded program (and is now read-only)
    pub executable: bool,
}
```

## Account

> `anchor_lang::accounts::account::Account`

账户（Account），对 `AccountInfo` 的包装，并能够自动反序列化 `T`。

```rust
#[derive(Clone)]
pub struct Account<'info, T: AccountSerialize + AccountDeserialize + Clone> {
    account: T,
    info: &'info AccountInfo<'info>,
}
```

### 基本功能

1. 检查 `T::owner()` 是否等于 `info.owner`，这就要求 `T: Owner`。通常使用 `#[account]` 宏来自动为 `T` 实现 `Owner`，并使用 `crate::ID()` 也就是合约地址作为它的 owner。
2. 此外，它还会保证 `!(Account.info.owner == SystemProgram && Account.info.lamports() == 0)`，即账户所有者不是系统程序（必须是非系统合约），并且账户的 SOL 余额不能为 0。

## AccountLoader

> `anchor_lang::accounts::account_loader::AccountLoader`

账户加载器，用于按需零拷贝反序列化的提取器，类型 `T` 需要实现 `ZeroCopy`。

```rust
#[derive(Clone)]
pub struct AccountLoader<'info, T: ZeroCopy + Owner> {
    acc_info: &'info AccountInfo<'info>,
    phantom: PhantomData<&'info T>,
}
```

### 基本功能

`load_init`: 在初始化时（只能）运行一次，得到一个 `RefMut<T>`，可读写。
`load`: 加载只读引用 `Ref<T>`。
`load_mut`: 加载读写引用 `RefMut<T>`。

## InterfaceAccount

> `anchor_lang::accounts::interface_account::InterfaceAccount`

接口账户，用来支持 `T: Owners` 的情况。即有多个程序拥有这种类型的账户数据，引入时是为了支持 token-2022 [#2386](https://github.com/coral-xyz/anchor/pull/2386)。

```rust
#[derive(Clone)]
pub struct InterfaceAccount<'info, T: AccountSerialize + AccountDeserialize + Clone> {
    account: Account<'info, T>,
    // The owner here is used to make sure that changes aren't incorrectly propagated
    // to an account with a modified owner
    owner: Pubkey,
}
```

## 基本功能

1. 检查所有者，即 `T::owners().contains(InterfaceAccount.info.owner)`。
2. 同 `Account` 的第二项。

## Interface

> `anchor_lang::accounts::interface::Interface`

接口，用来表示实现了某种接口的程序中的某一个。

```rust
#[derive(Clone)]
pub struct Interface<'info, T>(Program<'info, T>);
```

TODO: 例子。

## Program

> `anchor_lang::accounts::program::Program`

程序，表示一个程序/合约。

```rust
#[derive(Clone)]
pub struct Program<'info, T> {
    info: &'info AccountInfo<'info>,
    _phantom: PhantomData<T>,
}
```

## Signer

> `anchor_lang::accounts::signer::Signer`

签名者，检查 `Signer.info.is_signer == true`。

```rust
#[derive(Debug, Clone)]
pub struct Signer<'info> {
    info: &'info AccountInfo<'info>,
}
```

## SystemAccount

> `anchor_lang::accounts::system_account::SystemAccount`

系统账户，检查账户的拥有者是不是系统程序（即 `SystemAccount.info.owner == SystemProgram`）。

```rust
#[derive(Debug, Clone)]
pub struct SystemAccount<'info> {
    info: &'info AccountInfo<'info>,
}
```

## Sysvar

> `anchor_lang::accounts::sysvar::Sysvar`

系统变量，检查账户是否是系统变量。

```rust

pub struct Sysvar<'info, T: solana_program::sysvar::Sysvar> {
    info: &'info AccountInfo<'info>,
    account: T,
}
```

TODO: 写一篇 post 单独介绍实现了 `solana_program::sysvar::Sysvar` 的类型。

## UncheckedAccount

强调**不做任何检查**的账户，需要手动在合约指令中做检查。

```rust
#[derive(Debug, Clone)]
pub struct UncheckedAccount<'info>(&'info AccountInfo<'info>);
```