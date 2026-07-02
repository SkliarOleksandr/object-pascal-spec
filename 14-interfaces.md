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
MethodResolution = "procedure" InterfaceName "." Method "=" ClassMethod ";"
                 | "function"  InterfaceName "." Method "=" ClassMethod ";" ;
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
- Interface properties expose only accessor names (no storage), resolved on the
  implementing object.
