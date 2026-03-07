# Multi-Threading — Advanced C++

## Thread Basics (Quick Recall)

```cpp
#include <thread>

void task(int id) { std::cout << "Thread " << id << "\n"; }

std::thread t1(task, 1);   // Start thread
std::thread t2(task, 2);

t1.join();                  // Main waits for t1 to finish
t2.detach();                // t2 runs independently (daemon-like)

std::cout << "Hardware concurrency: " << std::thread::hardware_concurrency();
```

---

## Mutex Types — When to Use Which

```cpp
#include <mutex>
#include <shared_mutex>

// 1. std::mutex — Basic exclusive lock
std::mutex m;
{
    std::lock_guard<std::mutex> lg(m);  // Lock on construction, unlock on destruction (RAII)
    // Critical section
}

// 2. std::unique_lock — Flexible: can unlock early, defer, or transfer lock
std::unique_lock<std::mutex> ul(m);   // Locked immediately
ul.unlock();                          // Manual unlock
ul.lock();                            // Re-lock
// OR deferred: 
std::unique_lock<std::mutex> ul2(m, std::defer_lock);
ul2.lock();   // Lock later

// 3. std::shared_mutex — Multiple readers OR single writer
std::shared_mutex rwMutex;

// Reader: many can read simultaneously
std::shared_lock<std::shared_mutex> readLock(rwMutex);   // Shared (read) lock

// Writer: exclusive access
std::unique_lock<std::shared_mutex> writeLock(rwMutex);  // Exclusive (write) lock

// 4. std::recursive_mutex — Same thread can lock multiple times
std::recursive_mutex rm;
void recursiveFunc(int n) {
    std::lock_guard<std::recursive_mutex> lg(rm);  // OK to lock again in recursion
    if (n > 0) recursiveFunc(n - 1);
}
```

---

## Condition Variable — Thread Synchronization

> Worker waits for data. Producer notifies when data is available.

```cpp
#include <condition_variable>
#include <queue>

std::mutex              mtx;
std::condition_variable cv;
std::queue<int>         dataQueue;
bool                    done = false;

// Producer thread
void producer() {
    for (int i = 0; i < 5; i++) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        {
            std::lock_guard<std::mutex> lg(mtx);
            dataQueue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        cv.notify_one();  // Wake one waiting consumer
    }
    { std::lock_guard<std::mutex> lg(mtx); done = true; }
    cv.notify_all();      // Wake all waiting consumers to check done flag
}

// Consumer thread
void consumer() {
    while (true) {
        std::unique_lock<std::mutex> ul(mtx);
        // Wait: release lock, sleep until condition is true
        cv.wait(ul, []{ return !dataQueue.empty() || done; }); // Spurious wake protection

        while (!dataQueue.empty()) {
            int val = dataQueue.front();
            dataQueue.pop();
            std::cout << "Consumed: " << val << "\n";
        }
        if (done) break;
    }
}

// Usage
std::thread p(producer);
std::thread c(consumer);
p.join(); c.join();
```

---

## std::atomic — Lock-Free Shared Data

```cpp
#include <atomic>

// For simple shared counters/flags — faster than mutex for primitives
std::atomic<int>  counter{0};
std::atomic<bool> ready{false};

void increment() {
    for (int i = 0; i < 1000; i++)
        counter++;  // Atomic increment — no mutex needed, no race condition
}

std::thread t1(increment), t2(increment);
t1.join(); t2.join();
std::cout << counter;  // Always 2000 — no lost updates

// Common use: flag to signal threads
std::atomic<bool> shutdownFlag{false};

void worker() {
    while (!shutdownFlag.load()) {  // .load() for explicit atomic read
        // Do work
    }
}
void main_logic() {
    std::thread w(worker);
    std::this_thread::sleep_for(std::chrono::seconds(1));
    shutdownFlag.store(true);   // .store() for explicit atomic write
    w.join();
}
```

---

## std::future and std::promise — Async Result Passing

```cpp
#include <future>

// Promise: one-time communication channel between threads
std::promise<int>  promise;
std::future<int>   future = promise.get_future();

// Producer thread
std::thread producer([&promise] {
    // Do computation...
    int result = 42;
    promise.set_value(result);  // Send result to future
});

// Consumer: blocks until value is ready
int result = future.get();      // Blocks here until producer calls set_value()
std::cout << result;            // 42
producer.join();

// ----- std::async (simpler) -----
// Runs function asynchronously, returns future automatically
auto fut = std::async(std::launch::async, []() {
    return 2 + 2;
});

// Do other work here while async runs...
int r = fut.get();   // Block until done, get result: 4
```

---

## std::async Launch Policies

```cpp
// std::launch::async — Run in new thread immediately
auto f1 = std::async(std::launch::async, heavyTask);

// std::launch::deferred — Run lazily in calling thread when .get() is called
auto f2 = std::async(std::launch::deferred, heavyTask);
// heavyTask hasn't run yet
f2.get();  // Now heavyTask runs synchronously here

// Default (async | deferred) — Implementation decides (avoid this ambiguity)
auto f3 = std::async(heavyTask);
```

---

## Deadlock — Causes and Prevention

```cpp
// DEADLOCK SCENARIO
std::mutex m1, m2;

void thread1() {
    std::lock_guard<std::mutex> lg1(m1);   // Lock m1
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::lock_guard<std::mutex> lg2(m2);   // Wait for m2 (thread2 holds m2, waiting for m1)
}

void thread2() {
    std::lock_guard<std::mutex> lg2(m2);   // Lock m2
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    std::lock_guard<std::mutex> lg1(m1);   // Wait for m1 (thread1 holds m1, waiting for m2)
}
// Both threads wait forever → DEADLOCK

// ---- SOLUTION 1: Always lock in the same order
// Thread1 and Thread2 both lock m1 then m2 → no circular wait

// ---- SOLUTION 2: std::lock() + std::adopt_lock — atomic multi-lock
void safeThread1() {
    std::lock(m1, m2);                           // Lock BOTH atomically — either both or neither
    std::lock_guard<std::mutex> l1(m1, std::adopt_lock);  // Adopt already-locked m1
    std::lock_guard<std::mutex> l2(m2, std::adopt_lock);  // Adopt already-locked m2
}

// ---- SOLUTION 3: std::scoped_lock (C++17) — easiest
void safeThread2() {
    std::scoped_lock lock(m1, m2);  // Locks both atomically, unlocks on scope exit
    // Critical section
}
```

---

## Thread-Local Storage

```cpp
// Each thread gets its own copy of the variable
thread_local int threadId = 0;

void worker(int id) {
    threadId = id;           // Only affects THIS thread's copy
    std::cout << threadId;   // Prints this thread's own value
}

std::thread t1(worker, 1);
std::thread t2(worker, 2);
// No sharing, no mutex needed — each thread has independent threadId
```

---

## Producer-Consumer with std::queue (Bounded Buffer)

```cpp
class BoundedQueue {
    std::queue<int>          q;
    std::mutex               mtx;
    std::condition_variable  notFull;   // Signal: space available
    std::condition_variable  notEmpty;  // Signal: data available
    size_t                   maxSize;

public:
    BoundedQueue(size_t max) : maxSize(max) {}

    void push(int val) {
        std::unique_lock<std::mutex> ul(mtx);
        notFull.wait(ul, [this]{ return q.size() < maxSize; }); // Wait if full
        q.push(val);
        notEmpty.notify_one();  // Signal consumer: data is ready
    }

    int pop() {
        std::unique_lock<std::mutex> ul(mtx);
        notEmpty.wait(ul, [this]{ return !q.empty(); }); // Wait if empty
        int val = q.front();
        q.pop();
        notFull.notify_one();   // Signal producer: space is available
        return val;
    }
};
```

---

## Common Multi-Threading Pitfalls

```cpp
// 1. Data Race — two threads write same var without sync
int x = 0;
// Thread A: x++          (read, add, write)
// Thread B: x++          (read, add, write)
// Result: could be 1 instead of 2 — non-atomic operation
// Fix: std::atomic<int> x = 0; or use mutex

// 2. Forgetting to join or detach (program terminates thread)
{
    std::thread t(task);
    // Forgot t.join() or t.detach()
}  // std::terminate() called here! Thread destroyed while running.
// Fix: always join/detach, or use RAII thread wrapper

// 3. Lambda capturing by reference in thread
int val = 10;
std::thread t([&val]{ std::cout << val; });  // val may be destroyed before thread runs!
t.join();
// Fix: capture by value [val] or ensure lifetime

// 4. Spurious wakeups with condition_variable — ALWAYS use predicate
cv.wait(lock);                                // WRONG: can wake for no reason
cv.wait(lock, []{ return dataReady; });       // CORRECT: re-checks condition
```
