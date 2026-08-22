# libpdx-elevate — status

**Wave:** R49 shared library
**Current milestone:** M2 (core implementation) — complete

See `design/tooling/r49-r50-plan.md` §5.14 in paideia-os for the full breakdown.

## Milestone rollup

| ID              | Title                                                                          | State  |
|-----------------|--------------------------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + wrap elv_pack_request from R48.M7 codec                             | LANDED |
| M1-002 (#2)     | svc.elevate-broker binding + block-on-reply skeleton                           | LANDED |
| M2-001 (#3)     | auto-approve path: consult elevate_policy.pdx before human hop                 | LANDED |
| M2-002 (#4)     | human-approve path with timeout (default 30s, per-request configurable)        | LANDED |
| M2-003 (#5)     | Cap<KIND_ELEVATE_CHANNEL=0x191> with bounded-lifetime self-invalidation        | LANDED |

## M1 — design + skeleton (complete)

- `src/elevate_request.pdx` (issue #1): mirror of the R48.M7 ElevateChannel
  wire schema. Provides ELV_* constants, REQ frame layout offsets, client-
  side validators (`elevate_request_cap_mask_valid`,
  `elevate_request_duration_valid`), the op-word packer
  (`elevate_request_pack_op_word`, mirror of kernel `elv_pack_req`), and
  `elevate_request_write_frame` — assembles a full 32-byte REQ frame into
  a caller-provided buffer.
- `src/elevate_client.pdx` (issue #2): svc.elevate-broker binding +
  block-on-reply skeleton. Provides `elevate_client_lookup_broker` (wraps
  `svc_lookup` with the canonical name), an eight-slot stats table
  matching the shape of `ElevateBroker._elevate_broker_stats`, and
  `elevate_client_request(caps, dur, req_buf, reply_buf)` — the M1
  send-and-block skeleton that returns `ELVC_STUB` once the request is
  assembled and the broker endpoint resolved.

## M2 — core implementation (complete)

- `src/elevate_client_policy.pdx` (issue #3): client-side mirror of
  the kernel `ElevatePolicy` table. 16-row × 48-byte table shared
  magic (`PDXP`), install/check/hit primitives with the same match
  semantics as `ep_check` / `ep_hit_row`. Used as the fast-path
  classifier by `elevate_client_request_ex` (see M2-002) to pick
  between the fast (500 ms) and human (30 s) recv timeouts.
- `src/elevate_client_send.pdx` (issue #4): real send + bounded recv.
  Provides:
    - `elevate_client_set_human_timeout` / `elevate_client_set_fast_timeout`
      + matching getters (defaults 30 s / 500 ms).
    - `elevate_client_pack_hdr(reply_ep_id, payload_len)` — inline
      packer for the 8-byte IPC frame header per `Frame` layout.
    - `elevate_client_send_req(broker_ep_id, reply_ep_id, req_buf)` —
      publishes the 32-byte REQ payload via
      `endpoint_write_pending`.
    - `elevate_client_recv_reply(reply_ep_id, reply_buf, timeout_ns)`
      — bounded polling loop on `endpoint_take_pending` using
      `hpet_now_ns` for the deadline; refuses `ELVC_ERR_TIMEOUT`
      once now >= deadline. Also validates that the returned frame
      is a well-formed APR (op == `ELV_OP_APR`, non-zero expire_ns).
    - `elevate_client_check_grant(reply_buf, requested_caps)` —
      mirrors `elv_check_grant`: granted must be a subset of
      requested.
    - `elevate_client_set_target_fp_lo(fp)` / `_get_target_fp_lo()`
      — process-global user identity consulted by the policy step.
      Kept as a process-global so the full-flow entry point fits in
      the 6-register SysV window.
    - `elevate_client_request_ex(caps, dur, req_buf, reply_ep_id,
                                 reply_buf, timeout_ns)` — the
      full-flow M2 entry point: policy consult → REQ assemble → REQ
      send → APR recv → grant subset check. `timeout_ns == 0` selects
      fast or human default based on the policy classification;
      target_fp_lo comes from the process-global set via
      `elevate_client_set_target_fp_lo`.
- `src/elevate_client_cap.pdx` (issue #5): `Cap<KIND_ELEVATE_CHANNEL>`
  wrapper with bounded lifetime.
    - `elevate_client_cap_mint` — thin wrapper over the kernel
      `elevate_channel_cap_mint_inner`; kernel default `expire_ns = 0`
      is compensated by a client-side shadow deadline map (16 slots,
      one per possible row_id).
    - `elevate_client_cap_bind_expire(row_id, dur_ns)` — records
      `hpet_now_ns() + dur_ns` in the shadow map.
    - `elevate_client_cap_expired(row_id)` — 1/0/err against
      `hpet_now_ns()`.
    - `elevate_client_cap_check_and_revoke(row_id)` — if past
      deadline, calls kernel `elevate_channel_cap_revoke`; returns
      `ELCC_STATE_LIVE` / `ELCC_STATE_REVOKED` / `ELCC_ERR_*`.
      Idempotent (kernel `REVOKE_ALREADY` collapses to
      `ELCC_STATE_REVOKED`).
    - `elevate_client_cap_narrow_stub(rights)` — STUB awaiting
      libpdx-cap.M2 `cap_narrow_rights`; returns rights unchanged
      plus the `ELCC_NOTE_NO_NARROW` status code so a caller can
      detect the pending upgrade site.

**Error bands added at M2:**
- `0xFFFFEA01..0xFFFFEA05` — new client-side transport codes
  (`ELVC_ERR_TIMEOUT`, `ELVC_ERR_BAD_REPLY`, `ELVC_ERR_GRANT_INVALID`,
  `ELVC_ERR_BAD_EXPIRE`, `ELVC_ERR_BAD_REPLY_EP`).
- `0xFFFFEA10..0xFFFFEA1F` — `ElevateClientPolicy` band
  (`ELCP_ERR_BAD_ARG`, `RATE_LIMIT`, `NO_MATCH`, `ENOSPC`, `BAD_DUR`,
  `BAD_CAPS`).
- `0xFFFFEA30..0xFFFFEA3F` — `ElevateClientCap` band
  (`ELCC_ERR_BAD_ROW`, `MINT_FAIL`, `NO_DEADLINE`, `REVOKE_FAIL`,
  `ELCC_STATE_LIVE`, `EXPIRED`, `REVOKED`, `ELCC_NOTE_NO_NARROW`).

## Cross-repo dependencies

- **paideia-os (at HEAD)** — `KIND_ELEVATE_CHANNEL = 0x191`
  (#1626), broker registration (#1627), wire codec (#1549), policy
  table (#1550) all present. M2 additionally consumes kernel
  primitives: `endpoint_write_pending`, `endpoint_take_pending`,
  `hpet_now_ns`, `elevate_channel_cap_mint_inner`,
  `elevate_channel_cap_revoke`.
- **libpdx-cap.M2** — NOT yet landed. `elevate_client_cap.pdx`
  ships `elevate_client_cap_narrow_stub` as a placeholder; when
  libpdx-cap.M2 lands, replace the stub body with a call to
  `cap_narrow_rights`. No M2 flow blocks on the stub returning the
  passthrough value.

## Followups for paideia-os (not blocking M2)

- Kernel-side `elevate_channel_row_set_expire(row_id, expire_ns)` or
  a mint variant accepting `expire_ns` would let the shadow deadline
  map collapse back into the row's `[+32]` slot. `elevate_client_cap.pdx`
  API is designed to make that migration transparent to callers.
- Broker daemon body (currently `ELVB_DISPATCH_STUB` in
  `src/kernel/core/ipc/elevate_broker.pdx`) — until it consumes REQ
  frames and produces APR replies, `elevate_client_request_ex` will
  reliably time out. M4 will exercise the full loop once the
  broker daemon lands.

## Next

M3 — semantic-pipe / audit integration (extends `UEJ_KIND_ELEVATE`
via libpdx-audit) + retry-with-backoff for transient broker
unavailability. Depends on libpdx-audit.M2.
