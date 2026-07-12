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
- Add the constructor `dplane_nht_event_update(struct rnh *rnh, const struct prefix *prev_resolved_prefix, uint32_t prev_resolved_nhg_id)` that allocates a ctx and enqueues it.
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

## 4A. Second trigger point: local link down (directly-connected nexthop)

### 4A.1 The gap

Section 4 covers only nexthops that carry an **RNH (Nexthop Tracker)** registration — i.e. *recursive* nexthops learned via BGP/IGP. A **directly-connected** nexthop (e.g. `fc06::2` reachable over `Ethernet12` as a connected `/120`) has **no RNH**. When its egress interface goes down, `zebra_rnh_eval_nexthop_entry()` is never invoked for it (zebra logs `"<pfx> has no tracking NHTs. Bailing"`), so **no NHT event is emitted** and fpmsyncd's backwalk never fires.

Observed symptom (system test `test_topology1_local_failure` / `test_topology2_local_failure`): after `ifconfig Ethernet12 down`, the dead nexthop `fc06::2` is **not** pruned from any global `NEXTHOP_GROUP_TABLE` entry; `swss.rec` shows zero `NEXTHOP_GROUP_TABLE:N SET` ops; only the full-resync slow path runs.

### 4A.2 Where the interface-down path already lives

`if_down()` (`zebra/interface.c`) already walks the per-interface NHG list on link down:

```
if_down(ifp)
  → if_down_nhg_dependents(ifp)                 // interface.c
      frr_each(nhg_connected_tree, &zif->nhg_dependents, rb_node_dep)
          zebra_nhg_check_valid(rb_node_dep->nhe);   // marks NHG invalid only
```

`zif->nhg_dependents` is exactly the set of NHGs whose egress interface is `ifp`; the directly-connected singleton NHGs for that link are in it. This is the natural, already-iterating trigger point — it runs synchronously before the async RIB re-evaluation.

### 4A.3 Change

Add a second emit entry point that does **not** require a `struct rnh`, and call it from `if_down_nhg_dependents()` for each directly-connected singleton nhe:

- New constructor `dplane_nht_event_update_connected(const struct nexthop *nh, uint32_t prev_resolved_nhg_id)` in `zebra/zebra_dplane.c`:
  - Gated by `zebra_nhg_fib_enabled` (same knob as §6); returns immediately when off or when `nh == NULL`.
  - Builds a host prefix (`/32` or `/128`) from `nh->gate` as `rnh_prefix` (this is the bare nexthop address fpmsyncd strips and uses as its lookup key).
  - Sets `prev_resolved_nhg_id = nhe->id` (the singleton's own zebra NHG id — the start point for fpmsyncd `backwalkPicCore`).
  - Sets `curr_resolved_nhg_id = 0` and a zeroed `curr_resolved_prefix` (nexthop is now unreachable → satisfies the Phase 1 gate).
  - Leaves `prev_resolved_prefix` zeroed (unused by the connected-nexthop backwalk).
- In `if_down_nhg_dependents()`, for each `rb_node_dep->nhe` that is a singleton (`ZEBRA_NHG_IS_SINGLETON`) with an IPv4/IPv6 gate, call the new constructor with `nhe->id` **before** `zebra_nhg_check_valid()` clears its flags.

### 4A.4 Why fpmsyncd needs no change

fpmsyncd already stores every received NHG — including directly-connected singletons — in its `RIBNHGEntry` map (`addNHGFull` → `m_nhg_map.insert`). Therefore `backwalkPicCore(prev_resolved_nhg_id, failedNh)` resolves the singleton start entry, detects `isDirectNexthop`, disables it, and walks its dependents to rewrite the affected **global** composite NHGs in APPL_DB. The event supplies `prev_resolved_nhg_id = singleton nhe->id`, so the existing PIC-core path handles it end-to-end; the VRF-only `backwalkPicEdge` fallback is not relied upon here.

### 4A.5 Field contract for the connected-nexthop event

| Field | Value |
|-------|-------|
| `rnh_prefix` | host prefix of `nh->gate` (e.g. `fc06::2/128`) — fpmsyncd strips `/len` → `fc06::2` |
| `prev_resolved_prefix` | zeroed (unused) |
| `prev_resolved_nhg_id` | `nhe->id` of the directly-connected singleton (start point for `backwalkPicCore`) |
| `curr_resolved_prefix` | zeroed |
| `curr_resolved_nhg_id` | `0` (unreachable — passes Phase 1 gate) |

---

## 5. Error handling and corner cases

- `dplane_nht_event_update()` returns immediately when `rnh == NULL`; nothing is enqueued.
- `dplane_nht_event_update_connected()` returns immediately when `nh == NULL`, when `zebra_nhg_fib_enabled` is off, or when the nexthop gate is neither IPv4 nor IPv6 (defensive); nothing is enqueued.
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

- `zebra/zebra_dplane.h` — new enum + `struct dplane_nht_info` + accessor declarations + `dplane_nht_event_update_connected()` declaration.
- `zebra/zebra_dplane.c` — `dplane_nht_event_update()` + `dplane_nht_event_update_connected()` (connected-nexthop trigger, §4A) + ctx handling + `dplane_op2str()` + switch-enum cases.
- `zebra/zebra_rnh.c` — (1) add out parameter `route_entry_queued` to `zebra_rnh_resolve_nexthop_entry()`; (2) extend `zebra_rnh_eval_nexthop_entry()` signature and insert the trigger (`state_changed && !route_entry_queued`); (3) `zebra_rnh_clear_nhc_flag()` passes NULL for the new argument.
- `zebra/interface.c` — in `if_down_nhg_dependents()`, emit `dplane_nht_event_update_connected()` for each directly-connected singleton nhe before `zebra_nhg_check_valid()` (§4A).

---

## 8. Compatibility

- New enum and accessors are backward compatible.
- When `--nhg-fib` is disabled, zebra does not emit NHT events (status quo preserved).
- An older FPM provider that does not implement `DPLANE_OP_NHT_EVENT_UPDATE` → the event is dropped in the dplane queue (log warning); other dplane ops are unaffected.

---

## 9. Testing

FRR-side tests (topotest for NHT trigger, dplane ctx population, FPM message format, and QUEUED-transient suppression) are described in the consolidated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md) §topotest.
