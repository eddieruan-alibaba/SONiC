# RIB/FIB Convergence — sonic-fib

- **Repository**: `sonic-buildimage` (`src/libraries/sonic-fib/`)
- **Overview**: [`ribfib-convergence-overview.md`](ribfib-convergence-overview.md)
- **Related**: [`ribfib-convergence-dplane-encoding.md`](ribfib-convergence-dplane-encoding.md) (the FPM provider that consumes this library)
- **Tests**: [`ribfib-convergence-test.md`](ribfib-convergence-test.md)

---

## 1. Responsibility of this repository

Extend the `libnexthopgroup` encode/decode library with a new, **independent** `NhtEvent` message type: a new JSON schema, generated C++ class, and C-API. The `render_schema.py` code generator is **extended** (existing functions untouched) and new Jinja2 templates are added. Everything lives in namespace `fib`, mirroring the existing `NextHopGroupFull` file structure.

**Not in scope:** the FPM provider message assembly that *calls* this library (see the dplane-encoding sub-spec), FRR RNH changes (sonic-frr), fpmsyncd backwalk (sonic-swss), all tests (see test sub-spec).

---

## 2. New JSON schema

Add `src/libraries/sonic-fib/schema/NhtEvent.json` — 5 flat fields, no `$defs`, no nesting:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "NhtEvent.json",
  "title": "NhtEvent",
  "description": "NHT event carrying prev/curr resolved NHG info for fast-fixup",
  "type": "object",
  "properties": {
    "rnh_prefix":           { "type": "string" },
    "prev_resolved_prefix": { "type": "string" },
    "prev_resolved_nhg_id": { "type": "integer", "minimum": 0 },
    "curr_resolved_prefix": { "type": "string" },
    "curr_resolved_nhg_id": { "type": "integer", "minimum": 0 }
  },
  "required": ["rnh_prefix", "prev_resolved_prefix", "prev_resolved_nhg_id",
               "curr_resolved_prefix", "curr_resolved_nhg_id"],
  "additionalProperties": false
}
```

---

## 3. Extend render_schema.py (do not modify existing functions)

**Principle:** the `NextHopGroupFull` code path (`build_c_root_struct` / `build_root_struct` / `build_def_structs`, etc.) must not change. Support for `NhtEvent` is added by **new functions + a new dispatch branch** only.

**New pieces:**
- Add `_render_nhtevent()` (a dedicated renderer for the flat, `$defs`-free NhtEvent schema).
- In `main()`, dispatch to the new renderer when `schema.title == "NhtEvent"`; otherwise fall through to the unchanged `NextHopGroupFull` logic.

**Dispatch outline:**

```python
def main():
    # ... existing mode / args parsing ...
    schema_title = schema.get("title", "")

    if schema_title == "NhtEvent":
        _render_nhtevent(schema, ...)   # new branch + new templates
    else:
        # existing NextHopGroupFull logic, unchanged
        ...
```

---

## 4. New Jinja2 templates

In `src/libraries/sonic-fib/templates/` (all in namespace `fib`):

- `nhtevent.h.j2` — C++ class header (`fib::NhtEvent`).
- `nhtevent_json.h.j2` — encode/decode impl (`NhtEvent::to_json()` / `NhtEvent::from_json()`).
- `c_nhtevent.h.j2` — C-API header.

Templates mirror the existing `nexthopgroupfull*.j2` structure but are much simpler because NhtEvent has only 5 flat fields (no `$defs`, no nesting).

---

## 5. C-API

Add `src/libraries/sonic-fib/src/c-api/nhtevent_capi.{h,cpp}`:

```c
/* nhtevent_capi.h */
#ifdef __cplusplus
extern "C" {
#endif

/* forward declaration so callers do not need the generated header on the
 * default include path; callers that build the struct include
 * "src/c_nhtevent.h" explicitly. */
struct C_NhtEvent;

/* Encode: returns a heap-allocated JSON string; caller frees. */
char *nht_event_encode(
    const struct prefix *rnh_prefix,
    const struct prefix *prev_resolved_prefix,
    uint32_t             prev_resolved_nhg_id,
    const struct prefix *curr_resolved_prefix,
    uint32_t             curr_resolved_nhg_id);

/* Decode: parse JSON, fill the C struct. Returns non-zero on failure. */
int nht_event_decode(const char *json_str, struct C_NhtEvent *out);

#ifdef __cplusplus
}
#endif
```

`struct C_NhtEvent` (defined in the generated `c_nhtevent.h`):

```c
struct C_NhtEvent {
    char     rnh_prefix[64];
    char     prev_resolved_prefix[64];
    uint32_t prev_resolved_nhg_id;
    char     curr_resolved_prefix[64];
    uint32_t curr_resolved_nhg_id;
};
```

> **Include note (bug fixed during Phase 2):** `nhtevent_capi.h` must only *forward-declare* `struct C_NhtEvent`. Including the generated `c_nhtevent.h` directly from the public header only resolves with `-Isrc`, which broke the FRR-side build. Callers that materialize the struct include `src/c_nhtevent.h` themselves.

**Usage split:**
- The FRR-side FPM provider calls **only** `nht_event_encode()` (C code).
- fpmsyncd is C++ and uses `fib::NhtEvent::from_json()` directly — it does not go through the C-API.

---

## 6. Build rules

`src/libraries/sonic-fib/Makefile.am`:
- Add the code-gen rule for the NhtEvent schema → generate header / json impl / c-api header.
- Add the generated artifacts to `libnexthopgroup`'s dist headers and sources.
- `debian/*` — update install / `-dev` lists if the headers are published.

---

## 7. File layout (mirrors NextHopGroupFull)

```
sonic-buildimage/src/libraries/sonic-fib/
├── schema/
│   ├── NextHopGroupFull.json    ← existing, untouched
│   └── NhtEvent.json            ← new
├── src/
│   ├── c-api/
│   │   ├── nexthopgroup_capi.cpp/.h   ← existing, untouched
│   │   └── nhtevent_capi.cpp/.h       ← new
│   └── nexthopgroup_debug.cpp/.h      ← existing, untouched
├── templates/
│   ├── nexthopgroupfull*.j2           ← existing (4), untouched
│   └── nhtevent*.j2                   ← new (3)
├── scripts/
│   └── render_schema.py               ← extended (new function + branch)
├── Makefile.am                        ← add build rules
└── tests/
    └── test_nhtevent.cpp              ← new (see test sub-spec)
```

---

## 8. Error handling and boundaries

- `nht_event_encode`: NULL input prefix → return NULL; `inet_ntop` failure → return NULL.
- `nht_event_decode`: JSON parse failure → non-zero; missing required field → non-zero.
- All string fields length-checked (≤ 64, covers the longest IPv6 prefix textual form).

---

## 9. Compatibility

- The `NextHopGroupFull` code path is **untouched** (zero behavior change).
- The `FPM_NHA_JSON_STR` NLA type already exists and is reused (not modified).

---

## 10. Testing

Round-trip (encode→decode) and error-path unit tests for `NhtEvent` are described in the consolidated test sub-spec: [`ribfib-convergence-test.md`](ribfib-convergence-test.md) §sonic-fib UT.
