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
- ⚠️ *Default visibility differs by member kind* (dcc-verified, dcc32 37.0): with
  no `{$RTTI}` override, a `private` **field** is still visible via
  `TRttiType.GetFields` in extended RTTI, but a `private` **method** and a
  `private` **property** are NOT visible via `GetMethods`/`GetProperties`. This
  matches the documented system defaults
  `DefaultFieldRttiVisibility = [vcPrivate..vcPublished]` (all four
  visibilities) vs. `DefaultMethodRttiVisibility = DefaultPropertyRttiVisibility
  = [vcPublic, vcPublished]` (public/published only) — fields are RTTI'd at
  every visibility level by default, methods and properties are not.
- ⚠️ *`{$RTTI}` cannot precede the `unit`/`program`/`library` header*
  (dcc-verified, dcc32 37.0): placing `{$RTTI …}` as the first line of a unit,
  before the `unit Foo;` clause, fails with **`E2609 RTTI directive must be
  used after PROGRAM, UNIT or LIBRARY header`** (plus a knock-on "undeclared
  identifier" for the visibility-set constants, since the compiler hasn't
  processed the `System` unit's declarations yet at that point). The directive
  must always come after the header. (The failure is a normal, clearly-worded
  compiler error in this version — not the unintuitive internal-error-style
  failure described in some older references.)
- ⚠️ *Extended RTTI restriction and classic `published`/`{$M+}` RTTI are
  independent — and `published` members bypass the `{$RTTI}` visibility sets
  entirely* (dcc-verified, dcc32 37.0). Fully disabling extended RTTI with
  `{$RTTI EXPLICIT METHODS([]) FIELDS([]) PROPERTIES([])}` on a class that also
  has `published` members under `{$M+}`:
  - Classic RTTI is untouched: `GetTypeData(ClassInfo)^.PropCount` and
    `IsPublishedProp` still report the `published` property normally — the
    directive "doesn't cause any change on the traditional RTTI generated for
    published types" (matches the source material).
  - A `private`/`public` (non-`published`) field/method/property IS correctly
    hidden from `TRttiContext` by the empty visibility sets, as expected.
  - However, a `published` field, `published` property, AND `published`
    method **all still show up via `TRttiType.GetFields`/`GetProperties`/
    `GetMethods`** despite the empty `FIELDS([])`/`PROPERTIES([])`/
    `METHODS([])` sets — i.e. `published` members are exempt from the
    `{$RTTI}` visibility-set restriction even inside the *extended* RTTI API,
    not only in the classic one. A resolver modeling "does this member have
    extended RTTI" must special-case `published` as always-on, independent of
    the class's `{$RTTI}` settings.
- `{$WeakLinkRTTI ON|OFF}` and `{$StrongLinkTypes ON|OFF}` are two further
  directives governing whether extended RTTI **survives smart-linking**:
  `$WeakLinkRTTI` lets the linker drop a type (and its RTTI) entirely when
  nothing in the program references it; `$StrongLinkTypes` forces the opposite
  — every type and its extended RTTI is linked in even if unreferenced, which
  can noticeably increase program size on some projects.
  (book-stated, directive-recognition dcc32-confirmed: both directives compile
  without error, e.g. `{$WeakLinkRTTI ON} {$StrongLinkTypes ON}` at the top of
  a program — but their actual effect is on linker output size/behavior across
  a whole program's set of compiled units, which isn't observable from a single
  throwaway probe's diagnostics, so only directive recognition was verified,
  not the linking behavior itself.)

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
annotate (types, fields, methods, properties, parameters, generic type
parameters).

**Grammar**

```ebnf
AttributeGroup = "[" Attribute { "," Attribute } "]" ;
Attribute      = [ Ident ":" ]                       (* target: Result *)
                 ( TypeRef [ "(" [ ActualParams ] ")" ]
                 | Expression [ ".." Expression ] ) ; (* dropped: W1074 *)
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
  3. *The first bracket of an interface body* → a **GUID** or the first
     member's attribute group, by what it names (§14.1.1) - the one place
     the position does not decide.
  Use the syntactic context (what precedes the `[`) to choose.
- ⚠️ *A target before an attribute* (dcc64 37.0, RTTI-probed 2026-10-14,
  PasTree local/probe-b56 R1, R2, RE*, RT*): `[Result: Tag] function F` is
  F's result's attribute. The target is one attribute's, not the group's:
  `[Tag, Result: Other]` targets Other only (RTTI lists a result's
  attributes before the routine's own). It compiles on a method and on a
  unit-level function; before a procedure, a field, a property, a type or a
  parameter it is `E2289 Unresolved custom attribute`. The attribute is
  resolved and checked like any other (`[Result: Tag(Undeclared)]` is
  E2003, `[Result: TObject]` E2010). `Result` is the one target word:
  `[Self: Tag]`, `[Foo: Tag]`, `[1: Tag]` are W1074 and the attribute is
  dropped with nothing in it looked up (`[Foo: Tag(Undeclared)]` compiles),
  and `[Result]` alone is `E2029 ':' expected`.
- ⚠️ *Positions besides a declaration's head* (dcc64 37.0, RTTI-probed
  2026-10-14, local/probe-b56 GP, V, V2), each checked like any other
  (`[Zork]` there is W1074):
  - *A generic type parameter* (§16): before each name of a type, record,
    interface, procedural type or method parameter list - `TGen<[A] T>`,
    `TGen<K; [A] V: class>`, `TGen<T, [A] U>`, `procedure M<[A] T>`.
  - *A record's variant part* (§9.1.3): before a branch's field
    (`0: ([A] I: Integer; [A] [B] J: Byte)`), before a nested `case` (the
    group lands on that part's first field), before the tag
    (`case [A] Tag: Integer of` - RTTI shows A on Tag AND on the first
    field of the first branch), and as the last thing of a branch
    (`0: (I: Integer; [A])` compiles, no E2289).
- The attribute's `TypeRef` is resolved to a `TCustomAttribute` descendant; the
  `( … )` arguments must match one of its constructors (evaluated to build the
  attribute instance in RTTI). The trailing `Attribute` suffix is optional.
- ⚠️ *The suffixed name is looked up FIRST, through the whole search order.*
  `[X]` means `XAttribute` wherever that name is found - the local scopes,
  the unit, the used units, `System` - and the bare `X` only when
  `XAttribute` is found nowhere. A nearer bare `X` does not win: with a used
  unit declaring `SetupAttribute`, DUnitX's `[Setup] procedure Setup;`
  compiles (the method is not the attribute); a program's own `Bar =
  class(TCustomAttribute)` with `constructor Create(A, B: Integer)` loses to
  a used unit's `BarAttribute.Create(const S: string)` - `[Bar(1, 2)]` is
  `E2010 Incompatible types: 'string' and 'Integer'` and `[Foo('x')]` beside
  a local `Foo(A, B: Integer)` compiles. Both names in one unit: `[Baz('x')]`
  takes `BazAttribute(const S: string)` over `Baz(A, B: Integer)`. Whatever
  the suffixed name finds decides, a non-type too: a `const QAttribute = 1`
  makes `[Q]` `W1074 Unknown custom attribute: 'QAttribute'` beside a `Q =
  class(TCustomAttribute)`. A unit-qualified name takes the suffix the same
  way in that unit: `[UAt.Test(2)]` is `UAt.TestAttribute`. dcc64 37.0,
  probed 2026-10-12 (PasTree audit B27, `probe-b27` A1-A4).
- Attributes are only retained in binaries where extended RTTI is enabled
  (`{$RTTI}`); otherwise they parse but emit nothing.
- ⚠️ *Constructor arguments must be compile-time constant expressions*
  (dcc-verified, dcc32 37.0), because they are resolved at compile time to
  build the attribute instance in RTTI. Confirmed with an attribute whose
  constructor takes an `Integer`:
  - a literal (`[Tag(3)]`) — OK.
  - a `const` symbol (`[Tag(CVal)]` with `const CVal = 5;`) — OK.
  - a plain variable (`[Tag(GVar)]`) — **`E2026 Constant expression expected`**.
  - a function-call result (`[Tag(SomeFunc)]`) — **`E2026 Constant expression
    expected`** (same error as the variable case).
  - a class-reference argument (constructor parameter typed `TClass`, applied
    as `[ClassRef(TObject)]`) — OK, confirming class references are among the
    allowed constant-expression kinds alongside ordinals/strings/sets.
- ⚠️ *No `AttributeUsage`-style target restriction exists* (dcc-verified,
  dcc32 37.0): an ordinary `TCustomAttribute` descendant with no special
  marker or metadata compiles cleanly when applied to all four attributable
  target kinds — a type, a field, a method, and a formal parameter — in the
  same program, with no restriction-related error or warning from any of the
  four placements. Object Pascal has no language mechanism (unlike .NET's
  `AttributeUsageAttribute`) to declare that a given attribute class may only
  be attached to certain kinds of declarations; a resolver must not invent or
  enforce such a restriction.
- ⚠️ *A group annotates the next member, whatever ends before it*
  (dcc-verified, dcc64 37.0, RTTI-probed 2026-09-30): after a `var`,
  `const` or `type` section - at unit level, in a routine's declarations,
  and in a class or record body after a `var`, `class var`, `const` or
  nested `type` section - a group followed by no name of that section is
  the attribute of the routine, method or property after it:
  `class var F: Integer; [A] procedure M;` gives `M` the attribute `A`, and
  so does `const C = 1; [A] [B] procedure M;` (both). It crosses a
  visibility word: `private [A] public procedure M;` and
  `class var F: Integer; [A] public procedure M;` annotate `M`, and
  `const C = 1; [A] strict private FX: Integer;` annotates the field `FX`.
  A group before a name of the section is that declaration's
  (`var X: Integer; [A] Y: Integer;`). A group with no member after it - before
  a class's `end`, before `implementation`, before a routine's `begin` - is
  `E2289 Unresolved custom attribute`. The ancestry check still applies
  where the group lands: `var GX: Integer; [TNotAnAttr] procedure P;` is
  E2010.
- ⚠️ *Any expression is accepted as an attribute, and dropped* (dcc-verified,
  dcc64 37.0, probed 2026-10-02, x-f32 B01-B60): `['abc']`, `[1 + 2]`, `[-X]`,
  `[nil]`, `[(TA)]`, `[(TA)(1)]`, `[[1]]`, `[1..2]`, `[inherited]` compile with
  `W1074 Unknown custom attribute` and annotate nothing, and the names in them
  are not looked up: `[(Undeclared)]`, `[-Undeclared]`, `['a' + Undeclared]`
  compile. A NAME is resolved first: an attribute class takes its arguments and
  nothing more (`[TA + 1]`, `[TA(1) + 1]`, `[TA(1).ClassName]`, `[TA[1]]`,
  `[TA<Integer>]`, `[TA = TA]` are E2029 `',' or ']' expected`), its arguments
  checked (`[TA(Undeclared)]` is E2003); another type is E2010 (`[TObject]`,
  `[Integer]`); any other name - a constant, a variable, a function, an
  undeclared one - is W1074 `Unknown custom attribute: 'C'` (`[C]`, `[F(1)]`,
  `[Undeclared]`), and the tokens after it are skipped up to the `,` or `]`
  unchecked: `[C + 1]`, `[C.X]`, `[C(Undeclared)]`, `[Undeclared + 1]`, even
  `[C + + ]` and `[C (]` compile, while `[C )]`, `[C begin]`, `[C;]` are E2029.
  Only the FIRST segment of a dotted name decides it: `[Undeclared.X]` and
  `[SysUtils.Undeclared]` under `uses System.SysUtils` are W1074, but a unit
  qualifier is looked into - `[System.Undeclared]`, `[ThisUnit.Undeclared]` are
  E2003 (x-f33 U01-U26).
  A group's attributes are judged one by one (`[TA, 'x', TA(2)]` keeps both
  TAs), and so is a second GUID bracket after an interface's GUID (§14.1.1).
- *AST:* attach `attributes: [ { type, args[] } ]` to the annotated declaration node;
  an expression attribute is `Attribute { expr }`.

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
