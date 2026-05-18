<h1 align="center">RIB FIB Route Convergence Handling LLD </h1>

# Table of Contents <!-- omit in toc -->
- [1. Problem Statements](#1-problem-statements)
- [2. General Code Generation Rules](#2-general-code-generation-rules)
  - [1. Language Standard](#1-language-standard)
  - [2. Header Include Paths: Export vs. Internal Files](#2-header-include-paths-export-vs-internal-files)
- [3. FRR Modifications](#3-frr-modifications)
  - [RNH event information](#rnh-event-information)
    - [Core RNH Tracking Fields](#core-rnh-tracking-fields)
    - [Change Detection Logic](#change-detection-logic)
  - [Dplane Integration: New Event Type \& Context Structure](#dplane-integration-new-event-type--context-structure)
    - [New Dplane Operation Enum](#new-dplane-operation-enum)
    - [Extended Dplane Context](#extended-dplane-context)
    - [RNH Info Structure Definition](#rnh-info-structure-definition)
    - [Event Generation Workflow](#event-generation-workflow)
- [4. FPM Message Serialization (SONiC Integration)](#4-fpm-message-serialization-sonic-integration)
- [5. SONiC-fib Enhancements for NHT events](#5-sonic-fib-enhancements-for-nht-events)
  - [NhtEvent JSON Schema](#nhtevent-json-schema)
    - [Example: nexthop becomes unresolved (Phase 1 trigger)](#example-nexthop-becomes-unresolved-phase-1-trigger)
  - [Generated Code](#generated-code)
  - [C API for FRR Integration](#c-api-for-frr-integration)
  - [Data Flow](#data-flow)
- [6. FPMsyncd Modifications](#6-fpmsyncd-modifications)
  - [Existing NHG MGR codes](#existing-nhg-mgr-codes)
  - [`fib_nhg_trigger_node_quick_fixup()`](#fib_nhg_trigger_node_quick_fixup)
    - [Part 1: Global Table Context Backwalk](#part-1-global-table-context-backwalk)
    - [Part 2: VPN Context Backwalk](#part-2-vpn-context-backwalk)
  - [`fib_nhg_back_walk()`](#fib_nhg_back_walk)
    - [Parameters](#parameters)
    - [Design \& Extensibility](#design--extensibility)
    - [Execution Flow](#execution-flow)
      - [Recursion Flowchart](#recursion-flowchart)
  - [Changes for Part 1](#changes-for-part-1)
    - [Changes in  `class RIBNHGEntry`](#changes-in--class-ribnhgentry)
    - [Method Update: `RIBNHGEntry::getNextHopGroupFields()`](#method-update-ribnhgentrygetnexthopgroupfields)
      - [Algorithm Summary](#algorithm-summary)
        - [📦 Entry Types](#-entry-types)
        - [⚙️ Operating Modes](#️-operating-modes)
        - [🚦 Marker Behavior](#-marker-behavior)
        - [📉 Slot Consumption](#-slot-consumption)
    - [`fib_nhg_walk_spec_for_node_quick_fixup()`](#fib_nhg_walk_spec_for_node_quick_fixup)
    - [`fib_nhg_prune_spec_for_node_quick_fixup()`](#fib_nhg_prune_spec_for_node_quick_fixup)
    - [Part 1's call flow](#part-1s-call-flow)
  - [Changes for Part 2](#changes-for-part-2)
    - [Motivation](#motivation)
    - [Modifications to `RIBNHGTable` Class](#modifications-to-ribnhgtable-class)
    - [Backwalk for VPN-Scoped RIBNHGEntries](#backwalk-for-vpn-scoped-ribnhgentries)
    - [`fib_nhg_walk_spec_for_node_quick_fixup_sonic_nhg()`](#fib_nhg_walk_spec_for_node_quick_fixup_sonic_nhg)
    - [`fib_nhg_prune_spec_for_node_quick_fixup_sonic_nhg()`](#fib_nhg_prune_spec_for_node_quick_fixup_sonic_nhg)
    - [Part 2's call flow](#part-2s-call-flow)
    - [One variation for part 1](#one-variation-for-part-1)
- [7. Examples](#7-examples)
  - [Test Topology 1 Global table recursive routes](#test-topology-1-global-table-recursive-routes)
    - [Local Failure fc06::2 withdrawn](#local-failure-fc062-withdrawn)
    - [Remote Failure 1 (2064:200::1e withdrawn)](#remote-failure-1-20642001e-withdrawn)
    - [Remote Failure 2 (1::1 withdrawn)](#remote-failure-2-11-withdrawn)
    - [Summary of corrected Topology 1 expectations](#summary-of-corrected-topology-1-expectations)
  - [Test Topology 2 Global table recursive routes](#test-topology-2-global-table-recursive-routes)
    - [Local Failure (`fc06::2` withdrawn)](#local-failure-fc062-withdrawn-1)
    - [Remote Failure 1 (`2064:100::1d` withdrawn)](#remote-failure-1-20641001d-withdrawn)
    - [Remote Failure 2 (`1::1` withdrawn)](#remote-failure-2-11-withdrawn-1)
    - [Remote Failure 3 (`2::2` withdrawn)](#remote-failure-3-22-withdrawn)
    - [Summary of corrected Topology 2 expectations:](#summary-of-corrected-topology-2-expectations)
  - [Test Topology 3 SRv6 VPN case](#test-topology-3-srv6-vpn-case)
    - [Test local failure](#test-local-failure)
    - [Test remote failure](#test-remote-failure)
- [8. Test cases](#8-test-cases)
  - [FRR topotest](#frr-topotest)
  - [sonic-fib unit test](#sonic-fib-unit-test)
  - [sonic-swss unit test](#sonic-swss-unit-test)
  - [sonic-mgmt system level tests](#sonic-mgmt-system-level-tests)
    - [7-Node Topology Key Connections](#7-node-topology-key-connections)
    - [Route Origins](#route-origins)
    - [Helper Functions](#helper-functions)
      - [1. apply\_config\_cmmds\_to\_vtysh(nbrhost, cmd\_list)](#1-apply_config_cmmds_to_vtyshnbrhost-cmd_list)
      - [2. Record Collection](#2-record-collection)
      - [3. APPDB Assertion Helpers](#3-appdb-assertion-helpers)
    - [Failure Triggers](#failure-triggers)
    - [Recovery](#recovery)
    - [Assertions](#assertions)
    - [Test Cases](#test-cases)
      - [Topology 1 Static Routes (applied on PE3)](#topology-1-static-routes-applied-on-pe3)
      - [Topology 2 Static Routes (applied on PE3)](#topology-2-static-routes-applied-on-pe3)
      - [Test workflow per test case](#test-workflow-per-test-case)
- [9. Appendix](#9-appendix)
  - [Key refinements identified during LLM-assisted brainstorming:](#key-refinements-identified-during-llm-assisted-brainstorming)
  - [First Round Code Genearted](#first-round-code-genearted)
  - [Couple issues fixed for compiling](#couple-issues-fixed-for-compiling)
  - [Unit Test issues:](#unit-test-issues)
    - [Cached bogus pointer](#cached-bogus-pointer)
    - [Misunderstand m\_nexthop](#misunderstand-m_nexthop)
    - [Not handle IBNHGEntry::getNextHopGroupFields()'s change paths based on m\_resolved\_enable\_group](#not-handle-ibnhgentrygetnexthopgroupfieldss-change-paths-based-on-m_resolved_enable_group)


# 1. Problem Statements
In the current SONiC architecture, orchagent mitigates traffic loss during local port-down events by rapidly removing failed load-balancing members. However, in other failure scenarios, FRR generates a new Next Hop Group (NHG) and migrates dependent prefixes sequentially, causing the traffic loss window to scale linearly with the number of affected prefixes.

To eliminate this prefix-dependent convergence delay, we propose a Prefix-Independent Convergence (PIC) mechanism. When fpmsyncd receives a Next Hop Tracking (NHT) update from zebra, it triggers a targeted reconciliation process. By performing a dependency backwalk from the invalidated NHG, the system identifies all reliant NHGs and applies coordinated updates, bypassing sequential prefix migration. The primary objective is to immediately prune failed forwarding paths to minimize the traffic loss window, allowing the control plane to subsequently recalculate and install optimal routes in the background.

The implementation comprises Four core components:

* **FRR Modification**: Route all NHT events to the FPM interface exclusively through the dplane subsystem.
* **SONiC FPM modifications**: convert NHT events to fpm message.
* **sonic-fib Enhancement**: Introduce a new data schema to map NHT events, capturing both the affected next hop and its corresponding NHG identifier.
* **fpmsyncd Logic**:  Traverse all dependent NHGs and rapidly reconcile the affected forwarding state.


# 2. General Code Generation Rules
This Low-Level Design (LLD) is deliberately authored with implementation-grade specificity to function as a high-fidelity prompt for LLM-powered code generation tools (e.g., Qoder CLI: https://qoder.com/en/cli). The goal is to enable reliable, specification-to-code automation with minimal ambiguity.

## 1. Language Standard
  * C++14 is the required baseline for all SONiC C++ components (sonic-swss, sonic-fib, swss-common, etc.).
  * Do not use C++17/20 features unless the target branch explicitly migrates.
## 2. Header Include Paths: Export vs. Internal Files
When generating codes for Debian-packaged components, distinguish between public export headers and internal implementation files:'**
| File Type | Purpose | Include Path Style | Example |
|-----------|---------|-------------------|---------|
| **Export Header** | Installed to `/usr/include/...` for external consumers | Use **installed/public paths** (no `src/` prefix) | `#include "nexthopgroupfull.h"` |
| **Internal Source** | Compiled within the package; not installed | Use **relative build-tree paths** (e.g., `src/`) | `#include "src/nexthopgroupfull.h"` |

**Examples**:
```
// ✅ nexthopgroupfull_json.h (export header — installed)
#include "nexthopgroupfull.h"  // Resolves to /usr/include/nexthopgroupfull.h

// ✅ nexthopgroup_capi.cpp (internal implementation)
#include "src/nexthopgroupfull.h"
#include "src/nexthopgroupfull_json.h"
#include "src/c_nexthopgroupfull.h"
#include "src/nexthopgroup_debug.h"
```

Critical: Mixing these styles causes build failures:
* Export headers using src/ paths → break downstream consumers (file not found at install time).
* Internal files using flat paths → may conflict with system headers or fail in-tree builds.

# 3. FRR Modifications
This section outlines modifications to FRRouting (FRR) to enhance Next Hop Tracking (NHT) event propagation from Zebra to the dplane, enabling fpmsyncd to respond proactively to nexthop changes and minimize traffic loss during convergence.

## RNH event information
### Core RNH Tracking Fields
Each rnh (Route Next Hop) structure maintains the following state for change detection:

| Field | Description |Storage Location |
|:---|:---|:---|
| Tracked Prefix| The prefix being monitored for nexthop changes | rnh->node->p |
| Resolved Route Prefix | The prefix of the route currently resolving the tracked prefix | rnh->resolved_route |
| Resolution State | The active route_entry used to resolve the tracked prefix | rnh->state |

Memory Lifetime Warning:
The function copy_state() frees the existing resolution state (rnh->state) before assigning the new state. Therefore:
❌ Do NOT cache rnh->state as a pointer before calling copy_state() and dereference it afterward. The pointer becomes dangling.
✅ DO extract and cache any required fields (e.g., nhg_id) before invoking copy_state(), since primitive values remain valid independent of memory lifecycle.

### Change Detection Logic
The function zebra_rnh_eval_nexthop_entry() in https://github.com/FRRouting/frr/blob/master/zebra/zebra_rnh.c#L785 evaluates whether an incoming route update affects an RNH by comparing:
* Whether the incoming route_node *prn matches the RNH's current resolved prefix
* Whether the incoming route_entry *re differs from the RNH's cached state

```
static void zebra_rnh_eval_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
					 int force, struct route_node *nrn,
					 struct rnh *rnh,
					 struct route_node *prn,
					 struct route_entry *re,
					 bool route_entry_queued)
```

When a state change is detected (state_changed == true), Zebra must propagate the following context to dplane to enable informed FIB NHT updates.

The purpose for zebra sends NHT event to fpmsyncd is to give enought information which informs that the current nexthop its holding will make some changes. So fpmsyncd's FIB module could response this event properly for reducing traffic loss window. Therefore, the following information would need to pass down to dplane when state_changed is set.

| Field | Purpose | Unresolved Value |
|:---|:---|:---|
| rnh_prefix | Identifies affected NHGs | — |
| prev_resolved_prefix | Previous resolving route prefix | 0.0.0.0/0 (or equivalent zero prefix)
| prev_resolved_nhg_id | Previous resolving NHG identifier | 0 |
| curr_resolved_prefix | Current resolving route prefix | 0.0.0.0/0 |
| curr_resolved_nhg_id | Current resolving NHG identifier | 0 |

Note: Currently, all related context is sent to dplane. Processing follows a phased approach:
* Phase 1: Trigger backwalk when an RNH prefix becomes unresolvable. Walk from the previous resolved NHG ID to prune failed paths from dependent NHGs.
* Future: Extend handling to cover other caases.

In `zebra_rnh_resolve_nexthop_entry()`, route resolution may temporarily fail if a route entry is still queued and awaiting processing. To prevent unnecessary traffic disruption, we suppress the NHT dplane event (but NOT the entire evaluation) during this window.

**Approach**: Add a new output parameter `bool *route_entry_queued` to `zebra_rnh_resolve_nexthop_entry()`. The function tracks two internal flags (`saw_selected`, `saw_queued`) and sets `*route_entry_queued = true` when all selected candidate routes were skipped due to `ROUTE_ENTRY_QUEUED`. Callers that don't need this information pass `NULL`.

**Important**: The suppression only gates the `dplane_nht_event_update()` call inside `zebra_rnh_eval_nexthop_entry()`. The rest of the evaluation (state change detection, client notifications, pseudowire processing) proceeds normally. This avoids blocking protocol client notifications when routes are queued.

```c
// In zebra_rnh_evaluate_entry():
bool route_entry_queued = false;
re = zebra_rnh_resolve_nexthop_entry(zvrf, afi, nrn, rnh, &prn,
                                     &route_entry_queued);
// ... no early return for route_entry_queued ...
zebra_rnh_eval_nexthop_entry(zvrf, afi, force, nrn, rnh, prn, re,
                             route_entry_queued);

// In zebra_rnh_eval_nexthop_entry(), gate only the dplane event:
if (state_changed && !route_entry_queued) {
    dplane_nht_event_update(...);
}
// ... client notifications and pseudowire processing proceed regardless ...
```

**Recovery**: When the queued route finishes processing and gets offloaded (Step 3 in the code path below), FRR's normal NHT re-evaluation cycle triggers automatically. At that point, `zebra_rnh_resolve_nexthop_entry()` succeeds (the route is no longer queued) and the correct NHT event is sent to FPM. No explicit re-trigger mechanism is needed.

The queued state is handled in the following place in zebra_rnh_resolve_nexthop_entry().

```
			if (CHECK_FLAG(re->status, ROUTE_ENTRY_QUEUED)) {
				if (IS_ZEBRA_DEBUG_NHT_DETAILED)
					zlog_debug(
						"        Route Entry %s queued",
						zebra_route_string(re->type));
				continue;
			}
```

While FRR invokes zebra_rnh_resolve_nexthop_entry() from three different call sites, only the final invocation is relevant here. This is because SONiC always runs with --asic-offload=notify_on_offload enabled, making that specific code path the one we need to target.

```
Detailed Code Path (asic_offloaded = true)

   Step 1: BGP route arrives → process_subq_route() → rib_process()
     1 process_subq_route()                              [line 2684]
     2   └── rib_process(rn)                             [line 2696 → 1254]
     3         │
     4         ├── RNODE_FOREACH_RE_SAFE: find best route
     5         │     ├── old_selected = current SELECTED re
     6         │     ├── nexthop_active_update(rn, re)   [line 1340]
     7         │     └── rib_choose_best() → new_selected = BGP re  [line 1404]
     8         │
     9         ├── SET_FLAG(new_selected, ZEBRA_FLAG_SELECTED)       [line 1459]
    10         │
    11         ├── rib_process_add_fib(zvrf, rn, new_fib)            [line 1498]
    12         │   or rib_process_update_fib(zvrf, rn, old_fib, new_fib) [line 1496]
    13         │     │
    14         │     └── rib_install_kernel(rn, new, old)            [line 1014/1102]
    15         │           │
    16         │           ├── zebra_nhg_install_kernel(re->nhe)     [line 659]
    17         │           ├── dest->selected_fib = re               [line 671]
    18         │           ├── hook_call(rib_update, rn, "installing in kernel")  [line 677]
    19         │           │     └── (FPM gets notified here via hook)
    20         │           ├── dplane_route_add(rn, re)              [line 683]
    21         │           │     └── returns ZEBRA_DPLANE_REQUEST_QUEUED
    22         │           └── SET_FLAG(re->status, ROUTE_ENTRY_QUEUED)  ← ★ QUEUED SET [line 687]
    23         │
    24         ├── UNSET_FLAG(new->status, ROUTE_ENTRY_CHANGED)      [line 1016/1163]
    25         │
    26         └── rib_gc_dest(rn)                                   [line 1516]
    27               └── rib_can_delete_dest() → false (route exists)
    28               └── returns 0 — NO NHT evaluation
   At this point: Route is SELECTED, QUEUED, but NOT INSTALLED. No NHT evaluation runs, in 

   ---

   Step 2: Kernel ACK → rib_process_result() 
     1 rib_process_dplane_results()                      [line 5111]
     2   └── dplane_ctx_get_op() == DPLANE_OP_ROUTE_INSTALL
     3        └── dplane_ctx_get_notif_provider() == 0
     4             └── rib_process_result(ctx)           [line 5185 → 2000]
     5                   │
     6                   ├── Find matching re via RNODE_FOREACH_RE    [line 2052]
     7                   ├── seq = dplane_ctx_get_seq(ctx)            [line 2070]
     8                   │
     9                   ├── re->dplane_sequence == seq? YES          [line 2076]
    10                   │
    11                   ├── ★ QUEUED flag check:                    [line 2092-2102]
    12                   │   if (zrouter.asic_offloaded &&            // TRUE
    13                   │       status == ZEBRA_DPLANE_REQUEST_FAILURE)
    14                   │       UNSET QUEUED;                        // only on FAILURE
    15                   │
    16                   │   if (!zrouter.asic_offloaded ||           // FALSE
    17                   │       (CHECK_FLAG(re->flags, ZEBRA_FLAG_OFFLOADED) ||  // NOT SET
    18                   │        CHECK_FLAG(re->flags, ZEBRA_FLAG_OFFLOAD_FAILED))) // NOT SET
    19                   │   {
    20                   │       UNSET QUEUED;                        // ★ SKIPPED! condition is false
    21                   │   }
    22                   │
    23                   │   → ROUTE_ENTRY_QUEUED REMAINS SET
    24                   │
    25                   ├── status == SUCCESS:                       [line 2119]
    26                   │     SET_FLAG(re->status, ROUTE_ENTRY_INSTALLED) [line 2122]
    27                   │     rib_update_re_from_ctx()               [line 2143]
    28                   │     redistribute_update()                  [line 2169]
    29                   │
    30                   └── zebra_rib_evaluate_rn_nexthops(rn, seq, false)  ← ★ NHT EVAL [line 2260]
    31                         │
    32                         └── walks up tree, for each rnh:
    33                               └── zebra_evaluate_rnh(zvrf, afi, 0, p, safi)  [line 917]
    34                                     └── zebra_rnh_evaluate_entry()            [line ~488]
    35                                           └── zebra_rnh_resolve_nexthop_entry() [line ~556]
    36                                                 │
    37                                                 ├── rn = route_node_match(table, &nrn->p)
    38                                                 ├── RNODE_FOREACH_RE(rn, re):
    39                                                 │     ├── CHECK re REMOVED? no
    40                                                 │     ├── CHECK re SELECTED or FIB_OVERRIDE? YES ✓
    41                                                 │     ├── ★ CHECK_FLAG(re->status, ROUTE_ENTRY_QUEUED)?
    42                                                 │     │     → YES! Still set!                [line 611]
    43                                                 │     │     → zlog_debug("Route Entry %s queued",
    44                                                 │     │                   zebra_route_string(re->type))
    45                                                 │     │     → "Route Entry bgp queued"  ← ★★★ YOUR LOG
    46                                                 │     │     → continue; (SKIP this re)
    47                                                 │     └── no valid re found
    48                                                 └── returns NULL (re unresolved)
    49                                           └── zebra_rnh_eval_nexthop_entry(... re=NULL ...)
    50                                                 → if rnh->state was already NULL, no state change
    51                                                 → NHT clients NOT notified (or notified as unreachable)
   At this point: Route is SELECTED + INSTALLED + QUEUED. NHT skips it. Clients don't see the nexthop resolved.

   ---

   Step 3: FPM/ASIC notification → rib_process_dplane_notify() 
     1 rib_process_dplane_results()                      [line 5111]
     2   └── dplane_ctx_get_op() == DPLANE_OP_ROUTE_NOTIFY
     3        └── rib_process_dplane_notify(ctx)         [line 5189 → 2308]
     4              │
     5              ├── rn = rib_find_rn_from_ctx(ctx)              [line 2321]
     6              ├── Find matching re via RNODE_FOREACH_RE       [line 2343]
     7              │
     8              ├── ★ UNSET_FLAG(re->status, ROUTE_ENTRY_QUEUED)  ← QUEUED CLEARED [line 2361]
     9              ├── UNSET_FLAG(re->status, ROUTE_ENTRY_ROUTE_REPLACING)             [line 2362]
    10              │
    11              ├── re == dest->selected_fib? YES               [line 2368]
    12              │     ├── SET_FLAG(re->flags, ZEBRA_FLAG_OFFLOADED)                 [line 2410]
    13              │     └── (or ZEBRA_FLAG_OFFLOAD_FAILED)
    14              │
    15              ├── start_count = rib_count_installed_nh(re)    [line 2427]
    16              ├── fib_changed = rib_update_re_from_ctx(re, rn, ctx) [line 2432]
    17              ├── end_count = rib_count_installed_nh(re)      [line 2446]
    18              │
    19              ├── (if start_count==0, end_count>0):           [line 2464]
    20              │     SET_FLAG(re->status, ROUTE_ENTRY_INSTALLED)
    21              │     dplane_route_notif_update(rn, re, ...)
    22              │     redistribute_update(rn, re, NULL)
    23              │
    24              └── ★ zebra_rib_evaluate_rn_nexthops(rn, seq, false) ← NHT EVAL [line 2518]
    25                    │
    26                    └── walks up tree, for each rnh:
    27                          └── zebra_evaluate_rnh(zvrf, afi, 0, p, safi)
    28                                └── zebra_rnh_evaluate_entry()
    29                                      └── zebra_rnh_resolve_nexthop_entry()
    30                                            │
    31                                            ├── RNODE_FOREACH_RE(rn, re):
    32                                            │     ├── REMOVED? no
    33                                            │     ├── SELECTED? YES ✓
    34                                            │     ├── ★ QUEUED? NO! (cleared at line 2361) ✓
    35                                            │     ├── rnh_check_re_nexthops(re, rnh)? YES ✓
    36                                            │     └── MATCH! re found
    37                                            └── returns re, *prn = rn
    38                                      └── zebra_rnh_eval_nexthop_entry(... re=valid ...)
    39                                            ├── state_changed = 1
    40                                            ├── copy_state(rnh, re, nrn)
    41                                            ├── zebra_rnh_notify_protocol_clients() ← clients notified ✓
    42                                            └── zebra_rnh_process_pseudowires()
 ```

## Dplane Integration: New Event Type & Context Structure
### New Dplane Operation Enum
A new action enum DPLANE_OP_NHT_EVENT_UPDATE would be added in dplane_op_e https://github.com/FRRouting/frr/blob/master/zebra/zebra_dplane.h#L116

```
enum dplane_op_e {
    // ... existing ops ...
    DPLANE_OP_NHT_EVENT_UPDATE,  // New: RNH state change notification
};
```

### Extended Dplane Context
Extend struct zebra_dplane_ctx (https://github.com/FRRouting/frr/blob/master/zebra/zebra_dplane.c#L446) with a dedicated union member:

```
struct zebra_dplane_ctx {
    // ... existing fields ...
    union {
        // ... existing union members ...
        struct dplane_rnh_info rnh_info;  // New: RNH event payload
    };
};
```
### RNH Info Structure Definition
Define the payload structure to carry RNH change context:

```
struct dplane_rnh_info {
  struct prefix p; // Tracked RNH prefix

  struct prefix previous_resolved_prefix;
  uint32_t previous_resolved_nhg_id;

  struct prefix current_resolved_prefix;
  uint32_t current_resolved_nhg_id;

}
```

### Event Generation Workflow

#### `copy_state()` Fix: Preserve NHE ID

The existing `copy_state()` call uses `zebra_nhe_copy(re->nhe, 0)`, which zeroes the NHE ID in the cached state. This breaks `prev_nhg_id` caching — when `zebra_rnh_eval_nexthop_entry()` reads `rnh->state->nhe->id` before `copy_state()`, it gets 0 instead of the actual previous NHG ID.

**Fix**: Change to `zebra_nhe_copy(re->nhe, re->nhe->id)` to preserve the NHE ID in the copy.

#### NHT Event Generation in `zebra_rnh_eval_nexthop_entry()`

Before any state mutation (before `copy_state()` is called), cache the previous state as primitive values:
```c
uint32_t prev_nhg_id = (rnh->state && rnh->state->nhe)
                        ? rnh->state->nhe->id : 0;
struct prefix prev_resolved;
prefix_copy(&prev_resolved, &rnh->resolved_route);
```

After state change detection (`state_changed || force`), generate the NHT dplane event — gated by both `state_changed` and `!route_entry_queued`:
```c
if (state_changed && !route_entry_queued) {
    struct prefix curr_resolved;
    uint32_t curr_nhg_id;
    enum zebra_dplane_result dplane_res;

    prefix_copy(&curr_resolved, &rnh->resolved_route);
    curr_nhg_id = (rnh->state && rnh->state->nhe)
                  ? rnh->state->nhe->id : 0;

    if (IS_ZEBRA_DEBUG_NHT)
        zlog_debug("NHT event: rnh=%pFX prev_nhg=%u curr_nhg=%u",
                   &nrn->p, prev_nhg_id, curr_nhg_id);

    dplane_res = dplane_nht_event_update(
        &nrn->p, &prev_resolved, prev_nhg_id,
        &curr_resolved, curr_nhg_id);
    if (dplane_res != ZEBRA_DPLANE_REQUEST_QUEUED)
        zlog_warn("NHT event enqueue failed for rnh=%pFX: result=%d",
                  &nrn->p, dplane_res);
}
```

Client notifications and pseudowire processing follow unconditionally (same as before).

#### `dplane_nht_event_update()` Function

| Aspect | Specification |
|:---|:---|
| Declared in | `zebra/zebra_dplane.h` |
| Implemented in | `zebra/zebra_dplane.c` |
| Signature | `enum zebra_dplane_result dplane_nht_event_update(const struct prefix *rnh_prefix, const struct prefix *prev_resolved_prefix, uint32_t prev_nhg_id, const struct prefix *curr_resolved_prefix, uint32_t curr_nhg_id)` |
| Return | `ZEBRA_DPLANE_REQUEST_QUEUED` on success, `ZEBRA_DPLANE_REQUEST_FAILURE` on error |
| Kernel skip | Always sets `dplane_ctx_set_skip_kernel(ctx)` — NHT events only target FPM consumers |
| NULL check | Returns failure immediately if `rnh_prefix` is NULL |
| Debug logging | Logs rnh prefix and NHG IDs at `IS_ZEBRA_DEBUG_DPLANE_DETAIL` level |
| Cleanup | Frees ctx on enqueue failure |

Note: The function takes `const struct prefix *` parameters (not `struct rnh *`) for clean dplane separation — the dplane layer should not know about RNH internals.

#### Accessor Functions

Declared in `zebra/zebra_dplane.h`, implemented in `zebra/zebra_dplane.c`:

| Function | Return Type |
|:---|:---|
| `dplane_ctx_get_nht_rnh_prefix(ctx)` | `const struct prefix *` |
| `dplane_ctx_get_nht_prev_resolved_prefix(ctx)` | `const struct prefix *` |
| `dplane_ctx_get_nht_prev_resolved_nhg_id(ctx)` | `uint32_t` |
| `dplane_ctx_get_nht_curr_resolved_prefix(ctx)` | `const struct prefix *` |
| `dplane_ctx_get_nht_curr_resolved_nhg_id(ctx)` | `uint32_t` |

Each accessor calls `DPLANE_CTX_VALID(ctx)` and reads from `ctx->u.rnh_info`.

#### Boilerplate Switch Cases

`DPLANE_OP_NHT_EVENT_UPDATE` must be added to every exhaustive switch on `dplane_op_e` across these files:

| File | Switch Location | Action |
|:---|:---|:---|
| `zebra/zebra_dplane.c` | `dplane_op2str()` | Return `"NHT_EVENT_UPDATE"` |
| `zebra/zebra_dplane.c` | `dplane_ctx_free_internal()` | `break` (no resources to free) |
| `zebra/zebra_dplane.c` | `kernel_dplane_log_detail()` | Log rnh prefix via `dplane_ctx_get_nht_rnh_prefix()` |
| `zebra/zebra_dplane.c` | `kernel_dplane_handle_result()` | Increment `dg_other_errors` on failure |
| `zebra/zebra_rib.c` | `rib_process_dplane_results()` | `break` (no-op, handled by FPM provider) |
| `zebra/dplane_fpm_nl.c` | `fpm_nl_enqueue()` | `break` (handled by sonic-buildimage's FPM provider, not upstream) |
| `zebra/kernel_netlink.c` | `nl_put_msg()` | Return `FRR_NETLINK_ERROR` (not a kernel op) |
| `zebra/kernel_socket.c` | `kernel_update_multi()` | Log error + set failure (not a kernel op) |
| `zebra/zebra_script.c` | Lua dplane ctx push | `break` (no Lua binding) |

#### `zebra_rnh_clear_nhc_flag()` Update

This function also calls `zebra_rnh_resolve_nexthop_entry()`. Pass `NULL` for the `route_entry_queued` parameter since it doesn't need suppression information:
```c
re = zebra_rnh_resolve_nexthop_entry(zvrf, afi, nrn, rnh, &prn, NULL);
```

#### Include Dependency

Add `#include "zebra/zebra_dplane.h"` to `zebra/zebra_rnh.c` for the `dplane_nht_event_update()` declaration and `enum zebra_dplane_result` type.

# 4. FPM Message Serialization (SONiC Integration)

**File**: `src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c`

## Include Dependency

Add at the top with existing sonic-fib includes:
```c
#include <nexthopgroup/c-api/nhtevent_capi.h>
#include <nexthopgroup/c_nhtevent.h>
```

Both headers are needed: `c_nhtevent.h` provides the `struct C_NhtEvent` definition, `nhtevent_capi.h` declares `nhtevent_json_from_c_nht()`.

## Custom Message Type

Add to the `enum custom_nlmsg_types` block:
```c
RTM_NEWNHTEVENT = 6000,
```

## Encoder Function: `netlink_nhtevent_msg_encode()`

A standalone encoder function following the same pattern as `netlink_nhg_fib_msg_encode()`.

| Aspect | Specification |
|:---|:---|
| Signature | `static ssize_t netlink_nhtevent_msg_encode(const struct zebra_dplane_ctx *ctx, void *buf, size_t buflen)` |
| Return value | `-1` on failure, `0` when msg doesn't fit in buffer, otherwise number of bytes written |
| Netlink header | Uses `struct rtmsg` (not `nhmsg`); type = `RTM_NEWNHTEVENT`; flags = `NLM_F_CREATE \| NLM_F_REQUEST` |
| Family | `req->rtm.rtm_family = rnh_pfx ? rnh_pfx->family : AF_UNSPEC` |
| Buffer check | Returns `0` if `buflen < sizeof(*req)` |
| JSON attribute | Encoded as `FPM_NHA_JSON_STR` via `nl_attr_put()` with return value check |

### Field Extraction and NULL Handling

| Field | Accessor | NULL/Zero-family Fallback |
|:---|:---|:---|
| `rnh_prefix` | `dplane_ctx_get_rnh_prefix(ctx)` | `::/0` |
| `prev_resolved_prefix` | `dplane_ctx_get_rnh_prev_resolved_prefix(ctx)` | `::/0` |
| `prev_resolved_nhg_id` | `dplane_ctx_get_rnh_prev_resolved_nhg_id(ctx)` | (uint32_t, always valid) |
| `curr_resolved_prefix` | `dplane_ctx_get_rnh_curr_resolved_prefix(ctx)` | `::/0` |
| `curr_resolved_nhg_id` | `dplane_ctx_get_rnh_curr_resolved_nhg_id(ctx)` | (uint32_t, always valid) |

For each prefix field: if the pointer is non-NULL and `family != 0`, convert via `prefix2str()` into a `char[64]` buffer. Otherwise, use the fallback string `::/0`.

### JSON Serialization

Populate a `struct C_NhtEvent` from the extracted fields, then pass its pointer to the C API:
```c
struct C_NhtEvent c_nht = {};
/* rnh_prefix, prev_resolved_prefix, curr_resolved_prefix are char[64] locals
   populated by prefix2str() or fallback above */
strlcpy(c_nht.rnh_prefix, rnh_prefix, sizeof(c_nht.rnh_prefix));
strlcpy(c_nht.prev_resolved_prefix, prev_resolved_prefix, sizeof(c_nht.prev_resolved_prefix));
c_nht.prev_resolved_nhg_id = prev_resolved_nhg_id;
strlcpy(c_nht.curr_resolved_prefix, curr_resolved_prefix, sizeof(c_nht.curr_resolved_prefix));
c_nht.curr_resolved_nhg_id = curr_resolved_nhg_id;

json_str = nhtevent_json_from_c_nht(&c_nht);
```

This matches the sonic-fib C API signature: `char* nhtevent_json_from_c_nht(const struct C_NhtEvent* c_nht)`.

### Error Handling

| Condition | Action |
|:---|:---|
| `nhtevent_json_from_c_nht()` returns NULL | `zlog_err("%s: nhtevent_json_from_c_nht failed for rnh=%s", __func__, rnh_prefix)` → return -1 |
| `nl_attr_put()` returns false (buffer overflow) | `zlog_err` → `free(json_str)` → return -1 |

### Debug Logging

On successful encode: `zlog_debug("%s: encoded NHT event rnh=%s ret=%zd", __func__, rnh_prefix, ret)`

### Memory

`free(json_str)` after netlink message is fully constructed (before return).

## Switch Case in `fpm_nl_enqueue()`

```c
case DPLANE_OP_NHT_EVENT_UPDATE:
    rv = netlink_nhtevent_msg_encode(ctx, nl_buf, sizeof(nl_buf));
    if (rv <= 0) {
        zlog_err("%s: netlink_nhtevent_msg_encode failed", __func__);
        dplane_ctx_set_status(ctx, ZEBRA_DPLANE_REQUEST_FAILURE);
        return 0;
    }
    nl_buf_len = (size_t)rv;
    break;
```

This follows the same error-handling pattern as other encoder call sites: on failure, set status and return 0 (message not enqueued).

# 5. SONiC-fib Enhancements for NHT events
A new JSON schema `NhtEvent.json` is added to the [sonic-fib](https://github.com/eddieruan-alibaba/sonic-fib) repository, following the same template-driven generation pattern as `NextHopGroupFull.json`.

## NhtEvent JSON Schema

The schema captures the full `dplane_rnh_info` payload so that future phases need no wire-format changes.

| Field | JSON Type | Description | Unresolved Value |
|:---|:---|:---|:---|
| `rnh_prefix` | string (addr/len) | Tracked RNH prefix, e.g. `2064:100::1d/128` | — |
| `prev_resolved_prefix` | string (addr/len) | Previous resolving route prefix | `0.0.0.0/0` or `::/0` |
| `prev_resolved_nhg_id` | integer | Previous resolving NHG identifier | 0 |
| `curr_resolved_prefix` | string (addr/len) | Current resolving route prefix | `0.0.0.0/0` or `::/0` |
| `curr_resolved_nhg_id` | integer | Current resolving NHG identifier | 0 |

Prefix strings use `addr/len` notation. All 5 fields are required. The schema enforces `"required"` and `"additionalProperties": false` to catch malformed JSON at parse time.

**Schema file** (`schema/NhtEvent.json`):
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "NhtEvent.json",
  "title": "NhtEvent",
  "description": "NHT (Nexthop Tracking) event from FRR zebra to fpmsyncd",
  "type": "object",
  "properties": {
    "rnh_prefix": {
      "type": "string",
      "position": 1,
      "default_value": "\"\"",
      "description": "Tracked RNH prefix in addr/len format (e.g. 2064:100::1d/128)"
    },
    "prev_resolved_prefix": {
      "type": "string",
      "position": 1,
      "default_value": "\"\"",
      "description": "Previous resolving route prefix in addr/len format. 0.0.0.0/0 or ::/0 when unresolved"
    },
    "prev_resolved_nhg_id": {
      "type": "integer",
      "position": 1,
      "minimum": 0,
      "default_value": "0",
      "description": "Previous resolving NHG identifier. 0 when unresolved"
    },
    "curr_resolved_prefix": {
      "type": "string",
      "position": 1,
      "default_value": "\"\"",
      "description": "Current resolving route prefix in addr/len format. 0.0.0.0/0 or ::/0 when unresolved"
    },
    "curr_resolved_nhg_id": {
      "type": "integer",
      "position": 1,
      "minimum": 0,
      "default_value": "0",
      "description": "Current resolving NHG identifier. 0 when unresolved (Phase 1 trigger)"
    }
  },
  "required": [
    "rnh_prefix",
    "prev_resolved_prefix",
    "prev_resolved_nhg_id",
    "curr_resolved_prefix",
    "curr_resolved_nhg_id"
  ],
  "additionalProperties": false
}
```

Note: The `"default_value"` field uses C++ literal notation (`"\"\""` for empty string, `"0"` for integer) — these are passed directly into generated code as initializers.

### Example: nexthop becomes unresolved (Phase 1 trigger)

```json
{
  "rnh_prefix": "2064:200::1e/128",
  "prev_resolved_prefix": "2064:200::1e/128",
  "prev_resolved_nhg_id": 258,
  "curr_resolved_prefix": "::/0",
  "curr_resolved_nhg_id": 0
}
```

Phase 1 uses `rnh_prefix` (converted to nexthop address) and `prev_resolved_nhg_id` when `curr_resolved_nhg_id == 0`.

## Generated Code

The Jinja2 template pipeline generates four files from `schema/NhtEvent.json`:

| Generated File | Purpose |
|:---|:---|
| `src/nhtevent.h` / `src/nhtevent.cpp` | C++ `fib::NhtEvent` POD struct with default initializers (no custom constructors, copy ops, or equality operators needed for simple scalar/string fields) |
| `src/nhtevent_json.h` | ADL-based `to_json()`/`from_json()` for nlohmann/json integration, plus `nhtevent_to_json_string()` and `nhtevent_from_json_string()` convenience helpers |
| `src/c_nhtevent.h` | C struct `C_NhtEvent` with `NHT_PREFIX_MAXLEN` (64) fixed-size char arrays for FRR, wrapped in `extern "C"` guards |

### Design Decisions

* **POD struct over class**: `NhtEvent` has only 5 trivial fields (3 strings, 2 integers). Default copy/move/equality from the compiler are sufficient. This reduces template complexity and generated code size.
* **ADL `to_json`/`from_json`**: The idiomatic nlohmann/json pattern. Enables `nlohmann::ordered_json j = evt;` and `j.get<NhtEvent>()` directly.
* **`nhtevent_to_json_string()`**: Named helper (not generic `to_json_string()`) to avoid collisions with other types in the `fib` namespace.
* **C API takes `const struct C_NhtEvent*`**: Single-pointer parameter is more extensible than individual arguments — adding fields requires no signature change.

## C API for FRR Integration

**Files**: `src/c-api/nhtevent_capi.h` and `src/c-api/nhtevent_capi.cpp`

**Function**: `char* nhtevent_json_from_c_nht(const struct C_NhtEvent* c_nht)`

| Aspect | Specification |
|:---|:---|
| Linkage | `extern "C"` — callable from C (FRR) |
| Input | `const struct C_NhtEvent*` (generated C struct from `c_nhtevent.h.j2`) |
| Output | `malloc`'d JSON string; caller must `free()`. Returns `NULL` on failure. |
| Serialization flow | 1. Field-by-field copy from `C_NhtEvent` → `fib::NhtEvent` (POD struct) <br> 2. Call `fib::nhtevent_to_json_string(evt)` to produce JSON <br> 3. `malloc` + `memcpy` the result string for C ownership |
| Error handling | NULL pointer check → log `FIB_LOG(ERROR)` + return NULL <br> Serialization exception (catch-all) → log `FIB_LOG(ERROR)` + return NULL <br> malloc failure → log `FIB_LOG(ERROR)` + return NULL |
| Debug logging | On success, log JSON string length and content at `DEBUG` level via `FIB_LOG` |
| Header guard | Standard `#ifndef NHTEVENT_CAPI_H` with `extern "C"` block; forward-declares `struct C_NhtEvent` |

FRR's `fpm_nl_enqueue()` calls `nhtevent_json_from_c_nht()` in the `DPLANE_OP_NHT_EVENT_UPDATE` case to serialize the event before sending over the FPM socket. fpmsyncd deserializes via `nhtevent_from_json_string()`.

## Generated Code Templates

Four Jinja2 templates in `templates/` are rendered by `scripts/render_schema.py` to produce generated code from the JSON schema. Each template follows the existing `nexthopgroupfull` conventions.

### Template: `nhtevent.h.j2` (mode: `header`)

Generates the C++ POD struct header.

| Aspect | Description |
|:---|:---|
| Output | `#pragma once` header in `namespace fib` |
| Structure | A single `struct {root_struct_name}` with one member per schema property |
| Field iteration | Iterates `structs[root_struct_name].fields` (consistent with `nexthopgroupfull.h.j2`) |
| Default values | Each field uses `= {field.default_value}` inline initializer from schema's `default_value` (C++ literal notation) |
| Includes | `<string>`, `<cstdint>` |

### Template: `nhtevent.cpp.j2` (mode: `source`)

Generates an empty implementation file (POD structs need no custom methods).

| Aspect | Description |
|:---|:---|
| Output | Includes the header; contains only a namespace placeholder comment |
| Rationale | POD struct — no constructors, destructors, or operators needed |
| Context | Only `root_struct_name` needed |

### Template: `nhtevent_json.h.j2` (mode: `json_bindings`)

Generates nlohmann/json ADL bindings and string helpers.

| Aspect | Description |
|:---|:---|
| Pattern | nlohmann ADL: free functions `to_json(ordered_json&, const T&)` and `from_json(const ordered_json&, T&)` |
| `to_json` | Constructs `ordered_json` initializer list from all `root_struct.fields` (preserves field order) |
| `from_json` | Calls `j.at("{field.name}").get_to(obj.{field.name})` for each field (strict — throws on missing keys) |
| String helpers | `nhtevent_to_json_string(const T&)` → `ordered_json(obj).dump()` <br> `nhtevent_from_json_string(const string&)` → `parse` + `get<T>()` |
| All functions | `inline` in header (header-only library pattern) |
| Includes | The struct header + `<nlohmann/json.hpp>` + `<string>` |

### Template: `c_nhtevent.h.j2` (mode: `c_header`)

Generates a C-compatible struct for FRR integration.

| Aspect | Description |
|:---|:---|
| Output | `#ifndef` guarded C header with `extern "C"` block |
| Constant | `#define NHT_PREFIX_MAXLEN 64` for fixed-size char arrays |
| Field mapping | `char*` schema fields → `char name[NHT_PREFIX_MAXLEN]`; numeric fields → direct C type |
| Field iteration | Uses `root_struct.fields` directly |
| Includes | `<stdint.h>`, `<stdbool.h>` |

### `render_schema.py` Template Selection

| Aspect | Description |
|:---|:---|
| Template naming | Derived from schema `title.lower()` — e.g., `"NhtEvent"` → base `"nhtevent"` |
| Mode → filename | `header` → `{base}.h.j2`, `source` → `{base}.cpp.j2`, `json_bindings` → `{base}_json.h.j2`, `c_header` → `c_{base}.h.j2` |
| No explicit map | Convention-based discovery; adding a new schema only requires matching template filenames |

Template context by mode:

| Mode | Context Variables |
|:---|:---|
| `header` | `enums`, `structs`, `special_structs`, `root_struct_name` |
| `source` | `root_struct_name` |
| `json_bindings` | `enums`, `root_struct_name`, `root_struct`, `special_structs`, `all_structs` |
| `c_header` | `c_enums`, `structs`, `special_structs`, `root_struct`, `root_struct_name` |

Note: The `header` template uses `structs[root_struct_name].fields` (consistent with `nexthopgroupfull.h.j2`), while `json_bindings` and `c_header` use `root_struct.fields` directly.

## Data Flow

```
FRR (C)                        sonic-fib                      fpmsyncd (C++)
-----------                    ---------                      --------------
dplane_rnh_info
  |
  v
fpm_nl_enqueue()
  DPLANE_OP_NHT_EVENT_UPDATE
  |
  +-> nhtevent_json_from_c_nht() -> JSON string -> FPM socket
                                                      |
                                                      v
                                     nhtevent_from_json_string() -> fib::NhtEvent
                                                                       |
                                                                       v
                                                   fib_nhg_trigger_node_quick_fixup(
                                                       nexthop_addr,
                                                       prev_resolved_nhg_id)
```

# 6. FPMsyncd Modifications
The input data for this NHT event is detailed in the preceding sections. Under the Phase 1 approach, FPMsyncd will invoke fib_nhg_trigger_node_quick_fixup() when current_resolved_nhg_id is zero, indicating that the tracked nexthop address cannot be resolved. Additional scenarios will be addressed in future updates.

## Existing NHG MGR codes
Here are current  nhg mgr 's  codes

* https://github.com/eddieruan-alibaba/sonic-swss/blob/rib_fib/fpmsyncd/nhgmgr.cpp
* https://github.com/eddieruan-alibaba/sonic-swss/blob/rib_fib/fpmsyncd/nhgmgr.h

## NHT Event Message Reception (`onNhtEventMsg`)

### Message Dispatch in `onMsgRaw()`
The NHT event message uses netlink type `RTM_NEWNHTEVENT` (6000). In `onMsgRaw()`, the message length is calculated using `struct rtmsg` as the header:
```cpp
else if(h->nlmsg_type == RTM_NEWNHTEVENT)
{
    len = (int)(h->nlmsg_len - NLMSG_LENGTH(sizeof(struct rtmsg)));
}
```

### `RouteSync::onNhtEventMsg()`
Parses the netlink message and invokes the backwalk trigger.

**Message format**: The encoder (in sonic-buildimage) uses `struct rtmsg` as the netlink message header with a single `NHA_JSON_STR` attribute containing the JSON-serialized NHT event.

**Implementation**:
```cpp
void RouteSync::onNhtEventMsg(struct nlmsghdr *h, int len)
{
    struct rtmsg *rtm = (struct rtmsg *)NLMSG_DATA(h);
    struct rtattr *tb[RTA_MAX + 1] = {};

    netlink_parse_rtattr(tb, RTA_MAX, RTM_RTA(rtm), len);

    if (!tb[NHA_JSON_STR]) {
        SWSS_LOG_ERROR("NHT event message without JSON string");
        return;
    }

    char *json_str = (char *)RTA_DATA(tb[NHA_JSON_STR]);

    try {
        fib::NhtEvent nht_event = fib::nhtevent_from_json_string(string(json_str));

        // Phase 1: Only trigger when curr_resolved_nhg_id == 0 (nexthop unreachable)
        if (nht_event.curr_resolved_nhg_id != 0) {
            return;  // not Phase 1, skip
        }

        // Convert rnh_prefix to nexthop address (strip /prefix_len)
        string nexthop_addr = nht_event.rnh_prefix;
        size_t slash_pos = nexthop_addr.find('/');
        if (slash_pos != string::npos) {
            nexthop_addr = nexthop_addr.substr(0, slash_pos);
        }

        m_rib_fib_nhg_mgr.fib_nhg_trigger_node_quick_fixup(
            nexthop_addr, nht_event.prev_resolved_nhg_id);

    } catch (const std::exception &e) {
        SWSS_LOG_ERROR("Failed to parse NHT event JSON: %s", e.what());
    }
}
```

**Key design points**:
1. Uses `struct rtmsg` + `RTM_RTA(rtm)` to extract attributes (matching the encoder's format).
2. JSON deserialization is wrapped in `try/catch` to handle malformed messages gracefully.
3. Early-return with debug log when `curr_resolved_nhg_id != 0` — Phase 1 only handles nexthop-unreachable events.
4. The `rnh_prefix` (e.g., `"fc06::2/128"`) is stripped of its prefix length to produce the nexthop address for the backwalk.

## `fib_nhg_trigger_node_quick_fixup()`
This function serves as the primary entry point invoked when `fpmsyncd` receives a Next Hop Tracking (NHT) event from Zebra. It accepts the following parameters:
* **Nexthop Address**: The specific IPv4 or IPv6 address associated with the event.
  * *Note*: Input prefixes are converted to nexthop addresses because, for the cases we are interested, the tracked RNH represents a specific next-hop address rather than a network prefix.
* **Resolved NHG ID**: The previously resolved Next Hop Group ID for the given nexthop address. This value serves as the starting point for the backwalk traversal.

The function performs two primary operations:

### Part 1: Global Table Context Backwalk
Locates the starting RIBNHGEntries for the global table context and triggers a backwalk that updates all relevant NHGs.

**Entry point resolution (two-step lookup)**:
1. **Try `resolved_nhg_id` first**: Call `getEntry(resolved_nhg_id)`. If found, use it as the sole starting entry.
2. **Fallback to `m_nexthop_to_global_RIBNHG`**: If `getEntry(resolved_nhg_id)` returns nullptr (the `resolved_nhg_id` points to a connected-route NHG not in our table — see "One variation for part 1" below), fall back to `m_nexthop_to_global_RIBNHG.getGlobalEntries(nexthop_addr)` which returns the set of global-table RIBNHGEntries whose gateway matches the nexthop address.

```
    // Phase 1: Global Table Context
    global_entries = {}
    entry_by_id = getEntry(resolved_nhg_id)
    if entry_by_id != nullptr:
        global_entries = {entry_by_id}
    else:
        // resolved_nhg_id points to connected-route NHG (not in our table)
        // Fallback: use nexthop address to find leaf RIBNHGEntries
        global_entries = m_nexthop_to_global_RIBNHG.getGlobalEntries(nexthop_addr)
```

**For each global entry**, it initializes a `fib_nhg_walking_ctx` structure with the following configuration:
* **Walk context fields**:
  * `visited_node_set`: Starts as an empty `std::set<uint32_t>`. Nodes are added one by one as they are visited.
    * *Note: Even if a node is already present in the visited set, the algorithm may still initiate a forward walk to its dependents to propagate any pending updates.*
  * `modified_node_set`: Starts as an empty `std::set<uint32_t>`. When walk_spec applies a fixup to a node (gateway match or depends propagation), that node ID is added. Downstream nodes check whether any of their depends entries appear in this set to determine relevance.
  * `rib_nhg_table`: Pointer to the `RIBNHGTable` for entry lookups during traversal.
* **Walk Specification**: Set to `fib_nhg_walk_spec_for_node_quick_fixup` in part 1, which defines the operations to perform on each visited node.
* **Prune Specification**: Set to `fib_nhg_prune_spec_for_node_quick_fixup` in part 1, which determines whether traversal should terminate at the current node.
* **Nexthop Address**: The address identifying the impacted routing resource.

**Backwalk invocation**: The trigger function calls `fib_nhg_back_walk(start_entry->getRIBID(), ctx)` directly on each starting entry. The generic back_walk infrastructure invokes walk_spec on it — since the starting node's gateway always matches the withdrawn nexthop (for remote failures), walk_spec returns `true` and the node gets its `m_resolved_enable_group` updated. The backwalk then continues to the starting node's dependents via the standard recursion path.

### Part 2: VPN Context Backwalk
Uses the incoming nexthop address to locate all `RIBHNGEntry` instances that reference it across different VPN contexts. The function iterates through each matching entry and triggers a backwalk from that node to update the corresponding SONiC NHGs.

**Note**: 
1. Currently, backwalks are not initiated using the original NHG ID. This capability may be introduced in future updates once specific use cases are validated.
2. Since these NHG have VPN contexts, we can't use part 1's walk to reach these nodes. 

## `fib_nhg_back_walk()`
This function provides a generalized backwalk infrastructure within the `RIBNHGTable` class, enabling traversal across all managed `RIBNHGEntry` objects.

### Parameters
* `id` (`uint32_t`): The Zebra NHG ID, used as a key to retrieve the corresponding `RIBNHGEntry` via `getEntry()`.
* `ctx` (`fib_nhg_walking_ctx&`): A configuration structure containing:
  * `visited_node_set` (`std::set<uint32_t>`): Tracks visited node IDs during the walk.
  * `modified_node_set` (`std::set<uint32_t>`): Tracks nodes that were modified by walk_spec (gateway match or depends propagation). Downstream nodes check whether any of their depends entries appear in this set to determine relevance.
  * `updated_sonic_nhg_keys` (`std::set<std::string>`): Used in Part 2 to deduplicate SONiC NHG updates when multiple RIBNHGEntries map to the same SONiC NHG key.
  * `fib_nhg_walk_spec_func` (`std::function<bool(RIBNHGEntry*, fib_nhg_walking_ctx&)>`): A callback function that applies necessary updates to the current node.
  * `fib_nhg_prune_spec_func` (`std::function<bool(RIBNHGEntry*, bool, fib_nhg_walking_ctx&)>`): A callback function that determines whether traversal should terminate at the current node. Receives the walk_result and the context.
  * `nexthop_address` (`std::string`): The nexthop address identifying the impacted routing resource.
  * `rib_nhg_table` (`RIBNHGTable*`): Pointer to the RIBNHGTable for looking up entries during traversal.

```cpp
struct fib_nhg_walking_ctx {
    std::set<uint32_t> visited_node_set;
    std::set<uint32_t> modified_node_set;
    std::set<std::string> updated_sonic_nhg_keys;
    std::function<bool(RIBNHGEntry*, fib_nhg_walking_ctx&)> fib_nhg_walk_spec_func;
    std::function<bool(RIBNHGEntry*, bool, fib_nhg_walking_ctx&)> fib_nhg_prune_spec_func;
    std::string nexthop_address;
    RIBNHGTable* rib_nhg_table = nullptr;
};
```

### Design & Extensibility
While currently utilized for NHG quick fixup operations, this infrastructure is designed to be extensible and reusable for other backwalk-driven workflows.

### Execution Flow
1. **Entry Retrieval**: Fetches the target `RIBNHGEntry` using the provided `id` via `getEntry()`.
2. **Walk Specification Execution**: Invokes the walk callback (`fib_nhg_walk_spec_func`) on the retrieved entry. This may apply state updates to the object (e.g., marking failed members as inactive).
3. **Visited State Tracking**: Adds the current node ID and its modification status to `visited_node_set`.
4. **Prune Evaluation**: Invokes the prune callback (`fib_nhg_prune_spec_func`), passing the return value from the walk callback and a reference to the current `RIBNHGEntry`. The callback determines whether traversal should halt.
5. **Recursive Traversal**: If pruning is not triggered, the function retrieves the dependents list of the current `RIBNHGEntry` and recursively invokes `fib_nhg_back_walk()` for each dependent node. The ECMP vs single-path continuation logic is handled entirely within `prune_spec` -- `fib_nhg_back_walk` itself simply iterates all dependents when not pruned.

####  Recursion Flowchart
```mermaid
flowchart TD
    START(["fib_nhg_back_walk(id, ctx)"]) --> GET_ENTRY["getEntry(id)"]
    GET_ENTRY --> NULL_CHECK{"entry is null?"}
    NULL_CHECK -->|Yes| LOG_WARN["Log warning"] --> RETURN_END(["return"])
    NULL_CHECK -->|No| ADD_VISITED["Add id to visited_node_set<br/>for debugging"]

    ADD_VISITED --> CALL_WALK["walk_result = walk_spec(entry, ctx)"]

    CALL_WALK --> CALL_PRUNE["prune = prune_spec(entry, walk_result)"]

    CALL_PRUNE --> PRUNE_CHECK{"prune == true?"}
    PRUNE_CHECK -->|Yes| RETURN_PRUNE(["return<br/>branch pruned"])
    PRUNE_CHECK -->|No| GET_DEPENDENTS

    GET_DEPENDENTS["Get entry.dependents list"] --> LOOP{"For each<br/>dependent_id"}

    LOOP --> RECURSE["fib_nhg_back_walk<br/>dependent_id, ctx"]
    RECURSE --> LOOP

    LOOP -->|All done| RETURN_DONE(["return"])
```

**Note**: `visited_node_set` is maintained for debugging and audit purposes only. Nodes are NOT skipped on revisit -- a node may need re-evaluation when reached from multiple paths (e.g., diamond topologies where a parent depends on two siblings, and the second sibling is modified after the parent's first visit).

## Changes for Part 1
### Changes in  `class RIBNHGEntry`
* New Field: `m_resolved_enable_group`
Need to add a new field m_resolved_enable_group to track if resolved NHG is enabled or not.

```
        /*
         * Resolved group of the entry.
         * Contains <ribID, bool> pairs to indicate if the resolved NHG is enabled.
         * - All paths are set as enabled upon Add/Update events from FRR.
         * - Paths are marked as disabled during the backwalk process.
         * - For leaf NHGs (depends.empty()), a self-reference {self_id, true} is added
         *   after depends-based population so walk_spec can mark the leaf disabled.
         */
        unordered_map<uint32_t, bool> m_resolved_enable_group;
```

* New Field: `m_gateway`
Need to add a new string field `m_gateway` to store NHG's gateway address in single path or recursive case. In single path case, `m_gateway` is the same as `m_nexthop`. But in recursive case, `m_gateway` is the recursive nexthop address, while `m_nexthop` stores this NHG's final resolved nexthop addresses.
```

        /*
         * Gateway  str in FV vector of the entry
         */
        string m_gateway = "";
```
Need to create an accessing api for this m_gateway. It would be used during convergence backwalk. 
```
string RIBNHGEntry::getGatewayAddress() {
    return m_gateway;
}
```

This field is populated from nexthopgroupfull's gate field, when its type is specified, a.k.a recursive or single path case. 
```
    if (nhg.type == fib::NEXTHOP_TYPE_IPV6 || nhg.type == fib::NEXTHOP_TYPE_IPV6_IFINDEX ||
        nhg.type == fib::NEXTHOP_TYPE_IPV4 || nhg.type == fib::NEXTHOP_TYPE_IPV4_IFINDEX) {
        m_gateway = gaddr_to_string(nhg.gate, nhg.type);
        SWSS_LOG_DEBUG("gateway address: %s", m_gateway.c_str());
    }
```

**Self-reference initialization**: When populating `m_resolved_enable_group`, iterate the depends list and set each entry to `true`. For leaf NHGs (`depends.empty()`), add `{self_id, true}` as a self-reference so the entry has a state that walk_spec can set to `false` when the gateway matches.

Note: Forward walk calculations are compared against this stored state to determine whether the node requires an update.

* New Field: `m_last_appdb_fields`
Caches the last APPDB field string written by this entry. Before calling `writeToDB()`, compare the new fields against this cached value. If identical, skip the write (compare-and-skip deduplication).

```
        /*
         * Cached APPDB fields for deduplication.
         * Compared before each writeToDB() to avoid redundant APPDB writes
         * when the flat resolved paths haven't changed.
         */
        string m_last_appdb_fields;
```

### New Helper Methods

#### `isAllDisabled()` (static file-scope helper)
Checks if all entries in a `RIBNHGEntry`'s `m_resolved_enable_group` are `false`. Used in walk_spec's Visit-for-State pattern: before evaluating relevance, each depends entry is checked — if all its paths are disabled, the corresponding entry in the parent's `m_resolved_enable_group` is marked disabled.

```cpp
static bool isAllDisabled(RIBNHGEntry* entry) {
    auto& enable_group = entry->getResolvedEnableGroup();
    for (const auto& kv : enable_group) {
        if (kv.second) return false;
    }
    return true;
}
```

#### `RIBNHGEntry::resetResolvedEnableGroup()`
Resets all entries in `m_resolved_enable_group` to `true`. Called when the NHG is updated by a new Add/Update event from FRR, restoring the entry to its fully-enabled state.

```cpp
void resetResolvedEnableGroup() {
    for (auto& kv : m_resolved_enable_group) {
        kv.second = true;
    }
}
```

#### `RIBNHGEntry::regenerateFields()`
Combines `getNHGFields()` + `syncFvVector()` into a single call. Used by the PIC backwalk after modifying `m_resolved_enable_group` — regenerates the flat nexthop/ifname/weight strings with disabled paths excluded, then syncs the FV vector.

```cpp
int regenerateFields() {
    if (getNHGFields() != 0) return -1;
    return syncFvVector();
}
```

#### `RIBNHGEntry::addDependentsMember()` / `removeDependentsMember()`
Manage the reverse-pointer graph. `addDependentsMember(id)` inserts `id` into `m_dependents`; `removeDependentsMember(id)` erases it.

#### `RIBNHGTable::addNHGDependents()` / `removeNHGDependents()`
Bulk reverse-pointer operations. For each entry ID in the `depends` set, look up its `RIBNHGEntry` and call `addDependentsMember(id)` or `removeDependentsMember(id)`. This ensures the dependency graph is bidirectional — when entry A depends on entry B, B's `m_dependents` contains A's ID, enabling the backwalk to discover A from B.

```cpp
int RIBNHGTable::addNHGDependents(std::set<uint32_t> depends, uint32_t id) {
    for (auto dep_id : depends) {
        RIBNHGEntry* e = getEntry(dep_id);
        if (e == nullptr) return -1;
        e->addDependentsMember(id);
    }
    return 0;
}

void RIBNHGTable::removeNHGDependents(std::set<uint32_t> depends, uint32_t id) {
    for (auto dep_id : depends) {
        RIBNHGEntry* e = getEntry(dep_id);
        if (e == nullptr) continue;
        e->removeDependentsMember(id);
    }
}
```

### Method Update: `RIBNHGEntry::getNextHopGroupFields()`
Before iterating `m_resolvedGroup` to build the flat nexthop/ifname/weight strings, call `resolveLeafEnableFlags()` to determine each leaf's effective enable/disable status. Skip any leaf that resolves to disabled.

#### New Helper: `resolveLeafEnableFlags()`

**Problem**: `m_resolved_enable_group` tracks depends-level IDs (direct children), while `m_resolvedGroup` contains leaf-level IDs (entries with `num_direct==0`). For intermediate nodes, these sets don't overlap — a direct lookup of a leaf ID in `m_resolved_enable_group` always misses.

**Solution**: Recursively walk the depends tree from the current node down to the leaves, respecting enable/disable gates at each level.

```cpp
std::unordered_map<uint32_t, bool> RIBNHGEntry::resolveLeafEnableFlags() {
    std::unordered_map<uint32_t, bool> result;

    /* Leaf node: m_resolved_enable_group has self-reference {self_id: bool} */
    if (m_depends.empty()) {
        for (const auto& kv : m_resolved_enable_group) {
            result[kv.first] = kv.second;
        }
        return result;
    }

    /* Non-leaf: initialize all leaves in m_resolvedGroup as disabled */
    for (const auto& nh : m_resolvedGroup) {
        result[nh.first] = false;
    }

    /* Walk each depends subtree */
    for (uint32_t dep_id : m_depends) {
        /* If this depends path is disabled at our level, skip entire subtree */
        auto it = m_resolved_enable_group.find(dep_id);
        if (it != m_resolved_enable_group.end() && !it->second) {
            continue;
        }

        /* Recurse into the dep entry to get its leaf flags */
        RIBNHGEntry* dep_entry = m_table->getEntry(dep_id);
        if (dep_entry == nullptr) continue;

        auto dep_leaf_flags = dep_entry->resolveLeafEnableFlags();

        /* Merge: a leaf is enabled if ANY enabled depends path has it enabled */
        for (const auto& lf : dep_leaf_flags) {
            if (lf.second && result.count(lf.first)) {
                result[lf.first] = true;
            }
        }
    }

    return result;
}
```

**Key properties**:
- **Leaf nodes**: direct lookup in own `m_resolved_enable_group` (self-reference `{self_id: bool}`)
- **ECMP nodes** (depends are leaves): one level of recursion resolves directly since leaf IDs match enable_group keys
- **Intermediate nodes**: recursion walks through depends → their depends → ... → leaves
- **Union semantics**: if a leaf is reachable via ANY enabled path, it's enabled
- **Recursion depth**: matches the NHG tree depth (typically 3-4 levels max)

#### Updated `getNextHopGroupFields()` Usage

```cpp
int RIBNHGEntry::getNextHopGroupFields() {
    // ...
    /* Resolve leaf-level enable flags by walking the depends tree */
    auto leaf_flags = resolveLeafEnableFlags();

    for (const auto &nh: m_resolvedGroup) {
        uint32_t id = nh.first;

        /* Skip disabled paths based on resolved leaf flags */
        auto leaf_it = leaf_flags.find(id);
        if (leaf_it != leaf_flags.end() && !leaf_it->second) {
            SWSS_LOG_NOTICE("NextHop id %d skipped (disabled via resolved leaf flags)", id);
            continue;
        }
        // ... build nexthop, ifname, weight strings from enabled entries
    }
    // ...
}
```

#### Example Trace (Topology 1, fc06::2 down, NHG 257)

NHG 257: `m_depends={238}`, `m_resolved_enable_group={238: true}`, `m_resolvedGroup={234, 237}`

1. `resolveLeafEnableFlags()` on NHG 257:
   - dep 238 enabled → recurse into 238
   - `resolveLeafEnableFlags()` on NHG 238:
     - `m_depends={234, 237}`, `m_resolved_enable_group={234: true, 237: false}`
     - dep 234 enabled → recurse into 234 (leaf): returns `{234: true}`
     - dep 237 disabled → skip
     - returns `{234: true, 237: false}`
   - Merge into result: `{234: true, 237: false}`
2. `getNextHopGroupFields()` skips 237, includes only 234 → APPDB output: `{fc08::2}`


### `fib_nhg_walk_spec_for_node_quick_fixup()`
This function provides the Part 1-specific implementation of the `fib_nhg_walk_spec_func` callback. The caller registers this function before initiating the backwalk, ensuring consistent logic across all nodes in a single traversal.

* Input: Reference to the current `RIBNHGEntry` node and `fib_nhg_walking_ctx&`.
* Output: bool indicating traversal continuation.
  * true: Fixup succeeded; backwalk should continue from this node.
  * false: Fixup was unnecessary or failed; backwalk halts. Returns false if the NHG is unrelated to the impacted nexthop or has already been updated.
* Execution Logic:
  1. **Visit-for-State (first)**: For each depends entry, retrieve its `RIBNHGEntry` via `ctx.rib_nhg_table->getEntry()`. If `isAllDisabled(dep_entry)` returns true, mark that dep as disabled in the current node's `m_resolved_enable_group`. This propagates disabled state from children before evaluating relevance.
  2. **Relevance Evaluation**: Determine if the node requires a fixup by checking:
    * Gateway match: If the entry's gateway address matches `ctx.nexthop_address`:
      * Leaf NHG (no depends): mark self-reference disabled in `m_resolved_enable_group`, add to `modified_node_set`, return true immediately.
      * Non-leaf: mark ALL entries in `m_resolved_enable_group` as disabled.
    * Modified depends: If no gateway match, check if any depends entry appears in `ctx.modified_node_set`.
  3. **All-disabled check**: If all entries in `m_resolved_enable_group` are disabled, add to `modified_node_set` and skip APPDB write (return true).
  4. **Database Synchronization**: Call `entry->regenerateFields()` (which invokes `getNHGFields()` + `syncFvVector()`) to regenerate the flat nexthop strings with disabled paths excluded. Then call `ctx.rib_nhg_table->writeToDB(entry)` to update APPDB — guarded by `needCreateSonicObject() && getSonicObjID() != 0`.
     * APPDB writes are skipped when:
       1. All paths are disabled (no valid nexthop to write).
       2. SONiC ID is zero (entry not assigned to APPDB, used for convergence walk only).

* Walk Spec Decision Flowchart

```mermaid
flowchart TD
    START(["walk_spec(entry, ctx)"]) --> GET_DEPS["Retrieve depends list"]
    GET_DEPS --> EVAL_STATE["Visit-for-State:<br/>For each depends entry,<br/>retrieve its RIBNHGEntry.<br/>If ALL m_resolved_enable_group<br/>entries are false, mark disabled<br/>in current node"]

    EVAL_STATE --> CHECK_GW{"Gateway matches<br/>impacted nexthop?"}

    CHECK_GW -->|Yes| IS_LEAF{"Leaf NHG?<br/>no depends"}
    IS_LEAF -->|Yes| DISABLE_SELF["Mark self-reference<br/>disabled in<br/>m_resolved_enable_group"]
    IS_LEAF -->|No| DISABLE_ALL["Mark ALL depends<br/>disabled in<br/>m_resolved_enable_group"]

    DISABLE_SELF --> REGEN["getNextHopGroupFields<br/>recursive resolution"]
    DISABLE_ALL --> REGEN

    CHECK_GW -->|No| CHECK_MOD{"Any depends entry<br/>in modified_node_set?"}
    CHECK_MOD -->|Yes| REGEN
    CHECK_MOD -->|No| RETURN_FALSE(["return false<br/>visited for state only"])

    REGEN --> SINGLE_PATH{"Single remaining path<br/>being disabled?"}
    SINGLE_PATH -->|Yes| SKIP_APPDB["Skip APPDB write"]
    SINGLE_PATH -->|No| WRITE_DB["writeToDB to APPDB"]

    SKIP_APPDB --> ADD_MOD["Add node to<br/>modified_node_set"]
    WRITE_DB --> ADD_MOD
    ADD_MOD --> RETURN_TRUE(["return true<br/>fixup applied"])
```

### `fib_nhg_prune_spec_for_node_quick_fixup()`
This function implements the Part 1-specific `fib_nhg_prune_spec_func` callback. Like the walk spec, it is registered by the caller prior to traversal.

* Input:
  * Reference to the current `RIBNHGEntry` node.
  * Boolean return value from the walk spec function.
  * Reference to `fib_nhg_walking_ctx` for accessing shared traversal state.
* Output: bool indicating whether to prune (halt) the backwalk at this node.
* Prune logic
```
fib_nhg_prune_spec_for_node_quick_fixup(entry, walk_result, ctx):
    // Rule 1: Multi-path nodes where nexthop wasn't matched always continue --
    //         dependents above may have gateways matching the nexthop
    if entry.depends.size() >= 2 and walk_result == false:
        return false   // don't prune

    // Rule 2: No update means no propagation needed
    if walk_result == false:
        return true    // prune

    return false       // node was modified, continue
```
### Part 1's call flow
```mermaid
sequenceDiagram
    participant Zebra as Zebra (FRR)
    participant FPM as dplane_fpm_sonic
    participant Syncd as fpmsyncd
    participant Trigger as fib_nhg_trigger_node_quick_fixup()
    participant Table as RIBNHGTable
    participant Walk as fib_nhg_back_walk()
    participant WSpec as walk_spec()
    participant PSpec as prune_spec()
    participant Entry as RIBNHGEntry
    participant APPDB as APP_DB

    Note over Zebra,APPDB: Phase 1: current_resolved_nhg_id == 0 (nexthop unreachable)

    Zebra->>FPM: DPLANE_OP_NHT_EVENT_UPDATE<br/>(rnh_prefix, prev_resolved_nhg_id,<br/>curr_resolved_nhg_id=0)
    FPM->>Syncd: FPM NHT message<br/>(nexthop_addr, resolved_nhg_id)
    Syncd->>Trigger: call(nexthop_addr, resolved_nhg_id)

    Note over Trigger: Part 1: Global Table Context
    Trigger->>Table: getEntry(resolved_nhg_id)
    Table-->>Trigger: RIBNHGEntry* (or null)

    alt resolved_nhg_id lookup fails (connected-route NHG not in table)
        Trigger->>Table: getGlobalEntries(nexthop_addr)
        Table-->>Trigger: set<RIBNHGEntry*> (0..N)
    end

    loop For each global entry
        Trigger->>Trigger: Initialize fib_nhg_walking_ctx:<br/>- visited_node_set = {}<br/>- modified_node_set = {}<br/>- walk_spec = quick_fixup<br/>- prune_spec = quick_fixup<br/>- nexthop_address<br/>- rib_nhg_table = table

        Trigger->>Walk: fib_nhg_back_walk(start_entry_id, ctx)

            loop DFS traversal
                Walk->>Table: getEntry(id)
                Table-->>Walk: RIBNHGEntry*

                Walk->>WSpec: walk_result = walk_spec(entry, ctx)
                Note over WSpec: See Section Walk Spec Decision Flowchart
                WSpec->>Entry: Visit-for-State:<br/>isAllDisabled(dep) check
                WSpec->>WSpec: Check relevance:<br/>(a) gateway match<br/>(b) depends in modified_node_set

                alt Relevant (fixup needed)
                    WSpec->>Entry: update m_resolved_enable_group
                    WSpec->>Entry: regenerateFields()
                    Entry->>APPDB: writeToDB() [if not all-disabled]
                    WSpec->>WSpec: Add id to modified_node_set
                    WSpec-->>Walk: return true
                else Not relevant (visit-for-state only)
                    WSpec-->>Walk: return false
                end

                Walk->>Walk: Add id to visited_node_set

                Walk->>PSpec: call(entry, walk_result)
                Note over PSpec: ECMP + not matched → don't prune<br/>Not modified → prune<br/>Modified → don't prune
                alt Pruned
                    Walk->>Walk: return (stop this branch)
                end

                Walk->>Entry: get dependents list
                loop For each dependent_id
                    Walk->>Walk: recurse(dependent_id, ctx)
                end
            end
    end
```

## Changes for Part 2
### Motivation
`RIBNHGEntry` objects used to create SONiC NHG table entries carry VPN context information, whereas NHT-resolved NHGs from Zebra do not. Consequently, a standard backwalk traversal cannot naturally reach these VPN-scoped entries. To bridge this gap, we introduce an explicit lookup mechanism.

### Modifications to `RIBNHGTable` Class
* New Field: `m_nexthop_to_global_RIBNHG`
A new index maps a nexthop address string to the set of `RIBNHGEntry*` pointers in the **global routing table context** (default VRF). This map is the fallback for Part 1 when `resolved_nhg_id` lookup fails (connected nexthop case):
```
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_global_RIBNHG;
```

* Renamed Field: `m_nexthop_to_vrf_RIBNHG` (was `m_nexthop_to_RIBNHG_map`)
The existing VRF/VPN nexthop map is renamed for clarity. It maps a nexthop address string to the set of `RIBNHGEntry*` pointers that reference it in **VRF context** (VPN NHGs):
```
    std::map<std::string, std::set<RIBNHGEntry*>> m_nexthop_to_vrf_RIBNHG;
```

* Supporting APIs (uniform interface for both maps)
```
    // Global map APIs (Part 1 fallback)
    void addGlobalEntry(const std::string& nexthop, RIBNHGEntry* entry) {
        m_nexthop_to_global_RIBNHG[nexthop].insert(entry);
    }

    bool removeGlobalEntry(const std::string& nexthop, RIBNHGEntry* entry) {
        auto it = m_nexthop_to_global_RIBNHG.find(nexthop);
        if (it == m_nexthop_to_global_RIBNHG.end()) return false;
        if (it->second.erase(entry) == 0) return false;
        if (it->second.empty()) m_nexthop_to_global_RIBNHG.erase(it);
        return true;
    }

    const std::set<RIBNHGEntry*>& getGlobalEntries(const std::string& nexthop) const {
        static const std::set<RIBNHGEntry*> emptySet;
        auto it = m_nexthop_to_global_RIBNHG.find(nexthop);
        return it != m_nexthop_to_global_RIBNHG.end() ? it->second : emptySet;
    }

    // VRF map APIs (Part 2, renamed from addEntry/removeEntry/getEntries)
    void addVrfEntry(const std::string& nexthop, RIBNHGEntry* entry) {
        m_nexthop_to_vrf_RIBNHG[nexthop].insert(entry);
    }

    bool removeVrfEntry(const std::string& nexthop, RIBNHGEntry* entry) {
        auto it = m_nexthop_to_vrf_RIBNHG.find(nexthop);
        if (it == m_nexthop_to_vrf_RIBNHG.end()) return false;
        if (it->second.erase(entry) == 0) return false;
        if (it->second.empty()) m_nexthop_to_vrf_RIBNHG.erase(it);
        return true;
    }

    const std::set<RIBNHGEntry*>& getVrfEntries(const std::string& nexthop) const {
        static const std::set<RIBNHGEntry*> emptySet;
        auto it = m_nexthop_to_vrf_RIBNHG.find(nexthop);
        return it != m_nexthop_to_vrf_RIBNHG.end() ? it->second : emptySet;
    }
```
* Integration Points
  * **Global map population**: `addGlobalEntry()` is called in `NHGMgr::addNewNHGFull()` when the entry belongs to the global routing table (NOT VPN context) AND has a non-empty gateway address. This includes leaf NHGs (e.g., NHG 237 with gateway fc06::2) and intermediate NHGs (e.g., NHG 258 with gateway 2064:200::1e). ECMP NHGs without explicit gateway are excluded.
  * **Global map removal**: `removeGlobalEntry()` is called in `NHGMgr::delNHGFull()` for global-table entries with gateway.
  * **VRF map population**: `addVrfEntry()` is called in `NHGMgr::addNewNHGFull()` when `entry->needCreateSonicObject()` returns true (unchanged logic, just renamed).
  * **VRF map removal**: `removeVrfEntry()` is called in `NHGMgr::delNHGFull()` when `entry->hasSonicGatewayObj()` returns true.

### Backwalk for VPN-Scoped RIBNHGEntries
For each `RIBNHGEntry*` returned by `getVrfEntries(nexthop)`, we explicitly trigger `fib_nhg_back_walk()` to propagate state changes.

### `fib_nhg_walk_spec_for_node_quick_fixup_sonic_nhg()`
This variant of the walk spec function is tailored for SONiC NHG updates:
1. **Scope-Limited Updates**: Only modifies the SONiC NHG representation within the RIBNHGEntry; no changes are made to the PIC (Platform Independent Context) state.
2. **Deduplication Tracking**: Since multiple `RIBNHGEntry` instances may map to a single SONiC NHG (many-to-one relationship), the `updated_sonic_nhg_keys` field (`std::set<std::string>`) in the walking context tracks which SONiC NHG keys have already been updated during the traversal. The key is formed as `nexthop + "|" + vpnSid + "|" + segSrc`. If the key is already present, the write is skipped.

### `fib_nhg_prune_spec_for_node_quick_fixup_sonic_nhg()`
This prune spec function currently mirrors the behavior of f`ib_nhg_prune_spec_for_node_quick_fixup()`:
* Halts traversal if the node was not modified by the walk spec.

Note: Future enhancements may introduce VPN-specific pruning logic if use cases warrant it.

### Part 2's call flow
```mermaid
sequenceDiagram
    participant Trigger as fib_nhg_trigger_node_quick_fixup()
    participant Table as RIBNHGTable
    participant Walk as fib_nhg_back_walk()
    participant WSpec as walk_spec_sonic_nhg()
    participant PSpec as prune_spec_sonic_nhg()
    participant Entry as RIBNHGEntry
    participant APPDB as APP_DB

    Note over Trigger,APPDB: Part 2: VPN Context (after Part 1 completes)

    Trigger->>Table: getVrfEntries(nexthop_addr)
    Table-->>Trigger: set<RIBNHGEntry*>

    loop For each VPN-scoped RIBNHGEntry*
        Trigger->>Trigger: Initialize new fib_nhg_walking_ctx:<br/>- walk_spec = sonic_nhg<br/>- prune_spec = sonic_nhg<br/>- updated_sonic_nhg_keys = {}

        Trigger->>Walk: call(entry.id, ctx)

        loop DFS traversal (same structure as Part 1)
            Walk->>WSpec: call(entry, ctx)
            Note over WSpec: Scope-limited: only SONiC NHG,<br/>no PIC state changes.<br/>Dedup via updated_sonic_nhg_keys.
            alt SONiC NHG key not yet updated
                WSpec->>Entry: update SONiC NHG representation
                WSpec->>APPDB: writeToDB()
                WSpec->>WSpec: Add key to updated_sonic_nhg_keys
                WSpec-->>Walk: return true
            else Already updated (dedup)
                WSpec-->>Walk: return false
            end

            Walk->>PSpec: call(entry, walk_result)
            Note over PSpec: Same logic: prune if walk_result == false
        end
    end
```

### One variation for part 1
In the following example, 2064:100::1d points NHG 235, which points to NHG 232 and 236. Both NHG 232 and 236's nexthop is /128 prefix. 

```
PE3# show ipv6 route 2064:100::1d next
Routing entry for 2064:100::1d/128
  Known via "bgp", distance 20, metric 0, best
  Last update 16:27:18 ago
  Nexthop Group ID: 235
  Received Nexthop Group ID: 233
  * fc06::2, via Ethernet12, weight 1
  * fc08::2, via Ethernet4, weight 1

PE3# show nexthop rib 235
ID: 235 (zebra)
     RefCnt: 15     Flags: 0x3
     Uptime: 16:27:38
     VRF: default(No AFI)
     Nexthop Count: 2
     Valid, Installed
     Depends: (232) (236)
           via fc06::2, Ethernet12 (vrf default), weight 1
           via fc08::2, Ethernet4 (vrf default), weight 1
PE3# show nexthop rib 232
ID: 232 (zebra)
     RefCnt: 19     Flags: 0x3
     Uptime: 16:27:41
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
     Interface Index: 140
           via fc06::2, Ethernet12 (vrf default), weight 1
     Dependents: (235)
PE3# show nexthop rib 236
ID: 236 (zebra)
     RefCnt: 19     Flags: 0x3
     Uptime: 16:27:49
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
     Interface Index: 143
           via fc08::2, Ethernet4 (vrf default), weight 1
     Dependents: (235)
                                          via fc08::2, Ethernet4, weight 1, 3d02h57m
```

But since fc06::2 and fc08::2 are connected routes, zebra would resolve them via its interface prefix. Zebra doesn't have ARP/ND learnt routes in its LPM table.
```
PE3# show ipv6 route fc06::2 next
Routing entry for fc06::/120
  Known via "connected", distance 0, metric 0, best
  Last update 16:54:22 ago
  Nexthop Group ID: 210
  Received Nexthop Group ID: 209
  * directly connected, Ethernet12, weight 1

Routing entry for fc06::/120
  Known via "kernel", distance 0, metric 256
  Last update 16:54:22 ago
  Nexthop Group ID: 210
  Received Nexthop Group ID: 209
  * directly connected, Ethernet12, weight 1

PE3# show nexthop rib 210
ID: 210 (zebra)
     RefCnt: 7     Flags: 0x203
     Uptime: 16:35:26
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed, Initial Delay
     Interface Index: 140
           is directly connected, Ethernet12 (vrf default), weight 1
PE3# 
```
So their NHT would be via connected routes' NHG instead of ARP/ND learnt routes' NHG. This would lead NHG look up fail in part one. 
```
fc06::2(Connected)
 resolved via connected, prefix fc06::/120
 is directly connected, Ethernet12 (vrf default), weight 1
 Client list: bgp(fd 71)
fc08::2(Connected)
 resolved via connected, prefix fc08::/120
 is directly connected, Ethernet4 (vrf default), weight 1
 Client list: bgp(fd 71)
```

Therefore, we need `m_nexthop_to_global_RIBNHG` to track all interface's nexthops (single-path nexthop RIBNHGEntries in the global table context). In Part 1, if `getEntry(resolved_nhg_id)` fails (because `resolved_nhg_id` points to the connected-route NHG, e.g., NHG 210, which is NOT in our RIBNHGEntry table), we fall back to `m_nexthop_to_global_RIBNHG.getGlobalEntries(nexthop_addr)` to find the set of leaf RIBNHGEntries whose gateway matches the nexthop address (e.g., NHG 232 with gateway fc06::2). In Part 2, `m_nexthop_to_vrf_RIBNHG.getVrfEntries(nexthop_addr)` returns VPN-scoped RIBNHGEntries for the VRF context backwalk.


# 7. Examples
## Test Topology 1 Global table recursive routes
Assume we have 2064:100::1d/128 and 2064:200::1e/128 learnt via eBGP. Each route has two paths via fc06::2 and fc08::2

```
B>* 2064:100::1d/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
B>* 2064:200::1e/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
```
Then we add the following static configurations
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 2064:200::1e
ipv6 route 3::3/128 1::1
ipv6 route 3::3/128 2::2
ipv6 route 4::4/128 1::1
```

The zebra's nexthop groups would be created as
```
PE3# show ipv6 route 1::1 next
Routing entry for 1::1/128
  Known via "static", distance 1, metric 0, best
  Last update 00:12:12 ago
  Nexthop Group ID: 256
  Received Nexthop Group ID: 254
    2064:100::1d (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
  *   fc08::2, via Ethernet4, weight 1
    2064:200::1e (recursive), weight 1
      fc06::2, via Ethernet12 (duplicate nexthop removed), weight 1
      fc08::2, via Ethernet4 (duplicate nexthop removed), weight 1

PE3# show ipv6 route 2::2  next
Routing entry for 2::2/128
  Known via "static", distance 1, metric 0, best
  Last update 00:12:20 ago
  Nexthop Group ID: 258
  Installed Nexthop Group ID: 238
  Received Nexthop Group ID: 255
    2064:200::1e (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
  *   fc08::2, via Ethernet4, weight 1

PE3# show ipv6 route 3::3  next
Routing entry for 3::3/128
  Known via "static", distance 1, metric 0, best
  Last update 00:08:09 ago
  Nexthop Group ID: 262
  Received Nexthop Group ID: 260
    1::1 (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
  *   fc08::2, via Ethernet4, weight 1
    2::2 (recursive), weight 1
      fc06::2, via Ethernet12 (duplicate nexthop removed), weight 1
      fc08::2, via Ethernet4 (duplicate nexthop removed), weight 1

PE3# show ipv6 route 4::4  next
Routing entry for 4::4/128
  Known via "static", distance 1, metric 0, best
  Last update 00:08:17 ago
  Nexthop Group ID: 263
  Installed Nexthop Group ID: 238
  Received Nexthop Group ID: 259
    1::1 (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
  *   fc08::2, via Ethernet4, weight 1
```

These nexthop's NHGFUll Json objects are stored in test_topology_1.json. The node graph is shown as the following.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'background': '#ffffff', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'nodeTextColor': '#000000', 'lineColor': '#E6C229', 'edgeLabelBackground': '#ffffff', 'primaryBorderColor': '#cccccc'}}}%%
graph TD
    %% ───────────── NODE DEFINITIONS ─────────────
    N234["234, fc08::2"]
    N237["237, fc06::2"]
    N238["238"]
    N257["257, 2064:100::1d"]
    N258["258, 2064:200::1e"]
    N263["263, 1::1"]
    N264["264, 2::2"]
    N256["256"]
    N262["262"]

    %% ───────────── DEPENDENCY EDGES ─────────────
    N234 --> N238
    N237 --> N238
    N238 --> N257
    N238 --> N258
    N238 --> N263
    N238 --> N264
    N257 --> N256
    N258 --> N256
    N263 --> N262
    N264 --> N262

    %% ───────────── STYLING ─────────────
    classDef base fill:#a8d5a2,stroke:#2d5a2d,stroke-width:2px;
    classDef core fill:#f9d079,stroke:#8c5e1c,stroke-width:2px;
    classDef inter fill:#b3cde0,stroke:#2a4d69,stroke-width:2px;
    classDef leaf fill:#f0a8a8,stroke:#8b1c1c,stroke-width:2px;

    class N234,N237 base;
    class N238 core;
    class N257,N258,N263,N264 inter;
    class N256,N262 leaf;
```
Above graph could be presented as the following table

| NHG | Role | Gateway(s) | Depends | Dependents | m_resolved_enable_group init |
|:---|:---|:---|:---|:---|:---|
| 234 | leaf | fc08::2 | -- | [238] | {234: true} (self-ref) |
| 237 | leaf | fc06::2 | -- | [238] | {237: true} (self-ref) |
| 238 | core ECMP | fc06::2, fc08::2 | [234, 237] | [257, 258, 263, 264] | {234: true, 237: true} |
| 257 | intermediate | 2064:100::1d | [238] | [256] | {238: true} |
| 258 | intermediate | 2064:200::1e | [238] | [256] | {238: true} |
| 263 | intermediate | 1::1 | [238] | [262] | {238: true} |
| 264 | intermediate | 2::2 | [238] | [262] | {238: true} |
| 256 | route 1::1 | composite | [257, 258] | -- | {257: true, 258: true} |
| 262 | route 3::3 | composite | [263, 264] | -- | {263: true, 264: true} |
### Local Failure fc06::2 withdrawn
The NHT event contains "nexthop=fc06::2, resolved_nhg_id=210" (connected-route NHG for fc06::/120).

**Entry point resolution**: `getEntry(210)` returns nullptr (NHG 210 is a connected-route NHG, not in our RIBNHGEntry table). Fallback: `getGlobalEntries("fc06::2")` returns {NHG 237} (leaf, gateway fc06::2).
FIB triggers the recursive walk via DFS from 237: `237 → 238 → 257 → 256 → 258 → 256 (revisit) → 263 → 262 → 264 → 262 (revisit)` (234 never reached -- not in backwalk path from 237)

The detailed handling procedure is the following and the initial modified_set is empty.
1. 237 (STARTING, leaf fc06::2):
  * Gateway matches nexthop. Self-ref disabled: `{237: false}`. Skip APPDB (single path).
  * `modified_set += 237`.
2. 238 (ECMP, depends [234, 237]):
  * 237 in modified_set → relevant. 237 fully disabled → mark: `{234: true, 237: false}`.
  * Flat: `{fc08::2}`. APPDB written. `modified_set += 238`.
3. 257 (2064:100::1d, depends [238]):
  * 238 in modified_set → relevant. No gateway match (2064:100::1d != fc06::2).
  * Flat: 238 → `{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 257`.
4. 256 (route 1::1, depends [257, 258]):
  * 257 in modified_set → relevant.
  * Flat: 257→238→`{fc08::2}`, 258→238→`{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 256`.
5. 258 (2064:200::1e, depends [238]):
  * 238 in modified_set → relevant.
  * Flat: 238 → `{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 258`.
6. 256 (revisit, depends [257, 258]):
  * 258 now also in modified_set. Regen: same `{fc08::2}`. APPDB same (compare-and-skip).
7. 263 (1::1, depends [238]):
  * 238 in modified_set → relevant.
  * Flat: 238 → `{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 263`.
8. 262 (route 3::3, depends [263, 264]):
  * 263 in modified_set → relevant.
  * Flat: 263→238→`{fc08::2}`, 264→238→`{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 262`.
9. 264 (2::2, depends [238]):
  * 238 in modified_set → relevant.
  * Flat: 238 → `{fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 264`.
10. 262 (revisit, depends [263, 264]):
  * 264 now also in modified_set. Regen: same `{fc08::2}`. APPDB same (compare-and-skip).

The final result:
* **APPDB Updated**: {238, 257, 258, 263, 264, 256, 262}
* **State Modified (skip APPDB)**: {237}
* **Not Reached**: {234}


**Call flow trace through the sequence diagram**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(237) | Leaf. Gateway fc06::2 matches nexthop. Self-ref: {237: false}. Single path -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[238] | -- | {237} |
| 2 | back_walk(238) | ECMP. Depends [234,237]. 237 fully disabled -> mark {234:true, 237:false}. Relevance: 237 in modified_set. Regen: {fc08::2}. | true | not pruned (modified) | yes -> dependents=[257,258,263,264] | 238: {fc08::2} | {237,238} |
| 3 | back_walk(257) | Depends [238]. 238 NOT fully disabled (partial). No gateway match (2064:100::1d != fc06::2). But 238 in modified_set -> relevant. Regen via 238 -> {fc08::2}. | true | not pruned | yes -> dependents=[256] | 257: {fc08::2} | {237,238,257} |
| 4 | back_walk(256) | Depends [257,258]. 257 in modified_set -> relevant. Regen: 257->{fc08::2}, 258 via 238->{fc08::2}. | true | not pruned | yes -> dependents=[] (none) | 256: {fc08::2} | {237,238,257,256} |
| 5 | back_walk(258) | Depends [238]. 238 in modified_set -> relevant. Regen via 238 -> {fc08::2}. | true | not pruned | yes -> dependents=[256] | 258: {fc08::2} | {237,238,257,256,258} |
| 6 | back_walk(256) | **Revisit**. Depends [257,258]. 258 now also in modified_set. Regen: same {fc08::2}. | true | not pruned | yes -> dependents=[] | 256: {fc08::2} (same) | {237,238,257,256,258} |
| 7 | back_walk(263) | Depends [238]. 238 in modified_set -> relevant. Regen via 238 -> {fc08::2}. | true | not pruned | yes -> dependents=[262] | 263: {fc08::2} | {237,238,257,256,258,263} |
| 8 | back_walk(262) | Depends [263,264]. 263 in modified_set -> relevant. Regen: 263->{fc08::2}, 264 via 238->{fc08::2}. | true | not pruned | yes -> dependents=[] | 262: {fc08::2} | {237,238,257,256,258,263,262} |
| 9 | back_walk(264) | Depends [238]. 238 in modified_set -> relevant. Regen via 238 -> {fc08::2}. | true | not pruned | yes -> dependents=[262] | 264: {fc08::2} | {237,238,257,256,258,263,262,264} |
| 10 | back_walk(262) | **Revisit**. Depends [263,264]. 264 now also in modified_set. Regen: same {fc08::2}. | true | not pruned | yes -> dependents=[] | 262: {fc08::2} (same) | {237,238,257,256,258,263,262,264} |

### Remote Failure 1 (2064:200::1e withdrawn)
The NHT event contains "nexthop=2064:200::1e, resolved_nhg_id=258" (route-level NHG, gateway 2064:200::1e).
FIB triggers `fib_nhg_back_walk(258, ctx)`.

1. **258** (STARTING, gateway `2064:200::1e`):
   - Gateway matches nexthop.
   - Disable all depends: `{238: false}`. Single depend, fully disabled -> skip APPDB.
   - `modified_set = {258}`. Continue to dependents `[256]`.
2. **256** (route 1::1, depends `[257, 258]`):
   - `258` in `modified_set` -> relevant!
   - `258` fully disabled -> mark: `{257: true, 258: false}`.
   - Regen: `257 -> 238 -> {fc06::2, fc08::2}`. APPDB: `{fc06::2, fc08::2}`.
   - `modified_set += 256`. Dependents=[] -> end.

The final result:
* **APPDB Updated**: {256}
* **State Modified (skip APPDB)**: {258}
* **Not Reached**: {234, 237, 238, 257, 263, 264, 262}

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(258) | Gateway 2064:200::1e == nexthop -> **match**. Disable all depends: {238: false}. Single depend, fully disabled -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[256] | -- | {258} |
| 2 | back_walk(256) | Depends [257, 258]. 258 fully disabled -> mark {257:true, 258:false}. 258 in modified_set -> relevant. Regen: 257->{fc06::2, fc08::2} (via 238, all enabled). | true | not pruned | yes -> dependents=[] | 256: {fc06::2, fc08::2} | {258, 256} |


### Remote Failure 2 (1::1 withdrawn)
The NHT event contains "nexthop=1::1, resolved_nhg_id=263" (route-level NHG, gateway 1::1).
FIB triggers `fib_nhg_back_walk(263, ctx)`.

1. **263** (STARTING, gateway `1::1`):
   - Gateway matches nexthop.
   - Disable all depends: `{238: false}`. Single depend, fully disabled -> skip APPDB.
   - `modified_set = {263}`. Continue to dependents `[262]`.
2. **262** (route 3::3, depends `[263, 264]`):
   - `263` in `modified_set` -> relevant!
   - `263` fully disabled -> mark: `{263: false, 264: true}`.
   - Regen: `264 -> 238 -> {fc06::2, fc08::2}`. APPDB: `{fc06::2, fc08::2}`.
   - `modified_set += 262`. Dependents=[] -> end.

The final result:
* **APPDB Updated**: {262}
* **State Modified (skip APPDB)**: {263}
* **Not Reached**: {234, 237, 238, 257, 258, 264, 256}

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(263) | Gateway 1::1 == nexthop -> **match**. Disable all depends: {238: false}. Single depend, fully disabled -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[262] | -- | {263} |
| 2 | back_walk(262) | Depends [263, 264]. 263 fully disabled -> mark {263:false, 264:true}. 263 in modified_set -> relevant. Regen: 264->{fc06::2, fc08::2} (via 238, all enabled). | true | not pruned | yes -> dependents=[] | 262: {fc06::2, fc08::2} | {263, 262} |

### Summary of corrected Topology 1 expectations

| Test Case | APPDB Updated | State Modified (skip APPDB) | Not Reached |
|:---|:---|:---|:---|
| Local Failure (fc06::2) | 238, 257, 258, 263, 264, 256, 262 | 237 | 234 |
| Remote Failure 1 (2064:200::1e withdrawn) | 256 | 258 | 234, 237, 238, 257, 263, 264, 262 |
| Remote Failure 2 (1::1 withdrawn) | 262 | 263 | 234, 237, 238, 257, 258, 264, 256 |


## Test Topology 2 Global table recursive routes
Assume we have 2064:100::1d/128 has two nexthops which are via Ethernet12 and Ethernet4, respectively.

```
B>* 2064:100::1d/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
B>* 2064:200::1e/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
```

We then added the following static configurations to construct an example that illustrates the complexity of handling various NHG group scenarios. Similar route dependencies could also be achieved through a routing protocol.
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 fc06::2
ipv6 route 3::3/128 fc08::2
ipv6 route 4::4/128 2::2
ipv6 route 4::4/128 3::3
```

In zebra, the routes are pointing to the following nexthops or nexthop groups.
```
PE3# show ipv6 route 1::1 next
Routing entry for 1::1/128
  Known via "static", distance 1, metric 0, best
  Last update 00:00:36 ago
  Nexthop Group ID: 263
  Received Nexthop Group ID: 261
    2064:100::1d (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
  *   fc08::2, via Ethernet4, weight 1
    2064:200::1e (recursive), weight 1
      fc06::2, via Ethernet12 (duplicate nexthop removed), weight 1
      fc08::2, via Ethernet4 (duplicate nexthop removed), weight 1

PE3# show ipv6 route 2::2  next
Routing entry for 2::2/128
  Known via "static", distance 1, metric 0, best
  Last update 00:00:55 ago
  Nexthop Group ID: 235
  Received Nexthop Group ID: 233
  * fc06::2, via Ethernet12, weight 1

PE3# show ipv6 route 3::3  next
Routing entry for 3::3/128
  Known via "static", distance 1, metric 0, best
  Last update 00:00:58 ago
  Nexthop Group ID: 232
  Received Nexthop Group ID: 231
  * fc08::2, via Ethernet4, weight 1

PE3# show ipv6 route 4::4   next
Routing entry for 4::4/128
  Known via "static", distance 1, metric 0, best
  Last update 00:00:49 ago
  Nexthop Group ID: 269
  Received Nexthop Group ID: 267
    2::2 (recursive), weight 1
  *   fc06::2, via Ethernet12, weight 1
    3::3 (recursive), weight 1
  *   fc08::2, via Ethernet4, weight 1

PE3# show ipv6 route 2064:100::1d next
Routing entry for 2064:100::1d/128
  Known via "bgp", distance 20, metric 0, best
  Last update 00:01:22 ago
  Nexthop Group ID: 236
  Received Nexthop Group ID: 234
  * fc06::2, via Ethernet12, weight 1
  * fc08::2, via Ethernet4, weight 1

PE3# show ipv6 route 2064:200::1e next
Routing entry for 2064:200::1e/128
  Known via "bgp", distance 20, metric 0, best
  Last update 00:01:40 ago
  Nexthop Group ID: 236
  Received Nexthop Group ID: 234
  * fc06::2, via Ethernet12, weight 1
  * fc08::2, via Ethernet4, weight 1

```
These nexthop's NHGFUll Json objects are stored in test_topology_2.json. The node graph is shown as the following.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'background': '#ffffff', 'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'nodeTextColor': '#000000', 'lineColor': '#E6C229', 'edgeLabelBackground': '#ffffff', 'primaryBorderColor': '#cccccc'}}}%%
graph TD
    %% Node Definitions
    N232["232, fc08::2"]
    N235["235, fc06::2"]
    N236["236"]
    N270["270, 2::2"]
    N266["266, 3::3"]
    N260["260, 2064:100::1d"]
    N264["264, 2064:200::1e"]
    N263["263"]
    N269["269"]

    %% Dependency Edges
    N232 --> N236
    N232 --> N270
    N235 --> N236
    N235 --> N266
    N236 --> N260
    N236 --> N264
    N270 --> N269
    N266 --> N269
    N260 --> N263
    N264 --> N263

    %% Optional Styling for better visualization
    classDef base fill:#90EE90,stroke:#333,stroke-width:2px;
    classDef mid fill:#FFD700,stroke:#333,stroke-width:2px;
    classDef leaf fill:#FF7F7F,stroke:#333,stroke-width:2px;

    class N232,N235 base;
    class N236,N270,N266 mid;
    class N260,N264,N263,N269 leaf;
```
Above graph could be presented as the following table

| NHG | Role | Gateway(s) | Depends | Dependents | m_resolved_enable_group init |
|:---|:---|:---|:---|:---|:---|
| 232 | leaf | `fc08::2` | -- | `[236, 270]` | `{232: true}` (self-ref) |
| 235 | leaf | `fc06::2` | -- | `[236, 266]` | `{235: true}` (self-ref) |
| 236 | core ECMP | `fc06::2`, `fc08::2` | `[232, 235]` | `[260, 264]` | `{232: true, 235: true}` |
| 260 | intermediate | `2064:100::1d` | `[236]` | `[263]` | `{236: true}` |
| 264 | intermediate | `2064:200::1e` | `[236]` | `[263]` | `{236: true}` |
| 263 | route 1::1 | composite | `[260, 264]` | -- | `{260: true, 264: true}` |
| 266 | intermediate | `3::3` | `[235]` | `[269]` | `{235: true}` |
| 270 | intermediate | `2::2` | `[232]` | `[269]` | `{232: true}` |
| 269 | route 4::4 | composite | `[266, 270]` | -- | `{266: true, 270: true}` |

### Local Failure (`fc06::2` withdrawn)
**NHT:** `nexthop=fc06::2`, `resolved_nhg_id=210` (connected-route NHG for fc06::/120)

**Entry point resolution**: `getEntry(210)` returns nullptr (NHG 210 is a connected-route NHG, not in our RIBNHGEntry table). Fallback: `getGlobalEntries("fc06::2")` returns {NHG 235} (leaf, gateway fc06::2).

**DFS order from 235:** `235 → 236 → 260 → 263 → 264 → 263 (revisit) → 266 → 269`  
*(270, 232 never reached -- not in backwalk path from 235)*

1. **235** (STARTING, leaf `fc06::2`):
   - Match. Self-ref disabled. Skip APPDB. `modified_set += 235`.
2. **236** (ECMP, depends `[232, 235]`):
   - Gateway `fc06::2` matches. `235` in `modified_set`.
   - `235` fully disabled → mark: `{232: true, 235: false}`.
   - Flat: `{fc08::2}`. APPDB written. `modified_set += 236`.
3. **260** (`2064:100::1d`, depends `[236]`):
   - `236` in `modified_set` → relevant!
   - `236` partially disabled → NOT marked disabled.
   - Flat: `236 → {fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 260`.
4. **263** (route 1::1, depends `[260, 264]`):
   - `260` in `modified_set` → relevant!
   - Flat: `260 → 236 → {fc08::2}`. `264 → 236 → {fc08::2}`. Dedup: `{fc08::2}`.
   - APPDB: `{fc08::2}`. `modified_set += 263`.
5. **264** (`2064:200::1e`, depends `[236]`):
   - `236` in `modified_set` → relevant!
   - Flat: `236 → {fc08::2}`. APPDB: `{fc08::2}`. `modified_set += 264`.
6. **263** (revisit, route 1::1, depends `[260, 264]`):
   - `264` now also in `modified_set` → relevant. Regen: same `{fc08::2}`. APPDB same (compare-and-skip).
7. **266** (via `3::3`, depends `[235]`):
   - `235` in `modified_set` → relevant!
   - `235` fully disabled → mark: `{235: false}`. Fully disabled. Skip APPDB.
   - `modified_set += 266`. Continue to `[269]`.
8. **269** (route 4::4, depends `[266, 270]`):
   - `266` in `modified_set` → relevant!
   - `266` fully disabled → mark: `{266: false, 270: true}`.
   - Flat: `270` (enabled) → `232 → {fc08::2}`. APPDB: `{fc08::2}`.
   - `modified_set += 269`.

- **Updated (APPDB):** `236, 260, 264, 263, 269`
- **Modified (skip APPDB):** `235, 266`
- **Not touched:** `232, 270`

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(235) | Leaf. Gateway fc06::2 matches nexthop. Self-ref: {235: false}. Single path -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[236, 266] | -- | {235} |
| 2 | back_walk(236) | ECMP. Depends [232, 235]. 235 fully disabled -> mark {232:true, 235:false}. Relevance: 235 in modified_set. Regen: {fc08::2}. | true | not pruned (modified) | yes -> dependents=[260, 264] | 236: {fc08::2} | {235, 236} |
| 3 | back_walk(260) | Depends [236]. 236 NOT fully disabled (partial). No gateway match (2064:100::1d != fc06::2). 236 in modified_set -> relevant. Regen via 236 -> {fc08::2}. | true | not pruned | yes -> dependents=[263] | 260: {fc08::2} | {235, 236, 260} |
| 4 | back_walk(263) | Depends [260, 264]. 260 NOT fully disabled (236 still enabled in 260). 260 in modified_set -> relevant. Regen: 260->{fc08::2}, 264->{fc08::2} via 236. | true | not pruned | yes -> dependents=[] | 263: {fc08::2} | {235, 236, 260, 263} |
| 5 | back_walk(264) | Depends [236]. 236 in modified_set -> relevant. Regen via 236 -> {fc08::2}. | true | not pruned | yes -> dependents=[263] | 264: {fc08::2} | {235, 236, 260, 263, 264} |
| 6 | back_walk(263) | **Revisit**. Depends [260, 264]. 264 now also in modified_set. Regen: same {fc08::2}. | true | not pruned | yes -> dependents=[] | 263: {fc08::2} (same) | {235, 236, 260, 263, 264} |
| 7 | back_walk(266) | Depends [235]. 235 fully disabled -> mark {235: false}. No gateway match (3::3 != fc06::2). 235 in modified_set -> relevant. Regen: no enabled depends -> single path disabled -> skip APPDB. | true | not pruned | yes -> dependents=[269] | -- | {235, 236, 260, 263, 264, 266} |
| 8 | back_walk(269) | Depends [266, 270]. 266 fully disabled -> mark {266:false, 270:true}. 266 in modified_set -> relevant. Regen: 270->{fc08::2} (via 232). | true | not pruned | yes -> dependents=[] | 269: {fc08::2} | {235, 236, 260, 263, 264, 266, 269} |


### Remote Failure 1 (`2064:100::1d` withdrawn)
**NHT:** `nexthop=2064:100::1d`, `resolved_nhg_id=260` (route-level NHG, gateway `2064:100::1d`)

FIB triggers `fib_nhg_back_walk(260, ctx)`.

1. **260** (STARTING, gateway `2064:100::1d`):
   - Gateway matches nexthop.
   - Disable all depends: `{236: false}`. Single depend, fully disabled -> skip APPDB.
   - `modified_set = {260}`. Continue to dependents `[263]`.
2. **263** (route 1::1, depends `[260, 264]`):
   - `260` in `modified_set` -> relevant!
   - `260` fully disabled -> mark: `{260: false, 264: true}`.
   - Regen: `264 -> 236 -> {fc06::2, fc08::2}`. APPDB: `{fc06::2, fc08::2}`.
   - `modified_set += 263`. Dependents=[] -> end.

- **Updated (APPDB):** `263`
- **Modified (skip APPDB):** `260`
- **Not reached:** `232, 235, 236, 264, 266, 270, 269`

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(260) | Gateway 2064:100::1d == nexthop -> **match**. Disable all depends: {236: false}. Single depend, fully disabled -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[263] | -- | {260} |
| 2 | back_walk(263) | Depends [260, 264]. 260 fully disabled -> mark {260:false, 264:true}. 260 in modified_set -> relevant. Regen: 264->{fc06::2, fc08::2} (via 236, all enabled). | true | not pruned | yes -> dependents=[] | 263: {fc06::2, fc08::2} | {260, 263} |


### Remote Failure 2 (`1::1` withdrawn)
**NHT:** `nexthop=1::1`, `resolved_nhg_id=263`

1. **263** (STARTING): No gateway match. `modified_set` empty.
   - Continue to dependents of 263 → NONE. Backwalk ends.

- **Nothing updated.** 

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(263) | ECMP. Depends [260, 264]. Both enabled -> no state change. No gateway match (composite NHG). No depends in modified_set (empty). | false | **ECMP + not matched -> don't prune** | dependents=[] -> end | -- | {} |


### Remote Failure 3 (`2::2` withdrawn)
**NHT:** `nexthop=2::2`, `resolved_nhg_id=270` (route-level NHG, gateway `2::2`)

FIB triggers `fib_nhg_back_walk(270, ctx)`.

1. **270** (STARTING, gateway `2::2`):
   - Gateway matches nexthop.
   - Disable all depends: `{232: false}`. Single depend, fully disabled -> skip APPDB.
   - `modified_set = {270}`. Continue to dependents `[269]`.
2. **269** (route 4::4, depends `[266, 270]`):
   - `270` in `modified_set` -> relevant!
   - `270` fully disabled -> mark: `{266: true, 270: false}`.
   - Regen: `266 -> 235 -> {fc06::2}`. APPDB: `{fc06::2}`.
   - `modified_set += 269`. Dependents=[] -> end.

- **Updated (APPDB):** `269: {fc06::2}`
- **Modified (skip APPDB):** `270`
- **Not reached:** `232, 235, 236, 260, 264, 263, 266`

**Call flow trace**:

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(270) | Gateway 2::2 == nexthop -> **match**. Disable all depends: {232: false}. Single depend, fully disabled -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[269] | -- | {270} |
| 2 | back_walk(269) | Depends [266, 270]. 270 fully disabled -> mark {266:true, 270:false}. 270 in modified_set -> relevant. Regen: 266->{fc06::2} (via 235). | true | not pruned | yes -> dependents=[] | 269: {fc06::2} | {270, 269} |


### Summary of corrected Topology 2 expectations:
| Test Case | APPDB Updated | State Modified (skip APPDB) | Not Reached |
|:---|:---|:---|:---|
| Local Failure (`fc06::2`) | `236, 260, 264, 263, 269` | `235, 266` | `232, 270` |
| Remote Failure 1 (`2064:100::1d`) | `263: {fc06::2, fc08::2}` | `260` | `232, 235, 236, 264, 266, 270, 269` |
| Remote Failure 2 (`1::1`) | `--` | `--` | all |
| Remote Failure 3 (`2::2`) | `269: {fc06::2}` | `270` | `232, 235, 236, 260, 264, 263, 266` |

## Test Topology 3 SRv6 VPN case
Assume we have 2064:100::1d/128 and 2064:200::1e/128 learnt via eBGP. Each route has two paths via fc06::2 and fc08::2

```
PE3# show ipv6 route 2064:100::1d next
Routing entry for 2064:100::1d/128
  Known via "bgp", distance 20, metric 0, best
  Last update 01:03:58 ago
  Nexthop Group ID: 237
  Received Nexthop Group ID: 235
  * fc06::2, via Ethernet12, weight 1
  * fc08::2, via Ethernet4, weight 1
```
The zebra's nexthop groups related to these routes are
```
PE3# show nexthop rib 234
ID: 234 (zebra)
     RefCnt: 19     Flags: 0x3
     Uptime: 01:04:23
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
     Interface Index: 143
           via fc08::2, Ethernet4 (vrf default), weight 1
     Dependents: (237)
PE3#
PE3# show nexthop rib 238
ID: 238 (zebra)
     RefCnt: 19     Flags: 0x3
     Uptime: 01:04:28
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
     Interface Index: 140
           via fc06::2, Ethernet12 (vrf default), weight 1
     Dependents: (237)
PE3# show nexthop rib 237
ID: 237 (zebra)
     RefCnt: 15     Flags: 0x3
     Uptime: 01:04:28
     VRF: default(No AFI)
     Nexthop Count: 2
     Valid, Installed
     Depends: (234) (238)
           via fc06::2, Ethernet12 (vrf default), weight 1
           via fc08::2, Ethernet4 (vrf default), weight 1
```

We have SRv6 VPN routes via   2064:100::1d and  2064:200::1e
```
PE3# show ip route vrf Vrf1 192.100.0.1 next
Routing entry for 192.100.0.1/32
  Known via "bgp", distance 20, metric 0, vrf Vrf1, best
  Last update 01:04:38 ago
  Nexthop Group ID: 242
  Received Nexthop Group ID: 239
   2064:100::1d(vrf default) (recursive), label 16, weight 1
     fc06::2, via Ethernet12(vrf default), label 16, weight 1
     fc08::2, via Ethernet4(vrf default), label 16, weight 1
   2064:200::1e(vrf default) (recursive), label 32, weight 1
     fc06::2, via Ethernet12(vrf default), label 32, weight 1
     fc08::2, via Ethernet4(vrf default), label 32, weight 1
```

The zebra's nexthop groups for received NHs are
```
PE3# show nexthop rib 240
ID: 240 (zebra)
     RefCnt: 15     Flags: 0x403
     Uptime: 01:14:11
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
           via 2064:100::1d (vrf default) inactive, label 16, seg6 fd00:201:201:1::, weight 1
     Dependents: (239)
PE3# show nexthop rib 241
ID: 241 (zebra)
     RefCnt: 15     Flags: 0x403
     Uptime: 01:05:12
     VRF: default(IPv6)
     Nexthop Count: 1
     Valid, Installed
           via 2064:200::1e (vrf default) inactive, label 32, seg6 fd00:202:202:2::, weight 1
     Dependents: (239)
PE3# show nexthop rib 239
ID: 239 (zebra)
     RefCnt: 10     Flags: 0x403
     Uptime: 01:05:00
     VRF: default(No AFI)
     Nexthop Count: 2
     Valid, Installed
     Depends: (240) (241)
           via 2064:100::1d (vrf default) inactive, label 16, seg6 fd00:201:201:1::, weight 1
           via 2064:200::1e (vrf default) inactive, label 32, seg6 fd00:202:202:2::, weight 1
```
These nexthop's NHGFUll Json objects are stored in test_topology_3.json. The node graph is shown as the following.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryTextColor': '#000000', 'secondaryTextColor': '#000000', 'tertiaryTextColor': '#000000', 'nodeTextColor': '#000000', 'lineColor': '#333333'}}}%%
graph TD
    %% ───────────── CLUSTER 1: SRv6 ECMP Group ─────────────
    subgraph SRv6_ECMP [SRv6 ECMP Group]
        N240["240 (2064:100::1d via SRv6)"]
        N241["241 (2064:200::1e via SRv6)"]
        N239["239 "]
    end

    %% ───────────── CLUSTER 2: Standard ECMP Group ─────────────
    subgraph Std_ECMP [Standard ECMP Group]
        N234["234 (fc08::2)"]
        N238["238 (fc06::2)"]
        N237["237 "]
    end

    %% ───────────── DEPENDENCY EDGES ─────────────
    %% Cluster 1 Edges
    N240 --> N239
    N241 --> N239

    %% Cluster 2 Edges
    N234 --> N237
    N238 --> N237

    %% ───────────── STYLING ─────────────
    classDef src fill:#a8d5a2,stroke:#2d5a2d,stroke-width:2px;
    classDef dst fill:#f0a8a8,stroke:#8b1c1c,stroke-width:2px;

    class N240,N241,N234,N238 src;
    class N239,N237 dst;
```

### Test local failure
* Scenario: The route fc06::2/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address fc06::2 and the resolved_nhg_id=210 (connected-route NHG for fc06::/120).
* Entry point resolution: `getEntry(210)` returns nullptr. Fallback: `getGlobalEntries("fc06::2")` returns {NHG 238} (leaf, gateway fc06::2).
* Expected Result:
  * Part 1 starting from node 238
    1. **238** (STARTING, leaf `fc06::2`):
        - Match. Self-ref disabled. Skip APPDB. `modified_set += 238`.
    2. **237** (ECMP, depends `[234, 238]`):
        - Gateway `fc06::2` matches. `238` in `modified_set`.
        - `238` fully disabled → mark: `{234: true, 238: false}`.
        - Flat: `{fc08::2}`. APPDB written. `modified_set += 237`.
  * Part 2 starting from fc06::2
    _ no match for fc06::2 in m_nexthop_to_vrf_RIBNHG

**Call flow trace**:

Part 1 (Global Table Context):

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(238) | Leaf. Gateway fc06::2 matches nexthop. Self-ref: {238: false}. Single path -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[237] | -- | {238} |
| 2 | back_walk(237) | ECMP. Depends [234, 238]. 238 fully disabled -> mark {234:true, 238:false}. 238 in modified_set -> relevant. Regen: {fc08::2}. | true | not pruned (modified) | yes -> dependents=[] | 237: {fc08::2} | {238, 237} |

Part 2 (VPN Context): Lookup fc06::2 in `m_nexthop_to_vrf_RIBNHG` -> **no match** (no VPN NHGs use fc06::2 as gateway). Done.

### Test remote failure
* Scenario: The route 2064:100::1d/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2064:100::1d and the corresponding NHG ID 237.
* Expected Result:
  * Part 1 starting from node 237
    - No update
  * Part 2 starting from nexthop 2064:100::1d
    - Find NHG node 240 from  m_nexthop_to_vrf_RIBNHG, trigger fib_nhg_back_walk from this node
    1. **240** (STARTING, leaf `2064:100::1d`):
        - Match. Self-ref disabled. Skip APPDB. `modified_set += 240`.
    2. **239** (ECMP, depends `[240, 241]`):
        -  `240` in `modified_set`.
        - `240` fully disabled → mark: `{241: true, 240: false}`.
        - Update SONiC NHG from node 239: `{2064:200::1e}`. APPDB written. `modified_set += 239`.  

**Call flow trace**:

Part 1 (Global Table Context):

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(237) | ECMP. Depends [234, 238]. Both enabled -> no state change. No gateway match (ECMP). No depends in modified_set (empty). | false | **ECMP + not matched -> don't prune** | dependents=[] -> end | -- | {} |

Part 1 result: No updates. 237 has no dependents in the Standard cluster.

Part 2 (VPN Context): Lookup 2064:100::1d in `m_nexthop_to_vrf_RIBNHG` -> **finds NHG 240** (gateway 2064:100::1d via SRv6).

Part 2 backwalk from 240 (using `walk_spec_sonic_nhg` / `prune_spec_sonic_nhg`):

| Step | back_walk call | walk_spec evaluation | walk_spec result | prune_spec | Continue? | APPDB | modified_node_set after |
|:-----|:---------------|:---------------------|:-----------------|:-----------|:----------|:------|:------------------------|
| 1 | back_walk(240) | Leaf. Gateway 2064:100::1d matches nexthop. Self-ref: {240: false}. Single path -> skip APPDB. | true | not pruned (modified) | yes -> dependents=[239] | -- | {240} |
| 2 | back_walk(239) | ECMP. Depends [240, 241]. 240 fully disabled -> mark {240:false, 241:true}. 240 in modified_set -> relevant. Regen SONiC NHG: {2064:200::1e}. | true | not pruned (modified) | yes -> dependents=[] | SONiC NHG: {2064:200::1e} | {240, 239} |


# 8. Test cases
## FRR topotest
TODO

## sonic-fib unit test
Generated via LLM

## sonic-swss unit test
The test cases are implemented via gtest in `tests/mock_tests/fpmsyncd/nhg_nht_ut.cpp`, using the `FpmSyncdNhtBackwalk` fixture class.

### Test Fixture: `FpmSyncdNhtBackwalk`
- Sets up `NHGMgr` with mock APPDB pipeline
- Provides `loadTopologyFromJson(filename)`: loads topology JSON, parses each entry via `fib::from_json_string()`, adds entries in dependency order via `addNHGFull()`
- Provides `resetAllEnableGroups(ids)`: resets all entries to enabled state
- Provides `runPart1Backwalk(nexthop, start_id)`: runs Part 1 backwalk (`fib_nhg_back_walk()` with `fib_nhg_walk_spec_for_node_quick_fixup` / `fib_nhg_prune_spec_for_node_quick_fixup`), returns the walking context for inspection of `visited_node_set`, `modified_node_set`, etc.
- Provides `runPart2Backwalk(nexthop, start_id)`: runs Part 2 (sonic_nhg) backwalk (`fib_nhg_back_walk()` with `fib_nhg_walk_spec_for_node_quick_fixup_sonic_nhg` / `fib_nhg_prune_spec_for_node_quick_fixup_sonic_nhg`), returns the walking context

### Test Cases

#### Backwalk Tests (using `runPart1Backwalk` / `runPart2Backwalk`)

These tests call `fib_nhg_back_walk()` directly via helpers to inspect walk context sets (`visited_node_set`, `modified_node_set`) and `m_resolved_enable_group` state.

| Test Name | Topology | Trigger | Key Assertions |
|:----------|:---------|:--------|:---------------|
| `Topology1_LocalFailure` | test_topology_1.json | `runPart1Backwalk("fc06::2", 237)` | visited={237,238,257,256,258,263,262,264}; 234 not visited; 237 self-disabled; 238: {234:true, 237:false} |
| `Topology1_RemoteFailure1` | test_topology_1.json | `runPart1Backwalk("2064:200::1e", 258)` | visited={258,256}; 258: {238:false}; 256: {258:false, 257:true}; 238 untouched |
| `Topology1_RemoteFailure2` | test_topology_1.json | `runPart1Backwalk("1::1", 263)` | visited={263,262}; 263: {238:false}; 262: {263:false, 264:true}; 256,238 untouched |
| `Topology2_LocalFailure` | test_topology_2.json | `runPart1Backwalk("fc06::2", 235)` | visited={235,236,260,263,264,266,269}; 232,270 not visited; 235 self-disabled; 236: {235:false, 232:true}; 266: {235:false}; 269: {266:false, 270:true}; 232 untouched |
| `Topology2_RemoteFailure1` | test_topology_2.json | `runPart1Backwalk("2064:100::1d", 260)` | visited={260,263}; 260: {236:false}; 263: {260:false, 264:true} |
| `Topology2_RemoteFailure2` | test_topology_2.json | `runPart1Backwalk("1::1", 263)` | visited={263}, modified={}; 263: enable_group untouched {260:true, 264:true} (no gateway match, no modified deps, ECMP not pruned) |
| `Topology2_RemoteFailure3` | test_topology_2.json | `runPart1Backwalk("2::2", 270)` | visited={270,269}; 270: {232:false}; 269: {270:false, 266:true} |
| `Topology3_LocalFailure` | test_topology_3.json | `runPart1Backwalk("fc06::2", 238)` | visited={238,237}; 238 self-disabled; 237: {238:false, 234:true} |
| `Topology3_RemoteFailure` | test_topology_3.json | Part 1: `runPart1Backwalk("2064:100::1d", 237)` then Part 2: `runPart2Backwalk("2064:100::1d", 240)` | Part 1: visited={237}, modified={} (ECMP, no gateway match); 237 untouched. Part 2: visited={240,239}; 240 self-disabled; 239: {240:false, 241:true}; updated_sonic_nhg_keys non-empty |

#### General Tests (using `fib_nhg_trigger_node_quick_fixup`)

These tests use the high-level `fib_nhg_trigger_node_quick_fixup()` API to verify end-to-end behavior.

| Test Name | Topology | Trigger | Key Assertions |
|:----------|:---------|:--------|:---------------|
| `ResolvedEnableGroupInit` | test_topology_1.json | (no trigger) | Leaf 234: size=1, {234:true}; Leaf 237: size=1, {237:true}; ECMP 238: size=2, {234:true, 237:true}; Composite 256: {257:true, 258:true} |
| `ResetEnableGroup` | test_topology_1.json | `fib_nhg_trigger_node_quick_fixup("fc06::2", 237)` then `resetResolvedEnableGroup()` | After trigger: {237:false}; after reset: {237:true} |
| `NonExistentResolvedNhgId` | test_topology_1.json | `fib_nhg_trigger_node_quick_fixup("fc06::2", 99999)` | No crash (graceful fallback to getGlobalEntries) |

#### `resolveLeafEnableFlags()` Tests

These tests verify the recursive tree walk that bridges depends-level `m_resolved_enable_group` to leaf-level flags for `getNextHopGroupFields()` filtering.

| Test Name | Topology | Trigger | Key Assertions |
|:----------|:---------|:--------|:---------------|
| `ResolveLeafEnableFlags` | test_topology_1.json | `fib_nhg_trigger_node_quick_fixup("fc06::2", 237)` | Leaf 234: {234:true}; Leaf 237: {237:false}; ECMP 238: {234:true, 237:false}; Intermediate 257 (depends=[238]): resolves to {234:true, 237:false} by recursing into 238; Composite 256 (depends=[257,258]): {234:true, 237:false} via union merge |
| `ResolveLeafEnableFlagsRemoteFailure` | test_topology_1.json | `fib_nhg_trigger_node_quick_fixup("2064:200::1e", 258)` | 256 (depends=[257,258], 258 disabled): all leaves reachable via enabled 257, result {234:true, 237:true}; 258 (depends=[238], all-disabled): skips 238 subtree, result {234:false, 237:false} |

### Main Test Flow
Two test patterns are used:

**Pattern 1 — Direct backwalk (topology-specific tests):**
1. Load topology JSON, convert to `fib::NextHopGroupFull` objects, add via `addNHGFull()` in dependency order
2. Call `runPart1Backwalk(nexthop, start_id)` (or `runPart2Backwalk()` for sonic_nhg phase)
3. Assert walk context: `visited_node_set`, `modified_node_set`
4. Assert `m_resolved_enable_group` state on affected entries

**Pattern 2 — High-level trigger (general and resolveLeafEnableFlags tests):**
1. Load topology JSON via `loadTopologyFromJson()`
2. Trigger `fib_nhg_trigger_node_quick_fixup(nexthop, resolved_nhg_id)` — runs both Part 1 and Part 2 internally
3. Assert `m_resolved_enable_group` state and/or `resolveLeafEnableFlags()` output on affected entries

## sonic-mgmt system level tests
In current sonic-mgmt, https://github.com/sonic-net/sonic-mgmt/blob/master/tests/srv6/test_srv6_basic_sanity.py provides a system level test with 7 nodes topology. The previous example's three Topologies are built from this 7 nodes topology. Tests are added to the existing `test_srv6_basic_sanity.py` file with helpers in `srv6_utils.py`.

### 7-Node Topology Key Connections

| Node | Interface | IP | Peer Node | Peer Interface | Peer IP | Role |
|------|-----------|-----|-----------|----------------|---------|------|
| PE3 (DUT) | Ethernet12 | fc06::1 | P4 | Ethernet12 | fc06::2 | Local path |
| PE3 (DUT) | Ethernet4 | fc08::1 | P2 | Ethernet12 | fc08::2 | Local path |
| PE1 | Ethernet0 | fc00::72 | P1 | Ethernet112 | fc00::71 | PE1 uplink |
| PE1 | Ethernet4 | fc02::1 | P3 | Ethernet4 | fc02::2 | PE1 uplink |
| PE2 | Ethernet0 | fc00::76 | P1 | Ethernet116 | fc00::75 | PE2 uplink |
| PE2 | Ethernet8 | fc03::1 | P3 | Ethernet8 | fc03::2 | PE2 uplink |

### Route Origins

| Prefix | Origin Node | Reaches PE3 via |
|--------|-------------|-----------------|
| 2064:100::1d/128 | PE1 | P1/P3 → P2(fc08::2) + P4(fc06::2) |
| 2064:200::1e/128 | PE2 | P1/P3 → P2(fc08::2) + P4(fc06::2) |

Both prefixes have 2-path ECMP at PE3: via fc06::2 (Ethernet12→P4) and fc08::2 (Ethernet4→P2).

### Helper Functions

All helper functions are added to `tests/srv6/srv6_utils.py`:

#### 1. apply_config_cmmds_to_vtysh(nbrhost, cmd_list)

```python
def apply_config_cmmds_to_vtysh(nbrhost, cmd_list):
    """Apply a list of vtysh configuration-mode commands to a device."""
    for input_cmd in cmd_list:
        cmd = "vtysh -c 'configure terminal' -c '{}'".format(input_cmd)
        nbrhost.command(cmd)
```

#### 2. Record Collection

```python
def start_record_collection(duthost, testcase_name):
    """Start tailing swss.rec and sairedis.rec on DUT."""
    for rec in ["swss.rec", "sairedis.rec"]:
        prefix = rec.replace(".rec", "")
        outfile = "/tmp/{}_{}.rec".format(prefix, testcase_name)
        duthost.command(
            "nohup tail -f /var/log/swss/{} > {} 2>&1 &".format(rec, outfile))

def stop_record_collection(duthost, testcase_name):
    """Stop record collection and copy files to ~/testlogs/."""
    duthost.command("pkill -f 'tail -f /var/log/swss'")
    duthost.command("mkdir -p ~/testlogs")
    for prefix in ["swss", "sairedis"]:
        src = "/tmp/{}_{}.rec".format(prefix, testcase_name)
        duthost.command("cp {} ~/testlogs/".format(src))
```

#### 3. APPDB Assertion Helpers

```python
def assert_appdb_nexthop_removed(duthost, nexthop, timeout=10, poll_interval=1):
    """Poll APPDB until nexthop is absent from ALL NHG entries' nexthop field."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        result = duthost.command("redis-cli -n 0 KEYS 'NEXTHOP_GROUP_TABLE:*'")
        keys = parse_appdb_keys(result)
        found = False
        for key in keys:
            nh_result = duthost.command("redis-cli -n 0 HGET '{}' nexthop".format(key))
            nh_value = nh_result.get('stdout', '')
            if nexthop in nh_value:
                found = True
                break
        if not found:
            return  # success
        time.sleep(poll_interval)
    pytest_assert(False, "Nexthop '{}' still present in APPDB after {}s".format(nexthop, timeout))

def assert_appdb_nexthop_present(duthost, nexthop):
    """Assert nexthop exists in at least one NHG entry."""
    result = duthost.command("redis-cli -n 0 KEYS 'NEXTHOP_GROUP_TABLE:*'")
    keys = parse_appdb_keys(result)
    for key in keys:
        nh_result = duthost.command("redis-cli -n 0 HGET '{}' nexthop".format(key))
        nh_value = nh_result.get('stdout', '')
        if nexthop in nh_value:
            return  # found
    pytest_assert(False, "Nexthop '{}' not found in any APPDB NHG entry".format(nexthop))
```

### Failure Triggers

| Test Case | Mechanism | Nodes Affected | Expected NHT Event |
|-----------|-----------|----------------|-------------------|
| **Local failure** | PE3: `sudo ifconfig Ethernet12 down` | PE3 link-down | fc06::2 unreachable, `current_resolved_nhg_id=0` |
| **Remote failure** | P1: `sudo ifconfig Ethernet112 down` + P3: `sudo ifconfig Ethernet4 down` | PE1 isolated | 2064:100::1d withdrawn, `current_resolved_nhg_id=0` |

**Why this remote failure mechanism works**: PE1 has only two uplinks: Ethernet0→P1 and Ethernet4→P3. Shutting down the peer-side interfaces (P1:Ethernet112 + P3:Ethernet4) causes immediate link-down detection on PE1. BGP sessions drop within seconds (link-level, no 180s hold timer wait). PE1's originated route (2064:100::1d) is withdrawn from the network. No BFD is available in this topology.

### Recovery

| Failure | Recovery Command | Wait Time |
|---------|-----------------|-----------|
| Local | PE3: `sudo ifconfig Ethernet12 up` | 10s |
| Remote | P1: `sudo ifconfig Ethernet112 up` + P3: `sudo ifconfig Ethernet4 up` | 15s |

### Assertions

| Test Case | Assert ABSENT from nexthop field | Assert PRESENT in nexthop field | Timeout |
|-----------|--------------------------------|-------------------------------|---------|
| T1 Local (fc06::2 down) | `fc06::2` | `fc08::2` | 10s |
| T1 Remote (PE1 isolated) | `2064:100::1d` | `2064:200::1e` | 30s |
| T2 Local (fc06::2 down) | `fc06::2` | `fc08::2` | 10s |
| T2 Remote (PE1 isolated) | `2064:100::1d` | `2064:200::1e` | 30s |

### Test Cases

We create 4 independent test cases for Topology 1 and Topology 2. Each test:
- Sets up its own topology (static routes via vtysh)
- Verifies initial state before triggering failure
- Asserts APPDB convergence after failure
- Cleans up (recover + remove routes)

#### Topology 1 Static Routes (applied on PE3)
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 2064:200::1e
ipv6 route 3::3/128 1::1
ipv6 route 3::3/128 2::2
ipv6 route 4::4/128 1::1
```

#### Topology 2 Static Routes (applied on PE3)
```
ipv6 route 1::1/128 2064:100::1d
ipv6 route 1::1/128 2064:200::1e
ipv6 route 2::2/128 fc06::2
ipv6 route 3::3/128 fc08::2
ipv6 route 4::4/128 2::2
ipv6 route 4::4/128 3::3
```

#### Test workflow per test case

1. Apply static routes on PE3 via `apply_config_cmmds_to_vtysh()`
2. Wait 5s for NHG convergence
3. Verify initial state: both nexthops present in APPDB
4. Start record collection
5. Trigger failure (local: `ifconfig Ethernet12 down` on PE3; remote: `ifconfig Ethernet112 down` on P1 + `ifconfig Ethernet4 down` on P3)
6. Assert APPDB convergence (poll with timeout)
7. Stop record collection
8. Verify record patterns (NEXTHOP_GROUP_TABLE operations present)
9. Recover (bring interfaces back up), wait for reconvergence
10. Teardown: remove static routes via `no ipv6 route ...`

# 9. Appendix
This section documents all the issues met when using LLM to brainstorm, code generation, compile and testing. The main skills are similar to skills listed in https://github.com/obra/superpowers/tree/main/skills and https://github.com/numman-ali/openskills. At high level, we use brainstorm skill to go over LLD in detail and use writing-plan skill to generate codes after brainstorming.

## Key refinements identified during LLM-assisted brainstorming:
* **Intermediate Verification**: The LLM can emit step-by-step intermediate states to validate the correctness of recursive call outcomes.
* **Idempotency Clarification**: The LLM flagged a potential idempotency concern, prompting us to explicitly scope Phase 1 support to RNH transitions only into the unresolved state.
* **Trace-Driven Validation**: The LLM generates structured call flow trace artifacts and can cross-validate them against the step-by-step intermediate results to ensure behavioral consistency.
* **topology json error**: Flagged topology json error with missing nexthop type and gate in topology 2's json after LLM comparing LLD with json file file.

## First Round Code Genearted
| Repo | link | Comments |Unit Test cases |
|:-----|:---------------|:---------------------|:-----------------|
| sonic-frr | https://github.com/eddieruan-alibaba/sonic-frr/tree/eruan-ai | AI codes with fixing `Cached bogus pointer` | No, need to add topotest |
| sonic-buildimage | https://github.com/eddieruan-alibaba/sonic-buildimage/tree/eruan-ai | AI codes with fixing header including path | n/a |
| sonic-fib | https://github.com/eddieruan-alibaba/sonic-fib/tree/eruan-ai | AI codes with fixing header including path | Yes, included |
| sonic-swss | https://github.com/eddieruan-alibaba/sonic-swss/tree/eruan-ai2 | AI codes with C++ 14 fix and `Misunderstand m_nexthop`'s fix | Yes, included |
| sonic-mgmt | https://code.alibaba-inc.com/SONiC/sonic-mgmt/tree/eruan-ai/ | AI codes | [sonic-mgmt system level tests](#sonic-mgmt-system-level-tests) |
## Couple issues fixed for compiling
* Language Standard, C++14 vs C++17, See the [Language Standard](#1-language-standard) section
* Header Include Paths: Export vs. Internal Files, See the [Include Path Rules](#2-header-include-paths-export-vs-internal-files) section

## Unit Test issues:
###  Cached bogus pointer
In commit https://github.com/eddieruan-alibaba/sonic-frr/commit/53b7503dec00d3b3889e57c061b22de6b46a793d#diff-4bc931e0979fe77074f30df99e44b72472dc4b143a84a2929933d017bde5ce0d, AI generated codes the caches a pointer to a `struct route_entry`.
```
struct route_entry *old_re = rnh->state;
```

However, this pointer is invalidated during the execution of `copy_state()`, which explicitly frees the existing state:
```
static void copy_state(struct rnh *rnh, const struct route_entry *re,
		       struct route_node *rn)
{
	struct route_entry *state;

	if (rnh->state) {
		free_state(rnh->vrf_id, rnh->state, rn);
		rnh->state = NULL;
	}

	if (!re)
		return;

```

Accessing the cached pointer after copy_state() results in undefined behavior. In practice, this causes incorrect prev_nhg_id values in Zebra logs:

```
2026/05/05 17:45:52 ZEBRA: [JZW23-YKRTG] NHT event: rnh=2064:100::1d/128 prev_pfx=2064:100::1d/128 prev_nhg=463787600 curr_pfx=::/0 curr_nhg=0
```

**Resolution**:
Update the design specification to explicitly forbid caching route_entry pointers across state-copy operations. Instead, cache primitive values (e.g., nhg_id) which remain valid regardless of memory reallocation.

**Process Reflection**:
We need to analyze why this memory lifecycle detail was missed during the code generation. If LLM-assisted development requires the spec to explicitly describe every low-level memory management constraint, it implies that spec authors must possess deep implementation-level knowledge. We need to determine if this level of granularity is sustainable for future developing.

###  Misunderstand m_nexthop
LLM misunderstood m_nexthop's meaning and created the following API for getting NHG's gateway address 
```
string RIBNHGEntry::getGatewayAddress() {
    return m_nexthop;
}
```

Then use this api in the following walk spec, which is not correct.
```
    /* Step 2: Relevance check — gateway match */
    string gateway = entry->getGatewayAddress();
    if (!gateway.empty() && gateway == ctx.nexthop_address) {
        if (depends.empty()) {
            /* Leaf NHG: mark self-reference disabled, skip APPDB write */
            enable_group[entry_id] = false;
            ctx.modified_node_set.insert(entry_id);
            SWSS_LOG_NOTICE("walk_spec: leaf node %d gateway match, disabled self", entry_id);
            return true;
        } else {
            /* Non-leaf with gateway match: mark ALL depends disabled */
            for (auto& kv : enable_group) {
                kv.second = false;
            }
            is_relevant = true;
        }
    }
```
**Resolution**:
The fix is to update this LLD spec to add `m_gateway` field explictly. 

###  Not handle IBNHGEntry::getNextHopGroupFields()'s change paths based on m_resolved_enable_group
```
Method Update: RIBNHGEntry::getNextHopGroupFields()
Before recursing into each depends entry in m_resolvedGroup, check m_resolved_enable_group: if the depends entry is marked false (disabled), skip it. This produces the correct flat resolved paths reflecting the current enable/disable state at every level of the dependency chain.
```

**Root Cause**: `m_resolved_enable_group` tracks depends-level IDs (e.g., {238: true}), while `m_resolvedGroup` contains leaf-level IDs (e.g., {234, 237}). A direct lookup of leaf ID 237 in `m_resolved_enable_group` misses because 237 is not a key there — only the depends ID 238 is.

**Resolution**: Implemented `resolveLeafEnableFlags()` which recursively walks the depends tree from the current node down to leaves, respecting enable/disable gates at each level. `getNextHopGroupFields()` now calls this method once before iterating `m_resolvedGroup`, and uses the returned leaf-level flags for filtering. See [Method Update: `RIBNHGEntry::getNextHopGroupFields()`](#method-update-ribnhgentrygetnexthopgroupfields) for full details.