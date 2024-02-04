<!-- omit in toc -->
# Recursive Route Handling HLD
<!-- omit in toc -->
## Revision
| Rev |     Date    |       Author           | Change Description                |
|:---:|:-----------:|:----------------------:|-----------------------------------|
| 0.1 | Oct    2023 |                        | Initial Draft                     |

<!-- omit in toc -->
## Table of Content
- [Goal and Scope](#goal-and-scope)
- [FRR's Current Limitations](#frrs-current-limitations)
- [Triggers Events](#triggers-events)
- [FRR Current Approaches](#frr-current-approaches)
  - [NH Dependency Tree](#nh-dependency-tree)
  - [NHT List from Route Node](#nht-list-from-route-node)
  - [NHG Update Trigger](#nhg-update-trigger)
- [High Level Design](#high-level-design)
- [Low Level Design](#low-level-design)
  - [Routes Updating](#routes-updating)
    - [Data Structure Modifications](#data-structure-modifications)
      - [struct nhg\_hash\_entry](#struct-nhg_hash_entry)
      - [struct route\_entry](#struct-route_entry)
      - [struct hash \*nhgs\_ctx\_hash](#struct-hash-nhgs_ctx_hash)
    - [Routes Updating Handling](#routes-updating-handling)
  - [Fast Convergence for Route Withdrawal](#fast-convergence-for-route-withdrawal)
    - [Data Structure Modifications](#data-structure-modifications-1)
    - [Fast Convergence Handling](#fast-convergence-handling)
  - [The Handling of zebra\_rnh\_refresh\_dependents()](#the-handling-of-zebra_rnh_refresh_dependents)
  - [Dataplane refresh for Nexthop group change](#dataplane-refresh-for-nexthop-group-change)
  - [FPM's new schema for recursive NHG](#fpms-new-schema-for-recursive-nhg)
  - [Orchagent changes](#orchagent-changes)
- [Unit Test](#unit-test)
  - [Normal Case's Forwarding Chain Information](#normal-cases-forwarding-chain-information)
  - [Test Case 1: local link failure](#test-case-1-local-link-failure)
  - [Test Case 2: IGP remote link/node failure](#test-case-2-igp-remote-linknode-failure)
  - [Test Case 3: IGP remote PE failure](#test-case-3-igp-remote-pe-failure)
  - [Test Case 4: BGP remote PE node failure](#test-case-4-bgp-remote-pe-node-failure)
  - [Test Case 5: Remote PE-CE link failure](#test-case-5-remote-pe-ce-link-failure)
- [References](#references)

## Goal and Scope
A recursive route is a routing mechanism in which the routing decision for a specific destination is determined by referring to another routing table, which is then looked up recursively until a final route is resolved. Recursive routing is a key concept in routing protocols and is often used in complex network topologies to ensure that data reaches its intended destination, even when that destination is not directly reachable from the originating device. In many cases, recursive routes are used in VPN or tunneling scenarios.

## FRR's Current Limitations
FRR Zebra uses struct nexthop to track next hop information. If it is a recursive nexthop, its flags field would be set NEXTHOP_FLAG_RECURSIVE bit and its resolved field stores a pointer which points a list of nexthops obtained by recursive resolution. Therefore Zebra keeps hierarchical relationships on the recursive nexthops. 

Because the Linux kernel lacks support for recursive routes, FRR Zebra flattens the next-hop information of recursive routes when transferring it from Zebra to FPM or the Linux kernel. Currently, when a path goes down, Zebra would inform various protocol processes and let them replay routes update events accordingly. 

This leads an issue discussed in the SONiC Routing Working Group (https://lists.sonicfoundation.dev/g/sonic-wg-routing/files/SRv6%20use%20case%20-%20Routing%20WG.pptx).

<figure align=center>
    <img src="images/srv6_igp2bgp.jpg" >
    <figcaption>Figure 1. Alibaba issue Underlay routes flap affecting Overlay SRv6 routes <figcaption>
</figure> 

To solve this issue, we need to introduce Prefix Independent Convergence (PIC) to FRR/SONiC. PIC concept is described in IEFT https://datatracker.ietf.org/doc/draft-ietf-rtgwg-bgp-pic/. It is not a BGP feature, but a RIB/FIB feeature on the device. PIC has two basic concept, PIC core and PIC edge. The following HLD focuses on PIC edge's enhancement https://datatracker.ietf.org/doc/draft-ietf-rtgwg-bgp-pic/. This HLD is outline an approach which could prevent BGP load balancing updates from being triggered by IGP load balancing updates, a.k.a PIC core approach for the recursive VPN route support. 

Note: 
- This HLD only focus on recursive VPN routes support. Since SONiC doesn't have MPLS VPN support in master, the testing would focus on EVPN and SRv6 VPN only. 
- The similar approach could be applied to global table's recursive routes support. But that requires SAI APIs. Therefore, global table's recursive routes support is not in the scope of this HLD.(?? TODO It is per-vrf recursive enhancement, so the global will also benefit from this enhancement ??)

## Triggers Events
Here are a list of trigger events which we want to take care for getting faster routes convergence and minimizing hardware traffic loss. 

| Trigger Types |     Events    |       Possible handling          | 
|:---|:-----------|:----------------------|
| IGP local failure | A local link goes down | From RIB point of view, local interface routes would be removed. From this event, zebra_rib_evaluate_rn_nexthops() would be triggered. It is the PIC core handling case. This path has the same functionality as current orchagent quick fix up approach. |
| IGP remote failture | A remote link goes down, IGP leaf's reachability is not changed, only IGP paths are updated. | IGP updates IGP leaf's NHG. No need to trigger BGP update since reachability is not changed. The handling would via zebra_rib_evaluate_rn_nexthops(). It is the PIC core handling case. |
| IGP remote failure  | A remote IGP node failure or a remote IGP node is unreachable. But the remote PE route could be re-resolved via a new IGP path | IGP triggers IGP leaf delete event, which triggers zebra_rib_evaluate_rn_nexthops(). Since remote PE is still reachable, it is the PIC core handling case. |
| IGP remote failure  | A remote PE node failure or a remote PE node is unreachable | IGP triggers IGP leaf delete event, which triggers zebra_rib_evaluate_rn_nexthops(). This is the PIC edge handling case. |
| BGP remote failure  | BGP remote node down | It should be detected by IGP remote node down first before BGP reacts, a.k.a the same as the above steps. This is the PIC edge handling case |
| BGP remote failure | Remote BGP does not response, remote PE is still available. | BGP will trigger leaf updates. It is a BGP bug situation in deployment and handled via BGP convergence. It is not in PIC's scope |
| BGP local failure | local BGP does not response.| It is a BGP bug situation in deployment and handled via BGP convergence. It is not in PIC's scope |


## FRR Current Approaches
### NH Dependency Tree
struct nexthop contains two fields, *resolved and *reparent for tracking nexthop resolution's dependencies. 

	/* Nexthops obtained by recursive resolution.
	 *
	 * If the nexthop struct needs to be resolved recursively,
	 * NEXTHOP_FLAG_RECURSIVE will be set in flags and the nexthops
	 * obtained by recursive resolution will be added to `resolved'.
	 */
	struct nexthop *resolved;
	/* Recursive parent */
	struct nexthop *rparent;

https://github.com/FRRouting/frr/blob/858cc75b434344ae0b25eccaf6eef03debe4a031/lib/nexthop.h#L99C1-L105C26

When a routing entry is processed by rib_process(), it calls nexthop_active_update() to parse and refresh the nexthop active state. By nexthop_set_resolved(), *resolved is set to the nexthop of the route used to resolve this nexthop, *rparent will also be correspondingly set and the flag of this nexthop is set to NEXTHOP_FLAG_RECURSIVE.

### NHT List from Route Node
Each route node (struct rib_dest_t ) contains a nht field which lists out all NHT prefixes which depend on this route node. 

	/*
	 * The list of nht prefixes that have ended up
	 * depending on this route node.
	 * After route processing is returned from
	 * the data plane we will run evaluate_rnh
	 * on these prefixes.
	 */
	struct rnh_list_head nht;

nht is updated by zebra_rnh_store_in_routing_table() and zebra_rnh_remove_from_routing_table().

### NHG Update Trigger

When Zebra triggers NHG update

- Zebra rib process
- Dplane process complete
- Ddplane route notify is received
- Zfpm route updates

NHG update is carried out during replay of routes updating and zebra_rib_evaluate_rn_nexthops() can be seen as the entry point for this process. It starts from the incoming route node and retrieves its NHT list. Then it iterates through each prefix in the NHT list, utilizing the prefix to invoke zebra_evaluate_rnh().

1. Identify the new route entry to resolve nexthops in the NHT list.
2. Compare the new route entry with the old one, update the nexthop resolving state as the new route entry, and then send a nexthop change notification to protocol clients.
3. Protocol clients recalculate the path associated with the nexthop, then resend the route to Zebra.
4. Pass the route to rib_process().
5. The route's nexthop is recursively resolved, and the recursive one will be flattened.
6. Call zebra_rib_evaluate_rn_nexthops(), then go to step 1. This loop procedure builds/refreshes the recursive NHG chain.

## High Level Design
The main enhancements are in the following areas

- Zebra includes two enhancements for the recursive route. The first is to recalculate the route on route add/update independently, without relying on the protocol client for route updating. The second is to optimize the convergence logic of recursive NHG in the case of route withdrawal.
- Fpm needs to add a new schema to take each member as NHG id and update APP DB.
- Orchagent picks up event from APP DB and trigger NHG programming. Neighorch needs to handle this new schema without change too much on existing codes.

## Low Level Design

### Routes Updating
Consider the case of recursive routes for EVPN underlay

    B>  2.2.2.2/32 [200/0] (127) via 100.0.0.1 (recursive), weight 1, 00:00:02
      *                            via 10.1.0.65, Ethernet1, weight 1, 00:00:02
      *                            via 10.1.0.66, Ethernet2, weight 1, 00:00:02
      *                            via 10.1.0.67, Ethernet3, weight 1, 00:00:02
                                 via 200.0.0.1 (recursive), weight 1, 00:00:02
      *                            via 10.1.0.76, Ethernet4, weight 1, 00:00:02
      *                            via 10.1.0.77, Ethernet5, weight 1, 00:00:02
      *                            via 10.1.0.78, Ethernet6, weight 1, 00:00:02
    B>* 100.0.0.0/24 [200/0] (123) via 10.1.0.65, Ethernet1, weight 1, 00:00:02
      *                            via 10.1.0.66, Ethernet2, weight 1, 00:00:02
      *                            via 10.1.0.67, Ethernet3, weight 1, 00:00:02
    B>* 200.0.0.0/24 [200/0] (108) via 10.1.0.76, Ethernet4, weight 1, 00:00:53
      *                            via 10.1.0.77, Ethernet5, weight 1, 00:00:53
      *                            via 10.1.0.78, Ethernet6, weight 1, 00:00:53

As described in the above section, if node 10.1.0.67 for prefix 100.0.0.0/24 is gone, Zebra will explicitly update both routes for recursive convergence with the help of the BGP client, one for the prefix 100.0.0.0/24 and another for the prefix 2.2.2.2/32.

In this scenario, since the reachability of the prefix 2.2.2.2 remains unchanged and also Zebra has the dependency relationships between recursive NHGs, there is a chance to improve Zebra for route convergence by itself.

#### Data Structure Modifications
In order to enable Zebra to update routes without notifying protocol clients, it should be able to obtain the route node associated with the NHG that has undergone changes. Some back pointer fields need to be added. (?? TODO do we really need it??)

<figure align=center>
    <img src="images/data_struct.jpg" >
    <figcaption>Figure 2. data structure modification for routes update<figcaption>
</figure>

##### struct nhg_hash_entry 
New field struct list *routes in struct nhg_hash_entry

    struct nhg_hash_entry {
        ...

        struct nhg_connected_tree_head nhg_depends, nhg_dependents;

        /* List of routes for this nhe. */
        struct list *routes;

        ...
    }

##### struct route_entry
New field struct list *routes in struct route_entry

    struct route_entry {       
        ...

        /* dest referring to this re */
        struct rib_dest_t_ *pdest;

        ...
    }

Functions initialize the backwalk pointers.
``` c
static void rib_link(struct route_node *rn, struct route_entry *re, int process)
{
    rib_dest_t *dest;
    afi_t afi;
    const char *rmap_name;

    ...

    re_list_add_head(&dest->routes, re);
    re->pdest = dest;

    ...	
}
```

``` c
int route_entry_update_nhe(struct route_entry *re, struct nhg_hash_entry *new_nhghe)
{
    struct nhg_hash_entry *old;
    int ret = 0;

    if (new_nhghe == NULL) {
        if (re->nhe) {
            if (re->nhe->routes)
                listnode_delete(re->nhe->routes, re);
                zebra_nhg_decrement_ref(re->nhe);
            }
            re->nhe = NULL;
            goto done;
    }

    if ((re->nhe_id != 0) && re->nhe && (re->nhe != new_nhghe)) {
        old = re->nhe;

        route_entry_attach_ref(re, new_nhghe);
        if (!new_nhghe->routes)
            new_nhghe->routes = list_new();
        listnode_add(new_nhghe->routes, re);
        if (old) {
            if (old->routes)
                listnode_delete(old->routes, re);
            zebra_nhg_decrement_ref(old);
        }
    } else if (!re->nhe) {
        /* This is the first time it's being attached */
        route_entry_attach_ref(re, new_nhghe);
        if (!new_nhghe->routes)
            new_nhghe->routes = list_new();
        listnode_add(new_nhghe->routes, re);
    }
done:
    return ret;
}
```

##### struct hash *nhgs_ctx_hash
zebra_rib_evaluate_rn_nexthops() triggers routes updating through NHG backwalk. Without the assistance of protocol clients, a method needs to be introduced for looking up NHG based on the prefix of the NHT list. e.g. Finding NHG based on the prefix 100.0.0.1.

A new hash table *nhgs_ctx_hash is for this task.

    struct zebra_router {
        ...

        /* The hash of nexthop groups ctx associated with this router */
        struct hash *nhgs_ctx_hash;

        ...
    }

The hash stores the mapping information from the NHT prefix to the corresponding recursive NHG. zebra_nhg_insert_nhe_ctx() is invoked each time a new NHG is created in zebra_nhe_find(). Only the recursive one is inserted into the hash.

``` C
static bool zebra_nhe_find(struct nhg_hash_entry **nhe, /* return value */
               struct nhg_hash_entry *lookup,
               struct nhg_connected_tree_head *nhg_depends,
               afi_t afi, bool from_dplane)
{
    bool created = false;
    bool recursive = false;
    struct nhg_hash_entry *newnhe, *backup_nhe;
    struct nexthop *nh = NULL;

    ...

done:
    /* Reset time since last update */
    (*nhe)->uptime = monotime(NULL);

    if (created)
        zebra_nhg_insert_nhe_ctx(newnhe);

    return created;
}
```

``` c
static int zebra_nhg_insert_nhe_ctx(struct nhg_hash_entry *nhe)
{
    struct nhe_ctx lookup;
    struct nhe_ctx *exist = NULL;

    if (!nhe || !nhe->id)
        return -1;

    if (nhe->nhg.nexthop->next || (nhe->nhg.nexthop->type != NEXTHOP_TYPE_IPV4
        && nhe->nhg.nexthop->type != NEXTHOP_TYPE_IPV6)) {
        if (IS_ZEBRA_DEBUG_NHG)
            zlog_debug("%s: Skip NHG id (%u), vrf %s",
                __func__, nhe->id, vrf_id_to_name(nhe->nhg.nexthop->vrf_id));
        return -1;
    }

    lookup.nhe_id = nhe->id;
    lookup.vrf_id = nhe->nhg.nexthop->vrf_id;
    lookup.nh_type = nhe->nhg.nexthop->type;
    lookup.gate = nhe->nhg.nexthop->gate;

    exist = hash_lookup(zrouter.nhgs_ctx_hash, &lookup);

    if (exist && exist->nhe_id != nhe->id) {
        if (IS_ZEBRA_DEBUG_NHG)
            zlog_debug("%s: NHG id (%u), vrf %s => NHG id (%u), vrf %s",
                __func__, exist->nhe_id, vrf_id_to_name(exist->vrf_id),
                nhe->id, vrf_id_to_name(exist->vrf_id));

        exist->nhe_id = nhe->id;
        return 0;
    }

    hash_get(zrouter.nhgs_ctx_hash, &lookup, zebra_nhg_ctx_hash_alloc);

    ...
}
```

#### Routes Updating Handling
The newly added zebra_rnh_refresh_dependents() handles the routes updating, replacing the protocol client's notification. It will be detailed in the following sections.

<figure align=center>
    <img src="images/backwalk_functions.jpg" >
    <figcaption>Figure 3. routes updating function<figcaption>
</figure>

The replay of routes updating starts when zebra_rib_evaluate_rn_nexthops() function is called and should be stopped when the route node's NHT list is empty. In other words, there are no nexthops resolving depending on this route node. It retains the original approach of Zebra for updating the resolve state of each route, so the handling will continue until prefix 2.2.2.2. However, at dplane/fpm level, there is no need to refresh the recursive NHG for prefix 2.2.2.2 again, since the reachability of it hasn't changed, and the ID of this recursive NHG should remain unchanged.

<figure align=center>
    <img src="images/nhg_id_change.jpg" >
    <figcaption>Figure 4. NHG ID change for route convergence<figcaption>
</figure>

To maintain the NHG ID unchanged during recursive NHG, refer to the zebra_rnh_refresh_dependents section for the details.

### Fast Convergence for Route Withdrawal
As the case of recursive routes for EVPN underlay

    B>  2.2.2.2/32 [200/0] (127) via 100.0.0.1 (recursive), weight 1, 00:00:02
      *                            via 10.1.0.65, Ethernet1, weight 1, 00:00:02
      *                            via 10.1.0.66, Ethernet2, weight 1, 00:00:02
      *                            via 10.1.0.67, Ethernet3, weight 1, 00:00:02
                                 via 200.0.0.1 (recursive), weight 1, 00:00:02
      *                            via 10.1.0.76, Ethernet4, weight 1, 00:00:02
      *                            via 10.1.0.77, Ethernet5, weight 1, 00:00:02
      *                            via 10.1.0.78, Ethernet6, weight 1, 00:00:02
    B>* 100.0.0.0/24 [200/0] (123) via 10.1.0.65, Ethernet1, weight 1, 00:00:02
      *                            via 10.1.0.66, Ethernet2, weight 1, 00:00:02
      *                            via 10.1.0.67, Ethernet3, weight 1, 00:00:02
    B>* 200.0.0.0/24 [200/0] (108) via 10.1.0.76, Ethernet4, weight 1, 00:00:53
      *                            via 10.1.0.77, Ethernet5, weight 1, 00:00:53
      *                            via 10.1.0.78, Ethernet6, weight 1, 00:00:53

If the local interface Ethernet6 is down or the route "200.0.0.0/24 via 10.1.0.78, Ethernet6" receives an explicit withdrawal from the IGP node.

<figure align=center>
    <img src="images/route_delete.jpg" >
    <figcaption>Figure 5. rib deletion<figcaption>
</figure>

Rib deletion for interface down or route withdrawal is handled in rib_process(), then zebra_rnh_refresh_dependents() also handles route withdrawal case.

#### Data Structure Modifications
No Zebra original data structure modification is required as it leverages Zebra's NHG dependents chain.

#### Fast Convergence Handling
Fast convergence for route withdrawal is also handled in the zebra_rnh_refresh_dependents(). The detailed is in the next section.

### The Handling of zebra_rnh_refresh_dependents()

This new function is inserted into the existing route convergence process, allowing Zebra to autonomously achieve route convergence in the case where the reachability of recursive routes remains unchanged.

Provide a brief description of Zebra's original recursive convergence process.

<figure align=center>
    <img src="images/route_converge_original.jpg" >
    <figcaption>Figure 6. route convergence process<figcaption>
</figure>

Route/Nexthop dependents are built or refreshed from the bottom up with each invocation of zebra_rnh_eval_nexthop_entry().

After the insertion of zebra_rnh_refresh_dependents into the original recursive convergence process.

<figure align=center>
    <img src="images/zebra_rnh_refresh_dependents.jpg" >
    <figcaption>Figure 7. zebra_rnh_refresh_dependents()<figcaption>
</figure>

The route convergence logic in the red will be replaced by the blue section.

### Dataplane refresh for Nexthop group change
As the recursive NHG ID remains unchanged, Zebra is able to bypass forwarding this route to Dplane/FPM. In other words, the backwalk in Dplane/FPM terminates at the recursive NHG route.
Data Structure Modifications

TODO: Add a status flag ROUTE_ENTRY_NHG_ID_PRESERVED in struct route_entry? rib_process_update_fib() skip the routes with this flag?

### FPM's new schema for recursive NHG
We rely on BRCM and NTT's NHG changes.

### Orchagent changes
We rely on BRCM and NTT's NHG changes.

## Unit Test
### Normal Case's Forwarding Chain Information
### Test Case 1: local link failure
<figure align=center>
    <img src="images/testcase1.png" >
    <figcaption>Figure 8.local link failure <figcaption>
</figure>

### Test Case 2: IGP remote link/node failure
<figure align=center>
    <img src="images/testcase2.png" >
    <figcaption>Figure 9. IGP remote link/node failure
 <figcaption>
</figure>

### Test Case 3: IGP remote PE failure
<figure align=center>
    <img src="images/testcase3.png" >
    <figcaption>Figure 10. IGP remote PE failure
 <figcaption>
</figure>

### Test Case 4: BGP remote PE node failure
<figure align=center>
    <img src="images/testcase4.png" >
    <figcaption>Figure 11. BGP remote PE node failure
 <figcaption>
</figure>

### Test Case 5: Remote PE-CE link failure
<figure align=center>
    <img src="images/testcase5.png" >
    <figcaption>Figure 12. Remote PE-CE link failure
 <figcaption>
</figure>


## References
- https://github.com/sonic-net/SONiC/pull/1425
- https://datatracker.ietf.org/doc/draft-ietf-rtgwg-bgp-pic/
- https://github.com/sonic-net/SONiC/blob/master/doc/pic/bgp_pic_arch_doc.md
- https://github.com/eddieruan-alibaba/SONiC/blob/eruan-pic/doc/bgp_pic/bgp_pic.md
