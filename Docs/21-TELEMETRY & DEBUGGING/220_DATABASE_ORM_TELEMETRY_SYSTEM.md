# 220_DATABASE_ORM_TELEMETRY_SYSTEM.md

# VoltStack Quantum Database
## ORM Telemetry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 220 — ORM Telemetry System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md`  
**Siguiente documento:** `221_DATABASE_QUERY_PROFILER_SYSTEM.md`

---

# 1. Propósito

`ORM Telemetry System` define cómo VoltStack observará el comportamiento interno del ORM sin convertir la telemetría en una dependencia funcional de:

```text
EntityManager
IdentityMap
UnitOfWork
Change Tracking
Hydration
Persistence Engine
Relationship Loading
Repositories
Model API
```

El sistema deberá permitir responder preguntas como:

```text
¿Cuántas entidades están siendo administradas?

¿Cuánto crece el IdentityMap?

¿Cuántas entidades fueron hidratadas?

¿Cuántas fueron reutilizadas desde IdentityMap?

¿Cuántas entidades NEW, DIRTY o REMOVED existen?

¿Cuánto tarda el cálculo de ChangeSets?

¿Cuánto tarda flush()?

¿Cuántas operaciones INSERT / UPDATE / DELETE produce un flush?

¿Cuánto tiempo se consume en hydration?

¿Cuántas relaciones se cargan?

¿Cuántas cargas lazy ocurren?

¿Existen patrones N+1?

¿Cuánta memoria consume el ORM?

¿El EntityManager quedó TAINTED?

¿El ORM está acumulando estado entre requests?

¿Cuánto overhead introduce el ORM sobre las queries físicas?
```

La regla central será:

> **ORM Telemetry observa el estado, comportamiento y costo del ORM, pero nunca deberá modificar entidades, IdentityMap, UnitOfWork, ChangeSets, Hydration Plans, Persistence Plans ni EntityManager state.**

Formalmente:

```text
ORM
↓
State + Operations
↓
Telemetry
↓
Observe / Measure / Correlate
```

Nunca:

```text
Telemetry
↓
persist()
remove()
flush()
clear()
hydrate()
detach()
```

---

# 2. Posición arquitectónica

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry
├── ORM Telemetry              ← este documento
├── Query Profiler
├── Slow Query Detection
├── N+1 Telemetry
├── Debug Information
└── Developer Debug Toolbar
```

ORM Telemetry estará por encima de:

```text
EntityManager
UnitOfWork
IdentityMap
Hydrator
Persistence Engine
Relationship System
```

y se correlacionará hacia abajo con:

```text
Query Telemetry
Transaction Telemetry
Connection Telemetry
```

---

# 3. Distinciones fundamentales

VoltStack deberá preservar:

```text
ORM Telemetry
≠
ORM

ORM Telemetry
≠
Entity Lifecycle Event System

ORM Telemetry
≠
Persistence Event System

ORM Telemetry
≠
Query Telemetry

ORM Telemetry
≠
Profiler

EntityManager
≠
UnitOfWork

UnitOfWork
≠
Transaction

IdentityMap
≠
Entity Cache

Hydration
≠
Persistence

Entity State
≠
Database State

Managed Entity Count
≠
Database Row Count

Flush
≠
Commit

persist()
≠
INSERT

remove()
≠
DELETE

Dirty Entity
≠
UPDATE executed
```

---

# 4. Objetivos

El sistema deberá observar:

```text
EntityManager lifecycle
IdentityMap size
IdentityMap reuse
Entity states
UnitOfWork state
ChangeSet calculation
Persistence planning
Flush lifecycle
Hydration
Relationship loading
Repository operations
Model API operations
Entity lifecycle costs
Optimistic locking conflicts
Persistence consistency
EntityManager tainting
Memory behavior
Persistent runtime isolation
```

---

# 5. No objetivos

ORM Telemetry no deberá:

```text
insert entities
update entities
delete entities
hydrate entities
resolve relationships
change ChangeSets
clear IdentityMap
detach entities
begin transactions
retry transactions
select database connections
rewrite queries
```

---

# 6. ORMOperationId

Una operación ORM lógica tendrá:

```php
final readonly class ORMOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
find()
findOneBy()
repository query
model save()
flush()
refresh()
relationship load
lazy load
eager load
```

---

# 7. ORMOperationId ≠ QueryId

Una operación ORM puede producir:

```text
0 queries
1 query
N queries
```

Ejemplo:

```php
$user = $entityManager->find(User::class, 10);
```

si está en IdentityMap:

```text
ORMOperation = 1
QueryExecution = 0
```

---

# 8. ORM operation hierarchy

```text
ORMOperation
│
├── Metadata
├── IdentityMap
├── UnitOfWork
├── Hydration
├── Persistence
├── Relationship Loading
└── Query Operations
```

---

# 9. EntityManagerScopeId

Cada EntityManager scoped tendrá:

```text
EntityManagerScopeId
```

para correlación.

No deberá confundirse con:

```text
RequestId
TransactionId
ORMOperationId
```

---

# 10. EntityManager telemetry context

```php
final readonly class ORMTelemetryContext
{
    public function __construct(
        public EntityManagerScopeId $entityManagerScope,
        public ?ORMOperationId $operationId,
        public ?TransactionId $transactionId,
        public ?PersistenceDomainId $domainId,
        public ?TenantId $tenantId,
        public ?ShardId $shardId,
        public ORMTelemetryPolicy $policy,
    ) {}
}
```

---

# 11. EntityManager lifecycle

Conceptualmente:

```text
OPEN
↓
ACTIVE
↓
TAINTED?
↓
CLEAR?
↓
CLOSED
```

---

# 12. EntityManager state

De acuerdo con la arquitectura previa:

```text
OPEN
TAINTED
CLOSED
```

son estados relevantes.

Telemetry deberá observarlos.

---

# 13. EntityManager opened

Podrá registrar:

```text
db.orm.entity_manager.opened
```

como counter opcional.

---

# 14. EntityManager clear

Deberá observar:

```text
clear count
entities released
reason
```

sin iterar innecesariamente sobre cada entidad solo para telemetry.

---

# 15. Clear reason

Podrá correlacionarse con:

```text
EXPLICIT
MEMORY_GOVERNANCE
ERROR_RECOVERY
REQUEST_END
TAINT_RECOVERY
```

---

# 16. EntityManager close

Telemetry podrá registrar:

```text
EntityManager lifetime
operations performed
flushes
managed peak
taint events
```

---

# 17. EntityManager taint

Estado crítico:

```text
EntityManager = TAINTED
```

podrá producir:

```text
db.orm.entity_manager.tainted
```

---

# 18. Taint reason

```php
enum ORMTaintReason
{
    case UNKNOWN_TRANSACTION_OUTCOME;
    case PERSISTENCE_UNCERTAINTY;
    case INTERNAL_STATE_INCONSISTENCY;
    case HYDRATION_FAILURE;
    case OTHER;
}
```

---

# 19. TAINTED ≠ FAILED REQUEST

Un EntityManager tainted no implica necesariamente:

```text
whole application request failed
```

pero sí indica que ese context ORM no debe tratarse como confiable.

---

# 20. IdentityMap telemetry

El sistema deberá observar:

```text
current managed count
peak managed count
lookup count
reuse count
insertions
removals
clears
```

---

# 21. IdentityMap size

Métrica conceptual:

```text
db.orm.identity_map.size
```

---

# 22. IdentityMap size ≠ DB rows

```text
IdentityMapSize
≠
DatabaseRowCount
```

Representa objetos managed en el scope actual.

---

# 23. IdentityMap hit

Cuando se solicita:

```text
EntityType + Identifier + Context
```

y ya existe una instancia managed:

```text
IdentityMapReuse
```

---

# 24. IdentityMap hit ≠ cache hit

Regla:

```text
IdentityMap Hit
≠
Entity Cache Hit
≠
Result Cache Hit
```

---

# 25. IdentityMap telemetry metrics

```text
db.orm.identity_map.lookup
db.orm.identity_map.reuse
db.orm.identity_map.size
db.orm.identity_map.peak
```

---

# 26. Hit ratio caution

Podría calcularse:

```text
ReuseRatio =
Reuse
/
Lookup
```

pero no siempre implica rendimiento mejor o peor.

---

# 27. IdentityMap growth

Crecimiento sostenido puede ser importante en:

```text
Chunk Processing
Lazy Collection
Import
Long-running jobs
```

---

# 28. Growth ≠ leak

Un IdentityMap grande puede ser intencional.

Por tanto:

```text
LargeIdentityMap
≠
MemoryLeak
```

---

# 29. Memory pressure diagnostics

La telemetría podrá correlacionar:

```text
managed entities
IdentityMap size
UoW size
memory usage
processing mode
```

para advertir presión.

---

# 30. UnitOfWork telemetry

Deberá observar:

```text
NEW
MANAGED
DIRTY
REMOVED
DETACHED
```

según la representación aplicable.

---

# 31. UoW summary

```php
final readonly class UnitOfWorkTelemetrySnapshot
{
    public function __construct(
        public int $new,
        public int $managed,
        public int $dirty,
        public int $removed,
        public int $pendingRelationshipChanges,
    ) {}
}
```

---

# 32. Snapshot timing

Podrá tomarse:

```text
before flush
after change detection
after flush
after clear
```

---

# 33. Snapshot ≠ live reference

Nunca deberá entregar:

```text
UnitOfWork object
```

al provider.

---

# 34. Managed count duplication

Debe evitarse contar dos veces:

```text
IdentityMap managed count
```

y:

```text
UnitOfWork managed count
```

si tienen la misma semántica.

---

# 35. Canonical ownership

Se deberá documentar:

```text
IdentityMap
→ canonical managed identity count

UnitOfWork
→ pending persistence work
```

---

# 36. Change Tracking telemetry

Podrá medir:

```text
change detection duration
entities inspected
change sets produced
fields changed
```

---

# 37. ChangeSet count

```text
db.orm.change_sets
```

podrá registrar número agregado.

---

# 38. Fields changed

No deberán exportarse valores.

Opcionalmente:

```text
changed_field_count
```

o nombres safe bajo diagnostic policy.

---

# 39. ChangeSet telemetry ≠ ChangeSet mutation

El sistema deberá trabajar sobre:

```text
summary
```

no cambiar el contenido.

---

# 40. Change detection timing

```text
db.orm.change_detection.duration
```

permitirá identificar costes de:

```text
large UnitOfWork
expensive equality comparison
complex Value Objects
```

---

# 41. Dirty checking strategies

Si VoltStack soporta múltiples:

```text
snapshot
explicit
notification
deferred
custom
```

Telemetry podrá registrar estrategia bounded.

---

# 42. Strategy ≠ EntityType

No deberá crear series métricas por clase/entidad sin policy.

---

# 43. Flush telemetry

`flush()` será una de las operaciones ORM más importantes.

Se observará:

```text
flush count
flush duration
entities considered
change sets
persistence operations
queries produced
transaction correlation
final ORM consistency
```

---

# 44. Flush identity

Podrá reutilizar:

```text
PersistenceFlushId
```

como correlación canonical.

---

# 45. Flush ≠ commit

Siempre:

```text
ORM Flush
≠
Transaction Commit
```

---

# 46. Flush timing

Podrá descomponerse:

```text
Change Detection
Persistence Planning
Lifecycle Hooks
Query Generation
Execution
Reconciliation
```

---

# 47. Flush duration metric

```text
db.orm.flush.duration
```

será canonical para la operación ORM completa.

---

# 48. Persistence duration

Una submedición podrá pertenecer al Persistence subsystem.

ORM Telemetry podrá correlacionarla.

---

# 49. Flush summary

```php
final readonly class ORMFlushTelemetrySummary
{
    public function __construct(
        public PersistenceFlushId $flushId,
        public int $newEntities,
        public int $dirtyEntities,
        public int $removedEntities,
        public int $insertOperations,
        public int $updateOperations,
        public int $deleteOperations,
        public int $queryOperations,
        public Duration $duration,
        public ORMFlushOutcome $outcome,
    ) {}
}
```

---

# 50. Flush outcome

```php
enum ORMFlushOutcome
{
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 51. UNKNOWN

Si el transaction outcome queda incierto:

```text
Flush execution may be completed
Database durability unknown
```

ORM telemetry deberá reflejar uncertainty.

---

# 52. Flush completion ≠ durable state

Incluso:

```text
ORMFlushOutcome::COMPLETED
```

no necesariamente implica:

```text
Transaction COMMITTED
```

si el transaction owner es externo.

---

# 53. Persistence operation telemetry

Podrá observar:

```text
insert operation count
update operation count
delete operation count
batch count
```

sin duplicar Query Telemetry.

---

# 54. Entity operations ≠ statements

Ejemplo:

```text
100 entities inserted
↓
1 multi-row INSERT
```

Por tanto:

```text
EntityPersistenceOperations
≠
Statements
```

---

# 55. Hydration telemetry

El ORM deberá observar:

```text
result rows consumed
entities hydrated
entities reused
partial entities
DTO/projection hydration
hydration duration
```

---

# 56. HydrationOperationId

Podrá existir:

```php
final readonly class HydrationOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 57. Query correlation

```text
QueryId
↓
HydrationOperationId
↓
Entities
```

sin convertir entity IDs en telemetry dimensions.

---

# 58. Hydration duration

```text
db.orm.hydration.duration
```

o:

```text
db.hydration.duration
```

deberá tener un solo canonical owner.

---

# 59. Recommended ownership

El subsystem Hydration posee la medición precisa.

ORM Telemetry expone/agrupa la vista ORM.

---

# 60. Rows ≠ hydrated entities

JOIN example:

```text
100 physical rows
↓
20 User entities
↓
100 Order relationship rows
```

Por tanto:

```text
rows_consumed
≠
entities_hydrated
```

---

# 61. IdentityMap reuse during hydration

Cuando una entidad ya existe:

```text
row
↓
IdentityMap
↓
existing object reused
```

deberá registrarse:

```text
entity_reused
```

no:

```text
new_entity_hydrated
```

---

# 62. Hydration source

Podrá observarse:

```text
DATABASE_RESULT
ENTITY_CACHE
RESULT_CACHE
CUSTOM
```

---

# 63. Hydration source metric

Será bounded.

---

# 64. Partial entity telemetry

Deberá registrar:

```text
partial_entity_count
```

sin exponer automáticamente `LoadedFieldMask`.

---

# 65. Partial entity ≠ broken entity

Puede ser resultado intencional.

Pero deberá ser visible para profiling.

---

# 66. Projection telemetry

Se deberán distinguir:

```text
ENTITY
SCALAR
TUPLE
DTO
PROJECTION
ARRAY
```

como result/hydration shape.

---

# 67. Shape cardinality

Al ser enum bounded, puede utilizarse en métricas.

---

# 68. Hydration memory

Para datasets grandes:

```text
Hydrated objects
+
IdentityMap retention
+
relationships
```

pueden dominar memoria.

---

# 69. Memory telemetry

Podrán medirse:

```text
memory before ORM operation
memory after ORM operation
peak delta
```

solo cuando policy/runtime lo permita.

---

# 70. PHP memory caveat

```text
memory_get_usage()
```

no representa necesariamente memoria física real del proceso.

Telemetry deberá documentar:

```text
PHP-observed memory
```

no:

```text
exact process RSS
```

---

# 71. Memory measurement cost

No deberá hacerse en cada entidad.

Preferir:

```text
operation boundary
chunk boundary
flush boundary
```

---

# 72. Entity state telemetry

Podrá observarse el número de transiciones:

```text
NEW → MANAGED
MANAGED → DIRTY
DIRTY → MANAGED
MANAGED → REMOVED
REMOVED → DETACHED
```

---

# 73. Per-entity event explosion

No emitir telemetría individual por cada transición en producción normal.

---

# 74. Aggregated transitions

Ejemplo:

```text
db.orm.entity.transition{
    from="managed",
    to="dirty"
}
```

si el volumen lo permite.

---

# 75. EntityType dimension

`EntityType` puede ser bounded en una aplicación.

Pero deberá ser configurable debido a aplicaciones con:

```text
hundreds/thousands of entity types
```

---

# 76. Entity ID

Nunca como metric label.

---

# 77. Lifecycle correlation

Entity Lifecycle Event System podrá alimentar métricas agregadas.

Pero:

```text
Entity Lifecycle Event
≠
ORM Telemetry Signal
```

---

# 78. Repository telemetry

Operaciones como:

```php
$userRepository->find(10);
$userRepository->findActive();
```

podrán tener:

```text
ORMOperationId
origin=REPOSITORY
```

---

# 79. Repository method names

No deberán utilizarse automáticamente como metric labels si son dinámicos/no bounded.

---

# 80. Registered repository operation names

Si VoltStack ofrece un registry bounded:

```text
find
find_one
exists
count
```

podrán utilizarse.

---

# 81. Model API telemetry

Ejemplos:

```php
User::find(10);
$user->save();
$user->delete();
```

deberán converger al mismo motor de telemetry ORM.

---

# 82. Active Record telemetry ≠ second ORM telemetry

No habrá:

```text
ModelTelemetry
```

completamente separado del Data Mapper.

Ambos convergerán.

---

# 83. API origin

Podrá registrarse:

```php
enum ORMApiOrigin
{
    case ENTITY_MANAGER;
    case REPOSITORY;
    case MODEL_API;
    case RELATIONSHIP;
    case INTERNAL;
}
```

---

# 84. API origin ≠ persistence engine

Es una dimensión DX/diagnostic.

---

# 85. Relationship telemetry

Deberá observar:

```text
relationship load count
strategy
entities loaded
query count
batch size
partial coverage
lazy load count
eager load count
batch load count
```

---

# 86. Relationship identity

Podrá usar:

```text
RelationshipId
```

en traces/diagnostics.

No como metric label por default.

---

# 87. Relationship load strategies

```php
enum RelationshipLoadStrategyTelemetry
{
    case JOIN;
    case SELECT_IN;
    case BATCH;
    case LAZY;
    case EXPLICIT;
    case CACHE;
    case CUSTOM;
}
```

---

# 88. Lazy load telemetry

Cada lazy load podrá incrementar:

```text
db.orm.relationship.lazy_loads
```

---

# 89. Lazy load ≠ N+1

Un lazy load aislado puede ser correcto.

Por tanto:

```text
LazyLoad
≠
NPlusOne
```

---

# 90. N+1 correlation

ORM Telemetry deberá proveer:

```text
relationship identity
root operation
query correlation
load count
```

a:

```text
223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
```

---

# 91. Eager loading telemetry

Deberá observar:

```text
strategy chosen
root entities
related entities
query count
coverage
```

---

# 92. Eager load ≠ JOIN

Una eager load puede utilizar:

```text
JOIN
SELECT_IN
BATCH
```

---

# 93. Batch relationship telemetry

Podrá medir:

```text
owners batched
keys loaded
queries emitted
batch duration
```

---

# 94. Relationship coverage

Podrá representar:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 95. PARTIAL ≠ empty

Importante:

```text
PARTIAL collection
≠
complete empty collection
```

Telemetry deberá preservar esa semántica.

---

# 96. ORM query amplification

Una operación ORM puede producir:

```text
1 repository call
↓
12 SQL queries
```

Esto podrá calcularse como:

```text
QueryAmplification =
PhysicalQueryCount
/
LogicalORMOperationCount
```

---

# 97. Query amplification ≠ N+1 proof

Puede existir una operación legítima con múltiples queries.

Por tanto:

```text
HighAmplification
≠
N+1
```

---

# 98. Persistence telemetry correlation

ORM flush podrá correlacionarse con:

```text
PersistenceFlushId
PersistenceOperationId
TransactionId
QueryIds
```

---

# 99. Correlation graph

```text
ORMOperationId
     │
     ▼
PersistenceFlushId
     │
     ├── PersistenceOperation
     │       │
     │       ▼
     │     QueryId
     │
     ▼
TransactionId
```

---

# 100. Optimistic locking telemetry

Podrá observar:

```text
optimistic lock attempts
conflicts
retries by application
```

---

# 101. Conflict ≠ deadlock

```text
OptimisticLockConflict
≠
DatabaseDeadlock
```

---

# 102. Optimistic conflict metric

```text
db.orm.optimistic_lock.conflicts
```

con dimensiones bounded.

---

# 103. Entity version values

Nunca exportar versiones arbitrarias como labels.

---

# 104. Pessimistic locking correlation

ORM operation podrá indicar:

```text
lock_mode=PESSIMISTIC_WRITE
```

si la Query/Transaction telemetry ya conoce la semántica.

---

# 105. Persistence consistency telemetry

De acuerdo con 134:

```text
CONSISTENT
STALE
UNCERTAIN
INCONSISTENT
TAINTED
UNKNOWN
```

deberá poder observarse.

---

# 106. ORM consistency metrics

```text
db.orm.consistency.outcome
```

podrá utilizar el estado como dimensión bounded.

---

# 107. UNKNOWN preservation

No convertir:

```text
UNKNOWN
```

a:

```text
INCONSISTENT
```

ni:

```text
CONSISTENT
```

sin evidencia.

---

# 108. TAINTED EntityManager

Deberá considerarse evento diagnóstico de alta importancia.

---

# 109. Taint correlation

Ejemplo:

```text
TransactionOutcomeUnknown
↓
PersistenceConsistencyUnknown
↓
EntityManagerTainted
```

---

# 110. Taint telemetry

Debe conservar:

```text
reason
transaction correlation
flush correlation
scope ID
```

sin guardar entidades.

---

# 111. Clear after taint

Si el framework ejecuta:

```text
clear/close
```

por policy, telemetry observará la recuperación.

No la decidirá.

---

# 112. Batch persistence telemetry

Para:

```text
1000 entities
```

podrá registrar:

```text
batch count
entities per batch
statements per batch
duration
```

---

# 113. ORM batch ≠ Bulk API

```text
ORM Batch Persistence
≠
Bulk Insert/Update/Delete
```

como ya se definió.

---

# 114. Batch metrics

Podrán ser:

```text
db.orm.persistence.batch.size
db.orm.persistence.batch.duration
```

---

# 115. Entity factory interaction

`make()`:

```text
Factory
↓
Entity created
```

no deberá generar persistence telemetry ORM.

---

# 116. Factory create()

Si ejecuta persist/flush mediante ORM:

```text
Factory
↓
ORM
↓
Telemetry
```

sí será observable.

---

# 117. Seeder/Fixture

Solo generarán ORM telemetry cuando pasen por ORM.

---

# 118. Import modes

Import directo por Bulk Engine:

```text
no per-entity ORM telemetry
```

Import ORM_ENTITY:

```text
ORM telemetry applies
```

---

# 119. Lazy Collection integration

Una Lazy Collection de entidades puede provocar:

```text
repeated query
hydration
IdentityMap growth
relationship loading
```

---

# 120. Lazy iteration telemetry

ORM Telemetry deberá agregarse por:

```text
LazyTraversalId
```

cuando exista.

---

# 121. Chunk integration

Por cada chunk podrán observarse:

```text
entities hydrated
managed count delta
flushes
clears/detaches
```

---

# 122. Constant-memory illusion

Regla:

```text
Chunked Reads
≠
Constant ORM Memory
```

si IdentityMap retiene entidades.

---

# 123. Diagnostic candidate

Si:

```text
chunks processed ↑
IdentityMap size ↑ continuously
```

podrá generarse:

```text
ORM_MANAGED_STATE_GROWTH
```

como diagnóstico.

---

# 124. Growth diagnostic ≠ auto clear

Telemetry no ejecutará:

```php
$entityManager->clear();
```

---

# 125. Memory governance integration

El sistema de Resource Governance podrá decidir acciones.

Telemetry solo proporciona evidencia.

---

# 126. Metadata telemetry

Podrá observar:

```text
metadata cache hit/miss
metadata compilation duration
mapping resolution
```

pero los detalles fuertes se desarrollarán en Performance/Cache systems.

---

# 127. Metadata cache hit ≠ IdentityMap hit

Distinción obligatoria.

---

# 128. Mapping telemetry

En diagnostic mode:

```text
attribute mapping
entity metadata load
relationship metadata resolution
```

podrá medirse.

---

# 129. Mapping errors

Podrán clasificarse:

```text
MAPPING_ERROR
METADATA_ERROR
TYPE_MAPPING_ERROR
```

---

# 130. ORM failure taxonomy

```php
enum ORMFailureCategory
{
    case ENTITY_STATE;
    case METADATA;
    case MAPPING;
    case HYDRATION;
    case CHANGE_TRACKING;
    case PERSISTENCE_PLANNING;
    case PERSISTENCE_EXECUTION;
    case RELATIONSHIP;
    case OPTIMISTIC_LOCK;
    case CONSISTENCY;
    case MEMORY_PRESSURE;
    case ENTITY_MANAGER_TAINTED;
    case INTERNAL;
    case UNKNOWN;
}
```

---

# 131. ORM failure ≠ query failure

Query failure puede causar ORM failure.

Pero:

```text
ORMFailure
```

puede ocurrir antes de ejecutar cualquier SQL.

---

# 132. Failure stage

```php
enum ORMFailureStage
{
    case METADATA_RESOLUTION;
    case ENTITY_REGISTRATION;
    case CHANGE_DETECTION;
    case FLUSH_PLANNING;
    case HYDRATION;
    case RELATIONSHIP_LOADING;
    case PERSISTENCE;
    case RECONCILIATION;
    case CLEAR;
    case CLOSE;
}
```

---

# 133. Metrics principales

```text
db.orm.operations
db.orm.operation.duration

db.orm.entity_manager.active
db.orm.entity_manager.tainted
db.orm.entity_manager.clear

db.orm.identity_map.size
db.orm.identity_map.peak
db.orm.identity_map.lookup
db.orm.identity_map.reuse

db.orm.uow.new
db.orm.uow.dirty
db.orm.uow.removed

db.orm.change_detection.duration
db.orm.change_sets

db.orm.flush.count
db.orm.flush.duration
db.orm.flush.failures

db.orm.persistence.inserts
db.orm.persistence.updates
db.orm.persistence.deletes

db.orm.hydration.duration
db.orm.hydration.entities
db.orm.hydration.reused

db.orm.relationship.loads
db.orm.relationship.lazy_loads
db.orm.relationship.eager_loads
db.orm.relationship.batch_loads

db.orm.optimistic_lock.conflicts
db.orm.consistency.outcomes
```

---

# 134. Metric labels bounded

Permitidos potencialmente:

```text
orm.operation
orm.origin
orm.outcome
entity.state
hydration.shape
hydration.source
relationship.strategy
consistency.status
failure.category
```

---

# 135. Dangerous labels

Evitar:

```text
entity.id
tenant.id
query.id
transaction.id
relationship.id
repository method
raw class name
property name
```

sin policy explícita.

---

# 136. EntityType labels

Podrán habilitarse:

```text
orm.entity_type
```

solo si el registry demuestra cardinalidad bounded.

---

# 137. FQCN

No deberá ser external identity preferida.

Usar:

```text
stable EntityTypeId
```

cuando sea posible.

---

# 138. ORM tracing

Spans potenciales:

```text
db.orm.operation
db.orm.flush
db.orm.hydration
db.orm.relationship.load
```

---

# 139. Span explosion protection

No crear:

```text
one span per entity
```

por default.

---

# 140. Flush span

Ejemplo:

```text
db.orm.flush
├── change_detection
├── persistence_planning
├── transaction
│   ├── query
│   ├── query
│   └── query
└── reconciliation
```

---

# 141. Hydration span

Podrá ser child de Query span:

```text
db.query
└── db.orm.hydration
```

o phase event dependiendo de policy.

---

# 142. Relationship load span

```text
db.orm.relationship.load
└── db.query
```

podrá ayudar a detectar N+1.

---

# 143. EntityManager lifetime span

No deberá mantenerse abierto durante requests/worker demasiado largos salvo profiling explícito.

---

# 144. Operation spans preferred

Preferir spans por:

```text
flush
find
relationship load
hydration batch
```

---

# 145. Repository operation traces

Podrán representar:

```text
db.orm.repository
```

solo en detailed mode.

---

# 146. Model API traces

Podrán usar mismo `db.orm.operation` con:

```text
origin=model_api
```

---

# 147. Telemetry source ownership

Se deberá definir un dueño canonical por medición:

```text
IdentityMap metrics
→ IdentityMap

UoW metrics
→ UnitOfWork

Hydration metrics
→ Hydrator

Flush metrics
→ EntityManager/Flush Coordinator

Persistence counts
→ Persistence Engine

Relationship load metrics
→ Relationship Loading System
```

---

# 148. No double counting

Ejemplo:

```text
PersistenceInsertExecuted
```

no deberá incrementar múltiples veces:

```text
db.orm.persistence.inserts
```

desde Event Bridge y direct instrumentation.

---

# 149. ORMTelemetryRecord

```php
final readonly class ORMOperationTelemetryRecord
{
    public function __construct(
        public ORMOperationId $operationId,
        public ORMOperationType $operation,
        public ORMApiOrigin $origin,
        public ORMOperationOutcome $outcome,
        public Duration $duration,
        public ORMTelemetrySummary $summary,
    ) {}
}
```

---

# 150. ORMOperationType

```php
enum ORMOperationType
{
    case FIND;
    case FIND_MANY;
    case QUERY;
    case PERSIST;
    case REMOVE;
    case FLUSH;
    case REFRESH;
    case RELATIONSHIP_LOAD;
    case HYDRATION;
    case CLEAR;
    case CLOSE;
    case CUSTOM;
}
```

---

# 151. persist operation semantics

`persist()` telemetry deberá representar:

```text
registration/scheduling
```

No:

```text
INSERT
```

---

# 152. remove operation semantics

`remove()` telemetry representa:

```text
remove scheduling
```

No:

```text
DELETE executed
```

---

# 153. flush operation semantics

`flush()` puede producir:

```text
0 queries
1 query
N queries
```

---

# 154. Empty flush

Un flush sin cambios:

```text
flush()
↓
no pending work
```

deberá ser observable sin fingir SQL.

---

# 155. No-op flush metric

Podrá indicarse:

```text
orm.flush.noop=true
```

como diagnostic attribute bounded.

---

# 156. Flush amplification

Podrá calcularse:

```text
StatementsPerFlush
```

o:

```text
QueriesPerFlush
```

para profiling.

---

# 157. Flush amplification ≠ inefficiency

Un flush con muchas queries puede ser correcto.

Necesita análisis.

---

# 158. Repository find with IdentityMap

Caso:

```text
Repository::find(10)
↓
IdentityMap hit
↓
0 DB queries
```

Telemetry deberá mostrar:

```text
orm operation success
identity map reuse
query count 0
```

---

# 159. Repository find miss

```text
Repository::find(10)
↓
IdentityMap miss
↓
Query
↓
Hydration
↓
IdentityMap registration
```

---

# 160. Identity lookup telemetry

Se podrá observar:

```text
lookup
hit
miss
```

pero el concepto `miss` deberá definirse cuidadosamente.

---

# 161. IdentityMap miss ≠ entity missing

Significa:

```text
not currently managed
```

No:

```text
row absent from database
```

---

# 162. find null

Secuencia:

```text
IdentityMap miss
↓
DB query
↓
0 rows
↓
repository result = null
```

---

# 163. Hydration reuse

En JOIN:

```text
same User appears in 20 rows
```

IdentityMap puede reutilizar:

```text
1 User instance
```

20 veces.

Telemetry deberá diferenciar:

```text
row references
entity creations
entity reuse
```

---

# 164. ORM cache integration

ORM Telemetry podrá correlacionar:

```text
Entity Cache
Metadata Cache
Result Cache
Hydration Cache
```

---

# 165. Entity Cache hit ≠ IdentityMap hit

De nuevo:

```text
L1 managed state
≠
L2 entity cache
```

---

# 166. Cache telemetry ownership

Las métricas cache seguirán bajo Cache Telemetry/Cache System.

ORM solo correlaciona.

---

# 167. Serialization

ORM Telemetry nunca deberá serializar entidades para observarlas.

---

# 168. Lazy properties

Capturar telemetry no deberá acceder a:

```text
lazy relation
lazy property
proxy initializer
```

---

# 169. `__toString()` caution

No deberá invocarse automáticamente sobre entidades para generar etiquetas/logs.

Puede causar:

```text
side effects
lazy loading
errors
sensitive exposure
```

---

# 170. Entity safe representation

Usar:

```text
EntityTypeId
runtime identity
state
loaded-field count
```

según policy.

---

# 171. Sensitive mapped fields

Valores de:

```text
password_hash
secret
access_token
email
financial data
```

no deberán aparecer por default.

---

# 172. ChangeSet field names

Incluso nombres de campos pueden revelar modelo de negocio.

Su exposición deberá ser configurable.

---

# 173. ORM telemetry security policy

```php
interface ORMTelemetrySecurityPolicy
{
    public function allowEntityType(
        EntityType $type
    ): bool;

    public function allowFieldName(
        EntityType $type,
        FieldName $field
    ): bool;

    public function allowSourceLocation(): bool;
}
```

---

# 174. Data minimization

Métricas normalmente necesitan:

```text
counts
durations
outcomes
strategies
```

no datos de dominio.

---

# 175. Persistent runtime

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 176. EntityManager scoped state

Nunca deberán compartirse entre requests:

```text
IdentityMap
UnitOfWork
managed entities
dirty entities
pending removals
EntityManager taint state
ORM telemetry buffers
```

---

# 177. Telemetry isolation

Request A:

```text
managed_count=37
```

Request B debe comenzar:

```text
managed_count=0
```

para un fresh scoped EntityManager.

---

# 178. Physical shared metadata

Podrán compartirse:

```text
compiled entity metadata
mapping metadata
telemetry descriptors
metric instruments
immutable policies
```

---

# 179. No static current EntityManager

Prohibido:

```php
static ?EntityManager $current;
```

en telemetry.

---

# 180. Request-end diagnostics

Al terminar:

```text
EntityManager open?
IdentityMap non-empty?
UoW pending?
active transaction?
tainted?
```

podrán evaluarse.

---

# 181. Non-empty IdentityMap at request end

No necesariamente es error.

El scope puede limpiarse después.

---

# 182. Pending UoW at request end

Puede ser más relevante:

```text
NEW entities pending
DIRTY entities pending
REMOVED entities pending
```

---

# 183. Pending work diagnostic

Podrá generarse:

```text
ORM_PENDING_WORK_AT_SCOPE_END
```

según policy.

---

# 184. Pending work ≠ DB inconsistency

Si nunca hubo flush, DB puede seguir completamente consistente.

---

# 185. Tainted manager at request end

Deberá considerarse diagnóstico importante.

---

# 186. OpenSwoole coroutine isolation

Dos coroutines:

```text
Coroutine A
└── EntityManager A

Coroutine B
└── EntityManager B
```

no podrán compartir IdentityMap.

---

# 187. Shared EntityManager violation

Podrá detectarse en development:

```text
ENTITY_MANAGER_SCOPE_CROSSOVER
```

---

# 188. Worker memory accumulation

Telemetry podrá seguir:

```text
post-request memory
EntityManager objects released
diagnostic buffers released
```

para detectar crecimiento.

---

# 189. PHP GC caveat

Objetos liberados no implican inmediatamente reducción de RSS.

Diagnostics deberán evitar conclusiones falsas.

---

# 190. ORM profiling hooks

El Query Profiler podrá consumir:

```text
ORMOperationId
QueryIds
FlushId
Hydration timings
Relationship load metadata
```

---

# 191. Slow ORM operation

Podrá existir:

```text
ORM operation duration high
```

aunque las queries sean rápidas.

Ejemplo:

```text
hydration CPU heavy
change detection expensive
relationship assembly expensive
```

---

# 192. Slow ORM ≠ slow SQL

Distinción esencial:

```text
Slow ORM Operation
≠
Slow Query
```

---

# 193. Diagnostic decomposition

Ejemplo:

```text
ORM operation = 120 ms

SQL execution = 10 ms
Hydration = 75 ms
Relationship assembly = 20 ms
Other = 15 ms
```

Esto permite localizar overhead del ORM.

---

# 194. Query Profiler integration

El documento 221 utilizará:

```text
Query Telemetry
Connection Telemetry
Transaction Telemetry
ORM Telemetry
```

para análisis integrado.

---

# 195. N+1 system integration

El documento 223 deberá recibir:

```text
ORM root operation
RelationshipId
lazy load signals
QueryId
EntityType
operation context
```

---

# 196. N+1 detector state

No deberá vivir dentro del ORM Telemetry System como lógica general.

Podrá existir un bridge/context especializado.

---

# 197. Debug toolbar

El documento 225 podrá mostrar:

```text
ORM operations
managed entity count
flushes
hydration
lazy loads
N+1 warnings
```

a partir de records seguros.

---

# 198. ORM diagnostics API

Conceptualmente:

```php
DB::telemetry()
    ->orm()
    ->explain();
```

---

# 199. Example overview

```text
ORM TELEMETRY
────────────────────────────────

EntityManager:
  em_scope_17

Status:
  OPEN

Operations:
  18

Queries:
  12

Flushes:
  1

IdentityMap:
  current: 43
  peak: 48
  lookups: 61
  reuses: 18

UnitOfWork:
  NEW: 0
  DIRTY: 0
  REMOVED: 0

Hydration:
  rows: 82
  new entities: 43
  reused entities: 39
  duration: 6.8 ms

Relationships:
  eager loads: 2
  lazy loads: 5
  batch loads: 1

Warnings:
  1 possible N+1 sequence

Consistency:
  CONSISTENT
```

---

# 200. Flush diagnostic

```text
ORM FLUSH
────────────────────────────────

Flush:
  pf_018

Duration:
  41 ms

Before Flush:
  NEW: 4
  DIRTY: 8
  REMOVED: 1

Change Detection:
  3.8 ms

Persistence Plan:
  inserts: 4
  updates: 8
  deletes: 1

Physical Queries:
  7

Transaction:
  tx_024

Transaction Outcome:
  COMMITTED

ORM Consistency:
  CONSISTENT
```

---

# 201. Slow hydration diagnostic

```text
ORM HYDRATION
────────────────────────────────

Query:
  q_204

Rows:
  12,500

Entities Created:
  2,300

Entities Reused:
  10,200

Duration:
  418 ms

Query Execution:
  82 ms

ORM Hydration:
  318 ms

Observation:
  hydration dominates this operation

Suggestion:
  evaluate projection/DTO or lower result cardinality
```

---

# 202. IdentityMap growth diagnostic

```text
ORM MEMORY
────────────────────────────────

Traversal:
  lazy_42

Chunks:
  80

Entities Processed:
  40,000

IdentityMap:
  chunk 1:   500
  chunk 20:  10,000
  chunk 40:  20,000
  chunk 80:  40,000

Observation:
  managed entity count grows with traversal progress

Interpretation:
  processed entities remain managed

Recommendation:
  consider read-only projection,
  explicit detach policy,
  or dedicated EntityManager
```

---

# 203. Evidence vs recommendation

Como en otros subsistemas:

```text
Observed
Inferred
Recommended
```

deberán separarse.

---

# 204. ORM Telemetry policy

```php
final readonly class ORMTelemetryPolicy
{
    public function __construct(
        public bool $metrics,
        public bool $tracing,
        public bool $identityMap,
        public bool $unitOfWork,
        public bool $changeTracking,
        public bool $flush,
        public bool $hydration,
        public bool $relationships,
        public bool $memory,
        public bool $diagnostics,
        public ORMTelemetryBudget $budget,
    ) {}
}
```

---

# 205. Telemetry budget

```php
final readonly class ORMTelemetryBudget
{
    public function __construct(
        public int $maxOperationRecords,
        public int $maxRelationshipRecords,
        public int $maxEntityTypes,
        public int $maxDiagnosticEntries,
        public int $maxTrackedFlushes,
    ) {}
}
```

---

# 206. Budget exhaustion

Estrategias:

```text
aggregate
sample
drop detail
retain summary
```

---

# 207. No correctness impact

Budget exhaustion:

```text
ORM Telemetry degraded
```

no:

```text
ORM operation failed
```

---

# 208. Sampling

Se podrá aplicar sampling a:

```text
repository operations
relationship loads
hydration records
traces
diagnostics
```

---

# 209. Metrics sampling

Counters básicos normalmente no requieren sampling.

---

# 210. High-volume entity events

No deberán producir una señal externa por entidad.

---

# 211. Aggregation boundary

Preferir:

```text
per ORM operation
per flush
per hydration batch
per relationship batch
per request
```

---

# 212. Null instrumentation

Debe existir:

```php
final class NullORMTelemetryInstrumentation
    implements ORMTelemetryInstrumentation
{
    // near-zero overhead
}
```

---

# 213. ORMTelemetryInstrumentation

```php
interface ORMTelemetryInstrumentation
{
    public function startOperation(
        ORMOperationDescriptor $operation,
        ORMTelemetryContext $context,
    ): ORMTelemetryScope;
}
```

---

# 214. ORMTelemetryScope

```php
interface ORMTelemetryScope
{
    public function identityMapObserved(
        IdentityMapTelemetrySnapshot $snapshot
    ): void;

    public function unitOfWorkObserved(
        UnitOfWorkTelemetrySnapshot $snapshot
    ): void;

    public function queryObserved(
        QueryOperationId $query
    ): void;

    public function finish(
        ORMOperationOutcome $outcome
    ): void;
}
```

---

# 215. ORMOperationOutcome

```php
enum ORMOperationOutcome
{
    case SUCCESS;
    case FAILED;
    case CANCELLED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 216. UNKNOWN preservation

Si persistence reality es incierta:

```text
ORMOperationOutcome::UNKNOWN
```

deberá permanecer.

---

# 217. Telemetry failure policy

Un failure de telemetry:

```text
OpenTelemetry exporter
diagnostic buffer
metric provider
```

no deberá afectar ORM semantics por default.

---

# 218. No observer mutation

Telemetry callbacks no recibirán:

```text
mutable EntityManager
mutable UnitOfWork
mutable ChangeSet
mutable entity
```

por default.

---

# 219. Safe snapshots

Preferir:

```text
IdentityMapTelemetrySnapshot
UnitOfWorkTelemetrySnapshot
ORMFlushTelemetrySummary
HydrationTelemetrySummary
```

---

# 220. Testing API

```php
ORMTelemetry::assertOperationCount(3);
```

---

# 221. IdentityMap assertion

```php
ORMTelemetry::assertIdentityMapPeakBelow(100);
```

---

# 222. Query assertion

```php
ORMTelemetry::assertQueryCount(5);
```

---

# 223. Flush assertion

```php
ORMTelemetry::assertFlushCount(1);
```

---

# 224. Dirty entities assertion

```php
ORMTelemetry::assertNoPendingDirtyEntities();
```

---

# 225. Taint assertion

```php
ORMTelemetry::assertEntityManagerNotTainted();
```

---

# 226. Lazy load assertion

```php
ORMTelemetry::assertLazyLoadCount(0);
```

---

# 227. Hydration assertion

```php
ORMTelemetry::assertHydratedEntitiesBelow(500);
```

---

# 228. Query amplification assertion

```php
ORMTelemetry::assertQueriesPerOperationBelow(10);
```

---

# 229. Persistent scope assertion

```php
ORMTelemetry::assertFreshEntityManagerScope();
```

---

# 230. Testing deterministic clock

Debe soportarse:

```text
FakeORMTelemetryClock
```

para:

```text
flush timing
hydration timing
change detection timing
relationship load timing
```

---

# 231. Benchmark modes

Deberán evaluarse:

```text
ORM telemetry disabled
metrics only
metrics + tracing
full diagnostics
memory diagnostics
relationship diagnostics
```

---

# 232. Hot-path constraints

IdentityMap lookup es extremadamente frecuente.

Telemetry deberá evitar:

```text
string allocations
EntityType formatting
stack traces
full context reconstruction
provider network calls
```

en cada lookup.

---

# 233. IdentityMap metrics implementation

Preferir contadores locales/agregados:

```text
lookups++
reuses++
```

y exportación posterior.

---

# 234. Hydration hot path

No crear un telemetry object por campo asignado.

---

# 235. Change tracking hot path

No crear una señal por propiedad comparada.

---

# 236. Relationship hot path

Batch/aggregate por load operation.

---

# 237. Source-location capture

Podrá habilitarse solo para:

```text
lazy loads
N+1 candidates
slow ORM operations
```

en debug/profiling.

---

# 238. Source location cost

Capturar stack traces en cada relación lazy puede ser costoso.

Aplicar sampling/budget.

---

# 239. Debug mode ≠ unsafe mode

Ni siquiera en debug deberán exponerse:

```text
passwords
tokens
credentials
secret properties
```

---

# 240. Directory structure

```text
src/Quantum/Database/Telemetry/ORM/
│
├── Contract/
│   ├── ORMTelemetryInstrumentation.php
│   ├── ORMTelemetryScope.php
│   ├── ORMTelemetrySampler.php
│   └── ORMTelemetrySecurityPolicy.php
│
├── Identity/
│   ├── ORMOperationId.php
│   └── HydrationOperationId.php
│
├── Context/
│   ├── ORMTelemetryContext.php
│   ├── ORMTelemetryContextFactory.php
│   └── ORMTelemetryContextResolver.php
│
├── EntityManager/
│   ├── EntityManagerTelemetry.php
│   ├── EntityManagerTelemetryState.php
│   ├── EntityManagerTelemetrySummary.php
│   └── ORMTaintReason.php
│
├── IdentityMap/
│   ├── IdentityMapTelemetry.php
│   ├── IdentityMapTelemetrySnapshot.php
│   └── IdentityMapGrowthAnalyzer.php
│
├── UnitOfWork/
│   ├── UnitOfWorkTelemetry.php
│   ├── UnitOfWorkTelemetrySnapshot.php
│   └── UnitOfWorkTelemetrySummary.php
│
├── ChangeTracking/
│   ├── ChangeTrackingTelemetry.php
│   ├── ChangeTrackingTelemetryRecord.php
│   └── ChangeSetTelemetrySummary.php
│
├── Flush/
│   ├── ORMFlushTelemetry.php
│   ├── ORMFlushTelemetrySummary.php
│   └── ORMFlushOutcome.php
│
├── Hydration/
│   ├── ORMHydrationTelemetry.php
│   ├── ORMHydrationTelemetrySummary.php
│   └── HydrationTelemetryShape.php
│
├── Relationship/
│   ├── ORMRelationshipTelemetry.php
│   ├── RelationshipLoadTelemetryRecord.php
│   ├── RelationshipLoadStrategyTelemetry.php
│   └── RelationshipCoverageTelemetry.php
│
├── Locking/
│   ├── OptimisticLockTelemetry.php
│   └── PessimisticLockTelemetry.php
│
├── Consistency/
│   ├── ORMConsistencyTelemetry.php
│   └── ORMConsistencyTelemetryRecord.php
│
├── Memory/
│   ├── ORMMemoryTelemetry.php
│   ├── ORMMemorySnapshot.php
│   └── ORMMemoryGrowthDiagnostic.php
│
├── Operation/
│   ├── ORMOperationType.php
│   ├── ORMApiOrigin.php
│   ├── ORMOperationOutcome.php
│   ├── ORMOperationTelemetryRecord.php
│   └── ORMTelemetrySummary.php
│
├── Metric/
│   ├── ORMMetricRecorder.php
│   ├── ORMMetricRegistry.php
│   └── ORMMetricDescriptor.php
│
├── Trace/
│   ├── ORMSpanFactory.php
│   ├── ORMSpanEnricher.php
│   └── ORMTracePolicy.php
│
├── Diagnostics/
│   ├── ORMTelemetryInspector.php
│   ├── ORMTelemetryExplainer.php
│   ├── ORMDiagnosticBuffer.php
│   └── ORMStateLeakDetector.php
│
├── Policy/
│   ├── ORMTelemetryPolicy.php
│   ├── CompiledORMTelemetryPolicy.php
│   └── ORMTelemetryBudget.php
│
├── Instrumentation/
│   ├── DefaultORMTelemetryInstrumentation.php
│   ├── DefaultORMTelemetryScope.php
│   └── NullORMTelemetryInstrumentation.php
│
├── Testing/
│   ├── RecordingORMTelemetry.php
│   ├── ORMTelemetryAssertions.php
│   └── FakeORMTelemetryClock.php
│
└── Exception/
    ├── ORMTelemetryException.php
    ├── ORMTelemetryStateException.php
    ├── ORMTelemetrySecurityException.php
    └── ORMTelemetryBudgetException.php
```

---

# 241. Architectural invariants

## DB-OTEL-001
ORM Telemetry observará ORM sin controlarlo.

## DB-OTEL-002
ORM Telemetry no ejecutará persist().

## DB-OTEL-003
ORM Telemetry no ejecutará remove().

## DB-OTEL-004
ORM Telemetry no ejecutará flush().

## DB-OTEL-005
ORM Telemetry no ejecutará clear().

## DB-OTEL-006
ORM Telemetry no ejecutará detach().

## DB-OTEL-007
ORM Telemetry no hidratará entidades.

## DB-OTEL-008
ORM Telemetry no modificará ChangeSets.

## DB-OTEL-009
ORM Telemetry no modificará Persistence Plans.

## DB-OTEL-010
ORM Telemetry será distinta de Entity Lifecycle Events.

## DB-OTEL-011
ORM Telemetry será distinta de Persistence Events.

## DB-OTEL-012
ORM Telemetry será distinta de Query Telemetry.

## DB-OTEL-013
ORMOperationId será distinto de QueryId.

## DB-OTEL-014
Una ORM operation podrá producir cero queries.

## DB-OTEL-015
Una ORM operation podrá producir múltiples queries.

## DB-OTEL-016
EntityManagerScopeId será distinto de RequestId.

## DB-OTEL-017
EntityManagerScopeId será distinto de TransactionId.

## DB-OTEL-018
EntityManager taint será observable.

## DB-OTEL-019
Taint reason será explícita.

## DB-OTEL-020
TAINTED EntityManager no será automáticamente request failure.

## DB-OTEL-021
IdentityMap size será distinto de DB row count.

## DB-OTEL-022
IdentityMap hit será distinto de Entity Cache hit.

## DB-OTEL-023
IdentityMap hit será distinto de Result Cache hit.

## DB-OTEL-024
IdentityMap growth será observable.

## DB-OTEL-025
Large IdentityMap no será automáticamente memory leak.

## DB-OTEL-026
IdentityMap será canonical owner del managed identity count.

## DB-OTEL-027
UnitOfWork será canonical owner del pending work state.

## DB-OTEL-028
UoW snapshot será distinto de live UnitOfWork.

## DB-OTEL-029
ChangeSet telemetry no expondrá valores por default.

## DB-OTEL-030
ChangeSet telemetry no modificará ChangeSets.

## DB-OTEL-031
Change detection duration será observable.

## DB-OTEL-032
Flush será observable.

## DB-OTEL-033
Flush será distinto de commit.

## DB-OTEL-034
Flush completion será distinta de durable commit.

## DB-OTEL-035
Flush outcome UNKNOWN permanecerá UNKNOWN.

## DB-OTEL-036
ORM persistence operations serán distintas de physical statements.

## DB-OTEL-037
Batch entity insert podrá generar una sola physical query.

## DB-OTEL-038
Hydration telemetry será observable.

## DB-OTEL-039
Rows consumidos serán distintos de entities hydrated.

## DB-OTEL-040
Entity reused será distinto de entity newly hydrated.

## DB-OTEL-041
Hydration source será observable.

## DB-OTEL-042
Partial entity será observable.

## DB-OTEL-043
Partial entity no será tratada como broken entity.

## DB-OTEL-044
Hydration shape será bounded.

## DB-OTEL-045
Memory telemetry no se capturará por entidad.

## DB-OTEL-046
PHP memory será distinta de process RSS.

## DB-OTEL-047
Entity transitions serán agregadas por default.

## DB-OTEL-048
Per-entity external telemetry estará deshabilitada por default.

## DB-OTEL-049
EntityType estará sujeto a cardinality policy.

## DB-OTEL-050
EntityId no será metric label.

## DB-OTEL-051
Model API y Repository usarán el mismo ORM telemetry engine.

## DB-OTEL-052
Active Record no tendrá segundo telemetry engine.

## DB-OTEL-053
API origin podrá observarse.

## DB-OTEL-054
Relationship load strategy será observable.

## DB-OTEL-055
Lazy load será distinto de N+1.

## DB-OTEL-056
N+1 detection será un sistema separado.

## DB-OTEL-057
Eager loading será distinto de JOIN.

## DB-OTEL-058
Batch relationship loading será observable.

## DB-OTEL-059
Relationship PARTIAL coverage no será EMPTY.

## DB-OTEL-060
Query amplification será distinta de N+1 proof.

## DB-OTEL-061
Persistence correlation usará stable IDs.

## DB-OTEL-062
Optimistic lock conflict será distinto de deadlock.

## DB-OTEL-063
Optimistic lock conflicts serán observables.

## DB-OTEL-064
Entity version values no serán metric labels.

## DB-OTEL-065
Persistence consistency status será observable.

## DB-OTEL-066
UNKNOWN consistency permanecerá UNKNOWN.

## DB-OTEL-067
TAINTED status tendrá alta importancia diagnóstica.

## DB-OTEL-068
Telemetry no decidirá clear after taint.

## DB-OTEL-069
ORM batch será distinto de Bulk API.

## DB-OTEL-070
Factory make() no producirá persistence ORM telemetry.

## DB-OTEL-071
Factory create() podrá producir ORM telemetry si usa ORM.

## DB-OTEL-072
Seeder solo producirá ORM telemetry cuando use ORM.

## DB-OTEL-073
Fixture solo producirá ORM telemetry cuando use ORM.

## DB-OTEL-074
Direct bulk import no fingirá per-entity ORM telemetry.

## DB-OTEL-075
ORM_ENTITY import sí podrá producir ORM telemetry.

## DB-OTEL-076
Lazy Collection podrá causar IdentityMap growth.

## DB-OTEL-077
Chunked reading no implicará constant ORM memory.

## DB-OTEL-078
IdentityMap growth diagnostics no harán auto-clear.

## DB-OTEL-079
Resource Governance conservará autoridad.

## DB-OTEL-080
Metadata Cache hit será distinto de IdentityMap hit.

## DB-OTEL-081
Metadata telemetry no expondrá runtime objects.

## DB-OTEL-082
ORM failure será distinta de Query failure.

## DB-OTEL-083
Failure category será estructurada.

## DB-OTEL-084
Failure stage será estructurada.

## DB-OTEL-085
Metrics utilizarán dimensiones bounded.

## DB-OTEL-086
TenantId no será metric label por default.

## DB-OTEL-087
QueryId no será metric label.

## DB-OTEL-088
TransactionId no será metric label.

## DB-OTEL-089
RelationshipId no será metric label por default.

## DB-OTEL-090
Raw FQCN no será identidad externa preferida.

## DB-OTEL-091
ORM spans no se crearán por entidad por default.

## DB-OTEL-092
Flush span será una operación ORM de alto nivel.

## DB-OTEL-093
Hydration span podrá correlacionarse con Query span.

## DB-OTEL-094
Relationship load spans podrán correlacionarse con N+1 analysis.

## DB-OTEL-095
EntityManager lifetime span largo no será default.

## DB-OTEL-096
Cada medición tendrá canonical owner.

## DB-OTEL-097
Double counting deberá evitarse.

## DB-OTEL-098
persist() telemetry representará scheduling.

## DB-OTEL-099
persist() telemetry no representará INSERT.

## DB-OTEL-100
remove() telemetry representará scheduling.

## DB-OTEL-101
remove() telemetry no representará DELETE.

## DB-OTEL-102
Empty flush será observable.

## DB-OTEL-103
No-op flush no fingirá queries.

## DB-OTEL-104
Flush amplification será diagnóstico, no conclusión automática.

## DB-OTEL-105
IdentityMap miss no significará row absent.

## DB-OTEL-106
Hydration JOIN reuse será observable.

## DB-OTEL-107
Entity Cache será distinto de IdentityMap.

## DB-OTEL-108
ORM Telemetry no serializará entidades.

## DB-OTEL-109
Telemetry no activará lazy relations.

## DB-OTEL-110
Telemetry no llamará __toString() de entidades automáticamente.

## DB-OTEL-111
Sensitive mapped values no serán capturados por default.

## DB-OTEL-112
Field names podrán considerarse sensitive.

## DB-OTEL-113
Persistent runtime ORM state será scoped.

## DB-OTEL-114
IdentityMap no sobrevivirá accidentalmente entre requests.

## DB-OTEL-115
UnitOfWork no sobrevivirá accidentalmente entre requests.

## DB-OTEL-116
EntityManager taint state no sobrevivirá accidentalmente entre scopes.

## DB-OTEL-117
Immutable metadata podrá compartirse.

## DB-OTEL-118
No existirá static mutable current EntityManager en telemetry.

## DB-OTEL-119
Pending UoW at scope end será diagnosticable.

## DB-OTEL-120
Pending UoW no implicará DB inconsistency automáticamente.

## DB-OTEL-121
OpenSwoole EntityManager state será coroutine-safe.

## DB-OTEL-122
Cross-scope EntityManager usage será diagnosticable.

## DB-OTEL-123
Worker memory analysis distinguirá object release de RSS.

## DB-OTEL-124
Slow ORM operation será distinta de slow SQL.

## DB-OTEL-125
ORM overhead podrá descomponerse.

## DB-OTEL-126
Query Profiler consumirá ORM telemetry.

## DB-OTEL-127
N+1 detector consumirá ORM telemetry.

## DB-OTEL-128
Debug Toolbar consumirá safe ORM records.

## DB-OTEL-129
Diagnostics separarán observed, inferred y recommended.

## DB-OTEL-130
Telemetry policy será compilable.

## DB-OTEL-131
Telemetry budget será bounded.

## DB-OTEL-132
Budget exhaustion no romperá ORM correctness.

## DB-OTEL-133
High-volume entity signals serán agregadas.

## DB-OTEL-134
Null instrumentation estará disponible.

## DB-OTEL-135
Disabled telemetry tendrá overhead mínimo.

## DB-OTEL-136
IdentityMap hot path evitará allocations innecesarias.

## DB-OTEL-137
Hydration hot path no producirá signal por field.

## DB-OTEL-138
Change Tracking hot path no producirá signal por comparison.

## DB-OTEL-139
Relationship telemetry se agregará por load operation.

## DB-OTEL-140
Source-location capture será opt-in.

## DB-OTEL-141
Source-location capture estará sujeta a budget.

## DB-OTEL-142
Debug mode no autorizará secret exposure.

## DB-OTEL-143
Telemetry provider failure no cambiará ORM outcome por default.

## DB-OTEL-144
Telemetry callbacks no recibirán mutable EntityManager por default.

## DB-OTEL-145
Telemetry callbacks no recibirán mutable UnitOfWork por default.

## DB-OTEL-146
Telemetry callbacks no recibirán mutable entity por default.

## DB-OTEL-147
Safe snapshots serán preferidos.

## DB-OTEL-148
Testing podrá verificar operation count.

## DB-OTEL-149
Testing podrá verificar IdentityMap peak.

## DB-OTEL-150
Testing podrá verificar query counts.

## DB-OTEL-151
Testing podrá verificar flush count.

## DB-OTEL-152
Testing podrá verificar pending dirty entities.

## DB-OTEL-153
Testing podrá verificar EntityManager taint.

## DB-OTEL-154
Testing podrá verificar lazy loads.

## DB-OTEL-155
Testing podrá verificar hydration count.

## DB-OTEL-156
Testing podrá verificar query amplification.

## DB-OTEL-157
Testing podrá verificar fresh ORM scope.

## DB-OTEL-158
Fake clock permitirá timing determinista.

## DB-OTEL-159
ORM telemetry tendrá benchmarks propios.

## DB-OTEL-160
ORM Telemetry nunca sustituirá EntityManager.

## DB-OTEL-161
ORM Telemetry nunca sustituirá IdentityMap.

## DB-OTEL-162
ORM Telemetry nunca sustituirá UnitOfWork.

## DB-OTEL-163
ORM Telemetry nunca sustituirá Hydrator.

## DB-OTEL-164
ORM Telemetry nunca sustituirá Persistence Engine.

## DB-OTEL-165
ORM Telemetry nunca sustituirá Relationship Loader.

## DB-OTEL-166
ORM Telemetry nunca sustituirá Transaction Manager.

## DB-OTEL-167
ORM Telemetry nunca será Database truth.

## DB-OTEL-168
ORM Telemetry preservará incertidumbre.

## DB-OTEL-169
ORM Telemetry permanecerá provider-agnostic.

## DB-OTEL-170
ORM Telemetry estará desacoplada de herramientas de debug concretas.

---

# 242. Modelo formal de operación ORM

Sea:

```text
O
```

una operación ORM.

Puede producir:

```text
Q(O) = {q1, q2, ..., qn}
```

queries físicas.

Entonces:

```text
|Q(O)| >= 0
```

Por ejemplo:

```text
IdentityMap hit
⇒
|Q(O)| = 0
```

---

# 243. Modelo de IdentityMap

Sea:

```text
I
```

el IdentityMap.

Para una entidad:

```text
K = EntityType + Identifier + Context
```

si:

```text
K ∈ I
```

entonces:

```text
Lookup(K)
→ REUSE(existing instance)
```

y telemetry podrá incrementar:

```text
identity_map.reuse
```

---

# 244. Modelo de UnitOfWork

Sea:

```text
U =
{
    NEW,
    MANAGED,
    DIRTY,
    REMOVED
}
```

Telemetry observa:

```text
Snapshot(U,t)
```

sin modificar:

```text
U
```

---

# 245. Modelo de flush

Sea:

```text
F(U)
```

una operación flush.

Conceptualmente:

```text
F(U)
=
ChangeDetection(U)
+
PersistencePlan(U)
+
Execute(P)
+
Reconcile(U, DBKnowledge)
```

pero:

```text
F(U)
≠
TransactionCommit
```

por definición.

---

# 246. Modelo de hydration

Sea:

```text
R = {r1, r2, ..., rn}
```

un resultado físico.

Hydration produce:

```text
H(R)
=
application result shape
```

Telemetry podrá observar:

```text
RowsConsumed = n
EntitiesCreated = c
EntitiesReused = u
```

donde no existe obligación de que:

```text
n = c
```

---

# 247. Modelo de overhead ORM

Conceptualmente:

```text
T_orm_operation
=
T_query_execution
+
T_hydration
+
T_change_tracking
+
T_relationship_assembly
+
T_orm_coordination
```

dependiendo de la operación.

---

# 248. No aditividad estricta

Las fases pueden superponerse, especialmente con:

```text
streaming
lazy loading
batched hydration
```

Por tanto:

```text
T_total
```

no debe asumirse siempre como suma exacta.

---

# 249. Modelo de query amplification

Para `N` operaciones ORM:

```text
A =
PhysicalQueryExecutions
/
ORMOperations
```

pero:

```text
A high
```

no implica automáticamente:

```text
N+1
```

---

# 250. Modelo de memoria

Conceptualmente:

```text
M_orm
≈
M_identity_map
+
M_uow
+
M_hydrated_graph
+
M_snapshots
+
M_relationship_collections
+
M_telemetry_buffers
```

Telemetry deberá mantener:

```text
M_telemetry_buffers
```

acotado.

---

# 251. Arquitectura de interacción

```text
                      ORM API
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      EntityManager  Repository   Model API
            │           │           │
            └───────────┼───────────┘
                        ▼
                    ORM Operation
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
      IdentityMap    UnitOfWork   Metadata
           │            │
           │            ▼
           │       Change Tracking
           │            │
           └──────┬─────┘
                  ▼
                Flush
                  │
                  ▼
         Persistence Engine
                  │
                  ▼
             Query Engine
                  │
                  ▼
             Transaction
```

ORM Telemetry observa cada boundary mediante snapshots, counters, timings y correlación.

---

# 252. Correlation graph

```text
Request / Job
     │
     ▼
EntityManagerScopeId
     │
     ├── ORMOperationId
     │      │
     │      ├── QueryOperationId
     │      ├── HydrationOperationId
     │      └── RelationshipLoad
     │
     └── PersistenceFlushId
            │
            ├── PersistenceOperationId
            ├── QueryOperationId
            └── TransactionId
```

---

# 253. Filosofía arquitectónica

VoltStack seguirá:

```text
State summaries
over
live ORM object export

Operation correlation
over
isolated metrics

IdentityMap visibility
over
opaque managed state

Flush decomposition
over
"ORM is slow"

Hydration visibility
over
SQL-only profiling

Relationship semantics
over
query-count guesswork

Bounded aggregation
over
per-entity telemetry

Persistent-runtime isolation
over
request-lifetime assumptions

Evidence
over
automatic optimization

Observation
over
mutation
```

---

# 254. Regla maestra

> **VoltStack deberá poder explicar qué trabajo realizó el ORM, cuánto estado mantuvo, qué entidades administró, cuánto costó detectar cambios, hidratar resultados, cargar relaciones y sincronizar persistencia, y cómo ese trabajo se correlacionó con queries y transacciones, sin convertir telemetry en participante del ORM.**

En forma compacta:

```text
ORM Operation
↓
Observe State
↓
Measure Cost
↓
Correlate Queries
↓
Correlate Transaction
↓
Aggregate
↓
Diagnose
```

Nunca:

```text
Telemetry
↓
Mutate Entity / UoW / IdentityMap / Flush
```

---

# 255. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
○ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
○ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
○ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 256. Siguiente documento

```text
221_DATABASE_QUERY_PROFILER_SYSTEM.md
```

El siguiente documento deberá combinar la evidencia de:

```text
Query Telemetry
Connection Telemetry
Transaction Telemetry
ORM Telemetry
```

para construir un perfil explicable de cada consulta y operación relacionada:

```text
Query Operation
↓
Planning
↓
Compilation
↓
Connection Wait
↓
Execution
↓
Result Consumption
↓
Hydration
↓
ORM Assembly
```

incluyendo:

```text
query profile
phase decomposition
query fingerprint aggregation
hot queries
high-frequency queries
high-total-cost queries
connection wait contribution
database execution contribution
hydration contribution
transaction contribution
ORM contribution
cache contribution
replica/shard context
row counts
result cardinality
query plan metadata
EXPLAIN integration boundaries
query profile sessions
sampling
profile comparison
regression detection inputs
security/redaction
persistent runtime isolation
diagnostic reports
testing
```

bajo la regla:

> **El Query Profiler analiza evidencia de ejecución ya observada para explicar dónde se consume tiempo y recursos; no deberá convertirse en Query Optimizer ni modificar consultas automáticamente.**