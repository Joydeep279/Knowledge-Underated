# Single and Multithreaded Execution: From Silicon to Software

Let me take you through the complete architecture of how programs execute on modern systems, from the ground up.

## Part I: Foundation - What Is a Thread?

### Basic Definition
A **thread** is the smallest unit of execution that can be scheduled by an operating system. Think of it as a sequence of instructions that the CPU executes. A **process** is a running program that contains at least one thread, along with its memory space, file handles, and other resources.

### The Critical Distinction
- **Single-threaded**: One path of execution. Instructions execute sequentially, one after another.
- **Multi-threaded**: Multiple paths of execution within the same process, sharing the same memory space but executing independently.

## Part II: Hardware Foundation - How CPUs Actually Work

### The Physical Reality

Modern CPUs don't actually "run multiple things at once" in the way most people think. Here's what's really happening:

**Physical Cores**: Each core is an independent execution unit with its own:
- ALU (Arithmetic Logic Unit)
- Registers (fast storage: RAX, RBX, RIP, RSP, etc.)
- L1/L2 cache (private, fast memory)

**Hyperthreading/SMT (Simultaneous Multithreading)**: Intel's marketing term for presenting one physical core as two logical cores. The core has duplicate architectural state (registers, program counter) but shares execution resources.

```
Physical Core Architecture:
┌─────────────────────────────────┐
│   Logical Core 0  Logical Core 1│
│   ┌──────────┐    ┌──────────┐  │
│   │Registers │    │Registers │  │
│   └──────────┘    └──────────┘  │
│                                 │
│     Shared Execution Units      │
│   ┌──────────────────────────┐  │
│   │  ALU  FPU  Load/Store    │  │
│   └──────────────────────────┘  │
│                                 │
│        L1 Cache (shared)        │
└─────────────────────────────────┘
```

### Why This Matters
If you have a 4-core CPU with hyperthreading, you have:
- 4 physical cores
- 8 logical cores (presented to the OS)
- But only 4 sets of actual execution hardware

This is why 8 compute-intensive threads don't run twice as fast as 4 threads on a 4-core hyperthreaded CPU.

## Part III: Operating System Perspective - The Scheduler

### Thread Control Block (TCB)

The OS represents each thread with a data structure. In Linux (kernel source: `include/linux/sched.h`), this is `struct task_struct`:

```c
// Simplified from kernel/sched.h
struct task_struct {
    volatile long state;          // TASK_RUNNING, TASK_INTERRUPTIBLE, etc.
    void *stack;                  // Kernel stack pointer
    unsigned int flags;           // Thread flags
    int prio;                     // Scheduling priority
    
    struct mm_struct *mm;         // Memory descriptor (shared by threads in process)
    
    // CPU context - saved during context switch
    struct thread_struct thread;  // Architecture-specific: registers, FPU state
    
    // Scheduling
    struct sched_entity se;       // Scheduling entity for CFS
    struct sched_rt_entity rt;    // Real-time scheduling
    
    // Files, signals, etc.
    struct files_struct *files;   // File descriptor table (shared)
    struct signal_struct *signal; // Signal handlers (shared)
};
```

**Key Insight**: Threads in the same process share `mm` (memory), `files`, and `signal` structures, but have separate stacks and register states.

### Context Switching - The Magic Trick

When the OS switches from Thread A to Thread B:

```
1. Timer interrupt fires (or thread blocks on I/O)
2. CPU saves current state:
   - Push all registers onto Thread A's kernel stack
   - Save instruction pointer (RIP on x86-64)
   - Save stack pointer (RSP)
   - Save FPU/SSE state (FXSAVE instruction)
   
3. Scheduler runs:
   - Picks next thread (Thread B) based on policy
   - Updates current task pointer
   
4. Restore Thread B's state:
   - Switch to Thread B's stack (load RSP)
   - Restore registers from stack
   - Restore FPU state (FXRSTOR instruction)
   - Jump to saved instruction pointer
```

**Source Reference** (Linux `kernel/sched/core.c`):

```c
// Simplified from actual kernel code
static void __schedule(void) {
    struct task_struct *prev, *next;
    struct rq *rq;
    
    // Get current CPU's run queue
    rq = cpu_rq(smp_processor_id());
    prev = rq->curr;
    
    // Pick next task
    next = pick_next_task(rq, prev);
    
    if (prev != next) {
        // Actual context switch
        context_switch(rq, prev, next);
    }
}

// Architecture-specific (arch/x86/kernel/process_64.c)
__switch_to(struct task_struct *prev_p, struct task_struct *next_p) {
    // Save FS/GS segment registers
    save_fsgs(prev_p);
    
    // Switch stack pointers
    // Switch page tables if different process
    // Restore FS/GS for next task
    
    // The actual register swap happens here
    // Inline assembly that saves/restores all registers
}
```

### The Cost of Context Switching

This isn't free:
- **Direct costs**: 1-5 microseconds (saving/restoring registers, kernel overhead)
- **Indirect costs**: Cache invalidation, TLB flush, pipeline stall

**Why cache matters**: When Thread A runs, it loads data into L1/L2/L3 caches. When Thread B runs, it evicts A's data. When A runs again, cache misses → memory access → 100x slower.

## Part IV: Single-Threaded Execution - Deep Dive

### The Mental Model

In single-threaded execution, there's one program counter, one stack, one sequential flow:

```
Time →
Thread: [Instruction 1] → [Instruction 2] → [Instruction 3] → [Instruction 4]
```

### Real Example: HTTP Server (Single-Threaded)

```c
// Simplified single-threaded server
int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    bind(server_fd, ...);
    listen(server_fd, 10);
    
    while (1) {
        int client_fd = accept(server_fd, ...);  // BLOCKS here
        
        char buffer[1024];
        read(client_fd, buffer, 1024);           // BLOCKS here
        
        // Process request (takes time)
        process_request(buffer);
        
        write(client_fd, response, len);         // BLOCKS here
        close(client_fd);
    }
}
```

**The Problem**: While serving Client A, the server is completely blocked. Client B must wait. If `process_request()` takes 100ms, throughput = 10 requests/second max.

### Execution Timeline

```
Client A connects → accept() → read() → process (100ms) → write() → close()
                                                    ↑
                                        Client B waiting... waiting...
```

### When Single-Threading Wins

1. **CPU-bound tasks with no I/O**: Pure computation benefits from staying on one core (no context switching, cache stays hot)
2. **Simple state**: No locking, no race conditions, easier to reason about
3. **Predictable performance**: No scheduler jitter

**Example**: Redis (mostly single-threaded)
- Uses event loop (epoll/kqueue) for I/O multiplexing
- One thread handles thousands of connections
- Avoids locking overhead
- Trades multi-core utilization for simplicity

## Part V: Multi-Threaded Execution - Architecture

### The Threading Models

**1. Kernel-Level Threads (1:1 model)**
- Each user thread maps to a kernel thread
- OS scheduler manages them
- Linux (NPTL), Windows, modern macOS use this

```
User Space:    Thread1  Thread2  Thread3
                  ↓        ↓        ↓
Kernel Space:  KThread1 KThread2 KThread3
```

**2. User-Level Threads (N:1 model)**
- Multiple user threads map to one kernel thread
- User-space scheduler (no kernel involvement)
- Green threads, goroutines (M:N model)

**3. Hybrid (M:N model)**
- M user threads map to N kernel threads
- Go runtime, Erlang VM

### POSIX Threads (pthreads) - The Linux/Unix Standard

**Source Location**: `nptl/pthread_create.c` in glibc

```c
#include <pthread.h>

// Thread function signature
void* thread_function(void* arg) {
    int* value = (int*)arg;
    printf("Thread processing: %d\n", *value);
    
    // Do work...
    
    return NULL;  // Or return some result
}

int main() {
    pthread_t thread_id;
    int data = 42;
    
    // Create thread
    int result = pthread_create(
        &thread_id,           // Thread handle (output)
        NULL,                 // Attributes (NULL = default)
        thread_function,      // Function to run
        &data                 // Argument to pass
    );
    
    if (result != 0) {
        // Handle error
    }
    
    // Wait for thread to finish
    pthread_join(thread_id, NULL);
    
    return 0;
}
```

### What Actually Happens in pthread_create()

**Simplified glibc flow** (`nptl/pthread_create.c`):

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void*), void *arg) {
    
    // 1. Allocate thread descriptor
    struct pthread *pd = allocate_stack(attr);
    
    // 2. Set up thread-local storage (TLS)
    pd->header.tcb = pd;
    
    // 3. Copy argument and function pointer
    pd->arg = arg;
    pd->start_routine = start_routine;
    
    // 4. Make the actual system call
    int res = ARCH_CLONE(
        clone_flags,          // CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_THREAD
        STACK_VARIABLES_ARGS, // Stack address
        pd,                   // Thread descriptor
        &pd->tid,            // TID location
        tp                    // TLS pointer
    );
    
    return res;
}
```

**The clone() System Call** (Linux-specific):

```c
// arch/x86/kernel/process.c
SYSCALL_DEFINE5(clone, unsigned long, flags, unsigned long, newsp, ...)
{
    return _do_fork(flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}

// kernel/fork.c
long _do_fork(unsigned long clone_flags, ...) {
    struct task_struct *p;
    
    // Allocate new task_struct
    p = copy_process(clone_flags, ...);
    
    if (clone_flags & CLONE_THREAD) {
        // Share memory, files, etc. with parent
        p->mm = current->mm;
        p->files = current->files;
        // But separate stack and registers
    }
    
    // Put on scheduler queue
    wake_up_new_task(p);
    
    return p->pid;
}
```

**Key CLONE flags**:
- `CLONE_VM`: Share memory space (makes it a thread, not process)
- `CLONE_FS`: Share filesystem information
- `CLONE_FILES`: Share file descriptor table
- `CLONE_SIGHAND`: Share signal handlers
- `CLONE_THREAD`: Part of same thread group

## Part VI: The Memory Model - Where Things Get Complex

### Thread Memory Layout

```
Process Virtual Address Space (shared by all threads):
┌─────────────────────────────────┐ 0xFFFFFFFFFFFFFFFF (kernel space)
│       Kernel Space              │
├─────────────────────────────────┤ 0x00007FFFFFFFFFFF (user space limit)
│  Thread 3 Stack (8MB default)   │ ← RSP for Thread 3
├─────────────────────────────────┤
│  Thread 2 Stack                 │ ← RSP for Thread 2
├─────────────────────────────────┤
│  Thread 1 Stack (main)          │ ← RSP for Thread 1
├─────────────────────────────────┤
│         [Guard Pages]           │ (unmapped - causes segfault if overflow)
├─────────────────────────────────┤
│      Memory Mapped Files        │
│      (Shared Libraries)         │
├─────────────────────────────────┤
│           Heap                  │ (grows up, shared by all threads)
│     [malloc/new allocates]      │
├─────────────────────────────────┤
│        BSS (uninitialized)      │ (shared)
├─────────────────────────────────┤
│        Data (initialized)       │ (shared)
├─────────────────────────────────┤
│      Text (code segment)        │ (shared, read-only)
└─────────────────────────────────┘ 0x0000000000400000 (typical start)
```

### What's Shared vs. What's Private

**Shared**:
- Code segment (instructions)
- Global variables
- Heap (malloc'd memory)
- File descriptors
- Memory mapped regions

**Private (Thread-Local)**:
- Stack
- Registers (including instruction pointer)
- Thread-Local Storage (TLS)

### The Race Condition Problem

```c
// BROKEN CODE - Race condition
int counter = 0;  // Shared global

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // NOT ATOMIC!
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);
    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    
    printf("Counter: %d\n", counter);  // Expected: 2000000
                                        // Actual: ??? (less than 2M)
}
```

**Why It Breaks** - Assembly-level view:

```assembly
; C code: counter++
; What the CPU actually does:

mov    rax, [counter]     ; Load counter into register (Thread 1)
                          ; ← CONTEXT SWITCH CAN HAPPEN HERE
add    rax, 1             ; Increment register (Thread 1)
mov    [counter], rax     ; Write back (Thread 1)

; Meanwhile Thread 2 might have:
mov    rbx, [counter]     ; Load OLD value
add    rbx, 1             
mov    [counter], rbx     ; Overwrite Thread 1's increment!
```

**Timeline of the race**:

```
Time    Thread 1                    Thread 2                counter value
────────────────────────────────────────────────────────────────────────
t0      mov rax, [counter] (= 0)                           0
t1                                  mov rbx,[counter] (=0) 0
t2      add rax, 1         (rax=1)                         0
t3                                  add rbx, 1  (rbx=1)    0
t4      mov [counter], rax                                 1
t5                                  mov [counter], rbx     1  ← Lost update!
```

Two increments happened, but counter only increased by 1.

## Part VII: Synchronization Primitives - The Solutions

### 1. Mutex (Mutual Exclusion)

**Concept**: Only one thread can hold the lock at a time.

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);    // Acquire lock
        counter++;                     // Critical section
        pthread_mutex_unlock(&lock);  // Release lock
    }
    return NULL;
}
```

**Implementation** - Simplified from glibc (`nptl/pthread_mutex_lock.c`):

```c
int pthread_mutex_lock(pthread_mutex_t *mutex) {
    // Try to acquire with atomic compare-and-swap
    int oldval = atomic_compare_and_exchange_val_acq(
        &mutex->__data.__lock,
        1,    // New value (locked)
        0     // Expected value (unlocked)
    );
    
    if (oldval != 0) {
        // Lock was held - slow path
        // Use futex to sleep until available
        while (1) {
            if (atomic_exchange_acq(&mutex->__data.__lock, 2) == 0)
                break;  // Got it!
            
            // Sleep until woken by unlock
            futex_wait(&mutex->__data.__lock, 2);
        }
    }
    
    return 0;
}
```

**The Futex System Call** (Linux-specific):

Futex = "Fast Userspace Mutex"

```c
// kernel/futex.c
SYSCALL_DEFINE6(futex, u32 __user *, uaddr, int, op, u32, val, ...)
{
    // If op == FUTEX_WAIT:
    //   1. Check if *uaddr == val (still contended?)
    //   2. If yes, put thread to sleep in kernel wait queue
    //   3. Wake when another thread calls FUTEX_WAKE
    
    // If op == FUTEX_WAKE:
    //   1. Wake up N threads waiting on this address
}
```

**Why futex is clever**: 
- Uncontended case: pure userspace atomic operation (fast)
- Contended case: kernel sleep/wake (avoids busy-waiting)

### 2. Atomic Operations

For simple operations, use hardware atomics:

```c
#include <stdatomic.h>

atomic_int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1);  // Single atomic instruction
    }
    return NULL;
}
```

**Assembly** (x86-64):

```assembly
; atomic_fetch_add(&counter, 1) becomes:
lock add DWORD PTR [counter], 1

; The LOCK prefix:
; - Locks the memory bus
; - Ensures no other core can access this cache line
; - Guarantees atomicity
```

### 3. Condition Variables - Waiting for Events

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

// Thread 1: Wait for data
void* consumer(void* arg) {
    pthread_mutex_lock(&lock);
    
    while (!ready) {
        // Atomically: unlock mutex, sleep, re-lock on wake
        pthread_cond_wait(&cond, &lock);
    }
    
    // Use the data
    pthread_mutex_unlock(&lock);
    return NULL;
}

// Thread 2: Produce data
void* producer(void* arg) {
    pthread_mutex_lock(&lock);
    ready = 1;
    pthread_cond_signal(&cond);  // Wake one waiter
    pthread_mutex_unlock(&lock);
    return NULL;
}
```

**Why the while loop?** Spurious wakeups! A thread can wake from `pthread_cond_wait()` without a signal. Always recheck the condition.

### 4. Read-Write Locks

Multiple readers OR one writer:

```c
pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;
int shared_data = 0;

// Many threads can read simultaneously
void* reader(void* arg) {
    pthread_rwlock_rdlock(&rwlock);
    int value = shared_data;  // Read
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}

// Only one writer at a time, blocks all readers
void* writer(void* arg) {
    pthread_rwlock_wrlock(&rwlock);
    shared_data = 42;  // Write
    pthread_rwlock_unlock(&rwlock);
    return NULL;
}
```

## Part VIII: Real-World Multi-Threading Patterns

### Pattern 1: Thread Pool

Don't create/destroy threads per task (expensive). Reuse a pool:

```c
// Simplified thread pool
typedef struct {
    void (*function)(void*);
    void *argument;
} task_t;

typedef struct {
    pthread_t *threads;
    task_t *task_queue;
    int queue_size;
    int queue_front, queue_rear;
    
    pthread_mutex_t lock;
    pthread_cond_t notify;
    int shutdown;
} threadpool_t;

void* threadpool_worker(void* arg) {
    threadpool_t *pool = (threadpool_t*)arg;
    
    while (1) {
        pthread_mutex_lock(&pool->lock);
        
        // Wait for task or shutdown
        while (pool->queue_size == 0 && !pool->shutdown) {
            pthread_cond_wait(&pool->notify, &pool->lock);
        }
        
        if (pool->shutdown) {
            pthread_mutex_unlock(&pool->lock);
            pthread_exit(NULL);
        }
        
        // Get task from queue
        task_t task = pool->task_queue[pool->queue_front];
        pool->queue_front = (pool->queue_front + 1) % MAX_QUEUE;
        pool->queue_size--;
        
        pthread_mutex_unlock(&pool->lock);
        
        // Execute task (outside lock!)
        task.function(task.argument);
    }
}

void threadpool_add_task(threadpool_t *pool, void (*func)(void*), void *arg) {
    pthread_mutex_lock(&pool->lock);
    
    // Add to queue
    pool->task_queue[pool->queue_rear].function = func;
    pool->task_queue[pool->queue_rear].argument = arg;
    pool->queue_rear = (pool->queue_rear + 1) % MAX_QUEUE;
    pool->queue_size++;
    
    // Wake a worker
    pthread_cond_signal(&pool->notify);
    
    pthread_mutex_unlock(&pool->lock);
}
```

**Used by**: Web servers (Apache worker MPM, nginx threads), databases, Java Executor framework.

### Pattern 2: Producer-Consumer with Bounded Buffer

```c
#define BUFFER_SIZE 10

typedef struct {
    int buffer[BUFFER_SIZE];
    int count;
    int in;   // Producer index
    int out;  // Consumer index
    
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
} bounded_buffer_t;

void produce(bounded_buffer_t *bb, int item) {
    pthread_mutex_lock(&bb->mutex);
    
    // Wait if buffer full
    while (bb->count == BUFFER_SIZE) {
        pthread_cond_wait(&bb->not_full, &bb->mutex);
    }
    
    bb->buffer[bb->in] = item;
    bb->in = (bb->in + 1) % BUFFER_SIZE;
    bb->count++;
    
    pthread_cond_signal(&bb->not_empty);  // Wake consumer
    pthread_mutex_unlock(&bb->mutex);
}

int consume(bounded_buffer_t *bb) {
    pthread_mutex_lock(&bb->mutex);
    
    // Wait if buffer empty
    while (bb->count == 0) {
        pthread_cond_wait(&bb->not_empty, &bb->mutex);
    }
    
    int item = bb->buffer[bb->out];
    bb->out = (bb->out + 1) % BUFFER_SIZE;
    bb->count--;
    
    pthread_cond_signal(&bb->not_full);  // Wake producer
    pthread_mutex_unlock(&bb->mutex);
    
    return item;
}
```

### Pattern 3: Lock-Free Data Structures

For maximum performance, avoid locks entirely using atomics:

```c
// Lock-free stack (Treiber stack)
typedef struct node {
    int data;
    struct node *next;
} node_t;

typedef struct {
    atomic_uintptr_t top;  // Atomic pointer
} lfstack_t;

void push(lfstack_t *stack, int value) {
    node_t *new_node = malloc(sizeof(node_t));
    new_node->data = value;
    
    uintptr_t old_top, new_top = (uintptr_t)new_node;
    
    do {
        old_top = atomic_load(&stack->top);
        new_node->next = (node_t*)old_top;
        
        // Try to CAS: if top is still old_top, set it to new_node
    } while (!atomic_compare_exchange_weak(&stack->top, &old_top, new_top));
}

int pop(lfstack_t *stack) {
    uintptr_t old_top, new_top;
    node_t *node;
    
    do {
        old_top = atomic_load(&stack->top);
        if (old_top == 0) return -1;  // Empty
        
        node = (node_t*)old_top;
        new_top = (uintptr_t)node->next;
        
    } while (!atomic_compare_exchange_weak(&stack->top, &old_top, new_top));
    
    int value = node->data;
    free(node);  // Danger: ABA problem!
    return value;
}
```

**The ABA Problem**: 
1. Thread 1 reads top = A
2. Thread 2 pops A, pops B, pushes A back (top = A again)
3. Thread 1's CAS succeeds (top still equals A), but it's a different A!

**Solution**: Use generation counters or hazard pointers.

## Part IX: CPU Cache Coherency - The Hidden Complexity

### Why Caches Matter for Threading

Modern CPUs have multiple levels of cache:

```
Core 0:                Core 1:
┌─────────┐           ┌─────────┐
│ L1 32KB │           │ L1 32KB │  Private, ~4 cycles latency
├─────────┤           ├─────────┤
│ L2 256KB│           │ L2 256KB│  Private, ~12 cycles
└────┬────┘           └────┬────┘
     │                     │
     └─────────┬───────────┘
               │
         ┌─────┴──────┐
         │  L3 8MB    │              Shared, ~40 cycles
         └─────┬──────┘
               │
         ┌─────┴──────┐
         │  RAM       │              ~200 cycles
         └────────────┘
```

### MESI Protocol (Cache Coherency)

When multiple cores access the same memory:

**States**:
- **M**odified: This cache has the only valid copy, dirty (different from RAM)
- **E**xclusive: This cache has the only copy, clean (same as RAM)
- **S**hared: Multiple caches have this line
- **I**nvalid: This cache line is stale

**Example**:

```c
int x = 0;  // Initially in RAM

// Core 0 reads x
// Core 0 L1: x=0 (E - Exclusive)

// Core 1 reads x
// Core 0 L1: x=0 (S - Shared)
// Core 1 L1: x=0 (S - Shared)

// Core 0 writes x=1
// Core 0 L1: x=1 (M - Modified)
// Core 1 L1: x=? (I - Invalid, must re-fetch)
```

**Hardware implementation**: Cores snoop on the bus. When Core 0 writes, it broadcasts invalidation. Core 1's cache controller sees this and marks the line Invalid.

### False Sharing - The Silent Performance Killer

```c
struct {
    int counter1;  // Used by Thread 1
    int counter2;  // Used by Thread 2
} counters;

// Thread 1
void* thread1(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counters.counter1++;
    }
}

// Thread 2
void* thread2(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counters.counter2++;
    }
}
```

**The problem**: Cache lines are typically 64 bytes. `counter1` and `counter2` are in the same cache line!

```
Cache Line (64 bytes):
[counter1: 4B][counter2: 4B][padding: 56B]
     ↑              ↑
   Thread 1      Thread 2
```

Every write by Thread 1 invalidates Thread 2's cache line and vice versa, even though they're accessing different variables!

**Solution**: Padding

```c
struct {
    int counter1;
    char padding1[60];  // Force to different cache lines
    int counter2;
    char padding2[60];
} counters __attribute__((aligned(64)));
```

## Part X: Modern Concurrency Models

### 1. Async/Await (Event-Driven)

Instead of threads, use asynchronous I/O:

```javascript
// Node.js - single-threaded event loop
async function handleRequest(req, res) {
    // Doesn't block thread while waiting for DB
    const data = await database.query('SELECT * FROM users');
    res.json(data);
}
```

**Under the hood** (libuv - Node's I/O library):

```c
// Simplified event loop
void uv_run(uv_loop_t *loop) {
    while (has_active_handles(loop)) {
        // 1. Update time
        uv_update_time(loop);
        
        // 2. Run timers
        uv_run_timers(loop);
        
        // 3. Poll for I/O (epoll/kqueue/IOCP)
        uv_io_poll(loop, timeout);
        
        // 4. Run pending callbacks
        uv_run_pending(loop);
        
        // 5. Run idle/check handles
        uv_run_check(loop);
        uv_run_idle(loop);
    }
}
```

**Why it works**: I/O operations (network, disk) are slow. Instead of blocking, register a callback and let the OS notify when data is ready.

### 2. Go Goroutines (M:N Threading)

Go multiplexes many goroutines onto fewer OS threads:

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // This looks like it blocks, but it doesn't block the OS thread
    data := database.Query("SELECT * FROM users")
    json.NewEncoder(w).Encode(data)
}

func main() {
    // Launch 10,000 goroutines - not 10,000 OS threads!
    for i := 0; i < 10000; i++ {
        go handleRequest()
    }
}
```

**Go Runtime Scheduler** (`runtime/proc.go`):

```
M goroutines : N OS threads

┌──────────┐  ┌──────────┐  ┌──────────┐
│Goroutine1│  │Goroutine2│  │Goroutine3│  User level (cheap, ~2KB stack)
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┼─────────────┘
                   │
            ┌──────▼──────┐
            │  Processor  │  Logical CPU (P)
            │   (P) 0     │
            └──────┬──────┘
                   │
            ┌──────▼──────┐
            │  OS Thread  │  (M - Machine)
            │   (M) 0     │
            └─────────────┘
```

**Key Components**:

- **G (Goroutine)**: Lightweight user-space thread
- **M (Machine)**: OS thread
- **P (Processor)**: Scheduling context, one per logical CPU

**Scheduler Algorithm** (simplified from `runtime/proc.go`):

```go
// Simplified Go scheduler logic
func schedule() {
    // Get current goroutine context
    _g_ := getg()
    
    // Find a runnable goroutine
    gp := findrunnable() // Tries local queue, global queue, steal from others
    
    // Execute the goroutine
    execute(gp)
}

func findrunnable() *g {
    _g_ := getg()
    _p_ := _g_.m.p.ptr()
    
top:
    // 1. Try local run queue (lock-free)
    if gp := runqget(_p_); gp != nil {
        return gp
    }
    
    // 2. Try global run queue (requires lock)
    if sched.runqsize != 0 {
        lock(&sched.lock)
        gp := globrunqget(_p_, 0)
        unlock(&sched.lock)
        if gp != nil {
            return gp
        }
    }
    
    // 3. Poll network (non-blocking)
    if netpollinited() {
        list := netpoll(0) // Non-blocking
        if !list.empty() {
            gp := list.pop()
            injectglist(&list)
            return gp
        }
    }
    
    // 4. Work stealing from other P's
    for i := 0; i < 4; i++ {
        for enum := stealOrder.start(); !enum.done(); enum.next() {
            p2 := allp[enum.position()]
            if gp := runqsteal(_p_, p2); gp != nil {
                return gp
            }
        }
    }
    
    // Nothing found, park thread
    goto top
}
```

**Why Goroutines are Efficient**:

1. **Small stack**: Start at 2KB (vs 1-8MB for OS threads)
2. **Cheap context switch**: No kernel involvement, just save/restore a few registers
3. **Work stealing**: Idle threads steal work from busy ones
4. **Integrated I/O**: Network poller built into scheduler

**Example of Blocking I/O Handling**:

```go
// When a goroutine does blocking I/O:
conn, err := net.Dial("tcp", "example.com:80")

// Internally:
// 1. Goroutine calls into runtime (syscall)
// 2. Runtime registers fd with epoll/kqueue
// 3. Runtime parks goroutine (moves off M)
// 4. M picks up another goroutine to run
// 5. When I/O ready, runtime wakes goroutine
```

**Runtime Code** (`runtime/netpoll_epoll.go`):

```go
func netpollopen(fd uintptr, pd *pollDesc) int32 {
    var ev epollevent
    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
    
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}

func netpoll(delay int64) gList {
    var events [128]epollevent
    
    // Wait for I/O events (with timeout)
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    
    var toRun gList
    for i := int32(0); i < n; i++ {
        ev := &events[i]
        pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
        
        // Wake up goroutines waiting on this fd
        var mode int32
        if ev.events & (_EPOLLIN | _EPOLLRDHUP | _EPOLLHUP | _EPOLLERR) != 0 {
            mode += 'r'
        }
        if ev.events & (_EPOLLOUT | _EPOLLHUP | _EPOLLERR) != 0 {
            mode += 'w'
        }
        
        if mode != 0 {
            netpollready(&toRun, pd, mode)
        }
    }
    
    return toRun
}
```

### 3. Rust's Ownership Model - Compile-Time Safety

Rust prevents data races at compile time:

```rust
use std::thread;

// This WON'T compile - data race detected!
fn broken_example() {
    let mut data = vec![1, 2, 3];
    
    thread::spawn(|| {
        data.push(4);  // Error: closure may outlive current function
    });
    
    data.push(5);  // Error: cannot borrow as mutable
}

// Correct: Use Arc (Atomic Reference Counted) + Mutex
use std::sync::{Arc, Mutex};

fn safe_example() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));
    let data_clone = Arc::clone(&data);
    
    thread::spawn(move || {
        let mut d = data_clone.lock().unwrap();
        d.push(4);
    });
    
    let mut d = data.lock().unwrap();
    d.push(5);
}
```

**How It Works**:

1. **Ownership**: Every value has exactly one owner
2. **Borrowing**: Can have either:
   - Multiple immutable references (`&T`)
   - ONE mutable reference (`&mut T`)
3. **Compiler enforces**: No data races possible

**Example of Compile-Time Check**:

```rust
// Rust compiler's borrow checker analysis:

let mut x = 5;
let r1 = &x;      // Immutable borrow starts
let r2 = &x;      // OK - multiple immutable borrows allowed
println!("{} {}", r1, r2);  // r1, r2 last used here (scope ends)

let r3 = &mut x;  // OK - mutable borrow after immutable scope
*r3 = 10;

// This FAILS:
let mut y = 5;
let r1 = &y;         // Immutable borrow
let r2 = &mut y;     // ERROR: cannot borrow as mutable while borrowed as immutable
println!("{}", r1);
```

## Part XI: Performance Analysis and Profiling

### Understanding Thread Performance

**Key Metrics**:

1. **CPU utilization**: Are cores actually busy?
2. **Lock contention**: How long threads wait for locks
3. **Context switches**: How often (voluntary vs involuntary)
4. **Cache misses**: L1/L2/L3 miss rates

### Linux Profiling Tools

**1. perf - CPU Performance Counter**

```bash
# Profile a running process
perf record -p <PID> -g -- sleep 30

# View report
perf report

# See where time is spent
perf top

# Cache miss analysis
perf stat -e cache-references,cache-misses ./my_program
```

**Example Output**:

```
Performance counter stats for './multithreaded_app':

    45,234,567      cache-references
     2,123,456      cache-misses              #    4.69% of all cache refs

  5.234567890 seconds time elapsed
```

**2. strace - System Call Tracing**

```bash
# See all system calls
strace -c ./program

# Filter for futex (lock operations)
strace -e trace=futex ./program
```

**Output**:

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.23    0.123456          12     10234           futex
 23.45    0.064123           8      8123           read
 15.67    0.042789           5      8567           write
```

High futex time = lock contention!

**3. pstack - Thread Stack Traces**

```bash
# See what all threads are doing
pstack <PID>
```

**4. Custom Timing Code**

```c
#include <time.h>

struct timespec start, end;
clock_gettime(CLOCK_MONOTONIC, &start);

// Critical section
pthread_mutex_lock(&lock);
// ... work ...
pthread_mutex_unlock(&lock);

clock_gettime(CLOCK_MONOTONIC, &end);
long long elapsed_ns = (end.tv_sec - start.tv_sec) * 1000000000LL + 
                       (end.tv_nsec - start.tv_nsec);

printf("Lock held for %lld ns\n", elapsed_ns);
```

### Amdahl's Law - The Theoretical Limit

**Formula**: 
```
Speedup = 1 / ((1 - P) + P/N)
```

Where:
- P = Portion of code that can be parallelized
- N = Number of processors

**Example**:

If 90% of your code is parallelizable:

```
1 core:  Speedup = 1.0x
2 cores: Speedup = 1 / (0.1 + 0.9/2)  = 1.82x
4 cores: Speedup = 1 / (0.1 + 0.9/4)  = 3.08x
8 cores: Speedup = 1 / (0.1 + 0.9/8)  = 4.71x
∞ cores: Speedup = 1 / 0.1            = 10x (maximum possible)
```

**Key Insight**: The serial portion (10%) dominates. You can never get more than 10x speedup, regardless of cores.

**Real-World Example**:

```c
// Processing 1,000,000 items
// Setup: 1ms (serial)
// Process: 1000ms (parallelizable)
// Cleanup: 1ms (serial)

// Single-threaded: 1002ms
// 8 threads: 1 + 1000/8 + 1 = 127ms → 7.9x speedup
// 100 threads: 1 + 1000/100 + 1 = 12ms → 83.5x speedup? NO!
// Reality: Context switching, cache contention → maybe 40x
```

## Part XII: Common Threading Bugs and How to Debug Them

### 1. Deadlock

**Classic Example**: Dining Philosophers

```c
pthread_mutex_t fork[5];

void* philosopher(void* arg) {
    int id = *(int*)arg;
    int left = id;
    int right = (id + 1) % 5;
    
    while (1) {
        // Think
        
        pthread_mutex_lock(&fork[left]);   // Pick up left fork
        pthread_mutex_lock(&fork[right]);  // Pick up right fork
        
        // Eat
        
        pthread_mutex_unlock(&fork[right]);
        pthread_mutex_unlock(&fork[left]);
    }
}
```

**Deadlock Scenario**:
```
Time    Phil 0          Phil 1          Phil 2
t0      lock(fork0)
t1                      lock(fork1)
t2                                      lock(fork2)
t3      [wait fork1]
t4                      [wait fork2]
t5                                      [wait fork0]
        ← DEADLOCK: Everyone waiting →
```

**Solutions**:

**A. Lock Ordering** (always acquire locks in same global order):

```c
void* philosopher(void* arg) {
    int id = *(int*)arg;
    int first = min(id, (id + 1) % 5);
    int second = max(id, (id + 1) % 5);
    
    pthread_mutex_lock(&fork[first]);   // Always lock lower number first
    pthread_mutex_lock(&fork[second]);
    // Eat
    pthread_mutex_unlock(&fork[second]);
    pthread_mutex_unlock(&fork[first]);
}
```

**B. Trylock with Backoff**:

```c
while (1) {
    pthread_mutex_lock(&fork[left]);
    
    if (pthread_mutex_trylock(&fork[right]) == 0) {
        // Got both forks - eat!
        pthread_mutex_unlock(&fork[right]);
        pthread_mutex_unlock(&fork[left]);
        break;
    } else {
        // Couldn't get right fork - release left and retry
        pthread_mutex_unlock(&fork[left]);
        usleep(random() % 100);  // Random backoff
    }
}
```

**C. Resource Hierarchy** (number all resources, acquire in increasing order)

**Detecting Deadlock**:

```bash
# Linux: Check for deadlocked threads
gdb -p <PID>
(gdb) thread apply all bt

# Or use pstack
pstack <PID>

# Look for threads all waiting on pthread_mutex_lock
```

### 2. Race Condition - The Heisenbug

**Characteristic**: Bug disappears when you add logging/debugging!

```c
// Bug: Checking and using in separate operations
if (queue_size > 0) {  // Thread A checks
    // ← Thread B could empty queue here!
    item = dequeue();  // Thread A crashes!
}
```

**Why logging hides it**: Adding `printf()` changes timing, making the race window smaller.

**Fix**:

```c
pthread_mutex_lock(&lock);
if (queue_size > 0) {
    item = dequeue();
}
pthread_mutex_unlock(&lock);
```

### 3. Priority Inversion

**Scenario**:
- High-priority thread H needs lock L
- Low-priority thread L holds lock L
- Medium-priority thread M runs, preempting L
- H is blocked by L, which is blocked by M!

```
Priority:  High (H)  >  Medium (M)  >  Low (L)

Time    Thread H        Thread M        Thread L
t0                                      lock(L)
t1      [wait L]
t2                      [running]       [blocked by M]
t3      [still wait]    [running]       [blocked by M]
        ← Priority inversion: H waits for L, but M runs! →
```

**Solution**: Priority inheritance - L temporarily inherits H's priority.

**Linux Implementation** (`kernel/locking/rtmutex.c`):

```c
// When high-priority task blocks on lock held by low-priority task:
static void rt_mutex_adjust_prio(struct task_struct *task) {
    int prio = min(task->normal_prio, get_lock_owner_prio(task));
    
    if (task->prio != prio)
        rt_mutex_setprio(task, prio);  // Boost priority
}
```

### 4. ABA Problem (Lock-Free Structures)

**Example**:

```
Initial: top → A → B → C

Thread 1:                   Thread 2:
read top (A)
                            pop A
                            pop B
                            push A back
                            (top → A → C now)
CAS succeeds (top == A)
but B is gone!
```

**Solution**: Use tagged pointers or generation counters:

```c
typedef struct {
    uintptr_t ptr : 48;       // 48 bits for pointer
    uintptr_t counter : 16;   // 16 bits for generation
} tagged_ptr_t;

atomic_uintptr_t top;  // Contains both pointer and counter

void push(lfstack_t *stack, node_t *node) {
    tagged_ptr_t old_top, new_top;
    
    do {
        old_top.value = atomic_load(&stack->top);
        node->next = (node_t*)old_top.ptr;
        
        new_top.ptr = (uintptr_t)node;
        new_top.counter = old_top.counter + 1;  // Increment generation
        
    } while (!atomic_compare_exchange_weak(&stack->top, &old_top.value, new_top.value));
}
```

### 5. Memory Ordering Issues

**The Problem**: CPUs and compilers reorder instructions!

```c
// Thread 1
data = 42;
ready = 1;

// Thread 2
if (ready) {
    assert(data == 42);  // Can fail!
}
```

**Why?** Compiler or CPU might reorder Thread 1's instructions:

```assembly
; Compiler might generate:
mov [ready], 1      ; Set flag first
mov [data], 42      ; Then set data (reordered!)
```

**Solution**: Memory barriers

```c
// C11 atomics
atomic_int ready = 0;
int data;

// Thread 1
data = 42;
atomic_store_explicit(&ready, 1, memory_order_release);  // Barrier!

// Thread 2
if (atomic_load_explicit(&ready, memory_order_acquire) == 1) {
    assert(data == 42);  // Now guaranteed!
}
```

**Memory Ordering Types**:

```c
memory_order_relaxed:  // No ordering guarantees (fastest)
memory_order_acquire:  // No reads/writes before this can move after
memory_order_release:  // No reads/writes after this can move before
memory_order_acq_rel:  // Both acquire and release
memory_order_seq_cst:  // Sequentially consistent (slowest, safest)
```

**Assembly Example** (x86-64):

```assembly
; memory_order_release
mov [data], 42
mfence              ; Memory fence - wait for all prior writes
mov [ready], 1

; memory_order_acquire
mov eax, [ready]
test eax, eax
je not_ready
mfence              ; Memory fence - subsequent reads see prior writes
mov eax, [data]
```

## Part XIII: Threading Best Practices

### 1. Minimize Shared State

**Bad**:

```c
// Global state accessed by all threads
int global_counter = 0;
pthread_mutex_t lock;

void* worker(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&lock);
        global_counter++;
        pthread_mutex_unlock(&lock);
    }
}
```

**Good**:

```c
// Thread-local counters, combine at end
void* worker(void* arg) {
    int local_counter = 0;
    
    for (int i = 0; i < 1000000; i++) {
        local_counter++;  // No lock needed!
    }
    
    // Only lock once at the end
    pthread_mutex_lock(&lock);
    global_counter += local_counter;
    pthread_mutex_unlock(&lock);
}
```

**Performance Impact**:
- Bad version: 1,000,000 lock operations per thread
- Good version: 1 lock operation per thread
- Speedup: ~1000x for this code

### 2. Lock Granularity

**Coarse-grained** (one big lock):

```c
pthread_mutex_t big_lock;

void process_request(request_t *req) {
    pthread_mutex_lock(&big_lock);
    
    parse_request(req);
    query_database(req);
    format_response(req);
    
    pthread_mutex_unlock(&big_lock);
}
```

**Pros**: Simple, no deadlocks
**Cons**: Serializes everything, poor scalability

**Fine-grained** (many small locks):

```c
typedef struct {
    data_t data;
    pthread_mutex_t lock;
} bucket_t;

bucket_t hash_table[1000];

void insert(int key, data_t value) {
    int bucket = hash(key) % 1000;
    
    pthread_mutex_lock(&hash_table[bucket].lock);  // Lock only this bucket
    // ... insert ...
    pthread_mutex_unlock(&hash_table[bucket].lock);
}
```

**Pros**: Better parallelism
**Cons**: More complex, deadlock risk, more memory overhead

### 3. Use Thread-Safe Libraries

**Thread-safe functions** (POSIX):

```c
// Thread-safe versions (suffix _r for "reentrant")
struct tm* localtime_r(const time_t *timep, struct tm *result);
char* strtok_r(char *str, const char *delim, char **saveptr);
int rand_r(unsigned int *seedp);

// NOT thread-safe (use _r versions):
struct tm* localtime(const time_t *timep);  // Returns static buffer
char* strtok(char *str, const char *delim); // Uses static state
```

### 4. Avoid Busy-Waiting

**Bad**:

```c
// Spin-wait burns CPU
while (!ready) {
    // Busy loop - wastes CPU cycles
}
```

**Good**:

```c
pthread_mutex_lock(&lock);
while (!ready) {
    pthread_cond_wait(&cond, &lock);  // Sleep until signaled
}
pthread_mutex_unlock(&lock);
```

**Exception**: Spinlocks for very short critical sections (< 100 nanoseconds):

```c
pthread_spinlock_t spinlock;

pthread_spin_lock(&spinlock);
// Very brief operation (< few microseconds)
counter++;
pthread_spin_unlock(&spinlock);
```

**When to use spinlocks**:
- Critical section < 100ns
- Threads run on separate cores
- Can't afford scheduler overhead

### 5. Thread Pool Sizing

**Rule of thumb**:

- **CPU-bound**: `threads = num_cores` (or `num_cores + 1`)
- **I/O-bound**: `threads = num_cores * (1 + wait_time / compute_time)`

**Example**:

```
Task: 10ms compute, 90ms waiting for network
Ratio: 90/10 = 9
Cores: 8
Optimal threads: 8 * (1 + 9) = 80 threads
```

**Measurement-based tuning**:

```c
void calibrate_thread_pool() {
    int min_threads = num_cores;
    int max_threads = num_cores * 20;
    int best_threads = min_threads;
    double best_throughput = 0;
    
    for (int n = min_threads; n <= max_threads; n += num_cores) {
        double throughput = benchmark_with_threads(n);
        
        if (throughput > best_throughput) {
            best_throughput = throughput;
            best_threads = n;
        } else if (throughput < best_throughput * 0.95) {
            break;  // Performance degrading
        }
    }
    
    set_pool_size(best_threads);
}
```

## Part XIV: Real-World Case Studies

### Case Study 1: Web Server Architecture

**Apache MPM Worker** (Multi-Processing Module):

```
Process Model:
┌─────────────────────────────────────┐
│ Parent Process                      │
│  - Listens on port 80               │
│  - Manages child processes          │
└──────────┬──────────────────────────┘
           │
           ├─── Child Process 1
           │     ├─ Thread 1 (handles request)
           │     ├─ Thread 2 (handles request)
           │     └─ Thread 25
           │
           ├─── Child Process 2
           │     ├─ Thread 1
           │     └─ ...
           │
           └─── Child Process N
```

**Configuration**:

```apache
<IfModule mpm_worker_module>
    StartServers             3
    MinSpareThreads         75
    MaxSpareThreads        250
    ThreadsPerChild         25
    MaxRequestWorkers      400
    MaxConnectionsPerChild   0
</IfModule>
```

**Why this design?**

1. **Multiple processes**: Isolate memory (crash in one doesn't kill others)
2. **Threads per process**: Share memory within process (efficient)
3. **Thread pool**: Pre-spawned threads avoid creation overhead

**nginx** (Different approach - event-driven):

```
Process Model:
Master Process
  ├─ Worker Process 1 (event loop, handles 1000s of connections)
  ├─ Worker Process 2 (one per core)
  └─ Worker Process N
```

**Why nginx is faster for I/O**:
- One thread handles many connections (async I/O)
- No context switching between threads
- Lower memory overhead

### Case Study 2: Database Connection Pool

```c
typedef struct {
    connection_t *connections[MAX_CONN];
    int num_connections;
    int next_connection;
    
    pthread_mutex_t lock;
    pthread_cond_t available;
    
    int active_count;
} connection_pool_t;

connection_t* acquire_connection(connection_pool_t *pool) {
    pthread_mutex_lock(&pool->lock);
    
    // Wait while no connections available
    while (pool->active_count >= pool->num_connections) {
        pthread_cond_wait(&pool->available, &pool->lock);
    }
    
    // Get next connection (round-robin)
    connection_t *conn = pool->connections[pool->next_connection];
    pool->next_connection = (pool->next_connection + 1) % pool->num_connections;
    pool->active_count++;
    
    pthread_mutex_unlock(&pool->lock);
    return conn;
}

void release_connection(connection_pool_t *pool, connection_t *conn) {
    pthread_mutex_lock(&pool->lock);
    
    pool->active_count--;
    
    // Wake one waiting thread
    pthread_cond_signal(&pool->available);
    
    pthread_mutex_unlock(&pool->lock);
}
```

**Why connection pooling?**
- Creating TCP connection: 3-way handshake (~1-100ms)
- Database authentication: Another round trip
- Pool: Reuse existing connections (~0.01ms)

**Tuning**:

```
Too few connections:  Threads wait for connections
Too many connections: Database overload, memory waste

Optimal = (num_threads * active_percentage) + buffer
```

### Case Study 3: Video Game Engine (Frame-based Threading)

```c
// Main game loop
void game_loop() {
    while (running) {
        double frame_start = get_time();
        
        // 1. Input (main thread)
        process_input();
        
        // 2. Update (job system)
        job_t jobs[] = {
            {update_physics, physics_data},
            {update_ai, ai_data},
            {update_animations, anim_data},
            {update_particles, particle_data}
        };
        
        execute_jobs_parallel(jobs, 4);
        wait_for_jobs();
        
        // 3. Render (main thread - GPU commands must be serialized)
        render_frame();
        
        // 4. Frame rate limiting
        double frame_time = get_time() - frame_start;
        if (frame_time < TARGET_FRAME_TIME) {
            sleep(TARGET_FRAME_TIME - frame_time);
        }
    }
}
```

**Job System** (work-stealing queue):

```c
typedef struct {
    void (*function)(void*);
    void *data;
} job_t;

typedef struct {
    job_t queue[MAX_JOBS];
    atomic_int head;
    atomic_int tail;
} job_queue_t;

void worker_thread() {
    while (1) {
        job_t job;
        
        // Try to get job from own queue
        if (dequeue_local(&job)) {
            job.function(job.data);
            continue;
        }
        
        // Steal from other threads
        if (steal_job(&job)) {
            job.function(job.data);
            continue;
        }
        
        // No work - sleep
        wait_for_work();
    }
}
```

**Why this works**:
- Frame budget: 16.67ms for 60 FPS
- Parallel update: 4 jobs on 8 cores = 2ms (vs 8ms sequential)
- Leaves time for rendering and system overhead

## Part XV: Future of Threading - What's Next?

### 1. Heterogeneous Computing

**CPU + GPU threading**:

```cuda
// CUDA - thousands of lightweight threads
__global__ void vector_add(float *a, float *b, float *c, int n) {
    int index = blockIdx.x * blockDim.x + threadIdx.x;
    if (index < n) {
        c[index] = a[index] + b[index];
    }
}

// Launch 1,000,000 threads!
int threads_per_block = 256;
int num_blocks = (n + threads_per_block - 1) / threads_per_block;
vector_add<<<num_blocks, threads_per_block>>>(d_a, d_b, d_c, n);
```

**GPU threads are different**:
- Much lighter weight (hardware-scheduled)
- SIMT (Single Instruction, Multiple Threads)
- 1000s of threads per core (vs 2 for CPU)

### 2. Transactional Memory

**Concept**: Database-style transactions for memory

```c
// Hypothetical API
__transaction_atomic {
    // All reads/writes are transactional
    account1.balance -= 100;
    account2.balance += 100;
    
    // If conflict detected, automatically retry
}
```

**Hardware Support**: Intel TSX (Transactional Synchronization Extensions)

```c
// Actual Intel TSX API
if (_xbegin() == _XBEGIN_STARTED) {
    // Transaction
    counter++;
    _xend();
} else {
    // Fallback to mutex
    pthread_mutex_lock(&lock);
    counter++;
    pthread_mutex_unlock(&lock);
}
```

### 3. Language-Level Actors (Erlang Model)

**No shared memory - message passing only**:

```erlang
% Actor 1
receive
    {add, X, Y} -> 
        Result = X + Y,
        Sender ! {result, Result}
end.

% Actor 2
Actor1 ! {add, 5, 3},
receive
    {result, R} -> io:format("Result: ~p~n", [R])
end.
```

**Advantages**:
- No locks needed
- Natural distribution across machines
- Fault isolation

## Conclusion: The Mental Model

When you write multi-threaded code, visualize this stack:

```
Your Code (pthread_create, mutex_lock, etc.)
          ↓
User Space Library (glibc/libc - manages thread state)
          ↓
System Calls (clone, futex - enter kernel)
          ↓
OS Scheduler (picks next thread to run)
          ↓
Context Switch (save/restore CPU registers)
          ↓
Hardware (CPU cores, cache coherency, memory)
```

**Key Takeaways**:

1. **Threads share memory** - this is both power and danger
2. **Context switching isn't free** - measure before adding threads
3. **Cache coherency creates hidden costs** - design for data locality
4. **Locks are slow** - minimize critical sections, prefer lock-free when possible
5. **The scheduler is unpredictable** - never assume execution order
6. **Memory ordering matters** - use proper barriers on weakly-ordered architectures
7. **Single-threaded can be faster** - event loops + async I/O often win for I/O-bound work

## Part XVI: Advanced Topics - Going Deeper

### 1. NUMA (Non-Uniform Memory Access) Architecture

On modern multi-socket servers, memory access is non-uniform:

```
Server with 2 CPU sockets:

┌─────────────────────┐         ┌─────────────────────┐
│   CPU Socket 0      │         │   CPU Socket 1      │
│   ┌───┬───┬───┬───┐ │         │   ┌───┬───┬───┬───┐ │
│   │C0 │C1 │C2 │C3 │ │         │   │C4 │C5 │C6 │C7 │ │
│   └───┴───┴───┴───┘ │         │   └───┴───┴───┴───┘ │
│         ↓           │         │         ↓           │
│   ┌─────────────┐   │         │   ┌─────────────┐   │
│   │  Memory     │   │  ←QPI→  │   │  Memory     │   │
│   │  32 GB      │   │ (slow!) │   │  32 GB      │   │
│   │  (local)    │   │         │   │  (local)    │   │
│   └─────────────┘   │         │   └─────────────┘   │
└─────────────────────┘         └─────────────────────┘

Access times:
- Core 0 → Local memory (Socket 0):  ~100ns
- Core 0 → Remote memory (Socket 1): ~200ns (2x slower!)
```

**NUMA-aware programming**:

```c
#include <numa.h>
#include <numaif.h>

void* numa_aware_thread(void* arg) {
    int cpu = *(int*)arg;
    
    // Bind thread to specific CPU
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu, &cpuset);
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
    
    // Allocate memory on same NUMA node as CPU
    int node = numa_node_of_cpu(cpu);
    void *buffer = numa_alloc_onnode(BUFFER_SIZE, node);
    
    // Now memory accesses are local (fast)
    process_data(buffer);
    
    numa_free(buffer, BUFFER_SIZE);
    return NULL;
}
```

**Check NUMA topology**:

```bash
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 32768 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 32768 MB
node distances:
node   0   1 
  0:  10  21   # Local=10, Remote=21 (2.1x cost)
  1:  21  10
```

**Linux Kernel NUMA Balancing** (`kernel/sched/fair.c`):

```c
// Simplified NUMA task placement
void task_numa_fault(struct task_struct *p, int node) {
    // Track which NUMA nodes this task accesses
    p->numa_faults[node]++;
    
    // If task frequently accesses memory on node X,
    // but runs on node Y, migrate task to node X
    int preferred_node = find_most_accessed_node(p);
    
    if (preferred_node != task_node(p)) {
        migrate_task_to(p, preferred_node);
    }
}
```

### 2. CPU Affinity - Pinning Threads

**Why pin threads?**

1. **Cache locality**: Thread stays on same core, cache stays hot
2. **Predictable performance**: No migration overhead
3. **NUMA optimization**: Keep thread near its memory

**Linux API**:

```c
#include <sched.h>

void pin_thread_to_core(int core_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(core_id, &cpuset);
    
    pthread_t current_thread = pthread_self();
    int result = pthread_setaffinity_np(current_thread, 
                                       sizeof(cpu_set_t), 
                                       &cpuset);
    
    if (result != 0) {
        perror("pthread_setaffinity_np");
    }
}

// Usage
void* worker(void* arg) {
    int core = *(int*)arg;
    pin_thread_to_core(core);
    
    // This thread now only runs on specified core
    do_work();
    return NULL;
}
```

**Real-world example** - High-frequency trading:

```c
// Pin critical threads to specific cores
// Core 0-3: Trading logic (latency-critical)
// Core 4-7: Market data (high throughput)
// Core 8-15: Analytics (background)

void setup_trading_threads() {
    pthread_t threads[16];
    
    // Trading threads - isolated cores
    for (int i = 0; i < 4; i++) {
        int *core = malloc(sizeof(int));
        *core = i;
        pthread_create(&threads[i], NULL, trading_logic, core);
    }
    
    // Market data threads
    for (int i = 4; i < 8; i++) {
        int *core = malloc(sizeof(int));
        *core = i;
        pthread_create(&threads[i], NULL, market_data, core);
    }
    
    // Analytics threads
    for (int i = 8; i < 16; i++) {
        int *core = malloc(sizeof(int));
        *core = i;
        pthread_create(&threads[i], NULL, analytics, core);
    }
}
```

**Kernel-level control** (`/proc/irq/*/smp_affinity`):

```bash
# Isolate cores 0-3 from interrupts (for latency-critical threads)
echo 0-3 > /sys/devices/system/cpu/isolated
echo fff0 > /proc/irq/default_smp_affinity  # Only cores 4-15 handle IRQs

# Boot parameter to fully isolate cores
# isolcpus=0-3 nohz_full=0-3 rcu_nocbs=0-3
```

### 3. Real-Time Scheduling - Guaranteed Response Times

**Linux RT Priorities**:

```c
#include <sched.h>

void make_thread_realtime(pthread_t thread, int priority) {
    struct sched_param param;
    param.sched_priority = priority;  // 1-99 (higher = more priority)
    
    // SCHED_FIFO: Run until blocked or preempted by higher priority
    int result = pthread_setschedparam(thread, SCHED_FIFO, &param);
    
    if (result != 0) {
        perror("pthread_setschedparam");
    }
}

// Usage
void* critical_thread(void* arg) {
    // This thread now has real-time priority
    // It will preempt all normal threads
    
    while (1) {
        wait_for_event();
        process_event();  // Guaranteed to run within microseconds
    }
}

int main() {
    pthread_t thread;
    pthread_create(&thread, NULL, critical_thread, NULL);
    
    make_thread_realtime(thread, 80);  // High RT priority
    
    pthread_join(thread, NULL);
}
```

**RT Scheduling Policies**:

```c
// SCHED_FIFO: First-in-first-out
// - Runs until it blocks or yields
// - Preempts lower priority threads
// - No time slicing among same-priority threads

// SCHED_RR: Round-robin
// - Like FIFO but time-slices among same priority

// SCHED_DEADLINE: Deadline scheduling
// - Specify deadline, period, runtime
// - Kernel guarantees execution before deadline
struct sched_attr {
    uint32_t size;
    uint32_t sched_policy;      // SCHED_DEADLINE
    uint64_t sched_flags;
    
    int32_t  sched_nice;
    uint32_t sched_priority;
    
    uint64_t sched_runtime;     // Execution time (ns)
    uint64_t sched_deadline;    // Relative deadline (ns)
    uint64_t sched_period;      // Period (ns)
};

// Example: Task needs 5ms every 10ms
attr.sched_runtime = 5000000;   // 5ms
attr.sched_deadline = 10000000; // 10ms
attr.sched_period = 10000000;   // 10ms

sched_setattr(0, &attr, 0);
```

**Real-time example** - Industrial control:

```c
void* motor_control_thread(void* arg) {
    struct timespec next_activation;
    clock_gettime(CLOCK_MONOTONIC, &next_activation);
    
    while (1) {
        // Read sensors
        int position = read_encoder();
        int velocity = calculate_velocity();
        
        // Compute control signal (PID controller)
        int control = pid_compute(position, velocity);
        
        // Send to motor
        set_motor_pwm(control);
        
        // Wait for next 1ms cycle (deterministic!)
        next_activation.tv_nsec += 1000000;  // +1ms
        if (next_activation.tv_nsec >= 1000000000) {
            next_activation.tv_sec++;
            next_activation.tv_nsec -= 1000000000;
        }
        
        clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, 
                       &next_activation, NULL);
    }
}
```

### 4. Lock-Free Programming Deep Dive

**Memory Ordering - x86 vs ARM**:

**x86-64** (Total Store Order - TSO):
- Stores appear in order to other cores
- Relatively strong guarantees
- `LOCK` prefix for atomics

**ARM/ARM64** (Weak ordering):
- Stores can appear out of order
- Requires explicit barriers
- More opportunities for optimization but harder to reason about

**Example**: Dekker's Algorithm (mutual exclusion without atomic ops)

```c
// Thread 0
void lock_thread0() {
    flag[0] = 1;                    // I want the lock
    atomic_thread_fence(memory_order_seq_cst);  // ARM needs this!
    
    while (flag[1]) {               // Is thread 1 in critical section?
        if (turn != 0) {
            flag[0] = 0;
            while (turn != 0) {}    // Wait for my turn
            flag[0] = 1;
        }
    }
}

void unlock_thread0() {
    turn = 1;
    atomic_thread_fence(memory_order_seq_cst);
    flag[0] = 0;
}
```

**Without fences on ARM**, loads/stores can be reordered and break the algorithm!

**Compare-And-Swap (CAS) - The Building Block**:

**x86-64 Assembly**:

```assembly
; bool CAS(int *ptr, int expected, int new_value)
; Returns true if *ptr was expected (and updated it)

cas_x86:
    mov eax, esi                ; Load expected into EAX
    lock cmpxchg DWORD PTR [rdi], edx
    ; CMPXCHG atomically:
    ; - Compares EAX with [rdi]
    ; - If equal: ZF=1, [rdi] = edx
    ; - If not: ZF=0, EAX = [rdi]
    
    sete al                     ; Set AL = ZF (1 if success)
    ret
```

**ARM64 Assembly** (uses LL/SC - Load-Linked/Store-Conditional):

```assembly
; bool CAS(int *ptr, int expected, int new_value)

cas_arm64:
retry:
    ldaxr w3, [x0]              ; Load-Acquire Exclusive
    cmp w3, w1                  ; Compare with expected
    b.ne fail                   ; If not equal, fail
    
    stlxr w4, w2, [x0]          ; Store-Release Exclusive
    cbnz w4, retry              ; If store failed, retry
    
    mov w0, #1                  ; Success
    ret

fail:
    clrex                       ; Clear exclusive monitor
    mov w0, #0                  ; Failure
    ret
```

**Lock-Free Queue** (Michael-Scott Queue):

```c
typedef struct node {
    void *data;
    atomic_uintptr_t next;  // Atomic pointer to next node
} node_t;

typedef struct {
    atomic_uintptr_t head;
    atomic_uintptr_t tail;
} queue_t;

void enqueue(queue_t *q, void *data) {
    node_t *node = malloc(sizeof(node_t));
    node->data = data;
    atomic_store(&node->next, 0);
    
    while (1) {
        node_t *tail = (node_t*)atomic_load(&q->tail);
        node_t *next = (node_t*)atomic_load(&tail->next);
        
        // Check if tail is still the last node
        if (tail == (node_t*)atomic_load(&q->tail)) {
            if (next == NULL) {
                // Try to link new node at tail
                if (atomic_compare_exchange_weak(&tail->next, 
                                                 (uintptr_t*)&next,
                                                 (uintptr_t)node)) {
                    // Success! Try to swing tail to new node
                    atomic_compare_exchange_weak(&q->tail,
                                                 (uintptr_t*)&tail,
                                                 (uintptr_t)node);
                    return;
                }
            } else {
                // Tail is lagging, try to advance it
                atomic_compare_exchange_weak(&q->tail,
                                             (uintptr_t*)&tail,
                                             (uintptr_t)next);
            }
        }
    }
}

void* dequeue(queue_t *q) {
    while (1) {
        node_t *head = (node_t*)atomic_load(&q->head);
        node_t *tail = (node_t*)atomic_load(&q->tail);
        node_t *next = (node_t*)atomic_load(&head->next);
        
        if (head == (node_t*)atomic_load(&q->head)) {
            if (head == tail) {
                if (next == NULL) {
                    return NULL;  // Queue empty
                }
                // Tail lagging, try to advance
                atomic_compare_exchange_weak(&q->tail,
                                             (uintptr_t*)&tail,
                                             (uintptr_t)next);
            } else {
                void *data = next->data;
                
                // Try to swing head to next
                if (atomic_compare_exchange_weak(&q->head,
                                                 (uintptr_t*)&head,
                                                 (uintptr_t)next)) {
                    free(head);  // Safe to free old head
                    return data;
                }
            }
        }
    }
}
```

**Why lock-free is hard**:
1. ABA problem (solved with generation counters or hazard pointers)
2. Memory reclamation (when is it safe to free?)
3. Progress guarantees (starvation possible)
4. Subtle bugs (extremely hard to test/reproduce)

### 5. Hazard Pointers - Safe Memory Reclamation

**The Problem**: In lock-free structures, when can you safely free memory?

```c
// Thread A                     Thread B
node = head->next;              
                                free(head);  // Frees node!
data = node->data;              // CRASH! Use-after-free
```

**Solution**: Hazard pointers - mark what you're using

```c
#define MAX_THREADS 128
#define MAX_HAZARDS 4

atomic_uintptr_t hazard_pointers[MAX_THREADS][MAX_HAZARDS];

void* acquire_hazard(atomic_uintptr_t *ptr, int tid, int hazard_idx) {
    void *p;
    
    do {
        p = (void*)atomic_load(ptr);
        atomic_store(&hazard_pointers[tid][hazard_idx], (uintptr_t)p);
        // Double-check p hasn't changed (ABA protection)
    } while (p != (void*)atomic_load(ptr));
    
    return p;
}

void release_hazard(int tid, int hazard_idx) {
    atomic_store(&hazard_pointers[tid][hazard_idx], 0);
}

void safe_free(void *ptr) {
    // Add to retired list
    add_to_retired_list(ptr);
    
    // Periodically scan hazard pointers
    if (retired_list_size() > THRESHOLD) {
        for (each retired node) {
            bool in_use = false;
            
            // Check if any thread has a hazard pointer to this node
            for (int t = 0; t < MAX_THREADS; t++) {
                for (int h = 0; h < MAX_HAZARDS; h++) {
                    if (hazard_pointers[t][h] == (uintptr_t)node) {
                        in_use = true;
                        break;
                    }
                }
            }
            
            if (!in_use) {
                free(node);  // Safe to reclaim
            }
        }
    }
}

// Usage in lock-free data structure
void* dequeue_safe(queue_t *q) {
    int tid = get_thread_id();
    
    node_t *node = acquire_hazard(&q->head, tid, 0);
    // ... dequeue logic ...
    release_hazard(tid, 0);
    
    safe_free(old_head);  // Won't be freed if still in use
}
```

### 6. Read-Copy-Update (RCU) - Linux Kernel's Secret Weapon

**Concept**: Readers never block writers (or each other)

```c
// Kernel example: routing table lookup

// Reader (no locks!)
rcu_read_lock();  // Just marks section, no actual lock
route = lookup_route(dest_ip);
if (route) {
    use_route(route);  // Safe - won't be freed during this section
}
rcu_read_unlock();

// Writer
new_table = copy_routing_table();
update_route_in_new_table(new_table);

rcu_assign_pointer(routing_table, new_table);  // Atomic swap

synchronize_rcu();  // Wait for all readers to finish
free(old_table);    // Now safe to free
```

**How synchronize_rcu() works**:

```c
// Simplified from kernel/rcu/tree.c

void synchronize_rcu(void) {
    // Wait for grace period - ensures all CPUs have passed
    // through a quiescent state (not in RCU read-side critical section)
    
    // Implementation:
    // 1. Mark start of grace period
    // 2. Wait for all CPUs to context switch or go idle
    // 3. Grace period ends - all old readers gone
}
```

**Why RCU wins**:
- Readers have zero overhead (no atomics, no barriers)
- Scalability: unlimited concurrent readers
- Used extensively in kernel (networking, VFS, scheduler)

**Userspace RCU** (liburcu):

```c
#include <urcu.h>

struct data {
    int value;
    struct rcu_head rcu;
};

struct data *global_ptr;

// Reader
void reader() {
    rcu_read_lock();
    
    struct data *p = rcu_dereference(global_ptr);
    if (p) {
        printf("Value: %d\n", p->value);
    }
    
    rcu_read_unlock();
}

// Writer
void writer() {
    struct data *new_data = malloc(sizeof(*new_data));
    new_data->value = 42;
    
    struct data *old = rcu_xchg_pointer(&global_ptr, new_data);
    
    // Free old after grace period
    if (old) {
        call_rcu(&old->rcu, free_data);
    }
}
```

## Part XVII: Debugging Multi-Threaded Programs

### 1. ThreadSanitizer (TSan) - Detect Data Races

**Compile with TSan**:

```bash
gcc -fsanitize=thread -g -O1 program.c -o program
./program
```

**Example output**:

```
==================
WARNING: ThreadSanitizer: data race (pid=1234)
  Write of size 4 at 0x7fff12345678 by thread T1:
    #0 increment /home/user/test.c:45
    #1 worker /home/user/test.c:67

  Previous write of size 4 at 0x7fff12345678 by thread T2:
    #0 increment /home/user/test.c:45
    #2 worker /home/user/test.c:67

  Location is global 'counter' of size 4 at 0x7fff12345678
==================
```

**How TSan works**:
- Instruments every memory access
- Tracks happens-before relationships
- Detects conflicting accesses not ordered by synchronization

### 2. Helgrind (Valgrind) - Detect Lock Issues

```bash
valgrind --tool=helgrind ./program
```

**Detects**:
- Data races
- Lock ordering violations (potential deadlocks)
- Incorrect API usage

**Example output**:

```
==1234== Possible data race during write of size 4 at 0x12345678
==1234==    at 0x401234: increment (test.c:45)
==1234==    by 0x402345: worker (test.c:67)
==1234==  This conflicts with a previous write of size 4 by thread #2
==1234==    at 0x401234: increment (test.c:45)
==1234==    by 0x402345: worker (test.c:67)
```

### 3. GDB - Debugging Threads

```bash
gdb ./program
```

**Useful commands**:

```gdb
# List all threads
(gdb) info threads
  Id   Target Id         Frame
* 1    Thread 0x7ffff7fc1740 (LWP 1234) main () at test.c:100
  2    Thread 0x7ffff6fc0700 (LWP 1235) worker () at test.c:45
  3    Thread 0x7ffff5fbf700 (LWP 1236) worker () at test.c:45

# Switch to thread
(gdb) thread 2
[Switching to thread 2]

# Show backtrace of all threads
(gdb) thread apply all bt

# Set breakpoint only for specific thread
(gdb) break test.c:45 thread 2

# Show thread-local variables
(gdb) info locals
```

**Debugging deadlock**:

```gdb
# All threads are blocked? Check what they're waiting on:
(gdb) thread apply all bt

# Thread 1:
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  pthread_mutex_lock (mutex=0x601060 <lock_a>) at pthread_mutex_lock.c:80

# Thread 2:
#0  __lll_lock_wait () at ../sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  pthread_mutex_lock (mutex=0x601080 <lock_b>) at pthread_mutex_lock.c:80

# Deadlock! Thread 1 has lock_a, wants lock_b
#          Thread 2 has lock_b, wants lock_a
```

### 4. DTrace/SystemTap - System-Wide Tracing

**DTrace example** (Solaris/macOS):

```d
# Trace all pthread_mutex_lock calls
dtrace -n 'pid$target::pthread_mutex_lock:entry { 
    @locks[ustack()] = count(); 
}' -p <PID>

# Measure lock hold times
dtrace -n '
    pid$target::pthread_mutex_lock:return {
        self->ts = timestamp;
    }
    
    pid$target::pthread_mutex_unlock:entry {
        @hold_time = quantize(timestamp - self->ts);
    }
' -p <PID>
```

**SystemTap example** (Linux):

```stap
# Monitor context switches
probe scheduler.ctxswitch {
    printf("%d: %s -> %s\n", cpu(), 
           prev_task_name, next_task_name);
}

# Track futex operations
probe kernel.function("futex_wait") {
    printf("Thread %d waiting on futex %p\n", tid(), $uaddr);
}
```

### 5. Custom Deadlock Detection

```c
// Simple lock ordering check
#define MAX_LOCKS 100

typedef struct {
    pthread_mutex_t *lock;
    const char *file;
    int line;
} lock_info_t;

__thread lock_info_t held_locks[MAX_LOCKS];
__thread int num_held_locks = 0;

void debug_mutex_lock(pthread_mutex_t *lock, const char *file, int line) {
    // Check if we already hold a "later" lock (potential deadlock)
    for (int i = 0; i < num_held_locks; i++) {
        if (held_locks[i].lock > lock) {
            fprintf(stderr, "LOCK ORDER VIOLATION!\n");
            fprintf(stderr, "  Holding: %p (%s:%d)\n",
                   held_locks[i].lock, held_locks[i].file, 
                   held_locks[i].line);
            fprintf(stderr, "  Acquiring: %p (%s:%d)\n",
                   lock, file, line);
            abort();
        }
    }
    
    pthread_mutex_lock(lock);
    
    held_locks[num_held_locks].lock = lock;
    held_locks[num_held_locks].file = file;
    held_locks[num_held_locks].line = line;
    num_held_locks++;
}

#define LOCK(m) debug_mutex_lock(&(m), __FILE__, __LINE__)
```

## Part XVIII: Performance Optimization Strategies

### Strategy 1: Reduce Sharing

**Before** (false sharing):

```c
struct counter {
    int count;
} counters[NUM_THREADS];  // All in same cache line!

void* worker(void* arg) {
    int id = *(int*)arg;
    
    for (int i = 0; i < 1000000; i++) {
        counters[id].count++;  // Cache line ping-pong!
    }
}
```

**After** (cache-line aligned):

```c
struct counter {
    int count;
    char padding[60];  // Force to separate cache lines
} __attribute__((aligned(64))) counters[NUM_THREADS];

// Or use C11 _Alignas
struct counter {
    _Alignas(64) int count;
} counters[NUM_THREADS];
```

**Benchmark**:
```
False sharing:    5.2 seconds
Cache-aligned:    0.8 seconds (6.5x faster!)
```

### Strategy 2: Batching

**Before** (lock per operation):

```c
for (int i = 0; i < 1000000; i++) {
    pthread_mutex_lock(&lock);
    queue_add(item[i]);
    pthread_mutex_unlock(&lock);
}
```

**After** (batch operations):

```c
item_t batch[1000];
for (int i = 0; i < 1000000; i += 1000) {
    memcpy(batch, &item[i], sizeof(batch));
    
    pthread_mutex_lock(&lock);
    for (int j = 0; j < 1000; j++) {
        queue_add(batch[j]);
    }
    pthread_mutex_unlock(&lock);
}
```

**Result**: 1000x fewer lock operations

### Strategy 3: Reader-Writer Lock for Read-Heavy Workloads

```c
// 90% reads, 10% writes

pthread_rwlock_t rwlock;

// Many readers can run simultaneously
void* reader(void* arg) {
    pthread_rwlock_rdlock(&rwlock);
    // Read data (no contention with other readers)
    pthread_rwlock_unlock(&rwlock);
}

// Writers get exclusive access
void* writer(void* arg) {
    pthread_rwlock_wrlock(&rwlock);
    // Modify data (blocks all readers and writers)
    pthread_rwlock_unlock(&rwlock);
}
```

### Strategy 4: Lock-Free Single-Producer Single-Consumer Queue

```c
// Ring buffer - no locks needed!
typedef struct {
    void *buffer[SIZE];
    atomic_size_t head;  // Written by consumer only
    atomic_size_t tail;  // Written by producer only
    char padding[64 - 2*sizeof(atomic_size_t)];  // Prevent false sharing
} spsc_queue_t;

bool enqueue(spsc_queue_t *q, void *item) {
    size_t tail = atomic_load_explicit(&q->tail, memory_order_relaxed);
    size_t next_tail = (tail + 1) % SIZE;
    
    if (next_tail == atomic_load_explicit(&q->head, memory_order_acquire)) {
        return false;  // Queue full
    }
    
    q->buffer[tail] = item;
    atomic_store_explicit(&q->tail, next_tail, memory_order_release);
    return true;
}

void* dequeue(spsc_queue_t *q) {
    size_t head = atomic_load_explicit(&q->head, memory_order_relaxed);
    
    if (head == atomic_load_explicit(&q->tail, memory_order_acquire)) {
        return NULL;  // Queue empty
    }
    
    void *item = q->buffer[head];
    atomic_store_explicit(&q->head, (head + 1) % SIZE, memory_order_release);
    return item;
}
```

**Why it works**:
- Only producer writes `tail`
- Only consumer writes `head`
- No contention, no locks needed!
- Used in audio processing, game engines

---

## Final Thoughts: The Art and Science of Threading

Threading is a powerful tool, but it's not a magic bullet. The key insights:

**1. Measure first, optimize second**
- Don't assume threading makes things faster
- Profile to find actual bottlenecks
- Sometimes single-threaded + better algorithm wins

**2. Understand your workload**
- CPU-bound: threads = cores
- I/O-bound: threads >> cores (or use async I/O)
- Mixed: measure and tune

**3. Design for concurrency from the start**
- Retrofit is painful
- Think about data ownership
- Minimize shared state
- Choose appropriate synchronization primitives

**4. Simplicity > Performance (usually)**
- Start with coarse-grained locking
- Optimize only hot paths
- Complex lock-free algorithms are rarely worth it

**5. The hardware matters**
- Cache lines (64 bytes on x86)
- Memory ordering (x86 vs ARM)
- NUMA topology on servers
- Hyperthreading doesn't double performance

**6. Testing is hard**
- Race conditions are non-deterministic
- Use sanitizers (TSan, Helgrind)
- Stress test with different thread counts
- Think about edge cases

---

## Part XIX: Complete Working Examples

Let me provide several complete, production-quality examples that demonstrate the concepts:

### Example 1: High-Performance HTTP Server (Thread Pool)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>

#define PORT 8080
#define THREAD_POOL_SIZE 8
#define QUEUE_SIZE 256

// Task queue
typedef struct {
    int client_fd;
} task_t;

typedef struct {
    task_t tasks[QUEUE_SIZE];
    int front, rear, count;
    
    pthread_mutex_t lock;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    
    int shutdown;
} task_queue_t;

// Thread pool
typedef struct {
    pthread_t threads[THREAD_POOL_SIZE];
    task_queue_t queue;
} thread_pool_t;

// Initialize queue
void queue_init(task_queue_t *q) {
    q->front = q->rear = q->count = 0;
    q->shutdown = 0;
    pthread_mutex_init(&q->lock, NULL);
    pthread_cond_init(&q->not_empty, NULL);
    pthread_cond_init(&q->not_full, NULL);
}

// Add task to queue (blocking if full)
void queue_push(task_queue_t *q, task_t task) {
    pthread_mutex_lock(&q->lock);
    
    // Wait while queue is full
    while (q->count == QUEUE_SIZE && !q->shutdown) {
        pthread_cond_wait(&q->not_full, &q->lock);
    }
    
    if (q->shutdown) {
        pthread_mutex_unlock(&q->lock);
        return;
    }
    
    q->tasks[q->rear] = task;
    q->rear = (q->rear + 1) % QUEUE_SIZE;
    q->count++;
    
    pthread_cond_signal(&q->not_empty);
    pthread_mutex_unlock(&q->lock);
}

// Get task from queue (blocking if empty)
int queue_pop(task_queue_t *q, task_t *task) {
    pthread_mutex_lock(&q->lock);
    
    // Wait while queue is empty
    while (q->count == 0 && !q->shutdown) {
        pthread_cond_wait(&q->not_empty, &q->lock);
    }
    
    if (q->shutdown && q->count == 0) {
        pthread_mutex_unlock(&q->lock);
        return 0;  // Shutdown signal
    }
    
    *task = q->tasks[q->front];
    q->front = (q->front + 1) % QUEUE_SIZE;
    q->count--;
    
    pthread_cond_signal(&q->not_full);
    pthread_mutex_unlock(&q->lock);
    return 1;
}

// Handle HTTP request
void handle_request(int client_fd) {
    char buffer[4096];
    ssize_t bytes_read = read(client_fd, buffer, sizeof(buffer) - 1);
    
    if (bytes_read <= 0) {
        close(client_fd);
        return;
    }
    
    buffer[bytes_read] = '\0';
    
    // Parse request (simplified)
    char method[16], path[256];
    sscanf(buffer, "%s %s", method, path);
    
    // Generate response
    const char *response = 
        "HTTP/1.1 200 OK\r\n"
        "Content-Type: text/html\r\n"
        "Content-Length: 51\r\n"
        "Connection: close\r\n"
        "\r\n"
        "<html><body><h1>Hello from Thread Pool!</h1></body></html>";
    
    write(client_fd, response, strlen(response));
    close(client_fd);
}

// Worker thread function
void* worker_thread(void* arg) {
    thread_pool_t *pool = (thread_pool_t*)arg;
    
    printf("Worker thread %lu started\n", pthread_self());
    
    while (1) {
        task_t task;
        
        if (!queue_pop(&pool->queue, &task)) {
            break;  // Shutdown
        }
        
        handle_request(task.client_fd);
    }
    
    printf("Worker thread %lu exiting\n", pthread_self());
    return NULL;
}

// Initialize thread pool
void pool_init(thread_pool_t *pool) {
    queue_init(&pool->queue);
    
    for (int i = 0; i < THREAD_POOL_SIZE; i++) {
        pthread_create(&pool->threads[i], NULL, worker_thread, pool);
    }
}

// Shutdown thread pool
void pool_shutdown(thread_pool_t *pool) {
    pthread_mutex_lock(&pool->queue.lock);
    pool->queue.shutdown = 1;
    pthread_cond_broadcast(&pool->queue.not_empty);
    pthread_mutex_unlock(&pool->queue.lock);
    
    for (int i = 0; i < THREAD_POOL_SIZE; i++) {
        pthread_join(pool->threads[i], NULL);
    }
}

// Main server loop
int main() {
    int server_fd, client_fd;
    struct sockaddr_in address;
    int opt = 1;
    int addrlen = sizeof(address);
    
    // Create socket
    if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }
    
    // Set socket options
    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT,
                   &opt, sizeof(opt))) {
        perror("setsockopt");
        exit(EXIT_FAILURE);
    }
    
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = INADDR_ANY;
    address.sin_port = htons(PORT);
    
    // Bind
    if (bind(server_fd, (struct sockaddr*)&address, sizeof(address)) < 0) {
        perror("bind failed");
        exit(EXIT_FAILURE);
    }
    
    // Listen
    if (listen(server_fd, 10) < 0) {
        perror("listen");
        exit(EXIT_FAILURE);
    }
    
    printf("Server listening on port %d\n", PORT);
    printf("Thread pool size: %d\n", THREAD_POOL_SIZE);
    
    // Initialize thread pool
    thread_pool_t pool;
    pool_init(&pool);
    
    // Accept loop
    while (1) {
        if ((client_fd = accept(server_fd, (struct sockaddr*)&address,
                               (socklen_t*)&addrlen)) < 0) {
            if (errno == EINTR) continue;
            perror("accept");
            continue;
        }
        
        // Add to task queue
        task_t task = { .client_fd = client_fd };
        queue_push(&pool.queue, task);
    }
    
    pool_shutdown(&pool);
    close(server_fd);
    return 0;
}
```

**Compile and run**:
```bash
gcc -pthread -O2 -o server http_server.c
./server

# Test with:
# curl http://localhost:8080/
# ab -n 10000 -c 100 http://localhost:8080/  # Apache Bench
```

---

### Example 2: Parallel Matrix Multiplication (Work Stealing)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <stdatomic.h>
#include <time.h>

#define N 1024  // Matrix size
#define NUM_THREADS 8

typedef struct {
    double *A;
    double *B;
    double *C;
    int n;
} matrix_data_t;

// Work queue (atomic counter for work stealing)
typedef struct {
    atomic_int next_row;
    int total_rows;
    matrix_data_t *data;
} work_queue_t;

// Worker thread
void* matrix_multiply_worker(void* arg) {
    work_queue_t *queue = (work_queue_t*)arg;
    matrix_data_t *data = queue->data;
    
    while (1) {
        // Atomically get next row to process
        int row = atomic_fetch_add(&queue->next_row, 1);
        
        if (row >= queue->total_rows) {
            break;  // No more work
        }
        
        // Compute row
        for (int j = 0; j < data->n; j++) {
            double sum = 0.0;
            for (int k = 0; k < data->n; k++) {
                sum += data->A[row * data->n + k] * data->B[k * data->n + j];
            }
            data->C[row * data->n + j] = sum;
        }
    }
    
    return NULL;
}

// Parallel matrix multiplication
void matrix_multiply_parallel(double *A, double *B, double *C, int n) {
    pthread_t threads[NUM_THREADS];
    work_queue_t queue;
    matrix_data_t data = { .A = A, .B = B, .C = C, .n = n };
    
    atomic_init(&queue.next_row, 0);
    queue.total_rows = n;
    queue.data = &data;
    
    // Create threads
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, matrix_multiply_worker, &queue);
    }
    
    // Wait for completion
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
}

// Initialize random matrix
void matrix_random(double *M, int n) {
    for (int i = 0; i < n * n; i++) {
        M[i] = (double)rand() / RAND_MAX;
    }
}

// Verify result
void matrix_multiply_serial(double *A, double *B, double *C, int n) {
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            double sum = 0.0;
            for (int k = 0; k < n; k++) {
                sum += A[i * n + k] * B[k * n + j];
            }
            C[i * n + j] = sum;
        }
    }
}

double get_time() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec + ts.tv_nsec / 1e9;
}

int main() {
    double *A = malloc(N * N * sizeof(double));
    double *B = malloc(N * N * sizeof(double));
    double *C = malloc(N * N * sizeof(double));
    double *C_ref = malloc(N * N * sizeof(double));
    
    srand(42);
    matrix_random(A, N);
    matrix_random(B, N);
    
    // Serial version (for comparison)
    printf("Running serial multiplication...\n");
    double start = get_time();
    matrix_multiply_serial(A, B, C_ref, N);
    double serial_time = get_time() - start;
    printf("Serial time: %.3f seconds\n", serial_time);
    
    // Parallel version
    printf("Running parallel multiplication (%d threads)...\n", NUM_THREADS);
    start = get_time();
    matrix_multiply_parallel(A, B, C, N);
    double parallel_time = get_time() - start;
    printf("Parallel time: %.3f seconds\n", parallel_time);
    printf("Speedup: %.2fx\n", serial_time / parallel_time);
    
    // Verify correctness
    double max_error = 0.0;
    for (int i = 0; i < N * N; i++) {
        double error = fabs(C[i] - C_ref[i]);
        if (error > max_error) max_error = error;
    }
    printf("Max error: %e\n", max_error);
    
    free(A);
    free(B);
    free(C);
    free(C_ref);
    
    return 0;
}
```

**Expected output**:
```
Running serial multiplication...
Serial time: 4.521 seconds
Running parallel multiplication (8 threads)...
Parallel time: 0.632 seconds
Speedup: 7.15x
Max error: 1.234e-13
```

---

### Example 3: Producer-Consumer with Multiple Producers and Consumers

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>
#include <time.h>

#define BUFFER_SIZE 10
#define NUM_PRODUCERS 3
#define NUM_CONSUMERS 4
#define ITEMS_PER_PRODUCER 20

typedef struct {
    int buffer[BUFFER_SIZE];
    int count;
    int in;   // Producer index
    int out;  // Consumer index
    
    pthread_mutex_t mutex;
    pthread_cond_t not_empty;
    pthread_cond_t not_full;
    
    int producers_done;
    int total_produced;
    int total_consumed;
} shared_buffer_t;

shared_buffer_t shared;

void buffer_init(shared_buffer_t *buf) {
    buf->count = 0;
    buf->in = 0;
    buf->out = 0;
    buf->producers_done = 0;
    buf->total_produced = 0;
    buf->total_consumed = 0;
    
    pthread_mutex_init(&buf->mutex, NULL);
    pthread_cond_init(&buf->not_empty, NULL);
    pthread_cond_init(&buf->not_full, NULL);
}

void produce(shared_buffer_t *buf, int item) {
    pthread_mutex_lock(&buf->mutex);
    
    // Wait while buffer is full
    while (buf->count == BUFFER_SIZE) {
        pthread_cond_wait(&buf->not_full, &buf->mutex);
    }
    
    // Add item
    buf->buffer[buf->in] = item;
    buf->in = (buf->in + 1) % BUFFER_SIZE;
    buf->count++;
    buf->total_produced++;
    
    printf("[P%lu] Produced: %d (buffer: %d/%d)\n", 
           pthread_self() % 1000, item, buf->count, BUFFER_SIZE);
    
    // Signal consumers
    pthread_cond_signal(&buf->not_empty);
    pthread_mutex_unlock(&buf->mutex);
}

int consume(shared_buffer_t *buf, int *item) {
    pthread_mutex_lock(&buf->mutex);
    
    // Wait while buffer is empty and producers still active
    while (buf->count == 0 && buf->producers_done < NUM_PRODUCERS) {
        pthread_cond_wait(&buf->not_empty, &buf->mutex);
    }
    
    // Check if we're done
    if (buf->count == 0 && buf->producers_done == NUM_PRODUCERS) {
        pthread_mutex_unlock(&buf->mutex);
        return 0;  // No more items
    }
    
    // Get item
    *item = buf->buffer[buf->out];
    buf->out = (buf->out + 1) % BUFFER_SIZE;
    buf->count--;
    buf->total_consumed++;
    
    printf("[C%lu] Consumed: %d (buffer: %d/%d)\n", 
           pthread_self() % 1000, *item, buf->count, BUFFER_SIZE);
    
    // Signal producers
    pthread_cond_signal(&buf->not_full);
    pthread_mutex_unlock(&buf->mutex);
    return 1;
}

void* producer_thread(void* arg) {
    int id = *(int*)arg;
    free(arg);
    
    for (int i = 0; i < ITEMS_PER_PRODUCER; i++) {
        int item = id * 1000 + i;
        produce(&shared, item);
        
        // Simulate work
        usleep(rand() % 100000);
    }
    
    // Signal completion
    pthread_mutex_lock(&shared.mutex);
    shared.producers_done++;
    pthread_cond_broadcast(&shared.not_empty);  // Wake all consumers
    pthread_mutex_unlock(&shared.mutex);
    
    printf("[P%lu] Producer %d finished\n", pthread_self() % 1000, id);
    return NULL;
}

void* consumer_thread(void* arg) {
    int id = *(int*)arg;
    free(arg);
    
    int item;
    int count = 0;
    
    while (consume(&shared, &item)) {
        count++;
        
        // Simulate processing
        usleep(rand() % 150000);
    }
    
    printf("[C%lu] Consumer %d finished (consumed %d items)\n", 
           pthread_self() % 1000, id, count);
    return NULL;
}

int main() {
    pthread_t producers[NUM_PRODUCERS];
    pthread_t consumers[NUM_CONSUMERS];
    
    srand(time(NULL));
    buffer_init(&shared);
    
    printf("Starting %d producers and %d consumers\n", 
           NUM_PRODUCERS, NUM_CONSUMERS);
    printf("Buffer size: %d\n", BUFFER_SIZE);
    printf("Items per producer: %d\n\n", ITEMS_PER_PRODUCER);
    
    // Create producers
    for (int i = 0; i < NUM_PRODUCERS; i++) {
        int *id = malloc(sizeof(int));
        *id = i;
        pthread_create(&producers[i], NULL, producer_thread, id);
    }
    
    // Create consumers
    for (int i = 0; i < NUM_CONSUMERS; i++) {
        int *id = malloc(sizeof(int));
        *id = i;
        pthread_create(&consumers[i], NULL, consumer_thread, id);
    }
    
    // Wait for producers
    for (int i = 0; i < NUM_PRODUCERS; i++) {
        pthread_join(producers[i], NULL);
    }
    
    // Wait for consumers
    for (int i = 0; i < NUM_CONSUMERS; i++) {
        pthread_join(consumers[i], NULL);
    }
    
    printf("\n=== Final Statistics ===\n");
    printf("Total produced: %d\n", shared.total_produced);
    printf("Total consumed: %d\n", shared.total_consumed);
    printf("Expected: %d\n", NUM_PRODUCERS * ITEMS_PER_PRODUCER);
    
    if (shared.total_produced == shared.total_consumed &&
        shared.total_consumed == NUM_PRODUCERS * ITEMS_PER_PRODUCER) {
        printf("✓ SUCCESS: All items accounted for!\n");
    } else {
        printf("✗ ERROR: Mismatch detected!\n");
    }
    
    pthread_mutex_destroy(&shared.mutex);
    pthread_cond_destroy(&shared.not_empty);
    pthread_cond_destroy(&shared.not_full);
    
    return 0;
}
```

---

### Example 4: Barrier Synchronization (Parallel Phases)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

#define NUM_THREADS 8
#define NUM_ITERATIONS 5

typedef struct {
    int count;
    int total;
    pthread_mutex_t mutex;
    pthread_cond_t cv;
} barrier_t;

void barrier_init(barrier_t *barrier, int count) {
    barrier->count = count;
    barrier->total = count;
    pthread_mutex_init(&barrier->mutex, NULL);
    pthread_cond_init(&barrier->cv, NULL);
}

void barrier_wait(barrier_t *barrier) {
    pthread_mutex_lock(&barrier->mutex);
    
    barrier->count--;
    
    if (barrier->count == 0) {
        // Last thread - wake everyone
        barrier->count = barrier->total;
        pthread_cond_broadcast(&barrier->cv);
    } else {
        // Wait for others
        while (barrier->count != barrier->total) {
            pthread_cond_wait(&barrier->cv, &barrier->mutex);
        }
    }
    
    pthread_mutex_unlock(&barrier->mutex);
}

barrier_t barrier;

void* worker_thread(void* arg) {
    int id = *(int*)arg;
    free(arg);
    
    for (int iter = 0; iter < NUM_ITERATIONS; iter++) {
        // Phase 1: Computation
        printf("[Thread %d] Iteration %d: Computing...\n", id, iter);
        usleep((rand() % 500) * 1000);  // Simulate work
        
        printf("[Thread %d] Iteration %d: Reached barrier\n", id, iter);
        barrier_wait(&barrier);
        
        // All threads synchronized here!
        printf("[Thread %d] Iteration %d: Past barrier\n", id, iter);
        
        // Phase 2: Use results from phase 1
        usleep((rand() % 300) * 1000);
        
        barrier_wait(&barrier);
        printf("[Thread %d] Iteration %d: Complete\n", id, iter);
    }
    
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    
    barrier_init(&barrier, NUM_THREADS);
    
    printf("Starting %d threads with barrier synchronization\n\n", NUM_THREADS);
    
    for (int i = 0; i < NUM_THREADS; i++) {
        int *id = malloc(sizeof(int));
        *id = i;
        pthread_create(&threads[i], NULL, worker_thread, id);
    }
    
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    
    printf("\nAll threads completed\n");
    
    pthread_mutex_destroy(&barrier.mutex);
    pthread_cond_destroy(&barrier.cv);
    
    return 0;
}
```

---

## Part XX: Key Architectural Decisions - A Decision Tree

```
Need concurrency?
│
├─ YES → What type of workload?
│        │
│        ├─ CPU-bound computation
│        │  └─ Use threads = num_cores
│        │     Examples: matrix math, image processing, simulation
│        │
│        ├─ I/O-bound (network, disk)
│        │  ├─ Blocking I/O → Thread pool (threads >> cores)
│        │  └─ Can use async? → Event loop (epoll/io_uring)
│        │     Examples: web servers, databases
│        │
│        └─ Mixed workload
│           └─ Separate thread pools per workload type
│
└─ NO → Single-threaded is simpler and faster!
```

## Conclusion: The Complete Picture

Understanding threading requires knowledge at multiple layers:

**Layer 1: Hardware**
- CPU cores, cache hierarchy, memory bus
- Atomic instructions (LOCK, LL/SC)
- Cache coherency protocols (MESI)

**Layer 2: Operating System**
- Process vs thread model
- Scheduler algorithms (CFS, O(1))
- Context switching mechanics
- System calls (clone, futex)

**Layer 3: Runtime/Library**
- Thread creation (pthread_create → clone())
- Synchronization primitives (mutex, condvar, rwlock)
- Memory ordering (acquire/release semantics)

**Layer 4: Application**
- Design patterns (thread pool, producer-consumer)
- Lock-free algorithms (when appropriate)
- Performance optimization (reduce sharing, batching)

**Layer 5: Debugging/Profiling**
- Race detectors (TSan, Helgrind)
- Performance analysis (perf, DTrace)
- Deadlock detection

The journey from "let's add pthread_create()" to truly understanding multithreaded execution spans from silicon transistors to high-level application architecture. Each layer builds on the one below, and mastery requires understanding them all.

**Remember**: Threading is a tool, not a goal. The best code is often the simplest code that meets requirements. Thread only when necessary, measure always, and never underestimate the complexity you're introducing.