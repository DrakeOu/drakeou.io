# https如何保证安全

> 转载备份--原文连接   [https如何保证安全](https://zhuanlan.zhihu.com/p/110216210)

惯例，上 Wiki，什么是 HTTPS?

> **超文本传输安全协议**（英语：**H**yper**T**ext **T**ransfer **P**rotocol **S**ecure，缩写：**HTTPS**；常称为HTTP over TLS、HTTP over SSL或HTTP Secure）是一种通过[计算机网络](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E8%A8%88%E7%AE%97%E6%A9%9F%E7%B6%B2%E7%B5%A1)进行安全通信的[传输协议](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E7%B6%B2%E8%B7%AF%E5%82%B3%E8%BC%B8%E5%8D%94%E5%AE%9A)。HTTPS经由[HTTP](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/HTTP)进行通信，但利用[SSL/TLS](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E5%B1%82%E5%AE%89%E5%85%A8)来[加密](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%8A%A0%E5%AF%86)数据包。HTTPS开发的主要目的，是提供对[网站](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E7%B6%B2%E7%AB%99)服务器的[身份认证](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E8%BA%AB%E4%BB%BD%E9%AA%8C%E8%AF%81)，保护交换数据的隐私与[完整性](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%AE%8C%E6%95%B4%E6%80%A7)。这个协议由[网景](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E7%B6%B2%E6%99%AF)公司（Netscape）在1994年首次提出，随后扩展到[互联网](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E7%B6%B2%E9%9A%9B%E7%B6%B2%E8%B7%AF)上。

可以看出，Https 之所以安全，是经过了加密，那么新的问题又来了，SSL 和 TLS 分别是什么？是怎么加密的？

## SSL 和 TLS

SSL：（Secure Socket Layer，安全套接字层），位于可靠的面向连接的**网络层协议**和**应用层协议**之间的一种协议层。SSL通过互相认证、使用数字签名确保完整性、使用加密确保私密性，以实现客户端和服务器之间的安全通讯。该协议由两层组成：SSL记录协议和SSL握手协议。

TLS：(Transport Layer Security，传输层安全协议)，用于两个应用程序之间提供保密性和数据完整性。该协议由两层组成：TLS 记录协议和 TLS 握手协议。

SSL 和 TLS 是一种能够在服务器，machines 和通过网络运行的应用程序（列如，客户端连接到 web 服务器）之间提供身份认证和数据加密的加密协议。SSL是TLS的前世。多年来，新版本的发布用来解决漏洞，提供更强大支持，更安全的密码套件和算法。

SSL最初是由 Netscape 开发的，早在1995年以SSL 2.0的方式发布（1.0从未对公众发布）。在一些漏洞被发现之后，版本2.0在1996年很快被3.0所取代。注意：版本2.0和3.0有时会写成 SSLv2 和 SSLv3。

[TLS](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc2246)以SSL 3.0为基础于1999年作为SSL的新版本推出。

### 应该选 SSL 还是 TLS？

SSL2.0和SSL3.0已经被IEFT组织废弃（分别在2011年，2015年）多年来，在被废弃的SSL协议中一直存在漏洞并被发现 (e.g. [POODLE](https://link.zhihu.com/?target=https%3A//www.globalsign.com/en/blog/poodle-vulnerability-in-ssl-30/), [DROWN](https://link.zhihu.com/?target=https%3A//www.globalsign.com/en/blog/drown-attack-sslv2/))。大多数现代浏览器遇到使用废弃协议的 web 服务时，会降低用户体验（红线穿过挂锁标志或者https表示警告）来表现。因为这些原因，你应该在服务端禁止使用 SSL 协议，仅仅保留 TLS 协议开启。

## 为什么要用 TLS/SSL？

要想规避风险，要先知道风险点在哪。我们要知道 HTTPS 想要解决哪些问题，具体通过什么手段解决的。

常规的 HTTP 通信，有以下的问题。

1. **窃听风险**（eavesdropping）：第三方可以获知通信内容。
2. **篡改风险**（tampering）：第三方可以修改通信内容。
3. **冒充风险**（pretending）：第三方可以冒充他人身份参与通信。

因此，SSL/TLS 协议就是为了解决这三大风险而设计的，希望达到：

1. 所有信息都是**加密传播**，第三方无法窃听。
2. 具有**校验机制**，一旦被篡改，通信双方会立刻发现。
3. 配备**身份证书**，防止身份被冒充。

## 运行过程

SSL/TLS 协议的基本思路是采用[公钥加密法](https://link.zhihu.com/?target=http%3A//en.wikipedia.org/wiki/Public-key_cryptography)，也就是说，客户端先向服务器端索要公钥，然后用公钥加密信息，服务器收到密文后，用自己的私钥解密。

> **公开密钥密码学**（英语：**Public-key cryptography**）也称**非对称式密码学**（英语：**Asymmetric cryptography**）是[密码学](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%AF%86%E7%A2%BC%E5%AD%B8)的一种[算法](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E6%BC%94%E7%AE%97%E6%B3%95)，它需要两个[密钥](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%AF%86%E9%92%A5)，一个是公开密钥，另一个是私有密钥；公钥用作加密，私钥则用作解密。使用其中一个密钥把[明文](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E6%98%8E%E6%96%87)加密后所得的[密文](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%AF%86%E6%96%87)，只能用相对应的另一个密钥才能解密得到原本的明文；连最初用来加密的密钥也不能用作解密。由于加密和解密需要两个不同的密钥，故被称为非对称加密；不同于加密和解密都使用同一个密钥的[对称加密](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%AF%B9%E7%A7%B0%E5%8A%A0%E5%AF%86)。其中一个密钥可以公开，称为公钥，可任意向外发布；不公开的密钥为私钥，必须由用户自行严格秘密保管，绝不透过任何途径向任何人提供，也不会透露给被信任的要通信的另一方。

简单来说，非对称加密就是两个密码，公钥交给对方，私钥自己藏好，把想要传输的信息用私钥加密，然后交给对方，对方用公钥解密。这里仍然有两个问题：

### 如何保证公钥不被篡改？

解决方法：将公钥放在[数字证书](https://link.zhihu.com/?target=http%3A//en.wikipedia.org/wiki/Digital_certificate)中。只要证书是可信的，公钥就是可信的。

那么怎么验证数字证书可不可信呐？答案是 CA 来给你鉴定。

CA（Certificate Authority）是证书颁发机构，即颁发数字证书的机构，是负责发放和管理数字证书的权威机构。业界比较大的 CA 机构会被客户端软件广泛承认。比如一个浏览器承认某个 CA 的信用，就把这家机构的根证书（root certificate）内置在客户端当中，作为信任链的源头，后续的其他数字证书只要通过根证书的认证，就会认定为是可信的。根证书如何来判断是否信任某个数字证书？就得从数字证书的结构说起。

### 数字证书的认证

有过部署 HTTPS 服务经历的同学都知道，将自己的网站改为 HTTPS 需要做的第一步就是申请一个数字证书放在 web 服务器下面。数字证书主要包括以下内容：

- 证书颁发机构的名称
- 证书持有者公钥
- 证书签名用到的Hash算法
- 有效期等等其他信息

这些内容都是明文，然后将明文通过摘要算法获取 hash，通过**证书颁发机构**的私钥进行非对称加密，得到签名信息，这个步骤叫做数字签名。明文+hash+数字签名共同组成了这个数字证书。

现在客户端拿到了这个数字证书，使用证书里说明的 hash 算法对明文进行摘要运算，得到的 hash 值应该和证书里是一样的。然后客户端会查询这个证书的颁发机构名称，从自己内置的信任列表里找该机构的证书，这里有两种情况：

1. 找到了，说明是自己信任的 CA 颁发的，从该机构的数字证书中可以找到证书的公钥，用这个公钥解密之前拿到的数字证书，如果能成功解密，证明证书可信。会添加到信任列表。
2. 没找到，也不一定说明证书是假的，因为数字证书是一条信任链，A 信任 B，B 信任 C，就算内置了 A ，也不能直接用 A 验证 C。所幸 C 的证书内容里会包含 B 的名称和地址，浏览器会自动去找 B 的证书，然后再看看 B 是否可信，直到达到了根节点。或者到了根节点仍然不可信，整个信任链就没有建立，所有证书被浏览器判断为不可信。

如此以来，数字证书的认证就完成了，有两个结果，浏览器信任证书和不信任证书。前者一般会有标示，比如 Chrome 浏览器会在地址栏有个小锁。后者就会提示连接有风险。



![img](https://pic2.zhimg.com/80/v2-8b7d0061b37140b772755daa665474b1_720w.png)





![img](https://pic3.zhimg.com/80/v2-ccceb663b8c8f0d995ec10a5e68bb846_720w.jpg)



这一步完成之后，客户端只是拿到了一个准确、可信的服务端的公钥。而实际上的数据传输还没开始。

客户端拿到公钥之后（或之前，我也不确定），也会把自己的公钥发送过去，双方会运行 Diffie Hellman 算法，简称 DH 算法。DH 算法的作用是这样：通过二者的公钥推导出一个个一样的密码出来。通俗地说：双方会协商一个master-key，这个master-key 不会在网络上传输、交换，它们独立计算出来的，其值是相同的，只有它们自己双方知道。

然后以master-key 推导出 session-key，用于双方 SSL 数据流的加密/解密，采用对称加密，保证数据不被偷窥，加密算法一般用 AES。

同时以 master-key 推导出 hash-key，用于数据完整性检查（Integrity Check Verification）的加密密钥，HASH 算法一般有：MD5、SHA。

为什么要搞这么麻烦呐？因为非对称加密算法很消耗资源，所以只有前置的互相认证的步骤用非对称加密，后续的步骤都用更省资源的对称加密。

到这里，HTTPS 的大致流程就完成了，当然实际上的步骤会更加复杂，客户端和服务端的连接还会有随机数，为了减少资源消耗还会有一些缓存机制。

再回顾一下整个简化的流程：

1. 客户端请求服务端的数字证书。
2. 客户端根据内置的根证书校验服务端发来的证书。
3. 如果证书可信，客户端把自己的公钥发给服务端，然后用双方的公钥推导出一个会话密钥。
4. 后续的请求内容都用会话密钥进行加密和解密。

## 扩展

截了个知乎的证书，可以看到是三级的信任链，知乎的证书签发机构是`GeoTrust RSA CA 2018`，而 GeoTrust 的签发机构是`DigiCert Global Root CA`，就到根证书了。



![img](https://pic2.zhimg.com/80/v2-8028db10f79af1d4e39d83d7fe4fbd8d_720w.jpg)





![img](https://pic1.zhimg.com/80/v2-d20a5dcd8d1c36b8ed491f5a8d2ae738_720w.jpg)



另外注意风险：

1. 把公钥交给对方的时候要注意安全，公钥有可能被劫持。
2. 数字证书的认证步骤也不是绝对安全，有“中间人攻击”。当然这个也需要一定条件，就不细说了。

参考文章：

[TLS与SSL之间关系](https://link.zhihu.com/?target=https%3A//juejin.im/post/5b213a0ae51d4506d47dff0d)

[SSL/TLS协议运行机制的概述](https://link.zhihu.com/?target=https%3A//www.ruanyifeng.com/blog/2014/02/ssl_tls.html)

[详解https是如何确保安全的？](https://zhuanlan.zhihu.com/[http://www.wxtlife.com/2016/03/27/详解https是如何确保安全的？/](http://www.wxtlife.com/2016/03/27/详解https是如何确保安全的？/))

[密钥交换算法](https://link.zhihu.com/?target=https%3A//www.liaoxuefeng.com/wiki/1252599548343744/1304227905273889)