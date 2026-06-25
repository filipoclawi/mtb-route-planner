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
- [ ] Connect to a real local GraphHopper Switzerland server at `http://localhost:8989`.
- [ ] Add Trailforks browser/download ingestion. **Paused until Armin approves continuing beyond POC/deploy.**
- [ ] Add Swisstopo/geo.admin.ch elevation enrichment. **Paused until Armin approves continuing beyond POC/deploy.**
- [ ] Add gradient analysis and section-level metrics. **Paused until Armin approves continuing beyond POC/deploy.**
- [ ] Add optional public static publishing automation beyond this one-off POC deployment. **Paused.**

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
Use logged-in browser cookies/session to download official GPX into `runs/<run>/source_trails/`; cache by URL.

### Chunk 4 — Swisstopo elevation + gradients
Densify stitched route, call geo.admin.ch profile API in chunks, compute smoothed gradients, update viewer.

### Chunk 5 — publishing
Local-only by default; add explicit publish target with privacy/redaction checks.

## Current blocker / environment note
`http://localhost:8989/info` is currently connection-refused, so the first live GraphHopper integration cannot be exercised until the local GraphHopper Switzerland server is running. Java is not installed on this machine (`java: command not found`), so GraphHopper setup needs either a local JRE under `~/.hermes/` or a user-approved system Java install. The Switzerland PBF is reachable from Geofabrik and is about 512 MB. The code path is tested with a real local HTTP stub that mimics GraphHopper's `/route` response.
