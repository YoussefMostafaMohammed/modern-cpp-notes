# modern-cpp-notes

This repository contains personal study notes and practical examples for **Modern C++** (C++11–C++23).  
It covers Object-Oriented Programming, Smart Pointers, Templates, and Value Categories (lvalues, rvalues, universal references).  

Think of this as a mini-book for modern C++ concepts, with a clickable Table of Contents for easy navigation.

---

## 📚 Complete Table of Contents

### **1. Object-Oriented Programming (OOP)**
- [0.1 What is OOP? - The Four Pillars](1%20-%20OOP/OOP.md#01-what-is-oop--the-four-pillars)
- [0.2 Classes vs Objects - Blueprint vs Instance](1%20-%20OOP/OOP.md#02-classes-vs-objects--the-blueprint-vs-the-instance)
- [0.3 What is Inheritance? - "Is-A" Relationship](1%20-%20OOP/OOP.md#03-what-is-inheritance--the-is-a-relationship)
- [0.4 How Class Size is Measured](1%20-%20OOP/OOP.md#04-how-class-size-is-measured)
- [1.1 Access Control (`public`, `protected`, `private`)](1%20-%20OOP/OOP.md#11-public-protected-private--the-encapsulation-trinity)
- [1.2 `struct` vs `class`](1%20-%20OOP/OOP.md#12-struct-vs-class--the-default-difference)
- [1.3 `friend` - Encapsulation Breaker](1%20-%20OOP/OOP.md#13-friend--the-encapsulation-breaker)
- [1.4 `= default` - Compiler-Generated Functions](1%20-%20OOP/OOP.md#14-default--explicit-compiler-generated-functions)
- [1.5 `static` Keyword](1%20-%20OOP/OOP.md#15-the-static-keyword---class-level-vs-instance-level)
- [2.1 Inheritance Types (`public`, `protected`, `private`)](1%20-%20OOP/OOP.md#21-inheritance-types-public-protected-private)
- [2.2 Basic Virtual Functions](1%20-%20OOP/OOP.md#22-basic-virtual-functions--the-polymorphism-gateway)
- [2.3 Pure Virtual Functions & Abstract Classes](1%20-%20OOP/OOP.md#23-pure-virtual-functions--abstract-classes)
- [2.4 Virtual Destructors](1%20-%20OOP/OOP.md#24-virtual-destructors--the-memory-leak-preventer)
- [3.1 Diamond Problem Without `virtual`](1%20-%20OOP/OOP.md#31-the-diamond-problem-without-virtual)
- [3.2 Virtual Inheritance - Solution](1%20-%20OOP/OOP.md#32-virtual-inheritance--the-solution)
- [3.3 Memory Layout Cost Analysis](1%20-%20OOP/OOP.md#33-memory-layout-cost-analysis)
- [4.1 What Are Vtables & Vptrs](1%20-%20OOP/OOP.md#41-what-are-vtables-and-vptrs)
- [4.2 Single Inheritance Vtable Layout](1%20-%20OOP/OOP.md#42-single-inheritance-vtable-layout)
- [4.3 Multiple & Virtual Inheritance Vtables](1%20-%20OOP/OOP.md#43-multiple--virtual-inheritance---primary--secondary-vtables)
- [4.4 Thunks - `this` Adjustment](1%20-%20OOP/OOP.md#44-thunks--the-this-pointer-adjustment)
- [5.1 The VTT (Virtual Table Table)](1%20-%20OOP/OOP.md#51-the-vtt-virtual-table-table)
- [5.2 Construction Vtables](1%20-%20OOP/OOP.md#52-construction-vtables--safety-during-build)
- [5.3 Destruction Sequence & VTT Entries](1%20-%20OOP/OOP.md#53-destruction-sequence--vtt-entries)
- [6.1 RTTI: `type_info` & `dynamic_cast`](1%20-%20OOP/OOP.md#61-type_info-and-dynamic_cast)
- [6.2 `top_offset` - Most Derived Object](1%20-%20OOP/OOP.md#62-top_offset--finding-the-most-derived-object)
- [6.3 `vbase_offset` - Locating Virtual Bases](1%20-%20OOP/OOP.md#63-vbase_offset--locating-virtual-bases)
- [7.1 When to Use Virtual Inheritance](1%20-%20OOP/OOP.md#71-when-to-use-virtual-inheritance)
- [7.2 Performance Considerations](1%20-%20OOP/OOP.md#72-performance-considerations)
- [7.3 Common Pitfalls](1%20-%20OOP/OOP.md#73-common-pitfalls)
- [Appendix A.1 Dumping Vtables](1%20-%20OOP/OOP.md#a1-dumping-vtables-with-gccclang)
- [Appendix A.2 Decoding Mangled Names](1%20-%20OOP/OOP.md#a2-decoding-mangled-names)

### **2. Smart Pointers**
- [`std::unique_ptr` — Unique Ownership](2%20-%20SmartPointers/SmartPointers.md#stdunique_ptr--unique-ownership)
- [`std::move()` vs `.reset()`](2%20-%20SmartPointers/SmartPointers.md#stdmove-vs-reset)
- [`std::shared_ptr` — Shared Ownership](2%20-%20SmartPointers/SmartPointers.md#stdshared_ptr--shared-ownership)
- [`std::weak_ptr` — Weak Ownership](2%20-%20SmartPointers/SmartPointers.md#stdweak_ptr--weak-ownership)
- [Comparison Table](2%20-%20SmartPointers/SmartPointers.md#comparison-table)
- [Smart Pointer Internals](2%20-%20SmartPointers/SmartPointers.md#smart-pointer-internals)
- [Interview Questions & Deep Dive](2%20-%20SmartPointers/SmartPointers.md#interview-questions--deep-dive)
- [Practical Guidelines](2%20-%20SmartPointers/SmartPointers.md#practical-guidelines)

### **3. Templates**
- [Template Argument Deduction](3%20-%20Templates/Templates.md#3-template-argument-deduction)
- [Substitution in Templates](3%20-%20Templates/Templates.md#4-substitution-in-templates)
- [Reference Collapsing Rules](3%20-%20Templates/Templates.md#5-reference-collapsing-rules)
- [Forwarding (Universal) References](3%20-%20Templates/Templates.md#6-forwarding-universal-references)
- [Perfect Forwarding](3%20-%20Templates/Templates.md#12-perfect-forwarding)
- [Common Pitfalls & Special Cases](3%20-%20Templates/Templates.md#13-common-pitfalls)
- [Quick Summary Table](3%20-%20Templates/Templates.md#19-quick-summary-table)

### **4. Lvalues, Rvalues & Universal References**
- [lvalues and rvalues](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#1-lvalues-and-rvalues)
- [Differences between `T&` and `T&&`](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#2-differences-between-t-and-t)
- [Step-by-Step Deduction Flow](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#7-step-by-step-flow-argument--deduction--substitution--collapse--final-type)
- [Why `T` is deduced as `int&` for lvalues](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#8-why-t-is-deduced-as-int-for-lvalues-not-just-int)
- [Printing `T` and `arg` Types](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#11-printing-t-and-arg-type)
- [Advanced Topics & Best Practices](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#16-advanced-topics)
- [Visual ASCII Diagram: The Full Flow](4%20-%20Lvalue-Rvalue-UniversalReference/Lvalue-Rvalue-UniversalReference.md#20-visual-ascii-diagram-the-full-flow)

---

## License

This repository is for personal study and reference.
