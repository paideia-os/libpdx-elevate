# libpdx-elevate — status

**Wave:** R49 shared library
**Current milestone:** M1 (design + skeleton) — complete

See `design/tooling/r49-r50-plan.md` §5.14 in paideia-os for the full breakdown.

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
  assembled and the broker endpoint resolved. Actual endpoint invoke +
  block-on-recv arrive at M2.

**Error bands:**
- `0xFFFFE5EA..0xFFFFE5EF` — wire-schema errors, shared with kernel
  `ElevateChannel`. New at M1: `ELV_ERR_BAD_BUF` (0xFFFFE5EA).
- `0xFFFFEA00..0xFFFFEA0F` — client-side transport errors owned by this
  library. `ELVC_STUB` (0xFFFFEA00) is the M1 happy-path return.

## Upstream substrate (paideia-os, at HEAD)

- `KIND_ELEVATE_CHANNEL = 0x191` — landed at #1626.
- `svc.elevate-broker` registration seam (`ElevateBroker.elevate_broker_register`,
  `elevate_broker_dispatch`) — landed at #1627.
- Wire codec (`elv_pack_req`, `elv_mask_valid`, `elv_dur_valid`,
  `elv_check_grant`, ELV_OP_*, ELV_ERR_*) — landed at R48.M7-001 (#1549).

## Next

M2 — auto-approve path (consult `elevate_policy.pdx` before the human hop),
human-approve path with configurable timeout, and mint a
`Cap<KIND_ELEVATE_CHANNEL>` with bounded-lifetime self-invalidation.
Depends on libpdx-cap.M2 (cap narrowing for the returned sub-caps).
