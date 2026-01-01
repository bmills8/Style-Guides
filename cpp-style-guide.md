Mills C++ Style Guide

CH1) C++ Version

CH2) Header Files
Self-contained HeadersThe #define GuardInclude What You UseForward DeclarationsDefining Functions in Header FilesNames and Order of Includes

CH3 Scoping
NamespacesInternal LinkageNonmember, Static Member, and Global FunctionsLocal VariablesStatic and Global Variablesthread_local Variables

CH4) Classes
Doing Work in ConstructorsImplicit ConversionsCopyable and Movable TypesStructs vs. ClassesStructs vs. Pairs and TuplesInheritanceOperator OverloadingAccess ControlDeclaration Order

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








---------------------------------------- CHAPTER 7 ----------------------------------------


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















