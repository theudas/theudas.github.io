---
author: Ruoying Tan
date: '2026-02-25T21:44:13+08:00'
draft: false
math: true
title: '不等式证明'
tags: ['数学', '高考', '证明题']
---

## 题目描述

**已知：** $a > 0, b > 0$ 且 $a + b = 2$。

**求证：** $\sqrt{a+1} + \sqrt{b+1} \le 2\sqrt{2}$。


## 证明过程
### 方法一：利用最基础的代数运算性质
要证：$\sqrt{a+1}+\sqrt{b+1} \le 2\sqrt{2}$

即证：$(\sqrt{a+1}+\sqrt{b+1})^2 \le (2\sqrt{2})^2$

化简可得：$a+1+2\sqrt{(a+1)(b+1)}+b+1 \le 8$

整理可得：$a+b+2+2\sqrt{ab+a+b+1} \le 8$

代入$a+b=2$可得：$4+2\sqrt{ab+3} \le 8$

即$\sqrt{ab+3} \le 2$

两边平方可得：$ab \le 1$

代入$a+b=2$可得：$a(2-a) \le 1$

由于$a>0$且$b=2-a>0$，故$0<a<2$

由二次函数单调性可知当$a=1$时$a(2-a)$最大值为1

即$a(2-a) \le 1$成立，原不等式得证

### 方法二：利用柯西不等式 (Cauchy-Schwarz Inequality)

根据柯西不等式：$(x_1y_1 + x_2y_2)^2 \le (x_1^2 + x_2^2)(y_1^2 + y_2^2)$

令 $x_1 = \sqrt{a+1}, x_2 = \sqrt{b+1}$, $y_1 = 1, y_2 = 1$。

代入公式可得：$(\sqrt{a+1} \cdot 1 + \sqrt{b+1} \cdot 1)^2 \le [(\sqrt{a+1})^2 + (\sqrt{b+1})^2] \cdot (1^2 + 1^2)$


化简不等式可得：$(\sqrt{a+1} + \sqrt{b+1})^2 \le (a + 1 + b + 1) \cdot 2$


代入 $a + b = 2$，则：$(\sqrt{a+1} + \sqrt{b+1})^2 \le (2 + 2) \cdot 2 = 8$


由于左侧大于0，两边同时开平方根可得：$\sqrt{a+1} + \sqrt{b+1} \le \sqrt{8} = 2\sqrt{2}$

当且仅当 $\frac{\sqrt{a+1}}{1} = \frac{\sqrt{b+1}}{1}$，即 $a = b = 1$ 时等号成立

证毕


### 方法三：利用算术-平方平均不等式 (QM-AM)

对于任意正数 $x, y$ 有 $\frac{x+y}{2} \le \sqrt{\frac{x^2+y^2}{2}}$。

令 $x = \sqrt{a+1}, y = \sqrt{b+1}$。

根据不等式可得：$\frac{\sqrt{a+1} + \sqrt{b+1}}{2} \le \sqrt{\frac{(\sqrt{a+1})^2 + (\sqrt{b+1})^2}{2}}$


简化不等式可得：$\frac{\sqrt{a+1} + \sqrt{b+1}}{2} \le \sqrt{\frac{a + 1 + b + 1}{2}} = \sqrt{\frac{a + b + 2}{2}}$


代入 $a + b = 2$ 可得：$\frac{\sqrt{a+1} + \sqrt{b+1}}{2} \le \sqrt{\frac{2 + 2}{2}} = \sqrt{2}$


两边同乘以2：$\sqrt{a+1} + \sqrt{b+1} \le 2\sqrt{2}$

当且仅当 $a+1 = b+1$，即 $a = b = 1$ 时等号成立

证毕