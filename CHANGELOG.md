# libpdx-elevate — CHANGELOG

All notable changes to `libpdx-elevate` are recorded here.  The format
follows the paideia-os manifest convention: one section per released
version, newest first, dated by release-tag day.  Every entry is
authoritative for what the corresponding signed `manifest.pdxsig`
covers.

---

## Unreleased — v1.1.0 in progress — hardening the elevate gate

Post-1.0.0 enhancement wave (`.plans/enhancement-plan.md`).  The
1.0.0 client half was complete but advisory-only: nothing forced a
consumer to present a grant before performing the privileged
operation it gated.  v1.1.0 adds a credential-shaped surface
alongside the status-shaped one.

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
