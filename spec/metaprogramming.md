# Compile-time Programming and Reflection

## Overview

Dot compile-time programming uses the ordinary expression and statement model
over a restricted set of immutable/compiler-managed values. It provides:

- `comptime fn` for reusable compile-time computation;
- implicitly compile-time trait bodies and attribute expressions;
- `reflect(Type)` for typed declaration metadata;
- typed `quote` expressions for parsed syntax values;
- `${...}` for category-checked splicing; and
- `emit` for explicit declaration insertion.

There is no textual preprocessor. Generated source text is never reparsed from a
string; generation operates on typed syntax values.

## Compile-time values

The compile-time evaluator supports:

- `bool`, signed integers, `byte`, `f32`, `f64`, `str`, and enum values;
- tuples and arrays recursively containing compile-time-supported values;
- the built-in meta-type `type`;
- reflection metadata values listed below;
- `identifier` and typed syntax values;
- typed applied-attribute values; and
- optionals recursively containing compile-time-supported non-owning values.

Compile-time evaluation does not support runtime object handles, optional
owning values containing objects, references, raw pointers, runtime function
values, exceptions, panic, threads, atomics, or destructors. A compile-time
array is compiler-managed and may be mutated by compile-time code without a
runtime allocation or reference count.

When their element type is compile-time-supported, compile-time arrays expose
all constructors, members, equality, and iteration specified for runtime arrays
in [Built-in Runtime Types](builtins.md#arrays). Element-copy, ordering, bounds,
and invalidation semantics are the same. Managed allocation and atomic
ownership effects do not occur; exhaustion of a compile-time resource limit is
a compile error rather than `allocation_error`. An operation that would be
runtime undefined behavior is likewise a compile error.

`type` values are equal exactly when their represented runtime types are
equivalent after alias expansion. They have no ordering.

In a compile-time context whose expected type is the meta-type `type`, a runtime
type designator is a primary compile-time value. Thus `derive(Self)`,
`record_type(i32)`, and an attribute argument expecting `type` pass type values;
they do not construct runtime values. A generic type parameter likewise denotes
its substituted type value when used as a compile-time expression. A type
designator in an ordinary runtime value context remains ill-formed.

## Compile-time expressions

A compile-time expression is an expression evaluated by the compiler in one of
these contexts:

- namespace constant initialization;
- a generic value argument;
- a constraint or trait body;
- an attribute argument or default;
- a `comptime fn` body;
- a `comptime` declaration block;
- enum explicit values; or
- syntax quotation splices.

All ordinary arithmetic, comparison, string, tuple, and compile-time array
operations retain their Dot semantics. If runtime evaluation would have
undefined behavior, compile-time evaluation must instead issue a compile error
at the operation. Throwing, `try`, `catch`, and `panic` are invalid in a
compile-time context.

Only `comptime fn` may be called by compile-time code. An ordinary runtime
function is never implicitly interpreted, even if its body appears pure.

## Compile-time functions

```dot
comptime fn make_name(prefix : str, suffix : str) -> identifier {
    return identifier@from(prefix + suffix);
}
```

A `comptime fn` follows ordinary function, generic, default-parameter,
constraint, overload, and return rules, with these additions:

- every parameter and return type must support compile-time values;
- its body is evaluated only during compilation;
- it cannot be converted to a runtime function value;
- it cannot declare or call runtime operations; and
- it cannot contain `emit`, because it has no lexical emission scope.

A `comptime fn` is declared at namespace scope. It is not a runtime static
method and cannot be declared as a type member.

Compile-time recursion is permitted within implementation resource limits.
Evaluation is deterministic for a fixed compiler version, language version,
source graph, target metadata, and explicit build inputs.

## Compile-time blocks

```dot
struct Example {
    value : i32;

    comptime {
        emit derive_equality(Self);
    }
}
```

A `comptime { ... }` block may occur directly in a namespace, struct, object, or
enum body. It executes during declaration generation and may declare
compile-time locals, branch, loop, call compile-time functions, inspect
reflection, construct syntax, issue diagnostics, and execute `emit`.

It cannot occur in an ordinary runtime function, constructor, destructor,
method, trait body, or generated declaration. It creates a compile-time lexical
scope; its locals do not become declarations in the surrounding scope.

## Build metadata

The compile-time environment exposes immutable values:

```dot
build.target.os   : const str
build.target.arch : const str
build.profile     : const str
```

They are explicit compilation inputs. `os` and `arch` use the canonical names
documented by the implementation and lockfile; `profile` is the selected build
profile, with at least `debug` and `release` supported. Programs may compare
these strings and conditionally emit declarations.

No other environment, command line, path, time, randomness, or process state is
ambiently visible.

## Reflection entry point

`reflect` is a compile-time-only, non-overloadable intrinsic over a declaration
designator. Supported forms are:

```dot
type_data := reflect(T);                    // type_info
namespace_data := reflect(my_namespace);   // namespace_info
functions := reflect(overloaded_name);     // function_info[]
trait_data := reflect(copyable);            // trait_info
attribute_data := reflect(json_name);       // attribute_declaration_info
```

A type operand is a compile-time value of meta-type `type`. Namespace, function,
trait, and attribute operands are resolved declaration designators rather than
runtime expressions. A function operand always returns `function_info[]`; a
non-overloaded name produces one element and an overloaded name produces every
accessible overload in declaration order. Any other value expression is
invalid.

Reflecting an alias as a type reflects its expanded target; alias declarations
are visible through the containing type or namespace's alias list. Locals,
runtime values, parameters, and function bodies are not directly reflectable.

Reflection obeys access control at the invocation location. Inaccessible private
members are omitted rather than returned with a flag. A generator emitted into
a type has that type's member access; a namespace generator has its ordinary
module/namespace access.

Metadata lists preserve declaration order. Generated members appear after all
handwritten members in deterministic emission order once generation has
completed. The phase rules below define what is visible during generation.

Where declarations in one reflected namespace come from different source
files, their canonical source order is the lexicographic tuple `(owner, module,
package-relative source path, byte offset)`. `owner` orders executable sources
first by executable target name, the current workspace library second, and
dependency packages afterward by `(package name, exact version, content hash)`.
An executable's private module uses its target name for the `module` component.
Within one owner, module and path use Unicode code-point order. This order is a
metadata/generation order only and never affects name lookup or overload
resolution.

## Reflection metadata

The following are compiler-provided compile-time types. Their fields are
read-only. Array-valued fields are immutable snapshots; a caller may copy them
into a mutable compile-time array.

### Common enums

```dot
enum type_kind {
    primitive, struct_type, object_type, enum_type, tuple_type,
    array_type, optional_type, reference_type, raw_pointer_type,
    function_type
}

enum visibility {
    public, private
}

enum generic_parameter_kind {
    type_parameter, value_parameter
}

enum self_kind {
    none, mutable_reference, const_reference, enum_value
}

enum attribute_target {
    namespace_target, struct_target, object_target, enum_target,
    enum_value_target, field_target, alias_target, fn_target,
    constructor_target, destructor_target, operator_target,
    parameter_target, trait_target, attribute_declaration_target,
    type_parameter_target, value_parameter_target
}
```

These enum declarations are built into the compile-time environment. Their
enumerators follow normal qualification rules.

### `type_info`

`type_info` provides:

| Field | Type | Meaning |
| --- | --- | --- |
| `type` | `type` | represented type |
| `kind` | `type_kind` | constructed/nominal kind |
| `is_const` | `bool` | whether the outermost type construction is const-qualified |
| `unqualified_type` | `type` | `type` with only its outermost const qualification removed |
| `name` | `str` | unqualified nominal name, empty for anonymous constructed types |
| `qualified_name` | `str` | canonical source name, empty when no source spelling exists |
| `documentation` | `str` | joined documentation text or empty string |
| `visibility` | `visibility` | nominal declaration visibility; public for constructed types |
| `generic_arguments` | `generic_argument_info[]` | normalized arguments |
| `fields` | `field_info[]` | accessible fields in declaration order |
| `methods` | `method_info[]` | accessible ordinary methods in declaration order |
| `constructors` | `constructor_info[]` | accessible constructors in declaration order |
| `operators` | `operator_info[]` | accessible operator methods in declaration order |
| `destructor` | `destructor_info?` | accessible destructor metadata, if declared |
| `nested_types` | `type_info[]` | accessible directly nested nominal types |
| `aliases` | `alias_info[]` | accessible direct aliases |
| `enumerators` | `enum_value_info[]` | enum entries or empty list |
| `enum_underlying_type` | `type?` | present exactly for an enum |
| `tuple_element_types` | `type[]` | ordered tuple elements; empty for other kinds and for `()` |
| `array_element_type` | `type?` | present exactly for an array |
| `optional_value_type` | `type?` | present exactly for an optional |
| `referenced_type` | `type?` | present exactly for a reference, excluding the reference's outer const access qualification |
| `pointee_type` | `type?` | present exactly for a raw pointer and preserving pointee const qualification |
| `function_parameter_types` | `type[]` | ordered parameters for a function type; empty for other kinds and zero-parameter functions |
| `function_return_type` | `type?` | present exactly for a function type |
| `function_abi` | `str?` | `"Dot"` or `"C"`, present exactly for a function type |

For `const &T`, `is_const` is true and `referenced_type` is `T`. For
`*const T`, `is_const` is false and `pointee_type` is `const T`; for `const *T`,
`is_const` is true and `pointee_type` is `T`. `generic_arguments` describes only
arguments to a generic nominal declaration and is empty for constructed types;
the dedicated decomposition fields above describe constructed types.

Primitive and constructed built-in types report all language-provided
constructors, methods, and operators that have ordinary callable signatures.
Intrinsic array element-place indexing is described by `kind` and
`array_element_type` rather than a synthetic `operator_info`; generated code may
still emit ordinary indexing syntax. Built-in types' `fields` and nested
declaration lists are empty unless this specification explicitly defines one.

### Declaration metadata

`namespace_info` contains:

```text
name : str
qualified_name : str
visibility : visibility
documentation : str
namespaces : namespace_info[]
types : type_info[]
functions : function_info[]
constants : constant_info[]
aliases : alias_info[]
traits : trait_info[]
attribute_declarations : attribute_declaration_info[]
```

Every list contains accessible direct declarations only and preserves source
declaration order. Namespace reopenings are merged into one metadata value;
their declarations use the canonical source order above.
Documentation from reopenings is concatenated in that order with newline
separators. Applied attributes from all reopenings are likewise combined in
that order; applying one non-repeatable attribute to more than one reopening of
the same namespace is a duplicate-attribute compile error.

`function_info` contains:

```text
name : str
parameters : parameter_info[]
return_type : type
generic_parameters : generic_parameter_info[]
constraint : expression?
visibility : visibility
documentation : str
is_comptime : bool
abi : str                           // "Dot" or "C"
```

Its parameter list contains no receiver. `constant_info` contains:

```text
name : str
type : type
visibility : visibility
documentation : str
fn value() -> <the type named by type>
```

`value()` is a compiler dependent-return-type intrinsic and returns the
constant's compile-time value. It is not an ordinary method that a program can
declare.

`field_info` contains:

```text
name : str
type : type
index : i64
visibility : visibility
documentation : str
```

`parameter_info` contains:

```text
name : str
type : type
index : i64
has_default : bool
default_expression : expression?
```

The default syntax value is present only when one was written and is resolved
in the declaration context. It cannot be executed by reflection automatically.

`generic_parameter_info` contains:

```text
name : str
kind : generic_parameter_kind
index : i64
value_type : type?
```

`value_type` is present only for value parameters.

`method_info` contains:

```text
name : str
parameters : parameter_info[]       // includes self at index zero
return_type : type
self_kind : self_kind
generic_parameters : generic_parameter_info[]
constraint : expression?
visibility : visibility
documentation : str
is_comptime : bool
abi : str                           // "Dot" for ordinary methods
```

`constructor_info` contains `name`, `parameters`, `return_type`,
`generic_parameters`, `constraint`, `visibility`, and `documentation` with the
same types as the corresponding `method_info` fields. Parameters do not include
`self`; `return_type` is the containing nominal type.

`operator_info` contains every `method_info` field plus `symbol : str`.

`destructor_info` contains `self_parameter : parameter_info`, `visibility :
visibility`, and `documentation : str`. It has no name, return type, generic
parameters, constraint, or ABI field.

`alias_info` contains `name : str`, `target_type : type`, `generic_parameters :
generic_parameter_info[]`, `constraint : expression?`, `visibility :
visibility`, and `documentation : str`.

`trait_info` contains `name : str`, `generic_parameters :
generic_parameter_info[]`, `constraint : expression?`, `visibility :
visibility`, and `documentation : str`. `constraint` is the optional
prerequisite constraint syntax. Trait body syntax is not exposed.

`attribute_declaration_info` contains `name : str`, `parameters :
parameter_info[]`, `repeatable : bool`, `targets : attribute_target[]`,
`visibility : visibility`, and `documentation : str`.

`enum_value_info` contains:

```text
name : str
index : i64
underlying_value : i128
visibility : visibility
documentation : str
```

All version-1 enum underlying values fit `i128`, including the `byte` range.

`generic_argument_info` contains:

```text
kind : generic_parameter_kind
type_value : type?
value_type : type?
fn value() -> <the type named by value_type>
```

For a type argument, `type_value` is present, `value_type` is absent, and
calling `value()` is a compile error. For a value argument, `type_value` is
absent, `value_type` is present, and the dependent-return `value()` intrinsic
returns the normalized compile-time argument value.

### Attribute access on metadata

Every declaration metadata value, including `type_info`, provides dependent
intrinsics:

```dot
info.has_attribute(attribute_declaration) -> bool
info.attribute(attribute_declaration) -> AppliedAttributeType
info.attributes(attribute_declaration) -> AppliedAttributeType[]
```

The argument is the resolved attribute declaration, not a string. For a
non-repeatable attribute, `attribute` returns its typed immutable applied value
and is a compile-time error when absent. For a repeatable attribute,
`attributes` returns all applications in source order. `attributes` also works
for non-repeatable attributes and returns zero or one value. Calling singular
`attribute` for a repeatable declaration is a compile error.

Constructed and primitive `type_info` values with no source declaration have no
applied attributes: `has_attribute` is false and `attributes` is empty for every
attribute; singular `attribute` therefore fails as absent.

The applied value has read-only fields named after the attribute parameters with
their declared types. There is no dynamic string-key attribute map in version 1.

## Attribute declarations

Libraries declare typed attribute schemas:

```dot
attribute json_name(value : str) on field;
attribute json_skip() on field;
attribute range(min : i64, max : i64) on field, parameter;
attribute route(
    method : str,
    path : str,
    authenticated : bool = false
) on fn;
attribute tag(value : str) repeatable on struct, object, field;
```

The grammar is:

```text
attribute Name(parameters) [repeatable] on target [, target ...] ;
```

An attribute cannot be generic. Its parameters follow ordinary positional,
default-order, and type rules, but every type must be one of:

- primitive except `void`;
- `str` or enum;
- meta-type `type`; or
- tuple/array recursively containing allowed attribute types.

References, raw pointers, objects, optionals containing disallowed values,
function values, and arbitrary runtime structs are invalid attribute parameter
types. Defaults must be compile-time expressions.

Attribute defaults and applied arguments are deliberately phase-independent:
their evaluation may not execute `reflect`, `quote`, `emit`, `compile_error`, or
`compile_warning`, including transitively through a `comptime fn`. They may use
literals, types, namespace constants, build metadata, arithmetic/logic,
tuples/arrays, and compile-time functions that remain within that subset.

The exact target vocabulary is:

```text
namespace, struct, object, enum, enum_value, field, alias,
fn, constructor, destructor, operator, parameter,
trait, attribute, type_parameter, value_parameter
```

Each target may appear once. An empty target list is invalid. `fn` includes
ordinary and `comptime fn`; ABI functions remain functions. Nested nominal types
use their ordinary `struct`, `object`, or `enum` target.

## Attribute uses

```dot
#json_name("user_name")
name : str;

#route("GET", "/users", authenticated=true)
fn users() -> Response { ... }

#json_skip
cached_hash : i64;
```

`#name` is shorthand for `#name()`. Attribute names use ordinary lookup and may
be namespace-qualified after `#`. Arguments may be positional or named;
positional arguments must precede named arguments. Each parameter may be bound
once. Explicit arguments are evaluated in written order. Defaults then fill
omitted parameters and are evaluated in parameter order, with earlier bound
parameters available exactly as for an ordinary call.

Every argument is evaluated at compile time and must exactly match its parameter
type after contextual literal typing. The compiler verifies
that the declaration kind is an allowed target.

A non-repeatable attribute may occur at most once on one declaration. A
`repeatable` attribute may occur multiple times; source order is retained.
Stacked attributes all attach to the immediately following declaration.

Parameter and generic-parameter attributes appear immediately before the
parameter within its comma-separated list. Enum-value attributes appear before
the enumerator. Attributes on a namespace appear before its declaration and
apply to that declaration/reopening only, not automatically to contained
declarations.

Attributes are compile-time-only passive metadata. They do not automatically
execute a generator and have no runtime representation unless generated code
embeds their values.

The compiler predeclares these root attributes:

```dot
attribute repr(value : str) on struct, enum;
attribute symbol(value : str) on fn;
```

Only `#repr("C")` and the C-interoperability uses of `#symbol` have built-in
semantics in version 1. Other values for `repr` are compile errors.

## Identifiers and syntax values

`identifier` is a compile-time type representing one valid non-keyword Dot
identifier. It provides:

```dot
identifier@from(value : str) -> identifier
fn text(self : const &Self) -> str
```

Construction fails with a compile error when the string does not satisfy the
identifier grammar or is reserved. It does not perform name lookup.

Typed immutable syntax values are:

```text
declaration
expression
statement
type_syntax
field_initializer
```

They contain parsed Dot syntax plus hygienic binding and source-location
metadata. They cannot be converted to or from arbitrary strings.

## Quotation

Quotation names its result category:

```dot
quote declaration { fn generated() -> void {} }
quote expression { left + right }
quote statement { return value; }
quote type { Result<i32> }
quote field_initializer { field = value }
```

The braces must contain exactly one node of the named category after singleton
splices. A declaration quote cannot contain an import or a `comptime` block. A
statement quote includes its terminating semicolon where the quoted statement
requires one.

Quotation is parsed and its non-spliced syntax checked when the `comptime fn` or
block containing it is checked. Complete semantic checking occurs after splices
and emission.

## Splicing

`${compile_time_expression}` inserts a compile-time value into a quote. The
expression is evaluated when the quote is constructed. Accepted combinations
are:

| Quote position | Accepted splice value |
| --- | --- |
| identifier position | `identifier` |
| type position | `type` or `type_syntax` |
| expression position | `expression` or an allowed primitive/string/enum/type literal value |
| statement-list position | `statement` or `statement[]` |
| declaration-list position | `declaration` or `declaration[]` |
| field-initializer-list position | `field_initializer` or `field_initializer[]` |

A list splice is valid only where the grammar expects a comma- or
statement-separated list; it inserts elements in array order. An empty list
inserts nothing. A category mismatch is a compile error at the splice.

Splicing a compile-time `str` into an expression creates a string literal with
the exact bytes. Splicing a number, Boolean, byte, or enum creates an exactly
typed literal/qualified value. Splicing `type` never creates a runtime type
value; it inserts type syntax.

## Hygiene and name resolution

Quotation follows these deterministic rules:

- A local, parameter, generic parameter, or nested declaration introduced
  inside a quote receives a fresh hygienic identity. References to the same
  written binding inside that quote refer to that identity.
- A literal name referring to an existing declaration is resolved in the
  quotation's definition context and records that declaration identity.
- A literal existing name that cannot be resolved at quote definition is a
  compile error; emission does not perform accidental capture.
- `Self` is the one exception: it resolves to the nominal type into which the
  declaration is emitted.
- A spliced `identifier` deliberately performs ordinary lookup in the emission
  context, or declares the written name when placed in a declaration position.
- Names contained in a spliced syntax value retain that syntax value's hygiene
  unless the value itself was built with spliced identifiers.

Generated top-level/member declaration names are not automatically renamed.
They enter the target scope with their written/spliced spelling, and collisions
are ordinary compile errors.

## Emission

Inside a `comptime` block:

```dot
emit declaration_value;
emit declaration_array;
```

`emit` accepts `declaration` or `declaration[]` and inserts into the immediately
containing namespace or nominal type. It returns `void` and is not an ordinary
function.

Namespace emission may add namespace constants, functions, aliases, nominal
types, traits, attributes, extern declarations, nested namespace declarations,
and further declarations valid at namespace scope. Type emission may add
methods, operators, constructors, aliases, nested types, traits, and attributes
valid for that type.

Version 1 forbids emission of:

- storage fields or enum enumerators;
- destructors;
- imports or export-imports;
- another `comptime` block; or
- declarations whose syntax category is invalid for the target scope.

An emitted constructor cannot change the already frozen field set. Generated
code follows ordinary visibility, overload, type, const, and privacy rules.

## Worked generation example

The following illustrates the required ability to derive serialization for a
struct. `JsonWriter`, `JsonReader`, and the serialization trait/functions are
library declarations rather than language built-ins.

```dot
comptime fn derive_json(Target : type) -> declaration[] {
    write_fields : statement[] = [];
    read_fields : field_initializer[] = [];

    for (field in reflect(Target).fields) {
        if (!json_serializable<field.type>) {
            compile_error(
                reflect(Target).qualified_name + "." + field.name
                + " is not JSON serializable"
            );
        }

        field_name := identifier@from(field.name);

        write_fields.push(quote statement {
            writer.write_field(
                ${field.name},
                self.${field_name}
            );
        });

        read_fields.push(quote field_initializer {
            ${field_name} = deserialize_json<${field.type}>(reader)
        });
    }

    return [
        quote declaration {
            fn write_json(
                self : const &Self,
                writer : &JsonWriter
            ) -> void {
                writer.begin_object();
                ${write_fields}
                writer.end_object();
            }
        },
        quote declaration {
            constructor from_json(reader : &JsonReader) -> Self {
                return Self {
                    ${read_fields}
                };
            }
        },
    ];
}

struct User {
    #json_name("user_name")
    name : str;
    age : i32;

    comptime {
        emit derive_json(Self);
    }
}
```

This example relies on category checking as follows: `field.name` splices as a
string expression, `field_name` as an identifier, `field.type` as a type, the
statement array into a statement list, and the initializer array into a
`Self` field-initializer list. Generated methods see `Self` as `User`.

Automatic serialization of object graphs is a library policy because shared
identity and owning cycles require an encoding policy. A generator may reject
objects with `compile_error`, delegate to user-provided serialization, or emit
an identity-table algorithm; the language does not choose one.

## Generation phases

For each program, compilation proceeds as if in these phases:

1. Parse every source file and build handwritten declaration shells.
2. Resolve imports, namespaces, non-dependent names, and nominal identities.
3. Resolve field types, generic signatures, attribute declarations, and
   callable signatures.
4. Evaluate and attach declaration attributes.
5. Freeze every struct/object field list and enum enumerator list.
6. Complete generation of imported dependency declarations.
7. Execute `comptime` blocks over frozen snapshots and collect emissions.
8. Install all emissions into their target scopes, diagnose collisions, and
   make generated declarations visible together.
9. Type-check completed handwritten and generated bodies and instantiate
   on-demand generics/traits.
10. Lower the completed program to the backend.

All generators targeting one type or namespace observe the same frozen
handwritten snapshot and cannot observe emissions from another generator in
that target. They execute in canonical source order only to make
diagnostic/emission order deterministic; changing that order cannot make one
generator's emissions visible to another.

Reflecting `Self` while generating `Self` returns the frozen handwritten
snapshot. Reflecting another type requires that type's generation to complete
first. If completing `A` requires completed reflection of `B` and completing
`B` requires completed reflection of `A`, the compiler diagnoses a generation
phase cycle. Imported packages are completed before their users.

After installation, ordinary later type checking and trait evaluation see
generated declarations. Metadata reports generated declarations after
handwritten declarations in emission order.

## Compile-time diagnostics

The compiler provides:

```dot
compile_error(message : str) -> void
compile_warning(message : str) -> void
```

They are valid only during compile-time evaluation. `compile_error` immediately
fails the current compilation path. `compile_warning` records a warning and
continues. Diagnostics include:

- the call site;
- the active `comptime fn` stack;
- the generator invocation or trait/attribute context; and
- relevant reflected and emitted declaration locations.

A type error in generated code reports both the generated construct and the
quotation/splice that produced it.

For compile-time definite-return analysis, an unconditional `compile_error`
call is terminal in the same way that runtime `panic` is terminal. Subsequent
syntax is still type-checked.

## Determinism, sandbox, and limits

Compile-time code has no ambient filesystem, network, environment-variable,
clock, random, subprocess, or arbitrary host-language access. Future explicit
build inputs require a later language or manifest revision.

The implementation may impose documented limits on instruction count, recursion
depth, allocated compile-time memory, generated declarations, syntax-node count,
and diagnostic count. Exceeding a limit is a compile error, never undefined
behavior. A conforming implementation supports at least the minimum limits in
[Conformance](conformance.md#minimum-implementation-limits).

Reflection exposes source declarations and signatures, not runtime values,
function bodies, backend C++ ASTs, addresses, size, alignment, padding, or
private declarations inaccessible at the invocation site.
