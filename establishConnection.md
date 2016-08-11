## Establishing a connection

### The application should start whether a connection to RabbitMQ can be established or not

<b>Spring AMQP </b>
Our Spring application will start just fine even though it cannot connect to RabbitMQ. Spring will attempt to reconnect every so often.

<b>Java AMQP</b>
Our Java application should not attempt to connect to RabbitMq from the main thread otherwise our application will fail to start if we do not handle the exception properly. At least, we should catch the exception and schedule another attempt (ideally using exponential back-off).

### If the connection to RabbitMQ is lost, our application should reconnect

In case of a connection failure, we can delegate to Spring AMQP and/or Java AMQP to automatically recover the connection, channels and consumers by using `connectionFactory.setAutomaticRecovery(true)`.

<b>Spring AMQP </b>
We have the possibility of setting up exponential back-off retries and maximum number of retries too.

<b>Java AMQP</b>
There are 2 issues with Java AMQP. The first one is that it is very limited and the retry interval is fixed and we define a maximum number of attempts either.
And the second issue is that we need to wait until we have a connection in order to create the queues, exchanges and bindings. This may not be a problem and it boils down to application design and preferences.

### If the library reconnects by itself, should I add a ShutdownListener to the connection?

It really depends on whether the shutdown was initiated by the application and how much you rely on the library to automatically recover “everything” if the shutdown was not initiated by the app.
If the shutdown was initiated by the application, you definitely don’t want to reconnect but to terminate the application. The question is whether you need to do some clean up and where you want to implement them. Should this `ShutdownListener` trigger the clean up of resources or instead should the application logic who initiated the shutdown who is in charge of it?
If the shutdown was not initiated by the application and we rely on the library to automatically recover then why would I need a `ShutdownListener`? Definitely for tracking purposes, either to log it or to record it in some Health Status bean exposed via a `/health` http endpoint.

### Once more, do I need to add a ShutdownListener to each channel?
A channel will receive a shutdown event if:

- the connection closed (`isHardError = true`). If we are using automatic
recovery we don’t have much to do but to wait until it recovers (we can add a RecoveryListener, https://www.rabbitmq.com/api- guide.html#recovery-listeners). Regardless who initiated the close operation, we may want to clean up some resources (e.g. db connection, socket connection, etc) when we disconnect from Rabbit or only when the application terminates.
- the application closed the connection or the channel itself (`initByAp = true`). If we need to do some clean up, when do we do it? On this ShutdownListener’s callback or before this callback?
- The application invoked an operation in the channel that caused an exception (`isHardError = false`). The channel is automatically closed, consumers will stop receiving messages and automatic recovery does not trigger in this case. It is up to the application do deal with it. The producer thread can quickly handle the situation within the try/catch block. It does not need a ShutdownListener. If the exception originates within the consumer’s callback then the consumer has to create a new channel and registers again as a consumer because its subscription is already cancelled. But this scenario could have been handled via the Consumer.handleCancel method.

Regardless what triggers the shutdown event and whether we want to do something about it or not, we should at least track it. Spring AMQP automatically logs these events but we may want to change the logging statements and/or track in any other way.

### Which methods, in the DefaultConsumer class, should I override in addition to handleDelivery?
The other relevant methods are `handleCancel` and `handleShutdownSignal`. We ignore `handleCancelOk` unless we want to do something in our application when we cancel the subscription. Maybe we want to log it for troubleshooting purposes. Or maybe our application has to call some tear down procedure.

`handleCancel` is called when RabbitMq cancels the subscription. This could be happen due to the following reasons:
- The node with non-HA queue goes down while the consumer is connected to a different node, or
- The node with the master HA queue goes down while the consumer is connected to a different node and the consumer has instructed RabbitMq to notify when this happens:

    ```
    Channel channel = ...;
    Consumer consumer = ...;
    Map<String, Object> args = new HashMap<String, Object>(); args.put("x-cancel-on-ha-failover", true);
    ￼￼
    channel.basicConsume("my-queue", false, args, consumer);
    ```

It is advisable that we implement this method so that we can track it (e.g. have a counter and expose that counter thru a `/health` HTTP endpoint) and we can tear down some state and issue another subscription.

It is mandatory to implement this method for non-HA queues. For non-durable we could use this method to re-subscribe but be aware that we need to declare the queue again because it will not exist.

`handleShutdownSignal` is called when either the connection or the channel was closed, unexpectedly or gracefully. Do we need to care about this method if we already have a `ShutdownListener` on the channel and/or on the connection? It really depends on your application design. If your consumer needs to do some tear down procedure, say it has to release some resources (db connection, socket connection, etc.), this method and also the `handleCancel` are very convenient to trigger a tear down procedure. Furthermore if you keep some health or stats state about your subscriptions so that you can expose thru a `/health` HTTP endpoint, you need these two methods to keep that state consistent in addition to `handleCancelOk`.

### Do I need to add an ExceptionHandler to the ConnectionFactory?
By default, Java AMQP and Spring AMQP will log any exception to the standard error

<b>Spring AMQP</b>
If our DefaultConsumer throws an exception from the `handleDelivery` method, Spring automatically rejects the (`requeue`) message so that RabbitMQ can redeliver it again to another consumer.

<b>Java AMQP</b>
In Java AMQP there is a catch we need to be aware of. If we are using client acknowledgements and our consumer’s method handleDelivery throws an exception, and we don’t have a custom ExceptionHandler, the message remains not acknowledged as far RabbitMQ is concern. If our Prefetch size is 1, that means RabbitMQ will not deliver any more messages. When our application closes the channel and/or the connection, RabbitMq makes the message “ready” again.

### How many channels do I need per connection?
There are no hard rules here. We must ask ourselves why would I want to have more than one channel per connection? One pretty valid reason is to separate producers from consumers, in other words, one channel to send messages and another to receive them. Another reason is to separate different type of producers, say one producer sends messages of type A and another producer sends messages of type B and most likely those producers are running simultaneously, i..e on different threads. Another reason, which may not be that evident, is to increase throughput. Rather than having one thread sending data to an exchange, we have several threads sending data to the same exchange over the same connection. Do we gain any increase in throughput? In a multi-core machine, various threads could run on different cores, in parallel, against a single connection. However, as we add more channels to a connection we get a diminish return. We hit a limit, either in the number of cores available or due to heavy contention writing to the single connection. In a 4 cpu machine, I have noticed that beyond 8-10 channels I get diminshing return.

On the client side, a channel may be a costless but on the server side, in RabbitMq, they are expensive in terms of memory and cpu cost. Each channel means 4 Erlang processes with their associated data structures (memory) and stats emissions (cpu) that runs every N seconds which sends stats to the stats db. As the number of channels (and obviously connections) increases, so does the load in RabbitMq.
