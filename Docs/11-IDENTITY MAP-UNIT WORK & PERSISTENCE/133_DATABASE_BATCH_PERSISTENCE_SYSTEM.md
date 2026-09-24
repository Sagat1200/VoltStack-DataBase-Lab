# 133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md

# VoltStack Quantum Database
## Database Batch Persistence System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 133 — Database Batch Persistence System  
**Bloque:** 11 — Identity Map, Unit of Work & Persistence  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Batch Persistence System` define la arquitectura mediante la cual VoltStack puede agrupar múltiples operaciones de persistencia compatibles para reducir:

- round trips hacia la base de datos;
- compilaciones repetitivas;
- preparación repetida de statements;
- overhead del driver;
- cambios de contexto;
- costo de binding;
- latencia total de `flush()`.

Sin alterar la semántica individual de las entidades.

El principio central es:

> **El batching en VoltStack será una optimización física del `PersistencePlan`, nunca una modificación de la semántica ORM.**

Por tanto:

```text
Individual Persistence Semantics
            │
            ▼
     PersistencePlan
            │
            ▼
   Batch Optimization
            │
            ▼
      BatchPlan
            │
            ▼
       Execution
```

y nunca:

```text
Batch Optimization
      ↓
invent ORM semantics
```

---

# 2. Problema

Supongamos:

```php
for ($i = 0; $i < 1000; $i++) {
    $user = new User(...);

    $entityManager->persist($user);
}

$entityManager->flush();
```

Sin batching:

```text
INSERT User #1
INSERT User #2
INSERT User #3
...
INSERT User #1000
```

puede producir:

```text
1000 executions
1000 parameter binding phases
1000 protocol exchanges
```

Aunque muchas operaciones sean estructuralmente equivalentes.

VoltStack deberá poder transformar físicamente:

```text
InsertOperation<User>#1
InsertOperation<User>#2
InsertOperation<User>#3
...
```

en grupos compatibles:

```text
BatchInsertStep<User>
├── Row #1
├── Row #2
├── Row #3
└── ...
```

sin perder la relación:

```text
Batch Row
↔
Persistence Operation
↔
Entity
↔
Execution Outcome
```

---

# 3. Objetivos

El sistema deberá proporcionar:

```text
operation grouping
batch eligibility
dependency-aware batching
multi-row INSERT
prepared statement reuse
UPDATE batching
DELETE batching
generated-value correlation
batch sizing
parameter-limit awareness
packet-size awareness
platform capability awareness
partial-failure modeling
optimistic-lock preservation
lifecycle preservation
outcome reconciliation
retry safety
memory governance
telemetry
```

---

# 4. No objetivos

El sistema no será responsable de:

```text
entity change detection
UnitOfWork ownership
SQL generation
transaction ownership
relationship semantics
entity lifecycle definition
database connection routing
schema design
migration batching
generic ETL
```

Estas responsabilidades pertenecen a otros subsistemas.

---

# 5. Posición arquitectónica

```text
EntityManager
     │
     ▼
UnitOfWork
     │
     ▼
Flush System
     │
     ▼
Persistence Planner
     │
     ▼
Logical PersistencePlan
     │
     ▼
┌─────────────────────────────┐
│ Batch Persistence System    │
│                             │
│ Eligibility                 │
│ Grouping                    │
│ Dependency Analysis         │
│ Batch Sizing                │
│ Physical Strategy           │
└──────────────┬──────────────┘
               │
               ▼
      Physical PersistencePlan
               │
               ▼
       Persistence Executor
               │
               ▼
          Query Engine
               │
               ▼
        Execution Engine
               │
               ▼
            Driver
```

---

# 6. Regla de arquitectura

La arquitectura distingue:

```text
Logical Persistence Operation
```

de:

```text
Physical Persistence Execution Strategy
```

Ejemplo:

```text
100 InsertEntityOperation
```

pueden físicamente ejecutarse como:

```text
1 multi-row INSERT
```

pero siguen existiendo conceptualmente:

```text
100 semantic insert outcomes
```

---

# 7. Fórmula fundamental

```text
Batching
=
PhysicalExecutionOptimization(
    SemanticallyStablePersistencePlan
)
```

con la condición:

```text
SemanticsBeforeBatch
=
SemanticsAfterBatch
```

dentro de las garantías de la plataforma.

---

# 8. Distinciones fundamentales

```text
Batch Persistence
≠ Bulk Query

Batch Persistence
≠ Bulk Import

Batch Persistence
≠ Query Builder bulk UPDATE

Batch Persistence
≠ UnitOfWork

Batch Persistence
≠ Flush

Batch Persistence
≠ Transaction

Batch Persistence
≠ SQL Compiler

Batch Persistence
≠ Driver batching

Batch Group
≠ Transaction

Batch Failure
≠ Transaction Rollback

Batch Success
≠ Transaction Commit

Multi-row INSERT
≠ ORM semantic insert
```

---

# 9. Batch Persistence vs Bulk Operations

Es una distinción especialmente importante.

## Batch Persistence

Parte de entidades administradas:

```text
Entity
→ UnitOfWork
→ ChangeSet
→ PersistenceOperation
→ Batch
```

Preserva:

```text
IdentityMap
Snapshots
Lifecycle
Generated Values
EntityState
Optimistic Locking
```

## Bulk Operation

Ejemplo:

```php
User::query()
    ->where('status', 'inactive')
    ->update(['archived' => true]);
```

puede operar directamente mediante Query Engine.

No necesariamente:

```text
hydrate entities
run lifecycle
update IdentityMap
update snapshots
```

Por tanto:

> **Batch Persistence optimiza operaciones ORM individuales; Bulk Persistence expresa una operación colectiva diferente.**

---

# 10. Modelo conceptual

```text
PersistencePlan
     │
     ▼
BatchAnalyzer
     │
     ├── dependencies
     ├── operation shape
     ├── entity metadata
     ├── generated values
     ├── locking
     └── platform capabilities
     │
     ▼
BatchEligibility
     │
     ▼
BatchGroupBuilder
     │
     ▼
BatchGroup[]
     │
     ▼
BatchSizing
     │
     ▼
BatchExecutionStrategy
     │
     ▼
PhysicalPersistencePlan
```

---

# 11. Componentes principales

```text
BatchPersistenceOptimizer
BatchEligibilityAnalyzer
BatchCompatibilityAnalyzer
BatchGroupBuilder
BatchDependencyAnalyzer
BatchSizer
BatchExecutionStrategyResolver
BatchInsertExecutor
BatchUpdateExecutor
BatchDeleteExecutor
BatchOutcomeCorrelator
BatchGeneratedValueCorrelator
BatchReconciler
BatchRetryAnalyzer
```

---

# 12. BatchPersistenceOptimizer

Contrato:

```php
interface BatchPersistenceOptimizer
{
    public function optimize(
        PersistencePlan $plan,
        BatchPersistenceContext $context,
    ): BatchOptimizationResult;
}
```

---

# 13. Input

Consume:

```text
validated
+
stabilized
+
dependency-aware
PersistencePlan
```

No debe intentar corregir un plan semánticamente inválido.

---

# 14. Output

Produce:

```php
final readonly class BatchOptimizationResult
{
    public function __construct(
        public PhysicalPersistencePlan $plan,
        public BatchOptimizationStatistics $statistics,
        public BatchDiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 15. Logical vs Physical Plan

Puede resultar útil formalizar dos niveles:

```text
LogicalPersistencePlan
        ↓
PhysicalPersistencePlan
```

El primero describe:

```text
what must happen
```

El segundo:

```text
how compatible operations will be executed
```

---

# 16. Invariante

```text
Logical Plan
```

es la fuente de verdad semántica.

El physical plan no podrá eliminar significado necesario para reconciliación.

---

# 17. BatchGroup

```php
final readonly class BatchGroup
{
    public function __construct(
        public BatchGroupId $id,
        public BatchOperationType $type,
        public BatchShape $shape,
        public array $operations,
        public BatchExecutionStrategy $strategy,
    ) {}
}
```

---

# 18. BatchOperationType

```php
enum BatchOperationType
{
    case INSERT;
    case UPDATE;
    case DELETE;
    case RELATIONSHIP_INSERT;
    case RELATIONSHIP_DELETE;
}
```

---

# 19. Batch identity

Cada batch deberá poseer:

```text
BatchGroupId
```

para:

```text
telemetry
diagnostics
outcome correlation
failure reporting
```

---

# 20. Batch Shape

Dos operaciones no son batch-compatible solo porque pertenezcan a la misma entidad.

Ejemplo:

```text
UPDATE users
SET name = ?
WHERE id = ?
```

y:

```text
UPDATE users
SET name = ?, email = ?
WHERE id = ?
```

tienen shapes diferentes.

---

# 21. BatchShape

Puede incluir:

```text
operation type
target table
column set
predicate shape
returning requirements
generated-value requirements
locking/version requirements
platform strategy
```

---

# 22. Shape fingerprint

```text
BatchShapeFingerprint
=
Hash(
    OperationType,
    Target,
    Columns,
    PredicateShape,
    ReturningShape,
    LockingShape
)
```

---

# 23. Fingerprint ≠ semantic identity

El fingerprint solo ayuda a agrupar.

Siempre deberá verificarse compatibilidad estructural real.

---

# 24. Eligibility

Una operación es batchable si:

```text
BatchEligible(op)
=
SemanticallyCompatible
∧
DependencyCompatible
∧
PlatformSupported
∧
OutcomeCorrelatable
∧
GeneratedValuesCorrelatable
∧
LockingSemanticsPreserved
```

---

# 25. BatchEligibilityResult

```php
final readonly class BatchEligibilityResult
{
    public function __construct(
        public bool $eligible,
        public BatchEligibilityReasonCollection $reasons,
        public ?BatchShape $shape,
    ) {}
}
```

---

# 26. Razones de exclusión

Ejemplos:

```text
DEPENDENCY_BOUNDARY
DIFFERENT_TARGET
DIFFERENT_COLUMN_SHAPE
GENERATED_VALUE_DEPENDENCY
UNSUPPORTED_RETURNING_SEMANTICS
OPTIMISTIC_LOCK_CORRELATION_UNSAFE
PLATFORM_UNSUPPORTED
PARAMETER_LIMIT
CUSTOM_OPERATION
NON_BATCHABLE_EXTENSION
OUTCOME_CORRELATION_UNSAFE
```

---

# 27. Same table ≠ same batch

Estas operaciones:

```text
INSERT users(name)
INSERT users(name, email)
```

no necesariamente forman el mismo multi-row INSERT.

---

# 28. Same entity ≠ same batch

También pueden existir:

```text
User INSERT
User UPDATE
User DELETE
```

que nunca pertenecen al mismo grupo físico.

---

# 29. Dependency boundaries

Supongamos:

```text
Insert A
   ↓ generated ID
Insert B
```

No podrá ejecutarse:

```text
Batch(A, B)
```

si `B` necesita el ID producido por `A`.

---

# 30. Independent siblings

En cambio:

```text
Insert B1 ─┐
Insert B2 ─┼─ depend on already resolved Parent#42
Insert B3 ─┘
```

pueden ser batchables.

---

# 31. Dependency layers

El planner puede producir:

```text
Layer 0
Layer 1
Layer 2
```

El batching opera principalmente dentro de capas compatibles.

---

# 32. Fórmula

```text
BatchGroup ⊆ DependencyCompatibleExecutionLayer
```

salvo estrategias explícitas que preserven dependencias internas.

---

# 33. Stable ordering

Aunque la base de datos no requiera cierto ordering, VoltStack deberá conservar ordering determinista para:

```text
diagnostics
generated-value correlation
lifecycle reconciliation
testing
```

---

# 34. Batch sequence number

Cada operación puede conservar:

```text
LogicalOperationSequence
```

independientemente de su posición física.

---

# 35. INSERT batching

Caso ideal:

```text
INSERT User A
INSERT User B
INSERT User C
```

con mismo shape.

---

# 36. Multi-row INSERT

Puede compilarse como:

```sql
INSERT INTO users (name, email)
VALUES (?, ?),
       (?, ?),
       (?, ?)
```

si la plataforma y las necesidades semánticas lo permiten.

---

# 37. SQL no pertenece al Batch System

El Batch System selecciona una estrategia como:

```text
MultiRowInsertStrategy
```

pero el SQL real sigue siendo generado por:

```text
Query Model
→ SQL Compiler
```

---

# 38. MultiRowInsertQueryModel

El sistema podrá producir un Query Model equivalente:

```text
InsertQuery
├── columns
└── rows[]
```

---

# 39. Generated IDs

Los identificadores generados complican multi-row INSERT.

Ejemplo:

```text
User A → ?
User B → ?
User C → ?
```

Después del INSERT se necesita:

```text
A ↔ id 101
B ↔ id 102
C ↔ id 103
```

---

# 40. Nunca inferir IDs ingenuamente

No se deberá asumir universalmente:

```text
firstId = 101
nextId = 102
nextId = 103
```

---

# 41. Razón

Puede fallar por:

```text
sequences
triggers
concurrent inserts
auto-increment configuration
platform behavior
generated UUIDs
custom generators
```

---

# 42. GeneratedValueCorrelation

Solo podrá utilizarse batching si:

```text
GeneratedValueCorrelation
```

es suficientemente confiable.

---

# 43. Estrategias

```php
enum GeneratedValueCorrelationStrategy
{
    case NONE_REQUIRED;
    case CLIENT_GENERATED;
    case RETURNING_PER_ROW;
    case DRIVER_CORRELATED;
    case PLATFORM_PROVEN_ORDER;
    case UNSUPPORTED;
}
```

---

# 44. Client-generated IDs

Ejemplo:

```text
UUID / ULID generated before INSERT
```

facilita batching:

```text
Entity already knows identifier
```

sin afirmar todavía existencia en DB.

---

# 45. Server-generated IDs

Requieren soporte de plataforma/driver capaz de correlacionar resultados.

---

# 46. Unsafe correlation

Si no puede demostrarse:

```text
row ↔ generated ID
```

VoltStack deberá degradar a:

```text
smaller batches
```

o:

```text
individual execution
```

---

# 47. Correctness over batching

Regla:

```text
UnknownGeneratedValueCorrelation
→
DoNotBatch
```

---

# 48. Generated columns

No solo IDs.

También:

```text
created_at
version
computed columns
database defaults
trigger-produced values
```

pueden necesitar correlación.

---

# 49. RETURNING

Cuando la plataforma permita semántica equivalente a:

```text
INSERT ... RETURNING ...
```

podrá facilitar:

```text
row result
↔
entity operation
```

---

# 50. Platform capability

Nunca:

```php
if ($database === 'postgresql') {
    ...
}
```

en el Batch System.

Debe utilizar:

```php
$capabilities->supports(
    DatabaseCapability::MULTI_ROW_INSERT_RETURNING
);
```

---

# 51. Prepared statement reuse

No todo batching requiere multi-row SQL.

Ejemplo:

```text
PreparedStatement
    ↓
execute row A
execute row B
execute row C
```

puede reducir overhead aun manteniendo ejecuciones individuales.

---

# 52. Strategies

```php
enum BatchExecutionStrategy
{
    case INDIVIDUAL;
    case REUSED_PREPARED_STATEMENT;
    case MULTI_ROW_STATEMENT;
    case DRIVER_NATIVE_BATCH;
}
```

---

# 53. Strategy selection

```text
BatchShape
+
PlatformCapabilities
+
DriverCapabilities
+
GeneratedValueRequirements
+
ParameterLimits
+
FailureSemantics
        ↓
BatchExecutionStrategy
```

---

# 54. Reused prepared statements

Adecuado cuando:

```text
same SQL shape
different parameters
```

pero multi-row no es seguro o soportado.

---

# 55. Driver-native batch

Solo se utilizará si el driver declara un contrato claro sobre:

```text
ordering
per-operation results
generated values
failure behavior
atomicity
```

---

# 56. Driver batching ≠ ORM batching

El ORM Batch System decide:

```text
semantic grouping
```

El driver puede proporcionar:

```text
physical execution primitive
```

---

# 57. UPDATE batching

Más complejo que INSERT.

Supongamos:

```text
UPDATE User A name
UPDATE User B name
UPDATE User C name
```

---

# 58. Prepared statement reuse

La estrategia universalmente más simple:

```sql
UPDATE users
SET name = ?
WHERE id = ?
```

ejecutado varias veces con statement reutilizado.

---

# 59. Multi-row UPDATE

Algunas plataformas permiten estrategias mediante:

```text
CASE
VALUES
temporary tables
join updates
```

pero no deberán usarse si cambian la semántica.

---

# 60. Optimistic locking

Ejemplo:

```sql
UPDATE users
SET name = ?, version = ?
WHERE id = ?
  AND version = ?
```

Cada entidad necesita conocer:

```text
affected row count
```

---

# 61. Batch optimistic locking

Si se agrupan múltiples updates y solo se recibe:

```text
affectedRows = 9
```

para 10 entidades, no sabemos necesariamente cuál falló.

---

# 62. Regla

```text
CannotCorrelateOptimisticOutcomePerEntity
→
DoNotUseThatBatchStrategy
```

---

# 63. Reused statements

Pueden mantener:

```text
per-execution row count
```

y preservar optimistic locking.

---

# 64. Advanced batch UPDATE

Solo si la plataforma puede devolver correlación suficiente.

---

# 65. Different ChangeSets

```text
User A: name
User B: email
User C: name + email
```

producen grupos diferentes.

---

# 66. UPDATE grouping

```text
Group 1 → SET name
Group 2 → SET email
Group 3 → SET name,email
```

---

# 67. Version columns

El batch shape incluye:

```text
version predicate
version update strategy
generated version requirements
```

---

# 68. DELETE batching

Ejemplo:

```text
DELETE User#1
DELETE User#2
DELETE User#3
```

---

# 69. Individual prepared DELETE

```sql
DELETE FROM users WHERE id = ?
```

reutilizando statement.

---

# 70. Set-based DELETE

Podría producir:

```sql
DELETE FROM users
WHERE id IN (?, ?, ?)
```

solo si preserva semántica.

---

# 71. Problem with optimistic delete

Si existe:

```text
id + version
```

un simple `IN` puede perder correlación.

---

# 72. Composite predicates

También:

```text
(id = ? AND version = ?)
OR
(id = ? AND version = ?)
```

puede crecer rápidamente.

---

# 73. Delete outcome

Cada semantic delete debe conservar:

```text
SUCCEEDED
FAILED
CONFLICT
UNKNOWN
```

cuando la arquitectura necesite granularidad individual.

---

# 74. Relationship batching

Join table:

```text
user_roles
```

puede producir:

```text
INSERT relationship A
INSERT relationship B
INSERT relationship C
```

---

# 75. Join-table insert

Normalmente excelente candidato para multi-row INSERT porque puede no requerir generated IDs.

---

# 76. Join-table delete

Puede utilizar:

```text
prepared statement reuse
```

o estrategia set-based si conserva correlación requerida.

---

# 77. Relationship semantics

El batching no decide:

```text
cascade
ownership
orphan removal
```

Eso ya fue decidido antes.

---

# 78. Lifecycle preservation

Supongamos:

```text
Entity A
Entity B
Entity C
```

insertadas en batch.

Cada una sigue teniendo semánticamente:

```text
postPersist(A)
postPersist(B)
postPersist(C)
```

si sus operaciones correspondientes fueron exitosas con suficiente certeza.

---

# 79. Batch callback

VoltStack no reemplazará automáticamente:

```text
3 postPersist
```

por:

```text
1 postBatchPersist
```

---

# 80. Optional batch lifecycle

Podrá existir un evento operacional adicional:

```text
BatchPersistenceExecuted
```

pero será distinto del lifecycle de entidad.

---

# 81. Pre lifecycle

Los:

```text
prePersist
preUpdate
preRemove
```

ya deben haber participado en stabilization antes del batching físico.

---

# 82. No lifecycle during grouping

`BatchGroupBuilder` no ejecuta callbacks de entidad.

---

# 83. No hidden entity mutation

Batch optimization deberá ser:

```text
side-effect free
```

respecto al estado de las entidades.

---

# 84. Batch size

No siempre conviene:

```text
1 batch = every compatible operation
```

---

# 85. Limits

Debe considerar:

```text
maximum parameters
maximum SQL length
maximum packet size
driver limits
server limits
memory budget
generated result size
latency target
```

---

# 86. BatchSizer

```php
interface BatchSizer
{
    public function size(
        BatchGroupCandidate $candidate,
        BatchSizingContext $context,
    ): BatchSizingDecision;
}
```

---

# 87. BatchSizingDecision

```php
final readonly class BatchSizingDecision
{
    public function __construct(
        public int $maxOperations,
        public int $maxParameters,
        public ?int $estimatedMaxBytes,
        public BatchSizingReasonCollection $reasons,
    ) {}
}
```

---

# 88. Parameter formula

Para un multi-row INSERT:

```text
ParameterCount
=
RowCount × ParametersPerRow
```

Por tanto:

```text
RowCount
≤
floor(
    PlatformParameterLimit
    /
    ParametersPerRow
)
```

---

# 89. Reserved parameters

Si existen parámetros adicionales:

```text
RowCount
≤
floor(
    (ParameterLimit - ReservedParameters)
    /
    ParametersPerRow
)
```

---

# 90. Example

```text
parameter limit = 1000
parameters/row = 4

max rows = 250
```

antes de aplicar otros límites.

---

# 91. Query size

También:

```text
EstimatedStatementBytes
≤
StatementSizeBudget
```

---

# 92. Packet size

En plataformas con límites de paquete/protocolo:

```text
EstimatedPayloadBytes
≤
EffectivePacketBudget
```

---

# 93. Safety margin

No utilizar el límite absoluto.

```text
EffectiveLimit
=
DeclaredLimit × SafetyFactor
```

por ejemplo configurable.

---

# 94. No hard-coded vendor constants

Los límites deberán provenir de:

```text
PlatformCapabilities
ConnectionCapabilities
DriverCapabilities
Configuration
```

---

# 95. Unknown limit

Si el límite es desconocido:

```text
UNKNOWN ≠ UNLIMITED
```

---

# 96. Conservative default

Se utilizará un tamaño seguro configurable.

---

# 97. Configuration

Ejemplo:

```php
'database' => [
    'orm' => [
        'batch_persistence' => [
            'enabled' => true,
            'default_size' => 100,
            'max_size' => 1000,
            'adaptive' => true,
        ],
    ],
],
```

---

# 98. Configuration ≠ capability

Configurar:

```text
max_size = 1000
```

no significa que la plataforma soporte 1000 operaciones.

---

# 99. Effective batch size

```text
EffectiveBatchSize
=
min(
    ConfiguredLimit,
    PlatformLimit,
    DriverLimit,
    ParameterLimitDerivedSize,
    PayloadDerivedSize,
    MemoryDerivedSize,
    SemanticLimit
)
```

---

# 100. Adaptive batching

VoltStack podrá ajustar tamaños usando señales operacionales.

---

# 101. Inputs adaptativos

```text
recent execution latency
statement size
driver feedback
memory pressure
timeouts
packet-limit errors
```

---

# 102. No semantic adaptation

El algoritmo adaptativo solo cambia:

```text
physical batch size
```

Nunca:

```text
transaction semantics
lifecycle semantics
optimistic locking
```

---

# 103. Adaptive state scope

No almacenar estadísticas mutables peligrosas globalmente.

---

# 104. Shared adaptive model

Si en el futuro se comparte, deberá ser:

```text
thread-safe
bounded
tenant-safe
connection-profile-aware
```

---

# 105. V1 recommendation

Usar sizing determinista basado en capacidades y configuración.

Adaptive batching puede añadirse posteriormente.

---

# 106. Batch boundaries

Un grupo lógico puede dividirse:

```text
Group 1000 rows
```

en:

```text
Batch #1 100
Batch #2 100
...
Batch #10 100
```

---

# 107. Boundary semantics

La división física no cambia:

```text
PersistenceOperationId
```

de cada operación.

---

# 108. Operation correlation

Cada item conserva:

```php
final readonly class BatchItem
{
    public function __construct(
        public BatchItemIndex $index,
        public PersistenceOperationId $operationId,
        public EntityReference $entity,
        public ParameterSet $parameters,
    ) {}
}
```

---

# 109. EntityReference

Aquí deberá ser una referencia interna segura al objeto/identidad necesaria para reconciliación.

No un EntityManager global.

---

# 110. BatchOutcome

```php
final readonly class BatchExecutionOutcome
{
    public function __construct(
        public BatchGroupId $batchId,
        public BatchExecutionStatus $status,
        public array $itemOutcomes,
        public BatchExecutionEvidence $evidence,
    ) {}
}
```

---

# 111. BatchExecutionStatus

```php
enum BatchExecutionStatus
{
    case SUCCEEDED;
    case FAILED;
    case PARTIALLY_SUCCEEDED;
    case UNKNOWN;
}
```

---

# 112. Item outcome

```php
enum BatchItemStatus
{
    case SUCCEEDED;
    case FAILED;
    case CONFLICT;
    case NOT_EXECUTED;
    case UNKNOWN;
}
```

---

# 113. Batch success

```text
Batch SUCCEEDED
```

solo si la evidencia permite considerar exitosos todos los items relevantes.

---

# 114. Partial success

Ejemplo:

```text
Item 1 SUCCEEDED
Item 2 SUCCEEDED
Item 3 FAILED
Item 4 NOT_EXECUTED
```

produce:

```text
PARTIALLY_SUCCEEDED
```

si no existe una transacción que revierta con certeza los efectos.

---

# 115. Unknown batch

Ejemplo:

```text
multi-row statement sent
connection lost
```

puede producir:

```text
Batch = UNKNOWN
Items = UNKNOWN
```

---

# 116. No invented per-row certainty

Si la DB solo informa:

```text
statement affected 100 rows
```

pero no permite correlacionar resultados individuales requeridos, VoltStack no inventará:

```text
item #1 success
...
item #100 success
```

si la semántica concreta requiere más evidencia.

---

# 117. Statement atomicity

Algunas operaciones SQL individuales pueden ser atómicas dentro del motor.

Pero:

```text
StatementAtomicity
≠
TransactionCommit
```

---

# 118. Batch atomicity

Debe modelarse explícitamente:

```php
enum BatchAtomicity
{
    case STATEMENT_ATOMIC;
    case TRANSACTIONAL;
    case PER_ITEM;
    case UNKNOWN;
}
```

---

# 119. Atomicity source

Proviene de:

```text
platform semantics
driver semantics
transaction context
execution strategy
```

---

# 120. Transaction integration

Batching no crea una transacción arbitrariamente.

Recibe el contexto coordinado por Flush/Transaction System.

---

# 121. Joined transaction

```text
Batch executed successfully
Transaction still ACTIVE
```

sigue siendo posible.

---

# 122. Rollback

Si la transacción se revierte:

```text
batch execution success
≠
durable persistence
```

---

# 123. Batch failure ≠ rollback

El executor no deberá afirmar rollback salvo evidencia del Transaction Manager.

---

# 124. Retry

Batch retry es particularmente delicado.

---

# 125. Unsafe retry

Si:

```text
batch outcome = UNKNOWN
```

no se debe reenviar automáticamente.

---

# 126. Why

Podría duplicar:

```text
INSERT
side effects from triggers
version increments
relationship rows
```

---

# 127. Retry analysis

```text
SafeBatchRetry
=
KnownExecutionOutcome
∧
KnownTransactionOutcome
∧
ReplaySafeOperations
∧
GeneratedValuesReconciled
∧
NoUnsafeLifecycleSideEffects
```

---

# 128. Partial retry

Solo podría reintentar:

```text
NOT_EXECUTED
```

o:

```text
certainly failed + replay-safe
```

items.

---

# 129. Never retry succeeded items

Salvo rollback cierto de la transacción que contenía el batch.

---

# 130. Insert idempotency

Client-generated unique identifiers pueden facilitar idempotencia, pero:

```text
unique ID
≠
automatic replay safety
```

porque pueden existir triggers y otros efectos.

---

# 131. Optimistic locking batch outcome

Cada optimistic operation necesita:

```text
expected version
actual affected result
```

suficientemente correlacionado.

---

# 132. Conflict

```text
affectedRows = 0
```

en operación individual puede significar:

```text
OptimisticLockConflict
```

según mapping/policy.

---

# 133. Batched conflict

Si no puede determinarse qué entidad produjo el conflicto:

```text
batch strategy is semantically insufficient
```

---

# 134. Strategy fallback

El resolver deberá degradar:

```text
MultiRow
    ↓
PreparedStatementReuse
    ↓
Individual
```

hasta encontrar una estrategia segura.

---

# 135. Fallback hierarchy

Conceptualmente:

```text
Most Efficient Safe Strategy
           ↓
        selected
```

No:

```text
Most Efficient Strategy Regardless of Semantics
```

---

# 136. BatchStrategyResolver

```php
interface BatchExecutionStrategyResolver
{
    public function resolve(
        BatchGroupCandidate $candidate,
        BatchExecutionCapabilities $capabilities,
    ): BatchExecutionStrategyDecision;
}
```

---

# 137. Strategy decision

Debe explicar:

```text
selected strategy
rejected strategies
reasons
effective limits
```

---

# 138. Explainability

Útil para:

```text
debug toolbar
telemetry
performance tuning
tests
```

---

# 139. Platform capabilities

Capacidades potenciales:

```text
MULTI_ROW_INSERT
MULTI_ROW_INSERT_RETURNING
MULTI_ROW_GENERATED_VALUES
PREPARED_STATEMENT_REUSE
DRIVER_NATIVE_BATCH
BATCH_PER_ITEM_ROW_COUNT
BATCH_ORDERED_RESULTS
DELETE_RETURNING
UPDATE_RETURNING
MAX_BIND_PARAMETERS
MAX_STATEMENT_BYTES
```

---

# 140. Capability status

Deberá respetar el modelo existente:

```text
SUPPORTED
WITH_LIMITATIONS
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 141. UNKNOWN capability

No se interpretará como `SUPPORTED`.

---

# 142. MySQL

La estrategia concreta dependerá de capabilities detectadas.

El Batch System no codifica directamente reglas MySQL.

---

# 143. MariaDB

Se tratará como plataforma first-class independiente.

No:

```text
MariaDB = MySQL
```

---

# 144. PostgreSQL

Sus capacidades podrán habilitar estrategias distintas.

Pero la decisión sigue siendo capability-driven.

---

# 145. SQLite

Debe respetar:

```text
parameter limits
statement limits
transaction behavior
RETURNING capabilities according to runtime version
```

sin asumir capacidades por nombre solamente.

---

# 146. Version ≠ Capability

Regla heredada:

```text
DatabaseVersion
≠
DatabaseCapability
```

---

# 147. Runtime capability discovery

Puede combinar:

```text
platform family
server version
driver
connection settings
feature probes
configuration
```

---

# 148. Capability snapshot

El batch planner deberá consumir una vista estable de capacidades durante un flush.

---

# 149. No capability mutation mid-batch

La estrategia física no deberá cambiar arbitrariamente dentro de un batch ya congelado.

---

# 150. Prepared statement cache

Batching podrá aprovechar:

```text
PreparedStatementSystem
```

pero no será propietario global de ese cache.

---

# 151. Statement reuse scope

Debe respetar:

```text
connection
transaction
statement lifecycle
driver validity
```

---

# 152. No cross-connection statement reuse

Un statement preparado en:

```text
Connection A
```

no se reutiliza en:

```text
Connection B
```

---

# 153. Read/write routing

Todas las operaciones de batch persistence son write intent.

---

# 154. Write target

Se resuelve downstream mediante las reglas de conexión/routing.

---

# 155. Single target group

Un batch no mezclará:

```text
database target A
database target B
```

---

# 156. Tenant isolation

Tampoco:

```text
tenant A
tenant B
```

---

# 157. Shard isolation

Tampoco:

```text
shard A
shard B
```

---

# 158. BatchCompatibilityKey

Puede incluir:

```text
PersistenceTarget
+
IdentityNamespace
+
EntityIdentityDomain
+
OperationShape
+
TransactionContext
```

---

# 159. Cross-target grouping

Prohibido.

---

# 160. PersistencePlan integration

El batching debe ocurrir después de que el logical plan conozca dependencias.

---

# 161. Recommended pipeline

```text
PersistencePlanner
      ↓
LogicalPersistencePlan
      ↓
BatchPersistenceOptimizer
      ↓
PhysicalPersistencePlan
      ↓
PersistenceExecutor
```

---

# 162. Alternative

El planner puede tener una fase física interna.

Aun así deben mantenerse conceptos separados.

---

# 163. Why separate

Permite:

```text
disable batching
compare plans
test semantics
benchmark strategies
explain optimizer decisions
```

---

# 164. Batching disabled

Debe ser posible:

```php
'batch_persistence' => [
    'enabled' => false,
],
```

---

# 165. Semantic equivalence test

Con batching on/off:

```text
Final ORM semantic result
```

deberá ser equivalente.

---

# 166. Determinism

Para mismas:

```text
operations
capabilities
configuration
```

el Batch Optimizer deberá producir un plan determinista.

---

# 167. No timing-based grouping

No agrupar entidades simplemente porque llegaron:

```text
within 5 milliseconds
```

durante un flush.

---

# 168. Flush plan is closed

Batching trabaja sobre operaciones ya estabilizadas.

---

# 169. Streaming batch execution

Para grandes planes, materializar todos los parámetros puede consumir demasiada memoria.

---

# 170. Batch parameter stream

Podrá existir:

```php
interface BatchParameterSource
{
    public function batches(): iterable;
}
```

---

# 171. But

El logical plan deberá conservar información suficiente para:

```text
dependency
correlation
reconciliation
failure reporting
```

---

# 172. Streaming constraints

No podrá liberar una operación antes de conservar la evidencia necesaria para reconciliarla.

---

# 173. Memory budget

```php
final readonly class BatchMemoryBudget
{
    public function __construct(
        public int $maxEstimatedBytes,
    ) {}
}
```

---

# 174. Effective size by memory

```text
BatchSize
≤
MemoryBudget / EstimatedBytesPerItem
```

---

# 175. Large BLOBs

Los BLOBs pueden hacer que:

```text
100 rows
```

sea un batch excesivo.

---

# 176. Value-size estimation

El sizing podrá considerar:

```text
string lengths
binary lengths
JSON payload size
LOB strategy
```

---

# 177. LOB handling

LOBs pueden requerir:

```text
streaming
individual execution
special driver binding
```

---

# 178. LOB batch eligibility

Será capability-driven.

---

# 179. Batch reconciliation

Después de ejecución:

```text
BatchExecutionOutcome
      ↓
BatchOutcomeCorrelator
      ↓
PersistenceOperationOutcome[]
      ↓
FlushReconciler
```

---

# 180. Key principle

El batch desaparece como optimización física antes de la reconciliación semántica final.

---

# 181. Semantic expansion

```text
Batch Outcome
      ↓
Individual Persistence Outcomes
```

cuando la evidencia lo permita.

---

# 182. Unknown expansion

Si no:

```text
Batch UNKNOWN
→ affected individual operations UNKNOWN
```

---

# 183. Insert reconciliation

Para cada insert exitoso:

```text
generated values
→ entity
→ EntityKey
→ IdentityMap
→ EntityState
→ Snapshot
```

---

# 184. Update reconciliation

Por item:

```text
persisted baseline
generated version
snapshot
clean/dirty state
```

---

# 185. Delete reconciliation

Por item:

```text
EntityState
IdentityMap removal
Snapshot removal
```

---

# 186. Partial batch reconciliation

Solo operaciones con outcomes suficientemente ciertos se reconciliarán como exitosas.

---

# 187. No optimistic cleanup

No marcar todo el batch como limpio si solo conocemos:

```text
some operation failed
```

---

# 188. Uncertain reconciliation

Puede taint:

```text
EntityManager
```

de acuerdo con Flush Failure Policy.

---

# 189. Lifecycle after batch

El orden lógico de `post*` deberá permanecer determinista.

---

# 190. Physical order vs lifecycle order

Puede definirse:

```text
post lifecycle order
=
logical operation order
```

cuando sea posible.

---

# 191. Failure case

Solo operaciones con semantic success reciben success lifecycle events.

---

# 192. Batch partial failure

Ejemplo:

```text
A success
B conflict
C not executed
```

produce:

```text
postUpdate(A)
no postUpdate(B)
no postUpdate(C)
```

---

# 193. Multi-row atomic failure

Si un statement multi-row falla atómicamente:

```text
all items failed/not applied
```

según evidencia de plataforma.

---

# 194. Unknown atomicity

Si atomicity no puede determinarse:

```text
UNKNOWN
```

se conserva.

---

# 195. Batch and snapshots

Batching nunca reemplaza:

```text
per-entity snapshot
```

por un único batch snapshot.

---

# 196. Batch and IdentityMap

No existe:

```text
BatchIdentityMap
```

---

# 197. Batch and EntityState

No existe estado ORM:

```text
BATCHED
```

---

# 198. Why

`BATCHED` es estrategia física, no lifecycle/state de entidad.

---

# 199. Batch and ChangeSet

Cada entidad conserva su ChangeSet.

---

# 200. Batch group references ChangeSets

No los fusiona semánticamente.

---

# 201. Extensions

Un custom persistence operation podrá declarar:

```text
batchable
```

solo mediante contrato explícito.

---

# 202. BatchablePersistenceOperation

```php
interface BatchablePersistenceOperation
{
    public function batchDescriptor(): BatchOperationDescriptor;
}
```

---

# 203. Default custom operation

```text
NOT_BATCHABLE
```

---

# 204. Extension safety

Una extensión deberá declarar:

```text
compatibility key
parameter shape
result correlation
failure semantics
generated-value semantics
```

---

# 205. Incomplete extension contract

Resultado:

```text
individual execution
```

no guessing.

---

# 206. Extension collision

Dos extensiones no podrán reclamar de manera incompatible la misma estrategia.

---

# 207. Registry

```text
BatchStrategyRegistry
```

será frozen en producción.

---

# 208. Persistent runtime

VoltStack debe soportar:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 209. Shared immutable state

Puede compartirse:

```text
batch strategy definitions
compiled operation shapes
immutable capability descriptors
configuration snapshots
```

---

# 210. Scoped mutable state

Siempre:

```text
BatchGroup
BatchExecutionContext
BatchOutcome
GeneratedValueCorrelation
BatchStatistics for current flush
parameter buffers
entity references
tenant context
transaction context
```

---

# 211. No process-global pending batch

Prohibido:

```php
static array $pendingBatch;
```

---

# 212. Why

Podría mezclar:

```text
Request A
Request B
Tenant A
Tenant B
```

---

# 213. FrankenPHP

Al finalizar request:

```text
batch execution state
```

debe liberarse.

---

# 214. RoadRunner

Igual.

---

# 215. OpenSwoole

Cada coroutine/contexto lógico deberá permanecer aislado.

---

# 216. Concurrent execution

Un mismo EntityManager no deberá ejecutar batches concurrentemente sin soporte explícito.

---

# 217. Parallel batches

En el futuro podrían ejecutarse batches independientes en paralelo.

Pero requeriría:

```text
dependency independence
transaction compatibility
connection isolation
deterministic reconciliation
thread/coroutine-safe context
```

---

# 218. V1

No paralelizar batches de un mismo flush.

---

# 219. Reason

La latencia ganada puede no justificar:

```text
state races
generated-value races
transaction complexity
reconciliation ordering complexity
```

---

# 220. Telemetry

Métricas:

```text
orm.batch.total
orm.batch.items
orm.batch.insert
orm.batch.update
orm.batch.delete

orm.batch.size
orm.batch.duration

orm.batch.strategy.individual
orm.batch.strategy.prepared_reuse
orm.batch.strategy.multi_row
orm.batch.strategy.driver_native

orm.batch.fallback
orm.batch.partial
orm.batch.unknown
orm.batch.retry

orm.batch.parameters
orm.batch.estimated_bytes

orm.batch.generated_values
orm.batch.generated_value_fallback

orm.batch.optimistic_conflicts
```

---

# 221. Optimization ratio

```text
BatchCompressionRatio
=
LogicalOperationCount
/
PhysicalExecutionCount
```

Ejemplo:

```text
1000 logical inserts
10 statements
ratio = 100
```

---

# 222. Round-trip reduction

```text
RoundTripReduction
=
1 -
(
PhysicalRoundTrips
/
UnbatchedRoundTrips
)
```

---

# 223. Batch hit ratio

```text
BatchEligibleOperations
/
CandidateOperations
```

---

# 224. Telemetry dimensions

Labels seguros pueden incluir:

```text
operation type
strategy
platform family
outcome
```

con cardinalidad controlada.

---

# 225. No entity IDs

No:

```text
user_id=948271
```

como metric label.

---

# 226. No tenant IDs

No por defecto.

---

# 227. Trace

```text
orm.flush
 └── persistence.execute
      ├── batch.insert
      │    ├── compile
      │    ├── bind
      │    ├── execute
      │    └── correlate
      ├── batch.update
      └── batch.delete
```

---

# 228. Debug Toolbar

Ejemplo:

```text
Persistence Batching

Logical operations:      1,240
Physical executions:        18
Compression ratio:        68.9x

INSERT
  logical:               1000
  batches:                 10
  avg batch:              100

UPDATE
  logical:                200
  batches:                  5
  strategy: prepared reuse

DELETE
  logical:                 40
  batches:                  3

Fallbacks:
  generated ID safety:      2
  optimistic locking:       1

Largest batch:
  100 rows
  600 parameters
```

---

# 229. Diagnostics

El sistema podrá explicar:

```text
Why wasn't this operation batched?
```

Ejemplo:

```text
Operation P-104

Batching:
  rejected

Reason:
  generated values cannot be correlated safely

Fallback:
  individual execution
```

---

# 230. Security

Batching no debe debilitar:

```text
parameter binding
tenant isolation
write routing
authorization boundaries
sensitive-data policies
```

---

# 231. Parameter safety

Los valores siguen siendo:

```text
typed parameters
```

No concatenación manual.

---

# 232. SQL injection

Multi-row generation continúa pasando por SQL Compiler.

---

# 233. Tenant mixing

BatchGroup deberá validar:

```text
same effective persistence target
```

---

# 234. Sensitive telemetry

No registrar valores de parámetros sin política explícita.

---

# 235. Failure model

Errores propuestos:

```text
DatabaseBatchPersistenceException
├── BatchOptimizationException
├── BatchEligibilityException
├── BatchCompatibilityException
├── BatchGroupingException
├── BatchShapeException
├── BatchSizingException
├── BatchParameterLimitException
├── BatchPayloadLimitException
├── BatchCapabilityException
├── BatchStrategyResolutionException
├── BatchExecutionException
├── BatchPartialExecutionException
├── BatchOutcomeUnknownException
├── BatchOutcomeCorrelationException
├── BatchGeneratedValueException
├── BatchGeneratedValueCorrelationException
├── BatchOptimisticLockException
├── BatchDependencyException
├── BatchTargetMismatchException
├── BatchTenantMismatchException
├── BatchShardMismatchException
├── BatchRetrySafetyException
├── BatchReconciliationException
├── BatchExtensionException
├── BatchConcurrentAccessException
├── BatchRuntimeIsolationException
└── BatchPersistenceInvariantException
```

---

# 236. Testing Strategy

Debe existir una matriz de pruebas amplia.

---

# 237. Insert grouping

Múltiples INSERT compatibles producen batch.

---

# 238. Different insert shape

No se agrupan.

---

# 239. Different table

No se agrupan.

---

# 240. Different tenant

No se agrupan.

---

# 241. Different shard

No se agrupan.

---

# 242. Different database target

No se agrupan.

---

# 243. Dependency boundary

Parent y child con generated dependency respetan ordering.

---

# 244. Independent children

Pueden agruparse después del parent.

---

# 245. Client-generated IDs

Permiten batch cuando las demás condiciones se cumplen.

---

# 246. Server-generated IDs

Solo batch si la correlación es segura.

---

# 247. Unsafe generated ID

Fallback a individual.

---

# 248. Multi-row RETURNING

Correlaciona correctamente cada resultado.

---

# 249. Prepared statement reuse

Mismo statement, diferentes parámetros.

---

# 250. Update same shape

Se agrupa.

---

# 251. Update different shape

Se separa.

---

# 252. Optimistic update

Cada outcome permanece correlacionable.

---

# 253. Unsafe optimistic batch

Fallback.

---

# 254. Delete batching

Respeta semantic outcomes.

---

# 255. Join table inserts

Batch correcto.

---

# 256. Lifecycle

Batch de N entidades produce N lifecycle success events cuando corresponde.

---

# 257. pre lifecycle

No se ejecuta dentro del Batch Optimizer.

---

# 258. post lifecycle ordering

Determinista.

---

# 259. Batch sizing

Respeta configured maximum.

---

# 260. Parameter limit

Divide batch antes de exceder límite.

---

# 261. Unknown parameter limit

Usa conservative default.

---

# 262. Payload limit

Divide batch.

---

# 263. Large BLOB

Reduce tamaño o fallback.

---

# 264. Empty group

No produce statement.

---

# 265. Single item

Puede utilizar individual o prepared strategy sin cambiar semántica.

---

# 266. Partial execution

Outcomes individuales correctos.

---

# 267. Unknown outcome

No inventa success.

---

# 268. Statement atomic failure

Se representa según capabilities.

---

# 269. Transaction rollback

Batch success no se interpreta como durable.

---

# 270. Joined transaction

Batch no hace commit.

---

# 271. Retry unknown

Rechazado.

---

# 272. Retry not-executed

Solo si policy y replay safety lo permiten.

---

# 273. Generated values

Se asignan a la entidad correcta.

---

# 274. IdentityMap

Batch insert no crea identidades duplicadas.

---

# 275. Snapshots

Cada entidad obtiene baseline correcto.

---

# 276. PostUpdate mutation

Permanece dirty después de reconciliación.

---

# 277. Batching disabled

Produce semántica equivalente.

---

# 278. Determinism

Mismo input produce mismos groups.

---

# 279. Capability fallback

SUPPORTED/WITH_LIMITATIONS/UNKNOWN se manejan correctamente.

---

# 280. MariaDB vs MySQL

No se asumen capacidades idénticas.

---

# 281. SQLite limits

Se respetan límites efectivos.

---

# 282. Persistent worker

Request A no comparte batches con B.

---

# 283. OpenSwoole isolation

No hay contaminación entre coroutines.

---

# 284. Memory budget

Large UnitOfWork no excede arbitrariamente el presupuesto.

---

# 285. Streaming

No pierde información de reconciliación.

---

# 286. Extension

Custom batch operation requiere descriptor completo.

---

# 287. Extension incomplete

Fallback a individual.

---

# 288. Telemetry

No incluye identificadores sensibles de alta cardinalidad.

---

# 289. Anti-patterns

## 289.1 Agrupar por entidad solamente

Incorrecto.

## 289.2 Agrupar todo el UnitOfWork

Incorrecto.

## 289.3 Batch = Bulk Query

Incorrecto.

## 289.4 Inferir generated IDs consecutivos

Peligroso.

## 289.5 Ignorar optimistic locking

Incorrecto.

## 289.6 Usar un affected-row-count global para reconciliar entidades individuales sin evidencia

Incorrecto.

## 289.7 Batch cross-tenant

Prohibido.

## 289.8 Batch cross-shard

Prohibido.

## 289.9 Batch cross-connection

Prohibido.

## 289.10 Ignorar parameter limits

Incorrecto.

## 289.11 UNKNOWN limit = unlimited

Incorrecto.

## 289.12 Maximum batch size hard-coded por vendor

Incorrecto.

## 289.13 Multi-row SQL generado directamente por ORM

Prohibido.

## 289.14 Bypass del SQL Compiler

Prohibido.

## 289.15 Un lifecycle callback por batch en lugar de por entidad

Incorrecto.

## 289.16 Marcar todas las entidades clean tras partial failure

Incorrecto.

## 289.17 Retry automático de UNKNOWN batch

Peligroso.

## 289.18 Batch success = commit

Incorrecto.

## 289.19 Batch failure = rollback

Incorrecto.

## 289.20 Guardar batches pendientes en static state

Prohibido.

---

# 290. Architectural Invariants

## DB-ORM-BATCH-001
Batch Persistence será una optimización física.

## DB-ORM-BATCH-002
Batch Persistence no modificará la semántica ORM.

## DB-ORM-BATCH-003
Logical PersistencePlan será la fuente semántica de verdad.

## DB-ORM-BATCH-004
Physical PersistencePlan preservará las operaciones lógicas.

## DB-ORM-BATCH-005
Batch Persistence será distinto de Bulk Query.

## DB-ORM-BATCH-006
Batch Persistence será distinto de Flush.

## DB-ORM-BATCH-007
Batch Persistence será distinto de UnitOfWork.

## DB-ORM-BATCH-008
Batch Persistence será distinto de Transaction.

## DB-ORM-BATCH-009
Batch Persistence será distinto del SQL Compiler.

## DB-ORM-BATCH-010
Batch Group no será una transacción.

## DB-ORM-BATCH-011
Batch success no implicará commit.

## DB-ORM-BATCH-012
Batch failure no implicará rollback.

## DB-ORM-BATCH-013
Cada batch tendrá identidad diagnóstica.

## DB-ORM-BATCH-014
Cada operación conservará PersistenceOperationId.

## DB-ORM-BATCH-015
Cada batch item conservará correlación con su operación lógica.

## DB-ORM-BATCH-016
BatchShape será explícito.

## DB-ORM-BATCH-017
Shape fingerprint no sustituirá validación estructural.

## DB-ORM-BATCH-018
Same table no implicará same batch.

## DB-ORM-BATCH-019
Same entity type no implicará same batch.

## DB-ORM-BATCH-020
Different operation types no se mezclarán.

## DB-ORM-BATCH-021
Different targets no se mezclarán.

## DB-ORM-BATCH-022
Different tenants no se mezclarán.

## DB-ORM-BATCH-023
Different shards no se mezclarán.

## DB-ORM-BATCH-024
Different identity namespaces no se mezclarán cuando afecten persistencia.

## DB-ORM-BATCH-025
Batch eligibility será explícita.

## DB-ORM-BATCH-026
Batch eligibility considerará semántica.

## DB-ORM-BATCH-027
Batch eligibility considerará dependencias.

## DB-ORM-BATCH-028
Batch eligibility considerará capabilities.

## DB-ORM-BATCH-029
Batch eligibility considerará outcome correlation.

## DB-ORM-BATCH-030
Batch eligibility considerará generated-value correlation.

## DB-ORM-BATCH-031
Batch eligibility considerará optimistic locking.

## DB-ORM-BATCH-032
Unknown semantic compatibility implicará no batching.

## DB-ORM-BATCH-033
Dependency boundaries serán respetados.

## DB-ORM-BATCH-034
Batch groups no romperán topological ordering.

## DB-ORM-BATCH-035
Generated-value dependencies serán preservadas.

## DB-ORM-BATCH-036
Ordering será determinista.

## DB-ORM-BATCH-037
Multi-row INSERT será una estrategia física.

## DB-ORM-BATCH-038
Batch System no generará SQL directamente.

## DB-ORM-BATCH-039
Multi-row operations pasarán por Query Model.

## DB-ORM-BATCH-040
SQL Compiler seguirá siendo responsable del SQL.

## DB-ORM-BATCH-041
Generated IDs no se inferirán universalmente como secuenciales.

## DB-ORM-BATCH-042
Generated-value correlation deberá ser demostrable.

## DB-ORM-BATCH-043
Unsafe generated-value correlation deshabilitará esa estrategia.

## DB-ORM-BATCH-044
Client-generated ID no probará row existence.

## DB-ORM-BATCH-045
Server-generated values se reconciliarán con evidencia suficiente.

## DB-ORM-BATCH-046
RETURNING será capability-driven.

## DB-ORM-BATCH-047
Prepared statement reuse será una estrategia válida.

## DB-ORM-BATCH-048
Driver-native batching requerirá contrato explícito.

## DB-ORM-BATCH-049
Driver batching será distinto de ORM batching.

## DB-ORM-BATCH-050
UPDATE batching preservará ChangeSets individuales.

## DB-ORM-BATCH-051
Different UPDATE shapes producirán grupos diferentes.

## DB-ORM-BATCH-052
Optimistic locking será preservado por entidad.

## DB-ORM-BATCH-053
Un global affected-row-count insuficiente no será usado para inventar optimistic outcomes.

## DB-ORM-BATCH-054
Unsafe optimistic batching degradará de estrategia.

## DB-ORM-BATCH-055
DELETE batching preservará semantic outcomes.

## DB-ORM-BATCH-056
Relationship batching no decidirá cascade semantics.

## DB-ORM-BATCH-057
Relationship batching no decidirá orphan semantics.

## DB-ORM-BATCH-058
Entity lifecycle seguirá siendo por entidad.

## DB-ORM-BATCH-059
Batch operational events serán distintos de entity lifecycle.

## DB-ORM-BATCH-060
Pre-lifecycle habrá sido procesado antes del physical batching.

## DB-ORM-BATCH-061
Batch optimization no mutará entidades.

## DB-ORM-BATCH-062
Batch size será bounded.

## DB-ORM-BATCH-063
Batch sizing considerará parameter limits.

## DB-ORM-BATCH-064
Batch sizing considerará payload limits.

## DB-ORM-BATCH-065
Batch sizing considerará memory limits.

## DB-ORM-BATCH-066
Batch sizing considerará semantic limits.

## DB-ORM-BATCH-067
Configured max no sustituirá platform limits.

## DB-ORM-BATCH-068
Unknown platform limit no significará unlimited.

## DB-ORM-BATCH-069
Effective batch size será el mínimo de límites aplicables.

## DB-ORM-BATCH-070
Batch boundaries no cambiarán PersistenceOperationId.

## DB-ORM-BATCH-071
Batch execution preservará item ordering/correlation cuando sea requerido.

## DB-ORM-BATCH-072
BatchExecutionStatus será explícito.

## DB-ORM-BATCH-073
BatchItemStatus será explícito cuando pueda conocerse.

## DB-ORM-BATCH-074
Partial batch execution será first-class.

## DB-ORM-BATCH-075
UNKNOWN batch outcome será first-class.

## DB-ORM-BATCH-076
No se inventará per-row certainty.

## DB-ORM-BATCH-077
Statement atomicity será distinta de transaction commit.

## DB-ORM-BATCH-078
Batch atomicity será modelada explícitamente.

## DB-ORM-BATCH-079
Transaction integration será coordinada externamente.

## DB-ORM-BATCH-080
Batch executor no hará commit de joined transaction.

## DB-ORM-BATCH-081
Batch executor no afirmará rollback sin evidencia.

## DB-ORM-BATCH-082
Unknown batch no se reintentará ciegamente.

## DB-ORM-BATCH-083
Retry requerirá replay safety.

## DB-ORM-BATCH-084
Succeeded items no serán reintentados sin rollback cierto.

## DB-ORM-BATCH-085
Unique IDs no implicarán automáticamente retry safety.

## DB-ORM-BATCH-086
Strategy resolver seleccionará la estrategia más eficiente que siga siendo segura.

## DB-ORM-BATCH-087
Strategy fallback será permitido.

## DB-ORM-BATCH-088
Fallback no cambiará semántica.

## DB-ORM-BATCH-089
Capability detection será first-class.

## DB-ORM-BATCH-090
Vendor conditionals no gobernarán la arquitectura central.

## DB-ORM-BATCH-091
MySQL y MariaDB no se tratarán como capacidades idénticas.

## DB-ORM-BATCH-092
Version será distinta de capability.

## DB-ORM-BATCH-093
Capability UNKNOWN no será SUPPORTED.

## DB-ORM-BATCH-094
Capability snapshot será estable durante el plan físico.

## DB-ORM-BATCH-095
Prepared statement reuse respetará connection scope.

## DB-ORM-BATCH-096
Statements no se reutilizarán cross-connection.

## DB-ORM-BATCH-097
Persistence batching tendrá write intent.

## DB-ORM-BATCH-098
Un batch tendrá un único effective persistence target.

## DB-ORM-BATCH-099
Batching ocurrirá después de dependency planning.

## DB-ORM-BATCH-100
Batching podrá deshabilitarse.

## DB-ORM-BATCH-101
Batching enabled/disabled deberá preservar equivalencia semántica.

## DB-ORM-BATCH-102
Batch optimization será determinista.

## DB-ORM-BATCH-103
Batch grouping no dependerá de timing accidental.

## DB-ORM-BATCH-104
Streaming batch execution preservará correlation metadata.

## DB-ORM-BATCH-105
Streaming no liberará evidencia necesaria antes de reconciliation.

## DB-ORM-BATCH-106
Memory budget será explícito.

## DB-ORM-BATCH-107
Large values podrán reducir batch size.

## DB-ORM-BATCH-108
LOB batching será capability-driven.

## DB-ORM-BATCH-109
Batch outcomes se expandirán a semantic operation outcomes cuando sea posible.

## DB-ORM-BATCH-110
Unknown batch outcomes propagarán incertidumbre a operaciones afectadas.

## DB-ORM-BATCH-111
Insert reconciliation seguirá siendo per entity.

## DB-ORM-BATCH-112
Update reconciliation seguirá siendo per entity.

## DB-ORM-BATCH-113
Delete reconciliation seguirá siendo per entity.

## DB-ORM-BATCH-114
Partial reconciliation solo marcará éxitos suficientemente ciertos.

## DB-ORM-BATCH-115
No se limpiará todo un batch ante outcome parcial.

## DB-ORM-BATCH-116
IdentityMap no tendrá semántica especial de batch.

## DB-ORM-BATCH-117
EntityState no tendrá estado BATCHED.

## DB-ORM-BATCH-118
Snapshots seguirán siendo per entity.

## DB-ORM-BATCH-119
ChangeSets seguirán siendo per entity.

## DB-ORM-BATCH-120
Custom operations serán non-batchable por defecto.

## DB-ORM-BATCH-121
Custom batching requerirá descriptor explícito.

## DB-ORM-BATCH-122
Incomplete extension semantics degradarán a individual execution.

## DB-ORM-BATCH-123
Batch strategy registry podrá congelarse en producción.

## DB-ORM-BATCH-124
Mutable batch state será scoped.

## DB-ORM-BATCH-125
No existirá process-global pending batch.

## DB-ORM-BATCH-126
Request A no compartirá batch con Request B.

## DB-ORM-BATCH-127
Tenant A no compartirá batch con Tenant B.

## DB-ORM-BATCH-128
Coroutine scopes estarán aislados.

## DB-ORM-BATCH-129
Un mismo EntityManager no ejecutará batches concurrentes en V1.

## DB-ORM-BATCH-130
V1 no paralelizará batches dentro del mismo flush.

## DB-ORM-BATCH-131
Telemetry distinguirá logical operations de physical executions.

## DB-ORM-BATCH-132
Telemetry medirá batch size.

## DB-ORM-BATCH-133
Telemetry medirá strategy fallback.

## DB-ORM-BATCH-134
Telemetry medirá partial outcomes.

## DB-ORM-BATCH-135
Telemetry medirá unknown outcomes.

## DB-ORM-BATCH-136
Telemetry no expondrá entity IDs como labels.

## DB-ORM-BATCH-137
Telemetry no expondrá parameter values por defecto.

## DB-ORM-BATCH-138
Diagnostics explicarán por qué una operación no fue batchable.

## DB-ORM-BATCH-139
Batching no debilitará parameter binding.

## DB-ORM-BATCH-140
Batching no debilitará tenant isolation.

## DB-ORM-BATCH-141
Batching no bypassará SQL Compiler.

## DB-ORM-BATCH-142
Batching no bypassará write routing.

## DB-ORM-BATCH-143
Batch optimization failure podrá degradar de manera segura cuando corresponda.

## DB-ORM-BATCH-144
Semantic uncertainty no se resolverá a favor del rendimiento.

## DB-ORM-BATCH-145
Correctness tendrá prioridad sobre batch size.

## DB-ORM-BATCH-146
Correctness tendrá prioridad sobre round-trip reduction.

## DB-ORM-BATCH-147
Generated-value safety tendrá prioridad sobre multi-row execution.

## DB-ORM-BATCH-148
Optimistic-lock safety tendrá prioridad sobre multi-row execution.

## DB-ORM-BATCH-149
Outcome certainty tendrá prioridad sobre throughput.

## DB-ORM-BATCH-150
Batch execution conservará relación entre physical execution y logical operations.

## DB-ORM-BATCH-151
Batch plan no redefinirá EntityKey.

## DB-ORM-BATCH-152
Batch plan no redefinirá EntityState.

## DB-ORM-BATCH-153
Batch plan no redefinirá lifecycle.

## DB-ORM-BATCH-154
Batch plan no redefinirá transaction ownership.

## DB-ORM-BATCH-155
Batch plan no redefinirá relationship ownership.

## DB-ORM-BATCH-156
Batch plan no redefinirá ChangeSet.

## DB-ORM-BATCH-157
Batch plan no redefinirá persistence dependencies.

## DB-ORM-BATCH-158
Physical grouping será reversible conceptualmente hacia logical operations.

## DB-ORM-BATCH-159
Toda reconciliación deberá poder razonar sobre las operaciones lógicas originales.

## DB-ORM-BATCH-160
VoltStack solo aplicará batching cuando pueda demostrar que la optimización física conserva identidad, dependencias, lifecycle, generated values, locking, failure semantics, transaction boundaries y reconciliación de cada operación ORM involucrada.

---

# 291. Estructura propuesta

```text
src/Quantum/Database/ORM/Persistence/Batch/
│
├── Contract/
│   ├── BatchPersistenceOptimizer.php
│   ├── BatchEligibilityAnalyzer.php
│   ├── BatchCompatibilityAnalyzer.php
│   ├── BatchSizer.php
│   ├── BatchExecutionStrategyResolver.php
│   ├── BatchOutcomeCorrelator.php
│   └── BatchReconciler.php
│
├── Model/
│   ├── BatchGroup.php
│   ├── BatchGroupId.php
│   ├── BatchItem.php
│   ├── BatchItemIndex.php
│   ├── BatchShape.php
│   ├── BatchShapeFingerprint.php
│   ├── BatchOperationType.php
│   └── BatchCompatibilityKey.php
│
├── Eligibility/
│   ├── DefaultBatchEligibilityAnalyzer.php
│   ├── BatchEligibilityResult.php
│   ├── BatchEligibilityReason.php
│   └── BatchEligibilityReasonCollection.php
│
├── Grouping/
│   ├── BatchGroupBuilder.php
│   ├── BatchGroupCandidate.php
│   ├── BatchDependencyAnalyzer.php
│   └── BatchGroupingResult.php
│
├── Strategy/
│   ├── BatchExecutionStrategy.php
│   ├── BatchExecutionStrategyDecision.php
│   ├── IndividualExecutionStrategy.php
│   ├── PreparedStatementReuseStrategy.php
│   ├── MultiRowExecutionStrategy.php
│   ├── DriverNativeBatchStrategy.php
│   └── BatchStrategyRegistry.php
│
├── Insert/
│   ├── BatchInsertExecutor.php
│   ├── MultiRowInsertPlanBuilder.php
│   └── InsertBatchDescriptor.php
│
├── Update/
│   ├── BatchUpdateExecutor.php
│   ├── UpdateBatchDescriptor.php
│   └── OptimisticBatchUpdateAnalyzer.php
│
├── Delete/
│   ├── BatchDeleteExecutor.php
│   └── DeleteBatchDescriptor.php
│
├── Generated/
│   ├── GeneratedValueCorrelationStrategy.php
│   ├── BatchGeneratedValueCorrelator.php
│   ├── BatchGeneratedValueResult.php
│   └── GeneratedValueCorrelationEvidence.php
│
├── Sizing/
│   ├── DefaultBatchSizer.php
│   ├── BatchSizingContext.php
│   ├── BatchSizingDecision.php
│   ├── BatchMemoryBudget.php
│   ├── BatchParameterBudget.php
│   └── BatchPayloadBudget.php
│
├── Execution/
│   ├── BatchExecutionContext.php
│   ├── BatchExecutionOutcome.php
│   ├── BatchExecutionStatus.php
│   ├── BatchItemOutcome.php
│   ├── BatchItemStatus.php
│   ├── BatchAtomicity.php
│   └── BatchExecutionEvidence.php
│
├── Reconciliation/
│   ├── DefaultBatchOutcomeCorrelator.php
│   ├── DefaultBatchReconciler.php
│   └── BatchReconciliationResult.php
│
├── Retry/
│   ├── BatchRetryAnalyzer.php
│   ├── BatchRetryDecision.php
│   └── BatchReplaySafety.php
│
├── Extension/
│   ├── BatchablePersistenceOperation.php
│   ├── BatchOperationDescriptor.php
│   └── BatchPersistenceExtension.php
│
├── Runtime/
│   ├── BatchRuntimeContext.php
│   └── BatchRuntimeResetter.php
│
├── Telemetry/
│   ├── BatchPersistenceTelemetry.php
│   ├── BatchOptimizationStatistics.php
│   └── BatchDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 292. Fórmulas fundamentales

## 292.1 Batch eligibility

```text
BatchEligible(op)
=
SemanticCompatibility(op)
∧
DependencyCompatibility(op)
∧
TargetCompatibility(op)
∧
CapabilitySupport(op)
∧
OutcomeCorrelationSafe(op)
∧
GeneratedValueCorrelationSafe(op)
∧
LockingSemanticsPreserved(op)
```

---

# 293. Batch compatibility

Para operaciones `a` y `b`:

```text
Compatible(a,b)
=
SameOperationType
∧
SamePersistenceTarget
∧
SameIdentityNamespace
∧
CompatibleShape
∧
NoDependencyViolation
∧
CompatibleResultRequirements
```

---

# 294. Batch group

```text
BatchGroup
=
{
    op ∈ PersistencePlan
    |
    Compatible(op, GroupShape)
}
```

respetando dependency layers.

---

# 295. Effective batch size

```text
EffectiveBatchSize
=
min(
    ConfiguredMax,
    PlatformMax,
    DriverMax,
    ParameterDerivedMax,
    PayloadDerivedMax,
    MemoryDerivedMax,
    SemanticMax
)
```

---

# 296. Parameter-derived size

```text
ParameterDerivedMax
=
floor(
    (MaxParameters - ReservedParameters)
    /
    ParametersPerItem
)
```

---

# 297. Batch compression

```text
BatchCompressionRatio
=
LogicalPersistenceOperations
/
PhysicalExecutions
```

---

# 298. Generated-value safety

```text
GeneratedValueBatchSafe
=
NoGeneratedValuesRequired
∨
ClientGeneratedValues
∨
PerItemGeneratedValuesCorrelatable
```

---

# 299. Optimistic locking safety

```text
OptimisticBatchSafe
=
NoOptimisticLocking
∨
PerOperationAffectedOutcomeCorrelatable
```

---

# 300. Safe strategy

```text
SelectedStrategy
=
argmax Efficiency(strategy)
```

subject to:

```text
SemanticSafety(strategy) = true
```

---

# 301. Batch semantic equivalence

```text
Semantics(
    Execute(BatchedPhysicalPlan)
)
=
Semantics(
    Execute(LogicalOperationsIndividually)
)
```

dentro de las garantías de ejecución declaradas.

---

# 302. Batch outcome

```text
BatchOutcome
=
Aggregate(
    ItemExecutionEvidence,
    StatementEvidence,
    TransactionContext,
    PlatformAtomicity
)
```

---

# 303. Partial batch

```text
PartialBatch
=
∃ a,b:
    Outcome(a) = SUCCEEDED
∧
Outcome(b) ∈ {
    FAILED,
    CONFLICT,
    NOT_EXECUTED,
    UNKNOWN
}
```

si los éxitos no fueron revertidos con certeza.

---

# 304. Unknown propagation

```text
BatchOutcome = UNKNOWN
∧
NoPerItemEvidence
⇒
AffectedItemOutcomes = UNKNOWN
```

---

# 305. Retry safety

```text
SafeBatchRetry
=
ExecutionOutcomeKnown
∧
TransactionOutcomeKnown
∧
ReplaySafe
∧
GeneratedValuesReconciled
∧
LifecycleSideEffectsSafe
```

---

# 306. Reconciliation

```text
BatchExecutionOutcome
→
IndividualPersistenceOutcome[]
→
FlushReconciliation
```

---

# 307. Persistent runtime safety

```text
SafeBatchRuntime
=
ScopedBatchState
∧
ScopedEntityReferences
∧
ScopedTransactionContext
∧
ScopedTenantContext
∧
NoProcessGlobalPendingBatch
∧
DeterministicReset
```

---

# 308. Arquitectura maestra

```text
                       UnitOfWork
                           │
                           ▼
                    Flush Stabilization
                           │
                           ▼
                   Persistence Planner
                           │
                           ▼
                 Logical PersistencePlan
                           │
                           ▼
               ┌───────────────────────┐
               │ Batch Eligibility     │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │ Dependency Analysis   │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │ Shape Grouping        │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │ Capability Analysis   │
               └──────────┬────────────┘
                          │
                          ▼
               ┌───────────────────────┐
               │ Batch Sizing          │
               └──────────┬────────────┘
                          │
                          ▼
                 Strategy Resolution
                 /        |          \
                /         |           \
          Individual   Prepared     Multi-row
                       Reuse
                \         |           /
                 \        |          /
                          ▼
                PhysicalPersistencePlan
                          │
                          ▼
                  Persistence Executor
                          │
                          ▼
                      Database
                          │
                          ▼
                  Execution Evidence
                          │
                          ▼
                  Outcome Correlator
                          │
                          ▼
              Individual Operation Outcomes
                          │
                          ▼
                    Flush Reconciler
              ┌───────────┼─────────────┐
              ▼           ▼             ▼
         IdentityMap   Snapshots     EntityState
              │           │             │
              └───────────┼─────────────┘
                          ▼
                   Lifecycle post*
```

---

# 309. Master Formula

```text
Database Batch Persistence System
=
Logical Persistence Operations
+
Batch Eligibility Analysis
+
Operation Shape Modeling
+
Compatibility Keys
+
Dependency-Aware Grouping
+
Deterministic Ordering
+
Persistence Target Isolation
+
Tenant Isolation
+
Shard Isolation
+
Capability-Aware Strategy Resolution
+
Prepared Statement Reuse
+
Multi-Row Persistence
+
Driver-Native Batch Integration
+
Generated Value Correlation
+
Generated Identity Safety
+
Optimistic Lock Preservation
+
Relationship Batch Support
+
Batch Sizing
+
Parameter Limit Governance
+
Payload Limit Governance
+
Memory Governance
+
LOB Awareness
+
Physical Plan Optimization
+
Per-Operation Correlation
+
Partial Failure Modeling
+
UNKNOWN Outcome Preservation
+
Atomicity Modeling
+
Transaction Boundary Preservation
+
Retry Safety
+
Per-Entity Reconciliation
+
Lifecycle Preservation
+
IdentityMap Preservation
+
Snapshot Preservation
+
ChangeSet Preservation
+
Persistent Runtime Isolation
+
Telemetry
+
Diagnostics
+
Extension Governance
```

---

# 310. Master Rule

> **En VoltStack, el Batch Persistence System podrá reducir cientos o miles de operaciones físicas mediante agrupación, reutilización de statements o instrucciones multi-row únicamente cuando pueda demostrar que la transformación conserva la semántica de cada operación lógica. Ninguna mejora de throughput tendrá prioridad sobre la identidad de las entidades, las dependencias del `PersistencePlan`, los valores generados, el optimistic locking, los lifecycle hooks, los límites transaccionales, la certeza de los resultados o la reconciliación correcta del `UnitOfWork`. Cuando esa equivalencia no pueda demostrarse, VoltStack deberá degradar a una estrategia física más conservadora, incluso hasta la ejecución individual.**

---

# 311. Estado del Bloque 11

Con este documento:

```text
123_DATABASE_IDENTITY_MAP_SYSTEM.md
124_DATABASE_UNIT_OF_WORK_ARCHITECTURE.md
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
127_DATABASE_PERSISTENCE_ENGINE.md
128_DATABASE_PERSISTENCE_PLANNER_SYSTEM.md
129_DATABASE_INSERT_PERSISTENCE_SYSTEM.md
130_DATABASE_UPDATE_PERSISTENCE_SYSTEM.md
131_DATABASE_DELETE_PERSISTENCE_SYSTEM.md
132_DATABASE_FLUSH_SYSTEM.md
133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md
```

La cadena queda:

```text
Entity
  ↓
EntityManager
  ↓
UnitOfWork
  ├── IdentityMap
  ├── ChangeTracking
  └── Snapshots
       ↓
     Flush
       ↓
Persistence Planner
       ↓
Logical PersistencePlan
       ↓
Batch Persistence Optimizer
       ↓
Physical PersistencePlan
       ↓
Persistence Engine
       ↓
Query Engine
       ↓
Execution Engine
       ↓
Database
       ↓
Execution Outcomes
       ↓
Reconciliation
```

Solo resta un documento para cerrar el **Bloque 11 — Identity Map, Unit of Work & Persistence**.

---

# 312. Siguiente documento

```text
134_DATABASE_PERSISTENCE_CONSISTENCY_SYSTEM.md
```

Este documento deberá cerrar el bloque formalizando la consistencia transversal entre:

```text
Entity State
IdentityMap
UnitOfWork
ChangeSets
Snapshots
PersistencePlan
Execution Outcomes
Database State
Generated Identities
Relationships
Transactions
Flush
Batch Persistence
Lifecycle
```

y deberá responder una cuestión crítica:

```text
¿Cuándo puede VoltStack afirmar realmente
que su representación ORM es consistente
con lo que se sabe de la base de datos?
```

Deberá definir, entre otros:

```text
consistency domains
ORM consistency model
in-memory consistency
identity consistency
snapshot consistency
change-set consistency
persistence-plan consistency
execution consistency
database-state knowledge
known vs unknown state
transaction-aware consistency
pre-commit consistency
post-commit consistency
rollback reconciliation
partial execution consistency
unknown execution outcomes
tainted persistence contexts
generated identity consistency
relationship consistency
batch consistency
external database mutations
bulk-operation invalidation
raw SQL invalidation
refresh and clear strategies
consistency recovery
reconciliation policies
consistency assertions
persistent-runtime isolation
failure semantics
telemetry
testing
```

La regla central deberá ser:

> **VoltStack nunca confundirá consistencia interna del ORM con certeza sobre el estado durable de la base de datos: cuando una operación, transacción o efecto externo tenga resultado desconocido, esa incertidumbre será representada explícitamente en lugar de fabricar una visión aparentemente consistente.**