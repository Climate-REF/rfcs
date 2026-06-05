- Feature Name: `regression_baselines`
- Start Date: 2026-06-04
- RFC PR: [CMIP-REF/rfcs#5](https://github.com/CMIP-REF/rfcs/pull/0005)

# Summary

Effectively diagnostic regression **baselines** for use by the testing suite.
Having baseline output for a diagnostic is an important contract between the development team and the diagnostic implementor.

Each baseline (one test-case run) has **two bundles**:

1. A small, human-readable **committed bundle** (the REF-shaped `series.json` + CMEC bundles)
   committed directly to the source repo.
   This should be stable and able to be reproduced by the CI.
2. A large, binary **native bundle** (the diagnostic's native outputs)
   stored outside git in an object store
   and referenced from a single git-committed **`manifest.json`**
   (sha256 digests of the committed bundle, the native files, and the extraction inputs).

The REF CLI (`ref test-cases`) fetches, runs, and mints baselines
with the goal of making it simple for diagnostic developers to contribute test output without requiring custom pytest implementation.

The **object-store backend is left as an explicit open decision** (see Unresolved questions) —
this RFC specifies the *layering, manifest, CLI, and CI*,
which are backend-agnostic behind a small `NativeStore` interface.

# Motivation

Diagnostic regression testing today is flaky, opaque, and bloats the repo:

- **Bloat.** `tests/.../regression` is ~1 GB on disk.
  The git repo size is growing in a way that isn't sustainable.
  We want to minimise the volume of data needed to setup the repo.
- **No clear "this-changed" signal.**
  It's hard to see the failures for new diagnostics as we aren't following a consitent process.
  The REF-shaped output (`series.json`, CMEC bundles) is small JSON that diffs cleanly in the PR
  and is what reviewers most need to see.
- **The extraction regression never runs on a PR.**
  It runs only post-merge/nightly today,
  so a PR that breaks extraction goes green, then red after merge.
  The integration tests are currently broken and require a refresh of the regression data.
- **Most diagnostic authors are external scientists.**
  Require minimal setup, no new credentials.
  A contributor should need only the REF.
- **Security.**
  The integration runner shares a writable cache so we must be careful that fork code must never reach it or any write credential.

The native bundle is not a review-only artifact as it is the input fixtures for integration tests.
`Diagnostic.run()` is `execute()` (runs the diagnostic, writes native)
then `build_execution_result()` (reads native -> CMEC bundle + series).
`RegressionValidator` already replays *committed native* through `build_execution_result` without re-running the diagnostic.
That makes a fast, fork-safe extraction test possible on a stock runner — *if* the native is fetchable.

# Reference-level explanation

## Two bundles, one manifest

For each `(provider, diagnostic, test-case)`:

```
packages/<provider>/tests/test-data/<diagnostic>/<test-case>/
  catalog.yaml            # committed: dataset metadata (exists today, content-hashed)
  catalog.paths.yaml      # gitignored: per-user local paths (exists today)
  regression/
    series.json           # committed bundle (the gate + diff signal)
    diagnostic.json        # committed bundle (CMEC metric bundle)
    output.json           # committed bundle (CMEC output bundle)
  manifest.json           # committed: source of truth binding committed bundle <-> native bundle
```

The native bytes (provenance + matched `*.nc` / `*.png`) live in the object store, **not** in git.

`manifest.json`:

```json
{ "schema": 1,
  "committed":  { "series.json": "<sha256>", "diagnostic.json": "<sha256>", "output.json": "<sha256>" },
  "native":  { "<relpath>": { "sha256": "...", "size": ... } },
  "extraction_inputs": { "files_series_digest": "<sha256>", "input_selectors_digest": "<sha256>" } }
```

- `committed` binds the committed bundle bytes to the manifest **for integrity only**:
  CI recomputes the committed digests and asserts they equal `manifest.committed`.
  A commit that updates `series.json` but not the manifest (or vice-versa) **fails loudly** —
  closing the one consistency leg content-addressing does not cover.
  This is a tamper/desync check on the in-repo files, **not** the regression comparator
  (a re-run is compared by *content with tolerance*, not by digest — see Comparison).
- `native` lists the curated native blobs by content digest (dedups, survives history).
  These digests are **authored only by `mint`**: the value is the sha256 of the exact bytes
  `mint` uploaded to the store, so it is a *fetch address + integrity check*, never a reproducibility gate.
- `extraction_inputs` hashes the diagnostic's `files`/`series` patterns and the catalog `input_selectors` —
  the real extraction inputs the current catalog hash ignores —
  so a change to extraction behaviour bumps the manifest *at PR time*.

**Native digests do not gate reproducibility.**
Native bytes are nondeterministic (`.nc`/`.png` embed timestamps, library versions, and provenance, floats jitter across platforms),
so re-running `execute()` on a different machine yields different bytes resulting in a different digest for an unchanged diagnostic.
The CI therefore never recomputes a native digest and compares it to the manifest.
A mismatch is expected, and not a failure.

The deterministic gate is the **committed bundle**, compared by content with tolerance (see Comparison).
The `native` block **churns on every real mint** (timestamps and maybe SHAs shift),
so it is not a clean "native meaningfully changed" signal.
Only the `committed` block is a semantic diff signal
Remint when the committed bundle changes or a new test-case lands, not on native drift.

The consistency actually enforced:

| pair                                                 | enforced              | how                                             |
| ---------------------------------------------------- | --------------------- | ----------------------------------------------- |
| in-repo committed bundle ↔ `manifest.committed`      | yes — integrity gate  | CI recompute equals (tamper/desync check)       |
| replayed committed bundle ↔ in-repo committed bundle | yes — regression gate | content compare with tolerance (see Comparison) |
| stored native bytes ↔ `manifest.native` digest       | by construction       | `mint` hashes what it uploads                   |
| re-executed native ↔ `manifest.native` digest        | **no — by design**    | nondeterministic; gate is the committed bundle  |

## Curated capture (kills the bloat at the source)

The diagnostic already declares which native files matter via `files` + `series` glob patterns
with production publishing copies only those (the CMEC `output.json` subset).
The regression capture must use the **same curation** instead of copying the whole working directory.
Diagnostic authors control what is stored/pruned through those existing patterns — no new mechanism.

## `NativeStore` interface

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

The backend will be prototyped using an R2 bucket on Cloudflare,
with the CI having write credentials to mint new baselines.

## CLI (extends the existing `ref test-cases` app)

| Subcommand                           | Purpose                                                                               | Creds              |
| ------------------------------------ | ------------------------------------------------------------------------------------- | ------------------ |
| `run --provider P --diagnostic D`    | run `execute()` + `build_execution_result`, write committed bundle + a *local* manifest | none (local)       |
| `sync [--provider P]`                | fetch native blobs named by the committed manifest(s)                                 | none (public read) |
| `replay --provider P --diagnostic D` | fetch native, run `build_execution_result`, compare to committed bundle               | none (public read) |
| `mint --provider P`                  | `run` + `put` native to the store + author the canonical `manifest.native`            | write creds        |

`mint` is the only credentialed verb, and the only writer of canonical native digests,
so anonymous users can fetch the native baseline.
A dev `run` produces native and a local manifest for previewing the committed-bundle diff;
its native digests are advisory and never committed.
A PR commits only the committed bundle + `manifest.{committed,extraction_inputs}` —
the `native` block is authored exclusively by the gated post-merge `mint`,
so native digests exist iff `mint` wrote them (no advisory-then-wrong window on `main`).

## Tiered CI

The PR tier runs **credential-free** so it can safely execute fork code:
the danger was never `execute()` itself,
it was `execute()` holding write creds on a shared writable cache.
Strip both and a fork's diagnostic runs harmlessly on a throwaway public runner.
Minting (the only `put`) stays post-merge behind a manual gate.

- **PR — any, incl forks — public `ubuntu-latest`, no secrets, no creds.**
  Routed by changed-files so the check is *honest*:
  - **New diagnostic / new test-case, or `execute()` changed** -> run `execute()` + `build_execution_result`
    on the public runner, then compare the freshly built committed bundle (tolerant).
    No stored baseline is fetched, so a brand-new test-case validates on its own PR — no follow-up needed.
    The check is named **`pr-execute`**.
  - **Extraction changed (`build_execution_result`, `files`/`series`, committed/manifest) and native already exists** ->
    cheap path: `ref test-cases sync` (public read) -> `replay` -> compare committed bundle.
    The check is named **`extraction-replay`**.
  - A fetch *miss* for an **existing** manifest entry is a **failure, not a skip**
    (today, missing fixtures silently skip -> false green);
    a *new* entry has no stored native by definition and takes the execute path instead.
  - **Coverage:** the public runner handles ~90% of test-cases.
    Some heavy ESMValTool ocean diagnostics exceed its compute/memory/disk and will fail there —
    those wait for the future private runner (see below); they are not a silent skip.
- **Merge to `main` — gated mint, the only context holding write creds.**
  A job bound to a GitHub **Environment (`mint`) with required reviewers**:
  the run pauses until a maintainer approves, releasing the write creds only onto **already-merged (trusted) code**.
  On approve: `ref test-cases mint` -> `execute()` + `put` native + author `manifest.native` (bot commit back to `main`).
  **Mint set** = test-cases whose committed bundle or `extraction_inputs` changed in the merge diff;
  a manual `workflow_dispatch` can target a specific `--provider`/`--diagnostic` for backfills or store-loss recovery.
- **Nightly — gated runner.**
  Full sweep; drift detection backstop;
  opens an auto-PR with the new manifest + a human-readable summary if outputs drift.
- **Future — private read-only runner.**
  For the heavy diagnostics the public runner cannot host.
  Safe to run fork code only if each job is **ephemeral / container-per-job**, mounts conda + data **read-only**,
  carries **no write creds**, and has **controlled egress** (no shared writable cache — the L46 hazard).
  Deferred: start public-only.

## PR workflow

Two phases: the diagnostic **author** first builds and runs the test-case **locally**,
then opens a PR that the **reviewer** drives through CI to merge.

### 1. Author a new diagnostic (local, no creds)

```mermaid
sequenceDiagram
    actor Author as Diagnostic author
    participant Repo as Local checkout
    participant CLI as ref test-cases
    participant Store as NativeStore

    Author->>Repo: add diagnostic (new class + test cases)
    Author->>CLI: ref test-cases run --provider P --diagnostic D
    CLI->>CLI: execute() then build_execution_result
    CLI-->>Repo: write committed bundle + local manifest

    opt native already minted (changed/existing diagnostic)
        Author->>CLI: ref test-cases replay --provider P --diagnostic D
        CLI->>Store: sync native (public read)
        CLI-->>Author: tolerant diff vs committed bundle
    end

    Author->>Author: preview committed-bundle diff (small JSON)
    Author->>Repo: commit committed bundle + manifest.committed/extraction_inputs
```

A local `run` produces native + advisory digests for preview only —
the author commits no `native` block.
Canonical `manifest.native` is authored later by the gated post-merge `mint`.

### 2. Make a PR (review + CI)

```mermaid
sequenceDiagram
    actor Author as Diagnostic author
    participant PR as PR / GitHub
    participant CI as PR CI (public ubuntu-latest, no creds)
    actor Reviewer as Reviewer / maintainer
    participant Mint as Mint job (gated env, write creds)
    participant Store as NativeStore

    Author->>PR: open PR (diagnostic + committed bundle + manifest.committed/extraction_inputs)
    PR->>CI: trigger checks (routed by changed files)

    alt new diagnostic or execute() changed
        CI->>CI: run execute() + build_execution_result (no creds)
        CI->>CI: compare freshly built committed bundle (tolerant)
        CI-->>PR: pr-execute check + bundle diff comment
    else extraction changed, native already exists
        CI->>Store: ref test-cases sync (public read)
        alt native fetch hit
            CI->>CI: replay then compare committed bundle (tolerant)
            CI-->>PR: extraction-replay check + bundle diff comment
        else native missing for an existing entry
            CI-->>PR: FAIL (no silent skip)
        end
    end

    Note over CI: heavy ESMValTool ocean diagnostics may exceed public limits (future private runner)

    Reviewer->>PR: review committed-bundle diff (small JSON, human-readable)
    Reviewer->>PR: approve + merge

    PR->>Mint: merge to main queues mint
    Mint->>Reviewer: pause for required-reviewer approval
    Reviewer->>Mint: approve (releases write creds onto trusted code)
    Mint->>Mint: execute() + build_execution_result
    Mint->>Store: put native (write creds)
    Mint->>PR: bot commit manifest.native to main
```

### 3. CI routing (decision view)

How the PR check is chosen from the changed files, and where each path lands:

```mermaid
flowchart TD
    A[PR opened] --> B{What changed?}

    B -->|"new diagnostic or execute() changed"| X["run execute() + build_execution_result on public runner, no creds"]
    B -->|"extraction only, native exists"| C["sync native (public read)"]

    C --> D{Native fetch hit?}
    D -->|yes| E["replay then rebuild committed bundle"]
    D -->|"no (existing entry)"| F["FAIL: no silent skip"]

    X --> G{Diff within tolerance?}
    E --> G
    G -->|yes| H["check green (pr-execute / extraction-replay)"]
    G -->|no| I["red: bundle diff comment, author fixes"]
    I --> A
    F --> A

    H --> M{Reviewer approves + merges?}
    M -->|no| A
    M -->|yes| P["merge to main queues gated mint"]
    P --> Q{Maintainer approves mint?}
    Q -->|yes| O["put native + author manifest.native (bot commit)"]
    Q -->|no| P
```

The PR runner is public and **credential-free**, so a fork's `execute()` never touches write creds or a writable cache —
the L46 hazard is removed, not gated around.
Minting is the only `put`, and it runs **post-merge behind a required-reviewer environment**,
so write creds reach only already-merged (trusted) code.
The public runner covers ~90% of test-cases;
heavy ESMValTool ocean diagnostics that exceed its limits wait for the future private read-only runner.

## Comparison

The regression gate compares a freshly replayed committed bundle to the in-repo reference
**by content, not by bytes**.
Byte/digest equality is too strict: it surfaces least-significant-bit float flips
and other cross-platform noise (a fork replays on stock `ubuntu-latest`, a mint ran elsewhere),
turning harmless jitter into red CI.
Content comparison needs awareness of each artifact's shape:

| artifact | comparison |
| -------- | ---------- |
| `series.json` (numeric) | structural keys exact; values by relative tolerance (`rtol`/`atol`) |
| CMEC `diagnostic.json` / `output.json` | structural JSON compare; numerics with tolerance; volatile fields (timestamps, abs paths, library versions, provenance) normalised out via an ignore-list |

- Tolerances are **declared per diagnostic/test-case** — a global default with per-case overrides,
  since some metrics are inherently noisier than others.
  (pytest-regressions precedent: `num_regression` tolerances, `data_regression` structural compare.)
- **Fast path:** if bytes are already equal, pass without the content compare; only mismatches fall through to the tolerant path.
- The `manifest.committed` sha256 is **not** this comparator — it is the integrity/desync check above.

## Determinism

Content comparison still requires removing avoidable nondeterminism at capture time:

- **Execution-dir timestamp.**
  ESMValTool writes a `recipe_<YYYYMMDD>_<HHMMSS>` dir that gets baked into `output.json`.
  Sanitise `recipe_\d{8}_\d{6}` -> a placeholder in both capture and comparison.
  Tracked as [Climate-REF/climate-ref#713](https://github.com/Climate-REF/climate-ref/issues/713).
- **Floats.**
  Any float values should be compared using a relative tolerance to the baseline.
  Resolving these kinds of CI-dev machine differences is time consuming.
- **Binary files.**
  We don't compare SHAs of generated files as they will differ depending on who generated the files (especially with NetCDF files).
  We shouldn't try and chase perfect hash consistency.
  The SHA in the manifest is for validating a fetched blob against its store address, not for gating reproducibility.

# Drawbacks

- The REF CLI owns fetch/mint/credential plumbing —
  more surface to maintain and test than off-loading to an existing data-management tool.
- Native is a cache of a regenerable artifact:
  losing a digest still referenced by a reachable manifest breaks historical replay
  (mitigated by reachability-aware retention).
- Committed-bundle minting is effectively a maintainer/CI action for conda providers.
  external authors usually cannot run `execute()` locally.
- The public PR runner covers ~90% of test-cases but not all:
  heavy ESMValTool ocean diagnostics exceed its compute/memory/disk
  and have no automated PR signal until the future private runner lands.
- Historical bloat remains: existing committed binaries are not rewritten out of history
  (deliberate — the repo is already used by externals).

# Rationale and alternatives

**Why two bundles.**
Keeping the small REF-shaped committed bundle in git
puts the review signal where reviewers already work (the PR diff)
and keeps the gate human-readable;
moving the large binaries out of git removes the bloat
while still making them fetchable for the extraction test.
The split maps cleanly onto what the REF already does for catalogs
(`catalog.yaml` committed, `catalog.paths.yaml` gitignored).

**Why a manifest + CLI rather than committing the bytes.**
Measured native is too large to commit (pmp 250 MB, ilamb 61 MB).
A content-addressed manifest gives dedup, per-commit pinning of the exact native bytes,
and a fetchable fixture without binaries in git.
The clean in-PR semantic diff comes from the committed bundle, not the native digests
(which churn on every mint — see the manifest section).

**Alternatives considered:**

- **Commit curated native to git.**
  Rejected by measurement (too large for pmp/ilamb).
  Viable only for esmvaltool — not worth a per-provider split of the storage model.
- **Git LFS.**
  Pointer churn remains; GitHub LFS egress costs; no anonymous public read.
- **Regenerate-on-demand, store nothing.**
  The cheap `extraction-replay` path skips conda + `execute()` by fetching stored native;
  dropping the store forces every PR onto the slow execute path.
  Storing native keeps the common extraction-only PR fast.
  Per-provider the `NativeStore` interface still allows mixing
  (a provider with a low fetch rate could regenerate instead of store).

**Impact of not doing this.**
Contributors keep hand-rolling baselines,
PRs keep shipping output changes without a visible diff,
and silent upstream regressions stay invisible until a manual check.

# Prior art

- The REF's existing **catalog split** —
  the exact "commit small metadata, keep bytes out of git" pattern, here extended to outputs.
- **pytest-regressions** (`--force-regen`) — the regenerate-the-committed-bundle workflow this mirrors.
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
- **Private read-only runner** for the heavy diagnostics the public runner cannot host —
  isolation model (ephemeral container, read-only mounts, no creds, controlled egress) and when to build it.
- Scope of the manual `mint` **`workflow_dispatch`** (targeting, backfills, store-loss recovery)
  beyond the automatic merge-diff mint set.

# Future possibilities

- Rich diff renderers in the PR comment (PNG diffs, NetCDF summary-stat diffs).
- Reuse the same `NativeStore` + manifest for large reference/observational inputs too big for git.
- A `ref test-cases preview` that serves locally what the API would render for a PR,
  straight from a synced native fixture (already the `replay` path).
- Signed manifests for end-to-end baseline provenance.
