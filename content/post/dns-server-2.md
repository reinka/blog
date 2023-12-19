+++ author = "Andrei Pöhlmann"
title = "Building a DNS server from scratch: DNS message Question & Answer"
date = "2023-12-20"
description = "Let's continue build our own DNS server from scratch by adding the DNS Question & Answer sections"
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


Building upon the foundations laid in our [previous post on creating a DNS server](https://andrei.poehlmann.dev/post/dns-server-1/), we now progress to the next critical component: handling the DNS message's Question and Answer sections. This blog post will explore how to construct these two sections, which are vital for the server's ability to interpret queries and provide appropriate responses. We'll delve into the intricacies of encoding domain names into the DNS query format and unpacking them from raw byte data, using Python's powerful structuring capabilities.

# DNS Question section 

## Understanding the DNS Question section

The DNS Question section, as specified in [RFC 1035 Section 4.1.2](https://www.rfc-editor.org/rfc/rfc1035#section-4.1.2), contains the query name (QNAME), query type (QTYPE), and query class (QCLASS). 

```
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                     QNAME                     /
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QTYPE                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QCLASS                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```


The QNAME field is of variable length and represents the domain name being queried, broken down into labels and encoded in a specific format: each label is preceded by its length (represented by one byte). The entire QNAME ends with a zero-length octet (null byte).

QTYPE specifies the type of query (e.g., A, MX, etc.), and QCLASS denotes the class of the query, typically the Internet (IN). Both of these fields have a length of 2 bytes.

## Impelemtning the DNS Question

As we'll see in the next section, the DNS Answer needs to encode the domain name the same way like the DNS Qeustion, which is why we outsource it to a standalone helper fucntion `encode_str_to_bytes`:

```python3
def encode_str_to_bytes(data: str) -> bytes:
    parts = data.split(".")
    result = b""
    for part in parts:
        length = len(part)
        result += length.to_bytes(1, byteorder="big") + part.encode()
    result += b"\x00"
    return result

class DNSQuestion:
    def __init__(self, domain: str, qtype: str = 1, qclass: str = 1) -> None:
        self.qname = self.encode(domain)
        self.qtype = qtype
        self.qclass = qclass

    def encode(self, domain: str) -> bytes:
        return encode_str_to_bytes(domain)

    def to_bytes(self) -> bytes:
        return self.qname + struct.pack("!HH", self.qtype, self.qclass)
```

Most of the work is done by our encoder: it takes a domain name, splits it into its constituent labels, and encodes each label with its length in bytes, followed by the label itself. The entire sequence is then terminated with a null byte, adhering to the [DNS encoding standard described in RFC1035](https://www.rfc-editor.org/rfc/rfc1035#section-4.1.2). 

The DNSQuestion class encapsulates this logic, storing the domain name, query type, and query class, and providing a method `to_bytes` to combine these elements into the correct byte format for DNS queries (see the [previous post here](https://andrei.poehlmann.dev/post/python-bitwise-ops/) if you're not familiar with the `struct` module).

# DNS Answer section

# ## Understanding the DNS Answer section 

```
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                                               /
    /                      NAME                     /
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TYPE                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     CLASS                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      TTL                      |
    |                                               |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                   RDLENGTH                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--|
    /                     RDATA                     /
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

The DNS Answer is a little bit more complex then the question since it contains the resolved IP address of the domain the DNS Question asks for encoded in bytes (RDATA). 

The domain name is repeated from the Question section and is followed by the type and class, which are typically the same as in the Question. 

The TTL field dictates the duration for which the answer can be cached. 

The RDLENGTH field specifies the length of the RDATA, which is crucial as it varies depending on the type of record. For an 'A' record, the RDATA will be the IP address of the queried domain, encoded in a 4-byte format for IPv4 addresses.


## Impelemtning the DNS Answer

Here's an example how DNS answer implementation might look like:

```python3
class DNSAnswer:
    def __init__(
        self,
        name: str,
        ip: str,
        atype: int = 1,
        aclass: int = 1,
        ttl: int = 60,
        rdlength: int = 4,
    ) -> None:
        self.name = self.encode(name)
        self.type = (atype).to_bytes(2, byteorder="big")
        self.aclass = (aclass).to_bytes(2, byteorder="big")
        self.ttl = (ttl).to_bytes(4, "big")
        self.length = (rdlength).to_bytes(2, "big")
        self.rdata = self.ipv4_to_bytes(ip)

    def encode(self, data: str) -> bytes:
        return encode_str_to_bytes(data)

    def ipv4_to_bytes(self, ip: str) -> bytes:
        res = b""
        for part in ip.split("."):
            res += int(part).to_bytes(1, "big")

        return res

    def to_bytes(self) -> bytes:
        return self.name + self.type + self.aclass + self.ttl + self.length + self.rdata
```

It encodes the domain name using the previously defined `encode_str_to_bytes` helper function. 

The name, type, aclass, ttl, and rdlength fields are encoded into byte format according to RFC1035.

The `ipv4_to_bytes` method converts the IP address into its byte representation. 

The final `to_bytes` method concatenates all these fields, forming a complete DNS Answer section in the correct byte format for transmission.


# Putting it together

With the code from our previous posts [here](https://andrei.poehlmann.dev/post/dns-server-1/) and [here](https://andrei.poehlmann.dev/post/python-bitwise-ops/), our DNS server can now listen to a request and parse the header from that request. Since we haven't written a parser for the DNS question yet, we'll just assume an IPv4 A record and hardcode it in a first step like so: 

```python3
import socket
import struct


class DNSHeader:
    def __init__(
        self,
        hid: int = 1234,
        qr: int = 1,
        opcode: int = 0,
        aa: int = 0,
        tc: int = 0,
        rd: int = 0,
        ra: int = 0,
        z: int = 0,
        rcode: int = 0,
        qdcount: int = 1,
        ancount: int = 1,
        nscount: int = 0,
        arcount: int = 0,
    ):
        self.id = hid
        self.qr = qr
        self.opcode = opcode
        self.aa = aa
        self.tc = tc 
        self.rd = rd
        self.ra = ra 
        self.z = z 
        self.rcode = rcode
        self.ancount = ancount
        self.qdcount = qdcount
        self.nscount = nscount
        self.arcount = arcount

    @staticmethod
    def from_bytes(message: bytes) -> "DNSHeader":
        # start & end indices in bytes
        start, end = (0, 6 * 2)
        header = message[start:end]
        hid, flags, qdcount, ancount, nscount, arcount = struct.unpack(
            "!HHHHHH", header
        )
        qr = (flags >> 15) & 0x1
        opcode = (flags >> 11) & 0xF
        aa = (flags >> 10) & 0x1
        tc = (flags >> 9) & 0x1
        rd = (flags >> 8) & 0x1
        ra = (flags >> 7) & 0x1
        z = (flags >> 4) & 0x7
        rcode = flags & 0xF
        return DNSHeader(
            hid,
            qr,
            opcode,
            aa,
            tc,
            rd,
            ra,
            z,
            rcode,
            qdcount,
            ancount,
            nscount,
            arcount,
        )

    def to_bytes(self) -> bytes:
        flags = (
            (self.qr << 15)
            | (self.opcode << 11)
            | (self.aa << 10)
            | (self.tc << 9)
            | (self.rd << 8)
            | (self.ra << 7)
            | (self.z << 4)
            | (self.rcode)
        )
        return struct.pack(
            "!HHHHHH",
            self.id,
            flags,
            self.qdcount,
            self.ancount,
            self.nscount,
            self.arcount,
        )


def encode_str_to_bytes(data: str) -> bytes:
    parts = data.split(".")
    result = b""
    for part in parts:
        length = len(part)
        result += length.to_bytes(1, byteorder="big") + part.encode()
    result += b"\x00"
    return result


class DNSQuestion:
    def __init__(self, domain: str, qtype: str = 1, qclass: str = 1) -> None:
        self.qname = self.encode(domain)
        self.qtype = qtype
        self.qclass = qclass

    def encode(self, domain: str) -> bytes:
        return encode_str_to_bytes(domain)

    def to_bytes(self) -> bytes:
        return self.qname + struct.pack("!HH", self.qtype, self.qclass)


class DNSAnswer:
    def __init__(
        self,
        name: str,
        ip: str,
        atype: int = 1,
        aclass: int = 1,
        ttl: int = 60,
        rdlength: int = 4,
    ) -> None:
        self.name = self.encode(name)
        self.type = (atype).to_bytes(2, byteorder="big")
        self.aclass = (aclass).to_bytes(2, byteorder="big")
        self.ttl = (ttl).to_bytes(4, "big")
        self.length = (rdlength).to_bytes(2, "big")
        self.rdata = self.ipv4_to_bytes(ip)

    def encode(self, data: str) -> bytes:
        return encode_str_to_bytes(data)

    def ipv4_to_bytes(self, ip: str) -> bytes:
        res = b""
        for part in ip.split("."):
            res += int(part).to_bytes(1, "big")

        return res

    def to_bytes(self) -> bytes:
        return self.name + self.type + self.aclass + self.ttl + self.length + self.rdata


def main():
    print("Starting UDP server...")

    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
        s.bind(("127.0.0.1", 2053))
        while True:
            try:
                data, addr = s.recvfrom(512)
                print(f"Received data from {addr}: {data}")
                header = DNSHeader.from_bytes(data)
                # ovwerrite received flags for our reply
                header.qr, header.ancount, header.arcount, header.nscount = 1, 1, 0, 0
                domain = "example.com"
                q = DNSQuestion(domain)
                a = DNSAnswer(domain, "8.8.8.8")
                s.sendto(header.to_bytes() + q.to_bytes() + a.to_bytes(), addr)
            except Exception as e:
                print(f"Error receiving data: {e}")
                break


if __name__ == "__main__":
    main()
```

If we now start our server

```bash
$ python3 dns.py
Starting UDP server...
```

and run the following `dig` command in another terminal we should see the following:

```bash
$ dig @127.0.0.1 -p 2053 example.com

; <<>> DiG 9.10.6 <<>> @127.0.0.1 -p 2053 example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31351
;; flags: qr; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            60      IN      A       8.8.8.8

;; Query time: 0 msec
;; SERVER: 127.0.0.1#2053(127.0.0.1)
;; WHEN: Tue Dec 19 15:11:49 CET 2023
;; MSG SIZE  rcvd: 56

```

Note, that `dig`'s [ID mismatch warning of our first version](https://andrei.poehlmann.dev/post/dns-server-1/) is gone now because we parsed the ID from the request and used it to construct our response DNS message.

# Next steps

We now have a DNS server that can listen to a request and parse the header from it. In a next post we'll extend our `DNSQuestion` class to parse the question section of the request so that we can get rid of the hard-coded assumptions made above.
