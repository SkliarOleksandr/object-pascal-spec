# 13 — Properties & Events

Properties (the encapsulation mechanism), their array/indexed/default forms,
streaming specifiers, and the method-pointer-based event model.

Builds on [ch.11](11-classes.md); method pointers recap [ch.06
§6.6](06-routines.md#66-procedural-types).

## Chapter grammar umbrella

```ebnf
PropertyDecl = "property" Ident [ PropArray ] [ ":" TypeRef ]
               [ "index" ConstExpr ]
               [ "read" Designator ] [ "write" Designator ]
               [ "stored" ( Ident | ConstExpr ) ]
               [ "default" ConstExpr | "nodefault" ]
               [ "implements" IdentList ]
               [ "default" ] ";"                         (* trailing 'default' = default array prop *)
               { PropDirective } ;
PropArray    = "[" FormalParams "]" ;
```

---

## 13.1 Properties

### 13.1.1 Property declaration

| | |
|---|---|
| **Introduced** | Delphi 1 (1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A property exposes controlled access to state via `read`/`write` specifiers that
name a field, method, or (for class props) class member.

**Example**

```pascal
type
  TFoo = class
  private
    FValue: Integer;
    procedure SetValue(V: Integer);
  public
    property Value: Integer read FValue write SetValue;
  end;
```

**Semantics & parsing notes**

- ⚠️ *Properties are not storage.* `read`/`write` map to a field (direct) or a
  method (getter/setter). A read on a method-backed property lowers to a call; a
  write to a method lowers to a setter call; field-backed lowers to a field access.
  The resolver must classify each specifier and the parser must keep them.
- Read-only (no `write`) and write-only (no `read`) are legal.
- `read`/`write`/`index`/`stored`/`default`/`nodefault`/`implements` are
  **directives** (B.4.2), keyword only inside a property declaration — elsewhere
  valid identifiers.
- ⚠️ *A property may be NAMED by any directive word* — including the specifier
  words themselves: `property default: string read FData;`,
  `property on: string ...`, `property index: ...` all compile (dcc-verified).
  Only the parse position distinguishes the name from a specifier.
- *AST:* `PropertyDecl { name, type, reader?, writer?, index?, … }`.

### 13.1.2 Array properties

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A property indexed by one or more parameters; access uses `[]`.

**Example**

```pascal
property Items[Index: Integer]: TItem read GetItem write SetItem;
```

**Semantics & parsing notes**

- ⚠️ The getter/setter take the index parameter(s); `Obj.Items[3]` lowers to
  `Obj.GetItem(3)` (read) / `Obj.SetItem(3, v)` (write). Array properties **cannot**
  be field-backed — `read`/`write` must be methods.
- The `[ ... ]` here is a parameter list in the *declaration*, but indexing at the
  *use* site (shares `Selector` `[ExprList]`, §B.8).

### 13.1.3 Indexed properties (`index` directive)

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Multiple properties can share one getter/setter, distinguished by an `index`
constant passed to the accessor.

**Example**

```pascal
property Left:   Integer index 0 read GetCoord write SetCoord;
property Top:    Integer index 1 read GetCoord write SetCoord;
// GetCoord(Index: Integer): Integer;  SetCoord(Index, Value: Integer);
```

**Semantics & parsing notes**

- The `index N` constant is prepended to the accessor call. The accessor signature
  must accept the leading index parameter. Distinct from *array* properties.

### 13.1.4 Default array property

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

An array property may be marked `default`, allowing `Obj[i]` shorthand
for `Obj.Prop[i]`.

**Example**

```pascal
property Items[I: Integer]: TItem read GetItem; default;
// usage: MyList[0]  ==  MyList.Items[0]
```

**Semantics & parsing notes**

- ⚠️ *Trailing `default`* (after the specifiers) marks the **default array
  property** — distinct from the `default ConstExpr` *value* specifier (13.1.5).
  Disambiguate by position/operand: bare `default;` = default array property;
  `default 0;` = default value. The parser must not conflate them.
- `Obj[i]` on a class with a default array property lowers to the property access.
- ⚠️ *Overloaded default array properties:* a class may declare **several** array
  properties under the SAME name, each marked `default`, differing by index
  signature — indexing picks the overload by index type. dcc-verified in the RTL:
  `System.RegularExpressions.TGroupCollection` has `property Item[const Index:
  Integer]` and `property Item[const Index: string]`, both `default`. A resolver
  must treat a same-name property redeclaration as an overload, not an E2004.
  (An earlier revision of this section said "one array property per class may be
  marked default" — that was wrong.)
- *AST:* `isDefaultArrayProp: true`.

### 13.1.5 Streaming specifiers: `default`, `nodefault`, `stored`

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

For `published` properties, these control component streaming (whether a value is
written to the .dfm).

**Example**

```pascal
property Width: Integer read FWidth write SetWidth default 100;
property Caption: string read FCaption write SetCaption stored FHasCaption;
```

**Semantics & parsing notes**

- ⚠️ `default ConstExpr` here is a **streaming hint** (don't store if equal to this
  value) — it does **not** initialize the field. Common misconception; note it.
- `stored` takes a boolean constant/field/method controlling whether to persist.
- Only meaningful for `published` properties with RTTI.

---

## 13.2 `published` properties & RTTI

### 13.2.1 Published properties

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Properties in a `published` section generate RTTI and appear in the Object
Inspector / streaming system.

**Semantics & parsing notes**

- Requires `{$M+}` or a `TPersistent` ancestor (ch.11 §11.2.1). Published property
  types are restricted (ordinal, string, set, class, method-pointer). The resolver
  enforces the allowed-type set.
- Full RTTI model → [ch.19](19-rtti-attributes.md).

---

## 13.3 Events

### 13.3.1 Method pointers & events

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Events are **properties of a method-pointer type** (`procedure(...) of object`),
enabling the delegation/event-handler model.

**Example**

```pascal
type
  TNotifyEvent = procedure(Sender: TObject) of object;
  TButton = class
  private
    FOnClick: TNotifyEvent;
  published
    property OnClick: TNotifyEvent read FOnClick write FOnClick;
  end;
```

**Semantics & parsing notes**

- An event is just a property whose type is a **method pointer** (`of object`,
  ch.06 §6.6). Assigning a handler stores a `(code, Self)` pair.
- Calling `FOnClick(Self)` after an `Assigned(FOnClick)` check is the standard
  fire pattern.
- Anonymous methods (`reference to`, ch.17) can also back events on newer designs,
  but the classic `of object` event is the RTTI/streaming-compatible form.
- *AST:* a `PropertyDecl` whose `type` is a method-pointer `ProceduralType`.
