# RIB/FIB Convergence — sonic-swss

- **Repository**: `sonic-swss` (`fpmsyncd/`)
- **Overview**: [`ribfib-convergence-overview.md`](ribfib-convergence-overview.md)
- **Depends on**: [`ribfib-convergence-sonic-fib.md`](ribfib-convergence-sonic-fib.md) (`fib::NhtEvent` decode)
- **Tests**: [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

---

## 1. Responsibility of this repository

Implement the NHT-event-driven backwalk fast-fixup mechanism inside `fpmsyncd`, covering both PIC core and PIC edge. All new methods are integrated into the existing `NHGMgr` class; no new source files are added.

**Scope:**
- `fpmsyncd/routesync.cpp` — dispatch the new `RTM_NEWNHTEVENT` (6000) message.
- `fpmsyncd/nhgmgr.h/.cpp` — extend the `NHGMgr` class with `onNhtEvent()` / `backwalkPicCore()` / `backwalkPicEdge()` / `resolveLeafPaths()` / `collectAllLeafPaths()`, plus two new reverse indices.

**Not in scope:** FRR changes, FPM message assembly, sonic-fib encode/decode (all in other sub-specs), all tests (see test sub-spec).

---

## 2. routesync message dispatch

`fpmsyncd/routesync.cpp::onMsgRaw()` whitelist + length check + dispatch:

```cpp
// onMsgRaw whitelist addition
&& (h->nlmsg_type != RTM_NEWNHTEVENT)   // new

// length-check branch (NHT event uses struct rtmsg)
case RTM_NEWNHTEVENT:
    hdr_len = sizeof(struct rtmsg);
    break;

// dispatch
case RTM_NEWNHTEVENT:
    onNhtEventMsg(h, len);
    return;
```

New `RouteSync::onNhtEventMsg(struct nlmsghdr *h, int len)`:

```cpp
void RouteSync::onNhtEventMsg(struct nlmsghdr *h, int len) {
    /* 1. Extract the JSON string from the FPM_NHA_JSON_STR NLA. */
    /* 2. fib::NhtEvent::decode(); on failure log a warning and drop. */
    /* 3. Phase 1 gate: curr_resolved_nhg_id != 0 → return. */
    /* 4. Call m_nhgmgr->onNhtEvent(event). */
}
```

**`RTM_NEWNHTEVENT` constant:** must match `sonic-buildimage/src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` (6000). Define it locally on the fpmsyncd side (or share a common header if one is available from sonic-fib).

---

## 3. New methods on the NHGMgr class

Add to `class NHGMgr` in `fpmsyncd/nhgmgr.h`:

```cpp
class NHGMgr {
public:
    /* existing addNHGFull / delNHGFull / getRIBNHGEntryByRIBID etc. */

    /* new */
    void onNhtEvent(const fib::NhtEvent& event);

private:
    void backwalkPicCore(ribID startNhgId, const std::string& failedNexthop);
    void backwalkPicEdge(const std::string& failedNexthop);

    /*
     * Filtered forward walk: recurse along enable_group and collect valid paths.
     * `visited` is a defensive cycle guard (should never trigger on a proper DAG).
     */
    std::vector<NexthopPath> resolveLeafPaths(
        ribID nhgId,
        const std::map<ribID, NodeState>& modifiedSet,
        std::set<ribID>& visited);

    /* Unfiltered forward walk: collect every leaf path from a healthy subtree. */
    std::vector<NexthopPath> collectAllLeafPaths(
        ribID nhgId,
        std::set<ribID>& visited);

    /*
     * Reverse index #1: NEXTHOP → RIBNHGEntry* (Global scope; used by PIC core)
     * key: nexthop address string (e.g. "2064:100::1d")
     * value: set of Global-scope RIB NHG entries referencing this nexthop
     */
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_global_RIBNHG;

    /*
     * Reverse index #2: NEXTHOP → RIBNHGEntry* (VRF/VPN scope; used by PIC edge)
     * Same shape and purpose as #1, scoped to VRF/VPN.
     */
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_vrf_RIBNHG;

    /* Maintenance helpers (route the RIB NHG to the correct index by scope). */
    void indexNexthopToRIBNHG(RIBNHGEntry* entry);
    void unindexNexthopToRIBNHG(RIBNHGEntry* entry);

    /* direct-nexthop match: equivalent to entry ∈ m_nexthop_to_global_RIBNHG[failedNh] */
    bool isDirectNexthop(RIBNHGEntry* entry, const std::string& failedNh);
};

struct NodeState {
    bool                   fully_disabled;
    std::map<ribID, bool>  enable_group;   /* per-dep enabled state under this NHG */
};

struct NexthopPath {
    std::string nexthop;
    std::string ifname;
    /* Add seg6_src / vpn_sid etc. SRv6 metadata if the backwalk output needs it. */
};
```

---

## 4. Reverse-index maintenance (key design)

### 4.1 Data structure

| Index | Scope | Consumer | Data structure |
|---|---|---|---|
| `m_nexthop_to_global_RIBNHG` | Global-table RIB NHGs | `backwalkPicCore()` | `map<string, set<RIBNHGEntry*>>` |
| `m_nexthop_to_vrf_RIBNHG` | VRF/VPN-table RIB NHGs | `backwalkPicEdge()` | `map<string, set<RIBNHGEntry*>>` |

**Both indices have identical shape**, partitioned only by RIB NHG scope (Global vs VRF/VPN). Both backwalks start from RIB NHG entries and share the same `fib_nhg_back_walk()` framework — PIC core and PIC edge differ only in **where the start-point set comes from**.

### 4.2 Why the value is a `RIBNHGEntry*` pointer instead of a RIB NHG ID

**Core reason: warm reboot reassigns NHG IDs.** With an ID-keyed index, every old ID becomes invalid after a warm reboot and the whole index must be rebuilt. A `RIBNHGEntry*` pointer, by contrast, is re-populated naturally during the reboot-time "delete-then-re-add" through `unindexNexthopToRIBNHG` / `indexNexthopToRIBNHG`, so the index stays semantically continuous.

**Dangling-pointer avoidance:**
- `delNHGFull()` calls `unindexNexthopToRIBNHG(entry)` **before** `m_rib_nhg_table->delEntry()` (the entry is still valid at that point), so the index never retains a pointer to a deleted entry.
- `updateExistingNHGFull()` does not change the nexthop list (Zebra semantics: a nexthop change goes through delete-then-re-add), so the entry address is stable across an update.
- Therefore the index never holds a pointer to a freed object.

### 4.3 Maintenance points

| Trigger | Action |
|---|---|
| `addNHGFull()` adds a RIB NHG | Iterate `RIBNHGEntry::m_nexthop` (comma-separated), pick the index by RIB NHG scope (Global / VRF), insert the `entry` pointer into the set. |
| `delNHGFull()` removes a RIB NHG | **Before** the entry is deleted, iterate `m_nexthop`, remove the `entry` pointer from the corresponding index; drop the key when the set is empty. |
| `updateExistingNHGFull()` updates a RIB NHG | **Do not maintain the index** (Zebra: a nexthop-list change goes through delete-then-re-add, not update; the entry address is unchanged). |

**Strong constraint:** every code path that removes a RIB NHG must synchronously call `unindexNexthopToRIBNHG(entry)` **before** `delEntry()`. Violating this (delete-then-unindex) leaves a dangling pointer in the index, and backwalk dereferencing it is a use-after-free crash.

### 4.4 Scope determination

`indexNexthopToRIBNHG(entry)` picks the index by the `RIBNHGEntry` VRF attribute:
- Global table (vrf_id == the default VRF) → `m_nexthop_to_global_RIBNHG`.
- Other VRF / VPN → `m_nexthop_to_vrf_RIBNHG`.

The exact field depends on the `RIBNHGEntry` internals (either `NextHopGroupFull.vrf_id` or a dedicated VRF marker).

---

## 5. `onNhtEvent()` flow

```
onNhtEvent(event):
  nexthop_addr = strip prefix length from event.rnh_prefix
  start_nhg_id = event.prev_resolved_nhg_id
  backwalkPicCore(start_nhg_id, nexthop_addr)   // PIC core
  backwalkPicEdge(nexthop_addr)                 // PIC edge
```

**Phase 1 gate:**
- The routesync dispatch layer **only rejects `curr_resolved_nhg_id != 0`**.
- The `prev_resolved_nhg_id == 0` case is handled naturally inside the backwalk (PIC core lookup fails → skip; PIC edge does not depend on `prev_id` → runs as usual).

---

## 6. `backwalkPicCore()` logic

### 6.1 Direct-nexthop match definition

- Test: `RIBNHGEntry::m_nexthop` (comma-separated list of direct nexthops) **contains `failedNh`**.
- Examples:
  - NHG 260 (`m_nexthop = "2064:100::1d"`) + failedNh = 2064:100::1d → **match**.
  - NHG 238 (`m_nexthop = "fc06::2,fc08::2"`) + failedNh = fc06::2 → **match** (one leg of a directly-attached ECMP group).
  - NHG 256 (composite, `m_nexthop` empty or not containing failedNh) → **no match** (indirectly impacted; falls to the depends-intersection branch).
- Implementation: `isDirectNexthop(entry, failedNh)` is equivalent to `entry ∈ m_nexthop_to_global_RIBNHG[failedNh]` (reuses the reverse index — no per-backwalk string parsing).

### 6.2 Prune rule (generic, no node-type check)

After visiting a dependent and computing `walk_result` (whether this node was hit by walk_spec — direct-nexthop match or a non-empty depends intersection):

| depends.size() | walk_result | Decision |
|---|---|---|
| Any | true (hit) | **Do not prune**; keep propagating upward. |
| ≥ 2 (multi-path) | false (miss) | **Do not prune** (defensive; normal backwalk reaching here means some dep is in modifiedSet, so intersection usually hits; this guards edge cases where walk_spec returns false). |
| 1 (single-path) | false (miss) | **Prune** (a single dependency chain that missed is truly unrelated — safe to stop). |

**Note:** gateway NHGs get no special treatment — if they neither contain failedNh nor have a dep in modifiedSet, `walk_result = false` and the generic rule prunes them naturally. The HLD's "stop at gateway NHG" is a **surface behavior** of this simplified generic rule.

### 6.3 Pseudo-code

```
backwalkPicCore(startId, failedNh):
  modifiedSet = {}
  entry_start = m_rib_nhg_table.getEntry(startId)

  # Start: mark fully_disabled only when m_nexthop directly contains failedNh.
  if entry_start exists and isDirectNexthop(entry_start, failedNh):
    modifiedSet[startId] = { fully_disabled: true, enable_group: all false }
  # else if entry_start exists but is composite (m_nexthop does not contain failedNh):
  #   leave the start unlabeled; it is only the entry point for iterating dependents.
  #   if entry_start.dependents is also empty → PIC core has nothing to do; log DEBUG.
  # else: start lookup failed → log warning, still call doBackwalk (returns immediately);
  #   PIC edge still runs afterward.

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
        if hit_direct:
          # Direct failure: mark every dep disabled
          enable_group = { every dep in dep_entry.depends → false }
          modifiedSet[dependent] = { fully_disabled: true, enable_group }
          # Skip APPDB write and do not mutate in-memory NHG (a later normal
          # FRR NHG update overwrites state).
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

          visited_local = {}
          paths = resolveLeafPaths(dependent, modifiedSet, visited_local)
          if paths not empty: rewrite this NHG in APPDB (APPDB only; in-memory NHG untouched)
          # paths empty → skip APPDB write and do not mutate in-memory NHG.

      # Generic prune rule
      if not walk_result and dep_entry.depends.size() < 2:
        continue   # single-path miss: prune

      doBackwalk(dependent)

  doBackwalk(startId)
```

### 6.4 Highlights
- **Every dependent performs the direct-nexthop match** (not only the start).
- **Revisits are allowed** — for diamond dependencies, `modifiedSet` records and overwrites; nodes are not skipped.
- **`modifiedSet` is ephemeral scratch state** for the current backwalk; it never mutates the in-memory NHG.
- **The direct-nexthop match reuses `m_nexthop_to_global_RIBNHG`** — no re-parsing of `m_nexthop`.
- **Prune decisions use the generic rule** — no `isGatewayNhg()` type check.

---

## 7. `backwalkPicEdge()` logic

PIC edge shares the **same backwalk framework** as PIC core (the `doBackwalk` in §6.3). The only difference is **where the start-point set comes from**: PIC core uses `prev_resolved_nhg_id`; PIC edge pulls the start set from `m_nexthop_to_vrf_RIBNHG`.

```
backwalkPicEdge(failedNh):
  Look up m_nexthop_to_vrf_RIBNHG[failedNh] → set of RIBNHGEntry* (VRF/VPN scope)
  If empty → silently return (normal case; no VRF/VPN RIB NHG references this nexthop)

  For each entry in the set:
    startId = entry->getRIBIDNum()
    Run the same doBackwalk logic used by backwalkPicCore(startId, failedNh)
```

**Note:** PIC edge covers VRF/VPN-scope RIB NHGs; PIC core covers Global-scope RIB NHGs. The two reverse indices **do not overlap** by construction — no deduplication needed.

---

## 8. Forward-walk twin functions

### 8.1 `resolveLeafPaths()` — filtered

```
resolveLeafPaths(nhgId, modifiedSet, visited):
  # Defensive cycle detection — should not trigger on a proper DAG.
  if nhgId in visited:
    log ERROR("resolveLeafPaths: cycle detected at RIB NHG %u", nhgId)
    return []
  visited.insert(nhgId)

  # Defensive fallback: callers guarantee nhgId ∈ modifiedSet, but if a future
  # refactor calls this unexpectedly, degrade safely.
  if nhgId ∉ modifiedSet:
    return collectAllLeafPaths(nhgId, visited)

  entry = m_rib_nhg_table.getEntry(nhgId)
  myState = modifiedSet[nhgId]

  if entry is a leaf:
    if myState.fully_disabled: return []
    else: return [entry.nexthop + interface]

  results = []
  for dep in entry.depends:
    if myState.enable_group[dep] == false: continue   # branch disabled
    if dep ∈ modifiedSet:
      results += resolveLeafPaths(dep, modifiedSet, visited)   # dep affected, recurse
    else:
      results += collectAllLeafPaths(dep, visited)             # dep healthy, collect all
  return results
```

**Caller contract:** the top-level caller (e.g. from `doBackwalk`) passes an empty `visited` set. `visited` is shared within one forward walk to guard against cycles.

### 8.2 `collectAllLeafPaths()` — unfiltered

```
collectAllLeafPaths(nhgId, visited):
  if nhgId in visited:
    log ERROR("collectAllLeafPaths: cycle detected at RIB NHG %u", nhgId)
    return []
  visited.insert(nhgId)

  entry = m_rib_nhg_table.getEntry(nhgId)
  if entry is null: return []
  if entry is a leaf: return [entry.nexthop + interface]
  results = []
  for dep in entry.depends:
    results += collectAllLeafPaths(dep, visited)
  return results
```

---

## 9. APPDB write

- Target table: `NEXTHOP_GROUP_TABLE` (the SONiC NHG or gateway NHG corresponding to the hit RIB NHG).
- Write path: reuse the existing `NHGMgr` `m_rib_nhg_table->writeToDB(entry)` / `ProducerStateTable` channel.
- **Only overwrite changed fields** (nexthop / ifname / weight / seg_src / vpn_sid, etc.); do not rewrite the entire entry.
- APPDB write failure → log error; no retry (a later normal NHG update overwrites); no rollback.

**Key constraint:** backwalk only writes APPDB; it never mutates the in-memory `RIBNHGEntry`. The in-memory NHG is refreshed by the subsequent normal FRR NHG update.

---

## 10. Error handling and boundaries

### 10.1 Message layer
- Missing / zero-length `FPM_NHA_JSON_STR` NLA → log warning, drop.
- JSON decode failure → log warning, drop.
- Missing required field → log warning, drop.

### 10.2 State consistency
- **Start lookup fails:** `m_rib_nhg_table.getEntry(prev_resolved_nhg_id)` absent → log warning, skip PIC core; **PIC edge still runs**.
- **Dependent lookup fails:** log warning, skip that dependent.

### 10.3 Cycle safety net
- Forward walk's internal `visited set<ribID>` detects a cycle → log ERROR + return `[]`.
- No fixed depth ceiling (`MAX_NHG_RECURSION = 2` is FRR-side serialization capacity, unrelated to runtime walk depth).

### 10.4 Diamond-dependency revisit
- Zebra NHGs form a strict DAG; the same node may be reached via multiple paths.
- **Revisits allowed:** doBackwalk does not skip; modifiedSet records state, and a revisit overwrites and re-triggers upward propagation if the new value differs.

### 10.5 APPDB write failure
- log error; no retry; no rollback.

### 10.6 Concurrency
- fpmsyncd consumes messages single-threadedly; no locking needed.

### 10.7 All-fail but walk continues
- NHG fully disabled → do not write APPDB; `doBackwalk(dependent)` still continues (propagates fully_disabled into upstream enable_group computation).

### 10.8 PIC edge silent case
- `m_nexthop_to_vrf_RIBNHG[failedNh]` has no match → **silently return, no warning**.

### 10.9 Phase 1 gate
- routesync dispatch only rejects `curr_resolved_nhg_id != 0`.
- `prev_resolved_nhg_id == 0` handled naturally inside backwalk.

---

## 11. Files touched

- `fpmsyncd/routesync.h` — declare `onNhtEventMsg`.
- `fpmsyncd/routesync.cpp` — dispatch `RTM_NEWNHTEVENT`.
- `fpmsyncd/nhgmgr.h` — new `NHGMgr` methods + two reverse indices + `NodeState` struct.
- `fpmsyncd/nhgmgr.cpp` — new method impls + `addNHGFull` / `delNHGFull` call `indexNexthopToRIBNHG` / `unindexNexthopToRIBNHG`.

---

## 12. Compatibility

- The new `RTM_NEWNHTEVENT` branch does not affect existing message handling.
- `NEXTHOP_GROUP_TABLE` write format reuses existing fields.
- The reverse indices are pure in-memory structures — no CONFIG_DB / APP_DB compatibility concern.
- An older FRR (no NHT event) → fpmsyncd never receives the message, backwalk is not triggered, behavior degrades to the existing path.

---

## 13. Testing

fpmsyncd UT (algorithm branch coverage, index maintenance, message parsing, error handling) is described in the consolidated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md) §fpmsyncd UT.

---

## Appendix A. Implementation notes

Small gotchas kept out of the design body so it reads cleanly.

- **Class name casing:** the class is `NHGMgr` (all caps), not `NhgMgr` — an easy source of compile typos when adding the new methods.
- **`RTM_NEWNHTEVENT` value lives in two repos:** the constant `6000` is defined both here (fpmsyncd) and in `dplane_fpm_sonic.c` (sonic-buildimage). The two must stay in sync; a mismatch silently drops the event at dispatch. Prefer a shared header if one becomes available from sonic-fib.
- **Use-after-free invariant (see §4.3):** every code path that removes a RIB NHG must call `unindexNexthopToRIBNHG(entry)` **before** `delEntry()`. Reversing the order leaves a dangling `RIBNHGEntry*` in the reverse index, and the next backwalk that dereferences it crashes.
