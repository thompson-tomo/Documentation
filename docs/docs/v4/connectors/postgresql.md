# PostgreSQL

This connector simplifies accessing [PostgreSQL](https://www.postgresql.org/) databases.
It supports the following .NET drivers:

- [Npgsql](https://www.nuget.org/packages/Npgsql), which provides an ADO.NET `DbConnection`
- [Npgsql.EntityFrameworkCore.PostgreSQL](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL), which provides [Entity Framework Core](https://learn.microsoft.com/ef/core) support

The remainder of this topic assumes that you are familiar with the basic concepts of Steeltoe Connectors. See [Overview](./usage.md) for more information.

## Using the PostgreSQL connector

To use this connector:

1. Create a PostgreSQL server instance or use a [docker container](https://github.com/SteeltoeOSS/Samples/blob/main/CommonTasks.md#postgresql).
1. Add NuGet references to your project.
1. Configure your connection string in `appsettings.json`.
1. Initialize the Steeltoe Connector at startup.
1. Use the driver-specific connection/client instance.

### Add NuGet References

To use this connector, add a NuGet reference to `Steeltoe.Connectors`. If you're using Entity Framework Core, add a
NuGet reference to `Steeltoe.Connectors.EntityFrameworkCore` instead.

Also add a NuGet reference to one of the .NET drivers listed above, as you would if you were not using Steeltoe.

### Configure connection string

The available connection string parameters for PostgreSQL are described in the [Npgsql documentation](https://www.npgsql.org/doc/connection-string-parameters.html).

The following example `appsettings.json` uses the docker container referred to earlier:

```json
{
  "Steeltoe": {
    "Client": {
      "PostgreSql": {
        "Default": {
          "ConnectionString": "Server=localhost;Database=steeltoe;Uid=steeltoe;Pwd=steeltoe"
        }
      }
    }
  }
}
```

### Initialize Steeltoe Connector

Update your `Program.cs` to initialize the Connector:

```csharp
using Steeltoe.Connectors.PostgreSql;

var builder = WebApplication.CreateBuilder(args);
builder.AddPostgreSql();
```

### Use NpgsqlConnection

To obtain an `NpgsqlConnection` instance in your application, inject the Steeltoe factory in a controller or view:

```csharp
using Microsoft.AspNetCore.Mvc;
using Npgsql;
using Steeltoe.Connectors;
using Steeltoe.Connectors.PostgreSql;

public class HomeController : Controller
{
    public async Task<IActionResult> Index(
        [FromServices] ConnectorFactory<PostgreSqlOptions, NpgsqlConnection> connectorFactory)
    {
        var connector = connectorFactory.Get();
        await using NpgsqlConnection connection = connector.GetConnection();
        await connection.OpenAsync();

        NpgsqlCommand command = connection.CreateCommand();
        command.CommandText = "SELECT 1";
        object? result = await command.ExecuteScalarAsync();

        ViewData["Result"] = result;
        return View();
    }
}
```

A complete sample app that uses `NpgsqlConnection` is provided at https://github.com/SteeltoeOSS/Samples/tree/main/Connectors/src/PostgreSql.

### Use Entity Framework Core

To retrieve data from PostgreSQL in your app using Entity Framework Core, use the following steps:

1. Define your `DbContext` class:

    ```csharp
    public class AppDbContext : DbContext
    {
        public DbSet<SampleEntity> SampleEntities => Set<SampleEntity>();

        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options)
        {
        }
    }

    public class SampleEntity
    {
        public long Id { get; set; }
        public string? Text { get; set; }
    }
    ```

1. Call the `UseNpgsql()` Steeltoe extension method from `Program.cs` to initialize Entity Framework Core:

    ```csharp
    using Steeltoe.Connectors.EntityFrameworkCore.PostgreSql;
    using Steeltoe.Connectors.PostgreSql;

    var builder = WebApplication.CreateBuilder(args);
    builder.AddPostgreSql();

    builder.Services.AddDbContext<AppDbContext>(
        (serviceProvider, options) => options.UseNpgsql(serviceProvider));
    ```

1. After you have configured and added your `DbContext` to the service container,
you can inject it and use it in a controller or view:

    ```csharp
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.EntityFrameworkCore;

    public class HomeController : Controller
    {
        public async Task<IActionResult> Index([FromServices] AppDbContext appDbContext)
        {
            List<SampleEntity> entities = await appDbContext.SampleEntities.ToListAsync();
            return View(entities);
        }
    }
    ```

A complete sample app that uses Entity Framework Core with PostgreSQL is provided at https://github.com/SteeltoeOSS/Samples/tree/main/Connectors/src/PostgreSqlEFCore.

## Cloud Foundry

This Connector supports the following service brokers:

- [Tanzu for Postgres on Cloud Foundry](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-for-postgres-on-cloud-foundry/10-1/postgres/index.html)
- [Tanzu Cloud Service Broker for GCP](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/tanzu-cloud-service-broker-for-gcp/1-9/csb-gcp/reference-gcp-postgresql.html)
- [Tanzu Cloud Service Broker for AWS](https://techdocs.broadcom.com/us/en/vmware-tanzu/platform-services/tanzu-cloud-service-broker-for-aws/1-14/csb-aws/reference-aws-postgres.html)

You can create and bind an instance to your application by using the Cloud Foundry CLI.

1. Create PostgreSQL service:

   ```shell
   cf create-service postgres your-plan samplePostgreSqlService
   ```

1. Bind service to your app:

   ```shell
   cf bind-service sampleApp samplePostgreSqlService
   ```

1. Restage the app to pick up change:

   ```shell
   cf restage sampleApp
   ```

## Kubernetes

This Connector supports the [Service Binding Specification for Kubernetes](https://github.com/servicebinding/spec).
It can be used through the [Services Toolkit](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-application-platform/1-12/tap/services-toolkit-install-services-toolkit.html).

For details on how to use this, see the instructions at https://github.com/SteeltoeOSS/Samples/tree/main/Connectors/src/PostgreSql#running-on-tanzu-platform-for-kubernetes.
