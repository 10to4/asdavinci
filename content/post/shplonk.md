---
title: "SHPLONK"
date: 2022-12-10T00:00:00+08:00
categories: ["zkp"]
---

在Plonk协议中，多项式承诺是非常重要的一个环节，协议中的多项式承诺实现了多个多项式在两个点上的open。但是在实际的应用场景下，有可能会出现需要多个多项式在多个不同点处open的场景，如果能通过改进多项式承诺协议来使得plonk支持多个多项式在不同的子集点处的open。

若 $ z \in S $，那么 $Z_S(x) = \prod_{z \in S} (X-z)$。

假设有一个数据集$T = \{z_1, z_2, ..., z_t\}$，并且它的子集$S_1,S_2,...,S_k \subset T$。如果有k个多项式$f_1(x), f_2(x),...,f_k(x)$，它们的承诺分别为$cm_1, cm_2,...,cm_k$，要分别在$S_1,S_2,...,S_k $上open。

首先若要使用KZG10对一个多项式做验证的话，那么

* prover计算多项式
  $$
  h(x) = \frac{f_i(x) - r_i(x)}{Z_{S_i}(x)}
  $$


  其中$r_i(x)$ 是一个$|s_i|$ 阶的多项式，且满足再$f_i(x) = r_i(x), x \in S_i$。

  再使用srs参数计算证明$W =[h(x)]_1 = g^{h(x)}$，发送给verifier。

* Verifier 拿到证明W后进行验证，这里需要两次pairing运算。
 
  $$
  e(cm_i - [r_i(x)]_1, g_2) = e(W,g_2 ^ {k})
  $$

  其中 $k = Z_{S_i}(x)$

如果要一次性得对这k个多项式进行验证，只需要引入一个随机数

* verifier 生成一个随机数$\gamma$，并发送给prover

* prover 拿着随机数将k组多项式的证明计算
  $$
  h_i(x) =   \frac{f_i(x) - r_i(x)}{Z_{S_i}(x)}
  $$


  再使用srs参数计算证明$W_i =[h(x)_i]_1 = g^{h_i(x)}$，发送给verifier。

* verifier拿到证明W后进行验证

  $  \prod_{i \in [k]} e(\gamma^{i-1} (cm_i-[r_i(x)]_1),  g_2) $ 

  $ =  \prod_{i \in [k]}e(W_i,g_2^{Z_{S_i}(x)})$

这个合并尚还不能降低计算量，需要2k次pairing运算。SHPlonk 中在验证步骤做了一个优化

* 首先verifier先计算一组

  $$
  Z_i = g_2^{Z_{T/S_i}(x)},i \in [k]
  $$

  其中$Z_{T/S_i}(x) = \frac{Z_T(x)}{Z_{S_i(x)}}$

* prover 拿着随机数将k组多项式的证明计算合并起来

  $$
  h(x) = \sum_{i \in [k]}\gamma^{i-1} \cdot  \frac{f_i(x) - r_i(x)}{Z_{S_i}(x)} \\ = \sum_{i \in [k]}\gamma^{i-1} \cdot  \frac{(f_i(x) - r_i(x)) \cdot Z_{T/S_i}(x)}{Z_{T}(x)} \\ =   \frac{\sum_{i \in [k]}\gamma^{i-1} \cdot(f_i(x) - r_i(x)) \cdot Z_{T/S_i}(x)}{Z_{T}(x)} 
  $$

  再使用srs参数计算证明$W =[h(x)]_1 = g^{h(x)}$，发送给verifier。
  
* 那么验证步骤就被改造为

  $$
  \prod_{i \in [k]} e(\gamma^{i-1} (cm_i-[r_i(x)]_1),  Z_i) =e(W,g_2^{Z_T(x)})
  $$

  这样pairing运算的数量就降低到了k+1次。

这里右侧已经不能合并了，未来进一步优化计算量

* 首先令
  $$
  f_Z(X) = \sum_{i \in [k]}\gamma^{i-1} \cdot Z_{T/S_i}(X)\cdot(f_i(X) - r_i(X))
  $$
  那么
  $$
  h(X) = \frac{f_Z(X)}{Z_T(X)}
  $$
  再使用srs参数计算证明$W =[h(x)]_1 = g^{h(x)}$，发送给verifier。

* verifier 产生一个随机数$z$，并发送给prover。

* 再继续变换多项式的形式
  $$
  L(x) = f_Z(X) - h(X)\cdot Z_T(z)
  $$
  其中
  $$
  f_Z(X) = \sum_{i \in [k]}\gamma^{i-1} \cdot Z_{T/S_i}(z)\cdot(f_i(X) - r_i(z))
  $$
  显然$L(z) = 0$。所以L(x)必然能被$(x-z)$整除。

  所以prover可以继续构造证明
  $$
  h'(x) = \frac{L(x)}{x-z}
  $$
  再使用srs参数计算证明$W' =[h'(x)]_1 = g^{h'(x)}$，发送给verifier。这里就相当于把再S个点处opening的问题转换到了在一个点open的问题上了。

* Verifier 首先去计算

  $F = [f_Z(x)]1 = \sum_{i \in [k]}\gamma^{i-1} \cdot Z_{T/S_i(z)}(cm_i - [r_i(z)]_1) - Z_T(z)\cdot W$

  再继续验证
  $$
  e(F,[1]_2) = e(W',[x-z]_2)
  $$
  只剩下两个pairing 和k+1次的椭圆曲线上的乘法运算，这远远小于k+1次的计算消耗。
  
  原本的方案中，右侧的pairing已经不能合并了，所以这里优化的关键就是将$Z_{T/S}(x)$ 和 $Z_{Y}(x)$ 的计算变换成了在一个已知点处的计算，即$Z_{T/S}(z)$ 和 $Z_{Y}(z)$，因为$Z_{T/S}(x)$ 和 $Z_{Y}(x)$ 多项式是公开的，z是公开的，那么计算就可以只能在有限域上进行了，而不必管srs，因此原本pairing运算中在g2上算的这两项都可以直接乘到g1上了，这样也就合并了原来的多个pairing。 

fflonk也是一种新的优化方案，它在SHPlonk的基础之上设计的优化方案，它继续沿用了SHPlonk的验证步骤，并结合了一种新的类似FFT运算的方法。fflonk的验证步骤中仅需要3次椭圆曲线上的运算，及2次pairing，与SHPlonk相比，又进一步优化了计算步骤。