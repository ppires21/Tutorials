# Python Fundamentals — Coding guide

> A practical guide that teaches you **how to program in Python** through small, focused examples. 
> No origin story, no history. Just **syntax, patterns, and best practices** that you can copy, adapt, and run.

---

## 0) How to run examples

```bash
python --version          # see Python version
python file.py            # run a script
python -i file.py         # run, then drop into REPL with its variables
python -m venv .venv      # create virtual env
source .venv/bin/activate # activate (macOS/Linux)
pip install <pkg>         # install packages
```
**Explanation**
- `python file.py` runs files; REPL (`python`) is great for quick checks.
- Virtual environments isolate dependencies; always use one per project.

---

## 1) Variables, Types, Strings, Numbers

```python
x = 10           # int
y = 3.14         # float
name = "Ada"     # str
is_ok = True     # bool
n = None         # absence of a value

# type conversion
count = int("42")
pi_text = str(3.1415)

# f-strings (formatted strings)
msg = f"Hello {name}, count={count}, pi≈{y:.2f}"
print(msg)

# common string ops
s = "  data,science  "
print(s.strip())          # remove outer spaces
print(s.upper())          # 'DATA,SCIENCE'
print("data" in s)        # membership: True/False
print(",".join(["a","b","c"]))  # 'a,b,c'
```
**Key points**
- Python is dynamically typed but values still have types.
- `f"{var}"` is the idiomatic way to format strings.
- Truthiness: `0, "", [], None` are treated as False in conditionals.

---

## 2) Collections — list, tuple, set, dict

```python
# list: ordered, mutable
nums = [1, 2, 3]
nums.append(4)
nums[0] = 99
print(nums)          # [99, 2, 3, 4]
print(nums[-1])      # last element: 4
print(nums[1:3])     # slicing: [2,3]

# tuple: ordered, immutable
point = (10, 20)

# set: unique items, no order
colors = {"red", "green", "red"}
print(colors)        # {'red', 'green'}

# dict: key-value map
person = {"name": "Ada", "age": 36}
print(person["name"])               # 'Ada'
person["job"] = "Engineer"          # add
print(person.get("missing", "N/A")) # default if key absent
```
**Comprehensions**
```python
squares = [n*n for n in range(6)]                 # list comp
evens   = {n for n in range(10) if n % 2 == 0}    # set comp
index   = {name: i for i, name in enumerate(["a","b","c"])}
```
**Copy pitfalls (mutability)**
```python
a = [1, 2, 3]
b = a                 # same list object!
a.append(4)
print(b)              # [1,2,3,4]

c = a[:]              # shallow copy
import copy
d = copy.deepcopy(a)  # deep copy (for nested structures)
```

---

## 3) The `collections` Module (super useful)

```python
from collections import Counter, defaultdict, deque, namedtuple

Counter("banana")          # counts chars: {'b':1,'a':3,'n':2}

dd = defaultdict(int)      # default value for missing keys
dd["seen"] += 1            # becomes 1, no KeyError

q = deque([1,2,3])
q.appendleft(0); q.append(4); q.pop()

Point = namedtuple("Point", "x y")
p = Point(10, 20)          # p.x == 10, p.y == 20
```
**When to use**
- `Counter` for frequencies; `defaultdict` to avoid `KeyError`.
- `deque` for fast append/pop from both ends.
- `namedtuple` for small immutable records.

---

## 4) Operators & Expressions (quick tour)

```python
# arithmetic: + - * / // % **
7 // 2     # floor division -> 3
7 % 2      # modulo -> 1
2 ** 10    # power -> 1024

# comparison: == != < <= > >=
# boolean: and, or, not
# identity: "is" (same object?), membership: "in"
```
**Short-circuit**
```python
def ok(): print("ok"); return True
def ko(): print("ko"); return False
print(ko() and ok())   # 'ko' then stops (False)
print(ok() or ko())    # 'ok' then stops (True)
```

---

## 5) Control Flow — `if/elif/else`, loops, `range/zip/enumerate`

```python
x = 7
if x > 10:
    print("big")
elif 5 <= x <= 10:
    print("medium")
else:
    print("small")

for i in range(3):        # 0,1,2
    print(i)

names = ["Ada", "Linus"]
ages  = [36, 54]
for name, age in zip(names, ages):
    print(name, age)

for i, name in enumerate(names, start=1):
    print(i, name)

# while loop
n = 3
while n > 0:
    print(n); n -= 1
```
**`break` / `continue`**
```python
for i in range(5):
    if i == 2:
        continue   # skip 2
    if i == 4:
        break      # stop at 4
    print(i)
```

---

## 6) Exceptions — `try/except/else/finally`, custom exceptions

```python
class DataError(Exception):
    """Custom domain error."""

def parse_int(s: str) -> int:
    try:
        return int(s)
    except ValueError as e:
        raise DataError(f"Invalid int: {s}") from e
    finally:
        pass  # cleanup if needed
```
**Guidelines**
- Catch specific exceptions. Re-raise as domain errors if helpful.
- `else` block runs when no exception; `finally` always runs.

---

## 7) Functions — parameters, defaults, `*args/**kwargs`, annotations

```python
def greet(name: str, times: int = 1) -> None:
    for _ in range(times):
        print(f"Hello, {name}!")

def demo(a, b=0, *args, scale=1.0, **kwargs):
    """
    a, b           -> positional/keyword
    *args          -> extra positional
    scale=         -> keyword-only (after *args)
    **kwargs       -> extra keyword
    """
    return (a + b + sum(args)) * scale
```
**Gotchas with mutable defaults**
```python
def bad(acc=[]):          # BAD: list shared across calls
    acc.append(1); return acc

def good(acc=None):
    if acc is None: acc = []
    acc.append(1); return acc
```

---

## 8) Functional tools — `lambda`, `map/filter/reduce`, `sorted`

```python
from functools import reduce

nums = [1,2,3,4]
doubles = list(map(lambda x: x*2, nums))
evens   = list(filter(lambda x: x%2==0, nums))
total   = reduce(lambda a,b: a+b, nums, 0)
by_len  = sorted(["pear","fig","apple"], key=len)
```
**Tip**: Prefer comprehensions over `map/filter` when readability wins.

---

## 9) Scope & Closures (LEGB), `nonlocal`/`global`

```python
def make_counter():
    count = 0            # enclosing scope
    def inc():
        nonlocal count   # modify outer variable
        count += 1
        return count
    return inc

c = make_counter()
print(c(), c(), c())     # 1 2 3
```

---

## 10) Modules & Packages

```
mypkg/
├── __init__.py      # makes this a package
├── util.py          # module
└── models/
    ├── __init__.py
    └── user.py
```
```python
# util.py
def add(a,b): return a+b

# elsewhere
from mypkg.util import add
import mypkg.models.user as u
```
**Rules of thumb**
- Group code by feature/responsibility.
- Avoid circular imports; move shared pieces to a common module.

---

## 11) Classes, Methods, Inheritance, `super()`

```python
class Shape:
    def area(self) -> float:
        raise NotImplementedError

class Rect(Shape):
    def __init__(self, w: float, h: float):
        self.w = w; self.h = h
    def area(self) -> float:
        return self.w * self.h

class Square(Rect):
    def __init__(self, side: float):
        super().__init__(side, side)
```
**Special methods**
```python
class Vec:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __repr__(self):
        return f"Vec({self.x}, {self.y})"
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
```
**`@classmethod` & `@staticmethod`**
```python
class User:
    def __init__(self, name): self.name = name
    @classmethod
    def from_dict(cls, d): return cls(d["name"])
    @staticmethod
    def greet(): return "hi"
```

---

## 12) Dataclasses (best of both worlds)

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Order:
    id: int
    items: List[str] = field(default_factory=list)  # avoid mutable default bug
    total: float = 0.0
```
**Why**
- Auto `__init__`, `__repr__`, `__eq__`.
- Type hints required → better IDE & static checks.

---

## 13) Context Managers — `with`, custom ones

```python
# file I/O: auto-close even on error
with open("data.txt") as f:
    for line in f:
        print(line.strip())
```
**Custom with class**
```python
class timer:
    import time
    def __enter__(self):
        from time import perf_counter as pc
        self.t0 = pc(); return self
    def __exit__(self, exc_type, exc, tb):
        from time import perf_counter as pc
        print(f"took {pc()-self.t0:.3f}s")
```
**Or with decorator**
```python
from contextlib import contextmanager

@contextmanager
def open_readonly(path):
    f = open(path)
    try:
        yield f
    finally:
        f.close()
```

---

## 14) Decorators — add behavior without editing functions

```python
import time, functools

def timeit(fn):
    @functools.wraps(fn)              # keep name/docstring
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            dt = time.perf_counter()-t0
            print(f"{fn.__name__} took {dt:.3f}s")
    return wrapper

@timeit
def heavy(n):
    return sum(i*i for i in range(n))
```
**Use cases**: timing, logging, caching (`functools.lru_cache`), retry wrappers.

---

## 15) Data Handling — CSV, JSON, Pandas (essentials)

```python
# CSV (small/simple) — stdlib
import csv
with open("input.csv") as f:
    for row in csv.DictReader(f):
        print(row["name"], row["age"])

# JSON (nested structures) — stdlib
import json
data = json.load(open("data.json"))
print(data["items"][0]["id"])
```
**Pandas quickstart**
```python
import pandas as pd

df = pd.read_csv("input.csv")   # or pd.read_json(...)
print(df.head(), df.shape, df.dtypes, sep="\n")

# select/transform
df = df.assign(price_eur=df["price_usd"]*0.92)\
       .rename(columns={"price_usd":"priceUSD"})

# to dataclasses
from dataclasses import dataclass
@dataclass
class Item: id:int; name:str; price_eur:float
items = [Item(**row) for row in df.to_dict("records")]
```
**Handling issues**
- Validate schemas, handle missing values (`df.fillna(...)`), and log/skip bad rows when needed.

---

## 16) Testing — pytest basics (assertions, fixtures, parametrize, mocking)

```
project/
├── src/
│   └── app.py
└── tests/
    └── test_app.py
```

**test_app.py**
```python
import pytest
from src.app import add

def test_add():
    assert add(2,3) == 5

@pytest.fixture
def db():
    return {"items":[1,2,3]}

@pytest.mark.parametrize("a,b,ans", [(1,2,3),(0,0,0)])
def test_param(a,b,ans):
    assert add(a,b) == ans

def test_mock_requests(monkeypatch):
    import requests
    def fake_get(url): 
        class R: status_code=200; text="ok"
        return R()
    monkeypatch.setattr(requests, "get", fake_get)
    assert requests.get("http://x").text == "ok"
```
**Run**
```bash
pip install pytest
pytest -q      # run all tests
```

---

## 17) Type Hints & Static Typing (mypy/pyright)

```python
from typing import Optional, Iterable

def mean(xs: Iterable[float]) -> float:
    total, n = 0.0, 0
    for x in xs: total += x; n += 1
    return total / n if n else 0.0

def find(name: str, pool: list[str]) -> Optional[int]:
    try:
        return pool.index(name)
    except ValueError:
        return None
```
**Check types**
```bash
pip install mypy
mypy src/              # static type check
```
- Use `pyproject.toml` to configure tools consistently.

---

## 18) Linting, Formatting & Automation (black, isort, ruff, pre-commit)

**Tooling install**
```bash
pip install black isort ruff pre-commit
```
**Run once**
```bash
black .        # auto-format
isort .        # sort imports
ruff .         # fast lints
```
**Pre-commit (auto run on commit)**
```yaml
# .pre-commit-config.yaml
repos:
- repo: https://github.com/psf/black
  rev: 24.8.0
  hooks: [{id: black}]
- repo: https://github.com/pycqa/isort
  rev: 5.13.2
  hooks: [{id: isort}]
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.6.9
  hooks: [{id: ruff}]
```
```bash
pre-commit install   # enables hooks
```

---

## 19) Cheatsheet — What to reach for

- **Looping**: `for`, `enumerate`, `zip`, comprehensions.
- **Collections**: `dict`, `list`, `set`, `Counter`, `defaultdict`, `deque`.
- **Errors**: `try/except`; define custom exceptions for domain errors.
- **Functions**: defaults, `*args/**kwargs`, higher-order functions.
- **OOP**: classes, dataclasses when you just need structured data.
- **Files**: `with open(...)` to ensure closing; custom context managers for setup/teardown.
- **Testing**: `pytest` + fixtures + parametrization; mock IO/network.
- **Typing**: add hints; run `mypy` for early bug catching.
- **Style**: `black`, `isort`, `ruff`; automate with `pre-commit` and CI.

---

You now have the core building blocks to **read, transform, validate, test, and ship** Python code confidently. Keep this file as a reference and tweak the examples into your own utilities and apps.
