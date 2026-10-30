# 12 — Inheritance & Polymorphism

Single inheritance, the virtual-method model (`virtual`/`override`/`dynamic`/
`reintroduce`), abstract and sealed/final modifiers, message methods, and the
runtime type operators `is`/`as`.

Builds on [ch.11](11-classes.md). Operator semantics for `is`/`as` are in
[ch.04 §4.9](04-expressions-operators.md#49-type-test--type-cast-operators-is-as).

## Chapter grammar umbrella

```ebnf
ClassHeader   = "class" [ "abstract" | "sealed" ] [ "(" Ancestor [ "," InterfaceList ] ")" ] ;
MethodDirective = "virtual" | "dynamic" | "override" | "abstract" | "final"
                | "reintroduce" | "overload" | "message" ConstExpr
                | "static" | "inline" | CallConv ;
```

---

## 12.1 Inheritance

### 12.1.1 Single inheritance

| | |
|---|---|
| **Introduced** | Delphi 1 (1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A class extends exactly one ancestor (default `TObject`), inheriting its members
and adding/overriding.

**Example**

```pascal
type
  TAnimal = class
    procedure Speak; virtual;
  end;
  TDog = class(TAnimal)        // single ancestor
    procedure Speak; override;
  end;
```

**Semantics & parsing notes**

- ⚠️ *Single inheritance only* for classes (multiple inheritance is achieved via
  interfaces, ch.14). The ancestor list `( Ancestor, IFoo, IBar )` has **at most
  one class** (first position) followed by interfaces.
- Omitted ancestor ⇒ `TObject`. Only the WHOLE list can be omitted: a list
  that starts with an interface is `E2021 Class type required`
  (`TFoo = class(IInterface)`, dcc64 37.0, `probe-b19` Q5; the same body
  under `class(TObject, IInterface)` compiles) - the first entry of a
  class's heritage list is always its ancestor. A tool indexing descendants
  must still give a class with NO list its implicit TObject edge: that is
  the most common class there is.
- ⚠️ *An INHERITED member outranks a unit-level routine of the same name inside a
  method body* — including one in the method's OWN unit, and including one
  declared in the `implementation` section right next to it. dcc-verified on
  `FMX.Dialogs.Win`, which compiles: `TCustomFileDialog` declares a 1-parameter
  method `GetFileNames`, the implementation section declares a 4-parameter
  procedure of the same name, and `GetFileNames(FShellItems)` inside the
  descendant `TCustomFileOpenDialog.GetResults` means the METHOD. A resolver that
  joins only a class's OWN member scope into a method body binds the global and
  then reports the call as `E2035 Not enough actual parameters` — the diagnostic
  points at the argument count, but the defect is the binding.
  - *The same holds against a PREDEFINED name* - stated, with its probe, in
    11.4.1 ("A member - including an INHERITED one - outranks a
    compiler-PREDEFINED name"). Beyond that section's `Text`, it covers
    `property Word: string` (bare `Word` in a descendant's method is the
    property, not the type) and any member named `Real`, `Insert`, `Delete`,
    `Copy`, `Round`, ... which component code is full of.
  - *And against a USED UNIT's name.* A dotted `uses` makes the unit reachable
    by its LAST segment, so `uses Some.Header` puts `Header` in scope as a unit
    — but a bare unit name is never a value, so in a class with a `Header`
    member the member wins. The shape is common wherever a library splits a unit
    family by dotted prefix and names a property after one of the leaf units.
    - ⚠️ *An implementation that exempts "namespace tokens" from
      undeclared-identifier reporting must apply that exemption AFTER the member
      walk, not before it.* Tested first, the exemption skips exactly the names
      that had a member to find, and the failure is silent: the qualifier types
      as nothing, so a `with` over it opens no scope and every name in the body
      is reported instead.
  - ⚠️ *Consequence for an implementation that resolves inherited members in a
    LATER pass than it type-checks:* the early pass sees the predefined name,
    types the expression from it, and reports an assignment as `E2010
    Incompatible types`. A later pass can fix the binding but cannot unsay the
    diagnostic. The type checker therefore must not judge an assignment whose
    operand resolved to a TYPE NAME — that is a mis-binding, not a type
    mismatch, and a compiler has separate errors for a type used as a value.
- ⚠️ *Assignment compatibility between a class and its ancestor is covariant,
  one direction only* (dcc-verified, dcc32 37.0: with `TAnimal = class; TDog =
  class(TAnimal);` and `A: TAnimal; D: TDog;`, `A := D;` compiles, but
  `D := A;` fails as `E2010 Incompatible types: 'TDog' and 'TAnimal'`). A
  descendant-typed value can always be assigned or passed where the ancestor
  type is expected — it IS one — but the reverse needs an explicit checked
  (`as`) or unchecked (hard-cast) narrowing, since a variable's declared type
  is not proof of the object's actual runtime type. See §12.4.1.
- ⚠️ *The ancestor may be a NESTED type named through its outer one*, commonly
  from another unit: `TMemoTextSettings = class(TTextSettingsInfo.
  TCustomTextSettings)` (`FMX.Memo`, and the same line in eight sibling units).
  Failing to resolve that reference is silent and disproportionate — the class is
  left with NO ancestry, so every inherited member its methods touch becomes a
  false `E2003`, and only the members it happens to REDECLARE keep working.
- *AST:* `ClassType { ancestor, interfaces[] }`.

### 12.1.2 `inherited`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Calls the ancestor's implementation of the current (or a named) method.

**Example**

```pascal
procedure TDog.Speak;
begin
  inherited;            // calls TAnimal.Speak (same name & args)
  Writeln('Woof');
end;
```

**Semantics & parsing notes**

- ⚠️ *A named `inherited X` names a METHOD or a PROPERTY* of an ancestor:
  `inherited M;` and `Z := inherited P;` compile, while a field (`inherited
  F := 1`), a class var, a nested type (`SizeOf(inherited TN)`) or a class
  constant (`inherited K`) is `E2096 Method identifier expected` (dcc64 37.0,
  2026-10-15, PasTree local/probe-cc P7).
- ⚠️ *Bare `inherited`* (no name) calls the ancestor method of the **same name**,
  forwarding the **same arguments** — the resolver must reconstruct the current
  method's signature. `inherited Foo(x)` names an explicit ancestor method.
- ⚠️ *Bare `inherited` takes selectors directly:* `inherited.AsExtended`
  (FMX.Grid.Style.pas) — allow a selector chain on the bare form.
- ⚠️ *A bare `inherited` is a value:* the call of the method it names, typed
  by that method's result - `Result := inherited + 1` over an ancestor `V:
  Integer` returning 7 gives 8, and `I := inherited.X` reads the X of the
  record the ancestor's `G: TR` returns (I1 below).
- ⚠️ *Inside a nested routine* (or an anonymous method) of a method, a bare
  `inherited` still means the enclosing **method** - its name and its
  arguments, not the nested routine's. In `procedure TN.Fin; procedure Inner;
  begin inherited; end;` it calls TBase.Fin, though TBase declares an `Inner`
  too; in `function TN.W(A: Integer): Integer; function Sub(B: Integer):
  Integer; begin Result := inherited + B; end;` it passes W's `A` (I2: W(4)
  is `40 + 1`).
- ⚠️ *The arguments a bare `inherited` passes are the method's DECLARED
  parameters*, which the implementation of a non-overloaded method may leave
  unwritten: `procedure TDer.Foo; begin inherited; end;` for a declared
  `procedure Foo(A: Integer);` calls TBase's `Foo(A: Integer)`, not the
  parameterless `Foo` overload beside it (I1). Among the ancestor's
  overloads, those arguments pick the one called.
- ⚠️ *In a helper's method* (15.3.1) no helper is ever what `inherited`
  means - neither the helper itself nor its ancestor helpers. A named
  `inherited X` is the **extended type's own** X, declared there or
  inherited; a bare `inherited` is the method of the method's name in the
  extended type's **ancestor**. With `TH0 = class helper for TMid` declaring
  `St`, `TH = class helper(TH0) for TMid` declaring `St` and `Fin`, and TMid
  overriding TBase's virtual `St`: in TH.St, `inherited St` runs TMid.St and a
  bare `inherited` TBase.St; in TH.Fin a bare `inherited` runs TBase.Fin (I1
  prints `TH.St/TMid.St/TBase.St`). Since a helper's method hides the
  extended type's method of the same name for every caller (15.3.3), this is
  how a helper "patches" a method and still calls the original.
- ⚠️ *In a `message` method* (12.3.1) a bare `inherited` calls the ancestor's
  handler for the **same message id**, whatever its name, and DefaultHandler
  when there is none - the name rule above does not apply there. The id is
  compared by VALUE: with `message MBase + 1` in TBase.WMFoo and in
  TMid.MyFoo, MyFoo's bare `inherited` runs WMFoo, and TMid.MyBar's (`MBase +
  2`, no ancestor handler) runs TBase.DefaultHandler (dcc64 37.0, 2026-10-30,
  runtime output of local/probe-park/e244 MI.dpr; PasTree audit E2-44).
- *Evidence:* dcc64 37.0, 2026-10-12, runtime output of PasTree's
  local/probe-b30 I1.dpr (the value, declared-arity and helper rules) and
  I2.dpr (the nested-routine rule); the message-id rule from the PasTree
  audit's runtime probe (E1-22, 2026-09-30).
- `inherited` is a **reserved word** (§B.4.1) and may appear as the head of a
  `Designator` (§B.8).
- *AST:* `Inherited { methodName?, args? }`.

---

## 12.2 Virtual methods & polymorphism

### 12.2.1 `virtual` / `override`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`virtual` enables late binding via the VMT; `override` replaces an inherited
virtual method while remaining polymorphic.

**Semantics & parsing notes**

- ⚠️ *Signature match required:* `override` must match the ancestor's `virtual`/
  `dynamic` signature exactly; otherwise it is an error (use `reintroduce`/
  `overload` to deliberately diverge). Resolver checks against the ancestor VMT.
- ⚠️ *The `override` errors, by cause:*
  - no inherited method of that name: `E2137 Method 'Nope' not found in base
    class`;
  - one exists but differs in a parameter TYPE or the result type
    (`V(X: string)` over `V(X: Integer)`, `R: Byte` over `R: Integer`):
    `E2037 Declaration of 'V' differs from previous declaration`;
  - one exists but is not virtual: `E2170 Cannot override a non-virtual
    method`; it is `final`: `E2352 Cannot override a final method` (12.2.5);
  - redeclaring a virtual method with no directive is legal but warned:
    `W1010 Method 'Hid' hides virtual method of base type 'TA'` (12.2.3).
  NOT checked: parameter NAMES (`V(Y: Integer); override` over `V(X:
  Integer)` compiles), default values (`W(X: Integer = 2); override` over
  `= 1` compiles, and a call `A.W` takes the default of the STATIC type: 1
  through a `TA` variable, 2 through a `TB`, both running TB.W), and
  visibility (a `private` override of a `public` virtual compiles, and the
  base-typed call still dispatches to it). dcc64 37.0, 2026-10-30 (spec
  audit g4, local/probe-park/spec-audit/g4 IOvr.dpr, IOvr2.dpr; run-time
  output for the dispatch and defaults).
- A call through a base-typed reference dispatches to the most-derived `override`.
- ⚠️ *Which slot an `override` joins - the chain a "find overrides" tool
  walks* (dcc64 37.0, probed 2026-10-10, `probe-b19` Q1, Q2):
  - a descendant that redeclares the name `virtual` (or `dynamic`) without
    `override` starts a NEW slot (W1010 `hides virtual method of base
    type`), and so does `reintroduce; virtual`: with `TA.Draw; virtual`,
    `TB = class(TA) Draw; virtual` and `TC = class(TB) Draw; override`, a
    `TA`-typed TC runs TA.Draw and a `TB`-typed one TC.Draw - TC overrides
    TB's slot, not TA's;
  - a redeclaration with no directive hides the slot for good below it:
    an `override` in its descendant is `E2170 Cannot override a non-virtual
    method`;
  - `reintroduce; overload` with ANOTHER parameter list does not end the
    original slot: with `TF = class(TA) Draw(X: Integer); reintroduce;
    overload` and `TG = class(TF) Draw; override`, a `TA`-typed TG runs
    TG.Draw;
  - which of these a class does is decided per overload, by that
    declaration's own directives, never by its siblings'.
- ⚠️ *In an `object` type (11.5) a descendant's `virtual` IS the override*
  (`probe-b19` Q3, Q4): `TO2 = object(TO1) procedure M; virtual; end` over
  `TO1.M; virtual` is late-bound through a `^TO1` (prints TO2's), and
  `procedure M; override;` there is not a directive - `override` reads as
  a field after a method: `E2037 Declaration of 'M' differs from previous
  declaration`, `E2169`.
- *AST:* method `binding ∈ { static, virtual, dynamic }`, `isOverride`.

### 12.2.2 `dynamic`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Like `virtual` but dispatched via a sparse dynamic-method table (smaller code,
slower calls) — semantically identical, an optimization trade-off.

**Semantics & parsing notes**

- Overridden with `override` just like `virtual`. The choice is performance/size,
  not behaviour — record the binding kind but treat polymorphism identically.

### 12.2.3 `reintroduce`

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Suppresses the "method hides ancestor" warning when intentionally redeclaring a
name that exists in the ancestor (without overriding it).

**Example**

```pascal
function Create(X: Integer): TThing; reintroduce; overload;
```

**Semantics & parsing notes**

- `reintroduce` starts a **new** non-overriding member that shadows the inherited
  one; calls through a base reference still hit the ancestor version. Often paired
  with `overload`.
- `reintroduce` on a method that hides nothing (no ancestor member of that
  name) is accepted silently - no error, no hint (dcc64 37.0, 2026-10-30,
  spec audit g4, local/probe-park/spec-audit/g4 IOvr.dpr, `TB.Fresh`).
- ⚠️ *Plain redefinition of a NON-virtual ancestor method needs no keyword at
  all, and is static shadowing rather than late binding* (dcc-verified, dcc32
  37.0) — distinct from the `reintroduce`-for-hiding-a-virtual case above. If
  `TBase.Speak` is a plain (non-`virtual`/non-`dynamic`) method, a descendant
  `TDerived` may redeclare `procedure Speak;` with no directive whatsoever
  (not even `reintroduce`); the compiler emits no hint, because there is no
  virtual ancestor method to warn about hiding. The result is two entirely
  separate, statically-bound symbols: calling `Speak` through a
  `TBase`-typed variable that happens to hold a `TDerived` instance still
  runs `TBase.Speak`'s body (the call is resolved from the variable's
  *declared* type, at compile time); only a `TDerived`-typed variable (or a
  cast to `TDerived`) reaches `TDerived.Speak`. Contrast with `virtual`/
  `override` (§12.2.1), where a base-typed call dispatches to the
  most-derived override regardless of the variable's declared type.
- *AST:* `isReintroduce: true`.

### 12.2.4 `abstract` methods

| | |
|---|---|
| **Introduced** | Delphi 1 (`virtual; abstract`) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `virtual; abstract` method declares a contract with no body; descendants must
override it.

**Example**

```pascal
procedure Speak; virtual; abstract;
```

**Semantics & parsing notes**

- ⚠️ `abstract` requires `virtual`/`dynamic` (`procedure Ab; abstract;` is
  `E2167 Abstract methods must be virtual or dynamic`). Instantiating a class
  with an unoverridden abstract method is allowed to compile but raises
  `EAbstractError` at the call — a semantic/runtime note.
- ⚠️ *`inherited` aimed at an abstract method:* a BARE `inherited` in the
  override is compiled to nothing - no call, no exception (the method goes
  on); a NAMED `inherited Ab` compiles and raises `EAbstractError` when it
  runs. dcc64 37.0, 2026-10-30 (spec audit g4,
  local/probe-park/spec-audit/g4 IAbs.dpr, IOvr2.dpr; run-time output).
- ⚠️ *The compile-time signal and the run-time signal are two different
  mechanisms, firing at two different sites* (dcc-verified, dcc32 37.0).
  Constructing an instance of a class with an unoverridden abstract method
  (`TBase.Create` where `TBase` declares `procedure Go; virtual; abstract;`
  and no descendant overrides it) compiles with `W1020 Constructing instance
  of 'TBase' containing abstract method 'TBase.Go'` — a **warning**, not an
  error, so the build still succeeds and the instance is created. Only
  actually *calling* the abstract method on such an instance raises
  `EAbstractError` (RTL runtime error 210 if unhandled) — the warning fires
  once per unoverridden abstract method, at the construct site; the exception
  fires only if and when that method is invoked.
- ⚠️ *`abstract` as a class-level directive (`class abstract(Ancestor)` —
  the `ClassHeader`/`ClassType` grammar's `[ "abstract" | "sealed" ]`/
  `[ Abstract ]` option) is documentation only — it does NOT block
  instantiation, unlike C++/Java* (dcc-verified, dcc32 37.0: `TFoo = class
  abstract(TBase)` with zero actual abstract methods — `TFoo.Create` compiles
  with no warning at all and the instance works normally). It is orthogonal
  to the per-METHOD `virtual; abstract` combination above: marking the CLASS
  `abstract` signals intent to readers/tooling, but only an actual
  unoverridden `virtual; abstract` method triggers `W1020` at construction
  (previous bullet) — a class can be marked `abstract` and still be freely
  instantiated if it has no such method.
- *AST:* `isAbstract: true` (no body).

### 12.2.5 `sealed` classes & `final` methods

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`sealed` forbids subclassing a class; `final` forbids further overriding of a
virtual method.

**Example**

```pascal
type
  TLeaf = class sealed (TBase) end;

procedure DoIt; override; final;
```

**Semantics & parsing notes**

- `sealed`/`final` are **directives** (B.4.2). The resolver must reject a class
  deriving from a `sealed` class, and an `override` of a `final` method.
- ⚠️ *Where the class modifiers stand* (dcc-verified, dcc64 37.0,
  2026-09-30): `abstract` and `sealed` follow `class` or `object` (an
  object type takes both too) right after the word, before the ancestors -
  `class sealed(TBase)`, a forward `class abstract;`, `class
  sealed(TObject);` stopping at its ancestors. dcc takes the word there
  whatever follows it: `class` NEWLINE `abstract: Integer;` is E2029, not a
  field named `abstract`; after the ancestors a field may be named so
  (`class abstract(TObject) abstract: Integer; end` compiles). A repeat is
  one (`class abstract abstract` compiles to the `.dcu` of `class
  abstract`), both together are `E2383 ABSTRACT and SEALED cannot be used
  together`. After `record`, `interface` and `dispinterface` the word is a
  member's name: `record abstract: Integer; sealed: Byte; end` compiles,
  `record sealed end` is E2029 (a field with no type). A helper takes
  neither (`class sealed helper for T` and `class helper sealed for T`:
  E2029). `packed` comes before `class`: `packed class sealed` compiles,
  `class sealed packed` does not. Deriving from a sealed class is `E2353
  Cannot extend sealed class`.
- `final` needs a virtual slot: on a static method (`procedure NF; final;`)
  it is `E2351 Final methods must be virtual or dynamic`; overriding a
  `final` one is `E2352 Cannot override a final method`. dcc64 37.0,
  2026-10-30 (spec audit g4, local/probe-park/spec-audit/g4 IOvr.dpr).
- *AST:* `isSealed` on class, `isFinal` on method.

---

## 12.3 Message methods

### 12.3.1 `message` methods

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A method dispatched by an integer message id (Windows messaging / custom).

**Grammar**

```ebnf
MessageMethod = "procedure" Ident "(" FormalParam ")" ";" "message" ConstExpr ";" ;
```

**Example**

```pascal
procedure WMSize(var Msg: TWMSize); message WM_SIZE;
```

**Semantics & parsing notes**

- `message` is a **directive** taking a constant message id; the single `var`
  parameter receives the message record. Dispatched dynamically by id, not by name.
- ⚠️ *The shape is enforced:* an instance `procedure` with exactly ONE `var`
  parameter, typed (`var Msg: TWMSize`, `var Msg: Integer`) or untyped
  (`var Msg`). A value parameter, two parameters, none, a `function` or a
  `class procedure` are each `E2138 Invalid message parameter list`. A
  handler for the same id in a descendant needs no `override` and may have
  another name (TB's `M3` for id 100 replaces TA's `M2` under `Dispatch`;
  its bare `inherited` reaches `M2`, 12.1.2). dcc64 37.0, 2026-10-30 (spec
  audit g4, local/probe-park/spec-audit/g4 IMsg.dpr, IOvr2.dpr).
- *AST:* `MethodDecl { …, messageId }`.

---

## 12.4 Runtime type tests & casts

### 12.4.1 `is`, `as`, and hard casts

| | |
|---|---|
| **Introduced** | Delphi 1; `is not` 13.0 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`is` tests the runtime type; `as` performs a checked downcast; `TClass(obj)` is an
unchecked hard cast.

**Example**

```pascal
if Sender is TButton then
  TButton(Sender).Caption := 'X';     // hard cast (unchecked)
Lbl := Sender as TLabel;              // checked: raises EInvalidCast on mismatch
```

**Semantics & parsing notes**

- *`as`* raises `EInvalidCast` if the object is not of the target type; the **hard
  cast** `TType(obj)` does not check — undefined behaviour on mismatch.
- Operates on class and interface references; for interfaces `as` also does
  `QueryInterface`.
- `is not` (13.0) negates the test (ch.04 §4.9.1).
- ⚠️ *Statically unrelated classes are a COMPILE-time error, not a False:*
  with sibling classes `TDog` and `TCat`, `D is TCat` and `D as TCat` (for
  `D: TDog`) are `E2010 Incompatible types: 'TDog' and 'TCat'` - the test
  needs the target to be a descendant (or ancestor) of the operand's static
  type; the hard cast `TCat(D)` compiles. An interface operand takes a class
  target (`I is TDog`, `I as TObject` compile), but a class operand cannot
  be tested against an interface TYPE with `is` (`O is IInterface`: `E2015
  Operator not applicable to this operand type`). dcc64 37.0, 2026-10-30
  (spec audit g4, local/probe-park/spec-audit/g4 IIsAs.dpr).
- *AST:* `IsExpr`, `AsExpr`, `TypeCastExpr` (ch.04 §4.10).
