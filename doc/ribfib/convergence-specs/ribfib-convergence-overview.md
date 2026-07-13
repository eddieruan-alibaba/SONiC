# RIB/FIB Convergence Project Design

This document is the consolidated design overview for the `ribfib-convergence` project. Goal: on top of the SONiC RIB/FIB layered architecture (Phase 1), implement an NHT-event-driven fpmsyncd backwalk fast-fixup mechanism to shorten route convergence time when a nexthop fails, covering both PIC core and PIC edge.

**Repositories involved** (all working on branch `ribfib_2_yuqing`):
- `sonic-frr` — Zebra RNH/dplane NHT event generation
- `sonic-buildimage`
  - `src/sonic-frr/dplane_fpm_sonic/` — SONiC-specific FPM provider (message assembly / encoding)
  - `src/libraries/sonic-fib/` — `libnexthopgroup` encode/decode library
- `sonic-swss/fpmsyncd` — the NHGMgr backwalk core
- `sonic-mgmt/tests/srv6/` — system-level convergence tests

**Phase 1 prerequisites (already in place):**
- FRR: `zebra_dplane_ctx` carries depends/dependents; `NEXTHOP_GROUP_RECEIVED` flag; `--nhg-fib` knob
- sonic-fib: `libnexthopgroup` library + `NextHopGroupFull` JSON schema
- fpmsyncd: `NHGMgr` class with RIB NHG table (`RIBNHGTable`) / SONiC PIC content table (`SonicPICContentTable`) / SONiC NHG ID map; PIC context / gateway NHG; SRv6 VPN path; APP_STATE_DB

---

## NHT event generation (FRR side)

**Detailed spec:** [`ribfib-convergence-frr.md`](ribfib-convergence-frr.md)

### Architecture decisions
- When a Zebra RNH state change settles (and candidate route entries have left the QUEUED transient), send an NHT event to the dplane.
- dplane adds `DPLANE_OP_NHT_EVENT_UPDATE`; the SONiC-specific FPM provider serializes it as `RTM_NEWNHTEVENT` (type 6000).
- Preserve NHG ID needs no special handling — Zebra's own hash mechanism guarantees determinism.

### Trigger point
- **Function:** `sonic-frr/zebra/zebra_rnh.c::zebra_rnh_eval_nexthop_entry()`
- **Prerequisite change:** `zebra_rnh_resolve_nexthop_entry()` gains an out parameter `route_entry_queued` to disambiguate the two causes of `re == NULL` (truly unreachable vs candidate skipped due to QUEUED).
- **Trigger condition:** `state_changed && !route_entry_queued`

### NHT message format (on the FPM wire)
```
nlmsghdr: type = RTM_NEWNHTEVENT (6000)
struct rtmsg: rtm_family = address family of rnh_prefix
NLA FPM_NHA_JSON_STR: JSON, 5 fields
```
JSON fields: `rnh_prefix`, `prev_resolved_prefix`, `prev_resolved_nhg_id`, `curr_resolved_prefix`, `curr_resolved_nhg_id`.

---

## NhtEvent encode/decode (sonic-fib)

**Detailed spec:** [`ribfib-convergence-sonic-fib.md`](ribfib-convergence-sonic-fib.md)

### Architecture decisions
- `NhtEvent` is **fully independent** from `NextHopGroupFull`: new schema, new C-API, new templates, mirrored file layout, namespace `fib`.
- **Do not modify** existing functions in `render_schema.py` — support NhtEvent via a new helper + a new `main()` branch.

### Key deliverables
- Schema: `src/libraries/sonic-fib/schema/NhtEvent.json`
- Generated C++ class: `fib::NhtEvent` (encode/decode)
- C-API: `nhtevent_capi.cpp/.h` (used by FRR encode)
- Templates: `nhtevent.h.j2` / `nhtevent_json.h.j2` / `c_nhtevent.h.j2`

---

## FPM message encoding (dplane_fpm_sonic.c)

**Detailed spec:** [`ribfib-convergence-dplane-encoding.md`](ribfib-convergence-dplane-encoding.md)

### Architecture decisions
- The SONiC-specific FPM provider `src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` (which replaces stock `dplane_fpm_nl.c` at runtime) assembles the `RTM_NEWNHTEVENT` message.
- It adds the `RTM_NEWNHTEVENT = 6000` enum value and a `DPLANE_OP_NHT_EVENT_UPDATE` op-dispatch handler that calls `nht_event_encode()` and puts the JSON into `FPM_NHA_JSON_STR`.

---

## Backwalk core algorithm (fpmsyncd)

**Detailed spec:** [`ribfib-convergence-swss.md`](ribfib-convergence-swss.md)

### Architecture decisions
- **All methods integrated into the `NHGMgr` class**; no new source files.
- Backwalk is **pure computation + APPDB write only**: it never mutates the in-memory NHG; the in-memory NHG is refreshed by the subsequent normal FRR NHG update.
- Supports Global table / SRv6 VPN / VXLAN EVPN.

### Two reverse indices (key Grill-Me decision)
- `m_nexthop_to_global_RIBNHG : map<string, set<RIBNHGEntry*>>` — Global view, used by PIC core
- `m_nexthop_to_vrf_RIBNHG : map<string, set<RIBNHGEntry*>>` — VRF/VPN view, used by PIC edge
- Identical shape, partitioned by RIB NHG scope, non-overlapping by construction
- The value stores `RIBNHGEntry*` (not a RIB NHG ID, not a SONiC NHG ID): warm reboot makes Zebra reassign NHG IDs, which would invalidate an ID-keyed index wholesale; entry pointers stay consistent because the reboot-time del/add re-populates the index through unindex/index.
- Maintenance points: **add/del only, never update** (Zebra semantics: a nexthop change goes through delete-then-re-add); `delNHGFull` calls `unindexNexthopToRIBNHG` before `delEntry`, so there is no dangling pointer.

### Direct-nexthop match (key Grill-Me decision)
- Test: `RIBNHGEntry::m_nexthop` (comma-separated) contains `failedNh`
- Implementation: `isDirectNexthop(entry, failedNh)` = `entry ∈ m_nexthop_to_global_RIBNHG[failedNh]`

### Generic prune rule (key Grill-Me decision)
No `isGatewayNhg()` type check. Based on `walk_result` + `depends.size()`:

| walk_result | depends.size() | Decision |
|---|---|---|
| true | any | do not prune |
| false | ≥ 2 (multi-path) | do not prune (defensive) |
| false | 1 (single-path) | prune |

The HLD's "stop at gateway NHG" is a surface behavior of this generic rule.

### PIC core / PIC edge share one framework
Both backwalks use the same `fib_nhg_back_walk()` doBackwalk logic; they differ only in **where the start-point set comes from**:
- PIC core: `prev_resolved_nhg_id` as the start
- PIC edge: `m_nexthop_to_vrf_RIBNHG[failedNh]` as the start set

### Forward walk cycle safety net
`resolveLeafPaths` / `collectAllLeafPaths` pass a top-level `visited set<ribID>`; on `nhgId ∈ visited` → log ERROR + return `[]`. **No fixed depth ceiling** (`MAX_NHG_RECURSION` is FRR serialization capacity, unrelated to runtime).

### Fast-fixup policy
| Situation | APPDB action | in-memory NHG | Walk continues |
|---|---|---|---|
| Valid paths remain | rewrite (remaining paths) | unchanged | yes |
| All fail | skip write | unchanged | yes (propagate fully_disabled upward) |
| Single-path miss | skip write | unchanged | no (prune) |

---

## Error handling and boundaries (cross-repo)

| Scenario | Handling |
|---|---|
| FPM NLA missing / JSON decode failure | log warning, drop |
| PIC core start lookup failure | log warning, skip PIC core, **PIC edge still runs** |
| dependent lookup failure | log warning, skip that dependent |
| Forward walk revisit (cycle guard) | log ERROR, truncate and return `[]` |
| Diamond-dependency backwalk revisit | modifiedSet overwrite + re-propagate upward (**allowed**) |
| APPDB write failure | log error; no retry, no rollback |
| PIC edge empty lookup | silent return (normal case) |
| PIC core start is composite with no dependents | log DEBUG, PIC core no-op |
| curr_resolved_nhg_id != 0 | rejected at routesync dispatch layer; Phase 1 does not process |

---

## Testing

**Detailed spec:** [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

All tests are consolidated in the test sub-spec. Three-layer split:

| Layer | Location | Covers |
|---|---|---|
| UT | `sonic-swss/tests/mock_tests/` | algorithm branches, boundaries, message parsing, error handling, index maintenance (≥80% line coverage) |
| topotest | `sonic-frr/tests/topotests/` | NHT trigger conditions, dplane ctx, FPM message format, QUEUED-transient suppression |
| system | `sonic-mgmt/tests/srv6/` | end-to-end timing (NHG before ROUTE), real SRv6 VPN gateway prune, no route flap |

**Key new helper** (`sonic-mgmt/tests/srv6/srv6_utils.py`): `verify_nhg_before_routes()` — parses `swss.rec` and asserts the NHG_TABLE SET timestamp precedes the ROUTE_TABLE SET timestamp.

**Baseline:** the 4 existing cases are kept as a regression baseline; this cycle adds 3 new cases (SRv6 gateway prune / topology1 timing / no route flap).

---

## Sub-spec navigation

- **Master design (consolidated):** [`ribfib-convergence-design.md`](ribfib-convergence-design.md)
- **sonic-frr — FRR-side NHT event generation:** [`ribfib-convergence-frr.md`](ribfib-convergence-frr.md)
- **sonic-fib — NhtEvent encode/decode library:** [`ribfib-convergence-sonic-fib.md`](ribfib-convergence-sonic-fib.md)
- **dplane-encoding — FPM provider message assembly:** [`ribfib-convergence-dplane-encoding.md`](ribfib-convergence-dplane-encoding.md)
- **sonic-swss — NHGMgr backwalk:** [`ribfib-convergence-swss.md`](ribfib-convergence-swss.md)
- **Test — UT + topotest + sonic-mgmt (all tests):** [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

## Change log

| Date | Feature | Change |
|------|---------|--------|
| 2026-07-05 | ribfib-convergence | Initial version: FRR NHT→dplane, sonic-fib NhtEvent encode/decode, fpmsyncd NHGMgr backwalk (PIC core + PIC edge), three-layer test strategy |
| 2026-07-06 | ribfib-convergence | Post-Grill-Me refactor: two reverse indices `m_nexthop_to_global_RIBNHG` + `m_nexthop_to_vrf_RIBNHG`; precise direct-nexthop match; removed isGatewayNhg, prune → generic rule; forward walk uses visited set; `re == NULL` disambiguated via `route_entry_queued` out param. Split into per-repo sub-specs |
| 2026-07-10 | ribfib-convergence | Reverse index value changed from RIB NHG ID to `RIBNHGEntry*` (warm reboot reassigns NHG IDs). Specs consolidated to English only; sub-specs reorganized into FRR / sonic-fib / dplane-encoding / swss + one dedicated test sub-spec |
