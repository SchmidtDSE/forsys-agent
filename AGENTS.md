# AI Agent Guide — forsys-agent

A [geo-agent](https://github.com/boettiger-lab/geo-agent) client app. **Do not write JavaScript
here** — the map, chat, agent and tool modules load from the geo-agent CDN. This repo is
configuration and documentation only.

The authoritative guide for how a client app is structured, the full `layers-input.json` schema, and
the deployment patterns is
[geo-agent-template/AGENTS.md](https://github.com/boettiger-lab/geo-agent-template/blob/main/AGENTS.md).
Read that first. Only the app-specific notes below are additional.

## What this app is

A proof of principle that the [ForSys](https://www.forsysplanning.org/) greedy spatial heuristic for
fuel-treatment planning can be executed as DuckDB SQL over an H3 hex grid, rather than as an R
program over a precomputed stand adjacency graph. `about.html` is the algorithm documentation and
doubles as the app's About page; `README.md` covers the same ground for developers. Keep the two in
sync — they intentionally overlap.

## Files

| File | Role |
|---|---|
| `index.html` | Shell. CDN pinned to geo-agent **v3.25.0** (4 places). Third-party `integrity=` hashes are copied verbatim — never hand-edit one. |
| `layers-input.json` | Layers, LLM settings, welcome examples. `links.docs: "about.html"` is what renders the **About** link in the chat footer. |
| `system-prompt.md` | The algorithm skeleton the agent executes. |
| `about.html` | The About page — full algorithm write-up. |

## App-specific gotchas

- **`about.html` must be copied by the k8s init container.** `k8s/deployment.yaml` copies four files,
  not the template's three. Dropping the fourth makes the About link 404 with no other symptom.
- **The system prompt deliberately contains SQL.** The template guide says keep SQL out of the system
  prompt because `get_stac_details` supplies paths and schemas at runtime — correct, and the reason
  the skeleton there uses `<<placeholder>>` paths the agent must fill from `get_stac_details`. The
  *algorithm structure* is the exception the guide allows: it is domain context no tool can provide.
  Keep it that way — add algorithm shape, never hardcoded paths or column listings.
- **Verified numbers appear in three files** (`about.html`, `README.md`, and the funnel in
  `about.html`). If the pipeline or weights change, re-run against the live catalog and update all of
  them, or delete the figures. Do not let stale numbers stand.
- **PMTiles carry only a subset of parquet columns.** Every `tooltip_fields` and `default_filter`
  field in `layers-input.json` was verified against the PMTiles vector-layer metadata. Verify again
  before adding one — a wrong name fails silently.
- **A leading `~` on a model id is OpenRouter's floating alias** — `~deepseek/deepseek-v4-flash-latest`
  "always redirects to the latest model in the DeepSeek V4 Flash family" (OpenRouter's own
  description). All 11 tilde ids on OpenRouter end in `-latest`; pinned/dated ids carry no tilde.
  geo-agent does not interpret the tilde — it passes the string through — so this is purely an
  upstream convention. Keep "Latest" in the label so the floating behaviour is visible to users.
- **No markdown in `welcome.message` or `welcome.examples`.** geo-agent renders both with
  `escapeHtml` (chat-ui.js), not through `marked` — unlike normal chat messages. Any `**bold**` shows
  up as literal asterisks. Keep the welcome text plain prose.
- **PAD-US: screen protection status on `combined`, never on `fee`.** Designations overlay fee
  ownership and are absent from `fee`. Granite Chief Wilderness (GAP 1, ~27k acres) sits inside Tahoe
  NF but appears only in `combined`. The first version of this app got this wrong: the exclusion
  matched zero cells and the algorithm would site mechanical treatment inside designated wilderness.
  This applies to the map layers too — the `gap1-no-mech` layer is on `combined-pmtiles` for the same
  reason. GAP 3 = focal candidates; GAP 1/2 = excluded; absent from PAD-US = private land.
- **`h0` in every hex-to-hex join.** This is the dominant performance lever, not a micro-optimization.
  See `README.md` → Aggregation discipline.

## Deploying

```bash
git push && kubectl rollout restart -f k8s/deployment.yaml   # push FIRST — the pod git-clones
kubectl rollout status -f k8s/deployment.yaml
```

Name and namespace come from the manifest; never hardcode them from an example.
