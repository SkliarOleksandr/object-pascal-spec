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
  initialize multiple variables` (dcc 37.0).
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
- *AST:* `InlineVar { name, type?, init?, pos }` as a statement node.

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
  smallest fitting type unless the expression forces wider) — the same literal
  rules as an inline `var` (3.1.3): `100` ⇒ Integer, `5000000000` ⇒ Int64,
  `'a'` ⇒ Char, `'abc'` ⇒ string, `1.5` ⇒ Extended on Win32 / Currency on
  Win64 (dcc 37.0).
- ⚠️ *A true constant HAS that type for member access:* `const CI = 100;`
  then `CI.ToString`, and `const CS = 'abc';` then `CS.Length`, both compile
  (dcc 37.0) — the helper is looked up on the inferred type, so a resolver
  must record it rather than treat the constant as untyped.
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
- *Resolution order* for an unqualified identifier: innermost block scope →
  enclosing scopes → `with` targets (see [05](05-statements.md) §5.7) → unit scope
  → used units (last-`uses`-wins). The parser builds scopes; resolution is a
  separate pass, but it depends on **declaration position** for inline vars.
- *AST:* no dedicated node; scope tables are built from declaration nodes.
