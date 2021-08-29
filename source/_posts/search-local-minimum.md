---
title: 寻找局部最小
date: 2021-08-28 21:04:13
tags: ['algorithm', 'binary search']
---

给你一个数组，长度为N，**相邻元素互不相等**，请返回任意局部最小值的索引。

局部最小定义：

1. `0` 位置如果比 `1` 位置数小，则 `0` 位置为局部最小；
2. `N-1` 位置如果比 `N-2` 位置数小，则 `N-1` 位置为局部最小；
3. `i` 位置的数，如果比 `i-1` 和 `i+1` 位置的数都笑，则 `i` 位置为局部最小。

----

* Java 实现

```java
public static int getLocalMinimumIndex(int[] array) {
    if (array == null || array.length == 0) {
        return -1;
    }

    if (array.length == 1) {
        return 0;
    }

    if (array[0] < array[1]) {
        return 0;
    }

    if (array[array.length - 1] < array[array.length - 2]) {
        return array.length - 1;
    }

    int left = 1;
    int right = array.length - 2;
    while (left < right) {
        int mid = left + ((right - left) >> 1);
        if (array[mid] > array[mid - 1]) {
            right = mid - 1;
        } else if (array[mid] > array[mid + 1]) {
            left = mid + 1;
        } else {
            return mid;
        }
    }

    return left;
}
```

----

* Rust 实现

```rust
fn get_local_minimum_index(array: &[i32]) -> Option<usize> {
    let len = array.len();
    if len == 0 {
        return None;
    }
    if len == 1 {
        return Some(0);
    }

    if array[0] < array[1] {
        return Some(0);
    }

    if array[len - 1] < array[len - 2] {
        return Some(len - 1);
    }

    let mut left = 1;
    let mut right = len - 2;
    while left < right {
        let mid = mid(left, right);
        if array[mid] > array[mid - 1] {
            right = mid - 1;
        } else if array[mid] > array[mid + 1] {
            left = mid + 1;
        } else {
            return Some(mid);
        }
    }

    Some(left)
}

/// 计算两个无符号值的中间值
fn mid<T: Ord + Div<Output = T> + Add<Output = T> + Copy + Sub<Output = T> + From<u8>>(
    v1: T,
    v2: T,
) -> T {
    match v1.cmp(&v2) {
        Ordering::Less => {
            let v = v2 - v1;
            v1 + v / T::from(2u8)
        }
        Ordering::Equal => v1,
        Ordering::Greater => {
            let v = v1 - v2;
            v2 + v / T::from(2u8)
        }
    }
}

/// 判断一个索引是否为局部最小值的索引
fn is_local_minimum(array: &[i32], index: usize) -> bool {
    let len = array.len();
    if len == 0 {
        return false;
    }
    if len == 1 {
        return index == 0;
    }

    let range = 0..len;
    if !range.contains(&index) {
        return false;
    }

    // 1. `0` 位置如果比 `1` 位置数小，则 `0` 位置为局部最小；
    if index == 0 && array[0] < array[1] {
        return true;
    }

    // 2. `N-1` 位置如果比 `N-2` 位置数小，则 `N-1` 位置为局部最小；
    if index == len - 1 && array[len - 1] < array[len - 2] {
        return true;
    }

    // 3. `i` 位置的数，如果比 `i-1` 和 `i+1` 位置的数逗笑，则 `i` 位置为局部最小。
    array[index] < array[index - 1] && array[index] < array[index + 1]
}

// 判断一个数组是否符合要求
fn is_array_valid(array: &[i32]) -> bool {
    if array.is_empty() {
        return false;
    }

    let mut last = array[0];

    for v in &array[1..] {
        if last == *v {
            return false;
        }

        last = *v;
    }

    true
}

/// 测试代码
#[test]
fn test_get_local_minimum_index() {
    for _ in 0..1000 {
        // 生成随机数组
        let array = loop {
            // Cargo.toml里加上rand = "0.8"
            let array: [i32; 32] = rand::random(); 
            // 确保数组符合条件
            if !is_array_valid(&array) {
                continue;
            }
            break array;
        };

        if let Some(index) = get_local_minimum_index(&array) {
            assert!(is_local_minimum(&array, index));
        } else {
            assert!(false);
        }
    }
}
```