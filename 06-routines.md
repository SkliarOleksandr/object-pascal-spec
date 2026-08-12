# 06 — Procedures & Functions

Routine declarations, the full parameter-passing model (the part with the most
parser-relevant detail), overloading, inlining, calling conventions, procedural
types, and external/`noreturn` declarations.

Shared productions (`Block`, `TypeRef`, `Ident`, `ConstExpr`) → [Appendix
B](B-lexical-grammar.md). Anonymous methods → [ch.17](17-anonymous-methods.md).

## Chapter grammar umbrella

```ebnf
ProcedureDeclSection = ProcHeading ";" [ Directives ]
                       ( Block ";" | AsmBlock ";" | ForwardOrExternal ) ;
ProcHeading = ( "procedure" | "function" ) [ ImplName ]
              [ "(" [ FormalParams ] ")" ]
              [ ":" ResultType ] ;             (* ResultType only for function *)
ImplName    = Ident [ GenericArgs ] { "." Ident [ GenericArgs ] } ;
              (* implementation headers of generic types carry the type params
                 per segment: procedure TList<T>.Add / TDict<K,V>.TEnum.MoveNext *)
FormalParams = FormalParam { ";" FormalParam } ;
FormalParam  = [ AttributeGroup ] [ ParamModifier ]
               AttrName { "," AttrName }
               [ ":" ParamType ] [ "=" ConstExpr ] ;
AttrName     = [ AttributeGroup ] Ident ;
               (* [Ref] may precede or follow the modifier, and may precede
                  EACH name in the list: "const [REF] CLSID, [REF] IID: TGUID"
                  (Datasnap.DSIntf.pas) — see §6.2.3 / ch.19 §19.3.3 *)
ParamModifier = "var" | "const" | "out" ;
```

---

## 6.1 Routine declaration

### 6.1.1 Procedures & functions

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`procedure` (no result) and `function` (returns a value via the implicit
`Result` variable or the routine name).

**Example**

```pascal
procedure Log(const Msg: string);
begin Writeln(Msg); end;

function Add(A, B: Integer): Integer;
begin Result := A + B; end;
```

**Semantics & parsing notes**

- ⚠️ *`Result` is an implicit local* in every function (since Delphi 2). Assigning
  the **function name** (`Add := …`) is the classic-Pascal equivalent and still
  legal — the resolver must treat the routine name as an alias of `Result` inside
  the body. `Exit(value)` also sets it (ch.05 §5.6.3).
- ⚠️ *Assigning `Result` does NOT terminate the function* — unlike C's `return`,
  it is an ordinary assignment to an ordinary local variable, and execution falls
  through to whatever statement follows. Only `Exit`/`Exit(value)` (ch.05 §5.6.3)
  or falling off the end of the block actually returns. (dcc-verified, dcc32
  37.0: a function that does `Result := 1;` then more statements then
  `Result := 2;`, with no `Exit` in between, prints from both statements after
  each assignment and returns `2` — the last write, not the first — to its
  caller.)
- A function whose `Result` is never assigned returns an undefined value (managed
  types excepted) — a hint/warning, not a parse error.
- *AST:* `Routine { kind, name, params[], resultType?, directives[], body? }`.

### 6.1.2 Forward declarations

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`forward` declares a routine's signature now and defines its body later in the
same unit — enabling mutual recursion.

**Grammar**

```ebnf
ForwardOrExternal = "forward" ";" | ExternalDecl ;
```

**Semantics & parsing notes**

- The later definition may omit the parameter list (it is taken from the forward
  declaration). `interface`-section routine headers are implicitly forward (ch.01).
- ⚠️ *Applies equally to a QUALIFIED method implementation* — `procedure
  TFoo.Bar;` may drop the parameter list of a class-declared `procedure
  Bar(AParam: Integer);` exactly like a plain forward-declared global routine
  (dcc-verified: `Vcl.CheckLst.pas`'s `TCustomCheckListBox.ToggleClickCheck`
  is declared with `(Index: Integer)` and implemented as bare
  `ToggleClickCheck;`, using `Index` freely in the body). The resolver must
  pull the parameter NAMES (not just count) from the matched declaration into
  the implementation's own body scope in this case — omitting them is not
  the same as the routine having zero parameters. See ch.11 §11.1.3 for the
  method-pairing rule this interacts with.
- *AST:* link the forward header to its later body; one logical routine.

---

## 6.2 Parameters

> The parameter modifier determines **how the argument is bound** (by value, by
> reference, read-only) — central to both semantics and codegen. The parser
> records the modifier; the type-checker enforces lvalue/const rules.

> ⚠️ **The order in which actual-parameter expressions are evaluated is
> unspecified by the language** — do not encode any particular order as a rule,
> and do not let a resolver/optimizer assume left-to-right evaluation is safe to
> rely on for side effects. (dcc-verified, dcc32 37.0 — implementation-defined,
> NOT a language guarantee: `F(SideEffect(1), SideEffect(2), SideEffect(3))`
> printed `eval 3`, `eval 2`, `eval 1` — i.e. the CURRENT compiler evaluates
> actual arguments **right-to-left** — before the call itself ran with the
> arguments in their written left-to-right positions (`1 2 3`). Treat this as
> one observed data point about today's dcc32, not a rule a conforming parser
> or resolver may depend on; a future compiler version or optimization level is
> free to evaluate in a different order.)

### 6.2.1 Value parameters

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

No modifier — the argument is **copied**; modifications inside don't affect the
caller.

```pascal
procedure P(X: Integer);   // X is a local copy
```

**Semantics & parsing notes**

- For managed types (strings, dynamic arrays, interfaces), the copy is reference-
  counted, not a deep copy.

### 6.2.2 `var` (reference) parameters

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Pass by reference; the callee reads and writes the caller's variable.

```pascal
procedure Swap(var A, B: Integer);
```

**Semantics & parsing notes**

- ⚠️ The argument **must be an assignable lvalue** of the *exact* type (no implicit
  conversion) — enforce in the type-checker.
- ⚠️ *"Assignable lvalue" here is narrower than an assignment-statement LHS*
  (ch.05 §5.1.1): a genuine addressable variable/field/array-element/dereference
  qualifies, but a literal constant, an arbitrary expression, a function-call
  result, and — critically — a **property**, even one with a `write` specifier,
  do **not**. A property lowers to a setter *call*, which has no address to hand
  the callee, so it cannot bind to `var`/`out` even though `Foo.Val := X` is a
  perfectly legal assignment. (dcc-verified, dcc32 37.0, all four rejected
  identically against `procedure P(var X: Integer)`: `P(5)`, `P(A + B)`,
  `P(GetVal)` where `GetVal` is a function, and `P(Foo.Val)` where `Val` is a
  read/write property all report `E2197 Constant object cannot be passed as var
  parameter`.) A resolver must not reuse the assignment-LHS check for
  `var`/`out` argument binding.

### 6.2.3 `const` parameters (and `const [Ref]`)

| | |
|---|---|
| **Introduced** | `const` Delphi 1; `[Ref]` attribute ~XE2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`const` = read-only parameter; the compiler may pass by reference for efficiency.
The `[Ref]` attribute forces by-reference passing.

```pascal
function Hash(const Data: TBytes): Cardinal;
procedure Draw(const [Ref] R: TRect);      // guaranteed by-reference
```

**Semantics & parsing notes**

- Assigning to a `const` parameter inside the body is a **compile error**.
- `const [Ref]` guarantees reference passing (relevant for large records /
  interop). `[Ref]` is a compiler-recognized **parameter attribute**
  (ch.19 §19.3.3) and may be written **before or after** `const`
  (`[Ref] const X` / `const [Ref] X`). The RTL uses it heavily in COM/DirectX
  headers (`const [Ref] ppResources: ID3D11Resource`).
- ⚠️ *Not Delphi:* FreePascal's `constref` keyword is the FPC equivalent of
  `const [Ref]` — it is **not accepted by the Delphi compiler** (the D13 sources
  contain it only inside `{$IFDEF FPC…}` branches, e.g. `System.Skia.pas`). A
  Delphi parser must reject it.

### 6.2.4 `out` parameters

| | |
|---|---|
| **Introduced** | Delphi 3 (COM-era) |
| **Deprecated** | — |
| **Status** | ✅ Current |

By-reference output parameter; the incoming value is not used (managed types are
cleared on entry).

```pascal
procedure GetSize(out W, H: Integer);
```

**Semantics & parsing notes**

- `out` is a **directive** (B.4.2), keyword only in this position. Like `var`, the
  argument must be an lvalue.

### 6.2.5 Default (optional) parameters

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A trailing value parameter may specify a default constant, making it optional.

```pascal
procedure Show(Msg: string; Modal: Boolean = True);
```

**Semantics & parsing notes**

- ⚠️ *Trailing-only rule:* once one parameter has a default, **all following**
  parameters must too. Defaults must be **constant expressions** and are only
  allowed on value/`const` parameters (not `var`/`out`). Enforce both.
- Interacts with overloading (6.3) — ambiguous calls are a semantic error.
- ⚠️ *Dynamic-array and interface-typed parameters may default only to `nil`.*
  For dynamic arrays, any other value — an array-literal constructor or a typed
  constant of the same array type — is rejected outright, because it is not
  accepted as a constant expression in this position. (dcc-verified, dcc32
  37.0: `A: TArray<Integer> = nil` compiles; `A: TArray<Integer> = DefArr` (a
  typed constant of the same array type) and `A: TArray<Integer> = [1, 2, 3]`
  (an array-literal default) both fail `E2026 Constant expression expected`.)
  For interfaces, `A: IInterface = nil` likewise compiles (dcc-verified) — a
  non-`nil` interface default was not separately probed, because an interface
  reference has no OTHER form of compile-time constant expression to write in
  the first place, so `nil` being the only option follows from that, not from
  an independently observed rejection.
- ⚠️ *Record-typed parameters cannot have a default value at all* — not even one
  naming a constant of the same record type. (dcc-verified, dcc32 37.0:
  `R: TPoint2 = Origin`, with `Origin` a typed constant of type `TPoint2`, is
  `E2268 Parameters of this type cannot have default values`.)

### 6.2.6 Open array & `array of const` parameters

| | |
|---|---|
| **Introduced** | open arrays Delphi 1; `array of const` Delphi 1/2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`array of T` as a parameter type accepts arrays of any length; `array of const`
accepts a heterogeneous list (variant-like `TVarRec`).

**Grammar**

```ebnf
ParamType = TypeRef
          | "array" "of" ( TypeRef | "const" )      (* open array / array of const *)
          | "string" | "file" ;                      (* untyped open string/file params *)
```

**Example**

```pascal
function Sum(const A: array of Integer): Integer;
procedure Log(const Fmt: string; const Args: array of const);
```

**Semantics & parsing notes**

- ⚠️ *`array of T` as a parameter ≠ a dynamic array type.* It is an **open array
  parameter** — distinct grammar and ABI (passes a pointer + high bound). Use
  `Low`/`High`/`Length` inside. Distinguish from a named dynamic-array type
  argument.
- `array of const` builds a `TVarRec` array; the literal `[a, 'b', 3]` at the call
  site is an open-array constructor, not a set.

### 6.2.7 Untyped parameters

| | |
|---|---|
| **Introduced** | Pascal/Turbo |
| **Deprecated** | — |
| **Status** | ⚠️ Legacy |

A `var`/`const`/`out` parameter with **no type** accepts any type.

```pascal
procedure FillZero(var Buf; Count: Integer);
```

**Semantics & parsing notes**

- Only `var`/`const`/`out` parameters may be untyped (a bare value param needs a
  type). Inside, the parameter is typeless and usually reinterpreted via a cast or
  `absolute`.

---

## 6.3 Overloading

### 6.3.1 The `overload` directive

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Multiple routines may share a name if marked `overload` and distinguished by
parameter signature.

```pascal
function Area(R: TRect): Integer; overload;
function Area(C: TCircle): Double; overload;
```

**Semantics & parsing notes**

- `overload` is a **directive**. Resolution picks the best match by argument types
  (with implicit-conversion ranking); ambiguity is a compile error.
- ⚠️ Overload resolution interacts with **default parameters** and **distinct type
  aliases** (ch.02) — both can change which candidate wins.
- ⚠️ *An overload declared only in the IMPLEMENTATION section joins the interface
  section's set for the same unit.* dcc-verified: a 3-parameter `Conv` in the
  interface and a 4-parameter `Conv` in the implementation, and both call shapes
  compile from inside that implementation.
  - The two are naturally separate SYMBOLS in separate scopes, and must stay so
    — chaining them would export the implementation-only overload to every
    importer, which is exactly what it is not. A resolver therefore has to
    consult BOTH chains whenever it reasons over "all the overloads of this
    name": a call inside the implementation resolves to the nearer head, and
    measuring it against that chain alone reports a perfectly good call to the
    interface overload as having the wrong number of arguments.
- ⚠️ *Across an ANCESTOR — including an ancestor INTERFACE — the set spans the
  hierarchy only when the declarations carry `overload`.* dcc-verified both
  ways: with `IBase` declaring `Take(const A: TSpot)` and `IMore =
  interface(IBase)` declaring `Take(const A: TArray<TSpot>)`, `M.Take(S)`
  compiles when both are marked `overload` and is `E2010 Incompatible types`
  when neither is — the derived declaration simply HIDES the ancestor's, and
  the argument never gets a second candidate to fit.
  - Worth stating because the shape is common in layered interface libraries: a
    base interface declares the single-item method, the derived one adds the
    batch method, and only the `overload` directive keeps both callable through
    the derived reference. A resolver that merges the ancestry unconditionally
    accepts code dcc rejects; one that never merges it rejects code dcc
    accepts.
- ⚠️ *Return type is never part of overload resolution.* Two routines with
  identical parameter lists that differ only in result type are not distinct
  overloads — the second declaration collides with the first rather than
  extending an overload set. (dcc-verified, dcc32 37.0:
  `function F(X: Integer): Integer; overload;` followed by
  `function F(X: Integer): string; overload;` fails with
  `E2004 Identifier redeclared: 'F'` — the *same* diagnostic dcc32 gives for a
  plain non-overloaded re-declaration, not an overload-specific error.) A
  resolver's overload key must be the parameter-type signature only.
- *Parser impact:* none beyond recording the directive; resolution is semantic.

---

## 6.4 Inlining

### 6.4.1 The `inline` directive

| | |
|---|---|
| **Introduced** | Delphi 2005 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Requests the compiler expand the routine body at the call site.

```pascal
function Min(A, B: Integer): Integer; inline;
```

The `{$INLINE ON | OFF | AUTO}` compiler directive controls whether the `inline`
hint above is honored at each call site that is in scope when the directive is
active (it is a compile-time switch, not per-routine — its setting at the CALL
SITE governs, and it can be toggled between calls in the same file):

- `ON` (the default) — a call to a routine marked `inline` is expanded in place;
  a call to a routine WITHOUT the hint is compiled as a normal call.
- `OFF` — the `inline` hint is ignored everywhere it is in effect; every call
  compiles as a normal call, even to a routine marked `inline`.
- `AUTO` — the compiler may inline **even routines with no explicit `inline`
  hint**, at its own discretion, in addition to honoring explicit hints.

**Semantics & parsing notes**

- `inline` is a **directive** here (note: it is *also* a reserved word in some
  lists — Delphi treats it as a directive in this position). It is a *hint*; the
  compiler may decline.
- ⚠️ *`{$INLINE ON}` vs `{$INLINE OFF}` vs `{$INLINE AUTO}` are independently
  dcc-verified as producing different codegen*, not just accepted syntax.
  (dcc-verified, dcc32 37.0, single-call program `Writeln(Min(3, 5))`:
  under `{$INLINE ON}` with `Min` marked `inline`, generated code is 39000
  bytes; under `{$INLINE OFF}` with the SAME `inline`-marked `Min`, it is
  39016 bytes — 16 bytes more, i.e. the hint was ignored and a real `CALL` was
  emitted. Removing the `inline` marker entirely and compiling under
  `{$INLINE ON}` also yields 39016 bytes — confirming `ON` alone does not
  inline a routine that lacks the hint. Removing the marker but switching to
  `{$INLINE AUTO}` yields 39000 bytes again — the SAME size as the explicit
  `inline` + `ON` case — confirming `AUTO` inlined the call on its own
  initiative, with no `inline` directive present at all.) The exact byte counts
  are toolchain/version-specific; what is confirmed is the ON/OFF/AUTO
  *ordering* and that AUTO's extra behavior (inlining un-hinted routines) is
  real, not merely documented.
- ⚠️ *Cross-unit inlining requires the CALLER'S unit to transitively `uses` every
  unit the inline body itself references* — dcc32 does not pull in a hidden
  dependency just to honor the hint; instead it silently degrades to a normal
  (non-inlined) call and, with `{$HINTS ON}`, emits an explicit compiler Hint
  naming the missing unit. (dcc-verified, dcc32 37.0: unit `UnitA2` has
  `function CallsHelperInline: Integer; inline;` whose body calls `Helper`
  from `UnitB2` (which `UnitA2` itself `uses`); a program that `uses UnitA2`
  but NOT `UnitB2` compiles cleanly but with
  `Hint: H2443 Inline function 'CallsHelperInline' has not been expanded
  because unit 'UnitB2' is not specified in USES list`. Adding `UnitB2` to the
  program's own `uses` clause — nothing else changed — removes the hint and
  shrinks the generated code by 12 bytes, confirming the call really was
  expanded in place once the transitive dependency was visible to the
  CALLER's unit.) This is the mechanism, not a guess: the failure mode is
  graceful (a plain call, never an error) and self-reports via `H2443` whenever
  `{$HINTS ON}` (the default) is active.
- The book additionally states inlining never crosses a CIRCULAR unit reference
  (ch.4 of the audited book, ~p.118); this was not independently probed here —
  no clean two-unit circular-`uses` repro was built — so treat it as
  book-stated, not dcc-verified.

---

## 6.5 Calling conventions

### 6.5.1 Convention directives

| | |
|---|---|
| **Introduced** | Delphi 1+ (`register` default); `winapi` later alias |
| **Deprecated** | — |
| **Status** | ✅ Current |

Directives controlling argument passing/cleanup ABI.

**Grammar**

```ebnf
CallConv = "register" | "stdcall" | "cdecl" | "pascal" | "safecall" | "winapi" | "fastcall" ;
```

**Semantics & parsing notes**

- All are **directives** (B.4.2). `register` is the Delphi default; `winapi` maps to
  the platform's native API convention (`stdcall` on Win32, etc.).
- ⚠️ *Placement is loose (all corpus-shipped):* the convention may follow the
  heading **without** a separating semicolon (`function F(...): Bool stdcall;`,
  System.SysUtils.pas), appear inside anonymous-method literals before the body
  (`function(...): HResult stdcall begin`, Vcl.Edge.pas), and several directives
  may run together after one `;` without separators
  (`...; cdecl varargs;`, System.Curl.pas).
- ⚠️ *The semicolon after the LAST directive in a run is OPTIONAL* (dcc-verified):
  `procedure A; stdcall` followed directly by the next declaration compiles, as
  do `procedure P; platform deprecated` + `procedure`/`begin`, and
  `function F; external shell32 name 'X'` + newline + the next `function`.
  A parser must treat any following declaration keyword (or body `begin`) as a
  valid terminator for a directive run and for the `external` clause.
- `safecall` additionally wraps the routine in HRESULT/exception marshalling (COM).
- Mostly codegen, but the parser must accept them in the directive list after a
  heading.

---

## 6.6 Procedural types

### 6.6.1 Procedure/function pointer types

| | |
|---|---|
| **Introduced** | Pascal/D1; `of object` D1; `reference to` Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Types whose values are routines: plain pointers, method pointers (`of object`),
and closures (`reference to`, ch.17).

**Grammar**

```ebnf
ProceduralType = ( "procedure" | "function" )
                 [ "(" [ FormalParams ] ")" ] [ ":" ResultType ]
                 [ "of" "object" | "reference" "to" ] [ CallConv ] ;
```

**Example**

```pascal
type
  TNotify   = procedure(Sender: TObject) of object;   // method pointer
  TComparer = reference to function(A, B: Integer): Integer;  // closure
  TThunk    = procedure;                                // plain pointer
```

**Semantics & parsing notes**

- ⚠️ Three distinct runtime shapes: a **plain** pointer (code address), a **method
  pointer** (`of object` = code + `Self`, 8/16 bytes), and a **reference** (an
  interface-backed closure that can capture locals). Keep the kind on the type
  node — assignment compatibility differs across them.
- ⚠️ *Writing the NAME of a procedural value in a value position CALLS it* — the
  classic Pascal rule, and the one that decides what `.Member` after it means.
  With `ValueFunc: TFunc<IValue>` (that is, `reference to function: IValue`),
  `ValueFunc.GetValue` means `ValueFunc().GetValue`, so `GetValue` is IValue's
  member and not the closure's — `System.Bindings.Outputs` does exactly this.
  `@ValueFunc` is how the value itself is named instead.

  Two shapes are NOT called this way, and a resolver needs both to stop it: a
  procedural type with a parameter the CALLER must supply, and a `procedure`
  type, which has no result to take a member from. The same
  rule seen from the other side is why a guard may be procedural — `if
  AShouldStop then` on a `reference to function: Boolean` is legal because the
  guard's type is the RESULT (ch.05).
  - ⚠️ *"Must supply" and not "declares none":* a parameter with a DEFAULT is
    supplied by the compiler, so a type whose parameters are all defaulted is
    called by writing its name like any other. dcc-verified in both directions
    on `TF = function(A: Integer = 0): TStyler` — `F.Color` compiles, and the
    same line against `function(A: Integer): TStyler` is `E2035 Not enough
    actual parameters`. A resolver that tests for the PRESENCE of a parameter
    list therefore stops one shape too early; the test is whether any parameter
    lacks a default. (Component libraries reach for this deliberately: a global
    `function(AControl: TControl = nil): TStyleServices` reads as a value at
    every call site that wants the default.)
- `of object` and `reference to` use the directives/reserved words `object` and
  `reference`; `reference` is a directive (B.4.2).

---

## 6.7 External declarations

### 6.7.1 `external` (and `name`, `index`, `delayed`, `varargs`)

| | |
|---|---|
| **Introduced** | `external` D1; `delayed` Delphi XE2; `varargs` D-early |
| **Deprecated** | — |
| **Status** | ✅ Current |

Binds a routine to an external library function instead of a Pascal body.

**Grammar**

```ebnf
ExternalDecl = "external" [ ConstExpr ]            (* library name *)
               [ "name" ConstExpr | "index" ConstExpr ]
               [ "dependency" ConstExpr { "," ConstExpr } ]   (* linked-lib hints *)
               [ "delayed" ] ";" ;
```

**Example**

```pascal
function MessageBox(hWnd: HWND; lpText, lpCaption: PChar; uType: UINT): Integer;
  stdcall; external 'user32.dll' name 'MessageBoxW';
```

**Semantics & parsing notes**

- An `external` routine has **no `Block`** — the body is the OS/library symbol.
- `name`/`index` (directives) select the imported symbol; `delayed` defers load
  until first call. `varargs` (with `cdecl`) allows C-style variadic calls.
- ⚠️ *A `delayed` import's DLL/symbol lookup happens at the FIRST CALL, not at
  process load* — if the library or symbol cannot be resolved at that point,
  dcc32 raises `EExternalException` right there (an ordinary, catchable
  exception), rather than failing to load the executable at all. (dcc-verified,
  dcc32 37.0: `procedure NoSuchProc; external 'NoSuchLib_ZZZ.dll' name
  'NoSuchSymbolZZZ' delayed;` compiles cleanly — with a `W1002` warning that
  `delayed` is platform-specific — and calling `NoSuchProc` for the first time
  raises `EExternalException` with message `External exception C06D007E`,
  caught by an ordinary `except on E: Exception` handler around the call.)
- `dependency` (directive) lists additional libraries the import needs at link
  time — used mainly by the mobile/posix toolchains
  (`external libc name 'dlopen' dependency 'dl'`).
- *AST:* `Routine { …, external: { lib, symbol?, index?, delayed } }`.

---

## 6.8 The `noreturn` directive

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Marks a routine that never returns to its caller (always raises/halts), enabling
better flow analysis.

**Example**

```pascal
procedure Fatal(const Msg: string); noreturn;
```

**Semantics & parsing notes**

- `noreturn` is a **directive** (B.4.2, new in 13.0) in the routine's directive
  list. Tells the flow analyser that code after a call to it is unreachable, and
  suppresses "function might not return a value" warnings on paths that end in it.
- *AST:* flag `noReturn: true` on the routine node.

---

## 6.9 Nested routines

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A routine may be declared inside another routine's `Block`, with access to the
enclosing routine's locals.

```pascal
procedure Outer;
  procedure Inner;          // nested
  begin ... end;
begin Inner; end;
```

**Semantics & parsing notes**

- Nested routines create a lexical scope chain — the resolver must allow `Inner`
  to see `Outer`'s locals (captured via a static link / frame pointer).
- *AST:* nested `Routine` within the parent's declaration sections.

---

## 6.10 Inline assembly (`asm … end`)

| | |
|---|---|
| **Introduced** | Turbo Pascal / Delphi 1; x64 support XE2; not supported on ARM targets 🧪 |
| **Deprecated** | — |
| **Status** | ✅ Current (Intel targets only) |

Embeds assembler instructions, either as a **statement** inside a normal body or
as the **entire routine body** (replacing `begin…end`).

**Grammar**

```ebnf
AsmStmt  = AsmBlock ;                       (* as a statement inside a Block *)
AsmBlock = "asm" { ?assembler token? } "end" ;
(* a routine body may be an AsmBlock instead of a begin..end Block; the
   'assembler' directive on the heading is legacy/optional *)
```

**Example**

```pascal
function GetSP: NativeUInt;
asm                       // asm as the whole routine body
  MOV RAX, RSP
end;

procedure P;
begin
  asm                     // asm as a statement
    NOP
  end;
end;
```

**Semantics & parsing notes**

- ⚠️ *Lexer mode switch:* the content between `asm` and its matching `end` is
  **not Object Pascal** — it is BASM (built-in assembler) with its own lexical
  rules: `@@Label:` local labels, register names, `$`-hex and Intel-style `0FFh`
  hex, `//` and `{ }` comments still valid. A parser should treat the body as an
  opaque token stream: scan for the closing `end` at nesting level 0, honouring
  string/comment boundaries. Do **not** try to parse it as Pascal.
- ⚠️ *Closing-`end` detection must treat `@` as a word character:* BASM labels
  may be NAMED `END` — `JS @@END` / `@@END:` ship in Vcl.Graphics.pas — so the
  scan for the closing `end` must require the preceding character to be neither
  an identifier character **nor `@`**. BASM also accepts double-quoted strings
  (`CMP AL,"'"`, System.SysUtils.pas).
- Pascal identifiers (locals, params, globals) may be referenced from BASM —
  symbol resolution *into* the asm body is a semantic/codegen concern; the parser
  only needs to capture the raw text/tokens.
- Platform-gated: x86/x64 only; ARM/ARM64 compilers reject `asm` (RTL guards these
  blocks with `{$IFDEF}`s — 53 uses in RTL 13 sources). The old `assembler`
  directive on headings is accepted and ignored (legacy).
- *AST:* `AsmStmt { rawTokens / sourceRange }` as a statement node or routine body.
