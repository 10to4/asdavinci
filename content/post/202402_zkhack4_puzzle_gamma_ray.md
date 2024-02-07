---
title: "【解题】ZKHACK IV Puzzle#1: Gamma Ray"
date: 2024-02-02T19:00:00+08:00
categories:
  - zkp
  - curve
---
题目链接：[Gamma Ray](https://zkhack.dev/zkhackIV/puzzleF1.html)
## 1. 问题描述

首先来看一下问题描述：

> Bob was deeply inspired by the Zcash design for private transactions and had some pretty cool ideas on how to adapt it for his requirements. He was also inspired by the Mina design for the lightest blockchain and wanted to combine the two. In order to achieve that, Bob used the MNT7653 cycle of curves to enable efficient infinite recursion, and used elliptic curve public keys to authorize spends. He released a first version of the system to the world and Alice soon announced she was able to double spend by creating two different nullifiers for the same key...
> 
> Bob 在隐私交易上深受Zcash的设计的启发，并在如何修改它来满足自己的需求上有一些想法。他在最轻量级区块链上也受到了Mina的启发，他想将两者结合起来。为了实现这个目的，Bob使用了MNT7653循环曲线来实现有效的无限递归，并且使用椭圆曲线公钥来认证花费。他向世界发布第一个版本的系统，但是Alice很快就宣称她能够通过创建同一个key上的两个不同的nullifiers来实现双花...

Bob结合来Zcash 和 Mina 的设计思想，设计了一套他隐私交易协议，但是不幸的是，协议中存在一个双花漏洞。这道题里面我们要做的事情其实就是实现双花。
## 2. 代码原理分析

我们先结合代码分析一下Bob的协议原理。这套协议里面使用了Merkel tree来保存所有的资产信息，每个叶子节点代表一笔资产。为了隐藏资产信息，叶子节点上并不直接保存资产信息，而是将资产信息的加密值保存在叶子节点上。当用户要花费一笔资产的时候，要生成验证的电路数据，最后需要使用Groth16进行验证。
我们来看一下几个关键步骤的原理：

**1.资产信息加密**

代码中我们用 `secret` 来表示要加密的资产信息，以下是产生约束代码的部分中，对 `secret`进行加密产生叶子节点数值的约束代码相关的部分：

```rust
let secret = FpVar::new_witness(ark_relations::ns!(cs, "secret"), || Ok(self.secret))?;
let secret_bits = secret.to_bits_le()?;
let base = G1Var::new_constant(ark_relations::ns!(cs, "base"), G1Affine::generator())?;
let pk = base.scalar_mul_le(secret_bits.iter())?.to_affine()?;
// Allocate Leaf
let leaf_g: Vec<_> = vec![pk.x];
```
这里对secret进行加密：$pk = base ^{secret}$ ，其中base是MNT6-753双线性群上的G1的生成元。pk是曲线上的一个点，然后取x轴的值做为保存在叶子结点上的值。

**2.merkle proof**

产生叶子节点后，接下来就是对叶子结点的值做证明。merkle root是公开的信息，用户要产生他的叶子节点对应的merkle path上的所有值做为proof。

```rust
let tree_proof = tree.generate_proof(i).unwrap();
	assert!(tree_proof	
	.verify(
		&leaf_crh_params,
		&two_to_one_crh_params,
		&root,
		leaf.as_slice()
	)
	.unwrap());
```
在这个例子中，一共有四个叶子节点，其中我们要验证的是第2个叶子节点。
	
**3.nullifier**

由于资产信息是隐藏的，所以除了用户本身，其它人无法获知资产的具体信息，但是需要一些办法来构造可以公开的唯一标识来标记资产信息。在Bob的协议中，他使用了一个字段 `nullifier` 来做为这个凭证，`nullifier` 是secret的哈希值，它使用了poseidon哈希算法实现的。原则上，这里 `nullifier` 应该可以用来避免双花的问题。

```rust
let nullifier = <LeafH as CRHScheme>::evaluate(&leaf_crh_params, vec![leaked_secret]).unwrap();
```
	
**4.Groth16**

Bob的协议里使用了Groth16做为零知识证明系统。Groth16 的执行需要基于椭圆曲线，这里选择了 MNT4-753曲线。

`use ark_mnt4_753::{Fr as MNT4BigFr, MNT4_753};`

在这段电路检查中有两个public input：merkle root 和nullifier，剩下的输入值是witness。
```rust
let c = SpendCircuit {
	leaf_params: leaf_crh_params.clone(),
	two_to_one_params: two_to_one_crh_params.clone(),
	root: root.clone(),
	proof: tree_proof.clone(),
	nullifier: nullifier.clone(),
	secret: leaked_secret.clone(),
};

let proof = Groth16::<MNT4_753>::prove(&pk, c.clone(), rng).unwrap();
assert!(Groth16::<MNT4_753>::verify(&vk, &vec![root, nullifier], &proof).unwrap());
```

**5.电路**

我们需要在电路里实现验证逻辑。结合上面的分析，我们可以总结一下电路验证的内容：

1）验证 `nullifier` 与`secret` 的关系

`nullifier` 是在资产信息保存到merkle tree的时候就确定下来的资产凭证，因此在花费这笔资产的时候需要取验证`nullifier` 字段，即需要验证要带到电路里执行验证的资产信息字段`secret` 的哈希值就是 `nullifier`。

2）验证叶子节点取值是`secret`的加密值

叶子节点是由 `secret` 加密而来的，因此我们需要验证要带到电路里执行验证的资产信息字段`secret` 的加密后的值是否是叶子节点上的值

3）验证叶子节点的merkle proof

获得叶子节点的取值后，需要再继续对叶子节点展开merkle proof的验证。
	
```rust
fn generate_constraints(
	self,
	cs: ConstraintSystemRef<ConstraintF>,
) -> Result<(), SynthesisError> {
	...
	// 验证 nullifier
	let nullifier_in_circuit =
	<LeafHG as CRHSchemeGadget<LeafH, _>>::evaluate(&leaf_crh_params_var, &[secret])?;
	nullifier_in_circuit.enforce_equal(&nullifier)?;

	// 加密secret
	let base = G1Var::new_constant(ark_relations::ns!(cs, "base"), G1Affine::generator())?;
	let pk = base.scalar_mul_le(secret_bits.iter())?.to_affine()?;
	// Allocate Leaf
	let leaf_g: Vec<_> = vec![pk.x];
	
	...
	// 验证merkle root
	cw.verify_membership(
		&leaf_crh_params_var,
		&two_to_one_crh_params_var,
		&root,
		&leaf_g,
	)?
	.enforce_equal(&Boolean::constant(true))?;
	
	Ok(())
}
```

## 3. 问题分析
上面的验证步骤包含了三个环节
1) nullifier = hash(secret)
2) leaf_g = pk.x, $pk = base^{secret}$
3) merkle proof(node)

其中哈希验证和merkle proof的验证步骤不太容易存在漏洞，问题最大的可能是在从 `secret` 生成叶子节点值leaf_g的环节。
先来看一下代码，代码这部分   
```rust
fn main() {
	...
	
	let tree_proof = tree.generate_proof(i).unwrap();
	assert!(tree_proof
	.verify(
		&leaf_crh_params,
		&two_to_one_crh_params,
		&root,
		leaf.as_slice()
	)
	.unwrap());
	
	let c = SpendCircuit {
		leaf_params: leaf_crh_params.clone(),
		two_to_one_params: two_to_one_crh_params.clone(),
		root: root.clone(),
		proof: tree_proof.clone(),
		nullifier: nullifier.clone(),
		secret: leaked_secret.clone(),
	};

	let proof = Groth16::<MNT4_753>::prove(&pk, c.clone(), rng).unwrap();
	assert!(Groth16::<MNT4_753>::verify(&vk, &vec![root, nullifier], &proof).unwrap());
	
	/* Enter your solution here */
	let nullifier_hack = MNT4BigFr::from(0);
	let secret_hack = MNT4BigFr::from(0);
	/* End of solution */
	
	assert_ne!(nullifier, nullifier_hack);
	
	let c2 = SpendCircuit {
		leaf_params: leaf_crh_params.clone(),
		two_to_one_params: two_to_one_crh_params.clone(),
		root: root.clone(),
		proof: tree_proof.clone(),
		nullifier: nullifier_hack.clone(),
		secret: secret_hack.clone(),
	};
	let proof = Groth16::<MNT4_753>::prove(&pk, c2.clone(), rng).unwrap();
	assert!(Groth16::<MNT4_753>::verify(&vk, &vec![root, nullifier_hack], &proof).unwrap());

}
```

根据这段代码，我们需要找到另一个秘密信息 $secret\\_hack$，它对应的哈希值 nullifier_hack 必须和 nullifier 不同，并且它可以通过电路约束的检查。

再回到生成叶子节点的步骤 leaf_g = pk.x, $pk = base^{secret}$。如果我们使用新的 $secret\_hack$ 可以生成同样的leaf_g，那么就可以达到目的。

令 $pk\\_hack = g1^{secret\_hack}$ ，那么若 leaf_g = pk_hack.x，只表明 pk_hack和 pk相等或者 pk_hack在椭圆曲线上的位置和 pk是对称的（即pk.x = pk_hack.x，pk.y = pk_hack.y）。

由于 $nullifier\_hack \neq nullifier$，所以 $secret\_hack \neq secret$，那么必然 $pk\_hack \neq pk$。所以必然 pk_hack 和 pk 是对称点。

## 4. 解答
根据上文的推断，我们可以得出
$pk\_hack = pk^{-1} = (base^{secret})^{-1} =base ^ {-secret} = base^{secret\\_hack}$。
由于 base 是 MNT6-753 上双线性群上的G1的生成元，所以 $secret\_hack = 0 -secret \mod r_6$，其中 $r_6$ 是 MNT6-753 上Fr的模。
但由于 secret_hack 是 MNT4-753的 Fr 域上的值，因此直接计算 secret_hack 是有问题的，而
$r_4 = 41898490967918953402344214791240637128170709919953949071783502921025352812571106773058893763790338921418070971888458477323173057491593855069696241854796396165721416325350064441470418137846398469611935719059908164220784476160001$$r_6 = 41898490967918953402344214791240637128170709919953949071783502921025352812571106773058893763790338921418070971888253786114353726529584385201591605722013126468931404347949840543007986327743462853720628051692141265303114721689601$
所以 $r_4 \gt r_6$，所以可以直接将 $r\_6$ 转成MNT4的 Fr 域上的值进行计算
```rust
/* Enter your solution here */
let secret_hack = MNT4BigFr::from(MNT6BigFr::MODULUS) - leaked_secret;
let nullifier_hack =
    <LeafH as CRHScheme>::evaluate(&leaf_crh_params, vec![secret_hack]).unwrap();
/* End of solution */
```

（也可以用 先计算 $r_4$ 和 $r_6$ 之间的差值，然后再用MNT4的 Fr 域上的 - secret 加上差值来计算secret_hack）
```rust
let delta = MNT4BigFr::from(-1) - MNT4BigFr::from(MNT6BigFr::from(- 1).into_bigint());
let secret_hack: MNT4BigFr = - leaked_secret - delta;
```
## 5. 参考资料

* https://zips.z.cash/protocol/protocol.pdf