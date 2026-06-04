- Feature Name: `regression_baselines`
- Start Date: 2026-06-04
- RFC PR: [CMIP-REF/rfcs#0000](https://github.com/CMIP-REF/rfcs/pull/0000)

# Summary

Manage diagnostic regression baselines as **two layers**:

1. A small, human-readable **golden** (the REF-shaped `series.json` + CMEC bundles)
   committed directly to the source repo —
   this is the test *gate* and the in-PR *diff signal*.
2. The large, binary **native diagnostic outputs** stored outside git in an object store
   and referenced from a single git-committed **`native_manifest.json`**
   (sha256 digests of the golden, the native files, and the extraction inputs).

The REF CLI (`ref test-cases`) fetches, runs, and mints baselines.
The native outputs double as the **input fixtures that test the native→bundle extraction step**,
so most PRs are verified fast on a stock runner without re-running the (expensive) diagnostic.

The **object-store backend is left as an explicit open decision** (see Unresolved questions) —
this RFC specifies the *layering, manifest, CLI, and CI*,
which are backend-agnostic behind a small `NativeStore` interface.

# Motivation

Diagnostic regression testing today is flaky, opaque, and bloats the repo:

- **Bloat.** `tests/.../regression` is ~1.0 GB on disk.
  Of that, **696 MB is input `climate_data` that the capture step wrongly copies in**;
  the real curated output is ~316 MB (esmvaltool 5 MB, ilamb 61 MB, pmp 250 MB).
  Some native is already hand-`.gitignore`d, so it is *missing* — no coverage, no parity.
  The bloat is a capture bug plus committing binaries, not an inherent need.
- **No clear "this-changed" signal.**
  Failures are a mix; you dig CI logs across many jobs.
  The REF-shaped output (`series.json`, CMEC bundles) is small JSON that diffs cleanly in the PR
  and is exactly the content that feeds the REF API —
  it is what reviewers most need to see.
- **The extraction regression never runs on a PR.**
  It runs only post-merge/nightly today,
  so a PR that breaks extraction goes green, then red after merge.
- **Most diagnostic authors are external scientists.**
  Minimal setup, no new credentials.
  A contributor should need only the REF.
- **Security.**
  The integration runner shares a writable cache;
  untrusted fork code must never reach it or any write credential.

A key reframe: **the native outputs are not review-only artifacts —
they are the input fixtures for the extraction step.**
`Diagnostic.run()` is `execute()` (runs the diagnostic, writes native)
then `build_execution_result()` (reads native → CMEC bundle + series).
`RegressionValidator` already replays *committed native* through `build_execution_result`
without re-running the diagnostic.
That makes a fast, fork-safe extraction test possible on a stock runner — *if* the native is fetchable.

# Reference-level explanation

## Two layers, one manifest

For each `(provider, diagnostic, test-case)`:

```
packages/<provider>/tests/test-data/<diagnostic>/<test-case>/
  catalog.yaml            # committed: dataset metadata (exists today, content-hashed)
  catalog.paths.yaml      # gitignored: per-user local paths (exists today)
  regression/
    series.json           # committed GOLDEN (the gate + diff signal)
    diagnostic.json        # committed GOLDEN (CMEC metric bundle)
    output.json           # committed GOLDEN (CMEC output bundle)
  native_manifest.json    # committed: source of truth binding golden <-> native
```

The native bytes (provenance + matched `*.nc` / `*.png`) live in the object store, **not** in git.

`native_manifest.json`:

```json
{ "schema": 1,
  "golden":  { "series.json": "<sha256>", "diagnostic.json": "<sha256>", "output.json": "<sha256>" },
  "native":  { "<relpath>": { "sha256": "...", "size": ... } },
  "extraction_inputs": { "files_series_digest": "<sha256>", "input_selectors_digest": "<sha256>" } }
```

- `golden` binds the committed golden bytes to the manifest:
  CI recomputes the golden digests and asserts they equal `manifest.golden`.
  A commit that updates `series.json` but not the manifest (or vice-versa) **fails loudly** —
  closing the one consistency leg content-addressing does not cover.
- `native` lists the curated native blobs by content digest (immutable, dedups, survives history).
- `extraction_inputs` hashes the diagnostic's `files`/`series` patterns and the catalog `input_selectors` —
  the real extraction inputs the current catalog hash ignores —
  so a change to extraction behaviour bumps the manifest *at PR time*.

Two manifest sections keep the diff readable:
a changed `native` block means native changed;
a changed `golden` block means extraction output changed.

## Curated capture (kills the bloat at the source)

The diagnostic already declares which native files matter via `files` + `series` glob patterns;
production publishing copies only those (the CMEC `output.json` subset).
The regression capture must use the **same curation** instead of copying the whole working directory.
This alone prunes the 696 MB `climate_data` input and the verbose run logs for free.
Diagnostic authors control what is stored/pruned through those existing patterns — no new mechanism.

## `NativeStore` interface (backend-agnostic)

The REF talks to the store through a thin Protocol
so the backend is swappable and unit-testable,
and so this RFC does not have to settle the backend to be implementable:

```python
class NativeStore(Protocol):
    def has(self, digest: str) -> bool: ...
    def fetch(self, digest: str, dest: Path) -> None: ...   # by sha256; public read, no creds
    def put(self, path: Path) -> str: ...                    # returns sha256; needs write creds
```

Required properties of whatever backend is chosen:

- **Anonymous public read** so fork CI and contributors fetch without credentials.
- **Writes gated to trusted contexts only** (CI on `main`, or maintainers).
- Content-addressed (sha256) storage;
  native addressed **per-test-case** so a fetch is one diagnostic's output (KB–32 MB).
- Retention that **never expires a digest still referenced by a reachable manifest**.

Backend selection is deferred to a maintainer decision (Unresolved questions).

## CLI (extends the existing `ref test-cases` app)

| Subcommand | Purpose | Creds |
|------------|---------|-------|
| `run --provider P --diagnostic D` | run `execute()` + `build_execution_result`, write golden + manifest locally | none (local) |
| `sync [--provider P]` | fetch native blobs named by the committed manifest(s) | none (public read) |
| `replay --provider P --diagnostic D` | fetch native, run `build_execution_result`, compare to golden | none (public read) |
| `mint --provider P` | `run` + `put` native to the store + update manifest | write creds |

`mint` is the only credentialed verb and runs only on trusted contexts.

## Tiered CI

- **PR — any, incl forks — stock `ubuntu-latest`, no secrets.**
  `ref test-cases sync` (public read) → `replay` → diff golden → post a PR comment.
  Figures surfaced but **non-blocking**.
  Routed by changed-files so the check is *honest*:
  - PR touches extraction (`build_execution_result`, `files`/`series`) or golden/manifest → run the replay;
    the check is named **`extraction-replay`** and its green means what it says.
  - PR touches `execute()` only → replay is a tautology →
    emit a **neutral "execute-changed: needs runner verification"** check (not green),
    linking the bootstrap path.
  - A fetch *miss* for a fixture the manifest says should exist is a **failure, not a skip**
    (today, missing fixtures silently skip → false green).
- **Same-repo PR — self-hosted runner, no upload — when a golden file changes.**
  Run the slow `execute()` to confirm the diagnostic still reproduces its native,
  catching breakage pre-merge.
  Gated with no labels: `if: github.event.pull_request.head.repo.full_name == github.repository`.
- **Merge to `main` — self-hosted runner, the only job holding write creds.**
  `ref test-cases mint` → `execute()` + `put` native + update manifest.
- **Nightly — self-hosted runner.**
  Full sweep; drift detection backstop;
  opens an auto-PR with the new manifest + a human-readable summary if outputs drift.
- **Fork PR needing new/changed native** (a new diagnostic, or an `execute()` change):
  a documented `workflow_dispatch` a maintainer triggers
  to mint on the trusted runner and push the golden + manifest back to the PR.
  Forks never run `execute()` on the self-hosted runner (untrusted code, shared cache).

## Self-hosted runner security

The integration runner uses ephemeral pods,
but mounts a **shared, writable cache** (datasets + provider software).
Untrusted fork code in such a job could poison that cache for the next trusted run;
ephemeral pods do not close this.
Therefore: never run untrusted `execute()` on the shared runner;
keep write credentials on the `main` (and explicitly-triggered) path only;
gate the same-repo PR job by `head.repo.full_name == github.repository`.
(A separate hardened runner with a read-only cache could later make fork `execute()` safe;
out of scope here.)

## Determinism (prerequisite for comparing bundles)

CMEC bundle content comparison (currently only `series.json` is compared)
requires removing nondeterminism first:

- **Execution-dir timestamp.**
  ESMValTool writes a `recipe_<YYYYMMDD>_<HHMMSS>` dir that gets baked into `output.json`.
  Sanitise `recipe_\d{8}_\d{6}` → a placeholder in both capture and comparison.
  Tracked as [Climate-REF/climate-ref#713](https://github.com/Climate-REF/climate-ref/issues/713).
- **Floats.**
  Replay path (stored native → extraction) is deterministic → exact compare.
  Execute path (re-run) has float jitter → compare structure/dimensions only,
  leaving numeric values to the replay path;
  any value that must be compared on the execute path uses a relative tolerance, never exact equality.

# Drawbacks

- The REF CLI owns fetch/mint/credential plumbing —
  more surface to maintain and test than off-loading to an existing data-management tool.
- Native is a cache of a regenerable artifact:
  losing a digest still referenced by a reachable manifest breaks historical replay
  (mitigated by reachability-aware retention).
- Golden minting is effectively a maintainer/CI action for conda providers
  (esmvaltool's env is ~1373 packages);
  external authors usually cannot run `execute()` locally.
- Fork `execute()` changes have no automated PR signal until bootstrapped —
  the fast path tests extraction, not the diagnostic algorithm.
- Historical bloat remains: existing committed binaries are not rewritten out of history
  (deliberate — the repo is already used by externals).

# Rationale and alternatives

**Why two layers.**
Keeping the small REF-shaped golden in git
puts the review signal where reviewers already work (the PR diff)
and keeps the gate human-readable;
moving the large binaries out of git removes the bloat
while still making them fetchable for the extraction test.
The split maps cleanly onto what the REF already does for catalogs
(`catalog.yaml` committed, `catalog.paths.yaml` gitignored).

**Why a manifest + CLI rather than committing the bytes.**
Measured native is too large to commit (pmp 250 MB, ilamb 61 MB).
A content-addressed manifest gives dedup, per-commit pinning,
and a clean in-PR "native changed" diff without binaries in git.

**Alternatives considered:**

- **Commit curated native to git.**
  Rejected by measurement (too large for pmp/ilamb).
  Viable only for esmvaltool (~5 MB) — not worth a per-provider split of the storage model.
- **Git LFS.**
  Pointer churn remains; GitHub LFS egress costs; no anonymous public read.
- **Regenerate-on-demand, store nothing.**
  The fast fork-PR extraction path runs on a stock runner with no conda/data and cannot regenerate;
  it must fetch.
  Per-provider, pmp's 250 MB *could* be regenerated on the self-hosted runner rather than stored
  if its fetch rate is low — the `NativeStore` interface allows mixing.

**Impact of not doing this.**
Contributors keep hand-rolling baselines,
PRs keep shipping output changes without a visible diff,
and silent upstream regressions stay invisible until a manual check.

# Prior art

- The REF's existing **catalog split** —
  the exact "commit small metadata, keep bytes out of git" pattern, here extended to outputs.
- **pytest-regressions** (`--force-regen`) — the regenerate-the-golden workflow this mirrors.
- Content-addressed artifact stores with a small in-git lock (DVC, Snakemake) — same idea;
  this RFC keeps the client surface to just the REF CLI
  rather than adding a separate toolchain on contributors.

# Unresolved questions

- **Object-store backend choice** — the main open decision, deferred deliberately.
  Candidate backends behind the `NativeStore` interface include
  an S3-compatible bucket or an OCI registry (e.g. GitHub Packages);
  selection should weigh anonymous-read, trusted-write identity, egress cost,
  content-addressing/GC, and any storage the org has already provisioned.
  **No backend is assumed by this RFC.**
- Per-provider **store-vs-regenerate** for the large providers (pmp 250 MB).
- Retention policy for non-current native versions.
- Whether the PR comment should render a **figure diff** (PNG) / **NetCDF stat diff**,
  or defer to a follow-up.
- Exact form of the fork-PR **bootstrap `workflow_dispatch`**.

# Future possibilities

- Rich diff renderers in the PR comment (PNG diffs, NetCDF summary-stat diffs).
- Reuse the same `NativeStore` + manifest for large reference/observational inputs too big for git.
- A `ref test-cases preview` that serves locally what the API would render for a PR,
  straight from a synced native fixture (already the `replay` path).
- Signed manifests for end-to-end baseline provenance.
