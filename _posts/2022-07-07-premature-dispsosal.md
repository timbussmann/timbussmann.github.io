---
layout: post
title: Premature disposal
---

Since the introduction of `async/await`, plenty of pitfalls could cause all sorts of errors and deadlocks. Often, these little mistakes were subtle and hard to detect for developers less experienced (less burned) with `async/await`.

For that reason, Particular Software started early shipping [a custom code analyzer](https://github.com/Particular/NServiceBus/blob/master/src/NServiceBus.Core.Analyzer/AwaitOrCaptureTasksAnalyzer.cs) with NServiceBus to help detect the very common mistake of missing `await` statements. Luckily, over the past few years, the tooling for .NET has drastically improved and can more reliably detect many of these mistakes. But still, it's relatively easy to fall into the remaining pitfalls when not paying close attention for a short moment. I'd argue that some new (fantastic) language features make this even more accessible.

## Old mistake, new packaging

Consider this standard error as old as `async/await` but looks slightly different thanks to the new syntax. In isolation, you might spot the error very quickly:

```csharp
static Task Main()
{
    using var disposableInstance = new MyDisposableClass();

    return DoSomething(disposableInstance);
}
```

This code can throw an `ObjectDisposedException`. Calling `await Main()` is roughly semantically equivalent to the following snippet, which makes the problem stand out more clearly:

```csharp
Task result;
using (var disposableInstance = new MyDisposableClass())
{
    result = DoSomething(disposableInstance);
}

await result;
```

The problem starts with not awaiting the call to `DoSomething` (one might argue that the problem begins at not naming the method `DoSomethingAsync` ;-) ) but instead returning the Task. It is a widespread and popular code optimization in intermediary code to avoid paying the price of the compiler-generated state machine for handling async code.

When a method only calls a single asynchronous method, returning the `Task` is much more efficient than marking the method with `async` and awaiting the invocation. However, in this case, it will cause the `Main` method to return immediately after the first asynchronous operation.

With the method completed, the `using` will also dispose of the `MyDisposableClass` instance. If the asynchronous code path inside `DoSomething` now continues and tries to access the reference of `MyDisposableClass`, it will already be disposed of and therefore raise an exception.

## Unpredictable behavior

Whether `DoSomething` runs into this problem depends on what it does internally. As with throwing Exceptions, the behavior between methods that return Tasks but are actually synchronous (e.g. `Task SyncMethod() => Task.CompletedTask;`) and truly asynchronous methods (e.g. `async Task AsyncMethod() => await Task.Yield();`) varies. Consider these two implementations of `DoSomething`:

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

The first implementation will run without any exception in the initially shown `Main` method because `DoSomething` will run to completion before it returns to `Main` to dispose the passed object reference.

The second version will throw an `ObjectDisposedException` because `Task.Yield()` will trigger the return to the caller and finish the remaining code of the asynchronous method later.

We need to assume that any real `Task` returning method will have an asynchronous code exception (like the 2nd version). In reality, a single method can even show both behaviors, entirely depending on runtime conditions:

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

An implementation like the presented one can avoid the cost of asynchronous methods when it is expected that the async path is only rarely used. However, it makes it even harder to notice the initial mistake of not `await`ing the method call because the code might work just fine most of times. Until it doesn't.

## Conclusion

The `using var` language feature is fantastic and much more convenient than having to define `using(...){ ... }` blocks all over the place. But with asynchronous code, it can be easy to miss the additional `using` keyword (imagine a scenario where the disposable instance is created at the very beginning).

No compiler or code analyzer detects such cases of "premature disposals" (or missing `await`s). It's crucial to keep an eye out for this and properly `await` any method that receives a reference to disposable objects.
