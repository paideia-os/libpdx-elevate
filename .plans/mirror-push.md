# libpdx-elevate — mirror-push runbook

**Scope:** procedure for pushing a signed `libpdx-elevate` release to
`pkgs.paideia-os/main/libpdx-elevate/$VERSION/` once the mirror repo
exists and `paideia-as v0.33-crypto-kdf` is reachable through the
toolchain. Originally filed under M5-001 (2026-08-22) as a v1.0.0-only
runbook; genericized on `$VERSION` (issue #39, 2026-09-05) once v1.1.1
and v1.1.2 landed on `main` without either cutting a signed mirror
push — every command below used to hardcode `1.0.0` and would have
silently re-pushed the wrong version the day this actually fires.

**As of 2026-09-05: this runbook has never been executed for real.**
`manifest.pdxsig` still carries `version = 1.0.0` and both
`[signature.author]` / `[signature.paideia-root]` still carry the
`<MLDSA65-SIG:STUB-PENDING-V0.33-CRYPTO-KDF>` sentinel — v0.33-crypto-kdf
has not landed in paideia-as yet. Meanwhile `main` has moved to
v1.1.2 (LE.M1-polish + LE.M2-hardening) via lightweight annotated git
tags that are landing markers, not the dual-ML-DSA-65-signed release
this runbook produces (see "Version history" below). **The first real
run of this runbook signs and pushes whatever version is HEAD on the
day v0.33-crypto-kdf becomes reachable — bind `$VERSION` to that, not
to 1.0.0.**

---

## Before running: version-bump checklist

Every step below is written against a `$VERSION` shell variable. Set
it once at the top of your session:

    VERSION=1.1.2          # or whatever `main` HEAD's version is

Before running Step 1, confirm every one of these reflects the
version you just set — each has drifted at least once already:

1. **`manifest.pdxsig`** — `[manifest] version` line, `released` date,
   `source-commit`, `source-tree-hash-sha256`, and the four
   `[cap-manifest]` / `[deps]` / `[changelog]` / `[docs]` hash fields
   are all still stamped for a stale version until Steps 2–3 refresh
   them. Do not assume a prior run left these current.
2. **`CHANGELOG.md`** — has a `## $VERSION` section (not
   `## Unreleased — v$VERSION`) with a real date, not "no signed tag
   yet."
3. **`STATUS.md`** — top-of-file "Version:" line names `$VERSION` as
   tagged, not "landed, tag/signing pending."
4. **`deps.list`** — re-check the PENDING/LANDED status of
   `libpdx-cap` and `libpdx-audit`. Both were PENDING at v1.0.0 and
   remained PENDING through v1.1.2; if either flipped to LANDED in the
   version you're pushing, the swap-site comments in
   `src/elevate_client_cap.pdx` / `elevate_client_journal.pdx` should
   already reflect it — this runbook does not re-verify source, only
   the manifest hash of `deps.list` as it stands.
5. **Sibling R49 libraries** (`libpdx-cap`, `libpdx-semantic-pipe`,
   `libpdx-argv`, `libpdx-audit`) — the dependency-ordering rule below
   is about what's LANDED at `pkgs.paideia-os/main` *at push time*,
   not what was LANDED when this doc was last edited.
6. **Git tag name** — `v$VERSION` must not already exist as an
   *annotated, unsigned* landing-marker tag (v1.1.1 never got one;
   v1.1.2 did — see below). Step 8's `git tag -a -s` either replaces
   that marker or, if git refuses to overwrite, the marker tag must be
   deleted and recreated as the real signed tag. Never silently skip
   the `-s` because a same-named unsigned tag already exists.

If any of 1–4 is stale, fix the source docs first (a separate commit)
— this runbook signs whatever `git rev-parse HEAD` and the working
tree currently say; it does not update prose.

---

**Dependency ordering.** libpdx-elevate ships to the mirror **before**
pkg's own M5 (§5.1) so that pkg can consume the elevate helper as one
of its own dual-signed deps at pkg's M5 install-flow landing. When
pkg.M4 closes and pkg.M5-001 opens, this runbook fires against the
`pkgs.paideia-os` staging area. The five R49 libraries push in the
order libpdx-cap → libpdx-semantic-pipe → libpdx-argv → libpdx-audit
→ libpdx-elevate; libpdx-elevate is last of the five because its two
PENDING deps (libpdx-cap, libpdx-audit) must ship first so that the
`deps.list` PENDING entries can flip to LANDED.

---

## Prerequisites

Everything in this list must be true before the runbook fires:

1. `paideia-as v0.33-crypto-kdf` (Argon2id + ChaCha20-Poly1305 +
   ML-DSA-65 verify) reachable through the toolchain on the release
   machine. Verified with `paideia-as --version | grep crypto-kdf`.
2. Author key `author_pk` for the paideia-os org present in the
   local keyring, unlocked for the session (Argon2id KDF over the
   author's passphrase per design/user/model.md §2.1).
3. `pkgs.paideia-os/staging/` writable by the release machine's
   push cap (KIND_NETWORK(post, pkgs.paideia-os/staging)).
4. `libpdx-cap`, `libpdx-semantic-pipe`, `libpdx-argv`, `libpdx-audit`
   — whatever their own current LANDED versions are — all present at
   `pkgs.paideia-os/main/`. Verified with:

       pkg list --repo=pkgs.paideia-os/main libpdx-cap libpdx-semantic-pipe \
                                            libpdx-argv libpdx-audit

5. The two PENDING entries in `deps.list` (libpdx-cap `narrow_stub`
   swap, libpdx-audit `journal_req/_apr` swap) may still be PENDING at
   `$VERSION` — the swap sites are documented and source-compatible.
   These stayed PENDING across v1.0.0 → v1.1.1 → v1.1.2; do not assume
   a specific target minor clears them (the v1.0.0 runbook's original
   "swap lands in v1.1.0" line was itself wrong — v1.1.0 was retired
   without a tag and the swap still hadn't landed by v1.1.2). Check
   `deps.list` directly rather than trusting any version number cited
   in prose, including in this file.

---

## Steps

### 1. Freeze the source tree

    VERSION=1.1.2                        # set once, reused below
    git checkout main
    git pull
    git tag -l v$VERSION                 # if it exists, confirm it is
                                          # a landing marker (unsigned,
                                          # no [signature] block change)
                                          # and not already the real
                                          # signed release before reusing it
    git log --oneline -1                 # note the HEAD commit

The `source-commit` field you are about to write into
`manifest.pdxsig` must match `git rev-parse HEAD` at this point. If
HEAD has moved, re-run the relevant witnesses in QEMU and re-hash the
source tree before proceeding (a new source-tree hash means the
manifest body changes and requires a fresh `paideia-as release --sign`
pass).

*Worked example (v1.0.0, 2026-08-22):* `source-commit` was pinned to
`796fe888cd384dfefc3d088652c1b02d3538f186`. That value is specific to
the 1.0.0 tree and must NOT be reused for any later version — it is
cited here only so a future reader can tell a stale copy-paste from a
real `$VERSION`-specific hash.

### 2. Verify manifest body is hash-stable

    ./tools/canonicalise-manifest.sh manifest.pdxsig \
        | sha256sum -                     # must match the digest of the
                                          # message paideia-as signs

(The `canonicalise-manifest.sh` script is expected to land in the
paideia-as v0.33 release under `tools/`; if absent at push time,
inline: strip lines beginning with `#`, LF-only line endings, trailing
whitespace off, sort keys within each `[section]`, collapse repeated
blank lines.)

### 3. Compute the four hash fields

Replace the four `<computed-at-release-sign-time>` placeholders in
`manifest.pdxsig` with the actual digests:

    sha256sum caps.decl        # -> [cap-manifest] caps.decl.canonical
    sha256sum deps.list        # -> [deps] deps.list.canonical
    sha256sum CHANGELOG.md     # -> [changelog] CHANGELOG.md
    sha256sum doc/libpdx-elevate.pdxdoc   # -> [docs] doc/libpdx-elevate.pdxdoc

Also update `[manifest] version = $VERSION`, `released = <today>`, and
`source-commit` / `source-tree-hash-sha256` from Step 1. Commit the
resulting hash-populated manifest.pdxsig on a release branch
(`release/v$VERSION`) — this is the manifest body both signatures will
cover.

### 4. Author-side sign

    paideia-as release --sign \
        --manifest manifest.pdxsig \
        --key author_pk \
        --output manifest.pdxsig

`paideia-as release --sign` scans for the sentinel
`<MLDSA65-SIG:STUB-PENDING-V0.33-CRYPTO-KDF>` in
`[signature.author]` and replaces it in place with the base64-
encoded 3309-byte ML-DSA-65 signature over the canonical body.
`[signature.paideia-root]` is left untouched at this step.

### 5. Build the package archive

    paideia-as release --pack \
        --manifest manifest.pdxsig \
        --output libpdx-elevate-$VERSION.pkg.tar

The archive contains: `src/`, `tests/`, `caps.decl`, `deps.list`,
`CHANGELOG.md`, `doc/`, `manifest.pdxsig`, `LICENSE`, `README.md`,
`STATUS.md`.

### 6. Push to staging

    pkg push --repo=pkgs.paideia-os/staging \
             --pkg libpdx-elevate-$VERSION.pkg.tar

At this point the archive is visible under
`pkgs.paideia-os/staging/libpdx-elevate/$VERSION/`. The Paideia
signing bot polls staging; when it picks up this release it:

  a. Verifies the author sig against the `author_pk` fingerprint the
     paideia-os org has registered.
  b. Verifies the source-tree hash by unpacking the archive and
     recomputing.
  c. Signs the same canonical body under `paideia_root_pk` (the R32
     root) and replaces the sentinel in `[signature.paideia-root]`.
  d. Copies the resigned archive to
     `pkgs.paideia-os/main/libpdx-elevate/$VERSION/`.
  e. Updates `pkgs.paideia-os/main/index.pdxsig` to include the new
     `{name=libpdx-elevate, version=$VERSION, hash=<sha256-of-tarball>}`
     entry, re-signed under `paideia_root_pk`.

### 7. Verify from a clean machine

On a machine that does NOT hold `author_pk` or `paideia_root_pk`:

    pkg install libpdx-elevate
    #   resolved:  libpdx-elevate-$VERSION (pkgs.paideia-os/main)
    #   verified:  author=paideia-os-team (ML-DSA-65)
    #   verified:  paideia-manifest (ML-DSA-65 root)
    #   cap-audit: [ (see caps.decl — no runtime caps for a library) ]
    #   installed: /pkgs/libpdx-elevate-$VERSION

### 8. Tag the release

    git tag -a -s v$VERSION -m "libpdx-elevate $VERSION — <one-line summary from CHANGELOG.md>"
    git push origin v$VERSION

The `-s` produces a git signature under the author's OpenPGP key
(separate from ML-DSA-65 — this is the git-side attestation only).
The ML-DSA-65 signature that gates pkg install lives in
`manifest.pdxsig`, not in the git tag.

If `v$VERSION` already exists as an unsigned landing-marker tag (see
"Version history" below), delete the marker (`git push origin
:refs/tags/v$VERSION && git tag -d v$VERSION`) before re-creating it
signed — do not leave two different objects claiming the same tag
name on the remote.

### 9. Update STATUS.md

Flip the current milestone's rollup row to LANDED, add a "$VERSION
released" line, and correct the top-of-file "Version:" line to say
"$VERSION (tagged)" instead of "landed on main, tag/signing pending."
Commit directly to `main` (`chore/release-status: libpdx-elevate
$VERSION landed`), reference this runbook.

---

## Version history

Context for whoever runs this next, so "which version do I bind
`$VERSION` to" has a paper trail instead of a guess:

- **v1.0.0** (tag `v1.0.0`, 2026-08-22) — first version to reach
  Step 3 of this runbook in spirit (the manifest's hash fields and
  sentinel signatures were drafted), but the mirror repo did not
  exist yet and `paideia-as v0.33-crypto-kdf` still isn't reachable —
  Steps 4 onward have never actually executed. `manifest.pdxsig` on
  `main` still reads `version = 1.0.0` today.
- **v1.1.0** — never tagged. Retired as a label once the LE.M1-M3
  multilevel-chain wave landed on top of it; superseded in-place by
  v1.1.1 per `CHANGELOG.md` / `STATUS.md`. If you find a stale
  reference to "the v1.1.0 tag" anywhere else in this repo, it is
  describing something that was never cut — do not go looking for it.
- **v1.1.1** — landed on `main` (commit `493df42`, 2026-09-02: LE.M3
  multilevel-chain foundations + `derive()` broker-slot fix). Also
  never tagged — superseded before a release pass ran, by v1.1.2 the
  same day.
- **v1.1.2** (tag `v1.1.2`, commit `9534007`, 2026-09-02) — LE.M1-
  polish (#40–#47) + LE.M2-hardening (#48–#52), 13 fixes. The tag is
  an **annotated, GPG-unsigned landing marker** (`git tag -v v1.1.2`
  reports "no signature found") — it records that this commit is a
  coherent release point on `main`, not that this runbook's Steps 4–9
  have run against it. Treat it the same way as the never-cut v1.1.0
  / v1.1.1 labels: a version number that exists in git history, not
  evidence of a signed mirror push.

**Pattern to recognize:** every version since v1.0.0 has landed on
`main`, and some have gotten an annotated git tag, without a signed
release ever being cut — because that release step is gated entirely
on `paideia-as v0.33-crypto-kdf`, which has not shipped. When it does,
`$VERSION` in this runbook should be bound to whatever is HEAD on
`main` at that moment (append a new bullet above once it's decided),
**not** rewound to v1.0.0 or to whichever version this doc happens to
have been last edited against.

---

## Rollback

If a bad archive reaches `pkgs.paideia-os/main/` and must be pulled:

1. Remove the archive path and rebuild `index.pdxsig` under
   `paideia_root_pk` without the entry.
2. Push a `libpdx-elevate` patch release (next patch number after
   `$VERSION`) with a `CHANGELOG.md` entry naming the withdrawn
   version and the reason.
3. Never re-use a version number for a different tarball — semver is
   immutable at the mirror.

Local machines that already installed `$VERSION` continue to work
(the manifest.pdxsig they hold verifies against the paideia_root_pk
they already trusted); the pull just removes the *availability* from
the mirror. A `pkg upgrade` on those machines will pick up the patch
release.

---

## Escalations

- Author key compromise: rotate `author_pk`, re-sign every extant
  release under the new key, push all archives to staging in the
  order libpdx-cap → …→ libpdx-elevate. Coordinate with the
  Paideia root re-sign gate to invalidate old-author-key manifests
  at `pkgs.paideia-os/main/`.
- Paideia root compromise: this is a §8 of `design/user/model.md`
  event; halt all mirror push activity and follow the R32
  root-rotation playbook (out of scope for this file).
