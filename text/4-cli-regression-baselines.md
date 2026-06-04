- Feature Name: `cli_regression_baselines`
- Start Date: 2026-06-04
- RFC PR: [CMIP-REF/rfcs#4](https://github.com/CMIP-REF/rfcs/pull/4)

# Summary
[summary]: #summary

Manage diagnostic regression baselines as **two layers with one source of truth**:

1. A small, human-readable **golden** (the REF-shaped `series.json` + CMEC bundles) committed
   directly to the source repo — this is the test *gate* and the in-PR *diff signal*.
2. The large, binary **native diagnostic outputs** stored in an object store and addressed by a
   single git-committed **`native_manifest.json`** (sha256 of golden, native, and extraction inputs).

The REF CLI (`ref test-cases`) is the only tool a contributor needs — it fetches, runs, and mints
baselines directly. There is **no DataLad / git-annex** in the contributor or CI path. The native
outputs double as the **input fixtures that test the native→bundle extraction step**, so most PRs
are verified fast on a stock runner without re-running the (expensive) diagnostic.

This RFC deliberately **shares the storage backend proposed in the `datalad_r2_baselines` RFC**
(Cloudflare R2 + the `auth.climate-ref.org` Worker for OIDC/PAT-gated writes + public anonymous
reads). It differs only in the developer-facing layer: a committed JSON golden + manifest + the REF
CLI, instead of git-annex pointer datasets and per-provider baseline repos.

# Motivation
[motivation]: #motivation

The pain points are the same as those in `datalad_r2_baselines` (outputs too large/binary for git,
no shared baselines, no PR-visible drift, no scheduled re-run, must work across forks/orgs). We
restate the constraints that drive *this* design specifically:

- **Most diagnostic authors are external scientists.** Minimal setup, no new creds, no new tooling.
  DataLad/git-annex is acknowledged (in that RFC's own Drawbacks) to "freak out potential
  contributors" and to present unreadable pointer files on GitHub. We want a contributor to need
  *only* the REF.
- **The "what-changed" signal must live in the PR diff itself**, reviewable with inline comments —
  not behind a `datalad get`. The REF-shaped output (`series.json`, CMEC bundles) is small JSON
  that diffs cleanly; it is also exactly the content that feeds the REF API, so it is what reviewers
  most need to see.
- **Failures must be obvious without log-archaeology across many CI jobs.** Today the regression
  (`test_cases`) suite runs only post-merge/nightly and surfaces a mix of failures with no crisp
  "this test-case changed" signal.
- **The repo must not bloat.** Measured today: `tests/.../regression` is ~1.0 GB on disk, of which
  **696 MB is input `climate_data` that the capture step wrongly copies in**; real curated output
  is ~316 MB (esmvaltool 5 MB, ilamb 61 MB, pmp 250 MB). The bloat is a capture bug plus committing
  binaries, not an inherent need.
- **Security on self-hosted runners.** The integration runner (`arc`) shares a writable cache;
  untrusted fork code must never reach it or any write credential.

A key reframe motivates the two-layer split: **the native outputs are not review-only artifacts —
they are the input fixtures for testing the extraction step.** `Diagnostic.run()` is
`execute()` (runs the diagnostic, writes native) followed by `build_execution_result()` (reads
native → CMEC bundle + series). The existing `RegressionValidator` already replays *committed
native* through `build_execution_result` without re-running the diagnostic. That makes a fast,
fork-safe extraction test possible on a stock runner — *if* the native is fetchable.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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
  native_manifest.json    # committed: the single source of truth (below)
```

The native bytes (provenance + matched `*.nc` / `*.png`) live in the object store, **not** in git.

`native_manifest.json`:

```json
{ "schema": 1,
  "golden":  { "series.json": "<sha256>", "diagnostic.json": "<sha256>", "output.json": "<sha256>" },
  "native":  { "<relpath>": { "sha256": "...", "size": ... } },
  "extraction_inputs": { "files_series_digest": "<sha256>", "input_selectors_digest": "<sha256>" } }
```

- `golden` binds the committed golden bytes to the manifest: CI recomputes the golden digests and
  asserts they equal `manifest.golden`. A commit that updates `series.json` but not the manifest (or
  vice-versa) **fails loudly** — closing the one consistency leg content-addressing does not cover.
- `native` lists the curated native blobs by content digest (immutable, dedups, survives history).
- `extraction_inputs` hashes the diagnostic's `files`/`series` patterns and the catalog
  `input_selectors` — the real extraction inputs that the existing `catalog` hash ignores — so a
  change to extraction behaviour bumps the manifest *at PR time*.

Two manifest sections keep the diff readable: a changed `native` block means native changed; a
changed `golden` block means extraction output changed.

## Curated capture (kills the bloat at the source)

The diagnostic already declares which native files matter via `files` + `series` glob patterns;
production publishing copies only those (the CMEC `output.json` subset). The regression capture must
use the **same curation** instead of copying the whole working directory. This alone prunes the
696 MB `climate_data` input and the verbose run logs for free. Diagnostic authors control what is
stored/pruned through those existing patterns — no new mechanism.

## Storage backend (shared with `datalad_r2_baselines`)

We reuse, unchanged:

- The Cloudflare **R2** bucket as the object store (free egress; public anonymous reads).
- The **`auth.climate-ref.org` Worker** that mints short-lived, prefix-scoped, S3-compatible
  credentials gated on GitHub identity (PAT for laptops, **Actions OIDC** with a repo allow-list for
  CI). This is the identity gate that makes "fork CI reads without secrets, only trusted contexts
  write" work — we do not reinvent it.

The REF talks to this backend through a thin Protocol so the backend is swappable and unit-testable:

```python
class NativeStore(Protocol):
    def has(self, digest: str) -> bool: ...
    def fetch(self, digest: str, dest: Path) -> None: ...   # by sha256; public read, no creds
    def put(self, path: Path) -> str: ...                    # returns sha256; needs write creds
```

Native is addressed **per-test-case** so a fetch is one diagnostic's output (KB–32 MB), not a
whole provider. Object key (publicurl view), mirroring the datalad RFC's scheme so both can coexist
on one bucket:

```
<provider>/<diagnostic>/<test-case>/<schema-version>/<sha256>
```

No git-annex layer: the manifest *is* the commit→bytes mapping, and the REF CLI fetches digests
directly over the public URL (read) or via Worker-minted creds (write).

## CLI (extends the existing `ref test-cases` app)

| Subcommand | Purpose | Creds |
|------------|---------|-------|
| `run --provider P --diagnostic D` | run `execute()` + `build_execution_result`, write golden + manifest locally | none (local) |
| `sync [--provider P]` | fetch native blobs named by the committed manifest(s) | none (public read) |
| `replay --provider P --diagnostic D` | fetch native, run `build_execution_result`, compare to golden | none (public read) |
| `mint --provider P` | `run` + `put` native to the store + update manifest | OIDC/PAT (write) |

`mint` is the only credentialed verb and is intended for CI on trusted contexts (and maintainers).

## Tiered CI

- **PR — any, incl forks — stock `ubuntu-latest`, no secrets.** `ref test-cases sync` (public read)
  → `replay` → diff golden → post a PR comment. Figures are surfaced but **non-blocking**. Routed by
  changed-files so the check is *honest*:
  - PR touches extraction (`build_execution_result`, `files`/`series`) or golden/manifest → run the
    replay; the check is named **`extraction-replay`** and its green means what it says.
  - PR touches `execute()` only → replay is a tautology → emit a **neutral
    "execute-changed: needs runner verification"** check (not a green), linking the bootstrap path.
    This avoids a green that a reviewer would misread as coverage.
  - A fetch *miss* for a fixture the manifest says should exist is a **failure, not a skip** (today,
    missing fixtures silently skip → false green).
- **Same-repo PR — `arc` runner, no upload — when a golden file changes.** Run the slow `execute()`
  to confirm the diagnostic still reproduces its native, catching breakage pre-merge. Gated with no
  labels: `if: github.event.pull_request.head.repo.full_name == github.repository`.
- **Merge to `main` — `arc`, the only job holding write creds.** `ref test-cases mint` →
  `execute()` + `put` native + update manifest. (OIDC repo allow-list via the Worker.)
- **Nightly — `arc`.** Full sweep; drift detection backstop; opens an auto-PR with the new manifest
  + a human-readable summary if outputs drift.
- **Fork PR needing new/changed native** (a new diagnostic, or an `execute()` change): a documented
  `workflow_dispatch` a maintainer triggers to mint on the trusted runner and push the golden +
  manifest back to the PR. Forks never run `execute()` on `arc` (untrusted code, shared cache).

## Determinism (prerequisite for comparing bundles)

CMEC bundle content comparison (currently only `series.json` is compared) requires removing
nondeterminism first:

- **Execution-dir timestamp.** ESMValTool writes a `recipe_<YYYYMMDD>_<HHMMSS>` dir that gets baked
  into `output.json`. Sanitise `recipe_\d{8}_\d{6}` → a placeholder in both capture and comparison.
  Tracked as [Climate-REF/climate-ref#713](https://github.com/Climate-REF/climate-ref/issues/713)
  (give ESMValTool a stable test execution dir).
- **Floats.** Replay path (stored native → extraction) is deterministic → exact compare. Execute
  path (re-run) has float jitter → compare structure/dimensions only, leaving numeric values to the
  replay path; any value that must be compared on the execute path uses a relative tolerance, never
  exact equality.

## Worked example (esmvaltool, a new diagnostic by an external author)

1. Author writes the diagnostic + its `files`/`series` patterns, runs `ref test-cases run` locally
   (or, for a heavy conda provider, asks a maintainer / CI to mint). This produces the golden JSON
   and the manifest.
2. Author commits golden + manifest in their PR. On push, the **PR fast path** (`extraction-replay`)
   on a stock runner fetches main's native (or, for a brand-new case, the maintainer-minted native),
   replays extraction, and posts the diff comment. The reviewer sees the REF-shaped output and any
   figures directly in the PR.
3. On merge, `main` mints the native to R2 and updates the manifest.

# Drawbacks
[drawbacks]: #drawbacks

- **No per-commit annex guarantee.** DataLad gives "this commit pins exactly these bytes" natively;
  we reproduce it with a manifest + a CI digest check rather than git-annex's symlink integrity. The
  check is one hash recompute, but it is our code, not the tool's.
- **The REF CLI owns more.** Fetch/mint/credential plumbing lives in `climate-ref` instead of in a
  battle-tested tool (git-annex). More surface to maintain and test.
- **Native is a cache of a regenerable artifact.** If the store loses a digest still referenced by a
  reachable manifest, historical replay breaks (mitigated by reachability-aware retention).
- **Golden minting is effectively a maintainer/CI action for conda providers** (esmvaltool's env is
  ~1373 packages); external authors usually cannot run `execute()` locally. The bootstrap dispatch
  makes this explicit but it is manual toil per such PR.
- **Fork `execute()` changes have no automated PR signal** until bootstrapped — the fast path only
  tests extraction, not the diagnostic algorithm.
- **Historical bloat remains.** This is future-looking; existing committed binaries are not rewritten
  out of history (deliberately — the repo is already used by externals).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

**Why this design.** It reuses the storage/identity backend the org has already provisioned (R2 +
Worker + OIDC + public read), so the hard, security-sensitive part is shared with
`datalad_r2_baselines`. It then optimises the *developer-facing* layer for the dominant audience —
external scientists who should need only the REF — and puts the review signal where reviewers
already work: the PR diff of small JSON. It reuses machinery the REF already has (the `files`/`series`
curation hook, the catalog split, sanitisation, and `RegressionValidator`'s replay), so it is mostly
wiring, not new subsystems.

**Relationship to `datalad_r2_baselines`.** This is the **CLI/manifest alternative** to that RFC's
**DataLad/git-annex** approach, over the **same backend**. The two are mutually exclusive at the
developer layer; they agree on R2, the Worker, OIDC, public read, content-addressing, schema-version
bumps, and nightly drift detection.

| Axis | This RFC (CLI + manifest) | `datalad_r2_baselines` |
|------|---------------------------|------------------------|
| Contributor tooling | REF only | + DataLad + git-annex |
| Review surface | committed JSON golden diff + manifest | git-annex pointer files |
| Repo layout | monorepo; native keyed by provider in one store | per-provider baseline git repos / subdatasets |
| Commit→bytes pin | manifest + CI digest check | git-annex native |
| Dedup / CAS | sha256 in manifest + store | git-annex keys |
| Reads (forks) | public R2 URL, no creds | public R2 URL, no creds |
| Writes | Worker OIDC/PAT (same) | Worker OIDC/PAT (same) |

**Other alternatives considered:**

- **GitHub Packages / OCI (ORAS) as the store.** Attractive (native CAS+GC, `GITHUB_TOKEN` read for
  forks, free egress) and was our earlier lean — but the R2 + Worker backend already exists and
  already solves the fork-read / trusted-write identity problem, so adopting OCI would fork the
  infrastructure for no net gain. Kept only as a contingency if R2 economics change.
- **Commit curated native to git.** Rejected by measurement (pmp 250 MB, ilamb 61 MB). Viable only
  for esmvaltool (~5 MB); not worth a per-provider split of the storage model.
- **Git LFS.** Pointer churn remains; GitHub LFS egress costs; no anonymous public read. Weaker than
  R2 on every axis we care about.
- **Per-provider baseline repos (as in the datalad RFC).** Rejected here: Jared is not sold on
  splitting providers (they must stay in lockstep with core), and a contributor touching three
  providers would juggle three baseline repos plus the source. We keep one store keyed by provider.

**Impact of not doing this.** Either we adopt DataLad (accepting contributor friction) or contributors
keep hand-rolling baselines, PRs keep shipping output changes without a visible diff, and silent
upstream regressions stay invisible until a manual check.

# Prior art
[prior-art]: #prior-art

- The REF's existing **catalog split** (`catalog.yaml` committed + `catalog.paths.yaml` gitignored)
  is the exact "commit the small metadata, keep the bytes out of git" pattern, here extended to
  outputs.
- **pytest-regressions** (`--force-regen`) — the regenerate-the-golden workflow this mirrors.
- **DVC / Snakemake** S3-artifact patterns — content-addressed artifacts with a small in-git lock;
  our manifest is the same idea without the full DVC toolchain on contributors.
- The sibling **`datalad_r2_baselines`** RFC — same backend, different (heavier) client model; this
  RFC is the deliberate lightweight counterpart.
- Conda-forge feedstocks (per-package repos) — the model the datalad RFC follows for the
  provider-repo split and which we explicitly decline for now.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- **Decide between this RFC and `datalad_r2_baselines`** (or a hybrid: this client model, with
  DataLad available as an optional power-user path on the same bucket). This is the core decision the
  two RFCs jointly pose.
- Per-provider **store-vs-regenerate**: pmp's curated native is 250 MB; if its fork-PR fetch rate is
  low it may be cheaper to regenerate on `arc` than to keep a fetchable fixture. The Protocol allows
  mixing per provider.
- **Retention** of non-current native versions (R2 object versioning + lifecycle) — align with the
  datalad RFC's 90-day proposal.
- Whether the PR comment should render a **figure diff** (PNG) and **NetCDF stat diff**, or keep that
  to a follow-up.
- Exact form of the **bootstrap `workflow_dispatch`** for fork PRs that introduce new native.
- Whether to add a CI check that flags **extraction-touching diffs with no manifest change** as a
  belt-and-braces guard on top of the `extraction_inputs` digest.

# Future possibilities
[future-possibilities]: #future-possibilities

- **Rich diff renderers** in the PR comment (side-by-side PNG diffs, NetCDF summary-stat diffs).
- **Reuse the same `NativeStore` + manifest** for large reference/observational inputs that are too
  big for git but smaller than ESGF-class data.
- **Reproduce the public/API output locally** straight from a synced native fixture (this is already
  the `replay` path) — a `ref test-cases preview` that serves what the API would render for a PR.
- **Signed manifests** (sigstore) so consumers can verify baseline provenance end-to-end.
- If providers are eventually split into separate repos, the per-provider object-key prefix already
  supports it without restructuring the store — the only change is where the golden + manifest are
  committed.
