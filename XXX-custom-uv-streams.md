| Title  | Custom UV Streams    |
|--------|----------------------|
| Author | @coderkevin          |
| Status | DRAFT                |
| Date   | 2015-03-23  22:25:12 |

## Overview

Currently libuv has a limited number of stream types it supports.  As libuv is used in more types of environments,
it makes sense to allow the stream structures to be extended to custom types.  Overall, the data structures of the
streams support this type of abstraction, leaving the code handling of streams to be augmented to handle a new type
of custom stream.  This will rely on each custom stream implementing its own functionality via function pointers,
which will be called in lieu of the normal thread handling that is done within the stream code for the existing
streams currently.

This proprosal revolves around adding a custom stream type that can be extended.  The primary difference between this new stream and the existing ones is that it will be an abstract class itself as well, and require the implementation of several function pointers as well as wrapping existing API functions for each specific implementation of a custom stream.

### File Descriptors

One potentially contentious issue with supporting custom streams is file descriptors.  Some custom streams may present a file descriptor, in which case the operation is vastly similar to the existing streams in the system.  However, other types of custom streams may only present a data buffer, and not a file descriptor or file handle.  This leaves one of two ways to handle such streams.  Either we deal with them separately in the loop (like the timer heap, for example), or we manufacture a file descriptor or file handle for them.

In order to match the way existing streams are done, it is more desirable to ensure that all custom streams have an associated file descriptor or file handle that can be polled as normal.  This leaves the routing of a non-fd stream to an fd interface as an exercise for the individual custom stream implementor.  However, as a part of this effort, it makes sense to demonstrate how this can be done with simple memory buffers, and it would be useful for testing as well.  Such an exercise would provide a reference implementation for those desiring to create a non-fd custom stream implementation.  See **Reference Implementation** below for details.

### Affected files

Since stream handling is pervasive throughout many source files, it will be necessary to add the custom stream handling into each of these files.  The approach here would be to implement this once for all custom streams, and map the specific stream-related functionality to function pointers that can then be extended by each custom stream type.  Therefore, all source files which deal with the existing streams will need to have custom stream handling as well.  Additionally, a custom.c file would be appropriate for holding the implementation details of the custom streams.

### Naming

The naming of the custom stream structures and functions will follow the precedents set by other streams, such as "uv_tcp_XXX" and "uv_pipe_XXX".  Therefore, all custom-stream related code will be prefixed with uv_custom_XXX.  Examples follow:

 * Custom Stream: ```uv_custom_s``` ```uv_custom_t```
 * Custom Stream API Functions: ```uv_custom_XXX```
 * Custom Stream Internal Functions: ```uv__custom_XXX```

### Data Structures

The basis of the custom stream implementation will look much the same as existing streams, except for two main differences.  Custom streams will have a void pointer context to hold relevant data for each custom stream type, and all internal stream functionality will be implemented using function pointers within the custom struct.  It will be up to each custom stream type implementation to provide the appropriate function pointers and manage the context they store for each stream.  Also, it is expected that each custom stream type will have its own API functions to initialize, open, and perform any other custom functionality.

``` C
/* custom stream implementation functions */
typedef int (*uv_custom_accept)(uv_stream_t* server, uv_stream_t* server);
typedef int (*uv_custom_listen)(uv_stream_t* server, int backlog, uv_connection_cb cb);
typedef int (*uv_custom_close)(uv_handle_t* handle);

/* custom stream data structure */
struct uv_custom_s {
  UV_HANDLE_FIELDS
  UV_STREAM_FIELDS
  UV_CUSTOM_PRIVATE_FIELDS
};

#define UV_CUSTOM_PRIVATE_FIELDS      \
  void* context;                      \
  uv_custom_accept accept;            \
  uv_custom_listen listen;            \
  uv_custom_close close;
  
/* custom stream API functions */
uv_custom_init(uv_loop_t* loop, uv_custom_t* handle);
uv_custom_open(uv_custom_t* handle, int fd);
uv_custom_close(uv_custom_t* handle);
```

Note that the custom stream will expect a normal fd for use and will set up an io_watcher with the fd just like TCP, UDP, and Named Pipes.  In this way, custom streams can reuse much of the core code for data flow.  It will be expected for the custom stream implementor to facilitate and provide such an fd upon stream open.

### Platform Abstraction and Build Architecture

Although there is a stream.c file that is platform specific, the custom stream function pointers are designed to match the Stream API, and not the underlying implementation for each platform.  This allows the custom stream implementation to handle the specifics of each platform as that stream requires.  It is up to the implementor of each custom stream type to decide upon which platforms to support.  It is expected that the implementation of each custom stream will be built separately and will bring in libuv as a dependency of its build process on each platform it supports.

### Reference Implementation

For testing and implementation purposes, a simple reference implementation will be created.  This will involve a simple memory buffer implementation of a custom stream that can be sent from the same process, or another process via shared memory.  This implementation would be a good start for anyone looking to use shared memory with their custom stream as well.

#### UNIX

In Linux and other UNIX environments, POSIX shared memory can be used to fulfill the file descriptor needs.  When a shared memory region is created, a file descriptor is returned and can be used for this purpose.

References:

[SHM Overview](http://man7.org/linux/man-pages/man7/shm_overview.7.html)

[How to create an fd for a memory region](http://stackoverflow.com/questions/12081720/in-linux-how-to-create-a-file-descriptor-for-a-memory-region)

#### Windows

For Windows, shared memory has to have an associated file in the system.  This is due to the permissions system for windows requiring common permissions between processes.  So, for the Windows implementation, a file will need to be created and used for the memory map.  This should not affect performance, as the memory buffer will stay resident in memory as long as it is active, and will be cleared on startup.

References:

[Managing Memory Mapped Files](https://msdn.microsoft.com/en-us/library/ms810613.aspx)
