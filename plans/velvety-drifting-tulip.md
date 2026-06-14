# Generic conflation MVT tile endpoint in saturnapi

## Context

Conflation pipelines (e.g. `polylines_conflation`) emit a **`FeaturesWithAGeometry`**
descriptor: one hilbert-sorted parquet per lookup partition (a county/region), holding
WKB `geometry-N` columns, per-row-group bbox stat columns
(`geometry-N_minx/miny/maxx/maxy` in the descriptor's pixel CRS), a `geometry-N_tiles`
column, and feature attributes such as `POLYLINES_CONFLATION_METADATA`.

Today there is no way to view this output as a slippy-map layer:
- `permit_explorer` solves a similar problem but is a **single-task, ingest-on-startup,
  bespoke** service with its own DuckDB index and deck.gl frontend — too heavy and not generic.
- `saturnapi` already serves MVT z/x/y tiles, but only for **pre-tiled** descriptors
  (`TiledFeatures`, `FeatureTiles`) via `features.get_vector_tile`
  (`services/saturnapi/saturnapi/features/__init__.py:18`). It rejects `FeaturesWithAGeometry`,
  and its `z > desc.zoom` guard (line 43) assumes `desc.zoom` is the native data zoom.
  For conflation output `desc.zoom` is `TRUE_WORLD_TILE_ZOOM` (the pixel-CRS reference zoom),
  so that guard would reject normal map zooms.

**Goal:** add a generic, on-demand MVT endpoint to `saturnapi` that tiles **any
`FeaturesWithAGeometry` (conflation) task** by `task_id` + `descriptor_name` +
`lookup_index` + feature/geometry class + `z/x/y`, reading geometry straight from the
per-lookup parquet with row-group bbox pruning. An external map frontend (the portal that
already consumes `/saturn/v0`) wires it in as a vector-tile layer. No frontend work and no
startup ingest in this repo.

Decisions confirmed with the user: **MVT vector tiles** · **backend-only (external portal
consumes it)** · **explicit `lookup_index`** (caller names the partition; server does not
geo-resolve partitions).

## Approach

On-demand, mirroring the existing `get_merged_features_tile` transform math
(`features/merged_feature_tiles.py:66`) but sourcing geometry from the
`FeaturesWithAGeometry` parquet instead of pre-merged tiles. Three code touch-points plus tests.

### 1. New module: `services/saturnapi/saturnapi/features/geometry_feature_tiles.py`

`get_geometry_features_tile(lookup_index, x, y, z, feature_class, geometry_class, prefix, md, desc, file_client, lines=False, max_features=...) -> bytes`

- Resolve `feature_meta = desc.class_metadata(feature_class.id)`; validate the layer
  `IdStrings.to_layer(feature_class.id, geometry_class.id)` exists in the descriptor's classes.
- Compute, exactly as the merged tiler does:
  `scale = 2.0 ** (z - desc.zoom)`, `offset = np.array([x * desc.tile_size, y * desc.tile_size])`.
  The request tile's footprint in **global pixel coords** (the CRS of the bbox stat columns) is
  `gx in [x*desc.tile_size/scale, (x+1)*desc.tile_size/scale]` (same for `gy`); add a 1-px buffer
  so geometries straddling the tile edge aren't dropped before clipping.
- Build a row-group-pruning predicate on the bounds columns from
  `IdStrings.to_geometry_bounds_columns(geometry_class.id)` →
  `(maxx >= gx_min) & (minx <= gx_max) & (maxy >= gy_min) & (miny <= gy_max) & (minx IS NOT NULL)`.
- Read with **row-group pushdown**, selecting only `hilbert`, the `geometry-N` WKB column, the
  bounds columns, and the attribute columns of `feature_meta`. Prefer the polars path
  `read_features_gpl` / `read_features_pl` (`saturnlib.features.features`, documented as preferred);
  the bbox predicate goes through `pl.scan_parquet` filtering so PyArrow/polars skips
  non-overlapping row groups (the parquet carries min/max stats on these columns). Apply a
  `LIMIT max_features` to bound payload size.
- For each row: WKB → shapely, `shapely.set_precision(shapely.transform(geom, lambda p: p * scale - offset), 1)`,
  skip zero-length collapsed geoms when `z != desc.zoom` (same guard as merged tiler line 123),
  `tile.add_geometry(layer, geom, props)` on a `cppaiutils.mvt.Writer(desc.tile_size, enforce_valid_mask=False)`.
  Optional `_lines` layer via `tile_poly_to_lines` when `lines=True` (reuse `..utils`).
- **Properties (generic + conflation-aware):** a small `_row_to_mvt_props(row)` helper that emits
  `strHilbert`/`hilbert` plus every attribute column as an MVT-safe scalar (numbers/strings/bools
  pass through; struct/list columns such as `POLYLINES_CONFLATION_METADATA.sources` are
  JSON-encoded, while its scalar subfields `agreement`, `region_count`, `quality_score`,
  `derived_from_firstparty`, `derived_from_thirdparty` are flattened to top-level props). Keeps the
  endpoint generic across `FeaturesWithAGeometry` tasks while surfacing conflation fields directly.
- `return tile.to_bytes()`.

Reuse the existing low-zoom strategy decision: start with **SOURCE geometries + `max_features`
cap at every zoom** (simplest, correct). Note as a follow-up the option to collapse to hilbert
centroid points / cluster boxes below a zoom threshold (the `ReturnType.from_z` +
`hilberts_to_px_points` pattern from `merged_feature_tiles.py`) if low-zoom payloads prove heavy.

### 2. New dispatcher entry in `services/saturnapi/saturnapi/features/__init__.py`

Add `get_geometry_vector_tile(uid, descriptor_name, lookup_index, z, x, y, feature_class, geometry_class, file_client, lines=False, max_features=...)`:
- Same metadata/tile-mask setup as `get_vector_tile` (lines 33–36): `get_task_metadata`,
  set single-lookup `tile_mask` via `get_lookup_tile_mask`, `md.descriptor(descriptor_name)`,
  `md.descriptor_prefix(...)`.
- Assert `isinstance(desc, FeaturesWithAGeometry | SpatioTemporalDataset)` else `ValueError`.
- **Coverage cull, not zoom guard:** use `md.is_valid_tile(x, y, z, lookup_index)` to early-return an
  empty tile (`Writer(desc.tile_size, enforce_valid_mask=False).to_bytes()`) when the tile is
  outside coverage. **Do not** apply the `z > desc.zoom` rejection — it's wrong for this descriptor's
  zoom semantics.
- Delegate to `get_geometry_features_tile`. Export it in `__all__`.

### 3. New route in `services/saturnapi/saturnapi/app.py`

Add under a new `tags=["Conflation"]` (sits alongside the `task_vector_tile` route, line 309):

```
@app.get(PREFIX + "task/{task_id}/conflation/{descriptor_name}/{lookup_index}/{z}/{x}/{y}/{feature_class_id}/{geometry_class_id}", tags=["Conflation"])
@common_exception_handler
async def task_conflation_vector_tile(...):
    tile_bytes = await asyncio.to_thread(features.get_geometry_vector_tile, ...,
        F.allow_missing(feature_class_id), G.allow_missing(geometry_class_id),
        file_client_with_cache, lines, max_features)
    return StreamingResponse(io.BytesIO(tile_bytes), media_type=MEDIA_TYPE_LOOKUP[FeatureExt.MVT])
```

Reuse `UrlParams` (`task_id`, `descriptor_name`, `lookup_index`, `z`, `x`, `y`,
`feature_class_id`, `geometry_class_id`, `lines`) from `entities.py`. Add one `EXAMPLE_*`
`TileExample` for a known conflation task and a `max_features` `UrlParams` helper (default e.g. 5000,
modeled on `UrlParams.max_points`). `descriptor_name` example list: `[FeaturesWithAGeometry, SpatioTemporalDataset]`.

## Critical files

| File | Change |
|---|---|
| `services/saturnapi/saturnapi/features/geometry_feature_tiles.py` | **new** — `get_geometry_features_tile` + `_row_to_mvt_props` |
| `services/saturnapi/saturnapi/features/__init__.py` | add `get_geometry_vector_tile` dispatcher + export |
| `services/saturnapi/saturnapi/app.py` | add `task_conflation_vector_tile` route |
| `services/saturnapi/saturnapi/entities.py` | add `max_features` UrlParam + conflation `TileExample` |
| `services/saturnapi/tests/test_conflation_tiles.py` | **new** — mirror `tests/test_tiled_features.py` |

## Reused building blocks (do not reinvent)

- Transform math + Writer usage: `features/merged_feature_tiles.py:66` (`get_merged_features_tile`).
- Bounds column names: `aiops_classes.IdStrings.to_geometry_bounds_columns(geometry_class.id)`.
- Layer name: `IdStrings.to_layer(feature_class.id, geometry_class.id)`.
- Row-group-pruned reads: `read_features_gpl` / `read_features_pl` (`saturnlib.features.features`).
- Metadata/tile-mask/prefix: `metadata.get_task_metadata`, `get_lookup_tile_mask`,
  `md.descriptor`, `md.descriptor_prefix`, `md.is_valid_tile`.
- Polygon outlines: `..utils.tile_poly_to_lines`.
- MVT writer: `cppaiutils.mvt.Writer`; media type `MEDIA_TYPE_LOOKUP[FeatureExt.MVT]`.

## Verification

1. **Unit test** (`tests/test_conflation_tiles.py`): build a small synthetic
   `FeaturesWithAGeometry` task in a `DictClient` (follow `tests/test_tiled_features.py` +
   `saturnlib.testing` fixtures) with a couple of polyline geometries carrying a
   `POLYLINES_CONFLATION_METADATA` attribute; assert the endpoint returns non-empty MVT bytes for a
   covering tile, an empty (header-only) tile for an out-of-coverage tile, and that decoded MVT
   props include flattened `agreement`/`quality_score`. Run `pytest services/saturnapi/tests/test_conflation_tiles.py -x`.
2. **Live smoke**: `bash services/saturnapi/dev_server.sh`, then
   `curl -o /tmp/t.mvt "http://localhost:8888/saturn/v0/task/<conflation_task_id>/conflation/<descriptor>/<lookup>/<z>/<x>/<y>/<fclass>/<gclass>"`
   against a real `polylines_conflation` task (dev bucket); confirm 200 + MVT bytes, and decode/inspect
   in a scratch script (`mapbox_vector_tile` / `cppaiutils.mvt`).
3. **Frontend**: hand the route template to the portal team to register as a MapLibre vector source /
   deck.gl `MVTLayer` (out of scope here).
4. `bash .gitlab/run_style.sh -i yes` before finishing.

## Open follow-ups (not in this change)

- Low-zoom point/cluster degradation if SOURCE payloads are too large.
- Optional geo-resolved variant (server picks `lookup_index` from the conflation lookup
  GeoDataFrame via `coarse_partition_ids_from_geom_*`) if callers shouldn't need to know partitions.
