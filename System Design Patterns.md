# System Design Patterns in C++

## Logger Design

### Concepts

- A logger captures runtime events (info, warnings, errors) and writes them to sinks (console, file, network).
- Key concerns: **thread safety**, **performance** (async), **log rotation**, **severity filtering**.

### Basic Logger

```cpp
#include <fstream>
#include <string>
#include <ctime>

enum class LogLevel { DEBUG, INFO, WARNING, ERROR };

class Logger {
    std::ofstream file;
    LogLevel      minLevel;

    std::string timestamp() {
        std::time_t t = std::time(nullptr);
        char buf[20]; std::strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", std::localtime(&t));
        return buf;
    }
    std::string levelStr(LogLevel l) {
        switch (l) {
            case LogLevel::DEBUG:   return "DEBUG";
            case LogLevel::INFO:    return "INFO";
            case LogLevel::WARNING: return "WARN";
            case LogLevel::ERROR:   return "ERROR";
        }
        return "";
    }
public:
    Logger(const std::string& path, LogLevel min = LogLevel::DEBUG)
        : file(path, std::ios::app), minLevel(min) {}

    void log(LogLevel level, const std::string& msg) {
        if (level < minLevel) return;  // Filter below threshold
        file << "[" << timestamp() << "] [" << levelStr(level) << "] " << msg << "\n";
        file.flush();
    }
};
```

### Thread-Safe Logging

- Protect the log write with a `std::mutex` to prevent interleaved output from multiple threads.

```cpp
#include <mutex>

class ThreadSafeLogger {
    std::ofstream file;
    std::mutex    mtx;
public:
    ThreadSafeLogger(const std::string& path) : file(path, std::ios::app) {}

    void log(const std::string& msg) {
        std::lock_guard<std::mutex> lock(mtx);  // Only one thread writes at a time
        file << msg << "\n";
        file.flush();
    }
};
```

### Async Logging

- Writing to disk can be slow. Async logging: **producer threads** push to a queue; a **dedicated worker thread** drains the queue to disk. Producers never block on I/O.

```cpp
#include <thread>
#include <queue>
#include <condition_variable>
#include <atomic>

class AsyncLogger {
    std::queue<std::string>  queue;
    std::mutex               mtx;
    std::condition_variable  cv;
    std::atomic<bool>        running{true};
    std::thread              worker;
    std::ofstream            file;

    void workerLoop() {
        while (running || !queue.empty()) {
            std::unique_lock<std::mutex> lock(mtx);
            cv.wait(lock, [this]{ return !queue.empty() || !running; });  // Wait for work
            while (!queue.empty()) {
                file << queue.front() << "\n"; queue.pop();  // Drain queue to disk
            }
            file.flush();
        }
    }
public:
    AsyncLogger(const std::string& path) : file(path, std::ios::app),
        worker([this]{ workerLoop(); }) {}

    void log(const std::string& msg) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            queue.push(msg);  // Producers only push — never touch the file
        }
        cv.notify_one();  // Wake the worker
    }

    ~AsyncLogger() {
        running = false; cv.notify_all();  // Signal shutdown
        if (worker.joinable()) worker.join();
    }
};
```

### Log Rotation

- When log file exceeds a size limit, close it and open a new one (optionally with a timestamp suffix).

```cpp
class RotatingLogger {
    std::string   basePath;
    std::ofstream file;
    size_t        maxBytes;
    size_t        bytesWritten = 0;

    void rotate() {
        file.close();
        // Rename old file with timestamp
        std::string newName = basePath + "." + std::to_string(std::time(nullptr)) + ".bak";
        std::rename(basePath.c_str(), newName.c_str());
        file.open(basePath, std::ios::app);  // Fresh file
        bytesWritten = 0;
    }
public:
    RotatingLogger(const std::string& path, size_t maxSz = 1024*1024)
        : basePath(path), file(path, std::ios::app), maxBytes(maxSz) {}

    void log(const std::string& msg) {
        if (bytesWritten + msg.size() > maxBytes) rotate();  // Rotate if over limit
        file << msg << "\n"; file.flush();
        bytesWritten += msg.size() + 1;
    }
};
```

---

## Thread Pool Design

### Concepts

- A **thread pool** pre-creates a fixed number of worker threads that wait for tasks.
- Tasks are submitted to a shared **task queue**; workers dequeue and execute them.
- Avoids the overhead of creating/destroying threads per task.

```
Producer → [Task Queue] → Worker Thread 1
                        → Worker Thread 2
                        → Worker Thread N
```

### Task Queue Design

- Thread-safe queue protected by a `mutex` + `condition_variable`.
- Workers wait (`cv.wait`) when queue is empty — avoids **busy waiting**.

### Worker Thread

- Each worker loops: lock queue → dequeue task → unlock → execute task.
- Exits cleanly when `stop` signal received and queue is drained.

### Full Thread Pool Implementation

```cpp
#include <thread>
#include <queue>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <vector>
#include <future>
#include <atomic>
#include <stdexcept>

class ThreadPool {
    std::vector<std::thread>          workers;
    std::queue<std::function<void()>> taskQueue;
    std::mutex                        mtx;
    std::condition_variable           cv;
    std::atomic<bool>                 stop{false};
    size_t                            maxQueueSize;

public:
    explicit ThreadPool(size_t numThreads, size_t maxQ = SIZE_MAX)
        : maxQueueSize(maxQ)
    {
        for (size_t i = 0; i < numThreads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx);
                        // Wait until there is a task OR stop is requested
                        cv.wait(lock, [this]{ return stop || !taskQueue.empty(); });
                        if (stop && taskQueue.empty()) return;  // Graceful shutdown
                        task = std::move(taskQueue.front());
                        taskQueue.pop();
                    }
                    cv.notify_one();  // Notify producers waiting due to full queue
                    try { task(); }   // Execute task outside the lock
                    catch (const std::exception& e) {
                        // Log exception — do NOT let it terminate the worker thread
                        fprintf(stderr, "Task threw: %s\n", e.what());
                    }
                }
            });
        }
    }

    // Submit a task; returns std::future to get the result later
    template <typename F, typename... Args>
    auto submit(F&& f, Args&&... args) -> std::future<std::invoke_result_t<F, Args...>> {
        using RetType = std::invoke_result_t<F, Args...>;
        auto task = std::make_shared<std::packaged_task<RetType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );
        std::future<RetType> result = task->get_future();
        {
            std::unique_lock<std::mutex> lock(mtx);
            // Block producer if queue is full — limits unbounded queue growth
            cv.wait(lock, [this]{ return taskQueue.size() < maxQueueSize || stop; });
            if (stop) throw std::runtime_error("ThreadPool stopped");
            taskQueue.emplace([task]{ (*task)(); });
        }
        cv.notify_one();  // Wake a sleeping worker
        return result;
    }

    // Graceful shutdown — wait for all tasks to finish
    ~ThreadPool() {
        stop = true;
        cv.notify_all();  // Wake all workers so they can check stop flag
        for (auto& t : workers)
            if (t.joinable()) t.join();
    }
};

// Usage
ThreadPool pool(4);  // 4 worker threads
auto fut = pool.submit([](int x){ return x * x; }, 7);
int result = fut.get();  // 49 — blocks until task completes
```

### Condition Variables — How to Avoid Busy Waiting

- **Busy waiting**: `while (!ready) {}` — wastes CPU spinning in a tight loop.
- **Fix**: `cv.wait(lock, predicate)` — atomically releases lock and suspends the thread until `cv.notify_*` is called AND predicate is true. Uses zero CPU while waiting.

```cpp
std::mutex mtx; std::condition_variable cv; bool ready = false;

// Consumer (waits for data)
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, []{ return ready; });  // Releases lock and sleeps; re-locks when notified

// Producer (sets data and signals)
{ std::lock_guard<std::mutex> lock(mtx); ready = true; }
cv.notify_one();  // Wake one waiting consumer
```

### Work Stealing

- Each thread has its **own local deque**. When a thread's deque is empty, it **steals** from the back of another thread's deque.
- Reduces contention on a single shared queue.

```
Thread 1 Deque: [T1, T2, T3]  ← Thread 1 pops from front
Thread 2 Deque: [T4, T5]      ← Thread 2 pops from front; if empty, steals T3 from Thread 1's back
```

### What If a Task Spawns New Tasks?

- Submit the new task back to the pool via the same `submit()` call.
- Beware of **deadlock**: if all threads are waiting for their sub-tasks and the queue is full (maxQueueSize is set), no thread is free to execute the sub-tasks.
- Solution: use **unbounded queues** or **separate priority lanes** for spawned tasks, or use a work-stealing pool.

### How to Prioritise Tasks

```cpp
// Use a priority queue instead of std::queue as the task queue
struct Task {
    int                    priority;
    std::function<void()>  fn;
    bool operator<(const Task& o) const { return priority < o.priority; }  // Lower priority first (min-heap)
};
std::priority_queue<Task> pq;  // Higher number = higher priority; served first
```

### Mutex, Atomic, Spinlock

| Mechanism | When to Use |
|---|---|
| `std::mutex` | General mutual exclusion; contended sections > a few microseconds |
| `std::atomic<T>` | Single variable updates (counters, flags). Lock-free, no OS overhead |
| Spinlock | Very short critical sections; waiting thread spins (busy waits). Bad for long waits |

```cpp
// Spinlock — no OS involvement; fast for tiny critical sections
class SpinLock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    void lock()   { while (flag.test_and_set(std::memory_order_acquire)) {} }  // Spin until free
    void unlock() { flag.clear(std::memory_order_release); }
};
```

### Deadlock, Race Condition, Starvation, False Sharing

| Problem | Description | Solution |
|---|---|---|
| **Deadlock** | Thread A waits for B's lock; B waits for A's lock. Circular wait. | Lock in consistent order; use `std::lock()` to lock multiple mutexes atomically |
| **Race Condition** | Outcome depends on thread execution order. | Protect shared data with mutexes or atomics |
| **Starvation** | A thread never gets scheduled / gets a lock. | Use fair locks; priority scheduling |
| **False Sharing** | Two threads write to different variables that share a cache line — forces cache invalidation. | Align hot variables to cache line (`alignas(64)`) |

```cpp
// False sharing — counter0 and counter1 likely share a 64-byte cache line
struct Counters { int counter0; int counter1; };

// Fix — pad each to its own cache line
struct alignas(64) PaddedCounter { int value; };
PaddedCounter c0, c1;  // c0 and c1 are on separate cache lines
```

---

## Producer-Consumer Pattern

```cpp
// Bounded buffer: producers wait when full; consumers wait when empty
template <typename T, size_t Cap>
class BoundedChannel {
    std::queue<T>           buf;
    std::mutex              mtx;
    std::condition_variable notFull, notEmpty;
public:
    void push(T item) {
        std::unique_lock<std::mutex> lock(mtx);
        notFull.wait(lock, [this]{ return buf.size() < Cap; });  // Wait if full
        buf.push(std::move(item));
        notEmpty.notify_one();  // Signal consumer
    }
    T pop() {
        std::unique_lock<std::mutex> lock(mtx);
        notEmpty.wait(lock, [this]{ return !buf.empty(); });     // Wait if empty
        T item = std::move(buf.front()); buf.pop();
        notFull.notify_one();  // Signal producer
        return item;
    }
};
```

---

## Reader-Writer Locks

- Multiple readers can read simultaneously; a writer needs exclusive access.
- C++17: `std::shared_mutex` provides `lock()` (exclusive) and `lock_shared()` (shared).

```cpp
#include <shared_mutex>
std::shared_mutex rwMtx;
std::string       sharedData;

void reader() {
    std::shared_lock<std::shared_mutex> lock(rwMtx);  // Shared — allows multiple readers
    std::cout << sharedData;
}
void writer(const std::string& newData) {
    std::unique_lock<std::shared_mutex> lock(rwMtx);  // Exclusive — blocks all readers/writers
    sharedData = newData;
}
```

---

## Double-Checked Locking

- Optimisation for lazy initialisation: check before locking to avoid lock overhead on the fast path.
- **Must** use `std::atomic` or `std::once_flag`; raw pointer check is not thread-safe.

```cpp
#include <mutex>
#include <atomic>

class Singleton {
    static std::atomic<Singleton*> instance;
    static std::mutex              mtx;
    Singleton() {}
public:
    static Singleton* getInstance() {
        Singleton* p = instance.load(std::memory_order_acquire);  // Fast read — acquired
        if (!p) {
            std::lock_guard<std::mutex> lock(mtx);
            p = instance.load(std::memory_order_relaxed);         // Re-check under lock
            if (!p) {
                p = new Singleton();
                instance.store(p, std::memory_order_release);     // Publish to other threads
            }
        }
        return p;
    }
};
std::atomic<Singleton*> Singleton::instance{nullptr};
std::mutex Singleton::mtx;

// Simpler alternative — C++11 guarantees static local is initialised exactly once
Singleton& getInstance2() {
    static Singleton inst;  // Thread-safe by C++11 standard
    return inst;
}
```

---

## File Parser Design

### Streaming vs Full File Load

| | Streaming (Incremental) | Full Load |
|---|---|---|
| Memory | Fixed small buffer | Entire file in RAM |
| Latency | Start parsing immediately | Must wait for full load |
| Use when | Large files, limited memory | Small files, random access needed |

### Full File Load

```cpp
#include <fstream>
#include <sstream>
std::string loadFile(const std::string& path) {
    std::ifstream f(path);
    if (!f) throw std::runtime_error("Cannot open file");
    std::ostringstream ss;
    ss << f.rdbuf();  // Read entire file into string buffer
    return ss.str();
}
```

### Streaming / Incremental Parsing

```cpp
// Parse CSV line by line — O(1) memory for any file size
void parseCSVStreaming(const std::string& path) {
    std::ifstream file(path);
    if (!file) throw std::runtime_error("Cannot open");
    std::string line;
    while (std::getline(file, line)) {  // One line at a time
        std::istringstream ss(line);
        std::string token;
        while (std::getline(ss, token, ','))  // Split by comma
            processToken(token);              // Handle each field
    }
}
```

### Error Handling

```cpp
// Return variant/error code instead of throwing in tight loops
enum class ParseError { None, InvalidFormat, UnexpectedEOF, DuplicateKey };

struct ParseResult {
    bool       ok;
    ParseError error;
    std::string value;
};

ParseResult parseField(const std::string& raw) {
    if (raw.empty()) return {false, ParseError::InvalidFormat, {}};
    return {true, ParseError::None, raw};
}
```

### Visitor Pattern for Parsing

- Separate what to parse (parser) from what to do with parsed data (visitor).

```cpp
struct INodeVisitor {
    virtual void visitString(const std::string& key, const std::string& val) = 0;
    virtual void visitInt(const std::string& key, int val) = 0;
    virtual ~INodeVisitor() = default;
};

class PrintVisitor : public INodeVisitor {
public:
    void visitString(const std::string& k, const std::string& v) override {
        printf("%s = \"%s\"\n", k.c_str(), v.c_str());
    }
    void visitInt(const std::string& k, int v) override {
        printf("%s = %d\n", k.c_str(), v);
    }
};
```

### Builder Pattern for Parsing

- Incrementally populate a complex object as tokens are parsed.

```cpp
struct Config { std::string host; int port = 0; bool tls = false; };

class ConfigBuilder {
    Config cfg;
public:
    ConfigBuilder& setHost(const std::string& h) { cfg.host = h; return *this; }
    ConfigBuilder& setPort(int p)                { cfg.port = p; return *this; }
    ConfigBuilder& setTLS(bool t)                { cfg.tls  = t; return *this; }
    Config build() { return cfg; }
};

Config c = ConfigBuilder().setHost("localhost").setPort(8080).setTLS(true).build();
```

### Handling Large Files

1. **Stream line by line** — never load whole file.
2. **Memory-mapped files** — map file into virtual memory; OS pages in only accessed regions.
3. **Parallel parsing** — split file into chunks; parse each chunk on a separate thread; merge.
4. **Fixed-size read buffer** — read in blocks (e.g., 4KB), process, then advance.

```cpp
// Memory-mapped file (POSIX)
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>

void* mmapFile(const char* path, size_t& size) {
    int fd = open(path, O_RDONLY);
    struct stat st; fstat(fd, &st); size = st.st_size;
    void* data = mmap(nullptr, size, PROT_READ, MAP_PRIVATE, fd, 0);  // OS pages on demand
    close(fd);
    return data;  // Access like array; OS handles paging
}
```

---

## Plugin Architecture Using Dynamic Libraries

### Concept

- Plugins are **shared libraries** (`.dll` on Windows, `.so` on Linux) loaded at runtime.
- The application defines an interface (pure abstract class); plugins implement it.
- The app never recompiles when a new plugin is added — just drop in a new `.so`/`.dll`.

### Step 1 — Define Plugin Interface (SDK Header)

```cpp
// IPlugin.h — shipped as part of the SDK
class IPlugin {
public:
    virtual const char* getName() const = 0;
    virtual void        execute()       = 0;
    virtual ~IPlugin()                  = default;
};

// Every plugin must export these two C functions (extern "C" avoids name mangling)
extern "C" IPlugin* createPlugin();
extern "C" void     destroyPlugin(IPlugin*);
```

### Step 2 — Create a Plugin (Compiled as `.dll` / `.so`)

```cpp
// MyPlugin.cpp — compiled as a shared library
#include "IPlugin.h"
#include <cstdio>

class MyPlugin : public IPlugin {
public:
    const char* getName() const override { return "MyPlugin v1.0"; }
    void        execute()       override { printf("MyPlugin executing!\n"); }
};

extern "C" IPlugin* createPlugin()         { return new MyPlugin(); }
extern "C" void     destroyPlugin(IPlugin* p) { delete p; }
```

### Step 3 — Load Plugin at Runtime (Host Application)

```cpp
// PluginLoader.cpp — platform: Windows
#include <windows.h>
#include "IPlugin.h"

class PluginLoader {
    HMODULE   lib     = nullptr;
    IPlugin*  plugin  = nullptr;
public:
    void load(const char* path) {
        lib = LoadLibraryA(path);                                    // Load .dll
        if (!lib) throw std::runtime_error("LoadLibrary failed");

        auto create  = (IPlugin*(*)())  GetProcAddress(lib, "createPlugin");   // Get factory fn
        auto destroy = (void(*)(IPlugin*)) GetProcAddress(lib, "destroyPlugin");

        if (!create) throw std::runtime_error("createPlugin not found");
        plugin = create();                // Call plugin factory
    }
    void run()    { if (plugin) plugin->execute(); }
    const char* name() { return plugin ? plugin->getName() : ""; }

    ~PluginLoader() {
        if (plugin) { /* call destroyPlugin */ }
        if (lib) FreeLibrary(lib);  // Unload .dll
    }
};

// Usage
PluginLoader loader;
loader.load("MyPlugin.dll");
printf("Loaded: %s\n", loader.name());
loader.run();  // "MyPlugin executing!"
```

### Key Design Rules

- Always use `extern "C"` on factory functions to avoid C++ name mangling.
- **Never** pass C++ STL objects (e.g., `std::string`) across plugin boundaries — ABI may differ.
- Use `std::unique_ptr` with a custom deleter to manage plugin lifetime safely.

```cpp
// Safe RAII plugin management
using DestroyFn = void(*)(IPlugin*);
auto safePlugin = std::unique_ptr<IPlugin, DestroyFn>(createPlugin(), destroyPlugin);
```

---

## LRU Cache Design

- **LRU (Least Recently Used)**: when capacity is exceeded, evict the item that was accessed furthest in the past.
- Implementation: `std::list` (ordered by recency) + `std::unordered_map` (O(1) lookup by key).
- All operations: `O(1)`.

```cpp
#include <list>
#include <unordered_map>

class LRUCache {
    int capacity;
    std::list<std::pair<int,int>>                           items;   // {key, value} MRU at front
    std::unordered_map<int, std::list<std::pair<int,int>>::iterator> cache;  // key → iterator

public:
    LRUCache(int cap) : capacity(cap) {}

    int get(int key) {
        auto it = cache.find(key);
        if (it == cache.end()) return -1;                   // Cache miss
        items.splice(items.begin(), items, it->second);     // Move to front (most recent)
        return it->second->second;
    }

    void put(int key, int value) {
        auto it = cache.find(key);
        if (it != cache.end()) {
            it->second->second = value;
            items.splice(items.begin(), items, it->second);  // Update and move to front
            return;
        }
        if ((int)items.size() == capacity) {
            cache.erase(items.back().first);  // Remove LRU entry from map
            items.pop_back();                 // Remove from list
        }
        items.emplace_front(key, value);      // Insert at front (most recent)
        cache[key] = items.begin();
    }
};

LRUCache lru(2);
lru.put(1, 10); lru.put(2, 20);
lru.get(1);     // 10; 1 is now most recent
lru.put(3, 30); // Evicts 2 (LRU); cache: {1:10, 3:30}
lru.get(2);     // -1 (evicted)
```

---

## Rate Limiter Design

### Token Bucket Algorithm

- A bucket holds up to `capacity` tokens. Each request consumes 1 token. Tokens refill at `rate` per second.
- If the bucket is empty, the request is **rejected** (or queued).

```cpp
#include <chrono>
#include <mutex>

class TokenBucketRateLimiter {
    double   tokens;
    double   capacity;
    double   refillRate;  // tokens per second
    std::chrono::steady_clock::time_point lastRefill;
    std::mutex mtx;

    void refill() {
        auto now     = std::chrono::steady_clock::now();
        double secs  = std::chrono::duration<double>(now - lastRefill).count();
        tokens       = std::min(capacity, tokens + secs * refillRate);  // Refill, clamp to max
        lastRefill   = now;
    }

public:
    TokenBucketRateLimiter(double cap, double rate)
        : tokens(cap), capacity(cap), refillRate(rate),
          lastRefill(std::chrono::steady_clock::now()) {}

    bool allow() {
        std::lock_guard<std::mutex> lock(mtx);
        refill();
        if (tokens >= 1.0) { tokens -= 1.0; return true; }  // Consume 1 token
        return false;  // Bucket empty — reject
    }
};

// Usage
TokenBucketRateLimiter limiter(10, 5.0);  // Bucket of 10 tokens, refill 5/second
if (limiter.allow()) { /* process request */ }
else                 { /* reject: 429 Too Many Requests */ }
```

### Fixed Window Counter

- Count requests per fixed time window (e.g., per second). Reject if count exceeds limit.

```cpp
class FixedWindowLimiter {
    int      limit;
    int      count = 0;
    std::chrono::steady_clock::time_point windowStart;
    std::chrono::seconds windowSize;
    std::mutex mtx;
public:
    FixedWindowLimiter(int lim, std::chrono::seconds win)
        : limit(lim), windowSize(win), windowStart(std::chrono::steady_clock::now()) {}

    bool allow() {
        std::lock_guard<std::mutex> lock(mtx);
        auto now = std::chrono::steady_clock::now();
        if (now - windowStart >= windowSize) {  // New window — reset count
            count = 0; windowStart = now;
        }
        if (count < limit) { ++count; return true; }
        return false;
    }
};
```
