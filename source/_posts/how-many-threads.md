---
title: 工作线程数设多少合适
date: 2021-08-24 21:21:47
tags: ['多线程', '高并发', 'formula']
mathjax: true
---

遵循以下公式：

$$
N_t = N_C * U_C~ * (1 + W/C)
$$

* 其中，$N_C$ 为处理器的核的数目，可以通过 `Runtime.getRuntime().avaliableProcessors()` 得到
* $U_C$ 是期望的 CPU 利用率（介于 0 到 1 之间）
* $W / C$ 是等待时间与计算时间的比值

例如，对于 CPU 密集型应用，期望 CPU 利用率为 `100%`，无等待纯计算，则有:

$$
N_t = N_C * 1 * (1 + 0/1) = N_C
$$

即工作线程数设置为处理器的核心数最合适。