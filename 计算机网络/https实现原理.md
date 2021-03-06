## 概述

都知道http + ssl 构成了https，也对相关概念有个印象如SSL，非对称加密，CA证书等

本文将着重介绍下面三个问题

- 为什么用了 HTTPS 就是安全的？

- HTTPS 的底层原理如何实现？

- 用了 HTTPS 就一定安全吗？



## HTTPS实现原理

https之所以是安全的，是因为https协议会对传输的数据进行加密，而加密过程中使用了 **非对称加密**，但其实，**HTTPS在内容传输上的加密是使用对称加密的，非对称加密只作用在证书验证阶段**



![image-20210318110554091](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210318110554091.png)



### 证书验证阶段

1. 浏览器发起https请求
2. 服务端返回https证书
3. 客户端验证证书是否合法，若不合法则提示告警



### 数据传输阶段

1. 当证书验证合法后，在本地生成随机数
2. 通过公钥加密随机数，并把加密后的随机数传输到服务端
3. 服务端通过私钥对随机数进行解密
4. 服务端通过客户端传入的随机数构造对称加密算法，对返回结果内容进行加密后传输



## 常见问题



### 为何数据传输时使用对称加密

首先，非对称加密的加解密效率是非常低的，而 http 的应用场景中通常端与端之间存在大量的交互，非对称加密的效率是无法接受的；



另外，**在 HTTPS 的场景中只有服务端保存了私钥**，一对公私钥只能实现单向的加解密，所以 HTTPS 中内容传输加密采取的是对称加密，而不是非对称加密。



### 为何需要CA机构颁发证书

HTTP 协议被认为不安全是因为传输过程容易被监听者勾线监听、伪造服务器，而 HTTPS 协议主要解决的便是网络传输的安全性问题。



若https的证书可以由任何人随意制作，这会带来“中间人攻击”问题



![image-20210318125647432](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210318125647432.png)



图中可见，虽然使用的是https进行了加密，但由于证书是其他中间人颁发的，其传输内容就可以被其完全窃取篡改



### 浏览器如何保证CA证书的合法性

一个正常的证书中，正常来说包含以下信息

- 颁发机构信息
- 公钥
- 公司信息
- 域名
- 有效期
- 指纹

- 。。。等等

首先，权威机构是要有认证的，**不是随便一个机构都有资格颁发证书**，不然也不叫做权威机构。另外，证书的可信性基于信任制，权威机构需要对其颁发的证书进行信用背书，**只要是权威机构生成的证书，我们就认为是合法的**。所以权威机构会对申请者的信息进行审核，不同等级的权威机构对审核的要求也不一样，于是证书也分为免费的、便宜的和贵的。



#### 浏览器如何验证证书合法性

当浏览器发起https请求时，服务器会返回网站的SSL证书，浏览器需要对证书做以下验证：

1. 验证域名、有效期等信息是否正确。证书上都包含这些信息，可以比较容易完成该部分验证
2. 判断证书是否合法。每份签发证书都可以根据验证链接查找到对应的根证书，操作系统、浏览器会在本地存储权威机构的根证书，利用本地根证书可以对对应机构签发证书完成来源验证

![image-20210318140731613](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210318140731613.png)

3. 判断证书是否被篡改，需要与CA服务器进行校验
4. 判断证书是否被吊销。通过CRL(Certificate Revocation List 证书注销列表)和OCSP（Online Certificates Status Protocol 在线证书状态协议）实现，其中OSCP可以用于第三步中，以减少与CA服务器的交互，提高验证效率



只有当上述步骤都满足的情况下，浏览器才认为证书是合法的



### 是否只有认证机构才能生成证书

从技术上谁都可以生成证书，只有有证书就能完成https传输



### 本地随机数被窃取怎么办

***如果随机数被窃取，第三方就可以用此随机数来模拟客户端向服务端发送请求***



证书验证是使用非对称加密完成的，但是传输过程是采用对称加密，而其中对称加密算法中重要的随机数由本地生成且存储于本地



**其实 HTTPS 并不包含对随机数的安全保证**，HTTPS 保证的只是传输过程安全，而随机数存储于本地，本地的安全属于另一安全范畴，应对的措施有安装杀毒软件、反木马、浏览器升级修复漏洞等。



## 参考文献

https://www.cnblogs.com/zbligang/articles/11987029.html



