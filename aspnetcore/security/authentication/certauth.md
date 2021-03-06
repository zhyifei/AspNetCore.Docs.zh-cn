---
title: 在 ASP.NET Core 中配置证书身份验证
author: blowdart
description: 了解如何在 ASP.NET Core for IIS 和 http.sys 中配置证书身份验证。
monikerRange: '>= aspnetcore-3.0'
ms.author: bdorrans
ms.date: 01/02/2020
no-loc:
- Blazor
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: security/authentication/certauth
ms.openlocfilehash: 2cee719014d57fa01b5e8b14edd703c192cfbe18
ms.sourcegitcommit: 70e5f982c218db82aa54aa8b8d96b377cfc7283f
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 05/04/2020
ms.locfileid: "82776638"
---
# <a name="configure-certificate-authentication-in-aspnet-core"></a>在 ASP.NET Core 中配置证书身份验证

`Microsoft.AspNetCore.Authentication.Certificate`包含类似于 ASP.NET Core 的[证书身份验证](https://tools.ietf.org/html/rfc5246#section-7.4.4)的实现。 证书身份验证发生在 TLS 级别，在它被 ASP.NET Core 之前。 更准确地说，这是验证证书的身份验证处理程序，然后向你提供可将该证书解析到的`ClaimsPrincipal`事件。 

将[主机配置](#configure-your-host-to-require-certificates)为使用证书进行身份验证，如 IIS、Kestrel、Azure Web 应用，或者其他任何所用的。

## <a name="proxy-and-load-balancer-scenarios"></a>代理和负载均衡器方案

证书身份验证是一种有状态方案，主要用于代理或负载均衡器不处理客户端和服务器之间的流量。 如果使用代理或负载平衡器，则仅当代理或负载均衡器：

* 处理身份验证。
* 将用户身份验证信息传递给应用程序（例如，在请求标头中），该信息对身份验证信息起作用。

使用代理和负载平衡器的环境中证书身份验证的替代方法是使用 OpenID Connect （OIDC） Active Directory 联合服务（ADFS）。

## <a name="get-started"></a>入门

获取并应用 HTTPS 证书，并将[主机配置](#configure-your-host-to-require-certificates)为需要证书。

在 web 应用中，添加对`Microsoft.AspNetCore.Authentication.Certificate`包的引用。 然后在`Startup.ConfigureServices`方法中，使用`services.AddAuthentication(CertificateAuthenticationDefaults.AuthenticationScheme).AddCertificate(...);`你的选项调用，同时提供一个`OnCertificateValidated`委托，用于对随请求发送的客户端证书进行任何补充验证。 将该信息转换为`ClaimsPrincipal`并在`context.Principal`属性上设置。

如果身份验证失败，此处理程序`403 (Forbidden)`将像你`401 (Unauthorized)`所料，返回响应，而不是。 原因是，在初次 TLS 连接期间应进行身份验证。 当它到达处理程序时，它的时间太晚。 无法将连接从匿名连接升级到证书。

还会`app.UseAuthentication();`在`Startup.Configure`方法中添加。 否则， `HttpContext.User`将不会设置为`ClaimsPrincipal`从证书创建。 例如：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddAuthentication(
        CertificateAuthenticationDefaults.AuthenticationScheme)
        .AddCertificate();
    // All the other service configuration.
}

public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    app.UseAuthentication();

    // All the other app configuration.
}
```

前面的示例演示了添加证书身份验证的默认方法。 处理程序使用通用证书属性构造用户主体。

## <a name="configure-certificate-validation"></a>配置证书验证

`CertificateAuthenticationOptions`处理程序具有一些内置的验证，这些验证是你应在证书上执行的最小验证。 默认情况下，将启用这些设置中的每一个。

### <a name="allowedcertificatetypes--chained-selfsigned-or-all-chained--selfsigned"></a>AllowedCertificateTypes = 链式、SelfSigned 或 All （链式 |SelfSigned)

默认值：30`CertificateTypes.Chained`

此检查将验证是否只允许使用适当的证书类型。 如果应用使用自签名证书，则需要将此选项设置为`CertificateTypes.All`或。 `CertificateTypes.SelfSigned`

### <a name="validatecertificateuse"></a>ValidateCertificateUse

默认值：30`true`

此检查将验证客户端提供的证书是否具有客户端身份验证扩展密钥使用（EKU），或者根本没有 Eku。 如规范所示，如果未指定 EKU，则所有 Eku 均视为有效。

### <a name="validatevalidityperiod"></a>ValidateValidityPeriod

默认值：30`true`

此检查将验证证书是否在其有效期内。 对于每个请求，处理程序将确保在其在其当前会话期间提供的证书过期时，该证书是有效的。

### <a name="revocationflag"></a>RevocationFlag

默认值：30`X509RevocationFlag.ExcludeRoot`

一个标志，该标志指定将检查链中的哪些证书进行吊销。

仅当证书链接到根证书时才执行吊销检查。

### <a name="revocationmode"></a>RevocationMode

默认值：30`X509RevocationMode.Online`

指定如何执行吊销检查的标志。

指定联机检查可能会导致长时间延迟，同时与证书颁发机构联系。

仅当证书链接到根证书时才执行吊销检查。

### <a name="can-i-configure-my-app-to-require-a-certificate-only-on-certain-paths"></a>我是否可以将我的应用程序配置为只需要特定路径上的证书？

这是不可能的。 请记住，证书交换已完成 HTTPS 会话的启动，在该连接上收到第一个请求之前，服务器会完成此操作，因此无法基于任何请求字段进行作用域。

## <a name="handler-events"></a>处理程序事件

处理程序有两个事件：

* `OnAuthenticationFailed`&ndash;如果在身份验证过程中发生异常，则调用，并允许您做出反应。
* `OnCertificateValidated`&ndash;在验证证书后调用，已经创建了验证并创建了一个默认主体。 此事件允许你执行自己的验证并增加或替换主体。 例如：
  * 确定你的服务是否知道该证书。
  * 构造自己的主体。 请看下面 `Startup.ConfigureServices` 中的示例：

```csharp
services.AddAuthentication(
    CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = context =>
            {
                var claims = new[]
                {
                    new Claim(
                        ClaimTypes.NameIdentifier, 
                        context.ClientCertificate.Subject,
                        ClaimValueTypes.String, 
                        context.Options.ClaimsIssuer),
                    new Claim(ClaimTypes.Name,
                        context.ClientCertificate.Subject,
                        ClaimValueTypes.String, 
                        context.Options.ClaimsIssuer)
                };

                context.Principal = new ClaimsPrincipal(
                    new ClaimsIdentity(claims, context.Scheme.Name));
                context.Success();

                return Task.CompletedTask;
            }
        };
    });
```

如果发现入站证书不符合额外的验证，请调用`context.Fail("failure reason")`失败原因。

对于实际功能，你可能需要调用在依赖关系注入中注册的服务，该服务连接到数据库或其他类型的用户存储。 使用传递到委托中的上下文访问你的服务。 请看下面 `Startup.ConfigureServices` 中的示例：

```csharp
services.AddAuthentication(
    CertificateAuthenticationDefaults.AuthenticationScheme)
    .AddCertificate(options =>
    {
        options.Events = new CertificateAuthenticationEvents
        {
            OnCertificateValidated = context =>
            {
                var validationService =
                    context.HttpContext.RequestServices
                        .GetService<ICertificateValidationService>();
                
                if (validationService.ValidateCertificate(
                    context.ClientCertificate))
                {
                    var claims = new[]
                    {
                        new Claim(
                            ClaimTypes.NameIdentifier, 
                            context.ClientCertificate.Subject, 
                            ClaimValueTypes.String, 
                            context.Options.ClaimsIssuer),
                        new Claim(
                            ClaimTypes.Name, 
                            context.ClientCertificate.Subject, 
                            ClaimValueTypes.String, 
                            context.Options.ClaimsIssuer)
                    };

                    context.Principal = new ClaimsPrincipal(
                        new ClaimsIdentity(claims, context.Scheme.Name));
                    context.Success();
                }                     

                return Task.CompletedTask;
            }
        };
    });
```

从概念上讲，验证证书是一种授权问题。 例如，在授权策略中添加一个颁发者或指纹，而不`OnCertificateValidated`是在中，这是完全可以接受的。

## <a name="configure-your-host-to-require-certificates"></a>将主机配置为需要证书

### <a name="kestrel"></a>Kestrel

在*Program.cs*中，按如下所示配置 Kestrel：

```csharp
public static void Main(string[] args)
{
    CreateHostBuilder(args).Build().Run();
}

public static IHostBuilder CreateHostBuilder(string[] args)
{
    return Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            webBuilder.UseStartup<Startup>();
            webBuilder.ConfigureKestrel(o =>
            {
                o.ConfigureHttpsDefaults(o => 
            o.ClientCertificateMode = 
                ClientCertificateMode.RequireCertificate);
            });
        });
}
```

> [!NOTE]
> 通过在调用 <xref:Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.ConfigureHttpsDefaults*> 之前调用 <xref:Microsoft.AspNetCore.Server.Kestrel.Core.KestrelServerOptions.Listen*> 创建的终结点将不会应用默认值。 

### <a name="iis"></a>IIS

在 IIS 管理器中完成以下步骤：

1. 从 "**连接**" 选项卡中选择你的站点。
1. 双击 "**功能视图**" 窗口中的 " **SSL 设置**" 选项。
1. 选中 "**需要 SSL** " 复选框，并选择 "**客户端证书**" 部分中的 "**要求**" 单选按钮。

![IIS 中的客户端证书设置](README-IISConfig.png)

### <a name="azure-and-custom-web-proxies"></a>Azure 和自定义 web 代理

有关如何配置证书转发中间件的详细[说明，请参阅托管和部署文档](xref:host-and-deploy/proxy-load-balancer#certificate-forwarding)。

### <a name="use-certificate-authentication-in-azure-web-apps"></a>在 Azure Web 应用中使用证书身份验证

Azure 不需要转发配置。 此设置已在证书转发中间件中进行设置。

> [!NOTE]
> 这要求存在 CertificateForwardingMiddleware。

### <a name="use-certificate-authentication-in-custom-web-proxies"></a>在自定义 web 代理中使用证书身份验证

`AddCertificateForwarding`方法用于指定：

* 客户端标头名称。
* 如何加载证书（使用`HeaderConverter`属性）。

例如`X-SSL-CERT`，在自定义 web 代理中，证书作为自定义请求标头传递。 若要使用它，请在中`Startup.ConfigureServices`配置证书转发：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddCertificateForwarding(options =>
    {
        options.CertificateHeader = "X-SSL-CERT";
        options.HeaderConverter = (headerValue) =>
        {
            X509Certificate2 clientCertificate = null;
        
            if(!string.IsNullOrWhiteSpace(headerValue))
            {
                byte[] bytes = StringToByteArray(headerValue);
                clientCertificate = new X509Certificate2(bytes);
            }

            return clientCertificate;
        };
    });
}

private static byte[] StringToByteArray(string hex)
{
    int NumberChars = hex.Length;
    byte[] bytes = new byte[NumberChars / 2];

    for (int i = 0; i < NumberChars; i += 2)
    {
        bytes[i / 2] = Convert.ToByte(hex.Substring(i, 2), 16);
    }

    return bytes;
}
```

然后`Startup.Configure` ，该方法将添加中间件。 `UseCertificateForwarding`调用`UseAuthentication`和`UseAuthorization`之前调用：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    ...

    app.UseRouting();

    app.UseCertificateForwarding();
    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

单独的类可用于实现验证逻辑。 由于本示例中使用了相同的自签名证书，因此请确保只能使用证书。 验证客户端证书和服务器证书的指纹是否匹配，否则，任何证书都可以使用，并且足以进行身份验证。 这将在`AddCertificate`方法中使用。 如果使用的是中间证书或子证书，也可以在此处验证使用者或颁发者。

```csharp
using System.IO;
using System.Security.Cryptography.X509Certificates;

namespace AspNetCoreCertificateAuthApi
{
    public class MyCertificateValidationService
    {
        public bool ValidateCertificate(X509Certificate2 clientCertificate)
        {
            // Do not hardcode passwords in production code
            // Use thumbprint or key vault
            var cert = new X509Certificate2(
                Path.Combine("sts_dev_cert.pfx"), "1234");

            if (clientCertificate.Thumbprint == cert.Thumbprint)
            {
                return true;
            }

            return false;
        }
    }
}
```

#### <a name="implement-an-httpclient-using-a-certificate-and-the-httpclienthandler"></a>使用证书和 HttpClientHandler 实现 HttpClient

HttpClientHandler 可以直接添加到 HttpClient 类的构造函数中。 创建 HttpClient 的实例时应格外小心。 然后，HttpClient 将随每个请求发送证书。

```csharp
private async Task<JsonDocument> GetApiDataUsingHttpClientHandler()
{
    var cert = new X509Certificate2(Path.Combine(_environment.ContentRootPath, "sts_dev_cert.pfx"), "1234");
    var handler = new HttpClientHandler();
    handler.ClientCertificates.Add(cert);
    var client = new HttpClient(handler);
     
    var request = new HttpRequestMessage()
    {
        RequestUri = new Uri("https://localhost:44379/api/values"),
        Method = HttpMethod.Get,
    };
    var response = await client.SendAsync(request);
    if (response.IsSuccessStatusCode)
    {
        var responseContent = await response.Content.ReadAsStringAsync();
        var data = JsonDocument.Parse(responseContent);
        return data;
    }
 
    throw new ApplicationException($"Status code: {response.StatusCode}, Error: {response.ReasonPhrase}");
}
```

#### <a name="implement-an-httpclient-using-a-certificate-and-a-named-httpclient-from-ihttpclientfactory"></a>使用证书和 IHttpClientFactory 中的命名 HttpClient 实现 HttpClient 

在下面的示例中，使用处理程序中的 ClientCertificates 属性将客户端证书添加到 HttpClientHandler 中。 然后，可以使用 ConfigurePrimaryHttpMessageHandler 方法在 HttpClient 的命名实例中使用此处理程序。 这是在 ConfigureServices 方法中的 Startup 类中设置的。

```csharp
var clientCertificate = 
    new X509Certificate2(
      Path.Combine(_environment.ContentRootPath, "sts_dev_cert.pfx"), "1234");
 
var handler = new HttpClientHandler();
handler.ClientCertificates.Add(clientCertificate);
 
services.AddHttpClient("namedClient", c =>
{
}).ConfigurePrimaryHttpMessageHandler(() => handler);
```

然后，可以使用 IHttpClientFactory 通过处理程序和证书获取命名实例。 使用 Startup 类中定义的客户端名称的 CreateClient 方法来获取实例。 可根据需要使用客户端发送 HTTP 请求。

```csharp
private readonly IHttpClientFactory _clientFactory;
 
public ApiService(IHttpClientFactory clientFactory)
{
    _clientFactory = clientFactory;
}
 
private async Task<JsonDocument> GetApiDataWithNamedClient()
{
    var client = _clientFactory.CreateClient("namedClient");
 
    var request = new HttpRequestMessage()
    {
        RequestUri = new Uri("https://localhost:44379/api/values"),
        Method = HttpMethod.Get,
    };
    var response = await client.SendAsync(request);
    if (response.IsSuccessStatusCode)
    {
        var responseContent = await response.Content.ReadAsStringAsync();
        var data = JsonDocument.Parse(responseContent);
        return data;
    }
 
    throw new ApplicationException($"Status code: {response.StatusCode}, Error: {response.ReasonPhrase}");
}
```

如果将正确的证书发送到服务器，则返回数据。 如果未发送证书或证书不正确，则返回 HTTP 403 状态代码。

### <a name="create-certificates-in-powershell"></a>在 PowerShell 中创建证书

创建证书是最难设置此流的部分。 可以使用`New-SelfSignedCertificate` PowerShell cmdlet 创建根证书。 创建证书时，请使用强密码。 如图所示，添加`KeyUsageProperty`参数和`KeyUsage`参数非常重要。

#### <a name="create-root-ca"></a>创建根 CA

```powershell
New-SelfSignedCertificate -DnsName "root_ca_dev_damienbod.com", "root_ca_dev_damienbod.com" -CertStoreLocation "cert:\LocalMachine\My" -NotAfter (Get-Date).AddYears(20) -FriendlyName "root_ca_dev_damienbod.com" -KeyUsageProperty All -KeyUsage CertSign, CRLSign, DigitalSignature

$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText

Get-ChildItem -Path cert:\localMachine\my\"The thumbprint..." | Export-PfxCertificate -FilePath C:\git\root_ca_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\"The thumbprint..." -FilePath root_ca_dev_damienbod.crt
```

> [!NOTE]
> `-DnsName`参数值必须与应用的部署目标匹配。 例如，用于开发的 "localhost"。

#### <a name="install-in-the-trusted-root"></a>在受信任的根中安装

根证书需要在主机系统上受信任。 默认情况下，不信任证书颁发机构创建的根证书。 下面的链接说明了如何在 Windows 上完成此操作：

https://social.msdn.microsoft.com/Forums/SqlServer/5ed119ef-1704-4be4-8a4f-ef11de7c8f34/a-certificate-chain-processed-but-terminated-in-a-root-certificate-which-is-not-trusted-by-the

#### <a name="intermediate-certificate"></a>中间证书

现在可以从根证书创建中间证书。 这并不是所有用例所必需的，但你可能需要创建多个证书，或者需要激活或禁用证书组。 `TextExtension`参数是设置证书的基本约束中的路径长度所必需的。

然后，可以将中间证书添加到 Windows 主机系统中的受信任中间证书。

```powershell
$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText

$parentcert = ( Get-ChildItem -Path cert:\LocalMachine\My\"The thumbprint of the root..." )

New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "intermediate_dev_damienbod.com" -Signer $parentcert -NotAfter (Get-Date).AddYears(20) -FriendlyName "intermediate_dev_damienbod.com" -KeyUsageProperty All -KeyUsage CertSign, CRLSign, DigitalSignature -TextExtension @("2.5.29.19={text}CA=1&pathlength=1")

Get-ChildItem -Path cert:\localMachine\my\"The thumbprint..." | Export-PfxCertificate -FilePath C:\git\AspNetCoreCertificateAuth\Certs\intermediate_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\"The thumbprint..." -FilePath intermediate_dev_damienbod.crt
```

#### <a name="create-child-certificate-from-intermediate-certificate"></a>从中间证书创建子证书

可以从中间证书创建子证书。 这是最终实体，不需要创建更多的子证书。

```powershell
$parentcert = ( Get-ChildItem -Path cert:\LocalMachine\My\"The thumbprint from the Intermediate certificate..." )

New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "child_a_dev_damienbod.com" -Signer $parentcert -NotAfter (Get-Date).AddYears(20) -FriendlyName "child_a_dev_damienbod.com"

$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText

Get-ChildItem -Path cert:\localMachine\my\"The thumbprint..." | Export-PfxCertificate -FilePath C:\git\AspNetCoreCertificateAuth\Certs\child_a_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\"The thumbprint..." -FilePath child_a_dev_damienbod.crt
```

#### <a name="create-child-certificate-from-root-certificate"></a>从根证书创建子证书

还可以直接从根证书创建子证书。 

```powershell
$rootcert = ( Get-ChildItem -Path cert:\LocalMachine\My\"The thumbprint from the root cert..." )

New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "child_a_dev_damienbod.com" -Signer $rootcert -NotAfter (Get-Date).AddYears(20) -FriendlyName "child_a_dev_damienbod.com"

$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText

Get-ChildItem -Path cert:\localMachine\my\"The thumbprint..." | Export-PfxCertificate -FilePath C:\git\AspNetCoreCertificateAuth\Certs\child_a_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\"The thumbprint..." -FilePath child_a_dev_damienbod.crt
```

#### <a name="example-root---intermediate-certificate---certificate"></a>示例根中间证书证书

```powershell
$mypwdroot = ConvertTo-SecureString -String "1234" -Force -AsPlainText
$mypwd = ConvertTo-SecureString -String "1234" -Force -AsPlainText

New-SelfSignedCertificate -DnsName "root_ca_dev_damienbod.com", "root_ca_dev_damienbod.com" -CertStoreLocation "cert:\LocalMachine\My" -NotAfter (Get-Date).AddYears(20) -FriendlyName "root_ca_dev_damienbod.com" -KeyUsageProperty All -KeyUsage CertSign, CRLSign, DigitalSignature

Get-ChildItem -Path cert:\localMachine\my\0C89639E4E2998A93E423F919B36D4009A0F9991 | Export-PfxCertificate -FilePath C:\git\root_ca_dev_damienbod.pfx -Password $mypwdroot

Export-Certificate -Cert cert:\localMachine\my\0C89639E4E2998A93E423F919B36D4009A0F9991 -FilePath root_ca_dev_damienbod.crt

$rootcert = ( Get-ChildItem -Path cert:\LocalMachine\My\0C89639E4E2998A93E423F919B36D4009A0F9991 )

New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "child_a_dev_damienbod.com" -Signer $rootcert -NotAfter (Get-Date).AddYears(20) -FriendlyName "child_a_dev_damienbod.com" -KeyUsageProperty All -KeyUsage CertSign, CRLSign, DigitalSignature -TextExtension @("2.5.29.19={text}CA=1&pathlength=1")

Get-ChildItem -Path cert:\localMachine\my\BA9BF91ED35538A01375EFC212A2F46104B33A44 | Export-PfxCertificate -FilePath C:\git\AspNetCoreCertificateAuth\Certs\child_a_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\BA9BF91ED35538A01375EFC212A2F46104B33A44 -FilePath child_a_dev_damienbod.crt

$parentcert = ( Get-ChildItem -Path cert:\LocalMachine\My\BA9BF91ED35538A01375EFC212A2F46104B33A44 )

New-SelfSignedCertificate -certstorelocation cert:\localmachine\my -dnsname "child_b_from_a_dev_damienbod.com" -Signer $parentcert -NotAfter (Get-Date).AddYears(20) -FriendlyName "child_b_from_a_dev_damienbod.com" 

Get-ChildItem -Path cert:\localMachine\my\141594A0AE38CBBECED7AF680F7945CD51D8F28A | Export-PfxCertificate -FilePath C:\git\AspNetCoreCertificateAuth\Certs\child_b_from_a_dev_damienbod.pfx -Password $mypwd

Export-Certificate -Cert cert:\localMachine\my\141594A0AE38CBBECED7AF680F7945CD51D8F28A -FilePath child_b_from_a_dev_damienbod.crt
```

使用根证书、中间证书或子证书时，可以根据需要使用指纹或 PublicKey 来验证证书。

```csharp
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography.X509Certificates;

namespace AspNetCoreCertificateAuthApi
{
    public class MyCertificateValidationService 
    {
        public bool ValidateCertificate(X509Certificate2 clientCertificate)
        {
            return CheckIfThumbprintIsValid(clientCertificate);
        }

        private bool CheckIfThumbprintIsValid(X509Certificate2 clientCertificate)
        {
            var listOfValidThumbprints = new List<string>
            {
                "141594A0AE38CBBECED7AF680F7945CD51D8F28A",
                "0C89639E4E2998A93E423F919B36D4009A0F9991",
                "BA9BF91ED35538A01375EFC212A2F46104B33A44"
            };

            if (listOfValidThumbprints.Contains(clientCertificate.Thumbprint))
            {
                return true;
            }

            return false;
        }
    }
}
```
