# Lab3: the TCP sender

## Overview

The `TCPSender` is a tool that translates *from* an outbound byte stream *to* segments that will become the payloads of unreliable datagrams.

## The TCP Sender

It will be your `TCPSender`’s responsibility to:

* Keep track of the receiver’s window.
* Fill the window when possible, by reading from the `ByteStream`, creating new TCP segments (including SYN and FIN flags if needed), and sending them.
* Keep track of which segments have been sent but not yet acknowledged by the receiver -- we call these “outstanding” segments.
* Re-send outstanding segments if enough time passes since they were sent, and they haven’t been acknowledged yet.

!!! question "Why am I doing this?"
    The basic principle is to send whatever the receiver will allow us to send (filling the window), and keep retransmitting until the receiver acknowledges each segment. This is called **“automatic repeat request” (ARQ)**.

### How does the `TCPSender` know if a segment was lost?

在这个部分，Lab文档讲了很多，包括一些细枝末节的东西，我总结一下核心的idea：

* In addition to *sending* those segments, the `TCPSender` also has to keep track of its *outstanding* *segments* until the sequence numbers they occupy have been fully acknowledged.
    * If the `TCPSender` found that the oldest-sent segment has been outstanding for too long without acknowledgment, it will retransmit this segment.
* What does it mean for "waiting too long"? 
    * 以**retransmission timeout** (RTO)为标准，如果一个segment在RTO时间内没有被acknowledged，那就是“waiting too long”。

来到具体的实现，`TCPSender`将会有一个retransmission timer: an alarm that can be started at a certain time, and the alarm goes off (or “expires”) once the RTO has elapsed.

* 每一次发送segment的时候，如果timer没打开，就打开一个新的timer，计时RTO。
* 如果所有发出的segments都被acknowledged，就关闭timer。
* 随着时间的推移，如果timer过期：
    * 重发最老的outstanding segment。
    * 如果window size is nonzero（说明receiver还有空间接受新的segment）：
        * 记录the number of *consecutive* retransmissions. 若重传次数过多，就认为网络连接失效
        * 重设RTO为之前的两倍 -- “exponential backoff”，目的为减少重传次数，避免网络拥塞。
    * Restart the timer.
* When the receiver gives the sender an ackno that acknowledges the successful receipt of new data:
    * Set the RTO back to its “initial value.”
    * Reset the count of “consecutive retransmissions” back to zero.
    * 如果仍有outstanding segments，restart the timer.

### Implementing the `TCPSender`

#### `void push( const TransmitFunction& transmit );`

这个函数将TCP segments准备好之后，通过`transmit` function发送出去。它让每个segment尽可能大，但不能超过receiver的window size和MAX PAYLOAD SIZE (1452 bytes). 发送segment之后，将其记录到`outstanding_segments`中，
如果是第一次发送，就打开timer。

#### `void receive( const TCPReceiverMessage& msg );`

这个函数处理来自receiver的message，包括acknowledgment number和window size。如果有新的acknowledgment number，
就把对应的segment从`outstanding_segments`中删除，如果所有的segments都被acknowledged，就关闭timer，
如果仍有outstanding segments，就restart timer。

#### `void tick( uint64 t ms_since_last_tick, const TransmitFunction& transmit );`

随着时间的推移，这个函数会被调用。我们实现的时候仅能参考`ms_since_last_tick`，而不能直接使用
真实的时间（方便Lab构造测试）。这个函数也会调用timer的`tick`函数，如果timer过期，就执行上述
的retransmission逻辑。