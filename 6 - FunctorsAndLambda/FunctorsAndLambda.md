# Modern C++ Functors, Lambdas & Compile-Time Customizability

## 📋 Table of Contents
1. [Designing for Reusability in Modern C++](#1-designing-for-reusability-in-modern-c)
2. [Customizability through Callbacks](#2-customizability-through-callbacks)
   - [What Is a Callback?](#what-is-a-callback)
   - [Types of Callbacks in C++](#types-of-callbacks-in-c)
   - [Why Were Functors Invented? (Historical Context)](#why-were-functors-invented-historical-context)
   - [Problem 1 – "Piping State" into Predicates](#problem-1--piping-state-into-predicates)
   - [Problem 2 – Function Pointers "Kill Inlining"](#problem-2--function-pointers-kill-inlining)
   - [Problem 3 – Overload Sets & Template Operators](#problem-3--overload-sets--template-operators)
   - [Functors: The Foundation](#functors-the-foundation)
   - [Lambdas: Compiler-Generated Functors](#lambdas-compiler-generated-functors)
     - [Lambda Return Type Specification](#lambda-return-type-specification)
     - [Lambda mutable Keyword](#lambda-mutable-keyword)
   - [std::function: Type-Erased Wrapper](#stdfunction-type-erased-wrapper)
   - [std::invoke: Universal Callable Invoker](#stdinvoke-universal-callable-invoker)
   - [std::bind: Partial Function Application](#stdbind-partial-function-application)
   - [Performance Deep Dive – Assembly & Benchmarks](#performance-deep-dive--assembly--benchmarks)
   - [Summary Table – Callback Types](#summary-table--callback-types)
3. [Customizability through Template Parameters](#3-customizability-through-template-parameters)
   - [Templates with Callable Parameters](#templates-with-callable-parameters)
   - [Templates with Types (Policy Classes)](#templates-with-types-policy-classes)
   - [Why Template Customizability Matters](#why-template-customizability-matters)
4. [Concepts (C++20)](#4-concepts-c20)
5. [Comparison: When to Use What](#5-comparison-when-to-use-what)
6. [Modern C++ Best Practices](#6-modern-c-best-practices)
7. [Real-World Idiomatic Patterns](#7-real-world-idiomatic-patterns)
8. [Decision Tree – Which One Should I Pick?](#8-decision-tree--which-one-should-i-pick)
9. [Complete Example – Flexible Logger System](#9-complete-example--flexible-logger-system)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Designing for Reusability in Modern C++

Reusable and maintainable code is a core principle of modern software development. Whether you're building a small class or a full subsystem, design your code so it can be used again—by you or by other developers. This idea follows well-known principles:

* **Write once, use often.**
* **Avoid duplication.**
* **DRY — Don't Repeat Yourself.**

### Why Reusability Matters

1. **You will need the same functionality again.**
   Reusable code saves you from rewriting the same logic in future projects.

2. **It reduces long-term effort.**
   A clean design prevents the need to reinvent solutions later.

3. **It supports teamwork.**
   Reusable components make it easier for others to integrate and understand your work.

4. **It avoids maintenance issues.**
   Duplicated code creates multiple points of failure. Centralizing logic prevents this.

5. **You benefit the most.**
   Over time, reusable components form a personal toolkit that speeds up future development.

Modern C++ provides powerful tools for reusable design: **callbacks** (function pointers, functors, lambdas) and **templates**. This guide explores these mechanisms in depth.

---

## 2. Customizability through Callbacks

In C++, a **callback** is a **function that you pass to another function or object to be called later**. This allows you to **customize behavior** without modifying the original code. Callbacks are widely used in event handling, error handling, asynchronous programming, and generic algorithms.

### What Is a Callback?

A callback is:

* A piece of code (function, lambda, or object with `operator()`)
* Passed to another function or class
* Called **later by that function/class** when some action occurs

Example in plain words: Imagine you hire a chef (callback) and tell a waiter (your function) to "call the chef when an order arrives." You don't care how the chef cooks; the waiter just calls them at the right time.

### Types of Callbacks in C++

C++ supports three main kinds of callbacks:

#### a) Function Pointer Callback

A **function pointer** is a pointer that stores the address of a function. The function must have a fixed signature.

```cpp
#include <iostream>

void myCallback(int x) {
    std::cout << "Function pointer callback: " << x << "\n";
}

void doWork(void (*callback)(int)) {
    callback(42); // call the callback
}

int main() {
    doWork(myCallback);
}
```

**Key points:**
* Works with **normal functions** only
* Simple and efficient
* Cannot capture external variables

#### b) Function Object Callback (Functor)

A **function object** is an object that defines the `operator()`. It behaves like a function.

```cpp
#include <iostream>

struct MyFunctor {
    void operator()(int x) const {
        std::cout << "Function object callback: " << x << "\n";
    }
};

template <typename Callback>
void doWork(Callback cb) {
    cb(42); // call the callback
}

int main() {
    MyFunctor f;
    doWork(f);
}
```

**Key points:**
* Can store **state** (member variables)
* Works with templates
* More flexible than function pointers

#### c) Lambda Callback

A **lambda** in C++ is an **inline anonymous function** that can be defined **at the point of use**. Lambdas are a modern C++ favorite for implementing callbacks because they are concise, flexible, and can capture variables from their surrounding scope.

**Key characteristics:**
1. **Anonymous**: No separate function declaration is required
2. **Inline**: Defined at the place you need it
3. **Captures variables**: By value `[var]` or by reference `[&var]`
4. **Flexible parameters**: Can take zero or more parameters, including references
5. **Works with `std::function`**: Can store or pass lambdas anywhere a callable is expected

##### Basic Lambda Example

```cpp
#include <iostream>
#include <functional>

void doWork(const std::function<void(int)>& callback) {
    callback(42); // call the lambda
}

int main() {
    int factor = 10;
    doWork([factor](int x) {
        std::cout << "Lambda callback: " << x * factor << "\n";
    });
}
```

* `[factor]` → captures `factor` **by value** (a copy)
* `(int x)` → the lambda accepts **one parameter**

##### Lambda with Multiple Parameters and References

```cpp
#include <iostream>
#include <functional>

void doWork(const std::function<void(int&, int&)>& callback) {
    int a = 5, b = 10;
    callback(a, b); // call the lambda
    std::cout << "After callback: a = " << a << ", b = " << b << "\n";
}

int main() {
    int factor = 3;
    doWork([&factor](int& x, int& y) {
        x *= factor;
        y *= factor;
    });
}
```

**Output:**
```
After callback: a = 15, b = 30
```

* `[&factor]` → captures `factor` **by reference**
* `int& x, int& y` → parameters passed **by reference**, allowing modification

### Why Were Functors Invented? (Historical Context)

Functors were **not invented to be cute syntax**; they were the **minimal, zero-overhead answer to three concrete problems** that 1990-era C++ library writers (mainly Stepanov and the STL team) kept hitting when they tried to write *generic, fast, reusable* algorithms.

| Problem (1990-era STL) | Free Function | Functor Solution |
|------------------------|---------------|------------------|
| 1. Need **state** inside comparator | global variable → not thread-safe, not re-entrant | store state as **data member** |
| 2. Function pointer **prevents inlining** | virtual call inside tight loop | template + `operator()` → **fully inlined** |
| 3. Need **overload set as value** | overload set has **no address** | class can have **many** `operator()` & be stored |

Stepanov’s early benchmarks showed **2-4× speed-ups** for small comparators like `std::less<int>` when they were functors instead of function pointers. That single optimisation made the STL competitive with hand-written C code.

### Problem 1 – "Piping State" into Predicates

"**Pipe state**" = carry the data **inside** the callable instead of globals.

**Bad – global variable**
```cpp
double gLimit = 3.14;
bool larger(double x) { return x > gLimit; }   // thread-unsafe, not re-entrant
```

**Good – state piped inside object**
```cpp
struct LargerThan {
    double limit;                           // <- the knob
    explicit LargerThan(double l) : limit(l) {}
    bool operator()(double x) const { return x > limit; }
};

std::vector<double> v{1, 4, 2, 9};
auto it1 = std::find_if(v.begin(), v.end(), LargerThan{3.0});
auto it2 = std::find_if(v.begin(), v.end(), LargerThan{7.0});
```

Each object carries its own `limit` – **no globals, no mutexes, no surprises**.

### Problem 2 – Function Pointers "Kill Inlining"

Function pointer = **run-time address** → compiler cannot inline through it.

**Code**
```cpp
bool cmp_ptr(int a, int b) { return a < b; }          // free function
using Func = bool(*)(int,int);

int sum_if_ptr(const int* v, int n, Func f)           // pointer
{
    int s = 0;
    for (int i = 0; i < n-1; ++i)  if (f(v[i], v[i+1])) s += v[i];
    return s;
}

template<class F>
int sum_if_obj(const int* v, int n, F f)              // functor, templated
{
    int s = 0;
    for (int i = 0; i < n-1; ++i)  if (f(v[i], v[i+1])) s += v[i];
    return s;
}
```

**Assembly (GCC 13 -O3)**
```asm
sum_if_ptr:   call    cmp_ptr@plt        ; **real call** inside hot loop
sum_if_obj:   cmp     DWORD PTR [rdi+4], eax   ; **inlined comparison**
```

**Benchmark** (1 M integers)  
`sum_if_ptr` ≈ **80 ms** – `sum_if_obj` ≈ **18 ms** → **4.4× faster**.

That is what "kills inlining" means: the **pointer is a firewall** the optimiser cannot cross.

### Problem 3 – Overload Sets & Template Operators

Free-function overload set **has no address** – you cannot store it, template it per object, or pass it as a single value.

**Functor – unlimited overloads + state**
```cpp
struct Hash {
    std::size_t seed;
    explicit Hash(std::size_t s = 0) : seed(s) {}

    template<class T>
    std::size_t operator()(const T& t) const
    { return seed ^ std::hash<T>{}(t); }
};

Hash h(42);
std::size_t a = h(123);              // int
std::size_t b = h(std::string{"x"}); // string
```

Same **object** works for **every type** and can be stored inside `unordered_map`:

```cpp
std::unordered_map<MyKey, std::string, Hash> table(0, Hash{123});
```

### Functors: The Foundation

A **functor** is **any C++ class/struct that overloads `operator()`**.

```cpp
struct Add {
    int bias;
    int operator()(int x) const { return x + bias; } // <-- makes it a functor
};

Add add5{5};
std::cout << add5(7); // 12
```

**Key super-powers**  
- Object semantics → copy, store in container, pass by value.  
- Carries **state** (data members) **per instance**.  
- Body visible to compiler → **inlinable** (zero-cost abstraction).  
- Can have **many** `operator()` overloads or even **template** `operator()`.

### Lambdas: Compiler-Generated Functors

Every lambda becomes an **unnamed class** (closure type) with an `operator()` whose signature matches the lambda.

```cpp
auto add = [bias = 10](int x){ return x + bias; };

// Roughly equivalent to:
class __lambda_unique {
    int bias;
public:
    __lambda_unique(int b) : bias(b) {}
    int operator()(int x) const { return x + bias; }
};
```

**Capture modes**
| Syntax | Meaning |
|--------|---------|
| `[=]` | copy **all** automatic variables by value |
| `[&]` | capture **all** by reference (lifetime danger!) |
| `[x,&y]` | mixed |
| `[x = std::move(obj)]` | C++14 **init capture** – perfect-forward movable objects |

** Stateless lambda → function pointer**
```cpp
auto fp = +[](){ };   // unary + triggers decay to void(*)()
```

#### Lambda Return Type Specification

C++11 introduced explicit lambda return type specification using the `->` syntax, giving you precise control over what your lambda returns. This is crucial when:
- The compiler cannot deduce the return type from a complex expression
- You have multiple return statements that need coercion to a common type
- You want SFINAE-friendly lambdas for template metaprogramming
- You need to ensure ABI compatibility across translation units

**Syntax and Mechanics:**
```cpp
auto lambda = [capture-list](parameters) -> return_type { body; };
```

The return type is placed after the parameter list and `->` (trailing return type syntax). This becomes especially important when your lambda's body isn't a simple single-expression return.

**When Explicit Return Types Are Essential:**

1. **Multiple Return Paths with Different Types:**
```cpp
// Error: compiler can't deduce common type
// auto bad = [](int x) { 
//     if (x > 0) return 42;      // int
//     else return 3.14;          // double - type mismatch!
// };

// Solution: explicitly specify double
auto good = [](int x) -> double { 
    if (x > 0) return 42;        // implicitly converts to double
    else return 3.14; 
};
```

2. **Complex Template Metaprogramming:**
```cpp
template<typename T>
auto make_converter(T factor) -> std::function<T(T)> {
    return [factor](T value) -> T {  // Explicit return ensures consistent type
        return value * factor;
    };
}
```

3. **SFINAE and Concept Constraints:**
```cpp
template<typename T>
auto maybe_process(T val) -> std::optional<T> {
    auto processor = [](auto x) -> std::optional<T> {
        if constexpr (std::is_arithmetic_v<decltype(x)>) {
            return x * 2;  // Arithmetic: process and return
        } else {
            return std::nullopt;  // Non-arithmetic: return empty
        }
    };
    return processor(val);
}
```

**Performance Implications:**
- Specifying return types prevents accidental type conversions inside the lambda
- Enables better optimization by giving the compiler explicit type information
- Reduces binary size by eliminating multiple template instantiations for the same logical return type

**Best Practices:**
- Use `auto` return type for simple, single-expression lambdas (let the compiler deduce)
- Specify return type explicitly for multi-statement lambdas or when type conversion is needed
- Always specify return type in public APIs where ABI stability matters
- Consider `-> decltype(auto)` for perfect forwarding of references

#### Lambda mutable Keyword

The `mutable` keyword fundamentally changes lambda semantics by removing the `const`-qualification from the lambda's `operator()`. By default, all captured-by-value variables are `const` inside the lambda body, mimicking function parameter behavior. `mutable` lifts this restriction.

**Core Concept:**
```cpp
int x = 5;
// By default: captured values are const
// auto lam = [x]() { x++; }; // ERROR: cannot modify const

// With mutable: removes const-qualifier
auto lam = [x]() mutable { x++; return x; }; // OK: modifies copy
std::cout << lam() << "\n"; // Prints 6
std::cout << x << "\n";     // Original x is still 5
```

**Key Mechanics:**
- Only affects **by-value captures** (`[=]` or `[x]`)
- By-reference captures (`[&]`) are always mutable (they refer to the original variable)
- Transforms the lambda's `operator()` from `void() const` to `void()`
- Each lambda invocation gets the **same modified state** (state persists across calls)

**Practical Use Cases:**

1. **Stateful Generator/Accumulator:**
```cpp
auto fibonacci_generator = [a = 0, b = 1]() mutable -> int {
    int current = a;
    a = b;
    b = current + b;
    return current;
};

// Each call produces the next Fibonacci number
std::cout << fibonacci_generator() << "\n"; // 0
std::cout << fibonacci_generator() << "\n"; // 1
std::cout << fibonacci_generator() << "\n"; // 1
std::cout << fibonacci_generator() << "\n"; // 2
```

2. **Cached Computation:**
```cpp
auto expensive_computation = [cache = std::optional<double>{}](double input) mutable {
    if (!cache.has_value()) {
        cache = some_expensive_operation(input); // Cache result
    }
    return cache.value();
};

// First call computes, subsequent calls use cache
double result1 = expensive_computation(3.14); // Computes
double result2 = expensive_computation(2.71); // Uses cached 3.14 result (BUG!)
```

**Critical Pitfall - Lifetime Bug:**
The above cache example demonstrates a common bug: the cache is **shared across all invocations** and won't be reset. For per-call caching, you need a custom functor class instead.

3. **Move-Only Type Capture:**
```cpp
std::unique_ptr<ExpensiveResource> resource = std::make_unique<ExpensiveResource>();

// Can't copy unique_ptr, must move-capture
auto processor = [res = std::move(resource)]() mutable {
    res->process(); // OK: we have mutable access to moved resource
};

// After move, original resource is empty
processor(); // Uses the captured unique_ptr
```

**Performance Considerations:**
- `mutable` has **zero overhead** - it's purely a semantic change
- Enables in-place modification of captured values, avoiding copies
- Dangerous with `[=]` capture-all: can lead to unintended state sharing
- Prefer explicit capture list `[x = std::move(y)]` with `mutable` for clarity

**Best Practices:**
- Use `mutable` sparingly - it's often a sign your lambda should be a named functor
- **Never** use `[=] mutable` - capture exactly what you need
- Document why mutability is required (e.g., "caches computation result")
- For multi-call state, consider a plain class with `operator()` instead
- When capturing move-only types, `mutable` is required to use them

**Comparison: mutable vs. Reference Capture:**
```cpp
int x = 0;
auto by_value = [x]() mutable { x++; };        // Modifies copy, original unchanged
auto by_ref   = [&x]() { x++; };              // Modifies original, no mutable needed

by_value(); by_value();
std::cout << x; // Still 0

by_ref(); by_ref();
std::cout << x; // Now 2 - original was modified
```

### std::function: Type-Erased Wrapper

`std::function<R(Args…)>` can hold **any** callable: free function, functor, lambda, member function bound with `std::bind`.

```cpp
void registerCallback(std::function<void(int)> f);   // accepts *anything*

registerCallback([](int x){ std::cout << x; });      // lambda
registerCallback(std::bind(&Foo::onEvent, &obj, _1));
```

**Cost**
* One virtual call indirection.  
* Small buffer optimisation (≈ 16-24 bytes); larger captures allocate on heap.  
* **Do not** create/destroy `std::function` inside hot loops.

### std::invoke: Universal Callable Invoker

**What is std::invoke?**
Introduced in C++17, `std::invoke` is a unified mechanism for calling **any callable object** with a consistent syntax. It abstracts away the differences between function pointers, member function pointers, member data pointers, and functors.

**Why It Exists:**
Before C++17, generic code had to specialize for different callable types:
```cpp
// Pre-C++17: Manual dispatch required
template<typename Callable, typename... Args>
decltype(auto) call(Callable&& c, Args&&... args) {
    if constexpr (std::is_member_function_pointer_v<Callable>) {
        return (std::forward<Args>(args).*c)(std::forward<Args>(args)...);
    } else {
        return std::forward<Callable>(c)(std::forward<Args>(args)...);
    }
}
```

`std::invoke` eliminates this boilerplate, providing a single, optimal call syntax for all callable types.

**Syntax:**
```cpp
#include <functional>

// Unified call syntax
std::invoke(callable, arg1, arg2, ...);
```

**Comprehensive Examples:**

1. **Regular Functions:**
```cpp
void print(int x) { std::cout << x; }
std::invoke(print, 42); // Equivalent to print(42)
```

2. **Member Functions:**
```cpp
struct Widget {
    void configure(int value) { /*...*/ }
};

Widget w;
std::invoke(&Widget::configure, w, 100);  // Equivalent to w.configure(100)
std::invoke(&Widget::configure, &w, 100); // Also works with pointer
```

3. **Member Data Access:**
```cpp
struct Point { int x, y; };
Point p{10, 20};
int x_val = std::invoke(&Point::x, p); // Equivalent to p.x
```

4. **Lambda and Functors:**
```cpp
auto lambda = [](int a, int b) { return a + b; };
std::invoke(lambda, 3, 4); // Equivalent to lambda(3, 4)
```

**Advanced Use Cases:**

1. **Generic Algorithms:**
```cpp
template<typename Container, typename Predicate>
void process_elements(Container& c, Predicate pred) {
    for (auto& elem : c) {
        // Works for functions, member functions, lambdas
        if (std::invoke(pred, elem)) {
            // Process element
        }
    }
}

// Usage with different callables
process_elements(vec, [](int x){ return x > 0; });          // Lambda
process_elements(vec, &SomeClass::is_valid, instance);      // Member function
process_elements(vec, std::greater{});                      // Functor
```

2. **Reflection-Like Patterns:**
```cpp
template<typename T, typename... Args>
void log_and_invoke(std::string_view name, T&& callable, Args&&... args) {
    std::cout << "Calling " << name << "...\n";
    auto result = std::invoke(std::forward<T>(callable), std::forward<Args>(args)...);
    std::cout << "Done\n";
    return result;
}
```

**Performance Characteristics:**
- **Zero overhead**: `std::invoke` is defined as a "magic" function that compilers optimize perfectly
- Assembly output is identical to direct invocation in optimized builds
- Enables inlining across all callable types
- SFINAE-friendly: `std::is_invocable_v<Callable, Args...>` can test callability at compile-time

**Best Practices:**
- **Always** use `std::invoke` in generic template code instead of manual `operator()` calls
- Combine with `std::apply` for tuple argument unpacking
- Use `std::is_invocable` to constrain templates instead of `std::is_callable`
- In C++20, prefer `std::invoke_r<T>` when you need explicit return type conversion

### std::bind: Partial Function Application

**What is std::bind?**
`std::bind` (C++11) creates a new callable by "binding" arguments to an existing callable. It's a form of **partial function application** that lets you fix some parameters while leaving others free.

**Core Concept:**
```cpp
#include <functional>

// Original function
void print_sum(int a, int b, int c) {
    std::cout << a + b + c << "\n";
}

// Bind first two arguments, leaving third free
auto add_10_20 = std::bind(print_sum, 10, 20, std::placeholders::_1);
add_10_20(30); // Prints 60 (10+20+30)
```

**Syntax and Placeholders:**
```cpp
using namespace std::placeholders;  // For _1, _2, _3,...

auto fn = std::bind(callable, arg1, arg2, _1, _2, arg3, _3);
// _n represents the nth argument to be provided later
```

**Detailed Examples:**

1. **Binding Member Functions:**
```cpp
class Logger {
public:
    void log(std::string_view level, std::string_view msg) {
        std::cout << "[" << level << "] " << msg << "\n";
    }
};

Logger logger;
auto log_error = std::bind(&Logger::log, &logger, "ERROR", _1);
log_error("File not found"); // [ERROR] File not found
```

2. **Reordering Arguments:**
```cpp
void normal_order(int a, int b, int c) { /*...*/ }
auto reversed = std::bind(normal_order, _3, _2, _1);
reversed(1, 2, 3); // Calls normal_order(3, 2, 1)
```

3. **Binding with Smart Pointers:**
```cpp
auto log_info = std::bind(&Logger::log, std::make_shared<Logger>(), "INFO", _1);
// Logger lifetime managed by shared_ptr
```

**When to Use std::bind vs. Lambdas:**

**Lambda Equivalent (Preferred):**
```cpp
// Instead of:
auto bind_version = std::bind(&Widget::process, &w, _1, 10);

// Modern lambda (more readable, better performance):
auto lambda_version = [&w](int x) { w.process(x, 10); };
```

**Why Lambdas Are Superior:**
1. **Readability**: Lambda syntax is more intuitive and easier to debug
2. **Performance**: Lambdas are easier for compilers to optimize (better inlining)
3. **Type Safety**: `std::bind` can silently slice or convert types
4. **Perfect Forwarding**: Lambdas support `auto&&` parameters and `std::forward`
5. **Overload Resolution**: Lambdas handle overload sets correctly; `std::bind` struggles

**When std::bind Still Makes Sense:**
1. **Legacy Codebases**: Existing code may already use it extensively
2. **Very Complex Binding**: When you need to bind many arguments with complex placeholder patterns
3. **C++98/03 Compatibility**: `std::bind` was originally `boost::bind`

**Performance Trade-offs:**
- `std::bind` typically generates more template instantiations
- Lambda captures are more explicit and optimizable
- `std::bind` can introduce unnecessary copies if not used carefully
- Modern compilers optimize both well, but lambdas have a slight edge

**Best Practices:**
- **Prefer lambdas** in all new code (C++14 and later)
- Replace `std::bind` with lambdas during refactoring
- Use `std::bind` only when integrating with legacy systems
- Never mix `std::bind` with generic lambdas - it creates unreadable code
- If you must use `std::bind`, document the equivalent lambda for clarity

### Performance Deep Dive – Assembly & Benchmarks

Sorting 10 M `int` with `std::sort`:

| Comparator | Time (ms) | Relative |
|------------|-----------|----------|
| Function pointer | 820 | 4.4× slower |
| Functor / lambda | 186 | 1.0× baseline |

**Assembly comparison:**
```asm
// Function pointer version
call    cmp_ptr@plt        ; real call inside hot loop

// Functor version
cmp     DWORD PTR [rdi+4], eax   ; inlined comparison
```

**Take-away**: tiny comparator cost **explodes** when it is not inlined.

### Summary Table – Callback Types

| Callback Type    | Syntax / Example                             | Notes                                   |
| ---------------- | -------------------------------------------- | --------------------------------------- |
| Function Pointer | `void (*cb)(int)`                            | Works with plain functions              |
| Function Object  | `struct Functor { void operator()(int){} };` | Can store state, works with templates   |
| Lambda           | `[](int x){}`                                | Can capture variables, concise          |
| `std::function`  | `std::function<void(int)> cb`                | Most flexible, wraps all callable types |

---

## 3. Customizability through Template Parameters

C++ templates allow you to write **generic, flexible, and reusable code**. With templates, the **caller can customize types, behaviors, or policies** without modifying the implementation. This approach enables both **compile-time behavior injection** and **policy-based design**.

Templates enable two main types of customization:
1. **Behavior injection via callables** (function pointers, functors, lambdas)
2. **Behavior injection via types / policy classes** (compile-time customization)

### Templates with Callable Parameters

Templates can accept **callable objects** to define behavior dynamically.

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
#include <functional>

// Template accepting any callable as a callback
template <typename Compare>
void sortVector(std::vector<int>& v, Compare comp) {
    std::sort(v.begin(), v.end(), comp);
}

int main() {
    std::vector<int> v = {1, 5, 3, 2};

    // Lambda as a callback
    int factor = 10;
    sortVector(v, [factor](int a, int b) {
        return (a * factor) < (b * factor);
    });

    // Functor as a callback
    struct Descending {
        bool operator()(int a, int b) const { return a > b; }
    };
    sortVector(v, Descending());

    // Function pointer as a callback
    auto ascending = [](int a, int b) { return a < b; };
    sortVector(v, ascending);
}
```

**Key points:**
* **Callbacks** define how the template behaves: function pointers, functors, or lambdas
* Multiple parameters and references are fully supported
* **Type-safe** and efficient

### Templates with Types (Policy Classes)

Templates can accept **types** as parameters, allowing compile-time strategy injection.

```cpp
#include <vector>
#include <memory>
#include <iostream>

// Custom allocator policy
template <typename T>
class MyAllocator {
public:
    using value_type = T;
    MyAllocator() = default;

    T* allocate(std::size_t n) {
        std::cout << "Allocating " << n << " elements\n";
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    void deallocate(T* ptr, std::size_t n) {
        std::cout << "Deallocating " << n << " elements\n";
        ::operator delete(ptr);
    }
};

// Vector template with customizable allocator
template <typename T, typename Allocator = std::allocator<T>>
using MyVector = std::vector<T, Allocator>;

int main() {
    MyVector<int> defaultVec;                 // default allocator
    MyVector<int, MyAllocator<int>> customVec; // custom allocator
}
```

**Explanation:**
1. `MyAllocator<int>` is a **policy class** defining memory allocation strategy
2. `MyVector` delegates allocation to the **provided type**
3. This is a **compile-time customization**: the container relies on a type that implements a specific interface
4. The caller can inject **different behaviors** without modifying the template

### Why Template Customizability Matters

* **Extreme flexibility**: Fully change behavior at compile time without changing code
* **Compile-time efficiency**: Templates avoid runtime overhead (no virtual calls)
* **Reusability**: Same template can be used with different behaviors or policies
* **Stateful strategies**: Functors or lambdas can store state while maintaining efficiency
* **Supports modern C++ design patterns**: Implements **policy-based design** and compile-time dependency injection

---

## 4. Concepts (C++20)

### What are Concepts?

* **Concepts** allow you to **constrain template parameters**.
* They let the compiler check **at compile time** whether a type satisfies certain requirements before instantiating the template.
* Think of them as **"interfaces for templates"**—but enforced at compile time, without runtime overhead.

### Breaking down the example

```cpp
template <typename T>
concept Logger = requires(T t, std::string_view msg) {
    { t.log(msg) } -> std::same_as<void>;
};
```

**Explanation line by line:**

1. `template <typename T>` → this is a normal template parameter.
2. `concept Logger` → defines a **concept** named `Logger`.
3. `requires(T t, std::string_view msg) { ... }` → specifies the **requirements** a type `T` must fulfill to satisfy the concept.

   * `T t` → an object of type `T`.
   * `std::string_view msg` → a string message to pass.
4. `{ t.log(msg) } -> std::same_as<void>;` → requires that `T` must have a **member function `log` that takes a `std::string_view` and returns `void`**.

If `T` doesn't have such a function, the compiler will **reject it at compile time**, instead of generating a confusing template error later.

### How to use this concept

```cpp
#include <string>
#include <string_view>
#include <concepts>
#include <iostream>

template <typename T>
concept Logger = requires(T t, std::string_view msg) {
    { t.log(msg) } -> std::same_as<void>;
};

// A type that satisfies Logger
class ConsoleLogger {
public:
    void log(std::string_view msg) { std::cout << msg << "\n"; }
};

// Template function constrained by Logger concept
template <Logger L>
void doSomething(L& logger) {
    logger.log("Hello C++20 Concepts!");
}

int main() {
    ConsoleLogger logger;
    doSomething(logger); // Works
}
```

* `doSomething` will only accept types that **satisfy the Logger concept**.
* If you try to pass a type without a `log` method returning `void`, the code will **fail to compile**.

### Why this is useful

1. **Better error messages** – compiler tells you exactly which constraint is violated.
2. **Self-documenting templates** – anyone reading `template <Logger L>` immediately knows what the type needs to provide.
3. **Safer templates** – ensures correct usage at compile-time, avoiding runtime surprises.

**Summary:**

> `Concepts` = a way to enforce template requirements at compile-time.
> `Logger` concept = any type passed as a template argument **must have a `log` method that takes a `std::string_view` and returns `void`.**

---

## 5. Comparison: When to Use What

| Approach | Runtime Overhead | Flexibility | Testability | Compile-Time | Use Case |
|----------|------------------|-------------|-------------|--------------|----------|
| **Function Pointers** | None | Low | Fair | N/A | C interoperability, simple callbacks |
| **Callbacks (functors/lambdas)** | Type-erasure (`std::function`) | Very High | Good | N/A | Event handlers, small inline behaviors |
| **Templates** | None (compile-time) | Medium* | Harder† | Full | Performance-critical, policy-based design |
| **std::invoke** | None (direct call) | Very High | Excellent | Full | Generic callable handling |
| **std::bind** | Minimal overhead | Medium | Fair | Limited | Legacy code, complex binding |

*Flexibility: Behavior fixed at compile time  
†Testing: Requires template-heavy test frameworks or explicit instantiation

**Detailed Decision Matrix:**

| You want… | Free function | Functor | Lambda | std::function | Template |
|-----------|---------------|---------|--------|---------------|----------|
| State per call | ❌ | ✅ | ✅ | ✅ | ✅ |
| Inlining inside algorithm | ✅ | ✅ | ✅ | ❌ | ✅ |
| Overload set *as* a value | ❌ | ✅ | ❌* | ✅ | ✅ |
| Templated call *per object* | ❌ | ✅ | ✅ | ❌ | ✅ |
| Runtime polymorphism | ❌ | ❌ | ❌ | ✅ | ❌ |
| Cross-DLL / pure C | ✅ | ❌† | ❌† | ❌ | ❌ |
| Create at call site | N/A | manual | ✅ | N/A | N/A |

\* Use `overload` helper for overload sets in lambdas  
† Stateless lambda converts to function pointer with unary `+`

---

## 6. Modern C++ Best Practices

### ✅ Do's

- **Pass functors by value & templated**: Maximum inlining potential
- **Use `std::function` for callbacks**: Type-erased, flexible
- **Leverage lambdas**: Capture state concisely with `[&]` or `[=]`
- **Apply `std::move`**: Efficiently transfer ownership of callables
- **Document with `[[nodiscard]]`**: For factory functions
- **Concepts (C++20)**: Constrain template parameters (see [Concepts section](#4-concepts-c20))
- **Mark callable `noexcept`**: Algorithms generate faster code
- **Use `std::unique_ptr` for ownership**: Clear, safe, self-documenting
- **Use `std::invoke` in generic code**: Uniform callable syntax
- **Specify lambda return types**: For complex lambdas or public APIs

### ❌ Don'ts

- **Raw pointers for ownership**: Use smart pointers
- **Static/global state**: Makes testing impossible and causes re-entrancy issues
- **Overuse `std::function`**: For simple templates, prefer direct callable types
- **Capture all by reference `[&]`**: Be explicit about captures
- **Create/destroy `std::function` in hot loops**: Allocation overhead dominates
- **Ignore lifetime when capturing by reference**: Dangling reference bugs are silent killers
- **Forget `constexpr` for compile-time callables**: Enables compile-time evaluation
- **Use `std::bind` in new code**: Prefer lambdas for clarity and performance
- **Use `[=] mutable`**: Capture only what you need, explicitly

---

## 7. Real-World Idiomatic Patterns

### A. Custom hash for unordered_map
```cpp
struct PairHash {
    std::size_t operator()(std::pair<int,int> p) const noexcept
    {
        auto h1 = std::hash<int>{}(p.first);
        auto h2 = std::hash<int>{}(p.second);
        return h1 ^ (h2 + 0x9e3779b9 + (h1<<6) + (h1>>2));
    }
};
std::unordered_map<std::pair<int,int>, std::string, PairHash> table;
```

### B. Thread-pool task
```cpp
pool.submit([data = std::move(big)](){
    process(data);
});
```

### C. Benchmark harness
```cpp
template<class F>
double timeit(F&& f, std::size_t iterations)
{
    auto t0 = std::chrono::steady_clock::now();
    for (std::size_t i = 0; i < iterations; ++i) f();
    return std::chrono::duration<double>(std::chrono::steady_clock::now() - t0).count();
}
```

### D. Visitor with overload set (C++17)
```cpp
auto printer = overload{
    [](int  v){ std::cout << "int "  << v; },
    [](auto&& v){ std::cout << "other"; }
};
std::visit(printer, variant_value);
```

### E. Resource-owning functor (RAII + behaviour)
```cpp
class ImageFilter {
    cv::Mat kernel;
public:
    explicit ImageFilter(cv::Mat k) : kernel(std::move(k)) {}
    cv::Mat operator()(const cv::Mat& src) const
    {
        cv::Mat dst;
        cv::filter2D(src, dst, -1, kernel);
        return dst;
    }
};
```

### F. Compile-time factorial
```cpp
struct Factorial {
    constexpr std::uint64_t operator()(unsigned n) const noexcept
    {
        return n ? n * (*this)(n-1) : 1;
    }
};

constexpr Factorial fac;
static_assert(fac(5) == 120);
```

### G. std::invoke with Visitor Pattern
```cpp
template<typename Variant, typename... Visitors>
void visit_variant(Variant&& v, Visitors&&... visitors) {
    auto overload_set = overload{std::forward<Visitors>(visitors)...};
    std::visit([&](auto&& arg) {
        std::invoke(overload_set, std::forward<decltype(arg)>(arg));
    }, std::forward<Variant>(v));
}
```

### H. mutable Lambda for Unique Resource
```cpp
auto resource_processor = [resource = std::make_unique<ExpensiveResource>()]() mutable {
    // resource is non-const inside mutable lambda
    resource->process();
    return std::move(resource); // Can transfer ownership out
};
```

---

## 8. Decision Tree – Which One Should I Pick?

```text
Need state?
├─ No ─→ Use free function (or static member) ─→ Done
│
└─ Yes ─→ Is it tiny & local?
          ├─ Yes ─→ Use lambda
          │          ├─ Stateless? → Can convert to function pointer with +
          │          └─ Needs to outlive scope? → Capture by value [=]
          │
          └─ No ─→ Need overloads / reuse across many TU?
                   ├─ Yes ─→ Use named functor (class/struct)
                   │          ├─ Make operator() const
                   │          └─ Mark noexcept if possible
                   │
                   └─ No ─→ Will it be stored / type-erased?
                            ├─ Yes ─→ Use std::function (watch for heap usage)
                            └─ No ─→ Keep template parameter (fastest)
```

**Additional Heuristics:**

- **Library API**: Expose both template (header-only) and `std::function` (stable ABI) versions
- **Plugin system**: Use function pointers + C API + trampolines
- **Testing**: Templates need more boilerplate; `std::function` is easier to mock
- **Performance budget**: >1M calls/sec → templates; <10K calls/sec → any approach works

---

## 9. Complete Example – Flexible Logger System

```cpp
// Modern, flexible logger using compile-time customization
#include <iostream>
#include <string_view>
#include <memory>
#include <functional>

// Concept to enforce logger interface
template <typename T>
concept Logger = requires(T t, std::string_view msg) {
    { t.log(msg) } -> std::same_as<void>;
};

// Generic logger that accepts any callable
class LoggerSystem {
public:
    explicit LoggerSystem(std::function<void(std::string_view)> logFn)
        : logFn_(std::move(logFn)) {}

    void process(const std::string& data) {
        logFn_("[INFO] Processing: " + data);
        // ... processing logic ...
        if (data.empty()) {
            logFn_("[WARN] Empty data received");
        }
    }

private:
    std::function<void(std::string_view)> logFn_;
};

// Custom logger implementations
struct ConsoleLogger {
    void operator()(std::string_view msg) const {
        std::cout << msg << "\n";
    }
};

struct FileLogger {
    std::string filename;
    explicit FileLogger(std::string fname) : filename(std::move(fname)) {}
    
    void operator()(std::string_view msg) const {
        // In real code: write to file
        std::cout << "[File " << filename << "] " << msg << "\n";
    }
};

// Stateful logger with mutable
struct CountingLogger {
    int count = 0;
    void operator()(std::string_view msg) {
        std::cout << "[" << ++count << "] " << msg << "\n";
    }
};

// Usage
int main() {
    // Lambda logger with explicit return type
    LoggerSystem lambdaLogger([](std::string_view msg) -> void {
        std::cerr << "λ: " << msg << "\n";
    });
    lambdaLogger.process("test data");

    // Functor logger
    LoggerSystem consoleLogger(ConsoleLogger{});
    consoleLogger.process("more data");

    // Stateful functor
    LoggerSystem fileLogger(FileLogger{"app.log"});
    fileLogger.process("critical data");

    // Counting logger (demonstrates stateful functor)
    CountingLogger counter;
    auto counting_wrapper = [&counter](std::string_view msg) -> void {
        counter(msg);
    };
    LoggerSystem countingLogger(counting_wrapper);
    countingLogger.process("first");
    countingLogger.process("second");
}
```

**Key points:**
* `LoggerSystem` is **decoupled** from logging implementation
* **Type-erasure** via `std::function` allows runtime flexibility
* **Concepts** could replace `std::function` for compile-time checks if only templates are used
* Easy to **test** by injecting mock loggers

---

## 10. Key Takeaways

1. **Callbacks Enable Reusability**: Pass behavior as callable objects instead of hardcoding logic.
2. **Modern C++ = Multiple Tools**: Function pointers, functors, lambdas, and templates are complementary.
3. **Lambdas are King**: For 90% of callbacks, lambdas provide the best balance of clarity and power.
4. **Performance Matters**: Use templates when virtual calls are unacceptable (can be 4× faster).
5. **Functors are Classes**: Every lambda is just a struct with `operator()` – internalise this and choices become obvious.
6. **No Global State**: "Piping state" means carrying it inside the callable, not in globals.
7. **Inlining is a Big Deal**: A function pointer can make your code 4× slower in tight loops.
8. **Concepts (C++20)**: Use concepts to enforce compile-time contracts and improve error messages.
9. **`std::function` Flexibility**: Use when you need to store heterogeneous callables, but avoid in hot loops.
10. **Design for Testing**: Make callables easily mockable from day one.
11. **std::invoke for Generic Code**: Use it to create truly generic callable handlers.
12. **Lambda Return Types**: Specify them for complex logic, public APIs, and compile-time guarantees.
13. **mutable Use with Care**: It's powerful but often indicates a design that should be a functor class.


---

## Appendix: Mechanical Transformation Rules

### From Function to Functor
```cpp
// Before
bool compare(int a, int b) { return a < b; }

// After
struct Compare {
    bool operator()(int a, int b) const { return a < b; }
};
```

### From Function to Template
```cpp
// Before
void doWork(bool (*cb)(int), int n);

// After
template<class Callback>
void doWork(Callback cb, int n);
```

### From Functor to Lambda
```cpp
// Before
struct Adder { int bias; int operator()(int x) const { return x + bias; } };

// After
auto adder = [bias = 10](int x){ return x + bias; };
```

### From std::bind to Lambda
```cpp
// Before
auto log = std::bind(&Logger::log, &logger, "INFO", _1);

// After
auto log = [&logger](std::string_view msg) { logger.log("INFO", msg); };
```

### From Direct Call to std::invoke
```cpp
// Before
template<typename F, typename... Args>
decltype(auto) call(F&& f, Args&&... args) {
    return std::forward<F>(f)(std::forward<Args>(args)...);
}

// After
template<typename F, typename... Args>
decltype(auto) call(F&& f, Args&&... args) {
    return std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
}
// Now works with member functions too!
```