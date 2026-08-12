# 08 — Arrays

Static, dynamic, and multidimensional arrays, open-array parameters (recap from
ch.06), and array constructors/operations. Dynamic arrays are managed,
reference-counted types.

Shared productions (`TypeRef`, `OrdinalType`, `ConstExpr`, `Expression`) →
[Appendix B](B-lexical-grammar.md).

## Chapter grammar umbrella

```ebnf
ArrayType   = "array" [ "[" IndexTypes "]" ] "of" ( TypeRef | "const" ) ;
IndexTypes  = IndexType { "," IndexType } ;
IndexType   = OrdinalType | SubrangeType ;       (* presence => static; absence => dynamic *)
```

---

## 8.1 Static arrays

### 8.1.1 Single-dimension static arrays

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A fixed-length array indexed by an ordinal/subrange type; the index type defines
both bounds and length.

**Grammar**

```ebnf
StaticArrayType = "array" "[" IndexTypes "]" "of" TypeRef ;
```

**Example**

```pascal
type
  TWeek = array[1..7] of string;
  TByEnum = array[TSuit] of Integer;   // indexed by an enum
```

**Semantics & parsing notes**

- ⚠️ *Index type must be ordinal* (integer subrange, char, enum, boolean). The
  bounds come from the index type's `Low`/`High` — not a separate length.
- Static arrays are **value types** (copied on assignment, no reference counting).
- ⚠️ *Array compatibility is NOMINAL, not structural.* Two independently-declared
  array types with identical shape and element type are still two distinct,
  mutually incompatible types — direct assignment between them is rejected:
  `T1 = array[1..5] of Integer; T2 = array[1..5] of Integer;` then `V1 := V2`
  (both static, no cast, no `type ... = type ...` equivalence declaration) is
  `E2010 Incompatible types: 'T1' and 'T2'` (dcc-verified, dcc32 37.0). A
  resolver must key array-type identity by the declared type symbol, never by
  a structural shape comparison (index bounds + element type) — two shapes that
  print identically are still not the same type unless one is declared as the
  other (`T2 = T1;`, a true alias) or the check is a subtype-in-context case
  like an open-array parameter.
- *AST:* `StaticArrayType { indexTypes[], elementType }`.

### 8.1.2 Multidimensional static arrays

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Multiple index types, or arrays of arrays (equivalent forms).

**Example**

```pascal
type
  TMatrix = array[1..3, 1..3] of Double;   // == array[1..3] of array[1..3] of Double
```

**Semantics & parsing notes**

- `array[A, B] of T` is **sugar** for `array[A] of array[B] of T`; the parser may
  normalise to nested single-dimension nodes or keep the index list.
- Element access `M[i, j]` likewise normalises to `M[i][j]`.

---

## 8.2 Dynamic arrays

### 8.2.1 Dynamic array types

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

An `array of T` **without** an index type: a managed, reference-counted,
heap-allocated, resizable, **0-based** array.

**Grammar**

```ebnf
DynamicArrayType = "array" "of" TypeRef ;     (* no "[" IndexTypes "]" *)
```

**Example**

```pascal
var A: array of Integer;
begin
  SetLength(A, 5);     // length set at runtime; A[0]..A[4]
  A[0] := 1;
end;
```

**Semantics & parsing notes**

- ⚠️ *Presence/absence of `[IndexTypes]` is the whole distinction* between a static
  (value, fixed, custom-based) and a dynamic (managed, runtime, 0-based) array.
  The parser branches on it.
- *Always 0-based:* `Low(A) = 0`, `High(A) = Length(A)-1`. Independent of
  `{$ZEROBASEDSTRINGS}` (that affects strings only).
- *Managed:* assignment shares the reference (copy-on-write via `SetLength`/
  `Copy`); lifetime automatic.
- `SetLength`, `Length`, `Copy`, `High`, `Low` are the intrinsics (ch.04 §4.11).
- ⚠️ *`SetLength` zero-fills newly-added elements.* Growing a dynamic array with
  `SetLength` initialises every **new** slot to zero/nil/empty — never garbage —
  while every **pre-existing** slot keeps its old value untouched. Confirmed
  both halves (dcc-verified, dcc32 37.0): `SetLength(A, 10)` on a never-touched
  `array of Integer` reads `A[9] = 0`; populating `A[0..9]` then
  `SetLength(A, 20)` leaves `A[0]..A[9]` at their old values while `A[10]..A[19]`
  read `0`. A semantic/codegen layer can rely on this — it is a real guarantee,
  not an implementation accident — but note it does **not** carry over to
  `string` (ch.07 §7.3.1): growing a string's length is NOT zero-filled.
- ⚠️ *The element type may be another dynamic array, and §8.1.2's indexing sugar
  applies there too.* `array of array of T` is the idiomatic jagged/2-D dynamic
  array — `SetLength(A, 2, 3)` takes one length per dimension — and both
  `A[0][1]` and the comma form `A[1, 2]` compile (dcc-verified). §8.1.2 states
  the `M[i, j]` ≡ `M[i][j]` normalisation for the STATIC form only, which reads
  as though the dynamic case were different; it is not.
  - *Implementation note:* written inline, the two nest as one array-type node
    inside another, so an element walk that peels a single level answers "array
    of T" where the index count says the caller wanted T. Peel to the innermost
    while the element is itself an inline array.
- *AST:* `DynamicArrayType { elementType }`.

### 8.2.2 Dynamic array concatenation & literal init

| | |
|---|---|
| **Introduced** | Delphi XE7 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`+` concatenates dynamic arrays; array values may be initialised with a
constructor literal `[…]`.

**Example**

```pascal
A := [1, 2, 3];        // dynamic array literal (XE7+)
B := A + [4, 5];       // concatenation (XE7+)
```

**Semantics & parsing notes**

- ⚠️ The `[1, 2, 3]` literal is an **array constructor** in a dynamic-array
  context, sharing syntax with set constructors (§B.9). The target **type**
  disambiguates set vs. array — the type-checker decides; the parser keeps a
  generic "bracket constructor" node.
- *AST:* `ArrayLiteral { elements[] }` (or shared `BracketConstructor`).
- `Insert(Item, DynArray, Index)` and `Delete(DynArray, Index, Count)` are
  intrinsics that mutate a **dynamic array** in place — inserting a value (or,
  for `Insert`, an open-array/slice of values) at `Index`, shifting later
  elements up, growing the array by one; and removing `Count` elements starting
  at `Index`, shifting later elements down, shrinking the array — analogous to
  their `string`/`TList` namesakes but operating directly on the array variable
  (taken by `var`, like `SetLength`). Confirmed on a `TArray<Integer>`-style
  dynamic array (dcc-verified, dcc32 37.0): `Insert(99, A, 2)` on `[1,2,3,4,5]`
  yields `[1,2,99,3,4,5]`; `Delete(A, 0, 2)` on that result yields `[99,3,4,5]`,
  `Length(A) = 4`.

---

### 8.2.3 The dynamic-array pseudo-constructor `T.Create(...)`

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A **dynamic array type** may be written with a `Create` pseudo-constructor that
builds a value from its arguments. It is spelled like a class constructor, but
no `Create` is declared anywhere — the compiler makes it up for the type.

**Example**

```pascal
type
  TMyArr = array of Byte;
var
  A: TMyArr;
begin
  A := TMyArr.Create($20, $20);       // two elements
  A := TMyArr.Create();               // legal, and empty
  A := TArray<Integer>.Create(1, 2);  // an instantiated generic, likewise
```

The RTL leans on it: `TBytes.Create($EF, $BB, $BF)` (System.SysUtils),
`TCharArray.Create('*', '?')` (System.IOUtils), `TArray<TGUID>.Create(...)`
(System.Win.Sensors).

**Semantics & parsing notes**

- The qualifier must be the **TYPE**. `A.Create(1, 2)` through a *variable* of
  that type is `E2671 Record, object, class type, or type helper required`
  (dcc32 37.0-probed) — the same error a **static** array gets:
  `TStat = array[0..1] of Byte; TStat.Create(1, 2)` is rejected.
- There is no member to resolve, so a name resolver must special-case the form
  rather than look `Create` up: the result type is the array type itself,
  exactly as a real constructor yields its class.
- Aliases hide it — `TBytes` is `TArray<Byte>` is `array of T` — so "is this a
  dynamic array" has to follow the alias chain, not read one node.

---

## 8.3 Open array parameters (recap)

### 8.3.1 Open arrays & `array of const`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`array of T` / `array of const` as a **parameter type** — see
[ch.06 §6.2.6](06-routines.md#626-open-array--array-of-const-parameters).

**Semantics & parsing notes**

- ⚠️ Repeat of the key trap: an `array of T` *parameter* is an **open array**, not
  a dynamic-array-typed parameter — different ABI. The open-array constructor
  `[a, b, c]` at the call site is an open-array value, not a set/dynamic array.
- `Slice(A, Count)` builds an open-array value covering the **first `Count`
  elements** of a static (or dynamic) array `A`, for passing a partial array
  where an open-array parameter is expected — without copying and without
  needing a `[…]` constructor at the call site. Confirmed (dcc-verified, dcc32
  37.0): given `S: array[0..9] of Integer` and a routine
  `procedure P(const Arr: array of Integer)`, `P(Slice(S, 3))` compiles and `Arr`
  inside `P` has `Length(Arr) = 3` covering `S[0]..S[2]`. `Slice` is itself an
  intrinsic, not an ordinary function — a resolver must recognise it rather than
  look it up as a declared symbol.
