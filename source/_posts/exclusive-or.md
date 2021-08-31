---
title: 异或运算
date: 2021-08-29 11:20:14
tags: ['algorithm', 'xor']
---

## 运算特性

1. 两个数进行异或运算，二进制位相同为 `0`，不同为 `1`；
2. 满足交换律和结合律；
3. 异或运算可以认为是无进位相加（二进制）；
4. 任何数与 `0` 异或仍得这个数，即`0 ^ N = N`；
5. 任何数与自身异或得 `0`，即 `N ^ N = N`。

---

## 典型应用场景

### 1. 交换

```java
int a = 1;
int b = 2;

a = a ^ b; // a = 1 ^ 2, b = 2
b = a ^ b; // a = 1 ^ 2, b = 1 ^ 2 ^ 2 = 1
a = a ^ b; // a = 1 ^ 2 ^ 1 = 2, b = 1
```

数组元素交换时，要确保交换的不是一个空间，可以相等，但不能是同一块内存跟自己进行异或运算：

* Java 实现

```java
void swap(int[] array, int i, int j) {
    if (i == j) {
        // 无法确保 i != j 一定要加这个检查，否则该位置值变为 0
        return;
    }

    array[i] = array[i] ^ array[j];
    array[j] = array[i] ^ array[j];
    array[i] = array[i] ^ array[j];
}
```

### 2. 找到出现奇数次的数

**题目：** 一个数组中有一种数出现了奇数次，其他数都出现了偶数次，怎么找到这种数。

* Java 实现

```java
int getOddTimesNumber(int[] array) {
    int xor = 0;
    for (int i: array) {
        xor ^= i;
    }
    return xor;
}
```

* Rust 实现

```rust
fn get_odd_times_number_0(array: &[i32]) -> i32 {
    let mut xor = 0;
    for i in array {
        xor ^= *i;
    }
    xor
}

fn get_odd_times_number_1(array: &[i32]) -> i32 {
    array.iter().fold(0, |a, b| a ^ *b)
}

#[test]
fn test_get_odd_times_number() {
    use rand::prelude::*;

    let mut rng = thread_rng();

    for _ in 0..100 {
        let mut vec = Vec::new();
        let odd_number = rng.gen_range(0..5);
        for i in 0..5 {
            let times: usize = rng.gen_range(1..3) * 2 + if i == odd_number { 1 } else { 0 };
            for _ in 0..times {
                vec.push(i);
            }
        }
        vec.shuffle(&mut rng);

        let get0 = get_odd_times_number_0(&vec);
        let get1 = get_odd_times_number_1(&vec);
        assert_eq!(odd_number, get0);
        assert_eq!(odd_number, get1);
    }
}
```

### 3. 提取整型数最右侧的 `1`

**题目：** 提取整型数最右侧的 `1`

例如：
```
0b10101000 ----> 0b00001000
      ^---只留这个 `1`
```

* Java 实现

```java
int getRightmostOne(int value) {
    return value & (~value + 1);
}
```

* Rust 实现

```rust
fn get_rightmost_one(value: i32) -> i32 {
    value & (!value + 1)
}

#[test]
fn test_get_rightmost_one() {
    for _ in 0..1000 {
        let a: i32 = rand::random();
        let b = get_rightmost_one(a);
        assert_eq!(b >> b.trailing_zeros(), 1);
        let bits = b.leading_zeros() + 1 + b.trailing_zeros();
        assert_eq!(bits, size_of::<i32>() as u32 * 8);
    }
}
```

### 4. 找到两个出现奇数次的数

**题目：** 一个数组中有两种数出现了奇数次，其他数都出现了偶数次，怎么找到这两种数。

* Java 实现

```java
int[] getOddTimesNumbers(int[] array) {
    int xor = 0;
    for (int i: array) {
        xor ^= i;
    }

    // xor == a ^ b
    // 因为 a != b （两种数），所以 a ^ b != 0，则必然存在为 1 的二进制位
    // 不妨就使用最后一个 1，即
    int one = xor & (~xor + 1);
    // 假设这个 1 在第 N 位
    int a = 0;
    for (int i: array) {
        // 将数组分为两类：1. N 位上为 1 的；2. N 位上为 0 的。
        // a 和 b 一定分别属于这两类，不会同属一类，因为 a ^ b 的 N 位是 1
        // 这里只把第 1 类异或起来，得到一种数
        if ((i & one) != 0) {
            a ^= i;
        }
    }

    // 用之前得到的这种数异或 xor，得到另一种
    return new int[]{a, a ^ xor};
}
```

* Java 实现（函数式）

```java
int[] getOddTimesNumbers(int[] array) {
    int xor = Arrays.stream(array).reduce(0, (a, b) -> a ^ b);

    int one = xor & (~xor + 1);
    int a = Arrays.stream(array).reduce(0, (v1, v2) -> {
        if ((v2 & one) != 0) {
            return v1 ^ v2;
        } else {
            return v1;
        }
    });

    return new int[] {a, xor ^ a};
}
```

* Rust 实现

```rust
fn get_odd_times_numbers(array: &[i32]) -> (i32, i32) {
    let mut xor = 0;
    for i in array {
        xor ^= *i;
    }

    let one = xor & (!xor + 1);
    let mut a = 0;
    for i in array {
        if one & *i != 0 {
            a ^= *i;
        }
    }

    (a, xor ^ a)
}
```

* Rust 实现（函数式）

```rust
fn get_odd_times_numbers(array: &[i32]) -> (i32, i32) {
    let xor = array.iter().fold(0, |a, b| a ^ *b);

    let one = xor & (!xor + 1);
    let a = array
        .iter()
        .fold(0, |a, b| if one & b != 0 { a ^ *b } else { a });

    (a, xor ^ a)
}
```

### 5. 计算某个数为 `1` 的二进制位数

例如：

```
0b00101100 --> 3
```

* Java 实现

```java
int countOnes(int a) {
    int count  = 0;

    while (a != 0) {
        count += 1;
        int one = a & (~a + 1);
        a ^= one; // 把这个 1 给去掉
    }

    return count;
}

```

* Rust 实现

```rust
fn count_ones(mut a: i32) -> u32 {
    let mut count = 0;

    while a != 0 {
        count += 1;
        // a ^= a & (!a + 1) 在 debug 编译条件下可能溢出，无法通过测试，release 无所谓；wrapping_add 在 debug 条件下也没问题
        a ^= a & (!a).wrapping_add(1);
    }

    count
}


#[test]
fn test_count_ones() {
    for _ in 0..1000 {
        let a = rand::random();
        assert_eq!(count_ones(a), a.count_ones());
    }
}
```
