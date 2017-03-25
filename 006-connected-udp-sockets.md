| Title  | Connected UDP sockets |
|--------|-----------------------|
| Author | @santigimeno          |
| Status | DRAFT                 |
| Date   | 2017-03-25 21:41:00   |


## Overview

Calling `connect()` on a UDP socket associates a remote address and port to the
socket, so every message sent by this socket is automatically sent to that
destination.

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
- In the event a v1.x patch is desired this could be accomplished by creating
a new method for the connected sockets or by passing `addr=NULL` to the
`send()` methods.

## Reference implementation

See: https://github.com/libuv/libuv/pull/1274