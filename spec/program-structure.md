# Program Structure and Name Resolution

## Programs, packages, modules, and files

A Dot program is one executable target together with the transitive closure of
the library packages it uses. Build-target and package selection are defined in
[Packages, Manifests, and Builds](packages-builds.md).

A **package** is a named, versioned distribution produced from one library. A
package contains one or more modules. A **module** is the unit of source import,
compilation, and top-level private visibility. A module contains one or more
`.dot` files. A **namespace** is only a lexical collection of declarations; it
does not own files, modules, or packages.

All files of a module are parsed and analyzed as a single unit. Module-scope
declaration order, including order across files, does not affect visibility.
The same source file must not belong to two modules of one library.

An executable's source files form one private, non-importable module. Exactly
one of them must collectively define the entry point described in
[Statements](statements.md#program-entry).

## Source-file structure

A source file contains, in order:

1. zero or more import or export-import declarations; then
2. zero or more module-scope declarations.

Comments and documentation comments may appear anywhere tokens may be
separated. An import after a non-import declaration is a compile error.

The import sets from every file in a module are unioned before name resolution.
Consequently, an import written in one file is available to every file of that
module. Repeating an identical import anywhere in the module is idempotent and
does not require a warning. If a package-wide import, specific-module import, or
re-export reaches the same declaration through more than one path, that
declaration participates in lookup and export only once.

## Imports

Version 1 accepts these forms:

```dot
import package_name;
import package_name.module_name;
import .module_name;

export import package_name;
export import package_name.module_name;
export import .module_name;
```

Package and module names are identifiers. A module name is one identifier and
cannot contain a dot. In `package.module`, the single dot separates a package
from one module.

`import package;` imports every public module of that package. A module whose
name begins with `_` is private to its library and is excluded. Importing a
specific external module requires that module to be public. `.module` imports a
module from the current library and may name a private module.

An ordinary import makes the imported public declarations available to the
current module. It does not make them available to modules that import the
current module. `export import` performs the ordinary import and additionally
includes the imported public declarations in the current module's exported
surface. Private declarations are always filtered out. `export import
._private_module;` is permitted: the private module remains unavailable for
direct external import, but its non-private declarations are exposed through
the exporting public module. Because imports never bind module names, this does
not expose the private module itself.

An external package may be imported only if the current module opted into that
package in `dot.yaml`. A relative module import needs no dependency entry.
Transitive packages are not directly importable unless the current module opts
into them, but declarations explicitly received through a re-export are usable
without separately importing their original module.

Package and module import graphs must be acyclic. A self-import is a cycle.
Cycles are diagnosed with at least one path forming the cycle.

Imports do not create namespaces and do not bind a qualifier named after the
package or module. Imported declarations retain the namespaces declared by
their source. Version 1 has no import alias or selective import.

## Namespaces

```dot
namespace graphics {
    namespace detail::encoding {
        // declarations
    }
}
```

`namespace a::b::c { ... }` is equivalent to lexically nested `namespace`
declarations. A namespace can be reopened in any file or module. Public
namespace declarations with the same fully qualified name denote the same
namespace throughout the program.

A namespace whose first component begins with `_` and is declared at module
scope is module-private. Private namespaces with the same spelling in different
modules are distinct. Every declaration lexically inside a private namespace is
unavailable outside that module even if its own name lacks an underscore.

Namespaces have no runtime value and cannot be passed or stored.

## Declaration name space

Within a scope, types, aliases, values, namespaces, functions, traits,
attributes, and compile-time declarations occupy one declaration name space.
Functions and constructors are the only declarations that form overload sets.
An alias is a declaration, not a second namespace entry for its target.

Two public namespace functions with the same fully qualified name combine into
one overload set when their effective signatures differ. Identical effective
signatures are a conflict. Any duplicate non-overloadable public name is a
compile error. These rules apply across imported packages and reopened
namespaces; source or import order never selects a winner.

Nested members of different nominal types occupy different member scopes.

Compiler-provided types, exception objects, intrinsics, attributes, metadata
types, and the `build` namespace occupy ordinary public root-namespace names.
A program declaration colliding with one is a duplicate declaration error even
when the spelling is not a reserved keyword.

## Scopes

Every braced block creates a lexical scope. A declaration is visible as follows:

- A module-scope declaration is visible throughout its module regardless of
  textual order.
- A type member is visible throughout the complete type body regardless of
  textual order.
- A local variable is visible after its declarator and initializer have
  completed, through the end of its lexical scope.
- A function parameter is visible after its declaration through the end of the
  function body. Earlier parameters are visible in later default expressions.
- A generic parameter is visible after its declaration through the constrained
  declaration.
- A catch binding is visible only in its catch block.
- A loop initializer or iteration binding has the scope specified in
  [Statements](statements.md#loops).

A local may shadow a declaration from an outer scope. A name may not be declared
twice in one lexical scope except as members of a valid overload set. The name
being declared is not in scope within its own initializer.

## Unqualified lookup

For a name used without `::`, lookup examines these levels in order:

1. the innermost lexical block, then successive enclosing lexical blocks;
2. the containing type member scope, if the use is within a type member;
3. the innermost containing namespace;
4. each successive parent namespace;
5. the root namespace.

Lookup stops at the first level containing at least one declaration with the
name. If all declarations found there are functions, they form the candidate
overload set. Otherwise, more than one distinct declaration is an ambiguity and
the program is ill-formed.

An imported declaration participates at its declared namespace level exactly
as a declaration from the current module would. Dot has no argument-dependent
lookup. Namespaces associated with argument types are never searched
automatically.

Within a type member, an unqualified field or method name is shorthand for
member access through `self` when no local declaration was found first and the
member is valid for that use. Explicit `self.member` is always permitted.

## Qualified lookup

`::name` starts lookup in the root namespace. Otherwise, the first component of
`name::child` is resolved by ordinary unqualified lookup. Every later component
is looked up only within the namespace, type, enum, or alias target resolved by
the preceding component; parent scopes are not searched.

Aliases are transparent during qualification. An enum value and nested type
must use qualification, for example `Color::red` and `Container::Iterator`.

An absent or ambiguous component is a compile error. A value cannot be used as
a `::` qualifier.

## Member lookup

For `value.member`, the compiler determines the static type of `value`, removes
reference transparency, and looks only in that type's member scope. Dot has no
inheritance or extension-method lookup. Overloaded methods with the member name
form a candidate set and are resolved as described in
[Declarations](declarations.md#overload-resolution).

Numeric tuple members such as `.0` are handled by the tuple rules rather than
ordinary member lookup. Built-in array, string, and optional members are
specified in [Built-in Runtime Types](builtins.md).

## Visibility

Visibility is determined by spelling:

- A module-scope declaration whose name begins with `_` is private to its
  module.
- A type member whose name begins with `_` is private to its containing type.
- A module whose name begins with `_` is private to its library.
- A declaration inside a private namespace is private to that namespace's
  module.
- Every other declaration is public unless this specification gives it a more
  restrictive rule.

Module-private declarations are visible from every file of their module and
from generated declarations emitted into that module. Type-private members are
visible from member bodies and generated declarations of the containing type.
They are not visible merely because code has a value of that type. Nested types
do not automatically gain access to private members of their containing type;
access is based on the lexical member body in which the use occurs.

A public declaration's externally visible signature must not expose a
module-private declaration or a private member type. This includes parameter,
return, field, generic constraint, alias target, and public attribute types.

There is no protected, friend, or file-private visibility. Private declarations
with identical spelling in different modules or types are distinct and must be
mangled distinctly by a compiler backend.

## `Self`

`Self` is a reserved type name available inside a `struct`, `object`, or `enum`
body and inside declarations generated into that type. It denotes the complete
containing nominal type with its current generic arguments. It is not available
inside a nested type to refer to the outer type; there it denotes the nested
type.

`Self` is not a runtime value. Constructor-specific `Self { ... }` syntax and
method `self` parameters are defined in [Declarations](declarations.md).
