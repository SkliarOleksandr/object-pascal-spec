# 16 — Generics

Parameterized types and methods, constraints, instantiation, and the
`<`-disambiguation that is the single hardest parsing problem in the language.

Shared productions (`TypeRef`, `GenericArgs`) → [Appendix B
§B.11](B-lexical-grammar.md#b11-type-references).

## Chapter grammar umbrella

```ebnf
GenericParams = "<" TypeParamDecl { ";" TypeParamDecl } ">" ;
TypeParamDecl = TypeParamName { "," TypeParamName } [ ":" ConstraintList ] ;
TypeParamName = { AttributeGroup } Ident ;               (* attributes: 19.3.2 *)
ConstraintList = Constraint { "," Constraint } ;
Constraint    = "class" | "record" | "constructor"      (* kind constraints *)
              | TypeRef ;                                 (* class/interface constraint *)
GenericArgs   = "<" TypeRef { "," TypeRef } ">" ;
```

---

## 16.1 Generic types

### 16.1.1 Generic classes, records, interfaces

| | |
|---|---|
| **Introduced** | Delphi 2009 (Win32 native; .NET earlier) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A type parameterized over one or more type parameters.

**Example**

```pascal
type
  TPair<TKey, TValue> = record
    Key: TKey;
    Value: TValue;
  end;

  TList<T> = class
    procedure Add(const Item: T);
  end;
```

**Semantics & parsing notes**

- The declaration name carries `GenericParams` (`TList<T>`); the parameters are in
  scope throughout the type body.
- ⚠️ *`<T, U>` is ONE parameter group carrying two names, not two groups* — the
  separator between groups is `;`, and `,` separates names **within** a group
  (`<T; U: class>` is two, `<T, U>` is one). A reader that takes the first name
  of each group silently loses every parameter after the first comma, and a
  positional binding of actuals to parameters then goes wrong rather than
  failing. After a `:` the `,` belongs to the CONSTRAINT list: `TG<T: class,
  U>` is one parameter `T` constrained by `class` and a type `U`, which is
  `E2003 Undeclared identifier: 'U'`; a second parameter after a constrained
  one needs `;` (`<T: class; U>`). dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 G10.dpr).
- ⚠️ *A generic RECORD has a size, and it is the body laid out with the
  parameters bound* — `SizeOf(TPair<Byte, Int64>)` is 16 by the ordinary rules
  of 9.1.2, with `TKey` and `TValue` replaced by the actuals. The parameters
  can appear anywhere a type can: a field, a static array's element, or the
  actual of a nested instantiation (`TOuter<T> = record V: TInner<T>; end`),
  and the substitution has to follow them there.
- ⚠️ *A generic record may NOT contain a variant part over a parameter* —
  `E2569 Type parameter 'T' may need finalization - not allowed in variant
  record`. A `class` constraint does not lift it (a class reference needs no
  finalization, yet `TR2<T: class>` with `0: (A: T)` is the same E2569); only
  `T: record` does - `TR3<T: record>` with the same variant part compiles and
  instantiates as `TR3<Integer>`. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 G09.dpr, G09b.dpr).
- *AST:* `GenericTypeDecl { name, typeParams[], constraints, body }`.

### 16.1.2 Overloading by arity

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

The same identifier may name types of different generic arity (`TFoo`,
`TFoo<T>`, `TFoo<T1,T2>`).

**Semantics & parsing notes**

- Resolution selects by the **number of type arguments** supplied at the use site.
  The resolver keys generic symbols by `(name, arity)`.
- ⚠️ *A reference supplying NO type arguments names the ARITY-0 declaration —
  even when a same-named GENERIC is nearer in scope.* Arity is part of the
  identity, so ordinary "nearest declaration wins" shadowing does not apply
  across arities. dcc-verified:

  ```pascal
  // unit A
  TBase = class ... end;                 // arity 0

  // unit B
  uses A;
  TBase<T: class> = class(TBase)         // arity 1, and its OWN heritage is
    ...                                  // the bare name -> A.TBase
  end;
  TDerived = class(TBase);               // zero args -> A.TBase, NOT B's
  ```

  Third-party component libraries do this deliberately: a plain base class in a
  core unit, a same-named generic wrapper over it in a unit above, and code in
  that upper unit using both spellings.
  - *Why an implementation must get this right rather than approximately right:*
    binding the bare name to the nearer GENERIC is not a small imprecision. The
    generic's own heritage is that same bare name, so it resolves to ITSELF —
    a self-reference that terminates the ancestor walk, and with it every
    inherited member of every descendant. Observed on a real code base: 100+
    false undeclared-identifiers, every one of them reported on a name declared
    three hops away from the mistake.
  - ⚠️ *The two declarations are usually in different USED UNITS, and that is
    where an implementation gets it wrong.* Looking the name up with the
    ordinary last-uses-wins rule returns whichever unit was imported later — and
    if that is the generic, a search that stops there has just re-found the
    wrong one. Arity is part of the identity, so a generic candidate must not
    END the search; the scan has to continue for a matching-arity declaration.
    The RTL itself sets this trap: `TObjectList` is a plain class in the
    container unit and a generic in the generic-collections unit, and any unit
    importing both can write either.
  - The rule is about the reference, not the declaration: the base of `T<...>`
    is *supposed* to select by the supplied argument count, so an
    arity-preference applied blindly inside generic-argument resolution would
    break every `TList<Integer>`. Note that an identifier *inside* `<...>` is
    itself a bare reference — `TDict<Pointer, TList>` means the arity-0
    `Pointer` — so the exemption covers the BASE of the reference only.
  - ⚠️ *The arity-0 candidate may be a PREDEFINED type rather than a
    declaration*, and then it is not in any unit's scope at all: a
    collections library declares `Pointer<T> = record` beside the compiler's own
    `Pointer`, so the search for a matching arity has to reach the predefined
    names too, not only the current unit and its imports. Every
    `Pointer(X) := nil` in that unit depends on it.
  - ⚠️ *Two GENERIC arities of one name are the same rule, and the one an
    implementation is most likely to miss* — "does this reference supply type
    arguments" separates `TFoo` from `TFoo<T>` and says nothing about
    `TFoo<T>` versus `TFoo<K,V>`. Both are generic; only the COUNT tells them
    apart, and only the head of the chain is registered under the name.
    - *Why it is worth its own bullet:* the two declarations are usually
      parallel designs of the same container, so they declare the SAME nested
      type names. Binding the qualifier to the wrong arity then resolves
      `TFoo<K,V>.PNode` to the other declaration's `PNode` — a type that exists,
      has most of the same members, and differs in one field. Nothing fails
      until that field is read, and the report lands in a method of the right
      class pointing at a field name that is genuinely absent from the type it
      was handed.
  - ⚠️ *The two arities may also sit in ONE unit, split across its sections* —
    the generic in the interface, a plain instantiation alias in the
    implementation:

    ```pascal
    interface
    type
      TSomeProvider<T: TSomeControl> = class(TBaseProvider<T>) ... end;
    implementation
    type
      TSomeProvider = class(TSomeProvider<TSomeControl>);   // short name for
                                                            // the common case
    ```

    This is idiomatic, not odd, and it is the hardest placement to get right
    because it falls between the two mechanisms an implementation naturally
    builds: same-name declarations chained within one scope, and a cross-unit
    search. Here the arities are in the same unit but different scopes, so
    neither applies. The alias is the nearer declaration, so its own heritage
    reference resolves to the alias ITSELF — the self-reference described above,
    and every inherited member named from the generic's method bodies goes
    undeclared.
- ⚠️ *A FORWARD declaration completes by `(name, arity)` too, and this composes
  with arity overloading in a way that is easy to get wrong.* A container
  library's iterator unit declares, in this order:

  ```pascal
  TIterator = class(...) ... end;          // arity 0
  TIterator<T> = class;                    // arity 1, FORWARD
  TList<T> = class(...) ... end;           // needs the forward
  TIterator<T> = class(...) ... end;       // arity 1, the completion
  ```

  The completion must be matched against **every** declaration the name already
  carries, not just the most recent or the first: an implementation that looks
  only at the head of the chain finds the arity-0 class, reads the mismatch as
  "a different type at a different arity", and declares a THIRD symbol. The
  empty forward then stays the arity-1 winner, so every method body of the real
  class loses its own fields *and* its inherited members — the diagnostics land
  far from the declaration that caused them. Note the non-generic sibling is
  what makes the shape appear at all; a plain forward + completion pair hides
  the defect completely.
- ⚠️ *A method IMPLEMENTATION is completed by arity as well, at every
  segment of its header.* `TFoo` and `TFoo<T>` may both declare `Clear` in
  one unit; `procedure TFoo<T>.Clear;` completes the generic type's and
  `procedure TFoo.Clear;` the plain one's, in either order. The method name
  carries its own arity the same way: beside `procedure Gen(A: Integer);
  overload;` a class may declare `procedure Gen<T>(A: Integer); overload;`
  - one parameter list - and `procedure TFoo.Gen<T>(A: Integer);` completes
  the generic one (dcc64 37.0, probed 2026-10-11: both pairs compile and
  run, each body reached by its own call). A tool pairing a declaration with
  its implementation by names and parameter types alone pairs both
  `Clear`s, and both `Gen`s, with one body.

---

## 16.2 Generic methods

### 16.2.1 Generic (parameterized) methods

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A method (in a generic or non-generic type) with its own type parameters.

**Example**

```pascal
function Max<T>(const A, B: T): T;
```

**Semantics & parsing notes**

- Type arguments may be **explicit** (`Max<Integer>(a, b)`) or **inferred** from
  the value arguments (`Max(a, b)`) — see 16.5.
- ⚠️ *A generic method with no required parameters is CALLED without
  parentheses, type arguments and all* — the ordinary paren-less call (ch.06),
  unchanged by the `<...>`. `Node.GetData<TItem>` in a value position is a call
  whose type is the result with `T := TItem`, exactly like
  `Node.GetData<TItem>()`: `var A := Node.GetData<TItem>;` makes `A` a
  `TItem`, `Node.GetData<TItem>.Site` reads the member straight off it, and
  `var Z: Integer := Node.GetData<TItem>;` is E2010 `Integer` vs `TItem`
  (dcc64 37.0, probed 2026-09-25; VirtualTreeView's `TVirtualNode.GetData<T>`
  is the everyday case). The parse gives no help here — the §16.3.1 follower
  rule reads `Id<Args>` followed by `;` as type arguments, and with no `(`
  there is no call node at all, only the instantiation — so the typer must
  decide it: `Id<Args>` whose `Id` binds a ROUTINE, in a value position, is a
  call. Typing it as a type expression (it is shaped like `TList<Integer>`)
  yields nothing, or the open `T`, and every member after it goes dark.
- ⚠️ *Constraints live on the DECLARATION here too* (§16.4.1 states the rule
  and its error), so a generic method's implementation header repeats a bare
  `<T>`:

  ```pascal
  // interface
  function GetNamedObject<T: TRttiNamedObject>(const AName: string): T;
  // implementation — no constraints, and dcc refuses them here
  function TRttiType.GetNamedObject<T>(const AName: string): T;
  ```

  A resolver answering "what members does a `T` value have" inside that body
  must therefore find the METHOD's declaration, not the enclosing type's
  parameters. Note the implementation name is a flat run of segments and **each
  segment may carry its own parameter list** (`TList<T>.Sort`,
  `TFinder.Pick<T>`), so the owner of a list is the segment immediately before
  it. System.Rtti's `GetNamedObject` is the real case: its body calls
  `Obj.HasName`, which only the constraint can answer for.
  The rule reaches past the innermost type: in `function TDict<K,
  V>.TEnum.MoveNext(A: V): Boolean` (TEnum nested in `TDict<K; V: TItem>`)
  the list follows `TDict`, so `A.Site` and a local `Loc: V` see TItem's
  members although the method belongs to TEnum. And among OVERLOADED generic
  methods the body's parameter is the one of the overload its header
  implements, matched by the parameter list like any overload (6.3.1): with
  `Get<T: TItem>(A: Integer): T; overload;` and `Get<T: TOther>(const S:
  string): T; overload;`, the body of `TFinder.Get<T>(const S: string)`
  accepts `Result.OtherM` and rejects `Result.ItemM` (`E2003`). dcc32 /
  dcc64 37.0, 2026-10-13 (PasTree audit B41: A2-13, B3-31; local/probe-b41
  G16).
- ⚠️ *A value parameter may reuse a type parameter's own name* — dcc-verified:

  ```pascal
  class function TGenericsCast<T, TT>.Cast(const T: T): TT;
  ```

  is legal (`T` the type parameter, `T` the parameter typed by it). The type
  parameters and the value parameter list are two DIFFERENT scopes — the type
  parameters' own scope sits between the method's declaring scope and the
  parameter/body scope, so the parameter's `T` shadows the type parameter's
  `T` for the rest of the signature and body, exactly as an inner block's
  local shadows an outer one (3.3.1); it is not a same-scope redeclaration. A
  resolver that collects a generic method's type parameters into the SAME
  scope as its value parameters (rather than a scope of their own) reports
  this shape as a false redeclaration error.
- ⚠️ *A generic method cannot be virtual, and an interface cannot declare
  one.* `procedure V<T>; virtual;`, `dynamic`, and a `class procedure
  CV<T>; virtual;` are each `E2533 Virtual, dynamic and message methods
  cannot have type parameters`; `procedure M<T>;` in an interface is `E2535
  Interface methods must not have parameterized methods`. So a generic
  method is always statically bound - no VMT slot, no interface slot - and a
  call to one never dispatches. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 G05.dpr).
- *AST:* `MethodDecl { typeParams[], … }`.

---

## 16.3 Instantiation & the `<` ambiguity

### 16.3.1 Generic instantiation syntax

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Supplying type arguments produces a concrete (closed) type.

**Example**

```pascal
var L: TList<Integer>;
L := TList<Integer>.Create;
P := TPair<string, Integer>.Create('a', 1);
```

**Semantics & parsing notes**

- ⚠️ **THE core parser ambiguity.** `<` after an identifier may begin a generic
  argument list **or** be the less-than operator. Object Pascal resolves it largely
  by **context plus bounded lookahead**:
  - In a **type position** (after `:` in a declaration, in `uses`-free type refs),
    `Id<…>` is always generic args.
  - ⚠️ *Not after `class of`:* dcc accepts no instantiation there at all.
    `TMeta = class of TG<Integer>;` is `E2029 ';' expected but '<' found`,
    and so is `type TM = class of TH<T>;` nested in `TH<T>` itself (a bare
    `class of TH` there is `E2003 Undeclared identifier: 'TH'`); through an
    alias, `TGI = TG<Integer>; TMeta2 = class of TGI;` is `E2021 Class type
    required`. What works is a non-generic class derived from the
    instantiation: `TGI = class(TG<Integer>) end; TMeta3 = class of TGI;`.
    dcc64 37.0, 2026-10-31 (PasTree audit E1-08,
    local/probe-park/specfu/E1-08 A1-A5).
  - In an **expression position**, dcc does not scan the tokens: it decides
    by what the head NAMES (next bullet) - `TList<Integer>.Create` is an
    instantiation because `TList` names a generic type, not because `>` is
    followed by `.`. Only a parser without that knowledge falls back on a
    bounded lookahead over the followers of `>`, which is an approximation
    (PasTree's, two bullets down).
  - ⚠️ *dcc decides by what the head NAMES, not by syntax* (dcc64 37.0,
    probed 2026-10-13): after a generic routine the list is type arguments
    whatever follows - `N.GetData<PRec>^.X`, `V.AsType<Integer> + 1`,
    `V.AsType<Integer> - 1`, `W.AsType<TBytes>[0]` all compile as generic
    calls; after a value, `F(A < B, C >= D)`, `F(A < B, C > (D))` and
    `F(0 < A, B >= 1)` are two comparisons each. Any follower rule based on
    syntax alone is therefore an approximation that misses in both
    directions; a parser that cannot see what the head names has to pick.
  - *A syntactic approximation (PasTree's, corpus-checked):* scan only after
    a NAME - a bare identifier or a member, whatever it is a member of
    (`FValues[I].AsType<Integer>`); a literal, call, index or parenthesis
    right before `<` is a value. Accept generic args when the token after
    `>` **cannot start an operand** (`;` `)` `]` `,` `=` `then` `do`
    `and`…): the comparison reading `(a < X) > ⟨follower⟩` would lack a
    right operand. Real cases: `Value.AsType<TBytes>;`, `AsType<char> =
    #0`, `if X.AsType<Boolean> then`, `@Proc<T>;`. After a DOTTED head
    (`X.Id<…>`, every generic method call above) `^ [ + -` are generic too:
    the comparison reading would compare a Boolean with a number, set or
    pointer, never type-correct outside Variant or operator-overload
    chains; after a bare head they stay operands, so `VA < VB > -VC` over
    Variants (which dcc compiles as a comparison chain) is kept. A fused
    `>=` closes the list only when the list has no top-level comma - with
    one, `A < B, C >= D` is two complete comparisons. `(` reads as a call or
    cast, except in the one shape below that the parse leaves open.
  - ⚠️ *`(` after a list with a top-level comma is decided by the head
    alone* (dcc64 37.0, 2026-10-30, PasTree audit E2-04, local/probe-park/e204
    G1.dpr, G3.dpr). In `F(H < I1, ..., In > (E))` the list is type
    arguments when `H` names a generic type (a cast) or a generic routine -
    a bare generic method, `H.Gen<Integer, string>(D)`, and an overload set
    that holds a generic overload, `H.Ovl<Integer, string>(D)` beside a
    plain `Ovl(X: Integer)`: one argument. It is `n` arguments - `H < I1`,
    the middle items as values, `In > (E)` - when `H` names a value: a
    variable, a field (`Fld`, `H.Fld`, `R.F`), a parameter, a constant, a
    property, a NON-generic routine called without parentheses (`PlainF`,
    `H.NonGen`), and a local that SHADOWS a generic method of the same name
    (3.3.2: the innermost declaration decides, its kind decides the
    reading). Three items give three arguments, `Foo3(A < B, E, C > (D))`
    for `Foo3(Boolean; Integer; Boolean)`. The overload chosen for `F`
    follows: `Foo(Gen<Integer, string>(D))` takes `Foo(Integer)`,
    `Foo(Fld < B, C > (D))` takes `Foo(Boolean; Boolean)`. The Delphi 13
    sources, a large client project and a third-party library set hold no
    comparison of that shape and seven generic calls and casts of it
    (`source.Aggregate<Integer, string>(5, ...)`, `IMap<TKey,
    TValue>(this)`).
  - *AST:* where the parse cannot decide, a FLAT `Ambiguous` node: an item
    of an argument, index or set-constructor list that is exactly `H <
    I1, ..., In > ( E )` - `H` a name or a dotted name, each `Ii` a name or
    a dotted name, `E` one expression, and the list's `,` or closer right
    after the `)` - has the leaves `H`, `I1` ... `In` and the parenthesis
    `( E )` as its children, in token order, no grouping; the `<`, the `,`s
    and the `>` are its own. The reading - a generic call or cast, `n`
    comparisons, or not yet known - is decided by what `H` binds to and
    lives beside the tree, which is not rewritten. Any other `(` shape (a
    tail with several arguments, `Aggregate<Integer, string>(5, F)`; an
    operator or selector after the `)`; a head with a prefix or an operand
    to its left) keeps the call or cast reading: in each of them the tail
    parses differently in the two readings, so no flat node can hold both.
  - *A `>` glued to a dereference*, `GetData<PRec>^.X`: lexed context-free,
    `^.` after an operator is a caret character (B.6.2); a `^.` character
    right after a comparison's `>` is never written, so a lexer can take
    `>^.` (no space) as `>` then `^`.
- Practical implementations keep a speculative parse / backtracking path for this.
  **Document the heuristic in the parser** — this is where most hand-written
  Object Pascal parsers get it wrong.
- ⚠️ *A closing `>` written with NO space before a following `=` lexes as ONE
  `>=` token*, a plain maximal-munch consequence with no generics-awareness in
  it: `TFoo<T, U>= class` and `AList.Sort<T>= ...`. dcc still compiles it (it
  is real source: a detours library declares `TGenericsCast < T, TT >= class
  (TObject)`), because dcc's own recursive-descent parser resolves the same
  ambiguity C++ resolves for nested `>>` closing two templates — expecting a
  `>` and finding a `>=`/`>>` instead means "consume one `>`'s worth, and leave
  the rest as the next token", not a syntax error. A parser must do the same
  splice wherever it expects a bare `>` to close a generic-parameter or
  type-argument list: accept a fused `>=` (or, for a doubly-nested close,
  `>>`) as satisfying that `>`, and re-present the remainder (`=`, or a single
  `>`) as the very next token rather than advancing past it. Skipping the
  split is not a rare-case shortcut — it silently drops the LAST type
  parameter or argument in the list (the parser stops scanning for more names
  the moment the fused token fails to compare equal to a bare `>`), and every
  subsequent reference to that parameter reads as undeclared.
- ⚠️ *Args on every dotted segment:* generic arguments may appear on **intermediate**
  segments of a qualified name, not just the last — nested types of generic types
  (`TDictionary<string, Integer>.TPairEnumerator`) and generic-method calls on
  them. See §B.11 `TypeSegment`.
- *A nested alias inside a generic type belongs to the instance:* with
  `TBox<T> = class type TItems = array of T; TGetA = TFunc<T>; TL =
  TList<T>; ... end`, a `TBox<TRec>`'s `Items` walks `TRec`, `GetA()` is a
  `TRec`, `for R in BX.L` walks `TRec` - the alias is resolved and then
  closed over the instance it was reached through (dcc32 / dcc64 37.0,
  2026-10-12).
- ⚠️ *Implementation headers:* method bodies of a generic type repeat the type
  parameters in the qualified name — `function TDelegatedComparer<T>.Compare(...)`,
  `procedure TDict<K,V>.TEnum.MoveNext;`. Here `<T>` is a **parameter re-declaration**
  (introduces `T` into the body's scope), not an instantiation — the RTL's
  `Generics.Defaults`/`Collections` implementations are wall-to-wall examples. See
  ch.06 `ImplName`.
- ⚠️ *Inheriting from an instantiation closes the ancestor's members over its
  arguments, and that substitution has to survive every hop of the walk.*
  `TThingList = class(TObjectList<TThing>)` inherits `property Items[I]: T` from
  `TList<T>` two hops up; at any use site inside or outside TThingList, `Items[I]`
  is a `TThing`. Stated as semantics this is obvious, and it is *still* the
  single most common place an implementation loses precision, in four distinct
  ways worth naming separately:
  - the substitution must COMPOSE along the chain — a descendant's arguments
    apply to its ancestor's heritage reference too, so a frame that survives only
    one hop leaves `T` open at the second;
  - it must be taken at the hop the member was FOUND at, not at the type the walk
    STARTED from — the starting type is frequently the one with no arguments at
    all;
  - it must travel with a NESTED type of the generic. Such a type has no
    arguments of its own — `TList<T>` declares `arrayofT = array of T` and
    returns it from `property List` — so there is nothing to substitute IN it
    and a substitution step naturally returns it untouched; but its definition
    is written in the enclosing generic's parameters, which is precisely the
    frame the next step (indexing it) needs. The RTL's own containers are built
    this way, so `with SomeList.List[I] do` is ordinary code. It travels the
    same way when the nested type is an ARGUMENT of another instantiation:
    `TLoader<F>` declaring `TElement = record Val: F; end` and `FKeys:
    TArray<TElement>`, then in a method of `TRecLoader = class(TLoader<TRec>)`
    a bare `FKeys[0].Val.Fld` is TRec's field (`.Nope` is E2003 'Nope', a
    member of TRec), and so is `FCache[0].Value[0].Fld` through `TCache =
    TPair<string, TArray<F>>`; under `TObjCache<K; V: class> =
    class(TLoader<V>)`, `FKeys[0].Val.Free` is V's constraint's (dcc64 37.0,
    2026-10-29, PasTree audit B3-35, local/probe-park/b335 GA.dpr, GB.dpr,
    GC.dpr - the ancestor in the same unit and in another);
  - and it must be recorded WITH the binding. A resolver that records only
    "which symbol does this name denote" and re-derives the type later has thrown
    away the only thing that closes it: nothing downstream can tell which hop the
    member came from. The symptom is a member whose type reads back as the open
    parameter `T` — after which indexing it, or opening a `with` over it, yields
    nothing and the whole enclosing body reports as undeclared.
- ⚠️ *Instantiations are STRUCTURALLY compatible, not nominally distinct per use
  site* — an explicit exception to Object Pascal's otherwise-nominal type
  system (contrast the ARRAY case, ch.08 §8.1: two independently-declared array
  types of identical shape stay incompatible). Two `TList<Integer>`
  instantiations written in different units aren't merely "compatible", they
  ARE the same type: the compiler builds one concrete type per `(generic
  template, type-argument list)` pair, wherever it's first needed, and every
  later use of the same pair resolves to that one type (dcc-verified, dcc32
  37.0: a `TList<Integer>` produced by one unit's function and a
  `TList<Integer>`-typed variable declared in the caller's own unit are
  directly assignment-compatible with no cast, and `TClass(TList<Integer>) =
  TClass(TList<Integer>)` evaluated at the two separate sites is `True`). A
  resolver should key instantiated-type identity by `(genericName, typeArgs)`,
  not by declaration or use site.
  - *The arguments are compared as TYPES, weak aliases followed (2.5.1):*
    with `PInfo = PiT.PInfo` in unit PiA, `TBox<string, PiA.PInfo>` and
    `TBox<string, PiT.PInfo>` are one type - `TypeInfo` of the two is equal,
    the records assign both ways and to an alias `TBoxA = TBox<string,
    PInfo>`, and a literal `function(const X: TBox<string, PiT.PInfo>):
    Boolean` passed where `Pick(const P: TPred<TBox<string, PInfo>>)` and
    `Pick(const P: TPred<Integer>)` are overloads calls the TBox one. A
    distinct `PDist = type PiT.PInfo` argument makes another instantiation:
    `TBox<string, PDist> := TBox<string, PiT.PInfo>` is `E2010`. A resolver
    that registers an instantiation under the symbol each argument was
    WRITTEN through must compare two such entries argument by argument after
    following the aliases (spring4d's `fServices.FirstOrDefault(function(const
    item: TPair<string, PTypeInfo>) ...)` meets `Spring.PTypeInfo =
    TypInfo.PTypeInfo` on one side and TypInfo's own name on the other).
    dcc64 37.0, 2026-10-13 (PasTree local/probe-instalias Pi1, Pi2).
- *Monomorphization: one method-body copy per distinct instantiation, even for
  a T-independent method.* Each closed instantiation gets its own compiled copy
  of every method, whether or not that method's body actually mentions `T` —
  the compiler does not detect and share type-parameter-independent code
  across instantiations (dcc-verified, dcc32 37.0: a generic class with one
  method that never references `T`, instantiated with ONE type argument and
  called twice, reported `131144 bytes code`; the identical program
  instantiating it with TWO distinct type arguments instead — same total call
  count, one call per instantiation — reported `131804 bytes code`: +660 bytes
  purely from adding a second distinct instantiation). A codegen-size estimate
  built on this AST should scale with the number of distinct instantiations
  actually used, not with the number of generic declarations.
- ⚠️ *A generic class's `class constructor` runs once PER instantiated type,
  not once overall* — each closed instantiation is its own type with its own
  class-level storage, so it gets its own class-constructor run and its own
  independent `class var`s. This is the generic-specific case that
  [`15-class-mechanics-helpers.md`
  §15.1.5](15-class-mechanics-helpers.md#1515-class-constructors--destructors)'s
  "runs once at unit initialization" doesn't cover — that description is
  accurate for an ordinary, non-generic class, just incomplete for a generic
  one (dcc-verified, dcc32 37.0: a generic class's `class constructor`
  incrementing a `class var` counter, instantiated and used as both
  `TBox<Integer>` and `TBox<string>`, printed its marker TWICE — once per
  instantiation — and each instantiation's own counter independently reached
  `1`, not `2`, confirming the `class var` storage itself is also
  per-instantiation, not shared across them).
- *AST:* `GenericInstantiation { genericName, typeArgs[] }`.

---

## 16.4 Constraints

### 16.4.1 Type-parameter constraints

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Constraints restrict the types a parameter accepts and unlock operations on it.

**Grammar**

```ebnf
Constraint = "class" | "record" | "constructor" | TypeRef ;
```

**Example**

```pascal
type
  TFactory<T: class, constructor> = class
    function MakeOne: T;
  end;
  TBox<T: IComparable> = class end;       // interface constraint
  TStore<T: TPersistent> = class end;     // specific-class constraint
```

**Semantics & parsing notes**

- *Kind constraints:* `class` (a reference type), `record` (a value type),
  `constructor` (has an accessible parameterless constructor — enabling
  `T.Create`). They may combine (`T: class, constructor`).
- **What each constraint UNLOCKS on a value of type T**, which is the question a
  member lookup asks (all dcc32 37.0-probed):
  - `T: class` — **TObject's members**. `V.Free` and `V.ClassName` compile under
    a bare `class` constraint, because a reference type is a class and every
    class descends from TObject - System's, not a `TObject` the `uses`
    clause names (§11.1.1, probe R4).
  - `T: record` and a lone `T: constructor` — **nothing**. The same `V.Free` is
    `E2003 Undeclared identifier` under either.
  - `T: TBase` / `T: IFoo` — that type's members, and its ancestors'.
  - `T.Create` needs the `constructor` keyword specifically: under `T: class`
    alone it is `E2568 Can't create new instance without CONSTRUCTOR constraint
    in type parameter declaration`.
  - ⚠️ *A lone `constructor` does not make T a class.* `T.Create` - through
    the TYPE - is the one thing it unlocks: `V.Create` on a value is `E2003`,
    and so are `V.ClassName`, `V.ToString` and `T.Create.Free` (the result
    is a T, with no members); `V = nil` is `E2015`. Together with `class`, T
    has all of TObject's members (`V.Free`, `T.Create.Free` compile). Every
    class type meets it - `TObject.Create` is always inherited, so even a
    class whose own `Create` takes parameters passes (the last bullet of
    this section says what `T.Create` then runs); a non-class does not:
    `TG<TRec>` (a record) and `TG<IInterface>` are `E2513 Type parameter 'T'
    must have one public parameterless constructor named Create`. dcc64 37.0
    also accepts `TG<Integer>` - read as a compiler gap, not a rule: an
    analyzer lays an open `T: constructor` out as a class reference, the
    only kind the constraint is written for. dcc64 37.0, 2026-10-16 (PasTree
    local/probe-ctor C1-C10, I_*).
- *Type constraints:* a class type (T must descend from it) or interface (T must
  implement it). `T: TBase, IFoo` combines.
  - ⚠️ *A constraint named through an alias is the aliased type* (dcc64 37.0,
    probed 2026-10-10, `probe-b18` P2): with `TBaseAlias = TBase`,
    `TG<T: TBaseAlias>` accepts `TG<TDerived>` and `TG<TBase>`, and
    `TG<TObject>` is `E2515 Type parameter 'T' is not compatible with type
    'TBase'` - the message names the real class, not the alias. A checker
    that compares the ancestry walk against the alias symbol rejects even the
    exact class.
  - ⚠️ *Which constraint lists are legal, and the error for each that is
    not* (dcc64 37.0, 2026-10-30, spec audit g5,
    local/probe-park/spec-audit/g5 G02.dpr, G02b.dpr):
    - `class, record` - `E2507 CLASS constraint and RECORD constraint cannot
      be specified together`; `record, constructor` - `E2516 Constraint
      'constructor' is conflicting with previous constraints`; `record, IFoo`
      - `E2666 Constraint 'record' and interface type constraint cannot be
      specified together`; `class, TBase` - `E2552 CLASS or RECORD constraint
      and class type constraint cannot be specified together`.
    - Repeats: `TBase, TBase` - `E2553 Class type constraint cannot be
      specified more than once`; `IFoo, IFoo` - `E2670 Identical interface
      type constraint cannot be specified more than once`; `constructor,
      constructor` - `E2603 Constraint 'constructor' cannot be specified
      more than once`.
    - Not a constraint type: `Integer` and `TObject` (alone or after another
      class, `TBase, TObject`) are `E2510 Type 'TObject' is not a valid
      constraint` - write `class` for "any class"; `string` is a keyword, so
      `<T: string>` is a syntax error (`E2029 Identifier expected`).
    - Legal: `class, IFoo`, `TBase, IFoo, constructor`, an interface alone
      (`IInterface` included), a class type alone (`TInterfacedObject`), and
      another parameter of the same list (`<T: TBase; U: T>`).
  - *What each kind rejects at instantiation:* `class` - `E2511 Type
    parameter 'T' must be a class type` for an interface or a record;
    `record` - `E2512 Type parameter 'T' must be a non-nullable value type`
    for `string` or a class (`Integer` is accepted); an interface type -
    `E2514 Type parameter 'T' must support interface 'IFoo'` for a class not
    implementing it (an interface type argument equal to the constraint is
    accepted); a class type - `E2515` (above). dcc64 37.0, 2026-10-30 (spec
    audit g5, local/probe-park/spec-audit/g5 G06.dpr).
  - ⚠️ *Combining means the UNION of what each unlocks, and a member lookup has
    to try them ALL.* `TKey: IComparable<TKey>, IEquatable<TKey>, IHashable`
    makes `AKey.Compare`, `AKey.Equals` and `AKey.Hash` equally legal, and a
    resolver that takes the first constraint answers only for the first
    interface. The failure is quiet in the usual way: the constraint it did
    consult has SOME members, so the walk succeeds often enough to look
    correct, and the reports land on whichever names happen to live on the
    second and third.
- ⚠️ Constraints are written **once, on the declaration**. A method body may not
  repeat them — `procedure TList<T>.Sort;` carries a bare `<T>` — so a tool that
  reads constraints off the parameter it finds must follow the body's parameter
  back to the declaring type's (for a generic METHOD's own parameters, back to
  the method's declaration - 16.2.1). Repeating one is a syntax error:
  `procedure TL<T: class>.Sort;` is `E2029 ',', ';' or '>' expected but ':'
  found`. dcc64 37.0, 2026-10-30 (spec audit g5,
  local/probe-park/spec-audit/g5 G08.dpr).
- *A generic METHOD's constraints are checked at an explicit
  instantiation like a type's* - `F.Pick<TObject>(O)` against `Pick<T:
  TItem>` is `E2515 Type parameter 'T' is not compatible with type 'TItem'`,
  `TCaster.Cast<Integer>(O)` against `Cast<T: class>` is `E2511`, and
  `F.Pick<TSub>(O)` with TSub a TItem descendant compiles. dcc32 / dcc64
  37.0, 2026-10-13 (PasTree audit B41, E2-34; local/probe-b41 GM).
- ⚠️ `class`/`record`/`constructor` here are **constraint keywords**, not type
  declarations — parse within the `GenericParams` constraint list, not as a class
  body.
- ⚠️ *The `constructor` constraint is not enforced against the ACTUAL type
  argument at instantiation — it only unlocks the `T.Create` syntax, it does
  not verify `T` genuinely has an accessible parameterless constructor of its
  own.* If the bound type's own `Create` takes required parameters (so a
  DIRECT, non-generic call to a parameterless `Create` on it would fail to
  compile), `T.Create` inside the generic body still compiles and, at run
  time, silently resolves to the nearest ancestor's parameterless constructor —
  ordinarily `TObject.Create` — leaving that type's own fields exactly as they
  were before allocation, never touched by its real constructor
  (dcc-verified, dcc32 37.0: `TFoo<T: constructor>` calling `T.Create` inside a
  method, instantiated as `TFoo<TNoDefaultCtor>` where `TNoDefaultCtor`
  declares only `constructor Create(AValue: Integer)`: the call compiles and
  runs, returning a correctly-classed/-sized `TNoDefaultCtor` instance whose
  field reads back as `0` — i.e. `TObject.Create`'s body ran, not
  `TNoDefaultCtor.Create`'s. A direct, non-generic `TNoDefaultCtor.Create;`
  with no arguments, by contrast, fails to compile with `E2035 Not enough
  actual parameters` — confirming the generic path is genuinely more
  permissive than a direct call, not just resolving to the same thing). A
  type-checker that rejects `TFoo<TNoDefaultCtor>` outright for failing the
  constraint would be STRICTER than dcc32 actually is here — match the
  compiler's leniency, don't "fix" it. Same on dcc64 37.0 (2026-10-30, spec
  audit g5, local/probe-park/spec-audit/g5 G01.dpr: `TG<TNoDef>.Make`
  prints `TNoDef 0`, the own `Create(A)` never ran).
- *AST:* per-parameter `constraints: [ kind | typeRef ]`.

---

## 16.5 Type inference

### 16.5.1 Inference for generic methods & inline vars

| | |
|---|---|
| **Introduced** | generic-method inference Delphi 2009; inline-var inference 10.3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Type arguments to a **generic method** can be inferred from value arguments; an
**inline `var`** infers its type from the initializer (ch.03 §3.1.3).

**Example**

```pascal
var X := Max(3, 7);          // T inferred as Integer, X inferred as Integer
```

**Semantics & parsing notes**

- ⚠️ *No inference for generic **types*** — `TList.Create` cannot infer `T`; you
  must write `TList<Integer>.Create`. Only generic **methods** infer. Enforce in
  the resolver.
- ⚠️ *Type arguments WRITTEN at the call site are the answer; inference only
  supplies what was not written.* `Cast<TSomeEdit>(AObject)` says what `T` is,
  and it has to, because the common shape of an explicitly-instantiated method
  is one whose parameters mention `T` nowhere at all:

  ```pascal
  class function Cast<T: class>(AObject: TObject): T; static;   // nothing to infer from
  ```

  A resolver that reaches for inference here binds nothing, leaves the call
  typed as the OPEN parameter, and loses every member after the dot. Worth
  saying plainly because the argument-driven half of this rule is the one
  everybody implements first.
- ⚠️ *A call written `Name<...>` cannot mean a NON-generic routine of that name*
  — §16.1.2's arity rule, on the method side - but the arity filter works
  only WITHIN one overload set. When a derived class declares a plain
  `GetValue(const AName: string): TBaseJ; overload;` in front of an
  ancestor's `GetValue<T>(const APath: string): T`, the inherited generic
  joins the candidate set and the written arity picks it:
  `D.GetValue<TItem>('x').Site := 1` compiles (System.JSON's
  `TJSONObject.GetValue` is shaped exactly like that). WITHOUT `overload`
  on the derived declaration, ordinary hiding applies: the plain method is
  the only candidate, and `Name<...>(...)` reads as two comparisons -
  `E2035 Not enough actual parameters` and `E2018 Record, object or class
  type required` - whether the call is qualified (`D.GetValue<TItem>('x')`),
  through `Self`, or unqualified inside a method. dcc64 37.0, 2026-10-31
  (PasTree audit E1-33, local/probe-park/specfu/E1-33 H1.dpr, H2.dpr prints
  `ok`).

  The same rule decides between OVERLOADS in one type. A plain and a generic
  method may share a name and a parameter list, differing only in the type
  parameters:

  ```pascal
  TVirtualNode = packed record
    function GetData(): Pointer; overload;
    function GetData<T>(): T; overload;
  end;
  ```

  `Node.GetData<TItem>` (with or without `()`) is the generic one and a
  `TItem`; bare `Node.GetData` is the plain one and a `Pointer`
  (`Node.GetData.Site` is E2671). Declaration order does not matter either
  way (dcc64 37.0, probed 2026-09-25, both orders). So written type arguments
  FILTER the candidates before value arguments are looked at: only generic
  methods with that many type parameters remain. A resolver that picks by
  value arguments alone sees two parameterless candidates, takes the first
  (the `Pointer` one), and the written `<TItem>` then has nothing to
  instantiate.
- ⚠️ *What inference can and cannot do* (dcc64 37.0, 2026-10-30, spec audit
  g5, local/probe-park/spec-audit/g5 G03.dpr, G03b.dpr, G03c.dpr):
  - `T` only in the result (`Make<T>: T`) - `I := TU.Make;` is `E2531
    Method 'Make' requires explicit type argument(s)`; the assignment target
    is not consulted.
  - Arguments of different types bound to one `T` do not unify to a common
    type: `Max(3, 7.5)` against `Max<T>(const A, B: T)` is `E2532 Couldn't
    infer generic type argument from different argument types for method
    'Max'`, although Integer converts to the real type; `Max(1, 'a')` the
    same. `Max(nil, nil)` infers `Pointer`.
  - Inference sees `T` directly or as an open array's element: `First([1,
    2])` against `First<T>(const A: array of T)` infers Integer. It does NOT
    look inside another generic's instantiation: `FirstD(A)` with `A:
    TArray<Integer>` against `FirstD<T>(const A: TArray<T>)` is `E2010
    Incompatible types: 'TArray<T>' and 'TArray<Integer>'`, and `FirstL(L)`
    with a `TList<Integer>` against `FirstL<T>(L: TList<T>)` likewise - the
    caller writes `FirstD<Integer>(A)`. A resolver that unifies `TArray<T>`
    with `TArray<Integer>` accepts calls dcc rejects.
- ⚠️ *A plain and a generic overload that both accept the arguments* (dcc64
  37.0, 2026-10-30, spec audit g5, local/probe-park/spec-audit/g5
  G03.dpr, G04.dpr): the generic one is instantiated with the argument's own
  type and then competes as an ordinary candidate (6.3.1). With `P(A:
  Integer)` and `P<T>(A: T)`, `P(1)` calls the plain one (both exact, the
  non-generic wins the tie) and `P('s')` the generic; `R('x')` calls `R(const
  A: string)` over `R<T>(const A: T)`. With `P(A: Int64)` instead, `P(B)` for
  `B: Byte` and `P(1)` call the GENERIC one - its exact match beats the plain
  one's widening. `P<Integer>(1)`, type arguments written, is always the
  generic (the filter above).
- Inference combines with inline-var inference for terse code.

---

## 16.6 Covariant results via generics

### 16.6.1 Generic methods returning a derived type

| | |
|---|---|
| **Introduced** | Delphi 2009 (pattern) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Object Pascal lacks general return-type covariance, but a generic method with a
constrained `T` can return the caller's specific type.

**Example**

```pascal
function Clone<T: TAnimal, constructor>(Src: T): T;
```

**Semantics & parsing notes**

- This is a *pattern*, not a separate language feature — it falls out of generic
  methods + constraints. Listed here because parser/type-checker consumers often
  ask whether Object Pascal supports return-type covariance: the answer is "not
  directly; use this pattern".
