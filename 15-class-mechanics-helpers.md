# 15 — Class-Level Mechanics & Helpers

Members that belong to the class rather than an instance (class methods/vars/
properties, class constructors), metaclasses (`class of`), and the helper
mechanism (class & record helpers, including helpers for intrinsic types).

Builds on [ch.11](11-classes.md)/[ch.12](12-inheritance-polymorphism.md).

## Chapter grammar umbrella

```ebnf
ClassVarDecl   = "class" ( "var" | "threadvar" ) FieldList ;
ClassMethod    = "class" ( "procedure" | "function" | "constructor" | "destructor" )
                 Ident [ "(" FormalParams ")" ] [ ":" ResultType ] ";"
                 { MethodDirective | "static" } ;
ClassProperty  = "class" "property" ... ;          (* as ch.13, class-scoped *)
ClassRefType   = "class" "of" TypeRef ;
HelperType     = "class" "helper" [ "(" Ancestor ")" ] "for" TypeRef ClassBody "end"
               | "record" "helper" "for" TypeRef ClassBody "end" ;
```

---

## 15.1 Class methods & class data

### 15.1.1 Class methods

| | |
|---|---|
| **Introduced** | Delphi 1; `class function` etc. |
| **Deprecated** | — |
| **Status** | ✅ Current |

A method called on the class itself; `Self` inside is the **class reference**
(metaclass), enabling polymorphic factory patterns.

**Example**

```pascal
type
  TShape = class
    class function Describe: string; virtual;
  end;
```

**Semantics & parsing notes**

- ⚠️ *`Self` is a class reference here*, not an instance — class methods may be
  `virtual` and dispatched via the metaclass (15.2). Distinguish from `static`
  class methods (15.1.4) which have no `Self`.
- *AST:* `MethodDecl { isClassMethod: true, … }`.

### 15.1.2 `class var`

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Storage shared by all instances of the class (one copy per class).

**Example**

```pascal
type
  TCounter = class
    class var Total: Integer;
  end;
```

**Semantics & parsing notes**

- ⚠️ `class var` opens a sub-section: fields after it are class-level **until** the
  next visibility specifier or `var`. Track the "class storage" flag while parsing
  the section.
- ⚠️ *`class threadvar`* is the thread-local variant — one copy **per class per
  thread**. Same sub-section behaviour; it can directly follow a visibility
  specifier on one line (`private class threadvar`, `protected class threadvar` —
  both used in the RTL: `System.Classes`, `System.Threading`). No initializers
  (same rule as unit-level `threadvar`, ch.03 §3.1.5).
- *AST:* `FieldDecl { isClassVar: true, threadLocal?: true }`.

### 15.1.3 Class properties

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A property at class level, backed by `class var`/class methods.

```pascal
class property Instance: TFoo read GetInstance;
```

### 15.1.4 `static` class methods

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A class method with **no implicit `Self`** — a plain namespaced function.

**Example**

```pascal
class function Make: TFoo; static;
```

**Semantics & parsing notes**

- `static` (directive) removes the class-reference `Self`; such methods cannot be
  `virtual`. Required for class methods used as ordinary procedure references.

### 15.1.5 Class constructors & destructors

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`class constructor`/`class destructor` run **once** automatically at unit
initialization/finalization — for class-level setup (not instance creation).

**Example**

```pascal
type
  TCache = class
    class constructor Create;   // runs once, at startup
    class destructor Destroy;   // runs once, at shutdown
  end;
```

**Semantics & parsing notes**

- ⚠️ *Not callable explicitly, take no parameters, run automatically.* Distinct from
  instance constructors. The parser records `kind = classConstructor/classDestructor`.
- ⚠️ *So it is never what a NAME means either* — which matters most where it
  shares a name with a real constructor, and that is the common case. `TRegistry`
  (System.Win.Registry) declares a private `class constructor Create` fourteen
  lines above the public parameterless `constructor Create`: `TRegistry.Create`
  means the latter, and a resolver that stops at the first declaration of the
  name binds the former with no diagnostic to show for it. Nor does one hide an
  INHERITED constructor — with a `class constructor Create` as a class's only own
  `Create`, `TDesc.Create(7)` still resolves to the ancestor's
  `constructor Create(A: Integer)` (dcc32 37.0-probed).
- The name is not fixed: `class constructor Init;` compiles, so these cannot be
  recognised by name — only by `class` together with `constructor`/`destructor`.
- ⚠️ *Ordering relative to the unit's own `initialization`/`finalization`,
  same unit* (dcc-verified, dcc32 37.0): a class constructor runs BEFORE its
  unit's `initialization` section, and a class destructor runs AFTER that
  unit's `finalization` section. For a unit declaring both, the observed order
  across a whole program run is: class constructor → `initialization` →
  (program body) → `finalization` → class destructor. Book-stated only, not
  independently confirmed here — verifying it would require inspecting
  binary/symbol presence rather than program output, which this probe pass did
  not attempt: class-constructor/destructor code is linked into the program
  only if the class is actually USED, whereas a unit's `initialization`/
  `finalization` is always linked in once the unit itself is compiled into the
  program (i.e. class ctors/dtors are more linker-friendly than unit init
  code for otherwise-unused classes).
- ⚠️ *At most one class constructor and one class destructor per class* — a
  second one is a compile-time error, `E2359` (dcc-verified, dcc32 37.0: a
  class declaring both `class constructor Create` and `class constructor Foo`
  fails with `E2359 Multiple class constructors in class TTestClass: Create
  and Foo`). The same one-per-class limit applies independently to class
  destructors.

---

## 15.2 Class references (metaclasses)

### 15.2.1 `class of` types

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A metaclass type whose values are **class references** (the class itself, not an
instance).

**Grammar**

```ebnf
ClassRefType = "class" "of" TypeRef ;
```

**Example**

```pascal
type
  TShapeClass = class of TShape;
var
  C: TShapeClass;
begin
  C := TCircle;
  Shape := C.Create;     // polymorphic construction via virtual constructor
end;
```

**Semantics & parsing notes**

- ⚠️ *Virtual-constructor pattern:* calling a `virtual` constructor through a
  `class of` reference instantiates the **actual** class — the language's
  polymorphic factory. The resolver must allow constructor calls on metaclass
  values.
  - ⚠️ *Virtuality picks the constructor BODY, not the allocated class*
    (dcc-verified, dcc32 37.0): `TObject.NewInstance` — the method that
    actually allocates and sizes the instance — is itself virtual and always
    dispatches on the metaclass value's RUNTIME class, regardless of whether
    `Create` is declared `virtual`. Calling `GetDerivedClass.Create` through a
    reference statically typed `class of TBase`, with a **non-virtual**
    `Create`, still allocates a correctly-sized instance of the derived class
    (`Obj is TDerived` is `True`, `Obj.InstanceSize` matches `TDerived`'s, not
    `TBase`'s) — it just runs `TBase.Create`'s code on it, statically bound,
    because `Create` itself never dispatches. Declaring `Create` `virtual` (and
    overriding it) is what makes the DERIVED class's *initialization code* run
    instead — the correct-class allocation was never in question either way.
- ⚠️ *The metaclass value may be any EXPRESSION, not just a variable or a type
  name* — most often a function result, which is how the factory is usually
  written:

  ```pascal
  with GetPainterClass.Create(Canvas, Self) do   // GetPainterClass: TPainterClass
    try
      MainPaint;                                 // member of the REFERENCED class
    finally
      Free;
    end;
  ```

  A resolver that recognises a constructor call only when the qualifier is a
  type NAME types this as nothing, and then every member reached through it is a
  false undeclared-identifier. The qualifier has to be TYPED and then unwrapped:
  `class of T` yields `T`, chasing alias links, since the reference type is
  almost always reached through one (`TPainterClass = class of TPainter`).
- `TObject`'s `ClassName`/`ClassParent`/`InheritsFrom` are class methods and can
  be called directly on a metaclass value or class-reference expression. ⚠️
  *`ClassType` is NOT one of them* (dcc-verified, dcc32 37.0: `TFoo.ClassType`
  where `TFoo` is a class reference, not an instance, fails with `E2076 This
  form of method call only allowed for class methods or constructor`) — it's
  an ordinary (instance) method, valid on an object instance
  (`Inst.ClassType`) but not on the class itself.
- *AST:* `ClassOf { baseClass }`.

---

## 15.3 Helpers

### 15.3.1 Class helpers

| | |
|---|---|
| **Introduced** | Delphi 2005/2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `class helper` injects additional methods/properties into an existing class
**without** subclassing it.

**Grammar**

```ebnf
ClassHelper = "class" "helper" [ "(" Ancestor ")" ] "for" TypeRef
                { ClassMember }
              "end" ;
```

**Example**

```pascal
type
  TStringsHelper = class helper for TStrings
    function Join(const Sep: string): string;
  end;
```

**Semantics & parsing notes**

- ⚠️ Helpers **cannot add instance fields** (no per-instance storage); only methods,
  class vars, properties (backed by existing state). Enforce in the type-checker.
- ⚠️ *A class helper may INHERIT from another helper* (dcc-verified, dcc32
  37.0: a helper `for T` declared `(Ancestor)` for another helper of the same
  `T` resolves the ancestor's members through the descendant helper) — that is
  what the optional `( Ancestor )` in the grammar above is, and it names a
  HELPER, not the extended type. The ancestor helper's members apply to the
  extended type as well, so a member walk that reaches a helper must try the
  helper's own ancestors before it continues into the `for` target.
  - ⚠️ *Record helpers cannot* (dcc-verified): `record helper(TBaseH) for T` is
    `E2029 ',' or ':' expected` — that production has no ancestor slot at all,
    so the `(` is read as the start of something else. The grammar in §15.3.2
    is the whole story there.
  - Helpers cannot form cycles (an ancestor must already be declared), so the
    walk is bounded by the declaration chain and needs no cycle guard of its
    own.
- ⚠️ *Since Delphi 10.1 Berlin, a class helper's methods cannot reach a
  visibility level that wouldn't already apply without the helper mechanism* —
  concretely: `strict private` members of the extended type are unreachable
  from a helper's methods (`E2003` undeclared identifier) even when the helper
  is declared in the SAME unit as the extended class (dcc-verified, dcc32
  37.0); and plain `private` members are unreachable from a helper declared in
  a DIFFERENT unit than the extended class (dcc-verified: `E2003` there too,
  same as any other cross-unit `private` access). What still works, and is
  *not* the old hole being described: a helper in the SAME unit as a class
  reading that class's plain (unit-scoped) `private` field compiles fine
  (dcc-verified) — for the ordinary reason that Object Pascal `private` is
  unit-scoped, not class-scoped, not because the helper mechanism grants any
  special access. Before 10.1 Berlin, a compiler bug let helpers reach private
  members across the real boundary too (cross-unit `private`, any-unit
  `strict private`) — never an intended feature, and closed since.
- *AST:* `HelperType { kind: class, ancestor?, forType, members[] }`.

### 15.3.2 Record helpers (incl. intrinsic types)

| | |
|---|---|
| **Introduced** | record helpers Delphi 2006; **helpers for intrinsic types** XE3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`record helper for T` adds methods to a record — or, since XE3, to **intrinsic
types** (`Integer`, `string`, `Boolean`, …), which is how `S.Length`/`I.ToString`
work.

**Example**

```pascal
type
  TIntHelper = record helper for Integer
    function ToString: string;
  end;
// usage: 42.ToString
```

**Semantics & parsing notes**

- ⚠️ Method-call syntax on a value of an intrinsic type (`42.ToString`,
  `S.ToUpper`) resolves through the active helper. The parser sees an ordinary
  `Designator` member access on a literal/expression — the resolver routes it.
- ⚠️ *The target is a TYPE, not a name* — and an intrinsic type has several
  names. A `record helper for Cardinal` applies to values declared `Cardinal`,
  `LongWord` and `UInt32` alike, because those are one type; it is `E2671` on an
  `Integer`. Same for `Integer`/`LongInt`/`Int32`, `Char`/`WideChar` and
  `string`/`UnicodeString` (all dcc32 37.0-probed). Two consequences for a
  resolver that keys helpers by symbol identity: a plain alias
  (`UInt32 = Cardinal`) must be followed to the type it names, and the names
  that are seeded separately have to be grouped by hand.
- ⚠️ *...but `T = type Base` is a DISTINCT type* (2 §2.5.1), so a helper for it
  is NOT a helper for `Base`, and — since at most one helper is active per type
  (§15.3.3) — registering it as one HIDES the real `Base` helper. Real shape:
  `TEditMask = type string` (System.MaskUtils) has its own helper, and treating
  that as a string helper made ordinary `SomeString.Substring` unresolvable
  throughout FMX.MaskEdit.
- Same no-instance-fields restriction as class helpers.
- Same private-access restriction as class helpers (§15.3.1): `strict private`
  members of the extended record are unreachable from a record helper's
  methods even within the same unit, and plain `private` members are
  unreachable across a unit boundary (dcc-verified, dcc32 37.0, same-unit
  plain-`private` case: a record helper reading its extended record's ordinary
  `private` field from within the same unit compiles fine, for the same
  unit-scoping reason as class helpers above).

### 15.3.3 Helper scope-resolution rule

| | |
|---|---|
| **Introduced** | Delphi 2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

**At most one helper is active for a given type at any point** — the one most
recently in scope (latest `uses` / nearest declaration) wins; others are hidden.

**Semantics & parsing notes**

- ⚠️ *Critical resolution rule:* unlike overloads, helpers do **not** accumulate.
  If two units both declare a helper for `TStrings`, only the last-in-scope one's
  methods are visible. The resolver must pick a single active helper per type per
  source position (`uses`-order dependent, ch.01 §1.2.1).
- This single-helper rule is a frequent source of "method not found" confusion —
  worth a diagnostic.

### 15.3.4 Where a helper is active (activation scope)

| | |
|---|---|
| **Introduced** | Delphi 2006 (with helpers) |
| **Deprecated** | — |
| **Status** | ✅ Current |

§15.3.3 says *which* helper wins when several are in scope. This section says
what "in scope" means in the first place — and the answer is not the helper
type's own name visibility. A helper activates for the **whole enclosing
non-structured scope**, regardless of how deeply the helper type is nested
inside other types, and regardless of the visibility section it sits in. What
does bound it is the **interface/implementation split**.

This is the same shape as the nested-enum element-name injection rule
([ch.02](02-fundamental-types.md) §2.2.4) — nesting namespaces the TYPE NAME
only (see [ch.11](11-classes.md) §11.4.1); the EFFECT lands at the enclosing
scope. Two different constructs, one law.

**Example**

```pascal
unit A;
interface
type
  TMatrix = record m11: Single; end;

  TOwner = class
  strict private                                  // visibility is irrelevant
    type
      TMatrixHelper = record helper for TMatrix
        const Identity = 42;
      end;
  end;

procedure P;

implementation

procedure P;
var
  M: TMatrix;
begin
  Writeln(TMatrix.Identity);   // OK — active at unit scope, and `TOwner` is
end;                           // never named (its own name is strict private)

end.
```

**Semantics & parsing notes**

- ⚠️ *Nesting does not confine a helper.* A helper declared as a nested type
  inside another class is active throughout the enclosing scope, exactly as if
  it had been declared there directly. Only its own TYPE NAME is namespaced
  (`TOwner.TMatrixHelper`) — and nothing ever has to spell that name for the
  helper to apply.
- ⚠️ *Visibility does not gate activation.* A `strict private` nested helper
  still applies outside its owner class. Visibility governs access to the
  helper type's NAME, never whether the helper is in scope for its target
  type. A resolver keyed on member visibility will wrongly reject these.
- The **interface/implementation** split is the real boundary, same as for any
  other declaration (ch.01 §1.2.1): a helper declared in the interface section
  is active in every unit that `uses` this one; a helper declared in (or nested
  inside a type declared in) the **implementation** section stays unit-local,
  and a cross-unit use of its members is a genuine `E2003`.
- Consequently the "active helper" set at a source position is: every helper
  for that type declared in this unit, plus every interface-section helper of
  every used unit — resolved down to ONE by §15.3.3's last-in-scope rule.
- dcc-verified (dcc32 37.0 / Delphi 13.1), each point separately: a nested
  interface-section helper resolves both bare inside the extended type's own
  methods and qualified as `T.Member` at unit level; the same nested helper
  applies in a second unit that merely `uses` the first, never naming the
  outer class; a `strict private` helper nested in an implementation-section
  class still applies outside that class in its own unit; the same
  implementation-section helper produces a real `E2003` when used from another
  unit; and two helpers for one type resolve to the last declared (§15.3.3).
- *AST:* no shape change — a nested `HelperType` is an ordinary nested
  `TypeDecl`. The rule is purely a name-resolution one: the collector must
  inject helper members at the nearest enclosing non-structured scope, not at
  the scope the declaration lexically sits in.
