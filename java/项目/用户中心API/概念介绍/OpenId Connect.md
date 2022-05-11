# OpenID Connect

[OpenId协议官方文档](https://openid.net/specs/openid-connect-core-1_0.html)

OpenID Connect 以下简称 OpenId

### 介绍

* OAuth2 只定义了获取和使用访问令牌来访问资源的机制，但没有定义提供标识信息（用户信息）的标准方法
* OpenId是OAuth 2.0授权过程的扩展，OpenId实现了身份验证。客户端通过在授权请求中包含 openid 范围（scope）值来获取身份验证的信息，返回结果的JSON中包含名为ID Token 的健，值为JWT格式，解析后内容为用户的信息。
* 比OAuth2多一个 UserInfo 标准接口
* 比OAuth2多一个ID Token
* OpenID Connect 是对OAuth2的 Authorize 接口做了扩展，在请求的 `scope` 参数里指定`openid`即视为启动OpenId Connect 认证流程，同时扩展了 `response_type` 参数，可以指定 `id_token`
* 比OAuth2多一个 Hybrid 授权流程

### 概念

##### ID Token

是一个JWT，包含认证信息和一些常用的用户信息

##### UserInfo 标准接口

客户端通过提供Access Token 查询用户详细信息

### 流程

简略流程图，不同模式流程有差异

```
+--------+                                   +--------+
|        |                                   |        |
|        |---------(1) 认证请求      -------->|        |
|        |                                   |        |
|        |  +--------+                       |        |
|        |  |        |                       |        |
|        |  |  用户 - |<--(2) 跳转到登录页  -->|        |
|        |  |  浏览器 |                       |        |
| 客户端  |  |        |                       |用户中心 |
|        |  +--------+                       |        |
|        |                                   |        |
|        |<--------(3) AuthN Response--------|        |
|        |                                   |        |
|        |---------(4) UserInfo Request----->|        |
|        |                                   |        |
|        |<--------(5) UserInfo Response-----|        |
|        |                                   |        |
+--------+                                   +--------+
```

### 认证

`response_type` 参数值决定使用哪种模式

##### 区别如下：

------

| Property                                                    | Authorization Code Flow | Implicit Flow | Hybrid Flow |
| ----------------------------------------------------------- | ----------------------- | ------------- | ----------- |
| 从 Authorization 端点返回所有令牌（ID Token、Access Token） | no                      | yes           | no          |
| 从 Token 端点返回所有令牌                                   | yes                     | no            | no          |
| 不返回 Token 而是返回 code（授权码）                        | yes                     | no            | no          |
| 客户端可以被认证                                            | yes                     | no            | yes         |
| 可以刷新令牌                                                | yes                     | no            | yes         |
| 一次请求获取所有Token                                       | no                      | yes           | no          |
|                                                             |                         |               |             |



### Hyrid 授权流程

对比 OAuth2 的 Implicit 授权会更安全，需要验证的项也越多

##### 混合流的步骤如下:

* 客户端准备一个身份验证请求，其中包含所需的请求参数。
* 客户端将请求发送到授权服务器。
* 授权服务器验证最终用户。
* 授权服务器获得最终用户的同意/授权。
* 授权服务器使用授权代码和一个或多个附加参数(取决于响应类型)将最终用户发送回客户端。
* 客户端使用令牌端点上的授权代码请求响应。
* 客户端接收响应，响应主体中包含ID令牌和访问令牌。
* 客户端验证ID令牌并检索最终用户的主题标识符。

流程和授权码流程一致

##### 与授权码流程的区别：

`response_type` 参数指定了多个值，`code id_token`, `code token`, 或者 `code id_token token`

| response_type       | access_token | id_token | code |
| ------------------- | ------------ | -------- | ---- |
| code id_token       |              | 返回     | 返回 |
| code token          | 返回         |          | 返回 |
| code id_token token | 返回         | 返回     | 返回 |













