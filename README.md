# libpdx-elevate

paideia-os shared library: client-side elevate protocol helper.

## Purpose

PaideiaOS has no `sudo` and no setuid bit. A tool that needs to touch a
system path does not *become* privileged; it asks a broker for a
**bounded-lifetime capability**, uses it, and lets it self-invalidate.
`libpdx-elevate` is the client half of that exchange: it mirrors the
R48.M7 `ElevateChannel` wire schema, resolves the broker by name,
assembles and publishes the request frame, waits for the approval frame
under a deadline, checks that what was granted is a subset of what was
asked, journals both halves to the audit trail, mints and expires the
resulting `Cap<KIND_ELEVATE_CHANNEL = 0x191>`, and retries the transport
when the broker is transiently unavailable.

The server half is `svc.elevate-broker` — a paideia-os kernel component
(`src/kernel/core/ipc/elevate_broker.pdx`) fronting the authoritative
`ElevatePolicy` table. This library neither implements nor substitutes
for it: the broker stays the sole enforcement point for auto-approval and
the sole minter of grants. The policy table this library carries
(`elevate_client_policy.pdx`) is a **cached pre-classification only**,
used to pick a recv deadline — a hit selects the fast timeout (500 ms,
the auto-approve expectation), a miss the human timeout (30 s, the
founder-in-the-loop expectation). A hit never skips the broker hop, and
drift is safe in both directions: a rule here but not in the kernel means
the client tries the fast path and the broker falls back to human
approval; a rule in the kernel but not here means the client
pessimistically assumes human approval.

Wire constants are redeclared here rather than linked from the kernel
`.o` graph, so a downstream tool can link this library without linking
every `ipc/*.o`. Each redeclaration names its kernel origin in-source,
and the M4 smoke matrix compares packer output against `elv_pack_req`.

## API surface

### `src/elevate_request.pdx` — module `ElevateRequest`

REQ-frame builder and the client-side mirrors of the kernel validators.

| Signature | Purpose |
| --- | --- |
| `elevate_request_cap_mask_valid(mask) -> u64 !{} @{}` | `1` iff `mask != 0` and no reserved bits (8..63) set; mirror of kernel `elv_mask_valid`. |
| `elevate_request_duration_valid(dur) -> u64 !{} @{}` | `1` iff `dur ∈ [1 s, 1 h]`; mirror of kernel `elv_dur_valid`. |
| `elevate_request_pack_op_word(caps, dur) -> u64 !{} @{}` | Pack `op \| (caps << 8)`; returns `ELV_ERR_BAD_MASK` / `ELV_ERR_BAD_DUR` on gate failure. Mirror of `elv_pack_req`. |
| `elevate_request_write_frame(buf, caps, dur) -> u64 !{mem} @{}` | Assemble the full 32-byte REQ frame into `buf`. `buf` is left untouched on any error (no partial write). |

### `src/elevate_client.pdx` — module `ElevateClient`

Broker name binding, client stats, and the M1 send-and-block skeleton.

| Signature | Purpose |
| --- | --- |
| `elevate_client_stats_reset() -> () !{mem} @{}` | Zero the eight-slot client counter table. Boot/teardown only. |
| `elevate_client_note(which) -> () !{mem} @{}` | Bounded increment of counter slot `which`; out-of-range is a no-op. |
| `elevate_client_stat(which) -> u64 !{mem} @{}` | Bounded read of counter slot `which`; out-of-range returns `0`. |
| `elevate_client_lookup_broker() -> u64 !{mem} @{}` | Resolve `"svc.elevate-broker"` via `svc_lookup`; returns the endpoint id or `SVC_LOOKUP_NONE`. |
| `elevate_client_request_norealize(caps, dur, req_buf, reply_buf) -> u64 !{mem} @{}` | **Never dispatches.** Gate buffers, assemble REQ, resolve broker, stop and return `ELVC_NOT_DISPATCHED`. Renamed at ENH-005 (#12) from `elevate_client_request` — that name is gone; do not gate a privileged operation on this return. Use `elevate_client_request_ex` or `elevate_client_acquire` instead. |

### `src/elevate_client_policy.pdx` — module `ElevateClientPolicy`

16-row × 48-byte cached policy table; mirror of the kernel `ElevatePolicy`
match semantics.

| Signature | Purpose |
| --- | --- |
| `elevate_client_policy_reset() -> () !{mem} @{}` | Scrub the 96-slot table and the 8-slot stats. Idempotent. |
| `elevate_client_policy_count() -> u64 !{mem} @{}` | Installed row count (stats slot 0). |
| `elevate_client_policy_stat(which) -> u64 !{mem} @{}` | Bounded stats read; out-of-range returns `0`. |
| `elevate_client_policy_row_va(idx) -> u64 !{mem} @{}` | Byte address of row `idx` (stride 48); `0` when out of range. |
| `elevate_client_policy_install(target_fp_lo, caps, max_dur, rate_per_hour) -> u64 !{mem} @{}` | Append a rule; returns the assigned index or `ELCP_ERR_BAD_CAPS` / `_BAD_DUR` / `_ENOSPC`. |
| `elevate_client_policy_check(target_fp_lo, caps, dur) -> u64 !{mem} @{}` | First matching row index, or `ELCP_ERR_NO_MATCH`. Match = enabled ∧ (target 0 or equal) ∧ `caps ⊆ row.caps` ∧ `dur ≤ row.max_dur`. |
| `elevate_client_policy_hit(idx) -> u64 !{mem} @{}` | Register a hit on a matched row; refuses `ELCP_ERR_RATE_LIMIT` past the row's cap (`rate_per_hour == 0` means unlimited). |

### `src/elevate_client_send.pdx` — module `ElevateClientSend`

The real transport half: header packing, publish, bounded recv, grant
validation, and the M2 full-flow entry point.

| Signature | Purpose |
| --- | --- |
| `elevate_client_set_human_timeout(ns) -> () !{mem} @{}` / `elevate_client_set_fast_timeout(ns) -> () !{mem} @{}` | Set the human-approve / fast-path default; `0` restores 30 s resp. 500 ms. |
| `elevate_client_get_human_timeout() -> u64 !{mem} @{}` | Read the human-approve default, resolving the constant when unset. |
| `elevate_client_get_fast_timeout() -> u64 !{mem} @{}` | Read the fast-path default, resolving the constant when unset. |
| `elevate_client_set_target_fp_lo(fp) -> () !{mem} @{}` | Bind the process-global user fingerprint used by the policy consult and the audit actor field. |
| `elevate_client_get_target_fp_lo() -> u64 !{mem} @{}` | Read that process-global fingerprint. |
| `elevate_client_pack_hdr(reply_ep_id, payload_len) -> u64 !{} @{}` | Pack the 8-byte IPC frame header, byte-identical to `frame_encode`. |
| `elevate_client_send_req(broker_ep_id, reply_ep_id, req_buf) -> u64 !{mem} @{}` | Publish the 32-byte REQ via `endpoint_write_pending`; any refusal collapses to `ELVC_ERR_SEND_FAIL`. |
| `elevate_client_recv_reply(reply_ep_id, reply_buf, timeout_ns) -> u64 !{mem} @{boot}` | Bounded poll of `endpoint_take_pending` against an `hpet_now_ns` deadline; validates op == APR and `expire_ns != 0`. |
| `elevate_client_check_grant(reply_buf, requested_caps) -> u64 !{mem} @{}` | `(granted & requested) == granted`; mirror of kernel `elv_check_grant`. |
| `elevate_client_request_ex(caps, dur, req_buf, reply_ep_id, reply_buf, timeout_ns) -> u64 !{mem} @{boot}` | Full M2 flow: policy consult → REQ assemble → broker lookup → send → recv → grant-subset check. `timeout_ns == 0` selects fast/human by policy classification. |

### `src/elevate_client_acquire.pdx` — module `ElevateClientAcquire`

Composed flow returning a grant **handle** (`row_id`) instead of a bare
status — ENH-001 (#11), the structural answer to "the elevate gate is
advisory, not a credential." Callers thread the returned `row_id` into
the privileged operation and re-assert it per-op via
`elevate_client_require` (below); omitting the gate now becomes a
missing argument, not a skipped statement.

| Signature | Purpose |
| --- | --- |
| `elevate_client_acquire(caps, dur, req_buf, reply_ep_id, reply_buf, mint_ctx_buf) -> u64 !{mem} @{boot, cap}` | Full `request_ex_r` flow, then mint + shadow-bind (expire + granted caps). Returns `row_id` (`< 16`) on success; every failure passes through the ORIGINAL band from whichever layer refused, or `ELCA_ERR_BAD_BUF` if `mint_ctx_buf == 0`. `mint_ctx_buf` is a 40-byte, 5-word caller buffer: `parent_ep_slot`, `request_id`, `requester_pid`, `target_cap_kind`, `target_cap_rights` (see the file header for the exact layout). |

### `src/elevate_client_require.pdx` — module `ElevateClientRequire`

Per-op re-assert against an `elevate_client_acquire` handle — ENH-002
(#13). No broker hop: a bounded local freshness check plus a
caps-subset check against the shadow grant map. A recursive walk over
N objects calls this once per item; the cost is a clock read.

| Signature | Purpose |
| --- | --- |
| `elevate_client_require_reset() -> () !{mem} @{}` | Scrub the 8-slot require stats. Boot/teardown only. |
| `elevate_client_require_note(which) -> () !{mem} @{}` / `elevate_client_require_stat(which) -> u64 !{mem} @{}` | Bounded stats increment / read; out of range is a no-op resp. `0`. |
| `elevate_client_require(row_id, needed_caps) -> u64 !{mem} @{cap, boot}` | Freshness (`elevate_client_cap_check_and_revoke`) then `(needed & granted) == needed` against the shadow grant map. `ELCA_ERR_EXPIRED` / `ELCA_ERR_CAPS_INSUFFICIENT` on refusal; `ELCC_ERR_BAD_ROW` / `_NO_DEADLINE` / `_NO_GRANT` pass through as caller-usage errors. |
| `elevate_client_require_j(actor_fp_lo, row_id, needed_caps) -> u64 !{mem} @{cap, boot}` | ENH-004: audit-wrapped `elevate_client_require` — journals one per-op record (`elevate_client_journal_op`) on every AUTHORIZED use, none on a refusal. The per-op record count against one REQ/APR pair is the grant-amplification signal. |

### `src/elevate_client_cap.pdx` — module `ElevateClientCap`

`Cap<KIND_ELEVATE_CHANNEL>` mint plus client-side bounded-lifetime
self-invalidation.

| Signature | Purpose |
| --- | --- |
| `elevate_client_cap_reset() -> () !{mem} @{}` | Scrub the 16-slot shadow deadline map and the stats table. |
| `elevate_client_cap_note(which) -> () !{mem} @{}` / `elevate_client_cap_stat(which) -> u64 !{mem} @{}` | Bounded stats increment / read; out of range is a no-op resp. `0`. |
| `elevate_client_cap_bind_expire(row_id, dur_ns) -> u64 !{mem} @{boot}` | Record `hpet_now_ns() + dur_ns` as the row's shadow deadline; `dur_ns == 0` clears it. |
| `elevate_client_cap_get_expire(row_id) -> u64 !{mem} @{}` | Read the shadow deadline; `0` means unbound or out of range. |
| `elevate_client_cap_bind_expire_abs(row_id, deadline_ns) -> u64 !{mem} @{}` | ENH-001: store an ABSOLUTE deadline (no `hpet_now_ns` addition) — the counterpart `elevate_client_acquire` uses to shadow a broker-supplied `expire_ns` without re-deriving it against a later clock read. |
| `elevate_client_cap_bind_grant(row_id, granted_caps) -> u64 !{mem} @{}` / `elevate_client_cap_get_grant(row_id) -> u64 !{mem} @{}` | ENH-001: shadow / read the APR's `granted_caps` for `row_id`, so `elevate_client_require` can re-check caps coverage with no broker hop. `0` means never bound (`ELCC_ERR_NO_GRANT`). |
| `elevate_client_cap_narrow_stub(rights_in) -> u64 !{} @{}` | **Stub** awaiting libpdx-cap.M2 `cap_narrow_rights`; returns rights unchanged. |
| `elevate_client_cap_mint(parent_ep_slot, rights, request_id, requester_pid, target_cap_kind, target_cap_rights) -> u64 !{mem} @{cap}` | Wrapper over the kernel `elevate_channel_cap_mint_inner`; returns `row_id` (< 16) or `ELCC_ERR_MINT_FAIL`. |
| `elevate_client_cap_expired(row_id) -> u64 !{mem} @{boot}` | `1` expired / `0` live, or `ELCC_ERR_BAD_ROW` / `ELCC_ERR_NO_DEADLINE`. |
| `elevate_client_cap_check_and_revoke(row_id) -> u64 !{mem} @{cap, boot}` | Self-invalidation: live → `ELCC_STATE_LIVE`; expired → kernel revoke → `ELCC_STATE_REVOKED` (idempotent; `REVOKE_ALREADY` collapses in). |

### `src/elevate_client_journal.pdx` — module `ElevateClientJournal`

Audit-first REQ/APR journaling through the kernel user-events journal.

| Signature | Purpose |
| --- | --- |
| `elevate_client_journal_reset() -> () !{mem} @{}` | Scrub the 8-slot journal stats. Boot/teardown only. |
| `elevate_client_journal_note(which) -> () !{mem} @{}` / `elevate_client_journal_stat(which) -> u64 !{mem} @{}` | Bounded stats increment / read; out of range is a no-op resp. `0`. |
| `elevate_client_journal_req(actor_fp_lo, caps, dur_ns) -> u64 !{mem} @{}` | Append a REQ record via `uej_append`; returns the seq (< 256) or `ELVJ_ERR_BAD_ACTOR` / `_REQ_JOURNAL_FAIL`. |
| `elevate_client_journal_apr(actor_fp_lo, granted_caps, expire_ns) -> u64 !{mem} @{}` | Append an APR record; returns the seq or `ELVJ_ERR_BAD_ACTOR` / `_APR_JOURNAL_FAIL`. |
| `elevate_client_request_ex_j(caps, dur, req_buf, reply_ep_id, reply_buf, timeout_ns) -> u64 !{mem} @{boot}` | Audit-wrapped flow: journal REQ (failure aborts *before* the wire hop) → `request_ex` → journal APR. An APR-journal failure after a grant surfaces so the caller can revoke. |
| `elevate_client_journal_op(actor_fp_lo, row_id, needed_caps) -> u64 !{mem} @{}` | ENH-004 (#16): append a per-op record (`ELVJ_EVT_OP = 3`) for one authorized use of an acquired handle; returns the seq or `ELVJ_ERR_BAD_ACTOR` / `_OP_JOURNAL_FAIL`. This is the grant-amplification signal: N of these against one REQ/APR pair. |

### `src/elevate_client_retry.pdx` — module `ElevateClientRetry`

Bounded retry-with-backoff over the audit-wrapped flow.

| Signature | Purpose |
| --- | --- |
| `elevate_client_retry_reset() -> () !{mem} @{}` | Scrub retry stats and clear the configured attempt cap. |
| `elevate_client_retry_note(which) -> () !{mem} @{}` / `elevate_client_retry_stat(which) -> u64 !{mem} @{}` | Bounded stats increment / read; out of range is a no-op resp. `0`. |
| `elevate_client_retry_set_max_attempts(n) -> u64 !{mem} @{}` | Set attempts, `n ∈ [1, 8]`; otherwise `ELVR_ERR_BAD_ARG`. |
| `elevate_client_retry_get_max_attempts() -> u64 !{mem} @{}` | Read attempts; an unset slot resolves to the default `3`. |
| `elevate_client_retry_backoff_ns_for_attempt(attempt) -> u64 !{mem} @{}` | Delay after 1-indexed `attempt` failed: 100 ms, 400 ms, then 1.6 s saturated. Out of range returns `0`. |
| `elevate_client_retry_delay(dur_ns) -> () !{mem} @{boot}` | Bounded busy-poll on `hpet_now_ns`; collapses to a no-op when the clock is unarmed. |
| `elevate_client_request_ex_r(caps, dur, req_buf, reply_ep_id, reply_buf, timeout_ns) -> u64 !{mem} @{boot}` | Retry-wrapped full flow. Retriable set is exactly `{ELVC_ERR_TIMEOUT, ELVC_ERR_SEND_FAIL, ELVC_ERR_LOOKUP_FAIL}`; everything else passes through. Exhaustion returns `ELVR_ERR_EXHAUSTED`. |

## Schemas exposed

**REQ payload — 32 bytes** (`ElevateRequest`, mirror of the kernel
`ElevateChannel` wire schema):

| Offset | Field | Type | Notes |
| --- | --- | --- | --- |
| `+0` | `op_word` | `u64` | `[7:0] = ELV_OP_REQ (1)`, `[23:8] = caps`, `[63:24] = 0` |
| `+8` | `caps` | `u64` | Authoritative bitmask; valid bits `0..7` (`ELV_CAP_MASK_VALID = 0x00FF`) |
| `+16` | `duration_ns` | `u64` | Must lie in `[1_000_000_000, 3_600_000_000_000]` |
| `+24` | `reserved` | `u64` | Always `0` |

Caps appear in both the op word and body word 0 by design: the op-word
copy lets a fast-path approver classify without decoding the body; the
body copy is the canonical source for the audit journal.

**APR payload — 32 bytes**, same shape, read out of `reply_buf`:
`[+0]` op word whose low byte must equal `ELV_OP_APR (2)`, `[+8]`
`granted_caps` (must satisfy `granted ⊆ requested`), `[+16]` `expire_ns`
(refused when `0`), `[+24]` reserved. Wire ops are `ELV_OP_REQ = 1`,
`ELV_OP_APR = 2`, `ELV_OP_EXP = 3`.

**IPC frame header — 8 bytes**, packed inline by
`elevate_client_pack_hdr`: `[7:0]` op, `[15:8]` ver (`ELVS_FRAME_VER = 1`),
`[31:16]` `reply_endpoint_id` (`0` = reply-on-source, else `1..127`),
`[63:32]` `payload_len` (32 for REQ and APR alike).

**Policy row — 48 bytes, six words** (`ElevateClientPolicy`):

| Offset | Field | Notes |
| --- | --- | --- |
| `+0` | `magic(32) \| version(16) \| flags(16)` | `ELCP_MAGIC = 0x50585044` (`'PDXP'`), `ELCP_VERSION = 1`, `ELCP_FLAG_ENABLED = 0x0001` |
| `+8` | `target_user_fp_lo` | `0` = any user |
| `+16` | `caps_mask` | Matches when `REQ.caps ⊆ this` |
| `+24` | `max_duration_ns` | Requested duration must not exceed it |
| `+32` | `rate_per_hour(32) \| window_hits(32)` | Rate in the low 32 bits; `0` = unlimited |
| `+40` | `reserved` | `0` |

The magic matches the kernel `ElevatePolicy` magic, so a boot-time loader
reading one on-disk policy file can feed the same row bytes to both
`ep_install` and `elevate_client_policy_install`.

**Audit record** — via `uej_append` with `UEJ_KIND_ELEVATE = 5`,
`subject_fp_lo = 0`, and three body words: `body0` is the discriminator
(`ELVJ_EVT_REQ = 1` / `ELVJ_EVT_APR = 2`, chosen to overlap the wire op
codes), `body1` is `caps` (REQ) or `granted_caps` (APR), `body2` is
`duration_ns` (REQ) or `expire_ns` (APR).

**Result bands** — each layer owns a distinct range, so one return value
says which layer refused: `ELV_ERR_*` `0xFFFFE5E0..EF` (request could not
be built), `ELVC_*` `0xFFFFEA00..0F` (transport), `ELCP_ERR_*`
`0xFFFFEA10..1F` (client policy), `ELVR_ERR_*` `0xFFFFEA20..2F` (retry),
`ELCC_*` `0xFFFFEA30..3F` (cap lifecycle), `ELVJ_ERR_*` `0xFFFFEA40..4F`
(journal). `0` is success in every band.

## Callers

- **[rm](https://github.com/paideia-os/rm)** — *verified.*
  `src/elevate.pdx` (`RmElevate`, rm.M3-004) calls
  `elevate_client_request` for `/system/` and cross-subtree targets, and
  treats `ELVC_OK` or `ELVC_STUB` as proceed, anything else as blocked.
- **[pkg](https://github.com/paideia-os/pkg)** — *verified.*
  `src/pkg_elevate.pdx` (`pkg_elevate_request_pdxfs_write_pkgs`,
  pkg.M3-004) calls `elevate_client_request` with `caps = 0x01` and a
  60 s duration, stashing the raw `ELVC_*` return for diagnostics.
- **[shell](https://github.com/paideia-os/shell)** — *likely caller
  (self-described elevate-integrated).* At HEAD, `src/broker_bind.pdx`
  names `svc.elevate-broker` in its service list, and `src/shell.pdx`,
  `src/exec.pdx`, `src/session.pdx` mirror this library's constants and
  idioms (`SH_KIND_ELEVATE_CHANNEL = 0x191`, the `ELVC_STUB` return
  convention). No direct call into a `libpdx-elevate` entry point was
  found in the shell sources.

## Version

**v1.0.0** — R49 shared library, first signed release (2026-08-22). M1–M5
closed. `CHANGELOG.md` carries the per-milestone 1.0.0 entry and
`STATUS.md` the milestone rollup. `caps.decl` declares the floor
capability set a *consuming process* must hold per entry point.
`deps.list` records the two forward references — `libpdx-cap >= 0.2.0`
(the `elevate_client_cap_narrow_stub` swap) and `libpdx-audit >= 0.3.0`
(the `journal_req` / `_apr` swap) — both wired to kernel primitives
today, both source-compatible when they land. `manifest.pdxsig` is the
dual-signed release manifest, its signature blocks carrying the
`<MLDSA65-SIG:STUB-PENDING-V0.33-CRYPTO-KDF>` sentinel until the
toolchain crypto tag is reachable. `doc/libpdx-elevate.pdxdoc` is the I7
reference, with a §POSIX DIFFERENCES five-axis contrast against `sudo` /
`setuid`. Milestone breakdown and cross-repo dependencies: see
`design/tooling/r49-r50-plan.md` §5.14 in
[paideia-os](https://github.com/paideia-os/paideia-os).

## Examples

**Minimal request (M1 surface, as `rm` uses it).** Two 32-byte scratch
buffers, one call; `ELVC_OK` or `ELVC_STUB` both mean proceed.

```
mov rdi, 0;                        // caps
mov rsi, 0;                        // dur
lea rdx, [rip + _elevate_req_buf];
lea rcx, [rip + _elevate_reply_buf];
call elevate_client_request;
cmp rax, 0;                        // ELVC_OK -> proceed
je  proceed;
mov r10, 0xFFFFEA00;               // ELVC_STUB -> proceed
cmp rax, r10;
je  proceed;                       // anything else: blocked
```

**Full flow with a seeded fast path (M2).** Seed the client policy with
the same rule the founder signed into the kernel table, bind the caller
identity, then let `timeout_ns = 0` pick the deadline by classification.

```
mov rdi, 0;                        // target_fp_lo = 0 (any user)
mov rsi, 0x01;                     // caps mask
mov rdx, 60000000000;              // max_duration_ns = 60 s
mov rcx, 5;                        // rate_per_hour
call elevate_client_policy_install;

mov rdi, r14;                      // caller fingerprint (low 64 bits)
call elevate_client_set_target_fp_lo;

mov rdi, 0x01;                     // caps
mov rsi, 60000000000;              // dur = 60 s
lea rdx, [rip + _req_buf];         // 32-byte scratch
mov rcx, 7;                        // reply_ep_id, must be in [1..127]
lea r8,  [rip + _reply_buf];       // 32-byte scratch
mov r9,  0;                        // 0 -> fast (hit) or human (miss)
call elevate_client_request_ex;
cmp rax, 0;                        // ELVC_OK: reply_buf holds the APR
jne handle_error;
```

**Audited, retried, lifetime-bounded (M3).** `_ex_r` journals every
attempt and retries only the three transport symptoms; the caller then
binds the shadow deadline and self-invalidates at expiry.

```
mov rdi, 3;                        // attempts in [1..8]
call elevate_client_retry_set_max_attempts;
mov rdi, 0x01;
mov rsi, 60000000000;
lea rdx, [rip + _req_buf];
mov rcx, 7;
lea r8,  [rip + _reply_buf];
mov r9,  0;
call elevate_client_request_ex_r;  // ELVR_ERR_EXHAUSTED if the broker
cmp rax, 0;                        // never recovered
jne handle_error;

// Bind the granted lifetime, then later self-invalidate.
mov rdi, r13;                      // row_id from elevate_client_cap_mint
mov rsi, 60000000000;              // dur_ns
call elevate_client_cap_bind_expire;

mov rdi, r13;
call elevate_client_cap_check_and_revoke;
// ELCC_STATE_LIVE (0xFFFFEA34) | ELCC_STATE_REVOKED (0xFFFFEA36)
```

## License

MIT — see LICENSE.
