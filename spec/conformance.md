# Diagnostics, Undefined Behavior, and Conformance

## Program validity

A program is **well-formed** when:

- every source and manifest satisfies lexical and syntactic grammar;
- imports and dependencies resolve without cycles or collisions;
- every declaration, type, expression, statement, generic instantiation,
  compile-time execution, and generated declaration satisfies its semantic
  rules;
- all required extern/native inputs resolve for the selected target; and
- no required implementation resource limit is exceeded.

An ill-formed program must receive at least one diagnostic. A compiler may stop
after the first diagnostic. It may attempt recovery and issue more diagnostics,
but must not produce a runnable artifact represented as a successful conforming
build.

Runtime undefined behavior does not make source statically ill-formed unless the
compiler evaluates the operation at compile time. A compiler may diagnose
provable runtime undefined behavior, but is not required to do so unless another
rule declares the program ill-formed.

## Required diagnostic quality

A diagnostic identifies at least the source file and byte or line/column
location of the primary violation. For cross-declaration errors it also
identifies the conflicting or required declaration. Generic, trait, and
generated-code diagnostics include the instantiation/generation chain specified
by their chapters.

Manifest errors identify the key/path or source pattern. Import and dependency
cycle diagnostics include a concrete cycle. Ambiguous overload diagnostics list
the tied viable candidates.

Diagnostic wording and formatting are implementation-defined.

## Warnings

Warnings never change whether a program is well-formed. An implementation may
warn about unused values, unused declarations, implicit switch fallthrough,
non-exhaustive switches, suspicious shadowing, likely dangling references, or
other conditions.

Repeated identical imports and overlapping imports that reach the same module
or declaration anywhere in one module are specifically idempotent and must not
produce a duplicate-import warning. A build option may promote warnings to tool
failure, but that is not a language compile error.

`compile_warning` deliberately requests a warning during compile-time
execution. Suppressing its display does not suppress generator execution.

## Undefined behavior

When an execution reaches undefined behavior, this specification imposes no
requirements on that execution. The compiler may assume a well-defined
execution never reaches such an operation. Undefined behavior in one thread
invalidates the program execution as a whole.

Version-1 undefined behavior consists of the following exhaustive language
categories. Libraries and extern contracts may document additional preconditions
for their own operations.

### Arithmetic and conversion

- Signed integer unary negation, increment/decrement, addition, subtraction, or
  multiplication whose mathematical result is not representable in the
  operation's declared type.
- Signed integer division or remainder by zero.
- Signed minimum divided or remaindered by `-1` when the quotient is not
  representable.
- A shift count less than zero or at least the left operand's bit width.
- Signed left shift of a negative value or to a mathematical result not
  representable by the signed type.
- Floating-to-integer or floating-to-`byte` conversion of NaN, infinity, or a
  truncated value outside the destination mathematical range.

Floating division by zero, floating overflow/underflow, NaN operations, integer
narrowing, and `byte` left-shift truncation are defined and are not in this list.

### References, pointers, and lifetime

- Reading, writing, calling, taking a derived address, or otherwise accessing
  through a dangling reference.
- Dereferencing or accessing through a null, dangling, incorrectly typed,
  insufficiently aligned, invalidated, or otherwise invalid raw pointer.
- Dereferencing `*void`.
- Accessing storage before a value lifetime begins or after it ends.
- Reusing an old reference/pointer after a new value happens to occupy the same
  address; version 1 has no lifetime-laundering operation.
- Using an array element reference, raw pointer, or iterator after an
  invalidating operation.

Merely copying, storing, destroying, or equality-comparing a dangling raw
pointer without dereferencing it is not undefined, provided the comparison types
are compatible. A dangling reference has no such general safe value operation;
copying its binding without accessing the target is permitted, but any
transparent value operation is not.

### Arrays and iterators

- Array indexing with an index outside `[0, length)`.
- `pop` on an empty array.
- `remove` outside `[0, length)` or `insert` outside `[0, length]`.
- A negative array size/capacity or capacity arithmetic overflow.
- Calling an iterator's `next()` after `done()` has returned true for its current
  state.
- Using an iterator after its source invalidated it.

Recursive structural equality that does not terminate is nontermination, not
undefined behavior.

### Concurrency

- A data race as defined in [Concurrency](concurrency.md#data-races).
- Unsynchronized use of invalidated state even when atomic ownership accounting
  itself remains correct.

Atomic ownership counts prevent ownership-control corruption but do not remove
these categories.

### C interoperability

- Importing a C-layout enum representation that matches no declared enumerator.
- Calling an external symbol whose actual calling convention, parameter types,
  result type, or layout is incompatible with its Dot extern declaration.
- An external function violating the validity/lifetime requirements of raw
  pointers or C-layout values exchanged with Dot.
- Accessing a typed pointer recovered from `*void` when it does not in fact point
  to a suitably aligned live value of that type.

An exception escaping a C boundary is not undefined; the generated boundary
terminates. `panic` and uncaught exception behavior are likewise defined fatal
behavior.

## Defined leaks and nontermination

The following are undesirable but defined:

- an owning cycle of object and/or array handles keeps its allocations alive
  indefinitely;
- deliberately retaining owning handles consumes memory;
- a user recursive function, constructor, trait-free runtime algorithm, or
  equality operator may not terminate; and
- an infinite loop may run indefinitely.

Runtime stack exhaustion caused by unbounded recursion is a resource failure.
The implementation may terminate with an implementation-defined fatal report;
it is not required to convert it to a Dot exception.

## Fatal but defined behavior

These operations terminate rather than produce undefined behavior:

- `panic` terminates without unwinding;
- an uncaught exception unwinds Dot scopes, reports, then terminates;
- an exception escaping a destructor terminates immediately;
- an exception escaping an exported C boundary terminates immediately; and
- irrecoverable runtime bookkeeping failure may terminate after reporting when
  even the required emergency `allocation_error` representation cannot be used.

Fatal statuses are nonzero. Exact numeric values and report formatting are
implementation-defined.

## Implementation-defined behavior

Every conforming implementation documents:

- supported target OS, architecture, ABI, C compiler, and C++ backend versions;
- ordinary Dot type layout, name mangling, and calling convention where tools
  expose them, while making clear they are not stable ABI;
- `#repr("C")` size, alignment, offsets, enum mapping, and external symbol
  character support for each target;
- NaN payload/sign propagation where IEEE-754 permits alternatives;
- compile-time and frontend resource limits at or above the required minima;
- runtime stack-overflow handling;
- fatal process statuses, stack-trace availability, and diagnostic formatting;
- canonical target strings and build-profile effects;
- local-registry and build-cache locations;
- lockfile content-hash byte canonicalization;
- raw linker-option interpretation; and
- maximum raw-string hash delimiters when above the minimum.

Implementation-defined behavior is not permission to alter source evaluation
order, ownership, destruction, arithmetic widths, const rules, exception
matching, or any other defined semantic rule.

## Unspecified but constrained choices

The implementation need not document:

- object/array allocation addresses;
- string inline-storage threshold and representation;
- array growth factor and excess capacity;
- internal atomic reference-count values;
- exact compiler-generated C++ source spelling; or
- machine-code sharing between generic instantiations.

These choices must remain unobservable except through behavior explicitly
allowed by the specification. For example, array excess capacity is observable
through `capacity()`, but no particular excess value is promised; aliases must
still agree on it.

## Compile-time failures

Any operation during compile-time evaluation that would be runtime undefined
behavior must instead receive a compile error. Exceeding a compile-time resource
limit is also a compile error and includes the active compile-time call stack.

`compile_error` is a required diagnostic. `compile_warning` is a warning and
does not invalidate the program unless external build policy promotes warnings.

## Minimum implementation limits

A conforming implementation must support at least:

| Resource | Minimum |
| --- | ---: |
| UTF-8 source file size | 1 MiB |
| identifier length | 255 bytes |
| lexical/block/type nesting depth | 256 |
| parameters in one callable | 255 |
| fields in one nominal type | 65,535 |
| enum values in one enum | 65,535 |
| tuple arity | 255 |
| generic parameters/arguments | 64 |
| overloads under one name | 1,024 |
| compile-time call depth | 256 |
| compile-time executed operations per target | 1,000,000 |
| compile-time live managed memory | 64 MiB |
| compile-time live syntax nodes | 1,000,000 |
| generated declarations per module | 100,000 |
| compile-time warnings per target | 1,024 |
| raw-string delimiter hashes | 255 |

An implementation may support more. A program exceeding a documented limit may
be rejected with a resource diagnostic and is not portable to all version-1
implementations.

Runtime arrays use `i64` lengths but remain limited by address space and
allocation. Failure to allocate managed storage throws `allocation_error` rather
than being a compile-time conformance limit.

## Compiler conformance

A conforming Dot compiler/build tool must:

1. accept and correctly execute every well-formed program within minimum
   resource limits, subject only to its documented target support;
2. diagnose every ill-formed program it processes;
3. implement all observable semantics independently of backend accidents;
4. preserve compile-time determinism for fixed explicit inputs;
5. generate C boundaries and layouts matching the selected target ABI; and
6. make no stable ordinary Dot ABI promise beyond this specification.

A compiler may provide extensions only behind an explicit non-version-1 mode.
An extension must not silently change a program compiled with `language: 1`.

## Generated C++ conformance

The initial backend targets C++20 and Clang, but users are not required to
inspect or compile generated C++ manually. If exposed, generated source must be
compiled with options and support runtime that preserve:

- fixed-width arithmetic and Dot overflow points;
- IEEE floating behavior without fast-math relaxations;
- left-to-right evaluation;
- non-null objects, structural values, and replacement assignment;
- atomic ownership accounting and memory ordering;
- destructor and exception rules; and
- target C ABI boundaries.

C++ undefined or implementation-defined behavior is not automatically inherited
by Dot. It is permissible only where this specification independently declares
the corresponding Dot behavior undefined or implementation-defined.
