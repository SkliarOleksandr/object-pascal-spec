# 07 — Strings & Characters

The string type family, their memory/lifetime model (managed, copy-on-write),
indexing rules (a version-sensitive trap), and string literals (defined in
[§B.6](B-lexical-grammar.md#b6-character--string-literals)).

## Chapter grammar umbrella

```ebnf
StringType = "string" [ "[" ConstExpr "]" ]      (* generic string / ShortString *)
           | "AnsiString" [ "(" ConstExpr ")" ]  (* with explicit codepage *)
           | "WideString" | "UnicodeString" | "RawByteString" | "UTF8String"
           | "ShortString" ;
PCharType  = "PChar" | "PAnsiChar" | "PWideChar" ;
```

---

## 7.1 String types

### 7.1.1 `string` / `UnicodeString`

| | |
|---|---|
| **Introduced** | `string` Pascal; **`string` = `UnicodeString` (UTF-16) since Delphi 2009** |
| **Deprecated** | — |
| **Status** | ✅ Current |

The default string type. Since 2009 it is `UnicodeString`: a managed, reference-
counted, copy-on-write sequence of `WideChar` (UTF-16 code units).

**Example**

```pascal
var S: string;          // UnicodeString
S := 'Привет';
```

**Semantics & parsing notes**

- ⚠️ *Version-sensitive aliasing:* before 2009 `string = AnsiString`; from 2009
  `string = UnicodeString`. Resolve `string` per target version (mirrors the
  `Char` situation, ch.02).
- *Managed type:* automatic lifetime; no manual free. Affects codegen
  (init/finalize), not parsing.

### 7.1.2 `AnsiString` (with code page)

| | |
|---|---|
| **Introduced** | Delphi 2; **code-page parameter** Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

A managed 1-byte-per-element string carrying a code page.

**Grammar**

```ebnf
AnsiStringType = "AnsiString" [ "(" ConstExpr ")" ] ;   (* code page, e.g. AnsiString(1251) *)
```

**Semantics & parsing notes**

- ⚠️ The `AnsiString(N)` form takes a **code-page constant in parentheses** as part
  of the *type* — do not mis-parse it as a type-cast call. Context (declaration
  position) disambiguates.
- `RawByteString` (7.1.5) is the code-page-agnostic variant.

### 7.1.3 `ShortString` / `string[N]`

| | |
|---|---|
| **Introduced** | Turbo Pascal / D1 |
| **Deprecated** | — (legacy) |
| **Status** | ⚠️ Legacy |

A fixed-capacity, length-prefixed 1-byte string of max length `N` (≤ 255).

**Example**

```pascal
type TName = string[20];   // ShortString of capacity 20
```

**Semantics & parsing notes**

- ⚠️ `string[N]` is a **distinct type form** — the `[N]` here is a *capacity*, not
  indexing. Only allowed in a type position. Not managed (value type, fixed size).
- `string` *without* `[N]` is the managed type (7.1.1). The presence of `[N]`
  changes the type entirely — the parser must branch on it.

### 7.1.4 `WideString`

| | |
|---|---|
| **Introduced** | Delphi 3 |
| **Deprecated** | — (use `UnicodeString`) |
| **Status** | ⚠️ Legacy |

A COM `BSTR`-compatible UTF-16 string; **not** reference-counted/copy-on-write
(OLE-allocated). Use mainly for COM interop.

### 7.1.5 `RawByteString`, `UTF8String`

| | |
|---|---|
| **Introduced** | Delphi 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`UTF8String` is an `AnsiString` with code page CP_UTF8; `RawByteString` is a
code-page-agnostic `AnsiString` used for byte-preserving parameter passing.

### 7.1.6 `PChar` and pointer-to-char types

| | |
|---|---|
| **Introduced** | D1; `PChar = PWideChar` since 2009 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Null-terminated string pointers for C/OS interop: `PChar` (= `PWideChar` since
2009), `PAnsiChar`, `PWideChar`.

**Semantics & parsing notes**

- A `string` ↔ `PChar` conversion is allowed via cast and is a frequent interop
  pattern; `PAnsiChar`/`PWideChar` also enable pointer math (`{$POINTERMATH}`,
  ch.04).

### 7.1.7 `UCS4Char` / `UCS4String`

| | |
|---|---|
| **Introduced** | Delphi (System unit, long-standing) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Full 32-bit Unicode-codepoint types, for representing characters outside the
UTF-16 basic-multilingual-plane range as a single value instead of a surrogate
pair.

**Example**

```pascal
var
  C: UCS4Char;
  S: UCS4String;
begin
  C := UCS4Char($1F600);                      // one full codepoint (an emoji)
  S := UnicodeStringToUCS4String('Hello');    // string -> UCS4String
end;
```

**Semantics & parsing notes**

- Both are declared in `System` (no extra `uses` needed): `UCS4Char = Cardinal;`
  and `UCS4String = array of UCS4Char` — an ordinary **type alias** and an
  ordinary **dynamic array** type, not new reserved words or a distinct string
  form. Confirmed real and usable (dcc-verified, dcc32 37.0): declaring
  variables of both types and assigning them compiles and runs;
  `UnicodeStringToUCS4String('Hello')` (System unit) returns a 6-element
  `UCS4String` — 5 codepoints plus a trailing `0` terminator — with
  `Cardinal(S[0]) = 72` (`'H'`), `Cardinal(S[1]) = 101` (`'e'`).
- Conversion helpers live in `System`: `UnicodeStringToUCS4String`,
  `UCS4StringToUnicodeString`, `WideStringToUCS4String`,
  `UCS4StringToWideString`, `WideCharToUCS4String`, `PUCS4Chars`.
- Because `UCS4String` is a plain dynamic array, §8.2's dynamic-array rules
  (0-based, managed, `SetLength`/`Length`/`Copy`/`High`/`Low`) apply to it
  unchanged — there is no special-cased "UCS4 string" semantics to track.

---

## 7.2 String indexing & character access

### 7.2.1 Element indexing and `{$ZEROBASEDSTRINGS}`

| | |
|---|---|
| **Introduced** | 1-based Pascal; **`{$ZEROBASEDSTRINGS}`** Delphi XE3/XE5 (mobile) |
| **Deprecated** | — |
| **Status** | 🧪 Platform-historical |

`S[i]` accesses a character. Strings are **1-based** by default on desktop, but a
directive (and the mobile NextGen compilers) made them **0-based**.

**Example**

```pascal
C := S[1];      // first char, default (1-based)
{$ZEROBASEDSTRINGS ON}
C := S[0];      // first char when zero-based
```

**Semantics & parsing notes**

- ⚠️ *Directive-dependent base index.* `S[i]` semantics depend on
  `{$ZEROBASEDSTRINGS}` state at that source position. A spec-accurate
  type/bounds layer must track this directive. Prefer `Low(S)`/`High(S)` to stay
  base-agnostic.
- Indexing yields a `Char`; assignable for mutable string types (triggers
  copy-on-write unique-ification).
- ⚠️ *`S.Chars[i]` (the record-helper property, §7.4.1) is a separate, ALWAYS
  0-based, read-only path — unaffected by `{$ZEROBASEDSTRINGS}`.* `S[1]` under
  the default directive and `S.Chars[0]` name the same character; flip
  `{$ZEROBASEDSTRINGS ON}` and `S[0]` now names it too, but `S.Chars[0]` still
  does — it never moves. Do not let a bounds/index layer that tracks the
  directive for `S[i]` also apply it to `.Chars[i]`; the two indexing paths are
  independent. See §7.4.1 for the read-only confirmation.

---

## 7.3 String operations

### 7.3.1 Concatenation, comparison, copy-on-write

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

`+` concatenates; relational operators compare ordinally; assignment shares the
buffer (reference-counted) until mutated (copy-on-write).

**Semantics & parsing notes**

- These are the operator semantics from ch.04 §4.7 applied to managed strings.
- *Copy-on-write* means writing through an index uniquifies the buffer — a runtime
  behaviour, transparent to the parser.
- ⚠️ *`SetLength` on `string` does NOT zero-fill new characters* — the opposite
  of the dynamic-array guarantee (§8.2.1). Growing a string's length can expose
  leftover heap garbage in the newly-added positions. Confirmed (dcc-verified,
  dcc32 37.0): after freeing a 50-char buffer (dropping its last reference,
  `S := ''`) to dirty the allocator, then `S := 'AB'; SetLength(S, 20)`, the
  bytes beyond the original 2 characters were **not** reliably zero — one
  position read back `90` (`'Z'`, a leftover byte from the freed buffer)
  rather than `0`. Code that needs zeroed character storage after `SetLength`
  must zero it explicitly (e.g. `FillChar`) — do not port the array guarantee
  over to strings.
- ⚠️ *A `const string` parameter is NOT reference-counted — it borrows the
  caller's buffer without taking a reference, which can dangle.* Unlike a
  by-value `string` parameter (which increments the buffer's ref-count for the
  duration of the call, so the buffer survives even if the caller reassigns its
  variable), a `const` (and likewise `var` / `const [ref]`) `string` parameter
  is passed as a raw pointer to the caller's existing data with no ref-count
  bump. If the callee calls back into code that reassigns/frees the caller's
  original variable — dropping what was, for a `const` parameter, the buffer's
  *only* counted reference — the buffer can be freed while the `const`
  parameter still refers to it. Reproduced as an outright crash (dcc-verified,
  dcc32 37.0): a `procedure TestConst(const S: string)` captures `PChar(S)`,
  then calls a helper that reassigns the caller's global string variable
  (dropping the last other reference to `S`'s buffer) and churns the allocator
  with a burst of same-size allocations; the very next use of `S` inside
  `TestConst` raises `EInvalidPointer: Invalid pointer operation`. The same
  test with the parameter changed to plain `S: string` (by value) does **not**
  crash — the callee's own copy kept its own reference alive throughout. This
  is a genuine language-level hazard, not merely a book claim: `const string`
  buys performance (no ref-count traffic) at the cost of the callee being
  unable to outlive a caller-side mutation during the call.

---

## 7.4 String (record) helpers

### 7.4.1 `string` intrinsic helper methods

| | |
|---|---|
| **Introduced** | Delphi XE3 (record helper for `string`) |
| **Deprecated** | — |
| **Status** | ✅ Current |

The RTL ships a **record helper** for `string` giving method syntax
(`S.Length`, `S.ToUpper`, `S.Split`, …). This is a *library* helper but it is
enabled by the *language* feature "record helpers for intrinsic types" (ch.15).

**Example**

```pascal
if S.StartsWith('http') then
  Parts := S.Split(['/']);
```

**Semantics & parsing notes**

- Method-call syntax on a string value resolves through the active helper (ch.15
  helper-resolution rules: only **one** helper is in scope at a time — the most
  recently declared/used wins).
- *Parser impact:* `S.Method` is an ordinary `Designator` member access; the
  resolver routes it to the helper.
- ⚠️ *`S.Chars[i]` is a read-only, unconditionally 0-based property* of the
  helper (`TStringHelper.GetChars`, a getter-only property — there is no
  setter). Confirmed both halves (dcc-verified, dcc32 37.0): `S := 'ABC';
  S.Chars[0]` reads `'A'` regardless of whether `{$ZEROBASEDSTRINGS}` is `ON`
  or `OFF` at that point (contrast §7.2.1's directive-sensitive `S[i]`), and
  `S.Chars[0] := 'X'` is rejected with `E2129 Cannot assign to a read-only
  property` — there is no mutation path through `.Chars`.
- ⚠️ *`string.Create(...)` / `string.Copy(...)` pseudo-constructors* exist,
  parallel to the dynamic-array `T.Create(...)` pattern (§8.2.3) — but unlike
  that array form, these are **not** compiler-synthesised magic. They are
  ordinary `class function ... static` members of `TStringHelper`
  (`Create(C: Char; Count: Integer): string`, `Create(const Value: array of
  Char...)`, `Copy(const Str: string): string`), so — like `.Chars` above and
  every other helper method — they only resolve when `System.SysUtils` (where
  `TStringHelper` lives) is in scope; without it, `string.Create(...)` is
  `E2671 Record, object, class type, or type helper required`, the exact
  error a bare `string.Create` gets with no helper active at all. Confirmed
  with `System.SysUtils` in `uses` (dcc-verified, dcc32 37.0):
  `string.Create('X', 5)` yields `'XXXXX'` (5 copies of the character);
  `string.Copy(S1)` yields a value equal to `S1` (`S1 = S2` is `True`) — the
  book's point being it forces a genuine copy rather than an aliasing
  assignment, though since `string` assignment is already copy-on-write, this
  mainly documents intent at the call site rather than changing observable
  behaviour before the first write.
