# Python Basics Interview Questions

## Table of Contents

### Data Types & Variables
- [Q1: What are Python's basic data types?](#q1)
- [Q2: What is the difference between mutable and immutable types?](#q2)
- [Q3: Explain the difference between list, tuple, set, and dictionary](#q3)
- [Q4: What is the difference between `is` and `==`?](#q4)
- [Q5: How does Python handle variable scope (LEGB rule)?](#q5)

### Strings & Collections
- [Q6: What are common string methods in Python?](#q6)
- [Q7: What is list comprehension and how do you use it?](#q7)
- [Q8: What is dictionary comprehension?](#q8)
- [Q9: How do you merge two dictionaries in Python?](#q9)
- [Q10: What is the difference between `append()` and `extend()`?](#q10)

### Functions
- [Q11: What are *args and **kwargs?](#q11)
- [Q12: What is the difference between a function and a method?](#q12)
- [Q13: Explain lambda functions in Python](#q13)
- [Q14: What are higher-order functions (map, filter, reduce)?](#q14)
- [Q15: What is a closure in Python?](#q15)

### Control Flow & Iteration
- [Q16: What is the difference between `range()` and `xrange()`?](#q16)
- [Q17: How do `break`, `continue`, and `pass` differ?](#q17)
- [Q18: What is the `enumerate()` function?](#q18)
- [Q19: How do you iterate over a dictionary?](#q19)
- [Q20: What is the `zip()` function?](#q20)

---

## Data Types & Variables

<a id="q1"></a>
### Q1: What are Python's basic data types?
**Answer:**

Python has several built-in data types:

| Category | Types | Examples |
|----------|-------|----------|
| Numeric | int, float, complex | `42`, `3.14`, `2+3j` |
| Sequence | str, list, tuple | `"hello"`, `[1,2,3]`, `(1,2,3)` |
| Mapping | dict | `{"key": "value"}` |
| Set | set, frozenset | `{1, 2, 3}` |
| Boolean | bool | `True`, `False` |
| None | NoneType | `None` |

```python
# Type checking
x = 42
print(type(x))  # <class 'int'>

# Type conversion
num_str = "123"
num_int = int(num_str)  # 123
num_float = float(num_str)  # 123.0

# Check if instance
isinstance(x, int)  # True
isinstance(x, (int, float))  # True - check multiple types
```

<a id="q2"></a>
### Q2: What is the difference between mutable and immutable types?
**Answer:**

**Immutable types** cannot be changed after creation. Any "modification" creates a new object.
**Mutable types** can be changed in place without creating a new object.

| Immutable | Mutable |
|-----------|---------|
| int, float, complex | list |
| str | dict |
| tuple | set |
| frozenset | bytearray |
| bool | |

```python
# Immutable example - string
s = "hello"
print(id(s))  # 140234866534256
s = s + " world"
print(id(s))  # 140234866534320 - different object!

# Mutable example - list
lst = [1, 2, 3]
print(id(lst))  # 140234866123456
lst.append(4)
print(id(lst))  # 140234866123456 - same object!

# Why it matters - function arguments
def modify_list(lst):
    lst.append(4)  # Modifies original list
    
def modify_string(s):
    s = s + " world"  # Creates new string, doesn't affect original

my_list = [1, 2, 3]
modify_list(my_list)
print(my_list)  # [1, 2, 3, 4] - modified!

my_str = "hello"
modify_string(my_str)
print(my_str)  # "hello" - unchanged!
```

<a id="q3"></a>
### Q3: Explain the difference between list, tuple, set, and dictionary
**Answer:**

| Feature | List | Tuple | Set | Dictionary |
|---------|------|-------|-----|------------|
| Syntax | `[1, 2, 3]` | `(1, 2, 3)` | `{1, 2, 3}` | `{"a": 1}` |
| Mutable | Yes | No | Yes | Yes |
| Ordered | Yes | Yes | No | Yes (3.7+) |
| Duplicates | Yes | Yes | No | Keys: No |
| Indexable | Yes | Yes | No | By key |
| Use case | General collection | Fixed data, dict keys | Unique items, membership | Key-value mapping |

```python
# List - ordered, mutable, allows duplicates
my_list = [1, 2, 2, 3]
my_list[0] = 10
my_list.append(4)

# Tuple - ordered, immutable, allows duplicates
my_tuple = (1, 2, 2, 3)
# my_tuple[0] = 10  # TypeError!
x, y, z, w = my_tuple  # Unpacking

# Set - unordered, mutable, no duplicates
my_set = {1, 2, 2, 3}  # {1, 2, 3}
my_set.add(4)
print(2 in my_set)  # O(1) lookup

# Dictionary - key-value pairs, ordered (3.7+)
my_dict = {"name": "John", "age": 30}
my_dict["email"] = "john@example.com"
```

<a id="q4"></a>
### Q4: What is the difference between `is` and `==`?
**Answer:**

- `==` compares **values** (equality)
- `is` compares **identity** (same object in memory)

```python
# Example with lists
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)  # True - same values
print(a is b)  # False - different objects
print(a is c)  # True - same object

# Integer caching (-5 to 256)
x = 100
y = 100
print(x is y)  # True - Python caches small integers

x = 1000
y = 1000
print(x is y)  # False - not cached

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True - Python interns short strings

# Best practices
# Use 'is' for None comparison
if x is None:
    pass

# Use '==' for value comparison
if x == 0:
    pass
```

<a id="q5"></a>
### Q5: How does Python handle variable scope (LEGB rule)?
**Answer:**

Python uses the **LEGB** rule for variable lookup:
1. **L**ocal - Inside the current function
2. **E**nclosing - Inside enclosing functions (closures)
3. **G**lobal - Module level
4. **B**uilt-in - Python's built-in names

```python
x = "global"  # Global scope

def outer():
    x = "enclosing"  # Enclosing scope
    
    def inner():
        x = "local"  # Local scope
        print(x)  # "local"
    
    inner()
    print(x)  # "enclosing"

outer()
print(x)  # "global"

# Using global keyword
counter = 0

def increment():
    global counter
    counter += 1

increment()
print(counter)  # 1

# Using nonlocal keyword (for enclosing scope)
def outer():
    count = 0
    
    def inner():
        nonlocal count
        count += 1
        return count
    
    return inner

counter = outer()
print(counter())  # 1
print(counter())  # 2
```

---

## Strings & Collections

<a id="q6"></a>
### Q6: What are common string methods in Python?
**Answer:**

```python
s = "  Hello, World!  "

# Case methods
s.lower()      # "  hello, world!  "
s.upper()      # "  HELLO, WORLD!  "
s.title()      # "  Hello, World!  "
s.capitalize() # "  hello, world!  "
s.swapcase()   # "  hELLO, wORLD!  "

# Whitespace methods
s.strip()      # "Hello, World!"
s.lstrip()     # "Hello, World!  "
s.rstrip()     # "  Hello, World!"

# Search methods
s.find("World")     # 9 (returns -1 if not found)
s.index("World")    # 9 (raises ValueError if not found)
s.count("l")        # 3
s.startswith("  H") # True
s.endswith("!  ")   # True

# Replace and split
s.replace("World", "Python")  # "  Hello, Python!  "
"a,b,c".split(",")            # ["a", "b", "c"]
"-".join(["a", "b", "c"])     # "a-b-c"

# Validation methods
"abc123".isalnum()   # True
"abc".isalpha()      # True
"123".isdigit()      # True
"   ".isspace()      # True

# Formatting
name = "Alice"
age = 30
f"Name: {name}, Age: {age}"           # f-string (Python 3.6+)
"Name: {}, Age: {}".format(name, age) # format method
"Name: %s, Age: %d" % (name, age)     # % formatting (legacy)
```

<a id="q7"></a>
### Q7: What is list comprehension and how do you use it?
**Answer:**

List comprehension is a concise way to create lists based on existing iterables.

**Syntax:** `[expression for item in iterable if condition]`

```python
# Basic list comprehension
squares = [x**2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With condition (filter)
even_squares = [x**2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]

# With if-else (transform)
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
# ["even", "odd", "even", "odd", "even"]

# Nested loops
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flattened = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Equivalent to:
flattened = []
for row in matrix:
    for num in row:
        flattened.append(num)

# Nested list comprehension (2D list)
matrix = [[i * j for j in range(1, 4)] for i in range(1, 4)]
# [[1, 2, 3], [2, 4, 6], [3, 6, 9]]

# When NOT to use list comprehension:
# 1. Complex logic that's hard to read
# 2. Side effects (printing, I/O)
# 3. When you don't need a list (use generator instead)
```

<a id="q8"></a>
### Q8: What is dictionary comprehension?
**Answer:**

Dictionary comprehension creates dictionaries from iterables.

**Syntax:** `{key_expr: value_expr for item in iterable if condition}`

```python
# Basic dictionary comprehension
squares = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# From two lists
keys = ["a", "b", "c"]
values = [1, 2, 3]
d = {k: v for k, v in zip(keys, values)}
# {"a": 1, "b": 2, "c": 3}

# With condition
even_squares = {x: x**2 for x in range(10) if x % 2 == 0}
# {0: 0, 2: 4, 4: 16, 6: 36, 8: 64}

# Swap keys and values
original = {"a": 1, "b": 2, "c": 3}
swapped = {v: k for k, v in original.items()}
# {1: "a", 2: "b", 3: "c"}

# Transform values
prices = {"apple": 1.0, "banana": 0.5, "orange": 0.75}
discounted = {k: v * 0.9 for k, v in prices.items()}
# {"apple": 0.9, "banana": 0.45, "orange": 0.675}

# Set comprehension (similar syntax)
unique_lengths = {len(word) for word in ["hello", "world", "python"]}
# {5, 6}
```

<a id="q9"></a>
### Q9: How do you merge two dictionaries in Python?
**Answer:**

```python
dict1 = {"a": 1, "b": 2}
dict2 = {"b": 3, "c": 4}

# Method 1: | operator (Python 3.9+) - RECOMMENDED
merged = dict1 | dict2
# {"a": 1, "b": 3, "c": 4}

# Method 2: |= operator for in-place merge (Python 3.9+)
dict1 |= dict2
# dict1 is now {"a": 1, "b": 3, "c": 4}

# Method 3: ** unpacking (Python 3.5+)
dict1 = {"a": 1, "b": 2}
merged = {**dict1, **dict2}
# {"a": 1, "b": 3, "c": 4}

# Method 4: update() method (modifies in place)
dict1 = {"a": 1, "b": 2}
dict1.update(dict2)
# dict1 is now {"a": 1, "b": 3, "c": 4}

# Method 5: dict() constructor with **unpacking
merged = dict(**dict1, **dict2)  # Note: keys must be strings

# Note: In all cases, dict2 values override dict1 for duplicate keys
```

<a id="q10"></a>
### Q10: What is the difference between `append()` and `extend()`?
**Answer:**

- `append()` adds a single element to the end of a list
- `extend()` adds all elements from an iterable to the end

```python
# append() - adds single element
lst = [1, 2, 3]
lst.append(4)
print(lst)  # [1, 2, 3, 4]

lst.append([5, 6])
print(lst)  # [1, 2, 3, 4, [5, 6]] - list added as single element!

# extend() - adds elements from iterable
lst = [1, 2, 3]
lst.extend([4, 5])
print(lst)  # [1, 2, 3, 4, 5]

lst.extend("ab")
print(lst)  # [1, 2, 3, 4, 5, "a", "b"] - string is iterable

# Using += (equivalent to extend)
lst = [1, 2, 3]
lst += [4, 5]
print(lst)  # [1, 2, 3, 4, 5]

# Performance comparison
# append() - O(1) amortized
# extend() - O(k) where k is length of iterable

# Common mistake
lst = [1, 2, 3]
result = lst.append(4)  # Returns None!
print(result)  # None
print(lst)  # [1, 2, 3, 4]
```

---

## Functions

<a id="q11"></a>
### Q11: What are *args and **kwargs?
**Answer:**

- `*args` collects positional arguments into a **tuple**
- `**kwargs` collects keyword arguments into a **dictionary**

```python
# *args - variable positional arguments
def sum_all(*args):
    print(type(args))  # <class 'tuple'>
    return sum(args)

sum_all(1, 2, 3, 4)  # 10

# **kwargs - variable keyword arguments
def print_info(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="John", age=30, city="NYC")
# name: John
# age: 30
# city: NYC

# Combined usage
def func(required, *args, **kwargs):
    print(f"Required: {required}")
    print(f"Args: {args}")
    print(f"Kwargs: {kwargs}")

func("hello", 1, 2, 3, name="John", age=30)
# Required: hello
# Args: (1, 2, 3)
# Kwargs: {"name": "John", "age": 30}

# Unpacking when calling functions
def greet(name, age, city):
    print(f"{name}, {age}, from {city}")

args = ("John", 30, "NYC")
greet(*args)  # Unpack tuple

kwargs = {"name": "John", "age": 30, "city": "NYC"}
greet(**kwargs)  # Unpack dictionary

# Order of parameters: regular, *args, keyword-only, **kwargs
def func(a, b, *args, keyword_only, **kwargs):
    pass
```

<a id="q12"></a>
### Q12: What is the difference between a function and a method?
**Answer:**

- **Function**: A standalone block of code that can be called by name
- **Method**: A function that is associated with an object/class

```python
# Function - standalone
def greet(name):
    return f"Hello, {name}"

greet("Alice")  # Called directly

# Method - bound to object
class Person:
    def __init__(self, name):
        self.name = name
    
    def greet(self):  # Instance method
        return f"Hello, {self.name}"
    
    @classmethod
    def create(cls, name):  # Class method
        return cls(name)
    
    @staticmethod
    def validate_name(name):  # Static method
        return len(name) > 0

person = Person("Alice")
person.greet()  # Called on instance

# Types of methods:
# 1. Instance method - first param is 'self'
# 2. Class method - first param is 'cls', uses @classmethod
# 3. Static method - no implicit first param, uses @staticmethod

# Built-in functions vs string methods
len("hello")        # Function
"hello".upper()     # Method

# Method is a bound function
print(type(person.greet))  # <class 'method'>
print(type(greet))         # <class 'function'>
```

<a id="q13"></a>
### Q13: Explain lambda functions in Python
**Answer:**

Lambda functions are anonymous, single-expression functions.

**Syntax:** `lambda arguments: expression`

```python
# Basic lambda
square = lambda x: x ** 2
square(5)  # 25

# Multiple arguments
add = lambda x, y: x + y
add(3, 4)  # 7

# With default arguments
greet = lambda name, greeting="Hello": f"{greeting}, {name}"
greet("Alice")  # "Hello, Alice"
greet("Alice", "Hi")  # "Hi, Alice"

# Common use cases:

# 1. Sorting with custom key
students = [("Alice", 85), ("Bob", 92), ("Charlie", 78)]
sorted(students, key=lambda x: x[1])  # Sort by grade
# [("Charlie", 78), ("Alice", 85), ("Bob", 92)]

# 2. With map()
numbers = [1, 2, 3, 4, 5]
squared = list(map(lambda x: x ** 2, numbers))
# [1, 4, 9, 16, 25]

# 3. With filter()
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4]

# 4. Inline callbacks
buttons = []
for i in range(5):
    buttons.append(lambda i=i: print(f"Button {i}"))  # Note: i=i to capture value

# Limitations:
# - Single expression only
# - No statements (no assignments, loops, etc.)
# - Less readable for complex logic

# When NOT to use lambda:
# Bad
f = lambda x, y: x if x > y else y

# Better - use def for named functions
def maximum(x, y):
    return x if x > y else y
```

<a id="q14"></a>
### Q14: What are higher-order functions (map, filter, reduce)?
**Answer:**

Higher-order functions take functions as arguments or return functions.

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# map() - apply function to each element
squared = list(map(lambda x: x ** 2, numbers))
# [1, 4, 9, 16, 25]

# Equivalent list comprehension (often preferred)
squared = [x ** 2 for x in numbers]

# filter() - keep elements where function returns True
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4]

# Equivalent list comprehension
evens = [x for x in numbers if x % 2 == 0]

# reduce() - accumulate values using function
total = reduce(lambda acc, x: acc + x, numbers)
# 15 (1+2+3+4+5)

# reduce with initial value
total = reduce(lambda acc, x: acc + x, numbers, 10)
# 25 (10+1+2+3+4+5)

# Practical examples:
words = ["hello", "world", "python"]

# map - transform
lengths = list(map(len, words))  # [5, 5, 6]
upper = list(map(str.upper, words))  # ["HELLO", "WORLD", "PYTHON"]

# filter - select
long_words = list(filter(lambda w: len(w) > 5, words))  # ["python"]

# reduce - aggregate
longest = reduce(lambda a, b: a if len(a) > len(b) else b, words)
# "python"

# Chaining
result = reduce(
    lambda acc, x: acc + x,
    map(lambda x: x ** 2,
        filter(lambda x: x % 2 == 0, numbers))
)
# 20 (4 + 16)

# More readable with list comprehension
result = sum(x ** 2 for x in numbers if x % 2 == 0)
```

<a id="q15"></a>
### Q15: What is a closure in Python?
**Answer:**

A closure is a function that remembers values from its enclosing scope even after that scope has finished executing.

```python
# Basic closure
def outer(x):
    def inner(y):
        return x + y  # 'x' is from enclosing scope
    return inner

add_5 = outer(5)
add_10 = outer(10)

print(add_5(3))   # 8
print(add_10(3))  # 13

# Closure maintains state
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    return increment

c1 = counter()
c2 = counter()

print(c1())  # 1
print(c1())  # 2
print(c2())  # 1 - independent counter

# Practical example: multiplier factory
def make_multiplier(n):
    def multiplier(x):
        return x * n
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15

# Accessing closure variables
print(double.__closure__)  # (<cell at 0x...>,)
print(double.__closure__[0].cell_contents)  # 2

# Common gotcha with loops
functions = []
for i in range(3):
    functions.append(lambda: i)  # All will return 2!

for f in functions:
    print(f())  # 2, 2, 2 - not 0, 1, 2!

# Fix: capture value with default argument
functions = []
for i in range(3):
    functions.append(lambda i=i: i)

for f in functions:
    print(f())  # 0, 1, 2
```

---

## Control Flow & Iteration

<a id="q16"></a>
### Q16: What is the difference between `range()` and `xrange()`?
**Answer:**

In **Python 3**, `xrange()` doesn't exist - `range()` behaves like Python 2's `xrange()`.

| Feature | Python 2 `range()` | Python 2 `xrange()` / Python 3 `range()` |
|---------|-------------------|------------------------------------------|
| Returns | List | Range object (iterator) |
| Memory | Creates full list in memory | Generates values on demand |
| Reusable | Yes | Yes (range object), No (iterator) |

```python
# Python 3 range() returns a range object
r = range(1000000)
print(type(r))  # <class 'range'>
print(r)        # range(0, 1000000)

# Memory efficient - doesn't create list
import sys
print(sys.getsizeof(range(1000000)))  # 48 bytes
print(sys.getsizeof(list(range(1000000))))  # ~8MB

# range features
r = range(0, 10, 2)  # start, stop, step
print(list(r))  # [0, 2, 4, 6, 8]

# Supports indexing and slicing
print(r[2])     # 4
print(r[-1])    # 8
print(r[1:3])   # range(2, 6, 2)

# Supports membership testing (O(1))
print(5 in range(10))   # True
print(5 in range(0, 10, 2))  # False

# Reverse iteration
for i in range(5, 0, -1):
    print(i)  # 5, 4, 3, 2, 1

# If you need a list, convert explicitly
numbers = list(range(10))
```

<a id="q17"></a>
### Q17: How do `break`, `continue`, and `pass` differ?
**Answer:**

| Statement | Purpose |
|-----------|---------|
| `break` | Exit the entire loop |
| `continue` | Skip to next iteration |
| `pass` | Do nothing (placeholder) |

```python
# break - exit loop entirely
for i in range(10):
    if i == 5:
        break
    print(i)
# Output: 0, 1, 2, 3, 4

# continue - skip current iteration
for i in range(10):
    if i % 2 == 0:
        continue
    print(i)
# Output: 1, 3, 5, 7, 9

# pass - do nothing (placeholder)
for i in range(10):
    if i % 2 == 0:
        pass  # TODO: implement later
    else:
        print(i)
# Output: 1, 3, 5, 7, 9

# pass in class/function definition
class MyClass:
    pass  # Empty class placeholder

def my_function():
    pass  # Empty function placeholder

# break with else clause
for i in range(5):
    if i == 10:  # Never true
        break
else:
    print("Loop completed without break")
# Output: "Loop completed without break"

# Nested loops - break only exits innermost
for i in range(3):
    for j in range(3):
        if j == 1:
            break
        print(f"i={i}, j={j}")
# Output: i=0,j=0 / i=1,j=0 / i=2,j=0
```

<a id="q18"></a>
### Q18: What is the `enumerate()` function?
**Answer:**

`enumerate()` adds a counter to an iterable, returning (index, value) pairs.

```python
# Basic usage
fruits = ["apple", "banana", "cherry"]

for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")
# 0: apple
# 1: banana
# 2: cherry

# Custom start index
for index, fruit in enumerate(fruits, start=1):
    print(f"{index}: {fruit}")
# 1: apple
# 2: banana
# 3: cherry

# Without enumerate (less Pythonic)
for i in range(len(fruits)):
    print(f"{i}: {fruits[i]}")

# enumerate returns an enumerate object
e = enumerate(fruits)
print(type(e))  # <class 'enumerate'>
print(list(e))  # [(0, 'apple'), (1, 'banana'), (2, 'cherry')]

# Practical examples:

# Finding index of specific item
for i, fruit in enumerate(fruits):
    if fruit == "banana":
        print(f"Found at index {i}")
        break

# Creating dictionary from list
fruit_dict = {i: fruit for i, fruit in enumerate(fruits)}
# {0: 'apple', 1: 'banana', 2: 'cherry'}

# Processing with index tracking
lines = ["first", "second", "third"]
for line_no, line in enumerate(lines, start=1):
    print(f"Line {line_no}: {line}")
```

<a id="q19"></a>
### Q19: How do you iterate over a dictionary?
**Answer:**

```python
d = {"a": 1, "b": 2, "c": 3}

# Iterate over keys (default)
for key in d:
    print(key)  # a, b, c

# Explicitly iterate over keys
for key in d.keys():
    print(key)  # a, b, c

# Iterate over values
for value in d.values():
    print(value)  # 1, 2, 3

# Iterate over key-value pairs
for key, value in d.items():
    print(f"{key}: {value}")
# a: 1
# b: 2
# c: 3

# With enumerate
for i, (key, value) in enumerate(d.items()):
    print(f"{i}: {key}={value}")

# Dictionary comprehension during iteration
squared = {k: v ** 2 for k, v in d.items()}
# {"a": 1, "b": 4, "c": 9}

# Safe deletion during iteration (create copy)
for key in list(d.keys()):  # list() creates a copy
    if d[key] < 2:
        del d[key]

# DON'T modify during iteration without copy
# for key in d:  # RuntimeError!
#     del d[key]

# Sorted iteration
for key in sorted(d.keys()):
    print(f"{key}: {d[key]}")

# Reverse iteration (Python 3.8+)
for key in reversed(d):
    print(key)
```

<a id="q20"></a>
### Q20: What is the `zip()` function?
**Answer:**

`zip()` combines multiple iterables element-wise into tuples.

```python
# Basic usage
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]

for name, age in zip(names, ages):
    print(f"{name} is {age}")
# Alice is 25
# Bob is 30
# Charlie is 35

# zip returns an iterator
z = zip(names, ages)
print(type(z))  # <class 'zip'>
print(list(z))  # [('Alice', 25), ('Bob', 30), ('Charlie', 35)]

# Unequal lengths - stops at shortest
names = ["Alice", "Bob"]
ages = [25, 30, 35]
print(list(zip(names, ages)))  # [('Alice', 25), ('Bob', 30)]

# Use zip_longest for unequal lengths
from itertools import zip_longest
print(list(zip_longest(names, ages, fillvalue=None)))
# [('Alice', 25), ('Bob', 30), (None, 35)]

# Multiple iterables
cities = ["NYC", "LA", "Chicago"]
for name, age, city in zip(names, ages, cities):
    print(f"{name}, {age}, {city}")

# Creating dictionary from two lists
d = dict(zip(names, ages))
# {"Alice": 25, "Bob": 30}

# Unzip (transpose)
pairs = [("a", 1), ("b", 2), ("c", 3)]
letters, numbers = zip(*pairs)
print(letters)  # ('a', 'b', 'c')
print(numbers)  # (1, 2, 3)

# Transpose matrix
matrix = [[1, 2, 3], [4, 5, 6]]
transposed = list(zip(*matrix))
# [(1, 4), (2, 5), (3, 6)]

# Parallel iteration with index
for i, (name, age) in enumerate(zip(names, ages)):
    print(f"{i}: {name} is {age}")
```

---

[← Back to Python Index](README.md) | [Python Advanced →](python-advanced.md)
