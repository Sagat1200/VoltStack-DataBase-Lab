# 151_DATABASE_EAGER_LOADING_SYSTEM.md

# VoltStack Quantum Database
## Database Eager Loading System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 151 — Database Eager Loading System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Eager Loading System` define cómo VoltStack solicita, planifica, ejecuta y ensambla relaciones que deben estar disponibles como parte de una operación de lectura antes de entregar su resultado lógico a la aplicación.

El objetivo es permitir APIs como:

```php
$users = User::query()
    ->with('profile', 'posts.comments', 'roles')
    ->get();
```

sin imponer que:

```text
with(...)
=
SQL JOIN
```

El sistema podrá seleccionar distintas estrategias físicas:

```text
EAGER REQUIREMENT
        │
        ▼
Fetch Graph
        │
        ▼
Eager Load Planner
        │
        ├── JOIN
        ├── SELECT_IN
        ├── BATCH
        ├── POLYMORPHIC_BATCH
        └── HYBRID
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
Relationship Assembly
        │
        ▼
Complete Logical Result
```

---

# 2. Regla central

> **Eager Loading expresa qué relaciones deben estar disponibles al completar una operación de lectura; no prescribe SQL JOIN. VoltStack deberá seleccionar y combinar estrategias de carga que satisfagan el Fetch Graph preservando cardinalidad lógica, identidad canónica, cobertura, consistencia, límites de recursos y semántica de la consulta raíz.**

Formalmente:

```text
EagerRequirement
≠
JoinRequirement
```

y:

```text
EagerLoad
=
FetchRequirement
+
StrategyPlanning
+
RelationshipLoading
+
Hydration
+
Assembly
+
Finalization
```

---

# 3. Eager Loading ≠ JOIN

Ésta es una de las reglas más importantes.

Una solicitud:

```php
User::query()
    ->with('posts')
    ->get();
```

puede resolverse mediante:

### JOIN

```sql
SELECT ...
FROM users
LEFT JOIN posts ...
```

### SELECT-IN

```text
Query 1:
    Users

Query 2:
    Posts WHERE author_id IN (...)
```

### BATCH

```text
Root owners
    ↓
compatible groups
    ↓
multiple bounded relationship queries
```

### HYBRID

```text
User.profile
    → JOIN

User.posts
    → SELECT_IN

User.roles
    → SELECT_IN

User.posts.comments
    → BATCH
```

La estrategia es una decisión del planner.

---

# 4. Posición arquitectónica

```text
Entity Query
    │
    ├── Root Query Semantics
    │
    └── Eager Fetch Requirements
             │
             ▼
      Relationship Fetch Graph
             │
             ▼
       Eager Load Planner
             │
       ┌─────┼──────────┐
       ▼     ▼          ▼
     JOIN SELECT_IN    BATCH
       │     │          │
       └─────┼──────────┘
             ▼
        Load Schedule
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
 Relationship Assembly
             │
             ▼
   Fetch Graph Finalization
             │
             ▼
      Logical Result
```

---

# 5. Eager Loading ≠ Relationship Loading

`Relationship Loading System` es la infraestructura general para obtener relaciones.

`Eager Loading System` especializa esa infraestructura para el caso:

> La relación debe estar disponible antes de entregar el resultado de la operación raíz.

Por tanto:

```text
EagerLoadingSystem
        ↓ uses
RelationshipLoadingSystem
```

---

# 6. Eager Loading ≠ Lazy Loading

Lazy:

```text
return entity
    ↓
later access relationship
    ↓
possibly execute I/O
```

Eager:

```text
execute root operation
    ↓
load required relationships
    ↓
finalize graph
    ↓
return result
```

---

# 7. Eager Loading ≠ Preloading Everything

Eager loading no significa:

```text
load every mapped relationship
```

Significa cargar únicamente las relaciones declaradas por:

- query;
- fetch graph;
- metadata policy;
- explicit fetch profile;
- extension controlada.

---

# 8. Eager Requirement

Conceptualmente:

```php
final readonly class EagerLoadRequirement
{
    public function __construct(
        public RelationshipPath $path,
        public EagerLoadPolicy $policy,
        public ?RelationshipConstraint $constraint = null,
        public ?ProjectionRequirement $projection = null,
    ) {}
}
```

---

# 9. RelationshipPath

Ejemplo:

```text
posts.comments.author
```

debe compilarse como path tipado:

```text
User
 └── posts : Post[]
      └── comments : Comment[]
           └── author : User
```

No conservarse únicamente como string durante runtime.

---

# 10. String API ≠ Internal Model

La API podrá aceptar:

```php
->with('posts.comments.author')
```

pero internamente deberá convertirse a:

```text
RelationshipPath
+
RelationshipIds
+
FetchGraphNodes
```

durante construcción/compilación.

---

# 11. Fetch Graph

Varias solicitudes:

```php
->with([
    'profile',
    'posts.comments',
    'posts.tags',
    'roles',
])
```

deben normalizarse:

```text
User
├── profile
├── posts
│   ├── comments
│   └── tags
└── roles
```

---

# 12. Prefix deduplication

Estas solicitudes:

```text
posts
posts.comments
posts.comments.author
posts.tags
```

no deberán producir cuatro árboles independientes.

Se normalizan en un único graph.

---

# 13. RelationshipFetchGraph

```php
final readonly class RelationshipFetchGraph
{
    /**
     * @param list<RelationshipFetchNode> $roots
     */
    public function __construct(
        public EntityType $rootType,
        public array $roots,
        public FetchGraphFingerprint $fingerprint,
    ) {}
}
```

---

# 14. RelationshipFetchNode

```php
final readonly class RelationshipFetchNode
{
    /**
     * @param list<RelationshipFetchNode> $children
     */
    public function __construct(
        public RelationshipId $relationship,
        public EagerLoadPolicy $policy,
        public ?RelationshipConstraint $constraint,
        public array $children,
    ) {}
}
```

---

# 15. EagerLoadPolicy

```php
enum EagerLoadPolicy
{
    case AUTO;
    case JOIN;
    case SELECT_IN;
    case BATCH;
}
```

`AUTO` deberá ser el modo recomendado para APIs generales.

---

# 16. Policy ≠ guarantee of physical SQL

Incluso una hint `JOIN` puede ser rechazada cuando:

- no es semánticamente válida;
- viola capabilities;
- rompe cardinalidad;
- excede resource policy;
- hace imposible streaming solicitado.

Según API, VoltStack podrá:

```text
fallback
```

o:

```text
fail strict strategy requirement
```

---

# 17. Hint vs Requirement

Conviene distinguir:

```php
enum FetchStrategyStrength
{
    case PREFER;
    case REQUIRE;
}
```

Ejemplo:

```php
->with(
    'posts',
    strategy: FetchStrategy::JOIN,
    strength: FetchStrategyStrength::PREFER,
);
```

---

# 18. Default AUTO

AUTO deberá considerar:

```text
Relationship Kind
Root Cardinality
Estimated Child Cardinality
Fetch Graph Shape
Pagination
Streaming
Parameter Limits
Projection
Constraints
Ordering
Tenant Context
Shard Context
Platform Capabilities
Resource Budget
```

---

# 19. To-One eager default

Una relación:

```text
MANY_TO_ONE
ONE_TO_ONE
```

suele ser buena candidata para:

```text
JOIN
```

porque normalmente no multiplica la raíz de forma arbitraria.

Pero no es regla absoluta.

---

# 20. To-Many eager default

Una relación:

```text
ONE_TO_MANY
MANY_TO_MANY
```

suele favorecer:

```text
SELECT_IN
```

para evitar amplificación de filas.

---

# 21. Heuristic ≠ invariant

Nunca codificar:

```php
if ($relationship->isToMany()) {
    return SELECT_IN;
}
```

como única lógica.

Es una heurística del planner.

---

# 22. Eager Load Planner

```php
interface EagerLoadPlanner
{
    public function plan(
        RootQueryContract $root,
        RelationshipFetchGraph $graph,
        EagerLoadPlanningContext $context,
    ): EagerLoadPlan;
}
```

---

# 23. Planner responsibilities

El planner determina:

- estrategias por nodo;
- dependencias;
- grouping;
- batches;
- joinable nodes;
- secondary queries;
- streaming requirements;
- execution ordering;
- resource requirements;
- graph finalization rules.

---

# 24. Planner no genera SQL

Nunca:

```text
EagerLoadPlanner
→ "SELECT ..."
```

Debe producir planes semánticos.

---

# 25. EagerLoadPlan

```php
final readonly class EagerLoadPlan
{
    /**
     * @param list<EagerLoadStage> $stages
     */
    public function __construct(
        public FetchGraphFingerprint $graph,
        public RootLoadPlan $root,
        public array $stages,
        public EagerLoadRequirements $requirements,
        public EagerLoadPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 26. Load Stage

Un stage representa relaciones que pueden cargarse después de que exista un conjunto concreto de owners.

Ejemplo:

```text
Stage 0:
    root Users + JOIN profile

Stage 1:
    User.posts
    User.roles

Stage 2:
    Post.comments
    Post.tags

Stage 3:
    Comment.author
```

---

# 27. Dependency graph

Para:

```text
User.posts.comments.author
```

la dependencia es:

```text
User
 ↓
posts
 ↓
Post owners
 ↓
comments
 ↓
Comment owners
 ↓
author
```

No se puede cargar `comments` antes de conocer los `Post`.

---

# 28. Stage DAG

Formalmente:

```text
FetchGraph
→
EagerLoadStageDAG
```

---

# 29. Deterministic planning

Mismo:

```text
RootQueryShape
+
FetchGraph
+
Metadata
+
Capabilities
+
Policy
```

deberá producir el mismo plan lógico.

---

# 30. Root Query Preservation

El eager loading no deberá cambiar accidentalmente qué entidades raíz devuelve la query.

Ejemplo:

```php
User::query()
    ->where('active', true)
    ->with('posts')
    ->get();
```

La relación `posts` no deberá:

- agregar users;
- eliminar users;
- duplicar logical users;
- cambiar LIMIT/OFFSET raíz.

---

# 31. Root semantic invariant

```text
RootResultWithoutEager
≡
RootResultWithEager
```

respecto de identidad/cardinalidad/ordering de la raíz, salvo que la API declare explícitamente otra semántica.

---

# 32. JOIN danger

Un JOIN mal utilizado puede convertir:

```text
LEFT JOIN
```

en comportamiento equivalente a:

```text
INNER JOIN
```

por predicates mal ubicados.

VoltStack deberá impedir esta alteración accidental.

---

# 33. Relationship predicates

Filtros usados únicamente para eager loading deberán permanecer en el ámbito de la relación.

No deben modificar root semantics inadvertidamente.

---

# 34. Root LIMIT problem

Ejemplo:

```text
Users LIMIT 10
LEFT JOIN Posts
```

Dependiendo de cómo se compile, un `LIMIT 10` aplicado después de multiplicar filas podría producir menos de diez users lógicos.

---

# 35. Pagination safety

Cuando existe:

```text
LIMIT
OFFSET
CURSOR PAGINATION
```

el planner deberá garantizar que la estrategia eager no altere la ventana raíz.

---

# 36. Recommended pagination strategy

Para to-many:

```text
Root paginated query
        ↓
resolve root owners
        ↓
SELECT_IN relationships
```

es normalmente más seguro.

---

# 37. JOIN + pagination

Solo deberá permitirse cuando el Query Planner pueda demostrar preservación semántica, por ejemplo mediante:

- derived table;
- CTE;
- two-stage plan;
- compatible windowing.

---

# 38. Cartesian Amplification

Supongamos:

```text
User
├── posts = 20
└── roles = 5
```

JOIN simultáneo:

```text
20 × 5 = 100 rows
```

por user.

---

# 39. General amplification

Para `n` colecciones join-loaded:

```text
Amplification
≈
Π cardinality(collection_i)
```

---

# 40. Join Explosion Risk

El planner deberá estimar:

```text
JoinExplosionRisk
```

antes de unir múltiples relaciones `to-many`.

---

# 41. JoinExplosionPolicy

```php
final readonly class JoinExplosionPolicy
{
    public function __construct(
        public int $maxJoinedToMany,
        public float $maxEstimatedAmplification,
        public bool $allowUnknownAmplification,
    ) {}
}
```

---

# 42. Unknown cardinality

Si no existe estimación confiable:

```text
UNKNOWN
```

no deberá tratarse automáticamente como:

```text
LOW
```

---

# 43. Conservative strategy

Ante riesgo desconocido, puede preferirse:

```text
SELECT_IN
```

para colecciones.

---

# 44. Hybrid Plan

Ejemplo:

```text
User
├── profile       JOIN
├── organization  JOIN
├── posts         SELECT_IN
│   ├── comments  SELECT_IN
│   └── tags      SELECT_IN
└── roles         SELECT_IN
```

Esto debe ser una operación eager lógica única aunque físicamente use varias consultas.

---

# 45. Eager Operation ≠ Single Query

Fundamental:

```text
One Logical Eager Operation
may execute
N Physical Queries
```

---

# 46. Select-In Eager Loading

Pipeline:

```text
Root Query
    ↓
Hydrate owners
    ↓
Extract canonical owner IDs
    ↓
Build relationship load batches
    ↓
Entity Query WHERE FK IN (...)
    ↓
Hydrate targets
    ↓
Group by owner
    ↓
Assemble
    ↓
Finalize coverage
```

---

# 47. Owner identity extraction

Debe utilizar identidad ORM canónica.

No depender de:

```text
(string) $entity->id
```

arbitrario.

---

# 48. Composite owner IDs

Ejemplo:

```text
OrderLine
PK = (order_id, line_number)
```

Select-In deberá soportar:

```text
composite identity predicates
```

según capabilities.

---

# 49. Composite capabilities

El planner puede seleccionar:

- tuple `IN`;
- OR-of-AND;
- temporary strategy futura;
- multiple batches.

Según `PlatformCapabilities`.

---

# 50. Parameter budget

Si:

```text
P = available parameters
K = parameters per owner
```

entonces:

```text
MaxOwners
≤ floor(P / K)
```

antes de considerar parámetros adicionales de constraints.

---

# 51. Effective parameter budget

Más precisamente:

```text
EffectiveOwnerBudget
=
floor(
    (PlatformParameterLimit - FixedQueryParameters)
    /
    OwnerIdentifierArity
)
```

---

# 52. Batch chunking

Owners:

```text
10,000
```

podrán dividirse:

```text
500
500
500
...
```

según budget.

---

# 53. Chunking ≠ partial coverage

Múltiples chunks que terminan correctamente pueden representar:

```text
COMPLETE
```

---

# 54. Failed chunk

Si falla un chunk:

```text
overall relationship load
→ PARTIAL / FAILED
```

según evidencia.

Nunca `COMPLETE`.

---

# 55. Owner grouping

Resultados:

```text
Post#1 author=User#10
Post#2 author=User#10
Post#3 author=User#11
```

se distribuyen:

```text
User#10.posts → [Post#1, Post#2]
User#11.posts → [Post#3]
```

---

# 56. Owners without results

Si `User#12` participó en un full select-in y no tuvo rows:

```text
User#12.posts
=
COMPLETE empty
```

---

# 57. Nested eager loading

Ejemplo:

```php
User::query()
    ->with('posts.comments.author')
    ->get();
```

Pipeline:

```text
Users
 ↓
Posts for Users
 ↓
Comments for Posts
 ↓
Authors for Comments
 ↓
Finalize graph
```

---

# 58. Breadth-oriented scheduling

Cuando existen ramas independientes:

```text
User
├── posts
├── roles
└── teams
```

pueden programarse en el mismo stage lógico.

---

# 59. Nested stages

```text
Stage 0
    Users

Stage 1
    posts
    roles
    teams

Stage 2
    posts.comments
    teams.members

Stage 3
    posts.comments.author
```

---

# 60. Stage completion

Un stage dependiente no deberá ejecutarse hasta conocer sus owners requeridos.

---

# 61. Empty intermediate owners

Si:

```text
User.posts = []
```

entonces:

```text
posts.comments
```

no requiere query.

---

# 62. Zero-owner optimization

```text
OwnerSet = ∅
→
SkipPhysicalQuery
```

sin perder semántica.

---

# 63. Deduplicate owners

El mismo target owner puede aparecer desde múltiples rutas.

Antes de Select-In:

```text
Owners
→ canonical EntityKey dedup
```

cuando las load semantics sean compatibles.

---

# 64. Owner dedup ≠ result dedup

Deduplicar owners para evitar queries redundantes no significa cambiar cardinalidad del resultado lógico raíz.

---

# 65. IdentityMap Integration

Cada entidad eager-loaded deberá pasar por las mismas reglas de identidad canónica que cualquier otra hidratación.

---

# 66. Existing managed target

Si `Post#100` ya existe:

```text
eager result Post#100
        ↓
IdentityMap hit
        ↓
reuse canonical Post#100
```

---

# 67. Dirty managed target

El eager loader no deberá sobrescribir automáticamente campos dirty del target.

Se aplicará la política del Hydration System.

---

# 68. Eager loading ≠ Refresh

Fundamental:

```text
with('posts')
```

no significa:

```text
force refresh posts and overwrite local state
```

---

# 69. Existing complete relationship

Si una relación requerida ya está:

```text
COMPLETE
```

y suficientemente confiable:

```text
skip load
```

puede ser válido.

---

# 70. Existing partial relationship

Si el eager requirement exige relación completa:

```text
PARTIAL
→
complete load required
```

---

# 71. Existing dirty collection

La carga deberá respetar:

```text
RelationshipMergePolicy
```

del documento anterior.

---

# 72. Local mutation preservation

Ejemplo:

```text
DB baseline:
A, B

local:
+C

eager DB result:
A, B
```

Resultado lógico:

```text
current graph:
A, B, C

baseline:
A, B

pending mutation:
+C
```

---

# 73. Constrained Eager Loading

API:

```php
User::query()
    ->with([
        'posts' => fn ($query) =>
            $query->where('published', true),
    ])
    ->get();
```

---

# 74. Constraint semantics

Debe distinguirse entre:

### Mapping Constraint

Parte permanente de la relación.

### Eager Constraint

Restricción particular de esta carga.

---

# 75. Mapping Constraint

Ejemplo conceptual:

```text
User.publishedPosts
=
Post where published = true
```

Si se cargan todos:

```text
coverage = COMPLETE
```

respecto de `publishedPosts`.

---

# 76. Ad-hoc Eager Constraint

Si:

```text
User.posts
```

se carga con:

```text
published = true
```

el resultado es:

```text
PARTIAL
```

respecto de `User.posts`.

---

# 77. Critical rule

```text
Constrained Eager Load
≠
Complete Relationship
```

salvo que el constraint forme parte de la definición canónica de esa relación.

---

# 78. Constrained result publication

Se recomienda que resultados ad-hoc muy filtrados no reemplacen silenciosamente la colección managed completa.

---

# 79. Named filtered relationship

Preferible:

```text
User.publishedPosts
```

cuando la asociación filtrada tiene significado estable de dominio.

---

# 80. Limit per relationship

Caso:

```text
latest 5 posts per user
```

no debe compilarse ingenuamente como:

```text
global LIMIT 5
```

sobre todos los owners.

---

# 81. Per-owner limit

Semánticamente:

```text
LIMIT 5
PER OWNER
```

---

# 82. Platform-dependent implementation

Puede requerir:

- window functions;
- lateral joins;
- correlated subquery;
- multi-query fallback.

El Eager Loader describe semántica.

El Query Engine/Compiler resuelve representación.

---

# 83. Per-owner limit coverage

Aunque obtenga exactamente cinco posts:

```text
coverage = PARTIAL
```

respecto de una relación no limitada.

---

# 84. Ordering

Metadata puede definir:

```text
User.posts
ORDER BY createdAt DESC
```

El eager loader deberá preservar ese ordering.

---

# 85. Ordering across batches

Si una colección se obtiene en varios chunks:

```text
chunk1
chunk2
chunk3
```

no deberá asumirse que concatenarlos preserva ordering global.

---

# 86. Collection final ordering

El plan deberá:

- generar query ordering suficiente;
- o realizar merge ordenado;
- o aplicar ordering lógico seguro.

---

# 87. Stable ordering

Cuando pagination/range dependa de ordering:

```text
ordering
```

debe ser estable y determinista.

---

# 88. Many-to-Many Eager Loading

Ejemplo:

```php
User::query()
    ->with('roles')
    ->get();
```

Select-In conceptual:

```text
User IDs
   ↓
Join Table
   ↓
Role IDs / Role data
   ↓
Role hydration
   ↓
Group memberships by User
```

---

# 89. Association payload

Si join table tiene:

```text
assigned_at
assigned_by
scope
```

el eager plan deberá saber si se modela como:

- association entity;
- pivot projection;
- relationship metadata.

---

# 90. Association Entity preference

Cuando el join posee identidad/comportamiento significativo:

```text
UserRole
```

puede ser preferible a esconderlo como simple pivot.

---

# 91. Polymorphic Eager Loading

Ejemplo:

```php
Comment::query()
    ->with('commentable')
    ->get();
```

Owners:

```text
Comment#1 → Post#10
Comment#2 → Video#7
Comment#3 → Post#12
```

---

# 92. Morph grouping

Planner:

```text
Post
    ids [10,12]

Video
    ids [7]
```

---

# 93. Query per morph type

Podrán ejecutarse queries separadas:

```text
Post query
Video query
```

sin convertirlo en N+1.

---

# 94. Polymorphic nested eager loading

Ejemplo conceptual:

```text
commentable
    Post  → author
    Video → channel
```

requiere fetch branches específicas por concrete morph type.

---

# 95. Morph Fetch Graph

```text
Comment.commentable
├── Post
│   └── author
└── Video
    └── channel
```

---

# 96. Morph whitelist

Solo tipos registrados podrán formar branches.

---

# 97. Unknown morph type

No deberá causar instanciación dinámica arbitraria.

---

# 98. Eager Loading Projections

Root query:

```php
User::query()
    ->select(['id', 'name'])
    ->with('posts')
    ->get();
```

debe conservar suficiente información para resolver la relación.

---

# 99. Required key projection

Si eager loading necesita:

```text
User.id
```

pero el usuario no lo seleccionó explícitamente, VoltStack debe tener una política definida.

---

# 100. Projection policy options

### Auto-include internal key

Agregar la columna internamente sin exponerla como proyección pública adicional.

### Strict

Fallar indicando que falta la identidad requerida.

---

# 101. Recommended ORM policy

Para managed entities:

> Incluir internamente los identificadores obligatorios para mantener identidad y eager loading, siempre que no cambie la forma pública del resultado.

---

# 102. Hidden Internal Projection

Conceptualmente:

```text
Public projection:
    name

Internal hydration projection:
    id
    name
```

---

# 103. Hidden column ≠ public field

El hecho de seleccionar internamente una FK/ID no obliga a exponerla en DTO/tuple/result público.

---

# 104. Relationship foreign keys

El planner puede requerir internamente:

```text
owner key
target key
join key
discriminator
```

para assembly.

---

# 105. Projection minimization

Eager loading no debe forzar:

```text
SELECT *
```

si el HydrationPlan conoce exactamente los campos necesarios.

---

# 106. Partial target entity

Si el usuario solicita columnas parciales de una entidad eager-loaded, deberán aplicarse las reglas de partial hydration.

---

# 107. Preferred alternative

Para lecturas altamente proyectadas:

```text
DTO / Projection
```

es preferible a partial managed entities cuando sea posible.

---

# 108. Aggregate Eager Loading

A veces no se necesita la relación completa sino:

```text
posts count
orders sum
latestComment timestamp
```

---

# 109. Aggregate ≠ Relationship Initialization

Ejemplo:

```php
User::query()
    ->withCount('posts')
    ->get();
```

no debe marcar:

```text
User.posts = COMPLETE
```

---

# 110. Aggregate knowledge

```text
KnownCount(posts)
≠
KnownMembers(posts)
```

---

# 111. Aggregate eager architecture

Debe modelarse como:

```text
RelationshipAggregateRequirement
```

separado del eager load de miembros.

---

# 112. Aggregate strategies

Podrán utilizar:

- correlated subquery;
- aggregate JOIN;
- grouped secondary query;
- window expression.

Sin alterar relationship coverage.

---

# 113. Exists eager metadata

Igualmente:

```text
withExists('posts')
```

solo produce:

```text
membership existence knowledge
```

no miembros.

---

# 114. HydrationPlan Composition

El eager plan deberá coordinar varios HydrationPlans.

Ejemplo:

```text
RootHydrationPlan<User>

RelationshipHydrationPlan<Post>

RelationshipHydrationPlan<Role>

NestedHydrationPlan<Comment>
```

---

# 115. Joined Hydration Plan

Cuando `profile` viene en root JOIN:

```text
Root Hydration Plan
├── User Entity Node
└── Profile Entity Node
```

con Relationship Assembly asociado.

---

# 116. Secondary Hydration Plan

Para Select-In:

```text
PostRelationshipQuery
    ↓
Post Hydration Plan
    ↓
owner binding
    ↓
Relationship Assembly
```

---

# 117. Plan reuse

Hydration plans compatibles deberán reutilizar el `Hydration Cache System`.

---

# 118. No runtime reflection discovery

El eager hot path no deberá reconstruir mappings mediante Reflection por cada row.

---

# 119. Result Assembly

La operación eager no termina cuando termina el último SQL statement.

Termina cuando:

```text
required graph
```

ha sido ensamblado y finalizado.

---

# 120. Completion formula

```text
EagerOperationComplete
⇔
RootComplete
∧
AllRequiredFetchNodesFinalized
```

---

# 121. Fetch Node completion

```text
FetchNodeComplete
⇔
OwnersKnown
∧
RequiredLoadsCompleted
∧
AssemblyCompleted
∧
CoverageEstablished
```

---

# 122. Graph completion

```text
GraphComplete
=
∀ required node:
    FetchNodeComplete(node)
```

---

# 123. Result visibility

Para operaciones buffered:

```text
execute
→ hydrate
→ eager load
→ finalize
→ return
```

La aplicación no deberá recibir el root graph antes de satisfacer las eager guarantees.

---

# 124. Streaming Conflict

Streaming complica eager loading.

Ejemplo:

```text
stream Users
+
eager posts
```

Si `posts` usa Select-In, se necesitan varios owners antes de hacer batch.

---

# 125. Streaming strategies

Posibles modos:

### JOIN streaming

Si el graph puede finalizarse por root boundary.

### Windowed Select-In

Leer `N` roots, eager-load sus relaciones y después yield.

### Full buffering

Resolver todos los roots primero.

---

# 126. Windowed eager streaming

```text
Root cursor
    ↓
buffer 100 Users
    ↓
load posts for 100
    ↓
finalize
    ↓
yield 100
    ↓
next window
```

---

# 127. Window size

Debe ser resource-policy-driven.

---

# 128. Streaming guarantee

Antes de yield:

```text
all eager-required relationships
for that root
must satisfy requested guarantee
```

---

# 129. Safe Yield

```text
SafeYield(root)
⇔
RootHydrated
∧
RequiredEagerSubgraph(root).Finalized
```

---

# 130. Joined streaming

Con JOIN to-many:

```text
User#1 + Post#1
User#1 + Post#2
User#2 + Post#3
```

`User#1` solo puede yield cuando se sabe que no habrá otra fila de `User#1`.

---

# 131. Ordering requirement

Por ello puede requerirse:

```text
ORDER BY root identity
```

o una garantía equivalente del plan.

---

# 132. No safe boundary

Si no puede demostrarse:

```text
buffer
```

o rechazar streaming eager.

---

# 133. Pagination + Eager Loading

Root pagination debe resolverse antes de que relaciones to-many modifiquen el row shape físico.

---

# 134. Cursor Pagination

Cursor identity/order columns deben permanecer estables.

El eager fetch no debe modificar el cursor semántico.

---

# 135. Count Query

Un paginator puede ejecutar:

```text
COUNT root
```

El eager graph no deberá contaminar ese count innecesariamente.

---

# 136. Eager relations not needed for count

Por default:

```text
PaginationCountQuery
```

no carga relaciones.

---

# 137. Unless root predicate depends on relation

Si la relación forma parte del filtro raíz:

```text
whereHas(...)
```

eso pertenece a query semantics, no eager loading.

---

# 138. whereHas ≠ with

Fundamental:

```text
whereHas('posts')
```

afecta qué users son raíz.

```text
with('posts')
```

afecta qué relaciones vienen cargadas.

---

# 139. Combining both

```php
User::query()
    ->whereHas('posts', ...)
    ->with('posts')
    ->get();
```

son dos requisitos semánticos distintos aunque puedan compartir optimizaciones.

---

# 140. Optimization sharing

El planner puede detectar oportunidades de reutilización.

Pero:

> compartir ejecución no debe fusionar semánticas diferentes.

---

# 141. Read Consistency

Todo stage eager deberá heredar o derivar una política compatible de consistencia.

---

# 142. Snapshot inconsistency risk

Ejemplo:

```text
Query 1 → Users
DB changes
Query 2 → Posts
```

Sin transacción/snapshot, el graph puede reflejar instantes distintos.

---

# 143. Eager consistency levels

Conceptualmente:

```php
enum EagerReadConsistency
{
    case DEFAULT;
    case SAME_TRANSACTION;
    case SNAPSHOT_REQUIRED;
    case READ_YOUR_WRITES;
    case EVENTUAL_ALLOWED;
}
```

---

# 144. Multi-query eager loading

Debe documentar que:

```text
multiple queries
```

no implican automáticamente una snapshot consistente.

---

# 145. Strong consistency request

Puede requerir:

```text
transaction
+
appropriate isolation
```

según DB/capabilities.

---

# 146. Eager Loader no crea transaction arbitraria

El loader puede declarar requerimientos.

Transaction Manager decide cómo satisfacerlos.

---

# 147. Existing transaction

Si ya existe una transacción:

```text
all eager stages
```

deberán respetar el mismo transaction context cuando corresponda.

---

# 148. Replica routing

Un stage no debe terminar accidentalmente en una replica incompatible con el root result.

---

# 149. Routing compatibility

El load plan deberá transportar:

```text
ReadIntent
ConsistencyRequirement
TransactionContext
StickyRequirement
```

---

# 150. Multi-Tenant Safety

Todo eager stage deberá preservar:

```text
TenantContext
```

del root operation.

---

# 151. Tenant propagation

```text
Root Query Tenant A
        ↓
Eager Stage 1 Tenant A
        ↓
Eager Stage 2 Tenant A
```

---

# 152. No tenant inference from row only

El tenant efectivo no deberá depender únicamente de una columna retornada.

Debe provenir del contexto autorizado de la operación.

---

# 153. Sharding

Owners de shards diferentes deberán separarse en batches compatibles.

---

# 154. Cross-shard relationships

Solo serán válidas cuando una extensión explícita defina su semántica.

---

# 155. Default cross-shard policy

```text
FORBID
```

para relaciones ORM normales.

---

# 156. Cancellation

La operación eager completa deberá compartir:

```text
CancellationToken
```

---

# 157. Cancellation between stages

Si:

```text
root loaded
posts loaded
roles pending
CANCEL
```

la eager operation no podrá afirmar éxito.

---

# 158. Root entities after cancellation

Entidades ya correctamente hidratadas pueden permanecer en el PersistenceContext.

Pero:

```text
requested eager graph
```

no fue completado.

---

# 159. No fake rollback

No deberá fingirse que las entidades ya observadas nunca existieron.

---

# 160. Deadline

Cada stage deberá comprobar el deadline antes de iniciar trabajo costoso.

---

# 161. Deadline-aware planning

El planner puede elegir una estrategia con menor round-trip count si es semánticamente segura.

Pero no sacrificar correctness para cumplir deadline.

---

# 162. Failure Model

Posibles estados:

```php
enum EagerLoadOutcomeStatus
{
    case SUCCESS;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 163. Stage outcomes

Cada stage debe conservar outcome independiente.

---

# 164. Example

```text
Root Users     SUCCESS
Profiles       SUCCESS
Posts          SUCCESS
Roles          FAILED
```

Global:

```text
FAILED / PARTIAL
```

según política de operación.

---

# 165. Required vs Optional Fetch Node

Puede existir distinción:

```php
enum EagerRequirementStrength
{
    case REQUIRED;
    case OPTIONAL;
}
```

---

# 166. Default

`with()` debe significar:

```text
REQUIRED for successful query result delivery
```

en el sentido de fetch guarantee.

No “best effort” silencioso.

---

# 167. Optional enrichment

Si se desea:

```text
best effort relationship enrichment
```

deberá ser una API explícita diferente.

---

# 168. Resource Governance

```php
final readonly class EagerLoadResourcePolicy
{
    public function __construct(
        public int $maxFetchDepth,
        public int $maxFetchNodes,
        public int $maxOwnersPerBatch,
        public int $maxBufferedRoots,
        public int $maxBufferedMembers,
        public int $maxQueries,
        public float $maxEstimatedJoinAmplification,
    ) {}
}
```

---

# 169. Query budget

Un graph exagerado:

```text
100 nested relationships
```

no deberá generar queries ilimitadas.

---

# 170. Query budget exhaustion

Debe:

- fallar;
- o requerir estrategia alternativa segura.

Nunca omitir relaciones requeridas y devolver `SUCCESS`.

---

# 171. Depth budget

Fetch graph:

```text
A.b.c.d.e.f...
```

deberá validarse antes de ejecución cuando sea posible.

---

# 172. Graph cycle

Ejemplo:

```text
User.posts.author.posts.author...
```

debe detectarse estructuralmente.

---

# 173. Explicit finite revisit

Un mismo relationship type puede aparecer en ramas finitas legítimas.

Cycle detection no debe confundir:

```text
repeated type
```

con:

```text
infinite graph
```

---

# 174. Max graph complexity

Podrá calcularse:

```text
GraphComplexity
=
Nodes
+
DepthWeight
+
ToManyWeight
+
PolymorphicBranchWeight
```

para governance.

---

# 175. Strategy Cost Model

Una futura implementación podrá estimar:

```text
Cost(strategy)
=
RoundTrips
+
RowsTransferred
+
RepeatedRootBytes
+
HydrationWork
+
AssemblyMemory
+
ParameterPressure
```

---

# 176. AUTO Strategy Selection

Conceptualmente:

```text
ChosenStrategy
=
argmin SafeStrategies Cost(strategy)
```

---

# 177. SafeStrategies

Solo estrategias que satisfagan:

```text
Semantics
Capabilities
Consistency
ResourcePolicy
StreamingRequirements
```

participan.

---

# 178. Cost never overrides correctness

Una estrategia más barata que cambia root cardinality es inválida.

---

# 179. N+1 Elimination

Eager loading debe permitir transformar:

```text
1 root query
+
N relationship queries
```

en algo cercano a:

```text
1 root query
+
K bounded eager queries
```

donde:

```text
K << N
```

---

# 180. N+1 elimination ≠ one query

Objetivo:

```text
bounded predictable query count
```

no necesariamente:

```text
exactly one SQL statement
```

---

# 181. Query count model

Para un fetch graph con Select-In:

```text
Queries
≈
1 root
+
Σ batches_per_fetch_node
```

---

# 182. Nested graph query count

Con owner counts pequeños y sin chunking:

```text
Queries
≈
1 + number_of_secondary_fetch_nodes
```

---

# 183. Batch limits

Con chunking:

```text
Queries(node)
=
ceil(
    OwnerCount(node)
    /
    OwnersPerBatch(node)
)
```

---

# 184. Telemetry

Métricas:

```text
database.orm.eager.operations
database.orm.eager.fetch_nodes
database.orm.eager.fetch_depth
database.orm.eager.queries

database.orm.eager.join_loads
database.orm.eager.select_in_loads
database.orm.eager.batch_loads
database.orm.eager.polymorphic_loads

database.orm.eager.root_entities
database.orm.eager.related_entities
database.orm.eager.identity_map_hits
database.orm.eager.identity_map_misses

database.orm.eager.join_amplification
database.orm.eager.batch_size
database.orm.eager.buffered_roots
database.orm.eager.buffered_members

database.orm.eager.partial_loads
database.orm.eager.failures
database.orm.eager.cancellations
```

---

# 185. Strategy telemetry

Debe ser posible saber:

```text
requested strategy
effective strategy
fallback reason
```

---

# 186. Diagnostics

Ejemplo:

```text
EAGER LOAD PLAN

Root:
    User

Root Query:
    paginated

Fetch Graph:
    profile
    posts.comments
    roles

Strategies:

    User.profile
        JOIN

    User.posts
        SELECT_IN

    Post.comments
        SELECT_IN

    User.roles
        SELECT_IN

Estimated Queries:
    4

Joined To-Many:
    0

Estimated Amplification:
    1.0

Streaming:
    WINDOWED

Window Size:
    100

Consistency:
    READ_YOUR_WRITES
```

---

# 187. Explain Plan

Debe existir conceptualmente:

```php
$plan->explain();
```

o:

```php
$eagerLoadExplainer->explain($plan);
```

---

# 188. No PII

El explain plan no deberá imprimir:

- IDs concretos;
- nombres;
- emails;
- valores de parámetros sensibles.

---

# 189. EagerLoadPlan Fingerprint

Conceptualmente:

```text
Fingerprint
=
H(
    RootQueryShape,
    FetchGraphShape,
    RelationshipMetadataGeneration,
    EntityMetadataGeneration,
    HydrationPlanGeneration,
    TypeRegistryGeneration,
    PlatformCapabilityGeneration,
    EagerPolicyGeneration,
    CompilerVersion
)
```

---

# 190. Runtime values excluded

Normalmente no incluir:

```text
owner IDs
query parameter values
current tenant object
EntityManager instance
```

si no modifican el shape.

---

# 191. Cacheable plan

```text
Cacheable
⇔
Immutable
∧
Deterministic
∧
NoScopedMutableState
```

---

# 192. Cached eager plan ≠ cached relationship data

El cache contiene:

```text
how to load
```

no:

```text
which entities were loaded
```

---

# 193. Persistent Runtime Safety

FrankenPHP puede reutilizar:

```text
EagerLoadPlan
FetchGraph
Compiled Relationship Metadata
HydrationPlanTemplate
Strategy Definitions
```

---

# 194. Must remain scoped

Nunca compartir:

```text
owners
owner IDs
current batches
EntityManager
IdentityMap
UnitOfWork
RelationshipLoadSession
ResultCursor
current tenant
current transaction
collection state
```

---

# 195. Request cleanup

Al terminar una operación/request:

```text
temporary eager load state
→ released
```

---

# 196. No static current graph

Rechazado:

```php
EagerLoader::$currentFetchGraph;
```

---

# 197. No static batch owners

Rechazado:

```php
EagerLoader::$owners;
```

---

# 198. FrankenPHP

La arquitectura deberá funcionar con múltiples requests secuenciales sobre el mismo worker sin contaminación.

---

# 199. RoadRunner

Misma regla.

---

# 200. OpenSwoole

Misma regla, con requisitos adicionales cuando exista concurrencia/coroutines.

---

# 201. Concurrency

El mismo mutable `PersistenceContext` no deberá modificarse concurrentemente por múltiples eager stages en V1.

---

# 202. Parallel future execution

Una futura implementación podría ejecutar:

```text
posts
roles
teams
```

en paralelo si:

- contexts son seguros;
- connections son compatibles;
- reconciliation es coordinada.

---

# 203. V1 recommendation

```text
deterministic staged execution
```

antes que complejidad prematura.

---

# 204. Public API

API mínima:

```php
User::query()
    ->with('profile')
    ->get();
```

---

# 205. Multiple relations

```php
User::query()
    ->with([
        'profile',
        'posts',
        'roles',
    ])
    ->get();
```

---

# 206. Nested

```php
User::query()
    ->with([
        'posts.comments.author',
        'roles.permissions',
    ])
    ->get();
```

---

# 207. Constrained

```php
User::query()
    ->with([
        'posts' => fn ($query) =>
            $query->where('published', true),
    ])
    ->get();
```

---

# 208. Strategy hint

Posible API avanzada:

```php
User::query()
    ->with(
        'posts',
        strategy: FetchStrategy::SELECT_IN,
    )
    ->get();
```

---

# 209. Fetch profile

Para graphs reutilizables:

```php
User::query()
    ->fetchProfile(UserFetchProfile::DETAIL)
    ->get();
```

---

# 210. Named fetch profile

Ejemplo:

```text
UserFetchProfile::DETAIL

User
├── profile
├── roles
└── posts
    ├── tags
    └── comments
```

---

# 211. Fetch Profile ≠ Entity Metadata default eager

Un profile es una selección reutilizable.

No obliga a que toda query de User cargue ese graph.

---

# 212. Default eager metadata

VoltStack puede soportarlo, pero debe utilizarse con cautela.

---

# 213. Recommendation

Preferir:

```text
LAZY / EXPLICIT defaults
+
query-specific eager graphs
+
named fetch profiles
```

sobre graph eager global masivo.

---

# 214. Error Taxonomy

```text
DatabaseEagerLoadingException
├── EagerLoadRequestException
├── EagerFetchGraphException
├── EagerFetchGraphValidationException
├── EagerFetchGraphDepthException
├── EagerFetchGraphCycleException
├── EagerFetchGraphResourceException
├── EagerLoadPlanningException
├── EagerLoadStrategyException
├── UnsupportedEagerLoadStrategyException
├── RequiredEagerStrategyUnavailableException
├── EagerJoinExplosionException
├── EagerRootSemanticViolationException
├── EagerPaginationSafetyException
├── EagerProjectionException
├── MissingEagerOwnerKeyException
├── EagerBatchException
├── EagerBatchParameterLimitException
├── EagerLoadAssemblyException
├── EagerLoadCoverageException
├── EagerLoadConsistencyException
├── EagerLoadMergeConflictException
├── EagerPolymorphicLoadException
├── EagerStreamingException
├── EagerStreamingBoundaryException
├── EagerLoadCancellationException
├── EagerLoadDeadlineException
├── EagerLoadResourceLimitException
├── EagerLoadRuntimeIsolationException
└── EagerLoadingInvariantException
```

---

# 215. Root Semantic Violation

Ejemplo de diagnóstico:

```text
EagerRootSemanticViolationException

Eager strategy JOIN for User.posts would alter the
logical root pagination window.

Requested root cardinality:
    25 users

Strategy:
    JOIN

Resolution:
    use SELECT_IN
```

---

# 216. Join Explosion Diagnostic

```text
EagerJoinExplosionException

Fetch graph contains multiple to-many JOIN requirements:

    User.posts
    User.roles
    User.teams

Estimated row amplification exceeds configured policy.
```

---

# 217. Testing Strategy

El sistema deberá probarse independientemente del SQL textual cuando sea posible.

---

# 218. Fetch Graph Tests

Validar:

- single relationship;
- multiple relationships;
- nested paths;
- prefix dedup;
- invalid path;
- cycles;
- max depth;
- polymorphic branches.

---

# 219. Planner Tests

Validar:

```text
to-one → JOIN candidate
to-many → SELECT_IN candidate
pagination + to-many → safe secondary strategy
multiple to-many → no uncontrolled join explosion
```

---

# 220. Root Semantics Tests

La misma root query:

```text
without eager
```

y:

```text
with eager
```

debe conservar:

- root identities;
- root count;
- root ordering;
- pagination window.

---

# 221. Identity Tests

Verificar que:

```text
same EntityKey
→ same managed object
```

entre root y eager results.

---

# 222. Dirty Entity Tests

Eager loading no sobrescribe dirty managed target.

---

# 223. Dirty Relationship Tests

Eager loading no destruye local collection additions/removals.

---

# 224. Coverage Tests

Probar:

- full load → COMPLETE;
- ad-hoc filter → PARTIAL;
- LIMIT → PARTIAL;
- failed chunk → not COMPLETE;
- zero rows full owner → COMPLETE empty.

---

# 225. Select-In Tests

Probar:

- single ID;
- composite ID;
- parameter chunking;
- zero owners;
- duplicate owners;
- tenant separation;
- shard separation.

---

# 226. Nested Tests

Probar:

```text
User.posts.comments.author
```

incluyendo branches vacías.

---

# 227. Polymorphic Tests

Probar:

- multiple morph types;
- unknown type;
- nested type-specific fetch graph;
- canonical identity reuse.

---

# 228. Pagination Tests

Probar:

- offset pagination;
- cursor pagination;
- root ordering;
- count query unaffected.

---

# 229. Streaming Tests

Probar:

- joined safe boundary;
- windowed Select-In;
- cancellation mid-window;
- no yield before eager graph complete.

---

# 230. Failure Tests

Probar:

```text
root succeeds
secondary query fails
```

sin devolver eager success falso.

---

# 231. Persistent Runtime Tests

Request A:

```text
Tenant A
User#1.posts
```

Request B en mismo worker:

```text
Tenant B
User#1.posts
```

Nunca compartir:

- entities;
- owners;
- relationship state;
- batches.

---

# 232. Plan Cache Tests

Verificar:

- immutable plan reuse;
- metadata generation invalidation;
- same query shape → same fingerprint;
- owner values no cambian fingerprint;
- no scoped references en cache.

---

# 233. Performance Tests

Medir:

```text
JOIN
vs
SELECT_IN
vs
BATCH
```

en:

- low cardinality;
- high cardinality;
- multiple to-many;
- nested graphs;
- large owner sets.

---

# 234. Architectural Invariants

## DB-ORM-EAGER-001

Eager loading no implicará JOIN.

## DB-ORM-EAGER-002

Eager loading será una especialización de Relationship Loading.

## DB-ORM-EAGER-003

Eager loading no será Lazy Loading.

## DB-ORM-EAGER-004

Eager loading no cargará automáticamente todo el entity graph.

## DB-ORM-EAGER-005

Fetch paths se resolverán contra Relationship Metadata.

## DB-ORM-EAGER-006

String fetch paths no serán el modelo interno final.

## DB-ORM-EAGER-007

Fetch Graph será tipado y validado.

## DB-ORM-EAGER-008

Fetch Graph será finito.

## DB-ORM-EAGER-009

Fetch Graph deduplicará prefixes equivalentes.

## DB-ORM-EAGER-010

AUTO seleccionará estrategia mediante planner.

## DB-ORM-EAGER-011

Strategy hint será distinta de semantic requirement.

## DB-ORM-EAGER-012

JOIN podrá rechazarse si es inseguro.

## DB-ORM-EAGER-013

To-one JOIN será heurística, no invariant.

## DB-ORM-EAGER-014

To-many SELECT_IN será heurística, no invariant.

## DB-ORM-EAGER-015

EagerLoadPlanner no generará SQL.

## DB-ORM-EAGER-016

EagerLoadPlan será semántico.

## DB-ORM-EAGER-017

Nested fetch graph producirá dependency stages.

## DB-ORM-EAGER-018

Child stage no ejecutará antes de conocer owners.

## DB-ORM-EAGER-019

Planning será determinista.

## DB-ORM-EAGER-020

Eager loading preservará root semantics.

## DB-ORM-EAGER-021

Eager loading no agregará logical root entities.

## DB-ORM-EAGER-022

Eager loading no eliminará logical root entities.

## DB-ORM-EAGER-023

Eager loading no duplicará logical root entities.

## DB-ORM-EAGER-024

Eager loading preservará root ordering.

## DB-ORM-EAGER-025

Eager loading preservará pagination window.

## DB-ORM-EAGER-026

Relationship-only predicates no modificarán root semantics accidentalmente.

## DB-ORM-EAGER-027

JOIN + pagination requerirá prueba de seguridad semántica.

## DB-ORM-EAGER-028

Multiple to-many JOINs deberán evaluar amplification.

## DB-ORM-EAGER-029

Unknown amplification no será asumida baja.

## DB-ORM-EAGER-030

Hybrid strategies estarán permitidas.

## DB-ORM-EAGER-031

Una operación eager podrá usar múltiples queries.

## DB-ORM-EAGER-032

Select-In utilizará canonical owner identities.

## DB-ORM-EAGER-033

Composite owner identifiers estarán soportados.

## DB-ORM-EAGER-034

Parameter limits serán capability-driven.

## DB-ORM-EAGER-035

Chunking no implicará partial coverage si todos los chunks terminan.

## DB-ORM-EAGER-036

Failed chunk impedirá COMPLETE.

## DB-ORM-EAGER-037

Owners sin rows podrán quedar COMPLETE empty.

## DB-ORM-EAGER-038

Nested branches vacías evitarán queries innecesarias.

## DB-ORM-EAGER-039

Owners podrán deduplicarse por EntityKey para batching.

## DB-ORM-EAGER-040

Owner batching dedup no cambiará root cardinality.

## DB-ORM-EAGER-041

IdentityMap será autoridad de identidad canónica.

## DB-ORM-EAGER-042

Existing managed target será reutilizado.

## DB-ORM-EAGER-043

Dirty target no será sobrescrito arbitrariamente.

## DB-ORM-EAGER-044

Eager loading no equivaldrá a refresh.

## DB-ORM-EAGER-045

Complete existing relation podrá evitar query redundante.

## DB-ORM-EAGER-046

Partial existing relation requerirá completion si fetch guarantee lo exige.

## DB-ORM-EAGER-047

Dirty relationship state respetará merge policy.

## DB-ORM-EAGER-048

Local relationship mutations no se perderán silenciosamente.

## DB-ORM-EAGER-049

Mapping constraint será distinto de ad-hoc eager constraint.

## DB-ORM-EAGER-050

Ad-hoc constrained load no implicará complete relationship.

## DB-ORM-EAGER-051

LIMIT per relationship será distinto de global LIMIT.

## DB-ORM-EAGER-052

Limited relationship result normalmente será PARTIAL.

## DB-ORM-EAGER-053

Relationship ordering definido por metadata será preservado.

## DB-ORM-EAGER-054

Multi-batch ordering será reconciliado correctamente.

## DB-ORM-EAGER-055

Many-to-many eager loading respetará association metadata.

## DB-ORM-EAGER-056

Polymorphic eager loading agrupará por concrete type.

## DB-ORM-EAGER-057

Polymorphic loading usará morph whitelist.

## DB-ORM-EAGER-058

Unknown morph type fallará explícitamente.

## DB-ORM-EAGER-059

Nested polymorphic branches podrán ser type-specific.

## DB-ORM-EAGER-060

Eager projections conservarán owner identity necesaria.

## DB-ORM-EAGER-061

Internal hydration columns no modificarán public result shape.

## DB-ORM-EAGER-062

Eager loading no requerirá SELECT *.

## DB-ORM-EAGER-063

Partial managed entity hydration seguirá Hydration rules.

## DB-ORM-EAGER-064

DTO/projection será preferible para lecturas altamente parciales.

## DB-ORM-EAGER-065

Aggregate eager data no inicializará relationship members.

## DB-ORM-EAGER-066

Known count será distinto de known members.

## DB-ORM-EAGER-067

Known exists será distinto de complete coverage.

## DB-ORM-EAGER-068

EagerLoadPlan podrá coordinar múltiples HydrationPlans.

## DB-ORM-EAGER-069

HydrationPlan será reutilizado, no duplicado.

## DB-ORM-EAGER-070

Eager hot path no dependerá de reflection discovery.

## DB-ORM-EAGER-071

Eager operation terminará después de graph finalization.

## DB-ORM-EAGER-072

Todos los required fetch nodes deberán finalizar antes de SUCCESS.

## DB-ORM-EAGER-073

Buffered result no escapará antes de eager finalization.

## DB-ORM-EAGER-074

Streaming requerirá eager-safe yield boundary.

## DB-ORM-EAGER-075

Windowed Select-In estará permitido.

## DB-ORM-EAGER-076

Window size será resource-governed.

## DB-ORM-EAGER-077

Joined streaming requerirá root completion boundary.

## DB-ORM-EAGER-078

Unsafe streaming requerirá buffering o rejection.

## DB-ORM-EAGER-079

Root pagination se resolverá independientemente de to-many row multiplication.

## DB-ORM-EAGER-080

Paginator count no cargará eager relationships innecesariamente.

## DB-ORM-EAGER-081

whereHas será distinto de with.

## DB-ORM-EAGER-082

Query filtering semantics serán distintas de fetch semantics.

## DB-ORM-EAGER-083

Execution sharing no fusionará semánticas incompatibles.

## DB-ORM-EAGER-084

Multi-query eager load no implicará snapshot consistency automática.

## DB-ORM-EAGER-085

Consistency requirement será explícito.

## DB-ORM-EAGER-086

Eager loader no creará transaction arbitraria.

## DB-ORM-EAGER-087

Existing transaction context será respetado.

## DB-ORM-EAGER-088

Secondary stages respetarán compatible read routing.

## DB-ORM-EAGER-089

Tenant context se propagará a todos los stages.

## DB-ORM-EAGER-090

Batches no mezclarán tenants incompatibles.

## DB-ORM-EAGER-091

Batches no mezclarán shards incompatibles.

## DB-ORM-EAGER-092

Cross-shard relationship requerirá extensión explícita.

## DB-ORM-EAGER-093

Cancellation será compartida por la eager operation.

## DB-ORM-EAGER-094

Cancelled operation no devolverá SUCCESS.

## DB-ORM-EAGER-095

Correctly hydrated entities no serán ficticiamente reverted por eager failure.

## DB-ORM-EAGER-096

Deadline será respetado por cada stage.

## DB-ORM-EAGER-097

Correctness no se sacrificará para cumplir deadline.

## DB-ORM-EAGER-098

Stage outcomes serán explícitos.

## DB-ORM-EAGER-099

Required fetch failure fallará la eager guarantee.

## DB-ORM-EAGER-100

Best-effort enrichment requerirá semántica explícita diferente.

## DB-ORM-EAGER-101

Fetch graph estará sujeto a resource governance.

## DB-ORM-EAGER-102

Query budget exhaustion no omitirá relaciones silenciosamente.

## DB-ORM-EAGER-103

Depth budget será validado.

## DB-ORM-EAGER-104

Cycle detection será distinto de repeated entity type detection.

## DB-ORM-EAGER-105

Strategy cost model solo comparará estrategias semánticamente válidas.

## DB-ORM-EAGER-106

Cost no podrá superar correctness.

## DB-ORM-EAGER-107

N+1 elimination no significará single-query requirement.

## DB-ORM-EAGER-108

Query count será observable.

## DB-ORM-EAGER-109

Effective strategy será observable.

## DB-ORM-EAGER-110

Fallback reason será observable.

## DB-ORM-EAGER-111

Diagnostics no expondrán PII.

## DB-ORM-EAGER-112

Eager plans podrán ser cacheables si son inmutables.

## DB-ORM-EAGER-113

Cached plan no almacenará entity instances.

## DB-ORM-EAGER-114

Cached plan no almacenará owner IDs runtime.

## DB-ORM-EAGER-115

Cached plan no almacenará EntityManager.

## DB-ORM-EAGER-116

Cached plan no almacenará IdentityMap.

## DB-ORM-EAGER-117

Cached plan no almacenará tenant mutable context.

## DB-ORM-EAGER-118

Immutable eager plans podrán compartirse en persistent workers.

## DB-ORM-EAGER-119

Mutable eager execution state será operation-scoped.

## DB-ORM-EAGER-120

No existirá static current fetch graph.

## DB-ORM-EAGER-121

No existirá static current owner batch.

## DB-ORM-EAGER-122

Request cleanup liberará eager execution state.

## DB-ORM-EAGER-123

Same mutable PersistenceContext no será concurrent-safe por defecto.

## DB-ORM-EAGER-124

V1 favorecerá deterministic staged execution.

## DB-ORM-EAGER-125

Public API será simple y compilada a estructuras tipadas.

## DB-ORM-EAGER-126

Named fetch profiles estarán permitidos.

## DB-ORM-EAGER-127

Fetch profile no será global mandatory eager loading.

## DB-ORM-EAGER-128

Global eager defaults deberán usarse conservadoramente.

## DB-ORM-EAGER-129

Eager Loading no generará SQL.

## DB-ORM-EAGER-130

Eager Loading no usará PDO directamente.

## DB-ORM-EAGER-131

Eager Loading no ejecutará Driver directamente.

## DB-ORM-EAGER-132

Eager Loading no hará flush.

## DB-ORM-EAGER-133

Eager Loading no hará commit.

## DB-ORM-EAGER-134

Eager Loading no hará persist.

## DB-ORM-EAGER-135

Eager Loading no implementará segunda IdentityMap.

## DB-ORM-EAGER-136

Eager Loading no implementará segundo Hydration Engine.

## DB-ORM-EAGER-137

Eager Loading no implementará segundo Query Builder.

## DB-ORM-EAGER-138

Eager Loading no confundirá physical rows con logical entities.

## DB-ORM-EAGER-139

Eager Loading preservará Relationship Coverage semantics.

## DB-ORM-EAGER-140

Eager Loading preservará LoadedFieldMask semantics.

## DB-ORM-EAGER-141

Eager Loading preservará Relationship Snapshot semantics.

## DB-ORM-EAGER-142

Eager Loading preservará UnitOfWork state.

## DB-ORM-EAGER-143

Eager Loading preservará EntityState.

## DB-ORM-EAGER-144

Eager Loading preservará canonical EntityKey.

## DB-ORM-EAGER-145

Eager Loading preservará morph security.

## DB-ORM-EAGER-146

Eager Loading preservará root cardinality.

## DB-ORM-EAGER-147

Eager Loading preservará root ordering.

## DB-ORM-EAGER-148

Eager Loading preservará root pagination.

## DB-ORM-EAGER-149

Eager Loading preservará local changes.

## DB-ORM-EAGER-150

Eager Loading preservará evidence-based completeness.

## DB-ORM-EAGER-151

Eager Loading preservará deterministic cleanup.

## DB-ORM-EAGER-152

Eager Loading preservará persistent-runtime isolation.

## DB-ORM-EAGER-153

Eager Loading será testeable mediante semantic plans.

## DB-ORM-EAGER-154

Eager Loading será explicable sin inspeccionar SQL textual.

## DB-ORM-EAGER-155

Eager Loading utilizará capability checks, no vendor conditionals.

## DB-ORM-EAGER-156

MySQL y MariaDB podrán exponer capabilities distintas.

## DB-ORM-EAGER-157

Platform capability UNKNOWN no será asumida supported.

## DB-ORM-EAGER-158

Unsupported required strategy fallará explícitamente.

## DB-ORM-EAGER-159

Fallback solo ocurrirá cuando la API/policy lo permita.

## DB-ORM-EAGER-160

El eager graph entregado como exitoso cumplirá todas sus garantías requeridas.

---

# 235. Anti-Patterns

## 235.1 `with()` = JOIN

```php
if ($query->hasWith()) {
    $sql .= ' JOIN ...';
}
```

**Rechazado.**

---

## 235.2 JOIN de todas las relaciones

```text
User
JOIN posts
JOIN comments
JOIN tags
JOIN roles
JOIN teams
```

**Rechazado como estrategia general.**

---

## 235.3 Pagination después de row multiplication

**Rechazado.**

---

## 235.4 Constrained eager = complete collection

**Rechazado.**

---

## 235.5 `withCount()` inicializa colección

**Rechazado.**

---

## 235.6 Query por owner

```text
foreach User:
    load Posts
```

como implementación eager estándar:

**Rechazado.**

---

## 235.7 Ignorar IdentityMap

**Rechazado.**

---

## 235.8 Sobrescribir dirty graph

**Rechazado.**

---

## 235.9 SELECT * obligatorio

**Rechazado.**

---

## 235.10 Compartir owner batches entre requests

**Rechazado.**

---

# 236. Estructura de directorios

```text
src/Quantum/Database/ORM/Relationship/Eager/
│
├── Contract/
│   ├── EagerLoadPlanner.php
│   ├── EagerLoadExecutor.php
│   ├── EagerLoadStrategyResolver.php
│   └── EagerLoadExplainer.php
│
├── Requirement/
│   ├── EagerLoadRequirement.php
│   ├── EagerLoadPolicy.php
│   ├── EagerRequirementStrength.php
│   ├── FetchStrategyStrength.php
│   └── RelationshipAggregateRequirement.php
│
├── Graph/
│   ├── RelationshipFetchGraph.php
│   ├── RelationshipFetchNode.php
│   ├── RelationshipPath.php
│   ├── FetchGraphFingerprint.php
│   ├── FetchGraphNormalizer.php
│   └── FetchGraphValidator.php
│
├── Plan/
│   ├── EagerLoadPlan.php
│   ├── EagerLoadPlanFingerprint.php
│   ├── EagerLoadStage.php
│   ├── EagerLoadStageId.php
│   ├── EagerLoadStageDependency.php
│   ├── EagerLoadRequirements.php
│   └── RootLoadPlan.php
│
├── Strategy/
│   ├── FetchStrategy.php
│   ├── JoinEagerLoadStrategy.php
│   ├── SelectInEagerLoadStrategy.php
│   ├── BatchEagerLoadStrategy.php
│   ├── PolymorphicEagerLoadStrategy.php
│   └── HybridEagerLoadStrategy.php
│
├── Planning/
│   ├── DefaultEagerLoadPlanner.php
│   ├── EagerLoadStagePlanner.php
│   ├── EagerLoadCostModel.php
│   ├── JoinExplosionAnalyzer.php
│   ├── ParameterBudgetCalculator.php
│   └── StreamingCompatibilityAnalyzer.php
│
├── Execution/
│   ├── DefaultEagerLoadExecutor.php
│   ├── EagerLoadExecutionContext.php
│   ├── EagerLoadStageExecutor.php
│   └── EagerLoadStageResult.php
│
├── Batch/
│   ├── EagerOwnerBatch.php
│   ├── EagerOwnerBatchKey.php
│   ├── EagerBatchPlanner.php
│   └── EagerBatchResultDistributor.php
│
├── Streaming/
│   ├── EagerStreamingPlan.php
│   ├── EagerStreamingWindow.php
│   └── EagerYieldBoundary.php
│
├── Consistency/
│   └── EagerReadConsistency.php
│
├── Resource/
│   ├── EagerLoadResourcePolicy.php
│   └── JoinExplosionPolicy.php
│
├── Profile/
│   ├── FetchProfile.php
│   ├── FetchProfileRegistry.php
│   └── FetchProfileCompiler.php
│
├── Outcome/
│   ├── EagerLoadOutcome.php
│   ├── EagerLoadOutcomeStatus.php
│   └── EagerLoadStageOutcome.php
│
├── Diagnostics/
│   ├── EagerLoadDiagnostics.php
│   └── DefaultEagerLoadExplainer.php
│
└── Exception/
    └── ...
```

---

# 237. Integración general

```text
                  Entity Query
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
Root Query Contract            Eager Requirements
                                      │
                                      ▼
                               Fetch Graph
                                      │
                                      ▼
                              EagerLoadPlanner
                                      │
                          ┌───────────┼───────────┐
                          ▼           ▼           ▼
                        JOIN      SELECT_IN     BATCH
                          │           │           │
                          └───────────┼───────────┘
                                      ▼
                             Relationship Loader
                                      │
                                      ▼
                                Query Engine
                                      │
                                      ▼
                               Execution Engine
                                      │
                                      ▼
                                  Results
                                      │
                                      ▼
                              Hydration System
                                      │
                                      ▼
                                 IdentityMap
                                      │
                                      ▼
                          Relationship Assembly
                                      │
                                      ▼
                          Relationship Coverage
                                      │
                                      ▼
                           Fetch Graph Finalizer
                                      │
                                      ▼
                            Application Result
```

---

# 238. Fórmulas fundamentales

## Eager Loading

```text
EagerLoad
=
FetchRequirement
+
SafeStrategyPlan
+
RelationshipExecution
+
Hydration
+
Assembly
+
Finalization
```

## Root preservation

```text
LogicalRoot(EagerQuery)
=
LogicalRoot(QueryWithoutFetchGraph)
```

salvo semántica explícita contraria.

## Join amplification

```text
EstimatedAmplification
≈
Π EstimatedToManyCardinality_i
```

## Select-In query count

```text
Queries(node)
=
ceil(
    Owners(node)
    /
    OwnersPerBatch(node)
)
```

## Graph completion

```text
GraphComplete
⇔
∀ node ∈ RequiredFetchGraph:
    NodeFinalized(node)
```

## Safe eager success

```text
EagerSuccess
=
RootSemanticsPreserved
∧
RequiredGraphComplete
∧
IdentityCanonical
∧
CoverageValid
∧
LocalChangesPreserved
∧
ConsistencyRequirementsSatisfied
```

## Safe streaming

```text
SafeYield(root)
=
RootComplete
∧
RequiredEagerSubgraph(root).Complete
```

## Strategy selection

```text
Strategy
=
argmin(
    Cost(SafeStrategies)
)
```

---

# 239. Master Formula

```text
Database Eager Loading System
=
Eager Requirements
+
Typed Relationship Paths
+
Fetch Graphs
+
Fetch Profiles
+
Strategy Policies
+
Eager Load Planner
+
Stage DAG
+
JOIN Loading
+
SELECT-IN Loading
+
Batch Loading
+
Hybrid Loading
+
Polymorphic Loading
+
Nested Loading
+
Root Semantic Preservation
+
Pagination Safety
+
Cartesian Amplification Protection
+
Owner Grouping
+
Parameter Budgeting
+
Hydration Plan Composition
+
IdentityMap Reconciliation
+
Relationship Assembly
+
Coverage Semantics
+
Dirty-State Merge
+
Projection Support
+
Aggregate Relationship Metadata
+
Streaming Windows
+
Consistency Requirements
+
Tenant/Shard Isolation
+
Cancellation
+
Deadlines
+
Resource Governance
+
Plan Caching
+
Telemetry
+
Diagnostics
+
Persistent Runtime Safety
+
Testing
```

---

# 240. Regla maestra final

> **Eager Loading en VoltStack será una garantía sobre el estado lógico del resultado entregado a la aplicación, no una instrucción sobre la forma del SQL.**

La aplicación podrá escribir:

```php
$users = User::query()
    ->with([
        'profile',
        'posts.comments.author',
        'roles.permissions',
    ])
    ->get();
```

mientras VoltStack podrá resolver internamente:

```text
Root Users
    │
    ├── profile
    │      JOIN
    │
    ├── posts
    │      SELECT_IN
    │        │
    │        └── comments
    │              SELECT_IN
    │                 │
    │                 └── author
    │                       SELECT_IN
    │
    └── roles
           SELECT_IN
              │
              └── permissions
                    BATCH
```

sin que la aplicación dependa de esas decisiones físicas.

La separación definitiva será:

```text
Application
    knows
    what graph it needs

Eager Loading
    knows
    how to plan graph acquisition

Relationship Loading
    knows
    how to load associations

Entity Query
    knows
    entity-aware query semantics

Query Engine
    knows
    queries

Compiler
    knows
    SQL dialect

Execution Engine
    knows
    execution

Hydration
    knows
    materialization

IdentityMap
    knows
    canonical identity

Relationship Assembly
    knows
    object graph membership
```

Con ello, VoltStack podrá optimizar agresivamente eager loading sin comprometer la semántica pública del ORM.

---

# 241. Siguiente documento

```text
152_DATABASE_LAZY_LOADING_SYSTEM.md
```

El siguiente documento deberá especializar `Relationship Loading` para cargas diferidas, definiendo:

- lazy references;
- lazy to-one;
- lazy collections;
- proxies;
- ghost objects;
- explicit lazy initializers;
- lazy loading policies;
- forbidden lazy loading;
- detached entities;
- closed PersistenceContext;
- hidden I/O;
- N+1 detection;
- recursive lazy loads;
- serialization hazards;
- logging/debugging hazards;
- lazy loading during lifecycle callbacks;
- transaction/read consistency;
- lazy collection operations;
- `EXTRA_LAZY`;
- count/contains/slice without full initialization;
- lazy initialization state machine;
- failure and retry semantics;
- cancellation;
- deadlines;
- persistent worker isolation;
- coroutine safety;
- telemetry;
- diagnostics.

Regla central propuesta:

> **Lazy Loading en VoltStack será una capacidad explícitamente gobernada para diferir la obtención de una relación hasta que su información sea requerida; nunca será una licencia para ejecutar I/O oculto desde cualquier objeto, contexto o fase del runtime.**