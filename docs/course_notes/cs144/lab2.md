# Lab 2: the TCP receiver

## Overview

In this lab, you will implement the `TCPReceiver`, the part of a TCP implementation that
handles the incoming byte stream.

The `TCPReceiver` receives messages from the peer's sender (via the `receive()` method) and turns them into calls to a `Reassembler`, which eventually writes to the incoming `ByteStream`. Applications read from this `ByteStream`.

Meanwhile, the `TCPReceiver` also generates messages that go back to the peer’s sender, via the `send()` method. These “receiver messages” are responsible for telling the sender:

1. the index of the "first unassembled" byte, which is called the "acknowledgment number"
or "**ackno**". This is the first byte that the receiver needs from the sender.
2. the available capacity in the output ByteStream -  the "**window size**".

Together, the **ackno** and **window** **size** describe describes the receiver’s **window**: a **range** **of** **indexes** that the TCP sender is allowed to send.

## The TCP Receiver

In TCP, 
**acknowledgment** means, "What's the index of the *next* byte that the receiver
needs so it can reassemble more of the `ByteStream`?" **Flow control** means, 
"What range of indices is the receiver interested and willing to receive?" (a function of its available capacity).
This tells the sender how much it's *allowed* to send.

### Translating between 64-bit indexes and 32-bit seqnos

In `Reassembler`, each individual byte has a 64-bit index. 

!!! tip "IMPORTANT"
    In the TCP headers, however, space is precious, and each byte's index in the stream
    is represented not with a 64-bit index but with a 32-bit "sequence number" or "seqno".
    This adds three complexities:

1. **Your implementation needs to plan for 32-bit integers to wrap around.** Streams in TCP can be arbitrarily long — there’s no limit to the length of a ByteStream that can be sent over TCP.
2. **TCP sequence numbers start at a random value**: The first sequence number for a stream
is a *random 32-bit number* called the Initial Sequence Number (ISN). 使用随机数的目的是improve
robustness and avoid getting confused by old segments belonging to earlier connections.
3. **The logical beginning and ending each occupy one sequence number**: The SYN 
(beginning-of-stream) and FIN (end-of-stream) control flags each occupy *one* sequence number.

Sequence numbers (**seqnos**) are transmitted in the header of each TCP segment.

Some terminology:

* **absolute sequence number**: always starts at zero and doesn’t wrap.
* **stream index**: an index for each byte in the stream, starting at zero.

Consider the byte stream `cat`:

|        element |    SYN     |     c      |   a   |   t   |  FIN  |
| -------------: | :--------: | :--------: | :---: | :---: | :---: |
|          seqno | $2^{32}-2$ | $2^{32}-1$ |   0   |   1   |   2   |
| absolute seqno |     0      |     1      |   2   |   3   |   4   |
|   stream index |            |     0      |   1   |   2   |       |

The figure shows the three different types of indexing involved in TCP:

| Sequence Numbers  | Absolute Sequence Numbers | Stream Indices        |
| :---------------- | :------------------------ | :-------------------- |
| Start at the ISN  | Start at 0                | Start at 0            |
| Include SYN/FIN   | Include SYN/FIN           | Omit SYN/FIN          |
| 32 bits, wrapping | 64 bits, non-wrapping     | 64 bits, non-wrapping |
| "seqno"           | "absolute seqno"          | "stream index"        |

Converting between sequence numbers and absolute sequence numbers is tricky.
To prevent these bugs systematically, we’ll represent sequence numbers with a custom type: `Wrap32`, and write the conversions between it and absolute sequence numbers (represented with `uint64_t`).

`Wrap32` is an example of a *wrapper* *type*: a type that contains an inner type (in this case `uint32_t`) but provides a different set of functions/operators.