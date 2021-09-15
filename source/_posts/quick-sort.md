---
title: 快速排序[WIP]
date: 2021-09-13 15:34:56
tags: ['algorithm', 'sort']
mathjax: true
---

## 分区问题


### 二分

将一个数组分为两个区域，小于等于N在左，大于N的在右；返回右侧的起始位置。

* Java 实现

```java
public class Partition {

    public static void main(String[] args) {
        int[] array = {2, 3, 5, 1, 2, 6, 4, 3, 8, 4, 3, 5, 1, 3, 5, 8};
        System.out.println(Arrays.toString(array));
        int p = partition(array, 4);
        System.out.println(p);
        System.out.println(Arrays.toString(array));
    }

    static int partition(int[] array, int value) {
        int left = -1;

        if (array == null) {
            return left;
        }

        for (int i = 0; i < array.length; ++i) {
            if (array[i] <= value) {
                int newLeft = left + 1;
                if (i != newLeft) {
                    // swap
                    array[i] = array[i] ^ array[newLeft];
                    array[newLeft] = array[i] ^ array[newLeft];
                    array[i] = array[i] ^ array[newLeft];
                }
                left = newLeft;
            }
        }

        return left + 1;
    }
}
```

输出内容：

```
[2, 3, 5, 1, 2, 6, 4, 3, 8, 4, 3, 5, 1, 3, 5, 8]
10
[2, 3, 1, 2, 4, 3, 4, 3, 1, 3, 6, 5, 8, 5, 5, 8]
```

* Rust 实现

```rust
#[test]
fn test_partition() {
    let mut array = [4, 2, 5, 7, 5, 4, 2, 9, 3, 5, 1];
    assert_eq!(array.partition(5), Index::Position(9));
    assert_eq!(array.partition(3), Index::Position(4));
    assert_eq!(array.partition(11), Index::Greater);
    assert_eq!(array.partition(0), Index::Less);
}

#[derive(Debug, Eq, PartialEq)]
pub enum Index {
    Less,
    Position(usize),
    Greater,
}

pub trait Partition<T> {
    fn partition(&mut self, value: T) -> Index;
}

impl<T: Ord> Partition<T> for [T] {
    // 返回大于部分的起始索引
    fn partition(&mut self, value: T) -> Index {
        let mut left = None;
        for i in 0..self.len() {
            if self[i] <= value {
                if let Some(left_val) = left {
                    let new_left_val = left_val + 1;
                    self.swap(i, new_left_val);
                    left = Some(new_left_val);
                } else {
                    self.swap(i, 0);
                    left = Some(0);
                }
            }
        }

        match left {
            None => Index::Less,
            Some(i) if i == self.len() - 1 => Index::Greater,
            Some(i) => Index::Position(i + 1),
        }
    }
}
```

----

### 荷兰 🇳🇱 国旗问题（三分）

将一个数组分为三个区域，小于N在左，等于N在中间，大于N的在右；返回中间和右侧的起始位置。

* Java 实现

```java

```

* Rust 实现

```rust
use std::cmp::Ordering;

#[test]
fn test_netherlands_flag_partition() {
    let mut array = [7, 1, 2, 0, 8, 5, 3, 9, 2, 6, 5, 1, 0, 8, 7, 4];
    // sorted:      [0, 0, 1, 1, 2, 2, 3, 4, 5, 5, 6, 7, 7, 8, 8, 9]
    // index(hex):  [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, A, B, C, D, E, F]
    assert_eq!(array.netherlands_flag_partition(2), (Some(4), Some(6)));
    assert_eq!(array.netherlands_flag_partition(7), (Some(0xB), Some(0xD)));
    assert_eq!(array.netherlands_flag_partition(-1), (None, Some(0)));
    assert_eq!(array.netherlands_flag_partition(0), (None, Some(2)));
    assert_eq!(
        array.netherlands_flag_partition(10),
        (Some(array.len()), None)
    );
    assert_eq!(
        array.netherlands_flag_partition(9),
        (Some(array.len() - 1), None)
    );
}

pub trait NetherlandsFlagPartition<T> {
    fn netherlands_flag_partition(&mut self, value: T) -> (Option<usize>, Option<usize>);
}

impl<T: Ord> NetherlandsFlagPartition<T> for [T] {
    // 返回相等部分和大于部分的起始索引
    fn netherlands_flag_partition(&mut self, value: T) -> (Option<usize>, Option<usize>) {
        let len = self.len();
        let mut left = None;
        let mut right = None;
        let mut i = 0;
        while i < right.unwrap_or(len) {
            match self[i].cmp(&value) {
                Ordering::Less => {
                    match left {
                        None => {
                            self.swap(0, i);
                            left = Some(0);
                        }
                        Some(left_value) => {
                            let new_left = left_value + 1;
                            self.swap(new_left, i);
                            left = Some(new_left);
                        }
                    }
                    i += 1;
                }
                Ordering::Equal => {
                    i += 1;
                }
                Ordering::Greater => {
                    match right {
                        None => {
                            self.swap(len - 1, i);
                            right = Some(len - 1);
                        }
                        Some(right_value) => {
                            let new_right = right_value - 1;
                            self.swap(new_right, i);
                            right = Some(new_right);
                        }
                    }

                    // i 不要自增，让下一次循环检查新换到前面的值
                }
            }
        }

        (left.map(|v| v + 1), right)
    }
}
```

* Rust 实现（用枚举表示结果）

```rust
use std::cmp::Ordering;

#[test]
fn test_netherlands_flag_partition() {
    let mut array = [7, 1, 2, 0, 8, 5, 3, 9, 2, 6, 5, 1, 0, 8, 7, 4];
    assert_eq!(
        array.netherlands_flag_partition(2),
        NetherlandsFlagResult::Three(4, 6)
    );
    assert_eq!(
        array.netherlands_flag_partition(7),
        NetherlandsFlagResult::Three(11, 13)
    );
    assert_eq!(
        array.netherlands_flag_partition(-1),
        NetherlandsFlagResult::Greater
    );
    assert_eq!(
        array.netherlands_flag_partition(0),
        NetherlandsFlagResult::ValueStart(2)
    );
    assert_eq!(
        array.netherlands_flag_partition(10),
        NetherlandsFlagResult::Less
    );
    assert_eq!(
        array.netherlands_flag_partition(9),
        NetherlandsFlagResult::ValueEnd(15)
    );

    let mut array = [2, 2, 2, 2, 2];
    assert_eq!(
        array.netherlands_flag_partition(2),
        NetherlandsFlagResult::Equal
    );
}

#[derive(Debug, Eq, PartialEq)]
pub enum NetherlandsFlagResult {
    /// 分为三部分，分别是 < value, == value, > value, 返回后两部分的起始索引
    Three(usize, usize),
    /// 分为两部分，== value, > value, 返回第二部分的起始索引
    ValueStart(usize),
    /// 分为两部分，< value, == value, 返回第二部分的起始索引
    ValueEnd(usize),
    /// 所有值都小于 value
    Less,
    /// 所有值都大于 value
    Greater,
    /// 所有值都等于 value
    Equal,
}

pub trait NetherlandsFlagPartition<T> {
    fn netherlands_flag_partition(&mut self, value: T) -> NetherlandsFlagResult;
}

impl<T: Ord> NetherlandsFlagPartition<T> for [T] {
    // 返回相等部分和大于部分的起始索引
    fn netherlands_flag_partition(&mut self, value: T) -> NetherlandsFlagResult {
        let len = self.len();
        let mut left = None;
        let mut right = None;
        let mut i = 0;
        while i < right.unwrap_or(len) {
            match self[i].cmp(&value) {
                Ordering::Less => {
                    match left {
                        None => {
                            self.swap(0, i);
                            left = Some(0);
                        }
                        Some(left_value) => {
                            let new_left = left_value + 1;
                            self.swap(new_left, i);
                            left = Some(new_left);
                        }
                    }
                    i += 1;
                }
                Ordering::Equal => {
                    i += 1;
                }
                Ordering::Greater => {
                    match right {
                        None => {
                            self.swap(len - 1, i);
                            right = Some(len - 1);
                        }
                        Some(right_value) => {
                            let new_right = right_value - 1;
                            self.swap(new_right, i);
                            right = Some(new_right);
                        }
                    }

                    // i 不要自增，让下一次循环检查新换到前面的值
                }
            }
        }

        match (left.map(|v| v + 1), right) {
            (None, Some(i)) => {
                if i == 0 {
                    NetherlandsFlagResult::Greater
                } else {
                    NetherlandsFlagResult::ValueStart(i)
                }
            }
            (Some(i), None) => {
                if i >= self.len() {
                    NetherlandsFlagResult::Less
                } else {
                    NetherlandsFlagResult::ValueEnd(i)
                }
            }
            (Some(i), Some(j)) => {
                if i >= self.len() {
                    NetherlandsFlagResult::Greater
                } else {
                    NetherlandsFlagResult::Three(i, j)
                }
            }
            (None, None) => NetherlandsFlagResult::Equal,
        }
    }
}
```

## 快速排序

### v1.0

步骤：

1. 选择数组最后一个元素 `X`，在 `0..array.length-1` 的范围上进行二分，小于等于 `X` 的在左，大于 `X` 的在右；
2. 将 `X` 与右侧的第一个元素交换；
3. 将 `X` 左侧与右侧的数组分别进行上述操作，进行递归，过程中 `X` 不需要再移动。

时间复杂度：$O(N)$

* Java 实现

```java

```

* Rust 实现

```rust

```

### v2.0

1. 选择数组一个元素 `X`，进行荷兰国旗三分，小于 `X` 的在左，等于 `X` 的在中间，大于 `X` 的在右；
2. 将 `X` 左侧与右侧的数组分别进行上述操作，进行递归，过程中位于中间的所有 `X` 不需要再移动。

时间复杂度：$O(N)$

* Java 实现

```java

```

* Rust 实现

```rust
#[test]
fn test_quick_sort() {
    for _ in 0..1000 {
        let mut array0: [i32; 32] = random();
        let mut array1 = array0.clone();

        array0.sort();
        array1.quick_sort();

        assert_eq!(array0, array1);
    }
}

pub trait QuickSort {
    fn quick_sort(&mut self);
}

impl<T: Ord + Clone> QuickSort for [T] {
    fn quick_sort(&mut self) {
        if self.len() < 2 {
            return;
        }

        let value = self[0].clone();

        match self.netherlands_flag_partition(value) {
            NetherlandsFlagResult::Three(start, end) => {
                self[..start].quick_sort();
                self[end..].quick_sort();
            }
            NetherlandsFlagResult::ValueStart(start) => {
                self[start..].quick_sort();
            }
            NetherlandsFlagResult::ValueEnd(end) => {
                self[..end].quick_sort();
            }
            NetherlandsFlagResult::Less | NetherlandsFlagResult::Greater => {
                self.quick_sort();
            }
            NetherlandsFlagResult::Equal => {
                return;
            }
        }
    }
}
```

### v3.0

将 v2.0 中选择数组元素改为随机选择，其他不变。

> 1. 划分值越靠近中间，性能越好；越靠近两边性能越差；
> 2. 随机算一个数进行划分的目的就是让好情况和坏情况都变成概率事件；
> 3. 把每一种情况都列出来，会有每种情况下的复杂度，但概率都是 $1/N$；
> 4. 所有情况都考虑，则时间复杂度就是这种概率模型下的长期期望。

时间复杂度：$O(N \times logN)$  
额外空间复杂度：$O(N \times logN)$

* Java 实现

```java

```

* Rust 实现

```rust
#[test]
fn test_quick_sort() {
    for _ in 0..1000 {
        let mut array0: [i32; 32] = random();
        let mut array1 = array0.clone();

        array0.sort();
        array1.quick_sort();

        assert_eq!(array0, array1);
    }
}

pub trait QuickSort {
    fn quick_sort(&mut self);
}

impl<T: Ord + Clone> QuickSort for [T] {
    fn quick_sort(&mut self) {
        if self.len() < 2 {
            return;
        }

        let index = rand::thread_rng().gen_range(0..self.len());
        let value = self[index].clone();

        match self.netherlands_flag_partition(value) {
            NetherlandsFlagResult::Three(start, end) => {
                self[..start].quick_sort();
                self[end..].quick_sort();
            }
            NetherlandsFlagResult::ValueStart(start) => {
                self[start..].quick_sort();
            }
            NetherlandsFlagResult::ValueEnd(end) => {
                self[..end].quick_sort();
            }
            NetherlandsFlagResult::Less | NetherlandsFlagResult::Greater => {
                self.quick_sort();
            }
            NetherlandsFlagResult::Equal => {
                return;
            }
        }
    }
}
```