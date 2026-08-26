# libpdx-elevate — status

**Wave:** R49 shared library
**Current milestone:** M5 (signed 1.0 release) — complete; M6
(enhancement wave, v1.1.0) in progress
**Version:** 1.0.0 (2026-08-22), v1.1.0 hardening underway

See `design/tooling/r49-r50-plan.md` §5.14 in paideia-os for the full
M1–M5 breakdown, and `.plans/enhancement-plan.md` for the M6
(post-1.0.0) rationale.

## M6 — enhancement wave (in progress)

Filed 2026-08-25 from the org-wide 14-repo enhancement audit
(`.plans/enhancement-plan.md`).  The audit's finding: the 1.0.0
client half is complete but *advisory* — nothing forced a consumer to
present a grant before performing the privileged operation it gated,
and the one entry point every real consumer called
(`elevate_client_request`) never dispatched, so its return value was
a coin-flip between "proceed" and "refuse" depending on which
consumer read it. M6 adds a credential-shaped surface alongside the
existing status-shaped one.

- **ENH-005 (#12, LANDED):** `elevate_client_request` renamed to
  `elevate_client_request_norealize`; `ELVC_STUB` renamed to
  `ELVC_NOT_DISPATCHED` (same value). Source-breaking by design — see
  `elevate_client.pdx` WHY THE RENAME.

## Milestone rollup

| ID              | Title                                                                          | State  |
|-----------------|--------------------------------------------------------------------------------|--------|
| M1-001 (#1)     | scaffold + wrap elv_pack_request from R48.M7 codec                             | LANDED |
| M1-002 (#2)     | svc.elevate-broker binding + block-on-reply skeleton                           | LANDED |
| M2-001 (#3)     | auto-approve path: consult elevate_policy.pdx before human hop                 | LANDED |
| M2-002 (#4)     | human-approve path with timeout (default 30s, per-request configurable)        | LANDED |
| M2-003 (#5)     | Cap<KIND_ELEVATE_CHANNEL=0x191> with bounded-lifetime self-invalidation        | LANDED |
| M3-001 (#6)     | request + response journal via libpdx-audit (extends UEJ_KIND_ELEVATE)         | LANDED |
| M3-002 (#7)     | retry-with-backoff for transient broker unavailability                         | LANDED |
| M4-001 (#8)     | auto-approve match/miss matrix against policy table                            | LANDED |
| M4-002 (#9)     | cap-lifetime enforcement test (past deadline -> kernel revoke)                 | LANDED |
| M5-001 (#10)    | dual-signed release + .pdxdoc + mirror push                                    | LANDED |

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

## M3 — audit + retry integration (complete)

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

- `src/elevate_client_retry.pdx` (issue #7, LANDED): retry-with-
  backoff wrapper over `elevate_client_request_ex_j`. Retriable
  set is `{ELVC_ERR_TIMEOUT, ELVC_ERR_SEND_FAIL,
  ELVC_ERR_LOOKUP_FAIL}` — the three transient broker-side
  symptoms. Everything else (client-side gates, policy refusals,
  journal failures) passes through unchanged. Provides:
    - `elevate_client_retry_set_max_attempts(n)` / `_get_max_attempts()`
      — bounds [1, 8], default 3.
    - `elevate_client_retry_backoff_ns_for_attempt(attempt)` —
      table lookup: 100 ms → 400 ms → 1.6 s → cap at 1.6 s.
    - `elevate_client_retry_delay(dur_ns)` — bounded busy-poll on
      `hpet_now_ns` (matches the recv-timeout idiom in
      `elevate_client_recv_reply`).
    - `elevate_client_request_ex_r(caps, dur, req_buf, reply_ep_id,
      reply_buf, timeout_ns) -> ELVC_*/ELVJ_ERR_*/ELVR_ERR_*` —
      the full-featured entry point: retry × (journal + policy +
      send + recv). Every attempt writes its own REQ audit record
      and the eventual success writes its APR record, giving the
      auditor a complete per-attempt trail.

**Error bands added at M3:**
- `0xFFFFEA20..0xFFFFEA2F` — `ElevateClientRetry` band
  (`ELVR_ERR_EXHAUSTED`, `ELVR_ERR_BAD_ARG`).
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
  daemon lands. The retry wrapper at M3-002 turns that timeout into
  a bounded three-attempt sequence rather than a single 30 s stall.
- Userspace-visible sleep primitive (R51 scheduler-wait syscall).
  Once landed, `elevate_client_retry_delay` becomes a thin wrapper
  and the polling loop retires.
- `tick_ns` wire in `uej_append` (currently placeholder 0 per
  user_events_journal.pdx L226). Not blocking; audit records still
  carry seq for ordering.

## M4 — tests + smoke (complete)

- `tests/elevate_client_policy_test.pdx` (issue #8, LANDED): boot
  witness `elevate_client_policy_witness` for the client-side
  auto-approve match/miss matrix.  Fifteen stages, single
  fingerprint `LIBPDX-ELEVATE M4 POLICY OK`.  Covers:
    - reset scrubs table + stats
    - install of a wildcard-target row (row 0) and a specific-
      target row (row 1) with distinct caps + duration caps
    - hit paths: wildcard-target match, specific-target match,
      HITS stat increments in lockstep
    - miss paths: caps not a subset, dur above row cap, target
      mismatch; NO_MATCHES stat increments correspondingly
    - install refusals: caps==0, caps has reserved bits set,
      dur==0, dur > 1h max; count unaffected
    - rate-limit exhaustion on row 1 (rate=5): 5 hits ok, 6th
      returns RATE_LIMIT and bumps RATE_DENIES
    - out-of-range hit(idx=16) -> BAD_ARG
- `tests/elevate_client_cap_test.pdx` (issue #9, LANDED): boot
  witness `elevate_client_cap_witness` for cap-lifetime
  enforcement.  Twelve stages, fingerprint `LIBPDX-ELEVATE M4 CAP
  OK`.  Covers:
    - reset scrubs shadow deadline map + stats
    - bind_expire / get_expire happy path (60s deadline, non-zero
      readback)
    - dur=0 clears; get_expire returns 0 after clear
    - row_id=17 gates: bind_expire and expired both refuse
      BAD_ROW
    - expired discrimination: live (0), no deadline
      (NO_DEADLINE), bad row (BAD_ROW)
    - check_and_revoke on live row -> STATE_LIVE + CHECKS_LIVE
      bumped (no kernel call)
    - check_and_revoke error passthrough: NO_DEADLINE and BAD_ROW
      surface unchanged
    - past-deadline enforcement: slot poked directly with a 1-ns
      absolute deadline (bypasses bind_expire so the compare is
      deterministic against hpet_now_ns); expired returns 1;
      check_and_revoke invokes kernel `elevate_channel_cap_revoke`
      and reports either STATE_REVOKED (kernel accepted) or
      ERR_REVOKE_FAIL (kernel BAD_SLOT because no cap was minted
      here); AUTO_REVOKES + REVOKE_FAILS sum to exactly 1

Both witnesses use the paideia-os boot-witness convention (mirror
of `tests/kernel/ipc/elevate_broker_synth.pdx` at #1627): stage-
tracked, klog OK on success, klog_s1_d1 FAIL with the failing
stage id, and reset on both exits so a failure cannot be mistaken
for a fault in the next witness in the boot sequence.  Labels
`ecpw_` and `eccw_` are disjoint from every other witness in the
tree.  Neither test needs the broker daemon body to be live: both
target client-side state machines that stand up without a live APR.

**Deferred to when the broker daemon lands (paideia-os followup):**
- Full REQ -> broker -> policy-consult -> APR -> journal loop
  witness that a request/response pair actually reaches the
  user_events_journal.  Today's `elevate_client_request_ex_j` +
  retry wrapper can be exercised only up to ELVC_ERR_TIMEOUT
  because the broker stub always returns DISPATCH_STUB.
- End-to-end mint of a real Cap<KIND_ELEVATE_CHANNEL> and the
  narrow -> revoke -> re-narrow cycle (blocked on libpdx-cap.M2
  cap_narrow_rights; today's `elevate_client_cap_narrow_stub`
  ships as documented placeholder).

## M5 — signed 1.0 release (complete)

- `manifest.pdxsig` (issue #10, LANDED): dual-signed release manifest
  per D4 (design/tooling/plan.md §6).  Canonical text form covers
  tool-name, version, source-commit
  (`796fe888cd384dfefc3d088652c1b02d3538f186`), source-tree hash
  (`sha256:1e53491d…add6e6b`), per-file `.pdx` hashes for src/ and
  tests/, `caps.decl` + `deps.list` + `CHANGELOG.md` +
  `doc/libpdx-elevate.pdxdoc` canonical hashes, and the kernel-
  substrate pin (paideia-os #1626 KIND_ELEVATE_CHANNEL, #1627
  broker registration, #1549 wire codec, #1550 policy table, #1544
  user_events_journal).  `[signature.author]` and
  `[signature.paideia-root]` carry the 42-byte sentinel
  `<MLDSA65-SIG:STUB-PENDING-V0.33-CRYPTO-KDF>` — the manifest body
  is hash-stable, so `paideia-as release --sign` against v0.33-
  crypto-kdf produces bit-identical signatures on any machine
  holding the author key.  Sentinel is scanned + replaced in place;
  no body change.
- `caps.decl` (issue #10, LANDED): capability manifest (D4) for
  each entry point: `elevate_client_lookup_broker`,
  `elevate_client_send_req`, `elevate_client_recv_reply`,
  `elevate_client_journal_req/_apr`, `elevate_client_cap_mint`
  / `_check_and_revoke`, `elevate_client_request_ex` / `_ex_j` /
  `_ex_r`.  Documentary at library level: the kernel cap-check
  runs against the consuming process's cap set; this file tells
  `paideia-as release --sign` what to hash into `manifest.pdxsig`
  and tells `pkg install libpdx-elevate` what to prompt the caller
  to grant.  KIND_SUPERVISOR nowhere; every cap narrowed to
  broker_ep_id / reply / row_id / uej.  Forward-note: libpdx-cap.M2
  swap adds `KIND_ELEVATE_CHANNEL(narrow, row_id)` in v1.1.0.
- `deps.list` (issue #10, LANDED): shared-lib dependencies with
  LANDED/PENDING/KERNEL status.  Two PENDING lines: `libpdx-cap
  >=0.2.0` (narrow_stub swap site in `elevate_client_cap.pdx`
  :90) and `libpdx-audit >=0.3.0` (journal_req/_apr swap site in
  `elevate_client_journal.pdx`).  Both PENDING is source-compatible
  with LANDED — swap lands in v1.1.0.  Seven KERNEL primitives
  enumerated with paideia-os issue references.
- `doc/libpdx-elevate.pdxdoc` (issue #10, LANDED): I7 doc file
  rendered by the `doc` tool (§5.3 of r49-r50-plan.md).  NAME,
  SYNOPSIS (asm + conceptual), DESCRIPTION, INVARIANTS
  (D3/D4/I4/I5/I7), per-ENTRY signature + semantics for the eight
  public entry points, full STATUS CODES table (all five error
  bands + exit-code mapping to I4 §4.1), two EXAMPLES (happy-path
  request_ex_r + cap-lifetime check_and_revoke re-entry), POSIX
  DIFFERENCES (five-axis contrast with `sudo`), SEE ALSO (five
  design-doc + three paideia-os-issue cross-refs).
- `CHANGELOG.md` (issue #10, LANDED): 1.0.0 section documents
  M1-M4 landings, M5 artifacts, KIND ordinal footprint, error
  bands, and the five known deferred paths (kernel row_set_expire,
  broker daemon body, R51 scheduler-wait syscall, libpdx-cap.M2,
  libpdx-audit.M2).
- `.plans/mirror-push.md` (issue #10, LANDED): runbook for
  pushing to `pkgs.paideia-os/main/libpdx-elevate/1.0.0/` when
  the mirror exists.  Nine-step procedure covering source freeze,
  manifest-body hash-stability check, hash-field population,
  author-side sign, package build, staging push, Paideia signing
  bot re-sign gate, verification from a clean machine, git tag.
  Dependency ordering: five R49 libs push in order libpdx-cap →
  libpdx-semantic-pipe → libpdx-argv → libpdx-audit →
  libpdx-elevate (elevate last so its two PENDING deps flip to
  LANDED before its push).  Rollback + escalation paths
  documented.

**Release tag:** `v1.0.0` (2026-08-22).

**Deferred to when paideia-as v0.33-crypto-kdf tag is reachable:**
- ML-DSA-65 signature population (author + paideia-root).  Sentinel
  scan + in-place replace; manifest body already frozen and hash-
  stable.
- Actual mirror push (blocked on `pkgs.paideia-os` mirror repo
  existing).  Runbook is ready.

## Next

Downstream: pkg.M3 wires libpdx-elevate as one of its dual-signed
deps.  When pkg.M4 closes and pkg.M5-001 opens, the mirror-push
runbook above fires against `pkgs.paideia-os/staging`.

Followup: v1.1.0 lands the two PENDING dep swaps (libpdx-cap
`cap_narrow_rights` + libpdx-audit `audit_begin`/`_record_output`/
`_commit`) once libpdx-cap.M2 and libpdx-audit.M2 close.  Entry-point
signatures unchanged; only status codes retire (`ELCC_NOTE_NO_NARROW`).
