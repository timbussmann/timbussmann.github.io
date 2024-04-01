---
layout: post
title: "Endless retries with NServiceBus"
---

One of the benefits of using messaging technologies is the ability to just retry messages that failed to process (I also wrote about recoverability concepts [here](https://docs.particular.net/architecture/recoverability)). NServiceBus provides [extensive recoverability features](https://docs.particular.net/nservicebus/recoverability/), like immediate and delayed retries. However, there are scenarios where users might be surprised that these recoverability functions don't properly apply and messages seem to be processed infinitely instead of being moved to the error queue after exhausting the configured amount of retries.

## NServiceBus recoverability

NServiceBus invokes message handlers as part of its message processing pipeline. When exceptions occur at any point during this pipeline (e.g. in a message handler or any behavior), the *recoverability pipeline* will be executed to determine what to do with the exception. This is the time when NServiceBus determines whether the message should be retried immediately, retried with a delay, or be moved to the error queue. 

However, if the NServiceBus endpoint process crashes completely (e.g. due to an OutOfMemory exception), the recoverability pipeline can't be invoked anymore. Instead, the message will become available for processing again (the exact behavior depends on the specific transport, e.g. with Azure Service Bus the lease lock will expire and the message will be available for consumers again). If the endpoint process always crashes on a specific message content, the recoverability features will never have the chance to interrupt this behavior, potentially leading to infinite retries.

Note: The NServiceBus recoverability pipeline can't be invoked preventively as the pipeline requires the exception details to determine the proper recoverability action. Some transports might store the exception information in memory to deal with transaction limitations (e.g. [MSMQ transport](https://github.com/Particular/NServiceBus.Transport.Msmq/blob/master/src/NServiceBus.Transport.Msmq/SendsAtomicWithReceiveNativeTransactionStrategy.cs#L84)) but such approaches also won't work in the described scenario as a process crash also wipes any temporary storage.


## Solutions

To prevent infinite retries of messages, the message processing must be interrupted before executing the handler which crashes the endpoint process. There is no built-in solution in NServiceBus but depending on the transport used we can build this ourselves. When using Azure Service Bus transport, this is not too difficult to achieve, as Azure Service Bus already tracks the delivery count of a message internally. We can make use of this in 2 ways:

### ASB native DLQ

Azure Service Bus already comes with a built-in dead-letter mechanism which can be configured using the queue's *max delivery count* setting. When the message has been delivered (but not consumed) to a client more than the configured amount, the message is moved to the dead-letter queue.

When you let NServiceBus create your ASB queues, this value is set to `int.MaxValue` to prevent it from interfering with the NServiceBus recoverability configuration. You can take control of the queue creation yourself and set a more restrictive message processing limit on the queues. This is a simple solution but has some disadvantages:

* NServiceBus doesn't allow the configuration of this value, not via [code-first configuration](https://docs.particular.net/transports/azure-service-bus/configuration) nor the [CLI tool](https://docs.particular.net/transports/azure-service-bus/operational-scripting). Therefore you need to create the queues with all the other required settings manually which can be error-prone.
* The message is moved to the Azure Service Bus dead-letter queue instead of the NServiceBus configured error queue which makes it more difficult to integrate with ServiceControl.

### Shortcutting the pipeline

You can read the delivery count value of the incoming message within the pipeline and decide to invoke the recoverability pipeline directly, preventing the failing message handler code from being invoked at all. The following code snippets show how to do that:

```csharp
class InterceptingBehavior : Behavior<ITransportReceiveContext>
{
    public override Task Invoke(ITransportReceiveContext context, Func<Task> next)
    {
        // retrieve the ASB native message type to access additional message metadata
        var asbMessage = context.Extensions.Get<ServiceBusReceivedMessage>();
        if (asbMessage.DeliveryCount >= 10)
        {
            throw new MessageDeliveriesExceededException();
        }

        return next();
    }
}

class MessageDeliveriesExceededException : Exception {}
```

The custom behavior will inspect the delivery count tracked by ASB and throw a custom exception if it exceeds our defined threshold. The behavior and exception need to be registered during endpoint configuration:

```csharp
configuration.Pipeline.Register(new InterceptingBehavior(), "Prevents endless retries due to process crashes");
// unrecoverable exceptions will move messages directly to the error queue, bypassing regular retries
configuration.Recoverability().AddUnrecoverableException<MessageDeliveriesExceededException>();
```

## Conclusion

It can be helpful to understand how NServiceBus' recoverability feature works to understand its limitations. Error cases that cause a total process shutdown should be very rare and for most users it's probably more useful to have good monitoring of your endpoint health to detect endpoint crashes rather than applying the described solutions. However, there might be cases where the described solutions can help to mitigate the problem more reliably and it's once more a good example of how powerful custom pipeline behaviors are.