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
