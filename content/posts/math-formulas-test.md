---
title: "数学公式渲染测试"
date: 2025-01-17T16:00:00+08:00
draft: false
categories: ["AI"]
tags: ["数学", "LaTeX", "公式"]
author: "Baike"
description: "测试博客中的数学公式渲染功能"
math: true
---

## 数学公式渲染测试

这篇文章用来测试博客中的LaTeX数学公式渲染功能。

### 行内公式

这是一个行内公式：$E = mc^2$，爱因斯坦的质能方程。

另一个例子：当 $a \neq 0$ 时，方程 $ax^2 + bx + c = 0$ 的解为 $x = \frac{-b \pm \sqrt{b^2-4ac}}{2a}$。

### 块级公式

#### 1. 二次方程求根公式

$$x = \frac{-b \pm \sqrt{b^2-4ac}}{2a}$$

#### 2. 欧拉公式

$$e^{i\pi} + 1 = 0$$

#### 3. 积分公式

$$\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}$$

#### 4. 矩阵表示

$$\begin{pmatrix}
a & b \\
c & d
\end{pmatrix}
\begin{pmatrix}
x \\
y
\end{pmatrix}
=
\begin{pmatrix}
ax + by \\
cx + dy
\end{pmatrix}$$

#### 5. 求和公式

$$\sum_{i=1}^{n} i = \frac{n(n+1)}{2}$$

#### 6. 极限

$$\lim_{x \to 0} \frac{\sin x}{x} = 1$$

#### 7. 偏微分方程

$$\frac{\partial^2 u}{\partial t^2} = c^2 \frac{\partial^2 u}{\partial x^2}$$

### 机器学习中的数学公式

#### 线性回归损失函数

$$J(\theta) = \frac{1}{2m} \sum_{i=1}^{m} (h_\theta(x^{(i)}) - y^{(i)})^2$$

#### Sigmoid函数

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

#### 交叉熵损失

$$H(p,q) = -\sum_{i} p_i \log(q_i)$$

#### 梯度下降

$$\theta := \theta - \alpha \frac{\partial}{\partial \theta} J(\theta)$$

### 概率论公式

#### 贝叶斯定理

$$P(A|B) = \frac{P(B|A) \cdot P(A)}{P(B)}$$

#### 正态分布

$$f(x|\mu,\sigma^2) = \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{(x-\mu)^2}{2\sigma^2}}$$

### 测试结果

如果你能看到上面的公式正确渲染（而不是LaTeX源码），说明数学公式功能已经正常工作了！

### 使用方法

在Markdown文章中使用数学公式：

1. **行内公式**：使用单个美元符号包围：`$公式$`
2. **块级公式**：使用双美元符号包围：`$$公式$$`
3. **文章头部**：添加 `math: true` 启用数学渲染

## 总结

LaTeX数学公式渲染让博客能够更好地展示技术内容，特别是AI、机器学习等需要大量数学公式的领域。
