# git.pp-spec

> The canonical, normative spec for the git.pp record format. The contract a port must satisfy to be conformant.

This is the *contract*. Implementation notes (rationale, performance, measured numbers) live in `DESIGN.md` and `docs/CHECKPOINT-V2.md` in `patchwork-quilt/git.pp-ng`. This document says "the leaf is SHA-256 over the canonical JSON of five fields, with these byte-level encodings" — it does not say "the implementation in `src/leaf.ts` looks like this."

## Quick index

- [SPEC.md](SPEC.md) — the full spec
- The leaf construction (CHECKPOINT-V2.md "Record leaf" in the hub, §2 here)
- The closure root construction (CHECKPOINT-V2.md "Closure root" in the hub, §4 here)
- The fold (DESIGN.md §2 in the hub, §5 here)
- The three-way verdict (CHECKPOINT-V2.md "Migration story" in the hub, §6 here)
- The conformance corpus (DESIGN.md "Conformance" in the hub, §7 here)

## Status

- **v1** (no `digest` field): stable, first-class forever
- **v2** (current, the `digest` field): stable, migration is writers-first
- **next** (proposed): broaden the leaf to cover dispute and settlement records per the §6 doctrine doc

## Doc-only discipline

A port is conformant if its leaf, closure root, fold, and three-way verdict are byte-identical to the hub's referee for the entire conformance corpus. A port author who wants to be conformant should:

1. Read this spec.
2. Read `DESIGN.md`, `docs/CHECKPOINT-V2.md`, `docs/FORMAT-V2.md`, `docs/ENGINEERING.md` in `patchwork-quilt/git.pp-ng`.
3. NOT read `src/leaf.ts`, `referee/fold.py`, or any other implementation.
4. Write the port from the spec alone. If you can't answer a question from the spec, that's a finding in your PR — not a permission to read the source.

This is the same discipline the existing ports followed (the doc-only-discipline statement is on each port's README), and it's the same test that proves the spec is complete.
