# Proposal: RIB/FIB Route Convergence

**Date**: 2026-04-16
**Change Name**: ribfib-convergence
**Status**: Draft

---

## What

Implement route convergence handling within the SONiC FIB block (fpmsyncd) using a backwalk infrastructure over the NHG dependency graph. When a nexthop becomes unreachable (NHT event), the system performs rapid fixups on affected NHGs by disabling failed paths, minimizing traffic loss before FRR reconverges.

This covers:
1. **Backwalk infrastructure** -- A generalized graph traversal mechanism over the SONiC Zebra NHG table that walks from a triggered NHG to all dependent NHGs via "resolve through" relationships.
2. **Quick fixup mechanism** -- Walk and prune spec callbacks that disable failed paths in affected NHGs and write updates to APPDB.
3. **PIC Core handling** -- Updating NHGs in the Zebra NHG chain when an intermediate nexthop is withdrawn.
4. **PIC Edge handling** -- Updating SONiC NHGs (gateway NHGs) when a nexthop address is withdrawn, using the NEXTHOP->SONIC NHG ID table.

## Why

In the current SONiC architecture, orchagent handles local port-down events with rapid load-balance member updates. However, for all other failure scenarios (remote nexthop withdrawal, intermediate route withdrawal), FRR creates new NHGs and migrates prefixes sequentially. The traffic loss window is proportional to the number of prefixes -- this is prefix-dependent convergence.

The RIB/FIB design introduces Prefix Independent Convergence (PIC). By maintaining NHG dependency chains in fpmsyncd and reacting to NHT events with a backwalk, all affected NHGs can be updated in a single pass, reducing convergence time from O(prefixes) to O(NHG_depth).

Key drivers:
- **SONiC Routing Working Group consensus**: NHG ID handling and PIC have been discussed extensively (meeting minutes from 2025-01 through 2025-03).
- **FRR community direction**: FRR team agreed to separate RIB (Zebra) from FIB (data plane), with each data plane managing its own FIB.
- **SRv6 VPN divergence**: Linux kernel forwarding chain differs from SONiC for SRv6 VPN, requiring a separate FIB translation layer.

## Scope

### In Scope
- `fib_nhg_trigger_node_quick_fixup()` -- Entry point for NHT event handling
- `fib_nhg_back_walk()` -- Backwalk traversal on `RIBNHGTable`
- `fib_nhg_walk_spec_for_node_quick_fixup` -- Walk spec callback for disabling failed paths
- `fib_nhg_prune_spec_for_node_quick_fixup` -- Prune spec callback for stopping traversal
- `m_resolved_enable_group` field in `RIBNHGEntry` -- Tracks enabled/disabled state of resolved paths
- `RIBNHGEntry::getNextHopGroupFields()` update -- Excludes disabled paths from APPDB output
- Test cases for 3 topologies: global table recursive routes (2 variants), SRv6 VPN case

### Out of Scope
- SONiC NHG table walk mechanism (marked as TODO in LLD)
- Changes to orchagent or syncd
- Warm reboot NHG ID persistence (separate feature, handled by Zebra NHG ID to SONiC NHG ID mapping)
- CLI for displaying FIB internal tables (separate task)
- FRR-side changes (original NHG, resolve-through/resolve-via in zebra_dplane_ctx, NHT trigger to dplane)

## Key Decisions

1. **Backwalk over forward walk**: NHT triggers a backwalk (child-to-parent in "resolve through" direction) because we need to update all NHGs that depend on the failed nexthop, not just the immediate children.
2. **ID-based lookups, not pointers**: Dependencies use NHG ID lookups rather than direct pointers for robustness.
3. **Visited set for cycle prevention**: Each backwalk maintains a `visited_node_set` to prevent infinite loops in the NHG graph.
4. **Dual-phase walk**: Phase 1 walks the Zebra NHG chain (PIC core). Phase 2 walks the NEXTHOP->SONIC NHG table (PIC edge). This separation handles global table and SRv6 VPN cases differently.
5. **Skip single-path disable**: If an NHG has only one path and it must be disabled, skip the APPDB update (don't write an empty NHG).

## Risks and Open Questions

| Risk | Mitigation |
|------|-----------|
| NHG graph could be deep, causing slow backwalk | Visited set bounds traversal; prune spec stops at gateway NHGs for SRv6 |
| Race between NHT fixup and FRR reconvergence | Fixup is idempotent; FRR's subsequent NHG update overwrites fixup state |
| Memory growth from m_resolved_enable_group | Map size bounded by NHG resolved members; reset on NHG update events |
| SONiC NHG table walk not yet implemented | Scoped out; PIC edge cases deferred for follow-up |

## Dependencies

- **Existing codebase**: `nhgmgr.cpp` / `nhgmgr.h` in sonic-swss fpmsyncd (rib_fib branch)
- **FRR changes**: Requires "resolve through" and "resolve via" fields in `zebra_dplane_ctx`, NHT trigger to dplane (Phase 1 FRR tasks from HLD)
- **RIBNHGTable and RIBNHGEntry**: Existing classes that manage the SONiC Zebra NHG table

## Success Criteria

1. All 3 test topologies pass (global recursive topology 1 & 2, SRv6 VPN)
2. Backwalk correctly identifies and updates only affected NHGs per NHT event
3. No unnecessary APPDB writes for unaffected NHGs
4. Quick fixup completes before FRR reconvergence for typical topologies
5. Unit test coverage >= 80% for new code in fpmsyncd
