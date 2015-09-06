## Introduction

RabbitMQ is a well-known implementation of AMQP. It allows you to build distributed scalable systems consisting of decoupled components that use asynchronous messaging.

While the following statement is not technically correct, I personally think of RabbitMQ and AMQP as "the modern JMS" or "JMS made cool".

I've played around with RabbitMQ for a while and found that asynchronous consumption of messages can be tricky. __If you don't completely understand the relation between Connection, Channel and BasicConsumer, your program may not be able to consume messages in parallel__.

This article discusses that issue and the way you can overcome it.

## The issue: async consumption may be not that async

Let's say we have the following consumer program (`ConcurrentRecv`):

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
        channel.exchangeDeclare(QUEUE_NAME, "fanout");
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME, QUEUE_NAME, "");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag,
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

It does the following:

1. Opens a new connection
2. Opens a new channel on that connection
3. Declares an exchange and queue (if it doesn't exist yet). Binds them.
4. Subscribes to the given queue

Now let's assume that some other program produces messages in that queue. I would expect that those messages are going to be consumed in parallel using a [thread pool](https://www.rabbitmq.com/api-guide.html#consumer-thread-pool) that is behind a Connection.

However experiment below shows that it's not the case. `MultipleSend` sends 100 message to the queue:

```java
package wheleph.rabbitmq_tutorial.concurrent_consumers;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;

public class MultipleSend {
    private static final Logger logger = LoggerFactory.getLogger(MultipleSend.class);

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");

        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(QUEUE_NAME, "fanout");
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME, QUEUE_NAME, "");

        for (int i = 0; i < 100; i++) {
            String message = "Hello world" + i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            logger.info(" [x] Sent '" + message + "'");
        }

        channel.close();
        connection.close();
    }
}
```

And the output shows that they are processed by `ConcurrentRecv` in multiple threads but not in parallel:

```
16:25:43,255 [main] ConcurrentRecv -  [*] Waiting for messages. To exit press CTRL+C
16:25:46,777 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world0
16:25:47,278 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world1
16:25:47,778 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world2
16:25:48,279 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world3
16:25:48,779 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world4
16:25:49,279 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world5
16:25:49,780 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world6
16:25:50,280 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world7
16:25:50,781 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world8
16:25:51,281 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world9
16:25:51,782 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world10
16:25:52,283 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world11
16:25:52,783 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world12
16:25:53,284 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world13
16:25:53,784 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world14
16:25:54,285 [pool-1-thread-4] ConcurrentRecv - Received (channel 1) Hello world15
16:25:54,786 [pool-1-thread-5] ConcurrentRecv - Received (channel 1) Hello world16
16:25:55,286 [pool-1-thread-5] ConcurrentRecv - Received (channel 1) Hello world17
```

Why does it happen? [API guide](https://www.rabbitmq.com/api-guide.html#consuming) says:

> Each Channel has its own dispatch thread. For the most common use case of one Consumer per Channel, this means Consumers do not hold up other Consumers. If you have multiple Consumers per Channel be aware that a long-running Consumer may hold up dispatch of callbacks to other Consumers on that Channel.

What it doesn't explicitly says is that the dispatch thread processes incoming messages serially. This is implemented in `com.rabbitmq.client.impl.ConsumerWorkService`.

## Solution 1: use multiple channels

An obvious way to achieve consumption parallelism is to increase the number of listening channels. This approach is used by [Spring AMQP](http://projects.spring.io/spring-amqp/) via `org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer.setConcurrentConsumers(int)`. See this [thread](http://forum.spring.io/forum/spring-projects/integration/amqp/112227-meaning-of-simplemessagelistenercontainer-s-concurrent-consumers).

## Solution 2: use internal thread pool

Another possible way to solve the problem is to make the consumers very lightweight. The only thing they will do is to put each consumed message in an internal thread pool (separated from the one used by `Connection`) and let Java concurrency framework to do the rest. This approach decouples _consuming_ from _processing_.

The program below (`ConcurrentRecv2`) implements this approach:

```java
package wheleph.rabbitmq_tutorial.concurrent_consumers;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.IOException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ConcurrentRecv2 {
    private static final Logger logger = LoggerFactory.getLogger(ConcurrentRecv2.class);

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException, InterruptedException {
        int threadNumber = 2;
        final ExecutorService threadPool =  new ThreadPoolExecutor(threadNumber, threadNumber,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());

        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");

        final Connection connection = connectionFactory.newConnection();
        final Channel channel = connection.createChannel();

        logger.info(" [*] Waiting for messages. To exit press CTRL+C");

        registerConsumer(channel, 500, threadPool);

        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                logger.info("Invoking shutdown hook...");
                logger.info("Shutting down thread pool...");
                threadPool.shutdown();
                try {
                    while(!threadPool.awaitTermination(10, TimeUnit.SECONDS));
                } catch (InterruptedException e) {
                    logger.info("Interrupted while waiting for termination");
                }
                logger.info("Thread pool shut down.");
                logger.info("Done with shutdown hook.");
            }
        });
    }

    private static void registerConsumer(final Channel channel, final int timeout, final ExecutorService threadPool)
            throws IOException {
        channel.exchangeDeclare(QUEUE_NAME, "fanout");
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.queueBind(QUEUE_NAME, QUEUE_NAME, "");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag,
                    Envelope envelope,
                    AMQP.BasicProperties properties,
                    final byte[] body) throws IOException {
                try {
                    logger.info(String.format("Received (channel %d) %s", channel.getChannelNumber(), new String(body)));

                    threadPool.submit(new Runnable() {
                        public void run() {
                            try {
                                Thread.sleep(timeout);
                                logger.info(String.format("Processed %s", new String(body)));
                            } catch (InterruptedException e) {
                                logger.warn(String.format("Interrupted %s", new String(body)));
                            }
                        }
                    });
                } catch (Exception e) {
                    logger.error("", e);
                }
            }
        };

        channel.basicConsume(QUEUE_NAME, true /* auto-ack */, consumer);
    }
}
```

If we launch `ConcurrentRecv2` and then `MultipleSend` from the previous example we'll see a different picture:

```
16:39:56,661 [main] ConcurrentRecv2 -  [*] Waiting for messages. To exit press CTRL+C
16:40:01,116 [pool-2-thread-4] ConcurrentRecv2 - Received (channel 1) Hello world0
16:40:01,116 [pool-2-thread-4] ConcurrentRecv2 - Received (channel 1) Hello world1
16:40:01,116 [pool-2-thread-4] ConcurrentRecv2 - Received (channel 1) Hello world2
... LOTS OF SIMILAR MESSAGES ...
16:40:01,138 [pool-2-thread-10] ConcurrentRecv2 - Received (channel 1) Hello world98
16:40:01,139 [pool-2-thread-3] ConcurrentRecv2 - Received (channel 1) Hello world99
16:40:01,616 [pool-1-thread-1] ConcurrentRecv2 - Processed Hello world0
16:40:01,617 [pool-1-thread-2] ConcurrentRecv2 - Processed Hello world1
16:40:02,117 [pool-1-thread-1] ConcurrentRecv2 - Processed Hello world2
16:40:02,117 [pool-1-thread-2] ConcurrentRecv2 - Processed Hello world3
16:40:02,617 [pool-1-thread-2] ConcurrentRecv2 - Processed Hello world5
```

Here we can see that at first all messages are quickly consumed from the queue and put into internal thread pool. And afterwards 2 threads (`pool-1-thread-1` and `pool-1-thread-2`) process them concurrently.

One important caveat is that once a message is put into the internal thread pool we have to be careful with it because if JVM exits before it's processed, the messages is essentially lost. To prevent this `ConcurrentRecv2` defines a shutdown hook that prevents JVM exit until all the messages are processed.
