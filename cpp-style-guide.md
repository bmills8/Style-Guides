----------------------------------------------------------------------------------------------------
Table of Contents: Mills C++ Style Guide - Draft v1.0
----------------------------------------

Preamble) 

CH1) C++ Version

CH2) Header Files Self-contained Headers 
The #define Guard
Include What You Use
Forward Declarations
Defining Functions in Header Files
Names and Order of Includes

CH3) Scoping
Namespaces
Internal Linkage
Nonmember, Static Member, and Global Functions
Local Variables
Static and Global Variables
thread_local Variables

CH4) Classes
Doing Work in Constructors
Implicit Conversions
Copyable and Movable Types
Structs vs. Classes
Structs vs. Pairs and Tuples
Inheritance
Operator Overloading
Access Control
Declaration Order
CH5) Functions
Inputs and Outputs
Write Short Functions
Function Overloading
Default Arguments
Trailing Return Type Syntax

CH6) Google-Specific Magic
Ownership and Smart Pointers
cpplint

CH7) Other C++ Features
Rvalue References
Friends
Exceptions
noexcept
Run-Time Type Information (RTTI)
Casting
Streams
Preincrement and Predecrement
Use of const
Use of constexpr, constinit, and consteval
Integer Types
Floating-Point Types
Architecture Portability
Preprocessor Macros
0 and nullptr/NULL
sizeof
Type Deduction (including auto)
Class Template Argument Deduction
Designated Initializers
Lambda Expressions
Template Metaprogramming
Concepts and Constraints
C++20 Modules
Coroutines
Boost
Disallowed Standard Library Features
Nonstandard Extensions
Aliases
Switch Statements

CH8) Inclusive Language

CH9) Naming
Choosing Names
File Names
Type Names
Concept Names
Variable Names
Constant Names
Function Names
Namespace Names
Enumerator Names
Template Parameter Names
Macro Names
Aliases
Exceptions to Naming Rules
CH10) Comments
Comment Style
File Comments
Struct and Class Comments
Function Comments
Variable Comments
Implementation Comments
Punctuation, Spelling, and Grammar
TODO Comments

CH11) Formatting
Line Length
Non-ASCII Characters
Spaces vs. Tabs
Function Declarations and Definitions
Lambda Expressions
Floating-point Literals
Function Calls
Braced Initializer List Format
Looping and Branching Statements
Pointer and Reference Expressions and Types
Boolean Expressions
Return Values
Variable and Array Initialization
Preprocessor Directives
Class Format
Constructor Initializer Lists
Namespace Formatting
Horizontal Whitespace
Vertical Whitespace
CH12) Exceptions to the Rules
Existing Non-conformant Code
Windows Code


----------------------------------------------------------------------------------------------------
PREAMBLE:
----------------------------------------
This serves as a modern but vanilla C++ style guide inspired by Google's C++ Style Guide. The motivation for this guide is to allow Exceptions resulting in a cascading effect throughout Google's guide inspiring a branch of that excellent product. Operating System specific guides are a future goal appendix goal to this guide after it enters production.

----------------------------------------------------------------------------------------------------
CHAPTER 1: C++ Version
----------------------------------------
Definition The specific revision of the ISO C++ Standard required for the project and the policy for adopting newer features.

Pros

Predictability: Ensures consistent behavior across developer workstations, CI/CD pipelines, and production environments.

Safety & Expressiveness: Modern standards provide tools like Concepts and std::expected that replace error-prone preprocessor macros and complex template metaprogramming.

Performance: Features like std::span and improved Move Semantics allow for zero-cost abstractions that are difficult to implement safely in older standards.

Cons

Toolchain Dependency: Requires modern compilers (GCC 10+, Clang 12+, MSVC 19.29+). Older "Long Term Support" (LTS) server environments may require custom toolchain installations.

Decision

Baseline: The project must use C++20. All code must compile under this standard without warnings.

Optionality: Features from C++23 are permitted only when verified against the project's primary compilers.

Verification: Use C++20 Feature-test macros (e.g., __cpp_lib_expected) to guard C++23 features if the codebase must remain portable to strictly C++20 environments.

Rationale C++20 is the "New Baseline" for systems programming. Concepts allow us to enforce our exception-safety rules at compile-time (e.g., ensuring a type has a noexcept move constructor). Coroutines and Ranges allow for highly efficient asynchronous and data-processing code that remains readable. By anchoring at C++20, we gain the performance of C and Assembly with the safety of a modern managed language.

Engineering Note: Avoid "Standard Chasing." Do not adopt a C++23 feature just because it exists. Only move beyond the C++20 baseline if the feature provides a measurable improvement in safety (like std::expected for error handling) or performance.


----------------------------------------------------------------------------------------------------
CHAPTER 2: Header Files
----------------------------------------
2.1. Self-contained Headers
Definition A header file must be capable of compiling in isolation.

Decision All headers must be self-contained. If a header uses std::vector, it must include <vector>.

Prohibited: Relying on the "accidental" inclusion of a dependency by a previous header in the .cc file.

Rationale Headers are the "API Contract." Non-self-contained headers create "Temporal Coupling" where the build breaks simply because someone reordered an include list. For systems-level work, this transparency is vital for understanding the true dependency graph of a module.

2.2. The #define Guard
Decision Use standard #ifndef guards. Do not use #pragma once.

Format: <PROJECT>_<PATH>_<FILE>_H_

Example: MILLS_NETWORK_CORE_SOCKET_H_

Rationale While #pragma once is common, it is not part of the C++ standard and can fail in complex build environments involving symbolic links or multiple mount points. Standard guards are universally robust and portable.

2.3. Include What You Use (IWYU)
Definition If a .h or .cc file uses a symbol, it should explicitly include the header providing that symbol.

Decision Do not rely on transitive includes (e.g., including <iostream> just because it happens to include <iosfwd>).

Rationale Transitive dependencies are unstable. A compiler update might change what <vector> includes internally, breaking your code if you weren't explicit. This is particularly important for Exception types (e.g., explicitly include <stdexcept>).

2.4. Forward Declarations
Decision Avoid forward declarations. Use #include instead.

Exceptions: Permitted only to break circular dependencies or when an included header is verified to add >1 second to the per-file compile time.

Rationale Forward declarations hide the dependency graph from the compiler. Crucially, they interact poorly with RAII: a std::unique_ptr<T> requires the full definition of T to generate the destructor code. Hiding the definition forces you to move the destructor to the .cc file, which is a "Design Tax" that often outweighs the compile-time benefit.

2.5. Defining Functions in Header Files
Decision Only define functions in headers if:

They are Templates.

They are constexpr or consteval.

They are small, high-performance accessors (usually 1-2 lines) marked inline.

Rationale Defining complex logic in headers increases "Binary Bloat" and significantly slows down rebuilds, as any change to the logic requires recompiling every file that includes that header.

2.6. Names and Order of Includes
Decision Use the following order, with a blank line between groups:

Related Header: (e.g., socket.cc starts with #include "socket.h")

C System Headers: (e.g., <unistd.h>)

C++ Standard Library: (e.g., <vector>)

Third-party Library Headers: (e.g., absl/strings/str_cat.h)

Project Headers: (e.g., "mills/core/util.h")

Rationale Including the related header first is a "Self-Containment Test." If socket.h is missing a dependency, socket.cc will fail to compile immediately.

Engineering Note on Chapter 2
Engineering Note: In an exception-enabled system, the "Cost of Inclusion" is not just compile time—it is binary size. Each function included in a header (especially templates) requires the compiler to generate exception-unwind tables in every translation unit where it is used. Keep headers lean to keep the "Sad Path" overhead manageable.



----------------------------------------------------------------------------------------------------
CHAPTER 3: Scoping
----------------------------------------
CHAPTER 3: Scoping
3.1. Namespaces
Definition A mechanism for logical grouping and symbol isolation.

Decision

Top-Level: All project code must reside within the mills namespace.

Nesting: Limit nesting to two levels deep (e.g., mills::network).

Header Restrictions: Never use using directives (e.g., using namespace std;) in headers. In .cc files, using directives are permitted only for specific symbols or within a restricted function scope.

Inline Namespaces: Restricted to ABI versioning.

Rationale For engineers transitioning from C, namespaces are effectively "compiler-managed prefixes." They are essential for preventing collisions in large binaries, but "using" directives in headers pollute the global namespace of every file that includes them, re-introducing the "Symbol Soup" we are trying to avoid.

3.2. Internal Linkage (Anonymous Namespaces and Static)
Decision

Use Anonymous Namespaces in .cc files for types and utility functions that do not need to be visible to other translation units.

Mark variables and constants in .cc files as static or const to ensure internal linkage.

Rationale This keeps the global symbol table lean, speeds up the linker, and prevents "One Definition Rule" (ODR) violations. It also signals to the reader that the code is a "private helper" for that specific file.

3.3. Nonmember, Static Member, and Global Functions
Decision

Prefer nonmember functions within a namespace over static member functions of a class when the function doesn't need access to private class data.

Global functions (outside a namespace) are strictly forbidden.

Rationale Placing functions outside of a class reduces "Class Bloat" and header coupling. It makes the code easier to test and more resilient to changes in class internals.

3.4. Local Variables
Decision Declare variables in the narrowest scope possible and initialize them immediately.

Rationale In an exception-safe codebase, stack unwinding only cleans up objects that have successfully finished their constructors. By delaying a declaration until you have the data to initialize it, you ensure you never have "uninitialized" or "half-baked" objects sitting on the stack if an exception is thrown.

3.5. Static and Global Variables
Definition Variables with static storage duration.

Decision

Forbidden: Objects with non-trivial constructors (those that allocate memory, open files, or take locks) cannot be global or static.

Permitted: Primitive types (PODs), pointers, and constexpr variables.

The Singleton Pattern: If a global-scope object is required, use a function-local static (The Meyers Singleton).

Rationale The Static Initialization Order Fiasco means the order in which globals across different files are created is undefined. More dangerously, if a global constructor throws an exception, the program calls std::terminate() immediately because there is no try/catch block around the program's entry point.

3.6. thread_local Variables
Decision Use thread_local only for truly thread-specific data (e.g., per-thread error contexts or random number seeds).

Rationale Similar to globals, thread_local constructors can throw. If this happens during thread creation (e.g., inside std::thread), it may be uncatchable depending on the platform. Furthermore, ensure thread_local destructors are noexcept, as they execute during thread termination where exception handling is extremely limited.

Engineering Note on Chapter 3
Engineering Note: A crash during global initialization is a "Silent Killer." It often occurs before the logger is initialized, leaving you with a program that simply vanishes. If you must have global-like state, use the "Leaky Singleton" (storing a pointer to a heap-allocated object that is never deleted) to ensure you are safely inside the main() execution context when the object is created.


----------------------------------------------------------------------------------------------------
CHAPTER 4: Classes
----------------------------------------
4.1. Doing Work in Constructors
Decision Constructors must establish the class invariant. If initialization fails (e.g., memory allocation, resource acquisition), the constructor must throw an exception.

Prohibited: "Zombie objects" that require a secondary IsValid() or Init() call.

Rationale Two-stage initialization creates "Temporal Coupling"—the risk of calling methods on an uninitialized object. By throwing, we ensure that any named object in a scope is fully functional.

Engineering Note: If a constructor throws, the object’s destructor is not called. However, sub-objects (members) that were already fully constructed are destroyed. Use RAII members (like std::unique_ptr) to ensure no leaks occur during a constructor failure.

4.2. Implicit Conversions
Decision All single-argument constructors and conversion operators must be marked explicit.

Rationale This prevents the compiler from creating "hidden" temporary objects. In systems-level code, a hidden temporary can lead to unexpected performance hits or, worse, a temporary being created and destroyed during an exception unwind, masking the original error.

4.3. Copyable and Movable Types
Decision * Rule of Three/Five/Zero: If you define a destructor, you must likely define or delete the copy/move constructors and assignment operators.

Prohibited: Implicitly allowing copies of classes that own unique resources (e.g., file descriptors, sockets). Mark these = delete.

Requirement: Move constructors and move assignment operators must be marked noexcept.

Rationale Standard containers (like std::vector) will only use move operations if they are guaranteed not to throw. If they aren't noexcept, the container will perform a slow copy to maintain exception safety.

4.4. Structs vs. Classes
Decision Use a struct only for passive data (PODs). If a type has an invariant, private data, or virtual functions, use a class.

4.5. Structs vs. Pairs and Tuples
Decision Prefer a named struct over std::pair or std::tuple whenever the elements have distinct meanings.

Rationale struct.status is significantly more readable and maintainable than std::get<0>(tuple). This clarity is vital when examining state inside a catch block.

4.6. Inheritance
Decision

Virtual Destructors: Any base class with virtual functions must have a virtual destructor.

Modern Syntax: Use override on all overridden virtual functions; use final only when a class is explicitly designed to be the end of a hierarchy.

Composition: Prefer composition over inheritance unless a clear "is-a" relationship exists.

Rationale Failure to provide a virtual destructor leads to undefined behavior during destruction. In an exception-rich environment where objects are frequently destroyed during unwinding, this is a critical safety requirement.

4.7. Operator Overloading
Decision

Overload operators only when their meaning is mathematically or logically intuitive.

Assignment (operator=): Must provide the Strong Exception Guarantee. Use the Copy-and-Swap idiom: create a local copy, then swap its guts with this.

Rationale If an assignment operator modifies part of an object and then throws, the object is left in a "half-mutated" corrupted state. Copy-and-Swap ensures that the object remains unchanged if an error occurs during the copy.

4.8. Access Control and Declaration Order
Decision

Access: Data members must be private.

Order: Group members by access level in this order: public:, protected:, private:. Within each group, follow this layout:

Types/Enums

Constants

Constructors/Destructor

Methods

Data Members

Rationale This standardizes the "Class Interface" so that an engineer can find the public API at the very top of the file without wading through implementation details.

----------------------------------------------------------------------------------------------------
CHAPTER 5: Functions
----------------------------------------
5.1. Inputs and Outputs
Decision

Inputs: * Pass by value for cheap types (integers, floats, std::string_view, std::span).

Pass by const T& for expensive types (strings, vectors, large structs).

Outputs: * Prefer Return by Value. Modern compilers use Return Value Optimization (RVO) and Move Semantics to eliminate copying.

Use Pointers (T* out) for mandatory out-parameters or when the caller must explicitly acknowledge the modification by using the & operator.

Prohibited: Using non-const references (T& out) for output parameters.

Rationale The "Pointers for Out-Parameters" rule is a vital safety signal at the call site. In an exception-heavy codebase, seeing UpdateState(&my_obj) alerts the engineer that my_obj is in a "mutable danger zone" before an exception might trigger stack unwinding.

5.2. Write Short Functions
Decision Functions should be small, focused, and perform a single task.

Aim for functions that fit on one screen (approx. 40 lines).

If a function has more than 3-4 parameters, consider grouping them into a struct.

Rationale Long functions are difficult to make "Strongly Exception Safe." The more state a function touches, the harder it is to ensure that all changes are rolled back if a throw occurs on line 50. Small functions promote "atomic" operations.

5.3. Function Overloading
Decision Overload only when the functions perform the same logical operation on different types.

Avoid Hiding: Do not define overloads in a derived class that hide overloads in the base class.

Ambiguity: Ensure overloads are distinct enough that the compiler doesn't rely on complex implicit conversion rules to pick one.

5.4. Default Arguments
Decision * Forbidden on Virtual Functions.

Permitted on non-virtual functions where they reduce boilerplate more effectively than an overload.

Rationale Default arguments are statically bound. If a virtual function is called through a base pointer, the default argument used is from the Base class, even if the Derived implementation is executed. This is a "Fundamental Architectural Fault" that leads to silent, non-throwing logic errors.

5.5. Trailing Return Type Syntax
Decision Use the auto Func() -> ReturnType syntax only when:

Writing Lambdas.

Writing Templates where the return type is dependent on template parameters (e.g., decltype).

It significantly improves readability of complex return types (like function pointers).

Rationale Traditional return-type syntax is the standard for the majority of the codebase to maintain familiarity for engineers with a C/Assembly background.

Engineering Note on Chapter 5
Engineering Note: To satisfy the Strong Exception Guarantee, follow the "Commit at the End" pattern. Perform all risky work (allocations, parsing, calculations) in local variables first. Only after you are certain no more exceptions will be thrown should you modify the out-parameter pointer or class member. Never leave an object in a "half-changed" state.


----------------------------------------------------------------------------------------------------
CHAPTER 6: Mills-Specific Magic (Modified Google version)
----------------------------------------
6.1. Ownership and RAII
Definition Resource Acquisition Is Initialization (RAII) ensures that the lifetime of a resource is tied to the lifetime of a stack-based object.

Decision

Mandatory RAII: Every resource (heap memory, file descriptors, mutexes, sockets) must be managed by a dedicated object whose destructor handles the release.

Prohibited: Manual new and delete. Manual malloc and free.

Rationale In an exception-enabled codebase, you cannot predict the exit point of a function. RAII is the only mechanism that guarantees cleanup during stack unwinding.

6.2. Smart Pointers
Decision

std::unique_ptr: The default choice for heap ownership. It has zero runtime overhead compared to a raw pointer.

std::shared_ptr: Permitted only when true shared ownership is required (e.g., a resource shared across multiple threads where no single thread is the "parent").

Allocation: Always use std::make_unique<T>() and std::make_shared<T>().

Prohibited: std::unique_ptr<T>(new T()).

Rationale std::make_unique is not just about brevity; it is an exception-safety requirement. In expressions like Func(unique_ptr<T>(new T()), ThrowingFunc()), C++ does not guarantee the order of evaluation. If new T() happens first and ThrowingFunc() throws before the unique_ptr is constructed, you have a memory leak. make_unique fixes this atomicity.

6.3. Raw Pointers and References
Decision

Raw Pointers: Use only for observation, never for ownership. A raw pointer should never be deleted.

Optionality: If a pointer can be null, use a raw pointer. If it must not be null, use a Reference.

Rationale This creates a clear visual distinction. If an engineer sees a unique_ptr, they know they are responsible for the object's life. If they see a raw pointer T*, they know they are just looking at something that someone else is managing.

6.4. Automated Tooling (Clang-Tidy)
Decision Use Clang-Tidy for static analysis instead of the legacy cpplint.

Requirement: All code must pass the bugprone-*, performance-*, and cppcoreguidelines-* checks.

Exceptions: Specifically use the clang-analyzer-cplusplus.Move check to catch "Use-after-move" bugs.

Rationale Human reviewers are poor at catching subtle exception-safety leaks or use-after-move faults. We rely on the LLVM toolchain to enforce the "Mechanical" parts of this style guide.

Engineering Note on Chapter 6
Engineering Note: Avoid "Smart Pointer Bloat." Just because we use unique_ptr doesn't mean everything should be on the heap. Prefer the stack. Stack-allocated objects are faster, have better cache locality, and are inherently exception-safe without the overhead of an allocator call.


----------------------------------------------------------------------------------------------------
CHAPTER 7: Other C++ Features
----------------------------------------
7.1. Exceptions (The Policy)
Decision Exceptions are for exceptional cases (contract violations, resource exhaustion).

Throw by value, catch by const reference.

Guarantees: Every function must at least provide the Basic Guarantee (no leaks).

7.2. noexcept
Decision Use noexcept liberally on:

Move constructors and move assignment operators.

Destructors.

Leaf-node utility functions that cannot fail.

Rationale This allows the compiler to omit exception-handling tables and permits STL containers to optimize moves.

7.3. Rvalue References (&&) and Friends
Decision * Use Rvalue References only for move semantics or "sink" parameters.

Friends: Permitted only within the same file/module to allow a unit test or a "Factory" class to access private constructors.

7.4. Type Deduction (auto and CTAD)
Decision

auto: Use only when the type is obvious from the right-hand side (e.g., auto x = std::make_unique<T>();) or for complex iterators.

CTAD: (Class Template Argument Deduction) is permitted for simple types like std::lock_guard lock(mutex);.

7.5. Use of const, constexpr, constinit, and consteval
Decision

const: Mandatory for all non-mutated variables.

constexpr: Use for any function or value that can be computed at compile time.

consteval: Use for functions that must be computed at compile time.

constinit: Use for globals to prevent the "Static Initialization Order Fiasco."

7.6. Integer and Floating-Point Types
Decision

Integers: Use <cstdint> types (int32_t, uint64_t). Avoid unsigned for arithmetic; use it only for bit-masks.

Floating-Point: Use double by default. Use float only for massive arrays or hardware-accelerated graphics where memory bandwidth is the bottleneck.

7.7. Lambda Expressions
Decision

Length: Max 5 lines.

Captures: Always be explicit. Prohibited: [=] and [&].

Rationale: Explicit captures (e.g., [ptr, &count]) make it clear if a variable will survive the stack unwind if an exception is thrown.

7.8. Template Metaprogramming, Concepts, and Constraints
Decision

Concepts (C++20): Mandatory for all new template code. Replace std::enable_if and SFINAE with requires clauses.

Complexity: Keep templates simple. If a template requires more than two levels of recursion, it must be flagged for architectural review.

7.9. C++20 Modules and Coroutines
Decision

Modules: Prohibited for now. Stick to #include guards until toolchain support (compilers/IDEs) matures across all platforms.

Coroutines: Permitted only for high-performance asynchronous I/O. They must be wrapped in a "Task" or "Future" type that is exception-safe.

7.10. Designated Initializers (C++20)
Decision Encouraged for POD structs.

Example: Window w{.width = 800, .height = 600};

Rationale: Prevents "Parameter Scrambling" errors where two adjacent integers are swapped.

7.11. Streams and Formatting
Decision

Avoid <iostream> in performance-critical code.

Use std::format (C++20) or std::print (C++23) for logging and string construction.

7.12. Switch Statements
Decision

Always include a default case unless switching over a complete enum class.

Use [[fallthrough]] explicitly for intentional fallthrough.

Engineering Note on Chapter 7
Engineering Note: In a system using Coroutines, an exception thrown inside a co_await can be difficult to trace. Ensure your coroutine "Promises" have a unhandled_exception() implementation that logs the error before the state machine terminates. Never let an exception escape a coroutine silently.


----------------------------------------------------------------------------------------------------
CHAPTER 8: Inclusive Language
----------------------------------------
Definition The practice of using professional, neutral, and technically precise language that avoids exclusionary jargon or historical bias.

Decision
- Variable & Function Naming: Inclusive language must be applied to the code itself, not just the comments.
- Gender Neutrality: Use gender-neutral pronouns (they/them/their) in all documentation and comments.
- Cultural Idioms: Avoid localized metaphors (e.g., "sanity check," "grandfathered in," "low-hanging fruit"). Use literal descriptions instead. 
- Terminology Mapping Use the following technically descriptive alternatives:

Legacy Term:
master / slave
blacklist / whitelist
native (e.g., code)
man-hours
sanity check

Recommended Alternative:
primary / secondary, leader / follower, main
blocklist / allowlist, denylist / passlist
built-in, internal, host
person-hours, engineer-hours
confidence check, smoke test, validation



Rationale: In a global engineering environment, clarity is paramount. Culturally specific idioms or historically loaded terms create "mental speed bumps" for non-native speakers and can alienate contributors. By using Technically Descriptive Language, we ensure the code is accessible, professional, and focused strictly on technical merit.

Engineering Note: Precision is the goal. "Allowlist" is actually a better technical term than "Whitelist" because it describes exactly what the mechanism does (it allows things), rather than relying on a color-based metaphor.


----------------------------------------------------------------------------------------------------
CHAPTER 9: Naming
----------------------------------------
9.1. Choosing Names
Decision Names must be descriptive and unambiguous. Avoid "vowel-dropping" abbreviations (e.g., use error_manager instead of err_mngr).

Rationale: Symbol length has zero cost at runtime. The "cost" is entirely in the human time spent deciphering what cnt actually counts.

9.2. File Names
Decision All lowercase with underscores (snake_case).

Example: socket_manager.cc, socket_manager.h.

Note: .cc is the preferred extension for source files; .h for headers.

9.3. Type Names (Classes, Structs, Aliases)
Decision Use PascalCase.

Example: class NetworkPacket;, using BufferList = std::vector<uint8_t>;.

Concept Names (C++20): Use PascalCase to match Type Names, as they represent requirements on types. (e.g., template <typename T> concept Parsable = ...;).

9.4. Variable Names
Decision

Local Variables & Parameters: snake_case (e.g., byte_count).

Class Data Members: snake_case with a trailing underscore (e.g., retry_limit_).

Constants & Enums: kPascalCase (e.g., kMaxBufferSize).

Rationale The trailing underscore is a "Safety Signal." Inside a 50-line method, it tells the engineer instantly: "If I modify this, I am changing the object's persistent state, which might affect exception safety."

9.5. Function Names
Decision Use PascalCase (e.g., InitializeHardware()).

Exception: Standard Library "shim" functions or very small accessors may use snake_case to match STL style (e.g., size(), is_empty()).

9.6. Namespace Names
Decision All lowercase, single words preferred. Avoid nested namespaces deeper than two levels.

Example: namespace mills::net { ... }

9.7. Template Parameter Names
Decision Use PascalCase.

For generic types, a single letter is acceptable: template <typename T>.

For specific requirements, use descriptive names: template <typename AllocatorType>.

9.8. Macro Names
Decision ALL_CAPS_WITH_UNDERSCORES.

Requirement: Must be prefixed with the project name (e.g., MILLS_DEBUG_BREAK).

9.9. Exceptions to Naming Rules
Decision When implementing interfaces for external C libraries or the Standard Library (e.g., custom iterators), you may use the naming convention required by that interface (usually snake_case).

Engineering Note on Chapter 9
Engineering Note: In an exception-based system, use the Exception Class name to document the failure. Never throw a generic std::runtime_error. Use a specific name like HardwareInitializationError. This makes your catch blocks more descriptive and allows for targeted recovery logic based on the type name alone.
 

----------------------------------------------------------------------------------------------------
CHAPTER 10: Comments
----------------------------------------
10.1. Comment Style
Decision Use the // (double-slash) style for all regular comments.

/* ... */ is reserved strictly for the top-of-file legal/header block or for temporary multi-line "code-toggling" during local debugging.

Punctuation: Comments should be grammatical, starting with a capital letter and ending with a period (unless they are a "hanging" variable comment).

10.2. File Comments
Decision Every file must start with a header block including:

License: Standard project-approved license text.

Summary: A brief description of what the file provides (e.g., "Implements the high-speed TCP socket interface").

Caveats: Any global warnings (e.g., "This file is not thread-safe").

10.3. Class and Struct Comments
Decision Every class must have a documentation block immediately preceding its definition.

Focus: Describe the "Invariant" (the state the class guarantees to maintain) and the "Ownership" (does it own its pointers or just observe them?).

10.4. Function Comments (The Exception Contract)
Decision Every function declaration in a header must document its interface. For an exception-based system, the Exception Contract is mandatory.

Throws: List the specific exception types and the conditions that trigger them.

Safety Level: Explicitly state if the function provides the Strong Exception Guarantee (state is rolled back on failure) or Basic Exception Guarantee (no leaks, but state may be modified).

Rationale Since C++ lacks checked exceptions, the comment is the only way a caller can write a correct try/catch block. Failing to document this is a "Fundamental Architectural Fault."

10.5. Implementation Comments (The "Why")
Decision

Intent over Action: Do not describe what the code is doing (e.g., i++; // increment). Describe why it is doing it (e.g., i++; // Skip the header byte).

Technical Debt: Any workaround for a hardware bug, compiler quirk, or assembly-level optimization must be explicitly documented with the reasoning.

Type Bypassing: Every reinterpret_cast or const_cast must have a comment explaining why the type system is being bypassed.

10.6. Variable Comments
Decision * Class Members: Comments should be above the member or to the right if they are short.

Units: For numerical variables, the units must be documented (e.g., int timeout_ms_; // Timeout in milliseconds).

10.7. TODO Comments
Decision Format: // TODO(username): Description of task.

Requirement: Every TODO must have a name attached.

Rationale: A TODO without an owner is "abandoned code" that never gets finished.

Engineering Note on Chapter 10
Engineering Note: Use the [[deprecated("reason")]] attribute in conjunction with comments. The attribute alerts the compiler, while the comment provides the migration path for the engineer. In an exception-enabled system, if you deprecate a function because its exception behavior was unsafe, use the comment to explicitly point to its safer replacement.



----------------------------------------------------------------------------------------------------
CHAPTER 11: Formatting
----------------------------------------
11.1. General Principles
Decision

Line Length: 100 characters.

Indentation: 2 spaces. Tabs are strictly forbidden.

Brace Style: K&R (Egyptian Braces). Opening braces stay on the same line as the control statement.

11.2. Function Declarations and Calls
Decision

If a function declaration or call exceeds the line limit, wrap it.

Indentation: Use a 4-space "hanging indent" for wrapped parameters to distinguish them from the 2-space indented function body.

Return Values: Do not put the return type on a separate line unless using a trailing return type for complex templates.

11.3. Pointer and Reference Alignment
Decision Align the asterisk (*) or ampersand (&) with the Type, not the variable name.

YES: int* buffer;, const std::string& name;

NO: int *buffer;, const std::string &name;

Rationale In C++, the pointer-ness or reference-ness is a fundamental part of the Type. This alignment emphasizes the type-safety of the language.

11.4. Braced Initializer Lists
Decision

Use braced initialization {} for POD structs and simple aggregates.

For constructors with side effects (or where {} would trigger a std::initializer_list overload unexpectedly), use ().

Spacing: No spaces inside the braces for simple lists: Point p{x, y};. Use one space for complex ones: auto data = MyType{ arg1, arg2 };.

11.5. Looping and Branching
Decision

Mandatory Braces: Always use braces for if, else, for, and while statements, even if the body is a single line.

Rationale: This prevents "Dangling Else" bugs and ensures that adding a log statement or a throw to a branch later doesn't break the logic.

11.6. Constructor Initializer Lists
Decision

If the list fits on one line, keep it there.

If it wraps, put the colon on a new line, indented 2 spaces, and list each member on its own line.

Order: Members must be initialized in the order they are declared in the class.

Example

C++

MyClass::MyClass(int x, int y)
  : x_count_(x),
    y_count_(y),
    buffer_(std::make_unique<char[]>(x)) {
}
Rationale C++ initializes members in declaration order, regardless of the list order. Matching the list to the declaration order prevents "Use-before-init" bugs during complex construction.

11.7. Horizontal and Vertical Whitespace
Decision

Operators: Use one space around assignment (=), comparison (==), and binary operators (+, -, &&, ||).

Vertical: Use one blank line between function definitions and between logical sections within a function. Do not exceed two consecutive blank lines.

11.8. Preprocessor Directives
Decision Preprocessor directives (#if, #define) are always left-aligned (column 0), even if they are inside an indented block of code.

Engineering Note on Chapter 11
Engineering Note: Formatting is a tool for Code Review. When every file follows the same spacing rules, "weird" code (like a manual reinterpret_cast or a raw new) stands out visually. Use vertical whitespace to separate the "Setup," "Action," and "Commit" phases of your functions to help reviewers spot where an exception might disrupt the flow.


----------------------------------------------------------------------------------------------------
CHAPTER 12: Exceptions to the Rules
----------------------------------------
12.1. Existing Non-conformant Code
Decision You may diverge from these rules when modifying legacy codebases that follow a different standard.

Priority: Local consistency within a file or module takes precedence over this guide to maintain readability for existing maintainers.

Refactoring: When a file is modified more than 50% by line count, it must be converted to full Mills Style compliance.

12.2. Windows Code
Windows development involves specific patterns derived from the Win32 API and COM. While we use a unified style, the following rules apply for Windows-specific modules:

1. Naming and Types

Naming: Do not use Hungarian notation (e.g., dwOffset). Stick to Mills naming (offset_).

Synonyms: Use Windows types (HANDLE, HWND, DWORD) only when interacting directly with the Windows API. Convert to standard C++ types (uint32_t, bool) as soon as the data enters the "Mills" logic layer.

Strings: Use wchar_t and std::wstring for Windows Unicode APIs. Use the L"string" literal format. Avoid TCHAR unless you are supporting legacy non-Unicode environments.

2. Exception Handling (SEH vs. C++ Exceptions)

Decision: Strictly use C++ Standard Exceptions (try/catch).

SEH: Do not use __try / __except (Structured Exception Handling) for general logic.

Interop: If you must catch hardware faults (like STATUS_ACCESS_VIOLATION), use a top-level SEH-to-C++ exception translator or let the program crash. Mixing __try and try in the same function is prohibited.

3. COM and Multiple Inheritance

Decision: While multiple inheritance is generally discouraged (Chapter 4), it is permitted when implementing COM interfaces or using ATL/WTL classes.

4. Compiler Settings

Requirement: Use Warning Level 4 (/W4) or /Wall in MSVC. Treat all warnings as errors (/WX).

Language Standard: Ensure the /std:c++20 (or c++latest) flag is set to enable the features defined in Chapter 1.

5. Pragmas and Attributes

Decision: Avoid #pragma once; use standard Mills #ifndef guards.

DLLs: Use __declspec(dllexport) and __declspec(dllimport) through a project-specific macro (e.g., MILLS_API) to ensure portability.

12.3. Precompiled Headers (PCH)
Decision To maintain project portability, do not include PCH headers (like stdafx.h or pch.h) explicitly in every file.

Alternative: Use the compiler's "Force Include" (/FI on MSVC) to inject the PCH automatically. This allows the same code to be compiled on Linux/GCC without modification.

Engineering Note on Chapter 12
Engineering Note: Windows API functions often return HRESULT or DWORD error codes. Since Mills is an exception-based guide, create a small "Shim" utility to check these returns and throw a corresponding WindowsException. This keeps your high-level logic clean and consistent with our Chapter 7 policy.

Example: CheckWin32(CreateFile(...)); // Throws on failure




