# Plan: Firmware Bundle Caching + Upgrade Available Indicator on C4
Date: 2026-03-12
Status: DRAFT — awaiting review

## Summary
Add a firmware bundle cache so `--manage, c, 4, fw` doesn't re-fetch every time, then use that cache to show an upgrade-available indicator on the c4 main display firmware line.

## Context
- Affected file: `oci_cli_ops.sh`
- Affected areas:
  - Lines ~500-520: Cache path constants (add `FW_BUNDLE_CACHE`)
  - Lines ~587-597: `CACHE_TTL_MAP` (add 30min TTL for firmware bundles)
  - Lines ~1555-1572: Cache cleanup list (add new cache)
  - Lines ~41817-41840: `manage_firmware_bundles()` API call (replace with `fetch_firmware_bundles()`)
  - Lines ~13198-13225: C4 display firmware line (add upgrade indicator)
- Dependencies: `FABRIC_CACHE`, `COMPUTE_HOST_CACHE`, `is_cache_fresh()`, `_cache_write()`
- Risk level: LOW — display-only changes + new cache (no mutations affected)

## Tasks

### Task 1: Add cache path constant + TTL (~line 520, ~line 597)
- [ ] Add `readonly FW_BUNDLE_CACHE="${CACHE_DIR}/fw_bundles.json"` after line 520
- [ ] Add `["$FW_BUNDLE_CACHE"]=1800` to `CACHE_TTL_MAP` (30 minutes — bundles change rarely)
- [ ] Add `"$FW_BUNDLE_CACHE"` to the cache cleanup array (~line 1562)

### Task 2: Create `fetch_firmware_bundles()` function (~after line 2755)
- [ ] Follow existing `fetch_*()` pattern:
  - Accept `$1 = platform`, `$2 = region` (optional, defaults to `FOCUS_REGION`)
  - Check `is_cache_fresh "$FW_BUNDLE_CACHE"` — return early if fresh
  - Call `oci compute firmware-bundle list --platform "$platform" --compartment-id "$TENANCY_ID" --region "$region" --all --output json`
  - Write raw JSON to `FW_BUNDLE_CACHE` via `_cache_write`
  - Store the platform in a comment header line for validation (re-fetch if platform changes)
- [ ] Return 0 on success, 1 on failure

### Task 3: Update `manage_firmware_bundles()` to use cache (~lines 41817-41840)
- [ ] Replace inline API call with: `fetch_firmware_bundles "$selected_platform"`
- [ ] Read `_fw_json` from `FW_BUNDLE_CACHE` instead of live API response
- [ ] Keep spinner — `fetch_firmware_bundles` handles it, or show "cached" if fresh
- [ ] Add `r` refresh option to force re-fetch (invalidate cache then re-call)

### Task 4: Add upgrade indicator to c4 display firmware line (~line 13218-13225)
- [ ] After reading `current_fw` from FABRIC_CACHE, check if `FW_BUNDLE_CACHE` exists
- [ ] If it does, derive the fabric's platform from COMPUTE_HOST_CACHE:
  - `grep "$fabric_ocid" "$COMPUTE_HOST_CACHE" | head -1 | cut -d'|' -f5`
- [ ] Find latest ACTIVE bundle OCID from cache (reuse update flow's jq pattern):
  ```bash
  jq -r '[.data.items[] | select(.["lifecycle-state"] == "ACTIVE")] | sort_by(.["display-name"]) | last | .id // ""'
  ```
- [ ] If `current_fw != latest_bundle_id` and both are non-empty/non-N/A, append upgrade indicator
- [ ] Format: `├─ Firmware: [NO_UPDATE] ↑  cur:..xxxx    tgt:..xxxx`
  - `↑` in cyan after badge, before padding to cur: column
  - Or: append `↑ upgrade` after tgt: field
- [ ] If FW_BUNDLE_CACHE doesn't exist, show no indicator (user hasn't visited `fw` yet)

### Task 5: Add cache invalidation
- [ ] In `_focus_set_region()` and `_focus_set_compartment()`, add `rm -f "$FW_BUNDLE_CACHE"` alongside existing cache invalidations
- [ ] Platform changes in `manage_firmware_bundles` should also invalidate (if cached platform differs from selected)

### Task 6: Version bump
- [ ] Increment `SCRIPT_VERSION` from 3.28.1

### Task 7: Test
- [ ] Run `shellcheck oci_cli_ops.sh` for syntax/quoting issues
- [ ] Verify `--manage c 4 fw` uses cache on second visit (no spinner delay)
- [ ] Verify `r` refresh in fw view forces re-fetch
- [ ] Verify c4 display shows `↑` indicator when upgrade available (after visiting fw once)
- [ ] Verify c4 display shows no indicator when FW_BUNDLE_CACHE missing
- [ ] Verify c4 display shows no indicator when current firmware IS the latest
- [ ] Edge case: empty COMPUTE_HOST_CACHE (no platform derivation possible — skip indicator)

## Security Considerations
- No credential, token, or OCID masking changes
- Cache file contains bundle OCIDs and display names (non-sensitive)
- No new mutations — read-only cache + display indicator

## Rollback
- Single file change to `oci_cli_ops.sh` — `git checkout` to revert
- Delete `cache/fw_bundles.json` to remove cache
- No format changes to existing caches
