---
title: Caulk / Caulk +
date: 2024-01-05T20:30:50+07:00
categories:
  - zkp
  - lookup
  - commitment
---
*本文为hackerhouse 期间阅读论文的学习笔记，如有笔误，欢迎指正。*

Caulk 是一种向量承诺方案，可以零知识的证明一个或多个值都属于一个向量，并且不泄露这一个或者多个值的位置。Caulk可以用来做Membership证明，也可以作为Lookup  Argument，并且比先前的方案在生成证明的效率上更有优势。
Caulk+ 是Caulk的优化版本方案，它用了一个被称为「polynomial divisibility check」的方法来替换原本的子协议，以提升Caulk证明的时间。

## 1. 前提知识

### 1.1 Commitment  Schemes

承诺方案（Commitment schemes）是一种很重要的密码学原语。它允许承诺的一方公开一个被称为承诺（commitment）的值，承诺的值与某个消息绑定（binding），同时又不泄露消息的具体内容（hiding）。
1. 向量承诺（Vector Commitments）
	向量承诺（Vector Commitments，简称 VCs）允许通过某种方式去承诺一组有顺序的值，或者称为向量，并且在后面可以在某个特定的位置打开（open）它。

2. 多项式承诺（Polynomial Commitments）
	多项式承诺（Polynomial Commitments）允许承诺这去承诺一个多项式，并且验证者可以用它来验证承诺多项式所声明的值的计算。可以用来验证多项式在某个点出的取值。

3. KZG10 做向量承诺
	KZG10 承诺方案原本是一种多项式承诺方案，它同样也可以用来做向量承诺，并证明某个值（$c_i$）等于被承诺的向量在某个特定位置（$i$）的值。
	假设有一个向量 $\vec{c_i}$，长度为`N`。我们可以用它生成一个多项式 $C(x) = \sum_{i}^N c_i \cdot  \lambda_i(x)$，其中 $\lambda_i(x)$ 是拉格朗日插值多项式，满足$C(\omega^i) = c_i$。 $\{ \omega^i \}$ 是一组单位根 $\mathbb{H} = \{1, \omega, …, \omega^{N-1} \}$，满足 $\omega^N = 1$。当且仅当证明者（prover）知道向量 $\vec{c_i}$的所有值，才能构造多项式 $C(x)$ 。那么就可以通过对多项式 $C(x)$ 做多项式承诺并验证，来代替直接去向量 $\vec{c_i}$的承诺和验证。

Caulk, Caulk+（以及很多其他的Lookup  Argument ，如Baloo，CQ，CQLin等）就是基于 KZG10 设计的方案，所以在讨论 Caulk, Caulk+ 之前，先来看看 KZG10方案的步骤。

### 1.2 KZG10 步骤

使用KZG10的方案来使得验证者（verifier）相信证明者（prover）知道一个多项式C(x)的步骤：
1. 首先，证明者（prover）和验证者（verifier）获得一组安全的 srs 参数
2. 证明者（prover）使用 SRS 参数生成 C(x) 的承诺（commitment）, 并公开承诺值
	$C = [c(\tau)]_1$
3. 验证者（verifier）生成随机数 $\alpha$ 并发送给证明者（prover）
4. 证明者（prover）计算多项式 C(x) 在$\alpha$ 处的取值 $y = C(\alpha)$，并生成证明 $\pi$，将 $(y, \pi)$ 发送给验证者（verifier）。
	$\frac{C(x)-y}{x-\alpha} = Q(x), \pi = [Q(\alpha)]_1$，
5. 验证者（verifier）执行验证
		$e(C-[y]_1, [1]_2) = e(\pi, [\alpha]_2 - [1]_2)$

KZG10 方案还可以用来一次做多点的打开（batch opening）和验证，协议步骤如下：
1. 首先，证明者（prover）和验证者（verifier）获得一组安全的 SRS 参数
2. 证明者（prover）使用 SRS 参数生成 C(x) 的承诺（commitment）, 并公开承诺值
		$C = [c(\tau)]_1$
3. 验证者（verifier）生成一组随机挑战数  $\{\alpha_1, \alpha_2, … ,\alpha_m \}$ 并发送给证明者（prover）
4. 证明者（prover）计算多项式：
	1. $z_I(x) = \prod_{i=1}^m (x-\alpha_i)$
	2. $c_I(x) = \sum_{i=1}^m y_i \tau_i(x)$，其中 $y_i = c(\alpha_i), i \in [1,k]$,  $c_I(\omega^i) = y_i$
	3. 证明者（prover）继续计算证明 $Q(x) = \frac{c(x) - c_I(x)}{z_I(x)}$, $\pi = [Q(x)]_1$
5. 证明者（prover）发送 $\{ \pi, \{y_i\}_{i=0}^m \}$ 给验证者（verifier）
6. 验证者（verifier）生成 $[C_I]_1 = [c_i(\tau)]_1$ 和 $z_I(x)$，并执行验证：
		$e(C-[C_I]_1, [1]_2) = e(\pi, [z_I(x)]_2)$

## 2. Caulk 实现原理及步骤

Caulk 实现的是在不泄露点的位置的前提下，验证向量在某个（或某些）点处的取值。这里分成在单个点验证和多个点验证两种情况来讨论。
### 2.1 single element
#### 1. 修改协议步骤
很显然使用最原生的 KZG10 还远不能实现 Caulk 的目标，原因是因为：
1）证明这（prover）在验证者（verifier）生成的一个随机值 $\alpha$处做验证，而 Caulk 需要在一个特定的点处做验证；
2）KZG10 中要求被验证的点及多项式在这个点处的取值y都是公开的，而 Caulk 需要隐藏这些值。

首先，KZG10可以验证任意位置的取值，包括 $\omega^i$，当前仅当证明者（prover）知道多项式C(x)在 $\omega^i$处的取值，他才能算出正确的证明信息。所以可以使用KZG10来验证多项式在特定点的取值。
$\frac{C(x) - y_v}{x - \omega^v} = Q(x)$ 

其次我们需要在上述多项的基础上继续做修改，来隐藏 $\omega^v$ 和 $y_v$，为了实现这个目标同时也需要隐藏 Q(x)。
1) $\omega^v$
	为了隐藏 $\omega^v$，这里由证明者（prover）产生了一个随机数 a，并生成多项式 $z(x) = a(x-\omega^v) = ax -b$。由于随机数a是不公开的，因此若证明者（prover）将z(x)的commitment发送给验证者（verifier），他无法从中提取  $\omega^v$ 的信息。
	但这里会引入一个问题，就是验证者（verifier）无法确信证明者（prover）发送的值确实是由多项式 z(x) = ax-b 计算而来的，而不是一个作弊的值。为了解决这个问题，这里使用了一个子协议 $\pi_{unity}$ 来解决这个问题（下文中展开说明）。
	
2) $y_v$
	为了隐藏 $y_v$，这里使用了 Pedersen Commitment。Pedersen Commitment也是一种类型的承诺方案，它允许证明者承诺某个值，而不透露它或能够改变它。（下文中展开说明）
	
3) Q(x)
	证明者（prover）产生一个随机数s，并生成一个新的多项式 $T(x) = \frac{Q(x)}{a} + hs$。为了保持等式成立，他需要继续生成另一个多项式 $S(x) = -r -sz(x)$。
	
现在我们可以开始构造新的等式了：
$$
\frac{C(x) - (y_v + hr)}{z(x)}  = \frac{\frac{C(x) - y}{x-\omega^v}}{a} + \frac{-hr}{z(x)} = \frac{Q(x)}{a} + \frac{-hr}{z(x)} = T(x) - hs - h \cdot \frac{r}{z(x)} = T(x) + h \frac{S(x)}{z(x)}
$$
于是我们最终得到一个关系：
$$ C(x) - (y_v + hr) = T(x) z(x) + hS(x)$$
#### 2. $\pi_{unity}$

对于多项式 $z(x) = a (x - \omega ^ v) = ax - b$。这里有一个重要的性质，由于$\omega^n=1$，所以 $b^n = (a \cdot \omega ^v)^n = a^n$。
为了验证这层关系，我们首先定义一组值：
1) $f_0 = z(1) = a-b$
2) $f_1 = z(\sigma) = a\sigma - b$
3) $f_2 = \frac{f_0 - f_1}{1-\sigma} = \frac{a(1-\sigma)}{1-\sigma} = a$
4) $f_3 = \sigma f_2 - f_1 = \sigma a - \sigma a + b =b$
5) $f_4 = \frac{f_2}{f_3} = \frac{a}{b}$
6) $f_{5+i} = f_{4+i}^2,  i\in [0,…,\log N)$ 
7) $f_{4+\log N} = 1$
其中 $\sigma$是子群的单位根 $\mathbb{V} = \{1,…, \sigma^{n-1}\}$，其中  $\sigma^n = 1$, $n = \log N +6$。我们再继续定义相对应的拉格朗日插值多项式 $\{\rho_i(x)\}$ 和vanishing多项式$z_{V_n}(X)$。
现在我们继续定义一个新的多项式，使得$f(\sigma^i) = f_i$。
$$
f(X) = (a-b) \rho_1(X) + (a\sigma - b) \rho_2(X) + a \rho_3(x) + b \rho_4(x) + \sum_{i=0}^{\log N} (\frac{a}{b})^{2^i} \rho_{5+i}(x) + r_0\rho_{5+\log(N)}(X) + r(X)z_{V_n}(X)
$$
其中$r(X) = r_1 + r_2 X + r_2 X^2$，其中$r_1, r_2, r_3$ 为随机数。
很显然：
1) $f(\sigma^0) = a \sigma^0 -b$
2) $f(\sigma^1) = a\sigma^1 - b$
3) $f(\sigma^2) (1-\sigma) = f(\sigma_0) - f(\sigma_1) = a$
4) $f(\sigma^3) = \sigma f(\sigma^2) - f(\sigma^1) = b$
5) $f(\sigma^4) * f(\sigma^3) = f(\sigma^2)$
6) $f(\sigma^{5+i}) = f(\sigma^{4+i})^2, i=0..\log{(N)}-1$
7) $f(\sigma^{4+\log N}) = 1$

我们再继续构造新的多项式
$p(X) = (f(X) - (aX - b)) \rho_1(X) +(f(X) - (aX - b)) \rho_2(X)$
$+ ((1-\sigma)f(X)+ (f(\sigma^{-1}X) - f(\sigma^{-2}X))) \rho_3(X) + (f(x) - (\sigma f(\sigma^{-1}X) - f(\sigma^{-2}X))) \rho_4(X)$
$+(f(x)f(\sigma^{-1}X) - f(\sigma^{-2}X)) \rho_5(X) + (f(X) - f(\sigma^{-1}X)f(\sigma^{-1}X))\prod_{i \notin [5; 4+log(N)]}(x-\sigma^{i}) + (f(\sigma^{-1}X) -1)\rho_n(X)$
当$x= \sigma^i, p(X) = 0$，并且p(X)可以被 $z_{V_n}(X)$整除。所以继续定义一个多项式关系$\hat{h}(X) z_{V_n}(X) = p(X)$。
为了隐藏信息，我们继续构造多项式 $h(X) = \hat{h(X)} + x^{d-1}z(X)$。
因此我们可以通过验证多项式p(X)来最终实现验证 $[z]_2$ 的目的。

完整的验证步骤：
1. 证明者（prover）产生随机数$r_0, r_1, r_2, r_3$，并生成多项式 $r(X) = r_1 + r_2X + r_3X^2$
2. 证明者（prover）继续定义多项式 $f(X), p(X), \hat{h}(X), h(X)$
	1. $f(X) = f(\sigma^0) \rho_1(X) + f(\sigma^1)\rho_2(X) + a \rho_3(x) + b \rho_4(x) + \sum_{i=0}^{\log N} (\frac{a}{b})^{2^i} \rho_{5+i}(x) + r_0\rho_{5+\log(N)}(X) + r(X)z_{V_n}(X)$
	2. $p(X) = (f(X) - (aX - b)) (\rho_1(X) + \rho_2(X))$
	   $+ ((1-\sigma)f(X)+ f(\sigma^{-1}X) - f(\sigma^{-2}X)) \rho_3(X) + (f(x) - (\sigma f(\sigma^{-1}X) - f(\sigma^{-2}X))) \rho_4(X)$
	    $+(f(x)f(\sigma^{-1}X) - f(\sigma^{-2}X)) \rho_5(X) + (f(X) - f(\sigma^{-1}X)f(\sigma^{-1}X))\prod_{i \notin [5; 4+log(N)]}(x-\sigma^{i}) + (f(\sigma^{-1}X) -1)\rho_n(X)$
	3. $\hat{h}(X) = \frac{p(X)}{z_{V_n}(X)}$
	4. $h(X) = \hat{h(X)} + x^{d-1}z(X)$
3. 证明者（prover）生成承诺$[F]_1 = [f(x)]_1, [H]_1 = [h(x)]_1$，并将其发送给验证者（verifier）
4. 验证者（verifier）产生随机值 $\alpha$，并发送给证明者（prover）
5. 证明者（prover）继续定义$\alpha_1 = \sigma^{-1}\alpha, \alpha_2 = \sigma^{-2}\alpha$，
   $p_{\alpha}(X) = -z_{V_n}(\alpha)h(X)+(f(X)-z(X))(\rho_1(\alpha)+\rho_2(\alpha)) + ((1-\sigma)f(X) - f(\alpha_2)+f(\alpha_1))\rho_3(\alpha)$
     $+(f(X)+f(\alpha_2)-\sigma f(\alpha_1))\rho_4(\alpha) + (f(X)f(\alpha_1) - f(\alpha_2))\rho_5(\alpha)$
     $+(f(X)-f(\alpha_1)f(\alpha_1))\prod_{i \notin [5; 4+log(N)]}(\alpha - \sigma^i) + (f(\alpha_1)-1)\rho_n(\alpha)$
6. 证明者（prover）生成多项式 f(x) 承诺在 $\alpha_1, \alpha_2$ 处的open的证明 $((v_1, v_2), \pi_1)$，再继续生成多项式$p_{\alpha}(X)$ 在$\alpha$ 处的open的证明$(0, \pi_2)$，并将证明发送给$(v_1, v_2, \pi_1, \pi_2)$ 验证者（verifier）
7. 验证者（verifier）计算$\alpha_1 = \sigma^{-1}\alpha, \alpha_2 = \sigma^{-2}\alpha$，及 $[P]_1, \pi_2 =[q]_1$
	 $[P]_1 = -z_{V_n}(\alpha)[H]_1 + (\rho_1(\alpha)+\rho_2(\alpha))[F]_1 + \rho_3(\alpha)((1-\rho)[F]_1 + v_1 -v_2) + \rho_4(\alpha)([F]_1 + v_2 - \sigma v_1)$
   $+ \rho_5(\alpha)(v_1[F]_1 - v_2) + \rho_n(\alpha)(v_1 -1) +\prod_{i \notin [5; 4+log(N)]}(\alpha - \sigma^i)([F]_1 - v_1^2)$
8. 验证者（verifier）执行验证步骤
	1. KZG.verify($[F]_1, (\alpha_1, \alpha_2), (v_1, v_2), \pi_1$)
	2. $e([P]_1, [1]_2) + e(-(\rho_1(\alpha) + \rho_2(\alpha)) - z_{v_n}(\alpha)[x^{d-1}]_1,[z]_2) = e([q]_1, [x-\alpha]_2)$
#### 3. $\pi_{ped}$

为了隐藏 $y_c$，这里使用了pedersen commitment，具体步骤如下：
1. 在Trusted Setup阶段，产生一个承诺值 $H = [h]_1$,h是一个隐藏的随机值，任何人都不知道它的具体值
2. 证明者（prover）生成两个随机数：$s_1,s_2$，产生一个承诺$R = [s_1 + hs_2]_1$，并将 R 发送给验证者（verifier）
3. 验证者（verifier）生成两个随机数：r，v，并发送给证明者（prover）
4. 证明者（prover）产生$y_c$的承诺$cm = [y_v + hr]$，并计算两个值$t_1 = s_1 + y_vc, t_2 = s_2 + rc$，然后将两个值发送给验证者（verifier）
5. 验证者（verifier）检查等式关系 $R + c \cdot cm =  [t_1 + ht_2]$.

#### 4. 验证步骤

完整的验证步骤总结如下：
1. 首先，证明者（prover）和验证者（verifier）获得一组安全的 srs 参数
2. 证明者（prover）使用 SRS 参数生成 C(x) 的承诺（commitment）, 并公开承诺值
	$C = [c(\tau)]_1$
3. 证明者（prover）选择一个位置 i 的值 $y_v$
4. 证明者（prover）产生两个随机值 a，s
5. 验证者（verifier）生成两个随机数 r，v，并将它们发送给证明者（prover）
6. 证明者（prover）产生 $y_v$ 的承诺$cm = [y_v + hr]$， 并公开
7. 证明者（prover）定义三个多项式
	1. $z(X) = a(X - \omega^{i})$
	2. $T(x) = \frac{C(x) - v}{z(X)}+sh$
	3. $S(X) = -r-sz(X)$
8. 证明者（prover）生成证明 $({[z]_2, [T]_1, [S]_2, \pi_{ped}, \pi_{unity}})$，并将其发送给验证者（verifier）
9. 验证者（verifier）检查
	1. $e(C-cm, [1]_2) = e([T]_1, [z]_2) + e([h]_1, [S]_2)$
	2. Verify($\pi_{ped}$)
	3. Verify($\pi_{unity}$)

### 2.2 subset elements

上一节我们讨论了如何使用KZG10方案在不揭露具体值和值所在位置的前提下验证向量中某个具体值的方案。本节我们继续验证向量中多个点（m个，集合定义为 $I$）的情况。这种对subset elements的验证也可以被称为KZG向量承诺方案的`position-hiding linkability`。
#### 1. 修改协议步骤
这里使用KZG10 的batch opening方案，与单点验证一样，同样需要做一些修改来隐藏信息。
1) $c_I(x)$
2) $z_I(x)$
3) $Q(x)$
##### 1. $z_I(x)$
首先，证明者（prover）产生一个随机值$r_1$，并生成 $z_I(x) = r_1 \prod_{i=1}^m (x-\omega^{k_i})$。为了验证者$z_I(x)$，需要通过以下方法执行验证：
1) 首先，构造一个子群，单位根为$v$，$\mathbb{V}_m = \{v^0, v^1,…, v^{m-1} \}$, 其中 $v^m = 1$。相对应的vanishing多项式定义为$z_{V_m}(x)$
2) 然后证明者（prover）构造一个新的多项式$u(x) = \sum_{i=0}^{m-1} \omega^{k_i} \mu_i(x)$，其中 $\{\mu_i(x)\}$ 是拉格朗日插值多项式，使得 $u(v^i) = \omega^{k_i}$
3) 因为当$x = \omega^{k_i}$时，$z_I(x) = 0$，所以当$x = v^i$时，$z_I(u(x)) = 0$。换句话说，$z_I(u(x))$可以被 $z_{V_m}(x)$ 整除。我们可以定义等式关系$z_I(u(x)) = z_{V_m}(x)H_2(x)$
4) 那么也就是说我们这里的验证分为两个部分：
	1)  $u(v^i) = \omega^{k_i}$，这里使用了一个子协议 $\pi_{unity}$ 来解决这个问题（下文中展开说明）。
	2) $z_I(u(x)) = z_{V_m}(x)H_2(x)$
##### 2. $c_I(x)$
为了隐藏$\{y_i\}$，我们要去构造一个新的多项式
$c_I(x) = \sum_{i=1}^m y_i \tau_i(x) + (r_2 + r_3 X + r_4 X^2) z_I(x)$.
证明者（prover）生成承诺 $[C_I]_1 = [c_I(x)]_1$.为了验证这个承诺，需要执行以下步骤：
1) 证明者（prover）产生随机数$a_{m+1}$，并构造另一个多项式$\phi(x) = \sum_{i=1}^m y_i \mu_i(x) + a_{m+1}z_{V_m}(x)$，其中$a_{m+1}$ 是由证明者（prover）生成的随机值，并计算承诺 $cm = [\phi(x)]_1$。
2) 由于$c_I(\omega^{k_i}) = y_i$，所以$c_I(u(v^i)) = y_i$。也就是说对于$i \in [0…m-1]$, $c_I(u(x)) = \phi(x)$。换句话说 $c_I(u(x)) - \phi(x)$ 可以被 $z_{V_m}(x)$ 整除。我们可以定义一个等式关系$c_I(u(x)) - \phi(x) = z_{V_m}(x)H_3(x)$

##### 3. $Q(x)$
现在，我们重新构造等式
$$
\frac{c(x) - c_I(x)}{z_I(x)} = \frac{c(x) - (\sum_{i=1}^m y_i \tau_i(x) + (r_2 + r_3 X + r_4 X^2) z_I(x))}{r_1 \prod_{i=1}^m(x-\omega^{k_i})}
$$
$$
= \frac{c(x) - (\sum_{i=1}^m y_i \tau_i(x))}{r_1 \prod_{i=1}^m(x-\omega^{k_i})} - \frac{(r_2 + r_3 X + r_4 X^2) z_I(x)}{r_1 \prod_{i=1}^m(x-\omega^{k_i})}  = \frac{Q(x)}{r_1} - (r_2 + r_3 X + r_4 X^2)
$$
因而我们再继续定义一个新的多项式 $H_1(x) = \frac{Q(x)}{r_1} - (r_2 + r_3 X + r_4 X^2)$.

#### 2.  $\pi_{unity}$
具体步骤如下：
1. 证明者（prover）生成一组随机数$t_1,...,t_m$
2. 证明者（prover）生成多项式$u_s(X) = \sum_{j=1}^m (\omega^{i_j})^{2^s} u_j(X) + t_sz_{v_m}(X), s=1,...,m$
3. 证明者（prover）继续定义多项式:
	1. $U(X, Y) = \sum_{s=1}^n u_{s-1}(X)\rho_s(Y)$
	2. $\widetilde{U}(X,Y) = U(X, Y) - u(X)\rho_1(Y)$
	3. $h_2(X) = \sum_{s=1}^n \rho_s(Y)H_s(X), H_s(X)=(u_{s-1}^2(X) - u_s(X))/z_{V_m}(X)$
4. 证明者（prover）计算承诺$[\widetilde{U}]_1 = [\widetilde{U}(x^n, x)]_1, [h_2]_1 =[h_2(x^n, x)]_1$，并发送给验证者（verifier）
5. 验证者（verifier）产生随机数 $\alpha$，并发送给证明者（prover）
6. 证明者（prover）定义多项式 $h_1(Y)=(U^2(\alpha, Y)-\sum_{s=1}^n u_{s-1}^2(\alpha) \rho_s(Y))/z_{V_n}(Y)$，并计算承诺$[h_1]_1 = [h_1(x)]_1$，并发送给验证者（verifier）
7. 验证者（verifier）生成随机数$\beta$，并发送给证明者（prover）
8. 证明者（prover）继续计算多项式P(Y)，及多项式$u(X), \widetilde{U}(X,Y), h_2(X,Y), \widetilde{U}(\alpha, Y), p(Y)$ 的证明
		1. $P(Y) = (U^2(\alpha, \beta)) - h_1(Y)z_{V_n}(\beta) - \widetilde{U}(\alpha, \beta \sigma) + id(\alpha)\rho_n(\beta)) - z_{V_m}(\alpha)h_2(\alpha, Y)$
		2. $(v_1, \pi_1) = KZG.Open(srs, u(x), X = \alpha)$
		3. $([\widetilde{U}(\alpha,x)]_1, \pi_2) = KZG.Open(srs, \widetilde{U}(X, Y), X = \alpha)$
		4. $([\widetilde{h}_2(\alpha,x)]_1, \pi_3) = KZG.Open(srs, \widetilde{h_2}(X, Y), X = \alpha)$
		5. $((0, v_2, v_3), \pi_4) = KZG.Open(srs, \widetilde{U}(\alpha, Y), Y = (1, \beta, \beta \sigma))$
		6. $(0, \pi_5) = KZG.Open(srs, p(Y), Y = \beta)$
9.证明者（prover）定义$([\widetilde{U}_\alpha]_1 = [\widetilde{U}(\alpha,x)]_1), [\widetilde{h}_{2,\alpha}]_1 = [\widetilde{h}_2(\alpha,x)]_1$，并将 $([\widetilde{U}_\alpha]_1, \widetilde{h}_{2,\alpha}]_1, v_1, v_2, v_3, \pi_1, \pi_2, \pi_3, \pi_4, \pi_5)$ 发送给验证者（verifier）
10. 验证者（verifier）计算
	1. $U = v_1 \rho_1(\beta) + v_2$
	2. $[P]_1 = U^2 - [h_1]_1 z_{V_n}(\beta) - (v_3 + id(\alpha)\rho_n(\beta)) - z_{V_m}(\alpha)[h_{2,\alpha}]_1$
11. 验证者（verifier）对$\pi_1, \pi_2, \pi_3, \pi_4, \pi_5$ 执行验证

#### 3. 验证步骤
完整的验证步骤总结如下：
1. 首先，证明者（prover）和验证者（verifier）获得一组安全的 SRS 参数
2. 证明者（prover）使用 SRS 参数生成 C(x) 和 $\phi(x)$的承诺（commitment）,并公开承诺值 $C = [c(\tau)]_1$, $cm = [\phi(x)]_1$
3. 证明者（prover）产生随机值 $r_1, r_2, r_3, r_4, r_5, r_6, r_7$
4. 证明者（prover）定义多项式：
	1. $z_I(x) = r_1 \prod_{i = 1}^m (x - \omega^{k_i})$
	2. $c_I(x) = \sum_{i=1}^m y_i \tau_i(x) + (r_2 + r_3 x + r_4 x^2) z_I(x)$
	3. $H_1(x) = \frac{Q(x)}{r_1} - (r_2 + r_3 x + r_4 x^2)$
	4. $u(x) = \sum_{i=1}^m \omega^{k_i} \mu_i(x) + (r_5 + r_6x + r_7x^2)z_{V_m}(x)$
5. 证明者（prover）为u(x)生成证明$\pi_{unity}$，并生成承诺$[C_I]_1 = [C_I(x)]_1, [z_I]_1 = [z_I(x)]_1, [u]_1 = [u(x)]_1, [H_1]_2 = [H_1(x)]_2$。然后证明者（prover）发送这几个承诺和证明$\pi_{unity}$给验证者（verifier）
6. 验证者（verifier）生成随机值$\mathbb{x}$，并将其发送给证明者（prover）
7. 证明者（prover）将两个多项式$z_I(u(x)) = z_{V_m}(X)H_2(x)$ 和 $c_I(u(x)) - \phi(x) = z_{V_m}(x)H_3(x)$结合起来生成新的多项式$z_I(u(x)) + \mathbb{x}(c_I(u(x)) - \phi(x)) = z_{V_m}(x)H_2(x)$。
8. 证明者（prover）计算证明$[H_2]_1 = [H_2(x)]_1$，并发送给验证者（verifier）
9. 验证者（verifier）生成新的随机值 $\alpha$，并发送给证明者（prover）
10. 证明者（prover）继续构造多项式$p_1(x), p_2(x)$
	1. $p_1(x) = z_I(x) + \mathbb{x}c_I(x)$
	2. $p_2(x) = z_I(u(\alpha)) + \mathbb{x}(c_I(u(\alpha)) - \phi(X)) - z_{V_m}(\alpha) H_2(x)$
11. 证明者（prover）生成多项式$p_1(x), p_2(x), u(x)$的证明，并发送给验证者（verifier）
12. 验证者（verifier）计算
	1. $[P_1]_1 = [z_I]_1 + \mathbb{x}[C_I]_1$
	2. $[P_2]_1 = v_2 - \mathbb{x} cm - z_{v_m}(\alpha)[H_2]_1$
13. 验证者（verifier）执行验证
	1. $\pi_{unity}$，
	2. KZG.Verify($[u]_1$)
	3. KZG.Verify($[P_1]_1$)
	4. KZG.Verify($[P_2]_1$)
	5. $e([C]_1 - [C_I]_1, [1]_2) = e([z_I]_1, [H_1]_2)$

#### 4. 复杂度

Caulk 的证明时间相比较于之前的 Lookup Argument 协议在证明时间上有了很大的提升。当向量的大小为N，而子向量的大小为m时，生成证明的时间复杂度为 $O(m \log N + m^2)$，验证证明的复杂度为 $O(\log(\log N))$。

## 3. Caulk+ 优化方案

Caulk+ 对 Caulk 的算法步骤做了改进，将生成证明的复杂度降低到$O(m^2)$，这样证明的复杂度仅与子集的大小 m 有关，与向量 v 的长度 N 无关。
### 3.1 证明计算优化
KZG10做单点验证时，检查的等式关系为： $\frac{C(x)-c_i}{x-\omega^i} = Q_i(x)$
KZG10做多点（batch）验证时，检查的等式关系为： $\frac{c(x) - c_I(x)}{z_I(x)} = Q(x)$
多点的证明($Q(x)$)和单点的证明$(Q_i(x))$之间存在着关系：

 $\frac{c(x) - c_I(x)}{z_I(x)} = \frac{c(x) - c_I(x)}{\prod_{i \in I}(x-\omega^i)} = \sum_{i \in I}\frac{C(x)-c_i}{(x-\omega^i) \prod_{j \in I, i \neq j}(\omega^i - \omega^j)} = \sum_{i \in I} \frac{Q_i(x)}{\prod_{j \in I, i \neq j}(\omega^i - \omega^j)}$

因此证明者（prover）可以预先计算好所有的$\{Q_i\}_{i \in [0..N)}$，这样在每次生成证明的时候，直接带入 $\{Q_i\}_{i \in [0..N)}$ 去计算 Q(x)。

### 3.2 协议步骤
下面是完整的协议步骤：
1. 首先，证明者（prover）和验证者（verifier）获得一组安全的 srs 参数
2. 证明者（prover）构造关于向量$\vec{c}$ 的多项式 C(x) ，并计算承诺$C = [c(\tau)]_1$
3. 证明者（prover）先计算两组承诺
	1. $\{[W_1^{(i)}(X)]_2\}_{i \in I}$, $W_1^{(i)}(X) = (C(X) - c_i)/(X - \omega^i)$
	2. $\{[W_2^{(i)}(X)]_2 \}_{i \in I}$，$W_2^{(i)}(X) = z_I(X)/(X - \omega^i)$
4. 证明者（prover）计算 $\phi(x)$的承诺（commitment）,并公开承诺值$cm = [\phi(x)]_1$
5. 证明者（prover）产生随机数$r_1, r_2, r_3, r_4, r_5$
6. 证明者（prover）计算多项式
	1. $z_I(x) = r_1 \prod_{i = 1}^m (x - \omega^{k_i})$
	2. $c_I(x) = \sum_{i=1}^m y_i \tau_i(x),  c_I'(x) = c_I(x) + (r_2 + r_3 x + r_4 x^2) z_I(x)$ 
	3. $u(x) = \sum_{i=1}^m \omega^{k_i} \mu_i(x) + (r_5 + r_6x)z_{V_m}(x)$，满足$u(v_i) = \omega^{u(i)}, i \in [m]$
7. 证明者（prover）计算承诺$[C_I]_1 = [C_I(x)]_1, [z_I]_1 = [z_I(x)]_1, [u]_1 = [u(x)]_1, [H_1]_2 = [H_1(x)]_2$。并发送给验证者（verifier）
8. 验证者（verifier）产生随机数 $\chi_1, \chi_2$，并发送给证明者（prover）
9. 证明者（prover）计算
	1. 将两个多项式$z_I(u(x)) = z_{V_m}(X)H_2(x)$ 和 $c_I(u(x)) - \phi(x) = z_{V_m}(x)H_3(x)$结合起来生成新的多项式$z_I(u(x)) + \chi_1(c_I(u(x)) - \phi(x)) = z_{V_m}(x)H_2(x)$。
	2. $[W_1(X) + \chi_2 W_2(X)]_2 = \sum_{i \in I} \frac{[W_1^{(i)}(x)]_2 + \chi_2[W_2^{(i)}(x)]_2}{\prod_{j \in I, i \neq j}\omega ^i - \omega^j}$
10. 证明者（prover）计算承诺，并发送给验证者（verifier）
	1.  $[W]_2 = r_1^{-1}[W_1(X) + \chi_2 W_2(X)]_2 - [r_2 + r_3x + r_4 x^2]_2$
	2. $[H_2]_1 = [H_2(x)]_1$
11. 验证者（verifier）产生随机数 $\alpha$，并发送给证明者（prover）
12. 证明者（prover）继续构造多项式$p_1(x), p_2(x)$
	1. $p_1(x) = z_I(x) + \chi_1c_I(x)$
	2. $p_2(x) = z_I(u(\alpha)) + \chi_1(c_I(u(\alpha)) - \phi(X)) - z_{V_m}(\alpha) H_2(x)$
13. 证明者（prover）生成多项式$p_1(x), p_2(x), u(x)$的证明，并发送给验证者（verifier）
14. 验证者（verifier）首先计算：
	1. $[P_1]_1 = [z_I]_1 + \chi_1[C_I]_1$
	2. $[P_2]_1 = v_2 - \chi_1 cm - z_{v_m}(\alpha)[H_2]_1$
15. 验证者（verifier）执行验证：
	1. KZG.Verify($u, \alpha, v_1, \pi_1$)
	2. KZG.Verify($p_1, v_1, v_2, \pi_2$)
	3. KZG.Verify($p_2, \alpha, 0, \pi_3$)
	4. $e([C]_1 - [C_I]_1 + \chi_2[x^n -1]_1, [1]_2) = e([z_I]_1, [W]_2)$

### 3.3 单点验证的协议改进

上一小节中，我们讨论的协议步骤，用「polynomial divisibility check」来替换原来的子协议，而对于单点验证的步骤中，若采用同样的优化，可能引入一个问题，就是无法保证$[z]_2$确实是由正确的z(x)多项式计算得来的。为了解决这个问题，这里可以用schnor协议来执行验证。

#### schnorr protocol相关部分的步骤

1. 证明者（prover）生成随机数 $k, \hat{v}, \hat{r}, \hat{k}$
2. 证明者（prover）生成多项式$A(x) = y_v + k(x-1)$,并生成承诺 $A = [A(x)]_1$
3. 证明者（prover）生成 $V'= [\hat{v} + \hat{r}h]_1$, $A' = [\hat{v} + \hat{k}(x-1)]_1$，并将A, A', V' 发送给验证者（verifier）
4. 验证者（verifier）生成随机值$\chi$
5. 证明者（prover）生成三个值，并发送给验证者（verifier）
	1. $s_v = \hat{v} + \chi v$
	2. $s_r = \hat{r} + \chi r$
	3. $s_k = \hat{k} + \chi k$
6. 验证者（verifier）执行验证
	1. $[s_v]_1 + s_r [h]_1 = V' + \chi  cm$
	2. $[s_v]_1 + s_k[x_1]_1 = A' + \chi A$

## 4. 参考文献

* https://eprint.iacr.org/2022/621.pdf
* https://eprint.iacr.org/2022/957.pdf
* https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html 
* https://eprint.iacr.org/2011/495.pdf
* https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf
* https://web.getmonero.org/resources/moneropedia/pedersen-commitment.html
* https://en.wikipedia.org/wiki/Schnorr_signature
