# Python Advanced Interview Questions

## Table of Contents

### Object-Oriented Programming
- [Q1: Explain the four pillars of OOP in Python](#q1)
- [Q2: What is the difference between `__init__` and `__new__`?](#q2)
- [Q3: What are class methods and static methods?](#q3)
- [Q4: Explain inheritance and Method Resolution Order (MRO)](#q4)
- [Q5: What are Python's magic/dunder methods?](#q5)

### Decorators & Generators
- [Q6: What are decorators and how do they work?](#q6)
- [Q7: How do you create a decorator with arguments?](#q7)
- [Q8: What are generators and how do they differ from iterators?](#q8)
- [Q9: Explain `yield` vs `return`](#q9)
- [Q10: What are generator expressions?](#q10)

### Advanced Concepts
- [Q11: What are context managers and the `with` statement?](#q11)
- [Q12: Explain Python's Global Interpreter Lock (GIL)](#q12)
- [Q13: What is the difference between shallow copy and deep copy?](#q13)
- [Q14: How does Python memory management work?](#q14)
- [Q15: What are metaclasses in Python?](#q15)

---

## Object-Oriented Programming

<a id="q1"></a>
### Q1: Explain the four pillars of OOP in Python
**Answer:**

| Pillar | Description |
|--------|-------------|
| Encapsulation | Bundling data and methods, hiding internal details |
| Abstraction | Hiding complexity, showing only essential features |
| Inheritance | Creating new classes based on existing classes |
| Polymorphism | Same interface for different underlying forms |

```python
from abc import ABC, abstractmethod

# 1. ENCAPSULATION - bundling data and methods
class BankAccount:
    def __init__(self, balance):
        self.__balance = balance  # Private attribute (name mangling)
    
    def deposit(self, amount):
        if amount > 0:
            self.__balance += amount
    
    def get_balance(self):
        return self.__balance

account = BankAccount(100)
# account.__balance  # AttributeError
account.deposit(50)
print(account.get_balance())  # 150

# 2. ABSTRACTION - hiding complexity via abstract base classes
class Shape(ABC):
    @abstractmethod
    def area(self):
        pass
    
    @abstractmethod
    def perimeter(self):
        pass

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height
    
    def area(self):
        return self.width * self.height
    
    def perimeter(self):
        return 2 * (self.width + self.height)

# shape = Shape()  # TypeError: Can't instantiate abstract class

# 3. INHERITANCE - creating classes from existing classes
class Animal:
    def __init__(self, name):
        self.name = name
    
    def speak(self):
        raise NotImplementedError

class Dog(Animal):
    def speak(self):
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self):
        return f"{self.name} says Meow!"

# 4. POLYMORPHISM - same interface, different implementations
def animal_sound(animal: Animal):
    print(animal.speak())

dog = Dog("Buddy")
cat = Cat("Whiskers")

animal_sound(dog)  # Buddy says Woof!
animal_sound(cat)  # Whiskers says Meow!

# Duck typing - Python's form of polymorphism
class Robot:
    def speak(self):
        return "Beep boop!"

robot = Robot()
animal_sound(robot)  # Beep boop! - Works because it has speak()
```

<a id="q2"></a>
### Q2: What is the difference between `__init__` and `__new__`?
**Answer:**

| Method | Purpose | When called | Returns |
|--------|---------|-------------|---------|
| `__new__` | Creates the instance | Before `__init__` | Instance |
| `__init__` | Initializes the instance | After `__new__` | None |

```python
class MyClass:
    def __new__(cls, *args, **kwargs):
        print(f"__new__ called with cls={cls}")
        instance = super().__new__(cls)
        return instance
    
    def __init__(self, value):
        print(f"__init__ called with value={value}")
        self.value = value

obj = MyClass(42)
# Output:
# __new__ called with cls=<class '__main__.MyClass'>
# __init__ called with value=42

# Use case 1: Singleton pattern
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True

# Use case 2: Immutable types (can't use __init__)
class ImmutablePoint(tuple):
    def __new__(cls, x, y):
        return super().__new__(cls, (x, y))
    
    @property
    def x(self):
        return self[0]
    
    @property
    def y(self):
        return self[1]

p = ImmutablePoint(3, 4)
print(p.x, p.y)  # 3 4

# Use case 3: Controlling instance creation
class LimitedInstances:
    _instances = []
    _max_instances = 3
    
    def __new__(cls):
        if len(cls._instances) >= cls._max_instances:
            raise RuntimeError("Maximum instances reached")
        instance = super().__new__(cls)
        cls._instances.append(instance)
        return instance
```

<a id="q3"></a>
### Q3: What are class methods and static methods?
**Answer:**

| Type | Decorator | First param | Access | Use case |
|------|-----------|-------------|--------|----------|
| Instance method | None | `self` | Instance & class | Normal methods |
| Class method | `@classmethod` | `cls` | Class only | Factory methods, alternative constructors |
| Static method | `@staticmethod` | None | Neither | Utility functions |

```python
class Person:
    species = "Human"  # Class attribute
    
    def __init__(self, name, age):
        self.name = name  # Instance attribute
        self.age = age
    
    # Instance method - has access to self
    def greet(self):
        return f"Hi, I'm {self.name}"
    
    # Class method - has access to cls
    @classmethod
    def from_birth_year(cls, name, birth_year):
        """Alternative constructor"""
        from datetime import date
        age = date.today().year - birth_year
        return cls(name, age)
    
    @classmethod
    def get_species(cls):
        return cls.species
    
    # Static method - no access to self or cls
    @staticmethod
    def is_adult(age):
        """Utility function"""
        return age >= 18

# Instance method
person = Person("Alice", 30)
print(person.greet())  # Hi, I'm Alice

# Class method - alternative constructor
person2 = Person.from_birth_year("Bob", 1990)
print(person2.age)  # 35 (or current year - 1990)

# Class method - accessing class attributes
print(Person.get_species())  # Human

# Static method - utility function
print(Person.is_adult(20))  # True
print(person.is_adult(15))  # False (can also call on instance)

# Inheritance behavior
class Employee(Person):
    species = "Employee"

# Class method respects inheritance
emp = Employee.from_birth_year("Charlie", 1995)
print(type(emp))  # <class 'Employee'>
print(Employee.get_species())  # Employee
```

<a id="q4"></a>
### Q4: Explain inheritance and Method Resolution Order (MRO)
**Answer:**

MRO determines the order in which base classes are searched when looking for a method. Python uses **C3 Linearization** algorithm.

```python
# Single inheritance
class Animal:
    def speak(self):
        return "Some sound"

class Dog(Animal):
    def speak(self):
        return "Woof!"

# Multiple inheritance
class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# Check MRO
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

print(D.mro())  # Same as above

d = D()
print(d.method())  # "B" - follows MRO

# The Diamond Problem
class A:
    def __init__(self):
        print("A.__init__")

class B(A):
    def __init__(self):
        print("B.__init__")
        super().__init__()

class C(A):
    def __init__(self):
        print("C.__init__")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D.__init__")
        super().__init__()

d = D()
# Output:
# D.__init__
# B.__init__
# C.__init__
# A.__init__

# super() follows MRO correctly - A.__init__ called only once!

# Using super() properly
class Parent:
    def __init__(self, name):
        self.name = name

class Child(Parent):
    def __init__(self, name, age):
        super().__init__(name)  # Call parent's __init__
        self.age = age

# Mixin pattern
class JSONMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class Person(JSONMixin):
    def __init__(self, name, age):
        self.name = name
        self.age = age

p = Person("Alice", 30)
print(p.to_json())  # {"name": "Alice", "age": 30}
```

<a id="q5"></a>
### Q5: What are Python's magic/dunder methods?
**Answer:**

Magic methods (dunder = double underscore) allow customization of class behavior.

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    # String representation
    def __repr__(self):
        """Developer-friendly representation"""
        return f"Vector({self.x}, {self.y})"
    
    def __str__(self):
        """User-friendly representation"""
        return f"({self.x}, {self.y})"
    
    # Comparison operators
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    
    def __lt__(self, other):
        return (self.x ** 2 + self.y ** 2) < (other.x ** 2 + other.y ** 2)
    
    # Arithmetic operators
    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)
    
    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
    
    def __rmul__(self, scalar):
        return self.__mul__(scalar)
    
    # Container methods
    def __len__(self):
        return 2
    
    def __getitem__(self, index):
        if index == 0:
            return self.x
        elif index == 1:
            return self.y
        raise IndexError("Index out of range")
    
    # Boolean
    def __bool__(self):
        return self.x != 0 or self.y != 0
    
    # Callable
    def __call__(self, scale=1):
        return Vector(self.x * scale, self.y * scale)
    
    # Context manager
    def __enter__(self):
        print("Entering context")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Exiting context")
        return False

# Usage examples
v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(repr(v1))      # Vector(3, 4)
print(str(v1))       # (3, 4)
print(v1 == v2)      # False
print(v1 + v2)       # (4, 6)
print(v1 * 2)        # (6, 8)
print(2 * v1)        # (6, 8) - uses __rmul__
print(len(v1))       # 2
print(v1[0])         # 3
print(bool(v1))      # True
print(v1(2))         # (6, 8) - callable

# Common magic methods table
"""
| Category | Methods |
|----------|---------|
| Creation | __new__, __init__, __del__ |
| String | __str__, __repr__, __format__ |
| Comparison | __eq__, __ne__, __lt__, __le__, __gt__, __ge__ |
| Arithmetic | __add__, __sub__, __mul__, __truediv__, __floordiv__, __mod__, __pow__ |
| Unary | __neg__, __pos__, __abs__, __invert__ |
| Container | __len__, __getitem__, __setitem__, __delitem__, __contains__, __iter__ |
| Callable | __call__ |
| Context | __enter__, __exit__ |
| Attribute | __getattr__, __setattr__, __delattr__, __getattribute__ |
| Hashing | __hash__ |
"""
```

---

## Decorators & Generators

<a id="q6"></a>
### Q6: What are decorators and how do they work?
**Answer:**

Decorators are functions that modify the behavior of other functions or classes.

```python
import functools
import time

# Basic decorator
def my_decorator(func):
    @functools.wraps(func)  # Preserves function metadata
    def wrapper(*args, **kwargs):
        print("Before function call")
        result = func(*args, **kwargs)
        print("After function call")
        return result
    return wrapper

@my_decorator
def say_hello(name):
    """Greet someone"""
    print(f"Hello, {name}!")

say_hello("Alice")
# Before function call
# Hello, Alice!
# After function call

# Without @syntax (equivalent)
# say_hello = my_decorator(say_hello)

# Practical decorators

# 1. Timing decorator
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

# 2. Logging decorator
def log_calls(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__} with args={args}, kwargs={kwargs}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

# 3. Retry decorator
def retry(max_attempts=3):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}")
        return wrapper
    return decorator

# 4. Caching decorator (memoization)
def memoize(func):
    cache = {}
    @functools.wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Stacking decorators
@timer
@log_calls
def process_data(data):
    return data.upper()

# Equivalent to: timer(log_calls(process_data))
```

<a id="q7"></a>
### Q7: How do you create a decorator with arguments?
**Answer:**

Decorators with arguments require an extra level of nesting.

```python
import functools

# Decorator with arguments
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")
# Hello, Alice!
# Hello, Alice!
# Hello, Alice!

# Decorator with optional arguments
def log(func=None, *, level="INFO"):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print(f"[{level}] Calling {func.__name__}")
            return func(*args, **kwargs)
        return wrapper
    
    if func is not None:
        return decorator(func)
    return decorator

# Both work:
@log
def func1():
    pass

@log(level="DEBUG")
def func2():
    pass

# Class-based decorator with arguments
class RateLimit:
    def __init__(self, max_calls, period):
        self.max_calls = max_calls
        self.period = period
        self.calls = []
    
    def __call__(self, func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            import time
            now = time.time()
            self.calls = [c for c in self.calls if now - c < self.period]
            if len(self.calls) >= self.max_calls:
                raise Exception("Rate limit exceeded")
            self.calls.append(now)
            return func(*args, **kwargs)
        return wrapper

@RateLimit(max_calls=5, period=60)
def api_call():
    print("API called")

# Decorator for methods with 'self'
def validate_positive(func):
    @functools.wraps(func)
    def wrapper(self, value):
        if value < 0:
            raise ValueError("Value must be positive")
        return func(self, value)
    return wrapper

class Account:
    def __init__(self):
        self.balance = 0
    
    @validate_positive
    def deposit(self, amount):
        self.balance += amount
```

<a id="q8"></a>
### Q8: What are generators and how do they differ from iterators?
**Answer:**

| Feature | Iterator | Generator |
|---------|----------|-----------|
| Creation | Class with `__iter__` and `__next__` | Function with `yield` |
| Memory | Can store all values | Generates values lazily |
| Reusability | Can be reused if `__iter__` returns self copy | Cannot be reused |
| Complexity | More boilerplate | Simpler syntax |

```python
# Iterator class
class CountDown:
    def __init__(self, start):
        self.start = start
    
    def __iter__(self):
        return self
    
    def __next__(self):
        if self.start <= 0:
            raise StopIteration
        self.start -= 1
        return self.start + 1

# Generator function (simpler!)
def countdown(start):
    while start > 0:
        yield start
        start -= 1

# Usage is the same
for i in countdown(5):
    print(i)  # 5, 4, 3, 2, 1

# Generator is an iterator
gen = countdown(3)
print(next(gen))  # 3
print(next(gen))  # 2
print(next(gen))  # 1
# print(next(gen))  # StopIteration

# Memory efficiency
import sys

# List stores all values
numbers_list = [x ** 2 for x in range(1000000)]
print(sys.getsizeof(numbers_list))  # ~8MB

# Generator creates values on demand
numbers_gen = (x ** 2 for x in range(1000000))
print(sys.getsizeof(numbers_gen))  # ~120 bytes

# Infinite generator
def infinite_counter():
    n = 0
    while True:
        yield n
        n += 1

counter = infinite_counter()
print(next(counter))  # 0
print(next(counter))  # 1
# Can go forever...

# Generator for large files
def read_large_file(file_path):
    with open(file_path, 'r') as f:
        for line in f:
            yield line.strip()

# Processes one line at a time, never loads entire file
# for line in read_large_file("huge_file.txt"):
#     process(line)
```

<a id="q9"></a>
### Q9: Explain `yield` vs `return`
**Answer:**

| Feature | `return` | `yield` |
|---------|----------|---------|
| Behavior | Exits function, returns value | Pauses function, returns value |
| State | Lost after return | Preserved between calls |
| Result | Single value | Generator object |
| Multiple calls | Restart from beginning | Continue from last yield |

```python
# return - exits function
def get_squares_return(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result  # Returns entire list

# yield - pauses and resumes
def get_squares_yield(n):
    for i in range(n):
        yield i ** 2  # Yields one value at a time

# yield preserves state
def stateful_generator():
    print("Start")
    yield 1
    print("After first yield")
    yield 2
    print("After second yield")
    yield 3
    print("End")

gen = stateful_generator()
print(next(gen))
# Start
# 1
print(next(gen))
# After first yield
# 2

# yield from - delegate to another generator
def generator1():
    yield 1
    yield 2

def generator2():
    yield from generator1()  # Delegate
    yield 3
    yield 4

print(list(generator2()))  # [1, 2, 3, 4]

# Two-way communication with send()
def accumulator():
    total = 0
    while True:
        value = yield total
        if value is not None:
            total += value

acc = accumulator()
next(acc)        # Initialize (returns 0)
print(acc.send(10))  # 10
print(acc.send(5))   # 15
print(acc.send(3))   # 18

# Generator with return value (Python 3.3+)
def gen_with_return():
    yield 1
    yield 2
    return "Done"

g = gen_with_return()
print(next(g))  # 1
print(next(g))  # 2
try:
    next(g)
except StopIteration as e:
    print(e.value)  # "Done"
```

<a id="q10"></a>
### Q10: What are generator expressions?
**Answer:**

Generator expressions are a concise way to create generators, similar to list comprehensions but with parentheses.

```python
# List comprehension - creates list in memory
squares_list = [x ** 2 for x in range(10)]

# Generator expression - creates generator
squares_gen = (x ** 2 for x in range(10))

print(type(squares_list))  # <class 'list'>
print(type(squares_gen))   # <class 'generator'>

# Memory comparison
import sys
print(sys.getsizeof([x ** 2 for x in range(1000)]))  # ~9000 bytes
print(sys.getsizeof((x ** 2 for x in range(1000))))  # ~120 bytes

# Usage
for square in squares_gen:
    print(square)

# Generator expression in function calls
# Parentheses can be omitted when only argument
total = sum(x ** 2 for x in range(10))
print(total)  # 285

# With condition
evens = (x for x in range(20) if x % 2 == 0)

# Multiple loops
pairs = ((x, y) for x in range(3) for y in range(3))

# Chaining generators
numbers = range(10)
squared = (x ** 2 for x in numbers)
filtered = (x for x in squared if x > 20)
print(list(filtered))  # [25, 36, 49, 64, 81]

# Practical examples

# Process large file
lines = (line.strip() for line in open('file.txt'))
non_empty = (line for line in lines if line)

# Any/all with generator (short-circuit)
has_even = any(x % 2 == 0 for x in [1, 3, 4, 7])  # True, stops at 4
all_positive = all(x > 0 for x in [1, 2, -3, 4])  # False, stops at -3

# When to use what:
# - List comprehension: need to iterate multiple times, need indexing
# - Generator expression: single iteration, memory efficiency
```

---

## Advanced Concepts

<a id="q11"></a>
### Q11: What are context managers and the `with` statement?
**Answer:**

Context managers handle setup and cleanup operations automatically, ensuring resources are properly managed.

```python
# Using with statement
with open('file.txt', 'w') as f:
    f.write('Hello, World!')
# File is automatically closed, even if exception occurs

# Class-based context manager
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None
    
    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.file:
            self.file.close()
        # Return True to suppress exception, False to propagate
        return False

with FileManager('test.txt', 'w') as f:
    f.write('Hello!')

# Using contextlib
from contextlib import contextmanager

@contextmanager
def file_manager(filename, mode):
    f = None
    try:
        f = open(filename, mode)
        yield f  # This is what 'as' receives
    finally:
        if f:
            f.close()

# Timer context manager
import time

@contextmanager
def timer(label):
    start = time.time()
    try:
        yield
    finally:
        end = time.time()
        print(f"{label}: {end - start:.4f}s")

with timer("Operation"):
    time.sleep(1)
# Operation: 1.0012s

# Database transaction context manager
@contextmanager
def transaction(connection):
    try:
        yield connection
        connection.commit()
    except Exception:
        connection.rollback()
        raise

# Multiple context managers
with open('input.txt') as infile, open('output.txt', 'w') as outfile:
    outfile.write(infile.read())

# Suppress exceptions
from contextlib import suppress

with suppress(FileNotFoundError):
    os.remove('nonexistent.txt')
# No error raised

# Reusable context manager
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in ['a.txt', 'b.txt']]
    # All files closed when block exits
```

<a id="q12"></a>
### Q12: Explain Python's Global Interpreter Lock (GIL)
**Answer:**

The GIL is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes simultaneously.

```python
import threading
import multiprocessing
import time

# GIL Impact demonstration
counter = 0

def increment(n):
    global counter
    for _ in range(n):
        counter += 1

# Threading (affected by GIL)
def test_threading():
    global counter
    counter = 0
    threads = [threading.Thread(target=increment, args=(1000000,)) for _ in range(2)]
    
    start = time.time()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    
    print(f"Threading: counter={counter}, time={time.time() - start:.2f}s")
    # Counter might not be 2000000 due to race conditions!

# Multiprocessing (bypasses GIL)
def test_multiprocessing():
    start = time.time()
    processes = [multiprocessing.Process(target=increment, args=(1000000,)) for _ in range(2)]
    
    for p in processes:
        p.start()
    for p in processes:
        p.join()
    
    print(f"Multiprocessing: time={time.time() - start:.2f}s")

# When GIL is released:
# 1. I/O operations (file, network)
# 2. time.sleep()
# 3. Some C extensions (NumPy)

# I/O-bound task (GIL released during I/O)
import urllib.request

def fetch_url(url):
    with urllib.request.urlopen(url) as response:
        return len(response.read())

# Threading works well for I/O-bound tasks
urls = ['http://example.com'] * 10

# Threaded (faster for I/O)
def threaded_fetch():
    threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

# CPU-bound task solutions:
# 1. multiprocessing - use separate processes
# 2. concurrent.futures.ProcessPoolExecutor
# 3. C extensions that release GIL
# 4. Alternative Python implementations (Jython, IronPython)

from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# For I/O-bound
with ThreadPoolExecutor(max_workers=5) as executor:
    results = executor.map(fetch_url, urls)

# For CPU-bound
def cpu_intensive(n):
    return sum(i ** 2 for i in range(n))

with ProcessPoolExecutor(max_workers=4) as executor:
    results = executor.map(cpu_intensive, [1000000] * 4)
```

<a id="q13"></a>
### Q13: What is the difference between shallow copy and deep copy?
**Answer:**

| Type | Behavior | Nested objects |
|------|----------|----------------|
| Assignment | Same object reference | Same reference |
| Shallow copy | New object, same nested references | Shared |
| Deep copy | New object, new nested objects | Independent |

```python
import copy

# Original list with nested list
original = [[1, 2, 3], [4, 5, 6]]

# Assignment - no copy, same reference
assigned = original
print(assigned is original)  # True

# Shallow copy - new list, but nested lists are shared
shallow = copy.copy(original)
# Or: shallow = original.copy()
# Or: shallow = list(original)
# Or: shallow = original[:]

print(shallow is original)       # False
print(shallow[0] is original[0]) # True - nested still shared!

# Modifying nested affects both
shallow[0][0] = 999
print(original[0][0])  # 999 - changed!

# Deep copy - completely independent
original = [[1, 2, 3], [4, 5, 6]]
deep = copy.deepcopy(original)

print(deep is original)       # False
print(deep[0] is original[0]) # False - nested also copied!

deep[0][0] = 999
print(original[0][0])  # 1 - unchanged!

# Visual representation:
"""
Assignment:     original ─────────┬───► [[1,2,3], [4,5,6]]
                assigned ─────────┘

Shallow copy:   original ─────────────► [[1,2,3], [4,5,6]]
                                           ↑         ↑
                shallow ──────────────► [ ref1  ,  ref2  ]

Deep copy:      original ─────────────► [[1,2,3], [4,5,6]]
                deep ─────────────────► [[1,2,3], [4,5,6]]
                                        (completely separate)
"""

# Dictionary copy
d = {'a': [1, 2, 3], 'b': {'x': 1}}
shallow_d = d.copy()  # or dict(d)
deep_d = copy.deepcopy(d)

# Custom class with __copy__ and __deepcopy__
class MyClass:
    def __init__(self, value, items):
        self.value = value
        self.items = items
    
    def __copy__(self):
        return MyClass(self.value, self.items)  # Shallow
    
    def __deepcopy__(self, memo):
        return MyClass(
            copy.deepcopy(self.value, memo),
            copy.deepcopy(self.items, memo)
        )
```

<a id="q14"></a>
### Q14: How does Python memory management work?
**Answer:**

Python uses a combination of reference counting and garbage collection.

```python
import sys
import gc

# Reference counting
a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (a + function argument)

b = a  # Increase reference
print(sys.getrefcount(a))  # 3

del b  # Decrease reference
print(sys.getrefcount(a))  # 2

# Memory allocation - small object pools
# Python pre-allocates small integers (-5 to 256)
a = 100
b = 100
print(a is b)  # True - same object (cached)

a = 1000
b = 1000
print(a is b)  # False - different objects

# String interning
s1 = "hello"
s2 = "hello"
print(s1 is s2)  # True - interned

s1 = "hello world!"
s2 = "hello world!"
print(s1 is s2)  # May be False - not interned (has space/special chars)

# Garbage collection for circular references
class Node:
    def __init__(self):
        self.ref = None

# Create circular reference
a = Node()
b = Node()
a.ref = b
b.ref = a

# Reference counting can't handle this
del a, b
# Objects still exist due to circular reference

# Garbage collector handles it
gc.collect()  # Force garbage collection

# Weak references - don't prevent garbage collection
import weakref

class MyClass:
    pass

obj = MyClass()
weak_ref = weakref.ref(obj)

print(weak_ref())  # <MyClass object>
del obj
print(weak_ref())  # None - object was garbage collected

# Memory profiling
# pip install memory_profiler
# @profile decorator shows line-by-line memory usage

# __slots__ for memory optimization
class WithoutSlots:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ['x', 'y']
    def __init__(self, x, y):
        self.x = x
        self.y = y

# WithSlots uses less memory (no __dict__)
print(sys.getsizeof(WithoutSlots(1, 2).__dict__))  # ~104 bytes
# WithSlots has no __dict__

# Generators for memory efficiency
# List - stores all values
squares_list = [x ** 2 for x in range(1000000)]  # ~8MB

# Generator - computes on demand
squares_gen = (x ** 2 for x in range(1000000))  # ~120 bytes
```

<a id="q15"></a>
### Q15: What are metaclasses in Python?
**Answer:**

Metaclasses are "classes of classes" - they define how classes behave.

```python
# Everything is an object, including classes
class MyClass:
    pass

print(type(MyClass))      # <class 'type'>
print(type(MyClass()))    # <class 'MyClass'>

# 'type' is the default metaclass
# Class creation: type(name, bases, dict)
MyClass = type('MyClass', (), {'x': 1, 'method': lambda self: 'hello'})

# Custom metaclass
class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        # Called when class is created
        print(f"Creating class: {name}")
        # Add automatic attribute
        namespace['created_by'] = 'MyMeta'
        return super().__new__(mcs, name, bases, namespace)
    
    def __init__(cls, name, bases, namespace):
        # Called after class is created
        print(f"Initializing class: {name}")
        super().__init__(name, bases, namespace)
    
    def __call__(cls, *args, **kwargs):
        # Called when creating instance
        print(f"Creating instance of {cls.__name__}")
        return super().__call__(*args, **kwargs)

class MyClass(metaclass=MyMeta):
    pass

# Output:
# Creating class: MyClass
# Initializing class: MyClass

obj = MyClass()
# Output: Creating instance of MyClass

print(MyClass.created_by)  # MyMeta

# Practical use case: Singleton metaclass
class SingletonMeta(type):
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Singleton(metaclass=SingletonMeta):
    pass

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True

# Practical use case: Auto-register subclasses
class PluginMeta(type):
    plugins = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # Don't register base class
            mcs.plugins[name] = cls
        return cls

class Plugin(metaclass=PluginMeta):
    pass

class AudioPlugin(Plugin):
    pass

class VideoPlugin(Plugin):
    pass

print(PluginMeta.plugins)
# {'AudioPlugin': <class 'AudioPlugin'>, 'VideoPlugin': <class 'VideoPlugin'>}

# Alternative: __init_subclass__ (Python 3.6+)
class Plugin:
    plugins = {}
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.plugins[cls.__name__] = cls

class AudioPlugin(Plugin):
    pass

# Simpler than metaclass for most use cases!
```

---

[← Python Basics](python-basics.md) | [Back to Python Index](README.md) | [Django Basics →](django-basics.md)
