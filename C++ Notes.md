# C++ Notes

## Summary of Constructor

### Default Constructor

```cpp
Base::Base(){}
```

### Parametrized Constructor

```cpp
Base::Base(const int& i){}
```

### Copy Constructor

```cpp
Base::Base(const Base& rhs){}

Base Obj1;
Base newObj = Obj1; //Creating a new object (newObj) and copying all data from Obj1 to newObj.
```

### Move Constructor

- Used with `std::move(objectToBeMoved);`

```cpp
Base::Base(Base&& rhs) noexcept {}

Base Obj1;
Base newObj = std::move(Obj1); //Moving all data from Obj1 to newObj.
//Obj1 will become a hollow object after the move is complete and should not be used in further operations.
```

## Summary operator overloading

- ### Relational operator overloading

    `>, < , = =, <=, >=`

- #### Comparison operator `==` overloading

    ```cpp
        bool operator==(const Base& rhs) const
        {
            //Some comparison code...
            return (mData == rhs.mData);
        }
        
        Base obj1;
        Base obj2;
        
        if(obj1 == obj2)//If true objects have same data.
        {}
        ```

- ### Assignment operators overloading

    `=, +=,*=, /=,-=, %=`

- #### Assignment operator `=` overloading

    ```cpp
      Base obj1;
      Base obj2;
      
      Base& operator=(const Base& rhs)
      {
          //Some copy code...
          return *this;
      }
      obj2 = obj1; //Assign/Copy data from obj1 to obj2.

      Base& operator+=(const Base& rhs)
      {
          mData = mData + rhs.mData;
          return *this;
      }
      obj2 += obj1; //Add and assign data from obj1 to obj2.
      ```

- ### Subscript operators `[]` overloading

    ```cpp
    int operator[] (bool) const { return 10;}

    Base obj;
    int i = obj[false];
    ```

- ### Function call  operators `()` overloading

    ```cpp
    bool operator() (int a) const { return (a == 10);}

    Base obj;
    bool i = obj(100);
    ```

- ### Arrow operators `->` overloading

    ```cpp
    Base* operator-> () const { return this;}
    ```

- ### `new` and `delete` operators overloading

    ```cpp
    void* operator new (size_t size){ return ::operator new(size); }
    void* operator new[] (size_t size){ return ::operator new[](size); }

    void operator delete(void* ptr) { ::operator delete(ptr); }
    void operator delete[](void* ptr) { ::operator delete[](ptr); }
    ```

- ### Typecast operators overloading

    ```cpp
    operator bool() { return (mData != nullptr); }

    Base obj;
    bool b = (bool)obj;
    bool b = static_cast<bool>(obj);
    ```

- ### Pre/Post Increment operators overloading

    ```cpp
    Base operator++() 
    { 
        mData = mData + 1;
        return Base(mData); 
    } //Pre increment
    Base operator++(int) 
    {
        Base b(mData);
        mData = mData + 1;
        return b; 
    } //Post increment

    Base obj;
    ++obj;//Pre increment
    obj++;//Post increment
    ```

## Summary of vptr and vtable in C++

### vptr (Virtual Pointer)

- Added by compiler and Exists as a hidden member in each object of the class.
- Points to the vtable corresponding to the actual class type of the object.
- Set by the constructor of the class during object creation.
- Ensures that the correct virtual function is called based on the object's type at runtime.

### vtable (Virtual Table)

- A table of function pointers, created by the compiler for each class with virtual functions.
- Contains pointers to the virtual functions of the class, with one entry per virtual function.
- For derived classes, it may override entries in the vtable to point to overridden functions in the derived class.
- Shared among all objects of the same class.

## vptr setup and polymorphism

### During object instantiation of a derived class

1. Base constructor called.
2. Very first, Vptr set to Base class Vtable.
3. Polymorphic function calls will resolve to Base class function calls
4. Derived class constructor called.
5. Vptr updated to Derived class Vtable.
6. Polymorphic function calls will resolve to Derived class function calls.

### Conclusion

The key reason polymorphism doesn't work as expected is that during base class construction, the object isn't yet of the derived type. Therefore, any virtual function calls will use the vtable of the base class.

### During object destruction of a derived class

1. Derived destructor called.
2. Vptr already pointing to Derived class Vtable.
3. Hence Polymorphic function calls will resolve to Derived class function calls.
4. Base destructor called.
5. Vptr updated to Base class Vtable.
6. Polymorphic function calls will resolve to Base class function calls.

### Conclusion

- Object "Demotion": When the derived class destructor finishes, the object is effectively "demoted" to its base class type.
- vptr Resetting: Since the derived class's destructor has already run, the vptr now points to the base class's vtable. Any virtual function calls in the base class destructor will dispatch to the base class's versions, not the derived class's versions.

To avoid overriding a method in derived class, mark the method `final` in the base class.

```cpp
class Base {
    virtual void func1() final {}  // final prevents any derived class from overriding func1
};
```

Mark base class final to avoid any other class from inheriting it.

```cpp
class Base final {};
class Derived: public Base {}; //Compiler failure.
```

Pointer memory address of an object always stays the same irrespective of the type/cast type.

```cpp
B* ptr = new B();
A* ptr2 = ptr;
A* ptr2 = dynamic_cast<A*>(ptr);
ptr and ptr2 are pointer with same memory location.
```

## Const Variables

Data is const. Data cannot be changed.

```cpp
const int data = 10;
data = 20; // Error: assignment of read-only variable 'x'
```

Cannot change Data but pointer can be changed.

```cpp
const int* dataPtr = &x;  //Data is const, pointer is not
*dataPtr = 20; // Error: cannot modify value through ptr
```

Data can be changed but pointer cannot be changed.

```cpp
int data = 10;
int* const dataPtr = &data; //pointer itself is const
dataPtr = &x; // Error: cannot change the address stored in ptr
```

both data and pointer are const. hence, cannot be changed

```cpp
int data = 10;
const int* const dataPtr = &data; //Data and ptr are const
*dataPtr = 20; // Error: cannot modify value through ptr
dataPtr = &x;  // Error: cannot change the address stored in ptr
```

### Const class data variable/member

- Const data members should always be initialized using an initializer list.
- Neither `const` nor non-`const` methods can modify a `const` class data member.

```cpp
class MyClass {
public:
    const int x;
    MyClass(int val) : x(val) {} // 1. Const data members should always be initialize using initializer list 
    void getX() const { x = 100; } // 2. Error: Cannot modify const class data member from any method
};
```

### static const class member variable/data

- Cannot be modified once created.
- For integral types, must be initialized inside the class body. For non-integral types, must be defined outside the class body.

```cpp
class MyClass {
public:
    static const int x = 10;  //const static variable must be initialized inside the class
};
```

## Const class methods

- A `const` method cannot modify any class member variable/data.
- A `const` method can only call other `const` methods.

```cpp
class MyClass {
public:
    int x;
    int getX() const { x = 100; } // Error: const method: Cannot modify class data 
};
```

### mutable class member variable/data

- only member variable defined as mutable can be modified in const method.

```cpp
class MyClass {
public:
    mutable int x; 
    void setX(int val) const { x = val; }  // OK: const method: Can modify x because x is mutable
};
```

### const and non const overridden virtual function

- If the object is const then const method will always be preferred. Even the base methods is overridden in derived class.

```cpp
class Base {
public:
    virtual void display() const {}
    virtual void display()  {}
};

class Derived : public Base {
public:
    void display() override {}};

int main() {
    const Base* b = new Derived();
    b->display();  // Calls Base::display because Derived::display is not const
}
```

## Diamond problem in C++ inheritance

### Problem

   In diamond problem, most derived `class D` is getting 2 copies of `class A` from `class B` and `class C`. Which is not logical.

```cpp
class A {
public:
    int data;
    virtual void func(){}
};

class B : public A {};

class C : public A {};

class D : public B, public C {};
```

### Solution  

   Mark common class as virtual while inheriting it in intermediate classes would resolve the issue. but the responsibility of instantiation of class A falls on the most derived class in our case `class D`.
   `class B : public virtual A {};`

```cpp
class A {
public:
    int data;
};

class B : public virtual A {};

class C : public virtual A {};

class D : public C, public B {};
```

### Diamond class instantiation  and  constructor/destructor call sequence

- Constructors call sequence depends on inheritance list of `class D`.
     e.g. `class D : public C, public B`
         A( ) -> C( ) -> B( ) -> D( )
- Destructor call sequence is reverse of Constructor call.
    ~D( ) -> ~B( ) -> ~C( ) -> ~A( )

## STL(Standard Template Library)

### Smart Pointer

- Smart pointer are used the automate/manage the life cycle and allocated memory for heap allocated objects in C++.
- Smart pointer are just templatised class wrappers over the class whose life cycle is to be managed.

- #### `std::unique_ptr<T>`

  - Once a object is created on heap. The ownership/responsibility of that object can only be taken by a single owner at a time.
  - Ownership of heap allocated object can be moved to another `std::unique_ptr<T>` object but cannot be copied.

    ```cpp
    std::unique_ptr<Base> uniqueOwner1 = std::make_unique<Base>(argumentForBaseConstructor); //std::make_unique() will create an object of typ Base on heap and then will return std::unique_ptr object as the sole owner of that object.
    std::unique_ptr<Base> uniqueOwner2 = uniqueOwner1; //ERROR: Ownership cannot be copied or taken by multiple owner.
    std::unique_ptr<Base> uniqueOwner2 = std::move(uniqueOwner1);//OK: ownership can be moved from uniqueOwner1 to uniqueOwner2. Which leaves uniqueOwner1 in a hollow state and should not be used further in code.
    ```

- #### `std::shared_ptr<T>`

  - The ownership/responsibility of an heap allocated object can be shared between multiple owner and will only be released after all the owner are destroyed.
  - `std::shared_ptr<T>` uses reference counting to keep track of how many owners are owning the heap allocated object. Once, the count hits 0 the heap allocated object will be deleted.
  - `std::shared_ptr<T>` object can be copied, but will increase the ref count by one.
  - The ownership can be moved to another object. but will not increase the ref count.

    ```cpp
    std::shared_ptr<Base> sharedOwner1 = std::make_shared<Base>(argumentForBaseConstructor); //std::make_shared() creates an object of type Base on heap and returns a std::shared_ptr as the owner of that object. Ref count will be 1.
    std::shared_ptr<Base> sharedOwner2 = sharedOwner1; //OK: Ownership can be copied/taken by multiple owners. Now ref count = 2
    std::shared_ptr<Base> sharedOwner3 = std::move(sharedOwner1);//OK: ownership can be moved from sharedOwner1 to sharedOwner3. Which leaves sharedOwner1 in a hollow state and should not be used further in code. Hence, ref count is still 2.
    ```

- #### `std::weak_ptr<T>`

  - `std::weak_ptr<T>` does not take any kind of ownership of any object. it only observes/reference `std::shared_ptr<T>` without incrementing the ref count.
  - `std::weak_ptr<T>` can be copied or moved without ref count increment or decrement.

    ```cpp
    std::shared_ptr<Base> sharedOwner1 = std::make_shared<Base>(argumentForBaseConstructor); //ref count - 1
    std::weak_ptr<Base> weak1 = sharedOwner1; //ref count - 1
    std::weak_ptr<Base> weak2 = weak1; //ref count - 1
    std::weak_ptr<Base> weak3 = std::move(weak2);//ref count - 1
    ```

  - How to check if the shared object being observed is still alive?

    ```cpp
    std::shared_ptr<Base> newShared = weak1.lock();
    if(newShared){ /*Heap allocated object Still alive.*/ } 
    else{ /*Heap allocated object is deleted.*/ } 
    ```

## Multi-Threading

- Inside a main process, multiple tasks can be performed simultaneously.
- A single large task can be broken down into smaller independent tasks, run simultaneously, and their results combined as the output of the larger task.
- The power of multi-core CPUs can be leveraged using multi-threading to complete a large task faster.

- ### Thread creation

    ```cpp
    #include<thread>

    void funcName(int parameter1)
    {
        //Some processing.
    }

    int parameter1 = 100;
    std::thread th1(funcName, parameter1); //Thread gets created and starts executing the passed function with given parameter value independently.
    int parameter2 = 100;
    std::thread th2(funcName, parameter2); //2nd Thread gets created and starts executing the same passed function with some different given parameter value independently.

    th1.detach(); //th1 will be detached and will complete its task independently. We will not track it.
    th2.join();//Wait for thread 2 to complete its task and join.
    ```

- ### `std::thread::joinable()` function

    returns `false`:
  - If thread is empty and has nothing given to execute.
  - Is already been joined or detached.
  - Or moved to another object.

    ```cpp
    std::thread th1();//Empty thread
    bool isJoinable = th1.joinable();//Returns false because thread is empty.
    
    std::thread th2(funcName, parameter1); //Non-Empty thread. Has been given something to execute.
    if(th2.joinable())//Returns true. Because th2 is not empty and can join back.
        th2.join();//wait for it to join.
    
    bool isJoinable = th2.joinable();//Returns false. Because th2 has already joined just above.
    std::thread th3(funcName, parameter1);
    std::thread th4 = std::move(th3);//Moved thread from th3 to th4
    bool isJoinable = th3.joinable();//Returns false. Because thread moved to another object.
    th4.detach();
    bool isJoinable = th4.joinable();//Returns false. Because, th4 has been detached and can no longer join back.
    ```

- ### Race Condition and Data Race

  - When more than one concurrent thread accesses and modifies the same shared data, and the final outcome differs depending on the order of access, it is called a **Race Condition**.
  - When two or more threads access the same memory simultaneously and at least one of them modifies the data, it is called a **Data Race**.
  - To solve this issue, we use **Mutual Exclusion (Mutex)** as a thread synchronization mechanism.

- ### class `std::mutex` locking mechanism

    ```cpp
    #include <thread>
    #include <mutex>
    int shared_data = 10;
    std::mutex x;

    void funcName()
    {
        //Some code...
        std::unique_lock<std::mutex> lock(x, std::defer_lock);//Create a lock without locking yet (defer_lock prevents auto-locking on construction).
        lock.lock();//Explicitly lock the section of code; locking thread proceeds, others wait for the lock to be released.
        if((shared_data % 2) == 0)//If even add 3
            shared_data = shared_data + 3;
        else //If odd add 2
            shared_data = shared_data + 2;
        lock.unlock();//Unlock the section of code and let any other single thread to lock the same section and use it.
        //Some more code...
    }
    ```

- ### `class std::lock_guard<std::mutex>` and `class std::unique_lock<std::mutex>` uses

    ```cpp
    std::mutex x;
    void funName()
    {
     {
        std::lock_guard<std::mutex> lock1(x);//Simultaneously create the lock and lock the below section of code upto next scope end.
        //Some code...
        //Some more code...
     }//As lock1 scope ends automatically unlock above section of code.
     std::unique_lock<std::mutex> lock2(x);//Just create the lock.
     lock2.lock(); //Explicitly lock the below section of code.
     //Some code...
     lock2.unlock(); //Explicitly unlock the above section of code.
    }
    ```

## Move Semantics

- Move semantics allow transferring resources (heap memory, file handles, etc.) from a temporary/expiring object to another instead of copying them, making it faster.
- An **lvalue** is a named, addressable object. An **rvalue** is a temporary, unnamed object (e.g., result of `std::move()` or a function return).
- `std::move()` casts an lvalue into an rvalue reference (`&&`), enabling the move constructor/assignment instead of copy.
- After a move, the source object is left in a valid but **unspecified (hollow) state** and must not be used.

```cpp
class MyClass {
public:
    int* data;

    MyClass(int val) : data(new int(val)) {}  // Constructor

    // Move constructor: steal resources from rhs instead of copying
    MyClass(MyClass&& rhs) noexcept : data(rhs.data) {
        rhs.data = nullptr;  // Leave rhs in a hollow state
    }

    // Move assignment operator
    MyClass& operator=(MyClass&& rhs) noexcept {
        if (this != &rhs) {
            delete data;        // Free existing resource
            data = rhs.data;    // Steal from rhs
            rhs.data = nullptr; // Hollow out rhs
        }
        return *this;
    }

    ~MyClass() { delete data; }
};

MyClass obj1(42);
MyClass obj2 = std::move(obj1); // Calls move constructor; obj1.data is now nullptr
MyClass obj3(100);
obj3 = std::move(obj2);         // Calls move assignment; obj2.data is now nullptr
```

---

## Virtual Constructor and Virtual Destructor

### Virtual Constructor

- C++ does **not** support virtual constructors. A constructor cannot be virtual because the vtable isn't set up until the constructor runs.
- The closest alternative is the **Virtual Clone idiom** (Factory Method pattern).

```cpp
class Base {
public:
    virtual Base* clone() const { return new Base(*this); } // Virtual clone idiom
    virtual ~Base() {}
};

class Derived : public Base {
public:
    Derived* clone() const override { return new Derived(*this); } // Covariant return type
};

Base* b = new Derived();
Base* copy = b->clone(); // Creates a Derived object even though we only have a Base*
```

### Virtual Destructor

- Mark base class destructor `virtual` whenever a class is meant to be inherited.
- Without `virtual`, deleting a derived object through a base pointer only calls the base destructor, causing a **resource leak**.

```cpp
class Base {
public:
    virtual ~Base() { /* Cleanup base resources */ }  // virtual ensures derived destructor is also called
};

class Derived : public Base {
public:
    int* data = new int(10);
    ~Derived() { delete data; }  // This would be skipped without virtual ~Base()
};

Base* obj = new Derived();
delete obj; // With virtual ~Base(): calls ~Derived() then ~Base(). Without: only ~Base() -> memory leak.
```

---

## Runtime and Compile-Time Polymorphism

### Compile-Time Polymorphism (Static Dispatch)

- Resolved at **compile time**. The compiler decides which function to call.
- Achieved via **function overloading** and **templates**.
- Faster: no vtable lookup overhead.

```cpp
// Function overloading
void draw(int shape)   { /* draw by id */ }
void draw(float angle) { /* draw by angle */ }

draw(5);     // Compiler picks draw(int) at compile time
draw(3.14f); // Compiler picks draw(float) at compile time

// Templates
template<typename T>
T add(T a, T b) { return a + b; }

add(1, 2);      // Compiler instantiates add<int> at compile time
add(1.5, 2.5);  // Compiler instantiates add<double> at compile time
```

### Runtime Polymorphism (Dynamic Dispatch)

- Resolved at **runtime** via vptr -> vtable lookup.
- Achieved via **virtual functions** and inheritance.
- Slightly slower due to indirection through vtable.

```cpp
class Shape {
public:
    virtual void draw() const {}
    virtual ~Shape() {}
};

class Circle : public Shape {
public:
    void draw() const override { /* circle draw */ }
};

class Square : public Shape {
public:
    void draw() const override { /* square draw */ }
};

Shape* s = new Circle();
s->draw(); // Resolved at runtime: calls Circle::draw() via vtable
```

---

## 4 Pillars of OOP

### 1. Encapsulation

- Bundling data and the functions that operate on it inside a class, and restricting direct access to internals.

```cpp
class BankAccount {
    double balance = 0.0;  // Private: cannot be accessed directly from outside
public:
    void deposit(double amount) { if (amount > 0) balance += amount; }
    double getBalance() const { return balance; }  // Controlled access
};
```

### 2. Abstraction

- Exposing only **what** an object does, hiding **how** it does it.

```cpp
class Engine {
public:
    void start() { fuelInject(); ignite(); }  // User only calls start(), not internals
private:
    void fuelInject() {}
    void ignite() {}
};
```

### 3. Inheritance

- A derived class acquires properties and behaviour of a base class, enabling code reuse.

```cpp
class Animal {
public:
    void breathe() { /* common behaviour */ }
};

class Dog : public Animal {  // Dog inherits breathe() from Animal
public:
    void bark() {}
};
```

### 4. Polymorphism

- Same interface, different behaviour depending on the actual object type at runtime or compile time.

```cpp
Shape* shapes[] = { new Circle(), new Square() };
for (auto* s : shapes)
    s->draw(); // Each shape draws itself differently through the same interface
```

---

## Association and Composition

### Association

- A **"uses-a"** or **"knows-a"** relationship. Both classes exist independently; neither owns the other.
- The associated object is typically passed in (not created or owned by the class).

```cpp
class Engine {};

class Car {
    Engine* engine; // Car "uses" an Engine but does NOT own it; Engine can exist without Car
public:
    Car(Engine* e) : engine(e) {}  // Engine is injected (association)
};

Engine eng;
Car car(&eng);  // eng and car have independent lifetimes
```

### Composition

- A **"has-a"** (strong ownership) relationship. The composed object's lifetime is controlled by the owning class.
- If the owner is destroyed, the composed objects are also destroyed.

```cpp
class Wheel {};

class Car {
    Wheel wheels[4]; // Car "owns" Wheels; Wheels are created and destroyed with Car
};
// When Car is destroyed, all Wheel objects inside it are also destroyed
```

---

## Public, Private, and Protected Inheritance

| Access in Base | `public` Inheritance | `protected` Inheritance | `private` Inheritance |
|---|---|---|---|
| `public`    | `public` in Derived    | `protected` in Derived | `private` in Derived |
| `protected` | `protected` in Derived | `protected` in Derived | `private` in Derived |
| `private`   | Not accessible          | Not accessible          | Not accessible        |

### Public Inheritance (`is-a` relationship)

- Base's public/protected members remain public/protected in Derived. Most common inheritance.

```cpp
class Animal {
public:
    void breathe() {}
protected:
    int heartRate = 60;
};

class Dog : public Animal {
    // breathe() is still public here
    // heartRate is still protected here
    void check() { heartRate = 80; } // OK: can access protected member
};

Dog d;
d.breathe(); // OK: public member accessible on object
```

### Protected Inheritance

- Base's public members become protected in Derived. Cannot call them on a Derived object from outside.

```cpp
class Dog : protected Animal {
    // breathe() becomes protected here
    void check() { breathe(); } // OK: accessible inside Derived
};

Dog d;
d.breathe(); // ERROR: breathe() is now protected, not accessible outside class
```

### Private Inheritance (`implemented-in-terms-of` relationship)

- All base members become private in Derived. No further inheritance chain gets access.

```cpp
class Dog : private Animal {
    // breathe() becomes private here
    void check() { breathe(); } // OK: accessible inside Derived
};

Dog d;
d.breathe(); // ERROR: private, not accessible outside class
```

---

## Types of Casts

### `static_cast`

- Compile-time cast. No runtime checks. Used for well-defined, safe conversions (numeric types, known up/downcast in hierarchy).

```cpp
double d = 3.14;
int i = static_cast<int>(d); // Truncates to 3; compiler checks type compatibility

Base* b = new Derived();
Derived* der = static_cast<Derived*>(b); // Downcast; unsafe if b is not actually a Derived
```

### `dynamic_cast`

- Runtime cast. Checks actual object type using RTTI. Returns `nullptr` (pointer) or throws `std::bad_cast` (reference) if cast is invalid.
- Requires base class to have at least one `virtual` function.

```cpp
Base* b = new Derived();
Derived* d = dynamic_cast<Derived*>(b); // OK: b actually points to a Derived; d is valid
if (d) { /* safe to use */ }

Base* b2 = new Base();
Derived* d2 = dynamic_cast<Derived*>(b2); // Returns nullptr; b2 is not a Derived
```

### `const_cast`

- Removes or adds `const`/`volatile` from a pointer or reference. The only cast that can do this.

```cpp
const int x = 10;
int* p = const_cast<int*>(&x); // Removes const from pointer
// Modifying *p when x is a truly const object is undefined behavior — use with caution
```

### `reinterpret_cast`

- Reinterprets the raw bit-pattern of a pointer as a completely different type. No conversion. Purely compile-time.
- **Highly unsafe**: programmer bears full responsibility for correctness.

```cpp
int value = 42;
char* bytes = reinterpret_cast<char*>(&value); // Treat the int's bytes as a char array
// bytes[0] is the first byte of value; useful for binary serialization / low-level inspection

uintptr_t addr = reinterpret_cast<uintptr_t>(&value); // Convert pointer to integer (e.g., log address)
int* ptr2 = reinterpret_cast<int*>(addr);              // Convert back to pointer
```

---

## When `reinterpret_cast` is Used

- **Low-level memory / byte inspection**: reading raw bytes of an object (serialization, hashing).
- **Hardware / memory-mapped I/O**: casting a fixed integer address to a typed pointer for register access.
- **Implementing allocators or memory pools**: casting `void*` from a pool to a typed pointer.
- **Network packet parsing**: treating a raw byte buffer as a structured type.
- **Storing a pointer in an integer** (or vice versa) for platform-specific manipulation.

```cpp
// Memory-mapped hardware register at a fixed address
uint32_t* reg = reinterpret_cast<uint32_t*>(0x40021000); // Cast raw address to typed pointer
*reg = 0x01; // Write to hardware register

// Byte-level serialization: inspect individual bytes of a float
float f = 3.14f;
uint8_t* raw = reinterpret_cast<uint8_t*>(&f); // Access individual bytes of the float
for (int i = 0; i < sizeof(float); ++i)
    printf("byte[%d] = 0x%X\n", i, raw[i]);
```

---

## Difference Between `reinterpret_cast` and `dynamic_cast`

| Feature | `reinterpret_cast` | `dynamic_cast` |
|---|---|---|
| **When resolved** | Compile-time | Runtime |
| **Safety** | Unsafe — no checks | Safe — checks actual type via RTTI |
| **Hierarchy required** | No | Yes (base must have a virtual function) |
| **Converts** | Any pointer/integer to any other type | Pointers/references within a class hierarchy |
| **On failure** | Undefined behavior (no failure indicator) | Returns `nullptr` (pointer) or throws `std::bad_cast` (reference) |
| **Use case** | Low-level bit manipulation, hardware, serialization | Safe downcasting in polymorphic hierarchies |

```cpp
class Base { virtual void f() {} };
class Derived : public Base {};
class Unrelated {};

Base* b = new Derived();

// dynamic_cast: safe, checks at runtime
Derived*   d  = dynamic_cast<Derived*>(b);    // OK: b is actually a Derived
Unrelated* u  = dynamic_cast<Unrelated*>(b);  // Returns nullptr: unrelated type

// reinterpret_cast: no checks, just reinterprets bits — dangerous
Unrelated* u2 = reinterpret_cast<Unrelated*>(b); // Compiles, but using u2 is undefined behavior
```
