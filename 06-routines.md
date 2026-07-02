# 06 — Procedures & Functions

Routine declarations, the full parameter-passing model (the part with the most
parser-relevant detail), overloading, inlining, calling conventions, procedural
types, and external/`noreturn` declarations.

Shared productions (`Block`, `TypeRef`, `Ident`, `ConstExpr`) → [Appendix
B](B-lexical-grammar.md). Anonymous methods → [ch.17](17-anonymous-methods.md).

## Chapter grammar umbrella

```ebnf
ProcedureDeclSection = ProcHeading ";" [ Directives ]
                       ( Block ";" | AsmBlock ";" | ForwardOrExternal ) ;
ProcHeading = ( "procedure" | "function" ) [ ImplName ]
              [ "(" [ FormalParams ] ")" ]
              [ ":" ResultType ] ;             (* ResultType only for function *)
ImplName    = Ident [ GenericArgs ] { "." Ident [ GenericArgs ] } ;
              (* implementation headers of generic types carry the type params
                 per segment: procedure TList<T>.Add / TDict<K,V>.TEnum.MoveNext *)
FormalParams = FormalParam { ";" FormalParam } ;
FormalParam  = [ AttributeGroup ] [ ParamModifier ] [ AttributeGroup ] IdentList
               [ ":" ParamType ] [ "=" ConstExpr ] ;
               (* [Ref] may precede or follow the modifier:
                  "[Ref] const X: T" and "const [Ref] X: T" are both legal —
                  see §6.2.3 / ch.19 §19.3.3 *)
ParamModifier = "var" | "const" | "out" ;
```

---

## 6.1 Routine declaration

### 6.1.1 Procedures & functions

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`procedure` (no result) and `function` (returns a value via the implicit
`Result` variable or the routine name).

**Example**

```pascal
procedure Log(const Msg: string);
begin Writeln(Msg); end;

function Add(A, B: Integer): Integer;
begin Result := A + B; end;
```

**Semantics & parsing notes**

- ⚠️ *`Result` is an implicit local* in every function (since Delphi 2). Assigning
  the **function name** (`Add := …`) is the classic-Pascal equivalent and still
  legal — the resolver must treat the routine name as an alias of `Result` inside
  the body. `Exit(value)` also sets it (ch.05 §5.6.3).
- A function whose `Result` is never assigned returns an undefined value (managed
  types excepted) — a hint/warning, not a parse error.
- *AST:* `RoutineDecl { kind, name, params[], resultType?, directives[], body? }`.

### 6.1.2 Forward declarations

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

`forward` declares a routine's signature now and defines its body later in the
same unit — enabling mutual recursion.

**Grammar**

```ebnf
ForwardOrExternal = "forward" ";" | ExternalDecl ;
```

**Semantics & parsing notes**

- The later definition may omit the parameter list (it is taken from the forward
  declaration). `interface`-section routine headers are implicitly forward (ch.01).
- *AST:* link the forward header to its later body; one logical routine.

---

## 6.2 Parameters

> The parameter modifier determines **how the argument is bound** (by value, by
> reference, read-only) — central to both semantics and codegen. The parser
> records the modifier; the type-checker enforces lvalue/const rules.

### 6.2.1 Value parameters

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

No modifier — the argument is **copied**; modifications inside don't affect the
caller.

```pascal
procedure P(X: Integer);   // X is a local copy
```

**Semantics & parsing notes**

- For managed types (strings, dynamic arrays, interfaces), the copy is reference-
  counted, not a deep copy.

### 6.2.2 `var` (reference) parameters

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Pass by reference; the callee reads and writes the caller's variable.

```pascal
procedure Swap(var A, B: Integer);
```

**Semantics & parsing notes**

- ⚠️ The argument **must be an assignable lvalue** of the *exact* type (no implicit
  conversion) — enforce in the type-checker.

### 6.2.3 `const` parameters (and `const [Ref]`)

| | |
|---|---|
| **Introduced** | `const` Delphi 1; `[Ref]` attribute ~XE2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`const` = read-only parameter; the compiler may pass by reference for efficiency.
The `[Ref]` attribute forces by-reference passing.

```pascal
function Hash(const Data: TBytes): Cardinal;
procedure Draw(const [Ref] R: TRect);      // guaranteed by-reference
```

**Semantics & parsing notes**

- Assigning to a `const` parameter inside the body is a **compile error**.
- `const [Ref]` guarantees reference passing (relevant for large records /
  interop). `[Ref]` is a compiler-recognized **parameter attribute**
  (ch.19 §19.3.3) and may be written **before or after** `const`
  (`[Ref] const X` / `const [Ref] X`). The RTL uses it heavily in COM/DirectX
  headers (`const [Ref] ppResources: ID3D11Resource`).
- ⚠️ *Not Delphi:* FreePascal's `constref` keyword is the FPC equivalent of
  `const [Ref]` — it is **not accepted by the Delphi compiler** (the D13 sources
  contain it only inside `{$IFDEF FPC…}` branches, e.g. `System.Skia.pas`). A
  Delphi parser must reject it.

### 6.2.4 `out` parameters

| | |
|---|---|
| **Introduced** | Delphi 3 (COM-era) |
| **Deprecated** | — |
| **Status** | ✅ Current |

By-reference output parameter; the incoming value is not used (managed types are
cleared on entry).

```pascal
procedure GetSize(out W, H: Integer);
```

**Semantics & parsing notes**

- `out` is a **directive** (B.4.2), keyword only in this position. Like `var`, the
  argument must be an lvalue.

### 6.2.5 Default (optional) parameters

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A trailing value parameter may specify a default constant, making it optional.

```pascal
procedure Show(Msg: string; Modal: Boolean = True);
```

**Semantics & parsing notes**

- ⚠️ *Trailing-only rule:* once one parameter has a default, **all following**
  parameters must too. Defaults must be **constant expressions** and are only
  allowed on value/`const` parameters (not `var`/`out`). Enforce both.
- Interacts with overloading (6.3) — ambiguous calls are a semantic error.

### 6.2.6 Open array & `array of const` parameters

| | |
|---|---|
| **Introduced** | open arrays Delphi 1; `array of const` Delphi 1/2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`array of T` as a parameter type accepts arrays of any length; `array of const`
accepts a heterogeneous list (variant-like `TVarRec`).

**Grammar**

```ebnf
ParamType = TypeRef
          | "array" "of" ( TypeRef | "const" )      (* open array / array of const *)
          | "string" | "file" ;                      (* untyped open string/file params *)
```

**Example**

```pascal
function Sum(const A: array of Integer): Integer;
procedure Log(const Fmt: string; const Args: array of const);
```

**Semantics & parsing notes**

- ⚠️ *`array of T` as a parameter ≠ a dynamic array type.* It is an **open array
  parameter** — distinct grammar and ABI (passes a pointer + high bound). Use
  `Low`/`High`/`Length` inside. Distinguish from a named dynamic-array type
  argument.
- `array of const` builds a `TVarRec` array; the literal `[a, 'b', 3]` at the call
  site is an open-array constructor, not a set.

### 6.2.7 Untyped parameters

| | |
|---|---|
| **Introduced** | Pascal/Turbo |
| **Deprecated** | — |
| **Status** | ⚠️ Legacy |

A `var`/`const`/`out` parameter with **no type** accepts any type.

```pascal
procedure FillZero(var Buf; Count: Integer);
```

**Semantics & parsing notes**

- Only `var`/`const`/`out` parameters may be untyped (a bare value param needs a
  type). Inside, the parameter is typeless and usually reinterpreted via a cast or
  `absolute`.

---

## 6.3 Overloading

### 6.3.1 The `overload` directive

| | |
|---|---|
| **Introduced** | Delphi 4 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Multiple routines may share a name if marked `overload` and distinguished by
parameter signature.

```pascal
function Area(R: TRect): Integer; overload;
function Area(C: TCircle): Double; overload;
```

**Semantics & parsing notes**

- `overload` is a **directive**. Resolution picks the best match by argument types
  (with implicit-conversion ranking); ambiguity is a compile error.
- ⚠️ Overload resolution interacts with **default parameters** and **distinct type
  aliases** (ch.02) — both can change which candidate wins.
- *Parser impact:* none beyond recording the directive; resolution is semantic.

---

## 6.4 Inlining

### 6.4.1 The `inline` directive

| | |
|---|---|
| **Introduced** | Delphi 2005 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Requests the compiler expand the routine body at the call site.

```pascal
function Min(A, B: Integer): Integer; inline;
```

**Semantics & parsing notes**

- `inline` is a **directive** here (note: it is *also* a reserved word in some
  lists — Delphi treats it as a directive in this position). It is a *hint*; the
  compiler may decline.

---

## 6.5 Calling conventions

### 6.5.1 Convention directives

| | |
|---|---|
| **Introduced** | Delphi 1+ (`register` default); `winapi` later alias |
| **Deprecated** | — |
| **Status** | ✅ Current |

Directives controlling argument passing/cleanup ABI.

**Grammar**

```ebnf
CallConv = "register" | "stdcall" | "cdecl" | "pascal" | "safecall" | "winapi" | "fastcall" ;
```

**Semantics & parsing notes**

- All are **directives** (B.4.2). `register` is the Delphi default; `winapi` maps to
  the platform's native API convention (`stdcall` on Win32, etc.).
- `safecall` additionally wraps the routine in HRESULT/exception marshalling (COM).
- Mostly codegen, but the parser must accept them in the directive list after a
  heading.

---

## 6.6 Procedural types

### 6.6.1 Procedure/function pointer types

| | |
|---|---|
| **Introduced** | Pascal/D1; `of object` D1; `reference to` Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Types whose values are routines: plain pointers, method pointers (`of object`),
and closures (`reference to`, ch.17).

**Grammar**

```ebnf
ProceduralType = ( "procedure" | "function" )
                 [ "(" [ FormalParams ] ")" ] [ ":" ResultType ]
                 [ "of" "object" | "reference" "to" ] [ CallConv ] ;
```

**Example**

```pascal
type
  TNotify   = procedure(Sender: TObject) of object;   // method pointer
  TComparer = reference to function(A, B: Integer): Integer;  // closure
  TThunk    = procedure;                                // plain pointer
```

**Semantics & parsing notes**

- ⚠️ Three distinct runtime shapes: a **plain** pointer (code address), a **method
  pointer** (`of object` = code + `Self`, 8/16 bytes), and a **reference** (an
  interface-backed closure that can capture locals). Keep the kind on the type
  node — assignment compatibility differs across them.
- `of object` and `reference to` use the directives/reserved words `object` and
  `reference`; `reference` is a directive (B.4.2).

---

## 6.7 External declarations

### 6.7.1 `external` (and `name`, `index`, `delayed`, `varargs`)

| | |
|---|---|
| **Introduced** | `external` D1; `delayed` Delphi XE2; `varargs` D-early |
| **Deprecated** | — |
| **Status** | ✅ Current |

Binds a routine to an external library function instead of a Pascal body.

**Grammar**

```ebnf
ExternalDecl = "external" [ ConstExpr ]            (* library name *)
               [ "name" ConstExpr | "index" ConstExpr ]
               [ "dependency" ConstExpr { "," ConstExpr } ]   (* linked-lib hints *)
               [ "delayed" ] ";" ;
```

**Example**

```pascal
function MessageBox(hWnd: HWND; lpText, lpCaption: PChar; uType: UINT): Integer;
  stdcall; external 'user32.dll' name 'MessageBoxW';
```

**Semantics & parsing notes**

- An `external` routine has **no `Block`** — the body is the OS/library symbol.
- `name`/`index` (directives) select the imported symbol; `delayed` defers load
  until first call. `varargs` (with `cdecl`) allows C-style variadic calls.
- `dependency` (directive) lists additional libraries the import needs at link
  time — used mainly by the mobile/posix toolchains
  (`external libc name 'dlopen' dependency 'dl'`).
- *AST:* `RoutineDecl { …, external: { lib, symbol?, index?, delayed } }`.

---

## 6.8 The `noreturn` directive

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Marks a routine that never returns to its caller (always raises/halts), enabling
better flow analysis.

**Example**

```pascal
procedure Fatal(const Msg: string); noreturn;
```

**Semantics & parsing notes**

- `noreturn` is a **directive** (B.4.2, new in 13.0) in the routine's directive
  list. Tells the flow analyser that code after a call to it is unreachable, and
  suppresses "function might not return a value" warnings on paths that end in it.
- *AST:* flag `noReturn: true` on the routine node.

---

## 6.9 Nested routines

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

A routine may be declared inside another routine's `Block`, with access to the
enclosing routine's locals.

```pascal
procedure Outer;
  procedure Inner;          // nested
  begin ... end;
begin Inner; end;
```

**Semantics & parsing notes**

- Nested routines create a lexical scope chain — the resolver must allow `Inner`
  to see `Outer`'s locals (captured via a static link / frame pointer).
- *AST:* nested `RoutineDecl` within the parent's declaration sections.

---

## 6.10 Inline assembly (`asm … end`)

| | |
|---|---|
| **Introduced** | Turbo Pascal / Delphi 1; x64 support XE2; not supported on ARM targets 🧪 |
| **Deprecated** | — |
| **Status** | ✅ Current (Intel targets only) |

Embeds assembler instructions, either as a **statement** inside a normal body or
as the **entire routine body** (replacing `begin…end`).

**Grammar**

```ebnf
AsmStmt  = AsmBlock ;                       (* as a statement inside a Block *)
AsmBlock = "asm" { ?assembler token? } "end" ;
(* a routine body may be an AsmBlock instead of a begin..end Block; the
   'assembler' directive on the heading is legacy/optional *)
```

**Example**

```pascal
function GetSP: NativeUInt;
asm                       // asm as the whole routine body
  MOV RAX, RSP
end;

procedure P;
begin
  asm                     // asm as a statement
    NOP
  end;
end;
```

**Semantics & parsing notes**

- ⚠️ *Lexer mode switch:* the content between `asm` and its matching `end` is
  **not Object Pascal** — it is BASM (built-in assembler) with its own lexical
  rules: `@@Label:` local labels, register names, `$`-hex and Intel-style `0FFh`
  hex, `//` and `{ }` comments still valid. A parser should treat the body as an
  opaque token stream: scan for the closing `end` at nesting level 0, honouring
  string/comment boundaries. Do **not** try to parse it as Pascal.
- Pascal identifiers (locals, params, globals) may be referenced from BASM —
  symbol resolution *into* the asm body is a semantic/codegen concern; the parser
  only needs to capture the raw text/tokens.
- Platform-gated: x86/x64 only; ARM/ARM64 compilers reject `asm` (RTL guards these
  blocks with `{$IFDEF}`s — 53 uses in RTL 13 sources). The old `assembler`
  directive on headings is accepted and ignored (legacy).
- *AST:* `AsmBlock { rawTokens / sourceRange }` as a statement node or routine body.
