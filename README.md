# mathcore-units

Unit and dimension algebra for the mathcore/thales ecosystem.

Provides:

- Dimension algebra over SI base dimensions (length, mass, time, electric
  current, thermodynamic temperature, amount of substance, luminous intensity)
- `Unit` types: base, prefixed (scale), composed (derived), alternative-system
- Unit arithmetic (multiply, divide, power) with automatic dimension tracking
- Conversion between compatible units via SI-base factoring
- Serde-capable wire format (opt-in via `serde` feature)
- `no_std + alloc` support (opt in via `default-features = false`)

## Status

**Pre-release.** The specification is being drafted; API will settle before
the `0.1.0` release to crates.io. Consumers downstream of mathcore-units
(mathcore-constants, mathlex annotations, thales unit propagation) depend on
a stable `0.1.0` and should track the specification before integrating.

## Relationship to thales

This crate is a **subordinate member** of the thales workspace. Its API is
driven by thales's needs for unit-aware computer algebra, under the thales
v0.10.0 unit-propagation milestone and mathlex's M-R2 annotation work.
Backward compatibility is maintained additively once `0.1.0` ships.

See the workspace-level `CLAUDE.md` for governance notes.

## Install

Not yet published to crates.io. Track this repository or the thales workspace
for release readiness.

## License

MIT. See `LICENSE`.
