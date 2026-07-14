# RIB/FIB Convergence Overview

- **Feature**: ribfib-convergence
- **Corresponding HLD**: [`../ribfib.md`](../ribfib.md) (Phase 2 – Route Convergence Handling)

This document is the overview for the `ribfib-convergence` project. Goal: on top of the SONiC RIB/FIB layered architecture (Phase 1), implement an NHT-event-driven fpmsyncd backwalk fast-fixup mechanism to shorten route convergence time when a nexthop fails, covering both PIC core and PIC edge. It only summarizes what each module is responsible for; per-module implementation detail lives in the linked sub-specs.

**Repositories involved:**
- `sonic-frr` — Zebra RNH/dplane NHT event generation
- `sonic-buildimage`
  - `src/sonic-frr/dplane_fpm_sonic/` — SONiC-specific FPM provider (message assembly / encoding)
  - `src/libraries/sonic-fib/` — `libnexthopgroup` encode/decode library
- `sonic-swss/fpmsyncd` — the NHGMgr backwalk core
- `sonic-mgmt/tests/srv6/` — system-level convergence tests

**Prerequisites** — Phase 1 (RIB/FIB base infrastructure) has already landed across all repositories:
- `sonic-frr`: `zebra_dplane_ctx` carries depends/dependents; `NEXTHOP_GROUP_RECEIVED` flag; `--nhg-fib` global knob
- `sonic-fib` (`sonic-buildimage/src/libraries/sonic-fib`): `libnexthopgroup` encode/decode library + JSON schema
- `sonic-swss/fpmsyncd`: `nhgmgr` class with SONiC zebra NHG table (`RIBNHGTable`) / SONiC PIC content table / SONiC NHG ID map; PIC context / gateway NHG; SRv6 VPN path; APP_STATE_DB

---

## 1. Key Design Decisions

| # | Decision | Conclusion |
|---|----------|------------|
| 1 | Scope | Full set: FRR NHT→dplane + fpmsyncd backwalk + PIC core/edge |
| 2 | Backwalk granularity | Core + Edge; supports Global table / SRv6 VPN / VXLAN EVPN |
| 3 | Fast-fixup policy | If any valid paths remain → rewrite APPDB; if all fail → skip APPDB write but continue walking |
| 4 | NHT message carrier | FPM adds `RTM_NEWNHTEVENT` (6000); dplane adds `DPLANE_OP_NHT_EVENT_UPDATE` |
| 5 | Two trigger paths | Recursive nexthop (has RNH) → emit NHT event; directly-connected ECMP member on link down (no RNH) → re-send via `NEXTHOP_GROUP_REINSTALL_FPM_ONLY`, not an NHT event |
| 6 | Preserve NHG ID | No special handling. Zebra's built-in mechanism is sufficient |
| 7 | Prune policy | Generic rule on `walk_result` + `depends.size()`; "stop at gateway NHG" is a surface behavior, no `isGatewayNhg()` type check |
| 8 | Forward walk | Embedded in each backwalk step; recurses along enabled `depends` down to leaves |
| 9 | Implementation structure | Everything integrated into the `NhgMgr` class; no new files |
| 10 | CLI | Not in this cycle |

---

## 2. Module responsibilities

Each module below is described at a responsibility level only. Follow the link for the detailed design, data structures, and pseudo-code.

### 2.1 sonic-frr — NHT event generation

Detects a nexthop failure through two trigger points and drives fpmsyncd fast-fixup:

- **Trigger A (recursive nexthop):** when a Zebra RNH resolution settles (nexthop unreachable or resolution moved), emit a new `DPLANE_OP_NHT_EVENT_UPDATE` down the dplane, carrying the prev/curr resolved prefix + NHG id on the ctx and disambiguating the transient QUEUED case so events fire only on a settled state.
- **Trigger B (directly-connected ECMP member):** such members carry no RNH, so on interface down the affected NHG closure is flagged for FPM-only reinstall (`NEXTHOP_GROUP_REINSTALL_FPM_ONLY`) and re-sent through the existing NHG path — no new event type.

→ [`ribfib-convergence-frr.md`](ribfib-convergence-frr.md)

### 2.2 sonic-fib — NhtEvent encode/decode

Provides the `NhtEvent` JSON schema plus the generated C++ class and C-API used to encode (FRR side) and decode (fpmsyncd side) the event. Fully independent from `NextHopGroupFull`; `render_schema.py` is extended without touching existing code paths.

→ [`ribfib-convergence-sonic-fib.md`](ribfib-convergence-sonic-fib.md)

### 2.3 dplane-encoding — FPM provider message assembly

The SONiC-specific FPM provider serializes a `DPLANE_OP_NHT_EVENT_UPDATE` ctx into the `RTM_NEWNHTEVENT` (6000) netlink message, placing the encoded JSON into the `FPM_NHA_JSON_STR` NLA.

→ [`ribfib-convergence-dplane-encoding.md`](ribfib-convergence-dplane-encoding.md)

### 2.4 sonic-swss (fpmsyncd) — NHGMgr backwalk

Consumes the NHT event and runs the backwalk fast-fixup: PIC core (starting from the previously resolved NHG) and PIC edge (starting from VRF/VPN NHGs that reference the failed nexthop), driven by two reverse indices. Recomputes remaining valid paths and rewrites only APPDB; the in-memory NHG is left to the subsequent normal FRR NHG update.

→ [`ribfib-convergence-swss.md`](ribfib-convergence-swss.md)

### 2.5 Test — UT + topotest + sonic-mgmt

Three-layer test strategy: fpmsyncd UT for algorithm branches / boundaries / message parsing, FRR topotest for NHT trigger conditions and FPM message format, and sonic-mgmt E2E for end-to-end convergence timing and route-flap absence.

→ [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

---

## 3. Compatibility

- `RTM_NEWNHTEVENT` (6000) is a new FPM message type; older fpmsyncd instances silently ignore unknown types.
- The `NextHopGroupFull` code path is untouched.
- When the `--nhg-fib` knob is disabled, FRR does not emit NHT events (reusing the existing knob semantics).
