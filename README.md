# SIBR(Small Business Innovation Research) SDF Lattice Generator

Rust CLI/API for generating printable lattice meshes via signed-distance functions. Supports cubic, Kelvin, and BccXy unit cells on cube and cylinder primitives; produces STL/OBJ through a CPU-only marching-cubes pipeline with Taubin smoothing and optional QEM decimation.

**API docs:** <https://anjin-byte.github.io/SIBR_SDF_Lattice_Generator/> (auto-published from `main` via GitHub Actions).
**DEMO:** <https://anjin-byte.github.io/WoodwardFormanLatticeGen/> (WebGPU should be enabled)

## Setup

Requires Rust 1.85+ (2024 edition). From the workspace root:

```sh
cargo build --release
```

## Try it

**Smallest thing that produces output** — cubic lattice in a 10 mm cube, meshes in <1 s:

```sh
cargo run --release -p sibr-lattice -- \
  --primitive cube --half-extents 5,5,5 \
  --cell-topology cubic --cell-length 2 --strut-radius 0.2 \
  --grid-ratio 3 \
  -o out/cubic.stl
```

**Printable Kelvin cylinder** — 17 mm radius × 50 mm tall, ~8 s on one core:

```sh
cargo run --release -p sibr-lattice -- \
  --primitive cylinder \
  --cylinder-start 0,0,0 --cylinder-end 0,0,50 --cylinder-radius 17 \
  --cell-topology kelvin --cell-length 3 --strut-radius 0.3 \
  --grid-ratio 3 --smooth-iterations 15 \
  -o out/kelvin-cyl.stl
```

Defaults to MC33 (Chernyaev 1995 / Lewiner 2003) — topologically correct at every non-degenerate configuration. Pass `--extraction-method classic` for the legacy Lorensen-Cline 1987 algorithm if you need the old behavior.

Raw output is typically hundreds of MB — too big for most slicers. Decimate it:

```sh
cargo run --release -p xtask -- remesh \
  -i out/kelvin-cyl.stl \
  -o out/kelvin-cyl-slim.stl
```

QEM (Garland & Heckbert, via `meshoptimizer`) with a sub-printer-resolution error bound. Topology-preserving; manifold-safe.

For a batch pipeline covering four representative cylinder configs, see [`scripts/build_some_stuff.sh`](scripts/build_some_stuff.sh).

## Flags worth knowing

| Flag | Effect |
|---|---|
| `--cell-topology` | `cubic` \| `kelvin` \| `bccxy` |
| `--grid-ratio N` | Mesh cells per strut radius. 3 is the recommended baseline; higher → finer mesh, bigger file. |
| `--extraction-method` | `mc33` (Chernyaev 1995 / Lewiner 2003, **default**) or `classic` (Lorensen & Cline 1987). MC33 is topologically correct at every non-degenerate configuration — all seven Chernyaev ambiguous cases (3, 4, 6, 7, 10, 12, 13) are fully ported, and unambiguous cases use Lewiner's face-consistent tables. Output of `mc33` is never worse than `classic` on the same input; use `classic` only for differential testing or to reproduce pre-MC33 behavior. |
| `--smooth-iterations N` | Taubin smoothing passes. 10–20 typical; removes voxel artifacts. |
| `--help` | Everything else. |

## Repo layout

- **`crates/sdf`** — SDF primitives and combinators. Type-level exact-vs-bound precision tracking: composing a bound SDF into a position that requires exactness is a compile error.
- **`crates/lattice-gen`** — unit cells, `LatticeJob`, marching cubes, welding, Taubin smoothing, STL/OBJ export. CPU-only; no GPU dependency.
- **`crates/cli`** — the `sibr-lattice` binary. Thin orchestration over `lattice-gen`.
- **`crates/xtask`** — post-processing tools. Currently `remesh` (QEM decimation).

## Tests

```sh
cargo test --workspace
cargo clippy --all-targets --workspace -- -D warnings
```
