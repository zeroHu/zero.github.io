---
title: thirdAPISecret
date: 2024-03-03 10:33:29
tags: api
---
### 与第三方通信API
#### 方案一，提供两个接口（api/token, /api/{code}）第三方可以分为前端后后端来调用. 
初始化机构数据关键字段：appId、appSecret、RSA公钥、私钥。
```
// =========/api/token========
定义：获取accessToken

接口设计如下：

参数：
  body: {
    appId: { appId },
    appSecretRSAEncryptData: { rsa 公钥加密appSecret后的值 }
  }

返回：
  {
    token: { 一串数字 }
  }
返回过程判断：拿到参数后，取appId，对比数据库是否存在，存在则用对应的rsa解密appSecretRSAEncryptData字段的值，然后对比数据库appid对应存储的appSecret值是否一致。


// =========/api/{code}========
定义：业务接口

接口设计如下：

参数：
  headers: {
    timestamp: {时间戳},
    salt: {随机16位字符A用RSA公钥加密后},
    sign: {token + timestamp + 明文body(B)} - MD5 加密
    token: {从/api/token返回的token内容}
  }

  body: {
    将原业务内容（B）用随机16位字符A 用AES加密
  }



返回：
  {
    接口业务字段对应内容
  }
返回过程判断：
(1): 校验timestamp时间是否超范围
(2): 校验token是否有效
(3): 根据salt用RSA私钥解出随机16位字符A
(4): 根据A 用AES 解密body得到原业务内容（B）
(5): 根据token + timestamp + B 使用MD5 加密后与 sign对比是否一致。确保内容没有被篡改


```

#### 方案二，提供一个接口（/api/{code}）第三方后端调用接口获取数据
初始化机构数据关键字段：appId、appSecret
```
// =========/api/{code}========
定义：业务接口


参数：
header: {
  salt: 用随机16位字符A 用appSecret 采用AES 加密后
  sign: [appId + timestamp + body] --> MD5 加密
  timestamp: 时间戳
}

body: {
  appId: { appId },
  data: { 将原业务内容（B）用随机16位字符A 用AES加密 }
}

返回：
  {
    接口业务字段对应内容
  }
返回过程判断：
(1): 校验timestamp 是否符合时间范围
(2): 校验appId是否有效，查出appId对应的appSecret(C)
(3): 根据C解出salt的原随机数A
(4): 用随机数A解开原body的data字段，也就是业务内容B
(5): 根据sign的字段 appId + timestamp + body 通过MD5加密对比sign 可以查看是否有效
```

