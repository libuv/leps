| Title  | Windows HANDLEs not fds |
|--------|-------------------------|
| Author | @vtjnash                |
| Status | DRAFT                   |
| Date   | 2016-09-27              |


## Overview

Currently the libuv API requires passing in a MSVCRT-emulated file-descriptor,
and then immediately discards the wrapper and uses the native HANDLE instead.
This LEP proposes is to consistently remove this extra indirection from all APIs in libuv,
and replace all usages of `uv_os_sock_t` and `uv_file`, with `uv_os_fd_t`.
On Unix systems, this would be a `int fd` while on Windows it would be a `void* HANDLE`.

## Details

Original issue: [https://github.com/libuv/libuv/issues/856]()

On Windows, a file descriptor is emulated in user-space as a wrapper around the real Win32 API.
Indeed, the first operation libuv almost always must do is to throw away the wrapper (by calling `uv__get_osfhandle`).
However, since the wrapper is implemented in user-space, this adds unnecessary overhead,
makes the API less flexible, and is potentially error-prone
(it assumes that the caller and libuv are linking against the same copy of msvcrt).
A couple of functions already accept a `uv_os_sock_t`, essentially just to work around these limitations.
While libuv could continuing adding dual APIs for everything (e.g. see `uv_poll_init` vs. `uv_poll_init_socket`),
this doesn't seem particularly ideal to me.

The limitations of the current API include:
    
- Assumes the caller is linked against the same copy of msvcrt
- Artificial limits the maximum number of open handles to 2048
- The behavior is undefined when called from a GUI program [https://support.microsoft.com/en-us/kb/105305]()
- Some useful APIs (such as `pipe`) aren't emulated by msvcrt.

## Implementation

The `uv__get_osfhandle` translation code would be deleted from everywhere.

Addtionally, constants such as `UV_STDIN_FD` would be provided to provide cross-platform
reference to the standard constants for stdio:
on Unix these are 0, 1, 2; on Windows these are -10, -11, -12.

## Transition

Existing clients can initially transition to the new API by adding the following helper function code snippet:

    HANDLVE os_handle = _get_osfhandle(fd), handle;
    if (!DuplicateHandle(GetCurrentProcess(),
                         os_handle,
                         GetCurrentProcess(),
                         &handle,
                         0,
                         TRUE,
                         DUPLICATE_SAME_ACCESS)) {
      return INVALID_HANDLE_VALUE;
    }
    _close(fd);
    return handle;

Longer term, those clients would now be able to transition to using the Win32 API directly
instead of requiring indirection through the MSVCRT API.
