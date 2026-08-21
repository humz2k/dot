# Built-in Runtime Types and Operations

## General requirements

The types and members in this chapter are language-provided. They behave as if
they had the listed declarations, but their implementations may be compiler or
runtime intrinsics. A program cannot shadow a built-in member through an
extension or replace its semantics by declaration.

Members use ordinary call syntax and ordinary explicit reference rules. Method
calls automatically supply `self`.

## Strings

`str` is an immutable sequence of bytes. It does not promise valid UTF-8.
Embedded zero bytes are ordinary contents. Source literals begin as UTF-8, but
library and FFI operations may construct arbitrary byte sequences.

The compiler provides behavior equivalent to:

```dot
fn length(self : const &Self) -> i64;
fn is_empty(self : const &Self) -> bool;
operator [](self : const &Self, index : i64) -> byte;
operator +(self : const &Self, other : const &Self) -> str;
operator ==(self : const &Self, other : const &Self) -> bool;
operator <(self : const &Self, other : const &Self) -> bool;
operator <=(self : const &Self, other : const &Self) -> bool;
operator >(self : const &Self, other : const &Self) -> bool;
operator >=(self : const &Self, other : const &Self) -> bool;
```

`length` is the byte length and cannot be negative. `is_empty` is equivalent to
`length() == 0`. Indexing is unchecked and returns a byte value. Equality and
ordering compare unsigned bytes lexicographically. `!=` is derived from
equality. Concatenation allocates a value containing the exact left bytes then
right bytes; allocation failure throws `allocation_error`.

A string has value semantics. Equal strings need not share representation, and
representation identity is not observable. The implementation may use inline
small-string storage and atomically reference-counted immutable heap storage.
Heap-backed copies increment their ownership count; inline copies copy bytes.

The remainder of text encoding, searching, slicing, formatting, and Unicode
processing belongs to libraries rather than the version-1 language.

## Arrays

### Model

`T[]` is a non-null owning handle to one stable logical array identity containing
an ordered, mutable sequence of `T`. Aliases share mutations. The logical
identity owns an element buffer whose address and capacity may change.

An element has a logical lifetime independent of its current buffer address.
When an operation reallocates the buffer or shifts surviving elements to new
indices, the runtime relocates their representations without a Dot copy,
destructor call, ownership-count adjustment, or new logical lifetime. This
internal relocation has no source expression and is not Dot's absent move
operation. Its observable effect is the invalidation specified below. A newly
inserted element and a removed element still begin and end their logical
lifetimes normally.

The array strong reference count is atomic. Releasing the final handle destroys
elements in decreasing index order and releases the buffer and control storage.

### Constructors and literals

The compiler provides:

```dot
T[]@new() -> T[]
T[]@new(size : i64, initial_value : T) -> T[]
T[]@reserve(capacity : i64) -> T[]
```

- `new()` creates a fresh empty identity.
- `new(size, initial_value)` creates `size` elements, each initialized by
  copying `initial_value` in increasing index order.
- `reserve(capacity)` creates a fresh empty identity with capacity at least the
  request.

Negative requests or capacity arithmetic overflow are undefined behavior.
Allocation failure throws `allocation_error`. If element population fails for a
runtime reason before completion, initialized elements are destroyed in reverse
order and no array escapes. Structural copies themselves do not throw.

An array literal creates a fresh identity and initializes elements left-to-right.
`[]` requires an expected array type.

### Core members

Every `T[]` provides behavior equivalent to:

```dot
fn length(self : const &Self) -> i64;
fn capacity(self : const &Self) -> i64;
fn is_empty(self : const &Self) -> bool;

fn reserve(self : &Self, capacity : i64) -> void;
fn resize(self : &Self, size : i64, initial_value : T) -> void;
fn push(self : &Self, value : T) -> void;
fn pop(self : &Self) -> T;
fn insert(self : &Self, index : i64, value : T) -> void;
fn remove(self : &Self, index : i64) -> T;
fn clear(self : &Self) -> void;

fn forward_iterator(self : const &Self) -> array_iterator<T>;
```

The explicit signatures describe parameter-copy behavior. For example,
`push(existing_struct)` first copies into its by-value parameter and then copies
that parameter into element storage; a fresh constructor result may directly
initialize the parameter, but there is no move into the element.

### Member semantics

- `length()` is the current number of live elements.
- `capacity()` is at least `length()` and is the number of elements that can be
  held without another buffer allocation.
- `is_empty()` is `length() == 0`.
- `reserve(n)` ensures `capacity() >= n`, never reduces capacity, and does
  nothing—including no invalidation—when `n <= capacity()`.
- `resize(n, value)` destroys elements in decreasing index order when shrinking
  and copies `value` into new elements in increasing index order when growing.
- `push(value)` appends a copy at the old length.
- `pop()` copies the final element into the result, destroys the stored element,
  reduces length, and returns the copy.
- `insert(i, value)` initializes one new element by copying `value` before old
  index `i`; `i == length()` appends. Existing elements at and after `i` retain
  their values, logical lifetimes, and relative order while their indices
  increase by one.
- `remove(i)` copies element `i` into the result, destroys that logical element
  exactly once, shifts later surviving elements down without copying or ending
  them, and returns the result copy.
- `clear()` destroys every element in decreasing index order, sets length to
  zero, and retains capacity.
- Built-in `array[index]` is an intrinsic element-place operation rather than a
  call to a declared `operator []`. It produces the actual owning element slot,
  with const determined by the receiver access path. Reading the place copies
  its stored `T`; assigning it replacement-assigns that slot. Applying `&` or
  `raw &` follows the ordinary address rule, so an object/array element yields a
  non-owning reference/pointer to its allocation rather than to its owning slot.

`pop` on an empty array, indexing outside `[0, length)`, insertion outside
`[0, length]`, removal outside `[0, length)`, negative size/capacity, and
capacity arithmetic overflow are undefined behavior.

Growth strategy and excess capacity are unspecified. Allocation failure during
`reserve`, growing `resize`, `push`, or `insert` throws `allocation_error` and
leaves array contents, length, capacity, and identity unchanged.

### Invalidation

Any successful operation that changes array length or capacity invalidates all
references, raw pointers, and iterators to its elements, even if the physical
buffer did not move. `reserve(n)` with `n <= capacity()` and read-only operations
do not invalidate. Replacing an array handle invalidates nothing in the old
logical array for other aliases, but non-owning access through the replaced
handle may dangle if it released the final owner.

Use of an invalidated element reference, raw pointer, or iterator is undefined
behavior.

### Equality and identity

Array equality is available exactly when `T == T` is available and returns
`bool`. It first compares lengths, then compares corresponding elements in
increasing index order, stopping at the first inequality. It performs no cycle
detection; user-defined recursive equality may fail to terminate.

`same_identity` compares stable logical array identity and does not compare
contents. Internal buffer reallocation does not change logical identity.

### Iteration

Arrays provide a `forward_iterator()` accepted by the iterator-loop protocol.
It returns the compiler-provided generic type `array_iterator<T>`, which has no
public constructor and provides:

```dot
fn done(self : const &Self) -> bool;
fn next(self : &Self) -> T;
```

`array_iterator<T>` has struct value semantics. It conceptually contains an
owning array handle and an `i64` cursor; structurally copying it produces an
independent cursor, copies the owning handle, and increments the array count.
It has no equality, identity, or user-declared members beyond those listed.

The iterator begins at index zero, retains an owning array handle so the logical
array remains alive, reports done when its index equals the current length, and
returns the current element by value before incrementing its index. Struct
elements copy and object/array elements copy owning handles. Any length- or
capacity-changing operation invalidates the iterator and every copy of it.

Creating this built-in iterator atomically acquires one owner from the known-live
array receiver. This compiler-provided operation is the sole exception that
creates an array owner from non-owning array access; it does not expose a
general reference-to-owner conversion.

## Tuples

A tuple literal or expression directly initializes elements from index zero
upward. Tuple copying and destruction follow the lifetime rules in
[Values and Lifetime](values-lifetime.md).

Tuple equality is available exactly when equality is available for every
element type. It evaluates the left tuple and then the right tuple once, compares
elements in increasing index order, and stops at the first unequal pair. The
empty tuple equals every other empty tuple. `!=` is the negation of this
equality. Tuples have no automatic ordering or identity operation.

## Optional values

`T?` contains a discriminator and, when engaged, one directly owned `T` value.
It provides:

```dot
T?@some(value : T) -> T?

fn has_value(self : const &Self) -> bool;
fn value(self : &Self) -> &T;
fn value(self : const &Self) -> const &T;
```

`none` initializes an empty optional. Calling `some` first initializes its
by-value parameter under ordinary call rules, then copies that parameter once
into the engaged contained value. A fresh constructor argument may directly
initialize the parameter but does not remove the second copy. `has_value`
reports the discriminator. `value` returns a reference to the contained value
or throws a runtime-provided `bad_optional_access` object when empty.

Replacing or destroying an engaged optional destroys its contained value.
Copying it copies the value. Empty copies remain empty.

When `T` has equality, `T?` has equality:

- `none == none` is true;
- empty and engaged are unequal; and
- two engaged values compare their contained values.

Optional equality evaluates left then right and performs at most one contained
comparison. Optionals have no automatic ordering or identity operation.

## Function values

A function value is a non-null identity for one non-generic named function,
complete generic named-function instantiation, or non-capturing non-generic
lambda. Copying it is trivial and has no destruction effect. Calling it follows
its function type.

Function values have no `==` or ordering. `same_identity` compares function
identity. Two evaluations of the same named function instantiation have the
same identity. Two distinct lambda expression sites have different identity;
repeated evaluation of one non-generic lambda site has the same identity.

## Identity intrinsic

The compiler provides:

```dot
same_identity(left, right) -> bool
```

for same-typed objects, arrays, and function values, allowing const addition.
It examines operands without copying or changing ownership counts and cannot be
overloaded. Its exact meanings are given in
[Values and Lifetime](values-lifetime.md#identity).

## Standard runtime exception objects

The runtime provides public object types equivalent in role to:

```dot
object allocation_error {
    fn message(self : const &Self) -> str;
}

object bad_optional_access {
    fn message(self : const &Self) -> str;
}
```

The exact namespace containing these types is the root namespace in version 1.
Their constructors are runtime-provided and need not be directly callable by a
program. `allocation_error` is thrown when a required managed allocation cannot
be completed. The runtime must keep a non-allocating emergency representation
available so reporting allocation failure does not require another successful
general allocation.

`bad_optional_access` is thrown by empty optional `value()` access. Its
exception object likewise may use a shared immutable runtime instance because
exception identity has no specified meaning for this failure.

## Panic and fatal reporting

`panic(str) -> void` is a non-overloadable intrinsic described in
[Statements](statements.md#panic). Stack traces are required when the runtime
can obtain them without compromising termination; their availability and text
format are implementation-defined. Fatal process status must be nonzero.
