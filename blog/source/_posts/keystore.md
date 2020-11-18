---
title: Ethereum Keystore
copyright: true
date: 2019-05-19 14:21:54
categories:
- Cryptography
tags:
---

# Keystore简介

以太坊的每个外部账户都是由一对密钥定义的。下图是比特币公私钥与账户地址的关系。

![1](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110007.jpg)

<!-- more -->

keystore 文件（存储在`~/.ethereum/keystore`Linux或`C:\Users\<User>\Appdata\Roaming\Ethereum\keystore`Windows上）是用于签署交易的以太网私钥的加密版本。私钥通常用你创建帐户时设置的密码进行加密。如果你丢失了这个文件，你就丢失了私钥。当然，忘了密码也意味着你进不去这个账户了。如果丢了文件又忘了密码，那么恭喜你，心大的你只能看看账户里的余额，喝酒消愁了。

这个文件是**安全性**（攻击者需要密钥库文件*和* 密码来窃取您的资金）和**可用性**之间的完美折衷（您只需要密钥库文件*和* 密码来使用您的钱）。为了让您发送一些以太网，大多数以太坊客户端会要求您输入密码（与您创建帐户时使用的密码）相同，以便解密您的以太坊私钥。解密后，客户端程序可以使用私钥来签署您的交易，并让您移动您的资金。

# Keystore文件内容

文件来源[此处](https://github.com/hashcat/hashcat/issues/1228)：

```
$ cat~ / .ethereum / keystore / UTC  -  <created_date_time>  -  008aeeda4d805471df9b2a5b0f38a0c3bcba786b 
{ 
    “crypto”：{ 
        “cipher”：“aes-128-ctr”，
        “cipherparams”：{ 
            “iv”：“83dbcc02d8ccb40e466191a123791e0e” 
        }，
        “ciphertext”：“d172bf743a674da9cdad04534d56926ef8358534d458fffccd4e6ad2fbde479c”，
        “kdf”：“scrypt”，
        “kdfparams”：{ 
            “dklen”：32，
            “n”：262144，
            “r”：1，
            “p”：8，
            “salt”：“ab0c7876052600dd703518d6fc3fe8984592145b591fc8fb5c6d43190334ba19“ 
        }，
        “mac”：“2103ac29920d71da29f15d75b4a16dbe95cfd7ff8faea1056c33131d846e3097” 
    }，
    “id”：“d57df606-1b86-447a-a60e-18a539f9d92e”，
    “version”：3 
}
```

一个粗略的JSON文件，包含许多参数。

# 分析文件内容

* 第一行字符串（008aeeda4d805471df9b2a5b0f38a0c3bcba786b ）是**address** 。

* **crypto**下的参数解析：
  * **cipher**：对称[AES算法](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)的名称；
  * **cipherparams**：上面“密码”算法所需的参数；
  * **ciphertext**：使用上面的“密码”算法加密的以太网私钥；
  * **kdf**：一个[密钥派生函数，](https://en.wikipedia.org/wiki/Key_derivation_function)用于让您使用密码加密密钥库文件；
  * **kdfparams**：上述“kdf”算法所需的参数；
  * **mac**：用于验证密码的[代码](https://en.wikipedia.org/wiki/Message_authentication_code) ；在密码输错时可以给出一个反馈。
* **version**: Keystore文件的版本，目前为第3版，也称为V3 KeyStore。
* **id** : uuid；例子中，V=4,T=a，表示uuid v4版本（基于随机数）；

让我们看看它们如何协同工作，在密码短语下保护keystore文件。

## 1.加密私钥

如前所述，以太坊帐户是用于对交易进行加密签名的私钥 - 公钥对。为了确保私钥不像普通文件一样存储在文件中（即可以访问该文件的任何人可读），使用强对称算法（**密码**）对其进行加密至关重要。

这些对称算法使用密钥来加密某些数据。结果数据是加密的，可以使用相同的方法和相同的密钥进行解密 - 因此名称为“对称”。对于本文，我们将此对称密钥称为**解密密钥，**因为它将用于解密我们的以太坊私钥。

这就是**cipher**，**cipherparams**和**ciphertext**对应的：

- **cipher**是用于加密以太坊私钥的对称算法。默认为`aes-128-ctr`,128代表位。又名“128位密钥”或“256位密钥”。数字越高，越安全。
- **cipherparams**是*aes-128-ctr*算法所需的参数。这里，唯一的参数是*iv*。
  * **iv**：aes-128-ctr加密算法需要的初始化向量，大小必须符合密码的要求。如果没有提供，则通过crypto.getRandomBytes生成随机数。这里是128bit。
- **ciphertext**是*aes-128-ctr*函数的加密输出也是之后的输入。

具体的加密流程如下：

1. 生成一个随机数作为盐 | 给定一个盐值，对密码进行加密（kdf）生成派生秘钥（即**解密密钥**）；
2. 然后从派生秘钥中截取16位字符作为派生密码；
3. 再生成一个16位的随机数作为向量iv；
4. 然后拿着生成的派生密码，向量，私钥进行加密得到密文ciphertext；
5. 最后拿着生成的密文，和派生秘钥做hash处理作为校验码mac；
6. 将相关参数和密文保存到JSON文件keystore中。

所以，您需要先得到**解密密钥**。实际上，记住你设置的密码即可，下小节讲解。

![2](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110012.png)

## 2.用密码保护它

为了确保轻松解锁您的帐户，您无需记住非常长且非用户友好的**解密密钥**。相反，以太坊开发人员选择了基于密码短语的保护 - 也就是说，您只需要输入密码来得到**解密密钥**。

为了能够实现这一点，以太坊使用[密钥派生函数](https://en.wikipedia.org/wiki/Key_derivation_function)，在给定密码和参数列表的情况下计算**解密密钥**。

**kdf**和**kdfparams**：

- **kdf**是用于从密码短语计算（或“衍生”）**解密密钥**的密钥派生函数。这里，**kdf**的值是*scrypt* **。**
- **kdfparams**是{% post_link KDF之Scrypt *scrypt* %}函数所需的参数。这里，*dklen*，*n*，*r*，*p*和*salt*是**kdf**函数的参数。
  
  * `dklen：32`，- 派生密钥长度（以字节为单位）。对于某些密码设置，这必须匹配那些块大小。小于等于(2^32 - 1) * 32的正整数。
  * `n：262144`，- 迭代计数。geth的默认值为262144。 n 和 r 决定了占用的内存区域大小和哈希迭代次数（占用的内存大小为 128⋅n⋅r bytes，迭代次数为 2⋅n⋅r），可以修改的参数是 n。把 n 值定在 65536，这样可以兼容大部分低端配置的手机。必须大于1，是2的幂且小于2^(128 * r / 8)。
  *  `r：8`，- 底层散列的块大小，默认为8。决定了连续读大小（sequential read size），通常不应该修改。
  *  `p：1` - 并行化因素。默认为1。小于等于((2^32-1) * 32) / (128 * r)的正整数。
  *  `salt` - kdf的随机盐。大小必须符合KDF（密钥派生函数）的要求。如果没有提供，则通过crypto.getRandomBytes生成随机数。
  
  > 减少/覆盖scrypt设置为1024 * 8 * 1能更快破解？
  > 不可以。更改  N  将改变迭代计数，这将改变散列。即使您的字典中包含明文，它也不会破解散列。
  >
  > 另外，来计算一下默认参数下，并行计算需要多少内存：
  >
  > 1：（128 \* 8）\* 262144 = 1024 \* 262144 = 268,435,456字节= 256MB
  > 2：64（AMD卡）\* 64（[RX Vega 64具有64个CU]
  >      (https://www.techpowerup.com/gpudb/2871/radeon-rx-vega-64）= 4,096个并行计算
  > 3：256MB \* 4,096 = 1,048,576 MB RAM = *Vega 64需要**1,024** GB RAM*

> - 所以，如果我忘记了Eth钱包的密码，那么可能需要几年时间才能破解？！？！
> - 对。
>    //n的值越大，加密级别越高，当然消耗的内存越大。

在这里，使用**kdfparams**参数调整*scrypt*函数并将其**输入**密码，您将获得我们的**解密密钥**作为密钥派生函数的输出。

![3](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110022.png)

## 3.确保您的密码是正确的

我们已经描述了从密钥库文件和密码短语中确定我们的以太网私钥所需的一切。但是，如果用于解锁帐户的密码错误，会发生什么？

从我们到目前为止所看到的，所有操作（派生和解密）都将成功，但最后计算的以太坊私钥将不正确 - 这首先违背了使用密钥文件的要点！

我们需要保证键入的密码以解锁帐户是正确的，它与最初创建密钥库文件时输入的密码相同（记得`geth account new`吗？）。

这是密钥库文件中的**mac**值发挥作用的地方。在执行密钥派生函数之后，处理其结果（**解密密钥**）和**密文**并与**mac**进行比较（其作用类似于批准的印章）。如果结果与**mac**相同，则密码是正确的，并且解密可以开始。

在与**mac**进行比较之前，将**解密密钥**（仅第二个最左边的16个字节）和**密文**进行连接和散列（SHA3-256）。更多信息[在这里](https://github.com/hashcat/hashcat/issues/1228)。

![4](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110029.png)

# 总结

首先，您输入了密码，该密码被**kdf**密钥派生函数用来计算出**解密密钥**。然后，将新计算的**解密密钥**与**密文**一起处理，并与**mac**进行比较，以确保密码是正确的。最后，使用**解密密钥**通过**密码**对称解密函数**解密密文**。

瞧！解密的结果是您的以太网私钥。您可以在这里查看整个过程：

![5](https://cdn.jsdelivr.net/gh/JNhua/blog_images@master/img/20201029110041.png)

从图中可以看出，你的密码作为唯一的输入，你的以太网私钥作为唯一的输出。创建新的以太坊帐户时生成的keystore文件中提供了所需的其他信息。

出于这个原因，请确保您的密码是强大的（*并且您以某种方式记住它们！*）以保证窃取您的密钥库文件的攻击者无法轻松得到您的私钥。

# Refer.

[1]. [What is an Ethereum keystore file?](https://medium.com/@julien.maffre/what-is-an-ethereum-keystore-file-86c8c5917b97)

[2]. [Ethereum KDF - Scrypt](https://github.com/hashcat/hashcat/issues/1228)