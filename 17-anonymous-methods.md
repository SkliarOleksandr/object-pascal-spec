# 17 — Anonymous Methods & Closures

Inline routine literals that capture their surrounding variables, and the
`reference to` type family that holds them.

Procedural types recap → [ch.06 §6.6](06-routines.md#66-procedural-types).

## Chapter grammar umbrella

```ebnf
AnonMethodType = "reference" "to" ( "procedure" | "function" )
                 [ "(" FormalParams ")" ] [ ":" ResultType ]
                 { CallDirective } ;                      (* 6.5.1, 6.6.1 *)
AnonMethodExpr = ( "procedure" | "function" )
                 [ "(" FormalParams ")" ] [ ":" ResultType ]
                 { CallDirective }                        (* but "overload" *)
                 Block ;                                  (* a literal, used as an expression *)
```

---

## 17.1 `reference to` types

### 17.1.1 Anonymous-method reference types

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A type that can hold an anonymous method, a standalone routine, or a method —
implemented as an auto-generated interface (hence reference-counted).

**Example**

```pascal
type
  TProc1 = reference to procedure(X: Integer);
  TFunc1 = reference to function(const S: string): Integer;
```

**Semantics & parsing notes**

- ⚠️ Distinct from `of object` method pointers (ch.06): a `reference to` value is an
  **interface-backed closure** that can **capture local state**, is
  reference-counted, and is *not* a simple `(code, Self)` pair. Keep the kind on
  the type node — assignment compatibility differs.
- ⚠️ *Compiler-distinguished interface, not a structurally-equal one* (dcc-verified,
  dcc32 37.0): although a `reference to` value is implemented as a single-method
  interface with a hidden `Invoke` method, it is **not** assignment-compatible
  with a hand-declared single-method interface of the identical signature, even
  when that interface's method is itself named `Invoke`. Both directions fail:
  ```pascal
  type
    IMyCallback = interface function Invoke: Integer; end;
    TFuncInt    = reference to function: Integer;
  var
    Cb: IMyCallback; Fn: TFuncInt;
  begin
    Fn := function: Integer begin Result := 42; end;
    Cb := Fn;   // E2010 Incompatible types: 'TFuncInt' and 'IMyCallback'
  ```
  (and, symmetrically, assigning an `IMyCallback` instance to a `TFuncInt`
  variable gives `E2010 Incompatible types: 'TFuncInt' and 'IMyCallback'` as
  well). A resolver must track `reference to` types as their own nominal kind —
  never unify them with a structurally-matching user interface just because both
  lower to "interface with one method called Invoke".
  - *...yet a CLASS may list it as an interface.* `TMy = class(TInterfacedObject,
    TProc)` with a `procedure Invoke;` compiles, an instance assigns to a `TProc`
    variable, and `P()` (or bare `P;`) runs `TMy.Invoke`. Assign through a
    typed variable: `P := TMy.Create;` is `E2010 Incompatible types:
    'System.SysUtils.TProc' and 'TObject.Create'` - against a procedural
    target, `TMy.Create` is read as the constructor itself (a method
    reference), not called. dcc64 37.0, 2026-10-30 (spec audit g5,
    local/probe-park/spec-audit/g5 A03.dpr).
- ⚠️ *What else converts to a `reference to` type, and what does not* (dcc64
  37.0, 2026-10-30, spec audit g5, local/probe-park/spec-audit/g5 A02.dpr):
  with `R: TProc<Integer>`, a global procedure `R := Glob`, an instance method
  `R := C.M`, a class method `R := TC.CM`, a static class method `R :=
  TC.SM`, and an `of object` variable `R := MP` all compile (signatures
  matching). The reverse, `MP := R`, is `E2010 Incompatible types: 'TMeth'
  and 'TProc<Integer>'`. A routine NESTED in the current one is refused with
  the capture error, `R := Nested` - `E2555 Cannot capture symbol 'Nested'`
  (17.3.1).
- *AST:* `AnonMethodType { kind: proc|func, params[], resultType?,
  directives[] }` - `reference to procedure stdcall` keeps its convention,
  and only written into the type: no run after a `;` (6.6.1).

---

## 17.2 Anonymous method literals

### 17.2.1 Inline `procedure`/`function` literals

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `procedure`/`function` with a body but **no name**, used as an expression value.

**Example**

```pascal
var P: TProc1;
P := procedure(X: Integer)
     begin
       Writeln(X);
     end;
P(42);
```

**Semantics & parsing notes**

- ⚠️ *Headed expression with embedded `Block`:* the parser must, in **expression
  position**, recognise `procedure`/`function` followed by an optional param list,
  optional result, and a full `Block` (`begin..end`) — producing an
  **expression**, not a declaration. This dual role of the `procedure`/`function`
  keywords is a key parsing branch (declaration vs. literal) decided by context.
- ⚠️ *Closing parenthesis after `end`:* when an anonymous method is passed as a
  call argument, the call's closing `)` (and any following arguments) come
  **after** the literal's `end`. The parser must keep the argument-list nesting
  open across the whole embedded `Block` — a classic source of confusing syntax
  errors when hand-written parsers close the call too early.
- *AST:* `AnonMethod { kind, params[], resultType?, directives[], body,
  captured[] }`.
- ⚠️ *Directives may stand before the body* - `function(ErrorCode: HResult;
  ...): HResult stdcall begin` (FMX.WebBrowser.Win.pas) - and the literal's
  type follows them (dcc64 37.0, probed 2026-09-27): the conventions, `far`,
  `near`, `export`, `assembler` and `varargs`; `overload` there crashes dcc
  (`F2084 Internal Error: E17608`), a hint is `E1030`, a `;` before them
  `E2029`. A `stdcall` literal assigned to a plain `reference to function`
  is `E2010 Incompatible types`, and a plain literal to a `reference to ...
  stdcall` alike. Written twice, the last convention wins (`procedure cdecl
  register begin end` is a register literal).
- ⚠️ *A literal converts to a `reference to` type ONLY* (dcc-verified, dcc32
  37.0). Neither an `of object` method pointer nor a plain procedural type
  accepts one, even with an identical signature, and a parameter is no
  different from a variable:
  ```pascal
  uses System.Classes;          // TNotifyEvent = procedure(Sender: TObject) of object
  type TPlain = procedure(A: Integer);
  var E: TNotifyEvent; P: TPlain;
  procedure Take(AEvent: TNotifyEvent); begin end;
  begin
    E := procedure(Sender: TObject) begin end;  // E2010 Incompatible types: 'TNotifyEvent' and 'Procedure'
    P := procedure(A: Integer) begin end;       // E2010 Incompatible types: 'TPlain' and 'Procedure'
    Take(procedure(Sender: TObject) begin end); // E2010 Incompatible types: 'TNotifyEvent' and 'Procedure'
  ```
  The target's signature must match the literal's: parameter count and
  procedure-vs-function, a generic reference type (`TProc<Integer>`) with its
  type arguments applied.
  - *Overload resolution follows from it (6.3.1):* a candidate whose parameter
    the literal cannot convert to is not a candidate at all, whichever order
    the overloads are declared in. With `Make(AEvent: TNotifyEvent = nil)` and
    `Make(AProc: TProc)`, `Make(procedure begin end)` calls the `TProc` one;
    with `Take(AProc: TProc)` and `Take(AProc: TProc<Integer>)`,
    `Take(procedure(X: Integer) begin end)` calls `TProc<Integer>` and
    `Take(procedure begin end)` calls `TProc`. A resolver that scores the
    literal as merely "some procedural value" ties every such pair and picks
    by declaration order.
  - *Parentheses change nothing:* `Make((procedure begin end))` — the usual
    way a multi-line literal argument gets written — resolves exactly like the
    bare literal. A resolver testing the ARGUMENT node for the literal shape
    must look through the parenthesis node first.
- ⚠️ *A literal as an inline `var`'s initializer types the variable by the
  literal's own signature* - an anonymous procedural type no declaration
  names. Inside a routine, `var F := function: TA begin ... end;` makes `F()`
  and the paren-less `F` calls of a parameterless function: `F().OnlyA` and
  `F.OnlyA` read TA's member, `F().Nope` and `F.Nope` are `E2003 Undeclared
  identifier: 'Nope'`. A literal invoked where it is written is no call:
  `X := (function: TA begin ... end)().B;` is `E2066 Missing operator or
  semicolon`. dcc64 37.0, 2026-10-30 (PasTree audit E3-39,
  local/probe-park/e339 AL.dpr, local/probe-m-d P3.dpr).
- ⚠️ *A function returning a `reference to` value needs a double call to invoke
  the closure* (dcc-verified, dcc32 37.0). Given
  `function GetShowMethod: TIntProc;` (`TIntProc = reference to procedure(N: Integer)`):
  - `GetShowMethod;` (bare, as a statement) compiles and **only runs
    `GetShowMethod`'s own body**; the closure it returns is evaluated and
    immediately discarded — never invoked.
  - `GetShowMethod(3);` fails with **E2034 "Too many actual parameters"** —
    `GetShowMethod` itself takes zero parameters, so `3` cannot bind to it.
  - `IP := GetShowMethod;` (assigning the bare call to an `IP: TIntProc`
    variable) fails with **E2010 "Incompatible types: 'TIntProc' and
    'Procedure'"** — dropping the parens makes the compiler try to treat
    `GetShowMethod` itself (a routine of type "function: TIntProc") as the
    value being assigned to `IP`, rather than calling it and assigning the
    *result*; that routine type is not `TIntProc`, hence the mismatch. The
    empty-parens call `IP := GetShowMethod();` is required to force
    call-then-assign semantics.
  - `GetShowMethod()(3);` — first `()` calls `GetShowMethod` and produces the
    closure value, the second `()` invokes that closure with argument `3`.
    This chained-call form compiles and runs correctly; it is the idiomatic
    one-liner alternative to a temporary variable.
  - A resolver must therefore special-case "call expression whose *result
    type* is itself invokable, followed immediately by another argument
    list" as a **second, independent call** on the intermediate result — not
    reinterpret it as one call with the wrong arity.
  - ⚠️ By contrast, a *plain variable* of a parameterless `reference to
    procedure` type (not a function call) **does** invoke correctly when
    referenced bare in statement position — `P;` calls the closure exactly
    like `P();` in dcc32 37.0. Older guidance describing a bare parameterless
    `reference to` *variable* as requiring explicit `()` to avoid being
    mistaken for "getting its address" was not reproduced here; the
    ambiguity is real only for the bare-call-of-a-function-with-a-procedural-
    result-type case documented above.

---

## 17.3 Variable capture (closures)

### 17.3.1 Capturing enclosing variables

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

An anonymous method captures the **variables** (not just values) of its enclosing
scope; their lifetime is extended to match the closure.

**Example**

```pascal
function MakeCounter: TFunc<Integer>;
var N: Integer;
begin
  N := 0;
  Result := function: Integer
            begin
              Inc(N);          // captures N by reference
              Result := N;
            end;
end;
```

**Semantics & parsing notes**

- ⚠️ *Capture is by reference to the variable*, not by value-copy: all closures
  sharing a captured variable see each other's mutations. The compiler hoists
  captured locals into a heap frame whose lifetime is tied to the closure's
  ref count.
- ⚠️ *Loop-variable capture gotcha:* capturing a `for`-loop counter captures the
  single shared variable — a frequent bug. A semantic-analysis warning candidate.
  dcc gives no warning: three literals `function: Integer begin Result := I;
  end` stored in `for I := 0 to 2` print `3 3 3` (dcc64 37.0, 2026-10-30,
  spec audit g5, local/probe-park/spec-audit/g5 A04.dpr).
- ⚠️ *What cannot be captured - `E2555 Cannot capture symbol 'X'`* (dcc64
  37.0, 2026-10-30, spec audit g5, local/probe-park/spec-audit/g5
  A01p1.dpr-A01p10.dpr): every parameter passed by reference - `var` (also a
  `var S: string`, and when only its address `@V` is taken), `out`, an
  untyped `const U`, `const [ref] C`, and an open array `const A: array of
  Integer` - and a routine nested in the enclosing one (17.1.1). Value
  parameters and plain `const` parameters (`const C: Integer`, `const S:
  string`) are captured like locals, and a value parameter may be assigned in
  the literal. A NESTED routine (not a literal) uses the same `var`
  parameter freely; the restriction is the literal's, because its frame may
  outlive the call.
- ⚠️ *Own scope, own `Result`:* an anonymous method body owns its locals and its
  OWN implicit `Result` (typed by the literal's result type). Two sibling
  literals may declare the same local name (dcc-verified:
  System.JSON.Serializers has two adjacent literals each with `var LSer`), and
  `Result :=` inside a `function: Boolean` literal binds to the LITERAL's
  Boolean result even when the enclosing function returns `string`
  (System.SysUtils TraverseDirectory callbacks). The enclosing routine's
  `Result` is NOT capturable (dcc E2555 "Cannot capture symbol 'Result'") — a
  resolver that lets the enclosing `Result` leak into the literal's body
  produces false E2010.
  Only the enclosing FUNCTION's implicit Result is hidden that way: inside a
  procedure, `Result` in a literal is whatever the name means there - a field,
  a property (`property Result: TList` of the method's class), a local - and
  the literal captures it like any other name (`P := procedure begin
  Result.Clear; end` compiles, `Result.Bogus` is `E2003`). dcc32 / dcc64
  37.0, 2026-10-13 (PasTree audit B43, B4-16).
  A function literal's own `Result` follows the routine rule of 03 §3.1.3: a
  parameter named `Result` is `E2004`, reported at the literal's `begin` -
  `F := function(Result: Integer): Integer begin end` fails. A procedure
  literal has no `Result` to collide with (`procedure(Result: Integer)`
  compiles), and neither does a procedural type's parameter list
  (`reference to function(Result: Integer): Integer`). dcc64 37.0,
  2026-10-25 (PasTree audit D2-04).
- The resolver must compute the **capture set** (which enclosing locals/`Self` are
  referenced) to model lifetime; record it on the AST node.
- *AST:* `captured[]` listing captured symbols on the `AnonMethod`.
