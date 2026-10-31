# 05 — Statements & Control Flow

Statements are the executable units inside a routine body, between `begin` and
`end`. This chapter is the **exemplar** for the whole reference: it shows the
parser-oriented per-feature template (see
[README](README.md#per-feature-template)). Shared non-terminals
(`Expression`, `Designator`, `Ident`, `ConstExpr`, `TypeRef`…) are defined in
[Appendix B](B-lexical-grammar.md) and referenced by name here.

> Related: expressions & operators → [04](04-expressions-operators.md);
> `try`/`raise` → [18-exceptions.md](18-exceptions.md).

## Chapter grammar umbrella

```ebnf
Statement           = [ LabelId ":" ] ( SimpleStatement | StructuredStatement ) ;
StatementList       = Statement { ";" Statement } ;

SimpleStatement     = Assignment | CallStatement | GotoStmt | (* empty *) ;
StructuredStatement = CompoundStmt | IfStmt | CaseStmt
                    | ForStmt | ForInStmt | WhileStmt | RepeatStmt
                    | WithStmt | TryStmt | RaiseStmt     (* TryStmt/RaiseStmt → ch.18 *)
                    | AsmStmt ;                          (* inline assembly → ch.06 §6.10 *)
```

> **Parser note:** the `;` is a statement **separator**, not a terminator, so an
> *empty statement* is legal (e.g. a `;` right before `end`). The AST builder
> should silently drop empty statements.

---

## 5.1 Simple statements

### 5.1.1 Assignment statement (`:=`)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Assigns the value of an expression to a variable, field, property, or
dereferenced pointer.

**Grammar**

```ebnf
Assignment = Designator ":=" Expression ;
```

**Example**

```pascal
X := 42;
Edit1.Text := 'Hello';
PNode^.Value := 1;
```

**Semantics & parsing notes**

- *No compound assignment:* Object Pascal has **no** `+=`, `-=`, `++`, etc. `:=`
  is the only assignment operator. Do not add those to the grammar.
- *LHS class:* the `Designator` must be assignable — a variable, `var`/`out`
  parameter, field, indexed/array element, dereference, or a **property with a
  write specifier**. Assigning to a read-only property is a semantic error.
- *Property dispatch:* if the LHS resolves to a property, lower to the setter
  call/field-store named by its `write` clause — the AST should record that the
  target is a property so later passes can expand it.
- ⚠️ *This "assignable" class is NOT the same set that qualifies as a `var`-parameter
  argument*: a property with a `write` specifier is a valid `:=` LHS but is
  `E2197` as a `var` argument. The rule and its probe live in 6.2.2; do not
  reuse one "assignable designator" check for both contexts.
- *Type rule:* RHS must be assignment-compatible with the LHS type (implicit
  conversions per ch.04; otherwise error).
- *AST:* `Assign { target: Designator, value: Expression }`.

### 5.1.2 Procedure / method call statement

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Invokes a procedure, function (discarding the result), or method as a statement.

**Grammar**

```ebnf
CallStatement = Designator [ "(" [ ActualParams ] ")" ] ;
ActualParams  = Expression { "," Expression } ;
```

**Example**

```pascal
DoWork(Input, 10);
Application.Run;        // zero-arg: parentheses optional
Obj.Method;
```

**Semantics & parsing notes**

- ⚠️ *Key ambiguity — call vs. reference:* because a zero-arg call needs **no
  parentheses**, a bare `Designator` that names a routine is **ambiguous**
  between *calling* it and *referencing it as a procedural value*. Resolution is
  **context-sensitive**: in a context expecting a procedural/method-pointer type
  (assignment to a `procedure of object` var, `@`-free event hookup) it is a
  *reference*; otherwise it is a *call*. The parser must defer this to semantic
  analysis with the operand type known.
- *`@` operator:* `@Foo` / `Foo` distinctions interact with the typed-`@` mode
  `{$T}` / `{$TYPEDADDRESS}` (under `{$T+}`, `PB := @I` with `PB: ^Byte`, `I:
  Integer` is `E2008 Incompatible types`) - note for ch.04. dcc64 37.0,
  2026-10-30 (spec audit g3, local/probe-park/spec-audit/g3 TypedAt.dpr).
- *AST:* `Call { callee, args[] }` used in statement position
  (`ExprStmt`). Record whether parentheses were present (empty arg list vs.
  no arg list) — relevant for the reference-vs-call decision.

---

## 5.2 Compound & structured statements

### 5.2.1 The `begin … end` block

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Groups several statements into one, wherever a single statement is expected.

**Grammar**

```ebnf
CompoundStmt = "begin" [ StatementList ] "end" ;
```

**Example**

```pascal
begin
  Prepare;
  Execute;
end;
```

**Semantics & parsing notes**

- *Scope:* a compound statement declares nothing by itself, but it IS the scope
  of the **inline `var`/`const` declarations** (10.3+) written in it: such a
  name is visible from its declaration to the compound's `end` and no further -
  used after the block it is E2003 (undeclared); two sibling blocks may each
  declare the same name; a nested block may not redeclare a name an enclosing
  block of the routine declared (E2004) (see also 5.5.1; dcc64 37.0, probed
  2026-09-26). The full rule - what a body holds and what an inline
  declaration may hide - is 3.1.3's; this is its statement-side summary.
- *Finalization:* a managed inline variable is finalized at the end of its
  block. So is the temporary holding a call statement's DISCARDED managed
  result (`S;` for a function returning a string, an interface, a dynamic array
  or a record with managed fields): at the end of the statement list the call
  stands in - a routine's own block: its epilogue; as the body of an `if`, a
  loop, a `case` branch, a `with` or an exception handler, at the end of that
  statement. A `begin`/`end` around such a call therefore moves the
  finalization up to the call; around any other statement it changes nothing in
  the code (dcc64 37.0, probed 2026-09-26: a runtime destructor trace, and
  byte-identical `.dcu` files for every other shape).
- *Empty body:* `begin end` is legal.
- *AST:* `Block { statements[] }`.

---

## 5.3 Conditional statements

### 5.3.1 `if … then … else` (statement)

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Executes a branch based on a Boolean expression.

**Grammar**

```ebnf
IfStmt = "if" Expression "then" Statement [ "else" Statement ] ;
```

**Example**

```pascal
if Score >= 50 then
  ShowMessage('Pass')
else
  ShowMessage('Fail');
```

**Semantics & parsing notes**

- ⚠️ *Dangling `else`:* the grammar is ambiguous; resolve with the standard rule —
  an `else` binds to the **nearest** preceding `if` that has no `else`.
  The rule holds where a `case` or an `except` part could take the `else` too:
  in `case X of 1: if A then B else C end` and in `except on E: T do if A then
  B else C end` the `else` is the `if`'s - C runs for X = 1 when A is False,
  resp. for an exception of type T - and a second `else` goes on to the case or
  the except part; after a `;` (`1: if A then B; else C`) it is theirs (dcc64
  37.0, probed 2026-09-26 at run time and by wrapping each branch in
  `begin`/`end`: byte-identical `.dcu`).
- ⚠️ *No `;` before `else`:* a semicolon after the `then`-branch terminates the
  `if` statement, leaving `else` orphaned → syntax error. The parser must treat a
  `;` immediately before `else` as an error (common mistake worth a clear
  diagnostic).
- *Type rule:* the condition must be `Boolean` (Object Pascal does not coerce
  integers to Boolean here): `E2012 Type of expression must be BOOLEAN`. The
  same rule holds for `while`, `until` and an inline `if`'s condition; what is
  accepted besides Boolean (any `ByteBool`/`WordBool`/`LongBool`, a
  `Variant`, a record with `Implicit` to Boolean) is stated once in 2.1.1 /
  2.2.2. dcc64 37.0, 2026-10-30 (spec audit g3, local/probe-park/spec-audit/g3
  CondTy.dpr).
- *AST:* `IfStmt { cond, thenStmt, elseStmt? }`. Distinct node from the inline-`if`
  *expression* (5.4.1).

### 5.3.2 `case … of`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Selects one branch by matching an ordinal selector against constant labels/ranges.

**Grammar**

```ebnf
CaseStmt     = "case" Expression "of"
                 CaseSelector { ";" CaseSelector } [ ";" ]
                 [ "else" StatementList [ ";" ] ]
               "end" ;
CaseSelector = CaseLabels ":" Statement ;
CaseLabels   = CaseLabel { "," CaseLabel } ;
CaseLabel    = ConstExpr [ ".." ConstExpr ] ;
```

**Example**

```pascal
case KeyChar of
  'a'..'z', 'A'..'Z': HandleLetter;
  '0'..'9':           HandleDigit;
else
  HandleOther;
end;
```

**Semantics & parsing notes**

- ⚠️ *Selector type:* must be an **ordinal** type (integer, `Char`, enumerated,
  `Boolean`). **Strings and floats are NOT allowed** — do not parse string case
  labels (unlike C#/Pascal dialects that allow them). Enforce in semantic check.
  A string or real selector is `E2001 Ordinal type required`; a `Variant`
  selector is accepted (2.1.1's position table) and matches `V := '2'`
  against the label `2` at run time.
- *Labels:* each `CaseLabel` must be a **compile-time constant** (or constant
  range `a..b`); ranges and label sets must not overlap and must fit the selector
  type. Overlap is a compile error. The codes (dcc64 37.0):
  - a repeated value or an overlapping range (`1: ; 1:` or `1..5: ; 3..7:`)
    is `E2030 Duplicate case label`;
  - a reversed range `5..1` is `E2011 Low bound exceeds high bound`;
  - a typed constant (`const TC: Integer = 3`) is `E2026 Constant expression
    expected`;
  - a label of another type is `E2010` (`case E of 1:` over an enumeration:
    `'TE' and 'Integer'`; `case C of 'ab':` over a Char: `'Char' and
    'string'`);
  - a label outside the selector's range is `E1012 Constant expression
    violates subrange bounds` (`case B of 300:` over a Byte, `case W of -1:`
    over a Word) - an ERROR, except over an `Int64` / `UInt64` selector,
    where a label beyond 32 bits is only `W1012` (next bullet).
- ⚠️ *A 64-bit selector is compared on its low 32 bits only.* `case I64 of
  5000000000: ...` warns `W1012 Constant expression violates subrange
  bounds`, and at run time the label matches both `I64 = 5000000000` and
  `I64 = 705032704` (its low 32 bits); `case I64 of 3:` matches `I64 =
  4294967299`. A Cardinal selector with a label `4000000000` is exact and
  silent. Same on dcc32. A constant folder must truncate the selector and
  the labels to 32 bits for an Int64 / UInt64 case, not compare full values.
  dcc64 / dcc32 37.0, 2026-10-30 (spec audit g3,
  local/probe-park/spec-audit/g3 CaseLbl.dpr, CaseI64.dpr, CaseI64b.dpr).
- *`else` vs `otherwise`:* the standard keyword is `else`; some dialects accept
  `otherwise` — Delphi uses `else`.
- ⚠️ *case-`else` vs `if`-`else`:* an `else` right after a branch whose
  statement is an `if` without `else` is that `if`'s (5.3.1); the case gets
  one only after a `;` or after the `if`'s own `else` branch. A nested `case`
  takes the `else` before the outer one does.
- *AST:* `CaseStmt { selector, branches: [ { labels[], body } ], elseBranch? }`.
  Keep label ranges as `{ lo, hi }` pairs.

---

## 5.4 Conditional expressions

### 5.4.1 Inline `if` expression (ternary operator)

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

An **expression** that yields one of two values based on a condition — the
Object Pascal equivalent of `?:`.

**Grammar**

```ebnf
InlineIfExpr = "if" Expression "then" Expression "else" Expression ;
```

**Example**

```pascal
var Max := if A > B then A else B;
Caption := if Connected then 'Online' else 'Offline';
```

**Semantics & parsing notes**

- ⚠️ *Statement vs. expression:* the leading token `if` is shared with the
  `if`-*statement* (5.3.1). Disambiguate by **parse context** — in an expression
  position (RHS of `:=`, actual parameter, etc.) parse `InlineIfExpr`; in
  statement position parse `IfStmt`.
- *Mandatory `else`:* unlike the statement form, `else` is **required** (an
  expression must always produce a value) → no dangling-`else` problem here.
- *Type rule:* the branches must meet in a common type by the concrete rules
  below - NOT merely be assignable to one: both `1` and `2.5` are assignable
  to Double, yet `D := if C then 1 else 2.5` is `E2010 Incompatible types:
  'Integer' and 'Double'` and `D := if C then 2.5 else 1` `E2010 ... 'Double'
  and 'Integer'` - the refusal is symmetric and the Double target does not
  help; the same for variables (`if C then I else D`, `if C then D else I`)
  and for an integer with a Char or string (`if C then 1 else 'x'`, `if C
  then Ch else 1`). dcc64 37.0, 2026-10-31 (PasTree audit E1-42,
  local/probe-park/specfu/E1-42 E5.dpr). The common type is the type of the
  whole expression. What it is (dcc32 / dcc64 37.0, probed 2026-10-12
  through `var X := if ...`): one type stays itself, a distinct one too (`if C then MI else
  MI` is `TMyI`); integers meet in the narrowest type covering both, a
  constant adapting as in 4.2.1 - `if C then B else W` Word, `B else Sh`
  SmallInt, `C else I` and `C else Sh` Int64, `1 else C` Cardinal, `1 else
  I64` Int64, `I64 else U64` UInt64, `MW else W` Word; two reals Extended
  (`Sg else D`), Currency on dcc64 with a Currency value (`Cu else D`);
  text string when a side is string, Char or a literal (`'a' else 'bc'`, `A
  else 'x'`, `'x' else A` - the literal does NOT take the AnsiString's type
  here, unlike `+`), except that a Char value with a one-character literal
  stays Char (`Ch else 'x'`; `Ch else 'xy'` is string), and WideString
  beside WideString (`A else WS`); `nil` takes the other branch (`nil else
  Ob2` TO2, `Ob1 else nil` TO1, `nil else nil` Pointer); two classes the
  ancestor, in either order (`Ob2 else Ob1` and `Ob1 else Ob2` are TO1); a
  set constructor the other branch's set type (`St else [e1]` TES). An
  integer THEN branch with a real ELSE branch is `E2010` (`if C then 1 else
  2.5`), and so is a Currency branch with an integer one.
- *Evaluation:* short-circuit — only the selected branch is evaluated (contrast
  with the `IfThen` RTL functions, which evaluate both arguments).
- ⚠️ *Extent and precedence* (dcc64 37.0, probed 2026-09-26 - the grammar
  above is exactly what dcc does): the inline `if` is a **Factor** - it may
  stand as any operand (`X := 1 + if C then 10 else 20` compiles, `-if C then
  1 else 2` too) - and each of its three parts is a full `Expression`, so the
  `else` branch runs to the end of the enclosing expression: `if C then 1
  else 2 + 3` is `if C then 1 else (2 + 3)` (C true gives 1, not 4), `100 -
  if C then 10 else 20 - 5` is `100 - (if C then 10 else (20 - 5))`, `-if C
  then 1 else 2 + 3` is `-(if C then 1 else (2 + 3))`, and a relational
  operator is taken too: `B := if C then 1 else 2 = 2` is `E2010 'Integer'
  and 'Boolean'` (the `else` branch is `2 = 2`). Only parentheses end it
  early: `(if C then 1 else 2) + 3`. Nested forms need none: `if C1 then if
  C2 then 1 else 2 else 3`, `if C1 then 1 else if C2 then 2 else 3`.
- ⚠️ *Not a constant expression:* `const K = if True then 1 else 2;` is
  `E2026 Constant expression expected`.
- ⚠️ *No target type reaches the branches:* with `A: TArray<Integer>`, `A :=
  if C then [1] else [2]` is `E2010` (the constructors read as sets), and so
  is `A := if C then A + [X] else A`; array-valued branches without a
  constructor work (`if C then A else A2`, `if C then A + A2 else A2`).
- *AST:* `InlineIf { cond, thenExpr, elseExpr }`.

---

## 5.5 Loops

### 5.5.1 `for … to` / `downto`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) · inline `var` counter 10.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Counted loop over an ordinal range.

**Grammar**

```ebnf
ForStmt = "for" [ "var" ] Ident [ ":" TypeRef ] ":=" Expression
          ( "to" | "downto" ) Expression "do" Statement ;
```

**Example**

```pascal
for I := 1 to 10 do
  Sum := Sum + I;

for var J := High(A) downto Low(A) do   // inline counter, 10.3+
  Process(A[J]);
```

**Semantics & parsing notes**

- *Counter type:* must be an **ordinal** type. With inline `var` (10.3+) the type
  is **inferred** from BOTH bounds; an explicit `: TypeRef` is allowed.
  Integer bounds meet as the operands of `+` do (4.2.1): at least Integer,
  a constant bound taking the other bound's type when it fits, a 32-bit
  signed / unsigned mix Int64 - `B to B`, `B to W`, `Sh to B`, `2 to 9`,
  `B to 10` are Integer; `C to C` and `0 to C` Cardinal; `C to I`, `I to C`,
  `-1 to C`, `0 to I64`, `0 to 5000000000` Int64; `U64 to 0` UInt64; `0 to
  High(DynArr)` and `NI to 0` Int64 on dcc64 and Integer on dcc32; a distinct
  `MI to MI` (`type Integer`) Integer. A Char, enumeration or Boolean counter
  is its bounds' type (`Ch to 'z'` Char, `e0 to e2` the enumeration) - dcc32
  and dcc64 37.0, probed 2026-10-12.
- ⚠️ *What may be the counter* (dcc64 37.0): a local variable of the routine,
  or a VALUE parameter (`procedure P(N: Integer); begin for N := ...` compiles).
  dcc parses a full designator after `for` and rejects the wrong ones
  semantically, so `for R.F := ...` is not a syntax error:
  - `E1019 For loop control variable must be simple local variable` for a
    field (`R.F`, a method's own field `FI`), an array element (`A[0]`), a
    `var` parameter and an enclosing routine's local used from a nested
    routine;
  - `E2064 Left side cannot be assigned to` for a `const` parameter;
  - `E2032 For loop control variable must have ordinal type` for a real
    counter;
  - a unit/program-level variable is only `W1019` (same text) inside a
    routine, and silent as the counter of a loop in the main program block.
  The bounds must be assignment-compatible with the counter: `for L := 0 to
  300` over a Byte `L` is `E1012`, `for L := 0 to 'a'` over an Integer
  `E2010`. The same location rules hold for a `for ... in` element variable
  (5.5.2). dcc64 37.0, 2026-10-30 (spec audit g3,
  local/probe-park/spec-audit/g3 ForCtr.dpr).
- *Bounds evaluated once:* both bounds are computed **before** the first
  iteration and not re-evaluated — model this in lowering (don't re-emit the
  limit expression per iteration).
- ⚠️ *Counter is read-only in the body:* assigning to the loop variable inside the
  body is a compile error; its value **after** normal completion is undefined.
  Reading it after the loop warns `W1037 FOR-Loop variable 'I' may be
  undefined after loop` - but not when the loop body contains a `Break`
  (`for I := 0 to 3 do if I = 2 then Break; Writeln(I)` is silent). dcc64 37.0, 2026-10-30
  (spec audit g3, local/probe-park/spec-audit/g3 ForAfter.dpr).
  - *What counts as a write* - each `E2081 Assignment to FOR-Loop variable
    'I'`: `I := ...`; `Inc(I)` / `Dec(I, 2)`, bare or `System.Inc(I)`; a
    nested `for I := ...` (`to` or `downto`) or `for I in ...` on the same
    counter; any argument of `Read` / `ReadLn` (`ReadLn(I)`, `Read(F, I)`,
    `ReadLn(C, I)`); the value argument of `Val(S, I, Code)`. The same holds
    for an inline `for var K` counter (`for var K := 0 to 9 do for K := 0 to
    1 do`). Passing the counter to a `var` or `out` parameter, to an untyped
    `var` one (`FillChar(I, 4, 0)`) or as `Val`'s `Code` is only `W1015
    FOR-Loop variable 'I' cannot be passed as var parameter`; `@I` is
    accepted. dcc reports a nested `downto` loop inside a `to` loop twice, on
    the same line; once is enough for a tool. dcc64 37.0, 2026-10-29 (PasTree
    audit E3-15, local/probe-m-b/e315 fc.dpr, fc2.dpr, fc3.dpr).
- ⚠️ *Scope of inline counter:* limited to **the loop statement** — an exception
  to the general to-end-of-enclosing-block rule of 03 §3.1.3. Sibling loops may
  reuse the same counter name; a resolver that declares the counter into the
  enclosing block scope produces false E2004 redeclarations (dcc-verified).
- *AST:* `ForStmt { counter, inlineDecl?, startExpr, limitExpr, dir: to|downto, body }`.

### 5.5.2 `for … in` (for-in loop)

| | |
|---|---|
| **Introduced** | 2005 · inline `var` element 10.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Iterates over the elements of a collection.

**Grammar**

```ebnf
ForInStmt = "for" [ "var" ] Ident [ ":" TypeRef ] "in" Expression "do" Statement ;
```

**Example**

```pascal
for C in 'Pascal' do Write(C);
for var Item in MyList do Process(Item);
```

**Semantics & parsing notes**

- *Iterable resolution (the important bit):* the `in`-expression is valid if it is
  a string, static/dynamic array, or set — **or** a class, record or
  interface satisfying the **enumerator pattern**: it exposes `function
  GetEnumerator` returning a type with `function MoveNext: Boolean` and a
  `property Current`. The loop element type = the type of `Current`. The
  parser emits a `ForInStmt`; **semantic analysis** resolves `GetEnumerator`
  and the element type.
- ⚠️ *A helper's `GetEnumerator` counts for a class or a record only*
  (Delphi has no interface helpers):
  - a class helper's hides the class's own (15.3.3) - over `TList0` with its
    own Integer enumerator, `for var E in L do E.OnlyFoo` walks the helper's
    `TFoo`; a record helper serves a record that has none;
  - a string, an array or a set iterates natively and the helper is ignored:
    with a helper GetEnumerator on `string`, `TArray<Integer>` and `set of
    TEn`, the elements are still Char (`E.OnlyFoo` is `E2018`), Integer and
    TEn (`E2003 Undeclared identifier: 'OnlyFoo'`);
  - anything else is `E2430` even with a helper whose GetEnumerator fits:
    `Integer`, `Double`, an enum, a typed pointer - though the helper itself
    is active (`N.GetEnumerator.Current.OnlyFoo` compiles);
  - a record with BOTH its own GetEnumerator and a record helper's stops
    dcc64 with `F2084 Internal Error: AV00B366C3(009F0000)-R0000000C-0` -
    code that compiles never has one.
  dcc64 37.0, 2026-10-31 (PasTree audit E1-43,
  local/probe-park/specfu/E1-43 C1-C11).
- ⚠️ *The pattern is checked member by member, and stricter than "has these
  names"* (dcc64 37.0). Each failure is `E2431 for-in statement cannot
  operate on collection type 'X' because 'Y' does not contain a member for
  'M', or it is inaccessible`:
  - `Current` must be a PROPERTY: an enumerator with a function `Current`
    (and a `GetCurrent`), or a record with a FIELD `Current`, fails for
    `'Current'`;
  - `MoveNext` must be a function returning `Boolean`: one returning Integer
    fails for `'MoveNext'`, as does an enumerator without it;
  - ⚠️ the members must be PUBLIC, whatever the loop's position - NOT
    "accessible" in the ordinary 11.x sense: a `private`, `strict private` or
    `protected` `GetEnumerator` fails for `'GetEnumerator'` in the declaring
    unit too, and even in the class's own method (`for X in Self do`), while
    a direct `H.GetEnumerator` call of the protected one in the same unit
    compiles; a `private`
    `MoveNext` of the enumerator fails for `'MoveNext'`. A type with no
    `GetEnumerator` (`TObject`) fails the same way.
  A collection that is no candidate at all - an `Integer`, `nil` (typed
  `Pointer`) - is `E2430 for-in statement cannot operate on collection type
  'Integer'`; a TYPE written as the collection (`for X in TArray<Integer>
  do`) is `E2029 '(' expected but 'DO' found`. A `GetEnumerator` that is a
  PROCEDURE crashes dcc (`F2084 Internal Error: AV...`) - code that compiles
  never has one.
- *The element variable* follows the 5.5.1 counter's location rules (`E1019`
  for a field or a `var` parameter, `W1019` for a global in a routine) and is
  read-only in the body (`X := 5` and `Inc(X)` are `E2081`). `Current` need
  only be ASSIGNMENT-compatible with it: a Byte or Int64 variable over a
  `TArray<Integer>`, a `string` variable over a string's Chars and a Byte
  over a `set of Byte` all compile; a `string` over a `TArray<Integer>` is
  `E2010 Incompatible types: 'string' and 'Integer'`. A dereferenced pointer
  (`for X in P^`) is an ordinary collection expression. dcc64 37.0,
  2026-10-30 (spec audit g3, local/probe-park/spec-audit/g3 ForInEn.dpr,
  ForInPriv.dpr, ForInPriv2.dpr, ForInPriv3.dpr, ForInVar.dpr, Flow.dpr).
- A string walks its character type (7.2.1): `Char` over `string`,
  `UnicodeString` and `WideString`, `AnsiChar` over `AnsiString`,
  `ShortString`, `string[N]` (named or inline), `RawByteString`, `UTF8String`
  and any `type AnsiString(cp)` (dcc32 / dcc64 37.0, 2026-10-12).
- *Element type of an untyped `for var E in` over a collection with no named
  type* (dcc64 37.0, 2026-10-07, read off `TypeInfo` of the element):
  - `[a, b]` with no range is an **open array constructor**: `['.pas', '.inc']`
    walks `string`, `['a', 'b']` `Char`, `['a', 'bc']` and `['bc', 'a']`
    `string`, `[1, 2]` and `[1, 2, 300]` `Integer`, `[1, 3000000000]` `Int64`,
    `[B]` over a Byte `Byte`, `[AList]` the list's class, `[True]` `Boolean`;
    `[B, W]` over a Byte and a Word an anonymous subrange (`:2`); `[1.5, 2]`
    is E2624 (no common element type);
  - a range makes it a **set constructor**, and the loop walks its base type:
    `['a'..'c']` `AnsiChar` (a set of Char is a set of AnsiChar), an enum
    range the enum;
  - `+` joins arrays: `[ADir] + APaths`, `APaths + [ADir]`, `APaths + APaths`
    and `[ADir] + ['q']` over `APaths: TArray<string>` all walk `string`. A
    static array does not join a dynamic one (`CArr + APaths` is E2008).
- *Distinguish from 5.5.1* by the `in` keyword vs. `:=`.
- ⚠️ *Scope of inline element:* same rule as the 5.5.1 counter — the `for var E`
  element is scoped to **the loop statement**, not the enclosing block; sibling
  for-in loops may reuse the element name (dcc-verified: two consecutive
  `for var LWord in ...` loops over different arrays).
- *AST:* `ForInStmt { elementVar, inlineDecl?, collection, body }`.

### 5.5.3 `while … do`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Pre-tested loop; body may run zero times.

**Grammar**

```ebnf
WhileStmt = "while" Expression "do" Statement ;
```

**Example**

```pascal
while not Eof(F) do
  ReadLn(F, Line);
```

**Semantics & parsing notes**

- *Condition* must be `Boolean`, tested **before** each iteration.
- *AST:* `WhileStmt { cond, body }`.

### 5.5.4 `repeat … until`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Post-tested loop; body runs at least once.

**Grammar**

```ebnf
RepeatStmt = "repeat" [ StatementList ] "until" Expression ;
```

**Example**

```pascal
repeat
  Attempt;
until Succeeded or (Tries >= MaxTries);
```

**Semantics & parsing notes**

- *Self-bracketing:* `repeat`/`until` delimit a **statement list** directly — no
  `begin/end` is needed (unlike `while`/`for`, whose body is a single
  `Statement`). The grammar reflects this asymmetry.
- *Condition* is `Boolean`, tested **after** the body; loop ends when it is `True`.
- *AST:* `RepeatStmt { body: statements[], cond }`.

---

## 5.6 Flow-control statements

> ⚠️ **Lexical fact for all of 5.6.1–5.6.3 (and 5.6.5, `Halt`):** `Break`,
> `Continue`, and `Exit` are
> **standard (intrinsic) procedures declared in `System`, not reserved words.**
> They parse as ordinary `CallStatement`s. Their loop/routine-control meaning is
> applied during semantic analysis, which also rejects `Break`/`Continue` outside
> a loop. Consequence: they can be shadowed by a user identifier, and the lexer
> must **not** tokenise them as keywords. Verified at run time: with a program's
> own `procedure Break;` a `Break` in a `for` body calls it and the loop runs
> on; with `procedure Exit(N: Integer)` `Exit(5)` in a function calls it and
> the function goes on; `System.Break` / `System.Exit` reach the intrinsics
> past such a declaration (and past a variable named `Break`). dcc64 37.0,
> 2026-10-30 (spec audit g3, local/probe-park/spec-audit/g3 Shadow.dpr,
> ShadowL.dpr).
>
> Codes and limits (dcc64 37.0, 2026-10-30, Flow.dpr): `Break` / `Continue`
> outside a loop is `E2097 BREAK or CONTINUE outside of loop` - and an
> anonymous method's body is outside the loop it is written in
> (`for ... do P := procedure begin Break end` is E2097). Leaving a
> `finally` part with any of the three is `E2126` - the rule, and its
> empty-guarded-part exception, are 18's; from an `except` part all three
> are allowed (FinBrk.dpr).

### 5.6.1 `Break`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Exits the innermost enclosing `for`/`while`/`repeat` loop.

**Grammar**

```ebnf
(* no dedicated production — parsed as CallStatement "Break" *)
```

**Example**

```pascal
for I := 0 to Count - 1 do
  if Items[I] = Target then Break;
```

**Semantics & parsing notes**

- *Binds to* the innermost loop; error if used outside any loop.
- *AST:* may stay `Call("Break")` until a lowering pass rewrites it to
  `BreakStmt { targetLoop }`.

### 5.6.2 `Continue`

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Skips to the next iteration of the innermost enclosing loop.

**Grammar**

```ebnf
(* parsed as CallStatement "Continue" *)
```

**Example**

```pascal
for I := 0 to Count - 1 do
begin
  if Items[I] = nil then Continue;
  Process(Items[I]);
end;
```

**Semantics & parsing notes**

- Same intrinsic/shadowing rules as `Break`. `AST: ContinueStmt { targetLoop }`
  after lowering.

### 5.6.3 `Exit` / `Exit(value)`

| | |
|---|---|
| **Introduced** | `Exit` Pascal (pre-1995) · `Exit(value)` 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Leaves the current routine immediately; the `Exit(value)` form also sets the
function result.

**Grammar**

```ebnf
(* parsed as CallStatement: "Exit" [ "(" Expression ")" ] *)
```

**Example**

```pascal
function Find(const S: string): Integer;
begin
  for var I := 0 to High(Data) do
    if Data[I] = S then Exit(I);   // sets Result := I and returns
  Result := -1;
end;
```

**Semantics & parsing notes**

- *`Exit(value)`* is only legal inside a **function** and assigns `Result` before
  returning; in a procedure the parenthesised form is an error: `Exit(1)`
  there is `E2025 Procedure cannot have a result type` - but an EMPTY `Exit()`
  compiles in a procedure, as a plain `Exit`. `Exit(1, 2)` is `E2029 ')'
  expected but ',' found`; a bare `Exit` in a function is fine. dcc64 37.0,
  2026-10-30 (spec audit g3, local/probe-park/spec-audit/g3 Flow.dpr).
- Intrinsic, not a keyword (see the 5.6 note).
- *AST:* `ExitStmt { value? }`.

### 5.6.4 `goto` and labels

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ⚠️ Legacy |

Unconditional jump to a declared label.

**Grammar**

```ebnf
LabelDeclSection = "label" LabelId { "," LabelId } ";" ;
GotoStmt         = "goto" LabelId ;
LabelId          = Ident | ?digit-sequence? ;   (* numeric labels are legal & historic *)
```

**Example**

```pascal
label Done;
begin
  if Error then goto Done;
  // ...
  Done:
    Cleanup;
end;
```

**Semantics & parsing notes**

- *`goto` and `label` ARE reserved words* (unlike Break/Continue/Exit).
- *Label declaration:* every target label must be declared in a `label` section of
  the same block; numeric labels (digit sequences) are permitted. A label number is read as written, hex included: `label $A; goto $A;` compiles (dcc64 37.0, probed 2026-10-01, x-f16 L04) while `label 010` and `goto 10` are different labels (E2003) - the label is its text, not its value. Any number of digits is accepted (`99999`); a negative or real number is E2029, two labels without a comma too (E2029), a label set twice E2073, a label declared and used but never set E2074.
- *Jump restrictions:* never **out of / into** a procedure or function. Into a
  structured statement dcc is more permissive than standard Pascal, and not
  uniformly (dcc64 37.0, probed 2026-09-26): a `goto` INTO a compound
  statement, a `while` or `for` body or a `case` branch compiles and runs (into
  a `for` body the counter is undefined - W1036 only); one into or out of a
  `try` statement, a `for-in` whose enumerator has a destructor, or a block
  that declares an inline variable is E2127; one into either branch of an `if`
  - with or without an `else`, the target in a compound or not - stops the
  compiler with an internal error (F2084 `IRBB366`). Enforce what dcc enforces
  in semantic analysis; a label on a statement in an `if` branch that a `goto`
  outside it targets is code dcc cannot compile.
- *A `goto` target is looked up as a label only:* inside `with R do goto Done`
  over a record with a field `Done`, the jump goes to the routine's label
  `Done` (dcc32 / dcc64 37.0, 2026-10-13; PasTree audit B44, B4-17), and
  with no label `Done` declared the same `goto Done` is `E2003 Undeclared
  identifier: 'Done'` - the field is never a candidate (dcc64 37.0,
  2026-10-31, PasTree audit E1-30, local/probe-park/specfu/E1-30 L1.dpr
  prints `at label 5`, L2.dpr). A
  labeled STATEMENT `Done:` in such a body is different - dcc reads the name
  through the with scope there and fails.
- *AST:* `LabelDecl`, `LabeledStmt { label, stmt }`, `GotoStmt { label }`.

### 5.6.5 `Halt`

| | |
|---|---|
| **Introduced** | Pascal/Turbo (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Terminates the **entire program** immediately — not just the current
routine or loop.

**Grammar**

```ebnf
(* parsed as CallStatement: "Halt" [ "(" Expression ")" ] *)
```

**Example**

```pascal
Writeln('working...');
if Fatal then Halt(3);   // process exits now, with exit code 3
Writeln('never reached');
```

**Semantics & parsing notes**

- Intrinsic (declared in `System`), not a reserved word — same shadowing rule as
  the 5.6 note above (Appendix B lists it alongside `Break`/`Continue`/`Exit` in
  the plain-flow intrinsic family).
- `Halt` optionally takes one `Integer` argument used as the process exit code
  (defaults to 0 when omitted).
- ⚠️ *`Halt` bypasses `finally` blocks but NOT unit finalization* - it does
  **not** unwind the call stack the way a raised exception or a normal return
  would, but it does shut the program down in order: every used unit's
  `finalization` section still runs. (dcc-verified, dcc32 37.0: `Halt(3)` called
  inside a `try…finally` prints the text before the call, but neither the
  `finally` block's `Writeln` nor any statement after the call executes; the
  spawned process's exit code was observed to be exactly `3`. dcc64 37.0,
  2026-10-30: the same program using a unit whose `finalization` writes a
  line prints that line after `before Halt`, and nothing from the `finally`
  part - spec audit g3, local/probe-park/spec-audit/g3 HaltFin.dpr,
  HaltFinU.pas.)
- *Distinguish from `Exit`:* `Exit` leaves only the current routine and still runs
  enclosing `finally` blocks on the way out; `Halt` leaves the whole program,
  runs none of the pending `finally` blocks, and runs the units'
  finalization.
- *AST:* may stay `Call("Halt")` until a lowering pass rewrites it to
  `HaltStmt { exitCode? }`.

---

## 5.7 The `with` statement

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — (officially discouraged by the style guide) |
| **Status** | ⚠️ Legacy |

Opens a scope in which the members of one or more records/objects are accessible
without qualification.

**Grammar**

```ebnf
WithStmt = "with" Designator { "," Designator } "do" Statement ;
```

**Example**

```pascal
with Customer do
begin
  Name := 'Acme';
  Balance := 0;     // resolves against Customer unless shadowed
end;
```

**Semantics & parsing notes**

- ⚠️ *Name-resolution hazard (the whole reason to track this carefully):* inside
  the body, an unqualified identifier is resolved **first** against the members of
  the `with` targets, **then** against the enclosing scope. With multiple targets
  `with A, B do`, resolution goes **right-to-left** (the last/innermost target
  wins). This silently shadows locals/fields and is the classic `with` bug.
- ⚠️ *A target member outranks EVERYTHING else in scope* — "first" above is
  absolute, not merely "before the unit scope". dcc-verified: a member wins
  over an enclosing class's own field, a routine local, a **parameter**, a
  unit-level global, an **inline `var` declared inside the with body itself**
  (`with GR do begin var Shared: Integer; Shared := 42; end` is an error when
  `GR.Shared` is a `string` — the member still won), and even the implicit
  **`Result`** and **`Self`**: inside `function F: Integer`, `with R do Result
  := 'x'` compiles when `R.Result` is a `string`, so the member — not the
  function's return slot — is what that name means.
  - *Implementation consequence, and the one that is easy to get half-right:* a
    member hit is not a fallback for names the resolver failed to bind, so an
    EARLIER binding must be **overridden**, not merely gap-filled. This applies
    to every target, not only to the hard ones — a resolver that opens the scope
    and then leaves already-resolved names alone (the natural shape, since most
    passes only fill in what is still unbound) silently keeps the wrong answer
    for exactly the collisions this rule exists to describe. The failure is not
    a missing `E2003`: the name binds to something of the WRONG TYPE, so it
    surfaces later and elsewhere as a bogus `E2010 Incompatible types`.
  - *The override covers INHERITED members, and that is where a two-tier
    resolver leaks:* a target whose type is declared in the current unit but
    descends from a class in another one (`TMy = class(TControl)`, or any class
    with no clause at all — the implicit `TObject`) is only HALF open to a
    same-unit pass. Its own members are in scope; the inherited ones are not, so
    `with C do Name := 'x'` with `C: TMy` and a local `Name: Integer` keeps the
    local — and a pass that revisits only the bodies whose target it could NOT
    type never comes back to this one. dcc-verified: the line compiles, `Name`
    is `TControl.Name`, in a plain body, a `with R, C do` mixed with a same-unit
    record, and inside an anonymous method declared in the body. A record has no
    inherited tier, but it is not closed either: a record helper declared in
    ANOTHER unit adds members, and they outrank a same-unit declaration of the
    same name - with `UHelp`'s `record helper for TRec` declaring `function
    Bump: string` and a unit-level `function Bump: Integer` beside the body,
    `with R do S := Bump` compiles and prints `helper` (the control `S :=
    Tag` is `E2010`). A pass that sees only the same unit binds Bump to the
    local function. dcc64 37.0, 2026-10-31 (PasTree audit E1-30,
    local/probe-park/specfu/E1-30 W8.dpr, `-DCTRL`; W4.dpr re-confirms the
    nested-declaration list below: a nested block's inline var, a `for var`
    counter, an anonymous method's parameter and its own `var` local, and an
    `on X: Exception do` variable all lose to the member, E2010).
  - *The member outranks a declaration NESTED in the body as well*, not only
    the body's own inline var: a nested block's inline var, an anonymous
    method's parameter or local, an `on E: T do` variable, a `for var`
    counter. `with P do begin var X: string := 'a'; X := 5; end` (P.X an
    Integer) compiles and writes P.X; inside a `for Tag := ...` loop, `with C
    do Tag := 7` writes C's inherited Tag, so it is no `E2081` - while `with
    C do K := 7` over a target without K still is. dcc32 / dcc64 37.0,
    2026-10-13 (PasTree audit B44: E3-06, B4-13).
  - *Not every name in the body is looked up there:* the dispatch name of a
    named argument (`Ov.Add(Source := X)`, 4.10.1) is the OLE call's, never a
    member or an enclosing name - `with L do Ov.Add(Nonexistent := 1)`
    compiles - and a `goto` target is a label (5.6.4). dcc32 / dcc64 37.0,
    2026-10-13 (PasTree audit B44: A4-02, B4-17).
  - *A name NO target has resolves as it would outside the `with`* - and in a
    method body that means Self's members, inherited ones included, before a
    unit-level declaration of the method's unit, a predefined name or a used
    unit's declaration (the chain of §3.3.2). dcc-verified, dcc64 37.0: in
    `TEd = class(TCtl)`
    (TCtl in another unit, with a property `Text`, a public field `Kind` and
    a protected `Insert(const S: string)`), the body `with FCanvas do begin
    S := Text; Insert('x'); Kind := 7 end` - FCanvas's class having none of
    the three, declared in TCtl's unit or in TEd's own - reads TCtl.Text
    (not the predefined file type `Text`), calls TCtl.Insert past a
    unit-level `procedure Insert(A: Integer; B: string)` of TEd's unit, and
    assigns TCtl.Kind past a unit-level `const Kind = '...'`. The trap is the
    two-tier one again: a resolver that handles with bodies apart from
    method bodies and only fills in names it could not bind keeps the outer
    binding (`with Canvas do ... Text` in a control's `Paint` is the VCL's
    own shape).
- ⚠️ *A target after the first is resolved INSIDE the targets before it.* This
  is the point of the multi-target form and not merely a shorthand for several
  independent targets: target *k* is looked up in the scope opened by targets
  1..*k*-1, then the enclosing scope. The RTL relies on it heavily —
  `with DIB, dsbm, dsbmih do` (`Vcl.Graphics`) works only because `dsbm` is a
  field of `DIB` and `dsbmih` a field of `dsbm`, and `with
  TDragDockObject(ADragObject), FDockRect do` (`Vcl.Controls`) only because
  `FDockRect` is a field of the cast. A resolver that binds every target in the
  ENCLOSING scope leaves the later ones undeclared, and with them every member
  reached through them in the body.
- ⚠️ *The override rule two bullets up applies to a later TARGET too, not only to
  body names.* Target *k* resolving inside targets 1..*k*-1 means a member hit
  there OUTRANKS whatever else the name denotes — dcc-verified: with a global
  `mid: string` in scope, `with O, mid do W := 7` compiles and means `O.mid`
  (`W` exists only on the field's type). The spelling that catches
  implementations is a target name that also names a **used unit**: `with
  ZStream, ZLIB do` (`Vcl.Imaging.pngimage`) against `System.ZLib`, where `ZLIB`
  is a field of `ZStream`. A bare unit name is not a legal target at all, so a
  resolver that binds it as one and then treats that target as already resolved
  never opens the second scope, and every member of it (`next_out`,
  `avail_out`) becomes a false `E2003`.
- *Parser/AST guidance:* **do not** flatten `with` during parsing — keep
  `WithStmt { targets: Designator[], body }` so a later name-resolution pass can
  rewrite each unqualified member access to an explicit `target.member`. Multiple
  targets desugar to nested single-target `with`s (right-to-left) — which is
  also the cleanest way to state the two rules above together: the nesting gives
  both the right-to-left body precedence and the "target sees the earlier
  targets" scoping, because an inner `with`'s target sits inside the outer
  `with`'s body.
- *Targets* must be record/object/interface-typed designators. Anything else is
  `E2018 Record, object or class type required` — and note the follow-on, because
  it is what a resolver will see first: dcc then reports every member in the body
  as `E2003 Undeclared identifier` as well, since the scope never opened.
  dcc-verified on `with V.rgrc do` where `rgrc` is an `array of TRect` — indexing
  it (`V.rgrc[0]`) is what makes the target legal.
- A designator here includes the forms that reach a structured value indirectly,
  all dcc-verified as with targets: a dereference (`P^`), an index (`Arr[I]`), a
  member chain (`P^.rgrc[0]`), an `as` cast, a *constructor call* — which yields
  its CLASS, not a result type, in both the paren-less (`TFoo.Create`) and
  argumented (`TFoo.Create(Self)`) spellings — and indexing a class through its
  **default array property**, including one it merely INHERITS (`TCollection.
  Items`, so `with Coll[I] do` is legal on any TCollection descendant).
  Two of those forms hide an extra step for a resolver, both from `Vcl`:
  - the dereference may be of a pointer type written INLINE on the variable's own
    declaration — `PExtLogPen: ^TExtLogPen` as a local, then `with Result,
    PExtLogPen^ do` (`Vcl.Graphics.GetPenData`). No pointer type *symbol* exists
    anywhere in that shape, so the pointee is reachable only from the
    declaration's type node — the same trap as an inline `array[...] of T`.
  - the cast may name a NESTED type through its outer one, in another unit:
    `with TScrollBarStyleHook.TScrollWindow(FMDIScrollSizeBox) do SizeBox := True`
    (`Vcl.Forms` over a nested class of `Vcl.StdCtrls`).
- *The full list of accepted target forms*, all dcc-verified on 37.0 — worth
  enumerating because a resolver needs a case per form and a missing one costs
  the whole body, not one name:

  | form | example | notes |
  |---|---|---|
  | variable / field / parameter | `with Rec.Field do` | |
  | property | `with Canvas do` | including an INHERITED one, and a bare redeclaration (`property Items;`), which has no type of its own — 13.1.4 |
  | parameterless function call | `with GetRecord do` | the RESULT type; if the name is OVERLOADED, the arity-0 overload — see below |
  | constructor call | `with TFoo.Create do` | the CLASS; both spellings |
  | typecast | `with TVarData(X) do` | the cast's TYPE, not the callee's |
  | `as` cast | `with Obj as TSub do` | |
  | dereference | `with P^ do` | the POINTEE, chasing alias chains |
  | index | `with Arr[I] do` | element type, or a default array property; a comma list peels ONE LEVEL PER INDEX — `with M[I, J] do` over `TMatrix = array of TRow; TRow = array of TRec` opens TRec, not TRow (dcc-verified; a resolver that peels once per bracket pair opens the row and every member is `E2003`) |
  | `inherited Name` | `with inherited Canvas do` | 12.1.2 — the member is looked up from the ANCESTOR of the enclosing method's class, never from the class itself (`Vcl.ExtCtrls`) |
  | class reference | `with C do` where `C: class of TBase` | exposes TBase's class vars and class methods (15.2.1) |
  | bare class TYPE NAME | `with TBase do Tick` | same reach as the class reference — legal, and easy to miss because the target resolves to a TYPE rather than a value |
  | interface-typed designator | `with I do Go` | |
  | parenthesised designator | `with (R) do` | see the caveat below |
  | function call WITH arguments | `with GetRec(I) do` | the RESULT type of the overload the ARGUMENTS select (6.3.1) — not of the nearest declaration of the name; a constructor with arguments is still the CLASS |
  | generic METHOD, type arguments written | `with TGH.CreateControlWith<TImage>(Owner, Parent) do WrapMode := ...` | the result type with the written arguments substituted — `T` is TImage here (16.5.1); when they are not written they are inferred from the arguments the same way (an FMX grid library, 4 sites) |
  | generic CLASS instantiation as the qualifier | `with TFmxObjectFactory<TLayout>.CreateWithParent(P) do StyleName := ...` | a static class function's `T` result closed over the instantiation — TLayout (same library, 20 sites in one style builder) |
  | member chain through an OUTER target | `with Col do with Footers.Add do` | `Footers` is a member of the outer target, resolved by the outer with scope; the inner target types from there — including when `Footers` also names a used unit (see the qualifier rule below) |

- ⚠️ *A call target is subject to OVERLOAD RESOLUTION, against an empty argument
  list.* The syntax provides no arguments, so the target selects the arity-0
  overload (6.3.1) — and it may be the one the class does NOT declare. A UI
  automation library writes exactly that: the class overrides `function
  GetScreenBounds(out ABounds: TRect): Boolean` and inherits a parameterless
  `function GetScreenBounds: TRect` from two classes up, then says
  `with GetScreenBounds do X := (Left + Right) div 2`. A resolver that types the
  target by finding "the member of that name" — the natural implementation,
  since the *nearest* member is what every other reference wants — types the
  target as `Boolean` and reports every name in the body undeclared. Overload
  selection cannot be skipped here just because there is no argument list to
  parse; the empty list is what selects.
- ⚠️ *Two negatives, both easy to implement by accident:*
  - *The implicit pointer dereference does NOT extend to a with target.* Object
    Pascal lets `P.Field` stand for `P^.Field`, and a resolver that reuses its
    member-access walk for the target inherits that hop — but dcc rejects
    `with P do` outright with `E2018`, followed by `E2003` on every member in
    the body. The `^` is mandatory here. Being more permissive costs no false
    positive, only a missed diagnostic, which is why it survives unnoticed.
  - *Parentheses demote the target to a VALUE.* `with (R) do X := 1` opens the
    scope (no `E2018`) but reports `E2064 Left side cannot be assigned to` — the
    members are readable and not assignable. A resolver that treats `(R)` as
    transparent will silently accept writes dcc refuses.
- ⚠️ *The target's type is very often declared in ANOTHER unit* — in practice
  that is the common case, not the exception (`with LTZ.StandardDate do`,
  where the field's type comes from `Winapi.Windows` —
  `System.DateUtils.pas`). An implementation that opens the with scope by
  *joining* the target type's member scope therefore only covers same-unit
  targets, since a scope reference is meaningful only inside its own unit's
  model; cross-unit members have to be bound the same way any other
  cross-unit reference is. Getting this wrong makes every member of such a
  body read as an undeclared identifier (464 of them across the D13 RTL).
- ⚠️ *A target designator is evaluated in the ENCLOSING scope, not its own
  with scope* — the scope covers the body only. `with A.B do` resolves `A`
  and `B` outside, so an identifier inside a target expression must not be
  looked up among the target's own members.
- *Full precedence order for an unqualified name in the body:* innermost
  `with` before outer ones; within one `with`, targets right-to-left; then —
  because the with scope is opened *inside* the enclosing body — the
  enclosing method's own and inherited members; then used units; then the
  implicit `System`/`SysInit` units.
- ⚠️ *The override rule holds in QUALIFIER position too, against a used UNIT's
  name.* An undotted `uses` entry is a name in the outermost scope (1.2.3),
  and a with member outranks it exactly as it outranks a global. In a unit
  that uses `Lib.Footers`, `with Col do with Footers.Add do AggregateFunction
  := ...` compiles and means `Col.Footers.Add` (dcc64 37.0, probed 2026-09-19
  with a three-unit fixture; the real shape is a grid library's column dialog
  over its `...DataGrid.Footers` unit) - there not even by outranking: a
  dotted entry's last segment is no name at all (1.2.3, probed 2026-10-01).
  A resolver that decides "`Footers.X` is unit-qualified" by TEXT - matching
  the used units' last segments before it has looked at the with scope -
  never types the inner target, and the body is `E2003` again. The same text match is
  right for `System.Footers.X`, where the unit is a LONGER prefix and no
  member is asked about: only the bare head of the chain is subject to the
  with scope.
- ⚠️ *Implementation guidance, from the shapes above having been fixed one at
  a time:* the target's type is **the expression typer's answer**, and a
  resolver must not keep a SECOND typer for with targets. Every hand-kept
  target dispatch fell behind the expression typer by exactly the forms the
  next corpus used - the cast to a nested type, the class-reference
  constructor, `inherited Name`, the overload selected by arguments, the
  generic method with written arguments, the generic instantiation as a
  qualifier - and each miss cost a whole body of false `E2003`, never one
  name. What the with position ADDS to the expression typer is small and
  stated above: a bare type name or class reference opens the CLASS scope, a
  bare overloaded name selects the arity-0 overload, parentheses demote the
  target to a value, the implicit pointer dereference does not apply. Those
  are rules about the position; the typing itself is the ordinary one.
- The target forms are the table above (each was a real false-E2003 source in
  the D13 RTL: the typecast `with TVarData(ParamValues[I]) do` in
  `System.ObjAuto.pas` - a cast and a call look alike, decide by what the name
  resolves to - and the dereference `with LVarData^ do` through `PAlias =
  PVarData = ^TVarData` in `System.Variants.pas`). The forms compose: `with
  FindVarData(V)^ do` is call, pointer result, dereference
  (`System.Variants.pas`).
- ⚠️ *The with scope carries the target type's HELPER members too.* dcc-verified:
  with `TRecHelper = record helper for TRec`, `with R do Bump` calls the
  HELPER's method unqualified, and 15.3.3's rules apply unchanged (a helper
  member hides the type's own of the same name). So the scope a `with` opens is
  not "the type's member scope" but "whatever a member lookup on that type
  would find" — helpers, inherited members and implicit `TObject` included.
- ⚠️ *The with scope reaches INTO an anonymous method declared in the body.*
  dcc-verified: `with R do P := procedure begin Writeln(X) end` binds `X` to
  `R.X` and captures it. An implementation that splices the with scope in by
  reparenting must therefore not stop at the first nested scope boundary — the
  closure's own scope needs its parent link rerouted as well.
- *A member's type must be closed over the target's instantiation frame.* When
  the target is a generic instance, the member's DECLARED type is written in the
  open parameters: `FThreads: TThreadList<TWorker>` makes `LockList`'s declared
  `TList<T>` mean `TList<TWorker>`, so `with FThreads.LockList do` must
  substitute before looking anything up inside it. Skipping this finds the right
  member NAMES with wrong element types.
- *AST:* `WithStmt { targets: Designator[], body }`.
