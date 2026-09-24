# 314_DATABASE_CACHE_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Cache Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 314 — Database Cache Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `315_DATABASE_EVENT_SYSTEM_INTEGRATION.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual:

```text
VoltStack/Quantum/Database
```

se integra con el sistema general de caché de VoltStack:

```text
VoltStack/Quantum/Cache
```

sin convertir Cache en una dependencia obligatoria del núcleo Database y sin permitir que los datos almacenados en caché sean tratados como autoridad sobre el estado real de la base de datos.

La regla central será:

> **La caché de Database será una capa opcional de aceleración y reutilización de información; nunca será la fuente autoritativa de verdad de la base de datos ni podrá alterar las garantías transaccionales del sistema.**

Formalmente:

```text
Database Truth
      ↓
Cacheable Representation
      ↓
Cache Policy
      ↓
Cache Provider
      ↓
Cached Evidence
```

y nunca:

```text
Cache
  ↓
defines
  ↓
Database Truth
```

---

# 2. Principios fundamentales

VoltStack deberá preservar:

```text
Cache
≠
Database
≠
IdentityMap
≠
Transaction
≠
UnitOfWork
≠
Result Cursor
```

También:

```text
Query Cache
≠
Compiled Query Cache
≠
Result Cache
≠
Metadata Cache
≠
Entity Cache
≠
Hydration Cache
≠
Application Cache
```

Y especialmente:

```text
Physical Cache Hit
≠
Usable Cache Hit
```

porque un valor físicamente presente puede no satisfacer:

```text
consistency
tenant
shard
transaction
generation
schema
metadata
security
freshness
```

del contexto actual.

---

# 3. Relación con la arquitectura previa

Los documentos:

```text
186_DATABASE_CACHE_ARCHITECTURE.md
187_DATABASE_QUERY_CACHE_SYSTEM.md
188_DATABASE_RESULT_CACHE_SYSTEM.md
189_DATABASE_METADATA_CACHE_SYSTEM.md
190_DATABASE_ENTITY_CACHE_SYSTEM.md
191_DATABASE_CACHE_INVALIDATION_SYSTEM.md
192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md
```

definen la semántica interna de caché de Database.

Este documento define específicamente:

```text
cómo Database
        ↕
se integra con
        ↕
VoltStack Cache
```

Por tanto:

```text
186–192
=
Database caching semantics

314
=
Framework Cache integration boundary
```

---

# 4. Objetivos

La integración deberá permitir:

1. usar el paquete Cache de VoltStack cuando esté instalado;
2. operar Database sin Cache;
3. soportar múltiples proveedores;
4. aislar categorías de caché;
5. mantener claves deterministas;
6. soportar namespaces;
7. incorporar generaciones;
8. preservar aislamiento tenant/shard;
9. invalidar semánticamente;
10. integrar transacciones;
11. diferir publicación hasta `afterCommit`;
12. manejar rollback;
13. manejar resultados transaccionales `UNKNOWN`;
14. soportar caché local;
15. soportar caché distribuida;
16. soportar niveles L0/L1/L2;
17. proteger contra stampede;
18. evitar serialización insegura;
19. proteger información sensible;
20. soportar persistent workers;
21. proporcionar telemetría;
22. degradarse ante fallos del proveedor;
23. soportar testing;
24. permitir extensiones;
25. mantener Cache como dependencia opcional.

---

# 5. Arquitectura general

```text
                    VoltStack Database
                           │
                           ▼
                 Database Cache Layer
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
 Metadata Cache      Query/Plan Cache      Result Cache
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                           ▼
                DatabaseCacheGateway
                           │
                           ▼
                DatabaseCacheAdapter
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼
       NullCacheAdapter          VoltStackCacheAdapter
                                         │
                                         ▼
                               VoltStack/Quantum/Cache
                                         │
                      ┌──────────────────┼─────────────────┐
                      ▼                  ▼                 ▼
                    Memory             Redis             Other
```

---

# 6. Dependencia opcional

La dirección permitida será:

```text
Database
   ↓
Database Cache Contract
```

y el adapter externo:

```text
Database Cache Contract
        ↑
VoltStack Cache Adapter
        ↓
VoltStack Cache
```

Database Core no deberá requerir:

```php
use VoltStack\Quantum\Cache\RedisCache;
```

---

# 7. Null Object

Si Cache no está instalado:

```text
Database
   ↓
NullDatabaseCacheProvider
```

permitirá que Database continúe funcionando.

---

# 8. Cache disabled ≠ Database disabled

Desactivar Cache únicamente elimina una optimización.

No deberá alterar:

```text
query correctness
transaction correctness
ORM correctness
schema correctness
security correctness
```

---

# 9. Integration contract

Se propone conceptualmente:

```php
interface DatabaseCacheProvider
{
    public function get(
        DatabaseCacheKey $key,
    ): DatabaseCacheLookup;

    public function put(
        DatabaseCacheKey $key,
        CacheableDatabaseValue $value,
        DatabaseCachePolicy $policy,
    ): void;

    public function remove(
        DatabaseCacheKey $key,
    ): void;

    public function invalidate(
        DatabaseCacheInvalidation $invalidation,
    ): void;
}
```

---

# 10. No PSR-style leakage

Aunque VoltStack Cache pueda proporcionar APIs genéricas, Database no deberá diseñar su semántica alrededor de:

```php
get(string $key);
set(string $key, mixed $value);
```

como único contrato.

Database necesita conceptos adicionales:

```text
cache category
generation
consistency
identity
invalidation
scope
tenant
shard
transaction awareness
```

---

# 11. DatabaseCacheGateway

El punto principal de coordinación será:

```text
DatabaseCacheGateway
```

responsable de:

```text
policy
key creation
provider selection
consistency validation
serialization
invalidation
telemetry
```

sin convertirse en propietario de la semántica de cada caché especializada.

---

# 12. Specialized cache services

Se mantendrán servicios especializados:

```text
DatabaseMetadataCache
CompiledQueryCache
DatabaseResultCache
DatabaseEntityCache
HydrationPlanCache
```

Cada uno conoce su propio tipo de dato.

---

# 13. Gateway ≠ specialized cache

Ejemplo:

```text
ResultCache
   ↓
DatabaseCacheGateway
   ↓
Provider
```

El Gateway no deberá conocer cómo interpretar un:

```text
ORM Entity
```

---

# 14. Categorías

Se propone:

```php
enum DatabaseCacheCategory: string
{
    case METADATA = 'metadata';
    case COMPILED_QUERY = 'compiled_query';
    case QUERY_PLAN = 'query_plan';
    case RESULT = 'result';
    case ENTITY = 'entity';
    case HYDRATION_PLAN = 'hydration_plan';
    case SCHEMA_METADATA = 'schema_metadata';
    case CAPABILITY = 'capability';
}
```

Las categorías definitivas podrán evolucionar.

---

# 15. Categoría ≠ provider

Ejemplo:

```text
metadata
→ local memory

results
→ Redis
```

deberá ser válido.

---

# 16. Provider routing

Configuración:

```php
'cache' => [
    'metadata' => [
        'provider' => 'local',
    ],

    'compiled_queries' => [
        'provider' => 'local',
    ],

    'results' => [
        'provider' => 'redis',
    ],
],
```

---

# 17. Cache layers

VoltStack podrá soportar conceptualmente:

```text
L0
=
operation/request local

L1
=
worker/process local

L2
=
shared/distributed
```

---

# 18. L0 cache

Lifetime:

```text
Database Operation Scope
```

Ventajas:

```text
fast
isolated
no serialization required in some cases
```

---

# 19. L1 cache

Lifetime:

```text
worker/process
```

Útil para:

```text
compiled metadata
compiled queries
hydration plans
```

pero requiere control de generaciones.

---

# 20. L2 cache

Lifetime:

```text
cross-process / distributed
```

Ejemplo:

```text
Redis
```

Puede utilizarse para:

```text
result cache
entity cache
metadata artifacts
```

cuando la política lo permita.

---

# 21. L0 ≠ IdentityMap

Aunque ambos sean scope-local:

```text
L0 Cache
≠
IdentityMap
```

IdentityMap garantiza identidad de objetos ORM.

Cache no.

---

# 22. L1 ≠ static mutable global

Una caché local de worker deberá estar encapsulada y gobernada.

No deberá implementarse simplemente mediante:

```php
static array $cache = [];
```

disperso por Database.

---

# 23. Multi-level lookup

Podrá existir:

```text
Request
   ↓
L0
   ↓ miss
L1
   ↓ miss
L2
   ↓ miss
Database
```

---

# 24. Promotion

Un valor válido encontrado en L2 podrá promoverse a:

```text
L1
L0
```

si la policy lo permite.

---

# 25. Promotion ≠ automatic truth

Antes de promoción:

```text
Validate cache envelope
Validate generation
Validate context
Validate consistency
```

---

# 26. Cache Envelope

Los valores persistidos deberán envolverse conceptualmente en:

```text
DatabaseCacheEnvelope
├── Category
├── Key
├── Payload
├── CreatedAt
├── ExpiresAt?
├── Generation
├── SchemaGeneration?
├── MetadataGeneration?
├── TenantId?
├── ShardId?
├── ConsistencyMetadata
├── SerializationVersion
└── IntegrityMetadata?
```

---

# 27. Payload ≠ envelope

La metadata de validez no deberá confundirse con el dato almacenado.

---

# 28. Cache key architecture

Toda clave deberá derivarse de identidad semántica.

No de strings accidentales.

---

# 29. DatabaseCacheKey

Conceptualmente:

```php
final readonly class DatabaseCacheKey
{
    public function __construct(
        public DatabaseCacheCategory $category,
        public CacheNamespace $namespace,
        public CacheIdentity $identity,
        public CacheContextFingerprint $context,
    ) {}
}
```

---

# 30. Key composition

Dependiendo de categoría:

```text
CacheKey
=
Category
+
LogicalDatabase
+
SemanticIdentity
+
RelevantContext
+
RelevantGenerations
```

---

# 31. Relevant context

Puede incluir:

```text
tenant
shard
platform
capabilities
schema generation
metadata generation
query fingerprint
parameters
security scope
locale
timezone
```

únicamente cuando sean semánticamente relevantes.

---

# 32. No unnecessary cardinality

No toda dimensión deberá añadirse a toda clave.

Ejemplo:

```text
Compiled metadata
```

probablemente no necesita:

```text
request ID
```

---

# 33. Query result key

Conceptualmente:

```text
ResultCacheKey
=
QuerySemanticFingerprint
+
CanonicalParameters
+
LogicalDatabase
+
Tenant
+
ShardSet
+
SchemaGeneration
+
RelevantPolicy
```

---

# 34. SQL string ≠ query identity

No deberá utilizarse únicamente:

```text
hash("SELECT ...")
```

como identidad semántica universal.

---

# 35. Parameter canonicalization

Parámetros deberán representarse de forma:

```text
typed
deterministic
unambiguous
```

---

# 36. Example ambiguity

Estas entradas:

```text
int(1)
string("1")
bool(true)
```

no deberán producir la misma identidad si la semántica puede diferir.

---

# 37. Sensitive parameters

Las claves no deberán contener:

```text
password
token
PII
query parameters in clear text
```

---

# 38. Hashing sensitive identity

Cuando sea necesario incorporar un valor sensible:

```text
canonicalize
→ keyed digest / safe fingerprint
```

según política de seguridad.

---

# 39. Hash ≠ anonymization guarantee

Un hash simple de:

```text
email
phone
SSN
```

puede ser reversible mediante diccionario.

Por tanto deberá evitarse tratarlo automáticamente como dato no sensible.

---

# 40. Cache namespace

Se propone:

```text
voltstack.database
```

con subdivisiones:

```text
voltstack.database.metadata
voltstack.database.compiled-query
voltstack.database.result
voltstack.database.entity
```

---

# 41. Application isolation

Las claves deberán incluir identidad de aplicación/deployment cuando el provider sea compartido.

---

# 42. Environment isolation

No deberá existir colisión entre:

```text
production
staging
testing
```

---

# 43. Tenant isolation

En multitenancy:

```text
Tenant A
≠
Tenant B
```

en cualquier caché que contenga datos tenant-specific.

---

# 44. Tenant omission

Sólo podrá omitirse TenantId si se demuestra que el dato es global.

---

# 45. Shard isolation

Resultados específicos de shard deberán incorporar:

```text
ShardIdentity
```

o una identidad equivalente.

---

# 46. Cross-shard result

Un resultado federado deberá identificar el conjunto/topología relevante.

---

# 47. Topology generation

Si el resultado depende del mapa de shards:

```text
ShardMapGeneration
```

podrá formar parte de la validez.

---

# 48. Cache generation

Se define:

```text
CacheGeneration
```

como mecanismo lógico de invalidación masiva.

---

# 49. Generation bump

Ejemplo:

```text
metadata generation G5
→ deploy
→ G6
```

Los valores G5 pueden permanecer físicamente almacenados pero serán lógicamente inválidos.

---

# 50. Physical deletion ≠ logical invalidation

Esta separación permite:

```text
fast invalidation
eventual garbage collection
```

---

# 51. Generation types

Podrán existir:

```text
ApplicationGeneration
DatabaseConfigurationGeneration
SchemaGeneration
MetadataGeneration
CacheNamespaceGeneration
TenantGeneration
```

según necesidad.

---

# 52. Generation explosion

No deberán añadirse generaciones sin necesidad.

Cada una deberá representar una frontera real de compatibilidad.

---

# 53. TTL

TTL podrá utilizarse para:

```text
freshness
resource control
garbage collection
```

pero:

> **TTL no constituye por sí solo una garantía de consistencia.**

---

# 54. TTL ≠ invalidation

```text
Expiration
≠
Invalidation
≠
Eviction
```

---

# 55. Expiration

```text
time-based usability limit
```

---

# 56. Invalidation

```text
semantic knowledge that value should no longer be used
```

---

# 57. Eviction

```text
provider removes value for resource reasons
```

---

# 58. Cache consistency

Toda lectura deberá poder evaluarse contra:

```text
DatabaseCacheConsistencyPolicy
```

---

# 59. Possible policies

Conceptualmente:

```text
STRICT
TRANSACTION_AWARE
BOUNDED_STALENESS
EVENTUAL
BEST_EFFORT
DISABLED
```

No todas deberán exponerse inicialmente.

---

# 60. STRICT

Sólo reutiliza un valor cuando el sistema puede demostrar suficiente validez bajo la política.

---

# 61. BOUNDED_STALENESS

Permite antigüedad limitada explícita.

---

# 62. EVENTUAL

Acepta convergencia eventual donde el caso de uso lo permita.

---

# 63. BEST_EFFORT

Sólo deberá utilizarse para datos donde stale information sea aceptable.

---

# 64. Consistency policy ≠ transaction isolation

Son conceptos distintos.

---

# 65. Physical hit flow

```text
Provider
   ↓
HIT
   ↓
Envelope Validation
   ↓
Context Validation
   ↓
Generation Validation
   ↓
Consistency Validation
   ↓
USABLE / UNUSABLE
```

---

# 66. Lookup result

Se propone:

```text
DatabaseCacheLookup
├── MISS
├── HIT_USABLE
├── HIT_STALE
├── HIT_INVALID
├── HIT_INCOMPATIBLE
├── PROVIDER_FAILURE
└── UNKNOWN
```

---

# 67. Stale hit

Un stale hit normalmente se tratará como:

```text
logical miss
```

aunque podrá servir para políticas explícitas stale-while-revalidate.

---

# 68. Stale-while-revalidate

Podrá soportarse en categorías donde la semántica lo permita.

No deberá habilitarse universalmente.

---

# 69. Metadata stale-while-revalidate

Puede ser peligroso si el schema cambió.

Por tanto requiere generaciones/evidencia suficiente.

---

# 70. Transaction awareness

La integración deberá conocer el contexto transaccional relevante.

---

# 71. Fundamental transaction rule

> **VoltStack no deberá publicar en una caché compartida información derivada de cambios no confirmados.**

---

# 72. Incorrect flow

```text
UPDATE database
↓
put cache
↓
COMMIT
```

Si commit falla, Cache podría contener estado inexistente.

---

# 73. Correct flow

```text
UPDATE
↓
schedule cache mutation
↓
COMMIT
↓
afterCommit
↓
publish invalidation/update
```

---

# 74. Transactional Cache Mutation Buffer

Se propone:

```text
Transaction
   ↓
TransactionalCacheMutationBuffer
```

que acumule:

```text
invalidations
deferred puts
generation changes
```

---

# 75. Before commit

Las mutaciones compartidas permanecen pendientes.

---

# 76. Commit success

```text
COMMITTED
↓
flush deferred cache mutations
```

---

# 77. Rollback

```text
ROLLED_BACK
↓
discard deferred cache mutations
```

---

# 78. Commit UNKNOWN

Caso crítico:

```text
COMMIT sent
↓
connection lost
↓
outcome UNKNOWN
```

No puede asumirse:

```text
commit
```

ni:

```text
rollback
```

---

# 79. UNKNOWN policy

Ante outcome `UNKNOWN`, la estrategia por defecto deberá ser conservadora.

Por ejemplo:

```text
invalidate possibly affected cache identities
```

cuando sea posible determinar el conjunto.

---

# 80. UNKNOWN ≠ rollback

Nunca:

```text
UNKNOWN
→ discard invalidation
```

asumiendo rollback.

---

# 81. UNKNOWN ≠ committed cache publish

Tampoco:

```text
UNKNOWN
→ publish new cached state
```

asumiendo commit.

---

# 82. Conservative invalidation

Cuando el resultado sea incierto:

```text
remove potentially stale entries
```

es generalmente más seguro que publicar valores nuevos.

---

# 83. afterCommit failure

Si:

```text
DB commit succeeds
```

pero:

```text
cache invalidation fails
```

la transacción sigue:

```text
COMMITTED
```

---

# 84. Cache failure ≠ transaction rollback

Nunca deberá intentarse redefinir el outcome del commit.

---

# 85. Recovery

La falla de invalidación podrá activar:

```text
retry
outbox
generation bump
reconciliation
telemetry alert
```

según estrategia.

---

# 86. Durable invalidation

Para garantías fuertes podrá utilizarse un patrón equivalente a:

```text
Transactional Outbox
```

---

# 87. Outbox flow

```text
DB Transaction
├── business mutation
└── cache invalidation record
       ↓
COMMIT
       ↓
Outbox Consumer
       ↓
Cache invalidation
```

---

# 88. Outbox optionality

No será obligatorio para toda caché.

Se utilizará cuando el nivel de consistencia requerido lo justifique.

---

# 89. Result Cache

El Result Cache almacena:

```text
canonical result representations
```

no:

```text
live cursors
connections
EntityManager
managed entities
```

---

# 90. Result serialization

```text
Result
↓
Canonical Cache Representation
↓
Serializer
↓
Provider
```

---

# 91. Cache result reconstruction

```text
Provider payload
↓
Deserialize
↓
Validate
↓
Result reconstruction
```

---

# 92. Result cache ≠ hydration cache

El resultado puede requerir hidratación posterior.

---

# 93. Entity Cache

Entity Cache almacena:

```text
canonical entity state
```

no el objeto PHP managed.

---

# 94. Entity cache ≠ IdentityMap

Fundamental:

```text
EntityCache
≠
IdentityMap
```

---

# 95. Entity cache lookup

```text
EntityCache
↓
canonical state
↓
ORM Hydration
↓
IdentityMap
↓
managed entity
```

---

# 96. Existing managed entity

Si IdentityMap ya contiene la entidad:

```text
cache state
```

no deberá sobrescribir ciegamente cambios locales dirty.

---

# 97. Metadata Cache

Puede almacenar:

```text
compiled entity metadata
schema metadata
mapping metadata
```

---

# 98. Metadata immutability

Los objetos almacenados deberán ser:

```text
immutable
or serialized immutable representations
```

---

# 99. Compiled Query Cache

Puede almacenar:

```text
normalized query
semantic result
optimized plan
compiled statement representation
```

según el nivel definido.

---

# 100. Compiled query cache key

Deberá incorporar al menos las generaciones relevantes de:

```text
query semantics
platform
capabilities
schema
compiler
```

cuando correspondan.

---

# 101. Platform-specific compiled cache

SQL compilado para:

```text
PostgreSQL
```

no deberá reutilizarse para:

```text
MySQL
```

---

# 102. Version/capability differences

Incluso dentro de una plataforma:

```text
CapabilitySnapshot
```

puede cambiar el resultado del compiler.

---

# 103. Hydration Plan Cache

Puede almacenar:

```text
hydration plans
accessor plans
mapping plans
```

pero nunca:

```text
hydrated entities
```

---

# 104. Schema metadata cache

Introspection cache deberá incorporar:

```text
coverage
schema identity
generation
platform
```

---

# 105. Partial introspection

Un cache hit con:

```text
PARTIAL
```

no deberá convertirse en:

```text
COMPLETE
```

por estar cacheado.

---

# 106. Cache invalidation architecture

```text
Database Semantic Mutation
        ↓
Invalidation Resolver
        ↓
DatabaseCacheInvalidation
        ↓
Transaction Buffer
        ↓
Commit
        ↓
Cache Gateway
        ↓
Provider
```

---

# 107. Semantic invalidation

La invalidación deberá derivarse de:

```text
affected entities
tables
query dependencies
schema changes
tenant
shard
generations
```

no sólo de SQL strings.

---

# 108. SQL string invalidation anti-pattern

No:

```text
if SQL contains "users"
    clear users cache
```

---

# 109. Invalidation scope

Podrá ser:

```text
KEY
ENTITY
QUERY_DEPENDENCY
TABLE
TENANT
SHARD
NAMESPACE
GENERATION
GLOBAL
```

---

# 110. Smallest safe scope

Preferencia:

> utilizar la invalidación más pequeña que pueda demostrarse segura.

---

# 111. Unknown affected set

Si no puede determinarse de forma segura:

```text
broaden invalidation
```

---

# 112. Under-invalidation

Es más peligrosa que invalidación conservadora porque puede servir datos incorrectos.

---

# 113. Over-invalidation

Reduce rendimiento pero normalmente preserva correctness.

---

# 114. Raw SQL

Raw SQL representa un reto especial.

---

# 115. Raw read

Podrá cachearse sólo si el desarrollador proporciona suficiente metadata semántica.

---

# 116. Raw write

Una escritura raw deberá declarar:

```text
affected resources
```

o provocar invalidación conservadora.

---

# 117. Raw write example

```php
DB::rawMutation($sql)
    ->affectsTable('users')
    ->execute();
```

conceptualmente.

---

# 118. Unsafe raw mutation

Si no puede inferirse qué cambió:

```text
result/entity cache invalidation may broaden
```

según policy.

---

# 119. Schema changes

Una migración puede invalidar:

```text
schema metadata
compiled queries
hydration plans
entity metadata assumptions
results
```

según cambio.

---

# 120. Schema generation

Por ello:

```text
SchemaGeneration
```

deberá formar parte de las claves donde corresponda.

---

# 121. Metadata generation

Cambios de mapping ORM pueden invalidar:

```text
entity cache
hydration plans
compiled entity metadata
```

aunque el DB schema no cambie.

---

# 122. Deployment generation

Puede utilizarse para aislar artefactos incompatibles entre deployments.

---

# 123. Cache tag integration

Si VoltStack Cache soporta tags:

```text
Database Cache
→
Cache Tags Adapter
```

podrá aprovecharlos.

---

# 124. Tags ≠ architectural requirement

Database no deberá depender de que todos los providers soporten tags.

---

# 125. Tag emulation

Si se emulan tags:

```text
provider capability
```

deberá indicar coste y garantías.

---

# 126. Cache provider capabilities

Se propone descubrir:

```text
TTL
atomic add
compare-and-set
tags
distributed locks
multi-get
multi-delete
transactions
serialization
namespace clear
```

---

# 127. Provider capability ≠ Database capability

Debe distinguirse:

```text
CacheProviderCapability
≠
DatabaseCapability
```

---

# 128. Stampede problem

Cuando una clave popular expira:

```text
100 requests
↓
MISS
↓
100 DB queries
```

deberá poder mitigarse.

---

# 129. Stampede protection

Estrategias posibles:

```text
single-flight
distributed lock
lease
probabilistic early refresh
stale-while-revalidate
```

---

# 130. Single-flight

Dentro de un worker:

```text
one computation
+
many waiters
```

---

# 131. Distributed single-flight

Entre workers requerirá capability del provider.

---

# 132. Lock ≠ correctness requirement

Si el lock de stampede falla:

```text
fallback
```

deberá preservar correctness aunque aumente carga.

---

# 133. Lock timeout

No deberá bloquear indefinidamente una operación DB.

---

# 134. Cache penetration

Requests repetidos para datos inexistentes pueden provocar consultas continuas.

---

# 135. Negative caching

Podrá soportarse:

```text
NOT_FOUND
```

por TTL corto cuando sea seguro.

---

# 136. Negative cache ≠ NULL

Debe distinguirse:

```text
entity absent
column NULL
query result empty
cache miss
```

---

# 137. Negative cache invalidation

Un INSERT relevante deberá invalidar resultados `NOT_FOUND`.

---

# 138. Cache serialization

La serialización deberá ser:

```text
versioned
deterministic
safe
```

---

# 139. No arbitrary PHP unserialize

No deberá aceptarse payload externo no confiable mediante:

```php
unserialize($payload);
```

sin una estrategia segura.

---

# 140. Serialization format

Podrán utilizarse formatos:

```text
binary
JSON
MessagePack
custom canonical representation
```

mediante adapters.

---

# 141. Serialization version

Todo payload persistente deberá poder indicar:

```text
SerializationVersion
```

---

# 142. Unknown version

Resultado:

```text
HIT_INCOMPATIBLE
→ logical MISS
```

---

# 143. Class names in payload

No deberán utilizarse nombres PHP de clase como única identidad persistente.

---

# 144. Stable type IDs

Preferir:

```text
stable logical type identifiers
```

---

# 145. Compression

Podrá aplicarse para payloads grandes.

---

# 146. Compression threshold

Deberá ser configurable y medible.

---

# 147. Compression ≠ always faster

La decisión depende de:

```text
payload size
CPU
network
provider
```

---

# 148. Maximum payload

Cada categoría podrá definir límites.

---

# 149. Oversized result

Un resultado demasiado grande deberá:

```text
skip caching
```

en lugar de saturar el provider.

---

# 150. Cache admission policy

No todo valor cacheable debe almacenarse.

---

# 151. Admission factors

Podrán incluir:

```text
payload size
query cost
expected reuse
TTL
provider budget
sensitivity
```

---

# 152. Cache policy

Conceptualmente:

```php
final readonly class DatabaseCachePolicy
{
    public function __construct(
        public bool $enabled,
        public Duration $ttl,
        public DatabaseCacheConsistencyPolicy $consistency,
        public CacheAdmissionPolicy $admission,
        public CacheFailurePolicy $failure,
    ) {}
}
```

---

# 153. Per-query policy

El desarrollador podrá solicitar:

```php
User::query()
    ->where('active', true)
    ->cacheFor('30s')
    ->get();
```

si la API pública decide soportarlo.

---

# 154. Query cache hint

Esto será una:

```text
CacheIntent
```

no una orden absoluta.

---

# 155. Cache policy resolver

```text
Query Cache Intent
+
Global Policy
+
Security Policy
+
Transaction Context
+
Provider Capability
→
Effective Cache Policy
```

---

# 156. Transaction bypass

Dentro de una transacción, una lectura puede necesitar ignorar caché compartida.

---

# 157. Why

Porque la transacción puede requerir observar:

```text
its own uncommitted writes
locked rows
snapshot semantics
```

que una caché externa no representa.

---

# 158. Default transaction policy

Por defecto:

```text
shared result/entity cache reads
=
restricted/bypassed
```

dentro de transacciones donde la semántica pueda divergir.

---

# 159. Read-only transaction

Podrá permitir caché bajo políticas específicas si se demuestra equivalencia.

---

# 160. Pessimistic locking

Una query:

```text
SELECT ... FOR UPDATE
```

no deberá satisfacerse desde caché.

---

# 161. Locking query

Regla:

```text
Lock Intent
→
Database execution required
```

---

# 162. Write query

Nunca deberá considerarse ejecutada por existir información en cache.

---

# 163. Cache failure architecture

El provider puede fallar por:

```text
timeout
connection failure
serialization error
capacity
authentication
cluster outage
```

---

# 164. Cache failure ≠ DB failure

En la mayoría de escenarios:

```text
Cache failure
→
fallback to Database
```

---

# 165. Failure policies

```text
FAIL_OPEN
FAIL_CLOSED
BYPASS
DEGRADE
```

---

# 166. Default for acceleration caches

Normalmente:

```text
FAIL_OPEN
```

significa:

```text
cache unavailable
→ use DB
```

---

# 167. Fail-open naming caution

No significa ignorar seguridad.

Sólo:

```text
optimization unavailable
```

---

# 168. Security-sensitive cache

Si una caché forma parte de un control de seguridad externo, pertenecerá a otro subsistema/policy.

Database cache no deberá asumir esa responsabilidad.

---

# 169. Provider timeout

Cache deberá tener su propio timeout.

---

# 170. Cache timeout ≠ query timeout

```text
CacheLookupTimeout
≠
DatabaseStatementTimeout
```

---

# 171. Slow cache

Si Cache tarda más que DB:

```text
optimization has failed economically
```

aunque siga siendo funcional.

---

# 172. Cache circuit breaker

VoltStack Cache podrá integrar circuit breaker.

Database podrá consumir su estado, pero no implementará uno duplicado sin necesidad.

---

# 173. Provider health

```text
Cache Health
≠
Database Health
```

---

# 174. Cache telemetry

La integración deberá emitir métricas como:

```text
database.cache.lookup
database.cache.hit
database.cache.usable_hit
database.cache.miss
database.cache.stale
database.cache.invalidation
database.cache.provider_failure
database.cache.serialization.duration
database.cache.payload.bytes
database.cache.wait.duration
```

---

# 175. Physical vs usable hit metrics

Deben distinguirse:

```text
physical_hit
usable_hit
```

---

# 176. Hit ratio

No deberá calcularse únicamente con:

```text
provider HIT / requests
```

si muchos hits son inválidos.

---

# 177. Effective hit ratio

Más útil:

```text
usable_hits
/
cacheable_lookups
```

---

# 178. Cardinality

No utilizar:

```text
raw SQL
query parameters
tenant IDs
entity IDs
```

como labels métricos no acotados.

---

# 179. Tracing

Span:

```text
database.cache.lookup
```

podrá incluir:

```text
category
provider
outcome
layer
```

sin datos sensibles.

---

# 180. Query correlation

Podrá relacionarse mediante:

```text
query fingerprint
```

controlado, no SQL raw.

---

# 181. Cache diagnostics

Debug podrá mostrar:

```text
category
layer
provider
physical hit
usable hit
reason rejected
generation
TTL remaining
```

---

# 182. Example diagnostic

```text
Result Cache

Key: [REDACTED-FINGERPRINT]
Layer: L2
Provider: redis
Physical: HIT
Usable: NO
Reason: SCHEMA_GENERATION_MISMATCH
```

---

# 183. Developer Debug Toolbar

La integración con toolbar podrá mostrar:

```text
cache lookups
hits
misses
stale hits
invalidations
provider latency
bytes
```

---

# 184. Debug toolbar ≠ cache implementation

Sólo consume telemetry/diagnostics.

---

# 185. Persistent runtime architecture

Con FrankenPHP:

```text
Worker
├── L1 Database Cache
├── Request A
│   └── L0
├── Request B
│   └── L0
└── Request C
    └── L0
```

---

# 186. L0 reset

Al terminar cada request:

```text
L0
→ destroyed/reset
```

---

# 187. L1 persistence

L1 puede sobrevivir múltiples requests.

Por ello requiere:

```text
generation awareness
bounded memory
eviction
```

---

# 188. L1 tenant safety

Una L1 compartida por worker deberá incluir TenantId en claves relevantes.

---

# 189. No request object in L1

Nunca almacenar:

```text
Request
EntityManager
TransactionContext
DatabaseContext
managed entity
```

en L1.

---

# 190. FrankenPHP worker reload

Un deployment podrá:

```text
bump generation
or
recycle workers
```

para eliminar L1 incompatible.

---

# 191. RoadRunner

Aplicará las mismas reglas de worker persistence.

---

# 192. OpenSwoole

L1 deberá ser:

```text
coroutine-safe
```

si existe acceso concurrente.

---

# 193. Concurrent access

Una implementación memory cache deberá controlar:

```text
race conditions
double initialization
mutation
eviction concurrency
```

---

# 194. Immutable cached values

Preferir:

```text
immutable payloads
```

para L1 compartida.

---

# 195. Cache mutation

Actualizar una entrada deberá reemplazar el envelope completo.

No mutarlo parcialmente desde múltiples requests.

---

# 196. Memory budgets

L1 deberá tener límites:

```text
max entries
max bytes
TTL
eviction policy
```

---

# 197. Unbounded L1 prohibited

Nunca:

```php
static array $cache = [];
```

sin límites.

---

# 198. Cache eviction policy

Podrá utilizar:

```text
LRU
LFU
TTL
size-based
hybrid
```

como implementación.

---

# 199. Eviction policy ≠ consistency policy

No confundir.

---

# 200. Cache warming

VoltStack podrá soportar:

```text
database:cache:warm
```

para artefactos apropiados.

---

# 201. Warmable categories

Principalmente:

```text
metadata
compiled mappings
known compiled query templates
```

---

# 202. Result cache warming

Deberá ser explícito porque puede:

```text
execute database queries
consume significant resources
```

---

# 203. Cache clear

CLI:

```bash
php voltstack database:cache:clear
```

---

# 204. Scoped clear

Preferible:

```bash
php voltstack database:cache:clear --category=metadata
```

o:

```bash
php voltstack database:cache:clear --connection=main
```

---

# 205. Tenant clear

En sistemas multitenant:

```bash
php voltstack database:cache:clear --tenant=...
```

requerirá autorización y seguridad adecuadas.

---

# 206. Global clear

Deberá considerarse operación administrativa potencialmente costosa.

---

# 207. Cache inspect

Podrá existir:

```bash
php voltstack database:cache:status
```

---

# 208. Example output

```text
Database Cache

Metadata
  provider: local
  status: available
  generation: 14

Compiled Query
  provider: local
  status: available

Result
  provider: redis
  status: available
  consistency: transaction-aware
```

---

# 209. Cache prune

Podrá eliminar generaciones antiguas.

---

# 210. Prune safety

Sólo deberá borrar entradas pertenecientes a namespaces cuya propiedad pueda demostrarse.

---

# 211. Shared Redis safety

Nunca:

```text
FLUSHALL
```

como implementación de:

```text
database:cache:clear
```

---

# 212. Namespace ownership

VoltStack Database sólo podrá borrar:

```text
its own namespace
```

---

# 213. Provider prefix ≠ ownership proof

Un prefijo por sí solo puede ser insuficiente en escenarios administrativos peligrosos.

---

# 214. Cache security

Datos cacheados pueden contener:

```text
PII
business data
entity state
query results
```

por lo que deberán considerarse datos sensibles.

---

# 215. Encryption

VoltStack Cache podrá proporcionar encryption-at-rest/in-transit.

Database podrá requerirla mediante policy.

---

# 216. Cache encryption ≠ DB encryption

Son controles separados.

---

# 217. Shared provider

Si múltiples aplicaciones comparten Redis:

```text
namespace isolation
authentication
authorization
TLS
```

deberán configurarse.

---

# 218. Cache poisoning

Payloads deberán validarse antes de consumo.

---

# 219. Integrity

Podrá incluirse:

```text
signature
MAC
version
type identity
```

cuando el modelo de amenaza lo requiera.

---

# 220. Deserialization failure

Resultado:

```text
HIT_INVALID
→ remove/bypass
→ database fallback
```

normalmente.

---

# 221. Never trust cached class graph

No reconstruir objetos arbitrarios controlados por payload.

---

# 222. Entity cache security

Los datos deberán respetar:

```text
tenant
security scope
data classification
```

---

# 223. Authorization ≠ cache key only

No debe asumirse que incluir:

```text
user_id
```

en la clave reemplaza Authorization.

---

# 224. Authorization still applies

El resultado obtenido desde cache deberá seguir las políticas de acceso correspondientes cuando sea necesario.

---

# 225. Sensitive field caching

Campos marcados:

```text
NEVER_CACHE
```

podrán impedir que una entidad/proyección completa se almacene.

---

# 226. Cache redaction

Otra estrategia:

```text
cache only allowed fields
```

pero deberá preservar semántica del consumidor.

---

# 227. Partial cache state

No deberá presentarse como entidad completa sin LoadedFieldMask o representación apropiada.

---

# 228. Multitenancy integration

La arquitectura:

```text
TenantContext
      ↓
DatabaseContext
      ↓
CacheContext
      ↓
CacheKey
```

---

# 229. Missing tenant context

Para un cache namespace tenant-scoped:

```text
missing TenantContext
→ reject lookup
```

en vez de usar un namespace global accidentalmente.

---

# 230. Cross-tenant admin operations

Deberán ser explícitas y no reutilizar claves de un tenant arbitrario.

---

# 231. Tenant generation

Podrá utilizarse:

```text
TenantCacheGeneration
```

para invalidación masiva de un tenant.

---

# 232. Sharding integration

```text
Partition Router
      ↓
Resolved ShardSet
      ↓
Cache Identity
```

---

# 233. Routing before cache

Cuando la identidad del resultado dependa del shard:

```text
partition routing
```

deberá ocurrir antes de formar la clave final.

---

# 234. Replica integration

Una lectura desde cache no necesita seleccionar réplica si puede satisfacerse correctamente.

Pero su validez deberá respetar la consistencia solicitada.

---

# 235. Sticky writer semantics

Después de un write:

```text
sticky-to-writer
```

puede implicar bypass temporal de result cache compartida si la caché no garantiza read-your-writes.

---

# 236. Read-your-writes

Podrá satisfacerse mediante:

```text
L0 overlay
writer read
version-aware cache
cache bypass
```

según diseño.

---

# 237. Cache overlay

Una optimización futura podrá mantener:

```text
transaction/request-local cache overlay
```

pero no será equivalente a publicar en L2.

---

# 238. Backup integration

Cache no forma parte de la fuente autoritativa del backup DB.

---

# 239. Restore

Después de restore:

```text
cache generations
```

deberán invalidarse/reconciliarse.

---

# 240. Restore ≠ cache restore

Restaurar DB sin invalidar cache puede producir datos incompatibles.

---

# 241. Migration integration

Una migration exitosa deberá producir las invalidaciones estructurales necesarias.

---

# 242. Failed migration

No deberá bump automáticamente generaciones como si el nuevo schema estuviera completamente activo si el outcome no lo demuestra.

---

# 243. Zero-downtime migration

Durante expand/contract pueden coexistir:

```text
multiple schema-compatible generations
```

y la política de cache deberá contemplarlo.

---

# 244. Cache key versioning

Puede utilizar:

```text
application compatibility generation
```

para evitar que versiones incompatibles compartan payloads.

---

# 245. Rolling deployment

Ejemplo:

```text
Worker V1
Worker V2
Redis shared
```

requiere payloads:

```text
compatible
or
version-isolated
```

---

# 246. Serialization compatibility

Debe definirse:

```text
reader compatibility
writer compatibility
```

entre versiones.

---

# 247. Backward compatibility

Un nuevo release no deberá interpretar un payload viejo incompatible como válido.

---

# 248. Provider abstraction

VoltStack Cache podrá soportar:

```text
Memory
Filesystem
Redis
Memcached
custom providers
```

pero Database deberá declarar qué capacidades requiere por categoría.

---

# 249. Example

Result cache distribuido puede requerir:

```text
shared visibility
TTL
atomic operations
```

mientras Metadata L1 no.

---

# 250. Capability requirements

```text
DatabaseCacheCategory
        ↓
Required Cache Capabilities
        ↓
Provider Selection
```

---

# 251. Provider fallback

Si provider preferido no existe:

```text
fallback provider
```

sólo se utilizará si satisface los requisitos semánticos.

---

# 252. No silent downgrade

No:

```text
Redis unavailable
→ memory cache
```

si el resultado requiere compartir invalidaciones entre workers.

---

# 253. Degraded mode

Podrá ser:

```text
cache disabled
```

en vez de usar una caché semánticamente insuficiente.

---

# 254. Configuration integration

`313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md` proporcionará:

```text
DatabaseCacheIntegrationConfiguration
```

---

# 255. Example configuration

```php
'cache' => [
    'metadata' => [
        'enabled' => true,
        'provider' => 'local',
    ],

    'compiled_queries' => [
        'enabled' => true,
        'provider' => 'local',
    ],

    'results' => [
        'enabled' => true,
        'provider' => 'redis',
        'ttl' => '30s',
        'consistency' => 'transaction-aware',
    ],

    'entities' => [
        'enabled' => false,
    ],
],
```

---

# 256. Configuration compilation

Strings como:

```text
30s
transaction-aware
redis
```

deberán resolverse durante Config compilation.

---

# 257. Container integration

`312_DATABASE_CONTAINER_INTEGRATION_SYSTEM.md` deberá registrar:

```text
DatabaseCacheGateway
DatabaseCachePolicyResolver
DatabaseCacheKeyFactory
DatabaseCacheSerializer
DatabaseCacheInvalidationCoordinator
```

---

# 258. Lifetimes

Ejemplo:

```text
Cache Adapter
→ shared

Cache Policy Resolver
→ shared immutable/stateless

Transaction Cache Mutation Buffer
→ transaction scoped

L0 Cache
→ operation scoped
```

---

# 259. No transaction buffer singleton

Prohibido.

---

# 260. Event integration

El próximo sistema:

```text
315_DATABASE_EVENT_SYSTEM_INTEGRATION.md
```

permitirá publicar eventos relacionados con:

```text
cache invalidation
cache provider failure
cache generation change
```

sin convertir Events en requisito para correctness básico.

---

# 261. Events ≠ invalidation guarantee

Una invalidación crítica no deberá depender exclusivamente de un evento best-effort si la policy exige garantía más fuerte.

---

# 262. Telemetry integration

Database Cache utilizará el sistema general de Telemetry mediante adapters.

---

# 263. Cache telemetry disabled

Desactivar telemetry no deberá cambiar comportamiento de caché.

---

# 264. Testing architecture

Se propone:

```text
tests/Quantum/Database/Integration/Cache/
├── DatabaseCacheIntegrationTest.php
├── NullDatabaseCacheProviderTest.php
├── VoltStackCacheAdapterTest.php
├── DatabaseCacheKeyTest.php
├── DatabaseCacheEnvelopeTest.php
├── DatabaseCacheGenerationTest.php
├── DatabaseCacheConsistencyTest.php
├── TransactionAwareCacheTest.php
├── CacheRollbackTest.php
├── UnknownCommitCacheTest.php
├── CacheInvalidationTest.php
├── ResultCacheIntegrationTest.php
├── EntityCacheIntegrationTest.php
├── MetadataCacheIntegrationTest.php
├── CompiledQueryCacheIntegrationTest.php
├── MultiLevelCacheTest.php
├── CacheStampedeTest.php
├── CacheSerializationTest.php
├── CacheSecurityTest.php
├── CacheTenantIsolationTest.php
├── CacheShardIsolationTest.php
├── PersistentRuntimeCacheTest.php
└── CacheProviderFailureTest.php
```

---

# 265. Unit tests

Sin provider real:

```text
key generation
policy resolution
consistency decisions
envelope validation
generation checks
serialization contracts
invalidation planning
```

---

# 266. Integration tests

Con provider real:

```text
Redis
memory adapter
other official providers
```

cuando corresponda.

---

# 267. Real DB + real Cache

Casos importantes deberán usar:

```text
real DBMS
+
real cache provider
```

para demostrar interacción.

---

# 268. Transaction test

```text
BEGIN
UPDATE entity
schedule invalidation
ROLLBACK
```

Debe demostrar:

```text
shared cache mutation not published as committed state
```

---

# 269. Commit test

```text
BEGIN
UPDATE
COMMIT
afterCommit
invalidate
```

---

# 270. UNKNOWN test

Simular:

```text
COMMIT
connection loss at ambiguous boundary
```

y comprobar que no se publica estado nuevo como confirmado.

---

# 271. Tenant isolation test

```text
Tenant A key
≠
Tenant B key
```

para misma query/parámetros.

---

# 272. Shard isolation test

```text
Shard A
≠
Shard B
```

cuando el dato sea shard-specific.

---

# 273. Rolling deployment test

Dos serialization generations deberán coexistir sin interpretación incorrecta.

---

# 274. Provider outage test

```text
Cache down
→ DB still correct
```

para caches de aceleración fail-open.

---

# 275. Stampede test

100 concurrent misses deberán demostrar:

```text
bounded database recomputation
```

si single-flight está habilitado.

---

# 276. Stampede failure test

Si el mecanismo de lock falla:

```text
correctness preserved
```

aunque aumente DB load.

---

# 277. Persistent worker test

Miles de requests deberán demostrar:

```text
bounded L1 memory
no tenant leakage
generation correctness
scope reset
```

---

# 278. Security test

Intentar introducir payload malformado/manipulado deberá producir:

```text
invalid cache entry
```

no ejecución arbitraria.

---

# 279. Performance testing

Deberá medirse:

```text
cache lookup latency
serialization cost
deserialization cost
key construction
provider network latency
hit benefit
invalidation cost
stampede behavior
L1 memory growth
```

---

# 280. Cache benchmark correctness

Una mejora de hit ratio que sirva datos stale fuera de policy:

```text
is not a performance improvement
```

---

# 281. Benchmark categories

```text
L0 hit
L1 hit
L2 hit
miss
stale hit
invalid hit
serialization
invalidation
generation bump
```

---

# 282. Formal cache model

Sea:

```text
K = cache key
E = cached envelope
C = current database context
P = consistency policy
G = current generations
```

Un hit será usable sólo si:

```text
Usable(E, C, P, G)
=
Exists(E)
∧
Compatible(E, C)
∧
GenerationValid(E, G)
∧
ConsistencySatisfied(E, P)
∧
SecurityAllowed(E, C)
```

---

# 283. Physical hit

```text
PhysicalHit(K)
=
ProviderContains(K)
```

---

# 284. Usable hit

```text
UsableHit(K)
=
PhysicalHit(K)
∧
Usable(E, C, P, G)
```

Por tanto:

```text
PhysicalHit
↛
UsableHit
```

---

# 285. Transaction publication

Sea:

```text
M = cache mutation
T = transaction
```

Entonces:

```text
Publish(M)
only if
Outcome(T) = COMMITTED
```

para publicaciones que representan nuevo estado confirmado.

---

# 286. UNKNOWN transaction

Si:

```text
Outcome(T) = UNKNOWN
```

entonces:

```text
PublishNewState(M) = false
```

y:

```text
ConservativeInvalidation(M)
```

podrá ejecutarse según policy.

---

# 287. Cache correctness

Formalmente:

```text
CacheEnabled
⇒
ObservableDatabaseSemanticsPreserved
```

La activación de cache no deberá modificar la respuesta correcta definida por las políticas del sistema.

---

# 288. Cache transparency

Idealmente:

```text
Result(cache enabled)
=
Result(cache disabled)
```

dentro de la consistencia declarada.

---

# 289. Eventual policies

Para consistencia eventual:

```text
Result(cache enabled, t)
```

puede diferir temporalmente, pero sólo dentro del contrato explícito.

---

# 290. Invalidation safety

Sea:

```text
A = actual affected identities
I = invalidated identities
```

Para invalidación estricta:

```text
A ⊆ I
```

Es decir, puede existir sobre-invalidación:

```text
I > A
```

pero no under-invalidation:

```text
A ⊄ I
```

cuando correctness depende de ella.

---

# 291. Cache admission

Sea:

```text
V = expected reuse value
C = cache cost
```

conceptualmente:

```text
Admit
iff
V > C
```

además de cumplir:

```text
security
consistency
size
provider
```

---

# 292. Memory budget

Para L1:

```text
CurrentBytes
≤
ConfiguredBudget
```

salvo tolerancias transitorias acotadas.

---

# 293. Failure semantics

Para cache de aceleración:

```text
CacheFailure
⇒
DatabaseFallback
```

cuando:

```text
DatabaseAvailable
∧
PolicyAllowsFallback
```

---

# 294. Invariantes generales

## DB-CACHE-INT-001

Cache nunca será Database Truth.

## DB-CACHE-INT-002

Cache será opcional.

## DB-CACHE-INT-003

Database podrá funcionar sin Quantum Cache.

## DB-CACHE-INT-004

Physical Hit ≠ Usable Hit.

## DB-CACHE-INT-005

TTL ≠ consistency guarantee.

## DB-CACHE-INT-006

Expiration ≠ invalidation.

## DB-CACHE-INT-007

Invalidation ≠ eviction.

## DB-CACHE-INT-008

Entity Cache ≠ IdentityMap.

## DB-CACHE-INT-009

Result Cache ≠ Result Cursor.

## DB-CACHE-INT-010

Cache failure ≠ Database failure.

---

# 295. Key invariants

## DB-CACHE-INT-011

Keys serán deterministas.

## DB-CACHE-INT-012

Keys no contendrán secretos en claro.

## DB-CACHE-INT-013

Typed parameters conservarán identidad semántica.

## DB-CACHE-INT-014

Tenant-specific keys incluirán TenantIdentity.

## DB-CACHE-INT-015

Shard-specific keys incluirán ShardIdentity.

## DB-CACHE-INT-016

Platform-specific artifacts no cruzarán plataformas.

## DB-CACHE-INT-017

Capability-sensitive artifacts incorporarán generación/fingerprint relevante.

## DB-CACHE-INT-018

Schema-sensitive artifacts incorporarán schema generation cuando corresponda.

## DB-CACHE-INT-019

Application deployments incompatibles no compartirán payload.

## DB-CACHE-INT-020

Request ID no será usado innecesariamente en claves compartidas.

---

# 296. Transaction invariants

## DB-CACHE-INT-021

Uncommitted state no será publicado en shared cache.

## DB-CACHE-INT-022

Rollback descartará deferred cache publication.

## DB-CACHE-INT-023

Commit success permitirá afterCommit publication.

## DB-CACHE-INT-024

afterCommit cache failure no cambiará COMMITTED a FAILED.

## DB-CACHE-INT-025

UNKNOWN commit no se tratará como rollback.

## DB-CACHE-INT-026

UNKNOWN commit no publicará nuevo estado como confirmado.

## DB-CACHE-INT-027

Locking reads no serán servidas desde result cache.

## DB-CACHE-INT-028

Transaction cache bypass será policy-aware.

## DB-CACHE-INT-029

Cache mutation buffer será transaction-scoped.

## DB-CACHE-INT-030

Shared cache no sustituirá transaction isolation.

---

# 297. Persistent runtime invariants

## DB-CACHE-INT-031

L0 será operation-scoped.

## DB-CACHE-INT-032

L1 podrá sobrevivir requests.

## DB-CACHE-INT-033

L1 tendrá memory budget.

## DB-CACHE-INT-034

L1 será generation-aware.

## DB-CACHE-INT-035

L1 no almacenará EntityManager.

## DB-CACHE-INT-036

L1 no almacenará TransactionContext.

## DB-CACHE-INT-037

L1 no almacenará managed entities.

## DB-CACHE-INT-038

Tenant data no cruzará requests de tenants distintos.

## DB-CACHE-INT-039

OpenSwoole cache local será concurrency-safe.

## DB-CACHE-INT-040

Worker recycle podrá invalidar L1 completamente.

---

# 298. Security invariants

## DB-CACHE-INT-041

Payload cacheado será considerado potencialmente sensible.

## DB-CACHE-INT-042

Deserialization será segura.

## DB-CACHE-INT-043

No habrá arbitrary object unserialization.

## DB-CACHE-INT-044

Authorization no será reemplazada por cache key design.

## DB-CACHE-INT-045

Sensitive fields podrán prohibir caching.

## DB-CACHE-INT-046

Debug output será redacted.

## DB-CACHE-INT-047

Telemetry no expondrá cache payload.

## DB-CACHE-INT-048

Shared providers tendrán namespace isolation.

## DB-CACHE-INT-049

Database cache clear no utilizará `FLUSHALL`.

## DB-CACHE-INT-050

Cache poisoning será tratado como entrada inválida.

---

# 299. Provider invariants

## DB-CACHE-INT-051

Provider selection será capability-aware.

## DB-CACHE-INT-052

No habrá silent semantic downgrade.

## DB-CACHE-INT-053

Provider outage podrá degradar a DB cuando sea seguro.

## DB-CACHE-INT-054

Provider timeout será independiente de DB timeout.

## DB-CACHE-INT-055

Tags no serán requisito universal.

## DB-CACHE-INT-056

Distributed locks no serán asumidos universalmente.

## DB-CACHE-INT-057

Memory provider no sustituirá Redis cuando se requiera shared visibility.

## DB-CACHE-INT-058

Provider capability será verificable.

## DB-CACHE-INT-059

Provider health ≠ DB health.

## DB-CACHE-INT-060

Provider implementation no definirá Database consistency semantics.

---

# 300. Invalidation invariants

## DB-CACHE-INT-061

Invalidation será semántica.

## DB-CACHE-INT-062

Raw SQL mutation requerirá affected-resource evidence o invalidación conservadora.

## DB-CACHE-INT-063

Schema changes invalidarán artefactos incompatibles.

## DB-CACHE-INT-064

Metadata changes invalidarán artefactos ORM incompatibles.

## DB-CACHE-INT-065

Restore provocará reconciliación/invalidation apropiada.

## DB-CACHE-INT-066

Over-invalidation podrá aceptarse por seguridad.

## DB-CACHE-INT-067

Under-invalidation no será aceptada cuando rompa correctness.

## DB-CACHE-INT-068

Generation bump podrá invalidar lógicamente sin eliminación física.

## DB-CACHE-INT-069

Old generations podrán garbage-collectarse posteriormente.

## DB-CACHE-INT-070

Invalidation failure será observable.

---

# 301. Serialization invariants

## DB-CACHE-INT-071

Payload tendrá serialization version.

## DB-CACHE-INT-072

Unknown serialization version será incompatible.

## DB-CACHE-INT-073

Class name no será única identidad persistente.

## DB-CACHE-INT-074

Entity objects managed no serán serializados directamente.

## DB-CACHE-INT-075

Cursor no será serializado.

## DB-CACHE-INT-076

Connection no será serializada.

## DB-CACHE-INT-077

TransactionContext no será serializado.

## DB-CACHE-INT-078

Payload size será acotable.

## DB-CACHE-INT-079

Oversized payload podrá ser rechazado.

## DB-CACHE-INT-080

Compression no alterará semántica.

---

# 302. Developer experience invariants

## DB-CACHE-INT-081

Cache será opt-in/configurable por categoría.

## DB-CACHE-INT-082

Diagnostics distinguirán physical/usable hit.

## DB-CACHE-INT-083

Diagnostics explicarán por qué un hit fue rechazado.

## DB-CACHE-INT-084

CLI aplicará namespace-safe operations.

## DB-CACHE-INT-085

Per-query cache intent será policy-controlled.

## DB-CACHE-INT-086

Deshabilitar cache será una operación válida.

## DB-CACHE-INT-087

Cache configuration será tipada tras compilation.

## DB-CACHE-INT-088

Cache errors no ocultarán query errors.

## DB-CACHE-INT-089

Provider-specific detalles no contaminarán Query API.

## DB-CACHE-INT-090

Defaults favorecerán correctness.

---

# 303. Testing invariants

## DB-CACHE-INT-091

Cache-enabled y cache-disabled deberán preservar semántica.

## DB-CACHE-INT-092

Rollback cache behavior será probado.

## DB-CACHE-INT-093

UNKNOWN commit será probado.

## DB-CACHE-INT-094

Tenant isolation será probada.

## DB-CACHE-INT-095

Shard isolation será probada.

## DB-CACHE-INT-096

Serialization incompatibility será probada.

## DB-CACHE-INT-097

Provider outage será probada.

## DB-CACHE-INT-098

Persistent worker memory será probada.

## DB-CACHE-INT-099

Stampede protection será probada.

## DB-CACHE-INT-100

Malicious payload handling será probado.

---

# 304. Anti-pattern: cache as truth

Incorrecto:

```text
cache says entity exists
→ therefore DB entity exists
```

---

# 305. Anti-pattern: cache managed entities

```php
$cache->set(
    'user:1',
    $entityManager->find(User::class, 1)
);
```

si el objeto permanece managed.

Incorrecto.

---

# 306. Anti-pattern: write-through before commit

```text
UPDATE
↓
Cache PUT
↓
COMMIT
```

Incorrecto.

---

# 307. Anti-pattern: TTL-only correctness

```text
TTL = 5 seconds
→ data is consistent
```

Incorrecto.

---

# 308. Anti-pattern: SQL string identity

```php
$key = md5($sql);
```

como identidad completa del resultado.

Insuficiente.

---

# 309. Anti-pattern: tenant omission

```text
same query
+
same parameters
→ same key
```

ignorando tenant.

Crítico.

---

# 310. Anti-pattern: shard omission

Igualmente incorrecto cuando el resultado depende del shard.

---

# 311. Anti-pattern: cache everything

No todo query/result debe almacenarse.

---

# 312. Anti-pattern: unbounded local cache

```php
private static array $cache = [];
```

sin eviction/budget.

---

# 313. Anti-pattern: Redis required by Database Core

Database no deberá importar un proveedor concreto.

---

# 314. Anti-pattern: Redis down means application down

Para caches puramente aceleradoras:

```text
Redis failure
```

normalmente deberá provocar bypass/fallback, no fallo de Database.

---

# 315. Anti-pattern: FLUSHALL

Nunca como estrategia normal de invalidación.

---

# 316. Anti-pattern: cache bypasses authorization

```text
cached result exists
→ return without access check
```

Incorrecto.

---

# 317. Anti-pattern: stale schema metadata

Un introspection result cacheado antes de migration no deberá seguir considerándose válido automáticamente.

---

# 318. Anti-pattern: average hit ratio only

Una tasa de hit alta puede ocultar:

```text
stale hits
invalid hits
provider latency
huge serialization cost
```

---

# 319. Anti-pattern: stampede lock required for correctness

Si perder el lock produce datos incorrectos, la arquitectura está mal definida.

---

# 320. Anti-pattern: transaction UNKNOWN treated as rollback

Puede dejar cache stale.

---

# 321. Anti-pattern: serialization with arbitrary PHP object graphs

Introduce:

```text
compatibility
security
lifecycle
identity
```

problemáticos.

---

# 322. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Cache
```

---

# 323. Proposed structure

```text
src/Quantum/Database/Integration/Cache/
├── Contract/
│   ├── DatabaseCacheProvider.php
│   ├── DatabaseCacheSerializer.php
│   └── DatabaseCacheCapabilityProvider.php
│
├── Adapter/
│   ├── NullDatabaseCacheProvider.php
│   └── VoltStackCacheAdapter.php
│
├── Gateway/
│   └── DatabaseCacheGateway.php
│
├── Key/
│   ├── DatabaseCacheKey.php
│   ├── DatabaseCacheKeyFactory.php
│   ├── CacheNamespace.php
│   └── CacheContextFingerprint.php
│
├── Envelope/
│   └── DatabaseCacheEnvelope.php
│
├── Policy/
│   ├── DatabaseCachePolicy.php
│   ├── DatabaseCachePolicyResolver.php
│   ├── DatabaseCacheConsistencyPolicy.php
│   ├── CacheAdmissionPolicy.php
│   └── CacheFailurePolicy.php
│
├── Generation/
│   ├── CacheGeneration.php
│   ├── CacheGenerationManager.php
│   └── CacheGenerationSet.php
│
├── Invalidation/
│   ├── DatabaseCacheInvalidation.php
│   ├── CacheInvalidationResolver.php
│   ├── CacheInvalidationCoordinator.php
│   └── TransactionalCacheMutationBuffer.php
│
├── Serialization/
│   ├── DatabaseCachePayload.php
│   ├── SerializationVersion.php
│   └── DatabaseCachePayloadValidator.php
│
├── Stampede/
│   ├── CacheSingleFlight.php
│   └── CacheLeaseCoordinator.php
│
├── Telemetry/
│   └── DatabaseCacheTelemetry.php
│
└── Diagnostics/
    └── DatabaseCacheDiagnostics.php
```

---

# 324. Dependency model

```text
Query / ORM / Schema
        ↓
Specialized Database Cache
        ↓
DatabaseCacheGateway
        ↓
Database Cache Contract
        ↓
Integration Adapter
        ↓
VoltStack Quantum Cache
        ↓
Concrete Provider
```

Nunca:

```text
Redis
↑
ORM
```

---

# 325. Architectural interaction

Modelo completo:

```text
Application
    ↓
Database Public API
    ↓
Query / ORM
    ↓
Cache Policy Resolver
    ↓
Cache Lookup
    │
    ├── usable hit
    │      ↓
    │    return
    │
    └── miss/unusable
           ↓
       Query Engine
           ↓
       Execution Engine
           ↓
        Database
           ↓
       Result/Hydration
           ↓
       Cache Admission
           ↓
       Cache Publication
```

Para writes:

```text
Application
    ↓
Database Mutation
    ↓
Transaction
    ↓
Persistence
    ↓
Database
    ↓
Deferred Cache Mutation
    ↓
Commit Outcome
    ├── COMMITTED
    │      ↓
    │   afterCommit
    │      ↓
    │   invalidate/publish
    │
    ├── ROLLED_BACK
    │      ↓
    │   discard
    │
    └── UNKNOWN
           ↓
       conservative policy
```

---

# 326. Implementación por fases

## Fase 1 — V1

Implementar:

```text
DatabaseCacheProvider contract
Null adapter
VoltStack Cache adapter
Metadata cache integration
Compiled query cache integration
Result cache basic integration
Cache keys
Namespaces
TTL
Generation awareness
Transaction-aware invalidation
afterCommit publication
Rollback handling
Safe provider fallback
Telemetry
```

---

# 327. Fase 2

Agregar:

```text
Entity cache
multi-level L0/L1/L2
stampede protection
negative caching
advanced invalidation
provider capability negotiation
serialization versioning
```

---

# 328. Fase 3

Agregar:

```text
distributed invalidation
outbox-backed invalidation
rolling deployment compatibility
tenant generations
shard generations
cache warming
advanced diagnostics
```

---

# 329. Fase 4

Agregar:

```text
adaptive admission
automatic hot-key detection
cost-aware caching
distributed single-flight
advanced stale-while-revalidate
cache topology optimization
```

sin permitir que optimizaciones adaptativas modifiquen las garantías de consistencia.

---

# 330. Checklist de implementación

Antes de considerar completo el sistema:

- [ ] Cache es opcional.
- [ ] Null adapter implementado.
- [ ] VoltStack Cache adapter implementado.
- [ ] categorías separadas.
- [ ] claves tipadas.
- [ ] namespaces seguros.
- [ ] tenant isolation.
- [ ] shard isolation.
- [ ] generation awareness.
- [ ] schema generation integration.
- [ ] metadata generation integration.
- [ ] physical hit separado de usable hit.
- [ ] consistency policy implementada.
- [ ] transaction mutation buffer.
- [ ] afterCommit publication.
- [ ] rollback discard.
- [ ] UNKNOWN commit policy.
- [ ] result cache.
- [ ] metadata cache.
- [ ] compiled query cache.
- [ ] entity cache opcional.
- [ ] serialization versioning.
- [ ] payload validation.
- [ ] no arbitrary unserialize.
- [ ] size limits.
- [ ] failure fallback.
- [ ] provider capability checks.
- [ ] telemetry.
- [ ] diagnostics.
- [ ] CLI safe clear/status.
- [ ] L1 memory budget.
- [ ] FrankenPHP worker tests.
- [ ] tenant leak tests.
- [ ] cache poisoning tests.
- [ ] provider outage tests.
- [ ] stampede tests.
- [ ] transaction tests.

---

# 331. Principio definitivo

La integración Cache de VoltStack Database deberá regirse por cuatro reglas:

```text
1. Database is truth.
2. Cache is optional.
3. Cache reuse requires evidence of usability.
4. Transaction outcomes govern cache publication.
```

En forma resumida:

```text
Cache Hit
     ↓
Validation
     ↓
Consistency
     ↓
Context
     ↓
Usable?
 ┌───┴───┐
 │       │
YES      NO
 │       │
return   Database
```

---

# 332. Resultado arquitectónico

Con esta arquitectura VoltStack podrá utilizar:

```text
in-memory cache
Redis
distributed providers
future custom providers
```

para acelerar:

```text
metadata
query compilation
query plans
results
entities
hydration plans
schema metadata
```

sin acoplar Database a un proveedor concreto y, especialmente, sin sacrificar:

```text
transaction correctness
tenant isolation
shard isolation
security
persistent runtime safety
database truth
```

La arquitectura final conserva:

```text
Database
   ↓
Semantic Cache Layer
   ↓
Database Cache Integration Contract
   ↓
VoltStack Cache Adapter
   ↓
Quantum Cache
   ↓
Provider
```

en vez de:

```text
ORM
 ↓
Redis
```

---

# 333. Siguiente documento

```text
315_DATABASE_EVENT_SYSTEM_INTEGRATION.md
```

Definirá la integración entre:

```text
VoltStack/Quantum/Database
        ↕
VoltStack Event System
```

incluyendo:

```text
Database event bridge
query events
connection events
transaction events
ORM lifecycle events
persistence events
schema/migration events
cache-related events
event envelopes
event correlation
synchronous internal events
asynchronous external publication
afterCommit events
rollback handling
UNKNOWN transaction outcomes
event ordering
listener failure semantics
outbox integration
event redaction
tenant/shard context
persistent worker isolation
telemetry correlation
plugin listeners
event versioning
testing
```

manteniendo como reglas centrales:

```text
Event
≠
Command
```

```text
Event
≠
Transaction Outcome
```

y:

```text
Listener Failure After Commit
≠
Database Rollback
```