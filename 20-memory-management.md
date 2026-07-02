# 20 — Memory Management

The language's memory model: storage regions, manual object lifetime, the set of
**managed types** with automatic lifetime, reference counting, weak/unsafe
references, and the historical mobile ARC era.

Mostly *semantics* (codegen), little new syntax — but a parser's consumers (a
type-checker / lifetime analyzer) need this model.

---

## 20.1 Memory regions

### 20.1.1 Global, stack, and heap

| | |
|---|---|
| **Introduced** | Pascal (pre-1995) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Globals (unit `var`) live for program duration; locals and value parameters live
on the **stack**; objects, dynamic arrays, and `New`-allocated data live on the
**heap**.

**Semantics & parsing notes**

- Determines lifetime/finalization codegen, not syntax. The analyzer classifies
  each declaration's storage from its scope (ch.03 §3.3.1).

---

## 20.2 Manual object lifetime

### 20.2.1 `Create` / `Free` ownership

| | |
|---|---|
| **Introduced** | Delphi 1; unified manual model **10.4 Sydney** (ARC removed) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Class instances are **manually managed**: whoever creates an object is
responsible for freeing it (directly via `Free` or through an owner like
`TComponent`).

**Example**

```pascal
L := TStringList.Create;
try
  L.Add('x');
finally
  L.Free;
end;
```

**Semantics & parsing notes**

- ⚠️ *Current model (all platforms, 10.4+):* manual. The `try…finally`/`Free` idiom
  (ch.18) is the canonical pattern. Ownership (e.g. `TComponent.Owner`,
  `OwnsObjects`) is an RTL convention, not a language rule.
- A lifetime analyzer built on this AST should flag created-but-not-freed objects
  (heuristic), but the language does not enforce it.

---

## 20.3 Managed types (automatic lifetime)

### 20.3.1 The managed-type set

| | |
|---|---|
| **Introduced** | strings/dyn-arrays/interfaces D2–D4; **custom managed records 10.4** |
| **Deprecated** | — |
| **Status** | ✅ Current |

Some types have **compiler-managed** lifetime — the compiler inserts
initialization, finalization, and copy logic automatically: long **strings**,
**dynamic arrays**, **interfaces**, **`Variant`**, **anonymous-method references**,
and **custom managed records** (ch.09 §9.4).

**Semantics & parsing notes**

- ⚠️ *"Is this type managed?"* is a core type-system query: managed fields force the
  enclosing record/array to also get finalize codegen; managed types are
  zero-/nil-initialized on entry and finalized on scope exit. The type-checker must
  compute managedness transitively.
- `Default(T)`, `Initialize`, `Finalize` intrinsics interact with this.

---

## 20.4 Reference counting

### 20.4.1 Interface & string/array reference counting

| | |
|---|---|
| **Introduced** | strings D2; interfaces D3 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Strings and dynamic arrays use copy-on-write reference counting; **interface**
references call `_AddRef`/`_Release`, destroying the object at count zero (when
implemented via `TInterfacedObject`).

**Semantics & parsing notes**

- ⚠️ *Object-vs-interface aliasing hazard* (ch.14 §14.3.1): holding both an object
  reference and interface references to the same instance can free it prematurely.
- The compiler inserts `_Release` calls at scope exit for interface locals — the
  analyzer should model this for correctness checks.

---

## 20.5 Historical: mobile ARC

### 20.5.1 Automatic Reference Counting (XE4 – 10.3)

| | |
|---|---|
| **Introduced** | Delphi XE4 (iOS) |
| **Deprecated** | 10.3 Rio (deprecated) / **removed 10.4 Sydney** |
| **Status** | ❌ Removed (🧪 historical) |

On the mobile/NextGen compilers, **objects** were reference-counted (ARC) like
interfaces: `Free` was largely unnecessary, fields auto-nilled, `[weak]`/`[unsafe]`
broke cycles.

**Semantics & parsing notes**

- ⚠️ *Removed in 10.4* — all platforms returned to the manual model. A parser
  targeting current Delphi should treat object ARC as **historical**. Code written
  for ARC (relying on auto-free) may leak under the unified model.
- `[weak]`/`[unsafe]` (20.6) survived and apply to interface references.

---

## 20.6 Weak & unsafe references

### 20.6.1 `[weak]` and `[unsafe]` attributes

| | |
|---|---|
| **Introduced** | Delphi XE4 (mobile ARC); **all compilers 10.1 Berlin**; retained after ARC removal (10.4) |
| **Deprecated** | — |
| **Status** | ✅ Current |

Field attributes that opt a reference out of strong reference counting, to break
cycles or avoid keeping a target alive.

**Example**

```pascal
type
  TNode = class(TInterfacedObject, INode)
    [weak]   FParent: INode;    // zeroed when parent dies; thread-safe
    [unsafe] FOwner:  IOwner;   // raw, non-counted (dangling risk)
  end;
```

**Semantics & parsing notes**

- These are **attributes** (ch.19 syntax), parsed as `[weak]`/`[unsafe]` before the
  field. The semantic layer changes ref-count codegen: `[weak]` registers for
  zeroing, `[unsafe]` does nothing on assign/release.
- *AST:* attribute on the `FieldDecl`; analyzer flags strong reference cycles that
  these resolve.

---

## 20.7 Raw heap intrinsics

### 20.7.1 `New`/`Dispose`/`GetMem`/`FreeMem`

| | |
|---|---|
| **Introduced** | Pascal/D1 |
| **Deprecated** | — |
| **Status** | ✅ Current |

Low-level allocation for pointers — recap of [ch.10 §10.1.3](10-pointers-files.md#1013-manual-allocation-intrinsics).

**Semantics & parsing notes**

- ⚠️ `New`/`Dispose` are managed-aware (init/finalize the pointed-to type's managed
  fields); `GetMem`/`FreeMem` are **raw** (no init/finalize) — mixing them with
  managed types leaks/corrupts. The analyzer should pair allocs with frees and flag
  raw allocation of managed-field types.
- These are by-reference intrinsics (ch.04 §4.11); first argument must be an lvalue
  pointer.
