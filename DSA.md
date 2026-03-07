# Data Structures & Algorithms — Interview Prep Guide

> **Philosophy**: Learn to *recognize patterns*, not memorize solutions.  
> Each section = 1 core concept → how to spot it → solved example → key takeaways.  
> Difficulty: **Easy → Medium**. No hard DP or advanced graph theory.

---

## 🗺️ Pattern Recognition Map

> When you see this in a problem → think this pattern:

| Clue in Problem | Pattern to Use |
|---|---|
| Sorted array + find pair/target | **Two Pointers** |
| Subarray / substring of size k | **Sliding Window** |
| Sorted array + search | **Binary Search** |
| "All combinations / subsets" | **Backtracking** |
| "Optimal substructure" / min-max | **Dynamic Programming** |
| Shortest path in graph | **BFS** |
| Connected components / cycle | **DFS** |
| "Next greater/smaller" element | **Monotonic Stack** |
| Top-K / Kth largest | **Heap (priority_queue)** |
| Duplicate detection / fast lookup | **HashMap / HashSet** |
| Tree problems with subtree info | **DFS post-order** |

---

## Big-O Quick Reference

| Operation | Array | LinkedList | HashMap | BST |
|---|---|---|---|---|
| Access by index | O(1) | O(n) | – | – |
| Search | O(n) | O(n) | O(1) avg | O(log n) |
| Insert (end) | O(1) amort | O(1) | O(1) avg | O(log n) |
| Insert (middle) | O(n) | O(1)* | O(1) avg | O(log n) |
| Delete | O(n) | O(1)* | O(1) avg | O(log n) |

*with pointer to position

---

## 1. Two Pointers

### 🔍 When to Recognize
- Array/string is **sorted** (or can be sorted)
- You need to find a **pair** or **triplet** satisfying a condition
- Keywords: "find pair", "sum equals target", "remove duplicates"

### 📌 Core Idea
Start one pointer at the left, one at the right.  
Move them **inward** based on whether the current value is too big or too small.

```cpp
// ✅ EASY: Check if a sorted array has a pair summing to target
// Input: arr = [1, 2, 4, 6, 8], target = 10
// Output: true  (2 + 8 = 10)

bool hasPairSum(std::vector<int>& arr, int target) {
    int left = 0, right = arr.size() - 1;

    while (left < right) {
        int sum = arr[left] + arr[right];
        if      (sum == target) return true;
        else if (sum < target)  left++;   // Sum too small → move left pointer right
        else                    right--;  // Sum too big  → move right pointer left
    }
    return false;
}
// Time: O(n)  Space: O(1)
```

### 📌 Variation: Remove Duplicates In-Place

```cpp
// ✅ EASY: Remove duplicates from sorted array, return new length
// Input: [1, 1, 2, 3, 3, 4]
// Output: 4  (array becomes [1, 2, 3, 4, ...])

int removeDuplicates(std::vector<int>& nums) {
    if (nums.empty()) return 0;
    int slow = 0;  // 'slow' is the write pointer

    for (int fast = 1; fast < nums.size(); fast++) {
        if (nums[fast] != nums[slow]) {
            slow++;
            nums[slow] = nums[fast];  // Write unique element
        }
        // If same, fast just keeps moving forward
    }
    return slow + 1;
}
// Trick: slow pointer = last written unique. fast pointer = scanner.
```

### 📌 Variation: 3Sum (Medium)

```cpp
// 🔶 MEDIUM: Find all unique triplets that sum to 0
// Input: [-1, 0, 1, 2, -1, -4]
// Output: [[-1, -1, 2], [-1, 0, 1]]

std::vector<std::vector<int>> threeSum(std::vector<int>& nums) {
    std::sort(nums.begin(), nums.end());  // Sort first!
    std::vector<std::vector<int>> result;

    for (int i = 0; i < nums.size() - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;  // Skip duplicates for i

        int left = i + 1, right = nums.size() - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.push_back({nums[i], nums[left], nums[right]});
                while (left < right && nums[left]  == nums[left+1])  left++;  // Skip dupes
                while (left < right && nums[right] == nums[right-1]) right--; // Skip dupes
                left++; right--;
            } else if (sum < 0) left++;
            else                right--;
        }
    }
    return result;
}
// Time: O(n²)  Space: O(1) (excluding output)
```

### 🧠 Key Takeaways
- Always **sort first** if not already sorted
- Use `slow/fast` for in-place write problems
- For 3Sum: fix one pointer with a loop, two-pointer for the rest

---

## 2. Sliding Window

### 🔍 When to Recognize
- **"Contiguous subarray/substring"** with some constraint
- Keywords: "longest", "shortest", "exactly k", "at most k distinct"

### 📌 Core Idea
Maintain a window `[left, right]`. Grow it right, shrink it left when invalid.

```cpp
// ✅ EASY: Maximum sum of k consecutive elements
// Input: arr = [2, 1, 5, 1, 3, 2], k = 3
// Output: 9  (5+1+3)

int maxSumKWindow(std::vector<int>& arr, int k) {
    int windowSum = 0;
    // Build first window
    for (int i = 0; i < k; i++) windowSum += arr[i];

    int maxSum = windowSum;
    for (int i = k; i < arr.size(); i++) {
        windowSum += arr[i];       // New element enters from right
        windowSum -= arr[i - k];  // Old element leaves from left
        maxSum = std::max(maxSum, windowSum);
    }
    return maxSum;
}
// Time: O(n)  Space: O(1)
```

### 📌 Variation: Variable Window — Longest Substring Without Repeating (Medium)

```cpp
// 🔶 MEDIUM: Find length of longest substring without repeating characters
// Input: "abcabcbb"
// Output: 3  ("abc")

int lengthOfLongestSubstring(const std::string& s) {
    std::unordered_map<char, int> lastSeen;  // char → last index seen
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.size(); right++) {
        // If char was seen AND its last position is within our window
        if (lastSeen.count(s[right]) && lastSeen[s[right]] >= left) {
            left = lastSeen[s[right]] + 1;  // Shrink window past the duplicate
        }
        lastSeen[s[right]] = right;
        maxLen = std::max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Window = [left, right], always valid (no repeats inside)
```

### 📌 Variation: Minimum Window Substring (Medium)

```cpp
// 🔶 MEDIUM: Smallest substring of s containing all chars of t
// Input: s = "ADOBECODEBANC", t = "ABC"
// Output: "BANC"

std::string minWindow(const std::string& s, const std::string& t) {
    std::unordered_map<char, int> need, have;
    for (char c : t) need[c]++;

    int left = 0, formed = 0, required = need.size();
    int minLen = INT_MAX, minLeft = 0;

    for (int right = 0; right < s.size(); right++) {
        have[s[right]]++;
        // Check if this char's count satisfies the need
        if (need.count(s[right]) && have[s[right]] == need[s[right]])
            formed++;

        // Try to shrink window from left
        while (formed == required) {
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minLeft = left;
            }
            have[s[left]]--;
            if (need.count(s[left]) && have[s[left]] < need[s[left]])
                formed--;
            left++;
        }
    }
    return minLen == INT_MAX ? "" : s.substr(minLeft, minLen);
}
```

### 🧠 Key Takeaways
- **Fixed window** → compute first, then slide (add right, remove left simultaneously)
- **Variable window** → expand right freely, shrink left when constraint violated
- Use `unordered_map<char,int>` to track frequencies inside the window

---

## 3. Binary Search

### 🔍 When to Recognize
- Array is **sorted** (or "monotonically ordered")
- You're searching for a value, boundary, or doing "search on answer"
- Keywords: "find first/last", "minimum that satisfies", "rotated sorted array"

### 📌 Core Idea
Halve the search space each iteration. Classic off-by-one errors are common — **know your boundary conditions**.

```cpp
// ✅ EASY: Classic binary search
int binarySearch(const std::vector<int>& arr, int target) {
    int lo = 0, hi = arr.size() - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;  // NEVER use (lo+hi)/2 — integer overflow!
        if      (arr[mid] == target) return mid;
        else if (arr[mid] < target)  lo = mid + 1;
        else                         hi = mid - 1;
    }
    return -1;
}
```

### 📌 Template: Find First Position ≥ target (lower_bound)

```cpp
// Returns first index where arr[index] >= target
// (equivalent to std::lower_bound)
int lowerBound(const std::vector<int>& arr, int target) {
    int lo = 0, hi = arr.size();  // Note: hi = size, not size-1

    while (lo < hi) {             // Note: lo < hi, not lo <= hi
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target) lo = mid + 1;
        else                   hi = mid;  // Keep mid as potential answer
    }
    return lo;
}
// When loop ends, lo == hi == first valid position
```

### 📌 Variation: Binary Search on Rotated Sorted Array (Medium)

```cpp
// 🔶 MEDIUM: Search in rotated sorted array [4,5,6,7,0,1,2], target=0 → 4
// Key insight: one half is ALWAYS sorted after rotation

int searchRotated(std::vector<int>& nums, int target) {
    int lo = 0, hi = nums.size() - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;

        // Check which half is sorted
        if (nums[lo] <= nums[mid]) {          // Left half is sorted
            if (nums[lo] <= target && target < nums[mid])
                hi = mid - 1;                 // Target in left half
            else
                lo = mid + 1;                 // Target in right half
        } else {                              // Right half is sorted
            if (nums[mid] < target && target <= nums[hi])
                lo = mid + 1;                 // Target in right half
            else
                hi = mid - 1;                 // Target in left half
        }
    }
    return -1;
}
```

### 🧠 Key Takeaways
- Always use `mid = lo + (hi - lo) / 2` to avoid overflow
- For **"find boundary"** problems: use `lo < hi` and `hi = mid` (not `mid-1`)
- For **"find exact"** problems: use `lo <= hi` and both `lo = mid+1`, `hi = mid-1`

---

## 4. HashMap / HashSet Tricks

### 🔍 When to Recognize
- "Two Sum", frequency count, duplicate detection, anagram check
- Need O(1) lookup to complement or count

### 📌 Core Idea
Store what you've seen; check against what you need.

```cpp
// ✅ EASY: Two Sum — find indices of two numbers that add to target
// Input: nums = [2, 7, 11, 15], target = 9
// Output: [0, 1]  (nums[0] + nums[1] = 9)

std::vector<int> twoSum(std::vector<int>& nums, int target) {
    std::unordered_map<int, int> seen;  // value → index

    for (int i = 0; i < nums.size(); i++) {
        int complement = target - nums[i];
        if (seen.count(complement)) {
            return {seen[complement], i};  // Found the pair!
        }
        seen[nums[i]] = i;  // Record current before moving on
    }
    return {};
}
// Time: O(n)  Space: O(n)
// Trick: store complement, not the sum
```

### 📌 Variation: Valid Anagram

```cpp
// ✅ EASY: Check if two strings are anagrams
// "anagram" and "nagaram" → true

bool isAnagram(const std::string& s, const std::string& t) {
    if (s.size() != t.size()) return false;

    int freq[26] = {};  // For lowercase letters, use fixed array (fast)
    for (char c : s) freq[c - 'a']++;
    for (char c : t) freq[c - 'a']--;

    for (int f : freq) if (f != 0) return false;
    return true;
}
// Time: O(n)  Space: O(1) — fixed alphabet size
```

### 📌 Variation: Group Anagrams (Medium)

```cpp
// 🔶 MEDIUM: Group strings that are anagrams of each other
// Input: ["eat","tea","tan","ate","nat","bat"]
// Output: [["bat"],["nat","tan"],["ate","eat","tea"]]

std::vector<std::vector<std::string>> groupAnagrams(std::vector<std::string>& strs) {
    std::unordered_map<std::string, std::vector<std::string>> groups;

    for (const auto& s : strs) {
        std::string key = s;
        std::sort(key.begin(), key.end());  // Sorted form is the canonical key
        groups[key].push_back(s);
    }

    std::vector<std::vector<std::string>> result;
    for (auto& [key, group] : groups)
        result.push_back(group);
    return result;
}
// Insight: two strings are anagrams iff their sorted forms are equal
```

### 🧠 Key Takeaways
- `unordered_map` for O(1) avg. lookups; `map` for O(log n) sorted access
- For fixed alphabet (a-z), `int freq[26]{}` is faster than a map
- Canonical key pattern: sort the string to group anagrams

---

## 5. Stack

### 🔍 When to Recognize
- **Matching/nesting** problems (parentheses, brackets, tags)
- **"Next greater/smaller"** element queries
- Expression evaluation

### 📌 Core Idea
Stack preserves **ordering** — LIFO lets you remember the last "open" state.

```cpp
// ✅ EASY: Valid Parentheses — check if brackets are balanced
// "()[]{}" → true,  "([)]" → false

bool isValid(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            st.push(c);
        } else {
            if (st.empty()) return false;
            if (c == ')' && st.top() != '(') return false;
            if (c == ']' && st.top() != '[') return false;
            if (c == '}' && st.top() != '{') return false;
            st.pop();
        }
    }
    return st.empty();  // Stack must be empty at the end
}
```

### 📌 Variation: Next Greater Element (Monotonic Stack — Medium)

```cpp
// 🔶 MEDIUM: For each element, find the next element greater than it
// Input: [2, 1, 2, 4, 3]
// Output: [4, 2, 4, -1, -1]  (-1 if none)

// Pattern: Monotonic stack — stack holds candidates waiting for their "next greater"
std::vector<int> nextGreaterElement(const std::vector<int>& nums) {
    int n = nums.size();
    std::vector<int> result(n, -1);
    std::stack<int> stk;  // Stores indices (monotonically decreasing values)

    for (int i = 0; i < n; i++) {
        // Pop all elements smaller than nums[i] — nums[i] is their answer
        while (!stk.empty() && nums[i] > nums[stk.top()]) {
            result[stk.top()] = nums[i];
            stk.pop();
        }
        stk.push(i);
        // Remaining items in stack have no greater element → stay -1
    }
    return result;
}
// Insight: stack always stays decreasing (monotonic)
// When a new element is bigger, it "resolves" all smaller waiting elements
```

### 📌 Variation: Daily Temperatures (Medium)

```cpp
// 🔶 MEDIUM: How many days until a warmer temperature?
// Input:  [73, 74, 75, 71, 69, 72, 76, 73]
// Output: [ 1,  1,  4,  2,  1,  1,  0,  0]

std::vector<int> dailyTemperatures(std::vector<int>& temps) {
    int n = temps.size();
    std::vector<int> result(n, 0);
    std::stack<int> stk;  // Stores indices

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && temps[i] > temps[stk.top()]) {
            int idx = stk.top(); stk.pop();
            result[idx] = i - idx;  // Distance = current index - waiting index
        }
        stk.push(i);
    }
    return result;
}
// Same monotonic stack pattern as Next Greater — just store the distance!
```

### 🧠 Key Takeaways
- **Monotonic stack** = stack that maintains a sorted order (increasing or decreasing)
- Store **indices** in the stack, not values (you often need the index for distance/answer)
- When you see "next greater/smaller" → use monotonic stack, O(n) solution

---

## 6. Linked List

### 🔍 When to Recognize
- "Reverse", "detect cycle", "find middle", "merge two sorted lists"
- The **slow/fast pointer** trick is the KEY insight for most linked list problems

### 📌 Core Operations

```cpp
struct Node { int val; Node* next; Node(int v) : val(v), next(nullptr) {} };

// ✅ EASY: Reverse a linked list
// 1 → 2 → 3 → null  becomes  3 → 2 → 1 → null

Node* reverseList(Node* head) {
    Node* prev = nullptr;
    Node* curr = head;

    while (curr) {
        Node* nextNode = curr->next;  // 1. Save next
        curr->next = prev;            // 2. Reverse the link
        prev = curr;                  // 3. Move prev forward
        curr = nextNode;              // 4. Move curr forward
    }
    return prev;  // prev is the new head
}

// ✅ EASY: Find middle of linked list (slow/fast pointer)
Node* findMiddle(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;        // Move 1 step
        fast = fast->next->next;  // Move 2 steps
    }
    return slow;  // When fast hits end, slow is at middle
}

// ✅ EASY: Detect cycle (Floyd's Tortoise & Hare)
bool hasCycle(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;  // They meet → cycle exists
    }
    return false;
}
```

### 📌 Variation: Merge Two Sorted Lists (Easy-Medium)

```cpp
// ✅ EASY: Merge [1→2→4] and [1→3→4] → [1→1→2→3→4→4]

Node* mergeTwoLists(Node* l1, Node* l2) {
    Node dummy(0);   // Dummy head simplifies edge cases
    Node* curr = &dummy;

    while (l1 && l2) {
        if (l1->val <= l2->val) {
            curr->next = l1;
            l1 = l1->next;
        } else {
            curr->next = l2;
            l2 = l2->next;
        }
        curr = curr->next;
    }
    curr->next = l1 ? l1 : l2;  // Attach remaining
    return dummy.next;
}
// Trick: always use a dummy head node to avoid special-casing the first element
```

### 🧠 Key Takeaways
- **Slow/fast pointer**: fast moves 2x → when fast hits end, slow is at middle
- **Reverse**: 3 pointer dance — prev, curr, next
- **Dummy head**: always use when building new list or merging

---

## 7. Tree (DFS / BFS)

### 🔍 When to Recognize
- Tree traversal (inorder = sorted for BST)
- **DFS** for subtree info (height, path sums, LCA)
- **BFS / Level order** for level-by-level processing

```cpp
struct TreeNode {
    int val; TreeNode* left; TreeNode* right;
    TreeNode(int v) : val(v), left(nullptr), right(nullptr) {}
};
```

### 📌 Core: DFS on Trees

```cpp
// ✅ EASY: Maximum depth of binary tree
// Pattern: post-order DFS — compute children first, then merge results

int maxDepth(TreeNode* root) {
    if (!root) return 0;                                        // Base case
    int leftDepth  = maxDepth(root->left);                     // Left subtree
    int rightDepth = maxDepth(root->right);                    // Right subtree
    return 1 + std::max(leftDepth, rightDepth);                // Combine
}

// ✅ EASY: Symmetric tree (is it a mirror of itself?)
bool isMirror(TreeNode* left, TreeNode* right) {
    if (!left && !right) return true;               // Both null → symmetric
    if (!left || !right) return false;              // One null → not symmetric
    return (left->val == right->val)
        && isMirror(left->left, right->right)       // Outer pair
        && isMirror(left->right, right->left);      // Inner pair
}
bool isSymmetric(TreeNode* root) {
    return !root || isMirror(root->left, root->right);
}
```

### 📌 Variation: Path Sum (Check if root-to-leaf path sums to target)

```cpp
// ✅ EASY: Does any root-to-leaf path sum to targetSum?
// Idea: subtract node value, check if remaining == 0 at leaf

bool hasPathSum(TreeNode* root, int remainingSum) {
    if (!root) return false;
    remainingSum -= root->val;

    // Leaf node: check if we've used exactly the required sum
    if (!root->left && !root->right) return remainingSum == 0;

    return hasPathSum(root->left, remainingSum)
        || hasPathSum(root->right, remainingSum);
}
```

### 📌 Variation: Level Order Traversal (BFS — Easy)

```cpp
// ✅ EASY: Return nodes level by level
// Output: [[3], [9,20], [15,7]]

std::vector<std::vector<int>> levelOrder(TreeNode* root) {
    std::vector<std::vector<int>> result;
    if (!root) return result;

    std::queue<TreeNode*> q;
    q.push(root);

    while (!q.empty()) {
        int levelSize = q.size();   // Snapshot: how many nodes in this level
        std::vector<int> level;

        while (levelSize--) {
            auto node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(level);
    }
    return result;
}
// Key trick: snapshot q.size() BEFORE the inner loop
```

### 📌 Variation: Lowest Common Ancestor (Medium)

```cpp
// 🔶 MEDIUM: Find lowest common ancestor of two nodes p and q in BST
// BST property makes it easy: if both < root → go left, both > root → go right

TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    while (root) {
        if (p->val < root->val && q->val < root->val)
            root = root->left;   // Both in left subtree
        else if (p->val > root->val && q->val > root->val)
            root = root->right;  // Both in right subtree
        else
            return root;         // Split here → this IS the LCA
    }
    return nullptr;
}

// For a GENERAL binary tree (not BST):
TreeNode* lcaGeneral(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;  // Found one of them

    TreeNode* left  = lcaGeneral(root->left, p, q);
    TreeNode* right = lcaGeneral(root->right, p, q);

    if (left && right) return root;   // p and q are in different subtrees → root is LCA
    return left ? left : right;       // Both in same subtree
}
```

### 🧠 Key Takeaways
- **DFS (recursion)** naturally handles subtree problems via return values
- **Post-order**: compute children first (`if (!root) return base`), then combine
- **BFS on tree**: use `q.size()` snapshot to process one level at a time
- BST LCA: just follow the BST rules without recursion overhead

---

## 8. Graph (BFS / DFS)

### 🔍 When to Recognize
- "Islands", "connected components", "shortest path", "detect cycle"
- Grid problems are implicitly graphs (each cell = node, neighbors = edges)

### 📌 Core: BFS (Shortest Path)

```cpp
// ✅ EASY-MEDIUM: Number of islands
// Count connected groups of '1's in a grid

int numIslands(std::vector<std::vector<char>>& grid) {
    int rows = grid.size(), cols = grid[0].size();
    int count = 0;

    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            if (grid[r][c] == '1') {
                count++;
                // DFS to "sink" the entire island (mark as visited)
                std::function<void(int,int)> dfs = [&](int row, int col) {
                    if (row < 0 || row >= rows || col < 0 || col >= cols) return;
                    if (grid[row][col] != '1') return;
                    grid[row][col] = '0';  // Mark visited
                    dfs(row+1, col); dfs(row-1, col);
                    dfs(row, col+1); dfs(row, col-1);
                };
                dfs(r, c);
            }
        }
    }
    return count;
}
// Classic flood-fill DFS. Modify grid in-place to mark visited.
```

### 📌 Variation: BFS Shortest Path in Grid (Medium)

```cpp
// 🔶 MEDIUM: Shortest path from top-left to bottom-right (0=open, 1=blocked)
// Input: [[0,0,0],[1,1,0],[1,1,0]]
// Output: 4  (steps)

int shortestPath(std::vector<std::vector<int>>& grid) {
    int n = grid.size(), m = grid[0].size();
    if (grid[0][0] == 1 || grid[n-1][m-1] == 1) return -1;

    std::queue<std::tuple<int,int,int>> q;  // {row, col, distance}
    q.push({0, 0, 1});
    grid[0][0] = 1;  // Mark visited by changing value

    int dirs[][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!q.empty()) {
        auto [r, c, dist] = q.front(); q.pop();
        if (r == n-1 && c == m-1) return dist;

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr >= 0 && nr < n && nc >= 0 && nc < m && grid[nr][nc] == 0) {
                grid[nr][nc] = 1;  // Mark visited
                q.push({nr, nc, dist + 1});
            }
        }
    }
    return -1;
}
// BFS guarantees shortest path in unweighted graphs
```

### 🧠 Key Takeaways
- **DFS** for connected components, exhaustive exploration
- **BFS** for shortest path (unweighted) — it fans out level by level
- Mark visited **before** enqueuing (not after) to avoid duplicate processing
- Grid → define a `dirs` array for 4-directional movement

---

## 9. Dynamic Programming

### 🔍 When to Recognize
- "Minimum/maximum" of something over choices
- "How many ways" to do something
- Problem has **overlapping subproblems** (same computation repeated)
- Keywords: "optimal", "count paths", "minimum cost", "longest"

### 📌 Framework: 3 Steps to DP
1. **Define**: What does `dp[i]` mean?
2. **Base case**: What is `dp[0]`?
3. **Transition**: How does `dp[i]` relate to smaller states?

```cpp
// ✅ EASY: Climbing Stairs — n steps, can climb 1 or 2 steps at a time
// How many distinct ways to reach step n?
// Input: n = 4  Output: 5

int climbStairs(int n) {
    if (n <= 1) return 1;

    // dp[i] = number of ways to reach step i
    // dp[i] = dp[i-1] (came from step i-1) + dp[i-2] (came from step i-2)
    int prev2 = 1, prev1 = 1;  // dp[0]=1, dp[1]=1
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}
// Pattern: this IS Fibonacci. Recognize it!
```

```cpp
// ✅ EASY: Coin Change — min coins to make amount
// Input: coins = [1, 5, 11], amount = 15
// Output: 3  (5+5+5)  not (11+1+1+1+1=5 coins)

int coinChange(std::vector<int>& coins, int amount) {
    // dp[i] = minimum coins to make amount i
    std::vector<int> dp(amount + 1, amount + 1);  // Init with impossible value
    dp[0] = 0;  // Base: 0 coins to make 0

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = std::min(dp[i], dp[i - coin] + 1);
                //                         └── +1 for using this coin
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
// Time: O(amount × coins)  Space: O(amount)
```

### 📌 Variation: Longest Common Subsequence (Medium)

```cpp
// 🔶 MEDIUM: LCS of two strings (not substrings — can skip chars)
// Input: "abcde", "ace"  Output: 3  ("ace")

int longestCommonSubsequence(const std::string& s1, const std::string& s2) {
    int m = s1.size(), n = s2.size();
    // dp[i][j] = LCS of s1[0..i-1] and s2[0..j-1]
    std::vector<std::vector<int>> dp(m+1, std::vector<int>(n+1, 0));

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1[i-1] == s2[j-1])
                dp[i][j] = dp[i-1][j-1] + 1;            // Chars match: extend
            else
                dp[i][j] = std::max(dp[i-1][j], dp[i][j-1]); // Take best of skipping either
        }
    }
    return dp[m][n];
}
```

### 🧠 Key Takeaways
- Start with a **clear definition** of what `dp[i]` means in English
- Fill base cases first, then build up
- When you have 2 strings → 2D DP table (`dp[i][j]`)
- Space optimization: often can reduce 2D → 1D array

---

## 10. Heap / Priority Queue

### 🔍 When to Recognize
- "Top K", "Kth largest/smallest"
- "Merge K sorted lists/arrays"
- Keywords: "most frequent", "closest", "median"

### 📌 Core Idea
C++ `priority_queue` is a max-heap by default.  
Use `priority_queue<int, vector<int>, greater<int>>` for min-heap.

```cpp
// ✅ EASY: Find Kth Largest Element
// Input: nums = [3, 2, 1, 5, 6, 4], k = 2
// Output: 5

int findKthLargest(std::vector<int>& nums, int k) {
    // Min-heap of size k — the top is the Kth largest
    std::priority_queue<int, std::vector<int>, std::greater<int>> minHeap;

    for (int num : nums) {
        minHeap.push(num);
        if (minHeap.size() > k)
            minHeap.pop();  // Remove smallest — keep only top k
    }
    return minHeap.top();  // Smallest of top-k = Kth largest
}
// Time: O(n log k)  Space: O(k)
// Insight: min-heap of size k → top element is always the Kth largest
```

### 📌 Variation: Top K Frequent Elements (Medium)

```cpp
// 🔶 MEDIUM: Return the k most frequent elements
// Input: nums = [1,1,1,2,2,3], k = 2  Output: [1,2]

std::vector<int> topKFrequent(std::vector<int>& nums, int k) {
    // Step 1: Count frequencies
    std::unordered_map<int, int> freq;
    for (int n : nums) freq[n]++;

    // Step 2: Min-heap by frequency, size k
    // pair: {frequency, value}
    std::priority_queue<std::pair<int,int>,
                        std::vector<std::pair<int,int>>,
                        std::greater<>> minHeap;

    for (auto& [val, cnt] : freq) {
        minHeap.push({cnt, val});
        if (minHeap.size() > k) minHeap.pop();  // Drop least frequent
    }

    // Step 3: Extract results
    std::vector<int> result;
    while (!minHeap.empty()) {
        result.push_back(minHeap.top().second);
        minHeap.pop();
    }
    return result;
}
```

### 🧠 Key Takeaways
- **Max-heap**: `priority_queue<int>` — top() = largest
- **Min-heap**: `priority_queue<int, vector<int>, greater<int>>` — top() = smallest
- **Top-K largest** → use min-heap of size K (counter-intuitive but correct)
- **Top-K smallest** → use max-heap of size K

---

## 11. Backtracking

### 🔍 When to Recognize
- "Generate ALL combinations / subsets / permutations"
- Constraint satisfaction (Sudoku, N-Queens)
- Keywords: "all possible", "find all ways", "enumerate"

### 📌 Template

```
backtrack(state, choices):
    if state is complete:
        add to results
        return
    for each choice:
        make choice
        backtrack(state + choice, remaining choices)
        undo choice   ← THE BACKTRACK STEP
```

```cpp
// ✅ EASY-MEDIUM: Generate all subsets of [1,2,3]
// Output: [[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]

std::vector<std::vector<int>> subsets(std::vector<int>& nums) {
    std::vector<std::vector<int>> result;
    std::vector<int> current;

    std::function<void(int)> backtrack = [&](int start) {
        result.push_back(current);  // Every state is a valid subset

        for (int i = start; i < nums.size(); i++) {
            current.push_back(nums[i]);   // Choose
            backtrack(i + 1);             // Explore
            current.pop_back();           // Unchoose (backtrack)
        }
    };
    backtrack(0);
    return result;
}
```

### 📌 Variation: Combination Sum (Medium)

```cpp
// 🔶 MEDIUM: Find all combinations that sum to target (can reuse elements)
// Input: candidates=[2,3,6,7], target=7  Output:[[2,2,3],[7]]

std::vector<std::vector<int>> combinationSum(std::vector<int>& candidates, int target) {
    std::vector<std::vector<int>> result;
    std::vector<int> current;

    std::function<void(int, int)> backtrack = [&](int start, int remaining) {
        if (remaining == 0) { result.push_back(current); return; }
        if (remaining < 0) return;  // Pruning: exceeded target, stop branch

        for (int i = start; i < candidates.size(); i++) {
            current.push_back(candidates[i]);
            backtrack(i, remaining - candidates[i]);  // i not i+1, since reuse is allowed
            current.pop_back();
        }
    };
    backtrack(0, target);
    return result;
}
```

### 🧠 Key Takeaways
- Always **undo** changes after recursion (pop_back, restore state)
- `start` index prevents re-using previous elements (pass `i+1` for one-use, `i` for reuse)
- **Pruning** early (`if remaining < 0 return`) cuts branches → huge speedup

---

## 12. Binary Search On Answer

### 🔍 When to Recognize
- "Minimum/maximum value that satisfies a condition"
- The answer space is a **range of integers**
- You can write a function `isValid(x)` that is monotonic (once false, always false)

```cpp
// 🔶 MEDIUM: Koko's Eating Speed
// Koko has piles=[3,6,7,11], hours=8. What minimum eating speed finishes in time?
// Answer: speed=4 (reads 1+2+2+3 = 8 hours)

// Check: can koko finish at speed 'speed' within 'hours'?
bool canFinish(const std::vector<int>& piles, int speed, int hours) {
    int totalHours = 0;
    for (int pile : piles)
        totalHours += (pile + speed - 1) / speed;  // Ceiling division
    return totalHours <= hours;
}

int minEatingSpeed(std::vector<int>& piles, int H) {
    int lo = 1;                                              // Min possible speed
    int hi = *std::max_element(piles.begin(), piles.end()); // Max possible speed

    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (canFinish(piles, mid, H))
            hi = mid;       // This speed works, try lower
        else
            lo = mid + 1;   // Too slow, try higher
    }
    return lo;
}
// Binary search on the ANSWER SPACE, not on an array
// Time: O(n log max_pile)
```

---

## 13. Prefix Sum

### 🔍 When to Recognize
- "Sum of a subarray/range" queried multiple times
- "Number of subarrays with sum = k"

```cpp
// ✅ EASY: Subarray sum equals k
// Input: nums=[1,1,1], k=2  Output: 2  ([1,1] appears twice)

int subarraySum(std::vector<int>& nums, int k) {
    // prefix[i] = sum of nums[0..i-1]
    // subarray[i..j] sum = prefix[j+1] - prefix[i]
    // We want prefix[j+1] - prefix[i] == k
    // → prefix[i] == prefix[j+1] - k

    std::unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;  // Empty prefix (sum=0) exists once

    int runningSum = 0, count = 0;
    for (int num : nums) {
        runningSum += num;
        int needed = runningSum - k;
        if (prefixCount.count(needed))
            count += prefixCount[needed];  // Found matching prefix
        prefixCount[runningSum]++;
    }
    return count;
}
// Time: O(n)  Space: O(n)
// Insight: prefix sum + hash map = O(n) subarray sum queries
```

---

## 🎯 Interview Problem-Solving Checklist

When you get a problem:

1. **Read carefully** — note constraints (sorted? distinct? n size?)
2. **Identify the pattern** using the Pattern Map at the top
3. **Start with brute force** — describe it verbally (O(n²) is fine to mention)
4. **Optimize** — which pattern can reduce it?
5. **Trace through an example** before coding
6. **Write clean code** — name variables clearly, add comments
7. **State complexity** — always say Time AND Space

### Common Complexity Goals
| Input Size | Target Complexity |
|---|---|
| n ≤ 10 | O(n!) or O(2^n) |
| n ≤ 20 | O(2^n) |
| n ≤ 100 | O(n³) |
| n ≤ 1000 | O(n²) |
| n ≤ 10^6 | O(n) or O(n log n) |

---

## 🗂️ Sorting Quick Reference

```cpp
// std::sort — O(n log n), not stable
std::sort(v.begin(), v.end());                              // Ascending
std::sort(v.begin(), v.end(), std::greater<int>());         // Descending
std::sort(v.begin(), v.end(), [](int a, int b) {           // Custom comparator
    return a % 10 < b % 10;  // Sort by last digit
});

// Merge Sort — O(n log n), STABLE (preserves equal element order)
void merge(std::vector<int>& arr, int lo, int mid, int hi) {
    std::vector<int> tmp(hi - lo + 1);
    int i = lo, j = mid + 1, k = 0;
    while (i <= mid && j <= hi)
        tmp[k++] = arr[i] <= arr[j] ? arr[i++] : arr[j++];
    while (i <= mid) tmp[k++] = arr[i++];
    while (j <= hi)  tmp[k++] = arr[j++];
    for (int x = 0; x < k; x++) arr[lo + x] = tmp[x];
}

// Quick Sort — O(n log n) avg, O(n²) worst, in-place
int partition(std::vector<int>& arr, int lo, int hi) {
    int pivot = arr[hi], i = lo - 1;
    for (int j = lo; j < hi; j++)
        if (arr[j] <= pivot) std::swap(arr[++i], arr[j]);
    std::swap(arr[i + 1], arr[hi]);
    return i + 1;
}
```

---

## 📊 STL Containers Cheat Sheet

```cpp
// Vector — dynamic array
std::vector<int> v = {1,2,3};
v.push_back(4);          // O(1) amortized
v.pop_back();            // O(1)
v[i];                    // O(1) random access

// Deque — double-ended queue
std::deque<int> dq;
dq.push_front(1);        // O(1) both ends
dq.push_back(2);

// Stack (LIFO)
std::stack<int> st;
st.push(1); st.top(); st.pop();

// Queue (FIFO)
std::queue<int> q;
q.push(1); q.front(); q.pop();

// Priority Queue (max-heap by default)
std::priority_queue<int> maxH;                                        // Max-heap
std::priority_queue<int, std::vector<int>, std::greater<int>> minH;  // Min-heap

// Unordered Map — O(1) avg
std::unordered_map<int, int> um;
um[key] = val;
um.count(key);           // 1 if exists, 0 if not (use instead of find() for check)
um.find(key);            // Iterator

// Set — sorted unique elements O(log n)
std::set<int> s;
std::unordered_set<int> us;  // O(1) avg, unordered
```
