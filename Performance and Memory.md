# Performance & Memory Management in C++

## Memory Layout of a C++ Program

```
High Address
┌───────────────┐
│  Stack        │  ← Local variables, function call frames (LIFO, fast, auto-managed)
├───────────────┤
│  ↓ grows down │
│               │
│  ↑ grows up   │
├───────────────┤
│  Heap         │  ← Dynamic allocations (new/delete), manual management
├───────────────┤
│  BSS Segment  │  ← Uninitialized global/static variables
├───────────────┤
│  Data Segment │  ← Initialized global/static variables
├───────────────┤
│  Text Segment │  ← Program code (read-only)
Low Address
```

---

## Stack vs Heap

| | Stack | Heap |
|---|---|---|
| Allocation speed | Very fast (just move stack pointer) | Slow (find free block, bookkeeping) |
| Size | Small (~1-8 MB) | Large (limited by RAM) |
| Lifetime | Scope-based, auto-freed | Manual (new/delete) or smart ptr |
| Fragmentation | None | Yes, over time |
| Use for | Small, short-lived objects | Large objects, unknown lifetime |

```cpp
void example() {
    int stackVar = 10;           // Stack: fast, auto-freed when function returns
    int* heapVar = new int(10);  // Heap: slow, must delete manually
    delete heapVar;
}
```

---

## Memory Leak

Heap memory allocated but never released.

```cpp
// BAD: Memory Leak
void leaky() {
    int* p = new int[1000]; // 4000 bytes allocated
    // ... forgot delete[]   // LEAK — 4000 bytes lost forever
}

// GOOD: Use smart pointer — auto-freed when out of scope
void safe() {
    auto p = std::make_unique<int[]>(1000); // auto freed
}
```

**Detection tools:** Valgrind, AddressSanitizer (`-fsanitize=address`), Visual Studio Diagnostic Tools

---

## RAII (Resource Acquisition Is Initialization)

> Tie resource lifetime to object lifetime. Constructor acquires, destructor releases.

```cpp
// Raw approach — error prone
void raw() {
    FILE* f = fopen("file.txt", "r");
    // If exception thrown here → fclose never called → resource leak
    fclose(f);
}

// RAII — safe even with exceptions
class FileHandle {
    FILE* f;
public:
    FileHandle(const char* name) : f(fopen(name, "r")) {}
    ~FileHandle() { if(f) fclose(f); }  // Always runs, even on exception
};

void safe() {
    FileHandle f("file.txt"); // Acquired here
}                             // Automatically closed here — RAII
```

`std::unique_ptr`, `std::lock_guard`, `std::fstream` are all RAII classes.

---

## Object Pool / Memory Pool

> Pre-allocate a chunk of memory and reuse it — avoids repeated heap allocation overhead.

```cpp
// Simple fixed-size object pool
template<typename T, size_t N>
class ObjectPool {
    alignas(T) char buffer[N * sizeof(T)]; // Pre-allocated buffer
    std::vector<T*> free_;                  // List of available slots

public:
    ObjectPool() {
        for (size_t i = 0; i < N; i++)
            free_.push_back(reinterpret_cast<T*>(buffer + i * sizeof(T)));
    }

    T* acquire() {
        if (free_.empty()) return nullptr;  // Pool exhausted
        T* obj = free_.back();
        free_.pop_back();
        return new(obj) T();               // Placement new — construct in-place
    }

    void release(T* obj) {
        obj->~T();                          // Manually call destructor
        free_.push_back(obj);               // Return slot to pool
    }
};

// Usage
ObjectPool<MyClass, 100> pool;
MyClass* obj = pool.acquire();   // Fast! No heap alloc
pool.release(obj);               // Fast! No heap free
```

---

## Move Semantics — Avoid Unnecessary Copies

```cpp
std::vector<int> makeData() {
    std::vector<int> v(1000000, 0); // 4MB of data
    return v;  // NRVO / move: data is MOVED out, not copied
}

// Copy: expensive — duplicates all data
std::vector<int> a = makeData();
std::vector<int> b = a;             // Deep copy — 4MB copied

// Move: cheap — just transfer internal pointer
std::vector<int> c = std::move(a);  // 'a' is now empty; no data copied
```

```cpp
class Buffer {
    char* data;
    size_t size;
public:
    // Move constructor: steal the ptr, leave source empty
    Buffer(Buffer&& other) noexcept
        : data(other.data), size(other.size) {
        other.data = nullptr; // Prevent double-delete
        other.size = 0;
    }
};
```

---

## Copy Elision & RVO (Return Value Optimization)

```cpp
// Compiler creates the return value directly in the caller's space
// No copy or move happens at all!
std::string makeStr() {
    return std::string("hello"); // NRVO: no copy, constructed in-place
}

std::string s = makeStr(); // 0 copies, 0 moves in optimized build
```

---

## Cache Performance — Data Locality

```cpp
// BAD: Column-major traversal of row-major array → cache misses
int matrix[1000][1000];
for (int col = 0; col < 1000; col++)       // Jumps 1000 ints each step
    for (int row = 0; row < 1000; row++)
        matrix[row][col]++;

// GOOD: Row-major traversal → sequential memory → cache friendly
for (int row = 0; row < 1000; row++)       // Sequential memory access
    for (int col = 0; col < 1000; col++)
        matrix[row][col]++;
```

---

## Avoid False Sharing in Multi-Threaded Code

```cpp
// BAD: Both threads write to adjacent memory → same cache line → thrashing
struct SharedData {
    int a; // Thread 1 writes this
    int b; // Thread 2 writes this (but same 64-byte cache line as 'a')
};

// GOOD: Pad to separate cache lines (typically 64 bytes)
struct alignas(64) ThreadData {
    int value;
    // 60 bytes of padding to push next field to new cache line
};
```

---

## Custom Allocator (std::allocator replacement)

```cpp
// std containers accept an allocator template parameter
// Use custom allocator for pool allocation inside containers

template<typename T>
struct PoolAllocator {
    using value_type = T;
    T* allocate(std::size_t n) {
        return static_cast<T*>(myPool.alloc(n * sizeof(T)));
    }
    void deallocate(T* p, std::size_t) {
        myPool.free(p);
    }
};

std::vector<int, PoolAllocator<int>> fastVec; // Uses pool allocator
```

---

## Rule of 0 / 3 / 5

```cpp
// Rule of 0: If you don't manage raw resources, define NOTHING
// Use smart pointers and STL — compiler-generated specials work fine.
class Good {
    std::unique_ptr<int> data;  // smart ptr manages memory
    std::string name;           // STL manages string
    // No destructor, no copy/move — compiler handles everything correctly
};

// Rule of 3: If you define any of {destructor, copy ctor, copy=}, define ALL 3
// Rule of 5: Also add {move ctor, move=} for efficiency
class Rule5 {
    int* data;
    int  size;
public:
    Rule5(int n) : data(new int[n]), size(n) {}
    ~Rule5()                               { delete[] data; }
    Rule5(const Rule5& o)                  : data(new int[o.size]), size(o.size)
                                             { std::copy(o.data, o.data+size, data); }
    Rule5& operator=(const Rule5& o)       { Rule5 tmp(o); std::swap(*this, tmp); return *this; }
    Rule5(Rule5&& o) noexcept              : data(o.data), size(o.size)
                                             { o.data = nullptr; o.size = 0; }
    Rule5& operator=(Rule5&& o) noexcept   { std::swap(data, o.data); std::swap(size, o.size); return *this; }
};
```

---

## Common Performance Tips

```cpp
// 1. Reserve vector capacity to avoid reallocations
std::vector<int> v;
v.reserve(1000);           // Allocate once, no reallocs until 1001 elements

// 2. Use emplace_back instead of push_back (constructs in-place)
std::vector<std::string> sv;
sv.push_back(std::string("hello")); // Creates temp then moves
sv.emplace_back("hello");           // Constructs directly in vector — no temp

// 3. Avoid string copies — use string_view (C++17)
void print(std::string_view sv) { std::cout << sv; }  // No copy
print("hello");             // Works with string literals
print(someStdString);       // Works with std::string too

// 4. Prefer ++i over i++ for iterators/complex types
// i++ creates a copy, increments, returns old copy
// ++i increments and returns self — always cheaper or equal
for (auto it = v.begin(); it != v.end(); ++it) {}

// 5. inline functions — eliminate function call overhead for small functions
inline int square(int x) { return x * x; }
```
