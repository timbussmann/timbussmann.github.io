---
layout: post
title: Scatter-Gather in NServiceBus, part 1
---

This is article series is about showing how to implement the Scatter-Gather pattern correctly in NServiceBus. I've created this blog post because there are some common mistakes you can easily fall for, leading to bad performance and other issues.

Scatter-Gather is one of the patterns described in the [Enterprise Integration Patterns book](https://www.amazon.com/Enterprise-Integration-Patterns-Designing-Deploying/dp/0321200683). The [description of the pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/BroadcastAggregate.html) states:

> Use a Scatter-Gather that broadcasts a message to multiple recipients and re-aggregates the responses back into a single message.

The pattern consists of two parts: Broadcasting a message to multiple recipients and then using the [Aggregator pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Aggregator.html) to gather responses. A variation is to split an incoming message into smaller sub-messages (which each produce a response) instead of broadcasting. This variation is described as the [Splitter pattern](https://www.enterpriseintegrationpatterns.com/patterns/messaging/Sequencer.html). So technically, in this blog post, I'm going to use a Splitter+Aggregator sample but it's not a big difference to using the actual Scatter-Gather pattern when using NServiceBus and everything described here holds true in both cases.

Let's just get started:

### Scatter:
Scattering (or broadcasting, or splitting) is fairly easy with NServiceBus. This sample handler shows how to split an incoming message in a bunch of subsequent messages for further processing. Alternatively, this could also be a handler publishing an event to multiple subscribers.

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

NServiceBus takes care of batching the outgoing messages already. If the handler fails with an exception at any point, no message is sent*.

### Gather:
Let's say we expect a response message from every previously sent message and we want to continue the process once all responses have been received. This means we need some persistent state to keep track of the expected responses. Very often, people decide to use a [Saga](https://docs.particular.net/nservicebus/sagas) for this, as it provides a shared state across multiple message handlers. The Scatter-Gather implementation would probably look something like this:

```
class ScatterGather : Saga<ScatterGatherSagaData>,
    IAmStartedByMessages<OrderPlaced>,
    IHandleMessages<ProcessOrderItemResult>
{
    protected override void ConfigureHowToFindSaga(SagaPropertyMapper<ScatterGatherSagaData> mapper)
    {
        mapper.ConfigureMapping<OrderPlaced>(m => m.OrderId).ToSaga(s => s.OrderId);
        mapper.ConfigureMapping<ProcessOrderItemResult>(m => m.OrderId).ToSaga(s => s.OrderId);
    }

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
}
```

This looks quite nice, the Saga API provides a simple solution to implement the Scatter-Gather pattern. Unfortunately, it's a very suboptimal solution and you will run into problems. 

### Sagas and concurrency

Sagas are specifically designed to implement long-running business processes, while Scatter-Gather typically isn't very long-running but mostly just a short-lived aggregation process. This short-lived nature is also exactly what causes the problem with the shown implementation. In most cases, scattering is used to parallelize work which means that we expect a lot of responses to arrive back to the saga at the same time. When a handler of a saga is invoked, the data of the saga is loaded from the database. This leads to a situation where multiple responses want to update the same saga instance concurrently (keep in mind that NServiceBus can handle multiple messages concurrently by default). Sagas can't handle concurrency, there is only one message which can successfully complete a saga at one time because they all touch the same data.

Now, there are a few differences between the available saga persisters which boils down to whether they use pessimistic or optimistic concurrency control to resolve concurrency issues on the same saga instance.** 
* With optimistic concurrency control, concurrent access to the same saga data is only noticed once the message handler has completed and tries to update the saga's state. At this point, a concurrency exception will be raised by the database, failing the message. Recoverability mechanism will then ensure that the message is retried. You will notice some sort of concurrency related exceptions in your endpoint's log files if that occurs.
* With pessimistic concurrency control, concurrent attempts to load the same sagas are blocked (by using some form of locks) to ensure sequential access to the data. This means that there will most likely be no exceptions visible. However, the endpoint message processing throughput will slow down significantly as in the worst case only one message can be processed at the same time. This means that with pessimistic concurrency control, the endpoint can fall down to a concurrency level of one, allowing only a single message to complete after another. This can also cause database servers to assume a deadlock if the locks are held for too long, in which case you'd also see exceptions showing up.

While neither of the solutions is particularly helpful in this case, optimistic concurrency can make the situation a lot worse. Assuming that many messages to the same saga are in the queue, retrying a message due to a concurrency exception won't help much, as it will just enter a new race condition every time. After failing multiple times, recoverability will fail the message and move it to the error queue. This means the user has to manually retry the message later on instead of recoverability handling the issue automatically.

I hope you now have a better understanding of why a Saga is a bad choice to implement the Scatter-Gather patten in almost all cases. The only exceptions might be situations where the responses trickle in slowly after each other than in parallel or when you're only scattering a very low amount of messages (< 5). Sagas are made to implement long-running processes and while their API might invite for other applications sometimes, there is a high chance that it ends up being a bad choice.

We will look into a better solution for the Scatter-Gather pattern in the next blog post.


*There is no absolute guarantee that the outgoing messages are atomic with completing the incoming message unless you're using a transaction mode of `SendsAtomicWithReceive` or higher. There is always some edge case that might interrupt the actual dispatch operation (e.g. a network failure in the middle of dispatching the batch), but NServiceBus does it's best to keep the window for such edge cases as small as possible. This is a topic for another blog post though.

**SQL Persistence [uses pessimistic concurrency control for querying saga data](https://docs.particular.net/persistence/sql/saga-concurrency#concurrent-access-to-existing-saga-instances). NHibernate persistence [can be configured to use pessimistic concurrency control](https://docs.particular.net/persistence/nhibernate/saga-concurrency#adjusting-the-locking-strategy).
