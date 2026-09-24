# 154_DATABASE_N_PLUS_ONE_DETECTION_SYSTEM.md

# VoltStack Quantum Database
## Database N+1 Detection System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 154 — Database N+1 Detection System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database N+1 Detection System` define la infraestructura encargada de detectar, correlacionar, clasificar, explicar y reportar patrones de acceso a datos donde una operación inicial provoca múltiples cargas repetitivas que podrían haberse resuelto mediante una estrategia más eficiente.

Ejemplo clásico:

```php
$users = User::query()->get();

foreach ($users as $user) {
    echo $user->posts->count();
}
```

Patrón resultante:

```text
Query 1:
    SELECT users

Query 2:
    load User#1.posts

Query 3:
    load User#2.posts

Query 4:
    load User#3.posts

...

Query N+1:
    load User#N.posts
```

El sistema no se limitará a comparar strings SQL.

VoltStack deberá entender que las operaciones:

```text
User#1.posts
User#2.posts
User#3.posts
...
```

representan instancias repetidas de la misma intención semántica:

```text
Relationship:
    User.posts
```

dentro de un contexto común.

---

# 2. Regla central

> **VoltStack detectará N+1 mediante correlación semántica entre una operación raíz y cargas repetidas equivalentes, no simplemente contando strings SQL idénticos.**

Formalmente:

```text
NPlusOneDetection
=
SemanticObservation
+
RelationshipCorrelation
+
RootOperationCorrelation
+
RepeatedLoadAnalysis
+
ConfidenceEvaluation
+
DiagnosticReporting
```

No:

```text
NPlusOneDetection
=
CountIdenticalSqlStrings()
```

---

# 3. Objetivos

El sistema deberá:

1. detectar N+1 de relaciones;
2. detectar lazy-loading N+1;
3. detectar nested N+1;
4. detectar patrones polimórficos;
5. detectar consultas repetidas equivalentes;
6. correlacionarlas con una operación raíz;
7. distinguir queries legítimamente repetidas;
8. minimizar falsos positivos;
9. producir niveles de confianza;
10. identificar el relationship involucrado;
11. identificar el call site cuando sea posible;
12. recomendar eager/batch loading;
13. integrarse con Telemetry;
14. integrarse con Profiler;
15. integrarse con Debug Toolbar;
16. funcionar en CLI/jobs;
17. ser seguro en workers persistentes;
18. tener overhead acotado;
19. soportar sampling;
20. proporcionar assertions para testing.

---

# 4. Qué es N+1

Conceptualmente:

```text
1 Root Operation
+
N Similar Dependent Operations
```

donde las operaciones dependientes existen debido a elementos obtenidos por la operación raíz.

Formalmente:

```text
RootOperation R
produces
E = {e1, e2, ..., en}

and

∀ ei ∈ E:
    dependentLoad(ei, relationship X)
```

produce un candidato:

```text
N+1(R, X)
```

si dichas cargas podrían haberse agrupado o anticipado razonablemente.

---

# 5. N+1 ≠ muchas queries

Ejecutar 100 queries no implica automáticamente N+1.

Ejemplo:

```text
100 unrelated commands
```

no constituye necesariamente N+1.

Se requiere correlación.

---

# 6. N+1 ≠ SQL duplicado

Estas queries:

```sql
SELECT * FROM posts WHERE user_id = 1;
SELECT * FROM posts WHERE user_id = 2;
SELECT * FROM posts WHERE user_id = 3;
```

no son strings idénticos.

Semánticamente, sin embargo:

```text
RelationshipLoad(User.posts, owner=?)
```

es el mismo patrón.

---

# 7. SQL duplicado ≠ N+1

También puede existir:

```text
same query
same parameters
repeated 20 times
```

sin relación raíz-entidad.

Eso puede ser:

```text
Duplicate Query
```

pero no necesariamente:

```text
N+1
```

VoltStack distinguirá ambos problemas.

---

# 8. N+1 ≠ Lazy Loading

Lazy Loading es una estrategia válida.

```text
Lazy Loading
≠
N+1
```

El problema aparece cuando:

```text
Repeated Lazy Loads
+
Common Relationship
+
Common Parent Operation
```

generan un patrón evitable.

---

# 9. N+1 ≠ Eager Loading Failure

La ausencia de eager loading no implica automáticamente un error.

Puede ser correcto cargar una relación una sola vez.

---

# 10. N+1 ≠ Batch Loading Failure

Batch loading es una solución potencial.

Pero existen situaciones donde los loads no son batch-compatible.

Por tanto:

```text
Repeated Loads
≠
Automatically Fixable N+1
```

---

# 11. Posición arquitectónica

```text
ORM
│
├── Entity Query
├── Relationship System
│   ├── Relationship Loading
│   ├── Eager Loading
│   ├── Lazy Loading
│   ├── Batch Loading
│   └── N+1 Detection
│
├── IdentityMap
└── UnitOfWork

Cross-Cutting
│
├── Telemetry
├── Profiler
├── Debug Toolbar
└── Testing
```

---

# 12. Arquitectura general

```text
Database Operations
       │
       ▼
Observation Hooks
       │
       ▼
Semantic Event Normalization
       │
       ▼
Request/Operation N+1 Context
       │
       ▼
Correlation Engine
       │
       ├── Root Operation Correlation
       ├── Relationship Correlation
       ├── Query Fingerprinting
       ├── Call-Site Correlation
       └── Temporal Correlation
       │
       ▼
Candidate Detector
       │
       ▼
Confidence Evaluator
       │
       ▼
False Positive Filters
       │
       ▼
N+1 Finding
       │
       ├── Telemetry
       ├── Logs
       ├── Profiler
       ├── Debug Toolbar
       └── Test Assertions
```

---

# 13. Semantic observation

La detección deberá ocurrir lo más arriba posible en la arquitectura.

Preferible:

```text
RelationshipLoadRequested
```

sobre:

```text
SQL string executed
```

porque la primera contiene contexto ORM.

---

# 14. Observation sources

El detector podrá consumir eventos de:

```text
Entity Query
Relationship Loading
Lazy Loading
Batch Loading
Query Execution
Hydration
IdentityMap
```

pero no todos serán obligatorios para cada finding.

---

# 15. RelationshipLoadObserved

Evento conceptual:

```php
final readonly class RelationshipLoadObserved
{
    public function __construct(
        public OperationId $operation,
        public RelationshipId $relationship,
        public EntityKey $owner,
        public RelationshipLoadMode $mode,
        public RelationshipLoadShape $shape,
        public LoadOutcome $outcome,
        public ?CallSiteFingerprint $callSite,
    ) {}
}
```

---

# 16. Entity IDs y telemetry

El detector puede necesitar `EntityKey` internamente para correlación.

Pero:

> Entity IDs no deberán convertirse automáticamente en labels de métricas, logs públicos o traces de alta cardinalidad.

---

# 17. Root Operation

Una operación raíz puede ser:

```text
Entity Query
Repository Query
Model Query
Controller-triggered DB operation
Job operation
Command operation
Explicit ORM load graph
```

---

# 18. RootOperationId

Cada operación relevante deberá poder asociarse a:

```text
RootOperationId
```

scope-local.

---

# 19. Root ≠ HTTP request

Un request puede contener varias operaciones raíz.

```text
HTTP Request
├── Root Query A
├── Root Query B
└── Root Query C
```

---

# 20. Request-level fallback

Si no puede determinarse un root preciso, el detector puede correlacionar a nivel:

```text
request/job/command
```

pero con menor confianza.

---

# 21. CorrelationContext

```php
final readonly class NPlusOneCorrelationContext
{
    public function __construct(
        public OperationScopeId $scope,
        public ?RootOperationId $root,
        public PersistenceContextId $persistence,
        public DatabaseContextKey $database,
        public TenantContextKey $tenant,
        public ShardContextKey $shard,
    ) {}
}
```

---

# 22. Semantic fingerprint

La pieza central será:

```text
SemanticLoadFingerprint
```

---

# 23. Relationship fingerprint

Ejemplo:

```text
Relationship:
    User.posts

Mode:
    LAZY

Shape:
    COMPLETE

Scope:
    default
```

puede producir:

```text
RelationshipLoadFingerprint
```

---

# 24. Query fingerprint

Para operaciones sin metadata ORM suficiente:

```text
SELECT ...
WHERE user_id = ?
```

puede normalizarse a un fingerprint estructural.

---

# 25. Query fingerprint ≠ SQL hash

No:

```text
sha256(raw SQL)
```

como única estrategia.

Preferir:

```text
Query AST Shape
+
Semantic Metadata
+
Parameter Shape
```

---

# 26. Parameter values

Valores concretos:

```text
user_id = 1
user_id = 2
```

no deben producir fingerprints diferentes cuando representan el mismo patrón.

---

# 27. Parameter shape

Sí puede importar:

```text
scalar identifier
composite identifier
list
range
```

---

# 28. RelationshipLoadFingerprint

```php
final readonly class RelationshipLoadFingerprint
{
    public function __construct(
        public RelationshipId $relationship,
        public RelationshipLoadMode $mode,
        public LoadShapeFingerprint $shape,
        public ScopeFingerprint $scope,
    ) {}
}
```

---

# 29. QueryFingerprint

```php
final readonly class QueryFingerprint
{
    public function __construct(
        public QueryKind $kind,
        public SemanticQueryFingerprint $semantic,
        public ParameterShapeFingerprint $parameters,
    ) {}
}
```

---

# 30. Call-site fingerprint

Puede ser útil saber que los loads vienen de:

```text
UserController.php:84
```

o:

```text
UserResource::toArray()
```

---

# 31. Call site ≠ full stack trace

Capturar stack traces completos para cada query puede ser costoso.

VoltStack deberá soportar estrategias de costo gradual.

---

# 32. CallSiteCaptureMode

```php
enum CallSiteCaptureMode
{
    case NONE;
    case FIRST_ONLY;
    case SAMPLED;
    case ALWAYS;
}
```

---

# 33. Producción

Recomendación:

```text
NONE
```

o:

```text
SAMPLED
```

---

# 34. Desarrollo

Puede utilizar:

```text
FIRST_ONLY
```

por fingerprint.

---

# 35. Testing

Puede habilitarse:

```text
ALWAYS
```

cuando se necesite diagnóstico exacto.

---

# 36. Stack normalization

Frames internos de VoltStack deberán poder excluirse.

Ejemplo:

```text
VoltStack\ORM\...
VoltStack\Database\...
```

para encontrar el primer frame de aplicación.

---

# 37. CallSiteFingerprint

```text
ApplicationFrame
+
Function/Method
+
NormalizedSourceLocation
```

---

# 38. Source location stability

El número de línea puede cambiar frecuentemente.

Puede conservarse para diagnóstico, pero no necesariamente como parte completa del fingerprint estable.

---

# 39. Temporal correlation

Las cargas N+1 normalmente ocurren cerca temporalmente.

Ejemplo:

```text
Root Query
↓
Load #1
Load #2
Load #3
...
```

---

# 40. Temporal proximity ≠ proof

No deberá ser el único criterio.

---

# 41. Owner correlation

El detector puede observar:

```text
same relationship
different owners
```

como señal fuerte.

---

# 42. Same owner repeated

```text
User#1.posts
User#1.posts
User#1.posts
```

es más probablemente:

```text
duplicate load
```

o problema de caching/state,

no el N+1 clásico.

---

# 43. Distinct owner count

Métrica central:

```text
DistinctOwnersPerRelationshipFingerprint
```

---

# 44. Candidate formula

Un candidato fuerte puede definirse como:

```text
RepeatedLoadCount >= Threshold
AND
DistinctOwnerCount >= Threshold
AND
SameRelationshipFingerprint
AND
SameCorrelationScope
```

---

# 45. N+1 threshold

No todo:

```text
1 root + 2 relation queries
```

merece warning.

Debe existir política.

---

# 46. ThresholdPolicy

```php
final readonly class NPlusOneThresholdPolicy
{
    public function __construct(
        public int $minimumRepeatedLoads,
        public int $minimumDistinctOwners,
        public int $warningThreshold,
        public int $criticalThreshold,
    ) {}
}
```

---

# 47. Default conceptual

Por ejemplo:

```text
minimumRepeatedLoads = 3
minimumDistinctOwners = 3
```

Los valores finales serán configurables.

---

# 48. Severity

```php
enum NPlusOneSeverity
{
    case INFO;
    case WARNING;
    case HIGH;
    case CRITICAL;
}
```

---

# 49. Severity ≠ confidence

Un patrón puede ser:

```text
high confidence
low impact
```

o:

```text
medium confidence
very high impact
```

---

# 50. Confidence

```php
enum NPlusOneConfidence
{
    case LOW;
    case MEDIUM;
    case HIGH;
    case CERTAIN;
}
```

---

# 51. Confidence signals

Positivas:

- mismo RelationshipId;
- mismo root;
- múltiples owners;
- lazy loading;
- misma shape;
- misma call site;
- batch-compatible owners;
- loads secuenciales.

---

# 52. Negative signals

Reducen confianza:

- distintos scopes;
- distintos tenants;
- distintos shards;
- diferentes transacciones;
- diferentes call sites;
- cargas explícitas independientes;
- gran separación temporal;
- owners no batch-compatible.

---

# 53. Confidence scoring

Conceptualmente:

```text
ConfidenceScore
=
PositiveEvidence
-
AmbiguityPenalties
```

---

# 54. Score ≠ public contract

La implementación interna puede cambiar.

La API pública debería exponer categorías estables.

---

# 55. NPlusOneCandidate

```php
final readonly class NPlusOneCandidate
{
    public function __construct(
        public SemanticLoadFingerprint $fingerprint,
        public int $loadCount,
        public int $distinctOwners,
        public CorrelationEvidence $evidence,
    ) {}
}
```

---

# 56. Finding

Un candidate se convierte en finding después de:

```text
Threshold
+
Confidence
+
False Positive Filters
+
Policy
```

---

# 57. NPlusOneFinding

```php
final readonly class NPlusOneFinding
{
    public function __construct(
        public NPlusOneFindingId $id,
        public NPlusOnePatternKind $kind,
        public NPlusOneSeverity $severity,
        public NPlusOneConfidence $confidence,
        public int $loadCount,
        public int $distinctOwnerCount,
        public SemanticLoadFingerprint $fingerprint,
        public NPlusOneRecommendation $recommendation,
    ) {}
}
```

---

# 58. Pattern kinds

```php
enum NPlusOnePatternKind
{
    case RELATIONSHIP;
    case LAZY_RELATIONSHIP;
    case NESTED_RELATIONSHIP;
    case POLYMORPHIC_RELATIONSHIP;
    case EXTRA_LAZY_OPERATION;
    case DUPLICATE_QUERY;
    case REPEATED_ENTITY_LOOKUP;
}
```

---

# 59. RELATIONSHIP

Patrón genérico:

```text
N owners
→ same relationship
→ N loads
```

---

# 60. LAZY_RELATIONSHIP

El trigger proviene explícitamente de Lazy Loading.

---

# 61. NESTED_RELATIONSHIP

Ejemplo:

```php
foreach ($users as $user) {
    foreach ($user->posts as $post) {
        echo $post->comments->count();
    }
}
```

Puede producir:

```text
Users
  ↓
User.posts N+1
  ↓
Post.comments nested N+1
```

---

# 62. Nested correlation graph

```text
Root User Query
│
├── User.posts
│   ├── Post.comments
│   ├── Post.comments
│   └── Post.comments
│
└── User.posts
```

---

# 63. N+1 graph

El detector podrá representar findings como grafo.

```text
RootOperation
    │
    ▼
RelationshipFingerprint A
    │
    ▼
RelationshipFingerprint B
```

---

# 64. Nested depth

Debe registrarse:

```text
NPlusOneDepth
```

sin permitir grafos infinitos.

---

# 65. POLYMORPHIC_RELATIONSHIP

Ejemplo:

```text
Comment#1 → Post#10
Comment#2 → Video#7
Comment#3 → Post#20
...
```

---

# 66. Polymorphic N+1

No deberá clasificarse simplemente por tabla.

Debe considerar:

```text
Polymorphic RelationshipId
+
Resolved Target Type
+
Load Context
```

---

# 67. Multiple target types

Puede ser legítimo necesitar:

```text
1 query per target type
```

con batch loading.

Eso no es necesariamente N+1.

---

# 68. Polymorphic expected query count

Aproximadamente:

```text
ExpectedQueries
≈
DistinctCompatibleTargetGroups
```

no:

```text
NumberOfOwners
```

---

# 69. EXTRA_LAZY_OPERATION

Ejemplo:

```php
foreach ($users as $user) {
    echo $user->posts->count();
}
```

si `count()` genera una query por owner.

---

# 70. Extra-lazy operations

Pueden incluir:

```text
count
contains
slice
exists
```

---

# 71. Extra-lazy N+1 fingerprint

Debe incluir operation kind.

```text
User.posts
+
COUNT
```

---

# 72. Duplicate query detection

Ejemplo:

```text
SELECT User#10
SELECT User#10
SELECT User#10
```

---

# 73. DuplicateQueryFinding

Debe ser una categoría relacionada pero distinta.

---

# 74. Repeated entity lookup

Si IdentityMap debió evitar queries repetidas para la misma entidad, puede indicar:

- detached context;
- incorrect EntityManager usage;
- bypass ORM;
- missing identity reconciliation.

---

# 75. Query-level fallback

No todas las queries provienen del ORM.

VoltStack podrá detectar patrones estructurales en Query Engine.

---

# 76. Lower confidence

Un finding basado solo en query fingerprints tendrá normalmente menor confianza que uno basado en `RelationshipId`.

---

# 77. False positives

El detector deberá minimizar falsos positivos activamente.

---

# 78. Legitimate sequential loads

Ejemplo:

```text
different tenants
different DBs
different shards
```

pueden requerir queries separadas.

---

# 79. Batch compatibility integration

El detector podrá consultar:

```text
BatchCompatibilityAnalyzer
```

del documento 153.

---

# 80. Strong recommendation

Si:

```text
100 loads
+
100 batch-compatible owners
```

la recomendación es fuerte.

---

# 81. Weak recommendation

Si:

```text
100 loads
+
100 incompatible tenant contexts
```

no deberá sugerir simplemente:

```text
"Use eager loading"
```

como si una query única fuera posible.

---

# 82. Fixability

```php
enum NPlusOneFixability
{
    case DIRECT;
    case PARTIAL;
    case CONTEXT_DEPENDENT;
    case NOT_BATCHABLE;
    case UNKNOWN;
}
```

---

# 83. Fixability ≠ severity

Un problema severo puede ser difícil de agrupar.

---

# 84. Recommendation engine

El detector podrá generar recomendaciones semánticas.

---

# 85. Recommendation types

```php
enum NPlusOneRecommendationKind
{
    case USE_EAGER_LOADING;
    case USE_BATCH_LOADING;
    case PRELOAD_RELATION;
    case USE_WITH_COUNT;
    case USE_AGGREGATE_QUERY;
    case USE_IDENTITY_MAP_PATH;
    case REVIEW_EXTRA_LAZY_LOOP;
    case REVIEW_QUERY_STRUCTURE;
    case NONE;
}
```

---

# 86. Eager loading recommendation

Ejemplo:

```php
User::query()
    ->with('posts')
    ->get();
```

solo si es compatible con el caso detectado.

---

# 87. Batch loading recommendation

Puede recomendar:

```text
Batch-load User.posts for 100 compatible owners.
```

---

# 88. withCount-style optimization

Caso:

```php
foreach ($users as $user) {
    $user->posts->count();
}
```

podría recomendar una agregación.

Conceptualmente:

```php
User::query()
    ->withCount('posts')
    ->get();
```

si VoltStack expone esa API.

---

# 89. Recommendation ≠ automatic rewrite

El detector no modificará queries automáticamente.

---

# 90. Detection ≠ Optimization

Invariante:

```text
N+1 Detector
≠
Query Rewriter
```

---

# 91. Development mode

En desarrollo se podrá habilitar:

```text
detailed detection
+
call-site capture
+
immediate warning
```

---

# 92. Production mode

En producción:

```text
sampling
+
aggregated telemetry
+
bounded state
```

---

# 93. Strict testing mode

En testing:

```text
detect
+
collect
+
assert
```

---

# 94. Detection modes

```php
enum NPlusOneDetectionMode
{
    case OFF;
    case LIGHTWEIGHT;
    case STANDARD;
    case DETAILED;
    case TESTING;
}
```

---

# 95. OFF

Cero tracking adicional específico del detector.

---

# 96. LIGHTWEIGHT

Solo contadores/fingerprints básicos.

---

# 97. STANDARD

Relationship correlation + thresholds.

---

# 98. DETAILED

Añade:

- call sites;
- root graphs;
- richer diagnostics.

---

# 99. TESTING

Optimizado para assertions deterministas.

---

# 100. Runtime overhead

El detector deberá tener overhead predecible.

---

# 101. Complexity target

Para cada observed load, el tracking debería aproximarse a:

```text
O(1)
```

promedio mediante maps por fingerprint.

---

# 102. No global query history

No almacenar todas las queries indefinidamente.

---

# 103. Aggregation

Preferir:

```text
Fingerprint
→ Counter
→ DistinctOwnerTracker
→ FirstCallSite
→ Evidence
```

---

# 104. Bounded distinct-owner tracking

Para evitar memoria excesiva puede existir:

```text
ExactOwnerSet
```

hasta un límite.

Después:

```text
ApproximateDistinctCounter
```

si se implementa una estrategia segura.

---

# 105. Exactness in testing

Testing podrá exigir tracking exacto.

---

# 106. Sampling

Producción puede observar solo una fracción de scopes.

---

# 107. Scope sampling

Preferible:

```text
sample entire request/job
```

a:

```text
randomly sample individual relationship loads
```

porque conserva correlación.

---

# 108. Sampling decision

Debe tomarse una vez por scope.

---

# 109. Sampling determinism

Puede basarse en:

```text
TraceId
RequestId
OperationScopeId
```

mediante hash.

---

# 110. Sensitive data

No utilizar:

- email;
- user ID externo;
- raw tenant credentials;

para sampling.

---

# 111. Detection policy

```php
final readonly class NPlusOneDetectionPolicy
{
    public function __construct(
        public NPlusOneDetectionMode $mode,
        public NPlusOneThresholdPolicy $thresholds,
        public CallSiteCaptureMode $callSites,
        public float $sampleRate,
        public int $maxFingerprints,
        public int $maxTrackedOwnersPerFingerprint,
    ) {}
}
```

---

# 112. Resource governance

El detector deberá limitar:

```text
max fingerprints
max findings
max call sites
max graph depth
max tracked owners
max stack captures
```

---

# 113. Detector overload

Si los límites se exceden:

```text
DETECTION_TRUNCATED
```

deberá registrarse.

---

# 114. Truncation ≠ no N+1

El diagnóstico deberá indicar que el análisis fue parcial.

---

# 115. Request scope

Estado mutable:

```text
NPlusOneDetectionContext
```

será request/job/command scoped.

---

# 116. Persistent runtime

Nunca:

```php
private static array $findings;
```

---

# 117. FrankenPHP

```text
Worker
├── immutable detector config
├── immutable fingerprint logic
│
├── Request A
│   └── DetectionContext A
│
└── Request B
    └── DetectionContext B
```

---

# 118. RoadRunner

Mismo modelo de aislamiento.

---

# 119. OpenSwoole

Contexto deberá ser:

```text
coroutine-local
```

cuando exista concurrencia.

---

# 120. Context reset

Al finalizar scope:

```text
finalize findings
emit telemetry
clear context
```

---

# 121. Finalization

No deberá ejecutar nuevas queries.

---

# 122. Detector observation only

El detector será esencialmente read-only respecto al ORM.

---

# 123. No mutation

No deberá:

- inicializar relaciones;
- modificar EntityManager;
- modificar UoW;
- ejecutar eager loads;
- alterar query plan.

---

# 124. Event model

Eventos conceptuales:

```text
RootDatabaseOperationStarted
RootDatabaseOperationCompleted

RelationshipLoadRequested
RelationshipLoadCompleted

LazyRelationshipLoadTriggered
BatchRelationshipLoadDispatched

QueryExecuted

NPlusOneCandidateDetected
NPlusOneFindingConfirmed
```

---

# 125. Internal event path

Para hot path se podrá utilizar integración directa de bajo overhead en lugar de EventSystem general cuando sea necesario.

---

# 126. Architectural events ≠ domain events

No deben confundirse.

---

# 127. Batch loading awareness

Si 100 owners son cargados mediante:

```text
1 batch
```

el detector deberá saber que no existe N+1.

---

# 128. Batch operation observation

```text
BatchRelationshipLoadDispatched
owners = 100
queries = 1
```

puede alimentar métricas positivas.

---

# 129. Prevented N+1

VoltStack podrá medir:

```text
potential_individual_loads = 100
actual_batch_queries = 1
```

---

# 130. PreventedNPlusOneMetric

Debe ser telemetry informativa, no finding.

---

# 131. Eager loading awareness

Si:

```text
User + posts
```

se resolvió mediante eager strategy, no reportar N+1.

---

# 132. JOIN explosion ≠ N+1

Una eager query enorme con multiplicación de rows es otro problema de performance.

No clasificarlo como N+1.

---

# 133. IdentityMap awareness

Si el relationship load se satisface desde memoria:

```text
query count = 0
```

no es N+1 de base de datos.

---

# 134. Repeated lazy trigger without query

Puede ser un problema de aplicación, pero no necesariamente DB N+1.

---

# 135. Database operation count

El finding deberá distinguir:

```text
relationship access count
```

de:

```text
actual database query count
```

---

# 136. Example

```text
Relationship accesses: 100
IdentityMap hits: 80
DB loads: 20
```

El impacto real es 20 DB operations.

---

# 137. N+1 impact

Puede estimarse usando:

```text
DBQueryCount
Latency
Rows
Bytes
```

---

# 138. Impact score

Conceptualmente:

```text
Impact
=
QueryCountWeight
+
LatencyWeight
+
ResourceWeight
```

---

# 139. Impact score ≠ confidence

Mantener separados.

---

# 140. Query latency

100 queries de 0.1 ms y 100 queries de 50 ms tienen impactos diferentes.

---

# 141. Cumulative latency

```text
CumulativeRelationshipQueryDuration
```

deberá poder reportarse.

---

# 142. Wall time ≠ cumulative time

Con concurrencia:

```text
sum(query durations)
```

puede superar wall-clock duration.

Reportar claramente.

---

# 143. Query count reduction estimate

Si todos los owners son batch-compatible:

```text
PotentialQueriesAfterBatch
≈
ceil(
    DistinctOwners
    /
    EffectiveBatchCapacity
)
```

---

# 144. Potential saving

```text
PotentialQueriesSaved
=
ObservedQueries
-
EstimatedBatchQueries
```

---

# 145. Estimate labeling

Debe marcarse como:

```text
estimated
```

no como garantía.

---

# 146. NPlusOneFinding report

Ejemplo:

```text
N+1 DETECTED

Relationship:
    User.posts

Root:
    User entity query

Pattern:
    LAZY_RELATIONSHIP

Owners:
    100

Relationship accesses:
    100

Database loads:
    100

Estimated batch queries:
    1

Potential queries avoided:
    ~99

Confidence:
    HIGH

Severity:
    HIGH

Call site:
    UserResource::toArray()

Recommendation:
    Eager-load User.posts or batch-load the relation.
```

---

# 147. Nested report

```text
N+1 GRAPH

User Query
│
├── User.posts
│   Loads: 100
│
└── Post.comments
    Loads: 1,240
```

---

# 148. Polymorphic report

```text
POLYMORPHIC N+1

Relationship:
    Comment.commentable

Loads:
    500

Target types:
    Post: 300
    Video: 150
    Photo: 50

Observed DB queries:
    500

Estimated grouped queries:
    3

Confidence:
    HIGH
```

---

# 149. False-positive report

Si loads no son batch-compatible:

```text
Repeated relationship loading detected,
but owners span 12 shards.

Classification:
    REPEATED_LOAD

N+1 Fixability:
    CONTEXT_DEPENDENT
```

---

# 150. Logging

Un finding podrá generar log estructurado:

```text
database.n_plus_one.detected
```

---

# 151. Log deduplication

No emitir 100 warnings por el mismo fingerprint.

---

# 152. One finding per correlation group

Default:

```text
one finding
per
scope × root × fingerprint
```

---

# 153. Production aggregation

Puede agregarse posteriormente por:

```text
application route
job type
relationship
call site fingerprint
```

evitando cardinalidad excesiva.

---

# 154. Telemetry metrics

```text
database.n_plus_one.findings
database.n_plus_one.candidates
database.n_plus_one.queries
database.n_plus_one.distinct_owners
database.n_plus_one.cumulative_duration
database.n_plus_one.estimated_queries_avoidable
database.n_plus_one.lazy_findings
database.n_plus_one.nested_findings
database.n_plus_one.polymorphic_findings
database.n_plus_one.extra_lazy_findings
database.n_plus_one.duplicate_query_findings
database.n_plus_one.analysis_truncated
```

---

# 155. Metric labels

Permitidos:

```text
relationship kind
pattern kind
severity
confidence
runtime
database platform
```

Evitar:

```text
entity ID
raw SQL
tenant ID de alta cardinalidad
URL completa dinámica
```

---

# 156. Tracing

Span/event:

```text
db.orm.n_plus_one
```

atributos:

```text
pattern
relationship
query_count
owner_count
confidence
severity
fixability
```

---

# 157. Query traces

El finding podrá enlazarse conceptualmente con spans de queries mediante IDs internos.

---

# 158. No span mutation dependency

La detección deberá funcionar incluso si tracing está desactivado.

---

# 159. Profiler integration

El futuro:

```text
221_DATABASE_QUERY_PROFILER_SYSTEM.md
```

podrá mostrar findings.

---

# 160. Profiler panel

Conceptualmente:

```text
Database
├── Queries
├── Connections
├── Transactions
├── ORM
└── N+1
```

---

# 161. Debug Toolbar

El futuro documento:

```text
225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

podrá visualizar:

```text
⚠ 2 N+1 patterns
```

---

# 162. Toolbar details

Ejemplo:

```text
User.posts
100 queries
HIGH confidence

Post.comments
1,240 queries
CERTAIN confidence
```

---

# 163. No Debug Toolbar dependency

El detector no dependerá de la toolbar.

La toolbar consumirá findings.

---

# 164. Testing API

VoltStack deberá ofrecer assertions.

Ejemplo conceptual:

```php
$this->assertNoNPlusOne(function () {
    User::query()->with('posts')->get();
});
```

---

# 165. Alternative API

```php
Database::testing()
    ->detectNPlusOne()
    ->run(function () {
        // test
    })
    ->assertNone();
```

---

# 166. Assert specific relationship

```php
$this->assertNoNPlusOneFor(
    User::class,
    'posts',
    $callback,
);
```

---

# 167. Expect finding

También:

```php
$this->assertNPlusOneDetected(
    relationship: 'User.posts',
    callback: $callback,
);
```

útil para probar el propio detector.

---

# 168. Query count assertions

N+1 assertions no deberán reducirse a:

```php
assertQueryCount(2);
```

porque son conceptos distintos.

---

# 169. Deterministic testing

En TESTING mode:

- sampling OFF;
- exact counters;
- stable fingerprints;
- bounded but sufficiently high limits;
- deterministic call-site rules.

---

# 170. Baselines

Podrá existir posteriormente:

```text
N+1 baseline
```

para proyectos legacy.

---

# 171. Baseline use

Permitir temporalmente findings conocidos mientras se impide introducir nuevos.

---

# 172. Baseline ≠ suppression forever

Debe incluir:

- fingerprint;
- reason;
- expiration opcional.

---

# 173. Suppression

Puede existir una suppression explícita.

---

# 174. Suppression granularity

Por:

```text
RelationshipId
+
CallSite
+
Context
```

cuando sea posible.

---

# 175. No global ignore by default

Evitar:

```text
ignore all User.posts N+1
```

si solo un call site está justificado.

---

# 176. Suppression reason

Toda suppression debería poder documentar una razón.

---

# 177. Development exception mode

Opcionalmente:

```text
throw_on_n_plus_one = true
```

en testing/desarrollo.

---

# 178. Production

No lanzar excepciones por defecto.

---

# 179. NPlusOnePolicyAction

```php
enum NPlusOnePolicyAction
{
    case IGNORE;
    case RECORD;
    case LOG;
    case WARN;
    case THROW;
}
```

---

# 180. Severity policy

Ejemplo:

```text
INFO     → RECORD
WARNING  → LOG
HIGH     → WARN
CRITICAL → THROW in testing only
```

configurable.

---

# 181. Serialization-triggered N+1

Caso crítico:

```php
return $users;
```

y serializer accede automáticamente:

```text
user.posts
user.profile
...
```

---

# 182. Serializer correlation

El call-site puede apuntar al serializer/resource layer.

---

# 183. Serialization warning

Debe indicar que la relación fue lazy-loaded durante serialization.

---

# 184. Serialization should not hide N+1

Aunque el usuario no haya escrito un loop explícito.

---

# 185. Template-triggered N+1

Ejemplo:

```php
@foreach ($users as $user)
    {{ $user->posts->count() }}
@endforeach
```

mismo principio.

---

# 186. API resource N+1

También:

```text
Resource Transformer
→ relation access
```

---

# 187. GraphQL-like workloads

Si futuras integraciones producen field-level loaders:

```text
N+1 Detector
```

deberá seguir operando mediante semantic relationship events.

---

# 188. Resolver batching

Si existe batching correcto:

```text
no finding
```

aunque múltiples resolvers soliciten la relación.

---

# 189. CLI

Commands largos requieren scope explícito.

---

# 190. Jobs

Cada job debe tener:

```text
DetectionContext
```

independiente.

---

# 191. Long-running jobs

Puede requerirse segmentación interna:

```text
Job
├── Operation A
├── Operation B
└── Operation C
```

para evitar correlaciones falsas.

---

# 192. Manual operation boundary

API interna:

```php
$scope = $detector->beginOperation('import-users');
```

cuando sea necesario.

---

# 193. Operation name

No deberá formar parte de high-cardinality metric labels si es dinámica.

---

# 194. Raw Query Builder

Si el usuario usa:

```php
DB::table(...)
```

puede existir query-level duplicate detection.

Pero relationship-level N+1 puede no ser inferible.

---

# 195. Honest diagnostics

Debe decir:

```text
Repeated query pattern detected.
ORM relationship could not be determined.
```

No inventar una relación.

---

# 196. Raw SQL

Mismo principio.

---

# 197. Stored procedures

Pueden ocultar múltiples operaciones internas.

El detector solo reportará evidencia observable.

---

# 198. External DB calls

No inferir lo que no puede observar.

---

# 199. Detection confidence principle

```text
Unknown
≠
N+1
```

---

# 200. No fabricated certainty

Regla consistente con toda la arquitectura VoltStack.

---

# 201. Query fingerprint normalization

Debe ignorar cuando corresponda:

- parameter values;
- whitespace;
- generated aliases no semánticos.

---

# 202. Query fingerprint must preserve semantics

No debe ignorar:

- selected relation;
- join semantics;
- filters relevantes;
- scope;
- ordering cuando altera load shape;
- tenant/database context.

---

# 203. AST preferred

Si existe Query AST:

```text
fingerprint(AST)
```

es preferible a parsear SQL compilado.

---

# 204. Compiler independence

El detector no deberá depender del SQL exacto de MySQL/PostgreSQL.

---

# 205. Cross-platform consistency

El mismo patrón ORM deberá producir finding equivalente en:

- MySQL;
- MariaDB;
- PostgreSQL;
- SQLite.

---

# 206. Query fingerprint generation

Conceptualmente:

```text
Normalize
    ↓
Remove Runtime Parameter Values
    ↓
Preserve Semantic Structure
    ↓
Canonical Serialize
    ↓
Hash
```

---

# 207. Relationship fingerprint generation

```text
RelationshipId
+
Load Operation
+
Load Shape
+
Scope
+
Relevant Context
```

---

# 208. Fingerprint collision

Hash collision no debe producir corrupción.

Puede verificarse descriptor canónico cuando sea necesario.

---

# 209. Fingerprint storage

Internamente:

```text
hash
+
canonical descriptor
```

en modos detallados.

---

# 210. Query parameter privacy

No almacenar raw parameter values salvo debugging explícito y seguro.

---

# 211. Root query fingerprint

Puede permitir agrupar findings entre requests.

Pero no debe contener valores sensibles.

---

# 212. Cross-request aggregation

Pertenece principalmente a Telemetry/Profiler, no al detector scope-local.

---

# 213. Scope-local detector

El detector responde:

```text
Did N+1 occur in this operation?
```

---

# 214. Telemetry aggregation

Responde:

```text
How often does this N+1 happen across production?
```

---

# 215. Separation

```text
Detection
≠
Historical Analytics
```

---

# 216. Detection state

```php
final class NPlusOneDetectionContext
{
    /** @var array<string, FingerprintObservation> */
    private array $observations = [];

    /** @var list<NPlusOneFinding> */
    private array $findings = [];
}
```

Scope-local únicamente.

---

# 217. FingerprintObservation

Podrá mantener:

```text
load count
DB query count
distinct owner count
first occurrence
last occurrence
first call site
root operation
cumulative latency
batch compatibility evidence
```

---

# 218. Early candidate detection

No es necesario esperar al final del request para reconocer un candidato.

---

# 219. Finding finalization

Sin embargo, el finding final puede esperar suficiente evidencia.

---

# 220. Immediate warning

En desarrollo, cuando el threshold se cruza:

```text
emit once
```

---

# 221. Continued counting

Después del warning se continúa contando para el reporte final.

---

# 222. No warning spam

Nunca warning por cada query subsecuente.

---

# 223. Finding identity

```text
FindingKey
=
CorrelationScope
×
RootOperation
×
SemanticFingerprint
```

---

# 224. Nested finding identity

Incluye parent relationship path cuando sea relevante.

---

# 225. Call-site grouping

Mismo relationship desde call sites diferentes puede producir findings separados si ayuda al diagnóstico.

---

# 226. Policy configurable

Agrupación por call site puede depender del detection mode.

---

# 227. Database-side batching

Algunas plataformas/drivers pueden ejecutar optimizaciones internas.

El detector deberá basarse en operaciones realmente observadas, no asumir comportamiento oculto.

---

# 228. Query cache

Si 100 accesses producen 1 DB query y 99 result-cache hits:

```text
DB N+1 impact
```

es bajo o inexistente.

Pero puede existir:

```text
repeated logical load pattern
```

---

# 229. Cache masking

El detector puede reportar opcionalmente:

```text
N+1 pattern masked by cache
```

en modo detallado.

---

# 230. Cache dependence warning

Esto ayuda a detectar código que se volvería problemático tras cache miss.

---

# 231. Default

No elevarlo al mismo nivel que N+1 real sin política explícita.

---

# 232. Connection pooling

Número de conexiones no influye directamente en clasificación N+1.

---

# 233. Parallel N+1

100 queries ejecutadas concurrentemente siguen pudiendo ser N+1.

---

# 234. Concurrency does not erase pattern

```text
Parallelism
≠
Batching
```

---

# 235. Parallel N+1 impact

Puede reducir wall time pero aumentar carga de DB.

---

# 236. Detection must count operations

No solo tiempo.

---

# 237. Rate limiting diagnostics

Si un finding contiene miles de queries, el reporte debe resumir.

---

# 238. Query examples

Mostrar como máximo:

```text
first N normalized examples
```

en diagnóstico detallado.

---

# 239. No giant diagnostics

No almacenar/renderizar 50,000 queries.

---

# 240. Explain system

API conceptual:

```php
$report = $nPlusOneDetector->explain($finding);
```

---

# 241. Explanation

Debe responder:

```text
What happened?
Why is it classified as N+1?
How confident are we?
What was the root operation?
Which relationship caused it?
How many DB operations occurred?
Could they have been batched?
What can the developer change?
```

---

# 242. Example explanation

```text
100 User entities were produced by one root query.

The relationship User.posts was then lazily loaded
100 times from the same application call site.

All 100 owners shared the same:
- tenant
- shard
- database context
- relationship load shape

The Batch Relation Loading System reports them as compatible.

Therefore this pattern is classified as a HIGH-confidence N+1.

Suggested fix:
    eager-load or batch-load User.posts.
```

---

# 243. Architectural invariants

## DB-ORM-NPLUSONE-001

N+1 detection será semántica cuando exista metadata suficiente.

## DB-ORM-NPLUSONE-002

Raw SQL string equality no será la definición de N+1.

## DB-ORM-NPLUSONE-003

Muchas queries no implicarán automáticamente N+1.

## DB-ORM-NPLUSONE-004

SQL duplicado no implicará automáticamente N+1.

## DB-ORM-NPLUSONE-005

Lazy Loading no será considerado error por sí mismo.

## DB-ORM-NPLUSONE-006

Eager Loading no será obligatorio para toda relación.

## DB-ORM-NPLUSONE-007

Batch Loading será una posible solución, no parte del detector.

## DB-ORM-NPLUSONE-008

Detection no modificará queries.

## DB-ORM-NPLUSONE-009

Detection no inicializará relaciones.

## DB-ORM-NPLUSONE-010

Detection no modificará EntityManager.

## DB-ORM-NPLUSONE-011

Detection no modificará UnitOfWork.

## DB-ORM-NPLUSONE-012

Detection no generará SQL.

## DB-ORM-NPLUSONE-013

Detection no dependerá de PDO.

## DB-ORM-NPLUSONE-014

RelationshipId será preferido sobre SQL para correlación ORM.

## DB-ORM-NPLUSONE-015

RootOperation será distinto de HTTP request.

## DB-ORM-NPLUSONE-016

Un request podrá contener múltiples roots.

## DB-ORM-NPLUSONE-017

Fallback request-level reducirá confidence cuando corresponda.

## DB-ORM-NPLUSONE-018

Fingerprints ignorarán valores de parámetros no semánticos.

## DB-ORM-NPLUSONE-019

Fingerprints preservarán estructura semántica.

## DB-ORM-NPLUSONE-020

QueryFingerprint será preferentemente AST-based.

## DB-ORM-NPLUSONE-021

QueryFingerprint no dependerá del dialect SQL concreto.

## DB-ORM-NPLUSONE-022

Call-site capture será configurable.

## DB-ORM-NPLUSONE-023

Full stack capture no será obligatorio en producción.

## DB-ORM-NPLUSONE-024

Frames internos podrán filtrarse.

## DB-ORM-NPLUSONE-025

Temporal proximity no será evidencia suficiente por sí sola.

## DB-ORM-NPLUSONE-026

Distinct owner count será señal central.

## DB-ORM-NPLUSONE-027

Repeated same-owner load será distinguido del N+1 clásico.

## DB-ORM-NPLUSONE-028

Thresholds serán configurables.

## DB-ORM-NPLUSONE-029

Severity será distinta de confidence.

## DB-ORM-NPLUSONE-030

Impact será distinto de confidence.

## DB-ORM-NPLUSONE-031

Candidate será distinto de finding.

## DB-ORM-NPLUSONE-032

False-positive filters precederán al finding final.

## DB-ORM-NPLUSONE-033

Nested N+1 será representable.

## DB-ORM-NPLUSONE-034

Relationship graph cycles no generarán análisis infinito.

## DB-ORM-NPLUSONE-035

Polymorphic N+1 considerará target type groups.

## DB-ORM-NPLUSONE-036

Una query por target polymorphic group no será automáticamente N+1.

## DB-ORM-NPLUSONE-037

Extra-lazy operations serán observables.

## DB-ORM-NPLUSONE-038

COUNT N+1 será distinguible.

## DB-ORM-NPLUSONE-039

CONTAINS N+1 será distinguible.

## DB-ORM-NPLUSONE-040

SLICE N+1 será distinguible.

## DB-ORM-NPLUSONE-041

Duplicate Query será categoría separada.

## DB-ORM-NPLUSONE-042

Repeated Entity Lookup será categoría separada.

## DB-ORM-NPLUSONE-043

Query-level fallback tendrá menor confidence cuando corresponda.

## DB-ORM-NPLUSONE-044

Unknown no será convertido en certeza.

## DB-ORM-NPLUSONE-045

Batch compatibility podrá influir en fixability.

## DB-ORM-NPLUSONE-046

Incompatible tenants no serán considerados directamente batchable.

## DB-ORM-NPLUSONE-047

Incompatible shards no serán considerados directamente batchable.

## DB-ORM-NPLUSONE-048

Incompatible transactions no serán considerados directamente batchable.

## DB-ORM-NPLUSONE-049

Fixability será distinta de severity.

## DB-ORM-NPLUSONE-050

Recommendations serán semánticas.

## DB-ORM-NPLUSONE-051

Recommendations no reescribirán automáticamente código.

## DB-ORM-NPLUSONE-052

Development mode podrá capturar más contexto.

## DB-ORM-NPLUSONE-053

Production mode tendrá bounded overhead.

## DB-ORM-NPLUSONE-054

Testing mode será determinista.

## DB-ORM-NPLUSONE-055

Detector hot path apuntará a O(1) promedio por observación.

## DB-ORM-NPLUSONE-056

No existirá query history ilimitado.

## DB-ORM-NPLUSONE-057

Fingerprint state será bounded.

## DB-ORM-NPLUSONE-058

Owner tracking será bounded en producción.

## DB-ORM-NPLUSONE-059

Testing podrá usar tracking exacto.

## DB-ORM-NPLUSONE-060

Sampling preferirá scopes completos.

## DB-ORM-NPLUSONE-061

Sampling no destruirá correlación innecesariamente.

## DB-ORM-NPLUSONE-062

Sampling no usará secretos.

## DB-ORM-NPLUSONE-063

Detector tendrá resource governance.

## DB-ORM-NPLUSONE-064

Truncation será explícita.

## DB-ORM-NPLUSONE-065

Truncated analysis no equivaldrá a clean analysis.

## DB-ORM-NPLUSONE-066

DetectionContext será scope-local.

## DB-ORM-NPLUSONE-067

Detection state no será static global.

## DB-ORM-NPLUSONE-068

FrankenPHP requests estarán aislados.

## DB-ORM-NPLUSONE-069

RoadRunner requests estarán aislados.

## DB-ORM-NPLUSONE-070

OpenSwoole coroutines estarán aisladas.

## DB-ORM-NPLUSONE-071

Scope finalization no ejecutará queries.

## DB-ORM-NPLUSONE-072

Detector será observacional respecto al ORM.

## DB-ORM-NPLUSONE-073

Architectural events no serán domain events.

## DB-ORM-NPLUSONE-074

Batch-loaded owners no serán reportados como N+1 si no existe patrón DB repetitivo.

## DB-ORM-NPLUSONE-075

Eager-loaded relations no serán reportadas como N+1 por el simple número de rows.

## DB-ORM-NPLUSONE-076

JOIN explosion será un problema distinto.

## DB-ORM-NPLUSONE-077

IdentityMap hits serán diferenciados de DB loads.

## DB-ORM-NPLUSONE-078

Relationship access count será distinto de DB query count.

## DB-ORM-NPLUSONE-079

Cumulative latency será observable.

## DB-ORM-NPLUSONE-080

Wall time será distinto de cumulative query duration.

## DB-ORM-NPLUSONE-081

Potential query savings serán estimaciones.

## DB-ORM-NPLUSONE-082

Estimaciones serán etiquetadas como tales.

## DB-ORM-NPLUSONE-083

Findings serán deduplicados.

## DB-ORM-NPLUSONE-084

No habrá warning por cada query individual.

## DB-ORM-NPLUSONE-085

Telemetry evitará entity IDs como labels.

## DB-ORM-NPLUSONE-086

Telemetry evitará raw SQL como labels.

## DB-ORM-NPLUSONE-087

Tracing será opcional.

## DB-ORM-NPLUSONE-088

Detector funcionará sin tracing.

## DB-ORM-NPLUSONE-089

Profiler consumirá findings; detector no dependerá del profiler.

## DB-ORM-NPLUSONE-090

Debug Toolbar consumirá findings; detector no dependerá de ella.

## DB-ORM-NPLUSONE-091

Testing tendrá assertNoNPlusOne.

## DB-ORM-NPLUSONE-092

N+1 assertion será distinta de query-count assertion.

## DB-ORM-NPLUSONE-093

Testing deshabilitará sampling cuando requiera determinismo.

## DB-ORM-NPLUSONE-094

Baselines serán opcionales.

## DB-ORM-NPLUSONE-095

Suppressions serán explícitas.

## DB-ORM-NPLUSONE-096

Suppressions podrán requerir reason.

## DB-ORM-NPLUSONE-097

Production no lanzará N+1 exceptions por defecto.

## DB-ORM-NPLUSONE-098

Testing podrá convertir findings en exceptions.

## DB-ORM-NPLUSONE-099

Serialization-triggered N+1 será detectable.

## DB-ORM-NPLUSONE-100

Template-triggered N+1 será detectable.

## DB-ORM-NPLUSONE-101

Resource transformers podrán aparecer como call sites.

## DB-ORM-NPLUSONE-102

GraphQL-style resolvers podrán integrarse mediante semantic events.

## DB-ORM-NPLUSONE-103

Correct resolver batching no producirá finding.

## DB-ORM-NPLUSONE-104

CLI operations tendrán scope.

## DB-ORM-NPLUSONE-105

Jobs tendrán scopes independientes.

## DB-ORM-NPLUSONE-106

Long-running jobs podrán segmentarse en operations.

## DB-ORM-NPLUSONE-107

Raw Query Builder podrá tener query-level detection.

## DB-ORM-NPLUSONE-108

Raw Query detection no inventará RelationshipId.

## DB-ORM-NPLUSONE-109

Stored procedure internals desconocidos no serán inferidos.

## DB-ORM-NPLUSONE-110

External operations desconocidas no serán inferidas.

## DB-ORM-NPLUSONE-111

Unknown evidence no será fabricada.

## DB-ORM-NPLUSONE-112

Query normalization ignorará diferencias no semánticas.

## DB-ORM-NPLUSONE-113

Query normalization preservará scopes relevantes.

## DB-ORM-NPLUSONE-114

Cross-platform findings serán semánticamente consistentes.

## DB-ORM-NPLUSONE-115

Fingerprint collisions no deberán causar corrupción.

## DB-ORM-NPLUSONE-116

Raw parameter values no serán almacenados por defecto.

## DB-ORM-NPLUSONE-117

Historical analytics estará separado del detector.

## DB-ORM-NPLUSONE-118

DetectionContext responderá por un scope actual.

## DB-ORM-NPLUSONE-119

Telemetry podrá agregar findings entre scopes.

## DB-ORM-NPLUSONE-120

Early detection podrá ocurrir antes del final del scope.

## DB-ORM-NPLUSONE-121

Final findings requerirán evidencia suficiente.

## DB-ORM-NPLUSONE-122

Immediate warnings se emitirán una sola vez por finding key.

## DB-ORM-NPLUSONE-123

Counting continuará después del warning.

## DB-ORM-NPLUSONE-124

Finding identity incluirá correlation scope.

## DB-ORM-NPLUSONE-125

Nested findings podrán incluir parent path.

## DB-ORM-NPLUSONE-126

Call-site grouping será configurable.

## DB-ORM-NPLUSONE-127

Parallelism no será confundido con batching.

## DB-ORM-NPLUSONE-128

Parallel N+1 seguirá siendo detectable.

## DB-ORM-NPLUSONE-129

Diagnostics serán bounded.

## DB-ORM-NPLUSONE-130

Diagnostics no almacenarán miles de query examples.

## DB-ORM-NPLUSONE-131

Explain API describirá evidencia.

## DB-ORM-NPLUSONE-132

Explain API describirá confidence.

## DB-ORM-NPLUSONE-133

Explain API describirá fixability.

## DB-ORM-NPLUSONE-134

Explain API describirá recomendaciones.

## DB-ORM-NPLUSONE-135

Cache podrá reducir impacto observado.

## DB-ORM-NPLUSONE-136

Cache hits serán diferenciados de DB queries.

## DB-ORM-NPLUSONE-137

Cache-masked patterns podrán diagnosticarse opcionalmente.

## DB-ORM-NPLUSONE-138

Cache-masked patterns no tendrán por defecto la misma severidad que N+1 real.

## DB-ORM-NPLUSONE-139

Connection pooling no alterará la definición semántica.

## DB-ORM-NPLUSONE-140

Detector será independiente del application server concreto.

## DB-ORM-NPLUSONE-141

Detection policy será configurable.

## DB-ORM-NPLUSONE-142

Detection config immutable podrá compartirse entre workers.

## DB-ORM-NPLUSONE-143

Mutable observations nunca se compartirán entre requests.

## DB-ORM-NPLUSONE-144

N+1 detection deberá poder desactivarse completamente.

## DB-ORM-NPLUSONE-145

Modo OFF no mantendrá tracking específico innecesario.

## DB-ORM-NPLUSONE-146

Detector no será requisito para correctness del ORM.

## DB-ORM-NPLUSONE-147

Desactivar detector no cambiará resultados de queries.

## DB-ORM-NPLUSONE-148

Desactivar detector no cambiará relationship semantics.

## DB-ORM-NPLUSONE-149

Detección será una capacidad diagnóstica, no una dependencia funcional.

## DB-ORM-NPLUSONE-150

La optimización nunca justificará romper aislamiento o semántica para eliminar un finding.

---

# 244. Anti-patterns

## 244.1 Contar SQL idéntico

```text
if sameSql > 3:
    N+1
```

**Rechazado.**

---

## 244.2 Toda query repetida es N+1

**Rechazado.**

---

## 244.3 Todo Lazy Loading es N+1

**Rechazado.**

---

## 244.4 Todo N+1 debe convertirse en JOIN

**Rechazado.**

---

## 244.5 Capturar full stack en cada query de producción

**Rechazado por defecto.**

---

## 244.6 Guardar todos los entity IDs indefinidamente

**Rechazado.**

---

## 244.7 Estado static entre requests

**Rechazado.**

---

## 244.8 Lanzar excepción en producción por defecto

**Rechazado.**

---

## 244.9 Inventar relationship desde raw SQL

**Rechazado.**

---

## 244.10 Ignorar tenant/shard compatibility

**Rechazado.**

---

## 244.11 Reescribir queries desde el detector

**Rechazado.**

---

## 244.12 Corregir automáticamente el código

**Rechazado.**

---

## 244.13 Confundir JOIN explosion con N+1

**Rechazado.**

---

## 244.14 Confundir cache hit con DB query

**Rechazado.**

---

## 244.15 Metric labels con entity IDs

**Rechazado.**

---

# 245. Interfaces principales

```php
interface NPlusOneDetector
{
    public function observe(
        DatabaseObservation $observation,
        NPlusOneDetectionContext $context,
    ): void;

    /**
     * @return list<NPlusOneFinding>
     */
    public function findings(
        NPlusOneDetectionContext $context,
    ): array;
}
```

---

# 246. Correlation engine

```php
interface NPlusOneCorrelationEngine
{
    public function correlate(
        DatabaseObservation $observation,
        NPlusOneDetectionContext $context,
    ): CorrelationResult;
}
```

---

# 247. Fingerprint factory

```php
interface SemanticLoadFingerprintFactory
{
    public function create(
        DatabaseObservation $observation,
    ): SemanticLoadFingerprint;
}
```

---

# 248. Confidence evaluator

```php
interface NPlusOneConfidenceEvaluator
{
    public function evaluate(
        NPlusOneCandidate $candidate,
    ): NPlusOneConfidence;
}
```

---

# 249. False-positive filter

```php
interface NPlusOneCandidateFilter
{
    public function accepts(
        NPlusOneCandidate $candidate,
    ): bool;
}
```

---

# 250. Recommendation provider

```php
interface NPlusOneRecommendationProvider
{
    public function recommend(
        NPlusOneCandidate $candidate,
    ): NPlusOneRecommendation;
}
```

---

# 251. Fixability analyzer

```php
interface NPlusOneFixabilityAnalyzer
{
    public function analyze(
        NPlusOneCandidate $candidate,
    ): NPlusOneFixability;
}
```

---

# 252. Batch integration

```php
final class BatchAwareFixabilityAnalyzer
    implements NPlusOneFixabilityAnalyzer
{
    public function __construct(
        private BatchCompatibilityAnalyzer $compatibility,
    ) {}
}
```

La dependencia será hacia una abstracción de análisis, no hacia ejecución del batch.

---

# 253. Directory structure

```text
src/Quantum/Database/ORM/Diagnostics/NPlusOne/
│
├── Contract/
│   ├── NPlusOneDetector.php
│   ├── NPlusOneCorrelationEngine.php
│   ├── NPlusOneConfidenceEvaluator.php
│   ├── NPlusOneCandidateFilter.php
│   ├── NPlusOneRecommendationProvider.php
│   └── NPlusOneFixabilityAnalyzer.php
│
├── Context/
│   ├── NPlusOneDetectionContext.php
│   ├── NPlusOneCorrelationContext.php
│   └── RootOperationContext.php
│
├── Observation/
│   ├── DatabaseObservation.php
│   ├── RelationshipLoadObserved.php
│   ├── QueryExecutionObserved.php
│   ├── LazyLoadObserved.php
│   └── BatchLoadObserved.php
│
├── Fingerprint/
│   ├── SemanticLoadFingerprint.php
│   ├── RelationshipLoadFingerprint.php
│   ├── QueryFingerprint.php
│   ├── CallSiteFingerprint.php
│   ├── RootQueryFingerprint.php
│   └── SemanticLoadFingerprintFactory.php
│
├── Correlation/
│   ├── DefaultNPlusOneCorrelationEngine.php
│   ├── RootOperationCorrelator.php
│   ├── RelationshipCorrelator.php
│   ├── CallSiteCorrelator.php
│   └── TemporalCorrelator.php
│
├── Candidate/
│   ├── NPlusOneCandidate.php
│   ├── CandidateAccumulator.php
│   └── CandidateEvidence.php
│
├── Detection/
│   ├── DefaultNPlusOneDetector.php
│   ├── RelationshipNPlusOneDetector.php
│   ├── LazyNPlusOneDetector.php
│   ├── NestedNPlusOneDetector.php
│   ├── PolymorphicNPlusOneDetector.php
│   ├── ExtraLazyNPlusOneDetector.php
│   └── DuplicateQueryDetector.php
│
├── Confidence/
│   ├── NPlusOneConfidence.php
│   └── DefaultNPlusOneConfidenceEvaluator.php
│
├── Filtering/
│   ├── BatchCompatibilityFilter.php
│   ├── TenantCompatibilityFilter.php
│   ├── ShardCompatibilityFilter.php
│   ├── TransactionCompatibilityFilter.php
│   └── ExplicitLoadFilter.php
│
├── Finding/
│   ├── NPlusOneFinding.php
│   ├── NPlusOneFindingId.php
│   ├── NPlusOnePatternKind.php
│   ├── NPlusOneSeverity.php
│   └── NPlusOneFixability.php
│
├── Recommendation/
│   ├── NPlusOneRecommendation.php
│   ├── NPlusOneRecommendationKind.php
│   └── DefaultNPlusOneRecommendationProvider.php
│
├── Policy/
│   ├── NPlusOneDetectionPolicy.php
│   ├── NPlusOneThresholdPolicy.php
│   ├── NPlusOneDetectionMode.php
│   ├── NPlusOnePolicyAction.php
│   └── CallSiteCaptureMode.php
│
├── Explain/
│   ├── NPlusOneExplainer.php
│   └── NPlusOneDiagnosticReport.php
│
├── Telemetry/
│   ├── NPlusOneTelemetry.php
│   └── NPlusOneMetrics.php
│
├── Testing/
│   ├── NPlusOneTestObserver.php
│   └── NPlusOneAssertions.php
│
└── Exception/
    ├── NPlusOneDetectionException.php
    ├── NPlusOneThresholdExceededException.php
    ├── NPlusOneAnalysisTruncatedException.php
    └── NPlusOneAssertionFailedException.php
```

---

# 254. Dependency rules

Permitido:

```text
N+1 Detector
    ↓
Relationship Metadata

N+1 Detector
    ↓
Observation Contracts

N+1 Detector
    ↓
Batch Compatibility abstraction

N+1 Detector
    ↓
Telemetry contracts
```

No permitido:

```text
N+1 Detector
    ↓
SQL Compiler internals
```

ni:

```text
N+1 Detector
    ↓
PDO
```

ni:

```text
N+1 Detector
    ↓
HTTP Controller
```

---

# 255. Integration pipeline

```text
Application
    │
    ▼
Root Entity Query
    │
    ├──────────────► Root Observation
    │
    ▼
Entities
    │
    ▼
Relationship Access
    │
    ├──────────────► Relationship Observation
    │
    ▼
Lazy/Explicit Load
    │
    ├──────────────► Load Observation
    │
    ▼
Query Engine
    │
    ▼
Execution
    │
    ├──────────────► Execution Observation
    │
    ▼
N+1 Correlation Context
    │
    ▼
Candidate Accumulator
    │
    ▼
Threshold
    │
    ▼
Confidence
    │
    ▼
False Positive Filters
    │
    ▼
Finding
```

---

# 256. Detection example

Código:

```php
$users = User::query()->get();

foreach ($users as $user) {
    echo $user->posts->count();
}
```

Observaciones:

```text
Root:
    UserQuery#1

RelationshipLoad:
    User#1.posts

RelationshipLoad:
    User#2.posts

RelationshipLoad:
    User#3.posts
```

Fingerprint:

```text
RelationshipId:
    User.posts

Mode:
    LAZY

Operation:
    LOAD_COLLECTION
```

Accumulator:

```text
loads = 3
distinctOwners = 3
root = UserQuery#1
```

Resultado:

```text
N+1 Candidate
```

Después de compatibilidad:

```text
same tenant
same shard
same DB
same scope
same load shape
```

Resultado:

```text
HIGH confidence N+1
```

---

# 257. Corrected execution

```php
$users = User::query()
    ->with('posts')
    ->get();
```

Puede producir:

```text
Query 1:
    Users

Query 2:
    Batch User.posts
```

Detector:

```text
Relationship accesses:
    100

Individual DB relationship queries:
    0

Batch queries:
    1

N+1:
    NONE
```

---

# 258. Architectural formula

La detección conceptual será:

```text
NPlusOneFinding
=
RepeatedDependentDatabaseOperations
∩
SharedSemanticFingerprint
∩
SharedCorrelationContext
∩
DistinctOwners
∩
ThresholdExceeded
∩
SufficientEvidence
-
FalsePositiveConditions
```

---

# 259. Confidence formula

Conceptualmente:

```text
Confidence
=
f(
    RelationshipIdentity,
    RootCorrelation,
    DistinctOwners,
    LoadMode,
    CallSiteConsistency,
    BatchCompatibility,
    TemporalCorrelation,
    ContextConsistency
)
```

---

# 260. Impact formula

```text
Impact
=
f(
    DatabaseQueryCount,
    CumulativeLatency,
    RowsProcessed,
    BytesTransferred,
    EstimatedAvoidableQueries
)
```

---

# 261. Core separation

```text
Detection
    answers:
        "Is there an N+1 pattern?"

Batch Compatibility
    answers:
        "Could these loads be grouped?"

Recommendation
    answers:
        "What should the developer consider changing?"

Query Optimizer
    answers:
        "How should one query be optimized?"

Profiler
    answers:
        "How did database work behave?"

Telemetry
    answers:
        "How often does this happen over time?"
```

---

# 262. Master rule

La regla maestra será:

> **El detector N+1 de VoltStack observará intención ORM, relaciones, propietarios, operaciones raíz y ejecución real para identificar patrones repetitivos evitables, manteniendo separadas la detección, la severidad, la confianza, el impacto y la posibilidad real de optimización.**

Esto permite que VoltStack reporte:

```text
User.posts
100 lazy loads
100 DB queries
100 compatible owners
HIGH confidence
HIGH impact
DIRECTLY FIXABLE
```

en lugar de un diagnóstico pobre como:

```text
"Query repeated 100 times."
```

---

# 263. Resultado del bloque Relationships

Con este documento queda definido el bloque arquitectónico principal de relaciones:

```text
142 Relationship Architecture
        │
        ├── 143 One-to-One
        ├── 144 One-to-Many
        ├── 145 Many-to-One
        ├── 146 Many-to-Many
        ├── 147 Polymorphic
        │
        ├── 148 Relationship Metadata
        ├── 149 Relationship Persistence
        ├── 150 Relationship Loading
        │
        ├── 151 Eager Loading
        ├── 152 Lazy Loading
        ├── 153 Batch Relation Loading
        └── 154 N+1 Detection
```

La arquitectura resultante separa claramente:

```text
Relationship Definition
        ↓
Relationship Metadata
        ↓
Relationship Runtime State
        ↓
┌──────────────────────┐
│                      │
▼                      ▼
Persistence          Loading
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           Eager      Lazy      Batch
                                 │
                                 ▼
                         N+1 Prevention
                                 │
                                 ▼
                          N+1 Detection
```

con la precisión de que:

```text
N+1 Detection
```

es diagnóstico y no forma parte del camino obligatorio para correctness.

---

# 264. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 14
DATABASE TYPES, CASTING AND VALUE OBJECTS
```

La secuencia será:

```text
155_DATABASE_TYPE_SYSTEM.md
156_DATABASE_TYPE_REGISTRY_SYSTEM.md
157_DATABASE_VALUE_CONVERSION_SYSTEM.md
158_DATABASE_CASTING_SYSTEM.md
159_DATABASE_ENUM_MAPPING_SYSTEM.md
160_DATABASE_VALUE_OBJECT_MAPPING_SYSTEM.md
161_DATABASE_JSON_TYPE_SYSTEM.md
162_DATABASE_DATE_TIME_TYPE_SYSTEM.md
163_DATABASE_CUSTOM_TYPE_EXTENSION_SYSTEM.md
```

---

# 265. Siguiente documento

```text
155_DATABASE_TYPE_SYSTEM.md
```

Este documento deberá establecer el sistema de tipos canónico de `VoltStack/Quantum/Database` y definir claramente:

```text
PHP Type
≠
ORM Type
≠
Database Logical Type
≠
Platform Physical Type
≠
SQL Type Declaration
```

Deberá cubrir, entre otros:

- `DatabaseType`;
- `TypeId`;
- logical types;
- physical types;
- PHP type descriptors;
- database type descriptors;
- type compatibility;
- type affinity;
- nullability;
- scalar types;
- numeric types;
- string types;
- binary types;
- boolean;
- UUID/ULID;
- JSON;
- date/time;
- enum;
- decimal;
- value objects;
- custom types;
- parameter binding;
- result conversion;
- schema integration;
- ORM mapping;
- Query Type System integration;
- platform capabilities;
- type normalization;
- lossless/lossy conversion;
- overflow;
- precision/scale;
- timezone semantics;
- persistent-runtime safety;
- extensibility.

Regla central propuesta:

> **VoltStack representará los tipos de datos mediante un modelo lógico canónico independiente del dialecto, dejando que cada Platform determine su representación física y que los sistemas de conversión controlen explícitamente el tránsito entre valores PHP, valores ORM y valores de base de datos.**