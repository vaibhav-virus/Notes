# STL Quick Reference

## `std::stack<T>`

- **LIFO** (Last In, First Out). Backed by `std::deque` by default.
- No random access. Only top element is accessible.

```cpp
#include <stack>
std::stack<int> s;

s.push(10);    // [10]
s.push(20);    // [10, 20]  ← top
s.push(30);    // [10, 20, 30] ← top

s.top();       // 30  — peek without removing
s.pop();       // Removes 30. Stack: [10, 20]
s.size();      // 2
s.empty();     // false
```

### Quick Code Question — Valid Parentheses

```cpp
// Check if brackets are balanced: '(', ')', '{', '}', '[', ']'
bool isValid(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '{' || c == '[') st.push(c);  // Push opening
        else {
            if (st.empty()) return false;  // Closing with nothing open
            char top = st.top(); st.pop();
            if ((c == ')' && top != '(') ||
                (c == '}' && top != '{') ||
                (c == ']' && top != '[')) return false;
        }
    }
    return st.empty();  // All opened must be closed
}
// isValid("()[]{}") → true | isValid("([)]") → false
```

---

## `std::queue<T>`

- **FIFO** (First In, First Out). Backed by `std::deque` by default.
- Used in BFS, task scheduling.

```cpp
#include <queue>
std::queue<int> q;

q.push(10);    // [10]
q.push(20);    // [10, 20]
q.push(30);    // [10, 20, 30]

q.front();     // 10  — oldest element
q.back();      // 30  — newest element
q.pop();       // Removes 10 (front). Queue: [20, 30]
q.size();      // 2
q.empty();     // false
```

### `std::priority_queue<T>` — Max Heap by default

```cpp
std::priority_queue<int> maxHeap;
maxHeap.push(5); maxHeap.push(1); maxHeap.push(9);
maxHeap.top();   // 9 — largest always at top

// Min-Heap:
std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;
minHeap.push(5); minHeap.push(1); minHeap.push(9);
minHeap.top();   // 1 — smallest at top
```

### Quick Code Question — Sliding Window Maximum

```cpp
// Return max of each window of size k using deque
std::vector<int> maxSlidingWindow(std::vector<int>& nums, int k) {
    std::deque<int> dq;  // Stores indices; front = index of max in current window
    std::vector<int> result;
    for (int i = 0; i < (int)nums.size(); ++i) {
        // Remove indices out of window
        if (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        // Remove smaller elements — they can never be max
        while (!dq.empty() && nums[dq.back()] < nums[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) result.push_back(nums[dq.front()]);
    }
    return result;
}
// nums=[1,3,-1,-3,5,3,6,7], k=3 → [3,3,5,5,6,7]
```

---

## `std::vector<T>`

- Dynamic array. Contiguous memory. `O(1)` random access. `O(1)` amortized push_back.
- When capacity exceeded, doubles capacity and reallocates.

```cpp
#include <vector>
std::vector<int> v = {1, 2, 3};

v.push_back(4);    // {1, 2, 3, 4}
v.pop_back();      // {1, 2, 3}
v[1];              // 2  — O(1) access
v.at(1);           // 2  — O(1) access with bounds check (throws std::out_of_range)
v.front();         // 1
v.back();          // 3
v.size();          // 3
v.capacity();      // ≥ 3 (may be larger due to pre-allocation)
v.reserve(100);    // Reserve space for 100 elements to avoid re-allocations
v.resize(5, 0);    // {1, 2, 3, 0, 0} — resize and fill new spots with 0
v.erase(v.begin() + 1);  // Remove element at index 1 — O(n) shift
v.insert(v.begin(), 0);  // Insert 0 at front — O(n) shift
v.clear();         // Empties the vector (size=0, capacity unchanged)
```

#### Iterating

```cpp
for (int x : v) std::cout << x;           // Range-based for
for (auto it = v.begin(); it != v.end(); ++it) std::cout << *it;  // Iterator
```

### Quick Code Question — Two Sum

```cpp
// Return indices of two numbers that add up to target
std::vector<int> twoSum(std::vector<int>& nums, int target) {
    std::unordered_map<int, int> seen;  // value → index
    for (int i = 0; i < (int)nums.size(); ++i) {
        int complement = target - nums[i];
        if (seen.count(complement)) return {seen[complement], i};  // Found pair
        seen[nums[i]] = i;
    }
    return {};
}
// nums=[2,7,11,15], target=9 → [0,1]
```

---

## `std::unordered_map<K, V>` (HashMap)

- Key-value store. Average `O(1)` insert/lookup/delete. Worst case `O(n)` (hash collision).
- Backed by a hash table. Keys must be hashable.

```cpp
#include <unordered_map>
std::unordered_map<std::string, int> freq;

freq["apple"] = 3;      // Insert or update
freq["banana"]++;       // Increment (auto-initialises to 0 if key doesn't exist)
freq.count("apple");    // 1 if key exists, 0 otherwise (prefer over find for bool check)
freq.find("apple");     // Returns iterator; == freq.end() if not found

// Safe access
if (freq.count("mango"))
    int x = freq["mango"];  // Only access if key exists; [] inserts default otherwise

freq.erase("banana");   // Remove key

for (auto& [key, val] : freq)  // Structured binding (C++17)
    std::cout << key << ": " << val << "\n";
```

### `std::map<K, V>` vs `std::unordered_map<K, V>`

| Feature | `std::map` | `std::unordered_map` |
|---|---|---|
| Ordering | Sorted (BST) | Unordered (hash table) |
| Lookup | `O(log n)` | `O(1)` average |
| Custom key | Needs `operator<` | Needs `std::hash<K>` |
| Use when | Need sorted iteration | Need fast lookup |

### Quick Code Question — Group Anagrams

```cpp
// Group strings that are anagrams of each other
std::vector<std::vector<std::string>> groupAnagrams(std::vector<std::string>& strs) {
    std::unordered_map<std::string, std::vector<std::string>> groups;
    for (auto& s : strs) {
        std::string key = s;
        std::sort(key.begin(), key.end());  // Sorted string is the canonical key for anagrams
        groups[key].push_back(s);
    }
    std::vector<std::vector<std::string>> result;
    for (auto& [k, v] : groups) result.push_back(v);
    return result;
}
// ["eat","tea","tan","ate","nat","bat"] → [["eat","tea","ate"],["tan","nat"],["bat"]]
```

---

## `std::set<T>` and `std::unordered_set<T>` — Quick Reference

```cpp
#include <set>
std::set<int> s = {3, 1, 4, 1, 5};  // {1, 3, 4, 5} — sorted, no duplicates
s.insert(2);     // {1, 2, 3, 4, 5}
s.erase(3);      // {1, 2, 4, 5}
s.count(4);      // 1
s.find(4);       // iterator to 4, or s.end() if not found

#include <unordered_set>
std::unordered_set<int> us = {3, 1, 4};  // Unordered, O(1) average lookup
```
