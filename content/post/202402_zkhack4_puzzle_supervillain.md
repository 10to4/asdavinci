---
title: "【解题】ZKHACK IV Puzzle#2: Supervillain"
date: 2024-02-04T19:00:00+08:00
categories:
  - zkp
---
题目链接：[Supervillain](https://zkhack.dev/zkhackIV/puzzleF2.html)
## 1. 问题描述
首先看一下问题描述：
>Bob has been designing a new optimized signature scheme for his L1 based on BLS signatures. Specifically, he wanted to be able to use the most efficient form of BLS signature aggregation, where you just add the signatures together rather than having to delinearize them. In order to do that, he designed a proof-of-possession scheme based on the B-KEA assumption he found in the the Sapling security analysis paper by Mary Maller. Based the reasoning in the Power of Proofs-of-Possession paper, he concluded that his scheme would be secure. After he deployed the protocol, he found it was attacked and there was a malicious block entered the system, fooling all the light nodes...
>
>Bob 基于 BLS 签名在他的 L1 层已经设计了一个新的优化签名方案。尤其是，他想要使用最高效形式的BLS签名聚合，这里你只需将签名添加在一起，而不必对它们进行非线形聚合。为了做这件事，他基于B-KEA假设设计了一个proof-of-possession方案，这是他在Mary Maller的Sapling安全分析论文上发现的。基于Proofs-of-Possession论文的强大的原因，他总结出他的方案也是安全的。在他部署协议之后，他发现还是被攻击了，有恶意的块进去系统，欺骗过了所有的轻节点。

Bob基于BLS签名算法设计了一个聚合签名算法用在了他的L1链上，但是由于他的方案有漏洞，导致最后有恶意的区块被打包上链，我们要做的事情就是找到他的方案中的漏洞。


## 2. 代码原理分析
我们先结合代码分析一下Bob的协议原理。
### 2.1 POK
Bob应用到的第一个协议是PoK，即Proof of Konwledge。
> In [cryptography](https://en.wikipedia.org/wiki/Cryptography "Cryptography"), a **proof of knowledge** is an [interactive proof](https://en.wikipedia.org/wiki/Interactive_proof_system "Interactive proof system") in which the prover succeeds in 'convincing' a verifier that the prover knows something. What it means for a [machine](https://en.wikipedia.org/wiki/Abstract_machine "Abstract machine") to 'know something' is defined in terms of computation. A machine 'knows something', if this something can be computed, given the machine as an input. As the program of the prover does not necessarily spit out the knowledge itself (as is the case for [zero-knowledge proofs](https://en.wikipedia.org/wiki/Zero-knowledge_proofs "Zero-knowledge proofs") a machine with a different program, called the knowledge extractor is introduced to capture this idea. We are mostly interested in what can be proven by [polynomial time](https://en.wikipedia.org/wiki/Polynomial_time "Polynomial time") bounded machines. In this case the set of knowledge elements is limited to a set of witnesses of some [language](https://en.wikipedia.org/wiki/Formal_language "Formal language") in [NP](https://en.wikipedia.org/wiki/NP_(complexity) "NP (complexity)").
> 
> 在密码学中，**proof of knowledge**是一种交互式证明，证明者成功地使验证者信服证明者知道某事。机器“知道某事”意味着什么，是用计算来定义的。一台机器“知道一些东西”，如果这个东西可以被计算出来，把机器作为输入。由于证明者的程序不一定会吐出知识本身（就像零知识证明一样 [1](https://en.wikipedia.org/wiki/Proof_of_knowledge#cite_note-1) ），因此引入了具有不同程序的机器，称为知识提取器来捕获这个想法。我们最感兴趣的是多项式时间可以证明什么。在这种情况下，知识元素的集合仅限于 NP 中某种语言的一组witnesses。

这里用到PoK的目的是要证明prover知道后续签名的私钥，PoK的代码如下：
```rust
fn。derive_point_for_pok(i: usize) -> G2Affine {
	let rng = &mut ark_std::rand::rngs::StdRng::seed_from_u64(20399u64);
	G2Affine::rand(rng).mul(Fr::from(i as u64 + 1)).into()
}

#[allow(dead_code)]
fn pok_prove(sk: Fr, i: usize) -> G2Affine {
	derive_point_for_pok(i).mul(sk).into()
}

fn pok_verify(pk: G1Affine, i: usize, proof: G2Affine) {
	assert!(Bls12_381::multi_pairing(
		&[pk, G1Affine::generator()],
		&[derive_point_for_pok(i).neg(), proof]
	)
	.is_zero());
}
```
1.生成证明（pok_prove() 函数）

$proof = T \cdot {sk}$，其中$T$ 是G2上的一个随机点

2.验证证明（pok_verify() 函数）

$e(pk, T) = e(g_1, proof)$，其中 $pk = [sk]_1$

由于在 Bob 的协议里，后面要对多个证明做聚合，这里对PoK协议做了修改，使用 derive_point_for_pok() 函数生成 $T$。
$T_i = [rng \cdot (i+1)]_2$，其中 rng 是随机数，i 为index索引值。
### 2.2 BLS签名协议
```rust
fn hasher() -> MapToCurveBasedHasher<G2Projective, DefaultFieldHasher<Sha256, 128>, WBMap<Config>> {
	let wb_to_curve_hasher =
	MapToCurveBasedHasher::<G2Projective, DefaultFieldHasher<Sha256, 128>, WBMap<Config>>::new(
	&[1, 3, 3, 7],
	)
	.unwrap();
	wb_to_curve_hasher
}

#[allow(dead_code)]
fn bls_sign(sk: Fr, msg: &[u8]) -> G2Affine {
	hasher().hash(msg).unwrap().mul(sk).into_affine()
}

fn bls_verify(pk: G1Affine, sig: G2Affine, msg: &[u8]) {
	assert!(Bls12_381::multi_pairing(
		&[pk, G1Affine::generator()],
		&[hasher().hash(msg).unwrap().neg(), sig]
	)
	.is_zero());
}
```

其中 hasher() 函数用来对message信息做哈希计算，哈希结果为G2上的一个点。

bls_sign() 函数用私钥sk对哈希值签名，$sig = hash(msg) \cdot {sk}$。

bls_verify() 函数用来验证签名，$e(pk, hash(msg)) = e(g_1, sig)$，其中 $pk = [sk]_1$。

### 2.3 聚合签名方案分析
当Bob 设计了一个聚合方案来验证一组BLS签名，这组BLS签名是使用一组不同的私钥 $\{sk_i\}$ 对同一个msg的签名，假设这组签名的数量为n。首先将公钥聚合起来：

aggregate_key = $\sum_{i \in [0..n)} pk_i = \sum_{i \in [0..n)} [sk_i]_1$，其中每一个公钥$pk_i$ 对应私钥 $sk_i$，

接下来继续聚合签名：

aggregate_sig = $\sum_{i \in [0..n)} sig_i = hash(msg) \cdot {\sum_{i \in [0..n)} sk_i}$

接下来可以使用聚合的公钥和聚合的签名进行验证：

e(aggregate_key, hash(msg)) = e($g_1$, aggregate_sig)

这个聚合方案还有一个好处，是可以在原本的aggregate_key和aggregate_sig继续聚合任意多个新的公钥和签名，

即
e(aggregate_key+ $pk_{new}$, hash(msg)) = e($g_1$, aggregate_sig + $sig_{new}$)

## 3. 问题分析
```rust
fn main() {
	...
	
    let public_keys: Vec<(G1Affine, G2Affine)> = from_file("public_keys.bin");
    public_keys
        .iter()
        .enumerate()
        .for_each(|(i, (pk, proof))| pok_verify(*pk, i, *proof));
    let new_key_index = public_keys.len();
    let message = b"YOUR GITHUB USERNAME";

    /* Enter solution here */
    let new_key = G1Affine::zero();
    let new_proof = G2Affine::zero();
    let aggregate_signature = G2Affine::zero();
    /* End of solution */

    pok_verify(new_key, new_key_index, new_proof);
    let aggregate_key = public_keys
        .iter()
        .fold(G1Projective::from(new_key), |acc, (pk, _)| acc + pk)
        .into_affine();
    bls_verify(aggregate_key, aggregate_signature, message)
}
```
问题代码中，首先导入一组公钥及它们的证明，共 10 个。这道题的目的在不知道这10个私钥的前提下，通过聚合一个新的公钥的签名，来构造出一个假的聚合签名，使得它能通过验证，并且这个新的公钥要能通过pok验证。

即 e(aggregate_key, hash(message)) = e($g_1$, aggregate_sig + $sig_{new}$)，

其中 aggregate_key = $\sum_{i \in [0..10)} pk_i$ + new_key = $\sum_{i \in [0..10)} [sk_i]\_1 +$ new_key = $[\sum_{i \in [0..10)} sk_i$ + sk\_new$]_1$。(new\_key = $[$sk\_new$]_1$)

虽然我们不知道 $\\{sk_i\\}\_{i \in [0, 10)}$ 的取值，但是可以通过构造 new_key 来构造获得任意的 aggregate_key。

令 new_key = $\[k\]\_1 - \sum_{i \in [0..10)} pk_i$，而new_key对应的私钥为 sk_new = $k - \sum_{i \in [0..10)} sk_i$。

那么aggregate_key = $\sum_{i \in [0..10)} pk_i$ + new_key = $\sum_{i \in [0..10)} \[sk\_i\]\_1 + \[k\]\_1 - \sum_{i \in [0..10)} pk_i = \[k\]\_1$，k可以是任意值。

由于我们已经知道了 ${pk_i}\_{i \in \[0, 10)}$ 的PoK证明$\{proof_i\}\_{i \in [0, 10)}$ ，因此可以利用 $\{proof_i\}\_{i \in [0, 10)}$  来构造new_key 的PoK证明。

由于 $proof_i = T \cdot {sk_i} = \[rng \cdot (i+1) \cdot {sk_i}\]\_2$，

因而 $new\_proof = T_{10} \cdot {sk\_new} = \[rng \cdot 11\]\_2 \cdot (k - \sum_{i \in [0..10)} sk_i)$

$= \[rng \cdot 11 \cdot (k - \sum_{i \in \[0..10)} sk_i)\]_2$

$=\[rng \cdot 11 \cdot k\]\_2 - \sum_{i \in \[0..10)}\[rng \cdot 11 \cdot sk_i\]\_2$

$=\[rng \cdot 11 \cdot k\]\_2 -\sum_{i \in \[0..10)}\frac{11}{i+1}\[rng \cdot (i+1) \cdot sk_i\]\_2$

$=\[rng \cdot 11 \cdot k\]\_2 -\sum_{i \in \[0..10)}\frac{11}{i+1}proof_i$

接下来，为了通过BLS验证，需要生成聚合签名

aggregate_signature = $\sum_{i \in \[0..10)} sig_i$ + sig_new

= $hash(msg) \cdot {\sum_{i \in [0..n)} sk_i} + hash(msg) \cdot$ sk_new

$= hash(msg) \cdot ({\sum_{i \in [0..10)} sk_i}$ + sk_new) 

= $hash(msg) \cdot k$

## 4. 解答
总结一下解题步骤
1.生成一个任意值k，aggregate_key = $[k]_1$。

2.计算 new_key，new_key = $\[k\]\_1 - \sum_{i \in [0..10)} pk_i$。

3.计算 new_proof，new_proof = $\[rng \cdot 11 \cdot k\]\_2 -\sum_{i \in [0..10)}\frac{11}{i+1}proof_i$。

4.计算 aggregate_signature，aggregate_signature = $hash(msg) \cdot k$。

5.使用 new_key 和 new_proof 完成 PoK 验证。

6.计算 aggregate_key，aggregate_key = $\sum_{i \in [0..10)} pk_i +$ new_key。 

7.使用 aggregate_key，aggregate_signature 和 message 进行聚合签名验证 

e(aggregate_key, hash(message)) = e($g_1$, aggregate_signature)

以下代码以k=1为例，构造new_key，new_proof 和 aggregate_signature 来完成验证
```rust
let new_key = public_keys
    .iter()
    .fold(G1Projective::from(G1Affine::generator()), |acc, (pk, _)| acc - pk)
    .into_affine();
let new_proof = public_keys
.iter()
.enumerate()
.fold(G2Projective::from(derive_point_for_pok(0)), |acc, (i,(_, proof))| acc - proof.mul(Fr::from(1).div(Fr::from(i as u64 + 1))))
.into_affine().mul(Fr::from(new_key_index as u64 + 1)).into();
let aggregate_signature = bls_sign(Fr::from(1),message);
```

## 5. 参考资料

https://github.com/zcash/sapling-security-analysis/blob/master/MaryMallerUpdated.pdf_

https://rist.tech.cornell.edu/papers/pkreg.pdf_





