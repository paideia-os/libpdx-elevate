# libpdx-elevate — enhancement plan (post-1.0.0)

Authored 2026-08-25 during the org-wide 14-repo enhancement audit.
Scope: this repo only. Companion work in `rm`, `pkg`, `shell`, and the
paideia-os monorepo is noted but not filed from here.

---

## 1. Current state (verified against source at HEAD `e369a24`)

**Shape.** Seven `.pdx` modules, 3,395 source lines, two boot witnesses
(`tests/elevate_client_policy_test.pdx`, `tests/elevate_client_cap_test.pdx`),
`caps.decl`, `deps.list`, dual-signed `manifest.pdxsig`,
`doc/libpdx-elevate.pdxdoc`. M1–M5 all LANDED; issues #1–#10 all closed;
no open issues; tag `v1.0.0`.

**What "elevate" means here.** Not a privilege upgrade of the calling
process. The client assembles a 32-byte REQ frame, resolves
`svc.elevate-broker` via `svc_lookup`, publishes it, waits under a
deadline for a 32-byte APR frame, checks `granted ⊆ requested`, journals
both halves under `UEJ_KIND_ELEVATE = 5`, and mints a
`Cap<KIND_ELEVATE_CHANNEL = 0x191>` with a client-side shadow deadline.
The broker — a paideia-os kernel component — is the sole enforcement
point. The client policy table is a cached pre-classification that only
selects a recv deadline (500 ms fast / 30 s human); a hit never skips the
broker hop.

**Completeness of the client half: high.** Every layer named in
STATUS.md exists and is exercised. Two acknowledged stubs, both
documented and both forward-referenced in `deps.list`:
`elevate_client_cap_narrow_stub` (`src/elevate_client_cap.pdx:294`,
awaiting libpdx-cap.M2) and the journal bodies wired straight to kernel
`uej_append` instead of libpdx-audit. Neither is a defect.

**Completeness of end-to-end enforcement: zero, and not this repo's
fault.** The broker daemon body in paideia-os
(`src/kernel/core/ipc/elevate_broker.pdx`) is `ELVB_DISPATCH_STUB`
(STATUS.md:186–191), so `elevate_client_request_ex*` can only ever reach
`ELVC_ERR_TIMEOUT` today. That is an upstream gap.

**The one genuine library-side defect** is separate from that, and it is
the entry point the two real consumers actually call.

---

## 2. The fail-open entry point (finding that drives ENH-001/002/005)

`elevate_client_request` (`src/elevate_client.pdx:245–291`) is the M1
skeleton. Its success path is:

```
        // (4) M1 stop -- endpoint resolved; real dispatch is M2.
        mov rdi, 4;                      // ELVC_ST_STUBS
        call elevate_client_note;
        mov rax, 0xFFFFEA00;             // ELVC_STUB
```

It assembles the frame, resolves the broker, and returns. It never
sends, never receives, never validates a grant. README.md:60 keeps it as
"a source-compatibility surface" — but it is the *only* entry point
either real consumer calls, because it is the only one that fits in four
arguments and needs no reply endpoint:

- `rm` — `/tmp/pdx-readme-rm/src/elevate.pdx:234` calls it, then at
  `:240–247` treats **both** `ELVC_OK (0)` and `ELVC_STUB (0xFFFFEA00)`
  as `RE_PROCEED`. Fail-open.
- `pkg` — `/tmp/pdx-readme-pkg/src/pkg_elevate.pdx:192` calls it, then
  collapses every non-zero return to `PE_PARENT_UNAVAILABLE`, which
  `src/install.pdx:482` routes to `pi_err_parent`. Fail-closed.

Two consumers, the same call, the same return value, **opposite security
dispositions** — and the library sanctions the fail-open reading in its
own README example (README.md:230–244: "`ELVC_OK` or `ELVC_STUB` both
mean proceed"). That is a library-side API defect, not a consumer bug:
a sentinel that lives in the error band but is documented as a
proceed value is a coin-flip for every future consumer.

---

## 3. Does the API invite `rm`-style bypass? — recommendation: **yes, harden it**

The `rm -r` regression (a recursive path enforcing no elevate check while
the single-file path does) is `rm`'s bug to fix in `rm`'s repo. The
question for us is narrower: does *this* API make that class of bug easy?
It does, for one structural reason.

**The gate is an advisory boolean, not a credential.** Every entry point
in the library returns a status code. `elevate_client_request_ex` returns
`0` and leaves the APR in the caller's `reply_buf`
(`src/elevate_client_send.pdx:674–678`); minting the actual
`Cap<KIND_ELEVATE_CHANNEL>` is a *separate, optional* call the consumer
may simply never make (`elevate_client_cap_mint`). Nothing the privileged
operation subsequently performs consumes anything the elevate hop
produced. So "did I ask?" is a fact recorded nowhere the destructive code
path is forced to look, and skipping the question on one branch is
invisible at every level — no missing argument, no unbound value, no
kernel refusal.

Contrast the design the rest of PaideiaOS already uses: a capability is
the thing you *present*, so forgetting to obtain it fails at the point of
use. The elevate flow ends one step short of that. That is the gap.

**Recommendation.** Add a credential-shaped surface alongside (not
replacing) the status-shaped one:

1. A composed acquire that returns a **handle** — `row_id` — instead of a
   status (ENH-001). Consumers thread the handle into the privileged
   operation, so omitting the gate becomes a missing argument rather than
   a skipped statement.
2. A cheap per-op **re-assert** against that handle (ENH-002) — no broker
   hop, just "not expired, caps still cover this". A recursive walk calls
   it once per item; the cost is a clock read.
3. **Scope + budget** binding on the handle (ENH-003). This is the direct
   structural answer to the recursive case: one grant covering N objects
   becomes explicit and bounded rather than accidental and unbounded.

**Recommendation against**, for honesty: do *not* attempt a
`--recursive`-aware helper that knows about paths, trees, or argv. This
library has no filesystem vocabulary and should not grow one; scope is
expressed as an opaque fingerprint the consumer supplies. And do not
pretend a library can *force* a consumer to call anything — it cannot.
ENH-001/002/003 make the omission *structurally visible and audit-
detectable*, which is the achievable goal; ENH-004 closes the loop by
making grant amplification (1 REQ record, N mutating ops) legible to an
auditor after the fact, which is the D3 discipline this repo already
follows everywhere else.

---

## 4. `shell` non-linkage assessment: **shell's own unreadiness, not a library gap**

Verified read-only against `/tmp/pdx-readme-shell/src/`. Shell contains
**no call into any libpdx-elevate entry point**. What exists:

- `src/broker_bind.pdx:100` names `svc.elevate-broker` in a service list.
- `src/shell.pdx:168,199,227`, `src/session.pdx:165`,
  `src/line_reader.pdx:24,120`, `src/exec.pdx:19,84` cite this library in
  *comments and justifications* — copying its stats-table shape, its
  `sub rsp, 8` SysV idiom, and its `*_STUB`-is-the-happy-path convention
  (shell minted its own `LR_STUB` / `EX_STUB` / `BB_STUB` sentinels from
  it). Mirroring, not linking. README.md:203–208 already describes this
  accurately.

The blocker is shell-side and explicit in shell's own source:
`src/exec.pdx:435` says `EX_STUB` stands because "there is no userspace
`sys_execve` wrapper in the tree yet", and `src/line_reader.pdx:125` says
`LR_STUB` stands because `KIND_TTY` has not landed. Shell cannot elevate
on behalf of a command it cannot yet spawn. Nothing is missing from
libpdx-elevate that shell needs today; no ABI problem exists.

One forward-looking caveat, filed as ENH-006 and explicitly **not** a
shell blocker: this library assumes one elevate flow per process. The
actor fingerprint (`src/elevate_client_send.pdx:125`,
`_elevate_client_target_fp_lo`) and both timeout defaults (`:114–115`)
are process-global mutable singletons — a deliberate trade to fit
`elevate_client_request_ex` in the six-register SysV window (`:578–584`).
A shell brokering for several concurrent jobs would have them race. Worth
fixing before shell links, not before shell exists.

Also worth noting for whoever wires shell later: the ENH-005 disposition
rule matters most to shell, because shell has already copied the
`*_STUB`-means-proceed convention into three of its own sentinels.

---

## 5. Issue plan

Filed under milestone 6, *Enhancement v1.x — libpdx-elevate*.

| ID | # | Title | Effort | Deps |
| --- | --- | --- | --- | --- |
| ENH-001 | #11 | `elevate_client_acquire` — composed flow returning a grant handle | M | none |
| ENH-002 | #13 | `elevate_client_require` — per-op re-assert with no broker hop | S | #11 |
| ENH-003 | #15 | Grant scope + op-budget binding on the handle | M | #11, #13 |
| ENH-004 | #16 | Per-op audit record so grant amplification is detectable | S | #13 |
| ENH-005 | #12 | Retire `ELVC_STUB`-means-proceed; rename the non-dispatching M1 entry | S | none |
| ENH-006 | #17 | Explicit-context variants; retire the process-global singletons | M | none |
| ENH-007 | #14 | README/STATUS enforcement-status accuracy pass | XS | #12 |

ENH-001/002/003/004 (#11, #13, #15, #16) are the hardening core and land together as a
coherent v1.1.0 surface. ENH-005 is the only source-breaking change and
must be coordinated with `rm` and `pkg` (their repos, their issues).
ENH-006 is speculative-but-cheap insurance ahead of shell. ENH-007 is
documentation truth-in-advertising.

## 6. Notes for other repos (filed by their own owners, not from here)

- **rm** — fix the `-r` bypass; separately, stop treating `ELVC_STUB` as
  proceed (`src/elevate.pdx:244–247`). Blocked-by ENH-005 for the rename.
- **pkg** — already fail-closed; only the ENH-005 rename touches it.
- **shell** — no action until its exec path lands.
- **paideia-os** — the broker daemon body
  (`src/kernel/core/ipc/elevate_broker.pdx`, `ELVB_DISPATCH_STUB`) is the
  single thing standing between this library and real enforcement. Also
  still open from STATUS.md:179–197: kernel `elevate_channel_row_set_expire`
  (would retire the shadow deadline map), R51 scheduler-wait syscall
  (would retire the retry busy-poll), and `tick_ns` in `uej_append`.

## 7. v1.0.0 verdict

Defensible **as a client-side protocol helper**, which is what the tag
claims: M1–M5 closed, every layer implemented and witnessed, forward
references documented rather than hidden, manifest hash-stable. Not
defensible as "the elevate gate" — end-to-end denial has never once
occurred, because the broker daemon upstream is a stub and the entry
point both consumers call never dispatches. The tag stands; ENH-005 and
ENH-007 exist so the README stops implying otherwise.
