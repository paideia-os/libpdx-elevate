# libpdx-elevate — mirror-push runbook (M5-001)

**Scope:** procedure for pushing `libpdx-elevate 1.0.0` to
`pkgs.paideia-os/main/libpdx-elevate/1.0.0/` once the mirror repo
exists.  Filed in `.plans/` (not repo root) because the mirror does not
exist at 2026-08-22; this file is the load-bearing runbook for the day
it does.

**Dependency ordering.**  libpdx-elevate ships to the mirror **before**
pkg's own M5 (§5.1) so that pkg can consume the elevate helper as one
of its own dual-signed deps at pkg's M5 install-flow landing.  When
pkg.M4 closes and pkg.M5-001 opens, this runbook fires against the
`pkgs.paideia-os` staging area.  The five R49 libraries push in the
order libpdx-cap → libpdx-semantic-pipe → libpdx-argv → libpdx-audit
→ libpdx-elevate; libpdx-elevate is last of the five because its two
PENDING deps (libpdx-cap, libpdx-audit) must ship first so that the
`deps.list` PENDING entries can flip to LANDED.

---

## Prerequisites

Everything in this list must be true before the runbook fires:

1. `paideia-as v0.33-crypto-kdf` (Argon2id + ChaCha20-Poly1305 +
   ML-DSA-65 verify) reachable through the toolchain on the release
   machine.  Verified with `paideia-as --version | grep crypto-kdf`.
2. Author key `author_pk` for the paideia-os org present in the
   local keyring, unlocked for the session (Argon2id KDF over the
   author's passphrase per design/user/model.md §2.1).
3. `pkgs.paideia-os/staging/` writable by the release machine's
   push cap (KIND_NETWORK(post, pkgs.paideia-os/staging)).
4. `libpdx-cap 1.0.0`, `libpdx-semantic-pipe 1.0.0`, `libpdx-argv
   1.0.0`, `libpdx-audit 1.0.0` all present at
   `pkgs.paideia-os/main/`.  Verified with:

       pkg list --repo=pkgs.paideia-os/main libpdx-cap libpdx-semantic-pipe \
                                            libpdx-argv libpdx-audit

5. The two PENDING entries in `deps.list` (libpdx-cap `narrow_stub`
   swap, libpdx-audit `journal_req/_apr` swap) may still be PENDING
   at v1.0.0 — the swap sites are documented and source-compatible,
   and the swap lands in v1.1.0 (not this release).

---

## Steps

### 1. Freeze the source tree

    git checkout main
    git pull
    git tag -l v1.0.0                    # must NOT already exist
    git log --oneline -1                 # note the HEAD commit

The `source-commit` field in `manifest.pdxsig` at
`796fe888cd384dfefc3d088652c1b02d3538f186` must match `git rev-parse
HEAD` at this point.  If HEAD has moved, re-run the M4 witnesses in
QEMU and re-hash the source tree before proceeding (a new source-tree
hash means the manifest body changes and requires a fresh
`paideia-as release --sign` pass).

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

Commit the resulting hash-populated manifest.pdxsig on a release
branch (`release/v1.0.0`) — this is the manifest body both signatures
will cover.

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
        --output libpdx-elevate-1.0.0.pkg.tar

The archive contains: `src/`, `tests/`, `caps.decl`, `deps.list`,
`CHANGELOG.md`, `doc/`, `manifest.pdxsig`, `LICENSE`, `README.md`,
`STATUS.md`.

### 6. Push to staging

    pkg push --repo=pkgs.paideia-os/staging \
             --pkg libpdx-elevate-1.0.0.pkg.tar

At this point the archive is visible under
`pkgs.paideia-os/staging/libpdx-elevate/1.0.0/`.  The Paideia signing
bot polls staging; when it picks up this release it:

  a. Verifies the author sig against the `author_pk` fingerprint the
     paideia-os org has registered.
  b. Verifies the source-tree hash by unpacking the archive and
     recomputing.
  c. Signs the same canonical body under `paideia_root_pk` (the R32
     root) and replaces the sentinel in `[signature.paideia-root]`.
  d. Copies the resigned archive to
     `pkgs.paideia-os/main/libpdx-elevate/1.0.0/`.
  e. Updates `pkgs.paideia-os/main/index.pdxsig` to include the new
     `{name=libpdx-elevate, version=1.0.0, hash=<sha256-of-tarball>}`
     entry, re-signed under `paideia_root_pk`.

### 7. Verify from a clean machine

On a machine that does NOT hold `author_pk` or `paideia_root_pk`:

    pkg install libpdx-elevate
    #   resolved:  libpdx-elevate-1.0.0 (pkgs.paideia-os/main)
    #   verified:  author=paideia-os-team (ML-DSA-65)
    #   verified:  paideia-manifest (ML-DSA-65 root)
    #   cap-audit: [ (see caps.decl — no runtime caps for a library) ]
    #   installed: /pkgs/libpdx-elevate-1.0.0

### 8. Tag the release

    git tag -a -s v1.0.0 -m "libpdx-elevate 1.0.0 — R49 shared library, first signed release"
    git push origin v1.0.0

The `-s` produces a git signature under the author's OpenPGP key
(separate from ML-DSA-65 — this is the git-side attestation only).
The ML-DSA-65 signature that gates pkg install lives in
`manifest.pdxsig`, not in the git tag.

### 9. Update STATUS.md

Flip M5 rollup row to LANDED, add a "1.0.0 released" line.  Commit
directly to `main` (`chore/release-status: libpdx-elevate 1.0.0
landed`), reference this runbook.

---

## Rollback

If a bad archive reaches `pkgs.paideia-os/main/` and must be pulled:

1. Remove the archive path and rebuild `index.pdxsig` under
   `paideia_root_pk` without the entry.
2. Push a `libpdx-elevate 1.0.1` with a `CHANGELOG.md` entry
   naming the withdrawn 1.0.0 and the reason.
3. Never re-use `1.0.0` for a different tarball — semver is
   immutable at the mirror.

Local machines that already installed 1.0.0 continue to work (the
manifest.pdxsig they hold verifies against the paideia_root_pk they
already trusted); the pull just removes the *availability* from the
mirror.  A `pkg upgrade` on those machines will pick up 1.0.1.

---

## Escalations

- Author key compromise: rotate `author_pk`, re-sign every extant
  release under the new key, push all archives to staging in the
  order libpdx-cap → …→ libpdx-elevate.  Coordinate with the
  Paideia root re-sign gate to invalidate old-author-key manifests
  at `pkgs.paideia-os/main/`.
- Paideia root compromise: this is a §8 of `design/user/model.md`
  event; halt all mirror push activity and follow the R32
  root-rotation playbook (out of scope for this file).
