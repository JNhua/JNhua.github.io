---
title: hash
date: '2020/9/18 15:01:45'
updated: '2020/9/18 21:12:16'
tags: []
category:
  - 区块链
  - 密码学
mathjax: true
toc: false
---
# hash
以SHA256算法为例。
<!--more-->
SHA256是SHA-2下细分出的一种算法。
SHA-2，名称来自于安全散列算法2（英语：Secure Hash Algorithm 2）的缩写，一种密码散列函数算法标准，由美国国家安全局研发，属于SHA算法之一，是SHA-1的后继者。
对于任意长度的消息，SHA256都会产生一个256bit长的哈希值，称作消息摘要。这个摘要相当于是个长度为32个字节的数组，通常用一个长度为64的十六进制字符串来表示。
## 信息预处理
### STEP1：附加填充比特
在报文末尾进行填充，使报文长度在对512取模以后的余数是448
填充是这样进行的：先补第一个比特为1，然后都补0，直到长度满足对512取模后余数是448。
需要注意的是，信息必须进行填充，也就是说，即使长度已经满足对512取模后余数是448，补位也必须要进行，这时要填充512个比特。
因此，填充是至少补一位，最多补512位。
例：以信息“abc”为例显示补位的过程。
a,b,c对应的ASCII码分别是97,98,99
于是原始信息的二进制编码为：01100001 01100010 01100011
补位第一步，首先补一个“1” ： 0110000101100010 01100011 1
补位第二步,补423个“0”：01100001 01100010 01100011 10000000 00000000 … 00000000
> 为什么是448?
因为在第一步的预处理后，第二步会再附加上一个64bit的数据，用来表示原始报文的长度信息。而448+64=512，正好拼成了一个完整的结构。

### STEP2：附加长度值
附加长度值就是将原始数据（第一步填充前的消息）的长度信息补到已经进行了填充操作的消息后面。
SHA256用一个64位的数据来表示原始消息的长度。
因此，通过SHA256计算的消息长度必须要小于2^64，当然绝大多数情况这足够大了。
长度信息的编码方式为64-bit big-endian integer。
回到刚刚的例子，消息“abc”，3个字符，占用24个bit，末尾加上00000000 00000018。
## 逻辑运算
逻辑位运算。
![4c945b7bf1b7ec1e62acbaa9112c32d3.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENResource/p2442)
## 计算消息摘要
首先：将消息分解成512-bit大小的块(break message into 512-bit chunks)
![6c08ee2b2b16c69cf6e75e98b61ea1df.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENNote/p472?hash=6c08ee2b2b16c69cf6e75e98b61ea1df)
假设消息M可以被分解为n个块，于是整个算法需要做的就是完成n次迭代，n次迭代的结果就是最终的哈希值，即256bit的数字摘要。
一个256-bit的摘要的初始值H0，经过第一个数据块进行运算，得到H1，即完成了第一次迭代。
H1经过第二个数据块得到H2，……，依次处理，最后得到Hn，Hn即为最终的256-bit消息摘要。
将每次迭代进行的映射用$ Map(H_{i-1}) = H_{i} $表示，于是迭代可以更形象的展示为：
![25ec4d1e8c4301a43e1960501075bb9b.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENNote/p472?hash=25ec4d1e8c4301a43e1960501075bb9b)
> 8个哈希初值：这些初值是对自然数中前8个质数（2,3,5,7,11,13,17,19）的平方根的小数部分取前32bit而来。
> 64个常量：和8个哈希初值类似，这些常量是对自然数中前64个质数(2,3,5,7,11,13,17,19,23,29,31,37,41,43,47,53,59,61,67,71,73,79,83,89,97…)的立方根的小数部分取前32bit而来。

图中256-bit的Hi被描述8个小块，这是因为SHA256算法中的最小运算单元称为“字”（Word），一个字是32位。
此外，第一次迭代中，映射的初值设置为前面介绍的8个哈希初值，如下图所示：
![1a0df915b0c4d9439a5265c0fd59d242.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENNote/p472?hash=1a0df915b0c4d9439a5265c0fd59d242)

### 迭代
#### STEP1：构造64个字（word）
break chunk into sixteen 32-bit big-endian words w[0], …, w[15]
对于每一块，将块分解为16个32-bit的big-endian的字，记为w[0], …, w[15]
也就是说，前16个字直接由消息的第i个块分解得到
其余的字由如下迭代公式得到：
![f2098805de36b73a980268f69a5a3462.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENResource/p2450)

#### STEP2：进行64次循环
映射 $ Map(H_{i-1}) = H_{i} $ 包含了64次加密循环
即进行64次加密循环即可完成一次迭代
每次加密循环可以由下图描述：
![2b6b5ad6ec73e40907818601a7ee9a13.png](evernotecid://09243391-321F-42E2-BB9A-CDBE8BED8C36/appyinxiangcom/18710579/ENNote/p472?hash=2b6b5ad6ec73e40907818601a7ee9a13)
图中，ABCDEFGH这8个字（word）在按照一定的规则进行更新，其中
深蓝色方块是事先定义好的非线性逻辑函数，上文已经做过铺垫
红色田字方块代表 mod $ 2^{32} $ addition，即将两个数字加在一起，如果结果大于$ 2^{32}$ ，你必须除以 $ 2^{32} $并找到余数。
ABCDEFGH一开始的初始值分别为$ H_{i-1}(0),H_{i-1}(1),…,H_{i-1}(7) $
Kt是第t个密钥，对应我们上文提到的64个常量。
Wt是本区块产生第t个word。原消息被切成固定长度512-bit的区块，对每一个区块，产生64个word，通过重复运行循环n次对ABCDEFGH这八个字循环加密。
最后一次循环所产生的八个字合起来即是第i个块对应到的散列字符串$ H_{i} $
# 伪代码
```bash
Note: All variables are unsigned 32 bits and wrap modulo 232 when calculating


Initialize variables
(first 32 bits of the fractional parts of the square roots of the first 8 primes 2..19):
h0 := 0x6a09e667
h1 := 0xbb67ae85
h2 := 0x3c6ef372
h3 := 0xa54ff53a
h4 := 0x510e527f
h5 := 0x9b05688c
h6 := 0x1f83d9ab
h7 := 0x5be0cd19


Initialize table of round constants
(first 32 bits of the fractional parts of the cube roots of the first 64 primes 2..311):
k[0..63] :=
   0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5, 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
   0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3, 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
   0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc, 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
   0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7, 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
   0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13, 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
   0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3, 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
   0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5, 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
   0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208, 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2


Pre-processing:
append the bit '1' to the message
append k bits '0', where k is the minimum number >= 0 such that the resulting message
    length (in bits) is congruent to 448(mod 512)
append length of message (before pre-processing), in bits, as 64-bit big-endian integer


Process the message in successive 512-bit chunks:
break message into 512-bit chunks
for each chunk
    break chunk into sixteen 32-bit big-endian words w[0..15]

    Extend the sixteen 32-bit words into sixty-four 32-bit words:
    for i from 16 to 63
        s0 := (w[i-15] rightrotate 7) xor (w[i-15] rightrotate 18) xor(w[i-15] rightshift 3)
        s1 := (w[i-2] rightrotate 17) xor (w[i-2] rightrotate 19) xor(w[i-2] rightshift 10)
        w[i] := w[i-16] + s0 + w[i-7] + s1

    Initialize hash value for this chunk:
    a := h0
    b := h1
    c := h2
    d := h3
    e := h4
    f := h5
    g := h6
    h := h7

    Main loop:
    for i from 0 to 63
        s0 := (a rightrotate 2) xor (a rightrotate 13) xor(a rightrotate 22)
        maj := (a and b) xor (a and c) xor(b and c)
        t2 := s0 + maj
        s1 := (e rightrotate 6) xor (e rightrotate 11) xor(e rightrotate 25)
        ch := (e and f) xor ((not e) and g)
        t1 := h + s1 + ch + k[i] + w[i]
        h := g
        g := f
        f := e
        e := d + t1
        d := c
        c := b
        b := a
        a := t1 + t2

    Add this chunk's hash to result so far:
    h0 := h0 + a
    h1 := h1 + b
    h2 := h2 + c
    h3 := h3 + d
    h4 := h4 + e
    h5 := h5 + f
    h6 := h6 + g
    h7 := h7 + h

Produce the final hash value (big-endian):
digest = hash = h0 append h1 append h2 append h3 append h4 append h5 append h6 append h7
```

# 参考
[SHA256算法原理详解](https://blog.csdn.net/u011583927/article/details/80905740)




