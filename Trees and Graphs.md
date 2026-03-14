# Trees and Graphs

## Binary Tree

- A tree where each node has **at most 2 children**: `left` and `right`.
- No ordering constraint — unlike BST, children can be anything.
- **Root**: topmost node. **Leaf**: node with no children.
- **Height**: longest path from root to a leaf. **Depth**: distance from root to a node.

```cpp
struct Node {
    int data;
    Node* left  = nullptr;
    Node* right = nullptr;
    Node(int val) : data(val) {}
};
```

### Key Properties

| Property | Formula |
|---|---|
| Max nodes at level `l` | `2^l` |
| Max nodes in tree of height `h` | `2^(h+1) - 1` |
| Min height for `n` nodes | `floor(log2(n))` |

---

## Binary Search Tree (BST)

- A Binary Tree with ordering: `left->data < node->data < right->data` for every node.
- **Average** search/insert/delete: `O(log n)`. **Worst case** (degenerate/skewed): `O(n)`.

```cpp
struct BSTNode {
    int data;
    BSTNode* left  = nullptr;
    BSTNode* right = nullptr;
    BSTNode(int val) : data(val) {}
};
```

---

## Adding and Removing Elements in a Tree

### Insert into BST

- Recursively find the correct position by comparing values.

```cpp
BSTNode* insert(BSTNode* root, int val) {
    if (!root) return new BSTNode(val);  // Found the empty spot — insert here
    if (val < root->data)
        root->left  = insert(root->left,  val);  // Go left if smaller
    else if (val > root->data)
        root->right = insert(root->right, val);  // Go right if larger
    // val == root->data: duplicate; ignore or handle
    return root;
}
```

### Insert into Generic Binary Tree (level-order)

- Insert into first available position using BFS.

```cpp
#include <queue>
void insertBT(Node* root, int val) {
    Node* newNode = new Node(val);
    if (!root) { root = newNode; return; }
    std::queue<Node*> q;
    q.push(root);
    while (!q.empty()) {
        Node* curr = q.front(); q.pop();
        if (!curr->left)  { curr->left  = newNode; return; }  // Insert at first gap
        else               q.push(curr->left);
        if (!curr->right) { curr->right = newNode; return; }
        else               q.push(curr->right);
    }
}
```

### Delete from BST

Three cases:
1. **Leaf node** — just delete.
2. **One child** — replace node with its child.
3. **Two children** — replace with **in-order successor** (smallest in right subtree), then delete successor.

```cpp
BSTNode* findMin(BSTNode* node) {
    while (node->left) node = node->left;  // Leftmost = smallest
    return node;
}

BSTNode* deleteBST(BSTNode* root, int val) {
    if (!root) return nullptr;

    if (val < root->data)
        root->left  = deleteBST(root->left,  val);
    else if (val > root->data)
        root->right = deleteBST(root->right, val);
    else {
        // Case 1 & 2: 0 or 1 child
        if (!root->left)  { BSTNode* tmp = root->right; delete root; return tmp; }
        if (!root->right) { BSTNode* tmp = root->left;  delete root; return tmp; }
        // Case 3: 2 children — find in-order successor
        BSTNode* successor = findMin(root->right);
        root->data  = successor->data;               // Copy successor value up
        root->right = deleteBST(root->right, successor->data); // Delete successor
    }
    return root;
}
```

### Delete from Generic Binary Tree

- Replace target node's data with deepest-rightmost node's data, then delete deepest-rightmost node.

```cpp
void deleteBT(Node* root, int key) {
    if (!root) return;
    std::queue<Node*> q;
    q.push(root);
    Node* targetNode  = nullptr;
    Node* lastNode    = nullptr;
    Node* lastParent  = nullptr;
    bool  lastIsLeft  = false;

    while (!q.empty()) {
        Node* curr = q.front(); q.pop();
        if (curr->data == key) targetNode = curr;   // Mark node to delete
        if (curr->left)  { lastParent = curr; lastIsLeft = true;  lastNode = curr->left;  q.push(curr->left); }
        if (curr->right) { lastParent = curr; lastIsLeft = false; lastNode = curr->right; q.push(curr->right); }
    }
    if (targetNode) targetNode->data = lastNode->data; // Replace with deepest node's value
    if (lastIsLeft) lastParent->left  = nullptr;
    else            lastParent->right = nullptr;
    delete lastNode;
}
```

---

## Search in BST and Binary Tree

### Search in BST — `O(log n)` average

```cpp
BSTNode* searchBST(BSTNode* root, int val) {
    if (!root || root->data == val) return root;  // Base: not found or found
    if (val < root->data) return searchBST(root->left,  val);  // Must be in left subtree
    else                  return searchBST(root->right, val);  // Must be in right subtree
}
```

### Search in Generic Binary Tree — `O(n)`

```cpp
Node* searchBT(Node* root, int val) {
    if (!root) return nullptr;
    if (root->data == val) return root;
    Node* found = searchBT(root->left, val);
    if (found) return found;
    return searchBT(root->right, val);   // Try right subtree if not in left
}
```

---

## Tree Traversals

### In-Order (Left → Root → Right)

- Visits BST nodes in **sorted ascending** order.

```cpp
void inOrder(Node* root) {
    if (!root) return;
    inOrder(root->left);
    std::cout << root->data << " ";  // Visit after left subtree
    inOrder(root->right);
}
// BST example: tree {4,2,6,1,3,5,7} → 1 2 3 4 5 6 7
```

### Pre-Order (Root → Left → Right)

- Root visited **first** — useful for copying/serializing the tree structure.

```cpp
void preOrder(Node* root) {
    if (!root) return;
    std::cout << root->data << " ";  // Visit before children
    preOrder(root->left);
    preOrder(root->right);
}
// Output: 4 2 1 3 6 5 7
```

### Post-Order (Left → Right → Root)

- Root visited **last** — useful for deleting the tree (children deleted before parent).

```cpp
void postOrder(Node* root) {
    if (!root) return;
    postOrder(root->left);
    postOrder(root->right);
    std::cout << root->data << " ";  // Visit after both children
}
// Output: 1 3 2 5 7 6 4
```

### Level-Order (BFS)

- Visit nodes **level by level**, left to right.

```cpp
#include <queue>
void levelOrder(Node* root) {
    if (!root) return;
    std::queue<Node*> q;
    q.push(root);
    while (!q.empty()) {
        int levelSize = q.size();           // Number of nodes at current level
        for (int i = 0; i < levelSize; ++i) {
            Node* curr = q.front(); q.pop();
            std::cout << curr->data << " ";
            if (curr->left)  q.push(curr->left);
            if (curr->right) q.push(curr->right);
        }
        std::cout << "\n";  // Newline between levels
    }
}
// Output: 4 / 2 6 / 1 3 5 7
```

---

## BFS, DFS, and Dijkstra

### BFS (Breadth-First Search)

- Explores all neighbours at the present depth before moving deeper.
- Uses a **queue**. Guarantees shortest path in **unweighted** graphs.
- `O(V + E)` time.

```cpp
#include <queue>
#include <unordered_map>
#include <vector>

void bfs(int start, const std::unordered_map<int, std::vector<int>>& adj) {
    std::unordered_map<int, bool> visited;
    std::queue<int> q;
    q.push(start);
    visited[start] = true;

    while (!q.empty()) {
        int node = q.front(); q.pop();
        std::cout << node << " ";
        for (int neighbour : adj.at(node)) {
            if (!visited[neighbour]) {
                visited[neighbour] = true;
                q.push(neighbour);  // Enqueue unvisited neighbours
            }
        }
    }
}
// adj: {0:[1,2], 1:[3], 2:[3,4], 3:[], 4:[]}
// BFS from 0: 0 1 2 3 4
```

### DFS (Depth-First Search)

- Explores as **deep as possible** along each branch before backtracking.
- Uses a **stack** (or recursion). `O(V + E)` time.

```cpp
void dfs(int node, const std::unordered_map<int, std::vector<int>>& adj,
         std::unordered_map<int, bool>& visited) {
    visited[node] = true;
    std::cout << node << " ";
    for (int neighbour : adj.at(node)) {
        if (!visited[neighbour])
            dfs(neighbour, adj, visited);   // Recurse deeper before backtracking
    }
}

// Call: dfs(0, adj, visited);
// DFS from 0: 0 1 3 2 3 4  (order depends on adjacency list)
```

### DFS on Binary Tree

```cpp
// DFS on tree = pre-order traversal
void dfsBT(Node* root) {
    if (!root) return;
    std::cout << root->data << " ";  // Process node (pre-order)
    dfsBT(root->left);
    dfsBT(root->right);
}
```

### Dijkstra's Algorithm

- Shortest path from a source to **all nodes** in a **weighted, non-negative** graph.
- Uses a **min-heap (priority queue)**. `O((V + E) log V)` time.
- Does **not** work with negative weights (use Bellman-Ford instead).

```cpp
#include <queue>
#include <vector>
#include <limits>

using pii = std::pair<int, int>;  // {distance, node}

std::vector<int> dijkstra(int src, int V,
    const std::vector<std::vector<pii>>& adj)  // adj[u] = {weight, v}
{
    std::vector<int> dist(V, std::numeric_limits<int>::max());
    std::priority_queue<pii, std::vector<pii>, std::greater<pii>> minHeap;

    dist[src] = 0;
    minHeap.push({0, src});  // {distance, node}

    while (!minHeap.empty()) {
        auto [d, u] = minHeap.top(); minHeap.pop();
        if (d > dist[u]) continue;  // Skip stale entries (node already relaxed)

        for (auto [w, v] : adj[u]) {
            if (dist[u] + w < dist[v]) {       // Relaxation step
                dist[v] = dist[u] + w;
                minHeap.push({dist[v], v});
            }
        }
    }
    return dist;  // dist[i] = shortest distance from src to node i
}
```

| Algorithm | Data Structure | Works on | Shortest Path? |
|---|---|---|---|
| BFS | Queue | Unweighted graph | Yes (unweighted) |
| DFS | Stack / Recursion | Any graph/tree | No |
| Dijkstra | Min-Heap | Weighted (≥0) graph | Yes (weighted) |

---

## Is-Subtree Algorithm

### Subtree Check for Binary Tree

- Check if tree `S` is a subtree of tree `T`.
- Strategy: for each node in `T`, check if the subtree rooted there is **identical** to `S`.
- Time: `O(m * n)` where `m` = size of T, `n` = size of S.

```cpp
bool isSameTree(Node* t, Node* s) {
    if (!t && !s) return true;             // Both null — identical
    if (!t || !s) return false;            // One null — not identical
    if (t->data != s->data) return false;  // Data mismatch
    return isSameTree(t->left, s->left) && isSameTree(t->right, s->right);
}

bool isSubtree(Node* T, Node* S) {
    if (!S) return true;   // Empty tree is always a subtree
    if (!T) return false;  // Non-empty S can't be subtree of empty T
    if (isSameTree(T, S)) return true;         // Check if T itself matches S
    return isSubtree(T->left, S) || isSubtree(T->right, S);  // Check children
}
```

### Subtree Check for Undirected Unweighted Tree (Graph)

- Given an undirected tree (adjacency list), check if the subgraph rooted at `subRoot` is a subtree of the tree rooted at `root`.
- Pick a root, perform DFS, and compare structure using recursive matching.

```cpp
// Represents an undirected unweighted tree as adjacency list
using Graph = std::unordered_map<int, std::vector<int>>;

// Builds the subtree structure as a set of sorted child lists for comparison
bool matchSubtree(int u, int parentU, int v, int parentV,
                  const Graph& T, const Graph& S) {
    const auto& childrenT = T.at(u);
    const auto& childrenS = S.at(v);

    // Count unvisited (real) children
    std::vector<int> tKids, sKids;
    for (int c : childrenT) if (c != parentU) tKids.push_back(c);
    for (int c : childrenS) if (c != parentV) sKids.push_back(c);

    if (tKids.size() != sKids.size()) return false;  // Different number of children

    // NOTE: for a general unordered tree, use sorted/canonical form comparison
    for (size_t i = 0; i < tKids.size(); ++i)
        if (!matchSubtree(tKids[i], u, sKids[i], v, T, S)) return false;
    return true;
}

bool isSubtreeInGraph(int startT, int parentT, int subRoot,
                      const Graph& T, const Graph& S) {
    if (matchSubtree(startT, parentT, subRoot, -1, T, S)) return true;
    for (int child : T.at(startT)) {
        if (child == parentT) continue;
        if (isSubtreeInGraph(child, startT, subRoot, T, S)) return true;
    }
    return false;
}
```

---

## Quick Reference — Complexity

| Operation | BST (avg) | BST (worst) | Generic BT |
|---|---|---|---|
| Search | `O(log n)` | `O(n)` | `O(n)` |
| Insert | `O(log n)` | `O(n)` | `O(n)` (BFS) |
| Delete | `O(log n)` | `O(n)` | `O(n)` |
| Traversal | `O(n)` | `O(n)` | `O(n)` |
