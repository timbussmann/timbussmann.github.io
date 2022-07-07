---
layout: post
title: Premature disposal
---

# Premature disposal

Since the introduction of `async/await` there have been plenty of pitfalls that could cause all sort errors and deadlocks. Often, these little mistakes were subtle and hard to detect for developers less experienced (less burned) with `async/await`. 

For that reason, we started early shipping custom code analyzers to users of NServiceBus to help detect some common mistakes like missing `await` statements. Luckily, over the past few years the tolling has drastically improved and can more reliably detect a lot of these mistakes. But still, it's fairly easy to fall into the remaining pitfalls when not paying close attention for a short moment. I'd argue that some new (fantastic) language features make this even easier. In isolation you might spot the error very quickly:

```csharp
static Task Main()
{
    using var disposableInstance = new MyDisposableClass();

    return DoSomething(disposableInstance);
}
```

This code can throw an `ObjectDisposedException`. Calling `await Main()` is roughly semantically equivalent to the following snippet which makes the problem stand out more clearly:

```csharp
Task result;
using (var disposableInstance = new MyDisposableClass())
{
    result = DoSomething(disposableInstance);
}

await result;
```

The problem starts with not awaiting the call to `DoSomething` (one might argue that the problem really starts at not calling the method `DoSomethingAsync`). This is a very common and popular code optimization in intermediary code to avoid paying the price of the compiler-generated state machine for handling async code. When a method only calls a single asynchronous method, just returning the `Task` from that method is much more efficient than marking the method with `async` and awaiting the method call. However in this case it will cause the `Main` method to return as soon as the first asynchronous operation happens. With the method completed, the `using` will of course also dispose the `MyDisposableClass` instance. If the asynchronous code path inside `DoSomething` now continues and tries to access the reference of `MyDisposableClass` it will already be disposed and therefore (most likely) raise an exception.

The `using var` language feature is fantastic and much more convenient than having to define `using(...){ ... }` blocks all over the place. But in combination with asynchronous code, it can be fairly easy to miss the additional `using` keyword (imagine a much larger method where the disposable instance is created at the very beginning). There is currently no compiler or code analyzer that will detect such cases of "premature disposals" (or rather missing `await`s), so keep an eye out for this.

Sidenote: Whether `DoSomething` runs into this problem depends on what it does internally. As with throwing Exceptions, the behavior between methods that return Tasks but are actually synchronous (e.g. `Task SyncMethod() => Task.CompletedTask;`) and truly asynchronous methods (e.g. `async Task AsyncMethod() => await Task.Yield();`) varies. Consider these two implementations of `DoSomething`:

```csharp
static Task DoSomething(MyDisposableClass myDisposableClass)
{
    myDisposableClass.DoStuff();
    return Task.CompletedTask;
}
```

and

```csharp
static async Task DoSomething(MyDisposableClass myDisposableClass)
{
    await Task.Yield();

    myDisposableClass.DoStuff();
}
```

The first implementation will run without any exception in the intially shown `Main` method because `DoSomething` will run to completion before it returns to `Main` to dispose the passed object reference. The second version will throw an `ObjectDisposedException` because `Task.Yield()` will trigger the return to the caller and finish the remaining code of the asynchronous method later. We need to assume that any real `Task` returning method will have actual asynchronous code exception (like the 2nd version) though. In reality, a single method can even show both behaviors, completely depending on runtime conditions:

```csharp
static Task DoSomething(MyDisposableClass myDisposableClass)
{
    if (myDisposableClass.Validate)
    {
        return InvokeWithExpensiveValidation();

        async Task InvokeWithExpensiveValidation()
        {
            await ExpensiveValidationAsync();
            myDisposableClass.DoStuff();
        }
    }
    else
    {
        myDisposableClass.DoStuff();
        return Task.CompletedTask;
    }
}
```

An implementation like this can avoid the cost of asynchronous methods when it is expected that the async paths is only rarely used. This makes it even harder to notice the initial mistake to not `await` the method call because the code might work just fine most of time time, until it doesn't.

