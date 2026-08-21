# Values, Ownership, and Lifetime

## Storage duration

A runtime value has one of these storage durations:

- **local**: a local variable or parameter lives from completed initialization
  until its lexical scope or call ends;
- **field**: a field lives for the lifetime of its containing struct value or
  object allocation;
- **element**: an array or optional element lives while present in its backing
  storage;
- **temporary**: an unnamed materialized value lives until the end of its full
  statement unless direct initialization avoids a separate temporary; or
- **static constant**: a namespace constant exists for the program duration and
  requires no runtime destruction.

There is no uninitialized runtime value. Storage whose initialization fails does
not begin a value lifetime, except that already initialized subobjects are
destroyed during cleanup.

## Copying and direct initialization

Passing, returning, storing, or otherwise initializing from an existing
by-value expression performs the source type's copy operation:

- primitives and enums copy their bits/value;
- a struct or tuple recursively copies fields/elements in declaration/index
  order;
- an object copies its owning handle and atomically increments the strong count;
- an array copies its owning backing-store handle and atomically increments its
  count;
- a `str` copies its value representation, incrementing an atomic backing count
  when it uses shared heap storage;
- an optional copies its discriminator and, when engaged, copies its value;
- a reference copies its binding without changing ownership;
- a raw pointer copies its address;
- a function value copies its function identity.

Copying cannot invoke user code and cannot be overloaded. Dot has no move
operation. A backend may eliminate a copy only when doing so cannot change any
observable destructor execution, exception, identity, atomic ownership effect,
or inter-thread behavior allowed by this specification.

A fresh constructor result or `Self { ... }` initializer **directly
initializes** the destination when it is the complete initializer of a variable,
parameter, field, array/tuple literal element, `Self` field, or function result.
Direct initialization creates one value in the destination and no second
temporary. It is not a move and provides no operation applicable to an existing
source value. A by-value call parameter is itself a destination; a later copy
from that parameter inside the callee is a separate operation and is not elided
by this rule.

Returning an existing local or parameter by value copies it into the function
result before local destruction. Returning an object or array thus increments
the relevant ownership count; returning an existing struct makes a structural
copy.

## Struct values

Each struct value owns its fields directly. Its custom destructor, if any, runs
whenever that particular value's lifetime ends. Structural copies are
independent struct lifetimes even when some copied fields share object, array,
or string allocations.

Because copying is structural and cannot be customized, a struct that stores a
unique external resource only as a raw pointer or integer handle is unsafe to
copy unless its design makes duplicate values harmless. The language supplies
no unique-ownership exception to structural copying.

## Object ownership

An object value owns one strong reference to its allocation. Its strong count is
positive while any object value owns it. Copying increments the count; ending or
replacing an owning handle decrements it. The final decrement synchronously:

1. begins destruction on the thread performing that decrement;
2. runs the custom object destructor body;
3. destroys fields in reverse declaration order; then
4. releases allocation and ownership-control storage.

An `&Object` or `*Object` points to the allocation, not to a particular owning
handle variable. Taking either does not change the strong count and cannot be
used to recover an owning handle. Such access is valid only while at least one
owning handle keeps the allocation alive.

An object allocation cannot be structurally assigned as a whole. Assignment
through `&Object` is therefore invalid, although fields may be assigned and
mutable methods called.

Object handles are non-null. Owning cycles composed of object and/or array
handles are permitted and keep every count in the cycle positive, so their
destructors never run unless the cycle is manually broken. The leak itself is
defined behavior. Version 1 has no weak reference or cycle collector.

## Array and string ownership

An array value owns a non-null reference-counted backing allocation. Copies
share both elements and mutations. A string value has immutable value semantics;
its representation may store short contents inline or share immutable heap
storage. Representation choices are not observable through identity.

Array backing-store destruction destroys live elements in decreasing index
order and releases storage. Reallocation may create a new backing allocation
internally but does not change `same_identity` between aliases: every alias must
continue to denote the same logical array allocation and observe its updated
storage. Thus `same_identity` compares the stable array control identity, not a
transient element buffer address.

An `&T[]` or `*T[]` denotes the logical array without ownership, not an array
handle binding. It permits array operations while an independent owning handle
keeps the logical array alive but cannot initialize a new owning array handle.

## References

A reference is non-owning, non-null, and non-reseatable. It is created only from
an addressable existing place:

```dot
reference : &T = &value;
```

Literals, constructor results, function results, arithmetic results, and other
temporaries cannot be referenced. Locals, parameters, fields, array elements,
tuple elements, reference targets, and valid raw-pointer dereferences are
addressable.

A reference does not extend lifetime. It may be returned, stored, or copied even
when it will later dangle; the compiler need not diagnose this. Any value read,
write, member access, method call, address derivation, or further reference
creation through a dangling reference is undefined behavior.

Assignment to a reference expression assigns through the reference to its
target. It never rebinds the reference:

```dot
r : &i32 = &left;
r = right; // replaces left's value; r still refers to left
```

When a containing struct is replaced, the old reference field's lifetime ends
and the new field is initialized with the copied source binding. That operation
is replacement of the containing value, not reseating a live reference.

## Raw pointers

`raw &place` creates a raw pointer to the same storage that `&place` would
reference. For an object value it points to the object allocation; for an array
it points to the logical array. A raw pointer may be assigned `null`, reseated,
copied, and compared for equality with a compatible pointer or `null`.

Dereference is explicit:

```dot
value := *pointer;
field := (*pointer).field;
```

Dereferencing null, dangling, incorrectly typed, insufficiently aligned, or
otherwise invalid storage is undefined behavior. Merely storing or comparing a
dangling pointer is permitted as long as the comparison's pointer types are
compatible and no access is attempted. Equality compares addresses; no ordering
or arithmetic is defined.

`&*pointer` produces a reference after dereferencing. It has the same validity
requirements as any dereference. Converting through `*void` preserves the
address but establishes no right to access storage as the recovered type.

## Replacement assignment

For an assignable destination `D` and source expression `E`, `D = E` has these
semantics:

1. Evaluate the destination place once.
2. Evaluate `E` completely from left to right.
3. Create one complete snapshot value `R` in temporary storage by copying `E`,
   unless a fresh `E` directly initializes `R`.
4. If evaluation or creation of `R` throws, leave `D` unchanged.
5. End and destroy the old destination value.
6. Initialize the new destination value by copying `R`.
7. Destroy `R` at the end of the full expression, after the new destination has
   begun its lifetime.

The snapshot, second structural copy, and snapshot destruction are observable
under the normal ownership-count and destructor rules; assignment is not a
move. Copying and destruction cannot throw user exceptions, so steps 5 through
7 do not fail. A destructor attempting to throw terminates rather than resuming
assignment.

Self-assignment is valid because the snapshot precedes destruction. References
or pointers into the old destination or its direct fields may dangle after step
5. The result type of assignment is `void`; chained assignment is therefore
invalid.

Special cases are:

- Assignment to a const-qualified destination is invalid.
- Assignment to a reference expression replaces its target under this algorithm
  and does not rebind the reference.
- Assignment to a raw-pointer binding reseats it and does not affect the
  pointee.
- Assignment to an object handle releases its old allocation ownership and
  copies the new handle; it never assigns object fields wholesale.
- Assignment to an array handle changes which logical array the variable owns;
  aliases of the old array continue to own and observe the old logical array.

Whole-target assignment through `&Object`, `*Object`, `&T[]`, or `*T[]` is
invalid because those non-owning values denote allocations rather than owning
handle bindings. Their mutable fields, elements, and methods remain assignable
or callable.

Compound assignment evaluates the destination place once, computes the
corresponding binary operator using its old value and the right operand, then
performs replacement assignment. Its result is `void`.

## Destruction and scope exit

When a lexical scope exits, initialized locals declared directly in that scope
are destroyed in reverse completion-of-declaration order. This rule applies to
fallthrough, `return`, `break`, `continue`, and exception unwinding.

On function exit, nested/body locals are destroyed by the same scope rule, then
by-value parameters are destroyed in reverse parameter order. Reference and raw
pointer parameters have trivial destruction. The function result or thrown
value is initialized before either body locals or parameters are destroyed.

Before a control transfer leaves a scope, every affected local is destroyed.
The transferred return or thrown value is copied/directly initialized before
locals on which its source depends are destroyed.

A temporary materialized during a statement is destroyed at the end of that
full statement, in reverse completion-of-construction order. Temporaries used to
initialize a local are destroyed only after the local initialization completes.
No reference may bind to a temporary, so Dot has no temporary-lifetime
extension rule.

For this rule, each of the following is one **full expression**: an expression
statement; a local initializer; a `return` or `throw` operand;
an `if`/`elif`/`while` condition; each C-style `for` initializer, condition, and
increment clause; the iterator-loop source initializer; the `switch`
controlling expression; and each field expression in a `Self { ... }`
initializer. Call arguments, constructor arguments, assignment operands, and
literal elements are subexpressions of their enclosing full expression, not
separate boundaries.

Condition temporaries are destroyed after the Boolean value is obtained and
before entering a branch/body. A `for` increment's temporaries are destroyed
before the next condition. The switch controller is copied into hidden enum
storage before its temporaries are destroyed and label selection begins. A
return/throw result is initialized before operand temporaries are destroyed;
scope or exception unwinding follows afterward. In a `Self` initializer, each
field becomes initialized before that field expression's temporaries are
destroyed and before evaluation of the next field.

Fields are destroyed in reverse declaration order after the custom destructor
body. Tuple elements are destroyed in reverse index order. Array elements are
destroyed in decreasing index order. An engaged optional destroys its contained
value; an empty optional destroys nothing.

## Identity

`same_identity(left, right)` is a compiler-provided function-like intrinsic. It
accepts two values of the same object type, same array type, or same function
type after permitted const addition and returns `bool`.

- For objects, it compares allocation identity.
- For arrays, it compares stable logical backing-allocation identity.
- For function values, it compares the selected function or lambda declaration
  and, for a generic named function, its complete generic instantiation.

Operands are evaluated left-to-right but are inspected without by-value copies
or ownership-count changes. `same_identity` is not overloadable. It is invalid
for strings, structs, tuples, optionals, enums, primitives, references as
addresses, or raw pointers. A reference to an object or array is transparently
accepted as the referenced value; raw-pointer equality is used when addresses
themselves must be compared.

## Invalidated and ended values

After a value's lifetime ends, its former storage contains no Dot value until a
new lifetime begins there. Access through an old reference or pointer is
undefined behavior even if a new value of the same type later occupies the same
address; version 1 provides no pointer-laundering operation.

An array operation that invalidates element references ends the validity of all
such references and raw pointers as specified in
[Built-in Runtime Types](builtins.md#invalidation). The referenced element may
still logically exist at another address, but the old non-owning value cannot be
used to find it.
