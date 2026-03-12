# Plan: Firmware Bundle — Per-Fabric Counts Instead of Single-Fabric Indicator
Date: 2026-03-12
Status: DRAFT — awaiting review

## Summary
Replace the broken `head -1` firmware indicator (which only reads the first fabric) with per-fabric counts across all 3 affected views: main table, header summary, and expanded view.

## Context
- Affected file: `oci_cli_ops.sh`
- Affected areas: lines ~41802–41870 (data collection + header), ~41941–41945 (table indicator), ~42002 (expanded view call), ~42158–42196 (expanded view function)
- Unaffected: `_fw_overview` (already correct), `_fw_update_fabric_firmware` (already per-fabric)
- Dependencies: `FABRIC_CACHE`, `_col_print_row`, `_COL_ROW_SUFFIX` pattern
- Risk level: LOW — display-only changes, no API calls or mutations affected

## Tasks

### Task 1: Build fabric count maps (~line 41802)
- [ ] Replace the `head -1` block (lines 41802–41809) with a loop over ALL fabric cache lines
- [ ] Build two associative arrays: `_fw_cur_count[bundle_ocid]` → number of fabrics using it as current, `_fw_tgt_count[bundle_ocid]` → number targeting it
- [ ] Also build `_fw_state_count[state]` → count per firmware-update-state
- [ ] Track `_total_fabrics` count
- [ ] Skip `N/A` values when counting

### Task 2: Update header summary (~lines 41856–41870)
- [ ] Show `Fabrics: N total` line after platform
- [ ] Replace single "Current Firmware (fabric):" with a loop over `_fw_cur_count` keys
  - If 1 unique bundle: `Current Firmware: <OCID>  (all N fabrics)` or `(N/N fabrics)`
  - If multiple: each on its own line with count
- [ ] Same pattern for target firmware — only show bundles where target differs from current
- [ ] Replace single firmware-update-state with per-state counts if mixed

### Task 3: Update main table indicator (~lines 41941–41953)
- [ ] Remove the `_indicator` append-to-OCID approach (fixes column truncation bug too)
- [ ] Use `_COL_ROW_SUFFIX` pattern (existing convention from compute hosts, line 56595) to append badge after the row
- [ ] Format: `← cur(3/5)` green if current count > 0, `← tgt(2/5)` yellow if target count > 0
- [ ] If both current and target, show both: `← cur(3/5) tgt(2/5)`
- [ ] If only 1 fabric total, simplify to `← current` / `← target` (no counts needed)
- [ ] Pass clean `_fwid` to `_col_print_row` as OCID (no indicator appended)

### Task 4: Update expanded view — `_fw_view_all_expanded` (~line 42158)
- [ ] Change function to read fabric counts from `FABRIC_CACHE` directly (instead of receiving scalar args)
- [ ] Build same `_fw_cur_count` / `_fw_tgt_count` maps inside the function
- [ ] Update indicator at lines 42193–42195 to show `← current (3/5 fabrics)` / `← target (2/5 fabrics)`
- [ ] If 1 fabric total, show simple `← current` / `← target`
- [ ] Update call site at line 42002: remove `$_current_fw` `$_target_fw` args

### Task 5: Version bump
- [ ] Increment `SCRIPT_VERSION` and `SCRIPT_VERSION_DATE`

### Task 6: Test
- [ ] Run `shellcheck oci_cli_ops.sh` for syntax/quoting issues
- [ ] Verify with `--manage c 4 fw` — check table shows per-fabric counts
- [ ] Verify `all` (expanded view) shows per-fabric counts
- [ ] Verify `overview` is unchanged (already correct)
- [ ] Verify `update` flow is unchanged (already per-fabric)
- [ ] Edge case: 0 fabrics in cache (no indicators shown)
- [ ] Edge case: 1 fabric (simplified labels without counts)

## Security Considerations
- No credential, token, or OCID masking changes — indicators are display-only
- No new API calls introduced

## Rollback
- Single file change to `oci_cli_ops.sh` — `git checkout` to revert
- No cache format changes, no config changes
