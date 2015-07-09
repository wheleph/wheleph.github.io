## Introduction

RabbitMQ is a well-known implementation for AMQP protocol. It allows you to build distributed scalable systems consisting of decoupled components that use asynchronous messaging.

While the following statement is not technically correct, I personally think of RabbitMQ and AMQP as "the modern JMS" or "JMS made cool".

I've played around with RabbitMQ for a while and found that asynchronous consumption of messages can be tricky. __If you don't completely understand the relation between Connection, Channel and BasicConsumer, your consumer may and up being serial and not consume messages in parallel___.

This article discusses that possible issue and the way you may overcome it.

## The issue: async consumption may be not that async

Let's say we have class `ConcurrentRecv`:

```java
package wheleph.rabbitmq_tutorial.concurrent_consumers;

import com.rabbitmq.client.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;

public class ConcurrentRecv {
    private static final Logger logger = LoggerFactory.getLogger(ConcurrentRecv.class);

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException, InterruptedException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");

        final Connection connection = connectionFactory.newConnection();
        final Channel channel = connection.createChannel();

        logger.info(" [*] Waiting for messages. To exit press CTRL+C");

        registerConsumer(channel, 500);
    }

    private static void registerConsumer(final Channel channel, final int timeout)
            throws IOException {
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME, QUEUE_NAME, "");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override            public void handleDelivery(String consumerTag,
                                       Envelope envelope,
                                       AMQP.BasicProperties properties,
                                       byte[] body) throws IOException {
                logger.info(String.format("Received (channel %d) %s",
                        channel.getChannelNumber(),
                        new String(body)));

                try {
                    Thread.sleep(timeout);
                } catch (InterruptedException e) {
                }
            }
        };

        channel.basicConsume(QUEUE_NAME, true /* auto-ack */, consumer);
    }
}
```

It does the following stuff:

1. Opens the new connection
2. Opens the new channel on that connection
3. Declares an exchange and queue (if it doesn't exist yet)
4. Subscribes to the given queue

Now let's assume that someone (for example `MultipleSend`) produces 100 messages in the queue being listened. Well I would expect that those messages are going to be consumed in parallel using a thread pool that is behind a Connection.
