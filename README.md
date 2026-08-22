# libpdx-elevate

paideia-os shared library: client-side elevate protocol helper

## Status

**v1.0.0 — R49 shared library, first signed release (2026-08-22).**  M1-M5
closed; see `STATUS.md` for the milestone rollup and `CHANGELOG.md` for
the 1.0.0 entry.  See `design/tooling/r49-r50-plan.md` §5.14 in the
[paideia-os](https://github.com/paideia-os/paideia-os) repo for the
milestone breakdown, KIND allocations, cross-repo dependencies, and
per-milestone issue set.

## What this is

The client-side wrapper around the R48.M7 kernel elevate protocol
(`svc.elevate-broker` + `KIND_ELEVATE_CHANNEL = 0x191`).  Consumers
request a bounded-lifetime capability from the broker; the library
handles policy consult, wire assembly, bounded recv, grant-subset
validation, audit journaling (`uej_append` UEJ_KIND_ELEVATE = 5),
cap-lifetime enforcement, and retry-with-backoff on transient
broker unavailability.

Not `sudo`.  There is no `sudo` on PaideiaOS.  See
`doc/libpdx-elevate.pdxdoc` §POSIX DIFFERENCES for the five-axis
contrast.

## Release artifacts (M5)

- `manifest.pdxsig` — dual-signed release manifest (D4).  Sig blocks
  carry the `<MLDSA65-SIG:STUB-PENDING-V0.33-CRYPTO-KDF>` sentinel;
  populate with `paideia-as release --sign` when v0.33-crypto-kdf is
  reachable.
- `caps.decl` — capability manifest, per-entry-point.
- `deps.list` — cross-repo deps (libpdx-cap, libpdx-audit PENDING;
  kernel primitives LANDED).
- `doc/libpdx-elevate.pdxdoc` — I7 documentation, rendered by `doc`.
- `CHANGELOG.md` — 1.0.0 release notes.
- `.plans/mirror-push.md` — runbook for pushing to `pkgs.paideia-os`.

## License

MIT — see LICENSE.
