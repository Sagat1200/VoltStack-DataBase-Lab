# 187_DATABASE_QUERY_CACHE_SYSTEM.md

# VoltStack Quantum Database
## Query Cache System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 187 — Query Cache System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `186_DATABASE_CACHE_ARCHITECTURE.md`  
**Siguiente documento:** `188_DATABASE_RESULT_CACHE_SYSTEM.md`

---

# 1. Propósito

`Query Cache System` define el mecanismo mediante el cual VoltStack podrá reutilizar artefactos derivados del procesamiento de una consulta sin volver a ejecutar innecesariamente las mismas fases estructurales del Query Engine.

Su regla principal será:

> **El Query Cache de VoltStack almacenará artefactos derivados de la interpretación, normalización, análisis, planificación o compilación de una consulta; nunca almacenará las filas obtenidas al ejecutar dicha consulta.**

Por tanto:

```text
Query Cache
≠
Result Cache
```

y:

```text
QueryCacheHit
≠
DatabaseReadAvoided
```

Un Query Cache hit puede evitar:

- reconstrucción;
- normalización;
- validación repetitiva;
- análisis semántico;
- resolución de símbolos;
- inferencia de tipos;
- optimización;
- planificación;
- compilación;

pero la consulta podrá seguir necesitando ejecutarse contra la base de datos.

---

# 2. Objetivo

Una consulta en VoltStack puede atravesar:

```text
Developer API
    ↓
Query Builder
    ↓
Query Model
    ↓
AST
    ↓
Normalization
    ↓
Validation
    ↓
Semantic Analysis
    ↓
Optimization
    ↓
Planning
    ↓
Compilation
    ↓
Execution
```

Varias de estas fases pueden ser costosas y repetirse para consultas estructuralmente equivalentes.

Ejemplo:

```php
User::query()
    ->where('email', $email)
    ->first();
```

puede ejecutarse miles de veces con diferentes valores de `$email`.

La estructura semántica sigue siendo esencialmente:

```text
SELECT User
WHERE User.email = Parameter<string>
LIMIT 1
```

Por tanto, VoltStack podrá reutilizar trabajo derivado de esa estructura.

---

# 3. Regla fundamental

Formalmente:

```text
QueryCache
=
Cache<
    QuerySemanticIdentity,
    ReusableQueryArtifact
>
```

No:

```text
QueryCache
=
Cache<
    QueryParameters,
    DatabaseRows
>
```

Esto último pertenece a:

```text
188_DATABASE_RESULT_CACHE_SYSTEM.md
```

---

# 4. Query Cache vs Result Cache

| Característica | Query Cache | Result Cache |
|---|---|---|
| Almacena filas | No | Sí |
| Evita consultar DB | Generalmente no | Sí |
| Depende de valores persistidos | Generalmente no | Sí |
| Sensible a cambios de datos | Normalmente no | Sí |
| Sensible a metadata | Sí | Sí/posiblemente |
| Sensible al dialecto | Según artefacto | Según identidad |
| Puede ser process-local | Sí | Sí, pero limitado |
| Puede compartirse entre requests | Sí | Sí |
| Riesgo de stale business data | Bajo | Alto |
| Riesgo de stale structural artifact | Sí | Secundario |

---

# 5. Query Cache vs Compiled Query Cache

VoltStack ya definió:

```text
075_DATABASE_COMPILED_QUERY_CACHE_SYSTEM.md
```

Por tanto, tampoco deberán confundirse.

Conceptualmente:

```text
Query Cache
    │
    ├── Normalized Query
    ├── Semantic Analysis
    ├── Optimization Artifact
    ├── Logical Plan
    └── Physical Plan
```

mientras:

```text
Compiled Query Cache
    ↓
Platform-specific CompiledQuery
```

El `Compiled Query Cache` podrá considerarse una especialización situada en la etapa final del procesamiento.

---

# 6. Arquitectura

```text
                  Query Builder
                       │
                       ▼
                   Query Model
                       │
                       ▼
                     AST
                       │
                       ▼
              Query Fingerprinter
                       │
                       ▼
                 Query Cache
                ┌──────┴──────┐
                │             │
              HIT            MISS
                │             │
                │             ▼
                │       Query Pipeline
                │             │
                │      ┌──────┴──────┐
                │      ▼             ▼
                │   Semantic      Optimizer
                │      │             │
                │      └──────┬──────┘
                │             ▼
                │          Planner
                │             │
                │             ▼
                │      Reusable Artifact
                │             │
                │             ▼
                │         Cache Store
                │             │
                └─────────────┘
                       │
                       ▼
                    Compiler
                       │
                       ▼
                    Executor
```

---

# 7. QuerySemanticIdentity

La identidad del caché no deberá depender simplemente del SQL final.

Se definirá conceptualmente:

```text
QuerySemanticIdentity
=
Fingerprint(
    NormalizedQueryStructure
    +
    SemanticContext
    +
    RelevantMetadataGenerations
    +
    RelevantCapabilities
)
```

---

# 8. Query fingerprint

El componente central será:

```php
interface QueryFingerprinter
{
    public function fingerprint(
        QueryModel $query,
        QueryFingerprintContext $context,
    ): QueryFingerprint;
}
```

Resultado:

```php
final readonly class QueryFingerprint
{
    public function __construct(
        public string $algorithm,
        public string $value,
        public int $version,
    ) {}
}
```

---

# 9. Fingerprint ≠ hash accidental

No se hará:

```php
hash('sha256', serialize($query));
```

como arquitectura central.

El fingerprint deberá derivarse de una representación canónica.

---

# 10. Canonical Query Representation

Ejemplo:

```php
User::query()
    ->where('email', 'ana@example.com')
    ->first();
```

podrá normalizarse conceptualmente como:

```text
SELECT
    ENTITY(User)
FROM
    ENTITY(User)
WHERE
    FIELD(User.email)
    =
    PARAMETER(type=string, slot=0)
LIMIT
    1
```

---

# 11. Parameter values

Los valores de parámetros no deberán formar parte del fingerprint estructural cuando no alteren la semántica estructural.

Por tanto:

```text
email = ana@example.com
```

y:

```text
email = maria@example.com
```

pueden compartir:

```text
QuerySemanticFingerprint
```

---

# 12. Parameter values vs parameter types

Aunque el valor pueda ignorarse:

```text
ParameterType
```

puede ser relevante.

Ejemplo:

```text
integer
```

vs:

```text
uuid
```

pueden producir:

- binding diferente;
- coerción diferente;
- SQL diferente;
- operator resolution diferente.

---

# 13. Parameter shape

También podrá importar:

```text
scalar
list
tuple
composite identifier
```

Ejemplo:

```php
whereIn('id', [1, 2, 3])
```

puede producir una estructura física distinta dependiendo de:

- cardinalidad;
- platform capabilities;
- array binding;
- placeholder expansion.

---

# 14. Cardinality-sensitive compilation

Un Query Cache deberá distinguir:

```text
semantic query identity
```

de:

```text
physical compilation shape
```

Por ejemplo:

```text
IN (?, ?)
```

vs:

```text
IN (?, ?, ?, ?, ?)
```

pueden compartir análisis semántico pero no necesariamente el mismo artefacto compilado.

---

# 15. Multi-stage cache

Por ello VoltStack podrá implementar caches por etapa:

```text
Query Cache
│
├── Normalization Cache
├── Semantic Cache
├── Optimization Cache
├── Logical Plan Cache
├── Physical Plan Cache
└── Compiled Query Cache
```

No necesariamente todos deberán habilitarse.

---

# 16. CacheStage

```php
enum QueryCacheStage
{
    case NORMALIZED;
    case SEMANTIC;
    case OPTIMIZED;
    case LOGICAL_PLAN;
    case PHYSICAL_PLAN;
    case COMPILED;
}
```

---

# 17. Stage-specific identity

Cada etapa podrá requerir dimensiones adicionales.

Formalmente:

```text
Key(stage N)
=
BaseQueryIdentity
+
Dependencies(stage N)
```

---

# 18. Normalization Cache

Puede depender principalmente de:

```text
AST structure
NormalizationRulesGeneration
```

---

# 19. Semantic Cache

Puede depender de:

```text
Normalized AST
Schema Metadata
Entity Metadata
Type Registry
Relationship Metadata
Semantic Rules
```

---

# 20. Optimization Cache

Puede depender de:

```text
Semantic Graph
OptimizerRuleGeneration
OptimizationPolicy
Capability Profile
```

---

# 21. Logical Plan Cache

Puede depender de:

```text
Optimized Semantic Query
PlannerGeneration
LogicalPlanningPolicy
```

---

# 22. Physical Plan Cache

Puede depender adicionalmente de:

```text
PlatformCapabilities
ExecutionCapabilities
PhysicalPlanningPolicy
```

---

# 23. Compiled Query Cache

Puede depender de:

```text
Physical Plan
Dialect
Platform
CompilerGeneration
ParameterShape
```

---

# 24. Dependency principle

Cuanto más avanzada sea la etapa:

```text
later pipeline stage
→ more dependencies
```

generalmente.

---

# 25. QueryCacheEntry

Modelo conceptual:

```php
final readonly class QueryCacheEntry
{
    public function __construct(
        public QueryCacheKey $key,
        public QueryCacheStage $stage,
        public QueryArtifact $artifact,
        public QueryCacheDependencies $dependencies,
        public QueryCacheMetadata $metadata,
    ) {}
}
```

---

# 26. QueryArtifact

Contrato:

```php
interface QueryArtifact
{
    public function stage(): QueryCacheStage;
}
```

Implementaciones:

```text
NormalizedQueryArtifact
SemanticQueryArtifact
OptimizedQueryArtifact
LogicalQueryPlanArtifact
PhysicalQueryPlanArtifact
CompiledQueryArtifact
```

---

# 27. Immutability

Los artefactos almacenados deberán ser:

```text
immutable
```

o tratados como inmutables.

Nunca:

```text
Cache HIT
→ mutate cached AST
```

---

# 28. Shared mutation hazard

Si dos requests reciben el mismo objeto cached y uno modifica:

```text
PredicateNode
```

podría corromper consultas concurrentes.

Por tanto:

```text
CachedQueryArtifact
→ immutable
```

será un invariant central.

---

# 29. QueryCacheKey

```php
final readonly class QueryCacheKey
{
    public function __construct(
        public QueryFingerprint $query,
        public QueryCacheStage $stage,
        public QueryDependencyFingerprint $dependencies,
        public QueryCacheGeneration $generation,
    ) {}
}
```

---

# 30. Key namespaces

Ejemplo conceptual:

```text
voltstack:
database:
query:
semantic:
v3:
<fingerprint>
```

La representación física dependerá del backend.

---

# 31. Cache key version

Toda estrategia de fingerprint deberá poseer versión.

Ejemplo:

```text
query-fingerprint:v4
```

Un cambio del algoritmo:

```text
v4 → v5
```

produce namespace distinto.

---

# 32. No cross-version collision

Artefactos producidos por algoritmos incompatibles no deberán compartir key.

---

# 33. AST fingerprinting

El AST será una de las mejores fuentes de identidad estructural.

Ejemplo:

```text
SelectNode
├── Source(User)
├── Projection(Entity(User))
├── Predicate
│   └── Equal
│       ├── Field(User.email)
│       └── Parameter(string)
└── Limit(1)
```

podrá convertirse en una representación canónica.

---

# 34. Canonical node identity

Cada tipo de AST node deberá definir qué propiedades son semánticamente relevantes.

---

# 35. Node ordering

Cuando el orden sea semánticamente relevante:

```text
ORDER BY a, b
```

deberá preservarse.

---

# 36. Order-insensitive structures

Cuando una transformación haya demostrado equivalencia, el normalizer podrá canonicalizar.

No será responsabilidad del fingerprinter inventar equivalencias.

---

# 37. Fingerprinter rule

> El fingerprinter identifica la estructura que recibe; no será un segundo optimizer.

---

# 38. Query normalization

El `DATABASE_QUERY_NORMALIZATION_SYSTEM` será responsable de canonicalizaciones como:

```text
Equivalent AST
→ Normalized AST
```

El Query Cache aprovechará esa representación.

---

# 39. Before-normalization cache

Un cache temprano puede existir, pero tendrá menor tasa de reutilización.

---

# 40. After-normalization cache

Normalmente será más valioso:

```text
syntactic variations
→ same normalized representation
→ same fingerprint
```

---

# 41. Query Builder object identity

Nunca:

```text
spl_object_id($builder)
```

será query identity.

---

# 42. PHP closure identity

Closures dinámicas dentro de una query extension pueden hacerla:

```text
NON_CACHEABLE
```

si no existe representación semántica estable.

---

# 43. Raw expressions

Un:

```php
DB::raw(...)
```

requiere tratamiento explícito.

---

# 44. Raw expression fingerprint

Si el raw fragment es aceptado:

```text
RawExpression
→ normalized trusted representation
→ fingerprint
```

pero su dependencia/capability puede ser desconocida.

---

# 45. Raw expression safety

Query Cache nunca deberá interpretar raw SQL como equivalente semántico a un AST tipado sin evidencia.

---

# 46. Cacheability analysis

Se utilizará:

```php
interface QueryCacheabilityAnalyzer
{
    public function analyze(
        QueryModel $query,
        QueryCacheContext $context,
    ): QueryCacheabilityAnalysis;
}
```

---

# 47. QueryCacheability

```php
enum QueryCacheability
{
    case CACHEABLE;
    case CACHEABLE_WITH_CONSTRAINTS;
    case NON_CACHEABLE;
    case UNKNOWN;
}
```

---

# 48. UNKNOWN

Regla segura:

```text
UNKNOWN
→ BYPASS
```

---

# 49. Reasons

```php
enum QueryNonCacheableReason
{
    case UNSTABLE_EXTENSION;
    case UNKNOWN_RAW_EXPRESSION;
    case SESSION_DEPENDENT_STRUCTURE;
    case TEMPORARY_OBJECT;
    case UNKNOWN_TYPE;
    case UNKNOWN_CAPABILITY;
    case NON_DETERMINISTIC_COMPILATION;
    case EXPLICIT_BYPASS;
}
```

---

# 50. Non-deterministic data vs compilation

Una consulta:

```sql
SELECT RANDOM()
```

puede seguir teniendo una estructura compilable estable.

Por tanto:

```text
NonDeterministicResult
```

no implica necesariamente:

```text
NonCacheableCompiledQuery
```

---

# 51. Important distinction

```text
Query Structural Cacheability
≠
Result Cacheability
```

`RANDOM()` es el ejemplo clásico.

El Query Cache podría almacenar su plan/compilación.

El Result Cache probablemente no debería almacenar el resultado ordinariamente.

---

# 52. Current timestamp

Lo mismo ocurre con:

```text
CURRENT_TIMESTAMP
```

La estructura puede ser estable aunque el resultado cambie.

---

# 53. Semantic dependencies

Un query artifact puede depender de:

```text
EntityMetadataGeneration
RelationshipMetadataGeneration
SchemaMetadataGeneration
TypeRegistryGeneration
QueryRuleGeneration
OptimizerGeneration
PlannerGeneration
PlatformCapabilityGeneration
CompilerGeneration
ExtensionGeneration
```

---

# 54. QueryCacheDependencies

```php
final readonly class QueryCacheDependencies
{
    public function __construct(
        public MetadataGeneration $metadata,
        public TypeRegistryGeneration $types,
        public QueryRuleGeneration $queryRules,
        public ?OptimizerGeneration $optimizer,
        public ?PlannerGeneration $planner,
        public ?PlatformCapabilityFingerprint $platform,
        public ?CompilerGeneration $compiler,
    ) {}
}
```

La estructura concreta podrá variar por stage.

---

# 55. Dependency minimization

No se deberán agregar dependencias irrelevantes.

Si un normalized AST no depende del Platform:

```text
PlatformId
```

no deberá destruir innecesariamente su reutilización.

---

# 56. Maximum reuse principle

```text
CacheKey
=
minimum complete semantic dependency set
```

No:

```text
every possible runtime property
```

---

# 57. Correctness first

Sin embargo:

```text
MissingRelevantDependency
```

es peor que:

```text
ExtraDependency
```

porque puede reutilizar un artefacto inválido.

---

# 58. Metadata generation

Ejemplo:

```text
User.email
→ column email
```

Luego mapping cambia:

```text
User.email
→ column email_address
```

El Semantic/Compiled Query Cache anterior no deberá reutilizarse.

---

# 59. Generation invalidation

Preferencia:

```text
MetadataGeneration 41
→ MetadataGeneration 42
```

hace que el artefacto anterior deje de ser addressable.

---

# 60. No global clear required

No será necesario recorrer millones de keys para eliminarlas inmediatamente.

---

# 61. Type generation

Si cambia:

```text
domain.user_id
```

de representación:

```text
BIGINT
```

a:

```text
UUID
```

artefactos dependientes deberán invalidarse.

---

# 62. Relationship metadata

Cambiar:

```text
Order.customer
```

puede afectar:

- join resolution;
- eager loading;
- semantic graph;
- plans.

---

# 63. Schema metadata

Las queries raw/schema-aware pueden depender directamente del schema.

---

# 64. Optimizer generation

Agregar/modificar una optimization rule puede requerir regenerar:

```text
OptimizedQueryArtifact
```

pero no necesariamente:

```text
NormalizedQueryArtifact
```

---

# 65. Stage isolation

Esto permite:

```text
Optimizer change
→ invalidate optimization+
→ preserve normalization/semantic cache
```

---

# 66. Planner generation

Igualmente:

```text
Planner change
→ invalidate plans
→ preserve semantic analysis
```

---

# 67. Compiler generation

```text
Compiler change
→ invalidate compiled queries
```

sin eliminar metadata cache.

---

# 68. Platform identity

Un semantic query puede ser portable.

Un compiled query generalmente no.

Ejemplo:

```text
Semantic Query
      │
      ├── PostgreSQL Compiler
      ├── MySQL Compiler
      ├── MariaDB Compiler
      └── SQLite Compiler
```

---

# 69. Platform-independent caching

Se buscará reutilizar lo máximo posible antes de la etapa específica de plataforma.

---

# 70. Capability fingerprint

Para physical planning/compilation podrá utilizarse:

```text
PlatformCapabilityFingerprint
```

en vez de solamente:

```text
mysql
```

---

# 71. Version-specific capabilities

Dos versiones del mismo motor pueden poseer capabilities distintas.

Por tanto:

```text
VendorName
```

por sí solo puede ser insuficiente.

---

# 72. Capability identity

Conceptualmente:

```text
PlatformCapabilityFingerprint
=
Fingerprint(
    relevant capabilities
)
```

---

# 73. Capability minimization

Solo capabilities que afecten el artefacto deberán participar cuando sea viable.

---

# 74. Dialect identity

Compiled cache podrá incluir:

```text
DialectId
DialectVersion
CompilerGeneration
```

---

# 75. Connection identity

Una Query Cache key no deberá incluir physical connection ID salvo que exista una dependencia real de estado de conexión.

---

# 76. Session state

Si compilation depende de:

```text
session SQL mode
search path
collation
temporary schema
```

esa dimensión deberá:

1. incorporarse a la identidad; o
2. hacer el artefacto no cacheable.

---

# 77. Safe default for unknown session state

```text
UNKNOWN
→ BYPASS stage affected
```

---

# 78. Transaction independence

La mayoría de los structural query artifacts podrán reutilizarse dentro y fuera de una transacción.

---

# 79. Query Cache ≠ Result Cache transaction rule

El documento anterior estableció:

```text
ActiveTransaction
→ usually bypass shared Result Cache
```

Pero esto no implica:

```text
ActiveTransaction
→ bypass structural Query Cache
```

---

# 80. Structural cache safety

Si el artefacto no depende de visibility de datos:

```text
transaction state
```

no necesita formar parte de la key.

---

# 81. Transaction-specific structure

Si una query contiene:

```text
FOR UPDATE
```

su estructura sí deberá estar representada en el fingerprint.

---

# 82. Locking mode

```text
SELECT
```

y:

```text
SELECT ... FOR UPDATE
```

no son la misma query identity.

---

# 83. Isolation level

Normalmente el isolation level no cambia el SQL estructural de una query ordinaria.

No deberá agregarse a todas las keys indiscriminadamente.

---

# 84. Isolation-sensitive extension

Si un dialect/extension genera una estrategia distinta según isolation:

```text
isolation
```

deberá formar parte de esa etapa específica.

---

# 85. Read/write routing

Query routing no deberá contaminar automáticamente semantic query identity.

---

# 86. Primary vs replica

La misma query estructural:

```text
SELECT User#42
```

puede usar el mismo semantic/compiled artifact tanto en writer como replica si sus capabilities son equivalentes.

---

# 87. Routing plan cache

Un routing decision cache es un artefacto diferente.

No deberá confundirse con Query Cache.

---

# 88. Sharding

Una query:

```text
WHERE customer_id = ?
```

puede tener la misma estructura en todos los shards.

Por tanto:

```text
ShardId
```

no deberá incluirse automáticamente en structural query key.

---

# 89. Shard-dependent metadata

Si diferentes shards poseen schemas/capabilities incompatibles:

```text
ShardCapabilityGeneration
```

sí podrá ser necesario.

---

# 90. Partition routing

El valor concreto del shard key puede decidir la partición.

Eso pertenece a:

```text
Partition Routing
```

no necesariamente a query semantic identity.

---

# 91. Tenant isolation

Si todos los tenants comparten el mismo mapping/schema:

```text
TenantId
```

no deberá formar parte automáticamente del structural Query Cache.

---

# 92. Tenant-specific schema

Si:

```text
Tenant A
→ schema generation 4

Tenant B
→ schema generation 7
```

la key deberá distinguir la dependency generation correspondiente.

---

# 93. Logical dependency over tenant identity

Preferir:

```text
SchemaGeneration
```

sobre:

```text
TenantId
```

cuando eso capture completamente la diferencia semántica.

Esto aumenta reutilización segura.

---

# 94. Query parameter descriptors

Modelo:

```php
final readonly class QueryParameterShape
{
    public function __construct(
        public ParameterType $type,
        public ParameterCardinality $cardinality,
        public bool $nullable,
    ) {}
}
```

---

# 95. Parameter values excluded

Ejemplo:

```text
Parameter:
    type = UUID
    cardinality = SINGLE
```

forma parte del shape.

El UUID:

```text
550e8400-e29b-41d4-a716-446655440000
```

no.

---

# 96. Literal values

Los literales embebidos en AST requieren una decisión.

Ejemplo:

```text
LIMIT 10
```

puede afectar el plan.

---

# 97. Structural literal

Un literal que cambie estructura/semántica del plan deberá participar.

---

# 98. Bindable literal

Cuando el normalizer pueda convertirlo legítimamente en parameter:

```text
Literal
→ ParameterSlot
```

podrá aumentar reutilización.

---

# 99. Normalizer authority

El Query Cache no hará esta transformación por sí mismo.

---

# 100. Projection identity

Estas queries son distintas:

```text
SELECT id
```

y:

```text
SELECT id, email
```

aunque utilicen misma tabla y predicate.

---

# 101. Result shape

El resultado esperado también puede afectar:

```text
Hydration Plan
```

pero no siempre al semantic SQL plan.

---

# 102. Query artifact boundaries

Cada cache stage deberá declarar si incluye:

```text
ResultShape
HydrationMode
ORM Entity Projection
Tuple Projection
Scalar Projection
```

---

# 103. Hydration cache separation

```text
Query Plan
≠
Hydration Plan
```

Aunque ambos puedan compartir fingerprints derivados.

---

# 104. Hydration fingerprint

Podrá construirse a partir de:

```text
QueryProjectionIdentity
+
ResultShape
+
EntityMetadataGeneration
```

como se estableció en documento 141.

---

# 105. ORM integration

ORM Entity Query:

```php
User::query()
    ->where('active', true)
    ->get();
```

deberá converger en el mismo Query Engine.

---

# 106. No ORM-specific SQL cache

No se creará:

```text
ActiveRecordSQLCache
```

paralelo a:

```text
RepositorySQLCache
```

---

# 107. Single query infrastructure

```text
Model API
Repository
EntityManager
Raw Query Builder
        ↓
    Query Engine
        ↓
    Query Cache
```

---

# 108. API origin

La API desde la que se originó una query no deberá afectar la key si el Query Model resultante es semánticamente equivalente.

---

# 109. Repository query

```php
$users->active();
```

y un Query Builder equivalente podrán compartir artefactos.

---

# 110. Extension system

Custom query nodes deberán participar en fingerprinting mediante contratos explícitos.

---

# 111. Custom node contract

Ejemplo:

```php
interface QueryFingerprintContributor
{
    public function contribute(
        QueryFingerprintBuilder $builder,
    ): void;
}
```

---

# 112. No arbitrary serialization

Un custom node no podrá simplemente insertar:

```text
serialize($this)
```

como fingerprint confiable.

---

# 113. Stable extension identity

Una extensión deberá declarar:

```text
ExtensionId
ExtensionVersion
NodeSemanticIdentity
```

---

# 114. Extension generation

Cambio incompatible:

```text
ExtensionGeneration
```

invalidará artefactos relacionados.

---

# 115. Extension conflict

Dos extensiones no podrán registrar el mismo semantic node identity silenciosamente.

---

# 116. Cache store

Query Cache utilizará los contratos definidos por:

```text
186_DATABASE_CACHE_ARCHITECTURE.md
```

Conceptualmente:

```php
interface QueryCacheStore
{
    public function get(QueryCacheKey $key): QueryCacheLookup;

    public function put(
        QueryCacheKey $key,
        QueryCacheEntry $entry,
    ): void;
}
```

---

# 117. Specialized facade over generic store

Internamente podrá adaptar:

```text
DatabaseCacheStore
```

en lugar de exigir un backend separado.

---

# 118. L1 Query Cache

El primer nivel natural será:

```text
process memory
```

especialmente con FrankenPHP.

---

# 119. L1 characteristics

Ventajas:

- latencia mínima;
- sin serialización remota;
- ideal para immutable artifacts;
- alta reutilización en persistent workers.

---

# 120. L1 risk

Workers distintos tendrán caches distintos.

Generation handling será necesario para deployments/hot reload.

---

# 121. L2 Query Cache

Opcionalmente:

```text
Redis / shared cache
```

podrá compartir artifacts entre workers.

---

# 122. L2 usefulness

Especialmente útil cuando:

```text
CompilationCost
>>
RemoteCacheLookupCost
```

---

# 123. L2 not always beneficial

Para un artefacto extremadamente barato:

```text
Redis lookup
>
recompute
```

por lo que no deberá almacenarse automáticamente.

---

# 124. Stage-specific store

Configuración conceptual:

```php
'query_cache' => [

    'normalized' => 'memory',

    'semantic' => 'memory',

    'logical_plan' => 'memory',

    'physical_plan' => 'memory',

    'compiled' => 'memory',

];
```

Podrán existir L1/L2 por etapa.

---

# 125. Admission policy

El Query Cache podrá decidir:

```text
cache this artifact?
```

según:

- compilation cost;
- artifact size;
- expected reuse;
- stability;
- complexity.

---

# 126. One-shot queries

Una query generada una sola vez puede no justificar caching.

---

# 127. Frequency-based admission

Podrá utilizarse:

```text
admit after N observations
```

como optimización futura.

---

# 128. Complexity-based admission

Queries complejas pueden admitirse inmediatamente.

---

# 129. Artifact size

Un enormous semantic graph puede superar límites de cache.

---

# 130. Cache limits

Configurable:

```text
max_entries
max_entry_bytes
max_total_memory
max_dependencies
```

---

# 131. LRU/LFU

Un L1 podrá utilizar:

```text
LRU
LFU
Clock
TinyLFU
```

sin que la arquitectura central dependa del algoritmo.

---

# 132. Eviction

Eviction no significa query invalidation.

La consulta simplemente se procesará nuevamente.

---

# 133. Cache miss

Siempre deberá ser un camino normal:

```text
MISS
→ Query Pipeline
```

---

# 134. Cache unavailable

Para Query Cache:

```text
FAIL_OPEN
```

será el default natural.

---

# 135. Query correctness

Formalmente:

```text
Execute(Query, QueryCacheDisabled)
=
Execute(Query, QueryCacheEnabled)
```

respecto a semántica observable.

---

# 136. Query cache failure

Un fallo del cache no deberá impedir normalmente ejecutar la query.

---

# 137. Corrupted artifact

Si se detecta:

```text
fingerprint mismatch
invalid payload version
unknown node type
invalid dependency generation
```

deberá tratarse como:

```text
INVALID
→ evict
→ MISS
```

---

# 138. Artifact verification

Podrá existir:

```php
interface QueryArtifactVerifier
{
    public function verify(
        QueryArtifact $artifact,
        QueryCacheContext $context,
    ): QueryArtifactVerification;
}
```

---

# 139. Verification levels

```text
FAST
STANDARD
STRICT
```

podrán utilizarse según entorno.

---

# 140. Development mode

Puede comprobar más invariants.

---

# 141. Production mode

Deberá usar verificaciones suficientemente seguras pero de bajo costo.

---

# 142. Serialization

L1 in-memory puede conservar objetos immutable directamente.

L2 necesitará codec.

---

# 143. AST codec

Un AST distribuido deberá serializarse mediante formato versionado.

---

# 144. No closures

Cached artifacts distribuibles no deberán contener:

- closures;
- resources;
- PDO objects;
- connections;
- EntityManager;
- UnitOfWork;
- runtime callbacks.

---

# 145. Portable artifact rule

```text
DistributedQueryArtifact
→ pure data
```

---

# 146. Runtime references

No almacenar:

```text
Connection
Driver
PDOStatement
Request
TenantContext object
TransactionContext
```

dentro del artifact.

---

# 147. Symbol references

Utilizar:

```text
EntityTypeId
FieldId
TypeId
RelationshipId
```

estables.

---

# 148. Compiled accessors

Si contienen runtime-generated code, su cacheability dependerá de la estrategia segura de compilation cache.

---

# 149. Security

Query Cache deberá considerarse infraestructura interna confiable, pero sus payloads no se asumirán infalibles.

---

# 150. Cache poisoning protection

Un atacante no deberá controlar directamente:

```text
QueryCacheKey
```

mediante valores de parámetros ordinarios.

---

# 151. Parameter exclusion advantage

Al excluir valores:

```text
user input
```

de fingerprints estructurales, se reduce cardinalidad y exposición.

---

# 152. Raw SQL

Raw fragments sí pueden contener input si el desarrollador los construye incorrectamente.

Esto no deberá producir:

- secrets en cache keys;
- secrets en telemetry.

---

# 153. Fingerprint confidentiality

Preferir:

```text
hash(canonical representation)
```

para la key física.

---

# 154. Debug representation

La representación legible podrá mantenerse separada y sanitizada.

---

# 155. QueryCacheContext

```php
final readonly class QueryCacheContext
{
    public function __construct(
        public QueryCacheStage $stage,
        public QueryMetadataGeneration $metadataGeneration,
        public TypeRegistryGeneration $typeGeneration,
        public QueryEngineGeneration $queryEngineGeneration,
        public ?PlatformCapabilityFingerprint $platform,
        public RuntimeScopeId $runtimeScope,
    ) {}
}
```

No deberá contener mutable transaction/entity state innecesario.

---

# 156. RuntimeScopeId

Normalmente no formará parte de la key.

Sirve para:

- diagnostics;
- telemetry;
- local coordination.

---

# 157. Persistent runtime

Con FrankenPHP:

```text
Worker
├── Query Cache L1
├── Metadata Cache
├── Hydration Plan Cache
└── Compiled Query Cache
```

podrán sobrevivir entre requests.

---

# 158. Persistent runtime rule

Solo:

```text
immutable reusable structural state
```

deberá sobrevivir.

---

# 159. Forbidden persistent state

No conservar:

```text
parameter values
query results
current transaction
current tenant object
current connection lease
managed entities
```

como parte del Query Cache.

---

# 160. OpenSwoole

L1 deberá ser:

- concurrency-safe;
- immutable;
- bounded.

---

# 161. RoadRunner

Cada worker podrá poseer su propio L1.

---

# 162. Cross-worker generation

Una nueva deployment generation deberá hacer que workers antiguos/nuevos no compartan artefactos incompatibles.

---

# 163. Development hot reload

Cuando cambia:

```text
Entity mapping
```

el generation resolver deberá invalidar el artifact space correspondiente.

---

# 164. QueryCacheManager

Propuesta:

```php
final class QueryCacheManager
{
    public function lookup(
        QueryModel $query,
        QueryCacheStage $stage,
        QueryCacheContext $context,
    ): QueryCacheLookup;

    public function store(
        QueryCacheKey $key,
        QueryArtifact $artifact,
        QueryCacheContext $context,
    ): void;
}
```

---

# 165. Manager responsibilities

Sí:

- fingerprint;
- cacheability;
- key construction;
- lookup;
- verification;
- admission;
- store.

No:

- execute SQL;
- hydrate entities;
- manage transactions;
- mutate UnitOfWork.

---

# 166. Pipeline integration

Preferencia:

```text
QueryPipelineStage
      │
      ▼
QueryCacheInterceptor
      │
 ┌────┴─────┐
 HIT       MISS
 │           │
 │      Real Stage
 │           │
 │      Cache Store
 │           │
 └─────┬─────┘
       ▼
Next Pipeline Stage
```

---

# 167. No cache logic inside every stage

Evitar:

```php
if ($cache->has(...)) {
    ...
}
```

repetido en:

- analyzer;
- optimizer;
- planner;
- compiler.

Preferir infraestructura transversal controlada.

---

# 168. QueryCacheInterceptor

```php
interface QueryCacheInterceptor
{
    public function execute(
        QueryPipelineStage $stage,
        QueryArtifact $input,
        QueryPipelineContext $context,
        callable $next,
    ): QueryArtifact;
}
```

---

# 169. Stage contracts

Cada stage deberá poder declarar:

```text
cacheable output?
dependencies?
fingerprint inputs?
```

---

# 170. QueryCacheDescriptor

Modelo:

```php
final readonly class QueryCacheDescriptor
{
    public function __construct(
        public QueryCacheStage $stage,
        public bool $cacheable,
        public CacheScope $scope,
        public QueryCacheDependencyPolicy $dependencies,
    ) {}
}
```

---

# 171. Query Cache scope

Podrá ser:

```php
enum QueryCacheScope
{
    case NONE;
    case OPERATION;
    case PROCESS;
    case DISTRIBUTED;
}
```

---

# 172. OPERATION

Útil para evitar trabajo repetido dentro de una sola operación.

---

# 173. PROCESS

Ideal para persistent workers.

---

# 174. DISTRIBUTED

Solo para artefactos serializables y compatibles entre procesos.

---

# 175. Scope escalation

No todo artefacto `PROCESS` deberá ser automáticamente `DISTRIBUTED`.

---

# 176. Platform-independent semantic artifact

Buen candidato a distributed cache si:

- pure data;
- versionado;
- metadata generations compatibles.

---

# 177. Runtime-generated closure artifact

Puede ser:

```text
PROCESS only
```

o no cacheable.

---

# 178. Cache warming

VoltStack podrá precalentar queries conocidas.

Ejemplo:

```php
QueryCacheWarmer::register(
    fn () => User::query()->where('email', Parameter::string())->first()
);
```

La API definitiva deberá evitar ejecutar accidentalmente la query.

---

# 179. Warm compile

El warmup debe detenerse antes de execution:

```text
Build
→ Analyze
→ Optimize
→ Plan
→ Compile
→ Cache
```

No:

```text
→ Execute
```

---

# 180. CLI

Conceptualmente:

```text
php volt database:query-cache:warm
```

---

# 181. Inspect

```text
php volt database:query-cache:status
```

---

# 182. Clear

```text
php volt database:query-cache:clear
```

preferiblemente mediante generation advancement.

---

# 183. Explain

```php
DB::queryCache()->explain($query);
```

Resultado conceptual:

```text
QUERY CACHE EXPLAIN

Query:
    User where email = :string limit 1

Fingerprint:
    qfp:v3:91a8...

Stage:
    SEMANTIC

Cacheable:
    YES

Dependencies:
    EntityMetadataGeneration: 41
    TypeRegistryGeneration: 12
    QueryRulesGeneration: 8

Lookup:
    HIT

Artifact:
    SemanticQueryArtifact

Age:
    14m

Validity:
    VALID
```

---

# 184. Compiled explain

```text
Stage:
    COMPILED

Platform:
    PostgreSQL

Capabilities:
    3bc91...

Compiler Generation:
    22

Parameter Shape:
    [STRING]

Lookup:
    HIT
```

---

# 185. Diagnostics

Deberán poder responder:

```text
Why did this query miss cache?
```

---

# 186. MissReason

```php
enum QueryCacheMissReason
{
    case ENTRY_NOT_FOUND;
    case GENERATION_CHANGED;
    case DEPENDENCY_CHANGED;
    case ARTIFACT_INVALID;
    case PAYLOAD_VERSION_MISMATCH;
    case NON_CACHEABLE;
    case EXPLICIT_BYPASS;
    case BACKEND_UNAVAILABLE;
    case CAPABILITY_CHANGED;
}
```

---

# 187. Telemetry

Eventos:

```text
QueryCacheLookupStarted
QueryCacheHit
QueryCacheMiss
QueryCacheBypass
QueryCacheStored
QueryCacheEvicted
QueryCacheInvalidated
QueryCacheArtifactRejected
```

---

# 188. Metrics

```text
db.query_cache.hit
db.query_cache.miss
db.query_cache.bypass
db.query_cache.store
db.query_cache.eviction
db.query_cache.lookup.duration
db.query_cache.artifact.bytes
```

---

# 189. Stage metric

Dimensión bounded:

```text
stage=semantic
stage=optimized
stage=logical_plan
stage=physical_plan
stage=compiled
```

---

# 190. Fingerprint metric prohibition

Nunca:

```text
fingerprint=91a8...
```

como metric label.

---

# 191. Hit ratio

```text
HitRatio
=
Hits / (Hits + Misses)
```

pero no será suficiente para evaluar utilidad.

---

# 192. Saved work

Métrica más útil:

```text
EstimatedPipelineWorkSaved
```

o duración evitada.

---

# 193. Cache effectiveness

Conceptualmente:

```text
Effectiveness
=
AvoidedProcessingCost
-
CacheOverhead
```

---

# 194. Query cache can hurt

Un hit ratio alto no garantiza mejor rendimiento si:

```text
cache lookup cost
>
recompute cost
```

---

# 195. Sampling

Telemetry detallada podrá utilizar sampling en producción.

---

# 196. Testing

El sistema requerirá pruebas unitarias para:

- fingerprinting;
- normalization identity;
- parameter exclusion;
- parameter type inclusion;
- stage dependencies;
- generation invalidation;
- cacheability;
- artifact immutability;
- codec;
- L1;
- L2;
- failure handling.

---

# 197. Equivalent query test

```php
$q1 = User::query()->where('email', 'a@example.com');

$q2 = User::query()->where('email', 'b@example.com');
```

Esperado:

```text
StructuralFingerprint(q1)
=
StructuralFingerprint(q2)
```

si ambos valores se parametrizan.

---

# 198. Different projection test

```php
$q1 = User::query()->select('id');

$q2 = User::query()->select('id', 'email');
```

Esperado:

```text
Fingerprint(q1)
≠
Fingerprint(q2)
```

---

# 199. Lock test

```text
SELECT User
```

vs:

```text
SELECT User FOR UPDATE
```

deben producir identities distintas.

---

# 200. Metadata change test

```text
Generation 10
→ HIT

Generation 11
→ MISS
```

---

# 201. Compiler change test

Semantic cache puede seguir:

```text
HIT
```

mientras compiled cache:

```text
MISS
```

---

# 202. Platform test

Misma semantic query:

```text
Semantic Cache
→ shared
```

pero:

```text
PostgreSQL Compiled Cache
≠
MySQL Compiled Cache
```

cuando corresponda.

---

# 203. Tenant test

Dos tenants con metadata generation idéntica podrán reutilizar structural artifact cuando no exista dependencia tenant-specific.

---

# 204. Tenant schema divergence test

Si divergen:

```text
Generation A ≠ Generation B
```

deberán obtener artifacts separados.

---

# 205. Shard capability test

Shards equivalentes pueden compartir.

Shards con capability fingerprints distintos no deberán compartir physical artifacts incompatibles.

---

# 206. Persistent worker test

Request A y B podrán compartir:

```text
SemanticQueryArtifact
```

pero nunca:

```text
parameter value
TransactionContext
```

---

# 207. Concurrency test

100 coroutines solicitando misma query deberán:

- recibir artifact correcto;
- no mutarlo;
- no producir corrupción;
- opcionalmente utilizar single-flight.

---

# 208. Stampede test

Cold cache:

```text
100 concurrent misses
```

no deberá producir 100 compilaciones costosas si single-flight está habilitado.

---

# 209. Single-flight failure

Si el productor falla:

```text
waiters
```

deberán:

- recibir failure apropiado; o
- reintentar bounded;

nunca quedar bloqueados.

---

# 210. Corruption test

Payload corrupto:

```text
decode failure
```

deberá resultar en:

```text
evict/bypass
```

y no en ejecución de artifact parcialmente reconstruido.

---

# 211. Cache backend outage test

La query deberá seguir funcionando mediante pipeline normal bajo `FAIL_OPEN`.

---

# 212. Performance tests

Medir:

```text
cold query processing
warm semantic cache
warm plan cache
warm compiled cache
L1 lookup
L2 lookup
artifact encode/decode
```

---

# 213. Memory tests

Verificar:

```text
bounded L1
```

durante millones de query shapes.

---

# 214. High-cardinality query shapes

Input del usuario no deberá poder producir memoria infinita mediante shapes artificialmente distintos sin límites.

---

# 215. Cache pollution

Admission policy deberá limitar:

```text
one-use query fingerprints
```

que desplazan artefactos valiosos.

---

# 216. Query shape attack

Un atacante podría variar:

- number of predicates;
- projections;
- sort combinations;
- IN cardinality.

Resource Governance deberá limitar el impacto.

---

# 217. Security tests

Verificar:

- no secrets en physical keys;
- no parameter values en telemetry;
- no unsafe deserialization;
- no arbitrary class loading;
- bounded payloads.

---

# 218. Error hierarchy

```text
QueryCacheException
├── QueryCacheFingerprintException
├── QueryCacheKeyException
├── QueryCacheLookupException
├── QueryCacheWriteException
├── QueryCacheArtifactException
├── QueryCacheArtifactCorruptionException
├── QueryCacheDependencyException
├── QueryCacheGenerationException
├── QueryCacheCapabilityException
├── QueryCacheSerializationException
├── QueryCacheDeserializationException
├── QueryCacheAdmissionException
├── QueryCacheExtensionException
└── QueryCacheInvariantViolationException
```

---

# 219. Directory structure

Propuesta:

```text
src/Quantum/Database/Cache/Query/
│
├── QueryCacheManager.php
├── QueryCacheStage.php
├── QueryCacheKey.php
├── QueryCacheEntry.php
├── QueryCacheContext.php
├── QueryCacheLookup.php
├── QueryCacheScope.php
│
├── Fingerprint/
│   ├── QueryFingerprint.php
│   ├── QueryFingerprinter.php
│   ├── QueryFingerprintBuilder.php
│   ├── QueryFingerprintVersion.php
│   ├── AstFingerprinter.php
│   ├── ParameterShapeFingerprinter.php
│   └── PlatformCapabilityFingerprinter.php
│
├── Artifact/
│   ├── QueryArtifact.php
│   ├── NormalizedQueryArtifact.php
│   ├── SemanticQueryArtifact.php
│   ├── OptimizedQueryArtifact.php
│   ├── LogicalQueryPlanArtifact.php
│   ├── PhysicalQueryPlanArtifact.php
│   └── QueryArtifactVerifier.php
│
├── Cacheability/
│   ├── QueryCacheability.php
│   ├── QueryCacheabilityAnalyzer.php
│   ├── QueryCacheabilityAnalysis.php
│   └── QueryNonCacheableReason.php
│
├── Dependency/
│   ├── QueryCacheDependencies.php
│   ├── QueryDependencyFingerprint.php
│   ├── MetadataGenerationDependency.php
│   ├── TypeGenerationDependency.php
│   ├── OptimizerGenerationDependency.php
│   ├── PlannerGenerationDependency.php
│   ├── CompilerGenerationDependency.php
│   └── PlatformCapabilityDependency.php
│
├── Pipeline/
│   ├── QueryCacheInterceptor.php
│   ├── QueryCacheDescriptor.php
│   └── QueryCacheStageRegistry.php
│
├── Admission/
│   ├── QueryCacheAdmissionPolicy.php
│   └── QueryCacheAdmissionDecision.php
│
├── Store/
│   ├── QueryCacheStore.php
│   ├── L1QueryCache.php
│   ├── L2QueryCache.php
│   └── TieredQueryCache.php
│
├── Codec/
│   ├── QueryArtifactCodec.php
│   ├── QueryArtifactEncoder.php
│   └── QueryArtifactDecoder.php
│
├── Runtime/
│   ├── QueryCacheRuntimeState.php
│   ├── QueryCacheGenerationResolver.php
│   └── QueryCacheSingleFlight.php
│
├── Diagnostics/
│   ├── QueryCacheInspector.php
│   ├── QueryCacheExplainer.php
│   ├── QueryCacheDiagnosticReport.php
│   └── QueryCacheMissReason.php
│
├── Telemetry/
│   ├── QueryCacheTelemetry.php
│   ├── QueryCacheHitEvent.php
│   ├── QueryCacheMissEvent.php
│   └── QueryCacheStoredEvent.php
│
└── Exception/
    ├── QueryCacheException.php
    ├── QueryCacheFingerprintException.php
    ├── QueryCacheArtifactException.php
    ├── QueryCacheDependencyException.php
    └── QueryCacheInvariantViolationException.php
```

---

# 220. Dependency direction

```text
Query Builder
     ↓
Query Model / AST
     ↓
Query Pipeline
     ↓
Query Cache Infrastructure
     ↓
Optimizer / Planner / Compiler
```

El Query Cache consume contratos de esas etapas.

No deberá invertir la arquitectura:

```text
Compiler
→ Redis-specific cache logic
```

---

# 221. Integration with generic Cache

```text
QueryCache
     ↓
DatabaseCacheStore
     ↓
Database Cache Adapter
     ↓
VoltStack/Quantum/Cache
```

---

# 222. No mandatory cache package

Si `VoltStack/Quantum/Cache` no está instalado:

```text
InMemoryQueryCache
```

podrá proporcionar optimización local.

---

# 223. Null cache

También:

```text
NullQueryCache
```

permitirá desactivar completamente el sistema.

---

# 224. Architectural invariants

## DB-QCACHE-001
Query Cache no almacenará database result rows.

## DB-QCACHE-002
Query Cache será distinto de Result Cache.

## DB-QCACHE-003
Query Cache será distinto de Entity Cache.

## DB-QCACHE-004
Query Cache será distinto de IdentityMap.

## DB-QCACHE-005
Query Cache podrá incluir múltiples pipeline stages.

## DB-QCACHE-006
Compiled Query Cache será una especialización de etapa, no Result Cache.

## DB-QCACHE-007
Query identity será semántica/estructural.

## DB-QCACHE-008
Physical SQL string no será identidad universal.

## DB-QCACHE-009
Builder object identity no será query identity.

## DB-QCACHE-010
Parameter values no formarán parte del structural fingerprint por default.

## DB-QCACHE-011
Parameter types sí podrán formar parte del fingerprint.

## DB-QCACHE-012
Parameter cardinality podrá afectar physical artifacts.

## DB-QCACHE-013
Structural literals relevantes participarán en identity.

## DB-QCACHE-014
Normalizer será responsable de parameterization/canonicalization.

## DB-QCACHE-015
Fingerprinter no será optimizer.

## DB-QCACHE-016
AST fingerprinting utilizará canonical representation.

## DB-QCACHE-017
Fingerprint algorithm será versionado.

## DB-QCACHE-018
Incompatible fingerprint versions no compartirán entries.

## DB-QCACHE-019
Cached artifacts serán immutable.

## DB-QCACHE-020
Cached artifact no podrá mutarse por request.

## DB-QCACHE-021
Stage-specific dependencies serán explícitas.

## DB-QCACHE-022
Later stages podrán tener más dependencies.

## DB-QCACHE-023
Metadata generation invalidará artifacts dependientes.

## DB-QCACHE-024
Type generation invalidará artifacts dependientes.

## DB-QCACHE-025
Optimizer generation no invalidará necesariamente normalization artifacts.

## DB-QCACHE-026
Planner generation no invalidará necesariamente semantic artifacts.

## DB-QCACHE-027
Compiler generation invalidará compiled artifacts.

## DB-QCACHE-028
Platform-independent artifacts podrán reutilizarse entre platforms compatibles.

## DB-QCACHE-029
Platform-specific artifacts incluirán capability identity suficiente.

## DB-QCACHE-030
Vendor name no sustituirá capability model.

## DB-QCACHE-031
Physical connection ID no formará parte de key por default.

## DB-QCACHE-032
Session-dependent compilation deberá declarar dependencies o bypass.

## DB-QCACHE-033
UNKNOWN session semantics producirán bypass seguro.

## DB-QCACHE-034
Active transaction no deshabilitará automáticamente structural Query Cache.

## DB-QCACHE-035
Locking mode formará parte de semantic identity.

## DB-QCACHE-036
Isolation level no formará parte indiscriminadamente de todas las keys.

## DB-QCACHE-037
Isolation-sensitive compilation deberá declararlo.

## DB-QCACHE-038
Primary/replica routing no alterará semantic identity por sí mismo.

## DB-QCACHE-039
Shard ID no formará parte automáticamente de structural query identity.

## DB-QCACHE-040
Shard-specific capabilities podrán formar parte de physical identity.

## DB-QCACHE-041
Tenant ID no formará parte automáticamente de structural identity.

## DB-QCACHE-042
Tenant-specific metadata generation sí podrá separar artifacts.

## DB-QCACHE-043
Semantic dependency será preferible a coarse tenant identity cuando sea suficiente.

## DB-QCACHE-044
Projection formará parte de query identity.

## DB-QCACHE-045
Result shape será separado de Query Plan cuando corresponda.

## DB-QCACHE-046
Hydration Plan será distinto de Query Plan.

## DB-QCACHE-047
ORM Model API y Repository compartirán Query Cache.

## DB-QCACHE-048
No existirá segundo SQL cache exclusivo para Active Record.

## DB-QCACHE-049
API origin no alterará identity si Query Model es equivalente.

## DB-QCACHE-050
Custom query nodes declararán fingerprint estable.

## DB-QCACHE-051
Custom node no usará arbitrary object serialization como identity.

## DB-QCACHE-052
Extension identity será versionada.

## DB-QCACHE-053
Extension conflicts fallarán explícitamente.

## DB-QCACHE-054
UNKNOWN cacheability producirá bypass.

## DB-QCACHE-055
Structural cacheability será distinta de result cacheability.

## DB-QCACHE-056
Non-deterministic result no implica non-cacheable compilation.

## DB-QCACHE-057
Current-time query podrá tener structural artifact cacheable.

## DB-QCACHE-058
Raw expression no será asumida semanticamente conocida.

## DB-QCACHE-059
Raw fragments desconocidos podrán reducir cacheability.

## DB-QCACHE-060
Cache miss será un estado normal.

## DB-QCACHE-061
Query Cache podrá funcionar FAIL_OPEN.

## DB-QCACHE-062
Query correctness no dependerá de cache availability.

## DB-QCACHE-063
Corrupt artifact no será ejecutado.

## DB-QCACHE-064
Invalid artifact se convertirá en miss/bypass.

## DB-QCACHE-065
L1 será bounded.

## DB-QCACHE-066
L2 será opcional.

## DB-QCACHE-067
L2 no será usado cuando lookup sea más caro que recompute según policy.

## DB-QCACHE-068
Cache admission será policy-driven.

## DB-QCACHE-069
One-shot query no tendrá que cachearse.

## DB-QCACHE-070
Large artifact podrá rechazarse.

## DB-QCACHE-071
Cache eviction no cambiará query semantics.

## DB-QCACHE-072
Distributed artifacts serán pure data.

## DB-QCACHE-073
Distributed artifact no contendrá Connection.

## DB-QCACHE-074
Distributed artifact no contendrá Driver.

## DB-QCACHE-075
Distributed artifact no contendrá EntityManager.

## DB-QCACHE-076
Distributed artifact no contendrá UnitOfWork.

## DB-QCACHE-077
Distributed artifact no contendrá TransactionContext.

## DB-QCACHE-078
Distributed artifact no contendrá PHP resources.

## DB-QCACHE-079
Distributed artifact no contendrá closures sin estrategia explícita.

## DB-QCACHE-080
Stable identifiers serán preferidos a runtime object references.

## DB-QCACHE-081
Cache payload será versionado.

## DB-QCACHE-082
Unsafe deserialization estará prohibida.

## DB-QCACHE-083
Physical cache key no expondrá parámetros sensibles.

## DB-QCACHE-084
Telemetry no expondrá parameter values.

## DB-QCACHE-085
Persistent workers podrán compartir immutable query artifacts entre requests.

## DB-QCACHE-086
Persistent Query Cache no conservará parameter values.

## DB-QCACHE-087
Persistent Query Cache no conservará transaction state.

## DB-QCACHE-088
Persistent Query Cache no conservará tenant runtime objects.

## DB-QCACHE-089
Persistent Query Cache no conservará managed entities.

## DB-QCACHE-090
OpenSwoole cache será concurrency-safe.

## DB-QCACHE-091
RoadRunner workers podrán tener L1 independientes.

## DB-QCACHE-092
Deployment generation evitará cross-version incompatibility.

## DB-QCACHE-093
Development hot reload invalidará generations relevantes.

## DB-QCACHE-094
QueryCacheManager no ejecutará SQL.

## DB-QCACHE-095
QueryCacheManager no hidratará entities.

## DB-QCACHE-096
QueryCacheManager no administrará transactions.

## DB-QCACHE-097
QueryCacheManager no administrará UnitOfWork.

## DB-QCACHE-098
Cache pipeline será transversal, no lógica Redis dispersa.

## DB-QCACHE-099
Cada stage declarará cache semantics.

## DB-QCACHE-100
PROCESS scope será distinto de DISTRIBUTED scope.

## DB-QCACHE-101
Un artifact process-cacheable no será automáticamente distributed-cacheable.

## DB-QCACHE-102
Warmup no ejecutará queries.

## DB-QCACHE-103
Warmup se detendrá antes de Execution Engine.

## DB-QCACHE-104
Cache clear preferirá generation advancement.

## DB-QCACHE-105
Explain API mostrará cacheability y dependencies.

## DB-QCACHE-106
Miss reason será observable.

## DB-QCACHE-107
Metrics utilizarán bounded labels.

## DB-QCACHE-108
Fingerprint no será metric label.

## DB-QCACHE-109
Hit ratio no será única medida de efectividad.

## DB-QCACHE-110
Cache overhead será medido.

## DB-QCACHE-111
Equivalent parameterized queries podrán compartir fingerprint.

## DB-QCACHE-112
Different projections producirán identities distintas.

## DB-QCACHE-113
Locking y non-locking queries producirán identities distintas.

## DB-QCACHE-114
Metadata generation changes producirán miss donde corresponda.

## DB-QCACHE-115
Compiler generation podrá invalidar solo compiled stage.

## DB-QCACHE-116
Platform compilation artifacts serán separados cuando sean incompatibles.

## DB-QCACHE-117
Concurrent cache hits no mutarán shared artifact.

## DB-QCACHE-118
Single-flight será bounded.

## DB-QCACHE-119
Single-flight producer failure no bloqueará indefinidamente waiters.

## DB-QCACHE-120
Backend outage no impedirá query execution bajo FAIL_OPEN.

## DB-QCACHE-121
L1 memory será bounded.

## DB-QCACHE-122
High-cardinality query shapes estarán sujetos a resource governance.

## DB-QCACHE-123
Cache pollution podrá ser mitigada por admission policy.

## DB-QCACHE-124
User input no producirá keys ilimitadas.

## DB-QCACHE-125
Query Cache podrá operar sin VoltStack generic Cache package.

## DB-QCACHE-126
NullQueryCache será implementación válida.

## DB-QCACHE-127
InMemoryQueryCache será implementación válida.

## DB-QCACHE-128
Cache implementation no modificará Query Model original.

## DB-QCACHE-129
Cached semantic artifact conservará metadata identity.

## DB-QCACHE-130
Cached plan conservará planner compatibility.

## DB-QCACHE-131
Cached compiled artifact conservará compiler compatibility.

## DB-QCACHE-132
Capabilities desconocidas no serán asumidas compatibles.

## DB-QCACHE-133
Cross-shard reuse requerirá semantic compatibility.

## DB-QCACHE-134
Cross-tenant reuse requerirá semantic compatibility.

## DB-QCACHE-135
Cross-platform reuse dependerá del cache stage.

## DB-QCACHE-136
Normalization cache no dependerá de plataforma sin necesidad.

## DB-QCACHE-137
Semantic cache no dependerá de physical endpoint sin necesidad.

## DB-QCACHE-138
Physical plan cache podrá depender de execution capabilities.

## DB-QCACHE-139
Compiled cache podrá depender de parameter shape.

## DB-QCACHE-140
Result values nunca serán parte del Query Cache payload.

---

# 225. Pipeline recomendado

La ruta recomendada será:

```text
Query Model
    │
    ▼
Normalize
    │
    ▼
Normalized Fingerprint
    │
    ├──── Normalization Cache
    │
    ▼
Semantic Analysis
    │
    ├──── Semantic Cache
    │
    ▼
Optimizer
    │
    ├──── Optimization Cache
    │
    ▼
Logical Planner
    │
    ├──── Logical Plan Cache
    │
    ▼
Physical Planner
    │
    ├──── Physical Plan Cache
    │
    ▼
Compiler
    │
    ├──── Compiled Query Cache
    │
    ▼
Executor
```

---

# 226. Diseño de reutilización máxima

Idealmente:

```text
               Portable
                  │
Query ────────────┼───────────────
Normalization     │
Semantic          │
Optimization      │
Logical Plan      │
                  ▼
           Platform Boundary
                  │
Physical Plan     │
Compilation       │
                  ▼
               Execution
```

VoltStack deberá intentar mantener la frontera específica de plataforma tan abajo como sea razonablemente posible.

---

# 227. Fórmula de identidad por etapa

Podemos expresar:

```text
Kₙ
=
Fingerprint(
    Q
    +
    D₀
    +
    D₁
    +
    ...
    +
    Dₙ
)
```

donde:

```text
Q
=
canonical query identity
```

y:

```text
Dₙ
=
dependencies introduced by pipeline stage n
```

---

# 228. Correctness condition

Un artifact `A` podrá reutilizarse si:

```text
Fingerprint(CurrentQuery)
=
Fingerprint(A.Query)
```

y:

```text
∀ dependency d ∈ A.dependencies:
    Current(d) compatibleWith Cached(d)
```

---

# 229. Cache hit definition

Por tanto:

```text
QueryCacheHit
=
EntryFound
∧
FingerprintValid
∧
DependenciesValid
∧
GenerationValid
∧
ArtifactValid
∧
PolicyAllowsReuse
```

No:

```text
EntryFound
=
Hit
```

---

# 230. Performance model

Sin cache:

```text
Tquery
=
Tnormalize
+
Tsemantic
+
Toptimize
+
Tplan
+
Tcompile
+
Texecute
```

Con cache en etapa `n`:

```text
Tquery
=
Tlookup
+
Tremaining_pipeline
+
Texecute
```

Beneficio:

```text
SavedCost
=
Tskipped_pipeline
-
Tlookup
```

Solo conviene cuando:

```text
SavedCost > 0
```

---

# 231. Memory model

Para L1:

```text
Memory
≈
Σ ArtifactSize(entry)
+
IndexOverhead
+
DependencyMetadata
```

Deberá mantenerse:

```text
Memory <= ConfiguredBudget
```

---

# 232. Cache lifecycle

```text
Query
   ↓
Fingerprint
   ↓
Cacheability
   ↓
Lookup
   ├── HIT
   │    ↓
   │ Artifact Verification
   │    ↓
   │ Continue Pipeline
   │
   └── MISS
        ↓
      Compute
        ↓
      Admission
        ↓
      Store
        ↓
      Continue Pipeline
```

---

# 233. Diseño final

`Query Cache System` será una infraestructura estructural del Query Engine, no un mecanismo para almacenar información de negocio.

Su posición arquitectónica será:

```text
Query Model / AST
        ↓
Query Processing Pipeline
        ↕
   Query Cache
        ↓
    Compiler
        ↓
    Executor
        ↓
     Database
```

Nunca:

```text
Query Cache
    ↓
Cached customer rows
```

---

# 234. Regla maestra final

> **El Query Cache de VoltStack reutilizará conocimiento computacional sobre una consulta, no el resultado persistente de ejecutarla. Su identidad se derivará de la representación semántica canónica de la consulta y de las generaciones/capacidades que realmente afectan al artefacto almacenado, permitiendo máxima reutilización sin compartir artefactos incompatibles.**

Formalmente:

```text
QueryCacheEntry
=
CanonicalQueryIdentity
+
PipelineStage
+
MinimalCompleteDependencySet
+
ImmutableReusableArtifact
+
Generation
```

Y:

```text
QueryCacheHit
→ skip query-processing work
```

mientras:

```text
ResultCacheHit
→ skip database execution
```

Esta separación será obligatoria en toda la arquitectura de `VoltStack/Quantum/Database`.

---

# 235. Siguiente documento

```text
188_DATABASE_RESULT_CACHE_SYSTEM.md
```

El siguiente documento definirá el sistema encargado de almacenar y reutilizar **resultados obtenidos de la base de datos**, incluyendo:

```text
Result Cache Identity
Query + Parameter Fingerprinting
Result Payload Model
Canonical Result Representation
TTL
Freshness
Staleness
Transaction Awareness
Replica Awareness
Tenant Isolation
Shard Isolation
Dependency Tracking
Mutation Invalidation
Negative Results
Pagination
Aggregates
Entity Results
Result Shape
Cache Admission
Stampede Protection
Stale-While-Revalidate
Persistent Runtime
Security
Telemetry
Diagnostics
```

con la regla fundamental:

```text
Query Cache
→ caches computation about queries

Result Cache
→ caches data obtained by executing queries
```