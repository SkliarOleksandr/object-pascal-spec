# 04 — Expressions & Operators

The grammar of expressions and the precedence table are fixed in
[Appendix B](B-lexical-grammar.md) §B.7/§B.9; this chapter gives the **semantics**
of each operator group and the compiler-magic quasi-operators a parser must
recognise specially.

## 4.1 Operator precedence (recap)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995); `is not`/`not in` 13.0 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Four precedence levels, all binary operators left-associative. Full table in
[§B.7](B-lexical-grammar.md#b7-operators--punctuation-tokens).

**Semantics & parsing notes**

- ⚠️ Restating the critical surprise: `not` (unary, level 1) and `and`/`or`
  (levels 2/3) bind **tighter than the relational operators** (level 4), so
  `a = b and c` parses as `a = (b and c)`. The AST follows the table; never
  special-case it.
- *AST:* `BinaryExpr { op, left, right }`, `UnaryExpr { op, operand }`.

---

## 4.2 Arithmetic operators

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`+ - *` (numeric), `/` (real division, **always** yields a float), `div`
(integer division), `mod` (integer remainder), unary `+`/`-`.

**Example**

```pascal
Q := A div B;   R := A mod B;   F := A / B;   // F is floating-point
```

**Semantics & parsing notes**

- ⚠️ `/` on two integers **still produces a real** result type — do not infer
  Integer. `div`/`mod` require integer operands.
- `+` is overloaded across numeric, string, set, and dynamic-array operands
  (resolved by operand type). For records, `+` may be user-overloaded (ch.09).
- Operand promotion follows the numeric ladder (ch.02); mixed int/real ⇒ real.

---

## 4.3 Logical (Boolean) operators & short-circuit

| | |
|---|---|
| **Introduced** | Pascal; short-circuit controllable via `{$BOOLEVAL}` |
| **Deprecated** | — |
| **Status** | ✅ Current |

`and`, `or`, `xor`, `not` applied to **Boolean** operands.

**Semantics & parsing notes**

- ⚠️ *Same tokens as bitwise* (4.4) — the operand **type** disambiguates: Boolean
  operands ⇒ logical; integer operands ⇒ bitwise. The parser produces the same
  `BinaryExpr`; the type-checker picks the meaning.
- *Short-circuit:* by default (`{$BOOLEVAL OFF}`, the modern default) `and`/`or`
  short-circuit (right operand not evaluated when the result is determined).
  `{$BOOLEVAL ON}` forces full evaluation — a directive-dependent **semantic**,
  important when operands have side effects. Record the directive state.

---

## 4.4 Bitwise operators

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`and`, `or`, `xor`, `not` (bitwise) and `shl`, `shr` (bit shifts) on integers.

**Example**

```pascal
Flags := Flags or $04;
Hi := Value shr 8;
```

**Semantics & parsing notes**

- Integer operands ⇒ bitwise semantics (see 4.3 disambiguation note).
- `shl`/`shr` are at the multiplicative level (§B.7).

---

## 4.5 Relational & equality operators

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`=`, `<>`, `<`, `>`, `<=`, `>=` — comparison yielding `Boolean`. For sets,
`<=`/`>=` mean subset/superset.

**Semantics & parsing notes**

- Lowest precedence (level 4) — hence the parenthesisation requirement around
  Boolean sub-expressions.
- Overloadable on records (`Equal`, `LessThan`, … operators, ch.09).
- Operand types must be compatible; comparing incompatible types is a semantic
  error (except via implicit numeric promotion).

---

## 4.6 Set operators

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`+` union, `-` difference, `*` intersection, `in` membership, and
`=`/`<>`/`<=`/`>=` set relations.

**Example**

```pascal
if C in ['a'..'z'] then ...;
Both := S1 * S2;
```

**Semantics & parsing notes**

- `in` is a relational-level operator; its RHS is a set value or set constructor
  `[ … ]` (§B.9 `SetConstructor`).
- `not in` (13.0) is the negated membership test, parsed as the `not`+`in` pair at
  relational level.

---

## 4.7 String operators

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`+` concatenation; relational operators for lexicographic comparison.

**Semantics & parsing notes**

- `+` on string operands concatenates; mixed `Char`/`string` is allowed.
- Comparison is ordinal by code unit (case-sensitive); locale-aware comparison is
  an RTL concern, not a language operator.

---

## 4.8 Pointer & address operators

| | |
|---|---|
| **Introduced** | `@`/`^` Pascal; pointer math via `{$POINTERMATH}` Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`@` (address-of), `^` (dereference, postfix), and — under `{$POINTERMATH ON}` —
`+`/`-`/indexing on typed pointers.

**Example**

```pascal
P := @X;        // address-of
V := P^;        // dereference
Inc(P);         // pointer math when enabled
```

**Semantics & parsing notes**

- ⚠️ `^` is **both** a prefix type-constructor (`^T` = pointer-to-T, ch.10) and a
  **postfix** dereference (`p^`). Position decides — track which in the AST.
- `@` result type depends on `{$TYPEDADDRESS}` (typed vs. untyped pointer).
- ⚠️ *`@@` double address-of:* for a **procedural variable**, `@P` yields the
  routine address stored in it, while `@@P` yields the address **of the variable
  itself**. Grammatically it is just `@` applied twice (`"@" Factor` recursion) —
  no new token; the VCL uses it (`LPARAM(@@Hook)` in `Vcl.ActnMenus`).
- `{$POINTERMATH ON}` enables `+`/`-`/`[]` on typed pointers (treating them
  array-like) — a directive-gated semantic; `PByte`/`PAnsiChar` enable it locally.

---

## 4.9 Type-test & type-cast operators (`is`, `as`)

| | |
|---|---|
| **Introduced** | Delphi 1 (with classes) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`is` tests runtime class/interface type (Boolean); `as` performs a checked
downcast (raises on failure). Full treatment in [ch.12](12-inheritance-polymorphism.md).

**Example**

```pascal
if Obj is TButton then
  (Obj as TButton).Caption := 'OK';
```

**Semantics & parsing notes**

- `is` is relational-level; `as` is multiplicative-level (§B.7).
- Operands must be class or interface references; checked at compile and run time.

### 4.9.1 `is not` operator

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Negated type test — `X is not T` ≡ `not (X is T)`, but reads better and avoids the
precedence parentheses.

**Example**

```pascal
if Obj is not TButton then Exit;
```

**Semantics & parsing notes**

- Parsed as the `is`+`not` token pair at relational level (no new token, §B.4.1).
- *AST:* either a `BinaryExpr { op: isNot }` or `UnaryExpr(not, IsExpr)` — pick one
  representation and normalise.

---

## 4.10 Type casts

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`TypeName(Expr)` — value cast (numeric/ordinal conversion or reference reinterpret).

**Grammar**

```ebnf
TypeCast = TypeRef "(" Expression ")" ;     (* see §B.9 *)
```

**Example**

```pascal
B := Byte(I);
Ch := Char(65);
Btn := TButton(Sender);     // unchecked reference cast
```

**Semantics & parsing notes**

- ⚠️ *Cast vs. call ambiguity:* `Foo(x)` is a **type cast** iff `Foo` names a type,
  otherwise a **call**. The parser cannot tell syntactically — defer to symbol
  resolution.
- *Variable (hard) casts* reinterpret storage and can appear as an lvalue
  (`Byte(I) := 1`); *value casts* produce an rvalue. The kind depends on operand.
- *AST:* `TypeCastExpr { targetType, expr }`.

---

## 4.11 Compiler-intrinsic quasi-operators

| | |
|---|---|
| **Introduced** | Pascal/D1; `Default(T)` 2009; `TypeInfo`/`GetTypeKind` RTTI-era |
| **Deprecated** | — |
| **Status** | ✅ Current |

Built-in routines treated specially by the compiler (some accept **types** as
arguments, which ordinary functions cannot). The parser must recognise these
because their argument grammar differs from normal calls.

> The **complete intrinsic catalog** (≈85 routines, grouped by parser-behavior
> class, with version tags) lives in
> [Appendix B §B.4.3](B-lexical-grammar.md#the-complete-intrinsic-catalog).
> This section covers only the grammar-relevant forms.

**Examples & forms**

```pascal
Inc(X);  Dec(X, 2);            // mutate ordinal/pointer in place
Length(S);  SetLength(A, 10);  // dynamic length
SizeOf(TRec);                  // takes a TYPE
Default(TMyRecord);            // zero value of a TYPE (2009+)
Ord(c);  Chr(n);  Pred(x);  Succ(x);
Low(A);  High(A);              // bounds; accept type or value
Assigned(P);  Addr(X);
TypeInfo(TMyEnum);             // takes a TYPE -> PTypeInfo
```

**Semantics & parsing notes**

- ⚠️ *Type-or-value arguments:* `SizeOf`, `TypeInfo`, `Default`, `Low`/`High`,
  `GetTypeKind` accept a **type identifier** where the grammar otherwise expects an
  expression. The parser needs a special argument rule (accept `TypeRef` here) and
  must not mis-resolve the type name as a variable.
- ⚠️ *`var`-mutating intrinsics:* `Inc`/`Dec`/`SetLength`/`New`/`Dispose` take their
  first argument **by reference** — the argument must be an lvalue.
- These are identifiers in `System`, not keywords — but a real compiler hard-codes
  them. The AST may keep them as `CallExpr` and tag `intrinsic: true`.
- ⚠️ *`Slice(A, Count)`:* valid **only** as an actual argument to an open-array
  parameter (ch.06 §6.2.6) — it passes the first `Count` elements of `A`. Reject
  it in any other expression position (semantic check); the RTL uses it in
  `System.Classes`/`System.ObjAuto`.

### 4.11.1 `NameOf` intrinsic

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`NameOf(identifier)` returns the source **string name** of the identifier at
compile time.

**Example**

```pascal
ShowMessage(NameOf(Button1));   // 'Button1'
LogField(NameOf(TCustomer.Name));
```

**Semantics & parsing notes**

- Argument is an **identifier/qualified-identifier reference**, resolved but not
  evaluated; the result is a compile-time string constant.
- Intrinsic identifier, not a reserved word — needs the original-case spelling
  preserved by the lexer (§B.3).
- *AST:* `NameOfExpr { ref }` → folds to a string literal.

### 4.11.2 Formatted arguments of `Write`/`Writeln`/`Str`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Inside calls to the `Write`, `Writeln`, and `Str` intrinsics — and **only**
there — an argument may carry width/precision specifiers using `:`.

**Grammar**

```ebnf
WriteArg = Expression [ ":" Expression [ ":" Expression ] ] ;
(* valid only in the argument lists of Write / Writeln / Str *)
```

**Example**

```pascal
Writeln(X:10:2);        // width 10, 2 decimals
Write(F, Name:20);      // field width 20
Str(Val:0, S);          // RTL itself uses this (System.pas)
```

**Semantics & parsing notes**

- ⚠️ *Context-restricted grammar:* the `:width[:precision]` suffix is **not** part
  of the general expression grammar — it is legal only in these intrinsics'
  argument lists. A parser needs a special argument rule when the callee resolves
  to `Write`/`Writeln`/`Str` (or must parse `:` speculatively and validate later).
- ⚠️ Don't confuse with other `:` uses (declarations, case labels) — here it
  appears *inside* an actual-parameter list.
- `:precision` is only meaningful for floating-point values.
- Additionally, `Write`/`Writeln`/`Read`/`Readln` accept an optional leading file
  variable — their arity/typing is fully compiler-magical.
- *AST:* `WriteArg { value, width?, precision? }` within the intrinsic call node.

---

## 4.12 Operator overloading (cross-reference)

| | |
|---|---|
| **Introduced** | Delphi 2006 (records only) |
| **Deprecated** | — |
| **Status** | ✅ Current |

User-defined operators are declared on **records** (and managed records), not
classes. The operator *use* grammar is unchanged (the operators above); only
resolution changes. Declaration syntax (`class operator Add(...)`, `Implicit`,
`Equal`, etc.) is in [ch.09](09-records.md).

**Semantics & parsing notes**

- When an operator's operands are record types, resolution searches for a matching
  `class operator` (incl. `Implicit`/`Explicit` conversions). Otherwise built-in
  semantics apply.
- *Parser impact:* none at the operator site — same `BinaryExpr`/`UnaryExpr`; the
  type-checker dispatches to the overload.
