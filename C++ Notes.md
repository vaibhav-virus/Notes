# C++ Notes

##  Summary of vptr and vtable in C++


### vptr (Virtual Pointer):
-	Added by compiler and Exists as a hidden member in each object of the class.
-	Points to the vtable corresponding to the actual class type of the object.
-	Set by the constructor of the class during object creation.
-	Ensures that the correct virtual function is called based on the object's type at runtime.
###	vtable (Virtual Table):
-	A table of function pointers, created by the compiler for each class with virtual functions.
-	Contains pointers to the virtual functions of the class, with one entry per virtual function.
-	For derived classes, it may override entries in the vtable to point to overridden functions in the derived class.
-	Shared among all objects of the same class.


## vptr setup and polymorphism

### During object instantiation of a derived class
1.	Base constructor called.
2.	Very first, Vptr set to Base class Vtable.
3.	Polymorphic function calls will resolve to Base class function calls
4.	Derived class constructor called.
5.	Vptr updated to Derived class Vtable.
6.	Polymorphic function calls will resolve to Derived class function calls.

### Conclusion:
The key reason polymorphism doesn't work as expected is that during base class construction, the object isn't yet of the derived type. Therefore, any virtual function calls will use the vtable of the base class.

### During object destruction of a derived class
1.	Derived destructor called.
2.	Vptr already pointing to Derived class Vtable.
3.	Hence Polymorphic function calls will resolve to Derived class function calls.
4.	Base destructor called.
5.	Vptr updated to Base class Vtable.
6.	Polymorphic function calls will resolve to Base class function calls.
### Conclusion:
- Object "Demotion": When the derived class destructor finishes, the object is effectively "demoted" to its base class type.
- vptr Resetting: Since the derived class's destructor has already run, the vptr now points to the base class's vtable. Any virtual function calls in the base class destructor will dispatch to the base class's versions, not the derived class's versions.

To avoid overriding a method in derived class, mark method final in base class.
```cpp
virtual void Base::func1() final override {}
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
- Const data members should always be initialize using initializer list. 
- const/Non const method Cannot modify const class data member.
```cpp
class MyClass {
public:
    const int x;
    MyClass(int val) : x(val) {} // 1. Const data members should always be initialize using initializer list 
    int getX() const { x = 100; } // 2. Error: const/Non const method: Cannot modify const class data member
};
```

### static const class member variable/data
- Cannot be modified  once created.
- Must be initialized  inside the class body.
```cpp
class MyClass {
public:
    static const int x = 10;  //const static variable must be initialized inside the class
};
```

## Const class methods
- Const function cannot modify any class member variable/data.
- And can only call const function inside it
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

### Solution:  
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
