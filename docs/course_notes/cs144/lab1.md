# Lab 1: stitching substrings into a byte stream

Over the coming weeks, you’ll implement TCP yourself, to provide the byte-stream abstraction between a pair of computers separated by an unreliable datagram network.

* **TCP receiver**: receives datagrams and turns them into a reliable byte stream to be read from the socket by the application.
* **TCP sender**: divides its byte stream into short *segments* so that each segment fits into a single datagram, and sends the segments to the receiver.

The network might reorder these datagrams, or drop them, or deliver them more than once. **The receiver must reassemble the segments into the contiguous stream of bytes that they started out as.**

In this lab you’ll write the data structure that will be responsible for this reassembly: a *Reassembler*.

## Putting substrings in sequence

```cpp
// Insert a new substring to be reassembled into a ByteStream.
void insert( uint64_t first_index, std::string data, bool is_last_substring );

// How many bytes are stored in the Reassembler itself?
uint64_t bytes_pending() const;

// Access output stream reader
Reader& reader() { return output_.reader(); }
```

### What should the Reassembler store internally?

In principle, the `Reassembler` will have to handle three categories of knowledge:

1. Bytes that are the **next bytes** in the stream. The `Reassembler` should push these to the stream (`output_.writer()`) as soon as they are known.
2. Bytes that fit within the stream's available capacity but can't yet be written, because earlier bytes remain unknown. These should be stored internally in the `Reassembler`.
3. Bytes that lie beyond the stream's available capacity. These should be discarded. The `Reassembler`'s will not store any bytes that can't be pushed to the ByteStream either immediately, or as soon as earlier bytes become known.

The goal of this behavior is to **limit the amount of memory** used by the `Reassembler` and 
the `ByteStream`, no matter how the incoming substrings arrive.

![](img/lab1.png)