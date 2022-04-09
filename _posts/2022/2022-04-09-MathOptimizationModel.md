---
title: 数学优化模型分类
date: 2022-04-09 12:54:07 +08:00
tags:
- 数学
categories: [手册, 研究]
toc: true
comments: true
math: true
---

在云计算、边缘计算、雾计算等领域，优化问题十分常见，为方便后续研究工作，这里记录一些经常出现的数学模型以方便后续求解问题时的快速查阅，每个模型后的解释均来自引用论文。

> 目前仍在完善中，之前看的论文没有做记录，后续阅读的论文会不断扩充本篇内容
{: .prompt-warning}

> 引用的论文属于建模出的模型符合该类，并不是该类模型第一次出现、被定义的论文；被引用文章的出现与否取决于我个人的阅读到它们的顺序 😃
{: .prompt-info}

> 引用论文主要来源：
> * TCC
{: .prompt-tip}

## <font color=DodgerBlue>Mixed Integer Programming (MIP)</font>

### <font color=DodgerBlue>Two Stage Mixed Integer Programming</font>

$$
	\begin{array}{clc}
	\min _{z, x, v} & f_{1}(z)+f_{2}(x)+f_{3}(v) & \\
	\text { s.t. } & B_{1} z+B_{2} x & \leq b_{2} \\
	& B_{3} v & \leq b_{3} \\
	& A_{1} z+A_{2} x+A_{3} v & \leq b_{1} \\
	& z \in \mathbb{R}^{n}, x \in \mathbb{Z}^{m}, v \in \mathbb{R}^{p},
	\end{array}
$$

where Z is the set of integers. The first stage variables are z and x, and the second stage variables are v. The first and second stage constraints which are independent of each other are represented by the first two inequalities, while the constraints which couple the first and second stages are represented by the third inequality[^seholff].

## <font color=DodgerBlue>引用</font>

[^seholff]: Sehloff, David Karl, Maitreyee Marathe, Ashray Manur, and Giri Venkataramanan. "Self-sufficient participation in cloud-based demand response." IEEE Transactions on Cloud Computing (2021).
