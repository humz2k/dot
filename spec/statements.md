# Statements and Control Flow

## Blocks

A block is a brace-delimited sequence of statements:

```dot
{
    statement;
    // ...
}
```

Every block creates a lexical scope. Statements execute in source order until
one transfers control or throws. On every exit, initialized locals are destroyed
in reverse declaration order before control reaches the destination.

An empty statement consisting only of `;` is valid and does nothing.

## Expression statements

An expression followed by `;` evaluates the expression and discards any
non-reference result after the usual end-of-statement temporary lifetime.
Typical expression statements are calls, assignment, increment/decrement, and
constructor calls made only for their effects.

A bare name or literal expression statement is valid but an implementation may
warn that its value is unused. Warnings do not change validity.

## Local declarations

```dot
value : i32 = expression;
value := expression;
```

The initializer is evaluated before the binding begins its lifetime. A failed
initializer leaves no local to destroy. Declaration syntax and typing are
defined in [Declarations](declarations.md#local-variables).

## Destructuring declarations

Tuple declaration destructuring uses `:=`:

```dot
(left, right) := pair;
(name : str, count : const i32) := record;
((x, y), _) := nested;
```

The right expression is evaluated exactly once. Its type must be a tuple with
the same recursive shape as the pattern. Pattern rules are:

- `name` declares one inferred local;
- `name : Type` declares a local with the written type;
- `_` discards the corresponding element; and
- `(patterns...)` recursively matches a tuple.

Bindings initialize left-to-right using ordinary copy/direct-initialization
rules. An annotation must exactly match the corresponding element after at most
adding const. Inference retains the element's complete type, including a stored
reference type. Destructuring never implicitly takes a reference to a tuple
element.

All binding names in one pattern must be distinct and otherwise satisfy ordinary
same-scope redeclaration rules. They enter scope together after all bindings are
initialized; their destruction order is the reverse of left-to-right binding
initialization.

The tuple source remains alive through all binding initializations. If it is a
temporary, it is destroyed at the end of the declaration statement. A pattern
must declare at least one name. Tuple destructuring assignment to existing
variables is not supported.

## Conditional statements

```dot
if (condition) {
    // ...
} elif (other_condition) {
    // ...
} else {
    // ...
}
```

Parentheses around every condition and braces around every branch are required.
Each condition must have exactly type `bool` and is evaluated only if all
preceding conditions were false. At most one branch executes. `else if` is not
accepted; the token is `elif`.

Each branch block has its own scope. There is no conditional expression.

## Loops

### While loop

```dot
while (condition) {
    // ...
}
```

The condition has type `bool` and is evaluated before each iteration. A false
condition exits. `continue` destroys current body locals and reevaluates the
condition. `break` destroys body locals and exits. Version 1 has no `do while`.

### C-style for loop

```dot
for (initializer; condition; increment) {
    // ...
}
```

Each clause may be omitted. The initializer is either one local declaration or
one expression. It executes once. An omitted condition means `true`; otherwise
the condition must have type `bool` and is tested before each iteration. The
increment is one expression evaluated after each normally completed iteration
and after `continue`.

A local declared in the initializer is visible in the condition, increment, and
body. Its scope ends after the loop, and it is destroyed then. Body declarations
are recreated and destroyed each iteration.

### Iterator loop

```dot
for (element in collection) {
    // ...
}

for (element : ElementType in collection) {
    // ...
}
```

The loop is defined by this conceptual lowering, where hidden names cannot
collide with source names:

```dot
{
    __source := collection;
    __iterator := __source.forward_iterator();

    while (!__iterator.done()) {
        element := __iterator.next();
        // original body
    }
}
```

The actual compiler need not create observable source declarations, but must
preserve these semantics:

1. Evaluate `collection` exactly once and keep its value alive through the loop.
2. Resolve and call `forward_iterator()` by ordinary method rules. It may take
   mutable or const `self` according to the source value.
3. The result type must provide `done(self : const &Self) -> bool` and
   `next(self : &Self) -> Element`.
4. Before every iteration call `done()`. Call `next()` exactly once only when it
   returned false.
5. Initialize a fresh iteration binding from the `next()` result. An explicit
   type must exactly match after const addition; otherwise it is inferred.
6. Destroy the iteration binding and body locals at the end of each iteration.

Calling `next()` when `done()` is true is undefined behavior for a conforming
iterator. The built-in array iterator returns elements by value. A custom
iterator may deliberately return `&T`, in which case inference declares a
reference binding; the iterator remains responsible for its validity.

`continue` proceeds to the next `done()` call. `break` exits and destroys the
hidden iterator and source. `return` and exceptions likewise destroy them.
Mutating a source in a way that invalidates its active iterator makes subsequent
iterator use undefined behavior. Arrays treat every length- or capacity-changing
operation as such an invalidation.

### Loop control

`break;` exits the nearest lexically containing loop or `switch`. `continue;`
advances the nearest lexically containing loop and ignores intervening
switches. Use outside the respective construct is a compile error. Version 1
has no labels or value-producing `break`.

## Switch statement

```dot
switch (color) {
case Color::red:
    handle_red();
    break;
case Color::green:
case Color::blue:
    handle_cool_color();
    break;
default:
    handle_unknown();
}
```

The controlling expression is evaluated once and must have enum type. Every
`case` expression is a compile-time value of exactly that enum type. Two cases
with equal underlying values are a compile error even if they name different
enumerators. There may be at most one `default`, which may appear anywhere.

Execution begins at the first matching case, or at `default` when no case
matches, or after the switch when neither exists. It then proceeds through later
labels and statements until `break`, `return`, an exception, or the end. Cases
therefore fall through by default.

The switch body is one lexical scope, but each labeled statement region also
forms an implicit nested scope ending immediately before the next label. Locals
declared after one label are destroyed before control falls through to the next
and are not visible there. Consecutive labels with no statements are permitted.
This rule prevents control from jumping into the lifetime of a local.

A switch need not be exhaustive. An implementation may warn about omitted
enumerators or implicit fallthrough, but these warnings do not alter semantics.

## Return statement

```dot
return;
return expression;
```

`return;` is valid in `void` functions and destructors. `return expression;` is
required in non-void functions and constructors and invalid in a void function
or destructor.

The result expression is evaluated and copied/directly initialized into result
storage before parameters and locals are destroyed. Its type must exactly match
the declared return type after const addition. A returned reference expression
must already have the declared reference type; return does not implicitly take
an address or extend lifetime.

Returning from `main` sets the process status as described below.

Definite-return analysis is structural and conservative. `return`, `throw`, and
an unconditional expression statement whose call is the intrinsic `panic` do
not fall through. A block falls through exactly when its final reachable
statement can fall through. An `if` does not fall through only when it has an
`else` and every branch does not fall through. A loop does not fall through only
when its condition is the literal `true` (or the C-style condition is omitted)
and no reachable `break` exits that loop. A `switch` does not fall through only
when its labels cover every distinct declared enum value or contain `default`,
and no possible label path reaches the switch end or a `break` targeting it. A
`try` does not fall through only when its try block and every catch block do not
fall through. No other value-range or interprocedural proof is used.

Unreachable statements are still parsed, name-resolved, and type-checked. Their
runtime execution is, of course, prevented by the preceding control transfer.

## Exceptions

### Throwing

```dot
throw Error@new(message);
throw;
```

The first form requires an object value. The expression is evaluated, and an
owning handle is copied/directly initialized into exception storage before stack
unwinding begins. Exceptions are unchecked; functions do not declare a throws
set.

`throw;` is valid only dynamically and lexically inside an executing `catch`
block. It rethrows the same exception identity after destroying locals created
in that catch.

### Catching

```dot
try {
    operation();
} catch (error : const &ParseError) {
    report(error);
} catch (error : const &IoError) {
    recover(error);
} catch {
    report_unknown();
}
```

A `try` must have at least one catch. Typed catches are tested in source order by
exact nominal object type; there is no subtype matching. Their binding type must
be exactly `const &ObjectType`. The runtime retains the exception's owning
handle until its handler ends, so the binding remains valid.

A catch-all has no binding, matches every object type, and must be last. More
than one catch for the same exact object type is a compile error because later
ones are unreachable.

Before entering a matching handler, unwinding destroys every exited local and
partially initialized value in reverse order. A handler completes normally by
falling through, after which execution continues after the whole try statement.
An exception thrown by a handler begins a new search outside the current try.

Version 1 has no `finally`. Destructors provide scoped cleanup.

### Uncaught exceptions and destructors

An uncaught exception unwinds all active Dot scopes, reports its fully qualified
dynamic object type and an available stack trace, and terminates with a nonzero
implementation-defined process status. Standard exception objects also expose
`message()`, but arbitrary thrown objects need not.

An exception attempting to escape a destructor causes immediate termination.
An exception attempting to cross an exported `extern "C"` boundary is caught by
the generated boundary and causes immediate termination.

## Panic

```dot
panic("internal invariant failed");
```

`panic` is a non-overloadable runtime intrinsic with one `str` parameter and
return type `void`. It reports the message, source location, and an available
stack trace, then immediately terminates with a nonzero implementation-defined
status. It performs no stack unwinding and cannot be caught. Code after a panic
call remains type-checked; version 1 has no `never` type. For definite-return
analysis, a path ending in an unconditional panic call cannot fall through and
therefore satisfies a non-void callable's return requirement despite the call's
`void` expression type.

## Program entry

An executable must define exactly one public, non-generic function named
`::main` with one of these signatures:

```dot
fn main() -> i32
fn main(arguments : str[]) -> i32
```

It must use ordinary Dot linkage and cannot be overloaded, constrained, nested,
or declared in a library module. Its returned `i32` becomes the process exit
status using the host platform's ordinary mapping from a signed 32-bit value.

For the argument form, element zero is the executable path as supplied by the
host, followed by user arguments. POSIX argument bytes are preserved. A host
using a Unicode argument API is converted to UTF-8. Host APIs cannot supply
embedded zero bytes.

An exception escaping `main` follows uncaught-exception behavior. A normal
return destroys all locals and runtime-owned argument values before process
exit.
