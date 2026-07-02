# 02 — Fundamental & Simple Types

The type system underpins every later chapter. This chapter covers the
predefined simple types and the simple *user-defined* type forms (enumerations,
subranges, sets, aliases). Structured types have their own chapters: strings
[07](07-strings.md), arrays [08](08-arrays.md), records [09](09-records.md),
pointers [10](10-pointers-files.md), classes [11](11-classes.md).

Shared productions (`TypeRef`, `Ident`, `ConstExpr`) are in
[Appendix B](B-lexical-grammar.md).

## Chapter grammar umbrella

```ebnf
TypeSection = "type" { TypeDecl } ;
TypeDecl    = Ident [ GenericParams ] "=" [ "type" ] TypeSpec ";" [ HintDirectives ] ;
TypeSpec    = SimpleType | StringType | StructuredType | PointerType
            | ProceduralType | "type" "of" TypeRef ;
SimpleType  = OrdinalType | RealType ;
OrdinalType = IntegerType | CharType | BoolType | EnumType | SubrangeType ;
```

> The optional leading **`type`** in `TypeDecl` (`= type T`) creates a *distinct*
> type — see 2.5.1. This is a parser-visible distinction, keep it on the node.

---

## 2.1 Type categories

### 2.1.1 The type taxonomy

| | |
|---|---|
| **Introduced** | Pascal (pre-1995), extended over time |
| **Deprecated** | — |
| **Status** | ✅ Current |

Object Pascal types divide into: **simple** (ordinal, real), **string**,
**structured** (array, record, set, file, class, interface), **pointer**,
**procedural**, **variant**, and **generic** type parameters.

**Semantics & parsing notes**

- The parser needs this taxonomy to validate where a type may appear (e.g. only
  *ordinal* types are legal as `case` selectors, array index types, `set of`
  base, and `for` counters — recurring constraint).
- *AST:* a single `TypeRef`/`TypeDecl` node with a discriminant for the category.

---

## 2.2 Ordinal types

Ordinal types have a discrete, ordered set of values with `Ord`, `Pred`, `Succ`,
`Low`, `High`, `Inc`, `Dec` defined. They are the only types allowed as set
bases, array indices, `case` selectors, and `for` counters.

### 2.2.1 Integer types

| | |
|---|---|
| **Introduced** | core types Pascal/D1; `Int64` Delphi 4; `UInt64`/`NativeInt`/`NativeUInt` 2009-era; `FixedInt`/`FixedUInt` XE8; `NativeInt` **weak alias** 12 Athens |
| **Deprecated** | — |
| **Status** | ✅ Current |

Predefined integer types of fixed or platform-dependent width.

**Grammar**

```ebnf
IntegerType = "ShortInt" | "SmallInt" | "Integer" | "LongInt" | "Int64"
            | "Byte" | "Word" | "Cardinal" | "LongWord" | "UInt64"
            | "NativeInt" | "NativeUInt" | "FixedInt" | "FixedUInt" ;
(* these are predefined identifiers, NOT reserved words *)
```

**Example**

```pascal
var
  I: Integer;     // 32-bit signed
  B: Byte;        // 8-bit unsigned
  N: NativeInt;   // pointer-sized (32 or 64)
```

**Semantics & parsing notes**

- Integer type names are **predefined identifiers**, not keywords — they live in
  `System` and can in principle be shadowed (B.4). The lexer emits them as
  identifiers; resolution is semantic.
- ⚠️ *Platform-dependent widths:* `NativeInt`/`NativeUInt` are 32-bit on 32-bit
  targets and 64-bit on 64-bit targets. `LongInt`/`LongWord` are 32-bit on all
  current targets; `FixedInt`/`FixedUInt` are guaranteed 32-bit. A parser doesn't
  need the width, but a *semantic/type* layer does — record the nominal name and
  resolve width per target.
- *Literal typing:* an integer literal takes the smallest predefined type holding
  it (see B.5.1).

### 2.2.2 Boolean types

| | |
|---|---|
| **Introduced** | `Boolean` Pascal; `ByteBool`/`WordBool`/`LongBool` D1 (interop) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Boolean` plus three interop variants of different sizes.

**Grammar**

```ebnf
BoolType = "Boolean" | "ByteBool" | "WordBool" | "LongBool" ;
```

**Semantics & parsing notes**

- `Boolean` has ordinal values `False=0`, `True=1`. The `*Bool` types treat any
  non-zero as `True` (interop with C/WinAPI) — relevant to constant folding, not
  parsing.
- Conditions in `if`/`while`/`until`/`for-in` guards require a Boolean type — no
  integer-to-Boolean coercion.

### 2.2.3 Character types

| | |
|---|---|
| **Introduced** | `AnsiChar`/`WideChar` D1/D2; **`Char` = `WideChar` since Delphi 2009** (was `AnsiChar` before) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Char` is the default character type — 2-byte `WideChar` (UTF-16 code unit) since
the 2009 Unicode migration.

**Grammar**

```ebnf
CharType = "Char" | "AnsiChar" | "WideChar" | "UCS2Char" | "UCS4Char" ;
```

**Semantics & parsing notes**

- ⚠️ *Version-sensitive aliasing:* before Delphi 2009 `Char = AnsiChar` (1 byte);
  from 2009 `Char = WideChar` (2 bytes). A spec-accurate parser targeting multiple
  versions must resolve `Char` per target version.
- A single-quote 1-char literal is `Char`-compatible (B.6.1).

### 2.2.4 Enumerated types

| | |
|---|---|
| **Introduced** | Pascal; **explicit ordinal values** D-early; **`{$SCOPEDENUMS}`** ~2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A type defined by listing its named values; values are ordinals `0,1,2,…` unless
explicit values are assigned.

**Grammar**

```ebnf
EnumType   = "(" EnumElem { "," EnumElem } ")" ;
EnumElem   = Ident [ "=" ConstExpr ] ;
```

**Example**

```pascal
type
  TSuit  = (Clubs, Diamonds, Hearts, Spades);
  TError = (errNone = 0, errIO = 10, errFatal = 99);  // explicit values
```

**Semantics & parsing notes**

- *Explicit values* make the enum *non-contiguous* (a "sparse" enum). Sparse enums
  cannot be used as array index/`set of` bases — semantic constraint.
- ⚠️ *Scope of element names:* by default enum element identifiers are injected
  into the **enclosing scope** (so `Clubs` is directly visible). With
  `{$SCOPEDENUMS ON}` they must be qualified (`TSuit.Clubs`). This changes
  name resolution — the parser/semantic layer must honour the directive state at
  the type's declaration site.
- *AST:* `EnumType { elements: [ { name, explicitValue? } ], scoped: bool }`.

### 2.2.5 Subrange types

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A type restricted to a contiguous sub-range of an existing ordinal type.

**Grammar**

```ebnf
SubrangeType = ConstExpr ".." ConstExpr ;
```

**Example**

```pascal
type
  TDigit = 0..9;
  TUpper = 'A'..'Z';
```

**Semantics & parsing notes**

- ⚠️ *Parser ambiguity with sets/case:* `lo..hi` ranges appear in subrange types,
  set constructors (B.9), `case` labels, and array bounds. The `..` token is the
  marker; the surrounding context determines the production.
- Bounds must be compile-time constants of the same ordinal base type.
- *AST:* `SubrangeType { lo, hi, baseType }`.

---

## 2.3 Real (floating-point) types

### 2.3.1 Predefined real types

| | |
|---|---|
| **Introduced** | Pascal/D1; `Real` redefined to `Double` (D2+); `Real48` = legacy 6-byte |
| **Deprecated** | `Real48` discouraged |
| **Status** | ✅ Current |

Floating-point and fixed-scaled numeric types.

**Grammar**

```ebnf
RealType = "Real" | "Single" | "Double" | "Extended" | "Comp" | "Currency" | "Real48" ;
```

**Semantics & parsing notes**

- ⚠️ *`Extended` is platform-dependent:* 10 bytes on Win32, but **8 bytes (= `Double`)
  on Win64, macOS, and mobile**. Code relying on 80-bit precision is not portable —
  a semantic layer must resolve `Extended` per target.
- `Currency` is a 64-bit integer scaled by 10 000 (4 decimal places) — exact
  decimal arithmetic for money; not a binary float.
- `Comp` is a legacy 64-bit integer-valued float; avoid in new code.
- Real literals (B.5.2) default to `Extended`/`Double`.

---

## 2.4 Set types

### 2.4.1 `set of` ordinal

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A bitset over the values of a small ordinal base type.

**Grammar**

```ebnf
SetType = "set" "of" OrdinalType ;
```

**Example**

```pascal
type
  TSuits = set of TSuit;
var
  Hand: TSuits;
begin
  Hand := [Clubs, Spades];
  if Clubs in Hand then ...;
end;
```

**Semantics & parsing notes**

- ⚠️ *Base-type limit:* the base ordinal type must have **no more than 256
  possible values**, and ordinal values must be in `0..255`. Larger ranges are a
  compile error. Enforce semantically.
- Set *values* are written with the set-constructor `[ … ]` (B.9), which collides
  syntactically with array indexing — disambiguated by position.
- Operators: `+` (union), `-` (difference), `*` (intersection), `in`,
  `=`/`<>`/`<=`/`>=` (subset/superset). See ch.04.
- *AST:* `SetType { baseType }`.

---

## 2.5 User-defined simple types & aliases

### 2.5.1 Type aliases — weak vs. distinct

| | |
|---|---|
| **Introduced** | Pascal; distinct-type `type T = type Base` D-early; **`NativeInt` as weak alias** 12 Athens |
| **Deprecated** | — |
| **Status** | ✅ Current |

`type T = Base` introduces a **weak alias** (same type identity for assignment;
shares RTTI). `type T = type Base` introduces a **distinct type** (separate type
identity and RTTI, still assignment-compatible with the base in most contexts).

**Grammar**

```ebnf
(* the optional leading "type" in TypeDecl marks a distinct type *)
TypeDecl = Ident "=" [ "type" ] TypeSpec ";" ;
```

**Example**

```pascal
type
  TMyInt  = Integer;        // weak alias — same type as Integer
  TFooId  = type Integer;   // distinct type — own RTTI, own overload identity
```

**Semantics & parsing notes**

- ⚠️ *Keep the `type` marker on the AST.* The distinct-vs-weak choice affects
  **overload resolution** (a distinct type can select a different overload),
  **RTTI** (distinct types get their own type info), and implicit conversions.
- The leading `type` here is the reserved word `type` used as a *modifier*, not a
  section header — distinguish by position (inside a `TypeDecl` RHS).
- *AST:* `TypeAlias { name, baseType, distinct: bool }`.

### 2.5.2 Hint directives on type declarations

| | |
|---|---|
| **Introduced** | `deprecated`/`platform`/`library` Delphi 6/7; `experimental` later |
| **Deprecated** | — (these *produce* deprecation) |
| **Status** | ✅ Current |

Declarations may carry hint directives that the compiler reports on use.

**Grammar**

```ebnf
HintDirectives = { "deprecated" [ StringLiteral ] | "platform" | "library" | "experimental" } ;
```

**Example**

```pascal
type
  TOld = Integer deprecated 'use TNew';
```

**Semantics & parsing notes**

- These are **directives** (B.4.2), legal as identifiers elsewhere; here they
  trail a declaration. `deprecated` may take an optional message string.
- Relevant to this reference's own **Deprecated** field: this is the mechanism by
  which the compiler marks something deprecated.
- *AST:* attach `hints[]` to the declaration node.

---

## 2.6 Type identity & compatibility

### 2.6.1 Identity, compatibility, assignment-compatibility

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Three distinct relations the semantic layer must implement: **type identity**
(same declared type), **compatibility** (operable together), and
**assignment-compatibility** (RHS may be assigned to LHS).

**Semantics & parsing notes**

- *Not a parsing concern but a semantics one* — listed so the type-checker built on
  this AST has a defined contract:
  - Two named types are identical only if they originate from the **same
    declaration** (structural equivalence is *not* used for named types). This is
    why `type T = type Integer` differs from `Integer`.
  - Subrange types are compatible with their base and with overlapping subranges.
  - Numeric types are mutually compatible with implicit widening; narrowing needs
    a cast (ch.04).
- *AST:* no node; this drives the type-checker pass.
