# Getting Started

This is a short but speedy introduction to RabbitMQ and pika.
RabbitMQ has pretty great documentation,
so things here can come pretty close to repackaging those tutorials.


Pika is Python's client for RabbitMQ, and [recommended by RabbitMQ](https://www.rabbitmq.com/tutorials/tutorial-one-python).

The easiest way to get started is by using their container on [docker hub](https://hub.docker.com/_/rabbitmq).<br>
For the sake of clarity, it is recommended to get the `-management` version of the docker.<br>
This version comes with the management plugin, which provides a web interface.


```shell
docker run -d --name rqm-trial -p 15672:15672 rabbitmq:3.12.14-management
```
Or just open all ports
```shell
docker run -d --name rmq-trial -p 4369:4369 -p 5671:5671 -p 5672:5672 -p 15672:15672  rabbitmq:3.12.14-management
```
The main port to expose is 15672.<br>
This port is used for both the web interface 
and the general publish/subscribe system. 

When using SSL/TLS, port 15671 may be used for a secure connection.

See also: [Networking and RabbitMQ (rabbitmq.com)](https://www.rabbitmq.com/docs/networking#ports)

With the docker running, the web interface should be accessible:<br>
Default, plaintext: http://localhost:15672

The docker comes configured with only a single user:

| Username | Password |
|----------|----------|
| guest    | guest    |   

## Testing the connection
Now that the web interface works, it is time to check the connection using pika.

Pika is 100% python, so installation should be a breeze
```shell
python -m pip install pika
```


The connection details can be defined in various ways.
Two classes which can be used are:

1. For a granular approach, the `pika.ConnectionParameters` can be used to 
specify specific parameters and lean on sane defaults.
2. When using a url-style format (useful for CLI tools), 
`pika.URLParameters` can be used to parse the url and connect with its embeded parameters.

For demonstration purposes, `ConnectionParameters` will be used.<br>
There are also different types of connections to facilitate things like AsyncIO.<br>
For the sake of clarity, a `BlockingConnection` is used. <br>(for more details, see below)


```python
import pika
params = pika.ConnectionParameters(
    host='localhost'
)
con: pika.BlockingConnection = pika.BlockingConnection(params)
print("Connection is", "open" if con.is_open else "closed")
```
This code *should* work.

> **ConnectionRefused:** Check and see if the container exposed the right ports.<br>
Once a container is made, reconfiguring ports is a hassle.<br>
In this case, it's better to just recreate the docker.

With a working connection, 
the [official tutorials](https://www.rabbitmq.com/tutorials) should be quite accessible.

This is the end of this little tutorial.
The next file discusses 'publishing'.

For more information on connections, see below.

## Connections

There are several types of connections.
These types are specific to Python,
allowing for multiple implementation strategies.

**pika.BlockingConnection:**
Any I/O based operations are blocking on the calling thread.
This means operations either complete, wait, or fail.
This code is linear, and usually quite easy to read.
Concurrency is not provided,
and the KeyboardInterrupt may be 'ignored' sometimes.

**pika.SelectConnection:**
Polls the connection for events.
This version should not ignore the KeyboardInterrupt in any case.

**pika.adapters.asyncio_connection.AsyncioConnection:**
All I/O based operations are handled by AsyncIO.
Some methods which are relatively logic in BlockingConnection 
are handled differently in AsyncIO, which can be confusing.

**pika.adapters.tornado_connection.TornadoConnection**: (todo)
Used by the Tornado framework.