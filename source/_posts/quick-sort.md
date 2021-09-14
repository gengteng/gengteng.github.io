---
title: 快速排序[WIP]
date: 2021-09-13 15:34:56
tags: ['algorithm', 'sort']
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
pub trait Partition<T> {
    fn partition(&mut self, value: T) -> Option<usize>;
}

impl<T: Ord> Partition<T> for [T] {
    fn partition(&mut self, value: T) -> Option<usize> {
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

        left.map(|v| v + 1)
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
pub trait NetherlandsFlagPartition<T> {
    fn netherlands_flag_partition(&mut self, value: T) -> (isize, isize);
}

impl<T: Ord> NetherlandsFlagPartition<T> for [T] {
    fn netherlands_flag_partition(&mut self, value: T) -> (isize, isize) {
        let mut left = -1isize;
        let mut right = self.len() as isize;
        let mut i = 0;
        while i < right {
            match self[i as usize].cmp(&value) {
                Ordering::Less => {
                    left += 1;
                    self.swap(i as usize, left as usize);
                    i += 1;
                }
                Ordering::Equal => {
                    i += 1;
                }
                Ordering::Greater => {
                    right -= 1;
                    self.swap(i as usize, right as usize);
                }
            }
        }

        (left, right)
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

```