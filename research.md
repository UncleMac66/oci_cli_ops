# Research: C4 Firmware Bundles — Caching, Upgrade Detection, and Per-Fabric Counts
Date: 2026-03-12

## Files Analyzed
- `oci_cli_ops.sh` — lines 41734–42084 (`manage_firmware_bundles()` — main fw command handler)
- `oci_cli_ops.sh` — lines 42086–42218 (`_fw_overview()` — firmware overview per fabric)
- `oci_cli_ops.sh` — lines 42220–42316 (`_fw_view_all_expanded()` — expanded bundle view)
- `oci_cli_ops.sh` — lines 42318–42425 (`_fw_view_bundle_detail()` — single bundle detail)
- `oci_cli_ops.sh` — lines 42426–42660 (`_fw_update_fabric_firmware()` — update flow)
- `oci_cli_ops.sh` — lines 13148–13225 (c4 display — fabric list with firmware line)
- `oci_cli_ops.sh` — lines 1790–1800 (`fetch_gpu_fabrics()` — FABRIC_CACHE writer)
- `oci_cli_ops.sh` — lines 2740–2755 (`fetch_compute_hosts()` — COMPUTE_HOST_CACHE writer)
- `oci_cli_ops.sh` — lines 500 (cache path constants)

## Architecture

### Firmware Bundle API
- **List**: `oci compute firmware-bundle list --platform <platform> --compartment-id <tenancy> --all --output json`
  - Returns `.data.items[]` with fields: `id`, `lifecycle-state`, `display-name`, `description`, `time-created`, `time-updated`, `platforms[].platform`, `platforms[].versions[].component-type`, `platforms[].versions[].version[]`
- **Get**: `oci compute firmware-bundle get --firmware-bundle-id <id>`
  - Returns single bundle with same fields plus `release-notes`
- **Platform values**: Sourced from COMPUTE_HOST_CACHE field 5 (e.g., `X9-6-GPU`)

### Current Data Flow (no caching)
1. User enters `fw` from c4 menu → calls `manage_firmware_bundles()`
2. Function reads platforms from `COMPUTE_HOST_CACHE` field 5 (auto-fetches compute hosts if needed)
3. Makes live `oci compute firmware-bundle list` API call every time
4. Parses JSON into temp file `${TEMP_DIR}/fw_parsed_$$.txt` for table rendering
5. JSON is kept in `$_fw_json` local variable, passed to sub-functions as `$1`
6. When sub-functions need the list (overview, expanded, update), they receive `$_fw_json` as arg

### Per-Fabric Firmware (already implemented in manage_firmware_bundles)
Lines 41802-41815 already build `_fw_cur_count`, `_fw_tgt_count`, `_fw_state_count` associative arrays from ALL fabric cache lines. The header summary (lines 41864-41918) shows per-fabric counts. The table rows (lines 41990-42006) show `← cur(3/5)` / `← tgt(2/5)` via `_COL_ROW_SUFFIX`. The expanded view (lines 42228-42281) rebuilds counts internally.

**The plan.md tasks 1-4 are already complete.** The code at lines 41802-42084 already implements:
- Task 1: Fabric count maps (lines 41802-41815)
- Task 2: Header summary with per-fabric counts (lines 41864-41918)
- Task 3: Main table with `_COL_ROW_SUFFIX` pattern (lines 41990-42015)
- Task 4: Expanded view with internal count rebuild (lines 42228-42281)

### Upgrade Detection (partially implemented in update flow only)
In `_fw_update_fabric_firmware()` at line 42471-42473:
```bash
_latest_bundle=$(jq -r '[.data.items[] | select(.["lifecycle-state"] == "ACTIVE")] | sort_by(.["display-name"]) | last | .id // ""' <<< "$_fw_json")
```
Then at lines 42493-42497, shows `↑ upgrade available` badge per fabric in the update selection screen. This logic exists only in the update flow — it is NOT shown on the c4 main display (line 13198-13225 firmware line).

### C4 Display Firmware Line (lines 13198-13225)
Currently shows: `├─ Firmware: [NO_UPDATE]    cur:..xxxx    tgt:..xxxx`
- `fw_state` from FABRIC_CACHE field 10
- `current_fw` / `target_fw` from FABRIC_CACHE fields 8-9
- No firmware bundle list data available at this point (bundles not cached)
- Cannot determine if upgrade exists without firmware bundle data

### Platform → Fabric Mapping
- COMPUTE_HOST_CACHE field 5 = platform (e.g., `X9-6-GPU`), field 15 = GPUFabricID
- FABRIC_CACHE has no platform field — must derive from COMPUTE_HOST_CACHE
- All hosts in a fabric typically share the same platform, so: `grep fabric_ocid COMPUTE_HOST_CACHE | cut -d'|' -f5 | sort -u`

## Key Finding: What Needs to Be Built

### 1. Firmware Bundle Cache
Currently NO cache exists for firmware bundles. Every `fw` command makes a live API call. Need:
- New cache file: `FW_BUNDLE_CACHE` — store pre-parsed bundle data
- Key by platform (different platforms have different bundles)
- Cache format: `BundleOCID|LifecycleState|DisplayName|Description|TimeCreated|TimeUpdated|Platform`
- Or simpler: cache the raw JSON per platform to `cache/fw_bundles_<platform>.json`
- TTL: Can be longer than fabric cache (bundles change infrequently) — 30min or 1hr
- `_fw_json` local var currently holds the API response — save it to cache file after fetch
- On re-entry to `manage_firmware_bundles()`, check `is_cache_fresh` before calling API

### 2. Upgrade Available Indicator on C4 Display
To show upgrade availability on the c4 firmware line (line 13224), need:
1. Know the platform for each fabric → derive from COMPUTE_HOST_CACHE
2. Know the available bundles for that platform → read from FW_BUNDLE_CACHE
3. Compare current firmware OCID against latest ACTIVE bundle OCID
4. If different, show upgrade indicator

**Challenge**: At c4 display time (line 13163), firmware bundle data may not be cached yet. Options:
- **Option A**: Pre-fetch bundles during c4 data collection (adds API call latency)
- **Option B**: Only show indicator if cache exists (lazy — shows after first `fw` visit)
- **Option C**: Fetch bundles in background during c4 display (complex)

**Recommendation**: Option B — only show if FW_BUNDLE_CACHE exists. After user visits `fw` once, the cache is populated and subsequent c4 views show the indicator. Add `r` refresh in c4 to re-fetch bundles.

### 3. Latest Bundle Detection
The update flow's approach (line 42472) sorts ACTIVE bundles by display-name and picks last:
```bash
jq -r '[.data.items[] | select(.["lifecycle-state"] == "ACTIVE")] | sort_by(.["display-name"]) | last | .id'
```
This works because OCI firmware bundle display-names are typically versioned (e.g., `FW-2024.01`, `FW-2024.03`). Can reuse this pattern.

## Dependencies
- `FABRIC_CACHE` — fields 8-10 for current/target/state per fabric (already populated)
- `COMPUTE_HOST_CACHE` — field 5 for platform, field 15 for GPUFabricID (already populated)
- `is_cache_fresh()` — existing TTL mechanism
- `_cache_write()` — existing atomic cache writer
- `CACHE_DIR` — line 490, `cache/` directory

## Gotchas & Risks
- **Platform mismatch**: If compute hosts have multiple platforms, need to map the correct platform to each fabric (via GPUFabricID linkage in COMPUTE_HOST_CACHE)
- **Bundle sort order**: Display-name sort assumes lexicographic ordering reflects version order (true for OCI's naming convention but not guaranteed)
- **Single fabric per platform**: If all fabrics share one platform (typical for GPU clusters), simplifies to one bundle cache
- **Cache staleness**: Firmware bundles change rarely — 30-60min TTL is safe
- **API permissions**: `firmware-bundle list` requires read access — same IAM policies as existing fw flow

## Existing Conventions
- Cache paths: `readonly` declarations at line ~490-510
- Cache format: pipe-delimited text with `# Comment` headers
- TTL registry: `_CACHE_TTL_MAP` (if used) or inline `is_cache_fresh` checks
- Fetch pattern: `fetch_*()` → `is_cache_fresh` → API call → `_cache_write`
- Display reads cache, never calls API directly

## Recommendations

### Cache Design
1. Add `FW_BUNDLE_CACHE` path constant: `"${CACHE_DIR}/fw_bundles.json"` (raw JSON, keyed by platform in filename)
   - Or: `"${CACHE_DIR}/fw_bundles_${platform}.json"` for multi-platform support
2. Add `fetch_firmware_bundles()` function following existing `fetch_*()` pattern
3. In `manage_firmware_bundles()`, call `fetch_firmware_bundles` instead of inline API call
4. Set TTL to 30 minutes (bundles change infrequently)

### Upgrade Indicator Design
1. On c4 display, after reading FABRIC_CACHE line, check if `FW_BUNDLE_CACHE` exists for the fabric's platform
2. If cache exists, find latest ACTIVE bundle OCID and compare with `current_fw`
3. If upgrade available, append `↑ avail` to firmware line (short, fits column layout)
4. Derive platform via: `grep "$fabric_ocid" "$COMPUTE_HOST_CACHE" | head -1 | cut -d'|' -f5`

### Indicator Format Options for C4 Firmware Line
Current: `├─ Firmware: [NO_UPDATE]    cur:..xxxx    tgt:..xxxx`
With upgrade: `├─ Firmware: [NO_UPDATE] ↑  cur:..xxxx    tgt:..xxxx`
Or: `├─ Firmware: [NO_UPDATE]    cur:..xxxx    tgt:..xxxx  ↑ newer available`
