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
- Omitted ancestor ⇒ `TObject`.
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

- ⚠️ *Bare `inherited`* (no name) calls the ancestor method of the **same name**,
  forwarding the **same arguments** — the resolver must reconstruct the current
  method's signature. `inherited Foo(x)` names an explicit ancestor method.
- ⚠️ *Bare `inherited` takes selectors directly:* `inherited.AsExtended`
  (FMX.Grid.Style.pas) — allow a selector chain on the bare form.
- `inherited` is a **reserved word** (§B.4.1) and may appear as the head of a
  `Designator` (§B.8).
- *AST:* `InheritedExpr { methodName?, args? }`.

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
- A call through a base-typed reference dispatches to the most-derived `override`.
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

- ⚠️ `abstract` requires `virtual`/`dynamic`. Instantiating a class with an
  unoverridden abstract method is allowed to compile but raises
  `EAbstractError` at the call — a semantic/runtime note.
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
- *AST:* `IsExpr`, `AsExpr`, `TypeCastExpr` (ch.04 §4.10).
