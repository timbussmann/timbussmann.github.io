---
layout: post
title: Scatter-Gather in NServiceBus, part 2
---

In the [previous blog post](/2019/09/25/scatter-gather-pt1.html) we concluded that sagas struggle to deal with scenarios where multiple messages need to update the same saga instance. Depending on the persistence option, this can manifest itself differently in practice:
* For persistence options using pessimistic locking, the described scenario will slow down the endpoint's throughput as the locking mechanism will only allow a single message to be processed at a time per saga instance.
* For persistence options using optimistic concurrency control, the endpoint can run into many concurrency violation exceptions. These exceptions will force the losing transactions to retry, potentially causing large amounts of retries. At some point, messages might even be moved to the error queue due to exceeding the configured retry limits.

If you'd have to chose between one of the options, you'd probably prefer the pessimistic locking approach. The throughput limitations are most likely less severe than the impact of multiple retries and are way better than dealing with messages in the error queue. Luckily, most of the supported Saga persistence options support pessimistic locking for sagas by now (the MongoDB and ServiceFabric persistence packages just recently switched to pessimistic locking). Particular Software also vastly improved the official [documentation on saga concurrency](https://docs.particular.net/nservicebus/sagas/concurrency), describing different concurrency problems and approaches to solve those, a highly recommended read.

In most cases, using a persistence option with pessimistic locking capabilities will prevent major problems for you, at the cost of slower throughput at some times. However, there is another approach that doesn't impact the endpoint throughput and also won't cause concurrency exceptions and retries. It's a lot more complex than just using pessimistic locking but sometimes the throughput limitations can be an important factor or you might rely on a persistence technology that doesn't support pessimistic locking. Here's a rough overview of how this approach works:

1. Instead of aggregating the results in the saga directly, a simple message handler receives the response message.
2. The handler stores the response data in an append-only storage which allows extremely fast inserts without concurrency violations.
3. A separate process repeatedly checks if all expected responses arrived and will continue the business process managed by the saga.
4. When all responses have been received, the saga aggregates the data.

Let's look at some sample code:

### Scattering

The `ScatterGatherSaga` will dispatch a large amount of messages (e.g. 500) when the saga is started:

```
class ScatterGatherSaga : Saga<ScatterGatherSagaData>, 
    IAmStartedByMessages<StartSagaCommand>
{
    public async Task Handle(StartSagaCommand message, IMessageHandlerContext context)
    {
        Data.NumberOfMessages = message.NumberOfMessages;

        for (int i = 0; i < message.NumberOfMessages; i++)
        {
            await context.Send(new RequestResponseCommand()
            {
                BatchId = message.Id,
                RequestId = i
            });
        }
    }

    // mapping omitted
}

```

Another endpoint will respond to each `RequestResponseCommand` with a `ResponseMessage`. Instead of handling those directly as part of the saga, we implement a stateless message handler.

### Gathering

The handler designed to gather responses receives `ResponseMessage`s and store the results in an append-only manner into a database (e.g. MongoDB in this case):


```
class GatherHandler : IHandleMessages<ResponseMessage>
{
    public async Task Handle(ResponseMessage message, IMessageHandlerContext context)
    {
        var mongoSession = context.SynchronizedStorageSession.GetClientSession();
        var mongoCollection = mongoSession.Client.GetDatabase(Program.DatabaseName)
            .GetCollection<ResponseResult>("results");

        await mongoCollection.InsertOneAsync(mongoSession, new ResponseResult
        {
            Id = context.MessageId,
            Result = message.RequestResult,
            BatchId = message.BatchId
        });
    }
}
```

Let's have a look at the information stored when each response is received:
* Result: This one is not surprising, we want to aggregate this information once all results arrived.
* BatchId: This allows us to find all responses which belong together (to the same saga).
* Id: We use the message ID as a unique value to ensure that retries do not cause duplicate entries (this is a rare edge-case but it can always happen using transports without distributed transactions). In case of a duplicate, the insert will fail with a unique key constraint violation. For the sake of simplicity, I did not add any error handling for this sample code, but this exception should be handled.

For the append-only storage, use whatever database you're already using but design the structure in a way that all data is only added without any chance of concurrency conflicts.

The next step is to wait for all responses to arrive.

### Counting responses

This step can be implemented in many different ways. In this case, the saga triggers a timeout to check the number of received responses. This check repeats until the expected number of responses are stored in the database, at which point we can aggregate the data and finish the gathering process:

```
class ScatterGatherSaga : Saga<ScatterGatherSagaData>, 
    IAmStartedByMessages<StartSagaCommand>,
    IHandleTimeouts<CheckStatus>
{
    public async Task Handle(StartSagaCommand message, IMessageHandlerContext context)
    {
        Data.NumberOfMessages = message.NumberOfMessages;

        for (int i = 0; i < message.NumberOfMessages; i++)
        {
            await context.SendLocal(new RequestResponseCommand()
            {
                BatchId = message.Id,
                RequestId = i
            });
        }

        // set the first timeout
        await RequestTimeout<CheckStatus>(context, TimeSpan.FromSeconds(5));
    }

    public async Task Timeout(CheckStatus state, IMessageHandlerContext context)
    {
        var mongoSession = context.SynchronizedStorageSession.GetClientSession();
        var mongoCollection = mongoSession.Client.GetDatabase(Program.DatabaseName)
            .GetCollection<ResponseResult>("results");

        var totalDocuments = await mongoCollection.CountDocumentsAsync(mongoSession,
            Builders<ResponseResult>.Filter.Eq(d => d.BatchId, Data.BatchId));
        if (totalDocuments >= Data.NumberOfMessages)
        {
            Console.WriteLine($"All responses for saga {Data.BatchId} received.");
            var responses = await mongoCollection
                .FindAsync<ResponseResult>(mongoSession, Builders<ResponseResult>.Filter.Eq(r => r.BatchId, Data.BatchId));

            int result = responses.ToEnumerable().Sum(response => response.Result);
            Console.WriteLine("Total result is: " + result);
            MarkAsComplete();
        }
        else
        {
            Console.WriteLine($"not all responses came in. Currently at {totalDocuments}/{Data.NumberOfMessages}");
            await RequestTimeout<CheckStatus>(context, TimeSpan.FromSeconds(1));
        }
    }

    // mapping omitted
}
```

## Summary

In this blog post, we've looked at a more complex approach to deal with scatter-gather scenarios in NServiceBus by extracting the gathering process from the saga into a dedicated handler. With this approach, we can eliminate most of the problems caused by pessimistic locking or optimistic concurrency control and improve the performance or even avoid expensive retries and failed messages. 

The outlined approach is supposed to give some ideas on how to avoid the existing limitations of sagas in regards to concurrency, it is not a complete example nor does it cover all of the possible scenarios.
