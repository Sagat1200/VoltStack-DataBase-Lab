# 190_DATABASE_ENTITY_CACHE_SYSTEM.md

# VoltStack Quantum Database
## Entity Cache System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 190 — Entity Cache System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `189_DATABASE_METADATA_CACHE_SYSTEM.md`  
**Siguiente documento:** `191_DATABASE_CACHE_INVALIDATION_SYSTEM.md`

---

# 1. Propósito

`Entity Cache System` define la arquitectura de caché de segundo nivel orientada a entidades de VoltStack.

Su propósito será reutilizar representaciones persistentes de entidades entre distintos:

- requests;
- jobs;
- comandos;
- workers;
- procesos;
- nodos de aplicación;

sin romper las garantías del:

- `IdentityMap`;
- `UnitOfWork`;
- `EntityManager`;
- sistema de tipos;
- hydration;
- optimistic locking;
- transacciones;
- multitenancy;
- sharding;
- read/write routing;
- cache consistency.

Regla principal:

> **Entity Cache almacenará representaciones persistentes y versionadas de entidades, nunca instancias administradas por un EntityManager.**

Formalmente:

```text
EntityCache
=
CrossScopeReusablePersistentEntityState
```

mientras:

```text
IdentityMap
=
ScopeLocalCanonicalObjectIdentity
```

Por tanto:

```text
EntityCache
≠
IdentityMap
```

---

# 2. Problema

Considérese:

```php
$user = User::find($id);
```

Sin Entity Cache:

```text
ORM
 ↓
Query Engine
 ↓
Database
 ↓
Result
 ↓
Hydration
 ↓
User
```

Cada nuevo request puede repetir la consulta.

Con Entity Cache:

```text
ORM
 ↓
IdentityMap
 │
 ├── HIT
 │    ↓
 │  managed User
 │
 └── MISS
      ↓
 Entity Cache
      │
      ├── HIT
      │    ↓
      │ Cache Entry
      │    ↓
      │ Hydration/Reconstruction
      │    ↓
      │ IdentityMap
      │    ↓
      │ managed User
      │
      └── MISS
           ↓
        Database
```

---

# 3. Dos niveles de identidad/cache

La arquitectura distingue:

```text
L1 = IdentityMap
L2 = EntityCache
```

## L1

Scope local:

```text
Request / Job / Operation
```

Contiene:

```text
actual PHP object instances
```

## L2

Cross-scope:

```text
Process
Distributed Cache
Redis
other cache backend
```

Contiene:

```text
serialized/canonical persistent state
```

Nunca objetos administrados.

---

# 4. Regla de precedencia

Para entidades administradas:

```text
IdentityMap
>
EntityCache
>
Database
```

pero esta fórmula representa el orden de lookup, no autoridad absoluta sobre la verdad.

---

# 5. Autoridad

Debe distinguirse:

```text
IdentityMap
=
canonical object identity inside current PersistenceContext
```

```text
EntityCache
=
reusable cached persistent representation
```

```text
Database
=
persistent storage authority
```

---

# 6. Entity Cache no reemplaza la base de datos

Un cache hit significa:

```text
a reusable representation is available
```

No necesariamente:

```text
this is the newest possible database state
```

Eso dependerá de la política de consistencia.

---

# 7. Entity Cache ≠ Result Cache

`Result Cache` almacena:

```text
Query Q
→ Result R
```

`Entity Cache` almacena:

```text
EntityKey E
→ PersistentEntityState S
```

---

# 8. Ejemplo

Result Cache:

```text
SELECT users WHERE status = active
→ [1, 4, 8, 11]
```

Entity Cache:

```text
User#1  → state
User#4  → state
User#8  → state
User#11 → state
```

---

# 9. Query membership

Entity Cache no sabe necesariamente:

```text
which users satisfy status = active
```

Eso pertenece a query/result caching.

---

# 10. Entity Cache ≠ Metadata Cache

```text
Metadata Cache
→ what User means
```

```text
Entity Cache
→ persistent state of User#42
```

---

# 11. Entity Cache ≠ object cache

No deberá almacenarse:

```php
$redis->set('user:42', serialize($managedUser));
```

como arquitectura ORM oficial.

---

# 12. Razón

Una entidad administrada puede contener:

```text
EntityManager references
PersistentCollection
Lazy loaders
Proxy state
Runtime context
Pending changes
Relationship state
Lifecycle state
```

que no debe cruzar scopes.

---

# 13. EntityCacheEntry

VoltStack utilizará una representación explícita.

```php
final readonly class EntityCacheEntry
{
    public function __construct(
        public EntityCacheKey $key,
        public EntityMetadataGeneration $metadataGeneration,
        public EntityCacheVersion $cacheVersion,
        public EntityFieldState $fields,
        public LoadedFieldMask $coverage,
        public ?OptimisticVersion $optimisticVersion,
        public EntityCacheTimestamp $createdAt,
    ) {}
}
```

---

# 14. Entry ≠ Entity

Regla:

```text
EntityCacheEntry
≠
PHP Entity Object
```

---

# 15. EntityCacheKey

Identidad conceptual:

```text
EntityCacheKey
=
EntityIdentityNamespace
+
EntityType
+
CanonicalIdentifier
+
PersistenceDomain
```

---

# 16. PersistenceDomain

Debe evitar colisiones entre:

```text
database
tenant
shard
application
environment/deployment namespace
```

cuando sean semánticamente relevantes.

---

# 17. Fórmula ampliada

```text
EntityCacheKey(E)
=
ApplicationNamespace
× EntityType
× CanonicalIdentifier
× DatabaseDomain
× TenantDomain?
× ShardDomain?
```

---

# 18. Tenant isolation

Dos tenants:

```text
Tenant A → User#42
Tenant B → User#42
```

deben producir claves diferentes.

```text
CacheKey(A, User, 42)
≠
CacheKey(B, User, 42)
```

---

# 19. Shard isolation

Igualmente:

```text
Shard 1 → Order#100
Shard 7 → Order#100
```

no podrán colisionar.

---

# 20. EntityKey integration

El sistema deberá reutilizar la semántica de `EntityKey` definida por `IdentityMap`.

No crear una segunda noción incompatible de identidad.

---

# 21. EntityCacheKey ≠ EntityKey

Sin embargo pueden existir diferencias operacionales.

`EntityKey`:

```text
ORM canonical identity
```

`EntityCacheKey`:

```text
cache namespaced representation of canonical identity
```

---

# 22. Canonical identifier

IDs deberán normalizarse.

Ejemplos:

```text
42
"42"
UUID object
UUID string
binary UUID
```

no deberán producir accidentalmente cuatro identidades cuando semánticamente representan una.

---

# 23. Type System integration

La canonicalización del identificador deberá utilizar:

```text
Type System
Value Conversion
Entity Metadata
```

y no concatenación arbitraria.

---

# 24. Composite identifiers

Debe soportarse:

```text
OrderLine(
    orderId = 100,
    line = 4
)
```

mediante representación determinista.

---

# 25. No ambiguous concatenation

No:

```text
"100:4"
```

si puede generar ambigüedad.

Preferir estructura canónica/fingerprint.

---

# 26. Cache entry state

El payload almacenará únicamente estado persistente permitido.

Ejemplo:

```text
User#42
{
    id: 42,
    email: "user@example.com",
    status: "active",
    version: 7
}
```

---

# 27. No transient fields

Propiedades:

```text
#[Transient]
private bool $selected;
```

no deberán formar parte del entry salvo una extensión explícita.

---

# 28. No runtime fields

Nunca:

```text
isManaged
isDirty
currentEntityManager
lazyLoader
transactionContext
```

---

# 29. Canonical persistent representation

Idealmente:

```text
Application Value
      ↓
Casting
      ↓
Canonical Persistent Value
```

será la base semántica del payload.

---

# 30. Driver raw values

Evitar cachear directamente:

```text
PDO-specific values
driver buffers
resource handles
```

---

# 31. Cache representation

La representación debe ser:

```text
driver-independent
platform-aware only where necessary
versioned
typed
serializable
```

---

# 32. Field state

```php
final readonly class EntityFieldState
{
    /**
     * @param array<FieldId, CanonicalPersistentValue> $values
     */
    public function __construct(
        public array $values,
    ) {}
}
```

---

# 33. FieldId

Preferir identificadores estables:

```text
EntityType + FieldId
```

sobre depender únicamente del nombre PHP de la propiedad.

---

# 34. Razón

Refactors como:

```text
$emailAddress
→
$email
```

pueden requerir una transición controlada de metadata/cache.

---

# 35. Metadata generation

Cada entry deberá estar vinculado a:

```text
EntityMetadataGeneration
```

o fingerprint equivalente.

---

# 36. Mapping incompatibility

Si cambia:

```text
User.status:
string
→ enum
```

un entry anterior podría ser incompatible.

---

# 37. Metadata mismatch

Entonces:

```text
Cache Entry G1
+
Entity Metadata G2
→ MISS / REJECT
```

Nunca interpretar silenciosamente payload incompatible.

---

# 38. LoadedFieldMask

Entity Cache deberá conocer qué campos contiene.

```text
LoadedFieldMask
```

permite distinguir:

```text
NULL
```

de:

```text
MISSING
```

---

# 39. NULL ≠ MISSING

Regla:

```text
field exists with NULL
≠
field absent from cache entry
```

---

# 40. Full entity entries

Por defecto, el Entity Cache debería favorecer:

```text
complete cacheable persistent entity state
```

---

# 41. Partial entries

Los entries parciales introducen:

```text
coverage merging
stale field combinations
complex invalidation
partial entity hazards
```

Por tanto deberán estar restringidos.

---

# 42. Default policy

```text
FULL_ENTITY_ONLY
```

será la política recomendada.

---

# 43. Partial cache policy

Opcionalmente:

```php
enum EntityCacheCoveragePolicy
{
    case FULL_ENTITY_ONLY;
    case ALLOW_PARTIAL;
}
```

---

# 44. Partial entry identity

Si se permiten entries parciales, la coverage deberá formar parte de su semántica.

Nunca:

```text
partial entry
→ pretend complete entity
```

---

# 45. Hydration integration

Cache hit:

```text
EntityCacheEntry
      ↓
Entity Cache Hydrator
      ↓
Entity instance
      ↓
IdentityMap
      ↓
UnitOfWork baseline
```

---

# 46. EntityCacheHydrator

No deberá duplicar completamente el sistema de hydration.

Preferir reutilizar:

```text
Hydration Metadata
Type/Casting Pipeline
Entity Construction
IdentityMap
Snapshot establishment
```

---

# 47. Cache hydration ≠ DB result hydration

El source cambia:

```text
DB Result Row
```

vs:

```text
EntityCacheEntry
```

pero ambas rutas deben converger en las mismas invariantes ORM.

---

# 48. IdentityMap lookup primero

Antes de materializar un cache entry:

```text
IdentityMap::find(EntityKey)
```

debe comprobarse.

---

# 49. Existing managed entity

Si ya existe:

```text
User#42
```

administrado:

```text
EntityCache
```

no creará una segunda instancia.

---

# 50. Identity invariant

Dentro de un `PersistenceContext`:

```text
same EntityKey
→ same canonical managed object
```

aunque la fuente sea:

```text
database
entity cache
relationship loading
query hydration
```

---

# 51. Dirty managed entity

Caso crítico:

```text
IdentityMap:
User#42.email = "new@example.com"
DIRTY

EntityCache:
User#42.email = "old@example.com"
```

El cache no deberá sobrescribir la entidad dirty.

---

# 52. Merge rule

Por defecto:

```text
Managed Dirty State
>
Entity Cache State
```

---

# 53. No blind refresh

Un cache hit nunca implica:

```text
overwrite all managed fields
```

---

# 54. Explicit refresh

Una operación:

```php
$entityManager->refresh($user);
```

tendrá políticas propias y podrá:

```text
bypass entity cache
```

por default cuando se requiera DB freshness.

---

# 55. Refresh semantics

Debe distinguirse:

```text
refreshFromDatabase()
refreshFromCache()
```

si ambas APIs llegan a exponerse.

---

# 56. UnitOfWork baseline

Cuando una entidad se crea desde cache:

```text
cache canonical state
```

podrá convertirse en:

```text
UnitOfWork baseline snapshot
```

solo si el entry es válido y suficientemente completo.

---

# 57. Cache validity ≠ DB freshness

El baseline representa:

```text
ORM-known persistent baseline
```

no necesariamente prueba de que DB no cambió un microsegundo después.

---

# 58. Optimistic locking

Entity Cache debe integrarse estrechamente con:

```text
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
```

---

# 59. Version field

Si una entidad posee:

```php
#[Version]
private int $version;
```

el cache entry deberá conservar el valor.

---

# 60. Example

```text
EntityCacheEntry:
    User#42
    version = 17
```

---

# 61. Update

Después de update confirmado:

```text
version 17
→ version 18
```

el cache no deberá continuar sirviendo 17 como si fuera actual.

---

# 62. Version-aware invalidation

Podrá realizarse:

```text
invalidate(User#42, version=17)
```

o:

```text
replace(User#42, version=18)
```

según política.

---

# 63. Compare-and-set cache writes

Backends capaces podrán usar:

```text
CAS
```

para evitar que una actualización antigua sobrescriba una nueva.

---

# 64. Monotonic entity versions

Cuando el modelo tenga version token monotónico:

```text
CacheVersion(18)
>
CacheVersion(17)
```

permitirá rechazar stale writes.

---

# 65. Entity cache version ≠ optimistic version

Deben distinguirse.

```text
OptimisticVersion
=
domain/persistence concurrency token
```

```text
EntityCacheVersion
=
cache entry revision/control token
```

Pueden correlacionarse, pero no son necesariamente iguales.

---

# 66. Non-versioned entities

También podrán cachearse, pero su invalidación será menos robusta.

---

# 67. Recommended policy

Para entidades altamente mutables y críticas:

```text
optimistic versioning
+
entity cache
```

es preferible.

---

# 68. Transaction awareness

Un Entity Cache compartido no deberá publicar datos no comprometidos.

Regla:

> **Una mutación de Entity Cache derivada de una transacción no será visible globalmente antes de que el commit de la transacción sea confirmado.**

---

# 69. Incorrect sequence

Nunca:

```text
UPDATE users
↓
EntityCache.put(new state)
↓
COMMIT
```

porque commit puede fallar.

---

# 70. Correct conceptual sequence

```text
UPDATE users
↓
Transaction still ACTIVE
↓
register deferred cache mutation
↓
COMMIT confirmed
↓
publish cache mutation
```

---

# 71. Deferred cache mutation

```php
interface DeferredEntityCacheMutation
{
    public function apply(): void;
}
```

---

# 72. Transaction resource integration

El `TransactionResourceRegistry` podrá contener:

```text
cache.deferred_entity_mutations
```

---

# 73. Rollback

Si:

```text
ROLLBACK confirmed
```

las mutaciones pendientes de cache deberán descartarse.

---

# 74. Unknown commit

Caso crítico:

```text
COMMIT sent
↓
connection lost
↓
outcome UNKNOWN
```

VoltStack no sabe si DB confirmó.

---

# 75. UNKNOWN rule

Nunca:

```text
UNKNOWN
→ publish new cache value
```

como si el commit fuera confirmado.

---

# 76. Safe default

Para outcome UNKNOWN:

```text
invalidate affected entity keys
```

si es posible.

---

# 77. Why invalidate

Si DB:

```text
committed
```

el old cache es stale.

Si DB:

```text
rolled back
```

el new cache sería incorrecto.

Invalidation evita afirmar cualquiera de los dos.

---

# 78. Invalidation failure

Si además falla invalidation:

```text
DB outcome UNKNOWN
+
cache invalidation failed
```

deberá elevarse telemetría/consistency risk.

---

# 79. Transaction-local visibility

Dentro de una transacción:

```text
EntityManager/IdentityMap
```

deberá tener prioridad sobre shared Entity Cache.

---

# 80. Read-your-own-writes

Si una entidad fue modificada dentro de la transacción:

```text
shared cache old value
```

no deberá reemplazar la versión local.

---

# 81. Shared cache bypass

Después de una write dentro de la misma transaction:

```text
EntityCache lookup
```

para esa misma entidad podrá ser bypassed.

---

# 82. Transaction-local dirty set

El transaction/persistence context podrá conocer:

```text
modified EntityKeys
```

sin introducir ese estado en el Entity Cache global.

---

# 83. Flush ≠ commit

Después de:

```php
$entityManager->flush();
```

pero antes de commit:

```text
shared Entity Cache
```

no deberá actualizarse.

---

# 84. Autocommit operations

Si una operación se ejecuta mediante una transacción interna:

```text
cache publication
```

solo ocurrirá después de outcome confirmado.

---

# 85. Cache population on read

Lecturas podrán utilizar:

```text
read-through
```

---

# 86. Read-through flow

```text
IdentityMap MISS
      ↓
EntityCache MISS
      ↓
Database
      ↓
Hydrate
      ↓
IdentityMap
      ↓
EntityCache population
```

---

# 87. Transactional read population

Debe evitarse poblar cache con una vista cuya semántica no sea apropiada globalmente.

Ejemplo:

```text
transaction snapshot
```

puede observar estado que ya no corresponde a la vista global actual.

---

# 88. Default conservative rule

Lecturas ordinarias fuera de contextos especiales podrán poblar L2.

Lecturas bajo:

```text
historical snapshot
special isolation
temporal query
locking read
```

requerirán políticas explícitas.

---

# 89. Locking reads

Una consulta:

```text
FOR UPDATE
```

no implica que el resultado sea automáticamente apropiado para shared cache population.

---

# 90. Temporal entities

Consultas históricas:

```text
User#42 AS OF T1
```

no deberán usar la misma key que:

```text
User#42 CURRENT
```

---

# 91. Temporal cache domain

Si se soporta caching temporal:

```text
TemporalSnapshotIdentity
```

deberá formar parte de la key.

---

# 92. Soft deletes

Estado de soft delete forma parte del persistent entity state cuando el modelo lo define.

---

# 93. Global scopes

Un entity cache entry no deberá depender implícitamente de un global query scope que cambie el significado de la entidad.

---

# 94. Visibility vs existence

Puede existir:

```text
EntityCacheEntry(User#42)
```

aunque una política de autorización no permita al usuario actual verlo.

Por tanto:

```text
cache hit
≠
authorization granted
```

---

# 95. Security rule

Entity Cache nunca sustituirá:

```text
Authorization
Data Access Policy
Tenant Isolation
```

---

# 96. Authorization

El ORM deberá aplicar las reglas de acceso correspondientes incluso si el dato proviene de cache.

---

# 97. Tenant security

La separación por tenant debe ocurrir en la key antes del lookup.

No:

```text
lookup shared User#42
↓
then check tenant
```

---

# 98. Encryption

Datos sensibles almacenados en Entity Cache podrán requerir:

```text
encrypted cache backend
encrypted payload
restricted caching
```

según policy.

---

# 99. Sensitive fields

Algunos fields podrán declararse:

```text
NON_CACHEABLE
```

---

# 100. Entity cacheability metadata

```php
enum EntityCacheability
{
    case CACHEABLE;
    case NON_CACHEABLE;
    case CONDITIONAL;
}
```

---

# 101. Field cacheability

Opcionalmente:

```php
enum FieldCacheability
{
    case CACHEABLE;
    case EXCLUDE;
    case SENSITIVE;
}
```

---

# 102. Excluded fields

Si un field persistente requerido es excluido:

```text
entry becomes partial
```

y deberá respetar `LoadedFieldMask`.

---

# 103. Recommended security default

Si excluir un field hace insegura la reconstrucción completa:

```text
entity becomes non-cacheable
```

en vez de fingir completitud.

---

# 104. Relationships

Entity Cache deberá separar:

```text
entity scalar/value state
```

de:

```text
relationship graph state
```

---

# 105. To-one relationships

Puede almacenarse:

```text
organization_id = 10
```

como FK/persistent reference state.

No necesariamente:

```text
Organization object
```

---

# 106. Relationship references

Una referencia podrá representarse como:

```text
EntityReference(
    EntityType,
    EntityKey
)
```

---

# 107. To-many relationships

No deberán serializarse arbitrariamente:

```text
User.posts = [Post objects...]
```

dentro del User cache entry.

---

# 108. Razón

To-many collections poseen:

```text
membership
ordering
coverage
pagination
filters
pending deltas
lazy state
```

con lifecycle independiente.

---

# 109. Relationship cache

Si posteriormente se desea cachear memberships:

```text
RelationshipCache
```

deberá ser un concepto especializado.

---

# 110. Default entity entry

Preferir:

```text
Entity fields
+
identifier
+
version
+
to-one FK references
```

sin graph serialization profunda.

---

# 111. No recursive graph cache

Nunca:

```text
User
 └── Organization
      └── Users
           └── Organization
```

serializado recursivamente.

---

# 112. Polymorphic relationships

Podrá almacenarse:

```text
stable morph alias
+
canonical target identifier
```

Nunca depender del FQCN persistido.

---

# 113. Morph rule

```text
PersistedMorphType
≠
PHPClassName
```

también aplica al cache.

---

# 114. Value Objects

Los value objects deberán cachearse mediante su:

```text
canonical persistent representation
```

no necesariamente serializando el objeto PHP.

---

# 115. Enum values

Preferir:

```text
canonical persisted enum token
```

sobre serializar internamente el enum object.

---

# 116. JSON

JSON deberá mantener:

```text
SQL NULL
JSON null
missing field
```

correctamente diferenciados.

---

# 117. Temporal values

Usarán representación canónica del `Date Time Type System`.

No timezone implícita del worker.

---

# 118. Cache codecs

```php
interface EntityCacheCodec
{
    public function encode(
        EntityCacheEntry $entry,
    ): string;

    public function decode(
        string $payload,
        EntityCacheDecodeContext $context,
    ): EntityCacheEntry;
}
```

---

# 119. Codec requirements

Debe ser:

```text
versioned
deterministic where practical
safe
bounded
explicit
```

---

# 120. PHP unserialize

No deberá ser la estrategia obligatoria/default para distributed entity cache payloads.

---

# 121. Entity cache store

```php
interface EntityCacheStore
{
    public function get(
        EntityCacheKey $key,
    ): ?EntityCachePayload;

    public function put(
        EntityCacheKey $key,
        EntityCachePayload $payload,
        EntityCacheWritePolicy $policy,
    ): void;

    public function delete(
        EntityCacheKey $key,
    ): void;
}
```

---

# 122. Batch API

Importante para evitar N+1 de cache:

```php
interface BatchEntityCacheStore
{
    /**
     * @return array<string, EntityCachePayload>
     */
    public function getMany(
        EntityCacheKeySet $keys,
    ): array;

    public function putMany(
        EntityCacheEntrySet $entries,
    ): void;
}
```

---

# 123. Cache N+1

Cambiar:

```text
100 DB queries
```

por:

```text
100 Redis requests
```

sigue siendo ineficiente.

---

# 124. Batch entity lookup

```text
100 EntityKeys
      ↓
EntityCache.getMany()
      ↓
75 HIT
25 MISS
      ↓
DB batch query for 25
```

---

# 125. Relationship loading integration

`Batch Relation Loading` podrá aprovechar batch entity cache lookups para targets conocidos.

---

# 126. Cache hit ratio

Debe medirse separando:

```text
L1 IdentityMap Hit
L2 Entity Cache Hit
DB Fetch
```

---

# 127. Lookup result

```php
enum EntityCacheLookupStatus
{
    case HIT;
    case MISS;
    case STALE;
    case INCOMPATIBLE;
    case CORRUPTED;
    case BYPASSED;
}
```

---

# 128. HIT ≠ accepted

Un payload encontrado todavía deberá pasar:

```text
metadata compatibility
version validation
coverage validation
domain validation
security/cacheability policy
```

---

# 129. Cache stampede

Entidades muy populares pueden provocar:

```text
entry expires
↓
1000 requests
↓
1000 DB queries
```

---

# 130. Single-flight

Opcional:

```text
EntityCacheSingleFlight
```

podrá coordinar reconstrucción.

---

# 131. Lock scope

Locks de cache nunca deberán convertirse en database transaction locks.

---

# 132. Lock timeout

Cache regeneration lock deberá ser bounded.

---

# 133. Stale-while-revalidate

Puede ser útil en algunos dominios, pero no será garantía default del ORM.

---

# 134. Reason

Servir datos stale puede ser incorrecto para:

```text
financial balances
permissions
inventory
workflow state
```

---

# 135. Policy-driven consistency

```php
enum EntityCacheConsistencyPolicy
{
    case STRICT_INVALIDATION;
    case VERSION_VALIDATED;
    case TTL_BOUNDED;
    case STALE_WHILE_REVALIDATE;
    case CUSTOM;
}
```

---

# 136. Default

Para ORM general:

```text
STRICT_INVALIDATION
```

o `VERSION_VALIDATED` cuando exista infraestructura suficiente.

---

# 137. TTL ≠ consistency

Regla:

```text
TTL
≠
ConsistencyGuarantee
```

---

# 138. Short TTL

Un TTL de un segundo aún puede servir un valor incorrecto durante ese segundo.

---

# 139. Invalidation architecture

El documento siguiente:

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
```

definirá el mecanismo transversal.

Entity Cache solo deberá declarar:

```text
what dependencies it has
what entity keys changed
what invalidation it requires
```

---

# 140. Write-through

Una estrategia:

```text
DB commit
↓
cache replace
```

---

# 141. Cache-aside invalidation

Otra:

```text
DB commit
↓
cache delete
↓
next read repopulates
```

---

# 142. Recommended default

Para correctness:

```text
commit
↓
invalidate
```

es más simple que intentar mantener automáticamente un payload nuevo en todos los casos.

---

# 143. Update cache optimization

Cuando VoltStack conoce inequívocamente:

```text
complete new persistent state
+
generated values
+
version
+
commit success
```

podrá realizar:

```text
replace
```

---

# 144. Unknown generated values

Si DB puede modificar mediante:

```text
trigger
generated column
server default
```

y VoltStack no conoce el resultado:

```text
invalidate
```

será preferible.

---

# 145. Delete

Después de delete confirmado:

```text
EntityCache.delete(EntityKey)
```

---

# 146. Negative entity cache

Podrá almacenar:

```text
User#999 → NOT_FOUND
```

temporalmente.

---

# 147. Negative cache danger

Después:

```text
INSERT User#999
```

el negative entry debe invalidarse.

---

# 148. Negative cache entry

```php
final readonly class NegativeEntityCacheEntry
{
    public function __construct(
        public EntityCacheKey $key,
        public EntityMetadataGeneration $generation,
        public Instant $createdAt,
        public Duration $ttl,
    ) {}
}
```

---

# 149. Negative cache policy

Deberá ser:

```text
short-lived
generation-aware
transaction-aware
invalidated on insert
```

---

# 150. Negative cache inside transaction

No deberá afirmar globalmente ausencia basándose en una snapshot transaccional especial sin policy explícita.

---

# 151. Cache invalidation race

Caso:

```text
T1 reads old state
T2 updates + commits
T2 invalidates cache
T1 writes old state into cache
```

Resultado:

```text
stale cache resurrection
```

---

# 152. Critical race

Entity Cache deberá protegerse contra:

```text
stale repopulation after invalidation
```

---

# 153. Generation/token strategy

Una solución:

```text
EntityInvalidationGeneration
```

---

# 154. Read token

```text
read generation = G7
↓
DB query
↓
before cache put
verify generation still G7
```

Si cambió:

```text
G8
```

no publicar.

---

# 155. Version strategy

Con optimistic version:

```text
DB result version 17
```

no deberá sobrescribir:

```text
cache version 18
```

---

# 156. CAS

```text
putIfNewer()
```

será útil para stores compatibles.

---

# 157. EntityCacheVersionComparator

```php
interface EntityCacheVersionComparator
{
    public function compare(
        EntityCacheVersion $left,
        EntityCacheVersion $right,
    ): CacheVersionOrder;
}
```

---

# 158. Unordered versions

No todos los version tokens son ordenables.

Ejemplo:

```text
UUID revision token
```

Entonces podrán requerirse:

```text
equality/CAS semantics
```

---

# 159. Database-generated version

Debe capturarse antes de publicar un new cache entry.

---

# 160. Replica integration

El sistema de replicas introduce otro riesgo.

```text
Primary:
version 18

Replica:
version 17
```

---

# 161. Stale replica repopulation

Después de invalidar 17:

```text
read replica returns 17
↓
cache repopulates 17
```

sería incorrecto.

---

# 162. Replica-lag awareness integration

`179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md` deberá proporcionar evidencia para impedir repoblación insegura.

---

# 163. Sticky connection integration

Después de write:

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

podrá forzar lectura del writer durante consistency window.

---

# 164. Cache population source

El cache entry podrá registrar metadata operacional:

```php
enum EntityCachePopulationSource
{
    case WRITER;
    case REPLICA;
    case CACHE_REBUILD;
}
```

sin incluir datos sensibles.

---

# 165. Replica cache policy

Ejemplos:

```text
ALLOW_REPLICA_POPULATION
WRITER_ONLY_POPULATION
VERSION_VALIDATED_REPLICA
```

---

# 166. Default conservative policy

Para entidades versionadas y sensibles:

```text
writer-confirmed population
```

o version validation.

---

# 167. Failover integration

Tras failover:

```text
old primary
→ new primary
```

la validez de cache dependerá de garantías de replicación/topology.

---

# 168. Failover does not imply flush-all

No deberá ejecutarse automáticamente:

```text
flush entire cache
```

sin necesidad.

Pero deberá existir capacidad de:

```text
domain generation bump
```

si la consistencia no puede demostrarse.

---

# 169. Distributed database

En arquitectura distribuida:

```text
EntityCacheKey
```

debe contener suficiente contexto para evitar colisiones entre dominios de persistencia.

---

# 170. Sharding

El cache podrá ser:

```text
globally distributed
```

mientras sus keys permanezcan shard-aware.

---

# 171. Shard movement

Si una entidad cambia de shard:

```text
old cache domain
```

deberá invalidarse.

---

# 172. Partition routing

El Entity Cache no deberá decidir:

```text
which shard owns entity
```

salvo mediante integración con `Partition Routing`.

---

# 173. Routing before cache lookup

Si la identidad depende del shard:

```text
Partition Router
↓
Persistence Domain
↓
EntityCacheKey
↓
Entity Cache
```

---

# 174. Cache before routing?

Solo será válido si existe una:

```text
global canonical entity namespace
```

que garantice unicidad.

No deberá asumirse por default.

---

# 175. Multi-database entities

Una misma entity type puede mapearse a distintas bases.

Por tanto:

```text
EntityType + ID
```

no siempre basta.

---

# 176. EntityCacheRegion

VoltStack podrá agrupar entries en regiones.

```php
final readonly class EntityCacheRegion
{
    public function __construct(
        public EntityCacheRegionId $id,
        public EntityCachePolicy $policy,
    ) {}
}
```

---

# 177. Regions

Ejemplos:

```text
users
catalog
products
reference-data
billing
```

---

# 178. Region purpose

Permiten políticas diferentes:

```text
TTL
consistency
backend
encryption
capacity
eviction
```

---

# 179. Region ≠ table

No es obligatorio:

```text
one table = one cache region
```

---

# 180. Entity cache metadata

Entity mapping podrá declarar:

```php
#[Cacheable(
    region: 'catalog',
    policy: 'versioned'
)]
final class Product
{
}
```

---

# 181. Model API

Una API equivalente podrá ser:

```php
final class Product extends Model
{
    protected static function cache(): EntityCacheConfiguration
    {
        return EntityCacheConfiguration::versioned(
            region: 'catalog',
        );
    }
}
```

Ambas convergen en metadata canónica.

---

# 182. No static runtime cache state

La configuración puede declararse estáticamente.

Pero:

```text
cache connection
cache entries
current cache context
```

no vivirán como estado estático del Model.

---

# 183. EntityCacheManager

```php
interface EntityCacheManager
{
    public function find(
        EntityKey $key,
        EntityCacheReadContext $context,
    ): EntityCacheLookupResult;

    public function invalidate(
        EntityKey $key,
        EntityCacheInvalidationContext $context,
    ): void;
}
```

---

# 184. Manager responsibility

Coordina:

```text
key construction
policy
store
codec
validation
telemetry
```

No:

```text
SQL
EntityManager lifecycle
UnitOfWork change detection
transactions
```

---

# 185. EntityCacheReader

```php
interface EntityCacheReader
{
    public function get(
        EntityCacheKey $key,
        EntityCacheReadContext $context,
    ): EntityCacheLookupResult;
}
```

---

# 186. EntityCacheWriter

```php
interface EntityCacheWriter
{
    public function put(
        EntityCacheEntry $entry,
        EntityCacheWriteContext $context,
    ): EntityCacheWriteResult;
}
```

---

# 187. Separation

Permite:

```text
read policy
≠
write policy
```

---

# 188. Cache bypass

Queries podrán solicitar:

```php
User::query()
    ->withoutEntityCache()
    ->find($id);
```

---

# 189. Force refresh

```php
User::query()
    ->refreshEntityCache()
    ->find($id);
```

podría significar:

```text
bypass read
→ DB
→ replace cache after safe read
```

---

# 190. No cache API semantic ambiguity

Deben distinguirse:

```text
BYPASS_READ
BYPASS_WRITE
REFRESH
INVALIDATE
NORMAL
```

---

# 191. EntityCacheMode

```php
enum EntityCacheMode
{
    case NORMAL;
    case BYPASS;
    case READ_ONLY;
    case WRITE_ONLY;
    case REFRESH;
}
```

---

# 192. Query cache interaction

Un query puede devolver:

```text
[User#1, User#2, User#3]
```

desde Result Cache.

Luego Entity Cache puede resolver cada EntityKey.

---

# 193. Cached ID list pattern

Esto permite:

```text
Result Cache:
active-users → [1,2,3]

Entity Cache:
1 → User state
2 → User state
3 → User state
```

---

# 194. Advantage

Una actualización de `User#2` puede:

```text
invalidate entity 2
```

sin necesariamente invalidar todos los result caches si membership no cambió.

---

# 195. But membership changes

Si:

```text
User#2.status:
active → inactive
```

entonces:

```text
active-users result cache
```

también puede requerir invalidación.

Eso corresponde al dependency system del documento 191.

---

# 196. Metadata Cache interaction

Entity Cache consume:

```text
EntityMetadata
TypeMetadata
CastMetadata
RelationshipMetadata
```

---

# 197. Metadata Cache never consumes entity state

La dependencia es:

```text
Entity Cache
→ Metadata
```

no:

```text
Metadata Cache
→ Entity Cache runtime values
```

---

# 198. Cache Architecture integration

```text
Database Cache Architecture
          │
          ├── Query Cache
          ├── Result Cache
          ├── Metadata Cache
          └── Entity Cache
                    │
                    ▼
             IdentityMap
                    │
                    ▼
                 Entity
```

---

# 199. Correct lookup architecture

```text
Repository / Model API
          ↓
     EntityManager
          ↓
      IdentityMap
       /       \
     HIT       MISS
     ↓           ↓
 Entity      Entity Cache
               /     \
             HIT     MISS
              ↓        ↓
          Hydration    DB
              ↓        ↓
              └── Hydration
                    ↓
                IdentityMap
                    ↓
                  Entity
```

---

# 200. No direct Model→Redis

Prohibido arquitectónicamente:

```text
Model::find()
→ Redis directly
```

sin pasar por ORM/cache contracts.

---

# 201. Entity lifecycle events

Cache hydration no deberá emitir eventos como si la entidad hubiera sido:

```text
INSERTED
UPDATED
```

---

# 202. postLoad

La política de `postLoad` deberá ser explícita.

Si se ejecuta tanto para DB como cache hydration:

```text
postLoad
```

deberá conocer que representa lifecycle de materialización, no database read.

---

# 203. Lifecycle source

Podrá existir:

```php
enum EntityLoadSource
{
    case DATABASE;
    case ENTITY_CACHE;
}
```

---

# 204. Event handlers

No deberán asumir:

```text
postLoad
→ database query happened
```

---

# 205. Side effects

Lifecycle hooks con side effects son especialmente peligrosos con L2 cache.

Por ello deberán ser controlados/documentados.

---

# 206. Cache population event

Separado:

```text
EntityCachePopulated
```

---

# 207. Cache hit event

```text
EntityCacheHit
```

no será un entity lifecycle mutation event.

---

# 208. Telemetry

Eventos:

```text
EntityCacheLookupStarted
EntityCacheHit
EntityCacheMiss
EntityCacheBypassed
EntityCacheStale
EntityCacheInvalidated
EntityCachePopulated
EntityCacheWriteRejected
EntityCacheVersionConflict
EntityCacheCorruptionDetected
EntityCacheNegativeHit
```

---

# 209. Metrics

```text
db.entity_cache.hit
db.entity_cache.miss
db.entity_cache.stale
db.entity_cache.invalidated
db.entity_cache.populate
db.entity_cache.negative_hit
db.entity_cache.version_conflict
db.entity_cache.lookup.duration
```

---

# 210. Cardinality

No usar:

```text
entity_id=48384838
```

como metric label.

---

# 211. Safe labels

```text
entity_type
region
result
backend
policy
```

solo cuando cardinalidad sea bounded.

---

# 212. Tracing

Span opcional:

```text
database.entity_cache.lookup
```

pero deberá evitar overhead excesivo.

---

# 213. Diagnostics

```php
DB::cache()
    ->entities()
    ->explain(User::class, $id);
```

---

# 214. Diagnostic example

```text
ENTITY CACHE

Entity:
    App\Domain\User

Identifier:
    [REDACTED]

Region:
    users

L1 IdentityMap:
    MISS

L2 Entity Cache:
    HIT

Metadata Generation:
    G42

Coverage:
    COMPLETE

Optimistic Version:
    17

Consistency:
    VERSION_VALIDATED

Source:
    WRITER

Status:
    ACCEPTED
```

---

# 215. Production diagnostics

Entity identifiers y fields sensibles deberán redactarse por default.

---

# 216. Cache corruption

Un payload corrupto deberá:

```text
reject
invalidate
fallback to database
```

cuando policy lo permita.

---

# 217. Never hydrate corrupt payload

No deberá intentarse:

```text
best effort entity reconstruction
```

si faltan invariantes críticas.

---

# 218. Corruption vs stale

Distinguir:

```text
CORRUPTED
```

de:

```text
STALE
```

---

# 219. Incompatible

También:

```text
INCOMPATIBLE
```

para metadata generation/version mismatch.

---

# 220. Error hierarchy

```text
EntityCacheException
├── EntityCacheReadException
├── EntityCacheWriteException
├── EntityCacheDecodeException
├── EntityCacheCorruptionException
├── EntityCacheMetadataMismatchException
├── EntityCacheVersionConflictException
├── EntityCacheDomainMismatchException
├── EntityCacheCoverageException
├── EntityCacheConsistencyException
├── EntityCacheSecurityException
├── EntityCachePopulationException
├── EntityCacheInvalidationException
└── EntityCacheInvariantViolationException
```

---

# 221. Cache availability

Entity Cache deberá ser una optimización.

Por default:

```text
cache unavailable
→ database fallback
```

---

# 222. Cache failure ≠ DB failure

No deberá impedir automáticamente operaciones DB ordinarias.

---

# 223. Exceptions

Una aplicación puede elegir:

```text
REQUIRE_CACHE
```

para casos especializados, pero no será default ORM.

---

# 224. Cache write failure after DB commit

Caso:

```text
DB COMMIT confirmed
↓
cache invalidation fails
```

No puede revertirse el commit.

---

# 225. Result

Debe reportarse:

```text
DB operation successful
cache consistency degraded
```

no:

```text
transaction rolled back
```

---

# 226. Recovery

Podrán usarse:

```text
invalidation outbox
retry queue
generation bump
short TTL
background repair
```

---

# 227. Outbox integration

Para consistencia fuerte distribuida:

```text
DB Transaction
 ├── data mutation
 └── invalidation outbox record
        ↓
      COMMIT
        ↓
 Outbox Processor
        ↓
 Entity Cache Invalidator
```

---

# 228. Advantage

Esto reduce la ventana:

```text
DB commit succeeds
cache invalidation process crashes
```

---

# 229. But not zero latency

Outbox asincrónico introduce una ventana hasta procesar invalidación.

Por tanto, consistency policy deberá reflejarlo.

---

# 230. Synchronous + outbox

Una estrategia robusta:

```text
commit
↓
synchronous invalidate
↓
outbox as recovery
```

---

# 231. Cache consistency

El documento:

```text
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

definirá garantías globales.

---

# 232. Entity cache policy

```php
final readonly class EntityCachePolicy
{
    public function __construct(
        public EntityCacheRegionId $region,
        public EntityCacheConsistencyPolicy $consistency,
        public EntityCacheCoveragePolicy $coverage,
        public EntityCacheSecurityPolicy $security,
        public ?Duration $ttl,
        public bool $negativeCaching,
    ) {}
}
```

---

# 233. Policy compilation

La policy deberá compilarse con entity metadata durante boot.

---

# 234. Per-request policy mutation

No modificar metadata global.

Utilizar:

```text
EntityCacheReadContext
```

para overrides permitidos.

---

# 235. Read context

```php
final readonly class EntityCacheReadContext
{
    public function __construct(
        public PersistenceDomain $domain,
        public EntityCacheMode $mode,
        public ?TransactionContextReference $transaction,
        public CacheConsistencyRequirement $consistency,
    ) {}
}
```

---

# 236. Write context

```php
final readonly class EntityCacheWriteContext
{
    public function __construct(
        public PersistenceDomain $domain,
        public CacheMutationReason $reason,
        public TransactionOutcomeEvidence $transactionEvidence,
    ) {}
}
```

---

# 237. No TransactionContext object in shared cache

La referencia contextual es runtime-only.

Nunca serializada dentro del entry.

---

# 238. Cache mutation reasons

```php
enum CacheMutationReason
{
    case READ_POPULATION;
    case INSERT_COMMITTED;
    case UPDATE_COMMITTED;
    case DELETE_COMMITTED;
    case EXPLICIT_REFRESH;
    case INVALIDATION_REPAIR;
}
```

---

# 239. Resource governance

El Entity Cache deberá soportar:

```text
maximum entry size
maximum batch size
maximum serialization depth
timeout
memory budget
backend request budget
```

---

# 240. Huge entities

Una entidad con:

```text
10 MB JSON field
```

puede no ser apropiada para L2 cache.

---

# 241. Entry size policy

```text
entry > maxEntryBytes
→ BYPASS CACHE
```

---

# 242. Cacheability reason

Diagnostics deberán explicar:

```text
NON_CACHEABLE_TOO_LARGE
```

en vez de fallar silenciosamente.

---

# 243. Compression

Puede permitirse:

```text
payload compression
```

para entries grandes.

---

# 244. Compression tradeoff

```text
less network/memory
vs
more CPU
```

---

# 245. Encryption tradeoff

```text
better confidentiality
vs
CPU/key management
```

---

# 246. Serialization budgets

Decode deberá limitar:

```text
payload bytes
nested depth
collection size
string size
```

---

# 247. Cache poisoning

Payloads provenientes de distributed stores deberán validarse.

---

# 248. No dynamic class loading

Un cache payload no podrá indicar:

```text
"class": "Some\Arbitrary\Class"
```

y provocar autoload arbitrario.

---

# 249. Entity type resolution

Se realizará mediante:

```text
trusted EntityMetadataRegistry
```

---

# 250. Cache key privacy

Si IDs contienen información sensible:

```text
email
external account number
```

podrá utilizarse:

```text
hashed canonical key
```

---

# 251. Key hashing

```text
Raw Canonical Entity Identity
↓
namespaced cryptographic hash
↓
backend key
```

---

# 252. Hash collision

La implementación deberá utilizar algoritmos con riesgo de colisión aceptablemente bajo y conservar validación de identity dentro del payload cuando corresponda.

---

# 253. Cache backend

Posibles:

```text
InMemory
Redis
Memcached-like backend
Distributed Cache Plugin
Null
Tiered
```

---

# 254. In-memory L2

En FrankenPHP:

```text
worker-local cache
```

puede sobrevivir requests.

Pero no está sincronizado automáticamente con otros workers.

---

# 255. Worker-local consistency

Si existen múltiples workers:

```text
Worker A L2
Worker B L2
Worker C L2
```

la invalidación deberá alcanzar todos o usar TTL/generation/version policy.

---

# 256. Distributed backend

Redis simplifica sharing:

```text
Workers
   ↓
Shared Entity Cache
```

pero no elimina races de consistencia.

---

# 257. Distributed cache ≠ distributed transaction

Regla:

```text
Redis Entity Cache
+
SQL Database
≠
Atomic Distributed Transaction
```

---

# 258. No 2PC assumption

Core no asumirá two-phase commit entre DB y cache.

---

# 259. Correctness mechanism

Se basará en:

```text
commit ordering
invalidation
versions
generations
outbox/recovery
consistency policies
```

---

# 260. FrankenPHP

Arquitectura:

```text
FrankenPHP Worker
│
├── Frozen Metadata
├── Entity Cache Client
│
└── Request Scope
    ├── EntityManager
    ├── IdentityMap
    ├── UnitOfWork
    └── TransactionContext
```

---

# 261. No cross-request managed objects

Al terminar Request A:

```text
User object
```

no deberá permanecer administrado para Request B.

---

# 262. Cache entry may survive

Lo que sí puede sobrevivir:

```text
EntityCacheEntry(User#42)
```

---

# 263. RoadRunner

Misma regla.

---

# 264. OpenSwoole

Concurrent coroutines deberán tener:

```text
separate IdentityMaps
```

aunque compartan:

```text
EntityCache client/store
```

---

# 265. Thread/coroutine safety

In-memory Entity Cache deberá ser explícitamente concurrency-safe si se comparte.

---

# 266. No static arrays as distributed cache

Un:

```php
private static array $entities = [];
```

no será implementación oficial de L2 para runtimes concurrentes sin lifecycle, bounds e invalidation correctos.

---

# 267. Development mode

Podrá deshabilitarse Entity Cache por default para facilitar:

```text
debugging
mapping changes
data inspection
```

---

# 268. Production opt-in

A diferencia de Metadata Cache, Entity Cache debería ser:

```text
explicitly enabled
```

por entidad/región.

---

# 269. Reason

No todas las entidades se benefician.

---

# 270. Good candidates

```text
reference data
catalog entries
configuration entities
rarely changing entities
high-read entities
```

---

# 271. Poor candidates

```text
highly volatile counters
balances requiring strongest freshness
temporary workflow rows
very large entities
write-heavy entities
```

sin estrategias especializadas.

---

# 272. CLI

Conceptualmente:

```bash
php voltstack database:cache:entity:status
php voltstack database:cache:entity:clear
php voltstack database:cache:entity:invalidate User 42
php voltstack database:cache:entity:warm Product
```

---

# 273. Clear scope

`clear` deberá requerir región/entity domain cuando sea posible.

Evitar:

```text
global cache flush
```

---

# 274. Warmup

Entity warmup será distinto de Metadata warmup.

Porque requiere:

```text
actual application data
```

---

# 275. Warmup safety

No deberá cargar millones de entities accidentalmente.

Requerirá:

```text
explicit query
limits
batching
resource budgets
```

---

# 276. Testing strategy

Se requerirán pruebas de:

```text
identity
hydration
versions
transactions
rollback
unknown outcomes
replicas
shards
tenants
relationships
metadata compatibility
serialization
corruption
concurrency
persistent workers
security
performance
```

---

# 277. IdentityMap precedence test

```text
L1 HIT
+
L2 different state
→ same L1 object returned
```

---

# 278. Cache hydration identity test

```text
L1 MISS
L2 HIT
→ hydrate
→ register L1
→ subsequent lookup same object
```

---

# 279. Dirty entity test

L2 cache no sobrescribirá dirty managed entity.

---

# 280. Metadata mismatch test

G1 entry + G2 metadata → reject.

---

# 281. NULL/MISSING test

Debe preservarse diferencia.

---

# 282. Partial coverage test

Partial entry nunca será tratado como complete.

---

# 283. Transaction commit test

Cache mutation no será visible antes de confirmed commit.

---

# 284. Rollback test

Pending cache mutations se descartarán.

---

# 285. Unknown commit test

No se publicará new value como confirmado.

---

# 286. Flush test

`flush()` sin commit no publicará shared entity cache mutation.

---

# 287. Optimistic version test

Older entry no sobrescribirá newer version.

---

# 288. Stale repopulation test

Read anterior a invalidation no resucitará entry stale.

---

# 289. Replica lag test

Stale replica no deberá repoblar un versioned entry con versión anterior.

---

# 290. Tenant collision test

```text
Tenant A/User#1
Tenant B/User#1
```

serán independientes.

---

# 291. Shard collision test

Mismo ID en shards distintos no colisionará.

---

# 292. Negative cache insert test

Insert confirmado invalidará negative entry.

---

# 293. Delete test

Delete confirmado eliminará positive entry y podrá establecer negative entry según policy.

---

# 294. Relationship test

To-many graph no se serializará accidentalmente.

---

# 295. Polymorphic test

Stable alias se preservará sin FQCN persisted discriminator.

---

# 296. Security test

Payload no podrá causar arbitrary class loading.

---

# 297. Worker isolation test

Entity instance de Request A no aparecerá administrada en Request B.

---

# 298. Worker cache reuse test

L2 entry sí podrá reutilizarse.

---

# 299. Concurrent population test

Dos workers no deberán degradar correctness.

---

# 300. Cache outage test

ORM deberá caer a DB cuando policy lo permita.

---

# 301. Cache write failure test

DB commit confirmado no será reportado como rollback por failure de cache.

---

# 302. Performance benchmark

Medir:

```text
L1 lookup
L2 local lookup
L2 distributed lookup
L2 batch lookup
DB lookup
cache decode
cache hydration
cache serialization
```

---

# 303. Directory structure

```text
src/Quantum/Database/Cache/Entity/
│
├── EntityCacheManager.php
├── EntityCacheReader.php
├── EntityCacheWriter.php
├── EntityCacheKey.php
├── EntityCacheEntry.php
├── EntityCachePolicy.php
├── EntityCacheMode.php
├── EntityCacheRegion.php
├── EntityCacheLookupResult.php
├── EntityCacheLookupStatus.php
│
├── Identity/
│   ├── EntityCacheKeyFactory.php
│   ├── EntityCacheIdentity.php
│   ├── EntityCacheNamespace.php
│   └── EntityCacheKeyHasher.php
│
├── Entry/
│   ├── EntityFieldState.php
│   ├── EntityCacheVersion.php
│   ├── NegativeEntityCacheEntry.php
│   ├── EntityCachePopulationSource.php
│   └── EntityCacheEntryValidator.php
│
├── Coverage/
│   ├── EntityCacheCoveragePolicy.php
│   ├── EntityCacheCoverageValidator.php
│   └── CachedLoadedFieldMask.php
│
├── Hydration/
│   ├── EntityCacheHydrator.php
│   ├── EntityCacheHydrationContext.php
│   ├── EntityCacheHydrationResult.php
│   └── EntityCacheIdentityMapIntegrator.php
│
├── Version/
│   ├── EntityCacheVersionComparator.php
│   ├── EntityInvalidationGeneration.php
│   ├── EntityCacheVersionPolicy.php
│   └── EntityCacheVersionConflict.php
│
├── Transaction/
│   ├── DeferredEntityCacheMutation.php
│   ├── DeferredEntityCacheMutationSet.php
│   ├── TransactionEntityCacheCoordinator.php
│   └── EntityCacheCommitPublisher.php
│
├── Consistency/
│   ├── EntityCacheConsistencyPolicy.php
│   ├── EntityCacheConsistencyRequirement.php
│   ├── EntityCacheReadValidator.php
│   └── EntityCachePopulationValidator.php
│
├── Region/
│   ├── EntityCacheRegionId.php
│   ├── EntityCacheRegionRegistry.php
│   └── EntityCacheRegionPolicy.php
│
├── Store/
│   ├── EntityCacheStore.php
│   ├── BatchEntityCacheStore.php
│   ├── InMemoryEntityCacheStore.php
│   ├── RedisEntityCacheStore.php
│   ├── TieredEntityCacheStore.php
│   └── NullEntityCacheStore.php
│
├── Codec/
│   ├── EntityCacheCodec.php
│   ├── EntityCacheEncoder.php
│   ├── EntityCacheDecoder.php
│   └── EntityCachePayload.php
│
├── Context/
│   ├── EntityCacheReadContext.php
│   ├── EntityCacheWriteContext.php
│   └── CacheMutationReason.php
│
├── Population/
│   ├── EntityCachePopulator.php
│   ├── EntityCacheSingleFlight.php
│   ├── EntityCachePopulationPolicy.php
│   └── EntityCacheWarmup.php
│
├── Security/
│   ├── EntityCacheSecurityPolicy.php
│   ├── EntityCachePayloadProtector.php
│   └── EntityCacheSensitiveFieldPolicy.php
│
├── Diagnostics/
│   ├── EntityCacheInspector.php
│   ├── EntityCacheExplainer.php
│   └── EntityCacheDiagnosticReport.php
│
├── Telemetry/
│   ├── EntityCacheTelemetry.php
│   ├── EntityCacheHitEvent.php
│   ├── EntityCacheMissEvent.php
│   ├── EntityCacheInvalidatedEvent.php
│   └── EntityCacheVersionConflictEvent.php
│
└── Exception/
    ├── EntityCacheException.php
    ├── EntityCacheReadException.php
    ├── EntityCacheWriteException.php
    ├── EntityCacheCorruptionException.php
    ├── EntityCacheMetadataMismatchException.php
    ├── EntityCacheVersionConflictException.php
    ├── EntityCacheDomainMismatchException.php
    ├── EntityCacheConsistencyException.php
    └── EntityCacheInvariantViolationException.php
```

---

# 304. Architectural invariants

## DB-ECACHE-001
Entity Cache será un cache de segundo nivel.

## DB-ECACHE-002
IdentityMap seguirá siendo el primer nivel.

## DB-ECACHE-003
Entity Cache no sustituirá IdentityMap.

## DB-ECACHE-004
Entity Cache no almacenará managed entity instances.

## DB-ECACHE-005
Entity Cache almacenará persistent entity representations.

## DB-ECACHE-006
EntityCacheEntry no será Entity.

## DB-ECACHE-007
Entity Cache no almacenará EntityManager.

## DB-ECACHE-008
Entity Cache no almacenará UnitOfWork.

## DB-ECACHE-009
Entity Cache no almacenará TransactionContext.

## DB-ECACHE-010
Entity Cache no almacenará lazy loader runtime state.

## DB-ECACHE-011
Entity Cache no almacenará pending changes.

## DB-ECACHE-012
EntityCacheKey reutilizará canonical EntityKey semantics.

## DB-ECACHE-013
Canonical identifiers serán deterministas.

## DB-ECACHE-014
Composite identifiers no usarán encoding ambiguo.

## DB-ECACHE-015
Tenant domains estarán aislados.

## DB-ECACHE-016
Shard domains estarán aislados.

## DB-ECACHE-017
Database domains estarán aislados cuando corresponda.

## DB-ECACHE-018
Entity cache keys serán application namespaced.

## DB-ECACHE-019
Cache payloads serán metadata-generation aware.

## DB-ECACHE-020
Incompatible metadata entries serán rechazados.

## DB-ECACHE-021
NULL será distinto de MISSING.

## DB-ECACHE-022
Partial será distinto de complete.

## DB-ECACHE-023
Partial entry nunca fingirá completitud.

## DB-ECACHE-024
FULL_ENTITY_ONLY será el default recomendado.

## DB-ECACHE-025
Cache hydration respetará IdentityMap.

## DB-ECACHE-026
Cache hydration no creará segunda instancia canonical.

## DB-ECACHE-027
Dirty managed entities no serán sobrescritas por cache.

## DB-ECACHE-028
Managed state tendrá precedencia sobre stale L2 state.

## DB-ECACHE-029
Explicit refresh tendrá semántica diferenciada.

## DB-ECACHE-030
Cache baseline será ORM-known state, no prueba absoluta de DB freshness.

## DB-ECACHE-031
Optimistic version será preservada cuando exista.

## DB-ECACHE-032
EntityCacheVersion será distinta de OptimisticVersion.

## DB-ECACHE-033
Older cache version no sobrescribirá newer version cuando sea comparable.

## DB-ECACHE-034
Cache mutations no serán publicadas antes de confirmed commit.

## DB-ECACHE-035
Flush no será commit.

## DB-ECACHE-036
Flush no publicará shared cache mutation por sí mismo.

## DB-ECACHE-037
Rollback descartará deferred cache mutations.

## DB-ECACHE-038
UNKNOWN commit no será tratado como committed.

## DB-ECACHE-039
UNKNOWN commit favorecerá invalidation.

## DB-ECACHE-040
Cache invalidation failure no cambiará retroactivamente DB commit outcome.

## DB-ECACHE-041
Read-your-own-writes no dependerá de shared cache.

## DB-ECACHE-042
IdentityMap tendrá precedencia dentro de transaction.

## DB-ECACHE-043
Read population será transaction-aware.

## DB-ECACHE-044
Historical snapshots no usarán current entity cache key.

## DB-ECACHE-045
Locking reads no poblarán shared cache automáticamente.

## DB-ECACHE-046
Authorization no será sustituida por cache.

## DB-ECACHE-047
Cache hit no implicará authorization.

## DB-ECACHE-048
Tenant isolation ocurrirá antes del cache lookup.

## DB-ECACHE-049
Sensitive fields podrán ser non-cacheable.

## DB-ECACHE-050
Excluded required fields afectarán coverage.

## DB-ECACHE-051
Entity relationships no serán recursivamente serializadas.

## DB-ECACHE-052
To-many collections no serán embebidas arbitrariamente.

## DB-ECACHE-053
To-one references podrán representarse mediante canonical EntityReference.

## DB-ECACHE-054
Polymorphic references usarán stable morph aliases.

## DB-ECACHE-055
Persisted morph type no será FQCN.

## DB-ECACHE-056
Value Objects usarán canonical persistent representation.

## DB-ECACHE-057
Enums usarán canonical persisted tokens.

## DB-ECACHE-058
Temporal values no dependerán de worker timezone implícita.

## DB-ECACHE-059
Distributed payload codec será versionado.

## DB-ECACHE-060
Unsafe unserialize no será obligatorio/default.

## DB-ECACHE-061
Cache payload no podrá provocar arbitrary class loading.

## DB-ECACHE-062
Entity types serán resueltos mediante trusted metadata.

## DB-ECACHE-063
Batch cache lookup será soportable.

## DB-ECACHE-064
100 cache lookups no deberán requerir necesariamente 100 network round trips.

## DB-ECACHE-065
Cache stampede protection será bounded.

## DB-ECACHE-066
Cache locks no serán DB transaction locks.

## DB-ECACHE-067
TTL no será tratado como consistency guarantee.

## DB-ECACHE-068
Stale-while-revalidate no será default ORM consistency.

## DB-ECACHE-069
Entity cache consistency será policy-driven.

## DB-ECACHE-070
Invalidation será preferida a speculative replacement cuando new state sea incierto.

## DB-ECACHE-071
Generated DB values desconocidos favorecerán invalidation.

## DB-ECACHE-072
Delete confirmado invalidará positive entry.

## DB-ECACHE-073
Negative cache será explícito.

## DB-ECACHE-074
Negative cache será bounded.

## DB-ECACHE-075
Insert confirmado invalidará negative cache.

## DB-ECACHE-076
Negative cache será transaction-aware.

## DB-ECACHE-077
Stale repopulation deberá ser prevenible.

## DB-ECACHE-078
Invalidation generations podrán proteger contra resurrection.

## DB-ECACHE-079
CAS podrá proteger versioned writes.

## DB-ECACHE-080
Unordered version tokens no serán comparados arbitrariamente.

## DB-ECACHE-081
Replica lag será considerado para cache population.

## DB-ECACHE-082
Stale replica no deberá resucitar older version cuando exista evidencia de versión nueva.

## DB-ECACHE-083
Sticky writer semantics podrán influir cache population.

## DB-ECACHE-084
Failover podrá invalidar cache domain si consistency no puede demostrarse.

## DB-ECACHE-085
Failover no implicará automáticamente FLUSHALL.

## DB-ECACHE-086
Shard movement invalidará old cache domain.

## DB-ECACHE-087
Partition routing será distinto de entity caching.

## DB-ECACHE-088
Entity Cache no decidirá arbitrariamente shard ownership.

## DB-ECACHE-089
Cache regions serán explícitas.

## DB-ECACHE-090
Cache region no será equivalente a table.

## DB-ECACHE-091
Entity cacheability será metadata.

## DB-ECACHE-092
Runtime cache state no será static Model state.

## DB-ECACHE-093
EntityCacheManager no generará SQL.

## DB-ECACHE-094
EntityCacheManager no ejecutará DB queries directamente.

## DB-ECACHE-095
EntityCacheManager no administrará UnitOfWork.

## DB-ECACHE-096
Read policy podrá diferir de write policy.

## DB-ECACHE-097
Cache bypass será explícito.

## DB-ECACHE-098
Refresh será distinto de bypass.

## DB-ECACHE-099
Result Cache podrá almacenar entity ID lists.

## DB-ECACHE-100
Entity Cache podrá resolver states para cached ID lists.

## DB-ECACHE-101
Membership invalidation será distinta de entity state invalidation.

## DB-ECACHE-102
Entity Cache consumirá Metadata Cache, no al revés.

## DB-ECACHE-103
Cache hydration no emitirá INSERT/UPDATE lifecycle events.

## DB-ECACHE-104
Entity load source podrá ser observable.

## DB-ECACHE-105
Cache telemetry será distinta de entity lifecycle mutation events.

## DB-ECACHE-106
Corrupted payload será rechazado.

## DB-ECACHE-107
Corrupted payload no será hydrated best-effort.

## DB-ECACHE-108
STALE será distinto de CORRUPTED.

## DB-ECACHE-109
INCOMPATIBLE será distinto de STALE.

## DB-ECACHE-110
Cache availability será opcional por default.

## DB-ECACHE-111
Cache outage podrá degradar a database access.

## DB-ECACHE-112
Cache write failure post-commit no implicará rollback.

## DB-ECACHE-113
Recovery podrá usar outbox.

## DB-ECACHE-114
Outbox no será confundido con zero-latency consistency.

## DB-ECACHE-115
EntityCachePolicy será compilable.

## DB-ECACHE-116
Runtime overrides no mutarán global metadata.

## DB-ECACHE-117
Cache resource usage será bounded.

## DB-ECACHE-118
Oversized entities podrán bypass cache.

## DB-ECACHE-119
Compression será opcional.

## DB-ECACHE-120
Encryption será policy-driven.

## DB-ECACHE-121
Payload decode tendrá resource limits.

## DB-ECACHE-122
Cache key privacy podrá usar hashing.

## DB-ECACHE-123
Cache key hashing preservará domain separation.

## DB-ECACHE-124
Worker-local L2 no será asumido globalmente coherent.

## DB-ECACHE-125
Distributed cache no será distributed transaction.

## DB-ECACHE-126
Core no asumirá DB/cache 2PC.

## DB-ECACHE-127
Correctness dependerá de ordering/invalidation/versioning/policy.

## DB-ECACHE-128
FrankenPHP no compartirá managed entities entre requests.

## DB-ECACHE-129
FrankenPHP sí podrá reutilizar immutable cache entries/client infrastructure.

## DB-ECACHE-130
RoadRunner seguirá las mismas reglas.

## DB-ECACHE-131
OpenSwoole mantendrá IdentityMaps separados por execution scope.

## DB-ECACHE-132
Concurrent in-memory L2 será concurrency-safe.

## DB-ECACHE-133
Un static array sin lifecycle no será implementación suficiente.

## DB-ECACHE-134
Entity Cache será opt-in por default.

## DB-ECACHE-135
No todas las entidades serán cacheables.

## DB-ECACHE-136
High-read/low-write entities serán candidatas naturales.

## DB-ECACHE-137
Write-heavy entities requerirán análisis específico.

## DB-ECACHE-138
Entity warmup será distinto de metadata warmup.

## DB-ECACHE-139
Entity warmup estará resource-bounded.

## DB-ECACHE-140
Diagnostics redactarán identifiers sensibles.

## DB-ECACHE-141
Metrics evitarán entity IDs como labels.

## DB-ECACHE-142
Cache hit ratio distinguirá L1 y L2.

## DB-ECACHE-143
Entity Cache nunca será considerado database truth absoluta.

## DB-ECACHE-144
Entity Cache nunca será un segundo EntityManager.

## DB-ECACHE-145
Entity Cache nunca será un segundo UnitOfWork.

## DB-ECACHE-146
Entity Cache nunca decidirá transaction outcome.

## DB-ECACHE-147
Entity Cache nunca decidirá authorization.

## DB-ECACHE-148
Entity Cache nunca generará SQL.

## DB-ECACHE-149
Entity Cache nunca hará commit o rollback.

## DB-ECACHE-150
Entity Cache preservará las invariantes ORM independientemente del backend utilizado.

---

# 305. Modelo final

```text
                    ENTITY LOOKUP

Repository / Model API
          │
          ▼
     EntityManager
          │
          ▼
      IdentityMap
       /       \
    HIT         MISS
     │            │
     ▼            ▼
 Managed       Entity Cache
 Entity         /       \
              HIT       MISS
               │          │
               ▼          ▼
          Cache Entry   Database
               │          │
               ▼          ▼
           Validation   Result
               │          │
               └────┬─────┘
                    ▼
                Hydration
                    │
                    ▼
                IdentityMap
                    │
                    ▼
              Managed Entity
```

---

# 306. Write model

```text
Entity modified
      ↓
UnitOfWork
      ↓
Persistence Planner
      ↓
Database UPDATE
      ↓
Transaction ACTIVE
      ↓
Deferred Cache Mutation
      ↓
COMMIT
   /        \
CONFIRMED   UNKNOWN
   │           │
   ▼           ▼
Invalidate/  Invalidate
Replace      conservatively
```

Mientras:

```text
ROLLBACK
   ↓
Discard deferred mutation
```

---

# 307. Cache hierarchy

```text
L1
IdentityMap
│
│ scope-local
│ actual objects
│ strongest object identity semantics
│
▼

L2
Entity Cache
│
│ cross-scope
│ persistent representations
│ versioned / invalidatable
│
▼

Database
persistent authority
```

---

# 308. Consistency model

Entity Cache correctness depende de:

```text
Correct Entity Identity
        +
Metadata Compatibility
        +
Coverage Correctness
        +
Transaction Ordering
        +
Version Awareness
        +
Invalidation
        +
Replica Awareness
        +
Tenant/Shard Isolation
        +
Safe Population
```

Por tanto:

```text
CacheHit
≠
SafeEntityState
```

Sino:

```text
SafeEntityCacheHit
=
CacheHit
∧ IdentityValid
∧ MetadataCompatible
∧ CoverageSufficient
∧ PersistenceDomainMatches
∧ VersionAcceptable
∧ ConsistencyPolicySatisfied
```

---

# 309. Regla maestra final

> **El Entity Cache de VoltStack será una caché de segundo nivel que reutiliza representaciones persistentes de entidades entre scopes sin compartir objetos administrados ni estado mutable del ORM.**

La jerarquía será:

```text
IdentityMap
→ object identity within current scope

Entity Cache
→ reusable persistent entity representation

Database
→ persistent storage authority
```

La regla crítica será:

```text
EntityCacheEntry
≠
ManagedEntity
```

y:

```text
SharedCacheMutation
only after
ConfirmedTransactionOutcome
```

La integración con transacciones deberá garantizar:

```text
flush
≠
commit
```

y:

```text
UNKNOWN commit
≠
safe cache publication
```

La integración con el ORM garantizará:

```text
same EntityKey
→ same managed object
```

independientemente de que la entidad haya sido obtenida desde:

```text
Database
Entity Cache
Relationship Loader
Query Hydration
```

Finalmente:

```text
Entity Cache
```

será siempre una optimización de acceso y reutilización de estado persistente.

Nunca se convertirá en:

```text
second EntityManager
second UnitOfWork
database authority
authorization system
transaction manager
```

---

# 310. Siguiente documento

```text
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
```

El siguiente documento definirá el mecanismo transversal encargado de determinar:

```text
what changed
        ↓
what cache artifacts depend on it
        ↓
what must be invalidated
        ↓
when invalidation becomes visible
        ↓
how invalidation propagates
```

incluyendo:

```text
Dependency Graph
Invalidation Keys
Entity Invalidation
Query Invalidation
Result Invalidation
Metadata Invalidation
Relationship Membership
Transaction-Aware Invalidation
Commit Ordering
Unknown Commit Outcomes
Invalidation Generations
Tags
Version Tokens
Cache Regions
Cross-Worker Propagation
Distributed Invalidation
Outbox
Retry
Idempotency
Replica Lag
Sharding
Multitenancy
Failure Recovery
Race Prevention
Telemetry
Persistent Runtimes
```

bajo la regla:

```text
Cache invalidation
≠
cache clear
```

y especialmente:

```text
Database mutation
→ semantic change set
→ dependency resolution
→ targeted invalidation
```

en lugar de recurrir sistemáticamente a:

```text
FLUSH ALL CACHES
```