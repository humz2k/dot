# Concurrency and Memory Model

## Scope

Dot version 1 defines a shared-memory concurrency model but no thread-creation,
mutex, condition-variable, or user-facing atomic syntax. Those APIs belong to
the standard library and may be implemented by compiler/runtime intrinsics. Any
such library API must obey this chapter.

The model follows the C++20 abstract machine's concepts of memory locations,
sequenced-before, synchronization, happens-before, modification order, and data
races, as specialized below. The Dot specification, not emitted C++, controls
source behavior.

## Memory locations

A scalar primitive, object/struct field, array element, optional discriminator
or value, reference/pointer binding, and function-value binding occupies one or
more implementation memory locations. Distinct non-overlapping fields and array
elements are distinct locations unless the target C ABI explicitly combines
storage; Dot has no bit-fields.

An ordinary read observes a value written by an operation that happens before
it, subject to synchronization ordering. An ordinary write modifies its memory
location. Const qualification restricts source access but does not make the
memory atomic or immutable to other aliases.

## Sequencing

Within one thread, the left-to-right evaluation order and statement order
defined by this specification establish sequenced-before relationships.
Destructor calls and ownership-count operations occur at their specified points
in that order.

## Data races

Two operations **conflict** when they access the same memory location, at least
one is a write, and neither is an atomic operation supplied by the runtime or
standard atomic library.

A data race occurs when conflicting operations execute in different threads,
neither happens before the other, and they are not both ordered atomic
operations. A program execution containing a data race has undefined behavior.

Reference counting being atomic does not make object fields, array elements,
array length/capacity, optional contents, or user state thread-safe. Concurrent
reads of stable ordinary state are permitted; any unsynchronized concurrent
write conflicting with another access is a data race.

## Synchronization and atomics

Standard-library mutexes, thread operations, and atomics establish the same
happens-before relationships and memory-order meanings as their specified
C++20-equivalent operations. The library must document which operations are
acquire, release, sequentially consistent, or use another ordering.

The language does not expose C/C++ `volatile`; it is neither a synchronization
mechanism nor a source qualifier. Memory-mapped device access requires a future
or platform-specific intrinsic outside ordinary Dot semantics.

## Atomic ownership accounting

Strong reference counts for objects, logical arrays, and heap-backed strings
are always atomic. There is no version-1 thread-local non-atomic representation.

Copying an owning handle in one thread and publishing that copied handle through
proper synchronization permits another thread to own and release it safely.
The ownership runtime must ensure:

- increments and decrements do not corrupt the count;
- exactly one decrement observes the transition to zero;
- all field/element destruction happens once after that transition; and
- for every owning-handle release ordered before the zero transition in the
  count's atomic modification order, operations sequenced before that release
  happen before the destructor body and automatic field/element destruction.

A backend may implement this with relaxed increments, release decrement
read-modify-write operations, and an acquire operation or fence on the
zero-observing path, or with stronger ordering. It may not weaken the stated
happens-before guarantee.

These guarantees concern ownership bookkeeping only. Publishing an object
handle without synchronization can still race on the handle variable or object
fields.

## Destruction thread

The thread performing the strong-count decrement from one to zero synchronously
runs the object, array-element, or shared-string backing destruction. Dot does
not route destruction to the allocation thread or a special collector thread.

A program requiring destruction on a particular thread must arrange for the
final owning handle to be released there. Destructor access to unsynchronized
shared external state follows ordinary race rules.

## Arrays under concurrency

Aliases of one array share its logical mutable state. Concurrent access is safe
only when all operations are reads and no thread changes length, capacity, or
elements. A mutating member conflicts with reads or writes of any metadata it
changes and with affected element access. Standard synchronization is required.

Atomic ownership counts do not protect iterator or element-reference validity.
An unsynchronized mutation may cause both a data race and invalidated access;
either is sufficient for undefined behavior.

## Exceptions and concurrency

An exception propagates only within its throwing thread. It does not cancel or
join other threads. Uncaught exception termination and panic terminate the
process according to the runtime; other threads are not guaranteed to unwind.

## Backend obligation

A C++ backend must select ownership primitives and memory orders that implement
this chapter. It may not replace atomic reference counts with non-atomic counts
based solely on an optimization that assumes values remain thread-local, unless
it proves no owning handle can become observable by another thread and the
optimization cannot alter any allowed execution.
