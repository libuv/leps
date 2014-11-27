| Title  | Create sockets early |
|--------|----------------------|
| Author | @saghul              |
| Status | ACCEPTED             |
| Date   | 2015-06-19 09:41:00  |


## Overview

Currently we bind sockets late, which creates a number of problems:

- EMFILE errors can happen at unexpected times
- some settings (i.e., keep alive related) cannot be applied before the socket is created)
- certain socket options (such as SO_REUSEPORT) can only be applied before the socket is
  bound, and currently there is no way to get the fd before libuv calls `bind` on it.

Initial proposal://github.com/joyent/libuv/issues/368


## API

~~~~
int uv_tcp_init(uv_loop_t* loop, uv_tcp_t* handle, unsigned int flags);
int uv_udp_init(uv_loop_t* loop, uv_udp_t* handle, unsigned int flags);
~~~~

The `flags` argument is designed for extensibility: the lowest 8 bits are used to express
the socket domain, and the remaining ones are unspecified for now.
The socket domain can be either AF_INET, AF_INET6 or AF_UNSPEC. If AF_UNSPEC is specified then the
previous behavior (creating sockets lazily is maintained).

In order to keep backwards API in libuv v1.x, the following alternate names will be used:

~~~~
int uv_tcp_init_ex(uv_loop_t* loop, uv_tcp_t* handle, unsigned int flags);
int uv_udp_init_ex(uv_loop_t* loop, uv_udp_t* handle, unsigned int flags);
~~~~


## Resolution

This LEP is ACCEPTED and implemented in https://github.com/libuv/libuv/pull/400


## Acknowledgements

The author would like to thank Ben Noordhuis (@bnoordhuis) for his help during
the implementation's review.

