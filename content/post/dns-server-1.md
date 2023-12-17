+++ author = "Andrei Pöhlmann"
title = "Building a DNS server from scratch: UDP server & DNS message Header"
date = "2023-12-17"
description = "Let's build our own DNS server from scratch starting with a UDP server and the Header of the DNS message"
tags = [
"networking",
"Python",
"tutorial",
"dns"
]
categories = [
"networking"
]

thumbnail= ""
+++

The best way to learn a technology is to implement it yourself from scratch. Over a series of blog post we will implement our own DNS server in Python.

[RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035) intorduces the implementation and specification of DNS. [Section 4.2. Transport](https://datatracker.ietf.org/doc/html/rfc1035#section-4.2) mentions DNS message transportation can happen both over TCP or UDP. We will implement a UDP version of it which limits the message size to 512 bytes according to [4.2.1. UDP usage](https://datatracker.ietf.org/doc/html/rfc1035#section-4.2.1)

# UDP Server

DNS expects connection requests on port 53 however we will use port 1053 since ports up to 1024 are reserved ones.

We'll start with a simple UDP server that binds to port 1053 on the localhost and responds to every DNS message with the received message:

```python
import socket

def main():
    print("Starting UDP server...")

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind(('127.0.0.1', 1053))
        while True:
            try:
                data, addr = s.recvfrom(512)
                print(f'Received data from {addr}: {data}')

                s.sendto(data, addr)
            except Exception as e:
                print(f'Error receiving data: {e}')
                break

if __name__ == "__main__":
    main()
```

We can start the server via `python3 dns.py` and test it via `dig`, e.g.:


```bash
$ dig @127.0.0.1 -p 1053 hello      
;; Warning: query response not set

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 1053 hello
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17032
;; flags: rd ad; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;hello.				IN	A

;; Query time: 0 msec
;; SERVER: 127.0.0.1#1053(127.0.0.1)
;; WHEN: Sun Dec 17 10:54:42 CET 2023
;; MSG SIZE  rcvd: 34
```

Unsurprisingly, our DNS server did not provide an answer for the domain `hello`.

We should see server logs similar to:

```bash 
$ python3 dns.py
Starting UDP server...
Received data from ('127.0.0.1', 51181): b'\xd5h\x01 \x00\x01\x00\x00\x00\x00\x00\x01\x05hello\x00\x00\x01\x00\x01\x00\x00)\x10\x00\x00\x00\x00\x00\x00\x00'
```

# DNS Message

[Section 4.1. Format](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1) describes the format of a DNS message, which consists of 5 sections:

```
    +---------------------+
    |        Header       |
    +---------------------+
    |       Question      | the question for the name server
    +---------------------+
    |        Answer       | RRs answering the question
    +---------------------+
    |      Authority      | RRs pointing toward an authority
    +---------------------+
    |      Additional     | RRs holding additional information
    +---------------------+
```

We'll implement the header next.

## DNS header

### Understanding the DNS header

The DNS header is a critical component of the Domain Name System, acting as the control panel for every DNS message. This header controls how DNS queries and responses are processed and understood, forming the backbone of how domain names are resolved into IP addresses on the internet.

Its structure is defined in the [Section 4.1.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.1), which outlines the various fields contained within it. To understand the DNS header, it's essential to be familiar with its layout and the significance of each field:

```
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      ID                       |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    QDCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ANCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    NSCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ARCOUNT                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
Each field has a specific purpose:

**ID**: A 16-bit identifier assigned by the program that generates the DNS query. It's used for matching responses with queries.
**QR, Opcode, AA, TC, RD, RA, Z, RCODE**: These flags and codes specify various aspects of the DNS query/response, such as whether the message is a query or a response (QR), the type of query (Opcode), authoritative answer (AA), and more.
**QDCOUNT, ANCOUNT, NSCOUNT, ARCOUNT**: These count fields specify the number of entries in the question (QD), answer (AN), authority (NS), and additional (AR) sections, respectively.

The DNS header's format is further explained in [2.3.2. Data Transmission Order](https://datatracker.ietf.org/doc/html/rfc1035#section-2.3.2). In DNS, the most significant bit is on the left. For example, consider the representation of the number 170 (decimal):

```
     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |1 0 1 0 1 0 1 0|
    +-+-+-+-+-+-+-+-+
```
In this representation, the leftmost bit (labeled 0) is the most significant.

### Implementing the DNS Header

For simplicity, we'll implement a hardcoded DNS header:

```python
import struct

class DNSHeader:
    def __init__(self):
        # Initialize the DNS header fields with default values
        self.id = 1234  # Identifier
        self.qr = 1     # Query/Response Flag
        # Other flag fields: Opcode, AA, TC, RD, RA, Z, and RCODE
        self.opcode = self.aa = self.tc = self.rd = self.ra = self.z = self.rcode = 0
        # Initialize count fields for Question, Answer, Authority, and Additional sections
        self.qdcount = self.ancount = self.nscount = self.arcount = 0

    def to_bytes(self) -> bytes:
        # Combine the flag fields into a single 16-bit field
        flags = (
            (self.qr << 15)
            | (self.opcode << 11)
            | (self.aa << 10)
            | (self.tc << 9)
            | (self.rd << 8)
            | (self.ra << 7)
            | (self.z << 4)
            | self.rcode
        )
        # Pack the header fields into a bytes object
        return struct.pack(
            "!HHHHHH",
            self.id,
            flags,
            self.qdcount,
            self.ancount,
            self.nscount,
            self.arcount,
        )
```

This class defines the structure of a DNS header with all necessary fields. The `to_bytes` method is particularly important as it converts the header information into a byte format for the following reasons:

1. **Binary Encoding of Protocol Fields**: The DNS protocol, as outlined in the RFCs, defines its headers in a binary format. Each field in the header has a specific bit length and position. The `to_bytes` method handles the binary encoding of these fields, packing them into the correct order and format. This process involves using bitwise operations to place each field in its correct position within the header and then using a method like `struct.pack` to combine these fields into a single binary sequence.
2. **Network Communication is Byte-Oriented**: In network protocols, data is transmitted in the form of byte streams. The entities on either end of a communication channel understand and interpret these byte sequences according to the protocol's specifications. Since the DNS header is a structured piece of data with various fields (like IDs, flags, counts), it needs to be converted into a byte stream to be transmitted over the network. Our `to_bytes` method performs this conversion, ensuring the data is correctly formatted for network transmission.
3. **Ensuring Correct Endianness**: Endianness refers to the order in which bytes are arranged into larger numerical values. Different systems may use different endianness conventions (big-endian or little-endian). However, network protocols typically standardize on a specific endianness to ensure consistency across different systems. In the case of DNS and many other network protocols, big-endian (also known as network byte order) is used. The `to_bytes` method ensures that the DNS header fields are packed in big-endian format, making the data correctly interpretable by any system following the DNS protocol, by specifying the `!` as the first character of the format string `!HHHHHH`. Read more about these format strings in [the official documentation of the struct package](https://docs.python.org/3/library/struct.html#byte-order-size-and-alignment)


We can now return the header as a response by our server: 

```python
import socket

def main():
    print("Starting UDP server...")

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind(('127.0.0.1', 1053))
        while True:
            try:
                data, addr = s.recvfrom(512)
                print(f'Received data from {addr}: {data}')

                header = DNSHeader()
                header_bytes = header.to_bytes()
                s.sendto(header_bytes, addr)
            except Exception as e:
                print(f'Error receiving data: {e}')
                break

if __name__ == "__main__":
    main()
```

and observe the following `dig` output:

```bash
$ dig @127.0.0.1 -p 1053 hello
;; Warning: ID mismatch: expected ID 33788, got 1234
```

The output from `dig` indicates a warning about an ID mismatch. This is expected behavior, as our server uses a hardcoded ID, whereas `dig` expects a dynamically matched ID from the query.