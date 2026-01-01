# Collections & Data Structures

## Table of Contents

### Data Structures Overview
- [Q1: What are the main differences between ArrayList and LinkedList?](#q1)
- [Q2: What is the difference between HashMap and TreeMap?](#q2)
- [Q3: What is the difference between HashSet, LinkedHashSet, and TreeSet?](#q3)
- [Q4: What happens when two keys have the same hashCode in HashMap?](#q4)
- [Q5: Why should you override both hashCode() and equals()?](#q5)

### Collections Internals
- [Q6: How does ArrayList work internally?](#q6)
- [Q7: How does HashMap work internally?](#q7)
- [Q8: How does ConcurrentHashMap work internally?](#q8)
- [Q9: How does LinkedList work internally?](#q9)
- [Q10: How does TreeMap work internally?](#q10)

---

## Data Structures Overview

<a id="q1"></a>
### Q1: What are the main differences between ArrayList and LinkedList?
**Answer:**
| ArrayList | LinkedList |
|-----------|------------|
| Backed by dynamic array | Backed by doubly-linked list |
| O(1) random access | O(n) random access |
| O(n) insertion/deletion in middle | O(1) insertion/deletion (if node known) |
| Better for read-heavy operations | Better for write-heavy operations |
| Less memory overhead | More memory (stores next/prev pointers) |

<a id="q2"></a>
### Q2: What is the difference between HashMap and TreeMap?
**Answer:**
| HashMap | TreeMap |
|---------|---------|
| O(1) for get/put operations | O(log n) for get/put operations |
| No ordering guaranteed | Keys are sorted in natural order |
| Allows one null key | Does not allow null keys |
| Uses hashing | Uses Red-Black tree |
| Better for most use cases | Use when sorted order is needed |

<a id="q3"></a>
### Q3: What is the difference between HashSet, LinkedHashSet, and TreeSet?
**Answer:**
- **HashSet**: No ordering, O(1) operations, allows null
- **LinkedHashSet**: Maintains insertion order, O(1) operations
- **TreeSet**: Sorted order, O(log n) operations, no null allowed

<a id="q4"></a>
### Q4: What happens when two keys have the same hashCode in HashMap?
**Answer:**
When two keys have the same hashCode (collision):
1. Both entries are stored in the same bucket
2. Before Java 8: Uses LinkedList to chain entries
3. Java 8+: Uses LinkedList initially, converts to balanced tree (Red-Black) when bucket size exceeds 8 (TREEIFY_THRESHOLD)
4. To find correct value, `equals()` is called on each entry in the bucket

<a id="q5"></a>
### Q5: Why should you override both hashCode() and equals()?
**Answer:**
The contract states:
- If two objects are equal (`equals()` returns true), they **must** have the same hashCode
- If hashCodes are equal, objects may or may not be equal

If you override only `equals()`:
```java
Map<Person, String> map = new HashMap<>();
Person p1 = new Person("John");
Person p2 = new Person("John");
map.put(p1, "Engineer");
map.get(p2); // Returns null! Different hashCode, wrong bucket
```

---

## Collections Internals

<a id="q6"></a>
### Q6: How does ArrayList work internally?
**Answer:**
ArrayList is backed by a dynamically resizing array.

**Structure:**
```java
public class ArrayList<E> {
    transient Object[] elementData;  // Backing array
    private int size;                // Number of elements
    private static final int DEFAULT_CAPACITY = 10;
}
```

**Key operations:**

| Operation | Time Complexity | Description |
|-----------|-----------------|-------------|
| get(index) | O(1) | Direct array access |
| add(element) | O(1) amortized | May trigger resize |
| add(index, element) | O(n) | Shifts elements right |
| remove(index) | O(n) | Shifts elements left |
| contains(element) | O(n) | Linear search |

**Resizing mechanism:**
```java
// When array is full:
// 1. Create new array with 1.5x capacity
// 2. Copy all elements to new array
private void grow() {
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5x
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**Memory layout:**
```
ArrayList object:
┌─────────────────┐
│ elementData ────┼──► [obj0][obj1][obj2][null][null][null]
│ size: 3         │
└─────────────────┘
```

<a id="q7"></a>
### Q7: How does HashMap work internally?
**Answer:**
HashMap uses an array of buckets (nodes) with hash-based indexing.

**Structure (Java 8+):**
```java
public class HashMap<K,V> {
    transient Node<K,V>[] table;     // Bucket array
    int threshold;                    // Capacity * loadFactor
    final float loadFactor;           // Default 0.75
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    static final int TREEIFY_THRESHOLD = 8;  // Convert to tree
}
```

**How put() works:**
```
1. Calculate hash: hash = key.hashCode() ^ (key.hashCode() >>> 16)
2. Find bucket index: index = hash & (table.length - 1)
3. If bucket empty: create new Node
4. If key exists: update value
5. If collision: LinkedList until 8 nodes, then Red-Black Tree
6. If size > threshold: resize (double capacity)
```

**Visual representation:**
```
HashMap (capacity=16, loadFactor=0.75):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │... │
└────┴────┴────┴────┴────┴────┴────┴────┘
  │         │
  ▼         ▼
[K,V]     [K,V] → [K,V] → [K,V]  (LinkedList for collisions)
            │
            ▼ (if > 8 nodes)
          [Tree]  (Red-Black Tree)
```

**Time complexities:**
| Operation | Average | Worst (many collisions) |
|-----------|---------|------------------------|
| get() | O(1) | O(log n) with tree |
| put() | O(1) | O(log n) with tree |
| containsKey() | O(1) | O(log n) with tree |

<a id="q8"></a>
### Q8: How does ConcurrentHashMap work internally?
**Answer:**
ConcurrentHashMap provides thread-safe operations without locking the entire map.

**Java 8+ implementation:**
- Uses CAS (Compare-And-Swap) for updates
- Synchronized blocks only on individual buckets
- No locking for read operations (volatile reads)

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic operations
map.putIfAbsent("key", 1);
map.computeIfAbsent("key", k -> expensiveComputation());
map.merge("key", 1, Integer::sum);  // Atomic increment

// Safe iteration (weakly consistent)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    // Won't throw ConcurrentModificationException
}
```

**Comparison:**
| Feature | HashMap + synchronized | ConcurrentHashMap |
|---------|------------------------|-------------------|
| Lock granularity | Entire map | Per-bucket |
| Read performance | Blocked | Lock-free |
| Null keys/values | Allowed | NOT allowed |
| Atomic operations | No | Yes |

<a id="q9"></a>
### Q9: How does LinkedList work internally?
**Answer:**
LinkedList is a doubly-linked list implementation.

**Structure:**
```java
public class LinkedList<E> {
    transient Node<E> first;  // Head pointer
    transient Node<E> last;   // Tail pointer
    transient int size;
    
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
    }
}
```

**Visual representation:**
```
first                                    last
  │                                       │
  ▼                                       ▼
┌────────┐    ┌────────┐    ┌────────┐
│ null ← │ ←─ │   ←    │ ←─ │   ←    │
│ item   │    │ item   │    │ item   │
│   → ───│ ─► │   →  ──│ ─► │ → null │
└────────┘    └────────┘    └────────┘
```

**Time complexities:**
| Operation | Time |
|-----------|------|
| addFirst() / addLast() | O(1) |
| get(index) | O(n) |
| add(index, element) | O(n) |
| removeFirst() / removeLast() | O(1) |

<a id="q10"></a>
### Q10: How does TreeMap work internally?
**Answer:**
TreeMap is based on a Red-Black Tree, maintaining sorted order of keys.

**Structure:**
```java
public class TreeMap<K,V> {
    private transient Entry<K,V> root;
    private final Comparator<? super K> comparator;
    
    static final class Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left, right, parent;
        boolean color;  // RED or BLACK
    }
}
```

**Time complexities:** All O(log n) - get(), put(), remove(), containsKey()

**Comparison with HashMap:**
| Feature | HashMap | TreeMap |
|---------|---------|---------|
| Order | No order | Sorted by keys |
| Performance | O(1) average | O(log n) |
| Null keys | One allowed | Not allowed |
| Interface | Map | NavigableMap, SortedMap |

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(3, "three");
map.put(1, "one");
map.put(2, "two");

// Navigation methods
map.firstKey();           // 1
map.lastKey();            // 3
map.lowerKey(2);          // 1 (strictly less)
map.subMap(1, 3);         // {1=one, 2=two}
```

---

[← Back to Java Index](README.md)
