---
layout: post
title: Messaging technologies showdown - scaling clients
---

The term "messaging" has become as ambiguous as "service". There are many messaging technologies available and while the all provide publish-subscribe capabilities, there are many subtle differences across these technologies. One angle that I find helpful to distinguish these technologies is by looking at the client (subscriber) side.

## Logical and physical clients

There are two ways to think about clients: Logical and physical clients. The physical ones are the effective processes being deployed and running on a machine; I'll call these "client instances." On the other hand, a logical client represents the application code that is consuming messages from the queuing technology. A (logical) application needs to be deployed, at which point there will be at least one physical instance running. However, in modern environments, client instances are often scaled so that multiple client instances belong to the same "logical" client. This article is focused on scenarios where you will have multiple instances of a logical client.

Note: Scalability isn't the only reason for having multiple client instances. When deploying new versions in an environment that supports zero-downtime deployments (e.g., Kubernetes), you'll most likely encounter situations with multiple active client instances (running on different versions!).

## Publish/Subscribe without queues

MQTT is the most prominent example of a simple, lightweight publish-subscribe protocol. MQTT clients can only subscribe to topics, and there is no concept of message queues exposed in the protocol. Every client that subscribes to a topic receives a copy of the message. This is often called fanout or broadcast. This means that every deployed client instance will receive the same message in a scaled client environment. In the scenario of discrete events, it will be the client application's responsibility to deal with the problem of "deduplicating" the message that all clients of a logical application receive concurrently.

![](/assets/mqtt-clients.jpg)
This diagram shows how each subscriber instance will receive a copy of a published message.

Typically, MQTT is used in IoT scenarios where the scaling aspect isn't that important as physical devices don't scale that often or unexpectedly. This broadcasting behavior is also helpful for continuous data streams (like telemetry data), where the fanout behavior is typically preferred. One approach for scaling subscribers is to have a single subscriber that forwards messages from MQTT to a message queue (e.g., using AMQP), where it will be consumed by the logical application as described in the next section.

Note: With MQTT protocol version 5, MQTT introduced the concept of subscriber groups. Subscriber groups are a feature not supported by every MQTT broker that addresses the limitations of generic fanout messaging on scaled subscribers.

## Publish/Subscribe with queues

Message queues are another concept many messaging technologies use. It extends the publish-subscribe model with an explicit entity on the broker that subscribers can share. This allows clients to use a "competing consumers" pattern where a message in a queue is delivered to only one client instance. Therefore a queue can be assigned to a logical client and client instances all compete for messages on the same queue to ensure that the message is processed once independently from the number of clients available. This makes client scaling simple and very flexible, as clients can be scaled up or down at any point, and messages can be distributed easily across all active client instances.


![](/assets/amqp-clients.jpg)
In this diagram, each logical application has its own message queue. Using the "competing consumers" model, exactly one client instance is able to consumer a message per logical application.


One downside of this approach is that it makes broadcasts more difficult. If all clients should receive some messages, each client requires an additional, dedicated queue. This is why NServiceBus doesn't support broadcast events but instead recommends other approaches.

## Publish/Subscribe with logs

Event Stream Processing is another messaging approach that has gained much popularity. Kafka is the best example. AWS and Azure offer similar products "as-a-service." I'm using Kafka as the technology name here; it's so commonly used.

Kafka publishers store messages in a topic partition. Every topic has one or more partitions to persist messages. Subscribers can then read messages from a partition at any point without the message being removed. Kafka also defines the concept of client groups, which ultimately is the same as our "logical client." Each client group can have multiple client instances that belong to the group.

In Kafka, a partition can only be read by one client instance within the same client group. This has several implications:

* Similar to "traditional" message queues, one event will be processed by a single client instance within the logical application.
* It's harder to mix broadcast and competing consumer models at the same time because a client instance belongs to a single client group, and broadcasts effectively only work across client groups, not across client instances.
* The maximum number of client instances is limited by the number of partitions of a topic, as only one client instance can process a partition. This limits scalability (scaling up or down is also more complex as client instances need to be rebalanced across the partitions).
* Load balancing primarily depends on the partitioning strategy and the number of partitions. Over-simplified: Scaling subscriber instances is more difficult than using traditional message queuing technologies.

![](/assets/kafka-clients.jpg)
The diagram shows how each subscriber instance of a logical instance is assigned to one more topic partitions. A publisher publishes an event to a specific partition which will be read by one subscriber instance per logical application.

An additional difference of Kafka is that messages aren't removed from the broker once a client has processed it. Therefore, clients can decide to re-process messages.

## Conclusion

The competing-consumers model is often the most convenient and easy-to-use approach for most backend systems that consume discrete business-focused messages. Thinking about how the system behaves when scaling your logical applications can help you understand how different message technology choices will impact your system.
