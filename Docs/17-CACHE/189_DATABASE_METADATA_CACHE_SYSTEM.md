# 189_DATABASE_METADATA_CACHE_SYSTEM.md

# VoltStack Quantum Database
## Metadata Cache System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 189 — Metadata Cache System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `188_DATABASE_RESULT_CACHE_SYSTEM.md`  
**Siguiente documento:** `190_DATABASE_ENTITY_CACHE_SYSTEM.md`

---

# 1. Propósito

`Metadata Cache System` define la arquitectura mediante la cual VoltStack reutilizará conocimiento estructural previamente descubierto, normalizado, validado o compilado por `Quantum/Database`.

Su objetivo principal es evitar repetir trabajo costoso como:

```text
Reflection
Attribute scanning
Entity mapping compilation
Relationship mapping compilation
Schema introspection
Type resolution
Accessor compilation
Hydration metadata construction
Platform capability normalization
Extension metadata processing
```

Regla principal:

> **Metadata Cache almacenará conocimiento estructural derivado y reutilizable; nunca almacenará estado transaccional, entidades administradas, resultados de consultas ni información mutable perteneciente a un request, operación o worker execution scope.**

Formalmente:

```text
MetadataCache
=
ReusableStructuralKnowledge
```

y:

```text
MetadataCache
≠
ApplicationDataCache
```

---

# 2. Problema que resuelve

Considérese una entidad:

```php
#[Entity]
#[Table('users')]
final class User
{
    #[Id]
    #[Column(type: 'uuid')]
    private UserId $id;

    #[Column(length: 150)]
    private string $email;

    #[ManyToOne(target: Organization::class)]
    private Organization $organization;
}
```

Sin metadata compilada, VoltStack podría repetir:

```text
inspect class
↓
read attributes
↓
resolve types
↓
resolve columns
↓
resolve relationships
↓
validate mappings
↓
compile accessors
↓
build entity metadata
```

en múltiples requests.

En runtimes persistentes esto sería desperdicio innecesario.

Con Metadata Cache:

```text
Application Boot
      ↓
Metadata Lookup
      │
      ├── HIT
      │     ↓
      │  Compiled Metadata
      │
      └── MISS
            ↓
        Discover
            ↓
        Normalize
            ↓
         Validate
            ↓
         Compile
            ↓
          Cache
```

---

# 3. Distinción fundamental

Debe mantenerse:

```text
Query Cache
≠
Result Cache
≠
Metadata Cache
≠
Entity Cache
≠
IdentityMap
```

| Sistema | Qué almacena |
|---|---|
| Query Cache | artefactos derivados del procesamiento de queries |
| Result Cache | datos devueltos por queries |
| Metadata Cache | conocimiento estructural/compilado |
| Entity Cache | representaciones cacheables orientadas a entidades |
| IdentityMap | objetos administrados dentro del scope ORM |

---

# 4. Ejemplo conceptual

```text
Entity class
   ↓
Reflection / Attributes
   ↓
Raw Mapping Metadata
   ↓
Normalization
   ↓
Validation
   ↓
Compilation
   ↓
CompiledEntityMetadata
   ↓
Metadata Cache
```

Posteriormente:

```text
Request N
   ↓
Metadata Registry
   ↓
Metadata Cache HIT
   ↓
CompiledEntityMetadata
```

sin repetir reflection.

---

# 5. Metadata ≠ runtime state

La metadata deberá describir:

```text
what something is
how it is mapped
how it can be accessed
what capabilities it has
```

No:

```text
what is happening right now
```

Por ejemplo:

```text
User.email → column users.email
```

es metadata.

Pero:

```text
User#42.email changed
```

es estado de `UnitOfWork`.

---

# 6. Categorías de metadata

VoltStack podrá manejar al menos:

```text
Entity Metadata
Mapping Metadata
Relationship Metadata
Schema Metadata
Type Metadata
Platform Metadata
Query Semantic Metadata
Hydration Metadata
Persistence Metadata
Extension Metadata
Driver Metadata
Capability Metadata
```

---

# 7. Metadata source vs compiled metadata

Debe distinguirse:

```text
Metadata Source
```

de:

```text
Compiled Metadata
```

Ejemplo:

```text
PHP Attributes
      ↓
Source Metadata
      ↓
Compiler
      ↓
CompiledEntityMetadata
```

---

# 8. Source metadata

Puede provenir de:

```text
PHP attributes
PHP configuration
package definitions
extension contributors
schema introspection
driver/platform discovery
generated metadata
```

---

# 9. Compiled metadata

Será una representación:

- normalizada;
- validada;
- immutable;
- optimizada;
- preparada para hot paths.

---

# 10. Regla de hot path

Idealmente:

```text
Hot Path
→ no reflection
→ no attribute scanning
→ no mapping discovery
→ no repeated normalization
```

---

# 11. Arquitectura general

```text
Metadata Sources
      │
      ▼
Metadata Discovery
      │
      ▼
Normalization
      │
      ▼
Validation
      │
      ▼
Dependency Resolution
      │
      ▼
Compilation
      │
      ▼
Metadata Registry
      │
      ▼
Metadata Cache
```

En producción:

```text
Metadata Cache
      ↓
Metadata Registry
      ↓
ORM / Query / Schema / Hydration
```

---

# 12. MetadataCache

Contrato base:

```php
interface MetadataCache
{
    public function get(
        MetadataCacheKey $key,
    ): ?CompiledMetadata;

    public function put(
        MetadataCacheKey $key,
        CompiledMetadata $metadata,
    ): void;

    public function remove(
        MetadataCacheKey $key,
    ): void;
}
```

La implementación real podrá utilizar contratos más especializados.

---

# 13. No God Cache

No deberá existir una clase gigante:

```php
DatabaseMetadataCache
```

con conocimiento detallado de:

```text
entities
relationships
types
schema
drivers
platforms
hydrators
```

Preferir:

```text
MetadataCacheManager
        ↓
Typed Metadata Domains
```

---

# 14. Metadata domains

```php
enum MetadataDomain
{
    case ENTITY;
    case RELATIONSHIP;
    case SCHEMA;
    case TYPE;
    case PLATFORM;
    case HYDRATION;
    case PERSISTENCE;
    case EXTENSION;
}
```

---

# 15. Typed metadata

Preferir:

```php
EntityMetadataCache
RelationshipMetadataCache
SchemaMetadataCache
TypeMetadataCache
```

sobre:

```php
mixed get(string $key);
```

---

# 16. MetadataCacheKey

Modelo conceptual:

```php
final readonly class MetadataCacheKey
{
    public function __construct(
        public MetadataDomain $domain,
        public MetadataIdentity $identity,
        public MetadataGeneration $generation,
        public MetadataCompilerVersion $compilerVersion,
    ) {}
}
```

---

# 17. Fórmula de identidad

```text
MetadataCacheKey
=
Domain
+
StructuralIdentity
+
Generation
+
CompilerVersion
+
RelevantConfigurationFingerprint
```

---

# 18. Structural identity

Ejemplos:

```text
entity:App\Domain\User
relationship:User.organization
type:domain.money
platform:mysql
schema:primary
```

---

# 19. FQCN como metadata identity

Para metadata interna de PHP puede utilizarse el FQCN.

Sin embargo:

```text
Persisted Database Identity
≠
PHP FQCN
```

La regla previa sobre no persistir FQCN como discriminador sigue vigente.

---

# 20. Metadata fingerprint

```php
final readonly class MetadataFingerprint
{
    public function __construct(
        public string $algorithm,
        public string $value,
    ) {}
}
```

---

# 21. Fingerprint inputs

Dependiendo del dominio:

```text
mapping definitions
type definitions
relationship definitions
platform capabilities
extension registrations
compiler version
relevant configuration
```

---

# 22. Determinismo

Mismos inputs estructurales deberán producir:

```text
same normalized metadata
same fingerprint
```

---

# 23. No incluir datos irrelevantes

No deberá incluirse:

```text
request ID
current timestamp
worker PID
authenticated user
random UUID
```

en fingerprints estructurales.

---

# 24. Metadata generation

Se utilizará:

```php
final readonly class MetadataGeneration
{
    public function __construct(
        public string $value,
    ) {}
}
```

para separar generaciones incompatibles.

---

# 25. Generation ≠ timestamp

Aunque pueda derivarse de deployment/build information:

```text
MetadataGeneration
≠
CurrentTime
```

---

# 26. Generation purpose

Permite:

```text
Deployment A
→ generation G1

Deployment B
→ generation G2
```

sin que B interprete automáticamente metadata incompatible de A.

---

# 27. Deployment-safe keys

Ejemplo:

```text
voltstack:
database:
metadata:
app123:
generation-g17:
entity:
User
```

---

# 28. Blue-green deployment

Durante:

```text
Version A ────────┐
                  ├── shared Redis
Version B ────────┘
```

ambas versiones pueden utilizar generaciones diferentes.

---

# 29. No global flush requerido

Preferir:

```text
new generation
→ old entries become unreachable
```

sobre:

```text
FLUSHALL
```

---

# 30. Old generation cleanup

Las generaciones antiguas podrán eliminarse:

```text
asynchronously
TTL
deployment cleanup
cache garbage collection
```

---

# 31. Compiler version

La representación interna puede cambiar aunque el mapping no cambie.

Por tanto:

```text
MetadataCompilerVersion
```

deberá formar parte de compatibility.

---

# 32. Cache payload version

Además:

```text
MetadataPayloadVersion
```

permitirá evolucionar el formato serializado.

---

# 33. Entity metadata

Podrá cachearse:

```text
EntityType
EntityId
PHP class
table mapping
identifier mapping
field mappings
embedded mappings
relationship references
lifecycle configuration
change tracking policy
constructor/access strategy
```

---

# 34. Entity metadata no contiene entidades

Nunca:

```text
CompiledEntityMetadata
→ User object
```

---

# 35. Mapping metadata

Ejemplo:

```text
User.email
→ Column(users.email)
→ LogicalType(string)
→ nullable=false
→ length=150
```

---

# 36. Relationship metadata

El sistema definido en:

```text
148_DATABASE_RELATIONSHIP_METADATA_SYSTEM.md
```

produce metadata canónica candidata natural para caching.

---

# 37. Relationship metadata cache

Puede almacenar:

```text
RelationshipId
source entity type
target entity type
cardinality
owning side
inverse side
join mapping
cascade semantics
orphan semantics
fetch defaults
```

---

# 38. Relationship runtime state excluded

Nunca almacenar:

```text
collection initialized=true
User#42.organization loaded
pending relationship changes
```

Eso pertenece al scope ORM.

---

# 39. Type metadata

El `Type Registry` del documento 156 podrá utilizar metadata compilada para:

```text
type descriptors
parameter schemas
converter pipelines
platform mappings
capabilities
```

---

# 40. Registry vs cache

Debe distinguirse:

```text
TypeRegistry
=
runtime authoritative registry
```

de:

```text
MetadataCache
=
reusable storage mechanism
```

El cache no sustituye al registry.

---

# 41. Relationship registry

Misma regla:

```text
RelationshipMetadataRegistry
≠
RelationshipMetadataCache
```

---

# 42. Entity metadata registry

```text
EntityMetadataRegistry
```

será la API runtime autoritativa.

El cache será una fuente optimizada para poblarlo.

---

# 43. Registry lifecycle

```text
Boot
 ↓
Cache Load
 ↓
Validation
 ↓
Registry Population
 ↓
Freeze
```

---

# 44. Frozen registry

Después de boot:

```text
Registry
→ immutable/frozen
```

cuando la arquitectura del componente lo permita.

---

# 45. Runtime registration

No se permitirá registrar mappings arbitrariamente durante requests ordinarios.

---

# 46. Razón

Hot registration produciría:

```text
cache inconsistency
worker divergence
query plan invalidation complexity
hydration plan invalidation
concurrency hazards
```

---

# 47. Schema metadata

Debe distinguirse cuidadosamente.

```text
Schema Metadata Cache
```

puede almacenar resultados derivados de introspección.

Pero:

```text
CachedSchemaMetadata
≠
CurrentDatabaseSchemaTruth
```

---

# 48. Schema introspection cache

Ejemplo:

```text
SHOW / catalog introspection
       ↓
Schema Introspector
       ↓
Normalized Schema Model
       ↓
Metadata Cache
```

---

# 49. Staleness

El schema puede cambiar fuera del proceso.

Por tanto, introspection metadata deberá poseer:

```text
freshness
generation
source identity
coverage
```

---

# 50. Schema coverage

Debe preservarse:

```text
COMPLETE
PARTIAL
UNKNOWN
```

según las reglas del documento 96.

Nunca:

```text
cached PARTIAL
→ interpreted as COMPLETE
```

---

# 51. Not observed ≠ absent

Esta regla permanece:

```text
NotObserved
≠
Absent
```

aunque la metadata provenga del cache.

---

# 52. Schema migration integration

Después de una migration confirmada:

```text
Schema Metadata
```

afectada deberá:

```text
invalidate
```

o avanzar de generación.

---

# 53. External schema changes

Si alguien ejecuta DDL fuera de VoltStack:

```text
metadata cache
```

puede quedar stale.

---

# 54. Production policy

Aplicaciones que permiten cambios externos frecuentes podrán:

```text
disable long-lived schema metadata cache
use short TTL
explicit refresh
deployment generation
schema version validation
```

---

# 55. Platform metadata

Puede incluir:

```text
normalized capabilities
identifier limits
supported feature descriptors
type support descriptors
transaction feature metadata
SQL feature metadata
```

---

# 56. Capability discovery

Si determinadas capabilities se descubren mediante conexión:

```text
Database
 ↓
Capability Discovery
 ↓
Normalized Platform Capability Set
 ↓
Metadata Cache
```

---

# 57. Version ≠ capability

La regla permanece:

```text
DatabaseVersion
≠
DatabaseCapabilitySet
```

---

# 58. Capability cache identity

Deberá considerar:

```text
platform
server/version fingerprint
driver
relevant connection mode
extensions/features
```

cuando correspondan.

---

# 59. Capability staleness

Un failover podría llevar a otro servidor con capabilities diferentes.

Por tanto:

```text
Topology Change
```

podrá invalidar platform metadata asociada.

---

# 60. Connection-specific metadata

No toda metadata de una conexión es globalmente reusable.

Debe clasificarse.

---

# 61. Metadata stability

```php
enum MetadataStability
{
    case BUILD_STABLE;
    case DEPLOYMENT_STABLE;
    case TOPOLOGY_STABLE;
    case CONNECTION_STABLE;
    case SHORT_LIVED;
}
```

---

# 62. BUILD_STABLE

Ejemplos:

```text
PHP entity mapping
compiled accessors
relationship definitions
custom type definitions
```

---

# 63. DEPLOYMENT_STABLE

Puede depender de:

```text
application configuration
enabled packages
extension set
```

---

# 64. TOPOLOGY_STABLE

Puede depender de:

```text
current database server topology
platform capabilities
```

---

# 65. CONNECTION_STABLE

Puede depender de:

```text
session mode
driver configuration
```

y quizá no sea apropiada para shared cache.

---

# 66. SHORT_LIVED

Ejemplo:

```text
schema introspection during migration development
```

---

# 67. MetadataCachePolicy

```php
final readonly class MetadataCachePolicy
{
    public function __construct(
        public MetadataStability $stability,
        public MetadataCacheScope $scope,
        public ?Duration $ttl,
        public bool $validateOnLoad,
    ) {}
}
```

---

# 68. Cache scopes

```php
enum MetadataCacheScope
{
    case PROCESS;
    case DISTRIBUTED;
    case BUILD_ARTIFACT;
}
```

---

# 69. BUILD_ARTIFACT

VoltStack deberá contemplar metadata precompilada durante:

```text
build
deploy
warmup
```

---

# 70. Production optimization

Ideal:

```text
Source Code
   ↓
Build
   ↓
Compile Database Metadata
   ↓
Metadata Artifact
   ↓
Deploy
   ↓
Workers load compiled metadata
```

---

# 71. Build-time metadata

Esto permite acercarse a:

```text
zero reflection hot path
```

---

# 72. Metadata compiler command

Conceptualmente:

```bash
php voltstack database:metadata:compile
```

---

# 73. Warmup command

```bash
php voltstack database:cache:warm
```

podrá preparar:

```text
entity metadata
relationship metadata
type metadata
platform-independent metadata
```

---

# 74. Platform-dependent metadata

No siempre podrá compilarse sin conexión.

Por tanto deberá distinguirse:

```text
OfflineCompilableMetadata
```

de:

```text
RuntimeDiscoveredMetadata
```

---

# 75. Offline compilation

Ejemplos:

```text
PHP attributes
entity mappings
relationships
value object mappings
custom type definitions
compiled accessors
```

---

# 76. Runtime discovery

Ejemplos:

```text
actual database capabilities
schema introspection
server-specific extension support
```

---

# 77. Metadata dependency graph

Un metadata artifact puede depender de otros.

Ejemplo:

```text
EntityMetadata(User)
      │
      ├── Type(string)
      ├── Type(uuid)
      └── Relationship(User.organization)
                         ↓
                EntityMetadata(Organization)
```

---

# 78. MetadataDependencyGraph

```php
interface MetadataDependencyGraph
{
    public function dependenciesOf(
        MetadataIdentity $metadata,
    ): MetadataDependencySet;
}
```

---

# 79. Why dependencies matter

Si cambia:

```text
Type domain.money
```

pueden quedar incompatibles:

```text
EntityMetadata(Product)
HydrationPlan(Product)
PersistenceMetadata(Product)
QueryMetadata(Product.price)
```

---

# 80. Generation propagation

La arquitectura deberá permitir invalidar:

```text
dependent compiled artifacts
```

cuando cambia una dependencia estructural.

---

# 81. Dependency fingerprints

Una estrategia:

```text
EntityMetadataFingerprint
=
OwnDefinitionFingerprint
+
DependencyFingerprints
```

---

# 82. Merkle-like compilation

Conceptualmente:

```text
Field Type FP ─────┐
Relationship FP ───┼─→ Entity Metadata FP
Mapping FP ─────────┘
```

Esto permite detectar cambios transitivos.

---

# 83. Cycles

Relationships pueden formar ciclos:

```text
User
 ↕
Organization
```

El fingerprint system deberá ser cycle-safe.

---

# 84. Cycle handling

Preferir:

```text
stable symbolic dependency identities
+
separate generation/fingerprint resolution
```

en vez de recursión infinita.

---

# 85. Metadata compilation pipeline

```text
Discover
   ↓
Normalize
   ↓
Resolve Symbols
   ↓
Validate
   ↓
Resolve Dependencies
   ↓
Compile
   ↓
Fingerprint
   ↓
Freeze
   ↓
Store
```

---

# 86. Cache load pipeline

```text
Lookup
  ↓
Decode
  ↓
Version Check
  ↓
Generation Check
  ↓
Dependency Check
  ↓
Invariant Validation
  ↓
Registry Installation
```

---

# 87. Cache hit ≠ blindly trust bytes

Regla:

```text
EntryExists
≠
CompatibleMetadata
```

---

# 88. Compatibility validation

Un entry deberá verificarse contra:

```text
payload version
compiler version
metadata generation
domain
identity
configuration fingerprint
dependency fingerprint
```

según su política.

---

# 89. Corruption

Un payload corrupto deberá:

```text
reject
evict
recompile
```

cuando sea posible.

---

# 90. Never partially install metadata

No:

```text
load 70% entity metadata
→ registry ACTIVE
```

si el conjunto requerido debe ser atómico.

---

# 91. Metadata set installation

Preferir:

```text
load
↓
validate complete set
↓
freeze
↓
publish registry
```

---

# 92. Atomic registry publication

Para runtimes concurrentes:

```text
OldRegistry
     ↓
Compile NewRegistry privately
     ↓
Validate
     ↓
Atomic publish
```

cuando hot replacement sea soportado.

---

# 93. Default production model

Más simple y seguro:

```text
worker boot
→ load metadata
→ freeze
→ use for worker lifetime
```

Cambios:

```text
new deployment
→ new workers
```

---

# 94. Hot metadata reload

No será requisito del core.

Puede existir como tooling de desarrollo.

---

# 95. Development mode

En desarrollo:

```text
source file changes
→ metadata invalidation
→ recompile
```

puede ser conveniente.

---

# 96. Production mode

En producción:

```text
immutable deployment generation
```

será preferible.

---

# 97. Reflection elimination

Podrán compilarse accessors:

```text
PropertyReader
PropertyWriter
IdentifierAccessor
ConstructorInvoker
```

---

# 98. Compiled accessor

Contrato:

```php
interface PropertyAccessor
{
    public function read(object $object): mixed;

    public function write(
        object $object,
        mixed $value,
    ): void;
}
```

---

# 99. Accessor implementation

Puede usar:

```text
generated PHP
cached closures
precomputed reflection handles
specialized accessors
```

dependiendo del runtime y seguridad.

---

# 100. No arbitrary eval

Metadata cache no deberá depender de ejecutar strings arbitrarios provenientes del cache.

---

# 101. Generated PHP

Si VoltStack genera PHP durante build:

```text
generated file
```

deberá ser tratado como trusted deployment artifact, no como arbitrary remote cache payload.

---

# 102. Remote cache payload

Un Redis compromise no deberá convertirse directamente en:

```text
arbitrary code execution
```

---

# 103. Serialization

Preferir formatos:

```text
explicit
versioned
typed
validated
```

---

# 104. MetadataCodec

```php
interface MetadataCodec
{
    public function encode(
        CompiledMetadata $metadata,
    ): string;

    public function decode(
        string $payload,
        MetadataDecodeContext $context,
    ): CompiledMetadata;
}
```

---

# 105. Object serialization

Evitar depender indiscriminadamente de:

```php
serialize()
unserialize()
```

para metadata compartida.

---

# 106. Immutable metadata

Una vez compilada:

```text
CompiledMetadata
→ readonly/immutable
```

---

# 107. Why immutable

Permite:

```text
safe sharing
worker reuse
concurrent reads
stable fingerprints
predictable caches
```

---

# 108. Mutable runtime overlays

Si algún subsystem requiere información dinámica:

```text
CompiledMetadata
+
ScopedRuntimeContext
```

No modificar el metadata base.

---

# 109. Example

```text
RelationshipMetadata:
    fetchMode = LAZY
```

es estructural.

Pero:

```text
current operation forces EAGER
```

es runtime query/fetch context.

No debe mutar relationship metadata.

---

# 110. Hydration metadata

Podrán cachearse:

```text
field-to-column mappings
compiled accessors
constructor strategy
identifier assembly rules
value conversion pipeline descriptors
```

---

# 111. HydrationPlan distinction

Debe mantenerse:

```text
Hydration Metadata
≠
Hydration Plan
```

El primero describe estructura reusable.

El segundo puede depender de una query/result shape específica.

---

# 112. Hydration plan cache

El documento 141 ya define:

```text
Hydration Cache
```

Por tanto:

```text
Metadata Cache
```

no deberá duplicarlo.

---

# 113. Shared dependencies

Hydration Plan Cache puede depender de:

```text
EntityMetadataGeneration
TypeRegistryGeneration
```

provenientes del Metadata System.

---

# 114. Persistence metadata

Podrá incluir:

```text
insertable fields
updatable fields
generated values
identifier generation
version field
column ordering
dependency descriptors
```

---

# 115. Persistence metadata ≠ Persistence Plan

```text
Persistence Metadata
≠
Persistence Plan
```

El plan depende de cambios concretos del UnitOfWork.

---

# 116. Query semantic metadata

El Query Engine podrá consumir metadata como:

```text
field existence
field type
relationship path
column mapping
entity symbols
```

---

# 117. Query AST independence

El Metadata Cache:

```text
does not create SQL
does not execute query
does not mutate AST
```

---

# 118. Schema-aware query resolution

Documento 38 podrá consultar:

```text
Schema Metadata Registry
```

sin conocer cómo fue cacheada.

---

# 119. Abstraction boundary

Consumers deberán depender de:

```text
Metadata Registry / Resolver
```

no directamente de:

```text
RedisMetadataCache
```

---

# 120. Cache backend abstraction

```php
interface MetadataCacheStore
{
    public function get(
        MetadataCacheKey $key,
    ): ?MetadataCachePayload;

    public function put(
        MetadataCacheKey $key,
        MetadataCachePayload $payload,
    ): void;

    public function delete(
        MetadataCacheKey $key,
    ): void;
}
```

---

# 121. Backends

Posibles:

```text
InMemory
Filesystem
Redis
Build Artifact
Null
Tiered
```

---

# 122. Filesystem cache

Particularmente útil para:

```text
compiled PHP metadata
production deployment artifacts
```

---

# 123. Redis metadata cache

Puede ser útil en:

```text
multiple application servers
shared runtime-discovered metadata
```

pero introduce:

```text
network latency
serialization
availability dependency
```

---

# 124. L1 process cache

Una vez cargada metadata immutable:

```text
process memory
```

es normalmente el hot-path ideal.

---

# 125. Tiered architecture

```text
Registry
   ↓ miss
L1 Metadata Cache
   ↓ miss
L2 / Filesystem / Build Artifact
   ↓ miss
Metadata Compiler
```

---

# 126. Registry and L1 distinction

En muchos casos:

```text
Frozen Registry
```

ya funciona como hot runtime representation.

Por tanto, no se deberá añadir una L1 redundante sin beneficio.

---

# 127. Cold boot

Objetivo:

```text
Cold Boot
→ load compiled metadata efficiently
```

---

# 128. Warm worker

Objetivo:

```text
Warm Request
→ registry lookup only
```

---

# 129. Metadata cache failure

Para metadata derivable:

```text
cache unavailable
→ rebuild metadata
```

cuando sea seguro.

---

# 130. FAIL_OPEN semantics

```text
Metadata Cache Failure
→ source discovery/compiler fallback
```

será default cuando las fuentes estén disponibles.

---

# 131. Build-only deployments

Si producción se configura:

```text
compiled metadata required
source discovery disabled
```

entonces missing metadata puede ser:

```text
BOOT FAILURE
```

---

# 132. Strict production mode

Ejemplo:

```php
'database.metadata' => [
    'mode' => 'compiled_only',
];
```

Esto permite detectar deployments incompletos.

---

# 133. Modes

```php
enum MetadataRuntimeMode
{
    case DISCOVER_AND_CACHE;
    case PREFER_COMPILED;
    case COMPILED_ONLY;
}
```

---

# 134. DISCOVER_AND_CACHE

Adecuado para:

```text
development
```

---

# 135. PREFER_COMPILED

Adecuado para:

```text
general production
```

---

# 136. COMPILED_ONLY

Adecuado para:

```text
highly controlled production deployments
```

---

# 137. Cache stampede

En cold deployment con 100 workers:

```text
100 workers
→ same metadata compilation
```

puede desperdiciar recursos.

---

# 138. Build-time compilation preferred

La mejor solución para metadata estable:

```text
compile once during build
```

en vez de distributed runtime locks.

---

# 139. Runtime single-flight

Para runtime-discovered metadata podrá utilizarse:

```text
MetadataCompilationSingleFlight
```

---

# 140. Bounded locking

Nunca:

```text
wait forever for metadata lock
```

---

# 141. Compiler failure

Si la compilación falla:

```text
do not publish incomplete metadata
```

---

# 142. Negative metadata caching

Puede ser útil cachear temporalmente:

```text
mapping not found
unsupported capability
```

pero con cuidado.

---

# 143. Development risk

Si se agrega una clase nueva:

```text
negative cache
```

no deberá impedir descubrirla indefinidamente.

---

# 144. Negative cache policy

```text
short TTL
generation-bound
development disabled
```

---

# 145. Extension metadata

Paquetes oficiales o de terceros podrán contribuir:

```text
types
mapping strategies
query capabilities
platform capabilities
schema extensions
```

---

# 146. Extension registration

Debe ocurrir durante:

```text
bootstrap
```

antes de:

```text
metadata freeze
```

---

# 147. ExtensionSetFingerprint

La metadata compilada deberá considerar:

```text
enabled database extensions
```

cuando cambien su semántica.

---

# 148. Package version

No siempre será suficiente usar:

```text
package version
```

como fingerprint.

Preferir:

```text
semantic metadata contribution fingerprint
```

---

# 149. Conflict detection

Dos extensiones que registran:

```text
same TypeId
```

deberán fallar de manera determinista.

Nunca:

```text
last write wins
```

---

# 150. Cached conflicts

El cache no deberá ocultar conflictos de configuración actuales.

---

# 151. Cache validation vs current extension set

Un entry compilado con:

```text
Extension A
Extension B
```

no podrá cargarse silenciosamente si ahora existe:

```text
Extension A
Extension C
```

---

# 152. Security

Metadata puede revelar:

```text
table names
column names
class names
relationships
schema structure
```

Por tanto, sigue siendo información potencialmente sensible.

---

# 153. Cache access control

Backends distribuidos deberán estar protegidos.

---

# 154. No credentials

Nunca almacenar dentro de metadata ordinaria:

```text
database password
access token
private key
cloud secret
```

---

# 155. Connection configuration

Metadata podrá incluir:

```text
connection capability identity
```

pero no:

```text
raw credentials
```

---

# 156. Cache poisoning

Un atacante que modifique metadata podría intentar alterar mappings.

Por tanto:

```text
metadata cache
```

debe considerarse parte de la superficie de integridad.

---

# 157. Integrity validation

Opcionalmente:

```text
signed build artifacts
checksum
HMAC
immutable filesystem
```

podrán proteger metadata compilada.

---

# 158. Build artifact signing

Para entornos empresariales:

```text
Build
 ↓
Metadata Artifact
 ↓
Sign
 ↓
Deploy
 ↓
Verify
 ↓
Load
```

---

# 159. Remote metadata trust

No deberá convertirse metadata remota en código ejecutable arbitrario.

---

# 160. Schema metadata security

Schema introspection metadata no deberá exponerse automáticamente en APIs públicas.

---

# 161. Diagnostics redaction

Los reportes podrán mostrar:

```text
domain
generation
cache status
entry count
compiler version
```

sin exponer detalles sensibles innecesarios.

---

# 162. MetadataInspector

API:

```php
DB::metadata()->inspect();
```

---

# 163. Entity inspection

```php
DB::metadata()
    ->entity(User::class)
    ->explain();
```

---

# 164. Example diagnostic

```text
DATABASE METADATA

Entity:
    App\Domain\User

Source:
    COMPILED_CACHE

Generation:
    db-meta-g42

Compiler:
    v3

Mapping:
    VALID

Dependencies:
    types:string
    types:uuid
    relationship:user.organization

Registry:
    FROZEN

Reflection Used:
    NO
```

---

# 165. Cache status command

Conceptualmente:

```bash
php voltstack database:metadata:status
```

---

# 166. Output

```text
Database Metadata Cache

Mode:
    PREFER_COMPILED

Generation:
    g42

Entity Metadata:
    148 entries

Relationship Metadata:
    327 entries

Custom Types:
    18 entries

Schema Metadata:
    4 snapshots

Compilation:
    HEALTHY

Registry:
    FROZEN
```

---

# 167. Clear command

```bash
php voltstack database:metadata:clear
```

deberá limpiar exclusivamente metadata cache del namespace correspondiente.

No:

```text
FLUSHALL
```

---

# 168. Compile command

```bash
php voltstack database:metadata:compile
```

---

# 169. Validate command

```bash
php voltstack database:metadata:validate
```

---

# 170. Warm command

```bash
php voltstack database:metadata:warm
```

---

# 171. Dump command

En desarrollo:

```bash
php voltstack database:metadata:show App\\Domain\\User
```

---

# 172. No sensitive dump by default

Detalles de producción podrán requerir:

```text
debug permission
```

---

# 173. Telemetry

Eventos conceptuales:

```text
MetadataCacheLookupStarted
MetadataCacheHit
MetadataCacheMiss
MetadataCacheRejected
MetadataCompilationStarted
MetadataCompilationCompleted
MetadataCompilationFailed
MetadataRegistryFrozen
MetadataGenerationChanged
MetadataCacheCorruptionDetected
```

---

# 174. Metrics

```text
db.metadata_cache.hit
db.metadata_cache.miss
db.metadata_cache.compile
db.metadata_cache.compile.duration
db.metadata_cache.load.duration
db.metadata_cache.entries
db.metadata_cache.bytes
db.metadata_cache.invalid
```

---

# 175. Cardinality

Labels permitidos:

```text
domain=entity|relationship|type|schema
source=memory|filesystem|redis|build
result=hit|miss|invalid
```

Evitar:

```text
entity_class=App\Domain\User
```

como metric label de alta cardinalidad.

---

# 176. Tracing

Span:

```text
database.metadata.compile
```

podrá contener:

```text
domain
entry_count
source
cache_hit
```

---

# 177. No repeated spans on hot lookup

No deberá generar tracing excesivo por cada field lookup en producción.

---

# 178. Performance model

Sin cache:

```text
Cost
=
Discovery
+
Reflection
+
Normalization
+
Validation
+
Compilation
```

Con cache:

```text
Cost
=
Lookup
+
Decode
+
CompatibilityCheck
```

Con frozen registry:

```text
HotPathCost
≈
O(1) registry lookup
```

---

# 179. Memory model

Metadata es excelente candidata para memoria compartida dentro de un worker porque es:

```text
immutable
small relative to application data
frequently reused
structural
```

---

# 180. Memory budget

Aun así deberá existir:

```text
metadata memory budget
```

para aplicaciones con miles de entities/mappings.

---

# 181. Lazy metadata loading

Puede ahorrar memoria:

```text
load entity metadata on first use
```

pero introduce:

```text
runtime compilation/lookup complexity
```

---

# 182. Eager metadata loading

Puede mejorar:

```text
predictability
error detection
hot-path performance
```

---

# 183. Recommended production policy

Para aplicaciones medianas:

```text
compile all
validate all
load all required metadata
freeze registry
```

---

# 184. Very large applications

Podrán utilizar:

```text
module-scoped metadata bundles
```

---

# 185. Metadata bundle

Ejemplo:

```text
database-metadata/
├── core.php
├── billing.php
├── catalog.php
└── analytics.php
```

---

# 186. Bundle dependency graph

```text
billing
  ↓
core
```

deberá ser explícito.

---

# 187. Partial bundle failure

Un bundle requerido que falle no deberá producir un registry parcialmente funcional sin indicarlo.

---

# 188. Module isolation

Paquetes podrán contribuir metadata mediante:

```text
MetadataContributor
```

---

# 189. MetadataContributor

```php
interface MetadataContributor
{
    public function contribute(
        MetadataContributionContext $context,
    ): iterable;
}
```

---

# 190. Contributor restrictions

No deberá:

```text
execute arbitrary application queries
mutate EntityManager
depend on HTTP request
depend on authenticated user
```

---

# 191. Deterministic compilation

Metadata compilation deberá poder ejecutarse:

```text
CLI
CI
build server
worker boot
```

con el mismo resultado para mismos inputs.

---

# 192. Environment-specific config

Si configuración realmente altera mapping:

```text
configuration fingerprint
```

deberá reflejarlo.

---

# 193. Environment name alone

No usar simplemente:

```text
APP_ENV=production
```

como identidad si dos deployments production tienen mappings diferentes.

---

# 194. Relevant configuration fingerprint

Preferir hash de:

```text
actual semantic configuration
```

---

# 195. Database connection changes

Cambiar:

```text
host
```

no debería invalidar entity mappings.

Cambiar:

```text
platform/capabilities
```

sí puede invalidar metadata dependiente de plataforma.

---

# 196. Dependency precision

El sistema deberá evitar:

```text
any config change
→ invalidate all metadata
```

cuando pueda identificar dependencies más precisas.

---

# 197. But correctness first

Si dependency precision es desconocida:

```text
broader invalidation
```

es preferible.

---

# 198. Integration with Query Cache

Query Cache podrá incluir:

```text
MetadataGeneration
```

en sus fingerprints.

---

# 199. Example

Si cambia:

```text
User.email mapping
```

un cached semantic query:

```text
User.email = :email
```

puede quedar incompatible.

---

# 200. Query cache dependency

```text
QueryArtifact
→ EntityMetadataGeneration
```

o fingerprint equivalente.

---

# 201. Integration with Result Cache

Result Cache podrá depender de metadata generation cuando el payload o result shape dependa del mapping.

---

# 202. Result data vs metadata

Cambiar metadata no significa necesariamente que:

```text
database data changed
```

pero puede volver incompatible la interpretación del cached payload.

---

# 203. Integration with Entity Cache

Entity Cache dependerá fuertemente de:

```text
EntityMetadataGeneration
TypeMetadataGeneration
```

---

# 204. Integration with Schema Cache

Schema metadata será un dominio especializado dentro de esta arquitectura, pero podrá tener freshness policies diferentes.

---

# 205. Integration with Migration

Migration confirmada podrá emitir:

```text
SchemaMetadataInvalidation
```

---

# 206. Integration with Type System

Cambiar:

```text
Custom Type converter
```

puede requerir nueva metadata generation.

---

# 207. Integration with Value Conversion

Compiled conversion pipelines podrán ser referenciados desde metadata.

---

# 208. No runtime values in conversion metadata

Nunca almacenar:

```text
current converted value
```

como metadata.

---

# 209. Integration with Casting

Cast definitions compiladas podrán ser metadata.

---

# 210. Cast state

Los resultados concretos de casting no serán metadata.

---

# 211. Integration with Value Objects

`ValueObjectMappingMetadata` será candidato natural para compilation/cache.

---

# 212. Integration with Enum Mapping

Podrán cachearse:

```text
EnumTypeId
case mapping
aliases
storage strategy
```

---

# 213. Enum runtime value

```text
OrderStatus::PAID
```

en una entidad concreta no pertenece al Metadata Cache.

---

# 214. Integration with relationships

Fetch defaults son metadata.

Fetch state no.

```text
LAZY default
```

es metadata.

```text
User#42.posts currently loaded
```

no.

---

# 215. Integration with optimistic locking

La existencia y mapping del:

```text
version field
```

es metadata.

El valor:

```text
version = 17
```

de una entidad concreta no lo es.

---

# 216. Integration with pessimistic locking

Platform lock capabilities pueden ser metadata.

El lock actualmente adquirido no.

---

# 217. Integration with transactions

Supported isolation levels pueden ser capability metadata.

La isolation efectiva de una transacción activa pertenece a `TransactionContext`.

---

# 218. Integration with sharding

Shard mapping definitions estáticas podrán ser metadata.

La resolución:

```text
Entity#42 → shard-7
```

para una operación concreta pertenece al routing context.

---

# 219. Integration with replicas

Replica topology dinámica no deberá confundirse con build-stable metadata.

---

# 220. Metadata classification matrix

| Metadata | Stability | Typical Scope |
|---|---|---|
| Entity mapping | Build stable | Build/L1 |
| Relationship mapping | Build stable | Build/L1 |
| Custom type definitions | Build stable | Build/L1 |
| Compiled accessors | Build stable | Build/L1 |
| Schema snapshot | Short/topology stable | L1/L2 |
| Platform capabilities | Topology stable | L1/L2 |
| Current transaction isolation | Runtime | Never metadata cache |
| Current replica lag | Runtime | Never structural metadata |
| Entity field value | Runtime data | Never metadata cache |
| IdentityMap state | Scoped mutable | Never metadata cache |

---

# 221. Persistent runtime architecture

Para FrankenPHP:

```text
Worker Boot
    ↓
Load Compiled Metadata
    ↓
Freeze Registries
    ↓
Request 1
    ↓
reuse
    ↓
Request 2
    ↓
reuse
    ↓
Request N
```

---

# 222. No request reset required for immutable metadata

A diferencia de:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
```

la metadata immutable puede sobrevivir requests.

---

# 223. Critical condition

Solo si:

```text
metadata is truly immutable
```

---

# 224. Mutable metadata bug

Si un request modifica:

```text
EntityMetadata
```

podría contaminar todos los requests posteriores.

Por tanto:

```text
readonly/frozen
```

será obligatorio para compiled metadata compartida.

---

# 225. RoadRunner

Mismo modelo:

```text
worker-shared immutable metadata
+
request-scoped mutable database state
```

---

# 226. OpenSwoole

Metadata immutable podrá compartirse entre coroutines si sus objetos internos también son concurrency-safe.

---

# 227. Hidden mutable objects

No basta con:

```php
readonly class Metadata
```

si contiene:

```text
mutable collection
mutable registry
mutable closure state
```

---

# 228. Deep immutability

La meta será:

```text
DeepImmutable(CompiledMetadata)
```

---

# 229. Cache warmup lifecycle

```text
Application Build
        ↓
Discover Metadata
        ↓
Compile
        ↓
Validate
        ↓
Write Artifact
        ↓
Deployment
        ↓
Worker Boot
        ↓
Verify Artifact
        ↓
Load
        ↓
Freeze Registries
```

---

# 230. Failure model

Posibles fallos:

```text
CACHE_MISS
CORRUPTED_ENTRY
VERSION_MISMATCH
GENERATION_MISMATCH
DEPENDENCY_MISMATCH
COMPILER_MISMATCH
MAPPING_INVALID
EXTENSION_CONFLICT
SCHEMA_METADATA_STALE
CAPABILITY_METADATA_STALE
ARTIFACT_MISSING
ARTIFACT_INTEGRITY_FAILURE
```

---

# 231. Error hierarchy

```text
MetadataCacheException
├── MetadataCacheLookupException
├── MetadataCacheWriteException
├── MetadataCacheDecodeException
├── MetadataCacheCorruptionException
├── MetadataGenerationMismatchException
├── MetadataCompilerVersionMismatchException
├── MetadataDependencyMismatchException
├── MetadataArtifactMissingException
├── MetadataArtifactIntegrityException
├── MetadataCompilationException
├── MetadataRegistryFreezeException
├── MetadataExtensionConflictException
└── MetadataCacheInvariantViolationException
```

---

# 232. Cache miss is not exceptional

En modos:

```text
DISCOVER_AND_CACHE
PREFER_COMPILED
```

un miss ordinario no deberá generar excepción.

---

# 233. COMPILED_ONLY miss

En:

```text
COMPILED_ONLY
```

sí puede convertirse en:

```text
MetadataArtifactMissingException
```

---

# 234. Corruption policy

```text
PREFER_COMPILED:
    reject
    recompile if possible
    replace

COMPILED_ONLY:
    reject
    boot failure
```

---

# 235. Directory structure

```text
src/Quantum/Database/Cache/Metadata/
│
├── MetadataCacheManager.php
├── MetadataCache.php
├── MetadataCacheKey.php
├── MetadataCachePolicy.php
├── MetadataCacheScope.php
├── MetadataRuntimeMode.php
├── MetadataDomain.php
├── MetadataStability.php
├── MetadataGeneration.php
├── MetadataFingerprint.php
├── MetadataCompilerVersion.php
├── MetadataPayloadVersion.php
│
├── Contract/
│   ├── CompiledMetadata.php
│   ├── MetadataIdentity.php
│   ├── MetadataContributor.php
│   ├── MetadataCompiler.php
│   └── MetadataResolver.php
│
├── Entity/
│   ├── EntityMetadataCache.php
│   ├── CompiledEntityMetadata.php
│   └── EntityMetadataIdentity.php
│
├── Relationship/
│   ├── RelationshipMetadataCache.php
│   ├── CompiledRelationshipMetadata.php
│   └── RelationshipMetadataIdentity.php
│
├── Schema/
│   ├── SchemaMetadataCache.php
│   ├── CachedSchemaMetadata.php
│   └── SchemaMetadataFreshness.php
│
├── Type/
│   ├── TypeMetadataCache.php
│   ├── CompiledTypeMetadata.php
│   └── TypeMetadataIdentity.php
│
├── Platform/
│   ├── PlatformMetadataCache.php
│   ├── CompiledPlatformMetadata.php
│   └── PlatformCapabilityMetadata.php
│
├── Dependency/
│   ├── MetadataDependency.php
│   ├── MetadataDependencySet.php
│   ├── MetadataDependencyGraph.php
│   ├── MetadataDependencyResolver.php
│   └── MetadataDependencyFingerprint.php
│
├── Compilation/
│   ├── MetadataCompilationPipeline.php
│   ├── MetadataCompilationContext.php
│   ├── MetadataCompilationResult.php
│   ├── MetadataCompilationSingleFlight.php
│   └── MetadataBundleCompiler.php
│
├── Registry/
│   ├── MetadataRegistry.php
│   ├── MetadataRegistryBuilder.php
│   ├── FrozenMetadataRegistry.php
│   └── MetadataRegistryValidator.php
│
├── Store/
│   ├── MetadataCacheStore.php
│   ├── InMemoryMetadataCacheStore.php
│   ├── FilesystemMetadataCacheStore.php
│   ├── RedisMetadataCacheStore.php
│   ├── BuildArtifactMetadataStore.php
│   ├── TieredMetadataCacheStore.php
│   └── NullMetadataCacheStore.php
│
├── Codec/
│   ├── MetadataCodec.php
│   ├── MetadataEncoder.php
│   ├── MetadataDecoder.php
│   └── MetadataPayloadHeader.php
│
├── Artifact/
│   ├── MetadataArtifact.php
│   ├── MetadataArtifactManifest.php
│   ├── MetadataArtifactWriter.php
│   ├── MetadataArtifactLoader.php
│   └── MetadataArtifactVerifier.php
│
├── Invalidation/
│   ├── MetadataInvalidator.php
│   ├── MetadataGenerationManager.php
│   ├── MetadataInvalidationReason.php
│   └── MetadataDependencyInvalidator.php
│
├── Diagnostics/
│   ├── MetadataInspector.php
│   ├── MetadataExplainer.php
│   ├── MetadataDiagnosticReport.php
│   └── MetadataCacheStatus.php
│
├── Telemetry/
│   ├── MetadataCacheTelemetry.php
│   ├── MetadataCacheHitEvent.php
│   ├── MetadataCompilationEvent.php
│   └── MetadataGenerationChangedEvent.php
│
└── Exception/
    ├── MetadataCacheException.php
    ├── MetadataCacheCorruptionException.php
    ├── MetadataGenerationMismatchException.php
    ├── MetadataCompilationException.php
    └── MetadataCacheInvariantViolationException.php
```

---

# 236. Testing strategy

Se requerirán pruebas de:

```text
compilation
serialization
generation
dependencies
registries
extensions
schema metadata
platform metadata
persistent workers
concurrency
security
performance
deployment
```

---

# 237. Deterministic compilation test

Mismos inputs:

```text
Compile A
Compile B
```

deberán producir:

```text
same semantic metadata
same fingerprint
```

---

# 238. Changed mapping test

Cambiar:

```text
User.email length 150 → 255
```

deberá alterar metadata/fingerprint correspondiente.

---

# 239. Unrelated configuration test

Cambiar:

```text
application log level
```

no deberá invalidar Entity Metadata.

---

# 240. Dependency test

Cambiar:

```text
domain.money type
```

deberá invalidar metadata dependiente.

---

# 241. Relationship dependency test

Modificar relationship mapping deberá afectar fingerprints relacionados.

---

# 242. Cycle test

```text
User → Organization → User
```

no deberá producir recursión infinita.

---

# 243. Corrupted payload test

Debe:

```text
detect
reject
not install
```

---

# 244. Generation mismatch test

```text
G1 payload
+
G2 application
```

deberá rechazarse si son incompatibles.

---

# 245. Compiler mismatch test

Metadata producida por compiler incompatible deberá rechazarse.

---

# 246. Extension set test

Cambiar extensiones activas deberá invalidar metadata dependiente.

---

# 247. Conflict test

Dos TypeIds iguales deberán fallar determinísticamente.

---

# 248. Schema coverage test

`PARTIAL` deberá conservarse después de cache encode/decode.

---

# 249. Schema stale test

Migration confirmada deberá invalidar metadata correspondiente.

---

# 250. Platform topology test

Cambio de servidor/capabilities deberá evitar reuse incompatible.

---

# 251. Compiled-only test

Missing artifact deberá producir boot failure.

---

# 252. Prefer-compiled test

Missing artifact deberá permitir compilation fallback.

---

# 253. Worker reuse test

100 requests deberán reutilizar el mismo frozen registry.

---

# 254. Worker contamination test

Ningún request deberá poder mutar metadata compartida.

---

# 255. OpenSwoole concurrency test

Concurrent metadata reads deberán ser seguros.

---

# 256. Artifact integrity test

Metadata artifact modificado deberá detectarse si integrity verification está habilitada.

---

# 257. Secret test

No deberán encontrarse:

```text
DB_PASSWORD
tokens
private keys
```

en metadata artifacts.

---

# 258. Performance benchmark

Medir:

```text
cold discovery
cold compilation
artifact load
registry installation
warm lookup
memory usage
```

---

# 259. Expected performance objective

En warm runtime:

```text
Entity metadata lookup
≈
direct frozen registry access
```

sin:

```text
filesystem
Redis
reflection
attributes
database introspection
```

en el hot path ordinario.

---

# 260. Architectural invariants

## DB-MCACHE-001
Metadata Cache almacenará conocimiento estructural reutilizable.

## DB-MCACHE-002
Metadata Cache no almacenará query results.

## DB-MCACHE-003
Metadata Cache no almacenará managed entities.

## DB-MCACHE-004
Metadata Cache no sustituirá IdentityMap.

## DB-MCACHE-005
Metadata Cache no almacenará UnitOfWork state.

## DB-MCACHE-006
Metadata Cache no almacenará TransactionContext.

## DB-MCACHE-007
Compiled metadata será immutable.

## DB-MCACHE-008
Shared compiled metadata será profundamente immutable.

## DB-MCACHE-009
Runtime mutable state no se introducirá en compiled metadata.

## DB-MCACHE-010
Metadata source será distinta de compiled metadata.

## DB-MCACHE-011
Discovery será distinta de compilation.

## DB-MCACHE-012
Metadata Registry será distinta de Metadata Cache.

## DB-MCACHE-013
Registry será runtime authoritative source.

## DB-MCACHE-014
Cache será mecanismo de reutilización.

## DB-MCACHE-015
Hot path evitará reflection cuando exista metadata compilada.

## DB-MCACHE-016
Hot path evitará attribute scanning cuando exista metadata compilada.

## DB-MCACHE-017
Metadata identity será determinista.

## DB-MCACHE-018
Metadata fingerprints serán deterministas.

## DB-MCACHE-019
Request identifiers no participarán en structural fingerprints.

## DB-MCACHE-020
Current timestamps no participarán arbitrariamente en structural fingerprints.

## DB-MCACHE-021
Metadata generation separará deployments incompatibles.

## DB-MCACHE-022
Metadata generation no será equivalente a timestamp.

## DB-MCACHE-023
Compiler version podrá invalidar metadata.

## DB-MCACHE-024
Payload version podrá invalidar metadata serializada.

## DB-MCACHE-025
Old generation no requerirá global cache flush.

## DB-MCACHE-026
Blue-green deployments podrán coexistir con generaciones diferentes.

## DB-MCACHE-027
Entity metadata no contendrá entity instances.

## DB-MCACHE-028
Relationship metadata no contendrá relationship runtime state.

## DB-MCACHE-029
Type metadata no contendrá converted application values.

## DB-MCACHE-030
Schema metadata cache no será considerada automáticamente DB truth.

## DB-MCACHE-031
Schema metadata preservará introspection coverage.

## DB-MCACHE-032
PARTIAL no será convertido a COMPLETE.

## DB-MCACHE-033
UNKNOWN no será convertido a COMPLETE.

## DB-MCACHE-034
NotObserved no será convertido a Absent.

## DB-MCACHE-035
Migration confirmada podrá invalidar schema metadata.

## DB-MCACHE-036
External schema mutation podrá volver stale metadata.

## DB-MCACHE-037
Platform version no sustituirá capability metadata.

## DB-MCACHE-038
Topology changes podrán invalidar capability metadata.

## DB-MCACHE-039
Connection credentials nunca serán metadata cache payload.

## DB-MCACHE-040
Metadata stability será explícita.

## DB-MCACHE-041
Build-stable metadata podrá precompilarse.

## DB-MCACHE-042
Runtime-discovered metadata será distinguida de offline metadata.

## DB-MCACHE-043
Production podrá operar con compiled metadata artifacts.

## DB-MCACHE-044
Build artifact será versionado.

## DB-MCACHE-045
Build artifact podrá verificarse.

## DB-MCACHE-046
Metadata dependencies serán explícitas.

## DB-MCACHE-047
Dependency changes podrán invalidar dependents.

## DB-MCACHE-048
Dependency cycles serán manejados de forma segura.

## DB-MCACHE-049
Cache entry existence no implicará compatibility.

## DB-MCACHE-050
Metadata deberá validarse antes de registry installation.

## DB-MCACHE-051
Corrupt metadata no será instalada parcialmente.

## DB-MCACHE-052
Incompatible metadata no será instalada.

## DB-MCACHE-053
Registry publication no expondrá estado parcialmente construido.

## DB-MCACHE-054
Frozen registry no aceptará hot mutation ordinaria.

## DB-MCACHE-055
Runtime registration después de freeze estará prohibido por default.

## DB-MCACHE-056
Development hot reload será tooling, no core runtime invariant.

## DB-MCACHE-057
Production favorecerá immutable deployment generations.

## DB-MCACHE-058
Generated code será trusted build artifact, no remote arbitrary code.

## DB-MCACHE-059
Remote cache payload no podrá convertirse arbitrariamente en executable code.

## DB-MCACHE-060
Unsafe deserialization estará prohibida.

## DB-MCACHE-061
Metadata codecs serán versionados.

## DB-MCACHE-062
Metadata consumers dependerán de registries/resolvers.

## DB-MCACHE-063
Metadata consumers no dependerán directamente de Redis.

## DB-MCACHE-064
Metadata consumers no dependerán directamente de filesystem cache.

## DB-MCACHE-065
Cache backend será reemplazable.

## DB-MCACHE-066
In-memory metadata será bounded.

## DB-MCACHE-067
Filesystem cache podrá utilizarse para build artifacts.

## DB-MCACHE-068
Distributed cache será opcional.

## DB-MCACHE-069
Frozen registry podrá servir como hot L1.

## DB-MCACHE-070
No se añadirá L1 redundante sin beneficio.

## DB-MCACHE-071
Cold boot podrá cargar metadata compilada.

## DB-MCACHE-072
Warm requests usarán registry lookup.

## DB-MCACHE-073
Cache failure podrá degradar a compilation cuando policy lo permita.

## DB-MCACHE-074
COMPILED_ONLY no hará fallback silencioso.

## DB-MCACHE-075
COMPILED_ONLY missing artifact será error.

## DB-MCACHE-076
DISCOVER_AND_CACHE permitirá discovery.

## DB-MCACHE-077
PREFER_COMPILED favorecerá artifacts compilados.

## DB-MCACHE-078
Compilation failure no publicará metadata incompleta.

## DB-MCACHE-079
Negative metadata cache será bounded.

## DB-MCACHE-080
Negative metadata cache no será indefinida.

## DB-MCACHE-081
Extension registrations ocurrirán antes de freeze.

## DB-MCACHE-082
Extension conflicts no usarán last-write-wins.

## DB-MCACHE-083
Extension set compatibility será validada.

## DB-MCACHE-084
Package version por sí sola no será necesariamente semantic fingerprint.

## DB-MCACHE-085
Metadata cache será considerada superficie de integridad.

## DB-MCACHE-086
Secrets no serán almacenados.

## DB-MCACHE-087
Metadata diagnostics serán redactables.

## DB-MCACHE-088
Metrics evitarán labels de alta cardinalidad.

## DB-MCACHE-089
Telemetry no deberá registrar metadata completa indiscriminadamente.

## DB-MCACHE-090
Warm metadata lookup deberá evitar network I/O ordinario cuando esté en registry.

## DB-MCACHE-091
Metadata compilation deberá ser reproducible.

## DB-MCACHE-092
Environment name no sustituirá semantic configuration fingerprint.

## DB-MCACHE-093
Unrelated configuration no invalidará metadata innecesariamente.

## DB-MCACHE-094
Unknown dependency precision favorecerá broader invalidation.

## DB-MCACHE-095
Query Cache podrá depender de metadata generations.

## DB-MCACHE-096
Result Cache podrá depender de metadata generations.

## DB-MCACHE-097
Entity Cache deberá depender de entity/type metadata compatible.

## DB-MCACHE-098
Hydration Plan Cache permanecerá separado.

## DB-MCACHE-099
Persistence Plan permanecerá separado.

## DB-MCACHE-100
Query Plan permanecerá separado.

## DB-MCACHE-101
Mapping metadata podrá ser compartida.

## DB-MCACHE-102
Current persistence changes no serán compartidos.

## DB-MCACHE-103
Relationship fetch defaults serán metadata.

## DB-MCACHE-104
Relationship loaded state no será metadata.

## DB-MCACHE-105
Optimistic lock field mapping será metadata.

## DB-MCACHE-106
Current optimistic version value no será metadata.

## DB-MCACHE-107
Platform lock capabilities podrán ser metadata.

## DB-MCACHE-108
Current acquired locks no serán metadata.

## DB-MCACHE-109
Supported isolation capabilities podrán ser metadata.

## DB-MCACHE-110
Current effective transaction isolation no será metadata.

## DB-MCACHE-111
Static shard definitions podrán ser metadata.

## DB-MCACHE-112
Current shard routing result no será metadata.

## DB-MCACHE-113
Current replica lag no será structural metadata.

## DB-MCACHE-114
Persistent workers podrán compartir immutable metadata entre requests.

## DB-MCACHE-115
Persistent workers no compartirán mutable ORM state mediante metadata.

## DB-MCACHE-116
FrankenPHP workers reutilizarán frozen metadata safely.

## DB-MCACHE-117
RoadRunner workers reutilizarán frozen metadata safely.

## DB-MCACHE-118
OpenSwoole metadata compartida será concurrency-safe.

## DB-MCACHE-119
Readonly superficial no será suficiente si existen hijos mutables.

## DB-MCACHE-120
Compiled metadata deberá aspirar a deep immutability.

## DB-MCACHE-121
Metadata cache clear no realizará FLUSHALL.

## DB-MCACHE-122
Metadata namespaces serán application-scoped.

## DB-MCACHE-123
Metadata generations serán deployment-safe.

## DB-MCACHE-124
Cache corruption será observable.

## DB-MCACHE-125
Cache miss ordinario no será exception en modos no estrictos.

## DB-MCACHE-126
Compilation telemetry será bounded.

## DB-MCACHE-127
Metadata cache será una optimización removible cuando discovery esté disponible.

## DB-MCACHE-128
La ausencia del cache no cambiará la semántica del mapping.

## DB-MCACHE-129
Metadata Cache nunca generará SQL.

## DB-MCACHE-130
Metadata Cache nunca ejecutará queries de aplicación.

## DB-MCACHE-131
Metadata Cache nunca persistirá entities.

## DB-MCACHE-132
Metadata Cache nunca hará commit/rollback.

## DB-MCACHE-133
Metadata Cache nunca decidirá autorización.

## DB-MCACHE-134
Metadata Cache nunca resolverá authenticated user state.

## DB-MCACHE-135
Metadata Cache nunca se utilizará como service container.

## DB-MCACHE-136
Metadata Contributor no dependerá de HTTP request.

## DB-MCACHE-137
Metadata Contributor no dependerá de EntityManager mutable.

## DB-MCACHE-138
Metadata compilation podrá ejecutarse desde CLI.

## DB-MCACHE-139
Metadata compilation podrá ejecutarse en CI.

## DB-MCACHE-140
Metadata compilation podrá ejecutarse durante build.

## DB-MCACHE-141
Metadata compilation podrá ejecutarse en worker boot cuando policy lo permita.

## DB-MCACHE-142
Mismos semantic inputs producirán misma metadata.

## DB-MCACHE-143
Cache artifacts incompatibles serán rechazados antes del hot path.

## DB-MCACHE-144
Structural changes deberán propagarse a dependent cache fingerprints.

## DB-MCACHE-145
Cache invalidation será granular cuando exista evidencia suficiente.

## DB-MCACHE-146
Falsa precisión de dependency invalidation estará prohibida.

## DB-MCACHE-147
Security e integridad tendrán prioridad sobre cache hit rate.

## DB-MCACHE-148
Metadata cache no será fuente de application business data.

## DB-MCACHE-149
Database schema truth no será sustituida permanentemente por un snapshot cacheado.

## DB-MCACHE-150
La metadata compartida será conocimiento, nunca estado operacional mutable.

---

# 261. Modelo conceptual final

```text
                  DATABASE METADATA SYSTEM

PHP Attributes ────────────┐
Configuration ─────────────┤
Extensions ────────────────┤
Schema Introspection ──────┤
Platform Discovery ────────┘
             │
             ▼
      Metadata Discovery
             │
             ▼
        Normalization
             │
             ▼
         Validation
             │
             ▼
    Dependency Resolution
             │
             ▼
        Compilation
             │
             ▼
        Fingerprinting
             │
             ▼
       Metadata Cache
             │
             ▼
       Frozen Registries
             │
     ┌───────┼────────┬───────────┐
     ▼       ▼        ▼           ▼
    ORM    Query    Schema     Hydration
            │
            ▼
        Persistence
```

---

# 262. Build/runtime separation

```text
BUILD / BOOT
────────────────────────────────────────

Discover
Normalize
Validate
Compile
Fingerprint
Cache
Freeze

                 │
                 ▼

RUNTIME HOT PATH
────────────────────────────────────────

Registry Lookup
       ↓
Compiled Immutable Metadata
```

Objetivo:

```text
Reflection
Attribute scanning
Mapping compilation
Dependency resolution
```

fuera del hot path ordinario.

---

# 263. Fórmula de metadata compatible

Para un artifact `M` y runtime `R`:

```text
Compatible(M, R)
=
IdentityCompatible(M, R)
∧ GenerationCompatible(M, R)
∧ CompilerCompatible(M, R)
∧ PayloadCompatible(M, R)
∧ ConfigurationCompatible(M, R)
∧ DependenciesCompatible(M, R)
∧ ExtensionSetCompatible(M, R)
```

Solo entonces:

```text
MetadataCacheHit = TRUE
```

---

# 264. Metadata Cache Hit

Por tanto:

```text
CacheEntryExists
≠
MetadataCacheHit
```

Sino:

```text
MetadataCacheHit
=
EntryExists
+
IntegrityValid
+
VersionCompatible
+
GenerationCompatible
+
DependenciesCompatible
+
SemanticCompatibility
```

---

# 265. Modelo de ownership

```text
Metadata Compiler
→ creates metadata

Metadata Cache
→ stores metadata

Metadata Registry
→ exposes runtime metadata

ORM / Query / Schema
→ consume metadata
```

Nunca:

```text
Metadata Cache
→ becomes ORM
```

---

# 266. Relación con el cache architecture

```text
VoltStack Database Cache
│
├── Query Cache
│   └── reusable query-processing work
│
├── Result Cache
│   └── reusable database result data
│
├── Metadata Cache
│   └── reusable structural knowledge
│
├── Entity Cache
│   └── reusable entity-oriented state
│
├── Cache Invalidation
│   └── dependency/change propagation
│
└── Cache Consistency
    └── correctness and visibility guarantees
```

---

# 267. Regla maestra final

> **Metadata Cache en VoltStack será una infraestructura para convertir descubrimiento, reflection, mappings, capabilities y otras formas de conocimiento estructural en artefactos compilados, deterministas, versionados e inmutables que puedan reutilizarse de forma segura entre operaciones y, cuando corresponda, entre workers y deployments.**

La separación fundamental será:

```text
Metadata
=
Knowledge About Structure
```

mientras:

```text
Result
=
Data Produced By Query
```

y:

```text
Entity State
=
Application Object State
```

Por tanto:

```text
Metadata Cache
≠
Result Cache
≠
Entity Cache
≠
IdentityMap
```

La arquitectura ideal de producción será:

```text
Source Metadata
      ↓
Build-Time Compilation
      ↓
Validated Metadata Artifact
      ↓
Deployment
      ↓
Worker Boot
      ↓
Frozen Registry
      ↓
O(1)-like Runtime Lookup
```

manteniendo fuera del hot path:

```text
reflection
attribute discovery
mapping normalization
dependency analysis
structural validation
compilation
```

siempre que la metadata pueda conocerse previamente.

---

# 268. Siguiente documento

```text
190_DATABASE_ENTITY_CACHE_SYSTEM.md
```

El siguiente documento definirá la capa de caché orientada a entidades y deberá resolver especialmente:

```text
Entity Cache
vs
Result Cache
vs
IdentityMap

EntityKey
Entity Cache Entry
Canonical Entity State
Entity Snapshot Representation
Field Coverage
Partial Entity State
Entity Metadata Generation
Version Fields
Optimistic Locking
Relationship Boundaries
Entity Cache Hydration
IdentityMap Reconciliation
Dirty Managed Entities
Transaction Visibility
Read-Your-Own-Writes
Replica Awareness
Tenant Isolation
Shard Isolation
Cache Invalidation
Entity Versioning
Negative Entity Cache
Second-Level Cache
Persistent Workers
Security
Consistency
Telemetry
```

con la regla inicial:

```text
IdentityMap
→ first-level, scope-local object identity

Entity Cache
→ second-level, cross-scope reusable entity data
```

sin convertir jamás el Entity Cache en un segundo `EntityManager`.