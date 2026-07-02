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
