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
GUID          = "[" StringLiteral "]" ;            (* '{xxxxxxxx-....}' *)
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

- ⚠️ *GUID grammar:* `[ '{...}' ]` — a string literal in brackets, immediately
  after the (optional) ancestor. Don't confuse with an attribute (also `[...]`,
  ch.19) — context (interface body, GUID format) disambiguates.
- Interfaces have **no fields, no visibility sections, no method bodies**. All
  members are implicitly public. Properties list only accessor method names.
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

`[weak]` and `[unsafe]` attributes mark interface (or object) fields that do not
participate in (strong) reference counting — breaking reference cycles.

**Example**

```pascal
type
  TNode = class(TInterfacedObject, INode)
    [weak] FParent: INode;     // does not keep parent alive
  end;
```

**Semantics & parsing notes**

- ⚠️ These are **attributes** (`[weak]`/`[unsafe]`, ch.19 syntax) applied to a
  field, not directives. `[weak]` nils the reference when the target dies and is
  thread-safe; `[unsafe]` is a raw non-counted reference (faster, dangling-risk).
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
- ⚠️ *`is` cannot test interface-to-interface support — only object-to-interface
  (and object/interface-to-object, next bullet)* — given `F: IFoo` backed by an
  object that also implements `IBar`, `F is IBar` is a **compile-time error**,
  not a runtime `False` (dcc-verified, dcc32 37.0: `E2015 Operator not
  applicable to this operand type`). `F as IBar`, by contrast, DOES work
  interface-to-interface — it compiles and performs `QueryInterface` under the
  hood (dcc-verified: `(F as IBar).Bar` runs correctly), and so does
  `Supports(F, IBar, B)`, the RTL's recommended non-raising equivalent
  (dcc-verified). A resolver must reject `is` when BOTH operands are interface
  types while still allowing `as` (and `Supports`) for the identical pair.
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
