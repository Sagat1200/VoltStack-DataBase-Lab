# 108_DATABASE_MIGRATION_BATCH_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Batch System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 108 — Database Migration Batch System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Batch System` define la arquitectura mediante la cual VoltStack agrupa, identifica, ejecuta, registra, consulta, reanuda y revierte conjuntos de migraciones relacionadas por una misma operación de aplicación.

Un batch permite representar formalmente:

```text
"estas migraciones fueron aplicadas juntas
como parte de esta operación de migración"
```

sin convertir esa agrupación en:

- una dependencia semántica;
- una única transacción;
- una unidad indivisible;
- un deployment;
- una migration;
- un execution attempt;
- una garantía de atomicidad.

La pregunta fundamental es:

> **¿Cómo representa VoltStack un conjunto de migraciones aplicado conjuntamente sin confundir agrupación histórica, orden de dependencias, transacciones y ejecución runtime?**

---

# 2. Principio central

> **Un Migration Batch es una agrupación durable de migraciones asociadas a una misma operación lógica de aplicación; no es una migración ni una transacción.**

Por tanto:

```text
MigrationBatch
≠
Migration
```

y:

```text
MigrationBatch
≠
DatabaseTransaction
```

---

# 3. Separaciones fundamentales

VoltStack deberá preservar:

```text
Batch ID
≠
Migration ID

Batch ID
≠
Execution ID

Batch ID
≠
Deployment ID

Batch Membership
≠
Migration Dependency

Batch Order
≠
Dependency Order

Batch Number
≠
Migration Version

Discovery Order
≠
Batch Membership

Batch
≠
Transaction

Batch
≠
Migration Plan

Batch
≠
Execution Attempt

Batch Rollback
≠
Transaction Rollback

Partial Batch
≠
Failed Transaction
```

---

# 4. Posición arquitectónica

```text
Migration Discovery
        │
        ▼
Migration Catalog
        │
        ▼
Migration Repository
        │
        ▼
Pending Resolution
        │
        ▼
Migration Planner
        │
        ▼
┌──────────────────────────────┐
│     Batch Planning           │
│                              │
│ Pending Migration Set        │
│ Dependency Ordering          │
│ Batch Identity               │
│ Batch Membership             │
└──────────────┬───────────────┘
               ▼
       MigrationBatchPlan
               │
               ▼
┌──────────────────────────────┐
│     Batch Execution          │
│                              │
│ Migration A                  │
│ Migration B                  │
│ Migration C                  │
└──────────────┬───────────────┘
               ▼
       Migration Repository
               │
               ▼
         Batch History
```

---

# 5. Motivación

Supóngase que el repository contiene:

```text
M001 ✓
M002 ✓
M003 ✓
```

y aparecen:

```text
M004
M005
M006
```

Una ejecución:

```bash
php volt migration:run
```

puede aplicar:

```text
M004
M005
M006
```

como:

```text
Batch B17
```

El repository puede representar:

```text
M004 → B17
M005 → B17
M006 → B17
```

Esto permite posteriormente:

```text
rollback batch B17
```

sin asumir que:

```text
B17 = one transaction
```

---

# 6. Modelo conceptual

```text
MigrationBatch
├── Batch Identity
├── Sequence
├── Target Scope
├── Migration Membership
├── Application Order
├── Plan Fingerprint
├── Execution History
├── Completion State
├── Outcome Certainty
└── Metadata
```

---

# 7. Dos conceptos diferentes

Es importante distinguir:

```text
MigrationBatchPlan
```

de:

```text
MigrationBatchRecord
```

---

# 8. MigrationBatchPlan

Representa:

> El conjunto planificado de migraciones que se pretende aplicar como una unidad lógica de ejecución.

Es immutable.

---

# 9. MigrationBatchRecord

Representa:

> La identidad durable del batch y el resultado histórico de su aplicación.

Es almacenado por Migration Repository.

---

# 10. Batch execution

También existe:

```text
MigrationBatchExecution
```

que representa:

> El estado runtime mutable de un intento concreto de ejecutar un batch.

Por tanto:

```text
MigrationBatchPlan
≠
MigrationBatchRecord
≠
MigrationBatchExecution
```

---

# 11. Identidades

Se proponen:

```text
MigrationBatchId
MigrationBatchSequence
MigrationBatchExecutionId
```

---

# 12. MigrationBatchId

Identidad durable del batch.

Ejemplo conceptual:

```text
batch:01K5Y8G31AH6...
```

Puede utilizar:

- UUID;
- ULID;
- identificador equivalente.

No debe depender únicamente de un número incremental.

---

# 13. MigrationBatchSequence

Puede existir además:

```text
17
18
19
```

para DX.

Pero:

```text
MigrationBatchSequence
≠
MigrationBatchId
```

---

# 14. Razón

En sistemas distribuidos:

```text
MAX(batch) + 1
```

no es una estrategia segura de identidad.

---

# 15. Batch number Laravel-like

VoltStack puede ofrecer:

```text
Batch #17
```

como representación humana.

Internamente:

```text
BatchId = stable unique identity
BatchSequence = ordered display/reference value
```

---

# 16. Sequence allocation

Se propone:

```php
interface MigrationBatchSequenceAllocator
{
    public function next(
        MigrationTargetIdentity $target
    ): MigrationBatchSequence;
}
```

---

# 17. Allocation concurrency

La asignación deberá ser:

```text
atomic
```

o protegida mediante:

```text
migration coordination lock
```

---

# 18. Scope de sequence

Default:

```text
per migration repository target
```

No global a todo VoltStack.

---

# 19. Batch membership

Se propone:

```php
final readonly class MigrationBatchMember
{
    public function __construct(
        public MigrationId $migration,
        public MigrationBatchPosition $position,
        public MigrationPlanFingerprint $planFingerprint,
    ) {}
}
```

---

# 20. Membership ≠ dependency

Dos migrations pueden estar en el mismo batch sin depender entre sí:

```text
M10
M20
```

---

# 21. Dependency ≠ membership

También:

```text
M10 → M20
```

pueden pertenecer a batches diferentes.

---

# 22. Batch position

Puede almacenarse:

```text
1
2
3
```

para preservar el orden aplicado.

---

# 23. Position semantics

```text
BatchPosition
```

representa orden dentro del batch.

No reemplaza el dependency graph.

---

# 24. Orden del batch

Debe derivarse de:

```text
Migration Dependency Graph
+
Planner Constraints
+
Deterministic Tie-Breaking
```

---

# 25. Discovery order

Nunca:

```text
filesystem iteration order
```

como contrato de batch.

---

# 26. Filename order

Puede participar como tie-breaker convencional.

Pero:

```text
Filename Order
≠
Dependency Semantics
```

---

# 27. Batch planning

Se propone:

```php
interface MigrationBatchPlanner
{
    public function plan(
        MigrationPlan $migrationPlan,
        MigrationBatchPlanningContext $context
    ): MigrationBatchPlan;
}
```

---

# 28. Input principal

Normalmente:

```text
MigrationPlan
```

ya contiene:

```text
pending migrations
dependency resolution
execution ordering
strategy
```

El Batch Planner no deberá reconstruir esas decisiones.

---

# 29. Batch Planner responsibilities

Deberá decidir:

- batch identity;
- batch sequence;
- membership;
- logical ordering;
- target scope;
- batch policy;
- repository expectations;
- execution metadata;
- batch fingerprint.

---

# 30. Batch Planner no decide

No deberá decidir nuevamente:

```text
schema strategy
zero-downtime strategy
migration safety
SQL generation
transaction internals
```

---

# 31. MigrationBatchPlan

```php
final readonly class MigrationBatchPlan
{
    public function __construct(
        public MigrationBatchId $id,
        public MigrationBatchSequence $sequence,
        public MigrationTargetIdentity $target,
        public MigrationBatchMemberSet $members,
        public MigrationBatchPolicy $policy,
        public MigrationRepositoryFingerprint $repositoryFingerprint,
        public MigrationBatchPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 32. Plan fingerprint

Debe incluir semánticamente:

```text
batch ID
sequence
target
ordered membership
migration plan fingerprints
policy-relevant data
repository expectation
```

---

# 33. Fingerprint ≠ execution identity

Dos execution attempts del mismo batch plan pueden ser:

```text
E1
E2
```

sin cambiar:

```text
BatchPlanFingerprint
```

---

# 34. Batch policy

Se propone:

```text
MigrationBatchPolicy
```

con opciones como:

```text
STOP_ON_FAILURE
CONTINUE_INDEPENDENT
STRICT_ATOMIC_WHEN_POSSIBLE
RESUMABLE
NON_RESUMABLE
```

---

# 35. STOP_ON_FAILURE

Default recomendado:

```text
Migration A ✓
Migration B ✗
Migration C not started
```

---

# 36. Migration C state

Debe ser:

```text
NOT_STARTED
```

o:

```text
BLOCKED
```

si depende de B.

No:

```text
FAILED
```

---

# 37. CONTINUE_INDEPENDENT

Puede permitirse:

```text
A ✓
B ✗
C ✓
```

solo si:

```text
C independent of B
```

y policy lo permite.

---

# 38. Default conservador

VoltStack debería usar:

```text
STOP_ON_FAILURE
```

por default.

---

# 39. Batch ≠ atomic transaction

Un batch:

```text
B17
├── M1
├── M2
└── M3
```

puede ejecutar:

```text
TX1 → M1
TX2 → M2
DDL non-transactional → M3
```

---

# 40. Atomic batch

Solo podrá afirmarse:

```text
ATOMIC
```

si existe prueba real de que todas sus operaciones están protegidas por una única atomic boundary.

---

# 41. Atomicity classification

Se propone:

```text
ATOMIC
PARTIALLY_ATOMIC
NON_ATOMIC
UNKNOWN
```

---

# 42. Batch atomicity report

```php
final readonly class MigrationBatchAtomicityReport
{
    public function __construct(
        public MigrationBatchAtomicity $status,
        public array $transactionGroups,
        public array $nonTransactionalOperations,
        public array $evidence,
    ) {}
}
```

---

# 43. Batch execution

Se propone:

```php
interface MigrationBatchExecutor
{
    public function execute(
        MigrationBatchPlan $plan,
        MigrationBatchExecutionRequest $request
    ): MigrationBatchExecutionResult;
}
```

---

# 44. Reutilización

El Batch Executor deberá delegar cada migration a:

```text
Migration Execution System
```

del documento 106.

---

# 45. No second migration executor

Incorrecto:

```text
Batch Executor
→ custom SQL runner
```

Correcto:

```text
Batch Executor
→ Migration Executor
→ Schema/Query Execution
```

---

# 46. Batch execution lifecycle

```text
CREATED
   ↓
VALIDATING
   ↓
LOCKING
   ↓
RUNNING
   ↓
FINALIZING
   ↓
SUCCEEDED
```

Alternativas:

```text
PARTIALLY_APPLIED
FAILED
CANCELLED
TIMED_OUT
UNKNOWN
REQUIRES_RECOVERY
```

---

# 47. BatchExecutionId

Cada intento tendrá:

```text
MigrationBatchExecutionId
```

Ejemplo:

```text
Batch B17
├── Execution E1 → FAILED
└── Execution E2 → SUCCEEDED
```

---

# 48. Batch state vs execution state

```text
Batch durable state
≠
Batch execution attempt state
```

---

# 49. Member runtime states

Cada member puede estar:

```text
PENDING
READY
RUNNING
SUCCEEDED
FAILED
BLOCKED
SKIPPED
UNKNOWN
```

---

# 50. Batch state derivation

Ejemplo:

```text
M1 = SUCCEEDED
M2 = FAILED
M3 = NOT_STARTED
```

Batch:

```text
PARTIALLY_APPLIED
```

si M1 produjo efectos durables.

---

# 51. Failed batch

`FAILED` puro solo debe utilizarse cuando el batch contract falla y puede determinarse que:

```text
no durable member application remains
```

---

# 52. Partial batch

Formalmente:

```text
PartialBatch
=
∃ AppliedMember
∧
¬AllRequiredMembersApplied
```

---

# 53. Unknown batch

```text
UNKNOWN
```

si el estado de uno o más members relevantes no puede determinarse suficientemente.

---

# 54. Outcome certainty

Debe agregarse conservadoramente.

Ejemplo:

```text
M1 = CERTAIN_SUCCESS
M2 = UNKNOWN
```

Batch certainty:

```text
UNKNOWN
```

---

# 55. Aggregate certainty

Nunca:

```text
majority success
→ batch success
```

---

# 56. Execution order

```text
for member in batchPlan.membersInExecutionOrder()
```

pero respetando graph dependencies.

---

# 57. Dependency-aware scheduling

Conceptualmente:

```text
Ready Set
   ↓
Select deterministic member
   ↓
Execute
   ↓
Update graph
   ↓
Resolve newly-ready members
```

---

# 58. Parallel batch execution

Puede existir en el futuro.

Solo si:

```text
dependency independence
+
target concurrency safety
+
transaction independence
+
lock compatibility
+
policy allows
```

---

# 59. Default

```text
sequential
```

---

# 60. Razón

Migrations son:

```text
rare
high-impact
operationally sensitive
```

Por tanto throughput no debe sacrificar predictibilidad.

---

# 61. Coordination lock

El batch normalmente deberá adquirir:

```text
Migration Coordination Lock
```

para el target.

---

# 62. Lock ownership

Dos alternativas:

```text
Batch owns lock
```

o:

```text
Migration owns lock
```

La recomendación default:

```text
Batch owns coordination lock
```

para evitar que otro migrator intercale migrations.

---

# 63. Lock hierarchy

```text
Batch Lock
    ↓
Migration Execution
```

La migration puede usar locks internos adicionales cuando el plan lo requiera.

---

# 64. No lock double ownership

Migration Executor deberá conocer que el coordination lock ya está adquirido mediante:

```text
MigrationExecutionContext
```

---

# 65. Lock lease

Batch execution deberá conservar:

```text
MigrationLockLease
```

durante toda la ejecución cuando policy lo requiera.

---

# 66. Lock loss

Si se pierde:

```text
stop before next unsafe boundary
```

---

# 67. Batch execution record

Repository deberá poder almacenar:

```text
batch started
batch completed
batch partial
batch failed
batch unknown
```

---

# 68. Repository architecture

Conceptualmente:

```text
migration_batches
migration_batch_executions
migration_execution_history
migration_current_state
```

sin obligar aún a un schema físico exacto.

---

# 69. Batch record

Se propone:

```php
final readonly class MigrationBatchRecord
{
    public function __construct(
        public MigrationBatchId $id,
        public MigrationBatchSequence $sequence,
        public MigrationTargetIdentity $target,
        public MigrationBatchStatus $status,
        public Instant $createdAt,
        public ?Instant $completedAt,
        public MigrationBatchPlanFingerprint $planFingerprint,
    ) {}
}
```

---

# 70. Membership record

```php
final readonly class MigrationBatchMembershipRecord
{
    public function __construct(
        public MigrationBatchId $batch,
        public MigrationId $migration,
        public MigrationBatchPosition $position,
        public MigrationBatchMemberStatus $status,
    ) {}
}
```

---

# 71. Membership history

Una migration re-aplicada tras rollback puede pertenecer a un batch posterior.

Ejemplo:

```text
M42
├── B10 APPLY
├── R11 ROLLBACK
└── B15 RE-APPLY
```

Por tanto:

```text
MigrationId → one batch forever
```

es incorrecto.

---

# 72. Application occurrence

Puede ser útil introducir:

```text
MigrationApplicationId
```

para identificar cada aplicación durable de una migration.

---

# 73. MigrationApplicationId

Entonces:

```text
MigrationDefinition M42
        │
        ├── Application A1 → Batch B10
        └── Application A2 → Batch B15
```

---

# 74. Beneficio

Esto evita sobrecargar:

```text
MigrationId
```

con history runtime.

---

# 75. Applied state

El current repository state puede indicar:

```text
M42 → currently applied via A2
```

mientras history conserva A1.

---

# 76. Batch repository consistency

Ideal:

```text
record batch creation
+
record execution start
```

antes de efectos.

---

# 77. Batch record before execution

Crear batch record no significa:

```text
batch applied
```

Estado inicial:

```text
PLANNED
```

o:

```text
RUNNING
```

---

# 78. Member applied record

Solo se marcará:

```text
SUCCEEDED/APPLIED
```

cuando el Migration Execution contract se satisfaga.

---

# 79. Batch finalization

Solo:

```text
SUCCEEDED
```

si todos los required members completan su contract.

---

# 80. Batch repository write failure

Si migrations fueron aplicadas pero batch finalization falla:

```text
REQUIRES_RECOVERY
```

---

# 81. Batch record ≠ source of physical truth

Repository informa history.

Ante crash:

```text
repository
+
schema/data verification
```

pueden ser necesarios.

---

# 82. Crash scenario

```text
B17 RUNNING
M1 APPLIED
M2 APPLIED
process crashes before M3
```

Al reiniciar:

```text
do not create B18 blindly
```

Debe reconciliarse B17.

---

# 83. Resumable batch

Se propone:

```text
MigrationBatchResumeSystem
```

como parte del batch subsystem.

---

# 84. Resume request

```php
final readonly class MigrationBatchResumeRequest
{
    public function __construct(
        public MigrationBatchId $batch,
        public MigrationBatchResumePolicy $policy,
    ) {}
}
```

---

# 85. Resume ≠ retry from beginning

Debe:

```text
load batch
↓
load member states
↓
reconcile uncertain states
↓
validate plan/fingerprints
↓
identify incomplete members
↓
resume safe members
```

---

# 86. Resume preconditions

Debe verificar:

```text
same migration definitions
same plan compatibility
repository state coherent
target unchanged
schema assumptions valid
no unknown member requiring reconciliation
```

---

# 87. Resume after certain failure

Ejemplo:

```text
M1 success
M2 certain failure before effects
M3 not started
```

Puede:

```text
resume at M2
```

si M2 retry/replay contract lo permite.

---

# 88. Resume after unknown

No:

```text
UNKNOWN M2
→ rerun M2
```

Primero:

```text
reconcile M2
```

---

# 89. Reconciliation

Puede producir:

```text
APPLIED
NOT_APPLIED
PARTIALLY_APPLIED
UNKNOWN
```

---

# 90. Reconciliation provider

```php
interface MigrationBatchReconciler
{
    public function reconcile(
        MigrationBatchRecord $batch,
        MigrationBatchRepositorySnapshot $repository,
        MigrationRuntimeInspectionContext $context
    ): MigrationBatchReconciliationReport;
}
```

---

# 91. Resume and plan drift

Si la migration definition cambió:

```text
resume blocked
```

---

# 92. Resume and capability drift

Si DB capabilities cambiaron:

```text
replan/revalidate required
```

---

# 93. New migrations during resume

Supóngase que B17 quedó parcial y después aparece:

```text
M100
```

No debe añadirse silenciosamente a B17.

---

# 94. Immutable membership

Una vez creado durablemente:

```text
Batch Membership
=
immutable
```

---

# 95. New pending migration

Debe pertenecer a:

```text
future batch
```

después de resolver B17.

---

# 96. Batch completion invariant

```text
Batch SUCCEEDED
⇒
Every Required Member SUCCEEDED
```

---

# 97. Batch rollback

`rollback last batch` deberá resolver:

```text
latest rollback-eligible applied batch
```

según repository semantics.

---

# 98. Batch rollback ≠ reverse member loop

Debe utilizar:

```text
Migration Rollback System
```

del documento 107.

---

# 99. Pipeline

```text
Batch B17
    ↓
Resolve currently-applied members
    ↓
Dependency analysis
    ↓
Rollback target set
    ↓
Rollback Planner
    ↓
Rollback Execution
```

---

# 100. Batch membership and rollback

No todos los members históricos de B17 necesariamente siguen aplicados.

Ejemplo:

```text
B17:
M1
M2
M3

later:
M3 rolled back separately
```

`rollback B17` deberá considerar:

```text
current state
```

no solo membership original.

---

# 101. Rollback order

Derivado de:

```text
current applied dependency graph
```

normalmente reverse topological.

---

# 102. Batch rollback atomicity

No asumir:

```text
rollback B17
=
one transaction
```

---

# 103. Partial batch rollback

Puede existir:

```text
M3 rollback ✓
M2 rollback ✗
M1 not attempted
```

Resultado:

```text
PARTIALLY_ROLLED_BACK
```

---

# 104. Batch history after rollback

No eliminar B17.

Debe conservar:

```text
B17 APPLY history
R20 rollback history
```

---

# 105. Batch relation to deployment

Puede existir integración:

```text
Deployment D900
   └── Migration Batch B17
```

pero:

```text
DeploymentId
≠
BatchId
```

---

# 106. Deployment metadata

Batch puede almacenar:

```text
deployment correlation ID
release version
CI run ID
```

como metadata externa.

---

# 107. Metadata ≠ identity

Cambiar CI provider no cambia semántica del batch.

---

# 108. Package migrations

Un batch puede contener:

```text
application migrations
framework migrations
package migrations
```

si forman parte del mismo pending plan.

---

# 109. Source metadata

Cada member conserva:

```text
MigrationSourceIdentity
```

---

# 110. Package boundaries

Package membership:

```text
≠
batch boundary
```

por default.

---

# 111. Optional batch partitioning

Policy podría crear batches separados por:

```text
target
module
package
risk class
deployment phase
```

pero debe ser explícito.

---

# 112. Recommended default

Un invocation lógico sobre un target:

```text
one batch
```

para todas las pending migrations aceptadas.

---

# 113. Multi-target batches

Una migration command puede involucrar:

```text
Database A
Database B
```

---

# 114. Recomendación

Mantener:

```text
one durable batch per migration target
```

y usar:

```text
MigrationBatchGroup
```

para correlacionarlos.

---

# 115. BatchGroup

```php
final readonly class MigrationBatchGroup
{
    public function __construct(
        public MigrationBatchGroupId $id,
        public MigrationBatchIdSet $batches,
    ) {}
}
```

---

# 116. Razón

Evita fingir:

```text
distributed atomicity
```

---

# 117. Group result

Puede ser:

```text
ALL_SUCCEEDED
PARTIAL
FAILED
UNKNOWN
```

---

# 118. Multi-target example

```text
Batch Group G10
├── Database A → Batch BA → SUCCESS
└── Database B → Batch BB → FAILED
```

Resultado:

```text
G10 = PARTIAL
```

---

# 119. No hidden distributed transaction

VoltStack no implementará automáticamente:

```text
2PC
```

solo por agrupar batches.

---

# 120. Concurrent migrators

Caso:

```text
Worker A → plans B17
Worker B → plans B18
```

sobre mismo target.

Debe impedirse execution interleaving no controlado.

---

# 121. Coordination

Mediante:

```text
migration lock
+
repository freshness
+
batch sequence allocation
```

---

# 122. Stale batch plan

Si:

```text
RepositoryFingerprint planned = R10
```

pero al adquirir lock:

```text
CurrentRepositoryFingerprint = R11
```

entonces:

```text
STALE_BATCH_PLAN
```

---

# 123. No continue

Default:

```text
abort
→ replan
```

---

# 124. Lock-before-sequence alternative

Para máxima consistencia:

```text
Acquire lock
↓
refresh repository
↓
resolve pending
↓
allocate batch sequence
↓
finalize plan
↓
execute
```

---

# 125. Planning outside lock

Puede hacerse speculative planning antes del lock.

Pero deberá revalidarse después.

---

# 126. Long lock avoidance

Esto permite:

```text
expensive analysis outside lock
+
short freshness validation inside lock
```

---

# 127. Batch sequence gaps

Deben permitirse.

Ejemplo:

```text
17
18
20
```

No depender de contigüidad.

---

# 128. Sequence ≠ count

Nunca:

```text
batch sequence = number of successful batches
```

como invariant.

---

# 129. Failed batch sequence

Un batch fallido puede conservar su sequence.

No reciclarla.

---

# 130. Immutable batch identity

Una vez persistido:

```text
BatchId
BatchSequence
Membership
Target
```

no deben cambiar.

---

# 131. Batch labels

Puede existir metadata:

```text
release-2026.09.1
```

pero no será identity.

---

# 132. Batch status

Se propone:

```text
PLANNED
RUNNING
SUCCEEDED
PARTIALLY_APPLIED
FAILED
CANCELLED
TIMED_OUT
UNKNOWN
ROLLBACK_RUNNING
PARTIALLY_ROLLED_BACK
ROLLED_BACK
REQUIRES_RECOVERY
```

---

# 133. Current batch state complexity

Un batch puede haber:

```text
succeeded
then partially rolled back
```

Por eso puede ser mejor separar:

```text
ApplicationStatus
RollbackStatus
```

en lugar de un único enum gigantesco.

---

# 134. Recommended state model

```text
MigrationBatchApplicationState
MigrationBatchRollbackState
MigrationBatchExecutionHistory
```

---

# 135. Application state

```text
PLANNED
RUNNING
APPLIED
PARTIALLY_APPLIED
NOT_APPLIED
UNKNOWN
```

---

# 136. Rollback state

```text
NOT_REQUESTED
ROLLBACK_RUNNING
ROLLED_BACK
PARTIALLY_ROLLED_BACK
ROLLBACK_FAILED
ROLLBACK_UNKNOWN
```

---

# 137. Derived overall state

La UI/CLI puede derivar:

```text
MigrationBatchDisplayStatus
```

---

# 138. State machine

```text
PLANNED
   ↓
RUNNING
   ├─────────────┬───────────────┐
   ▼             ▼               ▼
APPLIED      PARTIAL         UNKNOWN
   │
   ▼
ROLLBACK_RUNNING
   ├─────────────┬───────────────┐
   ▼             ▼               ▼
ROLLED_BACK   PARTIAL         UNKNOWN
```

---

# 139. Failure history

Un execution attempt fallido no debe destruirse tras resume.

Ejemplo:

```text
B17
├── E1 FAILED
└── E2 SUCCESS
```

---

# 140. Current state vs attempt history

```text
BatchCurrentState
≠
BatchExecutionHistory
```

---

# 141. Batch result

```php
final readonly class MigrationBatchExecutionResult
{
    public function __construct(
        public MigrationBatchExecutionId $executionId,
        public MigrationBatchId $batchId,
        public MigrationBatchApplicationState $state,
        public MigrationOutcomeCertainty $certainty,
        public MigrationBatchMemberResultSet $members,
        public ?MigrationBatchFailureReport $failure,
        public MigrationBatchRecoveryDescriptor $recovery,
    ) {}
}
```

---

# 142. Member result

```php
final readonly class MigrationBatchMemberResult
{
    public function __construct(
        public MigrationId $migration,
        public MigrationBatchMemberStatus $status,
        public ?MigrationExecutionId $execution,
        public MigrationOutcomeCertainty $certainty,
    ) {}
}
```

---

# 143. Failure report

```php
final readonly class MigrationBatchFailureReport
{
    public function __construct(
        public MigrationBatchFailureKind $kind,
        public ?MigrationId $migration,
        public MigrationOutcomeCertainty $certainty,
        public Throwable $cause,
        public MigrationBatchFailureMetadata $metadata,
    ) {}
}
```

---

# 144. Failure kinds

```text
PLAN_STALE
LOCK_FAILURE
MEMBER_FAILURE
MEMBER_UNKNOWN
REPOSITORY_FAILURE
CANCELLATION
TIMEOUT
BUDGET_EXCEEDED
EXTENSION_FAILURE
INVARIANT_FAILURE
```

---

# 145. Recovery descriptor

```text
MigrationBatchRecoveryDescriptor
```

deberá incluir:

```text
batch ID
completed members
failed member
unknown members
not-started members
repository state
schema reconciliation requirements
resume eligibility
rollback eligibility
recommended action
```

---

# 146. Recovery actions

Posibles:

```text
RESUME
RECONCILE
ROLLBACK
FORWARD_RECOVERY
MANUAL_RECOVERY
REPLAN
```

---

# 147. Resume eligibility

```text
ResumeAllowed
=
BatchIdentityValid
∧
MembershipUnchanged
∧
DefinitionsUnchanged
∧
KnownCompletedMembers
∧
UncertainMembersReconciled
∧
CurrentStateCompatible
∧
SafetyStillSatisfied
```

---

# 148. Batch cancellation

Cancellation deberá propagarse al member activo.

---

# 149. Cancellation behavior

Después:

```text
active member outcome
```

debe resolverse antes de clasificar batch cuando sea posible.

---

# 150. Cancellation after completed members

Ejemplo:

```text
M1 ✓
M2 ✓
cancel before M3
```

Batch:

```text
PARTIALLY_APPLIED
```

no simplemente:

```text
CANCELLED
```

si existen effects durables.

---

# 151. Cancellation status vs effects

Puede conservarse:

```text
terminationReason = CANCELLED
applicationState = PARTIALLY_APPLIED
```

Esto es más preciso.

---

# 152. Deadline

Batch tiene:

```text
global deadline
```

que se propaga a migrations.

---

# 153. Migration deadline

Cada migration puede tener límite menor.

Effective:

```text
min(
    remaining batch deadline,
    migration deadline
)
```

---

# 154. Deadline exhausted before next migration

No iniciarla.

---

# 155. Timeout after applied members

Batch será:

```text
PARTIALLY_APPLIED
```

con:

```text
terminationReason = TIMEOUT
```

---

# 156. Retry

Batch-level retry no deberá simplemente ejecutar todo nuevamente.

---

# 157. Retry model

```text
Batch Retry
=
Resume/Reconciliation
```

más que:

```text
restart loop
```

---

# 158. Safe member retry

Delegado al:

```text
Migration Execution System
```

---

# 159. Batch retry budget

Puede limitar:

```text
max resume attempts
max total execution duration
max member retry attempts
```

---

# 160. Batch budgets

Se propone:

```text
max migrations
max total runtime
max lock duration
max retry attempts
max total backfill work
```

---

# 161. Budget exceeded

Si ningún member inició:

```text
FAILED/BLOCKED
```

puede ser cierto.

Si algunos fueron aplicados:

```text
PARTIALLY_APPLIED
```

---

# 162. Events

Se proponen:

```text
MigrationBatchPlanning
MigrationBatchPlanned
MigrationBatchStarting
MigrationBatchLockAcquiring
MigrationBatchLockAcquired
MigrationBatchMemberStarting
MigrationBatchMemberCompleted
MigrationBatchMemberFailed
MigrationBatchMemberBlocked
MigrationBatchPartiallyApplied
MigrationBatchCompleted
MigrationBatchFailed
MigrationBatchResumeStarting
MigrationBatchResumed
MigrationBatchRecoveryRequired
```

---

# 163. Event listeners

No podrán:

```text
change membership
change batch sequence
skip required migration
mark batch applied
forge member success
```

---

# 164. Telemetry

Debe poder medir:

```text
batch duration
lock wait
migration count
success count
failure count
blocked count
resume count
partial batches
unknown batches
batch rollback duration
```

---

# 165. Correlation

Telemetry deberá correlacionar:

```text
BatchGroupId?
BatchId
BatchExecutionId
MigrationId
MigrationExecutionId
TargetId
```

---

# 166. Security

Batch execution es privileged.

Debe validar:

```text
authorized caller
target authorization
plan integrity
migration approvals
raw operation policies
```

---

# 167. Batch tampering

Modificar serialized membership:

```text
fingerprint mismatch
```

---

# 168. Audit

Debe poder registrar:

```text
who
when
target
batch ID
batch sequence
members
plan fingerprint
approvals
execution result
rollback result
```

---

# 169. Persistent runtime

Shared:

```text
BatchPlanner
BatchExecutor
frozen registries
configuration
```

Operation-scoped:

```text
BatchExecutionSession
lock lease
member graph state
deadline
cancellation
repository session
recovery state
```

---

# 170. FrankenPHP

Nunca mantener:

```text
current batch
```

como static mutable state entre requests.

---

# 171. RoadRunner

Cada job deberá iniciar contexto limpio.

---

# 172. OpenSwoole

Cada coroutine deberá tener:

```text
independent BatchExecutionSession
```

---

# 173. Extension model

Se propone:

```text
MigrationBatchExtension
```

para:

```text
custom batch partitioning
custom metadata
custom sequence allocator
custom reconciliation evidence
custom grouping strategy
```

---

# 174. Extension restrictions

No podrá:

- modificar un batch persistido;
- introducir members no planificados;
- omitir required dependencies;
- reutilizar BatchId;
- falsificar repository state;
- convertir UNKNOWN en success sin evidencia.

---

# 175. Frozen registry

Registries deberán congelarse tras bootstrap.

---

# 176. Testing

El sistema deberá probar:

```text
empty pending set
single migration batch
multiple migration batch
dependency ordering
independent migrations
sequence allocation
concurrent allocation
lock contention
member failure
member unknown outcome
partial batch
resume
reconciliation
definition drift
repository drift
new migrations after partial batch
batch rollback
partial rollback
multi-target group
deadline
cancellation
budget exhaustion
persistent runtime isolation
```

---

# 177. Empty pending set

Si:

```text
Pending = ∅
```

recomendación:

```text
do not create empty durable batch
```

---

# 178. No-op result

CLI puede devolver:

```text
Nothing to migrate.
```

sin incrementar batch sequence.

---

# 179. Concurrent batch test

Simular:

```text
Migrator A
Migrator B
```

intentando migrar mismo target.

Solo uno debe pasar el coordination gate.

---

# 180. Crash test

Simular:

```text
M1 applied
M2 applied
crash
```

y comprobar que resume no reaplica M1/M2 ciegamente.

---

# 181. Definition drift test

Modificar M2 después de partial batch.

Resume debe bloquear.

---

# 182. Unknown outcome test

M2:

```text
COMMIT sent
connection lost
```

Batch deberá ser:

```text
UNKNOWN / REQUIRES_RECOVERY
```

hasta reconciliation.

---

# 183. Error hierarchy

```text
DatabaseMigrationBatchException
├── MigrationBatchPlanningException
├── MigrationBatchIntegrityException
├── MigrationBatchStalePlanException
├── MigrationBatchSequenceAllocationException
├── MigrationBatchIdentityCollisionException
├── MigrationBatchMembershipException
├── MigrationBatchExecutionException
├── MigrationBatchMemberExecutionException
├── MigrationBatchPartialExecutionException
├── MigrationBatchOutcomeUnknownException
├── MigrationBatchResumeException
├── MigrationBatchResumeNotAllowedException
├── MigrationBatchReconciliationException
├── MigrationBatchRepositoryException
├── MigrationBatchLockException
├── MigrationBatchRollbackException
├── MigrationBatchBudgetExceededException
├── MigrationBatchExtensionException
└── MigrationBatchInvariantException
```

---

# 184. Namespace

```text
VoltStack\Quantum\Database\Migration\Batch
```

---

# 185. Estructura propuesta

```text
Migration/
└── Batch/
    ├── Contract/
    │   ├── MigrationBatchPlanner.php
    │   ├── MigrationBatchExecutor.php
    │   ├── MigrationBatchReconciler.php
    │   └── MigrationBatchSequenceAllocator.php
    │
    ├── Identity/
    │   ├── MigrationBatchId.php
    │   ├── MigrationBatchGroupId.php
    │   ├── MigrationBatchExecutionId.php
    │   ├── MigrationBatchSequence.php
    │   └── MigrationApplicationId.php
    │
    ├── Planning/
    │   ├── DefaultMigrationBatchPlanner.php
    │   ├── MigrationBatchPlan.php
    │   ├── MigrationBatchPlanningContext.php
    │   ├── MigrationBatchPolicy.php
    │   └── MigrationBatchPlanFingerprint.php
    │
    ├── Member/
    │   ├── MigrationBatchMember.php
    │   ├── MigrationBatchMemberSet.php
    │   ├── MigrationBatchPosition.php
    │   └── MigrationBatchMemberStatus.php
    │
    ├── Group/
    │   ├── MigrationBatchGroup.php
    │   ├── MigrationBatchGroupResult.php
    │   └── MigrationBatchGroupCoordinator.php
    │
    ├── Atomicity/
    │   ├── MigrationBatchAtomicity.php
    │   ├── MigrationBatchAtomicityReport.php
    │   └── MigrationBatchAtomicityAnalyzer.php
    │
    ├── Execution/
    │   ├── DefaultMigrationBatchExecutor.php
    │   ├── MigrationBatchExecution.php
    │   ├── MigrationBatchExecutionRequest.php
    │   ├── MigrationBatchExecutionSession.php
    │   └── MigrationBatchExecutionStateMachine.php
    │
    ├── State/
    │   ├── MigrationBatchApplicationState.php
    │   ├── MigrationBatchRollbackState.php
    │   ├── MigrationBatchTerminationReason.php
    │   └── MigrationBatchDisplayStatus.php
    │
    ├── Repository/
    │   ├── MigrationBatchRecord.php
    │   ├── MigrationBatchMembershipRecord.php
    │   ├── MigrationBatchExecutionRecord.php
    │   └── MigrationBatchRepositorySnapshot.php
    │
    ├── Resume/
    │   ├── MigrationBatchResumeRequest.php
    │   ├── MigrationBatchResumePolicy.php
    │   ├── MigrationBatchResumePlanner.php
    │   └── MigrationBatchResumeResult.php
    │
    ├── Reconciliation/
    │   ├── MigrationBatchReconciler.php
    │   ├── MigrationBatchReconciliationReport.php
    │   └── MigrationBatchMemberReconciliation.php
    │
    ├── Result/
    │   ├── MigrationBatchExecutionResult.php
    │   ├── MigrationBatchMemberResult.php
    │   ├── MigrationBatchMemberResultSet.php
    │   └── MigrationBatchFailureReport.php
    │
    ├── Recovery/
    │   └── MigrationBatchRecoveryDescriptor.php
    │
    ├── Rollback/
    │   └── MigrationBatchRollbackCoordinator.php
    │
    ├── Budget/
    │   └── MigrationBatchBudget.php
    │
    ├── Telemetry/
    │   └── MigrationBatchTelemetry.php
    │
    ├── Extension/
    │   ├── MigrationBatchExtension.php
    │   └── MigrationBatchExtensionRegistry.php
    │
    └── Exception/
        └── ...
```

---

# 186. Invariantes

## DB-MIGRATION-BATCH-001
Batch será distinto de Migration.

## DB-MIGRATION-BATCH-002
Batch será distinto de transaction.

## DB-MIGRATION-BATCH-003
Batch será distinto de deployment.

## DB-MIGRATION-BATCH-004
Batch será distinto de execution attempt.

## DB-MIGRATION-BATCH-005
Batch ID será distinto de Migration ID.

## DB-MIGRATION-BATCH-006
Batch ID será distinto de Execution ID.

## DB-MIGRATION-BATCH-007
Batch ID será distinto de Deployment ID.

## DB-MIGRATION-BATCH-008
Batch sequence será distinta de Batch ID.

## DB-MIGRATION-BATCH-009
Batch sequence será distinta de Migration Version.

## DB-MIGRATION-BATCH-010
Batch sequence podrá contener gaps.

## DB-MIGRATION-BATCH-011
Batch sequence no será reutilizada por default.

## DB-MIGRATION-BATCH-012
Batch identity será estable.

## DB-MIGRATION-BATCH-013
Batch membership será immutable después de persistirse.

## DB-MIGRATION-BATCH-014
Batch membership será distinta de dependency.

## DB-MIGRATION-BATCH-015
Dependency será distinta de membership.

## DB-MIGRATION-BATCH-016
Batch position no sustituirá dependency graph.

## DB-MIGRATION-BATCH-017
Discovery order no definirá semantic batch order.

## DB-MIGRATION-BATCH-018
Filesystem iteration order no será contrato.

## DB-MIGRATION-BATCH-019
Filename podrá ser tie-breaker, no dependency.

## DB-MIGRATION-BATCH-020
Batch Planner consumirá Migration Plan.

## DB-MIGRATION-BATCH-021
Batch Planner no reconstruirá schema strategy.

## DB-MIGRATION-BATCH-022
Batch Planner no generará SQL.

## DB-MIGRATION-BATCH-023
Batch Plan será immutable.

## DB-MIGRATION-BATCH-024
Batch Plan será distinto de Batch Record.

## DB-MIGRATION-BATCH-025
Batch Plan será distinto de Batch Execution.

## DB-MIGRATION-BATCH-026
Batch fingerprint será determinista.

## DB-MIGRATION-BATCH-027
Execution attempt no cambiará plan fingerprint.

## DB-MIGRATION-BATCH-028
STOP_ON_FAILURE será default recomendado.

## DB-MIGRATION-BATCH-029
Independent continuation requerirá policy explícita.

## DB-MIGRATION-BATCH-030
Dependent member no ejecutará tras dependency failure.

## DB-MIGRATION-BATCH-031
Blocked member será distinto de failed member.

## DB-MIGRATION-BATCH-032
Not-started member será distinto de failed member.

## DB-MIGRATION-BATCH-033
Batch no implicará atomicity.

## DB-MIGRATION-BATCH-034
Atomic batch requerirá evidencia.

## DB-MIGRATION-BATCH-035
Atomicity será first-class classification.

## DB-MIGRATION-BATCH-036
Batch Executor delegará a Migration Executor.

## DB-MIGRATION-BATCH-037
Batch Executor no será segundo SQL runner.

## DB-MIGRATION-BATCH-038
Batch Execution ID será distinto de Batch ID.

## DB-MIGRATION-BATCH-039
Múltiples attempts podrán pertenecer al mismo batch.

## DB-MIGRATION-BATCH-040
Batch durable state será distinto de attempt state.

## DB-MIGRATION-BATCH-041
Member states serán first-class.

## DB-MIGRATION-BATCH-042
Partial batch será first-class.

## DB-MIGRATION-BATCH-043
Partial batch requerirá durable successful effects.

## DB-MIGRATION-BATCH-044
Unknown batch será first-class.

## DB-MIGRATION-BATCH-045
Unknown member podrá elevar batch certainty a UNKNOWN.

## DB-MIGRATION-BATCH-046
Majority success no definirá batch success.

## DB-MIGRATION-BATCH-047
Scheduling respetará dependency graph.

## DB-MIGRATION-BATCH-048
Parallel execution será opt-in.

## DB-MIGRATION-BATCH-049
Sequential execution será default.

## DB-MIGRATION-BATCH-050
Coordination lock será batch-scoped por default.

## DB-MIGRATION-BATCH-051
Batch lock no será transaction.

## DB-MIGRATION-BATCH-052
Batch lock ownership será explícito.

## DB-MIGRATION-BATCH-053
Lock loss detendrá ejecución segura.

## DB-MIGRATION-BATCH-054
Batch repository record podrá existir antes de effects.

## DB-MIGRATION-BATCH-055
PLANNED/RUNNING no significará APPLIED.

## DB-MIGRATION-BATCH-056
Member será APPLIED solo tras execution contract.

## DB-MIGRATION-BATCH-057
Batch será APPLIED solo tras required member contracts.

## DB-MIGRATION-BATCH-058
Repository failure después de effects será recovery state.

## DB-MIGRATION-BATCH-059
Repository no será única fuente de physical truth.

## DB-MIGRATION-BATCH-060
Migration Application ID podrá distinguir reapplications.

## DB-MIGRATION-BATCH-061
Migration ID no será sobrecargada como application occurrence.

## DB-MIGRATION-BATCH-062
Una migration podrá pertenecer históricamente a múltiples batches.

## DB-MIGRATION-BATCH-063
Current application state será distinto de history.

## DB-MIGRATION-BATCH-064
Crash durante batch será recoverable/reconcilable.

## DB-MIGRATION-BATCH-065
Resume será distinto de restart.

## DB-MIGRATION-BATCH-066
Resume cargará member state.

## DB-MIGRATION-BATCH-067
Resume verificará definition fingerprints.

## DB-MIGRATION-BATCH-068
Resume verificará target.

## DB-MIGRATION-BATCH-069
Resume verificará repository coherence.

## DB-MIGRATION-BATCH-070
Unknown member será reconciliado antes de retry.

## DB-MIGRATION-BATCH-071
Resume no reaplicará successful member ciegamente.

## DB-MIGRATION-BATCH-072
Definition drift bloqueará resume.

## DB-MIGRATION-BATCH-073
Capability drift requerirá revalidation.

## DB-MIGRATION-BATCH-074
New migration no será añadida a persisted partial batch.

## DB-MIGRATION-BATCH-075
Persisted membership será immutable.

## DB-MIGRATION-BATCH-076
New pending migration pertenecerá a future batch.

## DB-MIGRATION-BATCH-077
Batch success implicará all required members success.

## DB-MIGRATION-BATCH-078
Batch rollback utilizará Rollback System.

## DB-MIGRATION-BATCH-079
Batch rollback no será reverse foreach.

## DB-MIGRATION-BATCH-080
Batch rollback considerará current applied state.

## DB-MIGRATION-BATCH-081
Batch rollback no asumirá atomicity.

## DB-MIGRATION-BATCH-082
Partial batch rollback será first-class.

## DB-MIGRATION-BATCH-083
Rollback no eliminará batch history.

## DB-MIGRATION-BATCH-084
Batch será distinto de deployment.

## DB-MIGRATION-BATCH-085
Deployment correlation será metadata.

## DB-MIGRATION-BATCH-086
Package membership será distinta de batch membership.

## DB-MIGRATION-BATCH-087
Package boundaries no crearán batches implícitamente.

## DB-MIGRATION-BATCH-088
Batch partitioning custom será explícito.

## DB-MIGRATION-BATCH-089
Default será un logical batch por target/invocation.

## DB-MIGRATION-BATCH-090
Multi-target execution preferirá one batch per target.

## DB-MIGRATION-BATCH-091
Batch Group será distinto de Batch.

## DB-MIGRATION-BATCH-092
Batch Group no implicará distributed atomicity.

## DB-MIGRATION-BATCH-093
No se inventará hidden 2PC.

## DB-MIGRATION-BATCH-094
Partial Batch Group será first-class.

## DB-MIGRATION-BATCH-095
Concurrent migrators serán coordinados.

## DB-MIGRATION-BATCH-096
Repository freshness será revalidada tras lock.

## DB-MIGRATION-BATCH-097
Stale batch plan bloqueará execution por default.

## DB-MIGRATION-BATCH-098
Speculative planning podrá hacerse fuera del lock.

## DB-MIGRATION-BATCH-099
Speculative plan deberá revalidarse.

## DB-MIGRATION-BATCH-100
Sequence allocation será concurrency-safe.

## DB-MIGRATION-BATCH-101
Sequence gaps serán válidos.

## DB-MIGRATION-BATCH-102
Failed sequence no será reciclada automáticamente.

## DB-MIGRATION-BATCH-103
Batch target será immutable.

## DB-MIGRATION-BATCH-104
Batch labels no serán identity.

## DB-MIGRATION-BATCH-105
Application state será distinta de rollback state.

## DB-MIGRATION-BATCH-106
Display status podrá ser derived.

## DB-MIGRATION-BATCH-107
Attempt history será preservada.

## DB-MIGRATION-BATCH-108
Successful resume no borrará previous failed attempt.

## DB-MIGRATION-BATCH-109
Batch result será immutable.

## DB-MIGRATION-BATCH-110
Member result preservará outcome certainty.

## DB-MIGRATION-BATCH-111
Failure report será structured.

## DB-MIGRATION-BATCH-112
Recovery descriptor será first-class.

## DB-MIGRATION-BATCH-113
Resume eligibility será evidence-based.

## DB-MIGRATION-BATCH-114
Cancellation se propagará al member activo.

## DB-MIGRATION-BATCH-115
Cancellation será distinta de application state.

## DB-MIGRATION-BATCH-116
Cancelled batch podrá ser PARTIALLY_APPLIED.

## DB-MIGRATION-BATCH-117
Batch deadline será propagado.

## DB-MIGRATION-BATCH-118
Migration deadline no excederá remaining batch deadline.

## DB-MIGRATION-BATCH-119
Expired deadline impedirá iniciar nuevo member.

## DB-MIGRATION-BATCH-120
Timeout será distinto de effects state.

## DB-MIGRATION-BATCH-121
Batch retry será reconciliation/resume, no blind restart.

## DB-MIGRATION-BATCH-122
Member retry será delegado a Migration Execution.

## DB-MIGRATION-BATCH-123
Batch retry tendrá budget.

## DB-MIGRATION-BATCH-124
Budget exhaustion preservará partial state.

## DB-MIGRATION-BATCH-125
Events no podrán cambiar membership.

## DB-MIGRATION-BATCH-126
Events no podrán forge success.

## DB-MIGRATION-BATCH-127
Telemetry será correlacionable.

## DB-MIGRATION-BATCH-128
Batch execution será privileged.

## DB-MIGRATION-BATCH-129
Batch plan tampering será detectable.

## DB-MIGRATION-BATCH-130
Audit preservará batch identity y membership.

## DB-MIGRATION-BATCH-131
Runtime batch state será operation-scoped.

## DB-MIGRATION-BATCH-132
No habrá static current batch.

## DB-MIGRATION-BATCH-133
FrankenPHP no conservará batch state entre executions.

## DB-MIGRATION-BATCH-134
RoadRunner no conservará batch state entre jobs.

## DB-MIGRATION-BATCH-135
OpenSwoole aislará batch state por coroutine.

## DB-MIGRATION-BATCH-136
Extension registry será frozen.

## DB-MIGRATION-BATCH-137
Extensions no modificarán persisted membership.

## DB-MIGRATION-BATCH-138
Extensions no omitirán required dependencies.

## DB-MIGRATION-BATCH-139
Extensions no falsificarán repository state.

## DB-MIGRATION-BATCH-140
Empty pending set no creará durable batch por default.

## DB-MIGRATION-BATCH-141
No-op no consumirá sequence por default.

## DB-MIGRATION-BATCH-142
Concurrent execution será testeada.

## DB-MIGRATION-BATCH-143
Crash recovery será testeado.

## DB-MIGRATION-BATCH-144
Definition drift será testeado.

## DB-MIGRATION-BATCH-145
Unknown member outcome será testeado.

## DB-MIGRATION-BATCH-146
Resume semantics serán testeadas.

## DB-MIGRATION-BATCH-147
Partial rollback será testeado.

## DB-MIGRATION-BATCH-148
Multi-target partial result será testeado.

## DB-MIGRATION-BATCH-149
Batch state será explainable y auditable.

## DB-MIGRATION-BATCH-150
VoltStack nunca declarará un batch aplicado mientras exista un required member cuyo execution contract no haya sido satisfecho.

---

# 187. Anti-patterns

## 187.1 Batch = MAX + 1 sin coordinación

Incorrecto:

```php
$batch = $repository->maxBatch() + 1;
```

si múltiples migrators pueden competir.

---

## 187.2 Batch = transaction

Incorrecto:

```text
same batch
⇒
same transaction
```

---

## 187.3 Batch membership = dependencies

Incorrecto:

```text
M1 and M2 are in B17
⇒
M2 depends on M1
```

---

## 187.4 Retry entire batch

Incorrecto:

```php
foreach ($batch as $migration) {
    $migration->run();
}
```

otra vez después de partial failure.

---

## 187.5 Add new migrations to partial batch

Incorrecto:

```text
B17 partial
new M100 discovered
→ append M100 to B17
```

---

## 187.6 Delete failed batch

Incorrecto:

```text
batch failed
→ delete history
```

---

## 187.7 Rollback foreach reverse

Incorrecto:

```php
foreach (array_reverse($batch) as $migration) {
    $migration->down();
}
```

sin dependency/reversibility analysis.

---

## 187.8 Assume cancelled = nothing happened

Incorrecto:

```text
cancelled
⇒
batch not applied
```

---

## 187.9 Global static batch

Incorrecto:

```php
MigrationBatch::$current = 17;
```

---

## 187.10 Multi-database batch as atomic

Incorrecto:

```text
Database A + Database B
same BatchId
⇒
atomic
```

---

# 188. Ejemplo completo

Pending:

```text
M10 create customers
M20 add customers.email
M30 create customers.email index
```

Dependencies:

```text
M10 → M20 → M30
```

Planner:

```text
MigrationPlan P100
```

Batch:

```text
BatchId       B17
BatchSequence 17
Target        primary
Members:
  1 M10
  2 M20
  3 M30
```

Execution:

```text
Acquire lock
      ↓
Verify repository fingerprint
      ↓
Record B17 RUNNING
      ↓
Execute M10
      ↓
Record M10 applied
      ↓
Execute M20
      ↓
Record M20 applied
      ↓
Execute M30
      ↓
Record M30 applied
      ↓
Finalize B17
      ↓
Release lock
```

Result:

```text
B17 = APPLIED
```

---

# 189. Ejemplo de batch parcial

```text
B18
├── M40 ✓
├── M50 ✗
└── M60 not started
```

Repository:

```text
B18 = PARTIALLY_APPLIED

M40 = APPLIED
M50 = FAILED
M60 = NOT_STARTED
```

Recovery:

```text
Resume candidate:
M50

M40:
do not rerun
```

---

# 190. Ejemplo de unknown member

```text
M50
COMMIT sent
connection lost
```

Estado:

```text
M40 = APPLIED
M50 = UNKNOWN
M60 = BLOCKED
```

Batch:

```text
REQUIRES_RECOVERY
certainty = UNKNOWN
```

Antes de resume:

```text
Reconcile M50
```

---

# 191. Ejemplo de reapplication

```text
M42
```

History:

```text
B10 / A100 → APPLY SUCCESS
R11        → ROLLBACK SUCCESS
B15 / A200 → APPLY SUCCESS
```

Current:

```text
M42 = APPLIED
currentApplication = A200
```

Esto preserva history sin sobrecargar `MigrationId`.

---

# 192. Fórmula de membership

```text
BatchMembers(B)
=
OrderedSet(
    MigrationPlan.SelectedMigrations
)
```

una vez persistido:

```text
Immutable(BatchMembers(B))
```

---

# 193. Fórmula de batch success

```text
BatchApplied(B)
=
∀m ∈ RequiredMembers(B):
    MigrationExecutionContractSatisfied(m)
```

---

# 194. Fórmula de partial batch

```text
BatchPartiallyApplied(B)
=
∃m ∈ Members(B): DurableApplied(m)
∧
∃n ∈ RequiredMembers(B): ¬DurableApplied(n)
```

---

# 195. Fórmula de resume

```text
SafeBatchResume(B)
=
MembershipUnchanged(B)
∧
DefinitionsValid(B)
∧
RepositoryReconciled(B)
∧
NoBlockingUnknownOutcome(B)
∧
CurrentStateCompatible(B)
∧
RemainingMembersSafeToExecute(B)
```

---

# 196. Fórmula de atomicidad

```text
AtomicBatch(B)
=
∃T:
    ∀Effect e ∈ B,
    e ∈ T
∧
Commit(T) is the single durable visibility boundary
```

Si esto no puede probarse:

```text
AtomicBatch(B) = false/unknown
```

según evidencia.

---

# 197. Fórmula de consistencia del repository

```text
RepositoryBatchApplied(B)
⇒
∀m ∈ RequiredMembers(B):
    RepositoryMigrationApplied(m)
```

y:

```text
RepositoryMigrationApplied(m)
⇒
MigrationExecutionContractSatisfied(m)
```

---

# 198. Fórmula maestra

```text
Migration Batch System
=
Stable Batch Identity
+
Human-Friendly Sequence
+
Immutable Membership
+
Dependency-Aware Ordering
+
Batch Planning
+
Coordination Locking
+
Migration Execution Delegation
+
Per-Member State
+
Durable Batch History
+
Outcome Certainty
+
Partial Batch Modeling
+
Reconciliation
+
Safe Resume
+
Rollback Coordination
+
Multi-Target Grouping
+
Persistent Runtime Isolation
```

---

# 199. Regla arquitectónica maestra

> **VoltStack utilizará los batches para registrar y coordinar qué migraciones participaron en una misma operación lógica de aplicación, sin convertir esa agrupación en una falsa garantía de dependencia, transaccionalidad o atomicidad.**

La relación correcta será:

```text
Pending Migrations
        ↓
Migration Plan
        ↓
Batch Plan
        ↓
Immutable Membership
        ↓
Coordination Lock
        ↓
Per-Migration Execution
        ↓
Per-Member Durable State
        ↓
Batch Finalization
        ↓
Historical Repository Record
```

---

# 200. Resultado arquitectónico

Con este diseño VoltStack podrá soportar:

```text
simple sequential migration batches
package/application mixed batches
crash-safe batch history
partial batch detection
resume after failure
unknown-outcome reconciliation
batch rollback
re-application history
concurrent migrator protection
multi-database batch groups
persistent worker runtimes
```

sin reducir el concepto de batch al modelo:

```text
batch = MAX(batch) + 1
```

ni asumir:

```text
same batch
=
same transaction
```

El Batch System preserva cinco propiedades fundamentales:

```text
Identity
Membership
Traceability
Recoverability
Outcome Certainty
```

---

# 201. Relación con los siguientes documentos

El Batch System será utilizado por:

```text
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

pero mantiene límites claros:

```text
Batch System
≠
Schema Diff Migration Generator

Batch System
≠
Zero-Downtime Planner

Batch System
≠
Migration Safety Engine
```

---

# 202. Siguiente documento

```text
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
```

El siguiente documento deberá conectar formalmente:

```text
Schema Snapshot
        ↓
Schema Diff
        ↓
Migration Candidate
        ↓
Human/Policy Review
        ↓
Migration Definition / Migration Plan
```

y establecer especialmente:

```text
Schema Diff
≠
Migration

Difference
≠
Migration Operation

Detected Rename
≠
Proven Rename

Generated Migration
≠
Automatically Safe Migration

Schema Equality
≠
Data Migration Completeness

Diff-to-Migration
≠
Direct DDL Execution
```

incluyendo:

```text
diff interpretation
change classification
rename evidence
migration candidate generation
operation synthesis
dependency synthesis
data migration requirements
destructive change handling
platform compatibility
migration safety integration
human review
deterministic generation
generated migration fingerprints
round-trip verification
drift handling
extension model
```