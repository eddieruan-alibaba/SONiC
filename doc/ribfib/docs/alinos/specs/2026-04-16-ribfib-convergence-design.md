<h1 align="center">RIB/FIB Route Convergence Handling HLD</h1>

# Table of Contents <!-- omit in toc -->
- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions and Abbreviations](#3-definitions-and-abbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
  - [6.1 System Context](#61-system-context)
  - [6.2 NHG Dependency Graph](#62-nhg-dependency-graph)
  - [6.3 Backwalk Infrastructure](#63-backwalk-infrastructure)
- [7. High-Level Design](#7-high-level-design)
  - [7.1 fib\_nhg\_trigger\_node\_quick\_fixup](#71-fib_nhg_trigger_node_quick_fixup)
  - [7.2 fib\_nhg\_back\_walk](#72-fib_nhg_back_walk)
  - [7.3 Walk Spec: fib\_nhg\_walk\_spec\_for\_node\_quick\_fixup](#73-walk-spec-fib_nhg_walk_spec_for_node_quick_fixup)
  - [7.4 Prune Spec: fib\_nhg\_prune\_spec\_for\_node\_quick\_fixup](#74-prune-spec-fib_nhg_prune_spec_for_node_quick_fixup)
  - [7.5 RIBNHGEntry Changes](#75-ribnhgentry-changes)
    - [New Field: m\_resolved\_enable\_group](#new-field-m_resolved_enable_group)
    - [Method Update: getNextHopGroupFields()](#method-update-getnexthopgroupfields)
  - [7.6 Dual-Phase Walk](#76-dual-phase-walk)
  - [7.7 Workflow Diagrams](#77-workflow-diagrams)
    - [NHT Event Processing Flow](#nht-event-processing-flow)
- [8. SAI API](#8-sai-api)
- [9. Configuration and Management](#9-configuration-and-management)
  - [9.1 CLI](#91-cli)
  - [9.2 YANG Model](#92-yang-model)
  - [9.3 REST API](#93-rest-api)
  - [9.4 gNMI](#94-gnmi)
- [10. Warmboot and Fastboot Design Impact](#10-warmboot-and-fastboot-design-impact)
- [11. Restrictions and Limitations](#11-restrictions-and-limitations)
- [12. Testing Requirements and Design](#12-testing-requirements-and-design)
  - [12.1 Unit Tests](#121-unit-tests)
  - [12.2 Test Topology 1: Global Table Recursive Routes](#122-test-topology-1-global-table-recursive-routes)
  - [12.3 Test Topology 2: Global Table Recursive Routes (Variant)](#123-test-topology-2-global-table-recursive-routes-variant)
  - [12.4 Test Topology 3: SRv6 VPN Case](#124-test-topology-3-srv6-vpn-case)
- [13. Open/Action Items](#13-openaction-items)

---

# 1. Revision

| Rev | Date | Author | Change Description |
|:----|:------|:-------|:-------------------|
| 0.1 | 2026-04-16 | Eddie Ruan, Lingyu Zhang, Songnan Lin, Yuqing Zhao | Initial route convergence handling design |
| 0.2 | 2026-04-16 | (Socratic Exploration) | Refined: starting node behavior, null safety, depth limit, terminology, concurrent NHT, Phase 2 test scoping |

# 2. Scope

This document describes the route convergence handling mechanism within the SONiC FIB block (fpmsyncd). It covers the backwalk infrastructure for traversing NHG dependency graphs and the quick fixup mechanism for rapidly disabling failed paths upon NHT (Nexthop Tracking) events.

**In scope:**
- Backwalk infrastructure in `RIBNHGTable` (fpmsyncd)
- Quick fixup walk/prune spec callbacks
- `m_resolved_enable_group` tracking in `RIBNHGEntry`
- PIC Core handling (Zebra NHG chain backwalk)
- PIC Edge handling (SONiC NHG table lookup via NEXTHOP->SONIC NHG ID table)
- Test cases for global table and SRv6 VPN topologies

**Out of scope:**
- FRR-side changes (resolve-through/resolve-via in `zebra_dplane_ctx`, NHT trigger to dplane)
- Warm reboot NHG ID persistence
- SONiC NHG table walk mechanism (future work)
- CLI for FIB internal table display
- Changes to orchagent or syncd

# 3. Definitions and Abbreviations

| Term | Meaning |
|------|---------|
| RIB | Routing Information Base |
| FIB | Forwarding Information Base |
| NHG | Next Hop Group |
| NHT | Nexthop Tracking -- Zebra's mechanism for monitoring nexthop reachability |
| PIC | Prefix Independent Convergence |
| PIC Core | Convergence at the NHG chain level (updating NHGs that resolve through a failed intermediate NHG) |
| PIC Edge | Convergence at the SONiC NHG / gateway NHG level (updating NHGs that directly contain a failed nexthop address) |
| Resolved NHG | NHG resolved by Zebra containing nexthops with final outgoing interfaces and addresses |
| Original NHG | NHG information received by Zebra from protocol clients (unresolved) |
| Resolve Through | NHG "A" resolves through NHG "B" if A contains a nexthop resolved via B. A is in B's "resolve through" (dependents) list |
| Resolve Via | NHG "B" is in NHG "A"'s "resolve via" (depends) list |
| Backwalk | Traversal from an NHG to its dependents (parent direction in the resolve-through graph) |
| Forward Walk | Traversal from an NHG to its depends (child direction in the resolve-via graph) |
| Walk Spec | Callback function that determines what action to perform at each NHG node during traversal |
| Prune Spec | Callback function that determines whether to stop the backwalk at a given node |
| Gateway NHG | A SONiC-created NHG containing only forwarding information (also called PIC NHG), used in SRv6 VPN cases |

# 4. Overview

In the current SONiC architecture, orchagent handles local port-down events by rapidly updating failed load-balance members. For all other failure scenarios (remote nexthop withdrawal, intermediate route withdrawal), FRR creates a new NHG and migrates prefixes sequentially. The traffic loss window is proportional to the number of affected prefixes.

The RIB/FIB route convergence handling introduces a backwalk infrastructure within fpmsyncd. When Zebra detects a nexthop becoming unreachable via NHT, it sends an event to fpmsyncd containing the impacted nexthop address and its resolved NHG ID. fpmsyncd then:

1. **Phase 1 (PIC Core)**: Initiates a backwalk from the resolved NHG through the Zebra NHG chain, disabling failed paths in all dependent NHGs and writing updated entries to APPDB.
2. **Phase 2 (PIC Edge)**: Looks up the nexthop address in the NEXTHOP->SONIC NHG ID table and updates affected SONiC NHGs (gateway NHGs).

This rapid fixup minimizes traffic loss by completing NHG updates in a single traversal, independent of the number of prefixes referencing those NHGs.

# 5. Requirements

| ID | Requirement | Priority |
|----|-------------|----------|
| R1 | Upon NHT event, all NHGs containing (directly or recursively) the failed nexthop must be updated within a single backwalk pass | Must |
| R2 | NHGs not affected by the failed nexthop must NOT be updated (no unnecessary APPDB writes) | Must |
| R3 | Backwalk must handle arbitrary NHG dependency depth without infinite loops | Must |
| R4 | Walk and prune spec must be pluggable for future extensibility | Should |
| R5 | Quick fixup must be idempotent -- repeated NHT events for the same nexthop produce the same result | Must |
| R6 | PIC Core (Zebra NHG chain) and PIC Edge (SONiC NHG table) must be handled in separate phases | Must |
| R7 | For SRv6 VPN cases, backwalk must stop at gateway NHGs (prune spec) | Must |
| R8 | If an NHG has a single path that must be disabled, skip APPDB update (do not write empty NHG) | Must |
| R9 | Unit test coverage >= 80% for all new code | Must |
| R10 | Support IPv4 and IPv6 nexthop addresses | Must |

# 6. Architecture Design

## 6.1 System Context

```
                    +------------------+
                    |     Zebra (RIB)  |
                    |                  |
                    |  NHT monitoring  |
                    +--------+---------+
                             |
                    NHT event (nexthop_addr, resolved_nhg_id, original_nhg_id)
                             |
                             v
                    +------------------+
                    |   fpmsyncd       |
                    |                  |
                    |  +------------+  |
                    |  | FIB Block  |  |
                    |  |            |  |
                    |  | RIBNHGTable|  |       +----------+
                    |  | (Zebra NHG |--+------>|  APPDB   |
                    |  |  chain)    |  |       +-----+----+
                    |  |            |  |             |
                    |  | SONiC NHG  |  |             v
                    |  | table      |  |       +----------+
                    |  +------------+  |       | orchagent|
                    +------------------+       +----------+
```

The FIB block within fpmsyncd maintains two key tables:
- **SONiC Zebra NHG table**: Maps Zebra NHG IDs to `RIBNHGEntry` objects containing the `zebra_dplane_ctx` and SONiC metadata. This table mirrors Zebra's NHG hash table.
- **SONiC NHG table**: Stores SONiC-created NHGs (gateway NHGs for SRv6 VPN PIC).

## 6.2 NHG Dependency Graph

NHGs form a directed acyclic graph (DAG) through "resolve through" and "resolve via" relationships:

```
   NHG 239 (fc06::2, Ethernet12)    NHG 244 (fc08::2, Ethernet4)
         \                              /
          \                            /
           +--------+   +------------+
                    |   |
                  NHG 243  (resolved NHG for 2064:100::1d and 2064:200::1e)
                   /    \
                  /      \
            NHG 260      NHG 265
          (via 2064:     (via 2064:
          100::1d)       200::1e)
              |               |
              |               |
         NHG 264         NHG 267
       (via 1::1,       (via 1::1)
        via 2::2)
              |
         NHG 275
       (via 3::3,
        via 4::4)
```

- **Depends** (resolve via): downward edges -- NHG 264 depends on NHG 260 and NHG 265
- **Dependents** (resolve through): upward edges -- NHG 243's dependents include NHG 260 and NHG 265

A backwalk traverses upward (from NHG 243 to NHG 260, 265, then to 264, 267, etc.).

## 6.3 Backwalk Infrastructure

The backwalk infrastructure is a generalized graph traversal framework:

```
fib_nhg_trigger_node_quick_fixup(nexthop_addr, resolved_nhg_id, original_nhg_id)
    |
    v
Initialize fib_nhg_walking_ctx:
  - visited_node_set: {}
  - walk_spec: fib_nhg_walk_spec_for_node_quick_fixup
  - prune_spec: fib_nhg_prune_spec_for_node_quick_fixup
  - nexthop_address: nexthop_addr
    |
    v
fib_nhg_back_walk(resolved_nhg_id, ctx)
    |
    +---> getEntry(id)
    |     Add id to visited_node_set
    |
    +---> walk_spec(entry)  --> updates entry if relevant
    |     Returns: updated (bool)
    |
    +---> prune_spec(entry, updated) --> decides if walk continues
    |     Returns: prune (bool)
    |
    +---> if !prune:
            for each dependent_id in entry.dependents:
                if dependent_id not in visited_node_set:
                    fib_nhg_back_walk(dependent_id, ctx)
```

# 7. High-Level Design

## 7.1 fib_nhg_trigger_node_quick_fixup

**Entry point** invoked upon receiving an NHT event from Zebra.

**Parameters:**
- `nexthop_address`: The IPv4/IPv6 address of the impacted nexthop
- `resolved_nhg_id`: The resolved NHG ID associated with the impacted nexthop
- `original_nhg_id`: The original NHG ID (reserved for future use)

**Behavior:**
1. Initialize `fib_nhg_walking_ctx` with empty visited set, walk/prune spec functions, and the nexthop address.
2. Call `fib_nhg_back_walk(resolved_nhg_id, ctx)` to initiate PIC Core traversal.
3. (Future) Use nexthop address to trigger SONiC NHG table lookup for PIC Edge traversal.

The `original_nhg_id` is accepted but not currently used for initiating backwalk. This parameter enables future extensions for SRv6 VPN-specific convergence handling.

## 7.2 fib_nhg_back_walk

**Added to**: `RIBNHGTable` class

**Parameters:**
- `uint32_t id`: Zebra NHG ID to start or continue the walk from
- `fib_nhg_walking_ctx&`: Walking context (visited set, spec functions, nexthop address)

**Algorithm:**
1. Retrieve `RIBNHGEntry` via `getEntry(id)`. **If null (NHG was deleted between events), log warning and return** -- do not abort the walk.
2. Add `id` to `visited_node_set`.
3. **Depth check**: If current recursion depth exceeds 1000, log warning and return. This is defense-in-depth against pathological configurations.
4. Execute `walk_spec(entry)` -- may update the entry (disable failed paths, write to APPDB).
5. Execute `prune_spec(entry, walk_result)` -- determines if traversal continues.
6. If not pruned: iterate `entry.dependents`, skip visited nodes, recursively call `fib_nhg_back_walk()` on each.

**Cycle prevention**: The `visited_node_set` ensures each NHG is visited at most once per backwalk invocation.

**Starting node behavior**: The starting node (resolved_nhg_id from NHT event, e.g., NHG 243) is visited and walk_spec is executed on it. For quick fixup, the walk_spec typically returns `false` for the starting node (it is the trigger NHG, not a target for path disabling). The backwalk continues to the starting node's dependents regardless of the walk_spec result -- prune_spec determines whether to continue.

**Concurrent NHT events**: Since the backwalk is synchronous within fpmsyncd's event loop, concurrent NHT events (e.g., two nexthops failing simultaneously) are serialized. The second backwalk sees the state modifications from the first. This is correct behavior -- the second walk may find fewer nodes to update if they overlap.

**Traversal vs. state propagation direction**: The backwalk traverses UPWARD through "dependents" (resolve-through) edges. At each visited node, the walk_spec reads DOWNWARD through "depends" (resolve-via) edges to check children's `m_resolved_enable_group` state. This is not a contradiction -- the traversal direction determines which nodes to visit, while the state propagation reads from already-visited children (guaranteed by DFS ordering).

## 7.3 Walk Spec: fib_nhg_walk_spec_for_node_quick_fixup

**Input**: Current `RIBNHGEntry` node, walking context (nexthop address)

**Output**: `bool` -- `true` if fixup was applied, `false` if node is irrelevant

**Algorithm:**
1. Retrieve the IDs this entry depends on (resolve-via list).
2. Determine relevance:
   - Check if any of this entry's gateway addresses match the impacted nexthop.
   - Check if any dependent's `m_resolved_enable_group` contains newly disabled paths not yet reflected in this entry's `m_resolved_enable_group`.
3. If relevant, iterate through depends list:
   - Retrieve each dependent `RIBNHGEntry`.
   - Inspect its `m_resolved_enable_group` for disabled paths.
   - Mark corresponding paths in the current entry's `m_resolved_enable_group` as disabled.
4. Call `getNextHopGroupFields()` to regenerate APPDB output with only enabled paths, then call `writeToDB()`.
5. **Exception**: If the entry has only one remaining path and it must be disabled, skip the APPDB write.

## 7.4 Prune Spec: fib_nhg_prune_spec_for_node_quick_fixup

**Input**: Current `RIBNHGEntry` node, walk spec return value (`bool`)

**Output**: `bool` -- `true` to stop backwalk from this node, `false` to continue

**Algorithm:**
1. If walk spec returned `false` (node not updated): prune (stop).
2. If the node represents a SONiC NHG object (gateway NHG): prune (stop). This handles SRv6 VPN cases where the walk should not propagate beyond gateway NHGs.
3. Otherwise: continue (do not prune).

## 7.5 RIBNHGEntry Changes

### New Field: m_resolved_enable_group

```cpp
/*
 * Resolved group of the entry.
 * Contains <ribID, bool> pairs to indicate if the resolved NHG is enabled.
 * - All paths are set as enabled upon Add/Update events from FRR.
 * - Paths are marked as disabled during the backwalk process.
 */
unordered_map<uint32_t, bool> m_resolved_enable_group;
```

This map is keyed by the RIB NHG ID of each resolved member. On NHG install/update events from FRR, all entries are reset to `true` (enabled). During backwalk quick fixup, failed paths are set to `false` (disabled).

### Method Update: getNextHopGroupFields()

Modified to check `m_resolved_enable_group` before including a path in the output:
- If a path's RIB ID maps to `false` in `m_resolved_enable_group`, that path is excluded from the generated APPDB fields.
- This ensures only enabled (non-failed) paths are written to APPDB during quick fixup.

## 7.6 Dual-Phase Walk

Route convergence handling operates in two phases:

**Phase 1 -- PIC Core (Zebra NHG Chain):**
- Start from `resolved_nhg_id` provided by NHT event
- Backwalk through `RIBNHGEntry` dependents list
- Disable failed paths and update APPDB entries
- Handles cases where intermediate NHGs (e.g., 2064:100::1d) are withdrawn

**Phase 2 -- PIC Edge (SONiC NHG Table):**
- Use nexthop address to look up NEXTHOP->SONIC NHG ID table
- Returns a list of SONiC NHGs containing the nexthop
- Update each SONiC NHG; for SRv6 type NHGs, stop propagation
- Handles cases where gateway NHGs need direct nexthop removal

> **Note**: Phase 2 (SONiC NHG table walk) is marked as TODO in the current LLD and will be implemented in a follow-up change.

## 7.7 Workflow Diagrams

### NHT Event Processing Flow

```
NHT Event (nexthop=2064:100::1d, resolved_nhg=243)
    |
    v
fib_nhg_trigger_node_quick_fixup()
    |
    +--- Phase 1: fib_nhg_back_walk(243, ctx)
    |       |
    |       +---> NHG 243: walk_spec -> not direct match, but starting point
    |       |     dependents: [260, 265]
    |       |
    |       +---> NHG 260 (via 2064:100::1d): walk_spec -> relevant, single path
    |       |     disable 2064:100::1d path -> skip APPDB (single path)
    |       |     dependents: [264]
    |       |
    |       +---> NHG 265 (via 2064:200::1e): walk_spec -> irrelevant
    |       |     prune: yes (not updated)
    |       |
    |       +---> NHG 264 (via 1::1, via 2::2): walk_spec -> relevant
    |       |     1::1 depends on 260 (disabled) -> disable 1::1 path
    |       |     2::2 depends on 265 (still enabled) -> keep
    |       |     writeToDB() with remaining path 2::2
    |       |     dependents: [275]
    |       |
    |       +---> NHG 275 (via 3::3, via 4::4):
    |             3::3 depends on 264 (partially disabled) -> check
    |             4::4 depends on 264 -> already handled
    |             (continue recursively)
    |
    +--- Phase 2: SONiC NHG table lookup (future)
```

# 8. SAI API

No SAI API changes are required. The route convergence handling operates entirely within fpmsyncd, updating APPDB entries. Orchagent and syncd consume these APPDB entries through existing SAI programming paths.

# 9. Configuration and Management

## 9.1 CLI

The FIB block is enabled/disabled via a global knob at device initialization time (not modifiable at runtime). This knob is defined in the parent RIB/FIB HLD.

Future CLI additions for displaying FIB internal tables (Zebra NHG table, SONiC NHG table, backwalk statistics) are planned but out of scope for this change.

## 9.2 YANG Model

No YANG model changes are required for this change. The FIB enable/disable knob's YANG definition is part of the parent RIB/FIB HLD.

## 9.3 REST API

No REST API changes.

## 9.4 gNMI

No gNMI changes.

# 10. Warmboot and Fastboot Design Impact

The route convergence handling mechanism operates on in-memory data structures within fpmsyncd. During warm reboot:

- The `m_resolved_enable_group` state is transient and does not need persistence. Upon warm reboot recovery, FRR will re-send all NHG events, and the `m_resolved_enable_group` for each entry will be reset to all-enabled.
- The backwalk infrastructure itself is stateless between invocations and requires no warm reboot consideration.
- The NHG ID persistence (Zebra NHG ID to SONiC NHG ID mapping) is handled separately by the parent RIB/FIB design and is not affected by this change.

**Impact**: None. The convergence handling is purely reactive and its state is reconstructed from NHG events during recovery.

# 11. Restrictions and Limitations

1. **Phase 2 (PIC Edge) not implemented**: The SONiC NHG table walk for PIC Edge cases is deferred to a follow-up change. Only PIC Core handling is covered in the initial implementation. **SRv6 VPN PIC Edge convergence requires Phase 2 to be effective** -- without Phase 2, gateway NHG updates are not propagated.
2. **Original NHG ID not used for backwalk**: The `original_nhg_id` parameter is accepted but not used to initiate backwalk. This is reserved for future SRv6 VPN-specific convergence enhancements.
3. **Single-path NHG handling**: When an NHG has only one path and it must be disabled, the APPDB entry is not updated. Traffic using this NHG will continue to be forwarded to the failed path until FRR reconverges and sends a new NHG update. **This is no worse than the status quo (no-PIC behavior)** because there is no alternative path to switch to.
4. **No CLI visibility**: Internal backwalk state and statistics are not exposed via CLI in this change.
5. **FRR dependency**: The mechanism requires FRR to provide "resolve through" and "resolve via" information in `zebra_dplane_ctx`, and NHT triggers to the data plane. These are Phase 1 FRR tasks.
6. **Partial backwalk on failure**: If `getEntry()` returns null or `writeToDB()` fails during a backwalk, the walk logs a warning and continues to the next node. Partially updated state is strictly better than no update. FRR reconvergence will eventually fix any remaining stale entries.
7. **Defensive depth limit**: Backwalk recursion is limited to 1000 levels. This is defense-in-depth against pathological static route configurations creating extremely deep NHG chains.

# 12. Testing Requirements and Design

Testing uses model-based testing infrastructure to generate unit test cases. Test cases target >= 80% coverage in SWSS.

## 12.1 Unit Tests

Unit tests validate the following components:
- `fib_nhg_back_walk()` traversal correctness and cycle prevention
- `fib_nhg_walk_spec_for_node_quick_fixup` path relevance detection and disable logic
- `fib_nhg_prune_spec_for_node_quick_fixup` pruning decisions
- `m_resolved_enable_group` state transitions
- `getNextHopGroupFields()` output filtering for disabled paths
- APPDB write correctness (only enabled paths)

## 12.2 Test Topology 1: Global Table Recursive Routes

**Setup**: 2064:100::1d/128 and 2064:200::1e/128 via eBGP with two ECMP paths (fc06::2, fc08::2). Static routes create multi-level recursion:
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 2064:200::1e
ipv6 route 3::3/128 1::1
ipv6 route 3::3/128 2::2
ipv6 route 4::4/128 1::1
```

| Test Case | Trigger | Expected Updates | Expected No-Updates |
|-----------|---------|-----------------|---------------------|
| Local Failure (fc06::2 withdrawn) | NHT: fc06::2 + NHG ID | All NHGs except fc08::2's NHG | fc08::2's NHG |
| Local Failure (idempotency) | NHT: fc06::2 + NHG ID (repeat after 2064:100::1d update) | No further NHG updates | All |
| Remote Failure 1 (2064:100::1d withdrawn) | NHT: 2064:100::1d + NHG ID | NHGs for 2064:100::1d, 1::1, 4::4 | NHGs for 2::2, 3::3 |
| Remote Failure 2 (1::1 withdrawn) | NHT: 1::1 + NHG ID | NHGs for 1::1, 4::4 | NHGs for 2064:100::1d, 2::2, 3::3 |

## 12.3 Test Topology 2: Global Table Recursive Routes (Variant)

**Setup**: Same BGP routes. Different static routes with direct nexthop references:
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 fc06::2
ipv6 route 3::3/128 fc08::2
ipv6 route 4::4/128 2::2
ipv6 route 4::4/128 3::3
```

| Test Case | Trigger | Expected Updates | Expected No-Updates |
|-----------|---------|-----------------|---------------------|
| Local Failure (fc06::2 withdrawn) | NHT: fc06::2 + NHG ID | All NHGs: 2064:100::1d, 2064:200::1e, 1::1, 2::2, 3::3, 4::4 | -- |
| Remote Failure 1 (2064:100::1d withdrawn) | NHT: 2064:100::1d + NHG ID | NHGs for 2064:100::1d, 1::1 | NHGs for 2064:200::1e, 2::2, 3::3, 4::4 |
| Remote Failure 2 (1::1 withdrawn) | NHT: 1::1 + NHG ID | NHG for 1::1 | All others |
| Remote Failure 3 (2::2 withdrawn) | NHT: 2::2 + NHG ID | NHGs for 2::2, 4::4 | All others |

## 12.4 Test Topology 3: SRv6 VPN Case

**Setup**: Same BGP routes. SRv6 VPN routes via 2064:100::1d and 2064:200::1e with VPN SIDs:
```
B> 192.100.0.10/32 via 2064:100::1d (seg6 fd00:201:201:1::)
                   via 2064:200::1e (seg6 fd00:202:202:2::)
```

| Test Case | Trigger | Expected Updates | Expected No-Updates |
|-----------|---------|-----------------|---------------------|
| Local Failure (fc06::2 withdrawn) | NHT: fc06::2 + NHG ID | NHGs for 2064:100::1d, 2064:200::1e | SONiC NHGs (gateway NHGs) |
| Remote Failure (2064:100::1d withdrawn) | NHT: 2064:100::1d + NHG ID | SONiC NHG update | NHG for 2064:200::1e |

> **Note**: The "Remote Failure" test case requires Phase 2 (PIC Edge / SONiC NHG table walk) to be implemented. Until Phase 2 is available, this test case should be marked as expected-failure or deferred.

# 13. Open/Action Items

| # | Item | Status | Owner |
|---|------|--------|-------|
| 1 | Implement SONiC NHG table walk mechanism (Phase 2 / PIC Edge) | Open | TBD |
| 2 | Determine CLI approach for FIB internal table display (extend vtysh vs. new tool) | Open | TBD |
| 3 | FRR Phase 1 changes: resolve-through/resolve-via in zebra_dplane_ctx | Open | FRR Team |
| 4 | FRR Phase 1 changes: NHT trigger to dplane | Open | FRR Team |
| 5 | Validate backwalk performance under large-scale NHG graphs (10K+ NHGs) | Open | TBD |
| 6 | Define behavior for NHG updates racing with backwalk fixups | Open | TBD |
| 7 | Original NHG ID usage for SRv6 VPN-specific backwalk | Open | TBD |
