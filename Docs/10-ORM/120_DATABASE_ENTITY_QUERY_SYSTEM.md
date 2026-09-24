# 120_DATABASE_ENTITY_QUERY_SYSTEM.md

# VoltStack Quantum Database
## Database Entity Query System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 120 — Database Entity Query System  
**Bloque:** 10 — ORM  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Entity Query System` define la capa de consultas orientada a entidades dentro del ORM de VoltStack.

Su objetivo es permitir que la aplicación exprese consultas en términos de:

- entidades;
- campos;
- value objects;
- relaciones;
- colecciones;
- proyecciones;
- agregaciones;
- ordenamiento;
- scopes;
- criterios;

sin conocer:

- nombres físicos de tablas;
- nombres físicos de columnas;
- aliases SQL;
- dialectos;
- placeholders;
- parámetros driver-specific;
- SQL generado.

Ejemplo:

```php id="a7gwdr"
$users = $repository
    ->query()
    ->where('status', UserStatus::ACTIVE)
    ->where('profile.country', CountryCode::MX)
    ->with('roles')
    ->orderBy('createdAt', 'DESC')
    ->limit(100)
    ->get();
```

La consulta deberá convertirse en:

```text id="zi7df8"
Entity Query
      ↓
Entity Semantic Resolution
      ↓
Entity Metadata / Mapping
      ↓
Database Query Model / AST
      ↓
Semantic Query Engine
      ↓
Optimizer
      ↓
Planner
      ↓
SQL Compiler
      ↓
Executor
      ↓
Hydration / Projection
```

Principio central:

> **Entity Query expresa intención de consulta sobre el modelo de entidades y la traduce al Query Model universal de VoltStack; nunca genera ni ejecuta SQL directamente.**

---

# 2. Entity Query ≠ Repository

El Repository representa:

```text id="sb11mx"
logical entity collection
```

El Entity Query representa:

```text id="v3ll6a"
one structured query against that logical collection
```

Por tanto:

```text id="20gfsy"
Repository<User>
    ↓
query()
    ↓
EntityQuery<User>
```

---

# 3. Entity Query ≠ EntityManager

El EntityManager coordina:

```text id="8ht9ly"
PersistenceContext
IdentityMap
UnitOfWork
entity loading
flush
```

Entity Query solo expresa y ejecuta consultas de entidades a través de los subsistemas apropiados.

---

# 4. Entity Query ≠ Database Query Builder

Esta separación será fundamental.

`Database Query Builder` trabaja con:

```text id="sbpe4a"
tables
columns
expressions
joins
query AST
```

`Entity Query` trabaja con:

```text id="sl8j5p"
EntityType
entity field
relationship path
embedded field
entity projection
entity hydration mode
```

---

# 5. Entity Query ≠ SQL

Nunca deberá existir como arquitectura principal:

```php id="26442b"
$query->toSqlAndExecute();
```

La ruta correcta será:

```text id="5caf4c"
Entity Query
→ Query Model
→ Query Engine
→ Compiler
→ Executor
```

---

# 6. Entity Query ≠ Hydrator

Entity Query define:

```text id="sdj4jj"
result semantics
```

por ejemplo:

```text id="91zlf1"
managed entities
read-only entities
projection DTO
scalar
tuple
```

pero la reconstrucción real pertenece al Hydration System.

---

# 7. Entity Query ≠ Persistence Engine

Entity Query es principalmente una abstracción de lectura.

Las mutaciones bulk que eventualmente soporte deberán pasar por una capa explícita y gobernada.

No deberán convertir EntityQuery en Persistence Engine.

---

# 8. Posición arquitectónica

```text id="12lb7f"
Application
│
├── Model API
│
└── Repository
        │
        ▼
    EntityQuery
        │
        ├── Entity Metadata
        ├── Entity Mapping
        ├── Relationship Metadata
        ├── Type System
        └── Entity Scope Registry
                │
                ▼
      Entity Query Translator
                │
                ▼
       Database Query Model
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
        ┌───────┴─────────┐
        ▼                 ▼
    Hydration         Projection
        │
        ▼
   IdentityMap
```

---

# 9. Objetivos

El sistema deberá proporcionar:

- `EntityQuery<TEntity>`;
- `EntityQueryFactory`;
- referencias tipadas a campos;
- paths de relaciones;
- predicates;
- ordering;
- projections;
- aggregations;
- grouping;
- relationship-aware queries;
- eager-loading intents;
- result modes;
- hydration modes;
- scopes;
- criteria integration;
- specification integration;
- query translation;
- semantic validation;
- parameter conversion;
- debugging;
- extensibilidad;
- persistent-runtime safety.

---

# 10. No objetivos

Entity Query no deberá:

- descubrir mappings vía Reflection;
- construir EntityMetadata;
- modificar mappings;
- generar SQL;
- ejecutar driver commands;
- abrir Connection directamente;
- administrar UnitOfWork;
- implementar IdentityMap;
- ejecutar migrations;
- resolver autorización implícitamente;
- hardcodear tenants;
- hardcodear vendors.

---

# 11. Modelo conceptual

```text id="791d2n"
EntityQuery
=
Root Entity
+
Selection
+
Predicates
+
Relationship Paths
+
Grouping
+
Ordering
+
Pagination/Limit
+
Loading Intent
+
Result Mode
+
Execution Context
```

---

# 12. EntityQuery<TEntity>

Contrato conceptual:

```php id="7sajzh"
/**
 * @template TEntity of object
 */
interface EntityQuery
{
    public function where(
        string|EntityFieldReference $field,
        mixed $operatorOrValue,
        mixed $value = null,
    ): static;

    public function orderBy(
        string|EntityFieldReference $field,
        string $direction = 'ASC',
    ): static;

    public function limit(int $limit): static;

    public function offset(int $offset): static;

    public function with(string ...$relationships): static;

    /**
     * @return list<TEntity>
     */
    public function get(): array;

    /**
     * @return TEntity|null
     */
    public function oneOrNull(): ?object;
}
```

La interfaz final deberá evitar firmas ambiguas donde sea posible.

---

# 13. API ergonomics vs model interno

La API pública podrá ser compacta:

```php id="thgp9f"
$query->where('status', $status);
```

pero internamente deberá transformarse a nodos tipados:

```text id="j1gnsa"
EntityPredicate
├── field: EntityFieldReference(User.status)
├── operator: EQUAL
└── value: TypedEntityValue(UserStatus)
```

---

# 14. EntityQueryFactory

Se introduce:

```text id="ixxzen"
EntityQueryFactory
```

Contrato:

```php id="h68myt"
/**
 * @template TEntity of object
 */
interface EntityQueryFactory
{
    /**
     * @param class-string<TEntity>|EntityType $entity
     * @return EntityQuery<TEntity>
     */
    public function for(
        string|EntityType $entity,
    ): EntityQuery;
}
```

---

# 15. Factory responsibilities

La factory deberá resolver:

```text id="d39r75"
EntityType
EntityMetadata
EntityQueryContext
default scopes
result mode defaults
```

y producir un query object limpio.

---

# 16. Query state

Cada query deberá mantener su propio estado.

Nunca:

```text id="bkxqa1"
Repository::$currentWhere
```

ni:

```text id="ixk05r"
EntityQueryFactory::$lastQuery
```

---

# 17. Immutability

Se recomienda fuertemente un diseño:

```text id="5s22gc"
immutable query object
```

Ejemplo:

```php id="peki04"
$base = $users->query();

$active = $base->where(
    'status',
    UserStatus::ACTIVE,
);

$inactive = $base->where(
    'status',
    UserStatus::INACTIVE,
);
```

`$base` permanece sin modificar.

---

# 18. Why immutable

Beneficios:

- query reuse segura;
- menos side effects;
- mejor persistent-runtime safety;
- mejor debugging;
- fingerprints deterministas;
- composición más sencilla;
- testing más simple.

---

# 19. Fluent API

Inmutabilidad no impide:

```php id="sn5fqk"
$users
    ->query()
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

Cada operación puede devolver una nueva instancia.

---

# 20. EntityQueryContext

Se propone:

```text id="cpwrvm"
EntityQueryContext
```

---

# 21. Context contents

Puede contener:

```text id="7wekxa"
EntityManagerScopeId
DatabaseContext
EntityMetadataRegistry reference
scope set
consistency requirement
result mode
query hints
deadline/cancellation context
```

---

# 22. Context must not contain global mutable state

No:

```text id="vsa9tw"
static current tenant
static current query
static current connection
static current user
```

---

# 23. Root Entity

Toda EntityQuery tendrá un root:

```text id="wfy0h6"
RootEntityReference
```

Ejemplo:

```text id="fpegib"
EntityQuery<User>
root = User
```

---

# 24. Root entity ≠ table alias

El root:

```text id="hoisry"
User
```

no equivale a:

```text id="ff8jl5"
u
```

SQL alias.

Los aliases físicos se decidirán posteriormente.

---

# 25. EntityFieldReference

Se introduce:

```text id="h1pal9"
EntityFieldReference
```

Ejemplo:

```text id="womqs2"
User.status
```

---

# 26. Field reference model

```php id="rpddhc"
final readonly class EntityFieldReference
{
    public function __construct(
        public EntityType $entityType,
        public FieldPath $path,
    ) {}
}
```

---

# 27. FieldPath

Ejemplos:

```text id="x7r1wo"
email
profile.country
address.city
organization.owner.name
```

---

# 28. FieldPath taxonomy

Un path puede atravesar:

```text id="w2vo2t"
scalar field
embedded field
relationship
embedded inside relationship
```

---

# 29. String paths

La API ergonómica puede aceptar:

```php id="t2guxr"
'profile.country'
```

pero deberá resolver inmediatamente a una representación tipada antes de llegar al Query Engine.

---

# 30. Unknown path

Ejemplo:

```php id="7odv2u"
$query->where(
    'profile.doesNotExist',
    123,
);
```

deberá producir:

```text id="8m4luw"
UnknownEntityFieldPathException
```

antes de SQL.

---

# 31. EntityPathResolver

Se introduce:

```text id="m7gfu9"
EntityPathResolver
```

---

# 32. Resolution

```text id="14avzx"
User.profile.country
       │
       ▼
User metadata
       ↓
profile RelationshipMetadata
       ↓
Profile metadata
       ↓
country FieldMetadata
```

---

# 33. Relationship path resolution

Los joins necesarios deberán derivarse de la semántica del path.

Ejemplo:

```php id="7rp5qo"
$query->where(
    'organization.country.code',
    'MX',
);
```

---

# 34. Implicit logical joins

El sistema podrá crear:

```text id="72huy9"
logical relationship traversal
```

pero no deberá decidir aún:

```text id="ffgv8r"
physical join order
```

Eso pertenece a Query Planner/Optimizer.

---

# 35. Join semantics

La traducción deberá indicar la semántica necesaria:

```text id="5xdaci"
relationship traversal
null-preserving requirement
existence semantics
collection traversal
```

---

# 36. To-one traversal

Ejemplo:

```text id="8v08a1"
Post.author.email
```

es relativamente directo:

```text id="tfcdwv"
Post
→ author
→ User
→ email
```

---

# 37. To-many traversal

Ejemplo:

```php id="e2wblq"
User::query()
    ->where(
        'orders.status',
        OrderStatus::PENDING,
    );
```

posee semántica más compleja.

Puede significar:

```text id="w6cer5"
exists related Order
where status = PENDING
```

y no simplemente un scalar join naïve.

---

# 38. Collection path semantics

Para relaciones to-many deberá definirse explícitamente si:

```text id="zxx95n"
where('orders.status', PENDING)
```

significa:

```text id="v6dxei"
EXISTS matching relation
```

Recomendación:

```text id="cf2lwr"
to-many predicate traversal
→ EXISTS semantics by default
```

para evitar duplicación del root result.

---

# 39. Explicit relation predicates

También podrán existir APIs como:

```php id="fu6uqc"
$query->whereHas(
    'orders',
    fn (EntityQuery $orders) =>
        $orders->where(
            'status',
            OrderStatus::PENDING
        )
);
```

---

# 40. `whereHas()`

Semánticamente:

```text id="n5e41g"
EXISTS related entities satisfying predicate
```

---

# 41. `whereDoesntHave()`

Semánticamente:

```text id="idnfka"
NOT EXISTS
```

---

# 42. Relationship cardinality

Entity Query deberá consultar metadata para conocer:

```text id="fxu4gz"
ONE_TO_ONE
MANY_TO_ONE
ONE_TO_MANY
MANY_TO_MANY
```

y construir una traducción adecuada.

---

# 43. EntityPredicate

Se introduce:

```text id="iq1y7l"
EntityPredicate
```

---

# 44. Predicate taxonomy

```text id="ip7016"
Comparison
Null
In
Between
Like
Exists
RelationshipExists
And
Or
Not
Custom
```

---

# 45. Comparison

Ejemplo:

```php id="3i75sy"
$query->where('age', '>=', 18);
```

produce:

```text id="fat11i"
EntityComparisonPredicate
├── field: User.age
├── operator: GREATER_OR_EQUAL
└── value: 18
```

---

# 46. Operator enum

Internamente:

```text id="9jdrmc"
EQUAL
NOT_EQUAL
GREATER_THAN
GREATER_OR_EQUAL
LESS_THAN
LESS_OR_EQUAL
LIKE
NOT_LIKE
```

No depender de strings sin normalizar.

---

# 47. `whereNull()`

```php id="ye8xrw"
$query->whereNull('deletedAt');
```

produce:

```text id="npvniz"
NullPredicate
```

---

# 48. Null semantics

No traducir:

```text id="eilwko"
field = NULL
```

como una comparación normal.

Debe conservar semántica SQL/relacional correcta en el Query Model.

---

# 49. `whereIn()`

```php id="ny51rw"
$query->whereIn('status', [
    Status::ACTIVE,
    Status::PENDING,
]);
```

---

# 50. Empty IN

Debe definirse:

```text id="mnmh00"
IN []
→ FALSE
```

o el comportamiento canónico que adopte Query Engine.

No generar SQL inválido.

---

# 51. `whereBetween()`

```php id="oycfny"
$query->whereBetween(
    'createdAt',
    $start,
    $end,
);
```

---

# 52. Logical predicates

```php id="blqwze"
$query
    ->where('active', true)
    ->orWhere('role', Role::ADMIN);
```

internamente deberá construir un árbol.

---

# 53. Avoid precedence ambiguity

Una API fluida simple puede producir ambigüedad.

Se recomienda soportar grupos:

```php id="8y3tgr"
$query->whereGroup(
    fn (EntityPredicateBuilder $p) =>
        $p
            ->where('active', true)
            ->orWhere('role', Role::ADMIN)
);
```

---

# 54. Predicate tree

```text id="m9m8ta"
AND
├── status = ACTIVE
└── OR
    ├── country = MX
    └── role = ADMIN
```

---

# 55. Parameter values

Los valores deberán mantenerse separados de la estructura query.

```text id="yahcif"
Entity Predicate
+
Typed Parameters
```

---

# 56. Parameter conversion

Ejemplo:

```text id="vjduts"
UserStatus::ACTIVE
       ↓
Entity Type Mapping
       ↓
Database Type Conversion
       ↓
Bound Parameter
```

---

# 57. Entity Query never interpolates values

Nunca:

```text id="54py6z"
"status = '$status'"
```

---

# 58. TypedEntityParameter

Podrá existir:

```text id="j0cbav"
TypedEntityParameter
```

con:

```text id="dxmrxa"
entity field type
domain value
nullability
conversion descriptor
```

---

# 59. Parameter identity

Un Query AST parameter debe ser distinto del valor serializado.

Esto permitirá:

```text id="j68vvx"
prepared statements
compiled query cache
query deduplication
```

---

# 60. Entity expressions

El sistema podrá ofrecer expresiones tipadas:

```text id="8lweir"
EntityFieldExpression
EntityLiteralExpression
EntityFunctionExpression
EntityAggregateExpression
EntityCaseExpression
```

---

# 61. Expression delegation

No deberá recrearse un segundo expression language completo si puede adaptarse al Query Expression System de los documentos `27+`.

---

# 62. Entity expression layer

La capa ORM deberá limitarse a:

```text id="0ar3q3"
entity-aware semantic references
```

y traducirlas a:

```text id="nndv8r"
Database Query Expressions
```

---

# 63. Ordering

```php id="crlkvq"
$query->orderBy(
    'createdAt',
    Direction::DESC,
);
```

---

# 64. Direction

Internamente:

```text id="h8spzu"
ASC
DESC
```

más posibles políticas de:

```text id="o37mle"
NULLS_FIRST
NULLS_LAST
DEFAULT
```

---

# 65. Null ordering

Debe expresarse semánticamente.

No asumir que todos los vendors manejan null ordering igual.

---

# 66. Multiple ordering

```php id="q5to7g"
$query
    ->orderBy('status')
    ->orderBy('createdAt', 'DESC');
```

deberá conservar orden de prioridad explícito.

---

# 67. Limit

```php id="f0uvnk"
$query->limit(50);
```

validará:

```text id="mv2yzd"
limit >= 0
```

---

# 68. Offset

```php id="jo98yq"
$query->offset(100);
```

---

# 69. Offset pagination

Entity Query puede soportarla.

Pero cursor pagination se delegará posteriormente a:

```text id="okc46a"
DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 70. Projection

Entity Query deberá soportar:

```text id="h0neym"
ENTITY
PARTIAL ENTITY
DTO
TUPLE
SCALAR
AGGREGATE
```

con políticas claras.

---

# 71. Default result

```text id="2i2jo3"
EntityQuery<User>
```

deberá retornar por default:

```text id="fkzdlx"
managed User entities
```

---

# 72. Projection API

Ejemplo:

```php id="50e53m"
$users
    ->query()
    ->select(
        'id',
        'name',
        'email'
    )
    ->as(UserSummary::class)
    ->get();
```

---

# 73. DTO projection

Pipeline:

```text id="c0gkvl"
Entity field selection
      ↓
Database Query Model
      ↓
Result
      ↓
Projection Hydrator
      ↓
UserSummary
```

---

# 74. Projection ≠ Entity

`UserSummary` no entra a:

```text id="ky0h13"
IdentityMap
UnitOfWork
```

---

# 75. Partial Entity

Partial entities deberán ser una capacidad explícita y restringida.

No simplemente:

```text id="nevetp"
SELECT only some columns
→ pretend full entity
```

---

# 76. Default recommendation

```text id="k4d4ho"
DTO/Projection
>
Partial Entity
```

para queries parciales.

---

# 77. Scalar

Ejemplo:

```php id="715d80"
$count = $users
    ->query()
    ->count();
```

---

# 78. Tuple

```php id="xjgbfo"
$rows = $users
    ->query()
    ->select(
        'status',
        count('id')->as('total'),
    )
    ->groupBy('status')
    ->tuples();
```

---

# 79. Aggregations

Se deberán reutilizar conceptos del Query Aggregation System:

```text id="awn97g"
COUNT
SUM
AVG
MIN
MAX
```

---

# 80. Entity aggregation

Una expresión:

```text id="61po1l"
count(User.id)
```

deberá resolverse a:

```text id="y0te6j"
mapped field
→ database expression
```

---

# 81. Grouping

```php id="04b61y"
$query
    ->groupBy('status')
    ->select(
        'status',
        count('id')->as('total')
    );
```

---

# 82. Entity query validation

Deberá validar que cualquier field no agregado en selection sea compatible con grouping semantics según Query Engine.

---

# 83. Having

Podrá existir:

```php id="0zo9gf"
$query->having(
    count('id'),
    '>',
    100,
);
```

traducido al Query Model.

---

# 84. Distinct

```php id="xgkqga"
$query->distinct();
```

deberá conservar semántica lógica.

---

# 85. Entity distinct warning

`DISTINCT` sobre entities con eager joins puede ser más complejo que SQL-level distinct.

Por ello deberá diferenciarse cuando sea necesario:

```text id="j6bqb1"
row distinct
entity distinct
```

---

# 86. Entity uniqueness

Hydration + IdentityMap ya colapsan filas duplicadas correspondientes al mismo EntityKey.

Esto no equivale a cambiar automáticamente la semántica SQL.

---

# 87. Eager Loading

Entity Query deberá poder expresar:

```php id="y3tbag"
$query->with(
    'profile',
    'roles',
);
```

---

# 88. Loading intent

`with()` produce:

```text id="x6g5d1"
RelationshipLoadingIntent
```

No necesariamente:

```text id="bc489b"
JOIN
```

---

# 89. Loading strategies

El Relationship Loading System podrá escoger:

```text id="nis55m"
JOIN
SELECT_IN
SUBQUERY
BATCH
MULTI_QUERY
```

según:

```text id="772vhr"
relationship cardinality
result size
capabilities
loading policy
query shape
```

---

# 90. Query plan separation

Entity Query declara:

```text id="jr39q3"
what relationships should be loaded
```

El planner decide:

```text id="q4fxzx"
how to load them
```

---

# 91. Nested eager loading

```php id="jivndv"
$query->with(
    'posts.comments.author'
);
```

deberá resolverse a un:

```text id="3c5kq4"
RelationshipLoadGraph
```

---

# 92. RelationshipLoadGraph

```text id="o1usq4"
User
└── posts
    └── comments
        └── author
```

---

# 93. Load graph ≠ Query join tree

Puede terminar ejecutándose mediante múltiples queries.

---

# 94. Query scopes

Entity Query deberá soportar:

```text id="q6o3k1"
EntityQueryScope
```

---

# 95. Scope examples

```text id="ysj4ye"
SoftDeleteScope
TenantScope
TemporalScope
VisibilityScope
CustomGlobalScope
```

---

# 96. Scope application

```text id="7zgcqj"
EntityQuery
    ↓
Scope Pipeline
    ↓
Effective EntityQuery
```

antes de traducción.

---

# 97. Scope ≠ hidden SQL string

Un scope deberá añadir nodos semánticos:

```text id="yxx1kk"
EntityPredicate
EntityJoinIntent
EntityContextConstraint
```

no concatenar SQL.

---

# 98. Scope ordering

Deberá ser:

```text id="hc86m3"
deterministic
inspectable
```

---

# 99. Scope disabling

Cuando se permita:

```php id="crf8ks"
$query->withoutScope(
    SoftDeleteScope::class
);
```

deberá quedar visible en diagnostics.

---

# 100. Non-disableable scope

Algunas policies podrán marcar:

```text id="gobbfj"
NON_DISABLEABLE
```

por razones de integridad/seguridad, pero esto deberá ser explícito.

---

# 101. Scope ≠ Authorization

Incluso un security-aware scope:

```text id="u3xcm1"
does not automatically replace Authorization System
```

---

# 102. Model local scopes

El documento 114 permitió:

```php id="3apgcr"
User::query()
    ->active()
```

Internamente esto deberá convertirse a un:

```text id="r96h86"
EntityQueryScope
```

o una transformación equivalente tipada.

---

# 103. Repository specifications

El documento 119 introdujo:

```text id="kh8jo0"
EntitySpecification
```

Entity Query será el target natural de:

```php id="to1sby"
$specification->apply($query);
```

---

# 104. Specification composition

```text id="brml1m"
Specification
    ↓
Entity Query predicates
```

No SQL.

---

# 105. Query execution

EntityQuery no deberá ejecutar directamente sobre Connection.

Se introduce:

```text id="19c4ju"
EntityQueryExecutor
```

---

# 106. EntityQueryExecutor

Su responsabilidad:

```text id="pb17xi"
EntityQuery
    ↓
Translate
    ↓
Database Query Model
    ↓
Query Executor
    ↓
ORM Result Processing
```

---

# 107. Executor ≠ Database Query Executor

Separar:

```text id="lo3hjo"
EntityQueryExecutor
```

de:

```text id="tl7f6j"
DatabaseQueryExecutor
```

El primero entiende resultados ORM.

El segundo ejecuta planes/statements.

---

# 108. EntityQueryTranslator

Componente central:

```text id="fa5478"
EntityQueryTranslator
```

---

# 109. Translator inputs

```text id="u2smce"
EntityQuery
EntityMetadataRegistry
EntityMapping
Type System
Relationship Metadata
Scope-expanded query
```

---

# 110. Translator output

Debe producir:

```text id="8yb46k"
Database Query Model / Query AST
```

más, cuando sea necesario:

```text id="957xoq"
EntityResultMapping
HydrationIntent
RelationshipLoadingIntent
```

---

# 111. QueryTranslationResult

Se propone:

```php id="8ymt0i"
final readonly class EntityQueryTranslationResult
{
    public function __construct(
        public QueryModel $query,
        public EntityResultPlan $resultPlan,
        public EntityQueryFingerprint $fingerprint,
    ) {}
}
```

---

# 112. Translation ≠ SQL compilation

Debe mantenerse:

```text id="7z4b90"
Entity Query Translation
→ Query Model

SQL Compilation
→ SQL
```

---

# 113. EntityResultPlan

Describe cómo interpretar el resultado.

Podrá ser:

```text id="db2n3b"
ManagedEntityResultPlan
ReadOnlyEntityResultPlan
DetachedEntityResultPlan
ProjectionResultPlan
ScalarResultPlan
TupleResultPlan
```

---

# 114. Result plan ≠ result

Es una descripción inmutable.

No contiene rows ni entities vivas.

---

# 115. Managed Entity Result

Pipeline:

```text id="gz7l5a"
Query Result Rows
       ↓
Entity Hydration Plan
       ↓
EntityIdentifier
       ↓
EntityKey
       ↓
IdentityMap
       │
       ├── hit → reuse
       └── miss → instantiate/hydrate/register
```

---

# 116. Dirty managed entity policy

Si IdentityMap ya contiene una dirty entity:

```text id="b7w0a1"
normal query result
```

no deberá sobrescribirla automáticamente.

---

# 117. Existing entity hydration

Podrán existir políticas:

```text id="ndmnva"
REUSE_WITHOUT_OVERWRITE_DIRTY
REFRESH_CLEAN_FIELDS
EXPLICIT_REFRESH_ONLY
```

La semántica concreta se definirá con Hydration/State.

---

# 118. Read-only result

```php id="5itahq"
$query->readOnly();
```

podrá evitar:

```text id="w7l1j5"
snapshot tracking
dirty checking
UnitOfWork change tracking
```

---

# 119. Read-only ≠ immutable object

Una entidad PHP mutable puede ser retornada en modo read-only ORM.

Eso significa:

```text id="siqnzd"
changes are not tracked/persisted through this context
```

no que PHP prohíba setters.

---

# 120. Detached result

```php id="r7ozem"
$query->detached();
```

puede producir objetos que no permanecen registrados en IdentityMap/UoW.

---

# 121. Detached duplicate identity

Si el mismo EntityKey ya está managed, producir un detached object distinto deberá ser una operación explícita para evitar confusión.

---

# 122. Projection result

No utiliza IdentityMap.

---

# 123. Query result collection

Para entidades podrá existir:

```text id="lvuy4g"
EntityCollection<TEntity>
```

pero no es obligatorio que `get()` devuelva una clase especial en V1.

---

# 124. Collection benefits

Podría aportar:

```text id="1akpym"
typing
lazy transforms
pagination metadata
read-only semantics
```

pero debe evitar overhead innecesario.

---

# 125. `get()`

Semántica:

```text id="m8f6jl"
execute query
return all results according to result mode
```

---

# 126. `first()`

```php id="4n2dvg"
$query->first();
```

deberá adaptar:

```text id="lqqncv"
LIMIT 1
```

cuando sea semánticamente correcto.

---

# 127. `firstOrFail()`

Podrá lanzar:

```text id="zf7523"
EntityQueryNoResultException
```

---

# 128. `oneOrNull()`

Semántica propuesta:

```text id="hbttc6"
0 rows → null
1 logical entity → entity
>1 logical entity → NonUniqueResultException
```

---

# 129. `one()`

```text id="r0vvii"
0 logical results → NoResultException
1 → result
>1 → NonUniqueResultException
```

---

# 130. Logical result ≠ physical row

Especialmente con joins:

```text id="ow7pij"
one User
+
10 Role rows
```

puede producir:

```text id="mo3cja"
10 physical rows
```

pero:

```text id="9j0dvn"
1 logical entity
```

---

# 131. Count

`count()` deberá definir si cuenta:

```text id="webf0d"
physical rows
logical root entities
distinct logical entities
```

Para EntityQuery root count, default recomendado:

```text id="aiy5hd"
logical root entities
```

---

# 132. Count with joins

Esto puede requerir:

```text id="h28dep"
COUNT DISTINCT root identifier
```

o una estrategia equivalente.

La semántica debe venir primero; el planner decide implementación.

---

# 133. Exists

```php id="0cpakv"
$query->exists();
```

semántica:

```text id="3a4ygf"
Does at least one logical result satisfy the query?
```

---

# 134. Query normalization

Antes de traducción podrá existir:

```text id="0gtjz6"
EntityQueryNormalizer
```

---

# 135. Normalization examples

```text id="x2os5a"
where(field, '=', value)
→ equality predicate

empty AND
→ TRUE

empty OR
→ FALSE

duplicate order
→ policy
```

---

# 136. Normalization ≠ optimization

No confundir:

```text id="xg1dyt"
Entity Query normalization
```

con:

```text id="21pvgl"
Database Query Optimizer
```

---

# 137. Entity semantic validation

Se introduce:

```text id="kih20d"
EntityQueryValidator
```

---

# 138. Validation checks

Debe verificar:

- entity type known;
- fields exist;
- paths valid;
- relationship traversal valid;
- operators compatible;
- values compatible;
- projection valid;
- grouping valid;
- scopes valid;
- result mode compatible;
- loading intents valid.

---

# 139. Type validation

Ejemplo:

```text id="q2fkzi"
User.createdAt
type = DateTimeImmutable
```

y:

```php id="cq7pbu"
->where('createdAt', '>', new stdClass())
```

deberá fallar antes de Query Compilation.

---

# 140. Enum validation

```text id="szcb35"
User.status
→ UserStatus
```

deberá aceptar:

```text id="9fyit4"
UserStatus::ACTIVE
```

y convertir correctamente.

---

# 141. Identifier shortcut

Podrá existir:

```php id="iyypuu"
$query->whereKey($id);
```

---

# 142. `whereKey()`

Debe consultar:

```text id="pwqusb"
EntityIdentifierMetadata
```

y soportar:

```text id="y8w6ow"
simple ID
composite ID
typed ID
```

---

# 143. Composite ID

Ejemplo:

```php id="ee6xxd"
$query->whereKey(
    new OrderLineId(
        orderId: $orderId,
        line: 3,
    )
);
```

---

# 144. `whereKeyIn()`

Podrá optimizar consultas múltiples por identidad.

---

# 145. Polymorphism

Entity Query deberá soportar inheritance/polymorphic semantics posteriormente.

Ejemplo:

```php id="vk72ju"
$payments = $paymentRepository
    ->query()
    ->ofType(CardPayment::class)
    ->get();
```

---

# 146. Type predicate

Podrá generar un:

```text id="0ntm54"
EntityTypePredicate
```

que se traduzca según:

```text id="yzgbmr"
inheritance strategy
discriminator mapping
```

---

# 147. `instanceOf()`

Podrá existir semántica similar para queries polimórficas.

---

# 148. Query inheritance

La Entity Query no deberá conocer directamente:

```text id="ym4wc9"
discriminator SQL
```

Eso deriva del mapping.

---

# 149. Embedded queries

Ejemplo:

```php id="9cypvj"
$query->where(
    'address.city',
    'Monterrey'
);
```

---

# 150. Embedded resolution

```text id="f5b2o9"
User.address.city
       ↓
EmbeddedMapping
       ↓
mapped field(s)
```

---

# 151. Value object comparison

Ejemplo:

```php id="0x61zf"
$query->where(
    'email',
    Email::fromString('ana@example.com')
);
```

Type System realizará conversión.

---

# 152. Multi-column value objects

Un value object como:

```text id="6a5g9u"
Money(amount, currency)
```

puede requerir múltiples columnas.

Entity Query deberá permitir al Mapping/Type System expandir una comparación semántica.

---

# 153. ValueObject predicate expansion

```text id="j196z6"
price = Money(100, MXN)
```

podría traducirse a:

```text id="o8e6bf"
price_amount = 100
AND
price_currency = 'MXN'
```

sin que el caller conozca columnas físicas.

---

# 154. Query functions

La capa entity-aware podrá soportar funciones semánticas:

```text id="jz30am"
lower(User.email)
length(User.name)
year(Order.createdAt)
```

solo si pueden traducirse al Query Expression System.

---

# 155. Function registry

No deberá hardcodearse todo en EntityQuery.

Podrá utilizar:

```text id="awn5eq"
EntityQueryFunctionRegistry
```

que traduzca a funciones del Query Engine.

---

# 156. Function capability

Algunas funciones pueden requerir capabilities.

La incompatibilidad deberá aparecer antes o durante planificación/compilación apropiadamente.

---

# 157. Raw expressions

El escape hatch del documento 53 podrá integrarse.

Ejemplo:

```php id="x5lufd"
$query->whereRaw(
    RawEntityExpression::database(...)
);
```

solo con APIs claramente peligrosas/avanzadas.

---

# 158. Raw entity expression

Nunca deberá disfrazarse como field string normal.

---

# 159. Raw semantics

Cuando se use raw:

```text id="6tygd5"
portability may decrease
semantic analysis may be partial
optimization may be limited
security responsibility increases
```

---

# 160. Query hints

Podrán existir:

```text id="ic1vof"
EntityQueryHint
```

para comunicar intención a niveles inferiores.

---

# 161. Hints ≠ commands

Ejemplos:

```text id="bk7sxu"
read_only
prefer_batch_loading
expected_cardinality
consistency requirement
timeout
```

No:

```text id="c7ob1f"
USE INDEX xyz
```

como API ORM portable base.

---

# 162. Platform hints

Si se soportan hints vendor-specific deberán ir en un extension namespace explícito.

---

# 163. Query fingerprint

Se introduce:

```text id="drgphw"
EntityQueryFingerprint
```

---

# 164. Fingerprint purpose

Puede utilizarse para:

```text id="k9itoh"
debugging
translation cache
compiled query cache dependency
telemetry grouping
deduplication
```

---

# 165. Structural fingerprint

Debe distinguir:

```text id="f7u3qk"
query shape
```

de:

```text id="sz3v15"
parameter values
```

---

# 166. Example

Estas queries:

```php id="0khht7"
$query->where('id', 10);
$query->where('id', 20);
```

podrán compartir:

```text id="pzbdtu"
structural fingerprint
```

pero no:

```text id="n9qqhk"
parameter value fingerprint
```

---

# 167. Translation cache

Podrá cachearse:

```text id="4v9s29"
EntityQuery structure
→ Database Query Model template
```

si:

```text id="wk5fsa"
metadata fingerprint
mapping fingerprint
scope fingerprint
query structure fingerprint
```

coinciden.

---

# 168. No cache of live parameters in shared template

Shared cache no deberá contener request-sensitive mutable parameter values.

---

# 169. Query execution context

Cada ejecución deberá tener:

```text id="x24275"
EntityQueryExecutionContext
```

---

# 170. Execution context

Puede contener:

```text id="k4ypko"
deadline
cancellation token
consistency requirement
transaction context
database context
telemetry context
```

---

# 171. Timeout

Entity Query podrá ofrecer:

```php id="v929km"
$query->timeout(
    Duration::seconds(2)
);
```

traducido a Execution Engine.

---

# 172. Cancellation

La cancelación pertenecerá al sistema del documento 84.

Entity Query solo propaga el contexto.

---

# 173. Query retry

Entity Query no deberá realizar retries por sí sola.

Eso pertenece a:

```text id="rmqewq"
Execution Retry System
```

---

# 174. Read retries

Aunque las queries SELECT sean frecuentemente replayable, el policy system deberá decidir.

---

# 175. Persistent runtime

Shared immutable:

```text id="4cm55k"
EntityQueryFactory configuration
EntityQueryFunctionRegistry
EntityScopeRegistry
compiled translation descriptors
```

---

# 176. Operation-local

```text id="amnu3o"
EntityQuery object
parameters
predicate tree
loading graph
execution context
result cursor
```

---

# 177. FrankenPHP

Nunca:

```text id="u1pkgn"
static EntityQuery $currentQuery
```

---

# 178. OpenSwoole

Immutable query objects facilitan seguridad entre coroutines, pero contextos de ejecución siguen siendo aislados.

---

# 179. Multitenancy

Entity Query no hardcodeará:

```text id="h9jytg"
tenant_id
```

---

# 180. Tenant context

La query se crea dentro de:

```text id="ululje"
EntityManager
→ DatabaseContext
→ Tenant-aware identity namespace / routing
```

---

# 181. Tenant scope

Si la estrategia shared-table requiere predicate tenant:

```text id="7yw508"
TenantScope
```

puede contribuir una restricción semántica.

---

# 182. Tenant scope ≠ tenant connection resolution

Son conceptos diferentes.

---

# 183. Cross-tenant queries

Deberán ser:

```text id="m76fos"
explicit
privileged
specialized
```

si alguna vez se soportan.

No bypassar automáticamente TenantScope.

---

# 184. Read/write routing

EntityQuery normal declara:

```text id="7byepa"
READ
```

---

# 185. Locking queries

Si existe:

```php id="cfu63v"
$query->forUpdate();
```

la intención cambia a:

```text id="yhkab4"
locking read
```

y puede requerir:

```text id="k2vufq"
WRITE-capable/primary connection
active transaction
```

---

# 186. Pessimistic lock integration

Los detalles se definirán en:

```text id="6lc0ra"
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md
```

---

# 187. `forUpdate()` ≠ literal SQL

Debe producir:

```text id="sqp71d"
LockingIntent(PESSIMISTIC_WRITE)
```

---

# 188. Optimistic lock query

Puede cargar:

```text id="1vw12j"
version
```

como parte de result plan, pero optimistic locking principalmente afecta persistence.

---

# 189. Query consistency

EntityQuery podrá expresar:

```text id="7n8sk9"
EVENTUAL
READ_YOUR_WRITES
STRONG
PRIMARY_REQUIRED
```

según capacidades del futuro routing system.

---

# 190. Security

Entity Query debe garantizar parameter binding estructurado.

No concatenación de input.

---

# 191. Field names from untrusted input

Incluso con parameter binding, field names no son parámetros SQL.

Por tanto:

```php id="40issj"
$query->orderBy($request->input('sort'));
```

deberá validar que el field pertenece al EntityMetadata.

---

# 192. Sort allow-list

La aplicación puede usar:

```text id="fr847m"
EntityFieldReference
```

o validar el string contra metadata/policy.

---

# 193. Raw field injection

Nunca convertir un unknown field directamente en SQL identifier.

---

# 194. Authorization

Una query técnicamente válida no significa autorizada.

Authorization permanece separado.

---

# 195. Security-aware query integration

El Authorization System podrá aportar:

```text id="dlpr5w"
EntityAccessConstraint
```

mediante integración explícita, no mediante lógica escondida en el Query Builder.

---

# 196. Audit

Queries sensibles podrán emitir eventos/telemetry para:

```text id="c3b1fr"
actor
entity type
query category
scope
```

sin registrar parámetros sensibles de forma insegura.

---

# 197. Telemetry

Métricas:

```text id="jb26gj"
orm.entity_query.created
orm.entity_query.executed
orm.entity_query.translation.duration
orm.entity_query.validation.duration
orm.entity_query.result.count

orm.entity_query.managed_result
orm.entity_query.read_only_result
orm.entity_query.detached_result
orm.entity_query.projection_result
orm.entity_query.scalar_result

orm.entity_query.scope.count
orm.entity_query.relationship_path.depth
orm.entity_query.eager_load.count
orm.entity_query.cache.hit
orm.entity_query.cache.miss
```

---

# 198. Query tracing

```text id="d73b7p"
EntityQuery.execute
├── Apply Scopes
├── Normalize
├── Validate
├── Resolve Entity Paths
├── Translate
├── Database Query Pipeline
├── Execute
└── Process Result
```

---

# 199. Explain API

Podrá existir:

```php id="5lgcz0"
$explanation = $query->explain();
```

sin ejecutar la query necesariamente.

---

# 200. EntityQueryExplanation

Puede mostrar:

```text id="4289yl"
root entity
predicates
resolved fields
relationship traversals
applied scopes
loading intents
result mode
query fingerprint
translated Query Model
```

---

# 201. explain ≠ database EXPLAIN

Debe distinguirse:

```text id="4gud3t"
Entity Query Explain
```

de:

```text id="il6kwf"
Database Execution EXPLAIN
```

El primero explica traducción semántica.

El segundo explica el plan de ejecución DB.

---

# 202. Chained explain

Una herramienta avanzada podrá mostrar:

```text id="6hexym"
Entity Query
→ Query Model
→ Logical Plan
→ Physical Plan
→ SQL
→ DB EXPLAIN
```

pero manteniendo capas separadas.

---

# 203. Diagnostics

Ejemplo:

```text id="2ka2i4"
DB-ENTITY-QUERY-FIELD-002

Entity:
App\Entity\User

Path:
profile.countryCode

Problem:
Field "countryCode" does not exist on Profile.

Available:
country
countryName
locale
```

---

# 204. Relationship diagnostic

```text id="3lh4l4"
DB-ENTITY-QUERY-REL-005

Path:
orders.customer.email

Problem:
"customer" is not a relationship of Order.
```

---

# 205. Type diagnostic

```text id="3rp1dv"
DB-ENTITY-QUERY-TYPE-003

Field:
User.status

Expected:
UserStatus

Received:
string("active")

Policy:
Strict typed entity queries
```

---

# 206. Strictness

Podrá existir:

```text id="xzl9er"
STRICT
COERCIVE
COMPATIBILITY
```

para tipos de input.

---

# 207. Recommended default

Para el ORM Core:

```text id="fh5lcs"
STRICT
```

especialmente con:

```text id="wb2baq"
Enums
Value Objects
Typed IDs
```

---

# 208. Model API coercion

La capa Model API podría permitir una ergonomía más flexible, pero deberá normalizar antes de llegar a EntityQuery canónico.

---

# 209. Exception hierarchy

```text id="m3wzqj"
DatabaseOrmException
└── EntityQueryException
    ├── EntityQueryConstructionException
    ├── EntityQueryContextException
    ├── EntityQueryValidationException
    ├── UnknownEntityQueryTypeException
    ├── UnknownEntityFieldException
    ├── UnknownEntityFieldPathException
    ├── InvalidEntityFieldPathException
    ├── InvalidEntityQueryPredicateException
    ├── InvalidEntityQueryOperatorException
    ├── EntityQueryTypeMismatchException
    ├── EntityQueryRelationshipException
    ├── InvalidEntityRelationshipPathException
    ├── EntityQueryProjectionException
    ├── EntityQueryAggregationException
    ├── EntityQueryGroupingException
    ├── EntityQueryScopeException
    ├── EntityQueryTranslationException
    ├── EntityQueryExecutionException
    ├── EntityQueryNoResultException
    ├── EntityQueryNonUniqueResultException
    ├── EntityQueryResultModeException
    ├── EntityQueryExtensionException
    └── EntityQueryInvariantException
```

---

# 210. Query extension system

Se propone:

```text id="dcqg0u"
EntityQueryExtension
```

---

# 211. Extension examples

```text id="hl8dxw"
full-text entity queries
JSON entity functions
temporal entity queries
geographic entity queries
custom domain predicates
```

---

# 212. Extension contract

Una extensión deberá traducir semántica entity-aware hacia extensiones del Query Model.

No generar SQL directamente.

---

# 213. Extension registry

```text id="3gejvk"
EntityQueryExtensionRegistry
```

deberá ser:

```text id="5bdx5x"
frozen
deterministic
collision-aware
versioned
```

---

# 214. Extension collision

Dos extensiones reclamando la misma función/predicate incompatible deberán producir error.

---

# 215. No monkey patching

No modificar dinámicamente métodos de EntityQuery durante requests.

---

# 216. Testing strategy

El sistema requiere:

```text id="ystk79"
unit tests
semantic tests
translation tests
integration tests
hydration tests
scope tests
runtime isolation tests
cross-platform conformance tests
```

---

# 217. Query construction tests

Probar:

```text id="2v7rkb"
empty query
where
and/or
nested groups
order
limit
offset
distinct
```

---

# 218. Field resolution tests

```text id="97a4em"
simple field
renamed column
embedded field
relationship field
deep path
unknown field
```

---

# 219. Type tests

```text id="7wk879"
integer
string
bool
enum
UUID
ULID
DateTime
value object
JSON
null
invalid type
```

---

# 220. Relationship tests

```text id="wksvbs"
to-one traversal
to-many traversal
whereHas
whereDoesntHave
nested relations
many-to-many
polymorphic relation
```

---

# 221. Predicate tests

```text id="7t7rrk"
equal
not equal
greater
less
null
in
empty in
between
like
exists
and
or
not
```

---

# 222. Projection tests

```text id="u4zppb"
managed entity
read-only entity
detached entity
DTO
tuple
scalar
aggregate
partial entity policy
```

---

# 223. IdentityMap tests

Query de una entidad ya managed deberá reutilizar la instancia correcta.

---

# 224. Dirty state tests

Query repetida no deberá sobrescribir cambios locales no flushed.

---

# 225. Scope tests

```text id="83d6dr"
default scope
multiple scopes
scope order
disable scope
non-disableable scope
scope diagnostics
```

---

# 226. Eager loading tests

```text id="hl8234"
to-one
to-many
nested
multiple relationships
duplicate root rows
IdentityMap reconciliation
```

---

# 227. Translation tests

Una misma Entity Query semántica deberá producir un Query Model equivalente independientemente de API superficial usada.

---

# 228. Example

```php id="9sf5oa"
$query->where('status', $status);
```

y:

```php id="gl8kpv"
$query->where(
    EntityFields::user()->status(),
    EqualTo::value($status),
);
```

deberán converger cuando su semántica sea idéntica.

---

# 229. Cross-platform tests

La misma Entity Query portable deberá funcionar sobre:

```text id="ujpn7u"
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando sus requisitos estén soportados.

---

# 230. Persistent runtime tests

Verificar:

```text id="6htbox"
no predicate leakage
no parameter leakage
no tenant leakage
no scope mutation leakage
no query context leakage
```

entre requests.

---

# 231. Concurrency tests

Immutable query definitions podrán reutilizarse estructuralmente, pero execution contexts deberán permanecer aislados.

---

# 232. Performance tests

Medir:

```text id="mca4ce"
query construction
field resolution
relationship resolution
translation
fingerprinting
translation cache hit
result processing
```

---

# 233. Invariantes arquitectónicas

## DB-ENTITY-QUERY-001
Entity Query expresará consultas sobre EntityTypes.

## DB-ENTITY-QUERY-002
Entity Query será distinto de Repository.

## DB-ENTITY-QUERY-003
Entity Query será distinto de EntityManager.

## DB-ENTITY-QUERY-004
Entity Query será distinto de Database Query Builder.

## DB-ENTITY-QUERY-005
Entity Query será distinto de Query AST.

## DB-ENTITY-QUERY-006
Entity Query será distinto de SQL.

## DB-ENTITY-QUERY-007
Entity Query será distinto de Hydrator.

## DB-ENTITY-QUERY-008
Entity Query será distinto de Persistence Engine.

## DB-ENTITY-QUERY-009
Entity Query no generará SQL.

## DB-ENTITY-QUERY-010
Entity Query no ejecutará SQL directamente.

## DB-ENTITY-QUERY-011
Entity Query no abrirá Connection directamente.

## DB-ENTITY-QUERY-012
Entity Query no administrará UnitOfWork.

## DB-ENTITY-QUERY-013
Entity Query no implementará IdentityMap.

## DB-ENTITY-QUERY-014
Entity Query utilizará EntityMetadata.

## DB-ENTITY-QUERY-015
Entity Query utilizará EntityMapping.

## DB-ENTITY-QUERY-016
Entity Query root será EntityType, no table.

## DB-ENTITY-QUERY-017
Root Entity será distinto de SQL alias.

## DB-ENTITY-QUERY-018
Cada EntityQuery tendrá estado propio.

## DB-ENTITY-QUERY-019
No existirá static mutable query state.

## DB-ENTITY-QUERY-020
Query immutability será preferida.

## DB-ENTITY-QUERY-021
Fluent API no requerirá mutabilidad.

## DB-ENTITY-QUERY-022
EntityFieldReference será semántica ORM.

## DB-ENTITY-QUERY-023
Field path será distinto de column path.

## DB-ENTITY-QUERY-024
Unknown field no será tratado como SQL identifier.

## DB-ENTITY-QUERY-025
String field paths serán resueltos antes del Query Engine.

## DB-ENTITY-QUERY-026
Relationship paths serán validados.

## DB-ENTITY-QUERY-027
Embedded paths serán validados.

## DB-ENTITY-QUERY-028
Relationship traversal utilizará metadata.

## DB-ENTITY-QUERY-029
To-many traversal no será tratado como scalar join naïve.

## DB-ENTITY-QUERY-030
To-many predicate traversal tendrá semántica explícita.

## DB-ENTITY-QUERY-031
whereHas representará EXISTS semántico.

## DB-ENTITY-QUERY-032
whereDoesntHave representará NOT EXISTS semántico.

## DB-ENTITY-QUERY-033
Entity predicates serán tipados internamente.

## DB-ENTITY-QUERY-034
Comparison operators serán normalizados.

## DB-ENTITY-QUERY-035
NULL no será comparado mediante equality normal.

## DB-ENTITY-QUERY-036
Empty IN tendrá semántica canónica válida.

## DB-ENTITY-QUERY-037
Logical predicate precedence será explícita.

## DB-ENTITY-QUERY-038
Predicate grouping será representable.

## DB-ENTITY-QUERY-039
Valores estarán separados de la estructura query.

## DB-ENTITY-QUERY-040
Entity Query no interpolará parámetros.

## DB-ENTITY-QUERY-041
Type conversion utilizará Type System.

## DB-ENTITY-QUERY-042
Enum conversion utilizará mapping tipado.

## DB-ENTITY-QUERY-043
Value object conversion no requerirá conocimiento de columnas por el caller.

## DB-ENTITY-QUERY-044
Multi-column value objects podrán expandirse semánticamente.

## DB-ENTITY-QUERY-045
Entity expression layer no duplicará innecesariamente el Query Expression System.

## DB-ENTITY-QUERY-046
Ordering utilizará field semantics.

## DB-ENTITY-QUERY-047
NULL ordering será semántico y platform-aware downstream.

## DB-ENTITY-QUERY-048
Limit será validado.

## DB-ENTITY-QUERY-049
Offset será validado.

## DB-ENTITY-QUERY-050
Pagination avanzada se delegará al Pagination System.

## DB-ENTITY-QUERY-051
Default EntityQuery result será entity-aware.

## DB-ENTITY-QUERY-052
Managed entity result respetará IdentityMap.

## DB-ENTITY-QUERY-053
Projection result no entrará a IdentityMap.

## DB-ENTITY-QUERY-054
Scalar result no entrará a IdentityMap.

## DB-ENTITY-QUERY-055
Tuple result no entrará a IdentityMap.

## DB-ENTITY-QUERY-056
Projection será preferida sobre partial entity cuando sea viable.

## DB-ENTITY-QUERY-057
Partial entity será capability explícita.

## DB-ENTITY-QUERY-058
Partial entity no se fingirá fully-loaded.

## DB-ENTITY-QUERY-059
Aggregations reutilizarán Query Aggregation System.

## DB-ENTITY-QUERY-060
Grouping semantics serán validadas.

## DB-ENTITY-QUERY-061
Entity distinct será distinguible de raw row distinct cuando sea necesario.

## DB-ENTITY-QUERY-062
IdentityMap deduplication no cambiará automáticamente SQL semantics.

## DB-ENTITY-QUERY-063
Eager loading será loading intent.

## DB-ENTITY-QUERY-064
Eager loading no implicará necesariamente JOIN.

## DB-ENTITY-QUERY-065
Nested eager loading producirá load graph.

## DB-ENTITY-QUERY-066
Load graph será distinto de SQL join tree.

## DB-ENTITY-QUERY-067
Relationship loading strategy será downstream.

## DB-ENTITY-QUERY-068
Entity scopes serán tipados.

## DB-ENTITY-QUERY-069
Scopes no concatenarán SQL.

## DB-ENTITY-QUERY-070
Scope ordering será determinista.

## DB-ENTITY-QUERY-071
Scope application será inspectable.

## DB-ENTITY-QUERY-072
Scope disabling será explícito.

## DB-ENTITY-QUERY-073
Non-disableable scope deberá declararlo.

## DB-ENTITY-QUERY-074
Scope no sustituirá Authorization System.

## DB-ENTITY-QUERY-075
Model scopes convergerán a Entity Query semantics.

## DB-ENTITY-QUERY-076
Specifications producirán Entity Query semantics.

## DB-ENTITY-QUERY-077
Specifications no generarán SQL.

## DB-ENTITY-QUERY-078
EntityQueryExecutor será distinto de DatabaseQueryExecutor.

## DB-ENTITY-QUERY-079
EntityQueryTranslator producirá Query Model, no SQL.

## DB-ENTITY-QUERY-080
Translation result incluirá result semantics cuando sea necesario.

## DB-ENTITY-QUERY-081
EntityResultPlan será distinto del resultado runtime.

## DB-ENTITY-QUERY-082
Managed hydration reutilizará IdentityMap.

## DB-ENTITY-QUERY-083
Managed hydration no sobrescribirá dirty entity silenciosamente.

## DB-ENTITY-QUERY-084
Read-only ORM state será distinto de PHP object immutability.

## DB-ENTITY-QUERY-085
Detached results tendrán semántica explícita.

## DB-ENTITY-QUERY-086
Detached duplicate identity no será creado accidentalmente.

## DB-ENTITY-QUERY-087
`first()` podrá optimizar limit.

## DB-ENTITY-QUERY-088
`one()` validará cardinalidad lógica.

## DB-ENTITY-QUERY-089
`oneOrNull()` validará cardinalidad lógica.

## DB-ENTITY-QUERY-090
Logical entity count será distinto de physical row count.

## DB-ENTITY-QUERY-091
Root entity count tendrá semántica definida.

## DB-ENTITY-QUERY-092
Normalization será distinta de optimization.

## DB-ENTITY-QUERY-093
EntityQueryValidator operará antes de SQL compilation.

## DB-ENTITY-QUERY-094
Unknown EntityType fallará antes de ejecución.

## DB-ENTITY-QUERY-095
Type-incompatible predicate fallará antes de ejecución cuando sea detectable.

## DB-ENTITY-QUERY-096
whereKey reutilizará Identifier Metadata.

## DB-ENTITY-QUERY-097
whereKey soportará typed identifiers.

## DB-ENTITY-QUERY-098
whereKey soportará composite identifiers.

## DB-ENTITY-QUERY-099
Polymorphic query utilizará inheritance metadata.

## DB-ENTITY-QUERY-100
Entity Query no hardcodeará discriminator SQL.

## DB-ENTITY-QUERY-101
Embedded queries utilizarán Embedded Mapping.

## DB-ENTITY-QUERY-102
Query functions utilizarán un registry tipado.

## DB-ENTITY-QUERY-103
Function registry no generará SQL directamente.

## DB-ENTITY-QUERY-104
Raw escape hatch será explícito.

## DB-ENTITY-QUERY-105
Raw expression reducirá garantías y deberá ser visible.

## DB-ENTITY-QUERY-106
Query hints serán hints, no commands físicos.

## DB-ENTITY-QUERY-107
Vendor-specific hints estarán aislados.

## DB-ENTITY-QUERY-108
EntityQueryFingerprint será determinista.

## DB-ENTITY-QUERY-109
Structural fingerprint será separado de parameter values.

## DB-ENTITY-QUERY-110
Translation cache dependerá de metadata/mapping fingerprints.

## DB-ENTITY-QUERY-111
Shared translation cache no almacenará mutable request parameters.

## DB-ENTITY-QUERY-112
Execution context será operation-scoped.

## DB-ENTITY-QUERY-113
Timeout se propagará al Execution Engine.

## DB-ENTITY-QUERY-114
Cancellation se propagará al Cancellation System.

## DB-ENTITY-QUERY-115
Entity Query no implementará retry policy.

## DB-ENTITY-QUERY-116
Persistent workers no compartirán mutable EntityQuery state.

## DB-ENTITY-QUERY-117
Compiled query definitions podrán compartirse si son immutable.

## DB-ENTITY-QUERY-118
Current query no se almacenará en static state.

## DB-ENTITY-QUERY-119
Core Entity Query no hardcodeará tenant_id.

## DB-ENTITY-QUERY-120
Tenant constraints llegarán mediante contexto/scopes/integrations.

## DB-ENTITY-QUERY-121
Cross-tenant query será explícita.

## DB-ENTITY-QUERY-122
Entity Query declarará READ intent por default.

## DB-ENTITY-QUERY-123
Locking read declarará locking intent.

## DB-ENTITY-QUERY-124
forUpdate no generará literal SQL directamente.

## DB-ENTITY-QUERY-125
Physical read/write routing será downstream.

## DB-ENTITY-QUERY-126
Consistency requirements serán declarativas.

## DB-ENTITY-QUERY-127
Entity Query protegerá valores mediante binding.

## DB-ENTITY-QUERY-128
Untrusted field names serán validados contra metadata.

## DB-ENTITY-QUERY-129
Unknown sort field no llegará a SQL.

## DB-ENTITY-QUERY-130
Valid query no implicará authorization.

## DB-ENTITY-QUERY-131
Authorization integration será explícita.

## DB-ENTITY-QUERY-132
Sensitive query values podrán redactarse en telemetry.

## DB-ENTITY-QUERY-133
Entity Query telemetry no cambiará semántica.

## DB-ENTITY-QUERY-134
Entity Query Explain será distinto de DB EXPLAIN.

## DB-ENTITY-QUERY-135
Query explanation será posible sin ejecución cuando corresponda.

## DB-ENTITY-QUERY-136
Diagnostics preservarán entity paths.

## DB-ENTITY-QUERY-137
Strict type mode será soportado.

## DB-ENTITY-QUERY-138
Model API coercion no contaminará Entity Query canónico.

## DB-ENTITY-QUERY-139
EntityQuery extensions tendrán IDs estables.

## DB-ENTITY-QUERY-140
Extension registry será frozen.

## DB-ENTITY-QUERY-141
Extension conflicts producirán error.

## DB-ENTITY-QUERY-142
Extensions no usarán monkey patching runtime.

## DB-ENTITY-QUERY-143
Extensions traducirán a Query Model extensions.

## DB-ENTITY-QUERY-144
Extensions no generarán SQL directamente.

## DB-ENTITY-QUERY-145
Cross-platform portable Entity Queries conservarán la misma semántica lógica.

## DB-ENTITY-QUERY-146
MySQL/MariaDB/PostgreSQL/SQLite compilation permanecerá downstream.

## DB-ENTITY-QUERY-147
Repository utilizará Entity Query para consultas avanzadas.

## DB-ENTITY-QUERY-148
Model API utilizará la misma Entity Query semantics.

## DB-ENTITY-QUERY-149
Entity Query será la única capa ORM de consulta semántica sobre entidades.

## DB-ENTITY-QUERY-150
Toda Entity Query deberá convertirse al Query Model universal antes de cualquier compilación o ejecución física.

---

# 234. Anti-patterns

## 234.1 Entity Query genera SQL

Incorrecto:

```php id="vxv3ua"
final class EntityQuery
{
    public function toSql(): string
    {
        return "SELECT ...";
    }
}
```

como responsabilidad central.

Correcto:

```text id="y1m1y7"
EntityQuery
→ EntityQueryTranslator
→ QueryModel
→ SQLCompiler
```

---

# 235. Anti-pattern: column names in ORM API

Incorrecto:

```php id="p3fw53"
$query->where(
    'users.created_at',
    '>',
    $date,
);
```

Correcto:

```php id="c2d07m"
$query->where(
    'createdAt',
    '>',
    $date,
);
```

---

# 236. Anti-pattern: silent unknown field

Nunca:

```text id="n57nuq"
unknown ORM field
→ treat as raw column name
```

---

# 237. Anti-pattern: mutable shared query

Incorrecto:

```php id="8hd2rn"
final class Repository
{
    private EntityQuery $query;

    public function where(...)
    {
        $this->query->where(...);
    }
}
```

si el mismo estado persiste entre llamadas.

---

# 238. Anti-pattern: eager = join

Incorrecto:

```text id="exm6zx"
with('posts')
→ always LEFT JOIN posts
```

---

# 239. Anti-pattern: to-many direct join semantics

Incorrecto:

```text id="j9ac9w"
User.where('orders.status', PENDING)
→ naïve join
→ duplicate User results
```

sin considerar cardinalidad/EXISTS semantics.

---

# 240. Anti-pattern: overwrite dirty entity

Incorrecto:

```text id="17prq4"
managed User#10 dirty
    +
query User#10
    ↓
overwrite with DB row
```

---

# 241. Anti-pattern: partial entity as full entity

Incorrecto:

```text id="fmw789"
SELECT id,name
→ User fully loaded
```

sin unloaded-field state.

---

# 242. Anti-pattern: query-level authorization assumption

Incorrecto:

```text id="xzvdnv"
query works
⇒ actor authorized
```

---

# 243. Arquitectura de clases propuesta

```text id="azd0ru"
src/Quantum/Database/ORM/Query/
│
├── Contract/
│   ├── EntityQuery.php
│   ├── EntityQueryFactory.php
│   ├── EntityQueryExecutor.php
│   ├── EntityQueryTranslator.php
│   └── EntityQueryExtension.php
│
├── Query/
│   ├── DefaultEntityQuery.php
│   ├── EntityQueryContext.php
│   ├── EntityQueryOptions.php
│   └── EntityQueryHint.php
│
├── Reference/
│   ├── RootEntityReference.php
│   ├── EntityFieldReference.php
│   ├── EntityRelationshipReference.php
│   └── EntityPathReference.php
│
├── Path/
│   ├── FieldPath.php
│   ├── EntityPathResolver.php
│   ├── ResolvedEntityPath.php
│   └── EntityPathSegment.php
│
├── Predicate/
│   ├── EntityPredicate.php
│   ├── EntityComparisonPredicate.php
│   ├── EntityNullPredicate.php
│   ├── EntityInPredicate.php
│   ├── EntityBetweenPredicate.php
│   ├── EntityExistsPredicate.php
│   ├── EntityRelationshipExistsPredicate.php
│   ├── EntityAndPredicate.php
│   ├── EntityOrPredicate.php
│   ├── EntityNotPredicate.php
│   └── EntityPredicateBuilder.php
│
├── Expression/
│   ├── EntityExpression.php
│   ├── EntityFieldExpression.php
│   ├── EntityAggregateExpression.php
│   ├── EntityFunctionExpression.php
│   └── EntityExpressionTranslator.php
│
├── Parameter/
│   ├── TypedEntityParameter.php
│   └── EntityParameterNormalizer.php
│
├── Order/
│   ├── EntityOrder.php
│   ├── EntityOrderDirection.php
│   └── NullOrdering.php
│
├── Projection/
│   ├── EntitySelection.php
│   ├── EntityProjection.php
│   ├── DtoProjection.php
│   ├── ScalarProjection.php
│   └── TupleProjection.php
│
├── Aggregation/
│   ├── EntityAggregation.php
│   ├── EntityGrouping.php
│   └── EntityHaving.php
│
├── Relationship/
│   ├── RelationshipLoadingIntent.php
│   ├── RelationshipLoadGraph.php
│   └── RelationshipLoadNode.php
│
├── Scope/
│   ├── EntityQueryScope.php
│   ├── EntityScopeRegistry.php
│   ├── EntityScopePipeline.php
│   └── EntityScopeDescriptor.php
│
├── Result/
│   ├── EntityQueryResultMode.php
│   ├── EntityResultPlan.php
│   ├── ManagedEntityResultPlan.php
│   ├── ReadOnlyEntityResultPlan.php
│   ├── DetachedEntityResultPlan.php
│   ├── ProjectionResultPlan.php
│   ├── ScalarResultPlan.php
│   └── TupleResultPlan.php
│
├── Translation/
│   ├── DefaultEntityQueryTranslator.php
│   ├── EntityQueryTranslationContext.php
│   └── EntityQueryTranslationResult.php
│
├── Normalize/
│   └── EntityQueryNormalizer.php
│
├── Validation/
│   ├── EntityQueryValidator.php
│   ├── EntityQueryValidationReport.php
│   └── EntityQueryValidationIssue.php
│
├── Execution/
│   ├── DefaultEntityQueryExecutor.php
│   └── EntityQueryExecutionContext.php
│
├── Fingerprint/
│   └── EntityQueryFingerprint.php
│
├── Cache/
│   └── EntityQueryTranslationCache.php
│
├── Function/
│   ├── EntityQueryFunction.php
│   └── EntityQueryFunctionRegistry.php
│
├── Extension/
│   ├── EntityQueryExtensionRegistry.php
│   └── EntityQueryExtensionDescriptor.php
│
├── Telemetry/
│   ├── EntityQueryTelemetry.php
│   └── EntityQueryExplanation.php
│
└── Exception/
    └── ...
```

---

# 244. Dependencias permitidas

```text id="ax3e5p"
Entity Query
    ↓
Entity Metadata
Entity Mapping
Relationship Metadata
Type System
Query Model contracts
Query Expression contracts
Runtime DatabaseContext contracts
```

---

# 245. Dependencias prohibidas

Entity Query Core no deberá depender directamente de:

```text id="p10nlk"
PDO
MySQL compiler
PostgreSQL compiler
MariaDB compiler
SQLite compiler
Schema migration execution
HTTP Request
current authenticated user
mutable global tenant
```

---

# 246. API objetivo

Repository:

```php id="5s2f24"
$users = $entityManager
    ->repository(User::class);

$result = $users
    ->query()
    ->where(
        'status',
        UserStatus::ACTIVE
    )
    ->whereHas(
        'orders',
        fn ($orders) =>
            $orders->where(
                'status',
                OrderStatus::PENDING
            )
    )
    ->with(
        'profile',
        'roles'
    )
    ->orderBy(
        'createdAt',
        'DESC'
    )
    ->limit(100)
    ->get();
```

---

# 247. Model API objetivo

```php id="pz0115"
$result = User::query()
    ->where(
        'status',
        UserStatus::ACTIVE
    )
    ->with('profile')
    ->get();
```

Ambas APIs convergen a:

```text id="vwh69m"
EntityQuery<User>
```

---

# 248. Typed API futura

Para máxima seguridad estática podrá generarse:

```php id="a0pg2v"
User::query()
    ->where(
        UserFields::status(),
        EqualTo::value(
            UserStatus::ACTIVE
        )
    );
```

---

# 249. Coexistencia

VoltStack podrá ofrecer:

```text id="qs4sb2"
string field paths
```

para productividad y:

```text id="66se3v"
generated typed field references
```

para proyectos estrictos.

Ambas deberán producir exactamente la misma semántica interna.

---

# 250. Flujo completo

```text id="6xqqm1"
Application
    ↓
Repository / Model
    ↓
EntityQuery
    ↓
Apply Entity Scopes
    ↓
Normalize Entity Query
    ↓
Validate Entity Semantics
    ↓
Resolve Fields / Relationships
    ↓
Convert Domain Values
    ↓
Build Entity Translation Model
    ↓
EntityQueryTranslator
    ↓
Database Query Model / AST
    ↓
Database Semantic Query Engine
    ↓
Database Query Optimizer
    ↓
Database Query Planner
    ↓
SQL Compiler
    ↓
Execution Engine
    ↓
Database Result
    ↓
EntityResultPlan
    ├── Managed Hydration
    ├── ReadOnly Hydration
    ├── Detached Hydration
    ├── DTO Projection
    ├── Tuple
    └── Scalar
```

---

# 251. Fórmula de Entity Query

```text id="yhtp8y"
EntityQuery
=
RootEntity
+
EntitySelections
+
EntityPredicates
+
RelationshipPaths
+
Ordering
+
Grouping
+
LoadingIntents
+
ResultMode
+
ExecutionIntent
```

---

# 252. Fórmula de traducción

```text id="uk2bc5"
DatabaseQueryModel
=
Translate(
    Validate(
        Resolve(
            Normalize(
                ApplyScopes(
                    EntityQuery
                )
            )
        )
    )
)
```

---

# 253. Fórmula de field resolution

```text id="hro22x"
Resolve(EntityPath)
=
EntityMetadata
+
RelationshipMetadata
+
EmbeddedMapping
+
FieldMetadata
→
Canonical ORM Reference
```

---

# 254. Fórmula de query segura

```text id="pkrjjh"
SafeEntityQuery
=
KnownEntityType
∧
ValidEntityPaths
∧
TypeCompatibleValues
∧
BoundParameters
∧
ValidatedScopes
∧
NoRawIdentifierFallback
```

---

# 255. Fórmula de managed result

```text id="g3e6ox"
ManagedEntityResult(row)
=
ResolveIdentity(row)
→ EntityKey
→ IdentityMap
→ ReuseOrHydrate
→ UnitOfWorkRegistration
```

---

# 256. Fórmula de to-many predicate

```text id="6mt46w"
Predicate(
    Root.toMany.field = value
)
≈
EXISTS(
    related entity satisfying field = value
)
```

cuando esa sea la semántica canónica del traversal.

---

# 257. Fórmula de proyección

```text id="qm10au"
ProjectionQuery
=
EntitySemanticSelection
→ QueryModel
→ Result
→ ProjectionHydrator
```

sin:

```text id="p7ghow"
IdentityMap
UnitOfWork
```

---

# 258. Fórmula de runtime seguro

```text id="v7boio"
SafeEntityQueryRuntime
=
ImmutableSharedRegistries
+
OperationLocalQueryState
+
OperationLocalParameters
+
ScopedDatabaseContext
+
NoStaticMutableQuery
+
DeterministicTranslation
```

---

# 259. Master Formula

```text id="55la73"
Database Entity Query System
=
Typed Entity Query Model
+
EntityQuery Factory
+
Root Entity Semantics
+
Entity Field References
+
Relationship Path Resolution
+
Embedded Path Resolution
+
Typed Predicates
+
Domain Value Conversion
+
Ordering
+
Grouping
+
Aggregation
+
Entity Projections
+
Result Modes
+
Managed Hydration Intent
+
Read-Only/Detached Modes
+
Relationship Loading Intent
+
Entity Scope Pipeline
+
Specification Integration
+
Entity Query Normalization
+
Entity Semantic Validation
+
Entity Query Translation
+
Database Query Model Generation
+
Translation Fingerprints
+
Translation Cache
+
Execution Context Propagation
+
Locking/Consistency Intent
+
Extension Governance
+
Multitenancy Integration
+
Persistent Runtime Isolation
+
Diagnostics
+
Explainability
+
Telemetry
```

---

# 260. Master Rule

> **VoltStack Entity Query permite consultar el modelo de dominio sin exponer el modelo físico de la base de datos. Resuelve entidades, campos, relaciones, tipos y proyecciones hacia el Query Model estructurado del Database System, pero la optimización, planificación, compilación SQL y ejecución permanecen en sus capas correspondientes.**

---

# 261. Resultado arquitectónico

Después de este documento la separación queda:

```text id="ybwe5p"
Developer Query
      ↓
Entity Query
      ↓
ORM Semantic Translation
      ↓
Database Query Model
      ↓
Database Semantic Engine
      ↓
Optimizer
      ↓
Planner
      ↓
SQL Compiler
      ↓
Executor
```

Esto permite que VoltStack ofrezca:

```php id="dzmwsh"
User::query()
    ->where(
        'profile.country',
        CountryCode::MX
    )
    ->with('roles')
    ->get();
```

sin que:

```text id="2nr3tu"
User
Model
Repository
EntityQuery
```

necesiten conocer:

```text id="y98oum"
users table
profile_id
role join table
SQL aliases
prepared statement syntax
vendor dialect
```

---

# 262. Estado del bloque ORM

```text id="hmipsn"
112_DATABASE_ORM_ARCHITECTURE.md
        ↓
113_DATABASE_ENTITY_MODEL.md
        ↓
114_DATABASE_MODEL_API_SYSTEM.md
        ↓
115_DATABASE_ENTITY_METADATA_SYSTEM.md
        ↓
116_DATABASE_ENTITY_MAPPING_SYSTEM.md
        ↓
117_DATABASE_ATTRIBUTE_MAPPING_SYSTEM.md
        ↓
118_DATABASE_ENTITY_MANAGER_SYSTEM.md
        ↓
119_DATABASE_REPOSITORY_SYSTEM.md
        ↓
120_DATABASE_ENTITY_QUERY_SYSTEM.md
```

Ya se encuentran definidos:

```text id="d03awb"
ORM Core boundaries
Entity model
Model API
Metadata
Mapping
Attributes
EntityManager
Repositories
Entity-aware querying
```

Queda ahora formalizar cómo el ORM representa el estado runtime de cada entidad.

---

# 263. Siguiente documento

```text id="se606f"
121_DATABASE_ENTITY_STATE_SYSTEM.md
```

Este documento deberá definir con precisión:

```text id="h5uv8e"
EntityState
EntityStateMachine
NEW
MANAGED
CLEAN
DIRTY
REMOVED
DETACHED
READ_ONLY
UNKNOWN

object identity vs entity identity
registration
attachment/detachment
state transitions
dirty state relationship
generated identity transitions
remove cancellation
flush reconciliation
rollback effects
refresh effects
unknown persistence outcome
state inspection
state storage
runtime scope isolation
```

y deberá separar claramente:

```text id="sy8skd"
Entity Domain State
≠
Entity Identity State
≠
ORM Persistence State
≠
ChangeSet
≠
Snapshot
```

La regla central será:

> **Entity State describe la relación runtime entre una instancia de entidad y su Persistence Context; no describe el estado de negocio de la entidad y no debe almacenarse como una bandera intrusiva dentro del objeto de dominio.**