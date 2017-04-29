| Title  | uv_callback          |
|--------|----------------------|
| Author | @kroggen             |
| Status | DRAFT                |
| Date   | 2016-12-12 02:00:00  |


# Proposal: Implementation of uv_callback and deprecation of uv_async

This proposal inherit some ideas from LEP "request all the things" from @saghul but with some differences:

 * It does not require a request
 * It supports call coalescing (and non coalescing)
 * It supports synchronous and asynchronous calls

It also deprecates the uv_async_t.


# Usage Examples


## Sending progress to the main thread

In this case the calls can and must coalesce to avoid flooding the loop if the
work is running too fast.

The call coalescing is enabled using the UV_COALESCE constant.

### In the receiver thread

```C
uv_callback_t progress;

void on_progress(uv_callback_t *handle, void *value) {
   printf("progress: %d\n", (int)value);
}

uv_callback_init(loop, &progress, on_progress, UV_COALESCE);
```

### In the sender thread

```C
uv_callback_fire(&progress, (void*)value, NULL);
```


## Sending allocated data that must be released

In this case the calls cannot coalesce because it would cause data loss and memory leaks.

So instead of UV_COALESCE it uses UV_DEFAULT.

### In the receiver thread

```C
uv_callback_t send_data;

void on_data(uv_callback_t *handle, void *data) {
  do_something(data);
  free(data);
}

uv_callback_init(loop, &send_data, on_data, UV_DEFAULT);
```

### In the sender thread

```C
uv_callback_fire(&send_data, data, NULL);
```


## Firing the callback synchronously

In this case the thread firing the callback will wait until the function
called on the other loop returns.

The main difference from the previous example is the use of UV_SYNCHRONOUS.

This can be used when the worker thread does not have a loop.

### In the receiver thread

```C
uv_callback_t send_data;

void on_data(uv_callback_t *handle, void *data) {
  do_something(data);
  free(data);
}

uv_callback_init(loop, &send_data, on_data, UV_DEFAULT);
```

### In the sender thread

```C
uv_callback_fire(&send_data, data, UV_SYNCHRONOUS);
```


## Firing the callback and getting the result asynchronously

In this case the thread firing the callback will receive the result in its
own callback when the function called on the other thread loop returns.

Note that there are 2 callback definitions here, one for each thread.

### In the called thread

```C
uv_callback_t send_data;

void * on_data(uv_callback_t *handle, void *data) {
  int result = do_something(data);
  free(data);
  return (void*)result;
}

uv_callback_init(loop, &send_data, (uv_callback_cb)on_data, UV_DEFAULT);
```

### In the calling thread

```C
uv_callback_t data_sent;

void on_data_sent(uv_callback_t *handle, void *result) {
  printf("The result is %d\n", (int)result);
}

uv_callback_init(loop, &data_sent, on_data_sent, UV_DEFAULT);

uv_callback_fire(&send_data, data, &data_sent);
```


# Additions

## Typedef

	uv_callback_t 

## Functions

	uv_callback_init
	uv_callback_fire

# Constants

Used in uv_callback_init:

	UV_DEFAULT
	UV_COALESCE

Used in uv_callback_fire:

	UV_SYNCHRONOUS

This last one is declared like this:

```C
#define UV_SYNCHRONOUS  ((uv_callback_t*)-1);
```

It is used in the 3rd argument of uv_callback_fire() function that expects a
uv_callback_t*.

It is based on SQLITE_TRANSIENT constant, used in a function that expects a
pointer to a free function. If we call the function using this constant then
it will deal with it as an option. It can be used because there will never
be a function at address 0xFFFFFFFF (or its 64 bits counterpart).


# Implementation

This first implementation idea is done on top of uv_async, with just small
changes inside the current uv_async code.

## uv_callback_t

The uv_callback_t will have the same fields of uv_async_t with some additions:

```C
#define UV_CALLBACK_PRIVATE_FIELDS   \
  <other fields here>                \
  int usequeue;                      \
  uv_mutex_t mutex;                  \
```

If `usequeue==1` then `callback_t->data` points to a queue (singly linked list).

Optionally the queue can be in a private field instead of the public data field.

The mutex is used to access the queue.

## uv_callback_call_t

This structure will store the information for the non coalescing calls: the data
argument and the notification callback (if supplied) that must be fired after
this fired callback returns.

For each call there will be one of this in a linked list. The pointer to the first
will be in async->data or a private field.

```C
struct uv_callback_call_s {
  uv_callback_call_t *next; /* pointer to the next call in the queue */
  void *data;               /* data argument for this call */
  uv_callback_t *notify;    /* callback to be fired with the result of this one */
};
```

## uv_callback_cb

The callback function must have the data as an argument because the data
will be retrieved from the call queue.

```C
typedef void (*uv_callback_cb)(uv_callback_t* handle, void *data);
```

## uv_callback_init

The uv_callback_init will just initialize the structure with the given values.
Something like this:

```C
int uv_callback_init(
 uv_loop_t* loop,
 uv_callback_t* callback,
 uv_callback_cb callback_cb,
 int callback_type
){

  callback->loop = loop;
  callback->callback_cb = callback_cb;

  switch(callback_type) {
  case UV_DEFAULT:
    callback->usequeue = 1;
    callback->data = NULL;
    break;
  case UV_COALESCE:
    callback->usequeue = 0;
    break;
  default:
    return xxx;
  }

  return 0;
}
```

## uv_callback_fire

The uv_callback_fire will store the call info in the callback_call queue/list and
then probably do the same work of the `uv_async_send`.

```C
int uv_callback_fire(uv_callback_t* callback, void *data, uv_callback_t* notify) {

  if (!callback) return UV_ARGS;

  /* if there is a notification callback set, then the call must use a queue */
  if (notify!=NULL && notify!=UV_SYNCHRONOUS && callback->usequeue==0) return INVALID;  // -- check if allowed for UV_SYNCHRONOUS

  if (callback->usequeue) {
    /* allocate a new call info */
    uv_callback_call_t *call = malloc(sizeof(uv_callback_call_t));
    if (!call) return ENOBUFS;
    /* save the call info */
    call->data = data;
    call->notify = notify;
    /* add the call to the queue */
    uv_mutex_enter(&callback->mutex);
    llist_add(&callback->data, call);
    uv_mutex_leave(&callback->mutex);
  } else {
    callback->data = data;
  }

  /* here insert the code from uv_async_send, maybe modified */
}
```

## Firing the callback in the destination loop

The current uv_async code let the calls coalesce. This is not a problem and the
code can remain almost unchanged.

We only need one call in the destination thread so the values can be retrieved
from the call queue.

Here is a pseudo-code:

```javascript
while(data = dequeue(async->queue)):
  result = fire_callback(async->cb, data);
  if (call->notify) uv_callback_fire(call->notify, result);
  free(call);
```

# Note

If for some reason this LEP is not approved, feel free to get code and ideas from it.
