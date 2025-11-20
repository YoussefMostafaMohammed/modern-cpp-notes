# **`inline`, `using`, and `noexcept`**

## **Table of Contents**

1. [Introduction](#introduction)
2. [Part 1: Template Parameters and Type Aliases](#part-1-template-parameters-and-type-aliases)
   - [The TelemetryPolicy Example](#the-telemetrypolicy-example)
   - [Template Parameter Evaluation](#template-parameter-evaluation)
3. [Part 2: `inline` Variables - Solving the ODR Nightmare](#part-2-inline-variables-solving-the-odr-nightmare)
   - [What `inline` Does for Variables](#what-inline-does-for-variables)
   - [The Problem: Static Initialization Hell](#the-problem-static-initialization-hell)
   - [Before C++17: Two Bad Options](#before-c17-two-bad-options)
   - [After C++17: The `inline` Solution](#after-c17-the-inline-solution)
4. [Part 3: `using` Type Aliases - Replacing `typedef`](#part-3-using-type-aliases-replacing-typedef)
   - [The Readability Problem with `typedef`](#the-readability-problem-with-typedef)
   - [Template Aliases: The Game Changer](#template-aliases-the-game-changer)
   - [How Template Aliases Are Evaluated](#how-template-aliases-are-evaluated)
5. [Part 4: `noexcept` - The Savior of Move Semantics](#part-4-noexcept-the-savior-of-move-semantics)
   - [What `noexcept` Does](#what-noexcept-does)
   - [The `throw()` Disaster (Pre-C++11)](#the-throw-disaster-pre-c11)
   - [Three Critical Reasons for `noexcept`](#three-critical-reasons-for-noexcept)
6. [Part 5: Move Semantics and Exception Safety](#part-5-move-semantics-and-exception-safety)
   - [Why `std::vector` Refuses Throwing Moves](#why-stdvector-refuses-throwing-moves)
   - [The Strong Exception Guarantee](#the-strong-exception-guarantee)
   - [Complete Code Example with Output](#complete-code-example-with-output)
7. [Part 6: The Moved-From State](#part-6-the-moved-from-state)
   - [Why `nullptr` is Mandatory](#why-nullptr-is-mandatory)
   - [The Double-Free Prevention Rule](#the-double-free-prevention-rule)
   - [Exception Safety in Move Constructors](#exception-safety-in-move-constructors)
8. [Part 7: `std::move_if_noexcept` - Conditional Move vs Copy](#part-7-stdmove_if_noexcept---conditional-move-vs-copy)
   - [What It Does](#what-it-does)
   - [Implementation Details](#implementation-details)
   - [Where the Standard Library Uses It](#where-the-standard-library-uses-it)
   - [Performance Impact](#performance-impact)
9. [Part 8: Best Practices and Guidelines](#part-8-best-practices-and-guidelines)
10. [Part 9: Q&A Summary](#part-9-qa-summary)
11. [Appendix: Complete Code Examples](#appendix-complete-code-examples)

---

## **Introduction**

Modern C++ (C++11 and later) introduced several keywords that fundamentally changed how we write safe, performant code. This guide explores three critical features: `inline` variables, `using` type aliases, and `noexcept` specifications. Each was invented to solve painful problems that plagued C++ for decades. We'll examine real code, common misconceptions, and the performance implications of getting these right.

---

## **Part 1: Template Parameters and Type Aliases**

### **The TelemetryPolicy Example**

```cpp
template<Enums::TelemetrySrc_enum SRC, std::string_view UNIT = "%">
struct TelemetryPolicy {
    static constexpr Enums::TelemetrySrc_enum context = SRC;
    static constexpr std::string_view unit = UNIT;
    static constexpr float WARNING = 75.0f;
    static constexpr float CRITICAL = 90.0f;
    
    static constexpr Enums::SeverityLvl_enum inferSeverity(float val) noexcept {
        return (val >= CRITICAL) ? Enums::SeverityLvl_enum::CRITICAL
             : (val >= WARNING)  ? Enums::SeverityLvl_enum::WARNING
             : Enums::SeverityLvl_enum::INFO;
    }
};

// Type aliases for each sensor
using CPU = TelemetryPolicy<Enums::TelemetrySrc_enum::CPU>;  // unit = "%"
using GPU = TelemetryPolicy<Enums::TelemetrySrc_enum::GPU>;  // unit = "%"

// Special RAM unit
inline constexpr std::string_view ram_unit = "MB";
using RAM = TelemetryPolicy<Enums::TelemetrySrc_enum::RAM, ram_unit>;
```

This code demonstrates a policy-based design pattern using:
- **Non-type template parameters** (`Enums::TelemetrySrc_enum`, `std::string_view`)
- **Default template arguments**
- **Type aliases** (`using` declarations)
- **`inline` variables**

### **Template Parameter Evaluation: A Common Confusion**

**Question:** When we write `template<typename T> using Vec = std::vector<T>;` and then `Vec<int> intVector;`, shouldn't it evaluate to `std::vector<T><int>`?

**Answer:** No. The `<int>` **replaces** the `<T>`, it doesn't append. The compiler performs **template argument substitution**:

```cpp
// Step 1: The alias template
template<typename T>
using Vec = std::vector<T>;

// Step 2: Usage
Vec<int> intVector;

// Step 3: What the compiler does
// Vec<int>  →  std::vector<int>
// The <int> substitutes for T in the right-hand side
```

**Result:** `Vec<int>` is **exactly** `std::vector<int>`, not `std::vector<T><int>` (which is invalid syntax).

---

## **Part 2: `inline` Variables - Solving the ODR Nightmare**

### **What `inline` Does for Variables**

```cpp
inline constexpr std::string_view ram_unit = "MB";
```

The `inline` keyword (extended to variables in C++17) tells the compiler:  
*"This variable may appear identically in multiple translation units (.cpp files); merge them into a single definition."*

### **The Problem: Static Initialization Hell**

Before C++17, you had two terrible options for header-only constants:

#### **Option A: Header-only definition (Link-time disaster)**
```cpp
// telemetry.h
constexpr std::string_view ram_unit = "MB";  // ODR violation!
```

**Problem:** If two `.cpp` files include this, you get **multiple definitions** → linker error (`LNK2005` on MSVC, `multiple definition` on GCC).

#### **Option B: Split declaration/definition (Brittle and tedious)**
```cpp
// telemetry.h
extern const std::string_view ram_unit;  // Declaration

// telemetry.cpp
constexpr std::string_view ram_unit = "MB";  // Definition
```

**Problems:**
- Requires both `.h` and `.cpp` files
- Easy to forget updating one when changing the other
- Breaks header-only library design
- `constexpr` variables in headers still required definitions in some contexts

### **After C++17: The `inline` Solution**

```cpp
// Single header file, problem solved!
inline constexpr std::string_view ram_unit = "MB";
```

**Benefits:**
- **ODR-safe**: Identical definitions across translation units are merged
- **Header-only friendly**: Perfect for library development
- **Zero overhead**: No runtime cost, compile-time guarantee
- **Cleaner codebase**: No more split files for constants

**In your code:** `ram_unit` can be defined in a header included by 50 `.cpp` files, but the final binary contains exactly **one** instance—guaranteed by the compiler.

---

## **Part 3: `using` Type Aliases - Replacing `typedef`**

### **The Readability Problem with `typedef`**

Before C++11, `typedef` had unreadable **inside-out** syntax:

```cpp
// Old way (still valid, but don't)
typedef TelemetryPolicy<Enums::TelemetrySrc_enum::CPU> CPU;  // name in middle!
typedef std::vector<std::map<std::string, std::function<void()>>> CallbackMap;
//     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ "where's the name?"
```

### **The Game Changer: `using` Aliases**

```cpp
// Left-to-right, natural syntax
using CPU = TelemetryPolicy<Enums::TelemetrySrc_enum::CPU>;
using CallbackMap = std::vector<std::map<std::string, std::function<void()>>>;
```

**Benefits:**
- **Readability**: `using Name = Type;` reads like variable assignment
- **Template aliases**: `typedef` **cannot** be templated; `using` can:
  ```cpp
  template<typename T>
  using Vec = std::vector<T>;
  
  Vec<int> intVector;  // → std::vector<int>
  Vec<std::string> stringVector;  // → std::vector<std::string>
  ```

### **C++98 Workaround (The Horror)**
```cpp
// C++98: The "typedef inside a struct" hack
template<typename T>
struct Vec {
    typedef std::vector<T> type;
};

Vec<int>::type intVector;  // Ugh, extra ::type needed!
```

---

## **Part 4: `noexcept` - The Savior of Move Semantics**

### **What `noexcept` Does**

```cpp
void safe_function() noexcept;  // I promise: no exceptions will escape
```

Marks a function as **guaranteed not to throw exceptions**. Violating this calls `std::terminate()` immediately—no hidden runtime cost.

### **The `throw()` Disaster (Pre-C++11)**

C++98 had a broken feature called **exception specifications**:
```cpp
void old_func() throw();  // Means "won't throw" BUT...
```

**Problems:**
- **Runtime overhead**: Called `std::unexpected()` on violation
- **Runtime-checked**, not compile-time
- **No optimization benefits**
- **Deprecated in C++11**, **removed in C++17**

### **Three Critical Reasons for `noexcept`**

#### **1. Enable Move Semantics**
`std::vector` can only use move constructors if they're `noexcept`. Why?
- If moving might throw, `vector::resize` can't guarantee exception safety
- It would have to fall back to slower copy constructors to keep the original data intact if a throw occurs
- `noexcept` promises it's safe to move, so `vector` can optimize

#### **2. Compiler Optimization**
- Skip generating exception handling tables → smaller binary
- Eliminate `try/catch` overhead
- More aggressive code reordering

#### **3. Clear API Contracts**
Self-documenting and enforced at compile time.

---

## **Part 5: Move Semantics and Exception Safety**

### **Why `std::vector` Refuses Throwing Moves**

**The Strong Exception Guarantee:**  
`std::vector::push_back` promises: *If an exception is thrown, the vector remains in its original state.*

### **The Dangerous Scenario: Throwing Move**

```cpp
// vector has: [Resource("first"), Resource("second")] in old_buffer

// Step 1: Move-construct first element
new_buffer[0] = std::move(old_buffer[0]);
// old_buffer[0].data is now nullptr!

// Step 2: Move-construct second element (THROWS!)
new_buffer[1] = std::move(old_buffer[1]);  // Exception thrown mid-move

// Result:
// - old_buffer[0] is a zombie (already moved-from)
// - new_buffer[0] owns the stolen data
// - Cannot restore original state → Strong guarantee violated
```

### **The Safe Scenario: Copy**

```cpp
// If move is not noexcept, vector falls back to copy

// Step 1: Copy-construct first element
new_buffer[0] = old_buffer[0];  // old_buffer[0] unchanged

// Step 2: Copy-construct second element (THROWS!)
new_buffer[1] = old_buffer[1];  // Exception!

// Result:
// - old_buffer unchanged (still valid)
// - Destroy new_buffer → original state intact
// - Strong guarantee upheld
```

### **Complete Code Example**

```cpp
class Resource {
    std::string name;
    int* data;

public:
    // NOT noexcept!
    Resource(Resource&& other) noexcept(false)
        : name(std::move(other.name)), data(other.data) {
        other.data = nullptr;
        if (name.find("THROW") != std::string::npos) {
            throw std::runtime_error("Move failed!");
        }
    }
};

void testVector() {
    std::vector<Resource> vec;
    vec.emplace_back("first");
    vec.emplace_back("second");
    
    // This reallocation will use COPIES, not moves
    vec.emplace_back("THIRD");
}
```

**Output proving it uses copies:**
```
[COPY] first_copy (slow but safe)
[COPY] second_copy (slow but safe)
```

---

## **Part 6: The Moved-From State**

### **Why `nullptr` is Mandatory**

```cpp
Resource(Resource&& other) noexcept
    : data(other.data)  // Steal pointer (shallow copy)
{
    other.data = nullptr;  // Crucial!
}
```

**Two reasons:**

#### **1. Prevent Double-Free**
```cpp
// BAD: If we don't null it
~Resource() { delete[] data; }

// Scenario:
Resource a("test");
Resource b(std::move(a));  // a.data and b.data both point to same memory
// ~a() called → delete[] data (freed)
// ~b() called → delete[] data (double-free!) → CRASH
```

#### **2. Exception Safety**
If an exception is thrown after moving but before completing the operation, the moved-from object must be in a **destructible state**:
- `delete[] nullptr` is **safe** (no-op)
- `delete[] stolen-pointer` is **catastrophic** (double-free, heap corruption)

### **The State Timeline**

```cpp
Resource original("data");
// original.data = 0xABC, original.name = "data"

Resource moved_to = std::move(original);
// moved_to.data   = 0xABC (stolen)
// moved_to.name   = "data" (stolen)
// original.data   = nullptr (safe state)
// original.name   = "" (moved-from string)
```

---

## **Part 7: `std::move_if_noexcept` - Conditional Move vs Copy**

### **What It Does**

`std::move_if_noexcept` is a **conditional cast**:
- **If move is `noexcept`** → returns `T&&` (rvalue reference, moves)
- **If move can throw** → returns `const T&` (lvalue reference, copies)

### **Implementation Details**

```cpp
namespace std {
template<typename T>
typename std::conditional<
    std::is_nothrow_move_constructible<T>::value,  // Compile-time check
    T&&,                                           // If true: move
    const T&                                       // If false: copy
>::type
move_if_noexcept(T& x) noexcept {
    return static_cast<typename std::conditional<...>::type>(x);
}
}
```

### **How It Works**

```cpp
struct MayThrow {
    MayThrow(MayThrow&&) { /* might throw */ }  // Not noexcept!
};

struct Safe {
    Safe(Safe&&) noexcept { }  // noexcept!
};

MayThrow mt;
auto result1 = std::move_if_noexcept(mt);  // Returns const MayThrow& → COPIES

Safe s;
auto result2 = std::move_if_noexcept(s);   // Returns Safe&& → MOVES
```

### **Where the Standard Library Uses It**

#### **1. `std::swap`**
```cpp
template<typename T>
void swap(T& a, T& b) {
    T temp = std::move_if_noexcept(a);
    a = std::move_if_noexcept(b);
    b = std::move_if_noexcept(temp);
}
```

#### **2. `std::vector::resize` (Simplified)**
```cpp
for (size_t i = 0; i < old_size; ++i) {
    new_data[i] = std::move_if_noexcept(old_data[i]);
    // Moves if noexcept, copies otherwise
}
```

### **Performance Impact**

```cpp
struct Heavy {
    int* huge_array = new int[1000000];
    Heavy(Heavy&&) noexcept = default;  // noexcept move
};

struct LightButThrowy {
    std::string s;
    // Move might throw (not noexcept)
};

std::vector<Heavy> vec1;
vec1.push_back(Heavy{});  // Reallocation: MOVES → ~0.1ms

std::vector<LightButThrowy> vec2;
vec2.push_back(LightButThrowy{});  // Reallocation: COPIES → ~50ms
```

---

## **Part 8: Best Practices and Guidelines**

1. **Always mark move constructors `noexcept`** unless they can legitimately throw
2. **Use `inline` for header-only constants** (C++17)
3. **Prefer `using` over `typedef`** for all type aliases
4. **Mark simple getters/computations `noexcept`**
5. **Understand the strong exception guarantee** when designing containers
6. **Never leave moved-from objects in a state that could cause double-free**

---

## **Part 9: Q&A Summary**

### **Q1: Why use `inline` on variables?**
**A:** Solves the ODR violation problem, enabling header-only libraries with compile-time constants without linker errors.

### **Q2: What's wrong with `typedef`?**
**A:** Unreadable inside-out syntax; cannot be templated; requires ugly `::type` workarounds.

### **Q3: Is `noexcept` only for `constexpr` functions?**
**A:** NO. They are completely independent. `constexpr` means compile-time evaluable; `noexcept` means never throws at runtime.

### **Q4: Why does moved-from data become `nullptr`?**
**A:** To prevent double-free in destructors and ensure exception safety. `delete[] nullptr` is safe.

### **Q5: How does `std::move_if_noexcept` work?**
**A:** Compile-time check: if move is `noexcept`, returns `T&&` (moves); else returns `const T&` (copies).

### **Q6: Why won't `vector` use my move constructor?**
**A:** Your move constructor is not `noexcept`. `vector` falls back to copies to maintain the strong exception guarantee.

---

## **Appendix: Complete Code Examples**

### **Example 1: The Telemetry System**

```cpp
#include <string_view>

namespace Enums {
    enum class TelemetrySrc_enum { CPU, GPU, RAM };
    enum class SeverityLvl_enum { INFO, WARNING, CRITICAL };
}

template<Enums::TelemetrySrc_enum SRC, std::string_view UNIT = "%">
struct TelemetryPolicy {
    static constexpr Enums::TelemetrySrc_enum context = SRC;
    static constexpr std::string_view unit = UNIT;
    static constexpr float WARNING = 75.0f;
    static constexpr float CRITICAL = 90.0f;
    
    static constexpr Enums::SeverityLvl_enum inferSeverity(float val) noexcept {
        return (val >= CRITICAL) ? Enums::SeverityLvl_enum::CRITICAL
             : (val >= WARNING)  ? Enums::SeverityLvl_enum::WARNING
             : Enums::SeverityLvl_enum::INFO;
    }
};

// Type aliases
using CPU = TelemetryPolicy<Enums::TelemetrySrc_enum::CPU>;
using GPU = TelemetryPolicy<Enums::TelemetrySrc_enum::GPU>;
inline constexpr std::string_view ram_unit = "MB";
using RAM = TelemetryPolicy<Enums::TelemetrySrc_enum::RAM, ram_unit>;
```

### **Example 2: Move Semantics Demo**

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>

class Resource {
private:
    std::string name;
    int* data;

public:
    Resource(std::string n, int size = 100) 
        : name(std::move(n)), data(new int[size]) {
        std::cout << "  [CONSTRUCT] " << name << "\n";
    }

    // COPY (noexcept)
    Resource(const Resource& other) noexcept 
        : name(other.name + "_copy"), data(new int[100]) {
        std::cout << "  [COPY] " << name << " (slow but safe)\n";
    }

    // MOVE (NOT noexcept)
    Resource(Resource&& other) noexcept(false)  // Might throw!
        : name(std::move(other.name)), data(other.data) {
        std::cout << "  [MOVE] " << name << " (fast but might throw)\n";
        other.data = nullptr;
        if (name.find("THROW") != std::string::npos) {
            throw std::runtime_error("Move failed!");
        }
    }

    ~Resource() { 
        std::cout << "  [DESTROY] " << name << "\n";
        delete[] data; 
    }

    Resource& operator=(const Resource&) = delete;
    Resource& operator=(Resource&&) = delete;
};

void testVector() {
    std::vector<Resource> vec;
    std::cout << "\nAdding first element:\n";
    vec.emplace_back("first");
    
    std::cout << "\nAdding second element (triggers reallocation):\n";
    vec.emplace_back("second");
    
    std::cout << "\nFinal size: " << vec.size() << "\n";
}

int main() { testVector(); }
```

**Expected Output:**
```
Adding first element:
  [CONSTRUCT] first

Adding second element (triggers reallocation):
  [CONSTRUCT] second
  [COPY] first_copy (slow but safe)  // ← Copies instead of moves!
  [DESTROY] first

Final size: 2
```

### **Example 3: The `noexcept` Fix**

```cpp
// Same as above, but change move constructor to:
Resource(Resource&& other) noexcept  // Now noexcept!
    : name(std::move(other.name)), data(other.data) {
    std::cout << "  [MOVE] " << name << " (fast AND safe)\n";
    other.data = nullptr;
}
```

**Now the output shows:**
```
[MOVE] first (fast AND safe)  // ← Moves instead of copies!
```

---

## **Final Summary Table**

| Feature | Problem Solved | Before (The Pain) | After (The Relief) | Key Rule |
|---------|----------------|-------------------|-------------------|----------|
| **`inline` variables** | ODR violations, header-only libraries | Split .h/.cpp files, linker errors | Single header definition | Use for all header constants |
| **`using` aliases** | `typedef`'s unreadable syntax, no templates | `typedef A<B> C;` (backwards) | `using C = A<B>;` (clear) | Always prefer over `typedef` |
| **`noexcept`** | `throw()`'s overhead, move semantics disabled | Runtime-checked, slow, broken | Compile-time, optimizable, fast | Always mark move ctors noexcept |

