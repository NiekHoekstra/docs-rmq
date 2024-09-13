# Publish

Once a connection, it is time to start communicating.

1. Establish a channel for communication.
2. Declare a Queue
3. Send a message on the queue.

Creating a channel is done via the connection.<br>
The `BlockingConnection.channel` method does exactly this.<br>
Although this method accepts a `channel_number` argument, 
it is not need and should only be used when it is a necessity.

> **Signature:**<br>
> BlockingChannel.channel(self, channel_number: int | None=None) -> BlockingChannel

This channel can then be used to *declare* a queue.
Declaring can be considered a 'safe' method.
If the queue does not exist, it is created.
If it already exists, no errors are raised.
This method does return the pika frame,
but this isn't interesting for this markdown. 

If the queue is known to exist, this can be handled differently. 
More on this in a future markdown.


> **Signature:**<br>
> BlockingChannel.queue_declare(self, queue_name: str, ...) -> pika.frame.Method

After declaring the queue, the message can be published.

> BlockingChannel.basic_publish(exchange: str, queue: str, data: bytes)

For this message there is no exchange.
Exchanges can be used for a lot of things, but not needed for a basic queue.
An empty string will do just fine.

Note that the data is taken as bytes, not as a textual string.

**Example**

```python
import pika
from pika.adapters.blocking_connection import BlockingChannel

params = pika.ConnectionParameters(
    host='localhost'
)
con: pika.BlockingConnection = pika.BlockingConnection(params)
channel: BlockingChannel = con.channel()
channel.queue_declare('test_queue')
channel.basic_publish('', 'test_queue', b'Hello World')
```
With the message sent, it is time to check it via the web interface..

## Checking via the web interface

1. Open the web interface (http://localhost:15672)
2. Open the tab 'Queues and Streams'
3. Click on the name of the Queue you made.

This screen should tell you how many messages are still in the queue.<br>
This screen also allows reading of message without fully removing them from the queue.

| Ack mode                  | Description                                   |
|---------------------------|-----------------------------------------------|
| Nack message requeue True | Receive 1 or more messages, and requeue them  |
| Automatic Ack             | Receive message, removing them from the queue |
| Reject requeue True       | Receive 1 message, but keep it in RMQ.        |
| Reject requeue False      | Receive 1 message, removing it from RMQ.      |

Click "Get Message(s)" and the next message should be shown.<br>
If the message is *not* requeued, clicking the button again will show the next message.<br>
Without requeue, these messages are permanently removed.