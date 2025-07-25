# Resource Protection using JWT in ASP.NET Core

This library is a supplement to ASP.NET Core Security, adding functionality that helps you use Cloud Foundry Security services such as [Single Sign-On for VMware Tanzu](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/single-sign-on-for-tanzu/1-16/sso-tanzu/index.html) or a [UAA Server](https://github.com/cloudfoundry/uaa) for authentication and authorization using JSON Web Tokens (JWT) in ASP.NET Core web applications.

For more information, see also the [Steeltoe Security samples](https://github.com/SteeltoeOSS/Samples/blob/main/Security/src/AuthWeb/README.md).

General guidance for use of JWT is beyond the scope of this document. For information, see:

* [JWT IO](https://jwt.io/)
* [Wikipedia](https://en.wikipedia.org/wiki/JSON_Web_Token)

For information about the underlying Microsoft JWT Bearer Authentication library, see the [Microsoft documentation](https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication).

## Using JWT in ASP.NET Core

To use this library:

1. Add NuGet references.
1. Configure settings for the security provider.
1. Add and use the security provider in the application.
1. Secure your endpoints.
1. Create an instance of a Cloud Foundry Single Sign-On service and bind it to your application.

### Add NuGet References

To use this package, add NuGet package references to:

* `Steeltoe.Security.Authentication.JwtBearer`
* `Steeltoe.Configuration.CloudFoundry` (so that Cloud Foundry service bindings can be read by Steeltoe)

### Configure Settings

The Steeltoe JWT Bearer library configures the Microsoft JWT Bearer implementation, so all supported settings are available in [`Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerOptions`](https://learn.microsoft.com/dotnet/api/microsoft.aspnetcore.authentication.jwtbearer.jwtbeareroptions).

`JwtBearerOptions` is bound to configuration values at the key `Authentication:Schemes:Bearer`. The following example `appsettings.json` shows how to declare the audience for which tokens should be considered valid (such as when a token is issued to a specific web application and then passed to backend services to perform actions on behalf of a user):

```json
{
  "Authentication": {
    "Schemes": {
      "Bearer": {
        "ValidAudience": "sampleapi"
      }
    }
  }
}
```

#### Cloud Foundry Service Bindings

The Steeltoe package `Steeltoe.Configuration.CloudFoundry` reads Single Sign-On credentials from Cloud Foundry service bindings (`VCAP_SERVICES`) and re-maps them for Microsoft's JWT Bearer library to read. Add the configuration provider to your application with this code:

```csharp
using Steeltoe.Configuration.CloudFoundry;
using Steeltoe.Configuration.CloudFoundry.ServiceBindings;

var builder = WebApplication.CreateBuilder(args);

// Steeltoe: Add Cloud Foundry application and service info to configuration.
builder.AddCloudFoundryConfiguration();
builder.Configuration.AddCloudFoundryServiceBindings();
```

#### Local UAA

A UAA server, such as [UAA Server for Steeltoe](https://github.com/SteeltoeOSS/Dockerfiles/tree/main/uaa-server), can be used for local development of applications that will be deployed to Cloud Foundry. Configuration of UAA itself is beyond the scope of this topic, but configuring your `appsettings.Development.json` to work with a local UAA server is possible with the addition of settings like these:

```json
{
  "Authentication": {
    "Schemes": {
      "Bearer": {
        "Authority": "http://localhost:8080/uaa",
        "ClientId": "steeltoesamplesserver",
        "ClientSecret": "server_secret",
        "MetadataAddress": "http://localhost:8080/.well-known/openid-configuration",
        "RequireHttpsMetadata": false
      }
    }
  }
}
```

> [!IMPORTANT]
> If you want to use the Steeltoe UAA server without modification, some application configuration options will be limited.
> The Steeltoe UAA Server configuration is available in [uaa.yml](https://github.com/SteeltoeOSS/Dockerfiles/blob/main/uaa-server/uaa.yml#L111).

### Add and use JWT Bearer Authentication

Most of the JWT Bearer functionality is provided by the Microsoft libraries, so the only difference when using Steeltoe is the addition of calling `ConfigureJwtBearerForCloudFoundry` on the `AuthenticationBuilder`, as shown in the following example:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Steeltoe.Security.Authentication.JwtBearer;

// Add Microsoft Authentication services
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer()
    // Steeltoe: configure JWT to work with UAA/Cloud Foundry
    .ConfigureJwtBearerForCloudFoundry();

// Register Microsoft Authorization services
builder.Services.AddAuthorizationBuilder()
    // Create a named authorization policy that requires the user to have a scope with the same value
    .AddPolicy("sampleapi.read", policy => policy.RequireClaim("scope", "sampleapi.read"));
```

The middleware pipeline order is important:
Activate authentication and authorization services _after_ routing services, but _before_ controller route registrations, as shown in the following example.

```csharp
var app = builder.Build();

app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();

app.MapDefaultControllerRoute();

app.Run();
```

> [!NOTE]
> This feature requires the application to be compatible with reverse-proxy scenarios, such as when running in Cloud Foundry.
> [Reverse-proxy support is automatically configured by the configuration provider for Cloud Foundry](../configuration/cloud-foundry-provider.md#reverseproxy-and-forwarded-headers-support).

### Securing Endpoints

After the services and middleware have been configured, you can secure endpoints with the standard ASP.NET Core `Authorize` attribute, as follows:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[Route("api/[controller]")]
[ApiController]
public class SampleController : ControllerBase
{

    [HttpGet]
    [Authorize(Policy = "sampleapi.read")]
    public string Get()
    {
        return "You have permission to read from the sample API.";
    }
}
```

In the preceding example, if an incoming GET request is made to the `/api/sample` endpoint and the request does not contain a valid JWT bearer token for a user with a `scope` claim equal to `sampleapi.read`, the request is rejected.

See the [Steeltoe Security samples](https://github.com/SteeltoeOSS/Samples/blob/main/Security/src/AuthWeb/README.md) for examples using a user's access token to interact with downstream APIs, focusing on these locations:

* [Configure ASP.NET Core to save the user's token](https://github.com/SteeltoeOSS/Samples/blob/4c0b222a8ea201240bb6b99aae46434864cf4dd1/Security/src/AuthWeb/appsettings.json#L15)
* [Get the user's token](https://github.com/SteeltoeOSS/Samples/blob/4c0b222a8ea201240bb6b99aae46434864cf4dd1/Security/src/AuthWeb/Controllers/HomeController.cs#L60)
* [Include the token in a downstream request](https://github.com/SteeltoeOSS/Samples/blob/4c0b222a8ea201240bb6b99aae46434864cf4dd1/Security/src/AuthWeb/ApiClients/JwtAuthorizationApiClient.cs#L24)

### Single Sign-On for VMware Tanzu

When using Single Sign-On for VMware Tanzu, you must choose a service plan before you create a service instance.
If you do not have an existing service plan, a platform operator might need to create a new plan for you.
For information about configuring service plans for use by developers, a platform operator should follow the steps in the [Single Sign-On for Tanzu operator guide](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/single-sign-on-for-tanzu/1-16/sso-tanzu/operator-index.html).

After you have identified the service plan to use, create a service instance:

```shell
cf create-service p-identity your-plan sampleService
```

#### Bind and configure with app manifest

Using a manifest file when you deploy to Cloud Foundry is recommended, and can save some work with the SSO configuration.
For information about configuring an app manifest, see the [Single Sign-On documentation](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/single-sign-on-for-tanzu/1-16/sso-tanzu/config-apps.html#configure-app-manifest).

Consider this example manifest that names the application and buildpack, and configures properties for the SSO binding:

```yaml
applications:
- name: steeltoesamplesserver
  buildpacks:
  - dotnet_core_buildpack
  env:
    GRANT_TYPE: client_credentials
    SSO_AUTHORITIES: uaa.resource, sampleapi.read
    SSO_RESOURCES: sampleapi.read
    SSO_SHOW_ON_HOME_PAGE: false
  services:
  - sampleSSOService
```

#### Bind and configure manually

Alternatively, you can bind the instance manually and restage the app with the Cloud Foundry CLI.
Then you can configure the SSO binding with the web interface.

1. Bind service to your app:

   ```shell
   cf bind-service sampleApp sampleService
   ```

1. Restage the app to pick up change:

   ```shell
   cf restage sampleApp
   ```

For more information, see:

* the [Single Sign-On for Tanzu developer guide](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/single-sign-on-for-tanzu/1-16/sso-tanzu/developer-index.html)
* the [Steeltoe Security samples](https://github.com/SteeltoeOSS/Samples/blob/main/Security/src/AuthWeb/README.md)

### UAA Server

If Single Sign-On for Tanzu is not available or desired for your application, you can use UAA.

There is no service broker available to manage service instances or bindings for UAA, so you must have a [user provided service instance](https://docs.cloudfoundry.org/devguide/services/user-provided.html) to hold the credentials.

This command is an example of how the service instance can be created:

```shell
cf cups sampleService -p '{"auth_domain": "https://uaa.login.sys.cf-app.com","grant_types": [ "authorization_code", "client_credentials" ],"client_secret": "SOME_CLIENT_SECRET","client_id": "SOME_CLIENT_ID"}'
```

And to bind the service instance to the app:

```shell
cf bind-service sampleApp sampleService
```

For additional information, see the [UAA documentation](https://docs.cloudfoundry.org/concepts/architecture/uaa.html).
