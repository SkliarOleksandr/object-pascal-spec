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
             [ UsesClause ] ProgramBody "." ;
Unit       = "unit" QualifiedIdent [ HintDirectives ] ";"
             InterfaceSection ImplementationSection
             ( InitSection | "end" ) "." ;
Library    = "library" QualifiedIdent ";" [ UsesClause ] ProgramBody "." ;
ProgramBody = Block | { DeclSection } "end" ;   (* main "begin" optional, 1.1.1 *)
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
Program     = "program" QualifiedIdent [ "(" IdentList ")" ] ";"
              [ UsesClause ] ProgramBody "." ;
ProgramBody = Block | { DeclSection } "end" ;
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

- The legacy `(Input, Output)` program parameter list is accepted and ignored
  (dcc-verified, dcc64 37.0, probed 2026-10-02, x-f17 P01-P21): plain
  identifiers, at least one, commas between - any names, a repeat
  (`(A, A)`), the program's own name, an escaped keyword (`&begin`) compile;
  `()`, a number, a dotted name, a reserved word, `Input: Text` and a
  trailing comma are E2029. The names declare nothing and refer to nothing:
  `program P(Foo); begin Foo := 1; end.` is E2003, and
  `program P(Input); var Input: Integer;` compiles. A `library` takes none
  (`library L(A);` is E2029 `';' expected but '(' found`).
- ⚠️ The file ends with `end.` — the trailing **`.`** terminates the compilation
  unit; tokens after it are ignored. Not even lexed for errors: a change
  note after `end.` with an apostrophe (`don't`) is no E2052, only `W1011
  Text after final 'END.' - ignored by compiler` - in a unit as in a
  program (probed, dcc64 37.0, 2026-10-15, PasTree local/probe-b63 misc).
- ⚠️ The program's name is declared in its global scope: a global of the same
  name is `E2004 Identifier redeclared` (see §1.1.2 for the whole rule).
- ⚠️ *The main `begin ... end` is optional in a program too, not only in a
  library (1.1.3):* `program P; var X: Integer; end.` compiles to an
  executable that does nothing - the declarations are followed directly by
  `end.`. A program header takes no hint directive (unlike a unit's,
  1.1.2): `program P deprecated;` and `library L platform;` are `E2029 ';'
  expected but identifier 'deprecated' found`. dcc64 37.0, 2026-10-30 (spec
  audit g1, local/probe-park/spec-audit/g1 NoBegin.dpr, LibNoBegin.dpr,
  ProgHint.dpr, LibHint.dpr).
- *AST:* `Program { name, params, uses, block }`, `ProgramParams { names }`.

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
                        [ "finalization" StatementList ] "end"
                      | "begin" StatementList "end" ;   (* legacy: begin..end as init *)
(* a unit with neither ends "implementation ... end." - Unit, chapter umbrella *)
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
- ⚠️ *...but one scope for redeclaration.* An `implementation` declaration that
  takes an `interface` name is `E2004 Identifier redeclared` - a `var` beside an
  interface `var`, a `type` beside a `const`, a routine beside a `type`, a
  `label`, an enumeration value on either side (dcc64 37.0, probed 2026-09-27).
  Three kinds of pair compile: a routine's implementation or an overload the
  implementation adds (their mismatches are `E2037`/`E2267`, never `E2004`), a
  used unit's name on either side (`implementation uses B` beside an interface
  `var B`; an implementation `var B` hiding an interface `uses B`), and two
  types of different generic arity (`G<T>` beside `G`; two `G<T>` clash). A
  forward class declared in the interface cannot be completed in the
  implementation: `E2004` and `E2086`.
- *Forward completeness:* routines declared in `interface` must be defined in
  `implementation`; the interface declaration acts as an implicit forward.
- `initialization`/`finalization` are reserved words; the legacy bare `begin … end`
  before `end.` is an older equivalent of `initialization`. It takes no
  `finalization` part: `begin S; finalization T; end.` is `E2029 'END'
  expected but 'FINALIZATION' found`. Nor is there a `finalization` without
  `initialization`: `implementation finalization S; end.` is `E2029
  Declaration expected but 'FINALIZATION' found` (dcc64 37.0, 2026-10-30,
  spec audit g1, local/probe-park/spec-audit/g1 FinOnly.dpr, UFin.pas).
  The two spellings compile to the same
  code, but not to the same `.dcu`: under `$D+ $L+` the section's line
  records differ (on one line with its statements too; the word's case and
  column do not matter), under `-$D- -$L-` the files are identical - a
  tool that reprints a unit keeps the word as written (dcc64/dcc32 37.0,
  probed 2026-09-29).
- ⚠️ *The unit's declared name must match its file name* (dotted names too:
  `unit MyProject.Unit1;` must live in `MyProject.Unit1.pas`) — dcc-verified,
  dcc32 37.0: a file `Foo.pas` containing `unit Bar;` fails with
  `E1038 Unit identifier 'Bar' does not match file name`, and any program that
  `uses` it fails to build at all (`F1027`/`F2063 Could not compile used
  unit`). It does NOT apply to a program, a library or a package: `program
  Zed;` in `Q.dpr` builds `Q.exe`, `library Zed;` in `Q.dpr` builds `Q.dll`,
  and `package Zed;` in `Q.dpk` builds `Q.bpl` beside `Zed.dcp` (the `.dcp`
  takes the package's name, which a `requires` clause names) - dcc64 37.0,
  probed 2026-10-11. A rename of such a header obliges no file rename.
  A checker that resolves `uses` entries by file path (§1.2.2) can rely on
  this identity holding for every unit that actually compiles.
- ⚠️ *The module's own name is declared in its global scope - when that name
  is ONE identifier.* A unit-level declaration spelled like the unit's name is
  `E2004 Identifier redeclared`, in the `interface` and the `implementation`
  section alike and case-insensitively (`unit U; ... var u: Integer;` fails):
  `var`, `threadvar`, `const`, `resourcestring`, a non-generic `type`, a
  routine (both of its headers report), an enumeration value. A program's and
  a library's name work the same way (`program P; var P: Integer;` fails).
  What compiles is everything in a NESTED scope - a routine's local `var`,
  `const` or `type`, a parameter, a class or record field, method or property,
  a generic type parameter - and two module-level shapes: a GENERIC type
  (`type U<T> = class end;` in unit `U`: the declared name carries its arity),
  and anything in a unit with a DOTTED name, which is no single identifier
  (`unit NS.U;` compiles `var U`, `var NS` and `type U = class end`;
  `unit NS.Sub.U;` compiles `var Sub`). The name stays usable as a qualifier
  of the unit's own names (`U.P` in U's initialization). dcc-verified, dcc64
  37.0, 2026-09-27. Such a qualifier reaches the IMPLEMENTATION's declarations
  made so far as well as the interface's, and never a member of the enclosing
  type: `Xq.Own.QoHash(1, 2)` in a method of a class with its own `QoHash`
  calls an implementation-only routine, `Xq.Own.QoVer` an implementation
  constant past a same-named static method (Vcl.Graphics'
  `Vcl.Graphics.GetHashCode(ResData, ResDataSize)`, System.Character's
  `System.Character.UnicodeDataVersion` - dcc64 37.0, 2026-10-08). This is the other side of §1.2.1's hiding rule: a
  declaration may hide a USED unit's name, never its own module's. Where it
  bites in practice: the Form Designer names a form's or data module's global
  variable after its `Name`, so a data module named after its own unit
  (`SpikeData` in `SpikeData.pas`) gets `var SpikeData: TSpikeData;` - an
  E2004 nothing warns about until the build. A checker must report E2004 for
  such a declaration rather than let it hide the unit identifier.
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
ExportEntry   = QualifiedIdent [ "(" ParamList ")" ] [ "index" ConstExpr ] [ "name" ConstExpr ] [ "resident" ] ;
RequiresClause = "requires" QualifiedIdent { "," QualifiedIdent } ";" ;
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
- *Entry clauses (dcc64 37.0, probed 2026-10-01):* `index`, `name` and `resident` come in THAT order only, each at most once - `Q name 'a' index 3`, `Q resident name 'a'`, a repeated `name` / `index` / `resident` are all E2029 (", or ; expected"). `name` and `index` take any constant expression (`name 'a' + 'b'`, a named constant); an overload is picked by a parameter list - `P(A: Integer) name 'a'` (parameter NAMES are required: `P(Integer)` is E2067) - and the same routine may be exported twice under two names. An entry may be a bare `Q;`.
- ⚠️ *A library may omit the main `begin`-block entirely:*
  `library X; ... exports Foo name 'Test'; end.` (DUnit's testXpgenLib.dpr);
  so may a program (1.1.1).
- *Where `exports` may stand, and what it names* (dcc64 37.0, 2026-10-30,
  spec audit g1, local/probe-park/spec-audit/g1 ExpProg.dpr, LocalExp.dpr,
  ExpQual.dpr, ExpMeth.dpr): at the global level of a library, of a unit,
  and of a PROGRAM too (`program P; procedure Q; ... exports Q; begin end.`
  compiles); inside a routine it is `E2191 EXPORTS allowed only at global
  scope`. An entry is a routine's name, unit-qualified if wanted (`exports
  UExp.Q name 'q1';` compiles); a class's static class method is not
  exportable - `exports TK.S` is `E2276 Identifier 'TK' cannot be exported`.
- `requires`/`contains` are package-only clauses: `requires` names other
  PACKAGES (semantic analysis must not resolve them as units), dotted names
  included - `requires rtl, Nope.Sub.Pkg;` parses and fails only on the
  lookup (`E2202 Required package 'Nope.Sub.Pkg' not found`, reported as
  Fatal), while a string there is `E2029 Identifier expected but string
  constant found` (dcc64 37.0, 2026-10-30, spec audit g1,
  local/probe-park/spec-audit/g1 PkgDot.dpk, PkgDot2.dpk); `contains` names
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
- *Interface vs. implementation uses - circularity:* a cycle made only of
  INTERFACE `uses` is fatal, `F2047 Circular unit reference to 'UC1'`
  (UC1's interface uses UC2, UC2's interface uses UC1); the same cycle
  closed through an `implementation` uses compiles (UC3's interface uses
  UC4, UC4's implementation uses UC3). dcc64 37.0, 2026-10-30 (spec audit
  g1, local/probe-park/spec-audit/g1 Circ.dpr, Circ2.dpr).
- *One clause per section, right after its header:* a second `uses`, and a
  `uses` after a declaration (`program P; const A = 1; uses X;`), is `E2029
  Declaration expected but 'USES' found` (dcc64 37.0, 2026-10-30, spec audit
  g1, local/probe-park/spec-audit/g1 TwoUses.dpr, UsesLate.dpr).
- *Only ANOTHER unit can be used.* A unit naming itself is `F2092 Program or
  unit 'a1' recursively uses itself`; a program file found under the name is
  compiled as a unit and fails (`E2029 'UNIT' expected but 'PROGRAM' found`,
  then `F2063`) - and from a unit the program itself uses, dcc64 37.0 stops
  at `F2084 Internal Error`; `System` and `SysInit` are `E2004` (§1.2.4's
  implicit units). A uses-clause completion offers none of them (probed
  2026-10-13, dcc64 37.0, PasTree audit B49).
- ⚠️ *The interface section sees its own `uses` only.* A name in the
  interface resolves through the interface clause even when an
  implementation-used unit, listed later, declares it too: with
  `interface uses UA; var G: TFoo; H: TGen<Integer>; type TD =
  class(TFoo)` and `implementation uses UB`, both units declaring `TFoo`
  and `TGen<T>`, `G`, `H` and `TD`'s ancestor are UA's. In the
  implementation both clauses are visible, the implementation's first
  (§3.3.2 step 6). The real shape: `interface uses Vcl.Graphics` (the class
  `TBitmap`) beside `implementation uses Winapi.Windows` (the record
  `TBitmap = tagBITMAP`).
- A unit named twice - in both clauses, or twice in one - is `E2004
  Identifier redeclared` at the second entry. dcc64 37.0, probed 2026-10-12
  (PasTree audit B25, `probe-b25` P1, UD1, UD2).
- The `in 'file'` form binds a unit name to a source path - in a program's
  or a library's `uses` (`.dpr`) and in a package's `contains` (1.1.3); in a
  unit's `uses` it is `E2029 ',' or ';' expected but 'IN' found` (dcc64
  37.0, 2026-10-30, spec audit g1, local/probe-park/spec-audit/g1
  UnitIn.dpr, UIn.pas, LibIn.dpr).
- ⚠️ *A declaration may be spelled like a used unit's name:* it is legal,
  not E2004. An undotted entry is an identifier (§1.2.3), and a declaration
  of its name HIDES the bare unit name from that point on (dcc64 37.0:
  `uses B; var B: Integer;` compiles). A dotted entry's last segment is no
  identifier at all, so a declaration spelled like it hides nothing: `uses
  System.SysUtils; type SysUtils = Integer;` compiles and
  `System.SysUtils.TBytes` still works, while `SysUtils.TBytes` without the
  declaration is `E2003 Undeclared identifier: 'SysUtils'` even under
  `-NSSystem` (probed 2026-10-12, `probe-b25` P10, P11). The RTL relies on
  it: Winapi.WinSock2 declares `QOS = _QualityOfService` while using
  Winapi.Qos. A resolver that treats either as a redeclaration produces
  false E2004. The module's OWN name is the exception - redeclaring it is E2004
  (§1.1.2).
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
- *Unit aliases* (dcc `-A` / project setting `DCC_UnitAlias`, e.g.
  `WinTypes=Winapi.Windows`) rewrite a `uses` name to a different real unit
  BEFORE any file/namespace resolution runs — a whole-name match, applied at
  most once (aliases do not chain). Distinct from unit scope names: an alias
  changes WHICH unit a name means; a unit scope name only supplies a missing
  namespace prefix for a name that would otherwise be ambiguous/unqualified.
- ⚠️ A unit scope name is tried against the name **as spelled, dotted or not**
  — it is not restricted to unqualified names. `uses Generics.Collections` with
  `-NSSystem` resolves to `System.Generics.Collections`; dcc-verified, and there
  is no `Generics.Collections.pas` anywhere on disk, so no other rule can
  explain it.
- Prefixes are tried only **after** the as-spelled lookup fails, so a dotted
  name that does name a real unit (`Vcl.Forms`) is never captured by a prefix.

**Resolution order for a `uses` name**

1. Explicit `in '<path>'`, if present.
2. Unit alias (`-A`), whole-name, once.
3. As spelled: `<dotted-name>.pas`.
4. Each unit scope name in order: `<scope>.<name-as-spelled>.pas`.
5. (Tolerated by some toolchains, not by dcc) the trailing segment alone —
   pre-namespace file layouts where `System.SysUtils` lived in `SysUtils.pas`.

⚠️ **The program's `in` path locates the unit for every importer, through
every spelling that reaches it.** A unit the program file names with
`X in 'path'` (or the project's file list names) IS that file for the whole
project, and each candidate spelling of steps 2-4 is checked against those
units before the file lookup for that candidate: with
`Acme.Forms in 'patched\Acme.Forms.pas'` in the program and another
`Acme.Forms.pas` on the search path, a unit writing `uses Forms` under
`-NSAcme` compiles against the patched copy, `uses Foo.Bar` under `-NSAcme`
reaches a pinned `Acme.Foo.Bar` with no `Foo.Bar` anywhere, and `uses Foo`
under `-AFoo=Acme.Forms` reaches the pinned copy too. The order still holds:
an EARLIER candidate with a file of its own wins over a later one that is
pinned - with `Forms.pas` on the search path, `uses Forms` is that unit and
the pinned `Acme.Forms` is not reached (dcc64 37.0, probed 2026-10-10).

An unresolvable name is fatal: `F1027 Unit not found`. A project that builds
therefore has zero of them, which makes the count a usable health metric for
any tool that reads a project.

#### Where the default lists come from

⚠️ **The compiler has no built-in unit scope names or aliases.** They come from
the project template, not from dcc — verified: `dcc32 --no-config` on a program
whose only import is `System` fails with `F1027 Unit not found: 'System'`. A
tool that analyzes a bare `.dpr`/`.dpk`, or a directory of sources, must supply
the defaults itself or every legacy unqualified import will appear undefined.

The IDE builds both lists in two groups — a platform-neutral base plus a
Windows-conditioned group that comes first (unit scope names as written into a
new project's `.dproj`; aliases in `CodeGear.Common.Targets`):

| | Windows-only group | Base group (all platforms) |
|---|---|---|
| **Unit scope names** | `Winapi`, `System.Win`, `Data.Win`, `Datasnap.Win`, `Web.Win`, `Soap.Win`, `Xml.Win` | `Vcl`, `Vcl.Imaging`, `Vcl.Touch`, `Vcl.Samples`, `Vcl.Shell`, `System`, `Xml`, `Data`, `Datasnap`, `Web`, `Soap` |
| **Unit aliases** | `WinTypes`/`WinProcs` → `Winapi.Windows`; `DbiTypes`/`DbiProcs`/`DbiErrs` → `BDE` | `Generics.Collections` → `System.Generics.Collections`; `Generics.Defaults` → `System.Generics.Defaults` |

The two settings differ in how a project's own entry combines with the default:

- `DCC_Namespace` **replaces** — the project states the whole list (its own
  `.dproj` text appends `$(DCC_Namespace)`, so what the project writes is what
  the compiler gets).
- `DCC_UnitAlias` is **prepended** to the defaults, not substituted. The default
  aliases therefore apply even to a project that declares its own, and a tool
  must add the defaults for every project rather than only when the setting is
  absent.

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
- ⚠️ *A unit qualifies by the name WRITTEN in the `uses` entry - whole, never
  its last segment.* An undotted entry is an identifier in the outermost scope:
  with `uses Log` (resolved by `-NS Ns` to `Ns.Log`), `Log.Tail` qualifies and
  `Log('a')` is E2029 - the unit name hides the unit's own routine `Log`. A
  dotted entry declares no single identifier at all: with `uses Ns.Log`,
  `Log('a')` calls the routine and `Log.Tail` is E2035 (the routine, called
  short of its argument); with `uses Ns.Other`, `Other.F` and a bare `Other` are
  E2003; with `uses System.SysUtils`, `SysUtils.IntToStr` is E2003; in unit
  `Ns.Own`, `Own.G` is E2003 while `Ns.Own.G` compiles. A namespace-resolved
  entry does not qualify by its resolved name either: `uses B.C` resolved by
  `-NS A` to `A.B.C` takes `B.C.T`, and `A.B.C.T` is E2003 - except under
  `System`, which is a unit itself: with `uses SysUtils`, `System.SysUtils.
  IntToStr` compiles. dcc64 37.0, probed 2026-10-01. A resolver that binds a
  dotted entry under its last segment captures every bare call of a routine
  spelled like it - a logging unit `X.Log` exporting `Log` left its 40 calls
  with no references.
- ⚠️ *A unit qualifier's FIRST segment is looked up like any other name, so
  every name that outranks a used unit hides it there* - not only a local
  or a declaration of the unit itself: in a method, a member of the class or
  of an ancestor declared in ANOTHER unit (`TDer = class(TBase)`, TBase with
  a method `QLib2`: `QLib2.GV` in TDer's method is `E2018 Record, object or
  class type required`, the same text compiles in a plain routine of the
  unit); in a `with` body, a target's member (5.7: `with B do QLib2.GV` is
  E2018); and the head of a DOTTED entry too - with `uses Winapi.Windows`, a
  field named `Winapi` makes `Winapi.Windows.GetCurrentProcess` in the
  class's method E2018. dcc64 37.0, probed 2026-10-03. The real shape: a
  logging family class with a method named like the logging unit, read
  through `with Log.Family do`. A tool writing qualified names (a
  refactoring, a "qualify" option) must check the head against the members
  in scope, inherited and with-opened ones included.
- ⚠️ *What a unit qualifier reaches* (dcc-verified, dcc64 37.0, probed
  2026-10-02, x-f34 Q01-Q32): the unit's interface, and a missing member is
  `E2003 Undeclared identifier` on the MEMBER - `System.Missing`,
  `ThisUnit.Missing`, `System.SysUtils.Missing`, a type `System.TMissing`, an
  ancestor `class(System.TMissing)`, a call `System.Missing(1)`. Beyond the
  interface: every intrinsic qualifies by `System` (`System.Length(S)`,
  `System.SizeOf(I)`, `System.Write(1)`, `System.Exit`, `System.Halt`,
  `System.Integer(1)`, `System.Pointer(nil)` all compile), the keyword types do
  (`System.string`, `System.file`), and a unit qualifying itself reaches its
  implementation too (`ThisUnit.ImplVar` from a routine below the variable
  compiles).

### 1.2.4 Implicit units (`System`, `SysInit`)

| | |
|---|---|
| **Introduced** | Turbo Pascal (`System`); `SysInit` early 32-bit Delphi |
| **Deprecated** | — |
| **Status** | ✅ Current |

Every unit's interface scope is seeded with **two** units it never names in a
`uses` clause: `System` and `SysInit`.

**Semantics & parsing notes**

- `System` is the well-known case (every RTL/VCL identifier not otherwise
  qualified — `TObject`, `Integer`, `sLineBreak`, ... — resolves through it).
  Less obvious: `SysInit` gets the exact same treatment, silently, for a
  smaller set of linker/startup globals — `HInstance`, `ModuleIsLib`,
  `ModuleIsPackage`, `DllProc`, and a few more (`rtl/sys/SysInit.pas`). RTL
  code (`System.Classes.pas`'s `InternalReadComponentRes`, among others)
  reads `HInstance` bare with no `uses SysInit` anywhere in the file.
- dcc-verified: a unit with an **empty** `uses` clause (or none at all) still
  compiles a function body that reads `HInstance` or `ModuleIsLib`. Naming
  either unit explicitly (`uses System;` / `uses SysInit;`) is itself a
  compile error — `E2004 Identifier redeclared` — the exact same failure
  mode for both, confirming they're implemented the same way.
- Neither unit implicitly uses itself, and `SysInit` does not implicitly gain
  `System` some OTHER way than every other unit does — it has no `uses`
  clause of its own and relies on the same blanket `System` seeding.
- A resolver: after normal local/`uses` lookup misses, fall back to `System`
  first, then `SysInit`, before reporting an undeclared identifier — mirror
  this order for BOTH plain references and inherited-member lookups inside
  method bodies (both are ordinary last-resort scopes, not just an
  expression-level special case).
- Both names also QUALIFY with no `uses` entry: `SysInit.HInstance` is
  written exactly like `System.TObject` - Vcl.Dialogs and FMX.Dialogs.Win
  pass `SysInit.HInstance` where a with target's field `hInstance` hides the
  bare name, and Vcl.Controls' `TDragObject.Instance` returns it (both ship
  and compile with dcc64 37.0). The qualifier is no symbol of the unit: a
  tool that lists "names bound to nothing" must take `SysInit` for a unit
  name, like `System` and the unit's own name (PasTree's resolver rung,
  2026-10-08).
- ⚠️ *`System.` qualifies COMPILER INTRINSICS too, though nothing declares
  them* - the rule and its probes are 1.2.3's ("What a unit qualifier
  reaches"; `System.Delete(S, 1, 1)` and `System.Inc(N)` compile too, dcc32
  37.0). It is routinely used to reach the intrinsic past a member of the same name
  (`System.Delete` inside a class that has its own `Delete`, as
  `System.WideStrings` does). So the `System` qualifier's namespace is the
  unit's interface PLUS the compiler-provided names; a resolver or a
  completion list for `System.` must include the intrinsics, not just what
  `System.pas` textually declares.
- *AST:* no node — this is pure resolver seeding, invisible in the parse
  tree; a `uses[]` list never gains a `System`/`SysInit` entry.

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
- ⚠️ *Every long form moves the letter `{$IFOPT}` reads*, the ones without a
  letter of their own in the docs' table too: `{$TYPEINFO ON}` is `M+`,
  `{$DEFINITIONINFO ON}` is `{$YD}` and reads as `Y+` (its `OFF` as `Y-`),
  `{$STACKCHECKS ON}` is `S+`. The forms with a digit or a letter: `{$D0}`
  is `D-`, `{$D1}` and `{$D2}` are `D+`, `{$YD}` is `Y+`; `{$A4}` is `A+`,
  `{$A1}` `A-`, `{$Z2}` `Z+`. A group goes on after `{$D1,R+}` (R+), but
  not after `{$YD,R+}` - the R+ is dropped. `{$RUNONLY ON}` moves no letter
  (probed, dcc64 37.0, 2026-10-15, PasTree local/probe-b62 T, T2).
- ⚠️ *Option state is linear in preprocessing order and unit-local.* A
  switch set in an include holds after the include ends (`{$I x.inc}` with
  `{$R+}` inside, then `{$IFOPT R+}` in the includer: taken); a `{$PUSHOPT}`
  in an include pairs with the includer's `{$POPOPT}` (1.3.4); a switch
  written inside a routine body holds past its `end` (`{$R+}` in `Foo`'s
  body, `{$IFOPT R+}` after it: taken) - no declaration or block scopes it;
  a global switch (`{$D-}`) written after a declaration is still read by
  `{$IFOPT}` from there on; a unit's switches never reach the unit that uses
  it (unit `UA` ending in `{$R+}{$B+}`, the program's `{$IFOPT R+}` and
  `{$IFOPT B+}` after `uses UA`: not taken). So the options at a position are
  the project's start state with every live switch directive before it
  applied in order, an include's at its `{$I}` (probed, dcc64 37.0,
  2026-10-15, PasTree local/probe-b67 P1, P2).

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
- `{$IFDEF}`, `{$IFNDEF}`, `{$DEFINE}` and `{$UNDEF}` read the **leading
  identifier** of their argument and ignore whatever follows it - dcc-probed
  (35.0 and 37.0 agree): `{$ifdef CPU386)}` tests `CPU386` (Delphi 11's
  `getmem.inc` ships that stray paren), `{$DEFINE FOO BAR}` defines only
  `FOO`, `{$UNDEF FOO)}` undefines `FOO`.
- ⚠️ *A conditional symbol's name is an identifier (B.3), case-insensitive
  beyond ASCII too* (dcc64 37.0, probed 2026-10-15, PasTree local/probe-b65 D
  and DF1-DF5 over 1925 BMP letter pairs): `{$DEFINE <A-umlaut>X}` makes
  `{$IFDEF <a-umlaut>x}` true, and Cyrillic and Greek capitals match their
  small letters. The fold is the identifiers' (B.3) on 1911 of the pairs; on
  the other 14 a symbol name folds MORE: a title-case digraph (U+01C5 DZ
  with caron, U+01C8, U+01CB, U+01F2) matches its upper- and lower-case
  forms, the final sigma U+03C2 matches U+03A3, U+03F5 matches U+0395,
  U+1FBE matches U+0399, and the ohm, Angstrom and capital sharp s signs
  (U+2126, U+212B, U+1E9E) match U+03C9, U+00E5 and U+00DF - none of which
  is one identifier. The Kelvin sign and the long s still match neither `k`
  nor `s`.
- `{$IFOPT X+}`/`{$IFOPT X-}` tests the current **state of a switch directive**
  (e.g. `{$IFOPT R-}` = "if range checking is off") — ties the conditional
  pre-pass to the switch-state stack of 1.3.1. Used in the RTL (`System.pas`,
  `System.Variants`).
- ⚠️ *`{$IF}` terminator rules & `{$LEGACYIFEND}`:* historically `{$IF}` had to be
  closed with `{$IFEND}` (not `{$ENDIF}`). Modern compilers accept `{$ENDIF}` for
  `{$IF}` **unless** `{$LEGACYIFEND ON}` is in effect, which re-imposes the strict
  pairing. The conditional pre-pass must implement both modes (D13 sources set it:
  `Xml.*`, DUnit). The precise rule (dcc64 37.0, 2026-10-30, spec audit g1,
  local/probe-park/spec-audit/g1 Leg1-Leg4.dpr): in the default mode the two
  closers are interchangeable both ways - `{$IFDEF}`/`{$IFNDEF}` closed by
  `{$IFEND}` compiles too. Under `{$LEGACYIFEND ON}` each opener takes its
  own: `{$IF}...{$ENDIF}` is `E2029 $IFEND expected but $ENDIF found`,
  `{$IFDEF}...{$IFEND}` is `E2029 $ENDIF expected but $IFEND found`; the
  wrong closer still closes (a further `{$IFEND}` after it is `E1030 Invalid
  compiler directive: '$IFEND'`), and a False `{$IF}` is checked the same.
- Symbols come from `{$DEFINE}`, project options, built-ins (`MSWINDOWS`,
  `CPUX64`, etc.) and the program's application type (`CONSOLE`, below).
- ⚠️ *The built-in set is the compiler's, and the documentation's list is wrong
  in both directions.* Probed on every installed dcc 37.0 with `--no-config`
  over 1056 symbols (every one the Studio source tests plus the documented
  ones), 2026-09-29. Every target defines `CONDITIONALEXPRESSIONS`, `UNICODE`,
  `VER370`, `DCC`, `NATIVECODE`, `MANAGED_RECORD`, `WEAKREF`, `WEAKINTFREF` and
  `WEAK_NATIVEINT`; then:

  | Compiler | Adds |
  |---|---|
  | dcc32 | `MSWINDOWS WIN32 CPU386 CPUX86 CPU32BITS ASSEMBLER UNDERSCOREIMPORTNAME` |
  | dcc64 | `MSWINDOWS WIN64 CPUX64 CPU64BITS ASSEMBLER` |
  | dccosx64 | `MACOS MACOS64 OSX OSX64 POSIX POSIX64 CPUX64 CPU64BITS EXTERNALLINKER LLVM PIC` |
  | dccosxarm64 | `MACOS MACOS64 OSX OSX64 POSIX POSIX64 CPUARM CPUARM64 CPU64BITS EXTERNALLINKER LLVM PIC` |
  | dcciosarm64 | `IOS IOS64 MACOS MACOS64 POSIX POSIX64 CPUARM CPUARM64 CPU64BITS EXTERNALLINKER LLVM PIC` |
  | dcciossimarm64 | the same plus `IOSSIMULATOR` |
  | dccaarm | `ANDROID ANDROID32 POSIX POSIX32 CPUARM CPUARM32 CPU32BITS EXTERNALLINKER LLVM PIC` |
  | dccaarm64 | `ANDROID ANDROID64 POSIX POSIX64 CPUARM CPUARM64 CPU64BITS EXTERNALLINKER LLVM PIC` |

  No target defines `CPUINTEL` (not Win32, Win64 or macOS Intel), Android 32
  defines `ANDROID32` (not `ANDROID32ARM`), both macOS targets define `OSX` and
  `OSX64` (the Arm one too), and `UNDERSCOREIMPORTNAME` is dcc32's alone. The
  RTL takes other branches for the difference: `System.SysUtils` declares
  `TNativeIntHelper`/`TNativeUIntHelper` only `{$IF not defined(WEAK_NATIVEINT)}`
  (so there are none - §15.3.2), `System.TypInfo` and
  `System.Generics.Collections` have `WEAKREF` code. Not probed here: Linux64,
  WinArm64 (no compiler installed).
- ⚠️ *The switch start state* (what `{$IFOPT}` sees before any directive) is the
  same on every target: `A C D G H I L N O P V X Y` on, the rest off - `N+`
  included, although `$N` has done nothing for decades. That is the
  no-config state; the command line's `-$` switches change it for every
  unit, by the in-file rules (1.3.1): `-$D0` is `D-`, `-$D1` `D+`, `-$YD`
  `Y+`, `-$A4` `A+` with the cap 4, `-$Z2` `Z+`. A `.dproj` reaches dcc
  as exactly such switches - msbuild turns `DCC_Optimize=false` into
  `-$O-`, `DCC_DebugInformation=0` into `-$D0`, `DCC_SymbolReferenceInfo=1`
  into `-$YD`, `DCC_Alignment=4` into `-$A4` - so a Release configuration
  takes the `{$IFOPT D-}` branches (probed, dcc64 37.0 and msbuild,
  2026-10-15, PasTree local/probe-b62 Q, ms).
- ⚠️ *Where `CONSOLE` comes from.* It is no target's built-in: dcc defines
  it in every unit compiled with a program whose last live
  `{$APPTYPE CONSOLE}` stands before its `uses` clause (after it, the units
  are already compiled; a later `{$APPTYPE GUI}` before `uses` takes it
  back), but not in the program's own text. `-CC` (a `.dproj`'s
  `DCC_ConsoleTarget`) defines it in the program's text and in the units,
  unless the program's `{$APPTYPE GUI}` before `uses` takes it from the
  units; `-CG` is the same as nothing (probed, dcc64 37.0, 2026-10-15,
  PasTree local/probe-b62 con C1-C6).
- ⚠️ *`{$DEFINE}`/`{$UNDEF}` (and switch changes) are LOCAL to the unit being
  compiled* — a batch preprocessor must reset to the project define-set per
  file. Leaking one unit's defines into the next mis-branches conditional
  chains (empirically: System.ObjAuto.pas selects the wrong ASM variant).
- ⚠️ *Undocumented dcc tolerances* (all shipped in `System.ObjAuto.pas`,
  verified against RTL 13.0 — a conforming parser must accept them):
  1. **Multiple `{$ELSE}` in one chain** — `{$IF}…{$ELSE}…{$ELSE OTHERCPU}…{$ENDIF}`
     compiles; each `$ELSE` activates iff no earlier branch was taken.
  2. **Trailing junk after a `{$IF}` expression** is ignored —
     `{$IF SizeOf(Extended) >= 10)}` (stray `)`) compiles. The rule is
     "the first complete expression decides": `{$IF Defined(A) Defined(B)
     and Defined(C)}` (a missing `and`, in Spring4D) is `Defined(A)`,
     `{$IF 0 > 0)}` is False, `{$IF Defined(X) any words}` is `Defined(X)`,
     `{$IF SizeOf(Pointer) = 8 SizeOf(Integer) = 2}` is the first comparison.
     An operator left without its right operand is not junk: `{$IF not
     Defined(X) or}` is E2015 (probed, dcc64 37.0).
  3. Trailing text after `{$ELSE}`/`{$ENDIF}`/`{$IFEND}` is an ignored
     comment (`{$ENDIF OTHERCPU}`, `{$ELSE !CPUX86}`) — widely used.
- ⚠️ *`{$IF}` sees unit constants:* the real compiler evaluates `$IF` with
  visibility of constants/enums (`{$IF Ord(soBeginning) = STREAM_SEEK_SET}`,
  `Xml.adomxmldom.pas`) — full evaluation is only possible **after** semantic
  analysis. A standalone preprocessor must define a fallback policy for such
  expressions (e.g. evaluate to False and flag).
- ⚠️ *What "visible to `{$IF}`" actually means — position, not existence.* The
  expression is evaluated against the symbol table **as of that point in the
  file**, so a constant is only usable if it is declared *textually above* the
  directive. All three of these read as UNRESOLVED even though the name exists
  in the unit:

  ```pascal
  {$IF LateConst = 7}   // <- does NOT see the line below
  const LateConst = 0;
  ```

  the same directive in the *interface* over a constant in the *implementation*,
  and a constant in an enclosing routine's scope. Only an **untyped** constant
  resolves: a `var`, a **typed** constant (`const C: Integer = 7`), a function
  and a type name all fail to resolve too. Qualified constants DO resolve
  normally (`{$IF UConst.QC > 1}`, `{$IF System.MaxInt > 1}`) — but a qualified
  name whose unit is known and whose member is not is a hard **E2003**, not a
  fallback.
- ⚠️ *A qualifier in `{$IF}` is a unit's name exactly as in code (1.2.3)* -
  the name the `uses` clause WROTE, whole, or `System`, `SysInit`, or the
  unit's own - and the name may go on through a type: `{$IF Ns.QBase.QC =
  5}` with `uses Ns.QBase` (three parts), `{$IF Ord(UC.TE.e3) = 2}`, `{$IF
  System.SysUtils.MinDateTime < 0}`. The unit's OWN name reaches the
  implementation's declarations too, and by position like a bare name: in
  unit UQ, `{$IF UQ.IC > 1}` over an implementation `const IC = 5` above it
  is True, and `{$IF P3.Late > 1}` above `const Late` is E2003.
  `System.CompilerVersion` is the compiler's value like the bare name
  (System.pas declares it `0.0`, dcc assigns it), `System.RTLVersion`
  System's constant. Probed dcc64 37.0, 2026-10-14 (PasTree
  local/probe-b58 P1, P3, UQ).
- ⚠️ *An unresolvable name does not evaluate to False — the verdict depends on
  where it sits, and one case ABORTS the whole expression.* This is the single
  most surprising rule in the pre-pass, and any tool that models `$IF` has to
  reproduce it or it will silently pick the other branch:

  | shape | verdict |
  |---|---|
  | arithmetic (`X+1=1`, `1 shl X = 2`, `-X = 0`, `+X = 1`, `X div 2 = 0`) | **True** |
  | relational vs a numeric literal/const, either side (`X>1`, `1=X`, `X>KnownConst`, `X=$10`) | **True** |
  | `X in [1,2]`, `1 in [X]` | **True** |
  | a cast or an intrinsic over it (`Byte(X) = 1`, `Integer(X) <> 0`, `Ord(X) = 1`, `Odd(X)`, `Abs(X) = 1`, `High(X) > 1`) | **True** |
  | unresolvable vs unresolvable (`X > Y`) | **True** |
  | bare boolean (`X`, `(X)`) | **False** |
  | `not X`, `not not X`, `not (X >= '1.0')`, `(not X) = False` | **False** |
  | vs a string literal (`X >= '1.0'`, `'1.0' <= X`) | **False** |
  | vs a char literal (`X = 'a'`, `X = #65`) | **False** |
  | vs a Boolean literal (`X = True`, `X = False`, `not (X = True)`) | **False** |
  | any **dotted** name whose prefix is unknown, in EVERY shape - arithmetic and `not` included (`A.B`, `A.B > 1`, `A.B < 5`, `A.B = 0`, `A.B + 1 = 1`, `not A.B`, `A.B in [1]`, `A.B.C > 1`) | **False** |

  "Unresolvable" covers more than a name declared nowhere: a `var` and a
  typed constant are the same to `{$IF}` - `V > 1`, `TC <> 7` and `TC = 7`
  (with `const TC: Integer = 7`) all abort True, while the untyped `UC <> 7`
  is False; `{$IF V}` bare is E2012. An error token stays one under `not`:
  the inner False is not inverted (probed dcc64 37.0, 2026-10-14, PasTree
  local/probe-b59 P1, P2).

  The True cases are an **abort**, not a value: the verdict propagates to the
  top of the directive and the rest of the expression is discarded. All four of
  these take the TRUE branch —

  ```pascal
  {$IF not (X > 1)}        {$IF not not (X > 1)}
  {$IF ((X > 1))}          {$IF (X > 1) and False}
  ```

  — so `X > 1` and `not (X > 1)` are True *simultaneously*. Short-circuit still
  protects, because dcc evaluates strictly left to right and never reaches the
  name: `{$IF False and (X > 1)}`, `{$IF Defined(NOPE) and (X > 1)}` and
  `{$IF Declared(X) and (X > 1)}` are all False.

  The False cases are not a value either — they are an error token, silently
  swallowed at the top of the directive but a hard **E2015 Operator not
  applicable** the moment it has to feed `and`/`or`: `{$IF X >= '1.0'}`
  compiles and is False, `{$IF (X >= '1.0') and True}` does not compile.
- *What `{$IF}` folds - a constant expression's full arithmetic* (probed
  dcc64 37.0, 2026-10-14, PasTree local/probe-b59 P1, P2):
  - strings concatenate and compare as strings: `'ab' + 'cd' = 'abcd'` is
    True and `= 'zz'` False; a constant `cVer = '10.' + '5'` makes `cVer =
    '10.5'` True and `cVer >= '10.6'` False. A string against a number is
    E2008 (`SizeOf(Integer) + 'a' = 'x'`). A `#` character is part of the
    string: `#66 = 'B'`, `'a' + #66 = 'aB'`, `'a'#66'c' = 'aBc'`.
  - `and`, `or`, `xor` and `not` are BITWISE on integers: `(6 and 1) <> 0`
    is False, `(5 xor 4) = 1`, `(1 or 2) = 3`, `not 0 = -1`, `not $FF =
    -256` True, and a constant `M = $10 or $01` is `$11`.
  - `in` over a set constructor of numbers, ranges, characters or enum
    values: `3 in [1, 2, 3]`, `3 in [1..5]`, `'b' in ['a'..'c']`, `e2 in
    [e1, e2]` True, `7 in [1, 2, 3]` False.
  - the ordinal intrinsics: `Ord('A') = 65`, `Ord(e3) = 2`, `Ord(True) = 1`,
    `Chr(65) = 'A'`, `Succ(1) = 2`, `Succ('a') = 'b'`, `Pred(e2) = e1`,
    `Abs(-3) = 3`, `Abs(-3.5) = 3.5`, `Odd(3)`; `High` and `Low` of a builtin
    ordinal type (`High(Integer)`, `Low(Integer) = -2147483648`, `High(Byte)
    = 255`, `High(Cardinal) = 4294967295`, `High(Char) = #65535`,
    `High(AnsiChar) = #255`, `High(Boolean)`, `High(NativeInt) >
    2147483647` on Win64), of an enum (`High(TE) = e3`, `Low(TE) = e1`) and
    of a subrange (`High(TSub) = 9`); `MaxInt` and `MaxLongint`.
  - a hex or binary literal above `High(Int64)` is UInt64:
    `$FFFFFFFFFFFFFFFF > 0` and `$8000000000000000 > $7FFFFFFFFFFFFFFF` are
    True, `$FFFFFFFFFFFFFFFF = -1` False, and so for a 64-digit `%1000...`;
    `&1 = 1` (the `&` before a number is ignored, B.3).
  - `SizeOf` of a generic instantiation: `SizeOf(TPair<Byte, Int64>) = 16`
    with `TPair<K, V> = record Key: K; Value: V; end`, and a constant `C4 =
    SizeOf(TPair<Byte, Byte>)` reads 2 (local/probe-b60 P1); the layout is
    9.1.2's.
- *`Length` of a static array* is a constant in `{$IF}` even though the array
  is not: `Length(V)` of a `var V: array[0..7] of Byte` and of a typed
  constant array both evaluate exactly, although `V` alone, like any `var` or
  typed constant, does not resolve (above). The bounds are constant
  expressions of their own, evaluated in full - another constant, `SizeOf`,
  `Length`: `var VS: array[0..SizeOf(TR) - 1] of Byte` with `TR = record A, B:
  Integer; end` makes `{$IF Length(VS) = 8}` True, and `var VL: array[0..
  Length(VS) - 1] of Byte` makes `Length(VL)` 8 too. A name in a bound
  resolves by position like any other: `uses U2; var A: array[0..N] of Byte;
  const N = Length(A);`, with U2 declaring `N = 3`, binds the bound to U2's
  `N` (the own one is below it), so the own `N` is 4 and `{$IF N > 3}` is
  True (probed, dcc64 37.0, 2026-10-08). The index may be any ordinal type
  and the array type a named one: `array[TE]` of a three-value enum is 3,
  `array['a'..'e']` 5, `array[Boolean]` 2, `array[Byte]` 256, a constant
  `A3: TArr` with `TArr = array[1..4] of Byte` 4, and `array[TE, 0..9]` its
  first dimension, 3. Length of an untyped string constant and of a string
  literal fold too: `const S = 'abc'` gives `Length(S) = 3`, and
  `Length('xy')` is 2. `Length(TArr)`, a type name, is E2029 (probed dcc64
  37.0, 2026-10-14, local/probe-b58 P2).
- ⚠️ *`SizeOf` of an unresolvable type is `4`* — a real value, not an abort, so
  `{$IF SizeOf(TNoSuchType) <> 32}` is True while `{$IF SizeOf(TNoSuchType) = 32}`
  is False. `Declared(X)`/`Defined(X)` likewise answer a plain False and never
  abort, which is what makes the `{$IF not Declared(X)}` fallback idiom work at
  all. (Every rule in these three bullets probed by executing both branches, on
  dcc64 36.0 (D12) and 37.0 — byte-identical results, so this is long-standing
  behaviour and not a recent change.)
- ⚠️ *`Declared(X)` is the same circularity, and it is not a rare corner:* it
  asks whether an identifier is in scope, and the scope is built from the token
  stream this very directive decides. It is the standard portability guard —
  `{$IF not declared(UInt64)} UInt64 = QWord; {$IFEND}` — so a fallback of
  "False, and flag it" takes the wrong branch precisely where the name DOES
  exist, which is the normal case on a current compiler. 81 sites in one
  vendor's RTL+VCL+FMX sources.

  Getting the right answer takes two stages, and the split is what makes the
  cost bearable:
  1. **Compiler-provided names can be answered immediately** — they need no
     symbol table at all, and they cover the two commonest spellings
     (`Declared(AnsiChar)`, `Declared(UInt64)`), which happen to sit in the
     largest units. Only a POSITIVE is final here: a name the seed lacks may
     still come from an import. So the query needs three states — yes, no, and
     *ask someone else* — and the third records the name instead of guessing.
  2. **Everything else needs a second pass** over just those units, once their
     imports have been analyzed and before anything holds references into the
     models being replaced.

  Two details that are easy to get wrong, both found by implementing it:
  - the argument is a **designator, not an identifier**
    (`Declared(System.Embedded)`, in a platform header unit). Stopping at the
    dot asks about `System`, which is a unit name and answers True — the
    opposite branch. It resolves by the qualifier rule above and never
    errs: `Declared(UC.QC)`, `Declared(System.TObject)`,
    `Declared(SysInit.HInstance)`, `Declared(System.SysUtils.Exception)`,
    `Declared(System.Length)` (an intrinsic, 1.2.4) and `Declared(UC.TE.e1)`
    are True; `Declared(SysUtils.Exception)` with `uses System.SysUtils` (a
    last segment does not qualify), `Declared(UC.Missing)` and
    `Declared(Nope.X)` are False. A unit's name is declared
    (`Declared(System.SysUtils)`, `Declared(UQ)` in UQ). The own unit's name
    answers by position like a bare name, from the unit-level declarations
    only: `Declared(UQ.Later)` above `const Later` is False, and in a routine
    declaring `const Loc`, `Declared(P3.Loc)` is False where `Declared(Loc)`
    is True (probed dcc64 37.0, 2026-10-14, local/probe-b58 P1, P3, UQ).
  - the second pass must **not** consult the unit's OWN declarations WHOLE.
    The guard idiom declares the name inside the text it guards, so
    answering from the previous pass's model makes the answer flip every
    round: declared, so skip the text, so not declared, so take the text.
  - ...but dcc itself answers the unit's own names BY POSITION, like
    constants above: a name declared textually ABOVE the directive and in
    scope there is declared - in the interface or the implementation, and a
    routine's LOCAL declaration inside that routine's body (`var temp:
    PTimSort;` then `{$IF Declared(temp)} temp := ts; {$IFEND}` in the body,
    Spring4D) - while a name declared below, or inside its own `not
    Declared` guard, is not until after it. Leaving the own scope out
    entirely drops real code (a Windows header's guards over its own earlier
    record types: E2065). A lookup of the own declarations that lie before
    the directive's offset is exact and cannot oscillate: the guarded
    declaration is always after its guard (probed, dcc64 37.0).
  - The scope is the one in effect at the directive (probed 2026-09-27,
    dcc64 37.0): in a routine's body its locals, parameters, local
    constants, the routine's own name and - in a method - the class's
    fields are declared; a local of ANOTHER routine, earlier or later, is
    not, nor a local of a routine NESTED in this one; an inline `var` counts
    from its own declaration on. A forward declaration counts (`TFoo =
    class;` above, the full `TFoo` below: declared between them). The
    unit's own name and a name in its `uses` clause are declared. A
    generic's bare name is NOT: with only `TG<T>` declared,
    `Declared(TG)` is False - the same rule that makes a bare name skip a
    same-named generic (16.1.2).
  - "Above the directive" means in the stream being DECIDED, not in the
    first pass's: an earlier guard the second pass flips can add or remove
    a declaration a later guard asks about. FMX.Skia.Canvas declares a
    record under `not DECLARED(RTLVersion132)` - guessed True on the first
    pass, False once System answers - and asks `DECLARED(<that record>)`
    before using it; read off the first-pass model, the use stays and the
    record goes (E2003). A guard reads only what lies above it, so the
    decisions settle top down: re-decide until the own-name answers agree
    with the stream they produced.
  - "Above" is the order of the stream, includes spliced in: an `{$I}`
    BELOW the directive puts its text below it, however the includer's
    own lines run. `{$IF Declared(Foo)} {$DEFINE HAS_FOO} {$IFEND}` then
    `{$I foo.inc}` (which declares `Foo`) then `{$IFNDEF HAS_FOO} const
    Bar = 1; {$ENDIF}` compiles with `Bar` declared: the guard is False.
    The `not Declared(Foo)` form takes its text (probed, dcc64 37.0,
    2026-10-15, local/probe-b61 inc1, inc2).
  - A unit the decided branch newly imports (`{$IF Declared(KFlag)} uses
    UNew; {$IFEND}`) is an import like any other: its own imports are part
    of the closure, its identifiers are checked, and its own guards are
    decided the same way - dcc64 reports an undeclared identifier in UNew
    and binds an inherited member a class in the importer takes through
    UNew's ancestor from UNew's own import (probed, dcc64 37.0, 2026-10-15,
    local/probe-b61 sub2, sub2b).
  - The same holds for a constant's VALUE in a `{$IF}`: the declaration read
    is the one in the decided stream, above the directive, else an
    import's. Spring4D's shape: `{$IF not Declared(CPP_ABI_ADJUST)} const
    CPP_ABI_ADJUST = 0; {$IFEND}` then `{$IF CPP_ABI_ADJUST > 0)}` - System
    declares `CPP_ABI_ADJUST = 3 * SizeOf(Pointer)` when it defines
    `CPP_ABI_SUPPORT` itself (`{$IF DEFINED(WIN64) or
    DEFINED(EXTERNALLINKER)}` in System.pas), so the guard drops the own 0
    and dcc64 takes the second branch with 24, dcc32 not (0). Read off the
    first pass's model - which guessed the guard the other way and holds the
    own 0 - the value is stale (probed, dcc32/dcc64 37.0, 2026-09-30).
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
  positions/diagnostics. ⚠️ dcc has no insertion form (that is Free
  Pascal's): `{$I %DATE%}` names a file, and without one is `F1026 File not
  found: '%DATE%.pas'` (probed, dcc64 37.0, 2026-10-15). A missing include
  is F1026 and stops the compile.
- Beware include cycles; cap recursion. dcc's is `E2245 Recursive include
  file <path>`, fatal.
- ⚠️ *A name with no extension gets `.pas`* - before it is looked for
  anywhere, quoted or not, `-I` included: `{$I inc2}` reads `inc2.pas` with a
  file named plain `inc2` beside it, `{$I inc3}` with only `inc3.inc` is
  `F1026 File not found: 'inc3.pas'`, and with only a bare `inc4` it is
  F1026 `'inc4.pas'` too - never the bare name, never `.inc`. A name with an
  extension of its own (`inc5.defs`) is read as written (probed, dcc64 37.0,
  2026-10-15, PasTree local/probe-b63 ext P1-P7).
- ⚠️ *Where the file is looked for - beside the UNIT, not beside the include
  that names it.* dcc32/dcc64 37.0 probed with an elimination loop (every
  candidate directory holds the file, each run deletes the one taken), for an
  include named by the unit and for one named by an include, plain and with a
  subdirectory (`sub\inner.inc`): beside the unit (or program) being compiled,
  then the current directory, then `-I`, then the project file's directory -
  then F1026. Never beside the include file that names it, never on `-U`
  alone. So `inc\outer.inc` doing `{$I inner.inc}` reads the `inner.inc` next
  to the unit, not `inc\inner.inc`, and with only the latter present the unit
  does not compile. (2026-09-30.)
- ⚠️ *A conditional never crosses an include boundary.* dcc64 37.0 probed
  (2026-10-02): an include that opens `{$IFDEF}`/`{$IF}` and leaves it open
  is `E2280 Unterminated conditional directive` at the include (live or
  dead branch alike), even when the includer's `{$ENDIF}` follows; an
  include's `{$ENDIF}`/`{$IFEND}` does not close the includer's conditional
  (E2280 at the includer), and with no conditional open anywhere it is
  `E1030 Invalid compiler directive`; an include's `{$ELSE}`/`{$ELSEIF}`
  under the includer's live conditional skips the rest of the include and
  leaves the includer's text after it misread (E2029 further down). An
  include that opens and closes its own conditional is ordinary.

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
- A `{$POPOPT}` with no snapshot is a warning, not an error: `W1078
  {$POPOPT} with no matching {$PUSHOPT} in <file>`, and the compile goes on
  (probed, dcc64 37.0, 2026-10-15). The opposite, a `{$PUSHOPT}` still open
  when the unit ends, is a warning too, reported at the end of the unit:
  `W1079 Previous {$PUSHOPT} on line 4 not restored`; the snapshot dies with
  the unit (its switches never reach an importer anyway, 1.3.1) - dcc64
  37.0, 2026-10-30 (spec audit g1, local/probe-park/spec-audit/g1
  PushOpen.dpr, UPush.pas).

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
  `VER360`-style symbols are auto-`{$DEFINE}`d - exactly ONE, the
  compiler's own: dcc64 37.0 defines `VER370` and neither `VER360` nor
  `VER371`. A point release keeps the number (the installed build, dcc64
  37.0.57242: `CompilerVersion = 37.0` and System's `RTLVersion = 37.0`
  hold, no `RTLVersion131`/`RTLVersion132` is declared), so "this version or
  later" is written `{$IF CompilerVersion >= 36}`, never as a list of
  `VERnnn` (dcc64 37.0, 2026-10-30, spec audit g1,
  local/probe-park/spec-audit/g1 VerSym.dpr). The pre-pass must seed these per
  target so `{$IF}` evaluates correctly.
- Cross-reference [Appendix A](A-version-history.md) for the version↔number map.
