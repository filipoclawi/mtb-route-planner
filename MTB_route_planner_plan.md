# MTB route planner — stepwise build plan

## Goal
Build a local Hermes-facing MTB route-builder service that starts with a minimal vertical slice and grows toward Trailforks + GraphHopper + Swisstopo route analysis.

## Working directory
`/home/filipo/.hermes/workspace/mtb-route-planner`

## Progress tracker

- [x] Create project skeleton and plan file.
- [x] Minimal request schema: YAML with route name, one start coordinate, one local GPX trail, and GraphHopper URL.
- [x] Parse local Trailforks-style GPX with stdlib XML.
- [x] Call GraphHopper `/route` for access connector geometry.
- [x] Stitch connector + trail while removing duplicate boundary points.
- [x] Export unified GPX.
- [x] Export route analysis JSON and static placeholder Leaflet viewer.
- [x] Add CLI `build` and `inspect` commands.
- [x] Add automated test with local GraphHopper stub.
- [x] Publish the current POC viewer through GitHub Pages for user review.
- [x] Connect to a real local GraphHopper Switzerland server at `http://localhost:8989`.
- [x] Rerun the current POC against real GraphHopper and publish updated viewer.
- [x] Add Trailforks browser/download ingestion.
- [x] Add Swisstopo/geo.admin.ch elevation enrichment.
- [x] Add gradient analysis and section-level metrics.
- [x] Update viewer with elevation and 50 m smoothed gradient charts.
- [x] Add 25 m gradient mode, fixed public gradient color scale, mobile-first viewer UX, and map click/scrub readout.
- [x] Add optional public static publishing automation/history index for this GitHub Pages deployment.
- [x] Add authenticated Komoot export flow: public button only creates a Telegram approval request; actual Komoot import remains local and requires Armin approval in Telegram.

## Vertical chunks

### Chunk 1 — local POC (implemented)
Input local GPX + start coordinate; GraphHopper connector; stitched GPX; viewer.

Test command:

```bash
cd /home/filipo/.hermes/workspace/mtb-route-planner
python3 -m pytest -q
python3 -m mtb_route_builder.cli build --request examples/local_gpx_request.yaml --out runs/manual_test --graphhopper-url http://127.0.0.1:<stub-or-real>
```

### Chunk 2 — real GraphHopper
Install/import Switzerland graph, verify `/info`, available profiles and details, then run the same POC against localhost:8989. Recommended approach: install a local JDK under `~/.hermes/runtime/`, download a pinned GraphHopper web/server release under `~/.hermes/apps/graphhopper/`, download Geofabrik `switzerland-latest.osm.pbf` under `~/.hermes/data/graphhopper/`, and run it as a user-level systemd service. This keeps everything reversible and avoids Docker/sudo coupling while staying close to upstream GraphHopper docs.

### Chunk 3 — Trailforks ingestion
Implemented: use the logged-in Firefox Trailforks session/cookies with Trailforks' official GPX export endpoint, download source GPX into `runs/<run>/source_trails/`, and allow request YAML to specify `trailforks_trails` directly.

### Chunk 4 — Swisstopo elevation + gradients
Densify stitched route, call geo.admin.ch profile API, compute raw plus 20/25/50/100 m smoothed gradients, update viewer. Current viewer uses 25 m smoothed gradient for map coloring and charting, with separate route-direction color families: uphill green/yellow/red and downhill cyan/blue/purple. 50 m remains in analysis/table for comparison. Implemented for the current POC route with a single request below the 5,000-coordinate profile API limit; chunking remains a future hardening task for longer routes.

### Chunk 5 — publishing
Implemented: public GitHub Pages root is now a history index, with each published run copied under `results/<slug>/`; `latest.html` redirects to the latest result. Trailforks-derived public viewers include explicit Trailforks/Outside attribution.

### Chunk 6 — Komoot export auth gate
Implemented: viewer has an `Export to Komoot` button, but the static site never writes to Komoot and holds no secret. The button asks for browser confirmation, opens Telegram share with route slug/GPX URL, and relies on Armin's Telegram approval as the authenticator. After approval, run the local importer:

```bash
/home/filipo/.hermes/data/whatsapp-web-chromium/venv/bin/python scripts/export_to_komoot.py --slug <published-route-slug>
```

Dry-run validation:

```bash
/home/filipo/.hermes/data/whatsapp-web-chromium/venv/bin/python scripts/export_to_komoot.py --slug 2026-06-25-schindellegi-etzel-ii --dry-run
```

The script validates the slug against local `history.json`, reads the GPX from the published local repo mirror, uses Armin's Firefox Komoot cookies, imports as planned MTB-Enduro, continuous/single-stage, and follows the original route line if Komoot asks.

## Public POC viewer

- Live URL: https://filipoclawi.github.io/mtb-route-planner/
- GitHub repo: https://github.com/filipoclawi/mtb-route-planner
- Root page is now a history index of published route builds.
- Latest published result: `results/2026-06-25-schindellegi-etzel-ii/`.
- Previous Alp Clünas POC preserved at `results/2026-06-25-alp-cluenas-swisstopo-poc/`.
- Deployment verified in browser: history index and Etzel II viewer loaded, and no console errors.
- Published viewers include mobile-first layout, distinct uphill/downhill 25 m gradient map coloring, endpoint section markers, elevation/gradient charts, section gain/loss table, click/scrub map readout for distance/elevation/gradient, and an `Export to Komoot` button that only sends a Telegram approval request.

## GraphHopper local setup

Installed and verified:

- Java runtime: `/home/filipo/.hermes/runtime/java/temurin-21-jre/` (`openjdk 21.0.11`).
- GraphHopper: `/home/filipo/.hermes/apps/graphhopper/graphhopper-web-11.0.jar`.
- Config: `/home/filipo/.hermes/apps/graphhopper/config-switzerland-mtb.yml`.
- OSM extract: `/home/filipo/.hermes/data/graphhopper/switzerland-latest.osm.pbf`.
- Imported graph: `/home/filipo/.hermes/data/graphhopper/switzerland-gh-11-mtb`.
- Elevation cache: `/home/filipo/.hermes/data/graphhopper/srtm`.
- User service: `/home/filipo/.config/systemd/user/hermes-graphhopper.service`.
- Local endpoint: `http://127.0.0.1:8989`.
- Profiles: `mtb`, `bike`.
- Elevation/path-detail encoded values verified: `average_slope`, `max_slope`, `mtb_rating`, `surface`, `track_type`, `road_access`, `road_class`.

Useful commands:

```bash
systemctl --user status hermes-graphhopper.service
systemctl --user restart hermes-graphhopper.service
curl http://127.0.0.1:8989/info
```

Real GraphHopper POC run:

- Run dir: `/home/filipo/.hermes/workspace/mtb-route-planner/runs/real_graphhopper_poc`.
- Distance: 28.34 km.
- Connector: 0.35 km, 21 points, generated by real GraphHopper.
- Trail GPX: 27.99 km, 3126 points.
- Published viewer updated with this real-GraphHopper result.

## Swisstopo elevation + gradient slice

Implemented and verified:

- Swisstopo client: `mtb_route_builder/swisstopo_profile.py`.
- LV95 conversion and densification: `mtb_route_builder/geo.py`.
- Gradient metrics: `mtb_route_builder/gradients.py`.
- Viewer: elevation chart, 50 m smoothed gradient chart, section gain/loss and steepness table.
- Raw geo.admin.ch response saved per run at `swisstopo/elevation_profile_raw.json`.
- Enriched GPX and GeoJSON write Swisstopo-derived elevations.

POC run:

- Run dir: `/home/filipo/.hermes/workspace/mtb-route-planner/runs/swisstopo_gradient_poc`.
- Swisstopo profile samples: 2,836.
- Elevation model used: DTM2 for all samples.
- Total Swisstopo gain/loss: about 1,020 m / 1,933 m.
- Altitude range: 1,173–2,512 m.
- Steepest 50 m smoothed gradient: +25.0% / -54.6%.
- Local and deployed viewers verified with no browser console errors.

## Trailforks ingestion slice

Implemented and verified:

- New module: `mtb_route_builder/trailforks.py`.
- New CLI: `python3 -m mtb_route_builder.cli fetch-trailforks --url <trail-url> --out <path.gpx>`.
- Request YAML can now use `trailforks_trails` entries instead of pre-downloaded `local_trails`.
- Fetch uses existing logged-in Firefox cookies and the official Trailforks GPX export endpoint; it does not scrape map tiles or bypass account/download restrictions.
- Example request: `examples/trailforks_etzel_ii_request.yaml`.

Etzel II test run:

- Source: `https://www.trailforks.com/trails/etzel-ii/`.
- Start: Schindellegi-Feusisberg train station (`47.1764319, 8.7094217`).
- Run dir: `/home/filipo/.hermes/workspace/mtb-route-planner/runs/etzel_ii_trailforks`.
- Fetched GPX: `runs/etzel_ii_trailforks/source_trails/etzel-ii.gpx` (`application/gpx+xml`, 3,140 bytes, 24 GPX points).
- Unified route: 6.96 km, 302 points.
- Connector: 6.33 km, +302 m / -175 m.
- Trailforks Etzel II segment: 0.63 km, +1 m / -80 m.
- Swisstopo overall gain/loss: about +303 m / -255 m.
- Local viewer verified in browser with no console errors.

Verification commands:

```bash
cd /home/filipo/.hermes/workspace/mtb-route-planner
python3 -m compileall -q mtb_route_builder tests
python3 -m pytest -q
python3 -m mtb_route_builder.cli build --request examples/local_gpx_request.yaml --out runs/swisstopo_gradient_poc --graphhopper-url http://127.0.0.1:8989
python3 -m mtb_route_builder.cli inspect --run-dir runs/swisstopo_gradient_poc
```

## Current status / next pause point
GraphHopper Switzerland local routing, Trailforks GPX ingestion, Swisstopo elevation enrichment, gradient analysis, public history publishing, and Telegram-approved Komoot export scaffolding are implemented and verified. Keep future runs local-only by default unless Armin explicitly asks to publish; when publishing, add them under `results/<slug>/` and update `history.json`/root index. Never import to Komoot from the public site alone; only run `scripts/export_to_komoot.py` after Armin approves the specific slug in Telegram.
