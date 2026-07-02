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
VariantField  = ConstExpr { "," ConstExpr } ":" "(" [ FieldList ] ")" ;
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
- `Implicit`/`Explicit` define conversions to/from other types — they drive
  implicit/explicit cast resolution.
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
