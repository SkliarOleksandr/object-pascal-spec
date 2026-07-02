# Appendix A — Object Pascal Version History

Source-of-truth table for the **Introduced** / **Deprecated** tags used
throughout this reference. "Key language additions" lists only *language &
compiler* milestones — not RTL/VCL/IDE.

> All rows verified against official "What's New" pages / Embarcadero blog
> posts as of 2026-07-02.

| Version | Codename | Year | Key language / compiler additions |
|---------|----------|------|-----------------------------------|
| Turbo / Wirth Pascal | — | pre-1995 | Core Pascal: types, records, sets, pointers, `for`/`while`/`repeat`, units, `with`, `goto` |
| Delphi 1 | — | 1995 | Object Pascal: classes (`TObject` model), exceptions, properties, RTTI (classic), 16-bit |
| Delphi 2 | — | 1996 | 32-bit; long (`AnsiString`) strings; `Variant`; `Currency` |
| Delphi 3 | — | 1997 | **Interfaces** (`IInterface`/reference counting); packages; method resolution clauses |
| Delphi 4 | — | 1998 | **Dynamic arrays**; **method overloading**; **default parameters**; `Int64` |
| Delphi 5 | — | 1999 | (minor language changes) |
| Delphi 6 | — | 2001 | `Variant` improvements; cross-platform (Kylix) groundwork |
| Delphi 7 | — | 2002 | Compiler warnings/hints maturity; `deprecated`/`platform`/`library` hint directives |
| Delphi 8 | — | 2003 | .NET compiler (language extensions for .NET; later dropped) |
| Delphi 2005 | — | 2004 | **Inline functions** (`inline`); **`for-in` loop**; nested types/constants; multi-unit namespaces groundwork |
| Delphi 2006 | (BDS) | 2005 | **Records with methods**; **operator overloading** (on records); **class helpers**; **strict private/protected**; **`sealed`/`final`/`abstract`** classes; class `var` |
| Delphi 2007 | — | 2007 | (minor; Unicode prep) |
| Delphi 2009 | — | 2008 | **Generics** (Win32 native); **anonymous methods**; **Unicode `string` = `UnicodeString`**; `AnsiString` with codepage; ` rtti` foundations |
| Delphi 2010 | — | 2009 | **Extended RTTI**; **custom attributes**; **class constructors/destructors**; `delayed` external |
| Delphi XE | — | 2010 | Regular-expressions RTL (lib); language stable |
| Delphi XE2 | — | 2011 | **64-bit Windows compiler**; FireMonkey; `PByte`-style pointer math contexts |
| Delphi XE3 | — | 2012 | **Record helpers for intrinsic types** (string, Integer, Double, …) — confirmed |
| Delphi XE4 | — | 2013 | **Mobile (iOS) ARM compiler**; **ARC** (automatic reference counting) for objects on mobile 🧪; `String` immutability rules on mobile |
| Delphi XE5 | — | 2013 | Android ARM compiler |
| Delphi XE6 | — | 2014 | (minor) |
| Delphi XE7 | — | 2014 | **Dynamic array** `+` concatenation & literal init for managed types |
| Delphi XE8 | — | 2015 | `FixedInt`/`FixedUInt`; extended type helpers reach |
| Delphi 10.0 | Seattle | 2015 | (minor) |
| Delphi 10.1 | Berlin | 2016 | **`[weak]`/`[unsafe]`/`[volatile]` attributes on ALL compilers** (previously mobile/ARC-only); `UTF8String`/`RawByteString` restored on mobile |
| Delphi 10.2 | Tokyo | 2017 | **Linux (x64) compiler**; (minor language) |
| Delphi 10.3 | Rio | 2018 | **Inline variables** (`var x := …` in a block) + **type inference**; mobile ARC **deprecated** (path to removal) 🧪 |
| Delphi 10.4 | Sydney | 2020 | **Custom managed records** (record `operator Initialize/Finalize/Assign`); **ARC removed** — unified manual memory model on all platforms |
| Delphi 11 | Alexandria | 2021 | **Binary literals** (`%1010`); **digit separator** (`1_000_000`) |
| Delphi 12 | Athens | 2023 | **Multiline / long string literals** (triple-quote `'''`); weak type alias for `NativeInt`; NaN comparison handling; disabling FP exceptions across platforms |
| Delphi 13 | Florence | 2025 | **Inline `if` (ternary) operator**; **`NameOf`** intrinsic; **`{$PUSHOPT}`/`{$POPOPT}`** directives; **`is not` / `not in`** operators; **`noreturn`** directive; implicit `Self` in record `Initialize`/`Finalize` operators; 64-bit IDE (not language) |
| Delphi 13.1 | Florence | 2026 | **Arm64EC compiler** (Windows-on-Arm, LLVM 20) — *no new syntax*; compiler fixes: AnsiChar/ShortString overload regression, ternary-`if` type-compatibility improvements; new warning W1080 (`noreturn` proc with return path) |

## Notes on contentious version tags

- **Operator overloading**: introduced for *records* in Delphi 2006; classes do
  not support operator overloading.
- **Record helpers for intrinsic types**: **XE3, confirmed** (official docs +
  release coverage) — `TStringHelper`, `42.ToString` etc. date from XE3.
- **`[weak]`/`[unsafe]`/`[volatile]`**: introduced with mobile ARC (XE4-era),
  extended to **all compilers in 10.1 Berlin, confirmed**; survived ARC removal.
- **ARC** (🧪): lived only on the mobile/NextGen compilers (XE4 → 10.3),
  fully **removed in 10.4**. Treat all ARC-specific syntax as historical.
- **Delphi 13.1**: **confirmed no new language syntax** — Arm64EC is a new
  target/toolchain; the compiler changes are fixes (AnsiChar/ShortString
  overloads, ternary-`if` type compatibility) plus warning W1080.
- **`constref` is NOT Delphi** — it is FreePascal's equivalent of Delphi's
  `const [Ref]`. Confirmed: D13 sources use it only in `{$IFDEF FPC…}` branches.
