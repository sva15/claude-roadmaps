# Data Structures & Algorithms Basics
### A Complete Reference for DevOps Engineers

> **How to use this document:** You are not training to become a software developer. The goal here is specific: understand data structures well enough to reason about performance, read code confidently, and never be lost when a developer or architect mentions "this is O(n²)" or "we're using a hash map here." By the end of this document, these concepts will be clear and you'll be able to explain all of them.

---

## Table of Contents

1. [Why Data Structures Matter for DevOps Engineers](#1-why-data-structures-matter-for-devops-engineers)
2. [Big O Notation — Reasoning About Performance](#2-big-o-notation--reasoning-about-performance)
3. [Arrays](#3-arrays)
4. [Linked Lists](#4-linked-lists)
5. [Stacks](#5-stacks)
6. [Queues](#6-queues)
7. [Hash Maps / Hash Tables](#7-hash-maps--hash-tables)
8. [Sets](#8-sets)
9. [Trees — Concepts & Types](#9-trees--concepts--types)
10. [Binary Search Trees](#10-binary-search-trees)
11. [Heaps & Priority Queues](#11-heaps--priority-queues)
12. [Graphs](#12-graphs)
13. [Recursion — Understanding It Once and For All](#13-recursion--understanding-it-once-and-for-all)
14. [Sorting Algorithms](#14-sorting-algorithms)
15. [Searching Algorithms](#15-searching-algorithms)
16. [Where Data Structures Live in Real Systems](#16-where-data-structures-live-in-real-systems)
17. [Reading Code With Data Structures — Practical Examples](#17-reading-code-with-data-structures--practical-examples)
18. [Quick Reference Cheat Sheet](#18-quick-reference-cheat-sheet)

---

## 1. Why Data Structures Matter for DevOps Engineers

You might be thinking: "I'm not a developer. Why do I need to know data structures?"

Here is why — every system you operate uses them, whether you can see it or not:

| What you interact with | Data structure underneath |
|---|---|
| DNS lookup | Hash table (cache) + Tree (hierarchy) |
| Database index | B-Tree |
| Kubernetes pod scheduling | Heap (priority queue) |
| Git history | Directed Acyclic Graph (DAG) |
| Load balancer round-robin | Circular queue |
| Linux process list | Doubly linked list |
| File system directories | Tree |
| Network routing table | Trie / Hash table |
| Redis sorted sets | Skip list |
| Kafka log | Append-only array |
| Docker image layers | Stack |
| TCP connection queue | Queue |
| iptables rule matching | Linked list |

When a database query is slow, understanding B-Trees tells you why an index helps. When a deployment is queued, understanding priority queues tells you why some jobs run first. When a developer says "this operation is O(n²)", you need to know why that's terrifying at scale.

Data structures are the vocabulary of engineering conversations. Without them, you're guessing.

---

## 2. Big O Notation — Reasoning About Performance

**Big O notation** describes how an algorithm's performance (time or memory) scales as the input grows. It's the language engineers use to compare solutions.

### What Big O Answers

> "If I double the input size, how does the execution time change?"

That's the only question Big O answers. Not "how fast is it?" — but "how does it scale?"

### The Common Complexities

Ordered from best to worst:

```
O(1)        → Constant     — same speed regardless of input size
O(log n)    → Logarithmic  — grows very slowly
O(n)        → Linear       — grows proportionally to input
O(n log n)  → Linearithmic — slightly worse than linear
O(n²)       → Quadratic    — grows with the square of input
O(2^n)      → Exponential  — doubles with each additional input element
O(n!)       → Factorial    — catastrophically bad
```

### Visualizing Growth

```
n = 10:
O(1)     →       1 operation
O(log n) →       3 operations
O(n)     →      10 operations
O(n²)    →     100 operations
O(2^n)   →   1,024 operations

n = 1,000:
O(1)     →         1 operation
O(log n) →        10 operations
O(n)     →     1,000 operations
O(n²)    → 1,000,000 operations      ← 1 million!
O(2^n)   → more atoms than exist in the observable universe
```

When someone says "this code is O(n²) and we have 10,000 records" — that's 100 million operations. At 1 million records? 1 trillion operations. **That's why Big O matters.**

### O(1) — Constant Time

Performance is the same regardless of input size.

```python
# Get first element of a list — O(1)
# Doesn't matter if list has 10 or 10 million items
first = my_list[0]

# Dictionary/hash map lookup — O(1) average
user = users_dict["alice"]

# Push/pop from a stack — O(1)
stack.append(item)
```

*Real world:* Redis GET, hash table lookup, array index access.

### O(log n) — Logarithmic Time

Input is halved at each step. Grows very slowly — the most desirable after O(1).

```python
# Binary search — O(log n)
# Each comparison eliminates half the remaining elements
def binary_search(sorted_list, target):
    low, high = 0, len(sorted_list) - 1
    while low <= high:
        mid = (low + high) // 2
        if sorted_list[mid] == target:
            return mid
        elif sorted_list[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return -1
```

With 1 billion items, binary search takes only 30 steps (log₂(1,000,000,000) ≈ 30).

*Real world:* Database B-Tree index lookups, binary search, balanced tree operations.

### O(n) — Linear Time

Work grows in direct proportion to input size. Double the input → double the work.

```python
# Finding a value in an unsorted list — O(n)
# Must potentially check every element
def find_user(users_list, username):
    for user in users_list:
        if user["username"] == username:
            return user
    return None
```

*Real world:* Full table scan (no index), iterating through a log file, reading all lines in a config.

### O(n log n) — Linearithmic Time

Slightly worse than linear — typical of efficient sorting algorithms.

*Real world:* `sort()` in Python, Java, most languages. Merge sort, heap sort, Tim sort.

### O(n²) — Quadratic Time

For every element, you do work proportional to all elements. Nested loops over the same data.

```python
# Find all duplicate pairs — O(n²)
# For every element, compare with every other element
def find_duplicates(items):
    duplicates = []
    for i in range(len(items)):          # outer loop: n iterations
        for j in range(i+1, len(items)): # inner loop: n iterations
            if items[i] == items[j]:
                duplicates.append(items[i])
    return duplicates
```

*Real world:* Naive nested loop comparisons, bubble sort, selection sort. Acceptable for small n (< 1000). Catastrophic for large n.

### Rules for Calculating Big O

**Drop constants:** O(2n) → O(n). Constants don't change how it scales.

**Drop non-dominant terms:** O(n² + n) → O(n²). The n² term dominates as n grows.

**Different inputs, different variables:**
```python
def process(list_a, list_b):
    for x in list_a:      # O(a)
        for y in list_b:  # O(b)
            print(x, y)
# Total: O(a × b) — NOT O(n²) unless a == b == n
```

**Best, Average, Worst case:**
Big O usually describes the **worst case** (what happens at the worst possible input?).
- Best case (Ω, Omega): e.g., finding the element at position 0 → O(1)
- Average case (Θ, Theta): expected performance on random input
- Worst case (O, Big O): the guarantee no matter what

### Space Complexity

Big O applies to memory too, not just time.

```python
# Space O(1) — uses a fixed amount of memory regardless of input
def sum_list(numbers):
    total = 0        # one variable
    for n in numbers:
        total += n
    return total

# Space O(n) — allocates memory proportional to input
def double_all(numbers):
    result = []             # new list grows with input
    for n in numbers:
        result.append(n * 2)
    return result
```

*Real world:* A function that creates a copy of a dataset uses O(n) space. Recursive functions use O(depth) stack space — deep recursion can cause stack overflow.

---

## 3. Arrays

An **array** is the most fundamental data structure. A contiguous block of memory holding elements of the same type, accessed by index.

### How Arrays Work in Memory

```
Index:    0      1      2      3      4
        ┌──────┬──────┬──────┬──────┬──────┐
Values: │  10  │  20  │  30  │  40  │  50  │
        └──────┴──────┴──────┴──────┴──────┘
Memory:  1000   1004   1008   1012   1016    (each int = 4 bytes)
```

Because elements are contiguous in memory and the same size, accessing any element by index is a single arithmetic operation:

```
address of element[i] = base_address + (i × element_size)
array[3] = 1000 + (3 × 4) = 1012
```

This makes index access O(1) — instant, regardless of array size.

### Array Operations & Complexity

| Operation | Complexity | Explanation |
|---|---|---|
| Access by index (`arr[i]`) | O(1) | Direct memory calculation |
| Search (unsorted) | O(n) | Must check every element |
| Search (sorted, binary search) | O(log n) | Halve search space each step |
| Insert at end | O(1) amortized | Append to end |
| Insert at beginning/middle | O(n) | Must shift all elements right |
| Delete at end | O(1) | Just reduce length |
| Delete at beginning/middle | O(n) | Must shift all elements left |

### Dynamic Arrays (Lists in Python, ArrayList in Java, Vec in Rust)

A **dynamic array** grows automatically when it runs out of space:
1. Original capacity: 4 elements
2. 5th element is added → allocate new array with 2× capacity (8)
3. Copy all elements to new array
4. Continue

The doubling strategy means insertions at the end are O(1) **amortized** — occasionally expensive (when resizing), but cheap on average.

```python
# Python lists are dynamic arrays
my_list = []
my_list.append(10)     # O(1) amortized
my_list.append(20)     # O(1) amortized
my_list[0]             # O(1) — index access
my_list.insert(0, 5)   # O(n) — shifts all elements right
del my_list[0]         # O(n) — shifts all elements left
5 in my_list           # O(n) — linear search
```

### When to Use Arrays

✅ You need fast index-based access
✅ You know the size in advance or appending to the end is the main operation
✅ Memory locality matters (cache-friendly — elements are adjacent)

❌ You frequently insert/delete at the beginning or middle
❌ You need fast search (use a hash map instead)

### Arrays in the Real World

- **Kafka:** A partition is essentially an append-only array on disk (the log)
- **Time-series databases:** Store metric values in compressed arrays
- **Video/audio:** Raw audio samples, video frames stored as arrays
- **Network buffers:** Packets queued in ring buffers (circular arrays)

---

## 4. Linked Lists

A **linked list** is a sequence of **nodes**, where each node contains data and a pointer to the next node.

```
Singly Linked List:

┌────┬──┐   ┌────┬──┐   ┌────┬──┐   ┌────┬────┐
│ 10 │ ●────→ 20 │ ●────→ 30 │ ●────→ 40 │NULL│
└────┴──┘   └────┴──┘   └────┴──┘   └────┴────┘
head                                  tail

Each node: { data: 10, next: pointer_to_next_node }
```

Unlike arrays, nodes are scattered throughout memory — connected only by pointers.

```
Doubly Linked List:

NULL←──┌────┬──┐←───┌────┬──┐←───┌────┬──┐
       │ 10 │ ●────→ 20 │ ●────→ 30 │ ●────→NULL
       └────┴──┘    └────┴──┘    └────┴──┘

Each node has BOTH next and prev pointers
```

### Linked List Operations & Complexity

| Operation | Complexity | Explanation |
|---|---|---|
| Access by index | O(n) | Must traverse from head |
| Search | O(n) | Must traverse to find it |
| Insert at head | O(1) | Just update head pointer |
| Insert at tail | O(1) | If tail pointer maintained |
| Insert in middle | O(n) + O(1) | O(n) to find position, O(1) to insert |
| Delete at head | O(1) | Just update head pointer |
| Delete in middle | O(n) | Must find the node first |

### Array vs Linked List — The Key Tradeoff

| | Array | Linked List |
|---|---|---|
| **Index access** | O(1) ✅ | O(n) ❌ |
| **Insert/Delete at middle** | O(n) ❌ | O(1) ✅ (once you have the pointer) |
| **Memory usage** | Compact, contiguous | Extra memory for pointers |
| **Cache performance** | Excellent (contiguous) | Poor (scattered in memory) |
| **Dynamic sizing** | Requires copying | Trivial — just change pointers |

### When to Use Linked Lists

✅ Frequent insertions/deletions at the beginning or middle
✅ Size changes dramatically and unpredictably
✅ You never need random access by index
✅ Implementing other structures (stacks, queues, hash table chaining)

❌ You need fast index access
❌ Memory is constrained (pointer overhead)
❌ Cache performance matters

### Linked Lists in the Real World

- **Linux kernel:** The process list (`task_struct`) uses a doubly linked list
- **Browser history:** Doubly linked list — forward and back buttons
- **LRU Cache implementation:** Doubly linked list + hash map
- **Memory allocators:** Free memory blocks tracked in linked lists
- **TCP receive buffer:** Received packets in a linked list before reassembly

---

## 5. Stacks

A **stack** is a Last-In, First-Out (LIFO) data structure. Like a stack of plates — you add to the top and remove from the top.

```
Operations:  push (add)   pop (remove)   peek (look at top)

     push(D) →  │ D │  ← pop() returns D
                │ C │
                │ B │
                │ A │
                └───┘
```

### Stack Operations

| Operation | Complexity | Description |
|---|---|---|
| Push | O(1) | Add to top |
| Pop | O(1) | Remove from top |
| Peek/Top | O(1) | Look at top without removing |
| Is Empty | O(1) | Check if stack has elements |
| Search | O(n) | Must pop elements to find |

```python
# Python list as a stack
stack = []
stack.append("A")     # push — O(1)
stack.append("B")     # push
stack.append("C")     # push
top = stack[-1]       # peek — O(1), returns "C"
item = stack.pop()    # pop — O(1), returns "C"
```

### Where Stacks Appear in Real Systems

**Call Stack (every programming language):**
When function A calls function B which calls function C:
```
C is running...
┌───────────────┐
│ C() frame     │  ← current execution
│ local vars    │
├───────────────┤
│ B() frame     │  ← B called C, waiting
│ local vars    │
├───────────────┤
│ A() frame     │  ← A called B, waiting
│ local vars    │
└───────────────┘
```

When C returns → C's frame is popped → B resumes. This is why it's called a "call stack."

**Stack overflow:** When recursion goes too deep or local variables are too large, the call stack exceeds its limit (typically 8MB on Linux). The OS kills the process.

```bash
ulimit -s    # Show stack size limit (usually 8192 KB = 8MB)
```

**Docker image layers:**
Docker images are built as a stack of layers. Each `RUN`, `COPY`, `ADD` instruction adds a new layer on top. When you run a container, Docker overlays the layers (read-only) with a writable layer on top.

**Browser undo history:**
Each action pushed onto a stack. Ctrl+Z pops the last action and reverses it.

**Expression evaluation:**
Compilers and calculators use stacks to evaluate expressions:
`3 + 4 * 2` → convert to postfix → evaluate with a stack.

**Undo/redo in any editor.**

---

## 6. Queues

A **queue** is a First-In, First-Out (FIFO) data structure. Like a line at a bank — first person to arrive is the first to be served.

```
Enqueue (add to back) →  [A][B][C][D]  → Dequeue (remove from front)

New items enter here →  rear          front  ← Items leave here
```

### Queue Operations

| Operation | Complexity | Description |
|---|---|---|
| Enqueue | O(1) | Add to rear |
| Dequeue | O(1) | Remove from front |
| Peek/Front | O(1) | Look at front without removing |
| Is Empty | O(1) | Check if queue has elements |

```python
from collections import deque

queue = deque()
queue.append("A")       # enqueue — O(1)
queue.append("B")       # enqueue
queue.append("C")       # enqueue
front = queue[0]        # peek — O(1), returns "A"
item = queue.popleft()  # dequeue — O(1), returns "A"
# Use deque, NOT list, for queues — list.pop(0) is O(n) because it shifts elements
```

### Types of Queues

**Simple Queue:** FIFO — standard.

**Circular Queue (Ring Buffer):**
The rear wraps around to the beginning — efficient, fixed-size, no wasted space.
Used in: network packet buffers, audio/video streaming buffers, OS I/O buffers.

```
Ring Buffer (size 5):
position: [0][1][2][3][4]
data:     [A][B][C][ ][ ]
           ↑           ↑
          head        tail

After reading A, B and writing D, E, F:
data:     [F][ ][ ][D][E]
               ↑   ↑
              tail head   (wraps around)
```

**Priority Queue:**
Items dequeued in priority order, not arrival order. Highest-priority item always dequeued first. Implemented with a heap (covered in Section 11).

**Double-Ended Queue (Deque):**
Can add and remove from both ends efficiently.

### Queues in the Real World

- **Message queues (SQS, RabbitMQ):** Jobs processed in order received (FIFO queue)
- **TCP/IP network buffers:** Packets queued while being transmitted
- **Print spoolers:** Print jobs processed in order submitted
- **CPU scheduling:** Round-robin scheduling uses a circular queue of processes
- **Breadth-first search (BFS):** Graph/tree traversal uses a queue
- **Load balancer request queue:** Incoming requests queued when all servers busy
- **Kafka consumer group:** Each partition is a FIFO queue consumed by consumer workers

---

## 7. Hash Maps / Hash Tables

**Hash maps** (also called hash tables, dictionaries, or associative arrays) are the single most important data structure in practical programming. If you understand one thing deeply, make it this.

A hash map stores **key-value pairs** and provides O(1) average-case lookup, insertion, and deletion.

```
Hash map:
"alice" → {id: 1, email: "alice@example.com"}
"bob"   → {id: 2, email: "bob@example.com"}
"charlie" → {id: 3, email: "charlie@example.com"}
```

### How Hash Maps Work Internally

```
Step 1: hash("alice")
        Apply a hash function to the key
        "alice" → 8723641098 → mod 10 → bucket 8

Step 2: Store the value in bucket 8
        Buckets: [0][ ], [1][ ], ... [8][alice data], [9][ ]

Step 3: Lookup "alice"
        hash("alice") → 8 → go to bucket 8 → return data
        Regardless of how many keys are stored, lookup is O(1)
```

The **hash function** converts any key (string, number, object) to an integer. That integer is used (mod total buckets) to determine which bucket the value lives in.

### Collisions — When Two Keys Hash to the Same Bucket

A **collision** occurs when two different keys hash to the same bucket.

```
hash("alice") mod 10 = 4
hash("frank") mod 10 = 4   ← collision!
```

**Resolution strategies:**

**Chaining:** Each bucket holds a linked list of all key-value pairs that hashed to it.
```
Bucket 4: → ["alice", alice_data] → ["frank", frank_data] → NULL
```
Lookup: go to bucket 4, walk the list until you find "alice". Usually O(1) since lists are short. Degrades to O(n) if many collisions.

**Open Addressing (Linear Probing):** If bucket is occupied, try the next bucket, then the next.
```
"alice" → bucket 4 → occupied → try bucket 5 → empty → store here
```

### Hash Map Complexity

| Operation | Average Case | Worst Case |
|---|---|---|
| Insert | O(1) | O(n) — all keys collide |
| Lookup | O(1) | O(n) — all keys in one chain |
| Delete | O(1) | O(n) |

Worst case is theoretical — a good hash function distributes keys evenly, making collisions rare.

### Load Factor & Rehashing

The **load factor** = number of items / number of buckets.

As the load factor increases, collisions increase → performance degrades. When load factor exceeds a threshold (typically 0.7-0.75), the hash map **rehashes** — allocates a larger array and re-inserts all items.

```python
# Python dict — hash map with automatic rehashing
d = {}
d["alice"] = {"id": 1}   # O(1) average
d["bob"] = {"id": 2}     # O(1) average
d["alice"]               # O(1) average — lookup
del d["alice"]           # O(1) average — delete
"alice" in d             # O(1) average — membership check

# vs list membership check:
users_list = [{"name": "alice"}, {"name": "bob"}]
"alice" in [u["name"] for u in users_list]  # O(n) — must scan all items
```

The difference between checking membership in a list vs a dict:
- List: 1,000,000 items → up to 1,000,000 comparisons → O(n)
- Dict: 1,000,000 items → 1 hash calculation → O(1)

This is why "use a dict instead of a list for membership checks" is one of the most impactful performance tips.

### Hash Maps in the Real World

- **DNS cache:** domain → IP address mapping
- **Database buffer pool:** page_number → buffer_frame mapping
- **HTTP routing:** URL path → handler function mapping (nginx, Flask, Express)
- **Python's `dict`:** Every Python object's attributes stored in a hash map
- **Kubernetes pod labels:** label key-value pairs stored as hash maps
- **Environment variables:** Variable name → value (all OS env vars)
- **Redis:** The entire Redis key-value store is a hash table
- **JWT payload:** Key-value pairs decoded to a hash map
- **Nginx location blocks:** Internally uses hash tables for quick URL matching
- **Consistent hashing:** Used in distributed caches and load balancing

---

## 8. Sets

A **set** is a collection of unique elements — no duplicates allowed. Like a hash map but only keys, no values.

```python
# Python set
users_online = set()
users_online.add("alice")    # O(1)
users_online.add("bob")      # O(1)
users_online.add("alice")    # O(1) — silently ignored (already in set)

print(users_online)          # {"alice", "bob"} — only unique values

"alice" in users_online      # O(1) — same as hash map lookup
users_online.remove("bob")   # O(1)
```

### Set Operations

The real power of sets is mathematical set operations:

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

a | b    # Union: {1, 2, 3, 4, 5, 6, 7, 8}       — all elements from either
a & b    # Intersection: {4, 5}                   — only elements in both
a - b    # Difference: {1, 2, 3}                  — in a but not in b
a ^ b    # Symmetric difference: {1, 2, 3, 6, 7, 8} — in one but not both
```

### Sets in the Real World

- **Deduplication:** Find unique IP addresses in a log file — add all to a set, duplicates are silently dropped
- **Membership checking:** "Has user already seen this notification?" — set of seen notification IDs
- **Tag systems:** A post's tags stored as a set (can't have duplicate tags)
- **Permission systems:** A user's set of permissions
- **Network security:** IP blocklist — `if source_ip in blocked_ips_set`

```python
# Real example: Find unique IPs in a log file (O(n) time, O(unique_ips) space)
unique_ips = set()
with open("access.log") as f:
    for line in f:
        ip = line.split()[0]
        unique_ips.add(ip)
print(f"Unique IPs: {len(unique_ips)}")

# With a list this would be O(n²) — checking if each IP already in list
```

---

## 9. Trees — Concepts & Types

A **tree** is a hierarchical data structure with a root node at the top, branching into child nodes, forming a hierarchy with no cycles.

### Tree Terminology

```
                    ┌─────┐
                    │  A  │  ← Root (no parent)
                    └──┬──┘
              ┌────────┴────────┐
           ┌──┴──┐           ┌──┴──┐
           │  B  │           │  C  │  ← Internal nodes
           └──┬──┘           └──┬──┘
         ┌────┴────┐         ┌──┴──┐
      ┌──┴──┐  ┌──┴──┐    ┌──┴──┐
      │  D  │  │  E  │    │  F  │  ← Leaf nodes (no children)
      └─────┘  └─────┘    └─────┘

Root:    A (top node, no parent)
Parent:  B is parent of D and E
Child:   D and E are children of B
Leaf:    D, E, F (no children)
Siblings: D and E (same parent)
Depth:   A=0, B=1, C=1, D=2, E=2, F=2
Height:  2 (longest path from root to leaf)
```

### Tree Properties

**Binary tree:** Each node has at most 2 children (left and right).

**Balanced tree:** Height is O(log n) — no branch is much longer than another. Operations stay efficient.

**Unbalanced tree:** Can degrade to a linked list if all insertions go one direction.

```
Balanced (height = 3):        Unbalanced (height = 5):
        4                             1
       / \                             \
      2   6                             2
     / \ / \                             \
    1  3 5  7                             3
                                          \
                                           4
                                            \
                                             5
```

### Tree Traversals

How you visit every node in a tree. Three main orders:

```
Tree:
      1
     / \
    2   3
   / \
  4   5

Pre-order  (Root, Left, Right):  1, 2, 4, 5, 3
In-order   (Left, Root, Right):  4, 2, 5, 1, 3  ← gives sorted order for BST!
Post-order (Left, Right, Root):  4, 5, 2, 3, 1
Level-order (BFS — level by level): 1, 2, 3, 4, 5
```

```python
class Node:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None

def inorder(node):
    if node is None:
        return
    inorder(node.left)     # go left first
    print(node.val)        # visit root
    inorder(node.right)    # go right
```

### Trees in the Real World

| Use Case | Tree Type |
|---|---|
| Database indexes | B-Tree / B+ Tree |
| File systems (directories) | General tree |
| DNS hierarchy | General tree |
| HTML/XML DOM | General tree |
| Git commits | Directed Acyclic Graph (tree-like) |
| Decision trees (ML) | Binary tree |
| Expression parsing | Abstract Syntax Tree (AST) |
| Network routing (Trie) | Prefix tree |
| Kubernetes resource hierarchy | General tree |

---

## 10. Binary Search Trees

A **Binary Search Tree (BST)** is a binary tree with the ordering property:

> For every node, all values in the left subtree are smaller, and all values in the right subtree are larger.

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

Is this a valid BST?
- 8's left subtree: all < 8 ✅ (3, 1, 6, 4, 7 are all < 8)
- 8's right subtree: all > 8 ✅ (10, 14, 13 are all > 8)
- 3's left: 1 < 3 ✅ | 3's right: 6 > 3 ✅
- And so on recursively
```

### BST Operations

| Operation | Average (Balanced) | Worst (Unbalanced) |
|---|---|---|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |
| Find min/max | O(log n) | O(n) |
| In-order traversal | O(n) | O(n) |

In-order traversal of a BST gives all values in **sorted order** — this is why it's fundamental to database indexes.

### Search in a BST

```
Search for 6 in the BST above:
Start at root: 8 — is 6 < 8? Yes → go left
At node 3: is 6 > 3? Yes → go right
At node 6: found! ✅

Each step eliminates half the remaining nodes (in a balanced tree)
This is O(log n) search — exactly like binary search, but on a tree
```

### Self-Balancing BSTs

When insertions are not random, a BST can become unbalanced (tall and thin), losing its O(log n) advantage. Self-balancing variants fix this automatically:

**AVL Tree:** Rebalances after every insertion/deletion via rotations. Strict balance — height difference between subtrees ≤ 1.

**Red-Black Tree:** Less strictly balanced than AVL, but faster insertions/deletions. Used in: Java's `TreeMap`, C++ `std::map`, Linux kernel's process scheduler.

**B-Tree / B+ Tree:** Multi-way tree (each node can have many children). Optimized for disk reads — nodes are page-sized, minimizing disk I/O. **Used in every relational database index.**

```
B-Tree (order 3 — each node can have up to 3 keys, 4 children):

         [20  | 50]
        /      |     \
 [5|10|15] [25|30] [60|70|80]

One disk read per level → very fast even with millions of rows
Height of 3-4 covers billions of records
```

---

## 11. Heaps & Priority Queues

A **heap** is a special tree-based data structure that satisfies the **heap property**:

**Max-heap:** Every parent node is **greater than or equal to** its children. The maximum element is always at the root.

**Min-heap:** Every parent node is **less than or equal to** its children. The minimum element is always at the root.

```
Max-heap:              Min-heap:
       9                      1
      / \                    / \
     7   8                  3   2
    / \ / \                / \ / \
   4  5 6  3              7  5 6  4
```

### The Key Insight About Heaps

A heap is **not fully sorted** — it only guarantees that the min/max is at the top. This makes inserting and deleting efficient while always giving you instant access to the most important element.

### Heap Operations

| Operation | Complexity |
|---|---|
| Get max/min (peek root) | O(1) |
| Insert | O(log n) |
| Delete max/min | O(log n) |
| Build heap from array | O(n) |
| Search | O(n) — heap is not sorted |

### Priority Queue

A **priority queue** is an abstract data type where each element has a priority, and the highest-priority element is always dequeued first. Heaps are the standard implementation.

```python
import heapq

# Python's heapq is a min-heap
pq = []
heapq.heappush(pq, (3, "medium priority task"))
heapq.heappush(pq, (1, "critical task"))
heapq.heappush(pq, (5, "low priority task"))

heapq.heappop(pq)   # returns (1, "critical task") — lowest number = highest priority
heapq.heappop(pq)   # returns (3, "medium priority task")
heapq.heappop(pq)   # returns (5, "low priority task")
```

### Heaps in the Real World

**Kubernetes Pod Scheduler:**
When multiple pods are waiting to be scheduled, the scheduler uses a priority queue (heap) to always pick the highest-priority pod first.

**Process scheduling (Linux CFS):**
The Completely Fair Scheduler uses a red-black tree (functionally similar to a heap) to always find the process that has received the least CPU time.

**Dijkstra's shortest path algorithm:**
Uses a min-heap to always explore the nearest unvisited node next — the foundation of routing protocols (OSPF).

**Heap sort:**
Build a max-heap from an array → repeatedly extract max → sorted array. O(n log n), in-place.

**Top-K problems:**
"Find the 10 highest-memory processes" → min-heap of size 10. Process each item — if larger than heap minimum, replace it. O(n log k) time.

**Prometheus/time-series retention:**
Oldest data points expired using a min-heap ordered by timestamp.

---

## 12. Graphs

A **graph** is a collection of **nodes (vertices)** connected by **edges**. Unlike trees, graphs can have cycles, multiple paths between nodes, and nodes with no connections.

```
Undirected graph:           Directed graph (Digraph):
  A --- B                     A ──→ B
  |     |                     ↑     |
  C --- D --- E                C ←── D ──→ E
```

### Graph Terminology

- **Vertex (Node):** A data point (server, city, user, file)
- **Edge:** A connection between two vertices (network link, road, friendship, import)
- **Directed:** Edges have direction (one-way)
- **Undirected:** Edges go both ways (bidirectional)
- **Weighted:** Edges have a value (distance, cost, latency, bandwidth)
- **Degree:** Number of edges connected to a vertex
- **Path:** A sequence of vertices connected by edges
- **Cycle:** A path that starts and ends at the same vertex
- **DAG (Directed Acyclic Graph):** Directed graph with no cycles — critically important

### Graph Representation

**Adjacency List:** Each node stores a list of its neighbors.
```python
graph = {
    "A": ["B", "C"],
    "B": ["A", "D"],
    "C": ["A", "D"],
    "D": ["B", "C", "E"],
    "E": ["D"]
}
# Space: O(V + E) — efficient for sparse graphs
```

**Adjacency Matrix:** 2D array where `matrix[i][j] = 1` if edge exists.
```
  A B C D E
A[0,1,1,0,0]
B[1,0,0,1,0]
C[1,0,0,1,0]
D[0,1,1,0,1]
E[0,0,0,1,0]
# Space: O(V²) — good for dense graphs, fast edge lookup
```

### Graph Traversal

**BFS (Breadth-First Search):**
Explore all neighbors before going deeper. Uses a queue.
```python
from collections import deque

def bfs(graph, start):
    visited = set()
    queue = deque([start])
    visited.add(start)
    while queue:
        node = queue.popleft()
        print(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```
Use for: **Shortest path in unweighted graphs**, level-order traversal, finding all reachable nodes.

**DFS (Depth-First Search):**
Go as deep as possible before backtracking. Uses a stack (or recursion).
```python
def dfs(graph, start, visited=None):
    if visited is None:
        visited = set()
    visited.add(start)
    print(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```
Use for: **Cycle detection**, topological sort, maze solving, finding connected components.

### Graphs in the Real World

**Git commits — DAG:**
```
A ← B ← C ← D  (main branch)
         ↑
         E ← F  (feature branch)
```
Each commit points to its parents. No cycles (can't commit to the past). Merging creates a commit with two parents.

**Network topology:**
Servers and network devices are vertices. Network links are edges (weighted by bandwidth/latency). Routing algorithms (Dijkstra's, OSPF) find shortest paths.

**Kubernetes dependency graph:**
A Pod depends on a ConfigMap and a Secret. The scheduler must apply them in the right order. Kubernetes uses a DAG to resolve dependency order.

**Terraform dependency graph:**
`terraform graph` shows the DAG of resource dependencies — which resources must be created before others.

```bash
terraform graph | dot -Tpng > graph.png   # Visualize the dependency graph
```

**Microservice dependencies:**
Service A calls B and C. B calls D. D calls E. This is a graph — circular dependencies (A→B→A) are a serious design smell detected by analyzing the graph for cycles.

**Social networks:**
Users are vertices. Friendships/follows are edges. "Friends of friends" = BFS to depth 2.

**CDN routing:**
Finding the nearest edge node to a user = shortest path in a weighted graph.

---

## 13. Recursion — Understanding It Once and For All

**Recursion** is when a function calls itself to solve a smaller version of the same problem.

### The Anatomy of Every Recursive Function

Two parts are always required:

1. **Base case:** The condition where the function stops calling itself (the termination condition)
2. **Recursive case:** Where the function calls itself with a smaller/simpler input

```python
def countdown(n):
    # Base case — stop recursion
    if n <= 0:
        print("Go!")
        return
    # Recursive case — call with smaller input
    print(n)
    countdown(n - 1)   # ← calls itself with n-1

countdown(5)
# Output: 5, 4, 3, 2, 1, Go!
```

### How Recursion Uses the Call Stack

Every recursive call adds a frame to the call stack:

```
countdown(3):
┌──────────────────┐
│ countdown(3)     │  ← currently paused at countdown(n-1)
│ n = 3            │
├──────────────────┤
│ countdown(2)     │  ← currently paused at countdown(n-1)
│ n = 2            │
├──────────────────┤
│ countdown(1)     │  ← currently paused at countdown(n-1)
│ n = 1            │
├──────────────────┤
│ countdown(0)     │  ← currently executing (prints "Go!")
│ n = 0            │
└──────────────────┘

When countdown(0) returns → its frame is popped
countdown(1) resumes → returns
countdown(2) resumes → returns
countdown(3) resumes → returns
Stack unwinds
```

**Stack overflow** happens when recursion is too deep — the call stack fills up completely.

```bash
# Each recursive call uses ~hundreds of bytes of stack space
# Python's default recursion limit
import sys
sys.getrecursionlimit()   # 1000 by default

# Increase if needed (carefully)
sys.setrecursionlimit(10000)
```

### Classic Recursive Example — Factorial

```python
def factorial(n):
    # Base case
    if n == 0 or n == 1:
        return 1
    # Recursive case: n! = n × (n-1)!
    return n * factorial(n - 1)

factorial(5)
# = 5 × factorial(4)
# = 5 × 4 × factorial(3)
# = 5 × 4 × 3 × factorial(2)
# = 5 × 4 × 3 × 2 × factorial(1)
# = 5 × 4 × 3 × 2 × 1
# = 120
```

### Recursive Tree Traversal

Tree traversal is the most natural use of recursion because trees are themselves recursive structures (a tree is a root with children that are themselves trees):

```python
def inorder(node):
    if node is None:      # Base case — empty tree
        return
    inorder(node.left)    # Recurse on left subtree
    print(node.val)       # Process root
    inorder(node.right)   # Recurse on right subtree
```

### When to Use Recursion

✅ Problems that are naturally recursive: tree traversal, graph DFS, divide-and-conquer
✅ The input can be broken into smaller versions of the same problem
✅ Code clarity is more important than maximum performance

❌ Very deep recursion (use iteration instead to avoid stack overflow)
❌ Performance-critical code without memoization (recursive calls can recompute the same values many times)

### Memoization — Caching Recursive Results

```python
# Without memoization — O(2^n) — exponential!
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
# fibonacci(40) makes 330 million calls — terrible!

# With memoization — O(n) — each value computed once
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
# fibonacci(40) makes 41 calls — excellent!
```

This pattern — caching results of expensive recursive calls — is called **dynamic programming**.

---

## 14. Sorting Algorithms

You rarely implement sorting from scratch — languages have built-in sorts. But knowing *how* they work tells you when to use which, and what "the list is already sorted" means for performance.

### Comparison Sort Algorithms

#### Bubble Sort — O(n²) — Educational Only

Repeatedly swap adjacent elements if they're in the wrong order. Very slow — never use in production.

```
[5, 3, 1, 4, 2]
Pass 1: [3,5,1,4,2] → [3,1,5,4,2] → [3,1,4,5,2] → [3,1,4,2,5]
Pass 2: [1,3,4,2,5] → [1,3,2,4,5] → [1,3,2,4,5]
...eventually sorted
```

#### Merge Sort — O(n log n) — Stable, Reliable

Divide the array in half, recursively sort each half, merge the sorted halves.

```
[5, 3, 1, 4, 2]
          ↓ divide
    [5,3,1]    [4,2]
     ↓ divide   ↓ divide
  [5,3]  [1]  [4]  [2]
   ↓           ↓
 [3,5]  [1]  [2,4]
   ↓ merge     ↓ merge
  [1,3,5]    [2,4]
        ↓ merge
    [1,2,3,4,5]
```

Time: O(n log n) always — doesn't matter if input is sorted or reverse sorted.
Space: O(n) — needs extra array for merging.
**Stable:** Equal elements maintain their original relative order.

#### Quick Sort — O(n log n) average, O(n²) worst

Pick a **pivot** element. Partition: all elements less than pivot go left, greater go right. Recursively sort each partition.

```
[3, 6, 8, 10, 1, 2, 1]  ← choose 3 as pivot
[1, 2, 1]  [3]  [6, 8, 10]   ← partition
↓ recurse        ↓ recurse
[1, 1, 2]  [3]  [6, 8, 10]
      ↓
[1, 1, 2, 3, 6, 8, 10]
```

Average: O(n log n). Worst: O(n²) if always picking worst pivot (already sorted array).
Space: O(log n) stack space — in-place.
**Not stable** by default.

Python's sort (`Timsort`) and Java's `Arrays.sort` for objects are variations/hybrids of merge sort. Java's `Arrays.sort` for primitives uses dual-pivot quicksort.

### Non-Comparison Sorts

When your data has known characteristics, you can sort faster than O(n log n):

**Counting Sort — O(n + k):**
If values are integers in a small range [0..k], count occurrences of each value. Works when k is small.

**Radix Sort — O(nk):**
Sort integers digit by digit. Used in databases for sorting large integer datasets efficiently.

### Sorting Summary

| Algorithm | Best | Average | Worst | Space | Stable | Use When |
|---|---|---|---|---|---|---|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | Yes | Never (educational) |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Need stable sort |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | No | General — fast in practice |
| Tim | O(n) | O(n log n) | O(n log n) | O(n) | Yes | Python/Java built-in |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | No | In-place, no worst case |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | Small integer range |

```python
# In practice — just use the built-in sort (Timsort in Python)
my_list.sort()                              # Sort in place — O(n log n)
sorted_list = sorted(my_list)              # Returns new sorted list
sorted_users = sorted(users, key=lambda u: u["name"])  # Sort by field
sorted_desc = sorted(numbers, reverse=True) # Descending
```

---

## 15. Searching Algorithms

### Linear Search — O(n)

Check each element one by one until found (or exhausted).

```python
def linear_search(items, target):
    for i, item in enumerate(items):
        if item == target:
            return i
    return -1
```

When to use: Unsorted data, small datasets, or when you only search once (no benefit to sorting first).

### Binary Search — O(log n)

Works ONLY on **sorted** data. Repeatedly halve the search space:

```python
def binary_search(sorted_items, target):
    low, high = 0, len(sorted_items) - 1

    while low <= high:
        mid = (low + high) // 2

        if sorted_items[mid] == target:
            return mid                    # Found!
        elif sorted_items[mid] < target:
            low = mid + 1                 # Target is in right half
        else:
            high = mid - 1               # Target is in left half

    return -1                             # Not found

# Example:
arr = [1, 3, 5, 7, 9, 11, 13, 15]
binary_search(arr, 7)
# Check middle (7 at index 3): 7 == 7 → found at index 3!
# Only took 1 comparison (would take up to 3-4 for linear search)
```

**Why it's O(log n):**
- 8 elements → 3 comparisons max (log₂(8) = 3)
- 1,000,000 elements → 20 comparisons max (log₂(1,000,000) ≈ 20)
- 1,000,000,000 elements → 30 comparisons max

```python
# Python has binary search built in
import bisect
arr = [1, 3, 5, 7, 9]
bisect.bisect_left(arr, 7)   # Returns index where 7 would be inserted (index 3)
bisect.insort(arr, 6)        # Insert 6 maintaining sorted order
```

### When Binary Search Appears in Real Systems

- **Database index lookup:** B-Tree nodes use binary search within each page
- **Debugging with git bisect:** Binary search through commits to find which introduced a bug

```bash
git bisect start
git bisect bad HEAD          # Current commit is broken
git bisect good v1.0         # v1.0 worked fine
# Git binary searches through commits, checking each one
git bisect run pytest tests/ # Automatically test each commit
# git bisect finds the exact commit that introduced the bug in O(log n) steps
```

- **Finding the right server:** Consistent hashing uses binary search on a sorted ring
- **Configuration value lookup:** Binary search in sorted config arrays
- **`grep` on sorted log files:** Some log analysis tools binary search for timestamps

---

## 16. Where Data Structures Live in Real Systems

This section directly connects what you've learned to the systems you manage every day.

### PostgreSQL / Database Indexes — B-Tree

Every time you create an index in PostgreSQL:
```sql
CREATE INDEX idx_users_email ON users(email);
```

PostgreSQL creates a **B-Tree** index. This is why:
- Finding a user by email is O(log n) — not O(n)
- Range queries work: `WHERE created_at BETWEEN X AND Y` — B-Tree supports range scans
- Without an index: full table scan = O(n) = reads every row

```sql
-- This shows you the B-Tree in action:
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';
-- "Index Scan using idx_users_email" ← B-Tree lookup, O(log n)

EXPLAIN SELECT * FROM users WHERE username LIKE '%alice%';
-- "Seq Scan" ← Can't use B-Tree for leading wildcard, O(n)
```

### Redis — Hash Table + Skip List + Linked List

Redis is a data structures server. Its internal implementation:
- **Strings/Keys:** Hash table at the top level — O(1) lookup
- **Sorted Sets (ZADD/ZRANK):** Skip list (probabilistic balanced BST) — O(log n) operations
- **Lists (LPUSH/RPUSH):** Doubly linked list — O(1) push/pop from either end
- **Sets:** Hash table — O(1) membership
- **Hashes:** Hash table — O(1) field lookup

```bash
# These commands map directly to data structure operations:
SET key value        # Hash table insert — O(1)
GET key              # Hash table lookup — O(1)
LPUSH list val       # Linked list prepend — O(1)
ZADD zset score mem  # Skip list insert — O(log n)
ZRANGE zset 0 -1     # Skip list range scan — O(log n + output)
SADD myset member    # Hash table insert — O(1)
SISMEMBER myset mem  # Hash table lookup — O(1)
```

### Kubernetes — Multiple Data Structures

**etcd (Kubernetes' state store):** B-Tree index for fast key lookups.

**Pod scheduler:** Priority queue (heap) — always schedules highest-priority pending pod first.

**kube-proxy (service routing):** Hash maps — Service ClusterIP → backend Pod IPs.

**API server cache:** Hash maps — object UID → object state, for fast in-memory lookups.

```bash
# You can see the priority queue at work:
kubectl get pods --sort-by=.metadata.creationTimestamp
# Pods sorted — the scheduler processed them in priority queue order

# View pod priority
kubectl get pod mypod -o jsonpath='{.spec.priority}'
```

### Git — DAG (Directed Acyclic Graph)

Git's entire data model is a DAG of commits:

```bash
git log --graph --oneline
# * a3f5c9d (HEAD -> main) Fix login bug
# * 7b2e4f1 Add user authentication
# |\ 
# | * c8d1a3b (feature/payments) Add payment processing
# | * 9e7f2c4 Skeleton payment module
# |/
# * 4a1b8e2 Initial commit

# git log --graph visualizes the DAG
# Each commit is a node
# Parent pointers are directed edges
# No cycles — you can't be your own ancestor
```

```bash
git bisect start  # Binary search through the DAG to find a bug
```

### Linux Kernel — Linked Lists, Red-Black Trees, Hash Tables

The Linux kernel uses all of these:

```bash
# Process list — doubly linked list
ps aux   # You're reading the kernel's linked list of task_structs

# Kernel uses red-black trees for:
# - CFS scheduler (process scheduling)
# - Virtual memory area management
# - File descriptor management

# Hash tables for:
# - inode cache (filename → inode)
# - dentry cache (directory entry cache)
# - Network connection tracking
cat /proc/slabinfo   # Shows kernel slab allocator — manages these data structure caches
```

### Kafka — Append-Only Log (Array on Disk)

A Kafka partition is conceptually an append-only array:
- Producers append to the end: O(1) — sequential disk write (very fast)
- Consumers read sequentially from an offset: O(1) per message
- Offset is an index into the array: O(1) lookup

```
Offset:  0    1    2    3    4    5
Data:   [msg][msg][msg][msg][msg][msg]  → append new messages here
         ↑ consumer A is at offset 2
              ↑ consumer B is at offset 1
```

This is why Kafka is so fast — it's just reading and writing arrays sequentially to disk. Sequential disk I/O is 10-100× faster than random I/O.

### Nginx / Load Balancers — Hash Tables + Queues

Nginx uses:
- **Hash tables:** `server_names_hash` — O(1) lookup from `Host:` header to virtual host configuration
- **Red-black trees:** Timer management — which connections time out next
- **Queues (ring buffers):** Incoming connection queue, event queue for the event loop

```nginx
# This creates a hash table of server names for O(1) routing
server_names_hash_bucket_size 64;
server_names_hash_max_size 512;
```

---

## 17. Reading Code With Data Structures — Practical Examples

Now let's look at real patterns you'll see in codebases and configuration files, and understand them through the lens of data structures.

### Pattern 1: The Lookup Optimization

```python
# Slow version — O(n) per lookup, O(n²) total
def process_requests(requests, banned_ips):
    for req in requests:
        if req.ip in banned_ips:    # banned_ips is a list → O(n) scan
            reject(req)

# Fast version — O(1) per lookup, O(n) total
def process_requests(requests, banned_ips):
    banned_set = set(banned_ips)    # Convert to set — one-time O(n) cost
    for req in requests:
        if req.ip in banned_set:    # O(1) lookup
            reject(req)
```

When you see someone suggesting "put that in a set/dict instead of a list" — this is exactly why. You're changing O(n) lookups to O(1).

### Pattern 2: Rate Limiting With a Queue

```python
from collections import deque
import time

class RateLimiter:
    """Sliding window rate limiter — 100 requests per minute"""
    def __init__(self, limit=100, window=60):
        self.limit = limit
        self.window = window
        self.timestamps = deque()   # Queue of request timestamps

    def allow(self):
        now = time.time()

        # Remove timestamps older than the window (from front of queue)
        while self.timestamps and now - self.timestamps[0] > self.window:
            self.timestamps.popleft()   # O(1) dequeue

        if len(self.timestamps) < self.limit:
            self.timestamps.append(now)  # O(1) enqueue
            return True
        return False
```

A deque (double-ended queue) is the perfect data structure here — add to back, remove from front, both O(1).

### Pattern 3: LRU Cache (Linked List + Hash Map)

An LRU (Least Recently Used) cache evicts the least recently used item when full. This requires:
- O(1) lookup: hash map
- O(1) move-to-front on access: doubly linked list

```
Hash Map:                    Doubly Linked List (most → least recent):
"alice" → node_alice         [alice] ↔ [charlie] ↔ [bob]
"bob"   → node_bob
"charlie" → node_charlie

Access "bob":
1. Hash map: find node_bob in O(1)
2. Remove node_bob from its current position in list: O(1)
3. Move node_bob to front: O(1)
Result: [bob] ↔ [alice] ↔ [charlie]

Cache full, add "diana":
1. Evict least recent (tail): remove "charlie"
2. Add "diana" to front
Result: [diana] ↔ [bob] ↔ [alice]
```

This is exactly how Redis implements LRU eviction and how CPU caches work.

```python
from functools import lru_cache

@lru_cache(maxsize=1000)    # Python's built-in LRU cache
def expensive_computation(n):
    # This result is cached — second call with same n is O(1)
    return n * n * n
```

### Pattern 4: Topological Sort (Deployment Ordering)

When deploying Terraform or Kubernetes resources, dependencies must be applied in order. This is a topological sort of a DAG.

```
Resources:
VPC must exist before Subnet
Subnet must exist before EC2
Security Group must exist before EC2
EC2 must exist before Application Load Balancer

DAG:
VPC → Subnet → EC2 → ALB
         ↗
Security Group

Topological order: [VPC, Security Group, Subnet, EC2, ALB]
(Any order where each resource comes after its dependencies)
```

```bash
# Terraform does this automatically
terraform apply
# Terraform builds the dependency DAG, sorts it topologically,
# and applies resources in the correct order
# (with parallelism for resources with no dependency on each other)
```

### Pattern 5: Binary Search in Logs

```bash
# Searching for events at a specific time in a large sorted log file
# Instead of grep (O(n)), use binary search

# Find all log lines from 14:30 onwards
# Using awk to binary-search by line (conceptually):
grep -n "2024-01-15 14:30" access.log

# git bisect is binary search through commit history:
git bisect start
git bisect bad                    # current commit is broken
git bisect good v2.3.0            # this version was fine
# git checks commits in O(log n) order to find the breaking commit
```

---

## 18. Quick Reference Cheat Sheet

### Big O Complexity Summary

```
O(1)        → Hash map lookup, array index access, stack push/pop
O(log n)    → Binary search, BST search (balanced), heap insert/delete
O(n)        → Linear search, array traversal, linked list traversal
O(n log n)  → Merge sort, heap sort, quick sort (average)
O(n²)       → Nested loops over same data, bubble sort, naive duplicate finding
O(2^n)      → Recursive fibonacci without memoization, subset enumeration
```

### Data Structure Operations At a Glance

```
Array:
  Access:      O(1)   ← Index math — instant
  Search:      O(n)   ← Must scan
  Insert end:  O(1)   ← Append
  Insert mid:  O(n)   ← Must shift elements

Linked List:
  Access:      O(n)   ← Must traverse from head
  Search:      O(n)   ← Must traverse
  Insert head: O(1)   ← Just update pointer
  Insert mid:  O(n)+O(1) ← Find: O(n), insert: O(1)

Stack (LIFO):
  Push/Pop:    O(1)   ← Top only
  Peek:        O(1)

Queue (FIFO):
  Enqueue/Dequeue: O(1)   ← Rear and front

Hash Map:
  Lookup:      O(1) avg ← Hash + direct access
  Insert:      O(1) avg
  Delete:      O(1) avg

Binary Search Tree (Balanced):
  Search:      O(log n)
  Insert:      O(log n)
  Delete:      O(log n)

Heap:
  Get min/max: O(1)   ← Always at root
  Insert:      O(log n)
  Delete min/max: O(log n)
```

### Choosing a Data Structure

```
Need fast lookup by key?               → Hash Map (dict)
Need unique elements?                  → Set
Need LIFO (undo, call stack)?          → Stack
Need FIFO (task queue, BFS)?           → Queue
Need min/max quickly?                  → Heap (Priority Queue)
Need sorted data with fast search?     → BST / Sorted Array + Binary Search
Need hierarchical data?                → Tree
Need relationships with cycles?        → Graph
Need fast append + sequential read?    → Array / Dynamic Array
Need fast insert/delete at any point?  → Linked List
```

### Algorithm Complexity

```
Binary Search:         O(log n) — requires sorted input
Linear Search:         O(n)
BFS (Graph):           O(V + E) — V=vertices, E=edges
DFS (Graph):           O(V + E)
Merge Sort:            O(n log n) — always
Quick Sort:            O(n log n) avg, O(n²) worst
Heap Sort:             O(n log n) — always, in-place
Dijkstra's:            O((V + E) log V) — with priority queue
```

### Real System Mappings

```
Database index         → B-Tree
DNS cache              → Hash table
Redis sorted set       → Skip list
Redis list             → Doubly linked list
Git history            → DAG (Directed Acyclic Graph)
Kafka partition        → Append-only array (log)
Kubernetes scheduler   → Priority queue (heap)
Linux process list     → Doubly linked list
Nginx server routing   → Hash table
File system dirs       → Tree
LRU cache              → Hash map + Doubly linked list
Terraform/K8s deps     → DAG + Topological sort
Network routing        → Weighted graph + Dijkstra
```

---

## Data Structures Concepts Map

```
FOUNDATIONS
  Big O Notation
    O(1) → O(log n) → O(n) → O(n log n) → O(n²) → O(2^n)
    Time complexity + Space complexity
    Always ask: "What happens when input doubles?"

LINEAR DATA STRUCTURES
  Array        → O(1) index access, O(n) insert/delete middle
  Linked List  → O(1) insert at head, O(n) access by index
  Stack (LIFO) → push/pop from top, O(1) both
  Queue (FIFO) → enqueue rear, dequeue front, O(1) both

HASH-BASED STRUCTURES
  Hash Map     → O(1) average for all ops, key-value pairs
  Hash Set     → O(1) membership, uniqueness enforcement

TREE-BASED STRUCTURES
  Binary Tree  → hierarchical, two children per node
  BST          → ordered binary tree, O(log n) search/insert/delete
  B-Tree       → multi-way, disk-optimized → DATABASE INDEXES
  Heap         → min/max always at root → PRIORITY QUEUES

GRAPH STRUCTURES
  Undirected   → bidirectional edges
  Directed     → one-way edges
  DAG          → directed, no cycles → GIT, TERRAFORM, K8S DEPS
  Weighted     → edges have cost → ROUTING, SHORTEST PATH

KEY ALGORITHMS
  Binary Search    → O(log n), requires sorted data
  BFS              → shortest path (unweighted), level traversal
  DFS              → cycle detection, topological sort
  Merge/Quick Sort → O(n log n), general sorting
  Dijkstra's       → shortest path in weighted graph

RECURSION
  Base case + Recursive case
  Call stack grows with recursion depth
  Memoization → cache results → Dynamic Programming
  Tree/graph traversal → natural recursive structures

REAL SYSTEM CONNECTIONS
  Every index, cache, scheduler, and router uses these structures
  Knowing them lets you reason about WHY systems behave the way they do
```

---

## Conclusion — The Full Learning Path Complete

You now have all six documents:

| Document | What it gives you |
|---|---|
| **OS Fundamentals** | How computers actually work underneath everything |
| **Networking Fundamentals** | How data moves between machines |
| **Security Fundamentals** | How to protect systems and understand threats |
| **Database Fundamentals** | How data is stored, queried, and scaled |
| **System Design Fundamentals** | How large systems are architected |
| **Data Structures & Algorithms** | The vocabulary and reasoning tools of engineering |

These six areas form a complete mental model of how modern software infrastructure works — from the CPU cache line to the globally distributed CDN, from a single process on Linux to a Kubernetes cluster running 10,000 pods.

The shift you are making is from **task executor** to **system thinker**. That shift is not instant. But every time you read these documents, run the commands, and connect a concept to something you see at work — the understanding compounds.

The next step is applying it: when you configure nginx, think about hash tables. When you add an index, think about B-Trees. When you design an alert, think about SLOs. When a deployment fails, think about which layer of the stack to investigate first.

That is the difference between 6.5 years of experience and 6.5 years of *understanding*.

---

*Document version 1.0 — Data Structures & Algorithms Basics for DevOps Engineers. Final document in the CS Fundamentals series.*
