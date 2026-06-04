- Feature Name: `execution_lifecycle`
- Start Date: 2026-05-12
- RFC PR: [Climate-REF/rfcs#0003](https://github.com/Climate-REF/rfcs/pull/3)

# Summary

Consolidate the lifecycle of one diagnostic execution
— allocation, dispatch, run, classify, publish, ingest, finalise —
into one deep module (`ExecutionLifecycle`) behind one `Transport` port.
Add a declarative `ResourceHint` on every `Diagnostic`
and capture per-execution `Telemetry`,
so providers can express memory / CPU / wall-clock once
and future schedulers (SLURM, PBS, K8s)
plug in as thin adapters that only translate the envelope
and poll job state.

# Motivation

The lifecycle of one execution is currently fragmented across ~8 files
in 2 packages.
A single happy-path run touches
`solver.py` (allocates row + fragment, register_datasets, expunge, commit),
`climate_ref_core/executor.py` (`execute_locally`, `_is_system_error`,
`CondaCommandError` handling),
`climate_ref_core/diagnostics.py` (`ExecutionResult.build_from_output_bundle`,
a static factory that writes three JSON files to disk as a side effect of
constructing the result),
`climate_ref/executor/result_handling.py` (scratch→results copy ×4,
ingestion in nested-tx, dirty-flag toggled in 3 branches of the result path),
`climate_ref/executor/fragment.py` (PLACEHOLDER_FRAGMENT, group_short),
and one of `synchronous.py` / `local.py` / `climate_ref_celery/executor.py`
(each reimplementing the same reattach / commit / mark dance).

Concrete consequences today:

- **Providers have no place to declare parallelisation hints.**
  ESMValTool diagnostics that need 16 GB or 8 h have nowhere to say so;
  `LocalExecutor` applies one 6 h per-task timeout to every diagnostic
  (a single constructor default, not per-diagnostic),
  `CeleryExecutor` enforces no per-task timeout at all,
  and a future SLURM/PBS adapter has nothing to translate into `sbatch`/`qsub`.
- **Retry classification is scattered** across `_is_system_error`,
  the `CondaCommandError` branch, missing-log handling, `LocalExecutor`'s
  per-task timeout, and pool-shutdown abandonment.
- **The `dirty` flag is decided in three branches of the result path**
  (success path, non-retryable failure, retryable failure / missing log).
  User-initiated `dirty=True` resets in `cli/executions.py` (rerun / reset)
  are deliberately separate and stay outside the consolidated rule.
- **CV validation is silenced** (`logger.warning` with TODO instead of raising).
- **Tests mock subprocess + filesystem + DB + Celery** and import private
  helpers (`_is_system_error`) to compensate for the missing seam.
- **`ExecutionResult` construction is entangled with disk I/O.**
  It already pickles across the `ProcessPoolExecutor` boundary today
  (`_process_run` returns it to the parent), so picklability is not the
  problem — the problem is that the factory cannot be exercised without a
  writable output directory, which forces filesystem fixtures into every
  unit test that builds a result.

The CMIP REF is approaching deployment targets (SLURM / PBS / K8s).
The recurring lifecycle logic — fragment allocation, the reattach / commit /
mark dance, scratch→results promotion, retry classification — is already
shared in `result_handling.py`; what each new transport reimplements is the
dispatch / poll / timeout wiring around it.
This RFC pulls that shared logic behind one seam so a transport contributes
only its dispatch and poll, not a copy of the lifecycle.
This RFC is **not** about replacing SLURM, PBS, K8s, or Celery as schedulers.
It is about defining a single robust seam *above* them.

# Reference-level explanation

## Module boundary

```mermaid
classDiagram
    direction LR

    class ExecutionLifecycle {
        <<climate_ref.lifecycle>>
        +submit(execution, definition)
        +drain(timeout) None
        +replay_abandoned() list~int~
        -_FragmentAllocator
        -_Classifier
        -_Promoter
        -_Ingestor
        -_DirtyRule
        -_BundleWriter
    }

    class Transport {
        <<Protocol>>
        +name: ClassVar~str~
        +dispatch(envelope: ExecutionEnvelope) None
        +poll(block, timeout) Iterator~ExecutionOutcome~
        +shutdown(timeout) None
    }

    class InMemoryTransport { tests + SynchronousExecutor }
    class ProcessPoolTransport { replaces LocalExecutor }
    class CeleryTransport { climate-ref-celery }
    class SlurmTransport { future · sbatch --mem --cpus --time }
    class PbsTransport { future · qsub -l mem,ncpus,walltime }
    class K8sTransport { future · pod resources + deadline }

    ExecutionLifecycle ..> Transport : dispatch / poll
    InMemoryTransport ..|> Transport
    ProcessPoolTransport ..|> Transport
    CeleryTransport ..|> Transport
    SlurmTransport ..|> Transport
    PbsTransport ..|> Transport
    K8sTransport ..|> Transport
```

## Wire types

Picklable value objects only. No DB sessions, `Config`, or CV cross the boundary.

`ResourceHint` lives in `climate_ref_core` (the `Diagnostic` base class
declares it, and core cannot import the application package).
`ExecutionEnvelope`, `Telemetry`, `ExecutionOutcome`, and the `Transport`
protocol live in `climate_ref.lifecycle` alongside `ExecutionLifecycle`.

The default `wall_clock` is **6 h**, matching today's `LocalExecutor`
per-task timeout, so diagnostics that run for hours without declaring
`resources` keep their current budget (Celery enforces no limit today, so
nothing regresses there either). Tightening the default is a separate,
explicit decision.

```python
@attrs.frozen
class ResourceHint:
    memory_mb: int = 4096
    cpu: int = 1
    wall_clock: timedelta = timedelta(hours=6)   # matches current LocalExecutor budget
    queue: str | None = None    # transport-specific routing tag

@attrs.frozen
class ExecutionEnvelope:
    execution_id: int
    definition: ExecutionDefinition
    resources: ResourceHint
    # wall_clock travels in `resources`; the *deadline* is computed by the
    # transport when the job starts running, not here — a queued SLURM/PBS
    # job may wait hours before it begins, so anchoring the deadline at
    # submit time would expire jobs before they start.

@attrs.frozen
class Telemetry:
    duration: timedelta
    peak_rss_mb: int | None
    host: str
    exit_code: int | None
    transport_meta: Mapping[str, str]   # slurm jobid / k8s pod / celery task_id

@attrs.frozen
class ExecutionOutcome:
    execution_id: int
    result: ExecutionResult | None       # None ⇒ transport-side abandonment
    failure: ExecutionFailure | None     # timeout | broker_lost | pool_shutdown
    telemetry: Telemetry
```

`ExecutionResult` becomes pure data.
The current `build_from_output_bundle` factory is split into a pure
`ExecutionResult.from_bundle(definition, bundle)` and a worker-side
`_BundleWriter.write(definition, bundle)` that owns the JSON I/O.

## Diagnostic-side declaration

```python
class Diagnostic(AbstractDiagnostic):
    resources: ResourceHint = ResourceHint()        # project-wide default

    def resources_for(self, definition: ExecutionDefinition) -> ResourceHint:
        """Optional per-execution sizing. Default returns self.resources."""
        return self.resources


# Examples
class ESMValToolDiagnostic(CommandLineDiagnostic):
    resources = ResourceHint(memory_mb=16000, cpu=4, wall_clock=timedelta(hours=6))

class EnsoDiagnostic(Diagnostic):
    resources = ResourceHint(memory_mb=24000, cpu=8,
                             wall_clock=timedelta(hours=8), queue="bigmem")

class IlambDiagnostic(Diagnostic):
    resources = ResourceHint(memory_mb=8000, cpu=2, wall_clock=timedelta(hours=2))
    def resources_for(self, defn):
        n = len(defn.datasets.get_cmip6())
        return attrs.evolve(self.resources, memory_mb=8000 + 500 * n)
```

## Solver dispatch — before and after

```python
# Before: ~50 lines of bookkeeping in solver.py:709-757
#   PLACEHOLDER_FRAGMENT, assign_execution_fragment, attrs.evolve,
#   register_datasets, expunge, commit, executor.run, … executor.join

# After:
lifecycle = ExecutionLifecycle(config, db, transport)

for group, datasets, definition in planned_executions:
    execution = Execution(execution_group=group, dataset_hash=datasets.hash,
                          provider_version=definition.diagnostic.provider.version)
    lifecycle.submit(execution, definition)

lifecycle.drain(timeout=timeout)
```

## End-to-end sequence

```mermaid
sequenceDiagram
    autonumber
    participant S as Solver
    participant D as Diagnostic
    participant L as ExecutionLifecycle
    participant T as Transport
    participant W as Worker
    participant DB

    S->>L: submit(execution, definition)
    L->>D: resources_for(definition)
    D-->>L: ResourceHint(...)
    L->>DB: allocate fragment + register_datasets + expunge + commit
    L->>T: dispatch(ExecutionEnvelope)

    Note over T,W: sbatch · qsub · pool.submit · celery send · inline
    Note over T: deadline = job_start + resources.wall_clock (transport-side)
    T->>W: hand off envelope
    activate W
    W->>W: diagnostic.run · _BundleWriter.write · CV.validate · Telemetry
    W-->>T: ExecutionOutcome
    deactivate W

    S->>L: drain(timeout)
    loop until no in-flight executions
        L->>T: poll(block, timeout)
        T-->>L: ExecutionOutcome
        Note over L: _Classifier → SUCCESS | RETRY | GIVE_UP
        L->>DB: merge · promote artifacts · upsert outputs/scalars/series
        L->>DB: _DirtyRule.apply · save telemetry · mark_*
    end
```

## Retry + dirty rule (one source of truth)

Classification happens in two clearly separated places.

**Worker side** — exception → outcome. The worker runs the diagnostic and
maps the raised exception (or clean return) onto the booleans carried by
`ExecutionResult` / `ExecutionFailure`. This is where the exception-type
knowledge lives, because the exception is only ever raised on the worker:

```python
# worker-side, inside the run path
SYSTEM_ERRORS = (OSError, MemoryError, SystemExit, KeyboardInterrupt)  # → retryable
NON_RETRYABLE = (CondaCommandError,)                                   # → give up
```

This consolidates the exception-classification logic that is **today**
spread across `_is_system_error` and the separate `CondaCommandError`
branch in `execute_locally` into one worker-side function.

**Coordinator side** — outcome → decision. The policy never inspects
exception types; it maps the already-classified outcome onto a decision,
so the same rule applies identically to every transport (including remote
ones where the exception object never comes back):

```python
class RetryDecision(enum.Enum):
    SUCCESS = "success"
    RETRY   = "retry"     # leaves dirty=True
    GIVE_UP = "give_up"   # sets dirty=False, marks failed

class DefaultRetryPolicy:
    def classify(self, outcome: ExecutionOutcome) -> RetryDecision:
        if outcome.failure is not None:  # timeout | broker_lost | pool_shutdown
            return RetryDecision.RETRY
        r = outcome.result
        if r is None:                    return RetryDecision.RETRY
        if r.successful:                 return RetryDecision.SUCCESS
        return RetryDecision.RETRY if r.retryable else RetryDecision.GIVE_UP
```

Net effect: exception classification goes from two scattered sites to one
worker-side function, and the *transport-level* outcomes that today live in
the per-executor `join` loops (missing log, per-task timeout, pool-shutdown
abandonment) collapse into the single coordinator policy above.

## Idempotent ingest, telemetry

Ingestion uses `INSERT … ON CONFLICT DO NOTHING` on natural keys
(`(execution_id, output_type, short_name)` for outputs,
`(execution_id, dimensions_hash[, index_name])` for metric/series values),
so `replay_abandoned()` is safe.
Scratch-to-results copy uses `exist_ok=True`.

`DO NOTHING` (rather than `DO UPDATE`) is correct because every key is
scoped to `execution_id`, and each solve mints a **new** `Execution` row per
attempt: a retry produces a fresh `execution_id`, so its values never
collide with the abandoned attempt's. The conflict clause therefore only
guards re-ingestion of the *same* `execution_id` during replay — it never
silently keeps stale values from a previous attempt.

New `Execution` columns: `duration_seconds`, `peak_rss_mb`, `telemetry_meta JSON`.
No solver code reads these today; they exist so a future adaptive
`ResourceProvider` is a feature addition rather than a schema migration.

## Test impact

Delete: `_is_system_error` private-import tests,
subprocess patch chains in `test_providers.py`,
per-executor reattach tests,
`mark_execution_failed` mock chains.

Add boundary tests against `ExecutionLifecycle` + `InMemoryTransport`:
unique fragment per submit;
end-to-end success → `dirty=False`;
retryable failure leaves `dirty=True`;
non-retryable failure → `dirty=False`, `successful=False`;
re-drain idempotent (no double-insert);
`wall_clock` enforced uniformly across transports;
CV mismatch raises `ResultValidationError`;
`replay_abandoned` returns stranded IDs.

# Drawbacks

- **Celery loses fire-and-forget ingestion — the biggest trade-off.**
  Today `CeleryExecutor` attaches `link` / `link_error` callbacks
  (`handle_result` / `handle_failure`) so a worker ingests its own result
  with no live coordinator; the submitting process can exit immediately.
  The pull model (`Transport.poll` feeding `drain`) couples ingestion to a
  coordinator that stays alive for the whole batch. This is a real
  regression for the distributed case and is accepted deliberately: it buys
  one ingestion path and uniform retry/dirty handling across transports,
  and `replay_abandoned` (backed by `CeleryTransport` persisting task IDs
  alongside execution IDs) recovers a coordinator crash mid-drain. If
  detached submission turns out to be a hard requirement, a worker-side
  `IngestSink` callback can be added later without changing the seam — but
  the draft does **not** preserve it, and reviewers should weigh that.
- **Migration is wide and the seam swap is atomic.** Staggering applies to
  *adding* transports later, not to the cutover: `climate-ref-core` (wire
  types, `ResourceHint` on `Diagnostic`), `climate-ref`
  (`ExecutionLifecycle`, the transports), `climate-ref-celery`, and an
  Alembic migration all land together, because the seam replaces the
  `Executor` protocol the solver calls. Proposed landing order to bound
  risk: (1) add `ResourceHint` + telemetry columns (additive, no behaviour
  change); (2) introduce `ExecutionLifecycle` + `InMemoryTransport` +
  `ProcessPoolTransport` behind the existing solver entry point with the old
  executors still present; (3) port Celery; (4) delete the old executors.
- **Deleting the `Executor` protocol is a breaking public change.**
  `import_executor_cls` resolves an executor from a dotted path in `Config`,
  so the executor class is a documented extension point and any downstream
  custom executor implements it. The cutover must ship a deprecation cycle:
  keep `import_executor_cls` resolving known names to the new transports,
  warn on custom FQNs, and provide a config-migration note. This is not yet
  spelled out and is a precondition for merge.
- **CV becomes hard-fail.** Today's silent `logger.warning` becomes a
  raise. Intentional, but needs a one-cycle deprecation window where
  the violation is `ERROR` but not raised — and it ships in the same wide
  migration as the seam swap, so it must be feature-flagged to keep the two
  behaviour changes independently bisectable.
- **One fat class (~400–500 LOC).** Intentional depth, but reviewers
  should expect a large file. (LOC is an estimate, not a target.)
- **Resource hints can be wrong.** SLURM will OOM-kill a job whose
  declared memory is too low. Mitigated by a default that preserves current
  behaviour (4 GB / 1 CPU / 6 h), by `ProcessPoolTransport` ignoring
  everything except `wall_clock`, and by telemetry capture making the first
  failed run actionable.

# Rationale and alternatives

Three designs were considered.
The chosen interface is a deliberate hybrid.

```mermaid
quadrantChart
    title Design trade-off space
    x-axis "Surface area (concepts)" --> "Larger"
    y-axis "Defaults baked in" --> "More"
    quadrant-1 "Heavy & opinionated"
    quadrant-2 "Lean & opinionated"
    quadrant-3 "Lean & open"
    quadrant-4 "Heavy & open"
    "A - Minimal": [0.18, 0.55]
    "B - Maximally flexible": [0.92, 0.18]
    "C - Common-case optimised": [0.38, 0.92]
    "Hybrid (chosen)": [0.42, 0.7]
```

| Dimension             | A — Minimal | B — Maximal | C — Common-case | **Hybrid** |
| --------------------- | :---------: | :---------: | :-------------: | :--------: |
| Public surface        |      1      |      5      |        2        |     2      |
| Defaults baked in     |      3      |      1      |        5        |     4      |
| Bend without editing  |      3      |      5      |        2        |     3      |
| Migration churn       |      4      |      5      |        2        |     3      |
| Resource-hint support |      0      |      5      |        0        |     5      |
| Speculation tax       |      0      |      3      |        0        |     1      |

- **A — Minimal**: 2 methods, 1 port, everything else hidden.
  No place for resource hints or per-provider retry without later kwarg growth.
- **B — Maximal**: 5 ports (Transport, ArtifactStore, RetryPolicy,
  IngestSink, FragmentAllocator) + 7 hooks + entry-point plugin registry.
  Earned the wire-type split and `ResourceHint`; everything else is
  speculation.
- **C — Common-case**: one class, defaults sourced from `Config`,
  solver call site collapses to one line.
  A `ResultSink` callback that Celery silently ignores is an asymmetry
  that will trip someone, and there is still no place for resource hints.

**Chosen hybrid**: C's façade (one class, hot/cold method split) +
A's transport contract (`dispatch(envelope)` + `poll() -> Iterator[Outcome]`,
no result callback) + A's `BundleWriter` separation +
B's `ExecutionEnvelope`/`Telemetry` wire types.
Dropping the result callback is what costs Celery its fire-and-forget
ingestion (see Drawbacks); it is chosen for one uniform pull path, and a
worker-side `IngestSink` remains a non-breaking future addition.
Deferred: `ArtifactStore`, `IngestSink`, `LifecycleHooks`,
`FragmentAllocator` as a port, plugin registry.
The shallow `Executor` Protocol and the three concrete executors
are deleted (with a deprecation cycle for `import_executor_cls`; see
Drawbacks).

**Impact of not doing this**: each new transport reimplements its own
dispatch / poll / timeout wiring and re-derives retry and dirty handling
inline (the shared promotion/ingest helpers in `result_handling.py` already
exist, but nothing forces a new transport to route through them);
resource hints retrofit later through a new wire format (strictly larger
change); per-task timeout, CV validation, dirty-flag, and exception
classification stay scattered.

# Prior art

- **Dask `distributed`** — `resources=` annotations on submitted tasks
  inspire `ResourceHint`.
- **Snakemake / Nextflow** — first-class `resources:` directives
  translate transparently into SLURM / PBS / K8s. Same mental model at
  the diagnostic level.
- **Airflow** — executor / operator split; `BaseExecutor.execute_async`
  - `sync` is essentially `Transport.dispatch` + `Transport.poll`.
- **Celery** — `task_time_limit` + queue routing. `ResourceHint.queue`
  maps onto Celery queues; `wall_clock` onto `task_time_limit` /
  `task_soft_time_limit`. Today's `CeleryExecutor` uses neither.
- **Rust RFC process** — document shape inherited via this repo's
  template.

# Unresolved questions

To resolve through this RFC:

- Should `ResourceHint` include `gpu: int` / `io_intensive: bool`?
  Recommended default: add when a concrete adapter needs them.
- Where does the CV come from in tests? `PermissiveCV()` fixture
  vs. project CV baked into a constant.
- Is `replay_abandoned` automatic on `__init__` or explicit from the CLI?
  Draft assumes explicit.
- Is detached (coordinator-free) submission a hard requirement for Celery
  deployments? If yes, a worker-side `IngestSink` ships with the cutover
  rather than as a deferred addition.

To resolve through implementation:

- Exact upsert unique constraints + Alembic migration.
- Deprecation path for `import_executor_cls` / the executor FQN config key
  (name → transport mapping, warning on custom executors, migration note).
- `CeleryTransport.poll` semantics (per-task `AsyncResult.get` vs batch
  inspection).
- `peak_rss_mb` capture across macOS / Linux (`getrusage` unit difference).

Out of scope:

- Adaptive resource provider (telemetry columns land here; the
  provider is a follow-up RFC).
- GPU scheduling, multi-tenant queues.
- Replacing SLURM / PBS / Celery as schedulers.

# Future possibilities

Each item below is a self-contained follow-up enabled by this RFC:

- **SLURM / PBS transports** — thin adapters: `dispatch` builds a job
  script from `envelope.resources` and submits it, `poll` queries
  `squeue` / `qstat` and computes the deadline from job start time.
- **Adaptive `ResourceProvider`** — reads `Execution.peak_rss_mb` over
  a rolling window, suggests memory hint at p95 × 1.2.
- **Per-provider retry policies** — single `RetryPolicy` becomes
  `Mapping[str, RetryPolicy]` keyed on provider slug when concrete
  demand arrives.
- **Streaming partial outcomes** — `Transport.poll` already incremental;
  add a `PartialOutcome` event when a UI consumer appears.
- **S3 / HTTP artifact store** — extract `_Promoter` into an
  `ArtifactStore` port when a non-local target lands.
- **Pluggable ingest sinks** — Prometheus, audit log, S3 mirror as
  ordered `IngestSink` observers when a second sink materialises.
- **Cross-restart recovery** — extend `replay_abandoned` with
  `Transport.lookup(transport_meta)` so a SLURM job finished while the
  coordinator was down is picked up.

None of these are reasons to accept this RFC on their own;
they show the seam is the right shape for the directions the project is
plausibly heading, without pre-baking any of them.
