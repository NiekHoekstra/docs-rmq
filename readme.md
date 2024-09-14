# docs-rmq

This is alternative documentation for RabbitMQ.

The official RabbitMQ documentation is *pretty great*.
This repository demonstrates certain features which *might* not have come up
in the original documentation. This includes a few side notes.

Consider this repository me trying to "learn by teaching".
For those unfamiliar with the concept: to test your comprehension of a subject,
try to teach it to others.

## AsyncIO

The **key feature** for this repository will be AsyncIO (as implemented in pika).
Pika's [API reference](https://pika.readthedocs.io/en/stable/modules/adapters/asyncio.html)
is pretty good, but [their AsyncIO example](https://github.com/pika/pika/blob/96a92379346285a8f53dd0b3f76cb26f45baf350/examples/asynchronous_publisher_example.py)
seems to barely use AsyncIO functions.  It presents AsyncIO as a dedicate I/O layer which only interacts
with pika, and most of the example code is synchronous.

In a broader application landscape, AsyncIO is "shared" between several features.
A basic example would be an ASGI server (like Starlette/FastAPI) which
has to send data to clients, and this data is provided by RabbitMQ.

This repo *should* present more of a step-by-step approach.