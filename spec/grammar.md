# Syntactic Grammar

## Purpose

This chapter consolidates the version-1 source grammar. Lexical productions and
longest-token rules are normative in [Lexical Structure](lexical-structure.md).
Semantic chapters impose additional type, context, visibility, and phase rules.

EBNF notation is described in
[Lexical Structure](lexical-structure.md#grammar-notation). `IDENTIFIER`,
`INTEGER`, `FLOAT`, `BYTE`, `STRING`, and `RAW_STRING` denote tokens produced by
the lexer. Documentation comments attach before syntactic parsing and are not
shown in every declaration production.

## Source files and imports

```ebnf
source-file = { import-declaration }, { top-level-declaration } ;

import-declaration = [ "export" ], "import", import-path, ";" ;

import-path =
      IDENTIFIER
    | IDENTIFIER, ".", IDENTIFIER
    | ".", IDENTIFIER ;
```

After the first top-level declaration, an import token sequence is invalid.

## Declaration prefixes and attributes

```ebnf
declaration-prefix = { attribute-use } ;

attribute-use = "#", qualified-name,
                [ "(", [ attribute-argument-list ], ")" ] ;

attribute-argument-list = attribute-argument,
                          { ",", attribute-argument }, [ "," ] ;

attribute-argument = expression | IDENTIFIER, "=", expression ;
```

Positional attribute arguments must precede named arguments; this is a semantic
restriction because both are expressions syntactically. When an attribute
argument begins with `IDENTIFIER =` at its outermost level, the parser must take
the named-argument alternative; it is not an assignment expression.

## Top-level declarations

```ebnf
top-level-declaration = declaration-prefix,
    ( namespace-declaration
    | struct-declaration
    | object-declaration
    | enum-declaration
    | trait-declaration
    | attribute-declaration
    | alias-declaration
    | function-declaration
    | comptime-function-declaration
    | extern-declaration
    | namespace-constant-declaration
    | comptime-block ) ;
```

## Namespaces

```ebnf
namespace-declaration = "namespace", namespace-path, "{",
                        { top-level-declaration }, "}" ;

namespace-path = IDENTIFIER, { "::", IDENTIFIER } ;
```

Imports are not permitted inside a namespace body.

## Generic parameters and constraints

```ebnf
generic-parameter-list = "<", generic-parameter,
                         { ",", generic-parameter }, [ "," ], ">" ;

generic-parameter = declaration-prefix,
                    IDENTIFIER, [ ":", type ] ;

constraint-clause = "[", expression, "]" ;

generic-argument-list = "<", generic-argument,
                        { ",", generic-argument }, [ "," ], ">" ;

generic-argument = type | expression ;
```

Whether a generic parameter/argument is a type or value is determined from the
resolved generic declaration. A value parameter is the form with `: type`; a
type parameter omits it.

## Structs and objects

```ebnf
struct-declaration = "struct", IDENTIFIER, [ generic-parameter-list ],
                     [ constraint-clause ], type-body ;

object-declaration = "object", IDENTIFIER, [ generic-parameter-list ],
                     [ constraint-clause ], type-body ;

type-body = "{", { type-member-declaration }, "}" ;

type-member-declaration = declaration-prefix,
    ( field-declaration
    | constructor-declaration
    | destructor-declaration
    | method-declaration
    | operator-declaration
    | alias-declaration
    | struct-declaration
    | object-declaration
    | enum-declaration
    | trait-declaration
    | attribute-declaration
    | comptime-block ) ;

field-declaration = IDENTIFIER, ":", type, ";" ;
```

The semantic rules reject members not allowed by the containing kind.
`comptime fn` is namespace-scope only and therefore is not a type member.

## Enums

```ebnf
enum-declaration = "enum", IDENTIFIER, [ ":", enum-underlying-type ],
                   "{", { enum-item }, "}" ;

enum-underlying-type = "i8" | "i16" | "i32" | "i64" | "i128" | "byte" ;

enum-item = declaration-prefix,
    ( enum-value-declaration | enum-member-declaration ) ;

enum-value-declaration = IDENTIFIER, [ "=", expression ], [ "," ] ;

enum-member-declaration =
      constructor-declaration
    | method-declaration
    | operator-declaration
    | alias-declaration
    | struct-declaration
    | object-declaration
    | enum-declaration
    | trait-declaration
    | attribute-declaration
    | comptime-block ;
```

All enum values must precede all enum members. A comma is required after every
value except the final item when it is immediately followed by `}`. These rules
remove the syntactic overlap between an unqualified enum value and later items.

## Aliases

```ebnf
alias-declaration = "alias", IDENTIFIER, [ generic-parameter-list ],
                    [ constraint-clause ], "=", type, ";" ;
```

## Parameters and callables

```ebnf
parameter-list = "(", [ parameter, { ",", parameter }, [ "," ] ], ")" ;

parameter = declaration-prefix, IDENTIFIER, ":", type,
            [ "=", expression ] ;

function-declaration = "fn", IDENTIFIER, [ generic-parameter-list ],
                       parameter-list, "->", type,
                       [ constraint-clause ], function-body ;

comptime-function-declaration = "comptime", "fn", IDENTIFIER,
                                [ generic-parameter-list ], parameter-list,
                                "->", type, [ constraint-clause ], block ;

method-declaration = "fn", IDENTIFIER, [ generic-parameter-list ],
                     parameter-list, "->", type,
                     [ constraint-clause ], block ;

constructor-declaration = "constructor", IDENTIFIER,
                          [ generic-parameter-list ], parameter-list,
                          "->", "Self", [ constraint-clause ], block ;

destructor-declaration = "destructor", parameter-list, block ;

operator-declaration = "operator", overloadable-operator,
                       [ generic-parameter-list ], parameter-list,
                       "->", type, [ constraint-clause ], block ;

overloadable-operator =
      "+" | "-" | "*" | "/" | "%" | "!"
    | "==" | "<" | "<=" | ">" | ">=" | "[" , "]" ;

function-body = block ;
```

A bodyless function is accepted only by `extern-function-declaration` below.
Operator arity and containing-type restrictions are semantic.

## Traits

```ebnf
trait-declaration = "trait", IDENTIFIER, [ generic-parameter-list ],
                    [ constraint-clause ], block ;
```

The block is parsed as ordinary statements and then restricted to compile-time
trait semantics.

## Attribute declarations

```ebnf
attribute-declaration = "attribute", IDENTIFIER, parameter-list,
                        [ "repeatable" ], "on", attribute-target,
                        { ",", attribute-target }, ";" ;

attribute-target =
      "namespace" | "struct" | "object" | "enum" | "enum_value"
    | "field" | "alias" | "fn" | "constructor" | "destructor"
    | "operator" | "parameter" | "trait" | "attribute"
    | "type_parameter" | "value_parameter" ;
```

Several target words are contextual tokens in this production. Attribute
declaration parameters follow the semantic restrictions in
[Metaprogramming](metaprogramming.md#attribute-declarations).

## Namespace constants and compile-time blocks

```ebnf
namespace-constant-declaration = IDENTIFIER, ":", "const", type,
                                 "=", expression, ";" ;

comptime-block = "comptime", block ;
```

The written `type` after `const` must not itself begin with `const`.

## Extern declarations

```ebnf
extern-declaration = "extern", STRING,
    ( extern-function-declaration | extern-block ) ;

extern-block = "{", { declaration-prefix,
                      extern-function-declaration }, "}" ;

extern-function-declaration = "fn", IDENTIFIER,
                              [ generic-parameter-list ], parameter-list,
                              "->", type, [ constraint-clause ],
                              ( block | ";" ) ;
```

Semantic rules require the string `"C"`, forbid generics/constraints for C ABI,
and require C-safe types.

## Types

```ebnf
type = reference-type | raw-pointer-type | qualified-value-type ;

reference-type = [ "const" ], "&", non-reference-type ;

raw-pointer-type = [ "const" ], "*", [ "const" ], type ;

qualified-value-type = [ "const" ], postfix-type ;

non-reference-type = raw-pointer-type | qualified-value-type ;

postfix-type = primary-type, { "[", "]" | "?" } ;

primary-type =
      primitive-type
    | type-name
    | tuple-type
    | function-type
    | "(", type, ")" ;

primitive-type =
      "bool" | "i8" | "i16" | "i32" | "i64" | "i128"
    | "byte" | "f32" | "f64" | "str" | "void" ;

type-name = [ "::" ], type-name-component,
            { "::", type-name-component } ;

type-name-component = (IDENTIFIER | "Self"), [ generic-argument-list ] ;

tuple-type = "(", [ type, ",", { type, "," }, [ type ] ], ")" ;

function-type = [ "extern", STRING ], "fn", "(",
                [ type, { ",", type }, [ "," ] ], ")", "->", type ;
```

The tuple production requires a comma for one element; `()` is empty and `(T)`
is grouped by the separate parenthesized-type alternative. Duplicate const
qualification and reference-to-reference types are rejected semantically.

Postfix constructors associate left-to-right: `T?[]` and `T[]?` differ.

## Blocks and statements

```ebnf
block = "{", { statement }, "}" ;

statement =
      block
    | empty-statement
    | local-declaration
    | destructuring-declaration
    | expression-statement
    | if-statement
    | while-statement
    | for-statement
    | iterator-for-statement
    | switch-statement
    | break-statement
    | continue-statement
    | return-statement
    | throw-statement
    | try-statement
    | emit-statement ;

empty-statement = ";" ;

local-declaration =
      IDENTIFIER, ":", type, "=", expression, ";"
    | IDENTIFIER, ":=", expression, ";" ;

destructuring-declaration = tuple-binding-pattern, ":=", expression, ";" ;

binding-pattern =
      "_"
    | IDENTIFIER, [ ":", type ]
    | tuple-binding-pattern ;

tuple-binding-pattern = "(", binding-pattern, ",",
                        { binding-pattern, "," },
                        [ binding-pattern ], ")" ;

expression-statement = expression, ";" ;
```

Tuple binding requires at least one comma and at least one non-discard name by
semantic rule.

## Conditionals and loops

```ebnf
if-statement = "if", "(", expression, ")", block,
               { "elif", "(", expression, ")", block },
               [ "else", block ] ;

while-statement = "while", "(", expression, ")", block ;

for-statement = "for", "(", [ for-initializer ], ";",
                [ expression ], ";", [ expression ], ")", block ;

for-initializer =
      IDENTIFIER, ":", type, "=", expression
    | IDENTIFIER, ":=", expression
    | expression ;

iterator-for-statement = "for", "(", IDENTIFIER,
                         [ ":", type ], "in", expression, ")", block ;

break-statement = "break", ";" ;
continue-statement = "continue", ";" ;
```

The presence of `in` versus semicolons disambiguates the two `for` forms.

## Switch

```ebnf
switch-statement = "switch", "(", expression, ")", "{",
                   { switch-section }, "}" ;

switch-section = switch-label, { statement } ;

switch-label =
      "case", expression, ":"
    | "default", ":" ;
```

Within a switch parser, contextual `case` and `default` begin the next section
and therefore are not consumed as ordinary identifier expression statements.

## Return, throw, try, and emit

```ebnf
return-statement = "return", [ expression ], ";" ;

throw-statement = "throw", [ expression ], ";" ;

try-statement = "try", block, catch-clause, { catch-clause } ;

catch-clause =
      "catch", "(", IDENTIFIER, ":", type, ")", block
    | "catch", block ;

emit-statement = "emit", expression, ";" ;
```

`emit` is semantically valid only within the lexical execution context of a
compile-time block. It may be nested in that block's `if`, loop, switch, or
nested ordinary block, but not in a called `comptime fn`, trait body, or runtime
body. Typed catch type restrictions and catch-all ordering are semantic.

## Expressions

```ebnf
expression = assignment-expression ;

assignment-expression = logical-or-expression,
    [ assignment-operator, assignment-expression ] ;

assignment-operator = "=" | "+=" | "-=" | "*=" | "/=" | "%=" ;

logical-or-expression = logical-and-expression,
                        { "||", logical-and-expression } ;

logical-and-expression = bitwise-or-expression,
                         { "&&", bitwise-or-expression } ;

bitwise-or-expression = bitwise-xor-expression,
                        { "|", bitwise-xor-expression } ;

bitwise-xor-expression = bitwise-and-expression,
                         { "^", bitwise-and-expression } ;

bitwise-and-expression = equality-expression,
                         { "&", equality-expression } ;

equality-expression = relational-expression,
                      { ("==" | "!="), relational-expression } ;

relational-expression = shift-expression,
                        { ("<" | "<=" | ">" | ">="), shift-expression } ;

shift-expression = additive-expression,
                   { ("<<" | ">>"), additive-expression } ;

additive-expression = multiplicative-expression,
                      { ("+" | "-"), multiplicative-expression } ;

multiplicative-expression = prefix-expression,
                            { ("*" | "/" | "%"), prefix-expression } ;

prefix-expression =
      ("&" | "*" | "!" | "~" | "-" | "++" | "--"), prefix-expression
    | "raw", "&", prefix-expression
    | postfix-expression ;

postfix-expression = primary-expression, { postfix-suffix } ;

postfix-suffix =
      call-suffix
    | generic-argument-list, call-suffix
    | ".", member-name, [ generic-argument-list ], [ call-suffix ]
    | "[", expression, "]"
    | "++" | "--" ;

call-suffix = "(", [ argument-list ], ")" ;

argument-list = expression, { ",", expression }, [ "," ] ;

member-name = IDENTIFIER | INTEGER ;
```

An integer member name must be unsuffixed decimal and is valid only for tuple
access. A member suffix with a call is a method call; without a call it is a
field/tuple access, and methods cannot become bound values.

## Primary expressions

```ebnf
primary-expression =
      literal
    | "null"
    | "none"
    | qualified-name
    | constructor-expression
    | self-initializer
    | parenthesized-or-tuple-expression
    | array-literal
    | lambda-expression
    | reflect-expression
    | quote-expression ;

literal = INTEGER | FLOAT | BYTE | STRING | RAW_STRING | "true" | "false" ;

qualified-name = [ "::" ], name-component,
                 { "::", name-component } ;

name-component = (IDENTIFIER | "Self"), [ generic-argument-list ] ;

constructor-expression = type, "@", IDENTIFIER,
                         [ generic-argument-list ], call-suffix ;

self-initializer = "Self", "{", [ field-initializer-list ], "}" ;

field-initializer-list = field-initializer,
                         { ",", field-initializer }, [ "," ] ;

field-initializer = IDENTIFIER, "=", expression ;

parenthesized-or-tuple-expression = "(",
    [ expression, [ ",", { expression, "," }, [ expression ] ] ], ")" ;

array-literal = "[", [ expression, { ",", expression }, [ "," ] ], "]" ;

lambda-expression = "fn", parameter-list, "->", type, block ;

reflect-expression = "reflect", "(", expression, ")" ;

quote-expression = "quote", quote-category, quote-body ;

quote-category = "declaration" | "expression" | "statement"
               | "type" | "field_initializer" ;

quote-body = "{", <category-parsed token tree with splice holes>, "}" ;

splice = "${", expression, "}" ;
```

`Self { ... }` is semantically restricted to constructors. `reflect` and
`quote` are compile-time-only.

## Generic-angle disambiguation

In a type context, `<...>` following a type-name component is always a generic
argument list. In an expression context, `<` begins a generic argument list only
when all of these hold:

1. it immediately follows a name or member-name component;
2. a balanced `>` can be parsed using generic-argument grammar; and
3. the token after the balanced `>` can legally continue or follow a completed
   postfix expression, or the `>` ends the enclosing expression.

Otherwise `<` is the relational operator. When both parses would satisfy these
syntactic rules, the generic interpretation wins. Whitespace does not affect the
choice. Semantic resolution must then find a compatible generic declaration;
failure does not reinterpret the tokens as relational operators.

This rule parses `trait_name<T> && condition` and a standalone specialized
function value `function_name<T>` as generic names. It parses `a < b > c` as
relational operators because an identifier cannot immediately follow a
completed postfix expression. A parser may retain an ambiguous angle node until
declaration shells identify which generic arguments are types and which are
values.

## Semicolon summary

Semicolons terminate imports, fields, aliases, namespace constants, local and
destructuring declarations, expression statements, `return`, `throw`, `break`,
`continue`, empty statements, attribute declarations, and bodyless extern
functions. They do not follow braced namespace, type, trait, function,
constructor, destructor, operator, control-flow, extern-block, or compile-time
block definitions.
