---
title: Fast amortized KZG proof Note
date: 2024-01-13T1:00:00+08:00
categories:
  - commitment
  - FFT
---


论文地址：https://eprint.iacr.org/2023/033.pdf
## 1. 托普利茨矩阵和循环矩阵
### 1.1 托普利茨矩阵
> In [linear algebra](https://en.wikipedia.org/wiki/Linear_algebra "Linear algebra"), a **Toeplitz matrix** or **diagonal-constant matrix**, named after [Otto Toeplitz](https://en.wikipedia.org/wiki/Otto_Toeplitz "Otto Toeplitz"), is a [matrix](https://en.wikipedia.org/wiki/Matrix_(mathematics) "Matrix (mathematics)") in which each descending diagonal from left to right is constant. A Toeplitz matrix is not necessarily [square](https://en.wikipedia.org/wiki/Square_matrix "Square matrix").  
> 在线性代数中，以 Otto Toeplitz 命名的 Toeplitz 矩阵或对角线常数矩阵是一个矩阵，其中从左到右的每个下降对角线都是常数。Toeplitz 矩阵不一定是正方形的。
### 1.2 循环矩阵
> In [linear algebra](https://en.wikipedia.org/wiki/Linear_algebra "Linear algebra"), a **circulant matrix** is a [square matrix](https://en.wikipedia.org/wiki/Square_matrix "Square matrix") in which all rows are composed of the same elements and each row is rotated one element to the right relative to the preceding row. It is a particular kind of [Toeplitz matrix](https://en.wikipedia.org/wiki/Toeplitz_matrix "Toeplitz matrix").
> 在线性代数中，循环矩阵是一个方形矩阵，其中所有行都由相同的元素组成，并且每行相对于前一行向右旋转一个元素。它是一种特殊的托普利茨矩阵。

首先定义向量 $\vec{b} = [b_0, b_1, b_2,...,b_{n-1}]$，
再定义一个 $n \times n$ 的循环矩阵B，每一个行的元素就是 $\vec{b}$ 的值, 因此
$$
B = \begin{bmatrix} 
b_{n-1} & b_{n-2} & b_{n-3} & b_{n-4} & ... & b_0 \\ 
b_0 & b_{n-1} & b_{n-2} & b_{n-3} & ... & b_1 \\ 
b_1 & b_0 & b_{n-1} & b_{n-2} &  ... & b_2  \\ 
... \\ 
b_{n-3} & b_{n-4} & b_{n-5} & b_{n-6} & ... & b_{n-2} \\ 
b_{n-2} & b_{n-3} & b_{n-4} & b_{n-5} & ... & b_{n-1}
\end{bmatrix} 
$$

## 2. 循环乘法和托普利茨乘法
### 2.1 循环乘法
再定义两列向量 $\vec{c}$，其中
$$\vec{c} = \begin{bmatrix} c_0 \\ c_1 \\ c_2  \\ ... \\ c_{n-2} \\ c_{n-1} \end{bmatrix}$$

对B和 $\vec{c}$ 做矩阵乘法，$\vec{a} = B\vec{c}$。计算复杂度为$O(n^2)$。
$$\vec{a} = B\vec{c}= \begin{bmatrix} a_0 \\ a_1 \\ a_2  \\ ... \\ a_{n-2} \\ a_{n-1} \end{bmatrix}$$

这个矩阵乘法可以等价于对应的多项式的乘法计算。

定义多项式：
$b(X) = \sum_{i} b_i X^i, c(X) = \sum_{i}c_iX^i, a(X)=\sum_{i}a_iX^i$
其中 $a_i = \sum_{j+k =i-1  (\mod n)}b_jc_k$

那么多项式的乘法可以表示为： $a(X)=X\cdot b(X) \cdot c(X) \\ (\mod X^n-1)$。只要计算出多项式$a(X)$，多项式的系数就是向量 $\vec{a}$ 的值。

我们用 $\omega$ 表示单位根，$\omega^n=1$。那么$a(\omega^i) = \omega^i \cdot b(\omega^i) \cdot c(\omega^i)$
如果使用FFT来计算的话，
$\hat{\vec{b}} = FFT(\vec{b})$，这一步的复杂度为$O(n\log n)$
$\hat{\vec{c}} = FFT(\vec{c})$，这一步的复杂度为$O(n\log n)$
$\hat{\vec{a}} = \hat{b} \circ \hat{c} \circ (1, \omega, \omega^2,...,\omega^{n-1})$，这一步的复杂度为$O(n)$
$\vec{a} =iFFT(\hat{\vec{a}})$，这一步的复杂度为$O(n\log n)$
通过多项式乘法来计算向量 $\vec{a}$ 的计算复杂度为$O(n\log n)$

### 2.2 托普利茨乘法转换成循环乘法
假设有一个托普利茨矩阵F和向量$\vec{c}$，做矩阵乘法
$$
F = \begin{bmatrix} 
f_{n-1} & f_{n-2} & f_{n-3} & f_{n-4} & ... & f_0 \\ 
0 & f_{n-1} & f_{n-2} & f_{n-3} & ... & f_1 \\ 
0 & 0 & f_{n-1} & f_{n-2} &  ... & f_2  \\ 
... \\ 
0 & 0 & 0 & 0 & ... & f_{n-2} \\ 
0 & 0 & 0 & 0 & ... & f_{n-1}
\end{bmatrix} 
$$

$$\vec{c} = \begin{bmatrix} c_0 \\ c_1 \\ c_2  \\ ... \\ c_{n-2} \\ c_{n-1} \end{bmatrix}$$对F和 $\vec{c}$ 做矩阵乘法
$$\vec{a} = F\vec{c}= \begin{bmatrix} a_0 \\ a_1 \\ a_2  \\ ... \\ a_{n-2} \\ a_{n-1} \end{bmatrix}$$

为了能够提高计算 $\vec{a}$ 的效率，这里我们可以将托普利茨乘法转换成循环乘法。
首先将F扩展为$2n \times 2n$的矩阵F'，这里F' 就是一个循环矩阵：
$$
F' = \begin{bmatrix} 
f_{n-1} & f_{n-2} & f_{n-3} & f_{n-4} & ... & f_0 & 0 & 0 & ... & 0\\ 
0 & f_{n-1} & f_{n-2} & f_{n-3} & ... & f_1 & f_0 & 0 & ... & 0\\ 
0 & 0 & f_{n-1} & f_{n-2} &  ... & f_2 & f_1 & f_0 & ... & 0\\ 
... \\ 
0 & 0 & 0 & 0 & ... & f_{n-2} & f_{n-3} & f_{n-4} & ... & 0\\ 
0 & 0 & 0 & 0 & ... & f_{n-1} & f_{n-2} & f_{n-3} & ... & 0\\ 
f_0 & 0 & 0 & 0 & ... & 0 & f_{n-1} & f_{n-2} & ... & 0\\ 
f_1 & f_0 & 0 & 0 & ... & 0 & 0 & f_{n-1} & ... & 0\\ 
... \\ 
f_{n-2} & f_{n-3} & f_{n-4} & f_{n-5} & ... & 0 & 0 & 0 & ... & f_{n-1}\\ 
\end{bmatrix} 
$$
再将$\vec{c}$ 扩展为2n的向量：
$$\vec{c'} = \begin{bmatrix} c_0 \\ c_1 \\ c_2  \\ ... \\ c_{n-2} \\ c_{n-1} \\ 0 \\ ... \\ 0 \end{bmatrix}$$

计算矩阵乘法：
$$\vec{a'} = F'\vec{c'}= \begin{bmatrix} a_0 \\ a_1 \\ a_2  \\ ... \\ a_{n-2} \\ a_{n-1} \\ a_n \\ ... \\ a_{2n-1} \end{bmatrix}$$
这里 $\vec{a'}$ 的前 n 个值就是 $\vec{a}$。这样计算 $\vec{a}$ 的复杂度就是$O(2n \log(2n))$

## 3. KZG 计算优化
我们可以利用循环乘法的方法去提高KZG10计算证明的步骤。
### 3.1 KZG10 证明
假设多项式为$f(X) = f_0 + f_1 X + f_2 X^2 + ... + f_{d-1} X^{d-1}$，
有一组值$\xi_1, \xi_2,..., \xi_m$，满足$f(\xi_k) = z_k$。

对于任意一个$\xi_k$，计算KZG证明：$\pi_k = [h_k(\tau)]_1$。
$$
\frac{C(\tau)-z_k}{\tau-\xi_k} = h_k(\tau) = \begin{matrix} & f_{d-1} \tau ^{d-1} \\ & + (f_{d-1} \xi_k  + f_{d-2}) \tau^{d-2}  \\ & + (f_{d-1} \xi_k^2  + f_{d-2} \xi_k + f_{d-3}) \tau^{d-3}  \\ & + (f_{d-1}\xi_k^3 + f_{d-2} \xi_k^2 + f_{d-3} \xi_k + f_{d-4}) \tau^{d-4}  \\  &...  \\ &+ (f_0 + f_1 \xi_k  + f_2 \xi_k^2 + ... + f_{d-1}\xi_k^{d-1})\end{matrix} = \begin{matrix} (f_{d-1}\tau^{d-1} + f_{d-2}\tau^{d-2} + f_{d-3}\tau^{d-3} +...+ f_2 \tau + f_1) \\ + (f_{d-1}\tau^{d-2} + f_{d-2}\tau^{d-3} + f_{d-3}\tau^{d-4} +...+ f_3 \tau + f_2) \xi_k \\ + (f_{d-1}\tau^{d-3} + f_{d-2}\tau^{d-4} + f_{d-3}\tau^{d-5} +...+ f_4 \tau + f_3) \xi_k^2 \\ + (f_{d-1}\tau^{d-4} + f_{d-2}\tau^{d-5} + f_{d-3}\tau^{d-6} +...+ f_5 \tau + f_4) \xi_k^3 \\ ... \\ + (f_{d-1} \tau + f_{d-2}) \xi_k^{d-2} \\ +f_{d-1} \xi_{k}^{d-1}\end{matrix}
$$


令$h_k(\tau) = h_0 + h_1 \xi_k + h_2 \xi_k^2 + ... + h_{d-1} \xi_k^{d-1}$，其中 $h_i = (f_{d-1}[\tau^{d-i-1}] + f_{d-2} [\tau^{d-i-2}] + ... + f_{i+2} [\tau] + f_{i+1})$
定义多项式$h_k'(X) = h_0 + h_1 \ + h_2 X^2 + ... + h_{d-1} X^{d-1}$。计算出 $\vec{h} = (h_0, h_1,..., h_{d-1})$ 后，因此只要带入挑战点就可以直接去算证明。

为了计算 $h_i$，可以使用托普利茨乘法。
$$
\begin{bmatrix} h_0 \\ h_1 \\ h_2  \\ ... \\ h_{d-2} \\ h_{d-1} \end{bmatrix} = 
\begin{bmatrix} 
f_{d-1} & f_{d-2} & f_{d-3} & f_{d-4} & ... & f_0 \\ 
0 & f_{d-1} & f_{d-2} & f_{d-3} & ... & f_1 \\ 
0 & 0 & f_{d-1} & f_{d-2} &  ... & f_2  \\ 
... \\ 
0 & 0 & 0 & 0 & ... & f_{d-2} \\ 
0 & 0 & 0 & 0 & ... & f_{d-1}
\end{bmatrix}  \cdot 
\begin{bmatrix} [\tau^{d-1}] \\ [\tau^{d-2}] \\ [\tau^{d-3}]  \\ ... \\ [\tau] \\ [1] \end{bmatrix} 
$$
于是我们可以将托普利茨乘法转换成循环乘法，
令 $\hat{\vec{s}} = ([\tau^{d-1}], [\tau^{d-2}], [\tau^{d-3}],...,[\tau], [1], [0], [0], ...,[0])$
$\hat{\vec{c}} = (0,0,...,0, f_0, f_1, ..., f_{d-1})$

假设 2d 的单元根为 $\nu$，使用FFT来计算：
$\vec{y} = FFT(\hat{\vec{s}})$
$\vec{v} = FFT(\vec{c})$
$\vec{u} = \vec{y} \circ \vec{v} \circ (1, \nu, \nu^2,...,\nu^{d-1})$，这一步的复杂度为$O(n)$
$\hat{\vec{h}} =iFFT(\vec{u})$，$\hat{\vec{h}}$ 的前d个值就是 $\vec{h}$。
