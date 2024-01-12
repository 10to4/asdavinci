---
title: 【笔记】Groth16 Ceremony Introduction
date: 2022-04-29T21:10:00+07:00
categories:
  - groth16
  - zkp
  - setup
---


在 zkSNARK 协议中，经常会出现  Trusted Setup 步骤。这一步骤会产生一组基于电路的公共参数，被称为CRS（Common Reference String）。这些公共参数包括用于后续的证明和验证的证明密钥（pk）和验证密钥（vk）。Trusted Setup 过程中产生的一些随机值，被称为"toxic waste"。这些值是不能公开的，因为一旦被攻击者利用，它可以用来伪造的证明。所以产生 CRS 步骤必须由受信任方来完成，并且在创建 CRS 后立即销毁掉这些随机值。

通常我们很少考虑如何安全得生成CRS，只是简单地假设可以找到可信任的一方去生成CRS。但随着zkSNARK的使用日益广泛，尤其是引入区块链领域后，真正的货币价值受到威胁，而此时找到一个可以被广泛接受并值得信赖的一方几乎是不可能的。

Ben-Sasson等人[BCG+15]和随后一些研究中提出使用多方计算来生成CRS的方法。多个参与方共同参与生成CRS，这些参与方迭代构建CRS提供了随机性，随后诚实的参与者立即删除掉"toxic waste"。只要有一个贡献者是诚实的，那么就无法来伪造证明。但若所有参与者都联合起来作弊，依然可能会利用CRS的底层数学结构来创建不可靠的证明。

但是如果有很多无关联的独立参与者，那么所有参与者联合起来作弊的可能性就会降低到可以忽略不计。因此为了要最大化诚实和独立贡献者的数量，使用公开的仪式（ceremony）执行受信任的设置，其中多个参与者轮流执行计算。若社区用户对任何参与方都不信任，也可以亲自参与进来，并丢弃掉自己的 "toxic waste"。Zcash 项目在 2017 年就举行过这样的仪式。



### 1. ceremony 步骤

2017年，Bowe等人在论文[[BGM17](https://eprint.iacr.org/2017/1050.pdf)]中提出了一种新的MPC协议——MMORPG的协议，专门用于基于线性配对的zk-SNARKs，如Groth16。它解决了先前方案的一些缺点。它将Trusted Setup 分为两个阶段。第一阶段称为"Powers of Tau"，它主要是产生通用的设置参数，可用于方案的所有电路。第二阶段是结合特定的电路，将 Tau 次幂的输出转换为特定关系的 CRS。

前面提到，第一阶段产生的是通用的参数，它与特定的电路无关。因此在电路约束大小满足的前提下，一次仪式可以作为任意 Trusted Setup的第一阶段。这样做的一个好处是整个社区运行只要运行一个第一阶段仪式，就可以一直用下去，减轻了所有团队的负担。

第二阶段是将第一阶段产生的参数与具体项目的电路结合，产生相应的CRS。同样地，这一阶段也是有多方参与，只要有一方丢弃掉"toxic waste"，就可以认为这一阶段产生的参数是可信的。不过这一阶段就需要由各个项目方自己来执行了。



### 2. 开源工具

前面提到，为了减轻项目方的负担，社区中有一些开源的 Trusted Setup Ceremony 源码和可以参与的工具。

我们比较熟知的两个开源代码库有：[phase2-bn254](https://github.com/kobigurk/phase2-bn254) 和 [snarkjs](https://github.com/iden3/snarkjs) 。

[phase2-bn254](https://github.com/kobigurk/phase2-bn254) 是一个Rust语言实现的基于bn254曲线的开源库，loopring 的 Trusted Setup Ceremony 就是fork的这个仓库。

 [snarkjs](https://github.com/iden3/snarkjs) 是js版本的snark实现，它方便于用于网页上运行，运行时间会比较慢一些，但是相对来说比较容易做到用户友好。

两个开源库的使用方法可以参考开源库中的说明文档。

对于第一阶段，社区使用最广泛的一个开源仪式就是 **[perpetualpowersoftau](https://github.com/weijiekoh/perpetualpowersoftau)**，这个仪式是永久性的，参与者数量没有限制。任何zk-SNARK项目也都可以选择仪式的任何点来开始他们特定于电路的第二阶段。Loopring，Tornado Cash等知名项目，第一阶段用的都是 [perpetualpowersoftau](https://github.com/weijiekoh/perpetualpowersoftau) 产生的结果。通过调用 **[phase2-bn254](https://github.com/kobigurk/phase2-bn254)** 和 **[snarkjs](https://github.com/iden3/snarkjs)**  任何人都可以参与贡献。它的参数能支持的约束上限为`2 ^ 28` (约2.68亿条)，如果实际的使用的电路约束大小超过这个范围，则不能直接使用这个结果。

Trusted Setup的第二阶段仪式，各个项目方可以直接使用开源库 **[phase2-bn254](https://github.com/kobigurk/phase2-bn254)** 和 **[snarkjs](https://github.com/iden3/snarkjs)** 的生成算法来完成。但是这些开源库的调用方法并不简单，需要参与者有一定的技术背景。所以为了更多人参与进来做贡献，项目方通常会基于开源库开发一个用户友好的网页或者工具，以降低参与者的参与门槛。社区也有一些开源的工具，比如 **[Trusted Setup UI](https://medium.com/privacy-scaling-explorations/trusted-setup-ui-update-f8f95fc17a37)**，它是一套基于浏览器的 Trusted Setup Ceremony 组件。通过UI界面，参与者不需要了解Trusted Setup Ceremony 背后的原理和开源库信息，只需傻瓜式得在网页上点击按钮就可以参与贡献。zkopru 项目就是用的这个工具举行的仪式。

### 3. 原理介绍

#### 3.1 离散对数问题与"toxic waste"

离散对数问题的定义：“given a group *G*, a generator *g* of the group and an element *h* of *G*, to find the discrete logarithm to the base *g* of *h* in the group *G*（给定一个群G，群的生成元 g 和G上的元素h，以找到群 G 中以 g 为底的 h 的离散对数）”。离散对数问题并非全部都很难计算，取决于群。因此基于离散对数的加密系统来说， p 必须非常大（通常至少1024位）才能使加密系统安全。也就是说， zkSANRKs 所使用的曲线，必须保证离散对数问题很难被计算出来，即无法根据 $g^h$ 计算出 h。

指数运算满足这样一个原理：$(g^{r_1})^{r_2} = g^{r_1 \cdot r_2}$，因此当第一个参与者计算出一组值$\{g^{r_1}, g^{r_1^2}...\} $后，第二个参与者可以继续在此基础上计算 $\{g^{r_1 \cdot r_2}, g^{(r_1\cdot r_2)^2 }...\}$，依次类推最终t个参与者可以得到一组结果$\{g^{r_1 \cdot r_2...r_t}, g^{(r_1\cdot r_2...r_t)^2 }...\}$。我们无法通过这组结果计算出$r_1 \cdot r_2...r_t$，只有参与者知道各自提交的r。这组 $r_1, r_2...r_t$ 就是"toxic waste"，若参与者联合起来，就可以计算出$r_1 \cdot r_2...r_t$，但只要有一个人丢弃掉这个值，就无法再计算出$r_1 \cdot r_2...r_t$。

#### 3.2 Groth16 的 CRS 生成原理

Groth16 和 Plonk是使用最多的，需要进行Trusted Setup 的 zkSNAKRs 协议，这里我们以 Groth16 为例，详细介绍一下 Trusted Setup Ceremony 的原理步骤。

##### 3.2.1 CRS参数

Groth16 的Trusted Setup 最终要生成一组 CRS 参数，它分为 proving key 和 verification key 两部分：

$vk: \{g_1^\alpha, g_1^\beta, g_2^\beta, g_1^\gamma, g_1^\delta, g_2^\delta, ic = \{g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\gamma}}\}_{i=0}^l \}$，

$pk:\{vk, h =  \{g_1^{\frac{(\tau^m-1)\tau^i}{\delta}}\}_{i=l}^m l = \{g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta}}\}_{i=l}^n, a_1 = \{g_1^{u_i(\tau)}\}_{i=1}^n, b_1 = \{g_1^{v_i(\tau)}\}_{i=1}^n, b_2 = \{g_2^{v_i(\tau)}\}_{i=1}^n \}$

其中，包含五个随机值 $\tau$，$\alpha$，$\beta$，$\gamma$，$\delta$。

##### 3.2.2 Trusted Setup计算步骤

在讨论 Trusted Setup Ceremony 步骤之前，我们先来看一下Groth16 的 Setup 常规计算步骤。

1. 产生随机值 $\tau$，$\alpha$，$\beta$，$\gamma$，$\delta$，计算 $g_1^\alpha, g_1^\beta, g_2^\beta, g_1^\gamma, g_1^\delta, g_2^\delta$。

2. 生成n阶 $\tau$ 数组 $\{\tau^1,...\tau^m\}$

3. 计算 $t(\tau)\cdot \tau^i = (\tau^m-1)\cdot \tau^i, h(i) = g_1^{\frac{(\tau^m-1)\cdot \tau^i}{\delta}}$

4. 使用拉格朗日插值法计算 $u_i(\tau), v_i(\tau),\omega_i(\tau)$

    $u_i(x) = \sum l_{ij}(x)a_{ij}, \ v_i(x) = \sum r_{ij}(x)b_{ij}, \ \omega_i(x) = \sum o_{ij}(x)c_{ij}$

   以$u_i(x) = \sum l_{ij}(x)a_{ij}$ 为例，i 表示第 i 个 input， j 表示第 j 个约束，$a_{ij}$ 表示第 i 个 input在第 j 个约束中的系数，这里$l_{ij}(x)$的取值需要满足 $u_i(x_0) = a_{i0}, u_i(x_1) = a_{10},...,u_i(x_n) = a_{n0}$，也就是说需要满足 $l_{ij}(x_j) = 1, l_{ij}(x_k) = 0 (j!=k)$。同时这里也可以看出 $l_{ij} = r_{ij} = o_{ij}$。

   因此我们可以简写：

    $u_i(x) = \sum l_{j}(x)a_{ij}, \ v_i(x) = \sum l_{j}(x)b_{ij}, \ \omega_i(x) = \sum l_{j}(x)c_{ij}$

   为了方便进行fft计算，这里$[x_0,x_1,...,x_{n-1}]$ 取值为 $[\omega^0,\omega^1,...,\omega^{n-1}]$，在这组取值下就可以令 $l_j(x) =\frac{1}{n}\sum_{i=0}^{n-1} x^i\omega^{-ji} $。

   即 $l_{j}(x_k) = \frac{1}{n}\sum \omega^{ik}\omega^{-ji} = \frac{1}{n}\sum \omega^{i(k-j)}$，当 $k=j,l_{j}(x_k) = \frac{1}{n}\sum \omega^{i\cdot0} = \frac{n}{n} = 1$， 当 $k!=j, l_{j}(x_k) = \frac{1}{n}\sum \omega^{i\cdot(k-j)} = \frac{1}{n}\sum  \omega^{i} = 0$。

   接下来令 $x = \tau$，那么 $l_j(x) = l_j(\tau) = \frac{1}{n}\sum_{i=0}^{n-1}\tau^i\omega^{-ij}$。

   假设有一个多项式 $f(x)$，取 $x = [\omega^0,\omega^1,...,\omega^{n-1}]$后的计算结果为  $ [\tau^0,\tau^1,...,\tau^{n-1}]$，即$f(\omega^i) = \tau^i$。多项式 $f(x)$的拉格朗日系数$[a_0,a_1,...a_{n-1}]$的表示方法是 $a_j = \frac{1}{n}\sum f(x_i)\omega^{-ij} = \frac{1}{n}\sum \tau^i \omega^{-ij}$，这组值刚好与$l_j(\tau)$ 一致，因此，我们只需要使用反向fft去计算多项式 $f(x)$ 的拉格朗日系数就可以计算出$l_j(\tau)$。

   总结一下，计算 $u_i(\tau), v_i(\tau),\omega_i(\tau)$分为两步骤

   * 计算系数 $l_j(\tau)$，这组值与电路参数无关
   * 带入电路参数 $a_{ij},b_{ij},c_{ij}$ 计算 $u_i(\tau), v_i(\tau),\omega_i(\tau)$

5. 计算 $a_1 = \{g_1^{u_i(\tau)}\}_{i=1}^n, b_1 = \{g_1^{v_i(\tau)}\}_{i=1}^n, b_2 = \{g_2^{v_i(\tau)}\}_{i=1}^n, ic = \{g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\gamma}}\}_{i=0}^l ,l = \{g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta}}\}_{i=l}^n $

#### 3.3  Trusted Setup Ceremony 步骤

##### 第一阶段

第一阶段称为 "Powers of Tau"，计算一组字段` {tau_power_g1, tau_power_g2, alpha_tau_power_g1, beta_tau_power_g1, beta_g2 }`,其中

tau_power_g1 = $\{g_1^{\tau^0},g_1^{\tau^1}... \}$
tau_power_g2 = $\{g_2^{\tau^0},g_2^{\tau^1}... \}$
alpha_tau_power_g1 = $\{g_1^{\alpha\tau^0},g_1^{\alpha\tau^1}... \}$
beta_tau_power_g1 = $\{g_1^{\beta\tau^0},g_1^{\beta\tau^1}... \}$
beta_g2 = $g_2^\beta$

1. 初始化数组

   tau_power_g1 = $\{g_1,g_1... \}$
   tau_power_g2 = $\{g_2,g_2... \}$
   alpha_tau_power_g1 = $\{g_1,g_1... \}$
   beta_tau_power_g1 = $\{g_1,g_1... \}$
   beta_g2 = $g_2$

2. 第一个参与者做贡献

   * 读取初始化参数数组

   * 第一个贡献者计算一组key, 
      private Key：$\{ \tau_1,\alpha_1, \beta_1 \}$，
     verify key: $\{ (g_1^{s_{\tau_1}},g_1^{s_{\tau_1}\tau_1}), (g_1^{s_{\alpha_1}},g_1^{s_{\alpha_1}\alpha_1} ),(g_1^{s_{\beta_1}},g_1^{s_{\beta_1}\beta_1}), g_2^{s_{\tau_1}'\tau_1}, g_2^{s_{\alpha_1}'\alpha_1}, g_2^{s_{\beta_1}'\beta_1}\}$

   * 第一个贡献者带入 private key 计算出一组新的参数

     tau_power_g1 = $\{g_1^{\tau_1^0},g_1^{\tau_1^1}... \}$
     tau_power_g2 = $\{g_2^{\tau_1^0},g_2^{\tau_1^1}... \}$
     alpha_tau_power_g1 = $\{g_1^{\alpha_1\tau_1^0},g_1^{\alpha_1\tau_1^1}... \}$
     beta_tau_power_g1 = $\{g_1^{\beta_1\tau_1^0},g_1^{\beta_1\tau_1^1}... \}$
     beta_g2 = $g_2^{\beta_1} $

3. 第二个参与者做贡献

   * 读取第一个参与者产生的参数

   * 第二个贡献者继续产生一组key，
     private key: $\{ \tau_2,\alpha_2, \beta_2 \}$，
     verify key: $\{ (g_1^{s_{\tau_2}},g_1^{s_{\tau_2}\tau_2}), (g_1^{s_{\alpha_2}},g_2^{s_{\alpha_2}\alpha_2} ),(g_1^{s_{\beta_2}},g_1^{s_{\beta_2}\beta_2}), g_2^{s_{\tau_2}'\tau_2}, g_2^{s_{\alpha_2}'\alpha_2}, g_2^{s_{\beta_2}'\beta_2}\}$

   * 第二个贡献者带入 private key 计算出一组新的参数

     tau_power_g1 = $\{g_1^{(\tau_1\tau_2)^0},g_1^{(\tau_1\tau_2)^1}... \}$
     tau_power_g2 = $\{g_2^{(\tau_1\tau_2)^0},g_2^{(\tau_1\tau_2)^1}... \}$
     alpha_tau_power_g1 = $\{g_1^{\alpha_1\alpha_2(\tau_1\tau_2)^0},g_1^{\alpha_1\alpha_2(\tau_1\tau_2)^1}... \}$
     beta_tau_power_g1 = $\{g_1^{\beta_1\beta_2(\tau_1\tau_2)^0},g_1^{\beta_1\beta_2(\tau_1\tau_2)^1}... \}$
     beta_g2 = $g_2^{\beta_1\beta_2} $

4. 第三个参与者做贡献

   * 读取第二个参与者产生的参数

   * 第三个贡献者继续产生一组key，
     private key: $\{ \tau_3,\alpha_3, \beta_3 \}$，
     verify key: $\{ (g_1^{s_{\tau_3}},g_1^{s_{\tau_3}\tau_3}), (g_1^{s_{\alpha_3}},g_2^{s_{\alpha_3}\alpha_3} ),(g_1^{s_{\beta_3}},g_1^{s_{\beta_3}\beta_3}), g_2^{s_{\tau_3}'\tau_3}, g_2^{s_{\alpha_3}'\alpha_3}, g_2^{s_{\beta_3}'\beta_3}\}$

   * 第三个贡献者带入 private key 计算出一组新的参数

     tau_power_g1 = $\{g_1^{(\tau_1\tau_2\tau_3)^0},g_1^{(\tau_1\tau_2\tau_3)^1}... \}$
     tau_power_g2 = $\{g_2^{(\tau_1\tau_2\tau_3)^0},g_2^{(\tau_1\tau_2\tau_3)^1}... \}$
     alpha_tau_power_g1 = $\{g_1^{\alpha_1\alpha_2\alpha_3(\tau_1\tau_2\tau_3)^0},g_1^{\alpha_1\alpha_2\alpha_3(\tau_1\tau_2\tau_3)^1}... \}$
     beta_tau_power_g1 = $\{g_1^{\beta_1\beta_2\beta_3(\tau_1\tau_2\tau_3)^0},g_1^{\beta_1\beta_2\beta_3(\tau_1\tau_2\tau_3)^1}... \}$
     beta_g2 = $g_2^{\beta_1\beta_2\beta_3} $

5. 依次类推，后续参与者可以持续参与贡献...

6. 第i个参与者使用beacon随机值做贡献

   每个贡献者都要先产生一组随机的private key，而为了保证这些随机值是可信不可预测的，通常会引入一组beacon随机值来做贡献，这样才算完成了一个阶段的Trusted Setup。

   beacon随机值是来源于一些公共事件的随机结果，在某个时间之前是不可知的。它可以用一个延迟的哈希函数（例如，SHA256的2^ 40次迭代），在一些高熵和公开可用的数据上进行计算。数据来源可以是未来某个日期股票市场的收盘价，彩票开奖结果，或一个或多个区块链中特定高度的区块哈希值等等。

   * 读取第 i-1 个共享者产生的参数

   * 第i个贡献者使用beacon随机值产生一组key，
     private key: $\{ \tau_i,\alpha_i, \beta_i \}$，
     verify key: $\{ (g_1^{s_{\tau_i}},g_1^{s_{\tau_i}\tau_i}), (g_1^{s_{\alpha_i}},g_2^{s_{\alpha_i}\alpha_i} ),(g_1^{s_{\beta_i}},g_1^{s_{\beta_i}\beta_i}), g_2^{s_{\tau_i}'\tau_i}, g_2^{s_{\alpha_i}'\alpha_i}, g_2^{s_{\beta_i}'\beta_i}\}$

   * 第i个贡献者带入 private key 计算出一组新的参数

     tau_power_g1 = $\{g_1^{(\tau_1\tau_2\tau_3...\tau_i)^0},g_1^{(\tau_1\tau_2\tau_3...\tau_i)^1}... \}$
     tau_power_g2 = $\{g_2^{(\tau_1\tau_2\tau_3...\tau_i)^0},g_2^{(\tau_1\tau_2\tau_3...\tau_i)^1}... \}$
     alpha_tau_power_g1 = $\{g_1^{\alpha_1\alpha_2\alpha_3...\alpha_i(\tau_1\tau_2\tau_3...\tau_i)^0},g_1^{\alpha_1\alpha_2\alpha_3...\alpha_i(\tau_1\tau_2\tau_3...\tau_i)^1}... \}$
     beta_tau_power_g1 = $\{g_1^{\beta_1\beta_2\beta_3...\beta_i(\tau_1\tau_2\tau_3...\tau_i)^0},g_1^{\beta_1\beta_2\beta_3...\beta_i(\tau_1\tau_2\tau_3...\tau_i)^1}... \}$
     beta_g2 = $g_2^{\beta_1\beta_2\beta_3...\beta_i} $

7. 对参与者的贡献进行验证

   带入参与者的verify key进行验证，可以验证参与者是否按照正确的步骤执行

8. 准备第二阶段

   第二阶段的最终目的是为了生成一组与特定电路相关的 CRS 参数，那么在进入第二阶段之前，需要先将第一阶段产生的参数做一个转换，主要是计算拉格朗日系数（即3.2.2小节中的 $l_j(\tau)$）。

   即根据 tau_power_g1 $=\{g_1^{\tau^0},g_1^{\tau^1}... \}$，tau_power_g2 =  $\{g_1^{\tau^0},g_1^{\tau^1}... \}$，
   alpha_tau_power_g1 =  $\{g_1^{\alpha\tau^0},g_1^{\alpha\tau^1}... \}$，beta_tau_power_g1=  $\{g_1^{\beta \tau^0},g_1^{\beta\tau^1}... \}$ 
   计算出 
   $ g_{1,cof}=\{g_1^{l_j(\tau)}\}_{j=0}^{n-1} $, 

   $g_{2,cof}=\{g_2^{l_j(\tau)}\}_{j=0}^{n-1}$, 

   $g_{1,cof,\alpha}=\{g_1^{\alpha\cdot l_j(\tau)}\}_{j=0}^{n-1}$,

   $g_{1,cof,\beta}= \{g_1^{\beta \cdot l_j(\tau)}\}_{j=0}^{n-1}$,

   及计算 
   $h(i) = g_1^{(\tau^m-1)\cdot \tau^i }$。

   最终得到一组参数 {alpha_tau_power_g1, beta_tau_power_g1,beta_g2，$g_{1,cof}, g_{2,cof},g_{1,cof,\alpha},g_{1,cof,\beta},h(i)$}

##### 第二阶段

第二阶段就是结合电路去计算完整的CRS。 $\delta$和 $\gamma$ 是这一阶段的"toxic waste"。在Groth16协议中， $\delta$和 $\gamma$ 只要有一个值是随机值就可以了，所以在仪式中，通常会直接默认 $\gamma$ 为 1，只有一个随机值 $\delta$。

1. 初始化数组

   这一步主要就是开始构建完整的CRS参数，首先来看一下 CRS 参数的完整内容：

   $vk: \{g_1^\alpha, g_1^\beta, g_2^\beta, g_1^\gamma, g_1^\delta, g_2^\delta, ic = \{ g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\gamma}}\}_{i=0}^l \}$，

   $pk: \{vk, h =  \{g_1^{\frac{(\tau^m-1)\tau^i}{\delta}}\}_{i=l}^m l = \{g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta}}\}_{i=l}^n, a_1 = \{g_1^{u_i(\tau)}\}_{i=1}^n, b_1 = \{g_1^{v_i(\tau)}\}_{i=1}^n, b_2 = \{g_2^{v_i(\tau)}\}_{i=1}^n \}$

   其中 
   $g_1^\alpha =$alpha_tau_power_g1[0]$, 
   $g_1^\beta$= beta_tau_power_g1[0]$, 
   $g_2^\beta$ = beta_g2 $。

   这里我们先令 $\gamma = 1, \delta = 1$，那么 $g_1^\gamma= g_1^1$ ,$ g_1^\delta = g_1^1$, $g_2^\delta = g_2^1$。

   前文提到 $u_i(\tau) = \sum l_{j}(\tau)a_{ij}, \ v_i(\tau) = \sum l_{j}(\tau)b_{ij}, \ \omega_i(\tau) = \sum l_{j}(\tau)c_{ij}$，所以带入电路参数和 $g_{1,cof}, g_{2,cof},g_{1,cof,\alpha},g_{1,cof,\beta}$ 就可以计算出 $a_1,b_1,b_2,ic,l$。

   $h$前面已经计算出来了，因此这里我们就可以构造出一个  $\gamma = 1, \delta = 1$ 版本的 CRS。

2. 第一个参与者做贡献

   因为第二阶段的"toxic waste"仅有一个随机数 $\delta$，所以其实第二阶段的贡献去更新的字段就是包含$\delta$ 的项 $\{g_1^\delta, g_2^\delta,h,l \}$。每一个参与者贡献产生的参数都是一组完整的CRS参数。

   * 读取初始化参数数组

   * 第一个贡献者计算一组key,

     private key: $\delta_1$

     public key: $\{g_1^{s_1},  g_1^{s_1\delta_1}, g_2^{s_1'\delta_1},g_1^{\delta_1}\}$

   * 第一个贡献者带入 private key 更新参数

     $g_1^\delta = g_1^{\delta_1}$

     $g_2^\delta = g_2^{\delta_1}$

     $h =  \{(g_1^{(\tau^m-1)\tau^i})^{\frac{1}{\delta_1}}\}_{i=l}^m $

     $l = \{(g_1^{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)})^{\frac{1}{\delta_1}}\}_{i=l}^n$

3. 第二个参与者做贡献

   * 读取第一个参与者产生的参数

   * 第二个贡献者计算一组key,

     private key: $\delta_2$

     public key: $\{g_1^{s_2},  g_1^{s_2\delta_2}, g_2^{s_2'\delta_2},g_1^{\delta_1\delta_2}\}$

   * 第一个贡献者带入 private key 更新参数

     $g_1^\delta = g_1^{\delta_1\delta_2}$

     $g_2^\delta = g_2^{\delta_1\delta_2}$

     $h =  \{(g_1^{\frac{(\tau^m-1)\tau^i}{\delta_1}})^{\frac{1}{\delta_2}}\}_{i=l}^m $

     $l = \{(g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta_1}})^{\frac{1}{\delta_2}}\}_{i=l}^n$

4. 第三个参与者做贡献

   * 读取第二个参与者产生的参数

   * 第三个贡献者计算一组key,

     private key: $\delta_3$

     public key: $\{g_1^{s_3},  g_1^{s_3\delta_3}, g_2^{s_3'\delta_3},g_1^{\delta_1\delta_2\delta_3}\}$

   * 第三个贡献者带入 private key 更新参数

     $g_1^\delta = g_1^{\delta_1\delta_2\delta_3}$

     $g_2^\delta = g_2^{\delta_1\delta_2\delta_3}$

     $h =  \{(g_1^{\frac{(\tau^m-1)\tau^i}{\delta_1\delta_2}})^{\frac{1}{\delta_3}}\}_{i=l}^m $

     $l = \{(g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta_1\delta_2}})^{\frac{1}{\delta_3}}\}_{i=l}^n$

5. 依次类推，后续参与者可以持续参与贡献...

6. 第i个参与者使用beacon随机值做贡献

   与第一阶段相似，为了保证这些随机值是可信不可预测的，通常会引入beacon随机值来做贡献。

   * 读取第i-1个参与者产生的参数

   * 第i个贡献者计算一组key,

     private key: $\delta_i$

     public key: $\{g_1^{s_i},  g_1^{s_i\delta_i}, g_2^{s_i'\delta_i},g_1^{\delta_i\delta_i}\}$

   * 第三个贡献者带入 private key 更新参数

     $g_1^\delta = g_1^{\delta_1\delta_2\delta_3...\delta_i}$

     $g_2^\delta = g_2^{\delta_1\delta_2\delta_3...\delta_i}$

     $h =  \{(g_1^{\frac{(\tau^m-1)\tau^i}{\delta_1\delta_2...\delta_{{i-1}}}})^{\frac{1}{\delta_i}}\}_{i=l}^m $

     $l = \{(g_1^{\frac{\beta\cdot u_i(\tau)+\alpha\cdot v_i(\tau)+w_i(\tau)}{\delta_1\delta_2...\delta_{{i-1}}}})^{\frac{1}{\delta_i}}\}_{i=l}^n$

7. 对参与者的贡献进行验证

   带入参与者的verify key进行验证，可以验证参与者是否按照正确的步骤执行

   

### 4. 参考资料

https://medium.com/privacy-scaling-explorations/zkopru-trusted-setup-ceremony-f2824bfebb0f

https://medium.com/coinmonks/announcing-the-perpetual-powers-of-tau-ceremony-to-benefit-all-zk-snark-projects-c3da86af8377

https://github.com/Loopring/trusted_setup

https://eprint.iacr.org/2017/1050.pdf

https://eprint.iacr.org/2016/260.pdf

开源代码库：

snarkjs: https://github.com/iden3/snarkjs

phase2-bn254: https://github.com/kobigurk/phase2-bn254

Phase1 工具:
perpetualpowersoftau: https://github.com/weijiekoh/perpetualpowersoftau

Phase2工具:

Trusted Setup UI: https://medium.com/privacy-scaling-explorations/trusted-setup-ui-update-f8f95fc17a37

