| Title  | Threadpool handle    |
|--------|----------------------|
| Author | @saghul              |
| Status | REJECTED             |
| Date   | 2014-11-27 08:00:44  |


## Overview

Every now and then we get questions / complaints regarding the thread pool. It’s too
small, doesn’t scale, and so on. Right now the following stuff is ran on the thread pool:

- DNS requests (`uv_getaddrinfo`, `uv_getnameinfo`)
- File system operations (`uv_fs_*`)
- User code (through `uv_queue_work`)

The thread pool is also global and has a fixed size of 4, which currently can only
be changed using an environment variable.

This proposal explores the possibility of converting the thread pool to a handle,
with the following API:

~~~~
int uv_tpool_init(uv_loop_t* loop, uv_tpool_t* handle, int min_threads, int max_threads)
int uv_tpool_queue_work(uv_work_t*req, uv_tpool_t* handle, uv_work_cb work_cb, uv_after_work_cb after_work_cb)
~~~~

The thread pool would be created with a minimum number of threads, which could then
scale up tot he given max_threads. A resize API could be added in the future. Running
code on the threadpool is only safe from the event loop thread.

`uv_queue_work` would effectively disappear. The user would have to create her own
thread pool and use `uv_tpool_queue_work` manually.

DNS and file system APIs would change in order to accept a `uv_tpool_t` handle instead
of a loop. This gives the user the flexibility to create multiple pools of different
sizes and run DNS operations on one and file system operations in another one, for example.

Example `uv_getaddrinfo`:

~~~~
int uv_getaddrinfo(uv_getaddrinfo_t* req,
                   uv_tpool_t* handle,
                   uv_getaddrinfo_cb getaddrinfo_cb,
                   const char* node,
                   const char* service,
                   const struct addrinfo* hints);
~~~~


# Reasons for rejection

The libuv core team (part of it) gathered together at Node Interactive Amsterdam 2016 and decided
to withdraw this course of action and focus on a pluggable API for threaded operations. A new
LEP will be written about it.
