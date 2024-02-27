<!-- omit in toc -->
# Recursive Route Handling HLD
<!-- omit in toc -->
## Revision
| Rev |     Date    |       Author           | Change Description                |
|:---:|:-----------:|:----------------------:|-----------------------------------|
| 0.1 | Oct    2023 |                        | Initial Draft                     |

<!-- omit in toc -->
## Table of Content
- [Scope](#scope)
- [Requirements Overview](#requirements-overview)
- [Zebra Current Approach for Recursive Routes](#zebra-current-approach-for-recursive-routes)
  - [Data Structure for Recursive Handling](#data-structure-for-recursive-handling)
    - [NH Dependency Tree](#nh-dependency-tree)
    - [NHT List from Route Node](#nht-list-from-route-node)
  - [Recursive Route Handling](#recursive-route-handling)
- [High Level Design](#high-level-design)
  - [Triggers Events for Recursive Handling](#triggers-events-for-recursive-handling)
  - [Fast Convergence for Recursive Routes](#fast-convergence-for-recursive-routes)
    - [Data Structure Modifications](#data-structure-modifications)
      - [struct nhg\_hash\_entry](#struct-nhg_hash_entry)
      - [struct route\_entry](#struct-route_entry)
      - [struct rnh](#struct-rnh)
    - [The Handling of zebra\_rnh\_refresh\_dependents()](#the-handling-of-zebra_rnh_refresh_dependents)
  - [Nexthop Group Preserving](#nexthop-group-preserving)
    - [Data Structure Modifications](#data-structure-modifications-1)
    - [The Handling of nexthop\_active\_update()](#the-handling-of-nexthop_active_update)
  - [Special Considerations for EVPN Overlay Routes](#special-considerations-for-evpn-overlay-routes)
  - [Dataplane Refresh for Recursive route](#dataplane-refresh-for-recursive-route)
  - [FPM's new schema for recursive nexthop group](#fpms-new-schema-for-recursive-nexthop-group)
  - [Orchagent changes](#orchagent-changes)
- [Unit Test](#unit-test)
  - [Normal Case's Forwarding Chain Information](#normal-cases-forwarding-chain-information)
  - [Test Case 1: local link failure](#test-case-1-local-link-failure)
  - [Test Case 2: IGP remote link/node failure](#test-case-2-igp-remote-linknode-failure)
  - [Test Case 3: IGP remote PE failure](#test-case-3-igp-remote-pe-failure)
  - [Test Case 4: BGP remote PE node failure](#test-case-4-bgp-remote-pe-node-failure)
  - [Test Case 5: Remote PE-CE link failure](#test-case-5-remote-pe-ce-link-failure)
- [References](#references)

## Scope
This document is for reducing packet loss when Zebra handles convergence for millions of recursive routes on SONiC. Since SONiC doesn't have MPLS VPN support in master, the testing would focus on EVPN and SRv6 VPN only. 

## Requirements Overview
This HLD introducing a method for fast recovering from packet loss in the event of underlay ECMP path changes. Since Zebra serves as the control plane on SONiC, then corresponding adjustments may be made to FPM and Orchagent to collaborate with it.
- Fpm needs to add a new schema to take each member as nexthop group ID and update APP DB. (Rely on BRCM and NTT's changes)
- Orchagent picks up event from APP DB and trigger nexthop group programming. Neighorch needs to handle this new schema without change too much on existing codes. (Rely on BRCM and NTT's changes)

Because the Linux kernel lacks support for recursive routes, FRR Zebra flattens the nexthop information of recursive routes when transferring it from Zebra to FPM or the Linux kernel. Currently, when a path goes down, Zebra would inform various protocol processes and let them replay routes update events accordingly. This leads an issue discussed in the SONiC Routing Working Group (https://lists.sonicfoundation.dev/g/sonic-wg-routing/files/SRv6%20use%20case%20-%20Routing%20WG.pptx).

<figure align=center>
    <img src="images/srv6_igp2bgp.png" >
    <figcaption>Figure 1. Alibaba issue Underlay routes flap affecting Overlay SRv6 routes <figcaption>
</figure>

In order to solve the above routes flap issue, some enhancements of Zebra are necessary.

## Zebra Current Approach for Recursive Routes
### Data Structure for Recursive Handling
#### NH Dependency Tree
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

In Zebra, when a routing entry is processed by rib_process() function, it calls nexthop_active_update() to parse and refresh the nexthop resolving state. Then, *resolved is set to the nexthop of the route used to resolve this nexthop, and the flag NEXTHOP_FLAG_RECURSIVE is set to the nexthop to indicate its recursive state.

#### NHT List from Route Node
Each route node (struct rib_dest_t) contains an nht field, which stores all nexthop prefixes that depend on this route node.

	/*
	 * The list of nht prefixes that have ended up
	 * depending on this route node.
	 * After route processing is returned from
	 * the data plane we will run evaluate_rnh
	 * on these prefixes.
	 */
	struct rnh_list_head nht;

This list is updated when a new route is added or a nexthop is registered by the protocol clients.

### Recursive Route Handling
A brief description of Zebra's current recursive convergence process below

<figure align=center>
    <img src="images/route_converge_original.png" >
    <figcaption>Figure 2. route convergence process<figcaption>
</figure>

Recursive route handling is carried out during the replay of route updates, and zebra_rib_evaluate_rn_nexthops() can be seen as the entry point for this process. It starts from the incoming route node and retrieves its NHT list. Then, it iterates through each nexthop(prefix) in the NHT list, utilizing the prefix to invoke zebra_evaluate_rnh(). This function works as follows:

1. identify the new route entry to resolve the nexthop
2. compare the new route entry with the previous one, if they are not same, update the nexthop resolving state as the new route entry, and then send a nexthop change notification to protocol clients
3. protocol clients recalculate the path associated with the nexthop, then resend the corresponding route to Zebra.
4. Zebra processes this route and the route's nexthop is recursively resolved and also flattened
6. at the end of route updating, zebra_rib_evaluate_rn_nexthops() is called with the route's NHT list, and then it returns to step 1. This loop procedure contributes to recursive route convergence

## High Level Design

### Triggers Events for Recursive Handling
Here are a list of trigger events which we want to take care for getting recursive route convergence and minimizing hardware traffic loss. 

| Trigger Types |     Events    |       Possible handling          | 
|:---|:-----------|:----------------------|
| Case 1: IGP local failure | A local link goes down | Currently Orchagent handles local link down event and triggers a quick fixup which removes the failed path in HW ECMP. Zebra will be triggered from connected_down() handling. BGP may be informed to install a backup path if needed. This is a special PIC core case, a.k.a PIC local |
| Case 2: IGP remote link/node failture  | A remote link goes down, IGP leaf's reachability is not changed, only IGP paths are updated. | IGP gets route withdraw events from IGP peer. It would inform zebra with updated paths. Zebra would be triggered from zread_route_add() with updated path list. It is the PIC core handling case. |
| Case 3: IGP remote PE failure  | A remote PE node is unreachable in IGP domain. | IGP triggers IGP leaf delete event. Zebra will be triggered from zread_route_del(). It is the PIC edge handling case |
| Case 4: BGP remote PE node failure  | BGP remote node down | It should be detected by IGP remote node down first before BGP reacts, a.k.a the same as the above steps. This is the PIC edge handling case.|
| Case 5: Remote PE-CE link failure | This is remote PE's PIC local case.  | Remote PE will trigger PIC local handling for quick traffic fix up. Local PE will be updated after BGP gets informed. |

### Fast Convergence for Recursive Routes
Consider the case of recursive routes for EVPN underlay

    B>  2.2.2.2/32 [200/0] via 100.0.0.1 (recursive), weight 1, 00:11:50
      *                      via 10.1.0.16, Ethernet1, weight 1, 00:11:50
      *                      via 10.1.0.17, Ethernet2, weight 1, 00:11:50
      *                      via 10.1.0.18, Ethernet3, weight 1, 00:11:50
                           via 200.0.0.1 (recursive), weight 1, 00:11:50
      *                      via 10.1.0.26, Ethernet4, weight 1, 00:11:50
      *                      via 10.1.0.27, Ethernet5, weight 1, 00:11:50
      *                      via 10.1.0.28, Ethernet6, weight 1, 00:11:50
    B>* 100.0.0.0/24 [200/0] via 10.1.0.16, Ethernet1, weight 1, 00:11:57
      *                      via 10.1.0.17, Ethernet2, weight 1, 00:11:57
      *                      via 10.1.0.18, Ethernet3, weight 1, 00:11:57
    B>* 200.0.0.0/24 [200/0] via 10.1.0.26, Ethernet4, weight 1, 00:11:50
      *                      via 10.1.0.27, Ethernet5, weight 1, 00:11:50
      *                      via 10.1.0.28, Ethernet6, weight 1, 00:11:50

If the path 10.1.0.28 of prefix 200.0.0.0/24 is removed, Zebra will explicitly update both routes for recursive convergence with the help of the BGP client, one for 200.0.0.0/24 and another for 2.2.2.2/32. In this scenario, although the route 200.0.0.0/24 has one path removed, but its reachability for route 2.2.2.2/32 remains unchanged. Zebra has the dependency relationships between these recursive routes, so there is a chance to improve Zebra for route convergence.

<figure align=center>
    <img src="images/path_remove1.png" >
    <figcaption>Figure 3. path remove for recursive route<figcaption>
</figure>

#### Data Structure Modifications
In order to enable Zebra to replay recursive routes without notifying protocol clients, it should be able to obtain the route node associated with the nexthop that has undergone changes. Some back pointer fields need to be added.

<figure align=center>
    <img src="images/data_struct.png" >
    <figcaption>Figure 4. data structure modification for routes update<figcaption>
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

##### struct rnh
zebra_rib_evaluate_rn_nexthops() triggers routes updating through nexthop backwalk. Without the assistance of protocol clients, a method needs to be introduced for looking up nexthop hash struct(nhe) based on the prefix of the NHT list. e.g. Finding the nhe based on the prefix 100.0.0.1.

A new field nhe_id is added for this purpose.

    struct rnh {
        ...

        /* nhe id currently associated */
        uint32_t nhe_id;

        ...
    }

This field provides information about which nhe is associated with the NHT prefix.

nhg_id_rnh_add() is used to set this field and it is invoked each time a new singleton nhe is created in zebra_nhe_find().
``` c
static void nhg_id_rnh_add(struct nhg_hash_entry *nhe)
```
``` c
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
        nhg_id_rnh_add(*nhe);

    return created;
}
```

#### The Handling of zebra_rnh_refresh_dependents()

This new added function is inserted into the existing route convergence process, allowing Zebra to achieve route convergence in the case where the reachability of recursive routes remains unchanged.

<figure align=center>
    <img src="images/zebra_rnh_refresh_dependents.png" >
    <figcaption>Figure 5. zebra_rnh_refresh_dependents()<figcaption>
</figure>

The logic in the blue portion of the code serves fast route convergence updating. It won't completely replace the protocol client's notification for route updating in the following scenarios:
- The reachability of routes that the NHT list depends on changes, i.e., when the routes it depends on change completely to another one
- The routes that depends on a recursive route still using the original client's notification procedure

zebra_rnh_refresh_dependents() is called as follows:

``` c
static void zebra_rnh_eval_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
					 int force, struct route_node *nrn,
					 struct rnh *rnh,
					 struct route_node *prn,
					 struct route_entry *re)
{
    ...
    zebra_rnh_remove_from_routing_table(rnh);
    if (!prefix_same(&rnh->resolved_route, prn ? &prn->p : NULL)) {
        if (prn)
            prefix_copy(&rnh->resolved_route, &prn->p);
        else {
                /*
                * Just quickly store the family of the resolved
                * route so that we can reset it in a second here
                */
                int family = rnh->resolved_route.family;

                memset(&rnh->resolved_route, 0, sizeof(struct prefix));
                rnh->resolved_route.family = family;
        }

        copy_state(rnh, re, nrn);
        state_changed = 1;
    } else if (compare_state(re, rnh->state)) {
        if (re->nhe->id != rnh->state->nhe->id)
            zebra_rnh_refresh_dependents(rnh);
        else
            state_changed = 1;
        copy_state(rnh, re, nrn);
    }
    zebra_rnh_store_in_routing_table(rnh);
    ...
}
```

Explanation of the functions above:

1. rib_process() eventually calls zebra_rnh_eval_nexthop_entry() after finishing one route updating task
2. zebra_rnh_eval_nexthop_entry() see if a tracked nexthop has resolution changed, if it is true, the NHG of new resolving route is checked against the previous one, if they are not equal, zebra_rnh_refresh_dependents() is invoked, otherwise, the original protocol client notify mechanism is used
3. zebra_rnh_refresh_dependents() finds the corresponding nexthop group (nhe), then uses this nhe as parameter for zebra_rnh_eval_dependents()
4. zebra_rnh_eval_dependents() walks back and find the routes depends on the nhe, then requeue the routes to working queue for rib_process() again. A new added flag ROUTE_ENTRY_NHG_ID_PRESERVED is set for this route, to indicate that its recursive reachability is unchanged
5. The dataplane performs a fast NHG refresh if the route is flagged with ROUTE_ENTRY_NHG_ID_PRESERVED

### Nexthop Group Preserving
As previous section, once a route has some path changes, recursive route updating will proceed along the reverse path of dependency. By the original approach of the routes updating, all nexthop on that direction will be recreated. As shown in the diagram, when the path 10.0.1.28 is removed, all dependent nexthops originating from it will be recreated, as indicated by the red text in the diagram.

<figure align=center>
    <img src="images/nhg_change1.png" >
    <figcaption>Figure 6. nexthop dependents change<figcaption>
</figure>

However, for the view of the reachability of nexthop 73 (for route 2.2.2.2/32), there is no need to recreate it for recursive route updating again, since the reachability hasn't changed. After introducing the "Nexthop Group Preserving" enhancement, the desired goal is as illustrated in the following diagram.

<figure align=center>
    <img src="images/nhg_change2.png" >
    <figcaption>Figure 7. NHG dependents preserved<figcaption>
</figure>

The nexthop ID remains unchanged, facilitating a fast refresh by the dataplane.

#### Data Structure Modifications
``` c
/* The nexthop group id should remain unchanged during resolving */
#define ROUTE_ENTRY_NHG_ID_PRESERVED      0x80
```
This flag for struct route_entry indicates that the nexthop group shouldn't change during route's recursive resolving, it also implies that the route with this flag only has some nexthop path change, but the reachability of it remains same.

#### The Handling of nexthop_active_update()
The modification made to nexthop_active_update() preserves the associated nexthop group of routes with the ROUTE_ENTRY_NHG_ID_PRESERVED flag set during recursive route resolution. It recursively resolves them in place. Once the resolution is complete, the nexthop group resolution is refreshed, and no new nexthop groups are created.

### Special Considerations for EVPN Overlay Routes
TODO:

### Dataplane Refresh for Recursive route
As the recursive nexthop ID remains unchanged, Zebra is able to skip reinstall the routes again and only update the NHG to dataplane for fast dataplane refresh.

### FPM's new schema for recursive nexthop group
We rely on BRCM and NTT's changes.

### Orchagent changes
We rely on BRCM and NTT's changes.

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
