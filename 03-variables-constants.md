# 03 — Variables & Constants

Declarations of mutable and immutable storage, including the modern **inline
variable** form and its type inference, which materially affects scope
resolution inside a block.

Shared productions (`TypeRef`, `Expression`, `ConstExpr`, `Block`) → [Appendix
B](B-lexical-grammar.md).

## Chapter grammar umbrella

```ebnf
VarSection = ( "var" | "threadvar" ) VarDecl { VarDecl } ;
VarDecl    = IdentList ":" TypeRef [ VarInitOrAbsolute ] ";" [ HintDirectives ] ;
VarInitOrAbsolute = "=" ConstExpr            (* initialized *)
                  | "absolute" ( Ident | ConstExpr ) ;   (* overlay *)

ConstSection = ( "const" | "resourcestring" ) ConstDecl { ConstDecl } ;
ConstDecl    = Ident [ ":" TypeRef ] "=" ConstExpr ";" [ HintDirectives ] ;
```

---

## 3.1 Variables

### 3.1.1 Variable declaration (`var` section)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Declares one or more named, typed storage locations.

**Grammar**

```ebnf
VarDecl = IdentList ":" TypeRef [ "=" ConstExpr ] ";" ;
```

**Example**

```pascal
var
  X, Y: Integer;
  Name: string;
```

**Semantics & parsing notes**

- Multiple identifiers share one type (`X, Y: Integer`).
- A `var` section may appear at unit level, in a routine `Block`, and in a class
  (as fields, ch.11). Declaration sections may **interleave** with `const`/`type`
  in any order (B.12).
- *AST:* one `VarDecl { names[], type, init? }` (or split per name). A tree
  that keeps the names, the type and the initializer as bare sibling nodes
  must record where the names end: `P, T: C` and `P: T = C` (3.1.2) are both
  three identifiers, and so are a parameter's `(A, T: X)` and `(A: T = X)`
  (6.2). Only the separators tell them apart, so without the mark every
  consumer of the tree has to re-read the tokens.

### 3.1.2 Initialized (global) variables

| | |
|---|---|
| **Introduced** | Delphi 2 (global init) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A variable may be given an initial value with `= ConstExpr`. Only legal for
**single** identifiers (not an `IdentList`), and historically only for globals
(unit/program scope); inline locals (3.1.3) also support it.

**Example**

```pascal
var
  Counter: Integer = 0;
  Greeting: string = 'Hi';
```

**Semantics & parsing notes**

- The initializer must be a **constant expression**.
- ⚠️ Only one identifier may be initialized — `var A, B: Integer = 0;` is illegal.
  The parser should reject an `=` initializer when `IdentList` has more than one
  name (dcc64 37.0: `E2196 Cannot initialize multiple variables`). A routine's
  local `var` section takes no initializer at all, for one name as for several:
  `E2195 Cannot initialize local variables`.
- ⚠️ *Hint directives may sit BETWEEN the type and the initializer:*
  `Default8087CW: Word platform = $033F;` (System.pas). Parse hints in both
  positions.
- ⚠️ *Procedural-type variables put the calling convention after the `;`, and
  may still carry an initializer after it:*
  `SSL_COMP_free_compression_methods : procedure; cdecl = nil;`
  (IdSSLOpenSSLHeaders.pas). The directives are the procedural type's (6.6.1)
  and the initializer is the variable's - a name in it is a reference like any
  other. A typed constant (`const C: procedure; cdecl = CP;`), an inline const
  and an inline var (`var P: procedure; stdcall := SP;`) take the same shape
  (dcc64 37.0).
- Uninitialized globals are zero-filled; uninitialized **locals** are *not*
  (except managed types) — a semantics/codegen concern.

### 3.1.3 Inline variables & type inference

| | |
|---|---|
| **Introduced** | 10.3 Rio (2018) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `var` (or `const`) declaration that appears **as a statement inside a block**,
scoped from its point of declaration. With an initializer and no type, the type
is **inferred**.

**Grammar**

```ebnf
InlineVarStmt = "var" IdentList [ ":" TypeRef ] [ ":=" Expression ] ";" ;
(* note ':=' here, an executable initializer, vs '=' for a section ConstExpr.
   Multiple names are legal: `var V, S: string;` — System.SysUtils.pas.
   TypeRef is the FULL B.11 production, structural forms included — see the
   notes below. *)
```

**Example**

```pascal
begin
  var I: Integer := 0;
  var Name := Edit1.Text;     // inferred as string
  for var K := 1 to 10 do ... // inline counter (see 05 §5.5.1)
end;
```

**Semantics & parsing notes**

- ⚠️ *Two different initializer tokens:* a classic section `VarDecl` uses `= ConstExpr`
  (constant, compile-time); an inline `var` statement uses `:= Expression` (runtime,
  any expression). Parse them differently based on context (statement position ⇒
  inline form).
- ⚠️ *Scope:* an inline variable is visible **from its declaration to the end of the
  enclosing block** — not the whole routine. This is a block-scope rule absent from
  classic Pascal; the name-resolution pass must track declaration order/position,
  not just the block.
- ⚠️ *For-header exception:* a `for var K := ...` counter / `for var E in ...`
  element is scoped to **the loop statement itself**, NOT to the enclosing block —
  two sibling loops may reuse the same name without a redeclaration error
  (dcc-verified: two consecutive `for var LWord in ...` loops compile fine). A
  resolver applying the general to-end-of-block rule to for-header declarations
  produces false E2004. See 05 §5.5.1/§5.5.2.
- ⚠️ *No redeclaration within one body:* an inline `var`/`const` or a `for var`
  counter may not take a name its body already holds - `E2004 Identifier
  redeclared` (dcc64 37.0, probed 2026-09-27). A routine's body holds its
  parameters, its locals, `Result` and the inline declarations of its
  enclosing blocks that are still in scope: `procedure P(G: Integer); begin
  var G := 1; end;` fails, and so does `begin var G := 1; begin var G := 2;
  end; end`. An implementation's body holds what its declaration and its kind
  bring as well: the parameters and `Result` a header that omits them leaves
  to the declaration, `Self` in a method with one (an instance method, a
  constructor, a non-static class method, a record's or a helper's method -
  not a `class ... static` one), and the method's OWN type parameters
  (`procedure TC.M<T>`), not its class's (`procedure TG<T>.M` may declare a
  `T`). These hold against an ordinary local too: `var Self`, `var Result`,
  `var T`, or a `var G` beside a declared `(G: Integer)` the header omitted,
  are `E2004` - and a parameter named `Result` in a function, `Self`, or like
  the method's type parameter is `E2004` once, reported at the
  implementation's `begin`. The routine's own name is not its body's: `var F
  := 1` in function `F` compiles. The module's own body - the initialization
  section, the legacy `begin` form, a program's or library's main block -
  holds every module-level
  name, the interface's and the implementation's alike, at any block depth
  (`unit U; interface var G: Integer; implementation initialization begin var
  G := 1; end; end.` fails). The finalization section is a body of its own:
  its inline vars may take a module-level name. An anonymous method and a
  nested routine start a new body too. An `on E: T do` variable may hide an
  outer name, but an inline declaration inside that handler may not take `E`.
  What an inline declaration MAY hide: a name of an enclosing BODY (a global
  from a routine, a field from a method, an outer routine's local from a
  nested routine, a module-level name from the finalization section), a used
  unit's name, a generic type's name (its arity is part of it) and a name
  whose block has ended. Every statement list is such a block: a `repeat`
  body, the `try`, `except` and `finally` parts and a `case` statement's
  `else` part end their inline declarations' scope as `begin ... end` does.
- *Visible past its WHOLE declaration:* the declaration's own type and
  initializer still see the outer name it hides - `var G: Integer :=
  G.Count;` reads a global `G: TStringList`, `var X: Integer := X.Length;` a
  global string X, and `var Parent := Parent as TForm` the inherited
  property. dcc32 / dcc64 37.0, 2026-10-13 (PasTree audit B43, A2-09).
- *A sole statement is a block:* `if B then var A := 1;`, `case N of 1: var
  C := 2; end;`, `while B do var D := 3;` (and an `else` or `with` body) scope
  the variable to that statement - a later `var A` in the enclosing block is
  no redeclaration. The initialization section (and a unit's legacy `begin`)
  is a block as well: its inline var is visible from its declaration on, so
  `var GHidden := 5` there, hiding a used unit's GHidden, leaves the routines
  above the section reading the used unit's. dcc32 / dcc64 37.0, 2026-10-13
  (PasTree audit B43: B4-10, A4-06).
- *Type inference:* with `:=` and no `: TypeRef`, the type is the static type of the
  initializer expression. `var X := 1` ⇒ `Integer`.
- ⚠️ *What a LITERAL initializer infers (dcc 37.0, dcc32 and dcc64, printing
  `GetTypeName(TypeInfo(T))` through a generic method):*
  - an integer literal takes the narrowest of `Integer`, `Cardinal`, `Int64`,
    `UInt64` that holds its value **after the sign is applied**: `2147483647`
    ⇒ Integer, `2147483648` and `$FFFFFFFF` ⇒ Cardinal, `4294967296` ⇒ Int64,
    `18446744073709551615` ⇒ UInt64; `-2147483648` ⇒ Integer,
    `-2147483649` ⇒ Int64. A folded constant expression follows the same rule
    (`5000000000 + 1` ⇒ Int64), and a named true constant carries the type it
    was inferred with (`const CBig = 5000000000; var X := CBig` ⇒ Int64).
  - a real literal is `Extended` on Win32. **On Win64 a real literal written
    without an exponent, with at most four fractional digits and inside
    Currency's range is `Currency`**: `1.5`, `0.1`, `2.0`, `1.50`, `-1.5`,
    `1.5 + 1.5` and `922337203685477.5807` ⇒ Currency, while `3.14159`,
    `1.50000`, `1e3`, `1.5e0`, `922337203685478.0` and `A / 2` ⇒ Extended.
    `TCurrencyHelper` exists in System.SysUtils, so `X.ToString` compiles
    either way, but `X.Exponent` is E2003 on Win64. Probed on Win64 only;
    the other 64-bit targets are assumed to share it (Extended is
    Double-sized there too).
  - a string literal denoting exactly one UTF-16 unit is `Char` (`'a'`,
    `''''`, `#65`, `#$41`, `^M`); anything else is `string` (`''`, `'ab'`,
    `'a'#0`, `#$1F600` — a surrogate pair).
  - `True` ⇒ Boolean; `@X` ⇒ Pointer; `nil` is accepted (`var P := nil`
    compiles); `[1, 2]` ⇒ an anonymous set type; a bare class name ⇒ an
    anonymous class-reference type.
- ⚠️ *A routine name as the initializer is a CALL when the routine needs no
  arguments* (6.6.1 — parameterless, or every parameter defaulted): `var N :=
  GF.Add` with `Add: TStringList; overload` beside `Add(A: Integer): Integer;
  overload` infers TStringList. A routine that still requires an argument is
  `E2035 Not enough actual parameters`, not a procedural value.
- ⚠️ *Only one name may be initialized:* `var X, Y := 5;` is `E2196 Cannot
  initialize multiple variables` (dcc 37.0), and so is the typed
  `var X, Y: Integer := 5;` (dcc64 37.0). A type or an initializer is
  required: `var X;` is `E2029 ':' expected but ';' found`. An inline `const`
  declares ONE name: `const A, B = 5;` is `E2029 '=' expected but ',' found`.
- ⚠️ *Inline `const` keeps the `=` token but drops the constant-expression
  requirement.* An inline `const` (a `const` declaration appearing as a
  statement inside a block) is written `const Ident [: TypeRef] = Expression;`
  — still `=`, not `:=` (dcc-verified: `const X := Expr;` is
  `E2029 '=' expected but ':=' found`) — but unlike a section-level
  `ConstDecl` (B.10/3.2.1), the right-hand side may be **any** expression, not
  just one evaluable at compile time. dcc32 37.0 accepts, inside a routine
  body, both `const X = SomeFunc;` (a function call) and
  `const Y: Integer = GVar + 1;` (an expression over a plain variable), and
  both run at the declaration's position, taking whatever value the
  expression has there. The identical declaration at unit/section scope is
  rejected: `E2026 Constant expression expected`. Type inference for an
  untyped inline `const` follows the same rule as inline `var` — the static
  type of the initializer expression, not a narrowed constant-folded type.
- ⚠️ *The `:` slot takes full structural type syntax, not just named type
  references:* `var A: array[0..1] of Byte;` and `var S: set of Byte;` both
  compile inside a `begin`/`end` block (dcc64-verified 2026-08). The `TypeRef`
  in the grammar above therefore means the complete B.11 production
  (`array`/`set`/`record`/`^`/`string[N]` bodies included), not merely
  `TypeName`.
- *AST:* `InlineVar { names[], type?, init?, pos }` as a statement node. With
  the type and the initializer both optional, the identifiers alone do not
  fix the shape: `var X: K` and `var X := K` are two identifiers each, `var
  X, Y: K` and `var X: Y := K` three. A tree that keeps them as sibling nodes
  must record the names, and whether a single node after them is the type or
  the initializer.

### 3.1.4 `absolute` variables (overlay)

| | |
|---|---|
| **Introduced** | Pascal/Turbo (pre-1995) |
| **Deprecated** | — |
| **Status** | ⚠️ Legacy |

Declares a variable that occupies the **same memory** as another variable (a
typed reinterpretation overlay).

**Grammar**

```ebnf
AbsoluteVar = IdentList ":" TypeRef "absolute" ( Ident | ConstExpr ) ";" ;
```

**Example**

```pascal
var
  I: Integer;
  B: array[0..3] of Byte absolute I;   // B overlays I's bytes
```

**Semantics & parsing notes**

- `absolute` is a **directive** (B.4.2), keyword only in this position.
- No initializer allowed on an `absolute` variable.
- *AST:* `VarDecl { …, absolute: target }`.

### 3.1.5 `threadvar`

| | |
|---|---|
| **Introduced** | Delphi 2/3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Declares a variable with **thread-local storage** — each thread gets its own copy.

**Grammar**

```ebnf
ThreadVarSection = "threadvar" VarDecl { VarDecl } ;
```

**Semantics & parsing notes**

- `threadvar` is a **reserved word** (B.4.1), used only at unit level (not inside
  routines, and not inside a type body on its own - there only `class threadvar`
  is a member, §15.1.2).
- ⚠️ No initializer is allowed (thread-local storage is zero-initialized per
  thread). Reject `= ConstExpr` here.
- *AST:* mark the `VarDecl` with `storage = threadlocal`.

---

## 3.2 Constants

### 3.2.1 True constants

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A named compile-time constant with no explicit type — the type is taken from the
constant expression.

**Grammar**

```ebnf
TrueConstDecl = Ident "=" ConstExpr ";" ;
```

**Example**

```pascal
const
  Max = 100;            // sized by value - see below; `var X := Max` is Integer
  Pi  = 3.14159;        // Extended
  Hi  = 'Hello';        // string
```

**Semantics & parsing notes**

- The RHS must be evaluable at compile time (B.10), which includes constant
  folding of operators, `Ord`, `Length` of string consts, set constructors, etc.
- *Constant type:* inferred from the expression. What an inline `var X := C`
  gets from it follows the literal rules of 3.1.3: `100` ⇒ Integer,
  `5000000000` ⇒ Int64, `'a'` ⇒ Char, `'abc'` ⇒ string, `1.5` ⇒ Extended on
  Win32 / Currency on Win64 (dcc 37.0). The constant's OWN type - the one a
  member access on it sees - is narrower; see below.
- ⚠️ *A true constant HAS a type for member access:* `const CI = 100;`
  then `CI.ToString`, and `const CS = 'abc';` then `CS.Length`, both compile
  (dcc 37.0) — the helper is looked up on the constant's type, so a resolver
  must record it rather than treat the constant as untyped.
  Hint directives after the value change nothing: `CDep = 'abc' deprecated;`,
  `CPlat = 'xyz' platform;`, `CNum = 5 deprecated 'x';` are typed like the
  unhinted constant (`CDep.Bogus` is `E2003`, `var L := CDep` is a string);
  each hint is a node after the initializer, which is therefore not the
  declaration's last child. dcc32 / dcc64 37.0, 2026-10-13 (PasTree audit
  B42, A3-28; local/probe-b42).
- ⚠️ *An initializer that is an OPERATOR over other constants has a type
  too,* by the operator rules of 4.2.1: `const KindA = 'S'; KindB = 'T';
  Kinds = KindA + KindB;` is a `string`, and `Kinds.CountChar(C)` binds to
  TStringHelper. A resolver that types only literal initializers leaves every
  constant built from named constants - a common way to spell a character
  class - with no type, and every member reached through it with no
  declaration.
- ⚠️ *The constant's own type is narrower than what `var X := C` infers*
  (dcc32 and dcc64 37.0, probed 2026-09-25 by reading which record helper a
  member access binds to - each System.SysUtils helper's `MaxValue`
  differs - and `GetTypeName(TypeInfo(T))` through a generic method):
  - an integer constant is typed by its VALUE: the first of `ShortInt`,
    `Byte`, `SmallInt`, `Word`, `Integer`, `Cardinal`, `Int64`, `UInt64`
    that holds it. `0`, `127`, `-128` find TShortIntHelper; `128`, `255`
    TByteHelper; `256`, `32767`, `-129` TSmallIntHelper; `32768`, `65535`
    TWordHelper; `65536`, `-32769`, `2147483647` TIntegerHelper;
    `2147483648` TCardinalHelper; `4294967296` TInt64Helper. A FOLDED
    expression follows the same rule (`C + 1`, `C div 3`, `(C + 1) * 2`,
    `not C`, a class member constant `TFoo.K`). `SizeOf(C)` of `const C =
    100` is 1, and a generic argument `C` infers an anonymous subrange
    (`GetTypeName` prints `:2`).
  - a typecast or a typed intrinsic fixes the type instead of the value:
    `Byte(5)` is a Byte, `Integer(100)` an Integer, `High(Byte)` a Byte,
    `High(Integer)` an Integer, `Length('abc')` and `SizeOf(T)` an Integer,
    `Trunc`/`Round` an Int64, `Ord('A')` and `Ord(High(Char))` a Word; but
    `Ord(E)` of an enum value is by value again (a ShortInt). A folded
    expression OVER such a constant is by value: `Byte(5) + Byte(5)` is a
    ShortInt, and `not Byte(5)` folds to -6, a ShortInt - where `not B` over
    a Byte VARIABLE is a Byte (4.2.1).
  - a real literal written like a Currency (no exponent, at most four
    fractional digits, in range) is a **Currency constant on both
    compilers**: `const R = 1.5;` binds TCurrencyHelper, and `SizeOf(R)` is
    8 on dcc32 too (an Extended is 10 there), while `var X := R` is Extended
    on dcc32 (3.1.3). A folded `R + 1` is Extended on dcc32 and Currency on
    dcc64; `R + R2` over two such constants is Currency on both; `R / 2` and
    `3.14159 + R` are Extended.
  - a REAL typecast does not survive: `Double(1.5)` and `Single(2)` are
    Extended constants (TExtendedHelper), not Double and Single.
  - a typecast to a user SUBRANGE type (`TSub(3)` with `TSub = 1..10`) keeps
    the subrange inside its own unit - no helper applies, `C.MaxValue` is
    E2671, as for a variable of a subrange type - but read from ANOTHER unit
    the same constant binds by value (TShortIntHelper).
  - `var X := C` widens: an integer constant narrower than 32 bits gives an
    Integer (a `SizeOf` constant and `2147483647` give an anonymous 32-bit
    subrange instead), and `const R = 1.5` gives an Extended on dcc32. "The
    constant's type" and "the type of a variable initialized from it" are
    two answers. The constant's own type decides which sibling helper a
    member binds to - and so, for a member one sibling lacks (`Exponent` is
    not on TCurrencyHelper), whether it binds at all.
- ⚠️ *An equality comparison is a constant expression:* `const B = S =
  'abc';` is a Boolean constant. Token by token it also reads as a
  declaration whose initializer is missing, followed by the next declaration
  `S = 'abc';` - a parser that recovers from a missing initializer must not
  take that reading when the name after the first `=` is on the same line.
- A set constructor (`const S = [1, 2]`, `[meA, meB]`) and `nil` give an
  ANONYMOUS type. A constant `[]` passed to a generic method, or used as the
  initializer of an inline `var`, crashes dcc 37.0 with `F2084 Internal
  Error` on both compilers.
- *AST:* `ConstDecl { name, value, inferredType }`.

### 3.2.2 Typed constants (& writeable-constants directive)

| | |
|---|---|
| **Introduced** | Pascal/Turbo; `{$J}` / `{$WRITEABLECONST}` toggle Delphi-early |
| **Deprecated** | — |
| **Status** | ✅ Current |

A constant declared **with an explicit type**. Crucially, a typed constant is
actually an *initialized static variable*: it may be writeable depending on the
`{$WRITEABLECONST}` / `{$J}` directive.

**Grammar**

```ebnf
TypedConstDecl = Ident ":" TypeRef "=" ConstExpr ";" ;
```

**Example**

```pascal
const
  Origin: TPoint = (X: 0; Y: 0);     // structured typed constant
  Counter: Integer = 0;              // writeable iff {$J+}
```

**Semantics & parsing notes**

- ⚠️ *Typed constant ≠ true constant.* It has an address and storage. Under
  `{$J+}` (default off in modern code) it can be assigned to at runtime — a
  directive-dependent semantic, so the directive state at the declaration matters.
- Structured typed constants use **parenthesised aggregate** initializers:
  records `(Field: val; …)`, arrays `(a, b, c)`, sets `[…]`.
- ⚠️ *A record aggregate's `Field` names are NOT ordinary identifier
  references* — resolved structurally against the declared type's own
  fields, not by scope lookup (a semantic layer that treats `Field` in
  `(Field: val; …)` as a normal reference and looks it up in the enclosing
  scope chain will false-E2003 on it, since a field name generally isn't
  declared anywhere reachable that way — real bug, found on
  `UtilWindowClass: TWndClass = (style: 0; lpfnWndProc: ...)`,
  System.Classes.pas). Applies equally to a `var` with an initializer
  (3.1.2), not just `const` — the same aggregate grammar. A nested aggregate
  (record-typed field) resolves against THAT field's own type, recursively;
  array/set elements have no names and need no special handling.
- ⚠️ *String-literal → `TGUID` magic:* a typed constant (or initialized variable)
  of type `TGUID` may be initialized from a **GUID string literal** —
  `const IID_IFoo: TGUID = '{00000000-…-000046}';` — a compiler-special implicit
  conversion that exists *only* in constant-initializer position. Used throughout
  the D13 sources (`Vcl.AxCtrls`, tethering, Winapi headers). The parser sees an
  ordinary string initializer; the constant-evaluator must handle the conversion.
- ⚠️ *A string expression initializes an array of `Char`/`AnsiChar` - but
  not in parentheses.* `C: array[0..5] of AnsiChar = 'abc' + 'def';` compiles
  (the string fills the array, a concatenation of literals and string
  constants included); `= ('abc' + 'def')` is E2010 "Incompatible types:
  'AnsiChar' and 'string'": a `(` at the start of a structured type's value
  opens the aggregate, and the parenthesized string becomes its only
  ELEMENT. The same in every value position - a typed constant's, an
  initialized variable's, an aggregate element's (`array[0..1] of
  array[0..2] of AnsiChar = (('ab' + 'c'), 'def')` fails), a record
  constant's field value. For a scalar value (`Integer = (1 + 2)`, a string,
  a set, a `Byte` element) the parentheses are an ordinary expression. So a
  parser can tell the aggregate from a parenthesized expression only by the
  declared type; mORMot's character tables are written this way (probed on
  dcc64 37.0).
- *AST:* `TypedConstDecl { name, type, value, writeable }`.

### 3.2.3 `resourcestring`

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

String constants stored in a localizable resource table.

**Grammar**

```ebnf
ResourceStringSection = "resourcestring" { Ident "=" ConstExpr ";" } ;
```

**Example**

```pascal
resourcestring
  SHello = 'Hello, world';
  SError = 'Operation failed: %s';
```

**Semantics & parsing notes**

- `resourcestring` is a **reserved word** (B.4.1). The RHS must be a **string**
  constant expression (string literals + constant concatenation only).
- Behaves like a read-only `string` constant at use sites; differs only in storage
  and localizability.
- *Always a `string`, whatever the literal:* `SComma = ','` is no Char as a
  true constant of one character would be - `R1.Substring(0)` compiles, `var
  V := R1` infers a string, and `F(R1)` takes F(string) over F(Char). dcc32 /
  dcc64 37.0, 2026-10-13 (PasTree audit B42, A3-29).
- *AST:* `ResourceStringDecl { name, value }`.

---

## 3.3 Scope, lifetime & visibility

### 3.3.1 Scope and lifetime rules

| | |
|---|---|
| **Introduced** | Pascal; block-scoped inline vars 10.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Where a name is visible and how long its storage lives.

**Semantics & parsing notes** (drives the name-resolution pass)

- *Classic locals* (section `var` in a routine) are visible throughout the whole
  routine body and live for the call's duration.
- *Inline locals* (3.1.3) are visible only **from declaration point to end of the
  enclosing statement block** — narrower than the routine. Shadowing an outer name
  with a later inline `var` is legal and position-dependent only across bodies -
  a global, a field, an outer routine's local; a name of the inline var's own
  body (a parameter, a local, an enclosing block's inline var, in the module's
  body any module-level name) is `E2004` (3.1.3).
- ⚠️ *A managed-type inline/block-scoped local's LIFETIME, not just its
  visibility, ends at the enclosing sub-block* — it is finalized when that
  inner `begin … end` exits, not when the routine returns. dcc-verified,
  dcc32 37.0: an interface-typed inline `var` declared inside a nested
  `begin … end` inside a procedure runs its destructor (via
  `_Release`/`Destroy`) immediately at that inner block's `end`, before any
  code following the block runs — while an ordinary (non-inline,
  routine-top-level) local of the same managed type declared in the same
  procedure is finalized only at the routine's own exit, after the nested
  block has already finished. The same early-finalization timing applies to
  compiler-generated temporaries holding intermediate expression results
  inside the sub-block. A resolver/codegen that always emits finalization at
  routine exit is therefore wrong for any managed local declared inside a
  nested block — it must finalize at the innermost enclosing `CompoundStmt`'s
  end that the declaration is scoped to (mirrors the 3.1.3 for-header
  exception: scope AND lifetime both follow the narrowest enclosing block).
- *Globals* (unit `var`) live for program duration; `interface`-section vars are
  visible to importing units, `implementation`-section vars are unit-private (ch.01).
- *Resolution order* for an unqualified identifier: see §3.3.2 - the whole
  order in one place. The parser builds scopes; resolution is a separate
  pass, but it depends on **declaration position** for inline vars.
- *AST:* no dedicated node; scope tables are built from declaration nodes.

### 3.3.2 Name resolution order

| | |
|---|---|
| **Introduced** | Pascal; with, helpers, nested types and generics added their steps later |
| **Deprecated** | — |
| **Status** | ✅ Current |

Which declaration an UNQUALIFIED identifier means. The rules for single
constructs live in their chapters; this section is the order they fit into,
and the general approach behind it - the one place to read before changing
a resolver. Every step below is dcc-verified (dcc64 37.0) unless it says
otherwise; the probes are cited in the bullets and the sections referenced.

**The general approach**

1. **A chain of scopes, innermost first; the FIRST scope that declares the
   name decides.** The search does not go on to find a "better" candidate -
   a wrong kind, a wrong type or a wrong argument list in the scope that
   declares the name is an error there (`E2010`, `E2250`, `E2029`), not a
   reason to look further. This is what makes most of the hazards below
   hazards: a resolver that keeps searching after a hit, or that searches
   in another order, binds something that compiles - and is not dcc's.
2. **Three exceptions, each a filter on a candidate, not a reordering of the
   chain** - the candidate is passed over and the search goes on:
   - a TYPE of another generic arity than the reference is written with
     (§16.1.2: a bare `TFoo` passes over `TFoo<T>`, `TFoo<Integer>` over a
     plain `TFoo`, and a bare `Pointer` over a used unit's `Pointer<T>` to
     the predefined type - B.4);
   - a MEMBER the code cannot see (private of another unit, strict private
     outside its type): the walk goes on up the ancestry, then on to the
     unit (step 3 below);
   - a routine marked `overload` in a UNIT scope: the search goes on through
     the used units and merges their overloads of the name (§6.3.1) - but
     not from a MEMBER's overload set to the unit's routines (step 3);
   - nothing else. In particular an INSTANCE member reached where there is
     no instance (an outer type's, step 5) is `E2124`, not passed over.
3. **Position counts.** An inline `var` exists from its declaration on
   (§3.1.3); a conditional's `Declared()` sees only what lies above it
   (§1.3.2); and in a nested type's method dcc's answer can depend on what
   the unit wrote BEFORE (step 5's lookup history).
4. **A qualifier's head is an ordinary bare name** (§1.2.3): `A.B` looks up
   `A` by this same order - a member, a local, a with target's member named
   like a unit hides the unit there - and only then `B` in what `A` is.
   A MEMBER of the method's class hides a unit only inside a routine's
   BODY (its declarations and statements); in a type's declaration and in
   a routine heading - a method's implementation heading included - the
   head is still the unit. With an inherited property `uA` and a unit `uA`
   declaring `TX`, a field `F: uA.TX` and `procedure TKid.M(A: uA.TX)`
   compile, while `var L: uA.TX` in M's var section is `E2007` and
   `uA.TX.XF` in its statements `E2018` (probed 2026-10-13, dcc64 37.0,
   probe-b48 P1-P4).
   A head spelling the unit's OWN name makes `B` a unit-level declaration
   of either section made so far, never a member of the enclosing type
   (§1.1, the module-name bullet).
5. **A routine's heading is looked up outside the routine** (§6.2): the
   parameter types, defaults and result type see neither the routine's
   parameters nor its locals - only its own generic parameters.

**The chain, for a name in a routine body** - from the innermost:

1. **`with` targets**, the innermost `with` first and, in `with A, B do`,
   `B` before `A` (§5.7). A target member outranks EVERYTHING below,
   locals, parameters, an inline `var` declared in the body itself,
   `Result` and `Self` included. A target after the first is itself looked
   up inside the targets before it.
2. **The routine's own block, then the enclosing routines' blocks
   outward**: inline vars (from their declaration), locals, local
   constants and types, parameters, `Result` in a function (a field named
   `Result` in the method's class does not hide it - probed 2026-10-07,
   `x-f49\order`: `Result := 5` in `function TK.R: Integer` compiles beside
   a `Result: string` field), nested routines. A nested routine sees its
   enclosing routine's locals ahead of Self's members (probed: an
   enclosing method's local `V` over the field `Self.V`).
3. **In a method - and in a routine nested in one - Self's type**: the
   active class or record HELPER for the type first (§15.3.3: probed, a
   helper's `Own` over the class's own `Own`), then the type's own
   members, then its ancestors' up to TObject (an `object`'s up to its last
   written ancestor - it has no implicit root, §11.5), each ancestor's helpers by
   §15.3.3.
4. **The unit's own declarations**: unit level of the implementation and
   the interface, enum values of its enumerations (and of nested ones -
   §2.2.4).
5. **In a method of a NESTED type, the outer types' members**, the
   innermost outer type first (probed: in `TA.TB.TC.M` a bare `N` is
   `TB.N` over `TA.N`), each with its ancestry. Step 4 counts only what the
   unit has declared ABOVE the reference: a unit constant or routine
   declared below the method does not hide the outer member (probed
   2026-10-07, `x-f52\probe` UP: `KA` is TOuter's with the unit's `KA`
   after the method), while one declared between the outer type and the
   method does - in the method's `var` section too (UQ3: `L: TX` is the
   unit's `TX = Integer` declared after TOuter, not `TOuter.TX = string`).
   A GENERIC type's own type parameters rank at this step as well, not
   with the type's members in step 3: in a method body of `TGen<T>`, a
   unit-level type `T` declared above the body is what `T` means (UG,
   UG2 - see the declaration rules below). A generic METHOD's own type
   parameters do not: with a unit `U = Int64` above, `TK.Sz<U>`'s `var L:
   U` and `Sz2<U>(A: U)`'s `A` are the parameter (2 bytes for `Sz<Word>`;
   `x-f53probe` P4).
6. **The used units**: the implementation section's `uses` before the
   interface section's (probed 2026-10-07: `F` of a unit in the
   implementation `uses` over `F` of one in the interface `uses`), and in
   each clause the LAST entry first. An undotted entry's unit name is an
   identifier of this step and hides a routine of that name in a used unit
   (§1.2.3: `uses Log` makes `Log('a')` `E2029`); a dotted entry declares
   no identifier.
7. **`System`, then `SysInit`** (§1.2.4) - the predefined names and the
   intrinsics are `System`'s, so EVERY step above hides them (B.4: a local
   `Length`, a used unit's `procedure Low;`, a type `Text = Integer` of a
   used unit).

`E2003 Undeclared identifier` only when all of it misses.

**Outside a routine body - in a structured type's DECLARATION** (a field's
type, a class constant's value, a method's parameter and result types, and
the same heading repeated on the method's implementation) the chain is
steps 4 to 7 with the TYPE's own members in step 5's place - after the
unit's declarations made so far, not before them as in a method body.
dcc64 37.0, probed 2026-10-07 (`x-f52\probe` UQ, UQ4, UR):

- with a unit-level `TX = Int64` and `KX = 10` above `TOuter = class type
  TX = Byte; const KX = 1; C = KX; var F: TX; procedure P(A: TX); end`,
  `F` is an Int64, `C` is 10 and `P`'s `A` an Int64 (declaration and
  implementation heading alike), while a local `L: TX` in a method BODY of
  TOuter is the Byte; without the unit's declarations all are TOuter's;
- a nested type's declaration the same: `TInner = class F: TX; end` inside
  TOuter takes the unit's `TX` declared above TOuter, TOuter's `TX` when
  the unit's is declared after TOuter;
- the type's ANCESTORS' members are not visible from its own declaration at
  all: in `TDer = class(TBase)`, a field `G: TX` or a constant `C = KB`
  naming TBase's nested type or constant is `E2003` (the method body sees
  them - step 3), while a type NESTED in TDer sees them (§11.4.1: `TSub =
  class F: TX; end` inside TDer is TBase's `TX`);
- a GENERIC type's parameters rank there too: a unit-level `T = Int64`
  declared above `TGen<T> = class F: T; end` makes `F` an Int64 for
  `TGen<Word>` - in the method bodies as well (`var L: T` is 8 bytes), so a
  type parameter named like a unit type declared above the generic is that
  unit type everywhere in it (`x-f52\probe` UG). "So far" is literal: with
  the unit's `T = Int64` declared BELOW TGen, `F` is the parameter (2 bytes
  for `TGen<Word>`) and a method body written below that declaration still
  takes the unit's `T` (8 bytes) - UG2.
- the type's own members still come BEFORE the used units and System
  (`x-f53probe` P1: with a used unit's `TX = Int64` and `KX = 10`,
  TOuter's own `TX = Byte`, `KX = 1` and even a nested `Integer = Word`
  are what its field types and constants name) - only the unit's own
  declarations made so far are put ahead of them;
- the type's ancestors being out of reach, a USED unit's declaration is
  what such a name means: `G: TY` and `const C = KB` in `TDer =
  class(TBase)` take a used unit's `TY = Int64` and `KB = 70` over TBase's
  `TY = Byte` and `KB = 7`, whichever order the `uses` lists them in (P8);
  while a type NESTED in TDer takes TBase's - an outer type's ancestry
  ranks before the used units (P11: `TDer.TSub`'s `S: TY` is TBase's
  Byte with the used unit's `TY = Int64` beside it);
- "the same heading repeated on the implementation" is literal too: a
  unit-level `TX = Int64` declared between `TOuter = class type TX = Byte;
  procedure P(A: TX); end` and `procedure TOuter.P(A: TX)` - after TOuter
  in the interface, or in the implementation - makes the implementation's
  `A` the unit's Int64 against the declaration's Byte: `E2037 Declaration
  of 'P' differs from previous declaration` (P3);
- records alike (`TR = record type TX = Byte; var F: TX; end` below a unit
  `TX = Int64`: `F` is 8 bytes), and so are a parameter's DEFAULT value
  (`function D(A: Integer = KX)` is 10), a property's TYPE (`property PP:
  TX read FF` is an Int64; an ancestor's `TY` there is `E2003`) and the
  values of its `index` and `default` specifiers (`index KX`, `default KX`
  are 10) - P5, P9, P10;
- but NOT the names of a property's ACCESSORS - `read`, `write`,
  `stored`: those are member lookups, the type's own and its ancestors'
  members first (`property P: Integer read F` is the field `F` with a
  unit variable `F` declared above the type; `read FB` naming an
  ancestor's field compiles) - P6, P7.
  The same for the target of a method resolution clause (`function
  IGet.Get = GetIt;` names an inherited `GetIt` - `x-f53ixture` PE).

**After a dot** (`A.B`), `B` is looked up in what `A` denotes, by the same
first-hit rule: a unit's interface (plus the intrinsics for `System`, plus
the implementation for the unit naming itself - §1.2.3); a type's or an
instance's members - the type's active helper first (probed: `K.Own` is
the helper's), then the type, then its ancestry - with the same visibility filter; an enum type's values.
A `{$IF}` designator follows the same rule, the own unit's name by position
(§1.3.2).

**The steps in detail**

*Step 3: Self's ancestry before the unit.*

- ⚠️ *In a method body the class's members - its ANCESTORS' included - come
  before the unit scope, whatever the unit declares.* dcc64 37.0, probed
  2026-10-06: an inherited routine, field or method outranks a routine, a
  constant or a function of the method's own unit (`Kind = 0` with an
  inherited `Kind: Integer` beside a unit constant `Kind = 'x'` compiles), and
  `ToString` in a method reaches TObject's over a unit constant `toString`; a
  nested class's ancestor's member outranks the OUTER type's own member of the
  name. An `overload` on the inherited member does not take the search on to
  the unit's routines: `Pick(5)` against an inherited `Pick(A: Boolean);
  overload` beside a unit-level `Pick(A: Integer); overload` is E2250, the
  ancestor in the same unit or another (contrast 6.3.1's search on through the
  USED units). A resolver that joins only the class's own members into the
  body and falls to the unit scope binds the unit's declaration wherever the
  class itself lacks the name (System.Classes' `WriteError` for TStream's).

*The visibility filter.*

- ⚠️ *A member the code cannot see is passed over, not reported.* A
  descendant in another unit calling bare `Foo`, where the direct ancestor's
  `Foo` is `private`, runs the grandancestor's public `Foo`; with none, the
  unit's or a used unit's `Foo` - never E2361. A `strict private` member is
  passed over in a descendant of the same unit too (plain `private` is seen
  there - ch.11's friend rule). An overload set is passed over only when none
  of it is visible: TThread's `Synchronize` leads with a private overload and
  a descendant's `Synchronize(M)` calls the protected one. dcc64 37.0, probed
  2026-10-06. The VCL relies on it: a bare `Insert` in TWinControl
  descendants of Vcl.ComCtrls and Vcl.NumberBox is not TWinControl's private
  `Insert`.

*Step 5: the outer types, and dcc's lookup history.*

- ⚠️ *In a method of a NESTED type, the OUTER type's members come after the
  unit's own declarations and before the used units.* dcc64 37.0, probed
  2026-10-07: in `TOuter.TInner.M` (TInner's ancestry lacking the name) a
  bare name is the unit's own routine, constant or implementation-section
  constant over an instance method, class method or constant of TOuter
  (`const KO = 3` in TOuter loses to the unit's `KO = 30`); with no unit
  declaration of the name, TOuter's member - its own or an ancestor's, in
  this unit or another - outranks a used unit's routine and a System
  declaration (`MaxInt = 7` in TOuter is 7). An INSTANCE member of the outer
  type reached so is `E2124 Instance member inaccessible here`, not passed
  over - nothing to call it on.
  - ⚠️ *Lookup HISTORY changes that answer, and the text before the method
    decides it.* When the unit has already written the name BARE earlier,
    outside the nested type's methods, and it resolved there to a used
    unit's declaration or to an intrinsic, the nested method's bare name is
    that declaration too, past the outer type's member. Probed: `SS(nil)`
    (a routine of a used unit) in a routine above `TOuter.TInner.M` makes
    M's bare `SS` that routine, where without it `SS` is TOuter's inherited
    instance method and E2124; the same earlier use written `UO.SS`
    qualified, or placed AFTER M, changes nothing. `Default(X)` is the same
    with the intrinsic against an outer class property `Default`
    (System.Threading's TThreadPool), and so is a CLASS method of the outer
    type's ancestor - which compiles either way, so the difference shows at
    run time: `LaCS` is the ancestor's class function (7) in a nested method
    with no earlier bare use, a used unit's `function LaCS: Word` (1000)
    after `LaCS;` in a routine above; `Length('abc')` the ancestor's `class
    function Length` before a bare `Length('ab')` in a routine, the
    intrinsic (3) after it (dcc64 37.0, 2026-10-08). The VCL depends on it:
    `TScrollBarStyleHook.TScrollWindow.WMPaint`'s `StyleServices` is
    Vcl.Themes' function only because Vcl.StdCtrls calls
    `StyleServices(Self)` bare earlier - its outer type's ancestor TStyleHook
    has an instance `StyleServices`. A resolver ranking by scope alone binds
    the outer member there; one that rewrites the earlier calls qualified
    turns a compiling unit into an E2124.

*Names looked up by another rule than the order above.*

- ⚠️ *An attribute's name takes the `Attribute` suffix first* (§19.3.2): `[X]`
  is `XAttribute` wherever the whole order finds it, the bare `X` only when it
  finds none. A name in the first bracket of an interface body is looked up
  the same way, and what it finds makes it the first member's attribute or
  the GUID (§14.1.1, dcc64 37.0, probed 2026-10-14); so is the name after a
  `Result:` target (`[Result: Tag]`), while after any other target word
  nothing is looked up.
