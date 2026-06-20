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

- `lib/` — `duckdb_static.lib` + `parquet_extension.lib` + `json_extension.lib`,
  plus the third-party static libs (re2, fmt, zstd, fsst, utf8proc, mbedtls,
  yyjson, …). The extensions are **separate** libs: `duckdb_static` does not
  absorb them — in DuckDB's CMake only the *shared* `duckdb` target links
  extensions, so the consumer links the extension libs itself.
- `include/` — the DuckDB C++ public headers (`duckdb.hpp` + tree).

## Building / releasing

**Actions → "Build static DuckDB (Windows)" → Run workflow:**

- `duckdb_ref` — the `duckdb/duckdb` tag. Default **`v1.5.1`**, pinned to
  match stats_duck's ABI and bedevere-desktop's DuckDB. Bump in lockstep.
- `do_release` — check to publish a GitHub Release with the artifact.

The build compiles DuckDB from source (large codebase — expect a long run).

### Build constraints (learned the hard way)

- **Runner pinned to `windows-2022` (MSVC 14.4x).** DuckDB v1.5.1's bundled
  `fmt` uses `stdext::checked_array_iterator`, an MSVC STL extension that the
  VS "18" toolset (MSVC 14.51, now on `windows-latest`) removed. Bump only once
  the pinned `duckdb_ref` ships an `fmt` that compiles on the newer toolset.
- **Extensions are enabled with `-DBUILD_EXTENSIONS`** — the actual CMake cache
  variable. `CORE_EXTENSIONS` is only a *Makefile* alias that maps onto it;
  passing `-DCORE_EXTENSIONS` to `cmake` directly is silently ignored and
  yields a green but core-only lib (no parquet/json). Paired with
  `-DEXTENSION_STATIC_BUILD=1`, and the `parquet_extension` / `json_extension`
  targets are built explicitly.

## Consuming (sketch)

Download the release zip and add imported static targets, e.g.:

```cmake
# lib/ + include/ extracted to ${DUCKDB_STATIC_DIR}
add_library(DuckDB::Static STATIC IMPORTED)
set_target_properties(DuckDB::Static PROPERTIES
  IMPORTED_LOCATION "${DUCKDB_STATIC_DIR}/lib/duckdb_static.lib"
  INTERFACE_INCLUDE_DIRECTORIES "${DUCKDB_STATIC_DIR}/include")
# Link the extension libs + third-party libs the artifact ships too, then
# REGISTER parquet/json at init — they are statically linked, not auto-loaded.
```

> **Status: validated.** First green release is **`v1.5.1-win-x64`** — parquet
> and json linkage confirmed via `dumpbin` on the artifact (ParquetReader /
> ColumnReader / thrift / snappy and JSONCommon / JSONReader symbols present).
