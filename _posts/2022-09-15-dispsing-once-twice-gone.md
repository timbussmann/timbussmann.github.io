---
layout: post
title: Dispsing once, disposing twice, gone!
---

Chances that you've been using a dependency injection (DI) container in a C# application are incredibly high. All DI containers I know come with a _child container_ or _scope_ capability, as the Microsoft DI abstraction calls it. Scopes provide a dedicated instance lifetime controlled by the user. Consider the following code:

```csharp
interface I1
{
}

interface I2 : I1
{
}

class Impl : I2, IDisposable
{
    public static int DisposeCounter { get; set; }

    public void Dispose()
    {
        DisposeCounter++;
    }
}
```

```csharp
serviceCollection.AddScoped<Impl>();

// build IServiceProvider

using (var scope = serviceProvider.CreateScope())
{
    var service = scope.ServiceProvider.GetService<Impl>();
}
```

Disposing the scope will also call dispose on the `MyService` instance since it implements `IDisposable`. That's not too surprising, but let's look at some more nuanced scenarios:

What will the `DisposeCounter` value be if the resolved type does not implement `IDisposable`, but the implementation does?

```csharp
serviceCollection.AddScoped<I1>(_ => new Impl());

// build IServiceProvider

using (var scope = serviceProvider.CreateScope())
{
    var service = scope.ServiceProvider.GetService<I1>();
}
```

And does the value change if we resolve the instance from the container too?

```csharp
serviceCollection.AddScoped<I1>(sp => sp.GetRequiredService<Impl>());
serviceCollection.AddScoped<Impl>();

// build IServiceProvider

using (var scope = serviceProvider.CreateScope())
{
    var service = scope.ServiceProvider.GetService<I1>();
}
```

And what does happen if we resolve another interface which provides no compile-time information about the resolved implementation?

```csharp
serviceCollection.AddScoped<I1>(sp => sp.GetRequiredService<I2>());
serviceCollection.AddScoped<I2>(sp => sp.GetRequiredService<Impl>());
serviceCollection.AddScoped<Impl>();

// build IServiceProvider

using (var scope = serviceProvider.CreateScope())
{
    var service = scope.ServiceProvider.GetService<I1>();
}
```

As always, the answer is "it depends."

In our case, it depends on the dependency injection container used.

## Microsoft ServiceProvider

Let's start with the Microsoft ServiceProvider. By now, it's not just one of the most used DI containers but also acts as a reference implementation for many other containers.

If we build the `IServiceProvider` using `var serviceProvider = serviceCollection.BuildServiceProvider()` we get the following `DisposeCounter` values:

1. When resolving `I1` by building an `Impl` instance directly, `DisposeCounter` will be 1.
1. When resolving `I1` via resolving `Impl`, `DisposeCounter` will be 2.
1. When resolving `I1` via resolving `I2`, resolving `Impl`, `DisposeCounter` will be 3.

From that, we can conclude the implementation [inspects the resolved instances at runtime](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.DependencyInjection/src/ServiceLookup/ServiceProviderEngineScope.cs#L53) to determine whether they implement `IDisposable` or not. The container registration can't determine the runtime type of the factory method (or at least not easily).

However, The container doesn't deduplicate the disposable references to the same object instance (resolving the target service multiple times won't change the results).

## LightInject

Did you assume (or guess) the correct results for the Microsoft container? If not, don't worry. There is still a chance a different dependency injection container behaves as you expected it to.

Let's look at the results when we use LightInject by creating the `IServiceProvider` using `serviceCollection.CreateLightInjectServiceProvider()`:

1. When resolving `I1` by building an `Impl` instance directly, `DisposeCounter` will be 1. This is the same behavior as with Microsoft's implementation.
1. When resolving `I1` via resolving `Impl`, `DisposeCounter` will also be 1.
1. When resolving `I1` via resolving `I2`, resolving `Impl`, `DisposeCounter` will also be 1.

Similar to the Microsoft implementation, LightInject can detect resolved instances implementing `IDisposable`. But LightInject avoids disposing the same instance multiple times.

We can confirm that (unsurprisingly) both containers follow the [dependency injection guidelines](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines) which states:

> The container is responsible for cleanup of types it creates, and calls Dispose on IDisposable instances. Services resolved from the container should never be disposed by the developer.

## A word of warning

When working with _scoped_ lifetime, we can rely on the DI container disposing the created instances. The previously mentioned guidelines also apply to services resolved in _transient_ scope. When working with disposable services in _transient_ scope, carefully consult your DI containers documentation as the behavior there is much less consistent across containers, e.g., the [Ninject documentation](https://github.com/ninject/Ninject/wiki/Object-Scopes) states that transient services won't be disposed by the container. The Microsoft guidelines also explicitly recommend not registering `IDisposable` services in the transient lifetime:

> Don't register IDisposable instances with a transient lifetime. Use the factory pattern instead.

## Handling the ambiguity

Although we've seen that different containers might behave differently when disposing services, this should remain a fairly irrelevant detail in most cases. The Microsoft [guidelines on implementing `Dispose`](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose) states:

> To help ensure that resources are always cleaned up appropriately, a Dispose method should be idempotent, such that it is callable multiple times without throwing an exception. Furthermore, subsequent invocations of Dispose should do nothing.

As long as we stick to this guideline, the different container implementations shouldn't affect the correctness of your system.


## Conclusion

Dependency injection containers do not only automatically dispose dependencies directly implementing the `IDisposable` interface, they will also take care of any disposable implementation that is not visible to the consumer of the dependency. Therefore, when implementing an interface with a class that needs to dispose its resources, there is no need to push `IDisposable` to the implemented interface.

The exact behavior of the container in regards to disposing resolved instances can vary though. For more complicated dependency chains, predicting the number of times `Dispose` will be called can be tricky. However, this shouldn't matter because if you follow the general .NET coding guidelines, `Dispose` should be idempotent. That said, rest assured that plenty of implementations out there might run into issues in the shown examples (there is a reason this blog post exists ;)).
