# RIB/FIB Convergence Design Specification

- **Feature**: ribfib-convergence
- **Corresponding HLD**: [`../ribfib.md`](../ribfib.md) (Phase 2 – Route Convergence Handling)
- **Status**: designing
- **Author**: Yuqing Zhao
- **Date**: 2026-07-05
- **Prerequisites**: Phase 1 (RIB/FIB base infrastructure) has landed on the `ribfib_2_yuqing` branch across all three repositories:
  - `sonic-frr`: `zebra_dplane_ctx` carries depends/dependents; `NEXTHOP_GROUP_RECEIVED` flag; `--nhg-fib` global knob
  - `sonic-fib` (`sonic-buildimage/src/libraries/sonic-fib`): `libnexthopgroup` encode/decode library + JSON schema
  - `sonic-swss/fpmsyncd`: `nhgmgr` class with SONiC zebra NHG table (`RIBNHGTable`) / SONiC PIC content table / SONiC NHG ID map; PIC context / gateway NHG; SRv6 VPN path; APP_STATE_DB
  - The working branch for all three repos is `ribfib_2_yuqing`.

---

## 1. Overview

The goal of this SDD cycle is to implement the **NHT-driven backwalk fast-fixup mechanism** on top of Phase 1, covering both PIC core and PIC edge, so that route convergence time is minimized when a nexthop becomes unreachable.

**In scope:**
- **FRR side**: introduce `DPLANE_OP_NHT_EVENT_UPDATE` dplane event and FPM message type `RTM_NEWNHTEVENT` (6000), triggered from `zebra_rnh_eval_nexthop_entry`.
- **sonic-fib side**: add `NhtEvent` JSON schema + encode/decode. Extend `render_schema.py` (without modifying existing functions) and add new Jinja2 templates.
- **fpmsyncd side**: extend the `NhgMgr` class in-place with `onNhtEvent()` / `backwalkPicCore()` / `backwalkPicEdge()` / `resolveLeafPaths()` / `collectAllLeafPaths()` methods, plus two new reverse indices `m_nexthop_to_global_RIBNHG` / `m_nexthop_to_vrf_RIBNHG`.

**Out of scope:**
- CLI (deferred).
- Preserve-NHG-ID special handling (Zebra's own mechanism already guarantees ID stability).
- `curr_resolved_nhg_id != 0` scenarios (deferred to a follow-up phase).

---

## 2. Key Design Decisions

| # | Decision | Conclusion |
|---|----------|------------|
| 1 | Scope | Full set: FRR NHT→dplane + fpmsyncd backwalk + PIC core/edge |
| 2 | Backwalk granularity | Core + Edge; supports Global table / SRv6 VPN / VXLAN EVPN |
| 3 | Fast-fixup policy | If any valid paths remain → rewrite APPDB; if all fail → skip APPDB write but continue walking |
| 4 | NHT message carrier | FPM adds `RTM_NEWNHTEVENT` (6000); dplane adds `DPLANE_OP_NHT_EVENT_UPDATE` |
| 5 | Preserve NHG ID | No special handling. Zebra's built-in mechanism is sufficient |
| 6 | Phase 1 gating | routesync dispatch rejects `curr_resolved_nhg_id != 0`; only the nexthop-down case is processed |
| 7 | Prune policy | Generic rule on `walk_result` + `depends.size()`; "stop at gateway NHG" is a surface behavior, no `isGatewayNhg()` type check |
| 8 | Forward walk | Embedded in each backwalk step; recurses along enabled `depends` down to leaves |
| 9 | Implementation structure | Everything integrated into the `NhgMgr` class; no new files |
| 10 | CLI | Not in this cycle |

---

## 3. FRR-side Design (sonic-frr/zebra/)

### 3.1 dplane enum extension

Add a new value to `enum dplane_op_e` in `zebra/zebra_dplane.h`:

```c
DPLANE_OP_NHT_EVENT_UPDATE,
```

### 3.2 NHT fields carried on the dplane ctx

Add a new sub-struct `struct dplane_nht_info` that carries:

- `struct prefix rnh_prefix`
- `struct prefix prev_resolved_prefix`
- `uint32_t prev_resolved_nhg_id`
- `struct prefix curr_resolved_prefix`
- `uint32_t curr_resolved_nhg_id`

Provide matching `dplane_ctx_get_nht_*()` accessors.

### 3.3 Trigger point

**Function**: `zebra/zebra_rnh.c::zebra_rnh_eval_nexthop_entry()`.

**Background: two causes of `re == NULL`**

When `zebra_rnh_resolve_nexthop_entry()` iterates candidate route entries, any candidate with `ROUTE_ENTRY_QUEUED && !ROUTE_ENTRY_INSTALLED` is skipped via `continue`. The caller therefore receives `re == NULL` in two very different situations:

1. **The route was actually withdrawn** → the nexthop became unreachable (`curr_resolved_nhg_id == 0`, **the very case Phase 1 must catch**).
2. **All candidates are blocked by QUEUED** → transient state; no NHT event should be emitted yet.

**Using `re && ...` as the trigger condition would suppress both cases, blocking the primary use case.** The resolve function must expose an extra out parameter so the caller can tell them apart.

**Trigger condition (final):**

```c
bool route_entry_queued = false;
re = zebra_rnh_resolve_nexthop_entry(..., &route_entry_queued);

// state_changed covers both re==NULL (truly unreachable) and re-changed cases.
// route_entry_queued specifically rejects the "candidates blocked by QUEUED" transient.
if (state_changed && !route_entry_queued) {
    dplane_nht_event_update(rnh, &prev_resolved_route, prev_resolved_nhg_id);
}
```

**Behavior table:**

| Scenario | re | route_entry_queued | Emit NHT event? |
|---|---|---|---|
| Route withdrawn, nexthop unreachable | NULL | false | ✓ |
| Candidates blocked by QUEUED (transient) | NULL | true | ✗ |
| Resolution moved (IGP switch to a new NHG) | new RE | false | ✓ |
| No change | any | any | ✗ (`state_changed == false`) |

**Why disambiguate via route_entry_queued?**
- `ROUTE_ENTRY_QUEUED` indicates the route entry is still being processed by FRR's meta queue and may undergo further changes.
- Emitting the NHT event while a candidate is QUEUED would let fpmsyncd act on an intermediate state that FRR may later revise, causing event churn.
- Waiting until QUEUED clears guarantees fpmsyncd sees the settled resolution result.

**Resolve-side change:** in `zebra_rnh_resolve_nexthop_entry()`, when a candidate is skipped due to `ROUTE_ENTRY_QUEUED && !ROUTE_ENTRY_INSTALLED`, set the new out parameter `*queued = true`; the caller then knows why NULL was returned.

**Implementation notes:**
- **Before** `copy_state()`, cache `rnh->resolved_route` and `rnh->resolved_nhg_id` into local variables (used as the `prev_*` values).
- **After** `copy_state()`, the fields on `rnh` have been updated to the new values (used as the `curr_*` values).
- If any candidate is QUEUED at this point, **do not send** the event this time — the next evaluate (after QUEUED clears) will send it.

### 3.4 FPM send

The SONiC-specific FPM provider `sonic-buildimage/src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` recognizes `DPLANE_OP_NHT_EVENT_UPDATE` and produces:

```
nlmsghdr: type = RTM_NEWNHTEVENT (6000), flags = NLM_F_CREATE | NLM_F_REQUEST
struct rtmsg: rtm_family = address family of rnh_prefix
NLA FPM_NHA_JSON_STR: JSON string (the 5 NhtEvent fields)
```

### 3.5 The three NHT scenarios: prev/curr combinations

| Scenario | prev_resolved_nhg_id | curr_resolved_nhg_id |
|----------|----------------------|----------------------|
| Nexthop completely unreachable (route deleted) | 243 (old) | 0 |
| Nexthop resolution moved (IGP reconvergence, new NHG) | 243 (old) | 244 (new) |
| Nexthop came back up | 0 | 244 (new) |

**Phase 1 handles only `curr == 0`** — fpmsyncd returns immediately in all other cases.

---

## 4. sonic-fib Design (sonic-buildimage/src/libraries/sonic-fib/)

### 4.1 New JSON schema

Add `schema/NhtEvent.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "NhtEvent.json",
  "title": "NhtEvent",
  "type": "object",
  "properties": {
    "rnh_prefix":           { "type": "string" },
    "prev_resolved_prefix": { "type": "string" },
    "prev_resolved_nhg_id": { "type": "integer", "minimum": 0 },
    "curr_resolved_prefix": { "type": "string" },
    "curr_resolved_nhg_id": { "type": "integer", "minimum": 0 }
  },
  "required": ["rnh_prefix", "prev_resolved_prefix", "prev_resolved_nhg_id",
               "curr_resolved_prefix", "curr_resolved_nhg_id"],
  "additionalProperties": false
}
```

### 4.2 Extend render_schema.py without touching existing functions

**Principle:** the existing `NextHopGroupFull` code path must not change. Support for `NhtEvent` is added by introducing **new helper functions and new templates** only.

**New pieces:**
- Add a new function such as `build_nht_root_struct()` dedicated to the `NhtEvent` schema.
- Add a new branch in `main()` that detects an `NhtEvent`-shaped schema and dispatches to the new function + templates.

**New Jinja2 templates:**
- `templates/nhtevent.h.j2`
- `templates/nhtevent_json.h.j2`
- `templates/c_nhtevent.h.j2`

### 4.3 File layout (mirrors NextHopGroupFull)

```
sonic-fib/
├── schema/
│   ├── NextHopGroupFull.json    ← existing, untouched
│   └── NhtEvent.json            ← new
├── src/
│   ├── c-api/
│   │   ├── nexthopgroup_capi.cpp/.h   ← existing, untouched
│   │   └── nhtevent_capi.cpp/.h       ← new
│   └── nexthopgroup_debug.cpp/.h      ← existing, untouched
├── templates/
│   ├── nexthopgroupfull*.j2           ← existing, untouched
│   └── nhtevent*.j2                   ← new
└── tests/
    └── test_nhtevent.cpp              ← new
```

### 4.4 FRR-side usage

The FPM netlink provider assembles `RTM_NEWNHTEVENT` by calling the C-API `nht_event_encode()` to produce the JSON string, which is placed into the `FPM_NHA_JSON_STR` NLA.

### 4.5 fpmsyncd-side usage

`routesync` extracts the JSON payload from the `FPM_NHA_JSON_STR` NLA of an `RTM_NEWNHTEVENT` message, calls `NhtEvent::decode()` to obtain the struct, and passes it to `NhgMgr`.

---

## 5. fpmsyncd Design (sonic-swss/fpmsyncd/)

### 5.1 routesync message dispatch

Extend the message dispatch in `routesync.cpp` to handle `RTM_NEWNHTEVENT` (6000):

```
1. Extract JSON string from the FPM_NHA_JSON_STR NLA.
2. Call NhtEvent::decode() to obtain the struct.
3. If curr_resolved_nhg_id != 0 → Phase 1 returns immediately.
4. Otherwise call NhgMgr::onNhtEvent(event).
```

### 5.2 New methods on the NhgMgr class (no new files)

```cpp
class NHGMgr {
public:
    void onNhtEvent(const NhtEvent& event);

private:
    void backwalkPicCore(ribID startNhgId, const std::string& failedNexthop);
    void backwalkPicEdge(const std::string& failedNexthop);

    // Filtered forward walk: recurse along enable_group and collect
    // remaining valid paths from affected subtrees.
    // `visited` guards against unexpected cycles (should never trigger on a proper DAG).
    std::vector<NexthopPath> resolveLeafPaths(
        ribID nhgId,
        const std::map<ribID, NodeState>& modifiedSet,
        std::set<ribID>& visited);

    // Unfiltered forward walk: collect every leaf path from a healthy subtree.
    std::vector<NexthopPath> collectAllLeafPaths(
        ribID nhgId,
        std::set<ribID>& visited);

    // Reverse index #1: NEXTHOP → RIBNHGEntry* (Global scope; used by PIC core)
    // key: nexthop address string (e.g. "2064:100::1d")
    // value: set of Global-scope RIB NHG entries that reference this nexthop
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_global_RIBNHG;

    // Reverse index #2: NEXTHOP → RIBNHGEntry* (VRF/VPN scope; used by PIC edge)
    // key: nexthop address string
    // value: set of VRF/VPN-scope RIB NHG entries that reference this nexthop
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_vrf_RIBNHG;

    // Maintenance helpers (route the RIB NHG to the correct index based on scope).
    void indexNexthopToRIBNHG(RIBNHGEntry* entry);
    void unindexNexthopToRIBNHG(RIBNHGEntry* entry);
};

struct NodeState {
    bool fully_disabled;
    std::map<ribID, bool> enable_group;   // Per-dep enabled state under this NHG
};
```

**Role separation of the two reverse indices:**

| Index | Scope | Consumer | Data structure |
|---|---|---|---|
| `m_nexthop_to_global_RIBNHG` | Global-table RIB NHGs | `backwalkPicCore()` | `map<string, set<RIBNHGEntry*>>` |
| `m_nexthop_to_vrf_RIBNHG` | VRF/VPN-table RIB NHGs | `backwalkPicEdge()` | `map<string, set<RIBNHGEntry*>>` |

**Both indices have identical shape** (`nexthop → set<RIBNHGEntry*>`); they are just partitioned by RIB NHG scope (Global vs VRF/VPN). Both backwalks use RIB NHG entries as entry points and share the same `fib_nhg_back_walk()` framework — PIC core vs PIC edge differ only in **where the start-point set comes from**; the traversal logic is reused.

**Why the value is a `RIBNHGEntry*` pointer instead of a RIB NHG ID:**
1. **Warm reboot reassigns NHG IDs:** during warm reboot Zebra hands out fresh NHG IDs, so an ID-keyed index would be invalidated wholesale. Entry pointers stay valid because the del/add that happens during reboot re-populates the index through the `unindex`/`index` helpers.
2. **No dangling pointers by construction:** `delNHGFull` calls `unindexNexthopToRIBNHG(entry)` **before** `m_rib_nhg_table->delEntry(id)`, so a pointer is always removed from the index before its entry is freed.
3. **Direct equality test:** `isDirectNexthop(entry, failedNh)` becomes a simple `entry ∈ m_nexthop_to_global_RIBNHG[failedNh]` set-membership check — no extra ID→entry lookup.

**Index maintenance points:**

| Trigger | Action |
|---|---|
| `addNHGFull()` — new RIB NHG | For each nexthop in `m_nexthop`, add the `RIBNHGEntry*` pointer to the index chosen by the RIB NHG's scope (Global / VRF). |
| `delNHGFull()` — remove RIB NHG | For each nexthop, remove the `RIBNHGEntry*` pointer from the corresponding index; drop the key when the set becomes empty. |
| `updateExistingNHGFull()` — update RIB NHG | **Do NOT update the indices.** Zebra's semantics: if the nexthop list changes, the NHG is deleted and re-added, so `update` only touches depends/dependents/attributes. |

**Strong constraint:** every code path that removes a RIB NHG **must** synchronously call `unindexNexthopToRIBNHG(entry)` **before** the entry is freed (`delNHGFull` does this before `delEntry`). Otherwise the index would retain a dangling `RIBNHGEntry*`, and backwalk dereferencing it would be a use-after-free.

**Usage example in `backwalkPicCore`:**
```cpp
auto it = m_nexthop_to_global_RIBNHG.find(failedNh);
if (it == m_nexthop_to_global_RIBNHG.end()) {
    // no Global-scope RIB NHG references failedNh
} else {
    for (RIBNHGEntry* entry : it->second) {
        // ... doBackwalk / direct-nexthop match / forward walk ...
    }
}
```

**Usage example in `backwalkPicEdge`:**
```cpp
auto it = m_nexthop_to_vrf_RIBNHG.find(failedNh);
if (it == m_nexthop_to_vrf_RIBNHG.end()) {
    return;   // silent: no VRF/VPN RIB NHG references this nexthop
}
for (RIBNHGEntry* entry : it->second) {
    ribID startId(entry->getRIBIDNum());
    // ... reuse the same backwalk framework, write the corresponding APPDB ...
}
```

### 5.3 `onNhtEvent()` flow

```
onNhtEvent(event):
  nexthop_addr = strip prefix length from event.rnh_prefix
  start_nhg_id = event.prev_resolved_nhg_id
  backwalkPicCore(start_nhg_id, nexthop_addr)   # PIC core
  backwalkPicEdge(nexthop_addr)                 # PIC edge
```

### 5.4 `backwalkPicCore()` logic (final version)

**Precise definition of the "direct-nexthop match" criterion:**
- Test: **`RIBNHGEntry::m_nexthop` (a comma-separated list of direct nexthops) contains `failedNh`**.
- Examples:
  - NHG 260 (`m_nexthop = "2064:100::1d"`) with failedNh = 2064:100::1d → **match**.
  - NHG 238 (`m_nexthop = "fc06::2,fc08::2"`) with failedNh = fc06::2 → **match** (one leg of a directly-attached ECMP group).
  - NHG 256 (composite, `m_nexthop` empty or does not contain failedNh) → **no direct match** (indirectly impacted; falls through to the depends-intersection branch).
- Implementation: `isDirectNexthop(entry, failedNh)` is equivalent to `entry ∈ m_nexthop_to_global_RIBNHG[failedNh]` (reusing the reverse index defined in 5.2 — no per-backwalk string parsing needed).

**Prune rule (generic, no node-type check):**

After visiting a dependent and computing `walk_result` (whether this node was hit by walk_spec — either via direct-nexthop match or a non-empty depends intersection):

| depends.size() | walk_result | Decision |
|---|---|---|
| Any | true (hit) | **Do not prune**; keep propagating upward. |
| ≥ 2 (multi-path) | false (miss) | **Do not prune** (defensive: in normal flow we only reach this node because one of its deps is in `modifiedSet`, so the intersection check usually hits; this branch guards against edge cases where walk_spec returns false). |
| 1 (single-path) | false (miss) | **Prune** (a single dependency chain that missed is truly unrelated — safe to stop). |

**Note:** gateway NHGs get no special treatment — if they don't contain failedNh and have no dep in `modifiedSet`, `walk_result = false` and the generic rule prunes them naturally. The HLD's "stop at gateway NHG" statement is a **surface behavior** that emerges from this simplified generic rule.

**Pseudo-code:**

```
backwalkPicCore(startId, failedNh):
  modifiedSet = {}
  entry_start = m_rib_nhg_table.getEntry(startId)

  # Starting point: mark fully_disabled only when m_nexthop directly contains failedNh.
  if entry_start exists and isDirectNexthop(entry_start, failedNh):
    modifiedSet[startId] = { fully_disabled: true, enable_group: all false }
  # else if entry_start exists but is composite (m_nexthop does not contain failedNh):
  #   Leave the start unlabeled; it is purely the entry point for iterating dependents.
  #   If entry_start.dependents is also empty → PIC core has nothing to do; log DEBUG.
  # else: start lookup failed → log warning, still call doBackwalk (it returns immediately),
  #   PIC edge still runs.

  doBackwalk(nhgId):
    entry = m_rib_nhg_table.getEntry(nhgId)
    if entry is null: return

    if entry.dependents is empty and nhgId == startId and startId ∉ modifiedSet:
      log DEBUG("PIC core: start %u is composite with no dependents; nothing to do", startId)
      return

    for each dependent of entry:
      dep_entry = m_rib_nhg_table.getEntry(dependent)
      if dep_entry is null: log warning; continue

      # Compute walk_result
      hit_direct = isDirectNexthop(dep_entry, failedNh)
      hit_intersect = (dep_entry.depends ∩ modifiedSet.keys() != ∅)
      walk_result = hit_direct or hit_intersect

      if walk_result:
        # Update modifiedSet
        if hit_direct:
          # Direct failure: mark every dep disabled
          enable_group = { every dep in dep_entry.depends → false }
          modifiedSet[dependent] = { fully_disabled: true, enable_group }
          # Skip APPDB write and do not mutate the in-memory NHG structure
          # (a subsequent normal FRR NHG update will overwrite state).
        else:
          # Indirect impact: compute enable_group from modifiedSet
          enable_group = {}
          for d in dep_entry.depends:
            if d ∈ modifiedSet and modifiedSet[d].fully_disabled:
              enable_group[d] = false
            else:
              enable_group[d] = true

          all_disabled = (every value in enable_group is false)
          modifiedSet[dependent] = { fully_disabled: all_disabled, enable_group }

          paths = resolveLeafPaths(dependent, modifiedSet, visited_local)
          if paths is not empty: rewrite this NHG in APPDB (APPDB only; in-memory NHG untouched)
          # If paths is empty → skip APPDB write and do not mutate in-memory NHG.

      # Generic prune rule
      if not walk_result and dep_entry.depends.size() < 2:
        # Single-path miss: prune, do not recurse.
        continue

      # Otherwise (hit, or multi-path miss): keep going.
      doBackwalk(dependent)

  doBackwalk(startId)
```

**Highlights:**
- **Every dependent performs the direct-nexthop match** — not only the start node, so cases such as NHG 263 (with `m_nexthop` containing failedNh but present mid-walk) are handled correctly.
- **Revisits are allowed** — diamond-shaped dependency graphs can visit the same node through multiple paths; `modifiedSet` records the state and re-computation overwrites earlier values, then propagates upward again.
- **PIC edge is independent of PIC core's start** — a failure to look up the start node only skips PIC core; PIC edge still runs.
- **`modifiedSet` is ephemeral scratch state for the current backwalk** — it never mutates the in-memory NHG data structure. The backwalk only writes APPDB; the in-memory NHG is refreshed by the subsequent normal FRR NHG update.
- **The direct-nexthop match reuses `m_nexthop_to_global_RIBNHG`** (defined in 5.2) — no per-backwalk parsing of the `m_nexthop` string.
- **Prune decisions use the generic rule** — no `isGatewayNhg()` type check is needed.

### 5.5 `backwalkPicEdge()` logic

PIC edge shares the **same backwalk framework** as PIC core (the `doBackwalk` defined in 5.4). The only difference is **where the start-point set comes from**: PIC core takes `prev_resolved_nhg_id` from the NHT event; PIC edge pulls the start set from `m_nexthop_to_vrf_RIBNHG`.

```
backwalkPicEdge(failedNh):
  Look up m_nexthop_to_vrf_RIBNHG[failedNh] → set of RIBNHGEntry* (VRF/VPN scope).
  If empty → silently return (normal case, no VRF/VPN RIB NHG references this nexthop).

  For each entry in the set:
    startId = entry->getRIBIDNum()
    Invoke the same doBackwalk logic used in backwalkPicCore(startId, failedNh)
    (if the start's m_nexthop contains failedNh it is naturally marked fully_disabled;
     walk its dependents; the generic prune rule decides when to stop).
```

**Note:** PIC edge covers VRF/VPN-scope RIB NHGs; PIC core covers Global-scope RIB NHGs. The two reverse indices **do not overlap** by construction, so no deduplication is needed.

### 5.6 `resolveLeafPaths()` — filtered forward walk

```
resolveLeafPaths(nhgId, modifiedSet, visited):
  # Defensive cycle detection — should not trigger on a proper DAG.
  if nhgId in visited:
    log ERROR("resolveLeafPaths: cycle detected at RIB NHG %u", nhgId)
    return []
  visited.insert(nhgId)

  # Defensive check: callers guarantee nhgId ∈ modifiedSet, but fall back
  # safely if a future refactor calls this in an unexpected context.
  if nhgId ∉ modifiedSet:
    return collectAllLeafPaths(nhgId, visited)

  entry = m_rib_nhg_table.getEntry(nhgId)
  myState = modifiedSet[nhgId]

  If entry is a leaf:
    if myState.fully_disabled: return []
    else: return [entry.nexthop + interface]

  results = []
  for dep in entry.depends:
    if myState.enable_group[dep] == false: continue   # branch disabled
    if dep ∈ modifiedSet:
      results += resolveLeafPaths(dep, modifiedSet, visited)   # dep also affected, recurse
    else:
      results += collectAllLeafPaths(dep, visited)             # dep healthy, collect entire subtree
  return results
```

**Caller contract:** the top-level caller (e.g. from `doBackwalk`) passes in an empty `visited` set. The set is shared across the same forward walk to guard against cycles.

### 5.7 `collectAllLeafPaths()` — unfiltered forward walk

```
collectAllLeafPaths(nhgId, visited):
  if nhgId in visited:
    log ERROR("collectAllLeafPaths: cycle detected at RIB NHG %u", nhgId)
    return []
  visited.insert(nhgId)

  entry = m_rib_nhg_table.getEntry(nhgId)
  if entry is null: return []
  If entry is a leaf: return [entry.nexthop + interface]
  results = []
  for dep in entry.depends:
    results += collectAllLeafPaths(dep, visited)
  return results
```

---

## 6. Error Handling and Corner Cases

### 6.1 Message layer

- **FRR side:** `dplane_nht_event_update()` returns immediately when `rnh` is NULL; JSON encode failure → log an error and drop the event; a disconnected FPM socket reuses the existing pending queue.
- **fpmsyncd side:** missing or zero-length `FPM_NHA_JSON_STR` NLA → log a warning and drop; JSON decode failure → log a warning and drop; missing required field → log a warning and drop.

### 6.2 State consistency

- **Start node lookup fails:** `SONiC zebra NHG table[prev_resolved_nhg_id]` is absent → log a warning and skip PIC core; **PIC edge still runs**, using only `failedNh` (which does not depend on any NHG start).
- **Dependent lookup fails:** log a warning and skip that dependent.

### 6.3 Cycle safety net

Zebra NHGs form a strict DAG, so forward walks never encounter cycles in normal operation. Defensive design:

- `resolveLeafPaths()` / `collectAllLeafPaths()` maintain an internal `visited set<ribID>`.
- Before recursing, if `nhgId ∈ visited` → **log ERROR + return empty list** (no crash, no infinite loop).
- We do **not** use a fixed depth ceiling; `MAX_NHG_RECURSION` is FRR-side serialization capacity and is unrelated to the runtime walk depth.

### 6.4 Diamond dependencies (revisit)

- Zebra NHGs form a strict DAG, so there are no cycles, but the same node can be reached via multiple paths.
- **Revisits are allowed.** `doBackwalk` does not use a `visited` set to skip nodes. `modifiedSet` records state; if a revisit computes different `fully_disabled` or `enable_group` values, they **overwrite** the earlier ones and upward propagation is retriggered.

### 6.5 APPDB write failure

- Log an error; do not retry (a subsequent normal NHG update will overwrite); do not roll back.

### 6.6 Concurrency

- `fpmsyncd` consumes messages single-threadedly, so no additional locking is required.

### 6.7 Walk continues even when everything fails

- When an NHG is fully disabled → do not write APPDB; **still call `doBackwalk(dependent)`** so that upstream nodes can correctly propagate `fully_disabled` into their `enable_group` computation.

### 6.8 PIC Edge silent case

- If `m_nexthop_to_vrf_RIBNHG[failedNh]` has no match → **silently return without logging** (this is the normal case: no VRF/VPN RIB NHG references this nexthop).

### 6.9 Phase 1 gating

- The routesync dispatch layer only rejects `curr_resolved_nhg_id != 0`.
- `prev_resolved_nhg_id == 0` is handled naturally inside the backwalk: PIC core hits the "start node lookup fails" branch and is skipped; PIC edge does not depend on `prev_id` and runs as usual.

---

## 7. Testing Strategy

Tests for all repositories are consolidated in the dedicated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md). The three-layer responsibility split is summarized below; see the test sub-spec for the full case list.

| Layer | Location | Covers | Does not cover |
|-------|----------|--------|----------------|
| UT (fpmsyncd) | `sonic-swss/tests/mock_tests/` | Algorithm branches, corner cases, error handling, message parsing | End-to-end timing / FRR behavior |
| topotest (FRR) | `sonic-frr/tests/topotests/` | NHT trigger conditions, dplane ctx, FPM message format, QUEUED suppression | APPDB content |
| sonic-mgmt (E2E) | `sonic-mgmt/tests/srv6/` | NHG-before-ROUTE timing, real SRv6 VPN gateway prune, absence of route flap | Algorithm branch coverage |

---

## 8. Dependencies and Impact Scope

### 8.1 Files touched (preliminary)

**sonic-frr:**
- `zebra/zebra_dplane.h` — new enum + accessor declarations
- `zebra/zebra_dplane.c` — new `dplane_nht_event_update()` + ctx handling
- `zebra/zebra_rnh.c` — (1) add out parameter `route_entry_queued` to `zebra_rnh_resolve_nexthop_entry()`; (2) insert the trigger call into `zebra_rnh_eval_nexthop_entry` (`state_changed && !route_entry_queued`)

**sonic-buildimage:**
- `src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` — SONiC-specific FPM provider; assemble `RTM_NEWNHTEVENT`
- `src/libraries/sonic-fib/schema/NhtEvent.json` — new
- `src/libraries/sonic-fib/scripts/render_schema.py` — **extend** with new functions + new dispatch branch (existing functions untouched)
- `src/libraries/sonic-fib/templates/nhtevent*.j2` — 3 new templates
- `src/libraries/sonic-fib/src/c-api/nhtevent_capi.cpp/.h` — new (generated by code-gen)
- `src/libraries/sonic-fib/Makefile.am` — add build rules
- `src/libraries/sonic-fib/tests/test_nhtevent.cpp` — new

**sonic-swss/fpmsyncd:**
- `routesync.cpp` — dispatch `RTM_NEWNHTEVENT`
- `nhgmgr.h/.cpp` — new methods added inside the existing class (no new files)

**sonic-swss/tests/mock_tests:** new backwalk unit tests.

**sonic-frr/tests/topotests:** new NHT dplane / FPM message tests.

**sonic-mgmt/tests/srv6:** new `srv6_utils.py` helper + new cases in `test_srv6_basic_sanity.py`.

### 8.2 Compatibility

- `RTM_NEWNHTEVENT` (6000) is a new FPM message type; older fpmsyncd instances silently ignore unknown types.
- The `NextHopGroupFull` code path is untouched.
- When the `--nhg-fib` knob is disabled, FRR does not emit NHT events (reusing the existing knob semantics).

---

## 9. GBrain References

No matching historical records (the GBrain MCP was not connected in this session).

---

## Grill-Me Review Record

**Reviewed:** 2026-07-06
**Target:** `doc/ribfib/convergence-specs/ribfib-convergence-design.md`

### Key decisions

1. **Both reverse indices store `RIBNHGEntry*` pointers — not RIB NHG IDs, not SONiC NHG IDs.**
   - `m_nexthop_to_global_RIBNHG` — Global scope, consumed by PIC core.
   - `m_nexthop_to_vrf_RIBNHG` — VRF/VPN scope, consumed by PIC edge.
   - Both are typed `map<string, set<RIBNHGEntry*>>`.
   - Pointers over IDs: warm reboot makes Zebra reassign NHG IDs, which would invalidate an ID-keyed index wholesale; entry pointers stay consistent because the reboot-time del/add re-populates the index via the `unindex`/`index` helpers.
   - Identical shape, partitioned by RIB NHG scope; the two backwalks share the same `fib_nhg_back_walk()` framework.

2. **"gateway match" is precisely a direct-nexthop match.**
   - Test: `RIBNHGEntry::m_nexthop` (comma-separated) contains `failedNh`.
   - Implementation reuses `m_nexthop_to_global_RIBNHG` (`entry ∈ ...[failedNh]`) — no per-backwalk string parsing.

3. **`isGatewayNhg()` is removed; prune uses a generic rule.**
   - Hit (walk_result=true) → no prune.
   - Multi-path miss → no prune (defensive).
   - Single-path miss → prune.
   - The HLD's "stop at gateway NHG" is a surface behavior of this generic rule; no type check is needed.

4. **Indices are maintained on add/del only, never on update.**
   - Zebra semantics: nexthop changes go through delete-then-reinsert; `update` only touches depends/dependents/attributes.
   - Every delete path must synchronously call `unindexNexthopToRIBNHG()`.

5. **Forward walk uses a visited-set safety net, not a fixed depth ceiling.**
   - `MAX_NHG_RECURSION` is FRR-side serialization capacity, unrelated to runtime walk depth.
   - `resolveLeafPaths` / `collectAllLeafPaths` maintain an internal `visited set<ribID>`.
   - On revisit → log ERROR + return `[]`.

6. **When PIC core's start is composite with no dependents, log at DEBUG.**
   - Not an error; simply eases debugging "why did fpmsyncd not write APPDB this time".

7. **The two causes of `re == NULL` are disambiguated via a new resolve out parameter `route_entry_queued`.**
   - Trigger condition: `state_changed && !route_entry_queued`.
   - Preserved cases: route withdrawn, IGP switch to a new NHG.
   - Excluded case: candidates blocked by QUEUED (transient).

### Issues found (resolved)

- [x] The original "NEXTHOP → SONiC NHG ID table" does not exist in the code; two new `map<string, set<RIBNHGEntry*>>` reverse indices were introduced (entry pointers to avoid warm-reboot NHG ID churn).
- [x] `gateway == failedNh` was semantically ambiguous; now precisely defined as "`m_nexthop` contains failedNh".
- [x] `isGatewayNhg()` conflated with direct-nexthop matching; method removed and prune simplified.
- [x] `re == NULL` could not distinguish true unreachability from transient QUEUED; solved with the `route_entry_queued` out parameter.
- [x] `MAX_NHG_RECURSION = 2` was misused as a runtime depth limit; replaced with visited-set cycle guard.
- [x] PIC core / PIC edge table conflation; confirmed as two disjoint indices scoped by Global vs VRF/VPN.

### Design points that survived pressure testing

- FPM message `RTM_NEWNHTEVENT` (6000) + JSON with 5 fields is sound; assembled in `dplane_fpm_sonic.c`.
- The `NhtEvent` JSON schema is independent from `NextHopGroupFull`; `render_schema.py` is extended (existing functions untouched) with new Jinja2 templates.
- Backwalk only writes APPDB — it never mutates the in-memory NHG (which is refreshed by the subsequent normal FRR NHG update).
- Fast-fixup policy: paths remain → rewrite APPDB; all fail → skip APPDB write but continue walking (propagates `fully_disabled` upward).
- Revisits are allowed: `modifiedSet` overwrites on re-computation for diamond dependencies; no need to skip.
- Testing is layered: UT (algorithm branches) / topotest (FRR-side NHT messages) / sonic-mgmt (end-to-end timing).
