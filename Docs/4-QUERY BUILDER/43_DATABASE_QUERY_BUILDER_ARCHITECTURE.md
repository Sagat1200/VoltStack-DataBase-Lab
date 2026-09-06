# 43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md

# VoltStack Quantum Database
## Query Builder Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 43 — Query Builder Architecture  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Public Construction API  
**Versión:** 1.0

---

# 1. Propósito

El **Query Builder System** define la API programática utilizada para construir consultas de base de datos dentro de VoltStack.

Su objetivo principal es ofrecer una experiencia de desarrollo:

```text
simple
fluent
expressive
strongly structured
type-aware
IDE-friendly
extensible
safe
```

similar en ergonomía a Laravel, pero sustentada internamente por la arquitectura semántica de VoltStack.

Ejemplo:

```php
DB::table('users')
    ->where('active', true)
    ->where('age', '>=', 18)
    ->orderBy('name')
    ->limit(100)
    ->get();
```

Esta API no construirá directamente:

```sql
SELECT * FROM users ...
```

En su lugar:

```text
Developer API
      │
      ▼
Query Builder
      │
      ▼
Query Model
      │
      ▼
Query AST
```

El SQL aparecerá únicamente mucho después:

```text
AST
 │
 ▼
Semantic Engine
 │
 ▼
Optimizer
 │
 ▼
Planner
 │
 ▼
SQL Compiler
```

---

# 2. Regla maestra

La regla arquitectónica fundamental será:

> Query Builder construye consultas; SQL Compiler genera SQL.

Por tanto:

```text
QueryBuilder
≠
SqlBuilder
```

y:

```text
Query Builder
    MUST NOT
        generate SQL
        concatenate SQL
        quote identifiers
        allocate SQL placeholders
        bind PDO parameters
        inspect PDO
        execute native statements
```

---

# 3. Posición arquitectónica

```text
Application
    │
    ▼
Public Database API
    │
    ▼
Query Builder
    │
    ▼
Query Model
    │
    ▼
Query AST
    │
    ▼
Normalization
    │
    ▼
Validation
    │
    ▼
Semantic Analysis
    │
    ▼
SemanticQueryArtifact
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ▼
Compiler
    │
    ▼
Executor
```

---

# 4. Responsabilidad

Query Builder será responsable de:

```text
query construction
fluent developer API
identifier construction
expression construction
predicate construction
parameter declaration
projection construction
relation construction
join construction
aggregation construction
ordering construction
pagination construction
CTE construction
set operation construction
DML construction
query metadata declaration
escape-hatch construction
extension invocation
```

---

# 5. No responsabilidades

Query Builder no será responsable de:

```text
SQL generation
SQL dialect selection
schema introspection
symbol resolution
column resolution
query type inference
relation semantic resolution
join optimization
query optimization
execution planning
connection selection
connection acquisition
transaction execution
parameter binding
result hydration
ORM persistence
telemetry execution
retry handling
```

---

# 6. Arquitectura conceptual

```text
                     Query Builder
                          │
       ┌──────────────────┼──────────────────┐
       │                  │                  │
       ▼                  ▼                  ▼
 Expression API      Predicate API      Relation API
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                    Query Model
                          │
                          ▼
                       Query AST
```

---

# 7. Public API vs internal representation

Debe existir una separación clara entre:

```text
Public Builder API
```

y:

```text
Internal Query Representation
```

Ejemplo:

```php
$query
    ->where('status', 'active')
    ->where('age', '>=', 18);
```

podría producir conceptualmente:

```text
AndPredicate
├── ComparisonPredicate
│   ├── ColumnReference(status)
│   ├── EQUAL
│   └── ParameterExpression(P1)
└── ComparisonPredicate
    ├── ColumnReference(age)
    ├── GREATER_THAN_OR_EQUAL
    └── ParameterExpression(P2)
```

con:

```text
BindingSet
├── P1 → "active"
└── P2 → 18
```

---

# 8. Builder ≠ AST

El Builder no será el AST.

```text
QueryBuilder
≠
QueryAstNode
```

El Builder puede ser:

```text
ergonomic
incrementally mutable
developer-oriented
temporary
```

mientras el AST será:

```text
immutable
canonicalizable
analyzable
cacheable
framework-oriented
```

---

# 9. Builder mutability

VoltStack permitirá que el Builder sea mutable durante construcción.

Ejemplo:

```php
$query = DB::table('users');

$query->where('active', true);

$query->orderBy('name');
```

Esto resulta conveniente para aplicaciones dinámicas.

Pero esa mutabilidad estará limitada a:

```text
construction phase
```

---

# 10. Freeze boundary

Antes de entrar al Query Engine:

```text
Mutable Builder
      │
      ▼
freeze()
      │
      ▼
Immutable Query Artifact
```

---

# 11. QueryBuilderState

La mutabilidad deberá encapsularse.

Conceptualmente:

```php
final class QueryBuilderState
{
    // operation-local construction state
}
```

No deberá filtrarse a las fases posteriores.

---

# 12. Builder lifecycle

```text
CREATED
   │
   ▼
BUILDING
   │
   ▼
FINALIZING
   │
   ▼
FROZEN
   │
   ▼
SUBMITTED
```

Opcionalmente:

```text
DISCARDED
```

---

# 13. Builder reutilizable

La API deberá definir claramente qué significa reutilizar un Builder.

Ejemplo:

```php
$base = DB::table('users')
    ->where('active', true);

$admins = $base->copy()
    ->where('role', 'admin');

$customers = $base->copy()
    ->where('role', 'customer');
```

---

# 14. No accidental shared mutation

Debe evitarse:

```php
$base = DB::table('users');

$a = $base;
$b = $base;

$a->where('type', 'A');

// accidentalmente modifica $b
```

La API deberá documentar claramente las reglas de aliasing.

---

# 15. Estrategia recomendada

V1 podrá utilizar:

```text
mutable fluent builder
+
explicit clone/copy/fork
+
immutable final Query Model
```

Esto conserva ergonomía sin contaminar el Query Engine.

---

# 16. QueryBuilderFactory

La creación deberá centralizarse mediante:

```text
QueryBuilderFactory
```

Conceptualmente:

```php
$builder = $factory->table('users');
```

---

# 17. DB facade

La fachada podrá ofrecer:

```php
DB::table('users');
DB::query();
DB::select();
```

pero será únicamente una fachada de developer experience.

No contendrá estado global mutable de la query.

---

# 18. Facade resolution

Conceptualmente:

```text
DB Facade
   │
   ▼
DatabaseManager / Public Database API
   │
   ▼
QueryBuilderFactory
   │
   ▼
SelectQueryBuilder
```

---

# 19. Query builder families

La arquitectura soportará builders especializados:

```text
SelectQueryBuilder
InsertQueryBuilder
UpdateQueryBuilder
DeleteQueryBuilder
```

y builders/componentes auxiliares:

```text
ExpressionBuilder
PredicateBuilder
JoinBuilder
SubqueryBuilder
CteBuilder
AggregationBuilder
WindowBuilder
SetOperationBuilder
```

---

# 20. No MegaQueryBuilder

Evitar:

```php
class QueryBuilder
{
    public function select() {}
    public function insert() {}
    public function update() {}
    public function delete() {}
    public function migration() {}
    public function execute() {}
    public function hydrate() {}
    public function transaction() {}
}
```

---

# 21. Common query builder abstraction

Podrá existir una base interna:

```text
AbstractQueryBuilder
```

o composición mediante:

```text
BuilderComponents
```

Preferencia:

```text
composition
>
deep inheritance
```

---

# 22. Shared capabilities

Los builders pueden compartir servicios como:

```text
IdentifierFactory
ExpressionFactory
PredicateFactory
ParameterFactory
QueryMetadataBuilder
QueryModelFactory
```

---

# 23. SelectQueryBuilder

Ejemplo:

```php
DB::table('users')
    ->select('id', 'name', 'email')
    ->where('active', true)
    ->orderBy('name')
    ->limit(50);
```

produce:

```text
SelectQueryModel
```

---

# 24. InsertQueryBuilder

Ejemplo:

```php
DB::table('users')->insert([
    'name' => 'John',
    'email' => 'john@example.com',
]);
```

produce conceptualmente:

```text
InsertQueryModel
```

más bindings.

---

# 25. UpdateQueryBuilder

```php
DB::table('users')
    ->where('id', $id)
    ->update([
        'name' => 'John',
    ]);
```

produce:

```text
UpdateQueryModel
```

---

# 26. DeleteQueryBuilder

```php
DB::table('users')
    ->where('id', $id)
    ->delete();
```

produce:

```text
DeleteQueryModel
```

---

# 27. Construction vs execution

Debe existir conceptualmente una diferencia entre:

```text
build query
```

y:

```text
execute query
```

Aunque la API pública permita:

```php
$query->get();
```

internamente:

```text
get()
 │
 ▼
freeze builder
 │
 ▼
Query Artifact
 │
 ▼
Query Engine
 │
 ▼
Executor
```

---

# 28. Terminal operations

Métodos como:

```text
get()
first()
exists()
count()
insert()
update()
delete()
value()
pluck()
cursor()
```

pueden actuar como:

```text
terminal operations
```

---

# 29. Terminal operation boundary

Un terminal method no ejecutará SQL directamente.

Ejemplo:

```php
public function get(): Result
{
    $command = $this->finalizeQuery();

    return $this->queryGateway->execute($command);
}
```

Conceptualmente.

---

# 30. QueryGateway

Podrá existir un puerto:

```text
QueryGateway
```

entre Builder API y Query Engine.

---

# 31. QueryGateway responsibility

```text
Builder
   │
   ▼
QueryGateway
   │
   ▼
Query Processing Pipeline
```

El Builder no conocerá:

```text
Compiler
Executor
ConnectionManager
Driver
```

directamente.

---

# 32. Command/query separation

Podrán existir operaciones:

```text
ReadQueryRequest
WriteQueryRequest
```

o un artifact genérico:

```text
DatabaseQueryRequest
```

que encapsule:

```text
Query Artifact
BindingSet
Execution Metadata
```

---

# 33. Query + values separation

Regla:

```text
Query Structure
≠
Runtime Values
```

Por tanto:

```text
Query AST
+
BindingSet
```

serán artifacts separados.

---

# 34. Example

API:

```php
DB::table('users')
    ->where('email', $email)
    ->first();
```

estructura:

```text
ColumnReference(email)
EQUAL
ParameterExpression(P1)
```

runtime binding:

```text
P1 → $email
```

---

# 35. No literal interpolation

Nunca:

```text
email = '$email'
```

dentro del SQL generado por Builder.

---

# 36. Automatic parameterization

Por defecto:

```php
->where('age', '>', 18)
```

deberá crear:

```text
ParameterExpression
```

no:

```text
LiteralExpression(18)
```

cuando el valor proviene de runtime/application input.

---

# 37. Literal API

Cuando un literal semántico sea realmente necesario deberá ser explícito.

Por ejemplo:

```php
$query->where(
    DB::expr()->column('version'),
    '>',
    DB::expr()->literal(0)
);
```

La API final podrá variar.

---

# 38. Parameter identity

Builder asignará:

```text
ParameterId
```

pero no:

```text
SQL placeholder
```

---

# 39. Example

Builder:

```text
ParameterId P1
```

Compiler PostgreSQL podría producir:

```text
$1
```

Compiler PDO/MySQL:

```text
?
```

Otro binding profile:

```text
:vs_1
```

---

# 40. Identifier API

Identifiers deberán ser estructurados.

Ejemplo:

```php
->select('users.id')
```

deberá convertirse en:

```text
QualifiedColumnReference
├── qualifier: users
└── name: id
```

no almacenarse únicamente como SQL raw.

---

# 41. IdentifierFactory

Podrá existir:

```text
IdentifierFactory
```

para transformar public shorthand en identifiers estructurados.

---

# 42. Identifier parsing

Entrada:

```text
users.email
```

podrá convertirse en:

```text
QualifiedIdentifier
```

en el boundary del Builder.

---

# 43. Identifier validation

Builder puede realizar validación sintáctica básica.

Pero no debe comprobar:

```text
does users.email exist?
```

Eso corresponde a:

```text
Schema-Aware Semantic Resolution
```

---

# 44. Wildcard

```php
->select('*')
```

produce un wildcard AST explícito.

No deberá expandirse consultando el schema dentro del Builder.

---

# 45. Qualified wildcard

```php
->select('users.*')
```

produce:

```text
QualifiedWildcardExpression
```

La expansión real ocurrirá durante Semantic Analysis.

---

# 46. Expression API

VoltStack ofrecerá una API estructurada para expresiones.

Ejemplos conceptuales:

```php
DB::expr()->column('price');
DB::expr()->parameter($value);
DB::expr()->literal(1);
DB::expr()->function('lower', ...);
DB::expr()->add(...);
DB::expr()->case(...);
```

---

# 47. Expression builder output

Siempre:

```text
ExpressionNode
```

o un intermediate expression model convertible de forma determinista al AST.

---

# 48. Predicate API

Predicates deberán usar el sistema definido en:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Ejemplo:

```php
->where('age', '>=', 18)
```

produce:

```text
ComparisonPredicate
```

---

# 49. where shorthand

La API podrá aceptar:

```php
where($column, $value)
where($column, $operator, $value)
```

---

# 50. Operator normalization

Entrada:

```php
'>='
```

se convertirá a:

```text
ComparisonOperator::GREATER_THAN_OR_EQUAL
```

en el Builder boundary.

---

# 51. Unknown operator

Un operador desconocido:

```php
->where('age', 'banana', 18)
```

deberá fallar.

Nunca será concatenado como SQL.

---

# 52. Null shorthand

API:

```php
->where('deleted_at', null)
```

podrá normalizarse semánticamente a:

```text
NullPredicate
```

si esa es la semántica documentada de la API.

---

# 53. Explicit null API

También:

```php
->whereNull('deleted_at');
->whereNotNull('deleted_at');
```

---

# 54. Boolean groups

Ejemplo:

```php
$query->where(function ($q) {
    $q->where('role', 'admin')
      ->orWhere('role', 'manager');
});
```

La closure sólo será mecanismo de construcción.

---

# 55. Closure lifecycle

La closure:

```text
exists only while building
```

Después:

```text
Closure
   │
   ▼
Predicate Group
   │
   ▼
AST
```

No aparecerá en Query Artifact.

---

# 56. No closure serialization

Nunca cachear:

```text
builder closure
```

---

# 57. Nested predicate builder

La closure recibirá un builder limitado:

```text
PredicateGroupBuilder
```

preferentemente, no necesariamente el Query Builder completo.

---

# 58. Least privilege builders

Ejemplo:

```text
where closure
→ PredicateGroupBuilder

join closure
→ JoinConditionBuilder

CTE closure
→ QueryBuilderFactory / SubqueryBuilder
```

Esto reduce APIs inválidas por contexto.

---

# 59. AND / OR

El Builder conservará intención lógica explícita:

```text
AND
OR
NOT
```

---

# 60. Three-valued logic

El Builder no aplicará simplificaciones booleanas que puedan romper SQL three-valued logic.

---

# 61. IN

```php
->whereIn('id', $ids)
```

produce:

```text
InPredicate
+
CollectionParameter
```

---

# 62. No collection expansion

Builder no convertirá:

```text
[1,2,3]
```

en:

```text
?,?,?
```

---

# 63. CollectionParameter

Preferido:

```text
ParameterId P1
shape = COLLECTION
```

con binding:

```text
P1 → [1,2,3]
```

---

# 64. Empty collections

La semántica de:

```php
whereIn('id', [])
```

deberá ser definida por el Predicate System/normalization policy.

El Builder no generará SQL inválido.

---

# 65. BETWEEN

```php
->whereBetween('age', [18, 65])
```

produce:

```text
BetweenPredicate
```

---

# 66. EXISTS

```php
->whereExists(function ($q) {
    $q->from('orders')
      ->whereColumn('orders.user_id', 'users.id');
});
```

produce una subquery estructurada.

---

# 67. Correlation

El Builder no determina formalmente si:

```text
orders.user_id
```

está correlacionado con una relación exterior.

Eso pertenece a Semantic Analysis.

---

# 68. whereColumn

Debe distinguir:

```php
->where('id', $value)
```

de:

```php
->whereColumn('orders.user_id', 'users.id')
```

---

# 69. Value vs identifier

Esta separación evita interpretar ambiguamente strings.

```text
where('a', 'b')
```

significa:

```text
a = parameter("b")
```

no:

```text
a = column b
```

---

# 70. Column comparison API

Para comparar columnas se usará:

```text
whereColumn()
```

o expression API explícita.

---

# 71. Select projection

```php
->select('id', 'name')
```

produce una lista ordenada de:

```text
ProjectionItem
```

---

# 72. addSelect

Podrá existir:

```php
->addSelect('email');
```

---

# 73. Projection aliases

Ejemplo:

```php
->select([
    'user_email' => 'users.email',
]);
```

podrá producir:

```text
AliasedProjection
├── expression: users.email
└── alias: user_email
```

---

# 74. Alias ≠ SQL text

El alias será un identifier estructurado.

---

# 75. From

```php
DB::table('users')
```

produce:

```text
TableRelationReference(users)
```

---

# 76. Relation alias

```php
DB::table('users as u')
```

podrá ser shorthand.

Internamente:

```text
RelationReference
├── relation: users
└── alias: u
```

---

# 77. Prefer explicit internal representation

Aunque la API acepte:

```text
users as u
```

la representación interna nunca dependerá de volver a parsear ese string.

---

# 78. Explicit alias API

También podrá existir:

```php
DB::table('users')->as('u');
```

según diseño final de DX.

---

# 79. Joins

```php
->join('orders', 'orders.user_id', '=', 'users.id')
```

produce:

```text
JoinModel
├── kind: INNER
├── relation: orders
└── predicate
```

---

# 80. Join callback

```php
->join('orders', function ($join) {
    $join->on('orders.user_id', '=', 'users.id')
         ->where('orders.active', true);
});
```

---

# 81. JoinConditionBuilder

Debe distinguir:

```text
on(column, operator, column)
```

de:

```text
where(column, operator, runtimeValue)
```

---

# 82. Join semantics

Builder sólo construye el join declarado.

No determina:

```text
one-to-many
many-to-one
unique join
foreign key relationship
key preservation
null rejection
```

Eso pertenece al Semantic Engine.

---

# 83. Join order

El Builder conserva el orden declarado.

El Optimizer podrá modificarlo posteriormente cuando sea semánticamente seguro.

---

# 84. Subqueries

VoltStack soportará:

```text
subquery as relation
subquery as expression
subquery in predicate
scalar subquery
EXISTS subquery
IN subquery
```

---

# 85. Subquery builder

Un subquery deberá finalizarse en:

```text
Query Model / Query AST
```

antes de incorporarse a la consulta padre.

---

# 86. Builder ownership

Evitar guardar un mutable child builder dentro del parent.

Incorrecto:

```text
ParentBuilder
└── mutable ChildBuilder
```

Preferido:

```text
ChildBuilder
   │
   ▼
freeze
   │
   ▼
SubqueryArtifact
   │
   ▼
ParentBuilder
```

---

# 87. CTE

Ejemplo:

```php
$query->with('active_users', function ($q) {
    $q->from('users')
      ->where('active', true);
});
```

produce:

```text
CteDefinition
```

---

# 88. Recursive CTE

Podrá existir una API explícita:

```php
->withRecursive(...)
```

La validación semántica de recursión ocurrirá posteriormente.

---

# 89. Set operations

```php
$queryA->union($queryB);
```

produce:

```text
SetOperation
```

---

# 90. Set operation compatibility

Builder no necesita verificar schema/types completos.

Semantic Analysis comprobará:

```text
same output arity
compatible types
nullability
semantic compatibility
```

---

# 91. Aggregates

Podrán existir:

```php
->count()
->sum('total')
->avg('score')
->min('price')
->max('price')
```

como terminal helpers o expression helpers.

---

# 92. Aggregate semantic identity

`sum()` deberá producir una función/aggregate semántico.

No SQL:

```text
SUM(...)
```

directamente.

---

# 93. Grouping

```php
->groupBy('user_id');
```

produce:

```text
GroupingClause
```

---

# 94. HAVING

```php
->having('total', '>', 100);
```

deberá producir Predicate AST dentro del contexto HAVING.

---

# 95. Group validation

Builder no determinará si una projection es válida respecto a GROUP BY.

Eso pertenece al Semantic Engine.

---

# 96. Window functions

La API soportará posteriormente:

```text
WindowDefinition
WindowExpression
PartitionBy
WindowOrder
WindowFrame
```

sin construir sintaxis SQL directamente.

---

# 97. ORDER BY

```php
->orderBy('name', 'asc');
```

produce:

```text
OrderByItem
├── expression
└── direction: ASC
```

---

# 98. Order direction enum

Entrada string podrá normalizarse a:

```text
OrderDirection::ASC
OrderDirection::DESC
```

---

# 99. Invalid direction

```php
->orderBy('name', 'banana');
```

deberá fallar en el boundary.

---

# 100. NULL ordering

Podrá soportarse semánticamente:

```text
NULLS_FIRST
NULLS_LAST
PLATFORM_DEFAULT
```

sin insertar directamente:

```sql
NULLS FIRST
```

---

# 101. LIMIT

```php
->limit(10);
```

produce:

```text
LimitClause(10)
```

---

# 102. OFFSET

```php
->offset(20);
```

produce:

```text
OffsetClause(20)
```

---

# 103. Pagination

Pagination de alto nivel puede usar Builder, pero pertenecerá principalmente a:

```text
199_DATABASE_PAGINATION_SYSTEM.md
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 104. DISTINCT

```php
->distinct();
```

produce:

```text
DistinctSpecification
```

---

# 105. Locking

Podrá existir:

```php
->forUpdate();
->sharedLock();
```

pero internamente deberá producir:

```text
LockRequirement / LockClause Model
```

no SQL vendor-specific.

---

# 106. Platform-specific locking

Opciones como:

```text
NOWAIT
SKIP LOCKED
```

deberán ser capabilities semánticas.

---

# 107. Query metadata

Builder podrá declarar metadata:

```php
$query
    ->label('users.active')
    ->usePrimary()
    ->timeout(...)
    ->consistency(...)
    ->sensitive();
```

---

# 108. Metadata ownership

Estos métodos producirán:

```text
QueryMetadata
```

según:

```text
31_DATABASE_QUERY_METADATA_SYSTEM.md
```

---

# 109. Metadata ≠ runtime state

Builder no almacenará:

```text
Connection
Transaction
Deadline timer
Active Span
Tenant object
Current User
```

como QueryMetadata.

---

# 110. Query context

Builder tampoco será propietario de:

```text
QueryContext
```

del documento 32.

QueryContext aparece cuando comienza el procesamiento formal.

---

# 111. Query finalization

Conceptualmente:

```text
Builder State
    │
    ▼
Builder Finalizer
    │
    ├── Query Model
    ├── Parameter Definitions
    ├── BindingSet
    └── Query Metadata
```

---

# 112. QueryBuildArtifact

Podrá existir internamente:

```php
final readonly class QueryBuildArtifact
{
    public function __construct(
        public QueryModel $query,
        public ParameterDefinitionSet $parameters,
        public BindingSet $bindings,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 113. AST creation

Podrán existir dos estrategias:

```text
Builder
→ QueryModel
→ AST
```

o:

```text
Builder
→ AST-backed QueryModel
```

La decisión concreta deberá conservar:

```text
Builder ≠ AST
```

y:

```text
SQL generation absent
```

---

# 114. Recommended V1

Preferencia:

```text
Fluent Builder
      │
      ▼
Immutable Query Model
      │
      ▼
Query AST
```

porque mantiene una frontera clara entre:

```text
Developer construction model
```

y:

```text
compiler-oriented semantic structure
```

---

# 115. QueryModelFactory

El Builder podrá delegar finalización a:

```text
QueryModelFactory
```

---

# 116. QueryAstFactory

La conversión posterior:

```text
Query Model
→ Query AST
```

podrá utilizar:

```text
QueryAstFactory
```

o un:

```text
QueryModelLowerer
```

---

# 117. Lowering

El término:

```text
lowering
```

describe:

```text
higher-level query representation
→ lower-level canonical query structure
```

sin implicar SQL.

---

# 118. Builder shorthand lowering

Ejemplo:

```php
whereNull('deleted_at')
```

puede bajar directamente a:

```text
NullPredicate
```

---

# 119. Developer ergonomics

La API debe permitir el caso simple:

```php
$users = DB::table('users')
    ->where('active', true)
    ->get();
```

sin obligar al desarrollador a conocer:

```text
AST
SemanticGraph
QueryPlan
Dialect
Compiler
Driver
```

---

# 120. Progressive disclosure

Usuarios avanzados podrán acceder a:

```text
expressions
predicates
metadata
query artifacts
explain
raw expressions
extensions
```

sin complicar la API básica.

---

# 121. QueryBuilder contracts

El public contract deberá ser pequeño.

Evitar publicar toda la arquitectura interna como contratos estables.

---

# 122. Public API stability

El Builder será una de las APIs más sensibles a backward compatibility.

Por tanto:

```text
Public Builder API
```

deberá evolucionar cuidadosamente.

---

# 123. Internal builder components

Podrán permanecer:

```text
@internal
```

para permitir evolución.

---

# 124. Builder extensions

El Builder será extensible, pero mediante extension points controlados.

---

# 125. Extension examples

Una extensión podría añadir:

```php
$query->whereJsonContains(...);
$query->search(...);
$query->geoDistance(...);
```

si registra el comportamiento semántico necesario.

---

# 126. No macro-only semantics

Un método personalizado no podrá simplemente generar SQL arbitrario si pretende participar plenamente en:

```text
semantic analysis
optimization
portability
capability analysis
fingerprinting
```

---

# 127. Extension pipeline

Idealmente:

```text
Builder Extension
      │
      ▼
Extension Query Node
      │
      ▼
Semantic Extension
      │
      ▼
Optimizer Support
      │
      ▼
Planner Support
      │
      ▼
Compiler Support
```

---

# 128. Extension completeness

Una extensión deberá declarar qué fases soporta.

---

# 129. Extension capability

Ejemplo:

```text
Builder only
```

no será suficiente para una extensión portable completa.

---

# 130. Raw escape hatch

VoltStack permitirá:

```text
RawExpression
RawPredicate
RawOrder
RawQuery
```

cuando sea necesario.

---

# 131. Raw must be explicit

Nunca interpretar silenciosamente un string normal como raw SQL.

---

# 132. Example

Incorrecto:

```php
->where("age > 18")
```

si la API espera una columna.

Preferido:

```php
->whereRaw(...)
```

como escape hatch explícito.

---

# 133. Raw parameterization

Incluso Raw SQL deberá permitir bindings seguros:

```php
->whereRaw(
    'score > ?',
    [$score]
);
```

La API concreta podrá evolucionar.

---

# 134. Raw SQL policy

Raw constructs serán tratados según:

```text
RawSqlTrust
SecurityPolicy
PortabilityPolicy
```

---

# 135. Raw barrier

Raw constructs podrán actuar como:

```text
semantic barrier
optimizer barrier
portability barrier
```

cuando el framework no pueda comprender su contenido.

---

# 136. Security

Query Builder deberá ser seguro por defecto.

---

# 137. Value parameterization

Todos los valores runtime deberán parametrizarse automáticamente salvo APIs explícitas de literal/raw.

---

# 138. Identifier validation

Identifiers deberán pasar por construcción/validación estructurada.

---

# 139. Operator validation

Operators deberán normalizarse a enums/descriptors.

---

# 140. Direction validation

Order directions deberán validarse.

---

# 141. No dynamic SQL interpolation

Nunca:

```php
$query->whereRaw("email = '$email'");
```

como API recomendada.

---

# 142. Security boundaries

El Builder protege principalmente:

```text
construction boundary
```

El Compiler/Binding System/Driver protegerán otras fronteras.

---

# 143. Builder does not sanitize

El Builder no deberá depender de:

```text
string sanitization
```

para prevenir SQL injection.

La seguridad provendrá de:

```text
structured AST
+
parameterization
+
validated structural inputs
+
compiler escaping
```

---

# 144. Query safety policies

Podrán existir políticas que limiten:

```text
maximum joins
maximum predicates
maximum parameters
raw SQL usage
full-table update/delete
```

---

# 145. Full-table mutation protection

VoltStack podrá ofrecer protección contra:

```php
DB::table('users')->delete();
```

sin WHERE.

Pero la policy concreta pertenecerá al sistema de seguridad/safety.

---

# 146. Builder warning

El Builder puede marcar intención:

```text
FullTableMutation
```

pero no deberá convertirse en un God Security Manager.

---

# 147. Type awareness

Builder podrá aceptar hints explícitos:

```php
->where('id', '=', DB::param($id, UserIdType::class))
```

o equivalente.

---

# 148. Type inference ownership

Sin embargo, el Builder no será el Query Type Inference Engine.

---

# 149. Schema independence

Debe poder construirse una query:

```php
DB::table('users')
    ->where('id', $id);
```

sin tener conexión activa ni schema cargado.

---

# 150. Offline construction

Esto permitirá:

```text
tests
static analysis
query caching
offline compilation
code generation
persistent runtime prewarming
```

---

# 151. Builder and Schema

Builder podrá recibir explicit schema metadata únicamente mediante APIs especializadas futuras, pero nunca hará hidden schema I/O.

---

# 152. No DB calls

Durante:

```php
DB::table('users')
    ->where('active', true);
```

no deberá ocurrir:

```text
network I/O
connection acquisition
schema introspection
SQL execution
```

---

# 153. Lazy execution

Sólo terminal operations podrán iniciar el processing/execution pipeline.

---

# 154. Query introspection

Podrá existir:

```php
$query->toQueryArtifact();
```

o equivalente para tooling/testing.

---

# 155. No `toSql()` as core builder responsibility

Una API de developer convenience:

```php
$query->toSql();
```

podría existir eventualmente.

Pero internamente deberá ejecutar:

```text
Builder
→ Query Artifact
→ Semantic Processing
→ Planner
→ Compiler
```

para un target conocido.

Nunca:

```text
Builder::generateSql()
```

---

# 156. explain()

Podrá existir:

```php
$query->explain();
```

pero será un gateway al sistema de explain/planning.

---

# 157. dump()

Podría ofrecerse:

```php
$query->dump();
```

para inspeccionar estructura.

Debe distinguir:

```text
Builder dump
AST dump
Semantic explain
SQL explain
Execution explain
```

---

# 158. Builder fingerprint

Podrá existir un:

```text
QueryBuildFingerprint
```

para debugging.

Pero el fingerprint canónico de query deberá calcularse sobre artifacts normalizados.

---

# 159. Builder state not cache key

Nunca usar directamente:

```text
serialized QueryBuilder
```

como compiled-query cache key.

---

# 160. Source mapping

El Builder podrá registrar:

```text
SourceLocation
```

opcionalmente para diagnostics.

---

# 161. Source mapping cost

En producción podrá desactivarse o reducirse para minimizar overhead.

---

# 162. Query origin

Builder podrá declarar:

```text
QueryOrigin::APPLICATION
```

o permitir que ORM/Repository cambien el origin.

---

# 163. ORM-generated query

Ejemplo:

```text
Repository
   │
   ▼
Query Builder / Query Model API
   │
   ▼
Query AST
```

---

# 164. Same engine

Una query generada por ORM deberá usar el mismo:

```text
Expression System
Predicate System
Parameter System
Semantic Engine
Optimizer
Planner
Compiler
Executor
```

que una query construida con:

```php
DB::table(...)
```

---

# 165. No ORM Query Builder duplicate

No deberá existir:

```text
SQL Query Builder
+
ORM Query Builder
```

como engines completamente separados.

---

# 166. ORM facade

El ORM podrá ofrecer una API más rica:

```php
User::query()
```

pero deberá bajar al mismo Query Model/AST.

---

# 167. Active Record example

```php
User::query()
    ->where('active', true)
    ->get();
```

podrá producir el mismo core query representation que:

```php
DB::table('users')
    ->where('active', true)
    ->get();
```

con metadata ORM adicional fuera del core query semantics.

---

# 168. Query Builder scopes

El ORM podrá implementar:

```text
global scopes
local scopes
tenant scopes
soft-delete scopes
authorization filters
```

como transformaciones estructuradas.

---

# 169. Scope visibility

Estas transformaciones deberán ser visibles en:

```text
AST
metadata/provenance
semantic diagnostics
```

cuando corresponda.

---

# 170. No invisible SQL injection by framework

VoltStack no deberá añadir condiciones directamente en SQL después del Semantic Engine.

---

# 171. Policy injection

Preferido:

```text
Query Builder / Query Artifact
       │
       ▼
Policy Transformation
       │
       ▼
Visible AST Predicate
       │
       ▼
Semantic Analysis
```

---

# 172. Tenant integration

Multitenancy podrá producir:

```text
tenant predicate
schema requirement
connection requirement
partition requirement
```

mediante integration adapters.

No habrá:

```php
Tenant::current()
```

dentro del Builder core.

---

# 173. Persistent runtime

Builders serán:

```text
operation-local
```

y nunca:

```text
application singleton
worker singleton
static current builder
```

---

# 174. Request leakage

Nunca:

```text
Request A
→ Builder State A

Request B
→ accidentally inherits State A
```

---

# 175. Shared services

Podrán compartirse:

```text
QueryBuilderFactory
IdentifierFactory
ExpressionFactory
PredicateFactory
Frozen extension registries
Immutable descriptors
```

si son stateless/immutable.

---

# 176. Builder state

Siempre local:

```text
projections
predicates
joins
bindings
metadata under construction
ordering
grouping
limits
CTEs
set operations
```

---

# 177. Coroutine safety

Dos coroutines:

```text
Coroutine A
→ Builder A

Coroutine B
→ Builder B
```

no compartirán estado mutable.

---

# 178. No static parameter counter

Nunca:

```php
static $parameterIndex++;
```

globalmente.

---

# 179. Parameter allocator

Cada construcción tendrá su propio:

```text
ParameterIdAllocator
```

---

# 180. Deterministic parameter IDs

Cuando sea útil para fingerprints/tests:

```text
P1
P2
P3
```

deberán asignarse determinísticamente según estructura.

---

# 181. Copying builders

Al copiar un Builder deberá copiarse correctamente:

```text
builder state
parameter registry
metadata builder state
```

sin compartir mutabilidad peligrosa.

---

# 182. Copy-on-write

Una implementación futura podrá utilizar:

```text
copy-on-write
```

si benchmarks demuestran beneficio.

No es requisito V1.

---

# 183. Memory management

Builders no deberán retener:

```text
results
connections
statements
large schema snapshots
EntityManager
request objects
```

---

# 184. Large parameter collections

Un Builder puede recibir colecciones grandes.

No deberá expandirlas en miles de AST nodes.

---

# 185. Collection binding

Preferir:

```text
CollectionParameter
```

y dejar strategy selection al Planner/Binding Planner.

---

# 186. Generator values

Si el usuario proporciona:

```text
Generator
Traversable
```

el Parameter System determinará materialización/replayability.

Builder no deberá consumir silenciosamente generators varias veces.

---

# 187. Query builder diagnostics

Errores posibles:

```text
InvalidBuilderStateException
InvalidIdentifierException
InvalidOperatorException
InvalidOrderDirectionException
InvalidJoinDefinitionException
InvalidProjectionException
InvalidSetOperationDefinitionException
InvalidBuilderExtensionException
BuilderAlreadyFinalizedException
BuilderReuseException
```

---

# 188. Error quality

Mensajes deberán indicar:

```text
operation
invalid input
expected forms
source location when available
```

---

# 189. No sensitive values in errors

Ejemplo incorrecto:

```text
Invalid parameter with value:
super-secret-token
```

---

# 190. Builder diagnostics code

Ejemplos:

```text
DB_QUERY_BUILDER_INVALID_IDENTIFIER
DB_QUERY_BUILDER_INVALID_OPERATOR
DB_QUERY_BUILDER_INVALID_DIRECTION
DB_QUERY_BUILDER_INVALID_STATE
DB_QUERY_BUILDER_FINALIZED
DB_QUERY_BUILDER_EXTENSION_CONFLICT
```

---

# 191. Testing

El Builder deberá ser testeable sin base de datos.

---

# 192. Structural tests

Ejemplo:

```php
$query = DB::table('users')
    ->where('active', true)
    ->orderBy('name');

$artifact = $query->toQueryArtifact();
```

Se podrá verificar:

```text
relation users
predicate active = P1
binding P1 = true
order name ASC
```

---

# 193. No SQL assertion required

Los tests del Builder no deberán depender principalmente de:

```text
assertSame(
    'SELECT * FROM users WHERE active = ?',
    ...
)
```

Eso pertenece al Compiler.

---

# 194. Builder test categories

```text
construction
projection
predicate
parameter
join
subquery
CTE
set operation
aggregation
window
metadata
raw
extensions
copy/fork
finalization
persistent runtime
concurrency
security
architecture
```

---

# 195. Architecture tests

Query Builder no podrá importar:

```text
PDO
NativeConnection
ConnectionLease
ConnectionPool
Driver
Dialect SQL rendering internals
QueryExecutor implementation
EntityManager
UnitOfWork
IdentityMap
HTTP Request
AuthenticatedUser
Tenant Entity
OpenTelemetry Span
```

---

# 196. Dependency direction

```text
Public API
   │
   ▼
Query Builder
   │
   ├── Identifier Model
   ├── Expression System
   ├── Predicate System
   ├── Parameter System
   ├── Query Metadata
   └── Query Model
            │
            ▼
         Query AST
```

---

# 197. Forbidden direction

```text
Query Builder
    X
    ▼
SQL Compiler
```

como dependencia directa.

---

# 198. Forbidden direction

```text
Query Builder
    X
    ▼
Driver
```

---

# 199. Forbidden direction

```text
Query Builder
    X
    ▼
Connection
```

---

# 200. Execution bridge

El único puente de alto nivel será algo equivalente a:

```text
QueryExecutionGateway
```

---

# 201. QueryExecutionGateway

Conceptualmente:

```php
interface QueryExecutionGateway
{
    public function execute(
        QueryBuildArtifact $query
    ): QueryResult;
}
```

La firma real dependerá de las capas posteriores.

---

# 202. Command segregation

Puede ser preferible separar:

```text
QuerySubmissionGateway
```

de:

```text
QueryExecutionEngine
```

para que Builder dependa de una abstracción mínima.

---

# 203. Public terminal operations

Ejemplo conceptual:

```text
get()
    ↓
submit SELECT

insert()
    ↓
submit INSERT

update()
    ↓
submit UPDATE

delete()
    ↓
submit DELETE
```

---

# 204. Return types

El Builder no deberá asumir que todo retorna:

```text
array
```

Los tipos finales serán definidos por Result System.

---

# 205. Raw results vs ORM results

```text
DB::table(...)
```

podrá retornar Query Result/rows.

```text
User::query(...)
```

podrá pasar posteriormente por ORM Hydration.

El Query Engine base sigue siendo el mismo.

---

# 206. Public query object

Podrá considerarse una separación:

```text
QueryBuilder
```

para construcción y:

```text
PreparedQueryDefinition
```

para reutilización.

---

# 207. Query templates

Ejemplo futuro:

```php
$query = DB::table('users')
    ->where('email', DB::parameter('email'))
    ->prepareDefinition();
```

podría producir un template reusable.

---

# 208. Template reuse

```text
QueryTemplate
+
BindingSet A

QueryTemplate
+
BindingSet B
```

sin reconstruir estructura.

---

# 209. QueryTemplate ≠ native prepared statement

Crítico:

```text
Framework QueryTemplate
≠
PDOStatement
```

---

# 210. Dynamic queries

Builder deberá soportar:

```php
$query = DB::table('users');

if ($activeOnly) {
    $query->where('active', true);
}

if ($role !== null) {
    $query->where('role', $role);
}
```

sin comprometer estructura interna.

---

# 211. Conditional helpers

Podrán existir:

```php
->when(...)
->unless(...)
```

como DX helpers.

---

# 212. Conditional helper semantics

Estas funciones ejecutan lógica PHP durante construcción.

No se convierten en AST nodes.

---

# 213. Example

```php
$query->when(
    $role !== null,
    fn ($q) => $q->where('role', $role)
);
```

Después de build:

```text
when()
```

ya no existe.

Sólo existe el Predicate resultante.

---

# 214. Builder plugins

Los plugins no deberán obtener acceso irrestricto al estado interno.

Preferir:

```text
typed extension contracts
```

---

# 215. BuilderExtensionRegistry

Podrá existir:

```text
BuilderExtensionRegistry
```

con:

```text
register
validate
freeze
```

en bootstrap.

---

# 216. Extension method dispatch

Debe evitarse un sistema excesivamente mágico basado únicamente en:

```php
__call()
```

---

# 217. IDE friendliness

Preferir:

```text
typed extension interfaces
generated IDE metadata
explicit extension descriptors
```

cuando sea posible.

---

# 218. Macro support

Si se añade un sistema tipo macros, deberá diferenciar:

```text
DX Macro
```

de:

```text
Semantic Query Extension
```

---

# 219. DX Macro

Puede expandirse a APIs core existentes.

Ejemplo:

```text
whereActive()
→ where('active', true)
```

No requiere nuevo nodo semántico.

---

# 220. Semantic Extension

Ejemplo:

```text
vectorSimilarity()
```

puede requerir:

```text
new expression
new type semantics
capability requirement
planner support
compiler support
```

---

# 221. Macro safety

Macros deberán expandirse completamente antes de finalización.

---

# 222. Builder compilation

No deberá existir conceptualmente:

```text
BuilderCompiler
→ SQL
```

Pero sí podrá existir:

```text
BuilderFinalizer
```

o:

```text
QueryModelAssembler
```

---

# 223. BuilderFinalizer

Responsabilidades:

```text
validate builder construction state
freeze child artifacts
freeze parameter definitions
freeze metadata
produce immutable Query Model
produce BindingSet
```

---

# 224. BuilderFinalizer non-responsibilities

No:

```text
schema validation
symbol resolution
type inference
optimization
planning
SQL generation
execution
```

---

# 225. Builder structural validation

Puede detectar:

```text
SELECT with malformed projection structure
JOIN missing relation
BETWEEN missing bound
invalid operator string
invalid direction
duplicate builder-only aliases where structurally forbidden
```

---

# 226. Semantic validation later

No debe detectar mediante schema:

```text
unknown table
unknown column
ambiguous column
type mismatch
invalid foreign key relationship
```

---

# 227. Query metadata builder

Durante construcción podrá existir:

```text
MutableQueryMetadataBuilder
```

pero su salida será:

```text
Immutable QueryMetadata
```

---

# 228. Parameter registry

Cada builder root tendrá:

```text
QueryParameterRegistry
```

operation-local.

---

# 229. Nested builder parameter scopes

Subqueries tendrán:

```text
ParameterScope
```

propio o explícitamente relacionado.

---

# 230. Parameter collisions

Dos subqueries no deberán colisionar accidentalmente porque ambas utilicen:

```text
P1
```

internamente.

---

# 231. Scoped IDs

Conceptualmente:

```text
Q1:P1
Q1:S1:P1
Q1:S2:P1
```

o identidad equivalente.

---

# 232. Physical placeholders later

Compiler resolverá esos IDs a:

```text
$1
$2
$3
```

o equivalente.

---

# 233. Query builder snapshot

Para debugging podrá existir:

```text
QueryBuilderSnapshot
```

---

# 234. Snapshot content

```text
query kind
relations
projections
predicates
joins
grouping
ordering
parameter definitions
metadata summary
```

---

# 235. Snapshot excludes

```text
credentials
connection
transaction
native statement
unredacted sensitive values
```

---

# 236. Builder explain stages

Developer tooling podrá mostrar:

```text
BUILD
NORMALIZED
VALIDATED
SEMANTIC
OPTIMIZED
PLANNED
COMPILED
```

---

# 237. Example

```text
Query Stage: BUILD

FROM:
  users

WHERE:
  active = Parameter(P1)

ORDER:
  name ASC

BINDINGS:
  P1 = [boolean]
```

---

# 238. Developer API principles

La API pública seguirá:

```text
common things easy
advanced things possible
unsafe things explicit
platform-specific things visible
semantic intent preserved
```

---

# 239. Laravel-like ergonomics

VoltStack buscará conservar:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

---

# 240. Doctrine-like separation

Internamente mantendrá:

```text
Builder
≠
AST
≠
Semantic Engine
≠
Planner
≠
Compiler
≠
Executor
≠
Connection
```

---

# 241. VoltStack-specific advancement

VoltStack añadirá como arquitectura propia:

```text
Fluent DX
+
Structured Query Model
+
Semantic AST
+
Semantic Analysis
+
Semantic Graph
+
Optimizer
+
Planner
+
Capability System
+
Persistent Runtime Safety
```

---

# 242. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Builder\
```

---

# 243. Proposed directory structure

```text
Query/
└── Builder/
    ├── Contract/
    │   ├── QueryBuilderInterface.php
    │   ├── QueryBuilderFactoryInterface.php
    │   ├── QueryFinalizerInterface.php
    │   └── QuerySubmissionGatewayInterface.php
    │
    ├── Core/
    │   ├── QueryBuilderFactory.php
    │   ├── QueryBuilderState.php
    │   ├── QueryBuilderSnapshot.php
    │   └── QueryFinalizer.php
    │
    ├── Select/
    │   ├── SelectQueryBuilder.php
    │   ├── ProjectionBuilder.php
    │   └── SelectBuilderState.php
    │
    ├── Insert/
    │   ├── InsertQueryBuilder.php
    │   └── InsertBuilderState.php
    │
    ├── Update/
    │   ├── UpdateQueryBuilder.php
    │   └── UpdateBuilderState.php
    │
    ├── Delete/
    │   ├── DeleteQueryBuilder.php
    │   └── DeleteBuilderState.php
    │
    ├── Expression/
    │   ├── ExpressionBuilder.php
    │   └── ExpressionFactory.php
    │
    ├── Predicate/
    │   ├── PredicateBuilder.php
    │   ├── PredicateGroupBuilder.php
    │   └── PredicateFactory.php
    │
    ├── Relation/
    │   ├── RelationBuilder.php
    │   └── RelationFactory.php
    │
    ├── Join/
    │   ├── JoinBuilder.php
    │   ├── JoinConditionBuilder.php
    │   └── JoinBuilderState.php
    │
    ├── Subquery/
    │   ├── SubqueryBuilder.php
    │   └── SubqueryFinalizer.php
    │
    ├── Cte/
    │   ├── CteBuilder.php
    │   └── CteDefinitionBuilder.php
    │
    ├── SetOperation/
    │   └── SetOperationBuilder.php
    │
    ├── Aggregate/
    │   └── AggregateBuilder.php
    │
    ├── Window/
    │   └── WindowBuilder.php
    │
    ├── Identifier/
    │   ├── IdentifierFactory.php
    │   └── IdentifierParser.php
    │
    ├── Parameter/
    │   ├── QueryParameterRegistry.php
    │   ├── ParameterIdAllocator.php
    │   └── ParameterScope.php
    │
    ├── Metadata/
    │   └── QueryMetadataBuilder.php
    │
    ├── Raw/
    │   └── RawQueryBuilder.php
    │
    ├── Extension/
    │   ├── BuilderExtensionRegistry.php
    │   ├── BuilderExtensionDescriptor.php
    │   ├── BuilderMacro.php
    │   └── SemanticBuilderExtension.php
    │
    ├── Diagnostic/
    │   ├── QueryBuilderDiagnostic.php
    │   └── QueryBuilderDiagnosticCode.php
    │
    └── Exception/
        ├── QueryBuilderException.php
        ├── InvalidBuilderStateException.php
        ├── InvalidIdentifierException.php
        ├── InvalidOperatorException.php
        ├── InvalidOrderDirectionException.php
        ├── InvalidJoinDefinitionException.php
        ├── BuilderAlreadyFinalizedException.php
        └── BuilderExtensionConflictException.php
```

---

# 244. Builder ownership matrix

| Información | Owner |
|---|---|
| Fluent construction state | Query Builder |
| Runtime values during construction | BindingSet Builder |
| Parameter identity | Parameter System |
| Query structure | Query Model / AST |
| Expressions | Expression System |
| Predicates | Predicate System |
| Query metadata | Query Metadata System |
| Normalization | Query Normalization System |
| Structural validation | Query Validation System |
| Symbol resolution | Semantic Engine |
| Schema resolution | Semantic Engine |
| Type inference | Query Type Inference |
| Relation semantics | Relation Resolution |
| Constraints | Constraint Analysis |
| Semantic graph | Semantic Graph System |
| Optimization | Query Optimizer |
| Logical strategy | Query Planner |
| Physical strategy | Query Planner |
| SQL | SQL Compiler |
| Runtime binding | Binding System |
| Execution | Executor |
| Physical connection | Connection System |
| Native protocol | Driver |

---

# 245. Architectural invariants

## DB-QB-001

Query Builder no generará SQL.

## DB-QB-002

Query Builder no concatenará SQL.

## DB-QB-003

Query Builder no realizará SQL quoting.

## DB-QB-004

Query Builder no asignará SQL placeholders.

## DB-QB-005

Query Builder no realizará native parameter binding.

## DB-QB-006

Query Builder no dependerá de PDO.

## DB-QB-007

Query Builder no dependerá de Driver.

## DB-QB-008

Query Builder no dependerá de Connection.

## DB-QB-009

Query Builder no adquirirá conexiones.

## DB-QB-010

Query Builder no ejecutará statements directamente.

## DB-QB-011

Query Builder producirá Query Model/AST estructurado.

## DB-QB-012

Query Builder será diferente de Query AST.

## DB-QB-013

Query Builder podrá ser mutable únicamente durante construcción.

## DB-QB-014

Query Model final será immutable.

## DB-QB-015

Builder state nunca escapará al Query Engine.

## DB-QB-016

Runtime values serán separados de Query Structure.

## DB-QB-017

Runtime values serán parametrizados por defecto.

## DB-QB-018

Identifiers serán estructurados.

## DB-QB-019

Identifiers no serán tratados como runtime parameters.

## DB-QB-020

Operators serán estructurados.

## DB-QB-021

Order directions serán estructuradas.

## DB-QB-022

SQL keywords no serán runtime parameters.

## DB-QB-023

Column comparisons serán distintas de value comparisons.

## DB-QB-024

Builder closures desaparecerán después de construcción.

## DB-QB-025

Builder closures nunca serán serializadas.

## DB-QB-026

Builder closures nunca serán cacheadas.

## DB-QB-027

Collection values no serán expandidas a SQL placeholders por Builder.

## DB-QB-028

Collection parameters conservarán su shape.

## DB-QB-029

Wildcard no será expandido mediante schema dentro del Builder.

## DB-QB-030

Builder no realizará schema introspection.

## DB-QB-031

Builder no resolverá symbols.

## DB-QB-032

Builder no inferirá Query Types globalmente.

## DB-QB-033

Builder no resolverá relations semánticamente.

## DB-QB-034

Builder no inferirá foreign-key relationships.

## DB-QB-035

Builder no optimizará joins.

## DB-QB-036

Builder no seleccionará indexes.

## DB-QB-037

Builder no realizará query planning.

## DB-QB-038

Builder no conocerá SQL Dialect.

## DB-QB-039

Builder no tendrá vendor conditionals.

## DB-QB-040

Builder podrá funcionar offline.

## DB-QB-041

Construir una query no producirá network I/O.

## DB-QB-042

Construir una query no abrirá una conexión.

## DB-QB-043

Terminal operations serán bridges al Query Engine.

## DB-QB-044

Terminal operations no contendrán SQL execution logic.

## DB-QB-045

Query Builder deberá ser usable sin ORM.

## DB-QB-046

ORM deberá reutilizar el mismo Query Engine.

## DB-QB-047

Active Record no tendrá un segundo Query Engine.

## DB-QB-048

Repository API no tendrá un segundo Query Engine.

## DB-QB-049

Builder extensions serán controladas mediante registry.

## DB-QB-050

Extension registries serán frozen después de bootstrap.

## DB-QB-051

Macros serán distintas de semantic extensions.

## DB-QB-052

Raw SQL será explícito.

## DB-QB-053

Raw SQL nunca será inferido de strings normales.

## DB-QB-054

Raw constructs podrán usar parameter bindings.

## DB-QB-055

Raw constructs podrán ser semantic barriers.

## DB-QB-056

Builder será seguro por defecto.

## DB-QB-057

Builder no dependerá de sanitización de strings para seguridad.

## DB-QB-058

Parameterization será la estrategia predeterminada para valores.

## DB-QB-059

Builder no guardará credentials.

## DB-QB-060

Builder no guardará Transaction.

## DB-QB-061

Builder no guardará ConnectionLease.

## DB-QB-062

Builder no guardará EntityManager.

## DB-QB-063

Builder no guardará UnitOfWork.

## DB-QB-064

Builder no guardará Request.

## DB-QB-065

Builder no guardará AuthenticatedUser.

## DB-QB-066

Builder no guardará Tenant entity.

## DB-QB-067

Builder no guardará active telemetry Span.

## DB-QB-068

Builder será operation-scoped.

## DB-QB-069

No existirá global current Builder.

## DB-QB-070

No existirá static global ParameterId allocator.

## DB-QB-071

Concurrent builders tendrán states independientes.

## DB-QB-072

Builder deberá ser seguro para persistent runtimes.

## DB-QB-073

Builder copy/fork no compartirá mutabilidad peligrosa.

## DB-QB-074

Subqueries incorporadas serán artifacts finalizados, no child builders mutables.

## DB-QB-075

Parameter scopes evitarán colisiones.

## DB-QB-076

Physical placeholder identity no existirá en Builder.

## DB-QB-077

Metadata será distinta de Builder state.

## DB-QB-078

QueryContext será distinto de Builder state.

## DB-QB-079

ExecutionContext será distinto de Builder state.

## DB-QB-080

Builder diagnostics no expondrán secrets.

## DB-QB-081

Builder fingerprint no será compiled-query fingerprint.

## DB-QB-082

Builder serialization no será cache identity autoritativa.

## DB-QB-083

Canonical query identity se calculará sobre artifacts normalizados.

## DB-QB-084

Builder preservará projection order.

## DB-QB-085

Builder preservará declared join order antes de Optimization.

## DB-QB-086

Builder preservará logical predicate structure.

## DB-QB-087

Builder no aplicará simplificaciones inseguras de three-valued logic.

## DB-QB-088

Builder no realizará volatile expression folding.

## DB-QB-089

Builder no resolverá correlated references.

## DB-QB-090

Builder no validará GROUP BY mediante schema.

## DB-QB-091

Builder no validará set-operation type compatibility.

## DB-QB-092

Builder podrá realizar únicamente construction-level validation.

## DB-QB-093

Semantic errors pertenecerán al Semantic Engine.

## DB-QB-094

Compiler errors pertenecerán al Compiler.

## DB-QB-095

Execution errors pertenecerán al Executor.

## DB-QB-096

Builder API deberá priorizar IDE friendliness.

## DB-QB-097

Common query construction deberá requerir mínima configuración.

## DB-QB-098

Advanced behavior deberá estar disponible mediante progressive disclosure.

## DB-QB-099

Unsafe behavior deberá ser explícito.

## DB-QB-100

Platform-specific behavior deberá ser explícito o capability-driven.

## DB-QB-101

Builder deberá mantener separación entre value, identifier, expression y predicate.

## DB-QB-102

Builder deberá mantener separación entre query construction y query execution.

## DB-QB-103

Builder deberá mantener separación entre public API y internal Query Model.

## DB-QB-104

Builder deberá mantener separación entre Query Model y SQL representation.

## DB-QB-105

Policy-generated predicates deberán entrar antes del Semantic Engine.

## DB-QB-106

No se añadirán filtros invisibles directamente durante SQL compilation.

## DB-QB-107

Multitenancy será integración opcional.

## DB-QB-108

ORM será consumidor/integrador, no dependencia del Builder core.

## DB-QB-109

Query Builder será la API principal de construcción estructurada de queries.

## DB-QB-110

SQL Compiler seguirá siendo el único responsable de producir SQL.

---

# 246. Anti-patterns

## 246.1 SQL String Builder

```php
$sql = 'SELECT * FROM '.$table;
```

como implementación principal.

**Rechazado.**

---

## 246.2 PDO-aware Builder

```php
$builder->pdo->prepare(...);
```

**Rechazado.**

---

## 246.3 Placeholder-aware Builder

```php
$builder->bindings[] = '?';
```

confundiendo semantic parameter con SQL placeholder.

**Rechazado.**

---

## 246.4 Schema-aware hidden I/O

```php
$builder->select('*');

// silently queries information_schema
```

**Rechazado.**

---

## 246.5 Builder as AST

Utilizar el objeto mutable de developer API como representation autoritativa del Query Engine.

**Rechazado.**

---

## 246.6 Mutable subquery references

Guardar child builders mutables dentro del parent.

**Rechazado.**

---

## 246.7 Generic string semantics

```php
where('a', '=', 'b')
```

y adivinar si `b` es columna o valor.

**Rechazado.**

---

## 246.8 Vendor branches

```php
if ($driver === 'pgsql') {
    ...
}
```

dentro del Builder.

**Rechazado.**

---

## 246.9 Raw by default

Tratar expresiones desconocidas como SQL raw.

**Rechazado.**

---

## 246.10 Hidden framework predicates

Añadir tenant/security/soft-delete filters después del Semantic Engine directamente al SQL.

**Rechazado.**

---

## 246.11 Static builder state

```php
QueryBuilder::$current;
```

**Rechazado.**

---

## 246.12 Global parameter counter

```php
static $parameterId = 0;
```

**Rechazado.**

---

## 246.13 ORM duplicate query engine

```text
DB Query Builder Engine
+
ORM Query Builder Engine
```

con pipelines separados.

**Rechazado.**

---

## 246.14 `toSql()` implemented by Builder

```php
public function toSql()
{
    return 'SELECT ...';
}
```

**Rechazado.**

---

# 247. Ejemplo completo

Código:

```php
$query = DB::table('users as u')
    ->select(
        'u.id',
        'u.email',
    )
    ->join('orders as o', function ($join) {
        $join->on('o.user_id', '=', 'u.id')
             ->where('o.status', 'paid');
    })
    ->where('u.active', true)
    ->whereIn('u.id', $userIds)
    ->orderBy('u.email')
    ->limit(100);
```

---

# 248. Builder state

Conceptualmente:

```text
SelectBuilderState
│
├── FROM
│   └── users AS u
│
├── PROJECTIONS
│   ├── u.id
│   └── u.email
│
├── JOINS
│   └── orders AS o
│       ├── o.user_id = u.id
│       └── o.status = P1
│
├── WHERE
│   ├── u.active = P2
│   └── u.id IN P3
│
├── ORDER
│   └── u.email ASC
│
└── LIMIT
    └── 100
```

---

# 249. Parameter definitions

```text
P1
├── shape: SCALAR
└── type: UNKNOWN

P2
├── shape: SCALAR
└── type: UNKNOWN

P3
├── shape: COLLECTION
└── elementType: UNKNOWN
```

---

# 250. BindingSet

```text
P1 → "paid"
P2 → true
P3 → $userIds
```

---

# 251. Query Model

Después de finalización:

```text
SelectQueryModel
│
├── FromRelation
├── ProjectionList
├── JoinList
├── Predicate
├── Ordering
├── Limit
├── ParameterDefinitions
└── QueryMetadata
```

inmutable.

---

# 252. Query AST

Posteriormente:

```text
SelectQueryNode
│
├── RelationReference(users AS u)
│
├── Projection
│   ├── ColumnReference(u.id)
│   └── ColumnReference(u.email)
│
├── Join
│   ├── RelationReference(orders AS o)
│   └── AND
│       ├── o.user_id = u.id
│       └── o.status = Parameter(P1)
│
├── WHERE
│   └── AND
│       ├── u.active = Parameter(P2)
│       └── u.id IN CollectionParameter(P3)
│
├── ORDER BY
│   └── u.email ASC
│
└── LIMIT 100
```

---

# 253. Semantic processing

Después:

```text
AST
 │
 ▼
Normalization
 │
 ▼
Validation
 │
 ▼
Semantic Analysis
```

podrá descubrir:

```text
u → users
o → orders

u.id → UserId
o.user_id → UserId

P1 → OrderStatus
P2 → Boolean
P3 → Collection<UserId>

o.user_id = u.id
→ FK relationship

orders → users
→ MANY_TO_ONE

u.id IN P3
→ UserId collection constraint
```

Nada de esto tuvo que resolverlo el Builder.

---

# 254. Optimizer

El Optimizer podrá usar esos hechos para:

```text
predicate simplification
predicate placement
join analysis
redundant operation removal
constraint propagation
```

sin modificar el Builder original.

---

# 255. Planner

El Planner decidirá:

```text
collection binding strategy
join strategy
capability strategy
logical plan
physical plan
```

---

# 256. Compiler

Finalmente:

```text
Physical Query Plan
        │
        ▼
Target SQL Compiler
        │
        ▼
CompiledQuery
```

recién aquí aparece SQL.

---

# 257. Arquitectura completa

```text
                      APPLICATION
                           │
                           ▼
                      DB FACADE
                           │
                           ▼
                 QUERY BUILDER FACTORY
                           │
                           ▼
                    QUERY BUILDER
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   Expressions         Predicates         Relations
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
                    BUILDER FINALIZER
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
         Query Model   Bindings      Metadata
             │
             ▼
          Query AST
             │
             ▼
        Normalization
             │
             ▼
         Validation
             │
             ▼
      Semantic Analysis
             │
             ▼
    SemanticQueryArtifact
             │
             ▼
         Optimizer
             │
             ▼
          Planner
             │
             ▼
         Compiler
             │
             ▼
       CompiledQuery
             │
             ▼
         Executor
             │
             ▼
       Connection
             │
             ▼
          Driver
```

---

# 258. Fórmulas maestras

## Query Builder

```text
Query Builder
=
Fluent Construction API
+
Structured Identifiers
+
Expressions
+
Predicates
+
Relations
+
Parameters
+
Declarative Metadata
```

---

## Query Build Artifact

```text
QueryBuildArtifact
=
Immutable Query Model
+
Parameter Definitions
+
BindingSet
+
QueryMetadata
```

---

## Safe query construction

```text
Safe Query Construction
=
Structured Identifiers
+
Structured Operators
+
Automatic Parameterization
+
Explicit Raw Escape Hatches
+
Immutable Final Artifacts
```

---

## Builder boundary

```text
Builder Boundary
=
Developer Intent
→
Structured Query Representation
```

No:

```text
Developer Intent
→
SQL String
```

---

# 259. Diseño definitivo

La filosofía del Query Builder de VoltStack será:

```text
Laravel-like DX
        +
Strong Structured Query Model
        +
AST-based Internals
        +
Semantic Query Engine
        +
Capability-driven Architecture
        +
Persistent Runtime Safety
```

El desarrollador podrá escribir:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

mientras internamente VoltStack conservará:

```text
Builder
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
Semantic Graph
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Executor
```

sin permitir que las responsabilidades de esas capas se mezclen.

---

# 260. Conclusión

`QueryBuilder` será la capa de **developer experience para construcción de consultas**, no un generador de SQL.

La separación fundamental será:

```text
QueryBuilder
    │
    │ constructs
    ▼
Query Model / AST
```

mientras:

```text
SQL Compiler
    │
    │ renders
    ▼
SQL
```

y:

```text
Executor
    │
    │ executes
    ▼
Database
```

Esta separación permite combinar la facilidad de uso esperada por desarrolladores provenientes de Laravel con una arquitectura interna más estricta, extensible y adecuada para:

```text
semantic analysis
query optimization
multi-platform compilation
query caching
persistent runtimes
offline analysis
ORM reuse
advanced developer tooling
```

La regla definitiva del subsistema será:

> **El Query Builder expresa la intención del desarrollador en una representación estructurada; nunca convierte esa intención directamente en SQL.**

---

# 261. Siguiente documento

```text
44_DATABASE_SELECT_QUERY_BUILDER.md
```

El siguiente documento especializará esta arquitectura para las consultas `SELECT`, incluyendo:

```text
FROM
SELECT / projections
DISTINCT
WHERE
JOIN
GROUP BY
HAVING
WINDOW
ORDER BY
LIMIT / OFFSET
locking
subqueries
CTEs
set operations
terminal read operations
result-shape declarations
```

manteniendo:

```text
SelectQueryBuilder
      │
      ▼
SelectQueryModel
      │
      ▼
SelectQuery AST
```

y nunca:

```text
SelectQueryBuilder
      │
      X
      ▼
SQL String
```