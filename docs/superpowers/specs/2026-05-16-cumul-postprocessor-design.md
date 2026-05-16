# Cumul post-processor — design spec

**Status**: Draft — awaiting user review
**Author**: cedric@libeo.io
**Date**: 2026-05-16

## Goal

Add precipitation-accumulation variables to `.om` spatial files produced by the Open-Meteo pipeline, so they appear in the Infoclimat front (`../modeles-infoclimat/`) as selectable layers.

Open-Meteo's `data_spatial` only exposes the per-step precipitation (hourly increment). Forecasters and users routinely want rolling sums (last 6 h, last 24 h) and the run-total. These variables do not exist upstream.

## Scope — variables added

Twelve new variables, derived from four base variables that already exist in the upstream pipeline:

| Base variable                 | + `_run_total` | + `_6h_sum` | + `_24h_sum` |
|-------------------------------|----------------|-------------|--------------|
| `precipitation`               | ✓              | ✓           | ✓            |
| `rain`                        | ✓              | ✓           | ✓            |
| `showers`                     | ✓              | ✓           | ✓            |
| `snowfall_water_equivalent`   | ✓              | ✓           | ✓            |

**Semantics**, where `var[t]` is the hourly increment at forecast step `t`:

- `<var>_run_total[t] = sum(var[1..t])` — cumul since `t=0` of the run.
- `<var>_6h_sum[t] = sum(var[t-5..t])` if `t ≥ 5`, else NaN.
- `<var>_24h_sum[t] = sum(var[t-23..t])` if `t ≥ 23`, else NaN.

NaN is the chosen behaviour for early timesteps. The front already renders missing data as "no value" — no UI work needed.

## Non-goals (v1)

- **No backfill from previous runs.** Filling early timesteps from the previous run's data is deferred. Trade-off accepted: 24 h sum is undefined for the first 23 forecast hours of every new run.
- **No front-side changes** beyond the existing `getBaseUri()` flip (planned in INFOCLIMAT.md Phase 4). The post-processor must keep the file layout strictly Open-Meteo-compatible.
- **No new variables** beyond the 12 listed.
- **No model-specific logic**: same algorithm for every domain.
- **No reverse migration** of already-produced runs. Cumuls are computed only for new runs as they arrive.

## File format (confirmed)

**Investigation date**: 2026-05-16 — Task 0 spike, Phase-1 run `2026051615` AROME France HD.

### Layout: one `.om` per **timestep**, MULTI-VARIABLE

Each file (`2026-05-16T1600.om`, etc.) is an OmFileFormat v3 file whose root node is a **GROUP** (not an array). Confirmed via Python `omfiles` library (`OmFileReader.is_group == True`, `OmFileReader.is_array == False`). The Group tree for a typical timestep (t > 0) contains:

| Node name | Type | Shape / value |
|-----------|------|---------------|
| `precipitation` | array (float32) | (1791, 2801), chunks (32, 32) |
| `temperature_2m` | array (float32) | (1791, 2801), chunks (32, 32) |
| `crs_wkt` | scalar string | GEOGCRS WKT for WGS 84, bbox 37.5/−12/55.4/16 |
| `forecast_reference_time` | scalar int64 | Unix timestamp of run start |
| `valid_time` | scalar int64 | Unix timestamp of this timestep |
| `coordinates` | scalar string | `"lat lon"` |
| `created_at` | scalar int64 | Unix timestamp of file creation |

The first timestep (`T+0`, i.e. `2026-05-16T1500.om`) only contains `temperature_2m` because `MeteoFranceVariableDownloadable.skipHour0 == true` for precipitation — that is not a format anomaly, it is by design.

### OmFileReader Swift API to enumerate variables

Inside the OmFileFormat Swift package, the `OmFileReader` (v3) exposes a **tree of nodes**. To enumerate named variable children of a group:

```swift
let reader = try await OmFileReader(fn: try MmapFile(fn: FileHandle.openFileReading(file: path)))
// reader.isGroup == true for these spatial files
// reader.children → [OmFileReader]  (one per named node)
// Each child: child.name, child.isArray, child.asArray(of: Float.self) -> OmFileReaderArray
```

The equivalent Python API confirmed by inspection:
`reader.get_child_by_name("precipitation")` — returns the array node.
`reader.get_child_by_index(i)` — returns child by position.

The `asArray(of: Float.self)` on the root fails with `invalidDataType` for group-root files — this is expected. You must descend into the named child first.

### Impact on Task 3 (CumulOmRewriter)

The rewriter **cannot append in-place**. The `OmFileWriter` API only creates fresh files. The correct pattern:

1. Open existing `T.om` for read.
2. Create `T.om~` for write.
3. Copy all existing children (precipitation, temperature_2m, metadata scalars) verbatim.
4. Compute and append derived variables (`precipitation_run_total`, `precipitation_6h_sum`, `precipitation_24h_sum`, …) as new array children.
5. Atomic rename `T.om~ → T.om`.

No design change needed for Tasks 1–2 or 4–7. The "in-place modification" phrasing in the spec above should be understood as "read-then-rewrite-atomically", not an in-place byte-patch.

## Architecture

### New Swift command

`Sources/App/Commands/ComputeCumulsCommand.swift`, registered in `configure.swift` alongside the existing `download-*`, `sync`, etc. commands. Invocation:

```
openmeteo-api compute-cumuls --domain <domain> --run <YYYYMMDDHH>
```

Arguments:

- `--domain <id>` (required) — e.g. `meteofrance_arome_france_hd`, `meteofrance_arpege_europe`, `dwd_icon_d2`.
- `--run <YYYYMMDDHH>` (optional) — defaults to the latest run found on disk for the domain.

The command is **model-agnostic**: it reads whatever variables are present in the `.om` files, computes cumuls only for the base variables it finds, and writes them back.

### Algorithm

For each invocation:

1. **Discover run files.** Glob `<DATA_SPATIAL_DIRECTORY>/<domain>/YYYY/MM/DD/HHMMZ/*.om` for the requested run. Read the timesteps in chronological order.
2. **For each base variable** (`precipitation`, `rain`, `showers`, `snowfall_water_equivalent`):
   - Skip if the variable is absent from the `.om` files (some domains may not produce all four).
   - Load the variable for every timestep into memory. Per timestep, that's one full grid (~37.5 M cells for AROME France HD). Total memory ≈ 52 timesteps × 37.5 M cells × 4 bytes ≈ 7.8 GB per base variable for AROME HD — too much. **Implementation note: stream per latitude band**, not the entire grid at once.
   - For each cell, compute the three derived series: `_run_total`, `_6h_sum`, `_24h_sum`.
3. **Rewrite each timestep's `.om`** atomically (write to a temp file in the same directory, then `rename(2)`): original variables + the new derived variables present.
4. **Update `latest.json`**: extend the `variables[]` array with the names actually written.

### Atomicity

Each `.om` is replaced via temp-file-then-rename. `latest.json` is the last thing written; if the process crashes mid-run, the file's `variables[]` still reflects the previous, partially-cumul'd state. Re-running the command is idempotent: it overwrites with the same content.

### Triggering

In `docker-compose.yml`, the cumul service runs **after** the download service for each domain. Two patterns are acceptable; we'll pick one when writing the plan:

- **Sequential `command`**: a small shell wrapper that runs `openmeteo-api download-X && openmeteo-api compute-cumuls --domain X`.
- **Separate service with `depends_on: condition: service_completed_successfully`**.

### Build implication

Phase 1 used the upstream pre-built image `ghcr.io/open-meteo/open-meteo:latest`. Phase 2 introduces our own Swift code (`ComputeCumulsCommand.swift`), so we must build our image from our `Dockerfile`. First build ≈ 10–30 minutes (Swift release build); subsequent builds incremental.

## Memory & I/O sanity check

AROME France HD has the highest cost (0.01° grid). Numbers below are per run, per base variable, for the surface level:

- Grid: lat 37.5–55.4 × lon −12–16, step 0.01° → ~1790 × 2800 ≈ 5.0 M cells.
- Timesteps observed in Phase 1: 52.
- Naive in-RAM array (all timesteps × all cells × 4-byte float) ≈ 1.0 GB per base variable, 4.0 GB for the four base variables in parallel. Tight on commodity dev hardware; uncomfortable in a CI container.

Implementation should avoid the naive whole-grid-in-RAM approach. Strategies (to be chosen in the plan):

- Stream by latitude band: load all 52 timesteps for one band, compute the three derived series for that band, write, move to the next band.
- Or, since the format already supports chunked random access, work on the native chunk size.

Either way: peak memory should stay well under 1 GB. Sustained I/O on commodity SSD makes total wall time per run < 2 min, comfortable next to the 4-min Phase-1 download.

## Open questions deferred to the plan

- Exact .om read/write API in the OmFileFormat Swift module — to be confirmed by reading `Sources/App/Helper/OmSpatialTimestepWriter.swift`.
- Whether `latest.json` parsing is best done with `JSONDecoder` ad-hoc, or if a shared model already exists in the upstream code.
- Behaviour when the base variable is itself NaN at some cell (e.g. ocean masking): propagate as NaN in derived variables.

## Success criteria

1. After running `docker compose up arome-france-hd` (which chains download + cumuls), `latest.json` lists all original variables plus up to 12 new ones.
2. Each `.om` of the run contains the cumul variables — verified by re-reading via the OmFileFormat tools.
3. For a freshly-produced AROME France HD run, the front (`../modeles-infoclimat/` with `getBaseUri()` pointing at our local server) shows the new cumul variables in its selector and renders them on the map.
4. Re-running `compute-cumuls` on the same run is a no-op (idempotent).
5. No upstream `.om` consumer is broken: a vanilla Open-Meteo client that reads only `precipitation` still works.
