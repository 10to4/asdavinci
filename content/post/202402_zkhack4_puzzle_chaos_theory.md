---
title: "【解题】ZKHACK IV Puzzle#3: chaos theory"
date: 2024-02-04T19:00:00+08:00
categories:
  - zkp
---
题目链接：[Chaos theory](https://zkhack.dev/zkhackIV/puzzleF3.html)
## 1. 问题描述
> Bob designed a new one time scheme, that's based on the tried and true method of encrypt + sign. He combined ElGamal encryption with BLS signatures in a clever way, such that you use pairings to verify the encrypted message was not tampered with. Alice, then, figured out a way to reveal the plaintexts...
> 
> Bob 设计了一个新的一次性方案，这是基于加密+签名的久经考验的方法。他用一个聪明的方法将ElGamal加密算法和BLS签名结合，使得你可以使用pairings来验证加密信息未被篡改。然后 Alice 发现了一个方法能够揭示出明文... 

Bob 把ElGamal加密算法和BLS签名结合起来，设计了一套新方案，但是Alice可以从中破解出被加密的数据。我们接下来要做的就是找到Alice获取加密元象的方法。

## 2. 代码原理分析

### 2.1 ElGamal加密算法
> In [cryptography](https://en.wikipedia.org/wiki/Cryptography "Cryptography"), the **ElGamal encryption system** is an [asymmetric key encryption algorithm](https://en.wikipedia.org/wiki/Asymmetric_key_encryption_algorithm "Asymmetric key encryption algorithm") for [public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography "Public-key cryptography") which is based on the [Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange "Diffie–Hellman key exchange"). It was described by [Taher Elgamal](https://en.wikipedia.org/wiki/Taher_Elgamal "Taher Elgamal") in 1985. ElGamal encryption is used in the free [GNU Privacy Guard](https://en.wikipedia.org/wiki/GNU_Privacy_Guard "GNU Privacy Guard") software, recent versions of [PGP](https://en.wikipedia.org/wiki/Pretty_Good_Privacy "Pretty Good Privacy"), and other [cryptosystems](https://en.wikipedia.org/wiki/Cryptosystem "Cryptosystem"). 
> 
> 在密码学中，ElGamal 加密系统是一种用于公钥密码学的非对称密钥加密算法，它基于 Diffie-Hellman 密钥交换。Taher Elgamal 在 1985 年描述了它。ElGamal 加密用于免费的 GNU Privacy Guard 软件、最新版本的 PGP 和其他密码系统。

```rust
pub struct Message(G1Affine);

struct Sender {
	pub sk: Fr,
	pub pk: G1Affine,
}

pub struct Receiver {
	pk: G1Affine,
}

impl Sender {
	pub fn send(&self, m: Message, r: &Receiver) -> ElGamal {
		let c_2: G1Affine = (r.pk.mul(&self.sk) + m.0).into_affine();
		ElGamal(self.pk, c_2)
	}
}
```
ElGamal 加密算法的参与方分为sender和receiver，他们分别持有一组公私钥对。

ElGamal 加密算法产生的密文有两个值(c1, c2)，其中 $c_1 = pk_{sender} = g_1^{sk_{sender}}, c_2 = pk^{sk_{sender}}_{receiver} *  m$。

### 2.2 BLS签名算法

```rust
impl Sender {
	pub fn authenticate(&self, c: &ElGamal) -> G2Affine {
		let hash_c = c.hash_to_curve();
		hash_c.mul(&self.sk).into_affine()
	}
}

impl Auditor {
	pub fn check_auth(sender_pk: G1Affine, c: &ElGamal, s: G2Affine) -> bool {
		let lhs = { Bls12_381::pairing(G1Projective::generator(), s) };
		let hash_c = c.hash_to_curve();
		let rhs = { Bls12_381::pairing(sender_pk, hash_c) };
		lhs == rhs
	}
}
```

使用BLS签名对ElGamal 加密密文c进行签名验证。

authenticate()函数进行签名 $s = hash_c^{sk_{sender}}$；

check_auth() 函数进行验证 $e(g_1, pk_{sender}) = e(s, hash_c)$。

### 2.3 加密+签名方案

Bob的方案就是将 ElGamal 加密和签名结合起来，Sender 和 Receiver分别持有一对公私钥 (sk, pk)。使用ElGamal 加密算法产生一组对message进行加密的密文c，使用BLS签名算法对c生成签名s。根据Sender 和 Receiver的pk，密文c和签名s，生成结构体Blob。
```rust
pub struct Blob {
    pub sender_pk: G1Affine,  // Sender's public key
    pub c: ElGamal,          // Encrypted data
    pub s: G2Affine,         // BLS signature for c
    pub rec_pk: G1Affine,    // Receiver's public key
}
```

使用BLS签名验证算法对Blob数据进行验证。
## 3. 问题分析
问题代码中，首先生成一组 Message，然后导入一个Blob文件，然后对文件内容进行BLS验证。我们要做的是去获取ElGamal加密的原文是这组Message中的哪一个。
```rust
pub fn main() {
    ...

    let messages = generate_message_space();
    let mut file = File::open("blob.bin").unwrap();
    let mut data = Vec::new();
    file.read_to_end(&mut data).unwrap();
    let blob = Blob::deserialize_uncompressed(data.as_slice()).unwrap();

    // ensure that blob is correct
    assert!(Auditor::check_auth(blob.sender_pk, &blob.c, blob.s));

    /* Implement your attack here, to find the index of the encrypted message */
    unimplemented!();
    /* End of attack */
}
```
我们可以观察一下密文c。
$c_2 = pk^{sk_{sender}}_{receiver} *  m = (g_1^{sk_{receiver}})^{sk_{sender}}+m = g_1^{sk_{receiver} * sk_{sender}}+m$

$==> c_2 - m = g_1^{sk_{receiver} * sk_{sender}}$

$==> e(c_2 - m, g_2) = e(g_1^{sk_{receiver}}, g_2^{sk_{sender}}) = e(rec_{pk}, g_2^{sk_{sender}})$

虽然我们不知道 $g_2^{sk_{sender}}$，但是由于 $hash_c$ 和 $s = hash_c^{sk_{sender}}$，因此我们可以用s来代替$g_2^{sk_{sender}}$，
即 $e(c_2 - m, hash_c) = e(g_1^{sk_{receiver}}, hash_c^{sk_{sender}})= e(rec_{pk}, s)$

## 4. 解答

首先我们先进行hash_c = hash(c)。
接下来可以遍历 Message，使用 Message带入计算 re_sender_pk = $c_2 - m$。
计算 lhs = e(re_sender_pk, hash_c)，rhs = e(rec_pk, s)，若 lhs 与 rhs相等，则说明当前的message就是被加密的message

实现代码如下：
```rust
let hash_c = blob.c.hash_to_curve();
for (i, m) in messages.iter().enumerate() {
    let re_sender_pk = (blob.c.1 - m.0).into_affine();
    let lhs = { Bls12_381::pairing(re_sender_pk, hash_c) };
    let rhs = { Bls12_381::pairing(blob.rec_pk, blob.s) };
    if lhs == rhs {
        println!("index = {}", i);
    }
}
```
总之，通过循环访问所有可能的Message，我们可以确定 blob 中的密文对应哪条消息。如果它们匹配，则输出原始消息的索引。


