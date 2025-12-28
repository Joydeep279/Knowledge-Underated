# The Event Loop in Node.js: A Deep Dive into libuv's Runtime Architecture

## Part I: Foundational Context — The Runtime Substrate

### What is libuv?

`libuv` is a **multi-platform asynchronous I/O library** originally written for Node.js, but now used by many other projects (Neovim, Julia, Luvit, etc.). It abstracts platform-specific system calls and provides a unified event-driven model across Windows, Linux, macOS, and other Unix-like systems.

**Key Responsibility:**  
libuv provides the **kernel-level I/O abstraction** and the **event loop scheduler** that orchestrates all asynchronous operations in Node.js.

**Location in Node.js source:**
```
deps/uv/          ← libuv library
src/node.cc       ← Node.js initialization, binds JS to libuv
src/env.cc        ← Environment setup
, event loop attachment
```

When you execute `node app.js`, here's what happens at the systems level:

1. **Process Creation**: The OS spawns a process with a single main thread
2. **V8 Initialization**: JavaScript engine spins up heap and execution context
3. **libuv Initialization**: Event loop structure is allocated and initialized
4. **Binding Layer**: Node.js C++ bindings connect JS APIs to libuv primitives
5. **Script Execution**: Your code runs, registering callbacks with the event loop
6. **Loop Entry**: Control transfers to libuv's event loop dispatcher

---

## Part II: The Event Loop Architecture — Internal Structure

### Core Data Structure: `uv_loop_t`

The event loop is represented by the `uv_loop_t` structure. Let's examine its internal layout:

```c
// From deps/uv/include/uv.h and src/unix/internal.h

struct uv_loop_s {
  /* User data - for arbitrary use */
  void* data;

  /* Loop reference counting */
  unsigned int active_handles;
  void* handle_queue[2];           // Intrusive linked list of all handles

  /* Callback queues for different phases */
  void* pending_queue[2];          // I/O callbacks pending execution
  void* idle_handles[2];           // Idle handle queue
  void* prepare_handles[2];        // Prepare handle queue
  void* check_handles[2];          // Check handle queue
  void* closing_handles;           // Handles being closed

  /* Platform-specific I/O polling mechanism */
  void* watcher_queue[2];
  uv__io_t** watchers;             // Array of I/O watchers
  unsigned int nwatchers;          // Number of watchers
  unsigned int nfds;               // Number of file descriptors

  /* Timer heap - binary min-heap for timer management */
  void* timer_heap[3];             // Min-heap root + size + comparison fn
  uint64_t timer_counter;          // Monotonic timer ID counter

  /* System time tracking */
  uint64_t time;                   // Cached loop time (milliseconds)

  /* Thread pool for file I/O and CPU-intensive tasks */
  void* wq[2];                     // Work queue for thread pool
  uv_mutex_t wq_mutex;             // Mutex protecting work queue
  uv_async_t wq_async;             // Async handle for work completion

  /* Platform-specific polling backend */
  int backend_fd;                  // epoll_fd (Linux), kqueue_fd (macOS)
  void* signal_pipefd[2];          // Signal handling pipe

  /* Loop lifecycle flags */
  unsigned int stop_flag;
  unsigned int loop_iteration;     // Current iteration counter
};
```

**Memory Layout Insight:**  
This structure is allocated on the heap during `uv_loop_init()`. On a 64-bit system, it's typically around **1-2 KB** in size. Node.js maintains a **single** `uv_loop_t` instance per isolate (per thread in worker_threads).

---

## Part III: The Six Phases of Execution — Detailed Walkthrough

The event loop executes in **phases**, each handling a specific category of callbacks. Understanding these phases is critical to reasoning about execution order.

### Phase Diagram:

```
   ┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout/setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks deferred from previous iteration
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← Internal only (used by Node.js internals)
│  └─────────────┬─────────────┘      
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← Retrieve new I/O events; execute I/O callbacks
│  └─────────────┬─────────────┘      (blocks here if no timers scheduled)
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate() callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │      close callbacks      │  ← socket.on('close', ...)
│  └───────────────────────────┘
└──────────── Loop iteration ────────
```

### Core Loop Implementation

Let's examine the actual event loop dispatcher from `deps/uv/src/unix/core.c`:

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  // Check if there's any work to do
  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);  // Update cached time even if no work

  // Main event loop
  while (r != 0 && loop->stop_flag == 0) {
    // Update the loop's cached time (system clock)
    uv__update_time(loop);

    // PHASE 1: Run timers whose threshold has elapsed
    uv__run_timers(loop);

    // PHASE 2: Run pending callbacks from previous iteration
    ran_pending = uv__run_pending(loop);

    // PHASE 3: Run idle and prepare handles (internal)
    uv__run_idle(loop);
    uv__run_prepare(loop);

    // Calculate poll timeout
    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    // PHASE 4: Poll for I/O events (BLOCKS HERE if no timers)
    uv__io_poll(loop, timeout);

    // PHASE 5: Run check handles (setImmediate)
    uv__run_check(loop);

    // PHASE 6: Run close callbacks
    uv__run_closing_handles(loop);

    // Check if we're in UV_RUN_ONCE or UV_RUN_NOWAIT mode
    if (mode == UV_RUN_ONCE) {
      uv__update_time(loop);
      uv__run_timers(loop);  // Run timers again for UV_RUN_ONCE
    }

    // Recalculate whether loop should continue
    r = uv__loop_alive(loop);

    // For UV_RUN_ONCE or UV_RUN_NOWAIT, break after one iteration
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  // Loop has stopped - reset stop flag
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  // Return non-zero if there are still active handles/requests
  return r;
}
```

**Critical Understanding:**  
The `while` loop is the **heartbeat** of your Node.js application. Each iteration is one "tick" of the event loop.

---

## Part IV: Phase-by-Phase Deep Dive

### Phase 1: Timers — Binary Min-Heap Scheduling

**Source Location:** `deps/uv/src/timer.c`

Timers in libuv are managed using a **binary min-heap** data structure, where the timer with the smallest timeout is always at the root.

```c
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    // Peek at the root of the min-heap (earliest timer)
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    // Cast the heap node to a timer handle
    handle = container_of(heap_node, uv_timer_t, heap_node);

    // Check if this timer has expired
    if (handle->timeout > loop->time)
      break;  // No more timers ready, exit

    // Remove timer from heap
    uv_timer_stop(handle);
    uv_timer_again(handle);  // Re-insert if repeat > 0

    // Execute the callback
    handle->timer_cb(handle);
  }
}
```

**How Timers Work:**

1. When you call `setTimeout(fn, 100)`, Node.js creates a `uv_timer_t` handle
2. The timer is inserted into the min-heap with `timeout = loop->time + 100ms`
3. During each loop iteration, `uv__run_timers()` checks if `heap_min->timeout <= loop->time`
4. If true, the timer callback is executed and removed from the heap

**Why Min-Heap?**  
- **O(1)** access to the next timer to fire
- **O(log n)** insertion and removal
- Memory efficient: timers are stored contiguously

**Precision Caveat:**  
Timers are **not precise**. If the poll phase blocks for 50ms and your timer had 10ms remaining, it fires late. This is why Node.js timers are called "thresholds" not "guarantees".

---

### Phase 2: Pending Callbacks — Deferred I/O Completion

**Source Location:** `deps/uv/src/unix/core.c`

Some I/O callbacks are deferred to the next iteration to prevent **stack overflow** or to maintain phase ordering.

```c
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  // Move pending_queue to local queue
  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) {
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT);  // Execute the pending callback
  }

  return !QUEUE_EMPTY(&loop->pending_queue);
}
```

**When are callbacks deferred?**

1. **TCP connection errors**: If `connect()` fails immediately (e.g., ECONNREFUSED)
2. **Some write callbacks**: After kernel buffer flush
3. **UDP send errors**: When `sendto()` fails synchronously

**Why defer?**  
Maintains **consistent callback ordering** and prevents edge cases where synchronous errors would execute callbacks before the function even returns.

---

### Phase 3: Idle and Prepare — Internal Hooks

**Source Location:** `deps/uv/src/unix/loop-watcher.c`

These are **internal-only** phases used by Node.js for bookkeeping:

```c
void uv__run_idle(uv_loop_t* loop) {
  uv_idle_t* h;
  QUEUE queue;
  QUEUE* q;

  QUEUE_MOVE(&loop->idle_handles, &queue);

  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_idle_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->idle_handles, q);
    h->idle_cb(h);  // Execute idle callback
  }
}

void uv__run_prepare(uv_loop_t* loop) {
  uv_prepare_t* h;
  QUEUE queue;
  QUEUE* q;

  QUEUE_MOVE(&loop->prepare_handles, &queue);

  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_prepare_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->prepare_handles, q);
    h->prepare_cb(h);  // Execute prepare callback
  }
}
```

**Node.js Internal Usage:**

- **Prepare phase**: Used by Node.js for performance measurements (`perf_hooks`)
- **Idle phase**: Rarely used, but available for internal optimizations

**Userland access:**  
Not exposed to JavaScript. You cannot register idle/prepare callbacks from JS code.

---

### Phase 4: Poll — The I/O Epicenter

**This is the most complex and critical phase.** It's where the event loop interacts with the **kernel** to wait for I/O events.

**Platform-Specific Implementations:**

- **Linux**: `epoll()` — edge-triggered notification system
- **macOS/BSD**: `kqueue()` — kernel event notification
- **Windows**: `IOCP` (I/O Completion Ports) — proactive I/O model

Let's examine the Linux implementation (`deps/uv/src/unix/linux-core.c`):

```c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  struct epoll_event events[1024];
  struct epoll_event* pe;
  struct epoll_event e;
  int real_timeout;
  QUEUE* q;
  uv__io_t* w;
  sigset_t sigset;
  uint64_t sigmask;
  uint64_t base;
  int have_signals;
  int nevents;
  int count;
  int nfds;
  int fd;
  int op;
  int i;

  // If there's no I/O watchers, don't poll
  if (loop->nfds == 0) {
    assert(QUEUE_EMPTY(&loop->watcher_queue));
    return;
  }

  // Process watcher queue - register new I/O watchers with epoll
  while (!QUEUE_EMPTY(&loop->watcher_queue)) {
    q = QUEUE_HEAD(&loop->watcher_queue);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);

    w = QUEUE_DATA(q, uv__io_t, watcher_queue);
    assert(w->pevents != 0);
    assert(w->fd >= 0);
    assert(w->fd < (int) loop->nwatchers);

    e.events = w->pevents;
    e.data.fd = w->fd;

    // Determine epoll operation: ADD, MOD, or DEL
    if (w->events == 0)
      op = EPOLL_CTL_ADD;
    else
      op = EPOLL_CTL_MOD;

    // Register with epoll
    if (epoll_ctl(loop->backend_fd, op, w->fd, &e)) {
      if (errno != EEXIST)
        abort();  // Fatal error

      // If EEXIST, modify instead
      if (epoll_ctl(loop->backend_fd, EPOLL_CTL_MOD, w->fd, &e))
        abort();
    }

    w->events = w->pevents;
  }

  // Calculate real timeout (considers timers)
  real_timeout = timeout;

  // BLOCKING CALL - Wait for I/O events
  base = loop->time;
  count = 48; // Hardcoded batch size for signal checking
  nfds = epoll_wait(loop->backend_fd,
                    events,
                    ARRAY_SIZE(events),
                    real_timeout);

  // Update loop time after blocking
  uv__update_time(loop);

  // Process returned events
  for (i = 0; i < nfds; i++) {
    pe = events + i;
    fd = pe->data.fd;

    // Special handling for signal pipe
    if (fd == -1) {
      uv__signal_event(loop, pe->events);
      continue;
    }

    // Get the I/O watcher for this file descriptor
    assert(fd >= 0);
    assert((unsigned) fd < loop->nwatchers);

    w = loop->watchers[fd];

    if (w == NULL) {
      // Watcher was closed - ignore event
      epoll_ctl(loop->backend_fd, EPOLL_CTL_DEL, fd, pe);
      continue;
    }

    // Invoke the I/O callback
    w->cb(loop, w, pe->events);
  }
}
```

**Critical Behaviors:**

1. **Blocking Nature**: `epoll_wait()` blocks the main thread until:
   - An I/O event occurs
   - The timeout expires
   - A signal arrives

2. **Timeout Calculation**:
```c
int uv_backend_timeout(const uv_loop_t* loop) {
  if (loop->stop_flag != 0)
    return 0;

  if (!uv__has_active_handles(loop) && !uv__has_active_reqs(loop))
    return 0;

  if (!QUEUE_EMPTY(&loop->idle_handles))
    return 0;

  if (!QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  if (loop->closing_handles)
    return 0;

  return uv__next_timeout(loop);  // Returns ms until next timer
}
```

**Why This Matters:**  
- If you have a timer in 50ms, `epoll_wait()` blocks for **at most** 50ms
- If no timers are scheduled, it blocks **indefinitely** until I/O
- If `setImmediate()` is pending, timeout is **0** (non-blocking poll)

3. **File Descriptor Watcher Management**:

Each socket/file gets a `uv__io_t` structure:

```c
struct uv__io_s {
  uv__io_cb cb;          // Callback function
  void* pending_queue[2]; // Linked list for pending queue
  void* watcher_queue[2]; // Linked list for watcher queue
  unsigned int pevents;   // Pending events (POLLIN, POLLOUT)
  unsigned int events;    // Current events registered with epoll
  int fd;                 // File descriptor
};
```

When you create a TCP server:

```javascript
const server = net.createServer();
server.listen(3000);
```

Under the hood:
1. `socket()` syscall creates a file descriptor
2. `bind()` and `listen()` configure the socket
3. A `uv__io_t` watcher is created with `fd = socket_fd`
4. The watcher is registered with `epoll()` for `EPOLLIN` events
5. When a connection arrives, `epoll_wait()` returns, and the callback executes

---

### Phase 5: Check — setImmediate Execution

**Source Location:** `deps/uv/src/unix/loop-watcher.c`

```c
void uv__run_check(uv_loop_t* loop) {
  uv_check_t* h;
  QUEUE queue;
  QUEUE* q;

  QUEUE_MOVE(&loop->check_handles, &queue);

  while (!QUEUE_EMPTY(&queue)) {
    q = QUEUE_HEAD(&queue);
    h = QUEUE_DATA(q, uv_check_t, queue);
    QUEUE_REMOVE(q);
    QUEUE_INSERT_TAIL(&loop->check_handles, q);
    h->check_cb(h);  // Execute check callback
  }
}
```

**Why does `setImmediate()` exist?**

Consider this code:

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

Output order is **non-deterministic** if called from the main module, but **deterministic** if called from within an I/O callback:

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});

// Output is ALWAYS:
// immediate
// timeout
```

**Why?**  
- When inside an I/O callback (poll phase), we're **between** poll and check
- Check phase runs **before** the next timer phase
- `setImmediate()` is guaranteed to run before any timers

**setImmediate vs process.nextTick:**

`process.nextTick()` is **NOT** part of the event loop phases. It's a microtask queue that runs **after the current operation completes**, before transitioning to the next phase:

```
current phase operation
    ↓
process.nextTick queue  ← Runs here
    ↓
Promise microtasks      ← Then this
    ↓
next event loop phase
```

---

### Phase 6: Close Callbacks

**Source Location:** `deps/uv/src/unix/core.c`

```c
static void uv__run_closing_handles(uv_loop_t* loop) {
  uv_handle_t* p;
  uv_handle_t* q;

  p = loop->closing_handles;
  loop->closing_handles = NULL;

  while (p) {
    q = p->next_closing;
    uv__finish_close(p);
    p = q;
  }
}

static void uv__finish_close(uv_handle_t* handle) {
  assert(!uv__is_active(handle));
  assert(handle->flags & UV_HANDLE_CLOSING);
  assert(!(handle->flags & UV_HANDLE_CLOSED));

  handle->flags |= UV_HANDLE_CLOSED;

  // Call the close callback if provided
  if (handle->close_cb) {
    handle->close_cb(handle);
  }

  // Decrease active handle count
  uv__handle_unref(handle);
}
```

**When Close Callbacks Run:**

```javascript
const server = net.createServer();
server.listen(3000);

server.close(() => {
  console.log('Server closed');  // Runs in close phase
});
```

**Two-Step Closing Process:**

1. **Marking Phase**: `uv_close()` marks the handle as `UV_HANDLE_CLOSING`
2. **Execution Phase**: Next loop iteration executes `close_cb` and marks as `UV_HANDLE_CLOSED`

This ensures all pending I/O completes before the handle is fully destroyed.

---

## Part V: The Thread Pool — Offloading Blocking Operations

Not all operations can be made asynchronous at the kernel level. For these, libuv uses a **thread pool**.

**Thread Pool Operations:**
- File system I/O (all `fs.*` except `fs.watch`)
- DNS lookups (`dns.resolve`, `dns.lookup`)
- CPU-intensive crypto operations
- `zlib` compression
- User-defined tasks via `uv_queue_work()`

**Source Location:** `deps/uv/src/threadpool.c`

```c
// Thread pool configuration
static uv_thread_t default_threads[4];  // Default: 4 threads
static uv_once_t once = UV_ONCE_INIT;
static uv_cond_t cond;
static uv_mutex_t mutex;
static unsigned int idle_threads;
static unsigned int nthreads;
static QUEUE exit_message;
static QUEUE wq;  // Work queue
static volatile int initialized;

static void init_threads(void) {
  unsigned int i;
  const char* val;
  uv_sem_t sem;

  nthreads = ARRAY_SIZE(default_threads);

  // Allow environment variable override
  val = getenv("UV_THREADPOOL_SIZE");
  if (val != NULL)
    nthreads = atoi(val);

  // Clamp to [1, 1024]
  if (nthreads == 0)
    nthreads = 1;
  if (nthreads > MAX_THREADPOOL_SIZE)
    nthreads = MAX_THREADPOOL_SIZE;

  // Initialize synchronization primitives
  if (uv_cond_init(&cond))
    abort();
  if (uv_mutex_init(&mutex))
    abort();

  QUEUE_INIT(&wq);
  QUEUE_INIT(&exit_message);

  // Spawn worker threads
  for (i = 0; i < nthreads; i++)
    if (uv_thread_create(default_threads + i, worker, &sem))
      abort();

  initialized = 1;
}

static void worker(void* arg) {
  struct uv__work* w;
  QUEUE* q;

  // Semaphore used for initialization synchronization
  uv_sem_post((uv_sem_t*) arg);
  arg = NULL;

  uv_mutex_lock(&mutex);
  for (;;) {
    // Wait for work or exit message
    while (QUEUE_EMPTY(&wq))
      uv_cond_wait(&cond, &mutex);

    q = QUEUE_HEAD(&wq);

    // Check for exit message
    if (q == &exit_message) {
      uv_cond_signal(&cond);  // Wake another thread
      uv_mutex_unlock(&mutex);
      break;
    }

    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    uv_mutex_unlock(&mutex);

    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);  // Execute the work function (BLOCKING)

    // Mark work as done and signal via async handle
    uv_mutex_lock(&w->loop->wq_mutex);
    w->work = NULL;
    QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
    uv_async_send(&w->loop->wq_async);  // Wake event loop
    uv_mutex_unlock(&w->loop->wq_mutex);

    uv_mutex_lock(&mutex);
  }
}
```

**How Thread Pool Integration Works:**

1. **Submission** (Main Thread):
```c
void uv_queue_work(uv_loop_t* loop,
                   uv_work_t* req,
                   uv_work_cb work_cb,
                   uv_after_work_cb after_work_cb) {
  // Initialize work request
  uv__req_init(loop, req, UV_WORK);
  req->loop = loop;
  req->work_cb = work_cb;
  req->after_work_cb = after_work_cb;

  // Post to thread pool
  uv__work_submit(loop, &req->work_req, work_cb, after_work_cb);
}
```

2. **Execution** (Worker Thread):
```c
static void uv__queue_work(struct uv__work* w) {
  uv_once(&once, init_threads);

  w->wq[0] = w;
  w->wq[1] = w;

  uv_mutex_lock(&mutex);
  QUEUE_INSERT_TAIL(&wq, &w->wq);
  uv_cond_signal(&cond);  // Wake a worker thread
  uv_mutex_unlock(&mutex);
}
```

3. **Completion** (Main Thread):
```c
static void uv__work_done(uv_async_t* handle) {
  struct uv__work* w;
  uv_loop_t* loop;
  QUEUE* q;
  QUEUE wq;

  loop = container_of(handle, uv_loop_t, wq_async);
  uv_mutex_lock(&loop->wq_mutex);
  QUEUE_MOVE(&loop->wq, &wq);
  uv_mutex_unlock(&loop->wq_mutex);

  while (!QUEUE_EMPTY(&wq)) {
    q = QUEUE_HEAD(&wq);
    QUEUE_REMOVE(q);

    w = container_of(q, struct uv__work, wq);
    w->done(w, 0);  // Execute after_work_cb in main thread
  }
}
```

**Critical Understanding:**

The `uv_async_t` handle (`wq_async`) bridges the thread pool back to the event loop:

1. Worker thread completes work
2. Worker calls `uv_async_send()` → writes to a pipe
3. Main thread's `epoll_wait()` detects readable pipe
4. `uv__work_done()` runs in the poll phase
5. `after_work_cb` executes in the **main thread** (JavaScript callback runs here)

**Example Flow (fs.readFile):**

```javascript
fs.readFile('file.txt', (err, data) => {
  console.log(data);
});
```

Internal flow:
```
Main Thread                      Worker Thread
-----------                      -------------
1. fs.readFile() called
2. uv_queue_work() submitted →   3. Worker dequeues work
   (stores JS callback)          4. read() syscall (BLOCKS)
                                 5. Reads file data
                                 6. uv_async_send()
7. ← Pipe becomes readable       
8. uv__work_done() runs
9. after_work_cb executes
10. JS callback(err, data) fires
```

**Thread Pool Sizing:**

```bash
# Default: 4 threads
# Configure via environment variable:
UV_THREADPOOL_SIZE=8 node app.js

# Max: 1024 threads (but 8-16 is typical)
```

**Why Not Infinite Threads?**  
Each thread:
- Consumes **~2-8 MB** of stack space
- Adds **context switch overhead**
- Competes for CPU cores

More threads ≠ better performance. Optimal size ≈ **1.5 × CPU cores** for I/O-bound workloads.

---

## Part VI: Handles vs. Requests — The Two Primitives

libuv has two fundamental abstractions:

### Handles (Long-lived)

**Definition:** Objects representing long-lived resources that can perform operations.

```c
struct uv_handle_s {
  void* data;                      // User data
  uv_loop_t* loop;                 // Loop this handle belongs to
  uv_handle_type type;             // Handle type (TCP, UDP, Timer, etc.)
  uv_close_cb close_cb;            // Close callback
  void* handle_queue[2];           // Queue pointers
  unsigned int flags;              // Internal flags
  // ... type-specific fields
};
```

**Handle Types:**
- `uv_tcp_t` — TCP sockets
- `uv_udp_t` — UDP sockets
- `uv_timer_t` — Timers
- `uv_idle_t` — Idle callbacks
- `uv_prepare_t` / `uv_check_t` — Phase hooks
- `uv_fs_event_t` — File system watcher
- `uv_signal_t` — Signal handlers
- `uv_process_t` — Child processes

**Lifecycle:**
```
Created → Initialized → Active → Closing → Closed
   ↓           ↓           ↓         ↓         ↓
  new     uv_*_init   uv_*_start  uv_close  (freed)
```

**Reference Counting:**
```c
void uv_ref(uv_handle_t* handle);    // Increment ref count
void uv_unref(uv_handle_t* handle);  // Decrement ref count
```

**Critical Concept:**  
The event loop keeps running as long as there are **active** and **referenced** handles.

```javascript
const server = net.createServer();
server.listen(3000);
server.unref();  // Allows process to exit even though server is listening
```

---

### Requests (Short-lived)

**Definition:** Objects representing one-time operations.

```c
struct uv_req_s {
  void* data;                      // User data
  uv_req_type type;                // Request type
  void* active_queue[2];           // Queue pointers
  // ... type-specific fields
};
```

**Request Types:**
- `uv_write_t` — Write data to stream
- `uv_connect_t` — TCP connection
- `uv_shutdown_t` — Shutdown stream
- `uv_udp_send_t` — Send UDP packet
- `uv_fs_t` — File system operation
- `uv_work_t` — Thread pool work
- `uv_getaddrinfo_t` — DNS lookup

**Lifecycle:**
```
Created → Submitted → Active → Complete → (freed)
```

**Example (TCP Write):**

```javascript
socket.write('Hello');
```

Under the hood:
```c
uv_write_t* req = malloc(sizeof(uv_write_t));
uv_buf_t buf = uv_buf_init("Hello", 5);
uv_write(req, stream, &buf, 1, on_write_complete);

// on_write_complete callback runs when write finishes
```

---

## Part VII: Platform Differences — Abstraction Leakage

### Linux: epoll

**Advantages:**
- Edge-triggered mode reduces syscalls
- Efficient for large numbers of file descriptors
- O(1) event handling

**Limitations:**
- File I/O is always **blocking** (no kernel async support)
- Some operations (e.g., `inotify`) can exhaust file descriptor limits

### macOS/BSD: kqueue

**Advantages:**
- Can monitor files, directories, signals, processes
- More unified event model

**Limitations:**
- Level-triggered only (more syscalls)
- Some edge cases with file descriptor reuse

### Windows: IOCP

**Completely different model:**

- **Proactive I/O**: Operations complete and notify you (vs. reactive: readiness notification)
- Thread pool is **mandatory** for IOCP
- No file descriptor concept — uses `HANDLE` objects

```c
// Unix: "Tell me when socket is readable"
epoll_wait() → EPOLLIN → read() returns data

// Windows: "Read from socket, tell me when done"
ReadFile() → IOCP completion → data already in buffer
```

This is why libuv's abstraction is so valuable — it hides these differences.

---

## Part VIII: Microtasks and the JavaScript Integration

### The Complete Execution Order

```
┌── Event Loop Phase ──┐
│                      │
│  Phase operations    │  ← timers, I/O, etc.
│         ↓            │
│  process.nextTick()  │  ← Microtask queue (priority 1)
│         ↓            │
│  Promise microtasks  │  ← Microtask queue (priority 2)
│         ↓            │
│  Next phase          │
└──────────────────────┘
```

**Implementation in Node.js** (`src/node.cc`):

```cpp
void Environment::RunAndClearNativeImmediates() {
  // Process nextTick queue
  size_t count = native_immediate_callbacks_.size();
  if (count > 0) {
    // Execute all nextTick callbacks
    for (size_t i = 0; i < count; i++) {
      NativeImmediateCallback cb = native_immediate_callbacks_.front();
      native_immediate_callbacks_.pop_front();
      cb(this);
    }
  }
}

void MicrotasksPolicy::RunMicrotasks(Isolate* isolate) {
  // V8 handles Promise microtasks
  isolate->RunMicrotasks();
}
```

**Complete Example:**

```javascript
setImmediate(() => console.log('1'));
setTimeout(() => console.log('2'), 0);
Promise.resolve().then(() => console.log('3'));
process.nextTick(() => console.log('4'));

// Output:
// 4  ← nextTick (runs before anything)
// 3  ← Promise microtask
// 2  ← setTimeout (timers phase)
// 1  ← setImmediate (check phase)
```

---

## Part IX: Memory Model and Zero-Copy Optimizations

### Buffer Management

libuv uses the `uv_buf_t` structure for I/O:

```c
typedef struct uv_buf_t {
  char* base;    // Pointer to data
  size_t len;    // Length of data
} uv_buf_t;
```

**Zero-Copy Read Example:**

```c
// Callback to allocate buffer
void alloc_cb(uv_handle_t* handle, size_t size, uv_buf_t* buf) {
  // Allocate exactly what's needed
  buf->base = malloc(size);
  buf->len = size;
}

// Read callback
void read_cb(uv_stream_t* stream, ssize_t nread, const uv_buf_t* buf) {
  if (nread > 0) {
    // Data is already in buf->base (no copy!)
    process_data(buf->base, nread);
  }

  free(buf->base);  // Free the buffer
}

uv_read_start(stream, alloc_cb, read_cb);
```

**How Zero-Copy Works:**

1. `read()` syscall writes **directly** into `buf->base`
2. No intermediate kernel buffer copy (thanks to `alloc_cb` providing destination)
3. JavaScript `Buffer` wraps this memory without copying

**Node.js Buffer Pooling:**

```javascript
// Node.js pre-allocates 8KB buffer pools
const buf = Buffer.alloc(1024);  // Taken from pool
```

Under the hood:
```cpp
// src/node_buffer.cc
constexpr size_t kPoolSize = 8 * 1024;  // 8KB pool
char* buffer_pool = nullptr;
size_t buffer_pool_offset = 0;

void* Allocate(size_t size) {
  if (size > kPoolSize / 2) {
    return malloc(size);  // Large allocation: separate
  }

  if (buffer_pool_offset + size > kPoolSize) {
    buffer_pool = new char[kPoolSize];  // New pool
    buffer_pool_offset = 0;
  }

  void* ptr = buffer_pool + buffer_pool_offset;
  buffer_pool_offset += size;
  return ptr;
}
```

This reduces:
- **malloc() calls** (expensive syscall)
- **Memory fragmentation**
- **GC pressure** (fewer individual objects)

---

## Part X: Performance Implications and Gotchas

### 1. Timer Precision and setTimeout(fn, 0)

```javascript
setTimeout(() => console.log('A'), 0);
setTimeout(() => console.log('B'), 0);
setTimeout(() => console.log('C'), 0);
```

**What actually happens:**

All three timers are inserted into the min-heap with the **same timeout value**. When the timers phase runs:

```c
// From timer.c - timers are processed in heap order
while (handle->timeout <= loop->time) {
  execute_callback();
  // Next timer...
}
```

Order is **guaranteed** for timers with identical timeouts (FIFO within same timeout).

### 2. Starvation via setImmediate

```javascript
function recurse() {
  setImmediate(recurse);
}
recurse();

// Event loop never exits check phase!
```

**Solution:** Use `setTimeout(fn, 0)` for yielding to other phases:

```javascript
function process_batch() {
  // Process items
  if (more_work) {
    setTimeout(process_batch, 0);  // Yields to other phases
  }
}
```

### 3. Thread Pool Exhaustion

```javascript
// BAD: All 4 thread pool threads blocked
for (let i = 0; i < 4; i++) {
  fs.readFile('huge-file.txt', () => {});
}

// This DNS lookup now waits for a thread!
dns.lookup('example.com', () => {});
```

**Solution:**
```bash
UV_THREADPOOL_SIZE=128 node app.js
```

Or use streaming APIs:
```javascript
const stream = fs.createReadStream('huge-file.txt');
// Reads in chunks, doesn't block thread pool
```

### 4. Event Loop Lag Monitoring

```javascript
const start = Date.now();
setImmediate(() => {
  const lag = Date.now() - start;
  if (lag > 100) {
    console.warn(`Event loop lag: ${lag}ms`);
  }
});
```

**What causes lag?**
- **CPU-bound JS**: `JSON.parse()` on 100MB string
- **Blocking syscalls**: Synchronous `fs.*Sync()` calls
- **Thread pool saturation**: Too many file ops
- **Large heap**: GC pause (not libuv's fault, but feels like lag)

---

## Part XI: Advanced Patterns and Internals

### File Descriptor Lifecycle

```javascript
const server = net.createServer();
server.listen(3000);
```

**System Call Trace:**
```
1. socket(AF_INET, SOCK_STREAM)      → fd = 3
2. setsockopt(3, SO_REUSEADDR)       → success
3. bind(3, {port: 3000})             → success
4. listen(3, backlog)                → success
5. epoll_ctl(epfd, EPOLL_CTL_ADD, 3) → success
```

**Internal Structure:**

```c
struct uv_tcp_s {
  UV_HANDLE_FIELDS           // Base handle fields
  uv_connection_cb connection_cb;
  int delayed_error;
  UV_STREAM_FIELDS           // Stream-specific fields
  // TCP-specific:
  UV_TCP_PRIVATE_FIELDS
};

#define UV_TCP_PRIVATE_FIELDS \
  int fd;                      // File descriptor
```

When a connection arrives:
```
1. epoll_wait() returns fd=3 with EPOLLIN
2. Lookup: loop->watchers[3] → tcp_handle
3. Call: tcp_handle->io_watcher.cb()
4. Inside callback: accept() → new_fd = 4
5. Create client socket: uv_tcp_init() for fd=4
6. Call JS: connection_cb(client)
```

---

### Signal Handling

Signals are tricky in event loops. libuv uses a **self-pipe trick**:

```c
// From src/unix/signal.c

static int uv__signal_start(uv_signal_t* handle,
                            uv_signal_cb signal_cb,
                            int signum) {
  // Install signal handler
  sa.sa_handler = uv__signal_handler;
  sa.sa_flags = SA_RESTART;
  sigaction(signum, &sa, NULL);

  // Add to loop's signal watchers
  // ...
}

static void uv__signal_handler(int signum) {
  // Inside signal handler (async-signal-safe context)
  // Write to pipe to wake event loop
  write(signal_pipe[1], &signum, sizeof(signum));
}
```

In the event loop:
```c
// Poll phase detects readable signal pipe
// Reads signum from pipe
// Dispatches to appropriate signal handle
```

**Why?**  
Signal handlers can only call **async-signal-safe** functions. Writing to a pipe is safe; calling JavaScript is not.

---

### Child Process Management

```javascript
const { spawn } = require('child_process');
const child = spawn('ls', ['-l']);
```

**Under the hood:**

```c
int uv_spawn(uv_loop_t* loop, uv_process_t* handle,
             const uv_process_options_t* options) {
  int pipes[2][2];  // stdin, stdout

  // Create pipes for communication
  pipe(pipes[0]);  // stdin
  pipe(pipes[1]);  // stdout

  pid_t pid = fork();
  if (pid == 0) {
    // Child process
    dup2(pipes[0][0], STDIN_FILENO);
    dup2(pipes[1][1], STDOUT_FILENO);
    close(pipes[0][0]);
    close(pipes[1][1]);

    execvp(options->file, options->args);
    _exit(127);  // execvp failed
  }

  // Parent process
  close(pipes[0][0]);
  close(pipes[1][1]);

  handle->pid = pid;
  handle->exit_cb = options->exit_cb;

  // Register child PID for SIGCHLD monitoring
  uv__signal_register_child_watcher(loop, handle);

  return 0;
}
```

**SIGCHLD Handling:**

```c
static void uv__chld(uv_signal_t* handle, int signum) {
  uv_process_t* process;
  int status;
  pid_t pid;

  // Reap zombie processes
  while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
    // Find process handle
    process = find_process(pid);
    if (process) {
      process->status = status;
      uv__handle_stop(process);

      // Call exit callback
      if (process->exit_cb) {
        process->exit_cb(process, WEXITSTATUS(status), WTERMSIG(status));
      }
    }
  }
}
```

---

## Part XII: Real-World Debugging and Introspection

### Using NODE_DEBUG

```bash
NODE_DEBUG=net,timer node app.js
```

**What this does:**

Node.js has conditional logging:
```javascript
const debuglog = require('util').debuglog('net');

function createServer(options) {
  debuglog('createServer', options);
  // ...
}
```

Produces output like:
```
NET 12345: createServer { allowHalfOpen: false }
TIMER 12345: insert 0x7f8a3c0 timeout=1000
```

### Using --trace-events

```bash
node --trace-events-enabled --trace-event-categories v8,node,node.async_hooks app.js
```

Generates `node_trace.*.log` in **Chrome Trace Format**:

```json
{
  "traceEvents": [
    {"name": "uv_run", "ph": "B", "ts": 1000, "pid": 12345, "tid": 1},
    {"name": "timers", "ph": "B", "ts": 1050, "pid": 12345, "tid": 1},
    {"name": "timers", "ph": "E", "ts": 1080, "pid": 12345, "tid": 1},
    // ... more events
  ]
}
```

Open in `chrome://tracing` to visualize event loop phases!

### Using async_hooks

```javascript
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    console.log(`Init: ${type} (${asyncId}), triggered by ${triggerAsyncId}`);
  },
  before(asyncId) {
    console.log(`Before: ${asyncId}`);
  },
  after(asyncId) {
    console.log(`After: ${asyncId}`);
  },
  destroy(asyncId) {
    console.log(`Destroy: ${asyncId}`);
  }
});

hook.enable();

setTimeout(() => {
  console.log('Timer fired');
}, 100);

// Output:
// Init: Timeout (2), triggered by 1
// Before: 2
// Timer fired
// After: 2
// Destroy: 2
```

**How it works:**

```cpp
// src/async_wrap.cc

void AsyncWrap::MakeCallback(...) {
  // before() hook
  env->async_hooks()->CallbackScope::before();

  // Execute actual callback
  callback->Call(...);

  // after() hook  
  env->async_hooks()->CallbackScope::after();
}
```

Every async operation is wrapped, tracking:
- **init**: Resource created
- **before**: About to execute callback
- **after**: Finished executing callback
- **destroy**: Resource cleaned up

---

## Conclusion: The Invisible Choreography

The Node.js event loop is a **deterministic state machine** powered by libuv's platform abstractions. Every callback you write executes within this structured environment:

1. **Single-threaded execution**: Your JavaScript never runs in parallel with itself
2. **Phase-driven scheduling**: Operations are organized into predictable phases
3. **Non-blocking I/O**: Kernel notifications drive progress, not polling
4. **Thread pool abstraction**: Blocking operations execute off the main thread
5. **Microtask integration**: V8 and libuv coordinate for seamless async/await

**Mental Model:**

Think of the event loop as a **courtroom judge** processing a docket:

- **Timers phase**: "All cases scheduled for before 10:00 AM"
- **Poll phase**: "Waiting for witnesses to arrive" (blocks until I/O)
- **Check phase**: "Immediate motions to be heard"
- **Close phase**: "Final case dispositions"

Every operation in Node.js is either:
- **Enqueued** for future execution
- **Being executed** in the current phase
- **Completed** and removed from the scheduler

There is no magic. Every callback traces back to:
- A **timer expiration** (min-heap)
- An **I/O event** (epoll/kqueue/IOCP)
- A **thread pool completion** (async handle notification)
- A **microtask** (V8 integration)

**The Mastery:**

Understanding the event loop transforms you from a **user** of Node.js into an **architect** of Node.js systems. You now know:

- Why `setImmediate` vs `setTimeout(0)` matters
- How file operations don't block the event loop (thread pool)
- Why timer precision is probabilistic, not deterministic
- How Node.js achieves concurrency without parallelism
- Where platform differences leak through libuv's abstraction

This is the **invisible logic** that powers every Node.js application ever written.

The event loop doesn't just **run** your code—it **orchestrates** the symphony of I/O, timers, callbacks, and asynchronous operations into the high-performance runtime you depend on.

**Further Exploration:**

If you want to dive even deeper:

```bash
# Clone libuv source
git clone https://github.com/libuv/libuv.git
cd libuv

# Key files to study:
src/unix/core.c          # Main event loop
src/unix/linux-core.c    # Linux epoll implementation  
src/threadpool.c         # Thread pool implementation
src/timer.c              # Timer heap management
include/uv.h             # Public API definitions
```

**Performance Tuning Checklist:**

✅ Avoid `fs.*Sync()` calls (blocks event loop)  
✅ Size thread pool appropriately (`UV_THREADPOOL_SIZE`)  
✅ Use streams for large files (prevents thread pool exhaustion)  
✅ Monitor event loop lag with `setImmediate` timing  
✅ Batch timer-heavy operations  
✅ Use `unref()` for background handles  
✅ Consider worker threads for CPU-bound tasks  

**The Ultimate Truth:**

The event loop is not a feature—it's the **foundation**. Every line of asynchronous JavaScript you write is an instruction to this state machine. Understanding it deeply means understanding Node.js itself, from the V8 heap down to the kernel's I/O multiplexer.

You're no longer writing **on top of** Node.js. You're writing **with** its internal machinery, aligned to the phases, respectful of the thread pool, aware of the timer heap, and cognizant of the I/O watcher registry.

This is systems programming disguised as scripting.  
This is the event loop.
