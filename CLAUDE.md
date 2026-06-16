# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-file, zero-dependency HTML dashboard for CHAMP paper application intake operations. It visualizes entry latency and demographic distribution from a CSV export. All processing is client-side; nothing is uploaded or transmitted.

## Running the app

Open `champ_dashboard.html` in a browser. For auto-loading to work, serve it over HTTP rather than `file://`:

```
python3 -m http.server 8080
# then open http://localhost:8080/champ_dashboard.html
```

When served from disk, the drag-and-drop or file-picker fallback activates automatically. No build step, no install.

## Expected CSV format

The dashboard resolves columns by fuzzy name matching (case-insensitive, punctuation-stripped), so exact casing doesn't matter. Expected columns:

| Canonical name | Aliases recognized |
|---|---|
| `time_received` | timereceived, received, receivedat |
| `time_entered` | timeentered, entered, enteredat |
| `age` | applicant age |
| `location` | applicant location, town, municipality, city |
| `language` | applicant language |
| `homeless` | — |

Missing timestamp columns degrade gracefully (latency stats blank; demographic charts still work). The `homeless` column accepts truthy strings: `true`, `t`, `yes`, `y`, `1`.

## Architecture

Everything lives in `champ_dashboard.html` — one `<style>` block, one `<script>` block, no modules.

**State** — two globals drive all rendering:
- `ROWS` — normalized record array built during CSV ingest; each record has computed fields `ageBracket`, `lat` (latency in days), and `homeless` string
- `FILTERS` — `{ dimension: Set(values) }` keyed by dimension name (`town`, `lang`, `ageBracket`, `homeless`)

**Data flow:**
1. `autoLoad()` → PapaParse fetches `paper_app_stats.csv` relative to the HTML file; falls back to manual file picker
2. `ingest(rows, headers)` — normalizes raw CSV rows into `ROWS`; uses `pick()` for tolerant column resolution
3. `render()` — called on every filter change; recomputes stats and redraws all bar charts
4. `renderBars(containerId, dim, opts)` — single bar-chart renderer used by all dimension panels; applies cross-dimension filtering via `passesExcept(r, dim)` so a chart only excludes filters from *other* dimensions (brushing/linking behavior)
5. `renderLatencyHist()` — special-cased histogram that bins by fixed day-ranges; not filterable (display-only)

**Filtering** — `toggleFilter(dim, val)` adds/removes values from `FILTERS[dim]` and calls `render()`. The `passesExcept(r, exceptDim)` function implements linked filtering: when rendering bars for dimension X, records filtered by dimension X are shown unfiltered so the chart reflects its own population.

## Design tokens

CSS custom properties on `:root` define the palette. Key tokens:
- `--brass` / `--brass-deep` — accent color (header rule, selected bars, latency histogram fill)
- `--slate` — primary bar fill and interactive elements
- `--paper` / `--panel` — page vs. card backgrounds
- `--good` / `--warn` — success/error states
