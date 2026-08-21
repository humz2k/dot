# C ABI Interoperability

## ABI declarations

Version 1 recognizes the ABI string `"C"`:

```dot
extern "C" fn native_add(left : i32, right : i32) -> i32;

extern "C" {
    fn allocate(size : i64) -> *void;
    fn release(value : *void) -> void;
}
```

The ABI string is mandatory. Bare `extern` is invalid. An extern block applies
the ABI to every function declaration directly inside it and cannot contain
namespaces, types, constants, ordinary Dot functions, imports, or another
extern block.

A bodyless ABI function imports an externally defined symbol and must end with
`;`. An ABI function with a body exports a Dot implementation using that calling
convention and linkage.

Only `"C"` is supported. Another string is a compile error under language
version 1.

## Symbols and overloading

By default, an extern function's external symbol is its unqualified Dot function
name encoded as bytes. It may be overridden:

```dot
#symbol("library_native_add")
extern "C" fn add(left : i32, right : i32) -> i32;
```

The symbol string must be non-empty and contain no zero byte. Its remaining
accepted character set is implementation-defined for each target because some
object formats support non-identifier symbol characters.

`#symbol` may annotate only an `extern "C"` function declaration or definition.
Using it on any other declaration is a compile error.

C-linkage functions do not use Dot name mangling and therefore cannot overload
one external symbol. Multiple Dot declarations may share a Dot source name only
when every declaration has a distinct `#symbol` and ordinary Dot overload
resolution can distinguish their source signatures. Two definitions/imports of
one external symbol in a linked program are a link error.

Namespace qualification affects Dot lookup but not the default external symbol.
`#symbol` is required whenever the actual external name differs; distinct symbol
names are required to avoid a collision.

## C-safe primitive types

The following mappings are required on a target accepted by the compiler:

| Dot type | C ABI type |
| --- | --- |
| `i8` | an exact-width signed 8-bit integer compatible with `int8_t` |
| `i16` | exact-width signed 16-bit, compatible with `int16_t` |
| `i32` | exact-width signed 32-bit, compatible with `int32_t` |
| `i64` | exact-width signed 64-bit, compatible with `int64_t` |
| `byte` | unsigned 8-bit, compatible with `uint8_t` |
| `f32` | C `float` when it is the target ABI binary32 type, otherwise the target's exact binary32 ABI type |
| `f64` | C `double` when it is the target ABI binary64 type, otherwise the target's exact binary64 ABI type |
| `void` | C `void`, return position only |
| `*T` | pointer to the corresponding C-safe `T` |
| `*void` | C `void*` |

If the target C ABI lacks one required exact representation, the compiler must
reject an extern declaration using it rather than silently substitute another
width.

`bool` and `i128` are not C-safe in version 1. `str`, arrays, optionals,
references, ordinary structs/enums, objects, tuples, Dot function values,
generic values, and types with Dot destruction are not C-safe.

Const raw-pointer pointee qualification maps to C pointee `const`. A const
pointer binding does not affect ABI and is not meaningful on a by-value
parameter, though it remains checked inside a Dot function body.
Outer const qualification on any by-value C-safe parameter, result, or
C-layout field likewise does not change representation or calling convention;
it remains a Dot access restriction. It never permits removal of pointee const.

## C-layout structs

```dot
#repr("C")
struct NativePoint {
    x : f64;
    y : f64;
}
```

A C-layout struct:

- is still a Dot value-semantic struct;
- lays out fields in declaration order using the target C ABI's size, alignment,
  and padding rules;
- may contain only C-safe primitives, raw pointers to C-safe/incomplete native
  types, C-layout enums, and recursively C-layout structs;
- may declare methods, constructors, nested declarations, and operators because
  they do not affect representation;
- cannot contain a reference, object, string, array, optional, tuple, Dot
  function value, generic-dependent field with unknown C layout, or custom
  destructor; and
- cannot be generic in version 1.

It is a compile error when the target cannot represent the requested layout.
Padding bytes have unspecified values and are not included in Dot structural
semantics. Copying a C-layout struct copies fields semantically; a backend may
copy padding as part of an equivalent operation, but Dot code cannot observe
padding.

The implementation must document size, alignment, field offsets, and the C
declaration that interoperates with each emitted C-layout type for the selected
target.

## C-layout enums

```dot
#repr("C")
enum NativeStatus : i32 {
    ok,
    failed,
}
```

A C-layout enum requires an explicit C-safe signed integer or `byte` underlying
type. Its target ABI representation is the corresponding fixed-width integer
enumeration representation. The compiler must reject a target where a matching
C ABI enum representation cannot be guaranteed.

Receiving an underlying value that matches no declared enumerator creates an
invalid Dot enum value. Returning it from an extern call, loading it through a
C-layout struct, or otherwise importing it into ordinary Dot execution has
undefined behavior at the boundary. Authors needing arbitrary numeric values
must expose the underlying integer in the binding and validate it before a
user-written enum constructor.

Duplicate declared enum values remain permitted.

## Incomplete native pointees

An extern declaration may use a raw pointer to a module-private empty C-layout
struct as an opaque handle:

```dot
#repr("C")
struct _NativeHandle {}

#symbol("native_create")
extern "C" fn _native_create() -> *_NativeHandle;
```

The pointed-to layout need not be accessed by Dot. A C-layout struct with no
fields used only behind a raw pointer is treated as an incomplete native tag for
ABI purposes. It cannot be constructed, dereferenced, have `sizeof` reflected,
or be passed by value. Version 1 has no general incomplete-type declaration
syntax beyond this opaque binding convention.

An extern declaration exposing a private opaque tag must itself be private to
the same module; the ordinary rule against exposing private types in public
signatures still applies. A public binding may instead give the opaque tag a
public non-underscore name.

## C callbacks

```dot
callback : extern "C" fn(i32, *void) -> void = exported_callback;
```

All callback parameter and result types must be C-safe. Only a matching function
declared with `extern "C"` linkage can initialize a C callback value. It may be
an imported bodyless function or an exported Dot function body. Ordinary Dot
functions and lambdas do not convert to C linkage.

Invoking an exported Dot function from C enters Dot runtime execution. The
external caller must obey any runtime thread-registration requirement documented
by the implementation. An exception attempting to escape that boundary causes
immediate runtime termination; it never unwinds through C. An imported C
function value continues to denote the external C function and does not acquire
Dot exception behavior merely by being passed as a value.

C callback values are represented as the target C function-pointer type.
Version 1 does not define a null function value. Bindings needing a nullable
callback must expose `*void` and perform platform-specific validation outside
ordinary function-value semantics.

## Exported function bodies

```dot
extern "C" fn dot_add(left : i32, right : i32) -> i32 {
    return left + right;
}
```

An exported body follows ordinary Dot semantics. The compiler generates a C ABI
boundary that:

- translates C-safe parameters into their identical Dot representations;
- invokes the Dot body;
- translates the result back; and
- catches any escaping Dot exception and terminates.

Panic already terminates without crossing the boundary. Raw pointers received
from C are not automatically validated; invalid dereference remains undefined
behavior.

## Unsupported C features

Version 1 does not provide:

- C variadic declarations or calls;
- automatic C integer promotions at a Dot call site;
- C header parsing or macro import;
- C unions, bit-fields, flexible arrays, or `long double`;
- automatic ownership conventions for allocated pointers;
- pointer arithmetic, integer-pointer casts, or `volatile`; or
- C++ ABI declarations.

Bindings use explicitly written Dot declarations or source generated by an
external binding tool.

## Native link inputs

Libraries, frameworks, search paths, and raw linker options are build-target
properties in `dot.yaml`, not source declaration attributes. `#symbol` selects a
symbol only; it does not add a library. The exact manifest schema is defined in
[Packages and Builds](packages-builds.md#native-link-inputs).

## Backend conformance

Only `extern "C"` and `#repr("C")` expose a stable target ABI. Layout, mangling,
calling convention, ownership representation, and symbols of ordinary Dot
declarations are compiler-private and may change between compiler versions.

A generated-C++ compiler must use target compiler declarations and attributes
that implement these ABI rules. Merely emitting a superficially similar C++
type is insufficient when the target ABI differs.
