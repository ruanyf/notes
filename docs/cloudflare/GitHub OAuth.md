## 创建应用

GitHub App 和 OAuth App 都可以用于 OAuth 认证。GitHub App 的灵活性更高。

- 两者的[区别](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps)
- 如何从 OAuth App [迁移](https://docs.github.com/en/apps/creating-github-apps/about-creating-github-apps/migrating-oauth-apps-to-github-apps)到 GitHub App
- [创建 Oauth App](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)

一个用户或组织最多可以拥有 100 个 OAuth 应用。

创建应用：点击右上角头像/设置/开发者设置

## Web 应用授权流程

https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#web-application-flow

1. 用户将被重定向到 GitHub 认证页面。
2. 用户会被 GitHub 重定向回您的网站。
3. 您的应用使用用户的访问令牌访问 API。

###  [1. 请求用户的 GitHub 身份](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#1-request-a-users-github-identity)

```
GET https://github.com/login/oauth/authorize
```

此接口接受以下输入参数。

|查询参数|类型|必需的？|描述|
|---|---|---|---|
|`client_id`|`string`|必需的|您从 GitHub[注册](https://github.com/settings/applications/new)时收到的客户端 ID 。|
|`redirect_uri`|`string`|强烈推荐|用户授权后将被重定向到的应用程序中的 URL。有关[重定向 URL 的](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#redirect-urls)详细信息，请参见下文。|
|`login`|`string`|选修的|建议使用特定帐户登录并授权该应用程序。|
|`scope`|`string`|上下文相关|以空格分隔的[权限](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps)列表。如果未提供，`scope`则对于尚未为应用程序授权任何权限的用户，默认为空列表。对于已为应用程序授权权限的用户，不会向其显示包含权限列表的 OAuth 授权页面。相反，流程的此步骤将自动完成，并返回用户已为应用程序授权的权限集。例如，如果用户已执行两次 Web 流程，并分别授权了一个具有`user`特定权限的令牌和一个具有`repo`特定权限的令牌，则第三次未提供权限的 Web 流程`scope`将收到一个具有`user`特定`repo`权限的令牌。|
|`state`|`string`|强烈推荐|一个无法猜测的随机字符串。它用于防止跨站请求伪造攻击。|
|`code_challenge`|`string`|强烈推荐|用于通过 PKCE（代码交换证明密钥）保护身份验证流程。如果包含 PKCE，则此项为必需项。它必须是客户端生成的随机字符串的 43 个字符的 SHA-256 哈希值。有关此安全扩展的更多详细信息，请参阅[PKCE RFC](https://datatracker.ietf.org/doc/html/rfc7636)。`code_challenge_method`[](https://datatracker.ietf.org/doc/html/rfc7636)|
|`code_challenge_method`|`string`|强烈推荐|用于通过 PKCE（代码交换密钥验证）保护身份验证流程。如果`code_challenge`包含此功能，则为必需项。必须启用此功能`S256`——`plain`不支持代码挑战方法。|
|`allow_signup`|`string`|选修的|是否在 OAuth 流程中向未经身份验证的用户提供注册 GitHub 的选项。默认值为“否” `true`。`false`当策略禁止注册时请使用此选项。|
|`prompt`|`string`|选修的|如果设置为 true，则强制显示帐户选择器`select_account`。如果应用程序具有非 HTTP 重定向 URI 或用户登录了多个帐户，则也会显示帐户选择器。|
### [2. 用户会被 GitHub 重定向回您的网站。](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#2-users-are-redirected-back-to-your-site-by-github)

如果用户接受了您的请求，GitHub 会将用户重定向回您的网站，并`code`在重定向过程中使用一个临时代码参数以及您在上一步中提供的状态参数`state`。该临时代码将在 10 分钟后过期。如果状态不匹配，则表示该请求是由第三方发起的，您应该中止该流程。

用此凭证换取`code`访问令牌：

```
POST https://github.com/login/oauth/access_token
```

此接口接受以下输入参数。

| 参数名称            | 类型       | 必需的？ | 描述                                                                                                                                                                          |
| --------------- | -------- | ---- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client_id`     | `string` | 必需的  | 您从 GitHub 收到的 OAuth 应用客户端 ID。                                                                                                                                               |
| `client_secret` | `string` | 必需的  | 您从 GitHub 收到的 OAuth 应用客户端密钥。                                                                                                                                                |
| `code`          | `string` | 必需的  | 您在完成第一步后收到的代码。                                                                                                                                                              |
| `redirect_uri`  | `string` | 强烈推荐 | 这是用户授权后将被重定向到的应用程序中的 URL。我们可以使用此 URL 与颁发证书时最初提供的 URI 进行匹配`code`，以防止针对您服务的攻击。                                                                                                |
| `code_verifier` | `string` | 强烈推荐 | 用于通过 PKCE（代码交换证明密钥）保护身份验证流程。如果`code_challenge`在用户授权期间发送了此参数，则此参数为必填项。此参数必须是用于`code_challenge`在授权请求中生成此参数的原始值。根据您的应用程序架构，此参数可以存储在 cookie 中（`state`与参数一起），也可以存储在身份验证期间的会话变量中。 |

默认情况下，响应采用以下格式：

```shell
access_token=gho_16C7e42F292c6912E7710c838347Ae178B4a&scope=repo%2Cgist&token_type=bearer
```

如果您在请求头中指定格式，也可以接收不同格式的响应`Accept`。例如`Accept: application/json`：`Accept: application/xml`

```json
Accept: application/json
{
  "access_token":"gho_16C7e42F292c6912E7710c838347Ae178B4a",
  "scope":"repo,gist",
  "token_type":"bearer"
}
```

```xml
Accept: application/xml
<OAuth>
  <token_type>bearer</token_type>
  <scope>repo,gist</scope>
  <access_token>gho_16C7e42F292c6912E7710c838347Ae178B4a</access_token>
</OAuth>
```
### [3. 使用访问令牌访问 API](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#3-use-the-access-token-to-access-the-api)

访问令牌允许您代表用户向 API 发出请求。

```
Authorization: Bearer OAUTH-TOKEN
GET https://api.github.com/user
```

例如，在 curl 中，您可以像这样设置 Authorization 标头：

```shell
curl -H "Authorization: Bearer OAUTH-TOKEN" https://api.github.com/user
```

每次收到访问令牌时，都应该使用该令牌重新验证用户身份。用户在被要求授权应用时可能会更改其登录的帐户，如果您不在每次登录后都验证用户身份，则存在用户数据混淆的风险。