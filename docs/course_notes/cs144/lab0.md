# Lab 0: networking warmup

## Networking by hand

### Fetch a Web page & Send yourself an email

使用 `telnet` 命令，手动发送 HTTP、SMTP 请求。

### Listening and connecting

用 `netcat` 命令监听端口，然后用 `telnet` 命令连接。

## Writing a network program using an OS stream socket

!!! note "有趣的东西"
    - You will make use of a feature provided by the Linux kernel, and by most other operating systems: the ability to create a *reliable bidirectional byte stream* between two programs, one running on your computer, and the other on a different computer across the Internet
        - This feature is known as a *stream socket*.
    - The only thing the Internet really does is to give its “best effort” to deliver short pieces of data, called *Internet datagrams*, to their destination.
    - It’s normally the job of the operating systems on either end of the connection to turn “best-effort datagrams” (the abstraction the Internet provides) into “reliable byte streams” (the abstraction that applications usually want).

### webget

A program to fetch Web pages over the Internet using the operating system’s TCP support and stream-socket abstraction - 跟之前用 `telnet` 发送 HTTP 请求类似。

Starter code 中已经封装好了socket、file descriptor等相关API，按照逻辑调用即可。

* Create a TCP socket
* Connect to the host
* Send the HTTP GET request
* Read the response and write it to the standard output

```
❯ ./apps/webget cs144.keithw.org /hello
HTTP/1.1 200 OK
Date: Mon, 01 Jul 2024 07:05:42 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain
```
```
❯ cmake --build build --target check_webget
Test project /home/ubuntu/CS144-minnow/build
    Start 1: compile with bug-checkers
1/2 Test #1: compile with bug-checkers ........   Passed   16.12 sec
    Start 2: t_webget
2/2 Test #2: t_webget .........................   Passed    1.06 sec

100% tests passed, 0 tests failed out of 2

Total Test time (real) =  17.21 sec
Built target check_webget
```

## An in-memory reliable byte stream

通过上面的操作，我们能感受到 *"reliable byte stream"* 的抽象在网络中的重要性。
所以接下来我们会实现一个类似的抽象，但是在内存中进行。

我们可以从接口的角度来理解它，还是比较直观的：

### Interface

#### Writer

```cpp
void push( std::string data ); // Push data to stream, but only as much as available capacity allows.
void close();                  // Signal that the stream has reached its ending. Nothing more will be written.

bool is_closed() const;              // Has the stream been closed?
uint64_t available_capacity() const; // How many bytes can be pushed to the stream right now?
uint64_t bytes_pushed() const;       // Total number of bytes cumulatively pushed to the stream
```

#### Reader

```cpp
std::string_view peek() const; // Peek at the next bytes in the buffer
void pop( uint64_t len );      // Remove `len` bytes from the buffer

bool is_finished() const;        // Is the stream finished (closed and fully popped)?
uint64_t bytes_buffered() const; // Number of bytes currently buffered (pushed and not popped)
uint64_t bytes_popped() const;   // Total number of bytes cumulatively popped from stream
```

### Implementation

```cpp
class ByteStream
{
  // other stuff...

  uint64_t capacity_;
  uint64_t bytes_pushed_;
  uint64_t bytes_popped_;
  bool closed_;
  std::string buffer_;
  bool error_ {};
};
```

**最核心的问题：应该用什么数据结构来实现这个 buffer？**

我这里使用了 `std::string`，在`push`, `pop`等操作时，调用字符串的相关方法即可。

#### Example

```cpp
void Writer::push( string data )
{
  if ( closed_ || error_ )
    return;
  uint64_t available = available_capacity();
  uint64_t to_write = min( available, static_cast<uint64_t>( data.size() ) );
  buffer_.append( data, 0, to_write );
  bytes_pushed_ += to_write;
}
```

!!! failure
    我之前使用了`std::deque`，但它的内存不是连续的，所以无法直接从中创建 `std::string_view`。

    测试时会报错：

    ```
    =================================================================
    ==21534==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x615000000f00 at pc 0x7facec0240ad bp 0x7ffd51365bc0 sp 0x7ffd51365368
    READ of size 17 at 0x615000000f00 thread T0
        #0 0x7facec0240ac in MemcmpInterceptorCommon(void*, int (*)(void const*, void const*, unsigned long), void const*, void const*, unsigned long) ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:932
        #1 0x7facec024556 in __interceptor_memcmp ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:964
        #2 0x7facec024556 in __interceptor_memcmp ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:959
        #3 0x5647548b4425 in std::char_traits<char>::compare(char const*, char const*, unsigned long) (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x7c425) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #4 0x5647548d19da in std::basic_string_view<char, std::char_traits<char> >::compare(std::basic_string_view<char, std::char_traits<char> >) const (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x999da) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #5 0x5647548c772f in bool std::operator==<char, std::char_traits<char> >(std::basic_string_view<char, std::char_traits<char> >, std::__type_identity<std::basic_string_view<char, std::char_traits<char> > >::type) (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x8f72f) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #6 0x5647548bb6d1 in PeekOnce::execute(ByteStream&) const (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x836d1) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #7 0x5647548c9135 in TestHarness<ByteStream>::execute(TestStep<ByteStream> const&) (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x91135) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #8 0x5647548b2590 in stress_test(unsigned long, unsigned long, unsigned long) /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:58
        #9 0x5647548b3c8e in program_body() /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:77
        #10 0x5647548b3cb7 in main /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:84
        #11 0x7faceb3d0d8f in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58
        #12 0x7faceb3d0e3f in __libc_start_main_impl ../csu/libc-start.c:392
        #13 0x5647548b0a04 in _start (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x78a04) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)

    0x615000000f00 is located 0 bytes after 512-byte region [0x615000000d00,0x615000000f00)
    allocated by thread T0 here:
        #0 0x7facec039ba8 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:95
        #1 0x5647548ea4c8 in std::__new_allocator<char>::allocate(unsigned long, void const*) /usr/include/c++/13/bits/new_allocator.h:147
        #2 0x5647548e4453 in std::allocator<char>::allocate(unsigned long) /usr/include/c++/13/bits/allocator.h:198
        #3 0x5647548e4453 in std::allocator_traits<std::allocator<char> >::allocate(std::allocator<char>&, unsigned long) /usr/include/c++/13/bits/alloc_traits.h:482
        #4 0x5647548e4453 in std::_Deque_base<char, std::allocator<char> >::_M_allocate_node() /usr/include/c++/13/bits/stl_deque.h:583
        #5 0x5647548df1ff in std::_Deque_base<char, std::allocator<char> >::_M_create_nodes(char**, char**) /usr/include/c++/13/bits/stl_deque.h:684
        #6 0x5647548d50ee in std::_Deque_base<char, std::allocator<char> >::_M_initialize_map(unsigned long) /usr/include/c++/13/bits/stl_deque.h:658
        #7 0x5647548f8067 in std::_Deque_base<char, std::allocator<char> >::_Deque_base() /usr/include/c++/13/bits/stl_deque.h:460
        #8 0x5647548f7fe6 in std::deque<char, std::allocator<char> >::deque() /usr/include/c++/13/bits/stl_deque.h:855
        #9 0x5647548f6a9c in ByteStream::ByteStream(unsigned long) /home/ubuntu/CS144-minnow/src/byte_stream.cc:6
        #10 0x5647548b63d0 in ByteStreamTestHarness::ByteStreamTestHarness(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, unsigned long) (/home/ubuntu/CS144-minnow/build/tests/byte_stream_stress_test_sanitized+0x7e3d0) (BuildId: c6d7996a388c87d75222a8e99a442fa23629da7b)
        #11 0x5647548b14f8 in stress_test(unsigned long, unsigned long, unsigned long) /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:24
        #12 0x5647548b3c8e in program_body() /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:77
        #13 0x5647548b3cb7 in main /home/ubuntu/CS144-minnow/tests/byte_stream_stress_test.cc:84
        #14 0x7faceb3d0d8f in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:58

    SUMMARY: AddressSanitizer: heap-buffer-overflow ../../../../src/libsanitizer/sanitizer_common/sanitizer_common_interceptors.inc:932 in MemcmpInterceptorCommon(void*, int (*)(void const*, void const*, unsigned long), void const*, void const*, unsigned long)
    Shadow bytes around the buggy address:
    0x615000000c80: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
    0x615000000d00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    0x615000000d80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    0x615000000e00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    0x615000000e80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    =>0x615000000f00:[fa]fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
    0x615000000f80: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
    0x615000001000: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
    0x615000001080: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
    0x615000001100: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
    0x615000001180: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
    Shadow byte legend (one shadow byte represents 8 application bytes):
    Addressable:           00
    Partially addressable: 01 02 03 04 05 06 07 
    Heap left redzone:       fa
    Freed heap region:       fd
    Stack left redzone:      f1
    Stack mid redzone:       f2
    Stack right redzone:     f3
    Stack after return:      f5
    Stack use after scope:   f8
    Global redzone:          f9
    Global init order:       f6
    Poisoned by user:        f7
    Container overflow:      fc
    Array cookie:            ac
    Intra object redzone:    bb
    ASan internal:           fe
    Left alloca redzone:     ca
    Right alloca redzone:    cb
    ==21534==ABORTING
    ```

## Summary

Lab0以练手为主，较简单。