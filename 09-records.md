# 09 — Records

Records: value-type aggregates that have grown methods, properties, operator
overloading, and (since 10.4) custom managed lifetime. This chapter holds the
**operator-overloading declaration** syntax referenced from ch.04, and the
**managed-record** operators.

Shared productions (`TypeRef`, `ConstExpr`, field/method decls) → [Appendix
B](B-lexical-grammar.md); method bodies follow [ch.06](06-routines.md).

## Chapter grammar umbrella

```ebnf
RecordType  = [ "packed" ] "record"
                [ FieldList ]
                { RecordMember }
                [ VariantPart ]
              "end" ;
FieldList   = FieldDecl { ";" FieldDecl } [ ";" ] ;
FieldDecl   = IdentList ":" TypeRef ;
RecordMember = MethodDecl | PropertyDecl | ConstDecl | TypeDecl
             | ClassVarDecl | OperatorDecl | Visibility ;
Visibility  = [ "strict" ] ( "private" | "public" ) ;   (* records: private/public only *)
```

---

## 9.1 Record types

### 9.1.1 Simple records

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A value-type aggregate of named fields.

**Example**

```pascal
type
  TPoint = record
    X, Y: Integer;
  end;
```

**Semantics & parsing notes**

- *Value semantics:* assignment copies all fields; no reference counting (unless it
  contains managed fields, which are individually managed).
- ⚠️ *Local (stack) record variables are NOT zero-initialized — unlike globals.*
  A global/unit-level record variable's fields read as zero before any
  assignment, because static/BSS storage is zero-filled by the loader. A
  **local** record variable has no such guarantee: its fields hold whatever was
  last on that stack slot. Confirmed (dcc-verified, dcc32 37.0): a global
  `TRec` read before assignment shows `0` for every field; a local `TRec` in a
  procedure called right after another procedure that fills its own same-shaped
  local with `$7F7F7F7F` reads back `2139062143` (`$7F7F7F7F`) for every
  field — genuine leftover garbage, not a fluke of a "clean" stack. A semantic
  layer must not assume a local record reads as zero pre-assignment; only a
  record with a managed `Initialize` operator (§9.4.1) gets a real init
  guarantee on the stack.
- ⚠️ *A record type may be ANONYMOUS* — written directly in a declaration's type
  slot instead of being named, most often as an array element:

  ```pascal
  const
    TAB: packed array[0..1] of record
      offset, minimum: Cardinal;
    end = ((offset: 0; minimum: 16), (offset: 32; minimum: 48));
  ```

  Its fields are ordinary members: `with TAB[I] do offset` resolves. The
  awkward part is for a semantic layer — a type with no name cannot be
  identified by name, so an implementation that keys types by `(unit, symbol)`
  needs a SYNTHETIC symbol created where the inline type is collected. Giving it
  only a member scope is not enough: everything that answers "what type is this
  expression?" answers with a symbol, and without one the array's element type
  simply does not exist. The synthetic symbol must stay unnamed and unbound, so
  nothing can resolve to it or be shadowed by it.
- *AST:* `RecordType { fields[], members[], variantPart? }`.

### 9.1.2 `packed` records & alignment

| | |
|---|---|
| **Introduced** | `packed` Pascal; `{$ALIGN n}` Delphi-early |
| **Deprecated** | — |
| **Status** | ✅ Current |

`packed` removes field padding; `{$ALIGN}`/`{$A}` controls field alignment.

**Semantics & parsing notes**

- `packed` is a **reserved word** prefix. Alignment affects layout/`SizeOf`, not
  parsing — but record-compatibility for interop depends on it.
- ⚠️ *Semi-documented `end align N` clause:* a record may end with
  `end align 16;` — the RTL ships it with a conditional operand:
  `end align {$IFDEF CPU64BITS} 16 {$ELSE} 8 {$ENDIF};`
  (System.SysUtils.pas). The operand is a constant expression; a parser must
  accept `align` (contextual word) after the closing `end`.
- ⚠️ *The layout rule itself,* needed by anything that answers
  `{$IF SizeOf(TRec) = N}` (1.3.2) without compiling. Three steps:

  1. each field starts at the next offset that is a multiple of
     `Min(the field's own alignment, the current CAP)`;
  2. the record's **own alignment** is the largest cap-clipped field
     alignment it contains;
  3. the record's `SizeOf` is the running offset rounded **up** to that own
     alignment — so trailing padding is real
     (`record A: Integer; B: Byte; end` is 8, not 5).

  A scalar's alignment is `Min(its size, 8)`. `Extended` is the one type where
  size and alignment differ: 10 bytes on Win32, aligned to 8, so
  `record A: Byte; B: Extended; end` is 24. A nested record contributes **its
  own** alignment, not its size, and that alignment was fixed at ITS
  declaration site — a 9-byte `$A1` record inside an `$A8` record sits at
  offset 1, giving 10.
- ⚠️ *The alignment CAP is a ceiling, not a forced alignment, and it is
  positional.* `{$A1|2|4|8|16}`, the long `{$ALIGN 1|2|4|8|16}`, and
  `{$A-}`/`{$A+}` (= 1 and 8) all set it, and a unit routinely brackets one
  type with it, so the value that matters is the one in effect **at the
  record's declaration**, not at the end of the file. `packed` is a cap of 1
  for that record only.

  Because no builtin's natural alignment exceeds 8, `{$ALIGN 16}` produces
  layouts identical to `{$ALIGN 8}` — it raises a ceiling nothing reaches.
  Only 4, 2 and 1 change anything. `{$ALIGN 32}` is **not** valid (E1030);
  FastMM4 ships one, but inside an `{$IFDEF FPC}` branch.

  Measured for `record A: Byte; B: Int64; end`: cap 16 or 8 → 16, cap 4 → 12,
  cap 2 → 10, cap 1 or `packed` → 9. (All of 9.1.2's numbers here come from
  compiling ~20 record shapes for Win32 and Win64 and printing `SizeOf` at run
  time.)

### 9.1.3 Variant records (`case` part)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A record may end with a `case` part: overlapping fields sharing storage (a union),
optionally tagged.

**Grammar**

```ebnf
VariantPart   = "case" [ Ident ":" ] OrdinalType "of" VariantField { ";" VariantField } ;
VariantField  = ConstExpr { "," ConstExpr } ":" "(" [ FieldList ] [ VariantPart ] ")" ;
(* a variant branch may itself end with a nested variant part *)
```

**Example**

```pascal
type
  TShape = record
    case Kind: (skCircle, skRect) of
      skCircle: (Radius: Double);
      skRect:   (W, H: Double);
  end;
```

**Semantics & parsing notes**

- ⚠️ *Distinct grammar from a `case` statement* — the variants list field groups in
  parentheses, not statements. The leading token is `case` inside a record body
  (vs. statement context). The tag field is optional (`case OrdinalType of` with no
  name).
- ⚠️ *A named tag is a REAL FIELD, not an annotation on the type.* `case Tag:
  T of` **declares** `Tag` as an ordinary field of the record: it occupies
  storage and is freely readable and assignable (`R.Tag := 1`). dcc-verified
  both ways — the assignment compiles, and naming the tag grows `SizeOf` by
  the tag type's width (12 vs 8 for the same record with an anonymous `case
  Integer of`). A semantic layer must therefore DECLARE the name; treating it
  as a reference makes it read as an undeclared identifier (real bug, found
  on `System.Curl.pas`'s `case data: Integer of` and 7 more RTL tag names,
  one of them an *escaped* reserved word: `case &type: POINTER_INPUT_TYPE of`
  in `Winapi.Windows`).
- ⚠️ *Whether the tag is named cannot be decided from the node kind* — an
  anonymous `case Integer of` also leads with a plain identifier (the type
  name). The `:` after it is the only discriminator.
- ⚠️ *The tag type is a full `OrdinalType`, not merely a type NAME:* besides a
  named reference it may be an **inline anonymous enum** — as in the example
  above, `case Kind: (skCircle, skRect) of` — or a **subrange** (`case Tag:
  0..9 of`). A parser that accepts only a type name here fails on the `(` and
  then desynchronises: the branch LABELS get read as field names and
  `(Radius: Double)` as an enum type.
- Branch fields belong to the **enclosing record**, not to a sub-scope — every
  branch's fields, at every nesting depth, are members of the same record and
  must collect into its one member scope.
- All variant branches **overlay the same memory** (a union); no runtime checking.
- *AST:* `VariantPart { tagField?, tagType, branches: [ { labels[], fields[] } ] }`.

---

## 9.2 Records with methods

### 9.2.1 Methods, properties, class members

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Records may declare methods, properties, `class var`/`class` methods, nested
types/constants, and visibility sections — much like classes, but as a value type.

**Example**

```pascal
type
  TVec = record
    X, Y: Double;
    function Length: Double;
    class function Zero: TVec; static;
  end;
```

**Semantics & parsing notes**

- ⚠️ *No inheritance:* records have **no ancestor**, no `virtual`/`override`, and
  visibility is limited to `[strict] private` / `public` (no `protected`/
  `published`). Reject inheritance/virtual on records.
- `Self` inside a record method is the record instance (by reference for `var`-like
  methods).
- *AST:* same member nodes as classes, on a `RecordType`.

### 9.2.2 Record constructors

| | |
|---|---|
| **Introduced** | Delphi 2006 (parameterized only) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Records may have **parameterized** constructors.

**Example**

```pascal
type
  TVec = record
    X, Y: Double;
    constructor Create(AX, AY: Double);
  end;
```

**Semantics & parsing notes**

- ⚠️ *No parameterless record constructor* is allowed (classic rule) — initial
  value comes from zero-fill or a custom managed `Initialize` operator (9.4). The
  parser should reject `constructor Create;` with no parameters on a record.
- A record constructor does not allocate (value type); it initialises an instance.

---

## 9.3 Operator overloading

### 9.3.1 `class operator` declarations

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Records (not classes) may overload operators via `class operator` methods with
fixed names.

**Grammar**

```ebnf
OperatorDecl = "class" "operator" OperatorName "(" FormalParams ")" [ ":" ResultType ] ";" ;
OperatorName = "Implicit" | "Explicit" | "Negative" | "Positive" | "Inc" | "Dec"
             | "LogicalNot" | "Trunc" | "Round"
             | "Add" | "Subtract" | "Multiply" | "Divide" | "IntDivide" | "Modulus"
             | "LeftShift" | "RightShift" | "LogicalAnd" | "LogicalOr" | "LogicalXor"
             | "BitwiseAnd" | "BitwiseOr" | "BitwiseXor"
             | "Equal" | "NotEqual" | "GreaterThan" | "GreaterThanOrEqual"
             | "LessThan" | "LessThanOrEqual" | "In" ;
```

**Example**

```pascal
type
  TVec = record
    X, Y: Double;
    class operator Add(const A, B: TVec): TVec;
    class operator Implicit(const A: TVec): string;
    class operator Equal(const A, B: TVec): Boolean;
  end;
```

**Semantics & parsing notes**

- ⚠️ Operators are declared `class operator <Name>` with **named** operators
  (`Add`, `Equal`, `Implicit`…), not the symbolic tokens. Each maps to a
  symbolic/relational operator or conversion. The **use site** (ch.04) is
  unchanged; resolution dispatches to the matching `class operator`.
- ⚠️ *Operator names may be reserved words:* `class operator In(...)` and its
  implementation header `class operator TFontStyleExt.In(...)` ship in
  FMX.Graphics.pas — the name position after `operator` (and after `.` in the
  qualified form) must accept keywords.
- `Implicit`/`Explicit` define conversions to/from other types — they drive
  implicit/explicit cast resolution.
- ⚠️ *`Implicit`/`Explicit` may overload by RESULT TYPE ALONE* — a deliberate
  exception to the general routine-overloading rule (ch.06) that two overloads
  cannot differ only by return type. Two `class operator Implicit` declarations
  on the same record, with the same parameter list but different result types
  (e.g. `Implicit(A: TFoo): Integer` and `Implicit(A: TFoo): string`), both
  compile and each fires at its own cast site (dcc-verified, dcc32 37.0):
  `I := F` (target `Integer`) dispatches to the `Integer`-returning overload,
  `S := F` (target `string`) dispatches to the `string`-returning overload. The
  exception holds because the disambiguator is not the call signature but the
  **cast target type** at the use site — the same reason overload resolution
  works at all for a conversion operator. A resolver for these two operator
  kinds must therefore key candidate lookup by (param types, result type)
  jointly, unlike ordinary routine overloads which key on parameters only.
- *AST:* `OperatorDecl { opName, params[], resultType }` as a record member.

---

## 9.4 Custom managed records

### 9.4.1 `Initialize` / `Finalize` / `Assign` operators

| | |
|---|---|
| **Introduced** | 10.4 Sydney (2020) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A record can define lifetime hooks run automatically when an instance is
created, destroyed, or copied — enabling RAII-style value types (smart pointers,
scope guards).

**Grammar**

```ebnf
ManagedOp = "class" "operator" ( "Initialize" | "Finalize" )
              "(" [ ManagedSelfParam ] ")" ";"
          | "class" "operator" "Assign"
              "(" "var" Ident ":" TypeRef ";" "const" [ "[" "Ref" "]" ] Ident ":" TypeRef ")" ";" ;
ManagedSelfParam = ( "var" | "out" ) Ident ":" TypeRef ;
(* pre-13.0: explicit param required; 13.0: optional (implicit Self).
   The RTL uses the "out" form: class operator Initialize(out Dest: TLightweightMREW) *)
```

**Example**

```pascal
type
  TGuard = record
    FHandle: THandle;
    class operator Initialize(var G: TGuard);   // run on allocation
    class operator Finalize(var G: TGuard);      // run on deallocation
    class operator Assign(var Dest: TGuard; const [ref] Src: TGuard);  // run on copy
  end;
```

**Semantics & parsing notes**

- ⚠️ A record with **any** of these operators becomes a **managed type** — it gets
  automatic init/finalize/copy codegen and may no longer be a valid `case`/variant
  field or used where unmanaged layout is required. The type-checker must track
  "is managed".
- `Initialize` runs on every instance creation (locals, fields, array elements);
  `Finalize` on destruction; `Assign` replaces the default field-copy.
- ⚠️ *Parameter-passing matrix — which operators fire on a call.* Probed with a
  `TMgd` record whose `Initialize`/`Finalize`/`Assign` each print a marker
  (dcc-verified, dcc32 37.0):
  - **By value** (`procedure P(M: TMgd)`): the callee's local parameter slot
    gets `Initialize`, then `Assign` copies the caller's argument into it —
    `M` inside `P` is a genuine independent copy — and `Finalize` runs on that
    local slot when `P` returns. A by-value managed-record parameter is exactly
    as expensive as a local variable built by assignment.
  - **`const`, `var`, and `const [ref]`** (`procedure P(const M: TMgd)` /
    `var` / `const [ref]`): **none** of the three operators fire. All three
    pass a reference to the caller's own instance — no copy is made, so there
    is nothing to `Initialize`, `Assign`, or later `Finalize` on the callee's
    side.
  - **Function result of a managed-record type:** the function's hidden
    `Result` variable gets `Initialize` on entry (like any local of that type);
    on `Dest := SomeFunc`, the returned value is copied into `Dest` via
    `Assign`, and the temporary `Result` is then `Finalize`d — i.e. a
    managed-record-returning function call is, on the assignment side,
    indistinguishable from a by-value parameter pass: `Initialize` (for the
    temporary) + `Assign` (into the receiver) + `Finalize` (of the temporary).
  - A semantic/codegen layer must therefore treat "is this parameter mode
    copying" as the single switch that decides whether the three operators are
    even inserted: by-value and function-result positions copy (all three
    operators may run); `const`/`var`/`const [ref]` never copy (none run).
- ⚠️ *Managed records auto-finalize on exception unwind* — no `try-finally`
  needed. A local managed-record variable that has already had `Initialize`
  run, in a procedure that then raises before reaching its own end, still gets
  `Finalize` called on unwind, before the exception reaches its handler
  (dcc-verified, dcc32 37.0: a local `TMgd` created, then `raise
  Exception.Create(...)` immediately after — the `Finalize` marker prints
  *before* the `except` block's own output). The compiler emits the same
  implicit cleanup a `try-finally` would, scoped to every managed local in the
  unwound frames; no explicit `try-finally` is required to guarantee it, unlike
  a plain (unmanaged) object reference which leaks if not freed in a
  `finally`.
- *AST:* tag the `RecordType` `managed: true` and keep the three operator nodes.

### 9.4.2 Implicit `Self` in `Initialize`/`Finalize` (13.0)

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

From 13.0 the `Initialize`/`Finalize` operators may omit the explicit
`var Self`-style parameter; an implicit `Self` is provided.

**Example**

```pascal
type
  TGuard = record
    class operator Initialize;   // 13.0: implicit Self, no explicit param
    class operator Finalize;
  end;
```

**Semantics & parsing notes**

- ⚠️ *Version-gated grammar:* before 13.0 the explicit `(var X: TGuard)` parameter
  was **required**; from 13.0 it is **optional** and a `Self` is implied. The
  parser must accept both arities for these two operators on the target version,
  and bind `Self` to the instance when the parameter is absent.
- Applies only to `Initialize`/`Finalize` (not `Assign`, which needs its two
  explicit parameters).
