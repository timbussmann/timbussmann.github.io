---
layout: post
title: Fun with assembly versions
---

Assembly loading remains a gift that keeps on giving. On a previous blog post I've focused on the assembly-resolving logic that tries to load an assembly which isn't already loaded but is deployed along with our application. This post focuses specifically on the type-resolving behavior when trying to retrieve the type from an assembly that is already loaded but with a different version in the provided fully qualified assembly name. Here's a rough diagram explaining the deployment:

![](/assets/assembly-version-overview.png)

What do you expect is the outcome of the following code?

```csharp
var demoClassType = typeof(DemoClass);
Console.WriteLine(demoClassType.AssemblyQualifiedName); 
// -> "MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null"

var t1 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 2: " + t1);

var t2 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 1: " + t2);

var t3 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=3.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 3: " + t3);
```

One hundred points if your answer was the everlasting "it depends" ;) Let's look at a few cases:

## .NET Framework

Let's bring back the old .NET Framework for some nostalgia first. 

The [official documentation](https://learn.microsoft.com/en-us/troubleshoot/developer/visualstudio/general/assembly-version-assembly-file-version) states the following:

>  At runtime, Common Language Runtime (CLR) looks for assembly with this version number to load. But remember this version is used along with name, public key token and culture information only if the assemblies are strong-named signed. If assemblies aren't strong-named signed, only file names are used for loading.

Since I'm not using strong-naming here, the version should not be relevant. And indeed, all calls to `Type.GetType` return a result (equal to the result of `t1`).

```
MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null
Type.GetType for version 2: MyLibrary.DemoClass
Type.GetType for version 1: MyLibrary.DemoClass
Type.GetType for version 3: MyLibrary.DemoClass
```

## .NET (Core)

It hasn't been easy to find documentation on the .NET behavior. In .NET, a lot changed under the hood; for example, AppDomains aren't a thing. While things are similar, sometimes they are not. The above-presented code snippet returns:

```
MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null
Type.GetType for version 2: MyLibrary.DemoClass
Type.GetType for version 1: MyLibrary.DemoClass
Type.GetType for version 3:
```

It will be found as long as we're loading the same or a lower version number of the type. However, if we try to load a higher number, .NET returns `null`.

[As I pointed out in an earlier blog post](https://timbussmann.github.io/2021/10/18/assembly-resolving.html), using `Assembly.LoadFrom` can change the loading behavior quite a bit.

It doesn't just affect how/where it loads assembly from (e.g. if you can load assemblies from disk, that you're not referencing in your `deps.json` file) but also the version resolving behavior.

If we add a little `Assembly.LoadFrom("MyConsoleApp.dll");` (note that in .NET, we need to load the DLL, not the exe file), suddenly, all `Type.GetType` calls return a type.

Note: we have to load the assembly of the application that tries to resolve the type, not the assembly that contains the type—that wouldn't work!

```
MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null
Type.GetType for version 2: MyLibrary.DemoClass
Type.GetType for version 1: MyLibrary.DemoClass
Type.GetType for version 3: MyLibrary.DemoClass
```

## NUnit

You might think you'll ensure your assembly/resolving code works by writing unit tests. Let's look at the output when we run the code in an NUnit test:

```
[Test]
public void GetTypeTest()
{
    var demoClassType = typeof(DemoClass);
    TestContext.WriteLine(demoClassType.AssemblyQualifiedName);
    // -> "MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null"

    var t1 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null");
    TestContext.WriteLine("Type.GetType for version 2: " + t1);
    Assert.NotNull(t1);

    var t2 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null");
    TestContext.WriteLine("Type.GetType for version 1: " + t2);
    Assert.NotNull(t2);

    var t3 = Type.GetType("MyLibrary.DemoClass, MyLibrary, Version=3.0.0.0, Culture=neutral, PublicKeyToken=null");
    TestContext.WriteLine("Type.GetType for version 3: " + t3);
    Assert.NotNull(t3);
}
```

The successful test outputs the following results:

```
MyLibrary.DemoClass, MyLibrary, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null
Type.GetType for version 2: MyLibrary.DemoClass
Type.GetType for version 1: MyLibrary.DemoClass
Type.GetType for version 3: MyLibrary.DemoClass
```

All the `Type.GetType` calls also work with NUnit, although they didn't work in a straightforward .NET console application (representing the production application). NUnit [adds its own `Resolving` event handler](https://github.com/nunit/nunit/blob/master/src/NUnitFramework/framework/Internal/AssemblyHelper.cs#L106) to the default `AssemblyLoadContext`. As the [managed assembly loading algorithm](https://learn.microsoft.com/en-us/dotnet/core/dependency-loading/loading-managed) documents, that event will be raised when the assembly isn't initially found. NUnit hooks into this event and loads it anyway, ignoring version numbers.

Running the same test with xUnit shows the same behavior as the NUnit test (although I'm not quite sure how xUnit handles this, I suspect [this code](https://github.com/xunit/visualstudio.xunit/blob/ff89c51f426085d115fbf20b3527182fbd56b7f1/src/xunit.runner.visualstudio/Utility/AssemblyResolution/AssemblyHelper_NetCoreApp.cs#L93C6-L93C6) to be responsible for it). It means that changing the testing framework won't do much.


## Conclusion

Predicting the behavior of assembly and type loading operations remains tricky. The behavior can easily change in different environments. Due to the behavior in unit testing frameworks, this is also really hard to test reliably. Another approach is to ignore the version from the fully qualified assembly name: 

```csharp
// leave out version information
Type.GetType("MyLibrary.DemoClass, MyLibrary, Culture=neutral, PublicKeyToken=null");
```

This call works in the abovementioned examples and returns the expected type. Leaving out the version might be the most reliable way to resolve a type, but on the other hand, you must be sure that you don't have to deal with breaking changes on the loaded types.