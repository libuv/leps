| Title  | Connected UDP sockets |
|--------|-----------------------|
| Author | @santigimeno          |
| Status | DRAFT                 |
| Date   | 2017-03-25 21:41:00   |


## Overview

Calling `connect()` on a UDP socket associates a remote address and port to the
socket, so every message sent by this socket is automatically sent to that
destination.

## Motivation

Using connected UDP sockets can add some performance gains in the case that an
application has to send multiple messages to the same destination. Quoting
*Stevens UNIX Network Programming* for a thorough explanation:

> Performance
>
> When an application calls sendto on an unconnected UDP socket,
> Berkeley-derived kernels temporarily connect the socket, send the datagram,
> and then unconnect the socket (pp. 762–763 of TCPv2). Calling sendto for two
> datagrams on an unconnected UDP socket then involves the following six steps
> by the kernel:
>
>     Connect the socket
>
>     Output the first datagram
>
>     Unconnect the socket
>
>     Connect the socket
>
>     Output the second datagram
>
>     Unconnect the socket
>
> Another consideration is the number of searches of the routing table. The
> first temporary connect searches the routing table for the destination IP
> address and saves (caches) that information. The second temporary connect
> notices that the destination address equals the destination of the cached
> routing table information (we are assuming two sendtos to the same
> destination) and we do not need to search the routing table again
> (pp. 737–738 of TCPv2).
>
> When the application knows it will be sending multiple datagrams to the same
> peer, it is more efficient to connect the socket explicitly. Calling connect
> and then calling write two times involves the following steps by the kernel:
>
>     Connect the socket
>
>     Output first datagram
>
>     Output second datagram
>
> In this case, the kernel copies only the socket address structure containing
> the destination IP address and port one time, versus two times when sendto is
> called twice. [Partridge and Pink 1993] note that the temporary connecting of
> an unconnected UDP socket accounts for nearly one-third of the cost of each
> UDP transmission.


## API

The following functions are defined:

```c
int uv_udp_connect(uv_udp_t* handle, const struct sockaddr* addr);

int uv_udp_disconnect(uv_udp_t* handle);

int uv_udp_send(uv_udp_send_t* req,
                uv_udp_t* handle,
                const uv_buf_t bufs[],
                unsigned int nbufs,
                uv_udp_send_cb send_cb);

int uv_udp_sendto(uv_udp_send_t* req,
                  uv_udp_t* handle,
                  const uv_buf_t bufs[],
                  unsigned int nbufs,
                  const struct sockaddr* addr,
                  uv_udp_send_cb send_cb);

int uv_udp_try_send(uv_udp_t* handle,
                    const uv_buf_t bufs[],
                    unsigned int nbufs);

int uv_udp_try_sendto(uv_udp_t* handle,
                      const uv_buf_t bufs[],
                      unsigned int nbufs,
                      const struct sockaddr* addr);
```

Things to notice:

- Calling to `uv_udp_connect()` on an already connected socket causes a call
to `uv_udp_disconnect()` before performing the connection as in some cases, the
kernel returns `EINVAL` on consecutive `connect()` calls.
- Current `uv_udp_send()` and `uv_udp_try_send()` have been renamed to
`uv_udp_sendto()` and `uv_udp_try_sendto()` respectively, as it reflects better
that they are used for *unconnected* UDP sockets. This is a breaking change, so
this proposal should aim for v2.
- `uv_udp_send()` and `uv_udp_try_send()` are going to be used now for
*connected* UDP sockets.
- Trying to use `send()` methods with unconnected sockets will produce an
`UV_EDESTADDRREQ` error.
- Trying to use `sendto()` methods with connected sockets will produce an
`UV_EISCONN` error.
- The bind state of an UDP socket after being disconnected is not consistent
among different platforms. For example on Linux, the socket address and port are
set to 0 whereas on FreeBSD the address is set to 0 but the port is set to the
port the socket was initially bound to.
- In the event a v1.x patch is desired this could be accomplished by creating
a new method for the connected sockets or by passing `addr=NULL` to the
`send()` methods.

## Reference implementation

See: https://github.com/libuv/libuv/pull/1274
