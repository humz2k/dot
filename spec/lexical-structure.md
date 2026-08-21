# Lexical Structure

## Grammar notation

Grammar productions use EBNF:

- `name = ... ;` defines a production.
- Quoted text is a literal token.
- `A B` is concatenation.
- `A | B` is a choice.
- `[ A ]` is optional.
- `{ A }` is zero or more repetitions.
- `( A )` groups terms.
- Prose in angle brackets denotes a lexical condition that is inconvenient to
  express in EBNF.

Lexical analysis selects the longest token that can begin at the current byte.
If alternatives have equal length, a reserved keyword takes precedence over an
identifier. Whitespace and comments separate tokens but are otherwise ignored,
except inside literal tokens.

## Source encoding

A Dot source file must be valid UTF-8. The compiler must diagnose the first
invalid UTF-8 sequence. A UTF-8 byte-order mark is permitted only as the first
three bytes of a file and is ignored there. A byte-order mark elsewhere is an
invalid source character.

LF and CRLF are equivalent line endings. A standalone CR is whitespace but does
not increment the logical line number. Source locations are identified by file,
one-based line, and one-based Unicode-scalar column; diagnostics may additionally
report byte offsets.

Unicode is permitted in comments and string contents. Version-1 identifiers are
ASCII-only.

## Whitespace and comments

Whitespace consists of space, horizontal tab, vertical tab, form feed, LF, and
CR. Newlines have no grammatical significance.

`//` begins a line comment extending through, but not including, the next line
ending. `/*` begins a block comment ending at the next `*/`. Block comments do
not nest. End-of-file inside a block comment is a compile error.

`///` begins a documentation comment. A documentation group is a maximal
sequence of documentation comments separated only by whitespace and ordinary
comments; the ordinary comments are ignored. The group must be followed, with
only whitespace and ordinary comments intervening, by zero or more attribute
uses and then one declaration. It attaches to that declaration. Documentation
comments after the first attribute of a declaration prefix are invalid, as is a
group not followed by a declaration. Group lines are joined with newline
separators; ordinary comments never become documentation text.

Documentation text is retained in compile-time reflection metadata after
removing the leading `///` and at most one following space from each line.

## Identifiers

```ebnf
identifier = identifier-start , { identifier-continue } ;
identifier-start = "A"…"Z" | "a"…"z" | "_" ;
identifier-continue = identifier-start | "0"…"9" ;
```

Identifiers are case-sensitive. `$` is not an identifier character and is
reserved for metaprogramming splice syntax. A double underscore has no special
status.

A standalone `_` is a discard only in a binding pattern. Elsewhere it is an
ordinary identifier whose leading underscore gives it the visibility described
in [Program Structure](program-structure.md#visibility).

## Keywords

The following are reserved and cannot be used as identifiers:

```text
alias       attribute   bool        break       byte
catch       comptime    const       constructor continue
destructor  elif        else        emit        enum
export      extern      f32         f64          false
fn          for         i8          i16          i32
i64         i128        if          import       in
namespace   none        null        object       operator
quote       raw         reflect     return       Self
str         struct      switch      throw        trait
true        try         void        while
```

The following are contextual keywords and remain valid identifiers outside the
listed construct:

- `case` and `default` introduce labels in a `switch`.
- `on` and `repeatable` occur in an attribute declaration.

Primitive type names and `Self` are keywords. `type` and reflection metadata
type names are built-in declarations in the compile-time environment rather
than keywords, so a private nested declaration may shadow them under ordinary
lookup rules; doing so is discouraged.

## Punctuation and operators

The lexer recognizes these multi-character tokens before their prefixes:

```text
::  :=  ->  ++  --  +=  -=  *=  /=  %=
==  !=  <=  >=  &&  ||  <<  >>  ${
```

The remaining punctuation tokens are:

```text
{ } ( ) [ ] , ; : . @ # ?
+ - * / % = < > ! ~ & | ^ $
```

`$` alone is reserved and invalid in version 1. Only `${` begins a splice.

The parser may contextually split a `>>` token into two `>` delimiters, or a
`>=` token into `>` followed by `=`, only when the first part is required to
close an already recognized generic-argument list. This permits
`Outer<Inner<T>>` without whitespace. A `>>` inside a value argument remains a
shift when the value-expression grammar consumes it; no other token is split.

## Integer literals

```ebnf
decimal-digit = "0"…"9" ;
hex-digit = decimal-digit | "A"…"F" | "a"…"f" ;
binary-digit = "0" | "1" ;
octal-digit = "0"…"7" ;

decimal-integer = decimal-digit , { decimal-digit | "_" } ;
hex-integer = "0x" , hex-digit , { hex-digit | "_" } ;
binary-integer = "0b" , binary-digit , { binary-digit | "_" } ;
octal-integer = "0o" , octal-digit , { octal-digit | "_" } ;
integer-suffix = "i8" | "i16" | "i32" | "i64" | "i128" ;
integer-literal =
    (decimal-integer | hex-integer | binary-integer | octal-integer),
    [ integer-suffix ] ;
```

Prefix letters are lowercase. `_` must occur strictly between two digits valid
for that radix. A leading zero in a decimal literal does not select octal.

An explicitly suffixed literal has the suffix type and is ill-formed if its
non-negative magnitude is not representable by that type in the literal's
context. A leading `-` is a separate unary operator; the special case allowing
the magnitude of the most-negative signed value is described in
[Expressions](expressions.md#integer-arithmetic).

An unsuffixed integer literal uses its expected signed-integer or `byte` type
when one exists and the magnitude is representable. Contextual literal typing
is not a conversion. Without such an expected type, the compiler selects the
first representable type in `i32`, `i64`, `i128`. Failure to fit `i128` is a
compile error.

## Floating-point literals

```ebnf
decimal-digits = decimal-digit , { decimal-digit | "_" } ;
decimal-exponent = ("e" | "E"), [ "+" | "-" ], decimal-digits ;
floating-suffix = "f32" | "f64" ;
floating-literal =
      decimal-digits, ".", [ decimal-digits ], [ decimal-exponent ],
          [ floating-suffix ]
    | ".", decimal-digits, [ decimal-exponent ], [ floating-suffix ]
    | decimal-digits, decimal-exponent, [ floating-suffix ] ;
```

Digit separators follow the same between-digits rule as decimal integers.
Hexadecimal floating literals are not supported.

An explicitly suffixed literal has the suffix type. An unsuffixed literal uses
an expected `f32` or `f64` type only when rounding the mathematical decimal does
not overflow that format to infinity; otherwise it has type `f64`.
Decimal-to-binary conversion uses round-to-nearest, ties-to-even. An explicitly
suffixed literal, or a defaulted `f64` literal, that overflows produces signed
infinity. Underflow follows IEEE-754 gradual underflow, including rounding to
signed zero. Literal NaN and infinity spellings are not tokens; libraries may
provide named constants.

## Boolean, null, none, and byte literals

`true` and `false` have type `bool`.

`null` is contextually typed as a raw pointer. With no expected raw-pointer
type, `null` may initialize a local whose type is inferred as `*void`. It cannot
initialize an object, array, string, reference, function value, or optional.

`none` requires an expected optional type. It cannot by itself determine the
type of an inferred local.

```ebnf
byte-literal = "b'" , byte-character-or-escape , "'" ;
```

A byte literal must decode to exactly one byte. An unescaped source character is
valid only if its UTF-8 encoding is one byte and it is not `'`, `\`, LF, or CR.
Supported escapes are `\\`, `\'`, `\n`, `\r`, `\t`, `\0`, and `\xNN`.
`\u{...}` is also accepted only when its UTF-8 encoding is exactly one byte.
The literal has type `byte`.

## String literals

An ordinary string begins and ends with `"`. It may not contain an unescaped LF
or CR. Supported escapes are:

| Escape | Bytes appended |
| --- | --- |
| `\\` | `0x5c` |
| `\"` | `0x22` |
| `\n` | `0x0a` |
| `\r` | `0x0d` |
| `\t` | `0x09` |
| `\0` | `0x00` |
| `\xNN` | the byte given by two hexadecimal digits |
| `\u{H...}` | UTF-8 encoding of one Unicode scalar value |

A Unicode escape requires one through six hexadecimal digits and must not name
a surrogate or a value above U+10FFFF. An unknown or incomplete escape is a
compile error.

Raw strings use `r"..."` or a hash-delimited form:

```text
r"literal text"
r#"text containing " quotes"#
r##"text containing "# sequences"##
```

The number of opening and closing hashes must match. Raw contents are the exact
UTF-8 bytes between delimiters and may contain newlines. The implementation must
support at least 255 delimiter hashes and may diagnose a documented larger
limit.

Every string literal has type `str`. Source characters and Unicode escapes are
UTF-8 encoded, but runtime strings may later contain arbitrary bytes. Adjacent
string literals do not concatenate, and version 1 has no interpolation syntax.

## Tokenization examples

```dot
value := 0xff_i32;       // invalid: separator is not between digits
value := 0xffi32;        // valid i32 literal
fraction := 1.e3f64;     // valid f64 literal
bytes : byte[] = [b'a', b'\0', b'\xff'];
path := r#"C:\raw\"path"#;
```

The first declaration is ill-formed. Comments in this section are informative;
the token and typing rules above are normative.
