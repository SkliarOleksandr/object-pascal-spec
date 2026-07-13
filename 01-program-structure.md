# 01 — Program Structure & Compilation

The outermost grammar: how source files are shaped (programs, units, packages),
how units import each other and scope names, and how compiler directives —
especially conditional compilation and includes — alter the token stream the
parser sees.

> ⚠️ This chapter governs the **directive pre-pass**. A correct parser cannot
> treat directives as comments: `{$IFDEF}`/`{$INCLUDE}` change which tokens
> exist. See [§B.2.2](B-lexical-grammar.md#b22-compiler-directives).

## Chapter grammar umbrella

```ebnf
SourceFile = Program | Unit | Package | Library ;

Program    = "program" QualifiedIdent [ "(" IdentList ")" ] ";"
             [ UsesClause ] Block "." ;
Unit       = "unit" QualifiedIdent [ HintDirectives ] ";"
             InterfaceSection ImplementationSection
             [ InitSection ] "." ;
Library    = "library" QualifiedIdent ";" [ UsesClause ] Block "." ;
Package    = "package" QualifiedIdent ";" [ RequiresClause ] [ ContainsClause ] "end" "." ;
```

---

## 1.1 Program & unit structure

### 1.1.1 The program file

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

The main entry-point file (`.dpr`). Its `Block` body is the program's startup
code.

**Grammar**

```ebnf
Program = "program" QualifiedIdent [ "(" IdentList ")" ] ";"
          [ UsesClause ] Block "." ;
```

**Example**

```pascal
program Hello;
uses System.SysUtils;
begin
  Writeln('Hi');
end.
```

**Semantics & parsing notes**

- The legacy `(Input, Output)` program parameter list is accepted and ignored.
- ⚠️ The file ends with `end.` — the trailing **`.`** terminates the compilation
  unit; tokens after it are ignored.
- *AST:* `Program { name, uses, block }`.

### 1.1.2 The unit file

| | |
|---|---|
| **Introduced** | Turbo Pascal / Delphi 1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A reusable compilation unit (`.pas`) with a public `interface` and a private
`implementation`, plus optional initialization/finalization.

**Grammar**

```ebnf
InterfaceSection      = "interface" [ UsesClause ] { InterfaceDecl } ;
ImplementationSection = "implementation" [ UsesClause ] { ImplDecl } ;
InitSection           = "initialization" StatementList
                        [ "finalization" StatementList ]
                      | "begin" StatementList ;   (* legacy: begin..end as init *)
```

**Example**

```pascal
unit Math.Utils;
interface
  function Square(X: Integer): Integer;
implementation
  function Square(X: Integer): Integer;
  begin Result := X * X; end;
initialization
  RegisterStuff;
finalization
  CleanupStuff;
end.
```

**Semantics & parsing notes**

- ⚠️ *Two-scope visibility:* names declared in `interface` are exported (visible to
  importers); names in `implementation` are unit-private. The name-resolution pass
  must model these as two nested scopes.
- *Forward completeness:* routines declared in `interface` must be defined in
  `implementation`; the interface declaration acts as an implicit forward.
- `initialization`/`finalization` are reserved words; the legacy bare `begin … end`
  before `end.` is an older equivalent of `initialization`.
- *AST:* `Unit { name, interfaceDecls, implDecls, init?, final? }`.

### 1.1.3 Library & package files

| | |
|---|---|
| **Introduced** | `library` D1; `package` D3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`library` builds a shared library (DLL/.so/.dylib); `package` is a Delphi-specific
bundle of units (BPL).

**Grammar**

```ebnf
ExportsClause = "exports" ExportEntry { "," ExportEntry } ";" ;
ExportEntry   = Ident [ "index" ConstExpr ] [ "name" ConstExpr ] [ "resident" ] ;
RequiresClause = "requires" IdentList ";" ;
ContainsClause = "contains" ContainsEntry { "," ContainsEntry } ";" ;
ContainsEntry  = QualifiedIdent [ "in" StringLiteral ] ;   (* uses-like paths *)
```

**Semantics & parsing notes**

- `exports` (reserved word) lists routines exposed from a library, with directive
  modifiers `index`/`name` (directives, B.4.2).
- ⚠️ *`exports` is also legal inside a **unit*** (interface or implementation
  section) — the entries take effect when the unit is linked into a library. The
  RTL uses this (`System.Internal.MachExceptions.pas`). The parser must accept
  `ExportsClause` as a declaration in units, not only in library files.
- ⚠️ *A library may omit the main `begin`-block entirely:*
  `library X; ... exports Foo name 'Test'; end.` (DUnit's testXpgenLib.dpr).
- `requires`/`contains` are package-only clauses: `requires` names other
  PACKAGES (semantic analysis must not resolve them as units); `contains` names
  units and — like a program's `uses` — may bind each to a source path with
  `in 'path'` (dcc-verified: the RTL's own BuildWinRTL.dpk,
  `System.SysUtils in 'sys\System.SysUtils.pas'`). A `.dpk`'s contains-list is
  therefore a full uses-like dependency graph, analyzable like a program's.
- *AST:* `Library` / `Package`.

---

## 1.2 Units, uses & namespaces

### 1.2.1 The `uses` clause

| | |
|---|---|
| **Introduced** | Turbo Pascal / D1; `in 'file'` form D-early |
| **Deprecated** | — |
| **Status** | ✅ Current |

Imports other units; may appear once in the `interface` and once in the
`implementation` section (and in a program/library).

**Grammar**

```ebnf
UsesClause = "uses" UsesEntry { "," UsesEntry } ";" ;
UsesEntry  = QualifiedIdent [ "in" StringLiteral ] ;   (* "in" path used in .dpr *)
```

**Example**

```pascal
uses
  System.SysUtils, System.Classes,
  MyUnit in 'src\MyUnit.pas';
```

**Semantics & parsing notes**

- ⚠️ *Resolution-order significance:* when the same identifier is exported by
  multiple used units, the **last unit in the `uses` list wins**. The
  name-resolution pass must preserve `uses` order.
- *Interface vs. implementation uses:* interface-section imports are transitively
  visible considerations for circular-reference rules; `implementation` uses break
  interface circular dependencies.
- The `in 'file'` form binds a unit name to a source path (program/`.dpr` only).
- ⚠️ *A declaration may hide a used unit's name:* declaring a type/var/const with
  the same name as a used unit's (leaf) name is legal and HIDES the bare unit
  name from that point on — the unit stays reachable via its fully-qualified
  name. dcc-verified in the RTL: Winapi.WinSock2 declares
  `QOS = _QualityOfService` while using Winapi.Qos (leaf name `Qos`). A resolver
  that treats this as a redeclaration produces false E2004.
- *AST:* ordered `uses[]` per section, each `{ qualifiedName, path? }`.

### 1.2.2 Dotted (namespaced) unit names

| | |
|---|---|
| **Introduced** | Delphi XE2 (namespaces); concept earlier on .NET |
| **Deprecated** | — |
| **Status** | ✅ Current |

Unit names may be dotted (`System.SysUtils`); the leading segments form a
**namespace** prefix that can be defaulted via project "unit scope names".

**Semantics & parsing notes**

- ⚠️ A dotted name is **one unit identifier**, not member access — `System.Classes`
  is the unit `System.Classes`, parsed as a `QualifiedIdent` in `uses`/qualified
  references. Disambiguate `unit.scope.symbol` references by trying the longest
  matching unit name first.
- *Unit scope names* (a project setting) let `Classes` resolve to `System.Classes`
  implicitly. This is configuration the resolver consults; the parser just records
  the dotted name.

### 1.2.3 Qualified name resolution

| | |
|---|---|
| **Introduced** | Pascal; expanded with namespaces XE2 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A reference may be qualified `Unit.Symbol` (or `Namespace.Unit.Symbol`) to
disambiguate.

**Semantics & parsing notes**

- ⚠️ The dotted-name ambiguity is real: `A.B.C` could be unit `A.B` member `C`,
  unit `A` member `B` member `C`, or namespaced unit `A.B.C`. Resolution tries
  unit-name matches greedily, then falls back to member access. Keep the raw
  dotted token list on the AST so the resolver can re-segment.

---

## 1.3 Compiler directives

### 1.3.1 Switch & parameter directives

| | |
|---|---|
| **Introduced** | Turbo Pascal / D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Short toggles (`{$X+}`/`{$X-}` or long `{$OPTIMIZATION ON}`) and parameter
directives that set compiler options.

**Grammar**

```ebnf
SwitchDirective = "{$" SwitchLetter ( "+" | "-" ) "}"
                | "{$" LongOption ( "ON" | "OFF" | Value ) "}" ;
```

**Semantics & parsing notes**

- Many switches change **semantics** the type-checker depends on:
  `{$BOOLEVAL}` (short-circuit, ch.04), `{$WRITEABLECONST}`/`{$J}` (ch.03),
  `{$SCOPEDENUMS}` (ch.02), `{$POINTERMATH}`/`{$TYPEDADDRESS}` (ch.04),
  `{$M}`/RTTI, `{$ALIGN}`/packing (ch.09), and `{$MINENUMSIZE n}`/`{$Z}` —
  ⚠️ the latter changes the **storage size of enumerated types** (interop headers
  set `{$MINENUMSIZE 4}`). Track current switch state as a stack keyed by source
  position.

**Directive families a parser must at least recognise & skip** (heavily used in
the RTL):

| Family | Directives | Effect |
|---|---|---|
| Switches | `{$X+}`/`{$X-}`, long `{$OPTIMIZATION ON}` | option state (stacked; see `{$PUSHOPT}`) |
| Conditionals | `{$IFDEF}` `{$IF}` `{$IFOPT}` `{$ELSE(IF)}` `{$ENDIF}` `{$DEFINE}` `{$UNDEF}` | include/exclude token ranges (1.3.2) |
| Inclusion & linking | `{$I file}` / `{$INCLUDE}`, `{$L file.obj}` / `{$LINK}`, `{$R res}` / `{$RESOURCE}` | splice source / link object files / bind resources |
| Warnings & messages | `{$WARN ident ON\|OFF\|ERROR}`, `{$WARNINGS}`, `{$HINTS}`, `{$MESSAGE HINT\|WARN\|ERROR 'txt'}` | diagnostics control |
| Packaging | `{$WEAKPACKAGEUNIT}`, `{$DENYPACKAGEUNIT}`, `{$IMPLICITBUILD}` | package/BPL behaviour |
| C++Builder interop | `{$HPPEMIT}`, `{$EXTERNALSYM}`, `{$NODEFINE}`, `{$OBJTYPENAME}` | .hpp header generation only (System.pas alone has ~340) — safe to skip |
| IDE-only | `{$REGION 'x'}` / `{$ENDREGION}` | code folding; no compile effect |
| RTTI & layout | `{$RTTI}` (ch.19), `{$M}`, `{$METHODINFO}`, `{$ALIGN}`, `{$MINENUMSIZE}` | metadata & memory layout |
| Output/binary | `{$APPTYPE CONSOLE\|GUI}`, `{$SETPEFLAGS}`, `{$IMAGEBASE}`, `{$SONAME}`/`{$SOPREFIX}`/`{$SOVERSION}` (Linux), `{$E ext}` | executable/library output shaping |
| Codegen misc | `{$EXCESSPRECISION}` (x64 floats), `{$VARPROPSETTER}` (COM Variant setters), `{$LEGACYIFEND}` (see 1.3.2 ⚠️) | target-specific codegen / grammar switches |

### 1.3.2 Conditional compilation

| | |
|---|---|
| **Introduced** | Turbo Pascal; `{$IF}`/`{$ELSEIF}` expression form Delphi 6/2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Includes or excludes regions of source based on defined symbols or constant
expressions.

**Grammar**

```ebnf
CondCompile = "{$IFDEF" Ident "}"  | "{$IFNDEF" Ident "}"
            | "{$IF" ConstExpr "}" | "{$ELSEIF" ConstExpr "}"
            | "{$IFOPT" SwitchLetter ( "+" | "-" ) "}"      (* test a switch state *)
            | "{$ELSE}" | "{$ENDIF}" | "{$IFEND}"
            | "{$DEFINE" Ident "}" | "{$UNDEF" Ident "}" ;
```

**Example**

```pascal
{$IFDEF MSWINDOWS}
  UseWinApi;
{$ELSEIF Defined(LINUX)}
  UsePosix;
{$ENDIF}
```

**Semantics & parsing notes**

- ⚠️ *This is the core of the directive pre-pass.* Excluded regions are **not
  parsed** (their tokens are skipped entirely, including unbalanced constructs). A
  parser that tokenises first and parses later must apply conditional exclusion
  during/before tokenisation.
- `{$IF}` evaluates a **compile-time constant expression** that may use
  `Defined(X)`, `Declared(X)`, and predefined consts like `CompilerVersion`.
- `{$IFOPT X+}`/`{$IFOPT X-}` tests the current **state of a switch directive**
  (e.g. `{$IFOPT R-}` = "if range checking is off") — ties the conditional
  pre-pass to the switch-state stack of 1.3.1. Used in the RTL (`System.pas`,
  `System.Variants`).
- ⚠️ *`{$IF}` terminator rules & `{$LEGACYIFEND}`:* historically `{$IF}` had to be
  closed with `{$IFEND}` (not `{$ENDIF}`). Modern compilers accept `{$ENDIF}` for
  `{$IF}` **unless** `{$LEGACYIFEND ON}` is in effect, which re-imposes the strict
  pairing. The conditional pre-pass must implement both modes (D13 sources set it:
  `Xml.*`, DUnit).
- Symbols come from `{$DEFINE}`, project options, and built-ins (`MSWINDOWS`,
  `CPUX64`, `CONSOLE`, etc.).
- ⚠️ *`{$DEFINE}`/`{$UNDEF}` (and switch changes) are LOCAL to the unit being
  compiled* — a batch preprocessor must reset to the project define-set per
  file. Leaking one unit's defines into the next mis-branches conditional
  chains (empirically: System.ObjAuto.pas selects the wrong ASM variant).
- ⚠️ *Undocumented dcc tolerances* (all shipped in `System.ObjAuto.pas`,
  verified against RTL 13.0 — a conforming parser must accept them):
  1. **Multiple `{$ELSE}` in one chain** — `{$IF}…{$ELSE}…{$ELSE OTHERCPU}…{$ENDIF}`
     compiles; each `$ELSE` activates iff no earlier branch was taken.
  2. **Trailing junk after a `{$IF}` expression** is ignored —
     `{$IF SizeOf(Extended) >= 10)}` (stray `)`) compiles.
  3. Trailing text after `{$ELSE}`/`{$ENDIF}`/`{$IFEND}` is an ignored
     comment (`{$ENDIF OTHERCPU}`, `{$ELSE !CPUX86}`) — widely used.
- ⚠️ *`{$IF}` sees unit constants:* the real compiler evaluates `$IF` with
  visibility of constants/enums (`{$IF Ord(soBeginning) = STREAM_SEEK_SET}`,
  `Xml.adomxmldom.pas`) — full evaluation is only possible **after** semantic
  analysis. A standalone preprocessor must define a fallback policy for such
  expressions (e.g. evaluate to False and flag).
- *AST:* conditional structure is usually resolved away before the syntax tree;
  optionally retained as trivia for tooling.

### 1.3.3 Include files

| | |
|---|---|
| **Introduced** | Turbo Pascal / D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`{$INCLUDE file}` / `{$I file}` splices another file's source at that point.

**Grammar**

```ebnf
IncludeDirective = "{$I" FileRef "}" | "{$INCLUDE" FileRef "}" ;
```

**Semantics & parsing notes**

- ⚠️ Textual inclusion happens **before parsing** — the included tokens become part
  of the current unit's stream. Track an include stack for accurate source
  positions/diagnostics. `{$I %ENV%}` and `{$I %DATE%}` forms inject special
  values.
- Beware include cycles; cap recursion.

### 1.3.4 `{$PUSHOPT}` / `{$POPOPT}`

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Save and restore the current set of switch-directive options as a stack.

**Example**

```pascal
{$PUSHOPT}
{$OPTIMIZATION OFF}
// ... code compiled with optimization off ...
{$POPOPT}   // restore previous option state
```

**Semantics & parsing notes**

- Formalises the option **stack** the pre-pass already needs. `{$PUSHOPT}` snapshots
  all switch options; `{$POPOPT}` restores the last snapshot.
- Replaces ad-hoc per-switch save/restore (older `{$I+}{...}{$I-}` idioms).

### 1.3.5 Compiler-version symbols

| | |
|---|---|
| **Introduced** | `VERxxx` Turbo; `CompilerVersion`/`RTLVersion` consts Delphi 6+ |
| **Deprecated** | — |
| **Status** | ✅ Current |

Predefined symbols/constants used in `{$IF}`/`{$IFDEF}` to gate version-specific
code.

**Example**

```pascal
{$IF CompilerVersion >= 36}   // Delphi 12 Athens or later
  // use multiline string literals
{$IFEND}
```

**Semantics & parsing notes**

- `CompilerVersion` is a real constant (e.g. 36.0 = Athens, 37.0 = Florence);
  `VER360`-style symbols are auto-`{$DEFINE}`d. The pre-pass must seed these per
  target so `{$IF}` evaluates correctly.
- Cross-reference [Appendix A](A-version-history.md) for the version↔number map.
