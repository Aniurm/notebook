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

在这些不同的index之间转换比较tricky，所以引入了一个新的type `Wrap32`，用来表示sequence 
number，同时提供一些转换函数。

1. `static Wrap32 Wrap32::wrap( uint64 t n, Wrap32 zero point )`
      1. 给定absolute sequence number和“零点”，返回一个Wrap32 object(表示sequence number)。
      2. 将absolute sequence number与零点相加后取模就行。
2. `uint64 t unwrap( Wrap32 zero point, uint64 t checkpoint ) const`
      1. 给定一个“零点”和一个checkpoint，将sequence number转化为absolute sequence number。
      2. 因为sequence number是“取模”后的结果，所以有几个可能的值，选择一个最接近checkpoint的值。

### Implementing the TCPReceiver

TCPReceiver要干嘛？

1. receive messages from its peer’s sender and reassemble the `ByteStream` using a `Reassembler`.
      1. 它收到的message be like:

        ```cpp
        struct TCPSenderMessage
        {
            Wrap32 seqno { 0 };

            bool SYN {};
            std::string payload {};
            bool FIN {};

            bool RST {};

            // How many sequence numbers does this segment use?
            size_t sequence_length() const { return SYN + payload.size() + FIN; }
        };
        ```

2. send messages back to the peer’s sender that contain the acknowledgment number (ackno) and window size.

      1. 它发送回去的message be like:

        ```cpp
        struct TCPReceiverMessage
        {
            std::optional<Wrap32> ackno {};
            uint16_t window_size {};
            bool RST {};
        };
        ```

那么我们要implement的实际上是`TCPReceiver`的`receive`和`send`方法。

* `receive()`
    * 接受`TCPSenderMessage`，将sequence number转化为stream index，然后调用`Reassembler`的`insert`方法。
* `send()`
    * 生成`TCPReceiverMessage`，包含ackno和window size。

当然还有一些edge cases，比如：收到`RST`，收到`FIN`等等，会涉及一些index的tricky细节，不再赘述。

因为“重组乱序的字节流”这个问题在Reassembler中已经解决了，所以在TCPReceiver解决的问题
主要是“各种index之间的转换”，难度并不大，但是edge cases比较多。

一些我觉得有趣的小细节：在本次实验中，每一次`receive()`之后都会调用`send()`，将ackno和window size发送回去。
但是在实际的TCP协议中，会有一些优化，用于减少网络中的 ACK 包数量，提高带宽利用率，如：
Delayed ACK（使用一个 ACK 来确认多个数据段）、Piggybacking（将 ACK 放在数据包中）等等。
