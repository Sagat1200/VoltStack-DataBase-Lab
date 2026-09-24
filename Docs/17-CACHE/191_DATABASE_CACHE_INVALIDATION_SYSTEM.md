# 191_DATABASE_CACHE_INVALIDATION_SYSTEM.md

# VoltStack Quantum Database
## Cache Invalidation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 191 — Cache Invalidation System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `190_DATABASE_ENTITY_CACHE_SYSTEM.md`  
**Siguiente documento:** `192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md`

---

# 1. Propósito

`Cache Invalidation System` define la arquitectura transversal mediante la cual VoltStack determina qué artefactos de caché dejan de ser confiables después de un cambio semántico en el sistema de base de datos.

El sistema deberá coordinar invalidaciones para:

- Query Cache;
- Result Cache;
- Metadata Cache;
- Entity Cache;
- futuras cachés de relaciones;
- caches derivadas;
- caches distribuidas;
- caches locales de workers;
- caches específicas de plugins.

Regla maestra:

> **VoltStack invalidará caché a partir de cambios semánticos confirmados y dependencias conocidas; no a partir de strings SQL ni mediante `flush all` como estrategia normal.**

Formalmente:

```text
SemanticChangeSet
        ↓
Dependency Resolution
        ↓
Invalidation Plan
        ↓
Transaction Outcome
        ↓
Invalidation Publication
        ↓
Cache Consistency
```

---

# 2. Problema

Supongamos:

```php
$user->status = UserStatus::Inactive;

$entityManager->flush();
```

Este cambio puede afectar:

```text
Entity Cache
    User#42

Result Cache
    active-users

Result Cache
    admin-dashboard-users

Query-derived Cache
    active-user-count

Relationship Cache
    Team#7.members

Application Cache
    statistics.users.active
```

Una estrategia ingenua sería:

```text
UPDATE users ...
↓
cache.clear()
```

Eso es correcto solo en el sentido más bruto, pero destruye:

- hit ratio;
- escalabilidad;
- aislamiento;
- rendimiento;
- locality;
- capacidad de distribuir cache;
- eficiencia en aplicaciones grandes.

VoltStack necesita:

```text
What changed?
      ↓
What depends on it?
      ↓
Which artifacts are now unsafe?
      ↓
When may invalidation become visible?
      ↓
How is it propagated?
```

---

# 3. Cache invalidation ≠ cache clearing

Regla:

```text
Invalidation
≠
Clear Everything
```

`clear()` puede existir como herramienta administrativa.

No será el mecanismo normal de coherencia.

---

# 4. Arquitectura general

```text
Database / ORM Mutation
          │
          ▼
 Semantic Change Capture
          │
          ▼
 SemanticChangeSet
          │
          ▼
 CacheDependencyResolver
          │
          ▼
 CacheInvalidationPlanner
          │
          ▼
 CacheInvalidationPlan
          │
          ▼
 Transaction Coordinator
       /        \
   COMMIT      ROLLBACK
 confirmed    confirmed
      │           │
      ▼           ▼
  Publish       Discard
      │
      ▼
 CacheInvalidationBus
      │
 ┌────┼───────────┬─────────────┐
 ▼    ▼           ▼             ▼
L1? Entity      Result       Other
    Cache       Cache        Regions
```

El `IdentityMap` no es una caché distribuida y posee reglas de lifecycle propias; no deberá tratarse mecánicamente como otra región L2.

---

# 5. Conceptos principales

La arquitectura utilizará al menos:

```text
SemanticChange
SemanticChangeSet
CacheDependency
CacheDependencyKey
CacheInvalidationRequest
CacheInvalidationPlan
CacheInvalidationTarget
CacheInvalidationGeneration
CacheInvalidationToken
CacheInvalidationEvent
CacheInvalidationReceipt
```

---

# 6. SemanticChange

Representa un cambio lógico conocido.

```php
interface SemanticChange
{
    public function kind(): SemanticChangeKind;

    public function domain(): PersistenceDomain;
}
```

Ejemplos:

```text
EntityInserted
EntityUpdated
EntityDeleted
EntityFieldChanged
RelationshipMembershipChanged
TableChanged
SchemaChanged
MetadataGenerationChanged
PartitionMoved
TenantDatabaseChanged
```

---

# 7. SemanticChange ≠ SQL statement

No:

```text
UPDATE users SET status = ? WHERE id = ?
```

como unidad primaria de invalidación.

Preferir:

```text
EntityUpdated(
    entity = User,
    id = 42,
    fields = [status],
    oldVersion = 17,
    newVersion = 18
)
```

---

# 8. Razón

Dos SQL distintos pueden representar el mismo cambio semántico.

Asimismo, una sola operación semántica puede generar varios statements.

Por tanto:

```text
SQL Shape
≠
Cache Dependency Semantics
```

---

# 9. SemanticChangeSet

Una operación puede producir múltiples cambios:

```php
final readonly class SemanticChangeSet
{
    /**
     * @param list<SemanticChange> $changes
     */
    public function __construct(
        public SemanticChangeSetId $id,
        public array $changes,
        public PersistenceDomain $domain,
    ) {}
}
```

---

# 10. Ejemplo

Mover un usuario de equipo:

```text
User#42.team_id:
7 → 9
```

puede generar:

```text
EntityUpdated(User#42)
RelationshipMembershipChanged(Team#7.members)
RelationshipMembershipChanged(Team#9.members)
```

---

# 11. Origen de cambios

Los cambios podrán provenir de:

```text
ORM UnitOfWork
Bulk DML
Query Engine
Schema System
Migration System
Administrative Operations
External Change Feed Extension
```

---

# 12. Nivel de conocimiento

No todas las mutaciones conocen exactamente las entidades afectadas.

Ejemplo:

```sql
UPDATE users
SET status = 'inactive'
WHERE last_login_at < ?
```

Puede conocerse:

```text
table = users
field = status
predicate = ...
```

pero no necesariamente todos los IDs.

---

# 13. Change precision

```php
enum SemanticChangePrecision
{
    case ENTITY_EXACT;
    case KEY_SET;
    case RELATION_EXACT;
    case PREDICATE_BOUNDED;
    case TABLE_WIDE;
    case DOMAIN_WIDE;
    case UNKNOWN;
}
```

---

# 14. UNKNOWN ≠ no change

Regla crítica:

```text
UNKNOWN
≠
SAFE
```

Si VoltStack sabe que hubo una mutación pero no puede determinar exactamente qué entries afecta, deberá ampliar conservadoramente el scope de invalidación.

---

# 15. Escalation

Ejemplo:

```text
Exact Entity
    ↓ if unavailable
Entity Type / Table
    ↓
Cache Region
    ↓
Persistence Domain
```

Solo en último extremo:

```text
Global Cache Domain
```

---

# 16. Dependency model

La invalidación necesita saber:

```text
CachedArtifact
    depends on
SemanticData
```

Formalmente:

```text
CacheArtifact A
→ DependencySet D(A)
```

Si:

```text
ChangedData ∩ D(A) ≠ ∅
```

entonces `A` puede requerir invalidación.

---

# 17. CacheDependency

```php
interface CacheDependency
{
    public function key(): CacheDependencyKey;

    public function scope(): CacheDependencyScope;
}
```

---

# 18. Dependency types

```text
EntityDependency
EntityTypeDependency
EntityFieldDependency
TableDependency
RelationshipDependency
QueryPredicateDependency
SchemaDependency
MetadataDependency
TenantDependency
ShardDependency
PartitionDependency
CustomDependency
```

---

# 19. EntityDependency

Ejemplo:

```text
Entity(User, 42)
```

Un Entity Cache entry naturalmente depende de esta identidad.

---

# 20. EntityFieldDependency

Un resultado:

```text
active users
```

puede depender específicamente de:

```text
User.status
```

aunque su payload incluya más campos.

---

# 21. Field-aware invalidation

Si cambia:

```text
User.profile_photo
```

un cache de:

```text
COUNT(users WHERE status = active)
```

no necesita invalidarse si su dependencia conocida solo es:

```text
User.status
```

---

# 22. Dependency precision

Cuanto mayor sea la precisión:

```text
less unnecessary invalidation
```

pero también:

```text
more dependency metadata
+
more planning cost
```

---

# 23. Conservative correctness

VoltStack preferirá:

```text
false-positive invalidation
```

sobre:

```text
false-negative invalidation
```

cuando la evidencia sea incompleta.

---

# 24. False positive

Invalidar algo que todavía era válido:

```text
performance cost
```

---

# 25. False negative

No invalidar algo que ya no era válido:

```text
correctness risk
```

Por tanto:

```text
Correctness
>
Hit Ratio
```

---

# 26. Dependency Graph

Conceptualmente:

```text
User#42
  │
  ├── EntityCache(User#42)
  │
  ├── ResultCache(active-users)
  │
  └── ResultCache(team-7-users)

User.status
  │
  ├── ResultCache(active-users)
  └── ResultCache(active-user-count)

Team#7.members
  │
  └── RelationshipCache(Team#7.members)
```

---

# 27. Graph implementation

No es obligatorio mantener un grafo materializado gigantesco en memoria.

Podrá implementarse mediante:

- tags;
- generations;
- dependency fingerprints;
- reverse indexes;
- region generations;
- version tokens;
- hybrid strategies.

---

# 28. CacheDependencyRegistry

```php
interface CacheDependencyRegistry
{
    public function register(
        CacheArtifactKey $artifact,
        CacheDependencySet $dependencies,
    ): void;

    public function resolveAffected(
        SemanticChangeSet $changes,
    ): CacheArtifactSet;
}
```

---

# 29. Registry ≠ global PHP array

En aplicaciones distribuidas podrá requerir backend especializado.

---

# 30. Dependency Tags

Una estrategia práctica:

```text
entity:user:42
entity-type:user
field:user:status
table:users
relation:team:7:members
tenant:acme
shard:3
```

---

# 31. Tags are semantic

Los tags oficiales deberán generarse mediante objetos tipados.

No:

```php
"users-" . $id
```

disperso por el framework.

---

# 32. CacheDependencyKeyFactory

```php
interface CacheDependencyKeyFactory
{
    public function forEntity(EntityKey $key): CacheDependencyKey;

    public function forField(
        EntityType $entity,
        FieldId $field,
    ): CacheDependencyKey;
}
```

---

# 33. Namespace

Toda dependency key deberá incluir suficiente domain separation.

```text
Application
Environment
Database
Tenant?
Shard?
Dependency Type
Dependency Identity
```

---

# 34. Cross-tenant contamination

Prohibido:

```text
invalidate entity:user:42
```

si eso puede afectar accidentalmente:

```text
Tenant A User#42
Tenant B User#42
```

salvo que la invalidación global sea intencional.

---

# 35. Invalidation targets

```php
enum CacheInvalidationTargetKind
{
    case ARTIFACT;
    case ENTITY;
    case QUERY;
    case RESULT;
    case RELATIONSHIP;
    case REGION;
    case METADATA_GENERATION;
    case DOMAIN;
}
```

---

# 36. CacheInvalidationRequest

```php
final readonly class CacheInvalidationRequest
{
    public function __construct(
        public CacheInvalidationRequestId $id,
        public SemanticChangeSet $changes,
        public CacheInvalidationReason $reason,
        public CacheInvalidationConsistencyRequirement $consistency,
    ) {}
}
```

---

# 37. Request ≠ Plan

`Request` expresa:

```text
what changed
```

`Plan` expresa:

```text
what must be invalidated
and how
```

---

# 38. CacheInvalidationPlanner

```php
interface CacheInvalidationPlanner
{
    public function plan(
        CacheInvalidationRequest $request,
        CacheInvalidationPlanningContext $context,
    ): CacheInvalidationPlan;
}
```

---

# 39. Plan

```php
final readonly class CacheInvalidationPlan
{
    /**
     * @param list<CacheInvalidationOperation> $operations
     */
    public function __construct(
        public CacheInvalidationPlanId $id,
        public array $operations,
        public CacheInvalidationScope $scope,
    ) {}
}
```

---

# 40. Operations

```text
DeleteArtifact
DeleteEntityEntry
InvalidateTag
BumpGeneration
InvalidateRegion
InvalidateDomain
PublishInvalidation
RegisterRecovery
```

---

# 41. Delete vs generation bump

Dos estrategias fundamentales:

```text
Physical Deletion
```

y:

```text
Logical Invalidation
```

---

# 42. Physical deletion

```text
key K
↓
DELETE K
```

---

# 43. Logical invalidation

```text
generation G7
↓
G8
```

Los entries de `G7` dejan de ser válidos aunque sigan físicamente almacenados.

---

# 44. Generation model

```php
final readonly class CacheInvalidationGeneration
{
    public function __construct(
        public CacheDependencyKey $dependency,
        public int|string $generation,
    ) {}
}
```

---

# 45. Generation validation

Un cached artifact puede registrar:

```text
dependencies:
    User.status → G17
    users table → G91
```

Al leer:

```text
current generation == cached generation?
```

---

# 46. Generation mismatch

```text
Cached G17
Current G18
→ STALE
```

---

# 47. Advantages

Generation-based invalidation evita tener que encontrar y eliminar físicamente cada key dependiente.

---

# 48. Tradeoff

Puede requerir:

```text
generation lookups
metadata overhead
garbage collection
```

---

# 49. Hybrid strategy

VoltStack deberá permitir:

```text
exact key delete
+
generation bump
```

según tipo de caché.

---

# 50. Entity Cache strategy

Para:

```text
User#42
```

puede ser eficiente:

```text
delete exact EntityCacheKey
```

---

# 51. Result Cache strategy

Para cientos de resultados dependientes de:

```text
User.status
```

puede ser mejor:

```text
bump field/tag generation
```

---

# 52. Metadata Cache strategy

Metadata invalidation normalmente dependerá de:

```text
framework generation
mapping generation
compiled metadata fingerprint
deployment generation
```

no de cambios de filas.

---

# 53. Query Cache strategy

Un Query Cache que almacena compilación/plan normalmente depende de:

```text
query structure
dialect
platform capabilities
schema generation
metadata generation
```

No necesariamente de valores de filas.

---

# 54. Important distinction

```text
Data mutation
```

no deberá invalidar automáticamente:

```text
Compiled Query Cache
```

si su estructura continúa siendo válida.

---

# 55. Schema change

En cambio:

```text
ALTER TABLE
```

puede afectar:

```text
Query Cache
Metadata Cache
Result Cache
Entity Cache
```

según el cambio.

---

# 56. Schema invalidation

El Schema System deberá emitir:

```text
SchemaSemanticChangeSet
```

con suficiente precisión.

---

# 57. Migration integration

Las migraciones podrán producir:

```text
BeforeMigrationInvalidationPlan
AfterMigrationInvalidationPlan
```

cuando sea necesario.

---

# 58. Deployment compatibility

En rolling deployments pueden coexistir:

```text
Application Version A
Application Version B
```

con metadata distinta.

---

# 59. Generation namespace

Por tanto, cache generation podrá incluir:

```text
application schema generation
entity metadata generation
codec generation
```

---

# 60. Transaction-aware invalidation

Regla central:

> **Una invalidación derivada de una mutación transaccional no se publicará globalmente como cambio confirmado antes de que el commit sea confirmado.**

---

# 61. During transaction

```text
UPDATE
↓
SemanticChange captured
↓
Invalidation planned
↓
Deferred
```

---

# 62. Commit confirmed

```text
COMMIT confirmed
↓
Publish invalidation
```

---

# 63. Rollback confirmed

```text
ROLLBACK confirmed
↓
Discard invalidation plan
```

---

# 64. Flush

```text
flush()
```

puede producir semantic changes pendientes.

Pero:

```text
flush
≠
commit
```

---

# 65. Therefore

```text
flush
→ deferred invalidation
```

no:

```text
flush
→ globally confirmed invalidation
```

---

# 66. Why not invalidate immediately?

Invalidar antes del commit no necesariamente rompe correctness del dato —un miss puede volver a leer el estado anterior— pero:

- genera churn;
- puede provocar repopulation races;
- publica semántica de cambio antes de confirmación;
- complica rollback;
- reduce eficiencia.

VoltStack mantendrá ordering explícito.

---

# 67. Unknown commit outcome

Caso:

```text
COMMIT dispatched
↓
connection lost
↓
UNKNOWN
```

No sabemos si los cambios existen en DB.

---

# 68. UNKNOWN policy

En este caso:

```text
do not publish new cache values
```

pero:

```text
invalidate possibly affected cached artifacts
```

es generalmente seguro.

---

# 69. Conservative invalidation

Si DB hizo rollback:

```text
invalidation
→ unnecessary miss
```

Si DB hizo commit:

```text
invalidation
→ prevents stale cache
```

Por tanto:

```text
UNKNOWN
→ conservative invalidation
```

---

# 70. UNKNOWN precision

Se invalidarán únicamente los artefactos:

```text
possibly affected
```

cuando puedan conocerse.

---

# 71. Unknown change set

Si además el change set es incierto:

```text
UNKNOWN transaction
+
UNKNOWN affected rows
```

puede requerirse escalamiento a:

```text
table
region
domain
```

---

# 72. Transaction resource

El Transaction Context podrá registrar:

```text
cache.deferred_invalidations
```

en su `TransactionResourceRegistry`.

---

# 73. DeferredCacheInvalidationSet

```php
final class DeferredCacheInvalidationSet
{
    public function add(
        CacheInvalidationRequest $request,
    ): void;
}
```

---

# 74. Deduplication

Dentro de una transacción:

```text
invalidate User#42
invalidate User#42
invalidate User#42
```

deberá poder compactarse.

---

# 75. Coalescing

También:

```text
User#1
User#2
User#3
...
User#10000
```

puede convertirse en:

```text
EntityType(User)
```

o:

```text
table users generation bump
```

si el costo de exact invalidation supera el beneficio.

---

# 76. Invalidation cost model

El planner podrá considerar:

```text
number of exact keys
number of dependency tags
backend round trips
payload size
region size
consistency requirement
```

---

# 77. Correctness boundary

El cost model podrá ampliar invalidación.

Nunca reducirla por debajo de la evidencia necesaria para mantener la policy de consistencia.

---

# 78. Bulk operations

Especialmente importantes:

```php
User::query()
    ->where('status', 'inactive')
    ->update(['archived' => true]);
```

---

# 79. ORM bypass

Bulk DML puede no cargar entidades en UnitOfWork.

Aun así deberá producir:

```text
SemanticChangeSet
```

para cache invalidation.

---

# 80. Exact affected keys

Si la plataforma devuelve IDs afectados de forma confiable:

```text
KEY_SET
```

puede usarse.

---

# 81. Otherwise

```text
PREDICATE_BOUNDED
```

o:

```text
TABLE_WIDE
```

---

# 82. Raw SQL

`Raw SQL` representa un desafío.

```php
DB::statement(
    'UPDATE users SET status = ? WHERE ...'
);
```

VoltStack puede no entender completamente la semántica.

---

# 83. Raw mutation policy

Opciones:

```text
REQUIRE_DECLARED_DEPENDENCIES
CONSERVATIVE_TABLE_INVALIDATION
DOMAIN_INVALIDATION
BYPASS_WITH_EXPLICIT_UNSAFE_ACKNOWLEDGEMENT
```

---

# 84. Default

Para raw DML detectable:

```text
conservative invalidation
```

será preferible.

---

# 85. Explicit dependencies

API conceptual:

```php
DB::rawMutation($sql)
    ->affectsEntity(User::class)
    ->affectsField('status')
    ->execute();
```

---

# 86. Unsafe raw mutation

Una API que permita saltarse invalidation deberá ser explícita:

```php
->withoutCacheInvalidation(
    UnsafeCacheInvalidationBypass::acknowledge()
)
```

No un boolean casual.

---

# 87. External writers

Otro problema:

```text
VoltStack Application
      ↓
Database
      ↑
Other Application
ETL
Admin SQL
CDC consumer
```

VoltStack no observará automáticamente esos cambios.

---

# 88. Core limitation

> **Cache invalidation local no puede garantizar coherencia frente a escritores externos invisibles.**

---

# 89. Solutions

Podrán integrarse:

```text
CDC
binlog
WAL stream
database triggers
event bus
explicit external invalidation API
short TTL
version validation
```

---

# 90. ExternalChangeFeed

Contrato futuro:

```php
interface ExternalDatabaseChangeFeed
{
    public function subscribe(
        ExternalChangeConsumer $consumer,
    ): void;
}
```

---

# 91. External change normalization

Cambios externos deberán convertirse en:

```text
SemanticChangeSet
```

antes de entrar al sistema de invalidación.

---

# 92. No vendor leakage

Binlog/WAL specifics permanecerán en adapters.

Core consumirá eventos normalizados.

---

# 93. Distributed invalidation

Con varios workers/nodos:

```text
Node A
Node B
Node C
```

una invalidación local puede ser insuficiente.

---

# 94. Invalidation Bus

```php
interface CacheInvalidationBus
{
    public function publish(
        CacheInvalidationEvent $event,
    ): CacheInvalidationReceipt;
}
```

---

# 95. Bus implementations

```text
InProcess
Redis Pub/Sub
Redis Streams
Message Broker
Database Outbox
Custom Plugin
```

---

# 96. Publication ≠ application

Distinguir:

```text
Invalidation Published
```

de:

```text
Invalidation Applied Everywhere
```

---

# 97. Delivery semantics

Backends podrán ofrecer:

```text
AT_MOST_ONCE
AT_LEAST_ONCE
DURABLE_AT_LEAST_ONCE
```

Core no fingirá `EXACTLY_ONCE`.

---

# 98. Idempotency

Invalidation deberá diseñarse como:

```text
idempotent
```

siempre que sea posible.

---

# 99. Duplicate delivery

```text
Invalidate User#42
Invalidate User#42
```

debe ser segura.

---

# 100. Generation events

`BumpGeneration` requiere más cuidado.

No:

```text
G7 → G8
```

mediante “incrementa una vez” si el mismo mensaje puede llegar dos veces.

---

# 101. Idempotent generation token

Preferir:

```text
Set generation to token X
```

o comparar `InvalidationEventId`.

---

# 102. Monotonic sequence

Cuando exista infraestructura:

```text
G17 < G18 < G19
```

permite ignorar eventos antiguos.

---

# 103. Event ordering

Distributed delivery puede producir:

```text
Event 19
Event 18
```

---

# 104. Stale event

Un evento antiguo nunca deberá restaurar una generación anterior.

---

# 105. InvalidationEvent

```php
final readonly class CacheInvalidationEvent
{
    public function __construct(
        public CacheInvalidationEventId $id,
        public CacheInvalidationScope $scope,
        public CacheInvalidationToken $token,
        public CacheInvalidationReason $reason,
        public PersistenceDomain $domain,
        public Instant $occurredAt,
    ) {}
}
```

---

# 106. Event IDs

Deben permitir deduplicación cuando el backend lo requiera.

---

# 107. Event payload

No deberá incluir valores sensibles innecesarios.

Preferir:

```text
dependency identities
generations
regions
hashes
```

---

# 108. Cross-worker local caches

Cada worker podrá mantener:

```text
small local L1.5/L2 cache
```

pero deberá escuchar invalidation bus si requiere coherencia distribuida.

---

# 109. Worker generation snapshot

Otra estrategia:

```text
local entry generation
vs
shared generation
```

---

# 110. Pub/Sub loss

Pub/Sub no durable puede perder mensajes.

Por tanto, para políticas fuertes deberá combinarse con:

```text
generation validation
TTL
durable stream
outbox
```

---

# 111. Outbox architecture

```text
DB Transaction
│
├── Business Mutation
├── Cache Invalidation Record
│
└── COMMIT
       ↓
Outbox Dispatcher
       ↓
Invalidation Bus
       ↓
Consumers
```

---

# 112. Outbox atomicity

La creación del outbox record dentro de la misma DB transaction puede garantizar:

```text
data mutation committed
↔
invalidation intent persisted
```

para ese database resource.

---

# 113. Outbox ≠ immediate invalidation

Existe una ventana:

```text
commit
↓
outbox pending
↓
cache still stale
```

---

# 114. Mitigation

Puede combinarse:

```text
Synchronous Invalidation
+
Durable Outbox Recovery
```

---

# 115. Commit path

```text
COMMIT confirmed
↓
attempt synchronous invalidation
↓
success?
 ├── yes → done
 └── no  → durable recovery remains
```

---

# 116. Outbox before commit

El intent se escribe dentro de la transacción.

La publicación externa ocurre después del commit.

---

# 117. Rollback

Si la transacción hace rollback:

```text
business mutation
+
outbox record
```

ambos desaparecen.

---

# 118. UNKNOWN commit with outbox

Si outcome es UNKNOWN, el sistema puede no saber si el outbox existe.

Conservative invalidation local/direct seguirá siendo útil.

---

# 119. Recovery worker

```text
CacheInvalidationRecoveryWorker
```

podrá reprocesar intents pendientes.

---

# 120. Retry

Fallos de infraestructura de invalidation podrán reintentarse.

---

# 121. Retry ≠ transaction retry

No confundir:

```text
Database Transaction Retry
```

con:

```text
Cache Invalidation Delivery Retry
```

---

# 122. Post-commit retry

Una vez DB committed:

```text
do not replay business transaction
```

solo porque cache invalidation falló.

---

# 123. Retry policy

```php
final readonly class CacheInvalidationRetryPolicy
{
    public function __construct(
        public int $maxAttempts,
        public Duration $initialDelay,
        public BackoffStrategy $backoff,
    ) {}
}
```

---

# 124. Idempotency prerequisite

Retry seguro requiere invalidation operations idempotentes.

---

# 125. Dead-letter

Eventos que excedan retries podrán ir a:

```text
Invalidation Dead Letter Queue
```

o mecanismo equivalente.

---

# 126. Dead-letter handling

Debe producir:

```text
high-severity telemetry
consistency degradation signal
repair action
```

---

# 127. Cache repair

Herramientas:

```text
invalidate region
rebuild generation
replay outbox
compare cache/database versions
```

---

# 128. Repair ≠ normal operation

No deberá utilizarse un full rebuild constante para compensar un sistema de invalidación defectuoso.

---

# 129. Replica lag

Supongamos:

```text
Primary:
User#42 version 18

Replica:
User#42 version 17
```

Después:

```text
invalidate User#42
```

un request puede leer la replica y volver a cachear 17.

---

# 130. Invalidation alone is insufficient

Por tanto:

```text
Invalidation
+
Stale Replica
→ Stale Repopulation
```

---

# 131. Integration

El planner/read cache population deberá cooperar con:

```text
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

---

# 132. Write token

Después de una mutación podrá existir:

```text
MinimumRequiredDataVersion
```

o token equivalente.

---

# 133. Cache repopulation guard

```text
Candidate Version >= Required Version
```

cuando la versión sea comparable.

---

# 134. Unknown replica freshness

Si no puede demostrarse:

```text
safe to repopulate
```

podrá devolverse el dato según read consistency policy, pero no necesariamente almacenarlo en shared cache.

---

# 135. Failover

Tras cambio de primary:

```text
TopologyGeneration
```

podrá participar en cache validity.

---

# 136. Failover uncertainty

Si el nuevo primary puede haber perdido writes:

```text
cache state may be ahead of database
```

---

# 137. Cache-ahead problem

No solo existe:

```text
cache stale behind DB
```

también:

```text
cache newer than authoritative DB after failover
```

---

# 138. Topology generation

Una estrategia:

```text
DatabaseTopologyGeneration
```

incluida en consistency evidence.

---

# 139. Failover invalidation

Dependiendo de guarantees:

```text
no action
domain generation bump
region invalidation
full persistence-domain invalidation
```

---

# 140. Sharding

Invalidation debe mantener:

```text
ShardId
```

como parte del persistence domain.

---

# 141. Shard-local invalidation

```text
Shard 3 User#42
```

no debe invalidar:

```text
Shard 8 User#42
```

salvo que sean semánticamente la misma entidad global.

---

# 142. Resharding

Mover:

```text
Customer#42
Shard 3 → Shard 8
```

requiere invalidar:

```text
old domain
new domain
routing metadata
derived query results
```

---

# 143. Partition routing

Cambios en routing podrán producir:

```text
PartitionRoutingChanged
```

como semantic change.

---

# 144. Multitenancy

Invalidation events serán tenant-aware cuando exista el paquete Multitenancy.

---

# 145. Core decoupling

Core Database no dependerá de una implementación concreta de tenant.

Utilizará:

```text
PersistenceDomain
IsolationDimension
```

---

# 146. Tenant-wide invalidation

Debe ser posible:

```text
invalidate Tenant A catalog
```

sin afectar Tenant B.

---

# 147. SaaS integration

El paquete SaaS podrá registrar dependencies adicionales.

No deberá modificar las invariantes del core.

---

# 148. Relationship invalidation

Cambios de relaciones pueden afectar:

```text
entity state
relationship membership
query results
aggregates
```

---

# 149. Example

```text
Post#9.author:
User#1 → User#2
```

puede invalidar:

```text
Post#9
User#1.posts membership
User#2.posts membership
posts-by-user-1 result
posts-by-user-2 result
```

---

# 150. Owning side

La semántica de Relationship Metadata determinará qué cambio persistente ocurrió.

Invalidation no reinterpretará ownership ORM por su cuenta.

---

# 151. Many-to-many

```text
User#42.roles
```

membership change deberá generar:

```text
RelationshipMembershipChanged
```

independiente de si `Role#7` cambió internamente.

---

# 152. Membership ≠ target state

Regla:

```text
Role#7 changed
≠
User#42.roles membership changed
```

aunque caches que materialicen target data puedan depender de ambos.

---

# 153. Aggregates

Un cache:

```text
COUNT(User.roles)
```

depende de membership.

No necesariamente de:

```text
Role.name
```

---

# 154. Query dependency extraction

El Query/Semantic Engine podrá producir:

```text
QueryDependencyDescriptor
```

durante compilación.

---

# 155. Example

```php
User::query()
    ->where('status', 'active')
    ->count();
```

Dependency descriptor:

```text
EntityType: User
Field: User.status
Operation: COUNT
```

---

# 156. Result Cache registration

Al almacenar el resultado:

```text
ResultCacheEntry
+
DependencyDescriptor
```

---

# 157. Predicate dependency

Inicialmente VoltStack puede utilizar:

```text
field-level dependencies
```

sin intentar resolver matemáticamente todos los predicados.

---

# 158. Future precision

Posteriormente:

```text
PredicateDependency
```

podrá permitir invalidar solo cuando:

```text
old/new value crosses predicate membership
```

---

# 159. Example

Query:

```text
WHERE age >= 18
```

Cambio:

```text
age 20 → 21
```

membership no cambia.

Cambio:

```text
17 → 18
```

sí.

---

# 160. Predicate transition analyzer

Extensión futura:

```php
interface PredicateTransitionAnalyzer
{
    public function affectsMembership(
        QueryPredicate $predicate,
        FieldTransition $transition,
    ): Trilean;
}
```

---

# 161. Trilean

```text
YES
NO
UNKNOWN
```

---

# 162. UNKNOWN

```text
UNKNOWN
→ invalidate
```

para política conservadora.

---

# 163. Query functions

Predicados con:

```text
NOW()
RAND()
external functions
collation-dependent operations
custom DB functions
```

pueden impedir análisis exacto.

---

# 164. Time-dependent cache

Resultados dependientes del tiempo requieren:

```text
TTL
temporal dependency
explicit non-cacheable status
```

además de invalidation por writes.

---

# 165. Non-deterministic query

Podrá ser:

```text
NON_CACHEABLE
```

por default.

---

# 166. Metadata invalidation

Cambios en mapping deberán incrementar:

```text
MetadataGeneration
```

---

# 167. Metadata generation change

Puede hacer obsoletos:

```text
Entity Cache entries
Hydration plans
Query plans
Result cache decoding
```

---

# 168. Codec generation

Cambiar encoding de cache también requiere invalidación lógica.

---

# 169. Generation composition

```text
CacheArtifactGeneration
=
MetadataGeneration
+
SchemaGeneration?
+
CodecGeneration
+
PolicyGeneration
```

dependiendo del artefacto.

---

# 170. No universal generation

No todos los caches deberán depender de un único contador global.

Eso produciría invalidación excesiva.

---

# 171. Hierarchical generations

Preferir:

```text
Application Generation
 ├── Metadata Generation
 ├── Schema Generation
 ├── Entity Region Generations
 ├── Query Region Generations
 └── Tenant/Shard Generations
```

---

# 172. Parent generation

Un cambio de parent puede invalidar todos los descendants.

---

# 173. Region generation

```text
catalog:G18
```

permite invalidar una región completa eficientemente.

---

# 174. Garbage collection

Logical invalidation deja entries físicamente almacenados.

Deberán eliminarse mediante:

```text
TTL
LRU/backend eviction
generation garbage collector
background cleanup
```

---

# 175. GC ≠ correctness

Garbage collection libera recursos.

No determina validez semántica.

---

# 176. Invalidation token

```php
final readonly class CacheInvalidationToken
{
    public function __construct(
        public string $value,
    ) {}
}
```

Podrá ser:

```text
monotonic sequence
ULID
generation hash
version token
```

según backend.

---

# 177. Token ≠ timestamp

No utilizar únicamente wall-clock timestamps para ordering distribuido.

---

# 178. Clock skew

```text
Node A 12:00:01
Node B 11:59:59
```

puede romper orden basado solo en reloj.

---

# 179. Monotonic ordering

Cuando ordering sea necesario, usar fuente adecuada:

```text
database sequence
cache atomic counter
stream offset
logical generation
```

---

# 180. Invalidation ordering

No todas las invalidaciones requieren orden total global.

Preferir:

```text
ordering per dependency/domain
```

cuando sea suficiente.

---

# 181. Scalability

Un total-order global puede convertirse en bottleneck.

---

# 182. Invalidation batching

```php
$invalidator->invalidateMany($targets);
```

deberá ser soportado.

---

# 183. Backend round trips

```text
1000 invalidations
```

no deberán implicar obligatoriamente:

```text
1000 network requests
```

---

# 184. Batch limits

Aun así:

```text
max batch keys
max payload bytes
max operation duration
```

deberán ser bounded.

---

# 185. Backpressure

Si el invalidation bus está saturado:

```text
queue indefinitely
```

no será aceptable.

---

# 186. Backpressure policy

```text
BLOCK_BOUNDED
DEFER_DURABLY
FAIL_AND_RECOVER
ESCALATE
```

---

# 187. Never silently drop

Para invalidaciones requeridas por consistency policy:

```text
silent drop
```

está prohibido.

---

# 188. Best-effort caches

Algunas caches puramente de performance podrán usar garantías más débiles.

Eso deberá declararse explícitamente.

---

# 189. Invalidation consistency requirement

```php
enum CacheInvalidationConsistencyRequirement
{
    case BEST_EFFORT;
    case EVENTUAL_DURABLE;
    case SYNCHRONOUS_LOCAL;
    case SYNCHRONOUS_DISTRIBUTED;
}
```

El documento 192 definirá con mayor precisión el modelo de consistencia.

---

# 190. Synchronous distributed caveat

Incluso:

```text
SYNCHRONOUS_DISTRIBUTED
```

debe definir qué significa:

```text
all known nodes?
quorum?
shared backend mutation?
acknowledged consumers?
```

No usar términos ambiguos.

---

# 191. CacheInvalidationReceipt

```php
final readonly class CacheInvalidationReceipt
{
    public function __construct(
        public CacheInvalidationEventId $eventId,
        public CacheInvalidationDeliveryStatus $status,
        public CacheInvalidationEvidence $evidence,
    ) {}
}
```

---

# 192. Receipt ≠ global truth

Un receipt de publicación no prueba que cada proceso del mundo haya aplicado el evento.

---

# 193. Delivery statuses

```text
APPLIED
PUBLISHED
DURABLY_QUEUED
PARTIALLY_APPLIED
FAILED
UNKNOWN
```

---

# 194. UNKNOWN delivery

Deberá conservarse como UNKNOWN.

---

# 195. Fail closed?

No siempre significa fallar la operación de negocio, especialmente post-commit.

Pero sí:

```text
mark cache consistency degraded
+
schedule recovery
+
emit telemetry
```

---

# 196. Pre-commit cache failure

Una invalidation que todavía no debería publicarse no deberá abortar necesariamente una transacción simplemente porque el cache esté caído.

---

# 197. Post-commit responsibility

Después de commit:

```text
business correctness
```

y:

```text
cache consistency recovery
```

son responsabilidades relacionadas pero distintas.

---

# 198. Cache dependency lifecycle

```text
Artifact Created
↓
Dependencies Registered
↓
Artifact Valid
↓
Semantic Change
↓
Invalidated
↓
Eventually Evicted/Replaced
```

---

# 199. Atomic artifact registration

Evitar:

```text
store artifact
↓
crash
↓
dependencies never registered
```

si el sistema depende de reverse dependency indexes.

---

# 200. Registration ordering

Puede preferirse:

```text
build payload
register dependency generation snapshot
store artifact with snapshot atomically/consistently
```

según backend.

---

# 201. Generation snapshot advantage

Si dependencies están dentro del entry:

```text
dependency generations at creation
```

no requiere reverse registration completa.

---

# 202. Read validation

```text
load entry
↓
compare dependency generations
↓
valid?
```

---

# 203. Cost

Puede requerir múltiples generation lookups.

---

# 204. Generation vector

```php
final readonly class CacheDependencyGenerationVector
{
    /**
     * @param array<string, string|int> $generations
     */
    public function __construct(
        public array $generations,
    ) {}
}
```

---

# 205. Vector size

Debe ser bounded.

Queries con miles de dependencies podrán degradarse a dependencias más amplias.

---

# 206. Dependency compaction

Ejemplo:

```text
User#1
User#2
...
User#50000
```

puede compactarse a:

```text
User entity-type generation
```

---

# 207. Compaction tradeoff

```text
less metadata
+
more invalidation
```

---

# 208. Planner policy

```php
interface CacheDependencyCompactionPolicy
{
    public function compact(
        CacheDependencySet $dependencies,
    ): CacheDependencySet;
}
```

---

# 209. Cache invalidation scopes

```php
enum CacheInvalidationScope
{
    case ARTIFACT;
    case ENTITY;
    case RELATIONSHIP;
    case ENTITY_TYPE;
    case TABLE;
    case REGION;
    case TENANT;
    case SHARD;
    case DATABASE;
    case APPLICATION;
}
```

---

# 210. Escalation monotonicity

Si el planner escala:

```text
ENTITY
→ ENTITY_TYPE
```

no deberá posteriormente afirmar que solo invalidó la entidad exacta.

---

# 211. Invalidation evidence

Diagnostics deberán conservar:

```text
requested scope
effective scope
reason for escalation
```

---

# 212. Example

```text
Requested:
    14,327 exact User IDs

Backend budget:
    max exact invalidations = 5,000

Effective:
    User entity-type generation bump

Reason:
    dependency compaction threshold exceeded
```

---

# 213. Security

Invalidation infrastructure no deberá convertirse en canal de fuga de:

- IDs sensibles;
- tenant names;
- query parameters;
- personal data;
- credentials.

---

# 214. Hashed dependency keys

Podrán utilizarse cuando sea necesario.

---

# 215. Authorization of administrative invalidation

Comandos:

```text
invalidate tenant
invalidate database
invalidate application
```

deberán requerir permisos administrativos.

---

# 216. User input

No deberá permitirse que input HTTP arbitrario construya dependency keys internas sin validación.

---

# 217. Cache poisoning

Un actor no deberá poder publicar:

```text
generation = old generation
```

y revalidar entries stale.

---

# 218. Generation monotonicity

Una generation invalidation solo podrá avanzar.

```text
G18
→ G17
```

prohibido.

---

# 219. Event authenticity

Distributed invalidation podrá requerir:

```text
authenticated channel
signed messages
trusted broker
ACLs
```

según deployment.

---

# 220. Replay attacks

Eventos viejos reinyectados deberán ser ignorables mediante:

```text
event ID
generation ordering
domain epoch
```

---

# 221. Telemetry architecture

```text
CacheInvalidationObserved
CacheInvalidationPlanned
CacheInvalidationDeferred
CacheInvalidationPublished
CacheInvalidationApplied
CacheInvalidationEscalated
CacheInvalidationRetryScheduled
CacheInvalidationFailed
CacheInvalidationRecovered
CacheInvalidationUnknown
```

---

# 222. Metrics

```text
db.cache.invalidation.requests
db.cache.invalidation.targets
db.cache.invalidation.duration
db.cache.invalidation.escalations
db.cache.invalidation.failures
db.cache.invalidation.retries
db.cache.invalidation.recovery
db.cache.invalidation.unknown
db.cache.invalidation.generation_bumps
```

---

# 223. Labels

Bounded:

```text
cache_type
region
operation
scope
reason
result
backend
```

---

# 224. No raw IDs

No:

```text
entity_id
tenant_id
query_parameter
```

como labels de alta cardinalidad.

---

# 225. Tracing

Span conceptual:

```text
database.cache.invalidation
```

con:

```text
plan id
scope
target count
strategy
result
```

sanitizados.

---

# 226. Diagnostics API

```php
DB::cache()
    ->invalidation()
    ->explain($changeSet);
```

---

# 227. Example diagnostic

```text
CACHE INVALIDATION PLAN

Change:
    EntityUpdated

Entity:
    App\Domain\User

Changed Fields:
    status

Precision:
    ENTITY_EXACT

Dependencies:
    Entity(User#...)
    Field(User.status)
    EntityType(User)

Affected:
    Entity Cache: 1
    Result Cache Tags: 3
    Query Cache: 0
    Metadata Cache: 0

Transaction:
    ACTIVE

Publication:
    DEFERRED_UNTIL_COMMIT

Strategy:
    exact entity delete
    field generation bump

Escalation:
    NONE
```

---

# 228. Explain without execution

Debe ser posible generar un plan diagnóstico sin aplicar invalidation.

---

# 229. Dry run

CLI:

```bash
php voltstack database:cache:invalidate \
    --entity=User \
    --id=42 \
    --dry-run
```

---

# 230. CLI

Posibles comandos:

```bash
php voltstack database:cache:invalidation:status
php voltstack database:cache:invalidation:replay
php voltstack database:cache:invalidation:failed
php voltstack database:cache:invalidate:entity
php voltstack database:cache:invalidate:region
php voltstack database:cache:invalidate:tenant
php voltstack database:cache:invalidate:database
```

---

# 231. Dangerous operations

```text
application-wide invalidation
```

deberá requerir confirmación explícita en CLI interactiva o flag equivalente.

---

# 232. Persistent runtime

FrankenPHP, RoadRunner y OpenSwoole hacen especialmente importante la invalidación.

---

# 233. Traditional PHP request

En PHP-FPM muchos caches locales desaparecen al finalizar request.

---

# 234. Persistent worker

En:

```text
FrankenPHP Worker
RoadRunner
OpenSwoole
```

el estado local puede sobrevivir.

---

# 235. Therefore

Una invalidation distribuida deberá alcanzar:

```text
shared backend
+
worker-local reusable caches
```

cuando estos últimos existan.

---

# 236. Request-scoped state

No deberá invalidarse globalmente:

```text
current request IdentityMap
```

como si fuera L2.

---

# 237. Managed dirty entity

Si otro proceso invalida `User#42` mientras este scope posee `User#42` dirty:

```text
do not overwrite managed entity
```

---

# 238. External invalidation signal

El EntityManager podrá recibir una marca:

```text
externally stale
```

para decisiones futuras de refresh/commit.

---

# 239. But no async mutation

No deberá modificarse arbitrariamente el objeto administrado en mitad de código de aplicación debido a un invalidation event.

---

# 240. Isolation

```text
shared cache invalidation
```

no será:

```text
live object synchronization
```

---

# 241. Worker event handling

El handler de invalidation deberá ser:

```text
bounded
non-reentrant where necessary
concurrency-safe
```

---

# 242. OpenSwoole

Un invalidation handler no deberá manipular:

```text
another coroutine's UnitOfWork
```

---

# 243. Local cache generation

Preferir invalidar estructuras shared mediante:

```text
generation/token
```

en vez de recorrer objetos runtime de scopes activos.

---

# 244. Memory safety

Dependency registries locales deberán tener:

```text
bounds
TTL
eviction
generation cleanup
```

---

# 245. No unbounded reverse graph

Una aplicación con millones de cache keys no deberá mantener necesariamente:

```text
millions × dependencies
```

en RAM del worker.

---

# 246. Cache backend capabilities

```php
interface CacheInvalidationCapabilities
{
    public function supportsTags(): bool;

    public function supportsAtomicGeneration(): bool;

    public function supportsBatchDelete(): bool;

    public function supportsPubSub(): bool;

    public function supportsDurableStreams(): bool;

    public function supportsCompareAndSet(): bool;
}
```

---

# 247. Capability-driven design

No:

```php
if ($cache instanceof Redis) {
}
```

en core.

Preferir:

```text
capability negotiation
```

---

# 248. Unsupported capability

El planner deberá seleccionar otra estrategia.

Ejemplo:

```text
tags unsupported
→ generation namespace
```

---

# 249. No silent weakening

Si la estrategia alternativa no cumple consistency requirement:

```text
reject / degrade explicitly
```

Nunca fingir equivalencia.

---

# 250. Null cache

Con cache deshabilitado:

```text
InvalidationPlan
```

podrá convertirse en no-op explícito.

---

# 251. No-op evidence

Diagnostics:

```text
NO_CACHE_BACKEND_ENABLED
```

No fingir `APPLIED`.

---

# 252. Testing architecture

Pruebas requeridas:

```text
semantic change capture
dependency resolution
planning
transaction ordering
generation invalidation
distributed propagation
retry
outbox
replica lag
sharding
multitenancy
persistent workers
security
failure modes
performance
```

---

# 253. Exact entity test

```text
User#42 updated
→ User#42 cache invalidated
→ User#43 remains valid
```

---

# 254. Field dependency test

```text
User.avatar changed
```

no deberá invalidar un aggregate que solo dependa de:

```text
User.status
```

cuando la dependencia sea exacta.

---

# 255. Unknown dependency test

```text
UNKNOWN
→ conservative invalidation
```

---

# 256. Transaction rollback test

```text
mutation
→ deferred invalidation
→ rollback
→ no global invalidation publication
```

---

# 257. Transaction commit test

```text
mutation
→ deferred
→ commit confirmed
→ publish
```

---

# 258. Unknown commit test

```text
UNKNOWN
→ conservative invalidation
```

sin publicar new cache value.

---

# 259. Duplicate event test

Aplicar el mismo invalidation event dos veces deberá ser seguro.

---

# 260. Out-of-order test

```text
G19
then
G18
```

deberá conservar `G19`.

---

# 261. Retry test

Fallos transitorios no duplicarán efectos incorrectos.

---

# 262. Outbox rollback test

Rollback elimina mutation + outbox intent.

---

# 263. Outbox recovery test

Crash después de commit pero antes de publicación podrá recuperarse.

---

# 264. Raw DML test

Raw mutation no deberá omitir invalidation silenciosamente.

---

# 265. Bulk update test

Bulk update deberá escalar correctamente cuando IDs afectados sean desconocidos.

---

# 266. Replica stale repopulation test

Replica vieja no deberá revalidar cache stale cuando existe version evidence más nueva.

---

# 267. Tenant test

Tenant A invalidation no afectará Tenant B.

---

# 268. Shard test

Shard A invalidation no afectará Shard B salvo intención explícita.

---

# 269. Resharding test

Old/new domains se invalidarán.

---

# 270. Worker test

Worker B dejará de usar entry local invalidado por Worker A.

---

# 271. Pub/Sub loss test

Policy fuerte deberá detectar/mitigar pérdida mediante generation/durable mechanism.

---

# 272. Cache outage test

Invalidation failure deberá producir estado de degradación observable.

---

# 273. Security test

Eventos falsificados/antiguos no podrán retroceder generations.

---

# 274. Resource test

Dependency sets enormes deberán compactarse o rechazarse según policy.

---

# 275. Performance test

Medir:

```text
exact delete
batch delete
generation bump
tag invalidation
dependency resolution
plan construction
event publication
event consumption
```

---

# 276. Directory structure

```text
src/Quantum/Database/Cache/Invalidation/
│
├── CacheInvalidationManager.php
├── CacheInvalidationPlanner.php
├── CacheInvalidationRequest.php
├── CacheInvalidationPlan.php
├── CacheInvalidationOperation.php
├── CacheInvalidationTarget.php
├── CacheInvalidationScope.php
├── CacheInvalidationReason.php
├── CacheInvalidationReceipt.php
│
├── Change/
│   ├── SemanticChange.php
│   ├── SemanticChangeSet.php
│   ├── SemanticChangeKind.php
│   ├── SemanticChangePrecision.php
│   ├── EntityInserted.php
│   ├── EntityUpdated.php
│   ├── EntityDeleted.php
│   ├── EntityFieldChanged.php
│   ├── RelationshipMembershipChanged.php
│   ├── SchemaChanged.php
│   └── PartitionRoutingChanged.php
│
├── Dependency/
│   ├── CacheDependency.php
│   ├── CacheDependencyKey.php
│   ├── CacheDependencySet.php
│   ├── CacheDependencyRegistry.php
│   ├── CacheDependencyKeyFactory.php
│   ├── CacheDependencyGenerationVector.php
│   ├── EntityDependency.php
│   ├── EntityFieldDependency.php
│   ├── EntityTypeDependency.php
│   ├── RelationshipDependency.php
│   ├── TableDependency.php
│   └── SchemaDependency.php
│
├── Generation/
│   ├── CacheInvalidationGeneration.php
│   ├── CacheInvalidationToken.php
│   ├── CacheGenerationStore.php
│   ├── CacheGenerationValidator.php
│   └── CacheGenerationGarbageCollector.php
│
├── Planning/
│   ├── DefaultCacheInvalidationPlanner.php
│   ├── CacheInvalidationPlanningContext.php
│   ├── CacheDependencyResolver.php
│   ├── CacheDependencyCompactionPolicy.php
│   ├── CacheInvalidationCostModel.php
│   └── CacheInvalidationEscalationPolicy.php
│
├── Operation/
│   ├── DeleteArtifactOperation.php
│   ├── DeleteEntityOperation.php
│   ├── InvalidateTagOperation.php
│   ├── BumpGenerationOperation.php
│   ├── InvalidateRegionOperation.php
│   └── InvalidateDomainOperation.php
│
├── Transaction/
│   ├── DeferredCacheInvalidationSet.php
│   ├── TransactionCacheInvalidationCoordinator.php
│   ├── CacheInvalidationCommitListener.php
│   └── UnknownTransactionInvalidationPolicy.php
│
├── Distribution/
│   ├── CacheInvalidationBus.php
│   ├── CacheInvalidationEvent.php
│   ├── CacheInvalidationConsumer.php
│   ├── CacheInvalidationDeliveryStatus.php
│   ├── InProcessInvalidationBus.php
│   └── DistributedInvalidationAdapter.php
│
├── Outbox/
│   ├── CacheInvalidationOutbox.php
│   ├── CacheInvalidationOutboxRecord.php
│   ├── CacheInvalidationOutboxWriter.php
│   ├── CacheInvalidationOutboxDispatcher.php
│   └── CacheInvalidationRecoveryWorker.php
│
├── Retry/
│   ├── CacheInvalidationRetryPolicy.php
│   ├── CacheInvalidationRetryScheduler.php
│   └── CacheInvalidationDeadLetterHandler.php
│
├── External/
│   ├── ExternalDatabaseChangeFeed.php
│   ├── ExternalDatabaseChange.php
│   └── ExternalChangeNormalizer.php
│
├── Telemetry/
│   ├── CacheInvalidationTelemetry.php
│   ├── CacheInvalidationObservedEvent.php
│   ├── CacheInvalidationAppliedEvent.php
│   ├── CacheInvalidationFailedEvent.php
│   └── CacheInvalidationRecoveredEvent.php
│
├── Diagnostics/
│   ├── CacheInvalidationInspector.php
│   ├── CacheInvalidationExplainer.php
│   └── CacheInvalidationDiagnosticReport.php
│
└── Exception/
    ├── CacheInvalidationException.php
    ├── CacheInvalidationPlanningException.php
    ├── CacheInvalidationDependencyException.php
    ├── CacheInvalidationDeliveryException.php
    ├── CacheInvalidationGenerationException.php
    ├── CacheInvalidationSecurityException.php
    ├── CacheInvalidationResourceLimitException.php
    └── CacheInvalidationInvariantViolationException.php
```

---

# 277. Architectural invariants

## DB-CINV-001
Cache invalidation partirá de cambios semánticos.

## DB-CINV-002
SQL strings no serán la fuente primaria de semántica de invalidación.

## DB-CINV-003
Invalidation no será equivalente a cache clear.

## DB-CINV-004
Global clear no será estrategia normal.

## DB-CINV-005
SemanticChangeSet será explícito.

## DB-CINV-006
UNKNOWN change precision no significará safe.

## DB-CINV-007
La incertidumbre ampliará conservadoramente el scope.

## DB-CINV-008
False-positive invalidation será preferible a false-negative invalidation.

## DB-CINV-009
Dependencies serán semánticas.

## DB-CINV-010
Dependency keys serán tipadas.

## DB-CINV-011
Dependency keys estarán namespaced.

## DB-CINV-012
Tenant domains no colisionarán.

## DB-CINV-013
Shard domains no colisionarán.

## DB-CINV-014
CacheInvalidationRequest será distinto de Plan.

## DB-CINV-015
Planner no ejecutará DB mutations.

## DB-CINV-016
Plan será explícito e inspeccionable.

## DB-CINV-017
Physical deletion será distinta de logical invalidation.

## DB-CINV-018
Generation mismatch significará stale.

## DB-CINV-019
Generations podrán evitar reverse indexes masivos.

## DB-CINV-020
Generations tendrán garbage collection independiente.

## DB-CINV-021
GC no determinará correctness.

## DB-CINV-022
Entity Cache podrá usar exact invalidation.

## DB-CINV-023
Result Cache podrá usar dependency generations.

## DB-CINV-024
Metadata Cache no será invalidada por cualquier row mutation.

## DB-CINV-025
Query compilation cache no será invalidada por cualquier data mutation.

## DB-CINV-026
Schema changes podrán invalidar múltiples cache classes.

## DB-CINV-027
Transaction-derived invalidation será deferred.

## DB-CINV-028
Flush no publicará confirmed invalidation.

## DB-CINV-029
Commit confirmado permitirá publication.

## DB-CINV-030
Rollback confirmado descartará pending invalidation.

## DB-CINV-031
UNKNOWN commit no se convertirá en COMMITTED.

## DB-CINV-032
UNKNOWN commit favorecerá conservative invalidation.

## DB-CINV-033
UNKNOWN commit no publicará speculative new cache values.

## DB-CINV-034
Deferred invalidations serán transaction-scoped.

## DB-CINV-035
Duplicate deferred invalidations podrán compactarse.

## DB-CINV-036
Compaction podrá ampliar scope.

## DB-CINV-037
Compaction no podrá debilitar correctness.

## DB-CINV-038
Bulk DML participará en invalidation.

## DB-CINV-039
Bulk DML no requerirá cargar entidades para invalidar cache.

## DB-CINV-040
Raw DML no omitirá invalidation silenciosamente.

## DB-CINV-041
Unsafe invalidation bypass será explícito.

## DB-CINV-042
External writers serán una limitación explícita.

## DB-CINV-043
External change feeds serán normalizados a SemanticChangeSet.

## DB-CINV-044
Core no dependerá de binlog/WAL vendor specifics.

## DB-CINV-045
Distributed invalidation tendrá contrato explícito.

## DB-CINV-046
Published será distinto de Applied.

## DB-CINV-047
Exactly-once no será fingido.

## DB-CINV-048
Invalidation será idempotente cuando sea posible.

## DB-CINV-049
Duplicate events no restaurarán stale data.

## DB-CINV-050
Generation events soportarán deduplicación.

## DB-CINV-051
Old events no reducirán generation.

## DB-CINV-052
Ordering global no será requerido innecesariamente.

## DB-CINV-053
Per-domain ordering será preferible cuando baste.

## DB-CINV-054
Pub/Sub no durable no será suficiente para todas las policies.

## DB-CINV-055
Strong policies requerirán durable/generation safeguards.

## DB-CINV-056
Outbox podrá persistir invalidation intent atómicamente con DB mutation.

## DB-CINV-057
Outbox publication ocurrirá post-commit.

## DB-CINV-058
Rollback eliminará outbox intent transaccional.

## DB-CINV-059
Outbox no implicará immediate cache coherence.

## DB-CINV-060
Synchronous invalidation podrá combinarse con durable outbox.

## DB-CINV-061
Cache invalidation retry será distinto de transaction retry.

## DB-CINV-062
Post-commit invalidation failure no reejecutará business transaction.

## DB-CINV-063
Retries requerirán idempotency.

## DB-CINV-064
Dead-letter será observable.

## DB-CINV-065
Failed invalidations tendrán recovery path.

## DB-CINV-066
Replica lag será parte de stale repopulation analysis.

## DB-CINV-067
Invalidation sola no resolverá stale replica repopulation.

## DB-CINV-068
Version evidence podrá bloquear stale repopulation.

## DB-CINV-069
Unknown replica freshness no será asumida current.

## DB-CINV-070
Failover podrá cambiar cache validity.

## DB-CINV-071
Cache puede estar behind o ahead del DB.

## DB-CINV-072
Topology generation podrá participar en validity.

## DB-CINV-073
Failover no significará siempre global cache flush.

## DB-CINV-074
Sharding será parte del persistence domain.

## DB-CINV-075
Shard-local invalidation permanecerá aislada.

## DB-CINV-076
Resharding invalidará old y new domains cuando corresponda.

## DB-CINV-077
Partition routing changes serán semantic changes.

## DB-CINV-078
Multitenancy integration no introducirá dependencia obligatoria en core.

## DB-CINV-079
Tenant-wide invalidation será aislada.

## DB-CINV-080
Relationship membership será dependency propia.

## DB-CINV-081
Relationship membership será distinta de target entity state.

## DB-CINV-082
Query Engine podrá producir dependency descriptors.

## DB-CINV-083
Result Cache registrará dependencies.

## DB-CINV-084
Predicate analysis podrá ser incremental.

## DB-CINV-085
UNKNOWN predicate impact invalidará conservadoramente.

## DB-CINV-086
Non-deterministic queries podrán ser non-cacheable.

## DB-CINV-087
Time-dependent results requerirán temporal policy.

## DB-CINV-088
Metadata generation será explícita.

## DB-CINV-089
Codec generation podrá invalidar payloads.

## DB-CINV-090
No existirá necesariamente una única global generation.

## DB-CINV-091
Hierarchical generations serán soportables.

## DB-CINV-092
Parent generation podrá invalidar descendants.

## DB-CINV-093
Region generation será explícita.

## DB-CINV-094
Logical invalidation no requerirá physical deletion inmediata.

## DB-CINV-095
Timestamps solos no definirán distributed ordering.

## DB-CINV-096
Clock skew será considerado.

## DB-CINV-097
Batch invalidation será soportada.

## DB-CINV-098
Batch size será bounded.

## DB-CINV-099
Backpressure será explícita.

## DB-CINV-100
Required invalidations no se descartarán silenciosamente.

## DB-CINV-101
Best-effort policy será explícita.

## DB-CINV-102
Synchronous distributed tendrá semántica definida.

## DB-CINV-103
Receipt no será prueba de universal application.

## DB-CINV-104
UNKNOWN delivery seguirá siendo UNKNOWN.

## DB-CINV-105
Post-commit invalidation failure degradará cache consistency, no DB outcome.

## DB-CINV-106
Dependency registration tendrá lifecycle definido.

## DB-CINV-107
Generation snapshots podrán sustituir reverse indexes.

## DB-CINV-108
Dependency vectors serán bounded.

## DB-CINV-109
Large dependency sets podrán compactarse.

## DB-CINV-110
Compaction tradeoff será explícito.

## DB-CINV-111
Escalation será monotónica.

## DB-CINV-112
Diagnostics mostrarán requested y effective scope.

## DB-CINV-113
Invalidation infrastructure protegerá datos sensibles.

## DB-CINV-114
Administrative invalidation requerirá autorización.

## DB-CINV-115
Untrusted input no construirá internal dependency keys libremente.

## DB-CINV-116
Generations nunca retrocederán.

## DB-CINV-117
Distributed channels podrán requerir autenticación.

## DB-CINV-118
Replay de eventos antiguos será detectable.

## DB-CINV-119
Telemetry evitará high-cardinality IDs.

## DB-CINV-120
Explain podrá ejecutarse sin invalidar.

## DB-CINV-121
Persistent workers recibirán invalidation para reusable local caches.

## DB-CINV-122
IdentityMap no será tratado como distributed L2 cache.

## DB-CINV-123
External invalidation no sobrescribirá managed dirty objects.

## DB-CINV-124
Invalidation no será live object synchronization.

## DB-CINV-125
OpenSwoole no manipulará UoW de otra coroutine.

## DB-CINV-126
Worker-local dependency state será bounded.

## DB-CINV-127
Reverse dependency graph no crecerá sin límites.

## DB-CINV-128
Backend behavior será capability-driven.

## DB-CINV-129
Core no tendrá `instanceof Redis` como arquitectura.

## DB-CINV-130
Unsupported capabilities tendrán fallback explícito.

## DB-CINV-131
Fallback no debilitará consistency silenciosamente.

## DB-CINV-132
Null cache producirá no-op explícito.

## DB-CINV-133
No-op no fingirá applied invalidation.

## DB-CINV-134
Cache invalidation será independiente de SQL Compiler.

## DB-CINV-135
Cache invalidation no ejecutará SQL.

## DB-CINV-136
Cache invalidation no hará commit.

## DB-CINV-137
Cache invalidation no hará rollback.

## DB-CINV-138
Cache invalidation no decidirá entity ownership ORM.

## DB-CINV-139
Cache invalidation consumirá semantic evidence de los subsistemas apropiados.

## DB-CINV-140
Database truth y cache truth nunca serán asumidas idénticas sin evidence.

## DB-CINV-141
Invalidation será targeted siempre que la evidencia lo permita.

## DB-CINV-142
Correctness tendrá prioridad sobre hit ratio.

## DB-CINV-143
Global flush será herramienta administrativa, no primitive normal.

## DB-CINV-144
Invalidation failure será observable.

## DB-CINV-145
Recovery será parte de la arquitectura.

## DB-CINV-146
Invalidation intent podrá sobrevivir process crashes cuando la policy lo requiera.

## DB-CINV-147
Cache invalidation no será una distributed transaction implícita.

## DB-CINV-148
Database/cache 2PC no será requerido por core.

## DB-CINV-149
Consistency guarantees dependerán de evidencia real.

## DB-CINV-150
El sistema nunca declarará una cache coherente cuando el estado de invalidation sea UNKNOWN.

---

# 278. Modelo final

```text
                   DATABASE CHANGE
                         │
                         ▼
                 SemanticChangeSet
                         │
                         ▼
                Dependency Resolver
                         │
                         ▼
                Invalidation Planner
                         │
                         ▼
                Invalidation Plan
                         │
                ┌────────┴────────┐
                │                 │
          Transactional       Non-Transactional
                │                 │
                ▼                 ▼
             Deferred          Execute
                │
        ┌───────┴────────┐
        │                │
   COMMIT confirmed   ROLLBACK
        │                │
        ▼                ▼
     Publish           Discard
        │
        ▼
  Invalidation Bus
        │
 ┌──────┼────────┬───────────┐
 ▼      ▼        ▼           ▼
Entity Result  Metadata    Local/
Cache  Cache   Cache       Distributed
                         Cache Regions
```

---

# 279. Dependency formula

Para un artefacto `A`:

```text
Dependencies(A) = {D1, D2, ... Dn}
```

y un cambio `C`:

```text
Affected(A, C)
=
Intersects(
    Dependencies(A),
    SemanticImpact(C)
)
```

Cuando:

```text
Affected = YES
```

se invalida.

Cuando:

```text
Affected = NO
```

puede conservarse.

Cuando:

```text
Affected = UNKNOWN
```

la policy conservadora invalida.

Por tanto:

```text
YES     → INVALIDATE
NO      → KEEP
UNKNOWN → INVALIDATE
```

para la policy segura predeterminada.

---

# 280. Transaction formula

```text
Mutation
→ ChangeSet
→ InvalidationPlan
```

mientras la transacción está activa:

```text
InvalidationPlan
→ DEFERRED
```

Después:

```text
COMMITTED
→ PUBLISH
```

```text
ROLLED_BACK
→ DISCARD
```

```text
UNKNOWN
→ CONSERVATIVE_INVALIDATION
```

---

# 281. Distributed formula

```text
Committed Change
      ↓
Invalidation Event
      ↓
Durable/Synchronous Publication
      ↓
Consumers
      ↓
Generation/Delete
      ↓
Old Cache Entries Become Invalid
```

pero:

```text
Published
≠
Applied Everywhere
```

y:

```text
Cache Backend Updated
≠
Every Worker Has Observed Update
```

Estas diferencias deberán permanecer explícitas.

---

# 282. Relación con los sistemas de caché

```text
Database Cache Architecture
│
├── Query Cache
│      └── structural/schema dependencies
│
├── Result Cache
│      └── data/query dependencies
│
├── Metadata Cache
│      └── mapping/generation dependencies
│
└── Entity Cache
       └── canonical entity dependencies

                │
                ▼

       Cache Invalidation System

                │
                ▼

       Cache Consistency System
```

El sistema de invalidación responde:

> **¿Qué deja de ser válido y cómo notificamos ese hecho?**

El sistema de consistencia responderá:

> **¿Qué garantías puede afirmar VoltStack sobre lo que un lector observa antes, durante y después de esos cambios?**

---

# 283. Regla maestra final

La arquitectura definitiva será:

```text
Database Mutation
        ↓
Semantic Change
        ↓
Dependency Resolution
        ↓
Targeted Invalidation
        ↓
Transaction Outcome
        ↓
Safe Publication
        ↓
Distributed Propagation
        ↓
Consistency Evidence
```

Nunca:

```text
Database Mutation
        ↓
FLUSH ALL
```

como estrategia ordinaria.

Y nunca:

```text
Invalidation Failed
        ↓
Pretend Cache Is Consistent
```

La regla fundamental queda definida como:

> **VoltStack nunca considerará válida una caché únicamente porque el entry exista; la validez dependerá de que sus dependencias, generaciones, dominio de persistencia y evidencia de invalidación continúen siendo compatibles con la política de consistencia solicitada.**

Formalmente:

```text
CacheValidity(A)
=
ArtifactExists(A)
∧ DependencyStateValid(A)
∧ GenerationValid(A)
∧ DomainValid(A)
∧ MetadataCompatible(A)
∧ ConsistencyEvidenceSufficient(A)
```

Por tanto:

```text
Exists
≠
Valid
```

y:

```text
Invalidated
≠
PhysicallyDeleted
```

Esta separación permitirá que VoltStack pueda utilizar de forma segura:

```text
local caches
distributed caches
persistent workers
replicas
shards
multitenancy
result caching
entity caching
metadata caching
query caching
```

sin convertir la invalidación en una colección de llamadas arbitrarias a `cache()->forget()`.

---

# 284. Siguiente documento

```text
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 17 — Cache** definiendo el modelo formal de consistencia entre:

```text
Database State
Replica State
Transaction State
Cache State
Application Observation
```

incluyendo:

```text
Cache Consistency Levels
Freshness
Staleness
Read-Your-Writes
Monotonic Reads
Version Validation
Generation Validation
Transaction Visibility
Commit Ordering
Unknown Outcomes
Replica Lag
Sticky Reads
Failover
Entity Cache
Result Cache
Query Cache
Metadata Cache
Distributed Cache
Worker-Local Cache
Cross-Node Consistency
Eventual Consistency
Strong Cache Requirements
Consistency Evidence
Consistency Degradation
Stale-While-Revalidate
TTL Semantics
Recovery
Outbox
Sharding
Multitenancy
Persistent Runtime
Telemetry
Testing
```

bajo la regla central:

```text
Cache Consistency
≠
Cache Invalidation
```

porque:

```text
Invalidation
=
mechanism
```

mientras:

```text
Consistency
=
observable guarantee
```