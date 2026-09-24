# 208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md

# VoltStack Quantum Database
## Large Dataset Processing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 208 — Large Dataset Processing System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `207_DATABASE_EXPORT_SYSTEM.md`  
**Siguiente documento:** `209_DATABASE_EVENT_ARCHITECTURE.md`

---

# 1. Propósito

`Large Dataset Processing System` define la arquitectura superior mediante la cual VoltStack podrá ejecutar operaciones sobre datasets demasiado grandes para ser cargados, hidratados, transformados o persistidos completamente en memoria.

El sistema deberá coordinar:

```text
Pagination
Cursor Pagination
Chunk Processing
Lazy Collections
Bulk Insert
Bulk Update
Bulk Delete
Import
Export
Query Engine
Transactions
Read Routing
Sharding
ORM
Telemetry
Runtime
```

sin duplicar las responsabilidades de ninguno de ellos.

Ejemplo:

```php
$result = DB::largeDataset()
    ->from(
        DB::table('events')
            ->where('processed', false)
            ->orderBy('id')
    )
    ->batchSize(5_000)
    ->partitionBy('id')
    ->workers(8)
    ->checkpoint()
    ->process(function (LargeDatasetBatch $batch) {
        // procesamiento
    });
```

Desde Model API:

```php
User::query()
    ->where('legacy', true)
    ->processLargeDataset(
        batchSize: 1_000,
        handler: function (User $user) {
            // ...
        },
    );
```

Desde Repository:

```php
$repository
    ->queryForMigration()
    ->largeDataset()
    ->strategy(LargeDatasetStrategy::KEYSET)
    ->run($processor);
```

La regla central será:

> **Procesar un dataset grande en VoltStack no significa ejecutar una consulta enorme ni envolver un `foreach` alrededor de millones de entidades; significa construir una ejecución incremental, acotada, particionable, recuperable y gobernada por recursos sobre una definición estable del dataset.**

---

# 2. Posición arquitectónica

Este documento cierra:

```text
Block 19
Pagination, Batch & Large Data
```

compuesto por:

```text
199 Pagination
200 Cursor Pagination
201 Chunk Processing
202 Lazy Collection
203 Bulk Insert
204 Bulk Update
205 Bulk Delete
206 Import
207 Export
208 Large Dataset Processing
```

La relación será:

```text
                 Large Dataset Processing
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
    Traversal          Mutation            Transfer
       │                   │                   │
 ┌─────┼─────┐       ┌─────┼─────┐       ┌─────┴─────┐
 ▼     ▼     ▼       ▼     ▼     ▼       ▼           ▼
Chunk Lazy Cursor  Insert Update Delete  Import      Export
```

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
Large Dataset Processing
≠
Chunk Processing
≠
Pagination
≠
Cursor Pagination
≠
Lazy Collection
≠
Streaming Result
≠
Bulk Operation
≠
Import
≠
Export
≠
Job Queue
≠
Distributed Computing Platform
≠
Transaction
```

---

# 4. Large Dataset Processing ≠ Chunk Processing

Chunk Processing responde:

> ¿Cómo recorrer incrementalmente un dataset en unidades finitas?

Large Dataset Processing responde:

> ¿Cómo coordinar una operación completa, potencialmente larga, paralela, particionada y recuperable sobre ese dataset?

Por tanto:

```text
Large Dataset Processing
    ↓
may use
    ↓
Chunk Processing
```

pero:

```text
Large Dataset Processing
≠
Chunk Processing
```

---

# 5. Large Dataset Processing ≠ Pagination

Pagination está orientada a ventanas navegables.

Large Dataset Processing está orientado a:

```text
exhaustive processing
background operations
migrations
analytics
maintenance
transformation
data movement
```

---

# 6. Large Dataset Processing ≠ Lazy Collection

Lazy Collection ofrece consumo diferido.

Large Dataset Processing añade:

```text
partitioning
worker coordination
checkpoints
failure boundaries
resource governance
progress
recovery
```

---

# 7. Large Dataset Processing ≠ Queue

El sistema podrá integrarse con Jobs/Queues.

No deberá convertirse en otro sistema de colas.

```text
Large Dataset Planner
       ↓
Partition Plan
       ↓
Queue Integration
       ↓
Jobs
```

será posible posteriormente.

---

# 8. Large Dataset Processing ≠ Distributed Computing Platform

VoltStack Database podrá coordinar trabajo sobre datos distribuidos.

No intentará reemplazar sistemas como:

```text
Spark
Flink
Beam
```

Su alcance será el procesamiento database-centric del framework.

---

# 9. Objetivos

El sistema deberá proporcionar:

```text
bounded memory
incremental execution
deterministic traversal
partitioning
controlled parallelism
backpressure
checkpoints
resume
progress
cancellation
deadlines
failure isolation
retry policies
transaction policies
consistency policies
ORM memory governance
tenant isolation
shard awareness
runtime isolation
telemetry
diagnostics
```

---

# 10. Arquitectura general

```text
LargeDataset API
      │
      ▼
LargeDatasetRequest
      │
      ▼
LargeDatasetPlanner
      │
      ├── Source Analysis
      ├── Traversal Analysis
      ├── Partition Analysis
      ├── Consistency Analysis
      ├── Mutation Analysis
      ├── Transaction Analysis
      ├── Distribution Analysis
      ├── Resource Analysis
      └── Recovery Analysis
      │
      ▼
LargeDatasetPlan
      │
      ▼
LargeDatasetCoordinator
      │
      ├── Partition Coordinator
      ├── Worker Coordinator
      ├── Checkpoint Coordinator
      ├── Progress Coordinator
      ├── Resource Governor
      └── Cancellation Coordinator
      │
      ▼
Execution Units
      │
      ├── Chunk
      ├── Lazy
      ├── Stream
      ├── Bulk
      ├── Import
      ├── Export
      └── Custom
      │
      ▼
LargeDatasetResult
```

---

# 11. Principio de composición

El sistema deberá ser un:

```text
Coordinator
```

y no un:

```text
Second Query Engine
Second Bulk Engine
Second Import Engine
Second Export Engine
```

---

# 12. LargeDatasetRequest

Modelo conceptual:

```php
final readonly class LargeDatasetRequest
{
    public function __construct(
        public LargeDatasetSource $source,
        public LargeDatasetOperation $operation,
        public LargeDatasetOptions $options,
    ) {}
}
```

---

# 13. Source

```php
interface LargeDatasetSource
{
    public function query(
        LargeDatasetContext $context,
    ): QueryModel;
}
```

---

# 14. Operation

La operación describe qué se pretende hacer.

```php
interface LargeDatasetOperation
{
    public function characteristics(): OperationCharacteristics;
}
```

---

# 15. Operation categories

```php
enum LargeDatasetOperationType
{
    case READ;
    case TRANSFORM;
    case MUTATE;
    case IMPORT;
    case EXPORT;
    case AGGREGATE;
    case CUSTOM;
}
```

---

# 16. Operation characteristics

Deberá poder expresar:

```text
read-only
writes database
mutates ordering fields
deletes source rows
creates source rows
external side effects
idempotent
replay-safe
order-sensitive
partition-safe
```

---

# 17. UNKNOWN characteristics

Si el sistema no puede determinar una característica:

```text
UNKNOWN
```

no deberá convertirse automáticamente en:

```text
SAFE
```

---

# 18. Processing strategies

```php
enum LargeDatasetStrategy
{
    case AUTO;
    case CHUNK;
    case KEYSET;
    case LAZY;
    case STREAM;
    case BULK;
    case PARTITIONED;
    case CUSTOM;
}
```

---

# 19. AUTO

`AUTO` podrá elegir una estrategia basándose en:

```text
dataset shape
ordering
identifiers
mutation type
database capabilities
distribution topology
resource budget
consistency requirements
resume requirements
```

---

# 20. AUTO deberá ser explainable

Ejemplo:

```text
Selected Strategy:
    KEYSET

Reasons:
    stable identifier available
    dataset estimated > 10M rows
    source rows may be deleted
    resume requested
    offset traversal rejected
```

---

# 21. Dataset identity

Una ejecución durable deberá identificar qué dataset procesa.

```php
final readonly class DatasetIdentity
{
    public function __construct(
        public string $queryFingerprint,
        public string $orderingFingerprint,
        public string $domainFingerprint,
    ) {}
}
```

---

# 22. Query fingerprint

Será:

```text
semantic
canonical
parameter-aware
security-aware
```

y no simplemente:

```text
hash(SQL)
```

---

# 23. Dataset definition ≠ Dataset snapshot

Siempre:

```text
Same Dataset Definition
≠
Same Dataset Contents
```

---

# 24. Processing horizon

El sistema deberá modelar qué universo pretende recorrer.

```php
enum DatasetHorizon
{
    case LIVE;
    case CAPTURE_UPPER_BOUND;
    case SNAPSHOT;
    case FIXED_PARTITIONS;
    case CUSTOM;
}
```

---

# 25. LIVE

El conjunto puede cambiar durante la ejecución.

---

# 26. CAPTURE_UPPER_BOUND

Puede capturar:

```text
maximum ordering boundary
```

al comenzar.

---

# 27. Upper bound

Ejemplo:

```text
initial maximum id = 50,000,000
```

y procesar:

```text
id <= 50,000,000
```

---

# 28. Upper Bound ≠ Snapshot

Las filas existentes todavía pueden:

```text
change
disappear
```

---

# 29. SNAPSHOT

Busca mantener una vista lógica consistente.

Pero puede requerir:

```text
long-lived transaction
MVCC retention
connection pinning
database-specific capabilities
```

---

# 30. No implicit giant transaction

El sistema no deberá hacer automáticamente:

```text
BEGIN
process 100 million rows
COMMIT
```

---

# 31. Traversal model

La operación completa:

```text
Dataset
↓
Traversal
↓
Execution Units
↓
Processing
```

---

# 32. Execution Unit

Unidad mínima coordinada:

```php
final readonly class LargeDatasetExecutionUnit
{
    public function __construct(
        public ExecutionUnitId $id,
        public DatasetPartition $partition,
        public ?ContinuationBoundary $boundary,
        public ResourceBudget $budget,
    ) {}
}
```

---

# 33. Execution Unit ≠ Chunk

Una Execution Unit puede contener:

```text
multiple chunks
one shard partition
one key range
one import part
one export part
```

---

# 34. Partition architecture

Particionar significa dividir el universo lógico en subconjuntos procesables.

```text
Dataset D

D
├── P1
├── P2
├── P3
└── P4
```

Idealmente:

```text
P1 ∩ P2 = ∅
```

y:

```text
P1 ∪ P2 ∪ P3 ∪ P4 = D
```

bajo el horizon definido.

---

# 35. Partition strategies

```php
enum DatasetPartitionStrategy
{
    case NONE;
    case KEY_RANGE;
    case HASH;
    case SHARD;
    case TENANT;
    case TEMPORAL;
    case STATIC;
    case CUSTOM;
}
```

---

# 36. KEY_RANGE

Ejemplo:

```text
1          - 1,000,000
1,000,001  - 2,000,000
2,000,001  - 3,000,000
```

---

# 37. Range partitioning requirements

La key debe tener semántica adecuada para:

```text
ordering
comparison
boundaries
```

---

# 38. Gaps

Los IDs no necesitan ser contiguos.

```text
1
2
50
900
```

sigue siendo compatible con rangos.

---

# 39. Range size ≠ row count

Un rango:

```text
1..1,000,000
```

no garantiza:

```text
1,000,000 rows
```

---

# 40. HASH partitioning

Conceptualmente:

```text
hash(key) mod N
```

puede distribuir registros.

---

# 41. Hash partitioning tradeoff

Puede ofrecer distribución razonable, pero dificulta:

```text
natural ordering
range resume
human diagnostics
```

---

# 42. SHARD partitioning

Si la base ya está distribuida:

```text
Shard A
Shard B
Shard C
```

cada shard puede convertirse naturalmente en una partition.

---

# 43. TENANT partitioning

En operaciones administrativas autorizadas:

```text
Tenant A
Tenant B
Tenant C
```

pueden formar partitions independientes.

---

# 44. TEMPORAL partitioning

Ejemplo:

```text
2026-01
2026-02
2026-03
```

útil para:

```text
events
logs
transactions
history
```

---

# 45. Static partitions

La aplicación podrá definir:

```php
->partitions([
    $partitionA,
    $partitionB,
]);
```

---

# 46. Partition correctness

El planner deberá intentar demostrar:

```text
coverage
non-overlap
stable boundaries
```

cuando la estrategia lo requiera.

---

# 47. Partition confidence

```php
enum PartitionConfidence
{
    case PROVEN;
    case STRONGLY_INFERRED;
    case USER_DECLARED;
    case UNKNOWN;
}
```

---

# 48. UNKNOWN partition coverage

No deberá presentarse como:

```text
exact exhaustive processing
```

---

# 49. PartitionPlan

```php
final readonly class PartitionPlan
{
    public function __construct(
        public DatasetPartitionStrategy $strategy,
        public iterable $partitions,
        public PartitionConfidence $confidence,
    ) {}
}
```

---

# 50. Dynamic partition generation

No siempre será necesario materializar millones de partitions.

Podrá existir:

```text
PartitionGenerator
```

incremental.

---

# 51. Partition size

Deberá distinguirse:

```text
partition size
batch size
chunk size
bulk size
```

---

# 52. Ejemplo

```text
Partition:
    5,000,000 rows

Chunk:
    10,000 rows

Bulk mutation:
    2,000 rows
```

---

# 53. Worker architecture

```text
Coordinator
    │
    ├── Worker 1 → Partition A
    ├── Worker 2 → Partition B
    ├── Worker 3 → Partition C
    └── Worker 4 → Partition D
```

---

# 54. Worker ≠ OS process

Un worker lógico puede implementarse como:

```text
same-process task
coroutine
thread-like runtime unit
queue job
separate process
remote worker
```

según integración.

---

# 55. Core independence

Database core no dependerá de una implementación concreta de workers.

---

# 56. WorkerExecutor

```php
interface LargeDatasetWorkerExecutor
{
    public function execute(
        LargeDatasetExecutionUnit $unit,
        LargeDatasetExecutionContext $context,
    ): ExecutionUnitResult;
}
```

---

# 57. Worker count

```php
->workers(8)
```

será una policy de concurrencia, no una garantía de ocho conexiones simultáneas.

---

# 58. Concurrency governance

El Resource Governor podrá reducir concurrency si:

```text
connection pool exhausted
replica overloaded
memory pressure
destination slow
deadline pressure
```

---

# 59. Requested ≠ Effective parallelism

Siempre:

```text
Requested Parallelism
≠
Effective Parallelism
```

---

# 60. Bounded concurrency

No deberá existir:

```text
one worker per row
```

---

# 61. Work scheduling

Posibles políticas:

```php
enum WorkSchedulingPolicy
{
    case FIFO;
    case SMALLEST_FIRST;
    case LARGEST_FIRST;
    case SHARD_AFFINITY;
    case LOCALITY_AWARE;
    case CUSTOM;
}
```

---

# 62. Work stealing

Una futura implementación podrá permitir que workers libres tomen partitions pendientes.

---

# 63. Work stealing ≠ duplicate ownership

Debe existir ownership coordinado.

---

# 64. Partition ownership

Estado conceptual:

```text
AVAILABLE
↓
CLAIMED
↓
RUNNING
↓
COMPLETED
```

con ramas:

```text
FAILED
RETRYABLE
PAUSED
UNKNOWN
```

---

# 65. Claim token

```php
final readonly class PartitionClaim
{
    public function __construct(
        public PartitionId $partition,
        public ClaimToken $token,
        public WorkerId $worker,
        public ?Instant $expiresAt,
    ) {}
}
```

---

# 66. Lease-based ownership

Para workers distribuidos podrá utilizarse:

```text
claim
+
lease
+
heartbeat
```

---

# 67. Lease expiration ≠ worker definitely dead

Una pausa de red puede provocar expiration.

Por ello:

```text
Lease Expired
≠
Previous Worker Definitely Stopped
```

---

# 68. Fencing token

Para operaciones sensibles podrá utilizarse:

```text
monotonic fencing token
```

evitando que un worker antiguo continúe escribiendo después de perder ownership.

---

# 69. Fencing ≠ transaction isolation

Son mecanismos distintos.

---

# 70. Checkpoint architecture

El procesamiento durable podrá guardar:

```text
ProcessingId
PlanVersion
DatasetIdentity
PartitionId
ContinuationBoundary
OperationFingerprint
Tenant
Shard
TopologyGeneration
MetadataGeneration
ProcessedRecords
SuccessfulRecords
FailedRecords
Attempt
DestinationState
Timestamp
```

---

# 71. Checkpoint granularity

Puede ser:

```text
per operation
per partition
per chunk
per external output part
```

---

# 72. Recommended model

Para procesamiento particionado:

```text
Global Processing Checkpoint
        │
        ├── Partition A checkpoint
        ├── Partition B checkpoint
        └── Partition C checkpoint
```

---

# 73. Checkpoint ≠ progress

```text
Progress
=
observability

Checkpoint
=
recovery boundary
```

---

# 74. Progress may advance before checkpoint

Ejemplo:

```text
10,000 rows processed
checkpoint still at 8,000
```

---

# 75. Crash

Tras reinicio:

```text
rows 8,001..10,000
```

podrían repetirse.

---

# 76. At-least-once

Será una semántica común para processing resumable.

---

# 77. Exactly-once

No deberá prometerse automáticamente.

```text
Checkpoint
+
Retry
+
Transaction
≠
Exactly Once
```

---

# 78. External side effects

Ejemplo:

```php
foreach ($users as $user) {
    $mailer->send(...);
}
```

Si:

```text
email sent
↓
process crashes
↓
checkpoint not saved
```

el email puede repetirse.

---

# 79. Side-effect safety

Para efectos externos podrán requerirse:

```text
idempotency keys
outbox
deduplication
external transactional protocol
```

---

# 80. Database-only mutation

Puede ofrecer garantías más fuertes cuando:

```text
mutation
+
checkpoint
```

participan en la misma transacción local.

---

# 81. Same transaction checkpoint

Una implementación podrá:

```text
BEGIN
process chunk
write DB changes
write checkpoint
COMMIT
```

si checkpoint store está en la misma autoridad transaccional.

---

# 82. Different checkpoint store

Si checkpoint vive en otro sistema:

```text
DB transaction
≠
checkpoint transaction
```

y vuelve a existir dual-write.

---

# 83. Transaction policies

```php
enum LargeDatasetTransactionPolicy
{
    case NONE;
    case USE_EXISTING;
    case PER_BATCH;
    case PER_PARTITION;
    case WHOLE_OPERATION;
    case CUSTOM;
}
```

---

# 84. WHOLE_OPERATION

Deberá ser explícito y probablemente rechazado/warned para datasets gigantes según policy.

---

# 85. PER_BATCH

Frecuentemente será el balance práctico:

```text
BEGIN
process bounded batch
COMMIT
checkpoint
```

---

# 86. PER_PARTITION

Puede ser apropiado para partitions pequeñas/moderadas.

---

# 87. Transaction ownership

Siempre:

> **El Large Dataset System solo podrá commit/rollback transacciones que él mismo haya creado.**

---

# 88. Caller-owned transaction

Si existe:

```text
external transaction
```

el sistema deberá respetarla.

---

# 89. Parallel workers + transaction

Cada worker necesitará su propio contexto transaccional salvo una capacidad explícita que demuestre otra cosa.

---

# 90. No shared transaction object

Prohibido:

```text
Worker A ─┐
Worker B ─┼→ same mutable TransactionContext
Worker C ─┘
```

---

# 91. Failure model

Fallos posibles:

```text
QUERY_FAILURE
HYDRATION_FAILURE
PROCESSING_FAILURE
MUTATION_FAILURE
TRANSACTION_FAILURE
CHECKPOINT_FAILURE
WORKER_FAILURE
PARTITION_FAILURE
CONNECTION_FAILURE
DESTINATION_FAILURE
CANCELLATION
RESOURCE_EXHAUSTION
UNKNOWN_OUTCOME
```

---

# 92. Failure boundaries

El sistema deberá identificar:

```text
record
batch
partition
worker
operation
```

como boundaries diferentes.

---

# 93. Failure policy

```php
enum LargeDatasetFailurePolicy
{
    case STOP_ALL;
    case RETRY_UNIT;
    case RETRY_PARTITION;
    case SKIP_RECORD;
    case SKIP_BATCH;
    case SKIP_PARTITION;
    case CONTINUE_OTHER_PARTITIONS;
    case CUSTOM;
}
```

---

# 94. Default

Para operaciones mutables:

```text
STOP_ALL
```

será una opción segura por default salvo que la operación declare tolerancia.

---

# 95. Retry safety

Retry deberá considerar:

```text
idempotency
transaction outcome
external side effects
checkpoint position
operation type
```

---

# 96. UNKNOWN transaction outcome

Nunca:

```text
commit sent
connection lost
↓
blind retry
```

---

# 97. UNKNOWN

Debe preservarse y requerir reconciliación cuando corresponda.

---

# 98. Retry unit

Una unidad reintentable debe tener:

```text
stable identity
known boundaries
replay-safe semantics
```

---

# 99. Retry budget

```php
final readonly class RetryBudget
{
    public function __construct(
        public int $maxAttempts,
        public Duration $maxDuration,
        public BackoffPolicy $backoff,
    ) {}
}
```

---

# 100. Poison records

Un record que falla repetidamente podrá identificarse como:

```text
POISON_RECORD
```

---

# 101. Dead-letter integration

Una integración superior podrá enviar errores a:

```text
dead-letter store
queue
audit dataset
```

sin convertirlo en responsabilidad obligatoria del core.

---

# 102. Resource governance

El sistema deberá tratar recursos como parte de correctness operacional.

---

# 103. ResourceBudget

```php
final readonly class LargeDatasetResourceBudget
{
    public function __construct(
        public ?int $maxMemoryBytes,
        public ?int $maxConnections,
        public ?int $maxWorkers,
        public ?int $maxRows,
        public ?int $maxBatches,
        public ?int $maxDurationMs,
        public ?int $maxRetries,
        public ?int $maxPendingUnits,
    ) {}
}
```

---

# 104. Memory

Objetivo:

```text
M
≈
Workers × PerWorkerBoundedState
+
CoordinatorState
```

y no:

```text
M
≈
TotalDataset
```

---

# 105. Worker memory

```text
M_worker
≈
batch
+
hydration
+
transform buffers
+
ORM managed state
+
operation state
```

---

# 106. ORM IdentityMap

Aunque:

```text
batchSize = 1000
```

procesar:

```text
10,000 batches
```

puede dejar:

```text
10,000,000 managed entities
```

si no existe política de memoria.

---

# 107. ORM policy

```php
enum LargeDatasetOrmPolicy
{
    case KEEP_MANAGED;
    case DETACH_PROCESSED;
    case CLEAR_ENTITY_TYPE;
    case DEDICATED_ENTITY_MANAGER;
    case READ_ONLY_HYDRATION;
    case PROJECTION_ONLY;
    case CALLER_MANAGED;
}
```

---

# 108. Preferred mode

Para procesamiento analítico/read-only:

```text
PROJECTION_ONLY
```

o:

```text
READ_ONLY_HYDRATION
```

serán preferibles.

---

# 109. No silent EntityManager clear

El coordinator nunca limpiará estado ORM ajeno sin autorización explícita.

---

# 110. Dedicated EntityManager

Será recomendable para:

```text
long-running background processing
```

---

# 111. Adaptive batching

Podrá existir una extensión que ajuste:

```text
batch size
```

dinámicamente.

---

# 112. Inputs

Podrá observar:

```text
memory
query latency
processing latency
transaction duration
destination throughput
connection pressure
```

---

# 113. Adaptive algorithm

Conceptualmente:

```text
if memory pressure high:
    decrease batch

if throughput stable and memory low:
    increase batch
```

---

# 114. Hard bounds

Siempre:

```text
minBatchSize <= effectiveBatchSize <= maxBatchSize
```

---

# 115. Adaptive batching ≠ semantic change

Cambiar batch size no deberá cambiar qué registros pertenecen al dataset.

---

# 116. Backpressure

Debe propagarse upstream.

```text
Slow processor
      ↑
Traversal slows

Slow destination
      ↑
Transform slows
      ↑
Reader slows
```

---

# 117. No unbounded pending units

El coordinator deberá limitar:

```text
queued execution units
```

---

# 118. Worker queue

Conceptualmente:

```text
Partition Producer
       ↓
bounded work queue
       ↓
workers
```

---

# 119. Queue capacity

Será parte del ResourcePlan.

---

# 120. Cancellation

Toda ejecución larga deberá poder cancelarse.

---

# 121. CancellationToken

```php
interface CancellationToken
{
    public function isCancellationRequested(): bool;
}
```

---

# 122. Cooperative cancellation

Workers comprobarán cancellation en boundaries seguros.

---

# 123. Safe boundaries

Ejemplos:

```text
before next chunk
after transaction
before next partition
after destination flush
```

---

# 124. Forced cancellation

Podrá intentar:

```text
query cancellation
connection interruption
worker termination
```

según capacidades.

---

# 125. Forced termination

Puede producir:

```text
UNKNOWN
```

si no se conoce el outcome.

---

# 126. Deadline

```php
->deadline($instant)
```

podrá propagarse a:

```text
query timeout
transaction budget
worker budget
destination timeout
```

---

# 127. Deadline ≠ query timeout

Una operación puede tener:

```text
2 hour global deadline
```

y:

```text
30 second per-query timeout
```

---

# 128. Progress architecture

```php
final readonly class LargeDatasetProgress
{
    public function __construct(
        public int $processedRecords,
        public int $successfulRecords,
        public int $failedRecords,
        public int $completedPartitions,
        public int $activePartitions,
        public int $pendingPartitions,
        public ?int $estimatedTotalRecords,
    ) {}
}
```

---

# 129. Progress dimensions

Podrá medirse por:

```text
records
batches
partitions
bytes
time
```

---

# 130. Unknown total

Debe soportarse:

```text
totalRecords = UNKNOWN
```

---

# 131. Estimated total

No deberá representarse como exacto.

---

# 132. Progress aggregation

Para workers:

```text
Global Progress
=
aggregate(worker progress)
```

---

# 133. Progress ≠ durability

Una UI mostrando:

```text
95%
```

no significa que el checkpoint durable esté en 95%.

---

# 134. Progress monotonicity

Los contadores confirmados deberían ser monotónicos.

Los estimados pueden recalcularse.

---

# 135. Processing lifecycle

```text
CREATED
   ↓
PLANNING
   ↓
PLANNED
   ↓
RUNNING
   ├── PAUSING
   │      ↓
   │    PAUSED
   │
   ├── CANCELLING
   │      ↓
   │   CANCELLED
   │
   ├── FAILED
   │
   ├── UNKNOWN
   │
   └── FINALIZING
          ↓
       COMPLETED
```

---

# 136. Partition lifecycle

```text
PENDING
↓
CLAIMED
↓
RUNNING
├── COMPLETED
├── RETRYABLE
├── FAILED
├── PAUSED
├── CANCELLED
└── UNKNOWN
```

---

# 137. Batch lifecycle

```text
PLANNED
↓
FETCHING
↓
PROCESSING
↓
PERSISTING
↓
COMMITTING
↓
CHECKPOINTING
↓
COMPLETED
```

---

# 138. State ≠ evidence

Un estado deberá acompañarse de evidencia cuando exista incertidumbre.

---

# 139. ProcessingResult

```php
final readonly class LargeDatasetResult
{
    public function __construct(
        public ProcessingId $id,
        public LargeDatasetStatus $status,
        public LargeDatasetProgress $progress,
        public array $partitionResults,
        public ProcessingOutcomeEvidence $evidence,
    ) {}
}
```

---

# 140. Status

```php
enum LargeDatasetStatus
{
    case COMPLETED;
    case COMPLETED_WITH_SKIPS;
    case PARTIAL;
    case PAUSED;
    case CANCELLED;
    case FAILED;
    case UNKNOWN;
}
```

---

# 141. COMPLETED_WITH_SKIPS

Solo será válido cuando la policy permita skips.

---

# 142. PARTIAL

Significa que parte del trabajo tiene resultados confirmados pero la operación completa no terminó.

---

# 143. UNKNOWN

Significa que no puede afirmarse el outcome completo.

---

# 144. Read routing

Operaciones read-only podrán usar replicas.

Pero:

```text
Read Only
≠
Replica Safe
```

---

# 145. Routing policies

```text
WRITER_ONLY
ANY_ELIGIBLE
PIN_ENDPOINT
MINIMUM_POSITION
PER_PARTITION
CUSTOM
```

---

# 146. Parallel replica use

Partitions independientes podrán distribuirse entre replicas si la consistency policy lo permite.

---

# 147. Replica drift

Dos replicas pueden observar estados diferentes.

Por tanto:

```text
Parallel Replica Processing
≠
Single Snapshot
```

---

# 148. MONOTONIC processing

Puede requerir mantener:

```text
minimum observed replication position
```

---

# 149. Mutations

Toda operación que escriba deberá ir al authority endpoint correspondiente.

---

# 150. Sharding

El sistema deberá ser shard-aware.

---

# 151. Natural shard partitioning

```text
Logical Dataset
├── Shard A
├── Shard B
└── Shard C
```

es una partición natural.

---

# 152. Shard map generation

Checkpoints durables deberán poder guardar:

```text
ShardMapGeneration
```

---

# 153. Resharding

Si topology cambia durante pause/resume:

```text
old partition ownership
```

puede dejar de ser válido.

---

# 154. Resume after resharding

Deberá:

```text
validate
replan
reconcile
or reject
```

Nunca continuar ciegamente.

---

# 155. Cross-shard transaction

El sistema no fingirá:

```text
global ACID
```

si no existe infraestructura que lo soporte.

---

# 156. Per-shard transaction

Será la unidad natural para muchas operaciones.

---

# 157. Shard failure

Si:

```text
A ✓
B ✓
C ?
```

resultado global no podrá ser:

```text
COMPLETED
```

---

# 158. Tenant isolation

Tenant context será parte de:

```text
DatasetIdentity
ExecutionContext
Partition
Checkpoint
Telemetry context
```

cuando corresponda.

---

# 159. Cross-tenant processing

Deberá ser:

```text
explicit
authorized
auditable
```

---

# 160. Tenant partitioning

Puede permitir:

```text
one tenant per execution unit
```

reduciendo riesgo de context leakage.

---

# 161. No tenant context mutation

Un worker no reutilizará mutable tenant context entre partitions sin reset explícito.

---

# 162. Security

Authorization debe aplicarse antes de definir el dataset efectivo.

```text
Actor
↓
Authorization Scope
↓
Tenant Scope
↓
Query
↓
Partitioning
↓
Processing
```

---

# 163. Incorrecto

```text
Query all rows
↓
partition
↓
process
↓
discard unauthorized rows
```

---

# 164. Correcto

```text
Authorized Query
↓
partition
↓
process
```

---

# 165. Security fingerprint

Durable resume podrá incluir:

```text
authorization scope fingerprint
```

---

# 166. Permission changes

Resume deberá revalidar seguridad.

---

# 167. Cache integration

Large Dataset Processing no deberá llenar Result Cache por default.

---

# 168. Default

```text
Result Cache:
    BYPASS
```

para operaciones masivas.

---

# 169. Metadata caches

Podrán seguir utilizándose.

---

# 170. Mutation invalidation

Bulk mutations deberán utilizar las reglas de invalidación semántica ya definidas.

---

# 171. No FLUSHALL

Nunca:

```text
large mutation
↓
cache FLUSHALL
```

como solución arquitectónica general.

---

# 172. Import integration

```text
Large Dataset Processing
↓
Import partitions
↓
Import batches
↓
Bulk Insert/Update
```

---

# 173. Export integration

```text
Large Dataset Processing
↓
Export partitions
↓
Traversal
↓
Serialization
↓
Destination
```

---

# 174. Bulk integration

Operaciones:

```text
bulk insert
bulk update
bulk delete
```

podrán ser execution primitives.

---

# 175. Bulk ≠ per-row ORM loop

Si una operación puede expresarse semánticamente como bulk:

```text
Bulk Engine
```

deberá preferirse cuando preserve las reglas requeridas.

---

# 176. ORM lifecycle caveat

Bulk mutation puede bypass:

```text
entity lifecycle callbacks
IdentityMap synchronization
per-entity domain logic
```

según el sistema definido en 203–205.

El planner deberá conocerlo.

---

# 177. Processing mode

```php
enum ProcessingMode
{
    case ENTITY;
    case PROJECTION;
    case RAW_TYPED_ROW;
    case BULK;
}
```

---

# 178. ENTITY

Más expresivo.

Más costoso.

---

# 179. PROJECTION

Preferido para:

```text
read
analytics
transformations
exports
```

---

# 180. BULK

Preferido para operaciones set-based cuando sean semánticamente compatibles.

---

# 181. N+1

Entity processing no deshabilitará:

```text
N+1 Detection System
```

---

# 182. Eager loading

Para traversal por chunks:

```text
root chunk
↓
batch eager load relationships
↓
process
↓
release
```

será preferible a cargar todo el graph global.

---

# 183. Relationship coverage

Las reglas de:

```text
FULL
PARTIAL
UNKNOWN
```

seguirán vigentes.

---

# 184. Event interactions

Los eventos de Entity/ORM existentes no deberán confundirse con eventos del processing system.

Ejemplo:

```text
EntityUpdated
≠
LargeDatasetBatchCompleted
```

---

# 185. Event volume

Una operación de 100M rows no debe generar automáticamente 100M eventos operacionales de alto nivel.

---

# 186. Telemetry architecture

Eventos conceptuales:

```text
LargeDatasetPlanned
LargeDatasetStarted
PartitionCreated
PartitionClaimed
PartitionStarted
BatchStarted
BatchCompleted
PartitionCheckpointed
PartitionCompleted
WorkerStarted
WorkerStopped
WorkerFailed
LargeDatasetPaused
LargeDatasetResumed
LargeDatasetCancelled
LargeDatasetFailed
LargeDatasetCompleted
LargeDatasetOutcomeUnknown
```

---

# 187. Per-row telemetry

Deshabilitada por default.

---

# 188. Metrics

```text
db.large_dataset.operations
db.large_dataset.duration
db.large_dataset.records
db.large_dataset.batches
db.large_dataset.partitions
db.large_dataset.workers
db.large_dataset.retries
db.large_dataset.failures
db.large_dataset.skips
db.large_dataset.checkpoints
db.large_dataset.memory
db.large_dataset.backpressure
db.large_dataset.unknown
```

---

# 189. Bounded cardinality

No usar como metric labels:

```text
ProcessingId
PartitionId
record ID
tenant ID
raw query
raw cursor
```

si generan cardinalidad no controlada.

---

# 190. Diagnostics

API conceptual:

```php
DB::largeDataset()->explain($request);
```

---

# 191. Explain example

```text
LARGE DATASET PROCESSING PLAN

Dataset:
    UserMigrationDataset

Query:
    semantic fingerprint: 8f0a...

Estimated Cardinality:
    ~82,000,000

Operation:
    MUTATE

Mutation:
    non-ordering fields

Traversal:
    KEYSET

Ordering:
    tenant_id ASC
    id ASC

Horizon:
    CAPTURE_UPPER_BOUND

Partitioning:
    SHARD

Partitions:
    12

Requested Workers:
    8

Effective Worker Limit:
    6

Reason:
    connection budget

Chunk Size:
    5000

Bulk Size:
    1000

Transaction:
    PER_BATCH

Checkpoint:
    PER_BATCH

Retry:
    3 attempts
    exponential backoff

Consistency:
    READ_YOUR_WRITES

ORM:
    PROJECTION_ONLY

Result Cache:
    BYPASS

Tenant Isolation:
    REQUIRED

Resume:
    SUPPORTED

Exactly Once:
    NOT GUARANTEED

Warnings:
    External callback side effects detected.
    Operation must be idempotent for automatic retry.
```

---

# 192. Planner architecture

```php
final class LargeDatasetPlanner
{
    public function plan(
        LargeDatasetRequest $request,
        LargeDatasetPlanningContext $context,
    ): LargeDatasetPlan;
}
```

---

# 193. Planner responsibilities

```text
validate source
normalize dataset
determine operation characteristics
select traversal
validate ordering
choose horizon
plan partitions
determine concurrency
determine transactions
determine consistency
validate retry safety
plan checkpoints
calculate resource limits
validate distribution
validate security
produce warnings
```

---

# 194. Planner must not execute

No deberá:

```text
run queries
claim partitions
open transactions
spawn workers
write checkpoints
```

---

# 195. LargeDatasetPlan

```php
final readonly class LargeDatasetPlan
{
    public function __construct(
        public DatasetPlan $dataset,
        public TraversalPlan $traversal,
        public PartitionPlan $partitions,
        public WorkerPlan $workers,
        public TransactionPlan $transactions,
        public ConsistencyPlan $consistency,
        public RecoveryPlan $recovery,
        public ResourcePlan $resources,
        public SecurityPlan $security,
    ) {}
}
```

---

# 196. Plan immutability

Una vez iniciada la ejecución:

```text
Plan
```

será conceptualmente inmutable.

---

# 197. Adaptive runtime decisions

Cambios como:

```text
effective workers
batch size
backpressure
```

serán decisiones runtime dentro de límites permitidos por el plan.

---

# 198. Plan version

Checkpoints deberán incluir:

```text
PlanVersion
```

---

# 199. Resume compatibility

Una nueva versión de VoltStack podrá:

```text
accept
migrate
reject
```

un checkpoint antiguo.

Nunca interpretarlo incorrectamente.

---

# 200. Coordinator

```php
final class LargeDatasetCoordinator
{
    public function execute(
        LargeDatasetPlan $plan,
        LargeDatasetExecutionContext $context,
    ): LargeDatasetResult;
}
```

---

# 201. Coordinator responsibilities

```text
partition scheduling
worker lifecycle
progress aggregation
resource governance
checkpoint coordination
cancellation
failure aggregation
final result
```

---

# 202. Coordinator ≠ Worker

Coordinator no procesa directamente records salvo estrategia single-worker simplificada.

---

# 203. Single-worker optimization

Un plan pequeño podrá ejecutarse:

```text
Coordinator
→ Local Worker
```

sin infraestructura distribuida.

---

# 204. Same architecture

La semántica será la misma con:

```text
1 worker
```

o:

```text
N workers
```

cuando el operation contract permita paralelismo.

---

# 205. Persistent runtime

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 206. Scope-local mutable state

Nunca compartir:

```text
ProcessingId
current partition
current boundary
current transaction
current EntityManager
current tenant
current shard
current worker state
current checkpoint
current progress
```

entre operaciones concurrentes.

---

# 207. Static global state

Prohibido:

```php
LargeDatasetManager::$currentPartition;
```

---

# 208. Worker reuse

Un worker persistente deberá:

```text
finish unit
↓
close resources
↓
reset scoped DB state
↓
release tenant
↓
release transaction
↓
clear operation-owned ORM state
↓
accept next unit
```

---

# 209. FrankenPHP

El procesamiento largo no deberá contaminar el siguiente request manejado por el worker.

---

# 210. RoadRunner

La misma regla aplica al worker loop persistente.

---

# 211. OpenSwoole

Cada coroutine deberá poseer su:

```text
DatabaseContext
TransactionContext
TenantContext
EntityManager scope
ProcessingContext
```

---

# 212. Concurrency isolation

```text
Coroutine A
≠
Coroutine B
```

aunque compartan objetos inmutables.

---

# 213. Long-running work

Aunque pueda ejecutarse durante HTTP, grandes operaciones deberían poder integrarse con background jobs.

---

# 214. HTTP independence

Database Large Dataset Processing no dependerá de:

```text
HTTP request
HTTP response
browser
session
```

---

# 215. Queue integration

Será una integración posterior:

```text
LargeDatasetPlan
↓
Partition Units
↓
Queue Jobs
```

---

# 216. Serializable execution spec

No enviar a jobs:

```text
Connection
PDO
EntityManager
ResultCursor
Generator
open transaction
```

---

# 217. Worker reconstruction

Cada worker reconstruirá sus recursos desde:

```text
ExecutionUnit specification
+
scoped runtime context
```

---

# 218. Error hierarchy

```text
DatabaseException
└── LargeDatasetException
    ├── LargeDatasetPlanningException
    ├── DatasetTraversalException
    ├── DatasetPartitionException
    ├── DatasetPartitionOverlapException
    ├── DatasetPartitionCoverageException
    ├── DatasetWorkerException
    ├── DatasetClaimException
    ├── DatasetLeaseException
    ├── DatasetProcessingException
    ├── DatasetCheckpointException
    ├── DatasetResumeException
    ├── DatasetRetryUnsafeException
    ├── DatasetConsistencyException
    ├── DatasetTransactionException
    ├── DatasetResourceException
    ├── DatasetSecurityException
    ├── DatasetCancellationException
    └── DatasetOutcomeUnknownException
```

---

# 219. Error context

Errores podrán incluir:

```text
ProcessingId
AttemptId
PartitionId
WorkerId
BatchIndex
phase
continuation boundary fingerprint
transaction state
outcome evidence
```

---

# 220. Sensitive data

No incluir automáticamente:

```text
record contents
credentials
raw SQL
PII
secrets
```

---

# 221. Testing strategy

El sistema requerirá pruebas en múltiples niveles.

---

# 222. Unit tests

```text
planner
partition generators
range boundaries
fingerprints
checkpoint compatibility
retry safety
resource calculations
state transitions
```

---

# 223. Integration tests

```text
real DB traversal
transactions
chunk integration
bulk integration
import/export
replicas
shards
worker failures
runtime reuse
```

---

# 224. Crash recovery tests

Simular:

```text
crash before commit
crash after commit
crash before checkpoint
crash after checkpoint
worker lease expiration
destination uncertainty
```

---

# 225. Mutation tests

```text
insert during traversal
delete during traversal
update ordering key
update non-ordering key
resharding
tenant context changes
```

---

# 226. Memory tests

Procesar datasets suficientemente grandes para demostrar que:

```text
memory
```

no crece proporcionalmente al dataset bajo políticas bounded.

---

# 227. Concurrency tests

Verificar:

```text
no duplicate claims
no context leakage
bounded workers
partition isolation
transaction isolation
```

---

# 228. Persistent runtime tests

Ejecutar secuencialmente:

```text
Operation A
Operation B
Operation C
```

en el mismo worker y comprobar ausencia de state leakage.

---

# 229. Directory structure

```text
src/Quantum/Database/LargeData/
│
├── LargeDatasetRequest.php
├── LargeDatasetOptions.php
├── LargeDatasetPlanner.php
├── LargeDatasetPlan.php
├── LargeDatasetCoordinator.php
├── LargeDatasetResult.php
├── LargeDatasetStatus.php
├── ProcessingId.php
├── AttemptId.php
│
├── Dataset/
│   ├── LargeDatasetSource.php
│   ├── DatasetIdentity.php
│   ├── DatasetPlan.php
│   ├── DatasetHorizon.php
│   └── DatasetFingerprint.php
│
├── Operation/
│   ├── LargeDatasetOperation.php
│   ├── LargeDatasetOperationType.php
│   ├── OperationCharacteristics.php
│   └── ProcessingMode.php
│
├── Traversal/
│   ├── LargeDatasetStrategy.php
│   ├── LargeDatasetTraversal.php
│   ├── TraversalPlan.php
│   └── ContinuationBoundary.php
│
├── Partition/
│   ├── DatasetPartition.php
│   ├── DatasetPartitionStrategy.php
│   ├── PartitionPlan.php
│   ├── PartitionConfidence.php
│   ├── PartitionGenerator.php
│   ├── KeyRangePartitioner.php
│   ├── HashPartitioner.php
│   ├── ShardPartitioner.php
│   ├── TenantPartitioner.php
│   └── TemporalPartitioner.php
│
├── Worker/
│   ├── WorkerId.php
│   ├── WorkerPlan.php
│   ├── LargeDatasetWorkerExecutor.php
│   ├── LargeDatasetExecutionUnit.php
│   ├── LargeDatasetExecutionContext.php
│   ├── PartitionClaim.php
│   ├── ClaimToken.php
│   ├── WorkSchedulingPolicy.php
│   └── WorkerCoordinator.php
│
├── Recovery/
│   ├── LargeDatasetCheckpoint.php
│   ├── PartitionCheckpoint.php
│   ├── LargeDatasetCheckpointStore.php
│   ├── RecoveryPlan.php
│   ├── ResumeValidator.php
│   └── RetryBudget.php
│
├── Transaction/
│   ├── LargeDatasetTransactionPolicy.php
│   └── TransactionPlan.php
│
├── Consistency/
│   ├── LargeDatasetConsistency.php
│   └── ConsistencyPlan.php
│
├── Resource/
│   ├── LargeDatasetResourceBudget.php
│   ├── ResourcePlan.php
│   ├── ResourceGovernor.php
│   ├── BackpressureController.php
│   └── AdaptiveBatchController.php
│
├── Progress/
│   ├── LargeDatasetProgress.php
│   ├── ProgressAggregator.php
│   └── ProgressReporter.php
│
├── Cancellation/
│   ├── CancellationToken.php
│   └── Deadline.php
│
├── Diagnostics/
│   ├── LargeDatasetInspector.php
│   └── LargeDatasetExplainer.php
│
├── Telemetry/
│   └── LargeDatasetTelemetry.php
│
└── Exception/
    └── ...
```

---

# 230. Dependencias internas

```text
LargeData
│
├── Query Engine
├── Execution Engine
├── Chunk
├── Lazy
├── Bulk
├── Import
├── Export
├── Transaction
├── ORM
├── Read Routing
├── Sharding
├── Cache
└── Type System
```

No deberá invertir dependencias:

```text
Query Engine
    ↓
LargeData
```

como dependencia obligatoria.

---

# 231. Integraciones externas

```text
LargeData
├── Event System
├── Telemetry
├── Jobs/Queues
├── Security
├── Runtime
├── Storage
└── Administration
```

mediante contracts/adapters.

---

# 232. Architectural invariants

## DB-LARGE-001
Large Dataset Processing será distinto de Chunk Processing.

## DB-LARGE-002
Será distinto de Pagination.

## DB-LARGE-003
Será distinto de Cursor Pagination.

## DB-LARGE-004
Será distinto de Lazy Collection.

## DB-LARGE-005
Será distinto de Streaming Result.

## DB-LARGE-006
Será distinto de Bulk Operation.

## DB-LARGE-007
Será distinto de Import.

## DB-LARGE-008
Será distinto de Export.

## DB-LARGE-009
Será distinto de Queue.

## DB-LARGE-010
Será un coordinator, no un segundo Query Engine.

## DB-LARGE-011
No generará SQL directamente.

## DB-LARGE-012
No accederá al Driver directamente.

## DB-LARGE-013
Reutilizará Query Engine.

## DB-LARGE-014
Reutilizará Chunk Processing.

## DB-LARGE-015
Reutilizará Cursor/Keyset infrastructure.

## DB-LARGE-016
Reutilizará Lazy Collection.

## DB-LARGE-017
Reutilizará Bulk systems.

## DB-LARGE-018
Reutilizará Import/Export cuando corresponda.

## DB-LARGE-019
Dataset Definition será distinto de Dataset Snapshot.

## DB-LARGE-020
Same Dataset Definition no implicará Same Dataset Contents.

## DB-LARGE-021
Operation characteristics serán explícitas.

## DB-LARGE-022
UNKNOWN no será tratado como SAFE.

## DB-LARGE-023
AUTO será explainable.

## DB-LARGE-024
AUTO no cambiará semántica silenciosamente.

## DB-LARGE-025
Dataset identity será estable.

## DB-LARGE-026
Query fingerprint será semántico.

## DB-LARGE-027
Horizon será explícito.

## DB-LARGE-028
LIVE no implicará snapshot.

## DB-LARGE-029
Upper Bound no implicará snapshot.

## DB-LARGE-030
Snapshot no se fingirá.

## DB-LARGE-031
No se abrirá giant transaction automáticamente.

## DB-LARGE-032
Execution Unit será distinta de Chunk.

## DB-LARGE-033
Partition será distinta de Batch.

## DB-LARGE-034
Partition será distinta de Shard.

## DB-LARGE-035
Partition será distinta de Tenant.

## DB-LARGE-036
Partition coverage será modelada.

## DB-LARGE-037
Partition overlap será modelado.

## DB-LARGE-038
UNKNOWN coverage no implicará exhaustive processing.

## DB-LARGE-039
Range size no implicará row count.

## DB-LARGE-040
IDs no deberán ser contiguos.

## DB-LARGE-041
Hash partitioning no implicará natural ordering.

## DB-LARGE-042
Shard podrá ser una natural partition.

## DB-LARGE-043
Tenant partitioning será explícita.

## DB-LARGE-044
Dynamic partitions serán soportables.

## DB-LARGE-045
Worker será una abstracción lógica.

## DB-LARGE-046
Core no dependerá de worker runtime concreto.

## DB-LARGE-047
Requested Parallelism será distinto de Effective Parallelism.

## DB-LARGE-048
Concurrency será bounded.

## DB-LARGE-049
No habrá worker-per-row.

## DB-LARGE-050
Partition ownership será explícito.

## DB-LARGE-051
Claim será distinto de Transaction Lock.

## DB-LARGE-052
Lease Expired no implicará Worker Dead.

## DB-LARGE-053
Fencing será soportable.

## DB-LARGE-054
Fencing será distinto de Transaction Isolation.

## DB-LARGE-055
Checkpoint será distinto de Progress.

## DB-LARGE-056
Checkpoint será distinto de Snapshot.

## DB-LARGE-057
Checkpoint será distinto de Transaction Log.

## DB-LARGE-058
Checkpoint será versionado.

## DB-LARGE-059
Checkpoint será dataset-bound.

## DB-LARGE-060
Checkpoint será operation-bound.

## DB-LARGE-061
Checkpoint será tenant-bound cuando corresponda.

## DB-LARGE-062
Checkpoint será shard/topology-aware cuando corresponda.

## DB-LARGE-063
Progress podrá adelantarse al checkpoint.

## DB-LARGE-064
Crash podrá provocar replay.

## DB-LARGE-065
Checkpoint + Retry no implicará Exactly Once.

## DB-LARGE-066
External side effects requerirán estrategia adicional.

## DB-LARGE-067
Idempotency será distinta de Retry.

## DB-LARGE-068
Outbox será distinto de Checkpoint.

## DB-LARGE-069
Database rollback no deshará external side effects.

## DB-LARGE-070
Same-DB checkpoint podrá fortalecer atomicidad.

## DB-LARGE-071
Cross-system checkpoint introducirá dual-write concerns.

## DB-LARGE-072
Transaction policy será explícita.

## DB-LARGE-073
Whole-operation transaction no será default.

## DB-LARGE-074
El sistema solo commitirá transacciones propias.

## DB-LARGE-075
Caller-owned transactions serán respetadas.

## DB-LARGE-076
Workers no compartirán mutable TransactionContext.

## DB-LARGE-077
Failure boundary será explícito.

## DB-LARGE-078
Retry safety será evaluada.

## DB-LARGE-079
UNKNOWN commit no será blind-retried.

## DB-LARGE-080
Retry tendrá budget.

## DB-LARGE-081
Poison records serán representables.

## DB-LARGE-082
Dead-letter será integración opcional.

## DB-LARGE-083
Resource budget será parte del plan.

## DB-LARGE-084
Memory deberá ser bounded cuando sea posible.

## DB-LARGE-085
Dataset size no deberá determinar directamente memory size.

## DB-LARGE-086
Worker count multiplicará resource pressure.

## DB-LARGE-087
IdentityMap growth será considerado.

## DB-LARGE-088
EntityManager no será limpiado silenciosamente.

## DB-LARGE-089
Dedicated EntityManager será soportable.

## DB-LARGE-090
Projection mode será preferible cuando entidades no sean necesarias.

## DB-LARGE-091
Adaptive batching respetará hard bounds.

## DB-LARGE-092
Adaptive batching no cambiará dataset semantics.

## DB-LARGE-093
Backpressure será propagada.

## DB-LARGE-094
Pending work será bounded.

## DB-LARGE-095
Cancellation será soportada.

## DB-LARGE-096
Cancellation será distinta de Rollback.

## DB-LARGE-097
Forced cancellation podrá producir UNKNOWN.

## DB-LARGE-098
Deadline será distinta de Query Timeout.

## DB-LARGE-099
Progress total podrá ser UNKNOWN.

## DB-LARGE-100
Estimated total será distinto de Exact total.

## DB-LARGE-101
Progress será distinto de durability.

## DB-LARGE-102
PARTIAL será distinto de FAILED.

## DB-LARGE-103
UNKNOWN será distinto de FAILED.

## DB-LARGE-104
COMPLETED_WITH_SKIPS requerirá policy explícita.

## DB-LARGE-105
Read-only será distinto de Replica-safe.

## DB-LARGE-106
Pinned replica será distinta de Snapshot.

## DB-LARGE-107
Parallel replicas no implicarán single snapshot.

## DB-LARGE-108
Mutations irán a authority endpoint.

## DB-LARGE-109
Sharding será explícitamente considerado.

## DB-LARGE-110
Shard map generation podrá formar parte del checkpoint.

## DB-LARGE-111
Resharding invalidará assumptions antiguas.

## DB-LARGE-112
Resume después de resharding requerirá validación.

## DB-LARGE-113
Cross-shard ACID no será fingido.

## DB-LARGE-114
Shard failure no producirá fake success.

## DB-LARGE-115
Tenant isolation será preservada.

## DB-LARGE-116
Cross-tenant processing será explícito.

## DB-LARGE-117
Cross-tenant processing requerirá autorización.

## DB-LARGE-118
Tenant context no podrá filtrarse entre workers.

## DB-LARGE-119
Authorization ocurrirá antes de partitioning.

## DB-LARGE-120
Unauthorized rows no serán procesados y luego filtrados.

## DB-LARGE-121
Resume revalidará authorization.

## DB-LARGE-122
Result Cache será bypassed por default para operaciones masivas.

## DB-LARGE-123
Metadata Cache podrá usarse.

## DB-LARGE-124
Bulk invalidation será semántica.

## DB-LARGE-125
Large mutations no usarán FLUSHALL como estrategia general.

## DB-LARGE-126
Import será un execution primitive especializado.

## DB-LARGE-127
Export será un execution primitive especializado.

## DB-LARGE-128
Bulk será un execution primitive especializado.

## DB-LARGE-129
Bulk será distinto de ORM per-row loop.

## DB-LARGE-130
Bulk lifecycle differences serán explícitas.

## DB-LARGE-131
N+1 detection seguirá disponible.

## DB-LARGE-132
Eager relationships deberán cargarse incrementalmente.

## DB-LARGE-133
Relationship coverage será preservada.

## DB-LARGE-134
Operational events serán distintos de Entity events.

## DB-LARGE-135
Per-row telemetry estará deshabilitada por default.

## DB-LARGE-136
Telemetry tendrá bounded cardinality.

## DB-LARGE-137
Large Dataset Processing será explainable.

## DB-LARGE-138
Planner será distinto de Coordinator.

## DB-LARGE-139
Planner no ejecutará queries.

## DB-LARGE-140
Planner no abrirá transactions.

## DB-LARGE-141
Planner no reclamará partitions.

## DB-LARGE-142
Plan será inmutable conceptualmente.

## DB-LARGE-143
Adaptive decisions permanecerán dentro del plan.

## DB-LARGE-144
Plan version será checkpointed.

## DB-LARGE-145
Old checkpoints serán validados por versión.

## DB-LARGE-146
Coordinator será distinto de Worker.

## DB-LARGE-147
Single-worker y multi-worker compartirán arquitectura.

## DB-LARGE-148
Mutable processing state será scope-local.

## DB-LARGE-149
No habrá static current partition.

## DB-LARGE-150
Worker reuse requerirá reset.

## DB-LARGE-151
FrankenPHP state leakage será prohibido.

## DB-LARGE-152
RoadRunner state leakage será prohibido.

## DB-LARGE-153
OpenSwoole coroutine state leakage será prohibido.

## DB-LARGE-154
HTTP no será dependencia obligatoria.

## DB-LARGE-155
Queue no será dependencia obligatoria.

## DB-LARGE-156
Active DB resources no serán job payloads.

## DB-LARGE-157
Workers reconstruirán scoped resources.

## DB-LARGE-158
Errors preservarán phase/context.

## DB-LARGE-159
Errors no expondrán PII por default.

## DB-LARGE-160
UNKNOWN outcome será preservado.

## DB-LARGE-161
Correctness tendrá prioridad sobre throughput.

## DB-LARGE-162
Resource safety tendrá prioridad sobre maximal parallelism.

## DB-LARGE-163
Partition correctness tendrá prioridad sobre worker utilization.

## DB-LARGE-164
Security tendrá prioridad sobre processing convenience.

## DB-LARGE-165
Retry safety tendrá prioridad sobre automatic recovery.

## DB-LARGE-166
No se prometerá exactly-once sin protocolo que lo demuestre.

## DB-LARGE-167
Database reality será distinta de coordinator knowledge.

## DB-LARGE-168
Durable progress será distinto de observed progress.

## DB-LARGE-169
Execution concurrency será distinta de database concurrency control.

## DB-LARGE-170
Parallelism será distinto de correctness.

## DB-LARGE-171
Processing partition será distinta de physical database partition.

## DB-LARGE-172
Dataset processing no cambiará automáticamente topology.

## DB-LARGE-173
Persistent runtime cleanup será obligatorio.

## DB-LARGE-174
Operation-owned resources serán liberados al terminar.

## DB-LARGE-175
Cleanup failure será observable.

## DB-LARGE-176
Cancellation cleanup será obligatorio.

## DB-LARGE-177
Failure cleanup será obligatorio.

## DB-LARGE-178
Resume compatibility será validada antes de ejecutar.

## DB-LARGE-179
No se reutilizará checkpoint incompatible.

## DB-LARGE-180
Large Dataset Processing coordinará sistemas existentes en vez de duplicarlos.

---

# 233. Modelo formal

Sea un dataset lógico:

```text
D
```

y un horizon:

```text
H
```

El universo efectivo será:

```text
D_H
```

El partitioner produce:

```text
P(D_H)
=
{P1, P2, ..., Pn}
```

Idealmente:

```text
Pi ∩ Pj = ∅
∀ i ≠ j
```

y:

```text
⋃ Pi = D_H
```

cuando la estrategia garantice cobertura exhaustiva.

---

# 234. Procesamiento por partition

Cada partition:

```text
Pi
```

puede recorrerse como:

```text
Pi
=
C1 ∪ C2 ∪ ... ∪ Cm
```

donde:

```text
|Cj| <= BatchLimit
```

---

# 235. Keyset traversal

Para una partition:

```text
Q0 = Q ∧ PartitionPredicate(Pi)
```

y después:

```text
Qj
=
Q0
∧
After(Bj-1)
LIMIT N
```

donde:

```text
Bj-1
```

es la frontera confirmada anterior.

---

# 236. Concurrency model

Con:

```text
W
```

workers y:

```text
P
```

partitions:

```text
EffectiveConcurrency
<=
min(
    W,
    P,
    ConnectionBudget,
    ResourceBudget,
    RuntimeCapacity
)
```

---

# 237. Memory model

Idealmente:

```text
M_total
≈
M_coordinator
+
Σ M_worker(i)
```

con:

```text
M_worker(i)
=
O(batch_i + operation_state_i)
```

y no:

```text
O(total dataset)
```

---

# 238. Recovery model

Sea:

```text
C(Pi)
```

el checkpoint durable de una partition.

Después de failure:

```text
Resume(Pi)
=
ContinueAfter(C(Pi))
```

cuando:

```text
checkpoint valid
∧
dataset compatible
∧
operation compatible
∧
security compatible
∧
topology compatible
```

---

# 239. Exactly-once limitation

Aunque:

```text
Process(C)
→
Checkpoint(C)
```

exista, un crash entre ambos puede producir:

```text
replay
```

Por tanto:

```text
Checkpointed Processing
≠
Exactly Once
```

sin mecanismos adicionales.

---

# 240. Arquitectura final

```text
                         Application
                              │
                              ▼
                    Large Dataset API
                              │
                              ▼
                    LargeDatasetRequest
                              │
                              ▼
                    LargeDatasetPlanner
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
      Dataset Plan       Partition Plan      Resource Plan
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                   LargeDatasetCoordinator
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
       Worker Manager   Checkpoint Manager   Governor
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                       Execution Units
                              │
        ┌──────────────┬──────┼──────┬──────────────┐
        ▼              ▼      ▼      ▼              ▼
      Chunk           Lazy   Bulk   Import         Export
        │              │      │      │              │
        └──────────────┴──────┼──────┴──────────────┘
                              ▼
                         Query Engine
                              │
                              ▼
                       Execution Engine
                              │
                              ▼
                    Connection / Routing
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
                   Writer           Replicas
                     │
                     ▼
                 Database
```

---

# 241. Regla maestra final

> **Large Dataset Processing será la capa de coordinación para trabajos database-centric que exceden el tamaño práctico de una operación convencional. Su responsabilidad será dividir, recorrer, gobernar, recuperar y observar el trabajo, mientras Query, Chunk, Lazy, Bulk, Import, Export, Transaction, ORM y Execution Engine conservan sus responsabilidades especializadas.**

Siempre:

```text
Large Dataset Processing
≠
Chunk Processing
```

```text
Partition
≠
Chunk
```

```text
Execution Unit
≠
Transaction
```

```text
Worker
≠
Connection
```

```text
Worker
≠
OS Process
```

```text
Requested Parallelism
≠
Effective Parallelism
```

```text
Parallelism
≠
Correctness
```

```text
Lease Expired
≠
Worker Definitely Dead
```

```text
Checkpoint
≠
Progress
```

```text
Progress
≠
Durability
```

```text
Checkpoint
≠
Snapshot
```

```text
Checkpoint + Retry
≠
Exactly Once
```

```text
Upper Bound
≠
Snapshot
```

```text
Pinned Replica
≠
Snapshot
```

```text
Read Only
≠
Replica Safe
```

```text
Batch Size
≠
Memory Bound Guarantee
```

```text
Shard
≠
Partition
```

```text
Tenant
≠
Partition
```

```text
Processing Partition
≠
Physical Database Partition
```

```text
Retry
≠
Idempotency
```

```text
Database Rollback
≠
External Side-Effect Rollback
```

```text
Same Dataset Definition
≠
Same Dataset Contents
```

```text
Database Reality
≠
Coordinator Knowledge
```

y, especialmente:

```text
UNKNOWN
≠
SUCCESS
```

---

# 242. Ejemplo completo: migración masiva

```php
$result = User::query()
    ->where('legacy_format', true)
    ->orderBy('tenant_id')
    ->orderBy('id')
    ->largeDataset()
    ->strategy(LargeDatasetStrategy::KEYSET)
    ->horizon(DatasetHorizon::CAPTURE_UPPER_BOUND)
    ->partitionByShard()
    ->workers(8)
    ->batchSize(5_000)
    ->transaction(
        LargeDatasetTransactionPolicy::PER_BATCH
    )
    ->checkpointEveryBatch()
    ->retry(
        attempts: 3,
        backoff: Backoff::exponential()
    )
    ->processingMode(
        ProcessingMode::PROJECTION
    )
    ->process(function (LargeDatasetBatch $batch) {
        // transformation
    });
```

Internamente:

```text
Query
↓
Semantic Analysis
↓
Dataset Identity
↓
Shard Partitioning
↓
Keyset Traversal
↓
Execution Units
↓
Workers
↓
5,000-row batches
↓
Transformation
↓
Transaction
↓
Checkpoint
↓
Progress
↓
Next Batch
```

---

# 243. Ejemplo: 500 millones de registros

Supongamos:

```text
500,000,000 events
12 shards
24 logical partitions
8 workers
10,000 rows per chunk
```

El sistema no hará:

```text
SELECT * FROM events
↓
500M rows in memory
```

sino:

```text
500M logical dataset
        ↓
24 partitions
        ↓
8 concurrent workers maximum
        ↓
bounded keyset chunks
        ↓
10,000 rows
        ↓
process
        ↓
release
        ↓
checkpoint
        ↓
continue
```

De esta forma, el tamaño total del dataset deja de determinar directamente el tamaño de memoria requerido.

---

# 244. Ejemplo: actualización masiva

Cuando una operación pueda expresarse como:

```sql
UPDATE users
SET status = 'archived'
WHERE last_login_at < :cutoff
```

el planner no deberá convertirla innecesariamente en:

```text
load entity
↓
modify entity
↓
save entity
× millions
```

Podrá delegar a:

```text
204_DATABASE_BULK_UPDATE_SYSTEM.md
```

si las reglas de dominio, lifecycle y consistency permiten Bulk Update.

---

# 245. Ejemplo: lógica por entidad

Si la operación requiere:

```php
foreach ($users as $user) {
    $user->recalculateRiskProfile();
}
```

entonces podrá ser necesario:

```text
Chunk/Keyset
↓
Entity Hydration
↓
Domain Logic
↓
UoW
↓
Flush
↓
Memory Policy
```

El planner deberá reconocer que:

```text
set-based bulk
```

y:

```text
entity-domain processing
```

no son equivalentes.

---

# 246. Ejemplo: export distribuido

```text
12 shards
↓
12 shard partitions
↓
parallel workers
↓
NDJSON part per shard
↓
manifest
```

podrá utilizar conjuntamente:

```text
Large Dataset Processing
+
Export System
+
Sharding System
```

sin introducir un segundo Export Engine.

---

# 247. Ejemplo: import masivo

```text
2 TB source dataset
↓
Import partitions
↓
bounded parsing
↓
validation
↓
Bulk Insert
↓
checkpoint
↓
progress
```

combinará:

```text
206 Import
203 Bulk Insert
208 Large Dataset Processing
```

---

# 248. Filosofía de diseño

La filosofía de VoltStack para grandes datasets será:

```text
Do not materialize what can be streamed.

Do not stream what can be processed in bounded chunks more safely.

Do not iterate per entity what can be expressed correctly as a bulk operation.

Do not parallelize what cannot be partitioned safely.

Do not retry what cannot be replayed safely.

Do not claim exactly-once when only at-least-once can be demonstrated.

Do not hold global state in persistent workers.

Do not trade correctness for throughput silently.
```

---

# 249. Cierre del Bloque 19

Con este documento queda definido el bloque completo:

```text
BLOCK 19 — PAGINATION, BATCH & LARGE DATA

✓ 199_DATABASE_PAGINATION_SYSTEM.md
✓ 200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
✓ 201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
✓ 202_DATABASE_LAZY_COLLECTION_SYSTEM.md
✓ 203_DATABASE_BULK_INSERT_SYSTEM.md
✓ 204_DATABASE_BULK_UPDATE_SYSTEM.md
✓ 205_DATABASE_BULK_DELETE_SYSTEM.md
✓ 206_DATABASE_IMPORT_SYSTEM.md
✓ 207_DATABASE_EXPORT_SYSTEM.md
✓ 208_DATABASE_LARGE_DATASET_PROCESSING_SYSTEM.md
```

La arquitectura resultante puede visualizarse:

```text
                         LARGE DATA
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
         NAVIGATE          PROCESS           TRANSFER
            │                 │                 │
      ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴─────┐
      ▼           ▼     ▼           ▼     ▼           ▼
 Pagination    Cursor  Chunk       Lazy  Import      Export
                              │
                              ▼
                         BULK MUTATION
                              │
                  ┌───────────┼───────────┐
                  ▼           ▼           ▼
                Insert      Update      Delete
                              │
                              ▼
                    Large Dataset Coordinator
```

---

# 250. Resultado arquitectónico del bloque

VoltStack Database podrá abordar datasets desde tres niveles claramente distintos.

### Navegación

```text
Pagination
Cursor Pagination
```

para interfaces y acceso segmentado.

### Procesamiento incremental

```text
Chunk
Lazy Collection
Bulk Insert
Bulk Update
Bulk Delete
```

para operaciones acotadas y eficientes.

### Operaciones masivas coordinadas

```text
Import
Export
Large Dataset Processing
```

para procesos largos, recuperables, distribuibles y gobernados por recursos.

Esto evita convertir una única abstracción como:

```text
Collection
```

o:

```text
Query Builder
```

en un God Object responsable de todos los escenarios.

---

# 251. Siguiente bloque

El siguiente documento inicia:

```text
BLOCK 20 — DATABASE EVENTS
```

con:

```text
209_DATABASE_EVENT_ARCHITECTURE.md
```

El bloque estará compuesto por:

```text
209_DATABASE_EVENT_ARCHITECTURE.md
210_DATABASE_QUERY_EVENT_SYSTEM.md
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

La siguiente etapa establecerá una distinción fundamental:

```text
Database Event
≠
Domain Event
≠
Telemetry Event
≠
ORM Lifecycle Event
≠
Transaction Outcome
```

y definirá cómo Query Engine, Connection, Transaction, ORM, Persistence y los demás componentes podrán publicar eventos sin acoplarse directamente al Event System global de VoltStack.

---

# 252. Siguiente documento

```text
209_DATABASE_EVENT_ARCHITECTURE.md
```

Este documento definirá la arquitectura transversal de eventos de `Quantum/Database`, incluyendo:

```text
event contracts
event categories
event envelopes
event context
dispatch boundaries
sync/async semantics
pre-operation events
post-operation events
failure events
transaction-aware events
after-commit events
event ordering
listener isolation
listener failures
event mutability
event cancellation
event propagation
event metadata
event security
event payload governance
event extension points
persistent-runtime isolation
telemetry distinction
EventSystem integration
```

bajo una regla inicial:

> **Un evento de Database describirá un hecho o una fase observable del subsistema; no deberá convertirse en un mecanismo oculto para alterar arbitrariamente la semántica de Query, Transaction, ORM o Persistence.**