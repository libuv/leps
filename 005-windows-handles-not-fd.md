| Title  | Windows HANDLEs not fds |
|--------|-------------------------|
| Author | @vtjnash                |
| Status | ACCEPTED                |
| Date   | 2016-10-21              |


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
This change would mean libuv no longer needs to add duplicate APIs for every constructor
(e.g. see `uv_poll_init` vs. `uv_poll_init_socket`).

The limitations of the current API include:
    
- Assumes the caller is linked against the same copy of msvcrt
- Artificial limits the maximum number of open handles to 2048
- The behavior is undefined when called from a GUI program [https://support.microsoft.com/en-us/kb/105305]()
- Some useful APIs (such as `pipe`) aren't emulated by msvcrt.
- Requires allocation of the POSIX-compatibility wrapper (wastes memory and could fail)


## Implementation

The `uv_file` typedef would be deleted and replaced by `uv_os_fd_t` uniformly across all APIs.
This has minor code change implications for consumers of the libuv API (see the Transition section below).
But only cosmetic changes to the internal implementation of libuv,
and is invisible to external customers of libuv-based programs (such as child processes).

The `uv_spawn` implementation would be fixed to declare `fd` as a `uv_os_fd_t` instead of a `int`
in `uv_stdio_container_t.data`.

The `uv__get_osfhandle` translation code would be deleted from everywhere.

Addtionally, constants such as `UV_STDIN_FD` would be provided to provide cross-platform
reference to the standard constants for stdio:
on Unix these are 0, 1, 2; on Windows these are -10, -11, -12.

The APIs this change affects are:
 - uv_tty_init
 - uv_guess_handle
 - uv_pipe_open
 - uv_poll_init / uv_poll_init_socket (merged)
 - uv_fs_close
 - uv_fs_read
 - uv_fs_write
 - uv_fs_fstat
 - uv_fs_fsync
 - uv_fs_fdatasync
 - uv_fs_ftruncate
 - uv_fs_sendfile
 - uv_fs_futime
 - uv_fs_fchmod
 - uv_fs_fchown

## Transition

Existing clients can initially transition to the new API by using the following helper function code snippet
(which would be added verbatim to uv.h):

    #ifdef _WIN32


    static inline HANDLE uv_get_osfhandle(int fd) {
      return _get_osfhandle(fd);
    }


    static inline HANDLE uv_convert_fd_to_handle(int fd) {
      HANDLE new_handle;
      if (uv__duplicate_handle(NULL, uv_get_osfhandle(fd), &new_handle))
        return INVALID_HANDLE_VALUE;
      _close(fd);
      return new_handle;
    }


    #else


    static inline int uv_get_osfhandle(int fd) {
      return fd;
    }


    static inline int uv_convert_fd_to_handle(int fd) {
      return fd;
    }


    #endif

Longer term, those clients would now be able to transition to using the Win32 API directly
instead of requiring indirection through the MSVCRT API.
