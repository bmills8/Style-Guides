----------------------------------------------------------------------------------------------------
Table of Contents: Mills C++ Style Guide - Draft v1.0
----------------------------------------

CH1) C++ Version

CH2) Header Files
Self-contained Headers The #define Guard Include What You Use Forward Declarations Defining Functions in Header Files Names and Order of Includes

CH3 Scoping
Namespaces Internal Linkage Nonmember, Static Member, and Global Functions Local Variables Static and Global Variables thread_local Variables

CH4) Classes
Doing Work in Constructors Implicit Conversions Copyable and Movable Types Structs vs. Classes Structs vs. Pairs and Tuples Inheritance Operator Overloading Access Control Declaration Order

CH5) Functions
Inputs and OutputsWrite Short FunctionsFunction OverloadingDefault ArgumentsTrailing Return Type Syntax

CH6) Google-Specific Magic
Ownership and Smart Pointerscpplint

CH7) Other C++ Features
Rvalue ReferencesFriendsExceptionsnoexceptRun-Time Type Information (RTTI)CastingStreamsPreincrement and PredecrementUse of constUse of constexpr, constinit, and constevalInteger TypesFloating-Point TypesArchitecture PortabilityPreprocessor Macros0 and nullptr/NULLsizeofType Deduction (including auto)Class Template Argument DeductionDesignated InitializersLambda ExpressionsTemplate MetaprogrammingConcepts and ConstraintsC++20 modulesCoroutinesBoostDisallowed standard library featuresNonstandard ExtensionsAliasesSwitch Statements

CH8) Inclusive Language

CH9) Naming
Choosing NamesFile NamesType NamesConcept NamesVariable NamesConstant NamesFunction NamesNamespace NamesEnumerator NamesTemplate Parameter NamesMacro NamesAliasesExceptions to Naming Rules

CH10) Comments
Comment StyleFile CommentsStruct and Class CommentsFunction CommentsVariable CommentsImplementation CommentsPunctuation, Spelling, and GrammarTODO Comments

CH11) Formatting
Line LengthNon-ASCII CharactersSpaces vs. TabsFunction Declarations and DefinitionsLambda ExpressionsFloating-point LiteralsFunction CallsBraced Initializer List FormatLooping and branching statementsPointer and Reference Expressions and TypesBoolean ExpressionsReturn ValuesVariable and Array InitializationPreprocessor DirectivesClass FormatConstructor Initializer ListsNamespace FormattingHorizontal WhitespaceVertical Whitespace

CH12) Exceptions to the Rules
Existing Non-conformant Code Windows Code

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
CHAPTER 2: Header Files
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
CHAPTER 3: Scoping
----------------------------------------
3.1. Namespaces
Definition Namespaces provide a method for preventing name conflicts in large projects and organizing code into logical groups.

Pros

Collision Avoidance: Prevents "Symbol Soup" where two different libraries define a Logger class.

Organization: Clearly identifies the owner/module of a piece of code (e.g., mills::network::Socket).

Cons

Verbosity: Can lead to deeply nested code or repetitive prefixing.

Decision

All project code must be wrapped in a unique top-level namespace (e.g., mills).

Never use using namespace std; or any other namespace in a header file.

Inline namespaces are permitted only for ABI versioning.

Rationale For the assembly and C engineer, namespaces are essentially "mangled prefixes" that the compiler manages for you. They are essential for scale. However, "using" directives in headers pollute every file that includes that header, defeating the purpose of the namespace and creating silent name collisions that are difficult to debug.

3.2. Internal Linkage (Anonymous Namespaces and Static)
Definition Restricting the scope of a symbol to the single translation unit (.cc file) in which it is defined.

Decision When definitions in a .cc file do not need to be referenced outside that file, give them internal linkage by using an anonymous namespace or marking them static. Anonymous namespaces are preferred for types; static is acceptable for functions and variables.

Rationale This reduces the symbol table size and prevents the "One Definition Rule" (ODR) violations. It also allows the compiler to optimize more aggressively because it knows the symbol cannot be accessed from other units.

3.3. Local Variables
Definition Variables declared within a function or block scope.

Decision Place a function's variables in the narrowest scope possible, and initialize variables in the declaration.

Rationale In an exception-safe codebase, the narrower the scope, the better. When an exception is thrown, only objects that have been fully constructed in the current scope will have their destructors called. By delaying a variable's declaration until you have the data to initialize it, you ensure that you never have "half-initialized" local variables sitting on the stack during an unwind.

3.4. Static and Global Variables
Definition Variables with a lifetime that lasts for the duration of the program (static storage duration).

Pros

Shared State: Useful for truly global constants or "once-per-run" configuration.

Cons

Non-Deterministic Order: The order of initialization between different .cc files is undefined.

Exception Danger: If a global variable's constructor throws an exception, the program calls std::terminate() immediately. There is no try/catch block around the program's entry point that can catch a global constructor failure.

Decision

Forbidden: Class-type objects with non-trivial constructors/destructors are forbidden as global variables.

Permitted: Primitive types (PODs), pointers, and constexpr variables are allowed.

If you need a global object, use a function-local static (the "Meyers Singleton" pattern).

Rationale Global constructors are a "dark corner" of C++. Because they run before main(), any exception thrown is uncatchable. By using a function-local static, the object is initialized only the first time the function is called (thread-safe since C++11), and failures occur at a predictable point in your logic where they can be caught.

Engineering Note: A crash during global initialization is a fundamental architectural fault. It leaves no stack trace in many environments and no log entry. If you must have global-like state, use the "Leaky Singleton" or function-local static to ensure you are within the main() execution context when construction occurs.

Example

C++

// BAD: If this constructor throws, the app dies before main() starts.
DatabaseConnection g_db_connection("localhost"); 

// GOOD: Initialization happens inside a catchable context.
DatabaseConnection& GetDatabase() {
  static DatabaseConnection* db = new DatabaseConnection("localhost");
  return *db;
}
3.5. thread_local Variables
Definition Variables where each thread has its own independent instance.

Decision Use thread_local only for data that is truly thread-specific (like error buffers or random number generators). Avoid thread_local for heavy objects.

Rationale Like globals, thread_local constructors can throw. If they throw during thread creation, it may be impossible to catch depending on how the thread was spawned. Use with extreme caution.


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


4.4. Inheritance
Definition Inheritance allows a class to be defined in terms of another class, providing a way to create a hierarchy of types.

Pros

Polymorphism: Allows for code that operates on an interface rather than a concrete implementation.

Code Reuse: Shared logic can be moved to a base class.

Cons

Object Slicing: Occurs when a derived object is passed by value as a base object, losing its derived data.

Memory Corruption: If a base class is deleted via a pointer and does not have a virtual destructor, derived members are not cleaned up.

Decision

Virtual Destructors: Any class with a virtual function must have a virtual destructor. This is non-negotiable in an exception-based system to ensure cleanup during unwinding.

Composition over Inheritance: Prefer composition whenever possible. Use inheritance only when an "is-a" relationship is clearly defined.

override and final: Use the override keyword on all overridden virtual functions to catch signature mismatches at compile time.

Rationale In a codebase using exceptions, an object might be destroyed at any time. If you are holding a pointer to a base class and an exception triggers its destruction, the compiler must be able to find the derived destructor to prevent a memory leak or a partial destruction state.

4.5. Implicit Conversions
Definition The compiler's ability to automatically convert one type to another (e.g., passing an int to a function expecting a MyClass object via a single-argument constructor).

Pros

Convenience: Reduces the need for explicit casting or wrapping.

Cons

Hidden Bugs: Can lead to unexpected temporary objects being created and destroyed, which can hide performance issues or call the wrong function overload.

Decision

All single-argument constructors must be marked explicit.

Conversion operators (operator bool(), etc.) must be marked explicit.

Rationale "Magic" type conversions are dangerous for engineers debugging at the assembly/C level. Marking these as explicit forces the developer to acknowledge the conversion, making the intent clear and preventing the compiler from making assumptions that might result in a "surprise" exception from a temporary object you didn't know existed.

4.6. Operator Overloading
Definition Giving standard C++ operators (like +, ==, or []) custom logic for your classes.

Pros

Natural Syntax: Allows objects to behave like primitive types (e.g., vecA + vecB).

Cons

Implicit Complexity: A simple + can hide a massive allocation or a complex algorithm that might throw an exception.

Unintuitive Behavior: Overloading operators in ways that don't match their mathematical meaning (e.g., using << for something other than bit-shifting or streaming) confuses the reader.

Decision

Overload operators only when their meaning is obvious and matches built-in behavior.

Exception Safety: Operators should provide the Strong Exception Guarantee whenever possible. If an addition fails, neither operand should be modified.

Assignment Operators: operator= should be exception-safe. Use the "Copy-and-Swap" idiom to ensure that if an allocation fails during assignment, the original object remains untouched.

Rationale Operators are often used in contexts where the reader does not expect failure. If an operator throws, it must do so without leaving the operands in a "half-mutated" state.

Engineering Note: A throwing operator= that leaves an object in a corrupted state is a fundamental architectural fault. It makes it impossible to write transactional code. Always perform risky operations (like memory allocation) on a temporary object before swapping it into the final destination.

4.7. Access Control
Definition The use of public, protected, and private keywords.

Decision

Data members must be private. Use "Getters" and "Setters" if access is needed.

Use protected sparingly; it often indicates a design that could be better handled via composition or a public interface.

Rationale Encapsulation is the only way to protect the class invariant. If an exception is thrown, we need to know that the internal state of the object was only modified through controlled paths, making it easier to reason about the object's validity after a "catch" block.

----------------------------------------------------------------------------------------------------
CHAPTER 5: Functions
----------------------------------------
5.1. Inputs and Outputs
Definition The strategy for passing data into and receiving data out of functions.

Pros

Performance: Passing by reference avoids expensive copies of large objects.

Clarity: Clear distinction between data that is merely "read" and data that is "modified."

Cons

Implicit Modification: References can sometimes hide the fact that an input is being changed by the function.

Lifetime Risks: Passing by reference or pointer requires the caller to ensure the object outlives the function call.

Decision

Input-only parameters: Pass by const reference for class types (e.g., const std::string&). Pass by value for small, cheap-to-copy types (ints, doubles, small PODs).

Output/In-Out parameters: Use pointers (e.g., int* out_count). This makes the "modifiability" visible at the call site because the caller must use the & operator.

Return Values: Prefer returning by value. Modern compilers use Return Value Optimization (RVO) and Move Semantics to make this nearly free. If a function can fail, return std::optional<T> or throw an exception as per our Chapter 7 guidelines.

Rationale Google’s requirement for pointers for out-parameters is a "safety at the call site" rule that we will keep. By seeing UpdateStatus(&my_status), the engineer immediately knows my_status may change. This is especially helpful when tracing an exception back up the stack—you can visually identify which variables might have been mutated before the "throw" occurred.

5.2. Function Overloading
Definition Defining multiple functions with the same name but different parameter lists.

Pros

Consistency: Allows the same logical operation (e.g., Print()) to work on different types.

Cons

Call-Site Ambiguity: Complex overloading rules can make it difficult to determine which version of a function is being called, especially with implicit conversions.

Decision

Use overloading only when the functions perform the same logical task on different types.

Avoid "Hiding": Do not overload across base and derived classes in a way that hides base functions.

Avoid Ambiguity: Ensure that overloads are distinct enough that a reader (and the compiler) doesn't need to consult the ISO spec to know which one is chosen.

Rationale Overloading should simplify the API, not turn it into a puzzle. For the assembly/C engineer, overloading is just name-mangling; we want to ensure that mangling is predictable and intentional.

5.3. Default Arguments
Definition Specifying a value in the function signature that is used if the caller omits that argument.

Pros

Brevity: Reduces boilerplate for common use cases.

Cons

Hidden Logic: Can make it unclear what values are actually being used at the call site.

Virtual Function Risks: Default arguments are statically bound, which creates confusing behavior when used with virtual functions.

Decision

Forbidden on Virtual Functions: Never use default arguments on virtual methods.

Preferred over Overloading: If the logic is identical except for one optional parameter, use a default argument instead of creating a second overload.

Rationale Default arguments on virtual functions are a notorious C++ "gotcha": the function called is determined by the dynamic type, but the default argument is determined by the static type. This leads to objects behaving differently depending on whether they are accessed via a base or derived pointer—a fundamental architectural fault.

5.4. Trailing Return Type Syntax
Definition The auto FunctionName() -> ReturnType syntax introduced in C++11.

Pros

Lambda Consistency: Matches the syntax used for lambdas.

Template Clarity: Essential for certain template functions where the return type depends on the arguments (e.g., using decltype).

Decision

Use trailing return types only when necessary (e.g., in complex templates or when it significantly improves readability).

Otherwise, stick to the traditional ReturnType FunctionName() format for standard code.

Rationale While trailing return types are "modern," they can be jarring to engineers coming from a C/Assembly background. We use them as a tool for solving specific template problems, not as a blanket stylistic mandate.

Engineering Note on Chapter 5
Engineering Note: In an exception-enabled codebase, the "Output Parameter via Pointer" rule serves a dual purpose. It forces the developer to consider the Strong Exception Guarantee. If you are modifying an out-parameter pointer, ensure you do not commit changes to that memory until all code that could possibly throw has successfully completed. Never leave an out-parameter in a partially-modified state.


----------------------------------------------------------------------------------------------------
CHAPTER 6: Mills-Specific Magic (Modified Google version)
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

7.5. Casting
Definition The mechanism for explicitly converting an object from one type to another.

Pros

Necessity: Essential when dealing with legacy C APIs or polymorphic hierarchies.

Clarity: C++-style casts (static_cast, etc.) are searchable in codebases, unlike C-style casts (int)x.

Cons

Safety Risk: Casting can bypass the type system, leading to undefined behavior if done incorrectly.

Decision

Forbidden: Never use C-style casts (type)value or functional-style casts type(value).

Standard Casts: Use static_cast for safe, well-defined conversions. Use reinterpret_cast only for low-level bit-casting (common in assembly/hardware interfacing).

Dynamic Cast: As established, dynamic_cast is permitted only for navigating exception hierarchies or verified polymorphic types.

Rationale C-style casts are a "sledgehammer" that can perform a static_cast, a const_cast, and a reinterpret_cast simultaneously without telling you which one it chose. For a mid-tier engineer, the verbosity of static_cast is a feature, not a bug—it signals exactly what kind of type transformation is occurring.

7.6. Lambda Expressions
Definition Anonymous function objects capable of capturing variables from the surrounding scope.

Pros

Brevity: Allows for small, localized logic (especially for STL algorithms) without cluttering the namespace.

Readability: Keeps the logic close to where it is used.

Cons

Capture Risks: Capturing by reference ([&]) in a lambda that outlives the current scope is a guaranteed "use-after-free" bug.

Complexity: Large lambdas become difficult to read and debug.

Decision

Keep them short: If a lambda exceeds 5 lines, consider making it a named function.

Explicit Captures: Avoid [=] or [&]. Always explicitly list the variables you are capturing (e.g., [ptr, &count]).

Exception Specification: If a lambda cannot throw, mark it noexcept.

Rationale Lambdas are powerful but dangerous. Explicit captures force the engineer to think about the lifetime of the captured variables. This is especially critical when an exception is thrown; we need to be certain that captured references are still valid during the stack unwind.

7.7. Use of const
Definition A qualifier that prevents a variable or member function from being modified.

Pros

Compiler Optimization: Allows the compiler to make assumptions about data stability.

Thread Safety: const objects are inherently thread-safe for concurrent reads.

Documentation: Tells the reader (and the compiler) that this value is an "input" only.

Decision

"Const Everything": Use const by default for all variables that are not intended to be modified.

Const Member Functions: Mark all member functions const unless they explicitly modify the object's observable state.

Rationale In an exception-safe system, const is your best friend. It significantly reduces the "state space" you have to worry about when a function fails. If an object is const, you know for a fact its state didn't change before the exception was thrown, satisfying the Strong Exception Guarantee by default.

7.8. Preincrement and Predecrement
Decision Use prefix forms (++i, --i) unless the postfix form (i++) is explicitly required by the logic.

Rationale For simple integers, it doesn't matter. But for iterators (common in the STL), postfix increment creates a temporary copy of the iterator to return the "old" value. In an exception-enabled codebase, creating unnecessary temporaries is a performance tax we should avoid.

Engineering Note on Chapter 7
Engineering Note: When using reinterpret_cast for hardware or assembly interfacing, you are stepping outside the safety net of the C++ type system. Exceptions thrown across a reinterpret_cast boundary (e.g., from a callback) can lead to unrecoverable state corruption. Ensure that any C-style callback interfaces are marked extern "C" and have a "top-level" try/catch(...) to prevent exceptions from escaping into layers that cannot handle them.

CHAPTER 7: Other C++ Features (Cleanup)
7.9. Integer Types
Definition The use of built-in integer types (int, long, size_t, etc.).

Pros

Standardization: Using fixed-width types ensures the code behaves identically on 32-bit and 64-bit architectures.

Cons

Overflow/Underflow: Silent wrapping of integers can lead to critical security vulnerabilities or logic errors that are difficult to catch with exceptions.

Decision

Fixed-width types: Use <cstdint> types (int32_t, uint64_t) for all data where the size matters (e.g., file formats, network protocols, or hardware registers).

int: Use int only for small, general-purpose loop counters or indices where the value is guaranteed to stay within a 16-bit range.

Unsigned Integers: Avoid unsigned integers for arithmetic unless you are performing bitwise operations. Use them only for quantities that are inherently non-negative, but be wary of subtraction (which can wrap to a massive positive value).

Rationale For engineers working near the assembly layer, "size matters." Using int is ambiguous because its size can change between platforms. Fixed-width types provide the predictability needed for low-level systems.

7.10. Preprocessor Macros
Definition Code that is processed by the preprocessor before actual compilation (e.g., #define).

Pros

Global Configuration: Useful for feature flags or platform-specific switches.

Cons

Scope-blind: Macros do not respect namespaces or classes; they can accidentally replace code they weren't meant to touch.

Non-debuggable: You cannot step into a macro with a debugger.

Exception Hostile: Macros can hide multiple statements, making it impossible to determine which line inside a macro threw an exception.

Decision

Avoid Macros: Use const, constexpr, or inline functions instead of #define whenever possible.

If required: Use a unique prefix (e.g., MILLS_) and ALL_CAPS naming. Use the do { ... } while (0) idiom for multi-statement macros to ensure they behave like single statements.

Rationale Macros are a "legacy" tool that should be used sparingly in Modern C++. By using constexpr, you get the same performance benefits as a macro but with the full protection of the C++ type system and exception-handling visibility.

7.11. Type Deduction (auto)
Definition Allowing the compiler to deduce the type of a variable from its initializer.

Pros

Brevity: Avoids long, repetitive type names (especially with iterators or template types).

Refactoring Ease: If a function's return type changes, the auto variable updates automatically.

Cons

Obfuscation: It can be hard for a human reader to know what the type actually is without IDE assistance.

Decision

Use auto only when it improves readability. This usually means:

For iterators and template-heavy types.

When the type is explicitly mentioned in the same line (e.g., auto logger = std::make_unique<Logger>();).

Avoid auto for primitive types (ints, bools) where the type name is already short and clear.

Rationale We want the code to be scannable. If an engineer has to hover their mouse over every variable just to find out if it’s a pointer or an object, we’ve failed at clarity.

7.12. constexpr and constinit
Definition Keywords that ensure values or functions are evaluated at compile-time.

Decision Use constexpr for all constants and functions that can be computed at compile-time. Use constinit for global/static variables that must be initialized at compile-time to avoid the "Static Initialization Order Fiasco."

Rationale Everything done at compile-time is work that doesn't need to be done at runtime. Furthermore, constexpr functions cannot throw exceptions that propagate to runtime; if a constexpr evaluation fails, it is a compile-time error. This is the ultimate "Zero-Cost" safety mechanism.

Engineering Note on Chapter 7
Engineering Note: When using fixed-width integers for hardware offsets or buffer sizes, remember that C++ exceptions do not catch integer overflows. If you are calculating a buffer size that might overflow, you must check the bounds manually and throw an std::overflow_error. Implicit wrapping is a silent killer in systems-level C++.


----------------------------------------------------------------------------------------------------
CHAPTER 8: Inclusive Language
----------------------------------------
Definition The practice of using language that is professional, neutral, and avoids jargon or metaphors that are exclusionary or rooted in historical bias.

Decision

Code and Comments: Use gender-neutral language (e.g., "they/them" instead of "he/she").

Technical Terms: Avoid terms like "master/slave" or "blacklist/whitelist." Use technically descriptive alternatives such as "primary/secondary," "leader/follower," or "allowlist/blocklist."

Clarity over Idiom: Avoid culturally specific idioms (e.g., "ballpark estimate") that may confuse non-native speakers or engineers in different regions.

Rationale In a global engineering environment, clear and inclusive language is an asset to maintainability. It ensures that the documentation and code are accessible to all engineers, regardless of background, and focuses strictly on technical merit.


----------------------------------------------------------------------------------------------------
CHAPTER 9: Naming
----------------------------------------
9.1. General Naming Rules
Decision

Names should be descriptive; avoid cryptic abbreviations (e.g., use count instead of cnt).

For engineers at the C/Assembly level: do not be afraid of long names for complex objects. The compiler handles the symbol length; the human needs the clarity.

9.2. File Names
Decision

File names must be all lowercase and may include underscores (_).

Format: my_useful_class.cc, my_useful_class.h.

9.3. Type Names (Classes, Structs, Type Aliases)
Decision

Type names must start with a capital letter and have a capital letter for each new word (PascalCase).

Format: class RequestParser { ... };

9.4. Variable Names
Decision

Local Variables: All lowercase with underscores (snake_case).

Class Data Members: snake_case with a trailing underscore.

Constants: k followed by PascalCase (kPascalCase).

Example

C++

const int kMaxBufferLength = 1024;

class DataHandler {
 private:
  int error_count_; // Trailing underscore for members
 public:
  void Process(int retry_limit) { // No underscore for locals/params
    int local_count = 0;
  }
};
9.5. Function Names
Decision

Regular functions and class methods must use PascalCase (e.g., OpenFile(), CalculateTotal()).

Exception: Very small accessors (getters/setters) may use snake_case to match the variable name (e.g., count() and set_count()), though PascalCase is preferred for consistency.

9.6. Namespace Names
Decision

Namespaces must be all lowercase (e.g., namespace mills { ... }).

9.7. Enumerator Names
Decision

Use kPascalCase, similar to constants. Avoid the old C-style ALL_CAPS for enums to prevent collisions with preprocessor macros.

9.8. Exception Class Names
Decision

Exception classes must follow the PascalCase naming for types but should explicitly include the suffix Error or Exception.

Format: class DeviceTimeoutException : public std::runtime_error { ... };

Rationale The "Exception" suffix is an engineering flag. When a developer sees throw DeviceTimeoutException();, the naming clearly indicates that this object is intended for the stack-unwinding mechanism, not as a standard return type.

9.9. Macro Names
Decision

If a macro is absolutely necessary (see Chapter 7), it must be ALL_CAPS_WITH_UNDERSCORES.

Engineering Note on Naming
Engineering Note: In a system using exceptions, the "Function Name" rule is your first line of defense. If a function is marked noexcept, it is often helpful to denote that in the documentation or, in high-risk low-level code, by a suffix if the architectural fault of a throw is particularly dangerous. However, the best practice is to rely on const-correctness and noexcept keywords rather than Hungarian notation.


----------------------------------------------------------------------------------------------------
CHAPTER 10: Comments
----------------------------------------
10.1. Comment Style
Decision

Use the // (double-slash) style for almost all comments.

Use /* */ (block comments) only for high-level file headers or for "commenting out" large blocks of code during temporary local debugging.

Rationale Double-slash comments are easier to manage and don't suffer from "nesting" issues. They provide a clean, consistent look across the codebase.

10.2. File Comments
Definition A top-level comment block at the beginning of every .h and .cc file.

Decision Every file must start with a comment containing:

The license/copyright notice.

The purpose of the file (a 1-2 sentence high-level summary).

The primary author or owner team.

10.3. Class and Struct Comments
Decision Every non-trivial class definition must have a comment describing what it is and what it is used for. If a class is thread-safe or has specific requirements for destruction, it must be noted here.

10.4. Function Comments (The Exception Contract)
Definition The documentation preceding a function declaration that defines its interface.

Decision In addition to describing inputs and outputs, every function that can throw an exception must document its Exception Contract.

Throws: List the specific exceptions that may be thrown and under what conditions.

Safety: Note if the function provides the Strong Exception Guarantee (transactional) or only the Basic Guarantee.

Rationale Since C++ does not have "checked exceptions" like Java, the comment is the only way a caller knows how to safely use your function.

Engineering Note: If a function is marked noexcept, you do not need a "Throws" section. However, for functions that do throw, failing to document the contract is a fundamental architectural fault. It forces the caller to read your implementation to know how to handle errors, violating the principle of encapsulation.

Example

C++

// GOOD: Clear contract documentation
// Reads the configuration file from the given path.
// 
// Inputs: path - The absolute path to the .conf file.
// Returns: A Config object.
// Throws: 
//   - FileSystemException if the file is missing.
//   - ParseException if the format is invalid.
// Note: Provides the Strong Exception Guarantee.
Config LoadConfig(const std::string& path);
10.5. Implementation Comments (The "Why")
Decision

Do not comment the obvious (e.g., i++; // Increment i).

Do comment "tricky" logic, workarounds for compiler bugs, or performance optimizations that might look unintuitive.

If you use reinterpret_cast or a const_cast, you must provide a comment explaining why the type system is being bypassed.

10.6. TODO Comments
Decision Use TODO(username): description for temporary code or tasks that need to be completed later. Do not leave "naked" TODOs without a name/owner.



----------------------------------------------------------------------------------------------------
CHAPTER 11:
----------------------------------------
CHAPTER 11: Formatting
Definition The physical layout of the code (whitespace, braces, line lengths).

Decision

Line Length: 100 characters. Modern monitors can handle more than the traditional 80, but 100 remains scannable without horizontal scrolling.

Indentation: Use Spaces only. 2 spaces per indentation level. Tabs are forbidden to ensure consistency across different IDEs and terminal viewers.

Brace Style: Use K&R Style (the "Egyptian" brace). The opening brace stays on the same line as the statement; the closing brace starts a new line.

Rationale Formatting is about Muscle Memory. Once the team agrees on a style, your brain stops "reading" the formatting and starts reading the logic. We choose 2 spaces and K&R to maximize vertical screen real estate.

Example

C++

// GOOD: 2-space indentation, K&R braces
void MyFunction(int value) {
  if (value > kLimit) {
    DoSomething();
  } else {
    DoNothing();
  }
}
11.1. Function Calls and Long Lines
Decision If a function call or declaration is too long for one line, wrap it and indent the subsequent lines by 4 spaces (double indentation) to distinguish them from the function body.

CHAPTER 12: Exceptions to the Rules
Definition Recognizing that no style guide covers 100% of cases.

Decision

Legacy Code: When modifying a file that was written in a different style (e.g., an old C library), match the existing style of that file to maintain local consistency.

Generated Code: Code produced by tools (like Protocol Buffers or scanners) is exempt from these rules.

Hardware Constraints: In extreme assembly/embedded contexts where exceptions are physically unsupported by the hardware, the "Exceptions Permitted" rule is naturally suspended.


----------------------------------------------------------------------------------------------------
CHAPTER 12:
----------------------------------------







