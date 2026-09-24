# 153_DATABASE_BATCH_RELATION_LOADING_SYSTEM.md

# VoltStack Quantum Database
## Database Batch Relation Loading System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 153 — Database Batch Relation Loading System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Batch Relation Loading System` define la infraestructura ORM responsable de agrupar múltiples necesidades compatibles de carga de relaciones y resolverlas mediante un número acotado de operaciones contra la base de datos.

Problema clásico:

```php
$users = User::query()->limit(100)->get();

foreach ($users as $user) {
    foreach ($user->posts as $post) {
        // potencialmente 100 cargas independientes
    }
}
```

Sin batching:

```text
Load Users
   │
   ├── User#1.posts  → Query 1
   ├── User#2.posts  → Query 2
   ├── User#3.posts  → Query 3
   │
   └── User#100.posts → Query 100
```

Con Batch Relation Loading:

```text
User#1.posts
User#2.posts
User#3.posts
...
User#100.posts
      │
      ▼
Batch Relation Loading
      │
      ▼
Load posts for owners [1..100]
      │
      ▼
Distribute Results
      │
      ├── User#1 → [...]
      ├── User#2 → [...]
      └── User#100 → [...]
```

El objetivo no es simplemente producir un `WHERE IN`.

El sistema debe preservar:

- identidad canónica;
- `PersistenceContext`;
- `IdentityMap`;
- metadata de relación;
- tenant;
- shard;
- transacción;
- `ReadIntent`;
- scopes;
- ordering;
- filtros;
- cobertura;
- snapshots;
- pending mutations;
- cancelación;
- deadlines;
- límites del driver;
- límites de memoria;
- semántica de cada relación.

---

# 2. Regla central

> **Batch Relation Loading en VoltStack agrupará necesidades de carga compatibles para reducir round trips sin fusionar contextos, identidades, tenants, shards, transacciones, scopes o garantías semánticas que deban permanecer aisladas.**

Formalmente:

```text
BatchRelationLoading
=
CompatibleLoadRequirements
+
SafeGrouping
+
BoundedQueryPlanning
+
RelationshipExecution
+
IdentityReconciliation
+
OwnerDistribution
+
RelationshipAssembly
+
CoverageFinalization
```

No:

```text
BatchRelationLoading
=
CollectAllIDs
+
WHERE IN (...)
```

---

# 3. Objetivos

El sistema deberá:

1. reducir round trips;
2. mitigar N+1;
3. soportar eager loading;
4. soportar lazy batching;
5. soportar cargas explícitas;
6. soportar to-one;
7. soportar to-many;
8. soportar many-to-many;
9. soportar relaciones polimórficas;
10. soportar claves compuestas;
11. dividir batches demasiado grandes;
12. respetar límites de parámetros;
13. preservar IdentityMap;
14. preservar snapshots;
15. reconciliar pending mutations;
16. aislar tenants y shards;
17. respetar transacciones;
18. soportar cancelación;
19. soportar deadlines;
20. controlar memoria;
21. producir telemetría;
22. ser seguro en runtimes persistentes.

---

# 4. Batch Loading ≠ Eager Loading

Eager Loading describe:

> qué relaciones deben estar disponibles junto con una consulta raíz.

Batch Loading describe:

> cómo múltiples necesidades compatibles pueden resolverse conjuntamente.

Por tanto:

```text
Eager Loading
        │
        ▼
Relationship Load Requirements
        │
        ▼
Batch Relation Loading
```

pero también:

```text
Lazy Loading
        │
        ▼
Deferred Load Requirements
        │
        ▼
Batch Relation Loading
```

---

# 5. Batch Loading ≠ Lazy Loading

Lazy Loading decide **cuándo** cargar.

Batch Loading decide **qué cargas compatibles pueden combinarse**.

```text
Lazy:
    timing policy

Batch:
    grouping/execution strategy
```

---

# 6. Batch Loading ≠ N+1 Detection

`N+1 Detection` observa un patrón problemático.

`Batch Relation Loading` puede evitarlo o mitigarlo.

```text
Detection
≠
Optimization
```

---

# 7. Batch Loading ≠ Query Optimizer

El ORM puede decidir:

```text
owners A,B,C require relation R
```

El Query Optimizer decide posteriormente cómo optimizar el Query Model correspondiente.

---

# 8. Batch Loading ≠ SQL Compiler

El Batch Loader nunca construirá:

```sql
SELECT *
FROM posts
WHERE user_id IN (...)
```

como string.

Generará requerimientos/query models semánticos.

---

# 9. Posición arquitectónica

```text
ORM
│
├── EntityManager
├── IdentityMap
├── UnitOfWork
│
└── Relationship System
    │
    ├── Metadata
    ├── Persistence
    ├── Loading
    │   │
    │   ├── Eager Loading
    │   ├── Lazy Loading
    │   ├── Batch Relation Loading   ← este documento
    │   └── N+1 Detection
    │
    └── PersistentCollection
```

---

# 10. Pipeline general

```text
Relationship Requirements
          │
          ▼
Batch Candidate Collector
          │
          ▼
Compatibility Classifier
          │
          ▼
Batch Grouper
          │
          ▼
Batch Planner
          │
          ▼
Chunk Planner
          │
          ▼
Relationship Loading System
          │
          ▼
Entity Query
          │
          ▼
Query Engine
          │
          ▼
Execution Engine
          │
          ▼
Hydration
          │
          ▼
IdentityMap
          │
          ▼
Result Distributor
          │
          ▼
Relationship Assembly
          │
          ▼
Coverage / Snapshot Reconciliation
```

---

# 11. RelationshipLoadRequirement

La unidad lógica inicial será un requerimiento.

```php
final readonly class RelationshipLoadRequirement
{
    public function __construct(
        public RelationshipId $relationship,
        public EntityKey $owner,
        public RelationshipKnowledgeRequirement $requiredKnowledge,
        public RelationshipLoadContext $context,
    ) {}
}
```

---

# 12. Requirement ≠ Query

El requirement expresa:

```text
User#10 necesita User.posts COMPLETE
```

No:

```text
SELECT posts ...
```

---

# 13. Requirement ≠ Batch

Un requirement individual puede:

- satisfacerse desde memoria;
- resolverse mediante IdentityMap;
- agruparse;
- dividirse;
- ser rechazado;
- no requerir query.

---

# 14. Batch Candidate

Después de evaluar estado actual:

```text
RequiredKnowledge
-
CurrentKnowledge
=
MissingKnowledge
```

solo necesidades faltantes entran al batching.

---

# 15. Batch candidate elimination

Ejemplo:

```text
User#1.posts = COMPLETE
User#2.posts = UNINITIALIZED
User#3.posts = COMPLETE
```

La petición:

```text
load posts for [1,2,3]
```

produce:

```text
batch candidates = [User#2]
```

---

# 16. BatchKey

La agrupación requiere una clave semántica.

```php
final readonly class RelationBatchKey
{
    public function __construct(
        public RelationshipId $relationship,
        public PersistenceContextId $persistenceContext,
        public DatabaseContextKey $database,
        public TenantContextKey $tenant,
        public ShardContextKey $shard,
        public TransactionContextKey $transaction,
        public ReadIntent $readIntent,
        public RelationshipLoadShape $shape,
        public ScopeFingerprint $scope,
    ) {}
}
```

---

# 17. Regla de compatibilidad

Dos requirements:

```text
A
B
```

pueden agruparse solamente si:

```text
BatchKey(A) = BatchKey(B)
```

o existe una regla explícita que demuestre compatibilidad semántica.

---

# 18. BatchKey ≠ EntityKey

`EntityKey` identifica una entidad.

`BatchKey` identifica una clase compatible de operaciones de carga.

---

# 19. RelationshipId

La misma propiedad textual en entidades diferentes no implica misma relación.

```text
User.posts
Category.posts
```

son relaciones distintas.

---

# 20. PersistenceContext isolation

Nunca agrupar:

```text
EntityManager A
EntityManager B
```

aunque consulten la misma relación física.

---

# 21. Database context isolation

Nunca combinar requirements dirigidos a bases lógicas distintas.

---

# 22. Tenant isolation

```text
Tenant A User#10
Tenant B User#10
```

deben pertenecer a batches distintos.

---

# 23. Shard isolation

```text
Shard 1
Shard 2
```

no se combinan mediante un batch SQL convencional.

---

# 24. Transaction isolation

Una carga dentro de:

```text
Transaction T1
```

no se combinará con otra fuera de T1 si eso altera consistencia.

---

# 25. ReadIntent compatibility

No agrupar:

```text
PRIMARY_REQUIRED
```

con:

```text
REPLICA_ALLOWED
```

si la combinación degrada garantías.

---

# 26. Scope compatibility

Scopes distintos producen batches distintos.

Ejemplo:

```text
User.posts where published = true
```

vs:

```text
User.posts including drafts
```

---

# 27. Ordering compatibility

Dos cargas pueden compartir query solo si el resultado puede distribuirse preservando ordering requerido.

---

# 28. Projection compatibility

```text
full entity
```

y:

```text
partial projection
```

no deben fusionarse ciegamente.

---

# 29. RelationshipLoadShape

```php
final readonly class RelationshipLoadShape
{
    public function __construct(
        public RelationshipCardinality $cardinality,
        public RelationshipCoverageRequirement $coverage,
        public ?OrderingDescriptor $ordering,
        public ?FilterDescriptor $filter,
        public ?ProjectionDescriptor $projection,
    ) {}
}
```

---

# 30. Batch lifecycle

```text
COLLECTING
    │
    ▼
SEALED
    │
    ▼
PLANNED
    │
    ▼
EXECUTING
    │
    ▼
ASSEMBLING
    │
    ▼
COMPLETED
```

Errores:

```text
FAILED
CANCELLED
UNCERTAIN
```

---

# 31. Batch state

```php
enum RelationBatchState
{
    case COLLECTING;
    case SEALED;
    case PLANNED;
    case EXECUTING;
    case ASSEMBLING;
    case COMPLETED;
    case FAILED;
    case CANCELLED;
    case UNCERTAIN;
}
```

---

# 32. COLLECTING

Se aceptan nuevos requirements compatibles.

---

# 33. SEALED

No se agregan nuevos owners al batch actual.

Esto permite planificación determinista.

---

# 34. PLANNED

El batch ya fue transformado en uno o más chunks físicos/lógicos.

---

# 35. EXECUTING

Existe ejecución activa.

---

# 36. ASSEMBLING

Los resultados se distribuyen a owners y relaciones.

---

# 37. COMPLETED

Todos los requirements del batch alcanzaron el conocimiento requerido.

---

# 38. FAILED

Existe falla conocida.

No implica necesariamente que ningún owner haya obtenido información útil.

---

# 39. CANCELLED

La operación fue cancelada.

---

# 40. UNCERTAIN

No existe evidencia suficiente para afirmar el resultado completo.

---

# 41. Batch Planner

```php
interface RelationBatchPlanner
{
    public function plan(
        RelationBatch $batch,
        PlatformCapabilities $capabilities,
    ): RelationBatchPlan;
}
```

---

# 42. BatchPlan

```php
final readonly class RelationBatchPlan
{
    /**
     * @param list<RelationBatchChunk> $chunks
     */
    public function __construct(
        public RelationBatchKey $key,
        public array $chunks,
        public BatchAssemblyPlan $assembly,
    ) {}
}
```

---

# 43. Batch Plan ≠ Query Plan

`RelationBatchPlan` pertenece al ORM.

`Query Plan` pertenece al Query Engine.

```text
RelationBatchPlan
      │
      ▼
one or more Entity Queries
      │
      ▼
Query Planner
```

---

# 44. Batch Chunk

Un batch puede necesitar dividirse.

```text
Batch 10,000 owners
        │
        ▼
Chunk 1: 1..500
Chunk 2: 501..1000
...
```

---

# 45. Chunking ≠ pagination

Chunking divide una operación interna por restricciones.

No cambia la semántica del resultado lógico.

---

# 46. Razones para chunking

- límite de parámetros;
- packet size;
- SQL expression limits;
- memoria;
- latency;
- driver constraints;
- plataforma;
- composite keys;
- resource policy.

---

# 47. Platform capabilities

El Batch Planner deberá consultar capabilities.

Ejemplos conceptuales:

```php
$capabilities->maxBindParameters();
$capabilities->supportsRowValueIn();
$capabilities->supportsArrayParameters();
$capabilities->supportsValuesTable();
```

---

# 48. Version ≠ Capability

No:

```php
if ($driver === 'pgsql') {
    ...
}
```

en el ORM.

Preferir:

```php
if ($capabilities->supportsRowValueIn()) {
    ...
}
```

---

# 49. Parameter budget

Supongamos:

```text
MaxBindParameters = 999
```

y cada owner usa una clave simple.

El chunk no deberá exceder el presupuesto reservado.

---

# 50. Reserved parameters

Una query puede necesitar parámetros adicionales:

```text
tenant
scope
filters
soft-delete
authorization
```

Por tanto:

```text
OwnerParameterBudget
=
MaxBindParameters
-
ReservedQueryParameters
-
SafetyMargin
```

---

# 51. Composite identifiers

Si cada owner requiere:

```text
(country_id, customer_id)
```

entonces:

```text
ParametersPerOwner = 2
```

y:

```text
MaxOwnersPerChunk
≈
floor(
    OwnerParameterBudget
    /
    ParametersPerOwner
)
```

---

# 52. Parameter formula

General:

```text
ChunkCapacity
=
floor(
    AvailableBindBudget
    /
    BindCostPerOwner
)
```

---

# 53. Query size policy

El límite de parámetros no será el único límite.

También puede existir:

```text
maxOwnersPerBatch
maxEstimatedSQLSize
maxExpectedRows
maxMemoryBytes
```

---

# 54. BatchLoadBudget

```php
final readonly class BatchLoadBudget
{
    public function __construct(
        public int $maxOwnersPerBatch,
        public int $maxOwnersPerChunk,
        public int $maxChunks,
        public int $maxBindParameters,
        public int $maxExpectedRows,
        public int $maxBufferedBytes,
    ) {}
}
```

---

# 55. Bounded batching

VoltStack no deberá convertir:

```text
N+1
```

en:

```text
one gigantic query that exhausts the database
```

---

# 56. Batch size strategy

Posibles estrategias:

```php
enum BatchSizeStrategy
{
    case FIXED;
    case CAPABILITY_AWARE;
    case ADAPTIVE;
}
```

---

# 57. V1 recommendation

```text
CAPABILITY_AWARE
```

con límites configurables.

---

# 58. Adaptive batching

Podrá incorporarse posteriormente usando:

- historical latency;
- row cardinality;
- memory pressure;
- query complexity.

Pero no debe alterar garantías semánticas.

---

# 59. To-One Batch Loading

Ejemplo:

```text
Post#1.author → User#10
Post#2.author → User#20
Post#3.author → User#10
```

---

# 60. Identity extraction

Primero se obtienen target keys conocidas:

```text
User#10
User#20
User#10
```

---

# 61. Deduplication

El batch real necesita:

```text
User#10
User#20
```

---

# 62. IdentityMap first

Antes de consultar:

```text
User#10 → HIT
User#20 → MISS
```

Solo:

```text
User#20
```

necesita carga.

---

# 63. To-one distribution

Después:

```text
Post#1.author → canonical User#10
Post#2.author → canonical User#20
Post#3.author → canonical User#10
```

---

# 64. Same target identity

Varias relaciones pueden apuntar a la misma instancia canónica.

---

# 65. Null to-one

Si:

```text
Post#4.author_id = NULL
```

no entra al batch de targets.

La relación puede resolverse como ausencia conocida.

---

# 66. Unknown FK

Si `author_id` no fue cargado:

```text
UNKNOWN
```

no se tratará como `NULL`.

Puede requerir otro load plan.

---

# 67. To-Many Batch Loading

Ejemplo:

```text
User#1.posts
User#2.posts
User#3.posts
```

---

# 68. Semantic request

```text
Load Post
where relationship owner ∈ {User#1, User#2, User#3}
```

No SQL.

---

# 69. Distribution key

Cada fila/entidad relacionada debe poder asociarse al owner correcto.

Conceptualmente:

```text
Post#10 → owner User#1
Post#11 → owner User#1
Post#20 → owner User#2
```

---

# 70. OwnerDistributionMap

```php
interface OwnerDistributionMap
{
    public function add(
        EntityKey $owner,
        EntityKey $target,
    ): void;
}
```

---

# 71. Empty owner result

Si User#3 no recibe rows y la query completó con cobertura completa:

```text
User#3.posts = COMPLETE EMPTY
```

---

# 72. No rows ≠ empty if query incomplete

Si hubo cancelación/falla:

```text
no rows observed for User#3
```

no implica:

```text
User#3.posts = empty
```

---

# 73. Ordering

Supongamos:

```php
#[OneToMany(
    target: Post::class,
    orderBy: ['createdAt' => 'DESC']
)]
```

El resultado de cada owner debe preservar ese ordering.

---

# 74. Global result ordering ≠ per-owner ordering

Una query puede retornar:

```text
owner1 postA
owner2 postB
owner1 postC
```

El assembler debe construir correctamente cada colección.

---

# 75. Stable ordering

Si el ordering necesita estabilidad:

```text
created_at DESC
```

puede requerir tie-breaker:

```text
id DESC
```

según metadata/planner.

---

# 76. Many-to-One Batch Loading

Reutiliza el modelo to-one.

Ejemplo:

```text
100 Orders
→ 8 Customers
```

Solo 8 targets únicos deben considerarse.

---

# 77. One-to-One Batch Loading

Puede resolverse:

```text
owners → target lookup
```

manteniendo cardinalidad máxima de uno.

---

# 78. Cardinality violation

Si una one-to-one produce dos targets para el mismo owner:

```text
RelationshipCardinalityViolationException
```

No elegir uno arbitrariamente.

---

# 79. Many-to-Many Batch Loading

Ejemplo:

```text
User#1.roles
User#2.roles
User#3.roles
```

requiere considerar memberships.

---

# 80. Membership representation

Conceptualmente:

```text
Membership(User#1, Role#10)
Membership(User#1, Role#20)
Membership(User#2, Role#20)
```

---

# 81. Many-to-many pipeline

```text
Owners
  │
  ▼
Membership Load
  │
  ▼
Target Entity Keys
  │
  ▼
IdentityMap
  │
  ▼
Missing Target Load
  │
  ▼
Canonical Targets
  │
  ▼
Membership Distribution
```

---

# 82. One physical query not required

El planner puede elegir:

```text
JOIN
```

o:

```text
membership query
+
target query
```

sin cambiar semántica.

---

# 83. Membership deduplication

Duplicados físicos no deberán producir duplicados lógicos cuando collection semantics sea `SET`.

---

# 84. LIST semantics

Si la relación es `LIST`, duplicados/posición pueden ser semánticamente significativos.

---

# 85. Collection semantics

El assembler deberá respetar:

```text
SET
ORDERED_SET
LIST
```

y cualquier semántica soportada explícitamente.

---

# 86. Polymorphic Batch Loading

Caso:

```text
Comment#1.commentable → Post#10
Comment#2.commentable → Video#7
Comment#3.commentable → Post#20
```

---

# 87. Morph resolution

Primero:

```text
(post, 10)
(video, 7)
(post, 20)
```

se convierten mediante `MorphTypeRegistry`.

---

# 88. Polymorphic grouping

```text
Post:
    10
    20

Video:
    7
```

---

# 89. Formula

```text
PolymorphicBatchGroups
=
GroupBy(
    References,
    TargetEntityType
    × DatabaseContext
    × TenantContext
    × ShardContext
    × LoadShape
)
```

---

# 90. Unknown morph alias

Falla antes de intentar cargar targets arbitrarios.

---

# 91. Allowed target validation

El tipo resuelto debe pertenecer al target set permitido por la relación.

---

# 92. Polymorphic batch fanout

Una colección heterogénea puede requerir:

```text
number of queries
≈
number of distinct compatible target groups
```

---

# 93. Fanout governance

No deberá permitirse fanout ilimitado.

---

# 94. MorphBatchBudget

Puede limitar:

```text
maxDistinctTargetTypes
maxQueries
maxOwners
maxReferences
```

---

# 95. Composite owner identifiers

El sistema deberá soportar:

```text
OrderLine(order_id, line_number)
```

como owner.

---

# 96. Composite target identifiers

Igualmente:

```text
LocalizedArticle(article_id, locale)
```

---

# 97. Tuple predicates

La representación física dependerá de capabilities.

Puede ser:

```text
row-value IN
```

o:

```text
OR groups
```

o:

```text
VALUES relation
```

pero esto lo decide Query Engine/Compiler.

---

# 98. ORM responsibility

El ORM solo expresa:

```text
OwnerIdentitySet
```

tipado.

---

# 99. BatchOwnerSet

```php
final readonly class BatchOwnerSet
{
    /**
     * @param list<EntityKey> $owners
     */
    public function __construct(
        public array $owners,
    ) {}
}
```

---

# 100. Deduplication

Owners duplicados deberán canonicalizarse.

```text
[User#1, User#1, User#2]
→
[User#1, User#2]
```

sin perder múltiples consumers.

---

# 101. Consumer map

Un mismo requirement puede tener varios consumidores.

```text
EntityKey
→ list<RelationshipLoadConsumer>
```

---

# 102. DataLoader-style coordination

VoltStack podrá adoptar un patrón inspirado conceptualmente en DataLoader:

```text
request load(A)
request load(B)
request load(C)
       │
       ▼
collect compatible requests
       │
       ▼
dispatch batch
```

---

# 103. DataLoader concept ≠ JavaScript dependency

No implica dependencia de una implementación JS.

Es un patrón de coordinación.

---

# 104. Dispatch boundary

Debe existir un punto explícito donde el batch se selle.

Ejemplos:

- end of collection phase;
- before first result is required;
- explicit `dispatch()`;
- scheduler boundary;
- controlled micro-batch window.

---

# 105. No arbitrary waiting

El sistema no debe retrasar una operación indefinidamente esperando más candidates.

---

# 106. Synchronous PHP

En runtime síncrono, batching eager normalmente conoce todos los owners de antemano.

```text
root hydration
→ owner set
→ batch load
```

---

# 107. Lazy synchronous batching

Es más limitado porque el primer acceso necesita resultado inmediatamente.

---

# 108. Lazy batch candidates

Puede aprovechar:

- known sibling owners;
- pre-registered handles;
- collection graph;
- explicit batch context.

---

# 109. No semantic delay surprise

No deberá introducirse una espera artificial grande solo para acumular owners.

---

# 110. Async/coroutine runtime

OpenSwoole puede permitir micro-batching más agresivo.

Aun así:

```text
batch window
```

debe estar acotada.

---

# 111. BatchWindowPolicy

```php
final readonly class BatchWindowPolicy
{
    public function __construct(
        public int $maxCandidates,
        public int $maxDelayMicros,
    ) {}
}
```

---

# 112. Runtime-specific scheduling

El scheduler puede variar.

La semántica ORM no.

---

# 113. Eager Loading integration

Consulta:

```php
User::query()
    ->with('posts.comments')
    ->get();
```

puede producir:

```text
Phase 1:
    load users

Phase 2:
    batch User.posts

Phase 3:
    batch Post.comments
```

---

# 114. Breadth-first loading

Una estrategia recomendada para nested eager loading:

```text
Level 0: Users
Level 1: Posts
Level 2: Comments
```

---

# 115. Breadth-first advantage

Puede reducir:

- query explosion;
- recursive loading;
- duplicated loads.

---

# 116. Depth-first alternative

Puede existir para casos específicos, pero no debe ser default sin razón.

---

# 117. Nested batch graph

```text
User
├── posts
│   ├── comments
│   └── tags
└── profile
```

puede compilarse a:

```text
Batch Load Graph
```

---

# 118. Batch graph ≠ Query AST

Es planificación ORM de relaciones.

---

# 119. BatchLoadGraph

```php
final readonly class BatchLoadGraph
{
    /**
     * @param list<BatchLoadGraphNode> $roots
     */
    public function __construct(
        public array $roots,
    ) {}
}
```

---

# 120. Cycles

Relaciones pueden formar:

```text
User.posts.author.posts...
```

---

# 121. Cycle detection

El graph deberá detectar paths repetidos según:

```text
RelationshipId
+
FetchGraphPath
```

---

# 122. Cycle ≠ invalid mapping

El ciclo puede ser válido.

Lo que debe limitarse es expansión infinita.

---

# 123. Maximum depth

```text
maxRelationshipLoadDepth
```

puede actuar como guard.

---

# 124. Depth limit ≠ silent truncation

Si el usuario solicitó explícitamente una profundidad que excede política:

```text
RelationshipLoadDepthExceededException
```

o diagnóstico explícito.

---

# 125. IdentityMap reconciliation

Cada target entity debe pasar por IdentityMap.

---

# 126. Existing managed entity

Si existe:

```text
reuse canonical instance
```

---

# 127. Dirty managed entity

Batch hydration no deberá sobrescribir ciegamente campos dirty.

---

# 128. Hydration integration

Debe reutilizar:

```text
HydrationPlan
EntityHydrator
LoadedFieldMask
IdentityMap
Snapshot System
```

---

# 129. Batch loading ≠ custom hydration

No se creará un hydrator alternativo para batching.

---

# 130. Result distribution

Después de hidratación:

```text
Canonical Entity
+
Owner Association Information
```

se distribuye.

---

# 131. RelationshipAssemblySession

```php
interface RelationshipAssemblySession
{
    public function attach(
        EntityKey $owner,
        RelationshipId $relationship,
        object $target,
    ): void;

    public function finalizeOwner(EntityKey $owner): void;
}
```

---

# 132. Assembly session scope

Debe vivir únicamente durante la operación correspondiente.

---

# 133. No static assembly maps

Prohibido en persistent workers.

---

# 134. Coverage

Para cada owner:

```text
UNKNOWN
PARTIAL
COMPLETE
```

debe determinarse independientemente.

---

# 135. Batch success ≠ all owners complete automatically

Ejemplo:

```text
slice(0, 10)
```

para cada owner.

Resultado:

```text
coverage = PARTIAL
```

aunque la query termine correctamente.

---

# 136. Full relation batch

Si la query representa toda la relación y finaliza correctamente:

```text
coverage = COMPLETE
```

para cada owner incluido.

---

# 137. Empty result

Owner sin targets:

```text
COMPLETE + EMPTY
```

solo si el plan cubría toda la relación.

---

# 138. Partial failure

Supongamos:

```text
Chunk 1 → success
Chunk 2 → failure
Chunk 3 → not executed
```

---

# 139. Per-owner outcome

Los owners del chunk 1 pueden tener conocimiento válido.

Los demás no.

---

# 140. Batch outcome ≠ uniform owner outcome

Debe conservarse granularidad.

---

# 141. RelationBatchOutcome

```php
final readonly class RelationBatchOutcome
{
    /**
     * @param array<EntityKey, OwnerLoadOutcome> $owners
     */
    public function __construct(
        public BatchOutcomeStatus $status,
        public array $owners,
    ) {}
}
```

---

# 142. OwnerLoadOutcome

Puede expresar:

```text
SUCCESS_COMPLETE
SUCCESS_PARTIAL
FAILED
CANCELLED
UNKNOWN
NOT_EXECUTED
```

---

# 143. No fake rollback

Si chunk 1 hidrató entidades correctamente, una falla posterior no implica borrarlas del IdentityMap.

---

# 144. Relationship publication

Sin embargo, no deberá marcarse como completa una relación cuyo chunk no terminó.

---

# 145. Staging strategy

Para cada chunk puede usarse:

```text
Hydration
→ temporary distribution
→ validate completion
→ publish relationship assembly
```

---

# 146. Pending mutations

Caso:

```text
DB User#1.posts:
    A B

pending:
    +C
    -B
```

Batch loading debe terminar con:

```text
baseline:
    A B

logical:
    A C

pending:
    +C
    -B
```

---

# 147. Batch result ≠ overwrite collection

Nunca:

```text
collection = DB rows
```

sin reconciliación.

---

# 148. Collection merge

Debe considerar:

- baseline;
- known members;
- pending adds;
- pending removes;
- clear-all;
- collection semantics;
- ordering.

---

# 149. Clear-all pending

Si existe:

```text
CLEAR_ALL
```

el resultado DB puede servir como baseline, pero logical current seguirá vacío salvo nuevas additions.

---

# 150. Snapshot establishment

Una carga completa puede establecer:

```text
RelationshipSnapshot
```

sin convertir la carga en domain mutation.

---

# 151. Snapshot strategy

Para relaciones enormes, almacenar todos los IDs puede ser costoso.

---

# 152. Snapshot hints

Metadata/policy podrá seleccionar:

```text
FULL_IDENTITY_SET
HASHED_MEMBERSHIP
DATABASE_ASSISTED
DELTA_ONLY
```

según sistema general de change tracking.

---

# 153. Memory governance

Batch loading puede multiplicar memoria:

```text
owners
+
rows
+
hydrated entities
+
distribution maps
+
snapshots
```

---

# 154. Streaming tension

Para grandes resultados:

```text
streaming
```

puede reducir memoria.

Pero relationship assembly puede necesitar agrupar por owner.

---

# 155. Ordered-by-owner streaming

Si el plan garantiza:

```text
ORDER BY owner_key
```

el assembler puede finalizar owners progresivamente.

---

# 156. Streaming finalization

```text
Owner#1 rows
→ finalize Owner#1
→ release temporary state

Owner#2 rows
→ finalize Owner#2
```

---

# 157. Query ordering as optimization

El ORM puede solicitar un distribution-friendly ordering semántico.

El Compiler decide SQL.

---

# 158. Memory budget

```php
final readonly class BatchMemoryBudget
{
    public function __construct(
        public int $maxBufferedRows,
        public int $maxBufferedEntities,
        public int $maxEstimatedBytes,
    ) {}
}
```

---

# 159. Memory pressure

Al exceder presupuesto:

- reducir chunk;
- usar streaming;
- fallar;
- degradar a estrategia segura;

según policy.

---

# 160. No silent OOM strategy

Nunca asumir que un batch ilimitado es mejor.

---

# 161. Cancellation

Cada batch deberá aceptar:

```text
CancellationToken
```

---

# 162. Cancel before dispatch

Requirements pueden permanecer sin satisfacer.

---

# 163. Cancel between chunks

Chunks completados conservan resultados válidos.

Los restantes:

```text
CANCELLED / NOT_EXECUTED
```

---

# 164. Cancel during chunk

No marcar owners incompletos como `COMPLETE`.

---

# 165. Deadline

Un batch hereda el deadline del contexto.

---

# 166. Different deadlines

Requirements con deadlines incompatibles no deben agruparse si el batching puede violar el más estricto.

---

# 167. Effective deadline

Si se permite agrupar:

```text
EffectiveDeadline
=
min(requirement deadlines)
```

solo cuando todos acepten esa política.

---

# 168. Timeout budget per chunk

El planner podrá distribuir:

```text
remaining deadline
```

entre chunks.

---

# 169. Retry

Retries deberán ser por chunk o suboperación cuando sea seguro.

---

# 170. Retry entire batch

No debe repetirse ciegamente un chunk ya exitoso si no es necesario.

---

# 171. Read operation idempotence

Aunque las lecturas sean conceptualmente repetibles, pueden observar tiempos diferentes.

Por eso retry debe respetar consistency policy.

---

# 172. Transaction context

Dentro de una transacción:

```text
all compatible chunks
→ same transaction context
```

---

# 173. No transaction hopping

No:

```text
Chunk 1 → T1
Chunk 2 → new T2
```

si el batch prometía mismo snapshot.

---

# 174. Transaction ends mid-batch

Debe fallar/abortar según consistencia requerida.

No fingir continuidad.

---

# 175. Replica routing

Todos los chunks deben respetar `ReadIntent`.

---

# 176. Sticky connection

Después de writes, el batch puede requerir primary/sticky routing.

---

# 177. Replica lag

Una futura `ReplicaLagAwarenessSystem` podrá influir en routing.

Batch Loader no implementa esa lógica.

---

# 178. Security scopes

Todas las queries generadas por batch deben preservar:

- tenant scope;
- soft-delete;
- authorization/data-access scopes;
- relationship filters.

---

# 179. Batch optimization cannot remove security predicates

Invariante crítico.

---

# 180. Authorization ≠ batching

Batch Loader no decide si el usuario tiene permiso.

Consume el contexto de acceso correspondiente.

---

# 181. Mixed authorization contexts

No agrupar si el resultado visible difiere por contexto de autorización.

---

# 182. SecurityContextFingerprint

Puede formar parte de la BatchKey cuando la capa de acceso a datos lo requiera.

---

# 183. Query cache integration

Cada chunk puede pasar por Query/Result Cache.

Batch Loader no implementa cache paralelo.

---

# 184. Cache hit distribution

Un chunk satisfecho por cache debe pasar por la misma reconciliación/assembly semántica.

---

# 185. Cache ≠ IdentityMap

Targets recuperados de second-level cache aún deben reconciliarse con IdentityMap.

---

# 186. Query deduplication

Si dos batches producen requerimientos semánticamente idénticos dentro del mismo contexto, podrá existir deduplicación.

---

# 187. Batch deduplication

Debe ocurrir antes de ejecutar cuando sea seguro.

---

# 188. In-flight deduplication

En runtimes concurrentes:

```text
Batch A requests User.posts [1,2]
Batch B requests User.posts [1,2]
```

podrían compartir operación in-flight.

---

# 189. In-flight key

Debe incluir suficiente contexto:

```text
BatchKey
+
OwnerSetFingerprint
+
LoadRequirement
```

---

# 190. No global in-flight sharing

No entre requests independientes.

---

# 191. Concurrency

Batch Coordinator deberá ser scope-local.

---

# 192. Same EntityManager

No se asumirá seguro para mutaciones concurrentes.

---

# 193. Async execution

En futuras extensiones podrían ejecutarse batches independientes concurrentemente:

```text
User.posts
User.profile
User.roles
```

si:

- connection strategy lo permite;
- transaction semantics lo permiten;
- EntityManager assembly se serializa apropiadamente.

---

# 194. Parallel query execution ≠ parallel ORM mutation

Pueden ejecutarse queries concurrentemente y reconciliar resultados posteriormente de forma controlada.

---

# 195. Deterministic assembly

La concurrencia no debe hacer que IdentityMap/collections dependan del race order.

---

# 196. Batch ordering

Para batches equivalentes, la planificación debe ser determinista.

---

# 197. Owner ordering

Canonicalizar owner sets no deberá cambiar ordering de las colecciones relacionadas.

---

# 198. Deterministic chunking

Con mismas:

```text
capabilities
budget
owner set
metadata
```

debe producirse el mismo chunk plan.

---

# 199. Batch fingerprints

```text
BatchFingerprint
=
Hash(
    RelationshipId
    + LoadShape
    + ContextCompatibility
    + OwnerIdentityShape
)
```

No incluir secretos.

---

# 200. Compiled batch plan cache

Puede cachearse una plantilla estructural:

```text
relationship
+
identifier shape
+
platform capability profile
+
load shape
```

---

# 201. Do not cache owners

La cache de planes no almacenará:

- entidades;
- owner IDs concretos;
- tenant runtime state;
- current transaction.

---

# 202. Plan template

Ejemplo:

```text
ManyToOneBatchPlanTemplate
IdentifierArity: 1
Strategy: TARGET_ID_SET
MaxOwnersPerChunk: capability-derived
```

---

# 203. Metadata generation

El cache key deberá considerar:

```text
RelationshipMetadataGeneration
```

---

# 204. Capability generation

Cambios relevantes de capability/platform también invalidan planes incompatibles.

---

# 205. Persistent runtime

FrankenPHP mantiene workers vivos.

Por tanto, se separa:

```text
shareable immutable
```

de:

```text
request-scoped mutable
```

---

# 206. Shareable

Puede compartirse:

- RelationshipMetadata;
- BatchPlanTemplate;
- compiled accessors;
- capability profiles;
- immutable policies.

---

# 207. Scoped

Debe ser scoped:

- BatchCoordinator;
- candidate queue;
- owner sets;
- distribution maps;
- assembly sessions;
- cancellation;
- deadline;
- transaction;
- tenant;
- shard;
- in-flight batches.

---

# 208. Request reset

Al finalizar request:

```text
BatchCoordinator.clear()
CandidateQueue.clear()
InFlightRegistry.clear()
AssemblySessions.clear()
```

---

# 209. No implicit dispatch during reset

Reset no debe ejecutar batches pendientes.

---

# 210. Unresolved batches at scope end

Deben:

- cancelarse;
- descartarse;
- reportarse;

según lifecycle policy.

Nunca ejecutarse en el siguiente request.

---

# 211. FrankenPHP

Modelo recomendado:

```text
Worker
├── immutable batch infrastructure
│
├── Request A
│   └── BatchContext A
│
└── Request B
    └── BatchContext B
```

---

# 212. RoadRunner

Mismo principio.

---

# 213. OpenSwoole

Además deberá existir:

```text
coroutine-local context
```

cuando corresponda.

---

# 214. No static candidate queue

Prohibido:

```php
final class BatchLoader
{
    private static array $pending = [];
}
```

---

# 215. Large owner sets

Ejemplo:

```text
1,000,000 owners
```

no deberá materializarse ciegamente en un único array si existe una estrategia bounded.

---

# 216. Incremental owner batching

Puede procesarse:

```text
owners stream
→ bounded owner window
→ batch
→ release
```

---

# 217. Tradeoff

Incremental batching puede limitar oportunidades globales de deduplication.

El planner debe balancearlo mediante policy.

---

# 218. Query cardinality estimates

Si metadata/telemetry indica:

```text
User.posts average = 10,000
```

un batch de 500 owners podría ser peligroso.

---

# 219. Expected row budget

```text
ExpectedRows
=
OwnerCount
× EstimatedRelationshipCardinality
```

---

# 220. Estimate ≠ guarantee

Las estimaciones solo guían resource governance.

---

# 221. Hard limits

Debe haber límites independientes de estimaciones.

---

# 222. Relationship cardinality hints

Metadata puede declarar hints:

```text
LOW
MEDIUM
HIGH
UNBOUNDED
```

pero no sustituye observación real.

---

# 223. Telemetry-assisted planning

Una futura versión podrá mantener histogramas de cardinalidad.

---

# 224. No hot-path mandatory statistics service

El sistema debe funcionar sin telemetry histórica.

---

# 225. N+1 mitigation

Batch loading puede reducir:

```text
1 root query + N relation queries
```

a:

```text
1 root query + K bounded relation queries
```

donde:

```text
K << N
```

en escenarios compatibles.

---

# 226. Formula

```text
K
=
ceil(
    CompatibleOwners
    /
    EffectiveChunkCapacity
)
```

para relaciones simples, sujeto a estrategia.

---

# 227. Polymorphic formula

Aproximadamente:

```text
K
=
Σ chunks per compatible target-type group
```

---

# 228. N+1 elimination not guaranteed

Si cada owner pertenece a:

- tenant diferente;
- shard diferente;
- scope diferente;

el batching puede no ser posible.

---

# 229. Diagnostic explanation

El sistema debe poder explicar:

```text
Why were these loads not batched?
```

---

# 230. BatchExplainer

Ejemplo:

```text
BATCH EXPLAIN

Relationship:
    User.posts

Candidates:
    100

Compatible groups:
    3

Reasons for split:
    Tenant:
        2 groups

    ReadIntent:
        2 variants

    Parameter limit:
        2 chunks

Queries planned:
    5
```

---

# 231. Explainability

Cada split significativo debería tener un reason code.

---

# 232. BatchSplitReason

```php
enum BatchSplitReason
{
    case DIFFERENT_RELATIONSHIP;
    case DIFFERENT_PERSISTENCE_CONTEXT;
    case DIFFERENT_DATABASE;
    case DIFFERENT_TENANT;
    case DIFFERENT_SHARD;
    case DIFFERENT_TRANSACTION;
    case DIFFERENT_READ_INTENT;
    case DIFFERENT_SCOPE;
    case DIFFERENT_LOAD_SHAPE;
    case PARAMETER_LIMIT;
    case OWNER_LIMIT;
    case MEMORY_LIMIT;
    case ROW_LIMIT;
    case POLYMORPHIC_TARGET_TYPE;
}
```

---

# 233. Diagnostics ≠ planning logic

Reason codes describen decisiones ya tomadas.

No sustituyen policies/capabilities.

---

# 234. Telemetry

Métricas posibles:

```text
database.orm.relationship.batch.requests
database.orm.relationship.batch.candidates
database.orm.relationship.batch.groups
database.orm.relationship.batch.chunks
database.orm.relationship.batch.owners
database.orm.relationship.batch.targets
database.orm.relationship.batch.queries
database.orm.relationship.batch.duration
database.orm.relationship.batch.identity_map_hits
database.orm.relationship.batch.identity_map_misses
database.orm.relationship.batch.deduplicated_owners
database.orm.relationship.batch.deduplicated_targets
database.orm.relationship.batch.parameter_splits
database.orm.relationship.batch.memory_splits
database.orm.relationship.batch.polymorphic_groups
database.orm.relationship.batch.failures
database.orm.relationship.batch.cancellations
database.orm.relationship.batch.unknown_outcomes
```

---

# 235. Efficiency metrics

Útiles:

```text
owners_per_query
targets_per_query
queries_avoided
identity_map_hit_ratio
batch_fill_ratio
```

---

# 236. Batch fill ratio

```text
BatchFillRatio
=
ActualOwners
/
ConfiguredOrEffectiveCapacity
```

---

# 237. Queries avoided estimate

```text
QueriesAvoided
≈
IndividualLoads
-
ExecutedBatchQueries
```

No debe presentarse como exacto cuando existan otras optimizaciones.

---

# 238. High-cardinality warning

Puede detectarse:

```text
one batch
→ 500,000 rows
```

y generar diagnóstico.

---

# 239. Under-batching warning

También:

```text
100 batches
×
1 owner
```

cuando pudieron agruparse.

---

# 240. Over-batching warning

```text
1 batch
×
50,000 owners
```

con latencia/memoria problemática.

---

# 241. Security telemetry

No usar IDs de entidades como metric labels.

---

# 242. Trace spans

Posible jerarquía:

```text
orm.relationship.batch
├── batch.plan
├── batch.chunk
│   ├── query
│   └── hydrate
└── batch.assemble
```

---

# 243. Batch context attributes

Seguros:

```text
relationship.name
relationship.kind
owner.count
chunk.count
target.count
fetch.mode
platform
```

Evitar datos sensibles.

---

# 244. Error taxonomy

```text
DatabaseBatchRelationLoadingException
├── InvalidBatchRequirementException
├── BatchCompatibilityException
├── BatchGroupingException
├── BatchPlanningException
├── BatchCapacityException
├── BatchParameterLimitException
├── BatchMemoryLimitException
├── BatchRowLimitException
├── BatchChunkingException
├── BatchExecutionException
├── BatchPartialFailureException
├── BatchUnknownOutcomeException
├── BatchCancellationException
├── BatchDeadlineException
├── BatchDistributionException
├── BatchAssemblyException
├── BatchCardinalityViolationException
├── BatchCoverageException
├── BatchSnapshotException
├── BatchMutationMergeException
├── BatchIdentityConflictException
├── BatchTransactionContextException
├── BatchTenantContextException
├── BatchShardContextException
├── BatchSecurityContextException
├── BatchPolymorphicResolutionException
├── BatchPolymorphicFanoutException
├── BatchConcurrencyException
├── BatchRuntimeIsolationException
└── BatchInvariantViolationException
```

---

# 245. Error granularity

Un error global no deberá borrar información granular de owners/chunks.

---

# 246. Partial failure report

Ejemplo:

```text
BATCH RELATION LOAD PARTIAL FAILURE

Relationship:
    User.posts

Owners:
    1,500

Chunks:
    3

Chunk 1:
    SUCCESS
    Owners: 500

Chunk 2:
    FAILED
    Owners: 500

Chunk 3:
    NOT_EXECUTED
    Owners: 500

Global Status:
    FAILED_PARTIAL
```

---

# 247. Testing architecture

El sistema requiere pruebas de:

- grouping;
- compatibility;
- chunking;
- parameter limits;
- composite IDs;
- IdentityMap;
- to-one;
- to-many;
- many-to-many;
- polymorphic;
- ordering;
- scopes;
- tenant isolation;
- shard isolation;
- transactions;
- pending mutations;
- partial failures;
- cancellation;
- memory limits;
- persistent workers.

---

# 248. To-one tests

Caso:

```text
100 posts
10 unique authors
```

esperar aproximadamente:

```text
≤ bounded queries required for 10 unique targets
```

no 100.

---

# 249. IdentityMap tests

Si 8 de 10 authors ya existen:

```text
only 2 target identities
```

deben necesitar DB load.

---

# 250. To-many tests

```text
100 users
→ posts
```

debe distribuir cada post al owner correcto.

---

# 251. Empty collection tests

Owners sin rows deben quedar `COMPLETE EMPTY` cuando corresponda.

---

# 252. Partial query tests

Un slice no debe marcar full coverage.

---

# 253. Ordering tests

Cada owner conserva ordering definido.

---

# 254. Many-to-many tests

Probar:

- duplicate physical membership;
- SET semantics;
- LIST semantics;
- shared targets;
- pending add/remove.

---

# 255. Polymorphic tests

Probar:

```text
Post
Video
Photo
```

agrupados por tipo canónico.

---

# 256. Unknown alias tests

Debe fallar antes de arbitrary class loading.

---

# 257. Composite identifier tests

Deben comprobar chunk capacity correctamente.

---

# 258. Parameter limit tests

Con capability:

```text
max parameters = 100
```

ningún query model compilable deberá requerir más del presupuesto permitido.

---

# 259. Tenant tests

Owners de tenants distintos nunca terminan en el mismo batch.

---

# 260. Transaction tests

Requirements de transacciones distintas nunca se fusionan.

---

# 261. ReadIntent tests

`PRIMARY_REQUIRED` no se degrada a replica.

---

# 262. Cancellation tests

Cancelar después del chunk 1:

```text
chunk 1 → valid
chunk 2+ → not complete
```

---

# 263. Pending mutation tests

```text
DB = A,B
pending = +C,-B
```

debe resultar:

```text
logical = A,C
```

---

# 264. Worker isolation tests

Request A y Request B no comparten:

- candidates;
- batches;
- owner maps;
- in-flight operations;
- tenant;
- transaction.

---

# 265. Determinism tests

Misma entrada y capabilities producen mismo grouping/chunk plan.

---

# 266. Performance tests

Medir:

```text
1 owner
10 owners
100 owners
1,000 owners
10,000 owners
```

para relaciones de distintas cardinalidades.

---

# 267. Benchmark dimensions

- query count;
- latency;
- peak memory;
- hydration time;
- assembly time;
- IdentityMap lookup cost;
- chunk planning cost.

---

# 268. Benchmark comparison

Comparar:

```text
individual loading
vs
batch loading
vs
eager join strategy
```

sin asumir que batching siempre gana.

---

# 269. Architectural invariants

## DB-ORM-BATCH-001

Batch Relation Loading será una estrategia de Relationship Loading.

## DB-ORM-BATCH-002

Batch Relation Loading no generará SQL directamente.

## DB-ORM-BATCH-003

Batch Relation Loading no utilizará PDO directamente.

## DB-ORM-BATCH-004

Batch Relation Loading no será equivalente a Eager Loading.

## DB-ORM-BATCH-005

Batch Relation Loading no será equivalente a Lazy Loading.

## DB-ORM-BATCH-006

Batch Relation Loading no será equivalente a N+1 Detection.

## DB-ORM-BATCH-007

Un load requirement no será una query.

## DB-ORM-BATCH-008

Un load requirement no será un batch.

## DB-ORM-BATCH-009

Solo missing knowledge será candidato a load.

## DB-ORM-BATCH-010

Relaciones completas no serán recargadas sin razón explícita.

## DB-ORM-BATCH-011

BatchKey expresará compatibilidad semántica.

## DB-ORM-BATCH-012

BatchKey será distinto de EntityKey.

## DB-ORM-BATCH-013

RelationshipId formará parte de la compatibilidad.

## DB-ORM-BATCH-014

PersistenceContexts distintos no serán mezclados.

## DB-ORM-BATCH-015

Database contexts distintos no serán mezclados.

## DB-ORM-BATCH-016

Tenants incompatibles no serán mezclados.

## DB-ORM-BATCH-017

Shards incompatibles no serán mezclados.

## DB-ORM-BATCH-018

Transactions incompatibles no serán mezcladas.

## DB-ORM-BATCH-019

ReadIntents incompatibles no serán mezclados.

## DB-ORM-BATCH-020

Scopes incompatibles no serán mezclados.

## DB-ORM-BATCH-021

Load shapes incompatibles no serán mezclados.

## DB-ORM-BATCH-022

Ordering semantics serán preservadas.

## DB-ORM-BATCH-023

Projection semantics serán preservadas.

## DB-ORM-BATCH-024

Batch lifecycle será explícito.

## DB-ORM-BATCH-025

Batch sealing impedirá mutaciones estructurales posteriores.

## DB-ORM-BATCH-026

RelationBatchPlan será distinto de QueryPlan.

## DB-ORM-BATCH-027

Un batch podrá producir múltiples queries.

## DB-ORM-BATCH-028

Chunking no cambiará semántica lógica.

## DB-ORM-BATCH-029

Chunking respetará platform capabilities.

## DB-ORM-BATCH-030

Version checks no sustituirán capability checks.

## DB-ORM-BATCH-031

Parameter limits serán respetados.

## DB-ORM-BATCH-032

Reserved parameters serán considerados.

## DB-ORM-BATCH-033

Composite IDs consumirán múltiples parameter slots cuando corresponda.

## DB-ORM-BATCH-034

Batch size será bounded.

## DB-ORM-BATCH-035

Un batch gigante no será considerado automáticamente óptimo.

## DB-ORM-BATCH-036

To-one targets serán deduplicados por canonical identity.

## DB-ORM-BATCH-037

IdentityMap será consultado antes de target DB load.

## DB-ORM-BATCH-038

IdentityMap hits no requerirán query.

## DB-ORM-BATCH-039

To-one null conocido no requerirá query.

## DB-ORM-BATCH-040

Unknown FK no será tratado como NULL.

## DB-ORM-BATCH-041

To-many results serán distribuidos por owner.

## DB-ORM-BATCH-042

Owner sin rows solo será empty cuando exista evidencia completa.

## DB-ORM-BATCH-043

Cancelled query no producirá fake empty collections.

## DB-ORM-BATCH-044

Failed query no producirá fake empty collections.

## DB-ORM-BATCH-045

Per-owner ordering será preservado.

## DB-ORM-BATCH-046

One-to-one cardinality violations no serán ocultadas.

## DB-ORM-BATCH-047

Many-to-many batching reutilizará Membership Engine.

## DB-ORM-BATCH-048

Many-to-many no asumirá que join row es domain entity.

## DB-ORM-BATCH-049

Collection semantics serán preservadas.

## DB-ORM-BATCH-050

Polymorphic batching reutilizará MorphTypeRegistry.

## DB-ORM-BATCH-051

Persisted morph alias no será usado como arbitrary PHP class.

## DB-ORM-BATCH-052

Polymorphic loads serán agrupados por target type compatible.

## DB-ORM-BATCH-053

Polymorphic fanout será bounded.

## DB-ORM-BATCH-054

Composite owner IDs serán soportados.

## DB-ORM-BATCH-055

Composite target IDs serán soportados.

## DB-ORM-BATCH-056

ORM expresará identity sets, no vendor SQL.

## DB-ORM-BATCH-057

Owners duplicados serán canonicalizados.

## DB-ORM-BATCH-058

Canonicalización no perderá consumers.

## DB-ORM-BATCH-059

Batch coordination podrá seguir patrón DataLoader.

## DB-ORM-BATCH-060

DataLoader pattern no implicará dependencia JavaScript.

## DB-ORM-BATCH-061

Batch dispatch tendrá boundary explícito.

## DB-ORM-BATCH-062

Batching no esperará indefinidamente por candidates.

## DB-ORM-BATCH-063

Lazy batching no introducirá delays arbitrarios.

## DB-ORM-BATCH-064

Async micro-batching será bounded.

## DB-ORM-BATCH-065

Nested eager loading podrá utilizar breadth-first batching.

## DB-ORM-BATCH-066

Batch load graph será distinto de Query AST.

## DB-ORM-BATCH-067

Relationship graph cycles no serán mapping errors por sí mismos.

## DB-ORM-BATCH-068

Infinite graph expansion será impedida.

## DB-ORM-BATCH-069

IdentityMap será autoridad de identidad durante assembly.

## DB-ORM-BATCH-070

Managed dirty entities no serán sobrescritas ciegamente.

## DB-ORM-BATCH-071

Batch loading reutilizará Hydration System.

## DB-ORM-BATCH-072

Batch loading no tendrá hydrator alternativo.

## DB-ORM-BATCH-073

RelationshipAssemblySession será operation-scoped.

## DB-ORM-BATCH-074

Assembly maps no serán static.

## DB-ORM-BATCH-075

Coverage será determinada por owner.

## DB-ORM-BATCH-076

Batch success no implicará automáticamente COMPLETE.

## DB-ORM-BATCH-077

Partial load conservará PARTIAL.

## DB-ORM-BATCH-078

Full successful load podrá establecer COMPLETE.

## DB-ORM-BATCH-079

Batch outcome no será necesariamente uniforme por owner.

## DB-ORM-BATCH-080

Partial failures conservarán granularidad.

## DB-ORM-BATCH-081

Successful chunks no serán ficticiamente rolled back.

## DB-ORM-BATCH-082

Failed chunks no serán marcados complete.

## DB-ORM-BATCH-083

Pending additions serán preservadas.

## DB-ORM-BATCH-084

Pending removals serán preservadas.

## DB-ORM-BATCH-085

CLEAR_ALL será preservado.

## DB-ORM-BATCH-086

DB result no reemplazará ciegamente PersistentCollection.

## DB-ORM-BATCH-087

Loading mutations no serán domain mutations.

## DB-ORM-BATCH-088

Relationship snapshots podrán establecerse tras load válido.

## DB-ORM-BATCH-089

Snapshot strategy podrá ser bounded.

## DB-ORM-BATCH-090

Memory será gobernada explícitamente.

## DB-ORM-BATCH-091

Streaming podrá utilizarse para grandes resultados.

## DB-ORM-BATCH-092

Streaming no degradará relationship coverage.

## DB-ORM-BATCH-093

Owner-oriented streaming podrá liberar state progresivamente.

## DB-ORM-BATCH-094

Memory pressure no será ignorada.

## DB-ORM-BATCH-095

Cancellation será first-class.

## DB-ORM-BATCH-096

Cancelación entre chunks preservará chunks exitosos.

## DB-ORM-BATCH-097

Cancelación durante chunk no producirá COMPLETE falso.

## DB-ORM-BATCH-098

Deadlines serán respetados.

## DB-ORM-BATCH-099

Requirements con garantías temporales incompatibles no serán fusionados.

## DB-ORM-BATCH-100

Retries podrán operar por chunk.

## DB-ORM-BATCH-101

Retries no repetirán ciegamente trabajo exitoso.

## DB-ORM-BATCH-102

Transaction context será preservado.

## DB-ORM-BATCH-103

Same-snapshot batching no saltará entre transacciones.

## DB-ORM-BATCH-104

ReadIntent será preservado entre chunks.

## DB-ORM-BATCH-105

Security predicates no serán eliminados por batching.

## DB-ORM-BATCH-106

Batch loading no será autorización.

## DB-ORM-BATCH-107

Security contexts incompatibles no serán fusionados.

## DB-ORM-BATCH-108

Batch loading no implementará query cache paralelo.

## DB-ORM-BATCH-109

Cache hits pasarán por IdentityMap reconciliation.

## DB-ORM-BATCH-110

Query cache no sustituirá IdentityMap.

## DB-ORM-BATCH-111

In-flight deduplication será scope-local.

## DB-ORM-BATCH-112

Requests independientes no compartirán in-flight batches.

## DB-ORM-BATCH-113

EntityManager no será concurrent-mutation-safe por defecto.

## DB-ORM-BATCH-114

Parallel execution no implicará parallel uncontrolled assembly.

## DB-ORM-BATCH-115

Assembly será determinista.

## DB-ORM-BATCH-116

Chunking será determinista bajo mismas entradas.

## DB-ORM-BATCH-117

Batch fingerprints no incluirán secretos.

## DB-ORM-BATCH-118

Compiled plan cache no almacenará owners concretos.

## DB-ORM-BATCH-119

Compiled plan cache no almacenará entities.

## DB-ORM-BATCH-120

Metadata generation participará en invalidación.

## DB-ORM-BATCH-121

Immutable batch templates podrán compartirse entre workers.

## DB-ORM-BATCH-122

Mutable BatchCoordinator será scoped.

## DB-ORM-BATCH-123

Candidate queues serán scoped.

## DB-ORM-BATCH-124

Distribution maps serán scoped.

## DB-ORM-BATCH-125

Assembly sessions serán scoped.

## DB-ORM-BATCH-126

Pending batches no sobrevivirán al request.

## DB-ORM-BATCH-127

Scope reset no ejecutará batches implícitamente.

## DB-ORM-BATCH-128

FrankenPHP preservará request isolation.

## DB-ORM-BATCH-129

RoadRunner preservará request isolation.

## DB-ORM-BATCH-130

OpenSwoole preservará coroutine isolation.

## DB-ORM-BATCH-131

No existirá static global candidate queue.

## DB-ORM-BATCH-132

Large owner sets podrán procesarse incrementalmente.

## DB-ORM-BATCH-133

Expected cardinality será un hint, no una garantía.

## DB-ORM-BATCH-134

Hard resource limits existirán independientemente de estimates.

## DB-ORM-BATCH-135

Batch loading podrá reducir N+1 sin prometer una sola query.

## DB-ORM-BATCH-136

N+1 no siempre podrá eliminarse por incompatibilidad contextual.

## DB-ORM-BATCH-137

Batch splits serán explicables.

## DB-ORM-BATCH-138

Diagnostics podrán indicar por qué no se agrupó una carga.

## DB-ORM-BATCH-139

Telemetry observará batch efficiency.

## DB-ORM-BATCH-140

Telemetry no usará entity IDs sensibles como metric labels.

## DB-ORM-BATCH-141

Over-batching será observable.

## DB-ORM-BATCH-142

Under-batching será observable.

## DB-ORM-BATCH-143

High-cardinality batches serán observables.

## DB-ORM-BATCH-144

Error reporting preservará granularidad por chunk/owner.

## DB-ORM-BATCH-145

Testing incluirá límites reales de parámetros.

## DB-ORM-BATCH-146

Testing incluirá persistent worker isolation.

## DB-ORM-BATCH-147

Testing incluirá pending relationship mutations.

## DB-ORM-BATCH-148

Testing incluirá polymorphic fanout.

## DB-ORM-BATCH-149

Testing incluirá partial failures.

## DB-ORM-BATCH-150

Batch loading conservará exactamente la misma semántica observable que cargas individuales equivalentes, salvo diferencias permitidas de timing/performance.

---

# 270. Anti-patterns

## 270.1 `WHERE IN` como arquitectura

```text
Batch Loading = WHERE IN
```

**Rechazado.**

---

## 270.2 Un único batch global

```php
static array $pendingRelations;
```

**Rechazado.**

---

## 270.3 Mezclar tenants

```text
Tenant A IDs
+
Tenant B IDs
→ one query
```

**Rechazado.**

---

## 270.4 Mezclar transacciones

**Rechazado.**

---

## 270.5 Batch ilimitado

```text
500,000 IDs
→ one query
```

**Rechazado.**

---

## 270.6 Ignorar parameter limits

**Rechazado.**

---

## 270.7 Asumir que batch = one query

**Rechazado.**

---

## 270.8 Marcar empty después de query fallida

**Rechazado.**

---

## 270.9 Reemplazar PersistentCollection con DB rows

**Rechazado.**

---

## 270.10 Ignorar pending mutations

**Rechazado.**

---

## 270.11 SQL en Batch Planner

**Rechazado.**

---

## 270.12 Bypass de scopes

**Rechazado.**

---

## 270.13 Bypass de IdentityMap

**Rechazado.**

---

## 270.14 Query por cada target después del batch de memberships

**Rechazado cuando targets pueden agruparse.**

---

## 270.15 Fanout polimórfico ilimitado

**Rechazado.**

---

# 271. Estructura de directorios

```text
src/Quantum/Database/ORM/Relationship/Batch/
│
├── Contract/
│   ├── RelationBatchLoader.php
│   ├── RelationBatchPlanner.php
│   ├── RelationBatchCoordinator.php
│   ├── RelationBatchGrouper.php
│   └── RelationBatchResultDistributor.php
│
├── Requirement/
│   ├── RelationshipLoadRequirement.php
│   ├── RelationshipLoadShape.php
│   └── RelationshipKnowledgeRequirement.php
│
├── Key/
│   ├── RelationBatchKey.php
│   ├── BatchFingerprint.php
│   ├── BatchOwnerSet.php
│   └── SecurityContextFingerprint.php
│
├── Batch/
│   ├── RelationBatch.php
│   ├── RelationBatchState.php
│   ├── RelationBatchChunk.php
│   └── RelationBatchOutcome.php
│
├── Planning/
│   ├── DefaultRelationBatchPlanner.php
│   ├── RelationBatchPlan.php
│   ├── BatchPlanTemplate.php
│   ├── BatchChunkPlanner.php
│   ├── BatchSizeStrategy.php
│   └── BatchSplitReason.php
│
├── Budget/
│   ├── BatchLoadBudget.php
│   ├── BatchMemoryBudget.php
│   ├── MorphBatchBudget.php
│   └── BatchWindowPolicy.php
│
├── Strategy/
│   ├── ToOneBatchStrategy.php
│   ├── ToManyBatchStrategy.php
│   ├── ManyToManyBatchStrategy.php
│   ├── PolymorphicBatchStrategy.php
│   └── CompositeIdentityBatchStrategy.php
│
├── Distribution/
│   ├── OwnerDistributionMap.php
│   ├── DefaultOwnerDistributionMap.php
│   └── RelationshipAssemblySession.php
│
├── Coordination/
│   ├── DefaultRelationBatchCoordinator.php
│   ├── BatchCandidateCollector.php
│   ├── InFlightBatchRegistry.php
│   └── BatchDispatchBoundary.php
│
├── Graph/
│   ├── BatchLoadGraph.php
│   ├── BatchLoadGraphNode.php
│   └── BatchLoadCycleDetector.php
│
├── Outcome/
│   ├── BatchOutcomeStatus.php
│   ├── OwnerLoadOutcome.php
│   └── ChunkLoadOutcome.php
│
├── Diagnostics/
│   ├── BatchExplainer.php
│   ├── BatchDiagnosticReport.php
│   └── BatchSplitExplanation.php
│
├── Telemetry/
│   ├── BatchLoadingTelemetry.php
│   └── BatchEfficiencyMetrics.php
│
└── Exception/
    └── ...
```

---

# 272. Dependencias

```text
Batch Relation Loading
        │
        ├── Relationship Metadata
        ├── Relationship State
        ├── IdentityMap
        ├── UnitOfWork
        ├── PersistentCollection
        ├── Polymorphic Registry
        ├── Platform Capabilities
        │
        ▼
Relationship Loading System
        │
        ▼
Entity Query
        │
        ▼
Query Engine
        │
        ▼
Execution Engine
        │
        ▼
Hydration
```

Nunca:

```text
Batch Loader
→ PDO
```

ni:

```text
Batch Planner
→ SQL string
```

---

# 273. Fórmulas fundamentales

## Compatibilidad

```text
Compatible(A,B)
⇔
BatchKey(A) = BatchKey(B)
```

salvo reglas explícitas de compatibilidad.

## Missing knowledge

```text
MissingKnowledge
=
RequiredKnowledge
-
CurrentKnowledge
```

## Chunk capacity

```text
ChunkCapacity
=
floor(
    AvailableBindBudget
    /
    BindCostPerOwner
)
```

sujeto además a límites de owners, rows y memoria.

## Batch query count

Para una relación simple:

```text
Queries
≈
ceil(
    CompatibleOwners
    /
    EffectiveChunkCapacity
)
```

## Polymorphic batching

```text
PolymorphicGroups
=
GroupBy(
    References,
    EntityType
    × DatabaseContext
    × TenantContext
    × ShardContext
    × LoadShape
)
```

## Logical relationship after load

```text
LogicalRelationship
=
ObservedDatabaseBaseline
⊕
PendingRelationshipMutations
```

## Completeness

```text
Complete(owner, relationship)
⇒
FullRequiredCoverageObservedSuccessfully
```

## Resource-safe batch

```text
SafeBatch
=
Compatible
∧
WithinParameterBudget
∧
WithinOwnerBudget
∧
WithinRowBudget
∧
WithinMemoryBudget
∧
WithinDeadline
```

---

# 274. Arquitectura final

```text
                 Relationship Requirements
                           │
                           ▼
                 Candidate Collection
                           │
                           ▼
                    Compatibility
                           │
                           ▼
                     Batch Groups
                           │
                           ▼
                     Batch Planner
                           │
                ┌──────────┴──────────┐
                ▼                     ▼
          Capability Check      Resource Budget
                │                     │
                └──────────┬──────────┘
                           ▼
                       Chunking
                           │
                           ▼
                 Relationship Loader
                           │
                           ▼
                     Entity Query
                           │
                           ▼
                      Query Engine
                           │
                           ▼
                  Execution Engine
                           │
                           ▼
                       Hydration
                           │
                           ▼
                      IdentityMap
                           │
                           ▼
                  Result Distribution
                           │
                           ▼
                Relationship Assembly
                           │
                           ▼
                Snapshot / UoW Merge
                           │
                           ▼
                 Coverage Finalization
```

---

# 275. Regla maestra final

La arquitectura deberá garantizar:

```text
Batch Optimization
<
Semantic Correctness
```

Nunca al contrario.

En otras palabras:

> **VoltStack podrá convertir cientos o miles de necesidades de relaciones en un conjunto pequeño y acotado de operaciones, pero solo cuando todas las cargas agrupadas sean semánticamente compatibles. La reducción de queries jamás justificará mezclar tenants, shards, transacciones, scopes, políticas de lectura, identidades o estados ORM incompatibles.**

La API podrá seguir siendo sencilla:

```php
$users = User::query()
    ->with('posts.comments')
    ->get();
```

mientras internamente:

```text
Users
  ↓
Batch(User.posts)
  ↓
Posts
  ↓
Batch(Post.comments)
  ↓
Comments
```

se ejecuta mediante:

```text
Compatibility
+
Chunking
+
Capabilities
+
Resource Governance
+
IdentityMap
+
Hydration
+
Relationship Assembly
+
UnitOfWork Reconciliation
```

sin crear un segundo Query Engine ni romper las invariantes del ORM.

---

# 276. Siguiente documento

```text
154_DATABASE_N_PLUS_ONE_DETECTION_SYSTEM.md
```

El siguiente documento deberá cerrar el bloque de Relationships diseñando la detección sistemática del problema N+1:

```text
Root Query
   │
   ▼
N entities
   │
   ├── relationship query
   ├── relationship query
   ├── relationship query
   └── ...
         │
         ▼
N+1 Detector
```

Deberá cubrir:

- definición formal de N+1;
- query fingerprints;
- relationship fingerprints;
- call-site fingerprints;
- lazy-load correlation;
- root-query correlation;
- nested N+1;
- polymorphic N+1;
- extra-lazy N+1;
- duplicate-query detection;
- threshold policies;
- confidence levels;
- false-positive control;
- batch/eager loading recommendations;
- development warnings;
- production telemetry;
- profiler integration;
- Debug Toolbar integration futura;
- testing assertions;
- request/job scope;
- persistent runtime isolation;
- low-overhead instrumentation;
- sampling;
- diagnostics y explainability.

Regla central propuesta:

> **VoltStack detectará N+1 mediante correlación semántica entre una operación raíz y cargas repetidas equivalentes, no simplemente contando strings SQL idénticos.**