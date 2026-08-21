# Type System

## General model

Dot is statically typed. Every expression has one compile-time type. Except for
contextual literal typing and adding const qualification, a value must exactly
match the type required by its context. There is no implicit numeric,
user-defined, enum, pointer, optional, or object conversion.

Types are either nominal or constructed:

- Each `struct`, `object`, and `enum` declaration introduces a distinct nominal
  type.
- Primitive types are distinct built-in nominal types.
- Tuples, arrays, optionals, references, raw pointers, and function types are
  constructed from their component types.
- A type alias is transparent and introduces no new type identity.

Two instantiations of one generic nominal declaration are the same type exactly
when all corresponding type and value arguments are equal.

## Primitive types

Version 1 provides:

| Type | Meaning |
| --- | --- |
| `bool` | `true` or `false` |
| `i8` | signed two's-complement 8-bit integer |
| `i16` | signed two's-complement 16-bit integer |
| `i32` | signed two's-complement 32-bit integer |
| `i64` | signed two's-complement 64-bit integer |
| `i128` | signed two's-complement 128-bit integer |
| `byte` | unsigned 8-bit raw-storage value |
| `f32` | IEEE-754 binary32 |
| `f64` | IEEE-754 binary64 |
| `str` | immutable byte string |
| `void` | absence of a function result |

There is no `never`, character, general unsigned-integer, or implicit
machine-word integer type.

`void` is permitted only as a function or callback return type and as the
pointee of `*void`. A variable, field, parameter, array element, optional
element, tuple element, or generic value cannot have type `void`.

Integer operations are performed at the declared width; Dot does not apply C or
C++ integer promotions. Floating operations are performed at the declared
format using round-to-nearest, ties-to-even, with gradual underflow, signed zero,
infinities, and NaNs. An implementation must not reassociate floating
expressions, replace division with reciprocal multiplication, flush subnormals,
or contract operations into a fused operation unless an explicit future Dot
operation requests it. NaN payload propagation is implementation-defined, but
whether a result is NaN is not.

## Struct types

A `struct` is a value-semantic nominal type. Its value consists of one value for
each field. Struct values are structurally copied and destroyed as specified in
[Values and Lifetime](values-lifetime.md). A struct has no implicit default
value and cannot customize copying.

Distinct struct declarations remain distinct even when their fields are
identical. Generic struct instantiations are nominal instantiations of their
generic declaration.

## Object types

An `object` value is a non-null owning handle to one heap allocation of its
nominal object type. Its allocation contains the declared fields. Copying an
object value copies the owning handle and atomically increments its strong
reference count; it never clones the fields.

Object types have no subtype relationships. An object value cannot be null.
Optional ownership is expressed as `ObjectType?`.

## Enum types

An `enum` is a scoped nominal value type backed by an explicitly selected or
default integer representation. It contains no payload or fields. Every valid
ordinary Dot enum value corresponds to at least one declared enumerator.

Enums with the same enumerators are distinct when declared separately. Their
representation and operations are defined in
[Declarations](declarations.md#enums).

## Tuple types

```dot
(i32, str)
(i32,)
()
```

A tuple type is structurally identified by its ordered element types and arity.
`(T)` is the grouped type `T`; `(T,)` is a one-element tuple. `()` is the
zero-element tuple and is distinct from `void`.

Tuple elements may have any storable non-`void` type, including references.
Copying a tuple structurally copies its elements from index zero upward.

## Array types

`T[]` is a dynamically sized, mutable, reference-semantic array whose elements
have type `T`. `T` must be a storable non-`void` type. Arrays are always valid,
non-null owning handles. There are no fixed-size or stack arrays.

Postfix type constructors associate left-to-right:

```dot
T?[]   // array of optional T
T[]?   // optional array of T
```

Array behavior is defined in [Built-in Runtime Types](builtins.md#arrays).

## Optional types

`T?` is a value containing either `none` or one `T`. `T` must be a storable
non-`void`, non-reference type. Raw pointers, objects, arrays, strings, structs,
enums, tuples, and function values may be optional. An optional object owns its
object handle while engaged.

`T?` and `T` are distinct. There is no implicit injection into an optional.

## Reference types

`&T` is a non-null, non-owning, non-reseatable reference to mutable `T`.
`const &T` is a non-null reference through which `T` cannot be mutated. The
canonical spelling places `const` before `&`; `&const T` is not valid syntax.

A reference is transparent for member access, indexing, method/operator calls,
and assignment that is valid for its target. In a context whose expected type
is itself a reference, a reference expression denotes its binding rather than
copying its target. Consequently, a reference variable may be passed to another
reference parameter directly. Taking `&` of a non-reference place creates a
reference explicitly.

Reading a reference to a primitive, enum, struct, tuple, optional, string, raw
pointer, or function value in a by-value context copies the referenced value.
An `&Object` instead denotes the object allocation, and an `&T[]` denotes the
logical array; neither contains an owning handle that can be recovered. Such a
reference cannot initialize, be passed as, or return a by-value object/array.
It remains usable for field/element access, calls, and `same_identity` without
an ownership-count change. The same rule applies to an object/array place
obtained by raw-pointer dereference.

References may be fields, parameters, return values, tuple elements, and array
elements. They do not own or extend lifetime. There is no reference-to-reference
type: applying `&` to a reference expression preserves the same reference type
and target.

## Raw-pointer types

`*T` is a nullable, non-owning, reseatable raw pointer. A raw pointer may have
any non-reference runtime pointee type, including an incomplete nominal type,
or `void`. Forming, copying, comparing, and passing a pointer does not require a
complete pointee. Dereferencing it or accessing the resulting value requires
that pointee to be complete at the access site. `*void` cannot
be dereferenced. A raw pointer to a reference or to `void` with additional
postfix construction is ill-formed.

Const placement has these meanings:

| Type | Meaning |
| --- | --- |
| `*T` | mutable pointer binding, mutable pointee access |
| `*const T` | mutable pointer binding, const pointee access |
| `const *T` | const pointer binding, mutable pointee access |
| `const *const T` | const pointer binding, const pointee access |

The leading `const` controls whether the pointer variable may be reseated. The
`const` immediately after `*` controls access through dereference. Further
pointer nesting applies the rule recursively.

Raw pointers do not own or extend lifetime and support no pointer arithmetic.

Prefix type constructors bind less tightly than postfix `[]` and `?`. Thus
`*T[]` is a pointer to an array, `&T[]` is a reference to an array, and
`*T?` is a pointer to an optional. Parentheses select the opposite grouping:
`(*T)[]` is an array of raw pointers and `(&T, i32)` is a tuple whose first
element is a reference. The same precedence applies recursively to `const`.

## Function types

```dot
fn(i32, str) -> bool
extern "C" fn(i32, *void) -> void
```

A Dot function type is structurally identified by:

- its ABI (`Dot` when no `extern` appears);
- its ordered parameter types; and
- its return type.

Parameter names, defaults, and the source function name are not part of the
type. A function type is non-null. Generic functions are not function values
until instantiated with all generic arguments.

Only non-capturing functions and lambdas produce function values in version 1.
An ordinary Dot function type and an `extern "C"` function type are distinct and
do not convert.

## Const qualification

`const T` is an access-path-qualified view of `T`. It prevents replacement of
the qualified binding and mutation through that access path. Const does not
freeze the underlying allocation globally; another mutable alias may still
mutate it, subject to the concurrency rules.

Const propagates through directly owned value paths:

- a field reached through `const Struct` or `const Object` is const for that
  access;
- an element reached through a const array or tuple is const;
- an engaged value reached through a const optional is const;
- an owning object or array field reached through const remains a const view of
  its underlying allocation.

Const does not propagate through a separately declared reference or raw-pointer
pointee. If a const struct contains `target : &T`, using `target` may mutate the
`T` because the field's declared reference is mutable. If it contains `pointer
: *T`, the pointer binding is const for that access but dereferencing it still
produces mutable `T`. A field declared `const &T` or `*const T` remains const at
the pointee independently.

A mutable value or reference may be used where the same const-qualified type is
expected. This is the language's only general implicit qualification
conversion. Const qualification cannot be removed, including by copying an
owning handle:

```dot
source : const MyObject = obtain_const();
target : MyObject = source; // compile error
```

For a reference, `const &T` means reference-to-const rather than a const
reference binding, because all references are already non-reseatable.

## Type aliases

```dot
alias UserId = i64;
alias Values<T> = T[];
```

An alias is replaced by its target for all type-equivalence, overload, layout,
and reflection operations. Reflection may retain alias-declaration metadata
when reflecting a containing type, but `reflect(UserId)` reflects `i64`.

Alias expansion must terminate. Direct or indirect alias cycles are compile
errors even if a cycle could appear behind a pointer or optional.

## Type equivalence

After alias expansion and normalization of const spelling, types are equivalent
when one of these holds:

- they are the same primitive;
- they name the same nominal declaration with equal generic arguments;
- they are tuples of equal arity with pairwise equivalent element types;
- they are the same constructed kind (array, optional, reference, pointer) with
  equivalent component types and equal const qualification; or
- they are function types with equal ABI, parameter types, and return type.

`const T` is not equivalent to `T`, although the permitted qualification
conversion may make a value usable. Generic value arguments compare by type and
value.

## Explicit conversions

An explicit conversion is a named constructor call:

```dot
narrow : i8 = i8@from(wide);
textual := UserId@from(source);
```

User-defined `from` is an ordinary constructor name and receives no privileged
overload behavior. Primitive types provide these built-in constructors:

- every signed integer type from every signed integer type, `byte`, and `bool`;
- `byte` from every signed integer and `bool`;
- `bool` from every signed integer, `byte`, `f32`, and `f64`;
- `f32` and `f64` from every signed integer, `byte`, `bool`, `f32`, and `f64`;
- every signed integer and `byte` from `f32` and `f64`;
- an integer or `byte` from any enum, by first obtaining its underlying value;
- `*void` from any raw pointer and any typed raw pointer from `*void`.

There is no built-in integer-to-enum constructor. Enum conversion in that
direction exists only when the enum declares a matching user constructor.

Conversion results are:

- Integer-to-integer and integer-to-`byte` retain the low destination-width
  bits; a signed destination interprets those bits as two's complement.
- Integer-from-`byte` uses its value from 0 through 255.
- `bool@from(integer-or-byte)` is false exactly for zero.
- `bool@from(float)` is false exactly for positive or negative zero and true
  for every other value, including NaN and infinities.
- Integer-or-`byte` from `bool` produces zero or one.
- Floating-to-integer truncates toward zero. NaN, infinity, or a truncated value
  outside the mathematical destination range causes undefined behavior.
- Integer-to-floating and `f64`-to-`f32` use round-to-nearest, ties-to-even.
  `f32`-to-`f64` is exact.
- Pointer-to/from-`*void` preserves the address and const qualification; it
  cannot remove const.

No constructor is considered implicitly by assignment, calls, returns,
operators, overload resolution, array literals, or conditional checks.

## Storable and complete types

A **storable type** is any non-`void` runtime type other than an incomplete
nominal type by value. During declaration of `struct S`, a field cannot contain
`S` directly or through only tuples because that would require infinite size.
It may contain `S` through an object handle, array handle, optional object
handle, reference, or raw pointer where the representation has finite size.

An object handle, array handle, string, optional, reference, raw pointer,
function value, enum, primitive, and fully complete struct or tuple are storable.
Completeness is checked after alias expansion.
