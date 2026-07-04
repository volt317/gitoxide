# DIVERGENCE.md — this fork has permanently split from GitoxideLabs/gitoxide

## What happened

Up to commit `032aee458f49e9f6bf75993052fbd1287356f309`, this branch (`local_profile`) was a
near-mirror of upstream gitoxide plus 3 commits making `gix-protocol` optional (see the git log
for those — they touch `gix-protocol`, `gix-refspec`, `gix-shallow`, and `gix`'s own manifest/lib,
~90 lines total). Depending on it via `git` from an outside project meant Cargo had to validate
every manifest in upstream's ~70-crate workspace to resolve anything — including fuzz targets,
CLI tooling, and GUI code nothing here uses. That's how a fuzz target's `edition2024` requirement
(`gix-imara-diff/fuzz/Cargo.toml`) ended up breaking a build that never touched it.

This commit cuts that cord. The workspace is trimmed to exactly what's needed, and everything
else — the `gitoxide`/`ein` CLI and its entire dependency tree, `gix-tui`, `gix-tix`, `gix-fsck`,
`gix-lfs`, `gix-rebase`, `gix-sequencer`, `gix-macros`, `gix-note`, `gix-fetchhead`,
`tests/it`, and every crate's own `fuzz/` directory — is gone. `tests/tools` (gix-testtools) was
kept: it's this fork's own dev-dependency for testing itself with `cargo test --workspace`, adds
no networking or CLI surface, and losing it would mean this fork could no longer validate itself
independently before anything gets pulled downstream.

**This is a one-way door.** There is no path back to being "gitoxide plus a small patch" — the
workspace shape itself has changed. Future upstream changes need to be pulled in deliberately,
crate by crate, not merged wholesale.

## What's here: 57 crates

- **34 actively compiled**, traced with `cargo tree` against
  `gix = { default-features = false, features = ["sha1", "revision"] }` — the discover / resolve
  ref / read object path this fork exists for.
- **22 present but never compiled under that feature set** — `gix`'s own manifest declares them
  `optional = true` path dependencies, and Cargo requires a declared path dependency to exist and
  parse successfully even when its feature is off. This is the entire networking stack
  (`gix-protocol`, `gix-transport`, `gix-credentials`) plus worktree/status/submodule/filter
  machinery. Confirmed inert via `cargo tree` — present as reviewable source, linked into nothing.
- **`gix` itself**, plus **`tests/tools`** for this fork's own test suite.

## Tracking upstream from here

```bash
git remote add upstream https://github.com/GitoxideLabs/gitoxide.git
git fetch upstream

# What's changed in a specific crate since this fork split off?
git log 032aee458..upstream/main -- gix-hash/

# See the actual diff for review before deciding whether to pull it in
git diff 032aee458 upstream/main -- gix-hash/
```

Do this per-crate, on demand — e.g. when a CVE or correctness fix lands upstream in something this
fork actually uses. Don't re-merge upstream's `main` wholesale; that reintroduces everything this
divergence removed. If a pulled-in fix touches a crate's own dependencies, re-run the same
process used to build this fork in the first place: trace the real closure with `cargo tree`
against the features actually used, and only add what that closure requires.

## Downstream consumer

Vitriol (github.com/<owner>/Vitriol) vendors a copy of this trimmed set directly into its own repo
under `crates/vendor/gitoxide/` rather than depending on this fork via `git` — see that
directory's `PROVENANCE.md` for why (removes the build-time dependency on this repo staying
reachable, keeps the exact code under Vitriol's own review). When this fork picks up an upstream
fix worth having, re-vendor: copy the updated crate(s) into Vitriol's vendor tree and update its
`PROVENANCE.md`.
