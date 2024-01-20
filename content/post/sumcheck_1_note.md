---
title: 【笔记】Sumcheck
date: 2024-01-20T1:00:00+08:00
categories:
  - zkp
  - polynomoial
---

## 1 多项式求和验证

假设在有限域 $\mathbb{F}$ 上定义一个有 l 项输入的多项式 f，当变量的取值都为0或者1时，共有$2^l$ 组输入的取值。若有一个求和的就算，即将这$2^l$ 组取值带入多项式 f 计算，并将结果相加：

$H = \sum_{b_1,b_2,...,b_l\in\{0,1\}}f(b_1,b_2,...,b_l)$，

若要直接完成这个求和计算，就需要先算出多项式 f 在这 $2^l$ 组输入上的结果，然后再进行求和计算。这里需要的时间复杂度为 l 的指数时间 $O(2^l)$。但是当计算条件有限无法进行这个复杂度计算的时候该怎么办呢？这里引入一个委托计算的概念，verifier不需要进行计算，计算被委托给了prover去完成，verifier只需要通过很低的消耗去验证计算结果的正确性就可以了。这里所说的委托证明可以通过 Sumcheck 来完成的，在Sumcheck协议中，verifier 的验证时间复杂度仅为 O(dl)（d为多项式 f 的阶数）。

我们先来看一个最简单的例子，

1. 一元多项式
    
    假设多项式 f 只有一个输入项，即 f(x)，那么
    
    $H = \sum_{b_1\in\{0,1\}}f(b_1) = f(0)+f(1)$，
    
    这里只有两项， verifier 可以直接带入多项式 f 计算 $f(0),f(1)$，再验证 $H = f(0)+f(1)$ 。
    
2. 二元多项式
    
    当 f 被扩展到有两项输入变量时，即$f(x_1,x_2)$，那么 $H = \sum_{b_1,b_2\in\{0,1\}}f(b_1,b_2)$，
    我们可以将它变换为 $H=\sum_{b_1 \in \{0,1\}}(\sum_{b_2 \in \{0,1\}}f(b_1,b_2)) = \sum_{b_1 \in \{0,1\}}f_1(b_1)$，其中 $f_1(x) = \sum_{b_2 \in \{0,1\}}f(x,b_2)$。
    也就是说如果我们知道多项式 $f_1(x)$，我们就可以用验证一元多项式的方式去验证H了。 那么把验证变换成两步：
    （1）验证 $H=f_1(0)+f_1(1)$了；
    （2）验证多项式 $f_1(x)$。
    
    首先prover给出多项式 $f_1(x)$， 然后用验证一元多项式的方式去验证 $H=f_1(0)+f_1(1)$；
    
    接下来再去验证上一步骤给出的多项式  $f_1(x) = \sum_{b_2 \in \{0,1\}}f(x,b_2)$  是否正确。
    verifier可以直接去算，但很显然这是很低效的。验证多项式的一种方法，就是我们只需要带入一个随机数去验证它的计算结果就相当于验证这条曲线了。
    若 x 取随机数$r_1$，多项式 $f_1(x)$ 就可以变换为 $f_1(r_1) = H'= \sum_{b_2 \in \{0,1\}}f_2(x_2)$ ，于是 $f_1(r_1)$ 的验证也就是验证 $H' = f_2(0)+f_2(1)$ ，这里继续用验证一元多项式的方式去验证。
    
    与$f_1(x)$ 类似，prover 计算多项式 $f_2(x)$，并发送给verifier进行验证。若x取随机数 $r_2$，多项式 $f_2(r_2) = f(r_1,r_2)$，verifier可以直接去计算 $f(r_1,r_2)$ 来验证 $f_2(x)$。
    
    总结一下，$H = \sum_{b_1,b_2\in\{0,1\}}f(b_1,b_2)$ 的验证可以拆成两步骤进行，每一步骤都当做是一个一元多项式来进行验证。
    
3. 多元多项式
    
    当g 被扩展到有l个变量的时候，即 $f(x_1,x_2,...,x_l)$，求和公式可以表示为：
    
    $H = \sum_{b_1,b_2,...,b_l\in\{0,1\}}f(b_1,b_2,...,b_l)$，
    
    与二元多项式验证类似，这里将验证拆成l步骤进行，每一步骤都当做一个一元多项式来进行验证。验证步骤就是：
    $$
    \sum_{b_1,b_2,...,b_l\in\{0,1\}}f(b_1,b_2,...,b_l) \overset{?}{=} \sum_{b_1 \in \{0,1\}}f_1(b_1) = f_1(0)+ f_1(1)
    $$
$$
\rightarrow f_1(r_1) \overset{?}{=} \sum_{b_2 \in \{0,1\}}f_2(b_2) = f_2(0)+f_2(1)
$$
$$
\rightarrow f_2(r_2) \overset{?}{=} \sum_{b_3 \in \{0,1\}}f_3(b_3) = f_3(0)+f_3(1)
$$
$$...$$          $$
    \rightarrow f_{n-1}(r_{n-1}) \overset{?}{=} \sum_{b_n \in \{0,1\}}f_n(b_n) = f_n(0)+f_n(1)
    $$
    $$
    \rightarrow f_n(r_n) \overset{?}{=} f(r_1,r_2,...,r_n)
    $$
    

这个验证过程就是 Sumcheck 协议。prover 首先将计算结果H发送给 verifier，然后双方再进行 l 轮的交互完成验证，每一轮verifier向prover发送一个挑战数，对多项式f 的 l 个变量逐个进行挑战，直到最后将求和的验证归结到了对 $f(r_1,r_2,...,r_l)$ 的验证。

## 2 Sumcheck 完整协议

总结一下，Sumcheck 协议的完整步骤：
![sumcheck](/images/contents/sumcheck-protocol.png)

## 3 细节讨论

前面介绍了Sumcheck的步骤，但还有几个关键的细节，

1. 在每一轮的验证中，prover如何将多项式传递给verifier？
    
    对于一元多项式来说，它满足一个性质：给定一组 d+1 个点 $(x_i,y_i)$，其中任意两个点都不同，存在一条且仅存在一条通过这d+1个点的d阶多项式。那么prover若要将多项式 $f_i(x_i)$ 传递给verifier，只需要将 $f_i(x_i)$ 上 d+1 个点的取值传给 verifier即可（d为 $f_i(x_i)$的阶数）。
    
    这里由于prover在每一轮给verifier发送的proof 就是一元多项式，也就是d+1 个点的取值，因此 sumcheck 协议的proof size 就是 O(dl)。
    
2. verifier如何获取 $f(r_1,r_2,...,r_l)$ 进行验证？
    
    Sumcheck中没有给出明确的计算方法，而是假设 verifier 可以通过oracle直接获取到（这里verifier可能可以直接计算出来，也可能不行）。因而 verifier 能否获得正确的 $f(r_1,r_2,...,r_l)$ 是能否采用sumcheck 来进行求和验证的关键。

## 4 多线性扩展（Multilinear Extension）

通过前面的介绍我们已经知道了，Sumcheck 是用来验证有限域 $\mathbb{F}$ 上的有l个输入项的多项式 f 在各个变量取值都为二进制数时的取值的和的。但如果我们求和的值并非不少多项式在变量为二进制数时对应计算结果，是否也可以使用Sumcheck来验证呢？

这里我们就可以借用一个工具——multilinear extension。如果我们要对一组值进行求和验证时，我们可以将这组值转换成有限域 $\mathbb{F}$ 上某个多项式 f 在变量为二进制数时的计算值。

在介绍multilinear Extension之前，先来看另一个概念：

> Low-degree extensions(LDEs). Suppose g: $\{0,1\}^m \rightarrow \mathbb{F}$ is a function that maps m-bit elements into an element of $\mathbb{F}$. A polynomial extension of g is a low-degree m-variate polynomial $\tilde{g}(\cdot)$ such that $\tilde{g}(x) = g(x)$ for all $x \in \{0,1\}^m$.

**低阶扩展（Low-degree extension** 将函数 $g(x): \{0,1\}^m \rightarrow \mathbb{F}$ 变换成 $\tilde{g}(x): \mathbb{F}^m \rightarrow \mathbb{F}$，并且使得对于任意一个 $x\in \{0,1\}^m$，都满足 $\tilde{g}(x) = g(x)$。在这两个多项式中，x的数据类型稍有不同，g(x) 中 x 是用二进制数组表示的一个数字，$\tilde{g}(x)$ 中 x 表示将长度为m的二进制数组的映射到有限域的m维数组上，每一位的值保持不变依然是0或者1。

在低阶扩展多项式中，如果每一项的阶数都为1，那么这个多项式就是多线性扩展（Multilinear Extension 或MLE）多项式。对于任意一个多项式，它的MLE多项式是唯一的，可以用如下的方式来定义MLE多项式：

$$
\tilde{g}(x_1,...,x_m) = \sum_{e\in\{0,1\}^m}g(e)\cdot \prod_{i=1}^m(x_i \cdot e_i+(1-x_i)(1-e_i)) 
= \sum_{e \in \{0,1\}^m} g(e) \cdot \tilde{eq}(x,e) 
= <(g(0),...,g(2^m-1),(\tilde{eq}(x,0),...,\tilde{eq}(x,2^m-1))>
$$

其中 $\tilde{eq}(x,e)=\prod_{i=1}^m(x_i \cdot e_i+(1-x_i)(1-e_i))$, 它是多项式 $eq(x,e)$的 MLE，这里$eq(x,e)$的定义为：

$$
eq(x, e)= 
\\begin{equation}
\left \\{
	\begin{array}{lr}
	 1, if \ x=e  \\\\  0, otherwise
	\end{array}
\right.
\\end{equation}
$$

我们可以将任意的一元多项式变换为MLE，也可以直接将一组数值用MLE多项式表示出来。当把一组值的求和变成一个MLE多项式，就有可能使用Sumcheck来做验证了。
