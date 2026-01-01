----------------------------------------------------------------------------------------------------
Mills C++ Style Guide - Draft v1.0
----------------------------------------

----------------------------------------------------------------------------------------------------
CHAPTER 1: C++ Version
----------------------------------------
CHAPTER 1: C++ Version
Definition The specific revision of the ISO C++ Standard used for the project.

Pros

Predictability: Ensures all developers, CI/CD pipelines, and production environments use a consistent feature set.

Modernity: Allows the use of features like std::expected (C++23) or concepts (C++20) which significantly improve the safety of exception-based code.

Cons

Toolchain Requirements: Newer standards may require upgrading compilers (GCC/Clang/MSVC) and build tools (CMake/Bazel), which can be a hurdle for legacy environments.

Decision The project will use C++20 as the minimum baseline. Features from C++23 are encouraged where compiler support is verified across the team’s toolchains.

Rationale C++20 introduces Concepts and Coroutines, which are transformative for writing safe, expressive code. Specifically for our exception policy, C++20/23 features allow for more expressive template constraints, ensuring that generic code only accepts types with noexcept move constructors when performance is critical.


----------------------------------------------------------------------------------------------------
CHAPTER 2:
----------------------------------------
2.1. Self-contained Headers
Definition A header file must have everything it needs to compile on its own. If you include it, you shouldn't have to include anything else first.

Pros

Reduced Fragility: Prevents "Order-of-Include" bugs where a file compiles in one place but fails in another.

Ease of Use: Simplifies the developer experience; include "logger.h" just works.

Cons

Potential for Bloat: May lead to including more than is strictly necessary if not managed carefully.

Decision All header files must be self-contained. They must use #define guards and include all necessary dependencies for their own declarations.

Rationale In a complex system, headers often act as the "API Contract." If a header is not self-contained, it implies a hidden dependency. For engineers working near the assembly/C level, this is critical: transparency in what is being pulled into the translation unit is paramount for managing binary size and compile times.

2.2. The #define Guard
Definition A preprocessor mechanism to prevent a header from being included multiple times in a single translation unit.

Decision All headers will use #ifndef guards. While #pragma once is widely supported, the standard #ifndef guard remains the most portable and robust method across all toolchains.

Format: <PROJECT>_<PATH>_<FILE>_H_

Example

C++

#ifndef MILLS_CORE_NETWORK_SOCKET_H_
#define MILLS_CORE_NETWORK_SOCKET_H_

// Header content...

#endif  // MILLS_CORE_NETWORK_SOCKET_H_
2.3. Forward Declarations
Definition Declaring the existence of a type without providing its full definition.

Pros

Compile Speed: Reduces the "Include Web," potentially shaving minutes off large builds.

Lower Coupling: Limits the number of files that need to be recompiled when a header changes.

Cons

Exception Safety Obstacle: You cannot use a forward-declared type as a data member if it's managed by a std::unique_ptr unless the destructor is defined in the .cc file.

Hidden Types: Makes it harder for tools (and humans) to find the actual definition.

Decision Avoid forward declarations where possible; use #include instead. Forward declarations should be reserved for breaking circular dependencies or extremely heavy headers that are known to bottleneck build times.

Rationale Google’s guide is very pro-forward-declaration to save compile time. However, in our guide, correctness and RAII safety take precedence. Forward declarations often interact poorly with smart pointers and templates. For a mid-tier engineer, the slight hit to compile time is a fair trade for the clarity of having the full type definition available.

2.4. Names and Order of Includes
Definition The standard sequence for including headers in a .cc file.

Decision Use the following order to maximize the chance of catching "non-self-contained" header bugs:

Related header (e.g., filesystem.cc includes filesystem.h).

C system headers (e.g., <unistd.h>, <fcntl.h>).

C++ standard library headers (e.g., <vector>, <exception>).

Other libraries' .h files (e.g., absl/strings/str_cat.h).

Your project's .h files.

Rationale By including the "Related Header" first, you ensure it is self-contained. If it relies on a hidden dependency from <vector>, the compiler will complain immediately because <vector> hasn't been included yet.

Engineering Note on Chapter 2
Engineering Note: When dealing with exception-throwing code, be wary of "Include What You Use" (IWYU) tools that might suggest removing a header that is required for an exception type. If you catch std::runtime_error, you must include <stdexcept>, even if you don't use any other functions from that header.



----------------------------------------------------------------------------------------------------
CHAPTER :
----------------------------------------
----------------------------------------------------------------------------------------------------
CHAPTER :
----------------------------------------
----------------------------------------------------------------------------------------------------
CHAPTER :
----------------------------------------
----------------------------------------------------------------------------------------------------
CHAPTER 4: Classes
----------------------------------------
4.1. Doing Work in Constructors
Definition Constructors should perform the initialization required to establish the class invariant. We allow constructors to perform operations that might fail (e.g., file I/O, memory allocation).

Pros

Invariant Integrity: Ensures an object is fully functional the moment it is created.

Syntax Clarity: Allows for direct initialization (Stack s;) instead of manual two-stage setup (Stack s; s.Init();).

Cons

Error Propagation: Failure requires throwing an exception, which must be handled or propagated.

Cleanup Complexity: If a constructor throws, the object's own destructor is not called. Only sub-objects already constructed will be cleaned up.

Decision Constructors will perform all work necessary to reach a valid state. If a constructor cannot satisfy its contract, it must throw an exception. We explicitly forbid "zombie objects" that require a secondary IsValid() check.

Rationale "Two-Stage Initialization" is an anti-pattern that leads to "Temporal Coupling"—where a user must remember to call Init() before DoWork(). We use the compiler to enforce that an invalid object can never technically exist.

Engineering Note: If a constructor fails frequently due to "expected" issues (e.g., a file not existing), consider a Static Factory Method that returns std::optional or absl::StatusOr. Reserve throwing constructors for cases where failure represents a violation of the expected environment or contract.

4.2. Destructors and noexcept
Definition The destructor is responsible for the deterministic release of resources.

Decision Destructors must never throw an exception. They must be implicitly or explicitly marked noexcept.

Pros * Ensures safe stack unwinding and prevents immediate program termination via std::terminate.

Cons * Failures during cleanup must be handled internally (e.g., logged and suppressed) and cannot be reported to the caller.

Rationale C++ cannot handle a second exception being thrown while it is already processing one during stack unwinding. Destructors are the "cleanup crew"; they must finish their job without causing further panic.

4.3. Move Constructors and Assignment
Definition Move semantics allow resources owned by an rvalue to be "moved" into a new object rather than copied.

Pros

Performance: Enables "cheap" transfers of heavy objects (like std::vector) without heap allocations.

Move-Only Types: Essential for types like std::unique_ptr.

Cons

Implementation Burden: Requires careful manual implementation (or = default) to ensure a valid empty state for the source.

The "Copy-Fallback" Tax: If not marked noexcept, standard containers will perform a slow copy instead of a move to maintain safety.

Decision Move constructors and move assignment operators will be marked noexcept unless extraordinary architectural conditions prevent it.

Rationale A move is a transfer of existing ownership. Marking these noexcept is a hint to the STL that it is safe to optimize.

Engineering Note: If a move operation throws, it is often a sign of poor design or a fundamental architectural fault. A move should be a simple pointer swap. If your move requires an allocation that could fail, investigate the design immediately.

----------------------------------------------------------------------------------------------------
CHAPTER 6: Google-Specific Magic (Modified)
----------------------------------------
6.1. Ownership and RAII
Definition Resource Acquisition Is Initialization (RAII).

Decision Every resource (memory, file handles, mutexes) must be wrapped in a manager object. Manual new and delete are forbidden.

Rationale In a codebase with exceptions, RAII is the only way to prevent leaks. Stack unwinding only works if destructors handle the cleanup.

----------------------------------------------------------------------------------------------------
CHAPTER 7: Other C++ Features
----------------------------------------

7.1. Exceptions
Definition A mechanism for signaling errors that allows the program to transfer control to a handler elsewhere in the call stack.

Decision Exceptions are permitted and encouraged for truly exceptional circumstances where a function cannot fulfill its contract.

1. When to Throw

Contract Violations: Precondition/postcondition failures.

Constructor Failures: Inability to establish an invariant.

Operator Overloads: Errors where a return value cannot indicate failure (e.g., operator[]).

2. When to use Return Codes

Expected Failures: Use std::optional or Status for normal flow (e.g., UserNotFound).

High-Performance Loops: Avoid the overhead of stack unwinding in tight loops.

3. Safety Levels

All code must provide the Basic Exception Guarantee (no leaks).

Critical components should strive for the Strong Exception Guarantee (transactional).

7.2. Run-Time Type Information (RTTI)
Decision RTTI is generally restricted. However, dynamic_cast and typeid are permitted specifically for exception handling hierarchies to allow for granular error recovery.

7.3. 0 and nullptr/NULL
Decision * Use nullptr for all pointer values.

Use '\0' for the null character literal.

Use 0 only for numerical integer values.

Rationale Prevents ambiguity in function overloading (e.g., f(int) vs f(void*)).

7.4. Standard Library (STL)
Decision The STL is fully permitted, including features that throw.

Container Access: .at() is preferred when bounds checking is required.

Streams: Permitted, though std::format or absl::PrintF are preferred for performance.

Engineering Note: The STL is not a license for sloppiness. If an STL feature (like <regex>) is known for performance cliffs, it must be flagged for architectural review.
