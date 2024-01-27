---
title: "【笔记】Sangira: a Folding Scheme for Plonk"
date: 2024-01-27T23:00:00+08:00
categories:
  - zkp
  - folding
  - plonkish
---
论文链接：[sangira](https://github.com/geometryresearch/technical_notes/blob/main/sangria_folding_plonk.pdf)

在[Nova](https://eprint.iacr.org/2021/370.pdf) 论文中定义了一种基于Committed Relaxed R1CS的折叠方案，用来实现零知识的IVC（incrementally-verifiable computation）。这边方案可以用于折叠R1CS结构的instance-witness来做证明，参考[文章](../202401_relaxed_r1cs)。

Sangira是基于Plonk的电路结构 Plonkish 的变种实现的折叠方案。
## 1. Plonk 和 Relaxed Plonk
### 1.1 Plonk 电路结构
Plonk 的电路的结构为：
$C_{\mathcal{Q},i}(a, b, c) = (q_L)_i a_i + (q_R)_i b_i + (q_O)_i c_i + (q_M)_i a_i b_i + (q_C)_i$
其中$(q_L)_i, (q_R)_i, (q_O)_i, (q_M)_i, (q_C)_i$ 表示第i行的 selector 向量，我们用$\mathcal{Q} = (\mathbf{q}_L, \mathbf{q}_R, \mathbf{q}_O, \mathbf{q}_M, \mathbf{q}_C )$ 来表示所有的selector取值。
其中 $a_i, b_i, c_i$ 为第 i 行的左边，右边和输出值，这三行$(\mathbf{a}, \mathbf{b}, \mathbf{c})$ 都是 instance-witness $(\mathbf{X}, \mathbf{W})$（假设plonk电路的前n行为public input，后面s行的是witness）。
另外我们用 $\mathcal{S}$ 表示 copy constraint。
### 1.2 Relaxed Plonk 电路结构
Relaxed Plonk 电路结构为：
$C_{\mathcal{Q}, i}'(a, b, c) = u[(q_L)_i a_i + (q_R)_i b_i + (q_O)_i c_i] + (q_M)_i a_i b_i + u^2(q_C)_i + e_i$
与  Plonk 的电路结构稍有不同，多了两项$u, e_i$，其中 $e_i$ 是第i行的error term项。
Plonk 结构是 Relaxed Plonk 结构的一种特殊形式，其中 $u=1, e_i = 0$。

### 1.3 Committed Relaxed Plonk 电路结构
由于 $a_i, b_i, c_i$ 是witness，所以需要隐藏，error term项也需要隐藏
$\overline{W}_a = Com(pp_W, \vec{w_a}; r_a)$
$\overline{W}_b = Com(pp_W, \vec{w_b}; r_b)$
$\overline{W}_c = Com(pp_W, \vec{w_c}; r_c)$
$\overline{E} = Com(pp_E, \vec{e}; r_e)$
定义新的 instance-witness为 $(U, W)$:
$U = (\mathbf{X}, u, \overline{W}_a, \overline{W}_b, \overline{W}_c, \overline{E})$
$W = (\mathbf{W}, \mathbf{e}, r_a, r_b, r_c, r_e)$

## 2. Folding for Relaxed Plonk
给定一组Relaxed Plonk的selector，$\mathcal{Q} = \{\mathbf{q}_L, \mathbf{q}_R, \mathbf{q}_O, \mathbf{q}_M, \mathbf{q}_C \}$ ，和它对应的两组witness-inputs $(U_1, W_1)$ 和 $(U_2, W_2)$。我们想设计一种方案来化简这两组 witness-inputs 的检查成一组新的 witness-inputs $(U, W)$ 在相同Relaxed Plonk的selector上的检查，最直接的办法就是对这两组值做随机线性组合。

$(u'[q_L \circ a' + q_R \circ b' + q_O \circ c'] + q_M  \circ a'  \circ b' + u'^2 q_C + e') + r^2 \cdot (u''[q_L \circ a'' + q_R \circ b'' + q_O \circ c''] + q_M \circ a'' \circ b'' + u''^2 q_C + e'')$
$= (u'[q_L \circ a' + q_R \circ b' + q_O \circ c'] + r^2 \cdot (u''[q_L \circ a'' + q_R \circ b'' + q_O \circ c''])) + (q_M \circ a' \circ b' + r^2 \cdot (q_M \circ a'' \circ b'')) + (u'^2 q_C + r^2 \cdot u''^2 q_C) + (e_i' + r^2 \cdot e'')$
$=((u'+ r\cdot u'')(q_L \circ (a' + r \cdot a'') + q_R \circ (b' + r \cdot b'') + q_O \circ (c' + r \cdot c''))) - r \cdot u''(q_L \circ a' + q_R \circ b' + q_O \circ c') - r \cdot u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'')$
$+ q_M \circ (a' + r \cdot a'') \circ (b' + r \cdot  b'') -  r \cdot q_M \circ (a' \circ b'' + a'' \circ b') + (u'+ r \cdot u'')^2 q_C - 2 r \cdot u'u''  q_C + (e_i' + r^2 \cdot e'')$



若令 $a = a' + r a'', b= b'+ rb'', c = c' + rc''$

$e = e_i' + r^2 \cdot e'' - r \cdot (u''(q_L \circ a' + q_R \circ b' + q_O \circ c') +\cdot u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'') + q_M \circ (a' \circ b'' + a'' \circ b') + 2u'u''q_C)$

那么：

$(u'[q_L \circ a' + q_R \circ b' + q_O \circ c'] + q_M  \circ a'  \circ b' + u'^2 q_C + e') + r^2 \cdot (u''[q_L \circ a'' + q_R \circ b'' + q_O \circ c''] + q_M \circ a'' \circ b'' + u''^2 q_C + e'')$
$= u [q_L \circ a +  q_R \circ b + q_O \circ c] + q_M  \circ a  \circ b + u'^2 q_C + (e_i' + r^2 \cdot e'')$
$- r \cdot (u''(q_L \circ a' + q_R \circ b' + q_O \circ c') +\cdot u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'') + q_M \circ (a' \circ b'' + a'' \circ b') + 2u'u''q_C)$
$= u [q_L \circ a +  q_R \circ b + q_O \circ c] + q_M  \circ a  \circ b + u'^2 q_C + e$


## 3. Folding Scheme for Relaxed Plonk
下面来看一下完整的协议：

0.setup
 
   ${G}(1^\lambda) \rightarrow pp$：输出 $n,s \in \mathbb{F}$，及承诺的参数 $pp_W$ 和 $pp_E$，长度分别为 s 和 n+s+1。 
 
   $K(pp, (Q, S))  \rightarrow (pk, vk)$：输出pk 和 vk 
 
1.证明者（prover）拥有两组witness: $(\mathbf{W}', \mathbf{e}', r_a', r_b', r_c', r_e')$ 和$(\mathbf{W}'', \mathbf{e}'', r_a'', r_b'', r_c'', r_e'')$和对应的instance：$(\mathbf{X}', u', \overline{W}_a', \overline{W}_b', \overline{W}_c', \overline{E}')$ 和 $(\mathbf{X}'', u'', \overline{W}_a'', \overline{W}_b'', \overline{W}_c'', \overline{E}'')$ ，

验证者（verifier）获得这两组instance：$(\mathbf{X}', u', \overline{W}_a', \overline{W}_b', \overline{W}_c', \overline{E}')$ 和 $(\mathbf{X}'', u'', \overline{W}_a'', \overline{W}_b'', \overline{W}_c'', \overline{E}'')$ 

2.证明者（prover）产生随机数 $r_T$，并计算cross term T及其承诺，并将承诺发送给验证者（verifier）

$\mathbf{T} = u''(\mathbf{q}_L \circ \mathbf{a}' + \mathbf{q}_R \circ \mathbf{b}' + \mathbf{q}_O \circ \mathbf{c}') + u'(\mathbf{q}_L \circ \mathbf{a}'' + \mathbf{q}_R \circ \mathbf{b}'' + \mathbf{q}_O \circ \mathbf{c}'') + \mathbf{q}_M \circ (\mathbf{a}' \circ \mathbf{b}'' + \mathbf{a}'' \circ \mathbf{b}') + 2u'u'' \mathbf{q}_C$

$\overline{T} = Com(pp_E, \mathbf{T}; r_t)$

3.验证者（verifier）生成随机数 $r$

4.证明者（prover）和验证者（verifier）分别折叠两组instance为一组instance $U = (\mathbf{X}, u, \overline{W}_a, \overline{W}_b, \overline{W}_c, \overline{E})$

$\mathbf{X} = \mathbf{X}' + r \mathbf{X}''$

$\overline{W}_a = \overline{W}_a' + r \overline{W}_a''$

$\overline{W}_b = \overline{W}_b' + r \overline{W}_b''$

$\overline{W}_c = \overline{W}_c' + r \overline{W}_c''$

$\overline{E} = \overline{E}' - r \overline{T} + r^2 \overline{E}''$

5.证明者（prover）折叠两组witness为一组witness $W = (\mathbf{W}, e, r_a, r_b, r_c, r_e)$

$\mathbf{W} = \mathbf{W}' + r \mathbf{W}''$

$r_a = r_a' + r \cdot r_a''$

$r_b = r_b' + r \cdot r_b''$

$r_c = r_c' + r \cdot r_c''$

$\mathbf{e} = \mathbf{e}' -r \mathbf{T} + r^2 \mathbf{e}''$

$r_e = r_e' - r \cdot r_t + r^2 \cdot r_e''$

## 4. 2 阶 Custom Gates 的 Relaxed Plonk

对于有自定义门的Plonk来说，同样可以做折叠，以一个2阶的自定义门为例，我们来构造folding方案。
假设有一个2阶的门 $g(a, b, c)$  和它的 selector $q_G$: 
$G_i(a, b, c) = (q_G)\_i \cdot g(a_i, b_i, c_i)$
并将$g$拆成三部分，用$g_c, g_1, g_2$ 分别来表示常数项，一阶的项，二阶的项。

那么电路结构为：

$C_{\mathcal{Q}, i}'(a, b, c, u, e) = u[(q_L)\_i a_i + (q_R)\_i b_i + (q_O)\_i c_i + (q_G)\_i \cdot g_1(a_i, b_i, c_i)] + (q_M)\_i a_i b_i + (q_G)\_i \cdot g_2(a_i, b_i, c_i) + u^2(q_C)\_i + u^2 (q_G)\_i \cdot g_C + e_i$

两组witness-inputs $(U_1, W_1)$ 和 $(U_2, W_2)$ 折叠
$(u'[q_L a' + q_R b' + q_O c' + q_G \cdot g_1(a', b', c')] + q_M a' b' + q_G \cdot g_2(a', b', c') + u'^2 q_C + u'^2 q_G \cdot g_C + e')$
$+ r^2 (u''[q_L a'' + q_R b'' + q_O c'' + q_G \cdot g_1(a'', b'', c'')] + q_M a'' b'' + q_G \cdot g_2(a'', b'', c'') + u''^2 q_C + u''^2 q_G \cdot g_C + e'')$
$=(u(q_L \circ a + q_R \circ b + q_O \circ c + q_G \cdot g_1(a, b, c))) - r \cdot u''(q_L \circ a' + q_R \circ b' + q_O \circ c' + q_G \cdot g_1(a', b', c')) - r \cdot u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'' + q_G \cdot g_1(a'', b'', c''))$
$+ q_M \circ a \circ b -  r \cdot q_M \circ (a' \circ b'' + a'' \circ b') + q_G \cdot g_2(a, b, c) - r \cdot q_G \cdot g_{e}(a, b, c) + u^2 q_C + u^2 q_G \cdot g_C- 2 r \cdot u'u''  (q_C + q_G \cdot g_C) -r \cdot + (e_i' + r^2 \cdot e'')$

$=u[q_L a + q_R b + q_O c + q_G \cdot g_1(a, b, c)] + q_M a b + q_G \cdot g_2(a, b, c) + u^2 q_C + u^2 q_G \cdot g_C + (e_i' + r^2 \cdot e'')$

$- r \cdot (u''(q_L \circ a' + q_R \circ b' + q_O \circ c' + q_G \cdot g_1(a', b', c')) +u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'' + q_G \cdot g_1(a'', b'', c'')) + q_M \circ (a' \circ b'' + a'' \circ b') + q_G \cdot g_{e}(a, b, c)+ 2u'u''(q_C + q_G \cdot g_C))$
$=u[q_L a + q_R b + q_O c + q_G \cdot g_1(a, b, c)] + q_M a b + q_G \cdot g_2(a, b, c) + u^2 q_C + u^2 q_G \cdot g_C + e$

我们令 $e = (e_i' + r^2 \cdot e'') - r \cdot (u''(q_L \circ a' + q_R \circ b' + q_O \circ c' + q_G \cdot g_1(a', b', c')) + u' (q_L \circ a'' + q_R \circ b'' + q_O \circ c'' + q_G \cdot g_1(a'', b'', c'')) + q_M \circ (a' \circ b'' + a'' \circ b') + q_G \cdot g_{e}(a, b, c)+ 2u'u''(q_C + q_G \cdot g_C))$

其中 我们用$g_{e}(a, b, c)$来表示 $q_G \cdot g_2(a, b, c)$多出来的cross term项。 
