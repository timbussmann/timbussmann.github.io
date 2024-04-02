---
layout: post
title: "Endless retries with NServiceBus"
---

One of the benefits of using messaging technologies is the ability to retry messages that failed to process (I also wrote about recoverability concepts [here](https://docs.particular.net/architecture/recoverability)). NServiceBus provides extensive recoverability features, like immediate and delayed retries. However, there are scenarios where users might be surprised that these recoverability functions don't correctly apply and messages seem to be processed infinitely instead of being moved to the error queue after exhausting the configured amount of retries.

## NServiceBus recoverability

NServiceBus invokes message handlers as part of its message processing pipeline. When exceptions occur at any point during this pipeline (e.g., in a message handler or any behavior), the *recoverability pipeline* will be executed to determine what to do with the exception. This is when NServiceBus determines whether the message should be retried immediately, retried with a delay, or moved to the error queue. 

However,  the recoverability pipeline can't be invoked if the NServiceBus endpoint process crashes completely (e.g., due to an OutOfMemory exception). Instead, the message will become available for processing again (the exact behavior depends on the specific transport, e.g., with Azure Service Bus, the lease lock will expire, and the message will be available for consumers again).

If the endpoint process constantly crashes on a specific message content, the recoverability features will never have the chance to interrupt this behavior, potentially leading to infinite retries.

Note: The NServiceBus recoverability pipeline can't be invoked preemptively, as the pipeline requires the exception details to determine the proper recoverability action. Some transports might store the exception information in memory to deal with transaction limitations (e.g. [MSMQ transport](https://github.com/Particular/NServiceBus.Transport.Msmq/blob/master/src/NServiceBus.Transport.Msmq/SendsAtomicWithReceiveNativeTransactionStrategy.cs#L84)). Still, such approaches also won't work in the described scenario as a process crash also wipes any temporary storage.

## Solutions

To prevent infinite retries of messages, the message processing must be interrupted before executing the offending handler. There is no built-in solution in NServiceBus, but we can build this ourselves depending on the transport used. With the Azure Service Bus transport this is relatively easy to achieve: Azure Service Bus already tracks the delivery count of a message internally. We can make use of it in 2 ways:

### Azure Service Bus native Dead Letter Queue (DLQ)

Azure Service Bus already comes with a built-in dead-lettering mechanism, which can be configured using the queue's *max delivery count* setting. When a message has been delivered (but not consumed) to a client more than the configured amount, it is moved to the dead-letter queue.

When you let NServiceBus create your Azure Service Bus queues, this value is set to `int.MaxValue` to prevent it from interfering with the NServiceBus recoverability configuration. You can take control of the queue creation yourself and set a more restrictive message processing limit on the queues. This simple solution comes with downsides:

* NServiceBus doesn't allow the configuration of this value, not via [code-first configuration](https://docs.particular.net/transports/azure-service-bus/configuration) or the [CLI tool](https://docs.particular.net/transports/azure-service-bus/operational-scripting). Therefore, you need to create the queues with all the other required settings manually, which can be error-prone.
* The message is moved to the Azure Service Bus dead-letter queue instead of the NServiceBus configured error queue, making it more challenging to integrate with ServiceControl.

### Shortcutting the pipeline

You can read the delivery count value of the incoming message within the pipeline and decide to invoke the recoverability pipeline directly, preventing the invocation of failing message handler code. The following code snippets show how to do that:

```csharp
class InterceptingBehavior : Behavior<ITransportReceiveContext>
{
    public override Task Invoke(ITransportReceiveContext context, Func<Task> next)
    {
        // retrieve the ASB native message type to access additional message metadata
        var asbMessage = context.Extensions.Get<ServiceBusReceivedMessage>();
        // define a value that makes sense in your context
        if (asbMessage.DeliveryCount >= 30)
        {
            throw new MessageDeliveriesExceededException();
        }

        return next();
    }
}

class MessageDeliveriesExceededException : Exception {}
```

The custom behavior will inspect the delivery count tracked by Azure Service Bus and throw a custom exception if it exceeds the defined threshold. Both the behavior and the exception need to be registered during endpoint configuration:

```csharp
configuration.Pipeline.Register(new InterceptingBehavior(), "Prevents endless retries due to process crashes");
// unrecoverable exceptions will move messages directly to the error queue, bypassing regular retries
configuration.Recoverability().AddUnrecoverableException<MessageDeliveriesExceededException>();
```

Note: Make sure to read [number of possible retries in NServiceBus](https://docs.particular.net/nservicebus/recoverability/#total-number-of-possible-retries) when defining the threshold to make sure the behavior doesn't interfere with the regular retries behavior.

## Conclusion

Understanding how NServiceBus' recoverability feature works can help understand its limitations. Error cases that cause a total process shutdown should be very rare. For most users, it's probably more helpful to monitor their endpoint health well to detect endpoint crashes rather than apply the described approaches. However, there might be cases where the described solutions can help mitigate the problem more reliably, and it's once more a good example of how powerful custom pipeline behaviors are.
