# 141_DATABASE_HYDRATION_CACHE_SYSTEM.md

# VoltStack Quantum Database
## Database Hydration Cache System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 141 — Database Hydration Cache System  
**Bloque:** 12 — Hydration  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Hydration Cache System` define la arquitectura responsable de reutilizar los artefactos compilados necesarios para transformar resultados físicos de base de datos en resultados lógicos de VoltStack.

Su objetivo principal es evitar que cada ejecución tenga que repetir:

```text
metadata resolution
type resolution
result-slot binding
field-writer compilation
constructor resolution
identifier-reader compilation
relationship-plan compilation
tuple-plan compilation
grouping-plan compilation
deduplication-plan compilation
lifecycle-finalization analysis
```

El sistema almacenará principalmente:

```text
CompiledHydrationPlan
```

y otros artefactos inmutables relacionados.

Nunca deberá convertirse en un caché de datos.

---

# 2. Regla maestra

> **Hydration Cache almacena cómo hidratar; nunca almacena lo hidratado.**

Formalmente:

```text
HydrationCache
=
Cache<HydrationPlanCacheKey, CompiledHydrationPlan>
```

y nunca:

```text
HydrationCache
=
Cache<EntityKey, Entity>
```

ni:

```text
HydrationCache
=
Cache<Query, Result>
```

---

# 3. Distinciones fundamentales

```text
Hydration Cache
≠
Entity Cache

Hydration Cache
≠
Result Cache

Hydration Cache
≠
Query Cache

Hydration Cache
≠
Metadata Cache

Hydration Cache
≠
IdentityMap

Hydration Cache
≠
UnitOfWork

Hydration Cache
≠
Prepared Statement Cache

Hydration Cache
≠
Compiled Query Cache
```

Cada uno resuelve un problema diferente.

---

# 4. Qué se cachea

Artefactos válidos:

```text
CompiledHydrationPlan
CompiledEntityHydrationPlan
CompiledScalarHydrationPlan
CompiledTupleHydrationPlan
CompiledProjectionHydrationPlan
CompiledDtoHydrationPlan

CompiledIdentifierReader
CompiledFieldWriter
CompiledValueConverterBinding
CompiledEntityInstantiator
CompiledDtoInstantiator

CompiledGroupingPlan
CompiledDeduplicationPlan
CompiledLifecycleFinalizationPlan

AliasLookupMap
ResultSlotBindingMap
LoadedFieldMask
```

siempre que sean:

```text
immutable
deterministic
runtime-shareable
generation-bound
```

---

# 5. Qué nunca se cachea

Prohibido almacenar:

```text
Entity
EntityCollection
ResultTuple containing live entities
Result
ResultCursor
PDOStatement
Connection
Transaction
EntityManager
PersistenceContext
IdentityMap
UnitOfWork
SnapshotRegistry
ChangeSet
HydrationSession
HydrationContext
current tenant context
current request
current user
current coroutine
```

---

# 6. Arquitectura general

```text
                  Hydration Request
                         │
                         ▼
                HydrationPlanDefinition
                         │
                         ▼
                    Normalize
                         │
                         ▼
                  Fingerprint
                         │
                         ▼
              HydrationPlanCacheKey
                         │
                         ▼
               ┌─────────────────┐
               │ Hydration Cache │
               └────────┬────────┘
                        │
               ┌────────┴────────┐
               │                 │
              HIT               MISS
               │                 │
               ▼                 ▼
      CompiledHydrationPlan   Validate
               │                 │
               │                 ▼
               │              Optimize
               │                 │
               │                 ▼
               │              Compile
               │                 │
               │                 ▼
               │        CompiledHydrationPlan
               │                 │
               │                 ▼
               │              Admit?
               │                 │
               │          ┌──────┴──────┐
               │          │             │
               │         YES            NO
               │          │             │
               │          ▼             │
               │       Cache Put        │
               │          │             │
               └──────────┴─────────────┘
                          │
                          ▼
                      Hydrator
```

---

# 7. Cache lookup

La operación conceptual será:

```php
$plan = $cache->get($key);

if ($plan === null) {
    $plan = $compiler->compile($definition, $context);

    if ($admissionPolicy->allows($plan)) {
        $cache->put($key, $plan);
    }
}
```

La implementación real deberá incorporar:

```text
compatibility checks
concurrency coordination
generation checks
memory governance
telemetry
```

---

# 8. Contrato principal

```php
interface HydrationPlanCache
{
    public function get(
        HydrationPlanCacheKey $key,
    ): ?CompiledHydrationPlan;

    public function put(
        HydrationPlanCacheKey $key,
        CompiledHydrationPlan $plan,
    ): void;

    public function forget(
        HydrationPlanCacheKey $key,
    ): void;

    public function clear(): void;
}
```

---

# 9. Cache key

La clave deberá representar la compatibilidad estructural del plan.

```php
final readonly class HydrationPlanCacheKey
{
    public function __construct(
        public HydrationPlanFingerprint $fingerprint,
        public MetadataGenerationId $metadataGeneration,
        public TypeRegistryGenerationId $typeGeneration,
        public HydrationCompilerVersion $compilerVersion,
        public ?HydrationExtensionGenerationId $extensionGeneration = null,
    ) {}
}
```

---

# 10. Regla de identidad del caché

Dos planes podrán compartir entrada únicamente cuando:

```text
SameHydrationCacheEntry(A, B)
⇔
SemanticallyCompatible(A, B)
```

No basta con:

```text
same SQL string
```

---

# 11. SQL ≠ Hydration Cache Key

El mismo SQL puede hidratarse como:

```text
array
scalar
DTO
entity
tuple
projection
```

Por tanto:

```text
SameSQL
⇏
SameHydrationPlan
```

---

# 12. Query ≠ Hydration Shape

Ejemplo:

```sql
SELECT id, name FROM users
```

podría producir:

```text
User
UserSummaryDTO
array{id:int,name:string}
tuple<int,string>
```

Cada shape necesita semántica diferente.

---

# 13. HydrationPlanFingerprint

Debe representar la estructura lógica relevante.

Conceptualmente:

```text
Fingerprint
=
Hash(
    HydrationShape
    +
    ResultLayout
    +
    IdentifierBindings
    +
    FieldBindings
    +
    TypeBindings
    +
    ConstructionStrategies
    +
    RelationshipPlans
    +
    GroupingPlan
    +
    DeduplicationPlan
    +
    CardinalityPlan
    +
    StreamingPlan
    +
    LifecyclePlan
)
```

---

# 14. Determinismo

Mismo plan semántico bajo mismas generaciones:

```text
Fingerprint(A)
=
Fingerprint(B)
```

---

# 15. No incluir runtime state

Nunca incluir directamente:

```text
request ID
EntityManager object ID
IdentityMap object ID
transaction ID
current user ID
current tenant object ID
coroutine ID
```

---

# 16. Tenant awareness

El tenant no deberá formar parte de la key cuando únicamente cambia:

```text
runtime identity namespace
```

y no cambia la estructura de hidratación.

Pero si diferentes tenants poseen mappings estructuralmente distintos:

```text
Tenant A → schema/mapping generation 10
Tenant B → schema/mapping generation 27
```

la diferencia deberá reflejarse mediante una generación o namespace estructural apropiado.

---

# 17. Principio

```text
RuntimeTenantIdentity
≠
HydrationPlanStructure
```

salvo cuando la configuración del tenant cambie realmente esa estructura.

---

# 18. Metadata Generation

Todo plan dependiente de ORM metadata deberá estar ligado a:

```text
MetadataGenerationId
```

---

# 19. Cambio de metadata

Por ejemplo:

```text
User.name → User.fullName
```

puede invalidar planes anteriores.

---

# 20. Type Registry Generation

Si cambia:

```text
DECIMAL converter
UUID converter
JSON converter
Enum converter
```

los planes dependientes deberán invalidarse.

---

# 21. Extension Generation

Si una extensión cambia la compilación:

```text
custom hydrator
custom value object
custom tuple strategy
```

deberá cambiar:

```text
HydrationExtensionGenerationId
```

o una dependencia equivalente.

---

# 22. Compiler Version

Una actualización de VoltStack puede cambiar el formato o semántica de planes compilados.

Por ello:

```text
HydrationCompilerVersion
```

forma parte de compatibilidad.

---

# 23. Platform dependency

Un plan puede ser:

```text
PLATFORM_INDEPENDENT
```

o:

```text
PLATFORM_BOUND
```

---

# 24. Platform-bound plans

Cuando la representación física dependa de plataforma:

```text
MySQL
PostgreSQL
SQLite
MariaDB
```

la compatibilidad deberá incluir:

```text
PlatformCapabilityFingerprint
```

o representación equivalente.

---

# 25. Version ≠ Capability

No se deberá implementar:

```php
if ($mysqlVersion >= ...)
```

dentro del caché.

Debe utilizarse el modelo de capacidades ya definido.

---

# 26. Cache namespace

La arquitectura deberá soportar namespaces.

```text
HydrationCacheNamespace
```

---

# 27. Ejemplo

```text
voltstack.database.hydration.v1
```

---

# 28. Namespace components

Puede incluir:

```text
framework version
plan format version
application build ID
environment generation
```

---

# 29. Namespace ≠ Tenant

No utilizar un namespace por tenant salvo necesidad estructural real.

---

# 30. Cache hierarchy

VoltStack podrá implementar:

```text
L0 Request-local lookup
L1 Worker memory
L2 Shared application cache
L3 Persistent precompiled artifact store
```

pero no todos son obligatorios.

---

# 31. V1 recomendada

```text
L1 Worker Memory
```

como mecanismo principal.

Esto encaja especialmente bien con:

```text
FrankenPHP
```

---

# 32. L0 cache

Puede ser útil para evitar búsquedas repetidas dentro de una operación.

Pero:

```text
L0
```

debe ser scoped.

---

# 33. L1 worker cache

```text
worker process
└── immutable hydration plans
```

compartidos entre requests.

---

# 34. L2 shared cache

Podría utilizar:

```text
APCu-like shared memory
framework cache abstraction
specialized shared plan cache
```

solo si el formato es seguro para compartir.

---

# 35. L3 persistent cache

Futuro:

```text
disk
precompiled PHP artifact
binary artifact
deployment artifact
```

---

# 36. Cache tiers

```text
Hydration request
      │
      ▼
     L0
      │ miss
      ▼
     L1
      │ miss
      ▼
     L2
      │ miss
      ▼
     L3
      │ miss
      ▼
   Compile
```

---

# 37. Not mandatory complexity

VoltStack V1 no necesita implementar cuatro niveles.

La arquitectura debe permitir evolucionar hacia ellos.

---

# 38. Recommended V1

```text
Request Lookup
+
Worker Memory Cache
```

con:

```text
optional warmup
```

---

# 39. Cache admission

No todo plan compilado necesita entrar al caché.

---

# 40. Admission Policy

```php
interface HydrationCacheAdmissionPolicy
{
    public function shouldCache(
        CompiledHydrationPlan $plan,
        HydrationCacheAdmissionContext $context,
    ): bool;
}
```

---

# 41. Possible criteria

```text
plan cacheability
estimated size
reuse count
shape complexity
memory pressure
custom runtime dependencies
```

---

# 42. Standard plans

Planes estándar ORM deberán ser:

```text
CACHEABLE
```

por defecto.

---

# 43. Request-scoped plans

Si una extensión captura legítimamente una transformación scoped:

```text
REQUEST_SCOPED
```

no podrá entrar al worker cache.

---

# 44. Non-cacheable plans

```text
NON_CACHEABLE
```

se compilan/usan sin persistir.

---

# 45. Cacheability enum

```php
enum HydrationPlanCacheability
{
    case CACHEABLE;
    case REQUEST_SCOPED;
    case NON_CACHEABLE;
}
```

---

# 46. Immutability requirement

Un plan solo podrá entrar al shared cache cuando:

```text
Immutable(plan)
=
true
```

---

# 47. Deep immutability

No basta con:

```php
readonly class
```

si internamente referencia servicios mutables.

---

# 48. Required property

```text
DeepImmutable(plan)
```

o equivalente contractual.

---

# 49. Shared service references

Un plan puede referenciar objetos compartidos únicamente si también son:

```text
immutable
stateless
thread/coroutine safe
```

---

# 50. Example

Permitido:

```text
Immutable DecimalConverter
```

No:

```text
Converter containing current timezone mutable state
```

---

# 51. Context-dependent conversion

Debe recibir contexto durante ejecución o utilizar una policy immutable.

No capturar request state en compilación.

---

# 52. Cache hit

Un hit no significa simplemente:

```text
key exists
```

Debe significar:

```text
entry exists
∧
entry valid
∧
entry compatible
∧
entry not corrupted
```

---

# 53. Cache hit formula

```text
CacheHit
=
EntryFound
∧
GenerationCompatible
∧
FormatCompatible
∧
IntegrityValid
```

---

# 54. Stale entry

Una entrada stale deberá comportarse como:

```text
MISS
```

o ser eliminada.

Nunca deberá ejecutarse silenciosamente.

---

# 55. StaleHydrationPlanException

Puede utilizarse en modo estricto/debug.

En producción normalmente:

```text
stale
→ invalidate
→ compile
```

---

# 56. Compatibility checker

```php
interface HydrationPlanCompatibilityChecker
{
    public function check(
        CompiledHydrationPlan $plan,
        HydrationRuntimeDescriptor $runtime,
    ): HydrationPlanCompatibility;
}
```

---

# 57. Compatibility result

```php
enum HydrationPlanCompatibility
{
    case COMPATIBLE;
    case STALE;
    case FORMAT_INCOMPATIBLE;
    case PLATFORM_INCOMPATIBLE;
    case EXTENSION_INCOMPATIBLE;
    case UNKNOWN;
}
```

---

# 58. UNKNOWN

Nunca:

```text
UNKNOWN
→ COMPATIBLE
```

por defecto.

---

# 59. Conservative policy

```text
UNKNOWN
→ MISS / RECOMPILE
```

---

# 60. Cache invalidation

Invalidación deberá ser generation-driven.

---

# 61. Metadata invalidation

```text
MetadataGeneration
10
↓
11
```

Los planes ligados a `10` dejan de ser candidatos válidos.

---

# 62. No eager global deletion required

No siempre será necesario recorrer todo el caché.

Puede bastar con:

```text
generation mismatch
→ inaccessible stale entry
```

y eviction posterior.

---

# 63. Generational cache

Ejemplo:

```text
generation/84/...
generation/85/...
```

---

# 64. Advantage

Permite invalidación O(1) lógica:

```text
CurrentGeneration = 85
```

---

# 65. Old entries

Se eliminan mediante:

```text
eviction
background maintenance
worker restart
deployment cleanup
```

---

# 66. No stale execution

Aunque físicamente permanezcan en memoria.

---

# 67. Fine-grained invalidation

Futuro:

```text
EntityMetadataGeneration<User>
```

podría evitar invalidar planes no relacionados.

---

# 68. V1

Preferir:

```text
global metadata generation
```

por simplicidad y seguridad.

---

# 69. Development mode

En desarrollo:

```text
metadata changes
extensions changes
code changes
```

pueden ocurrir frecuentemente.

---

# 70. Development policy

```text
short-lived cache
generation validation
aggressive invalidation
```

---

# 71. Production policy

```text
immutable deployment
stable generations
aggressive reuse
prewarming
```

---

# 72. Deployment generation

Una aplicación puede tener:

```text
ApplicationBuildId
```

---

# 73. Build-scoped namespace

Ejemplo:

```text
hydration/build_20260907_01/...
```

facilita despliegues blue/green.

---

# 74. Warmup

VoltStack podrá precompilar planes frecuentes durante:

```text
application warmup
deployment
worker boot
```

---

# 75. Warmup sources

Por ejemplo:

```text
registered repositories
compiled entity queries
known model APIs
predefined projections
known DTO queries
```

---

# 76. Warmup ≠ execute query

Precalentar Hydration Plans no requiere consultar la base de datos si el layout puede conocerse durante compilación.

---

# 77. Warmup pipeline

```text
Known Query Shapes
       │
       ▼
Hydration Definitions
       │
       ▼
Normalize
       │
       ▼
Validate
       │
       ▼
Compile
       │
       ▼
Cache
```

---

# 78. Lazy compilation

Los planes no precalentados se compilan al primer uso.

---

# 79. Hybrid strategy

```text
hot known plans → warmup
rare plans      → lazy compilation
```

---

# 80. Cache stampede

En runtime concurrente, múltiples requests podrían fallar simultáneamente el mismo lookup.

---

# 81. Example

```text
Request A ─┐
Request B ─┼─ MISS same key
Request C ─┤
Request D ─┘
```

Sin coordinación:

```text
compile × 4
```

---

# 82. Single-flight compilation

VoltStack deberá permitir:

```text
one compiler
many waiters
```

para la misma key.

---

# 83. Contract

```php
interface HydrationPlanCompilationCoordinator
{
    public function resolve(
        HydrationPlanCacheKey $key,
        Closure $compiler,
    ): CompiledHydrationPlan;
}
```

---

# 84. Important

Single-flight coordination no deberá convertirse en lock global.

---

# 85. Key-scoped locking

Preferir:

```text
lock(cacheKey)
```

sobre:

```text
lock(allHydrationCompilation)
```

---

# 86. Double-check

Después de adquirir coordinación:

```text
check cache again
```

antes de compilar.

---

# 87. Algorithm

```text
lookup
 │
 ├─ HIT → return
 │
 └─ MISS
      │
      ▼
 acquire key compilation ownership
      │
      ▼
 lookup again
      │
      ├─ HIT → return
      │
      └─ MISS
           │
           ▼
         compile
           │
           ▼
        validate
           │
           ▼
          put
           │
           ▼
         publish
```

---

# 88. Compilation failure

Si el owner falla:

```text
waiters
```

deben recibir un outcome coherente.

---

# 89. No poisoned entry

Una compilación fallida nunca deberá introducir:

```text
partial compiled plan
```

al caché.

---

# 90. Negative caching

VoltStack podrá recordar temporalmente fallos deterministas.

Ejemplo:

```text
InvalidHydrationPlan
```

---

# 91. Purpose

Evitar recompilar miles de veces un plan estructuralmente inválido.

---

# 92. Negative caching constraints

Solo para errores:

```text
deterministic
structural
generation-bound
```

---

# 93. Do not negative-cache

Errores transitorios como:

```text
memory pressure
temporary extension unavailable
cancellation
deadline
```

no deberán quedar negativamente cacheados de forma larga.

---

# 94. Negative cache entry

```php
final readonly class NegativeHydrationCacheEntry
{
    public function __construct(
        public HydrationPlanCacheKey $key,
        public HydrationCompilationFailureCode $failure,
        public CacheExpiration $expiration,
    ) {}
}
```

---

# 95. No sensitive exception caching

No almacenar mensajes que puedan contener datos sensibles o stack traces completos.

---

# 96. Cache capacity

El caché deberá tener presupuesto explícito.

---

# 97. Capacity dimensions

```text
maximumEntries
maximumEstimatedBytes
maximumPlanBytes
```

---

# 98. Memory-first policy

En persistent workers es especialmente importante controlar:

```text
unbounded plan cardinality
```

---

# 99. Adversarial cardinality

Un usuario no deberá poder generar infinitas variantes de planes y agotar memoria.

---

# 100. Dynamic queries

Queries construidas dinámicamente pueden producir gran cantidad de fingerprints.

---

# 101. Protection

```text
admission policy
size limits
eviction
fingerprint normalization
per-shape governance
```

---

# 102. Eviction

Estrategias posibles:

```text
LRU
LFU
CLOCK
SEGMENTED_LRU
WEIGHTED_LRU
```

---

# 103. V1 recommendation

```text
Weighted LRU
```

o LRU simple con:

```text
entry count + memory budget
```

---

# 104. Why weighted

Un plan de:

```text
2 KB
```

no debería tener exactamente el mismo costo que uno de:

```text
500 KB
```

---

# 105. Plan weight

```text
PlanWeight
≈
EstimatedMemoryBytes
```

---

# 106. Memory estimator

```php
interface HydrationPlanMemoryEstimator
{
    public function estimate(
        CompiledHydrationPlan $plan,
    ): int;
}
```

---

# 107. Estimation

No necesita ser exacta byte a byte.

Debe ser suficientemente consistente para gobernanza.

---

# 108. Oversized plan

Si:

```text
PlanWeight > MaximumEntryWeight
```

podrá ejecutarse pero no cachearse.

---

# 109. Eviction ≠ invalidation

```text
Eviction
```

elimina por recursos.

```text
Invalidation
```

elimina por incompatibilidad.

---

# 110. Eviction ≠ correctness event

Evictar un plan válido solo afecta performance.

---

# 111. Invalidation = correctness concern

Usar un plan inválido sí puede afectar corrección.

---

# 112. Cache lifecycle

Estados conceptuales:

```text
ABSENT
   │
   ▼
COMPILING
   │
   ▼
READY
   │
   ├──► EVICTED
   │
   ├──► STALE
   │
   └──► INVALID
```

---

# 113. Cache entry

```php
final readonly class HydrationPlanCacheEntry
{
    public function __construct(
        public HydrationPlanCacheKey $key,
        public CompiledHydrationPlan $plan,
        public HydrationPlanCacheMetadata $metadata,
    ) {}
}
```

---

# 114. Cache metadata

Puede incluir:

```text
estimated size
creation timestamp
generation
format version
checksum
usage counters
```

---

# 115. Mutable usage counters

Si existen, deberán almacenarse fuera del immutable plan.

---

# 116. Important separation

```text
CompiledPlan
=
immutable

CacheEntryRuntimeStats
=
mutable cache infrastructure state
```

---

# 117. Plan checksum

Para cache persistente:

```text
Checksum
```

deberá validar integridad.

---

# 118. In-memory cache

Checksum puede ser innecesario.

---

# 119. Persistent cache

Debe validar:

```text
format
version
checksum
generation
compatibility
```

antes de utilizar artefacto.

---

# 120. Persistent plan serialization

Si se implementa:

```text
CompiledHydrationPlan
↓
SerializedHydrationPlan
```

el formato deberá ser explícitamente versionado.

---

# 121. Never serialize arbitrary closures

No se deberán persistir:

```text
anonymous closures
runtime service objects
PDO resources
Reflection objects tied to runtime state
```

sin una representación segura y reconstruible.

---

# 122. Serializable descriptors

Preferir:

```text
ConverterId
WriterId
InstantiatorId
TypeId
MetadataId
```

sobre objetos runtime arbitrarios.

---

# 123. Rehydrating the plan

Paradójicamente, un plan persistido puede requerir:

```text
artifact loading
```

pero no debe confundirse con ORM hydration.

---

# 124. Compiled PHP artifacts

Una estrategia futura:

```text
cache/hydration/
    hp_A18F.php
    hp_52BB.php
```

con código generado y validado.

---

# 125. Security

Archivos de caché deberán considerarse:

```text
trusted deployment artifacts
```

---

# 126. Cache poisoning protection

Nunca construir una cache key únicamente con valores controlados por usuario sin canonicalización estructural.

---

# 127. User input

Puede afectar valores de parámetros:

```text
WHERE email = ?
```

pero no debería crear un nuevo HydrationPlan.

---

# 128. Parameters ≠ Plan Shape

```text
email = alice@example.com
email = bob@example.com
```

deben reutilizar el mismo plan.

---

# 129. Critical anti-pattern

No:

```text
HydrationCacheKey
=
Hash(SQL + ParameterValues)
```

---

# 130. Correct direction

```text
HydrationCacheKey
=
Hash(ResultShape + StructuralBindings + Generations)
```

---

# 131. Sensitive values

Nunca deberán aparecer en:

```text
cache key
fingerprint
cache path
telemetry label
```

---

# 132. Class whitelist

Los plans que instancien entidades/DTOs deberán contener únicamente tipos resueltos desde metadata confiable.

---

# 133. Discriminator maps

Pueden cachearse si son:

```text
immutable whitelist maps
```

---

# 134. Example

```text
"user"  → User::class
"admin" → AdminUser::class
```

---

# 135. Never

```text
$row['class']
→ new $class
```

---

# 136. Cache poisoning via extensions

Las extensiones deberán participar antes de fingerprint final.

---

# 137. Rule

```text
ExtensionContribution
→ Normalize
→ Validate
→ Fingerprint
→ Compile
→ Cache
```

---

# 138. Not

```text
Cache Plan
→ mutate plan through extension
```

---

# 139. Post-cache mutation

Prohibida.

---

# 140. Cache invalidation API

```php
interface HydrationCacheInvalidator
{
    public function invalidateAll(): void;

    public function invalidateGeneration(
        MetadataGenerationId $generation,
    ): void;

    public function invalidatePlan(
        HydrationPlanFingerprint $fingerprint,
    ): void;
}
```

---

# 141. Internal vs public API

La invalidación granular puede ser interna inicialmente.

El desarrollador normal no debería tener que administrar manualmente el caché.

---

# 142. Framework lifecycle integration

Invalidación puede ocurrir durante:

```text
application rebuild
metadata recompilation
extension registry rebuild
development reload
deployment activation
```

---

# 143. No request-driven global clear

Una petición normal no deberá ejecutar:

```text
clear entire hydration cache
```

salvo operación administrativa explícita.

---

# 144. Cache consistency

El caché no necesita consistencia distribuida fuerte para corrección si:

```text
each entry independently validates its generations
```

---

# 145. Multi-worker environment

```text
Worker A → generation 84
Worker B → generation 84
```

pueden mantener caches físicos independientes.

---

# 146. Deployment transition

En deployment:

```text
old workers → build A
new workers → build B
```

cada build puede usar namespace diferente.

---

# 147. Blue/green deployment

Esto evita:

```text
new code
+
old hydration plan
```

---

# 148. FrankenPHP architecture

VoltStack tendrá FrankenPHP como runtime predeterminado.

Hydration Cache debe aprovechar el worker persistente.

---

# 149. FrankenPHP model

```text
Worker
├── Immutable Metadata
├── Type Registry
├── Hydration Plan Cache
│   ├── Plan A
│   ├── Plan B
│   └── Plan C
│
├── Request A
│   └── scoped HydrationSession
│
├── Request B
│   └── scoped HydrationSession
│
└── Request C
    └── scoped HydrationSession
```

---

# 150. Shared plans

Permitido:

```text
Plan A
```

para todas las requests compatibles.

---

# 151. Shared entities

Nunca.

```text
Entity(Request A)
≠
Entity(Request B)
```

aunque utilicen el mismo plan.

---

# 152. FrankenPHP request reset

No deberá vaciar el Hydration Plan Cache al terminar cada request.

---

# 153. Why

Eso eliminaría precisamente el beneficio del persistent runtime.

---

# 154. What is reset

Sí:

```text
HydrationSession
HydrationContext
temporary grouping state
temporary dedup state
current row state
current result
```

---

# 155. What survives

```text
CompiledHydrationPlan
immutable converters
immutable metadata
immutable writers
immutable instantiators
```

cuando sean seguros.

---

# 156. RoadRunner

Mismo principio:

```text
worker-scoped immutable cache
+
job/request-scoped mutable hydration state
```

---

# 157. OpenSwoole

El cache puede compartirse entre coroutines únicamente si sus entradas son realmente inmutables/concurrency-safe.

---

# 158. Coroutine state

Nunca dentro del plan.

---

# 159. Concurrency safety

El store deberá definir sus garantías.

---

# 160. Cache interface concurrency

Conceptualmente:

```text
ConcurrentReads = safe
ConcurrentMisses = coordinated
ConcurrentEviction = safe
ConcurrentGenerationChange = safe
```

---

# 161. PHP runtime implementations

La estrategia concreta dependerá del runtime.

No debe acoplarse el core a:

```text
FrankenPHP internals
RoadRunner internals
OpenSwoole internals
```

---

# 162. Runtime Adapter

Puede existir:

```php
interface HydrationCacheRuntimeAdapter
{
    public function createStore(
        HydrationCacheConfiguration $configuration,
    ): HydrationPlanCache;
}
```

---

# 163. Default implementation

```text
InMemoryHydrationPlanCache
```

---

# 164. Optional implementations

Futuras:

```text
PersistentHydrationPlanCache
SharedMemoryHydrationPlanCache
PrecompiledHydrationArtifactCache
```

---

# 165. No dependency on general Cache package

El core del Database System no debería requerir obligatoriamente:

```text
VoltStack/Quantum/Cache
```

para funcionar.

---

# 166. Important architectural rule

Hydration Cache es infraestructura interna de Database.

La integración con el Cache System general puede ser opcional.

---

# 167. Why

Evita dependencia circular:

```text
Database → Cache → Database
```

---

# 168. Cache package integration

Podrá existir mediante adapter:

```text
Database Hydration Cache
       │
       ▼
Cache Integration Adapter
       │
       ▼
VoltStack Cache
```

sin convertirlo en dependencia obligatoria.

---

# 169. Failure handling

El caché es una optimización.

Por tanto:

```text
CacheFailure
```

no debería impedir hidratación cuando sea posible recompilar.

---

# 170. Example

```text
cache read failure
→ compile directly
→ hydrate
```

---

# 171. But

Un artefacto corrupto no deberá ejecutarse.

---

# 172. Corruption

```text
corrupted entry
→ reject
→ invalidate
→ recompile
```

---

# 173. Cache write failure

Normalmente:

```text
plan compiled
cache put fails
```

puede continuar con el plan actual.

---

# 174. Outcome dimensions

```text
HydrationPlanOutcome = SUCCESS

HydrationCacheOutcome = WRITE_FAILED
```

pueden coexistir.

---

# 175. Cache outage

No deberá fingirse como:

```text
database outage
```

---

# 176. Error taxonomy

```text
DatabaseHydrationCacheException
├── HydrationCacheConfigurationException
├── HydrationCacheKeyException
├── HydrationCacheReadException
├── HydrationCacheWriteException
├── HydrationCacheInvalidationException
├── HydrationCacheCompatibilityException
├── StaleHydrationCacheEntryException
├── CorruptedHydrationCacheEntryException
├── HydrationCacheSerializationException
├── HydrationCacheDeserializationException
├── HydrationCacheVersionException
├── HydrationCacheChecksumException
├── HydrationCacheAdmissionException
├── HydrationCacheEvictionException
├── HydrationCacheCapacityException
├── HydrationCacheCompilationCoordinationException
├── HydrationCacheStampedeException
├── HydrationCacheSecurityException
├── HydrationCacheRuntimeIsolationException
└── HydrationCacheInvariantException
```

---

# 177. Failure policy

```php
enum HydrationCacheFailurePolicy
{
    case FAIL_OPEN;
    case FAIL_CLOSED;
    case STRICT;
}
```

---

# 178. FAIL_OPEN

Para errores puramente de infraestructura del caché:

```text
bypass cache
→ compile
→ continue
```

---

# 179. FAIL_CLOSED

Para errores que puedan comprometer:

```text
integrity
security
semantic correctness
```

---

# 180. Example

Checksum inválido:

```text
do not execute artifact
```

---

# 181. STRICT

Útil en testing/debug para hacer visibles todos los problemas.

---

# 182. Cache observability

Debe medirse sin alta cardinalidad innecesaria.

---

# 183. Metrics

```text
orm.hydration.cache.hit
orm.hydration.cache.miss
orm.hydration.cache.stale
orm.hydration.cache.invalid
orm.hydration.cache.corrupt

orm.hydration.cache.put
orm.hydration.cache.eviction
orm.hydration.cache.recompile

orm.hydration.cache.entries
orm.hydration.cache.memory.bytes

orm.hydration.cache.lookup.duration
orm.hydration.cache.compile.duration

orm.hydration.cache.singleflight.wait
orm.hydration.cache.singleflight.owner

orm.hydration.cache.negative.hit
orm.hydration.cache.negative.put
```

---

# 184. Hit ratio

```text
HitRatio
=
Hits
/
(Hits + Misses)
```

---

# 185. Compilation avoidance ratio

```text
CompilationAvoidance
=
CacheHits
/
HydrationPlanRequests
```

---

# 186. Eviction rate

```text
EvictionRate
=
Evictions
/
CacheInsertions
```

---

# 187. Stale ratio

Un stale ratio alto puede indicar:

```text
unstable metadata generations
development reload
incorrect generation design
```

---

# 188. Cache pressure

Puede observarse mediante:

```text
current bytes
max bytes
eviction frequency
oversized rejection
```

---

# 189. No sensitive metric labels

Nunca:

```text
user email
entity ID
tenant ID
SQL parameters
```

---

# 190. Fingerprints

No deberán utilizarse como metric labels por defecto.

---

# 191. Tracing

En debug/traces controlados sí puede registrarse:

```text
plan fingerprint prefix
cache tier
hit/miss
compile reason
```

---

# 192. Debug toolbar

Ejemplo:

```text
Hydration Plan Cache
────────────────────────────────────

Lookups:              1,824
Hits:                 1,731
Misses:                  93
Hit Ratio:            94.9%

Compilations:            81
Single-flight saved:     12

Entries:                418
Estimated memory:      5.7 MB
Budget:               32.0 MB

Evictions:                4
Stale entries:            8
Corrupted entries:        0

Current:
  Metadata generation:   84
  Type generation:       11
  Compiler version:       1
```

---

# 193. Diagnostics

Para un miss:

```text
MISS_REASON:
  NOT_FOUND
  METADATA_GENERATION_CHANGED
  TYPE_GENERATION_CHANGED
  COMPILER_VERSION_CHANGED
  EXTENSION_GENERATION_CHANGED
  PLATFORM_INCOMPATIBLE
  EVICTED
  CORRUPTED
  NON_CACHEABLE
```

---

# 194. Explain cache key

En debug:

```text
HydrationCacheKey
├── shape fingerprint
├── metadata generation
├── type generation
├── extension generation
├── compiler version
└── platform fingerprint
```

sin revelar datos sensibles.

---

# 195. Configuration

Propuesta:

```php
'database' => [
    'hydration' => [
        'cache' => [
            'enabled' => true,

            'store' => 'memory',

            'max_entries' => 4096,

            'max_memory' => '64MB',

            'eviction' => 'weighted_lru',

            'warmup' => true,

            'single_flight' => true,

            'negative_cache' => true,

            'development_validation' => true,
        ],
    ],
],
```

---

# 196. Secure defaults

Default recomendado:

```text
enabled              = true
store                = worker_memory
bounded              = true
generation validation= true
single-flight         = true
persistent artifacts = false
negative cache       = short-lived
```

---

# 197. Persistent artifact cache

No habilitar por defecto en V1 hasta disponer de:

```text
safe serialization
format versioning
integrity validation
deployment namespacing
```

---

# 198. Memory budget

Debe existir incluso si el usuario no lo configura explícitamente.

---

# 199. Never unbounded by default

Especialmente bajo FrankenPHP.

---

# 200. Adaptive capacity

Futuro:

```text
worker memory pressure
→ reduce hydration cache budget
```

mediante `ResourceGovernance`.

---

# 201. No arbitrary GC

El caché no deberá ejecutar:

```text
gc_collect_cycles()
```

como política primaria.

Debe controlar referencias correctamente.

---

# 202. Reference safety

Cached plan no debe mantener referencias indirectas hacia:

```text
EntityManager
Request
Closure capturing request
Container scoped service
```

---

# 203. Cacheability validator

Antes de admitir un plan:

```php
interface HydrationPlanCacheabilityValidator
{
    public function validate(
        CompiledHydrationPlan $plan,
    ): HydrationPlanCacheabilityReport;
}
```

---

# 204. Development deep validation

Puede inspeccionar el graph del plan buscando referencias prohibidas.

---

# 205. Production

Debe confiar principalmente en contratos y tipos previamente validados.

---

# 206. Testing architecture

El sistema requiere pruebas independientes de hidratación funcional.

---

# 207. Basic hit test

```text
compile
put
get
→ same compatible plan
```

---

# 208. Miss test

Unknown key:

```text
→ null/miss
```

---

# 209. Metadata generation test

```text
plan generation = 10
runtime generation = 11
```

→ miss/stale.

---

# 210. Type generation test

Cambio de converter invalida plan.

---

# 211. Compiler version test

Plan V1 no se ejecuta con compiler format incompatible V2.

---

# 212. Extension generation test

Cambio de extension contribution invalida plan.

---

# 213. Platform test

Platform-bound plan no se reutiliza sobre plataforma incompatible.

---

# 214. Same SQL different shape test

```text
Entity
DTO
Scalar
Tuple
```

producen keys distintas.

---

# 215. Different parameter test

```text
WHERE id = 1
WHERE id = 2
```

reutilizan plan si el shape es igual.

---

# 216. Sensitive parameter test

Valor no aparece en cache key.

---

# 217. Entity leakage test

Cached plan no mantiene referencia a entidad hidratada.

---

# 218. EntityManager leakage test

Cached plan no retiene manager.

---

# 219. PersistenceContext leakage test

No retiene context.

---

# 220. Request leakage test

Request A termina.

Request B reutiliza plan.

Ningún state de A es visible.

---

# 221. FrankenPHP test

Miles de requests reutilizan el mismo plan sin crecimiento proporcional de entradas.

---

# 222. RoadRunner test

Mismo principio.

---

# 223. OpenSwoole test

Lectura concurrente del plan no modifica su estado.

---

# 224. Single-flight test

100 misses concurrentes para la misma key:

```text
compilation count ≈ 1
```

---

# 225. Different-key concurrency

Planes diferentes pueden compilarse independientemente.

---

# 226. Failed compilation test

No se publica entrada parcial.

---

# 227. Negative cache test

Plan estructuralmente inválido no se recompila continuamente.

---

# 228. Negative generation test

Cambio de generación invalida negative entry.

---

# 229. Cancellation test

Compilación cancelada no genera negative cache permanente.

---

# 230. Capacity test

Superar límite activa eviction.

---

# 231. Oversized entry test

Plan puede ejecutarse sin entrar al caché.

---

# 232. LRU test

Entradas menos utilizadas son candidatas apropiadas.

---

# 233. Memory weight test

Planes grandes consumen mayor weight.

---

# 234. Eviction correctness test

Plan evicted puede recompilarse sin cambio semántico.

---

# 235. Corruption test

Artefacto corrupto jamás se ejecuta.

---

# 236. Checksum test

Checksum incorrecto fuerza invalidación.

---

# 237. Persistent version test

Formato incompatible se rechaza.

---

# 238. Warmup test

Planes precargados producen hit en primera request.

---

# 239. Warmup failure test

Un plan inválido no impide necesariamente warmup de otros planes salvo política strict.

---

# 240. Deployment namespace test

Build A y Build B no comparten artefactos incompatibles.

---

# 241. Performance benchmarks

Escenarios mínimos:

```text
cold simple entity plan
warm simple entity plan

cold 20-field entity
warm 20-field entity

entity + to-many relationship
tuple hydration plan
DTO hydration plan
partial entity plan

1k unique plans
10k unique plans

90% cache hit
99% cache hit

high concurrent same-key misses
high concurrent different-key misses

LRU under memory pressure
```

---

# 242. Performance objective

En hit:

```text
HydrationPlanResolution
```

deberá acercarse a:

```text
hash/key lookup
+
minimal compatibility validation
```

---

# 243. No hot-path recompilation

Una query shape frecuente no debería recompilar repetidamente.

---

# 244. Compilation amortization

```text
AmortizedPlanCost
=
CompilationCost
/
ReuseCount
```

A medida que:

```text
ReuseCount → large
```

el costo amortizado se aproxima a cero.

---

# 245. Architectural Invariants

## DB-ORM-HYDRATION-CACHE-001

Hydration Cache almacenará cómo hidratar, no lo hidratado.

## DB-ORM-HYDRATION-CACHE-002

Hydration Cache será distinto de Entity Cache.

## DB-ORM-HYDRATION-CACHE-003

Hydration Cache será distinto de Result Cache.

## DB-ORM-HYDRATION-CACHE-004

Hydration Cache será distinto de Query Cache.

## DB-ORM-HYDRATION-CACHE-005

Hydration Cache será distinto de Metadata Cache.

## DB-ORM-HYDRATION-CACHE-006

Hydration Cache será distinto de IdentityMap.

## DB-ORM-HYDRATION-CACHE-007

Hydration Cache será distinto de UnitOfWork.

## DB-ORM-HYDRATION-CACHE-008

Hydration Cache será distinto de PreparedStatement Cache.

## DB-ORM-HYDRATION-CACHE-009

CompiledHydrationPlan será el principal artefacto cacheable.

## DB-ORM-HYDRATION-CACHE-010

Solo artefactos inmutables podrán entrar a caches compartidos.

## DB-ORM-HYDRATION-CACHE-011

Readonly superficial no garantizará cacheability.

## DB-ORM-HYDRATION-CACHE-012

Shared plan deberá ser profundamente runtime-safe.

## DB-ORM-HYDRATION-CACHE-013

Hydrated entities nunca serán cacheadas aquí.

## DB-ORM-HYDRATION-CACHE-014

Result nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-015

ResultCursor nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-016

EntityManager nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-017

PersistenceContext nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-018

IdentityMap nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-019

UnitOfWork nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-020

HydrationSession nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-021

HydrationContext nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-022

Current request nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-023

Current transaction nunca será cacheada aquí.

## DB-ORM-HYDRATION-CACHE-024

Current user nunca será cacheado aquí.

## DB-ORM-HYDRATION-CACHE-025

Cache key representará semántica estructural.

## DB-ORM-HYDRATION-CACHE-026

SQL string no será suficiente como cache key.

## DB-ORM-HYDRATION-CACHE-027

Parameter values no formarán parte de hydration cache key normal.

## DB-ORM-HYDRATION-CACHE-028

Same SQL podrá producir diferentes hydration plans.

## DB-ORM-HYDRATION-CACHE-029

Different parameter values podrán reutilizar mismo plan.

## DB-ORM-HYDRATION-CACHE-030

Hydration shape participará en fingerprint.

## DB-ORM-HYDRATION-CACHE-031

Result layout participará cuando sea estructuralmente relevante.

## DB-ORM-HYDRATION-CACHE-032

Identifier bindings participarán en fingerprint.

## DB-ORM-HYDRATION-CACHE-033

Type bindings participarán en fingerprint.

## DB-ORM-HYDRATION-CACHE-034

Construction strategies participarán cuando afecten semántica.

## DB-ORM-HYDRATION-CACHE-035

Grouping strategy participará cuando corresponda.

## DB-ORM-HYDRATION-CACHE-036

Deduplication strategy participará cuando corresponda.

## DB-ORM-HYDRATION-CACHE-037

Cardinality semantics participarán cuando corresponda.

## DB-ORM-HYDRATION-CACHE-038

Streaming semantics participarán cuando afecten el plan.

## DB-ORM-HYDRATION-CACHE-039

Lifecycle finalization semantics participarán cuando correspondan.

## DB-ORM-HYDRATION-CACHE-040

Fingerprint será determinista.

## DB-ORM-HYDRATION-CACHE-041

Fingerprint no incluirá mutable request state.

## DB-ORM-HYDRATION-CACHE-042

Fingerprint no incluirá sensitive parameter values.

## DB-ORM-HYDRATION-CACHE-043

Fingerprint no incluirá entity IDs de resultados.

## DB-ORM-HYDRATION-CACHE-044

MetadataGeneration gobernará compatibilidad de metadata.

## DB-ORM-HYDRATION-CACHE-045

TypeRegistryGeneration gobernará compatibilidad de tipos.

## DB-ORM-HYDRATION-CACHE-046

ExtensionGeneration podrá gobernar extensiones relevantes.

## DB-ORM-HYDRATION-CACHE-047

CompilerVersion gobernará formato/semántica de compilación.

## DB-ORM-HYDRATION-CACHE-048

Platform capabilities participarán solo cuando sean relevantes.

## DB-ORM-HYDRATION-CACHE-049

Version de DB no sustituirá capability model.

## DB-ORM-HYDRATION-CACHE-050

Tenant ID no entrará automáticamente en cache key.

## DB-ORM-HYDRATION-CACHE-051

Tenant-specific structural mapping deberá diferenciarse estructuralmente.

## DB-ORM-HYDRATION-CACHE-052

Cache hit requerirá compatibilidad, no mera presencia.

## DB-ORM-HYDRATION-CACHE-053

Stale plan nunca será ejecutado silenciosamente.

## DB-ORM-HYDRATION-CACHE-054

UNKNOWN compatibility no se tratará como compatible por defecto.

## DB-ORM-HYDRATION-CACHE-055

Generation mismatch producirá miss/invalidation.

## DB-ORM-HYDRATION-CACHE-056

Invalidation será distinta de eviction.

## DB-ORM-HYDRATION-CACHE-057

Eviction no cambiará semántica de hydration.

## DB-ORM-HYDRATION-CACHE-058

Invalidation protegerá corrección.

## DB-ORM-HYDRATION-CACHE-059

Generational invalidation podrá ser lógica.

## DB-ORM-HYDRATION-CACHE-060

Old generation entry podrá existir físicamente sin ser reutilizable.

## DB-ORM-HYDRATION-CACHE-061

V1 podrá usar global metadata generation.

## DB-ORM-HYDRATION-CACHE-062

Fine-grained generation será optimización futura.

## DB-ORM-HYDRATION-CACHE-063

Development mode podrá invalidar agresivamente.

## DB-ORM-HYDRATION-CACHE-064

Production mode favorecerá immutable reuse.

## DB-ORM-HYDRATION-CACHE-065

Application build podrá formar parte del namespace.

## DB-ORM-HYDRATION-CACHE-066

Blue/green builds no compartirán planes incompatibles.

## DB-ORM-HYDRATION-CACHE-067

Cache hierarchy será extensible.

## DB-ORM-HYDRATION-CACHE-068

Worker-memory cache será implementación V1 recomendada.

## DB-ORM-HYDRATION-CACHE-069

Shared persistent cache no será requisito de V1.

## DB-ORM-HYDRATION-CACHE-070

Cache admission será explícita.

## DB-ORM-HYDRATION-CACHE-071

Standard compiled plans deberán ser cacheables por defecto.

## DB-ORM-HYDRATION-CACHE-072

Request-scoped plan no entrará a worker cache.

## DB-ORM-HYDRATION-CACHE-073

Non-cacheable plan podrá ejecutarse normalmente.

## DB-ORM-HYDRATION-CACHE-074

Oversized plan podrá ejecutarse sin ser cacheado.

## DB-ORM-HYDRATION-CACHE-075

Cache capacity será bounded.

## DB-ORM-HYDRATION-CACHE-076

Worker cache no será unbounded por defecto.

## DB-ORM-HYDRATION-CACHE-077

Maximum entries podrá configurarse.

## DB-ORM-HYDRATION-CACHE-078

Memory budget podrá configurarse.

## DB-ORM-HYDRATION-CACHE-079

Per-entry size podrá gobernarse.

## DB-ORM-HYDRATION-CACHE-080

Dynamic query cardinality no podrá crecer sin control.

## DB-ORM-HYDRATION-CACHE-081

Eviction policy será determinista respecto a su política.

## DB-ORM-HYDRATION-CACHE-082

Plan weight podrá representar memory cost.

## DB-ORM-HYDRATION-CACHE-083

Cache statistics mutables no residirán dentro del immutable plan.

## DB-ORM-HYDRATION-CACHE-084

Warmup no ejecutará queries innecesariamente.

## DB-ORM-HYDRATION-CACHE-085

Warmup podrá ocurrir durante worker boot.

## DB-ORM-HYDRATION-CACHE-086

Lazy compilation seguirá soportada.

## DB-ORM-HYDRATION-CACHE-087

Warmup y lazy compilation podrán coexistir.

## DB-ORM-HYDRATION-CACHE-088

Concurrent same-key misses deberán poder coordinarse.

## DB-ORM-HYDRATION-CACHE-089

Single-flight será key-scoped.

## DB-ORM-HYDRATION-CACHE-090

Single-flight no será global lock por defecto.

## DB-ORM-HYDRATION-CACHE-091

Cache deberá reconsultarse después de adquirir compilation ownership.

## DB-ORM-HYDRATION-CACHE-092

Failed compilation no publicará partial plan.

## DB-ORM-HYDRATION-CACHE-093

Waiters observarán un outcome coherente.

## DB-ORM-HYDRATION-CACHE-094

Negative caching solo se aplicará a errores apropiados.

## DB-ORM-HYDRATION-CACHE-095

Cancellation no será negative-cached permanentemente.

## DB-ORM-HYDRATION-CACHE-096

Transient failure no será tratado como structural invalidity.

## DB-ORM-HYDRATION-CACHE-097

Negative entries estarán generation-bound.

## DB-ORM-HYDRATION-CACHE-098

Negative entries no almacenarán sensitive exception payloads.

## DB-ORM-HYDRATION-CACHE-099

Persistent cache format será versionado.

## DB-ORM-HYDRATION-CACHE-100

Persistent cache podrá utilizar checksums.

## DB-ORM-HYDRATION-CACHE-101

Corrupted persistent artifact nunca será ejecutado.

## DB-ORM-HYDRATION-CACHE-102

Format-incompatible artifact nunca será ejecutado.

## DB-ORM-HYDRATION-CACHE-103

Arbitrary runtime closures no serán serializadas por defecto.

## DB-ORM-HYDRATION-CACHE-104

Persistent descriptors preferirán IDs estables.

## DB-ORM-HYDRATION-CACHE-105

Generated PHP artifacts procederán de metadata validada.

## DB-ORM-HYDRATION-CACHE-106

Database values nunca producirán executable cache code.

## DB-ORM-HYDRATION-CACHE-107

Cache files serán deployment-trusted artifacts.

## DB-ORM-HYDRATION-CACHE-108

User input no podrá seleccionar clases arbitrarias mediante cached plan.

## DB-ORM-HYDRATION-CACHE-109

Discriminator maps cacheados serán whitelists.

## DB-ORM-HYDRATION-CACHE-110

Extensions contribuirán antes del fingerprint final.

## DB-ORM-HYDRATION-CACHE-111

Cached plan no será mutado posteriormente por extensions.

## DB-ORM-HYDRATION-CACHE-112

Extension conflicts deberán resolverse antes de caching.

## DB-ORM-HYDRATION-CACHE-113

Cache failure puramente operacional podrá ser fail-open.

## DB-ORM-HYDRATION-CACHE-114

Cache integrity failure será fail-closed respecto al artefacto.

## DB-ORM-HYDRATION-CACHE-115

Cache read failure no será database query failure.

## DB-ORM-HYDRATION-CACHE-116

Cache write failure no invalidará automáticamente compiled plan actual.

## DB-ORM-HYDRATION-CACHE-117

HydrationPlanOutcome será distinto de HydrationCacheOutcome.

## DB-ORM-HYDRATION-CACHE-118

Corruption producirá invalidate/recompile.

## DB-ORM-HYDRATION-CACHE-119

Cache no administrará database transactions.

## DB-ORM-HYDRATION-CACHE-120

Cache no ejecutará SQL.

## DB-ORM-HYDRATION-CACHE-121

Cache no abrirá Connections.

## DB-ORM-HYDRATION-CACHE-122

Cache no hará flush.

## DB-ORM-HYDRATION-CACHE-123

Cache no administrará entities.

## DB-ORM-HYDRATION-CACHE-124

Cache no alterará UnitOfWork.

## DB-ORM-HYDRATION-CACHE-125

Cache no alterará IdentityMap.

## DB-ORM-HYDRATION-CACHE-126

FrankenPHP podrá conservar cache entre requests.

## DB-ORM-HYDRATION-CACHE-127

FrankenPHP request cleanup no deberá borrar immutable plan cache.

## DB-ORM-HYDRATION-CACHE-128

FrankenPHP sí deberá limpiar hydration session state.

## DB-ORM-HYDRATION-CACHE-129

RoadRunner seguirá la misma separación worker/scoped state.

## DB-ORM-HYDRATION-CACHE-130

OpenSwoole solo compartirá concurrency-safe immutable plans.

## DB-ORM-HYDRATION-CACHE-131

Coroutine state nunca residirá dentro del shared plan.

## DB-ORM-HYDRATION-CACHE-132

Concurrent reads del shared cache deberán ser seguras según adapter.

## DB-ORM-HYDRATION-CACHE-133

Concurrent miss coordination deberá ser segura según runtime.

## DB-ORM-HYDRATION-CACHE-134

Runtime-specific cache behavior estará detrás de contracts/adapters.

## DB-ORM-HYDRATION-CACHE-135

Database core no dependerá obligatoriamente del Cache package general.

## DB-ORM-HYDRATION-CACHE-136

General Cache integration será opcional.

## DB-ORM-HYDRATION-CACHE-137

No se introducirá dependencia circular Database↔Cache.

## DB-ORM-HYDRATION-CACHE-138

Cache metrics no incluirán sensitive data.

## DB-ORM-HYDRATION-CACHE-139

Plan fingerprint no será metric label de alta cardinalidad por defecto.

## DB-ORM-HYDRATION-CACHE-140

Cache hit ratio será observable.

## DB-ORM-HYDRATION-CACHE-141

Cache memory usage será observable.

## DB-ORM-HYDRATION-CACHE-142

Eviction count será observable.

## DB-ORM-HYDRATION-CACHE-143

Stale-plan count será observable.

## DB-ORM-HYDRATION-CACHE-144

Compilation count será observable.

## DB-ORM-HYDRATION-CACHE-145

Single-flight savings podrán observarse.

## DB-ORM-HYDRATION-CACHE-146

Miss reasons serán diagnosticables.

## DB-ORM-HYDRATION-CACHE-147

Cache diagnostics no revelarán parameter values.

## DB-ORM-HYDRATION-CACHE-148

Plan cacheability será validable.

## DB-ORM-HYDRATION-CACHE-149

Cacheable plan no retendrá scoped service indirectamente.

## DB-ORM-HYDRATION-CACHE-150

Cacheable plan no retendrá request closure indirectamente.

## DB-ORM-HYDRATION-CACHE-151

Cacheable plan no retendrá entity graph indirectamente.

## DB-ORM-HYDRATION-CACHE-152

Request A y Request B podrán compartir plan sin compartir ORM state.

## DB-ORM-HYDRATION-CACHE-153

Same plan reuse no implicará same entity instance across requests.

## DB-ORM-HYDRATION-CACHE-154

Plan reuse no alterará IdentityMap scope.

## DB-ORM-HYDRATION-CACHE-155

Plan reuse no alterará tenant identity namespace runtime.

## DB-ORM-HYDRATION-CACHE-156

Plan reuse deberá preservar exactamente las mismas hydration semantics.

## DB-ORM-HYDRATION-CACHE-157

Cache optimization nunca debilitará type safety.

## DB-ORM-HYDRATION-CACHE-158

Cache optimization nunca debilitará identity safety.

## DB-ORM-HYDRATION-CACHE-159

Cache optimization nunca debilitará runtime isolation.

## DB-ORM-HYDRATION-CACHE-160

VoltStack nunca considerará válida una entrada del Hydration Cache únicamente porque exista: deberá ser estructuralmente compatible, generation-compatible, íntegra y segura para el runtime actual.

---

# 246. Directory Structure

```text
src/Quantum/Database/ORM/Hydration/Cache/
│
├── Contract/
│   ├── HydrationPlanCache.php
│   ├── HydrationCacheAdmissionPolicy.php
│   ├── HydrationCacheInvalidator.php
│   ├── HydrationPlanCompatibilityChecker.php
│   ├── HydrationPlanCompilationCoordinator.php
│   ├── HydrationPlanMemoryEstimator.php
│   ├── HydrationPlanCacheabilityValidator.php
│   └── HydrationCacheRuntimeAdapter.php
│
├── Model/
│   ├── HydrationPlanCacheKey.php
│   ├── HydrationPlanCacheEntry.php
│   ├── HydrationPlanCacheMetadata.php
│   ├── HydrationCacheNamespace.php
│   ├── HydrationCacheTier.php
│   ├── HydrationPlanCompatibility.php
│   ├── HydrationCacheMissReason.php
│   ├── HydrationCacheFailurePolicy.php
│   ├── HydrationPlanCacheability.php
│   └── HydrationPlanWeight.php
│
├── Memory/
│   ├── InMemoryHydrationPlanCache.php
│   ├── HydrationCacheEntryMap.php
│   └── HydrationCacheMemoryBudget.php
│
├── Admission/
│   ├── DefaultHydrationCacheAdmissionPolicy.php
│   └── BoundedHydrationCacheAdmissionPolicy.php
│
├── Eviction/
│   ├── HydrationCacheEvictionPolicy.php
│   ├── LruHydrationCacheEvictionPolicy.php
│   └── WeightedLruHydrationCacheEvictionPolicy.php
│
├── Generation/
│   ├── HydrationCacheGeneration.php
│   ├── MetadataGenerationResolver.php
│   ├── TypeGenerationResolver.php
│   └── ExtensionGenerationResolver.php
│
├── Compilation/
│   ├── SingleFlightHydrationPlanCoordinator.php
│   ├── HydrationCompilationKeyLock.php
│   └── HydrationCompilationRegistry.php
│
├── Negative/
│   ├── NegativeHydrationCache.php
│   ├── NegativeHydrationCacheEntry.php
│   └── NegativeHydrationCachePolicy.php
│
├── Warmup/
│   ├── HydrationCacheWarmer.php
│   ├── HydrationWarmupPlanProvider.php
│   └── HydrationWarmupReport.php
│
├── Persistent/
│   ├── PersistentHydrationPlanCache.php
│   ├── SerializedHydrationPlan.php
│   ├── HydrationPlanSerializer.php
│   ├── HydrationPlanDeserializer.php
│   ├── HydrationPlanChecksum.php
│   └── HydrationPlanFormatVersion.php
│
├── Runtime/
│   ├── DefaultHydrationCacheRuntimeAdapter.php
│   ├── HydrationCacheRuntimeDescriptor.php
│   └── HydrationCacheRuntimeResetter.php
│
├── Telemetry/
│   ├── HydrationCacheTelemetry.php
│   ├── HydrationCacheMetrics.php
│   └── HydrationCacheDiagnostics.php
│
└── Exception/
    └── ...
```

---

# 247. Flujo completo

```text
                    Hydration Request
                           │
                           ▼
                 HydrationPlanDefinition
                           │
                           ▼
                       Normalize
                           │
                           ▼
                       Validate
                           │
                           ▼
                     Fingerprint
                           │
                           ▼
                 HydrationCacheKey
                           │
                           ▼
                  ┌────────────────┐
                  │ Cache Lookup   │
                  └───────┬────────┘
                          │
                ┌─────────┴──────────┐
                │                    │
               HIT                  MISS
                │                    │
                ▼                    ▼
        Compatibility Check      Single Flight
                │                    │
          ┌─────┴─────┐              ▼
          │           │          Compile Plan
    COMPATIBLE      STALE             │
          │           │               ▼
          │           └──────────► Validate
          │                           │
          │                           ▼
          │                     Cacheability
          │                           │
          │                 ┌─────────┴────────┐
          │                 │                  │
          │              CACHEABLE         BYPASS
          │                 │                  │
          │                 ▼                  │
          │             Cache Put              │
          │                 │                  │
          └─────────────────┴──────────────────┘
                            │
                            ▼
                  CompiledHydrationPlan
                            │
                            ▼
                       Hydrators
                            │
                            ▼
                    Logical Result
```

---

# 248. Fórmulas fundamentales

## 248.1 Cache key

```text
HydrationCacheKey
=
Fingerprint(
    HydrationSemantics
)
+
DependencyGenerations
+
CompilerCompatibility
```

---

# 249. Valid entry

```text
ValidCacheEntry
=
EntryExists
∧
IntegrityValid
∧
FormatCompatible
∧
MetadataCompatible
∧
TypeCompatible
∧
ExtensionCompatible
∧
PlatformCompatible
```

---

# 250. Hit

```text
CacheHit
=
LookupSuccess
∧
ValidCacheEntry
```

---

# 251. Cacheability

```text
Cacheable(plan)
=
Immutable(plan)
∧
Deterministic(plan)
∧
NoScopedState(plan)
∧
RuntimeSafe(plan)
∧
KnownDependencies(plan)
```

---

# 252. Safe shared plan

```text
SafeSharedPlan
=
DeepImmutable
∧
NoEntityReferences
∧
NoPersistenceContext
∧
NoRequestReferences
∧
NoTransactionReferences
∧
ConcurrencySafeDependencies
```

---

# 253. Generation invalidation

```text
PlanGeneration
≠
CurrentGeneration
⇒
PlanNotReusable
```

---

# 254. Cache memory

```text
CacheMemory
=
Σ Weight(CacheEntryᵢ)
```

sujeto a:

```text
CacheMemory
≤
ConfiguredMemoryBudget
```

---

# 255. Stampede prevention

```text
ConcurrentMisses(SameKey)
→
OneCompilation
+
NWaiters
```

---

# 256. Failure resilience

```text
CacheInfrastructureFailure
≠
HydrationFailure
```

si el plan puede recompilarse de manera segura.

---

# 257. Persistent runtime safety

```text
SafePersistentHydrationCache
=
ImmutableWorkerCache
∧
ScopedHydrationSessions
∧
GenerationValidation
∧
BoundedMemory
∧
ConcurrentMissCoordination
∧
NoCrossRequestEntityState
```

---

# 258. Performance model

```text
WithoutCache
=
N × CompilationCost
+
N × HydrationCost
```

Con caché:

```text
WithCache
=
CompilationCost
+
N × LookupCost
+
N × HydrationCost
```

para un plan reutilizado.

Dado:

```text
LookupCost
≪
CompilationCost
```

la ganancia aumenta con la frecuencia de reutilización.

---

# 259. Master Formula

```text
Database Hydration Cache System
=
Compiled Hydration Plan Reuse
+
Structural Fingerprinting
+
Generation-Aware Keys
+
Metadata Compatibility
+
Type Registry Compatibility
+
Extension Compatibility
+
Compiler Versioning
+
Platform Capability Compatibility
+
Cache Namespaces
+
Worker Memory Cache
+
Optional Multi-Tier Cache
+
Bounded Memory
+
Admission Policies
+
Weighted Eviction
+
Warmup
+
Lazy Compilation
+
Single-Flight Compilation
+
Stampede Prevention
+
Negative Caching
+
Stale Plan Detection
+
Generation-Based Invalidation
+
Persistent Artifact Versioning
+
Integrity Validation
+
Cache Poisoning Protection
+
Runtime Adapters
+
FrankenPHP Worker Reuse
+
RoadRunner Worker Reuse
+
OpenSwoole Concurrency Safety
+
Failure Isolation
+
Telemetry
+
Diagnostics
+
Testing
```

---

# 260. Master Rule

> **El Hydration Cache de VoltStack será un caché estructural de artefactos compilados, nunca un caché de datos. Solo podrán sobrevivir entre requests aquellos planes y componentes que sean inmutables, deterministas, generation-aware y completamente independientes del `EntityManager`, `PersistenceContext`, `IdentityMap`, `UnitOfWork`, entidades, resultados y cualquier otro estado mutable de ejecución.**

---

# 261. Arquitectura final del Bloque 12 — Hydration

Con este documento queda definida la arquitectura completa del subsistema:

```text
                    QUERY ENGINE
                         │
                         ▼
                  EXECUTION ENGINE
                         │
                         ▼
                       Result
                         │
                         ▼
               Result Hydration System
                         │
                         ▼
                  Hydration Plan
                         │
                ┌────────┴─────────┐
                │                  │
                ▼                  ▼
        Hydration Plan Cache   Hydration Session
                │                  │
                │                  │ scoped
                │                  ▼
                │          Result Interpretation
                │                  │
        immutable/shared           │
                │        ┌─────────┼──────────┐
                │        │         │          │
                ▼        ▼         ▼          ▼
             Entity    Scalar    Tuple    Projection/DTO
            Hydrator  Hydrator  Hydrator     Hydrator
                │
                ▼
        Identity Resolution
                │
                ▼
           IdentityMap
                │
                ▼
        Canonical Entity
                │
                ▼
         Field Hydration
                │
                ▼
        Relationship Assembly
                │
                ▼
        Loaded Field Knowledge
                │
                ▼
            Snapshot
                │
                ▼
          EntityState
                │
                ▼
          UnitOfWork
                │
                ▼
       Graph Finalization
                │
                ▼
             postLoad
                │
                ▼
          Logical Result
```

---

# 262. Invariante global del bloque

```text
Result
→ HydrationPlan
→ Type Conversion
→ Identity Resolution
→ Canonical Materialization
→ Graph Assembly
→ Managed-State Registration
→ Baseline Establishment
→ Lifecycle Finalization
→ Logical Result
```

Nunca:

```text
Result
→ arbitrary reflection
→ duplicate entities
→ guessed types
→ hidden queries
→ unmanaged ORM ambiguity
```

---

# 263. Fórmula final del sistema de Hydration

```text
VoltStack Hydration
=
Result Interpretation
+
Compiled Hydration Plans
+
Plan Caching
+
Type-Safe Conversion
+
Identity Canonicalization
+
IdentityMap Reuse
+
Entity Materialization
+
Scalar Materialization
+
Tuple Materialization
+
Projection/DTO Materialization
+
JOIN Deduplication
+
Relationship Graph Assembly
+
Partial-State Knowledge
+
Snapshot Establishment
+
UnitOfWork Registration
+
EntityState Management
+
Lifecycle Finalization
+
Streaming Semantics
+
Failure Isolation
+
Persistent Runtime Isolation
```

---

# 264. Resultado arquitectónico del Bloque 12

El sistema queda separado en siete responsabilidades principales:

```text
135 HYDRATION_ARCHITECTURE
 │
 ├── 136 ENTITY_HYDRATOR_SYSTEM
 │
 ├── 137 RESULT_HYDRATION_SYSTEM
 │
 ├── 138 SCALAR_HYDRATION_SYSTEM
 │
 ├── 139 TUPLE_HYDRATION_SYSTEM
 │
 ├── 140 HYDRATION_PLAN_SYSTEM
 │
 └── 141 HYDRATION_CACHE_SYSTEM
```

Con ello se establece la regla:

> **El Result Hydrator gobierna la forma global del resultado; el Entity Hydrator gobierna entidades; el Scalar Hydrator gobierna valores; el Tuple Hydrator gobierna composiciones; el Hydration Plan describe cómo hacerlo; y el Hydration Cache evita tener que redescubrir y recompilar esa descripción en cada ejecución.**

---

# 265. Bloque 12 — Estado

```text
BLOCK 12 — HYDRATION
══════════════════════════════════════════

135 DATABASE_HYDRATION_ARCHITECTURE        ✓
136 DATABASE_ENTITY_HYDRATOR_SYSTEM        ✓
137 DATABASE_RESULT_HYDRATION_SYSTEM       ✓
138 DATABASE_SCALAR_HYDRATION_SYSTEM       ✓
139 DATABASE_TUPLE_HYDRATION_SYSTEM        ✓
140 DATABASE_HYDRATION_PLAN_SYSTEM         ✓
141 DATABASE_HYDRATION_CACHE_SYSTEM        ✓

STATUS: COMPLETE
```

---

# 266. Siguiente bloque

El siguiente paso inicia:

```text
BLOCK 13 — RELATIONSHIPS
```

con:

```text
142_DATABASE_RELATIONSHIP_ARCHITECTURE.md
```

El documento deberá establecer la arquitectura común para:

```text
One-to-One
One-to-Many
Many-to-One
Many-to-Many
Polymorphic Relationships
Relationship Metadata
Relationship Persistence
Relationship Loading
Eager Loading
Lazy Loading
Batch Relation Loading
N+1 Detection
```

y, especialmente, separar:

```text
ORM Relationship
≠
Foreign Key

Relationship Metadata
≠
Schema Constraint

Relationship Loading
≠
Hydration

Relationship Persistence
≠
Entity Persistence

Eager Loading
≠
JOIN necesariamente

Lazy Loading
≠
Proxy necesariamente

Cascade
≠
Database CASCADE

Orphan Removal
≠
Cascade Remove

Collection Membership
≠
Collection Initialization

Relationship Graph
≠
Persistence Graph
```

La arquitectura resultante deberá conectar:

```text
Entity Metadata
      │
      ▼
Relationship Metadata
      │
      ├──────────────┐
      ▼              ▼
Query Resolution   Persistence Planning
      │              │
      ▼              ▼
Relationship       Relationship
Loading            Persistence
      │              │
      ▼              ▼
Hydration         UnitOfWork
      │              │
      └──────┬───────┘
             ▼
       Entity Graph
```

El principio central del nuevo bloque será:

> **Una relación ORM representa una asociación semántica entre entidades. Puede estar respaldada por claves foráneas, tablas intermedias, columnas discriminadoras u otras estrategias físicas, pero su semántica pertenece al modelo de objetos y no debe confundirse con la representación concreta utilizada por la base de datos.**