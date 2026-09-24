# 201_DATABASE_CHUNK_PROCESSING_SYSTEM.md

# VoltStack Quantum Database
## Database Chunk Processing System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 201 — Database Chunk Processing System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `200_DATABASE_CURSOR_PAGINATION_SYSTEM.md`  
**Siguiente documento:** `202_DATABASE_LAZY_COLLECTION_SYSTEM.md`

---

# 1. Propósito

`Database Chunk Processing System` define la arquitectura mediante la cual VoltStack podrá recorrer y procesar datasets grandes en unidades acotadas sin materializar el conjunto completo en memoria.

Ejemplo:

```php
User::query()
    ->where('active', true)
    ->orderBy('id')
    ->chunk(500, function (Chunk $chunk): void {
        foreach ($chunk as $user) {
            // Process user
        }
    });
```

También:

```php
User::query()
    ->where('status', UserStatus::PENDING)
    ->chunkById(
        size: 1000,
        callback: function (Chunk $users): void {
            // ...
        },
    );
```

El objetivo no será proporcionar navegación para una interfaz de usuario.

El objetivo será:

```text
Large Logical Dataset
        ↓
Bounded Read
        ↓
Chunk
        ↓
Processing
        ↓
Release Resources
        ↓
Next Chunk
```

La regla central será:

> **Un Chunk en VoltStack es una unidad acotada de lectura y procesamiento dentro del recorrido de un dataset; no representa por sí mismo una página de navegación, una transacción, un batch de escritura, un cursor de base de datos ni una garantía de snapshot.**

Formalmente:

```text
ChunkProcessing
=
TraversalStrategy
+
BoundedFetch
+
ProcessingBoundary
+
ProgressState
+
ResourcePolicy
+
FailurePolicy
```

y nunca:

```text
Chunk
=
Transaction
```

---

# 2. Posición dentro del Bloque 19

```text
199 Pagination
      ↓
navigation by numbered windows

200 Cursor Pagination
      ↓
navigation by logical boundaries

201 Chunk Processing
      ↓
bounded dataset processing

202 Lazy Collection
      ↓
deferred consumption

203 Bulk Insert
204 Bulk Update
205 Bulk Delete
      ↓
large mutations

206 Import
207 Export

208 Large Dataset Processing
```

Chunk Processing será una pieza fundamental para los documentos posteriores.

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
Chunk Processing
≠
Pagination
≠
Cursor Pagination
≠
Streaming
≠
Lazy Collection
≠
Batch Persistence
≠
Bulk Mutation
≠
Transaction
≠
Job
```

---

# 4. Chunk vs Pagination

Pagination responde:

> ¿Qué ventana del dataset debe mostrarse?

Chunk Processing responde:

> ¿Cómo recorro el dataset completo manteniendo acotado el uso de recursos?

Por tanto:

```text
Pagination
=
navigation contract

Chunk Processing
=
processing traversal contract
```

---

# 5. Chunk vs Cursor Pagination

Ambos pueden utilizar keyset traversal.

Pero:

```text
Cursor Pagination
→ external navigation token

Chunk Processing
→ internal processing continuation
```

Chunk Processing puede reutilizar infraestructura de Cursor Pagination sin exponer necesariamente cursores al consumidor.

---

# 6. Chunk vs Streaming

Streaming puede mantener:

```text
Connection
+
Result Cursor
```

abiertos durante largos periodos.

Chunking ejecuta lecturas acotadas:

```text
Query
→ 500 rows
→ process
→ release
→ next query
```

---

# 7. Chunk vs Lazy Collection

Una Lazy Collection representa una interfaz de consumo diferido.

Chunk Processing representa el mecanismo de traversal por bloques.

El documento 202 podrá construir:

```text
LazyCollection
        ↓
ChunkTraversal
        ↓
Database
```

---

# 8. Chunk vs Batch Write

Procesar:

```text
500 entities
```

no implica ejecutar:

```text
one bulk UPDATE
```

Chunk describe la unidad de lectura/procesamiento.

Bulk operations serán definidas en 203–205.

---

# 9. Chunk vs Transaction

Regla crítica:

```text
ChunkBoundary
≠
TransactionBoundary
```

Aunque una policy pueda decidir ejecutar cada chunk dentro de una transacción.

---

# 10. Objetivos

El sistema deberá soportar:

1. procesamiento incremental;
2. datasets grandes;
3. bounded memory;
4. offset chunking;
5. keyset chunking;
6. chunk-by-ID;
7. ordered chunking;
8. composite keys;
9. deterministic traversal;
10. mutation-aware traversal;
11. resumability;
12. checkpoints;
13. cancellation;
14. retry policies;
15. transaction policies;
16. ORM;
17. Query Builder;
18. projections;
19. read/write routing;
20. replicas;
21. sharding;
22. multitenancy;
23. resource governance;
24. persistent runtimes;
25. telemetry;
26. diagnostics;
27. extensibilidad.

---

# 11. Arquitectura general

```text
Developer API
      │
      ▼
ChunkRequest
      │
      ▼
ChunkTraversalPlanner
      │
      ├── Query Analysis
      ├── Ordering
      ├── Traversal Strategy
      ├── Mutation Safety
      ├── Transaction Policy
      ├── Distribution
      └── Resource Policy
      │
      ▼
ChunkTraversalPlan
      │
      ▼
ChunkRunner
      │
      ├── Fetch Chunk
      │       ↓
      │   Query Engine
      │       ↓
      │   Execution
      │       ↓
      │   Hydration
      │
      ├── Process Chunk
      │
      ├── Update Progress
      │
      ├── Checkpoint
      │
      └── Continue
              ↓
       ChunkProcessingResult
```

---

# 12. API básica

```php
$query->chunk(
    size: 500,
    callback: function (Chunk $chunk): void {
        // ...
    },
);
```

---

# 13. API Model

```php
User::query()
    ->where('active', true)
    ->chunk(500, function (Chunk $users): void {
        foreach ($users as $user) {
            // ...
        }
    });
```

---

# 14. Repository API

```php
$repository
    ->query()
    ->where(...)
    ->chunk(500, $processor);
```

Ambas APIs deberán converger en:

```text
Chunk Processing Engine
```

---

# 15. No segundo ORM

Nunca:

```text
ModelChunker
→ custom SQL

RepositoryChunker
→ different SQL
```

Ambos deberán usar:

```text
Query Engine
→ Compiler
→ Executor
→ Hydration
```

---

# 16. ChunkRequest

Propuesta:

```php
final readonly class ChunkRequest
{
    public function __construct(
        public int $size,
        public ChunkTraversalStrategy $strategy,
        public ChunkTransactionPolicy $transactionPolicy,
        public ChunkFailurePolicy $failurePolicy,
        public ?ChunkCheckpoint $resumeFrom = null,
    ) {}
}
```

---

# 17. Chunk size

Deberá cumplirse:

```text
size > 0
```

---

# 18. Maximum chunk size

Resource Governance podrá establecer:

```text
defaultChunkSize = 500
maxChunkSize = 10_000
```

---

# 19. Chunk size ≠ optimal size

No existe un tamaño universal.

Depende de:

```text
row/entity size
hydration cost
relationship graph
memory budget
query latency
network latency
processing cost
transaction duration
```

---

# 20. Adaptive sizing

Una futura extensión podrá ajustar tamaño dinámicamente.

Pero la V1 deberá favorecer comportamiento determinista.

---

# 21. Chunk

Contrato conceptual:

```php
/**
 * @template T
 */
final readonly class Chunk
{
    public function __construct(
        public array $items,
        public ChunkIndex $index,
        public ChunkMetadata $metadata,
    ) {}
}
```

---

# 22. Chunk index

Podrá representar:

```text
0
1
2
...
```

o:

```text
1
2
3
...
```

La convención interna deberá ser consistente.

Recomendación:

```text
ChunkIndex = zero-based
```

porque no representa una página de usuario.

---

# 23. Chunk metadata

```php
final readonly class ChunkMetadata
{
    public function __construct(
        public int $requestedSize,
        public int $actualSize,
        public bool $isFirst,
        public bool $isLast,
        public ChunkTraversalMetadata $traversal,
    ) {}
}
```

---

# 24. Empty dataset

Para dataset vacío:

```text
callback invocation count = 0
```

por defecto.

No deberá crearse un chunk vacío artificial salvo API explícita.

---

# 25. Final chunk

Si:

```text
requested = 500
actual = 173
```

normalmente:

```text
isLast = true
```

si la estrategia tiene evidencia suficiente.

---

# 26. Lookahead

También puede utilizarse:

```text
fetch size + 1
```

para probar continuidad.

---

# 27. Traversal strategies

```php
enum ChunkTraversalStrategy
{
    case OFFSET;
    case KEYSET;
    case IDENTIFIER;
    case CUSTOM;
}
```

---

# 28. OFFSET

Conceptualmente:

```text
Chunk 0:
LIMIT 500 OFFSET 0

Chunk 1:
LIMIT 500 OFFSET 500

Chunk 2:
LIMIT 500 OFFSET 1000
```

---

# 29. Ventaja OFFSET

Es simple y permite trabajar con queries donde keyset traversal no es fácilmente derivable.

---

# 30. Problema OFFSET

Sobre datasets mutables puede:

```text
skip rows
duplicate rows
increase query cost
```

---

# 31. Ejemplo de pérdida

Dataset:

```text
1 2 3 4 5 6
```

Chunk size:

```text
3
```

Primer chunk:

```text
1 2 3
```

Durante procesamiento se elimina `2`.

Siguiente:

```text
OFFSET 3
```

sobre:

```text
1 3 4 5 6
```

puede comenzar en:

```text
5
```

omitiendo `4`.

---

# 32. OFFSET no recomendado para mutaciones

Especialmente cuando el callback modifica el mismo conjunto que está recorriendo.

---

# 33. KEYSET

Conceptualmente:

```text
ORDER BY id ASC

Chunk 1:
id > MIN
LIMIT 500

last id = 500

Chunk 2:
id > 500
LIMIT 500
```

---

# 34. Keyset preferred

Para datasets grandes y mutables, keyset será normalmente la estrategia recomendada.

---

# 35. Reutilización de Cursor Pagination

`200_DATABASE_CURSOR_PAGINATION_SYSTEM.md` ya define:

```text
stable ordering
compound boundaries
lexicographic predicates
NULL semantics
tie-breakers
```

Chunk Processing deberá reutilizar esos conceptos.

No duplicarlos.

---

# 36. Internal continuation

La diferencia principal será:

```text
Cursor Pagination:
ContinuationToken may cross requests

Chunk Processing:
ContinuationState may remain internal
```

---

# 37. IDENTIFIER

Ergonomía:

```php
User::query()
    ->chunkById(500, $callback);
```

---

# 38. chunkById()

Conceptualmente equivale a:

```text
identifier-aware keyset traversal
```

No a una implementación SQL separada.

---

# 39. Default identifier

ORM metadata podrá resolver:

```text
Entity Identifier
```

pero no deberá asumir siempre una columna llamada:

```text
id
```

---

# 40. Composite IDs

`chunkById()` simple puede no ser suficiente.

Deberá existir una variante basada en ordered keyset para identificadores compuestos.

---

# 41. Identifier direction

Podrá soportarse:

```php
$query->chunkById(
    size: 500,
    callback: $processor,
    direction: SortDirection::ASC,
);
```

---

# 42. Ordered chunking

Ejemplo:

```php
$query
    ->orderBy('created_at')
    ->orderBy('id')
    ->chunkOrdered(500, $callback);
```

---

# 43. Deterministic ordering

Keyset chunking deberá requerir:

```text
stable total/effective ordering
```

según las reglas del documento 200.

---

# 44. Tie-breaker

Si:

```text
ORDER BY created_at
```

no es único, podrá añadirse:

```text
id
```

si metadata lo permite.

---

# 45. No heuristic guessing

No asumir:

```text
column named id
=
safe tie-breaker
```

sin metadata.

---

# 46. Mutation safety

Chunk Processing deberá analizar qué sucede cuando el callback modifica registros.

---

# 47. Mutation classes

Podemos distinguir:

```text
NON_MUTATING
MUTATES_NON_ORDERING_FIELDS
MUTATES_ORDERING_FIELDS
DELETES_ROWS
INSERTS_ROWS
UNKNOWN
```

---

# 48. Non-ordering mutation

Si se recorre:

```text
ORDER BY id
```

y se cambia:

```text
processed = true
```

sin cambiar `id`, keyset traversal puede continuar correctamente.

---

# 49. Ordering mutation

Si se recorre:

```text
ORDER BY score
```

y el callback cambia `score`, una fila puede cruzar la frontera.

---

# 50. Resultado

Puede:

```text
reappear
```

o:

```text
be skipped
```

dependiendo de la mutación.

---

# 51. MutationSafeTraversalPolicy

Propuesta:

```php
enum ChunkMutationPolicy
{
    case ASSUME_READ_ONLY;
    case ALLOW_NON_ORDERING_MUTATIONS;
    case WARN_ON_ORDERING_MUTATION;
    case REQUIRE_IMMUTABLE_ORDERING;
    case CUSTOM;
}
```

---

# 52. Static knowledge limitations

VoltStack no siempre podrá saber qué modificará un callback PHP.

Por tanto:

```text
UNKNOWN
≠
READ_ONLY
```

---

# 53. Explicit declaration

APIs avanzadas podrán permitir:

```php
$query->chunk(
    size: 500,
    callback: $processor,
    mutationPolicy: ChunkMutationPolicy::ALLOW_NON_ORDERING_MUTATIONS,
);
```

---

# 54. Filtering on mutable state

Caso muy común:

```php
$query
    ->where('processed', false)
    ->orderBy('id')
    ->chunkById(500, function ($rows) {
        // set processed = true
    });
```

Esto puede ser seguro respecto al boundary `id`, aunque las filas salgan del predicate después de procesarse.

---

# 55. Offset sería peligroso

Con OFFSET:

```text
processed rows disappear from result
→ offsets shift
→ rows skipped
```

Keyset evita ese problema si la key de recorrido no cambia.

---

# 56. Inserts concurrentes

Con:

```text
id > lastId
```

nuevos IDs mayores pueden aparecer durante el proceso.

---

# 57. Traversal horizon

Debe definirse si el proceso pretende:

```text
PROCESS_UNTIL_EXHAUSTED
```

o:

```text
PROCESS_INITIAL_SNAPSHOT_RANGE
```

---

# 58. PROCESS_UNTIL_EXHAUSTED

Puede incluir registros insertados durante el proceso si caen después de la frontera.

---

# 59. INITIAL_RANGE

Puede capturarse una frontera superior inicial.

Ejemplo:

```text
startMaxId = 1,000,000
```

y recorrer:

```text
lastId < id <= startMaxId
```

---

# 60. HorizonPolicy

```php
enum ChunkHorizonPolicy
{
    case LIVE;
    case CAPTURE_UPPER_BOUND;
    case SNAPSHOT;
    case CUSTOM;
}
```

---

# 61. SNAPSHOT

Requiere infraestructura transaccional/DB compatible.

No será asumido por default.

---

# 62. Long transaction danger

Procesar millones de filas dentro de una única transacción snapshot puede:

```text
retain MVCC versions
increase locks
increase storage pressure
block maintenance
increase failure cost
```

---

# 63. Default recomendado

Para workloads normales:

```text
KEYSET
+
LIVE or CAPTURE_UPPER_BOUND
+
bounded transaction scopes
```

según intención.

---

# 64. Transaction policies

```php
enum ChunkTransactionPolicy
{
    case NONE;
    case USE_EXISTING;
    case PER_CHUNK;
    case WHOLE_TRAVERSAL;
    case CUSTOM;
}
```

---

# 65. NONE

Chunk Processing no crea transacciones.

---

# 66. USE_EXISTING

Si existe una transacción externa, la utiliza.

Nunca la committeará.

---

# 67. PER_CHUNK

Cada chunk puede procesarse dentro de una nueva transacción propiedad del ChunkRunner.

---

# 68. WHOLE_TRAVERSAL

Todo el recorrido pertenece a una transacción.

Debe utilizarse con extrema cautela para datasets grandes.

---

# 69. Ownership rule

> **El componente que abre una transacción es responsable de cerrarla; Chunk Processing nunca hará commit de una transacción propiedad del caller.**

---

# 70. Flush ≠ commit

En ORM:

```text
EntityManager::flush()
≠
TransactionManager::commit()
```

La regla continúa aplicándose dentro de chunks.

---

# 71. Chunk callback + ORM

Ejemplo:

```php
User::query()
    ->chunkById(500, function (Chunk $users) use ($entityManager): void {
        foreach ($users as $user) {
            $user->markProcessed();
        }

        $entityManager->flush();
        $entityManager->clear();
    });
```

---

# 72. clear()

`EntityManager::clear()` puede ser útil para liberar managed entities.

Pero ChunkRunner no deberá ejecutarlo silenciosamente por default.

---

# 73. Razón

El EntityManager puede contener objetos no relacionados con el chunk.

---

# 74. Scoped clear

Una futura API podrá permitir:

```text
clear only chunk-managed entities
```

si el ORM puede garantizarlo correctamente.

---

# 75. Memory problem

Sin limpieza:

```text
Chunk 1 → 500 managed
Chunk 2 → 1000 managed
Chunk 3 → 1500 managed
...
```

IdentityMap puede crecer indefinidamente.

---

# 76. ORMChunkMemoryPolicy

Podrá definir:

```text
KEEP_MANAGED
DETACH_PROCESSED
CLEAR_ENTITY_TYPE
CALLER_MANAGED
```

si las garantías ORM correspondientes existen.

---

# 77. Default seguro

Para el motor genérico:

```text
CALLER_MANAGED
```

o una estrategia explícitamente solicitada.

Nunca limpiar estado ajeno silenciosamente.

---

# 78. Projection processing

Para procesos que no necesitan entidades:

```php
$query
    ->select('id', 'email')
    ->chunkById(1000, $callback);
```

puede reducir significativamente memoria.

---

# 79. Hydration strategy

Chunk Processing deberá respetar:

```text
135–141 Hydration Architecture
```

y podrá solicitar result shapes ligeros.

---

# 80. Chunk ≠ Hydrator

El ChunkRunner no deberá construir entidades directamente.

---

# 81. Progress state

Cada recorrido tendrá:

```text
ChunkProgress
```

---

# 82. ChunkProgress

Propuesta:

```php
final readonly class ChunkProgress
{
    public function __construct(
        public int $chunksCompleted,
        public int $itemsProcessed,
        public ?ChunkContinuation $continuation,
    ) {}
}
```

---

# 83. Progress ≠ checkpoint

Progress puede existir únicamente en memoria.

Checkpoint implica estado persistido/reanudable.

---

# 84. Checkpoints

VoltStack deberá soportar checkpoints opcionales.

```text
Chunk 1
→ success
→ checkpoint

Chunk 2
→ success
→ checkpoint

Chunk 3
→ failure

restart
→ resume after Chunk 2
```

---

# 85. Checkpoint content

Podrá contener:

```text
version
query fingerprint
ordering fingerprint
continuation boundary
domain binding
tenant/shard binding
horizon
completed chunk count
processed item count
metadata generation
```

---

# 86. Checkpoint ≠ pagination cursor

Puede reutilizar el mismo boundary model, pero tiene un contrato diferente.

```text
Pagination Cursor
→ navigation

Chunk Checkpoint
→ processing recovery
```

---

# 87. Checkpoint store

Interfaz:

```php
interface ChunkCheckpointStore
{
    public function load(
        ChunkProcessId $process
    ): ?ChunkCheckpoint;

    public function save(
        ChunkProcessId $process,
        ChunkCheckpoint $checkpoint
    ): void;

    public function delete(
        ChunkProcessId $process
    ): void;
}
```

---

# 88. Optional dependency

Chunk Processing core no dependerá obligatoriamente de:

```text
Redis
Database table
filesystem
```

para checkpoints.

---

# 89. Checkpoint atomicity

Problema:

```text
process side effect
↓
crash
↓
checkpoint not saved
```

Al reanudar, el item puede procesarse nuevamente.

---

# 90. Exactly once

VoltStack no deberá prometer:

```text
exactly-once processing
```

solo porque existen checkpoints.

---

# 91. Default processing semantics

La semántica real normalmente será:

```text
at-least-once
```

cuando existe recovery mediante checkpoints.

---

# 92. Idempotency

Callbacks reintentables deberían ser idempotentes o usar mecanismos transaccionales apropiados.

---

# 93. Checkpoint inside transaction

Cuando checkpoint y mutación viven en el mismo DB/domain, una integración especializada podría persistir ambos en la misma transacción.

---

# 94. Pero

```text
Checkpoint transaction
≠
arbitrary external side-effect transaction
```

No existe atomicidad automática con:

```text
email
HTTP API
filesystem
external queue
```

---

# 95. External side effects

Preferir:

```text
Outbox
Jobs
idempotency keys
```

cuando se necesite recovery robusto.

---

# 96. ChunkProcessId

Un proceso reanudable necesitará identidad estable.

Ejemplo:

```text
users.rebuild-search-index.v3
```

No usar automáticamente IDs aleatorios si se espera reanudación posterior.

---

# 97. Query binding

Checkpoint deberá estar ligado al query que originó el proceso.

No deberá reanudarse:

```text
WHERE status = pending
```

con checkpoint de:

```text
WHERE status = active
```

---

# 98. Checkpoint validation

Debe verificar:

```text
query
ordering
domain
tenant
shard
strategy
version
horizon
```

---

# 99. Failure policies

```php
enum ChunkFailurePolicy
{
    case STOP;
    case RETRY_CHUNK;
    case SKIP_CHUNK;
    case SKIP_ITEM;
    case CUSTOM;
}
```

---

# 100. Default

```text
STOP
```

es el comportamiento más seguro.

---

# 101. RETRY_CHUNK

Solo deberá utilizarse cuando el procesamiento sea:

```text
replay-safe
```

o exista una estrategia de idempotencia.

---

# 102. Retry ≠ rollback

Si el callback produjo un side effect externo antes de fallar:

```text
DB rollback
```

no lo revierte.

---

# 103. SKIP_CHUNK

Es una policy peligrosa porque puede dejar cientos/miles de elementos sin procesar.

Debe quedar registrado explícitamente.

---

# 104. SKIP_ITEM

Solo es posible si el runner procesa items individualmente o el callback expone fallos por item.

No debe inventarse sobre un callback opaco.

---

# 105. Failure granularity

Distinguir:

```text
FETCH_FAILURE
HYDRATION_FAILURE
CALLBACK_FAILURE
FLUSH_FAILURE
TRANSACTION_FAILURE
CHECKPOINT_FAILURE
CANCELLATION
UNKNOWN_OUTCOME
```

---

# 106. UNKNOWN outcome

Especialmente:

```text
COMMIT sent
↓
connection lost
```

debe conservar:

```text
UNKNOWN
```

---

# 107. UNKNOWN ≠ retry-safe

Nunca reintentar ciegamente un chunk después de commit desconocido.

---

# 108. Chunk execution states

Propuesta:

```text
PLANNED
FETCHING
PROCESSING
COMMITTING
CHECKPOINTING
COMPLETED
FAILED
CANCELLED
UNKNOWN
```

---

# 109. Cancellation

El proceso deberá aceptar:

```text
CancellationToken
```

---

# 110. Cancellation boundaries

Puede comprobarse:

```text
before fetch
after fetch
between items
before transaction commit
after chunk completion
```

según granularidad.

---

# 111. No cancellation falsa

Una vez enviado un commit:

```text
cancel requested
```

no significa que el commit no ocurrió.

---

# 112. Graceful stop

El callback podrá solicitar detener el recorrido:

```php
return ChunkControl::STOP;
```

---

# 113. ChunkControl

```php
enum ChunkControl
{
    case CONTINUE;
    case STOP;
}
```

---

# 114. STOP ≠ failure

Debe distinguirse:

```text
COMPLETED_EARLY
```

de:

```text
FAILED
```

---

# 115. Callback contract

Propuesta:

```php
/**
 * @template T
 */
interface ChunkProcessor
{
    /**
     * @param Chunk<T> $chunk
     */
    public function process(
        Chunk $chunk,
        ChunkProcessingContext $context,
    ): ChunkControl;
}
```

---

# 116. Closures

La API ergonomic podrá adaptar closures a `ChunkProcessor`.

---

# 117. Chunk context

```php
final readonly class ChunkProcessingContext
{
    public function __construct(
        public ChunkProcessId $processId,
        public ChunkProgress $progress,
        public DatabaseContext $database,
        public CancellationToken $cancellation,
        public ResourceBudget $budget,
    ) {}
}
```

---

# 118. Resource budget

Puede controlar:

```text
maximum rows
maximum chunks
maximum duration
memory budget
query timeout
processing deadline
```

---

# 119. Max rows

Ejemplo:

```php
$query->chunk(
    size: 500,
    maxItems: 10_000,
    callback: $processor,
);
```

debe detenerse tras el límite lógico configurado.

---

# 120. Query timeout

Cada fetch deberá respetar:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM
```

---

# 121. Overall deadline

Además podrá existir:

```text
TraversalDeadline
```

independiente del timeout individual de query.

---

# 122. Read routing

Chunks de lectura podrán utilizar replicas si la consistency policy lo permite.

---

# 123. Replica switching

Cambiar de replica entre chunks puede producir observaciones diferentes.

---

# 124. ReplicaConsistencyPolicy

Podrá establecer:

```text
ANY_ELIGIBLE
PIN_ENDPOINT
MINIMUM_POSITION
WRITER_ONLY
CUSTOM
```

---

# 125. PIN_ENDPOINT

Todo el recorrido intenta permanecer en el mismo endpoint.

No implica snapshot.

---

# 126. MINIMUM_POSITION

Cada siguiente replica deberá satisfacer al menos la posición observada requerida.

---

# 127. Writer-only

Recomendable cuando el callback escribe y necesita read-your-writes inmediato.

---

# 128. Read/write interaction

Si el mismo chunk:

```text
reads pending rows
→ marks them processed
```

la siguiente lectura debe respetar el estado requerido por el algoritmo.

---

# 129. Sticky routing

Se integra con:

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM
```

---

# 130. Chunk consistency

Perfiles conceptuales:

```php
enum ChunkConsistency
{
    case BEST_EFFORT;
    case MONOTONIC;
    case READ_YOUR_WRITES;
    case SNAPSHOT;
    case CUSTOM;
}
```

---

# 131. BEST_EFFORT

Adecuado para ciertos procesos analíticos/no críticos.

---

# 132. MONOTONIC

Evita retroceder respecto a evidencia de versión observada cuando la infraestructura lo permita.

---

# 133. READ_YOUR_WRITES

Importante para:

```text
read
→ mutate
→ next chunk
```

---

# 134. SNAPSHOT

Puede requerir transacción larga o mecanismo de snapshot específico.

No será default.

---

# 135. Sharding

Chunk Processing deberá integrar:

```text
183 Distributed Database
184 Sharding
185 Partition Routing
```

---

# 136. Single-shard

```text
Query
→ Shard A
→ Chunk traversal
```

---

# 137. Multi-shard

Hay dos estrategias principales:

```text
GLOBAL_ORDERED
PER_SHARD
```

---

# 138. GLOBAL_ORDERED

Se requiere:

```text
global ordering
+
merge
+
global continuation
```

similar a Cursor Pagination distribuida.

---

# 139. PER_SHARD

Para tareas de procesamiento, frecuentemente será mejor:

```text
Shard A → independent chunk worker
Shard B → independent chunk worker
Shard C → independent chunk worker
```

---

# 140. Per-shard parallelism

Permite paralelizar sin imponer un orden global innecesario.

---

# 141. Semantic requirement

El planner deberá preguntar:

```text
Does processing require global order?
```

Si no:

```text
PER_SHARD
```

puede ser preferible.

---

# 142. Shard checkpoint

Cada shard podrá mantener:

```text
ShardContinuation
```

independiente.

---

# 143. Global checkpoint

Conceptualmente:

```text
ChunkCheckpoint
└── shardStates
    ├── A → boundary
    ├── B → boundary
    └── C → boundary
```

---

# 144. Bounded state

Para miles de shards, checkpoint state deberá diseñarse cuidadosamente.

Puede requerir almacenamiento stateful en lugar de token monolítico.

---

# 145. Shard movement

Resharding puede invalidar o requerir transformar checkpoints.

No asumir compatibilidad.

---

# 146. ShardMapGeneration

Podrá formar parte de:

```text
ChunkCheckpointBinding
```

---

# 147. Cross-shard transactions

Chunk Processing no fingirá:

```text
one global ACID transaction
```

sobre shards independientes.

---

# 148. Parallel chunk processing

Podrá existir una extensión:

```text
ChunkParallelRunner
```

---

# 149. Pero

No toda query es paralelizable manteniendo semántica.

---

# 150. Parallelism strategies

```text
BY_SHARD
BY_PARTITION
BY_KEY_RANGE
CUSTOM
```

---

# 151. Parallelism ≠ concurrency safety

Procesar dos rangos simultáneamente puede producir:

```text
contention
deadlocks
duplicate work
ordering violations
```

si el workload no es independiente.

---

# 152. Range partitioning

Ejemplo:

```text
Worker 1:
id 1..100000

Worker 2:
id 100001..200000
```

solo es correcto si las fronteras son estables y disjuntas.

---

# 153. Work claiming

Para workers competitivos puede ser mejor usar:

```text
SELECT ... FOR UPDATE SKIP LOCKED
```

donde exista capacidad equivalente.

Pero esto será una estrategia especializada.

---

# 154. Work queue ≠ chunk traversal

No convertir automáticamente Chunk Processing en sistema de colas.

---

# 155. Platform capability

Funciones como:

```text
SKIP LOCKED
row value comparison
server cursors
snapshot export
```

deberán resolverse mediante capabilities.

---

# 156. Version ≠ capability

Se mantiene:

```text
DatabaseVersion
≠
FeatureCapability
```

---

# 157. MySQL / MariaDB

Serán plataformas distintas de primera clase.

No asumir capabilities idénticas.

---

# 158. PostgreSQL / SQLite

También deberán usar sus respectivos platform capability models.

---

# 159. Multitenancy

Tenant context deberá permanecer estable durante un traversal salvo operación administrativa explícita.

---

# 160. No tenant leakage

Nunca:

```text
Chunk 1 → Tenant A
Chunk 2 → Tenant B
```

por state leakage de worker.

---

# 161. Per-tenant processing

Una integración podrá realizar:

```text
Tenant A
  → chunks

Tenant B
  → chunks
```

como procesos separados.

---

# 162. Tenant checkpoint

El checkpoint deberá incluir/bind:

```text
TenantContext
```

cuando aplique.

---

# 163. Cross-tenant administrative processing

Debe ser explícito, autorizado y normalmente particionado por tenant.

---

# 164. Cache

Chunk Processing normalmente procesa datasets operacionales.

No deberá utilizar Result Cache automáticamente.

---

# 165. Razón

Un traversal largo sobre páginas cacheadas puede observar:

```text
mixed cache generations
```

y producir resultados inesperados.

---

# 166. CachePolicy

Default recomendado:

```text
ResultCache = BYPASS
```

para procesamiento de chunks, salvo solicitud explícita.

---

# 167. Metadata cache

Sí puede utilizarse normalmente porque almacena conocimiento estructural.

---

# 168. Entity cache

Debe evaluarse según consistency policy.

Para procesos masivos suele ser razonable bypass de L2 Entity Cache.

---

# 169. Cache invalidation

Si el callback modifica entidades, las invalidaciones deberán fluir por el sistema normal:

```text
UoW
→ semantic changes
→ transaction outcome
→ cache invalidation
```

ChunkRunner no ejecutará `FLUSHALL`.

---

# 170. Security

Chunk Processing deberá respetar:

```text
authorization scopes
tenant isolation
data access policy
sensitive data protection
```

---

# 171. Background processing

El hecho de ejecutarse en CLI/worker no elimina authorization/data-scope requirements.

---

# 172. Raw query escape hatch

Si se usa raw SQL para traversal, deberá declararse suficiente metadata de:

```text
ordering
continuation
routing
types
security
```

para soportar operaciones avanzadas.

---

# 173. Persistent runtime

En FrankenPHP:

```text
Request/Job A
→ ChunkProgress A

Request/Job B
→ ChunkProgress B
```

deben estar aislados.

---

# 174. Worker state

No almacenar:

```text
static $lastId
```

como estado de traversal.

---

# 175. RoadRunner/OpenSwoole

La misma regla aplica.

---

# 176. Coroutine isolation

Cada traversal concurrente deberá poseer:

```text
ChunkProcessingContext
ChunkProgress
TransactionContext
EntityManager scope
Continuation
```

independientes.

---

# 177. Long-lived workers

Los siguientes objetos podrán ser compartidos si son immutable/stateless:

```text
compiled metadata
chunk strategy registry
platform capabilities
query compiler infrastructure
```

---

# 178. No compartir

Nunca compartir entre traversals:

```text
EntityManager mutable state
IdentityMap
UnitOfWork
current chunk
continuation
checkpoint mutable state
transaction
tenant context
```

---

# 179. Planner

`ChunkTraversalPlanner` deberá:

```text
validate query
validate chunk size
resolve traversal strategy
resolve ordering
resolve continuation model
analyze mutation policy
resolve horizon
resolve consistency
resolve transaction policy
resolve routing/distribution
apply resource governance
produce immutable plan
```

---

# 180. Planner ≠ Executor

Siempre:

```text
ChunkTraversalPlanner
≠
ChunkRunner
≠
QueryExecutor
```

---

# 181. ChunkTraversalPlan

```php
final readonly class ChunkTraversalPlan
{
    public function __construct(
        public QueryModel $baseQuery,
        public ChunkTraversalStrategy $strategy,
        public ChunkOrderingPlan $ordering,
        public ChunkHorizon $horizon,
        public ChunkTransactionPlan $transaction,
        public ChunkConsistencyPlan $consistency,
        public ChunkDistributionPlan $distribution,
        public ChunkResourcePlan $resources,
    ) {}
}
```

---

# 182. Base query immutable

El runner no deberá mutar permanentemente el query original.

Cada fetch derivará:

```text
BaseQuery
+
ContinuationPredicate
+
ChunkLimit
```

---

# 183. Query derivation

```text
Q0 = BaseQuery

Q1 =
Q0
+ boundary B0
+ limit N

Q2 =
Q0
+ boundary B1
+ limit N
```

---

# 184. No predicate accumulation

Error a evitar:

```text
Q2 =
Q1
+ boundary B1
```

si eso deja acumuladas condiciones antiguas innecesarias.

Derivar siempre desde el modelo base apropiado.

---

# 185. Chunk runner

Responsabilidades:

```text
obtain continuation
derive fetch query
execute
hydrate
construct chunk
invoke processor
resolve transaction outcome
advance continuation
persist checkpoint
release scoped resources
repeat
```

---

# 186. Advance only after success

Regla crítica:

> **La continuación durable no deberá avanzar antes de que el procesamiento asociado haya alcanzado el outcome requerido por la policy.**

---

# 187. Ejemplo incorrecto

```text
fetch chunk
↓
save checkpoint "done"
↓
process chunk
↓
failure
```

provocaría pérdida de trabajo.

---

# 188. Orden correcto básico

```text
fetch
↓
process
↓
required transaction outcome
↓
checkpoint
↓
next
```

---

# 189. Checkpoint failure

Si:

```text
processing succeeded
checkpoint failed
```

el proceso puede repetir el chunk tras restart.

Por eso idempotency sigue siendo importante.

---

# 190. Checkpoint status

Puede distinguir:

```text
PENDING
COMMITTED
UNKNOWN
```

si existe protocolo más avanzado.

---

# 191. Process result

```php
final readonly class ChunkProcessingResult
{
    public function __construct(
        public ChunkProcessingStatus $status,
        public int $chunksProcessed,
        public int $itemsProcessed,
        public ?ChunkCheckpoint $lastCheckpoint,
        public ChunkProcessingMetrics $metrics,
    ) {}
}
```

---

# 192. Status

```php
enum ChunkProcessingStatus
{
    case COMPLETED;
    case COMPLETED_EARLY;
    case CANCELLED;
    case FAILED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 193. UNKNOWN preservation

Nunca:

```text
UNKNOWN
→
FAILED
```

si el sistema realmente desconoce el outcome de una transacción crítica.

---

# 194. Error hierarchy

```text
DatabaseException
└── ChunkProcessingException
    ├── InvalidChunkRequestException
    ├── InvalidChunkSizeException
    ├── ChunkPlanningException
    ├── ChunkOrderingException
    ├── ChunkTraversalException
    ├── ChunkMutationSafetyException
    ├── ChunkProcessingCallbackException
    ├── ChunkCheckpointException
    │   ├── ChunkCheckpointMismatchException
    │   ├── ChunkCheckpointCorruptedException
    │   └── ChunkCheckpointPersistenceException
    ├── ChunkRetryUnsafeException
    ├── ChunkTransactionException
    ├── ChunkDistributionException
    ├── ChunkResourceException
    └── ChunkConsistencyException
```

---

# 195. Directory structure

```text
src/Quantum/Database/Chunk/
│
├── Chunk.php
├── ChunkIndex.php
├── ChunkMetadata.php
├── ChunkControl.php
├── ChunkRequest.php
│
├── Traversal/
│   ├── ChunkTraversalStrategy.php
│   ├── ChunkTraversalPlanner.php
│   ├── ChunkTraversalPlan.php
│   ├── ChunkContinuation.php
│   ├── OffsetChunkTraversal.php
│   ├── KeysetChunkTraversal.php
│   ├── IdentifierChunkTraversal.php
│   └── CustomChunkTraversal.php
│
├── Ordering/
│   ├── ChunkOrderingPlan.php
│   └── ChunkOrderingResolver.php
│
├── Mutation/
│   ├── ChunkMutationPolicy.php
│   └── ChunkMutationSafetyAnalyzer.php
│
├── Horizon/
│   ├── ChunkHorizon.php
│   ├── ChunkHorizonPolicy.php
│   └── ChunkHorizonResolver.php
│
├── Processing/
│   ├── ChunkProcessor.php
│   ├── ChunkRunner.php
│   ├── ChunkProcessingContext.php
│   ├── ChunkProcessingResult.php
│   └── ChunkProcessingStatus.php
│
├── Progress/
│   └── ChunkProgress.php
│
├── Checkpoint/
│   ├── ChunkCheckpoint.php
│   ├── ChunkCheckpointStore.php
│   ├── ChunkCheckpointValidator.php
│   └── ChunkProcessId.php
│
├── Transaction/
│   ├── ChunkTransactionPolicy.php
│   └── ChunkTransactionPlan.php
│
├── Failure/
│   ├── ChunkFailurePolicy.php
│   └── ChunkRetryPolicy.php
│
├── Consistency/
│   ├── ChunkConsistency.php
│   └── ChunkConsistencyPlan.php
│
├── Distribution/
│   ├── ChunkDistributionPlan.php
│   ├── DistributedChunkPlanner.php
│   ├── ShardChunkContinuation.php
│   └── ChunkParallelRunner.php
│
├── Resource/
│   ├── ChunkResourcePolicy.php
│   └── ChunkResourcePlan.php
│
├── ORM/
│   └── ORMChunkMemoryPolicy.php
│
├── Diagnostics/
│   ├── ChunkInspector.php
│   └── ChunkExplainer.php
│
├── Telemetry/
│   └── ChunkTelemetry.php
│
└── Exception/
    └── ...
```

---

# 196. Telemetry

Eventos conceptuales:

```text
ChunkTraversalPlanned
ChunkTraversalStarted
ChunkFetchStarted
ChunkFetched
ChunkProcessingStarted
ChunkProcessed
ChunkTransactionCompleted
ChunkCheckpointSaved
ChunkRetryScheduled
ChunkTraversalStopped
ChunkTraversalCompleted
ChunkTraversalFailed
ChunkTraversalCancelled
ChunkTraversalOutcomeUnknown
```

---

# 197. Métricas

```text
db.chunk.traversals
db.chunk.duration
db.chunk.fetch.duration
db.chunk.process.duration
db.chunk.size
db.chunk.items
db.chunk.retries
db.chunk.failures
db.chunk.checkpoints
db.chunk.cancelled
db.chunk.memory
```

---

# 198. Bounded cardinality

No utilizar como metric labels:

```text
raw checkpoint
last ID
tenant ID
user ID
raw query
cursor values
```

---

# 199. Diagnostics

API conceptual:

```php
DB::chunks()->explain(
    User::query()
        ->where('processed', false)
        ->orderBy('id'),
    size: 1000,
);
```

---

# 200. Explain output

```text
CHUNK PROCESSING PLAN

Strategy:
    KEYSET / IDENTIFIER

Chunk Size:
    1000

Ordering:
    id ASC

Ordering Stability:
    STABLE

Mutation Policy:
    ALLOW_NON_ORDERING_MUTATIONS

Predicate:
    processed = false

Continuation:
    id > :last_id

Horizon:
    LIVE

Transaction:
    PER_CHUNK

Consistency:
    READ_YOUR_WRITES

Result Cache:
    BYPASS

Distribution:
    SINGLE SHARD

Checkpoint:
    ENABLED

Retry Safety:
    REQUIRES IDEMPOTENT PROCESSOR

Memory:
    bounded per chunk

Warnings:
    callback mutation cannot be statically proven
```

---

# 201. Testing matrix

| Área | Caso |
|---|---|
| Basic | empty dataset |
| Basic | one chunk |
| Basic | multiple chunks |
| Basic | partial final chunk |
| Size | invalid zero |
| Size | negative |
| Size | maximum exceeded |
| Offset | normal traversal |
| Offset | deletion during traversal |
| Keyset | integer ID |
| Keyset | UUID |
| Keyset | composite key |
| Ordering | ASC |
| Ordering | DESC |
| Ordering | duplicate values |
| Ordering | inferred tie-breaker |
| Mutation | non-order field |
| Mutation | ordering field |
| Mutation | delete |
| Mutation | insert |
| Horizon | live |
| Horizon | upper bound |
| Transaction | none |
| Transaction | existing |
| Transaction | per chunk |
| Transaction | whole traversal |
| Failure | callback |
| Failure | fetch |
| Failure | commit unknown |
| Retry | replay safe |
| Retry | unsafe |
| Checkpoint | save/resume |
| Checkpoint | mismatch |
| Checkpoint | corrupted |
| Cancellation | before fetch |
| Cancellation | during process |
| ORM | IdentityMap growth |
| ORM | explicit clear |
| Replica | endpoint switch |
| RYW | writer routing |
| Sharding | single shard |
| Sharding | per-shard |
| Sharding | global order |
| Tenant | isolation |
| Runtime | worker reuse |

---

# 202. Architectural invariants

## DB-CHUNK-001
Chunk Processing será distinto de Pagination.

## DB-CHUNK-002
Chunk Processing será distinto de Cursor Pagination.

## DB-CHUNK-003
Chunk Processing será distinto de Streaming.

## DB-CHUNK-004
Chunk Processing será distinto de Lazy Collection.

## DB-CHUNK-005
Chunk Processing será distinto de Bulk Mutation.

## DB-CHUNK-006
Chunk será distinto de Transaction.

## DB-CHUNK-007
Chunk será distinto de Job.

## DB-CHUNK-008
Chunk será una unidad acotada.

## DB-CHUNK-009
Dataset completo no será materializado por default.

## DB-CHUNK-010
Chunk size deberá ser positiva.

## DB-CHUNK-011
Chunk size tendrá resource limits.

## DB-CHUNK-012
Chunk size no se asumirá universalmente óptima.

## DB-CHUNK-013
Empty dataset no producirá callback vacío por default.

## DB-CHUNK-014
Final chunk podrá ser menor al tamaño solicitado.

## DB-CHUNK-015
OFFSET será una estrategia soportada.

## DB-CHUNK-016
KEYSET será una estrategia soportada.

## DB-CHUNK-017
IDENTIFIER será una especialización de keyset.

## DB-CHUNK-018
chunkById no creará un segundo query engine.

## DB-CHUNK-019
Entity identifier será resuelto mediante metadata.

## DB-CHUNK-020
No se asumirá siempre columna `id`.

## DB-CHUNK-021
Composite key traversal será soportable.

## DB-CHUNK-022
Keyset reutilizará Cursor Pagination semantics.

## DB-CHUNK-023
Chunk continuation no requerirá token externo.

## DB-CHUNK-024
OFFSET podrá sufrir drift.

## DB-CHUNK-025
OFFSET mutation hazards serán documentados.

## DB-CHUNK-026
Keyset será preferido para grandes datasets mutables.

## DB-CHUNK-027
Keyset requerirá ordering suficiente.

## DB-CHUNK-028
Tie-breakers serán metadata-driven.

## DB-CHUNK-029
No se inferirá tie-breaker solo por nombre.

## DB-CHUNK-030
Mutation of ordering fields será tratada explícitamente.

## DB-CHUNK-031
UNKNOWN callback mutation no será READ_ONLY.

## DB-CHUNK-032
Non-ordering mutation podrá ser compatible con keyset.

## DB-CHUNK-033
Filtering mutable state podrá ser compatible con keyset.

## DB-CHUNK-034
Traversal horizon será explícito.

## DB-CHUNK-035
LIVE horizon podrá incluir nuevos registros.

## DB-CHUNK-036
CAPTURE_UPPER_BOUND podrá limitar el conjunto inicial.

## DB-CHUNK-037
SNAPSHOT no será asumido.

## DB-CHUNK-038
Long snapshot transaction tendrá riesgos explícitos.

## DB-CHUNK-039
Chunk boundary no implicará transaction boundary.

## DB-CHUNK-040
Transaction policy será explícita.

## DB-CHUNK-041
NONE no abrirá transaction.

## DB-CHUNK-042
USE_EXISTING no committeará transaction externa.

## DB-CHUNK-043
PER_CHUNK solo cerrará transacciones que ChunkRunner haya abierto.

## DB-CHUNK-044
WHOLE_TRAVERSAL será opt-in.

## DB-CHUNK-045
Flush será distinto de commit.

## DB-CHUNK-046
ChunkRunner no hará EntityManager::clear silenciosamente.

## DB-CHUNK-047
ChunkRunner no borrará IdentityMap ajeno.

## DB-CHUNK-048
ORM memory management será explícito.

## DB-CHUNK-049
Projection processing será soportado.

## DB-CHUNK-050
ChunkRunner no será Hydrator.

## DB-CHUNK-051
ChunkProgress será distinto de Checkpoint.

## DB-CHUNK-052
Checkpoint será opcional.

## DB-CHUNK-053
Checkpoint podrá permitir resumability.

## DB-CHUNK-054
Checkpoint no garantizará exactly-once.

## DB-CHUNK-055
Recovery podrá producir at-least-once.

## DB-CHUNK-056
Idempotency seguirá siendo responsabilidad relevante.

## DB-CHUNK-057
Checkpoint será distinto de Pagination Cursor.

## DB-CHUNK-058
Checkpoint podrá reutilizar boundary infrastructure.

## DB-CHUNK-059
Checkpoint store será provider-agnostic.

## DB-CHUNK-060
Core no requerirá Redis.

## DB-CHUNK-061
Core no requerirá filesystem checkpoint.

## DB-CHUNK-062
Checkpoint advance ocurrirá después del outcome requerido.

## DB-CHUNK-063
Checkpoint no avanzará antes del procesamiento exitoso.

## DB-CHUNK-064
Checkpoint failure podrá causar replay.

## DB-CHUNK-065
External side effects no serán revertidos por DB rollback.

## DB-CHUNK-066
Outbox podrá integrarse para side effects.

## DB-CHUNK-067
ChunkProcessId será estable cuando se requiera resume.

## DB-CHUNK-068
Checkpoint estará ligado al query.

## DB-CHUNK-069
Checkpoint estará ligado al ordering.

## DB-CHUNK-070
Checkpoint estará ligado al persistence domain.

## DB-CHUNK-071
Tenant/shard binding será preservado.

## DB-CHUNK-072
Failure policy será explícita.

## DB-CHUNK-073
STOP será default seguro.

## DB-CHUNK-074
RETRY_CHUNK requerirá replay safety.

## DB-CHUNK-075
UNKNOWN transaction outcome no será reintentado ciegamente.

## DB-CHUNK-076
Rollback no implicará external side-effect rollback.

## DB-CHUNK-077
SKIP_CHUNK será explícito.

## DB-CHUNK-078
SKIP_ITEM requerirá granularidad suficiente.

## DB-CHUNK-079
Fetch failure será distinguible de callback failure.

## DB-CHUNK-080
Commit unknown permanecerá UNKNOWN.

## DB-CHUNK-081
Cancellation será soportada.

## DB-CHUNK-082
Cancellation no cambiará un commit ya enviado a "cancelled".

## DB-CHUNK-083
Graceful STOP será distinto de failure.

## DB-CHUNK-084
Callback closures podrán adaptarse a contrato tipado.

## DB-CHUNK-085
Resource budget será soportado.

## DB-CHUNK-086
Maximum rows podrá limitar traversal.

## DB-CHUNK-087
Query timeout será respetado.

## DB-CHUNK-088
Overall traversal deadline podrá ser independiente.

## DB-CHUNK-089
Replica routing respetará consistency policy.

## DB-CHUNK-090
Replica switching no implicará snapshot.

## DB-CHUNK-091
PIN_ENDPOINT no implicará snapshot.

## DB-CHUNK-092
READ_YOUR_WRITES será soportable.

## DB-CHUNK-093
Sticky routing será respetado.

## DB-CHUNK-094
Snapshot consistency será opt-in.

## DB-CHUNK-095
Single-shard traversal será soportado.

## DB-CHUNK-096
Multi-shard traversal será explícito.

## DB-CHUNK-097
Global ordered traversal será distinto de per-shard traversal.

## DB-CHUNK-098
Per-shard processing podrá paralelizarse.

## DB-CHUNK-099
Global order no será impuesto si no es necesario.

## DB-CHUNK-100
Shard checkpoints podrán ser independientes.

## DB-CHUNK-101
Distributed checkpoint state será bounded.

## DB-CHUNK-102
Resharding podrá afectar checkpoint compatibility.

## DB-CHUNK-103
ShardMapGeneration podrá participar en binding.

## DB-CHUNK-104
No se fingirá global ACID sobre shards.

## DB-CHUNK-105
Parallelism será explícito.

## DB-CHUNK-106
Parallelism no implicará concurrency safety.

## DB-CHUNK-107
Key ranges paralelos deberán ser disjuntos cuando esa sea la estrategia.

## DB-CHUNK-108
Work claiming será una estrategia especializada.

## DB-CHUNK-109
Chunk Processing no será Queue System.

## DB-CHUNK-110
Platform features se resolverán por capabilities.

## DB-CHUNK-111
Version será distinta de Capability.

## DB-CHUNK-112
MySQL será soportado.

## DB-CHUNK-113
MariaDB será soportado como plataforma propia.

## DB-CHUNK-114
PostgreSQL será soportado.

## DB-CHUNK-115
SQLite será soportado según capabilities.

## DB-CHUNK-116
Tenant context será estable durante traversal normal.

## DB-CHUNK-117
Worker reuse no filtrará tenant state.

## DB-CHUNK-118
Per-tenant traversal será aislado.

## DB-CHUNK-119
Cross-tenant processing será explícito y autorizado.

## DB-CHUNK-120
Result Cache será bypass recomendado para chunk processing.

## DB-CHUNK-121
Metadata Cache podrá utilizarse normalmente.

## DB-CHUNK-122
Entity Cache tendrá policy explícita.

## DB-CHUNK-123
ChunkRunner no ejecutará FLUSHALL.

## DB-CHUNK-124
Cache invalidation seguirá transaction outcome.

## DB-CHUNK-125
Authorization será respetada.

## DB-CHUNK-126
Background execution no omitirá security policy.

## DB-CHUNK-127
Raw query traversal requerirá metadata suficiente.

## DB-CHUNK-128
Persistent runtime state será scoped.

## DB-CHUNK-129
No se almacenará current continuation en static mutable state.

## DB-CHUNK-130
FrankenPHP worker reuse será seguro.

## DB-CHUNK-131
RoadRunner worker reuse será seguro.

## DB-CHUNK-132
OpenSwoole worker reuse será seguro.

## DB-CHUNK-133
Coroutine traversals estarán aislados.

## DB-CHUNK-134
Immutable compiled metadata podrá compartirse.

## DB-CHUNK-135
EntityManager mutable no se compartirá entre traversals.

## DB-CHUNK-136
IdentityMap no se compartirá entre traversals.

## DB-CHUNK-137
UnitOfWork no se compartirá entre traversals.

## DB-CHUNK-138
TransactionContext no se compartirá entre traversals.

## DB-CHUNK-139
Chunk Planner no ejecutará queries.

## DB-CHUNK-140
Chunk Plan será immutable.

## DB-CHUNK-141
Base Query no se mutará destructivamente.

## DB-CHUNK-142
Cada chunk query derivará del modelo base apropiado.

## DB-CHUNK-143
Continuation predicates no se acumularán incorrectamente.

## DB-CHUNK-144
ChunkRunner utilizará Query Engine.

## DB-CHUNK-145
ChunkRunner no generará SQL.

## DB-CHUNK-146
Compiler continuará siendo autoridad SQL.

## DB-CHUNK-147
Executor continuará siendo autoridad de ejecución.

## DB-CHUNK-148
Hydrator continuará siendo autoridad de hydration.

## DB-CHUNK-149
ORM continuará siendo autoridad de entities.

## DB-CHUNK-150
TransactionManager continuará siendo autoridad transaccional.

## DB-CHUNK-151
Chunk processing no creará un segundo persistence engine.

## DB-CHUNK-152
Durable continuation avanzará solo tras outcome requerido.

## DB-CHUNK-153
Processing result tendrá estado explícito.

## DB-CHUNK-154
COMPLETED_EARLY será distinto de COMPLETED.

## DB-CHUNK-155
PARTIAL será distinto de FAILED.

## DB-CHUNK-156
UNKNOWN permanecerá UNKNOWN.

## DB-CHUNK-157
Telemetry tendrá cardinalidad acotada.

## DB-CHUNK-158
Raw checkpoint no se registrará como metric label.

## DB-CHUNK-159
Sensitive IDs no se registrarán por defecto.

## DB-CHUNK-160
Chunk plans serán explainable.

## DB-CHUNK-161
Mutation warnings serán explainable.

## DB-CHUNK-162
Transaction policy será explainable.

## DB-CHUNK-163
Consistency policy será explainable.

## DB-CHUNK-164
Distribution plan será explainable.

## DB-CHUNK-165
Checkpoint compatibility será diagnosticable.

## DB-CHUNK-166
Retry safety será diagnosticable.

## DB-CHUNK-167
Correctness tendrá prioridad sobre throughput.

## DB-CHUNK-168
Chunking limitará memoria, pero no garantizará memoria constante si el callback retiene referencias.

## DB-CHUNK-169
ChunkRunner no podrá controlar referencias retenidas por código de aplicación.

## DB-CHUNK-170
Chunking no implicará automáticamente mejor rendimiento que streaming.

## DB-CHUNK-171
Traversal strategy dependerá del workload.

## DB-CHUNK-172
Offset será válido pero no default ideal para datasets grandes mutables.

## DB-CHUNK-173
Keyset será recomendado cuando exista ordering apropiado.

## DB-CHUNK-174
Snapshot semantics nunca serán inferidas de keyset.

## DB-CHUNK-175
Same checkpoint no implicará same database snapshot.

## DB-CHUNK-176
Same continuation no implicará same remaining dataset forever.

## DB-CHUNK-177
Chunk processing será deterministic solo dentro de las garantías declaradas.

## DB-CHUNK-178
External concurrency deberá formar parte del consistency model.

## DB-CHUNK-179
Database truth continuará perteneciendo a la base de datos.

## DB-CHUNK-180
Chunk state nunca será database truth.

---

# 203. Modelo formal

Sea:

```text
Q
=
logical query
```

y:

```text
N
=
chunk size
```

Sea también:

```text
B_i
=
continuation boundary after chunk i
```

Para keyset traversal:

```text
Q_0
=
Q
```

y:

```text
Q_i
=
Q
∧ After(B_(i-1))
LIMIT N
```

---

# 204. Chunk sequence

El resultado será:

```text
C_0, C_1, ..., C_k
```

donde:

```text
|C_i| <= N
```

---

# 205. Termination

Normalmente:

```text
|C_k| < N
```

puede indicar terminación.

O puede utilizarse lookahead:

```text
fetch N + 1
```

para evidencia explícita.

---

# 206. Continuation

Después de un chunk exitoso:

```text
B_i
=
Boundary(last(C_i))
```

para traversal forward keyset.

---

# 207. Safety property

Idealmente:

```text
C_i ∩ C_j = ∅
```

para:

```text
i ≠ j
```

pero esta propiedad depende de:

```text
stable ordering keys
dataset mutation
query predicates
consistency policy
```

No deberá prometerse universalmente.

---

# 208. Memory model

Si:

```text
M(item)
=
average hydrated memory per item
```

entonces la memoria aproximada del chunk será:

```text
M_chunk
≈
N × M(item)
+
ORM overhead
+
callback state
+
query/result overhead
```

---

# 209. Importante

Aunque `N` sea constante:

```text
TotalMemory
```

puede crecer si:

```text
IdentityMap retains entities
callback retains references
application caches objects
```

Por eso:

```text
Chunking
≠
Automatic constant memory
```

---

# 210. Recovery model

Con checkpoint durable `P_i`:

```text
Process(C_i)
→ RequiredOutcome(C_i)
→ Persist(P_i)
```

Reanudación:

```text
Resume(P_i)
→ derive B_i
→ fetch C_(i+1)
```

---

# 211. Exactly-once impossibility boundary

Si:

```text
Process(C_i)
```

incluye side effects no coordinados con:

```text
Persist(P_i)
```

entonces un crash entre ambos puede producir replay.

Por tanto:

```text
Checkpoint
+
Retry
≠
ExactlyOnce
```

---

# 212. Arquitectura final

```text
                     Developer API
                          │
                          ▼
                     ChunkRequest
                          │
                          ▼
                ChunkTraversalPlanner
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
        Ordering       Strategy       Horizon
            │             │             │
            ├─────────────┼─────────────┤
            ▼             ▼             ▼
       Consistency    Transaction   Distribution
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                 ChunkTraversalPlan
                          │
                          ▼
                     ChunkRunner
                          │
                 ┌────────┴────────┐
                 │                 │
                 ▼                 │
          Derive Chunk Query       │
                 │                 │
                 ▼                 │
             Query Engine          │
                 │                 │
                 ▼                 │
               Execute             │
                 │                 │
                 ▼                 │
              Hydrate              │
                 │                 │
                 ▼                 │
                Chunk              │
                 │                 │
                 ▼                 │
              Processor            │
                 │                 │
                 ▼                 │
        Transaction Outcome        │
                 │                 │
                 ▼                 │
             Checkpoint            │
                 │                 │
                 ▼                 │
       Advance Continuation ───────┘
                 │
                 ▼
        ChunkProcessingResult
```

---

# 213. Regla maestra final

> **Chunk Processing en VoltStack recorrerá datasets mediante lecturas acotadas, estrategias de continuación explícitas y límites claros de recursos, sin confundir la unidad de procesamiento con una página, una transacción, un batch de escritura o un snapshot.**

Siempre:

```text
Chunk
≠
Page
```

```text
Chunk
≠
Transaction
```

```text
Chunk
≠
Bulk Write
```

```text
Chunk
≠
Stream
```

```text
Checkpoint
≠
Exactly Once
```

```text
Keyset
≠
Snapshot
```

```text
Bounded Chunk Size
≠
Guaranteed Constant Memory
```

y:

```text
UNKNOWN Transaction Outcome
≠
Retry Safe
```

---

# 214. Resultado arquitectónico

Con esta arquitectura VoltStack podrá soportar:

```php
User::query()
    ->where('processed', false)
    ->chunkById(
        size: 1000,
        callback: function (Chunk $users): void {
            foreach ($users as $user) {
                $user->markProcessed();
            }
        },
    );
```

sobre:

```text
10 rows
10,000 rows
10,000,000 rows
```

sin que la API cambie conceptualmente, mientras el motor conserva:

```text
bounded reads
stable continuation
resource governance
transaction ownership
retry safety
checkpoint semantics
tenant isolation
shard awareness
persistent runtime safety
```

y reutiliza:

```text
Query Engine
Cursor/Keyset Infrastructure
Execution Engine
Hydration
ORM
Transaction System
Distribution
Telemetry
```

en lugar de crear implementaciones paralelas.

---

# 215. Relación con el siguiente sistema

Hasta ahora:

```text
199 Pagination
    ↓
finite navigable windows

200 Cursor Pagination
    ↓
logical navigation boundaries

201 Chunk Processing
    ↓
bounded processing traversal
```

El siguiente nivel permitirá escribir:

```php
foreach (
    User::query()
        ->where('active', true)
        ->lazy() as $user
) {
    // ...
}
```

sin que el desarrollador tenga que administrar manualmente los chunks.

Esto pertenece a:

```text
Lazy Collection
```

---

# 216. Siguiente documento

```text
202_DATABASE_LAZY_COLLECTION_SYSTEM.md
```

El siguiente documento definirá:

```text
Lazy Collection System
├── lazy database iterable
├── deferred execution
├── lazy query start
├── chunk-backed iteration
├── cursor-backed iteration
├── streaming-backed iteration
├── iterator lifecycle
├── single-pass semantics
├── replayability
├── transformation pipeline
├── map
├── filter
├── take
├── skip
├── flatMap
├── each
├── reduce
├── materialization boundaries
├── ORM hydration
├── IdentityMap pressure
├── cancellation
├── resource cleanup
├── connection lifetime
├── persistent runtime isolation
└── telemetry
```

estableciendo especialmente:

```text
Lazy Collection
≠
Loaded Collection
≠
Result Cursor
≠
Chunk
≠
Query Builder
```

con la regla central:

> **Una Lazy Collection de VoltStack representará un pipeline de consumo diferido sobre una fuente de datos potencialmente grande; diferirá la ejecución y materialización sin ocultar indefinidamente la propiedad de recursos, el I/O ni las fronteras de consistencia.**