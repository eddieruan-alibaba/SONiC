# RIB/FIB Convergence — Test
> **Developer-facing summary (LLD)** — This document is a low-level design summarized for developers to read.

- **Repositories**: `sonic-swss` (UT), `sonic-frr` (topotest), `sonic-mgmt` (E2E)
- **Overview**: [`ribfib-convergence-overview.md`](ribfib-convergence-overview.md)
- **Feature sub-specs**: [FRR](ribfib-convergence-frr.md) · [sonic-fib](ribfib-convergence-sonic-fib.md) · [dplane-encoding](ribfib-convergence-dplane-encoding.md) · [swss](ribfib-convergence-swss.md)

---

## 1. Purpose

This sub-spec consolidates **all** tests for the ribfib-convergence feature across the three repositories. The feature sub-specs intentionally carry no test detail; they point here.

## 2. Layered responsibility

| Layer | Location | Covers | Does not cover |
|-------|----------|--------|----------------|
| **UT** (fpmsyncd) | `sonic-swss/tests/mock_tests/` | Backwalk algorithm branches, corner cases, error handling, reverse-index maintenance, message parsing | End-to-end timing / FRR behavior |
| **topotest** (FRR) | `sonic-frr/tests/topotests/` | NHT trigger conditions, dplane ctx population, FPM/`RTM_NEWNHTEVENT` message format, QUEUED-transient suppression | APPDB content, backwalk algorithm |
| **E2E** (sonic-mgmt) | `sonic-mgmt/tests/srv6/` | NHG-before-ROUTE timing, real SRv6 VPN gateway prune, absence of route flap | Algorithm branch coverage |

**Coverage target (UT):** ≥ 80 % line coverage (inherited from the HLD goal).

---

## 3. fpmsyncd UT — NHGMgr backwalk

**Location:** `sonic-swss/tests/mock_tests/` (reusing the existing fpmsyncd mock framework). Suggested files: `backwalk_index_test.cpp`, `backwalk_pic_core_test.cpp`, `backwalk_forward_walk_test.cpp`.

### 3.1 Basic cases (message layer)

1. NHT message parsing (valid JSON / missing field / invalid type).
2. Phase 1 gate (`curr != 0` → return; `prev == 0` handled internally).
3. Start-node handling (direct-nexthop match / composite / lookup failure).

### 3.2 Algorithm branch coverage

Predefined topology JSON + injected event → verify APPDB changes.

4. `ut_backwalk_gateway_match_midwalk` — mid-walk direct-nexthop match (e.g. NHG 263 present mid-walk, not only at the start).
5. `ut_backwalk_diamond_dependency_revisit` — diamond dependency revisit consistency (modifiedSet overwrite).
6. `ut_backwalk_partial_failure_leaf_only` — NHG 258 fully_disabled; verify the upstream composite (NHG 256) collects only NHG 257's remaining paths.
7. `ut_backwalk_all_paths_fail_no_appdb_write` — all-fail → skip APPDB write; topology includes a sibling NHG to verify upstream still updates correctly.
8. `ut_backwalk_pic_edge_basic` — VRF/VPN reverse-index lookup yields the start, then runs the same backwalk framework.
9. `ut_backwalk_pic_edge_silent_miss` — VRF/VPN index has no match → silent return (no warning).
10. `ut_backwalk_forward_walk_cycle_defense` — inject an artificial cycle; the `visited` set logs ERROR and returns `[]`.

### 3.3 Reverse-index maintenance UT

11. `ut_index_add_del_roundtrip` — `addNHGFull` → `delNHGFull`; the index returns to its original state (pointer removed).
12. `ut_index_scope_routing` — a Global NHG lands only in the global index; a VRF NHG only in the vrf index.
13. `ut_index_no_update_maintenance` — `updateExistingNHGFull` leaves the index **unchanged**.

> **Note on the pointer-based index:** assertions retrieve the entry pointer via `getRIBNHGEntryByRIBID(ribID(...))` and check `index[nh].count(entry) == 1`, not a bare ID. This matches the `set<RIBNHGEntry*>` design (chosen because warm reboot reassigns NHG IDs).

---

## 4. sonic-frr topotest — FRR NHT dplane events

**Location:** `sonic-frr/tests/topotests/zebra_nht_event/` (extending the existing RIB/FIB topotest infrastructure). Artifacts: `test_nht_event.py`, `r1/frr.conf`, `r2/frr.conf`, `__init__.py`.

**Capture method:** a log-based check (grep for the `NHT_EVENT_UPDATE:` `zlog_info` trace), and/or a mock FPM listener (a Python socket on the FPM port parsing the netlink header + JSON payload).

### 4.1 Verification points

- `zebra_rnh_eval_nexthop_entry` enqueues `DPLANE_OP_NHT_EVENT_UPDATE` only when `state_changed && !route_entry_queued` holds (`route_entry_queued` is the new out parameter from `zebra_rnh_resolve_nexthop_entry()`).
- The dplane ctx `prev` / `curr` NHG ID fields are populated correctly.
- The `RTM_NEWNHTEVENT` message carries a valid `nlmsghdr` + `rtmsg` + complete 5-field `FPM_NHA_JSON_STR` (covers the dplane-encoding sub-spec end-to-end).
- Each of the three NHT scenarios produces the correct `prev/curr` combination — including `re == NULL` truly unreachable → **emit**; `re == NULL` due to QUEUED candidates → **do not emit**.

### 4.2 QUEUED-transient suppression negative test

- Use `--asic-offload=notify_on_offload` (or a similar knob) to stretch the QUEUED window on candidate route entries.
- Trigger a nexthop change while a candidate is still QUEUED → verify `RTM_NEWNHTEVENT` is **not** emitted.
- Re-trigger evaluate after QUEUED clears → the NHT event is now sent.

---

## 5. sonic-mgmt system tests — end to end

**Location:** `sonic-mgmt/tests/srv6/`. Reuse the existing framework (`srv6_utils.py` helpers + the pattern in `test_srv6_basic_sanity.py`).

### 5.1 Baseline (kept untouched, used as regression baseline)

- `test_topology1_local_failure` / `test_topology1_remote_failure`
- `test_topology2_local_failure` / `test_topology2_remote_failure`

### 5.2 New helper (`srv6_utils.py`)

- `verify_nhg_before_routes(record_file, nhg_pattern, route_pattern)` — parses `swss.rec` and asserts the `NEXTHOP_GROUP_TABLE` SET timestamp precedes the `ROUTE_TABLE` SET timestamp.

### 5.3 New cases

1. `test_srv6_gateway_prune` — VPN route scenario; fail an underlay nexthop; verify backwalk stops at the gateway NHG. `verify_nhg_before_routes` asserts the timing.
2. `test_convergence_timing_topology1` — reuse the topology1 setup and add the timing assertion.
3. `test_convergence_no_route_flap` — assert that during fast fixup there are no repeated SET/DEL entries on `ROUTE_TABLE`.

### 5.4 Known environment issue (to watch)

The 7-node force10 run passed the setup stage but the Run Test stage hit `KeyError: 'local'` inside `show ip route` (a sonic-utilities issue: protocol `'local'` missing from `proto_code`). This is unrelated to convergence logic but can block test-result parsing; track separately.

---

## 6. What each layer intentionally does NOT cover

- UT does not cover FRR trigger conditions (topotest) nor real-device end-to-end timing (sonic-mgmt).
- topotest does not cover APPDB content or the backwalk algorithm (UT).
- sonic-mgmt does not cover algorithm branch coverage (UT).
