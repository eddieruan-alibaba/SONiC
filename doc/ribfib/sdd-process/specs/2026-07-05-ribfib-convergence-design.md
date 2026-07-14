# RIB/FIB Convergence Phase 2 — Design Spec (SDD Process Artifact)
> **AI-facing SDD artifact** — This document is written to be consumed by an AI agent to generate code. It is a Spec-Driven Development process artifact, not human-oriented documentation.

> **What this is**: This document is the process artifact of the `ribfib-convergence` project across the SDD stages **brainstorming → design → grill-me**. It captures "how the intent was clarified, how approaches were weighed, how decisions evolved, and how the design was pressure-tested." It is AI-facing (for collaborative reconstruction). The clean, delivery-oriented **LLD lives in** [`../../convergence-specs/`](../../convergence-specs/) (6 documents); the two serve different purposes and complement each other.
>
> Source: the real human-AI dialogue of this project's SDD sessions (distilled, with key verbatim quotes preserved and translated).

- **Feature**: ribfib-convergence (route-convergence acceleration on top of the RIB/FIB layered architecture)
- **Corresponding HLD**: RIB/FIB HLD — Phase 2 Convergence
- **Stages**: brainstorming (intent / requirements / approaches) → design (decision evolution) → grill-me (pressure-test)
- **Delivered LLD**: [`../../convergence-specs/`](../../convergence-specs/) (overview + frr + sonic-fib + dplane-encoding + swss + test)
- **Implementation plans**: `../plans/2026-07-06-ribfib-convergence-{A,B,C}.md`

---

## 1. Intent and Constraints (Brainstorming Starting Point)

### 1.1 Original intent (verbatim)

> **User**: "I'm going to develop this using our SDD process, doing it AI-Native... the Phase 2 convergence part of the HLD is the feature I'm building this time."

Goal: on top of the RIB/FIB layered architecture (Phase 1 already landed), implement an **NHT-event-driven fpmsyncd backwalk fast-fixup mechanism** to shorten route-convergence time when a nexthop fails, covering both PIC core and PIC edge.

### 1.2 Hard constraints

- **Implementation isolation**: reference only the HLD and this project's working branch; **do not look at any other implementation branch's code** — the point is to design independently, then compare against existing implementations afterward to find gaps.
> **User**: "Apart from the ribfib HLD, you are not allowed to read any code content outside (this working branch)."
- **Repo scope**: `sonic-buildimage/` (contains the `sonic-frr` and `sonic-swss` submodules; `sonic-fib` has been merged into `src/libraries/`).
- **Process constraint**: the user explicitly required running brainstorming before propose.
> **User**: "Shouldn't we start with brainstorming first?"

---

## 2. Requirement-Clarification Q&A (Real Decisions)

The brainstorming stage clarified point by point; the final choices:

| # | Clarification | Decision |
|---|---------------|----------|
| 1 | Scope | **Full set**: FRR NHT→dplane + fpmsyncd backwalk + PIC core/edge |
| 2 | Backwalk granularity | **Both Core + Edge**, covering Global table / SRv6 VPN / VXLAN EVPN |
| 3 | Fast-fixup policy | Surviving path remains → rewrite that NHG to APPDB (prune the failed path); **all failed → do NOT write APPDB** (keep hardware / blackhole), but **backwalk keeps propagating upward** |
| 4 | NHT message carrier | FRR dplane adds `DPLANE_OP_NHT_EVENT_UPDATE`; FPM wire adds `RTM_NEWNHTEVENT` (type **6000**) + `struct rtmsg` + JSON (`FPM_NHA_JSON_STR`); **5 fields** |
| 5 | Preserve NHG ID | **No special handling** — Zebra's own deterministic ID computation is sufficient; the FIB side re-sends the whole NHG update on change |
| 6 | Phase 1 gate | **Only handle `curr_resolved_nhg_id == 0`** (nexthop fully unreachable); `curr != 0` (route switch / recovery) deferred to a later phase |
| 7 | CLI | Not this round |

NHT message 5 fields: `rnh_prefix`, `prev_resolved_prefix`, `prev_resolved_nhg_id`, `curr_resolved_prefix`, `curr_resolved_nhg_id`.

---

## 3. Approach Exploration and Trade-offs (fpmsyncd backwalk infrastructure)

Brainstorming proposed 3 candidate architectures:

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **A** (AI-recommended) | Separate `BackwalkEngine` class + `nhtevent.cpp/h` handler; nhgmgr only manages tables | Separation of concerns, independently unit-testable, later `curr!=0` only touches dispatch | 2 new file-pairs, build-config changes |
| **B** (chosen) | Fold everything into the existing `NhgMgr` class | No new files, direct access to table internals | nhgmgr is already large, walk logic coupled to table management, slightly harder to unit-test |
| **C** | Templated generic graph Walker + strategy pattern | Maximum reuse | Over-engineering (today there is only one walk-spec + two prune variants), heavier to compile/debug |

**Decision: choose B** (overruling the AI-recommended A).

> **User**: "I think Approach B — a thousand lines of code really isn't much; adding more files is what creates more to maintain."

Corresponds to LLD decision-table #9 "Implementation structure = everything integrated into the NhgMgr class, no new files."

---

## 4. Design Decisions and Evolution (design dialogue)

This section records, for each key decision, the **conclusion + alternatives considered + why rejected + evolution trajectory** — this is the core value of the SDD process artifact.

### 4.1 NHT trigger point

- **Conclusion**: emit the NHT event from `zebra_rnh_eval_nexthop_entry()`.
> **User**: "zebra_rnh_eval_nexthop_entry() — we should add it inside this function, right?"
- Rationale: this is the core decision point for NHT state changes; `copy_state()` provides prev/curr, and `state_changed` already exists.

### 4.2 The `route_entry_queued` out-parameter (disambiguating `re==NULL`) — **flipped by grilling**

- **v1 (brainstorm draft)**: `if (state_changed && re && ROUTE_ENTRY_QUEUED)` — the reasoning at the time was "QUEUED means the upper layer hasn't finished recomputing, so the lower layer needs to be notified."
- **Grill correction**: the AI found this condition **logically inverted**, and that `re &&` blocks the primary use case:
> **AI**: "The core trigger scenario of PIC Phase 1 is a nexthop becoming unreachable (curr==0); at that point resolve returns re=NULL... your condition `re &&` is false when re=NULL → the event is not sent, which blocks exactly the scenario that most needs sending."
- **Root cause**: `re == NULL` has two meanings — (1) the route was truly deleted (unreachable) → **must send**; (2) candidate routes are in a QUEUED transient → **must not send**.
- **Final**: add a `bool *route_entry_queued` out-parameter to `zebra_rnh_resolve_nexthop_entry()`, and change the trigger condition to `if (state_changed && !route_entry_queued)`. Truth table: deleted/unreachable (re=NULL, queued=false) → send; QUEUED transient (re=NULL, queued=true) → suppress; IGP switch (new re, queued=false) → send.
- This was the single most critical logic inversion the grilling caught in the whole design process.

### 4.3 Affected-NHG set → per-node enable/disable — **4 iterations**

- **v1, direct match only**: only checked "does this NHG contain failedNh?".
> **User**: "You can't just look at the failed nexthop; you also have to consider indirectly-failed ones, so we'd better have a set to store all affected NHGs." → introduced `affectedSet`.
- **v2, a boolean affectedSet is insufficient**: when forward-walking an affected non-leaf, it still collected paths from an "already fully dead" subtree.
> **AI**: "affectedSet is just a boolean flag (changed / unchanged); it can't distinguish 'fully disabled' from 'partially affected'."
- **v3, per-node enable/disable**: each modified node stores `{fully_disabled, enable_group{dep→bool}}`; the forward walk skips disabled deps.
> **User**: "Adopt the per-node enable/disable scheme."
- **v4, gateway match must be checked at every node**: checking the gateway match only at the start node misjudges mid-walk nodes.
> **User**: "The gateway match must be checked at every dependent node, not only at the start."
- **v5, start-node semantics**: the start is merely the walk entry; only mark `fully_disabled` when its own `gateway == failedNh`, otherwise do not add it to modifiedSet.
> **AI**: "Don't blindly mark it fully_disabled."

### 4.4 Two reverse indices (PIC Core vs PIC Edge)

- The draft conflated PIC Core / PIC Edge onto a single table.
> **User**: "I earlier used the 5.2 m_nexthop_to_global_RIBNHG directly for PicEdge, merging the two data sources into one table — that was my mix-up."
- **Conclusion**: two independent `map<string, set<RIBNHGEntry*>>` — `m_nexthop_to_global_RIBNHG` (PIC Core) and `m_nexthop_to_vrf_RIBNHG` (PIC Edge). They are non-overlapping by construction.
- **Rejected alternative**: a third `m_nexthop_to_sonic_NHG` table.
> **User**: "From nexthop to rib id we already find the zebra id, and from the zebra id we can find the sonic id... is it still necessary to maintain m_nexthop_to_sonic_NHG?" → **not needed** (each RIBNHGEntry already carries its SONiC NHG ID).
- **Why keep the nexthop→RIB-ID index**: precision — two routes may resolve the same `fc06::2`; `prev_resolved_nhg_id` locates the correct start node.

### 4.5 Storing `RIBNHGEntry*` pointer vs RIB NHG ID

- In discussion the AI leaned toward **storing the ID** (a pointer risks dangling on `delNHGFull()`; the HLD also prefers IDs as more robust; warm-reboot reshuffles NHG IDs, making raw pointers even more fragile).
- **Current code stores pointers**, recorded as a **low-priority improvement item** (safe as long as lifecycle add/remove is paired; it only crashes if one cleanup is missed), non-blocking. The LLD text describes it in ID terms.

### 4.6 Removing `isGatewayNhg()` → generic prune rule

- The initial idea was to detect gateway via the `SonicPICContent` type / `m_sonic_obj_type`.
- **Conclusion**: there is **no** `isGatewayNhg()` in the implementation. Prune uses a generic rule based on `walk_result` + topology: multipath miss → keep (defensive); single-path miss → prune; hit → propagate upward.
> **AI**: "The HLD's 'prune at the Gateway NHG' is simplified in the implementation to the generic 'prune on miss' — a Gateway NHG that doesn't contain failedNh simply won't be hit, so it is pruned naturally."

### 4.7 Forward-walk cycle guard: visited set (instead of a fixed depth cap)

- **Conclusion**: use a `visited set<ribID>`, **dropping the fixed `MAX_NHG_RECURSION` depth cap**.
> **AI**: "The DAG guarantee lives on the Zebra side; fpmsyncd is only a consumer and may receive out-of-order messages forming a transient cycle → recursion-only would crash; a fixed depth cap can't be tuned. Cheapest: one `set<ribID>` per forward walk; on revisit → log error + return empty."

### 4.8 sonic-fib NHT schema: mirror, don't modify

> **User**: "Let's keep the NHT event stuff separate from NexthopGroupFull; mirror the file structure rather than adding into the original files."
> **User**: "Don't randomly modify the existing functions in render_schema.py — extend it instead."
- **Conclusion**: add `schema/NhtEvent.json` + mirrored `nhtevent.*`; `render_schema.py` uses **new functions + a new `main()` branch**, leaving the `NextHopGroupFull` path untouched (to avoid regression risk).

### 4.9 Local link down — **adjusted by referencing the existing HLD and existing-PR implementation approach**

- **Background**: three test cases (`t1_local` / `t2_local` / `remote_igp_filtered`) failed — after an interface goes down, `fc06::2` was never pruned, `swss.rec` had no NHG SET, only the slow path ran. Root cause: directly-connected members have no RNH, so the §4.1 NHT trigger doesn't fire for them.
- **v1 approach**: have FRR also emit an NHT event for directly-connected members in `if_down_nhg_dependents()`, with fpmsyncd fallback handling. It was implemented on the working branch but **did not work**.
- **P0 root cause**: the NHT event ctx used **`dplane_provider_enqueue_to_zebra()`** — which pushes the ctx into zebra's result-callback queue and **bypasses the dplane provider chain**, so the FPM provider never sees it and fpmsyncd never receives the event. Every other normal dplane op uses `dplane_update_enqueue()`.
- **Adjusted approach (referencing the existing HLD and existing-PR implementation approach)**: local link down **does not emit an NHT event and does not run backwalk**; instead:
  1. On interface down, mark the failed nexthop **inactive**, including the corresponding member inside composite NHGs (pitfall: singleton weight=1 vs in-group weight=255 — the comparison must **ignore weight**);
  2. When assembling the NHG for the FPM, **skip inactive members** (the emitted group already excludes `fc06::2`);
  3. Trigger the affected NHG to be **re-sent to the FPM only, skipping the kernel** (the kernel already knows the link is down) — i.e. the `NEXTHOP_GROUP_REINSTALL_FPM_ONLY` / `INSTALLED_FPM_ONLY` mechanism. fpmsyncd handles it as an ordinary NHG update.
- **Benefits**: no new event type, no second emit path, fpmsyncd untouched, reuses the mature NHG-install pipeline.
- **Trigger-point choice**: the re-send is placed in **`if_down_nhg_dependents()` (the interface-down entry)**, not in the `zebra_nhg_set_valid(false)` branch.
> **AI**: "The true symmetry is 'link-up handler ↔ link-down handler', not 'set_valid(true) ↔ set_valid(false)'; set_valid is a low-level primitive also called during NHG delete / VRF teardown, so putting it there would over-trigger re-sends on teardown paths."

---

## 5. GBrain References

No relevant historical records (GBrain MCP was not connected in this round).

---

## 6. Grill-Me Pressure-Test Record

The design took shape **through being challenged**. The table below summarizes the pressure-tested decisions and their outcomes:

| Challenged decision | Outcome |
|---------------------|---------|
| `route_entry_queued`: QUEUED→send | **Flipped** to `!QUEUED→send` + out-param disambiguation (§4.2) |
| Boolean affectedSet | **Rejected**, changed to per-node enable/disable + per-node gateway check (§4.3) |
| Start node auto `fully_disabled` | **Revised** to gateway-conditional (§4.3 v5) |
| `re &&` trigger guard | **Removed** (it blocked the primary use case, §4.2) |
| `isGatewayNhg()` | **Deleted**, replaced by generic prune (§4.6) |
| Fixed recursion depth cap | **Replaced** by visited set (§4.7) |
| local link down via emitting an NHT event | **Adjusted** to `INSTALLED_FPM_ONLY` re-send (referencing the existing HLD/PR implementation approach, §4.9) |

### Design points that survived the pressure test unchanged

- The 5-field NHT message + `RTM_NEWNHTEVENT`(6000) / `DPLANE_OP_NHT_EVENT_UPDATE` carrier
- The fast-fixup policy (partial failure rewrites APPDB; full failure skips APPDB but keeps walking; never touches the in-memory NHG)
- Phase-1 gate = `curr_resolved_nhg_id == 0`
- Two independent reverse indices (global vs vrf)
- The generic prune rule (no `isGatewayNhg()`)
- The visited-set cycle guard
- Approach B (everything integrated into NhgMgr)

---

## 7. Related Artifacts

- **Delivered LLD** (clean, code-repo/community-facing): [`../../convergence-specs/`](../../convergence-specs/)
  - `ribfib-convergence-overview.md` / `-frr.md` / `-sonic-fib.md` / `-dplane-encoding.md` / `-swss.md` / `-test.md`
- **Implementation plans**: `../plans/2026-07-06-ribfib-convergence-{A,B,C}.md`
- **Consolidated design**: [`../ribfib-convergence_design.md`](../ribfib-convergence_design.md)
