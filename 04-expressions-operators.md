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
- *AST:* `BinaryOp { op, left, right }`, `UnaryOp { op, operand }`.
- ⚠️ *Precedence governs GROUPING, not evaluation ORDER* — and for the actual
  arguments of a routine call, evaluation order is unspecified by the language.
  See ch.06 §6.2 for the dcc32-observed (right-to-left, implementation-defined,
  not to be relied on) order for a multi-argument call.

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
- ⚠️ *`@` takes an INSTANCE method's address through the CLASS name, no
  instance involved* (dcc32 37.0-probed): `P := @TBaseCF.IP;` compiles for a
  plain (non-class) method `procedure IP;` and yields the code pointer
  (assignable to `Pointer`). Outside an address-of, the same designator is
  `E2124 Instance member inaccessible here` — the `@` context is what
  legalizes it. The vtable-building idiom rests on it (spring4d's
  `@TComparerInstance.AddRef` tables of `@TClass.Method` entries), so a
  member-resolution engine must reach instance members through a TYPE
  qualifier when the expression sits under `@`.
- `{$POINTERMATH ON}` enables `+`/`-`/`[]` on typed pointers (treating them
  array-like) — a directive-gated semantic; `PByte`/`PAnsiChar` enable it locally.
- ⚠️ *Do not confuse that with the OLD, unconditional rule:* for a pointer to an
  **array**, the `^` may simply be omitted before `[`, and equally before `.` for
  a pointer to a record. `P[I]` means `P^[I]` and `P.Field` means `P^.Field`,
  with no directive involved. dcc-verified with `PLine = ^TLine;
  TLine = array[Word] of TQuad` and POINTERMATH off, both as an expression and as
  a `with` target. The classic RTL/VCL scan-line idiom rests on it
  (`Vcl.Imaging.pngimage`'s `pPixelLine`, `Vcl.Imaging.GIFImg`).
  So a resolver typing an indexed or member expression must dereference a
  pointer base *before* looking for the array element or the field — and the two
  compose with alias chasing (`P[I].Field`, `P^.rgrc[0]`).

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
- ⚠️ *The right operand of `is` may be any CLASS-REFERENCE EXPRESSION, not
  only a type name* (dcc32 37.0-probed): with `CV: TPlainClass` (a `class of`
  variable) and `function GC: TPlainClass`, both `Obj is CV` and `Obj is GC`
  compile — the test is then against the class the expression yields at run
  time. The FMX sources ship it: `if FModel is DefineModelClass then`
  (`FMX.Presentation.Style`), where `DefineModelClass` is a virtual CLASS
  FUNCTION returning a metaclass. Official documentation states a type name;
  the grammar a parser must accept is an expression, and a completion/
  resolution engine must not filter that position to types only.

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
- *AST:* either a `BinaryOp { op: isNot }` or `UnaryOp(not, IsExpr)` — pick one
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

> The **complete intrinsic catalog** (≈90 routines, grouped by parser-behavior
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
  it in any other expression position: `E2193 Slice standard function only
  allowed as open array argument`. The RTL uses it in
  `System.Classes`/`System.ObjAuto`.
- ⚠️ *An argument of an INTRINSIC is never such a position*, even when that
  intrinsic's parameter is itself an open array — so the rule is narrower than
  "an open-array argument". `Insert(const Values: array of T; var Dest; Index)`
  is the decisive case and dcc32 37.0 rejects `Insert(Slice(A, 3), D, 0)`;
  `Concat(Slice(A, 3), D)` and `Writeln(Slice(A, 3))` go the same way, as does
  `Slice(Slice(A, 5), 3)`. Read it as: the position must be an argument of an
  ordinary *call*.
- ⚠️ *`A` may not be a dynamic array* — `Slice(D, 3)` with `D: array of Integer`
  is `E2016 Array type required`, even in a perfectly good argument position.
  A static array and an open-array parameter both work.

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
- *AST:* `FormattedArg { value, width?, precision? }` within the intrinsic call node.

---

### 4.11.3 OLE-automation named arguments

| | |
|---|---|
| **Introduced** | Delphi 2 (Variant/dispatch calls) |
| **Deprecated** | — |
| **Status** | ⚠️ Legacy (COM automation) |

In a call on a **`Variant`/`OleVariant`** an argument may be written
`Name := Expression`. The name is a *dispatch parameter name* passed to the
automation server; it is not an identifier in the program.

This is **not** general named-parameter syntax — Object Pascal has none. It
belongs to Automation alone, and the compiler says so itself: the ordering
diagnostic is `E2166 Unnamed arguments must precede named arguments in OLE
Automation call`.

**Grammar**

```ebnf
NamedArg = Identifier ":=" Expression ;
(* valid only in an argument/index list of a Variant-typed call *)
```

**Example**

```pascal
var Excel: Variant;
...
Excel.ActiveWorkbook.Charts[1].SeriesCollection.Add(
  Source := Excel.Worksheets[1].Range['B1:B20']);
```

**Semantics & parsing notes**

- ⚠️ *This is a second context-restricted argument form, and it collides with the
  assignment statement.* A parser that ends the argument at the identifier reads
  the `:=` as an assignment whose TARGET is the whole call, and reports `")"
  expected` — then the name, having been parsed as an ordinary reference, is a
  false `E2003`. Handle it in the argument rule, exactly like `:width` above.
- ⚠️ *The two sides are checked differently, and only the asymmetry makes it
  implementable.* dcc-verified on 37.0:
  - the NAME is not resolved at all — `V.Add(Nonexistent := 1)` compiles;
  - the VALUE is an ordinary expression, fully checked — `V.Add(Source :=
    Undeclared1)` is `E2003` on `Undeclared1`.
- ⚠️ *The gate is the STATIC TYPE of the callee being `Variant`/`OleVariant` —
  not "dispatch", and not the parameter name matching.* All dcc-verified, and
  the dispinterface result is the surprising one:

  | callee | `Foo(A := 1)` |
  |---|---|
  | `Variant` / `OleVariant` member | ✅ compiles |
  | global routine whose parameter really IS named `A` | ❌ `E2003` on `A` |
  | method of a class | ❌ `E2003` on `A` |
  | ordinary (early-bound) interface | ❌ `E2003` on `A` |
  | statically typed `dispinterface` | ❌ `E2003` on `A` |

  A `dispinterface` call is late-BOUND at run time but its signature is known at
  compile time, so the named form is rejected there just like a class method's.
  A resolver that exempts every `Name := Value` argument therefore loses real
  diagnostics; the exemption belongs to a `Variant`-typed callee (in practice: to
  calls the resolver could not bind statically).
- *The name is consumed even when it collides with a real variable.* With
  `Source: string` in scope, `V.Add(Source := 1)` compiles — as an assignment it
  would be `E2010 Incompatible types`. So the name is never an ordinary
  reference, and never an assignment target.
- *Named arguments must FOLLOW the positional ones.* `V.Add(1, Source := 2)` is
  fine; `V.Add(Source := 2, 1)` is `E2166`.
- *Also valid in an INDEX list*, for a Variant's indexed property:
  `V.Range[Source := 1] := 5` compiles. An implementation that handles only the
  parenthesised argument list still mis-parses this one.
- Only a bare identifier may appear on the left — nothing dotted or indexed.
- *Calling the Variant ITSELF is not this form:* `V(Source := 1)` is a syntax
  error (`E2066`). It is a member call or nothing.
- *AST:* `NamedArg { name, value }` within the call or index node.

### 4.11.4 Result types of the value-returning intrinsics

| | |
|---|---|
| **Introduced** | Pascal/D1 core; per-name tags as in B.4.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

The documentation gives most intrinsics a loose signature ("returns an
integer"); the compiler's answer is exact and, in a few places, surprising.
Everything below was probed on dcc32 and dcc64 37.0 (2026-09-07) by assigning
each call to a record variable and reading the type dcc names in
`E2010 Incompatible types: 'TProbe' and '<type>'`; the `var X := Intrinsic(...)`
inference (3.1.3) agrees with every row. Where the two compilers differ, both
answers are given.

**Fixed result, whatever the argument**

| Intrinsic | Result | Notes |
|---|---|---|
| `Ord(X)` | `Integer` | even of an `Int64`/`UInt64`/`NativeInt` argument |
| `Chr(X)` | `Char` | |
| `SizeOf(T\|X)`, `Hi(X)`, `Lo(X)` | `Integer` | `Hi`/`Lo` of any width, literals included |
| `Trunc(X)`, `Round(X)` | `Int64` | of a `Single` too; `Trunc(I64)` of an integer is `E2008`, `Round(I)` is accepted |
| `MulDivInt64` | `Int64` | |
| `Assigned`, `Odd`, `IsManagedType`, `IsConstValue`, `HasWeakRef`, `Eof`, `Eoln`, `SeekEof`, `SeekEoln` | `Boolean` | |
| `Pi` | `Extended` | dcc64 prints it as `Double`, which is what `Extended` IS on a 64-bit target; `var X := Pi` reads `Extended` on both |
| `Addr(X)`, `Ptr(N)`, `ReturnAddress`, `AddressOfReturnAddress`, `TypeInfo(T)`, `TypeHandle(T)` | `Pointer` | the untyped pointer, not `^T` or `PTypeInfo` |
| `GetTypeKind(T\|X)` | `System.TTypeKind` | |
| `Length(X)` | `Integer`; `NativeInt` for a **dynamic array** | so `Length(DynArr)` is `Int64` on dcc64 and `Integer` on dcc32; a string, static array, `ShortString` or `Variant` is `Integer` on both |

**The ordinal widening rule** (shared by `Pred`, `Succ`, `Low`, `High`, `Abs`
of an integer):

- `Byte`, `ShortInt`, `Word`, `SmallInt`, `Cardinal`, a subrange, a literal:
  **`Integer`** - `Pred(B)` of a `Byte` is an `Integer`, `High(Cardinal)` is
  an `Integer`.
- `Int64` **and `UInt64`**: **`Int64`** - `Succ(U64)` is an `Int64`,
  `High(UInt64)` is an `Int64`.
- `NativeInt`, `NativeUInt`: `NativeInt` (i.e. `Integer` on dcc32, `Int64` on
  dcc64) - the only rows the two compilers disagree on.
- An enum, `Char`, `AnsiChar`, `WideChar`, `Boolean`: **the argument's own
  type** - `Pred(E)` is the enum, `Succ(AC)` an `AnsiChar`.
- A subrange OVER an `Int64` range (`1..5000000000`): `Int64`.

**`Low`/`High` of a non-ordinal**

| Argument | `Low` | `High` |
|---|---|---|
| static array | index type, widened as above: `array[2..5]` and `array[Word]` give `Integer`, `array[TEnum]` gives `TEnum`, `array['a'..'z']` gives `Char`, `array[Boolean]` gives `Boolean` | same |
| dynamic array (value or type) | `Integer` | `NativeInt` (`Int64` on dcc64) |
| `string`, `AnsiString`, `WideString`, `ShortString` values | `Integer` | `Integer` |
| the long-string TYPE itself (`High(string)`) | `Integer` | `E2198 High cannot be applied to a long string` |
| a set type or value | `E2008` | `E2008` |

**Argument-dependent**

| Intrinsic | Rule (dcc32 / dcc64 where they differ) |
|---|---|
| `Abs(X)` | integer: the widening rule (`Abs(Byte)` = `Integer`, `Abs(UInt64)` = `Int64`, `Abs(NativeUInt)` = `NativeInt`). Real: **`Extended` on dcc32 for every real type**; on dcc64 `Currency` stays `Currency` and `Comp` stays `Comp` (their arithmetic is integral there), every other real is `Extended` (printed `Double`). `Abs(2.5)` is a real. `Abs(Variant)` is a real |
| `Sqr(X)` | integer: `Integer` for anything up to 32-bit SIGNED, but a 32-bit **unsigned stays `Cardinal`**, and the 64-bit ones keep their signedness (`Int64`, `UInt64`, `NativeInt`, `NativeUInt`). Real: `Extended` for every real, **`Currency` and `Comp` included** (unlike `Abs`) |
| `Swap(X)` | a 16- or 32-bit integer keeps its type (`Word`, `SmallInt`, `Integer`, `Cardinal`); `Byte`, `Int64` and a literal give `Integer` |
| `Copy(S, I, N)` / `Copy(A, I, N)` | the FIRST argument's type: `AnsiString` stays `AnsiString`, `ShortString` stays `ShortString`, a string literal is `string`, a dynamic array (incl. `TArray<T>`) stays that array type; `Copy(A)` with no range works for a dynamic array and is `E2035` for a string; a static array is `E2008` |
| `Concat(A, B, ...)` | all arguments of ONE type: that type (`AnsiString`, `WideString`, `UTF8String`, `RawByteString`, a dynamic array type) - with two exceptions: **`ShortString`+`ShortString` is `AnsiString`** and **`AnsiChar`+`AnsiChar` is `ShortString`**. Mixed string kinds, `Char` operands and literals: `string` |
| `Default(T)` | `T` for a record, an ordinal, a real, a string, a STATIC array. **`Pointer` for every reference type** - class, interface, dynamic array, procedural/method/reference type, class reference, pointer: `var X := Default(TObj); X.Free` is `E2018 Record, object or class type required` and `var A := Default(TDynI); A[0]` is `E2016` |
| `AtomicIncrement`/`AtomicDecrement`/`AtomicExchange`/`AtomicCmpExchange` | the first argument's type (`Integer`, `Cardinal`, `Int64`, `UInt64`, `NativeInt`, `Pointer`); an object reference is `E2008` |

**Semantics & parsing notes**

- ⚠️ *A typer that gives `Ord` "the argument's type" or `Trunc` "Integer" will
  produce false `E2010`s at the next assignment* - `I64 := Trunc(D)` is fine,
  `var N := Ord(I64)` is an `Integer`. The rows above are the contract.
- ⚠️ *`Length(DynArr)` is the one platform-dependent `Length`.* It shows in
  `var L := Length(A)` on dcc64: `L` is an `Int64`, and comparing it to an
  `Integer` counter is a widening, not an error.
- The dcc64 `Currency`/`Comp` exceptions under `Abs` match the 64-bit literal
  rule in 3.1.3: on that target `Currency` arithmetic does not promote to a
  real.

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
