# C++ Notes

## Summery of Constructor

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
//Obj will become hallow object after move is complete and should not used in further operation.
```

## Summary operator overloading

- ### Relational operator overloading

    `>, < , = =, <=, >=`

- #### Comparison operator `==` overloading

    ```cpp
        bool operator=(const Base& rhs) const
        {
            //Some comparison code...
            return (mData == rhs.mData);
        }
        
        Base obj1;
        Base obj2;
        
        if(obj1 == obj2)//If true objects have  same     data.
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
      obj2 = obj1 //Assign/Copy data from obj1 to obj2.

      Base& operator+=(const Base& rhs)
      {
          mData = mData + rhs.mData;
          return *this;
      }
      obj2 += obj1 //Add and assign data from obj1  to        obj2.
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
    bool b = static_cast<bool>obj;
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
    std::unique_ptr<Base> uniqueOwner2 = std::move(uniqueOwner1);//OK: ownership can be moved from  uniqueOwner1 to uniqueOwner2. Which leaves uniqueOwner1 in hallow state and should not be used further in code.
    ```

- #### `std::shared_ptr<T>`

  - The ownership/responsibility of an heap allocated object can be shared between multiple owner and will only be released after all the owner are destroyed.
  - `std::shared_ptr<T>` uses reference counting to keep track of how many owners are owning the heap allocated object. Once, the count hits 0 the heap allocated object will be deleted.
  - `std::shared_ptr<T>` object can be copied, but will increase the ref count by one.
  - The ownership can be moved to another object. but will not increase the ref count.

    ```cpp
    std::shared_ptr<Base> sharedOwner1 = std::make_shared<Base>(argumentForBaseConstructor); //std::make_shared() create an object of typ Base on heap and then will return std::shared_ptr object as the owner of that object. Red count will be 1.
    std::shared_ptr<Base> sharedOwner2 = sharedOwnerOwner1; //OK: Ownership can be copied/taken by multiple owner. Now ref count - 2
    std::shared_ptr<Base> sharedOwner3 = std::move(sharedOwner1);//OK: ownership can be moved from  sharedOwner1 to sharedOwner3. Which leaves sharedOwner1 in hallow state and should not be used further in code. hence, ref count is still 2.
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

- Inside a main process, multiple task can be performed simultaneously.
- A single large task can be broken down into smaller independent tasks and can be run simultaneously and their result can be combined as the output of larger task.
- The power of multi-core CPU can be leveraged suing multi-threading to a large complete task faster.

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

    th1.detach(); //th1 will be detached and will complete it's task. We will not track.
    th2.join();//Wait for thread 1 to complete task and join.
    ```

- ### `std::thread::joinable()` function

    returns `false`:
  - If thread is empty and has nothing given to execute.
  - Is already been joined or detached.
  - Or moved to another object.

    ```cpp
    std::thread th1();//Empty thread
    bool isJoinable = the1.joinable();//Return false because thread is empty.
    
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

  - When more than one concurrent thread access and modify the same shared data concurrently and depending on the order in which they access the data the final outcome differs. is called **Race Condition**.
  - When two or more thread access the same memory simultaneously and one of them changes the data in memory is called **Data Race**.
  - To solve this issue, we use  **Mutual Exclusion or Mutex** thread synchronization mechanism.`

- ### class `std::mutex` locking mechanism

    ```cpp
    #include <thread>
    #include <mutex>
    int shared_data = 10;
    std::mutex x;

    void funcName()
    {
        //Some code...
        std::unique_lock<std::mutex> lock(x);//Create a lock.
        lock.lock();//lock the section of code and let the locking thread go ahead and let others wait for it to release.
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
