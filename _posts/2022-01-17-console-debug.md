---
layout: post
title: Console Debug
---

The `[CallerArgumentExpression]` attribute is a new language feature introduced with C# 10. While the [official documentation](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.callerargumentexpressionattribute) does little to nothing to explain what it does, others have already covered the feature itself to a large extent. I quite like [Andrew Lock's post](https://andrewlock.net/exploring-dotnet-6-part-11-callerargumentexpression-and-throw-helpers/) about this feature.

Andrew's blog post, as well as most other authors, focuses on the potential of this feature to write easier "throw helpers." Helper methods that verify a specific input and throw meaningful exceptions. There is another neat usage, though:

## dbg! for the win

Having played a bit with Rust recently, I often found myself doing very sophisticated debugging, also known as console output. However, Rust is not judging me for that. It supports it via its useful [`dbg!` macro](https://doc.rust-lang.org/std/macro.dbg.html). It's pretty easy to understand:

```rust
let my_value = 42;
dbg!(my_value); // => "[src/main.rs:2] my_value = 42"
```

Instead of just printing the value, it also prints the expression (and the file name + line number). This handy macro makes debug output much easier to read as you don't have to pass additional output information to figure out what kind of information is currently printed on the console. It is beneficial if you're using multiple debug outputs. So, what about C#?

## Debug to the rescue?

Reader-friendly debug output in C# is a lot noisier:

```csharp
var myValue = 42;
Console.WriteLine($"{nameof(myValue)} = {myValue}"); // => "myValue = 42"
Console.WriteLine($"{nameof(myValue)} == 42 = {myValue == 42}"); // => "myValue == 42 = True"
```

Note that C# also has a [`Debug` class](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.debug), but it doesn't provide any more helpful APIs for this, so I'll ignore it for this post.

With `[CallerArgumentExpression]` we can build a simple helper method:

```csharp
using System.Runtime.CompilerServices;

public class ConsoleHelper
{
    public static void Debug(object value, [CallerArgumentExpression("value")] string? callerArgument = null)
    {
        Console.Error.WriteLine($"{callerArgument} = {objValue}"); // dbg! writes to the error output
    }
}
```

We can use the presented helper like this:

```csharp
var myValue = 42;
ConsoleHelper.Debug(myValue); // => "myValue = 42"
ConsoleHelper.Debug(myValue == 42); // => "myValue == 42 = True"
```

Much nicer to read! We could also provide the file name and line number information that Rust's `dbg!` prints using the [`[CallerFilePath]`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.callerfilepathattribute) and [`[CallerLineNumber]`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.callerlinenumberattribute) attributes.

## Console.Debug?

Rust's `dbg!` is even more powerful. It can output complex types with no/minimal effort. The C# helper currently can't do that as it relies on the `ToString()` implementation (for now).

It would also be nicer to use this helper method directly on the `Console` type (or on the `Debug` class). For example, something like `Console.Debug(myValue == 42)`. Unfortunately, C# doesn't allow static extension methods for classes.

Disclaimer: Obviously, using proper debugging tools is a better choice in most cases. The presented approach can be a friendly little helper. Here, the main intention was to show another use case for the `[CallerArgumentExpression]` attribute.
