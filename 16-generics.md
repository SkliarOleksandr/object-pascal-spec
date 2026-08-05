# 16 — Generics

Parameterized types and methods, constraints, instantiation, and the
`<`-disambiguation that is the single hardest parsing problem in the language.

Shared productions (`TypeRef`, `GenericArgs`) → [Appendix B
§B.11](B-lexical-grammar.md#b11-type-references).

## Chapter grammar umbrella

```ebnf
GenericParams = "<" TypeParamDecl { ";" TypeParamDecl } ">" ;
TypeParamDecl = IdentList [ ":" ConstraintList ] ;
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
    declaration*, and then it is not in any unit's scope at all: spring4d
    declares `Pointer<T> = record` beside the compiler's own `Pointer`, so the
    search for a matching arity has to reach the predefined names too, not only
    the current unit and its imports. Every `Pointer(X) := nil` in that unit
    depends on it.
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
  - In a **type position** (after `:` in a declaration, in `uses`-free type refs,
    after `class of`), `Id<…>` is always generic args.
  - In an **expression position**, the compiler scans ahead: if the tokens between
    `<` and a matching `>` parse as a comma-separated **type list** *and* the token
    after `>` is `(` , `.` , or another expected continuation, treat it as a generic
    instantiation; otherwise treat `<` as comparison.
  - The construct `TList<Integer>.Create` works because `>` is followed by `.`.
  - *Refined follower rule (corpus-validated by PasTree):* accept generic args
    whenever the token after `>` **cannot start an operand** (`;` `)` `]` `,`
    `=` `then` `do` `and`…). Provably safe: the comparison reading
    `(a < X) > ⟨follower⟩` would lack a right operand — a guaranteed syntax
    error — so no valid comparison is ever stolen. Real cases:
    `Value.AsType<TBytes>;`, `AsType<char> = #0`, `if X.AsType<Boolean> then`,
    `@Proc<T>;`. `(` remains the genuinely ambiguous follower and reads as a
    call, matching dcc.
- Practical implementations keep a speculative parse / backtracking path for this.
  **Document the heuristic in the parser** — this is where most hand-written
  Object Pascal parsers get it wrong.
- ⚠️ *Args on every dotted segment:* generic arguments may appear on **intermediate**
  segments of a qualified name, not just the last — nested types of generic types
  (`TDictionary<string, Integer>.TPairEnumerator`) and generic-method calls on
  them. See §B.11 `TypeSegment`.
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
    this way, so `with SomeList.List[I] do` is ordinary code;
  - and it must be recorded WITH the binding. A resolver that records only
    "which symbol does this name denote" and re-derives the type later has thrown
    away the only thing that closes it: nothing downstream can tell which hop the
    member came from. The symptom is a member whose type reads back as the open
    parameter `T` — after which indexing it, or opening a `with` over it, yields
    nothing and the whole enclosing body reports as undeclared.
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
- *Type constraints:* a class type (T must descend from it) or interface (T must
  implement it). `T: TBase, IFoo` combines.
- ⚠️ `class`/`record`/`constructor` here are **constraint keywords**, not type
  declarations — parse within the `GenericParams` constraint list, not as a class
  body.
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
