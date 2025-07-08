---
title: 利用 Microsoft Entra ID 保护 API
comments: true
date: 2025-03-29 16:05:54
tags:
categories:
 - asp.net
description: 该文档介绍两种使用Microsoft Entra ID（原Azure AD）保护API的方法：基于JWT的身份验证和基于OpenID Connect的授权。前者通过生成令牌、配置OWIN中间件及角色授权实现API保护，后者详细说明在Azure门户注册应用、配置认证参数，并在ASP.NET中集成OpenID Connect中间件的流程，均包含C#代码示例和关键配置步骤。
---

# 利用 Microsoft Entra ID 保护 API

> **Microsoft Entra ID** 旧名字是 Azure AD，详细信息请阅读官方文档：[Microsoft Entra ID 微软官方文档]([Microsoft 标识平台文档 - Microsoft identity platform | Microsoft Learn](https://learn.microsoft.com/zh-cn/entra/identity-platform/))
>
> **JWT** (JSON Web Token) 为轻量级的认证方案，具体原理请看：[一文读懂 JWT](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)
>
> 基于ASP.NET Web API2框架架构，设计并实现了 Microsoft Entra ID 身份认证与 JWT 令牌授权的双重安全机制，形成完整的API接口防护体系，确保服务调用的合法性与数据交互的安全性。

## 基于 JWT 身份验证与授权

> 本文所有的代码为 C#，但方法是通用的，可基于不同的平台和语言进行变换。

### 1.安装必要的 Nuget 包

- System.IdentityModel.Tokens.Jwt

- Microsoft.Owin.Security.Jwt

- Microsoft.AspNet.WebApi.Owin

- Microsoft.Owin.Host.SystemWeb

### 2.创建 JWT 生成工具类

```csharp
public class JwtManager
{
  private static readonly string SecretKey = "your-secret-key-至少32位长度"; *//* *实际部署时请替换为安全密钥*

  public static string GenerateToken(string username, string role, int expireMinutes = 60)
  {
        var securityKey = new SymmetricSecurityKey(Convert.FromBase64String(SecretKey));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);
        var claims = new[]
        {
              new Claim(ClaimTypes.Name, username),
              new Claim(ClaimTypes.Role, role),
              new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };
        var token = new JwtSecurityToken(
            issuer: "your-api",
            audience: "client-app",
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(expireMinutes),
            signingCredentials: credentials
        );
        return new JwtSecurityTokenHandler().WriteToken(token);
  }

}
```

### 3.配置 OWIN 中间件

```csharp
[assembly: OwinStartup(typeof(YourNamespace.Startup))]
namespace YourNamespace
{
    public class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            // 配置 JWT 验证
            var jwtOptions = new JwtBearerAuthenticationOptions
            {
                AuthenticationMode = AuthenticationMode.Active,
                TokenValidationParameters = new TokenValidationParameters()
                {
                    ValidateIssuer = true,
                    ValidIssuer = "your-api",
                    ValidateAudience = true,
                    ValidAudience = "client-app",
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(
                        Convert.FromBase64String(JwtManager.SecretKey)),
                    ValidateLifetime = true
                }
            };
            app.UseJwtBearerAuthentication(jwtOptions);
            // 初始化 Web API
            HttpConfiguration config = new HttpConfiguration();
            WebApiConfig.Register(config);
            app.UseWebApi(config);
        }
    }
}
```

### 4.创建登录接口

```csharp
public class LoginRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
}

[RoutePrefix("api/auth")]
public class AuthController : ApiController
{
    [HttpPost]
    [Route("login")]
    [AllowAnonymous]
    [ResponseType(typeof(string))]
    public IHttpActionResult Login(LoginRequest request)
    {
        // 示例验证逻辑（实际需替换为数据库验证）
        if (request.Username == "admin" && request.Password == "admin123")
        {
            var token = JwtManager.GenerateToken(request.Username, "Admin");
            return Ok(new { Token = token });
        }

        return Unauthorized(); // 401 未授权
    }
}

```

### 5.保护 API 断点

```csharp
[RoutePrefix("api/protected")]
[Authorize] // 整个控制器需要认证
public class ProtectedController : ApiController
{
    [HttpGet]
    [Route("admin")]
    [Authorize(Roles = "Admin")] // 仅允许 Admin 角色访问
    public IHttpActionResult AdminOnly()
    {
        var username = User.Identity.Name;
        return Ok($"Hello Admin {username}!");
    }

    [HttpGet]
    [Route("user")]
    public IHttpActionResult AnyAuthenticatedUser()
    {
        return Ok($"Welcome {User.Identity.Name}!");
    }
}
```

## 基于 Open ID Connecttion 和 Azure AD 的授权

官方文档：[OAuth 2.0 和 OpenID Connect 协议 - Microsoft identity platform | Microsoft Learn](https://learn.microsoft.com/zh-cn/entra/identity-platform/v2-protocols)

### 1.在 Azure Entra Admin Center 创建租户

官方文档：[Quickstart: Call a web API that is protected by the Microsoft identity platform - Microsoft identity platform | Microsoft Learn](https://learn.microsoft.com/zh-cn/entra/identity-platform/quickstart-web-api-aspnet-sign-in?tabs=aspnet-workforce)

1. 登录到 Azure Entra Admin Center；
   {% asset_img "image1.png" "image1" %}
2. 找到左侧的 Identity > Applications > App registrations；
   {% asset_img "image2.png" "image2" %}
3. 点击 New registration；
   a) 在 Name 输入显示名称；如：Reg_01；
   b) Supported account types 选择只允许当前组织的账号；
   c) 重定向 URI 选择 Web 类型，并输入如下三个 uri (以开发环境版本为例):
   ​              i. http://localhost:8080/
   ​             ii. http://localhost:8080/signin-oidc
   ​            iii. http://localhost:8080/signout-callback-oidc
4. 创建完成后，进入 detail 页面点击 Authentication；
   a) 将 ID tokens 选中并保存
   {% asset_img "image3.png" "image3" %}
5. 进入 Certificates & Secrets > Client Secrets。创建新的客户端密钥，并复制密钥值；
   {% asset_img "image4.png" "image4" %}
6. 进入 Overview，将 Application ID 与 Tenant ID 复制下来备用;
   {% asset_img "image5.png" "image5" %}

### 2.在 ASP.NET使用OpenIdConnectAuthenticationOptions

```csharp
public class Startup
{
    private string clientId = " clientId ";
    private string tenantId = " tenantId ";
    private string clientSecret = " clientSecret ";

    public void Configuration(IAppBuilder app)
    {      	 app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);
        app.UseCookieAuthentication(new CookieAuthenticationOptions {
            AuthenticationType = CookieAuthenticationDefaults.AuthenticationType,
            CookieManager = new SystemWebCookieManager(), // 修复 Cookie 管理问题
            CookieSameSite = SameSiteMode.None,
            CookieSecure = CookieSecureOption.SameAsRequest,
            ExpireTimeSpan = TimeSpan.FromMinutes(60)
        });

            配置 Azure AD OpenID Connect 认证
        app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
        {
            AuthenticationMode = AuthenticationMode.Active,
            ClientId = clientId,
            ClientSecret = clientSecret,
            Authority = $"https://login.microsoftonline.com/{tenantId}/v2.0",
            RedirectUri = "http://localhost:60026/signin-oidc",
            PostLogoutRedirectUri = "http://localhost:60026/",
            Scope = OpenIdConnectScope.OpenIdProfile + " email",
            ResponseType = OpenIdConnectResponseType.CodeIdToken,
            TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidIssuer = $"https://sts.windows.net/{tenantId}/"
            },                
            Notifications = new OpenIdConnectAuthenticationNotifications
            {
                AuthorizationCodeReceived = async context =>
                {
                    try
                    {
                        // 1. 获取 Token
                        var authContext = new AuthenticationContext($"https://login.microsoftonline.com/{tenantId}");
                        var clientCredential = new ClientCredential(clientId, clientSecret);
                        var result = await authContext.AcquireTokenByAuthorizationCodeAsync(
                            context.Code,
                            new Uri(context.RedirectUri),
                            clientCredential);
                        System.Diagnostics.Debug.WriteLine($"Access Token: {result.AccessToken}");                         
                        context.AuthenticationTicket.Identity.AddClaim(new Claim("access_token", result.AccessToken));
                    }
                    catch (Exception ex)
                    {
                        // 记录错误日志
                        System.Diagnostics.Trace.TraceError($"Token 获取失败: {ex}");
                        context.HandleResponse();
                        context.Response.Redirect("/Error?code=token_failure");
                    }                        
                },
                AuthenticationFailed = context =>
                {
                    System.Diagnostics.Trace.TraceError($"认证失败: {context.Exception}");
                    context.HandleResponse();
                    context.Response.Redirect("/Error?message=" + context.Exception.Message);
                    return Task.CompletedTask;
                }
            }
        }); 
        // 初始化 Web API
        HttpConfiguration config = new HttpConfiguration();
        WebApiConfig.Register(config);
        app.UseWebApi(config);
    }
}
```
1. 上述代码是在启动之前配置并启用 **Azure AD OpenID Connect** 认证服务。

2. 创建登录接口

```csharp
public class AccountController : Controller
{
    public void Login()
    {
        if (!Request.IsAuthenticated)
        {
            // 触发 Azure AD 登录
            HttpContext.GetOwinContext().Authentication.Challenge(
                new AuthenticationProperties { RedirectUri = "/" },
                OpenIdConnectAuthenticationDefaults.AuthenticationType);
        }
    }

    public void Logout()
    {
        // 注销本地 Cookie 和 Azure AD 会话
        HttpContext.GetOwinContext().Authentication.SignOut(
            CookieAuthenticationDefaults.AuthenticationType,
            OpenIdConnectAuthenticationDefaults.AuthenticationType);
    }
}
```

## 基于 JWT 与 Azure AD 设置授权

上面两章我们学习了如何使用 JWT 与 Azure AD(**也称 Azure Entra**)；当我们正在开发一个最小 API 服务，并想要使用 JWT 对 API 进行授权时，也需要调用该 API 的客户端拥有特定的权限。

**这就需要组合使用 Azure AD (Azure Entra)** **与 JWT** 。

整个授权的流程图如下：
{% asset_img "image6.png" "image6" %}

- 客户端在调用API前首先需要向授权服务器 (Azure Entra) 发出授权认证申请；

- 授权服务器得到申请并验证成功后会返回 access token；

- 客户端在调用API时会带上 access token；

- API 服务接收到请求后，会向授权服务器验证 access token，在验证成功后会响应给客户端。

### 1.启用 JWT

1. 在 startup 配置启用 **JwtBearerAuthenticationOptions**；

```csharp
apiApp.UseJwtBearerAuthentication(new JwtBearerAuthenticationOptions
{
    TokenValidationParameters = new TokenValidationParameters
    {
        ValidIssuer = $"https://sts.windows.net/{tenantId}/",
        ValidAudience = "api://" + clientId,
        IssuerSigningKeyResolver = (token, securityToken, kid, validationParameters) =>
        {
            var metadataAddress = $"https://login.microsoftonline.com/{tenantId}/.well-known/openid-configuration";
            var metadata = new ConfigurationManager<OpenIdConnectConfiguration>(
                metadataAddress,
                new OpenIdConnectConfigurationRetriever()
            ).GetConfigurationAsync().Result;
            return metadata.SigningKeys;
        }
    }
});

```

2. 使用 [Authorize] 注解标记接口，代表调用它之前需要 JWT 授权；

```csharp
public class TestController : ApiController
{
    [Authorize] // 要求 JWT 授权
    [HttpGet]
    [Route("api/test")]
    public IHttpActionResult GetData()
    {
        return Ok(new { message = "API 认证成功！" });
    }
}
```

### 2.配置 Azure AD

#### 2.1 注册客户端应用

**步骤 1**：注册客户端应用

1. 返回 **应用注册** → **新注册**。
   - **名称**：输入客户端应用名称（例如 MyClientApp）。
   - **支持的账户类型**：选择 **仅此组织目录中的账户**。
   - **重定向 URI**：选择 **Web**，输入 http://localhost（本地测试用）。
   - 点击 **注册**。

**步骤 2**：配置客户端凭据

1. 在客户端应用页面中，进入 **证书和密码** → **客户端密码** → **新建客户端密码**。
   - **说明**：输入名称（例如 MyClientSecret）。
   - **过期时间**：选择有效期（例如 6 个月）。
   - 点击 **添加**。
2. **复制客户端密码值**：保存此值（稍后无法再查看）。

#### 2.2 注册服务端应用

> **如果还没有服务端的 registration**，可以按照上述的步骤一注册新的应用。

1. 进入 ​**API** **权限** → ​**添加权限** → ​**我的 API**；
2. 选择之前注册的 API 应用（例如 MyProtectedAPI）；
3. 选择 ​委托的权限 或 ​应用程序权限；
4. 委托的权限：用户登录后访问 API（例如 access_as_api）；
5. 应用程序权限：后台服务直接访问 API（无需用户登录）；
6. 点击 ​**添加权限**。

#### 2.3 测试

利用 postman 代替客户端进行测试，需要注意以下几点：

1. Client_id 需要填客户端的 application id；
2. Client_secret 需要填客户端的 secret；
3. Scope 需要填服务端的 scope。

{% asset_img "image7.png" "image7" %}
Javascript 请求示例代码如下：

```csharp
const tokenUrl = 
`https://login.microsoftonline.com/${pm.environment.get("tenant_id")}/oauth2/v2.0/token`;
const clientId = pm.environment.get("client_id");
const clientSecret = pm.environment.get("client_secret");
const scope = pm.environment.get("api_audience") + "/.default";

pm.sendRequest({
  url: tokenUrl,
  method: 'POST',
  header: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: {
    mode: 'urlencoded',
    urlencoded: [
      { key: 'client_id', value: clientId },
      { key: 'client_secret', value: clientSecret },
      { key: 'scope', value: scope },
      { key: 'grant_type', value: 'client_credentials' }
    ]
  }
}, (err, res) => {
  if (!err) {
    pm.environment.set("access_token", res.json().access_token);
  }
});

```

> 获取 token 后，将 token 带入到请求中即可
>
> http://localhost:60026/api/test
>
> bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6IkpETmFfNGk0cjdGZ2lnTDNzS……

 