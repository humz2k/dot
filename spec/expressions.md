# Expressions and Operators

## Evaluation model

Except where short-circuiting or control flow prevents evaluation, every
subexpression is evaluated exactly once and from left to right. This applies to:

- operands of every unary and binary operator;
- the receiver before method-call arguments;
- call arguments and omitted-argument defaults in parameter order;
- array and tuple literal elements;
- constructor arguments;
- field initializers in field declaration order;
- assignment destination before source; and
- operands of `same_identity`.

`&&` and `||` are the only short-circuit binary operators. An implementation
must preserve this order even when generated C++ would otherwise leave it
unspecified.

An expression evaluates to a value, a reference value, a raw pointer, a
function value, or `void`. An **addressable place** denotes existing storage and
is listed in [Values and Lifetime](values-lifetime.md#references).

## Precedence and associativity

From highest to lowest:

| Level | Forms | Associativity |
| --- | --- | --- |
| Primary | literals, names, parenthesized/tuple/array values, lambda, quote | — |
| Postfix | call `()`, construction `@name()`, member `.`, index `[]`, postfix `++ --` | left |
| Prefix | `&`, `raw &`, `*`, `!`, `~`, unary `-`, prefix `++ --` | right |
| Multiplicative | `* / %` | left |
| Additive | `+ -` | left |
| Shift | `<< >>` | left |
| Relational | `< <= > >=` | left; chaining rejected by types |
| Equality | `== !=` | left; chaining rejected by types |
| Bitwise AND | `&` | left |
| Bitwise XOR | `^` | left |
| Bitwise OR | `|` | left |
| Logical AND | `&&` | left, short-circuit |
| Logical OR | `||` | left, short-circuit |
| Assignment | `= += -= *= /= %=` | right syntactically |

Although assignment parses right-associatively, it has type `void`, so `a = b =
c` is ill-formed. Parentheses may override precedence but do not change
left-to-right evaluation.

## Names and literals

A name expression is resolved according to
[Program Structure](program-structure.md). A function overload set is not a
runtime value until an expected function type or explicit generic arguments
select exactly one overload.

Literal types and contextual typing are defined in
[Lexical Structure](lexical-structure.md). Contextual typing applies from:

- an explicitly declared variable, field, parameter, or return type;
- the corresponding element type of an array or tuple literal;
- an operator's already selected built-in operand type; or
- a selected callable parameter.

Contextual typing does not search user conversions. During overload resolution,
an unsuffixed literal receives a parameter's expected type only when every
otherwise viable candidate has the same parameter type at that argument
position. When viable candidates disagree, the literal is typed as though it
had no expected type (`i32`, then `i64`, then `i128` for an integer; `f64` for a
float), and candidates must exactly match that chosen type. Thus overloads
`f(i32)` and `f(i64)` select `f(i32)` for `f(1)`, while a sole `f(i64)`
contextually types `1` as `i64`. Likewise, `i8@from(1000)` selects the built-in
`from(i32)` overload and then applies the specified narrowing conversion.

## Parenthesized and tuple expressions

`(expression)` is grouping. `(expression,)` is a one-element tuple. A tuple with
two or more elements uses commas, and `()` is the empty tuple.

Without an expected tuple type, each element is independently inferred. With an
expected tuple type of equal arity, each literal element is contextually typed
by the corresponding element type and must then match exactly after const
addition. Tuple elements evaluate left-to-right.

`tuple.N` accesses zero-based element `N`. `N` must be a decimal integer token
with no suffix and less than the tuple arity. The result is an addressable place
when the tuple expression is an addressable place. Const access propagates to
the direct element, except through a stored reference or raw-pointer pointee as
specified by the const rules.

## Array literals

```dot
[first, second, third]
[]
```

All non-empty elements must have exactly one type after contextual literal
typing; the compiler does not compute a common converted type. The literal
creates a fresh non-null array identity, evaluates and stores elements
left-to-right, and has type `T[]`.

An empty literal requires an expected array type. It creates a fresh empty array
of that type. It cannot by itself infer a local type.

If an element evaluation throws, already stored elements are destroyed in
reverse order and the fresh array is released.

## Member access

`receiver.member` evaluates the receiver once, performs reference transparency,
and resolves `member` in the static receiver type. Field access produces an
addressable place when the receiver is addressable. A method name produces a
method overload set and must immediately participate in call syntax; bound
method values do not exist.

A raw pointer has no implicit member access. `pointer.member` is invalid;
`(*pointer).member` is required. Object values and references use `.` rather
than `->`.

## Calls

```dot
function(arguments)
function<GenericArguments>(arguments)
receiver.method(arguments)
function_value(arguments)
```

Overload resolution selects the callable from statically known types before
runtime evaluation and does not execute user code. At runtime the callable or
receiver is evaluated first. Then, in parameter order, each explicit argument
is evaluated and its parameter is immediately initialized. Missing defaults
are subsequently evaluated and initialize their parameters in order. If any
argument/default evaluation or parameter initialization throws, already
initialized by-value parameters are destroyed in reverse parameter order and
the body is not entered.

A by-value argument copies the argument value unless a fresh constructor result
directly initializes the parameter. Passing a non-reference place to a
reference parameter requires explicit `&place`. A reference-valued expression
may be passed directly to an equivalent reference parameter. A non-owning
object or array reference/dereference cannot satisfy a by-value owning-handle
parameter.

The result of a `void` call is `void`. A non-void result is a fresh result value
or reference with the declared return type. A returned reference has no lifetime
extension.

A call through a function value must supply exactly its function type's arity.
Default arguments are available only when a call names a function, method, or
constructor declaration directly; converting that callable to a function value
does not carry its defaults.

## Constructor expressions

```dot
Type@constructor(arguments)
Type@constructor<GenericArguments>(arguments)
```

The type and constructor name are resolved statically. Arguments evaluate
left-to-right. The expression produces a fresh value under the construction
rules in [Declarations](declarations.md#constructors). Primitive built-in `from`
constructors use the conversion rules in [Type System](type-system.md#explicit-conversions).

`Self { ... }` is a constructor-only primary expression whose semantics are
defined in the same chapter. It is invalid in ordinary functions and methods.

## Indexing

`container[index]` evaluates the container then the index. Built-in arrays
require an `i64` index and produce an addressable element place. Built-in strings
require `i64` and produce a non-addressable `byte` value. Bounds are unchecked;
an index less than zero or at least the current length causes undefined behavior.

A user type may provide `operator []`. Normal method overload resolution selects
it after evaluating receiver and index. Its declared result controls whether the
expression is a value or reference.

## Address-taking and dereference

`&place` creates a non-null reference. `raw &place` creates a raw pointer.
The operand must be an addressable existing place and is evaluated once. For an
object value, both refer to the object allocation rather than the owning-handle
variable; for an array, both refer to the logical array rather than its owning
handle binding. Applying `&` to an existing reference preserves its target and
type. A const access path produces `const &T` or a raw pointer whose pointee is
`const T`; address-taking cannot remove const. `raw &` applied to a reference
also addresses the reference's target because a reference has no separately
addressable binding storage.

`*pointer` explicitly dereferences a raw pointer and produces an addressable
place of the pointee type. Null, dangling, misaligned, `*void`, or incorrectly
typed dereference is undefined behavior.

Prefix `&` is disambiguated from infix bitwise AND by grammar position.

## Arithmetic operators

### Integer arithmetic

For signed integer type `I`, unary `-`, binary `+`, `-`, `*`, `/`, and `%`
operate at `I`'s declared width and require same-type operands. A mathematical
result of unary negation, addition, subtraction, or multiplication outside `I`'s
range causes undefined behavior.

Division truncates toward zero. Division or remainder by zero is undefined
behavior. `minimum_I / -1` and `minimum_I % -1` are undefined because the
quotient is not representable. For valid operands:

```text
(a / b) * b + (a % b) == a
```

and the remainder has the sign of `a` or is zero.

A negative literal is unary negation applied to a positive literal. When the
positive token's magnitude is exactly one greater than the positive maximum and
the expected type is the corresponding signed type, the combined unary-minus
expression denotes that type's minimum value and is valid. The positive literal
alone remains invalid for that type.

### Floating arithmetic

`f32` and `f64` provide unary `-` and binary `+`, `-`, `*`, and `/` for
same-type operands. They follow IEEE-754, including:

- finite nonzero divided by signed zero produces appropriately signed infinity;
- zero divided by zero produces NaN;
- overflow produces signed infinity;
- invalid operations produce NaN; and
- each operation rounds once to its declared format using ties-to-even.

Floating `%` is not a built-in operator. NaN payload choice is
implementation-defined. No floating operation has undefined behavior merely
because an operand or result is NaN or infinite.

### String concatenation

`str + str` returns a new immutable string containing the left bytes followed by
the right bytes. Operands evaluate left-to-right. Allocation failure throws
`allocation_error` and leaves both operands unchanged.

### User arithmetic operators

When no applicable built-in operation exists, member operator lookup on the
left operand may select a declared operator. No implicit conversion is attempted.

## Bitwise and shift operators

Signed integer types provide `~`, `&`, `|`, `^`, `<<`, and `>>`. Binary bitwise
operands must have the same integer type. Operations use the fixed-width
two's-complement bit representation and return that type.

`byte` provides the same operations. Its binary bitwise operands are `byte`.

For both signed integers and `byte`, a shift count has exactly type `i64` and
must be in `[0, bit_width)`, otherwise behavior is undefined. `byte << count`
discards high bits and returns the low eight bits; `byte >> count` shifts in
zeros. Signed right shift is arithmetic and shifts in the sign bit. Signed left
shift is undefined when the left operand is negative or the mathematical result
is not representable by the signed type.

Bitwise and shift operators are not subject to numeric promotions. User types
cannot overload `~`, `&`, `|`, `^`, `<<`, or `>>` in version 1.

## Comparison and equality

Primitive integer and `byte` equality and ordering compare mathematical values.
`bool` provides only `==` and `!=`, with `false` distinct from `true`.

Floating comparisons follow IEEE-754. If either operand is NaN, `==`, `<`,
`<=`, `>`, and `>=` are false and `!=` is true. Positive and negative zero
compare equal.

Strings compare byte sequences. Ordering is lexicographic by unsigned byte,
with a proper prefix less than the longer string. Enums provide equality but no
built-in ordering. Arrays, tuples, and optionals use their structural equality
rules in [Built-in Runtime Types](builtins.md). Structs and objects have no
automatic equality; a matching declared `operator ==` is required.

References use the referenced type's value comparison. They do not compare
addresses. Raw-pointer `==` and `!=` compare addresses after permitted const
addition. Two non-null pointer operands are compatible only when their complete
pointer types become identical after adding, but never removing, pointee const
qualification. A typed pointer and `*void` therefore require an explicit
`@from` conversion before comparison. Either pointer type may compare with
`null`.

A user-defined `operator ==` supplies `!=` as its negation. Relational operators
do not derive one another.

Chained comparisons such as `a < b < c` parse left-associatively and are almost
always ill-formed because the first result is `bool`; Dot does not give them
special mathematical meaning.

## Logical operators

`!`, `&&`, and `||` require `bool` operands and return `bool`. `!` evaluates its
operand. `left && right` evaluates `right` only when `left` is true. `left ||
right` evaluates `right` only when `left` is false.

There is no truthiness conversion. Integers, pointers, objects, arrays, strings,
and optionals cannot be used as conditions without an explicit Boolean
operation.

User types may overload unary `!`; `&&` and `||` cannot be overloaded.

## Increment and decrement

Prefix and postfix `++` and `--` are built in for signed integers and floating
types. They are equivalent to adding or subtracting an exactly typed one, with
the same overflow and IEEE behavior. The operand must be a mutable addressable
place and is evaluated once.

- Prefix updates the place and returns the new value by value.
- Postfix snapshots the old value, updates the place, and returns the snapshot.

These operators cannot be overloaded. They are invalid for `byte`.

## Assignment expressions

The left operand of `=` must be a mutable assignable place. The right operand
must exactly match its target type after permitted const addition. Replacement
semantics are defined in [Values and Lifetime](values-lifetime.md#replacement-assignment).
Assignment returns `void`.

`+=`, `-=`, `*=`, `/=`, and `%=` evaluate the left place once, evaluate the
right operand, apply the corresponding built-in or declared binary operator,
then replacement-assign the result. They return `void` and cannot themselves be
overloaded. `%=` is unavailable when `%` is unavailable.

## Optional and identity operations

`none` is not equal to an ordinary `T`; it exists only as an optional value.
Optional member access must go through `has_value()` and `value()`; Dot has no
optional chaining or null-coalescing operator.

`same_identity` is the non-overloadable intrinsic defined in
[Values and Lifetime](values-lifetime.md#identity). Object `==` never falls back
to identity, and array `==` never falls back to identity.

## Compile-time-only expressions

`reflect`, `quote`, splice expressions, attribute values, trait invocations, and
syntax metadata values are described in [Metaprogramming](metaprogramming.md).
They are invalid in runtime evaluation contexts unless generated runtime code
contains only their resulting ordinary syntax.
