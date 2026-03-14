# Design Patterns in C++ — Concept-First Reference

> **Three families:** Creational (how objects are born) · Structural (how objects connect) · Behavioral (how objects talk)
>
> ⭐ = Must-know for interviews · patterns are listed roughly by interview frequency

---

## The Big Picture

| Family | Core Idea | Patterns |
|--------|-----------|----------|
| **Creational** | Control object creation | Singleton, Factory, Abstract Factory, Builder, Prototype |
| **Structural** | Compose objects/classes | Adapter, Decorator, Facade, Composite, Bridge, Proxy |
| **Behavioral** | Object communication & responsibility | Observer, Strategy, Command, Template Method, CRTP |

---

## Creational Patterns

### ⭐ 1. Singleton — *"Only one ever exists"*

**Concept:** A class that controls its own instantiation and guarantees exactly one instance for the lifetime of the program.

**Why "Singleton"?** The Latin *singulus* (one at a time) — only a single instance is ever created.

**When to use:** Shared resource with global state — Logger, Config, ThreadPool, EventBus.

**Pattern to spot:** Private constructor + static getter + deleted copy/assign.

```cpp
class Logger {
    Logger() {}
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
public:
    static Logger& getInstance() {
        static Logger instance;   // C++11 — thread-safe, no explicit mutex needed
        return instance;
    }
    void log(const std::string& msg) { std::cout << msg << "\n"; }
};

Logger::getInstance().log("Hello");  // Same object every call
```

> 💡 **Tip:** Prefer `static` local variable (Meyer's Singleton) over `new` — it's thread-safe in C++11 and auto-destructs.
>
> ⚠️ **Watch out:** Singletons are hard to unit test (hidden global state). Prefer dependency injection where possible.

---

### ⭐ 2. Factory Method — *"Let a function/subclass decide what to create"*

**Concept:** Hide the `new` — the caller asks for a product by *type/name*, not by constructing it directly. The factory decides the concrete class.

**Why "Factory"?** Like a real factory: you order a "car", the factory floor decides which model rolls out — you never see the assembly line.

**When to use:** When object creation logic is complex, or the type is decided at runtime (e.g., file parsers, shape creators).

**Pattern to spot:** A function/method returning a base-class pointer based on a parameter.

```cpp
struct Shape { virtual void draw() = 0; virtual ~Shape() = default; };
struct Circle : Shape { void draw() override { std::cout << "Circle\n"; } };
struct Square : Shape { void draw() override { std::cout << "Square\n"; } };

// Factory function — caller is decoupled from concrete types
std::unique_ptr<Shape> createShape(const std::string& type) {
    if (type == "circle") return std::make_unique<Circle>();
    if (type == "square") return std::make_unique<Square>();
    return nullptr;
}

auto s = createShape("circle");
s->draw();  // Circle — caller never wrote `new Circle`
```

> 💡 **Tip:** Extend via a `registry` (map of string → creator lambda) to add new types without changing the factory.

---

### ⭐ 3. Abstract Factory — *"Factory of related factories"*

**Concept:** Groups related factories under one interface. Guarantees that products from one factory are compatible (e.g., all Windows UI widgets together, not a mix of Win+Mac).

**Why "Abstract"?** The factory itself is abstract — you depend on the *interface*, not a specific OS's factory.

**When to use:** Cross-platform UI kits, theme engines, database driver families.

**Pattern to spot:** An interface with multiple `makeXxx()` methods, one concrete class per "family".

```cpp
struct Button   { virtual void click() = 0; virtual ~Button() = default; };
struct Checkbox { virtual void check() = 0; virtual ~Checkbox() = default; };

struct WinButton   : Button   { void click() override { std::cout << "Win click\n";  } };
struct WinCheckbox : Checkbox { void check() override { std::cout << "Win check\n";  } };

struct UIFactory {
    virtual std::unique_ptr<Button>   makeButton()   = 0;
    virtual std::unique_ptr<Checkbox> makeCheckbox() = 0;
    virtual ~UIFactory() = default;
};

struct WinFactory : UIFactory {
    std::unique_ptr<Button>   makeButton()   override { return std::make_unique<WinButton>();   }
    std::unique_ptr<Checkbox> makeCheckbox() override { return std::make_unique<WinCheckbox>(); }
};

// Usage: swap WinFactory → MacFactory to switch entire UI family
std::unique_ptr<UIFactory> factory = std::make_unique<WinFactory>();
factory->makeButton()->click();  // Win click
```

> 💡 **Tip:** Factory Method = one product. Abstract Factory = a *family* of products. If you only need one type, use Factory Method.

---

### ⭐ 4. Builder — *"Construct a complex object step-by-step"*

**Concept:** Separates construction from representation. Lets you build the same product differently (e.g., XML builder vs JSON builder) using the same steps.

**Why "Builder"?** Like a house builder who follows blueprints step-by-step rather than materializing the whole house at once.

**When to use:** Objects with many optional fields (avoid telescoping constructors), complex multi-step assembly (meal, HTTP request, query).

**Pattern to spot:** Chained setter methods returning `*this` (fluent API), finalized by a `build()` call.

```cpp
struct Car {
    std::string engine, color;
    int wheels = 4;
};

class CarBuilder {
    Car car;
public:
    CarBuilder& setEngine(std::string e) { car.engine = e; return *this; }
    CarBuilder& setColor(std::string c)  { car.color  = c; return *this; }
    CarBuilder& setWheels(int w)         { car.wheels = w; return *this; }
    Car build() { return car; }
};

Car c = CarBuilder().setEngine("V8").setColor("Red").setWheels(4).build();
```

> 💡 **Tip:** Modern C++ can use designated initializers `Car{.engine="V8", .color="Red"}` for simple cases — reserve Builder for genuinely complex construction.

---

### 5. Prototype — *"Clone an existing object instead of constructing from scratch"*

**Concept:** When object creation is expensive (deep graph, DB lookup), clone a pre-built prototype and tweak only what differs.

**Why "Prototype"?** Like a biological prototype cell that replicates itself.

**When to use:** Config variants, game entity templates, expensive-to-init objects.

**Pattern to spot:** A `clone()` virtual method returning a copy of itself.

```cpp
struct Config {
    std::string host;
    int port;
    virtual std::unique_ptr<Config> clone() const {
        return std::make_unique<Config>(*this);  // copy-construct
    }
};

Config base{"localhost", 8080};
auto devConfig = base.clone();
devConfig->port = 9090;  // base unchanged
```

> 💡 **Tip:** Prefer deep copy in `clone()`. In C++, the copy constructor does this automatically for value types.

---

## Structural Patterns

### ⭐ 6. Adapter — *"Convert one interface into another"*

**Concept:** Wraps an incompatible class to make it look like the interface the client expects. No changes to either side.

**Why "Adapter"?** Like a power socket adapter — same electricity, different plug shape.

**When to use:** Integrating legacy/third-party code into a modern interface.

**Pattern to spot:** A class that inherits the *target* interface but holds and delegates to the *adaptee*.

```cpp
class OldPrinter { public: void printText(const char* t) { printf("%s", t); } };

struct Printer { virtual void print(const std::string& t) = 0; virtual ~Printer() = default; };

class PrinterAdapter : public Printer {
    OldPrinter old;
public:
    void print(const std::string& t) override { old.printText(t.c_str()); }
};

std::unique_ptr<Printer> p = std::make_unique<PrinterAdapter>();
p->print("hello");
```

> 💡 **Tip:** Adapter vs Facade — Adapter makes *one* incompatible class usable. Facade simplifies *many* classes into one entry point.

---

### ⭐ 7. Decorator — *"Stack responsibilities at runtime"*

**Concept:** Wraps an object with the same interface and adds behavior before/after. Stacking multiple decorators is like layering.

**Why "Decorator"?** Like decorating a plain room — you add curtains, furniture, painting on top; the room's structure is unchanged.

**When to use:** Adding orthogonal features dynamically (logging, caching, compression, auth) without subclass explosion.

**Pattern to spot:** Constructor takes a `unique_ptr<SameInterface>`, stores it, delegates + extends.

```cpp
struct Coffee {
    virtual std::string desc() = 0;
    virtual double cost() = 0;
    virtual ~Coffee() = default;
};

struct Plain : Coffee {
    std::string desc() override { return "Coffee"; }
    double cost() override { return 1.0; }
};

struct MilkDecorator : Coffee {
    std::unique_ptr<Coffee> base;
    MilkDecorator(std::unique_ptr<Coffee> c) : base(std::move(c)) {}
    std::string desc() override { return base->desc() + " + Milk"; }
    double cost() override { return base->cost() + 0.5; }
};

auto c = std::make_unique<MilkDecorator>(std::make_unique<Plain>());
// c->desc() = "Coffee + Milk", c->cost() = 1.5
```

> 💡 **Tip:** Decorator = runtime wrapper chain. Template Method = compile-time override. Choose Decorator when behavior is opt-in and combinable.

---

### ⭐ 8. Facade — *"One simple door into a complex system"*

**Concept:** Provides a unified, simplified interface to a subsystem of classes. Hides complexity; doesn't add new capability.

**Why "Facade"?** From French *façade* (front face of a building) — you see only the clean front, not the plumbing behind.

**When to use:** Simplify startup, API entry points, SDK wrappers (e.g., `VideoPlayer.play()` hides codec/renderer pipeline).

**Pattern to spot:** A class that owns multiple subsystem objects and exposes high-level methods.

```cpp
class VideoDecoder { public: void decode() { /* ... */ } };
class AudioDecoder { public: void decode() { /* ... */ } };
class Renderer     { public: void render() { /* ... */ } };

class VideoPlayer {           // Facade
    VideoDecoder vd; AudioDecoder ad; Renderer r;
public:
    void play(const std::string& file) { vd.decode(); ad.decode(); r.render(); }
};

VideoPlayer player;
player.play("movie.mp4");    // Client knows nothing about decoders
```

---

### 9. Composite — *"Treat leaf and container uniformly"*

**Concept:** Build tree structures where individual items and groups of items are treated identically via a common interface.

**Why "Composite"?** A composite object is made of composed parts — but it exposes the same face as a single part.

**When to use:** File systems, UI widget trees, scene graphs, org charts.

**Pattern to spot:** Base class with virtual `operation()`. Leaf just executes. Composite stores children and recursively calls `operation()`.

```cpp
struct FSItem { virtual void show(int depth = 0) = 0; virtual ~FSItem() = default; };

struct File : FSItem {
    std::string name;
    void show(int d) override { std::cout << std::string(d*2,' ') << name << "\n"; }
};

struct Folder : FSItem {
    std::string name;
    std::vector<std::unique_ptr<FSItem>> children;
    void add(std::unique_ptr<FSItem> i) { children.push_back(std::move(i)); }
    void show(int d) override {
        std::cout << std::string(d*2,' ') << "[" << name << "]\n";
        for (auto& c : children) c->show(d + 1);  // Recursive — same interface
    }
};
```

---

### ⭐ 10. Proxy — *"A stand-in that controls access to the real object"*

**Concept:** Proxy sits between the client and the real object. It intercepts calls and can add lazy loading, access control, logging, or caching — without the client knowing.

**Why "Proxy"?** Like a legal proxy — you authorize someone to act *on your behalf*. The client talks to the proxy, not the real object directly.

**When to use:** Lazy init of heavy objects, access guards, remote object stubs, smart references.

**Pattern to spot:** Proxy and RealObject share the same interface; Proxy holds a pointer to RealObject.

```cpp
struct Image { virtual void display() = 0; virtual ~Image() = default; };

struct RealImage : Image {
    std::string file;
    RealImage(std::string f) : file(f) { std::cout << "Loading " << f << "\n"; }  // Expensive
    void display() override { std::cout << "Displaying " << file << "\n"; }
};

struct ImageProxy : Image {   // Same interface as RealImage
    std::string file;
    std::unique_ptr<RealImage> real;  // Null until first use
    ImageProxy(std::string f) : file(f) {}
    void display() override {
        if (!real) real = std::make_unique<RealImage>(file);  // Lazy load
        real->display();
    }
};

ImageProxy img("photo.jpg");  // No loading yet
img.display();                // Loading photo.jpg → Displaying photo.jpg
img.display();                // Displaying photo.jpg  (already loaded)
```

> 💡 **Tip:** Proxy vs Decorator — both wrap the same interface, but purpose differs: Proxy *controls access*, Decorator *adds behavior*.

---

### 11. Bridge — *"Decouple abstraction from implementation"*

**Concept:** Splits a large class (or family) into two separate hierarchies — **abstraction** (what) and **implementation** (how) — so they can vary independently.

**Why "Bridge"?** A bridge connects two separate structures. Here it connects the abstraction hierarchy to the implementation hierarchy without fusing them.

**When to use:** When both the abstraction and implementation need to be extensible independently. E.g., shapes × renderers, devices × remote controls.

**Pattern to spot:** Abstraction holds a pointer to `Implementor` interface. Both sides have separate class hierarchies.

```cpp
struct Renderer {  // Implementation hierarchy
    virtual void drawCircle(float r) = 0;
    virtual ~Renderer() = default;
};
struct OpenGLRenderer : Renderer { void drawCircle(float r) override { std::cout << "GL Circle r=" << r << "\n"; } };
struct VulkanRenderer : Renderer { void drawCircle(float r) override { std::cout << "VK Circle r=" << r << "\n"; } };

struct Shape {  // Abstraction hierarchy
    Renderer* rnd;  // Bridge to implementation
    Shape(Renderer* r) : rnd(r) {}
    virtual void draw() = 0;
};
struct Circle : Shape {
    float radius;
    Circle(Renderer* r, float rad) : Shape(r), radius(rad) {}
    void draw() override { rnd->drawCircle(radius); }  // Delegates to impl
};

OpenGLRenderer gl;
Circle c(&gl, 5.0f);
c.draw();  // GL Circle r=5
```

> 💡 **Tip:** Bridge ≈ Dependency Injection at the hierarchy level. It's the pattern behind "program to interfaces" applied to two dimensions simultaneously.

---

### 12. Flyweight — *"Share common state to save memory"*

**Concept:** When many objects share identical data, extract that shared (intrinsic) state into a single shared object. Each instance only stores its unique (extrinsic) state.

**Why "Flyweight"?** From boxing — a flyweight is extremely lightweight. The pattern makes many objects "weigh" less by sharing their shared parts.

**When to use:** Rendering millions of particles/characters/game tiles, font glyph caching, string interning.

**Pattern to spot:** A factory that returns existing objects from a cache instead of creating new ones.

```cpp
struct GlyphData {  // Intrinsic (shared) — shape, bitmap — same for all 'A's
    char ch; std::string bitmap;
};

class GlyphFactory {  // Flyweight factory / cache
    std::map<char, std::shared_ptr<GlyphData>> cache;
public:
    std::shared_ptr<GlyphData> get(char c) {
        if (!cache.count(c)) cache[c] = std::make_shared<GlyphData>(GlyphData{c, "bitmap_of_" + std::string(1,c)});
        return cache[c];  // Return shared instance
    }
};

// Extrinsic (unique) state: position on screen
struct Glyph {
    std::shared_ptr<GlyphData> data;  // Shared
    int x, y;                         // Unique per glyph instance
};

GlyphFactory factory;
Glyph a1{factory.get('A'), 0,  0};
Glyph a2{factory.get('A'), 10, 0};  // Same GlyphData object reused
```

> 💡 **Tip:** Intrinsic = shared, immutable. Extrinsic = unique, passed in by client. If you see a pool/cache returning shared objects — it's Flyweight.

---

## Behavioral Patterns

### ⭐ 10. Observer — *"Publish once, all subscribers react"*

**Concept:** An object (Subject) maintains a list of dependents (Observers) and notifies them automatically when its state changes. Decouples sender from receivers.

**Why "Observer"?** Observers *watch* the subject and react to events — like a news subscriber getting alerts.

**When to use:** Event systems, UI data-binding (MVC), pub/sub, reactive patterns.

**Pattern to spot:** `subscribe()/unsubscribe()` + `notify()` calling `update()` on each observer.

```cpp
struct Observer { virtual void update(const std::string& event) = 0; virtual ~Observer() = default; };

class EventBus {
    std::vector<Observer*> subs;
public:
    void subscribe(Observer* o)   { subs.push_back(o); }
    void unsubscribe(Observer* o) { subs.erase(std::remove(subs.begin(), subs.end(), o), subs.end()); }
    void notify(const std::string& e) { for (auto* o : subs) o->update(e); }
};

struct Logger   : Observer { void update(const std::string& e) override { std::cout << "LOG: " << e << "\n"; } };
struct Analytics: Observer { void update(const std::string& e) override { std::cout << "STAT: " << e << "\n"; } };

EventBus bus;
Logger log; Analytics ana;
bus.subscribe(&log); bus.subscribe(&ana);
bus.notify("OnClick");   // Both receive the event
```

> 💡 **Tip:** Use `weak_ptr` for observers to avoid dangling pointers when subscribers die before the subject.

---

### ⭐ 11. Strategy — *"Swap the algorithm, not the context"*

**Concept:** Define a family of algorithms behind an interface. Inject the desired one at runtime. Context delegates to it.

**Why "Strategy"?** A strategy is a chosen plan of action — you pick one approach and execute it.

**When to use:** Sorting, compression, payment processing, rendering modes — any place where the *how* varies independently of the *what*.

**Pattern to spot:** Context holds a `unique_ptr<Strategy>`, calls `strategy->execute()`. Strategies are interchangeable.

```cpp
struct SortStrategy { virtual void sort(std::vector<int>& v) = 0; virtual ~SortStrategy() = default; };

struct STLSort    : SortStrategy { void sort(std::vector<int>& v) override { std::sort(v.begin(), v.end()); } };
struct BubbleSort : SortStrategy {
    void sort(std::vector<int>& v) override {
        for (size_t i = 0; i < v.size(); i++)
            for (size_t j = 0; j < v.size()-1; j++)
                if (v[j] > v[j+1]) std::swap(v[j], v[j+1]);
    }
};

class Sorter {
    std::unique_ptr<SortStrategy> s;
public:
    Sorter(std::unique_ptr<SortStrategy> s) : s(std::move(s)) {}
    void setStrategy(std::unique_ptr<SortStrategy> ns) { s = std::move(ns); }
    void sort(std::vector<int>& v) { s->sort(v); }
};

Sorter sorter(std::make_unique<STLSort>());
sorter.setStrategy(std::make_unique<BubbleSort>());  // Swap at runtime
```

> 💡 **Tip:** In modern C++ you can skip the strategy class entirely — store a `std::function<void(vector<int>&)>` and pass lambdas.

---

### ⭐ 12. Command — *"Encapsulate a request as an object"*

**Concept:** Wraps an action (and its parameters) into an object. Enables undo/redo, queuing, and logging of operations.

**Why "Command"?** The object *is* the command — it carries everything needed to execute (or reverse) an action.

**When to use:** Undo/redo (text editors, CAD), macro recording, task queues, transactional systems.

**Pattern to spot:** Interface with `execute()` + `undo()`. Stack of commands = undo history.

```cpp
struct Command { virtual void execute() = 0; virtual void undo() = 0; virtual ~Command() = default; };

class TextEditor {
    std::string text;
public:
    void add(const std::string& t)  { text += t; }
    void remove(size_t n)           { if (n <= text.size()) text.resize(text.size() - n); }
    const std::string& get() const  { return text; }
};

class TypeCmd : public Command {
    TextEditor& ed; std::string typed;
public:
    TypeCmd(TextEditor& e, std::string t) : ed(e), typed(t) {}
    void execute() override { ed.add(typed); }
    void undo()    override { ed.remove(typed.size()); }
};

TextEditor editor;
std::stack<std::unique_ptr<Command>> history;

auto cmd = std::make_unique<TypeCmd>(editor, "Hello");
cmd->execute(); history.push(std::move(cmd));  // editor = "Hello"
history.top()->undo(); history.pop();          // editor = ""
```

---

### 13. Template Method — *"Fix the skeleton, vary the steps"*

**Concept:** Base class defines the overall algorithm structure (the skeleton). Abstract steps are filled in by subclasses. Prevents code duplication of the shared flow.

**Why "Template Method"?** The `mine()` method acts as a *template* for the algorithm — fixed scaffolding with variable steps.

**When to use:** Data pipelines (open → parse → process → close), game loops, test frameworks (setUp/test/tearDown).

**Pattern to spot:** A non-virtual `run()` in base class calling virtual private steps.

```cpp
class DataMiner {
public:
    void mine() {       // Skeleton: always the same order
        open();
        extract();      // Varies per format
        parse();        // Varies per format
        close();
    }
private:
    void open()  { std::cout << "Opening\n"; }
    void close() { std::cout << "Closing\n"; }
    virtual void extract() = 0;
    virtual void parse()   = 0;
};

class CSVMiner : public DataMiner {
    void extract() override { std::cout << "Read CSV rows\n"; }
    void parse()   override { std::cout << "Parse CSV\n"; }
};
```

> 💡 **Tip:** Inverse of Strategy — here the algorithm structure is fixed in the base, only steps vary. Strategy replaces the whole algorithm.

---

### 14. CRTP — *"Static polymorphism via self-referencing template"*

**Concept:** A derived class passes itself as a template argument to its base. The base calls methods on the derived type at **compile time** — no virtual dispatch, no vtable.

**Why "Curiously Recurring"?** It's unusual (*curious*) that a class template `Base<Derived>` is inherited by `Derived` itself — it references itself in its own definition.

**When to use:** High-performance code (hot loops, embedded), mixins adding behavior without virtual overhead.

**Pattern to spot:** `class Derived : public Base<Derived>` + `static_cast<Derived*>(this)->method()` in base.

```cpp
template<typename Derived>
class Animal {
public:
    void speak() { static_cast<Derived*>(this)->speakImpl(); }  // Compile-time dispatch
};

class Dog : public Animal<Dog> { public: void speakImpl() { std::cout << "Woof!\n"; } };
class Cat : public Animal<Cat> { public: void speakImpl() { std::cout << "Meow!\n"; } };

Dog d; d.speak();  // Woof! — zero vtable overhead
```

> 💡 **Tip:** Use `virtual` when you need runtime polymorphism (different types in one container). Use CRTP when the type is known at compile time and performance matters.

---

### ⭐ 15. State — *"Object changes behavior when its internal state changes"*

**Concept:** Instead of a giant `if/switch` on an enum, encapsulate each state as a class. The context delegates behavior to the current state object, which can transition to another state.

**Why "State"?** The pattern literally models state machines — each state is an object with its own behavior.

**When to use:** Vending machines, traffic lights, TCP connections, game AI, workflow engines.

**Pattern to spot:** Context holds `unique_ptr<State>`. Each state's method changes `context.state = newState`.

```cpp
struct TrafficLight;  // Forward declare

struct State { virtual void handle(TrafficLight& tl) = 0; virtual ~State() = default; };

struct TrafficLight {
    std::unique_ptr<State> state;
    TrafficLight(std::unique_ptr<State> initial) : state(std::move(initial)) {}
    void request() { state->handle(*this); }  // Delegates to current state
};

struct Red : State {
    void handle(TrafficLight& tl) override;
};
struct Green : State {
    void handle(TrafficLight& tl) override {
        std::cout << "Green → switching to Red\n";
        tl.state = std::make_unique<Red>();  // Transition
    }
};
void Red::handle(TrafficLight& tl) {
    std::cout << "Red → switching to Green\n";
    tl.state = std::make_unique<Green>();
}

TrafficLight tl(std::make_unique<Red>());
tl.request();  // Red → switching to Green
tl.request();  // Green → switching to Red
```

> 💡 **Tip:** State vs Strategy — Strategy is *externally* swapped; State *self-transitions* based on logic. If the object changes its own strategy, it's State.

---

### 16. Iterator — *"Traverse a collection without exposing its structure"*

**Concept:** Provides a standard way to access elements of a collection sequentially without knowing its internal layout (array, tree, list, etc.).

**Why "Iterator"?** It *iterates* — moves through items one by one, like a cursor or pointer that advances.

**When to use:** Custom containers, lazy sequences, graph traversal. (STL is entirely built on this pattern.)

**Pattern to spot:** `begin()/end()` returning iterator objects with `operator++`, `operator*`, `operator!=`.

```cpp
class NumberRange {
    int from, to;
public:
    NumberRange(int f, int t) : from(f), to(t) {}

    struct Iterator {
        int current;
        Iterator(int v) : current(v) {}
        int  operator*()  const { return current; }
        Iterator& operator++() { ++current; return *this; }
        bool operator!=(const Iterator& o) const { return current != o.current; }
    };

    Iterator begin() { return {from}; }
    Iterator end()   { return {to + 1}; }
};

for (int n : NumberRange(1, 5)) std::cout << n << " ";  // 1 2 3 4 5
// Range-for just calls begin/end — standard Iterator protocol
```

> 💡 **Tip:** Any class with `begin()/end()` works with range-based for. Iterator is the most prevalent pattern in C++ — STL containers are all Iterator-based.

---

### ⭐ 17. Chain of Responsibility — *"Pass a request along a chain until someone handles it"*

**Concept:** Each handler decides to handle the request or pass it to the next in chain. Sender doesn't know who will handle it.

**Why "Chain of Responsibility"?** The chain *shares* the responsibility — whoever is capable handles it, others pass it on.

**When to use:** Middleware pipelines, event bubbling, logging levels, authorization chains, UI event handling.

**Pattern to spot:** Each handler holds a pointer to the next handler. `handle()` either processes or forwards.

```cpp
struct Handler {
    Handler* next = nullptr;
    Handler* setNext(Handler* n) { next = n; return n; }  // Chaining setup
    virtual void handle(int level) {
        if (next) next->handle(level);  // Forward if unhandled
    }
    virtual ~Handler() = default;
};

struct LowHandler : Handler {
    void handle(int level) override {
        if (level <= 1) std::cout << "Low handler processed level " << level << "\n";
        else Handler::handle(level);  // Pass up
    }
};

struct HighHandler : Handler {
    void handle(int level) override {
        if (level > 1) std::cout << "High handler processed level " << level << "\n";
        else Handler::handle(level);
    }
};

LowHandler low; HighHandler high;
low.setNext(&high);      // Chain: low → high
low.handle(1);           // Low handler processed level 1
low.handle(3);           // High handler processed level 3
```

> 💡 **Tip:** This is how HTTP middleware works (`next()` in Express.js / filter chains in Java). Also how UI event bubbling propagates up the widget tree.

---

### 18. Mediator — *"Centralize communication between objects"*

**Concept:** Instead of components communicating with each other directly (tightly coupled N×N relationships), they all talk through a central mediator (N×1). Reduces dependencies.

**Why "Mediator"?** Like an air traffic controller — planes don't talk to each other; they relay all communication through the tower.

**When to use:** Chat rooms, UI form logic (one field affecting another), MVC Controllers, event dispatchers.

**Pattern to spot:** Components hold a reference to the Mediator. They call `mediator->notify(this, event)` instead of calling each other.

```cpp
struct Mediator;

struct Component {
    Mediator* med;
    Component(Mediator* m) : med(m) {}
    virtual void receive(const std::string& event) {}
};

struct Mediator {
    Component* c1 = nullptr; Component* c2 = nullptr;
    void notify(Component* sender, const std::string& e) {
        // Route: whatever c1 sends, c2 gets — and vice versa
        if (sender == c1 && c2) c2->receive(e);
        else if (sender == c2 && c1) c1->receive(e);
    }
};

struct Button : Component {
    using Component::Component;
    void click()  { std::cout << "Button clicked\n"; med->notify(this, "click"); }
};
struct Label  : Component {
    using Component::Component;
    void receive(const std::string& e) override { std::cout << "Label got: " << e << "\n"; }
};

Mediator med;
Button btn(&med); Label lbl(&med);
med.c1 = &btn; med.c2 = &lbl;
btn.click();  // Button clicked → Label got: click
```

> 💡 **Tip:** Mediator vs Observer — Observer broadcasts to all; Mediator routes selectively. Use Mediator when the routing logic is complex and two-way.

---

### ⭐ 19. Visitor — *"Add operations to objects without modifying them"*

**Concept:** Separates an algorithm from the object structure it operates on. You define a Visitor with a `visit()` overload for each concrete type. The object calls back with itself (`accept(visitor)` → `visitor.visit(*this)`).

**Why "Visitor"?** The visitor *visits* each node in a structure and performs an operation — like a tax auditor visiting departments.

**When to use:** AST traversal, scene graph rendering passes (CAD!), report generation over heterogeneous collections, serialization.

**Pattern to spot:** Double dispatch — `object.accept(v)` calls `v.visit(*this)`.

```cpp
struct Circle; struct Rectangle;  // Forward

struct Visitor {
    virtual void visit(Circle& c)    = 0;
    virtual void visit(Rectangle& r) = 0;
    virtual ~Visitor() = default;
};

struct Shape { virtual void accept(Visitor& v) = 0; virtual ~Shape() = default; };

struct Circle    : Shape { float r;   void accept(Visitor& v) override { v.visit(*this); } };
struct Rectangle : Shape { float w,h; void accept(Visitor& v) override { v.visit(*this); } };

struct AreaCalc : Visitor {  // New operation — no changes to Circle/Rectangle
    void visit(Circle& c)    override { std::cout << "Circle area: " << 3.14*c.r*c.r << "\n"; }
    void visit(Rectangle& r) override { std::cout << "Rect area: "   << r.w * r.h      << "\n"; }
};

std::vector<std::unique_ptr<Shape>> shapes;
shapes.push_back(std::make_unique<Circle>(Circle{5}));
shapes.push_back(std::make_unique<Rectangle>(Rectangle{4, 3}));

AreaCalc calc;
for (auto& s : shapes) s->accept(calc);  // Dispatches to correct visit()
```

> 💡 **Tip:** Visitor is perfect for CAD scenes — you can add rendering, export, validation passes without touching the geometry classes. The tradeoff: adding a new Shape type requires updating all Visitors.

---

### 20. Memento — *"Capture and restore an object's state"*

**Concept:** Saves a snapshot of an object's internal state so it can be restored later — without exposing internals. The stored snapshot is the *memento*.

**Why "Memento"?** A memento is a keepsake or memory token. Here it's a token holding the saved state.

**When to use:** Undo history (lighter alternative to Command when you just need state snapshots), game saves, transaction rollback.

**Pattern to spot:** Originator creates/restores from Memento. Caretaker stores the stack of mementos.

```cpp
struct Memento {  // Opaque snapshot — only Originator knows its internals
    std::string state;
};

class Editor {  // Originator
    std::string text;
public:
    void type(const std::string& t) { text += t; }
    Memento save()                  { return {text}; }        // Snapshot
    void restore(const Memento& m)  { text = m.state; }       // Rollback
    void show()                     { std::cout << text << "\n"; }
};

// Caretaker — manages history, doesn't know state internals
Editor ed;
std::stack<Memento> history;

ed.type("Hello"); history.push(ed.save());
ed.type(" World"); ed.show();    // Hello World
ed.restore(history.top()); history.pop();
ed.show();                       // Hello  ← rolled back
```

> 💡 **Tip:** Memento vs Command — Command stores the *operation* (and reverses it). Memento stores the *state* (and restores it). Use Memento when reversal logic is complex; Command when it's simple.

---

## Quick-Reference Cheat Sheet

| Pattern | Core Mechanism | Key Signal in Code |
|---------|---------------|-------------------|
| **Singleton** | Private ctor + static instance | `getInstance()` |
| **Factory Method** | Function returns base ptr | `create(string type)` |
| **Abstract Factory** | Interface of `makeX()` methods | One `Factory` per family |
| **Builder** | Fluent setters + `build()` | Method chaining |
| **Prototype** | `clone()` virtual method | `make_unique<T>(*this)` |
| **Adapter** | Wraps adaptee, inherits target | Has-a old, Is-a new |
| **Decorator** | Wraps same interface, extends | Ctor takes `unique_ptr<Same>` |
| **Facade** | Owns subsystems, simple API | One class, many internal objects |
| **Composite** | Recursive `operation()` | Leaf + Container share interface |
| **Proxy** | Same interface, controls access | Lazy ptr to real object |
| **Bridge** | Abstraction holds impl ptr | Two separate hierarchies |
| **Flyweight** | Cache/factory returns shared obj | Intrinsic vs extrinsic state |
| **Observer** | `subscribe/notify/update` | Event bus / pub-sub |
| **Strategy** | Inject `unique_ptr<Algorithm>` | `setStrategy()` |
| **Command** | `execute()` + `undo()` | History stack |
| **Template Method** | Non-virtual skeleton + virtual steps | `run()` calls `doStep()` |
| **CRTP** | `class D : Base<D>` | `static_cast<Derived*>(this)` |
| **State** | Context delegates to State object | `state->handle(*this)` |
| **Iterator** | `begin()/end()` + `operator++` | Custom container traversal |
| **Chain of Responsibility** | Handlers forward request down chain | `next->handle(req)` |
| **Mediator** | Central hub, components notify it | `mediator->notify(this, event)` |
| **Visitor** | Double dispatch: `accept(v)/visit()` | Operation separate from structure |
| **Memento** | Snapshot object saved externally | `save()` / `restore(memento)` |

---

## Most Important for Interviews (ranked)

### 🔴 Must Know (asked very frequently)
1. **Observer** — pub/sub, event systems, weak_ptr pitfalls
2. **Factory / Abstract Factory** — OCP, creation decoupling
3. **Singleton** — thread safety, Meyer's trick, DI tradeoffs
4. **Strategy** — inject behavior, `std::function` alternative
5. **Command** — undo/redo, CAD/editor systems
6. **State** — state machines, replaces switch/enum chains
7. **Decorator** — middleware, AOP, wrapper chains

### 🟡 Should Know (commonly discussed)
8. **Adapter** — legacy integration, interface bridging
9. **Builder** — complex construction, fluent API
10. **Composite** — scene graphs, UI trees (**Autodesk-relevant!**)
11. **Proxy** — lazy load, access control, smart references
12. **Visitor** — AST traversal, scene operations (**CAD-relevant!**)
13. **Chain of Responsibility** — middleware pipelines, event bubbling
14. **Template Method** — framework hooks, algorithm skeletons
15. **Memento** — snapshots, undo (vs Command tradeoff)

### 🟢 Good to Know (niche / C++-specific)
16. **CRTP** — zero-overhead polymorphism, performance-critical paths
17. **Bridge** — multi-dimensional extensibility
18. **Flyweight** — memory optimization, particle/glyph systems
19. **Mediator** — complex UI form logic, chat rooms
20. **Iterator** — always in use via STL, rarely designed from scratch
21. **Prototype** — cloning heavy objects, config templates
