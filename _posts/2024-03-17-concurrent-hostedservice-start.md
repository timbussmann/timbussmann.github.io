---
layout: post
title: "TIL: Concurrent IHostedService start with .NET 8"
---

When using `IHostedService` implementations to start or run custom code as part of a [generic host](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host?tabs=appbuilder) or web application instance, there are two major behaviors one should keep in mind:
1. Hosted services are started in the order of registration with the DI container. This is a subtle detail for most use cases, but when there are dependencies (maybe not immediately apparent to the user) between services, this is a common source of errors.
2. Hosted services are started sequentially. The host waits till the `Task` returned from the service's `StartAsync` method is completed before starting the next service (by the way, shutdown happens the same way but in reverse order).

So a quick example given these two services:

```csharp
var hostBuilder = new HostBuilder();
hostBuilder.ConfigureLogging(l => l.AddConsole());
hostBuilder.ConfigureServices(s =>
{
    // register hosted services in host
    s.AddHostedService<Service1>();
    s.AddHostedService<Service2>();
});

await hostBuilder.RunConsoleAsync();

public class Service1(ILogger<Service1> logger) : BaseService(logger);
public class Service2(ILogger<Service2> logger) : BaseService(logger);

public abstract class BaseService(ILogger logger) : IHostedService
{
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation($"Starting service ({DateTime.Now:T})");
        await Task.Delay(TimeSpan.FromSeconds(5));
        logger.LogInformation($"Completed starting service ({DateTime.Now:T})");
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation($"Starting service ({DateTime.Now:T})");
        await Task.Delay(TimeSpan.FromSeconds(5));
        logger.LogInformation($"Completed stopping service ({DateTime.Now:T})");
    }
}
```

gives the following result:

```log
info: Service1[0]
      Starting service (14:31:25)
info: Service1[0]
      Completed starting service (14:31:30)
info: Service2[0]
      Starting service (14:31:30)
info: Service2[0]
      Completed starting service (14:31:35)
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
```

However, with .NET the generic host received a new setting option that enables users to change the startup behavior to start hosted services concurrently. Adding the following configuration before running the host:

```
// When using the `WebApplication` builder for ASP.NET apps, you'll find this under `builder.Host. ConfigureHostOptions`

hostBuilder.ConfigureHostOptions(options =>
{
    options.ServicesStartConcurrently = true;
    options.ServicesStopConcurrently = true;
});
```

Changes the output to this:

```log
info: Service1[0]
      Starting service (14:36:13)
info: Service2[0]
      Starting service (14:36:13)
info: Service1[0]
      Completed starting service (14:36:18)
info: Service2[0]
      Completed starting service (14:36:18)
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
```

Services are now started concurrently and therefore the host startup time of the host was reduced to the slowest service's `StartAsnyc` method instead of a sum of all services.


## The problem with concurrent startup

I haven't seen this new feature being mentioned in any of the [.NET 8 _What's new_ articles](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8/overview) and only accidentally stumbled upon this thanks to [Steve Gordon's blog post](https://www.stevejgordon.co.uk/concurrent-hosted-service-start-and-stop-in-dotnet-8). While the new setting might improve your host's startup time I have a hunch that Microsoft hasn't made this very public yet as this might lead to hard-to-understand problems.

As an example: [NServiceBus uses a hosted service](https://github.com/Particular/NServiceBus.Extensions.Hosting/blob/master/src/NServiceBus.Extensions.Hosting/NServiceBusHostedService.cs) to start an endpoint when starting your host (e.g. connecting to the message broker). Users can send messages by injecting `IMessageSession`. So when ASP.NET starts and handles requests, you'd inject a `IMessageSession` to send a message triggered by a web request. However, if NServiceBus's hosted service only starts after ASP.NET starts accepting requests, using the message session might fail because the NServiceBus endpoint isn't ready yet ([NServiceBus addressed this problem](https://github.com/Particular/NServiceBus.Extensions.Hosting/pull/468) of course, it's just an attempt of giving an example where order matters). Such problems are hard to understand especially when using 3rd party libraries. Often the right order of registration would technically make the system behave correctly but that's hard to ensure in larger code bases. Enabling parallel startup significantly increases the risks of users running into such problems.


## Conclusion

Concurrent `IHostedService` startup is a cool new feature in the generic host but it might introduce unexpected problems which is why I'd probably not blindly enable it in every project while the majority of developers (including library authors) aren't fully aware of its existence yet.