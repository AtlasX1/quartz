# Concurrency & Multithreading: Theoretical Foundations and Practical Patterns

## Table of Contents
1. Concurrency Fundamentals and Models
2. Process and Thread Architecture
3. Synchronization Primitives and Mechanisms
4. Lock-Free Data Structures and Algorithms
5. Event-Driven Architecture and Async Models
6. JavaScript/Node.js Concurrency Model
7. Common Concurrency Patterns and Anti-patterns
8. Production Systems and Performance Considerations

---

## 1. Concurrency Fundamentals and Models

### 1.1 Concurrency vs. Parallelism

These terms are often conflated but represent distinct concepts:

**Concurrency**: Multiple tasks make progress on a single processor by interleaving execution.

**Parallelism**: Multiple tasks execute simultaneously on multiple processors.

**Visual Distinction:**
```
Single Core (Concurrency only):
Task A: [===]   [===]   [===]
Task B:     [===]   [===]   [===]
        Time →

Multi-Core (True Parallelism):
Core 1: Task A [===========]
Core 2: Task B [===========]
        Time →
```

**Mathematical Formulation:**
$$\text{Throughput} = \frac{\text{Tasks completed}}{\text{Time elapsed}}$$

- Concurrency increases throughput on single processor (task switching)
- Parallelism increases throughput on multiple processors (simultaneous execution)
- True parallelism requires both concurrency and multiple processors

### 1.2 Concurrency Models

Different models address concurrent execution in fundamentally different ways:

#### 1.2.1 Shared Memory Model

Multiple threads share a common memory space, communicating through shared variables.

```
┌─────────────────────────────────┐
│      Shared Memory              │
│  ┌──────────────────────────┐   │
│  │ Global variables, heap   │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
    ▲         ▲         ▲
    │         │         │
┌───────┐ ┌───────┐ ┌───────┐
│ Thd 1 │ │ Thd 2 │ │ Thd 3 │
└───────┘ └───────┘ └───────┘
```

**Characteristics:**
- Implicit communication through shared state
- Low latency communication (same memory)
- Requires synchronization to prevent race conditions
- Difficult to reason about (interleaving complexity)

**Languages**: Java, C++, Python (with GIL relaxation), C#

#### 1.2.2 Message Passing Model

Threads/processes communicate exclusively through messages, no shared memory.

```
┌──────────┐           Message           ┌──────────┐
│ Thread A │──────────────────────────────→│ Thread B │
└──────────┘                              └──────────┘
     ▲                                           │
     │──────────────────────────────────────────┘
```

**Characteristics:**
- Explicit communication through messages
- Higher latency (message serialization/transport)
- No race conditions (isolated memory)
- Easier to reason about (explicit communication)

**Languages**: Erlang, Go (channels), Rust (message passing), Actor frameworks

#### 1.2.3 Hybrid Model

Combines shared memory for local data with message passing for inter-actor communication.

**Example - Actor Model:**
```
┌─────────────────────────┐
│ Actor A                 │
│ ┌────────────────────┐  │
│ │ Isolated state     │  │ Message
│ └────────────────────┘  │─────────────→ [Message Queue]
└─────────────────────────┘
                          ▼
                    ┌─────────────────────────┐
                    │ Actor B                 │
                    │ ┌────────────────────┐  │
                    │ │ Isolated state     │  │
                    │ └────────────────────┘  │
                    └─────────────────────────┘
```

**Benefits:**
- Isolation of state (prevents race conditions)
- Explicit communication (easier reasoning)
- Natural distribution (can run on different machines)

### 1.3 The Memory Model

A memory model specifies **which values a read operation can return** when multiple threads access memory concurrently.

**Without Memory Model:**
- Compiler could reorder reads/writes for optimization
- CPU could execute operations out of order
- Cache inconsistencies unpredictable
- Behavior undefined (nightmare for debugging)

**Java Memory Model (JMM) - Key Guarantees:**

**Visibility**: If Thread A writes to volatile variable $x$, then Thread B reading $x$ sees the value Thread A wrote (or later):
$$\text{Write}(x, v_1) \xrightarrow{\text{happens-before}} \text{Read}(x) \Rightarrow \text{returns } v_1 \text{ or later}$$

**Ordering**: Synchronization actions establish ordering:
```
Thread A:                          Thread B:
lock(mutex)                        
write(x, 5)                        
unlock(mutex) ─┐                   ┌─ lock(mutex)
               └─happens-before─→ read(x) // Returns 5
```

**Atomicity**: Certain operations are indivisible:
```
// Not atomic - can interleave:
int tmp = x;  // Load
x = tmp + 1;  // Increment
x;            // Store

// Atomic operation (hardware-level):
atomic_increment(&x);  // Indivisible
```

---

## 2. Process and Thread Architecture

### 2.1 Process Model

A **process** is an isolated execution context with:
- Private virtual address space (memory isolation)
- Process-wide resources (file descriptors, signals)
- Protected from other processes (OS enforces isolation)

**Process Overhead:**
$$\text{Context Switch Cost} = \text{CPU time} + \text{TLB flushes} + \text{Cache invalidation}$$

Measured: 1-100 microseconds depending on OS and architecture.

**Advantages:**
- Complete isolation (crash in one doesn't affect others)
- Protected memory (one can't corrupt another's memory)
- Easier reasoning (independent state)

**Disadvantages:**
- Heavy resource usage (each process ~10-100MB minimum)
- Expensive context switching
- Communication requires IPC (Inter-Process Communication)

### 2.2 Thread Model

A **thread** is a lightweight execution context sharing:
- Same virtual address space (shared memory)
- Same process resources
- Separate stack and registers per thread

**Thread Overhead:**
$$\text{Thread Creation} \approx 1-10\text{ms} \text{ (OS threads)}$$
$$\text{Context Switch Cost} \approx 1-10\text{ microseconds} \text{ (same address space)}$$

**Shared Thread State:**
```
┌─────────────────────────────────┐
│ Process (Shared Memory)          │
│ ┌──────────────────────────┐    │
│ │ Heap                     │    │
│ │ Global variables         │    │
│ │ Code (text segment)      │    │
│ └──────────────────────────┘    │
└─────────────────────────────────┘
    ▲                       ▲
    │                       │
┌─────────────────┐  ┌─────────────────┐
│ Thread 1 Stack  │  │ Thread 2 Stack  │
└─────────────────┘  └─────────────────┘
```

**Advantages:**
- Lightweight (cheap to create)
- Low context switch overhead
- Fast communication through shared memory
- Less resource usage per thread

**Disadvantages:**
- Race conditions (shared state without synchronization)
- Debugging complexity (interleaving)
- One crash affects entire process
- Stack overflow in one thread can corrupt other threads

### 2.3 Green Threads and User-Space Scheduling

Some runtimes implement **green threads** - threads scheduled in user space rather than kernel:

**Kernel Threads**: OS scheduler manages thread scheduling
- Preemptive (OS can interrupt at any time)
- True parallelism on multicore
- Context switch invokes OS
- Cost: High context switch overhead

**Green Threads**: Runtime scheduler manages thread scheduling
- Can be preemptive or cooperative
- Lightweight (millions possible)
- Context switch is function call
- Cost: Low but no true parallelism (single kernel thread)

**Example - Go Goroutines:**
```
Go Runtime:
  ┌────────────────────────────────┐
  │ Scheduler (M goroutines)       │
  │ ┌──────────────────────────┐   │
  │ │ Goroutine pool management│   │
  │ │ Work stealing scheduler  │   │
  │ └──────────────────────────┘   │
  └────────────────────────────────┘
         ▼ (maps to)
    Kernel Threads (N threads)
    Executing on (P processors)
    
Where M >> N > P (millions of goroutines on few threads)
```

---

## 3. Synchronization Primitives and Mechanisms

### 3.1 Mutex (Mutual Exclusion Lock)

The most fundamental synchronization primitive. Ensures **only one thread holds the lock at a time**:

```
Thread A:                      Thread B:
lock(mutex)                    lock(mutex) // BLOCKED
critical_section()             // Waiting...
unlock(mutex) ─┐               ┌→ PROCEEDS
               └─ Available ──┘
```

**Spin Lock Implementation (simplified):**
```c
void spin_lock(atomic_int *lock) {
  while (atomic_exchange(lock, LOCKED) == LOCKED) {
    // Spin (busy wait) until lock acquired
  }
}

void spin_unlock(atomic_int *lock) {
  atomic_store(lock, UNLOCKED);
}
```

**Cost Analysis:**
$$\text{Lock Acquisition} = \begin{cases}
O(1) & \text{if uncontended} \\
O(n) & \text{if n threads waiting (spin cost)}
\end{cases}$$

**Deadlock Risk:**
```
Thread A:                      Thread B:
lock(M1)                       lock(M2)
lock(M2) // BLOCKED           lock(M1) // BLOCKED
// Waiting for M2             // Waiting for M1
// DEADLOCK: Neither progresses
```

**Prevention Strategies:**
1. **Lock ordering**: Always acquire locks in same order
2. **Lock timeout**: Detect deadlock via timeout
3. **Try-lock pattern**: Non-blocking lock acquisition

### 3.2 Semaphore

A synchronization primitive with a **counter** allowing multiple threads (counting semaphore) or one thread (binary semaphore):

```
Semaphore(count = 2):

Thread A: wait() → count=1 (permitted)
Thread B: wait() → count=0 (permitted)
Thread C: wait() // BLOCKED, count=0
Thread A: signal() → count=1, Thread C permitted

Conceptually:
wait():
  while (count == 0) { sleep() }
  count--

signal():
  count++
  wake(waiting_thread)
```

**Use Case - Resource Pool:**
```
Semaphore dbConnections(maxConnections = 5)

Worker A: wait() // Acquire connection
  // Use connection
Worker A: signal() // Release connection
```

**Bounded Buffer Problem:**
```
Buffer size = N
itemCount = Semaphore(0)  // Items available
spaceCount = Semaphore(N) // Spaces available
mutex = Mutex()

Producer:
  acquire(spaceCount)
  lock(mutex)
    buffer[writePtr++] = item
  unlock(mutex)
  release(itemCount)

Consumer:
  acquire(itemCount)
  lock(mutex)
    item = buffer[readPtr++]
  unlock(mutex)
  release(spaceCount)
```

### 3.3 Condition Variables

Allows threads to **wait for a condition** while holding a lock, then resume when notified:

```
Thread A:                          Thread B:
lock(mutex)                        lock(mutex)
while (!condition) {               condition = true
  wait(condVar, mutex)  // BLOCK   notify_all(condVar)
  // Automatically reacquire lock
}
// condition is true now
unlock(mutex)

wait() mechanism:
1. Release lock
2. Sleep (atomically)
3. Woken by signal/notify
4. Re-acquire lock
5. Check condition again (spurious wakeup)
```

**Producer-Consumer Pattern:**
```
Queue queue
Mutex m
CondVar full, empty

Producer:
  lock(m)
  while (queue.size == MAX) {
    wait(full, m)  // Wait if queue full
  }
  queue.push(item)
  notify_one(empty)  // Wake consumer
  unlock(m)

Consumer:
  lock(m)
  while (queue.empty()) {
    wait(empty, m)  // Wait if queue empty
  }
  item = queue.pop()
  notify_one(full)  // Wake producer
  unlock(m)
```

### 3.4 Barriers

Synchronization point where **threads wait until all reach the barrier**:

```
Thread A: [===code===] barrier() ──┐
Thread B: [===code===] barrier() ──┼─→ [All threads resume together]
Thread C: [===code===] barrier() ──┘
```

**Implementation Pattern:**
```
Barrier(num_threads = N):
  count = 0
  condition = CondVar()
  
arrive():
  lock(mutex)
  count++
  if (count == N) {
    notify_all(condition)  // Wake all
  } else {
    wait(condition, mutex)  // Sleep until all arrived
  }
  unlock(mutex)
```

**Use Case - Phase Synchronization:**
```
Phase 1: All workers compute local data
barrier() // All must finish Phase 1 before Phase 2
Phase 2: All workers aggregate data
barrier() // All must finish Phase 2 before Phase 3
Phase 3: Results finalized
```

---

## 4. Lock-Free Data Structures and Algorithms

### 4.1 Atomic Operations and Compare-And-Swap (CAS)

**Atomic operations** execute indivisibly without interruption:

```
Regular operation (non-atomic):
Load x into register      // Step 1
Increment register        // Step 2
Store register into x     // Step 3
// Can interleave between steps

Atomic increment:
atomic_increment(&x)      // Indivisible - no interleaving possible
```

**Compare-And-Swap (CAS)** - The fundamental lock-free primitive:

$$\text{CAS}(address, expected, new) = \begin{cases}
\text{Success} & \text{if } [address] == expected \text{ (swap to } new) \\
\text{Failure} & \text{if } [address] \neq expected \text{ (no change)}
\end{cases}$$

**Hardware Implementation** (x86):
```
lock cmpxchg mem, reg  // Atomic test-and-swap with memory lock
```

**Lock-Free Increment Using CAS:**
```
void atomic_increment(atomic_int *x) {
  while (true) {
    int old = *x;
    int new = old + 1;
    if (CAS(x, old, new)) {  // Succeeds if value unchanged
      break;
    }
    // Retry if value changed between read and CAS
  }
}
```

### 4.2 Lock-Free Stack

Using CAS to implement thread-safe stack without locks:

```
struct Node {
  int value;
  Node* next;
};

struct LockFreeStack {
  atomic<Node*> top;
  
  void push(int value) {
    Node* newNode = new Node{value, nullptr};
    
    while (true) {
      Node* oldTop = top.load();
      newNode->next = oldTop;
      
      if (top.compare_exchange_strong(oldTop, newNode)) {
        break;  // Success
      }
      // Retry if another thread modified top
    }
  }
  
  bool pop(int& value) {
    while (true) {
      Node* oldTop = top.load();
      
      if (oldTop == nullptr) {
        return false;  // Stack empty
      }
      
      Node* newTop = oldTop->next;
      
      if (top.compare_exchange_strong(oldTop, newTop)) {
        value = oldTop->value;
        delete oldTop;
        return true;
      }
      // Retry if another thread modified top
    }
  }
};
```

**Performance Characteristics:**
$$\text{Contention} = \frac{\text{CAS failures}}{\text{Total attempts}}$$

- **Uncontended**: CAS succeeds first try (O(1) with very low overhead)
- **Contended**: Many CAS failures (retry loops, eventual success but busy-wait)
- **Highly contended**: Lock-free worse than locks (spinning, cache thrashing)

### 4.3 ABA Problem

**Subtle bug** in lock-free algorithms:

```
Initial state: [A] → [B] → [C]

Thread 1:
  ptr = top;  // [A]
  // Preempted...

Thread 2:
  pop();      // Remove [A]
  // Somehow [A] reappears at top
  [A] → [D] → [C]

Thread 1:
  compare_exchange(A, B)  // Looks valid! But B is now orphaned
  // [B] → [C] lost (memory leak or corruption)
```

**Solution - Versioned Pointers:**

```
struct VersionedPtr {
  Node* ptr;
  uint64_t version;  // Incremented on every modification
};

// CAS now compares both pointer AND version
compare_exchange_strong(
  {oldPtr, version1},
  {newPtr, version1+1}
)
// ABA impossible - version proves genuine change
```

---

## 5. Event-Driven Architecture and Async Models

### 5.1 Event Loop Model

Handles **many concurrent operations with single thread** through event multiplexing:

```
┌──────────────────────────┐
│ Event Loop               │
│  while (!done) {         │
│    events = poll()       │
│    for event in events   │
│      handle(event)       │
│  }                       │
└──────────────────────────┘
    ▲                  │
    │                  ▼
[Pending Ops]   [Ready Events]
    ▲                  │
    │                  ▼
    └────────────────────┘
```

**Operating System Support - Select/Poll/Epoll:**

```
fd_set readfds, writefds;
FD_SET(socket1, &readfds);
FD_SET(socket2, &readfds);

// Block until socket1 OR socket2 has data
int ready = select(maxfd+1, &readfds, &writefds, NULL, timeout);

// Kernel tracks file descriptor state, returns ready fds
for (int i = 0; i < maxfd; i++) {
  if (FD_ISSET(i, &readfds)) {
    // File descriptor i has data ready
    handle_read(i);
  }
}
```

**Scalability:**
$$\text{Max Concurrent Connections} = \begin{cases}
O(n) \text{ with select (scans all fds)} \\
O(1) \text{ with epoll (returns ready fds only)}
\end{cases}$$

**Blocking vs. Non-Blocking Operations:**
```
Blocking I/O:
Operation starts → BLOCKS → Returns result

Non-Blocking I/O:
Operation starts → Returns immediately
  (with EAGAIN if not ready)

Event-Driven:
Operation starts (returns immediately)
When ready, event fires → callback invoked
```

### 5.2 Callbacks and Promise-Based Async

**Callback Model** (continuation-passing style):

```javascript
fetchData(url, function(err, data) {
  if (err) handleError(err);
  else {
    processData(data, function(err, result) {
      if (err) handleError(err);
      else {
        saveResult(result, function(err) {
          if (err) handleError(err);
          else console.log("Done");
        });
      }
    });
  }
});
// Callback Hell / Pyramid of Doom
```

**Promise Model** (composable async):

```javascript
fetchData(url)
  .then(data => processData(data))
  .then(result => saveResult(result))
  .then(() => console.log("Done"))
  .catch(err => handleError(err));
```

**Promise State Machine:**
```
┌──────────────┐
│  Pending     │
└──────┬───────┘
       │
       ├─→ Fulfilled (value available)
       │      └─→ .then() handlers execute
       │
       └─→ Rejected (error occurred)
              └─→ .catch() handlers execute
              
Once Fulfilled or Rejected → Immutable (no state changes)
```

**Promise Implementation (simplified):**
```javascript
class MyPromise {
  constructor(executor) {
    this.state = 'PENDING';
    this.value = undefined;
    this.handlers = [];
    
    const resolve = (value) => {
      if (this.state === 'PENDING') {
        this.state = 'FULFILLED';
        this.value = value;
        this.handlers.forEach(h => h.onResolve(value));
      }
    };
    
    const reject = (error) => {
      if (this.state === 'PENDING') {
        this.state = 'REJECTED';
        this.value = error;
        this.handlers.forEach(h => h.onReject(error));
      }
    };
    
    executor(resolve, reject);
  }
  
  then(onResolve, onReject) {
    return new MyPromise((resolve, reject) => {
      const handler = {
        onResolve: (value) => {
          try {
            resolve(onResolve(value));
          } catch (e) {
            reject(e);
          }
        },
        onReject: (error) => {
          try {
            resolve(onReject(error));
          } catch (e) {
            reject(e);
          }
        }
      };
      
      if (this.state === 'PENDING') {
        this.handlers.push(handler);
      } else if (this.state === 'FULFILLED') {
        handler.onResolve(this.value);
      } else {
        handler.onReject(this.value);
      }
    });
  }
}
```

### 5.3 Async/Await Syntax

**Syntactic sugar** over promises for sequential-looking code:

```javascript
// Promise version
function fetchAndProcess() {
  return fetchData(url)
    .then(data => processData(data))
    .then(result => saveResult(result))
    .then(() => "Done");
}

// Async/Await version (same logic, sequential appearance)
async function fetchAndProcess() {
  try {
    const data = await fetchData(url);
    const result = await processData(data);
    await saveResult(result);
    return "Done";
  } catch (err) {
    handleError(err);
  }
}

// Calling
fetchAndProcess().then(console.log).catch(console.error);
```

**Compilation to Promise Chains:**
```javascript
// Async/await
async function foo() {
  const a = await op1();
  const b = await op2(a);
  return op3(b);
}

// Compiles to approximately:
function foo() {
  return op1()
    .then(a => op2(a))
    .then(b => op3(b));
}
```

---

## 6. JavaScript/Node.js Concurrency Model

### 6.1 Single-Threaded Event Loop

JavaScript executes in a **single-threaded event loop** with asynchronous I/O:

```
┌─────────────────────────────────────────┐
│ Call Stack                              │
│ ┌─────────────────────────────────────┐ │
│ │ function execution                  │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Task Queue (Macrotasks)                 │
│ ┌─────────────┐ ┌─────────────┐         │
│ │ setTimeout  │ │ I/O ready   │ ...     │
│ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Microtask Queue                         │
│ ┌─────────────┐ ┌─────────────┐         │
│ │ Promise     │ │ queueMicro  │ ...     │
│ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Event Loop                              │
│ while (taskQueue.hasTasks()) {          │
│   if (callStack.empty()) {              │
│     runAllMicrotasks();                 │
│     runOneTask();                       │
│     render();                           │
│   }                                     │
│ }                                       │
└─────────────────────────────────────────┘
```

**Execution Order:**
$$\text{CallStack} \xrightarrow{\text{empty}} \text{Microtasks} \xrightarrow{\text{empty}} \text{OneTask} \xrightarrow{\text{render}} \text{Repeat}$$

**Example - Execution Timeline:**
```javascript
console.log('1');           // → Synchronous, logs immediately

setTimeout(() => {
  console.log('2');         // → Macrotask (Task queue)
}, 0);

Promise.resolve()
  .then(() => console.log('3'))  // → Microtask (Microtask queue)
  .then(() => console.log('4'));

console.log('5');           // → Synchronous, logs immediately

// Output: 1, 5, 3, 4, 2
// Reasoning:
// 1. Call stack: 1, 5
// 2. Microtask queue: 3, 4
// 3. Task queue: 2
```

### 6.2 The GIL in Python

Python has a **Global Interpreter Lock (GIL)** - a mutex preventing true parallelism:

```
┌──────────────────────────────────┐
│ Python Interpreter               │
│ ┌──────────────────────────────┐ │
│ │ GIL (Global Lock)            │ │
│ │ Only one thread executes     │ │
│ │ Python bytecode at a time    │ │
│ └──────────────────────────────┘ │
│                                  │
│ ┌─────────────┐ ┌─────────────┐ │
│ │ Thread 1    │ │ Thread 2    │ │
│ │ Holds GIL   │ │ Waits       │ │
│ │ Executing   │ │ for GIL     │ │
│ └─────────────┘ └─────────────┘ │
└──────────────────────────────────┘
```

**Performance Impact:**
```python
# CPU-bound work: GIL prevents parallelism
def cpu_task(n):
    sum = 0
    for i in range(n):
        sum += i
    return sum

# Sequential: ~1 second for 2x100M iterations
result = cpu_task(100000000) + cpu_task(100000000)

# Multithreaded: ~1 second still (GIL prevents parallelism)
t1 = Thread(target=cpu_task, args=(100000000,))
t2 = Thread(target=cpu_task, args=(100000000,))
t1.start(); t2.start()
t1.join(); t2.join()

# Multiprocess: ~0.5 seconds (true parallelism, separate interpreters)
p1 = Process(target=cpu_task, args=(100000000,))
p2 = Process(target=cpu_task, args=(100000000,))
p1.start(); p2.start()
p1.join(); p2.join()
```

**I/O Operations Release GIL:**
```python
# I/O-bound work: GIL released during I/O
def io_task():
    response = requests.get('http://example.com')  # Releases GIL
    return response.text

# Multithreaded: Concurrent I/O (GIL released)
# Both threads can do I/O simultaneously
threads = [Thread(target=io_task) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
```

---

## 7. Common Concurrency Patterns and Anti-patterns

### 7.1 Thread Safety Patterns

**Immutability:**
```
// THREAD-SAFE: No shared mutable state
const config = Object.freeze({
  maxConnections: 100,
  timeout: 5000
});

// All threads read config safely
thread1.use(config);
thread2.use(config);
```

**Synchronized Access:**
```
// THREAD-SAFE: Guarded by lock
class ThreadSafeCounter {
  private int count = 0;
  private final Object lock = new Object();
  
  public void increment() {
    synchronized(lock) {
      count++;  // Only one thread at a time
    }
  }
  
  public int get() {
    synchronized(lock) {
      return count;
    }
  }
}
```

**Copy-on-Write:**
```
// THREAD-SAFE: Readers don't block, writers create copy
class CopyOnWriteList<T> {
  private volatile T[] array;
  
  public T get(int index) {
    return array[index];  // Never blocks
  }
  
  public void add(T element) {
    synchronized(this) {
      T[] newArray = Arrays.copyOf(array, array.length + 1);
      newArray[array.length] = element;
      array = newArray;  // Volatile write
    }
  }
}
```

### 7.2 Common Anti-patterns

**Race Condition - Check-Then-Act:**
```
// UNSAFE: Race condition between check and act
if (list.isEmpty()) {  // Thread A checks
  // Thread B adds element here
  list.add(element);   // Thread A adds (no race visible)
}

// SAFE: Atomic check-and-act
synchronized(list) {
  if (list.isEmpty()) {
    list.add(element);  // Atomic
  }
}
```

**Double-Checked Locking Anti-pattern:**
```
// BROKEN: Still has race condition in some languages
class Singleton {
  private static Singleton instance;
  
  public static Singleton getInstance() {
    if (instance == null) {  // First check (no lock)
      synchronized(Singleton.class) {
        if (instance == null) {  // Second check (locked)
          instance = new Singleton();  // Can still have races
        }
      }
    }
    return instance;
  }
}

// CORRECT: Eager initialization or volatile
class Singleton {
  private static volatile Singleton instance;  // Volatile!
  
  public static Singleton getInstance() {
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null) {
          instance = new Singleton();  // Volatile write ensures visibility
        }
      }
    }
    return instance;
  }
}

// SIMPLER: Static initialization
class Singleton {
  public static final Singleton INSTANCE = new Singleton();
}
```

**Deadlock Scenarios:**
```
// DEADLOCK: Circular lock dependency
Thread A:
  lock(M1)
  lock(M2)  // Waits for M2

Thread B:
  lock(M2)
  lock(M1)  // Waits for M1 - DEADLOCK!

// SOLUTION: Establish lock ordering
Always lock: M1 → M2 (never M2 → M1)
```

---

## 8. Production Systems and Performance Considerations

### 8.1 Amdahl's Law and Speedup

**Amdahl's Law** quantifies speedup from parallelism:

$$S = \frac{1}{(1-p) + \frac{p}{n}}$$

Where:
- $S$ = speedup factor
- $p$ = fraction of code parallelizable
- $n$ = number of processors

**Examples:**
```
Parallelizable = 50%, Processors = 4:
S = 1 / (0.5 + 0.5/4) = 1 / 0.625 = 1.6x speedup
(Not 4x despite 4 processors!)

Parallelizable = 90%, Processors = 4:
S = 1 / (0.1 + 0.9/4) = 1 / 0.325 = 3.08x speedup

Parallelizable = 99%, Processors = 4:
S = 1 / (0.01 + 0.99/4) = 1 / 0.2575 = 3.88x speedup

Parallelizable = 99%, Processors = ∞:
S = 1 / (0.01 + 0) = 100x speedup (theoretical limit)
```

**Implication**: Even small sequential portions severely limit parallelism.

### 8.2 Thread Pool Architecture

Instead of creating threads per task (expensive), reuse threads:

```
┌─────────────────────────────────┐
│ Task Queue                      │
│ [Task1] [Task2] [Task3] ...     │
└─────────────────────────────────┘
        ▲
        │
┌─────────────────────────────────┐
│ Thread Pool (N worker threads)  │
│ ┌───────┐ ┌───────┐             │
│ │ Thd 1 │ │ Thd 2 │ ...         │
│ │ (busy)│ │(idle) │             │
│ └───────┘ └───────┘             │
└─────────────────────────────────┘
```

**Parameter Tuning:**
$$N_{\text{threads}} = \begin{cases}
\text{cpu cores} & \text{CPU-bound tasks} \\
\text{cpu cores} \times (1 + \text{wait ratio}) & \text{I/O-bound tasks}
\end{cases}$$

For I/O with 90% wait time:
$$N = \text{cores} \times (1 + 0.9) = \text{cores} \times 1.9$$

### 8.3 Performance Pitfalls

**Context Switch Thrashing:**
- Too many threads → excessive context switches
- Cost: 1-100 microseconds per switch
- Remedy: Thread pool, limit concurrent threads

**Cache Invalidation:**
- Lock contention → threads invalidate each other's caches
- Cost: 100-300 cycle cache miss (vs 1-3 cycles for cache hit)
- Remedy: Reduce contention, lock-free algorithms, separate cache lines

**GIL Thrashing (Python):**
- Threads releasing/acquiring GIL rapidly
- Cost: Overhead exceeds parallelism benefit
- Remedy: Use multiprocessing, async/await, Cython

---

## Key Takeaways

1. **Concurrency Models**: Shared memory (complex but fast), message passing (complex reasoning avoided), hybrid (actor model).

2. **Synchronization Primitives**:
   - **Mutex**: Mutual exclusion (simplest, most common)
   - **Semaphore**: Counter-based (resource pools)
   - **Condition Variable**: Wait for conditions
   - **Barrier**: Phase synchronization

3. **Lock-Free Programming**: Uses CAS for synchronization without locks. Uncontended is fast, but contended worse than locks. Solves GC pause and priority inversion issues.

4. **Event Loop**: Single-threaded model (JS, Node.js) handles millions of concurrent connections through I/O multiplexing.

5. **JavaScript Execution**: Synchronous code → Microtasks (Promises) → Macrotasks (setTimeout).

6. **Python GIL**: Prevents true CPU parallelism with threads. Use multiprocessing for CPU-bound, async for I/O-bound.

7. **Amdahl's Law**: Speedup is limited by serial portions. 10% serial = max 10x speedup even on ∞ processors.

8. **Thread Pools**: Reuse threads, avoid creation overhead. Size based on CPU cores (CPU-bound) or cores × (1 + wait ratio) (I/O-bound).

9. **Common Pitfalls**: Race conditions (check-then-act), deadlocks (establish lock ordering), context switch thrashing (limit threads), cache invalidation (reduce contention).

10. **Testing Concurrent Code**: Extremely difficult due to nondeterminism. Use tools (ThreadSanitizer, Helgrind), stress testing, property-based testing.

11. **Modern Approaches**: Prefer immutability, async/await over callbacks, functional programming with no shared state.

12. **Distributed Concurrency**: Message passing + event-driven (microservices, Kafka, distributed actors) avoids many lock-based problems.
