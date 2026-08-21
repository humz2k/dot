# Declarations

## Declaration prefixes

A declaration may be preceded by documentation comments and attribute uses.
Both attach to exactly the next declaration. Attribute semantics are defined in
[Metaprogramming](metaprogramming.md#attribute-declarations).

Visibility has no keyword. It is derived from leading underscores as specified
in [Program Structure](program-structure.md#visibility).

## Variables and constants

### Local variables

An explicitly typed local declaration is:

```dot
name : Type = expression;
```

An inferred declaration is:

```dot
name := expression;
```

Every local requires an initializer. The initializer is evaluated before the
new name enters scope. Its type must exactly match the declared type after at
most adding const. An inferred local receives the expression's complete type,
including const and reference qualification.

One ordinary declaration introduces one local. Comma-separated variable
declarations are invalid. Tuple declaration patterns are described in
[Statements](statements.md#destructuring-declarations).

Locals are mutable unless their type is const-qualified. A const local must be
initialized but cannot later be assigned through that access path.

### Namespace constants

Namespace scope permits only const-qualified value declarations:

```dot
maximum : const i32 = 100;
```

The initializer must be a compile-time expression and must be evaluated during
compilation. Namespace values have static storage for the program's duration,
but because their value is completely compile-time-defined, no runtime
initialization or destruction is performed. If the type has a runtime
destructor, the declaration is ill-formed. Namespace object, array, optional
owning, and heap-backed string values are therefore invalid unless the compiler
can represent the value as immutable static data with no runtime ownership
action; version 1 guarantees this only for primitives, enums, tuples of allowed
constants, and `str` literals.

Runtime mutable globals and function-local static variables are not supported.

## Type aliases

```dot
alias Index = i64;
alias Buffer<T> = T[];
```

An alias may occur at namespace scope or as a type member. It may have generic
parameters and constraints. Its target must be a valid type after substituting
the alias parameters. Alias transparency and cycle rules are defined in
[Type System](type-system.md#type-aliases).

## Struct declarations

```dot
struct Point {
    x : f64;
    y : f64;

    constructor new(x : f64, y : f64) -> Self {
        return Self { x = x, y = y };
    }
}
```

A struct declaration introduces a nominal value type. Its body may contain:

- fields;
- constructors and one destructor;
- methods and overloadable operators;
- nested structs, objects, enums, aliases, traits, and attribute declarations;
  and
- `comptime` generation blocks.

It may not contain a static method. A function without `self` belongs at
namespace scope. In particular, `comptime fn` declarations are namespace-scope
only; a type may run compile-time member generation through a `comptime` block.

The lexical order of fields is their declaration order and controls
initialization, structural copying, layout where layout is observable, and
destruction. Non-field member order has no runtime effect.

## Object declarations

```dot
object Node {
    value : i32;
    next : Self?;

    constructor new(value : i32) -> Self {
        return Self { value = value, next = none };
    }
}
```

An object declaration has the same permitted members as a struct and introduces
a nominal non-null owning-handle type. The field declaration order controls the
allocation's initialization and destruction.

Constructing an object creates a fresh allocation identity. No source operation
implicitly clones that allocation. An owning cycle is valid and leaks until the
cycle is broken.

## Fields

```dot
name : Type;
```

A field must have a complete storable type. Version 1 has no field initializer,
bit-field, unnamed field, flexible member, or explicit alignment declaration.
Every constructor must initialize every field.

A field whose name begins with `_` is private to its containing type. Other
fields are public unless the containing type or namespace is private. A public
field's type must be publicly nameable wherever the field is public.

A const-qualified field is initialized by construction and can change only when
the complete containing struct is replaced; object allocation fields are never
replaced as a whole. A reference field is non-reseatable during the containing
value's lifetime, but replacement of a containing struct ends the old field's
lifetime and creates a newly bound reference field.

## Constructors

### Declaration and invocation

```dot
constructor name<T>(parameters) -> Self [constraints] {
    // body
}

value := Type@name(arguments);
value := Type@name<GenericArguments>(arguments);
```

`constructor` is the required keyword. Every constructor has a non-empty
identifier name and explicitly returns `Self`. `default` has no special status.
A type has no implicit constructor.

Constructors may be overloaded and generic. They have no `self` parameter and
cannot be called using `.`. Constructor overload resolution follows ordinary
call rules after the containing type is fixed.

### Complete struct and object initialization

A struct or object constructor produces its result by returning either:

```dot
return Self {
    field_a = expression_a,
    field_b = expression_b,
};
```

or the result of another constructor of the same complete type:

```dot
return Self@other(arguments);
```

`Self { ... }` is valid only within a constructor of that `Self`. Every declared
field must appear exactly once; an unknown or repeated field is a compile error.
The initializer-list textual order is irrelevant. Field expressions are
evaluated and fields initialized in declaration order.

No constructed `self` exists while a constructor body or `Self` initializer is
executing. A constructor therefore cannot access `self`, take `&Self`, or expose
the value under construction. Constructor arguments and locals are ordinary
independent values.

For an object, storage and its ownership control block are allocated when the
`Self` initializer begins, before its first field expression. Allocation failure
throws `allocation_error`. If a field expression throws, already initialized
fields are destroyed in reverse declaration order and the allocation is
released; no owning handle escapes. Struct construction applies the same
partial-field cleanup without heap allocation.

Every reachable constructor path must return a complete result, throw, or end
in unconditional `panic`. Falling off the end is a compile error. Calling
another constructor is an ordinary call; recursive constructor calls are
permitted and may fail through ordinary runtime resource exhaustion.

Enum constructors instead return an ordinary expression of type `Self`, subject
to the enum-origin rules below. Compiler-provided primitive and constructed-type
constructors follow their built-in semantics and do not use `Self { ... }`.

Fresh constructor results directly initialize a variable, parameter, field,
literal element, or function result when used as that complete initializer.
This direct initialization does not create or move from a second temporary. It
is not a move operation and cannot invoke user code. Where a callable is allowed
to return an existing value—an ordinary function or enum constructor—doing so
performs the ordinary copy specified in
[Values and Lifetime](values-lifetime.md#copying-and-direct-initialization).

## Destructors

```dot
destructor(self : &Self) {
    // body
}
```

A struct or object may declare at most one destructor. Its sole parameter must
be exactly `self : &Self`; it has no written return type, generic parameters,
constraints, default arguments, or overloads. Enums cannot declare destructors.

A destructor cannot be called explicitly. When destruction begins, all fields
are live. The custom body executes first, then fields are destroyed
automatically in reverse declaration order. A missing custom destructor is
equivalent to an empty body followed by automatic field destruction.

A destructor has no source name and therefore no underscore-derived private
visibility. Its automatic invocation is never access-checked; reflection treats
a declared destructor as public whenever the containing type itself is
accessible.

Destructors are implicitly non-throwing. If an exception attempts to escape a
destructor, the runtime terminates immediately. `return;` may leave a destructor
body early; returning a value is invalid.

## Functions

```dot
fn name<T>(parameters) -> ReturnType [constraints] {
    // body
}
```

Every function explicitly writes its return type, including `-> void`. A
non-`void` function must not fall off the end: each reachable path must return a
value, throw, or end in unconditional `panic`. A `void` function may fall off
the end or use `return;`.

A parameter is `name : Type` with an optional default expression:

```dot
fn open(path : str, retries : i32 = 0) -> Handle { ... }
```

Parameters are mutable bindings unless const-qualified. By-value parameters
copy their arguments. Reference and raw-pointer parameters receive their
respective values. Parameters of type `void` are invalid.

Defaulted parameters must follow all non-defaulted parameters. A default
expression is resolved in the declaration's lexical context, may refer to
earlier parameters, and is evaluated at the call site after all explicit
arguments and earlier defaults, in parameter order. It must exactly match the
parameter type after permitted const addition. Defaults are not part of a
function's effective signature.

Ordinary calls are positional. Named argument syntax exists only for attribute
uses. Version 1 has no ordinary variadic function.

Direct and mutual recursion are permitted. Nested runtime function declarations
are not.

## Methods and `self`

An instance method is a function declared in a struct, object, or enum and must
explicitly declare a first parameter whose identifier is exactly `self`.
`self` is not a reserved keyword outside this required parameter position and
may otherwise be an ordinary identifier.

For structs and objects:

```dot
fn mutate(self : &Self, amount : i32) -> void { ... }
fn inspect(self : const &Self) -> i32 { ... }
```

No other `self` type is permitted. A mutable receiver may call either method; a
const receiver may call only a `const &Self` method.

For enums, `self` must be by value:

```dot
fn flip(self : Self) -> Self { ... }
```

Method-call syntax automatically supplies the receiver without a struct copy or
object reference-count increment:

```dot
value.inspect();
```

The receiver expression is evaluated once and before every explicit argument.
An unqualified member use in a method may be resolved through `self` as defined
by member lookup.

There are no static methods. Constructors are the only type-qualified callable
members without `self`.

Method declarations cannot be converted to function values in version 1,
whether bound to a receiver or referenced through a type. A method is invoked
only with receiver-dot-call syntax.

## Operator declarations

```dot
operator +(
    self : const &Self,
    other : const &Self
) -> Self {
    // ...
}
```

An operator declaration is a method of the left operand's nominal type. Its
first parameter follows the ordinary `self` rules. The remaining arity must
match the operator:

- unary `-` and `!`: only `self`;
- binary `+`, `-`, `*`, `/`, `%`, `==`, `<`, `<=`, `>`, `>=`: `self` and one
  right operand;
- indexing `[]`: `self` and one index operand.

`~` remains a built-in-only byte and signed-integer operation. Comparison
operators must return `bool`. Indexing may return a value or reference.
Operators may be generic and constrained.

Defining `operator ==` also defines `!=` as Boolean negation of that exact
overload. A separately declared `operator !=` is invalid. Assignment, compound
assignment, increment/decrement, `&&`, `||`, address-taking, dereference,
member access, constructor invocation, function call, and namespace
qualification cannot be overloaded.

## Enums

```dot
enum Bit : i8 {
    on,
    off,

    constructor from(value : bool) -> Self {
        if (value) {
            return Self::on;
        }
        return Self::off;
    }

    fn flip(self : Self) -> Self {
        if (self == Self::on) {
            return Self::off;
        }
        return Self::on;
    }
}
```

The underlying type defaults to `i32` and may explicitly be `i8`, `i16`, `i32`,
`i64`, `i128`, or `byte`. Each enumerator may have an explicitly assigned
compile-time integer expression. The first implicit value is zero; every later
implicit value is the preceding enumerator's mathematical value plus one.
Failure to fit the underlying type is a compile error.

Duplicate underlying values are permitted. Enumerators remain distinct
declarations in reflection but compare equal at runtime. Enumerators are always
qualified, including inside the enum body.

Enumerator declarations precede enum members. If any member follows, the last
enumerator must have a trailing comma. With no members, the final comma is
optional. An enum may contain constructors, value-`self` methods, operators,
nested types, aliases, traits, attributes, and `comptime` blocks, but no fields,
destructor, reference-`self` method, or static method.

An enum constructor return expression must have exactly type `Self` after const
addition. Ordinary Dot code can obtain such a value only from an enumerator, an
enum-typed parameter/local/field, or another constructor, so every ordinary Dot
enum value has a declared underlying value. The language has no built-in
integer-to-enum conversion. Integer-from-enum obtains the underlying value and
then applies the destination integer conversion. A C boundary that supplies an
undeclared representation is covered separately by the C-interoperability
rules.

Enum equality is built in. Ordering is not built in but may be declared by
operator methods.

## Nested declarations

Structs, objects, and enums may contain nested structs, objects, enums, aliases,
traits, and attribute declarations. A nested declaration is qualified with its
containing type, for example `Container::Iterator`. It has no implicit outer
instance and does not capture one.

Nested nominal types have independent `Self`, fields, constructors, methods,
and privacy. A name beginning with `_` is private to the immediately containing
type.

## Overload sets

Functions, methods, operators, and same-named constructors form separate
overload sets in their declaring scope. Two declarations conflict when, after
alias expansion, they have the same:

- name and callable kind;
- number and types of parameters;
- parameter const/reference qualification;
- ABI; and
- generic parameter kinds and types.

Return type, parameter names, defaults, generic constraints, and visibility do
not distinguish effective signatures. A conflicting signature is a compile
error even if its return type or constraints differ.

## Overload resolution

For a call, the compiler:

1. forms the overload set selected by name or member lookup;
2. removes candidates with wrong arity after accounting for defaults;
3. substitutes explicit and inferred generic arguments;
4. removes candidates whose constraints are false;
5. removes candidates whose arguments do not exactly match parameters after at
   most adding const and applying the permitted reference-target read for a
   by-value type other than an object or array owner; then
6. ranks the remaining candidates.

Ranking prefers, in order:

1. fewer const-adding qualification conversions;
2. a non-generic candidate over a generic candidate when otherwise tied.

A reference expression can be interpreted as its binding for a reference
parameter or as a target read for an eligible by-value parameter. These are
equally exact interpretations; if both leave otherwise tied overloads, the call
is ambiguous rather than preferring one implicitly.

No other ranking exists in version 1. In particular, user constructors, result
context, default-argument count, source order, namespace proximity, and
constraint implication do not rank candidates. More than one equally ranked
candidate is an ambiguity and a compile error.

The return type is checked against its context only after one overload is
selected.

## Function values and lambdas

A non-overloaded function name converts to its function value. An overloaded
name requires an expected function type that selects exactly one overload or
explicit generic arguments that do so.

A non-capturing lambda is:

```dot
fn(value : i32) -> i32 {
    return value * 2;
}
```

It may use namespace declarations and compile-time constants through ordinary
lookup but may not refer to a runtime local, parameter, or `self` from an
enclosing function. Each lambda expression site introduces one anonymous
non-generic function declaration. Function identity is that declaration.
Lambda parameters cannot have defaults because defaults are not part of a
function type.

Capturing lambdas and bound-method function values are invalid in version 1.

## Attribute declarations

The declaration syntax and target vocabulary are normative in
[Metaprogramming](metaprogramming.md#attribute-declarations). An attribute
declaration occupies the ordinary declaration name space and follows the same
visibility rules as another declaration.
