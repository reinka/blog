+++ author = "Andrei Pöhlmann"
title = "Asynchronous TCP & UDP server with Python"
date = "2023-01-15"
description = "Let's code up our own asynchronous TCP & UDP servers using Python's asyncio module."
tags = [
"networking",
"Python",
"tutorial",
]
categories = [
"networking"
]

thumbnail= ""
+++

Standard examples for coding up a TCP or UDP server using Python usually use the [`socket`](https://docs.python.org/3/library/socket.html#) library which by default operates in a synchronous, blocking mode, i.e. `socket` operations like `accept()`, `recv()`, and `send()` block the execution of your program until they complete their action. You could turn it non-blocking by specifying [`socket.setblocking(False)`](https://docs.python.org/3/library/socket.html#socket.socket.setblocking) but it comes with its own set of challenges and limitations: you need to continuously poll the socket to check if it's ready for reading/writing which can lead to less efficient use of CPU as you might be checking sockets that aren't ready. Additionally, in case of multiple connections, you'll have to track each connection and state, handle partial sends/receives and ensure that your polling loop is efficient.

An alternative to the `setblocking(False)` aproach would be to utilize the [`threading`](https://docs.python.org/3/library/threading.html) module which allows you to handle each client connection in a separate thread. It will make your code more straightforward and easier to read. Since each connection runs in its own thread multiple connections can run concurrently now and blocking operations have become less of an issue. However, threads have also their challenges: managing shared resources between threads is prone to issues like deadlocks, race conditions, etc. Also, due to context switching threads consume more memory and CPU.

Yet another alternative would be using the [`asyncio`](https://docs.python.org/3/library/asyncio.html) module: it provides a single threaded, single process design which can handle multiple connections efficiently. It uses an event loop and non-blocking I/O which is generally more resource-efficient than threads. The drawbacks however are that it can be more challenging to write and understand asynchronous code especially for those not familiar with the concept of coroutines and event loops. And not all libraries are compatible with `asyncio`, i.e. if the a library makes blocking I/O calls (e.g. the [Kubernetes Python Client](https://github.com/kubernetes-client/python)) it can block the entire event loop (though you can use `ThreadPoolExecutor`s as a workaround). So you might need to look out if your library is designed to work with `asyncio`'s event loop.

In the following sections, we'll use the `asyncio` library to code a simple, asynchronous TCP and UDP server. 

# Asynchronous TCP server

Recall that TCP is widely used because it provides reliable, ordered, and error-checked delivery of TCP segments between applications running on hosts communicating over an IP network. To achieve this, TCP establishes a connection between the sender and receiver, ensuring that data is delivered accurately and in the same order it was sent.

Here's an example of how we can create a simple asynchronous TCP server using `asyncio`:

```Python 

import asyncio

async def handle_client(reader, writer):
    # Get client's address
    addr = writer.get_extra_info('peername')

    # Read data from the client
    data_buffer = bytearray()
    while True:
        data = await reader.read(512)
        if not data:
            break
        data_buffer.extend(data)

    message = data_buffer.decode()
    print(f'Received {message} from {addr}')

    # Send the response back to the client
    writer.write(data_buffer)
    await writer.drain()

    # Close the connection
    print(f"Close the connection with {addr}")
    writer.close()

async def run_server():
    server = await asyncio.start_server(
        handle_client, '127.0.0.1', 8888
    )

    addr = server.sockets[0].getsockname()
    print(f'Serving on {addr}')

    async with server:
        await server.serve_forever()

if __name__ == '__main__':
    asyncio.run(run_server())
```

**Client Connection Handling:** 
* `handle_client` is a coroutine that manages individual client connections and is called each time a new client connects to the server. 
* data is read from the client in chunks (up to 512 bytes at a time) and continues `await`ing (**non-blocking!**) new data until the connection is closed or an EOF is sent by the client
* data is echoed back as a response and the connection is closed

**Running the server:**

* server creation is done by the [`asyncio.start_server`](https://docs.python.org/3/library/asyncio-stream.html#asyncio.start_server) function
* the server runs indefinitely within a context manager via `await server.serve_forever()`

**Testing**

After starting up our server, we can test our TCP server using `nc` from another terminal like so:

```bash
nc 127.0.0.1 8888
```


# UDP server

Using Python's `asyncio` module, we can also create an efficient UDP server. In the asyncio framework, handling UDP communication requires a slightly different approach compared to TCP. For the UDP server, we define a custom class that inherits from [`asyncio.DatagramProtocol`](https://docs.python.org/3/library/asyncio-protocol.html#datagram-protocols). This custom class is a key component required by asyncio's [`create_datagram_endpoint`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.create_datagram_endpoint) method, which is used to set up the UDP server.

By creating a class based on `asyncio.DatagramProtocol`, we provide a structure for handling data packets. The class methods, such as [`connection_made`](https://docs.python.org/3/library/asyncio-protocol.html#asyncio.BaseProtocol.connection_made) and [`datagram_received`](https://docs.python.org/3/library/asyncio-protocol.html#asyncio.DatagramProtocol.datagram_received), are designed to manage the stateless nature of UDP. They allow us to handle incoming packets (`datagram_received`), respond to them, and maintain any necessary state or context in the class instance. This approach provides a clean and organized way to encapsulate the UDP server's functionality, making the code easier to manage and extend.


```Python
import asyncio

class EchoUDPProtocol(asyncio.DatagramProtocol):
    def connection_made(self, transport):
        self.transport = transport

    def datagram_received(self, data, addr):
        message = data.decode()
        print(f"Received {message} from {addr}")
        # Echoing back the received message
        self.transport.sendto(data, addr)

async def run_server():
    print("Starting UDP server")
    # Bind to localhost on UDP port 8888
    loop = asyncio.get_running_loop()
    transport, _ = await loop.create_datagram_endpoint(
        lambda: EchoUDPProtocol(),
        local_addr=('127.0.0.1', 8888)
    )

    try:
        await asyncio.sleep(3600)  # Run for 1 hour
    finally:
        transport.close()

if __name__ == '__main__':
    asyncio.run(run_server())

```

**Handling UDP Packets:**

* The `EchoUDPProtocol` class, inheriting from `asyncio.DatagramProtocol`, manages the UDP communication.
* `connection_made` is invoked when the server is ready to accept data. It's important to note that in UDP, this doesn't represent a client connection like in TCP.
* `datagram_received` is called whenever a UDP packet is received. The server decodes the message, prints it, and sends an echo response to the client's address.

**Running the UDP Server:**

* Similar to the TCP server, the UDP server is set up to listen on localhost (`127.0.0.1`) and a specific port (`8888`).
* The server runs within an asyncio event loop. Here, it's configured to run for a fixed duration (1 hour) for demonstration purposes.

**Testing**

After starting up our UDP server we can test our UDP server using `nc`'s `-u` option for UDP:

```bash
nc -u 127.0.0.1 8888
```
