C# Style Guide

TODO:  back  up and punt with C# at Google Style Guide. incorporate and expand upon. 

Revision History
This section records published revisions of this document. Only changes that materially affect meaning, guidance, or rules are recorded here.

Version	Time Zone of Revision	Summary
v2025.12.26.0852	ET/UTC -5	Initial repository and structure.
v2025.12.26.1130	ET/UTC -5	First complete draft of core guidelines.

Table of Contents
1.	Philosophy + Scope
2.	Definitions
3.	Types & API Design
4.	Error Handling
5.	Async & Concurrency
6.	Testing
7.	Control Flow
8.	Organization
9.	Naming
10.	Tooling / Formatting


1. Philosophy + Scope

This guide exists to help engineers write clear, safe, idiomatic C# using Visual Studio and the .NET platform.
It prioritizes readability, correctness, and long-term maintainability over cleverness, premature optimization, or personal preference. Rules are designed to reduce common sources of bugs, friction, and ambiguity while preserving sound engineering judgment.
This guide is .NET-first:
•	Established C# and .NET conventions take precedence.
•	Google’s C++ style guide influences the intent of this document—discipline, clarity, and consistency—but not its syntax or idioms.
•	C++ patterns are not force-fitted into C#.
Core values
Optimize for readers
Code is read far more often than it is written. Prefer explicit, straightforward constructs.
Consistency over preference
Consistency reduces cognitive load and review friction. Follow established conventions.
Correctness before convenience
Fail fast when invariants are violated. Make invalid states hard to represent.
Practicality
Rules should be easy to follow and hard to misuse. Prefer guidance that aligns with IDE and tooling behavior.
Exceptions (intentional and disciplined)
This guide supports exceptions as a fundamental part of the .NET error model.
•	Exceptions represent exceptional conditions, not routine control flow.
•	Expected failure modes should use explicit, non-exception mechanisms.
•	Exceptions should be specific, meaningful, and handled deliberately at boundaries.
This approach balances robustness with clarity and aligns with established .NET practices.
2. Definitions
The following terms are used throughout this guide with specific and intentional meaning to avoid ambiguity and repeated explanation.
Public API
Any type, member, or behavior accessible outside its defining assembly, or intended for reuse by other components or consumers.
Public APIs are long-lived contracts and require higher standards for stability, documentation, and backward compatibility.
Internal API
Code intended for use within a single assembly or tightly coupled set of assemblies.
Internal APIs may evolve more freely than public APIs but are still expected to be readable, consistent, and testable.
Boundary
A point where control, responsibility, or trust changes.
Examples include:
•	Public method entry points
•	File system, network, or database access
•	Interop or unsafe code
•	Thread or task transitions
•	External libraries or services
Boundaries are the primary locations for validation, exception translation, logging, and cancellation handling.
Invariant
A condition that must always hold true for an object or system to be in a valid state.
Examples:
•	Required constructor parameters are non-null
•	A value object enforces its domain constraints
•	A collection wrapper never exposes invalid elements
Invariants should be enforced by construction whenever possible and violations should fail fast.
Expected Failure
A failure mode that is part of normal, correct program flow and is likely to occur during valid usage.
Examples:
•	Parsing user input
•	Cache misses
•	Conditional lookups
•	Resource not found
If callers are expected to handle the condition routinely, it is not exceptional.
Expected failures should generally be modeled without exceptions.
Exceptional Condition
A failure that represents an unexpected, invalid, or unrecoverable state.
Examples:
•	Violated invariants
•	Misuse of an API
•	Corrupted or inconsistent data
•	Environmental failures that prevent progress
Exceptional conditions are appropriate for exceptions.
Error Translation
The act of converting one form of failure into another at a boundary.
This may include:
•	Wrapping low-level exceptions in domain-specific exceptions
•	Converting exceptions into error responses
•	Preserving causal context while hiding implementation details
Fail Fast
The practice of detecting invalid states or incorrect usage as early as possible and terminating the current operation immediately.
Fail-fast behavior improves debuggability and prevents error propagation.
Hot Path
Code that executes frequently or is performance sensitive.
Hot paths may justify deviations from otherwise preferred patterns, but only when tradeoffs are understood, measured, and documented.
3. Types & API Design
This section defines how types and APIs are designed, with particular emphasis on public APIs. Public APIs are long-lived contracts; decisions made here are difficult to reverse and should favor clarity, correctness, and stability.
3.1 Public API Design
Principles
•	Public APIs are contracts, not implementation details.
•	Public APIs should be hard to misuse and easy to understand.
•	Once published, APIs should change slowly and deliberately.
Must
•	Minimize public surface area.
•	Public APIs must express nullability accurately.
•	Public APIs must not expose mutable internal state.
•	Public APIs must not rely on undocumented side effects.
•	Exceptions thrown by public APIs must be intentional and meaningful.
Should
•	Prefer clarity over flexibility.
•	Prefer explicit types over clever abstractions.
•	Prefer stable contracts over "convenient" shortcuts.
•	Validate inputs at API boundaries.
•	Keep public APIs small and cohesive.
May
•	Provide convenience overloads when they clearly improve usability.
•	Introduce abstractions when multiple consumers justify them.
Visibility and Surface Area
Guidance
•	Default to the narrowest visibility possible.
•	Make types public only when they are intended for external use.
Don’t
public class Parser
{
    public void ParseInternal(Input input) { }
}
Do
public class Parser
{
    public ParseResult Parse(Input input) { }

    private void ParseInternal(Input input) { }
}
Nullability as Part of the Contract
Guidance
•	Nullability annotations are part of the API contract.
•	If null is allowed, it must be expressed in the signature.
•	Do not rely on documentation or comments to explain null behavior.
Don’t
public Customer GetCustomer(string id)
{
    // may return null
}
Do
public Customer? GetCustomer(string id)
{
}
Exceptions in Public APIs
Guidance
•	Exceptions represent exceptional conditions, not expected outcomes.
•	Public APIs should not leak low-level or infrastructure exceptions.
•	If an exception can occur during valid usage, it should be documented.
Don’t
public Customer Load(string id)
{
    return _repository.Load(id); // may throw SqlException
}
Do
public Customer Load(string id)
{
    try
    {
        return _repository.Load(id);
    }
    catch (RepositoryUnavailableException ex)
    {
        throw new CustomerStoreUnavailableException(id, ex);
    }
}
Tradeoff: Exceptions vs Try-Pattern vs Result Types
Options
•	Throwing exceptions
•	TryXxx methods
•	Explicit result types
Pros
•	Exceptions: natural in .NET; clear failure signaling.
•	Try-pattern: avoids exceptions for expected failures; familiar to C# developers.
•	Result types: explicit and expressive; easy to compose and test.
Cons
•	Exceptions: expensive if used for control flow; easy to overuse.
•	Try-pattern: limited expressiveness; can become noisy.
•	Result types: additional types and ceremony; less idiomatic in some APIs.
Decision
•	Use exceptions for exceptional conditions.
•	Use Try-pattern when failure is expected and common.
•	Use result types when richer failure information is required.
Don’t
public int Parse(string text)
{
    return int.Parse(text);
}
Do
public bool TryParse(string text, out int value)
{
    return int.TryParse(text, out value);
}
Public Return Types and Collections
Guidance
•	Avoid returning mutable collections directly.
•	Prefer read-only abstractions for public APIs.
Tradeoff: IEnumerable<T> vs IReadOnlyList<T>
Pros
•	IEnumerable<T>: flexible; supports deferred execution.
•	IReadOnlyList<T>: explicit size and indexing; clear ownership and materialization.
Cons
•	IEnumerable<T>: may hide deferred execution; can be enumerated multiple times unintentionally.
•	IReadOnlyList<T>: requires materialization.
Decision
•	Use IReadOnlyList<T> when returning a materialized collection.
•	Use IEnumerable<T> only when deferred execution is intentional and documented.
Don’t
public List<Order> GetOrders()
{
    return _orders;
}
Do
public IReadOnlyList<Order> GetOrders()
{
    return _orders;
}
Async in Public APIs
Guidance
•	Public APIs performing I/O or long-running work should be async.
•	Async behavior must be explicit in naming and signatures.
Don’t
public Data Load()
{
    return LoadAsync().Result;
}
Do
public Task<Data> LoadAsync(CancellationToken cancellationToken)
{
}
Public API Versioning Mindset
Guidance
•	Removing or changing public APIs is a breaking change.
•	Prefer adding new APIs over changing existing ones.
•	Obsolete APIs deliberately and document replacements.
Summary
Public APIs should be boring, predictable, and difficult to misuse.
Cleverness belongs in implementation details, not contracts.
4. Error Handling
This section defines how errors are represented, propagated, and handled.
Error handling should make failures obvious, actionable, and contained, without obscuring normal control flow.
4.1 Principles
•	Errors should be explicit, not implicit.
•	Failures should be detected as early as possible.
•	Error handling should preserve context, not erase it.
•	The handling strategy should match the type of failure.
4.2 Categories of Failure
Understanding the category of failure determines the correct handling strategy.
Invariant Violations
•	Represent programmer errors.
•	Must fail fast.
•	Should not be recovered from locally.
Expected Failures
•	Occur during valid program flow.
•	Should not use exceptions.
•	Must be handled explicitly.
Exceptional Conditions
•	Unexpected or unrecoverable failures.
•	Appropriately represented by exceptions.
•	Often handled at boundaries.
4.3 Guard Clauses and Validation
Guidance
•	Validate inputs at boundaries.
•	Fail fast when invariants are violated.
•	Prefer built-in guard helpers where available.
Don’t
public void Process(Order order)
{
    if (order == null)
        return;

    // silent failure
}
Do
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);

    // proceed with valid state
}
4.4 Exceptions
When to Throw Exceptions
Throw exceptions when:
•	An invariant is violated
•	An API is misused
•	Progress cannot continue safely
•	A failure is truly unexpected
Don’t
if (!File.Exists(path))
    throw new Exception("File missing");
Do
if (!File.Exists(path))
    throw new FileNotFoundException("Input file not found.", path);
4.5 Expected Failure Patterns
Try-Pattern
Use when failure is common and expected.
Don’t
public int Parse(string text)
{
    return int.Parse(text);
}
Do
public bool TryParse(string text, out int value)
{
    return int.TryParse(text, out value);
}
Nullable Returns
Use when absence is a normal outcome.
Don’t
public Customer GetCustomer(string id)
{
    throw new CustomerNotFoundException(id);
}
Do
public Customer? GetCustomer(string id)
{
}
4.6 Tradeoff: Exceptions vs Try-Pattern vs Result Types
Pros
•	Exceptions: idiomatic in .NET; propagate naturally; preserve stack traces.
•	Try-pattern: cheap for expected failures; familiar and concise.
•	Result types: explicit failure modeling; rich error information; easy to test and compose.
Cons
•	Exceptions: expensive if misused; easy to overuse for control flow.
•	Try-pattern: limited expressiveness; awkward for complex failures.
•	Result types: additional types and ceremony; less common in some C# codebases.
Decision
•	Use exceptions for exceptional conditions.
•	Use Try-pattern for common, expected failures.
•	Use result types when failure details matter and justify the complexity.
4.7 Catching Exceptions
Guidance
•	Catch only exceptions you can handle meaningfully.
•	Do not catch broad exceptions in the middle of the stack.
•	Handle broad exceptions only at boundaries.
Don’t
try
{
    DoWork();
}
catch
{
    // swallow exception
}
Do
try
{
    DoWork();
}
catch (IOException ex)
{
    RecoverFromIoFailure(ex);
}
4.8 Exception Wrapping and Translation
Guidance
•	Translate exceptions at boundaries.
•	Preserve the original exception as InnerException.
•	Avoid leaking low-level implementation details.
Don’t
catch (SqlException)
{
    throw new Exception("Database error");
}
Do
catch (SqlException ex)
{
    throw new DataStoreUnavailableException("Customer database unavailable.", ex);
}
4.9 Logging and Exceptions
Guidance
•	Do not log an exception and then rethrow it unless adding meaningful context.
•	Avoid logging the same exception repeatedly up the stack.
•	Log at boundaries where ownership exists.
If you do log and rethrow, log once and include actionable context (for example, operation name and relevant identifiers).
Don’t
catch (Exception ex)
{
    _logger.LogError(ex, "Error");
    throw;
}
Do
catch (Exception ex)
{
    throw new OrderProcessingException(orderId, ex);
}
4.10 Cleanup and Resource Safety
Guidance
•	Use using / try-finally for cleanup.
•	Cleanup must occur even when exceptions are thrown.
Don’t
var stream = File.OpenRead(path);
// exception here leaks stream
Do
using var stream = File.OpenRead(path);
4.11 Summary
Error handling should make failures visible, intentional, and contained.
Silent failures and ambiguous error paths are worse than loud, explicit ones.
5. Async & Concurrency
This section defines how asynchronous code and concurrency are used.
Async and concurrency improve scalability and responsiveness, but they also introduce complexity. These rules favor correctness, clarity, and predictability over clever parallelism.
5.1 Principles
•	Async behavior must be explicit.
•	Prefer async I/O over blocking calls.
•	Concurrency should be bounded and intentional.
•	Avoid hidden thread-affinity assumptions.
•	Correctness always outweighs parallelism.
5.2 Async APIs
Guidance
•	Methods performing I/O or long-running work should be async.
•	Async methods must return Task, Task<T>, or ValueTask.
•	Async method names must end with Async.
Don’t
public Data Load()
{
    return LoadAsync().Result;
}
Do
public async Task<Data> LoadAsync()
{
    return await LoadInternalAsync();
}
5.3 Async All the Way
Guidance
•	Once async is introduced, propagate it through the call chain.
•	Do not block on async operations.
Don’t
var result = LoadAsync().Wait();
Don’t
var result = LoadAsync().Result;
Do
var result = await LoadAsync();
Blocking async code is a common source of deadlocks and thread-pool starvation.
5.4 Cancellation
Guidance
•	Cancellation is part of the async contract.
•	Accept a CancellationToken for cancellable operations.
•	Name it cancellationToken and place it last.
•	Propagate cancellation downstream.
Don’t
await Task.Delay(1000);
Do
await Task.Delay(1000, cancellationToken);
Cancellation and Exceptions
Do not swallow cancellation.
Don’t
catch (OperationCanceledException)
{
    // ignore
}
Do
catch (OperationCanceledException)
{
    throw;
}
5.5 Tradeoff: Task vs ValueTask
Pros
•	Task: simple and idiomatic; easy to compose; safe to await multiple times.
•	ValueTask: reduces allocations in hot paths; useful for frequently synchronous completions.
Cons
•	Task: allocation overhead in high-frequency paths.
•	ValueTask: more complex semantics; must not be awaited multiple times; easy to misuse.
Decision
•	Use Task by default.
•	Use ValueTask only when performance is measured and justifies the added complexity.
5.6 Concurrency and Shared State
Guidance
•	Types are not thread-safe by default.
•	Avoid shared mutable state.
•	Prefer immutability.
Don’t
public class Counter
{
    public int Value;
}
Do
public class Counter
{
    private int _value;

    public int Increment() => Interlocked.Increment(ref _value);
}
5.7 Locks and Synchronization
Guidance
•	Keep synchronization scopes small.
•	Never await inside a lock.
•	Never lock on this or public objects.
Don’t
lock (_gate)
{
    await SaveAsync();
}
Do
await _semaphore.WaitAsync(cancellationToken);
try
{
    await SaveAsync();
}
finally
{
    _semaphore.Release();
}
5.8 Parallelism
Guidance
•	Do not parallelize by default.
•	Parallelism must not change observable behavior.
•	Bound concurrency explicitly.
Don’t
items.AsParallel().ForAll(Process);
Do
foreach (var item in items)
{
    Process(item);
}
Tradeoff: Sequential vs Parallel Execution
Pros (Parallel)
•	Improved throughput for CPU-bound workloads.
Cons (Parallel)
•	Increased complexity.
•	Harder debugging.
•	Risk of contention and nondeterminism.
Decision
•	Use parallelism only when performance benefits are clear and measured.
5.9 Task Coordination
Guidance
•	Use Task.WhenAll to await multiple tasks.
•	Be explicit about error behavior.
Don’t
foreach (var item in items)
{
    _ = ProcessAsync(item);
}
Do
await Task.WhenAll(items.Select(ProcessAsync));
5.10 Fire-and-Forget Tasks
Guidance
•	Avoid fire-and-forget tasks.
•	If used, ownership and lifecycle must be explicit.
Don’t
ProcessAsync(item);
Do
_ = ProcessAsync(item).ContinueWith(
    t => LogFailure(t.Exception),
    TaskContinuationOptions.OnlyOnFaulted);
5.11 UI Thread Affinity
Guidance
•	Do not block the UI thread.
•	Marshal back to the UI thread only when updating UI state.
•	Keep UI event handlers short and async.
5.12 Summary
Async code should be obviously async, cancellable, and bounded.
If concurrency does not clearly improve correctness or performance, it likely adds unnecessary complexity.
6. Testing
Tests exist to increase confidence that the system behaves correctly. Good tests are readable, deterministic, and focused on behavior, not implementation.
6.1 Principles
•	Tests should validate public behavior and contracts.
•	Prefer determinism over realism when they conflict.
•	Make failures actionable (clear names, clear assertions).
•	Avoid brittle tests that break during refactors.
6.2 What to Test
Must
•	Test behavior that affects correctness, safety, or user-visible outcomes.
•	Test boundary conditions and failure modes that matter.
•	Tests must be deterministic and repeatable.
Should
•	Prefer fewer, higher-signal tests over lots of trivial ones.
•	Test "shape of behavior," not internal steps.
May
•	Skip tests for trivial pass-through code where the risk is negligible.
6.3 Test Structure
Guidance
Use an Arrange / Act / Assert structure (explicitly or implicitly).
Do
// Arrange
var parser = CreateParser();
var input = CreateValidInput();

// Act
var result = parser.Parse(input);

// Assert
result.Should().NotBeNull();
Don’t
var parser = CreateParser();
var result = parser.Parse(CreateValidInput());
result.Should().NotBeNull();
(The “Don’t” isn’t wrong—it’s just harder to scan and maintain.)
6.4 Test Naming
Must
•	Test names must describe behavior and outcome.
Should
Prefer the pattern:
•	Method_WhenCondition_ThenOutcome
•	Behavior_WhenCondition
Do
•	Parse_WhenInputIsInvalid_ThrowsInvalidTemplateException
•	TryGetCustomer_WhenMissing_ReturnsFalse
Don’t
•	Test1
•	ParserWorks
6.5 Assertions
Guidance
•	Assert on the most important behavior first.
•	Prefer specific assertions over broad ones.
Don’t
action.Should().Throw<Exception>();
Do
action.Should().Throw<InvalidOperationException>();
Should
•	Avoid asserting exact exception messages unless they are part of the contract.
6.6 Async Testing
Must
•	Async code must be tested with async/await.
•	Do not block on async (.Result, .Wait()).
Don’t
[Fact]
public void Loads()
{
    var data = LoadAsync().Result;
}
Do
[Fact]
public async Task LoadsAsync()
{
    var data = await LoadAsync();
}
Should
•	Avoid timing-based tests (Task.Delay) when possible.
•	Prefer deterministic coordination (signals, fakes, explicit triggers).
6.7 Exceptions in Tests
Guidance
•	Test exceptions as part of the contract when behavior requires it.
•	Assert on the correct type and relevant details.
Do
[Fact]
public void Parse_WhenInvalid_Throws()
{
    var parser = CreateParser();

    Action act = () => parser.Parse(CreateInvalidInput());

    act.Should().Throw<InvalidTemplateException>();
}
6.8 Mocks vs Fakes vs Real Dependencies
Tradeoff: Mocks vs Fakes
Pros (Mocks)
•	Quick to set up for boundary interactions.
•	Great for verifying calls and parameters.
Cons (Mocks)
•	Can encode implementation details.
•	Can produce brittle tests that block refactoring.
Pros (Fakes)
•	Often more behavior-focused.
•	More resilient to refactors.
Cons (Fakes)
•	May require more code upfront.
Decision
•	Prefer fakes for stable, reusable test dependencies.
•	Use mocks at true boundaries when interaction verification is the point of the test.
•	Avoid mocking internals or the system under test.
Don’t
// Mocking the thing you're testing is almost never meaningful.
var sut = new Mock<OrderProcessor>();
Do
// Mock the boundary.
var store = new Mock<IOrderStore>();
var sut = new OrderProcessor(store.Object);
6.9 Test Data
Should
•	Prefer clear, intention-revealing inputs over clever generators.
•	Use builders/helpers when they improve clarity and reduce noise.
Do
var order = OrderBuilder.Valid().WithItem("Widget").Build();
Don’t
var order = new Order { /* 30 fields */ };
6.10 Integration Tests
Must
•	Integration tests must clean up resources they create.
•	Integration tests must not depend on prior runs.
Should
•	Keep integration tests separate from unit tests (by folder, namespace, or naming).
•	Use realistic boundaries, but keep outcomes deterministic.
6.11 Concurrency Testing
Must
•	Tests must not rely on thread scheduling or timing to “probably pass.”
Should
•	Prefer deterministic coordination (events, barriers, controlled queues).
•	Treat stress/soak tests as separate from unit tests.
6.12 Flaky Tests
Must
•	Flaky tests must be fixed or removed.
•	“Just rerun it” is not an acceptable workflow.
6.13 Summary
Good tests are readable, deterministic, and behavior-focused.
A test suite you can’t trust is worse than no test suite at all.
7. Control Flow
Control flow should be obvious at a glance. Readers should be able to identify the happy path, error paths, and exit conditions without mentally simulating the code.
7.1 Principles
•	Prefer linear, readable flow over clever constructs.
•	Make the happy path obvious.
•	Keep indentation shallow.
•	Reduce surprise; control flow should do what it looks like it does.
7.2 Guard Clauses and Early Exit
Guidance
•	Validate inputs and invariants at the top of a method.
•	Return or throw early when conditions are not met.
•	Avoid deeply nested conditionals.
Don’t
if (order != null)
{
    if (order.Items != null)
    {
        if (order.Items.Count > 0)
        {
            Process(order);
        }
    }
}
Do
ArgumentNullException.ThrowIfNull(order);

if (order.Items is null || order.Items.Count == 0)
    return;

Process(order);
7.3 Braces
Guidance
•	Always use braces for control blocks, even for single-line bodies.
Don’t
if (isEnabled)
    DoWork();
Do
if (isEnabled)
{
    DoWork();
}
This prevents bugs during edits and makes diffs safer.
7.4 Conditionals
Guidance
•	Prefer positive, readable conditions.
•	Avoid double negatives and clever boolean expressions.
•	Extract complex conditions into well-named variables or methods.
Don’t
if (!(user == null || !user.IsActive))
{
    AllowAccess();
}
Do
if (user is not null && user.IsActive)
{
    AllowAccess();
}
Or, when clarity improves:
var canAccess = user is not null && user.IsActive;
if (canAccess)
{
    AllowAccess();
}
7.5 Switch and Pattern Matching
Guidance
•	Use switch expressions for value mapping.
•	Use switch statements for control flow.
•	Always handle the default case.
Don’t
string label;
if (status == Status.New)
    label = "New";
else if (status == Status.Ready)
    label = "Ready";
else
    label = "Unknown";
Do
var label = status switch
{
    Status.New => "New",
    Status.Ready => "Ready",
    _ => "Unknown",
};
7.6 Loops
Guidance
•	Prefer foreach when index access is not required.
•	Use for only when the index is meaningful.
•	Avoid modifying a collection while iterating it.
Don’t
for (int i = 0; i < items.Count; i++)
{
    Process(items[i]);
}
Do
foreach (var item in items)
{
    Process(item);
}
7.7 LINQ vs Explicit Loops
Tradeoff: LINQ vs Loops
Pros (LINQ)
•	Expressive and concise.
•	Good for query-like transformations.
Cons (LINQ)
•	Can obscure control flow.
•	Encourages hidden side effects when misused.
•	Harder to debug step-by-step.
Pros (Loops)
•	Explicit control flow.
•	Easy to debug.
•	Clear side effects.
Cons (Loops)
•	More verbose.
Decision
•	Use LINQ for simple, declarative transformations.
•	Use loops when side effects exist, logic is complex, or readability improves.
Don’t
items
    .Where(x => x.IsActive)
    .Select(x => { Process(x); return x; })
    .ToList();
Do
foreach (var item in items)
{
    if (!item.IsActive)
        continue;

    Process(item);
}
7.8 var Usage
Tradeoff: var vs Explicit Types
Pros (var)
•	Reduces repetition.
•	Keeps focus on intent.
Cons (var)
•	Can hide important type information.
•	Makes numeric types less obvious.
Decision
•	Use var when the type is obvious from the right-hand side or name.
•	Use explicit types when the type conveys meaning.
Don’t
var x = GetData();
Do
var report = GenerateReport();
Dictionary<string, Worksheet> worksheets = LoadWorksheets();
7.9 Multiple Returns
Guidance
•	Multiple returns are acceptable and often clearer.
•	Prefer early returns over flags or deeply nested logic.
Don’t
var success = false;

if (condition)
{
    success = DoWork();
}

return success;
Do
if (!condition)
    return false;

return DoWork();
7.10 goto
Guidance
•	Avoid goto.
•	Exception: extremely limited, low-level scenarios where it clearly improves clarity (rare).
If you feel tempted to use goto, refactor instead.
7.11 Summary
Control flow should read like a clear set of instructions, not a puzzle.
If a reader has to trace branches to understand intent, simplify.
8. Organization
Code organization should make it easy to find, understand, and change code.
A reader should be able to predict where something lives and what it does before opening the file.
8.1 Principles
•	Organize code by responsibility, not convenience.
•	Keep related things close together.
•	Minimize the amount of code visible at once.
•	Prefer structural clarity over clever layouts.
8.2 Files and Types
Guidance
•	Prefer one primary type per file.
•	File name should match the primary type name.
•	Keep supporting types private or internal where possible.
Don’t
// File: ParserStuff.cs
public class Parser { }
public class ParserHelper { }
public class ParserConfig { }
Do
// File: Parser.cs
public class Parser
{
    private class Config { }
}
8.3 Namespaces
Guidance
•	Use namespaces to reflect logical grouping, not implementation details.
•	Avoid deeply nested namespaces unless they add clarity.
Don’t
namespace Company.Product.Module.Internal.Helpers.Parsing.Experimental;
Do
namespace Company.Product.Parsing;
8.4 Member Ordering Within a Type
Guidance
Keep a consistent order so readers know where to look.
Recommended order:
11.	Constants / static fields
12.	Fields
13.	Constructors
14.	Public properties
15.	Public methods
16.	Protected methods
17.	Internal methods
18.	Private methods
19.	Nested types
Consistency matters more than the exact order.
8.5 Public API First
Guidance
•	Place the most important public members near the top of the type.
•	Push implementation details down.
Don’t
private void ParseInternal() { }

public ParseResult Parse(Input input)
{
    return ParseInternal();
}
Do
public ParseResult Parse(Input input)
{
    return ParseInternal();
}

private ParseResult ParseInternal()
{
}
8.6 Nested Types
Guidance
•	Nest types when they are tightly coupled to the parent.
•	Use nesting to reduce surface area, not to hide poor structure.
Don’t
public class ParserConfig { }
Do
public class Parser
{
    private sealed class Config { }
}
8.7 Partial Types
Tradeoff: Partial Types vs Refactoring
Pros (Partial Types)
•	Useful for generated code.
•	Can separate concerns in rare, large types.
Cons (Partial Types)
•	Harder to reason about behavior.
•	Code is split across files.
Decision
•	Use partial types only when required by tooling or generation is involved.
•	Prefer refactoring into smaller types first.
8.8 Regions
Guidance
•	Avoid #region in production code.
•	Prefer extraction into methods or types.
Don’t
#region Parsing
// 150 lines of logic
#endregion
Do
private ParseResult ParseInternal(Input input)
{
}
8.9 Grouping by Feature vs Layer
Tradeoff: Feature-based vs Layer-based Organization
Pros (Feature-based)
•	Related code lives together.
•	Easier navigation.
Cons (Feature-based)
•	Can duplicate abstractions if not careful.
Pros (Layer-based)
•	Familiar structure.
•	Clear architectural boundaries.
Cons (Layer-based)
•	Features are spread across folders.
Decision
•	Prefer feature-based grouping when it improves cohesion.
•	Use layer-based grouping when architectural boundaries matter more.
8.10 Using Directives
Guidance
•	Keep using directives minimal and relevant.
•	Avoid relying on unrelated transitive imports.
Don’t
using System.Net.Http;
using System.Text; // not used
Do
using System.Net.Http;
8.11 Large Files
Guidance
•	Large files are a smell, not a failure.
•	If a file becomes difficult to scan: extract methods, extract types, and reconsider responsibilities.
Avoid splitting files purely to reduce line count.
8.12 Summary
Good organization makes correct code easy to write and incorrect code harder to introduce.
If readers have to search to understand intent, reorganize.
9. Naming
Names are one of the highest-leverage tools in software. Good names make code self-explanatory; bad names force readers to reverse-engineer intent.
9.1 Principles
•	Optimize for readers, not writers.
•	Prefer clarity over brevity.
•	Use names that reflect intent, not implementation.
•	Follow established .NET naming conventions.
9.2 Casing and Conventions
Namespaces
•	Use PascalCase.
•	Prefer stable, logical groupings.
Do
namespace StyleGuides.CSharp;
namespace StyleGuides.PowerShell;
Types (classes, structs, records, enums, delegates)
•	Use PascalCase.
•	Use nouns or noun phrases.
Don’t
public class Process { }
public class HandleData { }
Do
public class OrderProcessor { }
public class SpreadsheetImporter { }
Interfaces
•	Use IPascalCase.
•	Name by capability or role.
Don’t
public interface IDataManager { }
Do
public interface IOrderStore { }
public interface IClock { }
Methods
•	Use PascalCase.
•	Use verbs or verb phrases.
Don’t
public void Customer() { }
Do
public void CreateCustomer() { }
public bool TryParseRow(string text, out Row row) { }
Properties
•	Use PascalCase.
•	Use nouns or adjective phrases.
•	Avoid Get/Set prefixes.
Don’t
public int GetCount { get; }
Do
public int Count { get; }
public bool IsEnabled { get; }
Events
•	Use PascalCase.
•	Prefer past-tense “something happened” naming.
Do
public event EventHandler Completed;
public event EventHandler ConnectionLost;
Local variables and parameters
•	Use camelCase.
•	Prefer meaningful names; one-letter names only for tiny scopes.
Don’t
var x = GetData();
var tmp = Parse(input);
Do
var report = GenerateReport();
var parsedOrder = Parse(input);
Private fields
Tradeoff: _camelCase vs camelCase vs m_ prefixes
Pros (_camelCase)
•	Highly recognizable as a field.
•	Common in modern C# codebases.
Cons (_camelCase)
•	Slight visual noise.
Decision
•	Use _camelCase for private fields. Do not use Hungarian notation or m_.
Do
private readonly ILogger _logger;
private int _retryCount;
Don’t
private ILogger logger;
private int m_retryCount;
private int iRetryCount;
9.3 Common Suffixes and Patterns
Async
•	Async methods returning Task / Task<T> must end with Async.
Don’t
public Task Load() => LoadInternalAsync();
public void SaveAsync() => SaveInternal(); // not async
Do
public Task LoadAsync() => LoadInternalAsync();
public void Save() => SaveInternal();
Try-pattern
•	TryXxx returns bool and uses out for the value.
•	No exceptions for expected failure.
Do
public bool TryGetCustomer(string id, out Customer customer)
{
    return _store.TryGet(id, out customer);
}
Exceptions
•	Exception types end with Exception.
•	Name them after the condition.
Do
public sealed class InvalidTemplateException : Exception { }
public sealed class DataStoreUnavailableException : Exception { }
Attributes
•	Attribute types end with Attribute. Usage may omit the suffix.
Do
public sealed class CachedAttribute : Attribute { }

// usage:
[Cached]
public void Load() { }
9.4 Acronyms and Abbreviations
Guidance
•	Prefer full words unless the abbreviation is widely understood in your domain.
•	Follow .NET casing for common acronyms.
Do
HttpClient client = new HttpClient();
JsonSerializerOptions options = new JsonSerializerOptions();
Guid correlationId = Guid.NewGuid();
DateTime createdUtc = DateTime.UtcNow;
Don’t
HTTPClient client = new HTTPClient();
DateTime createdUTC = DateTime.UtcNow;
9.5 Avoided Naming Patterns
Avoid generic “junk drawer” names
Avoid: Helper, Util, Manager, Stuff, Thing.
Don’t
public static class ExcelHelper { }
public class DataManager { }
Do
public static class ExcelCellFormatter { }
public class ReportRepository { }
Avoid type-encoding (Hungarian notation)
Don’t
string strName;
int iCount;
bool bIsReady;
Do
string name;
int count;
bool isReady;
9.6 File Naming
Guidance
•	File name matches the primary type in the file.
Do
•	OrderProcessor.cs → OrderProcessor
•	CustomerStore.cs → CustomerStore
9.7 Summary
Good names remove the need for comments and reduce bugs by preventing misunderstanding.
If a name needs explanation, consider renaming it.
10. Tooling / Formatting
Tooling and formatting exist to remove friction, not create it.
Style should be enforced by tools where possible, not by human debate.
10.1 Principles
•	Let tools handle formatting.
•	Optimize for clean diffs.
•	Avoid style discussions in code reviews.
•	Prefer defaults unless there is a strong reason not to.
10.2 Formatting Responsibility
Guidance
•	Formatting is not a personal choice.
•	Use the IDE’s formatter consistently.
•	Do not hand-align code for visual effect.
Don’t
var name     = customer.Name;
var address  = customer.Address;
var zipCode  = customer.ZipCode;
Do
var name = customer.Name;
var address = customer.Address;
var zipCode = customer.ZipCode;
Hand alignment creates noisy diffs and degrades over time.
10.3 Automatic Formatting
Guidance
•	Format code before committing.
•	Use a consistent formatting approach across the codebase.
•	Teams may use either format-on-save or manual formatting; choose one approach per project and keep it consistent.
•	Prefer predictable formatting and avoid formatting-only changes during code review.
10.4 Warnings and Analyzers
Guidance
•	Treat compiler warnings as actionable feedback.
•	Do not suppress warnings casually.
•	Prefer fixing root causes over silencing diagnostics.
Don’t
#pragma warning disable CS8602
value.ToString();
#pragma warning restore CS8602
Do
if (value is not null)
{
    value.ToString();
}
10.5 Nullable Reference Types
Guidance
•	Nullable reference warnings indicate potential bugs.
•	Address them directly rather than suppressing them.
•	Use ! (null-forgiving operator) sparingly and deliberately.
Don’t
return customer!.Name;
Do
ArgumentNullException.ThrowIfNull(customer);
return customer.Name;
10.6 Using Directives
Guidance
•	Keep using directives minimal.
•	Remove unused imports.
•	Avoid relying on transitive imports unintentionally.
Don’t
using System.Net.Http;
using System.Text; // unused
Do
using System.Net.Http;
10.7 Code Cleanup
Guidance
•	Prefer automated cleanup over manual edits.
•	Cleanup should not change behavior.
Examples of acceptable cleanup:
•	Formatting
•	Sorting imports
•	Removing unused variables
•	Applying safe refactorings
10.8 Comments and Formatting
Guidance
•	Formatting should not be used to “comment” code visually.
•	Blank lines are for logical separation, not decoration.
Don’t
DoWork();


DoMoreWork();
Do
DoWork();

DoMoreWork();
10.9 Whitespace and Line Length
Guidance
•	Favor readability over strict column limits.
•	Wrap lines when they become difficult to scan.
•	Avoid excessive horizontal scrolling.
10.10 Tabs vs Spaces
Decision
•	Use spaces, not tabs.
•	Let the IDE manage indentation.
10.11 Summary
Tooling exists to remove noise and prevent argument.
If formatting becomes a discussion, the tools are not being used effectively.


