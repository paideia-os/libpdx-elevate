# libpdx-elevate — CHANGELOG

All notable changes to `libpdx-elevate` are recorded here.  The format
follows the paideia-os manifest convention: one section per released
version, newest first, dated by release-tag day.  Every entry is
authoritative for what the corresponding signed `manifest.pdxsig`
covers.

---

## 1.1.2 — 2026-09-02 — LE.M1-polish + LE.M2-hardening (13 fixes across encoder, tests, docs, correctness gaps)

v1.1.2 bundles two follow-up waves on top of v1.1.1 into a single
patch bump.  The polish wave (LE.M1-polish, issues #40-#47, eight
fixes) closes documentation / diagnostics / test-tag / encoder-
correctness gaps without changing the overall shape; the hardening
wave (LE.M2-hardening, issues #48-#52, five fixes) closes correctness
gaps found by an adversarial audit of the v1.1.1 send + acquire +
derive paths.  None of the fixes is source-breaking to a correctly-
written v1.1.1 consumer.  Everything from the v1.1.1-unreleased state
carries forward unchanged.

### LE.M1-polish — eight fixes (issues #40-#47)

- **#40 (G1)** — Alignment fix in
  `elevate_client_request_norealize` (`src/elevate_client.pdx`
  L294-372).  Four callee-save pushes leave rsp%16==8 (still
  misaligned); added `sub rsp, 8` after the pushes + `add rsp, 8`
  before every pop, matching the shape every other 4-push wrapper in
  this library uses (e.g. `elevate_client_send_req` L326-330).
  Behavior: nested calls to `elevate_request_write_frame`,
  `elevate_client_lookup_broker`, and `elevate_client_note` now see
  aligned rsp on both entry and exit.  Justification comment updated
  to state "keeps rsp%16 aligned across nested calls" instead of the
  misleading "keeps rsp%16 stable".

- **#41 (G5)** — Stripped embedded `\n\0` from every test-fingerprint
  string in favor of just `\0`.  `klog_s1` frames the newline itself
  (paideia-os#2226 pattern), so the tests were duplicating it.  Four
  files touched, eight strings shrunk by 1 byte each:
  `tests/elevate_client_cap_test.pdx` (`eccw_ok` / `eccw_fail`,
  L59-60), `tests/elevate_client_policy_test.pdx` (`ecpw_ok` /
  `ecpw_fail`, L42-43),
  `tests/elevate_client_outcome_test.pdx` (`ecow_ok` / `ecow_fail`,
  L86-87), `tests/elevate_client_require_test.pdx` (`ecqw_ok` /
  `ecqw_fail`, L67-68).  Array sizes updated to match (each shrunk
  by 1).

- **#42 (G6)** — `doc/libpdx-elevate.pdxdoc` refreshed from v1.0.0 to
  v1.1.2.  Added every missing SYNOPSIS entry: renamed
  `elevate_client_request_norealize`, `_ex_ctx`, `_acquire`,
  `_require` / `_require_scoped` / `_require_j`, `_journal_op`, every
  `elevate_client_cap_*` extension (`bind_expire_abs`, `bind_grant`,
  `get_grant`, `bind_scope`, `get_scope`, `bind_budget`,
  `get_budget_remaining`, `consume_budget`, `derive`, `bind_parent`,
  `get_parent`, `get_depth`), and `elevate_client_classify_outcome`
  / `_request_outcome`.  STATUS CODES section rewritten to cover
  every current band (base 0xFFFFEA00..0xFFFFEA4F, ELCA_
  0xFFFFEA50..0xFFFFEA6F, extension 0xFFFFEA70..0xFFFFEA7F including
  the new ELCC_ERR_ZERO_GRANT from #47) and the ELVC_OUTCOME_* enum
  namespace.

- **#43 (G7)** — New test file
  `tests/elevate_client_cap_m3_test.pdx` (`ElevateClientCapM3Test`
  module, fingerprint `LIBPDX-ELEVATE LE.M3 CAP OK`).  Twelve stages
  covering the LE.M3 client-side refusal paths that had no witness
  before: BAD_CTX (null `mint_ctx_ptr`), BAD_ROW (parent >= 16),
  PARENT_EXPIRED (unbound parent), NO_GRANT (parent grant unbound),
  CHILD_CAPS_ESCALATION (bit-wise superset), depth walker sanity
  (root == depth 0, chain 4->3->2->1 == depth 3),
  DELEGATION_TOO_DEEP (parent depth == 3 refuses the derive),
  ELCC_ERR_CYCLE_DETECTED (poked 5<->6 cycle triggers the walker's
  bounded-hop defense), ELCC_ERR_ZERO_GRANT sanity vs.
  ELCC_ERR_BAD_ROW discrimination (#47 witness), and parent-map
  cleanup on revoke (asserts stage-12 that the parent slot is 0 after
  `check_and_revoke` on a past-deadline row; enabled by the G3 fix in
  the LE.M2-hardening wave, #49 below).  Post-mint state (clamped
  deadline, child grant binding) is NOT asserted — those need a live
  kernel row and belong in the deferred LE.M4 end-to-end broker
  witness.  Labels prefixed `em3w_`, disjoint from every other
  witness prefix.

- **#44 (G10)** — Renamed `elcq_rs_*` labels to `elcq_sc_*` in
  `elevate_client_require_scoped` (`src/elevate_client_require.pdx`
  L370).  Both `elevate_client_require_reset` (L111) and
  `elevate_client_require_scoped` (L370) had claimed the `elcq_rs_`
  prefix and both defined `elcq_rs_out`, which compiled today only
  because paideia-as label scope is function-scope; a future scope-
  tightening to file-scope would have broken the build.  Renamed the
  scoped function's prefix to `elcq_sc_` (mnemonic: "scoped").
  Reset's `elcq_rs_*` labels are unchanged.

- **#45 (G11)** — README.md Callers section gained a new "Structural
  consumers (fail-closed today, awaiting broker-cap plumbing)"
  subsection listing mount.pdxfs (`mount_elev_require_system`),
  umount.pdxfs (`elevate_request_force_unmount`), and mkfs.pdxfs
  (`mkfs_elev_require_device_write`) with their current stub
  function names and their planned `elevate_client_acquire` /
  `elevate_client_require` call sites.  None is migrated yet; they
  are architecturally committed callers whose fail-closed refusal
  path already routes through this library's error taxonomy.

- **#46 (G12)** — STATUS.md `Followups for paideia-os (not blocking
  M3)` section rewritten to the post-M3 landing state.  Struck items
  that LANDED per CHANGELOG's R90-XREPO.011.M1-006 entry:
  paideia-os#2117 (sched_wait), #2118 (row_set_expire), #2119
  (/system/policy), #2121 (fail-closed policy), #2122 (broker
  daemon dispatch body).  Section header now points at CHANGELOG as
  the authoritative post-M3 landing state so the two documents no
  longer disagree on which kernel-side blockers remain.

- **#47 (G13)** — New distinct sentinel `ELCC_ERR_ZERO_GRANT`
  (0xFFFFEA73, extension band, next after `ELCC_ERR_CYCLE_DETECTED`
  = 0xFFFFEA71 and `ELCC_ERR_BAD_CTX` = 0xFFFFEA72).
  `elevate_client_cap_bind_grant` (`src/elevate_client_cap.pdx`
  L531-570) now returns this code instead of reusing
  `ELCC_ERR_BAD_ROW` on the `granted_caps == 0` gate.  A consumer
  can now distinguish "row_id out of range" (BAD_ROW) from "grant
  mask empty" (ZERO_GRANT); both are client-side program-defect
  gates but have different fixes, so a merged code was actively
  misleading.  Justification updated in-source; witness stage 11 in
  `tests/elevate_client_cap_m3_test.pdx` exercises both codes and
  asserts they are distinct.  Folds cleanly with the LE.M2-hardening
  #50 (G4) derive rollback path: bind_grant's ZERO_GRANT return
  triggers the child-row revoke + shadow scrub.

### LE.M2-hardening — five correctness fixes (issues #48-#52)

- **#48 G2 — recv_reply rdx clobber (`src/elevate_client_send.pdx`)**
  `elevate_client_recv_reply` pre-fix computed
  `deadline = hpet_now_ns() + rdx`, but rdx is caller-saved and
  hpet_now_ns's internal rdtsc/imul path writes it (the imul high
  half).  The recv loop then spun against a `(now + garbage)`
  deadline — practically forever when the garbage exceeded any
  realistic elapsed clock, effectively neutralising the caller-
  supplied timeout.  Fix: stash timeout_ns into the already-pushed r14
  (callee-save) before hpet_now_ns and consume from r14 in the add.
  No new error code; the observable change is that a broker that
  never replies now surfaces `ELVC_ERR_TIMEOUT` on the caller's
  actual bound rather than hanging.

- **#49 G3 — check_and_revoke shadow-map cleanup
  (`src/elevate_client_cap.pdx`)**
  `elevate_client_cap_check_and_revoke` pre-fix cleared only
  `_elevate_client_cap_expire_map[row_id]` on the revoked branch,
  leaving `_grant_map`, `_scope_map`, `_budget_map`, and
  `_parent_map` populated with the pre-revoke values.  A long-lived
  process that then re-minted into the same row_id (without
  `elevate_client_cap_reset`) inherited the previous grant's shadow
  bookkeeping, and `elevate_client_require` answered against a stale
  granted_caps mask.  Fix: scrub every shadow slot for the row on
  the revoke branch (same map set `elevate_client_cap_reset` walks),
  restoring the "row present <=> every shadow field freshly bound"
  invariant.

- **#50 G4 — derive discards bind_* return codes
  (`src/elevate_client_cap.pdx`)**
  `elevate_client_cap_derive` pre-fix called `bind_expire_abs`,
  `bind_grant`, and `bind_parent` back-to-back and ignored every
  return, so a failure at any of them (post-G13 `ELCC_ERR_ZERO_GRANT`,
  a stale `ELCC_ERR_PARENT_ALREADY_BOUND` when G3 hadn't been
  applied, etc.) left the kernel row minted but half-bound in the
  shadow maps.  Fix: check each bind_* return; on the FIRST non-OK,
  scrub every shadow slot for the just-minted child and issue a
  best-effort `elevate_channel_cap_revoke` before propagating that
  ORIGINAL error code upward.  The refuse-then-propagate shape is
  order-independent with respect to #47 (bind_grant's ZERO_GRANT
  check) — either landing first, the other still folds cleanly.

- **#51 G8 — recv_reply upper-bound gate
  (`src/elevate_client_send.pdx`)**
  `elevate_client_recv_reply` pre-fix compared the substrate's
  returned payload_len only against the PENDING_TAKE_EMPTY sentinel
  and the lower bound of 32 (`< 32 -> BAD_REPLY`).  A corrupted or
  substrate-bug-produced len above the frame max let the subsequent
  `[reply_buf+0]/[+8]/[+16]` reads run against an implausible buffer.
  Fix: add `cmp rax, 8192; ja elcs_rr_bad_reply_len` (frame max
  matches `src/kernel/core/ipc/frame.pdx MAX_FRAME_BYTES`) with a
  new refusal code `ELVC_ERR_BAD_REPLY_LEN = 0xFFFFEA07`, distinct
  from `ELVC_ERR_BAD_REPLY` (semantic mismatch of a validly-shaped
  APR) so an auditor can separate "APR was validly shaped but
  semantically wrong" from "APR frame length was implausible before
  we ever tried".

- **#52 G9 — acquire null-check on reply_buf
  (`src/elevate_client_acquire.pdx`)**
  `elevate_client_acquire` pre-fix dereferenced `[r15+8]` (granted_
  caps) and `[r15+16]` (expire_ns) without a null gate on reply_buf,
  so an accidental null caller argument would #PF inside the library
  rather than surface a diagnostic.  Fix: null gate right after the
  mint_ctx_buf gate, refusing with a new code `ELCA_ERR_BAD_REPLY_BUF
  = 0xFFFFEA51` — distinct from `ELCA_ERR_BAD_BUF` (which covers
  mint_ctx_buf) so the consumer can tell "you forgot the context"
  from "you forgot the reply buffer".

### Error-band footprint added by this wave

- `0xFFFFEA07 ELVC_ERR_BAD_REPLY_LEN` — LE.M2-hardening #51 (G8).
  First use of the 0x07..0x0B gap between the transport error codes
  and the M1-reserved RECV/SEND_FAIL aliases at 0x0C/0x0D.
- `0xFFFFEA51 ELCA_ERR_BAD_REPLY_BUF` — LE.M2-hardening #52 (G9).
  Extends the `ELCA_*` band by one slot; 0x52..0x5F remain reserved.
- `0xFFFFEA73 ELCC_ERR_ZERO_GRANT` — LE.M1-polish #47 (G13).
  Extension band, distinguishing `bind_grant(row, 0)` from
  `bind_grant(bad_row, mask)`; 0x74..0x7F remain reserved.

### Dependency deltas

None.  `libpdx-cap` (still PENDING, blocks M2-001) and `libpdx-audit`
(still PENDING, blocks M2-002) statuses unchanged from v1.1.1.

---

## Unreleased — v1.1.1 (LE.M3-001 real-mint-args pass on top of the v1.1.0 wave, no signed tag yet)

v1.1.1 is a PATCH over the v1.1.0-unreleased state on `main`.  It
carries one narrow behavioural fix — `elevate_client_cap_derive` now
accepts a caller-owned `mint_ctx_ptr` and mints under the caller's
real broker cap slot, instead of the v1.1.0 xor-zero placeholders that
made the LE.M3-001 landing non-functional end-to-end (kernel-side
`kind_elevate_channel.pdx` indexes the passed slot straight into the
flat 256-slot cap_table with only a `KIND_IPC_ENDPOINT` check, so any
caller whose broker cap did not accidentally live at slot 0 got
`ELCC_ERR_MINT_FAIL`).  Source-breaking to any consumer that called
the 3-arg derive form — but the v1.1.0 3-arg form was documented but
non-functional, so no working consumer of it can exist.  See the
LE.M3-001 row below and the derive header in
`src/elevate_client_cap.pdx` for the full mint_ctx_ptr layout.

Everything from the v1.1.0-unreleased wave carries forward unchanged.
The v1.1.0 label is retired without a signed tag; v1.1.1 supersedes
it on `main` and the release step (`.plans/mirror-push.md`) will
tag/sign v1.1.1 directly.

Two waves rolled into a single v1.1.x minor bump — none source-breaking,
all additive over the v1.0.0 API (with the v1.1.1 exception noted
above for the `_derive` signature only):

* **M6 (enhancement wave):** issues #11–#17, all LANDED on `main`.
  Credential-shaped surface (`elevate_client_acquire` → `_require`),
  scope + budget binding, per-op audit record, retirement of the
  fail-open `ELVC_STUB`-means-proceed sentinel, and the initial
  explicit-context (`_ex_ctx`) variant.
* **LE.M1-M3 (multilevel-chain foundations):** issues #19–#28 + #37–#38,
  landed selectively (see per-ticket table below).  M3 is the
  substantive core — `elevate_client_cap_derive`, parent-row shadow map,
  depth-bounded delegation.  M1/M2 landed partially with the remainder
  documented as deferred to the follow-up wave.

See `STATUS.md`'s M6 close-out and the LE.M1-M3 table below for the
complete state.  Cutting the v1.1.0 tag + signed manifest is a
separate release step (`.plans/mirror-push.md`), not part of this
wave.

### LE.M1-M3 landings (this pass, 2026-09-02)

Follow-up enhancement wave filed under milestones LE.M1-polish,
LE.M2-hardening, LE.M3-multilevel.  Scope was narrowed to the
multilevel-chain foundations that other consumers depend on; M4-M7
(reap, attestation, audit sink, revoke-cascade) explicitly deferred
to a future wave.  Full per-ticket state:

| ID       | #  | State     | Notes                                                                                                                                     |
|----------|----|-----------|-------------------------------------------------------------------------------------------------------------------------------------------|
| M1-001   | 19 | DEFERRED  | Unit-test coverage for send/acquire/journal/retry — 4 new witnesses; carved out to keep this landing tight.                              |
| M1-002   | 20 | DEFERRED  | Explicit-context variants for `_ex_j` / `_ex_r` / `_acquire` — deferred until a concurrent consumer surfaces (see .plans/enhancement-plan.md §4 rationale). |
| M1-003   | 21 | DEFERRED  | Cut v1.1.0 signed release tag — a separate release step per `.plans/mirror-push.md`.                                                     |
| M1-004   | 22 | DEFERRED  | README/STATUS Callers list re-verification — needs external repo (rm, pkg) audit pass; scheduled with the release tag.                   |
| M1-005   | 37 | DEFERRED  | Result-band collision defence witness — carved out to keep the landing tight; the encoder already refuses duplicate `pub let` bindings.  |
| M2-001   | 23 | BLOCKED   | Swap `elevate_client_cap_narrow_stub` to real `libpdx-cap.cap_narrow_rights` — libpdx-cap.M2 has NOT landed; entry still stubbed as documented in `deps.list`. |
| M2-002   | 38 | BLOCKED   | Swap `journal_req/_apr/_op` bodies to libpdx-audit — `libpdx-audit` HEAD carries `!{mem, sysreg} @{cap, sched}` on its trio, which would force a v2.0 cap-manifest ripple on every downstream tool. Filed cross-repo request for a thin `!{mem}` wrapper on the libpdx-audit side; swap unblocks the moment that lands. |
| M2-003   | 24 | BLOCKED   | End-to-end broker witness against paideia-os live daemon — needs the broker dispatch body upstream (paideia-os#2122).                    |
| M2-004   | 25 | DEFERRED  | Fail-closed audit-first invariant regression witness — carved out; the invariant is enforced today at every REQ-journal call site by the audit-first sequence in `elevate_client_request_ex_j`. |
| M3-001   | 26 | **LANDED (v1.1.1 fix)** | `elevate_client_cap_derive(parent_row, child_caps, child_dur_ns, mint_ctx_ptr) -> child_row_id`. `mint_ctx_ptr` (v1.1.1) points at a caller-owned 3-word / 24-byte buffer — `[+0]=parent_ep_slot`, `[+8]=request_id`, `[+16]=requester_pid` — layout-compatible with the FIRST three words of `elevate_client_acquire`'s `mint_ctx_buf` so a consumer holding an acquire ctx may pass its same address here. The mint now actually uses the caller's broker-endpoint cap slot; v1.1.0's xor-zero placeholders made the feature return `ELCC_ERR_MINT_FAIL` end-to-end. Populates the child's shadow deadline (clamped `min(now+dur, parent_expire)`), grant map (subset-checked against parent, else `ELCC_ERR_CHILD_CAPS_ESCALATION 0xFFFFEA3B`), and parent-edge. Refuses on `ELCC_ERR_BAD_CTX 0xFFFFEA72` (null ctx), `ELCC_ERR_PARENT_EXPIRED 0xFFFFEA3C`, `ELCC_ERR_NO_GRANT 0xFFFFEA38`, `ELCC_ERR_DELEGATION_TOO_DEEP 0xFFFFEA3F`. Bumps `ELCC_ST_DERIVES = 5`. |
| M3-002   | 27 | **LANDED** | Parent-row shadow map (`_elevate_client_cap_parent_map`, 16 slots) + `elevate_client_cap_bind_parent`, `_get_parent`, `_get_depth`. `bind_parent` refuses `ELCC_ERR_PARENT_SELF_LOOP 0xFFFFEA3D` (r,r) and `ELCC_ERR_PARENT_ALREADY_BOUND 0xFFFFEA3E` (rebind attempt). `_get_depth` is a bounded walk returning `ELCC_ERR_CYCLE_DETECTED 0xFFFFEA71` (extension band) if the map is corrupted past MAX_DELEGATION_DEPTH hops. |
| M3-003   | 28 | **LANDED** | `ELCC_MAX_DELEGATION_DEPTH = 4` (root + up to 3 derived hops). Enforced in `elevate_client_cap_derive` before mint via `_get_depth(parent) < MAX - 1`. Widening later is backward-compatible; narrowing is not. |

Deferred M4-M7 tickets (#29-#36) are commented on their respective
GitHub issues with a pointer to this v1.1.0 release; they will be
picked up in the follow-up Phase 2 wave when other consumers surface
a need.  See:

- #29 LE.M4-001 per-cap-mask duration ceilings on validator
- #30 LE.M4-002 `elevate_client_cap_reap_expired` — idle-time sweep
- #31 LE.M5-001 delegation-with-attestation record (`ELVJ_EVT_ATTEST`)
- #32 LE.M5-002 attestation-required rows (`bind_attestation_required` flag)
- #33 LE.M6-001 signed audit sink to `/system/audit/elevate.log`
- #34 LE.M6-002 per-handle `last_audit_seq` + `last_audit_kind` attribution surface
- #35 LE.M7-001 `elevate_client_cap_revoke_cascade` over child chain
- #36 LE.M7-002 broker-triggered revoke propagation via `drain_broker_exp`

### Error-band footprint added by this pass

- `0xFFFFEA3B..0xFFFFEA3F` fill out the last five slots of the
  `ElevateClientCap` band with LE.M3 codes (see per-code table in
  `src/elevate_client_cap.pdx` header).
- `0xFFFFEA70..0xFFFFEA7F` opens an EXTENSION band; the v1.1.1
  patch adds `0xFFFFEA72 ELCC_ERR_BAD_CTX` (null `mint_ctx_ptr`)
  alongside `0xFFFFEA71 ELCC_ERR_CYCLE_DETECTED`.  Remainder reserved
  for the eventual M7 revoke-cascade codes.

### Dependency deltas

- `libpdx-cap` — status unchanged (PENDING, blocking M2-001).
- `libpdx-audit` — status unchanged (PENDING, blocking M2-002).  This
  pass adds a rationale note in `src/elevate_client_journal.pdx`
  header explaining the cap-manifest ripple that keeps the swap
  deferred; a companion issue on libpdx-audit will request a thin
  `!{mem}`-only wrapper.

- **ENH-005 (#12)** — `elevate_client_request` **renamed** to
  `elevate_client_request_norealize` (`elevate_client.pdx`); the old
  name no longer exists (source-breaking, intentional — see the
  file's WHY THE RENAME note).  `ELVC_STUB` renamed to
  `ELVC_NOT_DISPATCHED` (same value, `0xFFFFEA00`).  This retires the
  fail-open/fail-closed coin-flip documented in the enhancement plan
  §2: two real consumers (`rm`, `pkg`) called the same M1 skeleton and
  reached opposite dispositions from its return value.
- **ENH-001 (#11)** — `elevate_client_acquire` (new
  `elevate_client_acquire.pdx`): composes `elevate_client_request_ex_r`
  with a mint + shadow-bind into one call returning a `row_id` HANDLE.
  New `elevate_client_cap.pdx` primitives:
  `elevate_client_cap_bind_expire_abs` (shadow an absolute deadline
  without re-adding `hpet_now_ns`) and `elevate_client_cap_bind_grant`
  / `_get_grant` (shadow the APR's `granted_caps` per row, band
  `ELCC_ERR_NO_GRANT = 0xFFFFEA38`). New error band
  `0xFFFFEA50..0xFFFFEA5F` (`ELCA_*`).
- **ENH-002 (#13)** — `elevate_client_require` (new
  `elevate_client_require.pdx`): per-op re-assert against a handle
  from `elevate_client_acquire`, no broker hop. Checks freshness
  (`elevate_client_cap_check_and_revoke`) then `(needed & granted) ==
  needed` against the shadow grant map. New error band
  `0xFFFFEA60..0xFFFFEA6F` (`ELCA_ERR_EXPIRED`,
  `ELCA_ERR_CAPS_INSUFFICIENT`).
- **ENH-004 (#16)** — `elevate_client_journal_op`
  (`elevate_client_journal.pdx`, `ELVJ_EVT_OP = 3`,
  `ELVJ_ERR_OP_JOURNAL_FAIL = 0xFFFFEA43`) and `elevate_client_require_j`
  (`elevate_client_require.pdx`): one audit record per authorized
  per-op use of an acquired handle, making grant amplification (1
  REQ/APR pair, N mutating uses) legible to an auditor.
- **ENH-003 (#15)** — `elevate_client_require_scoped`
  (`elevate_client_require.pdx`, `ELCA_ERR_SCOPE_MISMATCH =
  0xFFFFEA62`) plus new `elevate_client_cap.pdx` shadow bindings:
  `bind_scope`/`get_scope` (opaque per-handle scope fingerprint) and
  `bind_budget`/`get_budget_remaining`/`consume_budget`
  (`ELCC_ERR_BAD_BUDGET = 0xFFFFEA39`, `ELCC_ERR_BUDGET_EXHAUSTED =
  0xFFFFEA3A`) — a bounded op count per handle, stored as `remaining +
  1` so exhaustion cannot be mistaken for "never bound". Both bindings
  are additive over plain `elevate_client_require`; a handle that never
  binds them behaves exactly as it did before v1.1.0.
- **ENH-006 (#17, scoped)** — `elevate_client_request_ex_ctx`
  (`elevate_client_send.pdx`, `ELVC_ERR_BAD_CTX = 0xFFFFEA06`, `ELVC_CTX_*_OFF`
  layout constants): explicit-context twin of `elevate_client_request_ex`
  reading `target_fp_lo` / fast / human / explicit timeout from a
  caller-owned `ctx_buf` instead of the three process-global mutable
  slots, so concurrent flows in one process do not race each other.
  Deliberately scoped to the core primitive only; `_ex_j` / `_ex_r` /
  `elevate_client_acquire` and the globals themselves are unchanged —
  full singleton retirement is deferred until a concurrent consumer
  needs it.
- **R90-XREPO.011.M1-006 (#18)** — client-side ALLOW / DENY / TIMEOUT
  reconciliation (new `elevate_client_outcome.pdx`, module
  `ElevateClientOutcome`). Reduces the open-ended `ELV*_ERR_*` taxonomy
  the transport / retry layers speak in to three enum variants
  (`ELVC_OUTCOME_ALLOW = 0`, `ELVC_OUTCOME_DENY = 1`,
  `ELVC_OUTCOME_TIMEOUT = 2`, distinct from every status band since all
  `ELV*_ERR_*` are ≥ `0xFFFFEA00`). Applies the org-wide fail-closed
  policy (paideia-os#2121) at the client boundary: only `ELVC_OK`
  surfaces as `ALLOW`, only `ELVC_ERR_TIMEOUT` and `ELVR_ERR_EXHAUSTED`
  surface as `TIMEOUT`, everything else — broker DENY (whichever wire
  shape it takes: `BAD_EXPIRE` / `GRANT_INVALID` / `BAD_REPLY`), packer
  / journal / policy failures, unknown values — folds to `DENY`.
  Landed the last kernel-side blockers unblock this: paideia-os#2117
  (sched_wait), #2118 (`elevate_channel_row_set_expire`), #2119
  (`/system/policy` format + seed), #2121 (fail-closed policy), #2122
  (broker daemon dispatch body). Surface: `elevate_client_classify_
  outcome(status)` (pure reduction, leaf) and `elevate_client_request_
  outcome(...)` (convenience composition over `elevate_client_request_
  ex`). Witness at `tests/elevate_client_outcome_test.pdx` drives all
  three variants across every representative input from every error
  band, plus a `0xDEADBEEF` unknown-value probe for the fail-closed
  default.
- **ENH-007 (#14)** — README/STATUS enforcement-status accuracy pass.
  Retired the README example claiming `ELVC_OK` or `ELVC_STUB` "both
  mean proceed" (the exact bug ENH-005 fixed) in favor of the
  `elevate_client_acquire` + `elevate_client_require` credential-shaped
  pattern. Marked `rm` as needing migration off the renamed entry point
  and `pkg` as needing only the name update. Added a `v1.1.0` section
  to the README's Version block and an M6 close-out table to
  `STATUS.md` stating plainly what changed and what is still deferred
  (broker daemon body, full singleton retirement, pending dep swaps).

---

## 1.0.0 — 2026-08-22 — R49 shared library, first signed release

First 1.0 release.  Establishes the client-side elevate protocol helper
that every R49/R50 tool imports for per-op capability requests.

### Wire + client shape (M1)

- **elevate_request.pdx** (#1) — mirror of the R48.M7 `ElevateChannel`
  wire schema.  ELV_* constants, REQ frame layout offsets,
  `elevate_request_cap_mask_valid`, `elevate_request_duration_valid`,
  `elevate_request_pack_op_word` (mirror of kernel `elv_pack_req`),
  `elevate_request_write_frame` (32-byte REQ into caller buffer).
- **elevate_client.pdx** (#2) — `svc.elevate-broker` binding.
  `elevate_client_lookup_broker` (wraps `svc_lookup`), an eight-slot
  stats table shape-matched to `ElevateBroker._elevate_broker_stats`,
  and the M1 send-and-block skeleton `elevate_client_request`.

### Core flow (M2)

- **elevate_client_policy.pdx** (#3) — client-side mirror of the
  kernel `ElevatePolicy` table (16-row × 48-byte, magic `PDXP`,
  install/check/hit primitives with `ep_check`/`ep_hit_row` match
  semantics).  Drives the fast-path classifier for
  `elevate_client_request_ex`.
- **elevate_client_send.pdx** (#4) — real send + bounded recv:
  configurable human/fast timeouts (defaults 30 s / 500 ms), inline
  IPC frame-header packer, `elevate_client_send_req`,
  `elevate_client_recv_reply` (bounded polling on
  `endpoint_take_pending` via `hpet_now_ns` deadline),
  `elevate_client_check_grant` (subset validation mirroring
  `elv_check_grant`), process-global `target_fp_lo` getter/setter,
  and `elevate_client_request_ex` — the full-flow entry point
  (policy consult → REQ assemble → REQ send → APR recv → grant
  subset check).
- **elevate_client_cap.pdx** (#5) — `Cap<KIND_ELEVATE_CHANNEL = 0x191>`
  wrapper with bounded lifetime.  Thin mint wrapper, client-side
  shadow deadline map (16 slots, one per row_id) to compensate the
  kernel default `expire_ns = 0`, `bind_expire` / `expired` /
  `check_and_revoke` (idempotent, `REVOKE_ALREADY` collapses to
  `STATE_REVOKED`), and `elevate_client_cap_narrow_stub` placeholder
  awaiting libpdx-cap.M2 `cap_narrow_rights`.

### Audit + retry (M3)

- **elevate_client_journal.pdx** (#6) — REQ + APR journal through the
  kernel's `uej_append` (UEJ_KIND_ELEVATE = 5).  `journal_req` /
  `journal_apr` primitives + `elevate_client_request_ex_j` audit-first
  wrapper: fetch actor_fp_lo → journal REQ → run elevate → journal APR
  on success.  REQ-journal failure aborts before the wire hop (D3
  audit-first discipline).  APR-journal failure after grant surfaces
  `ELVJ_ERR_APR_JOURNAL_FAIL` so the caller can revoke the grant.
- **elevate_client_retry.pdx** (#7) — retry-with-backoff over
  `elevate_client_request_ex_j`.  Retriable set is
  `{ELVC_ERR_TIMEOUT, ELVC_ERR_SEND_FAIL, ELVC_ERR_LOOKUP_FAIL}`.
  Bounds [1, 8] attempts (default 3); 100 ms → 400 ms → 1.6 s
  (capped) backoff; every attempt writes its own REQ audit record.
  `elevate_client_request_ex_r` is the full-featured entry point.

### Tests + smoke (M4)

- **tests/elevate_client_policy_test.pdx** (#8) — 15-stage boot
  witness `elevate_client_policy_witness`, fingerprint
  `LIBPDX-ELEVATE M4 POLICY OK`.  Wildcard/specific hits, subset/dur/
  target misses, install refusals (caps==0, reserved bits, dur==0,
  dur>1h), rate-limit exhaustion (row rate=5, sixth hit → RATE_LIMIT
  + RATE_DENIES bump), out-of-range hit(idx=16) → BAD_ARG.
- **tests/elevate_client_cap_test.pdx** (#9) — 12-stage boot witness
  `elevate_client_cap_witness`, fingerprint `LIBPDX-ELEVATE M4 CAP OK`.
  Bind/get happy path, row_id=17 refusals, expired discrimination
  (live / no-deadline / bad-row), past-deadline enforcement with
  direct slot poke (1-ns absolute deadline vs `hpet_now_ns`) →
  kernel `elevate_channel_cap_revoke` invocation and
  AUTO_REVOKES+REVOKE_FAILS sum == 1 invariant.

### Signed release + docs + mirror (M5)

- **caps.decl** — capability manifest declaring floor caps for every
  entry point.  Documentary at library level; the kernel cap-check
  runs against the *consuming process's* cap set at exec time.
- **deps.list** — cross-repo dependencies.  `libpdx-cap >=0.2.0`
  (PENDING, narrow_stub swap site) and `libpdx-audit >=0.3.0`
  (PENDING, journal_req/_apr swap site).  Kernel primitives
  enumerated with paideia-os issue references.
- **doc/libpdx-elevate.pdxdoc** — I7 documentation (per
  design/tooling/plan.md §4.2 I7).  Overview, invariants, per-entry
  API reference, examples, POSIX-difference annotation
  (`sudo`/`setuid` → per-op elevate protocol), cross-references
  into paideia-os `design/user/model.md` §5.
- **manifest.pdxsig** — dual-signed release manifest per D4 (§6 of
  design/tooling/plan.md): author key + Paideia root key.  Signature
  bytes carry the paideia-as v0.33-crypto-kdf stub marker until the
  toolchain crypto tag is reachable; the manifest body (name,
  version, source-tree hash, caps.decl hash, deps.list hash,
  changelog hash) is finalized and hash-stable, so a `paideia-as
  release --sign` re-run against v0.33-crypto produces the same
  message digest and only the ML-DSA-65 signature blocks need to
  populate.
- **.plans/mirror-push.md** — runbook for pushing this release to
  `pkgs.paideia-os/main/libpdx-elevate/1.0.0/` once the mirror
  exists (D4 / §6.3).  Documents the source-side layout, the
  bootstrap ordering (libpdx-elevate ships before pkg's M5 so that
  pkg can consume the elevate helper at its own M5 install-flow
  landing), and the two-key sign-off procedure.

### KIND ordinal footprint

- Consumes `KIND_ELEVATE_CHANNEL = 0x191` (kernel-allocated at
  paideia-os #1626).  Introduces no new kernel KIND at v1.0.

### Error bands (unchanged since M3, restated for auditor convenience)

| Band                    | Purpose                                    |
|-------------------------|--------------------------------------------|
| `0xFFFFEA01..0xFFFFEA05`| ElevateClient transport                    |
| `0xFFFFEA10..0xFFFFEA1F`| ElevateClientPolicy                        |
| `0xFFFFEA20..0xFFFFEA2F`| ElevateClientRetry                         |
| `0xFFFFEA30..0xFFFFEA3F`| ElevateClientCap                           |
| `0xFFFFEA40..0xFFFFEA4F`| ElevateClientJournal                       |

### Known deferred paths (documented in STATUS.md §"Followups")

- Kernel `elevate_channel_row_set_expire` API — would let the shadow
  deadline map in `elevate_client_cap.pdx` collapse back into the
  row's `[+32]` slot.
- Broker daemon body (`ELVB_DISPATCH_STUB` in paideia-os
  `src/kernel/core/ipc/elevate_broker.pdx`) — until it consumes REQ
  frames and produces APR replies, the full loop reliably times out.
  The retry wrapper turns that into a bounded three-attempt sequence.
- R51 scheduler-wait syscall — `elevate_client_retry_delay` will
  collapse to a thin wrapper once available.
- libpdx-cap.M2 `cap_narrow_rights` — retires
  `ELCC_NOTE_NO_NARROW`; v1.1.0 bump.
- libpdx-audit.M2 `audit_begin` / `audit_record_output` /
  `audit_commit` — retires the direct `uej_append` call in
  `elevate_client_journal_req/_apr`; v1.1.0 bump.
