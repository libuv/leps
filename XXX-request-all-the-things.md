| Title  | Request all the things  |
|--------|-------------------------|
| Author | @saghul                 |
| Status | DRAFT                   |
| Date   | 2014-11-27 07:43:59     |


## Overview

This proposal describes a new approach for dealing with operations in libuv. As of
right now, handles define an entity which is capable of performing certain operations.
These operations are sometimes a result of a request being sent and some other times a
result of a callback (which was passed by the user) being called. This proposal aims
to make this behavior more consistent, by turning several operations that currently
just take a callback into a request form.


### uv_read

(This was previously discussed, but it’s added here for completeness).

Instead of using a callback passed to `uv_read_start`, the plan is to use a `uv_read`
function which performs a single read operation. The initial prototype was defined
as follows:

~~~~
int uv_read(uv_read_t* req,
            uv_stream_t* handle,
            const uv_buf_t[] bufs,
            unsigned int nbufs,
            uv_read_cb, cb)
~~~~

The read callback is defined as:

~~~~
typedef void (*uv_read_cb)(uv_read_t* req, int status)
~~~~

This approach has one problem, though: memory for reading needs to be allocated upfront,
which might not be desirable in all cases. For this reason, a secondary version which takes
an allocation callback is also proposed:

~~~~
int uv_read_alloc(uv_read_t* req,
                  uv_stream_t* handle,
                  uv_alloc_cb alloc_cb,
                  uv_read_cb cb)
~~~~

Applications can use one or the other or mixed without problems.

Implementation details: we probably will want to have some `bufsml` of size 4 where we
copy the structures when the request is created, like `uv_write` does. Thus, the user can
pass a `uv_buf_t` array which is allocated on the stack, as long as the memory in each `buf->base`
is valid until the request callback is called.

Inline reading: if there are no conditions which would prevent otherwise, we could try to do
a read on the spot. This should work ok if the user provided preallocated buffers, because
we can hold on to them if we get EAGAIN. If `uv_read_alloc` is used, instead, we won’t attempt
to read on the spot because the allocation callback would have to be called and we’d end
up holding on to the buffer for too long, thus defeating the purpose of deferred allocation.
A best effort inline reading function is also proposed:

~~~~
int uv_try_read(uv_stream_t* handle,
                const uv_buf_t[] bufs,
                unsigned int nbufs)
~~~~

It does basically the analogous to `uv_try_write`, that is, attempt to read inline and
doesn’t queue a request if it doesn’t succeed.

### uv_stream_poll

In case `uv_read` and `uv_read_alloc` are not enough, another way to read or write on streams
would be to get a callback when the stream is readable / writable, and use the `uv_try_*`
family of functions to perform the reads and writes inline. The proposed API for this:

~~~~
int uv_stream_poll(uv_stream_poll_t* req,
                   uv_stream_t* handle,
                   int events,
                   uv_stream_poll cb)
~~~~

`events` would be a mask composed of `UV_READABLE` and / or `UV_WRITABLE`.

The callback is defined as:

~~~~
typedef void (*uv_stream_poll_cb)(uv_stream_poll_t* req, int status)
~~~~


### uv_timeout

Currently libuv implements repeating timers in the form of a handle. The current implementation
does not account for the time taken during the callback, and this has caused some trouble
every now and then, since people have different expectations when it comes to repeating timers.

This proposal removes the timer handle and makes timers a request, which gets its callback
called when the timeout is hit:

~~~~
int uv_timeout(uv_timeout_t* req,
               uv_loop_t* loop,
               double timeout,
               uv_timeout_cb cb)
~~~~

The `timeout` is now expressed as a double. The fractional part will get rounded up
to platform granularity. For example: 1.2345 becomes 1230 ms or 1,234,500 us,
depending on whether the platform supports sub-millisecond precision.

Timers are one shot, so no assumptions are made and repeating timers can be easily
implemented on top (by users).

The callback takes the following form:

~~~~
typedef void (*uv_timeout_cb)(uv_timeout_t* req, int status)
~~~~

The status argument would indicate success or failure. One possible failure is cancellation,
which would make status == `UV_ECANCELED`.

Implementation detail: Timers will be the first thing to be processed after polling for i/o.


### uv_callback

In certain environments users would like to get a callback called by the event loop, but
scheduling this callback would happen from a different thread. This can be implemented using
`uv_async_t` handles in combination with some sort of thread safe queue, but it’s not
straightforward. Also, many have fallen in the trap of `uv_async_send` coalescing calls,
that is, calling the function X times does not yield the callback being called X times; it’s
called at least once.

`uv_callback` requests will queue the given callback, so that it’s called “as soon as
possible” by the event loop. 2 API calls are provided, in order to make the thread-safe
version explicit:

~~~~
int uv_callback(uv_callback_t* req, uv_loop_t* loop, uv_callback_cb cb)
int uv_callback_threadsafe(uv_callback_t* req, uv_loop_t* loop, uv_callback_cb cb)
~~~~

The callback definition:

~~~~
typedef void (*uv_callback_cb)(uv_callback_t* req, int status)
~~~~

Implementation detail: since the callback request cannot be safely initialized outside
of the loop thread, when `uv_callback_threadsafe` is used, the request will be put
in a queue which will be processed by the loop at some point, fully initializing the
requests.

The introduction of `uv_callback` would deprecate and remove `uv_async_t` handles.
Now, in some cases it might be desired to just wake up the event loop, and having to
create a request might be too much, thus, the following API call is also proposed:

~~~~
void uv_loop_wakeup(uv_loop_t* loop)
~~~~

Which would just wake up the event loop in case it was blocked waiting for i/o.

Implementation detail: the underlying mechanism for waking up the loop will be decided
later on. The current `uv_async_t` machanism could remain (on Unix) or atomic ops
could be used instead.

Note: As a result of this addition, `uv_idle_t` handles will be deprecated an removed.
It may not seem obvious at first, but `uv_callback` achieves the same: the loop won’t block
for i/o if any `uv_callback` request is pending. This becomes even more obvious with the
“‘pull based’ event loop” proposal.


### uv_accept / uv_listen

Currently there is no way to stop listening for incoming connections. Making the concept
of accepting connections also request based makes the API more consistent and easier
to use: if the user decides so (maybe because she is getting EMFILE because she ran
out of file descriptors, for example) she can stop accepting new connections.

New API:

~~~~
int uv_listen(uv_stream_t* stream, int backlog)
~~~~

The uv_listen function loses its callback, becoming the equivalent of `listen(2)`.

~~~~
int uv_accept(uv_accept_t* req, uv_stream_t* stream, uv_accept_cb cb)
typedef void (*uv_accept_cb)(uv_accept_t* req, int status)
~~~~

Once a connection is accepted the request callback will be called with status == 0.
The `req->fd` field will contain a `uv_os_fd_t` value, which the user can use together
with `uv_tcp_open` for example. (This needs further investigation to verify it would
work in all cases).


### A note on uv_cancel

Gradually, `uv_cancel` needs to be improved to allow for cancelling any kind of requests.
Some of them might be a bit harder, but `uv_timeout` and `uv_callback` should be easy
enough to do.
