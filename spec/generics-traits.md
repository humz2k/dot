# Generics and Traits

## Generic declarations

Structs, objects, aliases, functions, methods, operators, constructors, traits,
and nested declarations may be generic. Enums and destructors are not generic.

A generic parameter list follows the declared name:

```dot
struct Pair<First, Second> { ... }
fn repeat<T, N : i64>(value : T) -> T[] { ... }
```

An untyped parameter such as `T` is a type parameter. A parameter written
`name : Type` is a compile-time value parameter. Generic parameter names enter
scope after their declaration; a value parameter's type may refer to earlier
type parameters only when the resulting type is one of the permitted
compile-time value types.

Version-1 value parameters may have type `bool`, `i8`, `i16`, `i32`, `i64`,
`i128`, `byte`, or an enum type. Floating, string, tuple, array, syntax,
reference, pointer, object, and struct values are not generic arguments.

A generic parameter may not have a default. Parameter packs do not exist.

## Generic arguments

Arguments are supplied in declaration order:

```dot
Pair<i32, str>
Matrix<4, 4, f64>
make_buffer<1024, byte>()
Container<i32>@from<str>(source)
```

Type arguments must name complete types unless the declaration context
explicitly permits indirection to an incomplete type. Value arguments must be
compile-time expressions exactly matching the value parameter type; contextual
literal typing is allowed, but implicit conversion is not.

Named generic arguments, omitted type-declaration arguments, and explicit
specialization declarations are unsupported.

Generic arguments for a function, method, operator, or constructor appear after
that callable's name. Arguments for the containing nominal type appear in the
type before `@` or member access.

## Inference

At a call, omitted callable type parameters are inferred by structurally
matching parameter types against explicit argument types. Inference:

- uses no result or expected return type;
- performs no user conversion;
- may add const only after inference;
- requires every occurrence of one parameter to infer the same type/value; and
- does not infer a value parameter unless it appears in a parameter type or
  another inferable compile-time position.

Any uninferred generic argument must be written explicitly. Inference failure
removes the candidate during overload resolution; if no candidate remains, the
call is ill-formed.

Examples:

```dot
fn identity<T>(value : T) -> T { return value; }

x := identity(1);       // T is i32
y := identity<i64>(1);  // literal context is i64
```

## Constraints

Constraints are compile-time Boolean expressions in square brackets.

For types, aliases, and traits they follow the generic parameter list:

```dot
struct Box<T> [storable<T>] { ... }
trait serializable<T> [public_type<T>] { ... }
```

For callables they follow the return type:

```dot
fn clone<T>(value : const &T) -> T [copyable<T>] { ... }
constructor from<T>(value : T) -> Self [convertible<T, Self>] { ... }
```

The expression may use `&&`, `||`, `!`, parentheses, trait invocations,
compile-time comparisons, and calls to `comptime fn`. It must have exactly type
`bool`.

After generic substitution, a false constraint makes that specialization or
overload inapplicable; it is not by itself a diagnostic. An error while
evaluating a well-formed constraint is a compile-time diagnostic. Constraints
do not distinguish otherwise identical effective overload signatures and do not
rank matching overloads by implication.

Inside a selected generic body, its constraints are known to be true. The
compiler may use that fact for diagnostics and dependent operation checking.

## Two-phase checking

At generic declaration time, the compiler must:

- parse the complete declaration;
- resolve and validate every non-dependent name and expression;
- validate generic parameter and constraint syntax;
- reject operations invalid regardless of arguments; and
- retain dependent syntax for instantiation.

A name or operation is dependent when its meaning or validity depends on at
least one generic parameter. At instantiation, after substituting concrete
arguments and accepting constraints, the compiler resolves dependent names and
type-checks every dependent operation.

Dot does not use C++'s accidental lookup timing: non-dependent names always bind
to the declaration visible at definition, while dependent member lookup occurs
on the substituted static type. Later declarations cannot change a
non-dependent binding.

An unconstrained generic is permitted to perform a dependent operation. Each
instantiation that lacks that operation is ill-formed and receives a diagnostic
at the instantiation plus the dependent source location.

## Instantiation and monomorphization

Each unique generic declaration and normalized argument list denotes one
specialization. Type aliases are expanded when normalizing type arguments;
compile-time values compare by type and value.

Specializations are conceptually monomorphized. The backend may share machine
code between semantically equivalent specializations only when no observable
type identity, reflection result, function identity, exception type, symbol,
destructor behavior, or concurrency behavior changes.

Instantiation is demand-driven when a specialization is required by reachable
type checking or compile-time evaluation. A compiler may eagerly instantiate
additional specializations for diagnostics but must not make a valid program
invalid merely because an unused, unconstrained specialization would fail.

Recursive instantiation is permitted when it reaches a finite already-known
specialization. If discovering a specialization requires an unbounded sequence
of new argument lists, the compiler must diagnose recursive instantiation. The
diagnostic includes the cycle or a finite prefix demonstrating growth.

Version 1 has no user-written explicit or partial specialization. Overloads and
constraints are the only specialization mechanism.

## Traits

A trait is a named, generic compile-time predicate:

```dot
trait default_constructible<T> {
    for (constructor in reflect(T).constructors) {
        if (
            constructor.name == "default"
            && constructor.parameters.length() == 0
            && constructor.return_type == T
        ) {
            return true;
        }
    }

    return false;
}
```

Trait bodies are implicitly compile-time and must return `bool` on every
reachable path. Empty trait bodies and falling off the end are compile errors.
They may use the compile-time subset, reflection, other traits, and `comptime
fn`, but cannot emit declarations or perform runtime operations.

Traits have no runtime value, `impl` declaration, method slot, associated type,
associated constant, vtable, inheritance, or nominal conformance list. A type
satisfies a trait exactly when evaluation with that type returns true.

Trait invocations use generic-call syntax and are compile-time `bool` values:

```dot
copyable<MyStruct>
same_shape<Left, Right>
```

## Constrained traits

A trait may have prerequisite constraints:

```dot
trait my_trait<T> [copyable<T> && other_trait<T>] {
    return reflect(T).fields.length() > 0;
}
```

Prerequisites evaluate before the body. If any is false, the trait evaluates to
false without executing its body. Within the body, every prerequisite is known
true. An error in a prerequisite is a compile-time error rather than false.

This behavior makes a constrained trait safe in Boolean composition: asking
whether a type satisfies it never executes a body whose stated preconditions do
not hold.

## Trait recursion and memoization

Trait evaluation is pure and deterministic. The compiler memoizes or behaves as
if it memoized each trait argument list.

If evaluating a trait requests the same trait specialization before a result is
known, and Boolean short-circuiting cannot determine the result without that
request, the recursion is unresolved and is a compile error. The compiler must
not arbitrarily treat recursion as true or false.

For example, `A<T> { return A<T>; }` is an error. `A<T> { return false && A<T>;
}` returns false because the recursive operand is not evaluated.

## Reflection-based signature matching

Traits inspect declarations through the typed metadata API in
[Metaprogramming](metaprogramming.md#reflection-metadata). A method match is
exact only when all requested properties match:

- name and callable kind;
- complete parameter types in order;
- `self` const/reference form;
- return type;
- generic parameter kinds and value-parameter types;
- ABI; and
- normalized constraint syntax when the query requests constraints.

Visibility is not part of signature identity, but reflection exposes only
declarations accessible at the trait invocation location. Defaults and
parameter names are metadata but do not affect callable signature matching.

The uppercase sketch operations `HAS_METHOD`, `HAS_CONSTRUCTOR`, `SAME_TYPE`,
`TYPE_OF`, and `CONSTRUCTED` are not language constructs.

## Diagnostics

A diagnostic caused by a generic or trait reports:

1. the primary failing dependent expression or compile-time operation;
2. the generic/trait declaration location; and
3. the chain of instantiations or trait evaluations leading to it.

Constraint-false candidates do not produce errors unless needed to explain why
an overload set has no viable member. Implementations must include their failed
constraints in that explanation.
