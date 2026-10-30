# 14 — Interfaces

Interface types, GUIDs, the reference-counting lifetime model, weak/unsafe
references, delegation via `implements`, and method-resolution clauses.

Builds on [ch.11](11-classes.md)/[ch.12](12-inheritance-polymorphism.md).
Memory/ref-count detail → [ch.20](20-memory-management.md).

## Chapter grammar umbrella

```ebnf
InterfaceType = ( "interface" | "dispinterface" ) [ "(" Ancestor ")" ]
                  [ GUID ]
                  { InterfaceMember }
                "end" ;
GUID          = "[" ConstExpr "]" ;                (* '{xxxxxxxx-....}' or a string const *)
InterfaceMember = MethodHeading ";" { MethodDirective }
                | PropertyDecl ;                   (* interface properties: accessors only *)
```

---

## 14.1 Interface declaration

### 14.1.1 Interface types & GUIDs

| | |
|---|---|
| **Introduced** | Delphi 3 (1997) |
| **Deprecated** | — |
| **Status** | ✅ Current |

An `interface` declares a contract of methods/properties with no implementation
or fields. An optional GUID identifies it (required for `QueryInterface`/`as`).

**Example**

```pascal
type
  IReader = interface
    ['{4F2E9A10-1234-5678-9ABC-DEF012345678}']
    function Read(out B: TBytes): Integer;
    property Position: Int64 read GetPos write SetPos;
  end;
```

**Semantics & parsing notes**

- ⚠️ *GUID grammar:* `[ '{...}' ]` — a bracketed constant expression, immediately
  after the (optional) ancestor. An attribute group (also `[...]`, ch.19) may
  stand in the same place; which one a bracket is, is decided by what it
  names - see the lookup rule below, not by the syntax.
- ⚠️ *The GUID need not be a literal* — a **named string constant** is accepted
  too: with `const SID_IFoo = '{...}';` in scope, `IFoo = interface [SID_IFoo]`
  compiles (dcc64-verified 2026-08). The production is `"[" ConstExpr "]"`,
  constrained semantically to a compile-time string in GUID format.
- ⚠️ *Any constant expression, ONE of them* (dcc64 37.0, probed 2026-10-01): a
  qualified constant `[U.SID]`, a concatenation `['{...-' + '...}']`,
  `[SID + '']`, a parenthesized `[(SID)]` and `[(TBox.K)]` (a record's
  constant) all compile and are the GUID (RTTI, probed 2026-10-14);
  `['{...}', 'x']` is E2029; a string not in GUID format is `E2204 Improper
  GUID syntax`. A typed `TGUID` constant compiles but is NOT the GUID: `[GT]`
  with `const GT: TGUID = '{...}'` is `W1074 Unknown custom attribute: 'GT'`
  and the interface's GUID is null (a correction: the 2026-10-01 probe
  checked only that it compiles).
- ⚠️ *Its names are looked up AROUND the type, not in it* (dcc64 37.0,
  probed 2026-10-12): with a unit `const S = '{...}'` and `IFoo = interface
  [S] function S: Integer; end`, the GUID is the constant's - the
  interface's own member `S` does not hide it - and an unknown name is
  `E2003` (`[NoSuchX]`). A rename of the constant must rewrite the clause.
- ⚠️ *GUID or attribute is decided by TYPE, in the first bracket only:* `[X]`
  right after the head (and ancestor) is the GUID when X is a string or TGUID
  constant, and the first member's attribute group when X is an attribute
  class (`interface [TAttr] procedure P;` - RTTI: no GUID, P has the
  attribute). Any LATER bracket is an attribute group, a string included: a
  second `['{...}']` is `W1074 Unknown custom attribute` and dropped, and so
  is a GUID written after an attribute group (the interface then has no
  GUID). A parser cannot decide `[X]` from syntax alone.
- ⚠️ *A name there is looked up as an attribute's* (dcc64 37.0, RTTI-probed
  2026-10-14, PasTree local/probe-b56 G1, G2): `[X]` and `[U.X]` take the
  `XAttribute` suffix first, through the whole search order (§19.3.2), the
  bare `X` only when that finds nothing, and what is found decides:
  - an attribute class - the first member's attribute, no GUID: `[Mark]`,
    `[MarkAttribute]`, `[U.Mk]` (U's MkAttribute). The suffix wins over a
    nearer constant: with `const S = '{...}'` and `SAttribute` in one unit
    `[S]` is the attribute, and so is `[Q]` with a local `const Q` and a used
    unit's `QAttribute`;
  - a true constant - the GUID (`[S]`, `[U.SID]`); a non-string one is
    `E2010 Incompatible types: 'Integer' and 'AnsiString'`;
  - another type - `E2010 Incompatible types: 'TBox' and
    'TCustomAttribute'`, and only the first segment decides, as in an
    attribute: `[TBox.K]` is that E2010 while `[(TBox.K)]` is the GUID;
  - a typed constant or a variable - `W1074`, dropped, no GUID (`[GT]`,
    `[System.SysUtils.EmptyStr]`);
  - nothing - `E2003`, and so for a name in a longer expression:
    `[Unknown1 + 1]` is E2003 and E2026, where an attribute group drops the
    same text with W1074 (§19.3.2).

  The items of a comma list are judged one by one, and a constant after a
  dropped item is still the GUID: `[GT, S]` is W1074 for GT and S's GUID.
- Interfaces have **no fields, no visibility sections, no method bodies**. All
  members are implicitly public. Properties list only accessor method names.
  A visibility word is `E2210 'private' directive not allowed in in interface
  type` (sic), a field `E2209 Field declarations not allowed in interface
  type`. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 I04.dpr).
- ⚠️ *What needs the GUID:* `Intf as INoGuid` and `Supports(X, INoGuid)` are
  `E2232 Interface 'INoGuid' has no interface identification` - a GUID-less
  interface can be reached only statically (assignment from an implementing
  class or a descendant interface) or by a hard cast `INoGuid(I)`, which
  compiles. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 I01.dpr, I01b.dpr).
- Default ancestor is `IInterface` (≡ `IUnknown`).
- *AST:* `InterfaceType { ancestor, guid?, members[] }`.

### 14.1.2 `dispinterface` (COM automation)

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — (COM-specific) |
| **Status** | 🧪 Platform-specific (Windows/COM) |

A dispatch interface for OLE Automation; members are dispatched by `dispid`.

**Semantics & parsing notes**

- `dispinterface` is a **reserved word**. Members carry `dispid ConstExpr`
  directives. Windows/COM only.

---

## 14.2 Implementing interfaces

### 14.2.1 Classes implementing interfaces

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A class lists implemented interfaces after its ancestor and provides matching
methods.

**Example**

```pascal
type
  TFileReader = class(TInterfacedObject, IReader)
    function Read(out B: TBytes): Integer;
    // ...
  end;
```

**Semantics & parsing notes**

- ⚠️ *Multiple interfaces, one class ancestor:* the parenthesised list is
  `( ClassAncestor, IFace1, IFace2, … )` — this is how Object Pascal gets
  "multiple inheritance of interface". The resolver must match each interface
  method to a class method by name+signature.
- Implementing class usually descends from `TInterfacedObject` (provides
  ref-counted `IInterface`).
- ⚠️ *How a method is matched* (dcc64 37.0, 2026-10-30, spec audit g5,
  local/probe-park/spec-audit/g5 I08.dpr, I09.dpr): by name and an exactly
  equal parameter list - `procedure B(S: string)` does not implement `B(const
  S: string)`. Visibility does not matter (a `strict private` method
  implements, and runs through the interface); a public method INHERITED from
  an ancestor that lists no interface counts; a `class` method does not. Any
  interface method left unmatched is `E2291 Missing implementation of
  interface method IFoo.A`, one error per method, reported at the class.
  - *The mapping is fixed per class that LISTS the interface.* With
    `TBase = class(TInterfacedObject, IFoo)` and a non-virtual `A`, a
    descendant declaring its own `A` but not listing IFoo still answers
    `TBase.A` through the interface; `TRelist = class(TBase, IFoo)` with its
    own `A` answers `TRelist.A`. Re-listing re-runs the matching.
- ⚠️ *An interface's ANCESTOR is not implemented unless it is listed too.*
  With `IDer = interface(IBase)` and `TOnlyDer = class(TInterfacedObject,
  IDer)`, a `TOnlyDer` value is not assignable to `IBase` (`E2010
  Incompatible types: 'IBase' and 'TOnlyDer'`), and at run time
  `Supports(D, IBase)` is False and `D as IBase` raises `EIntfCastError` even
  though `D: IDer` assigns statically to an `IBase` (interface to ancestor
  interface needs no QueryInterface). Listing `IDer, IBase` makes all three
  work. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 I02.dpr).
- *AST:* `interfaces[]` on the `ClassType`.

### 14.2.2 Method resolution clauses

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Maps an interface method to a differently-named class method, resolving name
clashes between multiple interfaces.

**Grammar**

```ebnf
MethodResolution = "procedure" InterfaceName [ TypeArgs ] "." Method "=" ClassMethod ";"
                 | "function"  InterfaceName [ TypeArgs ] "." Method "=" ClassMethod ";" ;
```

**Example**

```pascal
type
  TFoo = class(TInterfacedObject, IReader, IWriter)
    function IReader.Read = ReadImpl;
    function IWriter.Read = WriteRead;   // resolve clashing 'Read'
  end;
```

**Semantics & parsing notes**

- The `IFace.Method = ClassMethod` form appears in the class member list — parse it
  distinctly from a normal method declaration (it has `=` and dotted LHS).
- ⚠️ *Generic interface on the LHS:* the interface name may carry **type
  arguments** — dcc-verified in the RTL: `function IEnumerable<string>.
  GetEnumerator = GetEnumeratorStr;` (System.IOUtils), `IEnumerable<T>` variants
  (System.JSON). The `<...>` here is a type-argument REFERENCE to the implemented
  interface — semantic analysis must NOT treat it as generic parameter
  declarations the way it does for a qualified method implementation header
  (`TList<T>.Add` in 16 §16.3); doing so declares bogus symbols (`string`, `T`)
  into the class scope and yields false E2004.
  The interface name is matched WITH its argument count, arity being part of
  the identity (16.1.2): beside `IGen = interface function GetA: Integer;
  end` and `IGen<T> = interface(IGen) function GetT: T; end`, `function
  IGen<Integer>.GetT = DoGetT;` names the generic one, `function IGen.GetA =
  DoGetA;` the plain one.
- *The interface method is a member of that interface, inherited ones
  included*: `function IFoo.GetB = DoGetB;` with GetB declared in IFoo's
  ancestor IBase, and `function IFoo._AddRef = MyAddRef;` (IInterface's), are
  valid - and the interface may live in another unit than the class. dcc32 /
  dcc64 37.0, 2026-10-13 (PasTree audit B41: A4-04, E3-11; local/probe-b41
  mr).
  The interface must be in the class's OWN interface list: `procedure
  IBar.B = A1;` in a class listing only IFoo is `E2003 Undeclared
  identifier: 'IBar'` although IBar is a declared type - and so is
  `procedure IFoo.A = X;` in a descendant of a class that lists IFoo, when
  the descendant does not list it again; an interface method that does not
  exist is `E2003` on its name (`IFoo.Zzz`). dcc64 37.0, 2026-10-30 (spec
  audit g5, local/probe-park/spec-audit/g5 I03.dpr, I03b.dpr).
- ⚠️ *The class-method name is NOT always checked* (dcc-probed, dcc32 37.0,
  2026-09-16; UNDOCUMENTED): when an inherited member already satisfies the
  interface method AND that member is declared in an ancestor class which
  itself declares an interface list - any interface, related or not -
  `function IFoo.GetX = NoSuchMethod;` compiles and the clause is silently
  ignored (the inherited method answers the interface call; a real target in
  the same position IS honoured). Plain classes between that ancestor and the
  declaring class do not matter. The same clause is `E2003 Undeclared
  identifier` when the satisfying member is declared in an interface-less
  ancestor (even with an interface-declaring class further up), or when
  nothing implements the method. Real code ships the ignored form (a third-party layout unit, two
  clauses naming a method that exists nowhere). A tolerant analyzer should not
  report the name under an interface-declaring ancestor.

---

## 14.3 Reference counting & lifetime

### 14.3.1 `IInterface` reference counting

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Interface references are **managed**: assignment calls `_AddRef`/`_Release`; the
object self-destructs when the count hits zero (when implemented via
`TInterfacedObject`).

**Semantics & parsing notes**

- ⚠️ *Mixing object and interface references to the same instance is dangerous* —
  the ref count can drop to zero while an object reference still exists. A known
  hazard the type-checker can warn about, not parse.
- Interface variables are auto-released at scope exit (compiler inserts
  `_Release`).
- ⚠️ *Reference counting is per-class, not a fixed language guarantee* —
  `_AddRef`/`_Release` are ordinary (virtual) methods, so a class's ancestor
  decides whether they count anything at all. `TNoRefCountObject`
  (`System.SysUtils`, Delphi 11 — the older name `TSingletonImplementation`,
  `System.Generics.Defaults`, is now just an alias for the same code) implements
  `IInterface` with `_AddRef`/`_Release` that do nothing and return `-1`.
  `TComponent` does the same, because it already has its own ownership-based
  memory model. (dcc-verified, dcc32 37.0: a `TComponent`-descended class
  implementing a custom interface, assigned to an interface variable that then
  goes out of scope/is set to `nil`, is **not** destroyed — `ClassName` is still
  readable on it afterward and it must be freed manually; a `TInterfacedObject`-
  descended equivalent under the identical test IS destroyed at that point.) A
  resolver/lifetime-checker cannot assume "reachable only through an interface
  variable" implies "will be freed by refcounting" — it depends on which
  `_AddRef`/`_Release` the object's ancestor chain provides.
- Detailed model → [ch.20 §20.x](20-memory-management.md).

### 14.3.2 Weak & unsafe interface references

| | |
|---|---|
| **Introduced** | `[weak]`/`[unsafe]` Delphi XE4 (mobile ARC); **all compilers 10.1 Berlin** |
| **Deprecated** | — |
| **Status** | ✅ Current |

`[weak]` and `[unsafe]` attributes mark interface fields that do not
participate in (strong) reference counting - breaking reference cycles. (On
object references they mattered only under mobile ARC, XE4 - 10.3; see below.)

**Example**

```pascal
type
  TNode = class(TInterfacedObject, INode)
    [weak] FParent: INode;     // does not keep parent alive
  end;
```

**Semantics & parsing notes**

- ⚠️ These are **attributes** (`[weak]`/`[unsafe]`, ch.19 syntax) applied to a
  field, not directives. On an INTERFACE field `[weak]` nils the reference when
  the target dies and is thread-safe; `[unsafe]` is a raw non-counted reference
  (faster, dangling-risk). On an OBJECT reference `[weak]` zeroing existed only
  under mobile ARC (XE4 - 10.3, removed in 10.4 Sydney, §20.5.1). Every
  current compiler, the mobile ones included, accepts `[weak]` there but does
  nothing with it: the field is not zeroed when the object is freed (`W.O =
  nil` is FALSE after `O.Free`). The full rules - placement,
  `E2584`, `HasWeakRef` - are owned by §20.6.1. dcc64 37.0, 2026-10-30 (spec
  audit v, local/probe-park/spec-audit/v V1.dpr; g6 W01.dpr).
- The parser treats them as attributes on the field; the semantic layer changes
  ref-count codegen.

---

## 14.4 Delegation with `implements`

### 14.4.1 The `implements` directive

| | |
|---|---|
| **Introduced** | Delphi 3/4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A property can delegate an entire interface's implementation to an inner object
or interface field.

**Example**

```pascal
type
  TOuter = class(TInterfacedObject, IReader)
  private
    FInner: IReader;
    property Inner: IReader read FInner implements IReader;  // delegate
  end;
```

**Semantics & parsing notes**

- `implements IFace` on a property means the property's value supplies that
  interface's methods — the class need not reimplement them. `implements` is a
  **directive** (B.4.2).
- ⚠️ *The delegate's usual base class is `TAggregatedObject`* (`System.pas`),
  not `TInterfacedObject` — it is purpose-built for this role and has a
  different reference-count/query story: its `_AddRef`/`_Release` do not count
  anything of its own, and its `QueryInterface` reflects to a
  `[Unsafe]`-referenced `Controller: IInterface`, supplied via its
  `constructor Create(const Controller: IInterface)` — because, per its own
  doc comment in `System.pas`, an aggregated object "must have the same
  lifetime as their controller." (dcc-verified, dcc32 37.0: an outer
  `TInterfacedObject`-descended class constructs its delegate as
  `TJumperImpl.Create(Self)` — `Self` satisfies the required `Controller`
  parameter; calling a method of the delegated interface through the outer
  object's interface property correctly runs the delegate's code, and
  `(OuterIntf as TObject) = Outer` — the interface obtained from the OUTER
  class resolves back to the outer object, not the aggregate.) Consistent with
  having no ref-count of its own, the outer class still frees the delegate
  itself in its destructor through a plain object field (never an interface
  field for the delegate).
- ⚠️ *What an `implements` property must be* (dcc64 37.0, 2026-10-30, spec
  audit g5, local/probe-park/spec-audit/g5 I06.dpr, I06b.dpr): the
  interface must be in the class's list (`E2265 Interface 'IBar' not
  mentioned in interface list`); the property must be readable (`write`
  only: `E2261 Implements clause only allowed for readable property`); an
  interface-typed property must be of that interface (`IBar` property
  `implements IFoo`: `E2010 Incompatible types: 'IFoo' and 'IBar'`); one
  interface takes one `implements` (`E2257 Duplicate implements clause for
  interface 'IFoo'`). A CLASS-typed property need not implement the
  interface: `property P: TPlain read FP implements IFoo` with `TPlain =
  class procedure A; end` (no interface list) compiles, and `F.A` runs
  `TPlain.A` - the methods are matched by name in the property's class.
- ⚠️ *Class-typed and interface-typed delegation differ when the outer class
  has a method of its own* (same date, I07.dpr): with IFoo = `A`, `B` and the
  outer class declaring `B`, delegation through a CLASS-typed property
  (`Inner: TInner ... implements IFoo`) answers `F.B` with the OUTER `B` and
  `F.A` with the delegate's - the outer class's own methods win, the delegate
  fills the rest; through an INTERFACE-typed property (`Inner: IFoo ...
  implements IFoo`) every call goes to the delegate and the outer `B` is
  ignored. A resolver mapping interface methods to bodies must apply the
  outer class's own matches first only in the class-typed case.
- *AST:* `PropertyDecl { …, implements: [IFace] }`.

---

## 14.5 Interface properties & polymorphism

### 14.5.1 Interface references as polymorphic handles

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Any class implementing an interface can be referenced through it; `as`/`is` work
across interface and class types (via `QueryInterface`).

**Semantics & parsing notes**

- `Obj as IFace` performs `QueryInterface` (may raise); `Supports(Obj, IFace)`
  (RTL) is the non-raising test.
- ⚠️ *`is` never takes an INTERFACE on its right* - only a class (the
  object/interface-to-object forms, next bullet) - given `F: IFoo` backed by an
  object that also implements `IBar`, `F is IBar` is a **compile-time error**,
  not a runtime `False` (dcc-verified, dcc32 37.0: `E2015 Operator not
  applicable to this operand type`). The same holds with an OBJECT on the
  left: `X is IG` is E2015 for `X: TImplG` whose class lists IG, and for a
  `TInterfacedObject` or `TObject` variable alike (a correction: this bullet
  used to allow object-to-interface `is`; dcc64 37.0, 2026-10-30, spec audit
  g5, local/probe-park/spec-audit/g5 I05.dpr, I01c.dpr). The test is
  `Supports(X, IG)`.
  `Obj as IG` compiles only when `Obj`'s STATIC class implements some
  interface (it then has `QueryInterface`): `IO as IG` with `IO:
  TInterfacedObject` and `C as IG` with `C: TComponent` compile, while a
  `TObject` variable or a class with no interface in its chain is E2015 (same
  probes, I01b.dpr). `F as IBar`, by contrast, DOES work
  interface-to-interface — it compiles and performs `QueryInterface` under the
  hood (dcc-verified: `(F as IBar).Bar` runs correctly), and so does
  `Supports(F, IBar, B)`, the RTL's recommended non-raising equivalent
  (dcc-verified). A resolver must reject `is` whenever its right operand is an
  interface type while still allowing `as` (and `Supports`) for the identical
  pair.
- ⚠️ *Interface references cast back to their underlying object* — for an
  interface backed by a genuine Object-Pascal object (never a COM server, which
  has no such object to recover), all three reverse-cast forms work and refer
  to the SAME object: `IntfVar is TMyClass`, `IntfVar as TMyClass` (raises on
  failure), and the hard cast `TMyClass(IntfVar)` (yields `nil` on failure).
  (dcc-verified, dcc32 37.0: pointer-identity check `TTestImpl(Intf) = Original`
  where `Original := TObject(Intf)`; a method that exists only on the concrete
  class, not the interface, is reachable through both `TTestImpl(Intf)` and
  `Intf as TTestImpl`.) The target need not be the object's exact runtime
  class — casting to a BASE class of the actual class also succeeds, following
  ordinary class-compatibility rules (dcc-verified: `TTestBase(Intf)` succeeds
  where `TTestBase` is an ancestor of the actual `TTestImpl`, and
  `AsBase is TTestImpl` is `True` on the result).
- Interface properties expose only accessor names (no storage), resolved on the
  implementing object.
