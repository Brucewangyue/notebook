# Authorization Code

基于B/S架构的应用系统接入，需要有后台交互


### 时序图

![](../../resource/code flow.jpg)

##### 请求授权码

```http
GET /connect/authorize?

    client_id=6731de76-14a6-49ae-97bc-6eba6914391e
    &response_type=code
    &redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
    &response_mode=query
    &scope=openid%20offline_access%20https%3A%2F%2Fgraph.microsoft.com%2Fmail.read
    &state=12345
```

| 参数            | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| * `client_id`   | 分配给你的应用的应用程序（客户端）ID。                       |
| * `response_type` | 必须是 `code` 。                               |
| * `redirect_uri`  | 应用的 redirect_uri，应用可向其发送及从其接收身份验证响应。 |
| * `scope`         | 空格分隔列表。 对于请求的 `/authorize` 段，这可以涵盖多个资源，从而允许应用获得你要调用的多个 Web API 的同意。 |
| * `state`         | 同样随令牌响应返回的请求中所包含的值。 可以是想要的任何内容的字符串。 随机生成的唯一值通常用于 [防止跨站点请求伪造攻击](https://tools.ietf.org/html/rfc6749#section-10.12)。 在发出身份验证请求出现之前，此值对有关用户在应用中的状态的信息（例如前面所在的页面或视图）进行编码。 |

##### 响应

```http
  HTTP/1.1 302 Found
     Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
               &state=xyz
```

| 参数    | 说明                                                         |
| :------ | :----------------------------------------------------------- |
| `code`  | 应用请求的 authorization_code。 应用可以使用授权代码请求目标资源的访问令牌。 Authorization_codes 的生存期较短， 10 分钟后即过期。 |
| `state` | 如果请求中包含 state 参数，响应中就应该出现相同的值。 应用应该验证请求和响应中的 state 值是否完全相同。 |



##### 请求令牌

```http
 POST /connect/token HTTP/1.1
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded

	client_id=6731de76-14a6-49ae-97bc-6eba6914391e
     &grant_type=authorization_code
     &code=SplxlOBeZQQYbYS6WxSbIA
     &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
     &client_secret=JqQX2PNo9bpM0uEihUPzyrh    // NOTE: 只有web应用程序才需要. secret需要url编码
```
| 参数            | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| *`client_id`     |  分配给你的应用的应用程序（客户端）ID。                       |
| *`grant_type`    | 必须是授权代码流的 `authorization_code` 。                   |
| `scope`         | 范围的空格分隔列表。 范围必须全部来自单个资源，以及 OIDC范围（`profile`、`openid`、`email`）。 |
| *`code`          |  在流的第一个阶段中获取的 authorization_code。                |
| *`redirect_uri`  |  用于获取 authorization_code 的相同 redirect_uri 值。         |
| *`client_secret` |  在发送client_secret之前必须对其进行 URL 编码。 有关 URI 编码的详细信息，请参阅 [URI 常规语法规范](https://tools.ietf.org/html/rfc3986#page-12)。 |


##### 响应

```http
 HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache

     {
        "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...",
        "token_type": "Bearer",
        "expires_in": 3599,
        "scope": "API1.read API1.write API2",
        "refresh_token": "AwABAAAAvPM1KaPlrEqdFSBzjqfTGAMxZGUTdM0t4B4...",
        "id_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJhdWQiOiIyZDRkMTFhMi1mODE0LTQ2YTctOD...",
    }
```

| 参数            | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| `access_token`  | 请求的访问令牌。 应用可以使用此令牌验证受保护的资源，例如 Web API。 |
| `token_type`    | 指示令牌类型值。 唯一支持的类型是 Bearer                     |
| `expires_in`    | 访问令牌的有效期（以秒为单位）。                             |
| `scope`         | access_token 的有效范围。 可选 - 这是非标准的，如果省略，令牌将用于流的初始阶段中所请求的范围。 |
| `refresh_token` | OAuth 2.0 刷新令牌。 应用可以使用此令牌，在当前的访问令牌过期之后获取其他访问令牌。 Refresh_tokens 的生存期很长，而且可以用于延长保留资源访问权限的时间。 |
| `id_token`      | JSON Web 令牌 (JWT)。 应用可以解码此令牌的段，以请求已登录用户的相关信息。 应用可以缓存并显示值，机密客户端可以使用这些值进行授权。 **注意：** 仅当已请求 `openid` 作用域时提供。 |



### 刷新令牌

##### 请求

```http
POST /connect/token HTTP/1.1
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded

     grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```

| 参数            | 说明                                                         |
| :-------------- | :-------------------- | :----------------------------------------------------------- |
| `grant_type`    | 必须是授权代码流的此阶段的 `refresh_token` 。                |
| `refresh_token` | 在流的第二个阶段获取的 refresh_token。                       |

##### 响应

```json
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...",
    "token_type": "Bearer",
    "expires_in": 3599,
    "scope": "https%3A%2F%2Fgraph.microsoft.com%2Fmail.read",
    "refresh_token": "AwABAAAAvPM1KaPlrEqdFSBzjqfTGAMxZGUTdM0t4B4...",
    "id_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJub25lIn0.eyJhdWQiOiIyZDRkMTFhMi1mODE0LTQ2YTctOD...",
}
```

| 参数            | 说明                                                         |
| :-------------- | :----------------------------------------------------------- |
| `access_token`  | 请求的访问令牌。 应用可以使用此令牌验证受保护的资源，例如 Web API。 |
| `token_type`    | 指示令牌类型值。 Azure AD 唯一支持的类型是 Bearer            |
| `expires_in`    | 访问令牌的有效期（以秒为单位）。                             |
| `scope`         | access_token 的有效范围。                                    |
| `refresh_token` | 新的 OAuth 2.0 刷新令牌。 应该将旧刷新令牌替换为新获取的这个刷新令牌，以确保刷新令牌的有效期尽可能地长。 **注意：** 仅当已请求 `offline_access` 作用域时提供。 |
| `id_token`      | 无符号 JSON Web 令牌 (JWT)。 应用可以解码此令牌的段，以请求已登录用户的相关信息。 应用可以缓存并显示值，但不应依赖于这些值获取任何授权或安全边界。 **注意：** 仅当已请求 `openid` 作用域时提供。 |