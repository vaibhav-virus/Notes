# Pointers in C++

## What is a Pointer?

A pointer is a variable that **stores the memory address** of another variable.

```cpp
int x = 42;
int* ptr = &x;   // ptr holds the ADDRESS of x (e.g. 0x7ffd...)
*ptr = 100;      // Dereference: go to the address and change the value
// Now x == 100
```

---

## Pointer Basics

```cpp
int val = 10;
int* ptr = &val;  // & = "address-of" operator

// * has two meanings depending on context:
// 1. In declaration:   int* ptr  → ptr IS a pointer
// 2. In expression:    *ptr      → DEREFERENCE (go to the address and read/write value)

std::cout << ptr;   // prints address: 0x7ffd...
std::cout << *ptr;  // prints value:   10
```

---

## Pointer Arithmetic

```cpp
int arr[3] = {10, 20, 30};
int* p = arr;       // p points to arr[0]

p++;                // p now points to arr[1] (moves 4 bytes for int)
std::cout << *p;    // 20

p += 1;             // p now points to arr[2]
std::cout << *p;    // 30

// Difference between two pointers = number of elements between them
int* start = arr;
int* end   = arr + 3;
ptrdiff_t count = end - start; // 3 (not bytes, element count)
```

---

## Null Pointer

```cpp
int* ptr = nullptr;  // Modern C++ (prefer over NULL or 0)

if (ptr == nullptr)
    std::cout << "Pointer is null, don't dereference!";

// NEVER dereference a null pointer — undefined behavior / crash
// *ptr = 10; // CRASH
```

---

## Pointer to Pointer (Double Pointer)

```cpp
int val  = 5;
int* p   = &val;   // p  holds address of val
int** pp = &p;     // pp holds address of p

std::cout << val;   // 5
std::cout << *p;    // 5   (dereference once → val)
std::cout << **pp;  // 5   (dereference twice → val)

**pp = 99;          // modifies val through two levels of indirection
// val == 99
```

---

## Pointer vs Reference

| Feature | Pointer | Reference |
|---------|---------|-----------|
| Can be null | Yes (`nullptr`) | No (must bind to valid object) |
| Can be reassigned | Yes | No (always refers to same object) |
| Needs dereference | Yes (`*ptr`) | No (used directly) |
| Use case | Optional ownership, arrays | Aliases, function params |

```cpp
void byPointer(int* p)   { *p = 100; }  // Caller passes &x
void byReference(int& r) { r  = 100; }  // Caller passes x directly

int x = 0;
byPointer(&x);    // x = 100
byReference(x);   // x = 100
```

---

## const with Pointers (Quick Recall)

```cpp
int a = 1, b = 2;

const int* p1 = &a;    // Pointer to CONST data: data can't change, pointer can
// *p1 = 5;            // ERROR
p1 = &b;               // OK

int* const p2 = &a;    // CONST pointer: pointer can't change, data can
*p2 = 5;               // OK
// p2 = &b;            // ERROR

const int* const p3 = &a; // Both const: neither can change
// *p3 = 5;            // ERROR
// p3 = &b;            // ERROR
```

**Memory trick:** Read right-to-left. `const int* p` → "p is a pointer to const int"

---

## Function Pointers

A pointer that holds the address of a **function** instead of a variable.

```cpp
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

// Syntax: returnType (*pointerName)(paramTypes)
int (*op)(int, int) = &add;  // op points to add()
std::cout << op(3, 4);       // 7

op = &sub;                   // Reassign to sub
std::cout << op(3, 4);       // -1

// Use case: strategy pattern, callbacks
void applyOp(int x, int y, int(*func)(int,int)) {
    std::cout << func(x, y);
}
applyOp(10, 5, add);  // 15
applyOp(10, 5, sub);  // 5
```

---

## Pointer to Member (Class)

```cpp
class Dog {
public:
    int age = 3;
    void bark() { std::cout << "Woof!\n"; }
};

// Pointer to member DATA
int Dog::* agePtr = &Dog::age;
Dog d;
std::cout << d.*agePtr;       // 3

// Pointer to member FUNCTION
void (Dog::* barkPtr)() = &Dog::bark;
(d.*barkPtr)();               // Woof!
```

---

## void Pointer (Generic Pointer)

```cpp
void* vp;           // Can point to ANY type — but cannot be dereferenced directly

int   x = 10;
float f = 3.14f;

vp = &x;            // OK
vp = &f;            // OK

// Must cast before dereferencing
int* ip = static_cast<int*>(vp);
std::cout << *ip;   // 10

// Common use: malloc (C-style), generic buffers
```

---

## Raw Pointer vs Smart Pointer (When to Use What)

```cpp
// Raw pointer: use only when you DON'T own the resource
void process(Widget* w) { w->doStuff(); } // Just observer, no delete

// unique_ptr: sole ownership — auto deletes when out of scope
auto u = std::make_unique<Widget>();      // No need to delete manually

// shared_ptr: shared ownership — deletes when last owner gone
auto s = std::make_shared<Widget>();
auto s2 = s;  // ref count = 2; Widget lives until both s and s2 gone

// weak_ptr: observe shared_ptr without owning
std::weak_ptr<Widget> w = s;             // ref count still 2
if (auto locked = w.lock()) {            // Check if still alive
    locked->doStuff();
}
```

---

## Common Pointer Bugs to Avoid

```cpp
// 1. Dangling Pointer — pointing to freed memory
int* p = new int(5);
delete p;
// *p = 10;       // UNDEFINED BEHAVIOR: p is dangling

// Fix: set to nullptr after delete
delete p;
p = nullptr;

// 2. Memory Leak — forgetting to delete
int* p2 = new int(10);
// missing delete p2; → LEAK
// Fix: use smart pointers instead

// 3. Double Delete
int* p3 = new int(20);
delete p3;
// delete p3;     // UNDEFINED BEHAVIOR: double delete
// Fix: set to nullptr after first delete

// 4. Stack pointer escape — returning address of local variable
int* badFunc() {
    int local = 5;
    return &local;  // LOCAL is destroyed after return → DANGLING
}
```

---

## new / delete Internals

```cpp
// new = allocate on heap + call constructor
Widget* w = new Widget();      // 1. allocates memory, 2. calls Widget()

// delete = call destructor + free memory
delete w;                      // 1. calls ~Widget(), 2. frees memory

// Array versions
Widget* arr = new Widget[10];  // calls default constructor for each
delete[] arr;                  // MUST use delete[] for arrays; delete alone = UB

// Placement new — construct in pre-allocated buffer
char buffer[sizeof(Widget)];
Widget* pw = new (buffer) Widget();  // Constructs Widget in buffer
pw->~Widget();                       // Must manually call destructor
```
