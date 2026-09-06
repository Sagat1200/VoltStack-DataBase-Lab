# 106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Execution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 106 — Database Migration Execution System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Execution System` define la arquitectura runtime responsable de consumir un `MigrationPlan` previamente construido y coordinar su ejecución real contra uno o más targets de base de datos.

Este sistema materializa el límite entre:

```text
planned evolution
```

y:

```text
real database effects
```

La pregunta fundamental es:

> **¿Cómo ejecuta VoltStack un MigrationPlan de forma controlada, observable, cancelable, consistente y recuperable sin mezclar planning, compilation, repository history y driver execution?**

La arquitectura deberá coordinar:

- validación del plan;
- verificación de stale state;
- locks de migración;
- transactions;
- preconditions;
- schema operations;
- data operations;
- backfills;
- barriers;
- runtime validations;
- postconditions;
- retries;
- cancellation;
- deadlines;
- failure classification;
- outcome certainty;
- partial execution;
- repository updates;
- cleanup;
- telemetry;
- recovery information.

---

# 2. Principio central

> **Migration Execution transforma una estrategia inmutable en efectos runtime controlados; no redefine la estrategia durante su ejecución.**

Por tanto:

```text
MigrationPlan
=
immutable strategy
```

mientras:

```text
MigrationExecution
=
mutable runtime orchestration
```

y:

```text
Migration Executor
≠
Migration Planner
```

---

# 3. Separaciones fundamentales

VoltStack deberá mantener:

```text
Migration Definition
≠
Migration Plan
≠
Migration Execution
≠
Schema Execution
≠
Query Execution
≠
Migration Repository
```

También:

```text
Execution State
≠
Repository State
```

y:

```text
Failure
≠
Rollback
≠
Recovery
```

---

# 4. Posición arquitectónica

```text
Migration Catalog
       │
       ▼
Definitions
       │
       ▼
Migration Planner
       │
       ▼
MigrationPlan
       │
       ▼
┌───────────────────────────────┐
│ Migration Execution System    │
│                               │
│ Validation                    │
│ Locking                       │
│ Preconditions                 │
│ Transactions                  │
│ Schema/Data Execution         │
│ Backfills                     │
│ Barriers                      │
│ Retry                         │
│ Postconditions                │
│ Repository Recording          │
│ Cleanup                       │
└───────────────────────────────┘
       │
       ▼
Database / Repository
```

---

# 5. Responsabilidades

El Execution System deberá:

- validar que el plan sigue siendo ejecutable;
- verificar repository freshness;
- verificar schema assumptions cuando existan;
- resolver el runtime target;
- adquirir coordination locks;
- abrir execution sessions;
- ejecutar phases y execution units;
- coordinar transaction boundaries;
- delegar Schema execution;
- delegar Query execution;
- procesar backfills;
- evaluar barriers;
- evaluar preconditions/postconditions;
- respetar deadlines;
- respetar cancellation;
- aplicar retry contracts;
- registrar execution history;
- persistir applied migration records;
- clasificar failures;
- conservar outcome certainty;
- mantener checkpoints;
- liberar resources;
- generar resultados estructurados;
- emitir telemetry;
- preservar información para recovery.

---

# 6. No responsabilidades

El Executor no deberá:

```text
discover migrations
select pending migrations
rebuild dependency graph
invent planning strategy
infer new zero-downtime phases
silently alter transaction policy
silently change safety decisions
compile arbitrary SQL manually
reinterpret migration semantics
```

---

# 7. Contrato principal

Se propone:

```php
interface MigrationExecutor
{
    public function execute(
        MigrationPlan $plan,
        MigrationExecutionRequest $request
    ): MigrationExecutionResult;
}
```

---

# 8. MigrationExecutionRequest

```php
final readonly class MigrationExecutionRequest
{
    public function __construct(
        public MigrationExecutionPolicy $policy,
        public Deadline $deadline,
        public CancellationToken $cancellation,
        public MigrationExecutionTargetContext $target,
        public MigrationApprovalSet $approvals,
    ) {}
}
```

---

# 9. Plan + runtime context

Debe distinguirse:

```text
MigrationPlan
=
what should happen
```

de:

```text
MigrationExecutionRequest
=
under which live conditions it is attempted
```

---

# 10. Execution lifecycle

```text
RECEIVED
   ↓
VALIDATING
   ↓
WAITING_FOR_LOCK
   ↓
PREPARING
   ↓
RUNNING
   ↓
VERIFYING
   ↓
FINALIZING
   ↓
SUCCEEDED
```

con salidas alternativas:

```text
FAILED
PARTIALLY_APPLIED
CANCELLED
TIMED_OUT
UNKNOWN_OUTCOME
REQUIRES_RECOVERY
```

---

# 11. MigrationExecution

Estado mutable:

```php
final class MigrationExecution
{
    // runtime-only mutable state
}
```

Puede contener:

```text
execution ID
current phase
current execution unit
current operation
completed units
active transactions
lock lease
repository session
backfill checkpoints
failures
timings
cancellation state
deadline state
```

---

# 12. Execution ID

Cada ejecución tendrá:

```text
MigrationExecutionId
```

distinto de:

```text
MigrationId
MigrationPlanId
MigrationBatchId
```

---

# 13. Execution attempt identity

Una misma migration puede tener:

```text
Execution A → FAILED
Execution B → SUCCEEDED
```

pero una sola identidad de migration.

---

# 14. Execution result

```php
final readonly class MigrationExecutionResult
{
    public function __construct(
        public MigrationExecutionId $executionId,
        public MigrationExecutionStatus $status,
        public MigrationOutcomeCertainty $certainty,
        public MigrationExecutionSummary $summary,
        public ?MigrationFailureReport $failure,
        public MigrationRecoveryDescriptor $recovery,
    ) {}
}
```

---

# 15. Status vs certainty

Debe mantenerse:

```text
status
≠
outcome certainty
```

Ejemplo:

```text
status = FAILED
certainty = CERTAIN_FAILURE
```

versus:

```text
status = UNKNOWN_OUTCOME
certainty = UNKNOWN
```

---

# 16. Outcome certainty

Se reutilizará conceptualmente:

```text
CERTAIN_SUCCESS
CERTAIN_FAILURE
PARTIAL
UNKNOWN
```

---

# 17. Execution validation

Antes de cualquier efecto:

```text
MigrationPlan
     ↓
Execution Validator
```

deberá comprobar:

```text
plan completeness
plan version
plan fingerprint integrity
target identity
repository assumptions
schema assumptions
approval requirements
capabilities
runtime policy compatibility
```

---

# 18. Incomplete plan

Un plan:

```text
INCOMPLETE
```

no será ejecutable.

---

# 19. Conditionally complete plan

Puede ejecutarse solo si:

```text
runtime requirements satisfied
```

por ejemplo:

```text
approval present
maintenance window active
application barrier satisfied
```

---

# 20. Plan fingerprint verification

Si el plan fue serializado:

```text
Deserialize
   ↓
Recompute fingerprint
   ↓
Compare expected
```

Mismatch:

```text
MigrationPlanIntegrityException
```

---

# 21. Repository freshness

El plan puede declarar:

```text
ExpectedRepositoryFingerprint = R₀
```

antes de ejecución:

```text
RepositorySnapshot = R₁
```

Si:

```text
R₀ ≠ R₁
```

resultado:

```text
STALE_PLAN
```

---

# 22. Stale plan

El Executor no deberá:

```text
continue anyway
```

por defecto.

Debe:

```text
abort
```

y permitir que orchestration replanee.

---

# 23. Schema freshness

Cuando el plan dependa de:

```text
ExpectedSchemaFingerprint
```

deberá verificarse explícitamente.

---

# 24. Schema verification

Puede requerir:

```text
Schema Introspection
```

como operación runtime.

Esto es válido porque Execution sí puede hacer I/O.

---

# 25. Schema mismatch

Si:

```text
ObservedSchema ≠ PlannedAssumption
```

el plan puede ser:

```text
STALE
UNSAFE
UNKNOWN
```

según policy.

---

# 26. Migration Coordination Lock

Antes de ejecutar migrations contra un target:

```text
Acquire Migration Lock
```

cuando el plan/policy lo requiera.

---

# 27. Lock manager

```php
interface MigrationLockManager
{
    public function acquire(
        MigrationLockRequest $request
    ): MigrationLockLease;
}
```

---

# 28. Lock lease

```php
interface MigrationLockLease
{
    public function token(): MigrationLockToken;

    public function valid(): bool;

    public function renew(): void;

    public function release(): void;
}
```

según backend.

---

# 29. Fencing token

Para locks distribuidos se recomienda:

```text
FencingToken
```

monotónico.

Esto reduce riesgo de un executor antiguo recuperando actividad después de perder lease.

---

# 30. Lease loss

Si el executor pierde el lock:

```text
execution must not continue blindly
```

Debe:

```text
stop at safest boundary
mark uncertainty if necessary
trigger recovery workflow
```

---

# 31. Lock ≠ transaction

```text
Migration Coordination Lock
≠
Database Transaction
```

---

# 32. Lock acquisition timeout

Debe estar limitado por:

```text
Deadline
```

No esperar indefinidamente.

---

# 33. Lock failure

Puede producir:

```text
MigrationLockUnavailableException
MigrationLockTimeoutException
MigrationLockLostException
```

---

# 34. Execution session

Se propone:

```text
MigrationExecutionSession
```

operation-scoped.

---

# 35. Session contents

```text
execution ID
resolved target
connection leases
repository session
lock lease
transaction scopes
current checkpoint
deadline
cancellation
temporary diagnostics
```

---

# 36. Shared vs runtime state

Shared:

```text
MigrationExecutor service
frozen registries
configuration
strategies
```

Runtime:

```text
MigrationExecutionSession
```

---

# 37. Phase execution

Pipeline:

```text
MigrationPlan
  ↓
Phase 1
  ↓
Phase 2
  ↓
...
```

Cada phase deberá respetar dependencies.

---

# 38. Execution unit scheduling

Dentro de cada phase:

```text
ExecutionUnit DAG
```

puede ejecutarse secuencialmente por default.

---

# 39. Conservative default

Recomendación:

```text
migration execution = sequential by default
```

aunque el plan permita independencia.

---

# 40. Parallel execution

Solo si:

```text
plan marks independence
+
connections/resources available
+
locks compatible
+
policy allows concurrency
```

---

# 41. Parallelism ≠ reordering freedom

El Executor no puede reordenar arbitrariamente un DAG válido.

---

# 42. Preconditions

Antes de cada unit:

```text
evaluate preconditions
```

---

# 43. Precondition kinds

```text
RepositoryCondition
SchemaCondition
DataCondition
ApplicationCondition
ApprovalCondition
LockCondition
CapabilityCondition
CustomCondition
```

---

# 44. Precondition result

Se propone:

```text
SATISFIED
UNSATISFIED
UNKNOWN
ERROR
```

---

# 45. Unsatisfied precondition

Debe detener la operation.

No convertirla en:

```text
warning and continue
```

si es obligatoria.

---

# 46. Unknown precondition

Default recomendado:

```text
UNKNOWN = block
```

para cambios potencialmente destructivos.

---

# 47. Runtime validation

Ejemplo:

```text
ValidateNoNullValues(users.email)
```

podrá usar Query Engine.

---

# 48. Query execution boundary

```text
Migration Executor
   ↓
Validation Operation
   ↓
Query Engine
   ↓
Query Executor
```

No SQL manual improvisado.

---

# 49. Schema execution boundary

```text
Migration Executor
   ↓
Planned Schema Operation
   ↓
Schema Compiler
   ↓
Compiled Schema Commands
   ↓
Execution Engine
```

---

# 50. Compiler boundary

El Executor no debe generar SQL por sí mismo.

---

# 51. Compiled command lifetime

`CompiledSchemaCommandSet` puede ser immutable/cacheable.

Live statements no.

---

# 52. Data operation execution

Debe delegarse a:

```text
Query/Execution Engine
```

usando typed operations.

---

# 53. Raw operation

Si existe `RawMigrationOperation`, deberá pasar por:

```text
explicit raw execution boundary
```

con:

```text
trust policy
platform scope
timeout
logging redaction
transaction contract
```

---

# 54. Transaction management

El Executor coordina:

```text
Transaction Manager
```

según boundaries del plan.

---

# 55. Transaction lifecycle

```text
Begin
  ↓
Execute Units
  ↓
Verify Required Postconditions
  ↓
Commit
```

o:

```text
Begin
  ↓
Failure
  ↓
Rollback
```

si outcome permite.

---

# 56. Transaction ownership

```text
Migration Executor
=
orchestration owner
```

```text
Transaction Manager
=
transaction mechanism owner
```

---

# 57. Transaction boundaries immutable

El Executor no deberá fusionar o separar transaction groups arbitrariamente.

---

# 58. Transaction capability mismatch

Si runtime capability snapshot ya no coincide:

```text
abort
```

antes de ejecutar el boundary afectado.

---

# 59. Transaction begin failure

No implica que migration effects existan.

Normalmente:

```text
CERTAIN_FAILURE
```

---

# 60. Commit failure

Puede producir:

```text
UNKNOWN OUTCOME
```

según driver/platform error.

---

# 61. Commit uncertainty

Regla crítica:

```text
Commit failed
≠
Transaction rolled back
```

Puede desconocerse si commit llegó al servidor.

---

# 62. Outcome uncertainty propagation

Debe propagarse:

```text
Statement
→ Execution Unit
→ Migration
→ Repository/Recovery
```

---

# 63. Rollback attempt after uncertainty

No siempre es correcto.

Si connection se perdió tras `COMMIT`:

```text
rollback()
```

puede ser imposible o irrelevante.

---

# 64. No false rollback claim

Nunca reportar:

```text
rolled back successfully
```

sin evidencia.

---

# 65. Postconditions

Después de una operation:

```text
evaluate postconditions
```

si están definidas.

---

# 66. Postcondition failure

Si el SQL devolvió success pero postcondition falla:

```text
execution unit is not successful
```

---

# 67. Example

Operation:

```text
ADD COLUMN
```

Postcondition:

```text
column exists with expected type
```

Si introspection contradice:

```text
FAIL / UNKNOWN
```

según certainty.

---

# 68. Barriers

Un barrier puede requerir esperar por una condición externa.

---

# 69. Barrier evaluator

```php
interface MigrationBarrierEvaluator
{
    public function evaluate(
        MigrationBarrier $barrier,
        MigrationExecutionContext $context
    ): MigrationBarrierResult;
}
```

---

# 70. Barrier result

```text
SATISFIED
WAITING
UNSATISFIED
FAILED
UNKNOWN
```

---

# 71. Waiting

El Executor podrá:

```text
poll
wait
yield
```

según policy y barrier adapter.

---

# 72. Deadline interaction

Barrier waiting nunca puede superar:

```text
Execution Deadline
```

---

# 73. Manual approval barrier

El core no presenta UI.

Recibe:

```text
MigrationApprovalSet
```

o consulta un approval provider explícito.

---

# 74. External application barrier

Puede delegar a:

```text
ApplicationDeploymentBarrierProvider
```

sin acoplar el core a deployment tooling.

---

# 75. Backfill execution

Backfill tendrá runtime especializado.

---

# 76. Backfill executor

```php
interface MigrationBackfillExecutor
{
    public function execute(
        MigrationBackfillPlan $plan,
        MigrationBackfillExecutionContext $context
    ): MigrationBackfillResult;
}
```

---

# 77. Backfill chunk loop

Conceptualmente:

```text
Resolve next chunk
      ↓
Begin chunk transaction?
      ↓
Execute mutation
      ↓
Verify
      ↓
Checkpoint
      ↓
Continue
```

---

# 78. Chunking key

Debe ser estable.

Ejemplos:

```text
primary key
monotonic ID
explicit stable cursor
```

---

# 79. Offset pagination

No recomendar para mutable large backfills:

```text
OFFSET 1000000
```

por inconsistencia/performance.

---

# 80. Cursor-based backfill

Preferir:

```text
WHERE id > :last_id
ORDER BY id
LIMIT :chunk
```

cuando semánticamente válido.

---

# 81. Backfill progress

Debe persistirse cuando el plan lo requiera.

---

# 82. Progress store

Puede utilizar:

```text
Migration Execution Repository
```

o un:

```text
BackfillProgressStore
```

especializado.

---

# 83. Checkpoint

```text
BackfillCheckpoint
```

debe ser:

```text
typed
versioned
migration-scoped
operation-scoped
target-scoped
```

---

# 84. Checkpoint after commit

Regla:

```text
checkpoint advancement
```

deberá ocurrir solo cuando el chunk effect contract se considere completado.

---

# 85. Crash before checkpoint

Caso:

```text
chunk applied
crash
checkpoint not saved
```

resume puede repetir el chunk.

Por tanto el chunk debe ser:

```text
idempotent
```

o tener reconciliation.

---

# 86. Backfill retry

No se basará únicamente en driver error.

Debe considerar:

```text
chunk idempotency
checkpoint state
transaction certainty
mutation semantics
```

---

# 87. Dynamic throttling

Podrá existir:

```text
BackfillThrottlePolicy
```

para reducir presión operacional.

---

# 88. Throttling signals

Posibles:

```text
fixed delay
load signal
replica lag
query latency
external capacity provider
```

mediante adapters.

---

# 89. Planner vs executor throttling

Planner define:

```text
throttling requirement/policy
```

Executor aplica runtime behavior.

---

# 90. Retry execution

Cada `ExecutionUnit` tiene:

```text
MigrationRetryContract
```

---

# 91. Retry loop

```text
Attempt
  ↓
Failure classification
  ↓
Outcome certainty
  ↓
Retry contract
  ↓
Budget/deadline
  ↓
Retry? yes/no
```

---

# 92. Retry policy inputs

Debe considerar:

```text
failure kind
native error code
SQLSTATE
outcome certainty
idempotency
transaction state
checkpoint state
remaining deadline
attempt budget
```

---

# 93. Retry after partial delivery

Para streaming/observable data operation:

```text
partial externally visible effects
```

puede hacer retry inseguro.

---

# 94. Migration-level blind retry prohibited

Nunca:

```php
try {
    runMigration();
} catch (\Throwable) {
    runMigration();
}
```

---

# 95. Retry budget

Se propone:

```text
max attempts
max total retry duration
max cumulative backoff
```

---

# 96. Backoff

Puede utilizar:

```text
fixed
linear
exponential
jittered
```

pero no sobrepasar deadline.

---

# 97. Retry and migration lock

Durante backoff se deberá decidir explícitamente si:

```text
lock retained
```

o:

```text
lock released
```

Nunca accidentalmente.

---

# 98. Lock retention default

Para migration coordination lock:

```text
retain during short retries
```

puede ser preferible, sujeto a lease.

---

# 99. Cancellation

Toda execution deberá respetar:

```text
CancellationToken
```

---

# 100. Cancellation sources

```text
CLI signal
process shutdown
user request
deployment orchestration
deadline expiration
lost lock
system shutdown
```

---

# 101. Cancellation propagation

```text
Migration Executor
→ Execution Unit
→ Schema/Query Executor
→ Driver
```

cuando las capas lo soporten.

---

# 102. Cooperative cancellation

No todos los drivers pueden abortar inmediatamente.

Por tanto:

```text
Cancellation Requested
≠
Execution Stopped
```

---

# 103. Cancellation checkpoints

Entre operations/chunks:

```text
check cancellation
```

siempre.

---

# 104. Cancellation during non-cancellable operation

Debe esperar outcome o marcar incertidumbre si la conexión se rompe.

---

# 105. Deadline

Se utiliza deadline absoluto/monotonic-aware.

No reiniciar timeout por cada nested operation si el plan posee deadline global.

---

# 106. Time budget

Conceptualmente:

```text
RemainingTime =
ExecutionDeadline - MonotonicNow
```

---

# 107. Unit timeout

Puede existir límite menor:

```text
min(unitTimeout, remainingGlobalDeadline)
```

---

# 108. Timeout ≠ cancellation

Timeout es una causa de cancelación/límite.

Pero outcome puede quedar unknown.

---

# 109. Failure model

Se propone:

```php
final readonly class MigrationFailureReport
{
    public function __construct(
        public MigrationFailureKind $kind,
        public MigrationFailureScope $scope,
        public MigrationOutcomeCertainty $certainty,
        public ?MigrationIdentity $migration,
        public ?MigrationExecutionUnitId $unit,
        public ?MigrationPlanOperationId $operation,
        public Throwable $cause,
        public MigrationFailureMetadata $metadata,
    ) {}
}
```

---

# 110. Failure scopes

```text
PLAN_VALIDATION
LOCK
PRECONDITION
TRANSACTION
SCHEMA
DATA
BACKFILL
BARRIER
VALIDATION
POSTCONDITION
REPOSITORY
CLEANUP
UNKNOWN
```

---

# 111. Failure kinds

```text
CONFLICT
TIMEOUT
CANCELLED
CAPABILITY_CHANGED
DEADLOCK
CONNECTION_FAILURE
CONSTRAINT_FAILURE
INVALID_DATA
LOCK_LOST
REPOSITORY_FAILURE
OUTCOME_UNKNOWN
EXTENSION_FAILURE
INVARIANT_FAILURE
```

---

# 112. Failure ≠ exception hierarchy only

El exception puede ser causa.

El `FailureReport` conserva semántica estructurada para:

```text
retry
recovery
telemetry
repository
CLI
```

---

# 113. Native error preservation

Cuando exista:

```text
SQLSTATE
vendor code
native message
```

debe conservarse con redaction adecuada.

---

# 114. Partial execution

Se produce cuando:

```text
some committed effects exist
+
migration contract not completed
```

---

# 115. Example

```text
Add column ✓ committed
Backfill ✓ partially completed
Constraint validation ✗
```

Resultado:

```text
PARTIALLY_APPLIED
```

---

# 116. Partial ≠ unknown

```text
PARTIAL
```

significa que se conocen algunos efectos.

```text
UNKNOWN
```

significa que no puede determinarse suficientemente.

---

# 117. Recovery descriptor

Cada failure importante deberá poder producir:

```text
MigrationRecoveryDescriptor
```

---

# 118. Recovery descriptor contents

```text
completed phases
completed units
last known checkpoint
active/uncertain transaction
repository state
schema verification requirements
safe retry status
recommended recovery mode
```

---

# 119. Recovery descriptor ≠ automatic recovery

El documento futuro de recovery podrá consumirlo.

---

# 120. Repository execution history

Al empezar:

```text
record execution started
```

si configuration lo exige.

---

# 121. Applied record timing

El current applied record deberá escribirse solo cuando:

```text
required operations successful
+
required postconditions successful
+
execution contract satisfied
```

---

# 122. Repository write in same transaction

Cuando sea posible:

```text
BEGIN
migration effects
repository applied record
COMMIT
```

ideal.

---

# 123. Repository write after non-transactional DDL

Puede existir ventana:

```text
effect succeeded
↓
repository write pending
```

El sistema debe registrarla mediante execution history/checkpoint.

---

# 124. Repository write failure

Si migration effects succeeded pero repository write fails:

```text
migration cannot safely be reported simply FAILED
```

Puede ser:

```text
EFFECTS_APPLIED_REPOSITORY_UNRECORDED
```

y requerir recovery.

---

# 125. Applied truth invariant

El sistema buscará preservar:

```text
Repository APPLIED
⇒
Execution Contract Satisfied
```

aunque no siempre pueda preservar el inverso.

---

# 126. Execution history state transitions

```text
CREATED
 ↓
RUNNING
 ├── SUCCEEDED
 ├── FAILED
 ├── PARTIAL
 ├── CANCELLED
 └── UNKNOWN
```

---

# 127. Finalization

Después del último operation:

```text
evaluate final postconditions
↓
repository applied write
↓
execution history success
↓
cleanup
```

---

# 128. Cleanup

Debe ejecutarse incluso tras failure cuando sea seguro.

---

# 129. Cleanup responsibilities

```text
release cursors
release statements
rollback active known-open transactions
release connection leases
release migration locks
flush telemetry
dispose temp resources
```

---

# 130. Cleanup failure

Debe conservarse como:

```text
secondary failure
```

sin sobrescribir el primary failure.

---

# 131. Example

Primary:

```text
constraint creation failed
```

Cleanup:

```text
lock release failed
```

El result deberá conservar ambos.

---

# 132. Suppressed/secondary failures

Se propone:

```text
MigrationSecondaryFailureSet
```

---

# 133. Resource ownership

Cada live resource tendrá owner explícito:

```text
ExecutionSession
TransactionScope
BackfillSession
QueryExecution
```

---

# 134. No leaked resources

Después de execution:

```text
no live Statement
no live Cursor
no live ConnectionLease
no live Transaction
no live MigrationLockLease
```

deberá permanecer en shared state.

---

# 135. Persistent runtimes

Crítico para FrankenPHP:

```text
worker survives migration command/request
```

pero:

```text
execution state must not
```

---

# 136. FrankenPHP model

```text
Worker
├── shared immutable services
└── MigrationExecutionSession
      ├── lock
      ├── connection
      ├── transactions
      └── state
```

Todo el session state se descarta al finalizar.

---

# 137. RoadRunner

Mismo principio:

```text
job execution state
```

no sobrevive al job.

---

# 138. OpenSwoole

Cada coroutine deberá aislar:

```text
execution session
transaction
connection lease
repository session
lock state
```

---

# 139. No static current migration

Prohibido:

```php
MigrationExecutor::$currentMigration
```

---

# 140. No static current connection

Prohibido:

```php
MigrationExecutor::$connection
```

---

# 141. Runtime Context

Se propone:

```php
final readonly class MigrationRuntimeContext
{
    public function __construct(
        public MigrationExecutionId $execution,
        public MigrationTargetRuntime $target,
        public Deadline $deadline,
        public CancellationToken $cancellation,
        public MigrationExecutionPolicy $policy,
    ) {}
}
```

---

# 142. Target resolution

`MigrationTarget` lógico:

```text
billing
```

se resuelve a:

```text
ConnectionLease
PlatformContext
RepositoryTarget
```

durante runtime.

---

# 143. Target resolver

```php
interface MigrationExecutionTargetResolver
{
    public function resolve(
        MigrationTarget $target
    ): MigrationExecutionTarget;
}
```

---

# 144. Target mismatch

Si runtime target no corresponde al plan:

```text
MigrationTargetMismatchException
```

---

# 145. Capability snapshot verification

Plan generado con:

```text
CapabilityFingerprint C₀
```

runtime obtiene:

```text
C₁
```

si incompatibles:

```text
abort / replan
```

---

# 146. Version ≠ capability

Nunca asumir:

```text
same DB version = same effective capabilities
```

si configuration/extensions influyen.

---

# 147. Migration-level result

Una batch puede contener múltiples migrations.

Debe existir:

```text
MigrationExecutionResult
```

por migration y:

```text
MigrationBatchExecutionResult
```

superior.

---

# 148. Batch execution

Aunque el doc 108 profundizará:

```text
Batch
├── Migration A
├── Migration B
└── Migration C
```

cada migration mantiene result independiente.

---

# 149. Stop-on-failure default

Default recomendado:

```text
stop batch after first blocking migration failure
```

---

# 150. Continue-on-failure

Solo para scenarios explícitos donde migrations sean independientes y policy lo permita.

---

# 151. Dependency failure

Si B depende de A y A falla:

```text
B = BLOCKED_BY_DEPENDENCY
```

no `FAILED`.

---

# 152. Migration execution graph

El runtime puede mantener:

```text
MigrationExecutionGraphState
```

con estados:

```text
READY
RUNNING
SUCCEEDED
FAILED
BLOCKED
SKIPPED
UNKNOWN
```

---

# 153. Skipped ≠ applied

Una migration omitida por selection/policy no se considera aplicada.

---

# 154. Preview mode

Execution System no debería realizar efectos en:

```text
PREVIEW
```

Ese modo corresponde principalmente a planning/compilation tooling.

---

# 155. Validate-only execution

Puede existir:

```text
VALIDATE_ONLY
```

para ejecutar preflight runtime checks sin mutar.

---

# 156. Validation side effects

Debe garantizarse que validate-only no ejecute operaciones mutables.

---

# 157. Dry-run ambiguity

No llamar `dry-run` a algo que ejecuta queries mutables.

---

# 158. Events

Puede emitir:

```text
MigrationExecutionStarting
MigrationLockAcquiring
MigrationLockAcquired
MigrationPhaseStarting
MigrationUnitStarting
MigrationOperationStarting
MigrationOperationCompleted
MigrationOperationFailed
MigrationBackfillProgressed
MigrationPhaseCompleted
MigrationExecutionCompleted
MigrationExecutionFailed
MigrationRecoveryRequired
```

---

# 159. Event listeners

No deberán poder alterar hidden execution strategy.

---

# 160. Controlled hooks

Si se permiten hooks:

```text
before phase
after phase
```

deberán ser:

```text
typed
bounded
observable
explicitly registered
```

---

# 161. Hook failure

Se clasifica como:

```text
EXTENSION/HOOК failure
```

no se oculta.

---

# 162. Telemetry

Debe generar:

```text
execution duration
lock wait
phase duration
unit duration
operation duration
backfill rows
retry count
failure kind
outcome certainty
cleanup duration
```

---

# 163. Sensitive telemetry

Nunca incluir por default:

```text
credentials
secret values
full row data
full raw SQL
```

---

# 164. SQL telemetry

Puede registrar:

```text
query fingerprint
operation ID
compiler source map
```

en vez del SQL completo.

---

# 165. Source mapping

Failure deberá poder mapear:

```text
Driver Error
→ Compiled Command
→ Planned Schema Operation
→ MigrationPlanOperation
→ MigrationDefinition Operation
→ Migration Source
```

---

# 166. Developer diagnostics

Ejemplo:

```text
Migration failed.

Migration:
app:2026_09_06_add_email_constraint

Phase:
VALIDATE

Operation:
validate_no_null_users_email

Failure:
12 rows still contain NULL values.

No schema change was applied after this validation failure.
```

---

# 167. Unknown outcome diagnostic

```text
Migration outcome is uncertain.

Migration:
app:M42

Operation:
create_index_users_email

Cause:
Connection lost while waiting for server response.

Known:
Request was sent.

Unknown:
Whether the server committed the operation.

Action:
Run recovery/reconciliation before retrying.
```

---

# 168. Retry diagnostic

```text
Retry suppressed.

Reason:
Previous attempt outcome is UNKNOWN.

Operation:
ALTER TABLE users ...

Idempotency:
UNKNOWN

Safe retry:
NOT PROVEN
```

---

# 169. Security model

Execution is privileged.

Debe requerir:

```text
trusted plan
authorized caller
validated target
validated approvals
approved raw operation policy
```

---

# 170. Authorization boundary

El Executor puede recibir:

```text
ExecutionAuthorizationContext
```

pero no implementar todo Authorization subsystem internamente.

---

# 171. Plan tampering

Serialized plan modificado:

```text
fingerprint mismatch
```

debe fallar.

---

# 172. Raw SQL security

Raw operations requieren:

```text
explicit trusted marker
platform scope
redaction policy
```

---

# 173. Secrets from connections

Nunca deben incluirse en:

```text
result
diagnostic
telemetry
history metadata
```

---

# 174. Execution budgets

Se propone:

```text
max migrations
max phases
max units
max operations
max runtime
max retries
max backfill chunks
max affected rows?
max lock wait
```

según policy.

---

# 175. Budget exhaustion

Debe producir:

```text
MigrationExecutionBudgetExceededException
```

y outcome appropriate.

---

# 176. Affected-row budget

Para destructive data migration:

```text
DELETE
```

puede existir límite:

```text
maxAffectedRows
```

si semántica lo permite.

---

# 177. Row count uncertainty

Si driver no puede reportar reliable affected rows:

```text
UNKNOWN
```

no inventar exactitud.

---

# 178. Rate limiting

Backfills podrán respetar:

```text
operations per second
rows per second
I/O budget
```

mediante Resource Governance integration.

---

# 179. Extension model

Se propone:

```text
MigrationExecutionExtension
```

para:

```text
custom barrier evaluator
custom operation executor
custom checkpoint store
custom recovery evidence provider
```

---

# 180. Frozen registry

Execution extension registry:

```text
freeze after bootstrap
```

---

# 181. Extension constraints

No podrá:

- bypass required preconditions;
- forge success;
- mutate plan;
- hide failures;
- write applied repository record outside protocol;
- disable outcome certainty rules;
- leak live resources.

---

# 182. Custom operation executor

```php
interface MigrationCustomOperationExecutor
{
    public function supports(
        MigrationPlanOperation $operation
    ): bool;

    public function execute(
        MigrationPlanOperation $operation,
        MigrationOperationExecutionContext $context
    ): MigrationOperationResult;
}
```

---

# 183. Handler ambiguity

Dos handlers válidos para misma operation sin resolver:

```text
ExecutionExtensionConflictException
```

---

# 184. Testing strategy

Debe incluir:

```text
plan validation
stale repository
stale schema
lock acquisition
lock loss
transaction success
transaction rollback
commit uncertainty
precondition failure
postcondition failure
schema execution
data execution
backfills
checkpoint recovery
barriers
retry
cancellation
deadline
partial execution
unknown outcome
repository write failure
cleanup failure
persistent runtime isolation
extension conflicts
```

---

# 185. Deterministic execution tests

Aunque runtime tiene timing/externals, para mismas mocked outcomes:

```text
same state transition sequence
```

deberá producirse.

---

# 186. Fault injection

Se recomienda un:

```text
MigrationExecutionFaultInjector
```

solo para Testing namespace.

---

# 187. Fault points

Ejemplos:

```text
after lock acquisition
before transaction begin
after statement send
before commit response
after DDL success
before repository write
during checkpoint
during lock release
```

---

# 188. Crash simulation

Es vital probar:

```text
DDL success
↓
process crash
↓
repository record absent
```

---

# 189. Unknown commit test

Simular:

```text
COMMIT sent
connection lost
```

Resultado:

```text
UNKNOWN_OUTCOME
```

no `FAILED_ROLLED_BACK`.

---

# 190. Cancellation test

Cancelar durante backfill:

```text
stop at safe checkpoint
```

cuando sea posible.

---

# 191. Lock loss test

Si lease expira:

```text
execution stops before next unsafe unit
```

---

# 192. Cleanup test

Tras failure:

```text
connections released
transactions closed when determinable
lock release attempted
temporary resources cleared
```

---

# 193. Conformance suite

Custom executors/backends deberán pasar:

```text
MigrationExecutionConformanceSuite
```

---

# 194. Error hierarchy

Se propone:

```text
DatabaseMigrationExecutionException
├── MigrationExecutionValidationException
├── MigrationPlanIntegrityException
├── MigrationPlanStaleException
│   ├── MigrationRepositoryPlanStaleException
│   └── MigrationSchemaPlanStaleException
├── MigrationTargetResolutionException
├── MigrationTargetMismatchException
├── MigrationCapabilityChangedException
├── MigrationLockException
│   ├── MigrationLockUnavailableException
│   ├── MigrationLockTimeoutException
│   └── MigrationLockLostException
├── MigrationPreconditionException
├── MigrationTransactionExecutionException
├── MigrationSchemaExecutionException
├── MigrationDataExecutionException
├── MigrationBackfillExecutionException
├── MigrationBarrierExecutionException
├── MigrationValidationExecutionException
├── MigrationPostconditionException
├── MigrationRetryException
├── MigrationCancellationException
├── MigrationExecutionTimeoutException
├── MigrationPartialExecutionException
├── MigrationOutcomeUnknownException
├── MigrationRepositoryRecordingException
├── MigrationCleanupException
├── MigrationExecutionExtensionException
├── MigrationExecutionBudgetExceededException
└── MigrationExecutionInvariantException
```

---

# 195. Namespace

```text
VoltStack\Quantum\Database\Migration\Execution
```

---

# 196. Estructura propuesta

```text
Migration/
└── Execution/
    ├── Contract/
    │   ├── MigrationExecutor.php
    │   ├── MigrationExecutionValidator.php
    │   ├── MigrationExecutionTargetResolver.php
    │   └── MigrationOperationExecutor.php
    │
    ├── Core/
    │   ├── DefaultMigrationExecutor.php
    │   ├── MigrationExecutionRequest.php
    │   ├── MigrationExecutionSession.php
    │   ├── MigrationRuntimeContext.php
    │   └── MigrationExecutionPolicy.php
    │
    ├── Identity/
    │   └── MigrationExecutionId.php
    │
    ├── State/
    │   ├── MigrationExecutionStatus.php
    │   ├── MigrationExecutionStateMachine.php
    │   ├── MigrationOutcomeCertainty.php
    │   └── MigrationExecutionGraphState.php
    │
    ├── Result/
    │   ├── MigrationExecutionResult.php
    │   ├── MigrationExecutionSummary.php
    │   ├── MigrationOperationResult.php
    │   └── MigrationBatchExecutionResult.php
    │
    ├── Validation/
    │   ├── DefaultMigrationExecutionValidator.php
    │   ├── MigrationPlanFreshnessValidator.php
    │   ├── MigrationRepositoryFreshnessValidator.php
    │   └── MigrationSchemaFreshnessValidator.php
    │
    ├── Target/
    │   ├── MigrationExecutionTarget.php
    │   ├── MigrationExecutionTargetContext.php
    │   └── MigrationTargetRuntime.php
    │
    ├── Lock/
    │   ├── MigrationLockManager.php
    │   ├── MigrationLockRequest.php
    │   ├── MigrationLockLease.php
    │   ├── MigrationLockToken.php
    │   └── MigrationFencingToken.php
    │
    ├── Phase/
    │   ├── MigrationPhaseExecutor.php
    │   └── MigrationExecutionUnitScheduler.php
    │
    ├── Unit/
    │   ├── MigrationExecutionUnitExecutor.php
    │   ├── MigrationExecutionUnitContext.php
    │   └── MigrationExecutionUnitResult.php
    │
    ├── Condition/
    │   ├── MigrationConditionEvaluator.php
    │   ├── MigrationPreconditionEvaluator.php
    │   ├── MigrationPostconditionEvaluator.php
    │   └── MigrationConditionResult.php
    │
    ├── Schema/
    │   └── MigrationSchemaOperationExecutor.php
    │
    ├── Data/
    │   └── MigrationDataOperationExecutor.php
    │
    ├── Backfill/
    │   ├── MigrationBackfillExecutor.php
    │   ├── MigrationBackfillExecutionContext.php
    │   ├── MigrationBackfillResult.php
    │   ├── BackfillCheckpoint.php
    │   ├── BackfillProgressStore.php
    │   └── BackfillThrottlePolicy.php
    │
    ├── Barrier/
    │   ├── MigrationBarrierEvaluator.php
    │   ├── MigrationBarrierResult.php
    │   └── MigrationBarrierWaitPolicy.php
    │
    ├── Retry/
    │   ├── MigrationRetryExecutor.php
    │   ├── MigrationRetryDecision.php
    │   ├── MigrationRetryBudget.php
    │   └── MigrationRetryBackoff.php
    │
    ├── Transaction/
    │   ├── MigrationTransactionCoordinator.php
    │   ├── MigrationTransactionScope.php
    │   └── MigrationTransactionResult.php
    │
    ├── Cancellation/
    │   └── MigrationCancellationController.php
    │
    ├── Deadline/
    │   └── MigrationExecutionDeadline.php
    │
    ├── Failure/
    │   ├── MigrationFailureReport.php
    │   ├── MigrationFailureKind.php
    │   ├── MigrationFailureScope.php
    │   ├── MigrationFailureMetadata.php
    │   └── MigrationSecondaryFailureSet.php
    │
    ├── Recovery/
    │   ├── MigrationRecoveryDescriptor.php
    │   └── MigrationRecoveryEvidence.php
    │
    ├── Repository/
    │   └── MigrationExecutionRepositoryCoordinator.php
    │
    ├── Resource/
    │   ├── MigrationExecutionResourceOwner.php
    │   └── MigrationExecutionCleanupManager.php
    │
    ├── Budget/
    │   └── MigrationExecutionBudget.php
    │
    ├── Extension/
    │   ├── MigrationExecutionExtension.php
    │   ├── MigrationCustomOperationExecutor.php
    │   └── MigrationExecutionExtensionRegistry.php
    │
    ├── Telemetry/
    │   └── MigrationExecutionTelemetry.php
    │
    └── Exception/
        └── ...
```

---

# 197. Invariantes

## DB-MIGRATION-EXEC-001
Execution será distinto de Planning.

## DB-MIGRATION-EXEC-002
Execution consumirá un MigrationPlan immutable.

## DB-MIGRATION-EXEC-003
Executor no modificará el plan.

## DB-MIGRATION-EXEC-004
Executor no rediseñará migration strategy silenciosamente.

## DB-MIGRATION-EXEC-005
Executor no descubrirá migrations.

## DB-MIGRATION-EXEC-006
Executor no calculará pending migrations.

## DB-MIGRATION-EXEC-007
Executor no resolverá migration dependency graph desde cero.

## DB-MIGRATION-EXEC-008
Execution state será mutable y operation-scoped.

## DB-MIGRATION-EXEC-009
Execution state será distinto de repository state.

## DB-MIGRATION-EXEC-010
Execution ID será distinto de Migration ID.

## DB-MIGRATION-EXEC-011
Execution ID será distinto de Plan ID.

## DB-MIGRATION-EXEC-012
Execution status será distinto de outcome certainty.

## DB-MIGRATION-EXEC-013
Plan se validará antes de efectos.

## DB-MIGRATION-EXEC-014
Incomplete plan no será ejecutable.

## DB-MIGRATION-EXEC-015
Conditional requirements deberán satisfacerse antes de operations dependientes.

## DB-MIGRATION-EXEC-016
Plan fingerprint será verificable.

## DB-MIGRATION-EXEC-017
Stale repository state bloqueará execution por default.

## DB-MIGRATION-EXEC-018
Stale schema assumptions podrán bloquear execution.

## DB-MIGRATION-EXEC-019
Executor no ignorará plan freshness.

## DB-MIGRATION-EXEC-020
Migration coordination lock será explícito cuando se requiera.

## DB-MIGRATION-EXEC-021
Lock será distinto de transaction.

## DB-MIGRATION-EXEC-022
Lock acquisition respetará deadline.

## DB-MIGRATION-EXEC-023
Lost lock no permitirá continuar ciegamente.

## DB-MIGRATION-EXEC-024
Distributed lock podrá usar fencing token.

## DB-MIGRATION-EXEC-025
ExecutionSession será operation-scoped.

## DB-MIGRATION-EXEC-026
No habrá global mutable current migration.

## DB-MIGRATION-EXEC-027
No habrá global mutable current connection.

## DB-MIGRATION-EXEC-028
Phase order respetará dependencies.

## DB-MIGRATION-EXEC-029
Parallel execution será opt-in/capability-driven.

## DB-MIGRATION-EXEC-030
Parallel execution no violará graph dependencies.

## DB-MIGRATION-EXEC-031
Preconditions serán evaluadas antes de unit execution.

## DB-MIGRATION-EXEC-032
Unsatisfied mandatory precondition bloqueará operation.

## DB-MIGRATION-EXEC-033
Unknown critical precondition no se considerará satisfied.

## DB-MIGRATION-EXEC-034
Runtime validation usará Query Engine cuando corresponda.

## DB-MIGRATION-EXEC-035
Schema operations serán delegadas al Schema subsystem.

## DB-MIGRATION-EXEC-036
Executor no generará SQL directamente.

## DB-MIGRATION-EXEC-037
Data operations serán delegadas al Query/Execution Engine.

## DB-MIGRATION-EXEC-038
Raw operations serán explicit trust boundaries.

## DB-MIGRATION-EXEC-039
Transaction Manager será dueño del mecanismo transaccional.

## DB-MIGRATION-EXEC-040
Executor será dueño de la coordinación de transaction boundaries.

## DB-MIGRATION-EXEC-041
Executor no alterará boundaries arbitrariamente.

## DB-MIGRATION-EXEC-042
Commit failure podrá producir UNKNOWN outcome.

## DB-MIGRATION-EXEC-043
Commit failure no implicará rollback.

## DB-MIGRATION-EXEC-044
Outcome uncertainty se propagará.

## DB-MIGRATION-EXEC-045
Rollback success no se inventará.

## DB-MIGRATION-EXEC-046
Postconditions serán evaluadas cuando formen parte del plan.

## DB-MIGRATION-EXEC-047
Successful command + failed postcondition no será overall success.

## DB-MIGRATION-EXEC-048
Barrier será evaluado por adapter explícito.

## DB-MIGRATION-EXEC-049
Barrier wait respetará deadline.

## DB-MIGRATION-EXEC-050
Manual approval barrier no será implementado como prompt dentro del core.

## DB-MIGRATION-EXEC-051
Backfill será ejecutado por subsystem especializado.

## DB-MIGRATION-EXEC-052
Backfill chunking usará cursor estable cuando se requiera.

## DB-MIGRATION-EXEC-053
Checkpoint será scoped por migration/operation/target.

## DB-MIGRATION-EXEC-054
Checkpoint avanzará solo tras efecto aceptado.

## DB-MIGRATION-EXEC-055
Crash-before-checkpoint será considerado en retry/resume safety.

## DB-MIGRATION-EXEC-056
Backfill retry requerirá idempotency/reconciliation reasoning.

## DB-MIGRATION-EXEC-057
Runtime throttling será distinto de planning.

## DB-MIGRATION-EXEC-058
Retry seguirá contract del plan.

## DB-MIGRATION-EXEC-059
Retryable failure será distinto de safe retry.

## DB-MIGRATION-EXEC-060
Blind migration-level retry estará prohibido.

## DB-MIGRATION-EXEC-061
Retry considerará outcome certainty.

## DB-MIGRATION-EXEC-062
Retry considerará transaction state.

## DB-MIGRATION-EXEC-063
Retry considerará remaining deadline.

## DB-MIGRATION-EXEC-064
Retry tendrá budget.

## DB-MIGRATION-EXEC-065
Backoff no excederá global deadline.

## DB-MIGRATION-EXEC-066
Lock behavior durante retry será explícito.

## DB-MIGRATION-EXEC-067
Cancellation será first-class.

## DB-MIGRATION-EXEC-068
Cancellation requested será distinto de operation stopped.

## DB-MIGRATION-EXEC-069
Cancellation se propagará cuando sea soportada.

## DB-MIGRATION-EXEC-070
Cancellation será distinta de rollback.

## DB-MIGRATION-EXEC-071
Deadline será globalmente propagable.

## DB-MIGRATION-EXEC-072
Nested timeout no reiniciará deadline arbitrariamente.

## DB-MIGRATION-EXEC-073
Timeout podrá producir unknown outcome.

## DB-MIGRATION-EXEC-074
Failures serán structured.

## DB-MIGRATION-EXEC-075
Failure report conservará scope.

## DB-MIGRATION-EXEC-076
Failure report conservará certainty.

## DB-MIGRATION-EXEC-077
Native DB error data se conservará con redaction.

## DB-MIGRATION-EXEC-078
Partial execution será first-class.

## DB-MIGRATION-EXEC-079
Partial será distinto de Unknown.

## DB-MIGRATION-EXEC-080
Unknown outcome será first-class.

## DB-MIGRATION-EXEC-081
Recovery descriptor será generado cuando sea útil.

## DB-MIGRATION-EXEC-082
Recovery descriptor no ejecutará recovery automáticamente.

## DB-MIGRATION-EXEC-083
Execution history podrá registrar start antes de efectos.

## DB-MIGRATION-EXEC-084
Applied record se escribirá solo tras completed contract.

## DB-MIGRATION-EXEC-085
Repository writes compartirán transaction cuando sea posible.

## DB-MIGRATION-EXEC-086
Non-transactional write gap será reconocido.

## DB-MIGRATION-EXEC-087
Effects succeeded + repository write failed será recovery-relevant state.

## DB-MIGRATION-EXEC-088
Repository APPLIED implicará completed contract.

## DB-MIGRATION-EXEC-089
Finalization incluirá required postconditions.

## DB-MIGRATION-EXEC-090
Cleanup se intentará en success/failure paths.

## DB-MIGRATION-EXEC-091
Cleanup failure no sobrescribirá primary failure.

## DB-MIGRATION-EXEC-092
Secondary failures serán preservables.

## DB-MIGRATION-EXEC-093
Live resources tendrán owner explícito.

## DB-MIGRATION-EXEC-094
No live statements quedarán en shared state.

## DB-MIGRATION-EXEC-095
No live cursors quedarán en shared state.

## DB-MIGRATION-EXEC-096
No live connection leases quedarán en shared state.

## DB-MIGRATION-EXEC-097
No live transactions quedarán en shared state.

## DB-MIGRATION-EXEC-098
No live lock leases quedarán en shared state.

## DB-MIGRATION-EXEC-099
FrankenPHP executions estarán aisladas.

## DB-MIGRATION-EXEC-100
RoadRunner executions estarán aisladas.

## DB-MIGRATION-EXEC-101
OpenSwoole coroutine executions estarán aisladas.

## DB-MIGRATION-EXEC-102
Runtime target resolution será explícita.

## DB-MIGRATION-EXEC-103
Target mismatch será error.

## DB-MIGRATION-EXEC-104
Capability snapshot mismatch será detectado.

## DB-MIGRATION-EXEC-105
Version será distinta de capability.

## DB-MIGRATION-EXEC-106
Cada migration tendrá result independiente dentro de batch.

## DB-MIGRATION-EXEC-107
Stop-on-failure será default conservador.

## DB-MIGRATION-EXEC-108
Dependent migration tras parent failure será BLOCKED, no FAILED.

## DB-MIGRATION-EXEC-109
Skipped será distinto de Applied.

## DB-MIGRATION-EXEC-110
Validate-only no ejecutará mutaciones.

## DB-MIGRATION-EXEC-111
Events no modificarán hidden execution strategy.

## DB-MIGRATION-EXEC-112
Hooks serán typed y explicit.

## DB-MIGRATION-EXEC-113
Hook failure no será ocultado.

## DB-MIGRATION-EXEC-114
Telemetry será structured.

## DB-MIGRATION-EXEC-115
Telemetry no expondrá secrets por default.

## DB-MIGRATION-EXEC-116
Source mapping permitirá tracing de driver failure a migration source.

## DB-MIGRATION-EXEC-117
Plan authorization será separada de execution mechanics.

## DB-MIGRATION-EXEC-118
Plan tampering será detectable.

## DB-MIGRATION-EXEC-119
Raw operations requerirán trust policy.

## DB-MIGRATION-EXEC-120
Execution budgets serán explicitables.

## DB-MIGRATION-EXEC-121
Budget exhaustion no fingirá success.

## DB-MIGRATION-EXEC-122
Affected-row limits serán conservadores.

## DB-MIGRATION-EXEC-123
Unknown affected-row count no se inventará.

## DB-MIGRATION-EXEC-124
Backfill rate limiting podrá integrarse con resource governance.

## DB-MIGRATION-EXEC-125
Execution extension registry será frozen.

## DB-MIGRATION-EXEC-126
Extensions no podrán mutar plan.

## DB-MIGRATION-EXEC-127
Extensions no podrán forge success.

## DB-MIGRATION-EXEC-128
Extensions no podrán bypass repository protocol.

## DB-MIGRATION-EXEC-129
Extension handler ambiguity será error.

## DB-MIGRATION-EXEC-130
Execution subsystem tendrá fault-injection testing.

## DB-MIGRATION-EXEC-131
Commit uncertainty será testeada.

## DB-MIGRATION-EXEC-132
Lock loss será testeado.

## DB-MIGRATION-EXEC-133
Crash window será testeada.

## DB-MIGRATION-EXEC-134
Cleanup behavior será testeado.

## DB-MIGRATION-EXEC-135
Persistent runtime isolation será testeado.

## DB-MIGRATION-EXEC-136
Executor no asumirá que FAILED significa effects absent.

## DB-MIGRATION-EXEC-137
Executor no asumirá que CANCELLED significa rollback.

## DB-MIGRATION-EXEC-138
Executor no asumirá que connection loss significa transaction rollback.

## DB-MIGRATION-EXEC-139
Executor no reintentará operation con UNKNOWN outcome sin proof.

## DB-MIGRATION-EXEC-140
Repository recording será parte explícita del execution protocol.

## DB-MIGRATION-EXEC-141
Executor no actualizará applied checksum fuera del accepted application record.

## DB-MIGRATION-EXEC-142
Backfill checkpoints no sustituirán repository applied state.

## DB-MIGRATION-EXEC-143
Migration result será immutable.

## DB-MIGRATION-EXEC-144
Runtime mutable state no se serializará como plan.

## DB-MIGRATION-EXEC-145
Execution errors serán typed.

## DB-MIGRATION-EXEC-146
Partial failure será distinguible de repository failure.

## DB-MIGRATION-EXEC-147
Repository failure será distinguible de schema/data failure.

## DB-MIGRATION-EXEC-148
Recovery-required state será explicit.

## DB-MIGRATION-EXEC-149
Execution deberá ser observable y explainable.

## DB-MIGRATION-EXEC-150
VoltStack nunca reportará una migración como aplicada sin haber satisfecho su execution contract.

---

# 198. Anti-patterns

## 198.1 Executor que replanea

Incorrecto:

```php
if ($platform === 'sqlite') {
    // invent new strategy here
}
```

La estrategia pertenece al Planner.

---

## 198.2 Catch + retry all

Incorrecto:

```php
catch (\Throwable $e) {
    return $this->execute($plan, $request);
}
```

---

## 198.3 Commit failure = rollback

Incorrecto:

```text
commit threw exception
⇒
transaction rolled back
```

---

## 198.4 Applied record antes de ejecución

Incorrecto:

```text
insert migration row
↓
execute migration
```

sin protocol específico.

---

## 198.5 Delete repository record before rollback

Incorrecto:

```text
remove applied record
↓
attempt rollback
```

---

## 198.6 Ignore lost lock

Incorrecto:

```text
lock expired
→ continue
```

---

## 198.7 Backfill without checkpoint semantics

Incorrecto:

```text
loop chunks
```

sin considerar crash/resume.

---

## 198.8 Static runtime state

Incorrecto:

```php
MigrationExecutor::$currentTransaction
```

---

## 198.9 Cleanup overwrites main failure

Incorrecto:

```text
primary failure lost because releaseLock() failed
```

---

## 198.10 Unknown outcome treated as pending

Incorrecto:

```text
unknown
→ simply retry from start
```

---

# 199. Ejemplo completo

MigrationPlan:

```text
M42 — Normalize user emails

PRECHECK
  └── RepositoryFingerprint == R17

LOCK
  └── Acquire Migration Coordination Lock

EXPAND
  └── Add normalized_email nullable

BACKFILL
  └── Chunked users.email → normalized_email

VALIDATE
  └── No NULL normalized_email

CONTRACT
  ├── Set normalized_email NOT NULL
  └── Add unique constraint

VERIFY
  └── Introspect expected structure

FINALIZE
  └── Record Applied M42
```

Execution:

```text
Create Execution E900
      ↓
Validate Plan
      ↓
Verify Repository R17
      ↓
Acquire Lock L88
      ↓
Record E900 RUNNING
      ↓
Execute EXPAND
      ↓
Backfill chunk 1
      ↓ checkpoint
Backfill chunk 2
      ↓ checkpoint
...
      ↓
Validate no NULLs
      ↓
Execute CONTRACT transaction
      ↓
Verify schema
      ↓
Record M42 applied
      ↓
Record E900 success
      ↓
Release resources
```

---

# 200. Ejemplo de fallo parcial

```text
EXPAND ✓
BACKFILL chunks 1..900 ✓
BACKFILL chunk 901 ✗
```

Resultado:

```text
MigrationExecutionStatus:
PARTIALLY_APPLIED

Certainty:
PARTIAL

Recovery:
resume from checkpoint 900
subject to backfill replay safety
```

No:

```text
FAILED + restart everything
```

---

# 201. Ejemplo de commit incierto

```text
BEGIN
ALTER TABLE ...
COMMIT sent
connection lost
```

Resultado correcto:

```text
UNKNOWN_OUTCOME
```

Recovery descriptor:

```text
verify schema state
verify transaction effect
do not retry blindly
```

---

# 202. Ejemplo de repository write failure

```text
Schema migration completed
Postconditions passed
Repository INSERT failed
```

Resultado:

```text
REQUIRES_RECOVERY
```

con:

```text
physical effects likely committed
repository applied record absent
```

---

# 203. Correctness formula

```text
CorrectMigrationExecution
=
ValidPlan
∧
FreshAssumptions
∧
LockContractSatisfied
∧
PreconditionsSatisfied
∧
TransactionBoundariesRespected
∧
OperationContractsSatisfied
∧
OutcomeCertaintyPreserved
∧
PostconditionsSatisfied
∧
RepositoryProtocolSatisfied
∧
ResourcesReleased
```

---

# 204. Success formula

```text
MigrationSucceeded
=
AllRequiredUnitsSucceeded
∧
AllRequiredPostconditionsSucceeded
∧
AppliedRepositoryRecordPersisted
∧
NoBlockingUncertainty
```

---

# 205. Partial formula

```text
MigrationPartiallyApplied
=
CommittedEffectsExist
∧
¬MigrationExecutionContractCompleted
```

---

# 206. Unknown formula

```text
MigrationOutcomeUnknown
=
InsufficientEvidenceToDetermineRequiredEffects
```

---

# 207. Safe retry formula

```text
SafeRetry(Unit)
=
RetryPolicyAllows
∧
FailureClassRetryable
∧
OutcomeKnownEnough
∧
ReplayabilitySatisfied
∧
IdempotencyOrCompensationSatisfied
∧
TransactionStateCompatible
∧
CheckpointStateCompatible
∧
DeadlineRemaining
∧
RetryBudgetRemaining
```

---

# 208. Regla arquitectónica maestra

> **El Migration Execution System ejecuta exactamente el plan aceptado, preserva la certeza sobre los efectos realizados y nunca utiliza excepciones o retries como sustituto de un modelo explícito de estado.**

En forma resumida:

```text
Planner decides.
Executor coordinates.
Compiler represents.
Transaction Manager scopes.
Query/Schema engines perform.
Repository records.
Recovery reconciles uncertainty.
```

---

# 209. Resultado arquitectónico

Con este diseño VoltStack obtiene un runtime de migraciones capaz de manejar:

```text
simple development migrations
large production migrations
online schema changes
chunked backfills
transactional and non-transactional DDL
distributed migration locking
persistent workers
timeouts
cancellation
safe retries
partial execution
unknown outcomes
recovery workflows
auditable repository recording
```

sin reducir la ejecución a:

```php
foreach ($pending as $migration) {
    $migration->up();
    $repository->markAsRun($migration);
}
```

La ejecución se convierte en una capa formal que protege la coherencia entre:

```text
planned intent
database effects
execution certainty
persistent migration history
```

---

# 210. Siguiente documento

```text
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md
```

El siguiente documento deberá definir el modelo de reversión de VoltStack, incluyendo:

```text
rollback definition
rollback eligibility
reversibility classification
rollback target selection
reverse dependency analysis
batch rollback
explicit down operations
structural reversibility
data reversibility
compensation
forward recovery
rollback planning
rollback execution
partial rollback
unknown rollback outcome
repository state transitions
rollback safety
post-rollback verification
irreversible migrations
conditional reversibility
zero-downtime rollback boundaries
```

y deberá fijar especialmente:

```text
Rollback
≠
Reverse(UP)

Structural Reverse
≠
Data Recovery

Rollback
≠
Compensation

Rollback
≠
Forward Recovery

Migration marked Applied
≠
Automatically safe to rollback
```