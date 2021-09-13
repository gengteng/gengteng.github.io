---
title: 归并排序
date: 2021-09-12 00:05:51
tags: ['algorithm', 'sort']
---

## 递归方法

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

## 非递归方法

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