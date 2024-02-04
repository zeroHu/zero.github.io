---
title: jwt
date: 2024-02-04 11:14:45
tags: jwt
keywords: jwt auth jwe jws 用户信息交换认证
---
### JWT家族

#### JWT(json web token)
* 用途是：在网络上安全传输的开放标准，通常用于身份验证和信息交换
* 结构是：Header(头部:包含关于生成jwt的原信息，例如使用的加密算法和令牌的类型).Payload(负载：关于实体/用户的数据信息).Signature(签名：使用头部和负载以及秘密密钥生成的签名)

```
# jwt 示例
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.  
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.  
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ 

```



#### JWS(json web Signature)
* 其实我们大部分正常用于身份校验的JWT 就是 JWS，也就是头部一定要声明算法，然后经过了秘钥加密头部、负载后生成了Signature的内容。

#### JWE(json web encryption)

JWS 仅仅是对声明(claims)作了签名，保证了其不被篡改，但是其 payload(中段负载) 信息是暴露的。也就是 JWS 仅仅能保证数据的完整性而不能保证数据不被泄露。它不适合传递敏感数据。JWE 的出现就是为了解决这个问题的。具体的可以看下图：

JWE 生成步骤
* 与JWS Header相同标志加密算法和类型
* 生成一个随机的256位的 content encryption key（CEK）
* 使用 rsa(pkcs1)算法对token接收者的公钥对CEK进行加密，以生成JWE加密秘钥
* 生成随机128位JWE初始化向量
* 将附加身份平局参数加入Header
* 使用CEK作为加密秘钥，JWT初始化向量 和 上面的附加身份凭据参数，使用AES(cbc_hmac_sha_256)算法对纯文本执行身份证加密，生成密文，同时
会随之生成一个128位的Authentication Tag
* 将生成的五段加密串使用Base64进行编码，然后使用"."按照顺序连接起来就是组成Token

![服务器图](Snipaste_2024-02-04_15-41-12.png)


可见，JWE的计算过程相对繁琐，不够轻量级，因此适合与数据传输而非token认证，但该协议也足够安全可靠，用简短字符串描述了传输内容，兼顾数据的安全性与完整性。


