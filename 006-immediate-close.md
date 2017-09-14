| Title  | Immediate Close |
|--------|-----------------|
| Author | @ambrop72       |
| Status | DRAFT           |
| Date   | 2017-09-14      |

## Overview

Currently all libuv handle types can only be closed asynchronously using `uv_close`,
where the memory for the handle may only be released after the provided callback
has been invoked. This makes it harder to use the libuv APIs correctly in certain
application designs. Often the application just wants to close the handle and forget
about it, but it has to manage the asynchronous close operation anyway.

An important benefit of a single-threaded event loop is that it is (in theory)
possible to have the states of all components of an application "perfectly
synchronized". For example for timers, libuv already provides the guarantee that
after `uv_timer_stop` (and `uv_close`), the callback will not be called, until
a subsequent `uv_timer_start`. This means that the application does not have to
worry that the timer has already expired and the callback has been "queued" but not
invoked and cannot be unqueued (note that certain other libraries such as Boost ASIO
do not provide that guarantee). If this guarantee was not provided, applications
would have to be more complex and the code paths handling the situation described
would naturally be less tested/exercised.

It is therefore desired that the design of the libuv APIs maximizes useful guarantees
and minimizes complications/idiosyncrasies which are not inheretly a consequence of
the intended functionality being provided. The difficulty of implementation and
consistency between APIs should not by itself be the reason for not doing so. Such
clean APIs allow application code to more directly represent the desired application
behavior and naturally result in simpler and less buggy software. Being able to close
and immediately forget about a libuv handle when the application no longer needs it
is one such desired capability.

It is acknowledged that for some handle types on some operating systems (e.g. all
networking Windows), it is difficult to support immediate close due to the design
of the underlying OS interface; though it can *theoretically* can be provided,
by sacrificing some efficiency.

Regardless, this proposal is to provide immediate close capability for cases
when there are no underlying *OS resources and associated memory* that must be kept
alive until ongoing operations complete. A proposal to resolve *all* cases may be
created in the future.

## Example of Usefulness

Here is a short demonstration why immediate close is useful. Consider a C++
application design which makes heavy use of RAII for resource management.
In such a design, one may wrap timer functionality in a `Timer` class as shown.

```C++
class Timer : private noncopyable {
public:
  Timer(uv_loop_t *);
  ~Timer();
  void start(uint64_t timeout);
  void stop();
  bool isRunning() const;
protected:
  virtual void timerExpired(Timer *) = 0;
private:
  // implementation details (functions and data fields)
};
```

The Timer class provides the maximum possible guarantees:
- The `timerExpired()` virtual function is called *only* when the timer was running
  and necessarily corresponds to the most recent `start()` call. The timer must have
  transitioned from running to stopped state just prior to the `timerExpired()`
  call.
- The `Timer` object may be destructed from any context within the event loop thread.
  This includes self-destruction from its own `timerExpired()` call.

Now let's use this to implement a class which prints a message 10 times once per
second and reports to its owner when it's done.

```C++
class Printer : private Timer, noncopyable {
public:
  Printer(uv_loop_t *loop) :
    Timer(loop)
  {
    count = 0;
    Timer::start(1000);
  }
protected:
  virtual void printerIsDone() = 0;
private:
  int count;
  void timerExpired(Timer *) override final {
    printf("Hello\n");
    count++;
    if (count == 10) {
      return printerIsDone();
    }
    Timer::start(1000);
  }
};
```

The code is literally pure business logic. There is nothing about cleanup in that
code. And yet the `Printer` class can be destructed by its owner at any time,
and if that is done then `printerIsDone()` will not be called (being a virtual
function on that object, it obviously must not be because the object no longer
exists, but the callback could just as well use a different mechanism like
`std::function`). The automatically generated `~Printer` destructor will call the
`~Timer` destructor, preventing future `timerExpired()` calls and effectively
preventing future `printerIsDone()` calls.

The `Printer` class even allows its owner to destruct it from its own
`printerIsDone()` callback. This is because `Printer::timerExpired` does not access
any part of the `Printer` object after the `printerIsDone()` callback returns. There
is admittedly potential for bugs, however it is easy to spot such bugs as long as one
is aware of their potential. When one need to access the object after calling a
callback, the solution is simple (use a `Timer` with zero timeout or a class built
specifically for this purpose, perhaps with guaranteed LIFO processing).

With these guarantees, the `Printer` class just like `Timer` provides a clear and
safe API. The design and guarantees propagates upward in the application simplifying
code all over the place.

Having demonstrated the usefulness of the described design pattern, we come the the
realization that the `Timer` class cannot be implemented directly with `uv_timer_t`
because `uv_close` wants the user to keep the `uv_timer_t` memory around until it
calls the close callback. The `Timer` implementation can only work around this issue,
by dynamically allocating the `uv_timer_t` object and having the close callback finally
release the allocation.

## User API

For the API, the following is proposed:
- The return type of `uv_close` is changed from `void` to `int`.
- If the callback passed to `uv_close` is not null, the return value is 0 and the
  behavior is otherwise unchanged. This means that the close callback will be called
  when the handle memory can be released; callbacks for outstanding any asynchronous
  requests such as `uv_connect_t` and `uv_write_t` will be called before the close
  callback with `UV_ECANCELED` status.
- If the callback passed to `uv_close` is null, the function either closes the
  handle immediately and returns zero, or if immediate close is not possible does
  nothing and returns `UV_EBUSY`. In the latter case, the application could still
  call `uv_close` with non-null callback.

It is believed that the majority of applications pass non-null close callback, and for
these the change is backward-compatible (disregarding possible warnings about unused
return value).

Applications which call `uv_close` with null callback will generally break, for example
with `UV_EBUSY` result the close will never occur at all. The current documentation does
not specifically mention passing null callback, but it does say that "close_cb will be
called asynchronously after this call" which implicitly requires it to not be null.

An alternative approach would to return `UV_INPROGRESS` and start closing asynchronously
when `uv_close` is called with null callback but the close cannot be immediate. This
change would be more backward compatible with applications currently calling `uv_close`
with null callback. However, it has the issue that an asynchronous close might start
without a callback so the application would need the ability to set it after caling
`uv_close`.

A third option would be to add a new function such as `uv_close_now` which would be
equivalent to calling `uv_close` with null callback as in the first proposal, and to
not modify the behavior of `uv_close`.

A question may arise about whether and when callbacks of outstanding requests
(such as `uv_connect_t` and `uv_write_t`) would be called in case that immediate
close is successful. Two approaches are possible:
- These callbacks are called with `UV_ECANCELED` status from within `uv_close`.
  The advantage of this is that is preserves the existing guarantes that the
  callback of a request is eventually invoked.
- These callbacks are not called. The advantage of this is that this is a simpler
  and safer design. Because applications may not be expecting a callback from
  `uv_close` and may possibly need special cases in the callback implementations.

**However**, for the handle types for which immediate close support is proposed
(in the next section), there are no requests that could be canceled anyway. So if
the implementation of immediate close is initially limited to these handle types,
no decision is yet needed concerning this.

## Implementation

According to an analysis of the current libuv `master` branch code, the following
handle types could support immediate close without much effort: `uv_timer_t`,
`uv_async_t`, `uv_prepare_t`, `uv_check_t`, `uv_idle_t`. For these, immediate close
involves only ensuring that the handle is removed from any data structures and
verifying that immediate-closing a handle cannot cause use-after-free access from
various event dispatching functions.

On Windows, other handle types (at least `uv_tcp_t` and `uv_udp_t`) cannot generally be
closed immediately at any time. Specifically, if there is any I/O operation in progress,
the operation must complete before the OVERLAPPED structure and associated memory
can be released. Since this means that applications would have to support asynchronous
close of these handle types, this proposal recomments to not support immediate close
for these handle types even when it could be done.

On unix, all handle types except `uv_signal_t` could probably support immediate close,
judging from the fact that `uv_close` on these unconditionally calls
`uv__make_close_pending`. However, because such support could not be provided on Windows,
this proposal recommends to not support immediate close for handle types other than ones
mentioned above. Supporting immediate close on unix but not Windows would probably
lead to applications which work correctly on unix but fail on Windows, negating the
usefulness of libuv as a platform abstraction.

## Future Extensions

If this proposal passes, a future proposal may be created to support immediate close
for other commonly used handle types, especially `uv_tcp_t` and `uv_udp_t`.
However it is believed that this would require a new interface for buffer management
which allow libuv to maintain references to buffers; or copying data between application
and internal libuv buffers.
