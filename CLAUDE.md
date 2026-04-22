# mathcore-units — Project CLAUDE.md

Workspace member of `libraries/thales/`. Inherits workspace-level rules
from `../CLAUDE.md` and global rules from
`~/.config/claude/claude-ent/CLAUDE.md`.

## Governance

Subordinate to thales. API decisions ultimately serve thales's unit
propagation needs (v0.10.0 and beyond) and mathlex's M-R2 annotation
payload. Once `0.1.0` ships on crates.io, additions are free but
breaking changes require coordinated major bumps with thales.

## Status

Pre-specification. The authoritative spec (MU-1…MU-N) will land in this
repo as `SPECIFICATION.md` before any non-trivial code is written.
Until the spec is frozen:

- No architecture rules yet (they land with the spec).
- No tests beyond the `cargo init` placeholder.
- Experimental code is acceptable during spec iteration; everything in
  this state is throwaway.

Once `SPECIFICATION.md` is accepted, this file gains the crate's
architecture rules (analogous to thales's six invariants) and the
placeholder lib.rs is replaced by the spec's core types.

## Constraints that already apply

- MIT license (set at repo init).
- `no_std + alloc` target must be preserved — consumers include embedded
  and wasm environments. Use `std` feature as the default opt-out.
- Serde is opt-in via the `serde` feature. Never hard-depend on serde.
- Dimension algebra must be exact (integer exponents or explicitly
  rational where required by the spec).

## Active work

None yet. Specification drafting is the first task once the workspace
is wired and the repo is registered with workspace-qdrant.
