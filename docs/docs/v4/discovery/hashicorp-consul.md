# HashiCorp Consul discovery client

The Steeltoe Consul discovery client allows an application to register and unregister itself with a Consul server
and provides querying for service instances registered by other applications.

After it is activated, the client registers the currently running app with Consul and starts a background thread that
periodically sends a TTL heartbeat to the Consul server (this is optional).
The client doesn't fetch service registrations until they are asked for, and it doesn't cache them.

## Using HashiCorp Consul discovery client

To use this discovery client, add a NuGet package reference to `Steeltoe.Discovery.Consul` and initialize it from `Program.cs`.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Steeltoe: Add service discovery client for Consul.
builder.Services.AddConsulDiscoveryClient();

var app = builder.Build();
```

## Configuration settings

To get the Steeltoe discovery client to properly communicate with the Consul server, there are various settings and sections to configure.
What you provide depends on whether you want your application to register the running app and/or
whether it also needs to query for other apps.

The first section is for configuring the information needed to connect to the Consul server.
All of these settings must start with `Consul:`.

| Key | Description | Default |
| --- | --- | --- |
| `Host` | Hostname or IP address of the Consul server | `localhost` |
| `Port` | Port number the Consul server is listening on | `8500` |
| `Scheme` | Scheme to connect with the Consul server (`http` or `https`) | `http` |
| `Datacenter` | The datacenter name passed in each request to the server | |
| `Token` | Authentication token passed in each request to the server | |
| `WaitTime` | The maximum duration for a blocking request | |
| `Username` | Username for HTTP authentication | |
| `Password` | Password for HTTP authentication | |
| `Discovery:Enabled` | Whether to enable the Consul client | `true` |

The second section (optional) pertains to registering the running app.
All of these settings must start with `Consul:Discovery:`.

| Key | Description | Default |
| --- | --- | --- |
| `Register` | Whether to register the running app as a service instance | `true` |
| `FailFast` | Throw an exception (instead of logging an error) if registration fails | `true` |
| `Retry:Enabled` | Whether to try again when registering the running app fails | `false` |
| `Retry:MaxAttempts` | Maximum number of registration attempts (if retries are enabled) | `6` |
| `Retry:InitialInterval` | The time to wait (in milliseconds) after the first registration failure | `1000` |
| `Retry:Multiplier` | The factor to use when incrementing the next interval | `1.1` |
| `Retry:MaxInterval` | Upper bound (in milliseconds) for intervals | `2000` |
| `Deregister` | Whether to de-register the running app on shutdown | `true` |
| `ServiceName` | Friendly name with which to register the running app | computed |
| `Scheme` | Scheme with which to register the running app (`http` or `https`) | `http` |
| `HostName` | Hostname with which to register the running app (if `PreferIPAddress` is `false`) | computed |
| `IPAddress` | IP address with which to register the running app (if `PreferIPAddress` is `true`) | computed |
| `UseNetworkInterfaces` | Query the operating system for network interfaces to determine `HostName` and `IPAddress` | `false` |
| `PreferIPAddress` | Register the running app with IP address instead of hostname | `false` |
| `Port` | Port number with which to register the running app | |
| `UseAspNetCoreUrls` | Register with the port number ASP.NET Core is listening on | `true` |
| `InstanceId` | The unique ID under which to register the running app | computed |
| `Tags` | Array of tags used when registering the running app | |
| `Meta` | Metadata key/value pairs used when registering the running app | see [Configuring metadata](#configuring-metadata) |
| `InstanceGroup` | Metadata `group` value to use when registering the running app | |
| `InstanceZone` | Metadata zone value to use when registering the running app | |
| `DefaultZoneMetadataName` | Metadata key name for `InstanceZone` | `zone` |
| `RegisterHealthCheck` | Whether to enable periodic health checking for the running app | `true` |
| `HealthCheckCriticalTimeout` | Duration after which Consul deregisters the running app when in state `critical` [^1] | `30m` |
| `Heartbeat:Enabled` | Whether the running app periodically sends TTL (time-to-live) heartbeats [^1] | `true` |
| `Heartbeat:TtlValue` | How often a TTL heartbeat must be sent to be considered healthy | `30` |
| `Heartbeat:TtlUnit` | Unit for `TtlValue` (`ms`, `s`, `m` or `h`) | `s` |
| `Heartbeat:IntervalRatio` | Rate at which the running app sends TTL heartbeats, relative to `TtlValue` with `TtlUnit` | `0.66` |
| `HealthCheckPath` | Relative URL to the health endpoint of the running app [^2] | `/actuator/health` |
| `HealthCheckUrl` | Absolute URL to the health endpoint of the running app (overrides `HealthCheckPath`) [^2] | |
| `HealthCheckTlsSkipVerify` | Whether Consul should skip TLS verification for HTTP health checks [^2] | `false` |
| `HealthCheckInterval` | How often Consul should perform HTTP health checks [^2] | `10s` |
| `HealthCheckTimeout` | The timeout Consul should use for HTTP health checks [^2] | `10s` |

[^1]: This setting only affects operation when `RegisterHealthCheck` is `true`

[^2]: This setting only affects operation when `RegisterHealthCheck` is `true` and `Heartbeat:Enabled` is `false`

This section pertains to querying for app instances.
All of these settings must start with `Consul:Discovery:`.

| Key | Description | Default |
| --- | --- | --- |
| `DefaultQueryTag` | The tag to filter on when querying for service instances | |
| `QueryPassing` | Filter on health status passing when querying for service instances | `true` |

For more information about these settings, see the Consul documentation:

* [Registering services](https://developer.hashicorp.com/consul/api-docs/agent/service#register-service),
* [Health setup during registration](https://developer.hashicorp.com/consul/api-docs/agent/check#register-check) and
* [Filtering service instances](https://developer.hashicorp.com/consul/api-docs/health).

### Configuring metadata

Steeltoe sends metadata (string-based key/value pairs) when registering the currently running app.
Additional metadata can be added in configuration.

For example:

```json
{
  "Consul": {
    "Discovery": {
      "Meta": {
        "exampleKey1": "exampleValue1",
        "exampleKey2": "exampleValue2"
      }
    }
  }
}
```

By default, the following metadata is added:

| Key | Value |
| --- | --- |
| `group` | Value from `Consul:Discovery:InstanceGroup` |
| Value from `Consul:Discovery:DefaultZoneMetadataName` | Value from `Consul:Discovery:InstanceZone` |
| `secure` | Value at `Consul:Discovery:Scheme` equals `https` |

## Health Contributor

The `Steeltoe.Discovery.Consul` package includes an `IHealthContributor` that verifies that the Consul server is reachable.
This health contributor is automatically added to the service container.
