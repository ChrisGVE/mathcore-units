# mathcore-units — Specification

**Version:** draft-1 (2026-04-22)
**Status:** draft, pre-freeze
**Target release:** v0.1.0
**Crate kind:** data-only (types + catalog + aliases; no algebra logic)

This document specifies the authoritative contract for `mathcore-units`.
Once accepted, the types, enum tag names, and wire format described
here are frozen for the v0.x series. Additions (new UnitId variants,
new aliases, new ConstantId variants) are permitted in minor versions;
removals or renames require a major version bump coordinated with
thales.

The sibling documents `unitalg/SPECIFICATION.md` and
`mathcore-constants/SPECIFICATION.md` build on the types defined here.
Neither is in scope for this document.

---

## 1. Scope and non-goals

### 1.1 In scope

- The `Dimension` type (sparse map of base dimensions to exponents).
- The `UnitId` enum — closed, stable vocabulary of named unit atoms.
- The `System` enum — SI, Imperial, None. (MKS, CGS, Natural, Planck
  are NOT separate Systems in v0.1.0; their still-relevant units live
  inside SI as ordinary SI-derived legacy entries.)
- The `UnitSpec` static catalog entry — per-unit metadata and
  conversion-to-canonical-SI specification.
- The `UnitExpression` structural type — restricted mini-AST for unit
  arithmetic, with enum-valued leaves and a closed operator set.
- The `ConstantId` enum — ID-only, paired with values in
  mathcore-constants.
- The `Conversion` enum — Linear, Affine, Logarithmic, FromConstant,
  Expression (escape hatch).
- Alias lookup tables mapping parsed symbols to canonical `UnitId`.
- Prefix tables (SI decimal prefixes, binary prefixes).
- Vocabulary audit rules that guarantee zero ambiguity.
- Serde-capable wire format for every public type (opt-in feature).

### 1.2 Out of scope

- Unit arithmetic operations (`mul`, `div`, `pow`, `convert`, `factor`,
  `expand`, dimension check, system selection) — live in `unitalg`.
- Numeric values for constants — live in `mathcore-constants`.
- Parser integration — mathlex consumes these types via annotation
  and tokenizer but does not own them.
- Arc<Expr> interop — mathcore-units types never touch Arc<Expr>;
  thales's Rule 1 is respected by keeping unit data in a parallel
  type space.
- Currency units — rejected per session 8 (non-constant conversion
  rates).
- Infotech units (`bit`, `byte`, `flops`, `ips`, `MTBF`, `dB` for
  digital comms) — rejected per session 14 (adds noise without CAS
  value; any CS graduate performs these without a CAS).

---

## 2. Types

### 2.1 `BaseDim` — base dimension enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum BaseDim {
    Length,        // L, SI: meter
    Mass,          // M, SI: kilogram
    Time,          // T, SI: second
    Current,       // I, SI: ampere
    Temperature,   // Θ, SI: kelvin
    Amount,        // N, SI: mole
    Luminosity,    // J, SI: candela
    Angle,         // plane angle, SI: radian (dimensionless in strict SI, first-class here)
}
```

**Rationale for 8 base dimensions:** the seven SI base dimensions plus
plane angle. Angle as first-class enforces the `sin(x)` / `exp(x)`
consistency rule (argument must be dimensionless or angle). Solid angle
(steradian) is NOT a base dimension — its definition depends on the
embedding space (area/r² in 3D Euclidean, different in other spaces),
while plane angle (arc/radius) is universal. Steradian is treated as a
dimensionless named unit in the catalog.

**Extensibility:** adding a new BaseDim variant is a breaking wire-format
change. Additions only justified by broad mathematical coverage. The
enum is treated as closed for v0.x.

### 2.2 `Dimension` — exponent map

```rust
#[derive(Debug, Clone, Default, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub struct Dimension(pub BTreeMap<BaseDim, i32>);
```

Zero-exponent entries MUST be omitted. Two `Dimension`s are equal iff
their maps are equal (after zero removal). Dimensionless is represented
by an empty map.

**Arithmetic (convenience, spec-level):**
- Multiplication of dimensions: add exponents per base.
- Division: subtract exponents.
- Power: scale exponents.

These operations live in `unitalg`, but the type must support the
necessary trait bounds for their implementation (`Clone`, `Eq`,
iteration).

**Wire format:**
```json
{"Length": 1, "Time": -2}
```
Omitted keys have exponent 0. Empty object = dimensionless.

### 2.3 `System` — preferred-unit system

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum System {
    SI,          // meter-kilogram-second; includes SI-derived legacy units
                 //  (Gram, Dyne, Erg, Gauss, Poise, Stokes, bar, torr, atmosphere,
                 //  astronomical units, eV, etc.)
    Imperial,    // UK default; US customary variants exposed as explicit UnitIds
    None,        // system-neutral (dimensionless, counts)
}
```

The `System` tag on a unit controls display preference and
conversion-factor selection. It is not a separate dimension; units of
the same dimension across systems convert via Linear / Affine /
FromConstant conversion.

**Scope decision (v0.1.0):** only three systems. MKS dropped as
pre-SI redundant. CGS dropped as a system; its still-relevant units
(Gauss, Erg, Dyne, Poise, Stokes) live inside SI with explicit scale
factors — treated as conventional legacy aliases the way bar, torr, and
mmHg are. CGS's obsolete EM variants (StatVolt, StatCoulomb, Oersted,
Maxwell, abampere/abvolt) are dropped entirely.

**Natural and Planck conventions** are not separate Systems. The only
practical-use "natural" unit is `eV`, which ships as an SI-derived
energy unit with exact scale factor (2019 SI redefinition). Planck
units (Planck length, time, mass) are deferred to a future catalog
addition.

**Astronomy units** (AU, lightyear, parsec) are ordinary SI-derived
length units with large scale factors (exact modern definitions: IAU
2012 AU, c · Julian year for ly). No special treatment — same category
as km or nm.

**Imperial UK default:** "Imperial" refers to the UK system
post-1824. Length and mass units (Foot, Inch, Yard, Mile, Pound,
Ounce) are identical UK/US post-1959 international agreement, single
UnitId each. Volume units differ (UK gallon = 4.546 L, US gallon =
3.785 L): UK variant is the default UnitId (`Gallon`, `Pint`, `Quart`,
`FluidOunce`); US variants are explicit (`GallonUS`, `PintUS`,
`QuartUS`, `FluidOunceUS`).

### 2.4 `UnitId` — vocabulary enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum UnitId {
    // --- SI base ---
    Meter, Kilogram, Second, Ampere, Kelvin, Mole, Candela,
    Radian, Steradian,

    // --- SI derived (named) ---
    Hertz, Newton, Pascal, Joule, Watt, Coulomb, Volt, Farad, Ohm,
    Siemens, Weber, Tesla, Henry, Lumen, Lux, Becquerel, Gray, Sievert,
    Katal, DegreeCelsius,

    // --- SI-derived additional ---
    Liter,           // 10^-3 m³
    Minute, Hour, Day, Year,           // conventional time
    Degree, ArcMinute, ArcSecond,      // plane angle
    Electronvolt,                       // energy; exact factor since 2019 SI redefinition
    AstronomicalUnit, LightYear, Parsec,  // exact by definition
    Bar, Torr, Atmosphere, Mmhg,       // pressure (conventional)
    Barn,                               // nuclear cross-section (10^-28 m²)
    Jansky,                             // radio astronomy flux density
    Hectare,                            // area
    Tonne,                              // 1000 kg

    // --- SI-derived legacy (from CGS heritage, still in active use) ---
    Gram,                               // 10⁻³ kg; prefix target for sub-kg masses (mg, μg, ng)
    Dyne, Erg,                          // force, energy — astrophysics/older literature
    Poise, Stokes,                      // dynamic, kinematic viscosity — rheology/petroleum
    Gauss,                              // magnetic flux density — astrophysics pulsar/stellar fields

    // --- Imperial (UK default; US variants below) ---
    Foot, Inch, Yard, Mile, NauticalMile,       // UK = US, post-1959 agreement
    Pound, Ounce, Slug, Grain,                   // UK = US, post-1959 agreement
    Gallon, Quart, Pint, FluidOunce,             // UK (Gallon = 4.546 L)
    GallonUS, QuartUS, PintUS, FluidOunceUS,     // US customary (GallonUS = 3.785 L)
    DegreeFahrenheit, DegreeRankine,
    FootPound, BritishThermalUnit, Horsepower,

    // --- Radiation (specialty) ---
    Curie,                              // activity (obsolete Bq variant, still in historical data)
    Roentgen,                           // exposure (X-ray / gamma in air)
    // Rad (absorbed dose) and Rem (dose equivalent) dropped — obsolete;
    // Gray (Gy) and Sievert (Sv) cover the same quantities, and `rad`
    // alias would collide with Radian.

    // --- Mass (commercial / specialty) ---
    Carat,                              // 200 mg exactly (metric carat, ISO)

    // --- Logarithmic family ---
    Decibel,                            // dimensionless ratio (10 log10)
    DecibelMilliwatt,                   // dB ref 1 mW → power
    DecibelWatt,                        // dB ref 1 W → power
    DecibelSpl,                         // dB ref 20 μPa → sound pressure
    Neper,                              // natural-log ratio (ln)

    // --- Constants-used-as-units (for quantities expressed as "1.5 M_sun") ---
    SolarMass, EarthMass, JupiterMass, LunarMass,
    ElectronMass, ProtonMass, NeutronMass,
    SolarRadius, EarthRadius,
    SolarLuminosity,
    AtomicMassUnit,                     // u

    // --- None / dimensionless ---
    Count,                              // pure counting unit
    Percent, PerMille, PartsPerMillion, PartsPerBillion,
}
```

**Closure policy:** `UnitId` is closed for v0.x. Adding a variant is a
minor version bump (non-breaking because consumers don't pattern-match
exhaustively on external enums). Renaming or removing a variant is a
major version bump.

**Naming policy:** CamelCase, English, mathematician-physicist
convention. Avoid abbreviated variant names; abbreviations live in
aliases (§ 6).

### 2.5 `UnitSpec` — static catalog entry

```rust
#[derive(Debug, Clone)]
pub struct UnitSpec {
    pub id: UnitId,
    pub dimension: Dimension,
    pub system: System,
    pub canonical_symbol: &'static str,       // e.g. "m", "kg", "N"
    pub display_name: &'static str,           // e.g. "meter", "Newton"
    pub aliases: &'static [&'static str],     // e.g. ["metre", "meters"]
    pub conversion: Conversion,               // to canonical SI
    pub applies_prefix: PrefixPolicy,
}
```

**Not serde.** Static catalog entries are Rust-resident; consumers
query via `catalog::lookup(UnitId) -> &'static UnitSpec`.

### 2.6 `Conversion` — conversion-to-canonical-SI

```rust
#[derive(Debug, Clone)]
pub enum Conversion {
    /// Linear: x_si = scale · x_unit
    Linear { scale: Rational },
    /// Affine: x_si = scale · x_unit + offset   (e.g. Celsius, Fahrenheit)
    Affine { scale: Rational, offset: Rational },
    /// Logarithmic:
    ///     x_si = reference_unit_value · reference_scale · base^(x_unit / factor)
    /// where `reference_unit_value` is the canonical-SI value of the
    /// referenced unit (1.0 when reference is None, meaning dimensionless),
    /// `reference_scale` scales the reference (1 for dBW, 10⁻³ for dBm,
    /// 2×10⁻⁵ for dB SPL), and `factor` is 10 for power-dB or 20 for
    /// amplitude-dB.
    Logarithmic {
        reference: Option<UnitId>,  // None = dimensionless (plain dB, Neper)
        reference_scale: Rational,  // factor on the reference (1 for dBW, 1e-3 for dBm, 2e-5 for dB SPL)
        base: LogBase,              // base of the logarithm (Ten / E / Two)
        factor: i32,                // 10 for power-dB, 20 for amplitude-dB
    },
    /// Scale equals a constant's value (e.g. solar_mass = M_sun kg)
    FromConstant { id: ConstantId },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum LogBase {
    Ten,    // base 10 (all dB variants)
    E,      // base e (Neper)
    Two,    // base 2 (audio / bit-ratio contexts, reserved)
}
```

`scale` and `offset` use arbitrary-precision `Rational`. Exact where
possible (SI-2019 redefines several units with exact scale factors;
astronomy units defined exactly).

The v0.1.0 catalog is covered by Linear / Affine / Logarithmic /
FromConstant. An `Expression`-based escape hatch was considered and
dropped — no current catalog entry requires it, and keeping the set
closed simplifies unitalg's case analysis. If a future catalog entry
genuinely needs symbolic conversion, the Conversion enum gains a
variant in a minor version bump.

### 2.7 `PrefixPolicy` — prefix admissibility

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PrefixPolicy {
    SiDecimal,           // km, MPa, mA, nF, etc.
    SiDecimalRestricted, // only power-of-3 prefixes (no hecto/deca/deci/centi)
    None,                // prefixes not allowed (hours, days, AU, ly, …)
    Binary,              // reserved; not used in v0.x (infotech cut)
}
```

Rationale: some units accept all SI decimal prefixes (meter: km, cm,
mm, μm, nm, …); some accept only engineering-notation prefixes (Pa:
kPa, MPa, GPa — but NOT hPa in strict engineering contexts, though
widely used in meteorology — hence `SiDecimal` default with
`SiDecimalRestricted` as a style option); some accept none (AU,
lightyear, day).

The catalog's `applies_prefix` value combined with the catalog's
`canonical_symbol` and the prefix table determines the set of valid
input tokens for the unit.

### 2.8 `UnitExpression` — parallel mini-AST

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[cfg_attr(feature = "serde", serde(tag = "kind", content = "value"))]
pub enum UnitExpression {
    /// Leaf: a named unit atom.
    Atom { id: UnitId, prefix: Option<SiPrefix> },
    /// Leaf: numeric literal (scale factor in unit arithmetic).
    Literal(Rational),
    /// Binary operator: Mul, Div, Pow.
    Binary {
        op: BinaryOp,
        left: Box<UnitExpression>,
        right: Box<UnitExpression>,
    },
    /// Unary operator: Neg (for additive-unification) or Inv (for division shortcut).
    Unary { op: UnaryOp, child: Box<UnitExpression> },
    /// Logarithmic operator family: Log10, Log2, Ln, Exp.
    Log {
        op: LogOp,
        argument: Box<UnitExpression>,
        // Reference base (for dB-family units carrying a reference)
        reference: Option<UnitId>,
    },
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum BinaryOp { Mul, Div, Pow, Add, Sub }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum UnaryOp { Neg, Inv }

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum LogOp { Log10, Log2, Ln, Exp }
```

**Semantics of `Add`/`Sub` in `UnitExpression`:** these represent
*dimension-unification* operations, not pointwise addition of unit
values. `u + u` evaluates (via unitalg) to `u`, not `2u`. This is
encoded at the type level by the separate `UnitExpression` (as opposed
to reusing `Expression` which would imply pointwise arithmetic).

`Log` subtree is specialized for logarithmic units: the `reference`
field carries the reference-unit when the unit is a dB-family member
(e.g., dBm referenced to milliwatt). For plain log/ln/exp used in unit
algebra (thermodynamic entropy `k_B ln Ω`, etc.), `reference` is None.

**Wire format:**
```json
{"kind": "Atom", "value": {"id": "Meter", "prefix": "Kilo"}}
{"kind": "Binary", "value": {
    "op": {"kind": "Div"},
    "left": {"kind": "Atom", "value": {"id": "Meter"}},
    "right": {"kind": "Atom", "value": {"id": "Second"}}
}}
```

### 2.9 `SiPrefix` — prefix enum

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum SiPrefix {
    Quecto, Ronto, Yocto, Zepto, Atto, Femto, Pico, Nano, Micro, Milli,
    Centi, Deci,
    Deca, Hecto,
    Kilo, Mega, Giga, Tera, Peta, Exa, Zetta, Yotta, Ronna, Quetta,
}

impl SiPrefix {
    pub const fn exponent(self) -> i32 { /* -30, -27, -24, ..., 30 */ }
    pub const fn symbol(self) -> &'static str { /* "q", "r", ..., "Q" */ }
    pub const fn is_power_of_3(self) -> bool { /* true except Centi/Deci/Deca/Hecto */ }
}
```

**Prefix symbols (strict, case-sensitive):**

| Prefix | Symbol | Exponent |
|---|---|---|
| Quecto | `q` | 10⁻³⁰ |
| Ronto | `r` | 10⁻²⁷ |
| Yocto | `y` | 10⁻²⁴ |
| Zepto | `z` | 10⁻²¹ |
| Atto | `a` | 10⁻¹⁸ |
| Femto | `f` | 10⁻¹⁵ |
| Pico | `p` | 10⁻¹² |
| Nano | `n` | 10⁻⁹ |
| Micro | `μ` (U+03BC) or `µ` (U+00B5) | 10⁻⁶ |
| Milli | `m` | 10⁻³ |
| Centi | `c` | 10⁻² |
| Deci | `d` | 10⁻¹ |
| Deca | `da` | 10¹ |
| Hecto | `h` | 10² |
| Kilo | `k` | 10³ |
| Mega | `M` | 10⁶ |
| Giga | `G` | 10⁹ |
| Tera | `T` | 10¹² |
| Peta | `P` | 10¹⁵ |
| Exa | `E` | 10¹⁸ |
| Zetta | `Z` | 10²¹ |
| Yotta | `Y` | 10²⁴ |
| Ronna | `R` | 10²⁷ |
| Quetta | `Q` | 10³⁰ |

**Important: ASCII `u` is NOT accepted as Micro.** The AtomicMassUnit
(`u`) claims this letter as its canonical alias. Micro must be written
as `μ` (Greek letter) or `µ` (Unicode MICRO SIGN, U+00B5) — both
canonical. Fall-back plain-text input: spell out `micro` or use the
composite unit name directly (`micrometer` matches via alias lookup).

Binary prefixes (Kibi/Mebi/…) are NOT included. Removed with infotech
per session 14.

---

## 3. The `Conversion::FromConstant` mechanism

When a unit's scale factor is the value of a physical constant (solar
mass expressed in kilograms, electronvolt expressed in joules for a
conversion into SI), the `Conversion` is `FromConstant { id: ConstantId }`.

At conversion time, unitalg queries mathcore-constants for the
constant's value. mathcore-units never stores numerical values of
constants — it stores only `ConstantId`.

Example: the unit `SolarMass` has `conversion: Conversion::FromConstant {
id: ConstantId::SolarMass }`. A user writing `1.5 M_sun` parses as
quantity with unit SolarMass; when thales or unitalg converts this to
kilograms, it looks up ConstantId::SolarMass in mathcore-constants,
gets its kg value, multiplies.

---

## 4. `ConstantId` enum (ID-only)

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum ConstantId {
    // --- Fundamental physical (CODATA exact) ---
    SpeedOfLight,            // c = 299_792_458 m/s exactly
    PlanckConstant,          // h = 6.62607015e-34 J·s exactly
    ReducedPlanckConstant,   // ℏ = h / (2π)
    ElementaryCharge,        // e = 1.602176634e-19 C exactly
    BoltzmannConstant,       // k_B = 1.380649e-23 J/K exactly
    AvogadroNumber,          // N_A = 6.02214076e23 mol⁻¹ exactly
    // --- Fundamental physical (measured with uncertainty) ---
    GravitationalConstant,   // G
    FineStructureConstant,   // α
    ElectronMass, ProtonMass, NeutronMass,
    ElectronChargeToMass,    // e/m_e
    VacuumPermittivity,      // ε₀
    VacuumPermeability,      // μ₀ (exact by definition, but depends on system)
    // --- Astronomical ---
    SolarMass, EarthMass, JupiterMass, LunarMass,
    SolarRadius, EarthRadius,
    SolarLuminosity,
    AstronomicalUnit, LightYear, Parsec,    // defined-exact distances
    // --- Mathematical ---
    Pi, E, EulerMascheroni, GoldenRatio,
    // --- Chemical (examples; catalog expands) ---
    MolarGasConstant, FaradayConstant, RydbergConstant, StefanBoltzmannConstant,
    // --- Nuclear / atomic ---
    AtomicMassUnit,          // u = 1/12 mass of C-12
    BohrRadius, BohrMagneton, NuclearMagneton,
    ComptonWavelength,
}
```

`ConstantId` lives in mathcore-units so mathcore-units's catalog can
reference constants in `Conversion::FromConstant`. The actual values
(with units, uncertainties, citations) live in mathcore-constants.

**ConstantSpec unit-field contract:** mathcore-constants's
`ConstantSpec` will carry its unit as a `UnitExpression` (from § 2.8),
NOT a single `UnitId`. This is required because many constants have
compound units (`G` is m³·kg⁻¹·s⁻², `k_B` is J·K⁻¹, `N_A` is mol⁻¹,
`σ` is W·m⁻²·K⁻⁴). A single `UnitId` cannot express these. The
mathcore-constants spec embeds the UnitExpression type from here
without introducing a parallel type.

**Astronomy units in both enums:** `AstronomicalUnit`, `LightYear`,
`Parsec` appear in both the `UnitId` enum (as scaled-meter length
units) and the `ConstantId` enum (as named physical quantities). This
is intentional, not redundant:
- As `UnitId`: user writes `5 ly` meaning five lightyears of length
- As `ConstantId`: user writes `c · t / lightyear` where `lightyear`
  is a named physical distance constant in a main expression
The two use cases are disjoint (unit context vs. main-expression
context), and both are legitimate. The UnitId-side conversion is
`Linear { scale }` with the exact IAU value baked in; the
ConstantId-side carries the same value with citation metadata.
This split keeps mathcore-units pure-data (no physical numbers) and
mathcore-constants free to evolve its value schema independently.

---

## 5. Catalog

The catalog is a static `&'static [UnitSpec]` with a lookup index
`fn lookup_unit_id(UnitId) -> &'static UnitSpec`. Details of each
category below.

### 5.1 SI base (7 + radian)

| UnitId | Symbol | Dimension | Conversion |
|---|---|---|---|
| Meter | m | Length | Linear { scale: 1 } |
| Kilogram | kg | Mass | Linear { scale: 1 } |
| Second | s | Time | Linear { scale: 1 } |
| Ampere | A | Current | Linear { scale: 1 } |
| Kelvin | K | Temperature | Linear { scale: 1 } |
| Mole | mol | Amount | Linear { scale: 1 } |
| Candela | cd | Luminosity | Linear { scale: 1 } |
| Radian | rad | Angle | Linear { scale: 1 } |

All SI base have `system: SI`, `applies_prefix: SiDecimal`.

Steradian (sr) is a catalog entry with `dimension: empty` and
`display_name: "steradian"`. Not a base dimension.

### 5.2 SI derived named

| UnitId | Symbol | Dimension | Notes |
|---|---|---|---|
| Hertz | Hz | T⁻¹ | |
| Newton | N | L·M·T⁻² | |
| Pascal | Pa | L⁻¹·M·T⁻² | |
| Joule | J | L²·M·T⁻² | |
| Watt | W | L²·M·T⁻³ | |
| Coulomb | C | T·I | |
| Volt | V | L²·M·T⁻³·I⁻¹ | |
| Farad | F | L⁻²·M⁻¹·T⁴·I² | |
| Ohm | Ω | L²·M·T⁻³·I⁻² | Alias: "ohm" (ASCII) |
| Siemens | S | L⁻²·M⁻¹·T³·I² | |
| Weber | Wb | L²·M·T⁻²·I⁻¹ | |
| Tesla | T | M·T⁻²·I⁻¹ | **Case-sensitive bare `T` = Tesla, always** |
| Henry | H | L²·M·T⁻²·I⁻² | |
| Lumen | lm | J | (steradian is dimensionless) |
| Lux | lx | L⁻²·J | |
| Becquerel | Bq | T⁻¹ | |
| Gray | Gy | L²·T⁻² | |
| Sievert | Sv | L²·T⁻² | Same dimension as Gy; different meaning |
| Katal | kat | T⁻¹·N | |
| DegreeCelsius | °C | Temperature | Affine { scale: 1, offset: 273.15 } |

### 5.3 SI-derived additional

| UnitId | Symbol | Dimension | Notes |
|---|---|---|---|
| Liter | L, l | L³ | scale: 10⁻³ relative to m³. Uppercase `L` canonical. |
| Minute | min | Time | Affine? No — Linear { scale: 60 } |
| Hour | h | Time | Linear { scale: 3600 } |
| Day | d | Time | Linear { scale: 86400 } |
| Year | yr | Time | Linear { scale: 31557600 } (Julian year exact) |
| Degree | ° | Angle | Linear { scale: π/180 }; alias "deg" |
| ArcMinute | ′ | Angle | Linear { scale: π/10800 }; alias "arcmin" |
| ArcSecond | ″ | Angle | Linear { scale: π/648000 }; alias "arcsec" |
| Electronvolt | eV | Energy | Linear { scale: 1.602176634e-19 } (exact) |
| AstronomicalUnit | au | Length | Linear { scale: 149597870700 } (IAU 2012 exact) |
| LightYear | ly | Length | Linear { scale: 9460730472580800 } (c · Julian year, exact) |
| Parsec | pc | Length | Linear { scale: 648000·au/π } |
| Bar | bar | Pressure | Linear { scale: 100000 } |
| Torr | Torr | Pressure | Linear { scale: 101325/760 } |
| Atmosphere | atm | Pressure | Linear { scale: 101325 } (exact) |
| Mmhg | mmHg | Pressure | Linear { scale: 133.322387415 } |
| Barn | b | Area | Linear { scale: 10⁻²⁸ } |
| Jansky | Jy | W/m²/Hz | Linear { scale: 10⁻²⁶ } |
| Hectare | ha | Area | Linear { scale: 10000 } |
| Tonne | t | Mass | Linear { scale: 1000 } |

Minute/Hour/Day/Year have `applies_prefix: None`. Liter has SiDecimal.

### 5.4 SI-derived legacy (CGS heritage, still in active use)

CGS as a system is dropped (§ 2.3). A handful of CGS-heritage units
remain in active scientific use and are retained as ordinary
SI-derived entries with explicit scale factors — same category as bar,
torr, and mmHg. Obsolete CGS EM variants (StatVolt, StatAmpere,
StatCoulomb, Oersted, Maxwell, abampere / abvolt) are dropped.

| UnitId | Symbol | System | Notes |
|---|---|---|---|
| Gram | g | SI | Linear { scale: 10⁻³ } → kg. Prefix target for sub-kg masses (mg, μg, ng). Required because SI base kg already carries `kilo`. |
| Dyne | dyn | SI | Force; Linear { scale: 10⁻⁵ } → newton. Astrophysics / older literature. |
| Erg | erg | SI | Energy; Linear { scale: 10⁻⁷ } → joule. Astrophysics luminosities (1 L_sun ≈ 3.828 × 10³³ erg/s). |
| Poise | P | SI | Dynamic viscosity; Linear { scale: 0.1 } → Pa·s. Rheology / petroleum literature. |
| Stokes | St | SI | Kinematic viscosity; Linear { scale: 10⁻⁴ } → m²/s. Similar context. |
| Gauss | G | SI | Magnetic flux density; Linear { scale: 10⁻⁴ } → tesla. Astrophysics pulsar / stellar-field literature. **Case-sensitive: bare `G` = Gauss, not Giga prefix.** |

**Collision alert:** bare `G` = Gauss. The prefix `G` (giga) composes
only when followed by a unit symbol: `Gm` = gigameter, `GHz` = gigahertz.
Bare standalone `G` in a unit expression is Gauss. The gravitational
constant `G` is not a unit — it lives in the constants namespace and
in a unit-expression context never collides (constants don't appear in
unit-expression leaves). If a user's main expression uses `G` as the
gravitational constant, the token resolves in the main-expression
space, not the unit-annotation space.

### 5.5 Imperial (UK default; US variants explicit)

"Imperial" refers to the UK system post-1824. Length and mass units
are UK = US (post-1959 international agreement). Volume units
differ — UK variant is the default UnitId, US variant is explicit.

| UnitId | Symbol | Dimension | Linear scale → SI canonical |
|---|---|---|---|
| Foot | ft | Length | 0.3048 (exact, UK = US) |
| Inch | in | Length | 0.0254 (exact, UK = US) |
| Yard | yd | Length | 0.9144 (exact, UK = US) |
| Mile | mi | Length | 1609.344 (exact, UK = US statute mile) |
| NauticalMile | nmi | Length | 1852 (exact) |
| Pound | lb | Mass | 0.45359237 (exact, UK = US) |
| Ounce | oz | Mass | 0.028349523125 (UK = US avoirdupois) |
| Slug | slug | Mass | 14.593902937206 |
| Grain | gr | Mass | 6.479891e-5 |
| Gallon | gal | Volume | 4.54609e-3 (exact, **UK gallon**) |
| Quart | qt | Volume | 1.1365225e-3 (UK, = ¼ UK gallon) |
| Pint | pt | Volume | 5.6826125e-4 (UK, = ⅛ UK gallon) |
| FluidOunce | floz | Volume | 2.84130625e-5 (UK, = 1/160 UK gallon) |
| GallonUS | gal_us | Volume | 3.785411784e-3 (exact, US) |
| QuartUS | qt_us | Volume | 9.46352946e-4 (US) |
| PintUS | pt_us | Volume | 4.73176473e-4 (US) |
| FluidOunceUS | floz_us | Volume | 2.95735295625e-5 (US) |
| DegreeFahrenheit | °F | Temperature | Affine { scale: 5/9, offset: (273.15 - 32·5/9) } |
| DegreeRankine | °R | Temperature | Linear { scale: 5/9 } |
| FootPound | ft·lb | Energy | 1.3558179483314 |
| BritishThermalUnit | BTU | Energy | 1055.05585 (IT definition; catalog note lists variants) |
| Horsepower | hp | Power | 745.6998715822702 (mechanical HP; metric HP differs, not in catalog) |

**Strict alias enforcement — mass units:**
- `gr` → **Grain** only. Never accepted as Gram abbreviation.
- `g` → **Gram** only (§ 5.4 legacy entry).
- Users wanting Gram must use `g`; `gr` is imperial grain.
- Catalog's alias table enforces this via audit.

**Alias convention for US volume variants:** the US-specific UnitIds
accept aliases like `usgal`, `gal_us`, `us_gallon`, `gallon_us`. The
bare `gal` alias resolves to UK Gallon (the default). Callers targeting
US context must explicitly say `gal_us`.

### 5.5a Mass (commercial / specialty)

| UnitId | Symbol | Dimension | Conversion |
|---|---|---|---|
| Carat | ct | Mass | Linear { scale: 2e-4 } (exact, 200 mg per ISO metric carat) |

Alias `ct` — no collision: `c` (Centi) + `t` (Tonne) decomposition
would yield centi-tonne (10 kg), never encountered in practice;
direct-alias precedence resolves unambiguously to Carat.

Karat (as in gold purity, e.g., 18-karat gold) is a ratio, not a mass
unit — NOT in catalog. Use `PartsPerX` or a fraction for purity.

### 5.6 Logarithmic units

| UnitId | Symbol | Conversion |
|---|---|---|
| Decibel | dB | Logarithmic { reference: None, reference_scale: 1, base: Ten, factor: 10 } — plain power ratio |
| DecibelMilliwatt | dBm | Logarithmic { reference: Some(Watt), reference_scale: 10⁻³, base: Ten, factor: 10 } |
| DecibelWatt | dBW | Logarithmic { reference: Some(Watt), reference_scale: 1, base: Ten, factor: 10 } |
| DecibelSpl | dB SPL | Logarithmic { reference: Some(Pascal), reference_scale: 2×10⁻⁵, base: Ten, factor: 20 } |
| Neper | Np | Logarithmic { reference: None, reference_scale: 1, base: E, factor: 1 } |

The `reference_scale` encodes the per-variant reference level directly,
eliminating the earlier "offset for milli/μPa" prose workaround. For dB
variants with no physical reference (plain dB, Np), `reference` is
`None` and `reference_scale` is 1.

**Note on dB amplitude vs. power:** power dB uses factor 10 (`10 log10(P/P_ref)`);
amplitude dB uses factor 20 (`20 log10(A/A_ref) = 10 log10(A²/A_ref²)`).
The spec stores `factor` per dB variant.

### 5.7 Constants-used-as-units

| UnitId | Symbol | Dimension | Conversion |
|---|---|---|---|
| SolarMass | M_sun (also ☉) | Mass | FromConstant { id: SolarMass } |
| EarthMass | M_⊕ (also M_earth) | Mass | FromConstant { id: EarthMass } |
| JupiterMass | M_J | Mass | FromConstant { id: JupiterMass } |
| LunarMass | M_☾ (also M_moon) | Mass | FromConstant { id: LunarMass } |
| ElectronMass | m_e | Mass | FromConstant { id: ElectronMass } |
| ProtonMass | m_p | Mass | FromConstant { id: ProtonMass } |
| NeutronMass | m_n | Mass | FromConstant { id: NeutronMass } |
| SolarRadius | R_sun | Length | FromConstant { id: SolarRadius } |
| EarthRadius | R_⊕ | Length | FromConstant { id: EarthRadius } |
| SolarLuminosity | L_sun | Power | FromConstant { id: SolarLuminosity } |
| AtomicMassUnit | u (or Da / dalton) | Mass | FromConstant { id: AtomicMassUnit } |

Astronomical/particle-mass symbols use subscripts (`_sun`, `_earth`,
`_e`, `_p`, `_n`) to avoid collision with standalone unit letters or
prefixes. Unicode glyphs (`⊙`, `⊕`, `☾`) accepted as aliases where
unambiguous.

### 5.8 Dimensionless

| UnitId | Symbol | Dimension | Notes |
|---|---|---|---|
| Count | — | empty | Pure counting |
| Percent | % | empty | Linear { scale: 0.01 } |
| PerMille | ‰ | empty | Linear { scale: 0.001 } |
| PartsPerMillion | ppm | empty | Linear { scale: 1e-6 } |
| PartsPerBillion | ppb | empty | Linear { scale: 1e-9 } |
| Steradian | sr | empty | (named dimensionless) |

---

## 6. Aliases and the collision audit

### 6.1 Alias lookup

Alias table:

```rust
pub fn lookup_alias(s: &str) -> Option<UnitId>;
pub fn lookup_with_prefix(s: &str) -> Option<(Option<SiPrefix>, UnitId)>;
```

**Resolution order:**
1. `lookup_alias(s)` — direct match against a catalog alias (longest
   match wins).
2. If (1) fails, try prefix decomposition: split at every prefix
   boundary (1-char and 2-char prefixes), check if the remaining
   substring is a canonical symbol or alias.
3. If both fail, the token is not a valid unit.

**Example resolutions:**
- `"m"` → `(None, Meter)` via direct match
- `"km"` → no direct match; try `k` prefix + `m` → `(Some(Kilo), Meter)`
- `"cm"` → prefix decomposition `c` (Centi) + `m` (Meter) →
  `(Some(Centi), Meter)`. Centimeter is NOT a standalone UnitId (dropped
  with CGS system simplification); the prefix machinery handles it.
- `"T"` → direct match to Tesla. Never decomposed to Tera + nothing.
- `"TT"` → no direct match; prefix decomp: `T` + `T` → `(Some(Tera), Tesla)`.
- `"ms"` → ambiguous between `(Some(Milli), Second)` and `(None, Meter)·(None, Second)`. **Resolution:** the alias table lists `"ms"` as `(Some(Milli), Second)`. Implicit unit juxtaposition (`m · s` for meter-second) requires an explicit operator; bare token `ms` is always millisecond.

### 6.2 Alias list (by UnitId; non-exhaustive, catalog freeze will be exhaustive)

| UnitId | Aliases |
|---|---|
| Meter | metre, meters, metres |
| Kilogram | kilogramme, kilos |
| Second | sec, seconds, secs |
| Ampere | amp, amps |
| Celsius (Degree) | degC, celcius |
| Fahrenheit | degF |
| Ohm | ohm, Ω (Unicode) |
| Micro (prefix) | µ (U+00B5), μ (U+03BC) — NOT `u`; `u` is AtomicMassUnit |
| LightYear | ly, lyr, light-year, lightyear, lt-yr |
| Parsec | pc |
| AstronomicalUnit | AU, au |
| Year | yr, year, years (no single-letter alias — `y` is Yocto prefix) |
| Minute | min, minutes |
| Hour | hr, hour, hours (bare `h` = Hour standalone; Hecto needs a companion atom) |
| Day | day, days (bare `d` = Day standalone; Deci needs a companion atom) |
| SolarMass | M_sun, M_⊙, M_☉, Msun |
| ElectronMass | m_e, me |
| Hour | hr, hour, hours |
| Day | d (only when unambiguous; see § 6.3) |
| Minute | min, minutes |
| Percent | % |
| PerMille | ‰, permille |
| AtomicMassUnit | u, Da, dalton — `u` is the canonical AMU alias; Micro prefix must use `μ`/`µ` |

### 6.3 Collision audit rules

**Zero-tolerance policy:** before catalog freeze, a comprehensive
audit confirms no ambiguity exists. The audit tests (run in CI):

1. **Uniqueness:** for every alias string `s` in the alias table,
   `lookup_alias(s)` returns exactly one `UnitId`.
2. **Prefix vs. atom:** for every 1-character and 2-character alias
   `s`, if `s` also equals an `SiPrefix::symbol()`, the atom-
   interpretation takes precedence (i.e., the atom is also a direct
   catalog entry).
3. **Prefix + atom uniqueness:** for every token `t` of length ≥ 2, at
   most one decomposition `(prefix, atom)` is valid; if multiple,
   catalog must either (a) list one of them as a direct canonical
   alias (resolving by precedence rule) or (b) reject the token (no
   such unit accepted; consumer must use unambiguous form).
4. **Case-sensitivity:** `m` and `M`, `g` and `G`, `t` and `T` resolve
   to different units or prefixes. Catalog audit enforces case-sensitive
   uniqueness.
5. **Constant namespace separation:** a bare ASCII letter that matches
   both a unit symbol and a constant symbol is always resolved to the
   unit in *unit-annotation context*. The constant interpretation is
   valid only in *main-expression context*, where unitalg's tokenizer
   is not invoked. (Example: `G` is Gauss when in a unit expression,
   `G` is the gravitational constant when in a main expression
   variable — or `G_N` / `G` with constants-namespace flag.)

**Known collision resolutions (decided in spec, to be validated by audit):**

| Token | Resolves to | Rationale |
|---|---|---|
| `T` | Tesla | Case-sensitive; bare standalone = atom |
| `TT` | (Tera, Tesla) | Prefix decomp; no direct alias |
| `G` | Gauss (in unit context) | Bare atom takes precedence over Giga prefix |
| `Gm` | (Giga, Meter) | No direct alias `Gm`; decomp succeeds |
| `ms` | (Milli, Second) | Listed as direct alias (precedence over Meter·Second juxtaposition) |
| `m` | Meter | Bare, case-sensitive; Milli prefix needs a companion atom |
| `h` | Hour | Bare atom; Hecto prefix needs companion atom |
| `hm` | (Hecto, Meter) | Prefix decomp |
| `d` | Day | Bare atom (admitted only in time contexts; see § 6.4) |
| `cm` | (Centi, Meter) | Prefix decomp; Centimeter is NOT its own UnitId (dropped with CGS system simplification) |
| `c` | (Centi, ???) invalid alone; reserved as constant in main-expr context | Prefixes never standalone. `c` = speed-of-light in main expression; `c` bare in unit annotation is an error. |
| `e` | Reserved as electron-charge constant (main-expr); not a unit symbol | The letter is not a unit. Eulerian `e` lives in constants namespace, referenced as needed. |
| `rad` | Radian (unit) | Radiation absorbed-dose `rad` dropped from catalog to prevent collision; Gray (Gy) replaces it |
| `rem` | (undefined, reject) | Rem dropped from catalog (obsolete, Sievert replaces); token rejected in unit context |
| `u` | AtomicMassUnit | Direct alias for AMU. Micro prefix requires `μ` (U+03BC) or `µ` (U+00B5); ASCII `u` never resolves to Micro |
| `P` | Poise | Bare atom; Peta prefix needs companion |
| `R` | Roentgen | Bare atom; Ronna prefix needs companion (10²⁷ scale absurd in practice) |
| `ct` | Carat | Direct alias; centi-tonne decomp absurd (10 kg scale never encountered) |
| `gr` | Grain (imperial) | Direct alias; NEVER resolves to Gram |
| `g` | Gram | Bare atom; prefix target for sub-kg masses |

### 6.4 Prefix admissibility per unit

Not every unit accepts every prefix. The catalog encodes `applies_prefix`:

- `SiDecimal`: all SI decimal prefixes accepted (SI base + most SI
  derived)
- `SiDecimalRestricted`: only power-of-3 prefixes (engineering-style).
  Optional flag for stricter tokenization. Default is `SiDecimal`.
- `None`: no prefixes (Hour, Day, Year, AU, lightyear, Parsec,
  Degrees-of-angle, Celsius/Fahrenheit/Rankine, dB family)

Rationale for `None` on time and angle units: `km` is unambiguous, but
`ksec` or `Gday` are nonsensical; prefix ambiguity grows catastrophic
if allowed ( `min` = minute, not "milli-in"). Prefixes for time
measure by other unit choice (ms = millisecond; there is no "kilo-
second" even though SI permits it — prefer "hour" or "day" instead).

---

## 7. Wire format summary

All serde-capable types use `#[serde(tag = "kind", content = "value")]`.
Wire format is stable for v0.x.

Example document (Expression + unit annotation, wire format):
```json
{
  "expression": { ... mathlex Expression JSON ... },
  "annotations": {
    "x": {
      "unit": {
        "kind": "Binary",
        "value": {
          "op": {"kind": "Div"},
          "left": {"kind": "Atom", "value": {"id": "Meter", "prefix": null}},
          "right": {"kind": "Atom", "value": {"id": "Second", "prefix": null}}
        }
      }
    }
  }
}
```

---

## 8. Crate layout

```
mathcore-units/
├── Cargo.toml
├── LICENSE
├── README.md
├── CLAUDE.md
├── SPECIFICATION.md                  # this file
├── src/
│   ├── lib.rs                        # public re-exports + crate docs
│   ├── dimension.rs                  # BaseDim, Dimension
│   ├── system.rs                     # System
│   ├── unit_id.rs                    # UnitId (full enum)
│   ├── prefix.rs                     # SiPrefix + PrefixPolicy
│   ├── conversion.rs                 # Conversion, ConstantId
│   ├── unit_expression.rs            # UnitExpression, BinaryOp, UnaryOp, LogOp
│   ├── catalog/
│   │   ├── mod.rs                    # master lookup functions
│   │   ├── si_base.rs                # 7 SI base + radian
│   │   ├── si_derived.rs             # Hz, N, J, W, V, …, °C, liter, minute, hour, …
│   │   ├── si_legacy.rs              # Gram, Dyne, Erg, Gauss, Poise, Stokes (CGS heritage)
│   │   ├── imperial.rs               # UK default + US variants
│   │   ├── logarithmic.rs            # dB family + Neper
│   │   ├── constants_as_units.rs     # SolarMass, ElectronMass, …
│   │   └── dimensionless.rs          # Count, %, ‰, ppm, ppb, sr
│   └── alias.rs                      # alias + prefix decomposition
└── tests/
    ├── alias_collision_audit.rs      # CI-gated: zero ambiguity
    ├── serde_roundtrip.rs
    └── catalog_coverage.rs           # all UnitId have a UnitSpec
```

---

## 9. Versioning

- v0.1.0 ships with the types defined here + the full catalog.
- Minor versions: additions (new UnitId variants, new aliases, new
  ConstantId variants).
- Breaking change: any variant rename, variant removal, wire-format
  change. Requires major bump coordinated with thales.
- Feature flags:
  - `default = ["std"]`
  - `std`: enables `std::collections::BTreeMap`, `String`, etc.
    (otherwise `alloc`-only.)
  - `serde`: enables serde derives.

---

## 10. Requirements summary (MU-1..MU-N)

| ID | Requirement | Severity |
|---|---|---|
| MU-1 | 8-dimension BaseDim enum (7 SI + angle); solid_angle NOT a base dim | Blocker |
| MU-2 | Dimension as sparse BTreeMap<BaseDim, i32> with zero-omission | Blocker |
| MU-3 | 3-entry System enum (SI / Imperial / None); MKS, CGS, Natural, Planck not as Systems | Blocker |
| MU-4 | Closed UnitId enum with tag-and-content serde format | Blocker |
| MU-5 | Catalog: SI base + SI derived named + SI-derived additional + SI-derived legacy (CGS heritage) + Imperial (UK default + US variants) + logarithmic + constants-as-units + dimensionless | Blocker |
| MU-6 | Conversion enum with exactly 4 variants: Linear / Affine / Logarithmic / FromConstant | Blocker |
| MU-7 | ConstantId enum (ID-only, values in mathcore-constants) | Blocker |
| MU-8 | UnitExpression parallel mini-AST with restricted operators (no trig, no calculus, no relation); `+` and `−` carry dimension-unification semantics, not pointwise arithmetic | Blocker |
| MU-9 | Exhaustive alias + prefix-decomposition audit with zero ambiguity, CI-enforced | Blocker |
| MU-10 | Case-sensitive tokenization; single-letter bare symbols always atoms (T = Tesla, G = Gauss); prefixes compose only with a companion atom | Blocker |
| MU-11 | serde wire format stable for v0.x; golden fixtures per variant | Required |
| MU-12 | `no_std + alloc` support behind feature flag | Required |
| MU-13 | Crate publishes with no logic — only data + types + lookup tables | Required |
| MU-14 | Documentation coverage: README + WIRE-FORMAT + this SPECIFICATION | Required |
| MU-15 | Unicode glyph aliases always-on (µ, Ω, ⊙, ⊕, ☾, etc.); no feature flag | Required |
| MU-16 | Imperial volume units: UK default, US explicit variants (`Gallon` = UK, `GallonUS` = US; same pattern for pint/quart/floz) | Required |

---

## 11. Resolved decisions

Decisions confirmed during spec review (2026-04-22):

1. **Solid angle dropped** as a base dimension. Steradian remains as
   a dimensionless named unit in the catalog.
2. **Infotech dropped entirely.** No bit / byte / flops / ips / MTBF
   in v0.1.0. No digital-SNR dB variants either — dB family keeps
   physical-reference variants (dBm, dBW, dB SPL, Neper) only.
3. **`SiDecimalRestricted` ships** as an opt-in PrefixPolicy value.
   Default remains `SiDecimal`. Style-strict consumers can select
   power-of-3-only prefix admission.
4. **Unicode aliases always-on.** No feature flag for alias availability.
5. **`Conversion::Expression` dropped.** Conversion enum closes at 4
   variants. Escape hatch reintroduced in a later minor version if a
   real use case surfaces.
6. **CGS dropped as a System.** Still-relevant units (Gram, Dyne,
   Erg, Poise, Stokes, Gauss) retained as SI-derived legacy entries.
   Obsolete EM variants (StatVolt / StatCoulomb / StatAmpere /
   Oersted / Maxwell / abampere-abvolt) removed from UnitId entirely.
7. **Imperial = UK default, US explicit.** Length and mass UK = US
   (post-1959 international agreement); volume UK default with US
   variants as separate UnitIds.
8. **MKS, Natural, Planck dropped as Systems.** MKS was pre-SI,
   absorbed. Natural's only practical-use unit (eV) is an SI-derived
   energy entry. Planck units are deferred to a future catalog
   addition, not v0.1.0 scope.
9. **Rad (radiation absorbed dose) dropped.** Alias `rad` collides with
   Radian. Gray (Gy, SI) covers the same quantity. Obsolete.
10. **Rem dropped.** Sievert (Sv, SI) replaces. Obsolete.
11. **Micro prefix requires `μ` or `µ`.** ASCII `u` is AtomicMassUnit
    (AMU) only. Plain-text Micro input uses `μ`/`µ` directly, or the
    user spells the composite unit (e.g., `micrometer`).
12. **`gr` = Grain (imperial) only.** Never accepted as Gram
    abbreviation. Gram is `g`.
13. **Carat added.** Metric carat (ISO), 200 mg exactly, alias `ct`.
    No collision — centi-tonne decomposition absurd in practice.

All v0.1.0 design questions resolved. Catalog freeze requires the
exhaustive alias-and-prefix collision audit (CI-enforced, per MU-9).
Ready for implementation.
