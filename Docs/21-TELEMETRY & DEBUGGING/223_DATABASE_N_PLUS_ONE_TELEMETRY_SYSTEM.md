# 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md

# VoltStack Quantum Database
## N+1 Telemetry System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 223 — N+1 Telemetry System  
**Bloque:** 21 — Telemetry and Debugging  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md`  
**Siguiente documento:** `224_DATABASE_DEBUG_INFORMATION_SYSTEM.md`

---

# 1. Propósito

`N+1 Telemetry System` define la arquitectura mediante la cual VoltStack detectará, correlacionará, clasificará y explicará patrones de acceso a datos donde una operación raíz provoca múltiples consultas dependientes repetitivas que podrían haberse resuelto mediante una estrategia de carga más eficiente.

La regla central será:

> **Repetir una consulta muchas veces no demuestra por sí mismo un problema N+1; VoltStack deberá demostrar una correlación semántica entre una operación raíz, las entidades obtenidas y las consultas dependientes ejecutadas posteriormente.**

Por tanto:

```text
Repeated Query
≠
N+1

High Query Count
≠
N+1

Lazy Loading
≠
N+1

Relationship Loading
≠
N+1

N+1 Detection
≠
Automatic Eager Loading
```

---

# 2. Problema clásico

Ejemplo:

```php
$users = User::query()
    ->where('active', true)
    ->get();

foreach ($users as $user) {
    echo $user->profile->name;
}
```

Posible ejecución:

```text
Query 1
SELECT users...

Query 2
SELECT profiles WHERE user_id = 1

Query 3
SELECT profiles WHERE user_id = 2

Query 4
SELECT profiles WHERE user_id = 3

...

Query N+1
SELECT profiles WHERE user_id = N
```

Estructuralmente:

```text
1 root query
+
N dependent queries
```

Sin embargo, VoltStack no deberá detectar N+1 únicamente contando consultas.

---

# 3. Por qué el conteo no basta

Supongamos:

```text
SELECT configuration...
```

se ejecuta 50 veces.

Esto podría ser:

```text
50 operaciones independientes
```

y no:

```text
1 + N
```

Por tanto:

```text
RepeatedFingerprint
≠
NPlusOneProof
```

---

# 4. Objetivo

VoltStack deberá detectar:

```text
Root Database Operation
        │
        ▼
Root Query
        │
        ▼
Entity/Result Set
        │
        ├── Entity A
        │      └── dependent query
        │
        ├── Entity B
        │      └── dependent query
        │
        ├── Entity C
        │      └── dependent query
        │
        └── ...
               │
               ▼
        Repeated Relationship Access
```

La correlación deberá utilizar información semántica del ORM, Query Engine y Telemetry.

---

# 5. Posición arquitectónica

```text
Database Telemetry
│
├── Query Telemetry
├── Connection Telemetry
├── Transaction Telemetry
├── ORM Telemetry
├── Query Profiler
├── Slow Query Detection
│
├── N+1 Telemetry             ← este documento
│
├── Debug Information
└── Developer Debug Toolbar
```

---

# 6. Sistemas relacionados

El sistema utilizará información de:

```text
142 Relationship Architecture
148 Relationship Metadata
150 Relationship Loading
151 Eager Loading
152 Lazy Loading
153 Batch Relation Loading
154 N+1 Detection foundations

217 Query Telemetry
220 ORM Telemetry
221 Query Profiler
222 Slow Query Detection
```

El documento 154 define la semántica general de detección.

El documento 223 define su integración completa con:

```text
Telemetry
Profiling
Diagnostics
Metrics
Debugging
Developer Experience
```

---

# 7. Regla arquitectónica

```text
N+1 Detector
```

será un consumidor de eventos/evidencia.

No será:

```text
Query Executor
Relationship Loader
Query Optimizer
ORM Proxy
EntityManager
```

---

# 8. Flujo general

```text
ORM Operation
      │
      ▼
Root Query
      │
      ▼
Hydration
      │
      ▼
Entities
      │
      ▼
Relationship Access
      │
      ▼
Relationship Loader
      │
      ▼
Dependent Query
      │
      ▼
Telemetry Correlation
      │
      ▼
N+1 Candidate
      │
      ▼
Evidence Evaluation
      │
      ▼
N+1 Detection
```

---

# 9. Terminología

VoltStack distinguirá:

```text
Root Operation
Root Query
Parent Entity
Relationship Access
Dependent Query
Repeated Dependent Query
N+1 Candidate
N+1 Detection
N+1 Diagnostic
```

---

# 10. Root Operation

Representa la operación lógica que origina el patrón.

Ejemplo:

```text
UserRepository::findActiveUsers()
```

Puede incluir múltiples queries físicas.

---

# 11. Root Query

Consulta que produce el conjunto de entidades sobre las cuales posteriormente se accede una relación.

Ejemplo:

```text
SELECT users ...
```

---

# 12. Parent Entity

Entidad cuya relación provoca una carga dependiente.

Ejemplo:

```text
User#1
User#2
User#3
```

---

# 13. Relationship Access

Acceso semántico a:

```text
User.profile
```

No deberá inferirse únicamente de SQL.

---

# 14. Dependent Query

Consulta ejecutada como consecuencia del acceso a una relación.

Ejemplo:

```text
Profile WHERE user_id = ?
```

---

# 15. N+1 Candidate

Un patrón sospechoso pero todavía no confirmado.

```php
final readonly class NPlusOneCandidate
{
    public function __construct(
        public RootOperationId $rootOperationId,
        public RelationshipId $relationshipId,
        public QueryFingerprint $queryFingerprint,
        public int $occurrences,
        public NPlusOneEvidence $evidence,
    ) {}
}
```

---

# 16. N+1 Detection

Resultado confirmado o suficientemente probable:

```php
final readonly class NPlusOneDetection
{
    public function __construct(
        public NPlusOneDetectionId $id,
        public RootOperationId $rootOperationId,
        public RelationshipId $relationshipId,
        public QueryFingerprint $dependentFingerprint,
        public int $dependentQueryCount,
        public int $parentEntityCount,
        public NPlusOneSeverity $severity,
        public NPlusOneConfidence $confidence,
        public NPlusOneEvidence $evidence,
    ) {}
}
```

---

# 17. Detection ID

```php
final readonly class NPlusOneDetectionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

No será metric label.

---

# 18. Correlación semántica

El detector deberá intentar demostrar:

```text
RootOperation R
produces
EntitySet E

Relationship L
is accessed for
multiple entities e ∈ E

Each access causes
dependent query Q(e)
```

---

# 19. Modelo formal básico

Sea:

```text
E = {e1, e2, ..., en}
```

el conjunto de entidades raíz.

Si para una relación:

```text
L
```

se producen consultas:

```text
Q(e1)
Q(e2)
...
Q(en)
```

entonces existe un candidato N+1 cuando:

```text
Q(ei)
```

comparte una forma semántica común y varía principalmente por la identidad del padre.

---

# 20. Fingerprint

Ejemplo:

```sql
SELECT *
FROM profiles
WHERE user_id = ?
```

produce:

```text
QueryFingerprint F
```

para:

```text
user_id = 1
user_id = 2
user_id = 3
...
```

---

# 21. Fingerprint ≠ suficiente

También deberá existir correlación con:

```text
RelationshipId
RootOperationId
ParentEntityIdentity
RelationshipLoadId
```

cuando esté disponible.

---

# 22. RelationshipId

Se reutilizará el identificador estable definido por Relationship Metadata.

Ejemplo conceptual:

```text
App\Entity\User::profile
```

internamente representado mediante:

```text
RelationshipId
```

y no mediante strings arbitrarios.

---

# 23. RelationshipLoadId

Cada operación de carga podrá tener:

```php
final readonly class RelationshipLoadId
{
    public function __construct(
        public string $value,
    ) {}
}
```

para correlacionar:

```text
relationship access
→ loader
→ query
→ hydration
```

---

# 24. RootOperationId

Debe existir un identificador de correlación lógico:

```php
final readonly class RootOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 25. Query ID ≠ Root Operation ID

Una operación lógica puede generar:

```text
multiple QueryIds
```

---

# 26. Request ID ≠ Root Operation ID

Un request puede contener:

```text
multiple root database operations
```

---

# 27. Transaction ID ≠ Root Operation ID

Una transacción puede contener múltiples operaciones independientes.

---

# 28. ORM operation context

El sistema 220 deberá proporcionar contexto como:

```text
EntityType
RelationshipId
LoadStrategy
ParentIdentity
RootOperationId
HydrationContext
```

---

# 29. Relationship loading context

El sistema 150 deberá exponer evidencia:

```text
LAZY
EAGER
BATCH
EXPLICIT
```

---

# 30. Load strategy

```php
enum RelationshipLoadStrategy
{
    case EAGER;
    case LAZY;
    case BATCH;
    case EXPLICIT;
    case PRELOADED;
    case CUSTOM;
}
```

---

# 31. Lazy Loading

Lazy loading no es por sí mismo un problema.

Ejemplo:

```text
100 users loaded
1 relationship accessed
1 lazy query
```

No es N+1 significativo.

---

# 32. N+1 mediante Lazy Loading

Ejemplo:

```text
100 users
↓
100 accesses User.profile
↓
100 lazy queries
```

Sí puede constituir N+1.

---

# 33. Explicit Loading

También puede producir N+1:

```php
foreach ($users as $user) {
    $repository->loadProfile($user);
}
```

Aunque no exista proxy lazy.

---

# 34. N+1 ≠ Lazy Proxy Problem

El detector deberá ser independiente del mecanismo específico:

```text
proxy
magic property
repository
relationship accessor
explicit loader
```

cuando exista evidencia semántica suficiente.

---

# 35. Batch Loading

Ejemplo correcto:

```text
100 users
↓
collect IDs
↓
SELECT profiles WHERE user_id IN (...)
↓
assemble relationships
```

No es N+1.

---

# 36. Eager Loading

Puede producir:

```text
JOIN
```

o:

```text
SELECT_IN
```

o:

```text
BATCH
```

según planner.

N+1 Detector no deberá asumir:

```text
EAGER = JOIN
```

---

# 37. Eager Loading ≠ always optimal

Una eager load excesiva también puede ser costosa.

Pero pertenece a otro diagnóstico.

---

# 38. Candidate creation

Un candidato podrá crearse cuando:

```text
same root operation
+
same relationship
+
same dependent fingerprint
+
multiple parent identities
```

alcancen un mínimo configurable.

---

# 39. Minimum occurrence threshold

Ejemplo:

```text
3
```

consultas dependientes.

No deberá codificarse rígidamente.

---

# 40. Threshold ≠ proof

Tres queries repetidas solo crean evidencia.

La clasificación final dependerá de confianza.

---

# 41. NPlusOneConfidence

```php
enum NPlusOneConfidence
{
    case HIGH;
    case MEDIUM;
    case LOW;
    case UNKNOWN;
}
```

---

# 42. High confidence

Ejemplo:

```text
same RootOperationId
same RelationshipId
same dependent fingerprint
distinct ParentEntityIdentity
load strategy = LAZY
one query per relationship access
```

Resultado:

```text
HIGH
```

---

# 43. Medium confidence

Ejemplo:

```text
same root operation
same fingerprint
different bound parent IDs
temporal correlation
relationship metadata unavailable
```

---

# 44. Low confidence

Ejemplo:

```text
same fingerprint
same request
many executions
```

sin evidencia ORM.

---

# 45. UNKNOWN

Si la cobertura de telemetría es insuficiente:

```text
UNKNOWN
```

No se elevará artificialmente a HIGH.

---

# 46. Evidence model

```php
final readonly class NPlusOneEvidence
{
    public function __construct(
        public int $rootEntityCount,
        public int $relationshipAccessCount,
        public int $dependentQueryCount,
        public int $distinctParentCount,
        public ?RelationshipId $relationshipId,
        public QueryFingerprint $dependentFingerprint,
        public RelationshipLoadStrategy $loadStrategy,
        public NPlusOneTemporalEvidence $temporal,
        public NPlusOneParameterEvidence $parameters,
        public NPlusOneCoverage $coverage,
    ) {}
}
```

---

# 47. Evidence coverage

```php
enum NPlusOneCoverage
{
    case COMPLETE;
    case PARTIAL;
    case MINIMAL;
    case UNKNOWN;
}
```

---

# 48. COMPLETE

Podría incluir:

```text
root query
root entities
relationship metadata
relationship accesses
dependent queries
parent identities
load strategy
```

---

# 49. PARTIAL

Ejemplo:

```text
relationship accesses
+
dependent queries
```

pero root hydration incompleta.

---

# 50. MINIMAL

Solo:

```text
repeated fingerprint
+
temporal locality
```

---

# 51. UNKNOWN ≠ NONE

Ausencia de evidencia no demuestra ausencia de N+1.

---

# 52. Parent identity correlation

El detector podrá comprobar:

```text
Query parameter
↔
Parent entity identifier
```

sin almacenar necesariamente el valor real.

---

# 53. Privacy-safe identity correlation

En lugar de:

```text
user_id = 128947
```

podrá utilizarse:

```text
ParentIdentityToken
```

o hash interno efímero.

---

# 54. ParentIdentityToken

```php
final readonly class ParentIdentityToken
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 55. Token scope

Debe ser:

```text
operation-local
```

o:

```text
request-local
```

según implementación.

No deberá convertirse en identificador persistente de usuario.

---

# 56. Parameter evidence

Conceptualmente:

```text
Query A parameter token → Parent A
Query B parameter token → Parent B
Query C parameter token → Parent C
```

aumenta confianza.

---

# 57. Composite identifiers

Debe soportar:

```text
(order_id, line_number)
```

sin asumir IDs escalares.

---

# 58. Tenant-scoped identifiers

Identidad ORM efectiva:

```text
EntityType
+
Identifier
+
PersistenceDomain
+
TenantContext
```

según las reglas de IdentityMap.

---

# 59. Cross-tenant correlation

Prohibida por default.

---

# 60. Shard context

Dos queries similares en shards distintos no deberán correlacionarse incorrectamente.

---

# 61. Temporal locality

N+1 suele mostrar:

```text
Root query
↓
dependent query
dependent query
dependent query
dependent query
```

dentro de una ventana cercana.

---

# 62. Temporal locality ≠ proof

Workers concurrentes pueden ejecutar queries similares simultáneamente.

---

# 63. Concurrency awareness

El detector deberá utilizar:

```text
operation context
correlation IDs
fiber/coroutine scope
```

en lugar de depender únicamente del timestamp.

---

# 64. Async execution

En futuras operaciones concurrentes:

```text
Root
├── relation load A
├── relation load B
└── relation load C
```

pueden ejecutarse paralelamente.

Sigue siendo posible un patrón N+1 aunque no sea secuencial.

---

# 65. N+1 ≠ sequential-only

El detector deberá reconocer fan-out correlacionado.

---

# 66. N+1 fan-out

```text
              Root
        ┌──────┼──────┐
        ▼      ▼      ▼
       Q1     Q2      Q3
```

puede ser semánticamente equivalente al patrón secuencial.

---

# 67. Query count model

Conceptualmente:

```text
DependentRatio
=
dependentQueryCount
/
distinctParentCount
```

---

# 68. Ratio cercano a 1

Puede indicar:

```text
one dependent query per parent
```

---

# 69. Ratio > 1

Puede indicar:

```text
multiple relationship queries per parent
```

o relaciones anidadas.

---

# 70. Ratio < 1

Puede ocurrir por:

```text
IdentityMap hits
preloaded relations
conditional accesses
cached relations
batching
```

---

# 71. N+1 no requiere N exacto

Ejemplo:

```text
100 parents
63 dependent queries
```

puede seguir siendo un problema relevante.

---

# 72. N+1 family

El término N+1 se utilizará como categoría general de:

```text
per-parent dependent query amplification
```

no como igualdad matemática estricta.

---

# 73. Query amplification factor

Podrá calcularse:

```text
QueryAmplification
=
dependentQueryCount
/
rootQueryCount
```

---

# 74. Amplification ≠ severity

Un factor alto con queries extremadamente baratas puede tener impacto bajo.

---

# 75. Cost amplification

Podrá calcularse:

```text
DependentDatabaseTime
=
Σ dependentQueryDuration
```

---

# 76. Total N+1 cost

```text
NPlusOneCost
=
Σ dependent operation cost
```

podrá incluir:

```text
connection acquisition
database execution
transfer
hydration
ORM assembly
```

según profile coverage.

---

# 77. Cost attribution

Debe evitar double counting cuando fases se solapen.

---

# 78. NPlusOneSeverity

```php
enum NPlusOneSeverity
{
    case NOTICE;
    case WARNING;
    case HIGH;
    case CRITICAL;
}
```

---

# 79. Severity inputs

Podrá considerar:

```text
dependent query count
distinct parent count
aggregate duration
request budget percentage
relationship type
confidence
workload class
result cardinality
```

---

# 80. Ejemplo NOTICE

```text
5 parents
5 dependent queries
aggregate = 2 ms
```

---

# 81. Ejemplo WARNING

```text
50 parents
50 queries
aggregate = 90 ms
```

---

# 82. Ejemplo HIGH

```text
500 parents
500 queries
aggregate = 1.8 s
```

---

# 83. Ejemplo CRITICAL

```text
10,000 parents
10,000 queries
aggregate = 30 s
```

o cuando consume una parte crítica del presupuesto operacional.

---

# 84. Severity policy

Los números anteriores son ejemplos.

No deberán codificarse como reglas universales.

---

# 85. Workload awareness

Un N+1 en:

```text
HTTP interactive request
```

puede tener severidad diferente a:

```text
offline migration
```

---

# 86. Detection policy

```php
final readonly class NPlusOnePolicy
{
    public function __construct(
        public bool $enabled,
        public int $minimumDependentQueries,
        public int $minimumDistinctParents,
        public NPlusOneConfidence $minimumConfidence,
        public NPlusOneSeverityPolicy $severity,
        public NPlusOneSamplingPolicy $sampling,
        public NPlusOneDiagnosticPolicy $diagnostics,
    ) {}
}
```

---

# 87. Compiled policy

Durante bootstrap:

```text
Configuration
↓
Validation
↓
CompiledNPlusOnePolicy
```

---

# 88. Detector

```php
interface NPlusOneDetector
{
    public function evaluate(
        NPlusOneObservationSet $observations,
        NPlusOneDetectionContext $context,
    ): NPlusOneDetectionResult;
}
```

---

# 89. Incremental detector

Para requests largos podrá existir:

```php
interface IncrementalNPlusOneDetector
{
    public function observe(
        NPlusOneObservation $observation,
    ): void;

    public function finalize(): NPlusOneDetectionSet;
}
```

---

# 90. Incremental ≠ global mutable state

El detector deberá estar asociado al scope correspondiente.

---

# 91. Observation model

```php
interface NPlusOneObservation
{
    public function occurredAt(): Instant;
}
```

Especializaciones:

```text
RootQueryObservation
EntityHydratedObservation
RelationshipAccessObservation
RelationshipLoadObservation
DependentQueryObservation
BatchLoadObservation
CacheHitObservation
```

---

# 92. RootQueryObservation

```php
final readonly class RootQueryObservation implements NPlusOneObservation
{
    public function __construct(
        public RootOperationId $rootOperationId,
        public QueryFingerprint $fingerprint,
        public QueryId $queryId,
        public Instant $occurredAt,
    ) {}
}
```

---

# 93. EntityHydratedObservation

Conceptualmente:

```text
RootOperationId
EntityType
ParentIdentityToken
```

No deberá retener la entidad.

---

# 94. No entity retention

El detector nunca deberá almacenar:

```text
User object
EntityManager
IdentityMap
UnitOfWork
```

como parte de telemetry.

---

# 95. RelationshipAccessObservation

```php
final readonly class RelationshipAccessObservation
{
    public function __construct(
        public RootOperationId $rootOperationId,
        public RelationshipId $relationshipId,
        public ParentIdentityToken $parent,
        public RelationshipLoadStrategy $strategy,
        public Instant $occurredAt,
    ) {}
}
```

---

# 96. DependentQueryObservation

Deberá relacionar:

```text
QueryId
QueryFingerprint
RootOperationId
RelationshipLoadId
RelationshipId
ParentIdentityToken
```

cuando la evidencia exista.

---

# 97. BatchLoadObservation

Si:

```text
50 parents
↓
1 batch query
```

el detector deberá reconocer batching.

---

# 98. Batch evidence

Esto puede cancelar o reducir un candidato.

---

# 99. Cache hit

Una relación puede resolverse mediante:

```text
IdentityMap
Entity Cache
Relationship state
```

sin query.

Esto no contribuye a N+1 de consultas.

---

# 100. IdentityMap hit

No deberá registrarse como dependent query.

---

# 101. Relationship already loaded

No deberá registrarse como lazy query.

---

# 102. Relationship coverage

Si una colección está:

```text
PARTIAL
```

un acceso posterior puede legítimamente requerir otra carga.

El detector deberá preservar esta semántica.

---

# 103. Partial relation ≠ unloaded relation

Fundamental.

---

# 104. Polymorphic relationships

Ejemplo:

```text
Comment.commentable
```

puede producir queries hacia:

```text
posts
videos
images
```

---

# 105. Polymorphic N+1

El detector deberá agrupar por:

```text
RelationshipId
+
TargetEntityType
+
DependentFingerprint
```

cuando sea necesario.

---

# 106. Diferentes target types

No deberán mezclarse artificialmente como una sola query family.

---

# 107. Many-to-many

Ejemplo:

```text
User.roles
```

podría requerir:

```text
pivot lookup
+
role lookup
```

según estrategia.

El detector deberá entender que una sola carga lógica puede involucrar más de una query física.

---

# 108. Logical load ≠ physical query count

Por tanto:

```text
1 relationship load
```

puede generar:

```text
2 physical queries
```

sin representar dos N+1 independientes.

---

# 109. Relationship Load Graph

Conceptualmente:

```text
RelationshipLoad
│
├── Query A
├── Query B
└── Hydration
```

deberá correlacionarse como una unidad lógica.

---

# 110. Nested N+1

Ejemplo:

```php
foreach ($users as $user) {
    foreach ($user->posts as $post) {
        echo $post->author->profile->avatar;
    }
}
```

Puede generar:

```text
User.posts
      ↓
Post.author
      ↓
User.profile
```

---

# 111. N+1 tree

El detector deberá poder representar:

```text
Root
└── User.posts          N+1
    └── Post.author     N+1
        └── User.profile N+1
```

---

# 112. NPlusOneGraph

```php
final readonly class NPlusOneGraph
{
    public function __construct(
        public NPlusOneNode $root,
        public array $children,
    ) {}
}
```

---

# 113. Graph ≠ Entity Graph

Será un graph diagnóstico, no un graph ORM vivo.

---

# 114. Duplicate nested detection

El sistema deberá evitar producir cientos de diagnostics redundantes para el mismo árbol.

---

# 115. Parent-child detections

Podrán correlacionarse mediante:

```text
NPlusOneDetectionId
parentDetectionId
```

---

# 116. Source location

Cuando sea posible, se capturará:

```text
relationship access call site
```

más útil que el call site interno del Query Executor.

---

# 117. Ejemplo

En lugar de:

```text
LazyRelationshipLoader.php:183
```

mostrar:

```text
UserController.php:74
$user->profile->avatar
```

---

# 118. Source attribution

Puede utilizar:

```text
lightweight call-site token
```

del ORM telemetry.

---

# 119. Stack traces

No se capturarán completas para cada relationship access.

---

# 120. Conditional source capture

Podrá activarse:

```text
development
debug mode
sampled diagnostic
```

---

# 121. Query source

Podrá clasificarse:

```text
ORM_RELATIONSHIP
REPOSITORY_RELATIONSHIP
EXPLICIT_RELATION_LOAD
UNKNOWN
```

---

# 122. Raw Query Builder

Si una aplicación manualmente ejecuta:

```php
foreach ($ids as $id) {
    DB::table('profiles')
        ->where('user_id', $id)
        ->first();
}
```

no existe Relationship Metadata.

---

# 123. Generic N+1 heuristic

VoltStack podrá detectar un candidato mediante:

```text
same root operation
same fingerprint
changing parameter
tight correlation
high repetition
```

pero confianza será menor.

---

# 124. Semantic detector vs heuristic detector

Arquitectura:

```text
N+1 Detection
├── Semantic ORM Detector
└── Generic Query Pattern Detector
```

---

# 125. Semantic detector

Preferido.

Utiliza:

```text
RelationshipId
ParentIdentity
LoadStrategy
RootOperation
```

---

# 126. Generic detector

Fallback.

Utiliza:

```text
fingerprint
parameter shape
temporal locality
call site
operation context
```

---

# 127. Raw SQL

También puede generar N+1.

No deberá quedar completamente invisible.

---

# 128. Confidence degradation

Sin ORM metadata:

```text
HIGH
→ MEDIUM/LOW
```

según evidencia.

---

# 129. Parameter privacy

El detector genérico no necesita almacenar:

```text
actual parameter values
```

Puede usar:

```text
typed parameter fingerprints
ephemeral equality tokens
```

---

# 130. QueryParameterShape

Ejemplo:

```text
INT
STRING
UUID
DATE
```

---

# 131. Parameter variability

Puede detectar que:

```text
same query shape
+
one parameter changes per execution
```

sin conocer su contenido.

---

# 132. Temporal window

El detector genérico utilizará una ventana acotada.

No mantendrá queries históricas ilimitadamente.

---

# 133. Request boundary

Por default:

```text
N+1 correlation scope
=
database operation/request/job
```

No entre requests independientes.

---

# 134. Cross-request detection

Puede existir en observabilidad agregada, pero sería:

```text
repeated query pattern
```

no N+1 request-level confirmado.

---

# 135. Persistent runtimes

Especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 136. No cross-request leakage

```text
Request A
Root A
N+1 observations A

reset

Request B
must not inherit A
```

---

# 137. Fiber isolation

```text
Fiber A → correlation A
Fiber B → correlation B
```

---

# 138. Coroutine isolation

Misma regla.

---

# 139. Request reset

El reset deberá liberar:

```text
observations
candidates
identity tokens
source tokens
aggregates
graphs
```

---

# 140. Bounded memory

Un request con:

```text
100,000 relationship accesses
```

no deberá conservar 100,000 objetos completos.

---

# 141. Aggregation

Después de suficiente evidencia podrá mantenerse:

```text
count
distinct parent estimate
duration aggregate
sample queries
sample source locations
```

---

# 142. Exact vs approximate cardinality

Para grandes volúmenes podrá permitirse:

```text
approximate distinct count
```

si se etiqueta como estimación.

---

# 143. Estimate ≠ exact

Nunca reportar:

```text
10,000 exact parents
```

si fue aproximado.

---

# 144. Observation budget

```php
final readonly class NPlusOneObservationBudget
{
    public function __construct(
        public int $maxObservations,
        public int $maxCandidates,
        public int $maxSamplesPerCandidate,
        public int $maxSourceLocations,
    ) {}
}
```

---

# 145. Budget exhausted

El sistema deberá continuar con:

```text
PARTIAL
```

o agregación reducida.

No fallar la aplicación.

---

# 146. Telemetry must not break query execution

Un error interno del detector:

```text
NPlusOneTelemetryException
```

no deberá provocar rollback de la operación de negocio por default.

---

# 147. Strict testing mode

En tests podrá existir:

```text
failOnNPlusOne()
```

como policy explícita.

---

# 148. Production mode

Por default:

```text
detect
record
report
```

sin alterar comportamiento de negocio.

---

# 149. Testing API

Ejemplo:

```php
$this->database()
    ->assertNoNPlusOne(function () {
        $this->get('/users');
    });
```

---

# 150. Otra API

```php
NPlusOne::assertNone();
```

como facade de testing podría existir.

---

# 151. Testing facade ≠ runtime static state

La facade resolverá el collector scoped desde container.

---

# 152. Expected N+1

Tests podrán declarar excepciones:

```php
NPlusOne::allow(
    relationship: User::class . '::profile',
    maxQueries: 3,
);
```

pero deberá evitarse como escape indiscriminado.

---

# 153. Suppression policy

Puede existir:

```text
SUPPRESS
DOWNGRADE
ALLOW
```

para casos conocidos.

---

# 154. Suppression identity

Preferir:

```text
RelationshipId
+
RootOperation fingerprint
```

sobre raw SQL.

---

# 155. Suppression reason

Toda supresión deberá poder documentarse.

---

# 156. NPlusOneSuppression

```php
final readonly class NPlusOneSuppression
{
    public function __construct(
        public NPlusOneSuppressionRuleId $id,
        public string $reason,
    ) {}
}
```

---

# 157. Suppression ≠ absence

Una detección suprimida podrá contarse internamente.

---

# 158. Deduplication

Si una relación genera:

```text
500 queries
```

se produce:

```text
1 diagnostic agregado
```

no 500 warnings.

---

# 159. Diagnostic key

Conceptualmente:

```text
RootOperationFingerprint
+
RelationshipId
+
DependentQueryFingerprint
```

---

# 160. Nested diagnostic key

Podrá incorporar:

```text
relationship path
```

como:

```text
User.posts.author.profile
```

con cardinalidad controlada.

---

# 161. Diagnostic output

Ejemplo:

```text
N+1 QUERY DETECTED
────────────────────────────────────

Root operation:
  UserRepository::active()

Root query:
  users.active

Relationship:
  User.profile

Parent entities:
  124

Relationship accesses:
  124

Dependent queries:
  124

Load strategy:
  LAZY

Aggregate DB time:
  186 ms

Aggregate operation time:
  241 ms

Confidence:
  HIGH
```

---

# 162. Recommendation

Podrá añadirse:

```text
Consider eager or batch loading User.profile.
```

Pero:

```text
Recommendation
≠
automatic rewrite
```

---

# 163. Recommendation engine

```php
interface NPlusOneRecommendationProvider
{
    public function recommend(
        NPlusOneDetection $detection,
        NPlusOneDiagnosticContext $context,
    ): NPlusOneRecommendationSet;
}
```

---

# 164. Recommendation possibilities

```text
EAGER_LOAD
BATCH_LOAD
SELECT_IN
EXPLICIT_PRELOAD
REVIEW_ACCESS_PATTERN
REDUCE_RESULT_SET
```

---

# 165. No automatic JOIN recommendation

Porque:

```text
JOIN
```

puede romper:

```text
pagination semantics
root cardinality
memory behavior
```

o generar row explosion.

---

# 166. Planner authority

La estrategia final pertenece a:

```text
Relationship Loading Planner
```

no al detector.

---

# 167. Auto-fix prohibition

El detector no deberá hacer:

```php
$query->with('profile');
```

automáticamente.

---

# 168. Future optimization advisor

Podrá existir:

```text
DatabasePerformanceAdvisor
```

que consuma detecciones.

Pero seguirá separado.

---

# 169. Query cost

Cada dependent query podrá tener:

```text
QueryProfile
```

del documento 221.

---

# 170. Aggregate profile

```php
final readonly class NPlusOneAggregateProfile
{
    public function __construct(
        public int $queries,
        public Duration $totalDuration,
        public Duration $databaseDuration,
        public Duration $hydrationDuration,
        public Duration $maximumQueryDuration,
        public Duration $averageQueryDuration,
    ) {}
}
```

---

# 171. Slow query correlation

Un N+1 puede contener queries lentas.

Ejemplo:

```text
100 × 500 ms
```

Esto puede producir:

```text
N+1 detection
+
SlowQuery detections
```

---

# 172. Deduplication entre sistemas

Debug UI deberá correlacionarlos, no eliminar uno.

---

# 173. Fast N+1

Ejemplo:

```text
1,000 × 0.5 ms
=
500 ms
```

Ninguna query individual es lenta.

El patrón sigue siendo importante.

---

# 174. Slow Query Detection ≠ N+1 Detection

Esta es una de las razones principales para mantener ambos sistemas separados.

---

# 175. Cache interaction

Supongamos:

```text
100 relationship accesses
90 cache hits
10 DB queries
```

El detector deberá reflejar:

```text
100 accesses
10 dependent queries
90 cache-resolved
```

---

# 176. Cache hides N+1?

Puede existir un patrón de acceso ineficiente aunque el cache reduzca el costo.

VoltStack podrá clasificarlo como:

```text
RELATIONSHIP_ACCESS_AMPLIFICATION
```

pero deberá distinguirlo de:

```text
DATABASE_N_PLUS_ONE
```

---

# 177. IdentityMap effect

```text
100 accesses
1 managed related entity
```

puede evitar múltiples queries.

No deberá inventarse N+1.

---

# 178. Second-level Entity Cache

Igualmente.

---

# 179. Cache diagnostics

Podrán indicar:

```text
Potential N+1 currently masked by cache
```

solo con evidencia suficiente.

---

# 180. Replica routing

Dependent queries pueden ir a replica.

El detector deberá conservar:

```text
connection role
```

como contexto.

---

# 181. Sticky connection

Si root y dependent queries son forzadas al writer por sticky policy, no deberá atribuirse a N+1 como causa.

---

# 182. Replica lag

Es contexto independiente.

---

# 183. Transactions

N+1 puede ocurrir dentro o fuera de transacción.

---

# 184. Transaction context

Podrá incluir:

```text
transaction ID token
isolation
age
```

sin convertir transaction ID en metric label.

---

# 185. Locking relationships

Un loop con:

```text
SELECT ... FOR UPDATE
```

puede ser intencional.

El detector podrá reconocer:

```text
locking read
```

y ajustar recomendación/severidad.

---

# 186. Locking N+1

Sigue siendo posible, pero reemplazarlo por batch puede cambiar semántica de locks.

Por tanto:

```text
automatic optimization prohibited
```

---

# 187. Sharding

Ejemplo:

```text
100 users
distributed over 10 shards
```

cargar relación por usuario puede producir:

```text
100 shard queries
```

---

# 188. Shard-aware batching

Una alternativa podría ser:

```text
group parents by shard
↓
10 batch queries
```

---

# 189. Detector no hace shard regrouping

Solo podrá recomendarlo.

---

# 190. Cross-shard relationships

Si están prohibidas por arquitectura:

```text
N+1 detector
```

no deberá normalizarlas como patrón aceptable.

---

# 191. Distributed query count

Deberá distinguir:

```text
logical dependent loads
```

de:

```text
physical shard queries
```

---

# 192. Fan-out

Una sola carga lógica distribuida puede generar:

```text
N physical queries
```

sin ser ORM N+1.

---

# 193. Critical distinction

```text
Distributed Fan-Out
≠
N+1
```

---

# 194. Fan-out evidence

El Query Planner/Distribution telemetry deberá marcar queries como miembros de la misma:

```text
DistributedExecutionId
```

para evitar falsos positivos.

---

# 195. Pagination

Un resultado paginado de 20 entidades con 20 relationship queries puede ser N+1.

---

# 196. Cursor pagination

Misma regla.

---

# 197. Chunk processing

Cada chunk puede generar un N+1 local.

Ejemplo:

```text
chunk size = 500
500 relationship queries per chunk
```

---

# 198. Chunk scope

El detector podrá agrupar:

```text
per chunk
```

y también:

```text
per traversal
```

si existe correlación.

---

# 199. Lazy Collection

Una LazyCollection puede consumir durante mucho tiempo.

El scope de detección deberá ser bounded.

---

# 200. Lazy N+1

Ejemplo:

```php
User::query()
    ->lazy()
    ->each(fn ($user) => $user->profile->name);
```

Puede producir N+1.

---

# 201. Lazy traversal grouping

La correlación podrá utilizar:

```text
LazyTraversalId
```

o:

```text
RootOperationId
```

---

# 202. Streaming results

Un streaming result no deberá requerir almacenar todos los padres para detectar el patrón.

---

# 203. Incremental aggregation

Se mantendrá:

```text
count
distinct-parent evidence
samples
aggregate duration
```

de manera incremental.

---

# 204. Bulk operations

No deberán confundirse con N+1 simplemente por emitir varias queries internas planificadas.

---

# 205. Bulk operation ID

Queries internas podrán correlacionarse mediante:

```text
BulkOperationId
```

para evitar falsos positivos.

---

# 206. Import

Un import que hace lookup por fila podría generar un patrón similar a N+1.

Podrá clasificarse:

```text
PER_RECORD_LOOKUP_AMPLIFICATION
```

---

# 207. Import N+1

Puede reportarse bajo la familia general:

```text
QUERY_AMPLIFICATION
```

con subtipo:

```text
IMPORT_LOOKUP
```

en lugar de ORM relationship N+1.

---

# 208. Generalización

La arquitectura puede modelar:

```text
Query Amplification
├── ORM Relationship N+1
├── Repository N+1
├── Raw Query Loop
├── Import Per-Row Lookup
└── Application Query Fan-Out
```

---

# 209. Core scope inicial

La V1 deberá priorizar:

```text
ORM Relationship N+1
+
high-confidence repository/query-loop candidates
```

---

# 210. QueryAmplificationType

```php
enum QueryAmplificationType
{
    case ORM_RELATIONSHIP_N_PLUS_ONE;
    case REPOSITORY_N_PLUS_ONE;
    case QUERY_LOOP;
    case IMPORT_LOOKUP;
    case APPLICATION_FAN_OUT;
    case UNKNOWN;
}
```

---

# 211. N+1 remains specialized

Aunque exista esta generalización:

```text
N+1 Telemetry
```

mantendrá reglas específicas para relaciones ORM.

---

# 212. Metrics

Métricas posibles:

```text
database.n_plus_one.detected
database.n_plus_one.dependent_queries
database.n_plus_one.aggregate_duration
database.n_plus_one.parent_count
```

---

# 213. Metric labels permitidos

```text
severity
confidence
load_strategy
source
workload_class
```

---

# 214. Labels no recomendados

```text
RelationshipId
EntityClass
raw SQL
source file
tenant ID
query ID
request ID
```

por cardinalidad.

---

# 215. Relationship metrics

Si se necesitan relaciones específicas, deberán enviarse preferentemente a:

```text
logs
traces
diagnostic events
```

no a labels métricos globales.

---

# 216. Tracing

Podrá visualizarse:

```text
HTTP Request
│
├── DB users
│
├── DB profile
├── DB profile
├── DB profile
├── DB profile
│
└── N+1 diagnostic
```

---

# 217. Trace annotation

En lugar de crear un span pesado por detection podrá añadirse:

```text
database.n_plus_one = true
```

al span raíz relevante.

---

# 218. Span events

Podrá emitirse:

```text
db.n_plus_one.detected
```

con atributos bounded.

---

# 219. Debug toolbar

El documento 225 podrá presentar:

```text
Queries: 132
N+1 patterns: 2
Potential saved queries: 119
```

---

# 220. Potential saved queries

Será estimación.

Ejemplo:

```text
124 dependent queries
possible batch strategy ≈ 1 query
potential reduction ≈ 123
```

pero deberá marcarse:

```text
ESTIMATED
```

---

# 221. No optimization guarantee

Batching puede requerir:

```text
multiple batches
multiple shards
parameter limits
```

Por tanto no prometer:

```text
124 → exactly 1
```

---

# 222. Parameter limits

Una DB puede limitar:

```text
IN (...)
```

a cierta cantidad práctica/técnica.

El Relationship Planner decide batching.

---

# 223. Capability model

N+1 recommendations deberán consultar:

```text
PlatformCapabilities
```

si sugieren mecanismos específicos.

---

# 224. No vendor conditionals

No:

```php
if ($database === 'postgres') {
}
```

en detector genérico.

---

# 225. MySQL/MariaDB

Seguirán siendo plataformas separadas cuando las capabilities difieran.

---

# 226. Diagnostic model

```php
final readonly class NPlusOneDiagnostic
{
    public function __construct(
        public NPlusOneDetection $detection,
        public NPlusOneDiagnosticSummary $summary,
        public NPlusOneEvidenceView $evidence,
        public NPlusOneRecommendationSet $recommendations,
        public NPlusOneSourceContext $source,
    ) {}
}
```

---

# 227. Evidence view

Deberá estar redactada.

No expondrá IDs de entidades por default.

---

# 228. Ejemplo completo

```text
N+1 QUERY PATTERN
────────────────────────────────────

Relationship
  User.profile

Root operation
  UserRepository::findActive()

Root entities
  250

Relationship accesses
  247

Dependent DB queries
  247

Strategy
  LAZY

Dependent query fingerprint
  9f7c...

Aggregate DB time
  381 ms

Aggregate hydration time
  74 ms

Total attributed cost
  492 ms

Confidence
  HIGH

Source
  UserController.php:83

Recommendation
  Consider batch/eager loading User.profile.
```

---

# 229. Query samples

Podrán conservarse:

```text
maximum 1–3 normalized query samples
```

no cientos.

---

# 230. Sample parameters

Redactados por default.

---

# 231. Source samples

También bounded.

---

# 232. N+1 event

```php
final readonly class NPlusOneDetected
{
    public function __construct(
        public NPlusOneDetectionId $detectionId,
        public NPlusOneSeverity $severity,
        public NPlusOneConfidence $confidence,
        public QueryFingerprint $dependentFingerprint,
        public int $queryCount,
    ) {}
}
```

---

# 233. Event payload

Deberá ser pequeño.

El diagnostic completo podrá recuperarse mediante referencia/scoped collector.

---

# 234. Event system integration

El evento pertenece a:

```text
Database Telemetry
```

y podrá integrarse con el Event System general sin hacer que el detector dependa de listeners específicos.

---

# 235. Event failure

Un listener de telemetry no deberá romper la query por default.

---

# 236. Testing strict mode exception

Solo bajo policy explícita podrá convertirse en failure de test.

---

# 237. Error hierarchy

```text
NPlusOneTelemetryException
├── NPlusOneDetectionException
├── NPlusOneCorrelationException
├── NPlusOnePolicyException
├── NPlusOneDiagnosticException
├── NPlusOneBudgetException
└── NPlusOneExtensionException
```

---

# 238. Runtime behavior on detector failure

Default:

```text
record internal telemetry error
↓
disable/skip affected diagnostic
↓
continue application
```

---

# 239. Fail-open

Para observabilidad:

```text
fail-open
```

será default.

---

# 240. Security exception

La redacción y aislamiento de datos no deberán degradarse fail-open.

Si diagnostic no puede garantizar seguridad:

```text
drop diagnostic detail
```

---

# 241. Security principle

```text
Telemetry failure
must not expose data.
```

---

# 242. Extension architecture

```php
interface NPlusOneDetectionRule
{
    public function evaluate(
        NPlusOneCandidate $candidate,
        NPlusOneDetectionContext $context,
    ): NPlusOneRuleResult;
}
```

---

# 243. Rules

Implementaciones:

```text
RelationshipCorrelationRule
ParentIdentityCorrelationRule
FingerprintCorrelationRule
TemporalCorrelationRule
BatchExclusionRule
DistributedFanOutExclusionRule
CacheResolutionRule
SourceCorrelationRule
```

---

# 244. Rule result

```php
final readonly class NPlusOneRuleResult
{
    public function __construct(
        public NPlusOneRuleDecision $decision,
        public float $weight,
        public NPlusOneRuleEvidence $evidence,
    ) {}
}
```

---

# 245. Rule decision

```php
enum NPlusOneRuleDecision
{
    case SUPPORTS;
    case CONTRADICTS;
    case NEUTRAL;
    case UNKNOWN;
}
```

---

# 246. Weighted confidence

La V1 puede utilizar reglas deterministas.

No necesita un modelo probabilístico complejo.

---

# 247. Deterministic confidence

Ejemplo conceptual:

```text
Relationship correlation       HIGH evidence
Parent correlation             HIGH evidence
Fingerprint correlation        HIGH evidence
Batch exclusion                passed
Distributed fan-out exclusion  passed
```

→

```text
HIGH confidence
```

---

# 248. Explainability

El detector deberá explicar:

```text
Why was this classified as N+1?
```

---

# 249. Ejemplo de evidencia

```text
Detected because:

✓ 120 parent entities came from the same root operation.
✓ User.profile was accessed for 118 distinct parents.
✓ 118 dependent queries shared the same semantic fingerprint.
✓ Each query was correlated with a different parent identity.
✓ Relationship load strategy was LAZY.
✓ No batch load was observed.
```

---

# 250. Contradictory evidence

Ejemplo:

```text
100 physical queries
```

pero todas pertenecen a:

```text
one distributed fan-out operation
```

Resultado:

```text
NOT_N_PLUS_ONE
```

---

# 251. Detection decision

```php
enum NPlusOneDecision
{
    case DETECTED;
    case SUSPECTED;
    case NOT_DETECTED;
    case INCONCLUSIVE;
}
```

---

# 252. SUSPECTED

Útil para raw SQL o baja cobertura.

---

# 253. INCONCLUSIVE

Cuando evidencia contradictoria/incompleta impide decisión segura.

---

# 254. NOT_DETECTED ≠ impossible

Solo significa:

```text
evidence did not satisfy current policy
```

---

# 255. Detection result

```php
final readonly class NPlusOneDetectionResult
{
    public function __construct(
        public NPlusOneDecision $decision,
        public ?NPlusOneDetection $detection,
        public NPlusOneEvaluationSummary $summary,
    ) {}
}
```

---

# 256. False positives prioritization

La arquitectura deberá favorecer:

```text
high-confidence useful diagnostics
```

sobre:

```text
maximum number of warnings
```

---

# 257. Developer trust

Un detector que genera falsos positivos constantemente termina siendo deshabilitado.

Por tanto:

> **La precisión diagnóstica es parte de la experiencia de desarrollo.**

---

# 258. False negative policy

En modo:

```text
development/testing
```

podrá reducirse threshold para detectar candidatos más agresivamente.

---

# 259. Production policy

Preferirá:

```text
lower overhead
higher confidence
sampling
aggregation
```

---

# 260. Environment policy

```text
development
testing
staging
production
```

podrán usar políticas distintas.

---

# 261. Configuración conceptual

```php
return [

    'telemetry' => [

        'n_plus_one' => [

            'enabled' => true,

            'minimum_queries' => 3,

            'minimum_parents' => 3,

            'minimum_confidence' => 'medium',

            'generic_detection' => true,

            'source_location' => false,

            'sampling' => [
                'enabled' => true,
            ],

            'budgets' => [
                'max_candidates' => 128,
                'max_samples_per_candidate' => 3,
            ],

            'testing' => [
                'fail_on_detection' => false,
            ],

        ],

    ],

];
```

Valores ilustrativos.

---

# 262. Directory structure

```text
src/Quantum/Database/Telemetry/NPlusOne/
│
├── Contract/
│   ├── NPlusOneDetector.php
│   ├── IncrementalNPlusOneDetector.php
│   ├── NPlusOneDetectionRule.php
│   ├── NPlusOneRecommendationProvider.php
│   └── NPlusOneTelemetryCollector.php
│
├── Detection/
│   ├── NPlusOneDetection.php
│   ├── NPlusOneDetectionId.php
│   ├── NPlusOneDetectionResult.php
│   ├── NPlusOneDecision.php
│   ├── NPlusOneConfidence.php
│   ├── NPlusOneSeverity.php
│   └── NPlusOneCandidate.php
│
├── Observation/
│   ├── NPlusOneObservation.php
│   ├── NPlusOneObservationSet.php
│   ├── RootQueryObservation.php
│   ├── EntityHydratedObservation.php
│   ├── RelationshipAccessObservation.php
│   ├── RelationshipLoadObservation.php
│   ├── DependentQueryObservation.php
│   ├── BatchLoadObservation.php
│   └── CacheHitObservation.php
│
├── Correlation/
│   ├── RootOperationId.php
│   ├── RelationshipLoadId.php
│   ├── ParentIdentityToken.php
│   ├── NPlusOneCorrelationEngine.php
│   ├── RelationshipCorrelationIndex.php
│   └── QueryCorrelationIndex.php
│
├── Evidence/
│   ├── NPlusOneEvidence.php
│   ├── NPlusOneCoverage.php
│   ├── NPlusOneTemporalEvidence.php
│   ├── NPlusOneParameterEvidence.php
│   └── NPlusOneRuleEvidence.php
│
├── Rule/
│   ├── RelationshipCorrelationRule.php
│   ├── ParentIdentityCorrelationRule.php
│   ├── FingerprintCorrelationRule.php
│   ├── TemporalCorrelationRule.php
│   ├── BatchExclusionRule.php
│   ├── DistributedFanOutExclusionRule.php
│   ├── CacheResolutionRule.php
│   └── SourceCorrelationRule.php
│
├── Classification/
│   ├── QueryAmplificationType.php
│   ├── NPlusOneClassifier.php
│   └── NPlusOneSeverityResolver.php
│
├── Graph/
│   ├── NPlusOneGraph.php
│   ├── NPlusOneNode.php
│   └── NPlusOneGraphBuilder.php
│
├── Profile/
│   ├── NPlusOneAggregateProfile.php
│   └── NPlusOneCostCalculator.php
│
├── Policy/
│   ├── NPlusOnePolicy.php
│   ├── CompiledNPlusOnePolicy.php
│   ├── NPlusOnePolicyResolver.php
│   ├── NPlusOneSeverityPolicy.php
│   ├── NPlusOneSamplingPolicy.php
│   └── NPlusOneDiagnosticPolicy.php
│
├── Diagnostic/
│   ├── NPlusOneDiagnostic.php
│   ├── NPlusOneDiagnosticBuilder.php
│   ├── NPlusOneDiagnosticSummary.php
│   ├── NPlusOneEvidenceView.php
│   ├── NPlusOneSourceContext.php
│   ├── NPlusOneRecommendation.php
│   └── NPlusOneRecommendationSet.php
│
├── Suppression/
│   ├── NPlusOneSuppression.php
│   ├── NPlusOneSuppressionRuleId.php
│   └── NPlusOneSuppressionResolver.php
│
├── Telemetry/
│   ├── NPlusOneDetected.php
│   ├── NPlusOneMetricRecorder.php
│   └── NPlusOneTraceEnricher.php
│
├── Runtime/
│   ├── NPlusOneRuntimeContext.php
│   ├── NPlusOneObservationBudget.php
│   └── NPlusOneRuntimeResetter.php
│
├── Testing/
│   ├── NPlusOneAssertions.php
│   ├── RecordingNPlusOneDetector.php
│   └── FakeNPlusOneTelemetryCollector.php
│
└── Exception/
    ├── NPlusOneTelemetryException.php
    ├── NPlusOneDetectionException.php
    ├── NPlusOneCorrelationException.php
    ├── NPlusOnePolicyException.php
    ├── NPlusOneDiagnosticException.php
    ├── NPlusOneBudgetException.php
    └── NPlusOneExtensionException.php
```

---

# 263. Flujo de detección ORM

```text
Query Root
   │
   ▼
QueryTelemetry
   │
   ▼
Hydration
   │
   ├── Parent A
   ├── Parent B
   ├── Parent C
   └── Parent D
         │
         ▼
Relationship Access
         │
         ▼
RelationshipLoader
         │
         ▼
Dependent Query
         │
         ▼
QueryTelemetry
         │
         ▼
NPlusOneObservation
         │
         ▼
Correlation Engine
         │
         ▼
Candidate
         │
         ▼
Detection Rules
         │
         ▼
Confidence
         │
         ▼
Severity
         │
         ▼
Diagnostic
```

---

# 264. Flujo de exclusión por batch

```text
100 Parent Entities
       │
       ▼
Relationship Accesses
       │
       ▼
Batch Loader
       │
       ▼
1–N bounded batch queries
       │
       ▼
BatchLoadObservation
       │
       ▼
N+1 Candidate Evaluator
       │
       ▼
BatchExclusionRule
       │
       ▼
NOT_DETECTED
```

---

# 265. Flujo raw/query loop

```text
Application Loop
      │
      ├── Query F(param A)
      ├── Query F(param B)
      ├── Query F(param C)
      └── Query F(param D)
              │
              ▼
      Generic Pattern Detector
              │
              ├── same fingerprint
              ├── parameter variation
              ├── same operation
              ├── temporal correlation
              └── same call site
              │
              ▼
          SUSPECTED
```

Sin Relationship Metadata:

```text
confidence < semantic ORM detector
```

por default.

---

# 266. Architectural invariants

## DB-N1-001
Repeated Query será distinto de N+1.

## DB-N1-002
High Query Count será distinto de N+1.

## DB-N1-003
Lazy Loading será distinto de N+1.

## DB-N1-004
Relationship Loading será distinto de N+1.

## DB-N1-005
N+1 Detection será distinto de Automatic Eager Loading.

## DB-N1-006
N+1 Detection será distinto de Query Profiling.

## DB-N1-007
N+1 Detection será distinto de Slow Query Detection.

## DB-N1-008
N+1 Detection será distinto de Query Optimization.

## DB-N1-009
N+1 Detection será distinta de Query Execution.

## DB-N1-010
N+1 Detector no generará SQL.

## DB-N1-011
N+1 Detector no ejecutará SQL.

## DB-N1-012
N+1 Detector no modificará Query AST.

## DB-N1-013
N+1 Detector no modificará Query Plan.

## DB-N1-014
N+1 Detector no modificará EntityManager.

## DB-N1-015
N+1 Detector no modificará UnitOfWork.

## DB-N1-016
N+1 Detector no modificará IdentityMap.

## DB-N1-017
N+1 Detector no modificará relationship state.

## DB-N1-018
N+1 Detector no forzará eager loading.

## DB-N1-019
N+1 Detector no creará JOINs.

## DB-N1-020
N+1 Detector no hará batch automáticamente.

## DB-N1-021
Root Operation será distinta de Query.

## DB-N1-022
Request será distinto de Root Operation.

## DB-N1-023
Transaction será distinta de Root Operation.

## DB-N1-024
Relationship Access será distinto de Dependent Query.

## DB-N1-025
Logical Relationship Load será distinto de Physical Query.

## DB-N1-026
Una Relationship Load podrá generar múltiples queries físicas.

## DB-N1-027
Query fingerprint solo no probará N+1.

## DB-N1-028
Relationship correlation aumentará confidence.

## DB-N1-029
Parent identity correlation aumentará confidence.

## DB-N1-030
Load strategy será evidencia explícita.

## DB-N1-031
Temporal locality será evidencia, no prueba.

## DB-N1-032
Sequential execution no será requisito para N+1.

## DB-N1-033
Concurrent fan-out podrá representar N+1.

## DB-N1-034
Distributed fan-out será distinto de N+1.

## DB-N1-035
Batch loading será reconocido.

## DB-N1-036
Batch query no será contada como múltiples N+1 por cada parent.

## DB-N1-037
IdentityMap hit no será dependent DB query.

## DB-N1-038
Entity Cache hit no será dependent DB query.

## DB-N1-039
Loaded relationship no generará query ficticia.

## DB-N1-040
Partial relationship será distinta de unloaded relationship.

## DB-N1-041
Polymorphic relationships serán soportadas.

## DB-N1-042
Target entity type podrá participar en correlación polimórfica.

## DB-N1-043
Composite identifiers serán soportados.

## DB-N1-044
Parent identity tokens no expondrán IDs reales por default.

## DB-N1-045
Parent identity tokens serán scoped.

## DB-N1-046
Cross-tenant correlation estará prohibida por default.

## DB-N1-047
Shard context será preservado.

## DB-N1-048
Different shards no se mezclarán incorrectamente.

## DB-N1-049
N+1 no requerirá dependentQueryCount = parentCount.

## DB-N1-050
Conditional relationship access será soportado.

## DB-N1-051
Query amplification será medible.

## DB-N1-052
Query amplification será distinta de severity.

## DB-N1-053
Aggregate cost será medible.

## DB-N1-054
Aggregate cost evitará double counting.

## DB-N1-055
Severity será policy-driven.

## DB-N1-056
Confidence será distinta de severity.

## DB-N1-057
Coverage será distinta de confidence.

## DB-N1-058
UNKNOWN coverage no será COMPLETE.

## DB-N1-059
Missing evidence no demostrará ausencia de N+1.

## DB-N1-060
Candidate será distinto de Detection.

## DB-N1-061
SUSPECTED será distinto de DETECTED.

## DB-N1-062
INCONCLUSIVE será distinto de NOT_DETECTED.

## DB-N1-063
NOT_DETECTED no significará impossible.

## DB-N1-064
Semantic detector será preferido sobre heuristic detector.

## DB-N1-065
Raw SQL podrá ser analizado heurísticamente.

## DB-N1-066
Heuristic detection tendrá confidence apropiada.

## DB-N1-067
Raw parameter values no serán necesarios.

## DB-N1-068
Parameter evidence podrá utilizar tokens seguros.

## DB-N1-069
N+1 correlation scope será bounded.

## DB-N1-070
Cross-request repetition no será request-level N+1 confirmado.

## DB-N1-071
Request-local state será scoped.

## DB-N1-072
No habrá mutable static N+1 registry.

## DB-N1-073
Persistent runtime reseteará observations.

## DB-N1-074
Persistent runtime reseteará candidates.

## DB-N1-075
Persistent runtime reseteará identity tokens.

## DB-N1-076
Fiber/coroutine state será aislado.

## DB-N1-077
Detector no retendrá Entity objects.

## DB-N1-078
Detector no retendrá EntityManager.

## DB-N1-079
Detector no retendrá UnitOfWork.

## DB-N1-080
Detector no retendrá Connection.

## DB-N1-081
Observation memory será bounded.

## DB-N1-082
Candidate memory será bounded.

## DB-N1-083
Sample count será bounded.

## DB-N1-084
Source samples serán bounded.

## DB-N1-085
Budget exhaustion no romperá la aplicación.

## DB-N1-086
Budget exhaustion reducirá coverage cuando corresponda.

## DB-N1-087
Approximate cardinality será marcada como aproximada.

## DB-N1-088
Telemetry failure no romperá negocio por default.

## DB-N1-089
Testing podrá activar strict mode.

## DB-N1-090
Strict mode será explícito.

## DB-N1-091
Suppression será distinta de absence.

## DB-N1-092
Suppression deberá ser explicable.

## DB-N1-093
Deduplication agregará detecciones equivalentes.

## DB-N1-094
Una relación N+1 producirá un diagnostic agregado, no uno por query.

## DB-N1-095
Nested N+1 podrá representarse como graph.

## DB-N1-096
NPlusOneGraph no será Entity Graph.

## DB-N1-097
Nested diagnostics evitarán redundancia excesiva.

## DB-N1-098
Source location será opcional.

## DB-N1-099
Source location intentará apuntar al application call site.

## DB-N1-100
Stack trace completa no será capturada en cada query.

## DB-N1-101
Source capture será policy-driven.

## DB-N1-102
Recommendation será distinta de detection.

## DB-N1-103
Recommendation no modificará la aplicación.

## DB-N1-104
Recommendation no asumirá JOIN como solución universal.

## DB-N1-105
Relationship Planner conservará autoridad sobre estrategia.

## DB-N1-106
Eager loading será distinto de JOIN.

## DB-N1-107
Batch loading podrá ser recomendado.

## DB-N1-108
Select-in podrá ser recomendado cuando sea compatible.

## DB-N1-109
Recommendation respetará PlatformCapabilities.

## DB-N1-110
Detector no usará vendor conditionals como arquitectura principal.

## DB-N1-111
MySQL y MariaDB podrán tener capabilities diferentes.

## DB-N1-112
Version no será equivalente a capability.

## DB-N1-113
Slow Query Detection podrá coexistir con N+1 Detection.

## DB-N1-114
Fast queries podrán formar un N+1 costoso.

## DB-N1-115
Slow queries podrán formar un N+1.

## DB-N1-116
Slow query diagnostics no sustituirán N+1 diagnostics.

## DB-N1-117
Cache podrá reducir dependent query count.

## DB-N1-118
Cache hit no será DB query.

## DB-N1-119
Cache-masked amplification podrá diagnosticarse separadamente.

## DB-N1-120
Potential N+1 será distinto de Database N+1.

## DB-N1-121
Replica role será contexto.

## DB-N1-122
Replica lag será contexto independiente.

## DB-N1-123
Sticky routing será contexto independiente.

## DB-N1-124
Transaction context será preservado.

## DB-N1-125
Locking reads requerirán recomendaciones cuidadosas.

## DB-N1-126
Batching no deberá cambiar lock semantics automáticamente.

## DB-N1-127
Sharded N+1 será soportado conceptualmente.

## DB-N1-128
Logical dependent load será distinto de physical shard query.

## DB-N1-129
DistributedExecutionId evitará falsos positivos de fan-out.

## DB-N1-130
Pagination podrá contener N+1.

## DB-N1-131
Cursor pagination podrá contener N+1.

## DB-N1-132
Chunk processing podrá contener N+1.

## DB-N1-133
LazyCollection podrá contener N+1.

## DB-N1-134
Streaming detection deberá ser incremental.

## DB-N1-135
Lazy traversal detection deberá ser bounded.

## DB-N1-136
Bulk operations no serán N+1 por definición.

## DB-N1-137
Import per-record lookups podrán clasificarse como amplification.

## DB-N1-138
Query Amplification será una categoría más amplia que ORM N+1.

## DB-N1-139
V1 priorizará ORM N+1 de alta confianza.

## DB-N1-140
Metrics usarán labels bounded.

## DB-N1-141
RelationshipId no será metric label global por default.

## DB-N1-142
EntityClass no será metric label global por default.

## DB-N1-143
Raw SQL no será metric label.

## DB-N1-144
Tenant ID no será metric label.

## DB-N1-145
Query ID no será metric label.

## DB-N1-146
Request ID no será metric label.

## DB-N1-147
Specific relationship details pertenecerán a logs/traces/diagnostics.

## DB-N1-148
Trace annotations serán bounded.

## DB-N1-149
Potential saved query count será estimación.

## DB-N1-150
Potential saved query count no será garantía.

## DB-N1-151
Parameter limits podrán requerir múltiples batches.

## DB-N1-152
Detector será independiente de HTTP.

## DB-N1-153
Detector será independiente de CLI.

## DB-N1-154
Detector será independiente de Debug Toolbar.

## DB-N1-155
Detector será independiente de alert vendors.

## DB-N1-156
Diagnostic payload será redactado.

## DB-N1-157
Entity identifiers no serán expuestos por default.

## DB-N1-158
Sensitive parameters no serán expuestos.

## DB-N1-159
Telemetry failure nunca deberá provocar fuga de datos.

## DB-N1-160
Unsafe diagnostic detail deberá descartarse.

## DB-N1-161
Detection rules serán extensibles.

## DB-N1-162
Extension rules no podrán alterar Query Execution.

## DB-N1-163
Rule decisions serán explicables.

## DB-N1-164
Rule evidence será preservada.

## DB-N1-165
Deterministic confidence será suficiente para V1.

## DB-N1-166
Machine learning no será requisito.

## DB-N1-167
Developer trust será objetivo arquitectónico.

## DB-N1-168
False positives deberán minimizarse.

## DB-N1-169
Development podrá usar policy más sensible.

## DB-N1-170
Production podrá usar policy más conservadora.

## DB-N1-171
Configuration será compilable.

## DB-N1-172
Configuration no se reinterpretará completamente por query.

## DB-N1-173
Detection será deterministic bajo misma evidence/policy.

## DB-N1-174
NPlusOneDetection será immutable una vez finalizada.

## DB-N1-175
DetectionId no será metric label.

## DB-N1-176
RootOperationId no será metric label.

## DB-N1-177
RelationshipLoadId no será metric label.

## DB-N1-178
ParentIdentityToken no será persistido como user identifier.

## DB-N1-179
N+1 Telemetry será una capa observacional.

## DB-N1-180
N+1 Telemetry nunca se convertirá en un segundo Relationship Loading Engine.

---

# 267. Modelo formal

Sea una operación raíz:

```text
R
```

que produce entidades:

```text
E(R) = {e₁, e₂, ..., eₙ}
```

Para una relación:

```text
L
```

definimos:

```text
A(L, eᵢ)
```

como acceso a `L` sobre `eᵢ`.

---

# 268. Query dependiente

Si el acceso produce una query:

```text
Q(L, eᵢ)
```

entonces el conjunto de queries dependientes es:

```text
D(R,L)
=
{ Q(L,eᵢ) | eᵢ ∈ E(R) }
```

---

# 269. Forma semántica

Existe una función:

```text
F(Q)
```

que produce el fingerprint semántico.

Para un patrón típico:

```text
F(Q(L,e₁))
=
F(Q(L,e₂))
=
...
=
F(Q(L,eₙ))
```

mientras cambian los bindings relacionados con el parent.

---

# 270. Evidencia fuerte

Conceptualmente:

```text
NPlusOne(R,L)
```

tiene evidencia fuerte cuando:

```text
|E(R)| ≥ ParentThreshold

∧

|D(R,L)| ≥ QueryThreshold

∧

DistinctParents(D) ≈ |D(R,L)|

∧

SameSemanticFingerprint(D)

∧

RelationshipCorrelated(D,L)

∧

¬ BatchLoad(D)

∧

¬ DistributedFanOut(D)
```

---

# 271. No igualdad estricta

No exigiremos:

```text
|D(R,L)| = |E(R)|
```

porque:

```text
conditional access
cache hits
IdentityMap
already-loaded relationships
```

pueden reducir el número.

---

# 272. Amplificación

Definimos:

```text
Aq =
|D(R,L)|
/
max(1, RootQueryCount)
```

como amplificación de queries.

---

# 273. Parent coverage

```text
Cp =
DistinctParentsWithDependentQuery
/
max(1, RootParentCount)
```

---

# 274. Cost amplification

```text
Ca =
Σ Cost(Qᵢ)
```

para todas las queries dependientes atribuibles.

---

# 275. Confidence

Conceptualmente:

```text
Confidence =
f(
  relationship correlation,
  parent correlation,
  fingerprint similarity,
  temporal/operation correlation,
  batch exclusion,
  distributed fan-out exclusion,
  telemetry coverage
)
```

---

# 276. Filosofía

VoltStack deberá priorizar:

```text
Semantic evidence
        ↓
Correlation
        ↓
Classification
        ↓
Cost analysis
        ↓
Explanation
        ↓
Recommendation
```

sobre:

```text
Count queries
↓
Guess N+1
```

---

# 277. Regla maestra

> **VoltStack considerará N+1 a un patrón de amplificación dependiente cuando múltiples consultas puedan correlacionarse semánticamente con accesos individuales sobre entidades producidas por una misma operación raíz; la repetición de SQL, el lazy loading, el número elevado de queries o la proximidad temporal serán evidencia auxiliar, nunca prueba suficiente por sí solos.**

Arquitectura final:

```text
Root Operation
      ↓
Root Query
      ↓
Hydrated Parents
      ↓
Relationship Access
      ↓
Relationship Load
      ↓
Dependent Queries
      ↓
Telemetry Observations
      ↓
Correlation Engine
      ↓
Candidate
      ↓
Detection Rules
      ↓
Confidence
      ↓
Severity / Cost
      ↓
Diagnostic
      ↓
Recommendation
```

manteniendo:

```text
Detection
≠
Optimization

Recommendation
≠
Automatic Rewrite

Physical Fan-Out
≠
N+1

Repeated Query
≠
N+1
```

---

# 278. Estado del Bloque 21

```text
BLOCK 21 — TELEMETRY AND DEBUGGING

✓ 216_DATABASE_TELEMETRY_ARCHITECTURE.md
✓ 217_DATABASE_QUERY_TELEMETRY_SYSTEM.md
✓ 218_DATABASE_CONNECTION_TELEMETRY_SYSTEM.md
✓ 219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
✓ 220_DATABASE_ORM_TELEMETRY_SYSTEM.md
✓ 221_DATABASE_QUERY_PROFILER_SYSTEM.md
✓ 222_DATABASE_SLOW_QUERY_DETECTION_SYSTEM.md
✓ 223_DATABASE_N_PLUS_ONE_TELEMETRY_SYSTEM.md
○ 224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
○ 225_DATABASE_DEVELOPER_DEBUG_TOOLBAR_INTEGRATION.md
```

---

# 279. Siguiente documento

```text
224_DATABASE_DEBUG_INFORMATION_SYSTEM.md
```

El siguiente documento deberá definir la representación diagnóstica central mediante la cual VoltStack Database podrá transformar información proveniente de:

```text
Query Telemetry
Connection Telemetry
Transaction Telemetry
ORM Telemetry
Query Profiler
Slow Query Detection
N+1 Detection
Schema
Cache
Distribution
Runtime
```

en un modelo de debugging:

```text
seguro
estructurado
inmutable
redactado
serializable
bounded
provider-agnostic
```

sin exponer objetos runtime como:

```text
Connection
PDO
EntityManager
UnitOfWork
IdentityMap
Entity
ResultCursor
TransactionContext mutable
```

y preservando como regla central:

> **Debug Information será una representación diagnóstica derivada del estado observado, nunca el propio estado vivo del Database Runtime.**