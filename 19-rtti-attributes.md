# 19 — RTTI & Attributes

Run-time type information and custom **attributes** — the latter being the only
part with real *syntax* (the `[Attr(...)]` annotations the parser must handle and
not confuse with set/array constructors or GUIDs).

Shared productions (`TypeRef`, `Expression`) → [Appendix B](B-lexical-grammar.md).

## Chapter grammar umbrella

```ebnf
AttributeGroup = "[" Attribute { "," Attribute } "]" ;
Attribute      = TypeRef [ "(" [ ActualParams ] ")" ] ;   (* attribute class + ctor args *)
RttiDirective  = "{$RTTI" RttiVisibilitySpec "}" | "{$M" ("+"|"-") "}" ;
```

---

## 19.1 Classic RTTI

### 19.1.1 `TObject` RTTI & `{$M}` published info

| | |
|---|---|
| **Introduced** | Delphi 1; `{$M}` published RTTI D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Every object exposes `ClassType`, `ClassName`, `ClassParent`, `InheritsFrom`,
etc.; `published` members generate the classic RTTI used by streaming.

**Semantics & parsing notes**

- `{$M+}`/`{$TYPEINFO ON}` enables published RTTI for a class (and is implied by
  descending from `TPersistent`). Directive state at the class declaration governs
  what RTTI is emitted — track it.
- *Not parsing* beyond the directive; affects emitted metadata.

---

## 19.2 Extended RTTI

### 19.2.1 Extended RTTI & the `{$RTTI}` directive

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Rich run-time reflection over types, fields, methods, properties, and attributes
(the `System.Rtti` unit). The `{$RTTI}` directive controls which member
visibilities emit extended RTTI.

**Example**

```pascal
{$RTTI EXPLICIT METHODS([vcPublic, vcPublished]) FIELDS([vcPrivate..vcPublished])}
```

**Semantics & parsing notes**

- ⚠️ The `{$RTTI …}` directive has a **structured argument grammar**
  (`EXPLICIT`/`INHERIT`, then `METHODS`/`FIELDS`/`PROPERTIES` with visibility
  sets). The directive pre-pass must parse this rather than skip it, since it
  affects emitted metadata (and tools reading the AST may need it).
- Drives `TRttiContext`/`TRttiType` reflection; mostly an RTL concern.

---

## 19.3 Attributes

### 19.3.1 Attribute declarations

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A custom attribute is a class descending from `TCustomAttribute`; instances are
attached to declarations and read back via extended RTTI.

**Example**

```pascal
type
  TableAttribute = class(TCustomAttribute)
    constructor Create(const AName: string);
  end;
```

**Semantics & parsing notes**

- An attribute class is an ordinary class; only its **usage** has special syntax
  (19.3.2). By convention the `Attribute` suffix may be omitted at the use site
  (`[Table('X')]` finds `TableAttribute`).

### 19.3.2 Applying attributes

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Attributes are written in `[ ]` immediately **before** the declaration they
annotate (types, fields, methods, properties, parameters).

**Grammar**

```ebnf
AttributeGroup = "[" Attribute { "," Attribute } "]" ;
Attribute      = TypeRef [ "(" [ ActualParams ] ")" ] ;
```

**Example**

```pascal
type
  [Table('CUSTOMER')]
  TCustomer = class
    [Column('ID', [cpKey])]
    FId: Integer;
    procedure Save([Ref] const Data: TData);   // attribute on a parameter
  end;
```

**Semantics & parsing notes**

- ⚠️ **`[ … ]` is overloaded three ways** and the parser must disambiguate by
  **position**:
  1. *Before a declaration* → an **attribute group** (this section).
  2. *In an expression* → a **set / array constructor** (§B.9).
  3. *After an interface ancestor* containing a `'{…}'` string → a **GUID**
     (ch.14).
  Use the syntactic context (what precedes the `[`) to choose.
- The attribute's `TypeRef` is resolved to a `TCustomAttribute` descendant; the
  `( … )` arguments must match one of its constructors (evaluated to build the
  attribute instance in RTTI). The trailing `Attribute` suffix is optional.
- Attributes are only retained in binaries where extended RTTI is enabled
  (`{$RTTI}`); otherwise they parse but emit nothing.
- *AST:* attach `attributes: [ { type, args[] } ]` to the annotated declaration node.

### 19.3.3 Compiler-recognized ("magic") attributes

| | |
|---|---|
| **Introduced** | `[Ref]` ~XE2; `[weak]`/`[unsafe]`/`[volatile]` XE4 (mobile ARC), **all compilers 10.1 Berlin** |
| **Deprecated** | — |
| **Status** | ✅ Current |

A small set of attributes use the ordinary attribute *syntax* but are interpreted
by the **compiler itself** (they change codegen, not just RTTI metadata).

**Example**

```pascal
procedure Draw(const [Ref] R: TRect);        // parameter: force by-reference
type
  TObj = class
    [Volatile] FRefCount: Integer;           // field: volatile access semantics
    [weak]     FParent: INode;               // field: weak (non-counted, zeroed)
    [unsafe]   FOwner: IOwner;               // field: raw non-counted reference
  end;
```

**Semantics & parsing notes**

- ⚠️ *Same grammar, special resolution:* these parse exactly like custom attributes
  (19.3.2) — including on **formal parameters** (ch.06 grammar). The semantic
  layer must special-case the names:
  - `[Ref]` (on `const` params) — force by-reference passing (≡ `constref`).
  - `[Volatile]` (fields/vars) — every access is a real memory access; no register
    caching/reordering. The RTL marks e.g. `TObject.FRefCount` with it.
  - `[weak]` / `[unsafe]` (interface/object fields) — reference-counting opt-outs;
    full semantics in ch.14 §14.3.2 / ch.20 §20.6.
- Matching is by attribute-class identity in `System` (`VolatileAttribute` etc.),
  not raw string — but a lightweight parser may match by name.
- *Related toolchain attributes:* `[HPPGEN('C++ decl')]` (emit a custom C++Builder
  header declaration — hundreds of uses in `System.pas`/`System.Types.pas`) and
  `[Align(N)]` are likewise compiler-consumed. Ordinary attribute syntax; no
  grammar impact.
- *AST:* same attribute nodes; tag `magic: ref|volatile|weak|unsafe` after resolution.

---

## 19.4 `TValue` & reflective invocation

### 19.4.1 `TValue`

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`TValue` (RTL record) is a type-erased container holding a value of any type,
used by reflective property reads and method invocation.

**Semantics & parsing notes**

- Purely an RTL type (no syntax). Mentioned because reflective `Invoke`/`GetValue`
  APIs the parser's consumers may target return/accept `TValue`. Out of language
  scope otherwise.

### 19.4.2 Virtual method interceptors

| | |
|---|---|
| **Introduced** | Delphi 2010 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`TVirtualMethodInterceptor` (RTL) allows intercepting virtual calls at runtime via
RTTI — an RTL facility built on the metadata above, no language syntax.
