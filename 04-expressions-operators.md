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
  special-case it - but for the ONE exception dcc makes: a type name on the
  right of `is` ends that operand, `O is TFoo and C` is `(O is TFoo) and C`
  (§4.9, dcc64 37.0-probed 2026-09-26).
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
  Integer. `div`/`mod` require integer operands: `D div 2` and `I mod D` are
  `E2015 Operator not applicable to this operand type`.
- ⚠️ *`div` truncates toward zero and `mod` takes the DIVIDEND's sign*, at run
  time and when folded alike: `-7 div 2` is -3, `-7 mod 2` -1, `7 mod -2` 1
  (`A = (A div B) * B + A mod B`). A constant divisor of zero is `E2098
  Division by zero` (`const K = 1 div 0`). dcc64 37.0, 2026-10-30 (spec audit
  g2, local/probe-park/spec-audit/g2 A1.dpr, A4.dpr).
- `+` is overloaded across numeric, string, set, and dynamic-array operands
  (resolved by operand type). For records, `+` may be user-overloaded (ch.09).
- Operand promotion follows the numeric ladder (ch.02); mixed int/real ⇒ real.

### 4.2.1 Result type of an operator expression

| | |
|---|---|
| **Introduced** | Pascal (pre-1995); inference through `var X :=` 10.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

The type an operator expression has - what `var X := A op B` infers (3.1.3)
and what a member access on `(A op B)` is looked up on. Probed on dcc32 and
dcc64 37.0 (2026-09-25) through `GetTypeName(TypeInfo(T))` of the inferred
variable, over VARIABLES of each type; a constant expression folds and is
typed by its value instead (3.2.1).

| Expression | dcc32 | dcc64 |
|---|---|---|
| `B + B`, `B * B` (Byte), `W * W` (Word), `B shl 2`, `Sub + Sub` (a subrange) | anonymous subrange | same |
| `Sh + Sh` (ShortInt), `-B` | `Integer` | same |
| `not B` | `Byte` - `not` keeps its operand's type (`not I64` is `Int64`) | same |
| `I + C` (Integer, Cardinal) | `Int64` | same |
| `C + C` | `Cardinal` | same |
| `I + I64`, `U64 + I` | `Int64`, `UInt64` - the 64-bit operand's type | same |
| `NI + I` (NativeInt) | `Integer` | `Int64` |
| `I div B`, `I shl 2` | `Integer` | same |
| `MyI + MyI` (`TMyI = type Integer`) | `Integer` - the distinct type is lost | same |
| `I / I` | `Extended` | same |
| `Sg + Sg`, `Sg + I`, `Sg * D`, `E + D`, `DT + 1` (TDateTime), `-Sg` | `Extended` - real arithmetic is Extended whatever the operands | same |
| `D + Cu`, `Cu + I`, `Cu + Cu`, `Cu * Cu`, `-Cu` | `Extended` | **`Currency`** |
| `5000000000 + 1` | `Int64` - literals by value | same |
| `1.5 + 1.5` | `Extended` | `Currency` |
| `1.5 + 1` | `Extended` | `Extended` |
| `Ch + Ch`, `Ch + S`, `S + A` (string, AnsiString) | `string` | same |
| `A + A`, `AC + A` (AnsiChar) | `AnsiString` | same |
| `AC + AC` | anonymous - a short string (E2010 names it `ShortString`, 2026-10-12) | same |
| `A + 'x'`, `'x' + A`, `A + #0`, `SS + 'x'`, `SS + A`, `SS + SS`, `SS + AC` | `AnsiString` - a literal takes the other operand's string type, ShortString joins as AnsiString | same |
| `WS + 'x'`, `WS + A`, `A + WS`, `WS + SS`, `WS + U8` | `WideString` | same |
| `U8 + 'x'`, `U8 + RB` (`UTF8String`, `RawByteString`) | `UTF8String`; `RB + 'x'`, `RB + A` `RawByteString` - RawByteString takes the other Ansi kind on its left | same |
| `U8 + A`, `A + U8`, `SS + U8`, `U8 + AC`, `AC + 'x'`, `WS + S`, `SS + S`, `A + Ch` | `string` - two code pages, or a Unicode / Char side | same |
| `St + [e1]`, `St - [e1]`, `St * [e1]` | `TES` - the set operand's type; `[e1] + St` is an anonymous set | same |
| `P + 1` (PChar) | `PWideChar` - the pointer's type | same |
| `Arr + Arr`, `[S] + Arr`, `Arr + [S]` (`Arr: TArray<string>`), `Ints + [1]` | the array's type - `TArray<string>`, `TArray<Integer>` (dcc64, 2026-10-07) | same |
| `Names + [S]` (`TNames = array of string`) | `TNames` - a named array keeps its name | same |
| `[S] + [S]` | anonymous | same |
| `Names + Arr`, `Names + Other` (two array types of one element) | E2010 | same |
| `B > I`, `Bo and Bo` | `Boolean` | same |
| `BB and BB`, `Bo and LB`, `not WB`, `BB = Bo` (`ByteBool`, `LongBool`, `WordBool`) | `Boolean` (2.2.2; spec audit g2 C3.dpr, 2026-10-30) | same |
| `P - Q` (two `PChar`, two `PAnsiChar`, two `PByte`) | `Integer` - the distance in elements (`(P + 4) - P` is 4), a `NativeInt`; two `PInteger` under `{$POINTERMATH OFF}` are `E2015` (spec audit g2 C1.dpr, C7.dpr, A5.dpr) | `Int64` |
| `Ch + 1`, `Ch + I` (Char with an integer) | `W1058 Implicit string cast with potential data loss from 'string' to 'ShortString'` and `E2008 Incompatible types` - no Char arithmetic (`Succ`, `Chr(Ord(Ch) + 1)`; spec audit g2 A4.dpr, A5.dpr) | same |

**Semantics & parsing notes**

- ⚠️ *Arithmetic on narrow integers is NOT the wider operand's type.* A typer
  that answers "the wider of the two" gives `B + B` a Byte; dcc computes it
  in an anonymous subrange of Integer, and a resolver is closest to that
  with Integer. `not` is the exception that does keep the operand's type.
- ⚠️ *A 32-bit signed/unsigned mix is Int64,* not the left or the right
  operand's type.
- ⚠️ *There is no Single or Double arithmetic.* Every real operation yields
  Extended - which on dcc64 is the 8-byte type, so `Sg + Sg` there is a
  Double-sized Extended - except that on dcc64 an operation with a Currency
  operand stays Currency (Currency arithmetic is integral on that target,
  as `Abs` shows in 4.11.4).
- ⚠️ *A constant operand adapts to the other operand.* An integer literal,
  or a true constant naming one, takes the other operand's type when its
  value fits it: `C + 1`, `1 - C`, `C * 2`, `C div 2`, `C and $FF`,
  `C or 1`, `C + K` (`const K = 5`) over a Cardinal are Cardinal, `B and
  $FF` and `B and K` over a Byte Byte. One that does not fit keeps the
  type of its value (the ladder of 3.2.1) and mixes like a variable:
  `C + (-1)`, `C + KN` (`KN = -1`), `C + 5000000000` are Int64.
- ⚠️ *A folded constant expression is computed IN that type and does not
  widen* - a result outside it is `E2099 Overflow in conversion or
  arithmetic operation`, not a wider constant. `2147483647 + 1`, `MaxInt +
  1`, `High(Integer) + 1`, `2147483647 * 2`, `100000 * 100000` and
  `Integer(5) * 1000000000` (Integer by the rules above) fail, as do
  `4294967295 + 1`, `High(Cardinal) + 1` and `Cardinal(1) - 2` (Cardinal),
  `0 - 4294967295` (the 0 adapts to Cardinal), `65535 * 65537` (Word with
  Integer: Integer)
  and `High(Int64) + 1`. One 64-bit operand makes room: `Int64(2147483647) +
  1`, `2147483647 + Int64(1)` and `4294967295 + Int64(1)` fold to 2147483648
  and 4294967296, and `5000000000 + 1` is an Int64 already. Narrow operands
  compute in Integer as variables do (`Byte(200) + Byte(200)` is 400). A
  constant evaluator that folds in 64 bits and types the result by its value
  accepts all of these. dcc64 and dcc32 37.0, 2026-10-30 (spec audit g2,
  local/probe-park/spec-audit/g2 A7.dpr, A8.dpr).
- ⚠️ *Signedness, not width, decides a 32-bit mix.* A Cardinal with a signed
  operand of ANY width is Int64 (`C + Sh`, `C + SI`, `C + I`), with an
  unsigned one Cardinal (`C + B`, `C + W`, `C - C`); `-C` is Int64,
  `-W` and `-Sh` Integer, `-U64` UInt64. A 64-bit operand gives Int64,
  UInt64 whenever an unsigned operand is 64-bit (`I64 + U64`, `U64 +
  I64`).
- *A shift is its LEFT operand's type,* at least Integer, whatever the
  count: `C shl 2` and `C shl I` Cardinal, `I shl I64` Integer, `B shl
  I64` Integer, `I64 shl 2` Int64.
- *`and` / `or` / `xor` on integers cover rather than widen:* two narrow
  operands give the narrowest of ShortInt, Byte, SmallInt, Word, Integer
  holding both (`B and B` Byte, `B and W` Word, `Sh and B` SmallInt,
  `W or Sh` Integer, `B and -1` SmallInt, `Sh and 200` SmallInt); beside
  a 32-bit operand the result is Cardinal when an unsigned 32-bit operand is
  there and Integer otherwise (`C and I`, `C and Sh`, `C and -1`, `I and
  $FFFFFFFF` Cardinal; `B and I` Integer); beside a 64-bit one UInt64 or
  Int64 by the same rule (`I64 and C` Int64, `U64 and Sh` UInt64).
  `B and 300` and `B and $1FF` are an anonymous subrange (E2010 names it
  Integer).
- *A distinct integer type computes in its base* (2.5.1): `MI64 + MI64`
  and `MI64 + 1` (`TMyI64 = type Int64`) Int64, `MC + MC` (`type
  Cardinal`) Cardinal, `MC + I` Int64, `MW + MW` Integer, `MI and MI`,
  `MI shl 2`, `-MI` Integer - while `not MI` keeps `TMyI` (as `not`
  keeps every operand's type).
- *dcc64: Currency and Comp stay integral; a real CONSTANT is Currency only
  in additions.* With a Currency VALUE operand `+ - *` are Currency
  whatever the other operand (`Cu + D`, `D + Cu`, `Cu * Cu`, `Cu + 1`,
  `Cu * 1.5`, `Cu + Cm`); a Comp value with integers or Comp is Comp
  (`Cm + Cm`, `Cm * 2`, `Cm + I`, `-Cm`), with a real Extended (`Cm +
  D`, `Cm + 1.5`). Two Currency-shaped real constants (3.1.3) added or
  subtracted are Currency (`1.5 + 1.5`, `1.5 - 1.5`, `KCur + KH`), and so
  is a NAMED one with an integer (`KCur + 1`, `KCur - KI`, `KI + KCur`,
  `KCur + I`) - but a literal with an integer is Extended (`1.5 + 1`,
  `1.5 + I`, `I - 1.5`, `K + 1.5`), and so is any constant multiplied
  (`1.5 * 1.5`, `KCur * 2`) or combined with a real variable (`D + 1.5`,
  `D * Half`, `KCur + D`); `-1.5` and `-KCur` are Currency. `/`:
  Currency over an integer or a real variable, or an integer over Currency
  (`Cu / 2`, `Cu / I`, `Cu / D`, `Cu / Sg`, `I / Cu`, `2 / Cu`);
  Extended for `Cu / Cu`, `D / Cu`, `Cu / 1.5`, `Cm / 2`, `Cm / Cm`;
  Comp for an integer over Comp (`I / Cm`). dcc32 answers Extended for all
  of these.
- A weak alias operand is its target's row (2.5.1): `B + 1` with
  `B: TBig`, `TBig = Int64` is `Int64`, as are `Abs(B)`, `Succ(B)` and
  `Sqr(B)` (4.11.4); `Swap` of a `TW = Word` is `Word`. A distinct
  `type` loses its name, as the `MyI + MyI` row says. Overload resolution
  reads these types: `PB(B1 + B2)` takes `PB(X: Integer)` beside
  `PB(X: Byte)`, `R(Sg * Sg)` takes `R(X: Extended)` beside `R(X:
  Single)`, `PC(C + I)` takes `PC(X: Int64)` beside the Cardinal and
  Integer overloads (dcc64 37.0, 2026-10-12).
- An operand whose type is a record with overloaded operators takes the
  operator's declared result type (4.12).

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
  `BinaryExpr`; the type-checker picks the meaning. A Boolean with an integer
  is neither: `Bo and I` is `E2015 Operator not applicable to this operand
  type` (spec audit g2 A4.dpr). Any mix of `Boolean` and the `*Bool` types
  yields `Boolean` (2.2.2).
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
- ⚠️ *`shr` is a LOGICAL shift and the count is taken modulo the width*, for
  a constant as for a variable - a constant folder must reproduce both. The
  width is the left operand's computing type (4.2.1: at least Integer):
  `-8 shr 1` is 2147483644 (`const KN = -8 shr 1` too), not -4; `1 shl 33`
  is 2 (`const KS = 1 shl 33` too), an Int64 `1 shl 65` 2, and `B shl 1`
  of a Byte 255 is 510. dcc64 and dcc32 37.0, 2026-10-30 (spec audit g2,
  local/probe-park/spec-audit/g2 A1.dpr).

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
  error (except via implicit numeric promotion): `I = S` is `E2010
  Incompatible types: 'string' and 'Integer'`.
- *A comparison has its own warnings,* which a checker that reproduces dcc's
  output must emit: a signed with an unsigned 32-bit operand (`I < C`) is
  `W1023 Comparing signed and unsigned types - widened both operands` - and
  the widening is real, `-1 < Cardinal(1)` is TRUE; a constant outside the
  other operand's range gives `W1021 Comparison always evaluates to False`
  (`B < 0`, `B > 255` of a Byte, `C < 0` of a Cardinal) or `W1022 Comparison
  always evaluates to True` (`B >= 0`). dcc64 and dcc32 37.0, 2026-10-30
  (spec audit g2, local/probe-park/spec-audit/g2 A1.dpr, A4.dpr, A5.dpr).

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
- *The RHS of `in` is a full simple expression* - no type-name exception as
  for `is` (§4.9): `X in S + [3]` is `X in (S + [3])`, and `X in S and C` /
  `X in [1, 2] and C` are `X in (S and C)` - `E2015` (dcc64 37.0, probed
  2026-09-26). Write `(X in S) and C`.
- `not in` (13.0) is the negated membership test, parsed as the `not`+`in` pair at
  relational level.
- *The set relations are `=`, `<>`, `<=`, `>=` only:* `S1 < S2` and `S1 > S2`
  are `E2008 Incompatible types` (not E2015).
- *`in` with a left operand outside the set's range is simply False* - no
  diagnostic, no wrap-around: `300 in [1, 2]`, `-1 in S` and, with `I :=
  300`, `I in S` of a `set of Byte` holding 0..255 are all FALSE (300 is not
  read as 44). The constructor's own elements are range-checked instead:
  `I in [1, 300]` is `E1012 Constant expression violates subrange bounds`.
  dcc64 and dcc32 37.0, 2026-10-30 (spec audit g2,
  local/probe-park/spec-audit/g2 A4.dpr, A5.dpr, A10.dpr).

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
Inc(P);         // steps by SizeOf(P^) on any typed pointer, no directive needed (§10.1.4)
```

**Semantics & parsing notes**

- ⚠️ `^` is **both** a prefix type-constructor (`^T` = pointer-to-T, ch.10) and a
  **postfix** dereference (`p^`). Position decides — track which in the AST.
- `@` result type depends on `{$TYPEDADDRESS}` (typed vs. untyped pointer).
  Under the default `{$T-}` `@I` is the untyped `Pointer`: `PB := @I` with
  `PB: PByte` compiles, and `var Z := @I; Writeln(Z^)` is `E2054 Illegal type
  in Write/Writeln statement`. Under `{$T+}` it is a `^Integer`: `PB := @I` is
  `E2010 Incompatible types: 'PByte' and 'Pointer'` (dcc names the anonymous
  pointer type `Pointer`), while `var X := @I` then takes `PI := X` and `X^ +
  1`. The directive is positional, read where the `@` stands. dcc64 37.0,
  2026-10-30 (spec audit g2, local/probe-park/spec-audit/g2 C5.dpr, C6.dpr).
- ⚠️ *`@@` double address-of:* for a **procedural variable**, `@P` yields the
  routine address stored in it, while `@@P` yields the address **of the variable
  itself**. Grammatically it is just `@` applied twice (`"@" Factor` recursion) —
  no new token; the VCL uses it (`LPARAM(@@Hook)` in `Vcl.ActnMenus`).
  `@(@P)` is the same address as `@@P` (dcc64 37.0, probed 2026-09-26).
- ⚠️ *`@P` of a procedural variable is a DESIGNATOR, and parentheses make it a
  value* (dcc64 37.0, probed 2026-09-26): `@P := GetProcAddress(...)` assigns
  the variable (the RTL's delay-load idiom) and `(@P) := ...` is `E2064 Left
  side cannot be assigned to`; `F(@P)` passes the variable to a `var Pointer`
  parameter and `F((@P))` is `E2197 Constant object cannot be passed as var
  parameter`. The operand of `@` does not take parentheses either where it
  names a routine: `@(Proc)` and `@(F)` are `E2036 Variable required`,
  `@(TFoo.M)` `E2076`; of a variable, field, element or dereference they
  change nothing (`@(R.F)`, `@(A[1])`, `@(P^)` give the addresses of `@R.F`,
  `@A[1]`, `@P^`).
- *Extent:* `@` reaches over the whole selector chain after it - `@R.F` is
  `@(R.F)`, `@A[1]` is `@(A[1])`, `@P^` is `@(P^)` (the same probe).
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
  - ⚠️ *`P[I]` there is `(P + I)^`: its type is the POINTEE, whatever that is* -
    a scalar (`PChByte(A)[1]` is a Byte), and a RECORD as well: spring4d's
    `{$POINTERMATH ON} PSlice = ^TSlice` makes `p[n].len` a field of TSlice
    (dcc64 37.0, 2026-10-08). A resolver must not stop at "not an array".
  - ⚠️ *The character pointers have no declaration to chase.* `PChar`,
    `PWideChar` and `PAnsiChar` are compiler-provided (System.pas declares
    none of them), yet `P^` and `P[I]` are a `Char` / `WideChar` /
    `AnsiChar`, and a helper for that type applies: System.SysUtils'
    `FormatPtr^.IsNumber`, System.Classes' `Result^.IsLowSurrogate`,
    System.StrUtils' `Result[I].IsLetterOrDigit` read TCharHelper (dcc64
    37.0, 2026-10-08: `P^.IsDig`, `W^.IsDig`, `A^.Code` and `P[1].IsDig`
    against `record helper for Char` / `for AnsiChar` compile). A resolver
    seeding these names must seed their pointee with them.
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

`is` tests an object's runtime class (Boolean); `as` performs a checked
downcast (raises on failure) or an interface query. Full treatment in
[ch.12](12-inheritance-polymorphism.md) §12.4.1 (classes) and
[ch.14](14-interfaces.md) §14.5.1 (interfaces).

**Example**

```pascal
if Obj is TButton then
  (Obj as TButton).Caption := 'OK';
```

**Semantics & parsing notes**

- `is` is relational-level; `as` is multiplicative-level (§B.7).
- Operands must be class or interface references; checked at compile and run time.
  The compile-time part is concrete: `is`/`as` between unrelated classes is
  `E2010`, and an interface on the RIGHT of `is` is always `E2015 Operator not
  applicable to this operand type` - even when the object's class lists that
  interface (`C is IG` with `TC = class(TInterfacedObject, IG)`); use `as` or
  `Supports`. The per-case rules are owned by §12.4.1 and §14.5.1. dcc64 37.0,
  2026-10-30 (spec audit v, local/probe-park/spec-audit/v V2.dpr; g4 IIsAs.dpr,
  g5 I05.dpr).
- ⚠️ *The right operand of `is` may be any CLASS-REFERENCE EXPRESSION, not
  only a type name* (dcc32 37.0-probed): with `CV: TPlainClass` (a `class of`
  variable) and `function GC: TPlainClass`, both `Obj is CV` and `Obj is GC`
  compile — the test is then against the class the expression yields at run
  time. The FMX sources ship it: `if FModel is DefineModelClass then`
  (`FMX.Presentation.Style`), where `DefineModelClass` is a virtual CLASS
  FUNCTION returning a metaclass. Official documentation states a type name;
  the grammar a parser must accept is an expression, and a completion/
  resolution engine must not filter that position to types only.
- ⚠️ *A TYPE NAME on the right of `is` is taken alone, and the result goes on*
  (dcc64 37.0, probed 2026-09-26). The precedence table says `O is TFoo and C`
  is `O is (TFoo and C)`; dcc compiles it as **`(O is TFoo) and C`** - the
  RTL and VCL are full of `if Obj is TFoo and (...) then` (19 sites in 13
  Studio units). The type name - a class, an alias of one, a metaclass type,
  qualified (`System.TObject`, `TFoo.TInner`), generic (`TBox<Integer>`,
  `TBox<Integer>.TInner`), `&`-escaped - ends the right operand, and the `is`
  result then continues as the LEFT operand of the multiplicative and
  additive operators that follow: `O is TFoo or C and D` is `(O is TFoo) or
  (C and D)`, `O is TFoo + 1` is `(O is TFoo) + 1` (`E2008`), `O is TFoo and
  C = D` is `((O is TFoo) and C) = D` (with `O` no TFoo, `C` True and `D`
  False it is True, where `(O is TFoo) and (C = D)` would be False); `is
  not` alike. Any other right operand follows the table: with a metaclass
  variable `CV`, a function `GC`, an element `A[0]`, `ClassType`, a typed
  constant, a cast `TFooClass(CV)` or even a PARENTHESIZED type name `(TFoo)`,
  `O is X and C` is `O is (X and C)` - `E2015`, while `O is X` alone compiles
  for each. The LEFT operand is never affected: `C and O is TFoo` is `(C and
  O) is TFoo` (`E2015`), and `in` has no such rule (4.6).
- ⚠️ *The table's grouping on the right of `is` also COMPILES* where an
  overloaded operator yields a class reference (dcc64 37.0, probed
  2026-09-26): with `R: TRec` and `class operator LogicalAnd(const A: TRec;
  B: Boolean): TClass`, `O is R and C` is `O is (R and C)` and compiles
  (`O is R` alone is `E2015`); so do `O is R + 1` over an `Add` operator,
  `O is GetRec and C` (a parameterless function returning a TRec), `O is (R)
  and C` and `O is RA[0] and C`.
  - For a parser this means: read the right operand of `is` as a FACTOR.
    When that factor is NAME-SHAPED - an identifier, a dotted chain, type
    arguments - take it for the type name: build the `is` node, then
    continue the term and simple-expression loops with it as their left
    operand. Any other factor (a parenthesis, a call, an index, a literal)
    is a value: continue ITS term and simple-expression loops as the right
    operand, the table's grouping. That is exact on valid code but for one
    shape no parser can tell apart: a name-shaped VALUE whose overloaded
    operator yields a class reference (`O is R and C` above) - telling a
    type name from a value takes name resolution.

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
- ⚠️ §4.9's type-name rule holds here too: `O is not TFoo and C` is `(O is not
  TFoo) and C` (dcc64 37.0, probed 2026-09-26).
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
- *Numeric casts go one way:* an integer to a real type converts (`Double(I)`
  compiles), a real to an integer type is `E2089 Invalid typecast`
  (`Integer(D)` - use `Trunc`/`Round`). An integer cast of an out-of-range
  CONSTANT truncates silently - `B := Byte(300)` compiles with no E1012
  (2.6.1). dcc64 and dcc32 37.0, 2026-10-30 (spec audit g2,
  local/probe-park/spec-audit/g2 A4.dpr, A6.dpr).
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
- ⚠️ *`Assert` reports the line of the token AFTER its call*, not the line of
  the call: the line is a constant argument of `System._Assert` in the code,
  taken when the parser stands on the next token. `if C then` / `Assert(X)` /
  `else` on three lines reports the `else`'s line, `Assert(X)` with its `;`
  on the next line that line. So a reformatting that only moves the token
  after an `Assert` - an `end` put right behind the `)`, a line break before
  a `;` - changes the compiled code (dcc64 and dcc32 37.0, probed
  2026-09-27).

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
Everything below was probed on dcc32 and dcc64 37.0 through the type dcc
USES: the overload a call of the result picks among one overload per builtin
type (`F(Ord(AC)).R_Byte` compiles only when `F(X: Byte)` was chosen), and
the type `var X := Intrinsic(...)` infers (3.1.3) - the two agree on every
row (2026-10-12; first probed 2026-09-07). ⚠️ The E2010 text of a DIRECT
assignment is no witness: `R := Ord(AC)` names `Integer` and `R := Succ(C)`
`Integer` while dcc types both calls `Byte` and `Cardinal` - the message
widens every narrow integer, and this table was once read from it (the rows
for `Ord`, `Hi` / `Lo`, `Succ` / `Pred` / `High` / `Low` of a 32- or 64-bit
unsigned and `Swap(Byte)` were wrong). Where the two compilers differ, both
answers are given.

⚠️ *A constant argument folds.* `Ord`, `Hi`, `Lo` and `Swap` of a literal, a
true constant or an enumeration value are `Integer`: `Ord('A')`, `Ord(5)`,
`Ord(e1)`, `Ord(True)`, `Ord(KC)` (`const KC = 'A'`), `Hi(KI)`, `Swap(KW)`
(`const KW = Word(7)`) - a typed constant is a variable here (`Ord(KW)` of
`const KW: Word = 5` is `Word`). `Succ` / `Pred` of one keep the argument's
type (`Succ(KE)` the enumeration, `Succ('a')` `Char`). As an overload
argument a constant result is a constant VALUE: `F(Ord(5))` and
`F(High(Byte))` take the narrowest overload holding the value (3.1.3), as
any constant does.

**Fixed result, whatever the argument**

| Intrinsic | Result | Notes |
|---|---|---|
| `Chr(X)` | `Char` | |
| `SizeOf(T\|X)`, `FilePos(F)`, `FileSize(F)` | `Integer` | |
| `Hi(X)`, `Lo(X)` | `Word` | of any width (`Hi(B)`, `Hi(I64)`); of a constant `Integer` |
| `NameOf(X)` | `string` | |
| `Trunc(X)`, `Round(X)` | `Int64` | of a `Single` too; `Trunc(I64)` of an integer is `E2008`, `Round(I)` is accepted |
| `MulDivInt64` | `Int64` | |
| `Assigned`, `Odd`, `IsManagedType`, `IsConstValue`, `HasWeakRef`, `Eof`, `Eoln`, `SeekEof`, `SeekEoln`, `AtomicCmpExchange128` | `Boolean` | |
| `Pi` | `Extended` | dcc64 prints it as `Double`, which is what `Extended` IS on a 64-bit target; `var X := Pi` reads `Extended` on both |
| `Addr(X)`, `Ptr(N)`, `ReturnAddress`, `AddressOfReturnAddress`, `TypeInfo(T)`, `TypeHandle(T)` | `Pointer` | the untyped pointer, not `^T` or `PTypeInfo` |
| `GetTypeKind(T\|X)` | `System.TTypeKind` | |
| `Length(X)` | `Integer`; `NativeInt` for a **dynamic array** | so `Length(DynArr)` is `Int64` on dcc64 and `Integer` on dcc32; a string, static array, `ShortString` or `Variant` is `Integer` on both |

**`Ord(X)`**

- A builtin integer up to 32 bits keeps its type: `Ord(B)` `Byte`, `Ord(Sh)`
  `ShortInt`, `Ord(W)` `Word`, `Ord(SI)` `SmallInt`, `Ord(I)` `Integer`,
  `Ord(C)` `Cardinal`; a 64-bit one is `Integer` (`Ord(I64)`, `Ord(U64)`,
  and `Ord(NI)` / `Ord(NU)` on dcc64 - `Ord(NU)` is `Cardinal` on dcc32,
  where NativeUInt is Cardinal).
- `Char` / `WideChar` `Word`, `AnsiChar` `Byte`; `Boolean` `ShortInt`,
  `ByteBool` `Byte`, `WordBool` `Word`, `LongBool` `Integer`.
- An enumeration, or a subrange of any ordinal, is the NARROWEST of
  `ShortInt`, `Byte`, `SmallInt`, `Word`, `Integer`, `Cardinal` holding its
  ordinal range, signed first: a four-value enumeration, `False..True`,
  `1..10`, `-5..5` and `'a'..'z'` `ShortInt`; `0..200` `Byte`; `0..1000`
  and `#0..#$3000` `SmallInt`; `0..60000` `Word`; `0..70000` `Integer`.
- A constant: `Integer` (above).

**The ordinal step rule** (shared by `Pred`, `Succ`, `Low`, `High`):

- `Byte`, `ShortInt`, `Word`, `SmallInt`, an integer subrange: **`Integer`**
  - `Pred(B)` of a `Byte` is an `Integer`, `Succ` of a `0..200` too.
- A 32- or 64-bit builtin keeps its type: `Succ(C)` and `High(Cardinal)` are
  `Cardinal`, `Succ(U64)` and `High(UInt64)` `UInt64`, `Succ(NI)`
  `NativeInt`, `High(NativeUInt)` `NativeUInt` (so `Int64` / `UInt64` on
  dcc64, `Integer` / `Cardinal` on dcc32).
- An enum, `Char`, `AnsiChar`, `WideChar`, `Boolean`: **the argument's own
  type** - `Pred(E)` is the enum, `Succ(AC)` an `AnsiChar`; a subrange of
  one is its BASE type: `Pred(CS)` of a `'a'..'z'` is `Char`, `Succ(ES)` and
  `High(TES)` of an `e1..e2` the enumeration (E2010 prints `Enumeration`;
  `F(High(TES))` takes `F(X: TE)`).
- A subrange OVER an `Int64` range (`1..5000000000`): `Int64`.

`Abs` of an integer widens instead: `Byte` .. `Cardinal` `Integer` (`Abs(C)`
is `Integer`), `Int64` and `UInt64` `Int64`, `NativeInt` / `NativeUInt`
`NativeInt`.

**`Low`/`High` of a non-ordinal**

| Argument | `Low` | `High` |
|---|---|---|
| static array | index type, by the step rule above: `array[2..5]` and `array[Word]` give `Integer`, `array[TEnum]` gives `TEnum`, and so does an enumeration-value range `array[eA..eC]` or a named enumeration subrange `array[TES]` (dcc64, 2026-10-12: `P(High(A))` takes `P(X: TEnum)` beside `P(X: Integer)`), `array['a'..'z']` and `array[TCS]` of a `TCS = 'a'..'z'` give `Char` (2026-10-12), `array[Boolean]` gives `Boolean` | same |
| dynamic array (value or type) | `Integer` | `NativeInt` (`Int64` on dcc64) |
| `string`, `AnsiString`, `WideString`, `ShortString` values | `Integer` | `Integer` |
| the long-string TYPE itself (`High(string)`) | `Integer` | `E2198 High cannot be applied to a long string` |
| a set type or value | `E2008` | `E2008` |

**Argument-dependent**

| Intrinsic | Rule (dcc32 / dcc64 where they differ) |
|---|---|
| `Abs(X)` | integer: the widening note under the step rule (`Abs(Byte)` = `Integer`, `Abs(UInt64)` = `Int64`, `Abs(NativeUInt)` = `NativeInt`). Real: **`Extended` on dcc32 for every real type**; on dcc64 `Currency` stays `Currency` and `Comp` stays `Comp` (their arithmetic is integral there), every other real is `Extended` (printed `Double`). `Abs(2.5)` is a real. `Abs(Variant)` is a real |
| `Sqr(X)` | integer: `Integer` for anything up to 32-bit SIGNED, but a 32-bit **unsigned stays `Cardinal`**, and the 64-bit ones keep their signedness (`Int64`, `UInt64`, `NativeInt`, `NativeUInt`). Real: `Extended` for every real, **`Currency` and `Comp` included** (unlike `Abs`), and so is `Sqr(Variant)` (dcc32 / dcc64, 2026-10-12) |
| `Swap(X)` | a 16- or 32-bit integer keeps its type (`Word`, `SmallInt`, `Integer`, `Cardinal`); `Byte` and `ShortInt` give `SmallInt` (2026-10-12; the row said `Integer`), `Int64` and a constant `Integer` |
| `Copy(S, I, N)` / `Copy(A, I, N)` | the FIRST argument's type: `AnsiString` stays `AnsiString`, `ShortString` stays `ShortString`, a string literal is `string`, a dynamic array (incl. `TArray<T>`) stays that array type; `Copy(A)` with no range works for a dynamic array and is `E2035` for a string; a static array is `E2008` |
| `Concat(A, B, ...)` | the `+` of 4.2.1 folded left to right: all arguments of ONE type give that type (`AnsiString`, `WideString`, `UTF8String`, `RawByteString`, a dynamic array type), **`ShortString`+`ShortString` is `AnsiString`** and **`AnsiChar`+`AnsiChar` is `ShortString`**, and a literal takes the other arguments' string type - `Concat(A, 'x')`, `Concat('x', A)`, `Concat(A, A, 'x')`, `Concat(SS, 'x')`, `Concat(SS, A)` are `AnsiString`, `Concat(U8, 'x')` `UTF8String`, `Concat(WS, 'x')` and `Concat(WS, A)` `WideString` (dcc32 / dcc64, 2026-10-12; this row said "literals: string"). Two code pages (`Concat(A, U8)`), `Char` operands and literals alone: `string` |
| `Default(T)` | `T` for a record, an ordinal, a real, a string, a STATIC array. **`Pointer` for every reference type** - class, interface, dynamic array, procedural/method/reference type, class reference, pointer: `var X := Default(TObj); X.Free` is `E2018 Record, object or class type required` and `var A := Default(TDynI); A[0]` is `E2016` |
| `AtomicIncrement`/`AtomicDecrement`/`AtomicExchange`/`AtomicCmpExchange` | the first argument's type (`Integer`, `Cardinal`, `Int64`, `UInt64`, `NativeInt`, `Pointer`); an object reference is `E2008` |

**Semantics & parsing notes**

- ⚠️ *A typer that gives `Ord` a 64-bit argument's type or `Trunc` "Integer"
  will produce false `E2010`s at the next assignment* - `I64 := Trunc(D)` is
  fine, `var N := Ord(I64)` is an `Integer`. And one that widens `Ord(Ch)` to
  `Integer` binds the wrong overload: `IntToHex(Ord(Ch))` is the `UInt16`
  one, `Ord(Ch).ToHexString` TWordHelper's. The rows above are the contract.
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
