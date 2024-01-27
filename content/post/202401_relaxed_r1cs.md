---
title: 【笔记】A Folding Scheme for Committed Relaxed R1CS
date: 2024-01-27T18:00:00+08:00
categories:
  - zkp
  - r1cs
  - folding
---
论文链接：[Nove](https://eprint.iacr.org/2021/370.pdf) （第三章）

本文介绍一种折叠方案（folding scheme），Nova 协议中利用它的非交互性质和零知识性质去实现零知识的IVC（incrementally-verifiable computation）。

## 1. R1CS 和 Relaxed R1CS
### 1.1 R1CS

> Consider a finite field $\mathbb{F}$. Let the public parameters consists of size bounds $m, n, l \in \mathbb{N}$ where $m > l$. The R1CS structure consists of sparse matrices $A, B, C \in \mathbb{F}^{m \times m}$ with at most $n = \Omega(m)$ non-zero entries in each matrix. An instance $x \in \mathbb{F}^l$ consists of public inputs and outputs and is satisfied by a witness $W \in \mathbb{F}^{m - l - 1}$ if $(A \cdot Z) \circ (B \cdot Z) = (C \cdot Z)$, where $Z = (W, x, 1)$

在R1CS结构中，有三个 $m \times m$ 的矩阵 A，B，C 和一个长度为m向量 $Z$，它们需要满足$(A \cdot Z) \circ (B \cdot Z) = (C \cdot Z)$的关系，其中A，B，C三个矩阵用来表达约束关系，$Z = (W, x, 1)$ 向量的内容为输入，包括表示为W的witness，表示为 x 的 instance，即公开的输入输出，和一个1。

### 1.2 Relaxed R1CS

> Consider a finite field $\mathbb{F}$. Let the public parameters consists of size bounds $m, n, l \in \mathbb{N}$ where $m > l$. The relaxed R1CS structure consists of sparse matrices $A, B, C \in \mathbb{F}^{m \times m}$ with at most $n = \Omega(m)$ non-zero entries in each matrix. A relaxed R1CS instance consists of an error vector $E \in \mathbb{F}^m$, a scalar $u \in \mathbb{F}$，and public inputs and outputs $x \in \mathbb{F}^l$. An instance $(E, u, x)$ is satisfied by a witness $W \in \mathbb{F}^{m-l-1}$ if $(A \cdot Z) \circ (B \cdot Z) = u \cdot (C \cdot Z) + E$, where $Z = (W, x, u)$.

与R1CS结构稍有不同，在Relaxed R1CS结构中，除了三个 $m \times m$ 的矩阵 A，B，C 和一个长度为m向量 $Z$，instance中还包括另外两项：一个向量E 和 一个标量 u，它们要满足$(A \cdot Z) \circ (B \cdot Z) = u \cdot (C \cdot Z) + E$。而 $Z = (W, x, u)$。
R1CS结构是Relaxed R1CS结构的一种特殊形式，其中 $u=1, E = \vec{0}$。

### 1.3 Committed Relaxed R1CS
> Consider a finite field $\mathbb{F}$ and a commitment scheme Com over $\mathbb{F}$. Let the public parameters consist of size bounds $m, n, l \in \mathbb{N}$ where $m > l$, and commitment parameters $pp_W$ and $pp_E$ for vectors of size m and m-l-1 respectively. The committed relaxed R1CS structure consists of sparse matrices $A, B, C \in \mathbb{F}^{m \times m}$ with at most $n = \Omega(m)$ non-zero entries in each matrix. A committed relaxed R1CS instance is a tuple $(\overline{E}, u, \overline{W}, x)$ is satisfied by a witness $(E, r_E, W, r_W) \in (\mathbb{F}^m, \mathbb{F}, \mathbb{F}^{m-l-1})$ if $\overline{E} = Com(pp_E, E, r_E), \overline{W} = Com(pp_W, W, r_W)$, and $(A \cdot Z) \circ (B \cdot Z) = u \cdot (C \cdot Z) + E$, where $Z = (W, x, u)$.

由于W 和 E 中包含了witness的信息，所以需要使用多项式承诺把它们隐藏起来，即$\overline{E} = Com(pp_E, E, r_E), \overline{W} = Com(pp_W, W, r_W)$。我们称这种修改过的协议为 committed relaxed R1CS。
## 2. Folding for Relaxed R1CS
给定一组R1CS的矩阵 $A, B, C$，和它对应的两组witness-inputs $(x_1, W_1)$ 和 $(x_2, W_2)$。我们想设计一种方案来化简这两组 witness-inputs 的检查成一组新的 witness-inputs $(x, W)$ 在相同R1CS矩阵上的检查，最直接的办法就是对这两组值做随机线性组合。

首先由verifier 产生一个随机数 $r \in \mathbb{F}$。prover 和 verifier 就可以去计算： $x = x_1 +  r x_2, W = W_1 + r W_2$，

构造 $Z = (W, x, 1+r) = (W_1 + r W_2, x_1 + r x_2, 1+r)= Z_1 + r Z_2$。

那么：
$(A \cdot Z) \circ (B \cdot Z) = (A \cdot (Z_1 + r Z_2)) \circ (B \cdot (Z_1 + r Z_2))$
$= (A \cdot Z_1) \circ (B \cdot Z_1) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1)) + r^2 \cdot (A \cdot Z_2) \circ (B \cdot Z_2)$
$= (C \cdot Z_1) + r^2 (C \cdot Z_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1))$
$\neq C \cdot Z$

显然，新构造的Z不能满足相同R1CS矩阵上的检查，这里
1. 首先，增加了额外的项，被称为cross-term，使得其等于 $T = r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1))$
2. 其次，$(C \cdot Z_1) + r^2 (C \cdot Z_2) \neq C \cdot (Z_1 + r Z_2)$

因此，若使用R1CS结构直接去做 folding 是不太好做的，但如果使用Relaxed R1CS去做，似乎就容易很多
$(A \cdot Z) \circ (B \cdot Z) = (A \cdot Z_1) \circ (B \cdot Z_1) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1)) + r^2 \cdot (A \cdot Z_2) \circ (B \cdot Z_2)$
$= (u_1 \cdot C \cdot Z_1 + E_1) + r^2 \cdot  (u_2 \cdot C \cdot Z_2 + E_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1))$
$=  (u_1 \cdot C \cdot Z_1 + (r \cdot u_2) \cdot C \cdot (r \cdot Z_2)) + (E_1 + r^2 E_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1))$
$= (u_1 + r u_2) \cdot C \cdot (Z_1 + r Z_2) - r \cdot (u_1 \cdot C \cdot Z_2 + u_2 \cdot C \cdot Z_1) + (E_1 + r^2 E_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1))$
$= (u_1 + r u_2) \cdot C \cdot (Z_1 + r Z_2)  + (E_1 + r^2 E_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1) - (u_1 \cdot C \cdot Z_2 + u_2 \cdot C \cdot Z_1))$

若令 $u = u_1 + r u_2, Z = Z_1 + r \cdot Z_2$,

$E = E_1 + r^2\cdot E_2 + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1) - u_1 \cdot C \cdot Z_2 - u_2 \cdot C \cdot Z_1)$

那么

$(A \cdot Z) \circ (B \cdot Z)$

$= (u_1 + r u_2) \cdot C \cdot (Z_1 + r Z_2)  + (E_1 + r^2 E_2) + r \cdot ((A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1) - (u_1 \cdot C \cdot Z_2 + u_2 \cdot C \cdot Z_1))$

$= u \cdot C \cdot Z + E$

## 3. A Folding Scheme for Committed Relaxed R1CS
下面来看一下完整的协议：

0. setup
	1. ${G}(1^\lambda) \rightarrow pp$：输出 $m,n,l \in \mathbb{F}$，及承诺的参数 $pp_W$ 和 $pp_E$，长度分别为 m 和 m-l-1。
	2. $K(pp, (A, B, C)) \rightarrow (pk, vk)$：输出pk 和 vk 
1. 证明者（prover）拥有两组witness: $(E_1, r_{E_1}, W_1, r_{W_1})$ 和$(E_2, r_{E_2}, W_2, r_{W_2})$和对应的instance：$(\overline{E}_1, u_1, \overline{W}_1, x_1)$ 和 $(\overline{E}_2, u_2, \overline{W}_2, x_2)$ ，验证者（verifier）获得这两组instance：$(\overline{E}_1, u_1, \overline{W}_1, x_1)$ 和 $(\overline{E}_2, u_2, \overline{W}_2, x_2)$ 
2. 证明者（prover）产生随机数 $r_T$，并计算cross term T及其承诺，并将承诺发送给验证者（verifier）
	- $T = (A \cdot Z_1) \circ (B \cdot Z_2) + (A \cdot Z_2) \circ (B \cdot Z_1) - u_1 \cdot C \cdot Z_2 - u_2 \cdot C \cdot Z_1$，
	- $\overline{T} = Com(pp_E, T, r_T)$
3. 验证者（verifier）生成随机数 $r$
4. 证明者（prover）和验证者（verifier）分别折叠两组instance为一组instance $(\overline{E}, u, \overline{W}, x)$
	- $\overline{E} = \overline{E} + r \cdot \overline{T} + r^2 \cdot \overline{E}_2$
	- $u = u_1 + r \cdot u_2$
	- $\overline{W} = \overline{W}_1 + r \cdot \overline{W}_2$
	- $x = x_1 + r \cdot x_2$
5. 证明者（prover）折叠两组witness为一组witness $(E, r_E, W, r_W)$
	- $E = E_1 + r \cdot T + r^2 \cdot E_2$
	- $r_E = r_{E_1} + r \cdot r_T + r^2 \cdot r_{E_2}$
	- $W = W_1 + r \cdot W_2$
	- $r_W = r_{W_1} + r \cdot r_{W_2}$

至此，便可以将两组Relaxed R1CS结构的witness-instance 折叠为一组，在同样的Relaxed R1CS结构下进行验证。