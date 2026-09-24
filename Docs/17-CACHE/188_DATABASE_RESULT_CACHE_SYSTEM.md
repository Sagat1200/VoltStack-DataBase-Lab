# 188_DATABASE_RESULT_CACHE_SYSTEM.md

# VoltStack Quantum Database
## Result Cache System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 188 — Result Cache System  
**Bloque:** 17 — Cache  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `187_DATABASE_QUERY_CACHE_SYSTEM.md`  
**Siguiente documento:** `189_DATABASE_METADATA_CACHE_SYSTEM.md`

---

# 1. Propósito

`Result Cache System` define la infraestructura mediante la cual VoltStack podrá reutilizar de forma controlada resultados previamente obtenidos mediante la ejecución de consultas.

A diferencia del `Query Cache`, este sistema sí puede evitar una nueva consulta contra la base de datos.

Regla principal:

> **El Result Cache de VoltStack almacenará representaciones cacheables de resultados previamente obtenidos de la base de datos y únicamente podrá reutilizarlas cuando pueda demostrar que la identidad, contexto, visibilidad y política de consistencia del resultado almacenado son compatibles con la operación actual.**

Formalmente:

```text
ResultCacheHit
=
IdentityMatch
∧ ContextCompatible
∧ FreshnessAccepted
∧ VisibilityCompatible
∧ DependencyCompatible
∧ PolicyAllowsReuse
```

No:

```text
KeyExists
=
ResultCacheHit
```

---

# 2. Distinción fundamental

Debe mantenerse:

```text
Query Cache
≠
Result Cache
≠
Entity Cache
≠
IdentityMap
```

## Query Cache

Almacena:

```text
Normalized Query
Semantic Analysis
Query Plans
Compiled Query
```

## Result Cache

Almacena:

```text
Database Result Data
```

## Entity Cache

Almacena representaciones cacheables asociadas a entidades.

## IdentityMap

Mantiene la identidad canónica de objetos administrados dentro de un scope ORM.

---

# 3. Posición arquitectónica

```text
Application / ORM / Query API
            │
            ▼
        Query Engine
            │
            ▼
       Query Executor
            │
            ▼
     Result Cache Layer
       ┌────┴────┐
       │         │
      HIT       MISS
       │         │
       │         ▼
       │      Database
       │         │
       │         ▼
       │    Raw Result
       │         │
       │         ▼
       │    Cache Store
       │         │
       └────┬────┘
            ▼
       Result System
            │
            ▼
        Hydration
```

La integración exacta podrá variar según el tipo de resultado, pero el caché deberá permanecer conceptualmente entre:

```text
Query Execution Intent
```

y:

```text
Physical Database Execution
```

---

# 4. Regla de ejecución

Sin Result Cache:

```text
Query
  ↓
Compile
  ↓
Execute DB
  ↓
Result
```

Con Result Cache:

```text
Query
  ↓
ResultCacheKey
  ↓
Lookup
 ┌───────────────┐
 │               │
HIT             MISS
 │               │
 ▼               ▼
Cached Result   Execute DB
 │               │
 │               ▼
 │             Result
 │               │
 │               ▼
 │              Store
 │               │
 └───────┬───────┘
         ▼
      Consumer
```

---

# 5. Result Cache no será implícitamente universal

No todas las consultas deberán almacenarse.

Ejemplos potencialmente adecuados:

```text
catalog queries
configuration data
reference tables
aggregates
public listings
stable dashboards
expensive read queries
```

Ejemplos normalmente problemáticos:

```text
FOR UPDATE
transaction-sensitive reads
highly volatile data
security-context-sensitive queries
non-deterministic results
streaming queries
extremely large results
```

---

# 6. Cacheability explícita

VoltStack utilizará:

```php
enum ResultCacheability
{
    case CACHEABLE;
    case CACHEABLE_WITH_CONSTRAINTS;
    case NON_CACHEABLE;
    case UNKNOWN;
}
```

Regla:

```text
UNKNOWN
→ BYPASS
```

---

# 7. Opt-in vs automático

La arquitectura deberá soportar políticas como:

```text
DISABLED
EXPLICIT_ONLY
POLICY_DRIVEN
AUTOMATIC_SAFE
```

El default inicial recomendado será conservador:

```text
EXPLICIT_ONLY
```

o:

```text
POLICY_DRIVEN
```

No se deberá asumir que toda consulta `SELECT` es cacheable.

---

# 8. API conceptual

Ejemplo:

```php
$users = User::query()
    ->where('active', true)
    ->cacheFor(minutes: 5)
    ->get();
```

También:

```php
$result = DB::query()
    ->from('products')
    ->where('published', true)
    ->cacheFor(seconds: 30)
    ->get();
```

---

# 9. Named cache policy

Preferible para aplicaciones grandes:

```php
->cacheUsing('catalog')
```

Configuración:

```php
'result_cache' => [

    'policies' => [

        'catalog' => [
            'ttl' => '5 minutes',
            'consistency' => 'bounded_staleness',
        ],

    ],

];
```

---

# 10. No magic business semantics

VoltStack no deberá inferir:

```text
products table
→ safe for 5 minutes
```

sin política o metadata explícita.

---

# 11. ResultCacheKey

Modelo:

```php
final readonly class ResultCacheKey
{
    public function __construct(
        public QueryFingerprint $query,
        public ParameterFingerprint $parameters,
        public ResultShapeFingerprint $shape,
        public ResultVisibilityFingerprint $visibility,
        public ResultCacheGeneration $generation,
    ) {}
}
```

---

# 12. Fórmula de identidad

Conceptualmente:

```text
ResultCacheIdentity
=
QuerySemanticIdentity
+
CanonicalParameterValues
+
ResultShape
+
VisibilityContext
+
RelevantDataScope
+
Generation
```

---

# 13. Diferencia respecto al Query Cache

En `187`:

```text
Parameter values
→ generally excluded
```

En Result Cache:

```text
Parameter values
→ generally required
```

porque:

```text
WHERE user_id = 10
```

y:

```text
WHERE user_id = 20
```

producen resultados distintos.

---

# 14. ParameterFingerprint

```php
interface ParameterFingerprinter
{
    public function fingerprint(
        ParameterSet $parameters,
        ParameterFingerprintContext $context,
    ): ParameterFingerprint;
}
```

---

# 15. Canonical parameter representation

No deberá utilizarse simplemente:

```php
serialize($parameters);
```

Los valores deberán canonicalizarse según:

```text
Type System
Value Conversion semantics
Parameter type
```

---

# 16. Ejemplo UUID

Estas representaciones podrían ser semánticamente equivalentes:

```text
550E8400-E29B-41D4-A716-446655440000
```

y:

```text
550e8400-e29b-41d4-a716-446655440000
```

si el tipo UUID de VoltStack define canonicalización correspondiente.

---

# 17. Decimal

```text
10.50
```

no deberá convertirse accidentalmente en:

```text
10.5 float
```

si eso destruye precisión semántica.

---

# 18. DateTime

Los parámetros temporales deberán usar representación canónica definida por:

```text
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
```

No por timezone global del worker.

---

# 19. NULL

Debe distinguirse:

```text
NULL
```

de:

```text
MISSING
```

cuando la estructura de parámetros lo requiera.

---

# 20. Parameter order

Named parameters podrán canonicalizarse independientemente de su orden de construcción cuando semánticamente corresponda.

---

# 21. Lists

Para:

```php
whereIn('id', [5, 2, 9])
```

no se deberá reordenar automáticamente el valor para el fingerprint salvo que el Query Model garantice que la operación es order-insensitive.

---

# 22. Sensitive values

Los parámetros pueden contener:

- emails;
- tokens;
- identifiers;
- datos personales.

Nunca deberán aparecer directamente en physical cache keys.

Preferir:

```text
HMAC(canonical parameter payload)
```

o fingerprint criptográfico apropiado.

---

# 23. Result shape

Estas operaciones no son necesariamente intercambiables:

```php
->get();
->first();
->value('email');
->pluck('email');
->count();
```

Aunque compartan parte de la query.

---

# 24. ResultShapeFingerprint

Podrá contener:

```text
SCALAR
SCALAR_LIST
ROW
ROW_COLLECTION
TUPLE
ENTITY_INPUT
PROJECTION
AGGREGATE
```

---

# 25. Raw result vs hydrated result

Decisión central:

> El Result Cache base deberá preferir almacenar representaciones de resultado independientes del EntityManager y del IdentityMap, no objetos Entity administrados.

---

# 26. Nunca cachear managed entities directamente

Prohibido:

```text
Result Cache
→ User object currently managed by EntityManager
```

porque ese objeto pertenece a un scope ORM específico.

---

# 27. Preferencia de representación

```text
Database Result
    ↓
Canonical Cache Result
    ↓
Result Cache
```

Luego:

```text
Cached Canonical Result
    ↓
Hydration
    ↓
Entity / DTO / Scalar
```

---

# 28. Beneficio

Así:

```text
Cached data
```

puede reutilizarse entre:

- requests;
- workers;
- EntityManagers;
- runtimes.

---

# 29. CanonicalResultPayload

```php
interface CanonicalResultPayload
{
    public function shape(): ResultShape;
}
```

Implementaciones:

```text
ScalarResultPayload
RowResultPayload
RowCollectionResultPayload
TupleResultPayload
AggregateResultPayload
EmptyResultPayload
```

---

# 30. Row representation

Una fila cacheada deberá preservar:

```text
column identity
canonical value
NULL
type information where required
```

---

# 31. Driver raw values

Evitar almacenar valores cuyo significado dependa de un objeto driver activo.

---

# 32. Value conversion

Idealmente:

```text
Driver Raw Result
      ↓
Canonical Value Conversion
      ↓
Canonical Result Payload
      ↓
Result Cache
```

---

# 33. Hydration

Posteriormente:

```text
Canonical Result Payload
      ↓
Hydration Plan
      ↓
Entity/DTO/Scalar
```

---

# 34. IdentityMap integration

Cuando un resultado cacheado se hidrata como entidad:

```text
Cached Row
    ↓
Entity Hydrator
    ↓
IdentityMap
```

deberán aplicarse exactamente las mismas reglas que para un resultado proveniente de DB.

---

# 35. Canonical entity identity

Si:

```text
IdentityMap
already contains User#42
```

un Result Cache hit no deberá crear:

```text
another User#42 object
```

---

# 36. Dirty managed entity

Tampoco deberá sobrescribir silenciosamente una entidad administrada que tenga cambios locales.

---

# 37. Result Cache ≠ Entity Cache

Aunque un cached result contenga filas de entidades:

```text
ResultCache(Query)
```

sigue siendo distinto de:

```text
EntityCache(EntityKey)
```

---

# 38. Query-based identity

Ejemplo:

```text
active users ordered by name
```

pertenece naturalmente a Result Cache.

---

# 39. Entity-based identity

Ejemplo:

```text
User#42
```

podrá pertenecer al futuro:

```text
190_DATABASE_ENTITY_CACHE_SYSTEM.md
```

---

# 40. Freshness

Todo Result Cache entry deberá poseer política explícita de frescura.

---

# 41. Freshness model

```php
final readonly class ResultFreshnessPolicy
{
    public function __construct(
        public Duration $freshFor,
        public ?Duration $staleFor,
    ) {}
}
```

---

# 42. States

```php
enum ResultCacheFreshness
{
    case FRESH;
    case STALE_ACCEPTABLE;
    case EXPIRED;
    case UNKNOWN;
}
```

---

# 43. FRESH

```text
age <= freshFor
```

---

# 44. STALE_ACCEPTABLE

Puede utilizarse para:

```text
stale-while-revalidate
```

si la política lo permite.

---

# 45. EXPIRED

No deberá servirse ordinariamente.

---

# 46. UNKNOWN freshness

Default:

```text
UNKNOWN
→ MISS
```

---

# 47. TTL ≠ consistency

Regla:

```text
TTL
≠
ConsistencyGuarantee
```

Un TTL de 5 segundos no demuestra que el resultado sea correcto durante esos 5 segundos.

---

# 48. Staleness

Un resultado puede quedar stale inmediatamente después de ser almacenado:

```text
T0 cache result
T1 another transaction updates data
T2 cached result served
```

aunque:

```text
T2 - T0 = 10 ms
```

---

# 49. Consistency policy

Por tanto, Result Cache deberá declarar su modelo.

Ejemplo:

```php
enum ResultCacheConsistency
{
    case BEST_EFFORT;
    case TTL_BOUNDED;
    case INVALIDATION_DRIVEN;
    case VERSION_VALIDATED;
    case TRANSACTION_AWARE;
}
```

---

# 50. No fake strong consistency

VoltStack no deberá etiquetar un cache como:

```text
STRONG
```

si no existe un mecanismo que realmente garantice esa semántica.

---

# 51. Cache dependency

Una query puede depender de:

```text
users
orders
order_items
products
```

Por tanto, su resultado deberá poder asociarse a dependencies.

---

# 52. ResultDependencySet

```php
final readonly class ResultDependencySet
{
    /**
     * @param list<ResultDependency> $dependencies
     */
    public function __construct(
        public array $dependencies,
    ) {}
}
```

---

# 53. Dependency granularities

Posibles:

```text
DATABASE
SCHEMA
TABLE
ENTITY_TYPE
ENTITY_ID
QUERY_DOMAIN
CUSTOM_TAG
```

---

# 54. Coarse dependency

Ejemplo:

```text
TABLE: users
```

Cualquier mutación de `users` podría invalidar.

Ventaja:

```text
simple
```

Desventaja:

```text
over-invalidation
```

---

# 55. Fine-grained dependency

Ejemplo:

```text
ENTITY: User#42
```

permite invalidación más precisa.

---

# 56. Query predicate dependency

Inferir exactamente qué modificaciones afectan:

```text
WHERE age > 18
AND country = 'MX'
```

es mucho más complejo.

VoltStack no deberá fingir precisión que no posee.

---

# 57. Conservative invalidation

Cuando no pueda demostrar granularidad precisa:

```text
invalidate broader scope
```

será preferible a:

```text
serve known-stale data as fresh
```

cuando la política requiera invalidación.

---

# 58. Dependency extraction

Podrá existir:

```php
interface ResultDependencyAnalyzer
{
    public function analyze(
        QueryModel $query,
        QuerySemanticGraph $semanticGraph,
    ): ResultDependencyAnalysis;
}
```

---

# 59. Static dependencies

Normalmente podrán detectarse:

```text
FROM users
JOIN orders
JOIN products
```

---

# 60. Dynamic dependencies

Funciones, views, stored routines o raw SQL pueden impedir conocimiento completo.

Resultado:

```text
PARTIAL
UNKNOWN
```

---

# 61. Dependency coverage

```php
enum DependencyCoverage
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 62. Invalidation-driven policy

Si requiere dependencies completas:

```text
PARTIAL
→ NON_CACHEABLE
```

o una política de invalidación más amplia.

---

# 63. Cache tags

Ejemplo:

```text
db:table:users
db:entity:user:42
db:catalog
```

Las physical tags deberán evitar información sensible.

---

# 64. Mutation invalidation

Integración:

```text
Persistence
    ↓
Mutation Effects
    ↓
Cache Invalidation Planner
    ↓
Result Cache Invalidation
```

---

# 65. Invalidation timing

No deberá invalidarse como si una mutación fuera committed antes de que realmente lo sea.

---

# 66. Transaction problem

Ejemplo:

```text
BEGIN
UPDATE products ...
invalidate cache
ROLLBACK
```

La invalidación temprana es segura respecto a correctness pero innecesaria.

Peor sería:

```text
update uncommitted
populate shared cache
```

---

# 67. Recommended invalidation

Cuando sea posible:

```text
Mutation
   ↓
Transaction
   ↓
COMMIT confirmed
   ↓
afterCommit
   ↓
Invalidate affected cache entries
```

---

# 68. UNKNOWN commit

Si:

```text
CommitSent
+
ConnectionLost
=
UNKNOWN
```

no podrá asumirse que no ocurrió la mutación.

Política conservadora:

```text
UNKNOWN
→ invalidate potentially affected entries
```

cuando sea viable.

---

# 69. Invalidation failure

Si DB commit fue confirmado pero invalidación falla:

```text
Database = NEW
Cache = OLD
```

Existe riesgo de stale reads.

---

# 70. Mitigaciones

Podrán utilizarse:

```text
short TTL
generation tokens
outbox
invalidation journal
retry
version validation
```

---

# 71. Transactional outbox

Para necesidades más fuertes:

```text
DB Transaction
├── Business Mutation
└── CacheInvalidationOutboxRecord
```

Después:

```text
Outbox Consumer
→ Cache Invalidation
```

---

# 72. Outbox ≠ instant consistency

Existe una ventana hasta procesar el evento.

Por tanto, la semántica deberá documentarse.

---

# 73. Version token

Alternativa:

```text
ResultCacheKey
+
DataGeneration
```

Cuando cambia la generación:

```text
old cache entry
→ unreachable
```

---

# 74. Table generation

Ejemplo:

```text
users:generation=18
```

Tras mutation:

```text
users:generation=19
```

Las queries nuevas usan generation 19.

---

# 75. Generation trade-off

Ventaja:

```text
O(1) logical invalidation
```

Desventaja:

```text
coarse invalidation
old physical entries remain until eviction
```

---

# 76. Transaction reads

Regla recomendada:

> Un Result Cache compartido no deberá participar automáticamente en lecturas dentro de una transacción activa.

---

# 77. Default transaction behavior

```text
ActiveTransaction
→ SharedResultCache BYPASS
```

---

# 78. Razón

La transacción puede requerir:

```text
snapshot visibility
read-your-own-writes
locks
isolation semantics
```

que un resultado compartido no puede representar correctamente.

---

# 79. Read-your-own-writes

Ejemplo:

```php
DB::transaction(function () {
    User::find(42)->update(['name' => 'Ana']);

    return User::find(42);
});
```

Un cache global con nombre anterior sería incorrecto.

---

# 80. Transaction-local cache

VoltStack podrá tener una optimización separada:

```text
TransactionLocalResultMemoization
```

pero no deberá confundirse con shared Result Cache.

---

# 81. Transaction-local lifetime

```text
begin
→ create local memo
→ use
→ commit/rollback
→ discard
```

Nunca sobrevivirá al transaction scope.

---

# 82. Locking queries

Consultas:

```text
FOR UPDATE
FOR SHARE
```

deberán ser:

```text
NON_CACHEABLE
```

por default.

---

# 83. Why

Un cache hit evitaría adquirir el lock solicitado.

Eso cambiaría la semántica de la consulta.

---

# 84. Write queries

```text
INSERT
UPDATE
DELETE
```

no pertenecen al Result Cache ordinario.

---

# 85. RETURNING

Una operación:

```text
UPDATE ... RETURNING
```

puede producir filas, pero sigue siendo una mutación.

No deberá ser tratada como cacheable read.

---

# 86. Non-deterministic functions

Ejemplos:

```text
RANDOM()
CURRENT_TIMESTAMP
UUID_GENERATE()
```

podrán hacer el resultado:

```text
NON_CACHEABLE
```

salvo política explícita.

---

# 87. Query Cache contrast

La misma consulta podría tener:

```text
CompiledQueryCache = CACHEABLE
ResultCache = NON_CACHEABLE
```

---

# 88. Session-sensitive functions

Ejemplos conceptuales:

```text
CURRENT_USER
SESSION_SETTING(...)
```

requieren visibility/session context en key o bypass.

---

# 89. Security context

Una query puede depender de:

```text
current authenticated user
authorization scope
row-level policy
tenant
```

aunque esos valores no aparezcan directamente en SQL.

---

# 90. ResultVisibilityContext

Se definirá:

```php
final readonly class ResultVisibilityContext
{
    public function __construct(
        public DataDomainId $dataDomain,
        public ?SecurityScopeFingerprint $security,
        public ?TenantScopeFingerprint $tenant,
        public ?ShardScopeFingerprint $shard,
        public ?SessionVisibilityFingerprint $session,
    ) {}
}
```

---

# 91. Context minimization

Solo dimensiones que realmente afectan la visibilidad deberán incluirse.

---

# 92. Security first

Pero:

```text
uncertain visibility dependency
→ bypass
```

es preferible a compartir datos entre scopes incompatibles.

---

# 93. Tenant isolation

Nunca:

```text
Tenant A result
→ Tenant B
```

salvo que la consulta opere explícitamente sobre un dominio compartido cuya equivalencia haya sido demostrada.

---

# 94. Tenant identity

Podrá utilizarse un:

```text
TenantScopeFingerprint
```

en vez de exponer el tenant identifier físico.

---

# 95. Shared catalog

Si una tabla es global:

```text
countries
currencies
```

la política podrá declarar:

```text
GLOBAL_DATA_SCOPE
```

permitiendo compartir cache.

---

# 96. Sharding

Una query ejecutada en:

```text
Shard A
```

no deberá reutilizar el resultado de:

```text
Shard B
```

salvo que se trate explícitamente de datos replicados/globales equivalentes.

---

# 97. Shard identity

```text
ShardDataDomainFingerprint
```

deberá formar parte de la key cuando corresponda.

---

# 98. Distributed queries

Una consulta fan-out:

```text
Shard A
Shard B
Shard C
   ↓
Merged Result
```

podrá cachear:

```text
MergedDistributedResult
```

solo si la identidad representa todos los data domains participantes.

---

# 99. Replica awareness

Un resultado obtenido desde una réplica puede estar atrasado.

Por tanto:

```text
ResultFreshness
```

y:

```text
ReplicaFreshness
```

son conceptos distintos.

---

# 100. Replica provenance

El cache entry podrá registrar:

```text
source role
replication position
observed lag
consistency token
```

cuando estén disponibles.

---

# 101. Replica cache rule

Si una query requiere:

```text
read-your-own-writes
```

no podrá reutilizar un cache entry cuyo origen no satisfaga esa garantía.

---

# 102. Sticky reads

El sistema definido en:

```text
180_DATABASE_STICKY_CONNECTION_SYSTEM.md
```

deberá interactuar con Result Cache.

---

# 103. Sticky context

Si una operación está marcada:

```text
writer required
```

un cache hit no deberá eludir esa requirement salvo que la política del cache garantice una versión compatible.

---

# 104. Consistency token

VoltStack podrá utilizar:

```php
interface DataConsistencyToken
{
    public function isAtLeast(
        DataConsistencyToken $required,
    ): bool;
}
```

cuando el backend/platform pueda ofrecer un concepto equivalente.

---

# 105. No invented tokens

Si la plataforma no puede proporcionar esa evidencia:

```text
UNKNOWN
```

no será equivalente a:

```text
compatible
```

---

# 106. Failover

Después de failover:

```text
old primary
→ new primary
```

puede existir incertidumbre sobre resultados recientes.

---

# 107. Topology generation

Podrá utilizarse:

```text
DatabaseTopologyGeneration
```

para invalidar o reevaluar entries sensibles.

---

# 108. Failover does not always invalidate

Resultados de datos estáticos pueden seguir siendo correctos.

Por tanto, no deberá vaciarse todo el cache indiscriminadamente salvo policy.

---

# 109. Negative caching

Un resultado vacío también puede cachearse.

Ejemplo:

```text
User#999 not found
```

---

# 110. NegativeResultPayload

```php
final readonly class EmptyResultPayload implements CanonicalResultPayload
{
}
```

---

# 111. Negative caching risk

Un registro puede crearse inmediatamente después.

Por tanto, negative entries suelen requerir:

```text
short TTL
```

o invalidación precisa.

---

# 112. Null vs empty

Distinguir:

```text
scalar NULL
```

de:

```text
no row
```

de:

```text
empty collection
```

---

# 113. first()

```text
no row
```

no es necesariamente igual a:

```text
row with nullable value = NULL
```

---

# 114. Pagination

Cada página normalmente será un resultado distinto.

---

# 115. Offset pagination key

Debe incluir:

```text
offset
limit
ordering
filters
```

---

# 116. Cursor pagination

Debe incluir:

```text
cursor value
direction
limit
ordering identity
```

---

# 117. Cursor sensitivity

El cursor podrá contener datos sensibles.

Deberá fingerprintarse, no exponerse directamente en physical key.

---

# 118. Pagination invalidation

Insertar una fila al inicio de una lista puede afectar múltiples páginas.

La invalidación exacta puede ser costosa.

---

# 119. Conservative pagination policy

Puede invalidarse:

```text
entire query family
```

o utilizar TTL corto.

---

# 120. Query family

Podrá definirse:

```text
QueryFamilyFingerprint
```

que excluya pagination position pero preserve filtros/orden.

Útil para invalidación grupal.

---

# 121. Aggregates

Consultas como:

```text
COUNT
SUM
AVG
MIN
MAX
```

son candidatas naturales al cache.

---

# 122. Aggregate invalidation

Sin embargo, una sola mutation puede cambiar el aggregate.

Por tanto requieren dependency tracking adecuado.

---

# 123. COUNT

```text
COUNT(active users)
```

puede cambiar por:

- insert;
- delete;
- update de active.

---

# 124. Exact predicate invalidation

Determinar si:

```text
UPDATE users SET name = ...
```

afecta `COUNT(active)` requiere mutation semantic analysis.

VoltStack podrá comenzar con invalidación coarse.

---

# 125. Projection cache

Una query que devuelve:

```text
DTO
```

no deberá almacenar necesariamente el objeto DTO.

Preferir:

```text
canonical projection payload
```

y reconstruirlo.

---

# 126. DTO versioning

Si se decide almacenar DTO materializado, deberá incluir:

```text
DTO schema/version
```

pero el diseño base evitará esa dependencia.

---

# 127. Streaming

`StreamingResult` no deberá almacenarse automáticamente.

---

# 128. Why

Un stream puede ser:

- enorme;
- parcial;
- cancelado;
- consumido incrementalmente.

---

# 129. Complete result requirement

Solo un resultado cuya completitud haya sido confirmada podrá convertirse en ordinary cached collection.

---

# 130. Partial stream

```text
1000 expected
consumer stopped at 200
```

no podrá almacenarse como resultado completo.

---

# 131. Coverage

```php
enum ResultCoverage
{
    case COMPLETE;
    case PARTIAL;
    case UNKNOWN;
}
```

Default:

```text
PARTIAL/UNKNOWN
→ do not cache as complete result
```

---

# 132. Size governance

Antes de almacenar:

```text
ResultSize
<=
Policy.maxEntrySize
```

---

# 133. Large result

Un resultado de 500 MB no deberá enviarse automáticamente a Redis.

---

# 134. Admission policy

```php
interface ResultCacheAdmissionPolicy
{
    public function decide(
        ResultCacheCandidate $candidate,
    ): ResultCacheAdmissionDecision;
}
```

---

# 135. Candidate dimensions

Podrá considerar:

```text
result bytes
row count
query cost
execution duration
TTL
expected reuse
dependency coverage
backend capacity
```

---

# 136. Expensive query

Una consulta que tarda 2 segundos y produce 10 KB es buen candidato.

---

# 137. Cheap query

Una consulta que tarda:

```text
0.1 ms
```

puede ser más barata que:

```text
remote cache lookup
```

---

# 138. Cache effectiveness

```text
Benefit
≈
AvoidedDatabaseCost
-
CacheLookupCost
-
SerializationCost
-
InvalidationCost
```

---

# 139. L1 cache

Podrá existir:

```text
Process-local Result Cache
```

pero requerirá especial cuidado con staleness.

---

# 140. L2 cache

Un backend compartido:

```text
Redis
```

será más natural para resultados compartidos entre workers.

---

# 141. Tiered result cache

```text
L1
 ↓ miss
L2
 ↓ miss
DB
```

---

# 142. L1 stale risk

Si L2 invalida una entrada pero L1 no recibe la invalidación:

```text
L1
→ stale
```

---

# 143. Generation validation

Por ello L1 podrá usar:

```text
dependency generation
```

para verificar entries.

---

# 144. Pub/sub invalidation

Opcionalmente:

```text
L2 invalidation
→ event
→ worker L1 eviction
```

---

# 145. Pub/sub reliability

Pub/sub perdido no deberá ser la única garantía si la política exige mayor consistencia.

---

# 146. TTL safety net

Un TTL bounded podrá limitar cuánto tiempo permanece una invalidación perdida.

---

# 147. Stale-while-revalidate

Modelo:

```text
FRESH
→ serve

STALE_ACCEPTABLE
→ serve stale
→ one worker refreshes

EXPIRED
→ synchronous miss
```

---

# 148. SWR policy

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

# 149. SWR not for every query

No usar automáticamente para:

- balances;
- permissions;
- inventory reservations;
- security decisions;
- transaction-sensitive data.

---

# 150. Single-flight

Cold cache:

```text
100 requests
→ same expensive query
```

no debería causar necesariamente:

```text
100 DB executions
```

---

# 151. ResultCacheSingleFlight

```text
first request
→ producer

others
→ bounded wait
```

---

# 152. Single-flight scope

Podrá ser:

```text
PROCESS
DISTRIBUTED
```

según backend.

---

# 153. Lock timeout

Un distributed cache lock deberá ser:

- bounded;
- leased;
- recoverable.

Nunca infinito.

---

# 154. Producer crash

Los consumidores deberán poder continuar después de expiración del lease.

---

# 155. Cache stampede

Además de single-flight podrán existir:

```text
TTL jitter
probabilistic early refresh
stale-while-revalidate
```

---

# 156. TTL jitter

Evita que miles de entries expiren simultáneamente.

---

# 157. ResultCacheEntry

```php
final readonly class ResultCacheEntry
{
    public function __construct(
        public ResultCacheKey $key,
        public CanonicalResultPayload $payload,
        public ResultDependencySet $dependencies,
        public ResultCacheProvenance $provenance,
        public ResultCacheFreshnessMetadata $freshness,
        public ResultCacheEntryMetadata $metadata,
    ) {}
}
```

---

# 158. Provenance

```php
final readonly class ResultCacheProvenance
{
    public function __construct(
        public DataDomainId $dataDomain,
        public DatabaseRole $sourceRole,
        public ?DataConsistencyToken $consistencyToken,
        public ?ReplicaObservation $replica,
    ) {}
}
```

---

# 159. Provenance ≠ secret infrastructure dump

No deberá incluir:

- DB credentials;
- full connection strings;
- sensitive host information innecesaria.

---

# 160. CreatedAt

Cada entry deberá registrar:

```text
createdAt
```

con reloj monotónico/seguro donde corresponda para cálculos de edad.

---

# 161. Expiration

Preferir almacenar:

```text
freshUntil
staleUntil
```

o metadata equivalente.

---

# 162. Clock skew

En distributed cache, depender de relojes de múltiples workers requiere considerar skew.

Cuando sea posible, usar expiración nativa del backend y metadata coherente.

---

# 163. Serialization

Result payloads deberán usar codec explícito.

---

# 164. No PHP unserialize inseguro

Prohibido usar payloads externos no confiables mediante:

```php
unserialize($payload);
```

sin un modelo seguro.

---

# 165. ResultCacheCodec

```php
interface ResultCacheCodec
{
    public function encode(ResultCacheEntry $entry): string;

    public function decode(string $payload): ResultCacheEntry;
}
```

---

# 166. Codec version

Todo payload deberá incluir:

```text
format version
```

---

# 167. Schema evolution

Un cambio de metadata podrá hacer incompatible un cached row payload.

---

# 168. Metadata generation

Por tanto, entries ORM/projection-sensitive podrán depender de:

```text
EntityMetadataGeneration
TypeRegistryGeneration
HydrationPlanGeneration
```

---

# 169. Raw scalar result

Un:

```text
COUNT(*)
```

puede requerir menos dependencies estructurales.

---

# 170. Minimal complete dependencies

Regla:

```text
ResultCacheDependencies
=
minimum complete dependency set required for safe reuse
```

---

# 171. Data dependencies vs structural dependencies

Distinguir:

```text
StructuralDependencies
```

de:

```text
DataDependencies
```

---

# 172. Structural dependencies

Ejemplos:

```text
type generation
entity mapping
result shape
query semantics
```

---

# 173. Data dependencies

Ejemplos:

```text
users table generation
User#42 generation
catalog generation
```

---

# 174. Entry validity

Formalmente:

```text
Valid(E, C)
=
StructuralCompatible(E, C)
∧ DataDependenciesCompatible(E, C)
∧ FreshnessAccepted(E, C)
∧ VisibilityCompatible(E, C)
```

---

# 175. Data versioning

Una infraestructura futura podrá utilizar:

```text
DataVersionVector
```

---

# 176. Version vector

Ejemplo:

```text
users: 17
orders: 81
products: 34
```

El cache entry almacena:

```text
[17, 81, 34]
```

---

# 177. Validation

En lookup:

```text
current versions
==
cached dependency versions
```

entonces el entry sigue siendo compatible.

---

# 178. Version lookup overhead

Consultar demasiadas generations puede eliminar el beneficio del cache.

Por ello deberán agruparse o mantenerse eficientemente.

---

# 179. Generation cache

Los dependency generations podrán tener:

```text
fast local snapshot
```

con mecanismo de sincronización apropiado.

---

# 180. Mutation integration

Persistence Engine podrá emitir:

```text
DataMutationSummary
```

después de operaciones.

---

# 181. DataMutationSummary

Ejemplo:

```text
EntityType: User
EntityId: 42
Table: users
ChangedFields:
    email
    status
Operation:
    UPDATE
```

---

# 182. Cache invalidation planner

```php
interface ResultCacheInvalidationPlanner
{
    public function plan(
        DataMutationSummary $mutation,
    ): ResultCacheInvalidationPlan;
}
```

---

# 183. Invalidation plan

Puede contener:

```text
increment generation
evict keys
evict tags
publish invalidation
schedule refresh
```

---

# 184. Bulk mutations

```text
UPDATE users SET active = false ...
```

puede no enumerar todos los EntityIds.

Entonces:

```text
table/entity-type generation invalidation
```

será preferible.

---

# 185. Raw mutation

Una raw SQL mutation puede impedir dependency precision.

Política:

```text
invalidate broad affected domain
```

o exigir explicit tags.

---

# 186. External mutations

La base de datos puede cambiar fuera de VoltStack:

```text
admin tool
another service
ETL
trigger
stored procedure
```

---

# 187. Important limitation

Application-level invalidation no puede detectar automáticamente todas las mutaciones externas.

---

# 188. External mutation strategies

Podrán utilizarse:

```text
TTL
CDC
database change streams
shared invalidation service
explicit external hooks
version tables
```

como extensiones.

---

# 189. Result cache correctness model

La documentación/configuración deberá indicar claramente:

```text
What mutations can invalidate this cache?
```

---

# 190. Cache policy descriptor

```php
final readonly class ResultCachePolicy
{
    public function __construct(
        public ResultCacheConsistency $consistency,
        public Duration $freshFor,
        public ?Duration $staleFor,
        public ResultCacheScope $scope,
        public ResultCacheTransactionPolicy $transactionPolicy,
        public ResultCacheAdmissionPolicyId $admission,
    ) {}
}
```

---

# 191. Cache scopes

```php
enum ResultCacheScope
{
    case OPERATION;
    case PROCESS;
    case DISTRIBUTED;
}
```

---

# 192. Operation memoization

Dentro de una operación:

```text
same query
same parameters
same visibility
```

puede reutilizarse sin remote cache.

---

# 193. Operation memoization and mutations

Si ocurre una mutation dentro del mismo operation scope:

```text
memoized results
```

afectados deberán invalidarse.

---

# 194. Process scope

Más rápido, pero requiere sincronización/generation awareness.

---

# 195. Distributed scope

Permite compartir entre:

```text
FrankenPHP workers
RoadRunner workers
OpenSwoole workers
multiple servers
```

---

# 196. Cross-application cache

Dos aplicaciones no deberán compartir namespace salvo configuración explícita y compatibilidad demostrada.

---

# 197. Application generation

Physical key podrá incluir:

```text
ApplicationId
DeploymentGeneration
DatabaseCacheNamespace
```

---

# 198. Namespace collision

Dos aplicaciones con:

```text
users
```

no deberán colisionar accidentalmente.

---

# 199. Persistent runtime

El sistema deberá separar:

```text
Shared immutable cache infrastructure
```

de:

```text
Scoped lookup context
```

---

# 200. Request leakage

Nunca conservar:

```text
authenticated user object
request
EntityManager
TransactionContext
```

dentro de shared cache entry.

---

# 201. OpenSwoole

Todo L1 compartido entre coroutines deberá ser concurrency-safe.

---

# 202. FrankenPHP

Workers persistentes hacen especialmente valioso L1, pero aumentan riesgo de stale state si no existe reset/generation validation.

---

# 203. RoadRunner

Mismas reglas:

```text
worker-local cache
≠
global truth
```

---

# 204. Cache backend failure

Default:

```text
FAIL_OPEN
```

para ordinary Result Cache.

---

# 205. FAIL_OPEN

```text
cache unavailable
→ execute database query
```

---

# 206. Security-critical cache

Si una aplicación utiliza Result Cache para una operación cuya política exige garantías especiales, podrá elegir:

```text
FAIL_CLOSED
```

pero deberá ser explícito.

---

# 207. Cache backend must not become DB correctness dependency accidentally

La aplicación deberá funcionar sin cache para las políticas ordinarias.

---

# 208. Corrupted entry

```text
decode failure
invalid version
invalid shape
invalid dependency metadata
```

deberá producir:

```text
evict
+
MISS
```

---

# 209. Never partially hydrate corrupted cache

No deberá intentarse “rescatar” filas incompletas.

---

# 210. Compression

Payloads grandes podrán comprimirse.

---

# 211. Compression threshold

Solo cuando:

```text
compression benefit
>
CPU overhead
```

según policy.

---

# 212. Encryption

Si cache backend no se considera suficientemente confiable para determinados datos:

```text
encrypted payload
```

podrá ser requerido.

---

# 213. Sensitive data classification

Result Cache deberá poder recibir metadata como:

```text
PUBLIC
INTERNAL
SENSITIVE
RESTRICTED
```

desde políticas superiores.

---

# 214. Restricted data

Una policy podrá declarar:

```text
DISTRIBUTED_CACHE_FORBIDDEN
```

---

# 215. No cache secrets by default

Campos como:

```text
password hashes
access tokens
refresh tokens
private credentials
```

no deberán cachearse indiscriminadamente.

---

# 216. Query projection minimization

Una buena práctica:

```text
SELECT only required columns
```

también reduce exposición del cache.

---

# 217. Authorization

Result Cache no deberá asumir:

```text
same SQL
=
same authorization scope
```

---

# 218. Authorization fingerprint

Si autorización altera visibilidad:

```text
AuthorizationScopeFingerprint
```

deberá participar o la query será no cacheable.

---

# 219. Row-level security

Cuando DB RLS dependa de session state:

```text
session visibility context
```

deberá representarse.

Si no puede representarse con seguridad:

```text
BYPASS
```

---

# 220. Cache key explosion

Incluir:

```text
UserId
RoleSet
Permissions
Tenant
Locale
Currency
...
```

puede producir enorme cardinalidad.

---

# 221. Policy response

El sistema podrá decidir:

```text
cache not beneficial
→ bypass
```

aunque sea técnicamente posible.

---

# 222. Locale

Solo deberá formar parte de Result Cache identity si cambia realmente los datos obtenidos.

---

# 223. Presentation formatting

No deberá contaminar Database Result Cache si ocurre después de la capa DB.

---

# 224. Currency conversion

Si ocurre dentro del SQL/result semantics:

```text
currency
```

sí puede ser relevante.

---

# 225. Diagnostics

API conceptual:

```php
DB::resultCache()->explain($query);
```

---

# 226. Explain report

```text
RESULT CACHE EXPLAIN

Query:
    products where category_id = :id

Cacheable:
    YES

Policy:
    catalog

Query Fingerprint:
    qfp:v3:...

Parameter Fingerprint:
    pfp:v2:...

Data Domain:
    catalog

Dependencies:
    products
    categories

Dependency Coverage:
    COMPLETE

Freshness:
    FRESH

Transaction:
    NONE

Source:
    DISTRIBUTED_CACHE

Decision:
    HIT
```

---

# 227. Transaction explain

```text
Transaction:
    ACTIVE

Shared Cache Policy:
    BYPASS

Reason:
    TRANSACTION_VISIBILITY_NOT_REPRESENTABLE

Decision:
    MISS/BYPASS
```

---

# 228. Replica explain

```text
Cached Source:
    REPLICA

Observed Replica Position:
    ...

Required Consistency:
    READ_YOUR_WRITES

Compatibility:
    UNKNOWN

Decision:
    BYPASS
```

---

# 229. ResultCacheDecision

```php
enum ResultCacheDecision
{
    case HIT;
    case MISS;
    case BYPASS;
    case STALE_HIT;
    case REFRESH_REQUIRED;
    case INVALID;
}
```

---

# 230. Bypass reasons

```php
enum ResultCacheBypassReason
{
    case DISABLED;
    case NON_CACHEABLE_QUERY;
    case UNKNOWN_CACHEABILITY;
    case ACTIVE_TRANSACTION;
    case LOCKING_QUERY;
    case MUTATION_QUERY;
    case NON_DETERMINISTIC_RESULT;
    case UNKNOWN_VISIBILITY;
    case UNKNOWN_DEPENDENCIES;
    case RESULT_TOO_LARGE;
    case SECURITY_POLICY;
    case EXPLICIT_BYPASS;
}
```

---

# 231. Miss reasons

```text
ENTRY_NOT_FOUND
EXPIRED
DEPENDENCY_CHANGED
GENERATION_CHANGED
VISIBILITY_MISMATCH
CONSISTENCY_MISMATCH
PAYLOAD_INVALID
BACKEND_UNAVAILABLE
```

---

# 232. Telemetry

Eventos:

```text
ResultCacheLookupStarted
ResultCacheHit
ResultCacheMiss
ResultCacheBypass
ResultCacheStaleHit
ResultCacheStored
ResultCacheRejected
ResultCacheInvalidated
ResultCacheRefreshStarted
ResultCacheRefreshCompleted
ResultCacheCorruptionDetected
```

---

# 233. Metrics

```text
db.result_cache.hit
db.result_cache.miss
db.result_cache.bypass
db.result_cache.stale_hit
db.result_cache.store
db.result_cache.eviction
db.result_cache.invalidation
db.result_cache.lookup.duration
db.result_cache.payload.bytes
db.result_cache.saved_db_time
```

---

# 234. Bounded labels

Permitidos:

```text
backend=memory|redis
decision=hit|miss|bypass
scope=process|distributed
```

No:

```text
user_id=12345
query_hash=...
tenant_id=...
```

como high-cardinality metric labels.

---

# 235. Tracing

Span conceptual:

```text
database.result_cache.lookup
```

Atributos bounded:

```text
cache.hit
cache.scope
cache.policy
cache.freshness
dependency.coverage
```

---

# 236. No raw result telemetry

Nunca registrar el payload completo por default.

---

# 237. Testing architecture

Se requerirán:

```text
unit tests
integration tests
transaction tests
replica tests
tenant tests
shard tests
concurrency tests
security tests
performance tests
persistent-runtime tests
```

---

# 238. Basic hit test

```text
Query A
→ DB
→ cache

Query A again
→ cache
→ no DB
```

---

# 239. Parameter test

```text
User#10
≠
User#11
```

deberán producir keys distintas.

---

# 240. Canonical parameter test

Dos representaciones equivalentes del mismo valor lógico deberán producir mismo fingerprint cuando el Type System lo determine.

---

# 241. Projection test

```text
SELECT id
```

no deberá compartir payload con:

```text
SELECT id, email
```

---

# 242. Empty result test

```text
no rows
```

deberá cachearse correctamente cuando la policy lo permita.

---

# 243. NULL test

```text
scalar NULL
```

deberá distinguirse de:

```text
no row
```

---

# 244. TTL test

Verificar:

```text
FRESH
→ STALE_ACCEPTABLE
→ EXPIRED
```

---

# 245. Mutation invalidation test

```text
cache query
update dependency
commit
query again
```

deberá producir:

```text
MISS
```

bajo invalidation-driven policy.

---

# 246. Rollback test

Una mutación rollbacked no deberá introducir nuevos datos uncommitted al shared cache.

---

# 247. Unknown commit test

Debe aplicarse la política conservadora configurada.

---

# 248. Active transaction test

Shared cache deberá bypass por default.

---

# 249. Lock test

```text
FOR UPDATE
```

nunca deberá ser satisfecho desde ordinary Result Cache.

---

# 250. Read-your-own-writes test

Una mutation seguida de read en misma transaction deberá observar semantics transaccionales, no cached stale value.

---

# 251. Tenant isolation test

```text
Tenant A
```

nunca recibe entry de:

```text
Tenant B
```

---

# 252. Shared global data test

Datos explícitamente globales sí podrán compartir entries.

---

# 253. Shard isolation test

Mismo query/parameters en shards distintos deberá separar data domains.

---

# 254. Replica consistency test

Un cached replica result incompatible con required consistency deberá bypass.

---

# 255. Sticky read test

Un sticky-writer requirement deberá respetarse.

---

# 256. Streaming test

Un stream parcialmente consumido no deberá almacenarse como COMPLETE.

---

# 257. Large result test

Entries sobre `maxEntryBytes` deberán rechazarse.

---

# 258. Stampede test

100 concurrent misses deberán poder colapsarse mediante single-flight.

---

# 259. Producer failure test

No deberá dejar distributed lock permanente.

---

# 260. Corruption test

Payload corrupto:

```text
→ INVALID
→ eviction
→ DB execution
```

---

# 261. Backend outage test

Bajo FAIL_OPEN:

```text
Redis unavailable
→ DB continues
```

---

# 262. Worker leakage test

Un request no deberá encontrar security/transaction context residual del anterior.

---

# 263. Performance test

Comparar:

```text
DB query latency
L1 hit latency
L2 hit latency
decode latency
hydration latency
```

---

# 264. Cost test

Para queries baratas deberá verificarse que el cache no degrade significativamente performance.

---

# 265. Memory test

L1 deberá permanecer dentro de:

```text
configured memory budget
```

---

# 266. Security test

Verificar:

- no raw sensitive parameter keys;
- no cross-tenant leakage;
- no unsafe deserialization;
- no security-context mismatch;
- no unrestricted sensitive payload storage.

---

# 267. Error hierarchy

```text
ResultCacheException
├── ResultCacheKeyException
├── ResultCacheFingerprintException
├── ResultCacheLookupException
├── ResultCacheWriteException
├── ResultCacheInvalidationException
├── ResultCacheDependencyException
├── ResultCacheConsistencyException
├── ResultCacheVisibilityException
├── ResultCacheSerializationException
├── ResultCacheCorruptionException
├── ResultCacheAdmissionException
├── ResultCacheRefreshException
└── ResultCacheInvariantViolationException
```

---

# 268. Directory structure

```text
src/Quantum/Database/Cache/Result/
│
├── ResultCacheManager.php
├── ResultCacheKey.php
├── ResultCacheEntry.php
├── ResultCachePolicy.php
├── ResultCacheDecision.php
├── ResultCacheContext.php
├── ResultCacheScope.php
├── ResultCacheability.php
│
├── Fingerprint/
│   ├── ParameterFingerprint.php
│   ├── ParameterFingerprinter.php
│   ├── ResultShapeFingerprint.php
│   ├── ResultVisibilityFingerprint.php
│   └── QueryFamilyFingerprint.php
│
├── Payload/
│   ├── CanonicalResultPayload.php
│   ├── ScalarResultPayload.php
│   ├── RowResultPayload.php
│   ├── RowCollectionResultPayload.php
│   ├── TupleResultPayload.php
│   ├── AggregateResultPayload.php
│   └── EmptyResultPayload.php
│
├── Freshness/
│   ├── ResultCacheFreshness.php
│   ├── ResultFreshnessPolicy.php
│   ├── ResultCacheFreshnessMetadata.php
│   └── StaleWhileRevalidatePolicy.php
│
├── Consistency/
│   ├── ResultCacheConsistency.php
│   ├── DataConsistencyToken.php
│   ├── ResultCacheProvenance.php
│   └── ResultConsistencyValidator.php
│
├── Visibility/
│   ├── ResultVisibilityContext.php
│   ├── SecurityScopeFingerprint.php
│   ├── TenantScopeFingerprint.php
│   ├── ShardScopeFingerprint.php
│   └── SessionVisibilityFingerprint.php
│
├── Dependency/
│   ├── ResultDependency.php
│   ├── ResultDependencySet.php
│   ├── ResultDependencyAnalyzer.php
│   ├── DependencyCoverage.php
│   ├── DataVersion.php
│   └── DataVersionVector.php
│
├── Invalidation/
│   ├── ResultCacheInvalidationPlanner.php
│   ├── ResultCacheInvalidationPlan.php
│   ├── DataMutationSummary.php
│   ├── ResultCacheInvalidator.php
│   └── ResultCacheGenerationManager.php
│
├── Admission/
│   ├── ResultCacheAdmissionPolicy.php
│   ├── ResultCacheCandidate.php
│   └── ResultCacheAdmissionDecision.php
│
├── Store/
│   ├── ResultCacheStore.php
│   ├── L1ResultCache.php
│   ├── L2ResultCache.php
│   ├── TieredResultCache.php
│   └── NullResultCache.php
│
├── Codec/
│   ├── ResultCacheCodec.php
│   ├── ResultCacheEncoder.php
│   ├── ResultCacheDecoder.php
│   └── ResultCachePayloadVersion.php
│
├── Stampede/
│   ├── ResultCacheSingleFlight.php
│   ├── ResultCacheLease.php
│   └── ResultCacheRefreshCoordinator.php
│
├── Diagnostics/
│   ├── ResultCacheInspector.php
│   ├── ResultCacheExplainer.php
│   ├── ResultCacheDiagnosticReport.php
│   ├── ResultCacheMissReason.php
│   └── ResultCacheBypassReason.php
│
├── Telemetry/
│   ├── ResultCacheTelemetry.php
│   ├── ResultCacheHitEvent.php
│   ├── ResultCacheMissEvent.php
│   └── ResultCacheInvalidatedEvent.php
│
└── Exception/
    ├── ResultCacheException.php
    ├── ResultCacheConsistencyException.php
    ├── ResultCacheVisibilityException.php
    ├── ResultCacheCorruptionException.php
    └── ResultCacheInvariantViolationException.php
```

---

# 269. Public API

Ejemplos conceptuales:

```php
User::query()
    ->where('active', true)
    ->cacheFor(minutes: 5)
    ->get();
```

```php
Product::query()
    ->cacheUsing('catalog')
    ->paginate(20);
```

```php
DB::query()
    ->cache(false)
    ->get();
```

---

# 270. Explicit refresh

Podrá existir:

```php
$query
    ->refreshCache()
    ->get();
```

Semántica:

```text
BYPASS READ
→ execute DB
→ replace cache
```

---

# 271. Remember semantics

También podría ofrecerse:

```php
->rememberFor(...)
```

pero deberá mantenerse una terminología coherente con el resto de VoltStack.

Preferencia:

```text
cacheFor()
cacheUsing()
withoutResultCache()
refreshResultCache()
```

---

# 272. ResultCacheManager

```php
interface ResultCacheManager
{
    public function lookup(
        QueryExecutionRequest $request,
        ResultCacheContext $context,
    ): ResultCacheLookup;

    public function store(
        ResultCacheCandidate $candidate,
    ): void;

    public function invalidate(
        ResultCacheInvalidationPlan $plan,
    ): void;
}
```

---

# 273. Manager boundaries

El manager:

Sí:

```text
identity
lookup
freshness
consistency validation
visibility validation
admission
store
invalidation coordination
```

No:

```text
SQL compilation
DB execution
entity persistence
transaction commit
authorization decisions
```

---

# 274. Execution integration

```text
QueryExecutor
    │
    ▼
ResultCacheCoordinator
    │
    ├── HIT ───────────────┐
    │                      │
    └── MISS               │
         │                 │
         ▼                 │
     Statement             │
         │                 │
         ▼                 │
      Database             │
         │                 │
         ▼                 │
   Canonical Result        │
         │                 │
         ▼                 │
    Cache Store            │
         │                 │
         └────────┬────────┘
                  ▼
              Result API
```

---

# 275. Cache must not bypass execution semantics accidentally

Antes de cache lookup deberán conocerse suficientes propiedades como:

```text
locking intent
transaction state
data domain
visibility context
result shape
cache policy
```

---

# 276. Cache lookup ordering

No deberá realizarse:

```text
cache HIT
→ later discover query was FOR UPDATE
```

La cacheability debe evaluarse antes.

---

# 277. Recommended decision pipeline

```text
QueryExecutionRequest
        ↓
Cache Policy Resolution
        ↓
Cacheability Analysis
        ↓
Visibility Resolution
        ↓
Transaction Compatibility
        ↓
Data Domain Resolution
        ↓
Key Construction
        ↓
Lookup
        ↓
Freshness Validation
        ↓
Dependency Validation
        ↓
Consistency Validation
        ↓
HIT / MISS / BYPASS
```

---

# 278. Architectural invariants

## DB-RCACHE-001
Result Cache almacenará resultados de consultas, no Query Plans.

## DB-RCACHE-002
Result Cache será distinto de Query Cache.

## DB-RCACHE-003
Result Cache será distinto de Entity Cache.

## DB-RCACHE-004
Result Cache será distinto de IdentityMap.

## DB-RCACHE-005
Result Cache podrá evitar physical DB execution.

## DB-RCACHE-006
No toda query de lectura será cacheable.

## DB-RCACHE-007
UNKNOWN cacheability producirá bypass.

## DB-RCACHE-008
Result Cache identity incluirá parameter values semánticamente relevantes.

## DB-RCACHE-009
Parameter values serán canonicalizados.

## DB-RCACHE-010
Sensitive parameter values no aparecerán directamente en physical keys.

## DB-RCACHE-011
Query semantic identity participará en la key.

## DB-RCACHE-012
Result shape participará cuando afecte payload semantics.

## DB-RCACHE-013
Visibility context participará cuando afecte datos observables.

## DB-RCACHE-014
Tenant isolation deberá preservarse.

## DB-RCACHE-015
Shard data isolation deberá preservarse.

## DB-RCACHE-016
Global data sharing requerirá declaración explícita.

## DB-RCACHE-017
Managed Entity objects no se almacenarán directamente en shared Result Cache.

## DB-RCACHE-018
Cached entity rows pasarán nuevamente por Hydration/IdentityMap.

## DB-RCACHE-019
Cache hit no podrá crear segunda instancia managed de misma EntityKey.

## DB-RCACHE-020
Cache hit no sobrescribirá silenciosamente dirty managed entity.

## DB-RCACHE-021
Canonical result payload será independiente de EntityManager.

## DB-RCACHE-022
Canonical result payload será independiente de Connection.

## DB-RCACHE-023
Canonical result payload será independiente de TransactionContext.

## DB-RCACHE-024
Driver-specific runtime resources no serán cacheados.

## DB-RCACHE-025
TTL no será presentado como strong consistency.

## DB-RCACHE-026
Freshness y data consistency serán conceptos separados.

## DB-RCACHE-027
UNKNOWN freshness no será considerada fresh.

## DB-RCACHE-028
Expired entry será miss salvo política explícita.

## DB-RCACHE-029
Stale result solo podrá servirse bajo política explícita.

## DB-RCACHE-030
SWR será opt-in/policy-driven.

## DB-RCACHE-031
Security-sensitive queries no usarán SWR indiscriminadamente.

## DB-RCACHE-032
Result dependencies serán explícitas cuando se use invalidation-driven consistency.

## DB-RCACHE-033
Dependency coverage UNKNOWN no será tratada como COMPLETE.

## DB-RCACHE-034
Conservative invalidation será preferida a falsa precisión.

## DB-RCACHE-035
Mutation invalidation deberá respetar transaction outcome.

## DB-RCACHE-036
Uncommitted data nunca se publicará en shared Result Cache.

## DB-RCACHE-037
Confirmed rollback no publicará mutation results.

## DB-RCACHE-038
UNKNOWN commit no será asumido rollback.

## DB-RCACHE-039
UNKNOWN commit podrá requerir conservative invalidation.

## DB-RCACHE-040
Confirmed commit + invalidation failure será observable.

## DB-RCACHE-041
Outbox podrá utilizarse para invalidation reliability.

## DB-RCACHE-042
Outbox no será presentado como instant consistency.

## DB-RCACHE-043
Active transaction hará bypass del shared Result Cache por default.

## DB-RCACHE-044
Transaction-local memoization será distinta de shared Result Cache.

## DB-RCACHE-045
FOR UPDATE no será satisfecho desde Result Cache.

## DB-RCACHE-046
FOR SHARE no será satisfecho desde ordinary Result Cache.

## DB-RCACHE-047
Mutation queries no serán ordinary Result Cache candidates.

## DB-RCACHE-048
RETURNING de mutation no será tratado como cacheable SELECT.

## DB-RCACHE-049
Non-deterministic result será non-cacheable por default.

## DB-RCACHE-050
Structural Query Cache podrá seguir cacheando queries non-cacheable para Result Cache.

## DB-RCACHE-051
Session-dependent visibility deberá representarse o bypass.

## DB-RCACHE-052
Authorization-dependent visibility deberá representarse o bypass.

## DB-RCACHE-053
RLS desconocido producirá bypass.

## DB-RCACHE-054
Cross-tenant cache leakage estará prohibido.

## DB-RCACHE-055
Cross-shard leakage estará prohibido.

## DB-RCACHE-056
Replica provenance podrá formar parte de consistency validation.

## DB-RCACHE-057
Replica result no satisfará read-your-own-writes sin evidencia.

## DB-RCACHE-058
Sticky-writer semantics deberán respetarse.

## DB-RCACHE-059
UNKNOWN consistency-token compatibility no será tratada como compatible.

## DB-RCACHE-060
Failover podrá invalidar topology-sensitive entries.

## DB-RCACHE-061
Failover no exigirá global flush sin policy.

## DB-RCACHE-062
Negative results podrán cachearse explícitamente.

## DB-RCACHE-063
No-row será distinto de scalar NULL.

## DB-RCACHE-064
Empty collection será distinto de no-row cuando la API lo requiera.

## DB-RCACHE-065
Pagination position participará en result identity.

## DB-RCACHE-066
Cursor values sensibles serán fingerprinted.

## DB-RCACHE-067
Pagination invalidation podrá ser coarse.

## DB-RCACHE-068
Aggregates podrán cachearse.

## DB-RCACHE-069
Aggregate invalidation deberá considerar mutations dependientes.

## DB-RCACHE-070
DTO objects no serán shared-cache payload por default.

## DB-RCACHE-071
Streaming result no será cacheado automáticamente.

## DB-RCACHE-072
Partial stream no será marcado COMPLETE.

## DB-RCACHE-073
UNKNOWN result coverage no será almacenado como complete result.

## DB-RCACHE-074
Result size estará sujeto a governance.

## DB-RCACHE-075
Oversized entry podrá rechazarse.

## DB-RCACHE-076
Cache admission será policy-driven.

## DB-RCACHE-077
Cheap queries podrán bypass cache por costo.

## DB-RCACHE-078
Cache benefit deberá considerar lookup/serialization/invalidation overhead.

## DB-RCACHE-079
L1 será bounded.

## DB-RCACHE-080
L2 será opcional.

## DB-RCACHE-081
Tiered cache deberá preservar invalidation semantics.

## DB-RCACHE-082
L1 stale state deberá mitigarse.

## DB-RCACHE-083
Pub/sub perdido no será única garantía fuerte.

## DB-RCACHE-084
TTL podrá actuar como bounded safety net.

## DB-RCACHE-085
Single-flight será bounded.

## DB-RCACHE-086
Distributed locks tendrán lease/expiration.

## DB-RCACHE-087
Producer crash no dejará lock infinito.

## DB-RCACHE-088
TTL jitter podrá reducir stampedes.

## DB-RCACHE-089
ResultCacheEntry incluirá provenance suficiente.

## DB-RCACHE-090
Provenance no expondrá credentials.

## DB-RCACHE-091
Cache payload tendrá version.

## DB-RCACHE-092
Unsafe deserialization estará prohibida.

## DB-RCACHE-093
Structural metadata generation podrá invalidar payload compatibility.

## DB-RCACHE-094
Data dependencies serán distintas de structural dependencies.

## DB-RCACHE-095
Dependency generations podrán utilizarse para logical invalidation.

## DB-RCACHE-096
Old generations podrán permanecer físicamente hasta eviction.

## DB-RCACHE-097
Bulk mutations podrán requerir coarse invalidation.

## DB-RCACHE-098
Raw mutations podrán requerir explicit/broad invalidation.

## DB-RCACHE-099
External mutations no serán asumidas detectables.

## DB-RCACHE-100
TTL/CDC/hooks podrán cubrir external mutations según policy.

## DB-RCACHE-101
Result Cache consistency model será documentable.

## DB-RCACHE-102
Operation cache será distinto de process cache.

## DB-RCACHE-103
Process cache será distinto de distributed cache.

## DB-RCACHE-104
Cross-application namespace collision estará prohibido.

## DB-RCACHE-105
Deployment generations podrán separar incompatible payloads.

## DB-RCACHE-106
Persistent workers no conservarán request context dentro de entries.

## DB-RCACHE-107
Persistent workers no conservarán authenticated user objects dentro de entries.

## DB-RCACHE-108
OpenSwoole L1 será concurrency-safe.

## DB-RCACHE-109
Worker-local cache no será considerado global truth.

## DB-RCACHE-110
Ordinary cache backend failure será FAIL_OPEN.

## DB-RCACHE-111
FAIL_OPEN ejecutará DB query.

## DB-RCACHE-112
FAIL_CLOSED requerirá política explícita.

## DB-RCACHE-113
Corrupt entries no serán hidratados parcialmente.

## DB-RCACHE-114
Corrupt entries producirán eviction/miss.

## DB-RCACHE-115
Compression será policy-driven.

## DB-RCACHE-116
Sensitive data podrá prohibirse en distributed cache.

## DB-RCACHE-117
Query projection deberá minimizar datos innecesarios.

## DB-RCACHE-118
Authorization scope no será inferido solo desde SQL text.

## DB-RCACHE-119
High-cardinality visibility contexts podrán hacer caching no rentable.

## DB-RCACHE-120
Locale no participará si no altera DB result semantics.

## DB-RCACHE-121
Presentation formatting no contaminará DB Result Cache.

## DB-RCACHE-122
Diagnostics explicarán hit/miss/bypass.

## DB-RCACHE-123
Telemetry no registrará raw payloads por default.

## DB-RCACHE-124
Metric labels serán bounded.

## DB-RCACHE-125
User identifiers no serán metric labels.

## DB-RCACHE-126
ResultCacheManager no compilará SQL.

## DB-RCACHE-127
ResultCacheManager no ejecutará DB directamente.

## DB-RCACHE-128
ResultCacheManager no realizará authorization decisions.

## DB-RCACHE-129
Cacheability deberá resolverse antes del lookup.

## DB-RCACHE-130
Locking intent deberá conocerse antes del lookup.

## DB-RCACHE-131
Transaction compatibility deberá conocerse antes del shared lookup.

## DB-RCACHE-132
Data domain deberá resolverse antes de construir la key.

## DB-RCACHE-133
Cache key existence por sí sola no constituye hit.

## DB-RCACHE-134
Hit requerirá freshness compatible.

## DB-RCACHE-135
Hit requerirá dependency compatibility cuando la policy lo exija.

## DB-RCACHE-136
Hit requerirá visibility compatibility.

## DB-RCACHE-137
Hit requerirá consistency compatibility.

## DB-RCACHE-138
Result Cache nunca modificará Query Model.

## DB-RCACHE-139
Result Cache nunca modificará TransactionContext.

## DB-RCACHE-140
Result Cache nunca modificará EntityState directamente.

## DB-RCACHE-141
Hydration seguirá siendo responsable de construir entities.

## DB-RCACHE-142
IdentityMap seguirá siendo responsable de canonical entity identity.

## DB-RCACHE-143
UnitOfWork seguirá siendo responsable del tracking de managed entities.

## DB-RCACHE-144
Cache invalidation no será responsabilidad de Entity objects.

## DB-RCACHE-145
Persistence Engine producirá semantic mutation information donde corresponda.

## DB-RCACHE-146
Cache invalidation no convertirá rollback en commit semantics.

## DB-RCACHE-147
Result cache refresh no ejecutará una locking query como ordinary refresh.

## DB-RCACHE-148
Refresh failures no destruirán fresh entries innecesariamente.

## DB-RCACHE-149
Stale entries nunca se servirán fuera de su stale policy window.

## DB-RCACHE-150
Result Cache será una optimización removible sin cambiar la semántica contractual de la consulta.

---

# 279. Modelo de seguridad

La pregunta principal antes de reutilizar un resultado será:

```text
Can this consumer legitimately observe exactly this cached data
under the current database visibility contract?
```

Si la respuesta es:

```text
UNKNOWN
```

la decisión será:

```text
BYPASS
```

---

# 280. Modelo de consistencia

VoltStack distinguirá al menos:

```text
Cache Freshness
Database Visibility
Replica Freshness
Transaction Visibility
Dependency Validity
Application Consistency Requirement
```

No se reducirán a una sola propiedad:

```text
cache is fresh
```

---

# 281. Fórmula de reutilización

Para un entry `E` y contexto `C`:

```text
Reusable(E, C)
=
QueryCompatible(E, C)
∧ ParametersCompatible(E, C)
∧ ShapeCompatible(E, C)
∧ VisibilityCompatible(E, C)
∧ DataDomainCompatible(E, C)
∧ FreshnessAccepted(E, C)
∧ DependenciesValid(E, C)
∧ ConsistencySatisfied(E, C)
∧ PolicyAllows(E, C)
```

Solo entonces:

```text
ResultCacheDecision = HIT
```

---

# 282. Modelo de invalidación

```text
Database Mutation
        ↓
Mutation Summary
        ↓
Dependency Analysis
        ↓
Transaction Outcome
        ↓
Invalidation Plan
        ↓
┌───────────────┬────────────────┐
│               │                │
Generation    Key/Tag         Event/Outbox
Advance       Eviction        Publication
│               │                │
└───────────────┴────────────────┘
                ↓
        Result Cache State
```

---

# 283. Modelo de lectura completo

```text
Query
  ↓
Query Processing
  ↓
Execution Request
  ↓
Result Cache Policy
  ↓
Cacheability
  ↓
Transaction Check
  ↓
Visibility Context
  ↓
Data Domain
  ↓
Result Cache Key
  ↓
Lookup
  │
  ├── HIT
  │    ↓
  │ Validation
  │    ↓
  │ Canonical Result Payload
  │
  ├── STALE
  │    ↓
  │ Policy Decision
  │
  └── MISS/BYPASS
       ↓
    Database
       ↓
 Canonical Result
       ↓
 Admission Policy
       ↓
 Result Cache
       ↓
       └───────────────┐
                       ▼
                  Result System
                       ↓
                    Hydration
                       ↓
               Entity / DTO / Scalar
```

---

# 284. Relación con los siguientes documentos

El bloque deberá permanecer separado:

```text
186 Cache Architecture
        │
        ├── 187 Query Cache
        │       └── query-processing artifacts
        │
        ├── 188 Result Cache
        │       └── executed query results
        │
        ├── 189 Metadata Cache
        │       └── structural metadata
        │
        ├── 190 Entity Cache
        │       └── entity-oriented cached state
        │
        ├── 191 Cache Invalidation
        │       └── cross-cache invalidation
        │
        └── 192 Cache Consistency
                └── global consistency semantics
```

---

# 285. Regla maestra final

> **VoltStack nunca deberá utilizar la existencia de un resultado cacheado como evidencia suficiente para devolverlo. Un Result Cache hit será una decisión semántica que demuestre compatibilidad entre la consulta, sus parámetros, la forma del resultado, el dominio de datos, la visibilidad, las dependencias, la frescura y la política de consistencia requerida por la operación actual.**

Formalmente:

```text
ResultCacheHit
=
Identity
+
Visibility
+
Freshness
+
DependencyValidity
+
Consistency
+
Policy
```

y:

```text
CachedDataExists
≠
CachedDataIsSafeToServe
```

Además:

```text
Query Cache Hit
→ avoids query-processing work

Result Cache Hit
→ may avoid database execution

Entity Cache Hit
→ may avoid entity-oriented database retrieval

IdentityMap Hit
→ preserves object identity inside ORM scope
```

Estas cuatro optimizaciones deberán permanecer arquitectónicamente separadas.

---

# 286. Siguiente documento

```text
189_DATABASE_METADATA_CACHE_SYSTEM.md
```

El siguiente documento definirá el almacenamiento y reutilización de metadata estructural y compilada de Database:

```text
Schema Metadata
Entity Metadata
Mapping Metadata
Relationship Metadata
Type Metadata
Platform Metadata
Compiled Metadata
Metadata Generations
Metadata Fingerprints
Warmup
Persistent Workers
Reflection Elimination
Invalidation
Deployment Compatibility
Extension Metadata
Memory Governance
Telemetry
Diagnostics
```

con la regla:

```text
Metadata Cache
→ caches knowledge about database/application structure

Query Cache
→ caches work performed on queries

Result Cache
→ caches data returned by queries
```