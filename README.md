# duckdb-static

Prebuilt **static** DuckDB libraries for KoliStat projects, with the
`parquet` + `json` extensions linked in. Built from DuckDB's own CMake (not
the conan recipe, which breaks the parquet extension on MSVC).

**Why:** DuckDB ships only a DLL for Windows. Linking the static library lets
a consumer ship a single binary (no `duckdb.dll`) and reuse one DuckDB build
across projects. First consumer: [`bedevere-desktop`](https://github.com/KoliStat/bedevere-desktop).

## Artifacts

Each GitHub Release (tag `v<X.Y.Z>-win-x64`) carries
`duckdb-static-v<X.Y.Z>-windows-x64.zip` containing:

- `lib/` — `duckdb_static.lib` (+ any statically-linked extension libs)
- `include/` — the DuckDB C++ public headers (`duckdb.hpp` + tree)

## Building / releasing

**Actions → "Build static DuckDB (Windows)" → Run workflow:**

- `duckdb_ref` — the `duckdb/duckdb` tag. Default **`v1.5.1`**, pinned to
  match stats_duck's ABI and bedevere-desktop's DuckDB. Bump in lockstep.
- `do_release` — check to publish a GitHub Release with the artifact.

The build compiles DuckDB from source (large codebase — expect a long run).

## Consuming (sketch)

Download the release zip and add an imported static target, e.g.:

```cmake
# duckdb_static.lib + include/ extracted to ${DUCKDB_STATIC_DIR}
add_library(DuckDB::Static STATIC IMPORTED)
set_target_properties(DuckDB::Static PROPERTIES
  IMPORTED_LOCATION "${DUCKDB_STATIC_DIR}/lib/duckdb_static.lib"
  INTERFACE_INCLUDE_DIRECTORIES "${DUCKDB_STATIC_DIR}/include")
# Also link any separate extension .libs the artifact ships.
```

> **Status: bring-up.** The CMake flags (`CORE_EXTENSIONS`, the static target
> name, the produced `.lib` / header layout) are being validated against
> DuckDB v1.5.1 — the workflow will evolve over the first runs.
