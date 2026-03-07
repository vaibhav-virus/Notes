# Modern C++ (C++11 / C++14 / C++17)

## auto — Type Deduction

```cpp
// Let compiler deduce the type — less typing, same type safety
auto x = 42;              // int
auto p = 3.14;            // double
auto s = std::string("hi"); // std::string

// Very useful with complex iterator types
std::vector<std::pair<int,std::string>> v;
auto it = v.begin();      // instead of: std::vector<std::pair<int,std::string>>::iterator it

// auto with references and const
const auto& ref = x;      // const int&  — binds without copy
auto& ref2 = x;           // int&        — modifiable reference
```

---

## Range-based for Loop

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};

for (int x : v)         std::cout << x;  // copy each element
for (const int& x : v)  std::cout << x;  // read-only, no copy (preferred for heavy objects)
for (int& x : v)        x *= 2;          // modify in-place

// Works with any type that has begin()/end()
std::string s = "hello";
for (char c : s) std::cout << c;   // h e l l o
```

---

## Lambda Expressions

> Anonymous function objects. Capture surrounding variables.

```cpp
// Syntax: [capture](params) -> returnType { body }

auto add = [](int a, int b) { return a + b; };  // No capture
std::cout << add(3, 4);  // 7

// Capture by value [=]: copies of outer variables
int x = 10;
auto byVal = [=]() { return x + 1; };  // Captures copy of x
x = 99;
byVal(); // Returns 11 (captured old x=10)

// Capture by reference [&]: references to outer variables
auto byRef = [&]() { return x + 1; };  // Captures ref to x
x = 99;
byRef(); // Returns 100 (sees updated x=99)

// Capture specific variables
int a = 1, b = 2;
auto mixed = [a, &b]() { return a + b; };  // a by value, b by reference

// Mutable lambda — modify captured-by-value copy
auto counter = [count = 0]() mutable { return ++count; };
counter(); // 1
counter(); // 2

// Common use: passing to STL algorithms
std::vector<int> v = {3, 1, 4, 1, 5};
std::sort(v.begin(), v.end(), [](int a, int b){ return a > b; }); // sort descending
```

---

## std::function — Store Any Callable

```cpp
#include <functional>

// Can hold: lambda, function pointer, functor, member function ptr
std::function<int(int, int)> op;

op = [](int a, int b){ return a + b; };  // Lambda
std::cout << op(3, 4);   // 7

int add(int a, int b) { return a + b; }
op = add;                        // Function pointer
std::cout << op(3, 4);   // 7

// Use in callbacks / event systems
class Button {
    std::function<void()> onClick;
public:
    void setCallback(std::function<void()> cb) { onClick = cb; }
    void click() { if (onClick) onClick(); }
};
```

---

## Template Basics & Variadic Templates

```cpp
// Function template — works for any type T
template<typename T>
T maxVal(T a, T b) { return (a > b) ? a : b; }

maxVal(3, 5);       // T = int
maxVal(3.0, 5.0);   // T = double

// Class template
template<typename T>
class Box {
    T value;
public:
    Box(T v) : value(v) {}
    T get() const { return value; }
};

Box<int> intBox(42);
Box<std::string> strBox("hi");

// Variadic template (C++11) — takes any number of type arguments
template<typename... Args>
void printAll(Args... args) {
    (std::cout << ... << args) << "\n"; // C++17 fold expression
}

printAll(1, " ", 2.5, " ", "hello");  // "1 2.5 hello"
```

---

## constexpr — Compile-Time Evaluation

```cpp
// constexpr function: can be evaluated at compile time
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr int f5 = factorial(5); // Computed at compile time → 120
// No runtime overhead!

int arr[factorial(4)];            // OK — array size must be compile-time constant

// constexpr vs const:
// const    = value cannot change after initialization (may be runtime)
// constexpr = value computed at compile time (always constant)
constexpr int SIZE = 10;          // Compile-time constant
const int size = getInput();      // Runtime constant (valid if read from input)
```

---

## nullptr (C++11)

```cpp
// OLD, ambiguous:
void func(int x)   {}
void func(int* x)  {}
func(NULL);  // Which overload? Ambiguous — NULL is just 0 (int)!

// NEW, unambiguous:
func(nullptr);  // Always resolves to pointer overload
```

---

## Initializer Lists (C++11)

```cpp
// Uniform initialization — use {} everywhere
int x{5};
std::vector<int> v{1, 2, 3, 4};
std::map<std::string, int> m{{"Alice", 1}, {"Bob", 2}};

struct Point { int x, y; };
Point p{3, 4};    // Aggregate initialization

// Prevents narrowing conversions (unlike = or ())
int i = 3.7;    // OK — truncates silently
int j{3.7};     // ERROR at compile time — narrowing not allowed with {}
```

---

## Structured Bindings (C++17)

```cpp
std::pair<int, std::string> p{1, "Alice"};
auto [id, name] = p;         // id=1, name="Alice"
std::cout << id << name;

// Use with map iteration — much cleaner
std::map<std::string, int> scores{{"Alice", 90}, {"Bob", 85}};
for (auto& [name, score] : scores) {
    std::cout << name << ": " << score << "\n";
}

// Decompose tuple
auto [a, b, c] = std::make_tuple(1, 2.0, "three");
```

---

## std::optional (C++17)

```cpp
// Represent a value that may or may not exist — no need for sentinel values or null
#include <optional>

std::optional<int> findAge(const std::string& name) {
    if (name == "Alice") return 30;
    return std::nullopt;  // No value
}

auto age = findAge("Bob");
if (age.has_value())           // or: if (age)
    std::cout << *age;         // Dereference to get value
else
    std::cout << "Not found";

// With default
int a = age.value_or(-1);  // -1 if no value
```

---

## std::variant (C++17) — Type-Safe Union

```cpp
#include <variant>

std::variant<int, float, std::string> v;

v = 42;                             // Holds int
std::cout << std::get<int>(v);      // 42

v = "hello";                        // Now holds string
std::cout << std::get<std::string>(v); // "hello"

// Safe visitor pattern
std::visit([](auto& val) {
    std::cout << val;
}, v);

// std::get throws if wrong type — use std::get_if for safe access
if (auto* s = std::get_if<std::string>(&v))
    std::cout << *s;
```

---

## std::string_view (C++17) — Non-Owning String Reference

```cpp
// Like const std::string& but works with string literals without allocation
void print(std::string_view sv) {  // Accepts string&, const char*, string literal
    std::cout << sv.substr(0, 3);
}

print("hello world");    // No allocation — just a pointer + length
print(someStdString);    // No copy of std::string

// WARNING: string_view does NOT own the string.
// Don't store it beyond the lifetime of the underlying string!
std::string_view bad() {
    std::string s = "temp";
    return s;  // DANGER: s destroyed, view dangling
}
```

---

## Move Semantics & rvalue References

```cpp
// lvalue: has a name, addressable    e.g. int x = 5;
// rvalue: temporary, no name         e.g. 5, x+3, makeObj()

void process(int& x)  { /* takes lvalue */ }
void process(int&& x) { /* takes rvalue — can steal resources */ }

int a = 10;
process(a);           // calls lvalue version
process(10);          // calls rvalue version
process(std::move(a)); // Force treat 'a' as rvalue → calls rvalue version
                       // DON'T use 'a' after this — its state is unspecified

// Perfect forwarding — preserve lvalue/rvalue category through templates
template<typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg)); // forwards as lvalue if was lvalue, rvalue if was rvalue
}
```

---

## noexcept Specifier

```cpp
void safeFunc() noexcept {   // Guarantees this function won't throw
    // If exception does occur → std::terminate() called
}

// Move constructors MUST be noexcept for STL optimizations
// std::vector will use move instead of copy ONLY if move is noexcept
class MyClass {
public:
    MyClass(MyClass&&) noexcept {}   // noexcept → vector can use move on realloc
};

// Check at compile time
static_assert(noexcept(safeFunc()), "Must be noexcept");
```

---

## Type Aliases (using) vs typedef

```cpp
// Old style — harder to read
typedef std::vector<std::pair<int, std::string>> OldAlias;

// New style (C++11) — cleaner, works with templates
using StringIntPair = std::pair<int, std::string>;
using DataList      = std::vector<StringIntPair>;

// Template alias — impossible with typedef
template<typename T>
using Matrix = std::vector<std::vector<T>>;

Matrix<int>    intMatrix;
Matrix<double> dblMatrix;
```
