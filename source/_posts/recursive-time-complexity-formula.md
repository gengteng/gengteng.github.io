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
> 2. $\frac {N} {b})$ 为子问题规模；    
> 3. $O(N^d)$ 为每次递归完毕之后额外执行的操作的时间复杂度；  

则有：

* 如果 $\log_b a < d$，时间复杂度为 $O(N^d)$
* 如果 $\log_b a > d$，时间复杂度为 $O(N^{\log_b a})$
* 如果 $\log_b a = d$，时间复杂度为 $O(N^d \times \log N)$