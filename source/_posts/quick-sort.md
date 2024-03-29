---
title: 快速排序
date: 2021-09-13 15:34:56
tags: ['algorithm', 'sort']
mathjax: true
---

## 分区问题

### 二分

将一个数组分为两个区域，小于等于N在左，大于N的在右；返回右侧的起始位置。

* Java 实现

```java
static int partition(int[] array, int value, int start, int end) {
    if (array == null || start >= end) {
        return -1;
    }

    int left = -1;

    for (int i = start; i < end; i++) {
        if (array[i] <= value) {
            if (left == -1) {
                left = start;
            } else {
                left += 1;
            }
            swap(array, i, left);
        }
    }

    return left + 1;
}

static int partition(int[] array, int value) {
    return partition(array, value, 0, array.length);
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
    // 值比所有数组元素都小
    Less,
    // 能找到一个分区位置
    Position(usize),
    // 值比所有数组元素都大
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
static void swap(int[] array, int i, int j) {
    if (i == j) {
        return;
    }

    array[i] = array[i] ^ array[j];
    array[j] = array[i] ^ array[j];
    array[i] = array[i] ^ array[j];
}

static int[] netherlandsFlagPartition(int[] array, int value) {
    return netherlandsFlagPartition(array, value, 0, array.length);
}

static int[] netherlandsFlagPartition(int[] array, int value, int start, int end) {
    if (array == null || start >= end) {
        return null;
    }

    int left = -1;
    int right = end;
    int i = start;

    while (i < right) {
        if (array[i] < value) {
            left = left == -1 ? start : (left + 1);
            swap(array, left, i);
            i += 1;
        } else if (array[i] > value) {
            right -= 1;
            swap(array, right, i);
        } else { // array[i] == value
            i += 1;
        }
    }

    return new int[]{left + 1, right};
}
```

输出内容：

```
[10, 13]
[2, 3, 1, 2, 4, 3, 4, 3, 1, 3, 5, 5, 5, 8, 6, 8]
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
static void quickSort1(int[] array) {
    recurse1(array, 0, array.length);
}

static void recurse1(int[] array, int start, int end) {
    if (array == null || start >= end - 1) {
        return;
    }

    int pivot = array[end - 1];
    int idx = partition(array, pivot, start, end - 1);
    swap(array, idx, end - 1);
    recurse1(array, start, idx);
    recurse1(array, idx + 1, end);
}
```

* Rust 实现

```rust
#[test]
fn test_quick_sort1() {
    for _ in 0..1000 {
        let mut array0: [u8; 32] = rand::random();
        let mut array1 = array0.clone();

        array0.sort();
        array1.quick_sort1();

        assert_eq!(array0, array1);
    }
}

pub trait QuickSort1 {
    fn quick_sort1(&mut self);
}

impl<T: Ord + Clone> QuickSort1 for [T] {
    fn quick_sort1(&mut self) {
        let len = self.len();
        if len < 2 {
            return;
        }

        let value = self[len - 1].clone();

        match self[..len - 1].partition(value) {
            // value 比所有元素都小，把 value 挪到第一个位置，排序剩下的
            Index::Less => {
                self.swap(0, len - 1);
                self[1..].quick_sort1();
            }
            // 把 value 与第一个大于区的数交换
            Index::Position(i) => {
                self.swap(i, len - 1);
                self[..i].quick_sort1();
                self[i + 1..].quick_sort1();
            }
            // value比所有元素都大，不动，排序前面所有的
            Index::Greater => {
                self[..len - 1].quick_sort1();
            }
        }
    }
}
```

### v2.0

1. 选择数组一个元素 `X`，进行荷兰国旗三分，小于 `X` 的在左，等于 `X` 的在中间，大于 `X` 的在右；
2. 将 `X` 左侧与右侧的数组分别进行上述操作，进行递归，过程中位于中间的所有 `X` 不需要再移动。

时间复杂度：$O(N)$

* Java 实现

```java
static void quickSort2(int[] array) {
    recurse2(array, 0, array.length);
}

static void recurse2(int[] array, int start, int end) {
    if (array == null || start >= end - 1) {
        return;
    }

    int pivot = array[end - 1];
    int[] idxs = netherlandsFlagPartition(array, pivot, start, end);
    if (idxs == null) {
        return;
    }
    recurse2(array, start, idxs[0]);

    if (idxs[1] < end - 1) {
        recurse2(array, idxs[1], end);
    }
}
```

* Rust 实现

```rust
#[test]
fn test_quick_sort2() {
    for _ in 0..1000 {
        let mut array0: [i32; 32] = random();
        let mut array1 = array0.clone();

        array0.sort();
        array1.quick_sort2();

        assert_eq!(array0, array1);
    }
}

pub trait QuickSort2 {
    fn quick_sort2(&mut self);
}

impl<T: Ord + Clone> QuickSort2 for [T] {
    fn quick_sort2(&mut self) {
        if self.len() < 2 {
            return;
        }

        let value = self[0].clone();

        match self.netherlands_flag_partition(value) {
            NetherlandsFlagResult::Three(start, end) => {
                self[..start].quick_sort2();
                self[end..].quick_sort2();
            }
            NetherlandsFlagResult::ValueStart(start) => {
                self[start..].quick_sort2();
            }
            NetherlandsFlagResult::ValueEnd(end) => {
                self[..end].quick_sort2();
            }
            NetherlandsFlagResult::Less | NetherlandsFlagResult::Greater => {
                self.quick_sort2();
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
static void quickSort3(int[] array) {
    recurse3(array, 0, array.length);
}

static void recurse3(int[] array, int start, int end) {
    if (array == null || start >= end - 1) {
        return;
    }

    int pi = (int) (Math.random() * (end - start));
    System.out.println(start + pi);
    int pivot = array[start + pi];
    int[] idxs = netherlandsFlagPartition(array, pivot, start, end);
    if (idxs == null) {
        return;
    }
    recurse3(array, start, idxs[0]);

    if (idxs[1] < end - 1) {
        recurse3(array, idxs[1], end);
    }
}
```

* Rust 实现

```rust
#[test]
fn test_quick_sort3() {
    for _ in 0..1000 {
        let mut array0: [i32; 32] = random();
        let mut array1 = array0.clone();

        array0.sort();
        array1.quick_sort3();

        assert_eq!(array0, array1);
    }
}

pub trait QuickSort3 {
    fn quick_sort3(&mut self);
}

impl<T: Ord + Clone> QuickSort for [T] {
    fn quick_sort3(&mut self) {
        if self.len() < 2 {
            return;
        }

        let index = rand::thread_rng().gen_range(0..self.len());
        let value = self[index].clone();

        match self.netherlands_flag_partition(value) {
            NetherlandsFlagResult::Three(start, end) => {
                self[..start].quick_sort3();
                self[end..].quick_sort3();
            }
            NetherlandsFlagResult::ValueStart(start) => {
                self[start..].quick_sort3();
            }
            NetherlandsFlagResult::ValueEnd(end) => {
                self[..end].quick_sort3();
            }
            NetherlandsFlagResult::Less | NetherlandsFlagResult::Greater => {
                self.quick_sort3();
            }
            NetherlandsFlagResult::Equal => {
                return;
            }
        }
    }
}
```

### Swap 函数 

* Java 实现

```java
static void swap(int[] array, int i, int j) {
    if (i == j) {
        return;
    }

    array[i] = array[i] ^ array[j];
    array[j] = array[i] ^ array[j];
    array[i] = array[i] ^ array[j];
}
```