# 150_DATABASE_RELATIONSHIP_LOADING_SYSTEM.md

# VoltStack Quantum Database
## Database Relationship Loading System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 150 — Database Relationship Loading System  
**Bloque:** 13 — Relationships  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Relationship Loading System` define cómo VoltStack obtiene, materializa, ensambla y registra relaciones entre entidades cuando la información necesaria no está todavía disponible —o no está completamente disponible— dentro del `PersistenceContext`.

El sistema conecta:

```text
RelationshipMetadata
        +
Owner Entity
        +
PersistenceContext
        +
Relationship Load Policy
        ↓
RelationshipLoadRequest
        ↓
Relationship Loading Strategy
        ↓
Entity Query / Query Model
        ↓
Query Engine
        ↓
Execution Engine
        ↓
Result
        ↓
Hydration System
        ↓
IdentityMap Reconciliation
        ↓
Relationship Assembly
        ↓
Relationship State / Collection Coverage
```

El sistema deberá soportar:

- One-to-One;
- One-to-Many;
- Many-to-One;
- Many-to-Many;
- relaciones polimórficas;
- referencias `to-one`;
- colecciones `to-many`;
- carga explícita;
- lazy loading;
- eager loading;
- batch loading;
- join loading;
- select-in loading;
- subquery loading futuro;
- partial collections;
- `UNINITIALIZED / PARTIAL / COMPLETE`;
- IdentityMap;
- HydrationPlan;
- PersistentCollection;
- ciclos;
- deduplicación;
- read/write routing;
- cancellation;
- deadlines;
- resource governance;
- persistent runtimes.

---

# 2. Regla central

> **Relationship Loading materializa conocimiento de asociaciones mediante el Query Engine y lo integra al object graph preservando identidad canónica, cobertura, snapshots y semántica de hidratación; nunca convierte el acceso a una relación en SQL arbitrario fuera de una estrategia de carga explícitamente gobernada.**

Formalmente:

```text
Relationship Access
        ↓
Load Policy
        ↓
RelationshipLoadRequest
        ↓
RelationshipLoader
        ↓
Entity Query / Query Model
        ↓
Query Engine
        ↓
Execution
        ↓
Hydration
        ↓
Relationship Assembly
```

Nunca:

```text
$entity->relationship
        ↓
PDO::query(...)
```

---

# 3. Dirección arquitectónica

Relationship Persistence procesa:

```text
Object Graph
→ Database
```

Relationship Loading procesa:

```text
Database
→ Object Graph
```

Son sistemas complementarios pero independientes.

---

# 4. Posición dentro del ORM

```text
                  EntityManager
                       │
             ┌─────────┴─────────┐
             │                   │
          Repository       Relationship API
             │                   │
             └─────────┬─────────┘
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
                     Result
                       │
                       ▼
               Hydration System
                       │
              ┌────────┴─────────┐
              ▼                  ▼
          IdentityMap     Relationship Loader
                                  │
                                  ▼
                       Relationship Assembly
                                  │
                                  ▼
                         PersistentCollection
                                  │
                                  ▼
                       Relationship Snapshot
```

---

# 5. Relationship Loading ≠ Query Execution

Relationship Loading decide:

> Qué relación debe obtenerse y qué semántica de carga utilizar.

Execution Engine decide:

> Cómo ejecutar el `QueryModel` compilado.

---

# 6. Relationship Loading ≠ Hydration

Hydration convierte resultados físicos en resultados lógicos.

Relationship Loading coordina:

- qué relación se solicita;
- qué owners participan;
- qué estrategia usar;
- qué query lógico producir;
- cómo interpretar cobertura;
- cuándo una relación queda inicializada.

Por tanto:

```text
RelationshipLoader
uses
HydrationSystem
```

y no al contrario.

---

# 7. Relationship Loading ≠ Lazy Loading

Lazy loading es solamente una estrategia posible.

```text
Relationship Loading
├── Explicit
├── Lazy
├── Eager
├── Join
├── Select-In
├── Batch
└── Future strategies
```

---

# 8. Relationship Loading ≠ Eager Loading

Eager loading tampoco es el sistema completo.

Define cuándo cargar anticipadamente.

---

# 9. Relationship Loading ≠ Relationship Metadata

Metadata describe:

```text
User.posts
→ ONE_TO_MANY
→ target Post
→ mappedBy author
```

Loading utiliza esa información para construir una operación concreta.

---

# 10. Relationship Loading ≠ Relationship Persistence

Loading no genera:

```text
INSERT
UPDATE
DELETE
```

ni `RelationshipPersistenceOperation`.

---

# 11. Relationship Loading ≠ IdentityMap

El loader consulta y alimenta IdentityMap mediante los componentes ORM apropiados.

No implementa una segunda identidad.

---

# 12. Relationship Loading ≠ PersistentCollection

`PersistentCollection` mantiene estado runtime de una colección.

`RelationshipLoader` obtiene información para inicializarla o ampliarla.

---

# 13. Relationship Loading ≠ N+1 Detection

El loader puede emitir telemetría útil para detectar N+1.

El detector N+1 pertenece a otro sistema.

---

# 14. Concepto de Relationship Load Request

Toda carga deberá representarse explícitamente.

```php
final readonly class RelationshipLoadRequest
{
    /**
     * @param list<EntityReference> $owners
     */
    public function __construct(
        public RelationshipId $relationship,
        public array $owners,
        public RelationshipLoadMode $mode,
        public RelationshipLoadCoverage $requestedCoverage,
        public RelationshipLoadContext $context,
    ) {}
}
```

---

# 15. RelationshipLoadMode

```php
enum RelationshipLoadMode
{
    case EXPLICIT;
    case LAZY;
    case EAGER;
    case JOINED;
    case SELECT_IN;
    case BATCH;
}
```

---

# 16. Load Mode ≠ Fetch Policy

Fetch policy es configuración/intención.

Load mode es la estrategia efectiva seleccionada para una operación.

```text
ConfiguredFetchPolicy
        ↓
Strategy Resolution
        ↓
EffectiveLoadMode
```

---

# 17. Fetch Policy

Posible modelo:

```php
enum RelationshipFetchPolicy
{
    case LAZY;
    case EAGER;
    case EXPLICIT;
    case AUTO;
}
```

---

# 18. AUTO

`AUTO` permite al framework elegir una estrategia compatible según:

- cardinalidad;
- query shape;
- número de owners;
- capabilities;
- streaming;
- límites de parámetros;
- hints;
- telemetry feedback futuro.

Pero:

> AUTO puede elegir cómo cargar, no cambiar qué significa la relación.

---

# 19. Default recomendado

Para V1:

```text
To-One
    → LAZY or explicit/eager query override

To-Many
    → LAZY / EXPLICIT

Query-defined eager relationships
    → explicit fetch plan
```

evitando eager global agresivo.

---

# 20. RelationshipLoadContext

```php
final readonly class RelationshipLoadContext
{
    public function __construct(
        public PersistenceContextId $persistenceContext,
        public DatabaseContext $database,
        public ReadIntent $readIntent,
        public CancellationToken $cancellation,
        public ?Deadline $deadline,
        public RelationshipLoadPolicy $policy,
    ) {}
}
```

---

# 21. Context scoped

Nunca:

```php
RelationshipLoader::$currentTenant
```

ni:

```php
RelationshipLoader::$currentEntityManager
```

---

# 22. Load State

Una relación necesita conocer su estado de carga.

Para to-one:

```php
enum ToOneLoadState
{
    case UNINITIALIZED;
    case LOADED_NULL;
    case LOADED_REFERENCE;
    case STALE;
}
```

---

# 23. NULL ≠ UNINITIALIZED

Esta distinción es fundamental.

```text
LOADED_NULL
```

significa:

> La relación fue observada y no existe target.

Mientras:

```text
UNINITIALIZED
```

significa:

> El ORM todavía no conoce el valor.

---

# 24. To-Many Coverage

Para colecciones:

```php
enum CollectionCoverage
{
    case UNINITIALIZED;
    case PARTIAL;
    case COMPLETE;
}
```

---

# 25. Coverage invariant

```text
UNINITIALIZED
≠
PARTIAL
≠
COMPLETE
```

---

# 26. Empty collection

```text
count = 0
coverage = COMPLETE
```

significa:

> La relación está cargada completamente y está vacía.

No:

> La relación todavía no fue cargada.

---

# 27. Partial collection

Ejemplo:

```text
User.posts

loaded:
    last 10 posts

coverage:
    PARTIAL
```

No puede utilizarse para afirmar que esos diez posts son todos los posts.

---

# 28. Loaded knowledge

Formalmente:

```text
RelationshipKnowledge
=
ObservedMembers
+
Coverage
+
LoadGeneration
+
SourceContext
```

---

# 29. To-One Loading Pipeline

```text
Owner Entity
    ↓
RelationshipMetadata
    ↓
Target FK / Reference Metadata
    ↓
Load State Check
    ↓
IdentityMap Check
    ↓
Need Query?
    ↓ yes
Entity Query
    ↓
Execution
    ↓
Hydration
    ↓
IdentityMap
    ↓
Reference Assignment
    ↓
Snapshot / Load State
```

---

# 30. IdentityMap-first optimization

Cuando el owner contiene identidad suficiente del target:

```text
Post.author_id = 42
```

y `User#42` ya está managed:

```text
Relationship Load
→ IdentityMap hit
→ reuse User#42
```

sin necesidad de materializar una segunda instancia.

---

# 31. IdentityMap hit ≠ always no query

El IdentityMap prueba existencia de una instancia managed.

No necesariamente prueba que:

- esté fresca;
- tenga todos los campos necesarios;
- satisfaga un explicit refresh;
- pertenezca a una query con constraints adicionales.

La policy determina si el hit basta.

---

# 32. Canonical identity rule

Si el query devuelve:

```text
User#42
```

y IdentityMap ya contiene `User#42`:

```text
ReturnedObject
=
CanonicalManagedUser#42
```

---

# 33. No duplicate entity

Nunca:

```text
Post.author = UserObjectA#42
IdentityMap = UserObjectB#42
```

dentro del mismo compatible `PersistenceContext`.

---

# 34. Many-to-One Loading

Ejemplo:

```text
Post#100
author_id = 42
```

Metadata:

```text
Post.author
MANY_TO_ONE
target User
```

Carga:

```text
Post#100
    ↓
author identity = 42
    ↓
IdentityMap
    ├── HIT  → User#42
    └── MISS → Entity Query(User#42)
                     ↓
                  Hydration
                     ↓
                 User#42
```

---

# 35. Null foreign key

Si:

```text
author_id = NULL
```

y la FK fue realmente cargada:

```text
Post.author
→ LOADED_NULL
```

sin query.

---

# 36. Missing FK field

Si `author_id` no fue seleccionado:

```text
Missing
≠
NULL
```

Por tanto el loader no podrá concluir `LOADED_NULL`.

---

# 37. Loaded Field Mask integration

La decisión deberá consultar:

```text
LoadedFieldMask
```

cuando la identidad relationship dependa de campos parcialmente cargados.

---

# 38. One-to-One Loading

Semánticamente similar a to-one, pero puede requerir query inversa cuando el FK reside en la otra entidad.

Ejemplo:

```text
User.profile
mappedBy Profile.user
```

---

# 39. Inverse One-to-One

El owner `User` puede no contener `profile_id`.

Entonces:

```text
User#10
    ↓
Query Profile
WHERE profile.user_id = 10
```

a través de Entity Query.

---

# 40. Cardinality enforcement

Una One-to-One inverse query deberá producir:

```text
0 or 1 target
```

Si devuelve más:

```text
RelationshipCardinalityViolationException
```

---

# 41. Database constraint recommendation

Cuando sea físicamente representable, DB deberá respaldar la unicidad mediante:

```text
UNIQUE
```

El ORM no sustituye integridad concurrente.

---

# 42. One-to-Many Loading

Ejemplo:

```text
User#42.posts
```

Pipeline:

```text
User#42
    ↓
RelationshipMetadata
    ↓
Post.author mappedBy
    ↓
Entity Query:
    Post where author = User#42
    ↓
Execution
    ↓
Entity Hydration
    ↓
IdentityMap reuse
    ↓
Collection Assembly
    ↓
PersistentCollection
    ↓
coverage = COMPLETE
```

si el query representa la relación completa.

---

# 43. Query completeness

No toda query sobre una relación produce `COMPLETE`.

Ejemplo:

```text
User.posts
WHERE published = true
LIMIT 10
```

no necesariamente representa toda `User.posts`.

---

# 44. Coverage rule

```text
COMPLETE
⇔
QuerySemanticsCoverEntireRelationshipDefinition
```

---

# 45. Filtered relation

Si metadata define la propia relación como:

```text
publishedPosts
WHERE published = true
```

entonces cargar todos los miembros de esa definición sí puede producir `COMPLETE`.

---

# 46. Ad-hoc filter

Si el filtro fue agregado solo a la operación:

```text
posts()->where(...)
```

el resultado será:

```text
PARTIAL
```

respecto de `User.posts`.

---

# 47. LIMIT

En general:

```text
LIMIT
→ PARTIAL
```

salvo que la cardinalidad/metadata pruebe completitud.

---

# 48. OFFSET

Igualmente:

```text
OFFSET
→ PARTIAL
```

para una colección.

---

# 49. Many-to-Many Loading

Ejemplo:

```text
User.roles
```

requiere conceptualmente:

```text
User
   ↓
role_user
   ↓
Role
```

---

# 50. Many-to-Many Query

Entity Query podrá expresar:

```text
Role
JOIN role_user
WHERE role_user.user_id = :user
```

sin SQL manual en Relationship Loader.

---

# 51. Join table metadata

El loader obtiene de `RelationshipMetadata`:

- owner join bindings;
- target join bindings;
- join table;
- optional pivot metadata;
- ordering;
- filters;
- morph metadata cuando aplique.

---

# 52. Many-to-Many assembly

```text
Result rows
    ↓
Role EntityHydrator
    ↓
IdentityMap
    ↓
Membership Assembly
    ↓
User.roles PersistentCollection
```

---

# 53. Duplicate join rows

La colección `SET_LIKE` deberá deduplicar por:

```text
EntityKey
```

no por object equality general.

---

# 54. Duplicate physical row ≠ duplicate logical member

JOINs adicionales pueden multiplicar filas.

El assembly deberá preservar cardinalidad lógica.

---

# 55. Polymorphic To-One Loading

Ejemplo:

```text
Comment.commentable_type = "video"
Comment.commentable_id   = 25
```

Pipeline:

```text
Morph alias "video"
        ↓
MorphTypeRegistry
        ↓
Video EntityType
        ↓
EntityKey(Video#25)
        ↓
IdentityMap / Entity Query
        ↓
Canonical Video#25
```

---

# 56. Security rule

Nunca:

```php
new $databaseClassName();
```

basándose en contenido arbitrario de DB.

---

# 57. Unknown morph type

Debe producir:

```text
UnknownMorphTypeException
```

o policy explícita.

Nunca instanciar una clase desconocida.

---

# 58. Polymorphic batch loading

Owners:

```text
Comment#1 → Post#10
Comment#2 → Video#5
Comment#3 → Post#12
```

pueden agruparse:

```text
Post:
    ids 10,12

Video:
    id 5
```

---

# 59. Morph grouping

```text
BatchGroups
=
GroupBy(MorphType)
```

seguido de agrupación por compatible connection/tenant/shard context.

---

# 60. Explicit Loading

API conceptual:

```php
$entityManager
    ->relationships()
    ->load($user, 'posts');
```

o:

```php
$entityManager->load($user, 'posts');
```

---

# 61. Explicit loading advantage

Hace el I/O visible en el código.

Es especialmente útil para:

- APIs;
- background jobs;
- persistent runtimes;
- performance-sensitive paths;
- debugging N+1.

---

# 62. Already-loaded explicit relation

Policy posible:

```php
load($user, 'posts')
```

si `COMPLETE`:

```text
NO_OP
```

por default.

---

# 63. Explicit reload

Debe diferenciarse:

```php
reload($user, 'posts');
```

o:

```php
load(
    $user,
    'posts',
    mode: FORCE_REFRESH
);
```

---

# 64. Load ≠ Refresh

```text
LOAD
→ obtain missing relationship knowledge

REFRESH
→ intentionally replace/reconcile known relationship knowledge
```

---

# 65. Lazy Loading

Lazy loading difiere el I/O hasta acceso.

Conceptualmente:

```php
$user->posts
```

puede activar:

```text
LazyRelationshipInitializer
```

si la policy lo permite.

---

# 66. Lazy loading danger

El acceso aparentemente local:

```php
$user->posts
```

puede producir I/O.

Esto genera riesgos:

- N+1;
- I/O inesperado;
- query durante serialization;
- query después de cerrar context;
- query en destructor;
- query durante logging;
- query en worker equivocado.

---

# 67. Governed lazy loading

VoltStack no deberá implementar lazy loading como magia irrestricta.

Debe existir:

```text
LazyLoadingPolicy
```

---

# 68. LazyLoadingPolicy

```php
enum LazyLoadingPolicy
{
    case ALLOW;
    case ALLOW_WITH_TELEMETRY;
    case WARN;
    case FORBID;
}
```

---

# 69. Recommended development policy

Durante desarrollo:

```text
ALLOW_WITH_TELEMETRY
```

o:

```text
WARN
```

puede ser útil.

Para determinados dominios críticos:

```text
FORBID
```

---

# 70. Lazy loading outside context

Si la entidad está detached o su context ya terminó:

```text
lazy access
→ fail explicitly
```

---

# 71. No global EntityManager recovery

Nunca resolver:

```text
detached entity
→ global current EntityManager
→ hidden query
```

---

# 72. Lazy reference

Una to-one puede usar:

```text
LazyEntityReference
```

o proxy controlado.

---

# 73. Lazy collection

To-many puede utilizar:

```text
PersistentCollection
```

con initializer scoped.

---

# 74. Proxy ≠ entity identity

El proxy deberá utilizar el mismo:

```text
Canonical EntityType
+
EntityKey
```

que la entidad real.

---

# 75. Proxy promotion

Cuando se inicializa:

```text
Proxy(User#42)
→ hydrated canonical representation
```

sin crear una segunda identidad lógica.

---

# 76. Proxy implementation

La arquitectura no deberá obligar al dominio a depender de proxies para funcionar.

Podrán existir estrategias:

- proxy;
- lazy reference wrapper;
- generated subclass;
- explicit relationship accessor;
- ghost object, si PHP/runtime lo permite adecuadamente.

---

# 77. Eager Loading

Eager loading solicita relaciones como parte de una operación conocida.

Ejemplo:

```php
User::query()
    ->with('posts')
    ->get();
```

---

# 78. Eager loading ≠ JOIN necesariamente

`with('posts')` expresa:

```text
eager relationship requirement
```

no:

```text
must SQL JOIN
```

---

# 79. Eager strategy selection

Puede elegir:

```text
JOIN
SELECT_IN
BATCH
```

según relationship shape.

---

# 80. Join Loading

Ejemplo:

```text
User
LEFT JOIN Post
```

puede obtener users + posts en una consulta física.

---

# 81. Join row multiplication

Para:

```text
1 User
100 Posts
```

el resultado físico puede contener 100 filas con datos repetidos del User.

---

# 82. Logical deduplication

Hydration deberá reutilizar:

```text
User#42
```

mediante IdentityMap.

Relationship Assembly añadirá los posts a la colección correspondiente.

---

# 83. Multiple to-many JOIN problem

Ejemplo:

```text
User
JOIN posts
JOIN roles
```

si:

```text
10 posts
5 roles
```

puede generar:

```text
50 physical rows
```

por usuario.

---

# 84. Cartesian amplification

Por tanto:

```text
JOIN eager loading
```

no será estrategia universal.

---

# 85. Select-In Loading

Alternativa:

```text
SELECT Users
        ↓
owner IDs
        ↓
SELECT Posts
WHERE author_id IN (...)
```

---

# 86. Select-In benefits

Reduce:

- N+1;
- row multiplication;
- repeated root data.

---

# 87. Select-In limitations

Debe respetar:

- parameter limits;
- composite IDs;
- tenant boundaries;
- shard boundaries;
- query size;
- cancellation;
- memory limits.

---

# 88. Select-In chunking

Para `N` owners:

```text
owners
→ chunks
→ one relationship query per chunk
```

---

# 89. Chunk size

No deberá codificarse como constante universal.

Depende de:

```text
PlatformCapabilities
ParameterLimit
IdentifierArity
ConfiguredResourcePolicy
```

---

# 90. Composite ID formula

Si cada owner requiere `K` parámetros:

```text
MaxOwnersPerQuery
≤
floor(
    AvailableParameterBudget / K
)
```

---

# 91. Batch Loading

Batch loading combina solicitudes lazy/explicit compatibles.

Ejemplo:

```text
load User#1.posts
load User#2.posts
load User#3.posts
```

puede convertirse en:

```text
load posts for users [1,2,3]
```

---

# 92. Batch compatibility

Solo agrupar solicitudes con:

```text
same RelationshipId
compatible tenant
compatible connection
compatible shard
compatible fetch semantics
compatible filters
compatible consistency requirements
```

---

# 93. Batch key

Conceptualmente:

```text
RelationshipBatchKey
=
RelationshipId
×
DatabaseContext
×
TenantContext
×
ShardContext
×
LoadSemantics
```

---

# 94. No cross-tenant batch

Nunca:

```text
TenantA owners
+
TenantB owners
→ same relationship query
```

por default.

---

# 95. Batch result distribution

```text
Result targets
        ↓
Foreign/Join owner identity
        ↓
GroupBy OwnerEntityKey
        ↓
Assign each owner collection/reference
```

---

# 96. Owners with zero members

Deben quedar explícitamente:

```text
COMPLETE empty collection
```

si el batch query cubrió completamente la relación.

---

# 97. Critical batch invariant

No recibir rows para un owner no significa desconocimiento si:

```text
owner was included in complete load request
```

En ese caso:

```text
zero rows
→ known empty
```

---

# 98. Batch result set

El loader debe conocer qué owners fueron efectivamente cubiertos.

```text
RequestedOwners
≠
OwnersAppearingInRows
```

---

# 99. RelationshipLoadOutcome

```php
final readonly class RelationshipLoadOutcome
{
    /**
     * @param list<RelationshipOwnerLoadOutcome> $owners
     */
    public function __construct(
        public RelationshipId $relationship,
        public array $owners,
        public RelationshipLoadOutcomeStatus $status,
    ) {}
}
```

---

# 100. Outcome statuses

```php
enum RelationshipLoadOutcomeStatus
{
    case SUCCESS;
    case PARTIAL;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 101. Read UNKNOWN

Aunque reads suelen ser más fáciles de razonar que writes, puede existir incertidumbre operacional.

Ejemplo:

```text
stream partially consumed
connection lost
```

El sistema conoce:

- lo ya materializado;
- que el resultado completo no terminó.

---

# 102. Partial read

Una colección que iba a ser `COMPLETE` pero cuyo cursor falla a mitad:

```text
must not become COMPLETE
```

---

# 103. Failure rule

```text
IncompletePhysicalConsumption
→
CannotClaimCompleteCoverage
```

---

# 104. Streaming to-many loading

Una relación grande puede procesarse por streaming.

Pero la colección no deberá marcarse `COMPLETE` antes de llegar al final válido del grupo/result.

---

# 105. Safe finalization

```text
CollectionComplete
⇔
RequiredResultCoverageConsumedSuccessfully
```

---

# 106. Streaming group boundary

Cuando varias relaciones se ensamblan desde JOINs:

```text
MayFinalize(owner)
⇔
NoFutureRowCanExtendOwnerRelationship
```

---

# 107. Ordering requirement

Streaming grouped assembly puede requerir que resultados estén ordenados por root/owner key.

---

# 108. No ordering guarantee

Si no existe garantía:

```text
buffer
```

o utilizar otra estrategia.

Nunca finalizar prematuramente.

---

# 109. RelationshipLoadPlan

La estrategia efectiva deberá compilarse en un plan.

```php
final readonly class RelationshipLoadPlan
{
    public function __construct(
        public RelationshipId $relationship,
        public RelationshipLoadStrategy $strategy,
        public RelationshipQueryPlan $query,
        public RelationshipAssemblyPlan $assembly,
        public RelationshipCoveragePlan $coverage,
        public RelationshipLoadRequirements $requirements,
    ) {}
}
```

---

# 110. Relationship Load Plan ≠ Query Plan

El `RelationshipLoadPlan` describe:

```text
relationship semantics
+
loading strategy
+
assembly
+
coverage
```

El Query Plan describe ejecución lógica/física del query.

---

# 111. Relationship Query Plan

Puede contener una especificación para generar:

```text
EntityQuery
```

o un Query Model derivado por las capas correspondientes.

---

# 112. Assembly Plan

Define:

- owner key extraction;
- target key extraction;
- target entity hydration;
- grouping;
- collection membership;
- null-target semantics;
- deduplication;
- ordering;
- coverage finalization.

---

# 113. HydrationPlan integration

Un Relationship Load Plan deberá reutilizar:

```text
HydrationPlanTemplate
```

cuando corresponda.

---

# 114. Separation

```text
RelationshipLoadPlan
        ↓ uses
HydrationPlan
```

No:

```text
HydrationPlan
→ executes relationship queries
```

---

# 115. Joined hydration

Cuando la relación ya viene dentro del query raíz:

```text
ResultHydrator
+
RelationshipAssemblyPlan
```

pueden materializarla sin query adicional.

---

# 116. Already-present relationship data

Regla:

> Si los datos suficientes de una relación ya están presentes en el `Result`, Relationship Assembly deberá utilizarlos antes de programar una query adicional.

---

# 117. No duplicate query

```text
JOIN already loaded User.posts
+
lazy access User.posts
```

no deberá ejecutar otra query si coverage es `COMPLETE` y válida.

---

# 118. Partial joined relationship

Si JOIN/filter produjo solo parte:

```text
coverage = PARTIAL
```

Un acceso que requiera la colección completa puede necesitar una operación adicional.

---

# 119. Partial → Complete transition

Debe ser explícita:

```text
PARTIAL
    ↓ complete load
COMPLETE
```

---

# 120. Merge semantics

Al completar una colección parcial:

```text
existing observed members
+
new complete result
+
local explicit mutations
```

deben reconciliarse cuidadosamente.

---

# 121. Local dirty collection during load

Este es un caso crítico.

Ejemplo:

```text
baseline:
    A, B

local mutation:
    + C

reload from DB:
    A, B
```

No deberá perderse `C` silenciosamente.

---

# 122. Load vs Refresh policy

Normal load:

```text
preserve local relationship mutations
```

por default.

Explicit refresh puede utilizar política distinta.

---

# 123. RelationshipMergePolicy

```php
enum RelationshipMergePolicy
{
    case PRESERVE_LOCAL_CHANGES;
    case REFRESH_CLEAN_ONLY;
    case OVERWRITE;
    case FAIL_IF_DIRTY;
}
```

---

# 124. Recommended normal load policy

```text
PRESERVE_LOCAL_CHANGES
```

---

# 125. Recommended explicit refresh

```text
FAIL_IF_DIRTY
```

por default, con override explícito si el programador quiere overwrite.

---

# 126. Database state ≠ local graph

El loader no deberá asumir que un nuevo query puede sobrescribir arbitrariamente cambios locales.

---

# 127. To-One dirty relationship

Ejemplo:

```text
Post.author baseline = User#1
local state          = User#2
DB query returns     = User#1
```

Un lazy/eager load no deberá revertir `User#2`.

---

# 128. Dirty relationship load

Normalmente:

```text
reuse current local relationship state
```

o fallar si la operación solicitaba refresh.

---

# 129. Refresh semantics

`refreshRelationship()` deberá declarar qué ocurre con cambios locales.

---

# 130. Relationship snapshots

Después de una carga limpia completa:

```text
RelationshipSnapshot
=
ObservedDatabaseRelationshipState
```

---

# 131. Snapshot before domain/lifecycle mutation

Al igual que entity hydration:

```text
baseline
```

deberá establecerse antes de callbacks que puedan crear cambios posteriores.

---

# 132. Relationship post-load lifecycle

No deberá dispararse una vez por cada physical JOIN row.

---

# 133. Logical load event

Si se define un lifecycle/evento relationship-specific:

```text
postRelationshipLoad
```

deberá ocurrir por carga lógica finalizada, no por row.

---

# 134. Entity postLoad

La materialización de nuevos targets sigue las reglas de `EntityHydrator`.

El Relationship Loader no deberá disparar `postLoad` manualmente por duplicado.

---

# 135. Cycles

Ejemplo:

```text
User.posts
Post.author
```

Puede formarse un ciclo natural.

---

# 136. Cycle ≠ error

Los grafos ORM pueden ser cíclicos.

El problema es:

```text
recursive loading without termination
```

---

# 137. RelationshipLoadSession

```php
final class RelationshipLoadSession
{
    /** @var array<RelationshipLoadKey, RelationshipLoadState> */
    private array $activeLoads = [];
}
```

---

# 138. RelationshipLoadKey

Conceptualmente:

```text
OwnerEntityKey
×
RelationshipId
×
LoadSemantics
```

---

# 139. Active load detection

Si durante carga:

```text
User.posts
→ Post.author
→ User.posts
```

se vuelve a solicitar la misma operación compatible:

```text
reuse active knowledge
or defer
```

en lugar de recursión infinita.

---

# 140. Load recursion policy

```php
enum RelationshipLoadRecursionPolicy
{
    case REUSE_ACTIVE;
    case DEFER;
    case FAIL_ON_REENTRANT_IO;
}
```

---

# 141. Default

Preferir:

```text
REUSE_ACTIVE / DEFER
```

cuando sea semánticamente seguro.

---

# 142. Maximum depth

Eager graph expansion deberá tener límites.

```text
User
→ posts
→ comments
→ author
→ posts
→ ...
```

---

# 143. Fetch depth

Un fetch graph explícito puede definir:

```text
maxDepth
```

pero depth no sustituye cycle detection.

---

# 144. Fetch Graph

Una query puede solicitar:

```text
User
├── profile
├── posts
│   └── comments
└── roles
```

---

# 145. FetchGraph ≠ arbitrary recursion

Debe ser:

- typed;
- validated;
- finite;
- metadata-aware.

---

# 146. FetchGraphNode

```php
final readonly class RelationshipFetchNode
{
    /**
     * @param list<RelationshipFetchNode> $children
     */
    public function __construct(
        public RelationshipId $relationship,
        public RelationshipFetchPolicy $policy,
        public array $children = [],
    ) {}
}
```

---

# 147. Strategy per node

Un fetch graph puede resolverse:

```text
profile
→ JOIN

posts
→ SELECT_IN

posts.comments
→ BATCH

roles
→ SELECT_IN
```

---

# 148. Eager strategy planner

Esto sugiere:

```text
RelationshipFetchPlanner
```

que decide estrategias sin generar SQL.

---

# 149. Cost awareness

Planner podrá considerar:

```text
relationship cardinality estimate
owner count
join multiplication risk
parameter limits
streaming requirement
memory policy
```

---

# 150. Cost hint ≠ DB optimizer

El Relationship Fetch Planner no sustituye al Query Optimizer de DB/VoltStack.

Solo decide arquitectura de carga.

---

# 151. N+1 architecture

Ejemplo:

```text
SELECT users
for each user:
    SELECT posts
```

---

# 152. Loader telemetry

Cada lazy load deberá registrar contexto suficiente para detectar patrones repetitivos.

---

# 153. N+1 detection hook

Conceptualmente:

```php
interface RelationshipLoadObserver
{
    public function loading(RelationshipLoadObservation $observation): void;
}
```

---

# 154. Observer cannot alter correctness

Telemetry/N+1 detection podrá:

- registrar;
- advertir;
- recomendar;
- lanzar excepción en strict development policy.

No deberá cambiar silenciosamente los resultados.

---

# 155. Automatic batching

Una evolución futura puede convertir lazy loads compatibles en batch.

Pero debe mantener semántica determinista.

---

# 156. DataLoader-like model

Conceptualmente:

```text
Relationship requests
        ↓
Batch Queue
        ↓
Compatibility Grouping
        ↓
SELECT IN
        ↓
Distribute Results
```

---

# 157. Runtime scheduling caveat

En PHP tradicional un getter síncrono espera resultado inmediato.

Por tanto batching lazy automático requiere integración cuidadosa con:

- reactive runtime;
- coroutine runtime;
- explicit preload boundary;
- deferred access abstraction.

---

# 158. V1 recommendation

No depender de magic asynchronous batching para correctness.

Priorizar:

```text
explicit eager loading
+
select-in
+
explicit batch loading
```

---

# 159. Read Routing

Relationship queries deberán transportar:

```text
ReadIntent
```

hacia la infraestructura de routing.

---

# 160. Loader does not choose replica directly

Nunca:

```php
if ($relationship->lazy) {
    $connection = $replica;
}
```

---

# 161. Read/Write Router responsibility

El loader expresa:

```text
read intent
consistency requirements
transaction context
```

y el router decide conexión.

---

# 162. Read-your-writes

Caso:

```text
persist new Post
flush
$user->posts
```

Una replica atrasada podría no mostrar el nuevo Post.

---

# 163. Sticky/read consistency

La futura infraestructura de read/write routing deberá considerar:

```text
recent writes
active transaction
sticky primary
replica lag
consistency requirement
```

---

# 164. Relationship load consistency

Un `RelationshipLoadRequest` podrá incluir:

```php
enum RelationshipReadConsistency
{
    case DEFAULT;
    case READ_YOUR_WRITES;
    case PRIMARY_REQUIRED;
    case EVENTUAL_ALLOWED;
}
```

---

# 165. Transaction context

Dentro de una transacción:

```text
relationship query
```

normalmente deberá usar la misma transaction/connection context.

---

# 166. No cross-transaction surprise

Nunca cargar parte del graph desde una replica externa mientras la entidad owner participa en una transacción incompatible.

---

# 167. Stale managed targets

Una query puede devolver una entidad que ya está en IdentityMap pero stale.

Normal hydration policy decide:

```text
reuse
refresh
reject
```

Relationship Loader no sobrescribe por su cuenta.

---

# 168. Relationship stale state

La propia relación también puede estar:

```text
STALE
```

por:

- raw SQL;
- bulk mutation;
- external DB change;
- direct relationship operation;
- rollback;
- explicit invalidation.

---

# 169. STALE ≠ UNINITIALIZED

`STALE` significa:

> Existe conocimiento previo, pero ya no debe considerarse confiable como estado actual.

---

# 170. Stale load policy

Puede:

```text
reload
require explicit refresh
reuse with warning
```

según consistency policy.

---

# 171. Relationship generation

Cada relación runtime puede conservar:

```text
RelationshipLoadGeneration
```

para detectar reemplazos/invalidation.

---

# 172. Generation use

Conceptualmente:

```text
Generation 5
    load complete

external mutation
    ↓
Generation 6 / stale marker
```

---

# 173. Cancellation

Cada load request deberá respetar:

```text
CancellationToken
Deadline
```

---

# 174. Cancellation before query

Resultado:

```text
CANCELLED
```

sin cambiar coverage.

---

# 175. Cancellation during query

Si no existe resultado completo:

```text
do not mark COMPLETE
```

---

# 176. Cancellation during assembly

Las entidades ya correctamente hidratadas pueden permanecer managed.

Pero la relación actual no deberá publicarse como completa si no fue finalizada.

---

# 177. No fake rollback

Si targets fueron materializados correctamente antes de que falle assembly:

```text
relationship load failure
≠
pretend targets were never hydrated
```

---

# 178. Atomic publication

Para cargas buffered, se recomienda:

```text
assemble temporary membership
        ↓
validate
        ↓
finalize
        ↓
publish collection state
```

---

# 179. Large relationships

No siempre es viable bufferizar millones de miembros.

Por ello deberán existir estrategias especializadas.

---

# 180. Resource Governance

```php
final readonly class RelationshipLoadResourcePolicy
{
    public function __construct(
        public int $maxOwnersPerBatch,
        public int $maxBufferedMembers,
        public int $maxFetchDepth,
        public int $maxConcurrentLoads,
        public int $maxParameterBudget,
    ) {}
}
```

---

# 181. Resource limits

Los límites deben producir errores explícitos o estrategia alternativa.

Nunca truncar silenciosamente una relación y marcarla `COMPLETE`.

---

# 182. Truncation invariant

```text
ResourceLimitReached
→
PARTIAL / FAILED
```

nunca:

```text
ResourceLimitReached
→
COMPLETE
```

---

# 183. EXTRA_LAZY future mode

Colecciones enormes podrán soportar operaciones:

```text
count()
contains()
exists()
slice()
```

sin inicialización completa.

---

# 184. EXTRA_LAZY ≠ COMPLETE

Ejecutar:

```php
$user->posts->count();
```

no significa que `User.posts` haya sido cargada.

---

# 185. Query-backed collection operations

Estas operaciones deberán ir por Relationship Query APIs controladas.

No por SQL incrustado en Collection.

---

# 186. Count knowledge

```text
KnownCount
≠
KnownMembers
```

---

# 187. Contains knowledge

```text
KnownMembership(User.posts, Post#5)
≠
CompleteCollectionKnowledge
```

---

# 188. Coverage model extension

En el futuro podría evolucionar a:

```text
Coverage
├── NONE
├── PARTIAL_MEMBERS
├── COMPLETE_MEMBERS
├── COUNT_ONLY
├── MEMBERSHIP_FACTS
└── RANGE
```

Pero V1 puede conservar:

```text
UNINITIALIZED / PARTIAL / COMPLETE
```

más metadata auxiliar.

---

# 189. Pagination

Paginar una relación:

```text
posts page 1
```

produce una vista/proyección parcial.

No debería inicializar la colección managed completa por defecto.

---

# 190. Relationship Query View

Se recomienda separar:

```php
$user->postsRelation()
    ->paginate();
```

de:

```php
$user->posts
```

---

# 191. Why

Porque:

```text
paginated result
≠
managed complete relationship collection
```

---

# 192. Filtered relationship query

Igualmente:

```php
$user->postsRelation()
    ->where('status', 'published')
    ->get();
```

no debe reemplazar automáticamente `User.posts`.

---

# 193. Relationship query results

Pueden devolver entidades managed reutilizadas por IdentityMap sin afirmar coverage completa de la relación.

---

# 194. Ordering

RelationshipMetadata puede definir:

```text
default ordering
```

---

# 195. Ordered assembly

La colección deberá preservar ordering semántico cuando forme parte de la definición relationship.

---

# 196. Query-specific ordering

Un query ad-hoc con otro orden:

```text
does not redefine relationship ordering
```

---

# 197. Indexed collections

Si una relación usa key/index:

```text
productsBySku
```

assembly deberá detectar duplicate logical keys.

---

# 198. Duplicate collection key

Debe producir:

```text
RelationshipCollectionKeyCollisionException
```

salvo política explícita.

---

# 199. Relationship Load Cache

Relationship Loading no deberá introducir automáticamente un segundo-level relationship cache dentro de este documento.

---

# 200. First-level knowledge

El `PersistenceContext` ya contiene:

```text
loaded relationship state
```

como conocimiento scoped.

---

# 201. Second-level cache

Si existe en el futuro:

```text
Relationship Cache
```

deberá ser otro subsistema con invalidation/consistency explícitas.

---

# 202. Hydration Cache ≠ Relationship Cache

`Hydration Cache` almacena planes compilados.

No miembros de relaciones.

---

# 203. Query Result Cache ≠ Relationship State

Un result cache puede suministrar datos, pero el loader todavía debe:

- validar contexto;
- hidratar;
- reconciliar IdentityMap;
- establecer coverage correctamente.

---

# 204. Persistent runtime

Con FrankenPHP:

```text
Worker
├── Request A PersistenceContext
└── Request B PersistenceContext
```

deberán mantener relationship state completamente separado.

---

# 205. Shareable state

Puede compartirse:

```text
RelationshipMetadata
compiled load plans
immutable strategy definitions
compiled accessors
HydrationPlanTemplates
```

---

# 206. Scoped mutable state

Nunca compartir:

```text
current owner
current collection
active load session
batch queue
loaded members
coverage
dirty flags
EntityManager
IdentityMap
ResultCursor
tenant runtime context
```

---

# 207. RoadRunner/OpenSwoole

La misma separación aplica.

---

# 208. Coroutine safety

Un `RelationshipLoadSession` mutable no deberá asumirse concurrent-safe.

---

# 209. Parallel relationship loading

Una versión futura podrá cargar ramas independientes en paralelo:

```text
User.profile
User.roles
User.posts
```

si:

- runtime lo permite;
- transaction context lo permite;
- connection architecture lo permite;
- IdentityMap reconciliation está coordinada.

---

# 210. V1 concurrency rule

Preferir ejecución determinista secuencial dentro del mismo mutable `PersistenceContext`.

---

# 211. Error Taxonomy

```text
DatabaseRelationshipLoadingException
├── RelationshipLoadRequestException
├── RelationshipLoadPlanningException
├── RelationshipLoadStrategyException
├── RelationshipLoadExecutionException
├── RelationshipLoadAssemblyException
├── RelationshipLoadFinalizationException
├── RelationshipCardinalityViolationException
├── RelationshipCoverageException
├── RelationshipPartialLoadException
├── RelationshipLoadStateException
├── RelationshipAlreadyLoadingException
├── RelationshipLoadCycleException
├── RelationshipLoadDepthException
├── RelationshipBatchException
├── RelationshipBatchCompatibilityException
├── RelationshipCollectionKeyCollisionException
├── RelationshipMergeConflictException
├── DirtyRelationshipRefreshException
├── DetachedRelationshipLazyLoadException
├── ClosedPersistenceContextLoadException
├── RelationshipIdentityResolutionException
├── RelationshipTargetTypeException
├── UnknownMorphTypeException
├── RelationshipReadConsistencyException
├── RelationshipStreamingException
├── RelationshipLoadCancellationException
├── RelationshipLoadDeadlineException
├── RelationshipLoadResourceLimitException
├── RelationshipRuntimeIsolationException
└── RelationshipLoadingInvariantException
```

---

# 212. Detached entity lazy loading

Error recomendado:

```text
DetachedRelationshipLazyLoadException

Cannot lazy-load User.posts because User#42
is not attached to an active compatible PersistenceContext.
```

---

# 213. Diagnostics

Debe ser posible explicar una carga:

```text
RELATIONSHIP LOAD

Relationship:
    User.posts

Kind:
    ONE_TO_MANY

Owners:
    25

Configured Policy:
    EAGER

Effective Strategy:
    SELECT_IN

Queries:
    1

Physical Rows:
    315

Logical Members:
    315

IdentityMap Hits:
    72

IdentityMap Misses:
    243

Coverage:
    COMPLETE

Dirty Merge:
    none

Consistency:
    READ_YOUR_WRITES

Outcome:
    SUCCESS
```

---

# 214. Lazy diagnostic

```text
Relationship:
    User.posts

Trigger:
    LAZY_ACCESS

Call Site:
    application code location

Owner:
    User

Previous State:
    UNINITIALIZED

Strategy:
    SELECT

Outcome:
    SUCCESS

Potential N+1 Group:
    detected
```

sin exponer PII.

---

# 215. Telemetry

Métricas:

```text
database.orm.relationship.loads
database.orm.relationship.lazy_loads
database.orm.relationship.explicit_loads
database.orm.relationship.eager_loads
database.orm.relationship.join_loads
database.orm.relationship.select_in_loads
database.orm.relationship.batch_loads

database.orm.relationship.loaded_owners
database.orm.relationship.loaded_members

database.orm.relationship.identity_map_hits
database.orm.relationship.identity_map_misses

database.orm.relationship.partial_loads
database.orm.relationship.complete_loads
database.orm.relationship.failed_loads
database.orm.relationship.cancelled_loads

database.orm.relationship.batch_size
database.orm.relationship.fetch_depth
database.orm.relationship.query_count
database.orm.relationship.row_amplification
```

---

# 216. Row amplification metric

Conceptualmente:

```text
RowAmplification
=
PhysicalRows / LogicalRootEntities
```

para determinados eager joins.

---

# 217. N+1 metric

```text
RepeatedRelationshipLoads
=
Count(
    same RelationshipId
    within same operation/request scope
)
```

agrupado por query/load fingerprint.

---

# 218. Telemetry cardinality

No usar IDs concretos como metric labels.

---

# 219. Load Plan Cacheability

Un `RelationshipLoadPlan` puede ser cacheable si:

```text
Immutable
∧
Deterministic
∧
NoScopedMutableState
```

---

# 220. Cache key

Conceptualmente:

```text
RelationshipLoadPlanFingerprint
=
H(
    RelationshipMetadataGeneration,
    LoadStrategy,
    FetchGraphShape,
    QueryShape,
    HydrationPlanGeneration,
    TypeRegistryGeneration,
    PlatformCapabilityGeneration,
    LoadPolicyGeneration
)
```

---

# 221. Do not include entity instances

Nunca almacenar en plan cache:

```text
User object
EntityManager
IdentityMap
tenant object
current IDs
ResultCursor
```

---

# 222. Parameter values

Los IDs concretos de owners normalmente son runtime inputs.

No deben formar parte del plan fingerprint cuando no cambian shape.

---

# 223. Relationship Load Strategy interface

```php
interface RelationshipLoadStrategy
{
    public function supports(
        RelationshipLoadRequest $request,
        RelationshipMetadata $metadata,
    ): bool;

    public function plan(
        RelationshipLoadRequest $request,
        RelationshipMetadata $metadata,
        RelationshipLoadPlanningContext $context,
    ): RelationshipLoadPlan;
}
```

---

# 224. Strategy Registry

```text
RelationshipLoadStrategyRegistry
├── ExplicitSelectStrategy
├── LazySelectStrategy
├── JoinLoadStrategy
├── SelectInLoadStrategy
├── BatchLoadStrategy
└── PolymorphicBatchLoadStrategy
```

---

# 225. Registry order

No deberá depender de registro accidental.

Strategy selection debe ser:

- deterministic;
- priority-aware;
- validated.

---

# 226. RelationshipLoadCoordinator

```php
interface RelationshipLoadCoordinator
{
    public function load(
        RelationshipLoadRequest $request,
    ): RelationshipLoadOutcome;
}
```

---

# 227. Coordinator responsibilities

Coordina:

```text
validate
resolve strategy
build/resolve plan
execute entity queries
hydrate
assemble
merge
finalize coverage
emit outcome
```

---

# 228. Coordinator non-responsibilities

No:

- genera SQL;
- usa PDO;
- hace commit;
- persiste dirty relationships;
- implementa IdentityMap;
- interpreta arbitrary DB classes.

---

# 229. RelationshipAssembler

```php
interface RelationshipAssembler
{
    public function assemble(
        RelationshipAssemblyInput $input,
        RelationshipAssemblyPlan $plan,
        RelationshipLoadSession $session,
    ): RelationshipAssemblyResult;
}
```

---

# 230. Assembly Input

Puede contener:

```text
owner
hydrated target
owner grouping key
pivot values
physical row position
```

sin SQL.

---

# 231. ToOneRelationshipAssembler

Responsable de:

- null target;
- cardinality;
- canonical target;
- dirty merge policy;
- loaded state.

---

# 232. ToManyRelationshipAssembler

Responsable de:

- membership;
- dedup;
- ordering;
- grouping;
- coverage;
- PersistentCollection integration.

---

# 233. ManyToManyRelationshipAssembler

Añade semántica de:

- join membership;
- optional association payload;
- join owner grouping.

---

# 234. PolymorphicRelationshipAssembler

Añade:

- morph type;
- target type validation;
- grouping por concrete type.

---

# 235. RelationshipCoverageResolver

```php
interface RelationshipCoverageResolver
{
    public function resolve(
        RelationshipLoadPlan $plan,
        RelationshipExecutionEvidence $evidence,
    ): CollectionCoverage;
}
```

---

# 236. Coverage is evidence-based

Nunca:

```text
strategy = EAGER
→ automatically COMPLETE
```

La cobertura depende del query real y de su finalización.

---

# 237. Coverage formula

```text
CompleteCoverage
=
SemanticFullCoverage
∧
PhysicalExecutionCompleted
∧
AssemblyCompleted
∧
NoTruncation
```

---

# 238. Partial formula

```text
PartialCoverage
=
SomeValidKnowledge
∧
¬CompleteCoverage
```

---

# 239. No result formula

Para owner incluido en full coverage query:

```text
NoTargetRows
+
SuccessfulCompleteExecution
→
KnownEmpty
```

---

# 240. Relationship snapshot creation

Después de complete clean load:

```text
Snapshot
=
CanonicalRelationshipState
```

---

# 241. Partial snapshot

Un partial load no deberá crear un snapshot que parezca completo.

Puede conservar:

```text
PartialRelationshipObservation
```

separado del baseline completo.

---

# 242. Persistence integration

El documento 149 deberá interpretar correctamente:

```text
PARTIAL
```

para no convertir miembros no observados en removals.

---

# 243. Relationship Loading → Persistence contract

```text
Loading System
produces
Relationship Knowledge + Coverage

Persistence System
consumes
Relationship Knowledge + Coverage + Explicit Mutations
```

---

# 244. Architectural bridge

Esta separación es crítica:

```text
Loading tells us what we know.

Change Tracking tells us what changed.

Persistence tells us what to write.
```

---

# 245. Directory Structure

```text
src/Quantum/Database/ORM/Relationship/Loading/
│
├── Contract/
│   ├── RelationshipLoadCoordinator.php
│   ├── RelationshipLoadStrategy.php
│   ├── RelationshipAssembler.php
│   ├── RelationshipCoverageResolver.php
│   └── RelationshipLoadObserver.php
│
├── Request/
│   ├── RelationshipLoadRequest.php
│   ├── RelationshipLoadMode.php
│   ├── RelationshipLoadContext.php
│   ├── RelationshipReadConsistency.php
│   └── RelationshipLoadKey.php
│
├── Plan/
│   ├── RelationshipLoadPlan.php
│   ├── RelationshipLoadPlanFingerprint.php
│   ├── RelationshipQueryPlan.php
│   ├── RelationshipAssemblyPlan.php
│   ├── RelationshipCoveragePlan.php
│   └── RelationshipLoadRequirements.php
│
├── Strategy/
│   ├── RelationshipLoadStrategyRegistry.php
│   ├── ExplicitSelectStrategy.php
│   ├── LazySelectStrategy.php
│   ├── JoinLoadStrategy.php
│   ├── SelectInLoadStrategy.php
│   ├── BatchLoadStrategy.php
│   └── PolymorphicBatchLoadStrategy.php
│
├── Planner/
│   ├── RelationshipFetchPlanner.php
│   ├── RelationshipStrategyResolver.php
│   ├── RelationshipBatchPlanner.php
│   └── RelationshipLoadPlanCompiler.php
│
├── Assembly/
│   ├── ToOneRelationshipAssembler.php
│   ├── ToManyRelationshipAssembler.php
│   ├── ManyToManyRelationshipAssembler.php
│   ├── PolymorphicRelationshipAssembler.php
│   ├── RelationshipAssemblyInput.php
│   └── RelationshipAssemblyResult.php
│
├── State/
│   ├── ToOneLoadState.php
│   ├── CollectionCoverage.php
│   ├── RelationshipKnowledge.php
│   ├── RelationshipLoadGeneration.php
│   └── RelationshipLoadSession.php
│
├── Merge/
│   ├── RelationshipMergePolicy.php
│   ├── RelationshipStateMerger.php
│   └── DirtyRelationshipMergeGuard.php
│
├── Lazy/
│   ├── LazyLoadingPolicy.php
│   ├── LazyRelationshipInitializer.php
│   ├── LazyEntityReference.php
│   └── LazyCollectionInitializer.php
│
├── Batch/
│   ├── RelationshipBatchKey.php
│   ├── RelationshipBatch.php
│   ├── RelationshipBatchQueue.php
│   └── RelationshipBatchResultDistributor.php
│
├── FetchGraph/
│   ├── RelationshipFetchGraph.php
│   ├── RelationshipFetchNode.php
│   ├── RelationshipFetchPolicy.php
│   └── RelationshipFetchGraphValidator.php
│
├── Coverage/
│   ├── DefaultRelationshipCoverageResolver.php
│   └── RelationshipCoverageEvidence.php
│
├── Outcome/
│   ├── RelationshipLoadOutcome.php
│   ├── RelationshipOwnerLoadOutcome.php
│   └── RelationshipLoadOutcomeStatus.php
│
├── Resource/
│   └── RelationshipLoadResourcePolicy.php
│
├── Diagnostics/
│   ├── RelationshipLoadDiagnostics.php
│   └── RelationshipLoadExplainer.php
│
└── Exception/
    └── ...
```

---

# 246. Integration Matrix

| Sistema | Integración |
|---|---|
| Relationship Metadata | Define estructura y semántica |
| EntityManager | Proporciona PersistenceContext |
| Entity Query | Construye consultas entity-aware |
| Query Engine | Procesa Query Models |
| Execution Engine | Ejecuta |
| Hydration System | Materializa targets |
| Hydration Plan | Define result materialization |
| IdentityMap | Garantiza identidad canónica |
| UnitOfWork | Mantiene tracking |
| Snapshot System | Conserva baseline |
| Relationship Persistence | Consume coverage y mutations |
| Transaction System | Proporciona consistency context |
| Read/Write Routing | Selecciona conexión |
| Telemetry | Observa |
| N+1 Detection | Analiza patrones |
| Persistent Runtime | Define scope lifecycle |

---

# 247. Architectural Invariants

## DB-ORM-REL-LOAD-001

Relationship Loading no generará SQL.

## DB-ORM-REL-LOAD-002

Relationship Loading no utilizará PDO directamente.

## DB-ORM-REL-LOAD-003

Relationship Loading no ejecutará Driver APIs directamente.

## DB-ORM-REL-LOAD-004

Relationship Loading utilizará Entity Query/Query Engine.

## DB-ORM-REL-LOAD-005

Relationship Loading utilizará Hydration System para materialización.

## DB-ORM-REL-LOAD-006

Relationship Loading será distinto de Hydration.

## DB-ORM-REL-LOAD-007

Relationship Loading será distinto de Persistence.

## DB-ORM-REL-LOAD-008

Relationship Loading será distinto de Metadata.

## DB-ORM-REL-LOAD-009

Relationship Loading será distinto de IdentityMap.

## DB-ORM-REL-LOAD-010

Relationship Loading será distinto de PersistentCollection.

## DB-ORM-REL-LOAD-011

Lazy loading será una estrategia, no la arquitectura completa.

## DB-ORM-REL-LOAD-012

Eager loading será una estrategia/intención, no necesariamente JOIN.

## DB-ORM-REL-LOAD-013

Fetch policy será distinta de effective load mode.

## DB-ORM-REL-LOAD-014

UNINITIALIZED será distinto de loaded null.

## DB-ORM-REL-LOAD-015

UNINITIALIZED será distinto de complete empty.

## DB-ORM-REL-LOAD-016

PARTIAL será distinto de COMPLETE.

## DB-ORM-REL-LOAD-017

Missing FK field será distinto de NULL FK.

## DB-ORM-REL-LOAD-018

LoadedFieldMask participará en to-one resolution cuando sea necesario.

## DB-ORM-REL-LOAD-019

IdentityMap preservará instancia canónica.

## DB-ORM-REL-LOAD-020

Loader no creará duplicate managed identity.

## DB-ORM-REL-LOAD-021

IdentityMap hit no implicará automáticamente freshness.

## DB-ORM-REL-LOAD-022

Normal relationship loading no sobrescribirá dirty local state arbitrariamente.

## DB-ORM-REL-LOAD-023

Explicit refresh tendrá semántica distinta de load.

## DB-ORM-REL-LOAD-024

One-to-one respetará cardinalidad 0..1.

## DB-ORM-REL-LOAD-025

To-many coverage dependerá de query semantics.

## DB-ORM-REL-LOAD-026

Ad-hoc filtered query no marcará colección completa.

## DB-ORM-REL-LOAD-027

LIMIT normalmente producirá partial coverage.

## DB-ORM-REL-LOAD-028

OFFSET normalmente producirá partial coverage.

## DB-ORM-REL-LOAD-029

Many-to-many utilizará join metadata.

## DB-ORM-REL-LOAD-030

SET_LIKE membership deduplicará por canonical identity.

## DB-ORM-REL-LOAD-031

Duplicate physical rows no implicarán duplicate logical members.

## DB-ORM-REL-LOAD-032

Polymorphic loading utilizará MorphTypeRegistry.

## DB-ORM-REL-LOAD-033

Polymorphic loading no instanciará arbitrary DB class names.

## DB-ORM-REL-LOAD-034

Unknown morph type fallará explícitamente.

## DB-ORM-REL-LOAD-035

Polymorphic batch loading agrupará por concrete type.

## DB-ORM-REL-LOAD-036

Explicit loading hará I/O semánticamente visible.

## DB-ORM-REL-LOAD-037

Already complete relationship podrá producir no-op.

## DB-ORM-REL-LOAD-038

Lazy loading estará gobernado por policy.

## DB-ORM-REL-LOAD-039

Detached lazy loading fallará por defecto.

## DB-ORM-REL-LOAD-040

Closed-context lazy loading fallará.

## DB-ORM-REL-LOAD-041

Lazy loading no resolverá global EntityManager implícito.

## DB-ORM-REL-LOAD-042

Proxy identity será canonical entity identity.

## DB-ORM-REL-LOAD-043

Proxy initialization no creará segunda entidad lógica.

## DB-ORM-REL-LOAD-044

Eager loading no implicará JOIN obligatorio.

## DB-ORM-REL-LOAD-045

JOIN strategy deberá considerar row multiplication.

## DB-ORM-REL-LOAD-046

Multiple to-many JOINs deberán considerar cartesian amplification.

## DB-ORM-REL-LOAD-047

Select-In respetará parameter limits.

## DB-ORM-REL-LOAD-048

Select-In soportará composite identifiers.

## DB-ORM-REL-LOAD-049

Select-In chunk size será capability-driven.

## DB-ORM-REL-LOAD-050

Batch loading solo agrupará requests compatibles.

## DB-ORM-REL-LOAD-051

Batch loading no mezclará tenants por defecto.

## DB-ORM-REL-LOAD-052

Batch loading no mezclará shards incompatibles.

## DB-ORM-REL-LOAD-053

Owners sin rows quedarán known-empty si fueron completamente cubiertos.

## DB-ORM-REL-LOAD-054

Requested owners serán distintos de owners appearing in rows.

## DB-ORM-REL-LOAD-055

Incomplete read no marcará COMPLETE.

## DB-ORM-REL-LOAD-056

Cursor failure no fabricará complete coverage.

## DB-ORM-REL-LOAD-057

Streaming collection se finalizará solo en safe boundary.

## DB-ORM-REL-LOAD-058

Grouped streaming requerirá ordering suficiente o buffering.

## DB-ORM-REL-LOAD-059

RelationshipLoadPlan será distinto de QueryPlan.

## DB-ORM-REL-LOAD-060

RelationshipLoadPlan podrá reutilizar HydrationPlan.

## DB-ORM-REL-LOAD-061

HydrationPlan no ejecutará relationship queries.

## DB-ORM-REL-LOAD-062

Already joined data deberá reutilizarse antes de query adicional.

## DB-ORM-REL-LOAD-063

Partial joined data no se presentará como complete.

## DB-ORM-REL-LOAD-064

PARTIAL→COMPLETE será transición explícita.

## DB-ORM-REL-LOAD-065

Normal load preservará local dirty relationship changes.

## DB-ORM-REL-LOAD-066

Refresh dirty state requerirá policy explícita.

## DB-ORM-REL-LOAD-067

Database query no sobrescribirá local relationship mutations silenciosamente.

## DB-ORM-REL-LOAD-068

Complete clean load podrá establecer relationship snapshot.

## DB-ORM-REL-LOAD-069

Partial load no creará fake complete baseline.

## DB-ORM-REL-LOAD-070

Entity postLoad no se repetirá por duplicate JOIN row.

## DB-ORM-REL-LOAD-071

Relationship logical load event no será physical-row event.

## DB-ORM-REL-LOAD-072

Cyclic object graphs estarán permitidos.

## DB-ORM-REL-LOAD-073

Recursive loading infinito estará prohibido.

## DB-ORM-REL-LOAD-074

RelationshipLoadSession detectará active compatible loads.

## DB-ORM-REL-LOAD-075

Fetch graph será finite y validated.

## DB-ORM-REL-LOAD-076

Depth limit no sustituirá cycle detection.

## DB-ORM-REL-LOAD-077

Fetch strategy podrá variar por graph node.

## DB-ORM-REL-LOAD-078

Fetch planner no será DB Query Optimizer.

## DB-ORM-REL-LOAD-079

Lazy loads emitirán telemetría suficiente para N+1 analysis.

## DB-ORM-REL-LOAD-080

N+1 detector no alterará resultados silenciosamente.

## DB-ORM-REL-LOAD-081

Automatic batching futuro preservará semantics.

## DB-ORM-REL-LOAD-082

V1 no dependerá de asynchronous magic para correctness.

## DB-ORM-REL-LOAD-083

Relationship Loader transportará ReadIntent.

## DB-ORM-REL-LOAD-084

Relationship Loader no elegirá replica directamente.

## DB-ORM-REL-LOAD-085

Read/Write Router decidirá connection routing.

## DB-ORM-REL-LOAD-086

Relationship loads dentro de transaction respetarán transaction context.

## DB-ORM-REL-LOAD-087

Read-your-writes podrá requerir primary/sticky routing.

## DB-ORM-REL-LOAD-088

Managed target refresh será gobernado por hydration policy.

## DB-ORM-REL-LOAD-089

STALE será distinto de UNINITIALIZED.

## DB-ORM-REL-LOAD-090

Relationship invalidation podrá incrementar generation.

## DB-ORM-REL-LOAD-091

Cancellation será first-class.

## DB-ORM-REL-LOAD-092

Cancellation antes de query no alterará coverage.

## DB-ORM-REL-LOAD-093

Cancellation durante incomplete load no producirá COMPLETE.

## DB-ORM-REL-LOAD-094

Hydrated targets válidos no se borrarán ficticiamente por assembly failure.

## DB-ORM-REL-LOAD-095

Buffered relationship state se publicará después de validación/finalización.

## DB-ORM-REL-LOAD-096

Resource limit no truncará silenciosamente una relación completa.

## DB-ORM-REL-LOAD-097

EXTRA_LAZY operation no implicará collection initialization.

## DB-ORM-REL-LOAD-098

Known count será distinto de known members.

## DB-ORM-REL-LOAD-099

Known membership fact será distinto de complete coverage.

## DB-ORM-REL-LOAD-100

Paginated relationship result no inicializará complete managed collection por default.

## DB-ORM-REL-LOAD-101

Filtered relationship query no reemplazará complete collection automáticamente.

## DB-ORM-REL-LOAD-102

Relationship query result podrá reutilizar managed entities sin cambiar coverage.

## DB-ORM-REL-LOAD-103

Metadata ordering será distinto de query-specific ordering.

## DB-ORM-REL-LOAD-104

Indexed collection detectará logical key collisions.

## DB-ORM-REL-LOAD-105

Relationship Loading no introducirá hidden second-level cache.

## DB-ORM-REL-LOAD-106

Hydration Cache será distinto de Relationship Cache.

## DB-ORM-REL-LOAD-107

Result Cache será distinto de managed relationship state.

## DB-ORM-REL-LOAD-108

Immutable metadata/load plans podrán compartirse entre workers.

## DB-ORM-REL-LOAD-109

Mutable relationship state será request/operation scoped.

## DB-ORM-REL-LOAD-110

EntityManager no se almacenará en shared load plans.

## DB-ORM-REL-LOAD-111

IdentityMap no se almacenará en shared load plans.

## DB-ORM-REL-LOAD-112

ResultCursor no se almacenará en shared plan cache.

## DB-ORM-REL-LOAD-113

Current tenant object no se almacenará en shared plan cache.

## DB-ORM-REL-LOAD-114

RelationshipLoadSession no será process-global.

## DB-ORM-REL-LOAD-115

Same mutable load session no será concurrent-safe por defecto.

## DB-ORM-REL-LOAD-116

V1 priorizará deterministic sequential reconciliation.

## DB-ORM-REL-LOAD-117

Load plan cacheability requerirá immutability.

## DB-ORM-REL-LOAD-118

Owner runtime IDs no pertenecerán al plan fingerprint cuando no cambien shape.

## DB-ORM-REL-LOAD-119

Strategy selection será deterministic.

## DB-ORM-REL-LOAD-120

Strategy registry no dependerá de accidental registration order.

## DB-ORM-REL-LOAD-121

Coordinator no generará SQL.

## DB-ORM-REL-LOAD-122

Coordinator no hará commit.

## DB-ORM-REL-LOAD-123

Assembler no ejecutará hidden queries.

## DB-ORM-REL-LOAD-124

Coverage resolver utilizará execution evidence.

## DB-ORM-REL-LOAD-125

EAGER no implicará automáticamente COMPLETE.

## DB-ORM-REL-LOAD-126

Complete coverage requerirá semantic full coverage.

## DB-ORM-REL-LOAD-127

Complete coverage requerirá successful physical completion.

## DB-ORM-REL-LOAD-128

Complete coverage requerirá successful assembly.

## DB-ORM-REL-LOAD-129

Complete coverage requerirá absence of truncation.

## DB-ORM-REL-LOAD-130

Zero rows podrán significar known-empty cuando request cubrió al owner completamente.

## DB-ORM-REL-LOAD-131

Relationship Loading producirá knowledge + coverage.

## DB-ORM-REL-LOAD-132

Change Tracking decidirá cambios, no Loading.

## DB-ORM-REL-LOAD-133

Persistence consumirá knowledge/coverage, no Loading SQL.

## DB-ORM-REL-LOAD-134

Loading nunca inferirá persistence operation.

## DB-ORM-REL-LOAD-135

Loading nunca hará automatic flush.

## DB-ORM-REL-LOAD-136

Loading nunca hará automatic commit.

## DB-ORM-REL-LOAD-137

Loading nunca hará automatic entity persist.

## DB-ORM-REL-LOAD-138

Loading nunca reattachará detached entity silenciosamente.

## DB-ORM-REL-LOAD-139

Loading respetará EntityState.

## DB-ORM-REL-LOAD-140

Loading respetará tenant identity domain.

## DB-ORM-REL-LOAD-141

Loading respetará shard context.

## DB-ORM-REL-LOAD-142

Loading respetará DatabaseContext.

## DB-ORM-REL-LOAD-143

Loading respetará cancellation/deadline.

## DB-ORM-REL-LOAD-144

Loading respetará resource governance.

## DB-ORM-REL-LOAD-145

Loading preservará deterministic logical result semantics.

## DB-ORM-REL-LOAD-146

Loading telemetry no expondrá PII por defecto.

## DB-ORM-REL-LOAD-147

Loading diagnostics explicarán effective strategy.

## DB-ORM-REL-LOAD-148

Loading diagnostics explicarán resulting coverage.

## DB-ORM-REL-LOAD-149

Loading diagnostics explicarán query count y batching.

## DB-ORM-REL-LOAD-150

Loading preservará separación ORM → Query Engine → Compiler → Executor.

## DB-ORM-REL-LOAD-151

Relationship load errors no fabricarán clean state.

## DB-ORM-REL-LOAD-152

Incomplete graph assembly no escapará como graph completo.

## DB-ORM-REL-LOAD-153

Owner grouping utilizará canonical identity.

## DB-ORM-REL-LOAD-154

Target grouping utilizará canonical identity.

## DB-ORM-REL-LOAD-155

Collection membership no utilizará PHP loose equality.

## DB-ORM-REL-LOAD-156

Relationship load state será explícitamente observable internamente.

## DB-ORM-REL-LOAD-157

Relationship loading será testeable sin driver concreto.

## DB-ORM-REL-LOAD-158

Relationship loading será testeable sin SQL strings.

## DB-ORM-REL-LOAD-159

Relationship load plans serán explicables.

## DB-ORM-REL-LOAD-160

VoltStack preservará conocimiento parcial antes que fabricar conocimiento completo.

---

# 248. Anti-Patterns

## 248.1 Query desde getter de entidad

```php
public function getPosts(): array
{
    return PDO::query(...);
}
```

**Rechazado.**

---

## 248.2 SQL en PersistentCollection

```php
public function initialize(): void
{
    $pdo->query(...);
}
```

**Rechazado.**

La colección puede solicitar inicialización a infraestructura scoped, pero no conocer SQL/driver.

---

## 248.3 Eager = JOIN siempre

**Rechazado.**

---

## 248.4 Empty = not loaded

**Rechazado.**

---

## 248.5 Partial = complete

**Rechazado.**

---

## 248.6 LIMIT + COMPLETE

Sin evidencia adicional:

**Rechazado.**

---

## 248.7 Lazy load detached entity usando global manager

**Rechazado.**

---

## 248.8 Crear target duplicado ignorando IdentityMap

**Rechazado.**

---

## 248.9 Sobrescribir dirty collection al cargar

**Rechazado.**

---

## 248.10 Marcar COMPLETE después de cursor failure

**Rechazado.**

---

## 248.11 Mezclar tenants en batch

**Rechazado.**

---

## 248.12 N+1 invisible

Lazy loading sin observabilidad:

**Rechazado como diseño recomendado.**

---

# 249. Ejemplo integral — Lazy Many-to-One

```php
$post = $posts->find(100);

$author = $post->author;
```

Si `author` está `UNINITIALIZED`:

```text
Post#100
    ↓
LazyRelationshipInitializer
    ↓
RelationshipLoadRequest
    │
    ├── Post.author
    ├── owner Post#100
    └── LAZY
    ↓
author_id loaded?
    ↓ yes: 42
IdentityMap User#42?
    ├── HIT
    │    ↓
    │ User#42
    │
    └── MISS
         ↓
      EntityQuery(User#42)
         ↓
      Query Engine
         ↓
      Execution
         ↓
      EntityHydrator
         ↓
      IdentityMap
         ↓
      User#42
         ↓
Post.author = canonical User#42
```

---

# 250. Ejemplo integral — Eager Select-In

```php
$users = User::query()
    ->with('posts')
    ->get();
```

VoltStack puede resolver:

```text
Query 1
Users
    ↓
User IDs:
1,2,3,4,5
    ↓
RelationshipFetchPlanner
    ↓
SELECT_IN
    ↓
Query 2
Posts for owners 1..5
    ↓
Hydration
    ↓
Group by Post.author identity
    ↓
User#1.posts
User#2.posts
User#3.posts
User#4.posts
User#5.posts
    ↓
coverage = COMPLETE
```

si la relación fue cubierta completamente.

---

# 251. Ejemplo — Owner sin miembros

Owners:

```text
User#1
User#2
User#3
```

Query devuelve posts únicamente para:

```text
User#1
User#3
```

Si el query cubrió todos los owners:

```text
User#2.posts
=
[]
+
COMPLETE
```

No `UNINITIALIZED`.

---

# 252. Ejemplo — Partial relationship

```php
$latestPosts = $user
    ->postsRelation()
    ->orderByDesc('createdAt')
    ->limit(10)
    ->get();
```

Resultado:

```text
10 Post entities
```

pueden ser managed.

Pero:

```text
User.posts coverage
```

no deberá cambiar automáticamente a `COMPLETE`.

---

# 253. Ejemplo — Dirty merge

Baseline:

```text
User.roles:
    Admin
    Editor
```

Local:

```text
+ Auditor
```

DB load devuelve:

```text
Admin
Editor
```

Normal loading:

```text
Observed DB baseline:
    Admin
    Editor

Local mutation:
    + Auditor

Current object graph:
    Admin
    Editor
    Auditor
```

No se pierde el cambio local.

---

# 254. Ejemplo — Failed streaming

Relación esperada:

```text
User.posts
```

Se leen:

```text
Post 1
Post 2
Post 3
```

y luego:

```text
connection failure
```

Resultado:

```text
Observed:
    Post1, Post2, Post3

Coverage:
    PARTIAL / failed load

COMPLETE:
    false
```

---

# 255. Ejemplo — Polymorphic Batch

Owners:

```text
Comment#1 → post:10
Comment#2 → video:7
Comment#3 → post:12
Comment#4 → video:9
```

Planner:

```text
Group "post"
    → load Post [10,12]

Group "video"
    → load Video [7,9]
```

Después:

```text
IdentityMap reconciliation
        ↓
Comment#1.commentable = Post#10
Comment#2.commentable = Video#7
Comment#3.commentable = Post#12
Comment#4.commentable = Video#9
```

---

# 256. Fórmulas fundamentales

## Load Need

```text
NeedLoad
=
RequestedKnowledge
-
CurrentlyReliableKnowledge
```

---

## To-One Known Null

```text
LoadedFK
∧
FK = NULL
→
LOADED_NULL
```

---

## Missing FK

```text
¬LoadedFK
→
UnknownReference
```

no:

```text
→ NULL reference
```

---

## Complete Coverage

```text
CompleteCoverage
=
FullRelationshipSemantics
∧
ExecutionCompleted
∧
AssemblyCompleted
∧
NoTruncation
```

---

## Partial Coverage

```text
PartialCoverage
=
ObservedRelationshipKnowledge
∧
¬CompleteCoverage
```

---

## Batch Compatibility

```text
BatchCompatible(A,B)
⇔
SameRelationship
∧
CompatibleDatabaseContext
∧
CompatibleTenant
∧
CompatibleShard
∧
CompatibleLoadSemantics
∧
CompatibleConsistency
```

---

## Select-In Chunk

```text
OwnersPerBatch
≤
floor(
    ParameterBudget
    /
    IdentifierParameterArity
)
```

---

## Safe Streaming Finalization

```text
MayFinalizeRelationship(owner)
⇔
RequiredRowsConsumed
∧
NoFutureRowCanExtend(owner)
```

---

## Safe Relationship Load

```text
SafeLoad
=
ValidMetadata
∧
ValidOwnerContext
∧
CanonicalIdentity
∧
ValidHydration
∧
ValidAssembly
∧
EvidenceBasedCoverage
∧
LocalMutationSafety
```

---

# 257. Master Formula

```text
Relationship Loading System
=
Relationship Metadata
+
Load Requests
+
Fetch Policies
+
Strategy Resolution
+
Explicit Loading
+
Governed Lazy Loading
+
Eager Loading
+
Join Loading
+
Select-In Loading
+
Batch Loading
+
Polymorphic Loading
+
Fetch Graph Planning
+
Entity Query Integration
+
Hydration Plan Integration
+
IdentityMap Reuse
+
Relationship Assembly
+
Canonical Membership
+
Load State
+
Collection Coverage
+
Relationship Snapshots
+
Dirty-State Merge Policies
+
Cycle Protection
+
Streaming Boundaries
+
Read Consistency
+
Cancellation
+
Resource Governance
+
Persistent Runtime Isolation
+
Telemetry
+
N+1 Observation Hooks
+
Diagnostics
+
Testing
```

---

# 258. Regla maestra final

> **Relationship Loading no significa “ejecutar una consulta cuando se accede a una propiedad”. Significa resolver una solicitud de conocimiento relacional mediante una estrategia gobernada, obtener los datos por el Query Engine, materializarlos por Hydration, reutilizar la identidad canónica del PersistenceContext, ensamblarlos sin destruir cambios locales y declarar únicamente la cobertura que la evidencia realmente permite afirmar.**

La cadena canónica será:

```text
Relationship Access / Fetch Requirement
                ↓
RelationshipLoadRequest
                ↓
RelationshipFetchPlanner
                ↓
RelationshipLoadPlan
                ↓
Entity Query / Query Model
                ↓
Query Engine
                ↓
SQL Compiler
                ↓
Execution Engine
                ↓
Result
                ↓
Hydration System
                ↓
IdentityMap
                ↓
Relationship Assembly
                ↓
Merge With Local State
                ↓
Coverage Finalization
                ↓
Relationship Snapshot / Runtime State
                ↓
Application Object Graph
```

Esto conserva la separación fundamental:

```text
Loading
knows relationship semantics

Query Engine
knows queries

Compiler
knows SQL

Executor
knows execution

Hydration
knows materialization

IdentityMap
knows canonical entity identity

PersistentCollection
knows runtime collection state

Persistence
knows what relationship changes must be written
```

---

# 259. Resultado arquitectónico

Con esta arquitectura, VoltStack podrá ofrecer una API cómoda:

```php
$users = User::query()
    ->with('posts.comments', 'roles')
    ->get();
```

o:

```php
$entityManager->load($user, 'posts');
```

e incluso lazy loading controlado:

```php
$user->posts;
```

sin convertir esa ergonomía en una arquitectura acoplada a SQL.

Internamente podrá decidir:

```text
JOIN
SELECT_IN
BATCH
EXPLICIT_SELECT
LAZY_SELECT
POLYMORPHIC_BATCH
```

manteniendo invariantes comunes:

```text
same entity identity
same metadata
same Query Engine
same Hydration System
same PersistenceContext
same relationship coverage semantics
```

y preparando directamente los siguientes sistemas especializados de optimización de carga.

---

# 260. Siguiente documento

```text
151_DATABASE_EAGER_LOADING_SYSTEM.md
```

El siguiente documento deberá especializar la arquitectura anterior en la carga anticipada de relaciones, definiendo:

- eager fetch requirements;
- `with()` / fetch graph API;
- nested eager loading;
- eager load planning;
- JOIN vs SELECT-IN;
- multiple to-many explosion;
- cartesian amplification;
- strategy selection;
- eager load batching;
- owner grouping;
- nested relationship scheduling;
- partial vs complete eager results;
- filtered eager loading;
- constrained eager loading;
- polymorphic eager loading;
- aggregate eager loading;
- column projection;
- hydration plan composition;
- result assembly;
- IdentityMap reuse;
- ordering;
- pagination interaction;
- streaming compatibility;
- resource budgets;
- cancellation;
- read consistency;
- N+1 elimination;
- telemetry;
- persistent runtime safety.

Regla central propuesta:

> **Eager Loading expresa qué relaciones deben estar disponibles al completar una operación de lectura; no prescribe SQL JOIN. VoltStack deberá seleccionar y combinar estrategias de carga que satisfagan el fetch graph preservando cardinalidad lógica, identidad canónica, cobertura, límites de recursos y semántica de la consulta raíz.**