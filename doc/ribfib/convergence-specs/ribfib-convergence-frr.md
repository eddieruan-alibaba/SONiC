# RIB/FIB Convergence — sonic-frr sub-spec

- **Repository**: `sonic-frr`
- **Branch**: `ribfib_2_yuqing`
- **Project design**: [`ribfib-convergence-overview.md`](ribfib-convergence-overview.md)
- **Master spec**: [`ribfib-convergence-design.md`](ribfib-convergence-design.md)
- **Tests**: [`ribfib-convergence-test.md`](ribfib-convergence-test.md)
- **Status**: designing
- **Author**: Yuqing Zhao
- **Date**: 2026-07-05

---

## 1. Responsibility of this repository

Generate an NHT event when a Zebra RNH (Nexthop Tracker) state change settles, and hand it to the dplane so fpmsyncd can perform fast fixup. This spec covers **only the generation side**; message assembly and serialization happen in the SONiC-specific FPM provider (see the dplane-encoding sub-spec).

**Scope:**
- `zebra/zebra_dplane.h` / `zebra/zebra_dplane.c` — new dplane op enum + ctx fields + accessors + constructor.
- `zebra/zebra_rnh.c` — a new out parameter on the resolve side; the trigger point on the eval side.

**Not in scope:** FPM message assembly (belongs to `sonic-buildimage` — see dplane-encoding sub-spec), fpmsyncd parsing and backwalk (belongs to `sonic-swss`), all tests (see test sub-spec).

---

## 2. dplane enum extension

Add a new value to `enum dplane_op_e` in `zebra/zebra_dplane.h`:

```c
DPLANE_OP_NHT_EVENT_UPDATE,
```

In `zebra/zebra_dplane.c`:
- Add a `dplane_op2str()` branch for the new op.
- Add the constructor `dplane_nht_event_update(const struct prefix *rnh_prefix, const struct prefix *prev_resolved_prefix, uint32_t prev_resolved_nhg_id, const struct prefix *curr_resolved_prefix, uint32_t curr_resolved_nhg_id)` that allocates a ctx and enqueues it.
- The constructor takes **5 independent fields** (not a `struct rnh *`) so the dplane layer does not depend on RNH internals. This allows both recursive nexthop changes and local-link-down events to share the same constructor — the caller simply passes the appropriate prefix/nhg_id values regardless of how they were obtained.
- **Dispatch**: the ctx is enqueued via `dplane_update_enqueue()` — the standard dplane provider chain entry — so the event traverses all providers including the FPM provider. Using `dplane_provider_enqueue_to_zebra()` would bypass the FPM provider entirely.
- **skip_kernel**: the ctx is marked `dplane_ctx_set_skip_kernel()` before enqueueing, because NHT events are informational notifications for the FPM and must not be processed by the kernel provider.
- Add the corresponding `case` in every `-Wswitch-enum` switch over `enum dplane_op_e` inside `zebra_dplane.c` (3 spots).

> **Note:** do **not** touch `dplane_fpm_nl.c` — in this build it is replaced at runtime by the SONiC-specific `dplane_fpm_sonic.c` provider.

---

## 3. NHT fields carried on the dplane ctx

Add a new sub-struct `struct dplane_nht_info` (a member inside `zebra_dplane_ctx`):

```c
struct dplane_nht_info {
    struct prefix rnh_prefix;
    struct prefix prev_resolved_prefix;
    uint32_t      prev_resolved_nhg_id;
    struct prefix curr_resolved_prefix;
    uint32_t      curr_resolved_nhg_id;
};
```

Provide accessors used by the FPM provider to assemble the message:

```c
const struct prefix *dplane_ctx_get_nht_rnh_prefix(const struct zebra_dplane_ctx *ctx);
const struct prefix *dplane_ctx_get_nht_prev_resolved_prefix(const struct zebra_dplane_ctx *ctx);
uint32_t             dplane_ctx_get_nht_prev_resolved_nhg_id(const struct zebra_dplane_ctx *ctx);
const struct prefix *dplane_ctx_get_nht_curr_resolved_prefix(const struct zebra_dplane_ctx *ctx);
uint32_t             dplane_ctx_get_nht_curr_resolved_nhg_id(const struct zebra_dplane_ctx *ctx);
```

The FPM provider (`sonic-buildimage/src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c`) consumes these accessors.

---

## 4. Trigger point: `zebra_rnh_eval_nexthop_entry()`

### 4.1 Background: two causes of `re == NULL`

`zebra_rnh_resolve_nexthop_entry()` (in `zebra/zebra_rnh.c`) iterates candidate route entries; a candidate with `ROUTE_ENTRY_QUEUED && !ROUTE_ENTRY_INSTALLED` is skipped via `continue`. The caller therefore receives `re == NULL` in two very different situations:

1. **The route was actually withdrawn** → the nexthop became unreachable (`curr_resolved_nhg_id == 0`, **the very case Phase 1 must catch**).
2. **All candidates are blocked by QUEUED** → transient state; no NHT event should be emitted yet.

Using `re && ...` as the trigger condition would suppress both cases, blocking the primary use case. The resolve side must expose an extra out parameter so the caller can tell them apart.

### 4.2 Resolve-side change

Extend the signature of `zebra_rnh_resolve_nexthop_entry()` with an out parameter:

```c
static struct route_entry *
zebra_rnh_resolve_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
                                struct route_node *nrn, const struct rnh *rnh,
                                struct route_node **prn,
                                bool *route_entry_queued);
```

In the existing `continue` branch that skips a QUEUED candidate, set `*route_entry_queued = true` (only if the pointer is non-NULL). On return:
- If a usable `re` is found → keep `*route_entry_queued` as-is (it may be false, or it may have been set true earlier; the latter does not affect trigger correctness because `state_changed` still gates the send).
- If NULL is returned → `*route_entry_queued` reflects whether candidates were skipped due to QUEUED.

`zebra_rnh_clear_nhc_flag()` also calls resolve; pass `NULL` for the 6th argument there (it does not care about the queued reason).

### 4.3 Eval-side trigger condition

```c
static void zebra_rnh_eval_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
                                         int force, struct route_node *nrn,
                                         struct rnh *rnh,
                                         struct route_node *prn,
                                         struct route_entry *re,
                                         bool route_entry_queued)
{
    int state_changed = 0;
    /* Cache prev state BEFORE copy_state(). */
    struct prefix prev_resolved_route = rnh->resolved_route;
    uint32_t prev_resolved_nhg_id =
        (rnh->state && rnh->state->nhe) ? rnh->state->nhe->id : 0;

    /* ... existing resolve / copy_state / state_changed logic ... */

    /* Trigger NHT event (new). */
    if (state_changed && !route_entry_queued) {
        dplane_nht_event_update(rnh, &prev_resolved_route,
                                prev_resolved_nhg_id);
    }

    /* ... existing zebra_rnh_notify_protocol_clients etc. ... */
}
```

**Caller relationship:** the upper layer `zebra_rnh_evaluate_entry()` first calls resolve to obtain both `re` and `route_entry_queued`, then passes both into the eval function; the eval signature is extended accordingly.

**Implementation notes:**
- **Before** `copy_state()`, cache `rnh->resolved_route` and `rnh->state->nhe->id` into locals (the `prev_*` values). Note there is no `rnh->resolved_nhg_id` field; the resolved NHG ID lives at `rnh->state->nhe->id`.
- **After** `copy_state()`, the fields on `rnh` hold the new (`curr_*`) values.

### 4.4 Behavior table

| Scenario | re | route_entry_queued | Emit NHT event? |
|---|---|---|---|
| Route withdrawn, nexthop unreachable | NULL | false | ✓ |
| Candidates blocked by QUEUED (transient) | NULL | true | ✗ |
| Resolution moved (IGP switch to a new NHG) | new RE | false | ✓ |
| No change | any | any | ✗ (`state_changed == false`) |

### 4.5 The three NHT scenarios: prev/curr combinations

| Scenario | prev_resolved_nhg_id | curr_resolved_nhg_id |
|----------|----------------------|----------------------|
| Nexthop completely unreachable (route deleted) | 243 (old) | 0 |
| Nexthop resolution moved (IGP reconvergence, new NHG) | 243 (old) | 244 (new) |
| Nexthop came back up | 0 | 244 (new) |

**Phase 1 gate:** FRR always emits; fpmsyncd only processes the `curr == 0` case (all others are returned early by the routesync dispatch layer).

---

## 4A. Second trigger point: local link down (directly-connected member)

### 4A.1 The gap

Section 4 covers only nexthops that carry an **RNH (Nexthop Tracker)** registration — i.e. *recursive* nexthops learned via BGP/IGP. A **directly-connected** member of an ECMP group (e.g. `fc06::2` reachable over `Ethernet12` as a connected `/120`) has **no RNH**. When its egress interface goes down, `zebra_rnh_eval_nexthop_entry()` is never invoked for it, so the §4 trigger does not apply.

Observed symptom (system tests `test_topology1_local_failure` / `test_topology2_local_failure`): after `ifconfig Ethernet12 down`, the dead member `fc06::2` is **not** pruned from the affected `NEXTHOP_GROUP_TABLE` entries in APPL_DB; the composite group and the groups stacked on top of it keep advertising the stale member to the FPM.

### 4A.2 What already happens on link down

`if_down()` (`zebra/interface.c`) already walks the per-interface NHG list on link down:

```
if_down(ifp)
  → if_down_nhg_dependents(ifp)                 // interface.c
      frr_each(nhg_connected_tree, &zif->nhg_dependents, rb_node_dep)
          zebra_nhg_check_valid(rb_node_dep->nhe);
  → rib_update_handle_vrf_all(RIB_UPDATE_INTERFACE_DOWN, ...)   // RIB re-eval
```

`zebra_nhg_check_valid()` → `zebra_nhg_set_valid()` walks the dependents of the failed singleton and, using a **weight-insensitive** nexthop comparison (`nexthop_same_no_weight`), clears `NEXTHOP_FLAG_ACTIVE` on the matching member inside each composite group. (The weight-insensitive compare matters: a directly-connected nexthop has `weight = 1` on its own but is stored with a normalized weight once it becomes a group member, so an exact compare would miss it.)

So the member is correctly marked inactive, and the subsequent RIB re-evaluation re-visits the affected composite groups through `zebra_nhg_install_kernel()`.

### 4A.3 The missing piece

When only one member of an ECMP composite goes inactive, the composite **stays valid** (it still has another active member) and therefore stays `INSTALLED` / `INSTALLED_FPM_ONLY`. The install condition in `zebra_nhg_install_kernel()` only re-sends a group when it is not yet installed **or** carries an explicit reinstall marker:

```
VALID && ( !(INSTALLED || INSTALLED_FPM_ONLY)
           || REINSTALL || REINSTALL_FPM_ONLY ) && !QUEUED
```

An already-installed, still-valid composite with no reinstall marker fails this test, so nothing is re-sent to the FPM and the stale member persists. The dataplane serializer skips per-member `NEXTHOP_FLAG_ACTIVE` on purpose (received groups must keep inactive members represented), so the pruning has to be expressed by **re-sending the group** with the inactive member excluded from its resolved set — not by editing the member in place.

### 4A.4 Change: flag the affected closure for FPM-only reinstall

Add a recursive helper `zebra_nhg_flag_reinstall_fpm()` in `zebra/zebra_nhg.c` and call it from **`if_down_nhg_dependents()`** (`zebra/interface.c`) — the interface-down entry point. After `zebra_nhg_check_valid()` marks each directly-connected singleton invalid, the helper is called on each composite NHG in the singleton's `nhg_dependents` tree:

- The helper sets `NEXTHOP_GROUP_REINSTALL_FPM_ONLY` on the composite and recurses into `nhe->nhg_dependents`, so **every group stacked on top of the composite** (e.g. VPN / recursive parents that flatten the same member) is flagged too. This is required because each group materializes its own flattened member list; re-sending only the base composite would leave the parents stale.
- It only flags groups that are gated by `zebra_nhg_fib_enabled` and were already pushed to the dataplane (`INSTALLED` or `INSTALLED_FPM_ONLY`).
- The flag itself doubles as a visited-guard, so the dependents DAG is walked once per node.

**Why `if_down_nhg_dependents` and not `zebra_nhg_set_valid`:** `set_valid` is a generic primitive invoked during NHG deletion, VRF teardown, and shutdown — placing the reinstall trigger there risks spurious FPM updates in unrelated teardown paths. By placing it in `if_down_nhg_dependents` the trigger is explicitly scoped to interface failure events only.

The RIB re-evaluation that `if_down()` already triggers then re-visits these groups; with `REINSTALL_FPM_ONLY` set the install condition fires, the group is re-serialized (the inactive member is left out of the resolved set), and the update is sent **FPM-only** — the kernel already reflects the interface going down, so kernel programming is skipped.

### 4A.5 Flag lifecycle

`NEXTHOP_GROUP_REINSTALL_FPM_ONLY` is consumed and cleared by the existing dplane path when the group is enqueued for install (it sets skip-kernel and unsets the flag). If the group was mid-install (`QUEUED`) when flagged, the post-install callback re-triggers one more install so the flag is honored and then cleared. No new clearing logic is introduced.

### 4A.6 Why fpmsyncd needs no change

Each affected group is re-sent through the normal NHG update path. fpmsyncd processes it as an ordinary group update and rewrites the corresponding `NEXTHOP_GROUP_TABLE` entry in APPL_DB with the pruned member list. No new message type, event, or consumer logic is added on the fpmsyncd side.

---

## 5. Error handling and corner cases

- `dplane_nht_event_update()` returns immediately when `rnh_prefix == NULL`; nothing is enqueued.
- `zebra_nhg_flag_reinstall_fpm()` (§4A) is a no-op when `--nhg-fib` is off, when the group was not already pushed to the dataplane, or when it is already flagged (visited-guard); it never edits members in place and never touches kernel state.
- ctx allocation failure → log an error, drop the event, do not block the zebra main loop.
- FPM socket not connected → reuse the existing dplane pending queue; the NHT event shares the same queue (no new queue).
- If the caller passes NULL for `route_entry_queued`, resolve skips the assignment and behaves like the original logic (defensive).
- Preserve NHG ID needs **no** special handling — Zebra's own NHG hash already guarantees determinism.

---

## 6. `--nhg-fib` and other knob dependencies

- `--nhg-fib` already exists as the RIB/FIB global switch; when it is off, zebra emits no NHG-related dplane event (including this NHT event).
- **This design adds no new knob.**

---

## 7. Files touched

- `zebra/zebra_dplane.h` — new enum + `struct dplane_nht_info` + accessor declarations; constructor signature takes 5 independent fields (§2).
- `zebra/zebra_dplane.c` — `dplane_nht_event_update()` constructor (5-field signature, `skip_kernel`, `dplane_update_enqueue`) + ctx handling + `dplane_op2str()` + switch-enum cases (§2, §4).
- `zebra/zebra_rnh.c` — (1) add out parameter `route_entry_queued` to `zebra_rnh_resolve_nexthop_entry()`; (2) extend `zebra_rnh_eval_nexthop_entry()` signature and insert the trigger (`state_changed && !route_entry_queued`), passing 5 independent fields extracted from rnh; (3) `zebra_rnh_clear_nhc_flag()` passes NULL for the new argument (§4).
- `zebra/zebra_nhg.c` — add `zebra_nhg_flag_reinstall_fpm()` helper (§4A).
- `zebra/zebra_nhg.h` — declare `zebra_nhg_flag_reinstall_fpm()`.
- `zebra/interface.c` — in `if_down_nhg_dependents()`, after `zebra_nhg_check_valid()` marks the singleton invalid, call `zebra_nhg_flag_reinstall_fpm()` on each composite in the singleton's dependents tree (§4A).

---

## 8. Compatibility

- New enum and accessors are backward compatible.
- When `--nhg-fib` is disabled, zebra does not emit NHT events (status quo preserved).
- An older FPM provider that does not implement `DPLANE_OP_NHT_EVENT_UPDATE` → the event is dropped in the dplane queue (log warning); other dplane ops are unaffected.

---

## 9. Testing

FRR-side tests (topotest for NHT trigger, dplane ctx population, FPM message format, and QUEUED-transient suppression) are described in the consolidated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md) §topotest.
