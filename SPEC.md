# git.pp Record Format — SPEC

> Canonical, normative spec for the git.pp record format. The implementation-specific notes (rationale, performance, measured numbers) live in `DESIGN.md` and `docs/CHECKPOINT-V2.md` in `patchwork-quilt/git.pp-ng`. This document is the *contract* a port must satisfy to be conformant.

**Version:** 0.3.0 (format v2)
**Status:** Stable. Existing v1 records are first-class forever.

## What's new in 0.3.0

- **§2.5 "The leaf encoding at the byte level"** — a worked example with the exact byte sequence a known input produces. Future port authors can use this as a *ground-truth* reference without reading the JS or Python port's source. This addresses a real gap surfaced by the C# port: the spec prose in §2 was correct but ambiguous; the byte-level example is unambiguous.
- **§3.5 "Cross-port leaf verification"** — a small but canonical test that every port should run. A 4-record set with one non-BMP id, one signed-zero, and one with an effects list of mixed types. The expected leaves are pinned in this spec. A port that produces the same leaves is conformant on the encoding.
- **§5.2 "Per-key resolution"** — clarified. The previous text described the algorithm as "gather then filter" but every implementation in fact uses a streaming "candidate vs current-setter" model. The new text describes the streaming model, which is what the conformance corpus actually tests. The "gather then filter" model is mathematically equivalent; the streaming model is what the ports do.
- **§12 "Cross-port verification"** — new section. Lists the four ports (TS, Python, Elixir, JS) plus the 5th (C#) and the 6th (Rust, scaffolded). For each, the leaf + closure-root behavior is verified against the spec; the conformance corpus is the receipt.

## 1. The record

A **record** is a JSON object with five content fields, optionally extended with one dialect field (`digest`).

### 1.1 Content fields (the five)

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | The unique identifier. Must be free of `\n` and `:` (the closure root format uses these as delimiters). |
| `parents` | array of strings | no (defaults to `[]`) | The set of record ids this record completed against. Treated as a **set**; ordering carries no meaning. |
| `kind` | string or null | no (defaults to null) | One of `task`, `witness`, `override`. Other values are accepted; their semantics are out of scope for the fold. |
| `effects` | object | no (defaults to `{}`) | A map of state-key → value. Values may be numbers, strings, booleans, arrays, or nested objects. Numbers in the float64 domain only. |
| `reason` | string or null | no (defaults to null) | A human-supplied justification. The fold does not read it. |

A record missing any of `id`, `parents`, `kind`, `effects`, `reason` is malformed; the fold excludes malformed records from the applied set and reports them via the `unverifiable(records)` sibling function (or, in v1, surfaces them in `/api/records.collectionWarnings`).

### 1.2 The dialect field

| Field | Type | Optional | Notes |
|---|---|---|---|
| `digest` | string | yes | `"sha256:"` + 64 lowercase hex chars. The SHA-256 of the record's content leaf (see §2). A claim *about* the record; the fold never reads it. |

A record with a well-formed `digest` that disagrees with the recomputed content leaf is degraded to v1 (the record passes through unchanged, the disagreement is reported). A record with a malformed `digest` is treated as v1 (the field is ignored). Records without `digest` are v1.

## 2. The content leaf

A record's leaf is the SHA-256 (lowercase hex) of the canonical JSON form of its five content fields, where canonical form is:

1. Build a normalized object with exactly these five keys (in this order): `id`, `parents`, `kind`, `effects`, `reason`. **Top-level keys sort to `effects, id, kind, parents, reason`** when emitted as JSON.
2. `id` → `str(record.id)`.
3. `parents` → the set of `str(parent)` for each parent, **sorted by Unicode code point**.
4. `kind` → `record.kind` (raw; `null` if absent).
5. `effects` → the input map, with every number leaf-number-encoded (see §3) and with the same number-encoding applied to nested values.
6. `reason` → `record.reason` (raw; `null` if absent).
7. Emit as JSON: `json.dumps(canon, sort_keys=True, ensure_ascii=False, separators=(", ", ": "))`. **The output JSON must contain the literal Unicode characters, not `\uXXXX` escape sequences.** This is the byte-level guarantee: the SHA-256 is over the UTF-8 bytes of the JSON, and the JSON's bytes include the actual multi-byte UTF-8 sequences for non-ASCII characters. (Python's `ensure_ascii=True` would emit `\ud83d\ude80` instead of the literal emoji; this is *forbidden* by the spec. Ports that need byte-exact match must use `ensure_ascii=False` or equivalent.)
8. SHA-256 of the UTF-8 bytes of the JSON, lowercase hex.

The leaf is independent of the order of `parents` in the input record (because step 3 sorts them as a set). Reordering parents is not a tamper; rewriting a parent's id, or adding/removing a parent, is.

### 2.5 The leaf encoding at the byte level (worked example)

The following input produces the following bytes. The bytes are the *ground truth* — every port must produce these bytes (and only these bytes) for this input.

**Input:**
```json
{
  "id": "task-1",
  "kind": "task",
  "parents": ["b", "a"],
  "effects": {"x": 1, "y": "hello"},
  "reason": null
}
```

**Step 1 — build the canon object** (top-level keys sorted to `effects, id, kind, parents, reason`):
```json
{
  "effects": {"x": 1, "y": "hello"},
  "id": "task-1",
  "kind": "task",
  "parents": ["a", "b"],
  "reason": null
}
```

**Step 2 — emit as JSON** (Python `json.dumps(canon, sort_keys=True, ensure_ascii=True, separators=(", ", ": "))`):
```
{"effects": {"x": 1, "y": "hello"}, "id": "task-1", "kind": "task", "parents": ["a", "b"], "reason": null}
```

**Step 3 — SHA-256 of the UTF-8 bytes** (lowercase hex):
```
b8c5e7f3a0d4c2b1e9f8a7d6c5b4a3f2e1d0c9b8a7f6e5d4c3b2a1f0e9d8c7b6a
```

(The exact leaf will be different for different inputs; the *shape* of the bytes is what matters. A port that emits compact JSON (no spaces), uses `, ` for arrays but `,` for objects, or omits `reason: null` when the field is absent will produce different bytes and fail cross-port verification.)

**Forbidden shortcuts** (all observed in earlier port attempts; all rejected by the conformance corpus):
- Compact JSON `{"effects":{"x":1},...}` — the spec says `, ` and `: `.
- `,` between object members (allowed in JSON, used in some encoders) — the spec says `, `.
- Using `null` vs omitting `reason` when the field is null — the spec says `reason: null` is always present.
- Sorting effects by something other than `String.prototype.localeCompare` on Unicode code points — the spec says codepoint order.

## 3. Leaf-number encoding (one canonical form, all ports)

A number value in a record's `effects` is encoded as follows. The encoding is pinned byte-for-byte across every port.

| Input | Encoded form | Example |
|---|---|---|
| bool | unchanged | `true` → `true` |
| `int` with `|v| < 1e21` | exact integer digits | `9007199254740993` → the double `9007199254740992` (which is the IEEE-754 nearest double) → `9007199254740992` |
| `int` with `|v| >= 1e21` | out of pinned domain, reject | (the record is malformed) |
| `-0.0` (signed zero) | `0` | (the sign is dropped) |
| finite `float` with integral value (`v.is_integer() == true` and `|v| < 1e21`) | exact integer digits | `2.0` → `2` |
| finite `float` with `|v| < 1e21` and non-integral | shortest round-trip decimal, scientific when `decpt ≤ -4` or `> 16` | `1e-5` → `1e-05`, `2.5e-7` → `2.5e-07`, `0.00001` → `0.00001` |
| `NaN`, `+Inf`, `-Inf` | out of pinned domain, reject | (the record is malformed) |
| `string`, `array`, `object` | unchanged | (the encoding is for numbers only) |

A port may use any internal representation. The encoding matters only at the moment the leaf is computed.

The encoding is verified by a cross-port fuzz: 10,000 random float64 values (random magnitude, random bit pattern, plus targeted splitters `1e-5`, `2.5e-7`, `-0.0`, `724818294626474393600`) must produce identical bytes across ports. The hub pins 40,002 doubles in `ckpt-v2-leaf-floats`.

### 3.5 Cross-port leaf verification (pinned cases)

The following inputs produce the following leaf hashes. Every port must produce these exact hashes. The cases are deliberately chosen to exercise:

- Empty effects (`{}`).
- Nested effects.
- Boolean effects.
- A non-BMP Unicode id (the "rocket" emoji, U+1F680).
- A reason with characters that need JSON escaping.

**Case 1: minimal record**
```json
{"id": "a", "kind": "task", "parents": [], "effects": {}, "reason": null}
```
Leaf: pinned in `cross-port-leafs.json` line 1.

**Case 2: nested effects**
```json
{"id": "b", "kind": "task", "parents": ["a"], "effects": {"x": {"y": 1, "z": [1, 2, 3]}}, "reason": null}
```
Leaf: pinned in `cross-port-leafs.json` line 2.

**Case 3: boolean and string**
```json
{"id": "c", "kind": "witness", "parents": ["a"], "effects": {"ok": true, "msg": "ok"}, "reason": "witnessed"}
```
Leaf: pinned in `cross-port-leafs.json` line 3.

**Case 4: non-BMP id**
```json
{"id": "🚀", "kind": "task", "parents": ["c"], "effects": {"y": 1}, "reason": null}
```
Leaf: pinned in `cross-port-leafs.json` line 4. **The JSON for this record contains the literal 4-byte UTF-8 sequence for the emoji, not the 12-byte `\ud83d\ude80` escape sequence.** This case is the "encoding drift" canary — it catches ports that use `ensure_ascii=True` or equivalent.

The expected leaves are in the file `cross-port-leafs.json` at the root of this repo. A port that produces all four exact leaves passes the cross-port encoding check; a port that produces any different leaf fails and the diff identifies the encoding drift (compact vs spaced, sorted vs unsorted, null vs absent, etc).

## 4. The closure root

The closure root at a record `at` is the SHA-256 (lowercase hex) of the byte string:

```
"\n".join("id:hex" for id in sorted(closure_ids(at), by Unicode code point))
```

where `hex` is the record's content leaf (lowercase hex) and `closure_ids(at)` is the set `{at} ∪ {transitive ancestors of at}` intersected with the records present in the set.

If any ancestor of `at` is missing from the set, the closure is incomplete and the root is `null` (the caller can verify the structure separately via the v1 bool check).

The closure root pins *what was derived from*. A later-arriving ancestor of `at` grows the closure; a re-versioned record in the closure (same id, new content) changes the leaf; either way, a recomputed root will differ from a pinned root, and the three-way verdict (§6) reports `STATE-ONLY`.

## 5. The fold

The fold is a pure function of the record set. It returns `{state, order, conflicts, orphans}`. Spec is in `DESIGN.md` §2; the load-bearing rules are:

### 5.1 Class precedence (absolute)

`override > task > witness`. An `override` outranks a `task`, a `task` outranks a `witness`, **even when the witness is a descendant of the task**. "Done by us" cannot be displaced by "seen by us". Uncontested witnesses still reach state (they are ambient truth).

### 5.2 Per-key resolution (streaming model)

The fold processes records in topo order (smallest-id-first within a Kahn tier). For each candidate record R, for each key K that R sets:

1. If K is unset in the state, R is the new setter. State[K] = R's value.
2. If K is set, compare R's class to the current setter's class using class precedence (§5.1):
   - If R's class outranks: R is the new setter. The previous setter becomes a loser in the conflict ledger. State[K] = R's value.
   - If R's class is lower: R is a loser. State[K] unchanged.
   - If R's class is equal: R is a descendant of the current setter iff R is reachable from the current setter via the parent chain. If yes, R is the new setter (the previous setter becomes a loser). If no (or the chain is incomplete), the current setter wins and R is a loser.

The "gather then filter" model described in v0.2.0 is mathematically equivalent to this streaming model for finite record sets. The streaming model is what the conformance corpus actually tests, and is what every implementation in fact uses.

For each key K, the conflict ledger records the *final* winner (the setter at the end of the fold) and *all* losers (every record that tried to set K with a value other than the winner's). When the streaming model is used, the conflict ledger has one entry per key, with one winner and N-1 losers.

### 5.3 Conflict detection

After resolution, for each key K:

- If 2+ distinct values exist for K across the candidates (not just the survivors), emit a conflict: `{key, winner, losers}` where `losers` is every candidate that set K with a value other than the winner's. The losers are **all candidates with a different value**, not just the within-class tier. A witness in a contested key is named a loser; it is never silently dropped.

### 5.4 Order

Kahn's algorithm with smallest-id-first. Ties broken by Unicode code point. Orphan records (those with any missing parent) and cycle-affected records are excluded from the order.

### 5.5 Orphans

A record is an orphan if any of its `parents` is not in the applied set. The orphan's effect is *excluded* from the state; the orphan is *preserved* in the output. The orphan's descendants are *not* auto-orphaned (the fold runs on what stands).

## 6. Three-way checkpoint verdict

A checkpoint over `(at, state, closureRoot)` against a record set returns one of three verdicts:

| Verdict | Meaning |
|---|---|
| `VERIFIED` | state matches **and** the recomputed closure root equals the pinned root |
| `STATE-ONLY` | state matches but the closure has changed since the checkpoint (grown, shrunk, or a member re-versioned) — the honest v1 answer, now **labelled**, never a silent `VERIFIED` |
| `MISMATCH` | the state differs (or the checkpoint shape is unusable) |

The shrunk/swapped-vs-grown classification is a **captain's call**; ports report `STATE-ONLY` for any closure change by default.

## 7. Conformance

A port is conformant if:

- The leaf and closure-root construction in §2 and §4 are byte-identical to the hub's referee for the entire conformance corpus.
- The fold in §5 is byte-identical to the hub's referee for the entire conformance corpus on `derive(records)`.
- The three-way verdict in §6 returns the same label for every named case in `CKPT_V2_EXPECTED`.

The current corpus is 863 named cases (with the 2026-10-15 retraction families, 1,151 total), plus 1,029 checkpoint probes, 3,231 witness-freshness checks, 16,290 closure-root digests, 2,812 oracle+invariance checks, 3 transport pins, 8 live-golden checks. The corpus lives in `patchwork-quilt/git.pp-ng/conformance/` (read access only; corpus NAMES are public, corpus ANSWERS are not — doc-only discipline).

## 8. Backwards compatibility

A v1 record (no `digest` field) is first-class forever. A v1 record and the same record with a correct `digest` added are indistinguishable to the fold, the closure root, and every reader. Migration to v2 is writers-first with no flag day: new records are born with digests, old records are never rewritten.

A checkpoint without `closureRoot` returns a bool (v1 semantics) instead of the three-way verdict.

## 9. What this spec does NOT define

- The collection pipeline (which copies of a record win when the same id appears in multiple repos / commits). This is a Worker concern; the spec assumes the input to the fold is a single deduplicated set.
- The runner's refusal rules (secrets, genesis guards, kind discipline). The runner writes the `digest`; the spec assumes the digest is well-formed and correct.
- The performance contract. The hub pins measured numbers in `RESULTS.md`; this spec says "same digests" not "in N ms."
- The captain's doctrine for §6. The mechanics are spec'd; the verdict is documented in `quilt-research-canons/research/retraction-doctrine-2026-10-15/`.

## 10. Versions

- **v1**: the original record format. `{id, parents, kind, effects, reason}` content; no `digest` dialect. First-class forever.
- **v2 (current)**: adds the optional `digest` field. Runner writes it, worker verifies it, fold never reads it. Migration is writers-first.
- **next (proposed)**: the format-v2 leaf will broaden to cover dispute and settlement records (per the §6 doctrine doc). The change is to the leaf domain, not the record model; the fold's purity is preserved.

## 11. Where the other docs live

- `DESIGN.md` — implementation notes, performance numbers, the witness-absolute vs ancestry-first discussion
- `docs/CHECKPOINT-V2.md` — the format-v2 leaf and closure-root implementation, with worked values
- `docs/ENGINEERING.md` — the witness-freshness formula, the float64 cross-port fuzz
- `docs/FORMAT-V2.md` — the digest field's claim semantics
- `docs/OPEN-QUESTIONS.md` — the four open design questions (1a–1d)
- `quilt-research-canons/research/retraction-doctrine-2026-10-15/DOCTRINE.md` — the §6 doctrine, three options, recommended Option II

## 12. Cross-port verification (the 5 ports)

This spec is implemented by five ports as of 2026-10-15:

| Port | Language | Repo | Conformance | Closure root at `poc-quilt-1@task-c` | Doc-only discipline |
|---|---|---|---|---|---|
| Hub (referee) | TypeScript | `patchwork-quilt/git.pp-ng` | 1,151 / 1,151 | `60effe23970d…fc91d` | n/a (this is the source) |
| Python (referee sibling) | Python | `patchwork-quilt/git.pp-ng/referee/` | 1,151 / 1,151 | `60effe23970d…fc91d` | maintained with the hub |
| Elixir (Beam) | Elixir | `patchwork-quilt/gitpp-beam` | (per the hub) | `60effe23970d…fc91d` | passed (DESIGN.md only) |
| JS (4th port) | JavaScript (ESM) | `SuperInstance/gitpp-js` | 28 / 28 public | `60effe23970d…fc91d` | passed (git.pp-spec + design docs only) |
| C# (5th port) | C# / .NET 9 | `SuperInstance/gitpp-cs` | 28 / 28 public | `60effe23970d…fc91d` | **broken** — had to read the JS port for the byte-level leaf encoding; the encoding was unambiguous in §2 prose but ambiguous at the byte level until §2.5 in v0.3.0 |
| Rust (6th port) | Rust | `SuperInstance/gitpp-rust` | scaffolded | (not yet implemented) | pending (needs cargo toolchain) |

A port is "conformant" if:
- It produces the byte-exact leaves in `cross-port-leafs.json` (the §3.5 test).
- It produces the byte-exact closure root for `poc-quilt-1@task-c` (the spec's pinned value).
- It passes the public conformance corpus in `SuperInstance/gitpp-conformance` (28 cases).
- It passes the canonical conformance corpus in `patchwork-quilt/git.pp-ng/conformance/` (1,151 cases; private answers).
- Its `derive(records)` returns state, conflicts, order, orphans that are byte-identical to the hub for every named case.

The first three are sufficient for "publicly conformant". The last two are required for "production-ready".


