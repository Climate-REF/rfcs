- Feature Name: `execution_lifecycle`
- Start Date: 2026-05-12
- RFC PR: [Climate-REF/rfcs#0000](https://github.com/Climate-REF/rfcs/pull/0000)

# Summary
[summary]: #summary

Consolidate the lifecycle of a single diagnostic execution
— allocation, dispatch, run, classify, publish, ingest, finalise —
into one deep module (`ExecutionLifecycle`) sitting behind a single `Transport` port,
and surface a declarative `ResourceHint` on each `Diagnostic` so providers
can express memory / CPU / wall-clock expectations in one place
without reaching into the executor or scheduler layer.

# Motivation
[motivation]: #motivation

## Problem

The lifecycle of one diagnostic execution today is fragmented across roughly
eight files in two packages.
A single execution flows through, in order:

1. `solver.py` creates an `Execution` row with a `PLACEHOLDER_FRAGMENT`,
   calls `assign_execution_fragment` to flush, compute a group-short string,
   and rewrite the output fragment.
2. `solver.py` rebuilds the `ExecutionDefinition` via `attrs.evolve`,
   calls `execution.register_datasets`, expunges the row, commits.
3. `executor.run(definition, execution)` is called outside the transaction.
   Three implementations exist:
   `SynchronousExecutor` runs in-process,
   `LocalExecutor` submits to a `ProcessPoolExecutor`,
   `CeleryExecutor` sends to a broker with linked result handlers.
4. `execute_locally` (in `climate-ref-core`) wraps `diagnostic.run`,
   classifies exceptions via the private `_is_system_error`,
   special-cases `CondaCommandError`,
   and produces an `ExecutionResult` — whose `build_from_output_bundle`
   factory writes three JSON files to disk despite being a frozen attrs class.
5. `process_result` and `handle_execution_result` copy log, metric bundle,
   output bundle, series file, and every bundle-referenced plot/data/html
   file from scratch to results,
   then ingest scalar values, series values, and outputs in a nested
   transaction.
6. `ExecutionGroup.dirty` is toggled in three different branches across
   `handle_execution_result` and `mark_execution_failed`.

Concrete problems this fragmentation causes today:

- **Provider authors have no obvious place to express parallelisation hints.**
  ESMValTool diagnostics that need 16 GB of memory or eight hours of
  wall-clock have nowhere to declare it.
  The `LocalExecutor` enforces a 6 hour per-task timeout;
  the `CeleryExecutor` does not enforce a per-task timeout at all;
  neither passes resource requirements through to anything that could honour
  them.
  When the project eventually adds SLURM or PBS adapters,
  the natural way to wire memory / CPU / wall-clock into `sbatch` or `qsub`
  directives does not exist.
- **Retry classification is split across the file tree.**
  `_is_system_error` (in `climate-ref-core/executor.py`),
  `CondaCommandError` handling (in the same file),
  missing-log detection (in `result_handling.py`),
  per-task timeout (in `LocalExecutor`),
  and pool-shutdown abandonment (also in `LocalExecutor`)
  each set `retryable` independently.
  There is no single classifier function and no contract between
  `ExecutionResult.build_from_failure` and the executor about which
  exception types are retryable.
- **The `dirty` flag is decided in three places.**
  Success path sets it `False`;
  non-retryable failure sets it `False`;
  retryable failure leaves it `True`;
  missing log leaves it `True`.
  These branches are spread across `handle_execution_result` and the
  Local/Celery executors with no unifying decision function.
- **Tests have to mock subprocess, filesystem, DB, and Celery.**
  `tests/unit/test_providers.py` patches `subprocess` directly for the
  conda provider; `tests/unit/test_executor.py` imports the private
  `_is_system_error`; integration tests are the only thing that exercises
  the full happy path.
- **`ExecutionResult` is not picklable cleanly.**
  Its frozen factory writes three JSON files;
  re-construction in a test requires a real tmp_path filesystem.
- **The `Executor` Protocol is shallow.**
  Its `run` / `join` interface forces every adapter
  (Synchronous, Local, Celery, future SLURM/PBS/K8s)
  to reinvent the same dance:
  reattach the detached `Execution` row,
  open a fresh transaction,
  copy scratch to results,
  load the CV,
  ingest scalars and series,
  apply the dirty rule,
  mark the row.

## Why now

The CMIP REF is approaching the point where serious deployment targets are
plausible:
SLURM and PBS adapters,
distributed Celery deployments behind a managed broker,
possibly Kubernetes Jobs.
Each new transport currently means a new ~250-LOC adapter that re-implements
the lifecycle dance.
The same fragmentation also blocks anything resembling
**informed scheduling** — even simple things like "ESMValTool needs more memory
than ILAMB" or "the ENSO diagnostic should land on the `bigmem` queue"
have no representation in the codebase.

This RFC is **not** about replacing SLURM, PBS, or Celery as schedulers.
It is about defining the seam between *climate-ref* and a scheduler so the
scheduler has enough information to do its job,
and so the lifecycle around the scheduler is one robust, well-tested module
instead of eight thin ones.

## Expected outcomes

1. The solver dispatch loop shrinks from ~50 lines of bookkeeping to ~3.
2. A new transport adapter (SLURM, PBS, K8s) is a single class
   implementing `Transport`, around 80–120 LOC,
   with no copy-paste of fragment allocation, ingestion, CV loading,
   dirty-flag logic, or `Execution` reattachment.
3. Provider authors declare `resources: ResourceHint(...)` once on a
   diagnostic class and the rest of the system honours it.
4. Per-execution telemetry (peak memory, duration, host)
   is captured on every outcome and persisted on the `Execution` row,
   making future adaptive scheduling
   a feature addition rather than a schema change.
5. Unit tests run with zero filesystem and zero subprocess
   via an `InMemoryTransport` and a separated bundle writer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Module boundary

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ ExecutionLifecycle (climate_ref.lifecycle)                                   │
│                                                                              │
│   __init__(config, db, transport, *,                                         │
│            retry=DefaultRetryPolicy(),                                       │
│            cv=None,           # default: CV.load_from_file(config.paths…)    │
│            clock=datetime.utcnow)                                            │
│                                                                              │
│   submit(execution, definition) -> None                                      │
│   drain(timeout=None) -> None                                                │
│                                                                              │
│   replay_abandoned() -> list[int]    # find executions stranded across boot  │
│   dry_run(group, datasets) -> ExecutionDefinition                            │
│                                                                              │
│   OWNED (private):                                                           │
│     _FragmentAllocator   _Classifier   _Promoter      _Ingestor              │
│     _DirtyRule           _BundleWriter (worker-side)                         │
└──────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ Port — the ONLY thing that varies per backend
┌──────────────────────────────────────────────────────────────────────────────┐
│ Transport (Protocol, climate_ref_core.lifecycle.ports)                       │
│                                                                              │
│   name: ClassVar[str]                                                        │
│   dispatch(envelope: ExecutionEnvelope) -> None                              │
│   poll(block: bool, timeout: float | None)                                   │
│       -> Iterator[ExecutionOutcome]                                          │
│   shutdown(timeout: float | None) -> None                                    │
│                                                                              │
│ Production adapters:                                                         │
│   InMemoryTransport      (in climate-ref, used by tests & SynchronousExec)   │
│   ProcessPoolTransport   (in climate-ref, replaces LocalExecutor body)       │
│   CeleryTransport        (in climate-ref-celery)                             │
│                                                                              │
│ Future adapters (NOT in this RFC, but the seam supports them):               │
│   SlurmTransport         consumes ResourceHint as --mem/--cpus/--time/-p     │
│   PbsTransport           consumes ResourceHint as -l mem,ncpus,walltime,q    │
│   K8sTransport           consumes ResourceHint as pod resources + deadline   │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Wire types

Three frozen, picklable value types cross the transport boundary
(process pool, broker, future network hop).
None of them carry DB sessions, `Config`, or controlled vocabulary.

```python
@attrs.frozen
class ResourceHint:
    """Declarative resource expectation, lives on the diagnostic class.

    Per-execution overrides are produced by Diagnostic.resources_for().
    """
    memory_mb: int = 4096
    cpu: int = 1
    wall_clock: timedelta = timedelta(hours=2)
    queue: str | None = None   # transport-specific routing tag

@attrs.frozen
class ExecutionEnvelope:
    """Wire format sent to a worker. Picklable."""
    execution_id: int
    definition: ExecutionDefinition
    resources: ResourceHint
    deadline: datetime         # clock() + resources.wall_clock at submit time

@attrs.frozen
class Telemetry:
    """Captured by the worker, persisted on the Execution row by the coordinator."""
    duration: timedelta
    peak_rss_mb: int | None
    host: str
    exit_code: int | None
    transport_meta: Mapping[str, str]   # slurm jobid / k8s pod / celery task_id …

@attrs.frozen
class ExecutionOutcome:
    """What poll() yields back to the lifecycle."""
    execution_id: int
    result: ExecutionResult | None       # None ⇒ transport-side abandonment
    failure: ExecutionFailure | None     # timeout | broker_lost | pool_shutdown
    telemetry: Telemetry
```

`ExecutionResult` becomes pure data.
The current `build_from_output_bundle` factory
(which writes three JSON files inside a frozen-attrs constructor)
is split into `ExecutionResult.from_bundle(definition, bundle)` —
which constructs the value object without I/O —
and a separate `_BundleWriter.write(definition, bundle)`
that is invoked worker-side only.

## Diagnostic-side declaration

```python
class Diagnostic(AbstractDiagnostic):
    ...
    resources: ResourceHint = ResourceHint()

    def resources_for(self, definition: ExecutionDefinition) -> ResourceHint:
        """Optional per-execution sizing.

        Default returns ``self.resources`` unchanged.
        Diagnostics whose memory scales with input size override this.
        """
        return self.resources
```

Concrete provider declarations under the new system:

```python
# packages/climate-ref-esmvaltool/src/climate_ref_esmvaltool/diagnostics/base.py
class ESMValToolDiagnostic(CommandLineDiagnostic):
    resources = ResourceHint(
        memory_mb=16000, cpu=4, wall_clock=timedelta(hours=6),
    )

# packages/climate-ref-pmp/src/climate_ref_pmp/diagnostics/enso.py
class EnsoDiagnostic(Diagnostic):
    resources = ResourceHint(
        memory_mb=24000, cpu=8, wall_clock=timedelta(hours=8),
        queue="bigmem",
    )

# packages/climate-ref-ilamb/src/climate_ref_ilamb/standard.py
class IlambDiagnostic(Diagnostic):
    resources = ResourceHint(
        memory_mb=8000, cpu=2, wall_clock=timedelta(hours=2),
    )
    def resources_for(self, defn):
        n = len(defn.datasets.get_cmip6())
        return attrs.evolve(self.resources, memory_mb=8000 + 500 * n)
```

The base `Diagnostic` default of `(4 GB, 1 CPU, 2 h)` is the project-wide
sensible starting point.
Providers that need more either override the class attribute
or override `resources_for` for input-size scaling.

## Solver dispatch — before and after

**Before** (`solver.py:709-757`, condensed):

```python
execution = Execution(
    execution_group=execution_group,
    dataset_hash=definition.datasets.hash,
    output_fragment=PLACEHOLDER_FRAGMENT,
    provider_version=potential_execution.provider.version,
)
db.session.add(execution)
fragment = assign_execution_fragment(
    db.session, execution,
    provider_slug=provider_slug,
    diagnostic_slug=potential_execution.diagnostic.slug,
    selectors=potential_execution.selectors,
    group_id=execution_group.id,
)
definition = attrs.evolve(
    definition,
    output_directory=config.paths.scratch.resolve() / pathlib.Path(fragment),
)
execution.register_datasets(db, definition.datasets)
if execute:
    db.session.expunge(execution)
    pending = (definition, execution)
# … commit happens at top of loop …
if pending is not None:
    executor.run(definition=pending[0], execution=pending[1])
…
executor.join(timeout=timeout)
```

**After**:

```python
lifecycle = ExecutionLifecycle(config, db, transport)

for group, datasets, definition in planned_executions:
    execution = Execution(execution_group=group,
                         dataset_hash=datasets.hash,
                         provider_version=definition.diagnostic.provider.version)
    lifecycle.submit(execution, definition)

lifecycle.drain(timeout=timeout)
```

`submit` privately handles fragment allocation,
`register_datasets`,
the expunge/commit dance,
and `transport.dispatch`.
`drain` privately handles polling,
reattachment,
artifact promotion,
ingestion,
the dirty rule,
and telemetry persistence.

## End-to-end sequence

```
Solver        Diagnostic       Lifecycle           Transport          Worker             DB
  │ submit(e,d)   │                │                    │                  │                 │
  ├──────────────►│ resources_for(d)│                   │                  │                 │
  │ ◄──── ResourceHint(...)        │                    │                  │                 │
  │              │                 │ allocate fragment ─┼──────────────────┼────────────────►│
  │              │                 │ register_datasets ─┼──────────────────┼────────────────►│
  │              │                 │ session.expunge                                          │
  │              │                 │ commit ──────────────────────────────────────────────── │
  │              │                 │ deadline = now + r.wall_clock                            │
  │              │                 │ dispatch(envelope)─►│  (slurm sbatch / celery send /    │
  │              │                 │                    │   processpool.submit / inline)     │
  │              │                 │                    ├─►│ execute_locally(envelope)      │
  │              │                 │                    │  │   diagnostic.run               │
  │              │                 │                    │  │   _BundleWriter.write          │
  │              │                 │                    │  │   CV.validate (hard-fail)      │
  │              │                 │                    │  │   capture Telemetry            │
  │              │                 │                    │  │   return ExecutionOutcome      │
  │              │                 │                    │◄─│                                │
  │ drain()      │                 │ poll() ◄───────────│                                    │
  │ ────────────►│                 │ _finalize:                                              │
  │              │                 │   session.merge(eid) ───────────────────────────────────►│
  │              │                 │   _Promoter.copy(scratch→results, log+bundles+refs)     │
  │              │                 │   _Classifier(failure or result) → Success|Retry|GiveUp │
  │              │                 │   _Ingestor.upsert(outputs, scalars, series) ──────────►│
  │              │                 │   _DirtyRule.apply(decision) ──────────────────────────►│
  │              │                 │   execution.telemetry = outcome.telemetry ─────────────►│
  │              │                 │   execution.mark_{successful,failed} ─────────────────►│
  │ done         │                 │                    │                                    │
```

## Retry classification

A single `RetryPolicy.classify(outcome) -> RetryDecision` replaces:
`_is_system_error`,
the `CondaCommandError` branch in `execute_locally`,
the missing-log branch in `handle_execution_result`,
the per-task timeout handling in `LocalExecutor.join`,
the pool-shutdown abandonment in `LocalExecutor._fail_outstanding`.

```python
class RetryDecision(enum.Enum):
    SUCCESS = "success"
    RETRY   = "retry"   # leaves dirty=True
    GIVE_UP = "give_up" # sets dirty=False, marks failed

class DefaultRetryPolicy:
    SYSTEM_ERRORS = (OSError, MemoryError, SystemExit, KeyboardInterrupt)
    NON_RETRYABLE = (CondaCommandError,)

    def classify(self, outcome: ExecutionOutcome) -> RetryDecision:
        if outcome.failure is not None:
            # transport-side: timeout, broker_lost, pool_shutdown → retry
            return RetryDecision.RETRY
        result = outcome.result
        if result is None:
            return RetryDecision.RETRY
        if result.successful:
            return RetryDecision.SUCCESS
        return (RetryDecision.RETRY if result.retryable
                else RetryDecision.GIVE_UP)
```

The dirty rule is derived from the decision in one place:

```python
class DefaultDirtyRule:
    def apply(self, execution: Execution, decision: RetryDecision) -> None:
        if decision in (RetryDecision.SUCCESS, RetryDecision.GIVE_UP):
            execution.execution_group.dirty = False
        # RETRY: leave dirty=True so the next solve picks it up
```

## Idempotent re-ingest

Re-running `_finalize` on the same `execution_id`
(e.g. a join-time crash followed by `replay_abandoned`)
must not double-insert.
The ingester moves to upserts keyed on natural identifiers:

- `ExecutionOutput`: `(execution_id, output_type, short_name)`
- `ScalarMetricValue`: `(execution_id, dimensions_hash)`
- `SeriesMetricValue`: `(execution_id, dimensions_hash, index_name)`

Where SQLite is in use the implementation uses
`INSERT … ON CONFLICT DO NOTHING`;
on PostgreSQL the same construct works natively.
The scratch-to-results copy uses `shutil.copy` with `exist_ok=True`
on the destination directory.

## Telemetry

Three new columns are added to `Execution`:

```sql
ALTER TABLE execution ADD COLUMN duration_seconds   FLOAT  NULL;
ALTER TABLE execution ADD COLUMN peak_rss_mb        INTEGER NULL;
ALTER TABLE execution ADD COLUMN telemetry_meta     JSON   NULL;
```

`telemetry_meta` is a small JSON blob with hostname, exit code,
and transport-specific identifiers (Celery task id, SLURM job id, …).
None of these are read by the system at solve time;
they exist so that a future adaptive resource provider
can ask "what was the rolling p95 memory for diagnostic X
on dataset shape Y?" without a schema migration.

The shape of that future provider is **not** in this RFC.
A natural seam is:

```python
class ResourceProvider(Protocol):
    def hint_for(self, definition: ExecutionDefinition) -> ResourceHint: ...
```

with a `StaticResourceProvider` (uses `diagnostic.resources_for`)
and a future `AdaptiveResourceProvider` (reads telemetry, applies multipliers).
`ExecutionLifecycle` would gain a `resources:` constructor kwarg defaulting
to `StaticResourceProvider`.
This is intentionally deferred — the column existing today
is the only commitment.

## Module / file layout

```
packages/climate-ref-core/src/climate_ref_core/
├── lifecycle/
│   ├── __init__.py            ← public re-exports
│   ├── ports.py               ← Transport, ResourceHint, Envelope, Outcome, Telemetry
│   ├── results.py             ← ExecutionResult (pure data) + from_bundle factory
│   └── worker.py              ← execute_worker(envelope) — picklable entry point
│
packages/climate-ref/src/climate_ref/
├── lifecycle.py               ← ExecutionLifecycle class
├── transports/
│   ├── inmemory.py            ← InMemoryTransport
│   └── processpool.py         ← ProcessPoolTransport (was LocalExecutor)
└── executor/                  ← deleted; the Executor Protocol goes away
│
packages/climate-ref-celery/src/climate_ref_celery/
└── transport.py               ← CeleryTransport (was CeleryExecutor)
```

The existing `Executor` Protocol in `climate-ref-core/executor.py`
and the three concrete executors are deleted.
`execute_locally` becomes a thin wrapper around `execute_worker`
kept for backwards compatibility for one release cycle, then removed.

## Test impact

**Deleted tests** (boundary tests will replace them):

- `tests/unit/test_executor.py::test_is_system_error_*`
  — private-import tests on `_is_system_error`
- `tests/unit/test_providers.py::test_conda_*` subprocess patch chains
  for the conda provider's `run()` are replaced by transport-level boundary tests
- `tests/unit/executor/test_synchronous.py::*` reattachment tests
- `tests/unit/executor/test_local.py::*` future-handling tests
  beyond what `ProcessPoolTransport`'s own tests cover
- Per-executor `mark_execution_failed` mock-chain tests

**New boundary tests** (against `ExecutionLifecycle` with `InMemoryTransport`):

- `submit` allocates a unique fragment per execution
- `submit` persists `register_datasets` even when the transport dispatch fails
- `drain` ingests one successful outcome end-to-end and sets `dirty=False`
- `drain` re-dispatches a retryable failure on the next solve
  by leaving `dirty=True`
- `drain` marks a non-retryable failure with `dirty=False` and `successful=False`
- `drain` is idempotent: re-draining the same outcome does not double-insert
  scalar or series rows
- `drain` enforces `resources.wall_clock` uniformly — a worker that exceeds
  the deadline produces a retryable failure regardless of transport
- CV validation fails hard with a `ResultValidationError` when metrics
  reference unknown dimensions (replaces today's silent `logger.warning`)
- `replay_abandoned` returns the IDs of executions whose rows exist
  but never received an outcome

# Drawbacks
[drawbacks]: #drawbacks

## Migration cost

This is not a small refactor.
The dispatch loop, three executor classes, result handling,
fragment allocation, and ingestion all move at once.
A staged migration is possible — `InMemoryTransport` and `ProcessPoolTransport`
can ship before `CeleryTransport` — but the lifecycle module itself has to
land in one piece because every transport depends on it.

## CV validation becomes a hard fail

Today, when scalar or series metric values do not conform to the controlled
vocabulary, `result_handling.py` logs a warning with a `TODO`.
This RFC promotes that to a hard `ResultValidationError`.
Any in-flight diagnostic with a CV mismatch will start failing.
This is the *intended* behaviour — silent CV drift is a real source of data
quality issues — but it is a behaviour change that needs deliberate handling
during rollout (e.g. a one-release-cycle deprecation period where the
violation is logged with `ERROR` instead of `WARNING` but does not yet raise).

## A single fat class owns more responsibility

`ExecutionLifecycle` ends up around 400–500 LOC.
That is intentional: the things it owns are already coupled across eight
files today, and module *depth* is the design goal.
But it does mean the file is large; reviewers should expect that.

## The Celery boundary still has one quirk

In the current code, Celery's `link=handle_result.s(...)` chains a result
handler that runs on the broker side of the boundary.
In the new design, `CeleryTransport.poll` is what feeds outcomes back into
`drain`.
This is cleaner, but it means a coordinator-side crash mid-drain can leave
results "in flight" in the broker that the coordinator has not yet observed.
The mitigation is `replay_abandoned()`, which on next startup queries
unfinished executions and re-polls the broker for them.
This is a robustness improvement over today
(where a coordinator crash silently strands rows in `successful=NULL`),
but it is not free —
it requires `CeleryTransport` to persist task IDs alongside execution IDs
in a small table or in `telemetry_meta` so the broker can be queried later.

## Resource hints can be wrong

If a provider declares 4 GB for a diagnostic that actually needs 16 GB,
the SLURM adapter will OOM-kill the job.
Today's `LocalExecutor` runs in the parent address space and "just works"
up to the host's memory.
With `ResourceHint` in place, getting the number wrong is a new failure mode.
Mitigations:

- The default `ResourceHint(memory_mb=4096, cpu=1, wall_clock=2h)`
  is generous enough that most diagnostics run unchanged.
- `ProcessPoolTransport` deliberately ignores everything except
  `wall_clock` — running on a developer laptop does not get harder.
- Telemetry capture means the first failed run produces the data needed
  to adjust the hint upward.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Three designs were considered in detail.
Sketches and trade-offs follow; the chosen design is a deliberate hybrid.

## A — Minimal interface (rejected as a pure form)

Two public methods (`submit`, `drain`), one Protocol (`Transport`).
Everything else collapses behind the facade and cannot be customised
without editing the module.

```
Public surface:   ████░░░░░░░░░░░░░░░░  (smallest)
Defaults baked:   ████████░░░░░░░░░░░░
Future bend:      ██████████░░░░░░░░░░  (one port + edit-in-place)
Migration churn:  ████████████████░░░░  (deletes Executor Protocol)
```

Strength: minimal cognitive load. Weakness: no place to declare
resource hints, no place for per-provider retry policies; both end up
requiring constructor kwarg additions on first concrete demand.

## B — Maximally flexible (rejected as a pure form)

Five ports (`Transport`, `ArtifactStore`, `RetryPolicy`, `IngestSink`,
`FragmentAllocator`), seven lifecycle hooks, plugin discovery via entry
points, schema-versioned wire types.

```
Public surface:   ████████████████████  (5 ports × N methods + 7 hooks)
Defaults baked:   ████░░░░░░░░░░░░░░░░  (everything injectable)
Future bend:      ████████████████████  (port-shaped for K8s/S3/Prom)
Migration churn:  ████████████████████  (new wire format + plugin registry)
Speculation tax:  ██████████░░░░░░░░░░  (hooks + FragmentAllocator-as-port)
```

Strength: every anticipated future requirement has a seam already.
Weakness: heavy speculation tax — five ports without five concrete
implementations driving them is exactly the shape that calcifies into
"the way things have always been done".

## C — Common-case optimised (rejected as a pure form)

One class with `dispatch`, `drain`, plus `replay_abandoned`, `dry_run`,
`ingest`, `with_transport`. Sensible defaults sourced from `Config`.

```
Public surface:   ████████░░░░░░░░░░░░  (1 hot method + 4 cold methods)
Defaults baked:   ████████████████████  (highest)
Future bend:      ████████░░░░░░░░░░░░  (single port; widen kwargs)
Migration churn:  ██████░░░░░░░░░░░░░░  (smallest; adapters survive)
Hidden leak:      ████░░░░░░░░░░░░░░░░  (Celery ignores on_done callback)
```

Strength: the solver call site collapses to one line per iteration.
Weakness: a `ResultSink` callback that Celery silently ignores is an
asymmetry that will trip someone, and the design has no place for
resource hints.

## The chosen hybrid

- **C's façade** — `ExecutionLifecycle` is one class with `submit` and
  `drain` as the hot path and a small set of cold-path entry points.
- **A's transport contract** — `Transport.dispatch(envelope)` plus
  `poll()` returning `Iterator[ExecutionOutcome]`. No `ResultSink`
  callback. The lifecycle owns the result loop.
- **A's bundle-writer separation** — `ExecutionResult` is pure data;
  bundle JSON write moves into a worker-side `_BundleWriter`.
- **B's `ExecutionEnvelope` and `Telemetry` wire types** — earned by
  the actual provider-side pain (no place for resource hints) and by the
  need to capture per-execution actuals for future adaptive sizing.
- **Defer the rest of B** — no `ArtifactStore` port (S3 mirror is
  hypothetical), no `IngestSink` priority chain (DB-only today),
  no `LifecycleHooks` (today's `logger.*` calls suffice),
  no `FragmentAllocator` port (one implementation foreseeable),
  no entry-point plugin registry.
- **Delete the `Executor` Protocol.** It is a shallow interface
  dressed up as flexibility; `SynchronousExecutor` becomes
  `ExecutionLifecycle(transport=InMemoryTransport())`.

## Impact of not doing this

Each future transport (SLURM, PBS, K8s) re-implements ~250 LOC of
lifecycle wiring per adapter.
Resource hints have to be retrofitted later through a new wire format,
which is a strictly larger change than introducing them at the same time
as the lifecycle consolidation.
Per-task timeout, CV validation, dirty-flag rules, and retry
classification stay scattered, and integration tests remain the only
honest signal that the system works end-to-end.

# Prior art
[prior-art]: #prior-art

## Dask's `distributed`

Dask's scheduler accepts `resources=` annotations on submitted tasks
(e.g. `client.submit(func, resources={"GPU": 1})`).
The pattern of declaring resource needs on the unit-of-work and letting
the scheduler honour them is the direct inspiration for `ResourceHint`.
Dask does not concern itself with what is "above" the scheduler
(catalog, ingest, controlled vocabulary),
which is the layer this RFC is consolidating.

## Snakemake / Nextflow

Both pipeline frameworks have first-class resource directives
(`resources: mem_mb=…, runtime=…`) on rules / processes,
which translate transparently into SLURM / PBS / K8s directives.
This RFC reuses that mental model at the diagnostic level.

## Airflow operators

Airflow's executor/operator split is the closest analogue:
operators carry the work definition, executors decide how to run it.
Airflow learned the hard way that an executor that owns result
handling is too narrow an interface; their `BaseExecutor.execute_async`
plus `sync` (poll) split is essentially what `Transport.dispatch` /
`Transport.poll` is here.

## Celery's `task_time_limit` and queue routing

Celery already supports per-queue routing and per-task time limits.
This RFC's `ResourceHint.queue` maps directly onto Celery's queue
routing; `wall_clock` maps onto `task_time_limit` plus
`task_soft_time_limit`.
The `CeleryExecutor` in `climate-ref-celery` does not currently use
either; the new `CeleryTransport` does.

## The Rust RFC process itself

The shape of this document (Summary, Motivation, Reference-level,
Drawbacks, Rationale, Prior art, Unresolved, Future) is inherited from
[rust-lang/rfcs](https://github.com/rust-lang/rfcs)
via this repository's template.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

To resolve through the RFC discussion:

- **Should `ResourceHint` include I/O class (e.g. `io_intensive: bool`)
  or GPU (`gpu: int`)?**
  Neither has a concrete consumer today.
  The recommended default is to add fields when a concrete adapter
  needs them rather than speculating now.
- **Where does the controlled vocabulary live in test contexts?**
  Today `CV.load_from_file(config.paths.dimensions_cv)` is invoked
  per-result inside `handle_execution_result`.
  The RFC moves it to lifecycle construction, which means tests need
  a deterministic CV.
  A `PermissiveCV()` test fixture is one option;
  baking the project's CV into a constant for test use is another.
- **Should `replay_abandoned` be called automatically on
  `ExecutionLifecycle.__init__`?**
  Doing so makes recovery transparent;
  not doing so makes startup behaviour explicit.
  The current draft assumes it is explicit and called from the CLI.

To resolve through implementation:

- **Exact unique-constraint shape for upserts** —
  `(execution_id, output_type, short_name)` for `ExecutionOutput`
  is the obvious key, but the existing schema does not enforce it.
  An Alembic migration is needed.
- **`CeleryTransport.poll` semantics** —
  whether to use `AsyncResult.get(timeout=…)` per task
  or `app.control.inspect` for batch status.
  Implementation choice; does not affect the public interface.
- **`Telemetry.peak_rss_mb` capture on macOS** —
  `resource.getrusage` returns bytes on macOS and KB on Linux.
  A small platform shim is required worker-side.

Out of scope:

- **Adaptive resource provider** (reading rolling telemetry to suggest
  hints). Telemetry columns land here; the provider is a follow-up RFC.
- **GPU scheduling**.
  None of the current diagnostics use GPUs;
  when one does, `ResourceHint` gains a `gpu: int` field.
- **Multi-tenant queues**.
  `ResourceHint.queue` is enough for single-deployment routing;
  multi-tenant work (per-org queues, fair-share) is out of scope.
- **Replacing SLURM / PBS / Celery as schedulers**.
  Explicitly *not* in scope: the lifecycle owns the seam *above* the
  scheduler, not the scheduler itself.

# Future possibilities
[future-possibilities]: #future-possibilities

The deepening proposed here unlocks several follow-ups, each a
self-contained piece of work:

- **SLURM and PBS transports.**
  Each is a ~100 LOC adapter implementing `Transport`.
  `dispatch` builds a job script from `envelope.resources`,
  shells out to `sbatch` / `qsub`, records the job ID in `telemetry_meta`;
  `poll` queries `squeue` / `qstat` and surfaces completed jobs back as
  `ExecutionOutcome`s.
  No changes to the lifecycle or to provider code are required.

- **Adaptive `ResourceProvider`.**
  Reads `Execution.peak_rss_mb` over a rolling window per
  `(diagnostic_slug, dataset_shape_signature)`,
  suggests a memory hint at the rolling p95 × 1.2,
  applies a per-CLI flag to opt in.
  Schema is already in place from this RFC; the provider is the only
  new code.

- **Per-provider retry policies.**
  Currently `RetryPolicy` is a single injected object.
  ESMValTool's memory-exhaustion failures are retryable with a larger
  memory hint; a `Mapping[str, RetryPolicy]` keyed on provider slug
  becomes the natural extension when concrete demand arrives.

- **Streaming partial results.**
  `Transport.poll` already yields outcomes incrementally;
  a `PartialOutcome` event could be added to the iterator for
  progress reporting if a UI consumer appears.

- **S3 / HTTP artifact store.**
  If results need to land somewhere other than the local filesystem,
  an `ArtifactStore` port can be carved out of `_Promoter` later.
  The wire format does not change (scratch path remains relative);
  only the lifecycle's promote step is rewritten.

- **Pluggable ingest sinks.**
  Prometheus metrics, audit logs, S3 mirrors — each is a side-effect
  observer over `ExecutionOutcome`.
  A small `IngestSink` interface with ordered observers is a small
  follow-up if and when a second sink (beyond the DB) materialises.

- **Recovery on coordinator restart.**
  `replay_abandoned()` is in this RFC.
  Extending it to interrogate transport-side state (SLURM job that
  finished while the coordinator was down) is a natural follow-up that
  only needs `Transport.lookup(transport_meta) -> ExecutionOutcome | None`
  on each adapter.

None of the above are reasons to accept this RFC on their own;
they are listed to show that the seam introduced here is the right shape
for the directions the project is plausibly heading,
without baking any of them in prematurely.
