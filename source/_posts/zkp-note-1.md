---
title: 零知识证明笔记-1-密码学基础
date: 2024-05-25 15:11:36
tags: ['Zero-Knowledge Proof', 'Web3']
mathjax: true
---

这是结诚浩的《图解密码技术》一书的读书笔记，我是在去北京的火车上把其中感兴趣的部分草草过完的。[World of Z2O-K7E](https://learn.z2o-k7e.world) 只是建议过一下，我就只是过一下，感觉还是很有帮助的，至少看过了之后，我能感觉自己的大脑切换到了“密码学的上下文”，接收零知识证明相关的知识似乎更高效了。

## 基本常识

1. 不要使用保密的密码算法（即隐蔽式的安全性），应该使用得到广为验证的公开的密码算法，而保护好密钥。
2. 使用低强度的密码会给人一种错误的安全感，比不进行加密更危险。
3. 任何密码总有一天会被破解，除了不实用的一次性密码本和传说中的量子密码。
4. 密码只是信息安全的一部分，提防“社会工程学”。

## 对称密码

### DES / 3DES

一种将 64 位明文加密为 64 位密文的对称密码算法，需要以**分组**为单位对输入进行处理（这样的密码算法称为分组密码，block cipher）。DES 已经可以在现实时间内被破解。

三重DES（3DES）是为了增加 DES 的强度设计的。三重 DES 并不是三轮加密，而是加密、解密、加密。如果三轮的密钥都相同，就跟 DES 一样了。

第一轮和第三轮使用同样的密钥则称为 DES-EDE2。
三轮都不同则称为 DES-EDE3。

### AES

Rijndael 算法在 2000 年被选为 AES。

分组长度固定为 128 比特，密钥长度有 128、192、256 比特三种。

加密过程：SubBytes -> ShiftRows -> MixColumns -> AddRoundKey

解密过程: AddRoundKey -> InvMixColumns -> InvShiftRows -> InvSubBytes

### 如何选择

不用 DES，少用 3DES，尽量用 AES。

## 分组密码的模式

* **ECB**（Electronic CodeBook，电子密码本）：简单，快速，支持并行计算；明文中重复排列会反映在密文中，并且通过删除、替换密文分组可以对明文进行操作，对包含某些比特错误的密文解密时对应的分组会出错，不能抵御重放攻击。不要用。
* **CBC**（Cipher Book Chaining，密文分组连接模式）：推荐使用，重复的明文不会导致重复的密文，可以解密任意密文分组。**必须是先加密再 XOR**，如果反过来，则其效果等同于 **ECB**，可以直接从 IV （初始化向量）开始，依次将前后两组密文执行 XOR，得到的每组结果恰好就等于 ECB 每组的结果。
* **CFB** / **OFB**：不需要填充（padding）。
* **CTR**（CounTeR，计数器模式）：推荐使用，主动攻击者反转密文分组中的某些比特时，明文分组中相对应的比特也会被反转。

## 公钥密码

RSA 的基本步骤：

1. 取两个比较大的质数 p 和 q。
2. 求两者的乘积 $N = p \times q$。
3. 求 
$$
L = lcm(p - 1, q - 1)
$$
，即 p 和 q 的最小公倍数；有的资料说的是求欧拉函数值 $\phi = (p-1) \times (q-1)$，以下统称 $\phi$。

> GPT: 两者都有各自的使用场景，但在实际使用中，求欧拉函数值 ($\phi$) 更为常见。

4. 选择公钥 $E$，$E$ 与 $\phi$ 互质并且 $1 < E  < \phi$。
5. 求私钥 $D$，$D \times E \equiv 1$ (mod $\phi$)，即 D 为 E 的乘法逆元。
6. 这样得到公钥为 $(E, N)$，私钥为 $(D, N)$。

一个使用小质数实现的简单的 RSA：

```rust
use num::BigInt;
use rand::seq::index::sample;
use rand::{thread_rng, Rng};

const SMALL_PRIMES: [u64; 50] = [
    2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47, 53, 59, 61, 67, 71, 73, 79, 83, 89, 97,
    101, 103, 107, 109, 113, 127, 131, 137, 139, 149, 151, 157, 163, 167, 173, 179, 181, 191, 193,
    197, 199, 211, 223, 227, 229,
];

#[derive(Debug)]
struct KeyPair {
    private_key: u64,
    public_key: u64,
    modulus: u64,
}

impl KeyPair {
    pub fn generate() -> Self {
        let mut rng = thread_rng();
        let indices = sample(&mut rng, SMALL_PRIMES.len(), 2);
        // use crate rand to select two random prime numbers
        let p = SMALL_PRIMES[indices.index(0)];
        let q = SMALL_PRIMES[indices.index(1)];
        let modulus = p * q;
        let phi = (p - 1) * (q - 1);
        let public_key = loop {
            let public_key = SMALL_PRIMES[thread_rng().gen_range(0..SMALL_PRIMES.len())];
            if public_key < phi && gcd(public_key, phi) == 1 {
                break public_key;
            }
        };

        let mut private_key = 1;
        while (public_key * private_key) % phi != 1 {
            private_key += 1;
        }

        Self {
            private_key,
            public_key,
            modulus,
        }
    }

    pub fn encrypt(&self, message: BigInt) -> BigInt {
        message.pow(self.public_key as u32) % self.modulus
    }

    pub fn decrypt(&self, encrypted_message: BigInt) -> BigInt {
        encrypted_message.pow(self.private_key as u32) % self.modulus
    }

    // getters

    pub fn get_private_key(&self) -> (u64, u64) {
        (self.private_key, self.modulus)
    }

    pub fn get_public_key(&self) -> (u64, u64) {
        (self.public_key, self.modulus)
    }

    pub fn get_modulus(&self) -> u64 {
        self.modulus
    }
}

fn gcd(p0: u64, p1: u64) -> u64 {
    let mut p0 = p0;
    let mut p1 = p1;
    while p1 != 0 {
        let temp = p1;
        p1 = p0 % p1;
        p0 = temp;
    }
    p0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_rsa() {
        for _ in 0..100 {
            let key_pair = KeyPair::generate();
            let message = BigInt::from(rand::thread_rng().gen_range(6..key_pair.get_modulus()));
            let encrypted_message = key_pair.encrypt(message.clone());
            let decrypted_message = key_pair.decrypt(encrypted_message.clone());
            assert_eq!(decrypted_message, message);
            println!(
                "public key: ({}, {}), private key: ({}, {}), message: {}, encrypted: {}, decrypted: {}",
                key_pair.get_public_key().0, key_pair.get_public_key().1,
                key_pair.get_private_key().0, key_pair.get_private_key().1,
                message,
                encrypted_message,
                decrypted_message
            )
        }
    }
}
```

## 单向散列函数

单向散列函数（也称为消息摘要函数、哈希函数、杂凑函数），用来保证消息的完整性（integrity），也称为一致性。

输入的消息也称为原像（pre-image）。

输出的散列值也称为消息摘要（message digest）或者指纹（fingerprint）。

难以发现碰撞的性质称为抗碰撞性（collision resistance）。

### 算法选择

MD5 是不安全的。

SHA-1 不应该用于新用途。

SHA-2 （SHA-256以上） 、SHA-3 是安全的，可以使用。

## 消息认证码

消息认证码（Message Authentication Code）是一种确认完整性并进行认证的技术。

### 实现方法

* 单向散列函数，例如 HMAC。
* 分组密码，例如 AES-CMAC。
* 其他，例如流密码和公钥密码。

## 数字签名

使用公钥密码（比如 RSA、椭圆曲线）来实现。

* 绝对不要对意思不清楚的消息进行签名，可能无意中解密了公钥加密的密文。
* 依赖公钥基础设施，公钥必须属于真正的发送者，即使用证书来确认。

## 密钥交换

### Diffie-Hellman

一个使用小质数实现的简单的 Diffie-Hellman：

```rust
use num::BigInt;
use rand::thread_rng;
use rand::Rng;

const P_AND_G: [(u64, u64); 20] = [
    (7, 3),
    (11, 2),
    (13, 2),
    (17, 3),
    (19, 2),
    (23, 5),
    (29, 2),
    (31, 3),
    (37, 2),
    (41, 6),
    (43, 3),
    (47, 5),
    (53, 2),
    (59, 2),
    (61, 2),
    (67, 2),
    (71, 7),
    (73, 5),
    (79, 3),
    (83, 2),
];

// prime P and generator G
#[derive(Debug)]
pub struct PnG {
    prime: u64,
    generator: u64,
}

impl PnG {
    pub fn generate() -> Self {
        let (prime, generator) = P_AND_G[thread_rng().gen_range(0..P_AND_G.len())];
        Self { prime, generator }
    }

    pub fn get_prime(&self) -> u64 {
        self.prime
    }

    pub fn get_generator(&self) -> u64 {
        self.generator
    }

    pub fn generate_random_number(&self) -> u64 {
        // generate a number in the range of 1 to prime - 2
        thread_rng().gen_range(1..self.prime - 2)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_diffie_hellman() {
        for round in 0..10 {
            let pg = PnG::generate();
            println!(
                "{round}) Prime: {}, Generator: {}",
                pg.get_prime(),
                pg.get_generator()
            );
            let alice_number = pg.generate_random_number();
            let bob_number = pg.generate_random_number();
            println!(
                "{round}) Alice number: {}, Bob number: {}",
                alice_number, bob_number
            );
            let alice_partial_key =
                BigInt::from(pg.get_generator()).pow(alice_number as u32) % pg.get_prime();
            let bob_partial_key =
                BigInt::from(pg.get_generator()).pow(bob_number as u32) % pg.get_prime();
            println!(
                "{round}) Alice partial key: {}, Bob partial key: {}",
                alice_partial_key, bob_partial_key
            );
            let alice_full_key =
                BigInt::from(bob_partial_key).pow(alice_number as u32) % pg.get_prime();
            let bob_full_key =
                BigInt::from(alice_partial_key).pow(bob_number as u32) % pg.get_prime();
            println!(
                "{round}) Alice full key: {}, Bob full key: {}",
                alice_full_key, bob_full_key
            );
            assert_eq!(alice_full_key, bob_full_key);
        }
    }
}
```

## 椭圆曲线密码

椭圆曲线密码是利用“椭圆曲线上的离散对数问题”的复杂度来实现的。

$$
y^2 = x^3 + ax + b
$$

使用了椭圆曲线的 schnorr 的一个简单实现：

```rust
use k256::{NonZeroScalar, ProjectivePoint, PublicKey};
use rand::rngs::OsRng;

fn main() {
    // 使用操作系统级别的 RNG
    let mut rng = OsRng;

    // Alice 拥有一个私钥 a，计算公钥 aG
    // 私钥 a
    let a = NonZeroScalar::random(&mut rng);
    // 公钥 aG
    let a_g = ProjectivePoint::from(PublicKey::from_secret_scalar(&a));

    // 知道 G 和 aG，求 a，属于椭圆曲线上的离散对数问题，目前没有有效的解法
    let r = NonZeroScalar::random(&mut rng);
    // Alice 生成一个随机数 r，并计算 R = rG
    let r_g = ProjectivePoint::from(PublicKey::from_secret_scalar(&r));

    // Bob 提供一个挑战 c
    let c = NonZeroScalar::random(&mut rng);

    // Alice 计算响应 z = r + a * c (mod q)
    let ac = a * c; // 计算 a * c
    let z = r.add(&ac);

    let Some(z) = NonZeroScalar::new(z).into() else {
        eprintln!("计算 z 为零。");
        return;
    };

    // Alice 将 z 发送给 Bob，Bob 验证 zG ?= R + c*PK
    let z_g = ProjectivePoint::from(PublicKey::from_secret_scalar(&z));

    // 计算 c  * PK
    let c_pk = a_g * *c;

    // 计算 rG + c*PK
    let r_plus_cpk = r_g + c_pk;

    // 验证 zG ?= rG + c*PK
    // c * PK 与 a * c 的关系是？
    // 为什么这个证明是成立的
    // 为什么这个证明是不可伪造的
    // 回答:
    // c * PK = c * a * G = a * c * G = a * c * G = ac * G
    // PK = aG ?
    if z_g == r_plus_cpk {
        println!("验证成功，Alice 证明了她拥有私钥.");
    } else {
        println!("验证失败.");
    }
}

// 与 GPT 的交谈：
// 为什么 Alice 要先生成一个随机数？
// 答：为了证明她拥有私钥，她需要一个随机数 r，然后计算 R = rG，然后计算 z = r + a * c (mod q)
// 可以让 Bob 先生成随机数并发送给 Alice 吗？（然后 Alice 再生成 r ，发送 rG）
// 答：可以，但是这样会暴露 Bob 的私钥，因为 Bob 会发送 c * PK，PK = aG，所以 Bob 的私钥就会暴露
// 不对啊，Bob 只需要发送 c，不会暴露私钥啊
// 答：对，Bob 只需要发送 c，不会暴露私钥，但是这样会暴露 Bob 的公钥，Bob 的公钥是 aG，这样会暴露 Bob 的身份
// 不对啊，aG 是 Alice 使用私钥 a 计算出来的公钥，怎么会是 Bob 的公钥呢？
// 答：对，我理解错了，Bob 只需要发送 c，不会暴露私钥，也不会暴露公钥，这样是可以的
// Bob 先发送随机数给 Alice 真的没问题吗？Alice 不会获得没有 a 但是证明她拥有 a 的超能力吗？
// 答：Alice 不会获得没有 a 但是证明她拥有 a 的超能力，因为 Alice 需要 a 来计算 z = r + a * c (mod q)，如果没有 a，Alice 就无法计算 z
// Alice 不能用假的 a' 来计算 z 吗？一直用假的 a', 是否也能通过验证？
// 答：不能，因为 Bob 会用 a' 来计算 c * PK，然后验证 zG ?= R + c * PK，如果 a' 是错误的，那么 c * PK 也是错误的，验证就会失败
// PK 和 zG 都是用 a' 计算出来的，所以 Bob 永远都会验证成功，不管 Alice 用了什么 a'，验证都会成功，这样 Alice 是否就获得的超能力
// 答：不会，因为 Alice 无法计算出正确的 z，验证会失败
// z' = r + a' * c (mod q)，所以 z'G = R + c * a' * G 验证肯定成功吧？
// 答：不会，因为 c * a' * G != c * a * G，验证会失败
// 你好像没有理解，如果 Bob 实现发送了挑战 c，Alice 可以使用任何除了 a 之外的密钥替换 a（比如 a')，这样整个流程只是证明了 Alice 拥有一个密钥（这个密钥并不一定是 a）
// 答：对，我理解错了，如果 Bob 先发送了挑战 c，Alice 可以使用任何除了 a 之外的密钥替换 a（比如 a'），这样整个流程只是证明了 Alice 拥有一个密钥（这个密钥并不一定是 a）
// 所以 Alice 即使没有任何密钥，她临时编造一个，在这种情况下，Bob 永远都无法验证 Alice 是否拥有私钥，这样 Alice 就获得了超能力
// 所以顺序不能调换，对吗？
// 答：对，顺序不能调换，Alice 必须先生成一个随机数 r，然后计算 R = rG，然后计算 z = r + a * c (mod q)，然后发送 z 给 Bob，Bob 验证 zG ?= R + c * PK
```