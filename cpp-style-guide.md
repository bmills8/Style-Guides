Mills C++ Style Guide - Draft v1.0
CHAPTER 1: C++ Version
(Pending: Specification of C++17/20/23 baseline)

CHAPTER 4: Classes
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

CHAPTER 6: Google-Specific Magic (Modified)
6.1. Ownership and RAII
Definition Resource Acquisition Is Initialization (RAII).

Decision Every resource (memory, file handles, mutexes) must be wrapped in a manager object. Manual new and delete are forbidden.

Rationale In a codebase with exceptions, RAII is the only way to prevent leaks. Stack unwinding only works if destructors handle the cleanup.

CHAPTER 7: Other C++ Features
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
