# AsyncIO

Pika's usage of AsyncIO is based on callback (as opposed to coroutines).
This means many methods are still called in a linear (non-async) manner,
but their parameters are modified to accept a callback function.
A solid understand of AsyncIO's `Task` and `Future` objects is recommended.

**Note:** Callbacks are not allowed to be `async` functions.

As a result, it can be difficult to see how pika interacts with asyncio.
The answer is Tasks. 
Internally, the AsyncIOAdapter will orchestrate all these callbacks.
This is just like regular (synchronous) Python code using `asyncio.get_event_loop().create_task(...)`.

The complicated bit is having these synchronous callbacks interact with AsyncIO.
The simple answer lies in `Future.set_result`.

> **Context:**
> In the background, everything is an asyncio Task.
> This includes the invocation of the callback.
> It's possible to do some complicated things with `get_event_loop()`,
> but it's also possible to leverage `Future` objects.
> By using `Future.set_result`, the task will resume as soon as any running tasks `await`.
> Once the callback returns, it *will* resume. 

**Example** (basic connection)

```python
import asyncio
import time
from typing import Callable

import pika
from pika.adapters.asyncio_connection import AsyncioConnection

OpenCallback = Callable[[AsyncioConnection], None]
OpenErrorCallback = Callable[[AsyncioConnection, Exception], None]

async def main():
    loop = asyncio.get_running_loop()
    future = loop.create_future()
    connection = AsyncioConnection(
        pika.ConnectionParameters(host='localhost'),
        on_open_callback=future.set_result,
        on_open_error_callback=lambda con, ex: future.set_exception(ex)
    )
    print(connection.is_open)
    start = time.time()
    await future
    print(connection.is_open, time.time() - start)    

asyncio.run(main())
```

## Error Handling

AMQP communicates failures by closing the channel or connection. [\[1\]](https://github.com/pika/pika/issues/657#issuecomment-150626308)<br>
When taking the approach of using Future objects, this complicates event handling quite a bit.
An easy test is to create a user with a "configure regex" that is empty,
so it will error (close connection) when trying to declare a queue.

In most cases, the future would need to be accessible for the "close" handler.
This is complicated by concurrency, where multiple callbacks are registered.
Closing a channel or connection can be quite broad, 
meaning *all* of those futures need to have an exception set on them.

```python
import asyncio

from pika.adapters.asyncio_connection import AsyncioConnection

future: asyncio.Future | None = None


def on_close(*args, **kwargs):
    global future
    if future is not None:
        future.set_exception(ConnectionError())


async def main():
    global future
    loop = asyncio.get_running_loop()
    con: AsyncioConnection = ...
    con.add_on_close_callback(on_close)  # could also be an __init__ kwarg.
    
    future = loop.create_future()
    channel = con.channel(on_open_callback=future.set_result)
    channel.add_on_close_callback(on_close)
    await future
    
    future = loop.create_future()
    channel.queue_declare('rmq_q_test', callback=future.set_result)
    await future 

asyncio.run(main())
```
