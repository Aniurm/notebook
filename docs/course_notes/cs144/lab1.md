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

