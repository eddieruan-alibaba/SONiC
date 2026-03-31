<h1 align="center">RIB FIB Route Convergence Handling LLD </h1>

# Table of Contents <!-- omit in toc -->

# Problem Statements
In the current SONiC architecture, orchagent handles local port down events by rapidly updating failed load balance members to minimize the traffic loss window. However, in other failure scenarios, FRR creates a new Next Hop Group (NHG) and migrates prefixes sequentially. Consequently, the traffic loss window is proportional to the number of prefixes.

To address this, we propose a Prefix Independent Convergence (PIC) design. This mechanism triggers a process upon receiving a Next Hop Tracking (NHT) update event from zebra to fpmsyncd. fpmsyncd will initiate a backwalk from the flagged NHG to all dependent NHGs, allowing for a rapid fixup of the affected NHGs.

This Low-Level Design (LLD) focuses on implementing the backwalk infrastructure within the NHG Manager. The following prompt is designed to instruct an LLM to generate the necessary code for this implementation.

The input information from this NHT event is
* Nexthop address : used in walk spec to decide is the walked NHG is relevent or for SONiC NHG table walk, a.k.a PIC edge case.
* Its current resolved NHG ID : this NHG id is used to start backwalk, a.k.a PIC core case.

# Existing NHG MGR codes
Here are current  nhg mgr 's  codes

https://github.com/eddieruan-alibaba/sonic-swss/blob/rib_fib/fpmsyncd/nhgmgr.cpp
https://github.com/eddieruan-alibaba/sonic-swss/blob/rib_fib/fpmsyncd/nhgmgr.h

# fib_nhg_trigger_node_quick_fixup()
This function is invoked upon receiving a Next Hop Tracking (NHT) event sent from Zebra to fpmsyncd. It accepts two input parameters:
* Nexthop Address: The specific IPv4 or IPv6 address associated with the event.
* Resolved NHG ID: The resolved Next Hop Group ID for the nexthop address
* Origial NHG ID: The original Next Hop Group ID for the nexthop address

## Functionality:
This function initializes the fib_nhg_walking_ctx with the following configuration:
* Visited Set: An empty visited_node_set.
* Walk Specification: Uses fib_nhg_walk_spec_for_node_quick_fixup.
* Prune Specification: Uses fib_nhg_prune_spec_for_node_quick_fixup.
* Target: The nexthop address indicating the impacted resource.

Finally, it initiates the traversal by calling fib_nhg_back_walk() via Resolved NHG ID.

Currently, we don't use original NHG ID to initiate backwalk. This could be added in the future with proper use cases.

# fib_nhg_back_walk()
This method is added to the RIBNHGTable class to trigger a backwalk across all maintained RIBNHGEntry objects.

## Parameters:
* uint32_t id: The Zebra NHG ID, used as a key to retrieve the RIBNHGEntry via getEntry(uint32_t id).
* fib_nhg_walking_ctx: A struct containing:
    * visited_node_set: A set of visited node IDs.
    * fib_nhg_walk_spec_func: A callback function to apply necessary updates to the current node.
    * fib_nhg_prune_spec_func: A callback function to determine whether the backwalk should continue from the current node.
    * nexthop_address: Indicates the impacted nexthop.

## Usage:
While currently implemented for NHG quick fixup, this infrastructure is designed to be extensible for other purposes.

## Main Logic:
1. Retrieve the RIBNHGEntry using the incoming ID (getEntry(uint32_t id)).
2. Add the current node ID to the visited_node_set.
3. Execute the walk spec function on the found RIBNHGEntry.
    * This may trigger updates to the object (e.g., disabling failed members).
4. Execute the prune spec function to decide whether to continue the backwalk. This prun spec function gets the walk spec function return value and this RIBNHGEntry object as input.
5. If the backwalk continues:
    * Retrieve the dependency list of the current RIBNHGEntry.
    * Filter out any nodes already present in visited_node_set.
    * Recursively call fib_nhg_back_walk() on each remaining node in the dependency list.

# Changes in  class RIBNHGEntry
## New Field: m_resolved_enable_group
Need to add a new field m_resolved_enable_group to track if resovled NHG is enabled or not.

```
        /*
         * Resolved group of the entry.
         * Contains <ribID, bool> pairs to indicate if the resolved NHG is enabled.
         * - All paths are set as enabled upon Add/Update events from FRR.
         * - Paths are marked as disabled during the backwalk process.
         */
        unordered_map<uint32_t, bool> m_resolved_enable_group;
```

## Method Update: RIBNHGEntry::getNextHopGroupFields()
Added a validation check: if a path is marked as disabled in m_resolved_enable_group, its information is excluded from the output strings.

# fib_nhg_walk_spec_for_node_quick_fixup
This is a specific implementation of fib_nhg_walk_spec_func. The caller defines which spec function is utilized before initiating the backwalk; all nodes in a single traversal use the same spec function.

* Input: The current RIBNHGEntry node object.
* Output: A boolean indicating success.
    * true: Fixup successful; backwalk could be continued from this node.
    * false: Fixup not required or failed; backwalk stops. Returns false if the NHG is irrelevant to the target nexthop or has already been updated.

## Main Logic:
1. Retrieve the list of node IDs this entry depends on.
2. Determine relevance for update by checking:
    * If the RIBNHGEntry's gateway address (or its dependents' gateway addresses) matches the impacted nexthop.
    * If the dependents' m_resolved_enable_group contains disabled paths that have not yet been reflected in the current RIBNHGEntry's m_resolved_enable_group.
3. If an update is required, iterate through the dependents' list:
    * Retrieve the dependent RIBNHGEntry via getEntry(uint32_t id).
    * Inspect the dependent's m_resolved_enable_group to identify disabled paths.
    * Mark the corresponding paths in the current RIBNHGEntry's m_resolved_enable_group as disabled.
4. Trigger getNextHopGroupFields() on the current RIBNHGEntry, eventually calling writeToDB() to update APPDB with the remaining enabled paths. This completes the quick fixup for the current entry. Skip APPDB update if it is the only path and need to be disabled.

TODO: Implement a walk mechanism in the SONiC NHG table.

# fib_nhg_prune_spec_for_node_quick_fixup
This is a specific implementation of fib_nhg_prune_spec_func. Like the walk spec, this is defined by the caller prior to traversal.

* Input:
  * The current RIBNHGEntry node object.
  * The return value from walking spec.
* Output: A boolean indicating whether to prune (stop) the backwalk from this node.

## Main Logic:
1. If the node was not updated during the walk spec phase, stop the backwalk (no further propagation needed).
2. If the node represents a SONiC NHG object, stop the backwalk.


# Test cases
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

### Test case Local Failure Simulation
* Scenario 1: The route fc06::2/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address fc06::2 and the corresponding NHG ID.
* Expected Result: All affected Next Hop Groups (NHGs) are updated, except the NHG for fc08::2

* Scenario 2: The route 2064:100::1d/128 is updated after zebra handles fc06::2/12 is withdrawn event
* Trigger: An NHT event is generated containing the nexthop address fc06::2 and the corresponding NHG ID.
* Expected Result: No further NHG update since all of them have been updated.

### Test case Remote Failure Simulation 1
* Scenario: The route 2064:100::1d/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2064:100::1d/128 and the corresponding NHG ID.
* Expected Result:
  * Update NHG for 2064:100::1d/128
  * Update NHG for 1::1/128
  * Update NHG for 4::4/128
  * No Update NHG for 2::2/128
  * No Update NHG for 3::3/128


### Test remote failure 2
* Scenario: The route 1::1/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2064:100::1d/128 and the corresponding NHG ID.
* Expected Result:
  * No update NHG for 2064:100::1d/128
  * Update NHG for 1::1/128
  * Update NHG for 4::4/128
  * No Update NHG for 2::2/128
  * No Update NHG for 3::3/128

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

### Test local failure
* Scenario: The route fc06::2/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address fc06::2/128 and the corresponding NHG ID.
* Expected Result:
  * Update NHG for 2064:100::1d/128
  * Update NHG for 2064:100::1e/128
  * Update NHG for 1::1/128
  * Update NHG for 2::2/128
  * Update NHG for 3::3/128
  * Update NHG for 4::4/128


### Test remote failure 1
* Scenario: The route 2064:100::1d/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2064:100::1d/128 and the corresponding NHG ID.
* Expected Result:
  * Update NHG for 2064:100::1d/128
  * No update NHG for 2064:100::1e/128
  * Update NHG for 1::1/128
  * No update NHG for 2::2/128
  * No update NHG for 3::3/128
  * No update NHG for 4::4/128

### Test remote failure 2
* Scenario: The route 1::1/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 1::1/128 and the corresponding NHG ID.
* Expected Result:
  * No update NHG for 2064:100::1d/128
  * No update NHG for 2064:100::1e/128
  * Update NHG for 1::1/128
  * No update NHG for 2::2/128
  * No update NHG for 3::3/128
  * No update NHG for 4::4/128

### Test remote failure 3
* Scenario: The route 2::2/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2::2/128 and the corresponding NHG ID.
* Expected Result:
  * No update NHG for 2064:100::1d/128
  * No update NHG for 2064:100::1e/128
  * No update NHG for 1::1/128
  * Update NHG for 2::2/128
  * No update NHG for 3::3/128
  * Update NHG for 4::4/128

## Test Topology 3 SRv6 VPN case
Assume we have 2064:100::1d/128 and 2064:200::1e/128 learnt via eBGP. Each route has two paths via fc06::2 and fc08::2

```
B>* 2064:100::1d/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
B>* 2064:200::1e/128 [20/0] (243) (pic_nh 0) via fc06::2, Ethernet12, weight 1, 3d02h57m
*                                            via fc08::2, Ethernet4, weight 1, 3d02h57m
```

We have SRv6 VPN routes via   2064:100::1d and  2064:200::1e
```
B> 192.100.0.10/32 [20/0] via 2064:100::1d (vrf default) (recursive), label 16, seg6 fd00:201:201:1::, weight 1, 00:00:08
                            via fc06::2, Ethernet12 (vrf default), label 16, seg6 fd00:201:201:1::, weight 1, 00:00:08
                            via fc08::2, Ethernet4 (vrf default), label 16, seg6 fd00:201:201:1::, weight 1, 00:00:08
                          via 2064:200::1e (vrf default) (recursive), label 32, seg6 fd00:202:202:2::, weight 1, 00:00:08
                            via fc06::2, Ethernet12 (vrf default), label 32, seg6 fd00:202:202:2::, weight 1, 00:00:08
                            via fc08::2, Ethernet4 (vrf default), label 32, seg6 fd00:202:202:2::, weight 1, 00:00:08
```

### Test local failure
* Scenario: The route fc06::2/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address fc06::2/128 and the corresponding NHG ID.
* Expected Result:
  * Update NHG for 2064:100::1d/128
  * Update NHG for 2064:100::1e/128
  * No update for SONiC NHG


### Test remote failure
* Scenario: The route 2064:100::1d/128 is withdrawn.
* Trigger: An NHT event is generated containing the nexthop address 2064:100::1d/128 and the corresponding NHG ID.
* Expected Result:
  * No Update NHG for 2064:100::1e/128
  * Update for SONiC NHG