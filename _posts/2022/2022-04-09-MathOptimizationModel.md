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

## <font color=DodgerBlue>Integer Programming (IP)</font>

### <font color=DodgerBlue>Integer Linear Programming (ILP)</font>

#### 定义
$$
\begin{array}{ll}
	\operatorname{maximize} & \mathbf{c}^{\mathrm{T}} \mathbf{x} \\
	\text { subject to } & A \mathbf{x} \leq \mathbf{b} \\
	& \mathbf{x} \geq \mathbf{0} \\
	& \mathbf{x} \in \mathbb{Z}^{n}
\end{array}
$$

> 现有成熟的工具求解 ILP 及 MILP 问题，比如 CPLEX 等。
{: .prompt-tip}

> Lenstra et al.[^Lenstra] have proved that no algorithm is able to find a solution in polynomial time with a smaller approximation ratio than 3/2.
{: .prompt-info}

#### 实例

1. 通过近似算法进行求解，需要考察近似率（Approximation rate）[^Szalay]。

## <font color=DodgerBlue>Mixed Integer Programming (MIP)</font>

ILP 中仅有部分变量具有整数约束，其余变量为实数的情况下为 MIP

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

### <font color=DodgerBlue>Mixed Integer Non-Linear Programming (MINLP)</font>

#### <font color=DodgerBlue>定义</font>

$$
\begin{aligned}
&\min &&f(x, y)\\
&\text { s.t. } &&c_{i}(x, y)=0 \quad \forall i \in E\\
& &&c_{i}(x, y) \leq 0 \quad \forall i \in I\\
& &&x \quad \in X\\
& &&y \quad \in Y \qquad\text { integer }
\end{aligned}
$$

where each $c_i(x,y)$ is a mapping from $R^n$ to $R$, and $E$ and $I$ are index sets for equality and inequality constraints, respectively. Typically, the functions $f$ and $c_i$ have some smoothness properties, i.e., once or twice continuously differentiable[^neosGuide].

Software developed for MINLP has generally followed two approaches[^neosGuide]:

* **Outer Approximation/Generalized Bender's Decomposition**: These algorithms alternate between solving a mixed-integer LP master problem and nonlinear programming subproblems.
* **Branch-and-Bound**: Branch-and-bound methods for mixed-integer LP can be extended to MINLP with a number of tricks added to improve their performance.

#### <font color=DodgerBlue>实例</font>

1. 将某个约束条件暂时不考虑的情况下，将约束中的变量分为互不干扰的两组，进而将原问题分解为主从问题[^Zhou2022TCC]。
   > 优化问题建模是 MINLP，放宽约束进行了分解，但是之后疑似未还原约束或证明问题相等，使得提出的解法是具备前提条件的：We first consider the scenario that ESs do not cache the relevant services, and generate an initial computing strategy that satisfies the above constraints.

## <font color=DodgerBlue>引用</font>

[^seholff]: Sehloff, David Karl, Maitreyee Marathe, Ashray Manur, and Giri Venkataramanan. "Self-sufficient participation in cloud-based demand response." IEEE Transactions on Cloud Computing (2021).
[^Zhou2022TCC]: H. Zhou, Z. Zhang, D. Li and Z. Su, "Joint Optimization of Computing Offloading and Service Caching in Edge Computing-based Smart Grid," in IEEE Transactions on Cloud Computing, doi: 10.1109/TCC.2022.3163750.
[^neosGuide]: https://neos-guide.org/content/mixed-integer-nonlinear-programming
[^Lenstra]: J. K. Lenstra, D. B. Shmoys, and ́E. Tardos, “Approximation algo-rithms for scheduling unrelated parallel machines,”Mathematicalprogramming, vol. 46, no. 1, pp. 259–271, 1990.
[^Szalay]: Szalay, M., Matray, P. & Toka, L. Real-Time FaaS: Towards a Latency Bounded Serverless Cloud. Ieee Transactions Cloud Comput PP, 1–1 (2022).
  