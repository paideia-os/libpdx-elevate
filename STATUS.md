# libpdx-elevate — status

**Wave:** R49 shared library
**Current milestone:** M5 (signed 1.0 release) — complete; M6
(enhancement wave, v1.1.0) — complete, unreleased (no signed tag cut yet);
LE.M1-M3 (multilevel-chain foundations, v1.1.0 continuation) —
complete on main (see per-ticket table in `CHANGELOG.md`), unreleased;
v1.1.1 PATCH (LE.M3-001 real-mint-args pass) — complete on `main`,
supersedes the v1.1.0 label (which is retired without a signed tag);
LE.M5 (#31, #32, attestation) and LE.M7-001/002 (#35, #36, revoke
cascade + broker propagation) — complete on `main`, unreleased
**Version:** 1.0.0 (2026-08-22) tagged; v1.1.1 (v1.1.0-wave +
LE.M3-001 real-mint-args fix) landed on `main`, tag/signing pending a
release pass

See `design/tooling/r49-r50-plan.md` §5.14 in paideia-os for the full
M1–M5 breakdown, `.plans/enhancement-plan.md` for the M6 (post-1.0.0)
rationale, and the CHANGELOG's v1.1.0 entry for the LE.M1-M3 landings +
what's explicitly deferred to the LE.M4-M7 follow-up wave.

## LE.M1-M3 — multilevel-chain foundations (this pass, 2026-09-02)

Filed under milestones `LE.M1-polish`, `LE.M2-hardening`,
`LE.M3-multilevel` (issues #19-#28 + #37-#38).  M3 landed in full —
`elevate_client_cap_derive` + parent-row shadow map + depth-bounded
delegation.  M1/M2 landed selectively; see the per-ticket table in
`CHANGELOG.md`.

The narrow-slice landed here is the client-side primitive that lets a
consumer (a `shell` or `pkg`-style tool) derive a scoped, bounded
child cap from an already-held parent cap and shadow the delegation
chain in the library for a subsequent LE.M7 revoke-cascade to walk.
Kernel-side blocker (a real `elevate_channel_cap_derive_inner`
primitive that mints against a parent row's ep slot rather than a
caller-supplied broker cap) is tracked as a companion paideia-os
issue; the current implementation mints via `elevate_client_cap_mint`
using the caller's OWN broker-endpoint cap slot, which the caller
supplies via the v1.1.1 `mint_ctx_ptr` argument (a 3-word buffer:
parent_ep_slot, request_id, requester_pid — layout-compatible with
the first three words of `elevate_client_acquire`'s `mint_ctx_buf`).
This is functionally correct for shadow bookkeeping but consumes an
extra kernel-table row per derive.  The rewrite to the eventual real
derive primitive remains a plug-in replacement of one CALL site.

**v1.1.1 note (2026-09-02):** the v1.1.0 landing of LE.M3-001 was
documented as "mints via caller's own broker cap slot" but actually
hardcoded `parent_ep_slot`/`request_id`/`requester_pid` to zero at
the mint call, making the entire derive path return
`ELCC_ERR_MINT_FAIL` end-to-end for any real caller.  v1.1.1 wires
the caller-owned `mint_ctx_ptr` through — see the derive header in
`src/elevate_client_cap.pdx` and the LE.M3-001 row in `CHANGELOG.md`.
Source-breaking to a 3-arg caller of `_derive`; there are no working
3-arg callers because the 3-arg form was non-functional.

LE.M4, M6, M7 (issues #29-#30, #33-#36) — reap, audit sink,
revoke-cascade — all DEFERRED to the follow-up wave; comment posted
on each issue. LE.M5 (#31, #32, attestation), LE.M7-001/002 (#35,
#36, revoke cascade), LE.M4-001 (#29, per-cap-mask duration
ceilings), and LE.M4-002 (#30, `elevate_client_cap_reap_expired`)
have since landed — see below. LE.M6 (#33, #34, audit sink) remains
deferred.

## LE.M4-001 — per-cap-mask duration ceilings (landed, 2026-09-11)

`elevate_request_duration_valid_for(caps, dur)`
(`src/elevate_request.pdx`, #29) is the per-cap-mask ceiling gate.
Every set bit in `caps` looks up its ceiling in the read-only
`_elv_bit_ceiling_ns[8]` table; the combined-mask ceiling is
`min(_elv_bit_ceiling_ns[i])` over every set bit `i`.  Returns `1`
on pass or `ELV_ERR_DUR_EXCEEDS_CEILING = 0xFFFFE5E9` on fail (new
LE.M4-001 slot in the shared B1 band 0xFFFFE5E0..EF; the reserved
range under B1 shrinks to 0xFFFFE5E0..0xFFFFE5E8).  Named category
ceilings are surfaced as `ELV_CAP_R_ENUMERATE` / `_READ` (both 1 h),
`ELV_CAP_R_WRITE` (60 s), `ELV_CAP_R_UNLINK` / `_MOUNT` (both 30 s).
Bits 0..7 of the wire cap mask are classified per the CHANGELOG's
LE.M4-001 table (PDXFS_WRITE_SYSTEM as UNLINK-shaped, USER_REVOKE
as UNLINK-shaped, HW_MINT / DRIVER_MINT default to WRITE per issue
AC's "unknown = write's 60 s" conservative rule).

`elevate_request_pack_op_word` (same file) now runs three gates in
sequence — `cap_mask_valid` → BAD_MASK, `duration_valid` → BAD_DUR
(flat range, unchanged for source-compat), `duration_valid_for` →
DUR_EXCEEDS_CEILING (per-bit).  `elevate_request_write_frame`
inherits the ceiling gate transparently through its existing
`shr rax, 24; jnz` pack-err propagation clause; `buf` is left
UNMODIFIED on any ceiling refusal, preserving the partial-write
invariant documented at the write_frame return-code table.
`elevate_request_duration_valid` (the flat gate) is now marked
DEPRECATED for new pre-packer client-side use; kept for source-
compat + still called by the packer for the ELV_ERR_BAD_DUR
specific return.

Witness: `tests/elevate_request_ceiling_test.pdx` — fingerprint
`LIBPDX-ELEVATE LE.M4 CEILING OK`.  Eleven stages cover the
validator (read-only-passes, destructive-refuses, mixed-mask-
min-wins, boundary-at-ceiling), packer propagation, write_frame
propagation, named-ceiling equality invariants, and B1 band
membership + neighbor-slot distinctness for the new error.

Consumer impact: no source edit required today — every existing
caller reaches the ceiling gate via `elevate_request_write_frame`,
which returns the new error unchanged.  A consumer that used to
hold a destructive cap for > 60 s now fails at the packer instead
of at the broker, matching issue #29's stated end-state.

## LE.M4-002 — idle-time reap of the shadow deadline map (landed, 2026-09-11)

`elevate_client_cap_reap_expired() -> reaped_count : u64`
(`src/elevate_client_cap.pdx`, #30) walks the 16-slot shadow
deadline map and, for each slot whose deadline is past
`hpet_now_ns()`, calls `elevate_channel_cap_revoke` and scrubs the
row's shadow state via `elevate_client_cap_scrub_row`.  Returns the
count of rows freshly revoked (kernel `ELVC_OK`); rows the kernel
reports `ELVC_REVOKE_ALREADY` or that never made it into the kernel
table are still scrubbed to keep the shadow coherent, but are not
counted / audited — same OK-vs-ALREADY discrimination
`elevate_client_cap_revoke_cascade` already runs.  Idempotent by
construction: every touched slot has its deadline scrubbed to 0
regardless of the kernel revoke's outcome, so a back-to-back second
call finds no non-zero deadlines and returns exactly 0 before any
kernel round-trip.

Per successfully-revoked row, one `ELVJ_EVT_REAP = 7` audit record
lands via `elevate_client_journal_reap(actor_fp_lo, row_id,
deadline_ns)`.  `body2 = deadline_ns` lets an auditor subtract
wall-clock time on the surrounding uej seq to learn how long the row
lingered past its bound — the "return early after mint" pattern
issue #30 names as the reap's motivating case.  `ELVJ_EVT_REAP`
lands at 7 (not the "= 5" the issue AC text names, which collided
with `ELVJ_EVT_REVOKE_CASCADE`; same next-contiguous-free-slot
convention #35's own AC text-vs-landed-value deviation established).

An optional teardown helper `elevate_client_shutdown() ->
reaped_count : u64` composes `reap_expired` + `elevate_client_cap_
reset` + `elevate_client_journal_reset` for tools whose exit path
is "just stop".  Tools with their own explicit revoke discipline
may skip the helper.

Witness `tests/elevate_client_reap_test.pdx` (fingerprint
`LIBPDX-ELEVATE LE.M4 REAP OK`) covers: empty-map -> 0, 3-injected
past-deadline rows accepting count in [0, 3] under the tolerant
boot-witness pattern (kernel revoke branch is environment-dependent
without a live broker), shadow-scrub invariant, unconditional
idempotency on the second call, `journal_reap` OK-path,
`journal_reap` bad-actor gate, and `shutdown` clean teardown.  The
`ELVJ_EVT_REAP == 7` constant deviation is documented in the
CHANGELOG + the constant's own header comment; verifying it at
runtime would require the LE.M2-002 audit sink whose population
depends on libpdx-audit's `audit_append_leaf` (a PENDING dep) --
deliberately kept out of the witness so a boot regression in
libpdx-audit cannot mask a reap regression.

Consumer impact: none direct — housekeeping.  Consumers that leak
grants no longer leak them indefinitely if a boot-idle callback (or
their own exit path via `elevate_client_shutdown`) periodically
sweeps.

## LE.M5 — attestation (M5-001/002 landed; M5-003+ not filed)

`ELVJ_EVT_ATTEST = 4` (`src/elevate_client_journal.pdx`) plus its
entry point `elevate_client_cap_attest`, and `elevate_client_cap.pdx`'s
`bind_attestation_required` / `get_attestation_required` gate on
`elevate_client_cap_derive`. See the CHANGELOG's "LE.M5-001/002"
Unreleased entry for the full shape, error codes, and shadow-map
deltas. Not exercised here (deferred to an end-to-end broker witness,
same as LE.M3's own post-mint state): `elevate_client_cap_derive`'s
mint SUCCESS path when an attestation-required parent is correctly
attested, and the child-inherits-the-flag assertion, since neither is
reachable deterministically without a live broker in a boot witness.

## LE.M7-001/002 — revoke cascade + broker propagation (landed)

`elevate_client_cap_revoke_cascade(row_id)` (`src/elevate_client_cap.pdx`,
#35) force-revokes row_id and every transitive LE.M3-002 parent-map
descendant, deepest generation first, then row_id itself. No reverse
child-index exists (or is warranted at `ELCC_MAX_ROWS` = 16) — the walk
scans the flat 16-slot table per BFS growth pass instead. A kernel
revoke refusal other than "already revoked" aborts the whole walk
immediately (`ELCC_ERR_CASCADE_FAIL = 0xFFFFEA74`, new B9 extension
member); rows revoked before the refusal stay revoked and audited.
Second call on an already-swept chain returns 0 (idempotent). New
factored helper `elevate_client_cap_scrub_row` replaces what would have
been a third inline copy of the eight-map shadow scrub. New journal
event `ELVJ_EVT_REVOKE_CASCADE = 5` and entry point
`elevate_client_journal_revoke_cascade` (`src/elevate_client_journal.pdx`,
new B6 member `ELVJ_ERR_REVOKE_CASCADE_JOURNAL_FAIL = 0xFFFFEA45`,
journal stats widened 10→11 slots) — called best-effort per revoked row
(a failed revoke-audit does not block the revoke, unlike a failed
grant-audit elsewhere in this library).

`elevate_client_cap_drain_broker_exp(reply_ep_id)`
(`src/elevate_client_send.pdx`, #36) non-blockingly drains an
endpoint for pending `ELV_OP_EXP` frames and forwards each by row_id
to revoke_cascade. **Kernel-side note:** paideia-os does not emit
`ELV_OP_EXP` frames today (no reaper exists; tracked as paideia-os
#2122) — this is client-side scaffolding for when it does, the same
posture `elevate_client_cap_derive` already takes toward a not-yet-real
kernel mint primitive. The only documented EXP body layout names its
row-identifying field `request_id`, not `row_id`; this implementation
reads it as row_id per #36's literal acceptance criteria and documents
the naming gap for whoever lands the real emitter.

Tests: `tests/elevate_client_revoke_cascade_test.pdx`
(`LIBPDX-ELEVATE LE.M7 CASCADE OK` / `... EXP-DRAIN OK`). Per the same
no-live-broker constraint LE.M3's and LE.M5's own witnesses document,
the kernel-revoke happy path and the EXP-frame-triggers-cascade path
are not exercised end-to-end here — deferred alongside those.

## M6 — enhancement wave (complete)

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
- **ENH-001 (#11, LANDED):** `elevate_client_acquire`
  (`elevate_client_acquire.pdx`) — composed `request_ex_r` + mint +
  shadow-bind returning a `row_id` handle. New `elevate_client_cap.pdx`
  primitives: `elevate_client_cap_bind_expire_abs` (absolute-deadline
  shadow, no `hpet_now_ns` re-add), `elevate_client_cap_bind_grant` /
  `_get_grant` (granted-caps shadow map, `ELCC_ERR_NO_GRANT` on a
  never-bound row).
- **ENH-002 (#13, LANDED):** `elevate_client_require`
  (`elevate_client_require.pdx`) — cheap per-op re-assert against an
  acquired handle: freshness via `elevate_client_cap_check_and_revoke`
  + caps-subset against the shadow grant map, no broker hop.
- **ENH-004 (#16, LANDED):** `elevate_client_journal_op`
  (`elevate_client_journal.pdx`, `ELVJ_EVT_OP = 3`) and
  `elevate_client_require_j` (`elevate_client_require.pdx`) — one
  audit record per AUTHORIZED per-op use. The record count against a
  single REQ/APR pair is the grant-amplification signal the
  enhancement plan calls for (ENH-004 depends on ENH-002).
- **ENH-003 (#15, LANDED):** `elevate_client_require_scoped`
  (`elevate_client_require.pdx`) plus new `elevate_client_cap.pdx`
  primitives (`bind_scope`/`get_scope`, `bind_budget`/
  `get_budget_remaining`/`consume_budget`) — a handle may optionally be
  bound to an opaque scope fingerprint and/or a bounded op count, both
  additive over plain `elevate_client_require`. The budget shadow slot
  stores `remaining + 1` so exhaustion stays distinguishable from
  "never bound" (a plain 0-means-unbound encoding would let the
  (N+1)-th op silently read as unlimited).
- **ENH-006 (#17, LANDED, scoped):** `elevate_client_request_ex_ctx`
  (`elevate_client_send.pdx`) — explicit-context twin of
  `elevate_client_request_ex` taking `target_fp_lo` + fast/human/
  explicit timeout via a caller-owned `ctx_buf` instead of the three
  process-global mutable slots, so concurrent flows in one process do
  not race. Scoped deliberately: only the core full-flow primitive
  gets a ctx variant this release; the extension of the ctx discipline
  up through the journal/retry/acquire stack landed at LE.M1-002 (#20,
  below).
- **LE.M1-002 (#20, LANDED):** three new ctx-carrying entry points
  extending ENH-006 up the whole stack —
  `elevate_client_request_ex_j_ctx` (`elevate_client_journal.pdx`),
  `elevate_client_request_ex_r_ctx` (`elevate_client_retry.pdx`), and
  `elevate_client_acquire_ctx` (`elevate_client_acquire.pdx`). All
  three are 6-arg SysV calls -- `_acquire_ctx` was initially drafted
  as 7-arg (targeting paideia-as's stack-passed 7th-arg support in
  `tests/build-emit/sysv_x64_7arg_callee_stack_read.pdx`) but the
  build refused with `error[B1708]: a @no_frame (or unsafe-bodied)
  lambda cannot accept more than 6 parameters` -- 7-arg support is
  scoped to SAFE lambdas only, and every asm block in this library
  uses `unsafe { ... }`. The build-fix folds `mint_ctx_buf` into
  `ctx_buf` at `[+32]`: the ctx buffer grows from 4 words / 32 bytes
  (ENH-006 layout) to 5 words / 40 bytes; the 5th slot is
  ACQUIRE-ONLY (see the `ELVC_CTX_*_OFF` header block in
  `src/elevate_client_send.pdx` for the authoritative layout). Each
  ctx variant reads target_fp_lo/timeouts from `ctx_buf[+0..+32)` and
  dispatches through the next-level ctx twin so the same buffer
  threads through the whole audit + retry + acquire chain; `_ex_j_ctx`
  and `_ex_r_ctx` never read `[+32]`, so a caller composing only
  those two entry points may keep passing a 32-byte buffer. The three
  process-global-singleton legacy twins (`_ex_j`, `_ex_r`, `_acquire`)
  are refactored into thin wrappers over the ctx variants via a shared
  private helper `_elevate_client_build_default_ctx` — public
  signatures unchanged, byte-identical to the pre-refactor behaviour
  in the single-threaded-of-intent case (audit journal + retry stats +
  cap grant all identical). The `_acquire` wrapper allocates a
  40-byte stack ctx, lets the helper write words 0..3, then writes
  its own `mint_ctx_buf` argument to `[ctx+32]` before delegating to
  `_acquire_ctx`. `ELVC_ERR_BAD_CTX (0xFFFFEA06)` is the same B2 code
  every ctx entry point already returned; `_acquire_ctx` also raises
  `ELCA_ERR_BAD_BUF (0xFFFFEA50)` when `[ctx+32]` is 0 (a caller
  supplied ctx but forgot the acquire-only mint slot). A caller sees
  ONE band-B2 code across all four ctx call sites. Regression witness:
  `tests/elevate_client_ctx_expansion_test.pdx` (8 stages, fingerprint
  `LIBPDX-ELEVATE LE.M1 CTX-EXPANSION OK`; stage 6 covers the new
  `ELCA_ERR_BAD_BUF` gate on `[ctx+32]==0`). Stretch AC deferred: the
  "two concurrent flows under a single-threaded scheduler" witness
  requires user-space threading scaffolding this library does not
  expose today — filed as follow-up alongside a real concurrent
  consumer (shell).

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

## Followups for paideia-os (post-M3 landing state — see CHANGELOG for authoritative status)

Every kernel-side blocker below LANDED with the R90-XREPO.011.M1-006
wave; see `CHANGELOG.md`'s R90-XREPO.011.M1-006 entry for the
authoritative post-M3 landing state.  When these two documents ever
disagree, CHANGELOG wins.

- Kernel-side `elevate_channel_row_set_expire(row_id, expire_ns)` —
  **LANDED** as paideia-os#2118.  The shadow deadline map in
  `elevate_client_cap.pdx` can now be collapsed back into the row's
  `[+32]` slot; the API is unchanged and the migration is
  transparent to callers.  The shadow map stays in this library
  until a scheduled follow-up wave removes it.
- Broker daemon body — **LANDED** as paideia-os#2122.  The dispatch
  body in `src/kernel/core/ipc/elevate_broker.pdx` now consumes REQ
  frames and produces APR replies; `elevate_client_request_ex*` no
  longer reliably times out against the stub.  M4 end-to-end broker
  witness (issue #24) unblocks with this.
- Userspace-visible sleep primitive (R51 scheduler-wait syscall) —
  **LANDED** as paideia-os#2117.  `elevate_client_retry_delay` can
  now retire its bounded busy-poll on `hpet_now_ns` in favor of a
  thin `sched_wait` wrapper; scheduled follow-up.
- `/system/policy` format + seed — **LANDED** as paideia-os#2119.
  The client-side policy table (`elevate_client_policy.pdx`) reads
  from the same on-disk format the kernel-side authoritative table
  loads at boot.
- Fail-closed policy directive — **LANDED** as paideia-os#2121.  The
  client-side outcome reducer (`elevate_client_classify_outcome`,
  R90-XREPO.011.M1-006) enforces the same fail-closed default at the
  client boundary.
- `tick_ns` wire in `uej_append` (currently placeholder 0 per
  user_events_journal.pdx L226).  Not blocking; audit records still
  carry seq for ordering.  Deferred, no cross-repo dependency.

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

## M6 close-out (ENH-007, #14)

All seven M6 issues (#11–#17) are LANDED as of this pass. What v1.1.0
adds, in one place:

| Issue | Landed |
| --- | --- |
| ENH-005 (#12) | `elevate_client_request` → `elevate_client_request_norealize`; `ELVC_STUB` → `ELVC_NOT_DISPATCHED`. Source-breaking, intentional. |
| ENH-001 (#11) | `elevate_client_acquire` — composed flow returning a `row_id` handle. |
| ENH-002 (#13) | `elevate_client_require` — cheap per-op re-assert, no broker hop. |
| ENH-004 (#16) | `elevate_client_journal_op` / `elevate_client_require_j` — per-op audit trail. |
| ENH-003 (#15) | `elevate_client_require_scoped` — opaque scope + exhaustible op budget on a handle. |
| ENH-006 (#17) | `elevate_client_request_ex_ctx` — explicit-context variant of the core primitive. |
| LE.M1-002 (#20) | `elevate_client_request_ex_j_ctx` / `_ex_r_ctx` / `elevate_client_acquire_ctx` — extension of the ENH-006 ctx discipline through the whole audit/retry/acquire stack, plus refactor of the three legacy globals-based entry points into thin wrappers over the ctx twins via `_elevate_client_build_default_ctx`. Public signatures unchanged. |
| ENH-007 (#14) | This pass: README/STATUS accuracy — retired the "`ELVC_OK` or `ELVC_STUB` both mean proceed" example, marked `rm`/`pkg` as needing migration, added the v1.1.0 credential-shaped example. |

**What is still true and still deferred** (unchanged by M6, restated
here so this file does not overclaim): end-to-end enforcement is still
zero, because the paideia-os broker daemon body
(`src/kernel/core/ipc/elevate_broker.pdx`, `ELVB_DISPATCH_STUB`) does
not exist yet — every `_ex`/`_ex_j`/`_ex_r`/`_ctx`/`elevate_client_acquire`
call can only reach `ELVC_ERR_TIMEOUT` against a live broker today. M6
hardens the CLIENT-side API contract (unambiguous discrimination,
credential-shaped handles, auditable per-op amplification) so that
once the broker daemon lands, callers built against this surface are
already structurally correct rather than needing a second migration.
Also unchanged: the two libpdx-cap.M2 / libpdx-audit.M2 forward
references in `deps.list`, and the process-global singletons
`elevate_client_request_ex_ctx` provides an escape hatch from but does
not retire.

## Next

Downstream: pkg.M3 wires libpdx-elevate as one of its dual-signed
deps.  When pkg.M4 closes and pkg.M5-001 opens, the mirror-push
runbook above fires against `pkgs.paideia-os/staging` — now for a
v1.1.0 tag once one is cut (`manifest.pdxsig` / `caps.decl` /
`deps.list` reflect the M6 entry points already; the signed tag itself
is a separate release step not run by this pass).

Followup: v1.1.0+ lands the two PENDING dep swaps (libpdx-cap
`cap_narrow_rights` + libpdx-audit `audit_begin`/`_record_output`/
`_commit`) once libpdx-cap.M2 and libpdx-audit.M2 close.  Entry-point
signatures unchanged; only status codes retire (`ELCC_NOTE_NO_NARROW`).

Also open: `rm` migrating off the retired `elevate_client_request`
name (ENH-005, its own repo/issue) — see LE.M1-004 pass below for the
verified state. `pkg` has completed its name-only migration to
`elevate_client_request_norealize` as of pkg.ENH-008 (#33). The
LE.M1-002 (#20) landing above closes the ENH-006 follow-up "ctx
variants threaded all the way through the retry/journal/acquire
stack"; a shell-scale concurrent witness that drives two flows under
a single-threaded scheduler is the remaining stretch AC and is filed
as a follow-up (requires user-space threading scaffolding this
library does not yet expose).

## LE.M1-004 caller-list re-verification pass (#22, 2026-09-11)

Doc-only pass. The M6 close-out above catalogued caller state at the
ENH-007 timestamp (2026-08-25); this LE.M1-004 pass re-runs the same
grep-against-each-caller-repo discipline and refreshes the README's
Callers section so the v1.1.0 signed manifest is not shipped over
stale claims. Result per caller (mirrors README):

| Caller | State (2026-09-11) | Evidence |
| --- | --- | --- |
| `rm` | UNCHANGED — still on retired `elevate_client_request` | `rm@main` `src/elevate.pdx:234` still emits `call elevate_client_request;`; build breaks against v1.1.0 |
| `pkg` | MIGRATED (name rename only) | `pkg@main` `src/pkg_elevate.pdx:213` now emits `call elevate_client_request_norealize;` (pkg.ENH-008 / #33); disposition unchanged |
| `shell` | UNCHANGED — likely caller, still not linked | 9 `elevate_client` mentions across `broker_bind.pdx` / `shell.pdx` / `session.pdx` / `exec.pdx` / `dispatch.pdx` / `syscall.pdx`; ZERO are `call` instructions (all are shape-reference comments) |
| `mount.pdxfs` | UNCHANGED — fail-closed stub | `src/elevate.pdx` still `mov rax, 0; ret` |
| `umount.pdxfs` | UNCHANGED — fail-closed stub + NEW stub | existing `elevate_request_force_unmount` still `ELEV_DENY`; NEW landing `elevate_request_system_unmount` (umount.pdxfs.LE-001 / #22) also `ELEV_DENY` |
| `mkfs.pdxfs` | UNCHANGED — fail-closed stub | `src/elevate_wire.pdx` still fail-closed; `elevate_client_*` mentions are header planning notes |
| `mv` (newly surfaced) | Architecturally-committed, not yet linking libpdx-elevate | `mv@main` `src/elevate.pdx` (mv.M3-004 / mv#11) goes directly to `sys_svc_lookup("svc.elevate-broker")` + `sys_ipc_send`; grep count for `elevate_client` in the file: 0 |

Source: no changes. Files touched: `README.md` (Callers section
rewrite + candidate-callers subsection), this section, and
`CHANGELOG.md` Unreleased entry for #22. `caps.decl` / `deps.list` /
`manifest.pdxsig` unchanged.

Unblocks LE.M1-003 (v1.1.0 signed release tag): YES. The acceptance
criterion "signed manifest should not carry stale caller claims" is
satisfied by this pass — every caller row cites an at-HEAD grep
result. The tag itself is a separate mirror-push step
(`.plans/mirror-push.md`, not run by this pass).
