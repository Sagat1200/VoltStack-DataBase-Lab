# 186_DATABASE_CACHE_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Cache Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 186 — Database Cache Architecture  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `185_DATABASE_PARTITION_ROUTING_SYSTEM.md`  
**Siguiente documento:** `187_DATABASE_QUERY_CACHE_SYSTEM.md`

---

# 1. Propósito

`Database Cache Architecture` define la arquitectura general mediante la cual `VoltStack/Quantum/Database` podrá reutilizar información previamente calculada, compilada, obtenida o materializada sin confundir dicha información con el estado autoritativo de la base de datos.

La regla maestra será:

> **Un caché de Database en VoltStack será una optimización explícita sobre información cuya autoridad reside en otro componente; ningún cache hit convertirá automáticamente un valor almacenado en verdad actual de la base de datos.**

Formalmente:

```text
Cache
=
ReusableDerivedState
+
ValidityPolicy
+
Identity
+
Scope
+
ConsistencySemantics
```

y nunca:

```text
Cache
=
DatabaseTruth
```

La arquitectura deberá permitir optimizar:

- construcción de queries;
- análisis semántico;
- compilación;
- metadata;
- resultados;
- entidades;
- introspección;
- hydration plans;
- routing plans;
- otros artefactos derivados;

manteniendo correctamente:

- aislamiento;
- consistencia;
- transacciones;
- multitenancy;
- sharding;
- replicación;
- runtime persistente;
- seguridad;
- invalidación;
- observabilidad.

---

# 2. Problema arquitectónico

El término "cache" dentro de un sistema de base de datos puede referirse a mecanismos completamente diferentes.

Por ejemplo:

```text
Compiled Query Cache
Result Cache
Metadata Cache
Entity Cache
Hydration Plan Cache
IdentityMap
Schema Metadata Cache
Routing Plan Cache
Driver Prepared Statement Cache
```

Todos pueden reducir trabajo repetido, pero no tienen:

- la misma identidad;
- la misma duración;
- la misma autoridad;
- la misma invalidación;
- el mismo riesgo de stale data;
- la misma relación con transacciones.

Por tanto:

```text
Database Cache
≠
One Generic Key/Value Cache
```

VoltStack definirá una arquitectura común, pero conservará las semánticas particulares de cada tipo de caché.

---

# 3. Distinciones fundamentales

Las siguientes separaciones serán obligatorias:

```text
Database Cache
≠
Database

Cache Entry
≠
Database Row

Cache Hit
≠
Database Read

Query Cache
≠
Result Cache

Result Cache
≠
Entity Cache

Entity Cache
≠
IdentityMap

Metadata Cache
≠
Entity Cache

Hydration Cache
≠
Hydrated Entity Cache

Compiled Query Cache
≠
Query Result Cache

Application Cache
≠
Database Cache

Cache Invalidation
≠
Transaction Rollback

Cache Consistency
≠
Database Isolation

Cache Freshness
≠
Replica Freshness

TTL Expiration
≠
Semantic Invalidation

Cache Namespace
≠
Tenant

Cache Key
≠
Primary Key
```

---

# 4. Principio de autoridad

Cada caché deberá declarar cuál es su fuente autoritativa.

Ejemplos:

| Caché | Fuente autoritativa |
|---|---|
| Query plan cache | Query Engine |
| Compiled query cache | SQL Compiler |
| Metadata cache | Metadata compiler/registry |
| Schema metadata cache | Schema Introspection |
| Result cache | Database execution |
| Entity cache | Persisted database state |
| Hydration plan cache | Hydration compiler |
| Routing plan cache | Routing architecture |

Regla:

```text
CacheEntry
→ references authority

CacheEntry
≠ authority
```

---

# 5. Arquitectura general

```text
                    Application / ORM
                           │
                           ▼
                      Query Engine
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        Query Cache   Metadata Cache   Plan Cache
             │             │             │
             └─────────────┼─────────────┘
                           ▼
                       Compiler
                           │
                           ▼
                  Compiled Query Cache
                           │
                           ▼
                       Executor
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
             Result Cache        Database
                  │
                  ▼
               Hydration
                  │
                  ▼
               ORM Layer
                  │
          ┌───────┴────────┐
          ▼                ▼
     IdentityMap       Entity Cache
```

---

# 6. Dos familias principales

VoltStack distinguirá inicialmente dos grandes familias.

## 6.1 Structural Cache

Almacena artefactos derivados de configuración, metadata o estructura.

Ejemplos:

```text
CompiledMetadata
HydrationPlan
CompiledQuery
SemanticPlan
SchemaMetadata
RoutingPlan
```

Generalmente:

```text
StructuralCache
→ low volatility
→ generation/version invalidation
```

## 6.2 Data Cache

Almacena información derivada del estado de datos.

Ejemplos:

```text
QueryResult
EntityState
AggregateResult
```

Generalmente:

```text
DataCache
→ database mutation sensitive
→ consistency sensitive
```

Esta diferencia será fundamental.

---

# 7. Taxonomía

```text
Database Cache
│
├── Structural
│   ├── Query
│   ├── Compilation
│   ├── Metadata
│   ├── Hydration Plan
│   ├── Schema Metadata
│   └── Routing Plan
│
└── Data
    ├── Result
    └── Entity
```

No todos estos mecanismos pertenecerán necesariamente al mismo paquete físico.

---

# 8. CacheKind

Modelo conceptual:

```php
enum DatabaseCacheKind
{
    case QUERY;
    case COMPILED_QUERY;
    case RESULT;
    case METADATA;
    case ENTITY;
    case HYDRATION_PLAN;
    case SCHEMA_METADATA;
    case ROUTING_PLAN;
}
```

El enum no obliga a que todos tengan la misma implementación.

---

# 9. Query Cache

El `Query Cache` será desarrollado específicamente en:

```text
187_DATABASE_QUERY_CACHE_SYSTEM.md
```

Su responsabilidad será reutilizar artefactos asociados con queries que puedan ser identificados de forma semánticamente estable.

No deberá confundirse con:

```text
SELECT ... result cache
```

---

# 10. Compiled Query Cache

Ya definido conceptualmente en:

```text
075_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

Su función principal:

```text
Normalized Query / Plan
        ↓
Compiler
        ↓
CompiledQuery
        ↓
Cache
```

evitando recompilar SQL cuando las condiciones estructurales sean equivalentes.

---

# 11. Result Cache

Será desarrollado en:

```text
188_DATABASE_RESULT_CACHE_SYSTEM.md
```

Su función:

```text
Logical Query
+
Parameters
+
Execution Context
+
Consistency Identity
        ↓
Cached Result
```

Este es uno de los caches con mayor riesgo de stale data.

---

# 12. Metadata Cache

Será desarrollado en:

```text
189_DATABASE_METADATA_CACHE_SYSTEM.md
```

Almacenará artefactos como:

```text
EntityMetadata
RelationshipMetadata
MappingMetadata
CompiledAccessorMetadata
TypeMetadata
```

principalmente derivados de configuración/código.

---

# 13. Entity Cache

Será desarrollado en:

```text
190_DATABASE_ENTITY_CACHE_SYSTEM.md
```

Permitirá implementar un cache ORM de segundo nivel.

Conceptualmente:

```text
Database
   ↓
Canonical Persistent State
   ↓
Second-Level Entity Cache
   ↓
Hydration / Materialization
   ↓
IdentityMap
```

Nunca:

```text
EntityCache
=
IdentityMap
```

---

# 14. IdentityMap no es cache de segundo nivel

`IdentityMap` tiene una semántica distinta:

```text
PersistenceContext
+
EntityKey
→ canonical managed instance
```

Sus objetivos son:

- identidad de objetos;
- coherencia del grafo;
- evitar entidades duplicadas;
- integración con UnitOfWork.

No:

- compartir entidades entre requests;
- conservar entidades globalmente;
- actuar como distributed cache.

---

# 15. Scope del IdentityMap

```text
IdentityMap
→ request / operation / persistence context
```

Mientras:

```text
Entity Cache
→ may outlive request
```

---

# 16. Hydration Plan Cache

Documento `141_DATABASE_HYDRATION_CACHE_SYSTEM.md` estableció la idea fundamental:

```text
Hydration Cache
→ plans/accessors/compiled hydration artifacts

Hydration Cache
≠
hydrated entity instances
```

Esta separación se conserva.

---

# 17. Application Cache

VoltStack podrá tener un sistema general:

```text
VoltStack/Quantum/Cache
```

Database podrá integrarse con él.

Pero:

```text
Database Cache Architecture
≠
Generic Cache Architecture
```

Database necesita semánticas adicionales:

- transaction awareness;
- query identity;
- entity identity;
- database context;
- tenant context;
- shard context;
- replica consistency;
- mutation invalidation.

---

# 18. Dependency rule

Preferencia:

```text
Database
    ↓
Cache Contracts
```

y mediante integración:

```text
VoltStack Cache
    ↓
Database Cache Adapter
```

El core Database no deberá depender obligatoriamente de Redis, Memcached u otro proveedor.

---

# 19. Cache backend abstraction

Contrato conceptual:

```php
interface DatabaseCacheStore
{
    public function get(CacheKey $key): CacheLookup;

    public function put(
        CacheKey $key,
        CacheEntry $entry
    ): void;

    public function remove(CacheKey $key): void;
}
```

Sin asumir:

```text
Redis
Memcached
Filesystem
APCu
Database
```

---

# 20. Backend capabilities

No todos los stores ofrecen las mismas capacidades.

Ejemplos:

```text
TTL
Atomic CAS
Tagging
Bulk Get
Bulk Delete
Distributed Locks
Transactions
Pub/Sub
Versioned Keys
```

Por ello:

```text
CacheStore
→ CapabilitySet
```

---

# 21. Capability-driven architecture

Nunca:

```php
if ($cache instanceof RedisCache) {
    // ...
}
```

como regla arquitectónica.

Preferir:

```php
if ($capabilities->supports(CacheCapability::ATOMIC_COMPARE_AND_SWAP)) {
    // ...
}
```

---

# 22. CacheEntry

Modelo conceptual:

```php
final readonly class DatabaseCacheEntry
{
    public function __construct(
        public DatabaseCacheKey $key,
        public mixed $payload,
        public CacheMetadata $metadata,
        public CacheValidity $validity,
    ) {}
}
```

---

# 23. CacheMetadata

Puede contener:

```text
CreatedAt
ExpiresAt
Generation
Version
Tags
SourceFingerprint
ConsistencyMetadata
DependencyMetadata
```

No todos los caches utilizarán todos los campos.

---

# 24. Cache key architecture

Una cache key deberá representar toda dimensión capaz de modificar el resultado semántico.

Formalmente:

```text
CacheKey
=
Fingerprint(
    SemanticIdentity
    +
    ExecutionDomain
    +
    RelevantConfiguration
    +
    Generation
)
```

---

# 25. Key completeness

Si dos operaciones pueden producir resultados semánticamente diferentes, no deberán compartir accidentalmente una key.

---

# 26. Key stability

Si dos operaciones son semánticamente equivalentes, idealmente podrán compartir una key cuando el tipo de caché lo permita.

---

# 27. No raw SQL as universal identity

No se utilizará:

```text
hash(SQL string)
```

como identidad universal de todos los caches.

Dos SQL strings pueden ser:

- sintácticamente distintos pero semánticamente equivalentes;
- iguales pero ejecutados en contextos diferentes.

---

# 28. Execution domain

Para caches de datos, una key podrá necesitar:

```text
LogicalDatabase
Tenant
Shard
Partition
ReadConsistency
TransactionVisibility
```

según el caso.

---

# 29. Tenant isolation

Cuando Multitenancy esté instalado:

```text
CacheKey(Tenant A)
≠
CacheKey(Tenant B)
```

para información tenant-dependent.

Nunca se confiará únicamente en:

```text
same SQL
+
same parameters
```

---

# 30. Shard isolation

Igualmente:

```text
Shard A
≠
Shard B
```

cuando el resultado dependa del shard.

---

# 31. Database identity

Dos conexiones que apuntan a bases lógicas distintas no deberán compartir accidentalmente entradas.

---

# 32. Cache namespace

Conceptualmente:

```text
DatabaseCacheNamespace
├── Database
├── Tenant?
├── Shard?
├── CacheKind
└── Generation
```

---

# 33. Cache key privacy

No será obligatorio insertar valores sensibles directamente en la key.

Podrán utilizarse:

```text
CanonicalRepresentation
→ keyed hash / secure fingerprint
```

cuando sea necesario.

---

# 34. Cache key cardinality

Keys altamente variables son normales.

Metric labels altamente variables no.

Por tanto:

```text
CacheKey
≠
TelemetryLabel
```

---

# 35. Cache validity

VoltStack distinguirá al menos:

```php
enum CacheValidityState
{
    case VALID;
    case STALE;
    case EXPIRED;
    case INVALID;
    case UNKNOWN;
}
```

---

# 36. VALID

Existe evidencia suficiente para utilizar la entrada bajo la policy actual.

---

# 37. STALE

La entrada puede representar un estado anterior conocido.

Puede ser aceptable para ciertos reads.

---

# 38. EXPIRED

Su TTL o lease temporal terminó.

---

# 39. INVALID

Existe evidencia de que no debe utilizarse.

---

# 40. UNKNOWN

No existe evidencia suficiente para afirmar validez.

Regla:

```text
UNKNOWN
≠
VALID
```

---

# 41. TTL

TTL será solo una estrategia temporal:

```text
Entry
→ valid for N seconds
```

No demuestra que los datos no hayan cambiado durante ese tiempo.

---

# 42. TTL ≠ consistency

```text
TTL = 5 seconds
```

significa como máximo una política temporal.

No significa:

```text
data is transactionally consistent
```

---

# 43. Version invalidation

Structural caches deberán preferir frecuentemente:

```text
Generation
Version
Fingerprint
```

sobre TTL.

Ejemplo:

```text
EntityMetadataGeneration = 14
```

---

# 44. Dependency invalidation

Una entrada puede depender de:

```text
EntityType
Table
Relationship
ShardMap
SchemaVersion
MetadataGeneration
```

---

# 45. Dependency graph

Conceptualmente:

```text
Cache Entry
    │
    ├── Entity: Order
    ├── Table: orders
    ├── Shard: A
    └── Metadata Generation: 12
```

---

# 46. Tags

Un backend que soporte tags podrá representar esas dependencias directamente.

Pero:

```text
Tagging
```

no será obligatorio en todos los stores.

---

# 47. Tag emulation

Si se emula tagging, la implementación deberá declarar:

- costo;
- atomicidad;
- consistencia;
- race conditions.

---

# 48. Invalidation architecture

Será profundizada en:

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
```

Arquitectura general:

```text
Database Mutation
       ↓
Mutation Knowledge
       ↓
Invalidation Intent
       ↓
Transaction Outcome
       ↓
Invalidation Dispatch
       ↓
Affected Cache Entries
```

---

# 49. Invalidation timing

Regla fundamental:

> Una mutación no deberá invalidar globalmente información como si estuviera comprometida antes de conocerse el resultado transaccional cuando esa diferencia pueda romper la semántica del sistema.

---

# 50. Before commit

Supongamos:

```text
BEGIN
UPDATE order
invalidate cache
ROLLBACK
```

Una invalidación prematura puede ser segura en algunos modelos pero innecesariamente destructiva.

Más peligroso es publicar nuevos valores como comprometidos antes de commit.

---

# 51. Deferred invalidation

Modelo preferido:

```text
Mutation
    ↓
Transaction-local Invalidation Intent
    ↓
COMMIT confirmed
    ↓
Publish invalidation
```

---

# 52. Rollback

Si:

```text
ROLLBACK confirmed
```

los invalidation intents de commit pueden descartarse.

---

# 53. UNKNOWN transaction outcome

Caso:

```text
COMMIT sent
Connection lost
Outcome UNKNOWN
```

Cache system no deberá asumir:

```text
commit
```

ni:

```text
rollback
```

---

# 54. UNKNOWN policy

Una estrategia segura puede requerir:

```text
conservative invalidation
```

para las entradas posiblemente afectadas.

---

# 55. Invalidation ≠ rollback

Invalidar un caché no restaura:

- database rows;
- EntityManager;
- IdentityMap;
- PHP object graph.

---

# 56. Cache consistency architecture

Será profundizada en:

```text
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

La arquitectura deberá expresar:

```text
Strong-ish / Coordinated
Bounded Staleness
Eventual
Best Effort
Unknown
```

sin prometer garantías que el backend o la topología no pueda cumplir.

---

# 57. CacheConsistencyPolicy

Modelo conceptual:

```php
enum DatabaseCacheConsistencyPolicy
{
    case STRICT;
    case TRANSACTION_AWARE;
    case BOUNDED_STALENESS;
    case EVENTUAL;
    case BEST_EFFORT;
}
```

Los nombres definitivos podrán especializarse por tipo de caché.

---

# 58. STRICT no significa serializable

`STRICT` dentro de cache semantics no deberá confundirse con:

```text
SERIALIZABLE transaction isolation
```

---

# 59. Database isolation vs cache consistency

```text
Database Isolation
→ concurrent database visibility

Cache Consistency
→ relationship between cached state and authoritative state
```

---

# 60. Replica freshness vs cache freshness

Son dimensiones independientes.

Ejemplo:

```text
Primary
   ↓
Replica lag = 2s
   ↓
Result cached for 30s
```

La stale window efectiva puede ser mayor.

---

# 61. Freshness composition

Conceptualmente:

```text
ObservedFreshness
=
DatabaseReadFreshness
+
CacheAge
+
InvalidationDelay
```

No necesariamente suma aritmética exacta, pero representa capas de stale risk.

---

# 62. Transaction-aware reads

Una transacción activa introduce una frontera especial.

Por default:

```text
ActiveTransaction
+
Shared Result Cache
→ unsafe unless explicitly proven
```

---

# 63. Transaction-local modifications

Ejemplo:

```text
BEGIN

UPDATE users
SET name = 'Ana'

SELECT user
```

Un shared cache con el valor anterior:

```text
name = 'Maria'
```

no deberá ocultar el write visible dentro de la transacción.

---

# 64. Safe default

Shared data caches deberán normalmente ser bypassed dentro de transacciones activas salvo estrategia explícitamente transaction-aware.

---

# 65. Transaction-local cache

Podrá existir un cache efímero:

```text
TransactionLocalCache
```

pero:

```text
TransactionLocalCache
≠
SharedDatabaseCache
```

---

# 66. Transaction-local state

Deberá vivir en:

```text
TransactionContext
→ TransactionResourceRegistry
```

o mecanismo scoped equivalente.

Nunca globalmente.

---

# 67. Commit publication

Un transaction-local cached value no deberá promoverse automáticamente a shared cache sin:

- confirmed commit;
- known dependencies;
- valid consistency policy.

---

# 68. Rollback

Transaction-local cache se descarta.

---

# 69. UNKNOWN outcome

No deberá promoverse.

---

# 70. Result cache and transactions

La regla conservadora:

```text
Active transaction
→ bypass shared Result Cache
```

será un default razonable.

---

# 71. Entity cache and UnitOfWork

El Entity Cache no deberá sustituir al UnitOfWork.

Pipeline:

```text
Entity Cache
    ↓
Canonical State
    ↓
Hydration
    ↓
IdentityMap
    ↓
UnitOfWork
```

---

# 72. Existing managed entity

Si IdentityMap ya contiene:

```text
User#42
```

un Entity Cache hit no deberá crear otra instancia.

---

# 73. Dirty managed entity

Un cache hit no deberá sobrescribir silenciosamente una entidad managed y dirty.

---

# 74. Entity cache payload

Preferencia:

```text
Canonical Persistent State
```

sobre:

```text
serialized live PHP object
```

---

# 75. No PHP object serialization by default

No almacenar:

```php
serialize($entity);
```

como diseño central.

Riesgos:

- código arbitrario;
- class evolution;
- proxy state;
- UoW state;
- lazy loaders;
- runtime context;
- security;
- portability.

---

# 76. Canonical cache payload

Ejemplo:

```text
EntityCacheEntry
├── EntityType
├── CanonicalIdentifier
├── PersistentFieldValues
├── LoadedFieldMask
├── Version
├── MetadataGeneration
└── ConsistencyMetadata
```

---

# 77. Partial entity caching

Por default deberá evitarse.

Si se soporta:

```text
LoadedFieldMask
```

será obligatorio.

---

# 78. Query result payload

Result Cache deberá decidir explícitamente entre:

```text
raw driver rows
canonical result values
hydrated result shape
entity identifiers
```

La arquitectura preferirá representaciones que no capturen estado runtime accidental.

---

# 79. Cache serialization

Los backends distribuidos necesitarán encoding.

Deberá existir:

```text
CacheCodec
```

separado del cache semantics.

---

# 80. CacheCodec

Contrato conceptual:

```php
interface DatabaseCacheCodec
{
    public function encode(DatabaseCachePayload $payload): string;

    public function decode(string $payload): DatabaseCachePayload;
}
```

---

# 81. Codec ≠ PHP serialize

La implementación podrá utilizar formatos seguros y versionados.

---

# 82. Payload version

Toda representación persistente deberá poder declarar:

```text
PayloadVersion
```

---

# 83. Codec evolution

Un cambio incompatible deberá producir:

```text
cache miss / invalid entry
```

en vez de interpretar bytes incorrectamente.

---

# 84. Cache corruption

Datos corruptos en caché no deberán convertirse automáticamente en database corruption.

Preferencia:

```text
detect
→ evict
→ miss
→ source of truth
```

cuando sea seguro.

---

# 85. Cache failure semantics

Un cache backend puede:

- estar caído;
- responder lento;
- devolver corrupción;
- perder entries;
- rechazar writes.

Database deberá definir policy.

---

# 86. CacheAvailabilityPolicy

```php
enum CacheAvailabilityPolicy
{
    case FAIL_OPEN;
    case FAIL_CLOSED;
}
```

---

# 87. FAIL_OPEN

Para caches puramente optimizadores:

```text
Cache unavailable
→ bypass
→ Database
```

será normalmente preferible.

---

# 88. FAIL_CLOSED

Solo cuando la aplicación haya declarado que el cache forma parte de una garantía necesaria.

No será default para query/result/entity caching ordinario.

---

# 89. Cache stampede

Múltiples requests pueden detectar simultáneamente:

```text
MISS
```

y recalcular el mismo valor.

---

# 90. Stampede protection

Podrán utilizarse:

- request coalescing;
- single-flight;
- leases;
- probabilistic early refresh;
- bounded locks;
- stale-while-revalidate.

---

# 91. No mandatory distributed lock

La arquitectura no deberá asumir que todo backend soporta distributed locking.

---

# 92. Single-flight

Dentro del mismo proceso:

```text
Key K
→ one computation
→ multiple waiters
```

puede ser suficiente para ciertos structural caches.

---

# 93. Distributed stampede

Result/entity caches compartidos podrán requerir mecanismos adicionales.

---

# 94. Lock timeout

Nunca esperar indefinidamente por un cache fill lock.

---

# 95. Stale-while-revalidate

Podrá existir:

```text
STALE entry
→ temporarily serve
→ background refresh
```

solo cuando la consistency policy lo permita.

---

# 96. Stale-if-error

Igualmente:

```text
Database unavailable
+
stale cache exists
```

no significa automáticamente que deba servirse.

Será policy explícita.

---

# 97. Negative caching

VoltStack podrá cachear ciertos "not found".

Ejemplo:

```text
Entity User#999
→ NOT_FOUND
```

---

# 98. Negative cache risk

Posteriormente puede crearse:

```text
User#999
```

Por tanto, negative cache requiere invalidación/version/TTL apropiados.

---

# 99. Negative entry

Será distinta de:

```text
null payload
```

Ejemplo:

```php
enum CacheLookupState
{
    case HIT;
    case NEGATIVE_HIT;
    case MISS;
    case STALE;
    case ERROR;
}
```

---

# 100. Null semantics

```text
Cached NULL
≠
Cache MISS
```

---

# 101. MISSING semantics

También:

```text
MISSING
≠
NULL
≠
MISS
```

cuando el result shape lo requiera.

---

# 102. Cache lookup

```php
final readonly class CacheLookup
{
    public function __construct(
        public CacheLookupState $state,
        public ?DatabaseCacheEntry $entry,
    ) {}
}
```

---

# 103. Multi-level caching

VoltStack podrá soportar:

```text
L1 Process Cache
       ↓
L2 Distributed Cache
       ↓
Database
```

---

# 104. L1

Ejemplos:

- in-memory metadata;
- compiled query;
- hydration plans.

---

# 105. L2

Ejemplos:

- Redis-backed metadata;
- result cache;
- entity cache.

---

# 106. Multi-level consistency

L1 y L2 deberán compartir:

- generations;
- invalidation protocol;
- namespace semantics;

cuando almacenen el mismo tipo de información.

---

# 107. L1 invalidation problem

Invalidar Redis no elimina automáticamente una copia:

```text
worker-local L1
```

Por tanto, distributed invalidation podrá necesitar:

```text
generation tokens
```

---

# 108. Generation-based invalidation

Ejemplo:

```text
EntityCacheGeneration(User) = 17
```

Key:

```text
entity:user:17:42
```

Al avanzar a:

```text
18
```

la entrada anterior deja de ser addressable.

---

# 109. Logical invalidation

No siempre será necesario borrar físicamente entradas.

Puede bastar:

```text
advance generation
```

---

# 110. Garbage collection

Entradas antiguas podrán expirar posteriormente mediante TTL/eviction.

---

# 111. Generation granularity

Puede ser:

```text
Global
Database
Tenant
Shard
EntityType
EntityId
QueryDependencySet
```

Trade-off:

```text
fine-grained
→ precise
→ more metadata

coarse-grained
→ simpler
→ more invalidation
```

---

# 112. Cache dependency identity

Modelo conceptual:

```php
interface CacheDependency
{
    public function fingerprint(): CacheDependencyFingerprint;
}
```

---

# 113. Dependency types

```text
DatabaseDependency
TableDependency
EntityTypeDependency
EntityDependency
ShardDependency
TenantDependency
MetadataGenerationDependency
SchemaGenerationDependency
ShardMapGenerationDependency
```

---

# 114. Query dependency extraction

Result Cache podrá necesitar saber:

```text
Which data can invalidate this result?
```

Query semantic analysis podrá aportar dicha información.

---

# 115. SQL text insufficient

```sql
SELECT *
FROM orders
WHERE customer_id = 42
```

puede depender de:

```text
orders
customer 42
shard A
tenant X
```

No solo de un hash del SQL.

---

# 116. Mutation dependency extraction

Persistence Engine conoce semantic changes:

```text
EntityType
EntityKey
ChangedFields
Relationships
Shard
```

Esta información será preferible a intentar reconstruir invalidación leyendo SQL generado.

---

# 117. Invalidation source

Preferencia:

```text
Semantic Mutation
→ Cache Invalidation
```

sobre:

```text
SQL String
→ Parse SQL
→ Guess invalidation
```

---

# 118. Raw SQL problem

Raw SQL puede escapar del semantic persistence pipeline.

Por tanto, podrá requerir:

```text
explicit cache invalidation hints
```

o invalidación conservadora.

---

# 119. External database writers

Otro servicio puede modificar la misma DB sin pasar por VoltStack.

Entonces:

```text
Application-generated invalidation
```

no será suficiente.

---

# 120. External mutation strategies

Posibles integraciones:

- CDC;
- database notifications;
- event stream;
- version table;
- TTL;
- external invalidation bus.

No serán obligatorias en core.

---

# 121. Cache coherence scope

Toda consistency guarantee deberá declarar su scope.

Ejemplo:

```text
single process
single application cluster
single database writer
multi-service environment
```

---

# 122. No impossible guarantees

VoltStack no prometerá strong cache consistency si existen external writers invisibles sin mecanismo de coordinación.

---

# 123. Sharding integration

Cache identity deberá respetar:

```text
ShardId
ShardMapGeneration
```

cuando sea relevante.

---

# 124. Shard movement

Durante rebalancing:

```text
Key 42
Shard A → Shard B
```

una cache entry no deberá quedar accidentalmente asociada para siempre al physical shard anterior.

---

# 125. Logical identity preference

Cuando sea posible:

```text
Logical Entity Identity
+
Ownership Generation
```

será preferible a depender exclusivamente de physical endpoint.

---

# 126. Replica integration

Un Result Cache generado desde una replica puede reflejar estado atrasado.

Debe poder registrar:

```text
ReadSourceRole
FreshnessEvidence
ConsistencyToken?
```

cuando la policy lo necesite.

---

# 127. Sticky connection integration

Sticky read routing puede evitar replica stale reads, pero no invalida automáticamente un cache stale.

---

# 128. Failover integration

Cambiar writer:

```text
Primary A → Primary B
```

no deberá invalidar structural caches sin motivo.

Data caches dependerán de consistency semantics, no de endpoint identity por sí sola.

---

# 129. Load balancing

Load balancing no deberá cambiar cache identity cuando endpoints representan la misma semántica válida.

---

# 130. Distributed topology

Cache key deberá representar:

```text
logical execution domain
```

no necesariamente:

```text
physical server hostname
```

---

# 131. Schema integration

Schema changes pueden invalidar:

- metadata cache;
- compiled queries;
- hydration plans;
- result payload codecs;
- entity cache entries.

---

# 132. SchemaGeneration

Podrá formar parte de fingerprints.

---

# 133. Migration integration

Migration completion podrá emitir:

```text
SchemaGenerationChanged
```

---

# 134. Zero-downtime migration

Durante expand/contract podrán coexistir varias generaciones compatibles.

Por ello:

```text
schema version
```

no deberá ser una simple variable global mutable sin estrategia.

---

# 135. Type System integration

Cambios de:

```text
TypeRegistryGeneration
```

pueden invalidar:

- converters;
- hydration plans;
- metadata;
- entity cache codecs;
- query compilation artifacts.

---

# 136. Custom Type integration

`CustomType` deberá declarar si su representación cacheable es estable.

---

# 137. Value Object integration

Entity cache deberá almacenar una representación canonicalizable del value object, no necesariamente su objeto PHP runtime.

---

# 138. JSON integration

JSON cache payload deberá preservar:

```text
NULL
JSON null
missing
numbers
object/list distinction
```

según el Type System.

---

# 139. Date/time integration

No deberá introducir conversiones implícitas de timezone al cachear.

---

# 140. Enum integration

Persisted enum identity deberá utilizar su representación estable/canónica.

No ordinal runtime accidental.

---

# 141. Optimistic locking

Entity cache deberá conservar:

```text
Version
```

cuando una entidad utilice optimistic locking.

---

# 142. Versioned entity cache

Ejemplo:

```text
User#42
version=17
```

permite detectar que:

```text
cached version 16
```

está obsoleta cuando existe evidencia de versión 17.

---

# 143. Pessimistic locking

Un cache hit no sustituye un database lock.

---

# 144. Locking rule

Si una operación requiere:

```text
SELECT ... FOR UPDATE
```

un shared result/entity cache deberá ser bypassed.

---

# 145. Concurrency control

Cache no deberá romper las garantías definidas por:

```text
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
```

---

# 146. Serializable transaction

Un cached read fuera del mecanismo de la transacción puede romper el conjunto de observaciones esperado.

Por ello shared data cache deberá ser bypassed salvo soporte demostrado.

---

# 147. Query timeout

Cache lookup también deberá tener:

```text
timeout / deadline
```

compatible con la operación.

---

# 148. Cache slower than DB

El cache no deberá consumir todo el deadline dejando sin tiempo a la fuente autoritativa.

---

# 149. Cache budget

Conceptualmente:

```text
Operation Deadline
    ↓
Cache Lookup Budget
    ↓
Database Budget
```

---

# 150. Cancellation

Cache operations deberán poder observar cancellation cuando backend lo permita.

---

# 151. Resource governance

Se deberán limitar:

- entry size;
- payload size;
- number of dependencies;
- number of tags;
- key length;
- serialization depth;
- local memory;
- concurrent fills.

---

# 152. Oversized entry

No deberá cachearse automáticamente un resultado de cientos de MB.

---

# 153. CacheAdmissionPolicy

```php
interface CacheAdmissionPolicy
{
    public function admit(
        CacheCandidate $candidate
    ): CacheAdmissionDecision;
}
```

---

# 154. Admission factors

Puede considerar:

```text
PayloadSize
ComputeCost
ReadFrequency
ExpectedReuse
Volatility
ConsistencyRequirement
```

---

# 155. Not every result should be cached

```text
Cacheable
≠
ShouldCache
```

---

# 156. Eviction

Eviction es decisión de capacidad:

```text
remove entry to free resources
```

No es lo mismo que invalidation.

---

# 157. Eviction vs invalidation

```text
Eviction
→ resource policy

Invalidation
→ semantic validity
```

---

# 158. Expiration vs eviction

```text
Expiration
→ time policy

Eviction
→ capacity policy
```

---

# 159. Invalidation vs expiration

```text
Invalidation
→ evidence of semantic obsolescence

Expiration
→ temporal validity boundary
```

---

# 160. Cache warming

Structural caches podrán precargarse durante boot/deploy.

Ejemplos:

```text
Entity Metadata
Hydration Plans
Compiled Metadata
```

---

# 161. Data cache warming

Deberá ser explícito y normalmente application-specific.

---

# 162. Cold start

Cache miss deberá seguir siendo un estado normal.

---

# 163. Cache correctness invariant

La aplicación deberá funcionar correctamente aunque un cache puramente optimizador esté vacío.

Formalmente:

```text
Correctness(Database with empty optional cache)
=
Correctness(Database without optional cache)
```

---

# 164. Cache outage invariant

Para caches `FAIL_OPEN`:

```text
CacheUnavailable
→ PerformanceDegradation
```

no:

```text
SemanticCorruption
```

---

# 165. Security architecture

Database caches pueden contener datos sensibles.

Deberán considerarse:

- encryption at rest;
- transport security;
- namespace isolation;
- access control;
- payload minimization;
- secure deletion limitations;
- logging redaction.

---

# 166. Sensitive cache policy

Ciertos campos podrán declararse:

```text
NON_CACHEABLE
```

---

# 167. Field-level exclusion

Entity cache metadata podrá excluir:

- secrets;
- tokens;
- password hashes;
- sensitive blobs;

según policy.

---

# 168. Partial secure caching

Si se excluyen campos:

```text
LoadedFieldMask
```

deberá reflejarlo.

---

# 169. Cache poisoning

El sistema deberá protegerse contra:

- user-controlled keys;
- namespace collisions;
- malformed payloads;
- forged serialized values;
- untrusted type identifiers.

---

# 170. Type loading

Nunca:

```text
cache payload
→ arbitrary class name
→ autoload
```

sin metadata confiable.

---

# 171. Codec security

Prohibidos por default:

```text
eval()
unsafe unserialize()
dynamic executable payloads
```

---

# 172. Cache key DoS

Input arbitrario no deberá producir keys de tamaño ilimitado.

---

# 173. Cache amplification

Una sola request no deberá poder generar fan-out ilimitado de cache writes.

---

# 174. Observabilidad

Toda capa deberá exponer:

```text
Hit
Miss
Stale
NegativeHit
Error
Bypass
Write
Invalidation
Eviction
```

---

# 175. DatabaseCacheTelemetry

Contrato conceptual:

```php
interface DatabaseCacheTelemetry
{
    public function lookupStarted(CacheLookupContext $context): void;

    public function lookupCompleted(CacheLookupResult $result): void;

    public function writeCompleted(CacheWriteResult $result): void;

    public function invalidated(CacheInvalidationResult $result): void;
}
```

---

# 176. Metrics

Ejemplos:

```text
db.cache.lookup
db.cache.hit
db.cache.miss
db.cache.stale
db.cache.bypass
db.cache.error
db.cache.write
db.cache.invalidation
db.cache.eviction
db.cache.lookup.duration
db.cache.payload.bytes
```

---

# 177. Metrics by kind

Dimensión bounded:

```text
kind=query
kind=result
kind=metadata
kind=entity
kind=hydration_plan
```

---

# 178. No raw keys in metrics

Nunca:

```text
cache_key=user:123456789
```

como label.

---

# 179. Tracing

Span conceptual:

```text
database.cache.lookup
```

atributos:

```text
cache.kind
cache.result
cache.scope
cache.backend.type
```

sin valores sensibles.

---

# 180. Cache events

Eventos posibles:

```text
DatabaseCacheHit
DatabaseCacheMiss
DatabaseCacheStale
DatabaseCacheBypassed
DatabaseCacheWrite
DatabaseCacheInvalidated
DatabaseCacheError
```

---

# 181. Events observational

Los eventos de telemetry no deberán modificar cache correctness.

---

# 182. Diagnostics

API conceptual:

```php
DB::cache()->diagnostics();
```

---

# 183. Diagnostic report

Ejemplo:

```text
DATABASE CACHE

Result Cache:
    enabled

Entity Cache:
    enabled

Metadata Cache:
    enabled

Backend:
    redis

L1:
    process-memory

L2:
    distributed

Transaction Policy:
    bypass-shared-data-cache

Namespace Generation:
    18

Invalidation:
    transaction-aware

External Writer Awareness:
    none

Consistency:
    application-coordinated
```

---

# 184. Explain API

Conceptualmente:

```php
DB::cache()->explain($query);
```

podrá indicar:

```text
Cacheable:
    yes

Cache Kind:
    RESULT

Decision:
    BYPASS

Reason:
    ACTIVE_TRANSACTION
```

---

# 185. CacheDecision

```php
enum CacheDecision
{
    case USE;
    case BYPASS;
    case REFRESH;
    case REVALIDATE;
    case REJECT;
}
```

---

# 186. Bypass reasons

```text
ACTIVE_TRANSACTION
LOCKING_QUERY
NON_DETERMINISTIC_QUERY
RAW_QUERY_UNKNOWN_DEPENDENCIES
EXPLICIT_BYPASS
SECURITY_POLICY
OVERSIZED_RESULT
UNSUPPORTED_BACKEND_CAPABILITY
UNKNOWN_CONSISTENCY
```

---

# 187. Non-deterministic queries

Ejemplos conceptuales:

```text
random()
current time
session-dependent functions
```

pueden ser no cacheables o requerir semántica especial.

---

# 188. Current time

Una query cuyo resultado depende de `NOW()` no deberá reutilizarse indefinidamente bajo una key estructural.

---

# 189. Session state

Resultados dependientes de:

```text
session variables
temporary tables
connection-local state
```

requieren tratamiento especial.

---

# 190. Temporary tables

Shared result caching deberá normalmente estar deshabilitado.

---

# 191. User-defined DB functions

Su determinismo deberá declararse o considerarse desconocido.

---

# 192. Cacheability analysis

Podrá existir:

```php
interface QueryCacheabilityAnalyzer
{
    public function analyze(
        QuerySemanticModel $query,
        CacheContext $context,
    ): CacheabilityAnalysis;
}
```

---

# 193. Cacheability state

```php
enum Cacheability
{
    case CACHEABLE;
    case CACHEABLE_WITH_CONSTRAINTS;
    case NON_CACHEABLE;
    case UNKNOWN;
}
```

---

# 194. UNKNOWN cacheability

Safe default:

```text
UNKNOWN
→ BYPASS
```

---

# 195. Explicit developer API

Ejemplos conceptuales:

```php
$query->cacheFor(seconds: 30);
```

```php
$query->withoutCache();
```

```php
$query->cachePolicy(
    CachePolicy::boundedStaleness(...)
);
```

---

# 196. Explicit request ≠ guaranteed cache

Aunque el desarrollador solicite:

```text
cacheFor(30)
```

el sistema podrá rechazar caching si viola invariantes de seguridad/correctness.

---

# 197. ORM API

Conceptualmente:

```php
User::query()
    ->cacheFor(60)
    ->find(42);
```

No implicará Entity Cache necesariamente.

Puede ser Result Cache según API.

---

# 198. Entity cache policy

Entity metadata podrá declarar:

```php
#[CacheableEntity(...)]
class Product
{
}
```

pero la sintaxis definitiva se definirá en documento 190.

---

# 199. Metadata cache default

Metadata será un candidato natural a caching por default debido a:

```text
high reuse
+
low runtime volatility
```

---

# 200. Result cache default

No deberá habilitarse indiscriminadamente para todas las queries.

---

# 201. Entity cache default

Igualmente deberá ser opt-in/policy-driven inicialmente.

---

# 202. Structural cache default

Puede ser ampliamente automático si la invalidación está basada en fingerprints/generations.

---

# 203. Query execution path sin data cache

```text
Query
  ↓
Query Engine
  ↓
Compiler
  ↓
Executor
  ↓
Database
  ↓
Result
```

---

# 204. Query execution path con Result Cache

```text
Query
  ↓
Semantic Identity
  ↓
Cacheability Analysis
  ↓
Result Cache
  │
  ├── HIT ────────────────┐
  │                       │
  └── MISS                │
       ↓                  │
    Executor              │
       ↓                  │
    Database              │
       ↓                  │
    Cache Admission       │
       ↓                  │
    Cache Write           │
       │                  │
       └──────────────────┤
                          ▼
                        Result
```

---

# 205. ORM path con Entity Cache

```text
Entity Lookup
     ↓
IdentityMap
  ┌──┴─────────┐
 HIT          MISS
  │             ↓
  │       Entity Cache
  │       ┌─────┴─────┐
  │      HIT         MISS
  │       │            ↓
  │       │         Database
  │       │            ↓
  │       │       Cache Admission
  │       │            ↓
  │       │       Entity Cache
  │       │            │
  └───────┴────────────┘
              ↓
          Hydration
              ↓
          IdentityMap
              ↓
            Entity
```

---

# 206. IdentityMap first

Dentro de un PersistenceContext:

```text
IdentityMap lookup
```

deberá ocurrir antes de consultar shared Entity Cache para una identidad ya managed.

---

# 207. Why IdentityMap first

Porque garantiza:

```text
same EntityKey
→ same managed PHP instance
```

---

# 208. Entity cache is not canonical object identity

Puede almacenar estado, pero no la instancia managed canónica del scope.

---

# 209. Cache mutation pipeline

```text
Application Mutation
       ↓
UnitOfWork
       ↓
Persistence Plan
       ↓
Database Execution
       ↓
Transaction Outcome
       ↓
Cache Invalidation / Update
```

---

# 210. Write-through

Podrá existir:

```text
Database write
→ cache update
```

pero únicamente con transaction outcome suficientemente cierto.

---

# 211. Write-behind

No será el default del ORM Database Cache.

```text
cache write
→ later database write
```

cambia radicalmente persistence semantics.

---

# 212. Cache-aside

Será un patrón principal:

```text
lookup cache
→ miss
→ source
→ populate cache
```

---

# 213. Read-through

Podrá implementarse mediante adapter, conservando la misma semántica.

---

# 214. Cache update vs invalidation

Después de mutation puede elegirse:

```text
UPDATE_CACHE
```

o:

```text
INVALIDATE_CACHE
```

---

# 215. Default mutation strategy

La invalidación suele ser más segura cuando no puede reconstruirse con certeza el nuevo resultado.

---

# 216. Query result invalidation

Actualizar manualmente todos los posibles cached query results puede ser impracticable.

Por ello:

```text
dependency invalidation
```

será central.

---

# 217. Entity cache update

Para una entidad exacta puede ser más viable actualizar:

```text
EntityKey
→ new canonical state
```

tras confirmed commit.

---

# 218. Relationship cache effects

Modificar una relación puede invalidar:

- owner entity cache;
- target entity cache;
- collection result caches;
- aggregate result caches.

---

# 219. Relationship metadata

El Relationship Persistence System podrá producir dependency information.

---

# 220. N+1 interaction

Entity Cache puede reducir queries físicas y ocultar parcialmente un N+1.

Pero:

```text
N+1 semantic pattern
```

puede seguir existiendo.

---

# 221. N+1 telemetry

El detector deberá poder distinguir:

```text
relationship load attempted
+
cache hit
```

de:

```text
database query executed
```

---

# 222. Performance architecture

Cache deberá reducir trabajo, no simplemente moverlo.

---

# 223. Cache cost

Formalmente:

```text
Benefit
=
SourceCost
-
(
    LookupCost
    +
    SerializationCost
    +
    ConsistencyCost
    +
    InvalidationCost
)
```

---

# 224. Negative-value cache

Si:

```text
Benefit <= 0
```

el cache puede empeorar rendimiento.

---

# 225. Metadata local cache

Para metadata immutable:

```text
process memory
```

puede ser más eficiente que una llamada Redis por request.

---

# 226. Distributed metadata cache

Puede ayudar principalmente en:

- cold boot;
- large compiled metadata;
- multi-process deployments;

pero no sustituye un L1 local apropiado.

---

# 227. Result cache backend

Puede favorecer distributed store cuando múltiples workers deben compartir resultados.

---

# 228. Entity cache backend

Igualmente puede requerir shared/distributed store.

---

# 229. Persistent runtime optimization

FrankenPHP permite conservar structural caches en memoria entre requests.

Esto puede ser una ventaja importante.

---

# 230. Persistent runtime danger

No deberá conservarse:

- tenant-specific mutable state;
- transaction state;
- managed entities;
- UnitOfWork;
- request result cache accidental;
- temporary cache bypass flags.

---

# 231. Shared immutable cache

Preferencia:

```text
immutable compiled artifacts
```

para process-local persistent caches.

---

# 232. Cache generations in workers

Cada worker deberá detectar cambios relevantes de generation.

---

# 233. Hot reload

En development:

```text
code/mapping change
→ generation change
→ structural cache invalidation
```

---

# 234. Production deployment

Podrá utilizar:

```text
deployment generation
```

para separar artefactos incompatibles.

---

# 235. Blue/green deployment

Dos versiones de aplicación pueden coexistir.

Por ello:

```text
CacheNamespace
```

podrá incorporar compatibility generation.

---

# 236. Rolling deployment

No se deberá asumir que todos los workers cambian de versión simultáneamente.

---

# 237. Backward-compatible payloads

Los codecs podrán declarar compatibilidad entre generaciones.

---

# 238. Incompatible payload

Debe tratarse como miss, no reinterpretarse.

---

# 239. Cache warmup CLI

Conceptualmente:

```text
php volt database:cache:warm
```

---

# 240. Cache clear CLI

```text
php volt database:cache:clear
```

deberá permitir scopes específicos.

Ejemplo:

```text
--kind=metadata
--kind=result
--entity=Product
```

---

# 241. Clear all danger

Una operación:

```text
FLUSH ALL
```

sobre un backend compartido no deberá ser el mecanismo normal.

---

# 242. Namespaced clear

Preferir:

```text
advance namespace generation
```

---

# 243. Cache inspection CLI

Conceptualmente:

```text
php volt database:cache:status
```

---

# 244. Cache health

Health check podrá observar:

```text
backend availability
latency
codec compatibility
generation state
invalidation channel
```

---

# 245. Health ≠ correctness proof

Backend healthy no significa que las entradas estén semánticamente frescas.

---

# 246. Cache resilience

Cache failure deberá integrarse posteriormente con:

```text
235_DATABASE_RESILIENCE_ARCHITECTURE.md
```

---

# 247. Circuit breaker

Un cache backend lento podrá tener circuit breaker separado del database backend.

---

# 248. Cache retry

Retries deberán ser bounded.

No deberán aumentar tanto la latencia que la optimización se convierta en cuello de botella.

---

# 249. Cache retry ≠ query retry

Son dominios distintos.

---

# 250. Error hierarchy

```text
DatabaseCacheException
├── DatabaseCacheLookupException
├── DatabaseCacheWriteException
├── DatabaseCacheDeleteException
├── DatabaseCacheInvalidationException
├── DatabaseCacheSerializationException
├── DatabaseCacheDeserializationException
├── DatabaseCacheCorruptionException
├── DatabaseCacheKeyException
├── DatabaseCacheNamespaceException
├── DatabaseCacheConsistencyException
├── DatabaseCacheCapabilityException
├── DatabaseCacheUnavailableException
├── DatabaseCacheGenerationException
├── DatabaseCacheDependencyException
├── DatabaseCacheAdmissionException
├── DatabaseCacheSecurityException
└── DatabaseCacheInvariantViolationException
```

---

# 251. Error handling

Para un optional cache:

```text
CacheLookupException
→ telemetry
→ bypass
→ authoritative source
```

cuando policy sea `FAIL_OPEN`.

---

# 252. Cache write failure

Si DB query ya fue exitosa:

```text
Database success
+
cache population failure
```

no deberá convertirse automáticamente en database failure.

---

# 253. Invalidation failure

Es más delicado.

```text
Commit success
+
invalidation failure
```

puede dejar stale cache.

Deberá:

- registrarse;
- activar recovery policy;
- posiblemente avanzar generation;
- nunca fingir invalidation success.

---

# 254. After-commit failure

No puede deshacer el commit ya confirmado.

---

# 255. Reliable invalidation

Una arquitectura avanzada podrá integrar:

```text
Transactional Outbox
→ Invalidation Event
→ Cache Consumers
```

---

# 256. Outbox limitation

Incluso con outbox:

```text
commit
→ invalidation delivery delay
```

puede producir una ventana de stale data.

La consistency policy deberá reconocerla.

---

# 257. Cache consistency token

Una extensión podrá asociar:

```text
DatabaseVersion
LSN
CommitSequence
EntityVersion
```

a entradas.

---

# 258. Capability-specific tokens

No se asumirá un mecanismo universal entre MySQL, MariaDB, PostgreSQL y SQLite.

---

# 259. Platform abstraction

Database Cache deberá consumir capabilities sin introducir:

```php
if ($platform === 'postgresql') ...
```

por todas partes.

---

# 260. SQLite

Un deployment local SQLite puede utilizar:

```text
in-process cache
```

sin necesidad de distributed infrastructure.

---

# 261. Scale proportionality

VoltStack deberá funcionar desde:

```text
SQLite + no cache
```

hasta:

```text
Sharded DB
+
Replicas
+
Distributed Result Cache
+
Second-Level Entity Cache
+
Cluster Invalidation
```

sin cambiar la arquitectura central.

---

# 262. Zero-config principle

Una aplicación simple no deberá configurar Redis para poder usar Database.

---

# 263. Default architecture

Inicialmente:

```text
Structural caches
→ automatic in-process where safe

Data caches
→ opt-in
```

es una estrategia recomendada.

---

# 264. Cache configuration

Ejemplo conceptual:

```php
return [

    'database' => [

        'cache' => [

            'metadata' => [
                'enabled' => true,
                'store' => 'memory',
            ],

            'query' => [
                'enabled' => true,
                'store' => 'memory',
            ],

            'result' => [
                'enabled' => false,
            ],

            'entity' => [
                'enabled' => false,
            ],

        ],

    ],

];
```

La sintaxis final pertenecerá al Config Integration System.

---

# 265. Cache profiles

Podrán definirse:

```text
development
testing
production
high-consistency
read-heavy
```

sin introducir comportamiento mágico no observable.

---

# 266. Testing mode

Testing podrá deshabilitar data caches por default para evitar ocultar queries y side effects.

---

# 267. Cache testing

También deberá existir modo específico para verificar:

- hits;
- misses;
- invalidation;
- consistency;
- namespace isolation.

---

# 268. Deterministic tests

Tests no deberán depender de TTL wall-clock real cuando pueda utilizarse un fake clock.

---

# 269. Fake cache store

Testing deberá disponer de:

```text
InMemoryDatabaseCacheStore
```

con semántica controlable.

---

# 270. Fault injection

Tests deberán poder simular:

```text
timeout
miss
stale entry
corrupt entry
write failure
invalidation failure
backend outage
```

---

# 271. Concurrency tests

Deberán probar:

- simultaneous misses;
- invalidation races;
- read/write races;
- generation changes;
- multi-worker simulation.

---

# 272. Transaction tests

Casos mínimos:

```text
cache hit outside transaction
cache bypass inside transaction
commit + invalidation
rollback + no publication
unknown outcome
afterCommit invalidation failure
```

---

# 273. Sharding tests

Deberán verificar que:

```text
same EntityId
different shard
```

no colisione cuando la identidad lo permita.

---

# 274. Tenant tests

Igualmente:

```text
Tenant A User#1
≠
Tenant B User#1
```

en cache namespace.

---

# 275. Schema evolution tests

Payloads incompatibles deberán convertirse en miss seguro.

---

# 276. Persistent worker tests

Dos requests consecutivos en mismo worker no deberán compartir:

- transaction-local cache;
- tenant-specific temporary cache state;
- request bypass flags.

---

# 277. Cache benchmark suite

Deberá medir:

```text
lookup latency
serialization latency
hit benefit
miss overhead
invalidation cost
memory consumption
stampede behavior
```

---

# 278. Directory structure

Propuesta:

```text
src/Quantum/Database/Cache/
│
├── DatabaseCacheManager.php
├── DatabaseCacheKind.php
├── DatabaseCacheEntry.php
├── DatabaseCacheKey.php
├── DatabaseCacheNamespace.php
├── DatabaseCacheContext.php
├── DatabaseCachePolicy.php
├── CacheLookup.php
├── CacheLookupState.php
├── CacheValidityState.php
├── CacheDecision.php
│
├── Contract/
│   ├── DatabaseCacheStore.php
│   ├── DatabaseCacheCodec.php
│   ├── DatabaseCacheKeyFactory.php
│   ├── CacheAdmissionPolicy.php
│   ├── CacheDependency.php
│   └── QueryCacheabilityAnalyzer.php
│
├── Store/
│   ├── NullDatabaseCacheStore.php
│   ├── InMemoryDatabaseCacheStore.php
│   └── DatabaseCacheStoreAdapter.php
│
├── Key/
│   ├── CacheKeyFactory.php
│   ├── CacheKeyFingerprint.php
│   └── CacheNamespaceResolver.php
│
├── Dependency/
│   ├── CacheDependencySet.php
│   ├── DatabaseDependency.php
│   ├── EntityDependency.php
│   ├── EntityTypeDependency.php
│   ├── TableDependency.php
│   ├── TenantDependency.php
│   ├── ShardDependency.php
│   └── GenerationDependency.php
│
├── Generation/
│   ├── CacheGeneration.php
│   ├── CacheGenerationResolver.php
│   └── CacheGenerationManager.php
│
├── Policy/
│   ├── DatabaseCacheConsistencyPolicy.php
│   ├── CacheAvailabilityPolicy.php
│   ├── CacheAdmissionPolicy.php
│   ├── CacheExpirationPolicy.php
│   └── CacheSecurityPolicy.php
│
├── Codec/
│   ├── CachePayload.php
│   ├── CachePayloadVersion.php
│   ├── CacheEncoder.php
│   └── CacheDecoder.php
│
├── Transaction/
│   ├── TransactionCacheCoordinator.php
│   ├── TransactionLocalCache.php
│   └── DeferredCacheMutation.php
│
├── Invalidation/
│   ├── CacheInvalidationIntent.php
│   ├── CacheInvalidationCoordinator.php
│   └── CacheInvalidationResult.php
│
├── Stampede/
│   ├── CacheSingleFlight.php
│   ├── CacheLease.php
│   └── CacheRefreshCoordinator.php
│
├── Diagnostics/
│   ├── DatabaseCacheInspector.php
│   ├── DatabaseCacheExplainer.php
│   └── DatabaseCacheDiagnosticReport.php
│
├── Telemetry/
│   ├── DatabaseCacheTelemetry.php
│   ├── DatabaseCacheHit.php
│   ├── DatabaseCacheMiss.php
│   └── DatabaseCacheInvalidated.php
│
└── Exception/
    ├── DatabaseCacheException.php
    ├── DatabaseCacheLookupException.php
    ├── DatabaseCacheWriteException.php
    ├── DatabaseCacheInvalidationException.php
    ├── DatabaseCacheCorruptionException.php
    └── DatabaseCacheInvariantViolationException.php
```

---

# 279. Specialized subsystems

Sobre esta base:

```text
Database/Cache
```

podrán existir:

```text
Database/Cache/Query
Database/Cache/Result
Database/Cache/Metadata
Database/Cache/Entity
Database/Cache/Invalidation
Database/Cache/Consistency
```

---

# 280. DatabaseCacheManager

No deberá convertirse en God Object.

Responsabilidad:

```text
resolve cache subsystem
+
coordinate shared policies
```

No:

- interpretar queries;
- hidratar;
- persistir;
- ejecutar;
- administrar UnitOfWork;
- decidir transaction outcome.

---

# 281. Cache subsystem ownership

Cada cache especializado será dueño de:

```text
identity
cacheability
payload
validity
```

correspondiente a su dominio.

---

# 282. Shared infrastructure

Podrá compartir:

```text
Store abstraction
Codec
Generation
Telemetry
Diagnostics
Security
```

---

# 283. Cache lifecycle

Modelo:

```text
Candidate
   ↓
Cacheability Analysis
   ↓
Admission
   ↓
Encoding
   ↓
Stored
   ↓
Lookup
   ↓
Validation
   ├── VALID → HIT
   ├── STALE → policy
   ├── EXPIRED → MISS
   ├── INVALID → EVICT/MISS
   └── UNKNOWN → BYPASS/MISS
```

---

# 284. Cache read pipeline

```text
Request
   ↓
Build Semantic Key
   ↓
Resolve Namespace
   ↓
Check Policy
   ↓
Store Lookup
   ↓
Decode
   ↓
Validate Metadata
   ↓
Validate Generation
   ↓
Validate Consistency
   ↓
Return CacheLookup
```

---

# 285. Cache write pipeline

```text
Source Result
   ↓
Cacheability
   ↓
Admission
   ↓
Canonical Payload
   ↓
Dependency Extraction
   ↓
Generation Metadata
   ↓
Encode
   ↓
Store
```

---

# 286. Mutation pipeline

```text
Semantic Mutation
   ↓
Dependency Impact
   ↓
Invalidation Intent
   ↓
Transaction Context
   ↓
Outcome
   ├── COMMITTED
   │      ↓
   │   Publish
   │
   ├── ROLLED_BACK
   │      ↓
   │   Discard
   │
   └── UNKNOWN
          ↓
       Conservative Policy
```

---

# 287. Cache ownership rule

El cache subsystem nunca será propietario de la verdad persistente.

---

# 288. Cache bypass invariant

Toda capa cacheable deberá conservar un camino funcional:

```text
BYPASS
→ authoritative subsystem
```

---

# 289. Architectural invariants

## DB-CACHE-001
Cache nunca será considerado Database Truth.

## DB-CACHE-002
Cache Hit no equivaldrá a Database Read.

## DB-CACHE-003
Query Cache será distinto de Result Cache.

## DB-CACHE-004
Result Cache será distinto de Entity Cache.

## DB-CACHE-005
Entity Cache será distinto de IdentityMap.

## DB-CACHE-006
IdentityMap conservará identidad scoped de entidades.

## DB-CACHE-007
Entity Cache podrá sobrevivir al PersistenceContext.

## DB-CACHE-008
Metadata Cache será distinto de Data Cache.

## DB-CACHE-009
Structural Cache será distinto de Data Cache.

## DB-CACHE-010
Compiled Query Cache no almacenará query results.

## DB-CACHE-011
Hydration Plan Cache no almacenará managed entities.

## DB-CACHE-012
Database Cache será distinto de generic application cache.

## DB-CACHE-013
Database core no dependerá de Redis.

## DB-CACHE-014
Database core no dependerá de Memcached.

## DB-CACHE-015
Backend-specific behavior se expresará mediante capabilities.

## DB-CACHE-016
Cache key incluirá toda dimensión semánticamente relevante.

## DB-CACHE-017
SQL string no será identidad universal.

## DB-CACHE-018
Tenant-dependent entries estarán aisladas por tenant domain.

## DB-CACHE-019
Shard-dependent entries estarán aisladas por shard domain.

## DB-CACHE-020
Logical database identity formará parte del namespace cuando sea relevante.

## DB-CACHE-021
Cache keys sensibles podrán fingerprintarse.

## DB-CACHE-022
Raw cache keys no serán metric labels.

## DB-CACHE-023
VALID será distinto de STALE.

## DB-CACHE-024
STALE será distinto de EXPIRED.

## DB-CACHE-025
INVALID será distinto de UNKNOWN.

## DB-CACHE-026
UNKNOWN no será tratado como VALID.

## DB-CACHE-027
TTL no demostrará consistencia.

## DB-CACHE-028
Expiration será distinta de invalidation.

## DB-CACHE-029
Eviction será distinta de invalidation.

## DB-CACHE-030
Eviction será distinta de expiration.

## DB-CACHE-031
Structural caches podrán utilizar generation invalidation.

## DB-CACHE-032
Generation será parte de cache identity cuando sea requerida.

## DB-CACHE-033
Invalidation podrá ser dependency-driven.

## DB-CACHE-034
Mutation semantics serán preferibles a SQL parsing para invalidation.

## DB-CACHE-035
Raw SQL no producirá falsa dependency certainty.

## DB-CACHE-036
External writers reducirán las garantías si no existe coordinación.

## DB-CACHE-037
Cache consistency guarantee declarará su scope.

## DB-CACHE-038
Cache consistency será distinta de transaction isolation.

## DB-CACHE-039
Cache freshness será distinta de replica freshness.

## DB-CACHE-040
Active transaction será considerada por data caches.

## DB-CACHE-041
Shared Result Cache será bypassed en active transactions por default.

## DB-CACHE-042
Locking query bypassará shared data cache.

## DB-CACHE-043
Transaction-local cache será distinto de shared cache.

## DB-CACHE-044
Transaction-local state será scoped.

## DB-CACHE-045
Transaction-local cache se descartará en rollback.

## DB-CACHE-046
UNKNOWN transaction outcome no promoverá cache entries.

## DB-CACHE-047
Cache publication dependiente de commit requerirá confirmed commit.

## DB-CACHE-048
Cache invalidation failure posterior a commit no deshará el commit.

## DB-CACHE-049
Cache invalidation failure será observable.

## DB-CACHE-050
Rollback no equivaldrá a cache invalidation.

## DB-CACHE-051
Entity Cache no sustituirá UnitOfWork.

## DB-CACHE-052
Entity Cache no sustituirá IdentityMap.

## DB-CACHE-053
Existing managed entity será reutilizada.

## DB-CACHE-054
Cache hit no sobrescribirá dirty entity silenciosamente.

## DB-CACHE-055
Entity Cache preferirá canonical persistent state.

## DB-CACHE-056
Live PHP entity serialization no será default.

## DB-CACHE-057
Partial cached entity requerirá loaded-field semantics.

## DB-CACHE-058
CacheCodec será distinto de cache semantics.

## DB-CACHE-059
Payloads persistentes serán versionables.

## DB-CACHE-060
Incompatible payload será miss/invalid, no reinterpretación insegura.

## DB-CACHE-061
Corrupt optional entry podrá evictarse y reconstruirse.

## DB-CACHE-062
Optional cache failure podrá ser FAIL_OPEN.

## DB-CACHE-063
Cache outage no corromperá database semantics.

## DB-CACHE-064
Cache stampede será bounded.

## DB-CACHE-065
Distributed locking no será capability obligatoria.

## DB-CACHE-066
Cache fill lock tendrá timeout.

## DB-CACHE-067
Stale-while-revalidate requerirá policy compatible.

## DB-CACHE-068
Stale-if-error requerirá opt-in explícito.

## DB-CACHE-069
Negative hit será distinto de miss.

## DB-CACHE-070
Cached null será distinto de miss.

## DB-CACHE-071
Missing podrá ser distinto de null.

## DB-CACHE-072
Multi-level caches coordinarán generations cuando sea necesario.

## DB-CACHE-073
Invalidar L2 no implicará automáticamente invalidar L1.

## DB-CACHE-074
Generation tokens podrán coordinar L1/L2.

## DB-CACHE-075
Logical invalidation no requerirá siempre physical deletion.

## DB-CACHE-076
Old generations podrán limpiarse posteriormente.

## DB-CACHE-077
Fine-grained generation tendrá costo explícito.

## DB-CACHE-078
Query dependency extraction podrá consumir semantic query information.

## DB-CACHE-079
Persistence invalidation podrá consumir ChangeSet semantics.

## DB-CACHE-080
Database Cache no parseará generated SQL para reconstruir ORM intent cuando ya exista información semántica.

## DB-CACHE-081
External mutation awareness será una capability/integration.

## DB-CACHE-082
Sharding será parte de cache identity cuando sea relevante.

## DB-CACHE-083
Shard movement no dejará physical-location identity permanente.

## DB-CACHE-084
Replica source podrá afectar freshness evidence.

## DB-CACHE-085
Sticky routing no hará fresco un stale cache.

## DB-CACHE-086
Failover no invalidará structural cache por sí mismo.

## DB-CACHE-087
Load balancing no redefinirá semantic cache identity.

## DB-CACHE-088
Schema changes podrán invalidar structural/data cache artifacts.

## DB-CACHE-089
Metadata generation será observable.

## DB-CACHE-090
Type registry generation podrá participar en fingerprints.

## DB-CACHE-091
Custom type cache representation será explícita.

## DB-CACHE-092
JSON semantics se preservarán.

## DB-CACHE-093
Temporal semantics se preservarán.

## DB-CACHE-094
Enum identity será estable.

## DB-CACHE-095
Optimistic version podrá formar parte de Entity Cache metadata.

## DB-CACHE-096
Cache hit no sustituirá pessimistic lock.

## DB-CACHE-097
Concurrency control no será delegado al cache.

## DB-CACHE-098
Serializable transaction no utilizará shared cache sin prueba explícita.

## DB-CACHE-099
Cache lookup respetará operation deadline.

## DB-CACHE-100
Cache lookup no consumirá ilimitadamente el query budget.

## DB-CACHE-101
Cache payload size será bounded.

## DB-CACHE-102
Dependency count será bounded.

## DB-CACHE-103
Cache admission será distinta de cacheability.

## DB-CACHE-104
Cacheable no significará ShouldCache.

## DB-CACHE-105
Structural caches podrán calentarse.

## DB-CACHE-106
Data cache warming será explícito.

## DB-CACHE-107
Cold cache será estado válido.

## DB-CACHE-108
Optional cache vacío no cambiará correctness.

## DB-CACHE-109
Sensitive fields podrán excluirse.

## DB-CACHE-110
Excluded fields preservarán loaded-field semantics.

## DB-CACHE-111
Untrusted cache payload no cargará clases arbitrarias.

## DB-CACHE-112
Unsafe unserialize estará prohibido por default.

## DB-CACHE-113
Cache key size será bounded.

## DB-CACHE-114
Una request no generará cache amplification ilimitada.

## DB-CACHE-115
Telemetry será observational.

## DB-CACHE-116
Telemetry no modificará cache decisions.

## DB-CACHE-117
Metrics tendrán cardinalidad bounded.

## DB-CACHE-118
Raw sensitive cache payload no aparecerá en telemetry.

## DB-CACHE-119
CacheDecision será explícita.

## DB-CACHE-120
UNKNOWN cacheability producirá bypass por default.

## DB-CACHE-121
Developer cache request no podrá romper invariants.

## DB-CACHE-122
Result Cache no estará habilitado indiscriminadamente.

## DB-CACHE-123
Entity Cache será policy-driven.

## DB-CACHE-124
Safe structural caches podrán ser automáticos.

## DB-CACHE-125
Non-deterministic query requerirá cacheability policy explícita.

## DB-CACHE-126
Connection-local state podrá hacer una query non-cacheable.

## DB-CACHE-127
Temporary-table queries no serán shared-cacheable por default.

## DB-CACHE-128
Unknown DB function determinism no será tratado como deterministic.

## DB-CACHE-129
Cache bypass será siempre un camino válido para optional caches.

## DB-CACHE-130
IdentityMap lookup precederá Entity Cache cuando la entidad ya esté managed.

## DB-CACHE-131
Cache-aside será compatible con la arquitectura.

## DB-CACHE-132
Write-behind no será persistence default.

## DB-CACHE-133
Cache update e invalidation serán estrategias distintas.

## DB-CACHE-134
Relationship mutations podrán afectar múltiples dependencies.

## DB-CACHE-135
Cache hits no eliminarán semantic N+1 detection.

## DB-CACHE-136
Cache benefit considerará lookup y consistency cost.

## DB-CACHE-137
Process-local cache será preferible para ciertos immutable artifacts.

## DB-CACHE-138
Persistent workers podrán conservar immutable structural caches.

## DB-CACHE-139
Persistent workers no conservarán managed entities entre requests.

## DB-CACHE-140
Persistent workers no conservarán transaction-local cache entre requests.

## DB-CACHE-141
Persistent workers no conservarán tenant-local mutable state.

## DB-CACHE-142
Cache generations deberán funcionar durante rolling deployments.

## DB-CACHE-143
Blue/green versions podrán utilizar namespaces compatibles o separados.

## DB-CACHE-144
Global backend flush no será mecanismo normal de invalidation.

## DB-CACHE-145
CLI clear preferirá scoped/generation invalidation.

## DB-CACHE-146
Cache health no equivaldrá a cache freshness.

## DB-CACHE-147
Cache retries serán bounded.

## DB-CACHE-148
Cache retry será distinto de query retry.

## DB-CACHE-149
Cache write failure posterior a DB read no convertirá automáticamente el read en failure.

## DB-CACHE-150
Reliable invalidation podrá integrarse con Outbox.

## DB-CACHE-151
Outbox delivery delay será reconocido como consistency window.

## DB-CACHE-152
Database-specific consistency tokens serán capability-driven.

## DB-CACHE-153
Cache architecture funcionará sin distributed backend.

## DB-CACHE-154
SQLite no requerirá Redis.

## DB-CACHE-155
Zero-config Database seguirá siendo objetivo.

## DB-CACHE-156
Data caches serán opt-in inicialmente.

## DB-CACHE-157
Testing podrá controlar cache clock.

## DB-CACHE-158
Testing podrá inyectar cache faults.

## DB-CACHE-159
Concurrent miss behavior será testeable.

## DB-CACHE-160
Tenant cache isolation será testeable.

## DB-CACHE-161
Shard cache isolation será testeable.

## DB-CACHE-162
Transaction cache behavior será testeable.

## DB-CACHE-163
Codec evolution será testeable.

## DB-CACHE-164
Cache subsystem no será propietario de database truth.

## DB-CACHE-165
Cache Manager no será God Object.

## DB-CACHE-166
Cada specialized cache definirá su propia semantic identity.

## DB-CACHE-167
Shared infrastructure no eliminará specialized semantics.

## DB-CACHE-168
Cache read pipeline validará payload antes de HIT.

## DB-CACHE-169
Cache write pipeline extraerá dependencies antes de publication cuando sean necesarias.

## DB-CACHE-170
Mutation invalidation observará transaction outcome.

---

# 290. Fórmula arquitectónica

La arquitectura puede resumirse como:

```text
DatabaseCacheEntry
=
SemanticIdentity
+
ExecutionDomain
+
CanonicalPayload
+
Dependencies
+
Generation
+
Validity
+
ConsistencyEvidence
```

El lookup será:

```text
CacheLookup
=
ResolveKey
→ ResolveNamespace
→ Lookup
→ Decode
→ ValidateGeneration
→ ValidateDependencies
→ ValidateConsistency
→ Decision
```

---

# 291. Fórmula de seguridad

Para una entrada `C` y operación `O`:

```text
Use(C, O)
iff
IdentityMatches(C, O)
∧
DomainMatches(C, O)
∧
GenerationValid(C)
∧
ConsistencyPolicyAllows(C, O)
∧
SecurityPolicyAllows(C, O)
```

Si alguna condición es desconocida:

```text
UNKNOWN
→ BYPASS
```

por default.

---

# 292. Modelo de consistencia

```text
Database Truth
      │
      ▼
Authoritative Read
      │
      ▼
Cache Candidate
      │
      ▼
Consistency Metadata
      │
      ▼
Cache Entry
      │
      ▼
Future Lookup
      │
      ▼
Revalidation / Policy
      │
      ├── usable
      │
      └── bypass
```

---

# 293. Arquitectura de mutación

```text
                    Application
                         │
                         ▼
                     UnitOfWork
                         │
                         ▼
                 Persistence Engine
                         │
                         ▼
                    Transaction
                         │
             ┌───────────┴───────────┐
             │                       │
          COMMIT                  ROLLBACK
             │                       │
             ▼                       ▼
     Invalidation Publish       Discard Intent
             │
             ▼
     Shared Database Cache
```

Con tercer caso:

```text
UNKNOWN
   ↓
Conservative Consistency Policy
```

---

# 294. Arquitectura con VoltStack Cache

Integración futura:

```text
VoltStack/Quantum/Database
           │
           ▼
    Database Cache Contracts
           │
           ▼
   Database Cache Integration
           │
           ▼
    VoltStack/Quantum/Cache
           │
    ┌──────┼─────────┐
    ▼      ▼         ▼
 Memory   Redis    Other
```

La dependencia concreta estará invertida mediante contratos/adapters.

---

# 295. Arquitectura completa

```text
                         APPLICATION
                              │
                              ▼
                             ORM
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
             IdentityMap              Entity Cache
                 │                         │
                 └────────────┬────────────┘
                              ▼
                         Query Engine
                              │
             ┌────────────────┼────────────────┐
             ▼                ▼                ▼
         Query Cache     Metadata Cache    Plan Caches
             │                │                │
             └────────────────┼────────────────┘
                              ▼
                         SQL Compiler
                              │
                              ▼
                    Compiled Query Cache
                              │
                              ▼
                           Executor
                              │
                     ┌────────┴────────┐
                     ▼                 ▼
                 Result Cache      Database
                     │                 │
                     └────────┬────────┘
                              ▼
                           Result
```

Mutations:

```text
Persistence
    ↓
Transaction
    ↓
Outcome
    ↓
Invalidation Coordinator
    ↓
Query / Result / Entity Cache
```

---

# 296. Resultado arquitectónico

Con este diseño, VoltStack podrá utilizar caching sin convertirlo en una fuente oculta de inconsistencias.

La regla operacional será:

```text
Structural information
→ cache aggressively when generation-safe

Persistent data
→ cache only under explicit consistency semantics
```

Y:

```text
Performance Optimization
<
Correctness
<
Isolation
<
Security
```

donde la optimización nunca podrá invalidar las garantías superiores.

---

# 297. Relación con documentos anteriores

Esta arquitectura integra directamente:

```text
075_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
123_DATABASE_IDENTITY_MAP_SYSTEM.md
141_DATABASE_HYDRATION_CACHE_SYSTEM.md
164_DATABASE_TRANSACTION_ARCHITECTURE.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
176_DATABASE_READ_WRITE_CONNECTION_SYSTEM.md
177_DATABASE_READ_WRITE_ROUTING_SYSTEM.md
178_DATABASE_REPLICA_SYSTEM.md
179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
183_DATABASE_DISTRIBUTED_DATABASE_ARCHITECTURE.md
184_DATABASE_SHARDING_SYSTEM.md
185_DATABASE_PARTITION_ROUTING_SYSTEM.md
```

El principio unificador será:

```text
Cache consumes semantics from these systems.
Cache does not redefine their semantics.
```

---

# 298. Bloque 17 — Cache

La secuencia queda:

```text
186_DATABASE_CACHE_ARCHITECTURE.md
    ↓
187_DATABASE_QUERY_CACHE_SYSTEM.md
    ↓
188_DATABASE_RESULT_CACHE_SYSTEM.md
    ↓
189_DATABASE_METADATA_CACHE_SYSTEM.md
    ↓
190_DATABASE_ENTITY_CACHE_SYSTEM.md
    ↓
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
    ↓
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

Cada documento profundizará una responsabilidad separada.

---

# 299. Regla maestra final

> **VoltStack Database tratará todo caché como una copia, representación o artefacto derivado cuya autoridad reside fuera del caché. La utilización de una entrada requerirá demostrar que su identidad, dominio, generación y política de consistencia son compatibles con la operación actual; cuando esa evidencia sea insuficiente, VoltStack preferirá un cache miss o bypass antes que fabricar certeza.**

Formalmente:

```text
CacheHit
=
EntryFound
∧
IdentityValid
∧
DomainValid
∧
GenerationValid
∧
ConsistencyValid
∧
SecurityValid
```

No simplemente:

```text
EntryFound
=
CacheHit
```

Y para caches opcionales:

```text
Cache Miss
Cache Empty
Cache Bypass
Cache Unavailable
        ↓
Authoritative Source
        ↓
Correct Result
```

Esta propiedad permitirá que caching sea realmente:

```text
Optimization
```

y nunca una dependencia accidental para la corrección fundamental del sistema.

---

# 300. Siguiente documento

```text
187_DATABASE_QUERY_CACHE_SYSTEM.md
```

El siguiente documento definirá específicamente el **Query Cache System**, incluyendo:

```text
Query Semantic Identity
Query Fingerprinting
AST Fingerprinting
Normalized Query Identity
Parameterized Query Identity
Query Plan Caching
Cacheability Analysis
Platform/Dialect Separation
Metadata Generations
Compiler Generations
Query Cache Keys
Query Cache Entries
Query Cache Lifecycle
Query Cache Invalidation
Query Cache L1/L2
Persistent Runtime Reuse
Query Cache Telemetry
Query Cache Diagnostics
```

manteniendo la separación crítica:

```text
Query Cache
≠
Result Cache
```

porque:

```text
Query Cache
→ reuses work required to understand/prepare a query

Result Cache
→ reuses data previously obtained by executing a query
```

Esta distinción será una de las bases fundamentales del sistema de caching de `VoltStack/Quantum/Database`.