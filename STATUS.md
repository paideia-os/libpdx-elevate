# libpdx-elevate — status

**Wave:** R49 shared library
**Current milestone:** M3 (audit + retry integration) — M3-001 landed, M3-002 in progress

See `design/tooling/r49-r50-plan.md` §5.14 in paideia-os for the full breakdown.

## Milestone rollup

| ID              | Title                                                                          | State  |
|-----------------|--------------------------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + wrap elv_pack_request from R48.M7 codec                             | LANDED |
| M1-002 (#2)     | svc.elevate-broker binding + block-on-reply skeleton                           | LANDED |
| M2-001 (#3)     | auto-approve path: consult elevate_policy.pdx before human hop                 | LANDED |
| M2-002 (#4)     | human-approve path with timeout (default 30s, per-request configurable)        | LANDED |
| M2-003 (#5)     | Cap<KIND_ELEVATE_CHANNEL=0x191> with bounded-lifetime self-invalidation        | LANDED |
| M3-001 (#6)     | request + response journal via libpdx-audit (extends UEJ_KIND_ELEVATE)         | LANDED |
| M3-002 (#7)     | retry-with-backoff for transient broker unavailability                         | OPEN   |

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

## M3 — audit + retry integration (in progress)

- `src/elevate_client_journal.pdx` (issue #6, LANDED): REQ + APR
  journal through the kernel's `uej_append` (UEJ_KIND_ELEVATE = 5).
  Two-record shape means an auditor can reconstruct the full policy
  negotiation from the journal alone. Provides:
    - `elevate_client_journal_req(actor_fp_lo, caps, dur_ns) -> seq`
      — body0 = ELVJ_EVT_REQ (1), body1 = caps, body2 = dur_ns.
    - `elevate_client_journal_apr(actor_fp_lo, granted_caps, expire_ns)
      -> seq` — body0 = ELVJ_EVT_APR (2), body1 = granted_caps,
      body2 = expire_ns.
    - `elevate_client_request_ex_j(...)` — audit-first wrapper over
      the M2 `elevate_client_request_ex`. Sequence: fetch actor_fp_lo
      (process-global) → journal REQ → run elevate → journal APR on
      success. REQ-journal failure aborts before the wire hop (audit-
      first D3 discipline). APR-journal failure after grant returns
      `ELVJ_ERR_APR_JOURNAL_FAIL` so the caller knows to revoke the
      grant (audit chain is broken otherwise).
    - Per-op consent: `elevate_client_request_ex` already runs
      `elevate_client_policy_check` on every invocation (M2). The
      M3 wrapper preserves this per-call discipline and adds the
      journal pair, so every mutating op is both policy-consulted
      AND audit-recorded.

**Error band added at M3-001:**
- `0xFFFFEA40..0xFFFFEA4F` — `ElevateClientJournal` band
  (`ELVJ_ERR_BAD_ACTOR`, `ELVJ_ERR_REQ_JOURNAL_FAIL`,
  `ELVJ_ERR_APR_JOURNAL_FAIL`).

## Cross-repo dependencies

- **paideia-os (at HEAD)** — `KIND_ELEVATE_CHANNEL = 0x191`
  (#1626), broker registration (#1627), wire codec (#1549), policy
  table (#1550), user_events_journal (`uej_append`, #1544) all
  present. M2/M3 consume kernel primitives: `endpoint_write_pending`,
  `endpoint_take_pending`, `hpet_now_ns`, `svc_lookup`, `uej_append`,
  `elevate_channel_cap_mint_inner`, `elevate_channel_cap_revoke`.
- **libpdx-cap.M2** — NOT yet landed. `elevate_client_cap.pdx`
  ships `elevate_client_cap_narrow_stub` as a placeholder; when
  libpdx-cap.M2 lands, replace the stub body with a call to
  `cap_narrow_rights`. No M2/M3 flow blocks on the stub returning
  the passthrough value.
- **libpdx-audit.M2** — NOT yet landed. M3-001 wires directly to
  kernel `uej_append` under UEJ_KIND_ELEVATE (5), which is the same
  route libpdx-audit.M2 will eventually consume. When libpdx-audit
  lands, `elevate_client_journal_req/_apr` bodies swap to
  `audit_begin`/`audit_record_output`/`audit_commit` calls; the
  wrapper `elevate_client_request_ex_j` keeps its signature.

## Followups for paideia-os (not blocking M3)

- Kernel-side `elevate_channel_row_set_expire(row_id, expire_ns)` or
  a mint variant accepting `expire_ns` would let the shadow deadline
  map in `elevate_client_cap.pdx` collapse back into the row's
  `[+32]` slot. API is designed to make that migration transparent
  to callers.
- Broker daemon body (currently `ELVB_DISPATCH_STUB` in
  `src/kernel/core/ipc/elevate_broker.pdx`) — until it consumes REQ
  frames and produces APR replies, `elevate_client_request_ex*` will
  reliably time out. M4 will exercise the full loop once the broker
  daemon lands.
- `tick_ns` wire in `uej_append` (currently placeholder 0 per
  user_events_journal.pdx L226). Not blocking; audit records still
  carry seq for ordering.

## Next

M3-002 — retry-with-backoff for transient broker unavailability
(issue #7). Wraps `elevate_client_request_ex_j` with a bounded
retry loop for the transient broker-side symptoms
(`ELVC_ERR_TIMEOUT`, `ELVC_ERR_SEND_FAIL`, `ELVC_ERR_LOOKUP_FAIL`).
Then M4 — tests + smoke matrix.
