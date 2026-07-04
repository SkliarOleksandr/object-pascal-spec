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
- *AST:* one `VarDecl { names[], type, init? }` (or split per name).

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
  name.
- ⚠️ *Hint directives may sit BETWEEN the type and the initializer:*
  `Default8087CW: Word platform = $033F;` (System.pas). Parse hints in both
  positions.
- ⚠️ *Procedural-type variables put the calling convention after the `;`, and
  may still carry an initializer after it:*
  `SSL_COMP_free_compression_methods : procedure; cdecl = nil;`
  (IdSSLOpenSSLHeaders.pas).
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
   Multiple names are legal: `var V, S: string;` — System.SysUtils.pas *)
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
- *Type inference:* with `:=` and no `: TypeRef`, the type is the static type of the
  initializer expression. `var X := 1` ⇒ `Integer`.
- *AST:* `InlineVarDecl { name, type?, init?, pos }` as a statement node.

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
  routines).
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
  Max = 100;            // Integer
  Pi  = 3.14159;        // Extended
  Hi  = 'Hello';        // string
```

**Semantics & parsing notes**

- The RHS must be evaluable at compile time (B.10), which includes constant
  folding of operators, `Ord`, `Length` of string consts, set constructors, etc.
- *Constant type:* inferred from the expression (numeric literals take the
  smallest fitting type unless the expression forces wider).
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
- ⚠️ *String-literal → `TGUID` magic:* a typed constant (or initialized variable)
  of type `TGUID` may be initialized from a **GUID string literal** —
  `const IID_IFoo: TGUID = '{00000000-…-000046}';` — a compiler-special implicit
  conversion that exists *only* in constant-initializer position. Used throughout
  the D13 sources (`Vcl.AxCtrls`, tethering, Winapi headers). The parser sees an
  ordinary string initializer; the constant-evaluator must handle the conversion.
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
  with a later inline `var` is legal and position-dependent.
- *Globals* (unit `var`) live for program duration; `interface`-section vars are
  visible to importing units, `implementation`-section vars are unit-private (ch.01).
- *Resolution order* for an unqualified identifier: innermost block scope →
  enclosing scopes → `with` targets (see [05](05-statements.md) §5.7) → unit scope
  → used units (last-`uses`-wins). The parser builds scopes; resolution is a
  separate pass, but it depends on **declaration position** for inline vars.
- *AST:* no dedicated node; scope tables are built from declaration nodes.
