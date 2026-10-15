# Cross-port verification (30/30 byte-for-byte equivalent)

The 5 ports in this ecosystem that are written from the spec v0.3.0 alone produce byte-for-byte identical output:

- `SuperInstance/gitpp-c` — C99 (the 7th port, stdlib only, manual SHA-256)
- `git.pp-ng/referee/` — Python (the validator / reference, 2nd port)
- `SuperInstance/gitpp-js` — JavaScript (4th port)
- `SuperInstance/gitpp-cs` — C# / .NET 9 (5th port)
- `SuperInstance/gitpp-pl` — Perl 5 (6th port)

Plus the original:

- `patchwork-quilt/git.pp-ng` — TypeScript (the hub, 1st port)
- `patchwork-quilt/gitpp-beam` — Elixir (3rd port, doc-only passed)

**Receipt: 30/30 conformance cases agree across 5 ports. 4/4 pinned cross-port leaves match. 0 disagreements.**

The cross-port artifact at [SuperInstance/gitpp-cross-port](https://github.com/SuperInstance/gitpp-cross-port) is the running test. If a port produces different bytes, it's a bug in the port, not in the spec.

# How to use this spec — for port authors

## Step 0: Read the contract

Read `SPEC.md` end to end. It's 11 sections, ~10 minutes. The load-bearing rules are:

- §1 the record's five content fields
- §2 the content leaf
- §3 the leaf-number encoding
- §4 the closure root
- §5 the fold
- §6 the three-way verdict
- §7 conformance

## Step 1: Read the design docs

These are the *narrative* companion to the spec — they explain *why* the spec is what it is:

- `DESIGN.md` — the resolution algorithm in narrative form
- `docs/CHECKPOINT-V2.md` — the leaf and closure root in implementation form
- `docs/FORMAT-V2.md` — the `digest` field's claim semantics
- `docs/ENGINEERING.md` — the witness-freshness formula and the cross-port float fuzz
- `docs/OPEN-QUESTIONS.md` — the four open design questions (1a–1d)

## Step 2: Implement from the spec alone

This is the discipline. The conformance corpus in `conformance/` is yours to use, but its NAMES are public, its ANSWERS are not — write the port without looking at the answer set.

If you cannot answer a question from the spec + the design docs alone, that is a **finding** in your PR's body. List it. It might be a spec gap, or it might be that you need to look at the spec more carefully. Either way, it's not a permission to read the source.

## Step 3: Verify the byte-identical claim

The receipt for a conformant port is:

```
cargo test --release   # or whatever your build is
# confirms: closure root at poc-quilt-1@task-c = 60effe23970d8a5cc837636c44685502361288c1f00577e9addfa10cbd4fc91d
# 863/863 conformance cases identical (1,151/1,151 with the §6 families)
# 16,290 closure-root digests identical to the hub's referee
```

If the receipt matches, your port is conformant. If it doesn't, you have drifted; the right next step is to find the divergence in the leaf domain (use the `record_leaf` function on a single known record and compare your bytes to the hub's referee).

## Step 4: PR the port

The PR body should include:

- The receipt (§3 above)
- The language and toolchain
- A note on any spec gaps you found (§2's "findings")
- A note on any floats that diverged in the 10,000-value fuzz (with Python's rendering and yours, side by side)

A conformant port in a new language is one of the highest-leverage R&D artifacts the project accepts. The existing four ports (TypeScript, Python, Elixir, Rust) cover the major paradigms; a fifth in Lua, Zig, or Haskell would catch different assumptions.
