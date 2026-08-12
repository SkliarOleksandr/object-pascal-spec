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
| Delphi XE2 | — | 2011 | **64-bit Windows compiler**; FireMonkey; `PByte`-style pointer math contexts; initial macOS compiler (book-stated, lower confidence, not independently dcc-verified — see note below) |
| Delphi XE3 | — | 2012 | **Record helpers for intrinsic types** (string, Integer, Double, …) — confirmed |
| Delphi XE4 | — | 2013 | **Mobile (iOS) ARM compiler**; **ARC** (automatic reference counting) for objects on mobile 🧪; `String` immutability rules on mobile |
| Delphi XE5 | — | 2013 | Android ARM compiler |
| Delphi XE6 | — | 2014 | (minor) |
| Delphi XE7 | — | 2014 | **Dynamic array** `+` concatenation & literal init for managed types |
| Delphi XE8 | — | 2015 | `FixedInt`/`FixedUInt`; extended type helpers reach; first ARM 64-bit compiler, targeting iOS (book-stated, not independently dcc-verified — see note below) |
| Delphi 10.0 | Seattle | 2015 | (minor) |
| Delphi 10.1 | Berlin | 2016 | **`[weak]`/`[unsafe]`/`[volatile]` attributes on ALL compilers** (previously mobile/ARC-only); `UTF8String`/`RawByteString` restored on mobile |
| Delphi 10.2 | Tokyo | 2017 | **Linux (x64) compiler**; (minor language) |
| Delphi 10.3 | Rio | 2018 | **Inline variables** (`var x := …` in a block) + **type inference**; mobile ARC **deprecated** (path to removal) 🧪 |
| Delphi 10.4 | Sydney | 2020 | **Custom managed records** (record `operator Initialize/Finalize/Assign`); **ARC removed** — unified manual memory model on all platforms |
| Delphi 11 | Alexandria | 2021 | **Binary literals** (`%1010`); **digit separator** (`1_000_000`); native macOS ARM64 (Apple M1) codegen (book-stated, not independently dcc-verified — see note below) |
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
- **XE8 ARM64/iOS, Delphi 11 macOS ARM64/M1, XE2 macOS compiler — book-stated,
  NOT dcc-verified.** These are compiler-release-history claims from Marco
  Cantù's *Object Pascal Handbook* (Appendix A, p.523-525), not language-
  semantics rules a single `dcc32 37.0` invocation can probe — there is no
  "compile this and see" test for "which Delphi version first shipped an
  ARM64 codegen." Taken at face value from the book text. The only
  corroboration attempted: the currently-installed Delphi 13.1 (`dcc32 37.0`)
  toolchain at `C:\Program Files (x86)\Embarcadero\Studio\37.0\lib\` has
  distinct `iosDevice64`/`iossim64`/`iossimarm64` (ARM64 iOS) and
  `osx64`/`osxarm64` (ARM64 macOS) directories, so ARM64 iOS and ARM64 macOS
  targets do exist somewhere in this toolchain's lineage — consistent with,
  but not proof of, the book's specific version attributions (XE8 for iOS
  ARM64, Delphi 11 for macOS ARM64). No installed-SDK evidence was found that
  pins down XE2's claimed "initial macOS compiler" specifically (that predates
  ARM64 entirely — it would have been a 32/64-bit Intel macOS target, since
  superseded), so that row is flagged lower-confidence per the book's own
  phrasing (it introduces macOS support in the same sentence as Win64, but
  with noticeably less specificity than the ARM64 claims).
