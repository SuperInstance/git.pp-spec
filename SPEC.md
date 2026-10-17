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
eb70b332a71ea6c8aab9a3525ef0906f1875f64778b8f2d2f2ebcf27081c6b03
```

(The exact leaf will be different for different inputs; the *shape* of the bytes is what matters. A port that emits compact JSON (no spaces), uses `, ` for arrays but `,` for objects, or omits `reason: null` when the field is absent will produce different bytes and fail cross-port verification.)

**Forbidden shortcuts** (all observed in earlier port attempts; all rejected by the conformance corpus):
- Compact JSON `{"effects":{"x":1},...}` — the spec says `, ` and `: `.
- `,` between object members (allowed in JSON, used in some encoders) — the spec says `, `.
- Using `null` vs omitting `reason` when the field is null — the spec says `reason: null` is always present.
- Sorting effects by something other than `String.prototype.localeCompare` on Unicode code points — the spec says codepoint order.

## 3. Leaf-number encoding (one canonical form, all ports)

A number value in a record's `effects` is encoded as follows. The encoding is pinned byte-for-byte across every port.

**Scope note.** §3 pins the encoding of the **leaf** only. The serialization of the *state* returned by `derive()` is deliberately **not** pinned: ports may serialize the same float64 differently (JavaScript's `JSON.stringify(1e-5)` gives `0.00001`, Python's `repr(1e-5)` gives `1e-05`) without any of them being wrong. Two consequences:

- Cross-port **fold** conformance must compare state *by value*, not by bytes. See [`gitpp-fold-conformance`](https://github.com/SuperInstance/gitpp-fold-conformance).
- A consumer that needs a byte-stable state must re-encode each value through the §3 rules before serializing. The leaf is the only byte-stable commitment the spec makes.

| Input | Encoded form | Example |
|---|---|---|
| bool | unchanged | `true` → `true` |
| finite `float` with `|v| < 1e16` and `v.is_integer() == true` | exact integer digits | `2.0` → `2`, `9007199254740992.0` → `9007199254740992` |
| finite `float` with integral value but `|v| >= 1e16` (does not fit safely in int64) | shortest round-trip decimal, scientific | `7.2e20` → `7.2e+20` |
| finite `float` with `|v| < 1e16` and non-integral | shortest round-trip decimal, scientific when `decpt ≤ -4` or `> 16` | `1e-5` → `1e-05`, `2.5e-7` → `2.5e-07`, `0.00001` → `0.00001` |
| `int` (record supplied as JSON integer literal, `|v| < 1e16`) | exact integer digits | `1` → `1` |
| `int` (record supplied as JSON integer literal, `|v| >= 1e16`) | **treated as the float64 it becomes** — the domain rule, not the spelling rule | `724818294626474393600` → `7.248182946264744e+20` |
| finite `float` with `|v| >= 1e21` | out of pinned domain, reject | (the record is malformed) |
| `-0.0` (signed zero) | `0` | (the sign is dropped) |
| `NaN`, `+Inf`, `-Inf` | out of pinned domain, reject | (the record is malformed) |
| `NaN`, `+Inf`, `-Inf` | out of pinned domain, reject | (the record is malformed) |
| `string`, `array`, `object` | unchanged | `1.5e20` (number) and `"1.5e20"` (string) are **different** inputs and must produce different leaves |

**Every JSON number is a float64.** A port MUST NOT let its host decoder choose a different numeric type: Python's `json` returns an arbitrary-precision `int` for a bare integer literal, `JSON::PP` returns a `string` for one too large for a UV/IV (and an exact `int64` for one that fits, which no flag can distinguish from a string), and `System.Text.Json` returns an `int64`. All three shipped this. `{"x": 1000000000000000000}` must be `1e+18` (scientific, since `|v| >= 1e16`), not `1000000000000000000`. **The fix always belongs at the decode boundary, never in the encoder.** Python: `json.loads(..., parse_int=float)`. C#: `Numbers.ParseRecords`. Perl cannot recover it for literals in [2^63, 1e21) and documents the limitation.

A port MUST also distinguish a JSON number from a JSON **string** that happens to contain a valid number literal. This is not a theoretical concern: several standard-library JSON decoders return an indistinguishable scalar for both (`JSON::PP` in Perl is one), so a naive `looks_like_number()` test silently turns strings into numbers. The cross-port corpus now carries 3,000 records that pair a real number with a numeric-string twin for exactly this reason.

A port may use any internal representation. The encoding matters only at the moment the leaf is computed.

> **BEAM note (measured on OTP 25, `erlang-base` 1:25.2.3).** No built-in BEAM float format matches this section, so an Elixir/Erlang port must hand-roll the encoder:
>
> | value | `float_to_binary(V, [short])` | §3 requires |
> |---|---|---|
> | `1.0e-5` | `1.0e-5` | `1e-05` |
> | `2.5e-7` | `2.5e-7` | `2.5e-07` |
> | `0.0` | `0.0` | `0` |
> | `1.5e20` | `1.5e20` | scientific, 15/16/17 |
> | `9007199254740992.0` | `9.007199254740992e15` | `9007199254740992` |
> | `1.0e-4` | `0.0001` | `0.0001` (the only match) |
>
> Three BEAM-specific traps, all found by writing the port rather than by reading the manual:
>
> 1. **Integers are arbitrary-precision by default.** `1000000000000000000` is an integer, not a float, so the float64-domain rule above fails *by construction* rather than by accident. Every bare integer literal in a record is a bignum. This is the same defect class as `json` and `System.Text.Json`, but structural.
> 2. **`=:=` is type-exact.** `trunc(F) =:= F` is **always false** for a float, so a perfectly natural-looking integral check silently sends every integral float down the wrong branch. Use `float(trunc(F)) =:= F`.
> 3. **UTF-8 binary comparison *is* code-point order** (`lists:sort/1` over binaries gives `["0","a","z","~",🚀]`). §5.4 ordering comes free here — it is the one place BEAM is easier than JavaScript, Perl, or Python, all of which need explicit code-point iteration.

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

**Class is compared FIRST. Ancestry is a tie-break within a class, never a competing ranking.** Two records of different class are ordered by class alone, regardless of how they are related: an `override` that is an *ancestor* of a `task` still outranks it, and a `witness` that is a *descendant* of a `task` still loses to it. Ancestry is consulted only when the two classes are *equal* (§5.2 step 3), where it selects the more derived of two same-class records.

> **This scope used to be implied rather than stated**, and the gap was load-bearing. The parenthetical above is this clause's only mention of ancestry, and it sits as an exception — so it reads as "class beats ancestry", which inverts every contested key. A cross-reading experiment (`SuperInstance/gitpp-reading-room`) had an implementation adopt exactly that reading; an independent adjudicator scored it 0.030 against 0.960 for this one. The clause is now stated, not inferred.

**Unclassifiable records rank below `witness`.** A record whose `kind` is `null` or is not one of the three names has no class, and the fold must still rank it (§5.2). Such a record ranks **lowest**: it can win an uncontested key, and it loses every contest against any classifiable record, whatever their relationship in the ancestry DAG. An unclassifiable record never displaces a classifiable one.

An unrecognised `kind` **string** is not malformed. §1.1's malformed rule covers a record *missing* a content field; a record carrying an unrecognised value for `kind` is accepted and ranked lowest by this rule. Only a missing field makes a record malformed.

**Unclassifiable records are ranked `lowest` as a single class.** They are equal to each other, so a contest between two of them is decided by the equal-class rule in §5.2 step 3 — ancestry first, then topo order — exactly as for any other same-class pair. A contest between an unclassifiable record and a classifiable one is decided by class alone, and the classifiable record always wins.

> This closes a gap the v0.4.6 fix itself introduced. The first draft of this clause said an unclassifiable record "never displaces a classifiable one", which settles a mixed contest but is silent when *neither* record is classifiable, and silent on whether two unclassifiable records count as "equal" for the §5.2 step 3 tie-break. An independent adjudicator scored the text as not settling the all-unclassifiable contest at 0.700 against 0.340, and the all-unclassifiable ancestry case at 0.790 against 0.860 — a near-tie, which is the signature of a clause that a careful reader can talk themselves either way out of. Both are now stated. Self-auditing a fix is not optional: the gap was found by probing the *new* text, not the old.

### 5.2 Per-key resolution (streaming model)

The fold processes records in topo order (smallest-id-first, sequential — see §5.4). For each candidate record R, for each key K that R sets:

1. If K is unset in the state, R is the new setter. State[K] = R's value.
2. If K is set, compare R's class to the current setter's class using class precedence (§5.1):
   - If R's class outranks: R is the new setter. The previous setter becomes a loser in the conflict ledger. State[K] = R's value.
   - If R's class is lower: R is a loser. State[K] unchanged.
   - If R's class is equal: R is a descendant of the current setter iff R is reachable from the current setter via the parent chain. If yes, R is the new setter (the previous setter becomes a loser). If no (or the chain is incomplete), the current setter wins and R is a loser.

The "gather then filter" model described in v0.2.0 — gather every candidate for a key, drop any that is an ancestor of another, then take the highest-class survivor — is equivalent to this streaming model **whenever the survivors' classes differ**. It is not equivalent unconditionally: when two same-class records are unrelated in the ancestry DAG, neither is dropped, and "take the highest class among survivors" does not select between them, while the streaming model selects the first in topo order (§5.4).

The tie-break in that case is §5.4's: **among equal-class survivors that are mutually incomparable in ancestry, the first in topo order wins.** A port implementing the v0.2.0 model must apply it to match.

> **Corrected.** This section previously asserted unconditional equivalence. A cross-reading experiment (`SuperInstance/gitpp-reading-room`) searched the space and found the gap: `equivalence.py` in that repo checks four hand-stated cases, and the two-root same-class case is the one where the gather model is silent. An implementation following the old text silently disagreed with every streaming port on same-class sibling keys, having been told the two models were the same thing.

> **Watch the direction of the ancestry test.** Within the winning class, a candidate is *dominated* if **it is an ancestor of another candidate**. Survivors are therefore the **most-derived** records. Asking the inverse question — *"is another candidate my ancestor?"* — selects the *most-ancestral* record instead, which silently reverses every contested key in a chain. Four ports shipped this inversion at some point; the corpus case `d03` exists solely to catch it.

For each key K, the conflict ledger records the *final* winner (the setter at the end of the fold) and *all* losers (every record that tried to set K with a value other than the winner's). When the streaming model is used, the conflict ledger has one entry per key, with one winner and N-1 losers.

**`losers` is listed in the §5.4 order sequence** — the same order `order` uses, filtered to the losers. The text above fixes the *membership* of that set; without a stated order, three ports emitted three different serialisations of the same fold, and a consumer that hashes or diffs a ledger gets different bytes for identical semantics. The ledger is a receipt, so its serialisation is part of the contract.

> Added after a generated corpus produced the minimal witness. With records `m(override, x=2)`, `z(witness, x=3)`, `a(override, x=1, parents:[z])`, `order` is `["m","z","a"]` and the winner is `m`, so the losers are `z` and `a` — and the order sequence gives `["z","a"]` while sorted gives `["a","z"]`. Both are defensible readings of "N-1 losers", which is set language. Small cases cannot separate the two readings, which is why 29 hand-written cases never saw it: a case with one loser has only one possible ordering.

### 5.3 Conflict detection

After resolution, for each key K:

- If 2+ distinct values exist for K across the candidates (not just the survivors), emit a conflict: `{key, winner, losers}` where `losers` is every candidate that set K with a value other than the winner's. The losers are **all candidates with a different value**, not just the within-class tier. A witness in a contested key is named a loser; it is never silently dropped.

### 5.4 Order

Kahn's algorithm with smallest-id-first. Ties broken by Unicode code point. Orphan records (those with any missing parent) and cycle-affected records are excluded from the order.

**"Smallest-id-first" means: at each step, among the records whose remaining parents have all been emitted, emit the one with the smallest id. Readiness is re-evaluated after every emission.** This is the sequential reading, and it is what a heap-based Kahn does.

It is *not* the tier reading, in which an entire tier is released and ordered as a block before the next tier begins. The two differ exactly when emitting one record makes another ready whose id sorts before an already-ready record:

```
m (no parents)   z (no parents)   mm (parent m)

sequential:  m, then mm is ready and mm < z  ->  m, mm, z
tier:        the tier {m, z} is sealed       ->  m, z, mm
```

**`order` is a first-class output of `derive()`**, so this is not a presentation detail: it changes which record a consumer sees as processed first. A generated corpus of 165 record sets found 28 where the two readings differ in `order` while `state`, `conflicts` and `orphans` are identical — 17% of the space.

> Added after a generated corpus separated three different disagreements that had been reported as one. An independent adjudicator scored the sequential reading 0.890 against 0.470 for the tier reading, and confirmed that a heap-based Kahn *is* smallest-id-first (0.580 against 0.260 for "is not"). The clause is now stated rather than inferred.

### 5.5 Orphans

A record is an orphan if any of its `parents` is not in the applied set. The orphan's effect is *excluded* from the state; the orphan is *preserved* in the output. The orphan's descendants are *not* auto-orphaned (the fold runs on what stands). **A non-orphan descendant's own effects therefore DO enter the fold.** "Not auto-orphaned" means exactly that the record is an ordinary candidate: it is processed in topo order, it can set keys, and it can become the setter (§5.6). Only the record whose *own* parent id is missing is excluded — excluding a whole branch would contradict the sentence above and is not permitted.

**Cycle-affected records are orphans for output purposes.** They are excluded from the fold exactly like missing-parent records, and they MUST appear in the returned `orphans` list. A port that detects cycles to exclude them from the fold but forgets to report them has made the records vanish from the result entirely — see corpus cases `d13`, `d14`, `d23`.

### 5.6 Setter provenance (nudging is a read-then-write loop)

`derive()` returns `{state, order, conflicts, orphans}`. `state` is a flat
`{key: value}` map, so on its own it does **not** say which record set a key.
That information survives only where two records happened to contest the same
key, because `conflicts[].winner` records the setter there.

This makes a same-class nudge unreliable. §5.2 breaks a class tie by ancestry
first and id second, so a nudge parented on the *wrong* record neither
reliably wins nor reliably loses:

- nudge `a-nudge` (sorts before the winner) lands
- nudge `z-nudge` (sorts after)  is silently discarded
- same record, same kind, same reason, opposite outcome

The outcome depends on nothing but the nudge's own id. An agent cannot
author a reliable nudge without knowing the current setter, and the fold
withholds the setter for exactly the keys that are most stable.

**A port MUST therefore also return `setters`: `{key: record id}` for every
key present in `state`, contested or not.**

- The fold already computes this; it currently discards it.
- `conflicts` becomes a *diff* — what changed, who lost — rather than the only
  provenance in the result.
- This does **not** change any leaf or closure root, so it is additive to §7
  conformance and cannot invalidate a pinned digest.
- A nudge authored as `{id, kind, parents: [setters[key]], effects: {key: new}}`
  now lands deterministically, whatever the nudge's own id sorts as.

### 5.7 Gated application (optional)

§5.6 makes a nudge reliably expressible. It does not make it *right*. A
substrate that a development agent drives needs a place to ask "should this
apply at all?" before the record is written, and a defined answer when the
question cannot be answered.

This is deliberately **not** part of the fold. The fold is pure; a gate that
consults anything outside the record set is not the fold. A gate sits in
front of it, and its routing lives in code, not in whatever model answered.

Three actions — `act`, `review`, `block` — and one rule that outranks the
rest: **absent signal is not consent.** A gate that cannot answer blocks. A
gate that treats an unreachable model as approval will authorise everything.

[`gitpp-gate`](https://github.com/SuperInstance/gitpp-gate) is one
implementation, using TypeSafe's Jev for calibrated structured decisions.
Its measurements are the reason the intent gate is not permitted to block:
on a thin synthetic state, a well-founded nudge scored 0.35 and an
unjustified one 0.07 — real separation at the wrong scale, so a cutoff that
admits the bad case also blocks the good one. The parent gate, which does
separate (0.61 vs 0.15), carries the blocking.

## 6. Three-way checkpoint verdict

A checkpoint over `(at, state, closureRoot)` against a record set returns one of three verdicts:

| Verdict | Meaning |
|---|---|
| `VERIFIED` | state matches **and** the recomputed closure root equals the pinned root |
| `STATE-ONLY` | state matches but the closure has changed since the checkpoint (grown, shrunk, or a member re-versioned) — the honest v1 answer, now **labelled**, never a silent `VERIFIED` |
| `MISMATCH` | the state differs (or the checkpoint shape is unusable) |

**Order matters**: a port must check **state first**, then root. The reverse order
(was the case in gitpp-js v0.1.2 and earlier) can return `VERIFIED` when a
user submits the correct closure root but the wrong state, hiding real drift.
v0.1.3 fixes this by checking state first; the bug was found via the
hub-api dashboard.

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

This spec is implemented by seven ports as of 2026-10-15:

| # | Port | Language | Repo | Conformance | Leaf fuzz | Fold | Doc-only |
|---|---|---|---|---|---|---|---|
| 1 | Hub | TypeScript | `patchwork-quilt/git.pp-ng` | 1,151 / 1,151 | 20,043 / 20,043 | (not measured) | n/a |
| 2 | Python | Python | `gitpp-doctor` v0.2.6 | 1,151 / 1,151 | 20,043 / 20,043 | 29 / 29 | maintained with hub |
| 3 | Elixir | Elixir | `patchwork-quilt/gitpp-beam` | (per hub) | **diverges, static** | (not measured) | passed (DESIGN.md) |
| 4 | JS | JavaScript (ESM) | `gitpp-js` v0.1.6 | 28 / 28 public | 20,043 / 20,043 | 29 / 29 | passed (spec + design) |
| 5 | C# | C# / .NET 9 | `gitpp-cs` v0.3.4 | 28 / 28 public | 20,043 / 20,043 | 29 / 29 | passed (after v0.3.0 §2.5) |
| 6 | Perl | Perl 5 | `gitpp-pl` v0.1.9 | 28 / 28 public | 20,043 / 20,043 | 29 / 29 | passed (spec only) |
| 7 | C | C99, stdlib | `gitpp-c` v0.2.1 | 28 / 28 public | 20,043 / 20,043 | 29 / 29 | passed (spec only) |
| - | Rust | Rust | `gitpp-rust` | scaffolded | (not yet) | (not yet) | pending (needs cargo) |

**Three separate claims, three separate corpora.** They are not
interchangeable and the table keeps them in separate columns so they cannot be
conflated again:

- **Conformance** — the public 28-case corpus in `gitpp-conformance` (and the
  hub's private 1,151). One number per port.
- **Leaf fuzz** — `gitpp-cross-port`, byte-identical SHA-256 leaves over the
  20,043-record float64 corpus. All 5 leaf ports agree **20,043 / 20,043**,
  0 disagreements, 0 errors, ~37s. The number is the FULL corpus, not a sample.
- **Fold** — `gitpp-fold-conformance`, 29 hand-written cases whose expectations
  come from this spec and not from any port. All 5 ports pass **29 / 29**.

**Cross-port LEAF fuzz (spec §3 verification):** all 5 leaf ports
(Python, JS, C, C#, Perl) produce **byte-identical SHA-256 leaves** on
**20,043 / 20,043** random float64 records, 0 disagreements, 0 errors, ~37s.
Fuzz runner is in
`SuperInstance/gitpp-cross-port/fuzz.mjs`.

### Two implementations outside the five-port set are NOT conformant

Both for the same underlying reason: **neither had ever been byte-tested**, so
"all five ports agree" was a claim about five ports and said nothing about the
reference they were all ported from.

| impl | before | after | cause | report |
|---|---|---|---|---|
| `patchwork-quilt/git.pp-ng` (TypeScript hub) | **448 / 20,043 diverge** | **0 — 20,043 / 20,043** | `pyNumber()` tested `Math.abs(n) < 1e21` where §3 says `1e16` | [git.pp-ng#28](https://github.com/patchwork-quilt/git.pp-ng/issues/28) **fixed** |
| `patchwork-quilt/gitpp-beam` (Elixir) | static analysis, two boundaries | **unverified — no BEAM toolchain here** | the same stale `1e21`, plus `@json_safe_integer` (2^53−1) reused for *encoding* when §3's boundary is `1e16` | [gitpp-beam#2](https://github.com/patchwork-quilt/gitpp-beam/issues/2) **fix proposed, not executed** |

The Elixir case is more interesting, and it is specific to the BEAM.
`is_integer/1` is true for bignums, so a bare literal is an integer *by
construction*, and `encode/1`'s `is_integer` clause bypasses `leaf_number/1`
entirely. The constant that decides whether an integer survives a float64 round
trip is the right one for **decoding** and the wrong one for **encoding**; it
was doing one job and being asked to do a second. The two boundaries differ by
about 9.9e14, so this is a wide divergence rather than a boundary curiosity.

A conformance suite only covers what it is pointed at. Three bodies of code
agreed with each other and disagreed with the spec, and the one that mattered
most — the reference itself — had never been in the set at all.

The hub is now a measured participant rather than an assumption:
`gitpp-cross-port` reports **ports** and **hub** as two separate numbers,
because folding the reference into one `agree` count would make five
conformant ports look broken whenever the reference drifted. The Elixir port
has no toolchain in the environment that produced this table, so its row
reads *unverified* rather than *passing*.

A port is "conformant" if:
- It produces the byte-exact leaves in `cross-port-leafs.json` (the §3.5 test).
- It produces the byte-exact closure root for `poc-quilt-1@task-c` (the spec's pinned value).
- It passes the public conformance corpus in `SuperInstance/gitpp-conformance` (28 cases).
- It passes the canonical conformance corpus in `patchwork-quilt/git.pp-ng/conformance/` (1,151 cases; private answers).
- Its `derive(records)` returns state, conflicts, order, orphans that are byte-identical to the hub for every named case.
- It returns a `setters` map (§5.6) for every key in `state`, not only for contested keys.

The first three are sufficient for "publicly conformant". The last two are required for "production-ready".


