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
| **Introduced** | Delphi 2009/2010 |
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
- `TObject`'s `ClassType`/`ClassName`/`InheritsFrom` operate on metaclasses.
- *AST:* `ClassRefType { baseClass }`.

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
- *AST:* `HelperType { kind: class, forType, members[] }`.

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
- Same no-instance-fields restriction as class helpers.

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
