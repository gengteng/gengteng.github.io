---
title: 归并排序及应用
date: 2021-09-12 00:05:51
tags: ['algorithm', 'sort']
mathjax: true
---

## 归并排序

* 算法时间复杂度： $O(NlogN)$
* 相比冒泡、选择等时间复杂度为 $O(N^2)$ 的排序算法，没有浪费比较行为；
* 插入排序时即便使用二分查找插入位置，也需要将插入位置后的元素依次向右移动，每次插入复杂度为 $O(N)$。

### 递归方法

* Java 实现

```java
public class MergeSort {

    public static void main(String[] args) {
        Integer[] data = {12, 3, 2, 6, 4, 8, 6, 0, 9, 3};
        System.out.println(Arrays.toString(data));
        mergeSort(data);
        System.out.println(Arrays.toString(data));
    }

    static <T extends Comparable> void mergeSort(T[] array) {
        T[] help = (T[]) new Comparable[array.length];
        doMergeSort(array, help, 0, array.length);
    }

    static <T extends Comparable> void doMergeSort(T[] array, T[] help, int start, int end) {
        if (start == end - 1) {
            return;
        }

        int mid = start + ((end - start) >> 1);
        doMergeSort(array, help, start, mid);
        doMergeSort(array, help, mid, end);
        mergeSorted(array, help, start, mid, end);
    }

    static <T extends Comparable> void mergeSorted(T[] array, T[] help, int start, int mid, int end) {
        int h = start;
        int i = start;
        int j = mid;

        while (i < mid && j < end) {
            if (array[i].compareTo(array[j]) <= 0) {
                help[h++] = array[i++];
            } else {
                help[h++] = array[j++];
            }
        }

        while (i < mid) {
            help[h++] = array[i++];
        }

        while (j < end) {
            help[h++] = array[j++];
        }

        assert h == end;
        for (int n = start; n < end; n++) {
            array[n] = help[n];
        }
    }
}
```

输出内容：

```
[12, 3, 2, 6, 4, 8, 6, 0, 9, 3]
[0, 2, 3, 3, 4, 6, 6, 8, 9, 12]
```

* Rust 实现

```rust
#[test]
fn test_merge_sort() {
    for _ in 0..100 {
        let mut array0: [u8; 32] = rand::random();
        let mut array1 = array0.clone();
        array0.merge_sort();
        array1.sort();
        assert_eq!(array0, array1);
    }
}

trait MergeSort {
    fn merge_sort(&mut self);
}

impl<T: Ord + Clone> MergeSort for [T] {
    fn merge_sort(&mut self) {
        let len = self.len();
        if len == 0 || len == 1 {
            return;
        }

        let mid = len >> 1;
        self[..mid].merge_sort();
        self[mid..].merge_sort();

        let mut i = 0;
        let mut j = mid;
        let mut vec = Vec::with_capacity(len);

        while i < mid && j < len {
            if self[i] <= self[j] {
                vec.push(self[i].clone());
                i += 1;
            } else {
                vec.push(self[j].clone());
                j += 1;
            }
        }

        while i < mid {
            vec.push(self[i].clone());
            i += 1;
        }

        while j < len {
            vec.push(self[j].clone());
            j += 1;
        }

        self.clone_from_slice(&vec);
    }
}
```

### 非递归方法

* Java 实现

```java
public static <T extends Comparable<T>> void mergeSortLoop(T[] array) {
    if (array == null || array.length < 2) {
        return;
    }

    Comparable[] help = (T[]) new Comparable[array.length];

    int mergeSize = 1;

    while (mergeSize < array.length) {
        int left = 0;

        while (left < array.length) {
            int mid = left + mergeSize;

            if (mid >= array.length) {
                break;
            }

            int right = Math.min(mid + mergeSize, array.length);
            mergeSorted(array, help, left, mid, right);
            left = right;
        }


        if (mergeSize >= array.length >> 1) {
            break;
        }

        mergeSize <<= 1;
    }
}
```

* Rust 实现

```rust
#[test]
fn test_loop_merge_sort() {
    for _ in 0..100 {
        let mut array0: [u8; 32] = rand::random();
        let mut array1 = array0.clone();
        array0.loop_merge_sort();
        array1.sort();
        assert_eq!(array0, array1);
    }
}

impl<T: Ord + Clone> MergeSort for [T] {
    fn merge_sort(&mut self) {
        //...
    }

    fn loop_merge_sort(&mut self) {
        let len = self.len();
        if len < 2 {
            return;
        }

        let mut merge_size: usize = 1;
        while merge_size < len {
            let mut left = 0;

            while left < len {
                let mid = left + merge_size;
                if mid >= len {
                    break;
                }

                let right = (mid + merge_size).min(len);

                // merge sorted
                let mut vec = Vec::with_capacity(right - left);

                let mut i = left;
                let mut j = mid;
                while i < mid && j < right {
                    if self[i] <= self[j] {
                        vec.push(self[i].clone());
                        i += 1;
                    } else {
                        vec.push(self[j].clone());
                        j += 1;
                    }
                }

                while i < mid {
                    vec.push(self[i].clone());
                    i += 1;
                }

                while j < right {
                    vec.push(self[j].clone());
                    j += 1;
                }

                (&mut self[left..right]).clone_from_slice(&vec);

                left = right;
            }

            if merge_size > len >> 1 {
                break;
            }

            merge_size <<= 1;
        }
    }
}
```

## 求数组的小和

在一个数组中，一个数左边比它小的数的总和，叫数的小和，所有数的小和加起来叫数组小和。求数组小和。

> 例子：`[1, 3, 4, 2, 5]`
> 
> `1` 左边比自己小的数：没有  
> `3` 左边比自己小的数：`1`  
> `4` 左边比自己小的数：`1` `3`  
> `2` 左边比自己小的数：`1`  
> `5` 左边比自己小的数：`1` `3` `4` `2`  
> 
> 所以，数组的小和为 $1 + 1 + 3 + 1 + 1 + 3 + 4 + 2 = 16$

* Java 实现

```java
public class SmallSum {
    public static void main(String[] args) {
        System.out.println(smallSum(new int[]{1, 3, 4, 2, 5}) + " == " + getSmallSumSimple(new int[]{1, 3, 4, 2, 5}));
    }

    static int getSmallSumSimple(int[] array) {
        int smallSum = 0;
        for (int i = 0; i < array.length; ++i) {
            for (int j = 0; j < i; ++j) {
                if (array[j] < array[i]) {
                    smallSum += array[j];
                }
            }
        }
        return smallSum;
    }

    static int smallSum(int[] array) {
        int[] help = new int[array.length];
        return doMergeSmallSum(array, help, 0, help.length);
    }

    static int doMergeSmallSum(int[] array, int[] help, int start, int end) {
        if (end - start < 2) {
            return 0;
        }

        int mid = start + ((end - start) >> 1);
        int left = doMergeSmallSum(array, help, start, mid);
        int right = doMergeSmallSum(array, help, mid, end);
        return left + right + doMerge(array, help, start, mid, end);
    }

    static int doMerge(int[] array, int[] help, int start, int mid, int end) {
        int i = start;
        int j = mid;
        int k = start;
        int sum = 0;

        while (i < mid && j < end) {
            if (array[i] < array[j]) {
                int t = array[i++];
                help[k++] = t;
                sum += t * (end - j);
            } else {
                help[k++] = array[j++];
            }
        }

        while (i < mid) {
            help[k++] = array[i++];
        }

        while (j < end) {
            help[k++] = array[j++];
        }

        for (k = start; k < end; k++) {
            array[k] = help[k];
        }

        return sum;
    }
}
```