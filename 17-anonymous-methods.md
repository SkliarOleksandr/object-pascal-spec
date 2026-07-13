# 17 — Anonymous Methods & Closures

Inline routine literals that capture their surrounding variables, and the
`reference to` type family that holds them.

Procedural types recap → [ch.06 §6.6](06-routines.md#66-procedural-types).

## Chapter grammar umbrella

```ebnf
AnonMethodType = "reference" "to" ( "procedure" | "function" )
                 [ "(" FormalParams ")" ] [ ":" ResultType ] ;
AnonMethodExpr = ( "procedure" | "function" )
                 [ "(" FormalParams ")" ] [ ":" ResultType ]
                 Block ;                                  (* a literal, used as an expression *)
```

---

## 17.1 `reference to` types

### 17.1.1 Anonymous-method reference types

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A type that can hold an anonymous method, a standalone routine, or a method —
implemented as an auto-generated interface (hence reference-counted).

**Example**

```pascal
type
  TProc1 = reference to procedure(X: Integer);
  TFunc1 = reference to function(const S: string): Integer;
```

**Semantics & parsing notes**

- ⚠️ Distinct from `of object` method pointers (ch.06): a `reference to` value is an
  **interface-backed closure** that can **capture local state**, is
  reference-counted, and is *not* a simple `(code, Self)` pair. Keep the kind on
  the type node — assignment compatibility differs.
- *AST:* `AnonMethodType { kind: proc|func, params[], resultType? }`.

---

## 17.2 Anonymous method literals

### 17.2.1 Inline `procedure`/`function` literals

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A `procedure`/`function` with a body but **no name**, used as an expression value.

**Example**

```pascal
var P: TProc1;
P := procedure(X: Integer)
     begin
       Writeln(X);
     end;
P(42);
```

**Semantics & parsing notes**

- ⚠️ *Headed expression with embedded `Block`:* the parser must, in **expression
  position**, recognise `procedure`/`function` followed by an optional param list,
  optional result, and a full `Block` (`begin..end`) — producing an
  **expression**, not a declaration. This dual role of the `procedure`/`function`
  keywords is a key parsing branch (declaration vs. literal) decided by context.
- ⚠️ *Closing parenthesis after `end`:* when an anonymous method is passed as a
  call argument, the call's closing `)` (and any following arguments) come
  **after** the literal's `end`. The parser must keep the argument-list nesting
  open across the whole embedded `Block` — a classic source of confusing syntax
  errors when hand-written parsers close the call too early.
- *AST:* `AnonMethod { kind, params[], resultType?, body, captured[] }`.

---

## 17.3 Variable capture (closures)

### 17.3.1 Capturing enclosing variables

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

An anonymous method captures the **variables** (not just values) of its enclosing
scope; their lifetime is extended to match the closure.

**Example**

```pascal
function MakeCounter: TFunc<Integer>;
var N: Integer;
begin
  N := 0;
  Result := function: Integer
            begin
              Inc(N);          // captures N by reference
              Result := N;
            end;
end;
```

**Semantics & parsing notes**

- ⚠️ *Capture is by reference to the variable*, not by value-copy: all closures
  sharing a captured variable see each other's mutations. The compiler hoists
  captured locals into a heap frame whose lifetime is tied to the closure's
  ref count.
- ⚠️ *Loop-variable capture gotcha:* capturing a `for`-loop counter captures the
  single shared variable — a frequent bug. A semantic-analysis warning candidate.
- ⚠️ *Own scope, own `Result`:* an anonymous method body owns its locals and its
  OWN implicit `Result` (typed by the literal's result type). Two sibling
  literals may declare the same local name (dcc-verified:
  System.JSON.Serializers has two adjacent literals each with `var LSer`), and
  `Result :=` inside a `function: Boolean` literal binds to the LITERAL's
  Boolean result even when the enclosing function returns `string`
  (System.SysUtils TraverseDirectory callbacks). The enclosing routine's
  `Result` is NOT capturable (dcc E2555 "Cannot capture symbol 'Result'") — a
  resolver that lets the enclosing `Result` leak into the literal's body
  produces false E2010.
- The resolver must compute the **capture set** (which enclosing locals/`Self` are
  referenced) to model lifetime; record it on the AST node.
- *AST:* `captured[]` listing captured symbols on the `AnonMethod`.
