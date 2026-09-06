# 105_DATABASE_MIGRATION_PLANNER_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Planner System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 105 — Database Migration Planner System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Planner System` define la arquitectura responsable de transformar el conjunto de migraciones disponibles, el historial persistente, las dependencias, las capacidades de la plataforma y las políticas operacionales en un **plan de migración explícito, ordenado, validable, determinista e immutable**.

El Planner responde:

> **¿Qué migraciones deben ejecutarse, en qué orden, mediante qué fases y bajo qué requisitos, barreras, transacciones, locks y precondiciones?**

Formalmente:

```text
MigrationPlan =
Plan(
    MigrationCatalog,
    MigrationDefinitions,
    RepositorySnapshot,
    DependencyGraph,
    PlatformCapabilities,
    Policies,
    Target
)
```

El Planner **no ejecuta** el resultado.

---

# 2. Principio central

> **El Migration Planner convierte intención de evolución e historial conocido en una estrategia operacional explícita; nunca cambia la base de datos.**

La separación fundamental será:

```text
Migration Definition
        │
        ▼
   describes intent

Migration Repository
        │
        ▼
    records history

Migration Planner
        │
        ▼
   decides strategy

Migration Executor
        │
        ▼
    performs strategy
```

---

# 3. Regla maestra

```text
Planning ≠ Execution
```

Y además:

```text
Migration Planner
≠
Migration Executor
≠
Migration Repository
≠
Migration Discovery
≠
Schema Planner
≠
Schema Compiler
≠
Safety Analyzer
```

---

# 4. Posición arquitectónica

```text
Migration Sources
      │
      ▼
Migration Discovery
      │
      ▼
Migration Catalog
      │
      ├──────────────────────┐
      │                      │
      ▼                      ▼
Definitions           Repository Snapshot
      │                      │
      └──────────┬───────────┘
                 │
                 ▼
        Repository Reconciliation
                 │
                 ▼
          Dependency Analysis
                 │
                 ▼
        Compatibility Analysis
                 │
                 ▼
           Safety Analysis
                 │
                 ▼
         MIGRATION PLANNER
                 │
                 ▼
           MigrationPlan
                 │
                 ▼
        Migration Execution
```

---

# 5. Responsabilidades

El Planner deberá:

- seleccionar migraciones pendientes;
- resolver el migration target;
- validar dependencias;
- construir dependency closure;
- ordenar migraciones;
- detectar ciclos;
- transformar definiciones en operaciones planificables;
- coordinar Schema y Data operations;
- definir fases;
- definir barriers;
- establecer preconditions;
- establecer postconditions;
- incorporar requisitos de seguridad;
- incorporar requisitos de compatibilidad;
- incorporar requisitos de locking;
- definir transaction boundaries;
- determinar retry boundaries;
- modelar operaciones no transaccionales;
- modelar backfills;
- modelar zero-downtime phases;
- producir dependency graphs;
- producir un plan immutable;
- producir diagnostics;
- producir fingerprint determinista;
- detectar inputs inconsistentes;
- permitir preview/dry-run;
- mantenerse puro respecto de ejecución.

---

# 6. No responsabilidades

El Planner no deberá:

```text
open database connections
execute SQL
compile SQL
begin transactions
commit transactions
rollback transactions
acquire live locks
sleep/retry
discover migration files
modify repository records
introspect database implicitly
run backfills
authorize migrations
```

---

# 7. Entradas

El Planner recibirá un contexto explícito:

```php
final readonly class MigrationPlanningInput
{
    public function __construct(
        public MigrationCatalog $catalog,
        public MigrationDefinitionSet $definitions,
        public MigrationRepositorySnapshot $repository,
        public MigrationReconciliationReport $reconciliation,
        public MigrationDependencyGraph $dependencies,
        public PlatformCapabilitySnapshot $capabilities,
        public MigrationTarget $target,
        public MigrationPlanningPolicy $policy,
    ) {}
}
```

No deberá recuperar estos datos mediante estado global.

---

# 8. Planner contract

```php
interface MigrationPlanner
{
    public function plan(
        MigrationPlanningInput $input
    ): MigrationPlan;
}
```

La operación deberá aproximarse a una función pura:

```text
MigrationPlan =
f(ExplicitPlanningInput)
```

---

# 9. Determinismo

Para inputs equivalentes:

```text
Plan(I) = Plan(I)
```

El resultado deberá ser semánticamente idéntico.

No dependerá de:

```text
wall-clock time
random IDs
filesystem iteration order
global tenant
current connection
process ID
worker ID
```

salvo valores explícitos incluidos en el input.

---

# 10. MigrationPlan

Se propone:

```php
final readonly class MigrationPlan
{
    public function __construct(
        public MigrationPlanId $id,
        public MigrationTarget $target,
        public MigrationPlanPhaseSet $phases,
        public MigrationPlanDependencyGraph $dependencies,
        public MigrationPlanRequirementSet $requirements,
        public MigrationPlanDiagnosticBag $diagnostics,
        public MigrationPlanFingerprint $fingerprint,
        public MigrationPlanMetadata $metadata,
    ) {}
}
```

---

# 11. Plan immutable

Una vez creado:

```text
MigrationPlan
```

no podrá modificarse.

Cualquier modificación produce:

```text
MigrationPlan'
```

nuevo.

---

# 12. Plan ID ≠ fingerprint

Debe mantenerse:

```text
MigrationPlanId
≠
MigrationPlanFingerprint
```

El ID identifica una instancia lógica.

El fingerprint representa contenido estructural/semántico.

---

# 13. Plan ≠ execution instance

```text
MigrationPlan
=
immutable strategy
```

mientras:

```text
MigrationExecution
=
mutable runtime state
```

---

# 14. Pipeline de planning

```text
Planning Input
      │
      ▼
Input Validation
      │
      ▼
Repository Validation
      │
      ▼
Target Resolution
      │
      ▼
Pending Selection
      │
      ▼
Dependency Closure
      │
      ▼
Dependency Validation
      │
      ▼
Topological Ordering
      │
      ▼
Definition Expansion
      │
      ▼
Operation Classification
      │
      ▼
Compatibility Integration
      │
      ▼
Safety Integration
      │
      ▼
Strategy Resolution
      │
      ▼
Phase Construction
      │
      ▼
Transaction Planning
      │
      ▼
Lock Planning
      │
      ▼
Retry Planning
      │
      ▼
Barrier Construction
      │
      ▼
Pre/Postcondition Construction
      │
      ▼
Dependency Graph Finalization
      │
      ▼
Validation
      │
      ▼
Canonicalization
      │
      ▼
Fingerprint
      │
      ▼
Seal MigrationPlan
```

---

# 15. Pending migration selection

El Planner deberá comparar:

```text
MigrationCatalog
```

contra:

```text
MigrationRepositorySnapshot
```

para determinar:

```text
PendingMigrationSet
```

---

# 16. Pending definition

En términos simples:

```text
Pending =
Available
-
Applied
```

pero solo después de considerar:

```text
qualified identity
namespace
target
checksum drift
repository reconciliation
dependency validity
```

---

# 17. Pending ≠ safe to execute

Una migration puede ser:

```text
PENDING
```

y al mismo tiempo:

```text
BLOCKED
```

por incompatibilidad o dependencia.

---

# 18. Migration selection modes

Se propone:

```text
ALL_PENDING
UP_TO
ONLY
THROUGH_BATCH
DEPENDENCY_CLOSURE
PACKAGE_SCOPE
NAMESPACE_SCOPE
CUSTOM_SELECTION
```

---

# 19. Explicit target

Ejemplo:

```text
migrate up to M50
```

El Planner deberá calcular:

```text
DependencyClosure(M50)
```

No simplemente ejecutar todas las migrations con timestamp menor.

---

# 20. Dependency closure

Formalmente:

```text
Closure(M) =
M ∪ Dependencies(M) ∪ Dependencies(Dependencies(M)) ...
```

hasta alcanzar el cierre transitivo.

---

# 21. Already-applied dependencies

Una dependencia ya aplicada satisface el requirement solo si su estado es válido.

```text
APPLIED_AND_MATCHING
```

puede satisfacerla.

```text
APPLIED_BUT_DRIFTED
```

podrá bloquearla según policy.

---

# 22. Missing dependency

Si:

```text
M2 depends on M1
```

pero M1 no existe ni está registrado de forma válida:

```text
MissingMigrationDependencyException
```

---

# 23. Dependency graph

Se utilizará:

```text
MigrationDependencyGraph
```

con:

```text
Node = MigrationIdentity
Edge A → B = B depends on A
```

---

# 24. Topological ordering

El orden deberá derivarse mediante:

```text
TopologicalSort(MigrationDependencyGraph)
```

y no únicamente por filenames.

---

# 25. Timestamp order

Los timestamps podrán actuar como:

```text
default deterministic ordering hint
```

entre migrations independientes.

No como dependency semantics.

---

# 26. Stable ordering

Para dos nodos independientes:

```text
A
B
```

se usará un tie-breaker determinista, por ejemplo:

```text
declared sequence
timestamp
qualified identity
```

según convention.

---

# 27. Cycle detection

Si:

```text
A → B
B → C
C → A
```

el Planner deberá producir:

```text
MigrationDependencyCycleException
```

con cycle path.

---

# 28. Cycle diagnostic

Ejemplo:

```text
Migration dependency cycle detected:

app:M10
  → package.billing:M20
  → app:M30
  → app:M10
```

---

# 29. Explicit dependency

Preferir:

```php
public function dependencies(): array
{
    return [
        CreateUsers::class,
    ];
}
```

sobre inferencia mágica.

---

# 30. Implicit dependency inference

Podrá existir como análisis auxiliar, pero:

```text
inferred dependency
≠
declared dependency
```

y deberá conservar provenance/confidence.

---

# 31. Definition expansion

Cada `MigrationDefinition` puede contener:

```text
Schema Operations
Data Operations
Validation Operations
Backfill Operations
Custom Operations
```

El Planner las transforma en:

```text
MigrationPlanOperation
```

---

# 32. MigrationPlanOperation

```php
interface MigrationPlanOperation
{
    public function id(): MigrationPlanOperationId;

    public function migration(): MigrationIdentity;

    public function kind(): MigrationPlanOperationKind;
}
```

---

# 33. Operation kinds

```text
SCHEMA
DATA
BACKFILL
VALIDATION
BARRIER
CUSTOM
REPOSITORY
```

Las operaciones del repository podrán ser generadas por orchestration final, pero deben quedar explícitas.

---

# 34. Operation identity

```text
MigrationPlanOperationId
≠
MigrationIdentity
≠
SchemaNodeId
≠
ExecutionUnitId
```

---

# 35. Schema operation

Una migration puede declarar:

```php
$table->string('email');
```

y terminar representada como:

```text
MigrationDefinition
      ↓
Schema AST
      ↓
Schema Planner
      ↓
Planned Schema Operations
      ↓
MigrationPlanOperation
```

---

# 36. Migration Planner vs Schema Planner

Distinción crítica:

```text
Migration Planner
=
plans migration lifecycle
```

```text
Schema Planner
=
plans structural schema strategy
```

El primero puede delegar al segundo.

---

# 37. Ejemplo SQLite

Una migration solicita:

```text
DROP COLUMN legacy
```

El Migration Planner no deberá inventar SQL.

El Schema Planner puede decidir:

```text
Create replacement table
Copy data
Drop old table
Rename replacement
Recreate indexes
```

El Migration Planner incorpora esa estrategia al lifecycle global.

---

# 38. Nested planning

Arquitectura:

```text
Migration Planner
       │
       ├── Schema Planner
       │
       ├── Data Operation Planner
       │
       ├── Backfill Planner
       │
       ├── Safety Analyzer
       │
       └── Compatibility Analyzer
       │
       ▼
Unified MigrationPlan
```

---

# 39. Planner composition

Evitar un `MigrationPlanner` monolítico.

Se propone:

```text
MigrationPlanner
├── SelectionPlanner
├── DependencyPlanner
├── OperationPlanner
├── PhasePlanner
├── TransactionPlanner
├── LockPlanner
├── RetryPlanner
├── BarrierPlanner
├── ConditionPlanner
└── PlanValidator
```

---

# 40. Strategy resolution

Una operación lógica puede tener varias estrategias.

Ejemplo:

```text
Add NOT NULL column
```

puede resolverse como:

```text
Direct
```

o:

```text
Add nullable
→ backfill
→ validate
→ enforce NOT NULL
```

según:

```text
platform
table characteristics
policy
zero-downtime profile
```

---

# 41. Planner strategy ≠ compiler syntax

El Planner decide:

```text
what operational strategy
```

El Compiler decide:

```text
how each planned structural operation is represented in platform SQL
```

---

# 42. No hidden compiler emulation

Regla:

> Si una operación requiere una estrategia multi-step, esa estrategia deberá estar visible en el plan antes de llegar al Schema Compiler.

Incorrecto:

```text
Plan:
DROP COLUMN

Compiler:
secretly rebuild entire table
```

Correcto:

```text
Plan:
CreateReplacementTable
CopyData
DropOldTable
RenameReplacementTable
RecreateIndexes
```

---

# 43. Migration phases

El plan se dividirá en:

```text
MigrationPlanPhase
```

---

# 44. Phase examples

```text
PRECHECK
PREPARE
EXPAND
BACKFILL
VALIDATE
SWITCH
CONTRACT
FINALIZE
REPOSITORY_COMMIT
```

No todas las migrations requieren todas.

---

# 45. Phase ≠ migration

Una migration puede abarcar múltiples fases.

Asimismo una fase operacional puede contener operaciones de varias migrations si la policy lo permite.

---

# 46. Conservative default

Por defecto se recomienda mantener boundaries claras por migration:

```text
Migration A
  phases...

Migration B
  phases...
```

La fusión cross-migration deberá requerir una optimización explícita y demostrablemente segura.

---

# 47. Phase contract

```php
final readonly class MigrationPlanPhase
{
    public function __construct(
        public MigrationPlanPhaseId $id,
        public MigrationPlanPhaseKind $kind,
        public MigrationPlanOperationSet $operations,
        public MigrationPlanRequirementSet $requirements,
        public MigrationBarrierSet $barriers,
        public MigrationConditionSet $preconditions,
        public MigrationConditionSet $postconditions,
    ) {}
}
```

---

# 48. Phase dependencies

```text
Phase B depends on Phase A
```

deberá representarse explícitamente.

---

# 49. Execution unit

Dentro de una phase:

```text
MigrationExecutionUnit
```

representará una unidad ejecutable.

---

# 50. Execution unit ≠ operation

Una execution unit puede contener varias operaciones cuando comparten:

```text
transaction boundary
lock requirement
atomicity contract
retry contract
```

---

# 51. ExecutionUnit model

```php
final readonly class MigrationExecutionUnit
{
    public function __construct(
        public MigrationExecutionUnitId $id,
        public MigrationPlanOperationSet $operations,
        public MigrationTransactionRequirement $transaction,
        public MigrationLockRequirementSet $locks,
        public MigrationRetryContract $retry,
        public MigrationConditionSet $preconditions,
        public MigrationConditionSet $postconditions,
    ) {}
}
```

---

# 52. Transaction planning

El Planner deberá determinar:

```text
where transactions may begin/end
```

sin iniciarlas.

---

# 53. Transaction requirement

Se propone:

```text
TRANSACTION_REQUIRED
TRANSACTION_PREFERRED
TRANSACTION_ALLOWED
TRANSACTION_FORBIDDEN
IMPLICIT_COMMIT_POSSIBLE
PLATFORM_DEPENDENT
```

---

# 54. Transaction capability

Debe considerar:

```text
PlatformCapabilitySnapshot
```

No asumir:

```text
DDL is transactional
```

universalmente.

---

# 55. Migration transaction ≠ batch

Una migration puede necesitar:

```text
Transaction 1
NonTransactional Operation
Transaction 2
```

dentro del mismo batch.

---

# 56. Atomic migration

Una migration podrá declarar:

```text
REQUIRE_ATOMIC
```

Si la plataforma no puede satisfacerlo:

```text
MigrationAtomicityUnsatisfiedException
```

---

# 57. Best-effort atomicity

Otra policy:

```text
BEST_EFFORT_ATOMIC
```

permitirá fragmentación explícita con diagnostics.

---

# 58. Non-atomic plan

Debe marcarse:

```text
NON_ATOMIC
```

No fingir atomicidad.

---

# 59. Lock planning

El Planner describe:

```text
MigrationLockRequirement
```

No adquiere locks.

---

# 60. Lock categories

```text
MIGRATION_COORDINATION
SCHEMA_METADATA
TABLE
ROW_RANGE
APPLICATION_BARRIER
CUSTOM
```

---

# 61. Coordination lock

Normalmente el plan requerirá:

```text
MIGRATION_COORDINATION
```

para impedir migradores concurrentes sobre el mismo target.

---

# 62. Physical DB lock prediction

Locks internos que la base pueda tomar son:

```text
estimates / expectations
```

no garantías absolutas.

---

# 63. Lock impact

Puede clasificarse:

```text
NONE
LOW
MODERATE
HIGH
EXCLUSIVE
UNKNOWN
```

---

# 64. Lock requirement ≠ lock impact

```text
Requirement
=
what VoltStack must acquire/control
```

```text
Impact
=
what execution may cause in DB
```

---

# 65. Barrier

Un barrier impide avanzar hasta cumplir una condición.

```text
MigrationBarrier
```

---

# 66. Barrier examples

```text
BackfillCompletedBarrier
ReplicaCaughtUpBarrier
ApplicationVersionBarrier
ValidationPassedBarrier
ManualApprovalBarrier
TrafficDrainBarrier
```

---

# 67. Barrier ≠ sleep

Incorrecto:

```php
sleep(30);
```

como semántica de coordinación.

Correcto:

```text
Barrier:
wait until required condition is satisfied
```

La espera real pertenece al Executor/orchestrator.

---

# 68. Preconditions

Antes de una operation pueden requerirse:

```text
table exists
column absent
column nullable
no NULL values
repository fingerprint unchanged
minimum application version deployed
capability available
```

---

# 69. Preconditions ≠ assertions only

Una precondition es parte del contrato del plan.

Si falla:

```text
operation must not execute
```

salvo recovery policy explícita.

---

# 70. Postconditions

Ejemplos:

```text
column exists
constraint validated
backfill complete
repository record inserted
```

---

# 71. Physical verification

Algunas postconditions podrán requerir introspection/runtime queries.

El Planner solo las describe.

Executor/verification subsystem las evalúa.

---

# 72. Validation operations

Una migration puede necesitar:

```text
ValidateNoNullValues
ValidateUniqueValues
ValidateForeignKeyConsistency
```

antes de un cambio estructural.

---

# 73. DataValidationRequirement

Del Schema Diff/analysis puede llegar:

```text
DataValidationRequirement
```

El Migration Planner lo convierte en una operación/condition explícita.

---

# 74. No row queries during planning

El Planner no deberá ejecutar:

```sql
SELECT COUNT(*)
```

para verificar datos.

Debe producir:

```text
RuntimeValidationOperation
```

---

# 75. Backfill

Un backfill será first-class.

```text
BackfillOperation
≠
generic migration callback
```

---

# 76. Backfill model

```php
final readonly class MigrationBackfillOperation
{
    public function __construct(
        public BackfillId $id,
        public BackfillSource $source,
        public BackfillMutation $mutation,
        public BackfillChunkingPolicy $chunking,
        public BackfillRetryPolicy $retry,
        public BackfillProgressPolicy $progress,
    ) {}
}
```

---

# 77. Large backfill

No planificar:

```text
UPDATE huge_table SET ...
```

como única estrategia por defecto en perfiles zero-downtime.

Podrá transformarse en:

```text
Chunk 1
Chunk 2
...
Chunk N
```

como runtime strategy.

---

# 78. Planning vs runtime chunk count

Si el número de filas no se conoce sin I/O:

```text
Planner
```

no deberá inventar N.

En su lugar genera:

```text
DynamicChunkedBackfillPlan
```

---

# 79. Dynamic operation

El plan puede contener una estrategia runtime parametrizada sin contener cada fila/chunk concreto.

---

# 80. Backfill progress

Podrá requerir:

```text
checkpointing
```

para recovery.

El Planner define requirement; Executor mantiene progress state.

---

# 81. Retry planning

Debe distinguirse:

```text
Retryable Failure
≠
Safe Retry
```

heredando el principio del Query Execution Engine.

---

# 82. Retry contract

Cada execution unit podrá clasificarse:

```text
SAFE_TO_RETRY
CONDITIONALLY_RETRYABLE
NOT_RETRYABLE
UNKNOWN
```

---

# 83. Retry safety

Depende de:

```text
operation semantics
idempotency
transaction outcome
repository state
partial effects
replayability
platform behavior
```

---

# 84. Schema operation retry

Ejemplo:

```text
CREATE TABLE users
```

no deberá considerarse automáticamente retry-safe solo porque exista:

```text
IF NOT EXISTS
```

porque el objeto existente podría no corresponder a la definición esperada.

---

# 85. Retry after unknown outcome

Debe requerir:

```text
reconciliation / verification
```

antes de reintentar cuando el resultado previo sea incierto.

---

# 86. Retry boundary

Una unidad retryable debe tener límites claros:

```text
retry entire unit
```

No reintentar arbitrariamente una subsección desconocida.

---

# 87. Idempotency

Debe modelarse explícitamente:

```text
IDEMPOTENT
CONDITIONALLY_IDEMPOTENT
NON_IDEMPOTENT
UNKNOWN
```

---

# 88. Idempotency ≠ retry safety

Aunque:

```text
operation is idempotent
```

otros factores pueden hacer inseguro el retry.

---

# 89. Zero-downtime planning

El Planner deberá poder operar bajo:

```text
MigrationPlanningProfile::ZERO_DOWNTIME
```

---

# 90. Expand/contract

Patrón:

```text
EXPAND
   ↓
DEPLOY COMPATIBLE APP
   ↓
BACKFILL
   ↓
VALIDATE
   ↓
SWITCH
   ↓
DEPLOY NEW APP
   ↓
CONTRACT
```

---

# 91. Application-version barriers

Ejemplo:

Antes de eliminar columna `legacy_name`:

```text
ApplicationVersionBarrier
```

puede exigir que ninguna versión activa dependa de ella.

---

# 92. Migration Planner and deployment orchestration

El Planner puede describir:

```text
external deployment barrier
```

pero no despliega la aplicación.

---

# 93. Online index

Si PostgreSQL soporta estrategia concurrente:

```text
CREATE INDEX CONCURRENTLY
```

el Planner puede seleccionar:

```text
ConcurrentIndexStrategy
```

basándose en capabilities/policy.

El Schema Compiler renderiza la sintaxis.

---

# 94. Transaction consequence

Si una estrategia concurrente no puede ejecutarse dentro de transaction:

```text
TransactionRequirement = FORBIDDEN
```

deberá quedar explícito.

---

# 95. Compatibility input

El Planner consume resultados del:

```text
Database Schema Platform Compatibility System
```

y/o migration compatibility analysis.

---

# 96. Compatibility states

Ejemplo:

```text
SUPPORTED
SUPPORTED_WITH_STRATEGY
SUPPORTED_WITH_LOSS
UNSUPPORTED
UNKNOWN
```

---

# 97. Unsupported migration

Si no existe estrategia válida:

```text
UnsupportedMigrationPlanException
```

---

# 98. Supported with loss

Nunca degradar silenciosamente.

Debe requerir:

```text
explicit policy acceptance
```

cuando la transformación sea potencialmente lossy.

---

# 99. Safety integration

El Planner consume:

```text
MigrationSafetyAssessment
```

No sustituye al Safety System.

---

# 100. Safety levels

Ejemplo:

```text
SAFE
REQUIRES_VALIDATION
REQUIRES_APPROVAL
POTENTIALLY_DESTRUCTIVE
DESTRUCTIVE
UNKNOWN
```

---

# 101. Safety ≠ authorization

Una operación:

```text
SAFE
```

no significa:

```text
AUTHORIZED
```

La autorización pertenece a otra capa.

---

# 102. Destructive operation

El plan deberá conservar explícitamente:

```text
DROP TABLE
DROP COLUMN
TRUNCATING TYPE CHANGE
```

como destructivos.

No esconderlos dentro de un generic operation.

---

# 103. Manual approval

Una policy puede convertir:

```text
DESTRUCTIVE
```

en requirement:

```text
ManualApprovalBarrier
```

---

# 104. Approval token

El Planner no genera aprobación.

Solo declara:

```text
ApprovalRequirement
```

---

# 105. Migration requirements

Se propone:

```text
MigrationPlanRequirement
├── CapabilityRequirement
├── TransactionRequirement
├── LockRequirement
├── DataValidationRequirement
├── ApprovalRequirement
├── ApplicationVersionRequirement
├── RepositoryStateRequirement
├── SchemaStateRequirement
├── ResourceRequirement
└── ExtensionRequirement
```

---

# 106. Requirement state

Cada requirement podrá tener:

```text
SATISFIED_AT_PLAN_TIME
REQUIRES_RUNTIME_CHECK
REQUIRES_EXTERNAL_ACTION
UNSATISFIED
UNKNOWN
```

---

# 107. Planning-time capability

Ejemplo:

```text
supportsTransactionalDDL = true
```

puede satisfacerse al planear porque proviene del capability snapshot.

---

# 108. Runtime requirement

Ejemplo:

```text
table contains zero NULL values
```

requiere runtime verification.

---

# 109. External requirement

Ejemplo:

```text
application release >= 2026.09.2
```

puede depender de deployment system.

---

# 110. Repository precondition

El plan deberá capturar:

```text
ExpectedRepositoryFingerprint
```

o una representación equivalente.

---

# 111. Stale plan detection

Antes de ejecutar:

```text
CurrentRepositoryFingerprint
```

deberá compararse con el esperado si la policy lo requiere.

---

# 112. Repository changed

Si:

```text
Rcurrent ≠ Rplanned
```

el Executor no deberá continuar ciegamente.

Resultado:

```text
STALE_PLAN
```

---

# 113. Replanning

La capa de orchestration podrá:

```text
re-read repository
reconcile
re-plan
```

El Planner por sí mismo no ejecuta ese ciclo.

---

# 114. Schema precondition

También podrá almacenarse:

```text
ExpectedSchemaFingerprint
```

cuando el plan dependa de un snapshot físico concreto.

---

# 115. Schema changed

Si la base cambia fuera de migrations entre planning y execution:

```text
ExpectedSchemaFingerprint ≠ ActualSchemaFingerprint
```

podrá bloquearse.

---

# 116. Optimistic plan validation

Esto actúa como:

```text
optimistic concurrency control
```

sobre planning assumptions.

---

# 117. Plan dependency graph

No solo existen dependencies entre migrations.

También entre operations:

```text
MigrationPlanDependencyGraph
```

---

# 118. Operation graph

Ejemplo:

```text
AddNullableColumn
      ↓
BackfillColumn
      ↓
ValidateNoNull
      ↓
SetNotNull
```

---

# 119. Edge types

Se propone:

```text
REQUIRES_COMPLETION
REQUIRES_SUCCESS
REQUIRES_VALIDATION
REQUIRES_BARRIER
REQUIRES_TRANSACTION_BOUNDARY
REQUIRES_LOCK_RELEASE
```

---

# 120. DAG preference

El plan deberá ser un DAG cuando sea posible.

Los loops runtime, como chunk processing, se representan como una operación parametrizada, no como cycle del plan.

---

# 121. Plan cycle

Un cycle operacional no resoluble será:

```text
MigrationPlanCycleException
```

---

# 122. Plan canonicalization

Antes del fingerprint:

```text
canonical operation order
canonical dependency order
canonical requirement order
canonical diagnostics
```

---

# 123. Canonical order ≠ execution reorder freedom

El orden canonical sirve para representación.

Execution order se deriva del plan/dependencies.

---

# 124. Plan fingerprint

Se propone:

```text
MigrationPlanFingerprint =
H(
    selected migrations,
    migration semantic checksums,
    repository fingerprint,
    target,
    dependency graph,
    planned operations,
    platform capability fingerprint,
    planning policy fingerprint,
    planner version,
    extension fingerprint
)
```

---

# 125. Fingerprint exclusions

No incluir:

```text
object memory address
worker ID
random UUID
wall clock
temporary runtime handle
```

---

# 126. Fingerprint use

Puede servir para:

```text
audit
preview/execution correlation
stale plan detection
approval binding
testing
telemetry
```

---

# 127. Approval binding

Una aprobación avanzada podría autorizar:

```text
PlanFingerprint P
```

en vez de una vaga instrucción:

```text
approve migrations
```

Esto evita que el plan cambie después de aprobarse.

---

# 128. Plan serialization

Debe ser:

```text
versioned
deterministic
typed
portable where meaningful
```

---

# 129. Serialized plan ≠ executable authority

Un plan serializado no deberá considerarse automáticamente autorizado para ejecución.

Debe volver a validarse:

```text
version
fingerprint
target
repository state
capabilities
policy
authorization
```

---

# 130. Planning version

Se propone:

```text
MigrationPlannerVersion
```

para proteger fingerprints y serialization.

---

# 131. Extensions

El Planner deberá permitir:

```text
MigrationPlanningExtension
```

---

# 132. Extension examples

```text
custom operation planner
custom requirement provider
custom barrier planner
custom backfill planner
custom strategy selector
```

---

# 133. Extension registry

Debe ser:

```text
frozen after bootstrap
```

en persistent runtimes.

---

# 134. No last-wins

Dos extensions reclamando el mismo operation kind sin prioridad/contract explícito deberán causar conflicto.

---

# 135. Extension purity

Una planning extension no deberá:

```text
execute SQL
open connection
mutate repository
acquire lock
perform deployment
```

---

# 136. Explicit external facts

Si una extension necesita información externa, deberá recibir:

```text
PlanningFactSnapshot
```

precalculado.

No realizar I/O oculto.

---

# 137. Planning facts

Ejemplos:

```text
table size estimate
replica topology
deployment version
maintenance window
traffic class
```

---

# 138. Fact provenance

Cada fact debería poder conservar:

```text
source
capturedAt
certainty
freshness
```

cuando afecte decisiones críticas.

---

# 139. Fact ≠ guarantee

Un estimate:

```text
table has ~100M rows
```

no es verdad eterna.

Debe tratarse como:

```text
planning estimate
```

---

# 140. Cost-aware planning

El Planner podrá utilizar hints como:

```text
estimated table size
estimated lock duration
backfill cost class
```

para elegir estrategias.

Pero:

```text
cost preference
```

nunca puede invalidar correctness.

---

# 141. Resource planning

Puede declarar:

```text
MigrationResourceRequirement
```

como:

```text
LOW_IO
HIGH_IO
LOW_MEMORY
BACKGROUND_CAPABLE
EXCLUSIVE_WINDOW
```

---

# 142. Resource governance

El Executor/runtime decide cómo imponer budgets.

Planner solo declara requirements/hints.

---

# 143. Maintenance window

Puede existir:

```text
MaintenanceWindowRequirement
```

para operaciones de alto impacto.

---

# 144. Scheduling ≠ planning

El Planner no espera a que llegue la ventana.

Solo indica que la operation requiere una.

---

# 145. Cancellation semantics

El plan podrá clasificar execution units:

```text
CANCELLABLE
CANCELLABLE_AT_CHECKPOINT
NON_CANCELLABLE
PLATFORM_DEPENDENT
```

---

# 146. Cancellation ≠ rollback

Detener una operation no implica revertir efectos ya aplicados.

---

# 147. Compensating operation

Una migration podrá declarar:

```text
CompensationPlan
```

cuando rollback real no sea posible.

---

# 148. Compensation ≠ inverse

```text
Compensation
≠
mathematical inverse
```

Puede restaurar una condición operacional aceptable sin reconstruir exactamente el estado anterior.

---

# 149. Forward recovery

Para producción se recomienda soportar:

```text
FORWARD_RECOVERY
```

como estrategia first-class.

---

# 150. Rollback planning

Aunque el documento específico 107 profundizará en rollback, el Planner deberá ser capaz de generar:

```text
UP plan
DOWN plan
RECOVERY plan
```

bajo contratos separados.

---

# 151. Direction

```text
MigrationDirection
├── UP
├── DOWN
└── RECOVERY
```

---

# 152. Down ≠ reverse(up)

Regla crítica:

```text
PlanDown(M)
≠
Reverse(PlanUp(M))
```

necesariamente.

Debe construirse usando definición/policy específica.

---

# 153. Data loss

Si rollback implica pérdida irreversible:

```text
IRREVERSIBLE
```

debe quedar explícito.

---

# 154. Irreversible migration

Una migration puede declararse:

```text
UP_ONLY
```

o producir:

```text
RollbackUnavailableRequirement
```

---

# 155. Migration target

Debe ser first-class:

```php
final readonly class MigrationTarget
{
    public function __construct(
        public DatabaseTargetIdentity $database,
        public ?SchemaNamespace $schema,
        public ?MigrationNamespace $migrationNamespace,
    ) {}
}
```

---

# 156. Target ≠ connection

El plan contiene identidad lógica.

No:

```text
PDO
ConnectionLease
socket
```

---

# 157. Multi-database migration

Una migration que afecta varias bases introduce distributed semantics.

Core deberá ser conservador.

---

# 158. Cross-database atomicity

No asumir:

```text
transaction across DB A + DB B
```

salvo capability explícita.

---

# 159. Distributed plan

Podrá existir en extensiones futuras:

```text
DistributedMigrationPlan
```

pero no deberá contaminar el caso normal.

---

# 160. Multitenancy

El core no dependerá de Multitenancy.

Integration package podrá producir:

```text
MigrationTargetSet
```

por tenant.

---

# 161. Tenant planning isolation

Cada tenant deberá obtener:

```text
RepositorySnapshot
SchemaSnapshot
MigrationPlan
```

aislados.

---

# 162. No global current tenant

Prohibido:

```php
MigrationPlanner::$tenant;
```

---

# 163. Package migrations

El Planner respetará namespaces:

```text
app
package.billing
package.auth
```

y sus dependencies.

---

# 164. Cross-package dependencies

Podrán declararse:

```text
package.billing:M20
depends on
package.core:M5
```

de forma explícita.

---

# 165. Package removal

Si una migration aplicada pertenece a package ausente:

```text
MISSING_DEFINITION
```

No planear rollback automático.

---

# 166. Plan diagnostics

El Planner deberá producir diagnostics estructurados:

```text
INFO
WARNING
ERROR
BLOCKING
```

---

# 167. Diagnostic example

```text
Migration:
app:2026_09_06_add_email_not_null

Issue:
Column is being changed from nullable to non-nullable.

Requirement:
Runtime validation must prove zero NULL rows.

Strategy:
VALIDATE → SET NOT NULL
```

---

# 168. Explainability

Toda decisión estratégica importante debería poder responder:

```text
Why was this strategy selected?
```

---

# 169. Strategy explanation

Ejemplo:

```text
Strategy: ConcurrentIndexCreation

Reason:
- ZERO_DOWNTIME profile active
- platform supports concurrent index creation
- operation marked online-preferred
- transaction requirement allows non-transactional phase
```

---

# 170. Plan preview

Debe poder generarse sin ejecución:

```text
MigrationPlanPreview
```

---

# 171. Preview contents

Puede mostrar:

```text
pending migrations
order
phases
operations
destructive changes
transactions
locks
barriers
runtime validations
backfills
retry classifications
estimated impact
```

---

# 172. Preview ≠ SQL dump

Aunque tooling pueda incluir SQL compilado posteriormente:

```text
Migration Plan Preview
≠
SQL dump
```

---

# 173. Dry run

Un dry-run completo puede ser:

```text
Plan
→ Compile
→ Validate
→ Render Preview
```

sin ejecutar.

---

# 174. Plan validation

Antes de sealing:

```text
MigrationPlanValidator
```

deberá verificar invariants.

---

# 175. Validation examples

```text
all dependencies resolved
no graph cycles
all operation IDs unique
all phases reachable
all required capabilities satisfied or deferred explicitly
transaction constraints compatible
barriers well-formed
no operation orphaned
repository assumptions present
```

---

# 176. Unsatisfiable transaction plan

Ejemplo:

```text
Unit requires:
TRANSACTION_REQUIRED

Operation requires:
TRANSACTION_FORBIDDEN
```

Resultado:

```text
UnsatisfiableMigrationTransactionPlanException
```

---

# 177. Unsatisfiable barrier

Una barrier que depende de una postcondition imposible debe detectarse cuando sea demostrable estáticamente.

---

# 178. Plan completeness

Se propone:

```text
COMPLETE
CONDITIONALLY_COMPLETE
INCOMPLETE
UNKNOWN
```

---

# 179. Complete

Todo lo necesario para ejecutar está planificado, aunque existan runtime checks normales.

---

# 180. Conditionally complete

El plan depende de:

```text
external approval
runtime validation
deployment barrier
```

pero sabe qué hacer si se satisfacen.

---

# 181. Incomplete

Falta estrategia para una operación.

No debe ejecutarse.

---

# 182. Unknown

No puede determinarse seguridad/completitud con la información disponible.

Debe fallar de manera conservadora según policy.

---

# 183. Budget system

El planning deberá tener budgets explícitos:

```text
max migrations
max dependency edges
max operations
max phases
max requirements
max diagnostics
max strategy candidates
max expression depth
max serialized plan bytes
```

---

# 184. Budget exhaustion

No deberá producir silenciosamente un plan truncado.

Resultado:

```text
MigrationPlanningBudgetExceededException
```

---

# 185. Cancellation

Planning prolongado deberá poder recibir:

```text
CancellationToken
```

si el framework lo soporta.

---

# 186. Planner session

Estado temporal podrá residir en:

```text
MigrationPlanningSession
```

operation-scoped.

---

# 187. Shared planner

El servicio:

```text
DefaultMigrationPlanner
```

podrá ser shared si es immutable/stateless.

---

# 188. Persistent runtime

Con FrankenPHP:

```text
shared immutable planner
+
request/command scoped PlanningSession
```

---

# 189. RoadRunner/OpenSwoole

Mismo principio:

```text
no mutable plan state survives execution context
```

---

# 190. Thread/coroutine safety

Registries compartidos:

```text
immutable/frozen
```

Sessions:

```text
isolated
```

---

# 191. Caching

Un plan puede ser cacheable si todos sus inputs tienen fingerprints estables.

---

# 192. Plan cache key

Conceptualmente:

```text
H(
    catalogFingerprint,
    definitionFingerprint,
    repositoryFingerprint,
    schemaFingerprint?,
    capabilitiesFingerprint,
    targetFingerprint,
    policyFingerprint,
    plannerVersion,
    extensionFingerprint
)
```

---

# 193. Cached plan validation

Un cached plan nunca deberá ejecutarse sin validar assumptions relevantes.

---

# 194. Live resources

Un cached plan no contendrá:

```text
connection
statement
transaction
lock handle
cursor
stream
```

---

# 195. Security

El Planner trabaja con operaciones privilegiadas.

Debe impedir que:

```text
untrusted extension
```

inyecte una operation que omita safety/capability contracts.

---

# 196. Trust level

Puede existir:

```text
FRAMEWORK
APPLICATION
TRUSTED_EXTENSION
UNTRUSTED_EXTERNAL
```

para provenance.

Trust no elimina validación.

---

# 197. Raw migration operation

Un escape hatch podrá existir:

```text
RawMigrationOperation
```

pero deberá marcarse:

```text
NON_PORTABLE
OPAQUE
SECURITY_SENSITIVE
ANALYSIS_LIMITED
```

---

# 198. Raw SQL

Si una migration incluye SQL raw:

```text
Planner
```

no deberá fingir comprender sus efectos completos.

Puede requerir:

```text
explicit declared effects
explicit transaction requirements
explicit retry contract
explicit safety classification
```

---

# 199. Opaque operation

Por defecto:

```text
RetrySafety = UNKNOWN
Impact = UNKNOWN
Portability = NON_PORTABLE
```

hasta obtener contratos explícitos.

---

# 200. No arbitrary callback planning

Evitar migraciones representadas únicamente como:

```php
function () {
    // anything
}
```

porque destruyen capacidad de:

```text
preview
analysis
safety
planning
retry reasoning
zero-downtime strategy
```

---

# 201. Controlled imperative escape hatch

Podrá mantenerse para compatibilidad, pero deberá ser:

```text
ExplicitOpaqueMigrationOperation
```

y no la ruta arquitectónica preferida.

---

# 202. Telemetry

Planning puede emitir:

```text
migration.plan.started
migration.plan.completed
migration.plan.failed
migration.plan.blocked
migration.plan.strategy_selected
```

---

# 203. Metrics

Ejemplos:

```text
migration.plan.duration
migration.plan.migrations
migration.plan.operations
migration.plan.phases
migration.plan.destructive_operations
migration.plan.runtime_validations
migration.plan.backfills
migration.plan.barriers
```

---

# 204. Telemetry privacy

No incluir por defecto:

```text
raw SQL
full schema
sensitive data predicates
credentials
```

---

# 205. Testing

Debe incluir:

```text
selection tests
dependency closure tests
topological sorting
cycle detection
stable ordering
repository drift handling
schema strategy delegation
transaction planning
lock planning
barrier planning
backfill planning
retry planning
zero-downtime planning
raw operation behavior
fingerprinting
serialization
extension conflicts
budgets
persistent runtime isolation
```

---

# 206. Pure planning tests

La mayoría deberá ejecutarse sin base real:

```php
$plan = $planner->plan($input);
```

y verificar objetos tipados.

---

# 207. Determinism test

```text
Plan(Input) fingerprint
=
Plan(Input) fingerprint
```

en ejecuciones repetidas.

---

# 208. Ordering test

Distinto orden de iteración de source collections no deberá cambiar el plan canónico.

---

# 209. Capability tests

La misma migration sobre:

```text
PostgreSQL
MySQL
MariaDB
SQLite
```

podrá generar estrategias diferentes pero válidas.

---

# 210. No vendor branching in migration definition

No se desea:

```php
if ($database === 'sqlite') {
    ...
}
```

disperso en migrations.

Preferir:

```text
intent
→ capability analysis
→ planning strategy
```

---

# 211. Zero-downtime test

Cambio:

```text
nullable → NOT NULL
```

bajo zero-downtime deberá poder producir:

```text
Validate
→ Enforce
```

o estrategia expand/backfill/validate según contexto.

---

# 212. Stale repository test

Plan generado con:

```text
RepositoryFingerprint A
```

deberá ser rechazable si runtime encuentra:

```text
RepositoryFingerprint B
```

---

# 213. Crash recovery planning

El Planner deberá poder recibir reconciliation state:

```text
PARTIALLY_APPLIED
UNKNOWN
```

y no tratar la migration como pending normal.

---

# 214. Recovery planning mode

Se propone:

```text
MigrationPlanningMode
├── NORMAL
├── ROLLBACK
├── RECOVERY
├── PREVIEW
└── VALIDATE_ONLY
```

---

# 215. Normal mode

No deberá intentar resolver automáticamente uncertain previous executions.

---

# 216. Recovery mode

Podrá construir:

```text
verification
reconciliation
forward repair
compensation
```

según contratos disponibles.

---

# 217. Error hierarchy

```text
DatabaseMigrationPlannerException
├── InvalidMigrationPlanningInputException
├── MigrationTargetResolutionException
├── MigrationSelectionException
├── MissingMigrationDependencyException
├── MigrationDependencyCycleException
├── MigrationDependencyConflictException
├── MigrationRepositoryStateException
├── MigrationDefinitionDriftException
├── MigrationOperationPlanningException
├── MigrationStrategyResolutionException
├── UnsupportedMigrationPlanException
├── MigrationCompatibilityPlanningException
├── MigrationSafetyPlanningException
├── MigrationTransactionPlanningException
│   ├── MigrationAtomicityUnsatisfiedException
│   └── UnsatisfiableMigrationTransactionPlanException
├── MigrationLockPlanningException
├── MigrationBarrierPlanningException
├── MigrationBackfillPlanningException
├── MigrationRetryPlanningException
├── MigrationConditionPlanningException
├── MigrationPlanCycleException
├── MigrationPlanValidationException
├── MigrationPlanIncompleteException
├── MigrationPlanSerializationException
├── MigrationPlanningExtensionException
├── MigrationPlanningBudgetExceededException
├── MigrationPlanningCancelledException
└── MigrationPlanningInvariantException
```

---

# 218. Namespace

```text
VoltStack\Quantum\Database\Migration\Planner
```

---

# 219. Estructura propuesta

```text
Migration/
└── Planner/
    ├── Contract/
    │   ├── MigrationPlanner.php
    │   ├── MigrationPlanValidator.php
    │   ├── MigrationStrategyPlanner.php
    │   └── MigrationOperationPlanner.php
    │
    ├── Core/
    │   ├── DefaultMigrationPlanner.php
    │   ├── MigrationPlanningInput.php
    │   ├── MigrationPlanningContext.php
    │   ├── MigrationPlanningSession.php
    │   ├── MigrationPlanningMode.php
    │   └── MigrationPlanningProfile.php
    │
    ├── Plan/
    │   ├── MigrationPlan.php
    │   ├── MigrationPlanId.php
    │   ├── MigrationPlanMetadata.php
    │   ├── MigrationPlanCompleteness.php
    │   └── MigrationPlanFingerprint.php
    │
    ├── Selection/
    │   ├── MigrationSelectionPlanner.php
    │   ├── MigrationSelection.php
    │   ├── MigrationSelectionMode.php
    │   └── PendingMigrationSet.php
    │
    ├── Dependency/
    │   ├── MigrationDependencyPlanner.php
    │   ├── MigrationDependencyGraph.php
    │   ├── MigrationDependencyEdge.php
    │   ├── MigrationDependencyClosure.php
    │   └── MigrationTopologicalSorter.php
    │
    ├── Operation/
    │   ├── MigrationPlanOperation.php
    │   ├── MigrationPlanOperationId.php
    │   ├── MigrationPlanOperationKind.php
    │   ├── SchemaMigrationPlanOperation.php
    │   ├── DataMigrationPlanOperation.php
    │   ├── ValidationMigrationPlanOperation.php
    │   ├── BackfillMigrationPlanOperation.php
    │   └── RawMigrationPlanOperation.php
    │
    ├── Phase/
    │   ├── MigrationPlanPhase.php
    │   ├── MigrationPlanPhaseId.php
    │   ├── MigrationPlanPhaseKind.php
    │   └── MigrationPlanPhaseSet.php
    │
    ├── ExecutionUnit/
    │   ├── MigrationExecutionUnit.php
    │   ├── MigrationExecutionUnitId.php
    │   └── MigrationExecutionUnitSet.php
    │
    ├── Strategy/
    │   ├── MigrationStrategy.php
    │   ├── MigrationStrategySelector.php
    │   ├── MigrationStrategyCandidate.php
    │   ├── MigrationStrategyReason.php
    │   └── MigrationStrategyRegistry.php
    │
    ├── Transaction/
    │   ├── MigrationTransactionPlanner.php
    │   ├── MigrationTransactionRequirement.php
    │   ├── MigrationAtomicityRequirement.php
    │   └── MigrationTransactionBoundary.php
    │
    ├── Lock/
    │   ├── MigrationLockPlanner.php
    │   ├── MigrationLockRequirement.php
    │   ├── MigrationLockRequirementSet.php
    │   └── MigrationLockImpact.php
    │
    ├── Barrier/
    │   ├── MigrationBarrier.php
    │   ├── MigrationBarrierPlanner.php
    │   ├── BackfillCompletedBarrier.php
    │   ├── ValidationPassedBarrier.php
    │   ├── ApplicationVersionBarrier.php
    │   ├── ManualApprovalBarrier.php
    │   └── TrafficDrainBarrier.php
    │
    ├── Condition/
    │   ├── MigrationCondition.php
    │   ├── MigrationConditionSet.php
    │   ├── MigrationPrecondition.php
    │   └── MigrationPostcondition.php
    │
    ├── Backfill/
    │   ├── MigrationBackfillPlanner.php
    │   ├── MigrationBackfillOperation.php
    │   ├── BackfillChunkingPolicy.php
    │   ├── BackfillProgressPolicy.php
    │   └── DynamicChunkedBackfillPlan.php
    │
    ├── Retry/
    │   ├── MigrationRetryPlanner.php
    │   ├── MigrationRetryContract.php
    │   ├── MigrationRetrySafety.php
    │   └── MigrationIdempotency.php
    │
    ├── Requirement/
    │   ├── MigrationPlanRequirement.php
    │   ├── MigrationPlanRequirementSet.php
    │   ├── CapabilityRequirement.php
    │   ├── DataValidationRequirement.php
    │   ├── ApprovalRequirement.php
    │   ├── ApplicationVersionRequirement.php
    │   ├── RepositoryStateRequirement.php
    │   ├── SchemaStateRequirement.php
    │   └── ResourceRequirement.php
    │
    ├── ZeroDowntime/
    │   ├── ZeroDowntimeMigrationPlanner.php
    │   ├── ExpandContractStrategy.php
    │   └── OnlineMigrationStrategy.php
    │
    ├── Recovery/
    │   ├── MigrationRecoveryPlanner.php
    │   ├── MigrationCompensationPlan.php
    │   └── MigrationForwardRecoveryPlan.php
    │
    ├── Fact/
    │   ├── MigrationPlanningFact.php
    │   ├── MigrationPlanningFactSnapshot.php
    │   └── MigrationPlanningFactProvenance.php
    │
    ├── Validation/
    │   ├── DefaultMigrationPlanValidator.php
    │   └── MigrationPlanValidationReport.php
    │
    ├── Fingerprint/
    │   └── MigrationPlanFingerprinter.php
    │
    ├── Serialization/
    │   ├── MigrationPlanSerializer.php
    │   └── MigrationPlanDeserializer.php
    │
    ├── Extension/
    │   ├── MigrationPlanningExtension.php
    │   └── MigrationPlanningExtensionRegistry.php
    │
    ├── Budget/
    │   └── MigrationPlanningBudget.php
    │
    ├── Diagnostic/
    │   ├── MigrationPlanningDiagnostic.php
    │   └── MigrationPlanningDiagnosticBag.php
    │
    └── Exception/
        └── ...
```

---

# 220. Architectural invariants

## DB-MIGRATION-PLANNER-001
Planning será distinto de Execution.

## DB-MIGRATION-PLANNER-002
Migration Planner será distinto de Schema Planner.

## DB-MIGRATION-PLANNER-003
Migration Planner será distinto de Migration Repository.

## DB-MIGRATION-PLANNER-004
Migration Planner será distinto de Discovery.

## DB-MIGRATION-PLANNER-005
Migration Planner será distinto de Schema Compiler.

## DB-MIGRATION-PLANNER-006
Planner no abrirá conexiones.

## DB-MIGRATION-PLANNER-007
Planner no ejecutará SQL.

## DB-MIGRATION-PLANNER-008
Planner no iniciará transactions.

## DB-MIGRATION-PLANNER-009
Planner no hará commit.

## DB-MIGRATION-PLANNER-010
Planner no hará rollback.

## DB-MIGRATION-PLANNER-011
Planner no adquirirá live locks.

## DB-MIGRATION-PLANNER-012
Planner no modificará repository.

## DB-MIGRATION-PLANNER-013
Planner no hará hidden introspection.

## DB-MIGRATION-PLANNER-014
Planning inputs serán explícitos.

## DB-MIGRATION-PLANNER-015
MigrationPlan será immutable.

## DB-MIGRATION-PLANNER-016
MigrationPlan será distinto de MigrationExecution.

## DB-MIGRATION-PLANNER-017
Plan ID será distinto de Plan Fingerprint.

## DB-MIGRATION-PLANNER-018
Planning será determinista.

## DB-MIGRATION-PLANNER-019
Filesystem order no determinará plan semantics.

## DB-MIGRATION-PLANNER-020
Pending será calculado usando qualified identities.

## DB-MIGRATION-PLANNER-021
Pending no implicará safe.

## DB-MIGRATION-PLANNER-022
Target selection será explícita.

## DB-MIGRATION-PLANNER-023
Dependency closure será transitiva.

## DB-MIGRATION-PLANNER-024
Applied dependency deberá estar en estado aceptable.

## DB-MIGRATION-PLANNER-025
Missing dependency será error typed.

## DB-MIGRATION-PLANNER-026
Dependencies formarán graph explícito.

## DB-MIGRATION-PLANNER-027
Ordering se basará en dependencies.

## DB-MIGRATION-PLANNER-028
Timestamp será tie-breaker/convention, no dependency truth.

## DB-MIGRATION-PLANNER-029
Independent ordering será deterministic.

## DB-MIGRATION-PLANNER-030
Dependency cycles serán rechazados.

## DB-MIGRATION-PLANNER-031
Cycle diagnostics mostrarán path cuando sea posible.

## DB-MIGRATION-PLANNER-032
Declared dependency será preferida sobre inference.

## DB-MIGRATION-PLANNER-033
Inferred dependency conservará provenance.

## DB-MIGRATION-PLANNER-034
MigrationPlanOperation será typed.

## DB-MIGRATION-PLANNER-035
Operation ID será distinto de MigrationIdentity.

## DB-MIGRATION-PLANNER-036
Schema operation planning podrá delegarse al Schema Planner.

## DB-MIGRATION-PLANNER-037
Migration Planner no generará SQL.

## DB-MIGRATION-PLANNER-038
Multi-step schema emulation deberá estar visible en plan.

## DB-MIGRATION-PLANNER-039
Compiler no deberá inventar hidden migration strategy.

## DB-MIGRATION-PLANNER-040
Migration phases serán explicitables.

## DB-MIGRATION-PLANNER-041
Phase será distinta de Migration.

## DB-MIGRATION-PLANNER-042
ExecutionUnit será distinta de Operation.

## DB-MIGRATION-PLANNER-043
Transaction boundaries serán planificadas, no ejecutadas.

## DB-MIGRATION-PLANNER-044
Transactional DDL no se asumirá universalmente.

## DB-MIGRATION-PLANNER-045
Batch será distinto de Transaction.

## DB-MIGRATION-PLANNER-046
Atomicity requirements serán explícitos.

## DB-MIGRATION-PLANNER-047
Atomicity imposible producirá error o explicit degradation policy.

## DB-MIGRATION-PLANNER-048
Non-atomic plan será identificado.

## DB-MIGRATION-PLANNER-049
Lock requirements serán explícitos.

## DB-MIGRATION-PLANNER-050
Planner no adquirirá locks.

## DB-MIGRATION-PLANNER-051
Lock requirement será distinto de lock impact.

## DB-MIGRATION-PLANNER-052
Barrier será distinto de sleep.

## DB-MIGRATION-PLANNER-053
Barrier conditions serán explícitas.

## DB-MIGRATION-PLANNER-054
Preconditions formarán parte del plan contract.

## DB-MIGRATION-PLANNER-055
Failed precondition bloqueará execution salvo recovery contract.

## DB-MIGRATION-PLANNER-056
Postconditions serán explícitas cuando sean relevantes.

## DB-MIGRATION-PLANNER-057
Planner no ejecutará runtime validation queries.

## DB-MIGRATION-PLANNER-058
Data validation será first-class.

## DB-MIGRATION-PLANNER-059
Backfill será first-class.

## DB-MIGRATION-PLANNER-060
Backfill no será generic opaque callback por defecto.

## DB-MIGRATION-PLANNER-061
Unknown row count no generará fake static chunks.

## DB-MIGRATION-PLANNER-062
Dynamic backfill strategy podrá existir.

## DB-MIGRATION-PLANNER-063
Backfill progress requirement será explícito.

## DB-MIGRATION-PLANNER-064
Retryable Failure será distinto de Safe Retry.

## DB-MIGRATION-PLANNER-065
Retry contract será explícito por execution unit.

## DB-MIGRATION-PLANNER-066
Idempotency será distinta de retry safety.

## DB-MIGRATION-PLANNER-067
Unknown outcome requerirá verification/reconciliation antes de unsafe retry.

## DB-MIGRATION-PLANNER-068
Retry boundaries serán explícitos.

## DB-MIGRATION-PLANNER-069
ZERO_DOWNTIME será planning profile first-class.

## DB-MIGRATION-PLANNER-070
Expand/contract será strategy, no hidden behavior.

## DB-MIGRATION-PLANNER-071
Deployment barrier podrá ser descrita pero no ejecutada por Planner.

## DB-MIGRATION-PLANNER-072
Online schema strategies serán capability-driven.

## DB-MIGRATION-PLANNER-073
Transaction consequences de online strategy serán explícitas.

## DB-MIGRATION-PLANNER-074
Compatibility analysis será consumido, no reinventado arbitrariamente.

## DB-MIGRATION-PLANNER-075
Unsupported operations no tendrán silent fallback.

## DB-MIGRATION-PLANNER-076
Lossy strategy requerirá explicit policy.

## DB-MIGRATION-PLANNER-077
Safety assessment será consumido.

## DB-MIGRATION-PLANNER-078
Safety será distinta de authorization.

## DB-MIGRATION-PLANNER-079
Destructive operations permanecerán visibles.

## DB-MIGRATION-PLANNER-080
Manual approval será explicit requirement/barrier.

## DB-MIGRATION-PLANNER-081
Planner no generará approval authority.

## DB-MIGRATION-PLANNER-082
Requirements tendrán typed state.

## DB-MIGRATION-PLANNER-083
Runtime requirements no serán considerados satisfechos durante planning sin evidencia.

## DB-MIGRATION-PLANNER-084
External requirements permanecerán explícitos.

## DB-MIGRATION-PLANNER-085
Repository assumptions serán capturables.

## DB-MIGRATION-PLANNER-086
Stale repository plan será detectable.

## DB-MIGRATION-PLANNER-087
Planner no ejecutará automatic replanning loop.

## DB-MIGRATION-PLANNER-088
Schema assumptions podrán capturarse mediante fingerprint.

## DB-MIGRATION-PLANNER-089
Schema drift between plan/execution será detectable cuando el plan dependa de snapshot.

## DB-MIGRATION-PLANNER-090
Plan dependency graph incluirá operation dependencies.

## DB-MIGRATION-PLANNER-091
Runtime loops no crearán graph cycles artificiales.

## DB-MIGRATION-PLANNER-092
Unresolvable plan cycles serán rechazados.

## DB-MIGRATION-PLANNER-093
Plan canonicalization será deterministic.

## DB-MIGRATION-PLANNER-094
Canonical order será distinto de arbitrary execution reorder.

## DB-MIGRATION-PLANNER-095
Plan fingerprint será versionado.

## DB-MIGRATION-PLANNER-096
Plan fingerprint incluirá relevant input fingerprints.

## DB-MIGRATION-PLANNER-097
Plan fingerprint excluirá live resource identities.

## DB-MIGRATION-PLANNER-098
Approval podrá vincularse a plan fingerprint.

## DB-MIGRATION-PLANNER-099
Plan serialization será versionada.

## DB-MIGRATION-PLANNER-100
Serialized plan no implicará authorization.

## DB-MIGRATION-PLANNER-101
Extensions serán registradas mediante registry frozen.

## DB-MIGRATION-PLANNER-102
Extension conflicts no usarán silent last-wins.

## DB-MIGRATION-PLANNER-103
Planning extensions no ejecutarán I/O oculto.

## DB-MIGRATION-PLANNER-104
External planning facts serán snapshots explícitos.

## DB-MIGRATION-PLANNER-105
Planning facts conservarán provenance cuando sea relevante.

## DB-MIGRATION-PLANNER-106
Estimate será distinto de guarantee.

## DB-MIGRATION-PLANNER-107
Cost preference no podrá romper correctness.

## DB-MIGRATION-PLANNER-108
Resource requirements podrán ser declarados.

## DB-MIGRATION-PLANNER-109
Planner no impondrá runtime resource limits directamente.

## DB-MIGRATION-PLANNER-110
Maintenance window será requirement, no scheduling behavior.

## DB-MIGRATION-PLANNER-111
Cancellation semantics serán distinguibles de rollback.

## DB-MIGRATION-PLANNER-112
Compensation será distinta de inverse rollback.

## DB-MIGRATION-PLANNER-113
Forward recovery será first-class.

## DB-MIGRATION-PLANNER-114
UP, DOWN y RECOVERY serán directions distintas.

## DB-MIGRATION-PLANNER-115
DOWN no será simplemente reverse(UP).

## DB-MIGRATION-PLANNER-116
Irreversibility será explícita.

## DB-MIGRATION-PLANNER-117
MigrationTarget será first-class.

## DB-MIGRATION-PLANNER-118
MigrationTarget no contendrá live connection.

## DB-MIGRATION-PLANNER-119
Cross-database atomicity no será asumida.

## DB-MIGRATION-PLANNER-120
Multitenancy será integración opcional.

## DB-MIGRATION-PLANNER-121
Tenant plans estarán aislados.

## DB-MIGRATION-PLANNER-122
No habrá mutable global current tenant.

## DB-MIGRATION-PLANNER-123
Package namespaces serán respetados.

## DB-MIGRATION-PLANNER-124
Cross-package dependencies serán explícitas.

## DB-MIGRATION-PLANNER-125
Missing package definition no causará automatic rollback.

## DB-MIGRATION-PLANNER-126
Planning diagnostics serán structured.

## DB-MIGRATION-PLANNER-127
Strategy decisions serán explainable.

## DB-MIGRATION-PLANNER-128
Plan preview será posible sin execution.

## DB-MIGRATION-PLANNER-129
Plan preview será distinto de SQL dump.

## DB-MIGRATION-PLANNER-130
Dry run podrá incluir compilation sin execution.

## DB-MIGRATION-PLANNER-131
Plan deberá validarse antes de sealing.

## DB-MIGRATION-PLANNER-132
Unsatisfiable transaction requirements serán error.

## DB-MIGRATION-PLANNER-133
Plan completeness será first-class.

## DB-MIGRATION-PLANNER-134
Incomplete plan no será ejecutable.

## DB-MIGRATION-PLANNER-135
Unknown planning state no se degradará silenciosamente a safe.

## DB-MIGRATION-PLANNER-136
Planning tendrá budgets.

## DB-MIGRATION-PLANNER-137
Budget exhaustion no producirá truncated plan.

## DB-MIGRATION-PLANNER-138
Planning cancellation será explícita.

## DB-MIGRATION-PLANNER-139
PlanningSession será operation-scoped.

## DB-MIGRATION-PLANNER-140
Shared planner será immutable/stateless.

## DB-MIGRATION-PLANNER-141
Persistent runtime no conservará mutable planning state entre requests.

## DB-MIGRATION-PLANNER-142
Cached plan no contendrá live resources.

## DB-MIGRATION-PLANNER-143
Cached plan será revalidado antes de execution.

## DB-MIGRATION-PLANNER-144
Raw migration operations serán explicit opaque boundaries.

## DB-MIGRATION-PLANNER-145
Opaque operation no recibirá fake safety guarantees.

## DB-MIGRATION-PLANNER-146
Arbitrary callback no será arquitectura preferida.

## DB-MIGRATION-PLANNER-147
Planning telemetry no expondrá secrets por defecto.

## DB-MIGRATION-PLANNER-148
Planner tendrá conformance/determinism testing.

## DB-MIGRATION-PLANNER-149
Vendor-specific behavior se resolverá mediante capabilities/strategies, no conditionals dispersos en migrations.

## DB-MIGRATION-PLANNER-150
VoltStack nunca confundirá un plan válido con una migración ya ejecutada.

---

# 221. Anti-patterns

## 221.1 Ejecutar mientras se planea

Incorrecto:

```php
foreach ($pending as $migration) {
    $migration->up();
}
```

dentro del Planner.

---

## 221.2 Ordenar únicamente por timestamp

Incorrecto:

```php
sort($migrations);
```

sin dependency graph.

---

## 221.3 SQL dentro del Planner

Incorrecto:

```php
$sql = 'ALTER TABLE users ...';
```

El SQL pertenece al Compiler.

---

## 221.4 Emulación escondida

Incorrecto:

```text
Planner:
DropColumn

Compiler:
rebuild table secretly
```

---

## 221.5 Query de datos durante planning

Incorrecto:

```php
$count = $connection->query(
    'SELECT COUNT(*) FROM users WHERE email IS NULL'
);
```

El Planner deberá crear un runtime validation requirement.

---

## 221.6 Batch como transaction

Incorrecto:

```text
Batch 5
=
one transaction
```

---

## 221.7 Retry universal

Incorrecto:

```php
catch (\Throwable) {
    retry();
}
```

---

## 221.8 Rollback = reverse array

Incorrecto:

```php
$down = array_reverse($up);
```

---

## 221.9 Global tenant

Incorrecto:

```php
MigrationPlanner::$currentTenant = $tenant;
```

---

## 221.10 Raw callback como única abstracción

Incorrecto:

```php
new Migration(function () {
    // arbitrary application code
});
```

como modelo arquitectónico principal.

---

# 222. Ejemplo completo

Migration intent:

```text
Make users.email NOT NULL
```

Current state:

```text
users.email nullable
large production table
```

Policy:

```text
ZERO_DOWNTIME
```

Plan:

```text
MigrationPlan
│
├── PRECHECK
│   └── VerifyRepositoryFingerprint
│
├── PREPARE
│   └── VerifyColumnExists
│
├── BACKFILL
│   └── DynamicChunkedBackfill
│       SET email = generated/fallback value
│       WHERE email IS NULL
│
├── BARRIER
│   └── BackfillCompleted
│
├── VALIDATE
│   └── ValidateNoNullValues
│
├── BARRIER
│   └── ValidationPassed
│
├── CONTRACT
│   └── AlterColumnSetNotNull
│
├── VERIFY
│   └── VerifyColumnNotNull
│
└── REPOSITORY_COMMIT
    └── RecordMigrationApplied
```

El Planner describe esto sin ejecutar ninguna fase.

---

# 223. Flujo hacia Schema Compiler

```text
MigrationPlan
      │
      ▼
Schema MigrationPlanOperation
      │
      ▼
Planned Schema Operation
      │
      ▼
Schema Compiler
      │
      ▼
CompiledSchemaCommandSet
```

Por tanto:

```text
Migration Planner decides strategy.
Schema Compiler renders platform representation.
Migration Executor coordinates execution.
```

---

# 224. Flujo hacia Execution

```text
Immutable MigrationPlan
          │
          ▼
Execution Validation
          │
          ├── Repository unchanged?
          ├── Schema assumptions valid?
          ├── Approval valid?
          ├── Runtime conditions?
          └── Capabilities compatible?
          │
          ▼
Migration Executor
          │
          ├── transactions
          ├── locks
          ├── barriers
          ├── validations
          ├── backfills
          ├── schema commands
          ├── retries
          └── repository updates
```

---

# 225. Correctness model

```text
CorrectMigrationPlan
=
SelectionCorrect
∧
DependencyCorrect
∧
StrategyValid
∧
CapabilityValid
∧
SafetyPreserved
∧
TransactionConsistent
∧
LockRequirementsExplicit
∧
RetryBoundariesExplicit
∧
ConditionsExplicit
∧
Deterministic
∧
Immutable
∧
NonExecuting
```

---

# 226. Safety formula

```text
ExecutablePlan
=
CompletePlan
∧
DependenciesSatisfied
∧
CapabilitiesSatisfied
∧
SafetyRequirementsSatisfied
∧
RuntimePreconditionsSatisfied
∧
RepositoryAssumptionsValid
∧
SchemaAssumptionsValid
∧
AuthorizationSatisfied
```

El Planner puede producir el plan antes de que todas las condiciones runtime estén satisfechas.

El Executor no deberá ejecutarlo hasta validarlas.

---

# 227. Plan equivalence

Dos planes pueden ser:

```text
structurally different
```

pero:

```text
semantically equivalent
```

Sin embargo, para audit/reproducibility, VoltStack deberá fingerprintar la estrategia operacional concreta, no únicamente el estado final esperado.

---

# 228. Filosofía de diseño

VoltStack debe evitar dos extremos.

### Planner demasiado simple

```text
find pending
sort filenames
call up()
```

Esto pierde:

```text
dependencies
safety
zero downtime
retry reasoning
platform strategy
recovery
explainability
```

### Planner demasiado inteligente con I/O oculto

```text
query database
execute probes
modify schema
retry automatically
```

Esto destruye:

```text
determinism
testability
preview
reproducibility
separation of concerns
```

El equilibrio será:

> **Planner inteligente, pero puro respecto de efectos externos.**

---

# 229. Fórmula maestra

```text
MigrationPlan =
Plan(
    SelectedMigrations,
    DependencyClosure,
    RepositorySnapshot,
    ReconciliationState,
    SchemaState?,
    PlatformCapabilities,
    CompatibilityAssessments,
    SafetyAssessments,
    PlanningFacts,
    Target,
    Policies,
    FrozenExtensions
)
```

---

# 230. Regla final

> **El Migration Planner decide cómo debería desarrollarse una evolución de base de datos; el Migration Executor decide cuándo y bajo qué contexto runtime ejecutar cada paso, y los subsistemas inferiores realizan los efectos concretos.**

En forma compacta:

```text
Discovery finds.
Repository remembers.
Reconciliation compares.
Planner strategizes.
Compiler represents.
Executor orchestrates.
Driver performs.
```

---

# 231. Resultado arquitectónico

Con este diseño VoltStack podrá ofrecer una experiencia sencilla:

```php
Schema::table('users', function (TableBlueprint $table) {
    $table->string('email')->nullable(false);
});
```

mientras internamente puede convertir esa intención en:

```text
Migration Definition
        ↓
Semantic Migration Operation
        ↓
Compatibility Analysis
        ↓
Safety Analysis
        ↓
Migration Planning
        ↓
Validation
        ↓
Backfill
        ↓
Barrier
        ↓
Schema Strategy
        ↓
Transaction Boundaries
        ↓
Lock Requirements
        ↓
Compiled DDL
        ↓
Execution
        ↓
Verification
        ↓
Repository Update
```

Esto permite que VoltStack conserve la ergonomía esperada por desarrolladores Laravel/PHP sin limitar su arquitectura a un simple sistema:

```text
filename
→ up()
→ SQL
→ insert migration row
```

y establece la base para migraciones empresariales, recuperables, auditables, multi-plataforma y compatibles con runtimes persistentes.

---

# 232. Siguiente documento

```text
106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md
```

El siguiente documento definirá el runtime que consume `MigrationPlan` y coordina realmente:

```text
plan validation
repository freshness verification
migration coordination locks
preconditions
transactions
compiled schema commands
data operations
backfills
barriers
runtime validations
timeouts
cancellation
retry boundaries
failure classification
partial execution
unknown outcomes
postconditions
repository writes
execution history
cleanup
recovery information
```

estableciendo la separación:

```text
Migration Plan
=
immutable strategy

Migration Execution
=
live orchestration

Schema Execution
=
DDL effect execution

Query Execution
=
statement execution

Repository
=
persistent migration history
```