---
layout: post
title: Scatter-Gather in NServiceBus, part 1
---

This article series is about showing how to implement the Scatter-Gather pattern in NServiceBus. There are some common mistakes and hidden traps users can easily fall into, leading to bad performance and other issues.

Scatter-Gather is one of the patterns mentioned in the [Enterprise Integration Patterns book](https://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683). [It is described as](https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html):

> The Scatter-Gather routes a request message to the a number of recipients. It then uses an Aggregator to collect the responses and distill them into a single response message.

The pattern consists of two parts: Broadcasting a message to multiple recipients and then using the [Aggregator pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Aggregator.html) to gather responses.


### Broadcast

There are multiple ways to "broadcast" a message to multiple recipients. Typically, broadcasting is associated with [Publish-Subscribe](https://www.enterpriseintegrationpatterns.com/patterns/messaging/PublishSubscribeChannel.html) which is implemented using the `Publish` API in NServiceBus. However, there are many different patterns to send messages to multiple recipients and it depends on the type of the message, the recipients and the form of coupling between sender and receivers to find the best way to "broadcast".
In this blog post, I'm going to use the [Splitter pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Sequencer.html): An incoming message is split into multiple smaller messages which can then be processed independently (and parallel) from each other. The result of each "sub-message" is then aggregated back to a single result again. This pattern is implemented using regular `Send` operations in NServiceBus. Strictly speaking, this might not match 100% with the definition of the Scatter-Gather pattern as we're sending multiple messages of the same type (with different values) but for the remainer of this blog post, it doesn't make a big difference whether you publish an event or send commands.

```
public async Task Handle(OrderPlaced message, IMessageHandlerContext context)
{
    foreach (var orderItem in message.OrderItems)
    {
        await context.Send(new ProcessOrderItem
        {
            OrderId = message.OrderId,
            ItemId = orderItem.Id
        });
    }
}
```

Based on an incoming message, the snippet above splits the items in the order into separate processing commands and sends them out for parallel processing. NServiceBus takes care of batching outgoing messages: If the handler fails with an exception at any point, no message is sent.

### Aggregate
Whether we use events or commands, in both cases we expect multiple responses and we want to consolidate their results. This means we need some persistent state across all responses which stores each response's result. Very often, people decide to use a [Saga](https://docs.particular.net/nservicebus/sagas) for this, as it provides a shared state across multiple message handlers out of the box. The Scatter-Gather implementation would probably look something like this:

```
class ScatterGather : Saga<ScatterGatherSagaData>,
    IAmStartedByMessages<OrderPlaced>,
    IHandleMessages<ProcessOrderItemResult>
{
    public async Task Handle(OrderPlaced message, IMessageHandlerContext context)
    {
        Data.NumberOfItems = message.OrderItems.Count();
        foreach (var orderItem in message.OrderItems)
        {
            await context.Send(new ProcessOrderItem
            {
                OrderId = message.OrderId,
                ItemId = orderItem.Id
            });
        }
    }

    public async Task Handle(ProcessOrderItemResult message, IMessageHandlerContext context)
    {
        var totalProcessedOrders = ++Data.ProcessedOrders;
        if (totalProcessedOrders == Data.NumberOfItems)
        {
            await context.Publish(new OrderProcessed { OrderId = Data.OrderId });
            MarkAsComplete();
            Console.WriteLine("Order completed");
        }
    }
    
    protected override void ConfigureHowToFindSaga(SagaPropertyMapper<ScatterGatherSagaData> mapper)
    {
        mapper.ConfigureMapping<OrderPlaced>(m => m.OrderId).ToSaga(s => s.OrderId);
        mapper.ConfigureMapping<ProcessOrderItemResult>(m => m.OrderId).ToSaga(s => s.OrderId);
    }
}
```

This looks quite nice, the Saga API provides a simple solution to implement the Scatter-Gather pattern. Unfortunately, it's a very suboptimal solution that will run into issues at runtime. 

### Sagas and concurrency

Sagas are specifically designed to implement long-running business processes, while Scatter-Gather typically is mostly just a short-lived aggregation process. There are no specific numbers to define what's long-running or short-lived in terms of sagas though. In most cases, scattering is used to parallelize work which means that we expect a lot of responses to arrive back to the saga at the same time. Parallelizing work is most likely not relevant to your business process itself, so this fact should be a better indicator than the lifetime of the saga.

When a handler of a saga is invoked, the state of the saga is loaded from the database. This leads to a situation where multiple responses want to update the same saga instance concurrently (keep in mind that NServiceBus can handle multiple messages concurrently by default). Sagas can't handle concurrency, there is only one message which can successfully complete a saga at one time because they all touch the same data.

There are a few differences between the available saga persisters which boil down to whether they use pessimistic or optimistic concurrency control to resolve concurrency issues on the same saga instance.**
* With optimistic concurrency control, concurrent access to the same saga data is only noticed once the message handler has completed and tries to update the saga's state. At this point, a concurrency exception will be raised by the database, failing the message. Recoverability mechanism will then ensure that the message is retried. You will notice some sort of concurrency related exceptions in your endpoint's log files if that occurs.
* With pessimistic concurrency control, concurrent attempts to load the same sagas are blocked (by using some form of locks) to ensure sequential access to the data. This means that there will most likely be no exceptions visible. However, the endpoint message processing throughput will slow down significantly as in the worst case only one message can be processed at the same time. This means that with pessimistic concurrency control, the endpoint can fall down to a concurrency level of one, allowing only a single message to complete after another. This can also cause database servers to assume a deadlock if the locks are held for too long, in which case you'd also see exceptions showing up.

> NServiceBus.RecoverabilityExecutor Immediate Retry is going to retry message 'e203d802-1bca-4e0a-82e9-aad300e21fa1' because of an exception:  
>    MongoDB.Driver.MongoCommandException: Command update failed: WriteConflict.

This is an exerpt from the log file when you try to run this saga using the Particular Software MongoDB persistence. The MongoDB saga persistence uses optimistic concurrency control so concurrent saga updates are only detected once the data is being updated and rejected. As pointed out by the log statement, this exception isn't a problem itself as retrying the message should usually solve this problem.

While neither of the concurrency control modes is particularly helpful in this case, optimistic concurrency can make the situation a lot worse. Assuming that many messages to the same saga are in the queue, retrying a message due to a concurrency exception won't help much, as it will just enter a new race condition every time. After failing multiple times, recoverability will fail the message and move it to the error queue. This means the user has to manually retry the message later on instead of recoverability handling the issue automatically.

### Summary
While the broadcasting step of the Scatter-Gather pattern is straight-forward to implement with NServiceBus, the aggregation phase is trickier due to the high risk of race conditions. I hope you now have a better understanding of why a NServiceBus Saga is a suboptimal choice to implement the Scatter-Gather patten in almost all cases. The only exception might be a situation where responses trickle in slowly after each other, or when you're only scattering a very low amount of messages (< 5). Sagas are made to implement long-running processes and while their API might invite for other implementations, there is a high chance that it ends up being a bad choice. If some messages handled by your saga feel more like a technical implementation than related to the actual business process, it might be a good indicator to review that saga.

We will look into a better solution for the Scatter-Gather pattern in the next blog post.


**SQL Persistence [uses pessimistic concurrency control for querying saga data](https://docs.particular.net/persistence/sql/saga-concurrency#concurrent-access-to-existing-saga-instances). NHibernate persistence [can be configured to use pessimistic concurrency control](https://docs.particular.net/persistence/nhibernate/saga-concurrency#adjusting-the-locking-strategy).
