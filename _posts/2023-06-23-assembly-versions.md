---
layout: post
title: Fun with assembly versions
---

Assembly loading remains a gift that keeps on giving. On a previous blog post I've focused on the assembly-resolving logic that tries to load an assembly which isn't already loaded but is deployed along with our application. This post focuses specifically on the type-resolving behavior when trying to retrieve the type from an assembly that is already loaded but with a different version in the provided fully qualified assembly name. What do you expect the outcome of the following code to be:

```csharp
var demoClassType = typeof(DemoClass);
Console.WriteLine(demoClassType.AssemblyQualifiedName); 
// -> "Playground.DemoClass, Playground, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null"

var t1 = Type.GetType("Playground.DemoClass, Playground, Version=2.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 2: " + t1);

var t2 = Type.GetType("Playground.DemoClass, Playground, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 1: " + t2);

var t3 = Type.GetType("Playground.DemoClass, Playground, Version=3.0.0.0, Culture=neutral, PublicKeyToken=null");
Console.WriteLine("Type.GetType for version 3: " + t3);

```

100 Points if your answer was the obligatory "it depends" ;) Let's look at a few cases:

## .NET Framework

Brining back ye olde .NET Framework for some nostalgia first. The [official documentation](https://learn.microsoft.com/en-us/troubleshoot/developer/visualstudio/general/assembly-version-assembly-file-version) states:

>  At runtime, Common Language Runtime (CLR) looks for assembly with this version number to load. But remember this version is used along with name, public key token and culture information only if the assemblies are strong-named signed. If assemblies aren't strong-named signed, only file names are used for loading.

Since I'm not using strong-naming here, the version should not be relevant. And indeed, all calls to `Type.GetType` return a result (which is equal to the result of `t1`).

```
TODO console output
```

## .NET (Core)

It hasn't been that easy to find some documentation on the behavior on .NET. But .NET changed a lot under the hood since AppDomains aren't really a thing anymore, so while things are kinda similar, sometimes they are not. The above code returns:

```
TODO print console outut
```

So as long as we're loading the same or a lower version number of the type, it will be found. If we load a higher number however, .NET returns `null`.

But, [as I pointed out in an earlier blog post](TODO LINK), using `Assembly.LoadFrom` can change the loading behavior quite a bit. It doesn't just affect how/where it loads assembly from (e.g. if you can load assemmblies from disk that you're not referencing in your `deps.json` file) but also the version resolving behavior. So if we add a little `Assembly.LoadFrom("LoaderApp.dll");` (LoaderApp is the name of the console application that we're running), suddenly all `Type.GetType` calls return a type. Note that we have to load the assembly of the application that tries to resolve the type, not the assembly that contains the type (that wouldn't work)!

```
TODO print console outut
```

## nUnit

If you think you'll make sure your assembly/resolving code works by writing unit tests for it, let's look at the output when we run the code in an nUnit test:

```
test case code
```

results in the following test output:

```
TODO print console outut
```

With nUnit, all the `Type.GetType` calls also work, although they didn't work in a plain .NET console application (representing your production application). nUnit [adds it's own `Resolving` event handler](https://github.com/nunit/nunit/blob/master/src/NUnitFramework/framework/Internal/AssemblyHelper.cs#L106) to the default `AssemblyLoadContext`. As the [managed assembly loading algorithm](https://learn.microsoft.com/en-us/dotnet/core/dependency-loading/loading-managed) documents, that event will be raised when the assembly couldn't be found initially. nUnit hooks into this event and takes care of loading it anyway, ignoring version numbers in the process. Running the same test with xUnit shows the same behavior as the nUnit test (although I'm not quite sure how xUnit handles this, I suspect this code to be responsible for it), so changing the testing framework won't do much.


## Conclusion

Predicting the behavior of assembly and type loading operations remains tricky and behavior can easily change in different environments. Due to the behavior in unit testing frameworks, this is also really hard to test reliably. A fairly safe approach (when not using strong-naming) is to ignore the version from the fully qualified assembly name. E.g. the following call works in all of the examples mentioned above and returns the expected type:

```csharp
// leave out version information
Type.GetType("Playground.DemoClass, Playground, Culture=neutral, PublicKeyToken=null");
```