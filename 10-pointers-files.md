# 10 — Pointers & File Types

Typed and untyped pointers (and the dual role of `^`), plus the legacy `file`
types. Pointer *operators* (`@`, `^`, pointer math) are in
[ch.04 §4.8](04-expressions-operators.md#48-pointer--address-operators).

Shared productions (`TypeRef`) → [Appendix B](B-lexical-grammar.md).

## Chapter grammar umbrella

```ebnf
PointerType = "^" TypeRef ;
FileType    = "file" [ "of" TypeRef ] ;
```

---

## 10.1 Pointers

### 10.1.1 Typed pointers (`^T`)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A pointer to a value of a specific type.

**Grammar**

```ebnf
PointerType = "^" TypeRef ;
```

**Example**

```pascal
type
  PInteger = ^Integer;
  PNode    = ^TNode;
  TNode    = record Next: PNode; Value: Integer; end;
```

**Semantics & parsing notes**

- ⚠️ *Prefix `^` is a type constructor* (`^T`), distinct from the **postfix** `^`
  dereference (`p^`, ch.04). The parser disambiguates by position: `^` before a
  type → pointer type; `^` after an expression → dereference.
- ⚠️ *Forward type reference allowed:* `PNode = ^TNode` may reference `TNode`
  **before** it is declared, within the same `type` section. The resolver must do
  a two-pass / deferred binding for pointer targets in a type block.
  The window is that ONE `type` section: a target declared in a later
  `type` section of the same block is `E2003 Undeclared identifier:
  'TNode'` at the pointer, and dcc then also reports the later declaration
  as `E2004 Identifier redeclared: 'TNode'`. A target that is never declared
  is `E2003` as well. The target may be any type, a class included (`PFoo =
  ^TFoo; TFoo = class ... end;`). dcc64 37.0, 2026-10-30 (spec audit g4,
  local/probe-park/spec-audit/g4 PFwd.dpr, PFwd2.dpr).
- ⚠️ *Indexing a pointer to an array works WITHOUT the `^`* — `P: ^TSomeArray;
  ... P[I]` is legal and means `P^[I]`, dcc-verified both as an expression and
  as a `with` target, no `{$POINTERMATH}` required (that directive only
  extends the same courtesy to `PInteger`/`PByte`/... and other non-array
  typed pointers, §4.8). The element type is the array's, exactly as if the
  `^` had been written. Real code relies on this routinely (`ImageData[i]`
  over a `pPixelLine = ^TPixelLine` in image-decoding libraries; a
  `Stack: ^TStackArray` local in a regex engine's bracket-matching pass), and
  it holds whether the pointer type has a NAME of its own (`PSomeArray =
  ^TSomeArray`) or is written inline on the variable (`Stack: ^TStackArray`,
  no separate pointer-type declaration at all) — a resolver that only chases
  the array through a NAMED pointer type answers nothing for the inline
  spelling, and every member reached through such an indexed `with` reads as
  undeclared.
- ⚠️ *Selecting a member through a pointer works WITHOUT the `^` too* -
  `P: ^TRec; ... P.Id` means `P^.Id` for a pointer to a record or an
  `object`, whether the pointer type is named or written inline on the
  variable (System.Zip's `pField: ^TExtraField`, then `pField.FieldId`), and
  under `{$X-}` as well: it is not part of the extended syntax (dcc64 37.0,
  probed 2026-10-08). A resolver must look the member up in the POINTEE; for
  the inline spelling the pointee exists only as the declaration's type node.
- *AST:* `PointerType { targetType }`.

### 10.1.2 Untyped `Pointer`

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Pointer` is the generic untyped pointer; `nil` is the null pointer literal.

**Semantics & parsing notes**

- `Pointer` is assignment-compatible with any typed pointer in BOTH
  directions, no cast needed (`G := C` and `F := G` with `G: Pointer`, `C:
  PInteger`, `F: ^Byte` compile). `nil` (reserved word, §B.4.1) is a literal
  valid for any pointer/class/interface/procedural type.
- ⚠️ *Typed pointers are NOMINAL under the default `{$T-}`* (like arrays,
  8.1.1): every `^T` written in the source is its own type. `PInt2 = ^Integer`
  and System's `PInteger` are `E2010 Incompatible types: 'PInteger' and
  'PInt2'` in assignment and in `=` alike; two variables declared on
  separate lines as `A: ^Integer; X: ^Integer;` are `E2008 Incompatible
  types` (`A, B: ^Integer;` in ONE list share the type and do assign); an
  alias `PInt3 = PInteger` is the same type. dcc names an inline pointer
  type `Pointer` in messages (`E2010 Incompatible types: 'Pointer' and
  'PInteger'` for `A := C`). Under `{$T+}` (`{$TYPEDADDRESS ON}`) pointer
  types to the SAME base type become assignment-compatible and comparable
  (all the above compile), while pointers to different bases stay apart
  (`C := @B` with `B: Byte` is E2010) and a `var` parameter still needs the
  identical type (`E2033 Types of actual and formal var parameters must be
  identical: PInteger and PInt2`). dcc64 37.0, 2026-10-30 (spec audit g4,
  local/probe-park/spec-audit/g4 PCompat.dpr, PCompat2.dpr, PCompat3.dpr,
  PCompat4.dpr).
- Of the predefined typed pointers only `PByte` and the character pointers
  (`PChar`, `PWideChar`, `PAnsiChar`) carry `{$POINTERMATH}` semantics on their
  own (ch.04 §4.8); `PWord`, `PShortInt`, `PInteger`, `PCardinal`,
  `PUInt64`, `PNativeInt`, `PDouble` do not (`W + 1` is `E2015`; dcc64 37.0,
  2026-10-30, spec audit g4, local/probe-park/spec-audit/g4 PArith6.dpr).
  See 10.1.4.

### 10.1.3 Manual allocation intrinsics

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`New`/`Dispose` (typed), `GetMem`/`FreeMem`/`ReallocMem` (raw) manage heap
storage for pointers.

**Example**

```pascal
var P: PNode;
begin
  New(P);
  try
    P^.Value := 1;
  finally
    Dispose(P);
  end;
end;
```

**Semantics & parsing notes**

- These are compiler intrinsics (ch.04 §4.11) taking the pointer **by reference**
  (the argument must be an lvalue). `New`/`Dispose` also init/finalize managed
  fields of the pointed-to type. Detailed memory model → [ch.20](20-memory-management.md).

### 10.1.4 What each pointer kind supports (comparison, `Inc`/`Dec`, arithmetic)

| | |
|---|---|
| **Introduced** | Pascal; `{$POINTERMATH}` Delphi 2009 |
| **Deprecated** | - |
| **Status** | ✅ Current |

The operator rules of §4.8 split by pointer kind; a type checker needs the
whole matrix, since the same expression is legal on one pointer and E2015 on
the next.

**Semantics & parsing notes**

- `=` and `<>` work on every pointer (typed, untyped, `nil`), subject to the
  identity rule of 10.1.2.
- `Inc(P)` / `Dec(P)` / `Inc(P, N)` work on EVERY typed pointer with no
  directive, stepping by `SizeOf` of the pointee (`Inc(P)` on a `PInteger`
  advances 4 bytes, `Inc(P, 2); Dec(P)` a further 4). On an untyped
  `Pointer` they are `E2001 Ordinal type required`.
- `<`, `<=`, `>`, `>=`, `P + N`, `N + P`, `P - N` and `P - Q` need a pointer
  with pointer math: `PByte`, `PChar`/`PAnsiChar`/`PWideChar` always, any
  other typed pointer only under `{$POINTERMATH ON}`. Elsewhere each is
  `E2015 Operator not applicable to this operand type` (`P < Q` and `P - Q`
  over two `PInteger` with the directive off). With it, `P - Q` is the
  distance in ELEMENTS (3 for `@A[3] - @A[0]` over Integers; `B2 - B1` over
  `PByte` 4 bytes apart is 4) and `P + 2` advances `2 * SizeOf(pointee)`.
  An untyped `Pointer` never has pointer math: under `{$POINTERMATH ON}`
  too, `U < V` and `U + 1` are `E2015`.
- dcc64 37.0, 2026-10-30 (spec audit g4, local/probe-park/spec-audit/g4
  PArith.dpr - PArith5.dpr, PMath.dpr; run-time
  output for the step sizes).

---

## 10.2 File types

### 10.2.1 Typed, text, and untyped files

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — (legacy I/O; prefer streams) |
| **Status** | ⚠️ Legacy |

Classic Pascal file types: `file of T` (typed records), `TextFile`/`Text` (line
text), and untyped `file`.

**Grammar**

```ebnf
FileType = "file" [ "of" TypeRef ] ;
```

**Example**

```pascal
var
  F: file of TRecord;   // typed file
  T: TextFile;          // text file
  U: file;              // untyped file
```

**Semantics & parsing notes**

- `file` is a **reserved word** (§B.4.1). `file of T` is a typed file; bare `file`
  is untyped. `Text`/`TextFile` are predefined types (not a `file` form).
- File variables cannot be assigned/copied and have special parameter rules (passed
  by `var`). Operated on by the classic `Assign`/`Reset`/`Rewrite`/`Read`/`Write`
  intrinsics.
- ⚠️ *The exact restrictions and their codes:* `A := B` between two
  `TextFile`s or two `file of Integer` is `E2015 Operator not applicable to
  this operand type`; a by-value file parameter (`procedure P(F:
  TextFile)`) is `E2002 File type not allowed here`, while `var` and
  `const` parameters compile; a file function result is `E2024 Invalid
  function result type`; `file of file` is E2002. The element type of `file
  of T` must need no finalization: `file of string`, `file of Variant` and
  `file of TR` (a record holding a `string`) are `E2155 Type 'string' needs
  finalization - not allowed in file type`, while `file of ShortString` and
  `file of TObject` compile. dcc64 37.0, 2026-10-30 (spec audit g4,
  local/probe-park/spec-audit/g4 FFile.dpr).
- Listed for completeness; modern code uses `TStream` (RTL, out of scope).
- *AST:* `FileType { elementType? }`.
