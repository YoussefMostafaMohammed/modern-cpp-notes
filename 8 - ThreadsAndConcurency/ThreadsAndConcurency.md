# Modern C++ Concurrency & Type Safety

## 📋 Table of Contents
1. [C++ Type Casting Deep Dive](#type-casting-deep-dive)
   - [static_cast vs dynamic_cast](#static_cast-vs-dynamic_cast)
   - [const_cast](#const_cast)
   - [Decision Tree for Casting](#decision-tree-for-casting)
2. [Threading Fundamentals](#threading-fundamentals)
   - [std::thread vs std::jthread](#stdthread-vs-stdjthread)
   - [Join vs Detach](#join-vs-detach)
   - [Launching Threads](#launching-threads)
   - [Thread-Local Storage](#thread-local-storage)
   - [Cooperative Cancellation](#cooperative-cancellation)
   - [Threads in Classes](#threads-in-classes)
3. [Thread Pools](#thread-pools)
   - [Why Thread Pools Exist](#why-thread-pools-exist)
   - [Implementation Details](#thread-pool-implementation)
   - [When to Use Thread Pools](#when-to-use-thread-pools)
4. [Async, Futures, and Promises](#async-futures-and-promises)
   - [std::async](#stdasync)
   - [std::promise and std::future](#stdpromise-and-stdfuture)
   - [Shared Futures](#shared-futures)
   - [Exception Handling Across Threads](#exception-handling-across-threads)
   - [Promise/Future vs Condition Variables](#promisefuture-vs-condition-variables)
5. [Synchronization Primitives](#synchronization-primitives)
   - [std::mutex](#stdmutex)
   - [Condition Variables](#condition-variables)
   - [Why CV Needs Mutex](#why-cv-needs-mutex)
   - [Spinlocks](#spinlocks)
   - [Mutex Variants](#mutex-variants)
   - [Lock Guards and RAII](#lock-guards-and-raii)
   - [std::call_once](#stdcall_once)
   - [Double-Checked Locking (Anti-Pattern)](#double-checked-locking-anti-pattern)
6. [Atomic Operations](#atomic-operations)
   - [Why Atomics Matter](#why-atomics-matter)
   - [Atomic Types](#atomic-types)
   - [Common Operations](#common-operations)
   - [Atomic Smart Pointers](#atomic-smart-pointers)
   - [Atomic Reference](#atomic-reference)
7. [Advanced Synchronization](#advanced-synchronization)
   - [Latches](#latches)
   - [Barriers](#barriers)
   - [Semaphores](#semaphores)
8. [Race Conditions and Deadlocks](#race-conditions-and-deadlocks)
   - [Race Conditions](#race-conditions)
   - [Tearing](#tearing)
   - [Deadlocks](#deadlocks)
   - [False Sharing](#false-sharing)
9. [Coroutines](#coroutines)
   - [Coroutines vs Threads vs Async](#coroutines-vs-threads-vs-async)
   - [co_await, co_return, co_yield](#co_await-co_return-co_yield)
   - [Hybrid Patterns](#hybrid-patterns)
10. [std::optional](#stdoptional)
    - [The Problem of Sentinel Values](#the-sentinel-value-problem)
    - [When to Use Optional](#when-to-use-optional)
    - [Implementation Details](#implementation-details)
11. [Real-World Example: Multithreaded Logger](#multithreaded-logger)
12. [Best Practices & Decision Trees](#best-practices)
    - [Choosing Between Concurrency Tools](#choosing-concurrency-tools)
    - [Performance Comparison](#performance-comparison)
    - [Common Pitfalls](#common-pitfalls)
    - [Threading Design Guidelines](#threading-design-guidelines)

---

## Type Casting Deep Dive

### static_cast vs dynamic_cast

#### **What static_cast Does**
A compile-time cast for related types that trusts you blindly. No runtime checks.

```cpp
// 1. Numeric conversions
double pi = 3.14159;
int approx = static_cast<int>(pi); // Explicit truncation

// 2. void* pointer recovery
void* raw_memory = malloc(sizeof(int));
int* num = static_cast<int*>(raw_memory);

// 3. Up/down casting in inheritance
class Base {};
class Derived : public Base {};

Derived d;
Base* b = static_cast<Base*>(&d);  // Up-cast: safe, implicit
Derived* d2 = static_cast<Derived*>(b); // Down-cast: DANGEROUS, no runtime check
```

**What static_cast trusts you with:**
```cpp
Base* b = new Base; // NOT a Derived!
Derived* d = static_cast<Derived*>(b); // Compiles! But crashes on use
d->someDerivedMethod(); // UNDEFINED BEHAVIOR - memory corruption
```

**Why it exists**: To make conversions explicit. Without it, you'd use C-style casts that bypass type checking.

#### **What dynamic_cast Does**
A runtime-checked cast for polymorphic types (must have at least one virtual function).

```cpp
class Base { virtual ~Base() {} }; // Must be polymorphic
class Derived : public Base {};

Base* animal = new Derived;
Derived* dog = dynamic_cast<Derived*>(animal); // ✅ Returns valid pointer

Base* animal2 = new Base;
Derived* dog2 = dynamic_cast<Derived*>(animal2); // ✅ Returns nullptr (safe failure)

// With references: throws std::bad_cast on failure
Base& cat_ref = *animal2;
try {
    Derived& dog_ref = dynamic_cast<Derived&>(cat_ref); // Throws!
} catch(std::bad_cast&) { /* handle safely */ }
```

**Why it requires polymorphism**: It uses RTTI stored in the vtable to verify the object's real type at runtime.

**Performance cost**: ~50-200 cycles vs 0-1 for static_cast.

#### **Decision Tree for Casting**
```
Is it a pointer/reference to a class type?
├─ NO → Use static_cast (primitives, void*)
│
├─ YES → Is the class polymorphic (has virtual functions)?
│   ├─ NO → static_cast ONLY (dynamic_cast won't compile)
│   │
│   └─ YES → Are you down-casting Base→Derived?
│       ├─ YES → Use dynamic_cast (for safety)
│       └─ NO → Use static_cast (up-cast or cross-cast)
```

---

### const_cast

The **only cast** that can add/remove `const` qualifiers. It's a **code smell**.

```cpp
// LEGACY API that doesn't respect const
void legacy_print(char* str);

void modern_print(const std::string& s) {
    legacy_print(const_cast<char*>(s.c_str())); // Safe if s is non-const originally
}

// DANGEROUS: Modifying const object
const int x = 42;
int* ptr = const_cast<int*>(&x);
*ptr = 100; // UNDEFINED BEHAVIOR - may crash or corrupt
```

**Legitimate use cases**:
- Adapting to const-incorrect APIs
- Implementing caches with mutable members
- C-style API integration

**When to avoid**: Never use to modify actual `const` objects.

---

## Threading Fundamentals

### std::thread vs std::jthread

```cpp
// C++11: Must manually join or detach
void old_way() {
    std::thread t([]{ /* work */ });
    t.join(); // FORGET THIS = std::terminate() crash!
}

// C++20: Auto-joins on destruction
void new_way() {
    std::jthread t([]{ /* work */ }); // Automatic join
    // Destructor calls request_stop() and join()
}
```

**Key differences**:
- `jthread` automatically joins on destruction
- `jthread` supports cooperative cancellation via `std::stop_token`
- `jthread` is the modern default; use `std::thread` only when you need explicit detach

### Join vs Detach

**`join()`**: Parent waits for child thread to finish.
- Blocks calling thread until target thread completes
- Reaps thread resources (stack, handle)
- Must be called exactly once if thread is joinable

**`detach()`**: Parent releases child thread to run independently.
- Thread becomes daemon (runs until program exit)
- No cleanup guarantee - thread may be killed on program exit
- **Irreversible**: After detach, cannot join

**Rule**:
- **Always join** unless you need a true daemon that outlives its creator
- **Use `jthread`** to avoid forgetting to join

### Launching Threads

#### **Function Pointer**
```cpp
void counter(int id, int numIterations) {
    for (int i = 0; i < numIterations; ++i) {
        println("Counter {} has value {}", id, i);
    }
}

thread t1 { counter, 1, 6 }; // Function + args
thread t2 { counter, 2, 4 };
t1.join(); t2.join();
```

#### **Function Object**
```cpp
class Counter {
public:
    Counter(int id, int numIterations) : m_id{id}, m_numIterations{numIterations} {}
    void operator()() const {
        for (int i = 0; i < m_numIterations; ++i) {
            println("Counter {} has value {}", m_id, i);
        }
    }
private:
    int m_id, m_numIterations;
};

thread t1 { Counter{1, 20} }; // Temporary
Counter c{2, 12};
thread t2 { ref(c) }; // By reference (avoid copying)
t1.join(); t2.join();
```

#### **Lambda Expression**
```cpp
int id = 1, numIterations = 5;
thread t1 { [id, numIterations] {
    for (int i = 0; i < numIterations; ++i) {
        println("Counter {} has value {}", id, i);
    }
} };
t1.join();
```

#### **Member Function**
```cpp
class Request {
public:
    explicit Request(int id) : m_id{id} {}
    void process() { println("Processing request {}", m_id); }
private:
    int m_id;
};

Request req{100};
thread t { &Request::process, &req }; // Pointer to member + object
t.join();
```

### Thread-Local Storage

Each thread gets its own copy of a variable.

```cpp
int shared_var = 0;
thread_local int thread_var = 0;

void thread_func(int id) {
    println("Thread {}: shared={}, thread={}", id, shared_var, thread_var);
    ++thread_var;
    ++shared_var;
}

int main() {
    thread t1{thread_func, 1}; t1.join(); // shared=0, thread=0
    thread t2{thread_func, 2}; t2.join(); // shared=1, thread=0 (new copy)
}
```

### Cooperative Cancellation

**`std::jthread`** supports graceful cancellation:

```cpp
void threadFunction(stop_token token, int id) {
    while (!token.stop_requested()) {
        println("Thread {} working", id);
        this_thread::sleep_for(500ms);
    }
    println("Stop requested for thread {}", id);
}

int main() {
    jthread job1{threadFunction, 1};
    jthread job2{threadFunction, 2};
    
    this_thread::sleep_for(2s);
    // Destructor calls request_stop() and join() automatically
}
```

**Manual control**:
```cpp
jthread job{threadFunction, 1};
job.request_stop(); // Signal cancellation
job.join(); // Wait for graceful exit
```

**stop_token and stop_source**:
```cpp
stop_source src; // Controls cancellation
stop_token token = src.get_token(); // Observes cancellation

jthread t {[token]{ /* check periodically */ }};
src.request_stop(); // Cancel from another thread
```

---

### Threads in Classes

```cpp
class DataProcessor {
    std::jthread worker_; // C++20 for safety
    
public:
    void start() {
        worker_ = std::jthread([this] {
            while (!worker_.get_stop_token().stop_requested()) {
                process_data();
            }
        });
    }
    // No manual join needed - jthread handles it
};
```

**Important**: Capture `this` by value and pass to thread, or use lambda capture carefully to avoid dangling references.

---

## Thread Pools

### Why Thread Pools Exist

Creating threads is expensive (~2-8MB stack + kernel overhead). For many small tasks, thread creation overhead dominates execution time.

**Problem: Naked threads**
```cpp
for (int i = 0; i < 1000; ++i) {
    std::thread t(small_task);
    t.join(); // 1000 thread creations = 10-100ms overhead each = terrible performance
}
```

**Solution: Reuse threads**
```cpp
ThreadPool pool(4); // 4 persistent threads
for (int i = 0; i < 1000; ++i) {
    pool.enqueue(small_task); // Tasks queued, threads pick them up
    // Amortized overhead: near zero after initial creation
}
```

---

### Thread Pool Implementation

```cpp
class ThreadPool {
    std::vector<std::jthread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mtx_;
    std::condition_variable cv_;
    bool stop_ = false;
    
public:
    ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock lock(mtx_);
                        cv_.wait(lock, [this]{ return stop_ || !tasks_.empty(); });
                        
                        if (stop_ && tasks_.empty()) return;
                        
                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }
                    task(); // Execute OUTSIDE lock
                }
            });
        }
    }
    
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using ReturnType = decltype(f(args...));
        
        auto packaged_task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        
        auto future = packaged_task->get_future();
        
        {
            std::lock_guard lock(mtx_);
            if (stop_) throw std::runtime_error("enqueue on stopped pool");
            tasks_.emplace([packaged_task]{ (*packaged_task)(); });
        }
        
        cv_.notify_one();
        return future;
    }
    
    ~ThreadPool() {
        {
            std::lock_guard lock(mtx_);
            stop_ = true;
        }
        cv_.notify_all();
    }
};

// Usage
ThreadPool pool(4);
auto future = pool.enqueue([](int x){ return x * x; }, 5);
std::cout << future.get(); // 25
```

**Key features**:
- **Task queue**: Unbounded queue of `std::function<void()>`
- **Work stealing**: Threads wait on condition variable when idle
- **Future return**: `enqueue()` returns `std::future` for result retrieval
- **RAII shutdown**: Destructor stops all threads gracefully

---

### When to Use Thread Pools

**Use when**:
- Many short-lived tasks (task duration < thread creation overhead)
- Need to limit resource usage (control max concurrency)
- Want to amortize thread creation cost

**Don't use when**:
- Long-running daemon threads (just create dedicated threads)
- CPU-bound work that needs exactly N threads (use `std::thread` directly)
- Need priority scheduling (pools are FIFO)

**Real-world alternatives**:
- **Intel TBB**: Production-grade thread pool with work stealing
- **Boost.Asio**: `io_context` is an implicit thread pool
- **Microsoft PPL**: Concurrency runtime thread pool

---

## Async, Futures, and Promises

### std::async

```cpp
// Eagerly starts task on new thread (or thread pool)
auto future = std::async(std::launch::async, heavy_calc, 5);
// Main thread continues...
int result = future.get(); // Blocks until ready

// Launch policies:
std::launch::async → Force new thread
std::launch::deferred → Lazy evaluation (runs on .get())
Default → Implementation chooses (may use thread pool)
```

**Why use `async` over raw threads?**:
- **Automatic result transport** via future (no manual std::ref)
- **Exception propagation** (exceptions are caught and re-thrown)
- **Potential thread pooling** (default policy)
- **Cleaner API**: returns values naturally

**When to use**:
- Tasks with return values
- Need exception safety
- Short-lived parallelism (not long-running services)

---

### std::promise and std::future

**The one-time mailbox pattern**:

```cpp
std::promise<int> prom; // Write end
std::future<int> fut = prom.get_future(); // Read end

// In worker thread:
prom.set_value(42); // Seal envelope

// In main thread:
int val = fut.get(); // Open envelope
```

**Critical properties**:
-  **`set_value()` once**  : Second call throws `std::future_error`
-  **`get()` once**  : Extracts value, future becomes invalid
- **Move-only**: Cannot copy promise or future
- **Reference-counted**: Shared state keeps alive until both destroyed

**Exception transport**:
```cpp
try {
    throw std::runtime_error("Oops");
    prom.set_value(42);
} catch(...) {
    prom.set_exception(std::current_exception()); // Transport exception
}

try {
    int val = fut.get(); // Re-throws exception here
} catch(const std::exception& e) {
    // Handle error
}
```

---

### Shared Futures

**Problem**: Multiple threads need the same result

```cpp
std::promise<int> prom;
std::shared_future<int> sfut = prom.get_future().share();

// Both threads can get() the result
std::thread t1([sfut]{ std::cout << "T1: " << sfut.get(); });
std::thread t2([sfut]{ std::cout << "T2: " << sfut.get(); });

prom.set_value(42); // Both see 42
```

**Key difference**: `shared_future::get()` can be called multiple times; regular `future::get()` can only be called once.

---

### Exception Handling Across Threads

Exceptions thrown in one thread cannot be caught in another thread. C++ provides `std::exception_ptr` for cross-thread exception transport.

```cpp
void doSomeWork() {
    throw runtime_error("Exception from thread");
}

void threadFunc(exception_ptr& err) {
    try {
        doSomeWork();
    } catch(...) {
        err = current_exception(); // Capture exception
    }
}

void doWorkInThread() {
    exception_ptr error;
    jthread t{threadFunc, ref(error)};
    t.join();
    
    if (error) {
        rethrow_exception(error); // Re-throw in main thread
    }
}

int main() {
    try {
        doWorkInThread();
    } catch(const exception& e) {
        println("Caught: '{}'", e.what());
    }
}
```

**Key functions**:
- `current_exception()`: Captures current exception
- `rethrow_exception()`: Rethrows from another thread
- `make_exception_ptr()`: Creates exception_ptr from exception object

---

### Promise/Future vs Condition Variables

| Feature | `promise/future` | `condition_variable` |
|---------|------------------|----------------------|
| **Purpose** | One-time result delivery | Ongoing state signaling |
| **Data** | Single payload (value/exception) | No built-in data |
| **Lifetime** | Single use (set once) | Reusable (notify multiple times) |
| **Exception** | ✅ Automatic transport | ❌ Manual |
| **Use Case** | Return values from async tasks | Producer/consumer queues |

**Rule of thumb**: Use `promise/future` for "return this value later". Use `condition_variable` for "wait for this condition repeatedly".

---

## Synchronization Primitives

### std::mutex

The **basic mutual exclusion lock**. Ensures only one thread accesses shared data.

```cpp
std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard lock(mtx); // RAII lock
    counter++; // Protected
} // Auto-unlock
```

**Never manually lock/unlock** (exception unsafe):
```cpp
mtx.lock(); // ❌ Dangerous
if (error) throw; // ❌ Never unlocks = deadlock!
mtx.unlock();
```

**Mutex types**:
- `std::mutex`: Non-recursive (default)
- `std::recursive_mutex`: Can lock multiple times on same thread
- `std::timed_mutex`: `try_lock_for(duration)`
- `std::shared_mutex`: Multiple readers, single writer

**When to use atomics instead**:
```cpp
// For simple counters, atomics are 10x faster than mutex
std::atomic<int> counter = 0;
counter++; // Hardware atomic instruction (LOCK INC)
```

---

### Condition Variables

**Efficient waiting for conditions** without busy-waiting.

```cpp
std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

void consumer() {
    std::unique_lock lock(mtx);
    cv.wait(lock, []{ return !queue.empty(); }); // Blocks efficiently
    int data = queue.front();
    queue.pop();
}

void producer() {
    std::lock_guard lock(mtx);
    queue.push(42);
    cv.notify_one(); // Wake one waiter
}
```

**Critical pattern**: The predicate lambda handles spurious wakeups.

**Why CV needs mutex**: To make the **check-then-wait** operation atomic, preventing lost wakeups.

---

### Why CV Needs Mutex: The Lost Wakeup Problem

**Without mutex**:
```
Consumer: if (queue.empty()) → true
Producer: queue.push(42); cv.notify_one(); // Consumer not waiting!
Consumer: cv.wait(); // Waits forever - MISSED notification!
```

**With mutex**:
```
Consumer: lock(mtx); cv.wait(lock) → atomically unlocks + blocks
Producer: lock(mtx); queue.push(42); cv.notify_one(); // Consumer IS waiting
Consumer: wakes, relocks, checks condition = true, proceeds
```

The **unlock+block** must be atomic to prevent the race window.

---

### Spinlocks

A spinlock is a synchronization mechanism where a thread uses a busy loop to acquire a lock.

```cpp
atomic_flag spinlock = ATOMIC_FLAG_INIT;

void dowork(vector<unsigned>& data) {
    for (int i = 0; i < 100; ++i) {
        while (spinlock.test_and_set()) {} // Busy wait
        // Critical section
        data.push_back(i);
        spinlock.clear(); // Release lock
    }
}
```

**When to use**: Only when lock is held for extremely short durations (a few instructions).

**Warning**: Spinning wastes CPU cycles. Use only when contention is very low and critical section is tiny.

---

### Mutex Variants

**Non-timed mutexes**:
- `std::mutex`: Basic exclusive lock
- `std::recursive_mutex`: Can lock multiple times on same thread
- `std::shared_mutex`: Multiple readers, single writer (readers-writer lock)

**Timed mutexes** (all support `try_lock_for()` and `try_lock_until()`):
- `std::timed_mutex`: Timeout support
- `std::recursive_timed_mutex`: Recursive + timeout
- `std::shared_timed_mutex`: Readers-writer + timeout

**Shared ownership**: `shared_mutex`/`shared_timed_mutex` support:
- `lock_shared()`: Acquire read lock (shared)
- `unlock_shared()`: Release read lock
- Multiple threads can hold shared locks simultaneously
- Only one thread can hold exclusive lock (no shared locks active)

---

### Lock Guards and RAII

Always use RAII wrappers to ensure locks are released:

```cpp
std::mutex mtx;

// Basic RAII lock
void func1() {
    std::lock_guard lock(mtx); // Acquires on construction
    // ... work ...
} // Auto-releases on destruction

// Defer locking
void func2() {
    std::unique_lock lock(mtx, std::defer_lock);
    // ... do other work ...
    lock.lock(); // Lock later
    // ... critical section ...
} // Auto-releases

// Multiple locks (deadlock-free)
void func3() {
    std::mutex m1, m2;
    std::scoped_lock lock(m1, m2); // Acquires all locks safely
    // ... work ...
} // All released automatically
```

**Scoped lock**: C++17 `std::scoped_lock` acquires multiple locks atomically, preventing deadlocks.

---

### std::call_once

Ensures a function is called exactly once, even across multiple threads.

```cpp
once_flag g_onceFlag;

void initializeSharedResources() {
    println("Shared resources initialized.");
}

void processingFunction() {
    call_once(g_onceFlag, initializeSharedResources);
    // ... do work ...
}

int main() {
    vector<jthread> threads(5);
    for (auto& t : threads) {
        t = jthread{processingFunction};
    }
}
```

**Output**:
```
Shared resources initialized.
Processing
Processing
Processing
Processing
Processing
```

**Important**: If the function throws, another thread is selected to run it. On success, subsequent calls are no-ops.

---

### Double-Checked Locking (Anti-Pattern)

**AVOID THIS PATTERN** - use `call_once()` or magic statics instead.

```cpp
atomic<bool> g_initialized{false};
mutex g_mutex;

void processingFunction() {
    if (!g_initialized) { // First check (unlocked)
        unique_lock lock{g_mutex};
        if (!g_initialized) { // Second check (locked)
            initializeSharedResources();
            g_initialized = true;
        }
    }
}
```

**Problems**:
- Hard to get right (requires careful use of atomics)
- Can still have data races if not perfect
- `call_once()` is usually faster and safer
- Magic statics (function-local statics) are even better

**Proper solution**:
```cpp
void processingFunction() {
    static bool initialized = []{
        initializeSharedResources();
        return true;
    }();
    // ... rest of code ...
}
```

---

## Atomic Operations

### Why Atomics Matter

Without atomics, even simple operations like `++counter` are not thread-safe:

```cpp
int counter = 0;

// Thread 1: ++counter
load value (value = 1)
increment value (value = 2)
// Thread 2 interleaves!
load value (value = 1)
decrement value (value = 0)
store value (value = 0)
store value (value = 2) // OVERWRITES Thread 2's result
```

**Result**: Race condition and lost updates.

---

### Atomic Types

```cpp
// Atomic integer
std::atomic<int> counter{0};
counter++; // Thread-safe, hardware atomic instruction

// Named atomic types
atomic_bool, atomic_int, atomic_long, atomic_ullong, atomic_char, etc.

// Check if lock-free
static_assert(atomic<int>::is_always_lock_free, "Requires lock-free atomics");

// Custom types (must be trivially copyable)
struct Point { int x, y; };
atomic<Point> point; // May use lock if hardware doesn't support it
```

**Always lock-free**: `atomic_flag` is guaranteed to be lock-free. Use for spinlocks.

---

### Common Operations

```cpp
atomic<int> value{10};

// Load/store
int x = value.load(); // Atomic read
value.store(20); // Atomic write

// Exchange
int old = value.exchange(30); // old = 20, value = 30

// Compare-and-swap
int expected = 30;
bool success = value.compare_exchange_strong(expected, 40);
// If value == 30, sets to 40 and returns true
// Otherwise, loads current value into expected and returns false

// Arithmetic
value.fetch_add(5); // Returns old value, adds 5
value.fetch_sub(3); // Returns old value, subtracts 3
// Also fetch_and, fetch_or, fetch_xor for integral types
```

**Memory ordering**: Atomics support relaxed, acquire, release, acq_rel, and seq_cst ordering. **Use default (seq_cst) unless you're an expert**.

---

### Atomic Smart Pointers

`atomic<shared_ptr<T>>` makes the shared_ptr itself thread-safe:

```cpp
atomic<shared_ptr<int>> ptr;

// Thread-safe shared_ptr operations
ptr.store(make_shared<int>(42));
auto local = ptr.load(); // Gets atomic snapshot

// Note: Doesn't make the pointed-to object thread-safe
```

---

### Atomic Reference

`atomic_ref` provides atomic operations on existing non-atomic variables:

```cpp
int counter = 0;

void increment(int& c) {
    atomic_ref<int> atomicCounter{c};
    for (int i = 0; i < 100; ++i) {
        ++atomicCounter; // Atomic increment
    }
}

int main() {
    vector<jthread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(increment, ref(counter));
    }
    // Result is exactly 1000
}
```

**Key difference**: `atomic` owns the value; `atomic_ref` refers to existing memory.

---

## Advanced Synchronization

### Latches

**Single-use thread coordination point**. Threads block until a counter reaches zero.

```cpp
constexpr unsigned numThreads = 10;
latch latch_point{numThreads};

vector<jthread> workers;
for (unsigned i = 0; i < numThreads; ++i) {
    workers.emplace_back([&latch_point, i] {
        // Do work...
        println("Worker {} done", i);
        latch_point.count_down(); // Signal completion
    });
}

latch_point.wait(); // Wait for all workers
println("All workers finished");
```

**Use cases**:
- Wait for all worker threads to complete initialization
- Coordinate parallel data loading
- Barrier for single phase only (can't be reused)

---

### Barriers

**Reusable thread coordination** with phase completion callback.

```cpp
constexpr unsigned numRobots = 4;
constexpr unsigned numIterations = 3;

unsigned iteration = 1;
barrier robotBarrier{numRobots, [&] {
    if (iteration == numIterations) {
        println("All iterations complete");
        // Request shutdown
    } else {
        ++iteration;
        println("Starting iteration {}", iteration);
    }
}};

vector<jthread> robots;
for (unsigned i = 0; i < numRobots; ++i) {
    robots.emplace_back([&robotBarrier, i] {
        for (unsigned iter = 0; iter < numIterations; ++iter) {
            // Do work...
            robotBarrier.arrive_and_wait(); // Block until all done
        }
    });
}
```

**Key difference from latch**: Barrier **resets and repeats**; latch is one-time.

---

### Semaphores

**Lightweight synchronization** to control access to limited resources.

```cpp
// Counting semaphore: allow max 4 concurrent accesses
counting_semaphore semaphore{4};

vector<jthread> workers;
for (int i = 0; i < 10; ++i) {
    workers.emplace_back([&semaphore, i] {
        semaphore.acquire(); // Wait for slot
        println("Worker {} entered critical section", i);
        this_thread::sleep_for(1s);
        semaphore.release(); // Release slot
    });
}
```

**Binary semaphore** (`binary_semaphore`): Equivalent to a mutex (slot = 1).

**Use as notification**: Initialize to 0, `acquire()` waits, `release()` signals.

---

## Race Conditions and Deadlocks

### Race Conditions

**Definition**: Undefined behavior when multiple threads access shared data without synchronization, and at least one writes.

```cpp
// Broken counter
int counter = 0;
void increment() { ++counter; } // NOT atomic: load, inc, store

// With atomics (fixed)
atomic<int> safe_counter{0};
void safe_increment() { ++safe_counter; } // Hardware atomic
```

**Data race example** (increment vs decrement):
```
Thread 1: load(1) → inc(2) → store(2)
Thread 2:       load(1) → dec(0) → store(0) // Lost Thread 1's update!
```

**Result**: Final value can be 0, 1, or 2 depending on interleaving.

---

### Tearing

**Torn read/write**: Reading/writing partially updated data.

```cpp
// 64-bit value on 32-bit system may tear
struct Data { uint32_t a, b; }; // 8 bytes
Data shared{0, 0};

// Thread 1: write {1, 2}
shared = Data{1, 2}; // Not atomic - may write a then b

// Thread 2: read
Data local = shared; // May read {1, 0} (torn) or {0, 2}
```

**Solution**: Use `atomic<Data>` or align to cache line boundaries with `alignas`.

---

### Deadlocks

**Cycle of waiting**: Thread A holds lock 1 waiting for lock 2; Thread B holds lock 2 waiting for lock 1.

```cpp
mutex m1, m2;

void thread1() {
    lock_guard l1{m1};
    this_thread::sleep_for(10ms); // Force interleaving
    lock_guard l2{m2}; // DEADLOCK: waiting for m2
}

void thread2() {
    lock_guard l2{m2};
    lock_guard l1{m1}; // DEADLOCK: waiting for m1
}
```

**Visual representation**:
```
Thread 1 → [m1] → (waits for m2) → Thread 2 → [m2] → (waits for m1) → CYCLE
```

**Prevention**:
1. **Lock ordering**: Always acquire locks in same order
2. **std::lock()**: Acquire multiple locks atomically
3. **scoped_lock**: C++17 deadlock-free multi-lock

```cpp
// Safe with scoped_lock
void safe_thread1() {
    scoped_lock lock{m1, m2}; // Deadlock-free
}

void safe_thread2() {
    scoped_lock lock{m2, m1}; // Same locks, any order = safe
}
```

**Breaking deadlocks**: Use `try_lock_for()` with timeout and backoff.

---

### False Sharing

**Performance killer**: Threads modify independent variables that share a cache line.

```
Cache line (64 bytes):
[ var1 | var2 | ... ]
   ↑       ↑
Thread1  Thread2

Thread1 writes var1 → entire cache line locked → Thread2 blocked
Thread2 writes var2 → entire cache line locked → Thread1 blocked
```

**Detection**: Use profiler showing high cache coherence traffic.

**Solution**: Align variables to cache line boundaries.

```cpp
struct alignas(hardware_destructive_interference_size) Data {
    atomic<int> var1;
};

struct alignas(hardware_destructive_interference_size) Data2 {
    atomic<int> var2;
};
```

**When not to worry**: If variables are read-only or truly shared (not false).

---

## Coroutines

### Coroutines vs Threads vs Async

**Coroutines are cooperative, stackless functions** that suspend without blocking threads. They are **100-1000x lighter** than threads.

```cpp
// Threads: 2-8MB stack each, preemptive scheduling
std::thread t(some_work); // OS thread created

// Async: Uses OS threads underneath
auto future = std::async(some_work); // May create new thread

// Coroutines: ~100 bytes, cooperative scheduling
cppcoro::task<void> task = some_coroutine(); // No thread created
co_await task; // Runs on current thread, suspends at co_await
```

**Scalability**:
- Threads: ~1000 max
- Async tasks: ~1000 max
- Coroutines: **>1,000,000 max**

---

### co_await, co_return, co_yield

**`co_await`** - Suspend until awaitable is ready:
```cpp
cppcoro::task<int> fetch() {
    int data = co_await async_read(); // SUSPENDS here
    // Resumes later when data arrives
    co_return data;
}
```

**`co_return`** - Return value from coroutine:
```cpp
cppcoro::task<int> compute() {
    co_return 42; // Sets task result
}
```

**`co_yield`** - Produce value and suspend (generators):
```cpp
cppcoro::generator<int> fibonacci() {
    int a = 0, b = 1;
    while (true) {
        co_yield a; // Return a, resume here next call
        int next = a + b;
        a = b;
        b = next;
    }
}
```

---

### Hybrid Patterns: Combining Coroutines with Threads/Async

**Pattern 1: CPU-heavy work inside I/O coroutine**
```cpp
cppcoro::task<void> mixed_work() {
    // I/O (coroutine)
    auto data = co_await async_read();
    
    // CPU (threads)
    auto result = co_await cppcoro::schedule_on(cpu_pool, [&]{
        return heavy_calc(data); // Runs on thread pool
    });
    
    // I/O again
    co_await async_write(result);
}
```

**Pattern 2: async inside coroutine**
```cpp
cppcoro::task<int> parallel_work() {
    // Launch multiple async tasks (true parallelism)
    std::future<int> f1 = std::async(std::launch::async, []{ return work1(); });
    std::future<int> f2 = std::async(std::launch::async, []{ return work2(); });
    
    // Wait for both (coroutine orchestration)
    int r1 = co_await cppcoro::wait_for_future(f1);
    int r2 = co_await cppcoro::wait_for_future(f2);
    
    co_return r1 + r2;
}
```

**Standard Library Generators**:
```cpp
generator<int> getSequenceGenerator(int start, int count) {
    for (int i = start; i < start + count; ++i) {
        println("Generating {}", i);
        co_yield i; // Suspend, return value
    }
}

int main() {
    auto gen = getSequenceGenerator(10, 5);
    for (int value : gen) {
        println("Got: {}", value);
        cin.ignore(); // Resume on next iteration
    }
}
```

---

## std::optional

### The Sentinel Value Problem

Before `std::optional`, representing "no value" was error-prone:

```cpp
// ❌ Sentinel values collide
int find_user(int id) { if (not_found) return -1; }
// What if -1 is a valid ID?

// ❌ Pointers have ownership issues
User* find_user(int id) { if (not_found) return nullptr; }
// Who deletes? Memory leak risk

// ❌ Exceptions are slow and can't be used everywhere
User find_user(int id) { if (not_found) throw; }
```

---

### The Solution

```cpp
// Returns "maybe a User"
std::optional<User> find_user(int id) {
    if (not_found) return std::nullopt; // No value
    return user; // Has value
}

// Usage is explicit and safe
if (auto user = find_user(5)) {
    std::cout << user->name; // operator* for access
}
```

**Benefits**:
- **Type-safe**: `optional<User>` ≠ `User`
- **No heap allocation**: Stores `User` inside itself
- **Clear intent**: "This might not return a value"

---

### When to Use std::optional

1. **Functions that may not return a value**
2. **Lazy initialization**
3. **Optional configuration parameters**
4. **Cache lookups**
5. **Avoiding default construction**

**When NOT to use**:
- Error handling (use `std::expected` in C++23)
- Polymorphic objects (use `unique_ptr<Base>`)
- Very large objects (consider `optional<unique_ptr<T>>`)

**Example**:
```cpp
optional<Config> load_config(const string& path) {
    if (!file_exists(path)) return std::nullopt;
    return Config{path};
}

void process() {
    auto config = load_config("settings.json");
    if (!config) {
        config = Config{default_settings}; // Use default
    }
    // Use *config
}
```

---

## Real-World Example: Multithreaded Logger

A complete example showing threads, mutexes, condition variables, and graceful shutdown.

```cpp
class Logger {
public:
    Logger() {
        // Start background thread
        m_thread = thread{&Logger::processEntries, this};
    }
    
    ~Logger() {
        {
            lock_guard lock{m_mutex};
            m_exit = true;
        }
        m_condVar.notify_all(); // Wake thread
        m_thread.join();        // Wait for finish
    }
    
    void log(string entry) {
        lock_guard lock{m_mutex};
        m_queue.push(move(entry));
        m_condVar.notify_one(); // Wake thread
    }

private:
    void processEntries() {
        ofstream logFile{"log.txt"};
        unique_lock lock{m_mutex};
        
        while (true) {
            // Wait with predicate to handle spurious wakeups
            m_condVar.wait(lock, [this] {
                return m_exit || !m_queue.empty();
            });
            
            if (m_exit && m_queue.empty()) break;
            
            // Swap queue to process outside lock
            queue<string> localQueue;
            localQueue.swap(m_queue);
            lock.unlock();
            
            while (!localQueue.empty()) {
                logFile << localQueue.front() << endl;
                localQueue.pop();
            }
            
            lock.lock(); // Reacquire lock for next iteration
        }
    }
    
    mutex m_mutex;
    condition_variable m_condVar;
    queue<string> m_queue;
    thread m_thread;
    bool m_exit = false;
};

// Usage
void logMessages(int id, Logger& logger) {
    for (int i = 0; i < 10; ++i) {
        logger.log(format("Thread {}: {}", id, i));
        this_thread::sleep_for(50ms);
    }
}

int main() {
    Logger logger;
    vector<jthread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(logMessages, i, ref(logger));
    }
    // Destructor joins all threads
    // Logger destructor stops background thread gracefully
}
```

**Key lessons**:
- Use RAII for locks
- Process queue outside lock to avoid blocking producers
- Graceful shutdown with condition variable + exit flag
- `jthread` preferred for automatic cleanup

---

## Best Practices

### Choosing Concurrency Tools

| Task | Tool | Why |
|------|------|-----|
| Return value from async work | `std::async` | Automatic result transport |
| Long-running daemon | `std::jthread` | Auto-cleanup |
| CPU parallelism (all cores) | `std::thread` | Direct hardware control |
| Massive I/O concurrency (>1000) | **Coroutines** | Lightweight, readable |
| Producer/consumer queue | `condition_variable` | Reusable signaling |
| One-time async result | `promise/future` | Simple value transport |
| Optional return value | `std::optional` | Type-safe "maybe" |
| Limited resource access | `semaphore` | Control concurrency |
| Multi-phase synchronization | `barrier` | Reusable coordination |
| Single-phase coordination | `latch` | Simple countdown |
| One-time initialization | `call_once` | Thread-safe singleton |

---

### Common Pitfalls

1. **Forgetting to join threads** → Use `std::jthread`
2. **Using `static_cast` for down-casting** → Use `dynamic_cast` (polymorphic) or redesign
3. **Manual lock/unlock** → Always use RAII (`lock_guard`, `unique_lock`)
4. **Condition variable without predicate** → Always check condition in loop
5. **`std::optional` for errors** → Use `std::expected` (C++23)
6. **Coroutines for CPU parallelism** → They don't use multiple cores alone
7. **Detaching threads with shared state** → Dangling references = UB
8. **False sharing** → Align variables to cache lines
9. **Double-checked locking** → Use `call_once()` or magic statics
10. **Holding locks too long** → Keep critical sections minimal

---

### Threading Design Guidelines

**1. Use parallel Standard Library algorithms**
```cpp
// Instead of manual threads
std::transform(std::execution::par, begin, end, out, func);
```

**2. Minimize shared data**
- Design for single-thread ownership
- Pass data by value
- Immutable data is inherently thread-safe

**3. Prefer atomics over locks** (when possible)
```cpp
atomic<int> counter{0}; // Faster than mutex
```

**4. Use profiling tools**
- Visual Studio Concurrency Visualizer
- Intel VTune
- perf + flame graphs on Linux

**5. Understand debugger features**
- View threads in debugger
- Freeze/thaw threads
- Check for deadlocks

**6. Use higher-level libraries**
- Intel TBB (Threading Building Blocks)
- Microsoft PPL
- Boost.Asio
- cppcoro (coroutines)

**7. Test with thread sanitizers**
```bash
g++ -fsanitize=thread myprogram.cpp # Detects races at runtime
```

**8. Document thread safety**
```cpp
// Thread-safe: multiple readers/writers
void add_user(const User& u);

// Not thread-safe: external synchronization required
void modify_user(User& u);
```

**9. Start simple**
- Begin with `std::async` and `jthread`
- Add condition variables when needed
- Use atomics for counters
- Only use raw `std::thread` when necessary

**10. Read entire books on multithreading**
This chapter only scratches the surface. See *C++ Concurrency in Action* by Anthony Williams for deep coverage.

---

## Performance Comparison

| Operation | Latency | Memory | Max Instances | Use Case |
|-----------|---------|--------|---------------|----------|
| `std::thread` creation | 10-100 µs | 2-8 MB | ~1,000 | CPU-bound parallelism |
| `std::async` launch | 10-100 µs | 2-8 MB | ~1,000 | Task-based parallelism |
| Coroutine creation | 0.1-1 µs | ~100 bytes | **>1,000,000** | Massive I/O concurrency |
| Mutex lock (uncontended) | 20-50 cycles | 56 bytes | - | Rare contention |
| Mutex lock (contended) | 1000-5000 cycles | 56 bytes | - | High contention |
| `condition_variable` wait | 1-10 µs | 48 bytes | - | Producer/consumer |
| `std::optional<int>` | 0 cycles (inlined) | 8 bytes | - | Optional values |
| `std::atomic<int>` increment | 10-100 cycles | 4 bytes | - | Lock-free counters |
| `std::semaphore` acquire | 20-200 cycles | 4 bytes | - | Resource limiting |

**Key insight**: Coroutines enable **1,000x more concurrency** than threads for I/O-bound work. Atomics are **10-100x faster** than mutexes for simple operations.

---

## Appendix: Quick Reference

### Casting Cheatsheet
```cpp
static_cast: Compile-time, trust-based, no overhead
dynamic_cast: Runtime-checked, polymorphic only, ~50-200 cycles
const_cast: Add/remove const only, use sparingly
reinterpret_cast: Bit pattern reinterpretation (dangerous)
```

### Threading Cheatsheet
```cpp
std::jthread t([]{}); // Modern default - auto-joins
t.join();  // Wait for completion
t.detach(); // Let it run independently (rarely needed)
t.request_stop(); // Cooperative cancellation
```

### Synchronization Cheatsheet
```cpp
std::mutex mtx;
std::lock_guard lock(mtx); // RAII lock

std::condition_variable cv;
cv.wait(lock, predicate); // Efficient wait

std::promise<int> prom;
std::future<int> fut = prom.get_future();

std::latch latch{count};
latch.count_down(); latch.wait();

std::barrier barrier{count, callback};

std::counting_semaphore sem{max};
sem.acquire(); sem.release();
```

### Coroutine Cheatsheet
```cpp
cppcoro::task<int> coro() {
    int val = co_await async_op(); // Suspend
    co_return val; // Resume later
}

cppcoro::generator<int> gen() {
    co_yield value; // Return and suspend
}
```

### Optional Cheatsheet
```cpp
std::optional<User> user = find_user(id);
if (user) { *user; } // Has value
user.value_or(default_user); // Default if empty
user.has_value(); // Check explicitly
```

---

## Conclusion

Modern C++ concurrency provides a **layered toolbox**:

1. **Type casting**: Use `static_cast` by default, `dynamic_cast` for polymorphic safety
2. **Threads**: `std::jthread` for safety, `std::thread` for control
3. **Thread pools**: For high-throughput task processing (use TBB/PPL)
4. **Async/Futures**: For task-based parallelism with automatic result transport
5. **Mutexes/CVs**: For low-level synchronization (prefer RAII wrappers)
6. **Atomics**: For lock-free simple operations
7. **Latches/Barriers/Semaphores**: For coordination patterns
8. **Coroutines**: For massive I/O concurrency on few threads
9. **Optional**: For type-safe "maybe" values

**The modern way**: Use **coroutines for I/O**, **thread pools for throughput**, and **jthread for services**. Always prefer **type-safe abstractions** over manual implementation.