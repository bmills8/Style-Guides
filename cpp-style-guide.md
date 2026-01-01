----------------------------------------------------------------------------------------------------
Mills C++ Style Guide - Draft
----------------------------------------


----------------------------------------------------------------------------------------------------
CHAPTER 1 - C++ Version
----------------------------------------


----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH2) Header Files
Self-contained HeadersThe #define GuardInclude What You UseForward DeclarationsDefining Functions in Header FilesNames and Order of Includes

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH3 Scoping
NamespacesInternal LinkageNonmember, Static Member, and Global FunctionsLocal VariablesStatic and Global Variablesthread_local Variables

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH4) Classes
Doing Work in ConstructorsImplicit ConversionsCopyable and Movable TypesStructs vs. ClassesStructs vs. Pairs and TuplesInheritanceOperator OverloadingAccess ControlDeclaration Order

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH5) Functions
Inputs and OutputsWrite Short FunctionsFunction OverloadingDefault ArgumentsTrailing Return Type Syntax

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH6) Google-Specific Magic
Ownership and Smart Pointerscpplint

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH7) Other C++ Features
Rvalue ReferencesFriendsExceptionsnoexceptRun-Time Type Information (RTTI)CastingStreamsPreincrement and PredecrementUse of constUse of constexpr, constinit, and constevalInteger TypesFloating-Point TypesArchitecture PortabilityPreprocessor Macros0 and nullptr/NULLsizeofType Deduction (including auto)Class Template Argument DeductionDesignated InitializersLambda ExpressionsTemplate MetaprogrammingConcepts and ConstraintsC++20 modulesCoroutinesBoostDisallowed standard library featuresNonstandard ExtensionsAliasesSwitch Statements

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH8) Inclusive Language

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH9) Naming
Choosing NamesFile NamesType NamesConcept NamesVariable NamesConstant NamesFunction NamesNamespace NamesEnumerator NamesTemplate Parameter NamesMacro NamesAliasesExceptions to Naming Rules

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH10) Comments
Comment StyleFile CommentsStruct and Class CommentsFunction CommentsVariable CommentsImplementation CommentsPunctuation, Spelling, and GrammarTODO Comments

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH11) Formatting
Line LengthNon-ASCII CharactersSpaces vs. TabsFunction Declarations and DefinitionsLambda ExpressionsFloating-point LiteralsFunction CallsBraced Initializer List FormatLooping and branching statementsPointer and Reference Expressions and TypesBoolean ExpressionsReturn ValuesVariable and Array InitializationPreprocessor DirectivesClass FormatConstructor Initializer ListsNamespace FormattingHorizontal WhitespaceVertical Whitespace

----------------------------------------------------------------------------------------------------
CHAPTER  - 
----------------------------------------
CH12) Exceptions to the Rules
Existing Non-conformant Code Windows Code


----------------------------------------------------------------------------------------------------
...INDEX MAYBE? 
----------------------------------------










----------------------------------------------------------------------------------------------------
CHAPTER 4 
----------------------------------------
Classes (Constructors and Destructors). This is where the Google guide is most restrictive and where our change has the most impact.

2.x. Classes: Constructors and Destructors
2.x.1. Doing Work in Constructors
Definition Constructors should perform the initialization required to establish the class invariant. Unlike the base Google style, we allow constructors to perform operations that might fail (e.g., file I/O, memory allocation, or hardware initialization).

Pros

Invariant Integrity: Ensures an object is fully functional the moment it is created. No "half-baked" objects.

Syntax Clarity: Allows for direct initialization (Stack s;) instead of manual two-stage setup (Stack s; s.Init();).

Cons

Error Propagation: Failure requires throwing an exception, which must be handled by the caller or allowed to propagate.

Cleanup Complexity: If a constructor throws, the object's own destructor is not called. Only sub-objects (members) that were already fully constructed will be cleaned up.

Decision Constructors will perform all work necessary to reach a valid state. If a constructor cannot satisfy its contract, it must throw an exception. We explicitly forbid "zombie objects" that require a secondary IsValid() check.

Rationale In a language with exceptions, "Two-Stage Initialization" is an anti-pattern that leads to "Temporal Coupling"—where a user must remember to call Init() before DoWork(). By allowing constructors to throw, we use the compiler to enforce that an invalid object can never be used because it will never technically exist.

Example

C++
// GOOD: Constructor establishes the invariant or fails.
class FileLogger {
 public:
  FileLogger(const std::string& path) {
    file_handle_ = fopen(path.c_str(), "w");
    if (!file_handle_) {
      throw std::runtime_error("Could not open log file for writing.");
    }
  }
 private:
  FILE* file_handle_;
};

// BAD: Zombie object pattern.
class FileLogger {
 public:
  FileLogger(const std::string& path) : handle_(nullptr), path_(path) {}
  bool Init() { // If the user forgets to call this, the app crashes later.
    handle_ = fopen(path_.c_str(), "w");
    return handle_ != nullptr;
  }
};
0 and nullptr/NULL
Definition Guidelines for the representation of null pointers and the null character literal.

Pros

Type Safety: nullptr has its own type (std::nullptr_t), preventing it from being confused with an integer 0.

Readability: Clearly distinguishes between a numerical zero, a null character, and a null memory address.

Cons

Legacy Friction: Requires updating older C-style codebases where NULL or 0 was the standard.

Decision

Use nullptr for all pointer values.

Use '\0' for the null character literal.

Use 0 only for numerical integer values.

Rationale In C++, 0 is an integer. Before C++11, we used 0 or NULL (often a macro for 0) for pointers, which created ambiguity in function overloading. If you have void f(int) and void f(void*), calling f(0) resolves to the int version. Using nullptr ensures the void* version is called. While this seems trivial, it prevents subtle logic bugs in overloaded APIs and template metaprogramming.

2.x.2. Destructors and noexcept
Definition: The destructor is responsible for releasing resources.

The Rule: Destructors must never throw an exception. They must be implicitly or explicitly marked noexcept.

Pros: Ensures safe stack unwinding. Prevents immediate program termination via std::terminate.

Cons: Failures during cleanup (like failing to close a file descriptor) must be handled silently or logged within the destructor; they cannot be reported to the caller via the exception mechanism.

Rationale: C++ cannot handle a second exception being thrown while it is already processing one during stack unwinding. Destructors are the "cleanup crew"; they must finish their job without causing further panic.

2.x.3. Move Constructors and Assignment
Definition Move semantics allow the resources owned by an rvalue object to be "moved" into a new object rather than copied. This involves transferring pointers or handles and setting the source object to a valid but empty state.

Pros

Massive Performance Gains: Enables "cheap" transfers of heavy objects (like std::vector or std::string) without heap allocations.

Requirement for Move-Only Types: Essential for types that cannot be copied, such as std::unique_ptr or std::lock_guard.

Cons

Implementation Burden: Requires careful manual implementation (or = default) to ensure the source object is left in a valid state.

The "Copy-Fallback" Tax: If a move constructor is not marked noexcept, many standard library containers (e.g., std::vector) will ignore the move and perform a slow copy instead to maintain the Strong Exception Guarantee during reallocations.

Decision Move constructors and move assignment operators will be marked noexcept unless extraordinary architectural conditions prevent it. If a move operation is not noexcept, the engineer must document why a copy-fallback is acceptable for that specific type.

Rationale A move is fundamentally a transfer of existing ownership, not the acquisition of new resources. Therefore, it should not fail. Marking these noexcept is not just about documentation; it is a hint to the compiler and the STL that it is safe to optimize. Without this keyword, you lose the primary performance benefits of Modern C++ when using standard containers.

Example

C++
class DataBuffer {
 public:
  // GOOD: Clear use of noexcept for move semantics
  DataBuffer(DataBuffer&& other) noexcept 
      : data_(std::move(other.data_)), size_(other.size_) {
    other.size_ = 0;
  }

  // BAD: Missing noexcept. std::vector<DataBuffer> will now 
  // copy every element during a resize instead of moving.
  DataBuffer& operator=(DataBuffer&& other) {
    data_ = std::move(other.data_);
    return *this;
  }

 private:
  std::unique_ptr<char[]> data_;
  size_t size_;
};

3.x. Resource Management (RAII)
Definition: Resource Acquisition Is Initialization (RAII).

The Rule: Every resource must be wrapped in a manager object. Manual new and delete are forbidden.

Pros: Exception safety. When an exception is thrown, the stack unwinds and destructors are called automatically. If your memory is in a std::unique_ptr, it is freed. If it is a raw pointer, it is leaked.

Rationale: In a codebase that uses exceptions, RAII is the only way to prevent leaks. This moves the "cleanup" logic from the error-handling path to the object's lifecycle.












----------------------------------------------------------------------------------------------------
CHAPTER 7
----------------------------------------

3.x. Exceptions
Definition
Exceptions are a built-in C++ mechanism for signaling and handling errors that occur at runtime, allowing the program to transfer control from the point where an error occurs to a handler elsewhere in the call stack.

Pros
Cleaner Happy-Path: Separates error-handling logic from the main business logic, reducing nested if statements.

Constructor Safety: Allows constructors to signal failure, ensuring that an object cannot exist in an invalid or "zombie" state.

Generic Programming: Essential for operators and template functions (like those in the STL) where returning a Status object is syntactically or architecturally impossible.

Standard Compliance: Enables the full and safe use of the C++ Standard Library.

Cons
Implicit Control Flow: It is not always obvious which functions can throw, making it harder to reason about the program's execution path.

Binary Size: Exception handling metadata (unwind tables) can increase the size of the executable.

Performance: While the "happy path" is usually as fast as return codes, the "sad path" (throwing and catching) is significantly more expensive due to stack unwinding.

Safety Requirements: Requires rigorous use of RAII; manual memory management becomes a high-risk liability.

Decision
Exceptions are permitted and encouraged for handling truly exceptional circumstances—situations where a function cannot fulfill its contract and the caller is not expected to handle the condition as part of normal control flow.

1. When to Throw
Contract Violations: Use exceptions for precondition failures, postcondition failures, and invariant violations where the caller has provided invalid data or the system is in an unrecoverable state (e.g., out of memory, missing critical configuration).

Constructor Failures: If a constructor cannot fully initialize an object, it must throw.

Operator Overloads: Use exceptions for errors in operators where a return value cannot indicate failure (e.g., operator[], operator+).

2. When to use Return Codes (Status/StatusOr)
Expected Failures: Use return codes for conditions that are part of normal operation. For example, a FindUser() function should return a Status or std::optional if the user isn't found, rather than throwing.

High-Performance Loops: Avoid throwing in tight loops where the overhead of stack unwinding would be prohibitive.

3. Exception Safety Levels
All code must provide at least the Basic Exception Guarantee. Critical components should strive for the Strong Exception Guarantee.

Basic Guarantee: If an exception is thrown, no memory is leaked and objects remain in a valid (though perhaps indeterminate) state.

Strong Guarantee: If an exception is thrown, the state of the program is rolled back to exactly what it was before the function call (transactional).

No-throw Guarantee (noexcept): The function is guaranteed to never throw. This is mandatory for destructors, move constructors, and swap functions.

4. Syntax and Mechanics
Throw by Value, Catch by Const Reference: Always throw MyException(); and catch (const MyException& e). This prevents "slicing" and avoids unnecessary copying.

Inherit from std::exception: All custom exception classes must inherit (directly or indirectly) from std::exception or its standard derivatives (e.g., std::runtime_error).

Specify noexcept: Mark functions as noexcept whenever possible, especially for low-level utility functions. This allows the compiler to optimize and prevents std::terminate during unwinding.



7.x. Standard Library Features
Definition Guidelines for the use of the C++ Standard Template Library (STL), specifically focusing on features that were previously restricted due to exception-handling overhead or non-deterministic behavior.

Pros

Interoperability: Most third-party libraries expect standard types and behaviors (e.g., std::vector::at throwing std::out_of_range).

Code Redundancy: Reduces the need for custom "Safe" wrappers or Abseil alternatives where the STL equivalent is now safe to use with our exception policy.

Developer Familiarity: Modern C++ engineers are trained on the STL; using standard patterns reduces onboarding time.

Cons

Hidden Complexity: Some STL features (like <regex>) can have significant performance cliffs or hidden allocations.

Binary Bloat: Inclusion of heavy headers like <iostream> or <complex> can impact compile times and binary size.

Decision The Standard Library is fully permitted, including features that throw exceptions. We specifically lift restrictions on:

Container Access: Using .at() is preferred over [] when bounds checking is required and failure should trigger an exception.

Streams (<iostream>, <fstream>): Permitted for logging, file I/O, and string manipulation, though absl::PrintF or std::format (C++20) are often preferred for performance.

Algorithms: Full use of <algorithm> is encouraged to replace manual loops.

Rationale Google’s historical ban on certain STL features was largely a byproduct of their "No Exceptions" rule. Since we have embraced exceptions for contract violations, the STL becomes the most efficient and expressive toolset available. By using the standard containers and algorithms, we leverage decades of compiler optimization and community-vetted code.

Engineering Note: While the STL is permitted, it is not a license for sloppiness. Features like std::bind or std::function still carry overhead. If an STL feature is known for poor performance (like the aforementioned <regex>), it should still be flagged for architectural review in high-performance hot paths. A "Standard" library is not always an "Optimized" library for every use case.

Example

C++

// GOOD: Using STL exceptions for bounds safety
void ProcessData(const std::vector<int>& data) {
  try {
    int value = data.at(100); // Throws std::out_of_range if index is invalid
    DoWork(value);
  } catch (const std::out_of_range& e) {
    LOG_ERROR("Data vector was too small: %s", e.what());
    throw; // Re-throw if we can't recover here
  }
}

// BAD: Defensive programming that duplicates STL logic
int ProcessDataManual(const std::vector<int>& data) {
  if (data.size() <= 100) { // Redundant if we are already using an exception-safe flow
    return -1; 
  }
  return data[100];
}











