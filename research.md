# Research: C4 Firmware Bundle — Current/Target Indicators Across Multiple Fabrics
Date: 2026-03-12

## Files Analyzed
- `oci_cli_ops.sh` — lines 41760–42540 (firmware bundle display, overview, expanded view, update flow)
- `oci_cli_ops.sh` — lines 1770–1803 (fetch_gpu_fabrics → FABRIC_CACHE writer)
- `oci_cli_ops.sh` — line 491 (FABRIC_CACHE path constant)

## Architecture

### Data Source
- **API**: `oci compute compute-gpu-memory-fabric list` returns per-fabric fields:
  - `current-firmware-bundle-id` — the firmware OCID currently running on that fabric
  - `target-firmware-bundle-id` — the firmware OCID the fabric is upgrading to (if any)
  - `firmware-update-state` — e.g., NEEDS_ATTENTION, UP_TO_DATE, UPDATING
- **Cache**: Written as pipe-delimited text to `FABRIC_CACHE` (`cache/gpu_fabrics.txt`)
  - Format: `Name|Last5|OCID|State|Healthy|Avail|Total|CurrentFW|TargetFW|FWState|TimeCreated`
  - One line per fabric. Multiple fabrics = multiple lines.

### Bug: Only First Fabric Consulted
At line 41806, the code reads **only** `head -1` of the fabric cache:
```bash
_current_fw=$(grep -v "^#" "$FABRIC_CACHE" | head -1 | cut -d'|' -f8)
_target_fw=$(grep -v "^#" "$FABRIC_CACHE" | head -1 | cut -d'|' -f9)
_fw_state=$(grep -v "^#" "$FABRIC_CACHE" | head -1 | cut -d'|' -f10)
```
This means if you have 5 fabrics, and fabric 2–5 use a different firmware bundle, those are completely ignored. The `← current` / `← target` labels only reflect the first fabric's state.

### Where Indicators Appear (6 locations)

1. **Main table view** (line 41943): `← current` / `← target` appended to OCID in `_col_print_row`
   - Uses `_current_fw` / `_target_fw` from `head -1` — **BUG**
   - Also gets truncated due to column width (original user complaint)

2. **Header summary** (lines 41859–41869): Shows "Current Firmware (fabric):" and "Target Firmware (fabric):" labels
   - Same single-fabric values — **BUG**

3. **Expanded view** (`_fw_view_all_expanded`, line 42163): Passed `_current_fw`/`_target_fw` as args
   - Line 42194–42195: `← current` / `← target` per bundle — **BUG** (same single-fabric source)

4. **Overview** (`_fw_overview`, line 42029): Iterates ALL fabrics correctly ✅
   - Line 42069: `while IFS='|' read ... < <(grep -v "^#" "$FABRIC_CACHE")` — reads every fabric
   - Lines 42090–42111: Shows current/target per fabric correctly
   - Lines 42137–42141: Shows `← current` / `← target` per fabric in available bundles — correct

5. **Update flow** (`_fw_update_fabric_firmware`, line 42347): Reads all fabrics into arrays ✅
   - Line 42368–42377: Properly builds per-fabric arrays
   - Line 42470: Shows `← current` badge relative to the *selected* fabric — correct

6. **Update bundle selection** (line 42468–42474): Shows `← current` relative to selected fabric — correct ✅

### What Needs to Change

The **main table view** (location 1), **header summary** (location 2), and **expanded view** (location 3) all use the broken `head -1` pattern. These need to:

1. **Count** how many fabrics use each firmware bundle as current/target
2. **Display** counts like `← cur(3/5)` instead of bare `← current`
3. **Handle mixed states**: different fabrics may have different current and target bundles

### Proposed Indicator Format

In the table and expanded views, replace:
- `← current` → `← cur(3/5)` (3 of 5 fabrics using this as current)
- `← target` → `← tgt(2/5)` (2 of 5 fabrics targeting this)

In the header summary, replace single OCID display with a count-based summary:
- If ALL fabrics on same firmware: `Current Firmware: <OCID> (all 5 fabrics)`
- If mixed: Show each unique firmware with count

### Data Structures Needed

Build two associative arrays from FABRIC_CACHE:
```bash
declare -A _fw_current_count=()   # bundle_OCID → count of fabrics using it as current
declare -A _fw_target_count=()    # bundle_OCID → count of fabrics using it as target
local _total_fabrics=0
```

### Column Width Impact

The `← current` indicator is appended to the OCID field in `_ocid_display` (line 41945). Adding fabric counts makes this even longer. Options:
- Move indicator to a separate dedicated column (cleanest)
- Shorten to compact format: `← cur(3)` / `← tgt(2)`
- Keep in OCID column but use shorter format

## Dependencies
- `FABRIC_CACHE` — populated by `fetch_gpu_fabrics()`, read by all 6 locations
- `_col_print_row "FWB"` — column system controls width/visibility
- `_fw_view_all_expanded()` — receives `_current_fw`/`_target_fw` as positional args (signature must change)
- `_fw_overview()` — already correct, reads all fabrics independently

## Gotchas & Risks
- **Signature change**: `_fw_view_all_expanded` takes `$2=current_fw` `$3=target_fw` as scalar OCIDs. Must change to accept the count data or rebuild from cache internally.
- **Column truncation**: The OCID column already truncates `← current` to `← curren`. Adding counts makes this worse unless indicator is moved or shortened.
- **Zero fabrics**: If FABRIC_CACHE is empty or missing, counts should gracefully show nothing (existing `N/A` checks cover this).
- **Subshell context**: The `while read ... done < file` loop at line 41927 runs in main shell (not subshell), so associative arrays built before it will be accessible inside.

## Existing Conventions
- **Naming**: Functions prefixed with `_fw_` for firmware operations
- **Indicators**: `← current` in green, `← target` in yellow — consistent across all views
- **Color functions**: `color_firmware_state()` for firmware state coloring
- **Cache reading**: `grep -v "^#" "$CACHE_FILE"` to skip comment headers

## Recommendations
- Build `_fw_current_count` and `_fw_target_count` associative arrays from ALL fabric cache lines
- Replace scalar `_current_fw`/`_target_fw` with these maps in main table and expanded views
- Header summary: show per-bundle counts if mixed, or "all N fabrics" if uniform
- Keep indicator format concise to avoid column overflow: `← cur(3/5)` / `← tgt(2/5)`
- `_fw_overview` needs no changes — already correct
- `_fw_update_fabric_firmware` needs no changes — already per-fabric
- `_fw_view_all_expanded` signature: rebuild counts from cache inside the function (simpler than passing associative arrays)
