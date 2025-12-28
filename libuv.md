# Libuv: The Asynchronous I/O Foundation

*A systems-level dissection of the event-driven engine powering Node.js*

---

## Part I: Genesis and Architectural Philosophy

### What Libuv Actually Is

Libuv is not merely an "async library" — it's a **platform abstraction layer** that unifies disparate OS-level I/O mechanisms into a single, consistent event-driven interface. At its core, it's a sophisticated **multiplexed I/O reactor** combined with a **thread pool executor** for operations that cannot be made truly asynchronous at the kernel level.

The name itself reveals its origin: **lib** (library) + **uv** (originally "Unicorn Velociraptor," later retroactively interpreted as standing for "Unix & Vert.x" or simply as a meaningless suffix). It was extracted from Node.js in 2011 when Ryan Dahl and the core team realized they needed a cross-platform abstraction that could handle the impedance mismatch between Unix's rich I/O multiplexing facilities and Windows' fundamentally different I/O completion model.

### The Fundamental Problem Libuv Solves

Operating systems provide wildly different mechanisms for asynchronous I/O:

- **Linux**: `epoll` — edge-triggered or level-triggered readiness notifications
- **BSD/macOS**: `kqueue` — generalized event notification mechanism  
- **Solaris**: `event ports` — Solaris 10's scalable event completion
- **Windows**: `IOCP` (I/O Completion Ports) — true asynchronous I/O with completion notifications

These aren't just different APIs — they represent **fundamentally different concurrency models**:

```c
// Unix readiness model (epoll/kqueue)
// "Tell me when data is READY to be read"
epoll_wait(epfd, events, maxevents, timeout);
// You still must call read() to get the data

// Windows completion model (IOCP)  
// "Tell me when the I/O operation has COMPLETED"
GetQueuedCompletionStatus(iocp, &bytes, &key, &overlapped, timeout);
// The data is already in your buffer
```

Libuv reconciles this philosophical divide while exposing a unified API that feels natural in an event-driven architecture.

---

## Part II: Core Architecture — The Event Loop

### The Central Nervous System

The event loop is libuv's execution coordinator. It's not a simple `while(1)` loop — it's a phase-based state machine that orchestrates multiple I/O backends, timer wheels, and thread pool results.

**Source reference**: `src/unix/core.c:uv_run()` and `src/win/core.c:uv_run()`

```c
// Simplified event loop structure (from uv.h)
struct uv_loop_s {
  // Platform-specific I/O polling handle
  void* backend_fd;          // epoll fd (Linux), kqueue fd (BSD)

  // Timer management
  void* timer_heap;          // Min-heap of timers
  uint64_t time;             // Cached current time

  // Handle queues
  void* handle_queue[2];     // Active handles
  void* active_reqs;         // Pending requests

  // Thread pool for blocking operations
  uv__work* wq[2];           // Work queue
  uv_mutex_t wq_mutex;       // Thread safety

  // Phase-specific queues
  void* pending_queue;       // I/O callbacks ready to fire
  void* idle_handles;        // Idle handles
  void* prepare_handles;     // Pre-I/O poll handles
  void* check_handles;       // Post-I/O poll handles

  // Platform abstraction
  void* platform_fields;     // OS-specific data
};
```

### The Phase Execution Model

Each call to `uv_run()` executes phases in a precise order:

```
┌───────────────────────────┐
│       Update Time         │  Cache loop->time = uv__hrtime()
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│    Timers Execution       │  Execute expired timers
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│   Pending I/O Callbacks   │  Execute deferred I/O callbacks
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│    Idle Handles           │  Run uv_idle_t callbacks
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│    Prepare Handles        │  Run uv_prepare_t callbacks
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│     I/O Poll              │  epoll_wait / kevent / GQCS
│   (with timeout)          │  Block waiting for I/O events
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│    Check Handles          │  Run uv_check_t callbacks
└─────────────┬─────────────┘
              │
┌─────────────▼─────────────┐
│    Close Callbacks        │  Execute close callbacks
└─────────────┬─────────────┘
              │
              └─────────────► Loop again if active handles/requests exist
```

**Source**: `src/unix/core.c:uv_run()` — the actual implementation:

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);           // Phase 1: Update cached time
    uv__run_timers(loop);            // Phase 2: Run expired timers

    ran_pending = uv__run_pending(loop);  // Phase 3: Pending I/O
    uv__run_idle(loop);              // Phase 4: Idle handles
    uv__run_prepare(loop);           // Phase 5: Prepare handles

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);  // Calculate timeout

    uv__io_poll(loop, timeout);      // Phase 6: I/O POLL — THE CORE

    uv__run_check(loop);             // Phase 7: Check handles
    uv__run_closing_handles(loop);   // Phase 8: Close callbacks

    if (mode == UV_RUN_ONCE) {
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
```

### Why This Phase Architecture?

This design allows **predictable callback ordering** and **starvation prevention**:

1. **Timers first** — Time-based events get priority
2. **Pending callbacks** — Previously deferred I/O callbacks (from errors, etc.)
3. **Idle/Prepare** — Housekeeping before blocking
4. **I/O Poll** — The only blocking phase
5. **Check** — Process results immediately after I/O
6. **Close** — Resource cleanup happens last

The timeout calculation is critical:

```c
int uv_backend_timeout(const uv_loop_t* loop) {
  if (loop->stop_flag != 0)
    return 0;

  if (!uv__has_active_handles(loop) && !uv__has_active_reqs(loop))
    return 0;

  if (loop->idle_handles)
    return 0;  // Don't block if idle handles exist

  if (loop->pending_queue)
    return 0;  // Don't block if I/O callbacks pending

  if (loop->closing_handles)
    return 0;

  return uv__next_timeout(loop);  // Time until next timer
}
```

This ensures the loop **blocks only when appropriate** and wakes up precisely when work arrives or timers expire.

---

## Part III: Platform I/O Abstraction

### The Unix Backend — epoll on Linux

On Linux, libuv uses `epoll` in **level-triggered mode** by default for simplicity, though it internally tracks state to emulate edge-triggered behavior where beneficial.

**Source**: `src/unix/linux-core.c:uv__io_poll()`

```c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  struct epoll_event events[1024];
  struct epoll_event* pe;
  struct epoll_event e;
  int fd;
  int nfds;
  int i;

  // ... setup code ...

  nfds = epoll_wait(loop->backend_fd,      // epoll instance
                    events,                 // event buffer
                    ARRAY_SIZE(events),     // max events
                    timeout);               // timeout in ms

  // Update cached time after potentially blocking
  uv__update_time(loop);

  for (i = 0; i < nfds; i++) {
    pe = events + i;
    fd = pe->data.fd;

    // Retrieve I/O watcher associated with this fd
    w = loop->watchers[fd];

    if (w == NULL) {
      epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
      continue;
    }

    // Translate epoll events to libuv events
    w->cb(loop, w, pe->events);  // Invoke callback
  }
}
```

The watcher callback (`w->cb`) is **not** the user's JavaScript callback — it's an internal libuv function that translates the raw epoll event into handle-specific logic.

### The Windows Backend — IOCP Architecture

Windows is fundamentally different. IOCP is a **completion-based** model where you submit operations and get notified when they're **done**, not when they're ready.

**Source**: `src/win/core.c:uv__io_poll()`

```c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  OVERLAPPED_ENTRY overlappeds[128];
  ULONG count;
  ULONG_PTR key;
  OVERLAPPED* overlapped;
  uv_req_t* req;

  // GetQueuedCompletionStatusEx is the modern batching API
  success = GetQueuedCompletionStatusEx(
    loop->iocp,           // I/O completion port
    overlappeds,          // Output array
    ARRAY_SIZE(overlappeds),
    &count,
    timeout,
    FALSE);               // Not alertable

  uv__update_time(loop);

  for (i = 0; i < count; i++) {
    overlapped = overlappeds[i].lpOverlapped;

    if (overlapped == NULL) {
      // Handle special wake-up messages
      continue;
    }

    // Retrieve request from OVERLAPPED structure
    // (The OVERLAPPED is embedded in uv_req_t via CONTAINING_RECORD macro)
    req = uv__overlapped_to_req(overlapped);

    // Dispatch to request-specific completion handler
    req->u.io.overlapped.cb(loop, req, overlappeds[i].dwNumberOfBytesTransferred);
  }
}
```

**Key Windows-specific technique**: Libuv embeds `OVERLAPPED` structures inside its request objects, then uses pointer arithmetic (`CONTAINING_RECORD`) to recover the full request when Windows returns the completion:

```c
#define CONTAINING_RECORD(address, type, field) \
  ((type *)((char *)(address) - offsetof(type, field)))

// Example usage:
typedef struct {
  uv_req_t base;
  OVERLAPPED overlapped;  // Windows structure
  // ... other fields ...
} uv_write_t;

// When IOCP returns an OVERLAPPED*:
uv_write_t* req = CONTAINING_RECORD(overlapped, uv_write_t, overlapped);
```

This is a classic **intrusive data structure** pattern used extensively in kernel programming (like Linux's `list_head`).

---

## Part IV: Handle Types — The I/O Primitives

Libuv exposes various handle types, each representing a different I/O resource. All inherit from `uv_handle_t`:

```c
// Base handle structure (from uv.h)
struct uv_handle_s {
  void* data;                    // User data pointer
  uv_loop_t* loop;               // Owning event loop
  uv_handle_type type;           // Handle type enum
  uv_close_cb close_cb;          // Close callback
  void* handle_queue[2];         // Intrusive list pointers

  union {
    int fd;                      // Unix file descriptor
    void* reserved[4];           // Platform-specific
  } u;

  UV_HANDLE_PRIVATE_FIELDS       // Platform-specific expansion
};
```

### TCP Handle — Network I/O Deep Dive

**Source**: `src/unix/tcp.c` and `src/win/tcp.c`

A TCP handle wraps a socket file descriptor and integrates with the event loop's I/O polling mechanism.

**Connection establishment** on Unix:

```c
int uv_tcp_connect(uv_connect_t* req,
                   uv_tcp_t* handle,
                   const struct sockaddr* addr,
                   uv_connect_cb cb) {
  int r;

  // Create socket if needed
  if (handle->io_watcher.fd == -1) {
    r = uv__socket(addr->sa_family, SOCK_STREAM, 0);
    if (r < 0)
      return r;
    handle->io_watcher.fd = r;
  }

  // Make socket non-blocking
  uv__nonblock(handle->io_watcher.fd, 1);

  // Attempt connection (will return EINPROGRESS)
  r = connect(handle->io_watcher.fd, addr, addrlen);

  if (r != 0 && errno != EINPROGRESS)
    return UV__ERR(errno);

  // Register for POLLOUT events (writable when connected)
  uv__io_start(handle->loop,
               &handle->io_watcher,
               POLLOUT);

  // Store request state
  req->handle = (uv_stream_t*)handle;
  req->cb = cb;
  handle->connect_req = req;

  return 0;
}
```

When `epoll_wait` returns with `EPOLLOUT`, the internal watcher callback fires:

```c
static void uv__stream_io(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  uv_stream_t* stream = container_of(w, uv_stream_t, io_watcher);

  if (events & POLLOUT) {
    if (stream->connect_req) {
      uv__stream_connect(stream);  // Complete connection
    } else {
      uv__write(stream);            // Continue pending writes
    }
  }

  if (events & POLLIN) {
    uv__read(stream);               // Read available data
  }
}
```

**Connection completion handler**:

```c
static void uv__stream_connect(uv_stream_t* stream) {
  uv_connect_t* req = stream->connect_req;
  int error;
  socklen_t errorsize = sizeof(int);

  // Check if connection succeeded or failed
  getsockopt(stream->io_watcher.fd, SOL_SOCKET, SO_ERROR,
             &error, &errorsize);

  stream->connect_req = NULL;
  uv__req_unregister(stream->loop, req);

  if (error == 0) {
    uv__io_start(stream->loop, &stream->io_watcher, POLLIN);
  }

  // Invoke user's connect callback
  req->cb(req, (error == 0) ? 0 : UV__ERR(error));
}
```

### Reading from TCP — The Buffer Allocation Strategy

Libuv uses a **callback-based buffer allocation** strategy to avoid unnecessary copies:

```c
int uv_read_start(uv_stream_t* stream,
                  uv_alloc_cb alloc_cb,
                  uv_read_cb read_cb) {
  stream->alloc_cb = alloc_cb;
  stream->read_cb = read_cb;

  uv__io_start(stream->loop, &stream->io_watcher, POLLIN);
  return 0;
}
```

When data arrives:

```c
static void uv__read(uv_stream_t* stream) {
  uv_buf_t buf;
  ssize_t nread;

  // Ask user to allocate buffer
  stream->alloc_cb((uv_handle_t*)stream, 64 * 1024, &buf);

  do {
    nread = read(stream->io_watcher.fd, buf.base, buf.len);
  } while (nread < 0 && errno == EINTR);

  if (nread > 0) {
    // Invoke read callback with data
    stream->read_cb(stream, nread, &buf);
  } else if (nread == 0) {
    // EOF
    stream->read_cb(stream, UV_EOF, &buf);
  } else {
    // Error
    stream->read_cb(stream, UV__ERR(errno), &buf);
  }
}
```

This design allows the **user to control memory management** — Node.js, for instance, reuses buffers from a pool.

---

## Part V: Thread Pool — Handling Blocking Operations

Not all I/O can be made asynchronous at the kernel level:

- **File I/O** on most Unix systems (no true async file I/O kernel API)
- **DNS resolution** (getaddrinfo is blocking)
- **User-defined CPU-intensive work**

Libuv solves this with a **fixed-size thread pool** (default: 4 threads, configurable via `UV_THREADPOOL_SIZE`).

**Source**: `src/threadpool.c`

```c
// Thread pool global state
static uv_once_t once = UV_ONCE_INIT;
static uv_cond_t cond;
static uv_mutex_t mutex;
static unsigned int idle_threads;
static unsigned int nthreads;
static uv_thread_t* threads;
static uv_thread_t default_threads[4];
static QUEUE exit_message;
static QUEUE wq;               // Work queue
static QUEUE run_slow_work_message;
static QUEUE slow_io_pending_wq;

// Work item structure
struct uv__work {
  void (*work)(struct uv__work* w);      // Worker function
  void (*done)(struct uv__work* w, int status);  // Completion callback
  uv_loop_t* loop;
  void* wq[2];                           // Queue pointers
};
```

**Enqueueing work**:

```c
void uv__work_submit(uv_loop_t* loop,
                     struct uv__work* w,
                     void (*work)(struct uv__work* w),
                     void (*done)(struct uv__work* w, int status)) {
  uv_once(&once, init_once);  // Lazy thread pool initialization

  w->loop = loop;
  w->work = work;
  w->done = done;

  uv_mutex_lock(&mutex);
  QUEUE_INSERT_TAIL(&wq, &w->wq);

  if (idle_threads > 0)
    uv_cond_signal(&cond);  // Wake a sleeping worker

  uv_mutex_unlock(&mutex);
}
```

**Worker thread main loop**:

```c
static void worker(void* arg) {
  struct uv__work* w;
  QUEUE* q;

  for (;;) {
    uv_mutex_lock(&mutex);

    while (QUEUE_EMPTY(&wq)) {
      idle_threads++;
      uv_cond_wait(&cond, &mutex);  // Sleep until work arrives
      idle_threads--;
    }

    q = QUEUE_HEAD(&wq);

    if (q == &exit_message)  // Shutdown signal
      break;

    QUEUE_REMOVE(q);
    uv_mutex_unlock(&mutex);

    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);  // Execute blocking work

    // Signal completion to event loop
    uv_async_send(&w->loop->wq_async);
  }
}
```

**Completion notification** uses `uv_async_t` — a handle that allows threads to safely wake up the event loop:

```c
static void uv__work_done(uv_async_t* handle) {
  struct uv__work* w;
  uv_loop_t* loop = handle->loop;

  for (;;) {
    uv_mutex_lock(&loop->wq_mutex);

    if (QUEUE_EMPTY(&loop->wq)) {
      uv_mutex_unlock(&loop->wq_mutex);
      return;
    }

    q = QUEUE_HEAD(&loop->wq);
    QUEUE_REMOVE(q);
    uv_mutex_unlock(&loop->wq_mutex);

    w = container_of(q, struct uv__work, wq);
    w->done(w, 0);  // Invoke completion callback on event loop thread
  }
}
```

This architecture ensures **thread safety** — the worker threads never touch the event loop directly; they only communicate via lock-protected queues and async handles.

---

## Part VI: File System Operations

### The Disconnect Between POSIX and Async

POSIX file APIs (`open`, `read`, `write`, `close`) are **inherently synchronous**. Even with `O_NONBLOCK`, they don't provide true async semantics for regular files. Linux's `io_uring` (available since kernel 5.1) changes this, but libuv still primarily uses the thread pool for portability.

**File read implementation** (`src/unix/fs.c`):

```c
int uv_fs_read(uv_loop_t* loop,
               uv_fs_t* req,
               uv_file file,
               const uv_buf_t bufs[],
               unsigned int nbufs,
               int64_t offset,
               uv_fs_cb cb) {
  // Setup request
  INIT(READ);
  req->file = file;
  req->bufs = (uv_buf_t*)bufs;
  req->nbufs = nbufs;
  req->off = offset;

  if (cb) {
    // Async path — submit to thread pool
    POST;
  } else {
    // Sync path — execute immediately
    uv__fs_work(&req->work_req);
    return req->result;
  }
}
```

**Thread pool work function**:

```c
static void uv__fs_work(struct uv__work* w) {
  uv_fs_t* req = container_of(w, uv_fs_t, work_req);

  switch (req->fs_type) {
  case UV_FS_READ:
    if (req->off >= 0) {
      // Positioned read
      req->result = pread(req->file,
                          req->bufs[0].base,
                          req->bufs[0].len,
                          req->off);
    } else {
      // Sequential read
      req->result = read(req->file,
                         req->bufs[0].base,
                         req->bufs[0].len);
    }
    break;

  case UV_FS_OPEN:
    req->result = open(req->path, req->flags, req->mode);
    break;

  // ... other operations ...
  }

  if (req->result < 0)
    req->result = UV__ERR(errno);
}
```

**Completion callback**:

```c
static void uv__fs_done(struct uv__work* w, int status) {
  uv_fs_t* req = container_of(w, uv_fs_t, work_req);

  uv__req_unregister(req->loop, req);

  if (status == UV_ECANCELED) {
    req->result = UV_ECANCELED;
  }

  req->cb(req);  // Invoke user's callback on event loop thread
}
```

### Filesystem Watching — OS-Specific Mechanisms

Libuv abstracts platform file watchers:

- **Linux**: `inotify`
- **macOS**: `FSEvents` (preferred) or `kqueue`
- **Windows**: `ReadDirectoryChangesW`
- **BSD**: `kqueue` with `EVFILT_VNODE`

**Source**: `src/unix/linux-inotify.c`

```c
int uv_fs_event_start(uv_fs_event_t* handle,
                      uv_fs_event_cb cb,
                      const char* path,
                      unsigned int flags) {
  int wd;

  // Lazy initialize inotify fd
  if (handle->loop->inotify_fd == -1) {
    handle->loop->inotify_fd = inotify_init1(IN_NONBLOCK | IN_CLOEXEC);
    uv__io_init(&handle->loop->inotify_read_watcher,
                uv__inotify_read,
                handle->loop->inotify_fd);
    uv__io_start(handle->loop, &handle->loop->inotify_read_watcher, POLLIN);
  }

  // Add watch
  wd = inotify_add_watch(handle->loop->inotify_fd,
                         path,
                         IN_MODIFY | IN_ATTRIB | IN_MOVE | IN_CREATE | IN_DELETE);

  handle->wd = wd;
  handle->cb = cb;

  // Store in loop's watch descriptor table
  handle->loop->inotify_watchers[wd] = handle;

  return 0;
}
```

When events arrive:

```c
static void uv__inotify_read(uv_loop_t* loop, uv__io_t* w, unsigned int revents) {
  struct inotify_event* e;
  uv_fs_event_t* handle;
  char buf[4096];
  ssize_t nread;

  while (1) {
    nread = read(w->fd, buf, sizeof(buf));
    if (nread <= 0)
      break;

    for (char* p = buf; p < buf + nread; p += sizeof(*e) + e->len) {
      e = (struct inotify_event*)p;

      handle = loop->inotify_watchers[e->wd];
      if (!handle)
        continue;

      // Translate inotify event to libuv event
      int events = 0;
      if (e->mask & (IN_MODIFY | IN_ATTRIB))
        events |= UV_CHANGE;
      if (e->mask & (IN_CREATE | IN_DELETE | IN_MOVE))
        events |= UV_RENAME;

      handle->cb(handle, e->len ? e->name : NULL, events, 0);
    }
  }
}
```

---

## Part VII: DNS Resolution — The getaddrinfo Dilemma

DNS lookups are **inherently blocking** in POSIX (via `getaddrinfo()`). There's no standard async DNS API, so libuv routes all DNS through the thread pool.

**Source**: `src/unix/getaddrinfo.c`

```c
int uv_getaddrinfo(uv_loop_t* loop,
                   uv_getaddrinfo_t* req,
                   uv_getaddrinfo_cb cb,
                   const char* hostname,
                   const char* service,
                   const struct addrinfo* hints) {
  size_t hostname_len = hostname ? strlen(hostname) + 1 : 0;
  size_t service_len = service ? strlen(service) + 1 : 0;
  size_t hints_len = hints ? sizeof(*hints) : 0;
  size_t len = hostname_len + service_len + hints_len;

  // Allocate request with embedded strings
  req->loop = loop;
  req->cb = cb;
  req->addrinfo = NULL;
  req->hints = NULL;
  req->hostname = NULL;
  req->service = NULL;
  req->retcode = 0;

  // Copy strings into request
  char* buf = uv__malloc(len);
  memcpy(buf, hostname, hostname_len);
  req->hostname = buf;
  // ... similar for service and hints ...

  // Submit to thread pool
  uv__work_submit(loop,
                  &req->work_req,
                  uv__getaddrinfo_work,
                  uv__getaddrinfo_done);

  return 0;
}
```

**Worker function** (executes on thread pool):

```c
static void uv__getaddrinfo_work(struct uv__work* w) {
  uv_getaddrinfo_t* req = container_of(w, uv_getaddrinfo_t, work_req);

  // This call BLOCKS
  req->retcode = getaddrinfo(req->hostname,
                             req->service,
                             req->hints,
                             &req->addrinfo);
}
```

**Completion** (back on event loop):

```c
static void uv__getaddrinfo_done(struct uv__work* w, int status) {
  uv_getaddrinfo_t* req = container_of(w, uv_getaddrinfo_t, work_req);

  if (status == UV_ECANCELED) {
    if (req->addrinfo)
      freeaddrinfo(req->addrinfo);
    req->addrinfo = NULL;
  }

  // Invoke user callback
  req->cb(req, status, req->addrinfo);
}
```

### The c-ares Alternative

For high-performance servers, libuv **optionally** supports c-ares, an async DNS resolver. Node.js can be compiled with c-ares support, which eliminates thread pool usage for DNS.

---

## Part VIII: Timers — Min-Heap Implementation

Timers in libuv are implemented as a **min-heap** keyed by expiration time, allowing O(log n) insertion and O(1) access to the next expiring timer.

**Source**: `src/heap-inl.h` and `src/timer.c`

```c
// Timer heap structure (in uv_loop_t)
struct heap {
  struct heap_node* min;    // Root of min-heap
  unsigned int nelts;       // Number of elements
};

struct heap_node {
  struct heap_node* left;
  struct heap_node* right;
  struct heap_node* parent;
};
```

**Timer handle structure**:

```c
struct uv_timer_s {
  UV_HANDLE_FIELDS
  uv_timer_cb timer_cb;
  void* heap_node[3];        // Heap structure
  uint64_t timeout;          // Expiration time (absolute)
  uint64_t repeat;           // Repeat interval (0 = one-shot)
  uint64_t start_id;         // For stable ordering
};
```

**Starting a timer**:

```c
int uv_timer_start(uv_timer_t* handle,
                   uv_timer_cb cb,
                   uint64_t timeout,
                   uint64_t repeat) {
  uint64_t clamped_timeout = handle->loop->time + timeout;

  // Stop existing timer if active
  if (uv__is_active(handle))
    uv_timer_stop(handle);

  handle->timer_cb = cb;
  handle->timeout = clamped_timeout;
  handle->repeat = repeat;
  handle->start_id = handle->loop->timer_counter++;

  // Insert into min-heap
  heap_insert((struct heap*)&handle->loop->timer_heap,
              (struct heap_node*)&handle->heap_node,
              timer_less_than);

  uv__handle_start(handle);

  return 0;
}
```

**Timer comparison function**:

```c
static int timer_less_than(const struct heap_node* ha,
                           const struct heap_node* hb) {
  const uv_timer_t* a = container_of(ha, uv_timer_t, heap_node);
  const uv_timer_t* b = container_of(hb, uv_timer_t, heap_node);

  if (a->timeout < b->timeout)
    return 1;
  if (b->timeout < a->timeout)
    return 0;

  // Break ties with start_id for stable ordering
  return a->start_id < b->start_id;
}
```

**Running expired timers** (in event loop):

```c
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min((struct heap*)&loop->timer_heap);
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);

    if (handle->timeout > loop->time)
      break;  // No more expired timers

    uv_timer_stop(handle);
    uv_timer_again(handle);  // Re-queue if repeat > 0

    handle->timer_cb(handle);  // Invoke callback
  }
}
```

This design ensures that **checking for expired timers is O(1)** (just peek at heap root) and **finding next timeout is O(1)** (needed for calculating `epoll_wait` timeout).

---

## Part IX: Signal Handling — Thread Safety Challenges

Unix signal handling is notoriously tricky in multi-threaded programs. Libuv uses a **self-pipe trick** to convert asynchronous signals into synchronous I/O events.

**Source**: `src/unix/signal.c`

```c
static int uv__signal_loop_once_init(uv_loop_t* loop) {
  int err;

  // Create pipe: signal handler writes to [1], event loop reads from [0]
  err = uv__make_pipe(loop->signal_pipefd, 0);
  if (err)
    return err;

  // Register read end with event loop
  uv__io_init(&loop->signal_io_watcher,
              uv__signal_event,
              loop->signal_pipefd[0]);
  uv__io_start(loop, &loop->signal_io_watcher, POLLIN);

  return 0;
}
```

**Signal handler** (async-signal-safe):

```c
static void uv__signal_handler(int signum) {
  uv__signal_msg_t msg;
  uv_loop_t* loop;
  int saved_errno = errno;

  // Retrieve loop from thread-local storage or global
  loop = uv__signal_first_handle(signum)->loop;

  msg.signum = signum;
  msg.handle = NULL;  // Will be resolved in event loop thread

  // Write signal number to pipe (async-signal-safe)
  do {
    ssize_t r = write(loop->signal_pipefd[1], &msg, sizeof(msg));
    if (r == sizeof(msg))
      break;
  } while (errno == EINTR);

  errno = saved_errno;
}
```

**Event loop handler**:

```c
static void uv__signal_event(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  uv__signal_msg_t msg;
  uv_signal_t* handle;
  ssize_t nread;

  while (1) {
    nread = read(w->fd, &msg, sizeof(msg));
    if (nread != sizeof(msg))
      break;

    // Find all handles registered for this signal
    for (handle = loop->signal_handles[msg.signum];
         handle;
         handle = handle->next_signal) {
      handle->signal_cb(handle, msg.signum);
    }
  }
}
```

This architecture ensures that **signal callbacks execute on the event loop thread**, not in the signal handler context, making them safe to perform arbitrary operations.

---

## Part X: Process Spawning — Fork/Exec and Pipes

Child process spawning involves coordinating:
1. **pipe creation** for stdin/stdout/stderr
2. **fork()** to create child
3. **exec()** to load new program
4. **I/O redirection** in the child
5. **Error propagation** from child to parent

**Source**: `src/unix/process.c`

```c
int uv_spawn(uv_loop_t* loop,
             uv_process_t* process,
             const uv_process_options_t* options) {
  int pipes[3][2];  // stdin, stdout, stderr
  int signal_pipe[2];
  pid_t pid;

  // Create pipes for stdio
  for (int i = 0; i < 3; i++) {
    if (options->stdio[i].flags & UV_CREATE_PIPE) {
      uv__make_pipe(pipes[i], 0);
    }
  }

  // Create pipe for error reporting from child
  uv__make_pipe(signal_pipe, 0);
  uv__cloexec(signal_pipe[0], 1);
  uv__cloexec(signal_pipe[1], 1);

  pid = fork();

  if (pid == 0) {
    // ===== CHILD PROCESS =====

    // Redirect stdio
    if (pipes[0][0] != -1) {
      dup2(pipes[0][0], 0);  // stdin
      uv__close(pipes[0][0]);
      uv__close(pipes[0][1]);
    }
    // ... similar for stdout/stderr ...

    // Change working directory
    if (options->cwd)
      chdir(options->cwd);

    // Set environment
    if (options->env)
      environ = options->env;

    // Execute new program
    execvp(options->file, options->args);

    // If exec failed, write errno to signal pipe
    int err = errno;
    do {
      ssize_t r = write(signal_pipe[1], &err, sizeof(err));
    } while (r == -1 && errno == EINTR);

    _exit(127);
  }

  // ===== PARENT PROCESS =====

  uv__close(signal_pipe[1]);

  // Check if exec failed in child
  int exec_errno;
  ssize_t r;
  do {
    r = read(signal_pipe[0], &exec_errno, sizeof(exec_errno));
  } while (r == -1 && errno == EINTR);

  uv__close(signal_pipe[0]);

  if (r > 0) {
    // Exec failed
    uv__close_nocheckstdio(pipes[0][0]);
    uv__close_nocheckstdio(pipes[0][1]);
    // ... cleanup ...
    return UV__ERR(exec_errno);
  }

  process->pid = pid;

  // Register child for exit monitoring
  uv__handle_start(process);
  loop->child_watcher.cb = uv__chld;  // SIGCHLD handler

  return 0;
}
```

**Child exit monitoring** uses `SIGCHLD`:

```c
static void uv__chld(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  uv_process_t* process;
  int status;
  pid_t pid;

  for (;;) {
    do {
      pid = waitpid(-1, &status, WNOHANG);
    } while (pid == -1 && errno == EINTR);

    if (pid <= 0)
      break;

    // Find process handle for this PID
    process = uv__find_process(loop, pid);
    if (process == NULL)
      continue;

    process->status = status;

    // Invoke exit callback
    if (process->exit_cb)
      process->exit_cb(process,
                       WEXITSTATUS(status),
                       WTERMSIG(status));
  }
}
```

---

## Part XI: Async Handles — Inter-Thread Communication

`uv_async_t` provides a mechanism for **threads to wake up the event loop** safely. This is critical for the thread pool completion mechanism.

**Implementation** varies by platform:

### Unix: Self-Pipe or Eventfd

**Source**: `src/unix/async.c`

```c
int uv_async_init(uv_loop_t* loop, uv_async_t* handle, uv_async_cb async_cb) {
  int err;

  err = uv__async_start(loop);
  if (err)
    return err;

  uv__handle_init(loop, (uv_handle_t*)handle, UV_ASYNC);
  handle->async_cb = async_cb;
  handle->pending = 0;

  QUEUE_INSERT_TAIL(&loop->async_handles, &handle->queue);
  uv__handle_start(handle);

  return 0;
}
```

**Loop initialization** creates a shared eventfd:

```c
static int uv__async_start(uv_loop_t* loop) {
  if (loop->async_wfd != -1)
    return 0;

#ifdef __linux__
  // Use eventfd on Linux (more efficient)
  int fd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
  loop->async_wfd = fd;
  loop->async_io_watcher.fd = fd;
#else
  // Fallback to pipe
  int pipefd[2];
  uv__make_pipe(pipefd, 0);
  loop->async_wfd = pipefd[1];
  loop->async_io_watcher.fd = pipefd[0];
#endif

  uv__io_init(&loop->async_io_watcher, uv__async_io, loop->async_io_watcher.fd);
  uv__io_start(loop, &loop->async_io_watcher, POLLIN);

  return 0;
}
```

**Sending from another thread**:

```c
int uv_async_send(uv_async_t* handle) {
  // Atomic flag set
  if (cmpxchgi(&handle->pending, 0, 1) != 0)
    return 0;  // Already pending

  // Wake event loop
  do {
    uint64_t val = 1;
    ssize_t r = write(handle->loop->async_wfd, &val, sizeof(val));
  } while (r == -1 && errno == EINTR);

  return 0;
}
```

**Event loop handler**:

```c
static void uv__async_io(uv_loop_t* loop, uv__io_t* w, unsigned int events) {
  QUEUE queue;
  QUEUE* q;
  uv_async_t* h;

  // Drain eventfd/pipe
  uint64_t val;
  read(w->fd, &val, sizeof(val));

  // Process all pending async handles
  QUEUE_MOVE(&loop->async_handles, &queue);
  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_async_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->async_handles, q);

    if (cmpxchgi(&h->pending, 1, 0) == 0)
      continue;  // Not actually pending

    h->async_cb(h);
  }
}
```

### Windows: PostQueuedCompletionStatus

On Windows, async uses IOCP directly:

```c
int uv_async_send(uv_async_t* handle) {
  if (InterlockedExchange(&handle->pending, 1) != 0)
    return 0;

  PostQueuedCompletionStatus(handle->loop->iocp, 0, 0, &handle->async_req.overlapped);
  return 0;
}
```

---

## Part XII: Memory Management and Reference Counting

Libuv uses **manual reference counting** for resource management, distinct from handles being "active."

```c
struct uv_handle_s {
  // ...
  unsigned int flags;  // Includes ref count bits
};

#define UV_HANDLE_CLOSING       0x00000001
#define UV_HANDLE_CLOSED        0x00000002
#define UV_HANDLE_ACTIVE        0x00000004
#define UV_HANDLE_REF           0x00000008  // Contributes to loop alive
```

**Reference semantics**:

```c
void uv_ref(uv_handle_t* handle) {
  uv__handle_ref(handle);
}

void uv_unref(uv_handle_t* handle) {
  uv__handle_unref(handle);
}

static int uv__loop_alive(const uv_loop_t* loop) {
  return uv__has_active_handles(loop) ||
         uv__has_active_reqs(loop) ||
         loop->closing_handles != NULL;
}
```

A handle only keeps the loop alive if it's both **active** and **referenced**. This allows, for example, timers that don't prevent process exit:

```c
uv_timer_t timer;
uv_timer_init(loop, &timer);
uv_timer_start(&timer, timer_cb, 1000, 1000);
uv_unref((uv_handle_t*)&timer);  // Won't keep loop alive
```

---

## Part XIII: Integration with Node.js

Node.js embeds libuv and V8, using libuv's event loop as the execution scheduler.

**Node.js event loop phases** map to libuv phases:

```
Node.js Phase           Libuv Phase
─────────────────────   ─────────────────────
timers                  uv__run_timers()
pending callbacks       uv__run_pending()
idle, prepare           uv__run_idle(), uv__run_prepare()
poll                    uv__io_poll()
check                   uv__run_check()
close callbacks         uv__run_closing_handles()
```

**Binding layer** (simplified from `src/node.cc`):

```cpp
void RunAtExit(Environment* env) {
  env->RunAtExitCallbacks();
}

int Start(int argc, char** argv) {
  // Initialize V8
  v8::V8::Initialize();

  // Create libuv loop
  uv_loop_t* loop = uv_default_loop();

  // Create Node environment with V8 isolate
  Isolate* isolate = Isolate::New(...);
  Environment* env = CreateEnvironment(isolate, loop, ...);

  // Load and execute user script
  LoadEnvironment(env);

  // Run event loop until no active handles
  do {
    uv_run(loop, UV_RUN_DEFAULT);

    // Process V8 microtask queue between loop iterations
    isolate->RunMicrotasks();

    // Check for unhandled promise rejections
    EmitPendingUnhandledRejections(env);

  } while (uv_loop_alive(loop));

  // Cleanup
  RunAtExit(env);
  FreeEnvironment(env);

  return 0;
}
```

**JavaScript → C++ → Libuv binding** (e.g., `fs.readFile`):

```cpp
// JavaScript calls fs.readFile(path, callback)
// ↓
// Bindings in lib/fs.js prepare request
// ↓
void FSReqWrap::ReadFile(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  // Extract arguments
  Utf8Value path(isolate, args[0]);
  Local<Function> callback = args[1].As<Function>();

  // Create libuv request
  uv_fs_t* req = &wrap->req_;
  req->data = wrap;

  // Submit to libuv
  int err = uv_fs_read(env->event_loop(),
                       req,
                       file,
                       &buf,
                       1,
                       offset,
                       AfterRead);  // C++ completion callback
}

// When libuv completes on thread pool:
void AfterRead(uv_fs_t* req) {
  FSReqWrap* wrap = static_cast<FSReqWrap*>(req->data);
  Environment* env = wrap->env();

  // Enter V8 context
  HandleScope scope(env->isolate());
  Context::Scope context_scope(env->context());

  // Prepare JavaScript arguments
  Local<Value> argv[] = {
    Number::New(isolate, req->result),  // Bytes read or error
    Buffer::New(isolate, buf.base, req->result).ToLocalChecked()
  };

  // Invoke JavaScript callback
  wrap->MakeCallback(env->ondone_string(), 2, argv);

  // Cleanup
  uv_fs_req_cleanup(req);
}
```

**Microtask interleaving** — Node.js processes Promise microtasks between I/O phases:

```cpp
bool RunNextTicks(Environment* env) {
  if (!env->can_call_into_js())
    return true;

  Local<Function> callback = env->tick_callback_function();

  return !callback.IsEmpty() &&
         callback->Call(env->context(), Undefined(isolate), 0, nullptr)
                ->IsTrue();
}

// Called after each libuv phase
void ProcessImmediate(Environment* env) {
  HandleScope scope(env->isolate());

  // Process nextTick queue
  RunNextTicks(env);

  // Process setImmediate queue (maps to uv_check_t)
  if (env->immediate_info()->count() > 0) {
    env->RunAndClearImmediates();
  }
}
```

---

## Part XIV: Platform-Specific Optimizations

### Linux: io_uring Integration (Experimental)

Modern Linux kernels (5.1+) provide `io_uring`, a true async I/O interface. Libuv has experimental support:

```c
// Hypothetical io_uring backend structure
struct uv__io_uring {
  int ring_fd;
  struct io_uring_params params;
  unsigned* sq_head;     // Submission queue head
  unsigned* sq_tail;     // Submission queue tail
  unsigned* sq_mask;
  unsigned* sq_entries;
  struct io_uring_sqe* sqes;  // Submission queue entries

  unsigned* cq_head;     // Completion queue head
  unsigned* cq_tail;
  unsigned* cq_mask;
  struct io_uring_cqe* cqes;  // Completion queue entries
};
```

**Submission** (preparing I/O operations):

```c
static void uv__io_uring_submit(uv_loop_t* loop, uv_fs_t* req) {
  struct io_uring_sqe* sqe;
  unsigned tail;

  // Get next submission queue entry
  tail = *loop->io_uring.sq_tail;
  sqe = &loop->io_uring.sqes[tail & *loop->io_uring.sq_mask];

  // Prepare read operation
  sqe->opcode = IORING_OP_READ;
  sqe->fd = req->file;
  sqe->addr = (unsigned long)req->bufs[0].base;
  sqe->len = req->bufs[0].len;
  sqe->off = req->off;
  sqe->user_data = (uintptr_t)req;  // Store request pointer

  // Advance tail
  *loop->io_uring.sq_tail = tail + 1;

  // Submit to kernel
  io_uring_enter(loop->io_uring.ring_fd, 1, 0, 0);
}
```

**Completion harvesting**:

```c
static void uv__io_uring_poll(uv_loop_t* loop) {
  struct io_uring_cqe* cqe;
  uv_fs_t* req;
  unsigned head;

  head = *loop->io_uring.cq_head;

  while (head != *loop->io_uring.cq_tail) {
    cqe = &loop->io_uring.cqes[head & *loop->io_uring.cq_mask];

    // Retrieve request from user_data
    req = (uv_fs_t*)cqe->user_data;
    req->result = cqe->res;  // Result or error code

    // Invoke completion callback
    req->cb(req);

    head++;
  }

  *loop->io_uring.cq_head = head;
}
```

This eliminates thread pool overhead for file I/O, providing true kernel-level async file operations.

### macOS: kqueue Optimizations

macOS's `kqueue` supports multiple filter types beyond just file descriptors:

```c
// Timer using kqueue directly (alternative to timer heap)
struct kevent kev;
EV_SET(&kev,
       timer_id,              // ident
       EVFILT_TIMER,          // filter type
       EV_ADD | EV_ENABLE,    // flags
       NOTE_USECONDS,         // fflags (microsecond precision)
       timeout_us,            // data (timeout)
       (void*)handle);        // udata (user pointer)

kevent(loop->backend_fd, &kev, 1, NULL, 0, NULL);
```

**VNODE monitoring** for filesystem events:

```c
struct kevent kev;
EV_SET(&kev,
       fd,                    // File descriptor
       EVFILT_VNODE,          // Monitor vnode changes
       EV_ADD | EV_CLEAR,
       NOTE_WRITE | NOTE_DELETE | NOTE_EXTEND | NOTE_ATTRIB,
       0,
       (void*)handle);

kevent(loop->backend_fd, &kev, 1, NULL, 0, NULL);
```

### Windows: IOCP Advanced Patterns

**Overlapped I/O structure** embedding:

```c
typedef struct {
  OVERLAPPED overlapped;      // Must be first member
  uv_buf_t buf;
  WSABUF wsabuf;              // Windows scatter-gather buffer
  uv_stream_t* handle;
  uv_write_cb cb;
} uv_write_t;

// When submitting async write:
DWORD bytes;
int result = WSASend(
  socket,
  &req->wsabuf,               // Buffer descriptor
  1,                          // Buffer count
  &bytes,                     // Bytes sent (ignored in async)
  0,                          // Flags
  &req->overlapped,           // Overlapped structure
  NULL);                      // Completion routine (use IOCP instead)

// IOCP will later return &req->overlapped
// We recover full structure via CONTAINING_RECORD
```

**Cancellation** (for timeouts):

```c
// Cancel all pending I/O on a handle
CancelIoEx((HANDLE)socket, NULL);

// Or cancel specific operation:
CancelIoEx((HANDLE)socket, &req->overlapped);
```

---

## Part XV: Error Handling Philosophy

Libuv uses **negative error codes** (Unix errno-style) uniformly across platforms:

```c
// Error code mapping (from uv-errno.h)
#define UV__EOF     (-4095)
#define UV__UNKNOWN (-4094)
#define UV__EAI_ADDRFAMILY  (-3000)
#define UV__EAI_AGAIN       (-3001)
// ... etc

// Unix-specific errors
#define UV__EAGAIN  (-EAGAIN)
#define UV__EINTR   (-EINTR)
// Windows errors get mapped to Unix equivalents
```

**Error translation** on Windows:

```c
int uv_translate_sys_error(int sys_errno) {
  if (sys_errno <= 0)
    return sys_errno;  // Already a UV error

  switch (sys_errno) {
    case ERROR_FILE_NOT_FOUND:
    case ERROR_PATH_NOT_FOUND:
      return UV_ENOENT;

    case ERROR_ACCESS_DENIED:
    case ERROR_SHARING_VIOLATION:
      return UV_EACCES;

    case ERROR_INVALID_HANDLE:
      return UV_EBADF;

    // ... hundreds more mappings ...

    default:
      return UV_UNKNOWN;
  }
}
```

**Error strings**:

```c
const char* uv_strerror(int err) {
  return uv__strerror_r(err);  // Thread-safe strerror
}

const char* uv_err_name(int err) {
  switch (err) {
    case UV_EAGAIN: return "EAGAIN";
    case UV_EACCES: return "EACCES";
    // ... etc
  }
}
```

---

## Part XVI: Handle Lifecycle Management

Every handle follows a strict lifecycle:

```
┌──────────────┐
│     INIT     │  uv_xxx_init()
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   INACTIVE   │  Handle exists but not active
└──────┬───────┘
       │  uv_xxx_start() or similar
       ▼
┌──────────────┐
│    ACTIVE    │  Handle is registered with event loop
└──────┬───────┘
       │  uv_xxx_stop()
       ▼
┌──────────────┐
│   INACTIVE   │  Can be restarted
└──────┬───────┘
       │  uv_close()
       ▼
┌──────────────┐
│   CLOSING    │  Waiting for close callback
└──────┬───────┘
       │  After close_cb invoked
       ▼
┌──────────────┐
│    CLOSED    │  Memory can be freed
└──────────────┘
```

**Closing mechanism**:

```c
void uv_close(uv_handle_t* handle, uv_close_cb close_cb) {
  assert(!uv__is_closing(handle));

  handle->flags |= UV_HANDLE_CLOSING;
  handle->close_cb = close_cb;

  switch (handle->type) {
    case UV_TCP:
    case UV_UDP:
      // Stop I/O watcher
      uv__io_close(handle->loop, &((uv_stream_t*)handle)->io_watcher);
      break;

    case UV_TIMER:
      uv_timer_stop((uv_timer_t*)handle);
      break;

    // ... handle-specific cleanup ...
  }

  // Add to closing queue (processed in uv__run_closing_handles)
  uv__make_close_pending(handle);
}
```

**Deferred close callbacks**:

```c
void uv__run_closing_handles(uv_loop_t* loop) {
  uv_handle_t* handle;
  uv_handle_t* next;

  handle = loop->closing_handles;
  loop->closing_handles = NULL;

  while (handle) {
    next = handle->next_closing;

    // Platform-specific resource cleanup
    uv__handle_platform_close(handle);

    // Mark fully closed
    handle->flags |= UV_HANDLE_CLOSED;

    // Invoke user's close callback
    if (handle->close_cb) {
      handle->close_cb(handle);
    }

    handle = next;
  }
}
```

This two-phase close ensures:
1. **Resources released immediately** (sockets closed, etc.)
2. **Callbacks deferred** until it's safe (after current I/O phase)
3. **Memory still valid** when close callback runs

---

## Part XVII: Performance Characteristics

### Scalability Limits

| Resource | Linux (epoll) | macOS (kqueue) | Windows (IOCP) |
|----------|--------------|----------------|----------------|
| Max FDs | ~1M (ulimit) | ~64K | ~64K handles |
| Add/Remove | O(1) | O(log N) | O(1) |
| Wait | O(N_ready) | O(N_ready) | O(N_ready) |
| Memory | ~140 bytes/fd | ~200 bytes/fd | ~200 bytes/handle |

### Benchmarking Internal Operations

**Timer insertion** (1M timers):

```c
// Theoretical benchmark
uint64_t start = uv_hrtime();

for (int i = 0; i < 1000000; i++) {
  uv_timer_t timer;
  uv_timer_init(loop, &timer);
  uv_timer_start(&timer, NULL, random_timeout(), 0);
}

uint64_t elapsed = uv_hrtime() - start;
// Expected: ~50-100ns per insertion (heap insert is O(log N))
```

**I/O poll overhead** (empty loop):

```c
// No active handles, just measuring loop overhead
for (int i = 0; i < 1000000; i++) {
  uv_run(loop, UV_RUN_NOWAIT);
}
// Expected: ~200-500ns per iteration (syscall overhead)
```

**Thread pool saturation**:

```
Thread Pool Size | Concurrent Operations | Latency
-----------------|----------------------|----------
4 (default)      | 4                    | ~Twork
4                | 8                    | ~2×Twork (queuing)
4                | 100                  | ~25×Twork (severe queuing)
32               | 100                  | ~3×Twork (better, but more overhead)
```

---

## Part XVIII: Common Pitfalls and Anti-Patterns

### 1. **Blocking the Event Loop**

```c
// BAD: Synchronous sleep blocks entire event loop
uv_timer_start(&timer, timer_cb, 1000, 0);

void timer_cb(uv_timer_t* timer) {
  sleep(5);  // ❌ BLOCKS ALL I/O FOR 5 SECONDS
  printf("Timer fired\n");
}

// GOOD: Use another timer
void timer_cb(uv_timer_t* timer) {
  uv_timer_t* delay_timer = malloc(sizeof(*delay_timer));
  uv_timer_init(timer->loop, delay_timer);
  uv_timer_start(delay_timer, final_cb, 5000, 0);
}
```

### 2. **Use After Close**

```c
// BAD: Accessing handle after close
uv_close((uv_handle_t*)&timer, close_cb);
uv_timer_again(&timer);  // ❌ UNDEFINED BEHAVIOR

// GOOD: Only access in close callback
void close_cb(uv_handle_t* handle) {
  free(handle);  // ✓ Safe to free now
}
```

### 3. **Thread Pool Starvation**

```c
// BAD: Submitting too much blocking work
for (int i = 0; i < 1000; i++) {
  uv_fs_read(...);  // All serialize on 4-thread pool
}

// BETTER: Increase thread pool or batch operations
setenv("UV_THREADPOOL_SIZE", "16", 1);
```

### 4. **Memory Leaks in Callbacks**

```c
// BAD: Forgetting to cleanup request
void after_write(uv_write_t* req, int status) {
  // ❌ Memory leak: req is never freed
}

// GOOD: Always cleanup
void after_write(uv_write_t* req, int status) {
  free(req->bufs[0].base);  // Free buffer
  free(req);                 // Free request
}
```

### 5. **Incorrect Error Handling**

```c
// BAD: Ignoring errors
int r = uv_listen((uv_stream_t*)&server, 128, on_connection);
// ❌ What if bind failed?

// GOOD: Check every return value
int r = uv_listen((uv_stream_t*)&server, 128, on_connection);
if (r) {
  fprintf(stderr, "Listen error: %s\n", uv_strerror(r));
  return;
}
```

---

## Part XIX: Debugging Techniques

### 1. **Handle Leak Detection**

Libuv provides debugging hooks:

```c
void uv_walk(uv_loop_t* loop, uv_walk_cb walk_cb, void* arg) {
  uv_handle_t* handle;

  // Iterate all active handles
  QUEUE_FOREACH(q, &loop->handle_queue) {
    handle = QUEUE_DATA(q, uv_handle_t, handle_queue);
    walk_cb(handle, arg);
  }
}

// Usage:
void print_handle(uv_handle_t* handle, void* arg) {
  printf("Active handle: type=%d, ref=%d\n",
         handle->type,
         uv_has_ref(handle));
}

uv_walk(loop, print_handle, NULL);
```

### 2. **Event Loop Stall Detection**

```c
// Measure time between loop iterations
static uint64_t last_time = 0;

void prepare_cb(uv_prepare_t* handle) {
  uint64_t now = uv_hrtime();

  if (last_time > 0) {
    uint64_t delta = now - last_time;
    if (delta > 100000000) {  // 100ms
      fprintf(stderr, "Loop stalled for %llu ms\n", delta / 1000000);
    }
  }

  last_time = now;
}

uv_prepare_init(loop, &prepare);
uv_prepare_start(&prepare, prepare_cb);
```

### 3. **Stack Trace on Crash**

```c
#include <execinfo.h>
#include <signal.h>

void signal_handler(int sig) {
  void* array[10];
  size_t size = backtrace(array, 10);

  fprintf(stderr, "Error: signal %d:\n", sig);
  backtrace_symbols_fd(array, size, STDERR_FILENO);
  exit(1);
}

// In main():
signal(SIGSEGV, signal_handler);
signal(SIGABRT, signal_handler);
```

---

## Part XX: Future Directions

### 1. **io_uring Full Integration**

Complete replacement of thread pool for file I/O on Linux:
- Zero-copy operations
- Kernel-side buffering
- Reduced context switches

### 2. **WebAssembly Support**

Libuv compiled to WASM for in-browser use:
- Emscripten compatibility
- Browser API mapping (fetch → network I/O)

### 3. **Better Async Local Storage**

Thread-local storage for async contexts:

```c
uv_key_t context_key;

void uv_async_context_enter(void* context) {
  uv_key_set(&context_key, context);
}

void* uv_async_context_current() {
  return uv_key_get(&context_key);
}
```

### 4. **Structured Concurrency**

Automatic child task cancellation:

```c
uv_task_group_t group;
uv_task_group_init(&group);

uv_task_t* task1 = uv_task_spawn(&group, work_fn1);
uv_task_t* task2 = uv_task_spawn(&group, work_fn2);

// When group goes out of scope or cancelled, all tasks terminate
uv_task_group_cancel(&group);
```

---

## Conclusion: The Invisible Runtime

Libuv is the **substrate** upon which modern asynchronous I/O is built. It reconciles:

- **Unix readiness** with **Windows completion**
- **Single-threaded execution** with **parallel I/O**
- **Low-level syscalls** with **high-level event abstractions**

Its design reflects decades of OS evolution:
- The shift from `select()` (O(N)) → `epoll` (O(1))
- The recognition that **not all I/O can be async** (thread pool necessity)
- The need for **portable abstractions** across radically different platforms

When you write `const server = http.createServer()` in Node.js, you're invoking a **40,000-line C codebase** that coordinates:
- Socket multiplexing via epoll/kqueue/IOCP
- Timer wheels for timeouts
- Thread pools for DNS/file I/O
- Signal handling via self-pipes
- Process spawning with fork/exec orchestration

Libuv doesn't abstract away complexity — it **masters it**, exposing a coherent API while handling every edge case, race condition, and platform quirk internally.

**It is the hidden engine of modern network services.**

Every HTTP request, WebSocket message, database query, and file upload in the Node.js ecosystem flows through libuv's event loop — a testament to the power of **disciplined systems programming** applied to the hardest problem in concurrent I/O: making it **simple**.
