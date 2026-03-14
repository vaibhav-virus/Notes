# Advanced C++

## Operator Overloading Semantics

- Operator overloading lets user-defined types behave like built-in types with standard syntax.
- Overloaded operators are just **specially named functions** — `operator+`, `operator==`, etc.
- Keep semantics intuitive: `+` should add, `==` should compare. Surprising behaviour is a bug.
- Prefer member functions for operators that modify state (`=`, `+=`, `[]`, `()`).
- Prefer free functions for symmetric operators (`+`, `==`) so both sides can convert.

```cpp
class Vec2 {
public:
    float x, y;
    Vec2(float x, float y) : x(x), y(y) {}

    Vec2  operator+(const Vec2& rhs) const { return {x + rhs.x, y + rhs.y}; }
    Vec2& operator+=(const Vec2& rhs)      { x += rhs.x; y += rhs.y; return *this; }
    bool  operator==(const Vec2& rhs) const { return x == rhs.x && y == rhs.y; }
    Vec2  operator-() const                 { return {-x, -y}; }           // Unary negate

    Vec2  operator++()    { ++x; ++y; return *this; }      // Prefix ++
    Vec2  operator++(int) { Vec2 old=*this; ++x; ++y; return old; } // Postfix ++ (dummy int)
};

// Free function — std::ostream on left side, so cannot be a member
std::ostream& operator<<(std::ostream& os, const Vec2& v) {
    return os << "(" << v.x << ", " << v.y << ")";
}
```

---

## Cast Operator Overloading

- Lets an object convert **implicitly or explicitly** to another type.
- Syntax: `operator TargetType() const`
- Add `explicit` to prevent accidental implicit conversions.

```cpp
class Fraction {
    int num, den;
public:
    Fraction(int n, int d) : num(n), den(d) {}

    operator double() const { return static_cast<double>(num) / den; }  // Implicit
    explicit operator bool() const { return den != 0 && num != 0; }     // Explicit only
};

Fraction f(3, 4);
double d = f;          // Calls operator double() → 0.75
if (f) { /*valid*/ }  // Calls explicit operator bool() — works in bool context
```

---

## Move Semantics (Deep Dive)

- **lvalue**: has a name, has an address. **rvalue**: temporary, no persistent address.
- `T&&` is an **rvalue reference** — binds only to temporaries or `std::move()`-d objects.
- `std::move()` does **not** move — it just casts to `T&&`, enabling move constructor/assignment.

```cpp
class Buffer {
public:
    size_t size;
    int*   data;

    Buffer(size_t n) : size(n), data(new int[n]()) {}

    // Copy constructor — deep copy: O(n), allocates new memory
    Buffer(const Buffer& rhs) : size(rhs.size), data(new int[rhs.size]) {
        std::copy(rhs.data, rhs.data + size, data);
    }

    // Move constructor — steal rhs's pointer: O(1), no allocation
    Buffer(Buffer&& rhs) noexcept : size(rhs.size), data(rhs.data) {
        rhs.data = nullptr;  // Leave rhs in a valid but empty state — CRITICAL
        rhs.size = 0;
    }

    Buffer& operator=(Buffer&& rhs) noexcept {
        if (this == &rhs) return *this;
        delete[] data;          // Free own resource first
        data = rhs.data; size = rhs.size;
        rhs.data = nullptr; rhs.size = 0;
        return *this;
    }

    ~Buffer() { delete[] data; }
};

Buffer a(1000);
Buffer b = std::move(a);  // Move constructor O(1); a.data is now nullptr
```

### Rule of Five

If you define any of these, define all five:

| Special Function | Triggered By |
|---|---|
| Destructor | Object goes out of scope |
| Copy constructor | `T b = a;` |
| Copy assignment | `b = a;` |
| Move constructor | `T b = std::move(a);` |
| Move assignment | `b = std::move(a);` |

---

## `new` and `delete` Operators

### How They Work

- `new T` = `operator new(sizeof(T))` [allocate memory] + constructor call [initialise].
- `delete ptr` = destructor call [cleanup] + `operator delete(ptr)` [deallocate].

```cpp
int* p   = new int(42);  // Allocates + writes 42
delete p;                // Destructor (trivial) + frees

int* arr = new int[10];  // Array form
delete[] arr;            // MUST use delete[] (mismatching is UB)
```

### Overloading `new` / `delete` Per-Class

```cpp
class MyClass {
public:
    void* operator new(size_t size) {
        void* ptr = std::malloc(size);
        if (!ptr) throw std::bad_alloc();
        return ptr;
    }
    void operator delete(void* ptr) noexcept { std::free(ptr); }

    void* operator new[](size_t size)          { return std::malloc(size); }
    void  operator delete[](void* ptr) noexcept { std::free(ptr); }
};
```

### Placement `new`

- Constructs at a **pre-allocated** location. No heap allocation. Call destructor manually.

```cpp
alignas(MyClass) char buf[sizeof(MyClass)];
MyClass* obj = new (buf) MyClass();  // Construct into buf — no heap
obj->~MyClass();                     // Manually call destructor — NO delete
```

---

## Allocators in STL

- STL containers are **allocator-aware** — use an allocator policy for all memory operations.
- Custom allocators allow: pool allocation, shared memory, logging, tracking.

```cpp
// All STL containers accept an allocator as template parameter
std::vector<int>                       v1;  // Uses std::allocator<int>
std::vector<int, std::allocator<int>>  v2;  // Same, explicit
```

### Creating a Custom Allocator

```cpp
template <typename T>
struct LoggingAllocator {
    using value_type = T;

    LoggingAllocator() = default;
    template <typename U> LoggingAllocator(const LoggingAllocator<U>&) {}  // Rebind ctor

    T* allocate(std::size_t n) {
        printf("Alloc %zu * %zu bytes\n", n, sizeof(T));
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }
    void deallocate(T* ptr, std::size_t) noexcept { ::operator delete(ptr); }

    bool operator==(const LoggingAllocator&) const noexcept { return true; }
    bool operator!=(const LoggingAllocator&) const noexcept { return false; }
};

std::vector<int, LoggingAllocator<int>> v;
v.push_back(1);  // "Alloc 1 * 4 bytes"
```

### Pool Allocator (Performance Pattern)

- Pre-allocate a large chunk; hand out fixed-size blocks. No fragmentation. O(1) alloc/free.

```cpp
struct MemoryPool {
    struct Block { Block* next; };  // Each free block stores pointer to next free block
    Block*  freeList = nullptr;
    char*   memory   = nullptr;
    size_t  blockSize;

    MemoryPool(size_t blockSz, size_t count) : blockSize(blockSz) {
        memory = new char[blockSz * count];
        for (size_t i = 0; i < count; ++i) {
            Block* b = reinterpret_cast<Block*>(memory + i * blockSz);
            b->next  = freeList; freeList = b;  // Link blocks into free list
        }
    }
    void* allocate() {
        if (!freeList) throw std::bad_alloc();
        Block* b = freeList; freeList = freeList->next; return b;  // Pop
    }
    void deallocate(void* ptr) {
        Block* b = static_cast<Block*>(ptr);
        b->next = freeList; freeList = b;  // Push back
    }
    ~MemoryPool() { delete[] memory; }
};
```

---

## Memory Management

### RAII

- Tie resource lifetime to object lifetime. Acquire in constructor, release in destructor.
- Guarantees cleanup even on exceptions.

```cpp
class FileHandle {
    FILE* file;
public:
    FileHandle(const char* path, const char* mode) : file(std::fopen(path, mode)) {
        if (!file) throw std::runtime_error("Failed to open");
    }
    ~FileHandle() { if (file) std::fclose(file); }  // Auto cleanup
    FileHandle(const FileHandle&) = delete;          // Prevent double-close
    FileHandle& operator=(const FileHandle&) = delete;
    FILE* get() const { return file; }
};
```

### Stack vs Heap

| | Stack | Heap |
|---|---|---|
| Allocation | Automatic | Manual (`new` / `malloc`) |
| Deallocation | Automatic (scope end) | Manual or smart ptr |
| Size | Limited (~1–8 MB) | Large (RAM limit) |
| Speed | O(1) | Slower — allocator overhead |
| Use for | Local variables, small objects | Large data, dynamic lifetime |

### Shallow vs Deep Copy

```cpp
// Shallow (default) — both objects share same pointer → double-free crash
// Deep — allocate new memory and copy contents

class Data {
public:
    int* arr; int size;
    Data(int n) : size(n), arr(new int[n]()) {}

    Data(const Data& rhs) : size(rhs.size), arr(new int[rhs.size]) {
        std::copy(rhs.arr, rhs.arr + size, arr);  // Deep copy — own independent data
    }
    ~Data() { delete[] arr; }
};
```

### Object Pool

```cpp
template <typename T, int N = 100>
class ObjectPool {
    T    pool[N];
    bool inUse[N] = {};
public:
    T* acquire() {
        for (int i = 0; i < N; ++i)
            if (!inUse[i]) { inUse[i] = true; return &pool[i]; }
        return nullptr;
    }
    void release(T* obj) {
        int i = obj - pool;
        if (i >= 0 && i < N) { obj->~T(); new (obj) T(); inUse[i] = false; }
    }
};
```

### Avoiding Fragmentation

- Use **pool allocators** for same-sized objects — no fragmentation.
- Use **arena allocators** for short-lived batch work — free all at once.
- Prefer `std::vector` (contiguous) over `std::list` (scattered nodes in heap).
- Batch allocations — allocate many at once instead of one by one.

### Reducing Allocation Overhead (High-Frequency)

1. **Object Pool** — reuse pre-allocated objects.
2. **Arena/Bump Allocator** — single pointer increment per allocation; free all at once.
3. **Stack Allocation** — prefer local variables for short-lived objects.
4. **`reserve()`** — call `v.reserve(n)` before filling a vector to prevent reallocs.
5. **Move Semantics** — transfer ownership instead of deep-copying expensive resources.
