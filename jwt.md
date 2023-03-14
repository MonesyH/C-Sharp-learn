# 一、前置知识
## 1. jwt（json web token）
生成的token示例：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJ1bmlxdWVfbmFtZSI6ImFkbWluIiwiaWQiOiIxIiwicm9sZSI6ImFkbWluaXN0cmF0b3IiLCJuYmYiOjE2Nzg3NTg5MjUsImV4cCI6MTY3OTM2MzcyNSwiaWF0IjoxNjc4NzU4OTI1fQ.
GxxPDrg-WYMSVVEze2wm1r7hJ3iq06Qo5eQWyUZP25Y
```
有上面能看到token通过 **.** 点 来分割不同的三个部分：
* 头部（Header）：主要用于说明token类型和签名算法。
```
{ 
    "alg": "HS256", //alg：签名算法，这里是 HMAC SHA256
    "typ": "JWT",  //typ：token类型，这里是JWT
}
```
接着对其进行Base64Url编码，即可获取到Token的第一部分：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```
* 载荷（Payload）：是核心，主要用于存储声明信息，如token签发者、用户Id、用户角色等。
```
{
	"iss": "xxx",
	"iat": xxx,
	"nbf": xxx,
	"exp": xxx,
	"aud": "xxx",
	"name": "xxx"
}
```
   对Payload（记得去除所有换行和空格）进行Base64Url编码，即可获取到Token的第2部分：
  ```
eyJ1bmlxdWVfbmFtZSI6ImFkbWluIiwiaWQiOiIxIiwicm9sZSI6ImFkbWluaXN0cmF0b3IiLCJuYmYiOjE2Nzg3NTg5MjUsImV4cCI6MTY3OTM2MzcyNSwiaWF0IjoxNjc4NzU4OTI1fQ
  ```
* 签名（Signature）：主要用于防止token被篡改。当服务端获取到token时，会按照如下算法计算签名，若计算出的与token中的签名一致，才认为token没有被篡改。

  签名算法：
  1. 先将Header和Payload通过点（.）连接起来，即Base64Url编码的Header.Base64Url编码的Payload，记为 text
  2. 然后使用Header中指明的签名算法对text进行加密，得到一个二进制数组，记为 signBytes
  3. 最后对 signBytes 进行Base64Url编码，得到signature，即token的第三部分:
  ```
  GxxPDrg-WYMSVVEze2wm1r7hJ3iq06Qo5eQWyUZP25Y
  ```

## 2. claim

## 3. 加密算法
分为对称加密以及非对称加密
