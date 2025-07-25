# Cloud Foundry Integration

Integration with [Tanzu Apps Manager](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform/tanzu-platform-for-cloud-foundry/10-0/tpcf/console-index.html) is accomplished by adding the Cloud Foundry management endpoint to your application.

When used, this endpoint enables the following additional functionality on Cloud Foundry:

* A variant of the [hypermedia](./hypermedia.md) endpoint is registered at `/cloudfoundryapplication`.
* All endpoints are also mapped under the base path `/cloudfoundryapplication`.
* [Authentication and authorization](#security) for your Cloud Foundry environment is added to the request pipeline.

> [!IMPORTANT]
> The Cloud Foundry integration works only when the [Cloud Foundry Configuration Provider](../configuration/cloud-foundry-provider.md) has also been configured.

## Configure Settings

Typically, no additional configuration is needed. However, the following table describes the configuration settings that you can apply to the Cloud Foundry endpoint.
Each key must be prefixed with `Management:Endpoints:CloudFoundry`.

| Key | Description | Default |
| --- | --- | --- |
| `Enabled` | Whether the endpoint is enabled | `true` |
| `ID` | The unique ID of the endpoint | `""` |
| `Path` | The relative path at which the endpoint is exposed | same as `ID` |
| `RequiredPermissions` | Permissions required to access the endpoint when running on Cloud Foundry | `Restricted` |
| `AllowedVerbs` | An array of HTTP verbs at which the endpoint is exposed | `GET` |
| `ValidateCertificates` | Whether to validate server certificates | `true` |
| `ApplicationId` | The ID of the application used in permission checks | |
| `CloudFoundryApi` | The URL of the Cloud Foundry API | |

> [!TIP]
> The preceding settings apply only to the `/cloudfoundryapplication` hypermedia endpoint.
> To turn off *all* management endpoints starting with this path, set `Management:CloudFoundry:Enabled` to `false`.

## Enable HTTP Access

The URL path to the endpoint is computed by combining the global `Management:Endpoints:Path` setting with the `Path` setting described in the preceding section.
The default path is `/cloudfoundryapplication`.

See the [Exposing Endpoints](./using-endpoints.md#exposing-endpoints) and [HTTP Access](./using-endpoints.md#http-access) sections for the steps required to enable HTTP access to endpoints in an ASP.NET Core application.

To add the actuator to the service container, add a [CORS](#cross-origin-resource-sharing) policy, register security middleware and map its route, use the `AddCloudFoundryActuator` extension method.

Add the following code to `Program.cs` to use the actuator endpoint:

```csharp
using Steeltoe.Configuration.CloudFoundry;
using Steeltoe.Management.Endpoint.Actuators.CloudFoundry;

var builder = WebApplication.CreateBuilder(args);
builder.Configuration.AddCloudFoundry();
builder.Services.AddCloudFoundryActuator();
```

> [!TIP]
> It is recommended that you use `AddAllActuators()` instead of adding individual actuators;
> this enables individually turning them on/off at runtime via configuration.

### Cross Origin Resource Sharing

When viewing an application in Apps Manager, HTTP requests are sent directly to application instances, with the bearer token of the logged-in user attached.
The nature of this integration requires the use of Cross Origin Resource Sharing ([CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)).
The policy defined in Steeltoe intends to limit sharing to the minimum required for the endpoints to function; however, there is no way for Steeltoe to discover or infer what the origin of an Apps Manager instance could be.
As such, the default policy allows any origin, unless the policy is customized. You should consider [customizing the CORS policy](./using-endpoints.md#customizing-the-cors-policy) to be more specific.

## Security

When adding this management endpoint to your application, the Cloud Foundry security middleware is added to the request processing pipeline of your application.
The Cloud Foundry security middleware requires a valid UAA access token to be provided as part of any request to any of the management endpoints.
Additionally, the security middleware evaluates the token to determine whether the authenticated user has permission to access the management endpoint.

> [!NOTE]
> The Cloud Foundry security middleware is active only when your application is running on Cloud Foundry.

## External access

When running in Cloud Foundry, it is still possible to access the endpoints via the [hypermedia](./hypermedia.md) URL, which defaults to `/actuator`.
In other words, you can also access all your endpoints from this URL prefix. For example, the [info](./info.md) endpoint would be accessible at `/actuator/info`.

While the endpoints provided on the `/cloudfoundryapplication` path are secured as described above, the endpoints provided on the `/actuator` path are not.
For this reason, all endpoints are exposed by default at `/cloudfoundryapplication`, but only health and info are exposed by default at `/actuator`.
In addition, the endpoints may be secured by whatever security mechanism the application itself uses. For more details, see [securing actuators](./using-endpoints.md#securing-endpoints).

> [!CAUTION]
> An authorization policy on `/actuator` also applies to `/cloudfoundryapplication`, which then breaks the integration with Apps Manager.
> To prevent public access to `/actuator` when running on Cloud Foundry, consider configuring actuators to [use an alternate port](./using-endpoints.md#configure-global-settings).

## Sample Output

This endpoint returns a list of management endpoints, including itself.

The response is always returned as JSON:

```json
{
  "type": "steeltoe",
  "_links": {
    "beans": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/beans",
      "templated": false
    },
    "dbmigrations": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/dbmigrations",
      "templated": false
    },
    "env": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/env",
      "templated": false
    },
    "health": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/health",
      "templated": true
    },
    "heapdump": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/heapdump",
      "templated": false
    },
    "httpexchanges": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/httpexchanges",
      "templated": false
    },
    "info": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/info",
      "templated": false
    },
    "loggers": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/loggers",
      "templated": true
    },
    "mappings": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/mappings",
      "templated": false
    },
    "refresh": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/refresh",
      "templated": false
    },
    "threaddump": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication/threaddump",
      "templated": false
    },
    "self": {
      "href": "https://appname.cf-app.com/cloudfoundryapplication",
      "templated": false
    }
  }
}
```
