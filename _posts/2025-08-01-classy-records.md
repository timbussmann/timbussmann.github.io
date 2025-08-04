---
layout: post
title: "Classy Records"
---

Record types are certainly one of my favourite newer C# features (they were released in 2020, in case you want to feel old once more). They were adopted eagerly for a good reason. However, I've witnessed enough situations where a class would just have been a better choice due to the implementation. I just stumbled upon such a case when I found a record that implemented something like this in a codebase:

```csharp
public record Addition(int X, int Y)
{
    private long? cachedResult = null;
    public long Result => cachedResult ??= X + Y;
}
```

Let's assume that the computation of the addition would be complicated enough to justify caching the result. Do you immediately spot the issue (knowing that there must be an issue may spoil that question...)?

Let's write a test:

```csharp
[Test]
public void AdditionTests()
{
    var addition1 = new Addition(3, 4);
    Assert.That(addition1.Result, Is.EqualTo(7));
}
```

This passing test merely highlights how challenging it would be to test caching logic properly in code that isn't specifically written for the sake of a blog post. Changing the test to the following brings up the problem:

```csharp
[Test]
public void AdditionTests()
{
    var addition1 = new Addition(3, 4);
    Assert.That(addition1.Result, Is.EqualTo(7));

    var addition = addition1 with { Y = 10 };
    Assert.That(addition.Result, Is.EqualTo(13));
}
```

The issue should be fairly obvious by now. But in case you didn't see it, don't worry, multiple AI models failed too. For example, both Gemini 2.5 Pro and Claude Sonnet 3.7 predicted the outcome of the shown test incorrectly. Here's what Claude Sonnet 3.7 states:

![Claude Sonnet 3.7's Answer](/assets/claude-sonnet-3-7.png)

Claude Sonnet 4 and GPT-4.1 predicted the outcome correctly, though. GPT-4.1 explained the result in the following manner:

![GPT-4.1's Answer](/assets/gpt-4-1.png)

Ironically, the explanation of GPT-4.1 is contradictory as it claims the `cachedResult` field will not be copied over using the `with` expression and then correctly explains the actual behavior on the following sentence. 

So, in case you weren't aware that the `with` expression copies everything, including private fields, don't blame yourself; the AI is confused too. In the context of production code, I'd expect it to be even easier to miss such subtle mistakes, so keep looking out for records that really want to be classes.