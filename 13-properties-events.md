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
- ⚠️ *A specifier may name an INHERITED accessor, including one declared in
  another unit* — `property StatusBar: WordBool index 403 read GetWordBoolProp
  write SetWordBoolProp` where both accessors belong to an ancestor
  (`System.Win.InternetExplorer`'s `TWebBrowser` over
  `System.Win.OleControls`' `TOleControl` — the D13 RTL does this 47 times in
  those two units alone). A resolver that only runs its ancestor walk for
  method BODIES misses these: the specifier sits in the type DECLARATION, yet
  must resolve through the full cross-unit ancestor chain like any bare
  member reference. Note the ordering hazard for a deferred/multi-pass
  design: the ancestor walk reads the heritage clause of this same
  declaration, so heritage references must be resolved in an earlier round
  than the specifiers that depend on them.
- ⚠️ *A property is not a memory location, so it cannot be a `var`/reference
  argument* — even though a plain field of the same type works fine there. Two
  different call shapes hit two different errors (dcc-verified, dcc32 37.0):
  `Inc(Obj.Prop)` is `E2064 Left side cannot be assigned to` (the intrinsic
  lowers to a read-modify-write it cannot express through a property), while
  passing the property to a user-declared `procedure P(var X: Integer)` is
  `E2197 Constant object cannot be passed as var parameter` (the property read
  is treated as a non-addressable rvalue). This holds regardless of whether the
  property is field-backed or method-backed. See §13.1.6 for the one exception —
  a `var` parameter inside the property's own *setter*, gated behind
  `{$VARPROPSETTER}`.
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
- ⚠️ *The index parameter's NAME is a pure signature placeholder with no
  scope of its own* (dcc-verified: `System.Actions.pas`'s
  `TCustomShortCutList.ShortCuts`, whose getter is `function
  GetShortCuts(Index: Integer): TShortCut`) — nothing in the language can
  ever reference this name, anywhere, including inside the getter/setter:
  `Obj.Items[3]` lowers to the getter/setter by parameter POSITION, per the
  general call-binding rule (ch.06) — Pascal has no named-argument call
  syntax, so the property's own stated name is never matched against the
  getter's. (Every array property surveyed in the RTL happens to reuse the
  same name on both sides by convention — this is a STYLE choice, not a
  language requirement; nothing enforces it.) The resolver must still give
  this name a declaration somewhere reachable only from within the
  property's own bracket list, or it misreads as an undeclared reference.

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
  ⚠️ *Including an INHERITED one* — the search walks the ancestor chain, so
  `Coll[I]` is legal on any `TCollection` descendant even though `Items` is
  declared on `TCollection` itself. The RTL and VCL lean on this constantly
  (`TStrings.Strings`, `TCollection.Items`), and a resolver that only looks at
  the type's own members silently types `Obj[i]` as the COLLECTION instead of the
  element, which then makes every member reached through it undeclared.
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

### 13.1.6 Put-by-reference setters (`{$VARPROPSETTER}`)

| | |
|---|---|
| **Introduced** | Delphi 2009 (COM "put by ref" support) |
| **Deprecated** | — |
| **Status** | ✅ Current, but opt-in |

A property setter may take its value as a `var` parameter instead of a by-value
one, letting the setter observe/mutate the caller's variable as a side effect of
the assignment. This is disabled by default and gated behind the
`{$VARPROPSETTER ON}` compiler directive.

**Example**

```pascal
{$VARPROPSETTER ON}
type
  TMyIntegerClass = class
  private
    FNumber: Integer;
    function GetNumber: Integer;
    procedure SetNumber(var Value: Integer);   // var, not a plain by-value param
  public
    property Number: Integer read GetNumber write SetNumber;
  end;

procedure TMyIntegerClass.SetNumber(var Value: Integer);
begin
  Inc(Value);           // side-effect on the CALLER's variable
  FNumber := Value;
end;
```

**Semantics & parsing notes**

- ⚠️ *Without `{$VARPROPSETTER ON}`, a `var` setter parameter is a hard error* —
  `E2282 Property setters cannot take var parameters` (dcc-verified, dcc32 37.0;
  reproduced verbatim). With the directive on, the same declaration compiles.
- ⚠️ *Only a variable is assignable once the setter takes `var`* — a constant (or
  any other non-addressable expression) on the right-hand side of the assignment
  is `E2036 Variable required` (dcc-verified), exactly as for any other `var`
  parameter binding. `Mic.Number := 10;` fails; `Mic.Number := N;` (`N` a
  variable) succeeds.
- ⚠️ *The setter can mutate the caller's variable as a side effect of the
  assignment statement* — `Inc(Value)` inside the setter above changes the
  caller's `N` in place, in addition to whatever the setter does with
  `FNumber`. Two syntactically identical consecutive assignments
  (`Mic.Number := N; Mic.Number := N;`) therefore produce *different* runtime
  effects each time, since `N` itself changed after the first call
  (dcc-verified: this pattern turns an initial `N := 10` into `FNumber = 12`
  after two assignments — `N` becomes 11 then 12, and each new value is what
  gets stored).
- This is orthogonal to the general "properties are not addressable" rule in
  §13.1.1 — it does not make the *property itself* passable as a `var` argument
  anywhere; it only changes what the *setter's own parameter* looks like on the
  implementation side of the assignment.
- *AST:* no new grammar — the setter's parameter list already carries `var`/
  `const`/plain per the ordinary `FormalParams` production (ch.06); this is a
  semantic gate on an existing shape, keyed off the `{$VARPROPSETTER}` state.

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

- ⚠️ *Not a hard precondition — "requires `{$M+}`" undersells what actually
  happens.* Declaring an explicit `published` member on a class that is
  **neither** compiled with `{$M+}` **nor** descended from a class compiled with
  it (e.g. a plain `TObject` descendant) does **not** error: the compiler
  silently injects `{$M+}` for that type and emits `W1055 PUBLISHED caused RTTI
  ($M+) to be added to type '<Name>'` — a warning, not a rejection
  (dcc-verified, dcc32 37.0: a `TObject`-descended class with a `published`
  property compiles clean-except-for-the-warning, and the property is genuinely
  reachable afterwards through old-style RTTI, e.g. `TypInfo.GetPropValue`).
  Published property types are still restricted (ordinal, string, set, class,
  method-pointer); the resolver enforces the allowed-type set.
- ⚠️ *`{$M+}` (or a `TPersistent`/RTTI-bearing ancestor, ch.11 §11.2.1) changes
  the DEFAULT visibility of undecorated members* — a member with no explicit
  visibility keyword, positioned in the section right after the class header
  (before any `private`/`protected`/`public`/`published` keyword appears),
  defaults to `published` under `{$M+}` and to `public` otherwise
  (dcc-verified: an undecorated property in that position is found by
  `TypInfo.GetPropInfo` — i.e. is published — when the enclosing class is
  compiled under `{$M+}`, and is *not* found — i.e. is public — with the exact
  same source under a class with no `{$M+}`/`TPersistent` ancestry). This is the
  same default-visibility mechanism as ch.11 §11.2.1's ordinary
  `private`/`public` default, just with a different default keyword selected by
  the `{$M+}` state.
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
