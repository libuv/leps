| Title  | Pull based event loop  |
|--------|------------------------|
| Author | @saghul                |
| Status | DRAFT                  |
| Date   | 2014-11-27 07:54:35    |


## Overview

This LEP assumes “Request all the things” is implemented.

At this point the libuv event loop basically does the following:

1. Run timers, idle and prepare handles
2. Calculate the poll timeout
3. Block for i/o
4. Run all i/o callbacks
5. Goto 1

Callbacks can be fired at different moments of the loop run process (check, prepare,
idle handle callbacks), making it hard to reason about, and also hard to decompose
and run in stages. Here is the proposed new loop iteration process:

1. Calculate the poll timeout
2. Block for i/o (no callbacks are called)
3. Run all queued callbacks

In order to achieve this every callback in libuv (except allocation callbacks) needs
to  be attached to a request. Those handles which do not use requests for their
operation (`uv_poll_t`, `uv_fs_event_t`, …) will use internal requests to represent this.
Those requests will be queued in *a single* queue which the loop will iterate until
finished, right after polling for i/o. Any request queued while iterating the queue
will be processed in the next iteration.

While this process is internal to libuv, it is exposed with 3 API calls:

~~~~
uint64_t uv_backend_timeout(const uv_loop_t* loop)
~~~~

Returns the amount of time to block for i/o. This function becomes really simple: if
the request queue is non-empty: 0; else, if there are any timeout requests, the nearest
timeout, else infinity.

~~~~
void uv_backend_process(uv_loop_t* loop, uint64_t timeout)
~~~~

This function blocks for i/o for the given amount of time, and it executes all i/o operations,
putting completed requests in the request queue. No callback is called, except for the
allocation callbacks if necessary.

~~~~
void uv_backend_dispatch(uv_loop_t* loop)
~~~~

Runs the callbacks for all queued requests. If any request is added to the queue while
this function is running all callbacks, it will be deferred until the next iteration,
to avoid starvation.

Here is a pseudocode example of a simplified version of uv_run, which would run for ever,
until explicitly stopped:

~~~~
void my_uv_run(uv_loop_t* loop) {
    while (!loop->must_stop) {
        uv_backend_process(loop, uv_backend_timeout(loop));
        uv_backend_dispatch(loop);
    }
}
~~~~

This proposal should also make loop embedding easier, since one thread could block for
i/o and then another one run all callbacks using `uv_backend_dispatch`.

Since all callbacks are now run after polling for i/o, `uv_prepare_t` and `uv_check_t`
handles become obsolete and are removed.
