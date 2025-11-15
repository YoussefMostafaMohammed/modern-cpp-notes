# **C++ Templates & Name Mangling: A Comprehensive Guide**

*From basic instantiation to SFINAE and type traits—understanding the compile-time machinery that powers generic programming.*

---

## **📚 Table of Contents**

### **Chapter 1: Template Fundamentals**
- [1.1 What Are Templates? - Compile-Time Code Generation](#11-what-are-templates---compile-time-code-generation)
- [1.2 Template Terminology - Parameters, Arguments & Specializations](#12-template-terminology---parameters-arguments--specializations)

### **Chapter 2: Template Parameter Lists**
- [2.1 `class` vs `typename` - The Historical Distinction](#21-class-vs-typename---the-historical-distinction)

### **Chapter 3: Template Specialization**
- [3.1 Full/Explicit Specialization - Complete Overrides](#31-fullexplicit-specialization---complete-overrides)
- [3.2 Partial Specialization - Pattern Matching for Classes](#32-partial-specialization---pattern-matching-for-classes)

### **Chapter 4: Template Instantiation & Deduction**
- [4.1 Instantiation - From Template to Concrete Code](#41-instantiation---from-template-to-concrete-code)
- [4.2 Argument Deduction - How the Compiler Infers Types](#42-argument-deduction---how-the-compiler-infers-types)
- [4.3 Return Type Deduction - `auto` & `decltype(auto)`](#43-return-type-deduction---auto--decltypeauto)

### **Chapter 5: Reference Collapsing & Perfect Forwarding**
- [5.1 Reference Declaration - The Rules](#51-reference-declaration---the-rules)
- [5.2 Reference Collapsing - The Four Combinations](#52-reference-collapsing---the-four-combinations)
- [5.3 `std::forward` - Preserving Value Categories](#53-stdforward---preserving-value-categories)

### **Chapter 6: Type Traits & Metaprogramming**
- [6.1 What Are Type Traits? - Type Queries & Transformations](#61-what-are-type-traits---type-queries--transformations)
- [6.2 Standard Type Traits Categories](#62-standard-type-traits-categories)
- [6.3 `std::remove_reference` & `std::forward` - Behind the Scenes](#63-stdremove_reference--stdforward---behind-the-scenes)

### **Chapter 7: SFINAE - Substitution Failure Is Not An Error**
- [7.1 SFINAE Basics - Overload Resolution Escape Hatch](#71-sfinae-basics---overload-resolution-escape-hatch)
- [7.2 SFINAE in Practice - `std::enable_if`](#72-sfinae-in-practice---stdenable_if)
- [7.3 C++20 Concepts - SFINAE Done Right](#73-c20-concepts---sfinae-done-right)

### **Chapter 8: Name Mangling - The Linker's Language**
- [8.1 What is Name Mangling?](#81-what-is-name-mangling)
- [8.2 Template Name Mangling Examples](#82-template-name-mangling-examples)
- [8.3 Reading Mangled Names - `c++filt`](#83-reading-mangled-names---c++filt)

### **Chapter 9: Best Practices & Common Pitfalls**
- [9.1 Template Best Practices](#91-template-best-practices)
- [9.2 Common Template Errors](#92-common-template-errors)

---

## **Chapter 1: Template Fundamentals**

### **1.1 What Are Templates? - Compile-Time Code Generation**

Templates are **blueprints** that let the compiler generate type-specific code at compile-time. Think of them as **type-safe macros** or **parameterized code**.

```cpp
// Function template - generates code for ANY type T
template<typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// The compiler generates these specializations:
// int max(int a, int b) { ... }
// double max(double a, double b) { ... }
// std::string max(std::string a, std::string b) { ... }

// Usage - compiler deduces T automatically:
int m1 = max(5, 10);           // T = int
double m2 = max(3.14, 2.71);   // T = double
```

**Key Insight:** Templates are **not functions/classes** themselves. They are recipes the compiler uses to **create** functions/classes. No code exists until you **instantiate** the template.

**Question Answered:** *If templates didn't exist?*  
You'd write `max_int`, `max_double`, `max_string` manually, leading to code duplication and maintenance nightmares. Templates enable **DRY** (Don't Repeat Yourself) for types.

---

### **1.2 Template Terminology - Parameters, Arguments & Specializations**

#### **Template Parameter** - Placeholder in Declaration
```cpp
template<typename T, int N>  // T and N are template parameters
class Array {
    T data[N];  // Use parameters in template definition
};
```

#### **Template Argument** - Actual Type/Value Used
```cpp
Array<int, 100> myArray;  // int and 100 are template arguments
// This creates a specialization: Array<int, 100>
```

#### **Specialization** - Tailored Implementation
A **concrete version** of a template for specific arguments.

- **Implicit specialization**: Compiler-generated (from max<int>(a,b))
- **Explicit specialization**: You write it manually
- **Partial specialization**: Only for classes, pattern-based

#### **Instantiation** - The Creation Process
When the compiler generates code from a template + arguments.

```cpp
// Explicit instantiation:
template class Array<double, 50>;  // Force compiler to generate

// Implicit instantiation:
Array<float, 10> arr;  // Compiler generates this when it sees usage
```

#### **SFINAE** - Substitution Failure Is Not An Error
If substituting template arguments creates invalid code, that template is **silently discarded** from overload resolution rather than causing a hard error.

```cpp
template<typename T>
auto foo(T t) -> typename T::type { return t; }  // Only works if T::type exists

template<typename T>
auto foo(T t) -> decltype(t + 0) { return t + 0; }  // Fallback

foo(5);  // First template fails (int has no ::type), second succeeds
```

#### **Type Traits** - Compile-Time Type Queries
Metafunctions that operate on types, producing new types or compile-time boolean values.

```cpp
#include <type_traits>

std::is_same<int, int>::value          // true
std::is_same<int, double>::value       // false
std::remove_reference<int&>::type      // int
std::add_pointer<int>::type            // int*
```

---

## **Chapter 2: Template Parameter Lists**

### **2.1 `class` vs `typename` - The Historical Distinction**

```cpp
template<class T> struct S1 {};      // Old C++98 style
template<typename T> struct S2 {};   // Modern style

// Both are IDENTICAL in function
```

**Historical Context:**
- `class` was used in original C++ because it was an existing keyword
- `typename` was introduced later because `class` was confusing (not always a class type)
- **Modern C++**: **Prefer `typename`** for clarity

**Where you MUST use `typename`:**
```cpp
template<class T>
void foo() {
    typename T::value_type x;  // REQUIRED: tells compiler value_type is a type
    // Without typename, compiler thinks it's a static member
}
```

**Rule of Thumb:** Use `typename` everywhere except when declaring a template template parameter.

```cpp
template<template<class> class Container>  // Can't use typename here
struct MetaContainer {
    Container<int> ints;
};
```

---

## **Chapter 3: Template Specialization**

### **3.1 Full/Explicit Specialization - Complete Overrides**

A complete reimplementation for **specific** template arguments.

```cpp
// Primary template
template<typename T>
struct Printer {
    static void print(const T& v) {
        std::cout << "generic: " << v << "\n";
    }
};

// Full specialization for int
template<>
struct Printer<int> {
    static void print(const int& v) {
        std::cout << "int-special: " << v << " (0x" << std::hex << v << ")\n";
    }
};

// Usage
Printer<double>::print(3.14);  // "generic: 3.14"
Printer<int>::print(42);       // "int-special: 42 (0x2a)"
```

**Key Points:**
- Template<> with **empty parameter list** indicates full specialization
- **Must appear after** primary template declaration
- Compiler **uses this** instead of generating from primary

---

### **3.2 Partial Specialization - Pattern Matching for Classes**

**Only works for class templates!** Functions don't support partial specialization.

```cpp
// Primary template
template<typename T1, typename T2>
struct PairPrinter {
    static void print(const T1& a, const T2& b) {
        std::cout << "Generic pair: (" << a << ", " << b << ")\n";
    }
};

// Partial: When first type is int
template<typename T2>
struct PairPrinter<int, T2> {
    static void print(const int& a, const T2& b) {
        std::cout << "Int-first pair: [" << a << " | " << b << "]\n";
    }
};

// Partial: When both types are the same
template<typename T>
struct PairPrinter<T, T> {
    static void print(const T& a, const T& b) {
        std::cout << "Same-type pair: {" << a << ", " << b << "}\n";
    }
};

// Usage
PairPrinter<double, std::string>::print(3.14, "hi");  // Generic
PairPrinter<int, bool>::print(5, true);               // Int-first
PairPrinter<float, float>::print(1.1f, 2.2f);          // Same-type
```

**More Examples of Partial Specialization:**

```cpp
// Primary template
template<typename T>
struct Printer {
    static void print(const T& value) {
        std::cout << "Generic type: " << value << "\n";
    }
};

// Partial: For pointer types
template<typename T>
struct Printer<T*> {
    static void print(const T* value) {
        if (value)
            std::cout << "Pointer to " << typeid(T).name()
                      << " with value: " << *value << "\n";
        else
            std::cout << "Null pointer\n";
    }
};

// Partial: For const-qualified types
template<typename T>
struct Printer<const T> {
    static void print(const T& value) {
        std::cout << "Const value: " << value << "\n";
    }
};

// Partial: For arrays
template<typename T, size_t N>
struct Printer<T[N]> {
    static void print(T (&arr)[N]) {
        std::cout << "Array of " << N << " elements: ";
        for(size_t i = 0; i < N; ++i) std::cout << arr[i] << " ";
        std::cout << "\n";
    }
};

// Usage
int x = 5;
Printer<int*>::print(&x);        // "Pointer to int with value: 5"
Printer<const int>::print(10);   // "Const value: 10"
int arr[3] = {1,2,3};
Printer<int[3]>::print(arr);     // "Array of 3 elements: 1 2 3"
```

---

## **Chapter 4: Template Instantiation & Deduction**

### **4.1 Instantiation - From Template to Concrete Code**

**Implicit Instantiation:**
```cpp
template<typename T>
void foo(T t) { /* ... */ }

foo(5);  // Compiler implicitly instantiates foo<int>(int)
```

**Explicit Instantiation:**
```cpp
// In header: template<typename T> void foo(T);

// In .cpp file:
template void foo<int>(int);      // Force instantiation
template void foo<double>(double); // Force instantiation

// Now code is generated even if not used in this file
```

**Explicit Instantiation Declaration (C++11):**
```cpp
// In header:
extern template void foo<int>(int);  // Don't instantiate here, exists elsewhere
```

**Question Answered:** *When to use explicit instantiation?*  
- To **reduce compilation time** (template code in .cpp, not header)
- To **control code bloat** (only instantiate what you need)
- To **hide implementation** (template definitions can be in .cpp)

---

### **4.2 Argument Deduction - How the Compiler Infers Types**

```cpp
template<typename T>
void process(T value);

process(42);      // T deduced as int
process(3.14);    // T deduced as double
```

**Deduction Rules:**

| Context | Deduction Rule | Example |
|---------|----------------|---------|
| **Value parameter** | `T` becomes type of argument, decay happens | `void f(T)` with `int[3]` → `T = int*` |
| **Reference parameter** | `T` is exact type, no decay | `void f(T&)` with `int[3]` → `T = int[3]` |
| **Universal reference** | `T` deduced with reference collapsing | `void f(T&&)` with `int&` → `T = int&` |
| **Const qualification** | `const` is part of `T` for values, ignored for references | `void f(const T)` with `int` → `T = int` |

**Complex Deduction Example:**
```cpp
template<typename T, size_t N>
void printArray(T (&arr)[N]) {  // Deduces both T and N
    std::cout << "Array of " << N << " ints\n";
}

int arr[5];
printArray(arr);  // T = int, N = 5
```

---

### **4.3 Return Type Deduction - `auto` & `decltype(auto)`**

**C++14 `auto` Return Type:**
```cpp
template<typename T, typename U>
auto add(T t, U u) { return t + u; }  // Return type deduced from expression

auto result = add(5, 3.14);  // result is double
```

**C++11 Trailing Return Type:**
```cpp
template<typename T, typename U>
auto add(T t, U u) -> decltype(t + u) { return t + u; }
```

**C++14 `decltype(auto)` - Preserves References & CV-qualifiers:**
```cpp
int x = 5;
int& getX() { return x; }

auto a = getX();           // a is int (value, reference dropped)
decltype(auto) b = getX(); // b is int& (reference preserved!)

template<typename T>
decltype(auto) forward(T&& t) { return std::forward<T>(t); }
```

---

## **Chapter 5: Reference Collapsing & Perfect Forwarding**

### **5.1 Reference Declaration - The Rules**

In template type deduction, references can produce **reference-to-reference** scenarios that collapse:

### **5.2 Reference Collapsing - The Four Combinations**

The **reference collapsing rules** (from C++11):
```
T& &   → T&   (lvalue reference to lvalue reference → lvalue reference)
T& &&  → T&   (lvalue reference to rvalue reference → lvalue reference)
T&& &  → T&   (rvalue reference to lvalue reference → lvalue reference)
T&& && → T&&  (rvalue reference to rvalue reference → rvalue reference)
```

**Visual Decision Table:**
```
|---------------------|----------|------------|
|                     | T = U    | T = U&     |
|---------------------|----------|------------|
| T&& (universal ref) | U&&      | U&         |
| T (by value)        | U        | U          |
| T& (lvalue ref)     | U&       | U&         |
|---------------------|----------|------------|
```

**Example:**
```cpp
template<typename T>
void foo(T&& arg);  // Universal reference

int x = 5;
int& rx = x;

foo(x);    // T = int&, T&& = int& (collapses)
foo(42);   // T = int,  T&& = int&&
foo(rx);   // T = int&, T&& = int&
```

---

### **5.3 `std::forward` - Preserving Value Categories**

**The Problem:**
```cpp
template<typename T>
void wrapper(T&& arg) {
    func(arg);  // arg is always lvalue (has a name)!
                // Loses original value category
}
```

**The Solution:**
```cpp
template<typename T>
void wrapper(T&& arg) {
    func(std::forward<T>(arg));  // Preserves original value category
}
```

**How `std::forward` works:**
```cpp
template<typename T>
T&& forward(std::remove_reference_t<T>& arg) noexcept {
    return static_cast<T&&>(arg);
}

// If T = int&& (original was rvalue):
//   returns static_cast<int&&>(arg) → int&&
// If T = int& (original was lvalue):
//   returns static_cast<int&>(arg) → int&
```

**Question Answered:** *Why not just `static_cast<T&&>`?*  
Because reference collapsing would make it always an rvalue. `forward` uses `T` (which encodes original type) to control the cast.

**Complete Example:**
```cpp
void process(int& x) { std::cout << "lvalue\n"; }
void process(int&& x) { std::cout << "rvalue\n"; }

template<typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg));
}

int a = 5;
wrapper(a);        // "lvalue" - T = int&
wrapper(42);       // "rvalue" - T = int
wrapper(std::move(a)); // "rvalue" - T = int
```

---

## **Chapter 6: Type Traits & Metaprogramming**

### **6.1 What Are Type Traits? - Type Queries & Transformations**

Type traits are **compile-time functions** that operate on types. They live in `<type_traits>`.

**Two Categories:**

1. **Queries** - Ask questions about types (return `bool`):
```cpp
std::is_integral<int>::value          // true
std::is_same<int, int>::value         // true
std::is_reference<int&>::value        // true
```

2. **Transformations** - Produce new types:
```cpp
std::remove_reference<int&>::type     // int
std::add_pointer<int>::type           // int*
std::make_const<int>::type            // const int
std::remove_const<const int>::type    // int
```

**C++14 Helper Variables:**
```cpp
std::is_integral_v<int>               // true (no ::value needed)
std::is_same_v<int, int>              // true
```

---

### **6.2 Standard Type Traits Categories**

| Category | Examples | Purpose |
|----------|----------|---------|
| **Primary** | `is_void`, `is_null_pointer` | Identify fundamental types |
| **Composite** | `is_integral`, `is_floating_point` | Group type categories |
| **Properties** | `is_const`, `is_volatile` | Type qualifiers |
| **Relations** | `is_same`, `is_base_of` | Compare types |
| **Modifications** | `remove_cv`, `add_pointer` | Create new types |

### **6.3 `std::remove_reference` & `std::forward` - Behind the Scenes**

**`std::remove_reference` implementation:**
```cpp
template<typename T>
struct remove_reference {
    using type = T;
};

template<typename T>
struct remove_reference<T&> {  // Partial specialization for references
    using type = T;
};

template<typename T>
struct remove_reference<T&&> {  // Partial specialization for rvalue refs
    using type = T;
};

// Usage:
remove_reference<int&>::type   // int
remove_reference<int&&>::type  // int
remove_reference<int>::type    // int
```

**How it's used in `forward`:**
```cpp
template<typename T>
T&& forward(typename remove_reference<T>::type& arg) noexcept {
    return static_cast<T&&>(arg);
}

// Why remove_reference? To strip reference from T before creating reference parameter
// Ensures we can bind any T to the parameter
```

**Question Answered:** *Why is remove_reference necessary in forward?*  
To prevent reference-to-reference scenarios in the function parameter. It ensures the parameter is always a **reference to the base type**, which can bind to any value category.

---

## **Chapter 7: SFINAE - Substitution Failure Is Not An Error**

### **7.1 SFINAE Basics - Overload Resolution Escape Hatch**

If substituting template arguments makes invalid code, **that overload is discarded** rather than causing a compile error.

```cpp
template<typename T>
auto foo(T t) -> typename T::type {  // Only works if T::type exists
    return typename T::type{};
}

template<typename T>
auto foo(T t) -> decltype(t + 0) {   // Fallback: works for numeric types
    return t + 0;
}

struct HasType { using type = int; };
struct NoType {};

foo(HasType{});   // First overload works
foo(42);          // First fails (int has no ::type), second works
foo(NoType{});    // First fails, second works (NoType + 0 valid?)
```

**The "Hard Error" Boundary:**
SFINAE only works in **immediate context** of template declaration. If error occurs in **function body**, it's a hard error.

```cpp
template<typename T>
void bad(T t) {
    typename T::type x;  // Hard error here if T::type doesn't exist
}
```

---

### **7.2 SFINAE in Practice - `std::enable_if`**

**C++11/14 Way:**
```cpp
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
abs(T val) {
    return val < 0 ? -val : val;
}

// Only exists when T is integral type
// std::enable_if<condition, type>::type:
//   If condition true → type is the second parameter
//   If condition false → no ::type member (SFINAE trigger)
```

**C++14 Helper:**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>  // _t suffix gives type directly
abs(T val) { /* ... */ }
```

**Multiple Overloads:**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, std::string> type_name() {
    return "integer";
}

template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, std::string> type_name() {
    return "float";
}

type_name<int>();      // Returns "integer"
type_name<double>();   // Returns "float"
// type_name<std::string>(); // ERROR: no matching overload
```

---

### **7.3 C++20 Concepts - SFINAE Done Right**

Concepts provide **cleaner syntax** for SFINAE constraints.

```cpp
// C++20 Concept
template<typename T>
concept Integral = std::is_integral_v<T>;

template<Integral T>  // Clean constraint
T abs(T val) {
    return val < 0 ? -val : val;
}

// Or directly on function
auto abs(Integral auto val) {  // Abbreviated template syntax
    return val < 0 ? -val : val;
}
```

**Question Answered:** *Why SFINAE instead of concepts?*  
SFINAE works in C++98/11/14. Concepts are C++20 only. SFINAE is more flexible for complex constraints but produces **horrible error messages**.

---

## **Chapter 8: Name Mangling - The Linker's Language**

### **8.1 What is Name Mangling?**

**Name mangling** is the process compilers use to encode type information into function/object names so the linker can distinguish overloaded functions and templates.

**Why needed?**
- Linkers work with **symbol names** (text)
- `void foo(int)` and `void foo(double)` need different symbols
- Templates generate many functions with the same name but different types

---

### **8.2 Template Name Mangling Examples**

**Itanium ABI (GCC, Clang):**
```
_Z3fooIiEvv  → void foo<int>()
_Z3fooIdEvv  → void foo<double>()
_Z3fooIPKcEvv → void foo<char const*>()

_ZN1S3barIiEEvv → void S::bar<int>()

_Z3maxIiET_S0_S0_ → int max<int>(int, int)
_Z3maxIdET_S0_S0_ → double max<double>(double, double)
```

**MSVC:**
```
??$foo@H@@YAXXZ           → void foo<int>()
??$foo@N@@YAXXZ           → void foo<double>()
```

**Structure:**
```
_Z [function name] [template args] [parameter types]
_ZN [class name] [function name] [template args] [parameters]
```

**Question Answered:** *Why does mangling matter?*  
- **Debugging**: You see mangled names in linker errors
- **ABI**: Different compilers mangle differently (can't link MSVC with GCC)
- **Symbol collisions**: Mangling prevents template instantiations from conflicting

---

### **8.3 Reading Mangled Names - `c++filt`**

```bash
# Demangle with c++filt
echo "_Z3maxIiET_S0_S0_" | c++filt
# Output: int max<int>(int, int)

echo "_ZNSt6vectorIiSaIiEEixEm" | c++filt
# Output: std::vector<int, std::allocator<int> >::operator[](unsigned long)

# Demangle entire file
c++filt < mangled_symbols.txt > demangled.txt

# GDB automatically demangles
(gdb) p _Z3fooIiEvv  # Shows as foo<int>()
```

**Patterns to Recognize:**
- `_Z` = Start of mangled name
- `N...E` = Nested name (class::member)
- `Ii` = template argument `int`
- `Id` = template argument `double`
- `IPKc` = `char const*` (I = pointer, P = pointer, K = const, c = char)

**Question Answered:** *Can I write portable code that depends on mangling?*  
**NEVER**. Mangling is compiler-specific and not standardized. Use `extern "C"` for stable C linkage.

---

## **Chapter 9: Best Practices & Common Pitfalls**

### **9.1 Template Best Practices**

1. **Define templates in headers**
   ```cpp
   // foo.h
   template<typename T>
   void foo(T t) { /* impl */ }  // Must be visible to instantiate
   
   // Exception: Explicit instantiation in .cpp
   ```

2. **Use `typename` for dependent types**
   ```cpp
   template<typename T>
   void bar() {
       typename T::type x;  // REQUIRED
   }
   ```

3. **Prefer `enable_if_t` over `enable_if::type`**
   ```cpp
   // C++14 shorthand is cleaner
   std::enable_if_t<condition, T>
   ```

4. **Use forwarding references + `std::forward`**
   ```cpp
   template<typename T>
   void wrapper(T&& arg) {
       func(std::forward<T>(arg));  // Perfect forwarding
   }
   ```

5. **C++20: Use concepts when possible**
   ```cpp
   template<typename T>
   requires Integral<T>  // Clear constraints
   void foo(T val);
   ```

### **9.2 Common Template Errors**

1. **Forgot `typename`**
   ```cpp
   template<typename T>
   void f() { T::type x; }  // ERROR: missing typename
   ```

2. **Template argument deduction failure**
   ```cpp
   template<typename T>
   void foo(T t1, T t2);
   
   foo(5, 3.14);  // ERROR: T can't be both int and double
   ```

3. **SFINAE hard error**
   ```cpp
   template<typename T>
   void bad(T t) {
       t + t;  // Hard error if T doesn't support +
   }
   ```

4. **Recursive template instantiation**
   ```cpp
   template<typename T>
   struct Node {
       T value;
       Node<Node<T>> next;  // Oops! Infinite recursion
   };
   ```

5. **Mangling confusion in linker errors**
   ```
   undefined reference to `_Z3fooIiEvv'
   # Means: forgot to define foo<int>()
   ```

---

## **Complete Template Example: A Generic Container**

```cpp
#include <type_traits>
#include <memory>

template<typename T>
class Buffer {
    static_assert(std::is_trivially_copyable_v<T>, "T must be trivially copyable");
    
    T* data;
    size_t size;
    
public:
    template<typename... Args>
    explicit Buffer(size_t s, Args&&... args)
        : size(s), data(static_cast<T*>(::operator new(sizeof(T) * s))) {
        // Placement new with perfect forwarding
        for(size_t i = 0; i < s; ++i)
            ::new (&data[i]) T(std::forward<Args>(args)...);
    }
    
    ~Buffer() {
        for(size_t i = 0; i < size; ++i) data[i].~T();
        ::operator delete(data);
    }
    
    // Copy/move with perfect forwarding
    Buffer(const Buffer& other) : Buffer(other.size) {
        static_assert(std::is_copy_constructible_v<T>, "T must be copyable");
        std::uninitialized_copy(other.data, other.data + other.size, data);
    }
    
    Buffer& operator=(Buffer&& other) noexcept {
        static_assert(std::is_nothrow_move_assignable_v<T>, "T must be noexcept moveable");
        if(this != &other) {
            swap(data, other.data);
            swap(size, other.size);
        }
        return *this;
    }
};
```

---

## **Appendix: Quick Reference**

| Feature | Syntax | When to Use |
|---------|--------|-------------|
| **Function template** | `template<typename T> void f(T);` | Generic functions |
| **Class template** | `template<typename T> class C;` | Generic classes |
| **Full specialization** | `template<> void f<int>(int);` | Custom behavior for specific type |
| **Partial specialization** | `template<typename T> class C<T*>;` | Pattern matching (classes only) |
| **SFINAE** | `std::enable_if_t<cond, T>` | Conditional overloads (C++11/14) |
| **Concepts** | `template<Concept T>` | Modern conditional overloads (C++20) |
| **Type traits** | `std::is_same_v<T, U>` | Compile-time type queries |
| **Reference collapsing** | `T&& + T&& → T&&` | Universal reference mechanics |
| **Perfect forwarding** | `std::forward<T>(arg)` | Preserve value categories |
