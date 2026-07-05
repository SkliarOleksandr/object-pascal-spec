# 11 — Classes & Objects

The reference-type class model: declaration, fields, methods, visibility,
construction/destruction, `Self`, and nested members. Inheritance and
polymorphism are split into [ch.12](12-inheritance-polymorphism.md); properties
into [ch.13](13-properties-events.md); class-level (static) members and helpers
into [ch.15](15-class-mechanics-helpers.md).

Shared productions (`TypeRef`, `Ident`) → [Appendix B](B-lexical-grammar.md);
method bodies → [ch.06](06-routines.md).

## Chapter grammar umbrella

```ebnf
ClassType   = "class" [ Abstract ] [ "(" Ancestor [ "," InterfaceList ] ")" ]
                { ClassSection }
              "end"
            | "class" [ "(" Ancestor ")" ] ;          (* forward / no body *)
ClassSection = Visibility { ClassMember } ;
Visibility   = [ "strict" ] ( "private" | "protected" | "public" | "published" ) ;
ClassMember  = FieldDecl | MethodDecl | PropertyDecl
             | ConstDecl | TypeDecl | ClassVarDecl ;
MethodDecl   = ( "procedure" | "function" | "constructor" | "destructor" )
               Ident [ "(" FormalParams ")" ] [ ":" ResultType ] ";" { MethodDirective } ;
```

---

## 11.1 Class declaration

### 11.1.1 Class types

| | |
|---|---|
| **Introduced** | Delphi 1 (1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `class` is a heap-allocated **reference type**: variables hold references, not
values. All classes descend from `TObject`.

**Example**

```pascal
type
  TCustomer = class
    FName: string;
    procedure Greet;
  end;
```

**Semantics & parsing notes**

- ⚠️ *Reference semantics:* assignment copies the **reference**, not the object;
  two variables can alias one instance. Contrast records (ch.09, value types).
- A class declared `class;` or `class(TParent);` with no member block is a
  **forward declaration** (completed later in the same type section) — used for
  mutually referencing classes.
- *AST:* `ClassType { ancestor?, interfaces[], sections[], isForward }`.

### 11.1.2 Fields

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Instance data members. Each instance gets its own copy; fields are zero-
initialized on construction.

**Semantics & parsing notes**

- ⚠️ *Ordering rule (modern Delphi):* fields should precede methods within a
  visibility section; the compiler is lenient but the canonical grammar lists
  `FieldList` first. Class fields (`class var`) → ch.15.
- *AST:* `FieldDecl { names[], type, visibility }`.

### 11.1.3 Methods

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Procedures/functions bound to the class; the body is defined in the
`implementation` section, qualified by the class name.

**Example**

```pascal
procedure TCustomer.Greet;
begin
  Writeln('Hello, ', FName);
end;
```

**Semantics & parsing notes**

- The in-class declaration is a header; the body uses `TClass.Method` qualification
  in `implementation`. The resolver pairs them.
- Method directives (`virtual`, `override`, `overload`, `abstract`, `reintroduce`,
  `static`, `dynamic`, `message`, `inline`, …) follow the header — see ch.12/15.
- *AST:* `MethodDecl` header + linked body `RoutineDecl`.

---

## 11.2 Visibility

### 11.2.1 `private` / `protected` / `public` / `published` (+ `strict`)

| | |
|---|---|
| **Introduced** | basic D1; **`strict`** Delphi 2006; `published` D1 (RTTI) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Visibility sections control member access. `strict` tightens `private`/
`protected` to true per-class encapsulation.

**Example**

```pascal
type
  TFoo = class
  strict private
    FSecret: Integer;       // not even same-unit code can touch it
  private
    FInternal: Integer;     // same-unit "friend" access
  protected
    procedure DoIt; virtual;
  public
    constructor Create;
  published
    property Value: Integer read FInternal;   // RTTI-visible
  end;
```

**Semantics & parsing notes**

- ⚠️ *Unit-level "friend" rule:* plain `private`/`protected` members are visible to
  **all code in the same unit** (Object Pascal has no separate `friend`). `strict
  private`/`strict protected` (2006+) restrict to the class / class+descendants.
  The resolver enforces this — it needs the declaring unit on each member.
- ⚠️ *`published` requires RTTI:* `published` members generate runtime type info and
  are restricted to classes compiled with `{$M+}` (or descending from `TPersistent`).
  Allowed member types are constrained (e.g. method pointers for events, ordinal/
  string/class properties). See ch.13/19.
- `strict` is a **directive** (B.4.2) preceding `private`/`protected`.
- ⚠️ *Legacy fifth visibility — `automated`:* old OLE-Automation code may declare an
  `automated` section (like `published` but generating Automation dispatch info).
  Unused in the D13 sources, but still accepted by the compiler — the parser should
  accept it as a visibility keyword (it is a directive, B.4.2) and may flag it
  legacy.
- ⚠️ *Visibility words vs. member names (dcc-verified):* at **member-start**
  position, a bare `private`/`protected`/`public`/`published` is ALWAYS a
  section marker — `protected: string;` does not declare a field (E2029). But
  in identifier-list **continuation** the same words are ordinary names:
  `on, protected, sealed, abstract: string;` compiles. Every NON-visibility
  directive word is a valid field name even at list start
  (`default, index: Integer;` compiles). Disambiguation is purely positional.
- *AST:* tag each member with `{ visibility, strict }`.

---

## 11.3 Construction, destruction & `Self`

### 11.3.1 Constructors

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`constructor` methods allocate (when called on a class reference) and initialize
an instance. Conventionally named `Create`; multiple overloads allowed.

**Example**

```pascal
constructor TCustomer.Create(const AName: string);
begin
  inherited Create;
  FName := AName;
end;
```

**Semantics & parsing notes**

- ⚠️ *Dual nature:* called on a **class** (`TCustomer.Create`) it allocates + runs
  the body + returns the instance; called on an **instance** via `inherited` it
  only runs the body (no allocation). The codegen differs; the parser just records
  `kind = constructor`.
- Fields are zero-filled **before** the constructor body runs. If the constructor
  raises, the **destructor is called automatically** on the partially built object
  (ch.18 interaction).
- *AST:* `MethodDecl { kind: constructor, … }`.

### 11.3.2 Destructors & `Free`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`destructor Destroy` (always `override`) finalizes and deallocates. Code calls
`Free` (a `TObject` method that nil-checks then calls `Destroy`).

**Example**

```pascal
destructor TCustomer.Destroy;
begin
  FOwned.Free;
  inherited;          // inherited Destroy
end;
```

**Semantics & parsing notes**

- ⚠️ *Manual lifetime* (post-10.4, on all platforms): the creator is responsible for
  `Free`. The historic mobile **ARC** model (XE4–10.3) auto-freed — see
  [ch.20](20-memory-management.md). Treat ARC as historical 🧪.
- `Destroy` should be declared `override`; `Free` is not virtual (it dispatches to
  `Destroy`).
- *AST:* `MethodDecl { kind: destructor, … }`.

### 11.3.3 The `Self` identifier

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Inside an instance method, `Self` is the current instance; inside a class method,
`Self` is the class reference (ch.15).

**Semantics & parsing notes**

- `Self` is an **implicit parameter**, not a reserved word — but the compiler
  injects it. Unqualified member names resolve against `Self` implicitly.
- *AST:* member accesses may be normalised to `Self.member` during resolution.

---

## 11.4 Nested types & constants

### 11.4.1 Nested type/const declarations

| | |
|---|---|
| **Introduced** | Delphi 2005/2006 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A class may declare nested types, constants, and class vars, scoped to and
qualified by the class.

**Example**

```pascal
type
  TGrid = class
  public
    type TCell = record Row, Col: Integer; end;
    const MaxRows = 100;
  end;

var C: TGrid.TCell;     // qualified access
```

**Semantics & parsing notes**

- Nested types are referenced `OuterClass.NestedType`; they obey the enclosing
  class's visibility section.
- *AST:* nested `TypeDecl`/`ConstDecl` as class members.

---

## 11.5 Legacy `object` types

| | |
|---|---|
| **Introduced** | Turbo Pascal / D1 |
| **Deprecated** | — (superseded by `class`) |
| **Status** | ⚠️ Legacy |

The old Turbo-Pascal `object` type: a **value-type** OOP construct predating
`class`.

**Grammar**

```ebnf
ObjectType = "object" [ "(" Ancestor ")" ] { ClassSection } "end" ;
```

**Semantics & parsing notes**

- `object` is a **reserved word**. Unlike `class`, `object` instances are value
  types (stack/inline), support inheritance and virtual methods but not interfaces/
  RTTI in the modern sense. Retained for backward compatibility; avoid in new code.
- Distinguish from `of object` (method-pointer suffix, ch.06) — different context.
- *AST:* `ObjectType { … }` (value-type OOP node).
