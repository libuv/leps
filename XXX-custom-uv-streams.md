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
