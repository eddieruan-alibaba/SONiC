<!-- omit in toc -->
# Recursive Route Handling HLD
<!-- omit in toc -->
### Revision
| Rev |     Date    |       Author           | Change Description                |
|:---:|:-----------:|:----------------------:|-----------------------------------|
| 0.1 | Oct    2023 |                        | Initial Draft                     |

<!-- omit in toc -->
## Table of Content
- [Goal and Scope](#goal-and-scope)
  - [FRR's Current Limitations](#frrs-current-limitations)
- [FRR Current Approaches](#frr-current-approaches)
  - [NH dependency tree](#nh-dependency-tree)
  - [NHT list from route node](#nht-list-from-route-node)
  - [zebra\_rib\_evaluate\_rn\_nexthops](#zebra_rib_evaluate_rn_nexthops)
  - [Zebra triggers routes redownloading from protocol process](#zebra-triggers-routes-redownloading-from-protocol-process)
- [Triggers Events](#triggers-events)
- [High Level Design](#high-level-design)
- [Low Level Design](#low-level-design)
  - [Recursive nexthop change notification](#recursive-nexthop-change-notification)
  - [Dataplane refresh for Nexthop group change](#dataplane-refresh-for-nexthop-group-change)
  - [Zebra changes](#zebra-changes)
  - [FPM's new schema for recursive NHG](#fpms-new-schema-for-recursive-nhg)
  - [Orchagent changes](#orchagent-changes)
  - [NHG update Handling](#nhg-update-handling)
- [Unit Test](#unit-test)
- [References](#references)

## Goal and Scope
A recursive route is a routing mechanism in which the routing decision for a specific destination is determined by referring to another routing table, which is then looked up recursively until a final route is resolved. Recursive routing is a key concept in routing protocols and is often used in complex network topologies to ensure that data reaches its intended destination, even when that destination is not directly reachable from the originating device. In many cases, recursive routes are used in VPN or tunneling scenarios.

### FRR's Current Limitations
FRR zebra uses struct nexthop to track next hop information. If it is a recursive nexthop, its flags field would be set NEXTHOP_FLAG_RECURSIVE bit and its resolved field stores a pointer which points a list of nexthops obtained by recursive resolution. Therefore zebra keeps hierarchical relationships on the recursive nexthops. 

Because the Linux kernel lacks support for recursive routes, FRR zebra flattens the next-hop information of recursive routes when transferring it from Zebra to FPM or the Linux kernel. Currently, when a path goes down, zebra would inform various protocol processes and let them replay routes update events accordingly. 

This leads an issue discussed in the SONiC Routing Working Group (https://lists.sonicfoundation.dev/g/sonic-wg-routing/files/SRv6%20use%20case%20-%20Routing%20WG.pptx).

<figure align=center>
    <img src="images/srv6_igp2bgp.jpg" >
    <figcaption>Figure 1. Alibaba issue Underlay routes flap affecting Overlay SRv6 routes <figcaption>
</figure> 

The enhancement is discussed in https://datatracker.ietf.org/doc/draft-ietf-rtgwg-bgp-pic/. This HLD is outline an approach which could prevent BGP load balancing updates from being triggered by IGP load balancing updates. This is essentially the recursive VPN route support. 

Note: 
- This HLD only focus on recursive VPN routes support. Since SONiC doesn't have MPLS VPN support in master, the testing would focus on EVPN and SRv6 VPN only. 
- The similar approach could be applied to global table's recursive routes support. But that requires SAI APIs. Therefore, global table's recursive routes support is not in the scope of this HLD.

## FRR Current Approaches
### NH dependency tree
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

When a routing entry (re) is processed by rib_process(), this function calls nexthop_active_update() to parse and refresh this re. The corresponding nexthop for this re is handled by nexthop_set_resolved(), *resolved is set to the nexthop of the route used to resolve this nexthop, *rparent will also be correspondingly set and the flag of this nexthop is set to NEXTHOP_FLAG_RECURSIVE. This is the method by which the recursive nexthop is flattened.

### NHT list from route node
Each route node (struct rib_dest_t ) contains a nht field which lists out all nht prefixes which depend on this route node. 

	/*
	 * The list of nht prefixes that have ended up
	 * depending on this route node.
	 * After route processing is returned from
	 * the data plane we will run evaluate_rnh
	 * on these prefixes.
	 */
	struct rnh_list_head nht;

rib_gc_dest() invokes zebra_rib_evaluate_rn_nexthops() to update the nht.

### zebra_rib_evaluate_rn_nexthops

list https://github.com/FRRouting/frr/blob/master/zebra/zebra_rib.c#L850C1-L856C45

zebra_rib_evaluate_rn_nexthops() leverage the above list to trigger each depending recursive nh to get reevaluated.
It starts from the incoming route node (rn) and traverses upwards through all route nodes until it reaches a route node with a prefix of 0(default route). It obtains the nht list from the corresponding rib_dest_t. Then, it iterates through each prefix in the nht list, searching the routing table again. If it finds a route entry (re) for resolving, it updates rnh->resolved_route to this resolution. Subsequently, it calls zebra_rnh_store_in_routing_table to move the nht to the new rn's nht list corresponding to the resolved re.

TODO: Check how NHG update gets triggered. Couple discussion points
1. How to trigger NHG update
2. How to flatten IGP NHGs, and BGP NHG accordingly. In hardware, we only have two level of NHG budgets. In general, we need to collapse all IGP NHG into one hw NHG and BGP NHG into another. 

### Zebra triggers routes redownloading from protocol process
TODO: How does NH update inform other protocol processes for triggered routes redownloading? Couple discussion points
1. For kernel update, we might need to redownload all routes for flattening NHG.
2. For the address family which skip kernel programming, can we avoid this trigger. 
   
## Triggers Events
Here are a list of trigger events which we want to take care for getting faster routes convergence and minimizing hardware traffic loss. 

| Trigger Types |     Events    |       Possible handling          | 
|:---|:-----------|:----------------------|
| IGP local failure | A local link goes down | Local interface routes would be removed. From this event, zebra_rib_evaluate_rn_nexthops() would be triggered |
| IGP remote failture | A remote link goes down, IGP leaf's reachability is not changed, only IGP paths are updated. | IGP updates IGP leaf's NHG. No need to trigger BGP update since reachability is not changed. The handling would via zebra_rib_evaluate_rn_nexthops() ?? TODO ?? . It is recursive route handling case. |
| IGP remote failure  | A remote IGP node failure or a remote IGP node is unreachable. But the remote PE route could be re-resolved via a new IGP path | IGP triggers IGP leaf delete event, which triggers zebra_rib_evaluate_rn_nexthops(). Since remote PE is still reachable, it is recursive route handling case. |
| IGP remote failure  | A remote PE node failure or a remote PE node is unreachable | IGP triggers IGP leaf delete event, which triggers zebra_rib_evaluate_rn_nexthops(). This is the PIC handling case. |
| BGP remote failure  | BGP remote node down | It should be detected by IGP remote node down first before BGP reacts. This is the PIC handling case |
| BGP remote failure | Remote BGP does not response, remote PE is still available. | BGP will trigger leaf updates. It is a BGP bug situation in deployment and not in this HLD's scope |
| BGP local failure | local BGP does not response.| It is a BGP bug situation in deployment and not in this HLD's scope |


## High Level Design
The main changes are in the following areas

- Zebra would have an option to enable recursive route support. In this mode, zebra would pass both underlay and overlay NHGs to fpm. But zebra will still pass collapsed NHG to Linux kernel. 
- fpm needs to add a new schema to take each member as NHG id and update APP DB.
- orchagent picks up event from APP DB and trigger NHG programming. Neighorch needs to handle this new schema without change too much on existing codes.


## Low Level Design
### Recursive nexthop change notification
Zebra stores nexthop information that depends on a route node of a route entry. Whenever a change occurs in the routes or a new nexthop registers with Zebra, Zebra recalculates the route entries used to resolve the nexthops. It then sends an nexthop change notification to the client that registered this nexthop. In certain situations, when there is a change in the resolving routes upon which intermediate-level recursive nexthops depend, Zebra does not need to trigger a nexthop change notification to be sent to the routing clients. Instead, it may achieve routing convergence by itself to expedite the efficiency of routing calculations.
e.g.:

nexthop A --> nexthop B --> nexthop C1,C2,C3

A and B are recursive nexthops. If the route corresponding to C3 is withdrawn and Zebra can perform routing convergence calculations on its own, and if B is still active, there is no need to send a notification to Zebra clients about the change in the nexthop B.
		       
### Dataplane refresh for Nexthop group change
TODO:
### Zebra changes
To enable the zebra to backwalk during zebra_rib_evaluate_rn_nexthops(), it is necessary to add a reverse reference in nhe pointing to the re pointer that references it. Additionally, in re, a reverse reference pointing to rib_dest_t should also be added.

A new function will be introduced that, when called, triggers a backwalk. This is necessary due to changes in routing, requiring the reevaluation and update of routes, as well as the update of the RIB.

### FPM's new schema for recursive NHG
TODO

### Orchagent changes
TODO

### NHG update Handling
TODO

## Unit Test
TODO

## References
- https://github.com/sonic-net/SONiC/pull/1425
- https://github.com/eddieruan-alibaba/SONiC/blob/eruan-pic/doc/bgp_pic/bgp_pic.md
