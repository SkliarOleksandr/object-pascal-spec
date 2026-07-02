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
- *AST:* `PointerType { targetType }`.

### 10.1.2 Untyped `Pointer`

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Pointer` is the generic untyped pointer; `nil` is the null pointer literal.

**Semantics & parsing notes**

- `Pointer` is assignment-compatible with any typed pointer (one direction freely;
  the other needs a cast). `nil` (reserved word, §B.4.1) is a literal valid for any
  pointer/class/interface/procedural type.
- Predefined typed-pointer aliases (`PByte`, `PChar`, `PAnsiChar`, …) additionally
  enable `{$POINTERMATH}` semantics (ch.04 §4.8).

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
- Listed for completeness; modern code uses `TStream` (RTL, out of scope).
- *AST:* `FileType { elementType? }`.
