# forsys-agent

**A proof of principle: the ForSys greedy spatial heuristic, executed as DuckDB SQL over an H3 hex grid.**

[ForSys](https://www.forsysplanning.org/) is the US Forest Service's spatial planning platform for
landscape restoration and fuel management. This app reimplements its greedy project-building
heuristic as SQL against cloud parquet — no solver, no R runtime, no precomputed adjacency table —
and wraps it in a [geo-agent](https://github.com/boettiger-lab/geo-agent) map app so a planner can
drive it in plain language.

The default planning area is **Tahoe National Forest**: 6,329 treatment units at H3 resolution 8,
about 1.15M acres. That is squarely inside ForSys's own stated operating range (project level
~1,000 ha up to multi-regional efforts ~1M ha).

The full algorithm write-up — every stage with its SQL, the ForSys→SQL mapping table, verified
numbers, and the honest limits — is in [`about.html`](about.html), which is also the app's **About**
page, linked from the chat footer via `links.docs`.

---

## Why this is feasible (and MILP is not)

ForSys optimizes with, in its own words, *"a relatively simple greedy spatial heuristic instead of
more complicated mathematical programming approaches used in many other planning systems."*

That distinction is the whole argument. Mixed-integer programming needs a solver process holding the
entire problem in memory, and a branch-and-bound tree cannot be pushed down into a parquet scan. A
greedy heuristic needs neither — every stage is an aggregation or a graph expansion, which is what a
database does natively.

Two H3 properties make the port easier here than in ForSys itself:

- **Adjacency is free and analytic.** Patchmax derives a stand adjacency graph from polygon
  topology. On H3, `h3_grid_disk` *is* the graph and `h3_grid_distance` *is* the metric — computed
  from the cell index, with no topology step and no stored edge list.
- **Hexagons are already a supported ForSys decision unit** — "a standardized decision unit like
  hexagons" — and they are this catalog's standard join key.

### Division of labor

The part that is hard at scale is *assembling* the treatment-unit table: pulling vegetation,
ownership, burn history, carbon and housing exposure from separate global datasets onto a common
hex index. That is what this architecture is for. The optimization itself runs on a table of a few
thousand rows and is milliseconds of work. This app adds the cheap half on top of the expensive
half we already had.

---

## Specifying the planning area

Every stage operates on a `units` table of `(h8, h0)` rows, so the only thing that differs between
the three ways of setting a planning area is how that table gets built. Nothing downstream knows the
difference.

**A. Name an existing polygon** — cheapest and most precise, and the one to reach for first. There
are *no geometry operations at all*: the reference dataset is already hex-indexed, so naming a
polygon is a plain attribute filter. This is how the default Tahoe National Forest area is defined.

```sql
units AS (   -- a national forest
  SELECT DISTINCT h8, h0 FROM read_parquet('s3://public-padus/padus-4-1/fee/hex/h0=*/data_0.parquet')
  WHERE h0 IN (SELECT h0 FROM scope) AND Des_Tp = 'NF' AND Unit_Nm = 'Tahoe National Forest'
)
units AS (   -- a named subwatershed
  SELECT DISTINCT h8, h0 FROM read_parquet('s3://public-usgs-wbd/wbd/hu12/hex/h0=*/data_0.parquet')
  WHERE h0 IN (SELECT h0 FROM scope) AND name = 'Fordyce Creek'
)
```

Nameable polygons in the catalog include PAD-US units (`Unit_Nm`), HUC12/HUC10 subwatersheds
(`name`, `huc12`), counties, census places, congressional districts and ecoregions. The HUC12 layer
is in the layer menu as an outline overlay so subwatersheds can be read off the map and named
directly — within Tahoe NF they run ~28,000–33,000 acres each, a realistic project extent:

```
Deer Creek-North Yuba River             180201250202   32,760 ac
Dolly Creek-Middle Fork American River  180201280302   32,032 ac
Fordyce Creek                           180201250601   31,668 ac
Jim Crow Creek-North Yuba River         180201250203   30,940 ac
Rattlesnake Creek-South Yuba River      180201250602   30,394 ac
```

Because both sides are hex-indexed, named polygons **intersect** by joining on `h8 AND h0` — which
is how most real planning questions are phrased ("the part of Tahoe NF inside this watershed").

**B. A drawn polygon** — `draw_enabled: true`. The agent reads the geometry via `get_drawn_region`.
`h0` is derived from the cells themselves, which *replaces* the usual `scope` CTE; omit it and every
downstream join loses partition pruning.

```sql
units AS (
  SELECT h8, h3_cell_to_parent(h8, 0) AS h0
  FROM (SELECT UNNEST(h3_polygon_wkt_to_cells('POLYGON((...))', 8)) AS h8)
)
```

**C. An uploaded GeoJSON** — `upload_enabled: true`, accepting `Polygon`/`MultiPolygon` only. The
file goes browser → S3 directly (`public-output/uploads`); only the URL reaches the agent, via
`get_uploaded_dataset`. DuckDB reads it over HTTPS; `ST_Union_Agg` collapses a multi-feature file
into one area of interest.

```sql
units AS (
  WITH aoi AS (SELECT ST_Union_Agg(geom) AS g FROM ST_Read('https://…/uploads/aoi.geojson'))
  SELECT h8, h3_cell_to_parent(h8, 0) AS h0
  FROM (SELECT UNNEST(h3_polygon_wkt_to_cells(ST_AsText((SELECT g FROM aoi)), 8)) AS h8)
)
```

> **A drawn or uploaded polygon is not automatically treatable land.** A box dragged over Tahoe NF
> also covers private parcels, reservoir surface and land outside the boundary. Intersect the area of
> interest with the relevant ownership or vegetation layer before reporting acreage, and say which
> layer was used. Path A does not have this problem — part of why it is preferred.

---

## The algorithm in six stages

| Stage | ForSys concept | SQL / H3 implementation |
|---|---|---|
| 1 | — | `h0` partition scope, carried into every join |
| 2 | Treatment units | `SELECT DISTINCT h8, h0` over the planning area |
| 3 | Priorities | One CTE per objective, aggregated to `h8`, normalized with `MAX(x) OVER ()` |
| 4 | Thresholds + exclusions | `WHERE` clauses; anti-joins on `h8` |
| 5 | Seed search | `CROSS JOIN UNNEST(h3_grid_disk(seed, k))` + `HAVING COUNT(*) = <ring size>` |
| 6 | Sequential project selection | Recursive CTE, reject seeds within `2k` grid steps |

The two least obvious pieces:

**Seed scoring is a moving window.** Patchmax scores each stand as a potential seed — "the potential
objective score of a patch if it were built from that location" — and likens it to a moving-window
analysis. That is a neighborhood aggregation, so all candidate seeds are scored in one set-at-a-time
pass rather than one search per seed.

**The greedy outer loop needs no client driver.** With fixed-radius k-ring patches, "exclude
previously claimed stands" has an exact closed form: two patches overlap iff their seeds are within
`2k` grid steps. A recursive CTE carrying claimed seeds as a list, with a `LATERAL` subquery
supplying the `ORDER BY … LIMIT 1` a recursive term cannot hold, runs the whole greedy sequence in
one statement.

---

## Verified result

Weights 0.40 WUI exposure / 0.35 fuel hazard / 0.25 irrecoverable carbon, k=2 (19-cell, ~3,458-acre
project areas), 8 projects, run against the live catalog:

```
6,329 treatment units  →  5,753 eligible  →  2,151 valid seeds  →  8 ranked projects

rank  patch_score      lat        lon      acres
   1        13.81   39.5404  -120.8350      3458
   2        13.62   39.2775  -120.8800      3458
   3        13.53   39.3813  -120.2110      3458
   4        13.49   39.2401  -120.1010      3458
   5        13.48   39.2768  -120.3910      3458
   6        13.41   39.2985  -120.2150      3458
   7        13.32   39.5706  -120.5870      3458
   8        13.30   39.5914  -120.8050      3458
```

Scores decrease monotonically, as a greedy sequence must. Selections cluster near Truckee, Nevada
City–Grass Valley and Downieville — the behaviour a WUI-weighted objective should produce, and a
useful check that the moving-window exposure term works. Supporting figures from the same pipeline:
63.2% of Tahoe NF units lie within 5 hex rings (~4.6 km) of mapped WUI; mean conifer share is 0.82.

---

## Datasets

| Role | Collection | Source |
|---|---|---|
| Planning area + wilderness exclusion | `pad-us-4.1-fee` | USGS GAP, PAD-US 4.1 |
| Recent-burn exclusion | `calfire-2025-firep` | CAL FIRE FRAP |
| Community exposure priority | `silvis-wui-2020` | SILVIS Lab WUI 1990–2020 v4 |
| Fuel hazard priority | `cwhr13` (fractions asset) | CAL FIRE FRAP FVEG 2022 / CWHR |
| Carbon priority | `irrecoverable-carbon` | Conservation International v2 (2025) — **CC BY-NC 4.0** |
| Nameable planning areas | `usgs-wbd-hu12` | USGS Watershed Boundary Dataset (HUC12) |
| Validation: actual treatments | `facts-common-attributes-2026-06` | USFS FACTS, EDW 2026-06-24 |

The FACTS layer is what lifts this above a demo: it is the agency's own record of completed
activities, with `PURPOSE_CODE` marking fuels work. Asking whether the algorithm's top project areas
coincide with where the Forest Service has actually been treating is a real external check.

---

## Aggregation discipline

Three traps, each of which silently produces a wrong number rather than an error:

- **`h0` in every hex-to-hex join** — `ON a.h8 = b.h8 AND a.h0 = b.h0`. With `h0` in the join,
  DuckDB's dynamic partition pruning skips non-matching files before reading them. Without it, only
  row-group stats apply, which means opening every file for a ~100 ms footer read over HTTP.
- **Vector hex assets repeat per-feature attributes** on every cell a feature covers. Never `SUM` a
  per-feature total (`GIS_ACRES`, `HU2020`, `NBR_UNITS_*`) off a hex asset without deduplicating on
  `_cng_fid`. Prefer a *density* (`HUDEN2020`) or *fraction* (`frac`) column — safe to average with
  no dedup. Raster-derived hex (carbon) is one row per cell and safe to `SUM` directly.
- **Normalize inside the planning area** — `MAX(x) OVER ()` over the units CTE, not a global
  constant, or every score collapses toward zero.

---

## Repository structure

```
index.html          ← HTML shell; core JS/CSS from CDN, pinned to geo-agent v3.25.0
layers-input.json   ← map layers, LLM settings, welcome examples, links.docs → about.html
system-prompt.md    ← the algorithm skeleton the agent executes
about.html          ← the app's About page: full algorithm documentation
k8s/                ← Kubernetes manifests (nginx + server-injected LLM key)
```

No JavaScript here. The map, chat, agent and tool modules all load from the
[geo-agent](https://github.com/boettiger-lab/geo-agent) CDN; this repo is configuration and
documentation only.

---

## Local development

```bash
python3 -m http.server 8000
# open http://localhost:8000 and enter an API key in the settings panel
```

## Deployment

### GitHub Pages

`layers-input.json` ships an `llm.user_provided` block, so each visitor supplies their own API key
(stored in the browser only). Enable Pages with **Settings → Pages → Source → GitHub Actions**; the
workflow in `.github/workflows/gh-pages.yml` deploys on push to `main`.

### Kubernetes

Keys are injected server-side, so visitors need none.

```bash
kubectl apply -f k8s/
kubectl rollout status -f k8s/deployment.yaml
```

The pod git-clones this repo at startup, so **push before restarting** or you serve stale content:

```bash
git push && kubectl rollout restart -f k8s/deployment.yaml
```

Requires the `open-llm-proxy-secrets` secret in the target namespace. The init container copies
`index.html`, `layers-input.json`, `system-prompt.md` **and `about.html`** — that last one is easy
to forget when editing the manifest, and dropping it makes the About link 404.

---

## Limits

Stated plainly, because they matter more than the capability:

- **No optimality guarantee.** Greedy heuristics carry no bound, and neither does ForSys. These are
  reproducible priority orderings, not optima. A provable optimum needs MILP, which is out of scope
  for this architecture.
- **Compact patches only.** Project areas are k-rings. Patchmax grows irregular patches by
  Dijkstra-ordered expansion, letting a project follow a ridgeline or fuel break. That is expressible
  as recursive cost-distance relaxation over the hex graph — DuckDB's recursive term has no priority
  queue, so it is Bellman-Ford relaxation rather than true Dijkstra, and naive `UNION ALL` recursion
  has no dedup, so frontier work grows with ring depth. Fine for compact patches; large-radius
  irregular patches want the relaxation iterated with min-cost materialized per step.
- **No treatment-response modelling.** ForSys reports flame-length reduction, biomass produced,
  habitat restored. Those need treatment-effect models this catalog does not hold. This app
  prioritizes *where*, not *what treatment achieves*.
- **Coarse operability screening.** Real plans screen slope, yarding distance and road access; only
  vegetation, ownership and burn history are screened here.
- **Not a treatment prescription.** Output is a prioritization score under user-chosen weights. Real
  planning requires field verification, NEPA review and local knowledge.

---

## References

- [The ForSys Research Consortium](https://www.forsysplanning.org/) · [How it works](https://www.forsysplanning.org/how-it-works)
- [forsys-sp/patchmax](https://github.com/forsys-sp/patchmax) — spatial patch-selection algorithm
- [forsys-sp/forsysr](https://github.com/forsys-sp/forsysr) — R implementation of ForSys
- [USFS RMRS ForSys project](https://research.fs.usda.gov/rmrs/projects/forsys)
- [boettiger-lab/mcp-data-server](https://github.com/boettiger-lab/mcp-data-server) — the DuckDB/H3 MCP server
- [boettiger-lab/geo-agent](https://github.com/boettiger-lab/geo-agent) — map + agent framework

Not affiliated with or endorsed by the ForSys Research Consortium or the US Forest Service.
