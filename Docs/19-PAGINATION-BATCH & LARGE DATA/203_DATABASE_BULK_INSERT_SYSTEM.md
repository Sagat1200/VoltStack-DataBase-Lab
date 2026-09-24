# 203_DATABASE_BULK_INSERT_SYSTEM.md

# VoltStack Quantum Database
## Database Bulk Insert System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 203 — Database Bulk Insert System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `202_DATABASE_LAZY_COLLECTION_SYSTEM.md`  
**Siguiente documento:** `204_DATABASE_BULK_UPDATE_SYSTEM.md`

---

# 1. Propósito

`Database Bulk Insert System` define la arquitectura mediante la cual VoltStack podrá insertar grandes conjuntos de registros de manera eficiente, tipada, acotada y consciente de las capacidades de cada plataforma.

Ejemplo:

```php
DB::table('users')->insertBulk([
    [
        'name'  => 'Alice',
        'email' => 'alice@example.com',
    ],
    [
        'name'  => 'Bob',
        'email' => 'bob@example.com',
    ],
]);
```

También deberá soportar fuentes incrementales:

```php
DB::table('events')->insertBulk(
    rows: $eventGenerator,
    batchSize: 1000,
);
```

y eventualmente:

```php
User::bulkInsert($rows);
```

como Developer API de conveniencia, sin convertir el Model API en un segundo motor de persistencia.

La regla central será:

> **Bulk Insert en VoltStack será una operación explícita de persistencia masiva orientada a conjuntos de datos, optimizada mediante batches tipados y conscientes de plataforma; no equivaldrá a ejecutar `persist()` repetidamente ni fingirá semántica ORM completa cuando ésta haya sido deliberadamente omitida.**

Formalmente:

```text
BulkInsert
=
InputRows
+
TypeNormalization
+
Routing
+
BatchPlanning
+
Compilation
+
Execution
+
OutcomeAggregation
```

y nunca:

```text
BulkInsert
=
foreach ($entities as $entity) {
    $entityManager->persist($entity);
    $entityManager->flush();
}
```

---

# 2. Posición dentro del Bloque 19

```text
199 Pagination
200 Cursor Pagination
201 Chunk Processing
202 Lazy Collection
        │
        ▼
203 Bulk Insert
204 Bulk Update
205 Bulk Delete
        │
        ▼
206 Import
207 Export
208 Large Dataset Processing
```

Los documentos 199–202 se concentraron principalmente en lectura incremental.

A partir de este documento se define la arquitectura de mutaciones masivas.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Bulk Insert
≠
ORM Persist Loop
≠
Batch Persistence
≠
Seeder
≠
Factory
≠
Fixture
≠
Import
≠
Upsert
≠
Migration
≠
Raw SQL
```

---

# 4. Bulk Insert vs ORM Persistence

ORM tradicional:

```php
foreach ($users as $user) {
    $entityManager->persist($user);
}

$entityManager->flush();
```

implica potencialmente:

```text
EntityManager
→ UnitOfWork
→ Change Tracking
→ Persistence Planning
→ Lifecycle Semantics
→ IdentityMap
→ ORM Events
→ Query Engine
```

Bulk Insert:

```text
Rows
→ Bulk Insert Planner
→ Query Engine
→ Execution Engine
```

Es deliberadamente más directo.

---

# 5. Consecuencia fundamental

Bulk Insert no deberá fingir:

```text
EntityManager semantics
```

si esas semánticas no fueron realmente ejecutadas.

Por ejemplo, un Bulk Insert directo no deberá afirmar automáticamente que las entidades correspondientes están:

```text
MANAGED
```

en el `IdentityMap`.

---

# 6. Bulk Insert vs Batch Persistence

`133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md` pertenece al ORM.

Batch Persistence procesa múltiples cambios ORM preservando:

```text
EntityManager
UnitOfWork
EntityState
IdentityMap
relationship persistence
lifecycle
```

Bulk Insert está orientado a:

```text
set-based / row-oriented insertion
```

---

# 7. Bulk Insert vs Import

Import es un workflow superior:

```text
CSV / JSON / external source
        ↓
Parsing
        ↓
Validation
        ↓
Transformation
        ↓
Bulk Insert
```

Por tanto:

```text
Import
may use
Bulk Insert
```

pero:

```text
Bulk Insert
≠
Import
```

---

# 8. Bulk Insert vs Seeder

Seeder decide:

```text
qué datos poblar
```

Bulk Insert decide:

```text
cómo insertar eficientemente muchas filas
```

Un Seeder podrá consumir Bulk Insert.

---

# 9. Bulk Insert vs Factory

Factory construye:

```text
synthetic/domain values
```

Bulk Insert persiste:

```text
row sets
```

Un Factory/Test Data Generator podrá producir la entrada de Bulk Insert.

---

# 10. Objetivos

El sistema deberá soportar:

1. inserciones multi-row;
2. batches acotados;
3. iterables/generators;
4. Lazy Collections;
5. tipos canónicos;
6. bindings seguros;
7. múltiples columnas;
8. columnas opcionales;
9. defaults;
10. generated IDs;
11. `RETURNING` cuando exista;
12. capability-aware compilation;
13. límites de parámetros;
14. límites de packet/query;
15. batch sizing;
16. transacciones;
17. retries;
18. outcomes parciales;
19. duplicate/conflict policies explícitas;
20. sharding;
21. multitenancy;
22. read/write routing;
23. cache invalidation;
24. events;
25. telemetry;
26. persistent runtimes;
27. resource governance;
28. extensibilidad.

---

# 11. Arquitectura general

```text
BulkInsert API
      │
      ▼
BulkInsertRequest
      │
      ▼
BulkInsertInputNormalizer
      │
      ▼
BulkInsertSchemaResolver
      │
      ▼
BulkInsertPlanner
      │
      ├── Type Resolution
      ├── Column Shape
      ├── Routing
      ├── Capability Analysis
      ├── Batch Sizing
      ├── Conflict Policy
      ├── Returning Policy
      └── Transaction Policy
      │
      ▼
BulkInsertPlan
      │
      ▼
BulkInsertRunner
      │
      ├── Batch 1
      ├── Batch 2
      ├── Batch 3
      └── ...
      │
      ▼
Query Engine
      │
      ▼
SQL Compiler
      │
      ▼
Execution Engine
      │
      ▼
BulkInsertResult
```

---

# 12. Public API

Propuesta:

```php
$result = DB::table('users')->insertBulk(
    rows: $rows,
);
```

Configuración:

```php
$result = DB::table('users')->insertBulk(
    rows: $rows,
    options: new BulkInsertOptions(
        batchSize: 1000,
        transactionPolicy: BulkTransactionPolicy::WHOLE_OPERATION,
    ),
);
```

---

# 13. Streaming input

La entrada no deberá requerir necesariamente un `array`.

Contrato:

```php
/**
 * @template TRow
 */
interface BulkRowSource
{
    /**
     * @return iterable<TRow>
     */
    public function rows(): iterable;
}
```

---

# 14. Iterable support

Deberá aceptar conceptualmente:

```text
array
Iterator
Generator
LazyCollection
BulkRowSource
```

sin materializar obligatoriamente toda la fuente.

---

# 15. Ejemplo generator

```php
function users(): Generator
{
    for ($i = 1; $i <= 10_000_000; $i++) {
        yield [
            'external_id' => $i,
            'name' => "User {$i}",
        ];
    }
}

DB::table('users')->insertBulk(
    rows: users(),
    batchSize: 1000,
);
```

La memoria deberá permanecer acotada por:

```text
current input
+
current batch
+
execution overhead
```

salvo que otras capas retengan datos.

---

# 16. BulkInsertRequest

```php
final readonly class BulkInsertRequest
{
    public function __construct(
        public TableReference $table,
        public iterable $rows,
        public BulkInsertOptions $options,
        public DatabaseContext $context,
    ) {}
}
```

---

# 17. Row model

Una fila podrá normalizarse a:

```php
final readonly class BulkInsertRow
{
    /**
     * @param array<ColumnId, mixed> $values
     */
    public function __construct(
        public array $values,
    ) {}
}
```

---

# 18. Input rows ≠ SQL values

La entrada:

```php
[
    'created_at' => $instant,
]
```

no deberá convertirse manualmente en:

```text
'2026-09-10 19:00:00'
```

desde Bulk Insert.

El Type System mantiene autoridad sobre conversión.

---

# 19. Type pipeline

```text
Application Value
      ↓
Database Type System
      ↓
Canonical Persistent Value
      ↓
Platform Binding
      ↓
Compiler / Executor
```

---

# 20. Bulk Insert no será Type System

Nunca:

```text
BulkInsertRunner
→ ad hoc DateTime formatting
```

---

# 21. Column shape

Un problema fundamental:

```php
[
    ['name' => 'A', 'email' => 'a@example.com'],
    ['name' => 'B'],
]
```

Las filas tienen formas distintas.

---

# 22. Shape normalization

El sistema deberá decidir explícitamente:

```text
UNIFORM_REQUIRED
NORMALIZE_MISSING_TO_DEFAULT
GROUP_BY_SHAPE
CUSTOM
```

---

# 23. Default recomendado

Para máxima corrección:

```text
GROUP_BY_SHAPE
```

o `UNIFORM_REQUIRED`, dependiendo de la API.

---

# 24. Missing ≠ NULL

Regla crítica:

```text
Missing Column
≠
SQL NULL
```

Una columna ausente puede significar:

```text
use database default
```

mientras:

```text
column => null
```

significa insertar `NULL`.

---

# 25. Example

Tabla:

```text
status DEFAULT 'pending'
```

Fila:

```php
[
    'name' => 'Alice',
]
```

no deberá transformarse automáticamente en:

```php
[
    'name' => 'Alice',
    'status' => null,
]
```

---

# 26. ColumnShape

```php
final readonly class BulkInsertShape
{
    /**
     * @param list<ColumnId> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 27. Shape fingerprint

Filas con el mismo conjunto ordenado de columnas podrán compartir:

```text
BulkInsertShapeFingerprint
```

---

# 28. Grouping by shape

Entrada:

```text
Row A → [name,email]
Row B → [name]
Row C → [name,email]
```

podrá convertirse en:

```text
Batch Shape 1
    Row A
    Row C

Batch Shape 2
    Row B
```

si la policy permite reordenamiento.

---

# 29. Reordering caveat

Reordenar filas puede afectar:

```text
generated ID correlation
return order
side effects
diagnostics
```

Por tanto, el sistema deberá preservar un:

```text
InputRowIndex
```

---

# 30. Stable input identity

Cada fila podrá recibir:

```text
InputRowIndex = 0..N-1
```

para correlacionar resultados.

---

# 31. Reordering policy

```php
enum BulkInsertReorderingPolicy
{
    case PRESERVE_INPUT_ORDER;
    case ALLOW_SHAPE_GROUPING;
    case ALLOW_ROUTING_GROUPING;
    case FULL_SAFE_REORDER;
}
```

---

# 32. Query Model

Bulk Insert deberá producir un Query Model tipado.

Conceptualmente:

```text
InsertQueryModel
├── target table
├── columns
├── rows
├── conflict semantics
├── returning
└── metadata
```

---

# 33. Bulk Insert no genera SQL

Siempre:

```text
Bulk Insert Planner
→ Insert Query Model
→ SQL Compiler
```

Nunca:

```text
Bulk Insert Planner
→ SQL string
```

---

# 34. Multi-row insert

Una plataforma podrá compilar:

```sql
INSERT INTO users (name, email)
VALUES
    (?, ?),
    (?, ?),
    (?, ?);
```

cuando sus capabilities lo permitan.

---

# 35. Alternative execution

Otra plataforma/driver podría preferir:

```text
prepared statement
+
repeated batch execution
```

---

# 36. Capability-driven strategy

```php
enum BulkInsertExecutionStrategy
{
    case MULTI_ROW_STATEMENT;
    case PREPARED_BATCH;
    case SINGLE_ROW_LOOP;
    case NATIVE_BULK_PROTOCOL;
    case CUSTOM;
}
```

---

# 37. SINGLE_ROW_LOOP

Será fallback, no definición semántica de Bulk Insert.

Incluso si físicamente se ejecutan N statements, la operación lógica continúa siendo Bulk Insert.

---

# 38. Native bulk protocols

Futuras extensiones podrían aprovechar:

```text
PostgreSQL COPY
MySQL load mechanisms
driver-specific batch APIs
```

pero no deberán contaminar el core con vendor conditionals.

---

# 39. Native protocol ≠ standard insert

Deberá ser una capability/extension especializada con contratos propios.

---

# 40. Version ≠ Capability

Siempre:

```text
DatabaseVersion
≠
BulkInsertCapability
```

---

# 41. PlatformCapabilities

Podrán existir:

```text
supportsMultiRowInsert()
supportsInsertReturning()
supportsMultiRowReturning()
supportsDefaultKeywordInValues()
supportsNativeBulkLoad()
maxBindParameters()
maxStatementSize()
maxRowsPerInsert()
```

---

# 42. MySQL y MariaDB

Serán plataformas independientes.

No deberá asumirse:

```text
MySQL capability
=
MariaDB capability
```

---

# 43. PostgreSQL

Podrá aprovechar capacidades como `RETURNING` cuando el Platform Model confirme soporte efectivo.

---

# 44. SQLite

Los límites de parámetros y capabilities deberán respetarse dinámicamente.

---

# 45. Batch planning

El tamaño efectivo del batch será:

```text
EffectiveBatchSize
=
min(
    RequestedBatchSize,
    ParameterLimit,
    StatementLimit,
    PlatformRowLimit,
    ResourceBudget
)
```

---

# 46. Parameter calculation

Para:

```text
R = rows
C = bound columns per row
```

aproximadamente:

```text
Parameters = R × C
```

Por tanto:

```text
R <= floor(MaxParameters / C)
```

cuando todos los valores requieran binding.

---

# 47. Real calculation

Debe considerar también:

```text
expressions
defaults
returning
conflict clauses
platform overhead
reserved parameters
```

---

# 48. No hard-coded limits

No:

```php
if ($database === 'sqlite') {
    $max = 999;
}
```

en el Bulk Insert core.

Usar:

```text
PlatformCapabilitySystem
```

---

# 49. Batch size

Opciones:

```php
final readonly class BulkBatchPolicy
{
    public function __construct(
        public ?int $preferredRows = null,
        public ?int $maxRows = null,
        public ?int $maxParameters = null,
        public ?int $maxBytes = null,
    ) {}
}
```

---

# 50. Adaptive batching

Una extensión futura podrá ajustar batches según:

```text
latency
statement size
memory
server feedback
packet limits
```

---

# 51. Deterministic V1

La V1 deberá favorecer batch planning determinista y explainable.

---

# 52. Byte estimation

El tamaño exacto del statement puede no conocerse antes de compilación.

Podrá utilizarse:

```text
KNOWN
ESTIMATED
UNKNOWN
```

---

# 53. Conservative splitting

Ante incertidumbre cercana al límite, dividir batches será preferible a arriesgar un statement inválido.

---

# 54. Generated identifiers

Caso:

```text
id AUTO GENERATED
```

con 1000 filas.

El sistema deberá distinguir:

```text
Insertion Success
```

de:

```text
Generated IDs Known
```

---

# 55. No ID inference

Nunca inferir:

```text
first_id = 100
rows = 10
→ IDs 100..109
```

salvo capability explícita que garantice esa semántica.

---

# 56. Generated IDs

Estado conceptual:

```php
enum BulkGeneratedValueKnowledge
{
    case COMPLETE;
    case PARTIAL;
    case NONE;
    case UNKNOWN;
}
```

---

# 57. RETURNING

Si la plataforma soporta:

```text
INSERT ... RETURNING ...
```

el Query Model podrá solicitar:

```php
returning: ['id']
```

---

# 58. Multi-row RETURNING

No deberá asumirse que todas las plataformas:

```text
support returning
```

ni que:

```text
returned order
=
input order
```

sin garantía explícita.

---

# 59. Correlation

Cuando se necesite correlacionar:

```text
input row
↔
returned row
```

deberá existir una estrategia segura.

---

# 60. Correlation key

Podría utilizarse una clave estable proporcionada por aplicación:

```text
external_id
```

cuando sea apropiado.

---

# 61. No hidden correlation columns

VoltStack no deberá añadir columnas físicas temporales a tablas de usuario sin contrato explícito.

---

# 62. BulkInsertReturningPlan

```php
final readonly class BulkInsertReturningPlan
{
    public function __construct(
        public array $columns,
        public BulkReturnCorrelationStrategy $correlation,
    ) {}
}
```

---

# 63. Returning fallback

Si la plataforma no soporta RETURNING:

```text
REJECT
IGNORE
FOLLOW_UP_QUERY
CUSTOM
```

según policy.

---

# 64. FOLLOW_UP_QUERY risk

Una consulta posterior puede ser vulnerable a:

```text
concurrency
ambiguous matching
replica lag
```

No deberá fingir equivalencia con `RETURNING`.

---

# 65. Duplicate/conflict behavior

Bulk Insert deberá declarar qué ocurre ante conflictos.

---

# 66. ConflictPolicy

```php
enum BulkInsertConflictPolicy
{
    case ERROR;
    case IGNORE;
    case DO_NOTHING;
    case CUSTOM;
}
```

---

# 67. UPSERT

La actualización ante conflicto:

```text
INSERT ... ON CONFLICT ... DO UPDATE
```

es semánticamente más amplia.

Puede existir como:

```text
UpsertQuery
```

o extensión propia.

No deberá confundirse con el Bulk Insert básico.

---

# 68. Ignore semantics

`IGNORE` puede variar entre plataformas y potencialmente ocultar errores distintos a duplicate key.

Por eso:

```text
semantic conflict policy
```

debe compilarse cuidadosamente.

---

# 69. Platform normalization

VoltStack deberá definir semántica lógica:

```text
ON_CONFLICT_DO_NOTHING
```

y permitir al Platform Compiler decidir si puede representarla correctamente.

---

# 70. Unsupported semantics

Si una plataforma no puede representar la policy solicitada con garantías suficientes:

```text
BulkInsertUnsupportedCapabilityException
```

preferible a emulación insegura.

---

# 71. Validation

Antes de ejecutar deberán validarse:

```text
target table
column existence where metadata available
column shape
type compatibility
required fields where knowable
conflict configuration
returning configuration
routing requirements
resource limits
```

---

# 72. Validation ≠ DB constraints

La base de datos continúa siendo autoridad final para:

```text
NOT NULL
UNIQUE
CHECK
FK
```

---

# 73. Application validation

Bulk Insert no deberá ejecutar automáticamente todo el Validation System de modelos por fila.

Eso destruiría parte de su propósito y cambiaría semántica.

---

# 74. Optional validator

Puede configurarse:

```text
NONE
ROW_VALIDATOR
BATCH_VALIDATOR
CUSTOM
```

---

# 75. Invalid rows

Policies posibles:

```php
enum BulkInvalidRowPolicy
{
    case FAIL_OPERATION;
    case FAIL_BATCH;
    case SKIP_ROW;
    case COLLECT_ERRORS;
    case CUSTOM;
}
```

---

# 76. SKIP_ROW

Solo debe utilizarse si el usuario acepta explícitamente resultados parciales.

---

# 77. Transaction architecture

Bulk Insert deberá integrar:

```text
164–175 Transactions & Concurrency
```

---

# 78. Transaction policies

```php
enum BulkTransactionPolicy
{
    case NONE;
    case USE_EXISTING;
    case WHOLE_OPERATION;
    case PER_BATCH;
    case CUSTOM;
}
```

---

# 79. NONE

Cada statement utiliza semántica normal de autocommit/plataforma si no existe transacción.

---

# 80. USE_EXISTING

Utiliza una transacción externa.

Nunca la committeará.

---

# 81. WHOLE_OPERATION

El Bulk Insert Runner puede abrir una transacción para todos los batches.

```text
BEGIN
Batch 1
Batch 2
Batch 3
COMMIT
```

---

# 82. PER_BATCH

```text
BEGIN
Batch 1
COMMIT

BEGIN
Batch 2
COMMIT
```

reduce tamaño de transacción pero permite éxito parcial global.

---

# 83. Ownership rule

> **Bulk Insert solo podrá hacer commit o rollback de transacciones que él mismo haya creado.**

---

# 84. Whole-operation atomicity

Solo podrá afirmarse si:

```text
all batches
+
same transactional domain
+
same database/shard
+
supported transaction semantics
```

participan en la misma transacción.

---

# 85. Sharded bulk insert

Si filas pertenecen a:

```text
Shard A
Shard B
Shard C
```

no deberá afirmarse:

```text
WHOLE_OPERATION = global ACID
```

sin infraestructura distribuida explícita.

---

# 86. Transaction scope result

El plan deberá indicar:

```text
ATOMIC_SINGLE_DOMAIN
ATOMIC_PER_BATCH
ATOMIC_PER_SHARD
NON_ATOMIC
UNKNOWN
```

---

# 87. Failure model

Bulk Insert debe distinguir:

```text
validation failure
planning failure
routing failure
compilation failure
statement failure
batch failure
transaction failure
commit unknown
cancellation
```

---

# 88. Batch outcomes

Cada batch podrá producir:

```php
enum BulkBatchStatus
{
    case SUCCEEDED;
    case FAILED;
    case ROLLED_BACK;
    case PARTIAL;
    case UNKNOWN;
    case SKIPPED;
}
```

---

# 89. Operation outcomes

```php
enum BulkInsertStatus
{
    case SUCCEEDED;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 90. UNKNOWN preservation

Caso:

```text
COMMIT sent
↓
connection lost
```

resultado:

```text
UNKNOWN
```

No:

```text
FAILED
```

---

# 91. Retry architecture

Retry deberá diferenciar:

```text
statement retry
batch retry
whole-operation retry
```

---

# 92. Blind insert retry danger

Si:

```text
INSERT succeeded
response lost
```

reintentar puede:

```text
duplicate rows
trigger unique violation
produce duplicate side effects
```

---

# 93. UNKNOWN ≠ retry safe

Regla absoluta:

```text
UNKNOWN COMMIT
≠
SAFE TO RETRY
```

---

# 94. Replay safety

Podrá clasificarse:

```php
enum BulkReplaySafety
{
    case SAFE;
    case IDEMPOTENT_BY_CONSTRAINT;
    case IDEMPOTENT_BY_KEY;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 95. Unique constraint ≠ universal idempotency

Una unique constraint puede impedir duplicados, pero el segundo intento puede producir error.

Eso no significa automáticamente que toda la operación sea semánticamente idempotente.

---

# 96. Retry policy

Solo deberá habilitar retries cuando exista evidencia suficiente.

---

# 97. Partial failure

En una operación `PER_BATCH`:

```text
Batch 1 → committed
Batch 2 → committed
Batch 3 → failed
```

resultado global:

```text
PARTIAL
```

---

# 98. No fake rollback

VoltStack no deberá afirmar:

```text
Bulk Insert rolled back
```

si batches anteriores ya fueron committed.

---

# 99. Result

```php
final readonly class BulkInsertResult
{
    public function __construct(
        public BulkInsertStatus $status,
        public int $rowsAccepted,
        public int $rowsAttempted,
        public ?int $rowsInserted,
        public array $batchResults,
        public BulkGeneratedValuesResult $generatedValues,
        public BulkInsertOutcomeEvidence $evidence,
    ) {}
}
```

---

# 100. affected rows

`rowsInserted` puede ser:

```text
KNOWN integer
```

o desconocido dependiendo de:

```text
driver
conflict policy
platform semantics
```

---

# 101. No fabricated counts

Si el driver no permite conocer con precisión:

```text
inserted vs ignored
```

VoltStack deberá conservar:

```text
UNKNOWN
```

en lugar de inferirlo.

---

# 102. Row-level results

No deberán generarse obligatoriamente para millones de filas.

Eso rompería bounded memory.

---

# 103. Result detail policy

```php
enum BulkResultDetailLevel
{
    case SUMMARY;
    case PER_BATCH;
    case PER_ROW;
}
```

---

# 104. Default

```text
SUMMARY
```

o `PER_BATCH`.

---

# 105. PER_ROW warning

Para 100 millones de filas:

```text
100,000,000 result objects
```

es inaceptable en memoria.

Deberá utilizar:

```text
streamed result sink
external error sink
bounded collector
```

si se solicita detalle.

---

# 106. Error sink

```php
interface BulkRowErrorSink
{
    public function report(
        BulkRowFailure $failure
    ): void;
}
```

---

# 107. Sharding

Bulk Insert deberá resolver routing antes de ejecutar.

---

# 108. Row routing

Cada fila podrá mapear a:

```text
ShardId
```

mediante `Partition Routing System`.

---

# 109. Group by shard

Entrada:

```text
Row 1 → Shard B
Row 2 → Shard A
Row 3 → Shard B
```

podrá planificarse:

```text
Shard A Batch
    Row 2

Shard B Batch
    Row 1
    Row 3
```

si reordering policy lo permite.

---

# 110. Shard key missing

Una escritura ordinaria sharded deberá resolver un shard.

Si no puede:

```text
BulkInsertRoutingException
```

---

# 111. No broadcast write

Nunca:

```text
unknown shard
→ insert into every shard
```

---

# 112. Cross-shard result

El resultado deberá conservar:

```text
per-shard outcomes
```

cuando sea relevante.

---

# 113. Shard failure

```text
Shard A → committed
Shard B → failed
```

resultado:

```text
PARTIAL
```

salvo infraestructura distribuida que realmente pueda garantizar otra cosa.

---

# 114. Multitenancy

Tenant context será parte del persistence domain cuando aplique.

---

# 115. Tenant isolation

Una operación normal no mezclará filas de múltiples tenants accidentalmente.

---

# 116. Mixed-tenant input

Si se permite en una API administrativa:

```text
Tenant A rows
Tenant B rows
```

deberá particionarse explícitamente por tenant/domain.

---

# 117. No context leakage

En persistent runtimes:

```text
Batch A → Tenant A
Batch B → Tenant B
```

solo será válido si el plan lo declara.

Nunca por contaminación de contexto.

---

# 118. Read/write routing

Bulk Insert es:

```text
WRITE
```

Por tanto deberá ir a:

```text
writer
```

nunca a una replica de lectura.

---

# 119. Sticky connection

Después de insertar, el sistema podrá activar:

```text
read-your-writes / sticky semantics
```

según `180_DATABASE_STICKY_CONNECTION_SYSTEM.md`.

---

# 120. Cache invalidation

Bulk Insert deberá participar en:

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM
```

---

# 121. Semantic changes

Conceptualmente:

```text
BulkRowsInserted
TableChanged
EntityTypeChanged
QueryDependencyChanged
```

según nivel de conocimiento.

---

# 122. No per-row invalidation explosion

Para millones de filas:

```text
1 invalidation event per row
```

puede ser inviable.

---

# 123. Invalidation precision

Podrá escalar:

```text
ENTITY_EXACT
→
KEY_SET
→
TABLE_WIDE
→
DOMAIN_WIDE
```

según cardinalidad y conocimiento.

---

# 124. Correctness > cache precision

Cuando mantener invalidación exacta sea impráctico:

```text
broader invalidation
```

es preferible a dejar cache incorrecta.

---

# 125. Transaction outcome

Invalidación compartida solo después de:

```text
confirmed commit
```

---

# 126. Rollback

Cancela invalidaciones pendientes.

---

# 127. UNKNOWN

Ante commit desconocido:

```text
conservative invalidation
```

según Cache Consistency System.

---

# 128. Entity Cache

Bulk Insert directo no deberá poblar automáticamente L2 Entity Cache con entidades inexistentes en el current ORM scope.

---

# 129. IdentityMap

Bulk Insert no registra objetos en:

```text
IdentityMap
```

por default.

---

# 130. Existing managed entities

Caso peligroso:

```text
IdentityMap contains User(id=10)
```

y Bulk Insert intenta insertar otro `User(id=10)`.

La DB determinará constraint outcome.

Bulk Insert no deberá manipular el objeto existente para fingir reconciliación.

---

# 131. ORM coherence

Después de Bulk Insert, un EntityManager ya activo puede tener:

```text
stale query assumptions
stale relationship collections
```

aunque no tenga las nuevas entidades.

---

# 132. ORM bulk mutation notification

Podrá emitirse una señal interna:

```text
BulkPersistenceMutation
```

para que ORM context decida:

```text
invalidate assumptions
mark collections stale
require refresh
```

según política.

---

# 133. No automatic full clear

Bulk Insert no hará:

```php
$entityManager->clear();
```

silenciosamente.

---

# 134. Lifecycle events

Bulk Insert no deberá fingir:

```text
PrePersist
PostPersist
```

por cada entidad si no existen entidades.

---

# 135. Bulk events

En su lugar:

```text
BulkInsertPlanned
BulkInsertStarted
BulkInsertBatchStarted
BulkInsertBatchCompleted
BulkInsertCommitted
BulkInsertCompleted
BulkInsertFailed
```

---

# 136. ORM lifecycle hooks

Si el desarrollador requiere:

```text
entity constructors
domain methods
PrePersist
relationship cascade
UoW
```

deberá utilizar ORM persistence, no Bulk Insert directo.

---

# 137. Domain invariants

Bulk Insert puede saltarse invariantes encapsuladas en entidades.

Por eso deberá considerarse una API avanzada.

---

# 138. Model convenience API

Si existe:

```php
User::bulkInsert($rows);
```

deberá resolver metadata de:

```text
User → table/mapping/types
```

pero no construir 100,000 `User` objects.

---

# 139. Model API ≠ ORM lifecycle

La documentación deberá dejar claro:

```text
User::bulkInsert()
```

es:

```text
bulk row persistence using User mapping
```

no:

```text
100,000 User::save()
```

---

# 140. Entity API

Para Data Mapper podría existir:

```php
$entityManager->bulkInsert(
    User::class,
    $rows
);
```

pero deberá conservar la misma semántica explícita.

---

# 141. Better naming

También podrá exponerse mediante:

```text
BulkPersistenceManager
```

para evitar confusión con `EntityManager`.

---

# 142. Relationships

Bulk Insert de rows no ejecutará automáticamente:

```text
relationship graph persistence
cascade persist
orphan removal
many-to-many join synchronization
```

---

# 143. Foreign keys

Las FK values deberán proporcionarse explícitamente o derivarse mediante mapping permitido.

La DB conserva autoridad de integridad.

---

# 144. Generated parent IDs

Caso:

```text
insert parents
↓
need generated IDs
↓
insert children
```

requiere un workflow explícito.

No deberá ocultarse dentro de un simple Bulk Insert.

---

# 145. Bulk graph insertion

Podría diseñarse en una futura extensión, pero:

```text
Bulk Graph Insert
≠
Bulk Insert
```

---

# 146. Value Objects

Si se usa entity metadata:

```text
EmailAddress
Money
UUID
```

deberán convertirse mediante:

```text
Type System / Mapping
```

no mediante casts ad hoc.

---

# 147. JSON

JSON deberá pasar por:

```text
161_DATABASE_JSON_TYPE_SYSTEM
```

---

# 148. Date/time

Temporal values deberán pasar por:

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM
```

---

# 149. Enums

Enums deberán pasar por:

```text
159_DATABASE_ENUM_MAPPING_SYSTEM
```

---

# 150. Custom types

Deberán respetar:

```text
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM
```

---

# 151. Raw expressions

Podrá existir escape hatch:

```php
BulkValue::expression(...)
```

pero deberá seguir:

```text
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM
```

---

# 152. Security

Nunca concatenar valores del usuario en SQL.

---

# 153. Bindings

Todos los valores normales deberán utilizar:

```text
typed parameter binding
```

---

# 154. Identifier security

Table/column identifiers no deberán provenir de valores arbitrarios sin validación/resolución.

---

# 155. Sensitive data

Telemetry no deberá registrar:

```text
passwords
tokens
PII
full row payloads
```

por default.

---

# 156. Credential handling

Bulk Insert nunca tendrá acceso especial a credenciales fuera de Connection System.

---

# 157. Cancellation

El runner deberá aceptar:

```text
CancellationToken
```

---

# 158. Cancellation boundaries

Podrá verificarse:

```text
before source read
before batch planning
before batch execution
between batches
before transaction commit
```

---

# 159. Mid-statement cancellation

Dependerá de:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM
```

y capabilities del driver.

---

# 160. Cancellation after committed batches

Con `PER_BATCH`:

```text
Batch 1 committed
Batch 2 committed
cancel
```

resultado:

```text
PARTIAL / CANCELLED_WITH_COMMITTED_WORK
```

según modelo final.

Nunca `ROLLED_BACK`.

---

# 161. BulkInsertStatus detail

Puede acompañarse de:

```text
CompletionReason
```

para distinguir:

```text
EXHAUSTED_INPUT
CANCELLED
FAILED
RESOURCE_LIMIT
DEADLINE
UNKNOWN
```

---

# 162. Input failure

Un Generator puede lanzar excepción mientras produce filas.

Esto será:

```text
INPUT_SOURCE_FAILURE
```

no `DATABASE_FAILURE`.

---

# 163. Input source ownership

Bulk Insert no deberá cerrar recursos externos que no posee.

---

# 164. Lazy Collection input

Podrá consumir:

```text
LazyCollection<Row>
```

respetando su resource lifecycle.

---

# 165. Backpressure

Bulk Insert naturalmente podrá producir:

```text
read batch
↓
insert batch
↓
read next batch
```

evitando consumir la fuente más rápido de lo necesario.

---

# 166. Prefetch

Una optimización futura podrá prefetch el siguiente input batch.

Será bounded y policy-driven.

---

# 167. Resource governance

Deberá controlar:

```text
batch rows
batch bytes
parameters
memory
duration
total rows
open resources
concurrency
```

---

# 168. Resource budget

```php
final readonly class BulkInsertResourceBudget
{
    public function __construct(
        public ?int $maxRows,
        public ?int $maxBatchRows,
        public ?int $maxBatchBytes,
        public ?int $maxMemoryBytes,
        public ?int $maxDurationMs,
    ) {}
}
```

---

# 169. maxRows

Puede impedir accidentalmente:

```text
unbounded production insert
```

desde una fuente infinita.

---

# 170. Infinite source

Bulk Insert deberá poder detectar únicamente algunos casos.

Un Generator arbitrario puede ser infinito.

Por eso `maxRows`/cancellation/deadline son importantes.

---

# 171. Infinite iterable ≠ error by definition

Puede ser válido para un proceso controlado, pero una operación síncrona deberá tener policies claras.

---

# 172. Persistent runtimes

En FrankenPHP:

```text
Request A bulk insert
Request B bulk insert
```

deberán mantener aislados:

```text
input state
current batch
transaction
tenant
shard
result aggregation
cancellation
```

---

# 173. No static mutable batch

Nunca:

```php
BulkInsertRunner::$currentBatch
```

---

# 174. RoadRunner/OpenSwoole

La misma arquitectura deberá funcionar sin state leakage.

---

# 175. Coroutine concurrency

Dos bulk inserts concurrentes deberán poseer:

```text
independent BulkInsertContext
```

---

# 176. Shared infrastructure

Podrá compartirse:

```text
immutable metadata
compiled type metadata
platform capabilities
stateless planners
connection pools
```

---

# 177. No shared transaction context

Nunca entre operaciones independientes.

---

# 178. Concurrency

Dos Bulk Inserts pueden competir por:

```text
unique constraints
indexes
locks
storage
```

---

# 179. Database authority

No intentar resolver globalmente en memoria:

```text
"this value is unique"
```

como sustituto de constraint DB.

---

# 180. Deadlocks

Bulk Insert deberá integrar:

```text
171_DATABASE_DEADLOCK_HANDLING_SYSTEM
```

---

# 181. Retry boundary

Para deadlock:

```text
whole transaction retry
```

puede ser correcto cuando sea replay-safe.

No reintentar un statement arbitrariamente dentro de una transacción ya abortada.

---

# 182. Ordering and deadlocks

Ordenar filas por ciertas keys puede reducir deadlocks en algunos workloads.

Pero cualquier reorder deberá respetar:

```text
BulkInsertReorderingPolicy
```

---

# 183. Query timeout

Cada batch execution deberá respetar:

```text
QueryTimeoutPolicy
```

---

# 184. Operation deadline

Además:

```text
BulkInsertDeadline
```

podrá limitar toda la operación.

---

# 185. Telemetry

Eventos:

```text
BulkInsertPlanned
BulkInsertStarted
BulkInsertInputStarted
BulkInsertBatchPlanned
BulkInsertBatchStarted
BulkInsertBatchSucceeded
BulkInsertBatchFailed
BulkInsertBatchRetried
BulkInsertTransactionCommitted
BulkInsertTransactionRolledBack
BulkInsertOutcomeUnknown
BulkInsertCompleted
BulkInsertCancelled
BulkInsertFailed
```

---

# 186. Métricas

```text
db.bulk_insert.operations
db.bulk_insert.duration
db.bulk_insert.rows
db.bulk_insert.batches
db.bulk_insert.batch_size
db.bulk_insert.failures
db.bulk_insert.retries
db.bulk_insert.partial
db.bulk_insert.unknown
```

---

# 187. Cardinalidad

Labels permitidos deberán ser bounded, por ejemplo:

```text
platform
strategy
status
transaction_policy
```

Evitar:

```text
tenant_id
row_id
email
raw_sql
raw_table_from_user_input
```

---

# 188. Diagnostics

API conceptual:

```php
DB::bulk()->explainInsert(
    table: 'users',
    sample: [
        ['name' => 'A', 'email' => 'a@example.com'],
    ],
    estimatedRows: 1_000_000,
);
```

---

# 189. Explain output

```text
BULK INSERT PLAN

Target:
    users

Estimated Rows:
    1,000,000

Input:
    STREAMING

Columns:
    name
    email

Shape:
    UNIFORM

Execution Strategy:
    MULTI_ROW_STATEMENT

Preferred Batch:
    1000 rows

Effective Batch:
    500 rows

Limiting Capability:
    MAX_BIND_PARAMETERS

Transaction:
    PER_BATCH

Returning:
    NONE

Conflict Policy:
    ERROR

Routing:
    WRITER
    SINGLE SHARD

Result Detail:
    PER_BATCH

Cache Invalidation:
    TABLE/REGION COALESCED

ORM Lifecycle:
    BYPASSED

IdentityMap:
    NOT POPULATED

Replay Safety:
    UNKNOWN
```

---

# 190. Plan

```php
final readonly class BulkInsertPlan
{
    public function __construct(
        public TableReference $target,
        public BulkInsertExecutionStrategy $strategy,
        public BulkBatchPlan $batch,
        public BulkRoutingPlan $routing,
        public BulkTransactionPlan $transaction,
        public BulkInsertReturningPlan $returning,
        public BulkInsertConflictPlan $conflict,
        public BulkResultPlan $result,
        public BulkResourcePlan $resources,
    ) {}
}
```

---

# 191. Plan immutability

`BulkInsertPlan` deberá ser immutable una vez validado.

---

# 192. Planner ≠ Runner

```text
BulkInsertPlanner
≠
BulkInsertRunner
```

Planner decide.

Runner ejecuta.

---

# 193. Runner ≠ Query Executor

Runner coordina batches.

Cada query sigue siendo ejecutada por:

```text
Query Executor
```

---

# 194. Compiler authority

SQL continuará perteneciendo a:

```text
SQL Compiler
```

---

# 195. Driver authority

Protocol-level execution continuará perteneciendo a:

```text
Driver
```

---

# 196. Directory structure

```text
src/Quantum/Database/Bulk/
│
├── BulkOperation.php
├── BulkOperationStatus.php
├── BulkOperationContext.php
│
├── Insert/
│   ├── BulkInsertRequest.php
│   ├── BulkInsertOptions.php
│   ├── BulkInsertRow.php
│   ├── BulkInsertShape.php
│   ├── BulkInsertPlanner.php
│   ├── BulkInsertPlan.php
│   ├── BulkInsertRunner.php
│   ├── BulkInsertResult.php
│   ├── BulkInsertStatus.php
│   │
│   ├── Input/
│   │   ├── BulkRowSource.php
│   │   ├── BulkInsertInputNormalizer.php
│   │   └── BulkInputRowIndex.php
│   │
│   ├── Shape/
│   │   ├── BulkShapeResolver.php
│   │   ├── BulkShapePolicy.php
│   │   └── BulkInsertReorderingPolicy.php
│   │
│   ├── Batch/
│   │   ├── BulkBatchPolicy.php
│   │   ├── BulkBatchPlanner.php
│   │   ├── BulkBatchPlan.php
│   │   └── BulkBatchResult.php
│   │
│   ├── Strategy/
│   │   ├── BulkInsertExecutionStrategy.php
│   │   └── BulkInsertStrategyResolver.php
│   │
│   ├── Conflict/
│   │   ├── BulkInsertConflictPolicy.php
│   │   └── BulkInsertConflictPlan.php
│   │
│   ├── Returning/
│   │   ├── BulkInsertReturningPlan.php
│   │   ├── BulkGeneratedValuesResult.php
│   │   ├── BulkGeneratedValueKnowledge.php
│   │   └── BulkReturnCorrelationStrategy.php
│   │
│   ├── Transaction/
│   │   ├── BulkTransactionPolicy.php
│   │   └── BulkTransactionPlan.php
│   │
│   ├── Routing/
│   │   ├── BulkRoutingPlan.php
│   │   └── BulkRowRoutingResolver.php
│   │
│   ├── Result/
│   │   ├── BulkResultDetailLevel.php
│   │   ├── BulkRowErrorSink.php
│   │   └── BulkInsertOutcomeEvidence.php
│   │
│   ├── Resource/
│   │   ├── BulkInsertResourceBudget.php
│   │   └── BulkResourcePlan.php
│   │
│   ├── Diagnostics/
│   │   ├── BulkInsertInspector.php
│   │   └── BulkInsertExplainer.php
│   │
│   ├── Telemetry/
│   │   └── BulkInsertTelemetry.php
│   │
│   └── Exception/
│       └── ...
│
└── Contract/
    └── ...
```

---

# 197. Error hierarchy

```text
DatabaseException
└── BulkOperationException
    └── BulkInsertException
        ├── BulkInsertInputException
        ├── BulkInsertValidationException
        ├── BulkInsertShapeException
        ├── BulkInsertPlanningException
        ├── BulkInsertUnsupportedCapabilityException
        ├── BulkInsertRoutingException
        ├── BulkInsertBatchException
        ├── BulkInsertExecutionException
        ├── BulkInsertReturningException
        ├── BulkInsertConflictException
        ├── BulkInsertTransactionException
        ├── BulkInsertRetryUnsafeException
        ├── BulkInsertResourceException
        └── BulkInsertOutcomeUnknownException
```

---

# 198. Testing matrix

| Área | Caso |
|---|---|
| Basic | one row |
| Basic | multiple rows |
| Basic | empty source |
| Input | array |
| Input | iterator |
| Input | generator |
| Input | LazyCollection |
| Shape | uniform |
| Shape | missing column |
| Shape | NULL vs missing |
| Shape | grouping |
| Types | enum |
| Types | JSON |
| Types | datetime |
| Types | value object |
| Batch | parameter limit |
| Batch | row limit |
| Batch | byte limit |
| Batch | final partial batch |
| Strategy | multi-row |
| Strategy | prepared batch |
| Strategy | fallback |
| Returning | supported |
| Returning | unsupported |
| Returning | generated ID |
| Conflict | error |
| Conflict | do nothing |
| Transaction | none |
| Transaction | existing |
| Transaction | whole |
| Transaction | per batch |
| Failure | first batch |
| Failure | middle batch |
| Failure | commit unknown |
| Retry | safe |
| Retry | unsafe |
| Routing | writer |
| Sharding | one shard |
| Sharding | multiple shards |
| Tenant | isolation |
| Cache | invalidation |
| ORM | IdentityMap unchanged |
| Runtime | worker reuse |
| Cancellation | between batches |
| Resource | max rows |
| Telemetry | bounded labels |

---

# 199. Architectural invariants

## DB-BINS-001
Bulk Insert será distinto de ORM Persist Loop.

## DB-BINS-002
Bulk Insert será distinto de Batch Persistence.

## DB-BINS-003
Bulk Insert será distinto de Seeder.

## DB-BINS-004
Bulk Insert será distinto de Factory.

## DB-BINS-005
Bulk Insert será distinto de Import.

## DB-BINS-006
Bulk Insert será distinto de Upsert.

## DB-BINS-007
Bulk Insert será distinto de Migration.

## DB-BINS-008
Bulk Insert no generará SQL directamente.

## DB-BINS-009
Bulk Insert utilizará Query Engine.

## DB-BINS-010
SQL Compiler continuará siendo autoridad SQL.

## DB-BINS-011
Execution Engine continuará siendo autoridad de ejecución.

## DB-BINS-012
Driver continuará siendo autoridad protocol-level.

## DB-BINS-013
Bulk Insert podrá aceptar input incremental.

## DB-BINS-014
Bulk Insert no requerirá materializar todo el input.

## DB-BINS-015
Input Generator podrá consumirse incrementalmente.

## DB-BINS-016
Bulk Insert mantendrá batches acotados.

## DB-BINS-017
Input value será distinto de SQL value.

## DB-BINS-018
Type System continuará siendo autoridad de conversiones.

## DB-BINS-019
Bulk Insert no implementará casts temporales ad hoc.

## DB-BINS-020
Missing column será distinta de NULL.

## DB-BINS-021
Database default será preservable.

## DB-BINS-022
Row shape será explícita.

## DB-BINS-023
Heterogeneous shapes no serán normalizadas incorrectamente.

## DB-BINS-024
Shape grouping será policy-driven.

## DB-BINS-025
Reordering será explícito.

## DB-BINS-026
Input row identity podrá preservarse mediante index.

## DB-BINS-027
Multi-row insert será capability-driven.

## DB-BINS-028
Prepared batch será capability-driven.

## DB-BINS-029
Native bulk protocol será una extensión especializada.

## DB-BINS-030
Fallback single-row no cambiará semántica lógica de Bulk Insert.

## DB-BINS-031
Version será distinta de Capability.

## DB-BINS-032
MySQL y MariaDB no compartirán capabilities por suposición.

## DB-BINS-033
PostgreSQL capabilities serán resueltas por Platform.

## DB-BINS-034
SQLite limits serán capability-driven.

## DB-BINS-035
Batch size efectiva respetará parameter limits.

## DB-BINS-036
Batch size respetará statement limits.

## DB-BINS-037
Batch size respetará resource budget.

## DB-BINS-038
No existirán límites vendor hard-coded en core.

## DB-BINS-039
Adaptive batching será opcional.

## DB-BINS-040
V1 favorecerá planning determinista.

## DB-BINS-041
UNKNOWN size estimate permanecerá UNKNOWN.

## DB-BINS-042
Generated ID no será inferido por secuencia aritmética sin garantía.

## DB-BINS-043
Insertion success será distinto de generated-ID knowledge.

## DB-BINS-044
RETURNING será capability-driven.

## DB-BINS-045
Multi-row returning order no será asumido sin garantía.

## DB-BINS-046
Input/result correlation será explícita.

## DB-BINS-047
Bulk Insert no añadirá hidden physical columns.

## DB-BINS-048
Returning fallback no fingirá equivalencia.

## DB-BINS-049
Conflict behavior será explícito.

## DB-BINS-050
Upsert no será confundido con Insert.

## DB-BINS-051
Vendor-specific IGNORE no definirá semántica portable.

## DB-BINS-052
Unsupported semantics producirán error claro.

## DB-BINS-053
Pre-validation no sustituirá DB constraints.

## DB-BINS-054
DB continuará siendo autoridad final de integridad.

## DB-BINS-055
Model validation no se ejecutará automáticamente por fila.

## DB-BINS-056
Optional row validation será configurable.

## DB-BINS-057
Skip invalid row requerirá policy explícita.

## DB-BINS-058
Transaction policy será explícita.

## DB-BINS-059
NONE no abrirá transaction.

## DB-BINS-060
USE_EXISTING no committeará transaction externa.

## DB-BINS-061
WHOLE_OPERATION solo cerrará transaction propia.

## DB-BINS-062
PER_BATCH podrá producir outcome parcial.

## DB-BINS-063
Transaction ownership será respetado.

## DB-BINS-064
Whole-operation atomicity solo se afirmará con evidencia suficiente.

## DB-BINS-065
Cross-shard operation no fingirá global ACID.

## DB-BINS-066
Atomicity scope será reportable.

## DB-BINS-067
Batch outcomes serán explícitos.

## DB-BINS-068
Global outcome será explícito.

## DB-BINS-069
UNKNOWN commit permanecerá UNKNOWN.

## DB-BINS-070
UNKNOWN no será transformado en FAILED.

## DB-BINS-071
Retry boundary será explícito.

## DB-BINS-072
Blind retry después de unknown insert será prohibido.

## DB-BINS-073
Replay safety será evaluable.

## DB-BINS-074
Unique constraint no significará automáticamente semantic idempotency.

## DB-BINS-075
Partial committed work no será reportado como rolled back.

## DB-BINS-076
Rows inserted no serán inventadas.

## DB-BINS-077
Driver uncertainty será preservada.

## DB-BINS-078
Per-row results no serán default.

## DB-BINS-079
Result detail será configurable.

## DB-BINS-080
Per-row detail podrá utilizar external sink.

## DB-BINS-081
Bulk Insert será una write operation.

## DB-BINS-082
Bulk Insert nunca será routed a read replica.

## DB-BINS-083
Shard routing ocurrirá antes de ejecución.

## DB-BINS-084
Unknown shard no producirá broadcast write.

## DB-BINS-085
Rows podrán agruparse por shard cuando sea seguro.

## DB-BINS-086
Shard reordering respetará input correlation.

## DB-BINS-087
Per-shard outcomes serán preservados.

## DB-BINS-088
Tenant isolation será obligatoria.

## DB-BINS-089
Mixed tenant operations serán explícitas.

## DB-BINS-090
Persistent runtime no filtrará tenant state.

## DB-BINS-091
Sticky/read-your-writes podrá activarse tras write.

## DB-BINS-092
Bulk Insert participará en cache invalidation.

## DB-BINS-093
Cache invalidation seguirá confirmed transaction outcome.

## DB-BINS-094
Rollback cancelará invalidaciones pendientes.

## DB-BINS-095
UNKNOWN podrá provocar invalidación conservadora.

## DB-BINS-096
Invalidation no requerirá un evento por fila.

## DB-BINS-097
Invalidation podrá escalar a table/domain precision.

## DB-BINS-098
Correctness tendrá prioridad sobre cache precision.

## DB-BINS-099
Bulk Insert no poblará IdentityMap automáticamente.

## DB-BINS-100
Bulk Insert no creará managed entities automáticamente.

## DB-BINS-101
Bulk Insert no reconciliará objetos managed silenciosamente.

## DB-BINS-102
Bulk mutation podrá notificar stale ORM assumptions.

## DB-BINS-103
Bulk Insert no hará EntityManager::clear() automáticamente.

## DB-BINS-104
Bulk Insert no fingirá PrePersist/PostPersist por fila.

## DB-BINS-105
Bulk-specific events serán distintos de ORM lifecycle events.

## DB-BINS-106
Si se requieren lifecycle semantics completas deberá usarse ORM persistence.

## DB-BINS-107
Model::bulkInsert no equivaldrá a Model::save repetido.

## DB-BINS-108
Entity bulk API no creará segundo persistence engine.

## DB-BINS-109
Bulk Insert no persistirá relationship graphs automáticamente.

## DB-BINS-110
Cascade persist no será ejecutado automáticamente.

## DB-BINS-111
Many-to-many synchronization no será automática.

## DB-BINS-112
Foreign key integrity seguirá perteneciendo a DB.

## DB-BINS-113
Generated parent-child graph requerirá workflow explícito.

## DB-BINS-114
Bulk Graph Insert será sistema distinto.

## DB-BINS-115
Enums usarán Type System.

## DB-BINS-116
JSON usará Type System.

## DB-BINS-117
Date/time usará Type System.

## DB-BINS-118
Value Objects usarán Mapping/Type System.

## DB-BINS-119
Custom Types serán respetados.

## DB-BINS-120
Raw expressions serán escape hatch explícito.

## DB-BINS-121
Normal values usarán bindings.

## DB-BINS-122
Bulk Insert no concatenará user values en SQL.

## DB-BINS-123
Identifiers serán validados/resueltos.

## DB-BINS-124
Telemetry no expondrá row payloads.

## DB-BINS-125
Sensitive values no serán loggeados por default.

## DB-BINS-126
Cancellation será soportada.

## DB-BINS-127
Cancellation no revertirá batches ya committed.

## DB-BINS-128
Mid-statement cancellation dependerá del Execution System.

## DB-BINS-129
Input source failure será distinto de DB failure.

## DB-BINS-130
Bulk Insert respetará ownership de input resources.

## DB-BINS-131
LazyCollection input podrá consumirse incrementalmente.

## DB-BINS-132
Backpressure será preservable.

## DB-BINS-133
Prefetch será bounded y opcional.

## DB-BINS-134
Resource Governance controlará batch sizes.

## DB-BINS-135
Resource Governance podrá limitar total rows.

## DB-BINS-136
Infinite input deberá poder controlarse mediante budget/cancellation.

## DB-BINS-137
Persistent runtime state será operation-scoped.

## DB-BINS-138
Current batch no será static mutable state.

## DB-BINS-139
TransactionContext no se compartirá entre operaciones.

## DB-BINS-140
FrankenPHP worker reuse será seguro.

## DB-BINS-141
RoadRunner worker reuse será seguro.

## DB-BINS-142
OpenSwoole worker reuse será seguro.

## DB-BINS-143
Coroutine bulk operations estarán aisladas.

## DB-BINS-144
Immutable metadata podrá compartirse.

## DB-BINS-145
Concurrent inserts confiarán en DB constraints para integridad final.

## DB-BINS-146
In-memory uniqueness no sustituirá unique constraint.

## DB-BINS-147
Deadlock handling utilizará Transaction System.

## DB-BINS-148
Retry después de deadlock respetará transaction boundaries.

## DB-BINS-149
Row ordering para reducir contention será policy-driven.

## DB-BINS-150
Query timeout será respetado.

## DB-BINS-151
Operation deadline podrá ser independiente.

## DB-BINS-152
BulkInsertPlan será immutable.

## DB-BINS-153
Planner será distinto de Runner.

## DB-BINS-154
Runner será distinto de Executor.

## DB-BINS-155
Executor será distinto de Driver.

## DB-BINS-156
Bulk Insert no conocerá PDO directamente.

## DB-BINS-157
Bulk Insert no conocerá concrete runtime directamente.

## DB-BINS-158
Bulk Insert será explainable.

## DB-BINS-159
Effective batch size será diagnosticable.

## DB-BINS-160
Limiting capability será diagnosticable.

## DB-BINS-161
Atomicity scope será diagnosticable.

## DB-BINS-162
Replay safety será diagnosticable.

## DB-BINS-163
ORM lifecycle bypass será visible en diagnostics.

## DB-BINS-164
Cache invalidation strategy será explainable.

## DB-BINS-165
Telemetry tendrá bounded cardinality.

## DB-BINS-166
Raw row IDs no serán metric labels.

## DB-BINS-167
Bulk Insert podrá procesar millones de filas sin materializarlas todas.

## DB-BINS-168
Bounded batch no garantizará bounded total memory si el caller retiene datos.

## DB-BINS-169
Bulk Insert optimizará throughput sin sacrificar correctness contracts.

## DB-BINS-170
Bulk Insert nunca será database truth; la base de datos mantiene la autoridad final.

---

# 200. Modelo formal

Sea:

```text
R = {r1, r2, ..., rn}
```

el conjunto/secuencia de filas de entrada.

El planner construye una partición:

```text
B = {B1, B2, ..., Bk}
```

tal que:

```text
⋃ Bi = R
```

salvo filas rechazadas explícitamente por una validation policy.

---

# 201. Batch constraints

Para cada batch:

```text
|Bi| <= EffectiveBatchSize
```

y:

```text
Parameters(Bi)
<=
PlatformParameterLimit
```

además de otros límites conocidos.

---

# 202. Routing partition

En arquitectura sharded:

```text
R
→
R_s1 ∪ R_s2 ∪ ... ∪ R_sm
```

donde:

```text
R_si
```

contiene únicamente filas cuyo routing pertenece al shard `si`.

---

# 203. Shape partition

Dentro de cada dominio:

```text
R_si
→
ShapeGroup_1
ShapeGroup_2
...
```

cuando la estrategia de shape grouping esté habilitada.

---

# 204. Final planning

Conceptualmente:

```text
Rows
↓
Persistence Domain
↓
Shard
↓
Column Shape
↓
Batch Capacity
↓
Execution Batch
```

---

# 205. Outcome aggregation

Sea:

```text
O(Bi)
```

el outcome de cada batch.

El outcome global:

```text
O(Operation)
=
Aggregate(
    O(B1),
    O(B2),
    ...,
    O(Bk),
    TransactionPolicy
)
```

---

# 206. Atomic operation

Si todos los batches pertenecen a una única transacción confirmada:

```text
Commit = CONFIRMED
```

entonces puede afirmarse:

```text
Operation = SUCCEEDED
```

si todos los statements requeridos tuvieron éxito.

---

# 207. Unknown commit

Si:

```text
Commit = UNKNOWN
```

entonces:

```text
Operation = UNKNOWN
```

aunque todos los statements previos hubieran reportado éxito.

---

# 208. Partial model

Para `PER_BATCH`:

```text
Committed(B1)
∧
Committed(B2)
∧
Failed(B3)
```

implica:

```text
Operation = PARTIAL
```

---

# 209. Memory model

Para input streaming y batch máximo `K`:

```text
M_bulk
≈
M(current batch)
+
M(bindings)
+
M(compiled statement)
+
M(result detail)
+
M(source overhead)
```

No depende necesariamente de `N`, número total de filas.

---

# 210. Qualification

Sin embargo:

```text
PER_ROW result collection
```

puede convertir memoria en:

```text
O(N)
```

Por eso el detalle de resultado será explícito.

---

# 211. Arquitectura final

```text
                   Bulk Insert API
                          │
                          ▼
                  BulkInsertRequest
                          │
                          ▼
                Input Normalization
                          │
                          ▼
                 Type Resolution
                          │
                          ▼
                 Routing Resolver
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
             Tenant              Shard
                │                   │
                └─────────┬─────────┘
                          ▼
                  Shape Resolver
                          │
                          ▼
                   Batch Planner
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
       Capabilities   Resources     Transaction
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                   BulkInsertPlan
                          │
                          ▼
                   BulkInsertRunner
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
          Batch N                  Batch N+1
             │
             ▼
       InsertQueryModel
             │
             ▼
        SQL Compiler
             │
             ▼
      Execution Engine
             │
             ▼
      Transaction Outcome
             │
             ▼
      Cache/Event Effects
             │
             ▼
      BulkInsertResult
```

---

# 212. Regla maestra final

> **Bulk Insert optimizará la persistencia de grandes conjuntos de filas sin falsificar las garantías del ORM, de las transacciones, de los identificadores generados ni de los outcomes distribuidos.**

Siempre:

```text
Bulk Insert
≠
ORM Persist Loop
```

```text
Bulk Insert
≠
Batch Persistence
```

```text
Bulk Insert
≠
Import
```

```text
Bulk Insert
≠
Upsert
```

```text
Inserted
≠
Managed Entity
```

```text
Statement Success
≠
Transaction Commit
```

```text
Generated ID
≠
Inferred Sequence
```

```text
PER_BATCH
≠
Whole Operation Atomicity
```

```text
Cross-Shard Bulk Insert
≠
Global ACID Transaction
```

y:

```text
UNKNOWN Commit
≠
Safe Retry
```

---

# 213. Resultado arquitectónico

Con este sistema VoltStack podrá manejar:

```php
DB::table('telemetry_events')->insertBulk(
    rows: $events,
    options: new BulkInsertOptions(
        batchSize: 2000,
        transactionPolicy: BulkTransactionPolicy::PER_BATCH,
    ),
);
```

para:

```text
10 rows
10,000 rows
10,000,000 rows
100,000,000 rows
```

sin exigir que todas las filas existan simultáneamente en memoria.

La misma infraestructura podrá ser reutilizada posteriormente por:

```text
Seeder System
Test Data Generation
Import System
ETL
Data Migration Tools
Administrative Operations
Large Dataset Processing
```

sin duplicar:

```text
Query Engine
Type System
Compiler
Execution
Transactions
Routing
Cache Invalidation
Telemetry
```

---

# 214. Relación con Bulk Update y Bulk Delete

Bulk Insert introduce la infraestructura común:

```text
Bulk Operation
├── Input
├── Planning
├── Batching
├── Routing
├── Transaction
├── Failure
├── Result
├── Resource Governance
└── Telemetry
```

que deberá ser reutilizada por:

```text
204 Bulk Update
205 Bulk Delete
```

pero con diferencias fundamentales.

Insert trabaja principalmente con:

```text
new rows
```

Update deberá trabajar con:

```text
existing row selection
+
mutation specification
```

y Delete con:

```text
existing row selection
+
removal semantics
```

---

# 215. Siguiente documento

```text
204_DATABASE_BULK_UPDATE_SYSTEM.md
```

El siguiente documento definirá:

```text
Bulk Update System
├── set-based updates
├── key-based updates
├── heterogeneous row updates
├── CASE-based strategies
├── temporary-table strategies
├── batch update planning
├── predicates
├── optimistic locking
├── affected-row semantics
├── ORM synchronization
├── stale managed entities
├── relationship implications
├── soft-delete boundaries
├── transaction policies
├── retries
├── sharding
├── cache invalidation
├── events
├── telemetry
└── resource governance
```

estableciendo especialmente:

```text
Bulk Update
≠
foreach Entity::save()
```

```text
Bulk Update
≠
UnitOfWork Change Tracking
```

y la regla central:

> **Bulk Update modificará conjuntos de registros directamente mediante operaciones tipadas y planificadas, sin simular que el UnitOfWork observó individualmente cambios que nunca atravesaron el ciclo normal de entidades.**