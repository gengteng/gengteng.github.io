---
title: 关系代数除法
date: 2023-03-05 20:37:12
tags: ['database', 'relation algebra']
mathjax: true
---

# 定义

* $S$、$R$ 为关系模式，$s$、$r$ 为关系实例。
* 要计算关系除法，需要满足 $S ⊆ R$。
* 给定一个元组 $t$，令 $t[S]$ 代表元组 $t$ 在 $S$ 中属性上的投影；那么，$r ÷ s$ 是 $R-S$ 模式的一个关系。
* 元组 $t$ 在 $r ÷ s$ 中的充要条件是**满足以下两个**：
    1. $t$ 在 $Π_{R-S}(r)$ 中
    2. 对于 $s$ 中的每个元组 $t_s$，在 $r$ 中存在一个元组 $t_r$ 且满足：
        * $t_r[S] = t_s[S]$
        * $t_r[R-S] = t$
* 例子：

$r$: 

|A1|A2|
|---|---|
|a|*e*|
|a|*f*|
|b|e|
|b|g|
|c|*e*|
|c|*f*|
|d|f|

$s$:

|A2|
|---|
|*e*|
|*f*|

$r ÷ s$:

|A1|
|---|
|a|
|c|

# 公式

关系代数除法可以使用其它关系代数运算替代：

$$
r ÷ s = Π_{R-S}(r) - Π_{R-S}(Π_{R-S}(r) × s - Π_{R-S,S}(r))
$$

用上一节的例子就是：

$$
Π_{R-S}(r)
$$

|A1|
|---|
|a|
|b|
|c|
|d|

----

$$
Π_{R-S}(r) × s
$$

|A1|A2|
|---|---|
|a|e|
|a|f|
|b|e|
|b|f|
|c|e|
|c|f|
|d|e|
|d|f|

----

$$
Π_{R-S,S}(r)
$$

公式中是为了计算差（$-$）的时候两个操作数模式相同，对 $r$ 做了一下模式的颠倒，其实还是 $r$。

|A1|A2|
|---|---|
|a|e|
|a|f|
|b|e|
|b|g|
|c|e|
|c|f|
|d|f|

----

$$
Π_{R-S}(r) × s - Π_{R-S,S}(r)
$$

|A1|A2|
|---|---|
|b|f|
|d|e|

----

$$
Π_{R-S}(Π_{R-S}(r) × s - Π_{R-S,S}(r))
$$

|A1|
|---|
|b|
|d|

----

$$
Π_{R-S}(r) - Π_{R-S}(Π_{R-S}(r) × s - Π_{R-S,S}(r))
$$

|A1|
|---|
|a|
|c|