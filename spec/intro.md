# Dot Language Specification, Version 1

## Status and scope

This document set is the normative specification of version 1 of the Dot
programming language. It defines the source language, compile-time language,
module and package model, required runtime behavior, and C interoperability
surface. A conforming implementation may use any backend. The initial compiler
is expected to generate C++20, but generated C++ is not the definition of Dot
semantics.

The specification intentionally leaves the broad standard-library API outside
its scope. It does define the built-in values and operations that a compiler
must provide, including arrays, strings, optionals, ownership accounting,
exceptions, reflection metadata, and the package manifest consumed by the build
tool.

## Normative language

The words **must**, **must not**, **required**, **shall**, and **shall not** state
requirements. **May** grants permission. **Undefined behavior** means that this
specification places no requirements on the execution after the undefined
operation is reached. **Implementation-defined** behavior must be documented by
the implementation. An **ill-formed** program must receive a diagnostic and
must not be executed as a conforming Dot program.

Code examples use the `.dot` source-file extension. Grammar fragments use the
EBNF notation defined in [Lexical Structure](lexical-structure.md#grammar-notation).

## Design summary

Dot has two user-defined storage models:

- A `struct` is a structurally copied value. Copying cannot be customized.
- An `object` is a non-null, atomically reference-counted owning handle. Copying
  the value copies the handle, not the allocation.

The language has no move operation, copy constructor, inheritance, implicit
numeric conversion, general null object value, borrow checker, or overloadable
assignment. `&T` is a non-null non-owning reference, `*T` is a nullable raw
pointer, and invalid non-owning use is undefined behavior. Destructors provide
deterministic cleanup; object destruction occurs when the final owning handle is
released.

Traits are compile-time predicates. Metaprogramming uses deterministic Dot code,
typed reflection, typed syntax quotation, splicing, and explicit declaration
emission.

## Table of contents

1. [Lexical Structure](lexical-structure.md)
   - Source encoding, tokens, comments, identifiers, keywords, and literals.
2. [Program Structure and Name Resolution](program-structure.md)
   - Files, modules, imports, namespaces, visibility, scopes, and lookup.
3. [Type System](type-system.md)
   - Primitive, composite, function, reference, pointer, optional, const, and
     nominal types; type equivalence and conversions.
4. [Declarations](declarations.md)
   - Variables, constants, aliases, structs, objects, enums, fields,
     constructors, destructors, functions, methods, operators, and attributes.
5. [Values, Ownership, and Lifetime](values-lifetime.md)
   - Structural copying, reference counting, replacement assignment,
     temporaries, destruction, identity, and dangling access.
6. [Expressions and Operators](expressions.md)
   - Evaluation order, calls, construction, member and index access, arithmetic,
     comparison, assignment, and operator overloading.
7. [Statements and Control Flow](statements.md)
   - Blocks, declarations, conditionals, loops, iterator lowering, switches,
     returns, exceptions, and panic.
8. [Built-in Runtime Types and Operations](builtins.md)
   - Strings, arrays, optionals, function values, identity, and standard runtime
     exception objects.
9. [Generics and Traits](generics-traits.md)
   - Generic parameters, inference, instantiation, constraints, trait
     evaluation, and recursion.
10. [Compile-time Programming and Reflection](metaprogramming.md)
    - Compile-time evaluation, reflection metadata, attributes, quotation,
      hygiene, emission, and compilation phases.
11. [Concurrency and Memory Model](concurrency.md)
    - Data races, happens-before, atomic ownership counts, and destruction
      threads.
12. [C ABI Interoperability](c-interop.md)
    - `extern "C"`, C-safe types, layout attributes, callbacks, symbols, and
      exception boundaries.
13. [Packages, Manifests, and Builds](packages-builds.md)
    - Libraries, modules, executables, `dot.yaml`, dependency resolution,
      lockfiles, registries, targets, and build commands.
14. [Syntactic Grammar](grammar.md)
    - Consolidated version-1 source grammar.
15. [Diagnostics, Undefined Behavior, and Conformance](conformance.md)
    - Required diagnostics, resource limits, fatal behavior, undefined behavior,
      implementation-defined behavior, and backend conformance.

## Normative precedence

The chapter describing a construct's semantics takes precedence over an
informative example. The consolidated grammar controls whether token sequences
are syntactically valid; the semantic chapters may impose additional
well-formedness constraints. If two semantic passages appear inconsistent, the
more specific passage controls. Such an inconsistency is a specification defect
and must not be resolved by silently adopting backend C++ behavior.

## Versioning

This specification is selected by `language: 1` in `dot.yaml`. A compiler that
does not implement language version 1 must reject that manifest. Additions or
changes that alter accepted programs or observable behavior require another
language version unless this specification explicitly marks the matter as
implementation-defined.

## Explicitly absent features

Version 1 has no:

- inheritance, virtual dispatch, runtime interfaces, or downcasts;
- move operations, user-defined copying, or overloadable assignment;
- weak object references or automatic cycle collection;
- general unsigned integers other than `byte`;
- character type, fixed-size array, stack array, union, or payload enum;
- implicit conversion other than adding const qualification;
- pattern matching, tuple assignment destructuring, labeled control flow,
  ternary expression, or `if` expression;
- capturing closure, bound-method value, runtime reflection, or C variadic call;
- mutable runtime global, local static, `volatile`, pointer arithmetic, or
  general preprocessor;
- automatic procedural attributes, generated storage fields, or compile-time
  ambient I/O;
- protected visibility, friendship, file-private visibility, or import aliases.
