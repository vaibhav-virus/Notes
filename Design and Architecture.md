# Design & Architecture — Real-World C++ Systems

## Thread Pool Design

> Pre-create N worker threads. Submit tasks to a queue. Workers pick tasks and execute them.

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <vector>
#include <future>

class ThreadPool {
    std::vector<std::thread>          workers;       // Fixed pool of threads
    std::queue<std::function<void()>> tasks;         // Task queue
    std::mutex                        queueMutex;
    std::condition_variable           condition;
    bool                              stop = false;  // Shutdown flag

public:
    // Spawn N worker threads in constructor
    ThreadPool(size_t numThreads) {
        for (size_t i = 0; i < numThreads; i++) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        // Wait until there's a task or we're stopping
                        std::unique_lock<std::mutex> lock(queueMutex);
                        condition.wait(lock, [this] {
                            return stop || !tasks.empty(); // Wake up condition
                        });
                        if (stop && tasks.empty()) return;  // Shutdown gracefully
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();  // Execute task outside the lock
                }
            });
        }
    }

    // Enqueue a task and get a future for its result
    template<typename F, typename... Args>
    auto enqueue(F&& f, Args&&... args) -> std::future<decltype(f(args...))> {
        using ReturnType = decltype(f(args...));

        // Wrap the function in a packaged_task so we can return a future
        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        std::future<ReturnType> result = task->get_future();
        {
            std::lock_guard<std::mutex> lock(queueMutex);
            if (stop) throw std::runtime_error("ThreadPool is stopped");
            tasks.emplace([task]() { (*task)(); });
        }
        condition.notify_one();  // Wake one waiting worker
        return result;
    }

    // Destructor: signal stop, wake all, join all threads
    ~ThreadPool() {
        { std::lock_guard<std::mutex> lock(queueMutex); stop = true; }
        condition.notify_all();
        for (auto& w : workers) w.join();
    }
};

// Usage
ThreadPool pool(4);  // 4 worker threads

auto f1 = pool.enqueue([](int x) { return x * x; }, 5);
auto f2 = pool.enqueue([]() { return std::string("done"); });

std::cout << f1.get();  // 25 — blocks until result ready
std::cout << f2.get();  // "done"
```

**Key design decisions:**
- `condition_variable`: Avoid busy-waiting; workers sleep until work arrives
- `packaged_task` + `future`: Caller can wait for result asynchronously
- Shutdown: flush remaining tasks, then join

---

## Logger Design (Thread-Safe, Async)

```cpp
#include <mutex>
#include <fstream>
#include <sstream>
#include <chrono>

enum class LogLevel { DEBUG, INFO, WARN, ERROR };

class Logger {
    std::ofstream    file;
    std::mutex       logMutex;
    LogLevel         minLevel = LogLevel::DEBUG;

    Logger() : file("app.log", std::ios::app) {}

    std::string levelStr(LogLevel l) {
        switch(l) {
            case LogLevel::DEBUG: return "DEBUG";
            case LogLevel::INFO:  return "INFO ";
            case LogLevel::WARN:  return "WARN ";
            case LogLevel::ERROR: return "ERROR";
        }
        return "";
    }

    std::string timestamp() {
        auto now = std::chrono::system_clock::now();
        auto t   = std::chrono::system_clock::to_time_t(now);
        char buf[32];
        strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", localtime(&t));
        return buf;
    }

public:
    static Logger& getInstance() {
        static Logger instance;  // Meyers' Singleton — thread-safe in C++11
        return instance;
    }

    void setLevel(LogLevel l) { minLevel = l; }

    void log(LogLevel level, const std::string& message) {
        if (level < minLevel) return;           // Filter by minimum level

        std::lock_guard<std::mutex> lock(logMutex);  // Thread-safe write
        std::stringstream ss;
        ss << "[" << timestamp() << "] [" << levelStr(level) << "] " << message << "\n";
        std::cout << ss.str();                  // Console output
        file     << ss.str();                   // File output
        file.flush();                           // Flush immediately for crash safety
    }

    // Convenience macros style helpers
    void debug(const std::string& msg) { log(LogLevel::DEBUG, msg); }
    void info (const std::string& msg) { log(LogLevel::INFO,  msg); }
    void warn (const std::string& msg) { log(LogLevel::WARN,  msg); }
    void error(const std::string& msg) { log(LogLevel::ERROR, msg); }
};

// Usage
Logger::getInstance().info("Server started on port 8080");
Logger::getInstance().error("Connection failed");
```

---

## File Parser — CSV

```cpp
#include <fstream>
#include <sstream>
#include <vector>
#include <string>

struct Record {
    std::string name;
    int         age;
    double      score;
};

class CSVParser {
    std::string filename;

    std::vector<std::string> splitLine(const std::string& line, char delim = ',') {
        std::vector<std::string> tokens;
        std::stringstream ss(line);
        std::string token;
        while (std::getline(ss, token, delim))
            tokens.push_back(token);
        return tokens;
    }

public:
    CSVParser(const std::string& file) : filename(file) {}

    std::vector<Record> parse() {
        std::ifstream file(filename);
        if (!file.is_open())
            throw std::runtime_error("Cannot open: " + filename);

        std::vector<Record> records;
        std::string line;

        std::getline(file, line);  // Skip header row

        while (std::getline(file, line)) {
            if (line.empty()) continue;
            auto tokens = splitLine(line);
            if (tokens.size() < 3) continue;  // Skip malformed rows

            Record r;
            r.name  = tokens[0];
            r.age   = std::stoi(tokens[1]);
            r.score = std::stod(tokens[2]);
            records.push_back(r);
        }
        return records;
    }
};

// Usage
// CSV file: name,age,score
//           Alice,30,95.5
//           Bob,25,88.0
CSVParser parser("data.csv");
auto records = parser.parse();
for (const auto& r : records)
    std::cout << r.name << " " << r.age << " " << r.score << "\n";
```

---

## Memory Pool Design

```cpp
#include <cassert>

// Fixed-block memory pool — all allocations are the same size
class MemoryPool {
    size_t          blockSize;
    size_t          poolSize;
    std::vector<uint8_t> pool;    // Pre-allocated buffer
    std::vector<void*>   freeList;// Available blocks

public:
    MemoryPool(size_t blockSz, size_t count)
        : blockSize(blockSz), poolSize(count),
          pool(blockSz * count) {
        // Initialize free list — all blocks available
        for (size_t i = 0; i < count; i++)
            freeList.push_back(pool.data() + i * blockSize);
    }

    void* allocate() {
        if (freeList.empty())
            throw std::bad_alloc();    // Pool exhausted
        void* ptr = freeList.back();
        freeList.pop_back();
        return ptr;                    // Constant time — no OS call
    }

    void deallocate(void* ptr) {
        freeList.push_back(ptr);       // Just return to free list — no OS call
    }

    size_t available() const { return freeList.size(); }
};

// Usage
struct Particle { float x, y, z; float vx, vy, vz; };

MemoryPool pool(sizeof(Particle), 1000);  // Pre-allocate 1000 particle slots

Particle* p1 = static_cast<Particle*>(pool.allocate());
new (p1) Particle{0,0,0, 1,0,0};          // Placement new to construct

pool.deallocate(p1);                       // Return to pool — instant
```

---

## Static vs Dynamic Libraries

### Static Library (.lib / .a)

```
                  Compile Time
┌────────────┐       ┌────────────┐
│  main.cpp  │──────>│ my.lib/.a  │──────> app.exe (self-contained)
└────────────┘       └────────────┘
```

```bash
# Build static library
g++ -c mathlib.cpp -o mathlib.o      # Compile to object file
ar rcs libmath.a mathlib.o           # Archive into static lib (.a on Linux)

# Link with app
g++ main.cpp -L. -lmath -o app       # Linker copies lib code into app
# Result: app is completely self-contained — no dependency at runtime
```

**Pros:** Single binary, no runtime dependency
**Cons:** Larger executable, must recompile to update

---

### Dynamic Library (.dll / .so)

```
                  Run Time
┌────────────┐       ┌────────────┐
│   app.exe  │──────>│  math.dll  │  (loaded at runtime)
└────────────┘       └────────────┘
```

```bash
# Build shared library
g++ -shared -fPIC mathlib.cpp -o libmath.so   # -fPIC: position-independent code

# Link app (app stores a reference, not a copy)
g++ main.cpp -L. -lmath -o app
# Result: app needs libmath.so present at runtime
```

**Pros:** Smaller executable, update lib without recompiling app, shared across processes
**Cons:** Dependency at runtime ("DLL hell" if versions mismatch)

---

### DLL Exports on Windows (MSVC)

```cpp
// In library header
#ifdef BUILDING_DLL
  #define API __declspec(dllexport)  // When building the .dll
#else
  #define API __declspec(dllimport)  // When consuming the .dll
#endif

class API MathLib {
public:
    int add(int a, int b);
};
```

---

## Refactoring Concepts

### Extract Method

```cpp
// BEFORE: long, hard to test
void processOrder(Order& o) {
    // Validate
    if (o.qty <= 0 || o.price <= 0) throw std::invalid_argument("bad order");
    // Apply discount
    if (o.qty > 100) o.price *= 0.9;
    // Calculate total
    o.total = o.qty * o.price;
}

// AFTER: extracted into small, testable functions
bool isValid(const Order& o) { return o.qty > 0 && o.price > 0; }
void applyDiscount(Order& o)  { if (o.qty > 100) o.price *= 0.9; }
void calcTotal(Order& o)      { o.total = o.qty * o.price; }

void processOrder(Order& o) {
    if (!isValid(o)) throw std::invalid_argument("bad order");
    applyDiscount(o);
    calcTotal(o);
}
```

---

### Replace Conditional with Polymorphism

```cpp
// BEFORE: long if-else or switch — adding new types means editing this function
double calcArea(const Shape& s) {
    if (s.type == "circle")    return 3.14 * s.r * s.r;
    if (s.type == "rectangle") return s.w * s.h;
    return 0;
}

// AFTER: each class knows its own area — Open/Closed Principle
struct Shape    { virtual double area() const = 0; virtual ~Shape() = default; };
struct Circle   : Shape { double r; double area() const override { return 3.14*r*r; } };
struct Rectangle: Shape { double w, h; double area() const override { return w*h; } };

// Adding Triangle? Just add a new class — don't touch existing code!
```

---

### Dependency Injection — Testability

```cpp
// BAD: hard dependency — can't test without real database
class OrderService {
    Database db;  // Hardcoded concrete type
public:
    void save(const Order& o) { db.insert(o); }
};

// GOOD: inject through interface — can inject mock in tests
struct IDatabase {
    virtual void insert(const Order& o) = 0;
    virtual ~IDatabase() = default;
};

class OrderService {
    IDatabase& db;  // Depends on interface, not concrete type
public:
    OrderService(IDatabase& db) : db(db) {}
    void save(const Order& o) { db.insert(o); }
};

// Production
OrderService svc(realDatabase);

// Test — inject mock
struct MockDB : IDatabase {
    std::vector<Order> saved;
    void insert(const Order& o) override { saved.push_back(o); }
};
MockDB mock;
OrderService testSvc(mock);
```
