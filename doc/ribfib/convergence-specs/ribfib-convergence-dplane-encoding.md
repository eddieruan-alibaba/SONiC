# RIB/FIB Convergence — dplane FPM encoding

- **Repository**: `sonic-buildimage` (`src/sonic-frr/dplane_fpm_sonic/`)
- **Overview**: [`ribfib-convergence-overview.md`](ribfib-convergence-overview.md)
- **Depends on**: [`ribfib-convergence-frr.md`](ribfib-convergence-frr.md) (dplane op + accessors), [`ribfib-convergence-sonic-fib.md`](ribfib-convergence-sonic-fib.md) (`nht_event_encode()`)
- **Tests**: [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

---

## 1. Responsibility of this repository

`src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` is the SONiC-specific FPM provider. It serializes the FRR-side `DPLANE_OP_NHT_EVENT_UPDATE` dplane op into an `RTM_NEWNHTEVENT` (type 6000) netlink message and sends it over the FPM channel to fpmsyncd.

This provider **replaces** the stock `dplane_fpm_nl.c` at runtime — hence all SONiC-specific FPM message types (SRv6, PIC context, NHG FIB, and now NHT event) are assembled here.

**Not in scope:** the dplane op enum / ctx / accessors (sonic-frr — see FRR sub-spec), the JSON encode library (`nht_event_encode()`, sonic-fib — see sonic-fib sub-spec), fpmsyncd parsing (sonic-swss), all tests (see test sub-spec).

---

## 2. Message type enum extension

Add to the `custom_nlmsg_types` enum:

```c
enum custom_nlmsg_types {
    RTM_NEWSRV6LOCALSID  = 1000,
    RTM_DELSRV6LOCALSID  = 1001,
    RTM_NEWPICCONTEXT    = 2000,
    RTM_DELPICCONTEXT    = 2001,
    RTM_NEWSRV6VPNROUTE  = 3000,
    RTM_DELSRV6VPNROUTE  = 3001,
    RTM_NEWSIDLIST       = 4000,
    RTM_DELSIDLIST       = 4001,
    RTM_NEWNHGFIB        = 5000,
    RTM_DELNHGFIB        = 5001,
    RTM_NEWNHTEVENT      = 6000,   /* new */
};
```

> **Note:** an NHT event is an event notification; there is no `DEL` counterpart, so no 6001 is added.

---

## 3. Message layout

For a `DPLANE_OP_NHT_EVENT_UPDATE` ctx, produce:

```
nlmsghdr:
    nlmsg_type   = RTM_NEWNHTEVENT (6000)
    nlmsg_flags  = NLM_F_CREATE | NLM_F_REQUEST
    nlmsg_pid    = nl->snl.nl_pid

struct rtmsg:
    rtm_family   = address family of rnh_prefix (AF_INET / AF_INET6)
    other fields zeroed

NLA FPM_NHA_JSON_STR (attr type = 2):
    JSON string (the 5 NhtEvent fields)
```

---

## 4. Op dispatch handler

Add a `DPLANE_OP_NHT_EVENT_UPDATE` case to the provider's op dispatch. It pulls the 5 fields via the dplane accessors (defined in the FRR sub-spec) and calls `nht_event_encode()` (sonic-fib) to build the JSON string placed into `FPM_NHA_JSON_STR`.

```c
case DPLANE_OP_NHT_EVENT_UPDATE:
    /* assemble nlmsghdr + rtmsg */
    req->n.nlmsg_len   = NLMSG_LENGTH(sizeof(struct rtmsg));
    req->n.nlmsg_flags = NLM_F_CREATE | NLM_F_REQUEST;
    req->n.nlmsg_type  = RTM_NEWNHTEVENT;
    req->n.nlmsg_pid   = nl->snl.nl_pid;

    const struct prefix *rnh_p = dplane_ctx_get_rnh_prefix(ctx);
    req->r.rtm_family = rnh_p->family;

    /* call sonic-fib C-API to build the JSON */
    char *json_str = nht_event_encode(
        rnh_p,
        dplane_ctx_get_rnh_prev_resolved_prefix(ctx),
        dplane_ctx_get_rnh_prev_resolved_nhg_id(ctx),
        dplane_ctx_get_rnh_curr_resolved_prefix(ctx),
        dplane_ctx_get_rnh_curr_resolved_nhg_id(ctx));
    if (!json_str) {
        zlog_err("%s: Failed to encode NhtEvent JSON", __func__);
        return -1;
    }

    if (!nl_attr_put(&req->n, buflen, FPM_NHA_JSON_STR,
                     json_str, strlen(json_str) + 1)) {
        free(json_str);
        return -1;
    }
    free(json_str);
    break;
```

To materialize `struct C_NhtEvent` (if needed) the provider includes `src/c_nhtevent.h` explicitly; the public `nhtevent_capi.h` only forward-declares it (see sonic-fib sub-spec §5).

---

## 5. Error handling and boundaries

- `dplane_ctx_get_rnh_*()` returns NULL → log error, drop the message.
- `nht_event_encode()` returns NULL (allocation failure or invalid arg) → log error, drop the message.
- NLA put failure → free `json_str`, return error.

---

## 6. Compatibility

- `NextHopGroupFull` / other existing message assembly paths are **untouched**.
- The `FPM_NHA_JSON_STR` NLA type already exists and is reused.
- An older fpmsyncd that does not implement `RTM_NEWNHTEVENT` dispatch → the message is dropped by routesync's `onMsgRaw` type whitelist (it only accepts known types); other messages are unaffected.

---

## 7. Files touched

- `src/sonic-frr/dplane_fpm_sonic/dplane_fpm_sonic.c` — add `RTM_NEWNHTEVENT` enum value + `DPLANE_OP_NHT_EVENT_UPDATE` assembly handler.

---

## 8. Testing

The FPM-wire format (nlmsghdr + rtmsg + 5-field `FPM_NHA_JSON_STR`) is validated end-to-end by the FRR topotest via a mock FPM listener; see the consolidated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md) §topotest.
