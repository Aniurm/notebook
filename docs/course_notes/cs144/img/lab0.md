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

## An in-memory reliable byte stream