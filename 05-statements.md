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

> ⚠️ **Lexical fact for all of 5.6.1–5.6.3:** `Break`, `Continue`, and `Exit` are
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
- *Parser/AST guidance:* **do not** flatten `with` during parsing — keep
  `WithStmt { targets: Designator[], body }` so a later name-resolution pass can
  rewrite each unqualified member access to an explicit `target.member`. Multiple
  targets desugar to nested single-target `with`s (right-to-left).
- *Targets* must be record/object/interface-typed designators.
- *AST:* `WithStmt { targets: Designator[], body }`.
