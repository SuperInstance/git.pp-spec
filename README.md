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

## Cross-port verification (30/30 byte-for-byte equivalent)

[SuperInstance/gitpp-cross-port](https://github.com/SuperInstance/gitpp-cross-port) runs the same 30 cases through 4 ports (Python, JS, C#, Perl) and proves they produce byte-for-byte identical closure roots.

**Receipt: 30/30 cases agree. 4/4 pinned cross-port leaves match. 0 disagreements.**

The pinned closure root at `poc-quilt-1@task-c` (used as the cross-port gate):

```
60effe23970d8a5cc837636c44685502361288c1f00577e9addfa10cbd4fc91d
```

The 4 pinned cross-port leaf verification cases (in `cross-port-leafs.json`):

| Case | Input | Leaf |
|------|-------|------|
| 1 | minimal `{id:"a", kind:"task", parents:[], effects:{}, reason:null}` | `229f56e0…3124` |
| 2 | nested effects `{x: {y:1, z:[1,2,3]}}` | `64a3cd29…6ad9` |
| 3 | boolean + string effects | `97fd12ee…52f7` |
| 4 | non-BMP id (the rocket emoji 🚀) | `83d31b81…5e97` |

If a port produces a different leaf or root, that's a bug in the port, not in the spec. The cross-port artifact at [gitpp-cross-port](https://github.com/SuperInstance/gitpp-cross-port) is the running test that catches this.

## Versions

- **v0.3.0** (current): added §2.5 worked byte-level example, §3.5 4-case cross-port verification, §5.2 streaming fold, §12 cross-port verification table. Made the spec precise enough that 4 ports written from prose alone converge to byte-identical output.
- **v0.2.0**: 11 sections covering the full record format, the leaf, the closure root, the fold, the conflicts, the conformance corpus.
- **v0.1.0**: initial draft (kept for archeology).
