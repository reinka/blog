+++ author = "Andrei Pöhlmann"
title = "Bitwise Operations in Python: Encoding Network Protocol Flags into Byte Sequences"
date = "2023-12-19"
description = "Let's dive into the practical world of network programming with Python's bitwise operations"
tags = [
"networking",
"Python",
"dns"
]
categories = [
"networking"
]

thumbnail= ""
+++

Ever wondered if there's an actual use-case for byte and bitwise operations except for Leetcode style problems? Well, here's one.

[In a previous post](https://andrei.poehlmann.dev/post/dns-server-1/) we used Python's [`struct`](https://docs.python.org/3/library/struct.html) module to generate a byte sequence for DNS message headers. This post will expand on that concept, exploring how we can use bitwise operations to efficiently encode various flags and fields of network protocols into byte sequences. We'll delve into the nitty-gritty of combining different protocol-specific flags, demonstrating how to manipulate and shift bits in Python to create a compact and precise byte representation suitable for network transmission. By the end, you'll have a deeper understanding of both the theory and practical application of these techniques in network programming.


# Wait, but why?

Byte sequences are fundamental in network programming, where data must be precisely formatted for transmission. Python's [`struct`](https://docs.python.org/3/library/struct.html) module offers a streamlined way to convert between Python values and C structs represented as Python bytes. This capability is crucial for tasks like creating packet headers or encoding data to meet network protocol specifications. The [`struct`](https://docs.python.org/3/library/struct.html) module enables efficient and accurate data transformation, ensuring data integrity and compliance with protocol requirements, making it an invaluable tool in any network programmer's toolkit.

# Endocding a DNS header to a bytes object

Let's pick up our code from [BUILDING A DNS SERVER FROM SCRATCH: UDP SERVER & DNS MESSAGE HEADER](https://andrei.poehlmann.dev/post/dns-server-1/) and go through it step by step: 


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

[RFC1035](https://www.rfc-editor.org/rfc/rfc1035#section-4.1.1) describes the DNS header as the following bit sequence where each colum represents a bit of length 1:

```
                                1  1  1  1  1  1
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
E.g. ID is a 16-bit (or 2-byte) identifier, QR a 1-bit flag, etc. Note that ID, QDCOUNT, ANCOUNT, NSCCOUNT, and ARCOUNT are of the smae length of 16 bits / 2 bytes. However, the fields QR, Opcode, AA, TC, RD, RA, Z, and RCODE, each ranging from 1 to 4 bits in size, collectively form a composite 16-bit (or 2-byte) field.

That's why in the format string that packs the Python values into a bytes object, we specify 6 2-byte elements, corresponding to the size of each field in the DNS header. This is crucial for maintaining the structure and sequence of the header as defined by the RFC. The `!` in the format string ensures network ('big-endian') byte order, aligning with standard network communication protocols. `H` represents an unsigned short in C, which corresponds to 2 bytes in Python. This is ideal for encoding each field of the DNS header, as they are all 16-bit (or 2-byte) values.

The only extra work we have to do is for the 'flags', i.e. the fields QR, Opcode, AA, TC, RD, RA, Z, and RCODE. These fields, being of different bit lengths, need to be bit-shifted and combined into a single 16-bit field. This process involves shifting each flag field to its correct position in the sequence and using bitwise OR operations to merge them. This technique ensures that each flag occupies its designated place in the header, maintaining the strict structure outlined by the DNS protocol.

## Detailed Bit Manipulation

Here's a detailed description of the bitwise shifting used to construct the `flags`:

1. **QR (Query/Response Flag):**
* `self.qr << 15`: `qr` is the most significant bit. Shifting it left by 15 positions places it at the 16th bit (from the right).
2. **Opcode (Operation Code):** 
* `self.opcode << 11`: `opcode` occupies the next 4 bits. Shifting it left by 11 positions aligns it with bits 12-15.
3. **AA (Authoritative Answer Flag):**
* `self.aa << 10`: `aa` is next and is shifted to bit 11.
4. **TC (Truncation Flag):**
* `self.tc << 9`: `tc` goes to bit 10.
5. **RD (Recursion Desired):**
* `self.rd << 8`: `rd` is placed in bit 9.
6. **RA (Recursion Available):**
* `self.ra << 7`: `ra` occupies bit 8.
7. **Z (Reserved for Future Use):**
* `self.z << 4`: `z` takes the next 3 bits (bits 5-7), so it's shifted left by 4.
8. **RCODE (Response Code):**
* `self.rcode`: Finally, `rcode` occupies the last 4 bits (bits 1-4) and doesn't need shifting.

### Example:

Suppose `qr=1, opcode=0, aa=0, tc=0, rd=0, ra=0, z=0, and rcode=0`.
After shifting and OR-ing: `flags = 1000 0000 0000 0000` in binary (1 shifted to the 16th bit position), which equals `32768` in decimal.



# Decoding a DNS Header from Bytes

Continuing from our exploration of DNS header encoding, let's reverse the process. The `DNSHeader` class now includes a from_bytes static method, allowing us to reconstruct a `DNSHeader` object from a raw byte sequence. This method is particularly useful for parsing received DNS messages.

```python
class DNSHeader:
    def __init__(self, hid=1234, qr=1, opcode=0, aa=0, tc=0, rd=0, ra=0, z=0, rcode=0, qdcount=1, ancount=1, nscount=0, arcount=0):
        self.id = hid
        self.qr = qr
        self.opcode = self.aa = self.tc = self.rd = self.ra = self.z = self.rcode = rcode
        self.ancount = self.qdcount = qdcount
        self.nscount = self.arcount = arcount

    @staticmethod
    def from_bytes(message : bytes) -> 'DNSHeader':
        # start & end indices in bytes
        start, end = (0, 6*2)
        header = message[start:end]
        hid, flags, qdcount, ancount, nscount, arcount = struct.unpack('!HHHHHH', header)
        qr = (flags >> 15) & 0x1
        opcode = (flags >> 11) & 0xF
        aa= (flags >> 10) & 0x1
        tc = (flags >> 9) & 0x1
        rd = (flags >> 8) & 0x1
        ra = (flags >> 7) & 0x1
        z = (flags >> 4) & 0x7
        rcode = flags & 0xF
        return DNSHeader(hid, qr, opcode, aa, tc, rd, ra, z, rcode, qdcount, ancount, nscount,arcount)
```

Let's dissect `from_bytes` to understand how it decodes each field of the DNS header from a byte sequence. 

We begin by extracting the first 12 bytes (6 * 2), which correspond to the DNS header's length. Using `struct.unpack`, we decode these bytes back into the individual fields: `hid`, `flags`, and the counts (`qdcount, ancount, nscount, arcount`).
Next, we extract each flag from the 16-bit flags field using bitwise right-shifts and masks. Let's break down each of these operations.

## Detailed Bit Manipulation

1. **QR (Query/Response Flag):**
* `qr = (flags >> 15) & 0x1`
* The `flags` value is shifted right by 15 bits. This moves the most significant bit (MSB) of the 16-bit flags to the least significant bit (LSB) position.
The result is then `AND`-ed with `0x1` (`00000001` in binary) to isolate this bit. This operation effectively extracts the value of the QR bit.


2. **Opcode (Operation Code):** 
* `opcode = (flags >> 11) & 0xF`
* Here, `flags` is shifted right by 11 bits. This moves the 4-bit Opcode field (bits 12-15) down to the four least significant positions (bits 0-3).
The result is `AND`-ed with `0xF` (`00001111` in binary) to isolate these four bits. This extracts the Opcode value.


3. **AA (Authoritative Answer Flag):**
* `aa = (flags >> 10) & 0x1`
* Similar to QR, `flags` is shifted right by 10 bits, moving the AA bit into the LSB position.
`AND`-ing with `0x1` extracts the AA bit value.

4. **TC (Truncation Flag):**
* `tc = (flags >> 9) & 0x1`
* `flags` is shifted right by 9 bits, aligning the TC bit with the LSB.
The `AND` operation with `0x1` extracts the TC bit.

5. **RD (Recursion Desired):**
* `rd = (flags >> 8) & 0x1`
* The process is similar for the RD bit. `flags` is shifted right by 8 bits, and then `AND`-ed with `0x1`.

6. **RA (Recursion Available):**
* `ra = (flags >> 7) & 0x1`
* Here, `flags` is shifted right by 7 bits, aligning the RA bit with the LSB, and then extracted with an `AND` operation.

7. **Z (Reserved for Future Use):**
* `z = (flags >> 4) & 0x7`
* `flags` is shifted right by 4 bits. This aligns the three bits of the Z field (which is reserved and usually set to 0) to the three LSB positions.
As before, `AND`-ing with `0x7`  extracts these three bits.

8. **RCODE (Response Code):**
* `rcode = flags & 0xF`
* No shift is needed here, as the RCODE field is already in the four least significant bits of the flags.
The `AND` operation with 0xF extracts the value of RCODE.

# Wrapping up

In this post we went through the sophisticated process of encoding and decoding DNS headers using Python's struct module and bitwise operations. Understanding these concepts is crucial in network programming, as it forms the backbone of data transmission protocols.
