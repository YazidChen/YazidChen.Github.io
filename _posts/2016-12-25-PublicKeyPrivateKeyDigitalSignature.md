---
layout: post
title:  "公钥、私钥、数字签名"
categories: Security
description: 公钥、私钥、数字签名之间的联系。
keywords: 公钥, 私钥, 数字签名, public key, private key, digital signature
---
#  Public key, Private key, Digital signature #

## 公钥私钥

### 公钥加密 ###

假设有两个字母，**S**和**G**。我喜欢字母**S**，我将字母**S**当成私钥自己保留，此时别人并不知道我的私钥就是字母**S**。而字母**G**，我把它当成公钥，告诉了小明。此时小明知道我的公钥是字母**G**。

某天，小明要给我写一封重要的信，他就给这封信用我的公钥字母**G**加密，然后发给我。我看到信后，用公字母**S**进行解密，便看到了信的真实内容。

如果这封信路上被劫了，截获的人没有我的私钥，所以他解不开信件。

![](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191025094845.png?x-oss-process=style/Watermark)

这便是，**公钥加密，私钥解密**。

### 私钥数字签名 ###

假如我用私钥**S**加密，但是我的公钥**G**是暴露在外的，结果所有人都可以用公钥**G**解密，结果加密的内容，所有人都知道了。

那用私钥加密到底有什么作用？

假如我写了一封信给小明，小明看到信后怎么知道信就是我写的？他表示怀疑。那么我就用我的私钥**S**给一段内容**1**加密。然后附在信件里，发给小明，然后告诉小明信件里附了一段加密的内容，真实内容是**1**。小明收到信后，看到了加密的内容，他用我给的公钥**G**解密，发现果然是内容**1**，于是这封信必然是我写的。

当然，实际的会复杂一些，比如用Hash函数对信件本身做处理,得到 Digest，再加密成 Signature 。

![](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191025094926.png?x-oss-process=style/Watermark)

这便是，**私钥数字签名，公钥验证**。

### 数字证书 ###

有天隔壁老王偷偷潜入小明家，将我给小明的公钥**G**换成了他的公钥**W**，此后，老王变可以冒充我，用他自己的私钥做成数字签名，写信给小明。

后来，小明觉得不对劲，便要求我去找**证书中心（Certificate Authority，简称CA）**，为公钥**G**做认证。证书中心用自己的私钥，对我的公钥和一些相关的信息一起加密，生成**数字证书（Digital Certificate）**。

我拿到**数字证书**后，就可以放心了，以后写信个小明，只要在**数字签名**的同时，再加上**数字证书**就行了。

小明收到我的信后，用CA的公钥解开数字证书，就可以拿到我的真实公钥，之后小明就能验签了。

![](https://yazid-public.oss-cn-shenzhen.aliyuncs.com/blog/images/20191025094947.png?x-oss-process=style/Watermark)


