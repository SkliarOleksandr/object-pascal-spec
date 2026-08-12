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
  an error: `E2145 Re-raising an exception only allowed in exception handler`.
- ⚠️ *What "inside a handler" means* is **lexical**, and the deciding scope is the
  **nearest enclosing part of a `try` statement**, not any enclosing one. A `try`
  block or a `finally` part therefore *resets* the context an outer handler
  established (dcc32 37.0):

  ```pascal
  try except try finally raise; end; end;   // error — nearest part is `finally`
  try finally try except raise; end; end;   // legal — nearest part is `except`
  try except try raise; except end; end;    // error — nearest part is a try block
  try try except raise; end; finally end;   // legal
  ```

  Both the `on … do` bodies and the `else` branch of an `except` part count as
  handler context. An **anonymous method body does not reset it** — a bare
  `raise` written inside a `procedure begin … end` in a handler is accepted —
  which confirms the try-statement part is the *only* boundary. A named nested
  routine needs no separate rule: its body is never lexically inside a
  statement, so no part is in effect there and the `raise` is an error even when
  the routine is called from a handler.
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
- ⚠️ *`BaseException` is the FIRST (innermost) exception of the chain, distinct from
  `InnerException` once the chain is more than one level deep* — `InnerException`
  is only the immediately-preceding exception. With a single nesting they happen to
  be the same object; with two or more they diverge (dcc-verified, dcc32 37.0):

  ```pascal
  try
    raise Exception.Create('Hello');
  except
    try
      Exception.RaiseOuterException(Exception.Create('Another'));
    except
      Exception.RaiseOuterException(Exception.Create('A third'));
    end;
  end;
  // caught E: E.Message           = 'A third'
  //          E.InnerException.Message  = 'Another'   (one level up)
  //          E.BaseException.Message   = 'Hello'      (first/innermost)
  ```
- ⚠️ *`Exception.ToString` concatenates the whole chain's messages*, one per line
  (`sLineBreak`-joined, outermost first), rather than just the current exception's
  `Message` — for the chain above, `E.ToString` is `'A third'` + LF + `'Another'` +
  LF + `'Hello'` (dcc-verified). With no chaining, `ToString` equals `Message`.
- *AST:* not new syntax — `BaseException`/`ToString` are RTL members reached
  through ordinary member access on an `Exception`-typed expression.

### 18.5.2 Intercepting a raise: `Exception.RaisingException`

| | |
|---|---|
| **Introduced** | Delphi 2009 (alongside inner-exception chaining) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`Exception.RaisingException(P: PExceptionRecord)` is a `virtual` hook invoked on the
exception object right after it is constructed but **before** the `raise` actually
transfers control — overriding it gives a single post-creation function that runs
regardless of which constructor was used.

**Example**

```pascal
type
  ECustomException = class(Exception)
  protected
    procedure RaisingException(P: PExceptionRecord); override;
  end;

procedure ECustomException.RaisingException(P: PExceptionRecord);
begin
  Log(Message);          // fires before any `except` handler runs
  inherited;              // required: base impl wires up SetInnerException
end;
```

**Semantics & parsing notes**

- ⚠️ *Fires before the handler, not at construction* — the marker code in an
  overridden `RaisingException` runs strictly between `raise`'s object-construction
  step and the stack search for a handler; ordinary `Create` does not trigger it
  (dcc-verified: overriding `RaisingException` and calling `raise
  ECustomException.Create(...)` prints the override's output before the enclosing
  `except` block's own code runs).
- The base-class implementation calls the internal `SetInnerException` — an
  override that skips `inherited` breaks inner-exception chaining (18.5.1) for that
  exception type.
- Not new syntax — an ordinary virtual method override; included here because it
  changes exception-raising *semantics* (a guaranteed post-construction/pre-raise
  hook point), not because it affects parsing.

---

## 18.6 Exceptions during construction & destruction

### 18.6.1 A failing `Create` still runs `Destroy` (and the pseudo-hooks)

| | |
|---|---|
| **Introduced** | Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

If a constructor raises before returning, the runtime immediately invokes the
destructor on the **partially-initialized** object before the exception continues
propagating — the object is never silently discarded without cleanup. The same
non-resumption rule extends to `AfterConstruction`/`BeforeDestruction`: if
`AfterConstruction` raises, `BeforeDestruction` and the regular destructor still run.

**Example**

```pascal
type
  TObjectWithList = class
  private
    FStringList: TStringList;
  public
    constructor Create(Value: Integer);
    destructor Destroy; override;
  end;

constructor TObjectWithList.Create(Value: Integer);
begin
  if Value < 0 then
    raise Exception.Create('Negative value not allowed');  // FStringList never set
  FStringList := TStringList.Create;
end;

destructor TObjectWithList.Destroy;
begin
  if Assigned(FStringList) then      // ⚠️ mandatory guard — see notes
    FreeAndNil(FStringList);
  inherited;
end;
```

**Semantics & parsing notes**

- ⚠️ *A destructor must never assume the constructor ran to completion.* Every
  sub-object/resource field a destructor touches needs an `Assigned` (or
  equivalent) guard, because `Destroy` runs even when `Create` raised partway
  through — the destructor here can be reached with `FStringList` still `nil`
  (dcc-verified, dcc32 37.0: a constructor that sets one field, raises, then
  would have set a second field — the destructor is still entered and observes
  the first field set and the second unset).
- The caller never calls `Free` on an object whose `Create` raised: `raise`
  already triggered `Destroy` for it, and the assignment (`Obj := TFoo.Create`)
  never completes, so there is no reachable reference to free again. This is why
  the standard idiom creates the object *outside* a `try…finally` — construction
  is self-protecting; only post-construction use needs the `finally Obj.Free`.
- ⚠️ *Extends to the pseudo-hooks:* if an override of `AfterConstruction` raises,
  `BeforeDestruction` and `Destroy` both still run on the object (dcc-verified:
  overriding `AfterConstruction` to raise, with `BeforeDestruction` overridden to
  emit a marker — the marker fires, and the exception then propagates out of
  `Create` normally). `AfterConstruction`/`BeforeDestruction` are TObject's
  C++-compatibility pseudo-constructor/pseudo-destructor pair; this is not new
  syntax, but the same lifecycle guarantee applies to them as to `Create`/`Destroy`.
- This is fundamentally an exception-propagation rule rather than an ordinary
  control-flow one: the destructor call is *inserted* by the runtime as part of
  unwinding a failed construction, not written at the call site. See
  [ch.11](11-classes.md) for the general `Create`/`Destroy`/`AfterConstruction`/
  `BeforeDestruction` lifecycle model.
- *AST:* not new syntax — no parser impact; this is a runtime/codegen guarantee
  attached to every constructor call.
