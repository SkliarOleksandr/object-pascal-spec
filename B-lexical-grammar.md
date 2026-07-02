# Appendix B — Lexical Structure & Shared Grammar Productions

This appendix is the **lexer specification** plus the canonical **shared
non-terminals** referenced by every chapter. When a chapter writes `Expression`,
`Designator`, `Ident`, `TypeRef`, `ConstExpr`, or `Block`, it means the
production defined here.

EBNF notation is per [README](README.md#ebnf-notation).

---

# Part 1 — Lexical structure (the lexer)

The lexer turns source text into a token stream, discarding whitespace and
comments. Object Pascal is **case-insensitive** for identifiers and reserved
words; the lexer should fold case for matching but preserve the original
spelling on identifier tokens (needed for diagnostics and `NameOf`, 13.0).

## B.1 Source text & encoding

- Source files are read as text. A leading **UTF-8 BOM** is accepted and skipped.
- Line breaks: CRLF, LF, or CR each end a line. Track line/column for tokens.
- The lexer is otherwise byte/codepoint oriented; non-ASCII letters are permitted
  in identifiers per the rules in B.3.

## B.2 Whitespace & comments

Whitespace (space, tab, line breaks) separates tokens and is otherwise
insignificant (**except inside multiline string literals**, B.6.3).

### B.2.1 Comment forms

| | |
|---|---|
| **Introduced** | `{ }` and `(* *)` Pascal (pre-1995); `//` Delphi 2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Three comment syntaxes. `{ }` and `(* *)` are block comments; `//` runs to
end of line. Block comments **do not nest** with the *same* delimiter, but
`{ (* *) }` and `(* { } *)` nest across *different* delimiters.

**Grammar**

```ebnf
Comment      = LineComment | BraceComment | ParenStarComment ;
LineComment  = "//" { ?any char except line-break? } ;
BraceComment = "{" { ?any char except "}"? } "}" ;            (* unless a directive, B.2.2 *)
ParenStarComment = "(*" { ?any char? } "*)" ;
```

**Semantics & parsing notes**

- ⚠️ A `{` immediately followed by `$` is **not** a comment — it is a
  **compiler directive** (B.2.2). The lexer must peek the next char.
- Comments are tokens to *skip*, but a full-fidelity AST may attach them as
  trivia for tooling/formatting.

### B.2.2 Compiler directives

| | |
|---|---|
| **Introduced** | Pascal (pre-1995); `{$PUSHOPT}`/`{$POPOPT}` 13.0 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A comment whose first character is `$` is a compiler directive — it influences
compilation, conditional inclusion, and parsing itself.

**Grammar**

```ebnf
Directive = "{$" DirectiveBody "}" | "(*$" DirectiveBody "*)" ;
```

**Semantics & parsing notes**

- ⚠️ *Directives change what the parser sees.* Conditional-compilation directives
  (`{$IFDEF}`, `{$IF}`, `{$ELSEIF}`, `{$ELSE}`, `{$ENDIF}`, `{$IFNDEF}`) include
  or exclude whole token ranges; `{$INCLUDE}` / `{$I}` splices another file's
  tokens in. A correct parser must run a **directive/conditional pre-pass** (or
  interleave it with lexing) — it cannot treat directives as inert comments.
- `{$PUSHOPT}` / `{$POPOPT}` (13.0) snapshot and restore the current directive
  option state — model directive options as a stack.
- Full directive catalogue belongs in [01-program-structure.md](01-program-structure.md);
  the lexer only needs to *recognise and route* them here.

## B.3 Identifiers & case rules

| | |
|---|---|
| **Introduced** | Pascal (pre-1995); `&`-escaping Delphi 2005-era |
| **Deprecated** | — |
| **Status** | ✅ Current |

**Grammar**

```ebnf
Ident      = [ "&" ] IdentStart { IdentChar } ;
IdentStart = "A".."Z" | "a".."z" | "_" | ?Unicode letter? ;
IdentChar  = IdentStart | "0".."9" ;
```

**Semantics & parsing notes**

- *Case-insensitive:* `Count`, `count`, `COUNT` are the same identifier. Fold case
  for symbol lookup; keep original spelling on the token.
- ⚠️ *`&`-escape:* a leading `&` lets a **reserved word be used as an identifier**
  (e.g. `&begin`, `&type`). The `&` is not part of the name. Essential for
  interop and code generation.
- Identifiers may contain Unicode letters; the leading char is a letter or `_`.

## B.4 Reserved words vs. directives

This split is **critical for the lexer**.

### B.4.1 Reserved words (always keywords — cannot be identifiers without `&`)

```
and        array      as         asm        begin      case       class
const      constructor destructor dispinterface div      do         downto
else       end        except     exports    file       finalization finally
for        function   goto       if         implementation in       inherited
initialization inline  interface  is         label      library    mod
nil        not        object     of         or         packed     procedure
program    property   raise      record     repeat     resourcestring set
shl        shr        string     then       threadvar  to         try
type       unit       until      uses       var        while      with       xor
```

> **Compound operators (13.0):** `is not` and `not in` are parsed as the existing
> reserved words `is`/`not`/`in` in sequence; the parser recognises the pairs at
> the relational-operator level (B.7), no new tokens are introduced.

### B.4.2 Directives (context-sensitive — legal as identifiers elsewhere)

```
absolute   abstract   assembler  automated  cdecl      contains   default
delayed    deprecated dispid     dynamic    experimental export    external
far        final      forward    helper     implements index      local
message    name       near       nodefault  noreturn   operator   out
overload   override   package    pascal     platform   private    protected
public     published  read       readonly   reference  register   reintroduce
requires   resident   safecall   sealed     static     stdcall    stored
strict     unsafe     varargs    virtual    winapi     write      writeonly
```

**Semantics & parsing notes**

- ⚠️ *Directives are NOT reserved.* `index`, `name`, `read`, `write`, `message`,
  `out`, etc. are valid variable/field names; they act as keywords **only in
  their grammatical position** (e.g. `read`/`write` inside a property
  declaration). The lexer emits them as plain identifiers; the **parser**
  interprets them contextually.
- *`noreturn`* (13.0) is a new directive on procedure declarations.
- `NameOf` (13.0) is an **intrinsic function identifier**, not a reserved word —
  it parses as a normal call and is resolved by the compiler.

## B.5 Numeric literals

### B.5.1 Integer literals (decimal, hex, binary, digit separator)

| | |
|---|---|
| **Introduced** | decimal & hex (`$`) Pascal (pre-1995); **binary (`%`) and digit separator (`_`) 11 Alexandria** |
| **Deprecated** | — |
| **Status** | ✅ Current |

**Grammar**

```ebnf
IntLiteral  = DecLiteral | HexLiteral | BinLiteral ;
DecLiteral  = Digit { [ "_" ] Digit } ;
HexLiteral  = "$" HexDigit { [ "_" ] HexDigit } ;
BinLiteral  = "%" BinDigit { [ "_" ] BinDigit } ;
Digit       = "0".."9" ;
HexDigit    = Digit | "A".."F" | "a".."f" ;
BinDigit    = "0" | "1" ;
```

**Example**

```pascal
const
  Big  = 1_000_000;        // digit separator (11+)
  Mask = $FF_FF;           // hex + separator
  Bits = %1010_0001;       // binary literal (11+)
```

**Semantics & parsing notes**

- *Digit separator `_`* may appear **between** digits only; it is ignored when
  computing the value. It is not allowed leading/trailing or doubled adjacent to
  the prefix — treat such cases as lex errors.
- *Value type:* the literal's type is the smallest predefined integer type that
  holds it (up to `Int64`/`UInt64`); see ch.02 for the integer type ladder.
- *AST:* `IntLiteral { value, radix, rawText }` (keep `rawText` for fidelity).

### B.5.2 Real (floating-point) literals

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

**Grammar**

```ebnf
RealLiteral = Digit { [ "_" ] Digit }
              [ "." Digit { [ "_" ] Digit } ]
              [ ( "e" | "E" ) [ "+" | "-" ] Digit { Digit } ] ;
```

**Semantics & parsing notes**

- Must contain a `.` fraction **or** an exponent to be a real literal; otherwise
  it lexes as an integer.
- Default type is `Extended`/`Double` (platform-dependent); see ch.02.
- `$`/`%` prefixes are integer-only — no hex/binary floats.

## B.6 Character & string literals

### B.6.1 Quoted string & character literals

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

**Grammar**

```ebnf
StringLiteral = StringElement { StringElement } ;      (* adjacency = concatenation *)
StringElement = QuotedString | ControlChar | MultilineString ;
QuotedString  = "'" { ?any char except "'"? | "''" } "'" ;
ControlChar   = "#" IntLiteral ;                       (* e.g. #13  #$0A *)
```

**Example**

```pascal
S := 'It''s here';          // '' is an escaped single quote
Line := 'Hello'#13#10;      // string + control chars, concatenated by adjacency
C := 'A';                   // a 1-char string literal; Char if context demands
```

**Semantics & parsing notes**

- *Escaping:* the only in-string escape is `''` → a single `'`. There are no
  backslash escapes.
- *Concatenation by adjacency:* consecutive string elements (`'..'`, `#n`,
  multiline) with no operator between them form **one** literal — the lexer/parser
  must fold them.
- ⚠️ *Char vs. string:* a single-character literal `'A'` has an ambiguous type
  resolved by context — `Char` where a `Char` is expected, otherwise a 1-length
  `string`. Carry this as a "char-or-string literal" until typing.
- `#n` gives the character with ordinal `n` (decimal or `$hex`). `^X` caret
  notation (e.g. `^M` = `#13`) is also accepted.
- *AST:* `StringLiteral { segments[] }` or a folded constant value.

### B.6.2 Caret control characters

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`^X` denotes a control character (ordinal of uppercased `X` minus 64).

```pascal
CRLF = ^M^J;   // #13#10
```

### B.6.3 Multiline (triple-quoted) string literals

| | |
|---|---|
| **Introduced** | 12 Athens (2023) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A string spanning multiple source lines, delimited by `'''` (or any larger
**odd** count of quotes), for embedding SQL/JSON/HTML/XML without escaping.

**Grammar**

```ebnf
MultilineString = QuoteRun LineBreak { ?content line? LineBreak } Indent QuoteRun ;
QuoteRun        = "'''" { "'" } ;   (* odd count >= 3; open and close counts must match *)
```

**Example**

```pascal
const SQL =
  '''
  select *
  from Customer
  where Id = :Id
  ''';
```

**Semantics & parsing notes (indentation rules — important for the lexer):**

- *Opening:* the opening `'''` must be **followed by a line break**; content
  starts on the next line.
- *Closing:* the closing `'''` must be on a **line of its own**.
- ⚠️ *Indentation base:* the indentation (leading whitespace) of the **closing**
  `'''` defines the "ignored indentation". From every content line, that many
  leading whitespace columns are removed; extra indentation beyond it is kept.
- ⚠️ *Under-indent is an error:* no content line may be **less** indented than the
  closing `'''` → compile error.
- *Trailing newline:* the line break immediately before the closing `'''` is
  **omitted** from the value; add a blank line to force a trailing newline.
- *Embedding quotes:* use a larger odd run (`'''''`) to embed a literal `'''`.
- *AST:* lower to the same `StringLiteral` node as B.6.1 after de-indentation; keep
  a flag/raw text if source fidelity is needed.

## B.7 Operators & punctuation tokens

Symbolic tokens the lexer produces:

```ebnf
Punct = ":=" | "+" | "-" | "*" | "/" | "=" | "<>" | "<" | ">" | "<=" | ">="
      | "(" | ")" | "[" | "]" | "." | ".." | "," | ":" | ";" | "^" | "@"
      | "(." | ".)"                  (* legacy alternates for "[" and "]" *) ;
```

Word operators are reserved words (B.4.1): `and or xor not div mod shl shr in is as`.

**Precedence & associativity** (highest → lowest; all binary operators are
**left-associative**):

| Level | Operators | Category |
|-------|-----------|----------|
| 1 (highest) | `@`  `not`  unary `+`  unary `-`  `^`(deref, postfix) | unary |
| 2 | `*`  `/`  `div`  `mod`  `and`  `shl`  `shr`  `as` | multiplicative |
| 3 | `+`  `-`  `or`  `xor` | additive |
| 4 (lowest) | `=`  `<>`  `<`  `>`  `<=`  `>=`  `in`  `is`  `is not`(13)  `not in`(13) | relational |

**Semantics & parsing notes**

- ⚠️ *Boolean operators bind tighter than relational.* `a > 0 and b > 0` parses as
  `a > (0 and b) > 0` → type error. Real code must write `(a > 0) and (b > 0)`.
  This is the single most common precedence surprise; the parse tree follows the
  table strictly — do not special-case it.
- `as`/`is` are at multiplicative/relational levels respectively (see ch.12).
- `^` is both a **prefix** (pointer type, `^T`) and a **postfix** (dereference,
  `p^`) operator depending on position (see ch.10).

---

# Part 2 — Shared grammar productions

These are the canonical non-terminals other chapters reference. Detailed
semantics live in the cited chapters; here we fix the *syntax* and the *names*.

## B.8 Identifiers, names, and designators

```ebnf
IdentList      = Ident { "," Ident } ;
QualifiedIdent = Ident { "." Ident } ;                 (* unit.scope.name *)

Designator     = ( Ident [ GenericArgs ] | "inherited" [ Ident ] )
                 { Selector } ;
Selector       = "." Ident [ GenericArgs ]              (* member access; args → 16.3 *)
               | "[" ExprList "]"                       (* index / array prop *)
               | "(" [ ExprList ] ")"                   (* call *)
               | "^" ;                                  (* pointer dereference *)
ExprList       = Expression { "," Expression } ;
```

**Semantics & parsing notes**

- A `Designator` covers variable references, field/method access, array
  indexing, calls, and dereference in one left-to-right chain — this is the core
  shape the parser builds, then semantic analysis classifies each `Selector`
  (field vs. method-call vs. default-array-property, etc.).
- The **call-vs-reference** ambiguity (see [05](05-statements.md) §5.1.2) lives
  here: a trailing `(...)` is an explicit call; its **absence** is resolved by
  context.

## B.9 Expressions

```ebnf
Expression   = SimpleExpr [ RelOp SimpleExpr ] ;
RelOp        = "=" | "<>" | "<" | ">" | "<=" | ">=" | "in" | "is"
             | "is" "not" | "not" "in" ;                (* compound forms 13.0 *)
SimpleExpr   = [ "+" | "-" ] Term { AddOp Term } ;
AddOp        = "+" | "-" | "or" | "xor" ;
Term         = Factor { MulOp Factor } ;
MulOp        = "*" | "/" | "div" | "mod" | "and" | "shl" | "shr" | "as" ;
Factor       = Designator
             | Literal
             | "(" Expression ")"
             | "not" Factor
             | "@" Factor
             | SetConstructor
             | InlineIfExpr                              (* 13.0; see 05 §5.4.1 *)
             | TypeCast ;
Literal      = IntLiteral | RealLiteral | StringLiteral | "nil" ;
SetConstructor = "[" [ SetElement { "," SetElement } ] "]" ;
SetElement   = Expression [ ".." Expression ] ;
TypeCast     = TypeRef "(" Expression ")" ;
```

**Semantics & parsing notes**

- This grammar **encodes the B.7 precedence** structurally (relational → additive
  → multiplicative → factor). Implementations using precedence-climbing/Pratt
  parsing must reproduce the same four levels and left-associativity.
- `[ ... ]` is overloaded: a **set constructor** in expression position vs.
  **indexing** in a `Selector`. Position disambiguates.
- `TypeCast` vs. `call`: `TypeName(Expr)` is a type cast when the callee names a
  type; otherwise it is a call. Resolve via symbol kind.
- `InlineIfExpr` enters here, giving `if..then..else` an expression role.

## B.10 Constant expressions

```ebnf
ConstExpr = Expression ;   (* must be evaluable at compile time *)
```

**Semantics & parsing notes**

- Syntactically identical to `Expression`; the **constraint** (compile-time
  constant) is enforced semantically. Used for case labels, array bounds, enum
  values, default parameters, and `const` declarations.

## B.11 Type references

```ebnf
TypeRef    = TypeName                                   (* named/aliased type, incl. generics *)
           | "^" TypeRef                                (* pointer type *)
           | "string" [ "[" ConstExpr "]" ]            (* string / short string *)
           | "array" ... | "record" ... | "set" "of" ...   (* structural; see ch.02/08/09 *) ;
TypeName    = TypeSegment { "." TypeSegment } ;
TypeSegment = Ident [ GenericArgs ] ;
(* ⚠️ GenericArgs may appear on EVERY dotted segment, not just the last:
   TDictionary<K,V>.TPairEnumerator, TObjectList<T>.TEnumerator — the RTL's
   Generics.Collections relies on this heavily. *)
GenericArgs = "<" TypeRef { "," TypeRef } ">" ;
```

**Semantics & parsing notes**

- ⚠️ *Generic `<` ambiguity:* `<` begins a generic argument list **or** is the
  less-than operator. In *type* context (after a type name in a declaration,
  `TList<Integer>`) it is generic args; in *expression* context it is comparison.
  Delphi resolves this largely by context — the parser must know whether it is
  reading a type or an expression. Inline-type-args in expressions
  (`TList<Integer>.Create`) need lookahead. Full rules in ch.16.
- Structural type bodies (`record … end`, `array[…] of …`, `set of …`) are
  defined in their chapters; `TypeRef` is the umbrella reference.

## B.12 Block (declarations + body)

```ebnf
Block         = { DeclSection } CompoundStmt ;
DeclSection   = LabelDeclSection
              | ConstSection | TypeSection | VarSection
              | ProcedureDeclSection ;
ConstSection  = ( "const" | "resourcestring" ) { ConstDecl } ;
TypeSection   = "type" { TypeDecl } ;
VarSection    = ( "var" | "threadvar" ) { VarDecl } ;
```

**Semantics & parsing notes**

- A routine body is a `Block`. Declaration sections may repeat and interleave
  (`const` … `var` … `const` …) in modern Object Pascal — do not require a fixed
  order.
- *Inline `var`/`const`* (10.3+) appear **inside** `CompoundStmt` as statements,
  not in `DeclSection` — see ch.03; they change scope-resolution within the block.
- `CompoundStmt`, `StatementList`, and all statement productions are defined in
  [05-statements.md](05-statements.md).
