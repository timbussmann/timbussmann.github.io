---
layout: post
title: "The annoying migration of Annoy-o-bot"
---

A while ago, I built annoy-o-bot, a little GitHub application that lets you define reminders in code. Annoy-o-bot will create issues for your reminders, which can be convenient if you use GitHub often. Annoy-o-bot runs on a few Azure Functions, which I wanted to migrate to the newer isolated hosting model.

While the migration sounds fairly easy, it took me way longer than I expected, primarily due to several unforeseen breaking changes that I had to figure out individually. I've rolled back to the previous version several times over many weeks, as most problems never occurred when testing locally but only once deployed to production. I'll briefly summarize the major pain points I ran into, which I didn't find anything mentioned in the (otherwise valuable) upgrade guide.

## Missing package dependencies

One of my functions receives a set of CosmosDB documents as an input binding (`IEnumerable<ReminderDocument> dueReminders`). While this worked perfectly before, the new isolated runtime model required adding a package reference to the `Microsoft.Bcl.AsyncInterfaces`. The runtime would throw exceptions without this package reference when triggering functions. This was unexpected and tricky to figure out, as the code compiled perfectly fine, and only when this specific function was invoked would the error appear.

## Enabling request buffering

One of my functions is triggered by a callback from GitHub. To ensure the request comes from GitHub, the request is signed by GitHub. To verify the signature, Annoy-o-bot needs to hash the whole request body. The request body is later read again to be deserialized; therefore, the request body is accessed twice. Since the body is a stream, the signature validation logic reset the stream position to 0 after the validation. This worked well in the old in-process hosting model, but resetting the body stream position is no longer allowed by default in the isolated hosting model. Using `request.EnableBuffering()` before reading the body will allow changing the stream position again and save me from rewriting a bigger part of my function to ensure the body is only read once (yes, this would be better and more efficient, but my goal was to migrate with the lowest amount of effort).

## Async hash computation

The above-mentioned logic uses the `HMACSHA256` class to compute the hash. Specifically, it used the `ComputeHash` method, which now started to throw an exception:

> System.InvalidOperationException: Synchronous operations are disallowed.

This is again related to changes to accessing the `request.Body` as the stack trace indicates the source to be some ASP.NET Core HttpRequestStream logic. Switching to the asynchronous `ComputeHashAsync` method solved this issue

## Changing CosmosDB property mapping

This was my favorite, as I've spent way too much time on it, and it started challenging my sanity. All my local tests ran successfully, but in production, updates to the CosmosDB documents failed with weird exceptions like this one:

> Microsoft.Azure.Cosmos.CosmosException: Response status code does not indicate success: BadRequest (400); Substatus: 0; ActivityId: d9e669db-cc92-4b6c-8aca-6a21e7e62aad; Reason: ({"Errors":["One of the specified inputs is invalid"]} [...]

There were no changes on my documents other than changing to the new CosmosDB extension packages for the isolated hosting model. All my tests using the CosmosDB emulator and actual instances worked fine, except when running in Azure. I started to try to create different documents to figure out the offending document property. Interestingly, at some point (which I couldn't immediately reproduce and was too lazy to investigate further), the exception message slightly changed to indicate a missing document ID (note that the shown exception doesn't mention this anywhere). Now my document looks something like this:

```csharp
public class ReminderDocument
{
    [JsonProperty("id")]
    public string Id { get; set; }
    public long InstallationId { get; set; }
    public long RepositoryId { get; set; }
    ...
```

CosmosDB needs the ID property to be lowercase, so I used the `Newtonsoft.Json` attribute to configure the mapping. I looked at the `Microsoft.Azure.Functions.Worker.Extensions.CosmosDB` package and confirmed that it depends on `Newtonsoft.Json`.

Once I figured out it was something with the id property, I noticed that the isolated-hosting process model changes serialization to the `System.Text.Json` serializer (without saying a word!). The CosmosDB extension happily starts using that serializer, ignoring the Newtonsoft attribute mapping. After many hours of figuring this out, the fix was to change `[JsonProperty("id")]` to `[JsonPropertyName("id")]`.

Now, a little problem remained. As I mentioned, the tests used the Newtonsoft serializer as a default by the CosmosDB SDK. With that change, the tests started to fail because they didn't have the Azure Functions configuration that switches to the built-in JSON serializer. I updated my tests to use the same serializer config as the Azure Function using these lines:

```csharp
var options = new CosmosClientOptions()
{
    Serializer = new WorkerCosmosSerializer()
};
var cosmosClient = new CosmosClient(cosmosConnectionString, options);   
```

where `WorkerCosmosSerializer` is doing the same as the type with the same name [in the Cosmos extensions package](https://github.com/Azure/azure-functions-dotnet-worker/blob/main/extensions/Worker.Extensions.CosmosDB/src/WorkerCosmosSerializer.cs) which is private and therefore not accessible.

```
class WorkerCosmosSerializer : CosmosSerializer
{
    readonly JsonObjectSerializer jsonSerializer = JsonObjectSerializer.Default;

    public override T FromStream<T>(Stream stream)
    {
        using (stream)
        {
            return (T)jsonSerializer.Deserialize(stream, typeof(T), default)!;
        }
    }

    public override Stream ToStream<T>(T input)
    {
        var memoryStream = new MemoryStream();
        jsonSerializer.Serialize(memoryStream, input, typeof(T), default);
        memoryStream.Position = 0;
        return memoryStream;
    }
}
```

Now, I hoped there was a better way to do this, but honestly, I was just tired of dealing with all these migration problems, so this is what I have now, and I'm just glad it works.

## Conclusion

The migration from the Azure Functions in-process to the isolated hosting model turned out to be a major PITA due to many undocumented little tweaks under the hood. While I enjoy the extremely low operation cost for my little GitHub app thanks to Azure Functions, the hidden complexity of the framework definitely leaked through and bit me quite a bit here.

Before I started the migration, I made sure that I had solid test coverage of my app's functionality, which came in handy in verifying that the migration didn't do anything. While this made me more confident in the upgrade, unfortunately, there are too many blind spots around the Azure Functions hosting side that are very hard to include in local tests. Running integration tests on an actual Azure Function environment would be an absolute necessity when using Azure Functions professionally.

Anyway, enough ranting. I'm glad the migration is finally done, and I can start working on some code quality improvements that I had to set back during the migration.