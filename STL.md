# STL — Standard Template Library

## Container Quick Reference

| Container | Ordered? | Key Lookup | Insert | Use When |
|---|---|---|---|---|
| `vector` | Yes (index) | O(n) | O(1) amortized back | Default sequence container |
| `deque` | Yes (index) | O(n) | O(1) front/back | Queue with O(1) front insert |
| `list` | Yes (iter) | O(n) | O(1) anywhere | Frequent mid-insert/erase |
| `array` | Yes (index) | O(n) | Fixed size | Fixed-size stack array |
| `set` | Sorted | O(log n) | O(log n) | Unique sorted elements |
| `multiset` | Sorted | O(log n) | O(log n) | Sorted, duplicates allowed |
| `map` | Sorted by key | O(log n) | O(log n) | Key-value, sorted |
| `unordered_set` | No | O(1) avg | O(1) avg | Fast unique lookup |
| `unordered_map` | No | O(1) avg | O(1) avg | Fast key-value lookup |
| `stack` | LIFO | – | O(1) push/pop | LIFO access |
| `queue` | FIFO | – | O(1) push/pop | FIFO access |
| `priority_queue` | Heap | O(1) top | O(log n) | Max/Min element always on top |

---

## std::vector

```cpp
#include <vector>

std::vector<int> v{3, 1, 4, 1, 5};

v.push_back(9);          // Add to end                [3,1,4,1,5,9]
v.emplace_back(2);       // Construct in-place at end [3,1,4,1,5,9,2]
v.pop_back();            // Remove last element
v.insert(v.begin(), 0); // Insert 0 at front         [0,3,1,4,1,5,9]
v.erase(v.begin());      // Erase first element       [3,1,4,1,5,9]
v.size();                // Number of elements
v.capacity();            // Allocated capacity (may be > size)
v.reserve(100);          // Pre-allocate — avoid reallocs!
v.resize(10, 0);         // Resize to 10, fill new slots with 0
v.clear();               // Remove all elements (capacity unchanged)
v[2];                    // Access (no bounds check)
v.at(2);                 // Access with bounds check (throws std::out_of_range)
v.front();               // First element
v.back();                // Last element
v.data();                // Raw pointer to underlying array
```

---

## std::map — Sorted Key-Value

```cpp
#include <map>

std::map<std::string, int> scores;

scores["Alice"] = 90;                    // Insert or update
scores.insert({"Bob", 85});              // Insert only (won't overwrite)
scores.emplace("Charlie", 78);           // Most efficient insert

scores.count("Alice");                   // 1 if exists, 0 if not
scores.find("Alice");                    // Returns iterator (or end() if not found)

// Safe lookup pattern
if (auto it = scores.find("Dave"); it != scores.end())
    std::cout << it->second;

// Iteration — always in sorted key order
for (const auto& [name, score] : scores)
    std::cout << name << ": " << score << "\n";

scores.erase("Alice");                  // Remove by key
scores.size();                          // Number of entries
```

---

## std::unordered_map — Hash Map (O(1) avg lookup)

```cpp
#include <unordered_map>

std::unordered_map<std::string, int> freq;
freq["apple"]++;          // Increment count
freq["banana"] = 3;

// Same API as map, but O(1) avg vs O(log n)
// NO guaranteed order for iteration
// Use when: you don't need sorted order and want speed

// Custom hash for user types
struct PairHash {
    size_t operator()(const std::pair<int,int>& p) const {
        return std::hash<int>()(p.first) ^ (std::hash<int>()(p.second) << 1);
    }
};
std::unordered_map<std::pair<int,int>, int, PairHash> grid;
```

---

## std::set & std::unordered_set

```cpp
#include <set>

std::set<int> s{5, 3, 1, 4, 2};  // Automatically sorted: {1,2,3,4,5}

s.insert(3);      // Already exists — no duplicate inserted
s.erase(3);       // Remove 3
s.count(4);       // 1 if present, 0 if not
s.find(4);        // Iterator to element or end()

// Common use: check if element was seen before
std::unordered_set<int> seen;
for (int x : nums) {
    if (seen.count(x)) { /* duplicate! */ }
    seen.insert(x);
}
```

---

## std::priority_queue — Max Heap by Default

```cpp
#include <queue>

// MAX heap: largest element on top
std::priority_queue<int> maxPQ;
maxPQ.push(3);
maxPQ.push(1);
maxPQ.push(10);
maxPQ.top();    // 10 — largest
maxPQ.pop();    // Remove 10

// MIN heap: smallest element on top
std::priority_queue<int, std::vector<int>, std::greater<int>> minPQ;
minPQ.push(3);
minPQ.push(1);
minPQ.top();   // 1 — smallest

// Custom comparator for struct
struct Task { int priority; std::string name; };
auto cmp = [](const Task& a, const Task& b) { return a.priority < b.priority; };
std::priority_queue<Task, std::vector<Task>, decltype(cmp)> taskQ(cmp);
```

---

## std::stack & std::queue

```cpp
// Stack — LIFO
std::stack<int> st;
st.push(1); st.push(2); st.push(3);
st.top();   // 3 — peek
st.pop();   // Remove 3
st.empty(); // false

// Queue — FIFO
std::queue<int> q;
q.push(1); q.push(2); q.push(3);
q.front();  // 1 — peek front
q.back();   // 3 — peek back
q.pop();    // Remove front (1)
```

---

## STL Algorithms

```cpp
#include <algorithm>
#include <numeric>

std::vector<int> v{3, 1, 4, 1, 5, 9, 2, 6};

// Sorting
std::sort(v.begin(), v.end());                          // Ascending
std::sort(v.begin(), v.end(), std::greater<int>());     // Descending
std::sort(v.begin(), v.end(), [](int a, int b){ return a > b; }); // Custom

// Searching
auto it = std::find(v.begin(), v.end(), 5);             // Linear search
auto it2 = std::binary_search(v.begin(), v.end(), 5);  // Requires sorted vector
auto lb = std::lower_bound(v.begin(), v.end(), 5);      // First element >= 5
auto ub = std::upper_bound(v.begin(), v.end(), 5);      // First element > 5

// Transforms
std::transform(v.begin(), v.end(), v.begin(),           // Square each element
               [](int x){ return x * x; });

// Filter (copy_if)
std::vector<int> evens;
std::copy_if(v.begin(), v.end(), std::back_inserter(evens),
             [](int x){ return x % 2 == 0; });

// Count
int count5 = std::count(v.begin(), v.end(), 5);
int countEven = std::count_if(v.begin(), v.end(), [](int x){ return x%2==0; });

// Accumulate (sum/product)
int sum = std::accumulate(v.begin(), v.end(), 0);       // Sum with initial value 0
int prod = std::accumulate(v.begin(), v.end(), 1,
                           [](int a, int b){ return a * b; }); // Product

// Min/Max
int minVal = *std::min_element(v.begin(), v.end());
int maxVal = *std::max_element(v.begin(), v.end());
auto [mn, mx] = std::minmax_element(v.begin(), v.end()); // C++11

// Remove duplicates (sort first, then unique + erase)
std::sort(v.begin(), v.end());
v.erase(std::unique(v.begin(), v.end()), v.end());

// Reverse
std::reverse(v.begin(), v.end());

// Fill / generate
std::fill(v.begin(), v.end(), 0);              // Fill all with 0
int n = 0;
std::generate(v.begin(), v.end(), [&]{ return n++; }); // 0,1,2,3...

// any_of / all_of / none_of
bool anyEven  = std::any_of(v.begin(), v.end(),  [](int x){ return x%2==0; });
bool allPos   = std::all_of(v.begin(), v.end(),  [](int x){ return x > 0; });
bool noneNeg  = std::none_of(v.begin(), v.end(), [](int x){ return x < 0; });
```

---

## Iterators

```cpp
std::vector<int> v{1, 2, 3, 4, 5};

// Forward iteration
for (auto it = v.begin(); it != v.end(); ++it)
    std::cout << *it;

// Reverse iteration
for (auto it = v.rbegin(); it != v.rend(); ++it)
    std::cout << *it;  // 5, 4, 3, 2, 1

// Iterator categories (matters for algorithm complexity)
// InputIterator:    read-once, forward (istream)
// ForwardIterator:  read multi-pass (forward_list)
// BidirectionalIt:  ++/-– (list, map)
// RandomAccessIt:   +n, -n, [] (vector, deque, array)

// std::advance and std::distance
auto it = v.begin();
std::advance(it, 2);           // Move it 2 positions forward → points to v[2]=3
int dist = std::distance(v.begin(), it); // 2
```

---

## std::string Useful Methods

```cpp
std::string s = "Hello, World!";

s.size();           // 13
s.length();         // same as size()
s.empty();          // false
s.find("World");    // 7 (index), returns string::npos if not found
s.substr(7, 5);     // "World" (start=7, length=5)
s.replace(7, 5, "C++"); // "Hello, C++!"
s.append(" More");  // Append
s += " More";       // Same as append
s.erase(5, 2);      // Remove ", " → "HelloWorld!"
s.insert(5, ", ");  // Insert at index
s.rfind('l');       // Last occurrence index
std::stoi(s);       // String to int (throws if invalid)
std::to_string(42); // Int to string

// Case conversion (no built-in — use transform)
std::transform(s.begin(), s.end(), s.begin(), ::toupper);

// Split string (no built-in — use stringstream)
#include <sstream>
std::string line = "a,b,c";
std::stringstream ss(line);
std::string token;
while (std::getline(ss, token, ','))  // Split by ','
    std::cout << token << "\n";       // a, b, c
```
