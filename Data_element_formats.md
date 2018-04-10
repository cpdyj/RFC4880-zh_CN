# 3. 数据元素格式
本节介绍OpenPGP使用的数据元素。

## 3.1 标量数字
标量数字是无符号的，并且总是以大端格式存储。 使用n [k]来表示被解释的第k个八位字节，两个八位字节标量的值是（（n [0] << 8）+ n [1]）。 四字节标量的值是（（n [0] << 24）+（n [1] << 16）+（n [2] << 8）+ n [3]）。

## 3.2 多精度整数
多精度整数（也称为MPI）是用于保存大整数的无符号整数，例如用于加密计算的整数。

一个MPI由两部分组成：一个两字节标量作为MPI的长度（以位为单位），后跟一串包含实际整数的字节。  
这些八位组形成一个大端数字; 一个大端的数字可以通过在适当的长度前加上一个MPI来制作成MPI。

例子：（所有的数字都是十六进制的）

> 字节流[00 01 01]表示值为1的MPI。[00 09 01 FF]表示值为511的MPI。  
> 其他规则：  
> * MPI的大小是 ((MPI.length + 7) / 8) + 2个八比特组。  
> * MPI的长度字段描述从其最重要的非零位开始的长度。 因此，MPI [00 20 01] 不是正确的。 它应该是[00 01 01]。  
> * MPI的未使用位必须为零。  
> * 还要注意，当MPI被加密时，长度是指明文MPI。 它的密文可能是格式不正确的。  

## 3.3 秘钥ID
密钥ID是标识密钥的八字节标量。 实现**不应该**假定Key ID是唯一的。 下面的“增强密钥格式”部分介绍了如何形成密钥ID。

## 3.4 文本
除非另有规定，否则文本的字符集是Unicode [ISO 10646]的UTF-8 [RFC 3629]编码。

## 3.5 时间字段
时间字段是一个无符号的四字节数字，包含自1970年1月1日午夜以来经过的秒数。

## 3.6 密钥环
密钥环是文件或数据库中一个或多个密钥的集合。  
传统上，密钥环只是密钥的顺序列表，但也可以是任何合适的数据库。讨论密钥环或其他数据库的细节超出了本标准的范围。

## 3.7 String-to-Key (S2K) 规范
String-to-Key（S2K）标识符用于将密码字符串转换为对称密钥加密/解密密钥。它们目前用于两个地方：加密私钥环中私钥的秘密部分，并将密码转换为加密密钥以获得对称加密的消息。

### 3.7.1 String-to-Key (S2K) 标识符类型
S2K标识符当前支持3种类型，还有一些保留值：

|ID|S2K 类型|
|:--|:--------|
|0|简单 S2K|
|1|加盐的 S2K|
|2|保留值|
|3|迭代的加盐的S2K|
|100 to 110|私有/实验性 S2K|

这些内容的介绍在章节 3.7.1.1 - 3.7.1.3

#### 3.7.1.1 简单 S2K
直接对字符串进行哈希来生成key数据。这里展示哈希是如何完成的。（译注：Octet = 8bit 字节）

|||
|:-|:-|
|Octet 0|0x00|
|Ocect 1|哈希算法|

简单 S2K 哈希密码得到会话秘钥。完成此操作的方式取决于会话密钥的大小（取决于所使用的密码）和算法的输出。如果散列大小大于会话密钥大小，则使用散列的高位（最左边）八位字节作为密钥。

如果散列大小小于密钥大小，则会创建多个散列上下文实例 - 足以生成所需的密钥数据。 这些实例预先加(preloaded)了0,1,2，......八位字节的零（也就是说，第一个实例没有预加载，第二个实例预加载了1个八位字节的零，第三个预加载了两个八位字节的零， 等等）。(译注：没看懂，可能理解有误，待修正)

> 原文：If the hash size is less than the key size, multiple instances of the hash context are created -- enough to produce the required key data. These instances are preloaded with 0, 1, 2, ... octets of zeros (that is to say, the first instance has no preloading, the second gets preloaded with 1 octet of zero, the third is preloaded with two octets of zeros, and so forth).

由于数据是散列的，因此每个散列上下文都会独立给出。 由于上下文的初始化方式不同，它们将分别产生不同的散列输出。 一旦密码被散列，来自多个散列的输出数据被连接在一起，首先是最左边的散列，以产生密钥数据，右边的任何多余的八位字节被丢弃。

#### 3.7.1.2 加盐的 S2K
在 S2K 标识符中加入了“盐”值 -- 任意的一些数据 -- 与密码字符串一起被散列，用于阻止字典攻击。

|:-|:-|
|Octet 0|0x03|
|Octet 1|哈希算法|
|Octet 2-9|8-Octet “盐”值|
|Octet 10|count|

count使用以下公式编码至一个字节：

    #define EXPBIAS 6
        count = ((Int32)16 + (c & 15)) << ((c >> 4) + EXPBIAS);

上述公式以C语言表示，其中“Int32”是32位整数的类型，变量“c”是编码计数，Octet 10。

Iterated-Salted S2K hashes the passphrase and salt data multiple times.  The total number of octets to be hashed is specified in the encoded count in the S2K specifier.  Note that the resulting count value is an octet count of how many octets will be hashed, not an iteration count.

Initially, one or more hash contexts are set up as with the other S2K algorithms, depending on how many octets of key data are needed. Then the salt, followed by the passphrase data, is repeatedly hashed until the number of octets specified by the octet count has been hashed.  The one exception is that if the octet count is less than the size of the salt plus passphrase, the full salt plus passphrase will be hashed even though that is greater than the octet count. After the hashing is done, the data is unloaded from the hash context(s) as with the other S2K algorithms.


[RFC 3629]: https://tools.ietf.org/html/rfc3629 "\"UTF-8, a transformation format of ISO 10646\""
[ISO 10646]: https://tools.ietf.org/html/rfc4880#ref-ISO10646
