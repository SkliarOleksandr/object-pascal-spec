# object-pascal-spec

**Object Pascal language specification (Delphi → 13.1)** — written as the basis
for an AST parser.

> **Disclaimer.** This is an unofficial, community-authored specification of the
> Object Pascal language as implemented by Embarcadero Delphi. It is not
> affiliated with, endorsed by, or supported by Embarcadero Technologies.
> "Delphi" and related marks are trademarks of Embarcadero Technologies, Inc.
> All descriptions are original prose documenting publicly observable language
> behavior; no proprietary documentation text is reproduced.

A structured, feature-by-feature catalogue of the **Object Pascal language** as
implemented by Embarcadero Delphi, from the original 1995 dialect up to
**Delphi 13.1 Florence**.

> **Scope:** *language and compiler features only* — syntax, types, type system,
> OOP model, generics, RTTI, memory model, compiler directives.
> **Out of scope:** the RTL/VCL/FMX class libraries and the IDE. A class is only
> mentioned where it is *part of the language contract* (e.g. `TObject`,
> `IInterface`, `TValue`).

## Purpose — this is a parser/AST specification

The primary consumer of this document is **a hand-written lexer + parser that
builds an AST for Object Pascal**, not an end-user tutorial. Therefore every
feature must capture the information needed to *recognise and interpret* the
construct:

- **Concrete syntax** — an EBNF production precise enough to drive parsing.
- **Lexical facts** — which tokens are involved, reserved words vs. directives.
- **Disambiguation** — how the parser resolves grammar conflicts (dangling
  `else`, generic `<` vs. less-than, call vs. reference, `with` scoping, etc.).
- **Static semantics** — scope & name resolution, type/assignment compatibility,
  overload & operator resolution, evaluation order, what is legal where.
- **AST shape hints** — the node kind(s) the construct should lower to and the
  meaningful children/attributes a tree-builder needs to retain.

Prose descriptions stay short; the **Grammar** and **Semantics & parsing notes**
blocks are where the value lives.

## Primary sources

1. **Embarcadero DocWiki — Delphi Language Reference** (base, authoritative for syntax).
   https://docwiki.embarcadero.com/RADStudio/en/Delphi_Language_Reference
2. **"What's New" per release** (authoritative for *which version* a feature landed in),
   plus the official Embarcadero blog posts announcing language changes.

> Everything for **12 Athens** and **13 / 13.1 Florence** comes from the official
> "What's New" pages and Embarcadero blog announcements.

---

## How this reference is organised

The language is split into **20 chapters + 2 appendices**, one Markdown file each.
Grouping is *logical* (by concept), not chronological.

| # | File | Scope |
|---|------|-------|
| 01 | [01-program-structure.md](01-program-structure.md) | Units, programs, namespaces, comments, identifiers, keywords, compiler directives, conditional compilation, include files |
| 02 | [02-fundamental-types.md](02-fundamental-types.md) | Ordinal, integer, char, boolean, real, enumerated, subrange, set types, type aliases |
| 03 | [03-variables-constants.md](03-variables-constants.md) | Variables, inline variables, type inference, constants, typed constants, scope & lifetime |
| 04 | [04-expressions-operators.md](04-expressions-operators.md) | Operators, precedence, type casts & conversions, intrinsic routines |
| 05 | [05-statements.md](05-statements.md) | **(EXEMPLAR — fully populated)** if, inline-if/ternary, case, for, for-in, while, repeat, with, goto, break/continue |
| 06 | [06-routines.md](06-routines.md) | Procedures/functions, parameters, overloading, default params, inlining, calling conventions, procedural types, external, `noreturn` |
| 07 | [07-strings.md](07-strings.md) | String types, Unicode, char, string literals (incl. multiline), helpers, encodings |
| 08 | [08-arrays.md](08-arrays.md) | Static, dynamic, multidimensional, open arrays, `array of const` |
| 09 | [09-records.md](09-records.md) | Records, variant records, methods, operator overloading, custom managed records, record constructors |
| 10 | [10-pointers-files.md](10-pointers-files.md) | Pointer types, typed pointers, `file` types |
| 11 | [11-classes.md](11-classes.md) | Class declaration, fields, methods, visibility, constructors/destructors, nested types & constants |
| 12 | [12-inheritance-polymorphism.md](12-inheritance-polymorphism.md) | Inheritance, virtual/dynamic, override/reintroduce, abstract, sealed/final, `is`/`as` |
| 13 | [13-properties-events.md](13-properties-events.md) | Properties (array/indexed/default), published, method pointers, events |
| 14 | [14-interfaces.md](14-interfaces.md) | Interface declaration, GUIDs, reference counting, delegation, weak/unsafe references |
| 15 | [15-class-mechanics-helpers.md](15-class-mechanics-helpers.md) | Class methods/vars/properties, class constructors/destructors, class references (metaclasses), class & record helpers |
| 16 | [16-generics.md](16-generics.md) | Generic types & methods, constraints, covariance, type inference |
| 17 | [17-anonymous-methods.md](17-anonymous-methods.md) | Anonymous methods/closures, `reference to` types, variable capture |
| 18 | [18-exceptions.md](18-exceptions.md) | try/except/finally, raise, exception hierarchy, nested/inner exceptions |
| 19 | [19-rtti-attributes.md](19-rtti-attributes.md) | Classic & extended RTTI, attributes, `TValue`, virtual method interceptors |
| 20 | [20-memory-management.md](20-memory-management.md) | Stack/heap, `New`/`Dispose`/`GetMem`, lifetime, ARC history, weak/unsafe references |
| A | [A-version-history.md](A-version-history.md) | Appendix: version → year → codename → key language additions |
| B | [B-lexical-grammar.md](B-lexical-grammar.md) | Appendix: lexical structure — token list, reserved words vs. directives, literal grammar, comments/whitespace, and the shared EBNF productions referenced by every chapter |

---

## Feature numbering

Every feature has a stable ID `C.S.F`:

- **C** — chapter number (matches the file, e.g. `05`)
- **S** — section within the chapter
- **F** — feature within the section

Example: **`5.4.1`** = chapter 5, section 4, feature 1. IDs are stable — when
inserting new features, append rather than renumber.

## Per-feature template

Each feature is written using this exact block. The **Grammar** and
**Semantics & parsing notes** blocks are mandatory — they are what the parser
is built from.

````markdown
### 5.4.1 Inline `if` expression (ternary operator)

| | |
|---|---|
| **Introduced** | 13.0 Florence (2025) |
| **Deprecated** | — |
| **Status** | ✅ Current |

One- or two-sentence description of what the feature is and when to use it.

**Grammar**

```ebnf
InlineIfExpr = "if" Expression "then" Expression "else" Expression ;
```

**Example**

```pascal
var Max := if A > B then A else B;
```

**Semantics & parsing notes**

- *Context:* this is an **expression**, not a statement — it appears wherever
  an expression is expected (RHS of `:=`, argument, etc.). Distinguish from the
  `if`-*statement* (chapter 5.3.1) by parse context.
- *Disambiguation:* `else` is mandatory here, so there is no dangling-`else`
  problem; both branch expressions must be assignment-compatible to a common type.
- *Evaluation:* only the selected branch is evaluated (short-circuit), unlike the
  `IfThen` RTL functions.
- *AST:* `ConditionalExpr { cond, thenExpr, elseExpr }`. Result type = common type
  of the two branches.
````

### Field rules

- **Introduced** — version + codename + year. If a feature predates Delphi 1
  (classic Turbo/Wirth Pascal) use `Pascal (pre-1995)`. If the exact version is
  unverified, suffix with ` (verify)`.
- **Deprecated** — the version that *officially* deprecated the feature (compiler
  warning / docs), or `—` if still current. Removed features note ` / removed in X`.
- **Status** — one of the legend values below.
- **Grammar** — one or more EBNF productions (notation below). Reference shared
  productions (`Expression`, `TypeRef`, `Block`, `Ident`…) by name rather than
  re-deriving them; the canonical set lives in [Appendix B](B-lexical-grammar.md).
- **Semantics & parsing notes** — bullets covering, as applicable: parse context,
  disambiguation/conflict resolution, scope & name resolution, type rules,
  evaluation order, and the target **AST node shape**.

### EBNF notation

| Symbol | Meaning |
|--------|---------|
| `=` … `;` | production definition / terminator |
| `"abc"` | terminal (literal token / keyword) |
| `Foo` | non-terminal (defined elsewhere) |
| `[ x ]` | optional (0 or 1) |
| `{ x }` | repetition (0 or more) |
| `( x \| y )` | grouping / alternation |
| `?desc?` | informal terminal described in prose (e.g. `?digit?`) |

Reserved words are matched case-insensitively (Object Pascal is
case-insensitive for identifiers and keywords); the lexer normalises case.

## Status legend

| Badge | Meaning |
|-------|---------|
| ✅ Current | Supported and recommended |
| ⚠️ Legacy | Works, but superseded by a newer idiom |
| 🚫 Deprecated | Officially deprecated; avoid in new code |
| ❌ Removed | No longer compiles on current versions |
| 🧪 Platform-specific | Behaviour differs / existed only on some targets (e.g. mobile ARC) |

## Version shorthand

`1` `2` … `7`, `2005`–`2010`, `XE`–`XE8`, `10.0` Seattle, `10.1` Berlin,
`10.2` Tokyo, `10.3` Rio, `10.4` Sydney, `11` Alexandria, `12` Athens,
`13`/`13.1` Florence. Full table in [Appendix A](A-version-history.md).

---

## Status of this draft

🚧 **Skeleton / work in progress.**

- [x] Structure, conventions, numbering, version legend (this file)
- [x] Parser-oriented template (Grammar + Semantics/parsing notes), EBNF legend
- [x] Appendix A — version history
- [x] Chapter 05 — exemplar (fully populated, defines the template)
- [x] Appendix B — lexical grammar & shared EBNF productions
- [x] Chapter 01 — program structure & compilation
- [x] Chapter 02 — fundamental & simple types
- [x] Chapter 03 — variables & constants
- [x] Chapter 04 — expressions & operators
- [x] Chapter 05 — statements (exemplar)
- [x] Chapter 06 — procedures & functions
- [x] Chapter 07 — strings & characters
- [x] Chapter 08 — arrays
- [x] Chapter 09 — records
- [x] Chapter 10 — pointers & file types
- [x] Chapter 11 — classes & objects
- [x] Chapter 12 — inheritance & polymorphism
- [x] Chapter 13 — properties & events
- [x] Chapter 14 — interfaces
- [x] Chapter 15 — class-level mechanics & helpers
- [x] Chapter 16 — generics
- [x] Chapter 17 — anonymous methods & closures
- [x] Chapter 18 — exceptions
- [x] Chapter 19 — RTTI & attributes
- [x] Chapter 20 — memory management

- [x] **RTL 13.0 syntax sweep** — validated the reference against the shipped RTL
      sources (`Studio\37.0\source\rtl`). Added: inline assembly (§6.10),
      `Write`/`Str` colon-formatted arguments (§4.11.2), `Slice` (§4.11),
      generic args on every qualified-name segment + generic implementation
      headers (§B.8/B.11/16.3), `class threadvar` (§15.1.2), magic attributes
      `[Ref]`/`[Volatile]` (§19.3.3), `out` form of managed-record operators
      (§9.4.1), `{$IFOPT}` + directive-families table (§1.3.1–1.3.2),
      `dependency` on external declarations (§6.7.1).

**First full draft complete** — all 20 chapters + appendices A & B populated
against the parser template and cross-checked against the RTL 13.0 sources.
Next pass: verify `(verify)`-tagged version numbers (esp. 12/13 minor releases),
cross-check grammar productions for consistency, and expand any feature the
parser work surfaces as under-specified.

---

## License

[MIT](LICENSE) © 2026 Oleksandr Skliar. See the disclaimer at the top of this
file regarding trademarks and the unofficial status of this specification.
