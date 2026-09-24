# 205_DATABASE_BULK_DELETE_SYSTEM.md

# VoltStack Quantum Database
## Database Bulk Delete System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 205 — Database Bulk Delete System  
**Bloque:** 19 — Pagination, Batch & Large Data  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `204_DATABASE_BULK_UPDATE_SYSTEM.md`  
**Siguiente documento:** `206_DATABASE_IMPORT_SYSTEM.md`

---

# 1. Propósito

`Database Bulk Delete System` define la arquitectura mediante la cual VoltStack podrá eliminar grandes conjuntos de registros de forma directa, eficiente, tipada, gobernada y observable, sin cargar previamente cada registro como entidad ORM.

Ejemplo:

```php
$result = DB::table('sessions')
    ->where('expires_at', '<', $now)
    ->deleteBulk();
```

También deberá soportar eliminación por identificadores:

```php
$result = DB::table('notifications')->deleteBulkByKey(
    keys: [100, 101, 102, 103],
    key: 'id',
);
```

y fuentes incrementales:

```php
$result = DB::table('events')->deleteBulkByKey(
    keys: $eventIds,
    key: 'id',
    batchSize: 1000,
);
```

La regla central será:

> **Bulk Delete en VoltStack elimina directamente conjuntos de registros persistentes bajo una intención explícita de borrado; no simula `EntityManager::remove()` sobre entidades que nunca fueron cargadas, no fabrica lifecycle events ORM y no convierte automáticamente un hard delete en soft delete.**

Formalmente:

```text
BulkDelete
=
TargetSelection
+
DeleteIntent
+
DependencyAnalysis
+
Routing
+
BatchPlanning
+
TransactionPolicy
+
Execution
+
OutcomeAggregation
+
ORMCoherenceHandling
+
CacheInvalidation
```

Nunca:

```text
BulkDelete
=
foreach (LoadAllEntities() as $entity) {
    $entityManager->remove($entity);
}
```

---

# 2. Posición arquitectónica

Dentro del bloque actual:

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

Los tres sistemas bulk de mutación quedan conceptualmente:

```text
Bulk Insert
    │
    ├── crea registros
    │
Bulk Update
    │
    ├── modifica registros
    │
Bulk Delete
    │
    └── elimina registros
```

Todos compartirán infraestructura transversal, pero preservarán semánticas independientes.

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
Bulk Delete
≠
EntityManager::remove() × N
≠
UnitOfWork Removal
≠
Batch Persistence
≠
Soft Delete
≠
Cascade Remove
≠
Orphan Removal
≠
Data Retention
≠
Archival
≠
TRUNCATE
≠
Migration
```

Estas diferencias son críticas.

---

# 4. Bulk Delete vs ORM Remove

ORM:

```php
$user = $repository->find(10);

$entityManager->remove($user);
$entityManager->flush();
```

puede ejecutar:

```text
Entity
↓
EntityState
↓
UnitOfWork
↓
Relationship Analysis
↓
Cascade Rules
↓
Persistence Planner
↓
DELETE Query
```

Bulk Delete:

```text
Predicate / Keys
↓
BulkDeletePlanner
↓
DeleteQueryModel
↓
Query Engine
↓
Execution Engine
```

No requiere una entidad PHP.

---

# 5. Consecuencia principal

Después de:

```php
DB::table('users')
    ->where('status', 'inactive')
    ->deleteBulk();
```

puede existir:

```php
$user = $entityManager->find(User::class, 100);
```

en el `IdentityMap`.

La fila correspondiente podría haber desaparecido de la base de datos mientras el objeto continúa existiendo en memoria.

Por tanto:

```text
Database Row Deleted
≠
PHP Object Destroyed
```

y:

```text
Database State
≠
Current IdentityMap State
```

después de una mutación bulk externa al ORM.

---

# 6. El objeto PHP no desaparece

Una referencia:

```php
$user
```

puede continuar siendo perfectamente válida como objeto PHP.

Lo que cambia es su relación con el estado persistente.

Por ello:

```text
Row Deleted
≠
Object Garbage Collected
```

y tampoco necesariamente:

```text
Row Deleted
=
EntityState::REMOVED
```

porque el `UnitOfWork` nunca realizó esa transición.

---

# 7. Bulk Delete vs UnitOfWork Removal

En ORM:

```text
MANAGED
↓
remove()
↓
REMOVED
↓
flush()
↓
DELETE
```

Bulk Delete puede realizar:

```text
DELETE
```

sin que exista ninguna transición previa:

```text
MANAGED → REMOVED
```

---

# 8. Regla de verdad

VoltStack nunca deberá fabricar historial ORM.

Si el `UnitOfWork` no ejecutó:

```text
EntityState::REMOVED
```

el Bulk Delete no deberá afirmar que lo hizo.

---

# 9. Bulk Delete vs Batch Persistence

`133_DATABASE_BATCH_PERSISTENCE_SYSTEM.md` puede procesar muchas eliminaciones ORM:

```text
remove(entity A)
remove(entity B)
remove(entity C)
↓
UnitOfWork
↓
Batch Persistence
```

Bulk Delete trabaja directamente:

```text
DELETE FROM target
WHERE predicate
```

conceptualmente.

---

# 10. Bulk Delete vs Soft Delete

Esta distinción será obligatoria:

```text
Bulk Delete
≠
Soft Delete
```

Soft delete normalmente representa:

```text
UPDATE
SET deleted_at = ...
```

mientras hard delete representa:

```text
DELETE
```

---

# 11. Documento especializado

Soft Delete será definido formalmente en:

```text
269_DATABASE_SOFT_DELETE_SYSTEM.md
```

Bulk Delete no deberá duplicarlo.

---

# 12. Intención de eliminación

La API deberá representar explícitamente:

```text
DeleteIntent
```

y no inferir silenciosamente si el desarrollador quería:

```text
HARD_DELETE
SOFT_DELETE
ARCHIVE
PURGE
RETENTION_DELETE
```

---

# 13. DeleteIntent

Propuesta:

```php
enum DeleteIntent
{
    case HARD_DELETE;
    case SOFT_DELETE;
    case PURGE;
}
```

Sin embargo, `SOFT_DELETE` deberá delegar en el subsystem especializado cuando esté disponible.

---

# 14. Default

Una API llamada:

```php
deleteBulk()
```

deberá tener semántica documentada y estable.

Para el nivel Query Builder/Database bajo, la opción más clara es:

```text
deleteBulk()
=
physical delete
```

mientras APIs ORM superiores podrán ofrecer:

```php
softDeleteBulk();
```

---

# 15. No magia según modelo

No se recomienda:

```text
deleteBulk()
↓
inspect model
↓
sometimes DELETE
sometimes UPDATE deleted_at
```

en el API low-level.

La intención física debe permanecer visible.

---

# 16. Dos familias principales

El sistema deberá soportar:

```text
Set-Based Bulk Delete
Keyed Bulk Delete
```

---

# 17. Set-Based Bulk Delete

Ejemplo:

```php
DB::table('sessions')
    ->where('expires_at', '<', $now)
    ->deleteBulk();
```

Modelo:

```text
Target
+
Predicate
+
Delete Intent
```

---

# 18. Keyed Bulk Delete

Ejemplo:

```php
DB::table('users')->deleteBulkByKey(
    keys: [10, 20, 30],
    key: 'id',
);
```

Modelo:

```text
Key1
Key2
Key3
...
KeyN
```

---

# 19. Composite keys

Deberá soportarse:

```php
DB::table('tenant_settings')->deleteBulkByKey(
    keys: [
        [
            'tenant_id' => 10,
            'key' => 'theme',
        ],
        [
            'tenant_id' => 20,
            'key' => 'theme',
        ],
    ],
    key: [
        'tenant_id',
        'key',
    ],
);
```

---

# 20. Delete key

La key no necesariamente será PK.

Podrá ser una:

```text
Primary Key
Unique Key
Stable Natural Key
```

si su semántica es suficientemente precisa.

---

# 21. Non-unique key

Si:

```text
key = status
```

entonces:

```text
status = inactive
```

puede eliminar muchas filas por input key.

Default recomendado:

```text
REQUIRE_UNIQUE_KEY
```

para `deleteBulkByKey()`.

---

# 22. Predicate API

El sistema set-based deberá reutilizar el Query Builder:

```php
DB::table('logs')
    ->where('created_at', '<', $threshold)
    ->where('severity', 'debug')
    ->deleteBulk();
```

---

# 23. Query Model

El predicate deberá permanecer como:

```text
Query AST
```

y no convertirse anticipadamente en SQL.

---

# 24. Arquitectura general

```text
Bulk Delete API
      │
      ▼
BulkDeleteRequest
      │
      ▼
BulkDeleteNormalizer
      │
      ▼
BulkDeletePlanner
      │
      ├── Target Analysis
      ├── Predicate/Key Analysis
      ├── Safety Analysis
      ├── FK/Dependency Analysis
      ├── Routing
      ├── Batch Planning
      ├── Transaction Planning
      ├── Lock/Version Planning
      ├── Returning Planning
      ├── ORM Coherence
      ├── Cache Invalidation
      └── Resource Governance
      │
      ▼
BulkDeletePlan
      │
      ▼
BulkDeleteRunner
      │
      ▼
DeleteQueryModel(s)
      │
      ▼
Query Engine
      │
      ▼
Compiler
      │
      ▼
Execution Engine
      │
      ▼
BulkDeleteResult
```

---

# 25. Arquitectura compartida

Los documentos 203–205 podrán compartir:

```text
Database/Bulk/Common
```

con:

```text
BulkOperationContext
BulkBatchPlanner
BulkTransactionPolicy
BulkRoutingContext
BulkResourceBudget
BulkOperationStatus
BulkCancellation
BulkTelemetryContext
```

---

# 26. DeleteQueryModel

El Bulk Delete deberá converger al Query Engine existente.

Conceptualmente:

```php
final readonly class DeleteQueryModel
{
    public function __construct(
        public RelationTarget $target,
        public ?Predicate $predicate,
        public array $returning,
    ) {}
}
```

---

# 27. Bulk Delete no genera SQL

Nunca:

```text
BulkDeleteRunner
→ "DELETE FROM ..."
```

El SQL pertenece al Compiler.

---

# 28. Compiler

Las diferencias:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

deberán permanecer encapsuladas en:

```text
SQL Compiler
+
Platform Capability System
```

---

# 29. Seguridad de scope

Una de las mayores amenazas será:

```php
DB::table('users')->deleteBulk();
```

---

# 30. Default seguro

VoltStack deberá bloquear por default una eliminación sin selección explícita.

Propuesta:

```php
enum BulkDeleteScopeSafety
{
    case REQUIRE_PREDICATE;
    case ALLOW_ALL_EXPLICIT;
    case ALLOW_ALL;
}
```

---

# 31. Full-table delete

Para eliminar todas las filas deberá requerirse algo como:

```php
DB::table('temporary_events')
    ->allRows()
    ->deleteBulk();
```

---

# 32. allRows()

`allRows()` será una intención semántica:

```text
FullTableMutationIntent
```

no un simple:

```text
WHERE TRUE
```

---

# 33. WHERE TRUE

Esto:

```php
->whereRaw('1 = 1')
```

no deberá sustituir:

```php
->allRows()
```

para las protecciones de alto nivel.

---

# 34. DELETE ≠ TRUNCATE

Aunque se soliciten todas las filas:

```text
DELETE FROM table
≠
TRUNCATE TABLE
```

---

# 35. Razón

`TRUNCATE` puede diferir en:

```text
transaction semantics
identity reset
trigger behavior
locking
FK behavior
permissions
logging
platform behavior
```

---

# 36. No optimización automática a TRUNCATE

VoltStack no deberá transformar:

```text
DELETE all rows
```

en:

```text
TRUNCATE
```

como optimización transparente.

---

# 37. Truncate System

Si existe:

```php
DB::table('events')->truncate();
```

deberá ser una intención independiente.

---

# 38. Delete safety

El planner deberá considerar:

```text
has explicit predicate?
has explicit allRows?
estimated affected rows?
tenant scope present?
shard routing known?
FK dependencies?
soft-delete metadata?
production safety policy?
```

---

# 39. Impact estimation

Si el optimizer puede estimar:

```text
RowsEstimated = 50,000,000
```

podrá generarse:

```text
HIGH_IMPACT
```

---

# 40. Core no solicita confirmación

El Database core deberá producir:

```text
SafetyDiagnostic
```

La CLI podrá preguntar:

```text
Continue? [y/N]
```

pero esto no pertenece al core.

---

# 41. Hard affected-row limits

Podrá solicitarse:

```php
maxAffectedRows: 10_000
```

---

# 42. Dificultad

Un pre-count:

```text
SELECT COUNT(*)
```

antes del delete no garantiza que el mismo conjunto exista al ejecutar DELETE.

---

# 43. Race

```text
COUNT
↓
concurrent writes
↓
DELETE
```

puede producir diferente cardinalidad.

---

# 44. Guard classifications

VoltStack deberá distinguir:

```text
ESTIMATED
BEST_EFFORT
ENFORCED
```

---

# 45. Enforced limit

Puede requerir una estrategia:

```text
Select Keys
↓
Lock/Snapshot
↓
Validate Count
↓
Delete Selected Keys
```

según capacidades.

---

# 46. Foreign keys

Bulk Delete deberá considerar restricciones FK.

Ejemplo:

```text
users
  ↑
orders.user_id
```

Eliminar usuarios puede fallar si existen orders.

---

# 47. FK behavior

El resultado depende de:

```text
RESTRICT
NO ACTION
CASCADE
SET NULL
SET DEFAULT
```

y capacidades de plataforma.

---

# 48. Database cascade

Una FK:

```text
ON DELETE CASCADE
```

puede eliminar filas adicionales.

Por tanto:

```text
RowsDirectlyDeleted
≠
TotalRowsPhysicallyDeleted
```

---

# 49. Cascade visibility

El driver puede no informar cuántas filas indirectas fueron eliminadas.

No deberán inventarse.

---

# 50. Database Cascade ≠ ORM Cascade

Regla fundamental:

```text
Database ON DELETE CASCADE
≠
ORM cascade remove
```

---

# 51. ORM cascade remove

ORM cascade puede requerir:

```text
entity loading
lifecycle state
events
domain hooks
relationship traversal
```

---

# 52. Bulk Delete no simula ORM cascade

Nunca deberá afirmar:

```text
cascade remove executed
```

cuando solo actuó:

```text
database FK cascade
```

---

# 53. Orphan Removal

Igualmente:

```text
orphanRemoval
```

es una semántica ORM.

Bulk Delete no deberá recorrer asociaciones buscando orphans salvo un sistema especializado explícito.

---

# 54. Dependency analysis

Metadata de Schema y Relationship podrá advertir:

```text
Incoming FK
ORM Relationship
Cascade Difference
Potential Orphans
```

---

# 55. DependencyAnalysisResult

Propuesta:

```php
final readonly class BulkDeleteDependencyAnalysis
{
    public function __construct(
        public array $incomingForeignKeys,
        public array $ormRelationships,
        public array $databaseCascades,
        public array $restrictions,
        public BulkDeleteDependencyRisk $risk,
    ) {}
}
```

---

# 56. Risk levels

```text
NONE
LOW
MODERATE
HIGH
UNKNOWN
```

---

# 57. UNKNOWN ≠ SAFE

Si schema metadata es parcial:

```text
FK coverage = UNKNOWN
```

no deberá concluirse:

```text
no dependencies
```

---

# 58. Schema principle

Permanece:

```text
NotObserved
≠
Absent
```

---

# 59. Keyed delete batching

Para:

```text
10,000,000 IDs
```

no deberá generarse:

```text
WHERE id IN (?, ?, ..., 10M)
```

---

# 60. Batch planning

Conceptualmente:

```text
Input Keys
↓
Normalize
↓
Route
↓
Deduplicate
↓
Batch
↓
Delete Query Models
```

---

# 61. Effective batch size

```text
EffectiveBatchSize
=
min(
    RequestedBatchSize,
    ParameterCapacity,
    StatementCapacity,
    PlatformCapacity,
    ResourceBudget
)
```

---

# 62. Input sources

Se soportarán:

```text
array
Iterator
Generator
LazyCollection
BulkDeleteKeySource
```

---

# 63. Streaming

Ejemplo:

```php
DB::table('events')->deleteBulkByKey(
    keys: expiredEventIds(),
    key: 'id',
    batchSize: 2000,
);
```

deberá operar incrementalmente.

---

# 64. Backpressure

```text
Read Key Batch
↓
Normalize
↓
Route
↓
Execute
↓
Release
↓
Next Batch
```

---

# 65. Deduplication

Input:

```text
10
20
10
30
20
```

puede normalizarse a:

```text
10
20
30
```

dentro de un scope seguro.

---

# 66. Global deduplication

No deberá requerirse almacenar millones de IDs solo para deduplicar globalmente.

---

# 67. Deduplication modes

```php
enum BulkDeleteDeduplicationPolicy
{
    case NONE;
    case PER_BATCH;
    case GLOBAL_BOUNDED;
}
```

---

# 68. Duplicate delete

Eliminar la misma key dos veces puede producir:

```text
first → deleted
second → not found
```

por lo que la semántica de resultados debe considerar deduplicación.

---

# 69. Key normalization

Deberá utilizar:

```text
Type System
Schema Metadata
Canonical Identifier
```

---

# 70. Composite key normalization

El orden de componentes deberá ser determinista.

---

# 71. NULL key

Si una key única permite NULL, su semántica deberá respetar la plataforma.

No asumir:

```text
NULL = NULL
```

---

# 72. Empty input

```php
deleteBulkByKey(keys: [])
```

deberá producir:

```text
NO_OP
```

y jamás convertirse accidentalmente en:

```text
DELETE ALL
```

---

# 73. Invariante crítica

```text
Empty Key Set
≠
Missing Predicate
≠
All Rows
```

---

# 74. Key source failure

Si un Generator falla después de batches committed:

```text
PARTIAL
```

puede ser el outcome correcto.

---

# 75. Set-based strategy

Para una selección:

```text
WHERE expires_at < ?
```

la estrategia preferida será una operación set-based cuando sea segura.

---

# 76. Keyed strategies

El planner podrá elegir:

```text
IN_PREDICATE
VALUES_RELATION
TEMPORARY_RELATION
NATIVE_BULK_DELETE
CUSTOM
```

---

# 77. IN predicate

Adecuado para conjuntos moderados:

```text
DELETE
WHERE id IN (...)
```

como representación física posible.

---

# 78. VALUES relation

Algunas plataformas pueden utilizar una relación derivada de keys.

---

# 79. Temporary relation

Para millones de identificadores:

```text
Keys
↓
Temporary/Staging Relation
↓
Join Delete
```

puede ser más eficiente.

---

# 80. Temporary relation implications

Introduce:

```text
connection pinning
cleanup
transaction visibility
permissions
temporary storage
```

---

# 81. Strategy selection

Dependerá de:

```text
key count
key width
parameter limits
statement limits
platform capabilities
transaction policy
returning requirements
routing
resource budget
```

---

# 82. Version ≠ Capability

No deberá existir lógica central:

```php
if ($db === 'postgres') { ... }
```

La decisión debe pasar por:

```text
PlatformCapabilitySystem
```

---

# 83. Returning

Plataformas que soporten:

```text
DELETE ... RETURNING
```

podrán devolver:

```text
identifiers
versions
audit fields
selected deleted columns
```

---

# 84. API

Ejemplo:

```php
$result = DB::table('sessions')
    ->where('expires_at', '<', $now)
    ->deleteBulk(
        returning: ['id'],
    );
```

---

# 85. Returning all deleted data

Esto:

```text
RETURNING *
```

para millones de filas puede consumir enorme memoria.

---

# 86. Resource policy

Se deberán controlar:

```text
maxReturningRows
maxReturningBytes
```

---

# 87. Streaming returning

Podrá existir:

```text
BulkDeleteReturningSink
```

para evitar materialización completa.

---

# 88. Returning ≠ entity hydration

Los registros eliminados devueltos no deberán convertirse automáticamente en entidades managed.

---

# 89. Razón

La fila ya no existe.

Crear una entidad managed desde:

```text
DELETE RETURNING
```

sería semánticamente incorrecto.

---

# 90. Deleted projection

Podrá devolverse:

```text
DeletedRow
DeletedIdentifier
DTO
Tuple
Scalar
```

pero no una nueva entidad managed normal.

---

# 91. Returning + existing IdentityMap

Si se devuelve:

```text
id = 100
```

y existe:

```text
IdentityMap[User:100]
```

la coherence policy podrá identificarla como afectada.

---

# 92. ORM coherence policies

Propuesta:

```php
enum BulkDeleteOrmCoherencePolicy
{
    case IGNORE;
    case MARK_POTENTIALLY_DELETED;
    case DETACH_KNOWN_ENTITIES;
    case REJECT_IF_MANAGED_ENTITIES_MATCH;
    case CUSTOM;
}
```

---

# 93. Default recomendado

```text
MARK_POTENTIALLY_DELETED
```

cuando exista un EntityManager activo y pueda identificarse el EntityType.

---

# 94. Potentially deleted

El ORM podrá registrar:

```text
ExternalDeletionMarker
```

sin fingir:

```text
EntityState::REMOVED
```

---

# 95. New semantic state

Puede ser útil una condición interna:

```text
PERSISTENCE_PRESENCE_UNKNOWN
POTENTIALLY_DELETED_EXTERNALLY
```

separada del lifecycle state tradicional.

---

# 96. REMOVED

Debe reservarse para entidades que atravesaron el protocolo ORM correspondiente.

---

# 97. Known key deletion

Si:

```text
ids = [10,20,30]
```

el EntityManager puede detectar instancias managed con esas identities.

---

# 98. Detach policy

Una policy explícita podrá hacer:

```text
IdentityMap remove
+
EntityState → DETACHED
```

después de commit confirmado.

---

# 99. Detach ≠ remove

Esto es crítico:

```text
DETACHED
≠
REMOVED
```

---

# 100. External references

Aunque se retire del IdentityMap:

```php
$user
```

puede continuar existiendo.

---

# 101. Dirty managed entity

Caso crítico:

```text
User:100
managed + dirty

Bulk Delete
deletes User:100
```

---

# 102. No silent detach

Si hay cambios no flushed, la policy deberá poder:

```text
REJECT
MARK_CONFLICTED
DETACH_WITH_WARNING
```

pero nunca perder estado silenciosamente.

---

# 103. Flush posterior

Sin protección:

```text
Bulk Delete row
↓
later ORM flush
↓
UPDATE deleted row
```

puede producir:

```text
0 affected rows
```

y ser interpretado como:

```text
stale entity
optimistic conflict
missing row
```

---

# 104. UoW integration

El Bulk Delete podrá emitir:

```text
ExternalBulkDeletion
```

al `UnitOfWork`.

---

# 105. UoW authority

El `UnitOfWork` decidirá cómo representar:

```text
managed entity potentially missing
```

sin que Bulk Delete manipule internamente sus snapshots.

---

# 106. Snapshots

No deberán borrarse o modificarse silenciosamente.

---

# 107. Relationship coherence

Eliminar una entidad puede volver stale:

```text
to-one references
to-many collections
many-to-many membership
inverse associations
```

---

# 108. Example

```text
User 10
├── Order 1
├── Order 2
└── Order 3
```

Si User 10 es eliminado por Bulk Delete, un objeto:

```php
$order->customer
```

puede continuar apuntando al objeto `User`.

---

# 109. No automatic graph rewrite

Bulk Delete no deberá recorrer el grafo PHP intentando convertir automáticamente:

```text
Order.customer → null
```

---

# 110. Razón

La relación DB puede incluso ser:

```text
RESTRICT
CASCADE
SET NULL
```

y el estado in-memory podría contener cambios locales.

---

# 111. Relationship stale markers

El sistema podrá notificar:

```text
RelationshipPotentiallyStale
CollectionPotentiallyStale
ReferencePotentiallyMissing
```

---

# 112. Database cascade implications

Si:

```text
User delete
↓
ON DELETE CASCADE
↓
Orders deleted
```

entonces entidades `Order` managed también pueden quedar stale.

---

# 113. Cascade impact graph

Schema metadata podrá producir:

```text
users
↓ CASCADE
orders
↓ CASCADE
order_items
```

---

# 114. Dependency graph

Podrá utilizarse para estimar:

```text
DirectTarget
IndirectTargets
PotentialCascadeDepth
```

---

# 115. Pero

El sistema no deberá afirmar exactamente qué filas indirectas fueron borradas sin evidencia suficiente.

---

# 116. Cascade cycles

Schemas complejos pueden tener ciclos o cascadas diferidas.

El análisis deberá ser cycle-safe.

---

# 117. Optimistic delete

Bulk Delete deberá soportar optimistic locking.

Ejemplo keyed:

```php
[
    [
        'id' => 100,
        'version' => 5,
    ],
    [
        'id' => 101,
        'version' => 8,
    ],
]
```

---

# 118. Semántica

```text
DELETE
iff
CurrentVersion = ExpectedVersion
```

---

# 119. Outcomes

Por key:

```text
DELETED
CONFLICT
NOT_FOUND
UNKNOWN
```

cuando la estrategia permita distinguirlos.

---

# 120. Zero affected

Sin información adicional:

```text
0 rows
```

puede significar:

```text
not found
version mismatch
scope mismatch
tenant mismatch
concurrent delete
```

---

# 121. No inventar

No deberá convertirse automáticamente en:

```text
OptimisticLockException
```

sin evidencia.

---

# 122. Pessimistic locking

Bulk Delete puede ejecutarse dentro de una transacción que previamente adquirió locks.

No deberá adquirirlos implícitamente por default.

---

# 123. Concurrent delete

Dos operaciones:

```text
A deletes ID 10
B deletes ID 10
```

pueden producir:

```text
A → DELETED
B → NOT_FOUND / 0 affected
```

---

# 124. Idempotency

Un hard delete por key puede ser operacionalmente idempotente bajo ciertas definiciones:

```text
delete ID 10
delete ID 10 again
```

deja el mismo estado final.

Pero:

```text
same final state
≠
same observable side effects
```

---

# 125. Side effects

Triggers, audit systems, cascades o external events pueden cambiar la replay safety.

---

# 126. Retry safety

Por tanto:

```text
DELETE
```

no deberá marcarse universalmente como seguro para retry.

---

# 127. Retry policy

```php
enum BulkDeleteRetryPolicy
{
    case NEVER;
    case TRANSACTION_SAFE;
    case IDEMPOTENT_ONLY;
    case CUSTOM;
}
```

---

# 128. UNKNOWN commit

Si:

```text
COMMIT sent
↓
connection lost
```

entonces:

```text
UNKNOWN
```

---

# 129. Blind retry forbidden

No deberá ejecutarse automáticamente:

```text
retry delete
```

si el primer commit tiene outcome desconocido y existen side effects no replay-safe.

---

# 130. Transaction policies

Se reutilizará:

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

# 131. USE_EXISTING

Nunca:

```text
BulkDelete
→ commit caller transaction
```

---

# 132. WHOLE_OPERATION

Solo puede garantizar atomicidad global cuando:

```text
single transactional domain
+
all batches participate
+
commit confirmed
```

---

# 133. PER_BATCH

Ejemplo:

```text
Batch 1 → commit
Batch 2 → commit
Batch 3 → failure
```

resultado:

```text
PARTIAL
```

---

# 134. Cross-shard

Si:

```text
Batch 1 → Shard A
Batch 2 → Shard B
```

no deberá afirmarse:

```text
GLOBAL_ATOMIC
```

---

# 135. Savepoints

Podrán utilizarse mediante Transaction System.

Pero:

```text
Savepoint
≠
Independent Transaction
```

---

# 136. Sharding

Bulk Delete deberá resolverse antes de adquirir endpoint.

---

# 137. Set-based routing

Resultado conceptual:

```text
ONE_SHARD
SHARD_SET
ALL_SHARDS
UNKNOWN
```

---

# 138. UNKNOWN write routing

Default:

```text
UNKNOWN
→ REJECT
```

No:

```text
UNKNOWN
→ broadcast DELETE
```

---

# 139. Broadcast delete

Deberá requerir intención administrativa explícita.

---

# 140. Keyed routing

Cada key podrá resolverse:

```text
Key 10 → Shard A
Key 20 → Shard B
Key 30 → Shard A
```

y agruparse:

```text
Shard A → [10,30]
Shard B → [20]
```

---

# 141. Shard routing before batching

Preferentemente:

```text
Normalize
↓
Route
↓
Batch per shard
```

para preservar ownership.

---

# 142. Partition movement

Una eliminación durante rebalancing puede requerir:

```text
ShardMapGeneration
```

para evitar usar ownership obsoleto.

---

# 143. Routing evidence

`BulkDeletePlan` deberá registrar:

```text
ShardMapGeneration
RoutingConfidence
TargetShardSet
```

---

# 144. Tenant isolation

Toda eliminación tenant-aware deberá incluir contexto de tenant en:

```text
routing
query scope
cache invalidation
telemetry domain
ORM identity domain
```

---

# 145. Missing tenant context

Cuando sea obligatorio:

```text
MissingTenantContext
→ reject
```

---

# 146. No accidental cross-tenant delete

Especialmente:

```php
DB::table('documents')
    ->allRows()
    ->deleteBulk();
```

no deberá eliminar datos de otros tenants cuando existe aislamiento tenant-aware.

---

# 147. Tenant scope ≠ application predicate opcional

Cuando el persistence domain requiere tenant:

```text
Tenant Scope
=
mandatory structural context
```

---

# 148. Read/write routing

Delete siempre será:

```text
WRITE
```

---

# 149. Replica

Nunca:

```text
DELETE → replica
```

---

# 150. Sticky reads

Después de delete, read-your-writes puede requerir writer/sticky behavior.

Se reutilizará:

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

---

# 151. Cache invalidation

Bulk Delete deberá integrarse con:

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM
192_DATABASE_CACHE_CONSISTENCY_SYSTEM
```

---

# 152. Semantic changes

Podrán emitirse:

```text
EntityDeleted
EntityKeySetDeleted
EntityPredicateDeleted
RelationshipMembershipChanged
TableRowsDeleted
CascadePotentiallyDeleted
```

---

# 153. Exact keys

Para:

```text
IDs [10,20,30]
```

puede existir:

```text
ENTITY_EXACT
KEY_SET
```

---

# 154. Predicate delete

Para:

```text
created_at < X
```

puede requerirse:

```text
PREDICATE_BOUNDED
TABLE_WIDE
```

---

# 155. Database cascades

Si el DB elimina relaciones indirectas, la invalidación deberá incluir dependencias conocidas.

---

# 156. Unknown cascade coverage

Si no se conoce todo el impacto:

```text
UNKNOWN
→ conservative invalidation
```

---

# 157. Entity Cache

Entradas eliminadas deberán invalidarse tras commit confirmado.

---

# 158. Tombstones

Opcionalmente podrá utilizarse:

```text
Cache Tombstone
```

para evitar repoblar inmediatamente una entidad eliminada desde una replica atrasada.

---

# 159. Tombstone ≠ Soft Delete

Un cache tombstone significa:

```text
do not trust/repopulate this cached identity yet
```

No significa:

```text
database row is soft deleted
```

---

# 160. Replica lag problem

Secuencia:

```text
Writer:
DELETE User 10

Cache:
invalidate User 10

Replica:
still contains User 10

Read:
replica returns User 10

Cache:
repopulates stale entity
```

---

# 161. Protection

Podrá utilizarse:

```text
sticky writer
minimum replication position
entity tombstone
version evidence
bounded freshness policy
```

---

# 162. Readable ≠ Cacheable

Una replica puede ser técnicamente legible y aun no ser fuente segura para repoblar cache tras un delete reciente.

---

# 163. Result Cache

Queries dependientes de los registros eliminados deberán invalidarse.

---

# 164. Relationship Cache

Colecciones como:

```text
Customer.orders
```

deberán invalidarse cuando el delete pueda alterar membership.

---

# 165. Cache timing

Nunca publicar invalidación como resultado confirmado antes de saber el outcome transaccional, salvo mecanismos internos preparados para rollback.

Default:

```text
confirmed commit
→ publish invalidation
```

---

# 166. UNKNOWN commit

Requerirá política conservadora.

---

# 167. Cache failure post-commit

No puede deshacer el DELETE.

---

# 168. Events

Eventos conceptuales:

```text
BulkDeletePlanned
BulkDeleteStarted
BulkDeleteBatchStarted
BulkDeleteBatchCompleted
BulkDeleteConflictDetected
BulkDeleteCascadeRiskDetected
BulkDeleteCommitted
BulkDeleteOrmStateMarkedStale
BulkDeleteCacheInvalidationScheduled
BulkDeleteCompleted
BulkDeletePartial
BulkDeleteCancelled
BulkDeleteOutcomeUnknown
BulkDeleteFailed
```

---

# 169. Bulk events ≠ entity lifecycle events

No se emitirán automáticamente por cada fila:

```text
preRemove
postRemove
deleting
deleted
```

---

# 170. Razón

Las entidades podrían no existir en memoria.

---

# 171. Event explosion

Eliminar:

```text
20,000,000 rows
```

no deberá crear:

```text
20,000,000 framework events
```

por default.

---

# 172. Event aggregation

Preferir:

```text
operation
batch
summary
```

---

# 173. Domain events

Si el dominio requiere un evento individual por aggregate eliminado, probablemente deberá utilizarse un workflow ORM/domain específico y no Bulk Delete directo.

---

# 174. Audit

Bulk Delete deberá integrarse posteriormente con:

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

---

# 175. Audit metadata

Podrá registrar:

```text
operation type
logical target
actor/security context
tenant domain
shard domain
reason
timestamp
estimated/actual counts
outcome
```

---

# 176. No raw sensitive predicates

Audit/telemetry deberá evitar exponer valores sensibles innecesariamente.

---

# 177. Delete reason

Para operaciones administrativas puede requerirse:

```php
reason: 'retention-policy-90-days'
```

---

# 178. Reason ≠ authorization

Una razón descriptiva no concede permisos.

---

# 179. Authorization

Bulk Delete deberá respetar:

```text
Authorization
Data Access Security
Tenant Scope
Resource Scope
```

---

# 180. Authorization before execution

Nunca:

```text
delete
↓
check permission
```

---

# 181. Set-based authorization

Para autorización row-level, el scope autorizado deberá formar parte de la selección efectiva.

---

# 182. Bulk authorization challenge

No siempre es viable ejecutar una policy PHP individual para millones de registros sin cargarlos.

---

# 183. Policy translation

Sistemas avanzados podrán traducir authorization scope a:

```text
Query Predicate
```

cuando sea semánticamente seguro.

---

# 184. Unsupported authorization

Si una policy solo puede evaluarse sobre objetos individuales y no puede traducirse, Bulk Delete deberá:

```text
reject
```

o exigir una autorización administrativa explícita.

No deberá saltarse la policy.

---

# 185. Security principle

```text
Cannot Efficiently Authorize
≠
Authorized
```

---

# 186. Soft delete boundary

Una API ORM superior podría ofrecer:

```php
User::query()
    ->where(...)
    ->softDeleteBulk();
```

---

# 187. Soft-delete implementation

Conceptualmente delegará:

```text
Soft Delete System
↓
Bulk Update System
```

más que al hard Bulk Delete.

---

# 188. Purge

Después de soft delete podría existir:

```text
purge
```

que físicamente elimina registros.

---

# 189. Purge safety

Purge puede requerir:

```text
already soft deleted
retention age reached
legal hold absent
authorization
```

---

# 190. Data retention boundary

Esto pertenecerá principalmente a:

```text
270_DATABASE_DATA_RETENTION_SYSTEM.md
```

---

# 191. Archival boundary

Antes de eliminar puede requerirse archivar.

Eso pertenecerá a:

```text
271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
```

---

# 192. Bulk Delete no archivará automáticamente

Nunca:

```text
deleteBulk()
→ secretly archive everything first
```

---

# 193. Compliance

Sistemas de compliance podrán interceptar mediante policy:

```text
ALLOW
REJECT
REQUIRE_ARCHIVE
REQUIRE_APPROVAL
LEGAL_HOLD
```

---

# 194. Legal hold

Si un registro está bajo retención legal:

```text
Bulk Delete
→ REJECT
```

cuando la integración correspondiente esté activa.

---

# 195. Importante

El core Database no deberá conocer conceptos jurídicos específicos.

Usará extension points/policies.

---

# 196. Cancellation

`BulkDeleteRunner` deberá aceptar:

```text
CancellationToken
```

---

# 197. Between batches

```text
Batch 1 committed
Batch 2 committed
Cancellation
```

resultado:

```text
PARTIAL / CANCELLED_WITH_COMMITTED_WORK
```

---

# 198. Cancellation within statement

Dependerá de capacidades del Driver/Execution Engine.

---

# 199. Cancel requested ≠ cancelled

El driver puede no poder interrumpir una operación ya enviada.

---

# 200. Cancelled ≠ rolled back

Siempre:

```text
Cancellation
≠
Confirmed Rollback
```

---

# 201. Timeouts

Se reutilizará:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
```

---

# 202. Resource governance

Deberá controlar:

```text
input keys
batch size
parameter count
statement size
estimated affected rows
cascade risk
returning rows
returning bytes
duration
memory
concurrency
```

---

# 203. Cascade amplification

Una operación:

```text
delete 1,000 parents
```

podría provocar:

```text
delete 10,000,000 children
```

por cascadas DB.

---

# 204. Amplification estimate

Cuando metadata/estadísticas lo permitan:

```text
DirectDeleteEstimate
CascadeDeleteEstimate
TotalImpactEstimate
```

---

# 205. UNKNOWN cascade impact

Debe mostrarse como:

```text
UNKNOWN
```

no cero.

---

# 206. ResourceBudget

```php
final readonly class BulkDeleteResourceBudget
{
    public function __construct(
        public ?int $maxInputKeys,
        public ?int $maxDirectAffectedRows,
        public ?int $maxEstimatedCascadeRows,
        public ?int $maxBatchRows,
        public ?int $maxParameters,
        public ?int $maxReturningRows,
        public ?int $maxReturningBytes,
        public ?int $maxDurationMs,
        public ?int $maxMemoryBytes,
    ) {}
}
```

---

# 207. Failure model

Distinguir:

```text
INPUT_FAILURE
VALIDATION_FAILURE
SAFETY_FAILURE
AUTHORIZATION_FAILURE
DEPENDENCY_FAILURE
ROUTING_FAILURE
PLANNING_FAILURE
COMPILATION_FAILURE
CONSTRAINT_FAILURE
EXECUTION_FAILURE
OPTIMISTIC_CONFLICT
TRANSACTION_FAILURE
COMMIT_UNKNOWN
RETURNING_FAILURE
ORM_COHERENCE_FAILURE
CACHE_INVALIDATION_FAILURE
CANCELLATION
```

---

# 208. FK constraint failure

Ejemplo:

```text
DELETE parent
↓
RESTRICT FK
↓
constraint violation
```

deberá mapearse a un error DB estructurado.

---

# 209. Constraint error ≠ planner failure

Si metadata estaba incompleta, la DB continúa siendo autoridad final.

---

# 210. Trigger failure

Un trigger DB puede rechazar el delete.

Eso será:

```text
Execution/Database Failure
```

no ORM lifecycle failure.

---

# 211. Partial execution

En `PER_BATCH`:

```text
Batch 1 succeeded
Batch 2 succeeded
Batch 3 FK failure
```

resultado:

```text
PARTIAL
```

---

# 212. Whole transaction

Si todos estaban dentro de una transacción propia y rollback se confirma:

```text
FAILED
+
ROLLED_BACK_CONFIRMED
```

---

# 213. Rollback unknown

Si rollback no puede confirmarse:

```text
UNKNOWN
```

cuando la realidad persistente no pueda determinarse.

---

# 214. Result model

```php
final readonly class BulkDeleteResult
{
    public function __construct(
        public BulkDeleteStatus $status,
        public BulkDeleteCountResult $counts,
        public array $batchResults,
        public BulkDeleteReturningResult $returning,
        public BulkDeleteOrmCoherenceResult $ormCoherence,
        public BulkDeleteCascadeResult $cascade,
        public BulkDeleteOutcomeEvidence $evidence,
    ) {}
}
```

---

# 215. Status

```php
enum BulkDeleteStatus
{
    case SUCCEEDED;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 216. Counts

Deberá distinguirse:

```text
RequestedKeys
MatchedRows
DirectRowsDeleted
ReportedRows
CascadeRowsKnown
TotalRowsKnown
```

---

# 217. Unknown counts

Podrán ser:

```text
null
+
knowledge = UNKNOWN
```

---

# 218. No invented cascade counts

Si el DB reporta solo:

```text
100 direct rows
```

no deberá afirmarse:

```text
100 total rows
```

si existen cascadas.

---

# 219. Result per key

Para keyed delete puede existir opcionalmente:

```text
Key 10 → DELETED
Key 20 → NOT_FOUND
Key 30 → CONFLICT
```

solo cuando la estrategia proporcione evidencia suficiente.

---

# 220. Per-key detail cost

Obtener outcomes exactos por key puede requerir:

```text
RETURNING
extra queries
temporary relation
```

---

# 221. Detail policy

```php
enum BulkDeleteResultDetail
{
    case SUMMARY;
    case PER_BATCH;
    case PER_KEY;
}
```

---

# 222. Default

Para grandes datasets:

```text
SUMMARY
```

o `PER_BATCH` será más eficiente.

---

# 223. Persistent runtime

En FrankenPHP:

```text
Request A BulkDelete
Request B BulkDelete
```

no compartirán:

```text
keys
tenant
transaction
shard map snapshot
returning buffer
cancellation
ORM markers
```

---

# 224. Operation context

```php
final class BulkDeleteOperationContext
{
    // mutable operation-scoped state only
}
```

---

# 225. No static state

Nunca:

```php
BulkDeleteRunner::$deletedIds
```

---

# 226. RoadRunner

Mismo principio.

---

# 227. OpenSwoole

Cada coroutine deberá poseer su contexto independiente.

---

# 228. Shared infrastructure

Podrá compartirse únicamente infraestructura segura:

```text
immutable metadata
stateless planners
compiled schemas
platform capabilities
connection pool infrastructure
```

---

# 229. Telemetry metrics

```text
db.bulk_delete.operations
db.bulk_delete.duration
db.bulk_delete.batches
db.bulk_delete.requested_keys
db.bulk_delete.rows_reported
db.bulk_delete.conflicts
db.bulk_delete.constraint_failures
db.bulk_delete.partial
db.bulk_delete.unknown
db.bulk_delete.cascade_risk
db.bulk_delete.orm_stale
```

---

# 230. Cardinality control

No usar como labels:

```text
deleted IDs
tenant IDs
raw SQL
raw predicates
email addresses
tokens
full keys
```

---

# 231. Explain API

Conceptualmente:

```php
DB::bulk()->explainDelete(
    DB::table('sessions')
        ->where('expires_at', '<', $now),
);
```

---

# 232. Explain set-based

Salida conceptual:

```text
BULK DELETE PLAN

Mode:
    SET_BASED

Target:
    sessions

Delete Intent:
    HARD_DELETE

Selection:
    expires_at < :threshold

Scope Safety:
    PREDICATE PRESENT

Estimated Direct Rows:
    2,450,000

Foreign Keys:
    2 incoming

Database Cascades:
    1

Cascade Impact:
    UNKNOWN

ORM Relationships:
    3 potentially affected

Execution:
    SINGLE SET-BASED DELETE

Transaction:
    STATEMENT ATOMICITY

Returning:
    NONE

Routing:
    WRITER
    SINGLE SHARD

ORM Coherence:
    MARK ENTITY TYPE POTENTIALLY DELETED

Cache Invalidation:
    PREDICATE_BOUNDED
    CASCADE DEPENDENCIES INCLUDED

Retry Safety:
    CONDITIONAL

Overall Risk:
    HIGH
```

---

# 233. Explain keyed

```text
BULK DELETE PLAN

Mode:
    KEYED

Input:
    STREAMING

Key:
    id

Key Uniqueness:
    VERIFIED

Estimated Keys:
    5,000,000

Execution Strategy:
    IN_PREDICATE

Requested Batch Size:
    2,000

Effective Batch Size:
    900

Limiting Factor:
    PARAMETER CAPACITY

Deduplication:
    PER_BATCH

Transaction:
    PER_BATCH

Global Atomicity:
    NOT PROVIDED

Optimistic Locking:
    NONE

Returning:
    id

Routing:
    GROUP BY SHARD

ORM:
    KNOWN IDENTITIES MARKED POTENTIALLY DELETED

Cache:
    ENTITY KEY INVALIDATION
    RELATIONSHIP INVALIDATION

Cascade Risk:
    MODERATE
```

---

# 234. BulkDeletePlan

```php
final readonly class BulkDeletePlan
{
    public function __construct(
        public BulkDeleteMode $mode,
        public DeleteTarget $target,
        public DeleteIntent $intent,
        public BulkDeleteSelectionPlan $selection,
        public BulkDeleteSafetyPlan $safety,
        public BulkDeleteDependencyAnalysis $dependencies,
        public BulkDeleteExecutionPlan $execution,
        public BulkBatchPlan $batch,
        public BulkRoutingPlan $routing,
        public BulkTransactionPlan $transaction,
        public BulkDeleteLockPlan $locking,
        public BulkDeleteReturningPlan $returning,
        public BulkDeleteOrmCoherencePlan $orm,
        public BulkDeleteCachePlan $cache,
        public BulkDeleteResourcePlan $resources,
    ) {}
}
```

---

# 235. Planner ≠ Runner

```text
BulkDeletePlanner
```

deberá producir un plan inmutable.

```text
BulkDeleteRunner
```

lo ejecutará.

---

# 236. Runner ≠ Executor

Runner coordina:

```text
batches
routing
transaction boundaries
results
coherence
```

pero no implementa protocolo DB.

---

# 237. Executor

`QueryExecutor` continúa ejecutando statements.

---

# 238. Connection

Bulk Delete no deberá conocer:

```text
PDO
mysqli
pgsql extension
```

---

# 239. Driver

Driver continuará encapsulando detalles protocol-level.

---

# 240. Directory structure

```text
src/Quantum/Database/Bulk/
│
├── Common/
│   ├── BulkOperationContext.php
│   ├── BulkOperationStatus.php
│   ├── BulkBatchPlan.php
│   ├── BulkTransactionPolicy.php
│   ├── BulkRoutingPlan.php
│   └── BulkResourceBudget.php
│
└── Delete/
    ├── BulkDeleteRequest.php
    ├── BulkDeleteOptions.php
    ├── BulkDeleteMode.php
    ├── BulkDeletePlanner.php
    ├── BulkDeletePlan.php
    ├── BulkDeleteRunner.php
    ├── BulkDeleteResult.php
    ├── BulkDeleteStatus.php
    ├── DeleteIntent.php
    │
    ├── Selection/
    │   ├── BulkDeleteSelectionPlan.php
    │   ├── BulkDeleteScopeSafety.php
    │   └── FullTableDeleteIntent.php
    │
    ├── Key/
    │   ├── BulkDeleteKey.php
    │   ├── BulkDeleteKeySource.php
    │   ├── BulkDeleteKeyNormalizer.php
    │   ├── BulkDeleteDeduplicationPolicy.php
    │   └── BulkDeleteKeyBatch.php
    │
    ├── Dependency/
    │   ├── BulkDeleteDependencyAnalyzer.php
    │   ├── BulkDeleteDependencyAnalysis.php
    │   ├── BulkDeleteDependencyRisk.php
    │   └── BulkDeleteCascadeGraph.php
    │
    ├── Strategy/
    │   ├── BulkDeleteExecutionStrategy.php
    │   ├── BulkDeleteStrategyResolver.php
    │   ├── SetBasedDeleteStrategy.php
    │   ├── InPredicateDeleteStrategy.php
    │   ├── ValuesRelationDeleteStrategy.php
    │   └── TemporaryRelationDeleteStrategy.php
    │
    ├── Locking/
    │   ├── BulkDeleteLockPlan.php
    │   ├── BulkDeleteOptimisticConflict.php
    │   └── BulkDeleteConcurrencyPolicy.php
    │
    ├── Returning/
    │   ├── BulkDeleteReturningPlan.php
    │   ├── BulkDeleteReturningResult.php
    │   └── BulkDeleteReturningSink.php
    │
    ├── Count/
    │   ├── BulkDeleteCountResult.php
    │   └── BulkDeleteCountKnowledge.php
    │
    ├── ORM/
    │   ├── BulkDeleteOrmCoherencePolicy.php
    │   ├── BulkDeleteOrmCoherencePlan.php
    │   ├── BulkDeleteOrmCoherenceResult.php
    │   └── ExternalBulkDeletionNotifier.php
    │
    ├── Cache/
    │   ├── BulkDeleteCachePlan.php
    │   └── BulkDeleteTombstonePolicy.php
    │
    ├── Routing/
    │   └── BulkDeleteRoutingPlanner.php
    │
    ├── Retry/
    │   ├── BulkDeleteRetryPolicy.php
    │   └── BulkDeleteReplaySafety.php
    │
    ├── Resource/
    │   ├── BulkDeleteResourceBudget.php
    │   └── BulkDeleteResourcePlan.php
    │
    ├── Runtime/
    │   └── BulkDeleteOperationContext.php
    │
    ├── Diagnostics/
    │   ├── BulkDeleteInspector.php
    │   └── BulkDeleteExplainer.php
    │
    ├── Telemetry/
    │   └── BulkDeleteTelemetry.php
    │
    └── Exception/
        └── ...
```

---

# 241. Error hierarchy

```text
DatabaseException
└── BulkOperationException
    └── BulkDeleteException
        ├── BulkDeleteInputException
        ├── BulkDeleteValidationException
        ├── BulkDeletePlanningException
        ├── BulkDeleteScopeException
        ├── BulkDeleteUnsafeOperationException
        ├── BulkDeleteAuthorizationException
        ├── BulkDeleteKeyException
        ├── BulkDeleteDependencyException
        ├── BulkDeleteConstraintException
        ├── BulkDeleteExecutionException
        ├── BulkDeleteUnsupportedCapabilityException
        ├── BulkDeleteOptimisticConflictException
        ├── BulkDeleteRoutingException
        ├── BulkDeleteTransactionException
        ├── BulkDeleteRetryUnsafeException
        ├── BulkDeleteReturningException
        ├── BulkDeleteOrmCoherenceException
        ├── BulkDeleteResourceException
        └── BulkDeleteOutcomeUnknownException
```

---

# 242. Testing matrix

| Área | Caso |
|---|---|
| Set-based | predicate delete |
| Set-based | no predicate |
| Set-based | explicit allRows |
| Keyed | simple PK |
| Keyed | composite key |
| Keyed | unique natural key |
| Keyed | non-unique rejected |
| Input | empty |
| Input | array |
| Input | generator |
| Input | LazyCollection |
| Input | duplicate keys |
| Batch | parameter limit |
| Batch | streaming |
| FK | RESTRICT |
| FK | CASCADE |
| FK | SET NULL |
| FK | unknown metadata |
| ORM | no EntityManager |
| ORM | managed entity |
| ORM | dirty managed entity |
| ORM | known-key detach |
| ORM | relationship stale |
| Cascade | managed child entity |
| Locking | optimistic success |
| Locking | conflict |
| Locking | missing vs conflict unknown |
| Transaction | none |
| Transaction | existing |
| Transaction | whole operation |
| Transaction | per batch |
| Failure | middle batch |
| Failure | commit unknown |
| Retry | safe boundary |
| Retry | trigger side effects |
| Returning | IDs |
| Returning | large dataset |
| Cache | exact invalidation |
| Cache | predicate invalidation |
| Cache | tombstone |
| Replica | stale repopulation |
| Sharding | one shard |
| Sharding | multiple shards |
| Sharding | unknown routing |
| Tenant | tenant isolation |
| Authorization | translatable scope |
| Authorization | non-translatable policy |
| Cancellation | between batches |
| Runtime | worker reuse |
| Runtime | coroutine isolation |

---

# 243. Architectural invariants

## DB-BDEL-001
Bulk Delete será distinto de `EntityManager::remove()`.

## DB-BDEL-002
Bulk Delete será distinto de UnitOfWork removal.

## DB-BDEL-003
Bulk Delete será distinto de Batch Persistence.

## DB-BDEL-004
Bulk Delete será distinto de Soft Delete.

## DB-BDEL-005
Bulk Delete será distinto de Cascade Remove ORM.

## DB-BDEL-006
Bulk Delete será distinto de Orphan Removal.

## DB-BDEL-007
Bulk Delete será distinto de TRUNCATE.

## DB-BDEL-008
Bulk Delete no requerirá cargar entidades.

## DB-BDEL-009
Bulk Delete operará mediante Query Engine.

## DB-BDEL-010
Bulk Delete no generará SQL directamente.

## DB-BDEL-011
Compiler continuará siendo autoridad SQL.

## DB-BDEL-012
Execution Engine continuará siendo autoridad de ejecución.

## DB-BDEL-013
Driver continuará siendo autoridad protocol-level.

## DB-BDEL-014
Set-based y keyed delete serán semánticas distintas.

## DB-BDEL-015
Empty key set será NO_OP.

## DB-BDEL-016
Empty key set nunca significará delete all.

## DB-BDEL-017
Missing predicate será distinto de allRows.

## DB-BDEL-018
Full-table delete requerirá intención explícita bajo defaults seguros.

## DB-BDEL-019
WHERE TRUE no sustituirá allRows.

## DB-BDEL-020
DELETE all rows no se convertirá automáticamente en TRUNCATE.

## DB-BDEL-021
DeleteIntent será explícito.

## DB-BDEL-022
Hard Delete será distinto de Soft Delete.

## DB-BDEL-023
Soft Delete delegará al subsystem correspondiente.

## DB-BDEL-024
Purge será una intención diferenciable.

## DB-BDEL-025
Key podrá ser compuesta.

## DB-BDEL-026
Key podrá ser unique natural key cuando sea seguro.

## DB-BDEL-027
Non-unique key será rechazada por default en keyed delete.

## DB-BDEL-028
Key normalization utilizará Type System.

## DB-BDEL-029
Composite key normalization será determinista.

## DB-BDEL-030
Input podrá ser streaming.

## DB-BDEL-031
Streaming input no será materializado completamente.

## DB-BDEL-032
Backpressure será preservable.

## DB-BDEL-033
Batch size respetará parameter limits.

## DB-BDEL-034
Batch size respetará statement limits.

## DB-BDEL-035
Batch size respetará resource budgets.

## DB-BDEL-036
Deduplication no requerirá memoria ilimitada.

## DB-BDEL-037
Physical strategy será capability-driven.

## DB-BDEL-038
IN predicate será una estrategia, no la arquitectura.

## DB-BDEL-039
Temporary relation será opcional.

## DB-BDEL-040
Native delete strategy será extensible.

## DB-BDEL-041
Version será distinta de Capability.

## DB-BDEL-042
Foreign key metadata será considerada cuando esté disponible.

## DB-BDEL-043
NotObserved será distinto de Absent.

## DB-BDEL-044
Database cascade será distinto de ORM cascade.

## DB-BDEL-045
Bulk Delete no simulará ORM cascades.

## DB-BDEL-046
Bulk Delete no simulará orphanRemoval.

## DB-BDEL-047
Cascade row counts no serán inventados.

## DB-BDEL-048
Direct deleted count será distinto de total physical impact.

## DB-BDEL-049
Unknown cascade impact permanecerá UNKNOWN.

## DB-BDEL-050
Database Row Deleted será distinto de PHP Object Destroyed.

## DB-BDEL-051
External Bulk Delete no fabricará EntityState::REMOVED.

## DB-BDEL-052
REMOVED será reservado al lifecycle ORM real.

## DB-BDEL-053
Known deleted managed entities podrán marcarse externamente afectadas.

## DB-BDEL-054
DETACHED será distinto de REMOVED.

## DB-BDEL-055
Bulk Delete no hará EntityManager::clear() silenciosamente.

## DB-BDEL-056
Dirty managed entities no serán descartadas silenciosamente.

## DB-BDEL-057
Bulk Delete no reescribirá snapshots del UoW silenciosamente.

## DB-BDEL-058
Relationship graphs no serán reescritos automáticamente.

## DB-BDEL-059
Relationship stale state podrá notificarse.

## DB-BDEL-060
Database cascades podrán ampliar ORM stale scope.

## DB-BDEL-061
Optimistic locking reutilizará el subsystem existente.

## DB-BDEL-062
Zero affected no implicará automáticamente conflict.

## DB-BDEL-063
NOT_FOUND será distinto de CONFLICT.

## DB-BDEL-064
UNKNOWN permanecerá distinto de NOT_FOUND y CONFLICT.

## DB-BDEL-065
Pessimistic locking no será implícito.

## DB-BDEL-066
Retry safety será evaluada explícitamente.

## DB-BDEL-067
Delete final-state idempotency no implicará side-effect idempotency.

## DB-BDEL-068
UNKNOWN commit no será reintentado ciegamente.

## DB-BDEL-069
Transaction policy será explícita.

## DB-BDEL-070
USE_EXISTING no committeará transacción ajena.

## DB-BDEL-071
PER_BATCH podrá producir PARTIAL.

## DB-BDEL-072
Statement atomicity será distinta de operation atomicity.

## DB-BDEL-073
Cross-shard execution no fingirá global ACID.

## DB-BDEL-074
Savepoint será distinto de transaction.

## DB-BDEL-075
Bulk Delete será WRITE intent.

## DB-BDEL-076
Bulk Delete nunca irá a read replica.

## DB-BDEL-077
Routing ocurrirá antes de endpoint selection.

## DB-BDEL-078
UNKNOWN shard routing no significará broadcast.

## DB-BDEL-079
Broadcast delete requerirá intención explícita.

## DB-BDEL-080
Keyed deletes podrán agruparse por shard.

## DB-BDEL-081
Shard map generation podrá formar parte del plan.

## DB-BDEL-082
Tenant scope será estructural cuando el dominio lo requiera.

## DB-BDEL-083
Missing mandatory tenant context será error.

## DB-BDEL-084
Cross-tenant delete accidental será prevenido.

## DB-BDEL-085
Authorization ocurrirá antes de ejecución.

## DB-BDEL-086
Incapacidad de autorizar eficientemente no equivaldrá a autorización.

## DB-BDEL-087
Row-level authorization deberá formar parte del selection scope cuando sea traducible.

## DB-BDEL-088
Non-translatable authorization podrá impedir Bulk Delete.

## DB-BDEL-089
Bulk Delete no emitirá entity lifecycle events por fila.

## DB-BDEL-090
Bulk events serán agregados por operación/batch.

## DB-BDEL-091
Domain events individuales no serán fabricados.

## DB-BDEL-092
Audit será integración explícita.

## DB-BDEL-093
Delete reason no concederá authorization.

## DB-BDEL-094
Sensitive predicates no serán expuestos por default.

## DB-BDEL-095
Cache invalidation será semántica.

## DB-BDEL-096
Exact keys podrán producir exact invalidation.

## DB-BDEL-097
Predicate deletes podrán requerir broad invalidation.

## DB-BDEL-098
Unknown impact producirá invalidación conservadora.

## DB-BDEL-099
Entity Cache será invalidada tras commit confirmado.

## DB-BDEL-100
Cache tombstone será distinto de Soft Delete.

## DB-BDEL-101
Replica stale repopulation será considerada.

## DB-BDEL-102
Readable replica será distinta de cacheable replica.

## DB-BDEL-103
Relationship caches serán invalidadas cuando corresponda.

## DB-BDEL-104
Cache failure post-commit no revertirá delete.

## DB-BDEL-105
UNKNOWN commit podrá provocar invalidación conservadora.

## DB-BDEL-106
RETURNING será capability-driven.

## DB-BDEL-107
DELETE RETURNING no producirá entidades managed automáticamente.

## DB-BDEL-108
Returning grande será gobernado por resource policy.

## DB-BDEL-109
Streaming returning será soportable.

## DB-BDEL-110
Requested keys serán distintas de matched rows.

## DB-BDEL-111
Matched rows serán distintas de reported rows.

## DB-BDEL-112
Direct rows deleted serán distintas de cascade rows.

## DB-BDEL-113
Unknown counts no serán convertidos a cero.

## DB-BDEL-114
Per-key outcomes solo se afirmarán con evidencia suficiente.

## DB-BDEL-115
Result detail será configurable.

## DB-BDEL-116
Cancellation será soportada.

## DB-BDEL-117
Cancel requested será distinto de cancelled.

## DB-BDEL-118
Cancelled será distinto de rolled back.

## DB-BDEL-119
Committed batches no desaparecerán tras cancellation.

## DB-BDEL-120
Resource governance considerará cascade amplification.

## DB-BDEL-121
Estimated cascade impact será distinto de known cascade impact.

## DB-BDEL-122
Failure model distinguirá constraint failure.

## DB-BDEL-123
Constraint failure será distinto de planner failure.

## DB-BDEL-124
Trigger failure será execution failure.

## DB-BDEL-125
Post-commit ORM coherence failure no revertirá DB.

## DB-BDEL-126
Post-commit cache failure no revertirá DB.

## DB-BDEL-127
Result podrá separar DB outcome y secondary outcomes.

## DB-BDEL-128
Rollback confirmado será distinto de rollback desconocido.

## DB-BDEL-129
UNKNOWN persistirá cuando la realidad DB no pueda probarse.

## DB-BDEL-130
Persistent runtime state será operation-scoped.

## DB-BDEL-131
No habrá current deleted IDs static global.

## DB-BDEL-132
Tenant context no se filtrará entre requests.

## DB-BDEL-133
Transaction context no se filtrará entre requests.

## DB-BDEL-134
Returning buffers no se filtrarán entre requests.

## DB-BDEL-135
Cancellation state no se filtrará entre requests.

## DB-BDEL-136
FrankenPHP worker reuse será seguro.

## DB-BDEL-137
RoadRunner worker reuse será seguro.

## DB-BDEL-138
OpenSwoole worker reuse será seguro.

## DB-BDEL-139
Concurrent coroutines estarán aisladas.

## DB-BDEL-140
Immutable metadata podrá compartirse.

## DB-BDEL-141
BulkDeletePlan será immutable.

## DB-BDEL-142
Planner será distinto de Runner.

## DB-BDEL-143
Runner será distinto de Executor.

## DB-BDEL-144
Bulk Delete no conocerá PDO.

## DB-BDEL-145
Bulk Delete no conocerá runtime concreto.

## DB-BDEL-146
Bulk Delete no creará un segundo ORM.

## DB-BDEL-147
Bulk Delete no creará un segundo Transaction System.

## DB-BDEL-148
Bulk Delete no creará un segundo Cache System.

## DB-BDEL-149
Bulk Delete no creará un segundo Authorization System.

## DB-BDEL-150
Bulk Delete no creará un segundo Sharding System.

## DB-BDEL-151
Bulk Delete será explainable.

## DB-BDEL-152
Scope safety será explainable.

## DB-BDEL-153
Cascade risk será explainable.

## DB-BDEL-154
Transaction atomicity será explainable.

## DB-BDEL-155
Retry safety será explainable.

## DB-BDEL-156
ORM coherence consequences serán explainable.

## DB-BDEL-157
Cache invalidation precision será explainable.

## DB-BDEL-158
Routing será explainable.

## DB-BDEL-159
Telemetry tendrá bounded cardinality.

## DB-BDEL-160
Deleted identifiers no serán metric labels.

## DB-BDEL-161
Raw SQL no será metric label.

## DB-BDEL-162
Raw predicates no serán metric labels.

## DB-BDEL-163
Database continuará siendo autoridad del estado persistente.

## DB-BDEL-164
IdentityMap continuará siendo autoridad de identidad managed dentro del scope.

## DB-BDEL-165
Bulk Delete nunca convertirá incertidumbre en certeza.

---

# 244. Modelo formal

Sea:

```text
R
=
conjunto de filas del target
```

y:

```text
P(r)
=
predicate de selección
```

Entonces:

```text
D
=
{ r ∈ R | P(r) }
```

representa el conjunto lógico objetivo.

La operación intenta:

```text
R'
=
R \ D
```

sujeta a:

```text
constraints
transactions
concurrency
routing
database cascades
authorization
```

---

# 245. Keyed model

Sea:

```text
K
=
{k1, k2, ..., kn}
```

y:

```text
Resolve(ki)
```

la selección de la fila correspondiente.

Entonces:

```text
D
=
⋃ Resolve(ki)
```

bajo la semántica de key configurada.

---

# 246. Non-unique key problem

Si:

```text
|Resolve(ki)| > 1
```

la operación ya no tiene semántica:

```text
one key
→ one row
```

por lo que deberá declararse explícitamente o rechazarse.

---

# 247. Transaction outcome

Para batches:

```text
B1, B2, ..., Bn
```

con:

```text
PER_BATCH
```

la atomicidad es:

```text
AtomicityScope(Bi)
```

no:

```text
AtomicityScope(B1...Bn)
```

---

# 248. ORM model

Después de una eliminación externa:

```text
DBPresence(E)
=
false
```

mientras:

```text
IdentityMapContains(E)
=
true
```

puede seguir siendo cierto.

Esto no es una contradicción interna del ORM si está explícitamente modelado como:

```text
ExternalPersistenceChange
```

---

# 249. Cache model

Tras commit confirmado:

```text
DeletionSemanticChange
↓
Dependency Resolution
↓
Invalidation Plan
↓
Entity/Result/Relationship Cache Invalidation
```

y, cuando sea necesario:

```text
Deletion Tombstone
```

para impedir stale resurrection.

---

# 250. Arquitectura final

```text
                       Bulk Delete API
                              │
                              ▼
                     BulkDeleteRequest
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              Predicate Delete     Keyed Delete
                    │                   │
                    └─────────┬─────────┘
                              ▼
                         Normalizer
                              │
                              ▼
                     BulkDeletePlanner
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
     Safety              Dependencies            Routing
        │                     │                     │
        ├─────────────────────┼─────────────────────┤
        ▼                     ▼                     ▼
     Strategy              Batching             Locking
        │                     │                     │
        ├─────────────────────┼─────────────────────┤
        ▼                     ▼                     ▼
   Transaction             Returning            Resources
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
                       BulkDeletePlan
                              │
                              ▼
                       BulkDeleteRunner
                              │
                              ▼
                      DeleteQueryModel
                              │
                              ▼
                         Query Engine
                              │
                              ▼
                           Compiler
                              │
                              ▼
                           Executor
                              │
                              ▼
                    Transaction Outcome
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
        ORM Coherence    Cache Invalidation   Events
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                      BulkDeleteResult
```

---

# 251. Regla maestra final

> **VoltStack Bulk Delete deberá priorizar la verdad sobre la conveniencia: eliminar una fila directamente en la base de datos no significa que el ORM haya ejecutado `remove()`, que sus relaciones hayan sido reconciliadas, que sus lifecycle events hayan ocurrido o que todos sus efectos indirectos sean conocidos.**

Por tanto:

```text
Bulk Delete
≠
EntityManager::remove() × N
```

```text
Bulk Delete
≠
Soft Delete
```

```text
Bulk Delete
≠
ORM Cascade Remove
```

```text
Database ON DELETE CASCADE
≠
ORM Cascade
```

```text
Database Row Deleted
≠
PHP Object Destroyed
```

```text
Database Row Deleted
≠
EntityState::REMOVED
```

```text
DETACHED
≠
REMOVED
```

```text
Direct Rows Deleted
≠
Total Rows Deleted By Cascades
```

```text
Empty Key Set
≠
Delete All
```

```text
DELETE All
≠
TRUNCATE
```

```text
Zero Rows Affected
≠
Optimistic Conflict
```

```text
Final-State Idempotency
≠
Replay Safety
```

```text
Cancellation
≠
Rollback
```

```text
Cross-Shard Delete
≠
Global ACID
```

y:

```text
UNKNOWN
≠
FAILED
≠
ROLLED_BACK
≠
SAFE_TO_RETRY
```

---

# 252. Resultado arquitectónico del bloque Bulk

Con los documentos 203–205, VoltStack dispondrá de una arquitectura coherente:

```text
                 Database Bulk Mutation System

                          Bulk
                            │
           ┌────────────────┼────────────────┐
           │                │                │
           ▼                ▼                ▼
        INSERT            UPDATE           DELETE
           │                │                │
        creates           mutates          removes
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                      Common Layer
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Batching       Transactions     Routing
             │              │              │
             ├──────────────┼──────────────┤
             ▼              ▼              ▼
          Resources      Telemetry       Runtime
                            │
                            ▼
                       Query Engine
                            │
                            ▼
                         Compiler
                            │
                            ▼
                         Executor
                            │
                            ▼
                         Database
```

sin romper las fronteras:

```text
Bulk Layer
≠
ORM
≠
UnitOfWork
≠
Query Engine
≠
Compiler
≠
Driver
```

---

# 253. Siguiente documento

```text
206_DATABASE_IMPORT_SYSTEM.md
```

El siguiente documento definirá la arquitectura de importación masiva de información:

```text
Database Import System
├── Import Sources
├── CSV
├── JSON
├── NDJSON
├── streams
├── iterators
├── files
├── external adapters
├── schema mapping
├── column mapping
├── type conversion
├── validation
├── normalization
├── transformations
├── duplicate handling
├── error policies
├── checkpoints
├── resumability
├── batching
├── Bulk Insert integration
├── Bulk Update integration
├── Upsert strategies
├── transaction policies
├── tenant routing
├── shard routing
├── memory limits
├── backpressure
├── cancellation
├── progress
├── telemetry
└── persistent-runtime safety
```

bajo la regla:

> **Importar datos no será equivalente a insertar filas: el Import System será un pipeline gobernado de lectura, decodificación, normalización, validación, transformación, routing y persistencia que utilizará los sistemas Bulk como mecanismos de escritura, sin duplicar sus responsabilidades.**