# git.pp Record Format — SPEC

> Canonical, normative spec for the git.pp record format. The implementation-specific notes (rationale, performance, measured numbers) live in `DESIGN.md` and `docs/CHECKPOINT-V2.md` in `patchwork-quilt/git.pp-ng`. This document is the *contract* a port must satisfy to be conformant.

**Version:** 0.2.0 (format v2)
**Status:** Stable. Existing v1 records are first-class forever.

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
7. Emit as JSON: `json.dumps(canon, sort_keys=True, ensure_ascii=True, separators=(", ", ": "))`.
8. SHA-256 of the UTF-8 bytes of the JSON, lowercase hex.

The leaf is independent of the order of `parents` in the input record (because step 3 sorts them as a set). Reordering parents is not a tamper; rewriting a parent's id, or adding/removing a parent, is.

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

### 5.2 Per-key resolution

For each key K touched by any applied record:

1. Gather all applied records that set K — they are all *candidates*, regardless of class.
2. Filter to the highest class present (override if any override sets K, else task if any task sets K, else witness).
3. **Ancestry reduction** within the class: a candidate is dominated if any *other* candidate is its (transitive) ancestor within the applied set. The survivors are the most-derived.
4. **Tie-break** in order: most intent ancestors (`task` + `override`), fewest witness ancestors, lexicographically smallest id (by Unicode code point).
5. The first survivor is the winner; the winner's value is the derived value.

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

