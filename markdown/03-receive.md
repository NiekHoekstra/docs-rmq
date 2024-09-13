# Receive

Each message **MUST** be acknowledged or rejected.
When the RabbitMQ server dispatches a message to a consumer,
it will not be dispatched to another consumer until it has
been acknowledged or rejected (and maybe requeued).

There are a few ways a message can be resolved.

**ACK:** The message was processed. The message is removed from RMQ.

**NACK:** The message got an error.
By default, it will still remove the message from RMQ.

**Requeue:** On a NACK, this means the message should stay in the queue.
This might be relevant when processing fails due to errors specific to the processing unit.
This would be things like memory errors or connection errors,
which other consumers might not suffer from.

Reading messages can be simple or complex.
Most queues are simple:

1. Receive the message
2. Process the message
3. Tell RabbitMQ the message is done (or rejected).

Sometimes step 2 and 3 are swapped around,
meaning the message was removed from the queue before it was actually processed.

The *first* example will automatically acknowledge the message.
This means step 2 and 3 will be reversed here.
It also means the code does not have to explicitly acknowledge the message.

```python
import pika


def my_handler(channel, method, properties, body):
    print(channel, method, properties, body)


con: pika.BlockingConnection = ...
ch = con.channel()
ch.queue_declare('hello')
ch.basic_consume(
    'rmq_test',
    on_message_callback=my_handler,
    auto_ack=True
)

```

## The Handler

The handler contains 4 arguments, let's discuss those briefly:

**Channel:** The channel it was received on.

**Method:** General (meta) information bound to the message
This is a `pika.spec:Basic.Deliver`.
This class has a few interesting fields:

| Field        | Summary                                             |
|--------------|-----------------------------------------------------|
| delivery_tag | Tag unique to the message, used to (N)ACK messages. |
| redelivered  | True if this message was rejected and requeued.     |
| exchange     | The exchange used for routing                       |
| routing_key  | The 'routing_key', like a queue name.               |

**Properties:** This describes properties of the data.
This might include encoding-type (ASCII, UTF-8) and content-type (text/plain, application/json).
This allows for messages to be supplied in multiple formats,
without having to create a separate queue or transformation queues.
It is typed as `pika.spec.BasicProperties`.

[...more on StackOverflow](https://stackoverflow.com/questions/18403623/rabbitmq-amqp-basicproperties-builder-values)

**Body:** The message body, as bytes.

## Explicit (N)ACK

```python
import random

import pika
from pika.channel import Channel
from pika.spec import Basic, BasicProperties

def my_handler(channel: Channel, deliver: Basic.Deliver, props: BasicProperties, data: bytes):
    print(channel, deliver.routing_key, '>', data.decode('utf-8'))
    if random.randint(0, 10) == 5:
        print('NACK:', data)
        channel.basic_nack(delivery_tag=deliver.delivery_tag, requeue=True)
    else:
        if deliver.redelivered:
            print('ACK on redelivered message', data)
        channel.basic_ack(delivery_tag=deliver.delivery_tag)

def main():
    con: pika.BlockingConnection = ...
    channel = con.channel()
    channel.queue_declare('hello')
    channel.basic_consume(
        'rmq_test',
        on_message_callback=my_handler,
        auto_ack=False  # FALSE
    )
main()
```

## Batching / Prefetching

When messages are small and processing is fast,
it can be nice to handle messages in bulk.
However, RabbitMQ cannot do this at the data level, 
as it does not parse the data it is sending.

For example: If the message is a JSON payload, RabbitMQ will not concat messages
to a single JSON file.

**However,** it is possible to reduce I/O.
The channel can be configured to *enable* prefetching.
This means multiple messages can be sent to the client (our pika client)
and get processed as usual.

This is done by calling `basic_qos(prefetch_count=n)`.
When a channel starts fetching messages, it will always try to buffer this amount. 

**NOTE:** Batching is a two-way street.
If messages are acknowledged one-by-one, the queue will be refilled one-by-one.
This causes I/O and negates the benefits of batching.
To properly implement batching, use the basic_ack on the last message and
use the `multiple=True` flag.

It can be difficult to determine *when* there are no more pending messages.
It is recommended to buffer these messages internally so workers can "look ahead"
and apply the `basic_ack` when the buffer length indicates an end.

**Example:**
```python
import pika
from pika.channel import Channel
from pika.spec import Basic, BasicProperties

count = 0

def my_handler(channel: Channel, deliver: Basic.Deliver, props: BasicProperties, data: bytes):
    global count
    print(channel, deliver.routing_key, '>', data.decode('utf-8'))
    count += 1
    if count == 5:
        channel.basic_ack(delivery_tag=deliver.delivery_tag, multiple=True)
        count = 0

def main():
    con: pika.BlockingConnection = ...
    channel = con.channel()
    channel.basic_qos(prefetch_count=5)
    channel.queue_declare('hello')
    channel.basic_consume(
        'rmq_test',
        on_message_callback=my_handler,
        auto_ack=False  # FALSE
    )
main()
```
