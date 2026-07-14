# Project Design: RIB/FIB Convergence
> **AI-facing SDD artifact** — This document is written to be consumed by an AI agent to generate code. It is a Spec-Driven Development process artifact, not human-oriented documentation.

A route-convergence acceleration project on top of the RIB/FIB layered architecture (Zebra = RIB, fpmsyncd = FIB). Phase 1 landed the layering infrastructure; Phase 2 (this project, `ribfib-convergence`) implements an **NHT-event-driven fpmsyncd backwalk fast-fixup** so that when a nexthop fails, route-convergence time is minimized — covering PIC core, PIC edge, and the directly-connected local-link-down case.

> This document is the project-level **consolidated design** — the single source of truth that integrates the whole project's design in distilled form (per-component: role + key decisions + link to the detailed LLD).
> - Process artifact (brainstorming/design/grill-me decision evolution): [`specs/2026-07-05-ribfib-convergence-design.md`](specs/2026-07-05-ribfib-convergence-design.md)
> - Implementation plans: `plans/2026-07-06-ribfib-convergence-{A,B,C}.md`
> - Delivered LLD (per-component detail): [`../convergence-lld/`](../convergence-lld/)

## 1. End-to-end pipeline

The feature spans **four runtime components + one test layer**. A nexthop failure flows through them as follows:

```
                          ┌─────────────────────────────────────────────────────────────┐
   nexthop failure ──▶    │ sonic-frr (Zebra)                                             │
                          │  Trigger A: recursive nexthop → emit DPLANE_OP_NHT_EVENT_UPDATE│
                          │  Trigger B: local link down   → flag NHG REINSTALL_FPM_ONLY    │
                          └───────────────┬─────────────────────────────┬─────────────────┘
                       (Trigger A path)   │                             │  (Trigger B path)
                                          ▼                             │
                          ┌─────────────────────────────┐              │
                          │ sonic-fib                    │              │
                          │  encode NhtEvent → JSON       │              │
                          └───────────────┬─────────────┘              │
                                          ▼                             │
                          ┌─────────────────────────────┐              │
                          │ dplane_fpm_sonic (FPM provider)│             │
                          │  assemble RTM_NEWNHTEVENT(6000)│             │
                          └───────────────┬─────────────┘              │
                                          ▼                             ▼
                          ┌─────────────────────────────────────────────────────────────┐
                          │ sonic-swss / fpmsyncd                                         │
                          │  Trigger A: decode NhtEvent → NHGMgr backwalk fast-fixup       │
                          │  Trigger B: ordinary NHG update (member already pruned)        │
                          │  → write APPL_DB NEXTHOP_GROUP_TABLE                            │
                          └─────────────────────────────────────────────────────────────┘

   test layer: UT (fpmsyncd) · topotest (sonic-frr) · sonic-mgmt (end-to-end)
```

Key point: **Trigger A** (recursive nexthop) drives the new NHT-event pipeline through all four components; **Trigger B** (local link down) reuses the existing NHG-install path and only touches sonic-frr + fpmsyncd.

## 2. Component: sonic-frr — NHT event generation

Detects a nexthop failure and hands it to the dplane. Two triggers:

- **Trigger A (recursive nexthop, has RNH)**: from `zebra_rnh_eval_nexthop_entry()`, emit a new `DPLANE_OP_NHT_EVENT_UPDATE` carrying the prev/curr resolved prefix + NHG id.
  - `re==NULL` disambiguation: `zebra_rnh_resolve_nexthop_entry()` gains a `route_entry_queued` out-parameter; trigger condition `state_changed && !route_entry_queued` (true delete → send; QUEUED transient → suppress).
  - Enqueued via `dplane_update_enqueue()` and marked `dplane_ctx_set_skip_kernel()` so it traverses the FPM provider without being programmed into the kernel.
- **Trigger B (directly-connected ECMP member, no RNH)**: from `if_down_nhg_dependents()`, flag the affected NHG closure `NEXTHOP_GROUP_REINSTALL_FPM_ONLY` and re-send it to the FPM only (kernel already knows the link is down). No new event type; fpmsyncd handles it as an ordinary NHG update. Trigger placed at the interface-down entry, not in the `zebra_nhg_set_valid(false)` primitive (which would over-trigger during NHG delete / VRF teardown).
- **Preserve NHG ID**: no special handling — Zebra's deterministic ID computation is sufficient.

Naming: dplane ctx sub-struct `struct dplane_rnh_info` (union member `u.rnh_info`); accessors `dplane_ctx_get_rnh_*`; constructor `dplane_nht_event_update(rnh_prefix, prev_resolved_prefix, prev_nhg_id, curr_resolved_prefix, curr_nhg_id)`; `struct rnh` gains a `resolved_nhg_id` field maintained by `copy_state()`.

Detailed LLD: [`../convergence-lld/ribfib-convergence-frr.md`](../convergence-lld/ribfib-convergence-frr.md)

## 3. Component: sonic-fib — NhtEvent encode/decode library

Provides the wire contract shared by FRR (encode) and fpmsyncd (decode):

- New `schema/NhtEvent.json` — 5 fields: `rnh_prefix`, `prev_resolved_prefix`, `prev_resolved_nhg_id`, `curr_resolved_prefix`, `curr_resolved_nhg_id`.
- Generated C++ class + C-API (`nht_event_encode()` / `NhtEvent::decode()`), with mirrored `nhtevent.*` templates.
- `render_schema.py` is **extended** (new function + new `main()` branch); the existing `NextHopGroupFull` code path is left untouched to avoid regression. Fully independent from `NextHopGroupFull`.

Detailed LLD: [`../convergence-lld/ribfib-convergence-sonic-fib.md`](../convergence-lld/ribfib-convergence-sonic-fib.md)

## 4. Component: dplane-encoding — FPM provider message assembly

The SONiC-specific FPM provider (`dplane_fpm_sonic.c`, which replaces stock `dplane_fpm_nl.c` at runtime) serializes the event:

- Adds `RTM_NEWNHTEVENT` (6000) and a `DPLANE_OP_NHT_EVENT_UPDATE` dispatch handler.
- Assembles `nlmsghdr` (type 6000) + `struct rtmsg` (family of the RNH prefix) + the `FPM_NHA_JSON_STR` NLA, reading ctx via the `dplane_ctx_get_rnh_*` accessors and calling `nht_event_encode()` for the JSON payload.

Detailed LLD: [`../convergence-lld/ribfib-convergence-dplane-encoding.md`](../convergence-lld/ribfib-convergence-dplane-encoding.md)

## 5. Component: sonic-swss / fpmsyncd — NHGMgr backwalk fast-fixup

Consumes the NHT event and runs the fast-fixup. **Implementation structure**: everything integrated into the existing `NHGMgr` class, no new files (Approach B).

- **Dispatch**: `routesync` handles `RTM_NEWNHTEVENT` → decode → **Phase-1 gate** (only `curr_resolved_nhg_id == 0` proceeds) → `NHGMgr::onNhtEvent()`.
- **Two reverse indices** (`map<string, set<RIBNHGEntry*>>`, non-overlapping by construction):
  - `m_nexthop_to_global_RIBNHG` → **PIC core** (`backwalkPicCore`, start = `prev_resolved_nhg_id`)
  - `m_nexthop_to_vrf_RIBNHG` → **PIC edge** (`backwalkPicEdge`, start set from the index)
- **Affected set**: per-node `{fully_disabled, enable_group{dep→bool}}` (`modified_set`); the gateway match is checked at every node.
- **Forward walk**: `resolveLeafPaths()` / `collectAllLeafPaths()`, with a `visited set<ribID>` cycle guard (no fixed depth cap).
- **Fast-fixup policy**: surviving path → rewrite that NHG to APPDB; all failed → skip the APPDB write but keep walking upward. **Backwalk only writes APPDB; it never mutates the in-memory NHG** (refreshed by the subsequent normal NHG update).
- **Prune**: generic rule on `walk_result` + `depends.size()`, no `isGatewayNhg()` type check.
- **Known improvement (non-blocking)**: the indices currently store `RIBNHGEntry*` pointers; the HLD prefers IDs as more robust (warm-reboot reshuffles NHG IDs). Safe as long as lifecycle add/remove is paired.

Detailed LLD: [`../convergence-lld/ribfib-convergence-swss.md`](../convergence-lld/ribfib-convergence-swss.md)

## 6. Component: test

Three-layer strategy:

- **UT** (`sonic-swss/tests/mock_tests`): backwalk algorithm branches, reverse-index maintenance, message parsing, error handling.
- **topotest** (`sonic-frr/tests/topotests`): NHT trigger conditions, dplane ctx, FPM message format, QUEUED-transient suppression.
- **sonic-mgmt** (`tests/srv6`): end-to-end convergence timing (NHG-before-ROUTE), real SRv6 VPN gateway prune, no route flap.

Detailed LLD: [`../convergence-lld/ribfib-convergence-test.md`](../convergence-lld/ribfib-convergence-test.md)

## 7. Cross-cutting decision summary

| # | Decision | Component | Conclusion |
|---|----------|-----------|------------|
| 1 | Scope | all | Full set: FRR NHT→dplane + fpmsyncd backwalk + PIC core/edge |
| 2 | NHT message carrier | frr / sonic-fib / dplane-encoding | `RTM_NEWNHTEVENT`(6000) + `DPLANE_OP_NHT_EVENT_UPDATE`; JSON 5 fields |
| 3 | `re==NULL` disambiguation | frr | `route_entry_queued` out-param; `state_changed && !route_entry_queued` |
| 4 | Local link down | frr / swss | `INSTALLED_FPM_ONLY` re-send (Trigger B), not an NHT event |
| 5 | Implementation structure | swss | All in `NHGMgr`, no new files (Approach B) |
| 6 | Reverse indices | swss | Two disjoint maps (global → PIC core, vrf → PIC edge) |
| 7 | Fast-fixup policy | swss | Partial → rewrite APPDB; full-fail → skip but keep walking; never touch in-memory NHG |
| 8 | Prune | swss | Generic rule, no `isGatewayNhg()` |
| 9 | Cycle guard | swss | `visited set`, no fixed depth cap |
| 10 | Preserve NHG ID | frr | No special handling |
| 11 | sonic-fib independence | sonic-fib | New schema + extend `render_schema.py`; `NextHopGroupFull` untouched |

## Change Log

| Date | Feature | Change |
|------|---------|--------|
| 2026-07-05 | ribfib-convergence | Initial design: FRR NHT→dplane, sonic-fib NhtEvent encode/decode, fpmsyncd NHGMgr backwalk (PIC core + edge), three-layer test strategy |
| 2026-07-06 | ribfib-convergence | Post-grill-me refactor: two reverse indices, per-node enable/disable, removed isGatewayNhg in favor of generic prune, visited set, route_entry_queued disambiguation |
| 2026-07-13 | ribfib-convergence | Trigger B (local link down) changed to INSTALLED_FPM_ONLY re-send (referencing the existing HLD/PR implementation approach); naming aligned with the existing PR implementation; `struct rnh` gains `resolved_nhg_id` |
