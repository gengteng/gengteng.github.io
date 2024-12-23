---
title: Solana / Anchor 学习笔记 - PDA 派生
date: 2024-12-23 08:46:41
tags: ['solana', 'web3', 'PDA']
---

## `Pubkey`

```rust
pub struct Pubkey(pub(crate) [u8; 32]);
```

## `find_program_address`

```rust
pub fn find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> (Pubkey, u8) {
    Self::try_find_program_address(seeds, program_id)
        .unwrap_or_else(|| panic!("Unable to find a viable program address bump seed"))
}
```

## `try_find_program_address`

```rust
pub fn try_find_program_address(seeds: &[&[u8]], program_id: &Pubkey) -> Option<(Pubkey, u8)> {
    // Perform the calculation inline, calling this from within a program is
    // not supported
    #[cfg(not(target_os = "solana"))]
    {
        let mut bump_seed = [std::u8::MAX]; // 从 std::u8::MAX 开始尝试
        for _ in 0..std::u8::MAX {
            {
                let mut seeds_with_bump = seeds.to_vec();
                seeds_with_bump.push(&bump_seed); // 将 bump 作为种子的最后一部分
                match Self::create_program_address(&seeds_with_bump, program_id) {
                    Ok(address) => return Some((address, bump_seed[0])),
                    Err(PubkeyError::InvalidSeeds) => (),
                    _ => break,
                }
            }
            bump_seed[0] -= 1; // 尝试失败了就减 1 再尝试
        }
        None
    }
    // Call via a system call to perform the calculation
    #[cfg(target_os = "solana")]
    {
        // ...
    }
}
```

## `create_program_address`

```rust
pub fn create_program_address(
    seeds: &[&[u8]],
    program_id: &Pubkey,
) -> Result<Pubkey, PubkeyError> {
    if seeds.len() > MAX_SEEDS { // 种子最多有 16 个
        return Err(PubkeyError::MaxSeedLengthExceeded);
    }
    for seed in seeds.iter() {
        if seed.len() > MAX_SEED_LEN { // 每个种子最长 32
            return Err(PubkeyError::MaxSeedLengthExceeded);
        }
    }

    // Perform the calculation inline, calling this from within a program is
    // not supported
    #[cfg(not(target_os = "solana"))]
    {
        let mut hasher = crate::hash::Hasher::default(); // 就是一个 Sha256
        for seed in seeds.iter() {
            hasher.hash(seed);
        }
        // PDA_MARKER 是一个 21 字节长的字符串
        // const PDA_MARKER: &[u8; 21] = b"ProgramDerivedAddress";
        hasher.hashv(&[program_id.as_ref(), PDA_MARKER]);
        let hash = hasher.result();

        if bytes_are_curve_point(hash) { // PDA 账户需要确保不在爱德华曲线上
            return Err(PubkeyError::InvalidSeeds);
        }

        Ok(Pubkey::from(hash.to_bytes()))
    }
    // Call via a system call to perform the calculation
    #[cfg(target_os = "solana")]
    {
        let mut bytes = [0; 32];
        let result = unsafe {
            crate::syscalls::sol_create_program_address(
                seeds as *const _ as *const u8,
                seeds.len() as u64,
                program_id as *const _ as *const u8,
                &mut bytes as *mut _ as *mut u8,
            )
        };
        match result {
            crate::entrypoint::SUCCESS => Ok(Pubkey::from(bytes)),
            _ => Err(result.into()),
        }
    }
}
```

## `bytes_are_curve_point`

```rust
pub fn bytes_are_curve_point<T: AsRef<[u8]>>(_bytes: T) -> bool {
    #[cfg(not(target_os = "solana"))]
    {
        curve25519_dalek::edwards::CompressedEdwardsY::from_slice(_bytes.as_ref())
            .decompress()
            .is_some()
    }
    #[cfg(target_os = "solana")]
    unimplemented!();
}
```

## 总结

### 整体流程

1. 从 `[std::u8::MAX]` (即 `[255]`) 开始作为 bump 尝试，值依次递减至 0，bump 就是最后一个种子。
2. 检查种子数是否不超过 16 (包含 bump，也就是用户能输入的只有 15 个)，所有种子长度不超过 32.
3. 使用所有种子依次输入 Sha256 进行哈希，然后将 `program_id` 及一个 21 字节的常量 `PDA_MARKER` (`b"ProgramDerivedAddress"`) 输入进行哈希。
4. 计算哈希结果并判断是否在 Curve25519 上，如果不在，表示是合法的 PDA 地址，否则 bump 自减 1 从第二步开始重试。
5. 如果从 `[255]` 到 `[0]` 都找不到，则返回 None / 报错。

> Curve25519 是一种特定的爱德华曲线，它设计用于实现高效、安全的 Diffie-Hellman 密钥交换。