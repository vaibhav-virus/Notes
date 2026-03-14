# Linked Lists

## Singly Linked List

- Each node stores data and a pointer to the **next** node.
- No backward traversal. Tail's `next` is `nullptr`.
- `O(1)` insert/delete at head. `O(n)` at arbitrary position (must traverse).

```cpp
struct Node {
    int data;
    Node* next = nullptr;
    Node(int val) : data(val) {}
};
```

### Add Element — Singly Linked List

#### Insert at Head — `O(1)`

```cpp
Node* insertAtHead(Node* head, int val) {
    Node* newNode = new Node(val);
    newNode->next = head;  // New node points to old head
    return newNode;        // Return new head
}
```

#### Insert at Tail — `O(n)`

```cpp
Node* insertAtTail(Node* head, int val) {
    Node* newNode = new Node(val);
    if (!head) return newNode;  // Empty list — new node becomes head
    Node* curr = head;
    while (curr->next) curr = curr->next;  // Walk to last node
    curr->next = newNode;
    return head;
}
```

#### Insert at Position (0-indexed) — `O(n)`

```cpp
Node* insertAt(Node* head, int val, int pos) {
    Node* newNode = new Node(val);
    if (pos == 0) { newNode->next = head; return newNode; }    // Head insertion
    Node* curr = head;
    for (int i = 0; i < pos - 1 && curr; ++i) curr = curr->next;  // Walk to (pos-1)th node
    if (!curr) return head;          // Position out of range
    newNode->next = curr->next;
    curr->next    = newNode;
    return head;
}
```

### Remove Element — Singly Linked List

#### Remove at Head — `O(1)`

```cpp
Node* removeHead(Node* head) {
    if (!head) return nullptr;
    Node* tmp = head;
    head = head->next;  // Move head forward
    delete tmp;
    return head;
}
```

#### Remove by Value — `O(n)`

```cpp
Node* removeByValue(Node* head, int val) {
    if (!head) return nullptr;
    if (head->data == val) {             // Target is head
        Node* tmp = head;
        head = head->next;
        delete tmp;
        return head;
    }
    Node* curr = head;
    while (curr->next && curr->next->data != val)
        curr = curr->next;              // Walk until node before target
    if (curr->next) {
        Node* tmp  = curr->next;
        curr->next = tmp->next;         // Bypass the target node
        delete tmp;
    }
    return head;
}
```

### Search — Singly Linked List — `O(n)`

```cpp
Node* search(Node* head, int val) {
    while (head) {
        if (head->data == val) return head;  // Found
        head = head->next;
    }
    return nullptr;  // Not found
}
```

---

## Doubly Linked List

- Each node stores data, a pointer to **next** and a pointer to **prev**.
- Allows **bidirectional traversal**.
- `O(1)` insert/delete if you already have a pointer to the node.

```cpp
struct DNode {
    int data;
    DNode* next = nullptr;
    DNode* prev = nullptr;
    DNode(int val) : data(val) {}
};
```

### Add Element — Doubly Linked List

#### Insert at Head — `O(1)`

```cpp
DNode* insertAtHead(DNode* head, int val) {
    DNode* newNode = new DNode(val);
    newNode->next = head;
    if (head) head->prev = newNode;  // Old head's prev now points back to new node
    return newNode;
}
```

#### Insert at Tail — `O(n)` (O(1) with tail pointer)

```cpp
DNode* insertAtTail(DNode* head, int val) {
    DNode* newNode = new DNode(val);
    if (!head) return newNode;
    DNode* curr = head;
    while (curr->next) curr = curr->next;  // Walk to last node
    curr->next    = newNode;
    newNode->prev = curr;  // Link back to previous tail
    return head;
}
```

#### Insert After a Given Node — `O(1)`

```cpp
void insertAfter(DNode* node, int val) {
    if (!node) return;
    DNode* newNode    = new DNode(val);
    newNode->next     = node->next;
    newNode->prev     = node;
    if (node->next) node->next->prev = newNode;  // Old next's prev updated
    node->next        = newNode;
}
```

### Remove Element — Doubly Linked List

#### Remove a Given Node — `O(1)` (if pointer to node is known)

```cpp
DNode* removeNode(DNode* head, DNode* node) {
    if (!node) return head;
    if (node->prev) node->prev->next = node->next;  // Bypass in forward direction
    else            head = node->next;               // Removing head — update head
    if (node->next) node->next->prev = node->prev;  // Bypass in backward direction
    delete node;
    return head;
}
```

#### Remove by Value — `O(n)`

```cpp
DNode* removeByValue(DNode* head, int val) {
    DNode* curr = head;
    while (curr) {
        if (curr->data == val) return removeNode(head, curr);  // Reuse removeNode
        curr = curr->next;
    }
    return head;
}
```

### Search — Doubly Linked List — `O(n)`

```cpp
DNode* search(DNode* head, int val) {
    while (head) {
        if (head->data == val) return head;  // Found
        head = head->next;
    }
    return nullptr;  // Not found
}

// Reverse search (from tail)
DNode* searchReverse(DNode* tail, int val) {
    while (tail) {
        if (tail->data == val) return tail;
        tail = tail->prev;  // Walk backwards
    }
    return nullptr;
}
```

---

## Complexity Summary

| Operation | Singly LL | Doubly LL |
|---|---|---|
| Insert at head | `O(1)` | `O(1)` |
| Insert at tail | `O(n)` | `O(n)` / `O(1)` with tail ptr |
| Insert at known node | `O(n)` (find) | `O(1)` (after) |
| Delete at head | `O(1)` | `O(1)` |
| Delete known node | `O(n)` (find prev) | `O(1)` |
| Search | `O(n)` | `O(n)` |

---

## Common Interview Patterns

### Reverse a Singly Linked List — `O(n)`

```cpp
Node* reverse(Node* head) {
    Node* prev = nullptr;
    Node* curr = head;
    while (curr) {
        Node* next = curr->next;  // Save next before overwriting
        curr->next = prev;        // Reverse the link
        prev = curr;
        curr = next;
    }
    return prev;  // prev is the new head
}
```

### Detect Cycle — Floyd's Algorithm — `O(n)`

```cpp
bool hasCycle(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;        // Move 1 step
        fast = fast->next->next;  // Move 2 steps
        if (slow == fast) return true;  // Cycle detected — they meet inside the cycle
    }
    return false;
}
```

### Find Middle Node — `O(n)`

```cpp
Node* findMiddle(Node* head) {
    Node* slow = head;
    Node* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;  // Fast moves 2x — when fast hits end, slow is at middle
    }
    return slow;
}
```
