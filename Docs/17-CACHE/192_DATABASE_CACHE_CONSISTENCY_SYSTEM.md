# 192_DATABASE_CACHE_CONSISTENCY_SYSTEM.md

# VoltStack Quantum Database
## Cache Consistency System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 192 — Cache Consistency System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `191_DATABASE_CACHE_INVALIDATION_SYSTEM.md`  
**Siguiente documento:** `193_DATABASE_FACTORY_SYSTEM.md`

---

# 1. Propósito

`Cache Consistency System` define las garantías que VoltStack puede afirmar cuando una operación de lectura utiliza información almacenada en caché en lugar de consultar directamente el estado actual de la base de datos.

El sistema deberá coordinar semánticamente:

- Query Cache;
- Result Cache;
- Metadata Cache;
- Entity Cache;
- Cache Invalidation;
- Transaction Manager;
- Read/Write Routing;
- Replica Lag Awareness;
- Sticky Connections;
- Failover;
- Sharding;
- Partition Routing;
- Persistent Runtime;
- Multitenancy opcional;
- Telemetry.

La regla maestra será:

> **VoltStack solo utilizará un valor cacheado cuando exista evidencia suficiente de que ese valor satisface la política de consistencia solicitada; existencia, TTL o ausencia de una invalidación observada no constituyen por sí solos prueba de consistencia.**

Formalmente:

```text
CacheReadable(E, R)
=
EntryExists(E)
∧ EntryValid(E)
∧ DomainCompatible(E, R)
∧ GenerationCompatible(E)
∧ VersionEvidenceSufficient(E, R)
∧ ConsistencyPolicySatisfied(E, R)
```

Por tanto:

```text
Exists
≠
Valid
≠
Fresh
≠
ConsistentForThisRead
```

---

# 2. Cache Consistency ≠ Cache Invalidation

El documento anterior definió:

```text
Database Change
      ↓
Semantic Change
      ↓
Invalidation
```

Este documento define:

```text
Read Requirement
      +
Cache Evidence
      +
Database/Replica Evidence
      ↓
May this cached value be observed?
```

Por tanto:

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

---

# 3. Consistency ≠ freshness

Un entry puede ser antiguo pero todavía correcto.

Ejemplo:

```text
Cache created: 10:00
Current time: 10:30
Database unchanged
```

El entry tiene:

```text
Age = 30 minutes
```

pero puede seguir siendo consistente.

A la inversa:

```text
Cache created: 10:00:00
Database changed: 10:00:01
Current time: 10:00:02
```

El entry tiene solo dos segundos y ya puede estar stale.

Por tanto:

```text
Age
≠
Consistency
```

---

# 4. TTL ≠ correctness

Regla fundamental:

> **TTL limita cuánto tiempo puede sobrevivir un entry; no demuestra que el entry siga representando el estado requerido de la base de datos.**

Formalmente:

```text
TTLValid(E)
```

no implica:

```text
DataCurrent(E)
```

---

# 5. Expiration ≠ invalidation

Distinguir:

```text
Expiration
Invalidation
Eviction
Staleness
Consistency Failure
```

---

# 6. Expiration

El entry supera su límite temporal.

```text
now >= expiresAt
```

---

# 7. Invalidation

Existe evidencia semántica de que el entry ya no debe considerarse válido.

---

# 8. Eviction

El backend elimina el entry por razones de recursos.

Ejemplo:

```text
LRU
LFU
memory pressure
```

---

# 9. Staleness

El entry representa una versión anterior a la requerida.

---

# 10. Consistency failure

El sistema no puede demostrar que utilizar el entry satisfaga el contrato solicitado.

Esto puede ocurrir incluso cuando no se sabe que el entry esté stale.

---

# 11. UNKNOWN ≠ STALE

Un entry puede encontrarse en:

```text
KNOWN_CURRENT
KNOWN_STALE
UNKNOWN
```

`UNKNOWN` significa:

> VoltStack no posee evidencia suficiente para determinar su relación con el estado requerido.

---

# 12. UNKNOWN ≠ FRESH

Regla:

```text
UNKNOWN
≠
CURRENT
```

Las políticas estrictas deberán rechazarlo.

---

# 13. Modelo de estados

```php
enum CacheConsistencyState
{
    case CURRENT;
    case ACCEPTABLY_STALE;
    case STALE;
    case EXPIRED;
    case INVALIDATED;
    case UNKNOWN;
    case INCOMPATIBLE;
}
```

---

# 14. CURRENT

Existe evidencia suficiente para afirmar:

```text
EntryVersion >= MinimumRequiredVersion
```

o equivalente semántico.

---

# 15. ACCEPTABLY_STALE

El entry no necesariamente representa el estado más reciente, pero la policy solicitada permite esa antigüedad.

---

# 16. STALE

Existe evidencia de que el entry está detrás del estado mínimo requerido.

---

# 17. EXPIRED

La policy temporal ya no permite utilizarlo.

---

# 18. INVALIDATED

Una dependencia, generación o dominio ha sido invalidado.

---

# 19. UNKNOWN

No puede determinarse validez suficiente.

---

# 20. INCOMPATIBLE

El entry pertenece a otra:

- versión de metadata;
- schema generation;
- tenant;
- shard;
- database;
- codec;
- platform;
- topology;
- policy;
- application generation.

---

# 21. Consistency requirement

Cada lectura cacheable deberá tener una política efectiva.

```php
interface CacheConsistencyRequirement
{
    public function profile(): CacheConsistencyProfile;
}
```

---

# 22. Per-operation policy

No deberá existir la suposición:

```text
one cache consistency mode
for the entire application
```

Una lectura de catálogo puede aceptar staleness.

Una lectura financiera puede requerir evidencia mucho más fuerte.

---

# 23. Per-cache policy

Asimismo:

```text
Metadata Cache
```

posee semántica distinta de:

```text
Result Cache
```

y:

```text
Entity Cache
```

---

# 24. Per-domain policy

La policy podrá variar por:

```text
database
entity
query
repository
cache region
tenant
operation
request
```

---

# 25. Consistency profiles

VoltStack deberá soportar perfiles conceptuales como:

```php
enum CacheConsistencyProfile
{
    case STRICT;
    case READ_YOUR_WRITES;
    case BOUNDED_STALENESS;
    case EVENTUAL;
    case CACHE_ONLY;
}
```

`CACHE_ONLY` deberá considerarse una política especializada, no una garantía de actualidad.

---

# 26. STRICT

El entry solo podrá utilizarse cuando exista evidencia suficiente para satisfacer el estado mínimo requerido por la operación.

No significa necesariamente:

```text
query primary database every time
```

Puede existir una caché con version evidence suficientemente fuerte.

---

# 27. READ_YOUR_WRITES

Después de una escritura confirmada dentro del scope lógico:

```text
subsequent reads
```

no deberán observar una versión anterior a dicha escritura.

---

# 28. BOUNDED_STALENESS

Permite un límite explícito:

```text
maxAge
```

o preferentemente:

```text
maxVersionLag
maxReplicationLag
```

cuando exista evidencia adecuada.

---

# 29. EVENTUAL

Permite temporalmente información anterior, siempre bajo límites operacionales explícitos.

No significa:

```text
anything forever
```

---

# 30. CACHE_ONLY

No consulta la DB como fallback.

Ejemplo:

```text
cache hit
→ return

cache miss
→ explicit miss
```

Debe utilizarse únicamente cuando el dominio lo permita.

---

# 31. Strong consistency terminology

VoltStack evitará declarar simplemente:

```text
STRONG CACHE
```

sin especificar el modelo.

En sistemas distribuidos:

```text
strong
```

puede significar cosas distintas.

La documentación y APIs deberán expresar garantías concretas.

---

# 32. Consistency dimensions

Un `CacheConsistencyProfile` podrá componerse de:

```text
FreshnessRequirement
ReadYourWritesRequirement
MonotonicReadRequirement
TransactionVisibilityRequirement
VersionRequirement
GenerationRequirement
ReplicaRequirement
FailurePolicy
```

---

# 33. CacheConsistencyPolicy

```php
final readonly class CacheConsistencyPolicy
{
    public function __construct(
        public CacheConsistencyProfile $profile,
        public FreshnessRequirement $freshness,
        public ReadYourWritesRequirement $readYourWrites,
        public MonotonicReadRequirement $monotonicReads,
        public CacheFailurePolicy $failurePolicy,
    ) {}
}
```

---

# 34. Consistency evidence

VoltStack no deberá basar decisiones exclusivamente en flags.

Deberá trabajar con evidencia.

```php
interface CacheConsistencyEvidence
{
    public function kind(): CacheConsistencyEvidenceKind;
}
```

---

# 35. Evidence types

Ejemplos:

```text
GenerationEvidence
EntityVersionEvidence
TransactionEvidence
CommitPositionEvidence
ReplicationPositionEvidence
TopologyGenerationEvidence
SchemaGenerationEvidence
MetadataGenerationEvidence
InvalidationEvidence
CreationTimeEvidence
ExpirationEvidence
SourceEndpointEvidence
```

---

# 36. Evidence ≠ guarantee

Una evidencia individual puede no ser suficiente.

Ejemplo:

```text
TTL has not expired
```

es evidencia temporal.

No es prueba de que ningún write haya ocurrido.

---

# 37. Evidence set

```php
final readonly class CacheConsistencyEvidenceSet
{
    /**
     * @param list<CacheConsistencyEvidence> $evidence
     */
    public function __construct(
        public array $evidence,
    ) {}
}
```

---

# 38. Consistency evaluator

```php
interface CacheConsistencyEvaluator
{
    public function evaluate(
        CacheEntryMetadata $entry,
        CacheConsistencyRequirement $requirement,
        CacheConsistencyContext $context,
    ): CacheConsistencyDecision;
}
```

---

# 39. Decision

```php
final readonly class CacheConsistencyDecision
{
    public function __construct(
        public CacheConsistencyState $state,
        public CacheReadAction $action,
        public CacheConsistencyEvidenceSet $evidence,
        public CacheConsistencyReason $reason,
    ) {}
}
```

---

# 40. CacheReadAction

```php
enum CacheReadAction
{
    case USE_CACHE;
    case BYPASS_CACHE;
    case REFRESH_SYNCHRONOUSLY;
    case SERVE_STALE_AND_REFRESH;
    case REJECT;
}
```

---

# 41. Decision pipeline

```text
Cache Entry
    +
Read Requirement
    +
Runtime Context
    +
Transaction Context
    +
Consistency Evidence
        ↓
CacheConsistencyEvaluator
        ↓
CacheConsistencyDecision
        ↓
USE / BYPASS / REFRESH / STALE+REFRESH / REJECT
```

---

# 42. Cache metadata

Todo entry cuya consistencia dependa de contexto deberá incluir metadata suficiente.

Conceptualmente:

```php
final readonly class CacheEntryMetadata
{
    public function __construct(
        public CacheEntryId $id,
        public CacheKind $kind,
        public PersistenceDomain $domain,
        public CacheGenerationVector $generations,
        public ?DataVersion $dataVersion,
        public ?ReplicationPosition $sourcePosition,
        public ?TopologyGeneration $topologyGeneration,
        public Instant $createdAt,
        public ?Instant $expiresAt,
        public CachePayloadVersion $payloadVersion,
    ) {}
}
```

---

# 43. Cache payload ≠ metadata

Separar:

```text
CachePayload
```

de:

```text
CacheEntryMetadata
```

permite evaluar validez sin necesariamente interpretar todo el payload.

---

# 44. Version model

Cuando el dominio lo permita:

```text
Database State Version
```

será preferible a depender exclusivamente de tiempo.

---

# 45. Entity version

Para optimistic locking:

```text
User#42 version = 18
```

el Entity Cache puede almacenar:

```text
User#42
version = 18
```

---

# 46. Required version

Después de una escritura:

```text
MinimumRequiredVersion(User#42) = 18
```

Un entry:

```text
version 17
```

será rechazado.

---

# 47. Version comparison

Formalmente:

```text
CacheVersion(E) >= RequiredVersion(R)
```

cuando exista un orden válido entre versiones.

---

# 48. Version domains

No deberán compararse tokens pertenecientes a dominios incompatibles.

No:

```text
Shard A sequence 100
>
Shard B sequence 90
```

salvo que exista un ordering global explícito.

---

# 49. Opaque versions

Algunos backends pueden ofrecer:

```text
GTID
LSN
binlog position
WAL position
```

Estos deberán encapsularse como tokens tipados.

---

# 50. Core independence

Core no deberá depender directamente de:

```text
PostgreSQL LSN
MySQL GTID
```

Utilizará:

```text
ReplicationPosition
VersionEvidence
```

con adapters de plataforma.

---

# 51. Read-your-writes

Escenario:

```text
Request
│
├── UPDATE User#42
│
├── COMMIT
│
└── SELECT User#42
```

La última lectura no deberá obtener:

```text
Entity Cache version 17
```

si la escritura produjo:

```text
version 18
```

---

# 52. Write evidence

Después de commit confirmado, el scope podrá registrar:

```text
WriteConsistencyToken
```

---

# 53. WriteConsistencyToken

```php
final readonly class WriteConsistencyToken
{
    public function __construct(
        public PersistenceDomain $domain,
        public CacheDependencySet $dependencies,
        public ?DataVersion $minimumVersion,
        public ?ReplicationPosition $minimumPosition,
    ) {}
}
```

---

# 54. Sticky integration

El token podrá provocar:

```text
read from writer
```

mediante:

`180_DATABASE_STICKY_CONNECTION_SYSTEM.md`

hasta que una replica pueda demostrar suficiente progreso.

---

# 55. Cache integration

También podrá provocar:

```text
reject cache entry
```

si el entry no satisface el token.

---

# 56. Sticky DB read ≠ cache consistency

Aunque el router seleccione writer:

```text
cache hit
```

podría impedir llegar al writer.

Por eso Cache Consistency debe evaluarse antes de aceptar el entry.

---

# 57. Read pipeline

```text
Logical Read
    ↓
Consistency Requirement
    ↓
Cache Lookup
    ↓
Cache Consistency Evaluation
    │
    ├── USE_CACHE
    │
    └── BYPASS
           ↓
      Partition Routing
           ↓
      Read/Write Routing
           ↓
      Replica Eligibility
           ↓
      Connection
           ↓
      Database
```

---

# 58. Cache hit ≠ operation complete

Un hit es solamente:

```text
candidate value found
```

Hasta pasar consistency validation.

---

# 59. Semantic cache hit

VoltStack deberá distinguir:

```text
PHYSICAL_HIT
```

de:

```text
USABLE_HIT
```

---

# 60. Physical hit

Backend encontró el key.

---

# 61. Usable hit

El entry pasó:

```text
compatibility
validity
consistency
security
```

---

# 62. Metrics

Por tanto:

```text
cache.physical_hits
cache.usable_hits
cache.rejected_hits
```

deberán poder diferenciarse.

---

# 63. Transaction consistency

Una transacción activa cambia radicalmente las reglas.

---

# 64. Shared cache inside transaction

Una transacción puede tener:

```text
uncommitted local changes
```

que shared cache desconoce.

---

# 65. Therefore

Para determinados perfiles:

```text
Active Transaction
→ bypass shared Result Cache
```

será el default seguro.

---

# 66. Entity Cache inside transaction

Un Entity Cache entry puede utilizarse como fuente inicial solo cuando la policy de transacción y version evidence lo permitan.

Una vez materializada la entidad:

```text
IdentityMap
```

controla la identidad del objeto dentro del scope.

---

# 67. IdentityMap priority

Dentro de un PersistenceContext:

```text
IdentityMap canonical instance
>
Entity Cache object reconstruction
```

Nunca se reemplazará un objeto managed existente por otro creado desde cache.

---

# 68. Transaction-local cache

Podrá existir:

```text
TransactionLocalCache
```

para evitar trabajo repetido dentro de una transacción.

---

# 69. Transaction-local ≠ shared cache

Debe desaparecer al terminar el transaction scope.

---

# 70. Uncommitted state

Nunca deberá publicarse a:

```text
shared Entity Cache
shared Result Cache
```

antes del commit confirmado.

---

# 71. Commit

Después de:

```text
COMMIT CONFIRMED
```

podrán:

1. invalidarse valores anteriores;
2. publicarse nuevos valores seguros;
3. actualizarse consistency tokens.

---

# 72. Rollback

Después de rollback:

```text
pending shared cache publications
→ discard
```

---

# 73. Rollback ≠ object rewind

Como ya estableció Transaction Architecture:

```text
DatabaseRollback
≠
AutomaticObjectGraphRewind
```

Por tanto el cache system tampoco deberá asumir que los objetos PHP han vuelto automáticamente al baseline.

---

# 74. UNKNOWN transaction outcome

Caso:

```text
COMMIT sent
↓
connection lost
↓
UNKNOWN
```

---

# 75. Cache behavior under UNKNOWN

Default:

```text
Do not publish speculative new values
```

y:

```text
Conservatively invalidate possibly affected old values
```

---

# 76. Consistency state

Hasta obtener evidencia adicional:

```text
CacheConsistencyState = UNKNOWN
```

para las dependencias afectadas cuando la policy lo requiera.

---

# 77. UNKNOWN ≠ rollback

Nunca:

```text
UNKNOWN
→ assume old cache is correct
```

---

# 78. UNKNOWN ≠ commit

Nunca:

```text
UNKNOWN
→ publish new object state
```

---

# 79. Replica consistency

El sistema de caché deberá integrarse con:

`179_DATABASE_REPLICA_LAG_AWARENESS_SYSTEM.md`.

---

# 80. Cached-from-replica provenance

Un Result Cache entry podrá registrar:

```text
source role
source replication position
source topology generation
observed freshness evidence
```

cuando sea necesario.

---

# 81. Provenance

```php
final readonly class CacheDataProvenance
{
    public function __construct(
        public ConnectionRole $sourceRole,
        public ?ReplicationPosition $position,
        public ?TopologyGeneration $topology,
        public Instant $observedAt,
    ) {}
}
```

---

# 82. Provenance ≠ trust forever

Que un resultado fuera current al almacenarse no demuestra que continúe current después.

Necesita:

```text
invalidation
generation
version
TTL
```

según policy.

---

# 83. Stale repopulation

Problema:

```text
Primary version 18
Replica version 17
↓
cache invalidated
↓
read hits replica
↓
version 17 stored in cache
```

---

# 84. Repopulation guard

Antes de almacenar:

```text
CandidateVersion >= MinimumRequiredVersion
```

cuando exista evidencia comparable.

---

# 85. Unknown candidate version

Si no puede demostrarse:

```text
candidate sufficiently current
```

una policy estricta deberá impedir shared cache population.

---

# 86. Return ≠ cache

Importante:

Una policy puede permitir devolver un resultado pero no permitir compartirlo globalmente.

```text
Readable
≠
Cacheable
```

---

# 87. Cacheability decision

Separar:

```text
CanReadCandidate?
```

de:

```text
CanPopulateSharedCache?
```

---

# 88. CachePopulationDecision

```php
enum CachePopulationDecision
{
    case STORE_SHARED;
    case STORE_LOCAL_ONLY;
    case DO_NOT_STORE;
}
```

---

# 89. Monotonic reads

Una aplicación puede requerir:

> Una vez que un scope ha observado versión V, no deberá observar posteriormente una versión menor.

---

# 90. Observation token

```text
HighestObservedVersion
```

podrá almacenarse en:

```text
CacheConsistencyContext
```

---

# 91. Example

```text
Read #1 → User version 18
Read #2 → cache version 17
```

Debe rechazarse bajo monotonic reads.

---

# 92. Scope

Monotonicity puede aplicarse a:

```text
request
operation
session
job
transaction
```

según policy.

---

# 93. No global mutable observation state

Prohibido:

```php
static $lastSeenVersion;
```

---

# 94. Persistent workers

Observation state deberá ser scoped.

---

# 95. Bounded staleness

Podrá definirse temporalmente:

```php
new MaximumCacheAge(
    Duration::seconds(10)
);
```

---

# 96. Temporal bounded staleness

Aceptable para dominios donde:

```text
10-second-old data
```

sea explícitamente válido.

---

# 97. Temporal bound limitation

No demuestra que el entry esté solo una versión detrás.

Puede haber miles de writes en diez segundos.

---

# 98. Version-based bound

Cuando sea posible:

```text
maxVersionDistance
```

será semánticamente más fuerte.

---

# 99. Replication-based bound

También:

```text
maxReplicaLag
```

podrá formar parte de la policy.

---

# 100. Multiple bounds

Podrán combinarse:

```text
age <= 10s
AND
replicaLag <= 2s
AND
generation current
```

---

# 101. Eventual consistency

No elimina validaciones estructurales.

Incluso `EVENTUAL` deberá rechazar:

```text
wrong tenant
wrong schema generation
corrupt payload
invalidated entry
incompatible codec
```

---

# 102. Eventual ≠ unsafe

La relajación será sobre actualidad permitida, no sobre aislamiento o integridad.

---

# 103. Metadata Cache consistency

Metadata Cache es diferente.

Normalmente:

```text
MetadataGeneration == RuntimeMetadataGeneration
```

deberá ser exacto.

---

# 104. Stale metadata

No deberá servirse como:

```text
eventually consistent metadata
```

si eso puede interpretar incorrectamente entidades o tipos.

---

# 105. Metadata rule

> **La metadata requerida para interpretar correctamente datos deberá ser compatible con la generación activa del runtime.**

---

# 106. Query Cache consistency

El Query Cache definido en el documento 187 almacena artefactos reutilizables de query, no materialized result values.

Su consistencia dependerá principalmente de:

```text
Query fingerprint
Schema generation
Metadata generation
Dialect
Platform capabilities
Compiler/planner generation
Extension generation
```

---

# 107. Row writes

Un cambio:

```text
User#42.status
```

normalmente no invalida un query plan estructural.

---

# 108. Schema changes

En cambio:

```text
drop column
rename column
index/capability changes
```

pueden volver incompatible un artefacto query cacheado.

---

# 109. Result Cache consistency

Result Cache contiene datos derivados de ejecución.

Por tanto depende de:

```text
data mutations
query dependencies
tenant/shard domain
read provenance
generation
consistency requirement
```

---

# 110. Entity Cache consistency

Entity Cache almacena estado persistente reutilizable entre scopes.

No almacena:

```text
live managed object identity
```

---

# 111. Entity Cache hit

Pipeline:

```text
Entity Cache Payload
       ↓
Consistency Validation
       ↓
Type Conversion / Mapping
       ↓
Current PersistenceContext
       ↓
IdentityMap
       ↓
Canonical Managed Entity
```

---

# 112. Never share entity object

Prohibido:

```text
Worker A PHP User object
→ shared cache
→ Worker B reuses same managed object semantics
```

El payload deberá ser data, no contexto ORM vivo.

---

# 113. Dirty managed entity

Si el current IdentityMap ya contiene:

```text
User#42 DIRTY
```

un Entity Cache hit no deberá sobrescribirlo.

---

# 114. Existing entity state

Cache Consistency System podrá marcar:

```text
external state may be newer
```

pero la reconciliación pertenece al ORM/refresh policy.

---

# 115. Result Cache and entities

Si Result Cache contiene una representación de una consulta de entidades, deberá evitar crear identidades ORM duplicadas.

---

# 116. Preferred strategy

Cachear:

```text
entity IDs
scalar payloads
canonical persistent snapshots
```

y reensamblar mediante IdentityMap/Entity Cache según arquitectura.

---

# 117. Hydration Cache

`141_DATABASE_HYDRATION_CACHE_SYSTEM.md` almacena:

```text
hydration plans/accessors
```

no valores persistentes.

Su consistency contract es principalmente estructural.

---

# 118. Compiled Query Cache

`75_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md` también posee consistencia estructural.

---

# 119. Cache taxonomy

```text
Database Cache
│
├── Structural Caches
│   ├── Compiled Query Cache
│   ├── Query Cache
│   ├── Hydration Cache
│   └── Metadata Cache
│
└── Data Caches
    ├── Result Cache
    └── Entity Cache
```

Las políticas no serán idénticas entre ambos grupos.

---

# 120. Structural consistency

Principalmente:

```text
generation compatibility
schema compatibility
metadata compatibility
platform compatibility
extension compatibility
```

---

# 121. Data consistency

Adicionalmente:

```text
mutation dependencies
data versions
invalidation
replication
transaction visibility
read-your-writes
```

---

# 122. Schema evolution

Rolling migration:

```text
App V1
App V2
Database transitional schema
```

puede requerir múltiples generaciones compatibles.

---

# 123. Generation compatibility ≠ equality always

Una futura compatibility policy podrá declarar:

```text
Generation 18
compatible with 19
```

para determinados artefactos.

---

# 124. Default

Sin evidencia explícita:

```text
generation mismatch
→ incompatible
```

---

# 125. Cache payload versioning

Todo payload persistente de larga duración deberá poseer versión.

```text
CachePayloadVersion
```

---

# 126. Codec compatibility

Cambio de serializer:

```text
Codec V1
→ Codec V2
```

no deberá intentar decodificar payload antiguo arbitrariamente.

---

# 127. Safe deserialization

Prohibido utilizar cache payload como fuente de:

```text
arbitrary PHP object unserialization
```

sin contrato seguro.

---

# 128. Security consistency

Consistency incluye aislamiento.

Un entry de:

```text
Tenant A
```

nunca será consistente para:

```text
Tenant B
```

aunque los datos coincidan accidentalmente.

---

# 129. Domain compatibility

Formalmente:

```text
EntryDomain == RequiredPersistenceDomain
```

salvo mappings explícitos seguros.

---

# 130. PersistenceDomain

Puede incorporar:

```text
logical database
tenant
shard
partition domain
security domain
```

según configuración.

---

# 131. Tenant identity

Cache key y metadata deberán evitar colisiones cross-tenant.

---

# 132. Shard identity

Igualmente:

```text
Shard A User#42
≠
Shard B User#42
```

salvo identidad global explícita.

---

# 133. Partition movement

Durante resharding:

```text
Shard 3 → Shard 8
```

un entry del antiguo domain puede volverse incompatible aunque su payload siga representando el mismo valor.

---

# 134. Shard map generation

Podrá utilizarse:

```text
ShardMapGeneration
```

como evidence.

---

# 135. Topology change

No todo cambio de endpoint invalida datos.

Por tanto:

```text
Endpoint changed
≠
Cache invalidation automatically
```

---

# 136. Failover consistency

Failover puede producir escenarios:

```text
new primary fully caught up
new primary slightly behind
unknown promotion point
```

---

# 137. Known safe promotion

Si se demuestra continuidad:

```text
cache may remain valid
```

---

# 138. Unknown promotion

Si la authoritative state es incierta:

```text
cache consistency may become UNKNOWN
```

---

# 139. Conservative failover policy

Podrá:

```text
bump persistence-domain generation
```

cuando no pueda demostrarse continuidad.

---

# 140. Cache-ahead condition

Supongamos:

```text
Old Primary:
version 20

Promoted Replica:
version 18

Cache:
version 20
```

Aquí el cache no está stale-behind.

Está:

```text
ahead of current authoritative database
```

---

# 141. Cache-ahead ≠ valid

No deberá utilizarse automáticamente.

Puede representar writes perdidos/no presentes en el nuevo authority.

---

# 142. Authority epoch

Una solución:

```text
AuthorityEpoch
```

---

# 143. Cache authority metadata

```text
CacheEntry:
    authorityEpoch = 7
```

Después de failover incierto:

```text
CurrentAuthorityEpoch = 8
```

Entonces:

```text
7 != 8
→ incompatible/unknown
```

---

# 144. CacheGenerationVector

Una entrada puede depender de múltiples generaciones:

```text
Metadata = 19
Schema = 42
EntityRegion = 8
Tenant = 7
ShardMap = 15
AuthorityEpoch = 3
```

---

# 145. Vector validation

Formalmente:

```text
Valid(E)
=
∀ g ∈ RequiredGenerations(E):
    Compatible(
        CachedGeneration(g),
        CurrentGeneration(g)
    )
```

---

# 146. Vector size

Generation vectors deberán ser bounded.

---

# 147. Dependency compaction

Queries con miles de dependencies podrán depender de:

```text
EntityTypeGeneration
```

en vez de miles de entity generations.

---

# 148. Precision tradeoff

```text
fine-grained generation
→ higher hit ratio
→ more metadata

coarse generation
→ simpler
→ more invalidations
```

---

# 149. Stale-while-revalidate

VoltStack podrá soportar:

```text
STALE_WHILE_REVALIDATE
```

solo para policies que permitan servir stale.

---

# 150. SWR pipeline

```text
Entry stale but within stale window
        ↓
Serve stale
        ↓
Acquire refresh coordination
        ↓
Refresh asynchronously/synchronously
        ↓
Replace if safe
```

---

# 151. SWR ≠ strict

Una operación `STRICT` no deberá degradarse silenciosamente a SWR.

---

# 152. Stale window

```php
final readonly class StaleWhileRevalidatePolicy
{
    public function __construct(
        public Duration $freshFor,
        public Duration $staleFor,
    ) {}
}
```

---

# 153. SWR state

```text
0 ───────── freshFor ───────── staleFor ──────>
    FRESH                 STALE_ALLOWED    DEAD
```

---

# 154. Background refresh

En runtimes persistentes puede utilizarse infraestructura de concurrencia.

Pero:

```text
background task
```

deberá mantener:

- tenant context;
- shard context;
- security context;
- cancellation;
- resource limits.

---

# 155. No context capture leaks

Una background refresh no deberá retener un request object completo accidentalmente.

---

# 156. Stampede protection

Múltiples misses:

```text
1000 requests
→ same key expired
```

no deberían necesariamente generar:

```text
1000 DB queries
```

---

# 157. Single-flight

VoltStack podrá ofrecer:

```text
CacheRefreshCoordinator
```

---

# 158. Cache lock ≠ DB lock

Regla:

```text
CacheRefreshLock
≠
Pessimistic Database Lock
≠
Transaction
```

---

# 159. Refresh ownership

Solo coordina quién recalcula el entry.

No protege invariantes de negocio.

---

# 160. Refresh timeout

Si el owner falla:

```text
lock lease expires
```

y otro actor podrá refrescar.

---

# 161. Fencing

Para backends distribuidos, podrá utilizarse:

```text
fencing token
```

para evitar que un refresh antiguo sobrescriba uno más nuevo.

---

# 162. Example race

```text
Worker A refresh starts
Worker A stalls

Worker B refresh starts later
Worker B stores version 20

Worker A resumes
Worker A tries to store version 19
```

Sin protección:

```text
20 → 19
```

---

# 163. Monotonic population

Prohibido cuando versions son comparables:

```text
new cache value
<
existing accepted cache version
```

---

# 164. Compare-and-set

Backends con capability adecuada podrán utilizar:

```text
CAS
```

o fencing tokens.

---

# 165. Backend without CAS

Se utilizarán alternativas según consistency policy.

Si no puede garantizarse la policy:

```text
do not store
```

será válido.

---

# 166. Negative caching

VoltStack podrá cachear:

```text
not found
```

de forma explícita.

---

# 167. Negative entry

```text
User#42 = ABSENT
```

es un dato cacheado con dependencies.

---

# 168. Absence can become stale

Después:

```text
INSERT User#42
```

el negative entry deberá invalidarse.

---

# 169. Negative TTL

Normalmente deberá ser bounded y posiblemente menor que positive TTL.

---

# 170. Negative consistency

`ABSENT` deberá pasar las mismas validaciones de dominio/generation necesarias.

---

# 171. Tombstones

Entity Cache podrá utilizar tombstones para:

```text
recent deletion
```

cuando ayuden a evitar resurrection races.

---

# 172. Tombstone ≠ permanent absence

Tendrá lifecycle y version/generation explícitos.

---

# 173. Deletion version

Cuando exista:

```text
delete version 21
```

un cache population de version 20 no podrá revivir la entidad.

---

# 174. Result cache pagination

Una página cacheada:

```text
users page 3
```

puede volverse stale por:

```text
insert
delete
sort-field update
filter-field update
```

aunque ninguno de los IDs de la página haya cambiado.

---

# 175. Therefore

Result Cache consistency dependerá de:

```text
query semantic dependencies
```

no solo de IDs presentes en payload.

---

# 176. Aggregate consistency

```text
COUNT
SUM
AVG
MIN
MAX
```

requieren dependencies sobre los datos que determinan el aggregate.

---

# 177. Empty result

```text
[]
```

también es un resultado.

No significa:

```text
no dependency
```

---

# 178. Empty result invalidation

Un INSERT futuro puede cambiarlo.

---

# 179. Query Cache vs Result Cache

Recordatorio:

```text
Query Cache
=
reusable query artifact
```

```text
Result Cache
=
executed data result
```

No compartirán invalidation rules automáticamente.

---

# 180. Application Cache

Un cache creado por la aplicación:

```php
Cache::remember('homepage', ...)
```

no forma automáticamente parte de Database Cache Consistency.

---

# 181. Integration

VoltStack podrá permitir que Application Cache registre:

```text
DatabaseCacheDependency
```

explícitamente.

---

# 182. No magical dependency discovery

Core no intentará inferir arbitrariamente todas las dependencies dentro de closures de aplicación.

---

# 183. Cache bypass

API conceptual:

```php
User::query()
    ->withoutDatabaseCache()
    ->find(42);
```

---

# 184. Force refresh

```php
User::query()
    ->refreshDatabaseCache()
    ->find(42);
```

---

# 185. Consistency API

Conceptualmente:

```php
User::query()
    ->cacheConsistency(
        CacheConsistency::readYourWrites()
    )
    ->find(42);
```

---

# 186. Bounded staleness API

```php
Product::query()
    ->cacheConsistency(
        CacheConsistency::bounded(
            maxAge: Duration::seconds(30)
        )
    )
    ->get();
```

---

# 187. Strict API

```php
Account::query()
    ->cacheConsistency(
        CacheConsistency::strict()
    )
    ->find($id);
```

---

# 188. Default policy

El default deberá ser configurable por cache type/domain.

No deberá utilizarse una policy relajada universal solo por rendimiento.

---

# 189. Policy resolution

```text
Operation Override
      ↓
Repository/Entity Policy
      ↓
Cache Region Policy
      ↓
Database Policy
      ↓
Framework Default
```

---

# 190. Most-specific wins

Siempre que no viole restricciones obligatorias superiores.

---

# 191. Security floor

Por ejemplo, una operación no podrá relajar:

```text
tenant isolation
```

mediante una cache policy local.

---

# 192. Transaction floor

Tampoco podrá ignorar transaction visibility requirements mediante:

```text
->eventual()
```

si eso rompe la semántica activa.

---

# 193. Consistency policy resolver

```php
interface CacheConsistencyPolicyResolver
{
    public function resolve(
        DatabaseOperation $operation,
        CacheConsistencyContext $context,
    ): CacheConsistencyPolicy;
}
```

---

# 194. CacheConsistencyContext

```php
final readonly class CacheConsistencyContext
{
    public function __construct(
        public PersistenceDomain $domain,
        public ?TransactionContext $transaction,
        public ?WriteConsistencyToken $writeToken,
        public ?ObservationToken $observation,
        public RuntimeScopeId $scope,
    ) {}
}
```

---

# 195. Context ≠ service container

No deberá almacenar:

```text
Container
Request
Response
Credentials
```

---

# 196. Scoped mutable state

Observation/write tokens pertenecerán al scope correspondiente.

---

# 197. Shared immutable state

Podrán compartirse:

```text
compiled policies
metadata definitions
generation descriptors
capability definitions
```

---

# 198. Persistent runtime

FrankenPHP será el runtime principal de VoltStack.

Por tanto:

```text
worker lifetime
≠
request lifetime
```

---

# 199. Worker-local cache

Puede mejorar rendimiento:

```text
Shared Cache
      ↓
Worker L1
      ↓
Request
```

pero crea otra capa de consistencia.

---

# 200. L0/L1/L2 conceptual model

```text
L0
Operation / Request Local

L1
Worker / Process Local

L2
Shared Cache Provider
```

Los nombres podrán variar en implementación.

---

# 201. L0

Puede contener datos scoped con semántica local.

---

# 202. L1

Debe:

- ser bounded;
- ser concurrency-safe;
- observar generations/invalidation;
- no almacenar managed request objects;
- no filtrar tenant context.

---

# 203. L2

Puede ser:

```text
Redis
Memcached
database-backed provider
custom provider
```

mediante contratos.

---

# 204. Core independence

`Quantum/Database` no dependerá obligatoriamente de Redis.

---

# 205. Local stale window

L1 no podrá asumir:

```text
because L2 entry was once valid,
local copy is forever valid
```

---

# 206. Generation check

L1 podrá usar:

```text
local generation snapshot
```

contra:

```text
current generation evidence
```

---

# 207. Event-driven L1

También podrá recibir invalidation events.

---

# 208. Missed event

Si delivery no es durable, una policy estricta deberá contar con mecanismo adicional.

---

# 209. Worker reset

Al reset de worker/request deberán eliminarse todos los estados scoped.

---

# 210. Static mutable cache

Prohibido para información dependiente del request:

```php
private static array $entities = [];
```

---

# 211. Coroutine isolation

OpenSwoole requiere:

```text
Coroutine A context
≠
Coroutine B context
```

---

# 212. Fiber isolation

Lo mismo para fibers concurrentes.

---

# 213. FrankenPHP worker safety

El cache local podrá sobrevivir requests solo si su contenido está diseñado explícitamente para ello.

---

# 214. No managed object retention

Nunca mantener:

```text
EntityManager
UnitOfWork
managed Entity
TransactionContext
TenantContext
```

en shared worker cache.

---

# 215. Cache backend failure

Un cache puede estar:

```text
unavailable
timed out
partitioned
partially available
```

---

# 216. Cache failure ≠ database failure

Por defecto:

```text
cache unavailable
→ bypass cache
→ database
```

cuando sea seguro.

---

# 217. Failure policy

```php
enum CacheFailurePolicy
{
    case BYPASS_TO_DATABASE;
    case SERVE_STALE_IF_ALLOWED;
    case FAIL_OPERATION;
    case CACHE_ONLY_MISS;
}
```

---

# 218. Metadata cache failure

Si metadata puede recompilarse:

```text
cache failure
→ recompute
```

No utilizar stale incompatible metadata solo para evitar el costo.

---

# 219. Result cache failure

Normalmente:

```text
bypass to DB
```

---

# 220. Entity cache failure

Normalmente:

```text
bypass to ORM/database
```

---

# 221. Cache-only mode

Aquí:

```text
backend unavailable
```

puede requerir error explícito.

---

# 222. Failure transparency

No esconder:

```text
cache consistency degraded
```

en telemetry.

---

# 223. Partial cache outage

En un cluster:

```text
Node A sees cache
Node B cannot
```

ambos pueden continuar si DB fallback es válido, pero tendrán perfiles de rendimiento distintos.

---

# 224. Split cache cluster

Más peligroso:

```text
Node A writes cache partition A
Node B reads partition B
```

---

# 225. Backend consistency capability

El provider deberá declarar garantías relevantes.

---

# 226. CacheProviderCapabilities

```php
interface CacheProviderCapabilities
{
    public function supportsAtomicWrite(): bool;

    public function supportsCompareAndSet(): bool;

    public function supportsAtomicGeneration(): bool;

    public function supportsDistributedLock(): bool;

    public function supportsDurableInvalidation(): bool;

    public function consistencyModel(): CacheProviderConsistencyModel;
}
```

---

# 227. Capability-driven

La policy efectiva deberá considerar estas capabilities.

---

# 228. Provider limitation

Si una policy requiere:

```text
atomic generation advancement
```

y el provider no lo soporta:

```text
do not pretend support
```

---

# 229. Degradation

Podrá:

```text
bypass cache
```

en vez de debilitar semantics.

---

# 230. CacheConsistencyDegradation

```php
final readonly class CacheConsistencyDegradation
{
    public function __construct(
        public CacheConsistencyProfile $requested,
        public CacheConsistencyProfile $effective,
        public CacheConsistencyDegradationReason $reason,
    ) {}
}
```

---

# 231. Silent degradation

Prohibida para políticas donde cambie la garantía observable.

---

# 232. Explicit relaxation

La aplicación podrá permitir:

```text
STRICT preferred
EVENTUAL fallback
```

solo mediante policy explícita.

---

# 233. Resource governance

Consistency no deberá crear:

```text
unbounded version maps
unbounded dependency vectors
unbounded stale payloads
unbounded refresh tasks
```

---

# 234. Limits

Configurable:

```text
max cache entry bytes
max dependency count
max generation vector size
max stale age
max refresh concurrency
max refresh wait
max local entries
max local bytes
max negative TTL
```

---

# 235. Oversized result

Un resultado demasiado grande podrá ser:

```text
NON_CACHEABLE
```

---

# 236. Streaming results

Por defecto:

```text
StreamingResult
```

no deberá materializarse completamente solo para cachearlo.

---

# 237. Explicit streaming cache

Una extensión futura podría implementar:

```text
chunk cache
```

pero será una arquitectura distinta.

---

# 238. Serialization consistency

El payload deberá codificarse de forma determinista cuando fingerprints/version validation dependan de ello.

---

# 239. Payload corruption

Si checksum/decoder detecta corrupción:

```text
CORRUPT
→ treat as unusable
→ evict/invalidate
→ fallback
```

---

# 240. Corruption ≠ cache miss

Telemetry deberá distinguirlos.

---

# 241. Encryption

Datos sensibles cacheados podrán requerir cifrado según policy.

---

# 242. Encryption ≠ consistency

Cifrado protege confidencialidad.

No demuestra actualidad.

---

# 243. Authorization

Cache hit no bypassará Authorization.

---

# 244. Database cache ≠ access-control cache

Si una query normalmente está limitada por security context, dicho contexto deberá formar parte de la identidad/dependency cuando corresponda.

---

# 245. Security-sensitive result

No reutilizar:

```text
Admin-visible result
```

para:

```text
normal user
```

por compartir query fingerprint insuficiente.

---

# 246. Cache identity

La key podrá incluir:

```text
semantic query fingerprint
persistence domain
security scope fingerprint
tenant
shard
result shape
policy-relevant dimensions
```

---

# 247. Security fingerprint

No deberá incluir secrets raw.

---

# 248. Authorization changes

Si los permisos afectan el contenido materializado:

```text
authorization generation
```

podrá formar parte de dependencies.

---

# 249. Authentication identity

Solo deberá formar parte de key cuando semánticamente cambie el resultado.

---

# 250. No accidental per-user explosion

No incluir `user_id` automáticamente si el resultado no depende de él.

---

# 251. Consistency telemetry

Eventos conceptuales:

```text
CacheConsistencyEvaluated
CacheEntryAccepted
CacheEntryRejected
CacheEntryStaleServed
CacheEntryRefreshRequested
CacheEntryRefreshCompleted
CacheConsistencyDegraded
CacheConsistencyUnknown
CacheRepopulationRejected
```

---

# 252. Metrics

```text
db.cache.consistency.evaluations
db.cache.consistency.accepted
db.cache.consistency.rejected
db.cache.consistency.stale_served
db.cache.consistency.unknown
db.cache.consistency.degraded
db.cache.consistency.refreshes
db.cache.consistency.repopulation_rejected
```

---

# 253. Rejection reasons

Bounded labels:

```text
expired
generation_mismatch
version_too_old
transaction_active
read_your_writes
topology_changed
domain_mismatch
unknown_evidence
provider_failure
payload_incompatible
```

---

# 254. No raw keys

No usar:

```text
cache_key
entity_id
tenant_id
query_parameter
```

como labels sin control.

---

# 255. Tracing

Span:

```text
database.cache.consistency
```

podrá incluir:

```text
cache kind
requested profile
effective profile
decision
reason
```

---

# 256. Diagnostics

API conceptual:

```php
DB::cache()
    ->consistency()
    ->explain($query);
```

---

# 257. Diagnostic output

```text
CACHE CONSISTENCY ANALYSIS

Cache:
    RESULT

Requested:
    READ_YOUR_WRITES

Physical Entry:
    HIT

Entry Generation:
    41

Current Generation:
    41

Entry Version:
    17

Minimum Required Version:
    18

Transaction:
    NONE

Sticky Write Token:
    PRESENT

Decision:
    BYPASS_CACHE

Reason:
    ENTRY_BEHIND_REQUIRED_WRITE_VERSION

Next Source:
    WRITER
```

---

# 258. Another diagnostic

```text
CACHE CONSISTENCY ANALYSIS

Requested:
    BOUNDED_STALENESS

Max Age:
    30 seconds

Entry Age:
    12 seconds

Generation:
    CURRENT

Domain:
    MATCH

Invalidation:
    NONE OBSERVED

Decision:
    USE_CACHE

Guarantee:
    BOUNDED_TEMPORAL_STALENESS

Warning:
    TEMPORAL AGE DOES NOT PROVE LATEST DATABASE VERSION
```

---

# 259. Explainability

Toda rejection significativa deberá poder responder:

```text
Why was this cache entry rejected?
```

---

# 260. Consistency report

```php
interface CacheConsistencyInspector
{
    public function inspect(
        CacheEntryMetadata $entry,
        CacheConsistencyContext $context,
    ): CacheConsistencyReport;
}
```

---

# 261. CLI

Posibles comandos:

```bash
php voltstack database:cache:consistency
php voltstack database:cache:inspect <key>
php voltstack database:cache:generations
php voltstack database:cache:consistency:diagnose
php voltstack database:cache:consistency:health
```

---

# 262. Testing architecture

La suite deberá probar al menos:

```text
TTL
generation
version
read-your-writes
monotonic reads
transactions
rollback
unknown commit
replica lag
stale repopulation
failover
sharding
multitenancy
SWR
stampede
provider failure
persistent workers
security
serialization
resource limits
```

---

# 263. Fresh entry test

```text
generation current
version sufficient
domain match
→ USE_CACHE
```

---

# 264. Generation mismatch test

```text
cached G17
current G18
→ reject
```

---

# 265. TTL-only test

Entry dentro de TTL pero invalidated:

```text
→ reject
```

---

# 266. Expired but strict test

```text
→ refresh/bypass
```

---

# 267. SWR test

```text
expired from fresh window
within stale window
policy allows stale
→ serve stale + refresh
```

---

# 268. Strict SWR test

```text
STRICT
+
stale
→ do not serve stale
```

---

# 269. Read-your-writes test

```text
write version 18
cache version 17
→ reject
```

---

# 270. Monotonic read test

```text
observed 20
candidate 19
→ reject
```

---

# 271. Transaction test

Shared result cache shall be bypassed where transaction policy requires it.

---

# 272. Rollback test

Uncommitted cache publication deberá desaparecer.

---

# 273. UNKNOWN commit test

No speculative new cache state.

---

# 274. Replica repopulation test

```text
required 18
replica candidate 17
→ do not store shared
```

---

# 275. Failover epoch test

```text
entry epoch 4
authority epoch 5
→ reject/unknown
```

---

# 276. Tenant test

```text
Tenant A entry
Tenant B request
→ reject
```

---

# 277. Shard test

```text
Shard 3 entry
Shard 4 request
→ reject
```

---

# 278. Metadata test

Old mapping generation:

```text
→ reject
```

---

# 279. Query cache test

Row mutation should not invalidate structurally compatible query artifact.

---

# 280. Result cache test

Relevant row mutation shall invalidate dependent result.

---

# 281. Entity cache test

Cache hit must reuse current IdentityMap identity.

---

# 282. Dirty entity test

Cache shall not overwrite dirty managed entity.

---

# 283. Negative cache test

INSERT invalidates cached absence.

---

# 284. Deletion resurrection test

Older population cannot overwrite newer tombstone/version.

---

# 285. Stampede test

100 concurrent misses should respect refresh coordination.

---

# 286. Fencing test

Older refresh cannot overwrite newer accepted value.

---

# 287. Provider outage test

Fallback behavior must match failure policy.

---

# 288. Persistent worker test

No request/tenant state leakage.

---

# 289. Coroutine test

No consistency token leakage across coroutines.

---

# 290. Corruption test

Corrupt payload rejected and observable.

---

# 291. Resource test

Oversized dependency vectors are compacted/rejected according to policy.

---

# 292. Security test

Different authorization scopes never share incompatible result entries.

---

# 293. Performance testing

Medir:

```text
consistency evaluation latency
generation lookup cost
version comparison cost
L1 validation
L2 validation
SWR overhead
single-flight contention
repopulation checks
```

---

# 294. Directory structure

```text
src/Quantum/Database/Cache/Consistency/
│
├── CacheConsistencyManager.php
├── CacheConsistencyEvaluator.php
├── CacheConsistencyPolicy.php
├── CacheConsistencyPolicyResolver.php
├── CacheConsistencyProfile.php
├── CacheConsistencyRequirement.php
├── CacheConsistencyState.php
├── CacheConsistencyDecision.php
├── CacheReadAction.php
├── CacheConsistencyContext.php
│
├── Evidence/
│   ├── CacheConsistencyEvidence.php
│   ├── CacheConsistencyEvidenceSet.php
│   ├── GenerationEvidence.php
│   ├── EntityVersionEvidence.php
│   ├── TransactionEvidence.php
│   ├── CommitPositionEvidence.php
│   ├── ReplicationPositionEvidence.php
│   ├── TopologyGenerationEvidence.php
│   ├── AuthorityEpochEvidence.php
│   ├── SchemaGenerationEvidence.php
│   ├── MetadataGenerationEvidence.php
│   └── InvalidationEvidence.php
│
├── Requirement/
│   ├── FreshnessRequirement.php
│   ├── ReadYourWritesRequirement.php
│   ├── MonotonicReadRequirement.php
│   ├── VersionRequirement.php
│   └── TransactionVisibilityRequirement.php
│
├── Version/
│   ├── DataVersion.php
│   ├── MinimumRequiredVersion.php
│   ├── WriteConsistencyToken.php
│   ├── ObservationToken.php
│   ├── VersionComparator.php
│   └── AuthorityEpoch.php
│
├── Provenance/
│   ├── CacheDataProvenance.php
│   ├── ReplicationPosition.php
│   └── CacheSourceDescriptor.php
│
├── Population/
│   ├── CachePopulationEvaluator.php
│   ├── CachePopulationDecision.php
│   ├── CachePopulationGuard.php
│   └── MonotonicCacheWriter.php
│
├── Refresh/
│   ├── CacheRefreshCoordinator.php
│   ├── CacheRefreshLease.php
│   ├── CacheRefreshFencingToken.php
│   ├── StaleWhileRevalidatePolicy.php
│   └── CacheRefreshScheduler.php
│
├── Failure/
│   ├── CacheFailurePolicy.php
│   ├── CacheConsistencyDegradation.php
│   ├── CacheConsistencyDegradationReason.php
│   └── CacheProviderFailureHandler.php
│
├── Provider/
│   ├── CacheProviderCapabilities.php
│   ├── CacheProviderConsistencyModel.php
│   └── CacheConsistencyCapabilityResolver.php
│
├── Telemetry/
│   ├── CacheConsistencyTelemetry.php
│   ├── CacheConsistencyEvaluatedEvent.php
│   ├── CacheEntryRejectedEvent.php
│   ├── CacheConsistencyDegradedEvent.php
│   └── CacheRepopulationRejectedEvent.php
│
├── Diagnostics/
│   ├── CacheConsistencyInspector.php
│   ├── CacheConsistencyExplainer.php
│   └── CacheConsistencyReport.php
│
└── Exception/
    ├── CacheConsistencyException.php
    ├── CacheConsistencyViolationException.php
    ├── CacheConsistencyUnknownException.php
    ├── CacheGenerationMismatchException.php
    ├── CacheVersionMismatchException.php
    ├── CacheDomainMismatchException.php
    ├── CachePayloadCompatibilityException.php
    └── CacheConsistencyCapabilityException.php
```

---

# 295. Architectural invariants

## DB-CCS-001
Cache existence no implicará validez.

## DB-CCS-002
Cache validity no implicará actualidad absoluta.

## DB-CCS-003
Cache freshness no será equivalente a consistency.

## DB-CCS-004
TTL no será prueba de correctness.

## DB-CCS-005
Expiration será distinta de invalidation.

## DB-CCS-006
Eviction será distinta de invalidation.

## DB-CCS-007
Staleness será distinta de expiration.

## DB-CCS-008
UNKNOWN no será equivalente a CURRENT.

## DB-CCS-009
UNKNOWN no será equivalente a STALE.

## DB-CCS-010
Policies estrictas rechazarán evidencia insuficiente.

## DB-CCS-011
Consistency será evaluada por operación.

## DB-CCS-012
Cache kinds podrán tener policies distintas.

## DB-CCS-013
STRICT tendrá garantías explícitas.

## DB-CCS-014
READ_YOUR_WRITES impedirá observar una versión anterior a una escritura confirmada del scope.

## DB-CCS-015
BOUNDED_STALENESS tendrá límites explícitos.

## DB-CCS-016
EVENTUAL no significará unlimited stale.

## DB-CCS-017
CACHE_ONLY será explícito.

## DB-CCS-018
Strong consistency no será término ambiguo en APIs.

## DB-CCS-019
Consistency decisions utilizarán evidence.

## DB-CCS-020
Una evidencia individual no será necesariamente suficiente.

## DB-CCS-021
CacheConsistencyDecision será explícita.

## DB-CCS-022
USE_CACHE será distinto de physical cache hit.

## DB-CCS-023
Physical hit será distinto de usable hit.

## DB-CCS-024
Rejected hits serán observables.

## DB-CCS-025
Cache metadata será distinta de payload.

## DB-CCS-026
Version tokens serán domain-aware.

## DB-CCS-027
Version tokens incompatibles no serán comparados.

## DB-CCS-028
Vendor replication positions serán encapsuladas.

## DB-CCS-029
Core no dependerá de PostgreSQL LSN/MySQL GTID concretos.

## DB-CCS-030
Read-your-writes utilizará write evidence cuando exista.

## DB-CCS-031
Sticky routing y cache consistency cooperarán.

## DB-CCS-032
Sticky writer selection no validará automáticamente cache entries.

## DB-CCS-033
Cache consistency se evaluará antes de aceptar un hit.

## DB-CCS-034
Active transactions podrán forzar bypass de shared cache.

## DB-CCS-035
Uncommitted data nunca será publicado en shared cache.

## DB-CCS-036
Transaction-local cache será scoped.

## DB-CCS-037
Transaction-local cache será distinta de shared cache.

## DB-CCS-038
Commit confirmado podrá publicar cache state.

## DB-CCS-039
Rollback descartará pending cache publication.

## DB-CCS-040
Rollback no implicará PHP object graph rewind.

## DB-CCS-041
UNKNOWN transaction outcome seguirá siendo UNKNOWN.

## DB-CCS-042
UNKNOWN no publicará speculative new cache values.

## DB-CCS-043
UNKNOWN podrá provocar conservative invalidation.

## DB-CCS-044
Replica provenance podrá formar parte de evidence.

## DB-CCS-045
Replica provenance no será garantía perpetua.

## DB-CCS-046
Stale replica repopulation será protegida.

## DB-CCS-047
Readable será distinto de share-cacheable.

## DB-CCS-048
Cache population tendrá decisión independiente.

## DB-CCS-049
Monotonic reads podrán mantener observation tokens.

## DB-CCS-050
Observation state será scoped.

## DB-CCS-051
No existirá global mutable last-seen state.

## DB-CCS-052
Temporal staleness será explícita.

## DB-CCS-053
Temporal age no equivaldrá a version distance.

## DB-CCS-054
Multiple consistency bounds podrán combinarse.

## DB-CCS-055
EVENTUAL seguirá respetando domain isolation.

## DB-CCS-056
EVENTUAL seguirá respetando payload compatibility.

## DB-CCS-057
Stale metadata incompatible no será servida.

## DB-CCS-058
Query Cache tendrá consistency estructural.

## DB-CCS-059
Result Cache tendrá consistency de datos.

## DB-CCS-060
Entity Cache tendrá consistency de estado persistente.

## DB-CCS-061
IdentityMap seguirá siendo autoridad de object identity dentro del scope.

## DB-CCS-062
Entity Cache no compartirá managed objects.

## DB-CCS-063
Cache hit no sobrescribirá dirty managed entities.

## DB-CCS-064
Hydration Cache será estructural.

## DB-CCS-065
Compiled Query Cache será estructural.

## DB-CCS-066
Structural caches y data caches tendrán reglas distintas.

## DB-CCS-067
Generation mismatch será incompatible salvo compatibility explícita.

## DB-CCS-068
Payload version será explícita.

## DB-CCS-069
Unsafe arbitrary object deserialization estará prohibida.

## DB-CCS-070
Tenant mismatch será inconsistente.

## DB-CCS-071
Shard mismatch será inconsistente.

## DB-CCS-072
PersistenceDomain participará en cache identity.

## DB-CCS-073
Resharding podrá invalidar domain compatibility.

## DB-CCS-074
Endpoint change no invalidará automáticamente data cache.

## DB-CCS-075
Failover continuity se basará en evidence.

## DB-CCS-076
Unknown failover podrá degradar cache state a UNKNOWN.

## DB-CCS-077
Cache-ahead no será considerado automáticamente válido.

## DB-CCS-078
Authority epochs podrán proteger failover consistency.

## DB-CCS-079
Generation vectors serán soportados.

## DB-CCS-080
Generation vectors serán bounded.

## DB-CCS-081
Dependency compaction será permitida.

## DB-CCS-082
Compaction no romperá correctness.

## DB-CCS-083
SWR requerirá policy que permita stale.

## DB-CCS-084
STRICT no degradará silenciosamente a SWR.

## DB-CCS-085
Background refresh conservará execution domain.

## DB-CCS-086
Background refresh no retendrá request graphs innecesarios.

## DB-CCS-087
Stampede protection será coordinación de cache.

## DB-CCS-088
Cache refresh lock no será DB lock.

## DB-CCS-089
Cache refresh lock no será transaction.

## DB-CCS-090
Refresh leases serán bounded.

## DB-CCS-091
Fencing podrá impedir stale overwrite.

## DB-CCS-092
Older refresh no sobrescribirá newer accepted value cuando exista ordering.

## DB-CCS-093
CAS será capability-driven.

## DB-CCS-094
Falta de CAS no se ocultará si la policy lo requiere.

## DB-CCS-095
Negative caching será explícito.

## DB-CCS-096
Cached absence tendrá dependencies.

## DB-CCS-097
INSERT podrá invalidar negative entries.

## DB-CCS-098
Tombstone será distinto de permanent absence.

## DB-CCS-099
Older population no resucitará deleted entity cuando exista version evidence.

## DB-CCS-100
Paginated result dependencies no se limitarán a IDs presentes.

## DB-CCS-101
Aggregate caches tendrán semantic dependencies.

## DB-CCS-102
Empty result seguirá teniendo dependencies.

## DB-CCS-103
Query Cache será distinto de Result Cache.

## DB-CCS-104
Application Cache no quedará automáticamente bajo Database Cache Consistency.

## DB-CCS-105
Application Cache podrá registrar dependencies explícitas.

## DB-CCS-106
Core no hará magical closure dependency discovery.

## DB-CCS-107
Cache bypass será explícito.

## DB-CCS-108
Force refresh será explícito.

## DB-CCS-109
Consistency override será explícito.

## DB-CCS-110
Default policy será configurable.

## DB-CCS-111
Policy resolution será determinista.

## DB-CCS-112
Security isolation no podrá relajarse por local cache policy.

## DB-CCS-113
Transaction requirements no podrán relajarse inseguramente.

## DB-CCS-114
ConsistencyContext será scoped.

## DB-CCS-115
ConsistencyContext no contendrá service container.

## DB-CCS-116
Persistent worker no equivaldrá a request.

## DB-CCS-117
L0/L1/L2 tendrán lifecycle explícito.

## DB-CCS-118
Worker-local cache será bounded.

## DB-CCS-119
Worker-local cache observará invalidation/generations.

## DB-CCS-120
Worker-local cache no almacenará managed request objects.

## DB-CCS-121
Core no dependerá obligatoriamente de Redis.

## DB-CCS-122
Missed local invalidation events serán considerados en strong policies.

## DB-CCS-123
Worker reset eliminará scoped consistency state.

## DB-CCS-124
Static request-dependent caches estarán prohibidas.

## DB-CCS-125
Coroutine state permanecerá aislado.

## DB-CCS-126
Fiber state permanecerá aislado.

## DB-CCS-127
FrankenPHP worker-local reuse será explícitamente safe.

## DB-CCS-128
EntityManager no será almacenado en shared cache.

## DB-CCS-129
UnitOfWork no será almacenado en shared cache.

## DB-CCS-130
TransactionContext no será almacenado en shared cache.

## DB-CCS-131
Cache failure será distinto de DB failure.

## DB-CCS-132
Cache failure normalmente podrá bypassar a DB.

## DB-CCS-133
Metadata incompatible no será utilizada por fallback.

## DB-CCS-134
CACHE_ONLY failure tendrá semántica explícita.

## DB-CCS-135
Cache degradation será observable.

## DB-CCS-136
Provider capabilities serán explícitas.

## DB-CCS-137
Provider consistency model será explícito.

## DB-CCS-138
Capability insufficiency no será ocultada.

## DB-CCS-139
Bypass será preferible a false consistency.

## DB-CCS-140
Silent consistency degradation estará prohibida.

## DB-CCS-141
Explicit fallback policy podrá relajar garantías.

## DB-CCS-142
Resource governance será obligatoria.

## DB-CCS-143
Dependency vectors no crecerán sin límites.

## DB-CCS-144
Refresh concurrency será bounded.

## DB-CCS-145
Oversized results podrán ser non-cacheable.

## DB-CCS-146
Streaming results no se materializarán automáticamente para cache.

## DB-CCS-147
Corrupt payload será unusable.

## DB-CCS-148
Corruption será distinta de miss.

## DB-CCS-149
Encryption será distinta de consistency.

## DB-CCS-150
Cache hit nunca bypassará authorization.

## DB-CCS-151
Security scope podrá formar parte de cache identity.

## DB-CCS-152
Secrets raw no formarán parte de keys.

## DB-CCS-153
Authorization-dependent cache tendrá dependencies adecuadas.

## DB-CCS-154
Telemetry distinguirá physical y usable hits.

## DB-CCS-155
Telemetry evitará raw high-cardinality keys.

## DB-CCS-156
Consistency decisions serán explainable.

## DB-CCS-157
Diagnostics mostrarán requested y effective policy.

## DB-CCS-158
Diagnostics mostrarán evidence.

## DB-CCS-159
Cache consistency no generará SQL.

## DB-CCS-160
Cache consistency no ejecutará transactions.

## DB-CCS-161
Cache consistency no decidirá ORM object identity.

## DB-CCS-162
Cache consistency no sustituirá database constraints.

## DB-CCS-163
Cache consistency no sustituirá transaction isolation.

## DB-CCS-164
Cache consistency no sustituirá optimistic locking.

## DB-CCS-165
Cache consistency no sustituirá pessimistic locking.

## DB-CCS-166
Cache consistency no será un distributed transaction protocol.

## DB-CCS-167
Database truth continuará siendo autoridad persistente.

## DB-CCS-168
Cache será una representación derivada y reutilizable.

## DB-CCS-169
VoltStack no afirmará CURRENT sin evidencia suficiente.

## DB-CCS-170
Cuando la evidencia sea insuficiente, correctness tendrá prioridad sobre cache hit ratio.

---

# 296. Modelo formal

Sean:

```text
D(t)
=
authoritative database state at logical time t

C
=
cached artifact

V(C)
=
version represented by C

G(C)
=
generation vector represented by C

R
=
read consistency requirement

W(R)
=
minimum write/version evidence required by R
```

Un cache entry será utilizable si:

```text
Usable(C, R)
=
Exists(C)
∧ DomainCompatible(C, R)
∧ PayloadCompatible(C)
∧ GenerationCompatible(G(C))
∧ FreshnessAllowed(C, R)
∧ VersionRequirementSatisfied(V(C), W(R))
∧ TransactionVisibilitySatisfied(C, R)
∧ SecurityScopeCompatible(C, R)
```

Si cualquiera de estas condiciones es:

```text
FALSE
```

el entry será rechazado.

Si una condición necesaria es:

```text
UNKNOWN
```

entonces:

```text
STRICT
→ REJECT/BYPASS
```

mientras una policy más relajada podrá decidir explícitamente:

```text
SERVE_STALE
```

si su contrato lo permite.

---

# 297. Modelo de observación

VoltStack distinguirá tres realidades:

```text
Database Reality
Cache Knowledge
Application Observation
```

Formalmente:

```text
DatabaseReality
≠
CacheKnowledge
≠
ApplicationObservation
```

La caché puede estar:

```text
behind DB
equal to DB
ahead of current authority after failover
unknown relative to DB
```

---

# 298. Regla de read-your-writes

Si un scope ha confirmado una escritura `W` con versión `v`:

```text
ConfirmedWrite(scope, key, v)
```

entonces cualquier lectura posterior bajo `READ_YOUR_WRITES` deberá satisfacer:

```text
ObservedVersion(scope, key) >= v
```

cuando las versiones sean comparables.

Si no existe evidence suficiente:

```text
bypass cache
```

---

# 299. Regla de monotonic reads

Si:

```text
HighestObserved(scope, key) = v
```

una lectura posterior no deberá devolver:

```text
v' < v
```

bajo una policy monotónica.

---

# 300. Regla de transaction visibility

Para un transaction `T`:

```text
SharedCache
```

no podrá representar sus writes no confirmados como si fueran globalmente visibles.

Por tanto:

```text
Uncommitted(T)
→ NotPublishableToSharedCache
```

---

# 301. Regla de UNKNOWN

Si:

```text
CommitOutcome(T) = UNKNOWN
```

entonces:

```text
NewCacheValue(T)
→ NOT_PUBLISHABLE
```

y los entries posiblemente afectados podrán pasar a:

```text
INVALIDATED
or
UNKNOWN
```

según evidence/policy.

---

# 302. Regla de replica repopulation

Sea:

```text
M
=
minimum required position

P
=
position represented by replica candidate
```

si son comparables:

```text
P < M
→ DO_NOT_POPULATE_SHARED_CACHE
```

---

# 303. Regla de generations

Sea:

```text
GC
=
generation vector stored in cache

GR
=
generation vector required now
```

entonces:

```text
Compatible(GC, GR) = false
→ REJECT
```

---

# 304. Regla de domain isolation

```text
CacheDomain(C)
!=
ReadDomain(R)
→ REJECT
```

sin importar:

```text
TTL
version
payload equality
```

---

# 305. Regla de failover

Si authoritative epoch cambia:

```text
AuthorityEpoch(C)
!=
CurrentAuthorityEpoch
```

el entry deberá reevaluarse bajo la failover compatibility policy.

Sin evidencia de continuidad:

```text
→ UNKNOWN / REJECT
```

---

# 306. Flujo completo

```text
                    APPLICATION READ
                           │
                           ▼
                Database Query / ORM
                           │
                           ▼
              Consistency Requirement
                           │
                           ▼
                     Cache Lookup
                           │
                   ┌───────┴────────┐
                   │                │
                 MISS          PHYSICAL HIT
                   │                │
                   │                ▼
                   │      Consistency Evaluator
                   │                │
                   │       ┌────────┼──────────────┐
                   │       │        │              │
                   │      USE     STALE          REJECT
                   │       │      ALLOWED           │
                   │       │        │               │
                   │       ▼        ▼               │
                   │     RETURN   RETURN +           │
                   │             REFRESH             │
                   │                                 │
                   └────────────────┬────────────────┘
                                    ▼
                           Database Read Path
                                    │
                                    ▼
                          Partition Routing
                                    │
                                    ▼
                          Read/Write Routing
                                    │
                                    ▼
                          Replica Eligibility
                                    │
                                    ▼
                               Execute
                                    │
                                    ▼
                         Candidate Result
                                    │
                                    ▼
                       Population Consistency
                              Evaluation
                                    │
                         ┌──────────┼──────────┐
                         │          │          │
                       SHARE      LOCAL      DON'T
                         │          │          │
                         ▼          ▼          ▼
                        L2         L0/L1      RETURN
                         │          │          │
                         └──────────┴──────────┘
                                    │
                                    ▼
                                  RETURN
```

---

# 307. Integración final del Bloque 17

La arquitectura completa queda:

```text
186_DATABASE_CACHE_ARCHITECTURE
             │
             ├──────────────────────────────┐
             │                              │
             ▼                              ▼
187_QUERY_CACHE                    188_RESULT_CACHE
             │                              │
             ├──────────────┐               │
             │              │               │
             ▼              ▼               ▼
189_METADATA_CACHE     190_ENTITY_CACHE
             │              │
             └───────┬──────┘
                     ▼
          191_CACHE_INVALIDATION
                     │
                     ▼
          192_CACHE_CONSISTENCY
```

Y transversalmente:

```text
Transactions
Read/Write Routing
Replicas
Sticky Reads
Failover
Sharding
Partition Routing
Persistent Runtime
Security
Telemetry
```

---

# 308. Regla maestra final

VoltStack seguirá el principio:

```text
Cache Hit
   ↓
Candidate
   ↓
Validate Evidence
   ↓
Validate Domain
   ↓
Validate Generation
   ↓
Validate Version
   ↓
Validate Transaction Visibility
   ↓
Validate Requested Consistency
   ↓
Use or Reject
```

Nunca:

```text
Cache Hit
   ↓
Return
```

como regla arquitectónica universal.

La garantía final será:

> **La base de datos sigue siendo la autoridad persistente; la caché es una representación derivada. VoltStack solo permitirá que esa representación sustituya temporalmente una lectura de base de datos cuando pueda demostrar —con el nivel de evidencia exigido por la operación— que hacerlo preserva el contrato observable de consistencia.**

Por tanto:

```text
Fast
≠
Correct
```

pero el objetivo de VoltStack será:

```text
Correct
+
Observable
+
Predictable
+
Fast when safely possible
```

---

# 309. Cierre del Bloque 17 — Cache

Con este documento queda definido el subsistema:

```text
VoltStack/Quantum/Database/Cache
```

mediante:

```text
186 Cache Architecture
187 Query Cache
188 Result Cache
189 Metadata Cache
190 Entity Cache
191 Cache Invalidation
192 Cache Consistency
```

La arquitectura resultante establece explícitamente:

```text
Cache
≠
Database Truth

IdentityMap
≠
Entity Cache

Query Cache
≠
Result Cache

Hydration Cache
≠
Entity Cache

TTL
≠
Consistency

Expiration
≠
Invalidation

Invalidation
≠
Consistency

Physical Hit
≠
Usable Hit

Flush
≠
Commit

Rollback
≠
Object Graph Rewind

UNKNOWN
≠
SAFE

Replica Freshness
≠
Cache Freshness

Cache Lock
≠
Database Lock

Worker Lifetime
≠
Request Lifetime
```

Esto deja preparado el Database System para integrar caching de alto rendimiento sin sacrificar las invariantes establecidas por ORM, Transactions, Distribution y Persistent Runtime.

---

# 310. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOQUE 18
FACTORIES, SEEDERS & FIXTURES
```

con:

```text
193_DATABASE_FACTORY_SYSTEM.md
194_DATABASE_MODEL_FACTORY_SYSTEM.md
195_DATABASE_ENTITY_FACTORY_SYSTEM.md
196_DATABASE_SEEDER_SYSTEM.md
197_DATABASE_FIXTURE_SYSTEM.md
198_DATABASE_TEST_DATA_GENERATION_SYSTEM.md
```

---

# 311. Siguiente documento

```text
193_DATABASE_FACTORY_SYSTEM.md
```

Este documento definirá la arquitectura general de generación programática de objetos y datos para:

```text
Models
Entities
Tests
Seeders
Fixtures
Development Environments
Benchmarking
Synthetic Data
Relationship Graphs
Database Population
```

manteniendo una separación estricta:

```text
Factory
≠
Seeder
≠
Fixture
≠
Persistence Engine
≠
Random Data Generator
≠
Entity Hydrator
```