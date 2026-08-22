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
- ⚠️ *"All classes descend from `TObject`" includes classes with NO heritage
  clause* — `TFoo = class … end;` is `class(TObject)`. Consequence for name
  resolution: `TObject`'s members resolve **bare** inside such a class's own
  methods (`ClassName`, `Free`, `InitInstance` — the RTL does this
  constantly), so an ancestor walk must not stop where the heritage clause is
  merely absent. Records and legacy `object` types have no implicit ancestor;
  an interface's implicit `IInterface` contributes no implementation, so its
  members need no such walk.
- A class declared `class;` or `class(TParent);` with no member block is a
  **forward declaration** (completed later in the same type section) — used for
  mutually referencing classes.
- The optional `[ Abstract ]` in the `ClassType` grammar above is the
  class-level `abstract` directive (the same one in ch.12's `ClassHeader`
  grammar) — it does **not** block instantiation of the class. See
  [ch.12 §12.2.4](12-inheritance-polymorphism.md#1224-abstract-methods) for
  the compile-time warning that DOES fire, and only when an actual
  unoverridden `virtual; abstract` method is present.
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
- The qualified body MAY omit the parameter list when it matches the
  declared header exactly — same rule as a forward-declared global routine,
  see ch.06 §6.1.2 (dcc-verified in the RTL).
- Method directives (`virtual`, `override`, `overload`, `abstract`, `reintroduce`,
  `static`, `dynamic`, `message`, `inline`, …) follow the header — see ch.12/15.
- *AST:* `MethodDecl` header + linked body `Routine`.

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
- ⚠️ *`strict private` reaches into NESTED types, and the relation is asymmetric*
  (dcc32 37.0-probed both ways). A type nested in the declaring class may read
  its enclosing class's `strict private` members:

  ```pascal
  TOuter = class
  strict private
    FVal: Integer;
  public type
    TInner = class
      procedure Poke(const A: TOuter);
    end;
  end;

  procedure TOuter.TInner.Poke(const A: TOuter);
  begin
    A.FVal := 1;              // OK — a nested type is inside the declaring class
  end;
  ```

  The other direction is refused: an enclosing class reading a NESTED class's
  `strict private` member is `E2361 Cannot access private symbol
  TOuter.TInner.FIn`. So "inside the declaring type" includes what is nested in
  it, and not what it is nested in. `System.JSON.Builders` depends on the legal
  direction (`TJSONCollectionBuilder.TBaseCollection` reads the outer class's
  `FJSONWriter`).
- ⚠️ *`published` requires RTTI:* `published` members generate runtime type info and
  are restricted to classes compiled with `{$M+}` (or descending from `TPersistent`).
  Allowed member types are constrained (e.g. method pointers for events, ordinal/
  string/class properties). See ch.13/19.
- `strict` is a **directive** (B.4.2) preceding `private`/`protected`.
- ⚠️ *The "protected hack" is a direct consequence of the same-unit rule
  above* (dcc-verified, dcc32 37.0). Given `TTest` declared in unit `UTest`
  with a `protected` field, code in a *different* unit cannot reach that
  field directly — but declaring an empty descendant in the accessing unit,
  `TAccessHack = class(TTest);`, and hard-casting an existing `TTest`
  instance to it makes it reachable: `TAccessHack(SomeTestInstance).
  FProtectedField` compiles and works. The reason is exactly the rule above
  applied twice over: `TAccessHack` inherits the field, is declared in the
  SAME unit as the accessing code (so the unit-level friend rule grants
  access), and has an identical memory layout to `TTest` (it adds no
  members), so the cast is safe in practice even though it is an unchecked
  hard cast. This pattern is pervasive in the RTL/VCL source for reaching a
  base class's protected members from a sibling unit without adding a public
  accessor — legal, if discouraged, and considered part of the language
  specification rather than a bug.
- ⚠️ *A GENERIC class's plain `protected` is enforced only OUTSIDE method
  bodies* (dcc32 37.0-probed, four ways). Given `TNodeG<T> = class protected
  FX: Integer; end;` in unit A: a **method** of any class — unrelated,
  non-generic, in another unit — may read/write `N.FX` on a `TNodeG<Integer>`
  (or open `TNodeG<T>`) value, while the SAME access from a unit-level
  routine or a program's main block is `E2362 Cannot access protected
  symbol`. The other three cells of the matrix behave normally: `strict
  protected` of a generic is refused even from method bodies, and a
  NON-generic class's `protected` is refused from methods and unit-level
  code alike. So the leniency needs exactly (generic target) × (method-body
  accessor). spring4d's collections lean on it — `TLinkedList<T>`'s methods
  assign `node.fNext`/`node.fList`, protected members of `TLinkedListNode<T>`
  declared in another unit with no descent relation — so a checker that
  applies the written rule to generics reports legal shipping code. Whether
  this is intended or a dcc visibility-check gap, it is the compiler of
  record's observable behavior.
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
- ⚠️ *Calling `inherited Create` is OPTIONAL for constructors, not compulsory*
  (dcc-verified, dcc32 37.0: a descendant constructor that never calls
  `inherited Create` compiles cleanly, with no error or hint, and runs
  normally). Unlike C++/C#/Java, where invoking the base-class constructor is
  implicit and effectively required, Object Pascal leaves the call to the
  programmer. Omitting it does not fail to compile — it simply means the
  ancestor constructor's body never executes, so any state that body would
  have set up (beyond the automatic zero-fill of fields) stays at its
  zero-initialized default. This is the same `inherited` mechanism documented
  generally for methods (§12.1.2), but the compiler does not enforce the call
  specifically for constructors; good practice still calls it (see `Destroy`,
  §11.3.2, where the pattern is followed).
- ⚠️ *A constructor may be named anything — the `constructor` keyword marks
  the method, not the identifier `Create`* (dcc-verified, dcc32 37.0:
  `TFoo = class(TObject) constructor Init; end;` — both the custom `TFoo.Init`
  AND the inherited `TObject.Create` are valid, independently callable
  constructors on `TFoo`; declaring `Init` neither hides nor replaces
  `Create`). A custom-named constructor is *additive*, not a replacement: to
  actually prevent callers from reaching the inherited default construction
  path, a class must declare its OWN constructor named `Create`.
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
- ⚠️ *The QUALIFIER need not be the class that declares the nested type — it may
  merely INHERIT it.* A nested type is a member like any other, so
  `TDesc.TNested` resolves through `TDesc`'s ancestors, at any nesting depth and
  across units. dcc-verified: with `TInner` declaring nested `TDeep` and
  `TSub = class(TInner)` declaring nothing, `class(TBase.TSub.TDeep)` compiles.
  This is the qualified counterpart of the bare-name reach described below, and
  a resolver that searches only the qualifier's OWN members pays the same price
  as one that misses that. Alcinoe nests this three deep —
  `TALMemo.TDisabledStateStyle.TTextSettings =
  class(TALBaseEdit.TDisabledStateStyle.TTextSettings)`, where the middle segment
  declares no `TTextSettings` and its own ancestor `TBaseStateStyle` does — and
  the class then loses its whole ancestry, so the diagnostic surfaces one dot
  away, on `TTextSettings.Create`.
- A nested CLASS's own methods are implemented with the FULL qualified chain,
  one segment per nesting level: `procedure TGrid.TCell.Method;` (not just
  `TCell.Method;`) — resolve each segment inside the PREVIOUS segment's member
  scope (`TGrid` at unit scope, `TCell` inside `TGrid`'s scope), not all at
  unit scope, or a nested class's own name is invisible outside its outer
  class. See `06-routines.md` `ImplName` (arbitrary dotted chain) and
  `16-generics.md` §"nested generic" for the same rule under generics.
- ⚠️ *The DECLARATION of a nested type sees the members of every class it is
  nested in — and of each of those classes' ANCESTORS.* The same reach as the
  method-body rule below, one step earlier: it applies to the heritage clause
  itself. dcc-verified — with `TBase` declaring nested `TInner` and a const
  `MaxRows`, and `TDesc = class(TMid)` where `TMid = class(TBase)`, this
  compiles:

  ```pascal
  TDesc = class(TMid)
  public type
    TSub = class(TInner)                    // grandparent's nested type, bare
      A: array[0..MaxRows] of Integer;      // grandparent's const, bare
    end;
  end;
  ```

  `Vcl.Skia.pas`/`FMX.Skia.pas` rely on it (`TSkAnimatedPaintBox.TAnimation =
  class(TAnimationBase)`, a nested type of `TSkCustomAnimatedControl`). A
  resolver that misses it reports far more than the one name: the nested class
  is then left with NO ancestry, so its property specifiers (`read GetDuration`)
  and its methods' inherited constants fail too — 6 false undeclared-identifiers
  per Skia unit from a single unresolved heritage reference.
- ⚠️ *The body of such a method sees the members of EVERY qualifier segment —
  and of each segment's ANCESTORS.* Not just the innermost class's own
  inheritance chain: `TParallel.TLoopState32.TLoopStateFlag32.ShouldExit`
  (`System.Threading.pas`) freely uses `TLoopStateFlagSet`, a private nested
  type of `TLoopState` — the ancestor of the MIDDLE segment (`TLoopState32 =
  class sealed(TLoopState)`). Precedence stays innermost-first: the inner
  segment's own ancestry is searched before the outer segments'. A resolver
  that walks only the innermost segment's ancestors mis-reports these
  (16 false undeclared-identifiers in that one RTL unit).
  One refinement (dcc32 37.0-probed): the OUTER segments contribute name
  VISIBILITY, not a `Self` — an outer segment's (or its ancestor's) CLASS
  function/var/const is callable bare from the nested method's body, while
  its INSTANCE members resolve but fail with `E2124 Instance member
  inaccessible here` (there is no enclosing-class instance to call them on).
  So the diagnostic for a mis-scoped resolver differs by member kind: a
  missed class member surfaces as a false E2003, a missed instance member
  as the wrong ERROR KIND. FMX ships the class-side case bare —
  `TWinGestureEngine.TRealTimeStylus.CustomStylusDataAdded` calls
  `IsGesture(...)`, a class function of `TGestureEngine`, the OUTER class's
  cross-unit ancestor (`FMX.Gestures.Win`).
- ⚠️ *A member — including an INHERITED one — outranks a compiler-PREDEFINED
  name.* The implicit System scope is the outermost one, so a class member of
  the same name shadows it throughout the method body, in **every** position.
  dcc32 37.0-probed with `Text`, which is a predefined file type (ch.10 §10.3):

  ```pascal
  TBase = class
    property Text: string read FText;
  end;
  TDesc = class(TBase)
    function Use: Boolean;
    procedure Decl;
  end;

  function TDesc.Use: Boolean;
  begin
    Result := Text.IsEmpty;   // OK — the inherited PROPERTY, not the file type
  end;

  procedure TDesc.Decl;
  var
    F: Text;                  // E2007 — the property shadows the type here too
  ...
  ```

  FMX's canvas units rely on the first form (`TTextLayout.Text`, used bare in
  descendants across units). A resolver that binds predefined names in an early
  pass and only fills GAPS later gets this wrong silently: the name resolves, so
  no diagnostic appears — only the member after the dot fails, and ctrl+click
  lands on nothing.
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
- ⚠️ *Being a value type, it has a LAYOUT* (9.1.2 applies to it unchanged), plus
  one rule of its own: declaring a **virtual** member appends a VMT pointer
  **after** the fields, pointer-aligned — and that pointer does **not** raise
  the type's own alignment. So `object A: Integer; procedure P; virtual; end`
  is 16 bytes on Win64 while still aligning to 4, and a `Byte` before such a
  field gives 20, not 24. A derived object whose ancestor already has a VMT
  does **not** get a second one (that one, plus `C: Byte`, is 20). Fields of a
  derived object start where the ancestor's storage ended. An `object end` is
  0 bytes.
- ⚠️ *Two syntax restrictions dcc enforces here but not on records:* fields must
  precede methods (`E2169 Field definition not allowed after methods or
  properties`), and a **variant part is not allowed at all** — `case … of`
  inside an `object` is a syntax error.
- *AST:* `ObjectType { … }` (value-type OOP node).
