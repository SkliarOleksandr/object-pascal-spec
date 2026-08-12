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
  argument* (ch.06 §6.2.2) — a property with a `write` specifier is a valid `:=`
  LHS (it lowers to a setter call) but is **rejected** when passed to a `var`
  parameter, because there is no addressable storage to hand the callee.
  (dcc-verified, dcc32 37.0: `Foo.Val := 10` compiles, but passing that same
  `Foo.Val` to `procedure P(var X: Integer)` is `E2197 Constant object cannot be
  passed as var parameter`.) Do not reuse one "assignable designator" check for
  both contexts.
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
- *`@` operator:* `@Foo` / `Foo` distinctions interact with `{$M}`/typed-`@`
  modes — note for ch.04.
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

- *Scope:* a compound statement does **not** introduce a new declaration scope by
  itself — but **inline `var`/`const` declarations** (10.3+) inside it are scoped
  to the enclosing routine block from their point of declaration (see 5.5.1).
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
- ⚠️ *No `;` before `else`:* a semicolon after the `then`-branch terminates the
  `if` statement, leaving `else` orphaned → syntax error. The parser must treat a
  `;` immediately before `else` as an error (common mistake worth a clear
  diagnostic).
- *Type rule:* the condition must be `Boolean` (Object Pascal does not coerce
  integers to Boolean here).
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
- *Labels:* each `CaseLabel` must be a **compile-time constant** (or constant
  range `a..b`); ranges and label sets must not overlap and must fit the selector
  type. Overlap is a compile error.
- *`else` vs `otherwise`:* the standard keyword is `else`; some dialects accept
  `otherwise` — Delphi uses `else`.
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
- *Type rule:* both branch expressions must be assignment-compatible to a common
  result type; that common type is the type of the whole expression.
- *Evaluation:* short-circuit — only the selected branch is evaluated (contrast
  with the `IfThen` RTL functions, which evaluate both arguments).
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
  is usually **inferred** from the start expression; an explicit `: TypeRef` is
  allowed.
- *Bounds evaluated once:* both bounds are computed **before** the first
  iteration and not re-evaluated — model this in lowering (don't re-emit the
  limit expression per iteration).
- ⚠️ *Counter is read-only in the body:* assigning to the loop variable inside the
  body is a compile error; its value **after** normal completion is undefined.
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
  a string, static/dynamic array, or set — **or** any type satisfying the
  **enumerator pattern**: it (or a helper on it) exposes `function GetEnumerator`
  returning a type with `function MoveNext: Boolean` and a `property Current`.
  The loop element type = the type of `Current`. The parser emits a `ForInStmt`;
  **semantic analysis** resolves `GetEnumerator` (including via class/record
  helpers) and the element type.
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
> must **not** tokenise them as keywords.

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
  returning; in a procedure the parenthesised form is an error.
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
  the same block; numeric labels (digit sequences) are permitted.
- *Jump restrictions:* cannot jump **into** a structured statement from outside it,
  nor **out of / into** a procedure or function. Enforce in semantic analysis.
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
- ⚠️ *`Halt` bypasses `finally` blocks and unit finalization* — it ends the
  process directly and does **not** unwind the call stack the way a raised
  exception or a normal return would. (dcc-verified, dcc32 37.0: `Halt(3)` called
  inside a `try…finally` prints the text before the call, but neither the
  `finally` block's `Writeln` nor any statement after the call executes; the
  spawned process's exit code was observed to be exactly `3`.)
- *Distinguish from `Exit`:* `Exit` leaves only the current routine and still runs
  enclosing `finally` blocks on the way out; `Halt` leaves the whole program and
  runs none of them.
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
  | index | `with Arr[I] do` | element type, or a default array property |
  | `inherited Name` | `with inherited Canvas do` | 12.1.2 — the member is looked up from the ANCESTOR of the enclosing method's class, never from the class itself (`Vcl.ExtCtrls`) |
  | class reference | `with C do` where `C: class of TBase` | exposes TBase's class vars and class methods (15.2.1) |
  | bare class TYPE NAME | `with TBase do Tick` | same reach as the class reference — legal, and easy to miss because the target resolves to a TYPE rather than a value |
  | interface-typed designator | `with I do Go` | |
  | parenthesised designator | `with (R) do` | see the caveat below |

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
- A target may be any designator, not just a variable — and a resolver must
  type every form, or the whole body reads as undeclared (each was a real
  false-E2003 source in the D13 RTL):
  - a field: `with Rec.Field do`;
  - a parameterless function call: `with GetRecord do` — the routine's
    RESULT type;
  - a **typecast**: `with TVarData(ParamValues[I]) do VType := ...`
    (`System.ObjAuto.pas`) — the cast's TYPE itself, not the callee's
    declared type (syntactically identical to a call; disambiguate by what
    the name resolves to);
  - a **pointer dereference**: `with LVarData^ do` (`System.Variants.pas`) —
    the POINTEE type, chasing pointer-type aliases (`PAlias = PVarData =
    ^TVarData`);
  - and compositions: `with FindVarData(V)^ do` (call → pointer result →
    dereference), also `System.Variants.pas`.
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
