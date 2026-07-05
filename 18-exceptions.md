# 18 — Exceptions

Structured exception handling: `try…except`, `try…finally`, `raise`, exception
filters, re-raising, and nested/inner exceptions.

Shared productions (`StatementList`, `Expression`, `Block`) → [Appendix
B](B-lexical-grammar.md). These are the `TryStmt`/`RaiseStmt` deferred from
[ch.05](05-statements.md).

## Chapter grammar umbrella

```ebnf
TryStmt     = "try" StatementList ( ExceptPart | FinallyPart ) "end" ;
FinallyPart = "finally" StatementList ;
ExceptPart  = "except" ( StatementList
                       | ExceptionHandler { ExceptionHandler } [ "else" StatementList ] ) ;
ExceptionHandler = "on" [ Ident ":" ] TypeRef "do" Statement ";" ;
RaiseStmt   = "raise" [ Expression [ "at" Expression ] ] ;
```

---

## 18.1 `try…except`

### 18.1.1 The `try…except` block

| | |
|---|---|
| **Introduced** | Delphi 1 (1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Executes guarded statements; if an exception is raised, control transfers to the
`except` part.

**Example**

```pascal
try
  DoRisky;
except
  HandleAnything;     // catch-all form
end;
```

**Semantics & parsing notes**

- ⚠️ *Two `except` shapes:* either a bare `StatementList` (catch-all) **or** a list
  of `on … do` handlers (optionally with `else`). The parser must decide by
  looking for the `on` keyword as the first token of the `except` body.
- ⚠️ `try` blocks **cannot** have both `except` and `finally` directly — nest them
  (`try try … finally … end except … end`). Reject a single `try` with both parts.
- *AST:* `TryExceptStmt { body, handlers[], elseBody?, catchAll? }`.

### 18.1.2 `on E: EType do` exception filters

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Typed handlers selected by the exception's class; an optional identifier binds
the exception instance.

**Example**

```pascal
try
  Parse;
except
  on E: EConvertError do ShowMessage(E.Message);
  on E: EAccessViolation do Log(E);
else
  raise;              // unmatched: re-raise
end;
```

**Semantics & parsing notes**

- ⚠️ *Handler order matters:* handlers are tried top-to-bottom; a base-class handler
  placed before a derived one shadows it. The resolver/analyzer may warn; semantics
  follow source order.
- The bound identifier `E` is scoped to that handler's `Statement`; the instance is
  **freed automatically** when the handler exits (do **not** `Free` it).
- `else` (no type) catches anything unmatched.
- *AST:* `ExceptOn { varName?, excType, body }`.

---

## 18.2 `try…finally`

### 18.2.1 The `try…finally` block

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

The `finally` part **always** runs — on normal completion, on exception, or on
early `Exit`/`Break` — for guaranteed cleanup.

**Example**

```pascal
Obj := TThing.Create;
try
  Obj.Use;
finally
  Obj.Free;           // runs no matter what
end;
```

**Semantics & parsing notes**

- ⚠️ `finally` does **not** swallow the exception — it runs, then the exception
  continues propagating unless the block itself raises/exits. Distinguish from
  `except`.
- *AST:* `TryFinallyStmt { body, finallyBody }`.

---

## 18.3 `raise`

### 18.3.1 Raising and re-raising

| | |
|---|---|
| **Introduced** | Delphi 1; `RaiseOuterException`/inner 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`raise E` throws an exception object; a **bare** `raise` inside a handler
re-raises the current exception.

**Grammar**

```ebnf
RaiseStmt = "raise" [ Expression [ "at" Expression ] ] ;
```

**Example**

```pascal
raise EMyError.Create('boom');     // raise new
// inside an except handler:
raise;                              // re-raise current (preserves stack/origin)
raise E at ReturnAddress;           // raise at a specific address
```

**Semantics & parsing notes**

- ⚠️ *Bare `raise`* is only valid **inside an exception handler** — it re-throws the
  in-flight exception, preserving its original stack trace. Outside a handler it is
  an error. The analyzer must track handler context.
- `raise` **transfers ownership** of the exception object to the RTL — do not free
  the raised instance.
- `at Address` overrides the reported raise location (diagnostics).
- *AST:* `RaiseStmt { exception?, atAddr? }` (no `exception` ⇒ re-raise).

---

## 18.4 Exception hierarchy

### 18.4.1 The `Exception` base class

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

All catchable exceptions derive from `Exception` (RTL); `on E: Exception` catches
the broadest meaningful set.

**Semantics & parsing notes**

- A language-contract class (like `TObject`): the `on` filter type is normally an
  `Exception` descendant. Non-`Exception` objects can technically be raised but are
  outside normal handling.
- *Not parsing* — informs the type-checker's expectations for `on … do` types.

---

## 18.5 Nested & inner exceptions

### 18.5.1 Inner-exception chaining

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Raising a new exception while handling another preserves the original as the
**inner exception** (`Exception.InnerException`), via `Exception.RaiseOuterException`
/ `ThrowOuterException`.

**Example**

```pascal
try
  LowLevel;
except
  Exception.RaiseOuterException(EHighLevel.Create('wrapped'));
end;
```

**Semantics & parsing notes**

- This is an RTL mechanism (method calls), not new syntax — included because it
  changes the *semantics* of nested raises (chaining vs. replacing). No parser
  impact beyond ordinary calls.
- The compiler tracks the "current exception" per thread so nested `try` blocks and
  bare `raise` behave correctly.
