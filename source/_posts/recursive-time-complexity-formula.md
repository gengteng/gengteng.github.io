---
title: 递归算法时间复杂度公式
date: 2021-09-06 16:39:50
tags: ['algorithm', 'recursive', 'formula']
mathjax: true
---

当递归函数的时间执行函数满足如下的关系式时，可以使用公式法计算时间复杂度：

$$
T(N) = a \cdot T(\frac {N} {b}) + O(N^d)
$$

> 其中：  
> 1. $a$ 为递归次数；  
> 2. $\frac {N} {b}$ 为子问题规模；    
> 3. $O(N^d)$ 为每次递归完毕之后额外执行的操作的时间复杂度；  

* 如果 $\log_b a < d$，时间复杂度为 $O(N^d)$
* 如果 $\log_b a > d$，时间复杂度为 $O(N^{\log_b a})$
* 如果 $\log_b a = d$，时间复杂度为 $O(N^d \times \log N)$

----

## 例子

求数组 arr[l..r]中的最大值，用递归方法实现。

* Java 实现

```java
public class RecursiveMax {
    public static void main(String[] args) {
        Integer[] array = {6, 3, 5, 2, 1, 4, 0, 1, 7};
        System.out.println(getMax(array, 0, array.length - 1));
    }

    public static <T extends Comparable<T>> T getMax(T[] arr, int l, int r) {
        if (l == r) {
            return arr[l];
        }

        int mid = l + ((r - l) >> 1);
        T left = getMax(arr, l, mid);
        T right = getMax(arr, mid + 1, r);
        return left.compareTo(right) >= 0 ? left : right;
    }
}
```

其中：  
1. 递归次数 $a$ 为 $2$；  
2. 子问题规模 $\frac {N} {b}$ 为 $\frac {N} {2}$，即 $b$ 为 $2$；    
3. 每次递归完毕之后额外执行的操作的时间复杂度 $O(N^d)$ 为 $O(1)$，即 $d$ 为 $0$。 
  
满足：  
$$
\log_b a = log_2 2 = 1 > d = 0
$$  
所以，该算法复杂度为 $O(N^{\log_2 2}) = O(N)$