# R# Beginner Guide

R# is a small, statically typed programming language with Python-style conveniences and a native
compilation pipeline:

**R# source → R# compiler → C11 → C compiler → native executable**

This guide describes the language that is actually implemented by the repository this guide belongs to.
It does not describe planned features as though they already exist.

> **Repository status:** the checked-in CLI reports `rsharp 1.2.0`, although the archive/repository
> directory is named `rsharp-lang2.0.0`. The compiler and tests are the source of truth for syntax and
> behavior.

---

## 1. Installing and running R#

### Requirements

You need:

- Python 3.11 or newer.
- A C compiler. The normal compiler is `gcc`; set `CC=clang` if you want to use Clang.
- Linux x86-64 is the tested environment in the repository. Windows is supported only experimentally
  through WSL/MinGW in the existing documentation.

The compiler invokes the C compiler with GNU C11 mode.

### Run a single program

From the repository root:

```sh
sh bin/rsharp examples/hello.rsharp
```

The file can also be passed without `run`:

```sh
sh bin/rsharp examples/hello.rsharp
```

Explicit form:

```sh
sh bin/rsharp run examples/hello.rsharp
```

### Build a native executable

```sh
sh bin/rsharp build examples/primes.rsharp -o primes
./primes
```

### Check without running

```sh
sh bin/rsharp check examples/primes.rsharp
```

### Inspect generated C

```sh
sh bin/rsharp emit-c examples/primes.rsharp
```

### REPL

```sh
sh bin/rsharp repl
```

The REPL is replay-based: accepted history is compiled and executed again for each entry. Avoid
`input()`, `random`, and `time` in the REPL because replay makes those operations unsuitable for
interactive use.

### Run the test suite

```sh
sh bin/rsharp test tests
```

Tests use comments such as:

```rsharp
print(2 + 2); // expect: 4
```

and compile-failure tests use:

```rsharp
// expect-error: cannot assign
```

---

## 2. Your first R# program

Create `hello.rsharp`:

```rsharp
print("Hello, world!");
```

Run it:

```sh
sh bin/rsharp hello.rsharp
```

R# requires semicolons after ordinary statements.

You can also use a `main` function:

```rsharp
fn main() {
    print("Hello from main!");
}
```

A file can contain top-level script statements and `fn main()`. Top-level statements execute in
source order, and then `main()` runs if it exists.

Functions may be called before their declaration:

```rsharp
print(add(2, 3));

fn add(a: int, b: int) -> int {
    return a + b;
}
```

---

## 3. Comments

R# supports both `//` and `#` comments, plus block comments.

```rsharp
// C-style comment
# Python-style comment

/*
   Multi-line comment.
*/
print("comments are ignored");
```

`//` is a comment, not integer floor division. Integer `/` already truncates toward zero.

---

## 4. Variables and constants

### `let`

`let` creates an immutable binding:

```rsharp
let name = "Alice";
let age = 30;
```

You cannot assign to a `let` variable later.

### `mut`

Use `mut` when the binding itself must be changed:

```rsharp
mut score = 10;
score += 5;
score = 100;
```

### `const`

Constants are immutable:

```rsharp
const LIMIT = 100;
const PI: float64 = 3.14159;
```

A top-level `const` declared before a function is visible to that function:

```rsharp
const SCALE = 3;

fn scaled(x: int) -> int {
    return x * SCALE;
}
```

Variables must be initialized when declared:

```rsharp
let x = 10;       // valid
// let y;         // invalid
```

Names may be shadowed in an inner block:

```rsharp
let x = 1;

{
    let x = x + 10;
    print(x);      // 11
}

print(x);          // 1
```

---

## 5. Types

R# is statically typed.

The implemented core types are:

| Type | Meaning |
|---|---|
| `int` | signed 64-bit integer; `int64` is an alias |
| `float64` | 64-bit floating point |
| `bool` | `true` or `false` |
| `string` | string/byte sequence |
| `void` | no value |
| `T[]` | array of `T` |
| `map<T>` | string-keyed map containing values of `T` |
| `StructName` | a user-defined struct value |

Examples:

```rsharp
let count: int = 10;
let price: float64 = 2.50;
let enabled: bool = true;
let title: string = "R#";
let values: int[] = [1, 2, 3];
let scores: map<float64> = {"alice": 9.5};
```

There are no implicit numeric conversions:

```rsharp
let x = 1;
// let y = x + 2.0;       // error
let y = float64(x) + 2.0;
```

Useful conversions:

```rsharp
int(3.99)       // 3
float64(2)      // 2.0
str(42)         // "42"
```

`int(x)` truncates floating-point values and can panic if the value is outside the representable
integer range. `int("42")` parses a string.

---

## 6. Numbers and operators

### Integer arithmetic

```rsharp
print(7 + 2);    // 9
print(7 - 2);    // 5
print(7 * 2);    // 14
print(7 / 2);    // 3
print(7 % 2);    // 1
```

Integer arithmetic is checked. Overflow and integer division by zero cause a runtime panic.

### Floating-point arithmetic

```rsharp
print(1.5 + 2.25);
print(3.0 / 2.0);
print(2.0 ** 0.5);
```

### Exponentiation

```rsharp
print(2 ** 10);       // 1024
print(2.0 ** 0.5);
```

### Comparisons

```rsharp
print(3 < 4);
print(3 <= 3);
print(5 > 2);
print(5 >= 5);
print(5 == 5);
print(5 != 4);
```

Comparisons require compatible types. Strings can also be compared.

### Boolean operators

```rsharp
let ok = true and false;
let good = true or false;
let answer = not false;
```

The symbolic forms `&&`, `||`, and `!` are also supported:

```rsharp
if age >= 18 && enabled {
    print("allowed");
}
```

`and`/`or` short-circuit just like `&&`/`||`.

R# does not use truthiness. Conditions must actually be `bool`.

### Evaluation order

R# evaluates expressions from left to right, including function arguments:

```rsharp
fn tick(label: string) -> int {
    print(label);
    return 1;
}

print(tick("first") + tick("second"));
```

Output is:

```text
first
second
2
```

---

## 7. Strings

Strings use double quotes:

```rsharp
let name = "R#";
let message = "hello";
```

Supported escapes include:

```text
\n
\t
\r
\\
\"
\0
```

Concatenate with `+`:

```rsharp
let full = "Hello, " + "world!";
```

Repeat with an integer:

```rsharp
print("ab" * 3);     // ababab
```

### Interpolated strings

Use `$"..."`:

```rsharp
let name = "Alice";
let age = 30;

print($"Name: {name}, age: {age + 1}");
```

Literal braces are written as `{{` and `}}`:

```rsharp
print($"{{hello}}");  // {hello}
```

### String indexing and slicing

```rsharp
let s = "hello";

print(s[0]);       // h
print(s[-1]);      // o
print(s[1:3]);     // el
print(s[:2]);      // he
print(s[2:]);      // llo
```

String indexing is byte-oriented. The repository documents this as ASCII-oriented behavior; for
UTF-8 text, `length` counts bytes rather than Unicode characters.

```rsharp
print("héllo".length);   // 6 bytes
```

### String methods

Implemented methods:

```rsharp
s.upper()
s.lower()
s.strip()
s.contains(sub)
s.startswith(prefix)
s.endswith(suffix)
s.replace(old, new)
s.split(separator)
s.find(sub)
separator.join(parts)
```

Example:

```rsharp
let s = "  Hello, World  ";

print(s.strip().upper());
print("a,b,c".split(","));
print(", ".join(["a", "b", "c"]));
print("hello".replace("l", "L"));
print("hello".find("ll"));
```

`find()` returns an integer index. `ord()` converts a one-character string to an integer code and
`chr()` converts an integer code to a one-character string.

---

## 8. Arrays

An array is written `T[]`:

```rsharp
mut numbers = [10, 20, 30];
```

R# infers the element type from the elements.

For an empty array, give the type:

```rsharp
let empty: int[] = [];
mut names: string[] = [];
```

### Indexing

```rsharp
print(numbers[0]);
numbers[1] = 99;
```

Indexes are bounds checked.

Negative indexes are supported:

```rsharp
print(numbers[-1]);
```

### Slicing

```rsharp
let a = [0, 1, 2, 3, 4];

print(a[1:4]);  // [1, 2, 3]
print(a[:3]);   // [0, 1, 2]
print(a[2:]);   // [2, 3, 4]
```

### Length

```rsharp
print(numbers.length);
print(len(numbers));
```

### Array methods

```rsharp
numbers.push(40);
numbers.append(50);

let last = numbers.pop();

print(numbers.contains(20));

numbers.reverse();
numbers.sort();

numbers.extend([60, 70]);
numbers.clear();

let copy = numbers.copy();
```

`push`, `append`, `pop`, `reverse`, `sort`, `extend`, and `clear` require a mutable root array.

### Arrays are reference-like objects

Assigning an array does not duplicate its contents:

```rsharp
mut a = [1];
mut b = a;

b.push(2);

print(a);  // [1, 2]
print(b);  // [1, 2]
```

This is important when designing functions: an array parameter cannot be mutated through the
parameter binding unless the language rules permit a mutable root in the calling context. The current
checker deliberately protects immutable bindings from mutation.

---

## 9. Maps

Maps have string keys and one value type:

```rsharp
mut ages = {
    "alice": 30,
    "bob": 25
};
```

An empty map needs an annotation:

```rsharp
let empty: map<int> = {};
```

Access and assignment:

```rsharp
print(ages["alice"]);
ages["charlie"] = 40;
ages["alice"] += 1;
```

Methods:

```rsharp
ages.get("missing", 0);
ages.keys();
ages.values();
ages.remove("bob");
ages.clear();
```

You can test membership:

```rsharp
if "alice" in ages {
    print("found");
}
```

Iterate over key/value pairs:

```rsharp
for name, age in ages {
    print($"{name}: {age}");
}
```

Maps preserve insertion order in the current implementation.

Map lookup is currently linear, so maps are convenient but are not designed as a high-performance
hash-table implementation.

---

## 10. Functions

Functions require typed parameters.

```rsharp
fn add(a: int, b: int) -> int {
    return a + b;
}
```

Call them normally:

```rsharp
print(add(10, 20));
```

A function without `-> Type` returns `void`:

```rsharp
fn greet(name: string) {
    print($"Hello, {name}!");
}
```

A short expression form is available:

```rsharp
fn square(x: int) -> int => x * x;
fn say() => print("said");
```

### Recursion

```rsharp
fn factorial(n: int) -> int {
    if n <= 1 {
        return 1;
    }
    return n * factorial(n - 1);
}
```

### Return checking

A non-void function must return on every possible path:

```rsharp
fn positive(x: int) -> string {
    if x > 0 {
        return "yes";
    }
    return "no";
}
```

Functions are not first-class values yet. You cannot store a function in a variable, pass a function
as an argument, or create closures.

---

## 11. Control flow

### `if`

```rsharp
if score >= 90 {
    print("A");
} else if score >= 80 {
    print("B");
} else {
    print("C");
}
```

Parentheses around conditions are not required.

### `while`

```rsharp
mut i = 0;

while i < 10 {
    print(i);
    i += 1;
}
```

### `for` over a range

The `..` range is half-open:

```rsharp
for i in 0..5 {
    print(i);
}
```

prints `0` through `4`.

The upper bound is evaluated once.

### `range()`

Use Python-style ranges when you want a step:

```rsharp
for i in range(10, 0, -3) {
    print(i);
}
```

prints:

```text
10
7
4
1
```

`range()` takes one to three integer arguments.

### `for` over arrays and strings

```rsharp
for value in [10, 20, 30] {
    print(value);
}

for ch in "hi" {
    print(ch);
}
```

### `for` over maps

```rsharp
for key, value in ages {
    print(key, value);
}
```

### `break` and `continue`

```rsharp
for i in 0..100 {
    if i % 2 == 0 {
        continue;
    }
    if i > 20 {
        break;
    }
    print(i);
}
```

---

## 12. Built-in functions

The implemented built-ins are:

| Function | Purpose |
|---|---|
| `print(...)` | print space-separated values followed by a newline |
| `len(x)` | length of string, array, or map |
| `int(x)` | convert/parse to integer |
| `float64(x)` | convert/parse to float |
| `str(x)` | convert to string |
| `range(...)` | produce an integer array |
| `input(prompt?)` | read a line |
| `abs(x)` | absolute value |
| `min(a, b)` / `min(array)` | minimum |
| `max(a, b)` / `max(array)` | maximum |
| `sum(array)` | numeric sum |
| `sorted(array)` | sorted copy |
| `reversed(array)` | reversed copy |
| `round(x)` | round to integer |
| `read_file(path)` | read a text file |
| `write_file(path, text)` | write a text file |
| `exit(code)` | terminate with an exit code |
| `assert(condition, message?)` | runtime assertion |
| `ord(s)` | convert character string to integer |
| `chr(n)` | convert integer to character string |

Example:

```rsharp
let xs = [5, 3, 8, 1];

print(len(xs));
print(sum(xs));
print(min(xs), max(xs));
print(sorted(xs));
print(reversed(xs));
```

`round()` uses banker's rounding in the current implementation, matching the documented Python-style
behavior.

---

## 13. Modules

A module is another `.rsharp` file.

Suppose `maths.rsharp` contains:

```rsharp
pub fn square(x: int) -> int => x * x;
```

Import it:

```rsharp
import maths;

print(maths.square(5));
```

Aliases are supported:

```rsharp
import maths as m;

print(m.square(5));
```

You can import selected members:

```rsharp
from maths import square;

print(square(5));
```

And aliases:

```rsharp
from maths import square as sq;

print(sq(5));
```

Imports must be at the top level.

The `pub` keyword is accepted. Current modules effectively expose their functions/constants to the
importer; `pub` does not yet implement a complete visibility system.

Modules can contain functions, structs, constants, and imports.

### Module search

The loader checks native modules first, then local files/directories and vendored packages, then
bundled packages.

Typical local forms include:

```text
./m.rsharp
./m/lib.rsharp
rsharp_packages/m/lib.rsharp
```

---

## 14. Native modules

The compiler provides four native modules:

```rsharp
import math;
import random;
import time;
import game;
```

### `math`

Functions:

```text
sqrt(x)
pow(x, y)
floor(x)
ceil(x)
sin(x)
cos(x)
tan(x)
atan2(y, x)
log(x)
log10(x)
exp(x)
hypot(x, y)
gcd(a, b)
```

Constants:

```text
math.pi
math.e
math.tau
math.inf
```

Example:

```rsharp
import math;

print(math.sqrt(25.0));
print(math.pi);
print(math.gcd(84, 30));
```

Most mathematical arguments accept either `int` or `float64`.

### `random`

```rsharp
import random;

random.seed(123);
print(random.random());
print(random.randint(1, 6));
```

### `time`

```rsharp
import time;

print(time.now());
time.sleep(0.1);
```

---

## 15. Packages and Sharpie

R# includes an offline package system. `sharpie` is a friendly front-end around the package commands.

Create an application:

```sh
sh bin/sharpie new MyApp
cd MyApp
sh ../bin/sharpie run
```

Create a game:

```sh
sh bin/sharpie new MyGame
cd MyGame
sh ../bin/sharpie run
```

Initialize an existing directory:

```sh
rsharp pkg init
```

Add a bundled package:

```sh
rsharp pkg add @rsharp/text
```

Install/update bundled packages through Sharpie:

```sh
sharpie install
```

List dependencies:

```sh
rsharp pkg list
```

Search bundled packages:

```sh
rsharp pkg search text
```

Remove a dependency:

```sh
rsharp pkg remove text
```

Audit the vendored lockfile contents:

```sh
rsharp pkg audit
```

The package system is offline. There is no online registry yet, so commands cannot fetch arbitrary
packages from the Internet.

### Bundled packages

The repository currently bundles:

- `text`
- `stats`
- `algo`
- `vec`

Example:

```rsharp
from text import pad_left;

print(pad_left("42", 5, "0"));
```

---

## 16. Structs

Structs are implemented value types.

Names must begin with an uppercase letter.

```rsharp
struct Point {
    x: float64,
    y: float64
}
```

Create a value:

```rsharp
let p = Point {
    x: 10.0,
    y: 20.0
};
```

Access fields:

```rsharp
print(p.x, p.y);
```

Mutable fields require a mutable root:

```rsharp
mut p = Point { x: 10.0, y: 20.0 };

p.x = 15.0;
p.y += 5.0;
```

Structs are copied on assignment and function passing:

```rsharp
let a = Point { x: 0.0, y: 0.0 };
mut b = a;

b.x = 99.0;

print(a.x);  // 0.0
print(b.x);  // 99.0
```

Structs can contain scalars, earlier-declared structs, arrays, and maps.

The current implementation does not provide methods/`impl`, recursive structs, struct equality, default
field values, or generic structs.

---

## 17. A complete small program

This program combines functions, arrays, loops, interpolation, and sorting:

```rsharp
fn is_prime(n: int) -> bool {
    if n < 2 {
        return false;
    }

    mut d = 2;
    while d * d <= n {
        if n % d == 0 {
            return false;
        }
        d += 1;
    }

    return true;
}

fn main() {
    mut primes: int[] = [];

    for n in 0..60 {
        if is_prime(n) {
            primes.push(n);
        }
    }

    print($"{primes.length} primes below 60:");
    print(primes);
}
```

Build it:

```sh
rsharp build primes.rsharp -o primes
./primes
```

---

## 18. A practical project layout

A normal R# application can look like:

```text
my_app/
├── rsharp.toml
├── rsharp.lock
├── src/
│   ├── main.rsharp
│   └── math_helpers.rsharp
├── tests/
│   └── test_main.rsharp
└── rsharp_packages/
```

Create the project with:

```sh
rsharp pkg new app my_app
```

For a library:

```sh
rsharp pkg new library my_library
```

For a game:

```sh
rsharp pkg new game my_game
```

---

## 19. Testing

The repository's test runner treats `// expect:` comments as exact stdout expectations.

Example:

```rsharp
fn add(a: int, b: int) -> int => a + b;

print(add(2, 3)); // expect: 5
```

Expected compile failures use:

```rsharp
// expect-error: functions are not first-class values yet
```

Run:

```sh
rsharp test tests
```

This makes the compiler itself testable without requiring a separate test framework.

---

## 20. Games: what exists today

R# includes a real but deliberately small headless game runtime.

There is currently:

- no window;
- no GPU renderer;
- no audio;
- no input devices.

Instead, games use a deterministic software framebuffer and can export frames as PPM images.

A minimal game:

```rsharp
import game;

game "Demo";

fn update(dt: float64) {
}

fn draw() {
    game.clear(0, 0, 0);
}
```

A game must have at least `update(delta)` or `draw()`.

The full game API and lifecycle are covered in the Advanced Guide.

---

## 21. Important current limitations

R# is not yet equivalent to C, C++, or Python in feature count.

Not implemented in this repository include:

- classes;
- interfaces/traits;
- enums;
- pattern matching;
- generics;
- closures/function values;
- `Option`/`Result`;
- async/await;
- threads;
- exceptions;
- pointers/`unsafe`;
- FFI;
- garbage collection;
- LLVM/WASM backends;
- online package registry;
- dependency version-range resolver;
- publishing packages;
- formatter/linter/documentation generator;
- language server;
- debugger/profiler;
- GPU rendering;
- window/input/audio/3D game systems;
- ECS, physics, scenes, animation, UI, networking, hot reload.

The correct mental model is: **R# is a compact statically typed language with a working native C
backend and a growing standard/tooling layer, not yet a full general-purpose ecosystem.**

---

## 22. Recommended learning path

1. Write single-file scripts.
2. Learn `let`, `mut`, `const`, and the core types.
3. Practice functions and return types.
4. Learn arrays, strings, and maps.
5. Use `if`, `while`, `for`, `break`, and `continue`.
6. Learn interpolation and Python-style operations.
7. Split code into modules.
8. Add bundled packages with Sharpie.
9. Learn structs for larger data models.
10. Move to the Advanced Guide for compiler behavior, packages, native modules, testing, and games.
