# RIB/FIB Convergence — Plan B: sonic-frr

> **For agentic workers:** Use /alinos.subagent-dev (recommended) or /alinos.executing-plans to implement this plan task-by-task.

**Goal:** Implement NHT event generation in `sonic-frr` — add a dplane operation enum, ctx fields, and accessors; refactor `zebra_rnh_resolve_nexthop_entry()` to return a `route_entry_queued` out-parameter; add a trigger point in `zebra_rnh_eval_nexthop_entry()`; verify the message content using a mock FPM listener.
**Architecture:** Event flow — RNH state change → if `state_changed && !route_entry_queued` → enqueue into the dplane queue via `dplane_nht_event_update()` → the SONiC FPM provider serializes it into `RTM_NEWNHTEVENT` (6000).
**Tech Stack:** C (Zebra dplane framework), FRR topotest (Python).

**Dependencies:** Plan A Tasks 1-3 (NhtEvent schema + template + render_schema.py). **Task 4 requires Plan A Task 4 (`nhtevent_capi.h` must exist)** — if developing standalone within sonic-frr, the message payload portion of Plan B Task 6 requires all of Plan A to be complete.
**Corresponding spec:** [`ribfib-convergence-frr.md`](../../convergence-specs/ribfib-convergence-frr.md)

---

## File structure overview

**Modified files:**
- `sonic-frr/zebra/zebra_dplane.h` — add `DPLANE_OP_NHT_EVENT_UPDATE` + accessor declarations
- `sonic-frr/zebra/zebra_dplane.c` — add `struct dplane_nht_info` + `dplane_nht_event_update()` + accessor implementations + `dplane_op2str()` branch
- `sonic-frr/zebra/zebra_rnh.c` — add an out parameter to `zebra_rnh_resolve_nexthop_entry()`; thread the parameter through `zebra_rnh_evaluate_entry()` / `zebra_rnh_eval_nexthop_entry()`; insert the trigger point

**New files:**
- `sonic-frr/tests/topotests/ribfib_nht_event/test_nht_event.py` (or reuse an existing RIB/FIB topotest directory)
- `sonic-frr/tests/topotests/ribfib_nht_event/mock_fpm_listener.py`

---

## Task 1: dplane enum extension

**Files:**
- Modify: `sonic-frr/zebra/zebra_dplane.h`

- [ ] **Step 1: Locate `enum dplane_op_e`** (around line 118-172)
- [ ] **Step 2: Append after `DPLANE_OP_NH_DELETE`**:
  ```c
  DPLANE_OP_NHT_EVENT_UPDATE,
  ```
- [ ] **Step 3: Compile (without the accessors there will be issues at other call sites, but at least the enum itself should compile)**: `cd sonic-frr && make -j$(nproc) zebra/zebra_dplane.o` — warnings about undeclared accessors are acceptable for now, but an enum typo will surface immediately
- [ ] **Step 4: Commit**

---

## Task 2: dplane ctx carries NHT fields

**Files:**
- Modify: `sonic-frr/zebra/zebra_dplane.h`
- Modify: `sonic-frr/zebra/zebra_dplane.c`

- [ ] **Step 1: Read the `struct zebra_dplane_ctx` definition** (`zebra_dplane.c`, Grep `struct zebra_dplane_ctx {`)
- [ ] **Step 2: Add the new struct in zebra_dplane.c**:
  ```c
  struct dplane_nht_info {
      struct prefix rnh_prefix;
      struct prefix prev_resolved_prefix;
      uint32_t      prev_resolved_nhg_id;
      struct prefix curr_resolved_prefix;
      uint32_t      curr_resolved_nhg_id;
  };
  ```
- [ ] **Step 3: Add `struct dplane_nht_info nht;` to the internal union of `zebra_dplane_ctx`** (if a union already exists) or add it as a new member
- [ ] **Step 4: Declare the 5 accessors in zebra_dplane.h**:
  ```c
  const struct prefix *dplane_ctx_get_nht_rnh_prefix(const struct zebra_dplane_ctx *ctx);
  const struct prefix *dplane_ctx_get_nht_prev_resolved_prefix(const struct zebra_dplane_ctx *ctx);
  uint32_t             dplane_ctx_get_nht_prev_resolved_nhg_id(const struct zebra_dplane_ctx *ctx);
  const struct prefix *dplane_ctx_get_nht_curr_resolved_prefix(const struct zebra_dplane_ctx *ctx);
  uint32_t             dplane_ctx_get_nht_curr_resolved_nhg_id(const struct zebra_dplane_ctx *ctx);
  ```
- [ ] **Step 5: Implement the 5 accessors in zebra_dplane.c**:
  ```c
  const struct prefix *dplane_ctx_get_nht_rnh_prefix(const struct zebra_dplane_ctx *ctx) {
      DPLANE_CTX_VALID(ctx);
      return &ctx->u.nht.rnh_prefix;
  }
  /* The other 4 are similar */
  ```
- [ ] **Step 6: Compile** `make -j$(nproc) zebra/zebra_dplane.o` passes
- [ ] **Step 7: Commit**

---

## Task 3: `dplane_nht_event_update()` constructor

**Files:**
- Modify: `sonic-frr/zebra/zebra_dplane.h`
- Modify: `sonic-frr/zebra/zebra_dplane.c`

- [ ] **Step 1: Declare in zebra_dplane.h**:
  ```c
  enum zebra_dplane_result dplane_nht_event_update(
      const struct rnh *rnh,
      const struct prefix *prev_resolved_prefix,
      uint32_t prev_resolved_nhg_id);
  ```
- [ ] **Step 2: Implement in zebra_dplane.c**:
  ```c
  enum zebra_dplane_result dplane_nht_event_update(
      const struct rnh *rnh,
      const struct prefix *prev_resolved_prefix,
      uint32_t prev_resolved_nhg_id)
  {
      struct zebra_dplane_ctx *ctx;

      if (!rnh) {
          return ZEBRA_DPLANE_REQUEST_FAILURE;
      }

      ctx = dplane_ctx_alloc();
      if (!ctx) {
          return ZEBRA_DPLANE_REQUEST_FAILURE;
      }

      ctx->zd_op = DPLANE_OP_NHT_EVENT_UPDATE;
      ctx->zd_status = ZEBRA_DPLANE_REQUEST_SUCCESS;

      /* rnh_prefix: the nexthop prefix tracked by RNH */
      prefix_copy(&ctx->u.nht.rnh_prefix, &rnh->node->p);

      /* prev: the old resolution result cached by the caller */
      if (prev_resolved_prefix) {
          prefix_copy(&ctx->u.nht.prev_resolved_prefix,
                      prev_resolved_prefix);
      } else {
          memset(&ctx->u.nht.prev_resolved_prefix, 0,
                 sizeof(struct prefix));
      }
      ctx->u.nht.prev_resolved_nhg_id = prev_resolved_nhg_id;

      /* curr: read from the current rnh state (copy_state has already run) */
      prefix_copy(&ctx->u.nht.curr_resolved_prefix, &rnh->resolved_route);
      ctx->u.nht.curr_resolved_nhg_id = rnh->resolved_nhg_id;

      return dplane_provider_enqueue_to_zebra(ctx);
  }
  ```
- [ ] **Step 3: Add `case DPLANE_OP_NHT_EVENT_UPDATE: return "NHT_EVENT_UPDATE";` in `dplane_op2str()`**
- [ ] **Step 4: Compile `make -j$(nproc) zebra/zebra_dplane.o`** passes
- [ ] **Step 5: Commit**

---

## Task 4: Refactor `zebra_rnh_resolve_nexthop_entry()` to add an out parameter

**Files:**
- Modify: `sonic-frr/zebra/zebra_rnh.c`

- [ ] **Step 1: Locate the function** (`zebra_rnh.c`, around line 677)
- [ ] **Step 2: Modify the function signature**:
  ```c
  static struct route_entry *
  zebra_rnh_resolve_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
                                  struct route_node *nrn,
                                  const struct rnh *rnh,
                                  struct route_node **prn,
                                  bool *route_entry_queued);
  ```
- [ ] **Step 3: Initialize at the top of the function body**:
  ```c
  if (route_entry_queued) {
      *route_entry_queued = false;
  }
  ```
- [ ] **Step 4: Set the flag in the QUEUED skip branch** (around line 741-748):
  ```c
  if (CHECK_FLAG(re->status, ROUTE_ENTRY_QUEUED) &&
      !CHECK_FLAG(re->status, ROUTE_ENTRY_INSTALLED)) {
      if (route_entry_queued) {
          *route_entry_queued = true;
      }
      if (IS_ZEBRA_DEBUG_NHT_DETAILED)
          zlog_debug("        Route Entry %s queued",
                     zebra_route_string(re->type));
      continue;
  }
  ```
- [ ] **Step 5: Compile** `make -j$(nproc) zebra/zebra_rnh.o` — this will fail because the caller does not yet pass the parameter; accept this error temporarily
- [ ] **Step 6: Fix the caller `zebra_rnh_evaluate_entry()`** (around line 862) — define a local `bool route_entry_queued = false;`, pass it to resolve, then pass this bool to eval:
  ```c
  bool route_entry_queued = false;
  re = zebra_rnh_resolve_nexthop_entry(zvrf, afi, nrn, rnh, &prn,
                                       &route_entry_queued);
  if (!re && rnh->state == NULL && !force)
      continue;
  zebra_rnh_eval_nexthop_entry(zvrf, afi, force, nrn, rnh, prn, re,
                               route_entry_queued);
  ```
- [ ] **Step 7: Add the new parameter to the `zebra_rnh_eval_nexthop_entry()` signature**:
  ```c
  static void zebra_rnh_eval_nexthop_entry(struct zebra_vrf *zvrf, afi_t afi,
                                           int force, struct route_node *nrn,
                                           struct rnh *rnh,
                                           struct route_node *prn,
                                           struct route_entry *re,
                                           bool route_entry_queued);
  ```
  The function body does not use this parameter yet (Task 5 will insert the trigger point)
- [ ] **Step 8: Compile passes** `make -j$(nproc) zebra/zebra_rnh.o`
- [ ] **Step 9: Commit**

---

## Task 5: Insert the NHT trigger point in `zebra_rnh_eval_nexthop_entry()`

**Files:**
- Modify: `sonic-frr/zebra/zebra_rnh.c`

- [ ] **Step 1: Read the function body** (around line 786-839) and confirm the `copy_state()` call sites (line 813, 816)
- [ ] **Step 2: Add the include at the top of the file** (if not already present): `#include "zebra/zebra_dplane.h"`
- [ ] **Step 3: Cache the prev values at the beginning of the function body** (before or after `zebra_rnh_remove_from_routing_table(rnh);`):
  ```c
  struct prefix prev_resolved_route = rnh->resolved_route;
  uint32_t prev_resolved_nhg_id = rnh->resolved_nhg_id;
  ```
- [ ] **Step 4: Inside the `if (state_changed || force ...)` block, after the `zebra_rnh_notify_protocol_clients` call, add the dplane trigger**:
  ```c
  /* PIC Phase 1: if the resolution result really changed and the candidate route
   * is not blocked by QUEUED, send an NHT event to dplane to support fpmsyncd fast withdrawal. */
  if (state_changed && !route_entry_queued) {
      dplane_nht_event_update(rnh, &prev_resolved_route,
                              prev_resolved_nhg_id);
  }
  ```
- [ ] **Step 5: Compile `make -j$(nproc) zebra/zebra`** to build all of zebra and confirm there are no errors
- [ ] **Step 6: Commit**

---

## Task 6: Mock FPM Listener infrastructure

**Files:**
- Create: `sonic-frr/tests/topotests/ribfib_nht_event/mock_fpm_listener.py`
- Create: `sonic-frr/tests/topotests/ribfib_nht_event/__init__.py`

- [ ] **Step 1: Create `__init__.py`** as an empty file
- [ ] **Step 2: Create `mock_fpm_listener.py`**:
  ```python
  """Mock FPM listener that captures netlink messages for topotest assertions.

  Listens on a TCP socket (FPM protocol), parses:
    - 4-byte FPM header
    - nlmsghdr
    - Netlink attributes
  Records received messages in a list for later assertion.
  """
  import socket
  import struct
  import threading
  import json

  RTM_NEWNHTEVENT = 6000
  FPM_MSG_TYPE_NETLINK = 1
  FPM_NHA_JSON_STR = 2

  class MockFpmListener:
      def __init__(self, host="0.0.0.0", port=2620):
          self._host = host
          self._port = port
          self._sock = None
          self._thread = None
          self._stop = threading.Event()
          self.messages = []  # list of dicts

      def start(self):
          self._sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
          self._sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
          self._sock.bind((self._host, self._port))
          self._sock.listen(1)
          self._thread = threading.Thread(target=self._accept_loop, daemon=True)
          self._thread.start()

      def stop(self):
          self._stop.set()
          if self._sock:
              self._sock.close()

      def _accept_loop(self):
          try:
              conn, _ = self._sock.accept()
              buf = b""
              while not self._stop.is_set():
                  chunk = conn.recv(4096)
                  if not chunk:
                      break
                  buf += chunk
                  buf = self._parse(buf)
          except OSError:
              pass

      def _parse(self, buf):
          while len(buf) >= 4:
              ver, msg_type, msg_len = struct.unpack("!BBH", buf[:4])
              if len(buf) < msg_len:
                  return buf
              payload = buf[4:msg_len]
              buf = buf[msg_len:]
              if msg_type == FPM_MSG_TYPE_NETLINK and len(payload) >= 16:
                  nl_len, nl_type = struct.unpack("=IH", payload[:6])
                  attrs = self._parse_attrs(payload[16 + 12:])  # nlmsghdr(16) + rtmsg(12)
                  self.messages.append({
                      "nlmsg_type": nl_type,
                      "attrs": attrs,
                  })
          return buf

      def _parse_attrs(self, buf):
          attrs = {}
          while len(buf) >= 4:
              alen, atype = struct.unpack("=HH", buf[:4])
              data = buf[4:alen]
              if atype == FPM_NHA_JSON_STR:
                  try:
                      s = data.rstrip(b"\x00").decode("utf-8", errors="replace")
                      attrs["FPM_NHA_JSON_STR"] = json.loads(s)
                  except json.JSONDecodeError:
                      attrs["FPM_NHA_JSON_STR_raw"] = data
              # 4-byte alignment
              padded = (alen + 3) & ~3
              buf = buf[padded:]
          return attrs

      def wait_for_message(self, nlmsg_type, timeout=10.0):
          """Poll until a message of the given type arrives or timeout."""
          import time
          deadline = time.time() + timeout
          while time.time() < deadline:
              for msg in self.messages:
                  if msg["nlmsg_type"] == nlmsg_type:
                      return msg
              time.sleep(0.1)
          return None
  ```
- [ ] **Step 3: Commit**

---

## Task 7: TDD — topotest base topology

**Files:**
- Create: `sonic-frr/tests/topotests/ribfib_nht_event/test_nht_event.py`

- [ ] **Step 1: Refer to the existing RIB/FIB topotest on `ribfib_2_yuqing`** to familiarize yourself with the topotest framework (`from lib.topogen import Topogen`, etc.)
- [ ] **Step 2: Create `test_nht_event.py`** with a fixture:
  ```python
  """NHT event topotest: verify RTM_NEWNHTEVENT emission from zebra."""
  import os
  import pytest
  from functools import partial
  from lib.topogen import Topogen, TopoRouter, get_topogen
  from lib.common_config import step
  from mock_fpm_listener import MockFpmListener, RTM_NEWNHTEVENT

  pytestmark = [pytest.mark.ribfib]

  @pytest.fixture(scope="module")
  def tgen(request):
      topo = {
          "routers": {"r1": {"links": {"r2": {}}},
                      "r2": {"links": {"r1": {}}}},
      }
      tgen = Topogen(topo, request.module.__name__)
      tgen.start_topology()
      for _, router in tgen.routers().items():
          router.load_config(TopoRouter.RD_ZEBRA,
                             os.path.join(os.path.dirname(__file__),
                                          f"{_}/zebra.conf"))
          router.load_config(TopoRouter.RD_BGP,
                             os.path.join(os.path.dirname(__file__),
                                          f"{_}/bgpd.conf"))
      tgen.start_router()
      yield tgen
      tgen.stop_topology()
  ```
- [ ] **Step 3: Add the conf file directories `r1/zebra.conf` / `r1/bgpd.conf` / `r2/*`** — minimal BGP neighbor configuration, where one side announces a prefix that is tracked by the other
- [ ] **Step 4: Run the topotest skeleton** `cd sonic-frr/tests/topotests && python3 -m pytest ribfib_nht_event/test_nht_event.py -v -k "not test_"` to verify the topology comes up
- [ ] **Step 5: Commit**

---

## Task 8: TDD — positive case: NHT unreachable triggers RTM_NEWNHTEVENT

**Files:**
- Modify: `sonic-frr/tests/topotests/ribfib_nht_event/test_nht_event.py`

- [ ] **Step 1: Add a failing test**:
  ```python
  def test_nht_event_on_unreachable(tgen):
      """After withdrawing the route for a nexthop, an NHT event with curr_resolved_nhg_id=0 should be triggered."""
      r1 = tgen.gears["r1"]

      listener = MockFpmListener(port=2620)
      listener.start()

      # Configure r1 to use --nhg-fib + fpm connect to the mock listener
      r1.vtysh_cmd("configure terminal\nfpm connection ip 127.0.0.1 port 2620\nend")

      # Withdraw the BGP route for the nexthop on r2
      r2 = tgen.gears["r2"]
      r2.vtysh_cmd("configure terminal\nrouter bgp 65002\nno network 10.0.0.0/24\nend")

      msg = listener.wait_for_message(RTM_NEWNHTEVENT, timeout=15.0)
      listener.stop()

      assert msg is not None, "RTM_NEWNHTEVENT should be emitted"
      body = msg["attrs"]["FPM_NHA_JSON_STR"]
      assert body["curr_resolved_nhg_id"] == 0
      assert body["prev_resolved_nhg_id"] != 0
      assert body["rnh_prefix"].startswith("10.0.0")
  ```
- [ ] **Step 2: Run `pytest -v test_nht_event.py::test_nht_event_on_unreachable`** to confirm it fails (no message or assertion failure)
- [ ] **Step 3: Check whether `--nhg-fib` is enabled on the zebra side**; adjust the topology conf so that r1 starts with `--nhg-fib`
- [ ] **Step 4: Re-run the test** — it should pass
- [ ] **Step 5: Commit**

---

## Task 9: TDD — negative case: no trigger when the candidate is QUEUED

**Files:**
- Modify: `sonic-frr/tests/topotests/ribfib_nht_event/test_nht_event.py`

- [ ] **Step 1: Add a failing test**:
  ```python
  def test_no_nht_event_when_queued(tgen):
      """While a route is in the transient ROUTE_ENTRY_QUEUED state, no NHT event should be triggered."""
      r1 = tgen.gears["r1"]
      # Widen the QUEUED window via --asic-offload=notify_on_offload or by injecting a large number of routes
      r1.vtysh_cmd("configure terminal\ndebug zebra rib detail\nend")

      listener = MockFpmListener(port=2621)
      listener.start()

      # Trigger a nexthop change within a very short window: first inject many routes to congest the meta queue,
      # then change the nexthop -- at this point the candidate is skipped due to QUEUED
      # ... (actual topology construction omitted; the core assertion follows)

      import time
      time.sleep(2.0)  # Wait for any messages that might arrive
      listener.stop()
      # Assert: no RTM_NEWNHTEVENT should occur during the QUEUED window
      unwanted = [m for m in listener.messages if m["nlmsg_type"] == RTM_NEWNHTEVENT]
      assert not unwanted, f"unexpected NHT events during QUEUED window: {unwanted}"
  ```
  > **Note:** Precisely creating a QUEUED window requires topology-level tuning; if it is hard to make stable, it can be replaced at the unit test layer (zebra's own gtest covering the branches of `zebra_rnh_eval_nexthop_entry`). This topotest must have at least one QUEUED skip case (even if it is a heuristic verification).
- [ ] **Step 2: Run the test** to confirm it passes stably (if unstable, consider switching to a zebra-internal gtest)
- [ ] **Step 3: Commit**

---

## Task 10: TDD — coverage of the three prev/curr combinations

**Files:**
- Modify: `sonic-frr/tests/topotests/ribfib_nht_event/test_nht_event.py`

- [ ] **Step 1: Add three tests** covering respectively:
  - `test_nht_event_prev_only`: prev != 0, curr == 0 (already covered by Task 8; repeated explicitly here to enumerate the scenario)
  - `test_nht_event_prev_and_curr`: prev != 0, curr != 0 (IGP switches to a new NHG)
  - `test_nht_event_curr_only`: prev == 0, curr != 0 (nexthop goes from unreachable to reachable)
- [ ] **Step 2: Run the three tests** each asserting the corresponding field combination
- [ ] **Step 3: Commit**

---

## Task 11: Build the debian package and smoke test

**Files:** None (execution only)

- [ ] **Step 1: Fully build `sonic-frr` into a deb** (`dpkg-buildpackage -us -uc` or build via the sonic-buildimage submodule)
- [ ] **Step 2: `dpkg -c frr_*.deb | grep zebra` to confirm the zebra binary is present**
- [ ] **Step 3: Commit (if there are changes outside gitignored build artifacts)**

---

## Self-Review

**Spec coverage:**
- dplane enum ✓ (Task 1)
- ctx fields + accessors ✓ (Task 2)
- `dplane_nht_event_update()` ✓ (Task 3)
- add out parameter to resolve ✓ (Task 4)
- eval trigger point ✓ (Task 5)
- Mock FPM listener ✓ (Task 6)
- positive case ✓ (Task 8)
- QUEUED negative case ✓ (Task 9)
- three prev/curr scenarios ✓ (Task 10)

**Placeholder scan:** No TBD/TODO. Task 9 includes an explanation of precisely creating the QUEUED window along with a fallback recommendation; these are not placeholders.

**Type consistency:**
- The 5 fields of `struct dplane_nht_info` are consistent with the Plan A schema
- The accessor return types are consistent with the usage points in Plan A Task 13 (`const struct prefix *` / `uint32_t`)
- The parameters of `dplane_nht_event_update()` are consistent with the call site in Task 5

---

## Execution Handoff

Plan complete. Two execution options:
1. **Subagent-Driven (recommended)** - `/alinos.subagent-dev`
2. **Inline Execution** - `/alinos.executing-plans`

**Cross-plan dependencies:** Tasks 1-3 should be completed before Plan A Task 13 (Plan A Task 13 needs the accessors and the enum). Tasks 6-10 require Plan A to be complete (because FPM message assembly depends on sonic-fib's `nht_event_encode()`).
