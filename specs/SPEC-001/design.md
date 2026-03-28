# SPEC-001: Core Engine — Design

## Signatures

### Files Created or Modified

- `crates/golf-core/src/lib.rs` — crate root, public re-exports
- `crates/golf-core/src/grid.rs` — `NdGrid<N>` struct and indexing logic
- `crates/golf-core/src/cell.rs` — `Cell` type alias, age semantics
- `crates/golf-core/src/rules.rs` — `RuleSet`, `NeighborhoodKind`, named presets
- `crates/golf-core/src/neighbors.rs` — neighbor offset generation and counting for N dimensions
- `crates/golf-core/src/edge.rs` — `EdgeBehavior` enum, coordinate wrapping logic
- `crates/golf-core/src/engine.rs` — `SimulationEngine<N>`, step loop, entropy injection
- `crates/golf-core/src/event.rs` — `EntropyEvent` enum (consumed from golf-entropy in later specs, defined here for the engine interface)
- `crates/golf-core/src/preset.rs` — named rule presets (conway, highlife, day-night, 3d-life)
- `crates/golf-core/Cargo.toml` — crate manifest, no external dependencies beyond std

### Types Introduced

```rust
/// Cell state: 0 = dead, >0 = age (generation count since birth)
pub type Cell = u16;

/// N-dimensional grid backed by a flat buffer.
pub struct NdGrid<const N: usize> {
    extents: [usize; N],    // size per dimension
    strides: [usize; N],    // precomputed stride per dimension
    front: Vec<Cell>,       // current read buffer
    back: Vec<Cell>,        // write buffer (swapped after step)
}

/// How edges behave, per dimension.
pub enum EdgeBehavior {
    Toroidal,  // wraps around
    Bounded,   // dead border (neighbors outside grid count as dead)
}

/// Neighborhood topology.
pub enum NeighborhoodKind {
    Moore,        // all 3^N - 1 adjacent cells
    VonNeumann,   // 2N axis-aligned neighbors
}

/// Set-theoretic rule definition.
pub struct RuleSet {
    pub birth: BTreeSet<u32>,
    pub survival: BTreeSet<u32>,
    pub neighborhood: NeighborhoodKind,
}

/// Events injected between simulation steps.
pub enum EntropyEvent<const N: usize> {
    SetCells { coords: Vec<[usize; N]>, state: Cell },
    ClearRegion { origin: [usize; N], extents: [usize; N] },
    MutateRules(RuleSet),
}

/// Named rule presets.
pub enum Preset {
    Conway,      // B3/S23 2D Moore
    HighLife,    // B36/S23 2D Moore
    DayNight,    // B3678/S34678 2D Moore
    ThreeDLife,  // B5/S45 3D Moore
    Custom(RuleSet),
}

/// Top-level simulation engine.
pub struct SimulationEngine<const N: usize> {
    grid: NdGrid<N>,
    rules: RuleSet,
    edges: [EdgeBehavior; N],
    neighbor_offsets: Vec<[isize; N]>,  // precomputed at construction
    generation: u64,
}
```

### Functions / Traits / Contracts

```rust
// --- NdGrid<N> ---
impl<const N: usize> NdGrid<N> {
    /// Create a new grid with the given extents (all cells dead).
    pub fn new(extents: [usize; N]) -> Self;

    /// Total cell count (product of all extents).
    pub fn len(&self) -> usize;

    /// Read cell at N-dimensional coordinate.
    pub fn get(&self, coord: [usize; N]) -> Cell;

    /// Write cell at N-dimensional coordinate into the back buffer.
    pub fn set_back(&mut self, coord: [usize; N], value: Cell);

    /// Swap front and back buffers (pointer swap, zero copy).
    pub fn swap(&mut self);

    /// Flat index from N-dimensional coordinate.
    fn flat_index(&self, coord: [usize; N]) -> usize;

    /// Read the front buffer as a slice (for rendering).
    pub fn cells(&self) -> &[Cell];

    /// Grid extents.
    pub fn extents(&self) -> [usize; N];
}

// --- RuleSet ---
impl RuleSet {
    pub fn from_preset(preset: Preset) -> Self;
}

// --- Neighbor offset generation ---
/// Generate all neighbor offsets for the given neighborhood kind in N dimensions.
pub fn neighbor_offsets<const N: usize>(kind: &NeighborhoodKind) -> Vec<[isize; N]>;

/// Count live neighbors at a coordinate, respecting edge behavior.
pub fn count_neighbors<const N: usize>(
    grid: &NdGrid<N>,
    coord: [usize; N],
    offsets: &[[isize; N]],
    edges: &[EdgeBehavior; N],
) -> u32;

// --- SimulationEngine<N> ---
impl<const N: usize> SimulationEngine<N> {
    /// Construct engine with grid extents, rules, and edge behaviors.
    pub fn new(
        extents: [usize; N],
        rules: RuleSet,
        edges: [EdgeBehavior; N],
    ) -> Self;

    /// Inject entropy events. Applied immediately to the front buffer.
    pub fn inject_events(&mut self, events: impl IntoIterator<Item = EntropyEvent<N>>);

    /// Advance the simulation by one generation.
    /// Reads from front buffer, writes to back buffer, swaps.
    pub fn step(&mut self);

    /// Current grid state (read-only reference for rendering).
    pub fn grid(&self) -> &NdGrid<N>;

    /// Current generation counter.
    pub fn generation(&self) -> u64;

    /// Current rule set.
    pub fn rules(&self) -> &RuleSet;

    /// Replace the rule set (e.g., from MutateRules event).
    pub fn set_rules(&mut self, rules: RuleSet);
}
```

## Architecture

### Double-Buffer Strategy

The grid maintains two flat `Vec<Cell>` buffers of identical size. During `step()`:

1. Drain any pending `EntropyEvent`s and apply them to the front buffer.
2. For each cell in the front buffer, count live neighbors using precomputed offsets.
3. Apply birth/survival rules to determine the cell's next state.
4. Write the result to the corresponding position in the back buffer, updating age:
   - Dead cell with `birth` neighbor count -> `Cell = 1` (newborn).
   - Live cell with `survival` neighbor count -> `Cell = age + 1` (survives).
   - Otherwise -> `Cell = 0` (dead).
5. Swap front and back buffer pointers (`std::mem::swap`).
6. Increment generation counter.

This achieves NFR-2 (zero heap allocations in steady state) because both buffers are pre-allocated and never resized.

### Flat Buffer Indexing

An N-dimensional coordinate `[x0, x1, ..., x_{N-1}]` maps to a flat index via stride multiplication:

```
index = x0 * stride[0] + x1 * stride[1] + ... + x_{N-1} * stride[N-1]
```

Strides are precomputed at grid construction. For extents `[E0, E1, ..., E_{N-1}]`:

```
stride[N-1] = 1
stride[i]   = stride[i+1] * E_{i+1}   (for i = N-2 down to 0)
```

This is row-major order. The last dimension varies fastest, which gives good cache locality for 2D grids where the last dimension is the column (horizontal scan).

### Neighbor Offset Precomputation

At engine construction, all neighbor offsets are generated once and stored in a `Vec<[isize; N]>`.

**Moore neighborhood (N dims):** Iterate all combinations of `{-1, 0, 1}` for each of N dimensions. Exclude the origin `[0, 0, ..., 0]`. This produces `3^N - 1` offsets. For N=2 that is 8; for N=3 that is 26.

**Von Neumann neighborhood (N dims):** For each dimension `d`, produce two offsets: one with `+1` in dimension `d` and all others `0`, one with `-1` in dimension `d`. This produces `2N` offsets. For N=2 that is 4; for N=3 that is 6.

The offset generation uses a recursive or iterative cartesian-product approach over `{-1, 0, 1}^N`.

### Edge Behavior

Each dimension has an independent `EdgeBehavior`:

- **Toroidal:** When a neighbor offset would push a coordinate below 0 or above `extent - 1`, wrap using modular arithmetic: `(coord as isize + offset).rem_euclid(extent as isize) as usize`.
- **Bounded:** If any dimension's resulting coordinate is out of bounds, that neighbor is counted as dead (skipped).

The per-dimension independence means you can have a grid that wraps horizontally but has dead borders vertically.

### EntropyEvent Injection

Events are applied between steps, directly to the front buffer before the next step reads from it. The engine exposes `inject_events()` which processes events in order:

- `SetCells`: write the given `Cell` value at each coordinate.
- `ClearRegion`: set all cells within the bounding box to 0.
- `MutateRules`: replace the active `RuleSet` and recompute neighbor offsets.

This injection point is the interface for SPEC-003 (Entropy Trait). The entropy system will produce `EntropyEvent`s and the engine consumes them through this method.

## Data Model

```
Cell (u16)
  0     = dead
  1     = newborn (just birthed this generation)
  2+    = alive for N generations (age = value)
  65535 = maximum trackable age (saturating add)

NdGrid<N>
  extents: [usize; N]     -- e.g., [200, 50] for a 2D terminal grid
  strides: [usize; N]     -- precomputed for O(N) index calculation
  front: Vec<Cell>         -- len = product(extents)
  back: Vec<Cell>          -- same length, swapped after step

SimulationEngine<N>
  grid: NdGrid<N>
  rules: RuleSet
  edges: [EdgeBehavior; N]
  neighbor_offsets: Vec<[isize; N]>
  generation: u64
```

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| `3^N - 1` neighbor count explodes for large N (N=10 means 59,048 neighbors per cell) | Performance degrades beyond usability | Medium — users may try high N out of curiosity | Document practical N limits. Consider a compile-time or runtime cap (e.g., N <= 6). Log a warning for N > 4. |
| Const generic `N` requires monomorphization per dimension count, increasing binary size | Larger binary, longer compile times | Low — most users will use N=2 or N=3 | Provide type aliases `Grid2D = NdGrid<2>`, `Grid3D = NdGrid<3>`. Only monomorphize what is used. |
| `u16` cell age saturates at 65,535 generations | Cells that survive longer than 65,535 gens show incorrect age | Very low — most cells die well before this | Use `saturating_add`. Document the limit. u16 is sufficient for visual heatmaps. |
| Flat buffer for large 3D+ grids may exceed available memory | OOM for large extents in high dimensions | Medium | Validate total cell count at construction. Reject grids exceeding a configurable memory limit (default: 1 GiB). |
| No `unsafe` constraint may limit performance optimization options | Cannot use SIMD intrinsics or unchecked indexing in hot path | Low — bounds check elimination by the compiler is effective for sequential access patterns | Profile first. The compiler eliminates most bounds checks when iterating over known-length slices. Revisit if NFR-1 benchmarks fail. |

## ADR-001: Flat Buffer over HashMap for Grid Storage

**Status:** Accepted

**Context:** N-dimensional grids can be stored as a dense flat buffer (`Vec<Cell>`) or a sparse map (`HashMap<[usize; N], Cell>`). Sparse storage is memory-efficient for low-density grids but has high per-cell overhead for neighbor lookups.

**Decision:** Use a dense flat `Vec<Cell>` with stride-based indexing.

**Consequences:**
- Memory usage is proportional to total grid volume regardless of live cell count.
- Cache-friendly sequential access during step iteration.
- Zero-cost index computation (multiply-add chain).
- Sparse grids (e.g., a few gliders in a 1000x1000 grid) waste memory, but the target grid sizes (200x50 2D, 50x50x50 3D) are small enough that this is acceptable.
- If very large sparse simulations become a goal, a HashLife-style approach would be a separate spec.

## ADR-002: u16 Cell State over bool or enum

**Status:** Accepted

**Context:** Cell state could be `bool` (alive/dead), an enum (`Dead | Alive(u32)`), or a numeric type encoding age directly.

**Decision:** `type Cell = u16` where 0 = dead, >0 = age.

**Consequences:**
- 2 bytes per cell (vs 1 byte for bool, 8 bytes for enum with u32 age due to alignment).
- Age tracking is free — no separate data structure.
- Alive check is `cell > 0`, which is a single comparison.
- Maximum age of 65,535 is more than sufficient for rendering heatmaps.
- Slightly more memory than bool (2x), but the total for a 200x50 grid is 20 KiB — negligible.

## ADR-003: Const Generic N over Runtime Dimensionality

**Status:** Accepted

**Context:** Dimensionality could be a runtime parameter (`struct Grid { dims: usize, ... }`) or a compile-time const generic (`struct NdGrid<const N: usize>`).

**Decision:** Const generic `N`.

**Consequences:**
- The compiler can unroll neighbor iteration loops for known N, enabling vectorization.
- Coordinate types are `[usize; N]` and `[isize; N]` — stack-allocated, no heap.
- Each distinct N produces a separate monomorphization. Binary size grows linearly with the number of N values used.
- Cannot change dimensionality at runtime without dispatching through a trait object or enum. The CLI will need a match on the user-provided dimension count to instantiate the correct `SimulationEngine<N>`.
- For the target use case (N=2 default, N=3 occasionally, N=4+ rare), this trade-off strongly favors compile-time generics.
