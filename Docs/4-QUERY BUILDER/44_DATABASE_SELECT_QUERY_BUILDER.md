# 44_DATABASE_SELECT_QUERY_BUILDER.md

# VoltStack Quantum Database
## Select Query Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 44 — Select Query Builder  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / SELECT  
**Versión:** 1.0

---

# 1. Propósito

`SelectQueryBuilder` será la API especializada de VoltStack para construir consultas orientadas a lectura.

Ejemplo:

```php
$users = DB::table('users')
    ->select('id', 'name', 'email')
    ->where('active', true)
    ->orderBy('name')
    ->limit(100)
    ->get();
```

La API tendrá ergonomía familiar para desarrolladores provenientes de Laravel, pero internamente no construirá SQL.

El flujo será:

```text
Developer
   │
   ▼
SelectQueryBuilder
   │
   ▼
SelectQueryModel
   │
   ▼
SelectQueryNode
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
Optimizer
   │
   ▼
Planner
   │
   ▼
SQL Compiler
   │
   ▼
Executor
```

---

# 2. Regla fundamental

```text
SelectQueryBuilder
≠
SELECT SQL generator
```

El Builder representa:

```text
developer intent
```

mediante estructuras semánticas.

El Compiler representa esa estructura posteriormente como SQL.

Por tanto:

> `SelectQueryBuilder` construye una consulta SELECT semántica; nunca genera directamente una sentencia `SELECT`.

---

# 3. Responsabilidades

`SelectQueryBuilder` será responsable de construir:

```text
FROM
SELECT projections
DISTINCT
WHERE
JOIN
GROUP BY
HAVING
WINDOW
ORDER BY
LIMIT
OFFSET
LOCKING
CTEs
SUBQUERIES
SET OPERATIONS
QUERY METADATA
PARAMETER DEFINITIONS
BINDINGS
```

---

# 4. No responsabilidades

No será responsable de:

```text
SQL rendering
identifier quoting
placeholder generation
schema introspection
column existence validation
symbol resolution
type inference
join relationship inference
aggregate semantic validation
query optimization
index selection
physical planning
connection routing
connection acquisition
parameter binding
query execution
result hydration
ORM entity construction
```

---

# 5. Arquitectura

```text
                    SelectQueryBuilder
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
       ▼                   ▼                    ▼
 ProjectionBuilder   PredicateBuilder      RelationBuilder
       │                   │                    │
       │                   ├──────────┐         │
       │                   │          │         │
       ▼                   ▼          ▼         ▼
 Expressions           WHERE       HAVING     JOIN
       │                   │          │         │
       └───────────────────┴──────────┴─────────┘
                           │
                           ▼
                  SelectBuilderState
                           │
                           ▼
                   BuilderFinalizer
                           │
                           ▼
                   SelectQueryModel
                           │
                           ▼
                    SelectQueryNode
```

---

# 6. Public API

La API básica podrá comenzar con:

```php
DB::table('users');
```

conceptualmente equivalente a:

```php
DB::select()
    ->from('users');
```

La primera forma será la recomendada para el caso común.

---

# 7. Creación

Ejemplo:

```php
$query = DB::table('users');
```

creará conceptualmente:

```text
SelectQueryBuilder
└── FROM
    └── TableRelation(users)
```

No ocurrirá:

```text
database connection
schema introspection
SQL compilation
query execution
```

---

# 8. SelectBuilderState

Durante construcción podrá existir:

```php
final class SelectBuilderState
{
    // mutable operation-local state
}
```

Conceptualmente contendrá:

```text
from
projections
joins
where
groupBy
having
windows
orderBy
limit
offset
locking
ctes
setOperations
parameterRegistry
metadataBuilder
```

---

# 9. Estado operation-scoped

`SelectBuilderState` será:

```text
operation-local
mutable
temporary
non-cacheable
non-shared
```

Nunca:

```text
singleton
worker-global
request-global
static current query
```

---

# 10. Finalización

```text
SelectBuilderState
        │
        ▼
SelectBuilderFinalizer
        │
        ▼
Immutable SelectQueryModel
```

Después:

```text
SelectQueryModel
      │
      ▼
Query AST lowering
      │
      ▼
SelectQueryNode
```

---

# 11. Modelo conceptual

```php
final readonly class SelectQueryModel
{
    public function __construct(
        public ?RelationModel $from,
        public ProjectionList $projections,
        public bool $distinct,
        public JoinList $joins,
        public ?PredicateNode $where,
        public GroupingList $groupBy,
        public ?PredicateNode $having,
        public WindowDefinitionSet $windows,
        public OrderingList $orderBy,
        public ?LimitSpecification $limit,
        public ?OffsetSpecification $offset,
        public ?LockSpecification $lock,
        public CteDefinitionSet $ctes,
        public SetOperationList $setOperations,
        public ParameterDefinitionSet $parameters,
        public QueryMetadata $metadata,
    ) {}
}
```

La implementación definitiva podrá dividir estas estructuras en componentes especializados.

---

# 12. FROM

El origen más simple:

```php
DB::table('users');
```

produce:

```text
TableRelation
└── users
```

---

# 13. Explicit from()

También podrá utilizarse:

```php
DB::select()
    ->from('users');
```

---

# 14. Qualified relation

```php
DB::table('public.users');
```

podrá producir:

```text
QualifiedRelationIdentifier
├── namespace/schema: public
└── relation: users
```

La interpretación final dependerá del Platform Semantic Profile.

---

# 15. Relation alias

```php
DB::table('users as u');
```

podrá utilizarse como shorthand.

Internamente:

```text
AliasedRelation
├── relation
│   └── users
└── alias
    └── u
```

---

# 16. Alias explícito

La API avanzada podrá permitir:

```php
DB::table('users')
    ->as('u');
```

o:

```php
DB::select()
    ->from('users', as: 'u');
```

---

# 17. No SQL parsing interno

Aunque:

```text
users as u
```

sea aceptado en el boundary, después de interpretarlo deberá convertirse inmediatamente en representación estructurada.

No deberá conservarse como fragmento SQL.

---

# 18. FROM subquery

Ejemplo conceptual:

```php
$activeUsers = DB::table('users')
    ->where('active', true);

$query = DB::query()
    ->fromSubquery($activeUsers, 'active_users');
```

---

# 19. Freeze de subquery

Antes de incorporarse:

```text
Mutable Child Builder
       │
       ▼
freeze
       │
       ▼
Subquery Artifact
       │
       ▼
Parent Select Builder
```

El parent no almacenará el Builder hijo mutable.

---

# 20. Derived relation

La subquery se representará como:

```text
DerivedRelation
├── query
└── alias
```

---

# 21. VALUES relation

La arquitectura podrá soportar:

```text
VALUES
```

como una relación estructurada.

Ejemplo conceptual:

```php
$query->fromValues(
    rows: [
        [1, 'A'],
        [2, 'B'],
    ],
    columns: ['id', 'name'],
    alias: 'items',
);
```

---

# 22. Table functions

Plataformas que soporten table-valued functions podrán representarlas mediante:

```text
FunctionRelation
```

con capability requirements explícitos.

---

# 23. SELECT projections

Sin `select()` explícito:

```php
DB::table('users')->get();
```

podrá interpretarse como:

```text
SELECT wildcard
```

semánticamente.

---

# 24. Wildcard

Internamente:

```text
WildcardProjection
```

No:

```text
"*"
```

como SQL raw.

---

# 25. Wildcard expansion

El Builder no expandirá:

```text
*
```

a columnas.

Eso ocurrirá durante Schema-Aware Semantic Resolution.

---

# 26. Qualified wildcard

```php
$query->select('users.*');
```

produce:

```text
QualifiedWildcardProjection
└── qualifier: users
```

---

# 27. Explicit projections

```php
$query->select(
    'id',
    'name',
    'email',
);
```

produce:

```text
ProjectionList
├── ColumnReference(id)
├── ColumnReference(name)
└── ColumnReference(email)
```

---

# 28. Projection order

El orden deberá conservarse.

```php
->select('email', 'id', 'name')
```

debe representar exactamente:

```text
1 email
2 id
3 name
```

---

# 29. addSelect()

```php
$query
    ->select('id', 'name')
    ->addSelect('email');
```

produce:

```text
id
name
email
```

---

# 30. Expression projections

Podrán seleccionarse expresiones:

```php
$query->select(
    DB::expr()->lower(
        DB::expr()->column('email')
    )
);
```

---

# 31. Projection alias

API conceptual:

```php
$query->select([
    'normalized_email' => DB::expr()->lower(
        DB::expr()->column('email')
    ),
]);
```

produce:

```text
AliasedProjection
├── expression
│   └── FunctionExpression(lower)
└── alias
    └── normalized_email
```

---

# 32. Alias semántico

El alias será:

```text
Identifier
```

no SQL text.

---

# 33. ProjectionBuilder

Podrá existir:

```text
ProjectionBuilder
```

especializado en:

```text
column
wildcard
expression
alias
subquery
aggregate
window expression
```

---

# 34. selectRaw()

VoltStack podrá ofrecer:

```php
$query->selectRaw(...);
```

como escape hatch.

Debe producir:

```text
RawExpression
```

o:

```text
RawProjection
```

explícito.

---

# 35. Raw projection

Raw SQL deberá:

```text
be explicit
support safe bindings
carry trust metadata
carry portability classification
act conservatively in semantic analysis
```

---

# 36. DISTINCT

```php
$query->distinct();
```

produce:

```text
DistinctSpecification
```

---

# 37. DISTINCT boolean state

No se representará como string:

```text
"DISTINCT"
```

---

# 38. DISTINCT expressions

Si una plataforma soporta mecanismos como `DISTINCT ON`, éstos deberán modelarse como capability-dependent semantic features.

No deberán esconderse dentro de:

```php
distinct('column');
```

sin semántica definida.

---

# 39. WHERE

Ejemplo:

```php
$query->where('active', true);
```

produce:

```text
ComparisonPredicate
├── ColumnReference(active)
├── EQUAL
└── Parameter(P1)
```

---

# 40. Parameter binding

Separadamente:

```text
P1 → true
```

---

# 41. where(column, value)

Forma:

```php
where('active', true)
```

equivale conceptualmente a:

```text
active = Parameter
```

---

# 42. where(column, operator, value)

```php
where('age', '>=', 18)
```

produce:

```text
ComparisonPredicate
├── ColumnReference(age)
├── GREATER_THAN_OR_EQUAL
└── Parameter(P1)
```

---

# 43. Operator normalization

```text
>=
```

se transforma en:

```text
ComparisonOperator::GREATER_THAN_OR_EQUAL
```

---

# 44. Multiple where()

```php
$query
    ->where('active', true)
    ->where('age', '>=', 18);
```

produce conceptualmente:

```text
AND
├── active = P1
└── age >= P2
```

---

# 45. orWhere()

```php
$query
    ->where('role', 'admin')
    ->orWhere('role', 'manager');
```

produce:

```text
OR
├── role = P1
└── role = P2
```

---

# 46. Grouped predicates

```php
$query->where(function ($where) {
    $where
        ->where('role', 'admin')
        ->orWhere('role', 'manager');
});
```

produce:

```text
PredicateGroup
└── OR
    ├── role = P1
    └── role = P2
```

---

# 47. Closure disappears

La closure:

```text
exists only during construction
```

No formará parte de:

```text
QueryModel
AST
SemanticArtifact
CompiledQuery
```

---

# 48. whereNot()

Podrá existir:

```php
$query->whereNot(function ($where) {
    // ...
});
```

produciendo:

```text
NotPredicate
```

---

# 49. NULL predicates

```php
$query->whereNull('deleted_at');
$query->whereNotNull('deleted_at');
```

producen:

```text
NullPredicate
NotNullPredicate
```

---

# 50. Null shorthand

```php
$query->where('deleted_at', null);
```

podrá normalizarse al mismo significado definido por la API.

---

# 51. whereIn()

```php
$query->whereIn('id', $ids);
```

produce:

```text
InPredicate
├── ColumnReference(id)
└── CollectionParameter(P1)
```

---

# 52. whereNotIn()

Produce:

```text
NotInPredicate
```

o:

```text
InPredicate
└── negated = true
```

según el modelo definitivo del Predicate System.

---

# 53. Empty IN

El Builder no generará:

```sql
IN ()
```

La semántica de colección vacía se resolverá antes del Compiler.

---

# 54. Large IN

El Builder tampoco decidirá si una colección grande se compila mediante:

```text
expanded placeholders
native arrays
VALUES
temporary relation
other platform strategy
```

Eso pertenece al Binding Planner / Query Planner.

---

# 55. BETWEEN

```php
$query->whereBetween('age', [18, 65]);
```

produce:

```text
BetweenPredicate
├── value: age
├── lower: P1
└── upper: P2
```

---

# 56. NOT BETWEEN

```php
$query->whereNotBetween(...);
```

mantendrá negación explícita.

---

# 57. Column comparison

```php
$query->whereColumn(
    'users.id',
    '=',
    'orders.user_id'
);
```

produce:

```text
ComparisonPredicate
├── ColumnReference(users.id)
├── EQUAL
└── ColumnReference(orders.user_id)
```

---

# 58. Value vs column

```php
where('status', 'active')
```

significa:

```text
status = Parameter("active")
```

mientras:

```php
whereColumn('a', 'b')
```

significa:

```text
Column(a) = Column(b)
```

---

# 59. EXISTS

```php
$query->whereExists(function ($subquery) {
    $subquery
        ->from('orders')
        ->whereColumn(
            'orders.user_id',
            'users.id'
        );
});
```

produce:

```text
ExistsPredicate
└── Subquery
```

---

# 60. Correlation resolution

El Builder no determina formalmente que:

```text
users.id
```

pertenece al outer query.

El Semantic Engine resolverá la correlación.

---

# 61. Pattern predicates

La API podrá soportar:

```php
->whereLike(...)
->whereNotLike(...)
->whereContains(...)
->whereStartsWith(...)
->whereEndsWith(...)
```

---

# 62. Semantic pattern operations

`contains` no deberá convertirse inmediatamente a:

```text
%value%
```

como SQL final.

Podrá representarse semánticamente como:

```text
PatternPredicate
└── mode: CONTAINS
```

permitiendo al Compiler aplicar reglas de escape.

---

# 63. Case sensitivity

Podrá declararse:

```text
DEFAULT
CASE_SENSITIVE
CASE_INSENSITIVE
```

sin usar directamente conceptos vendor-specific como `ILIKE`.

---

# 64. JSON predicates

Podrán existir APIs como:

```php
->whereJsonContains(...)
->whereJsonPath(...)
```

pero deberán producir AST semántico.

---

# 65. JSON capability requirements

La disponibilidad real se resolverá mediante:

```text
Platform Capabilities
```

no:

```php
if ($driver === 'pgsql')
```

---

# 66. JOIN

Ejemplo:

```php
$query->join(
    'orders',
    'orders.user_id',
    '=',
    'users.id'
);
```

produce:

```text
JoinModel
├── kind: INNER
├── relation: orders
└── predicate
    └── orders.user_id = users.id
```

---

# 67. Join kinds

La API podrá soportar:

```text
INNER
LEFT
RIGHT
FULL
CROSS
```

según capability support.

---

# 68. Convenience methods

Podrán existir:

```php
->join(...)
->leftJoin(...)
->rightJoin(...)
->fullJoin(...)
->crossJoin(...)
```

---

# 69. JoinConditionBuilder

Para joins complejos:

```php
$query->leftJoin('orders as o', function ($join) {
    $join
        ->on('o.user_id', '=', 'users.id')
        ->where('o.status', 'paid');
});
```

---

# 70. `on()` semantics

```php
$join->on('a.id', '=', 'b.a_id');
```

significa:

```text
ColumnReference
operator
ColumnReference
```

---

# 71. Join `where()` semantics

```php
$join->where('o.status', 'paid');
```

significa:

```text
ColumnReference
operator
Parameter
```

---

# 72. Join condition composition

```php
$join
    ->on(...)
    ->orOn(...);
```

produce Predicate AST estructurado.

---

# 73. Join relation resolution

Builder no determina:

```text
foreign key
cardinality
uniqueness
key preservation
relationship direction
```

Eso corresponde a:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

---

# 74. Join nullability

Un `LEFT JOIN` puede hacer nullable una columna físicamente `NOT NULL`.

Eso se determinará semánticamente.

Builder sólo conserva:

```text
JoinKind::LEFT
```

---

# 75. Join order

El orden declarado se preservará en el Query Model inicial.

---

# 76. Optimized join order

Posteriormente:

```text
Semantic Query
       │
       ▼
Optimizer
       │
       ▼
Potentially reordered joins
```

si la transformación es válida.

---

# 77. GROUP BY

```php
$query->groupBy(
    'country',
    'status'
);
```

produce:

```text
GroupingList
├── ColumnReference(country)
└── ColumnReference(status)
```

---

# 78. Group expressions

Podrán utilizarse:

```text
ExpressionNode
```

como grouping keys.

---

# 79. Aggregate expressions

Ejemplo:

```php
$query->select([
    'country',
    'total' => DB::expr()->count('*'),
]);
```

produce una expresión aggregate semántica.

---

# 80. No SQL aggregate generation

El Builder no construirá:

```sql
COUNT(*)
```

como string.

---

# 81. HAVING

```php
$query
    ->groupBy('country')
    ->having(
        DB::expr()->count('*'),
        '>',
        10
    );
```

produce Predicate AST en:

```text
HAVING context
```

---

# 82. WHERE ≠ HAVING

Aunque ambos utilicen predicates:

```text
WHERE predicate
≠
HAVING predicate
```

por contexto semántico.

---

# 83. Aggregate validation

Builder no determina si:

```text
SELECT country, name, COUNT(*)
GROUP BY country
```

es semánticamente válido.

Eso pertenece al Semantic Analysis System.

---

# 84. Functional dependencies

Tampoco determinará si una columna no agrupada puede ser válida por dependencia funcional.

Eso pertenece al Constraint/Semantic System.

---

# 85. Window functions

VoltStack soportará expresiones window estructuradas.

Ejemplo conceptual:

```php
DB::expr()
    ->rowNumber()
    ->over(
        partitionBy: ['department_id'],
        orderBy: ['created_at']
    );
```

---

# 86. Window model

```text
WindowExpression
├── function
└── specification
    ├── partitionBy
    ├── orderBy
    └── frame
```

---

# 87. Named windows

Podrá soportarse:

```php
$query->window('recent', function ($window) {
    // ...
});
```

produciendo:

```text
NamedWindowDefinition
```

---

# 88. Window frame

El modelo deberá poder representar semánticamente:

```text
ROWS
RANGE
GROUPS

UNBOUNDED PRECEDING
CURRENT ROW
N PRECEDING
N FOLLOWING
UNBOUNDED FOLLOWING
```

cuando las capabilities lo permitan.

---

# 89. ORDER BY

```php
$query->orderBy('name');
```

equivale a:

```text
OrderByItem
├── expression: name
└── direction: ASC
```

---

# 90. DESC

```php
$query->orderBy('created_at', 'desc');
```

normaliza:

```text
OrderDirection::DESC
```

---

# 91. orderByDesc()

Convenience API:

```php
$query->orderByDesc('created_at');
```

---

# 92. Expression ordering

```php
$query->orderBy(
    DB::expr()->lower(
        DB::expr()->column('name')
    )
);
```

será soportado.

---

# 93. NULL ordering

Podrá declararse:

```text
PLATFORM_DEFAULT
NULLS_FIRST
NULLS_LAST
```

---

# 94. Capability resolution

El Builder no decidirá cómo emular `NULLS LAST` en una plataforma que no tenga sintaxis directa.

---

# 95. Random ordering

Una API como:

```php
$query->inRandomOrder();
```

deberá bajar a una función/ordering semántico.

No deberá insertar directamente:

```text
RAND()
```

o:

```text
RANDOM()
```

---

# 96. LIMIT

```php
$query->limit(50);
```

produce:

```text
LimitSpecification(50)
```

---

# 97. OFFSET

```php
$query->offset(100);
```

produce:

```text
OffsetSpecification(100)
```

---

# 98. Validation

Valores negativos deberán rechazarse en construction/validation boundary según la semántica pública definida.

---

# 99. take()/skip()

Para ergonomía Laravel-like podrán existir aliases:

```php
->take(50);
->skip(100);
```

que produzcan exactamente las mismas estructuras.

---

# 100. Pagination helpers

Métodos como:

```php
paginate();
simplePaginate();
cursorPaginate();
```

podrán existir como APIs superiores.

Sin embargo, la arquitectura específica pertenecerá a:

```text
199_DATABASE_PAGINATION_SYSTEM.md
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 101. LIMIT/OFFSET ≠ pagination system

El Builder sólo representa los componentes query necesarios.

---

# 102. CTE

Ejemplo:

```php
$activeUsers = DB::table('users')
    ->where('active', true);

$query = DB::query()
    ->with('active_users', $activeUsers)
    ->from('active_users');
```

---

# 103. CteDefinition

```text
CteDefinition
├── name
├── optional column list
├── query
├── recursion mode
└── materialization hints
```

cuando sean soportados.

---

# 104. CTE scope

Builder registra la estructura.

Semantic Analysis determina:

```text
scope
symbol visibility
dependencies
output relation
types
recursive references
```

---

# 105. Recursive CTE

Ejemplo conceptual:

```php
$query->withRecursive('tree', ...);
```

---

# 106. Recursive semantic validation

El Builder no intentará demostrar la validez semántica de la recursión.

---

# 107. CTE dependency cycles

Ciclos semánticos válidos o inválidos serán tratados por:

```text
Semantic CTE Dependency Analysis
```

---

# 108. Subquery expressions

Ejemplo:

```php
$query->select([
    'last_order_at' => DB::table('orders')
        ->select('created_at')
        ->whereColumn('orders.user_id', 'users.id')
        ->orderByDesc('created_at')
        ->limit(1),
]);
```

podrá convertirse en:

```text
ScalarSubqueryExpression
```

---

# 109. Scalar semantics

El Builder no garantiza que una scalar subquery produzca exactamente una columna/fila.

Validation/Semantic Analysis podrá establecer los requisitos correspondientes.

---

# 110. UNION

```php
$queryA->union($queryB);
```

produce:

```text
SetOperation
├── type: UNION
├── left
└── right
```

---

# 111. UNION ALL

```php
$queryA->unionAll($queryB);
```

produce:

```text
SetOperation
└── duplicatePolicy: ALL
```

---

# 112. INTERSECT

Podrá soportarse:

```php
$queryA->intersect($queryB);
```

cuando las capabilities lo permitan.

---

# 113. EXCEPT

Igualmente:

```php
$queryA->except($queryB);
```

---

# 114. Set-operation semantic compatibility

Builder no resolverá:

```text
output arity
common output types
nullability
collation compatibility
domain type compatibility
```

---

# 115. Set operation tree

Una cadena:

```text
A UNION B INTERSECT C
```

deberá conservar estructura explícita.

No depender de concatenación textual o precedencia SQL implícita.

---

# 116. Locking

La API podrá soportar:

```php
$query->forUpdate();
```

produciendo:

```text
LockSpecification
└── mode: UPDATE
```

---

# 117. Shared locking

Podrá existir:

```php
$query->forShare();
```

o API equivalente.

---

# 118. Lock wait policy

Semánticamente:

```text
WAIT
NO_WAIT
SKIP_LOCKED
PLATFORM_DEFAULT
```

---

# 119. Lock targets

Cuando la plataforma lo permita:

```text
LockTargetSet
```

podrá especificar relaciones concretas.

---

# 120. Locking capabilities

Builder únicamente declara intención.

Semantic/Capability/Planner determinan disponibilidad.

---

# 121. No vendor locking syntax

Nunca en Builder:

```php
if ($platform === 'postgresql') {
    $sql .= ' FOR UPDATE SKIP LOCKED';
}
```

---

# 122. Query metadata

Ejemplo:

```php
$query
    ->label('users.active')
    ->consistency(Consistency::READ_YOUR_WRITES)
    ->timeout(seconds: 2)
    ->usePrimary();
```

---

# 123. Metadata output

Produce:

```text
QueryMetadata
├── label
├── consistency requirement
├── statement timeout
└── connection preference
```

---

# 124. Query metadata does not select connection

```text
usePrimary()
```

declara un requirement/preference.

No obtiene una conexión.

---

# 125. Query intent

Un `SelectQueryBuilder` normalmente derivará:

```text
QueryIntent::READ
```

pero ciertas opciones pueden derivar:

```text
LOCKING_READ
```

---

# 126. Read intent ≠ replica eligibility

Una query `SELECT` no es automáticamente segura para replica.

Factores como:

```text
transaction
consistency
locking
session state
read-your-writes
platform semantics
```

pueden exigir primary.

---

# 127. Query Builder no enruta

La decisión final pertenece a:

```text
Read/Write Routing
Connection Resolution
Execution Context
```

---

# 128. Result shape declarations

El Builder podrá declarar intención de resultado:

```text
ROWS
SINGLE_ROW
SCALAR
CURSOR
STREAM
EXISTS
```

---

# 129. Result mode metadata

Esta intención podrá formar parte de:

```text
QueryRequirements
```

o del terminal operation request.

---

# 130. get()

```php
$query->get();
```

será un terminal operation.

---

# 131. get() pipeline

```text
SelectQueryBuilder
      │
      ▼
finalize()
      │
      ▼
SelectQueryArtifact + BindingSet
      │
      ▼
QuerySubmissionGateway
      │
      ▼
Query Processing Pipeline
      │
      ▼
Execution
      │
      ▼
Result
```

---

# 132. first()

```php
$query->first();
```

podrá representar:

```text
result expectation: SINGLE_ROW
```

y, cuando semánticamente apropiado:

```text
effective limit: 1
```

---

# 133. first() must not mutate reusable builder unexpectedly

Si:

```php
$query->first();
```

necesita `LIMIT 1`, deberá aplicarlo sobre:

```text
terminal-operation snapshot
```

o copia finalizada.

No deberá cambiar accidentalmente el Builder reutilizable.

---

# 134. firstOrFail()

Una API superior podrá:

```php
$query->firstOrFail();
```

La excepción por ausencia de resultado pertenece a developer API/result handling, no SQL Compiler.

---

# 135. value()

```php
$query->value('email');
```

podrá producir una query derivada con:

```text
single projection
single-row expectation
scalar result mode
```

---

# 136. pluck()

```php
$query->pluck('email');
```

podrá producir:

```text
single-column projection
collection result transformation
```

---

# 137. exists()

```php
$query->exists();
```

deberá expresar intención semántica de existencia.

---

# 138. exists() optimization

El Builder no necesita decidir la estrategia SQL exacta.

Podrá crear:

```text
ExistenceQueryRequest
```

y permitir que Planner/Compiler produzcan la estrategia eficiente.

---

# 139. doesntExist()

Podrá ser una operación derivada de existencia.

---

# 140. count()

```php
$query->count();
```

podrá construir:

```text
AggregateQueryRequest
```

o una query derivada estructurada.

---

# 141. Aggregate terminal operation

El diseño debe evitar que:

```php
count()
```

simplemente concatene:

```sql
SELECT COUNT(*)
```

---

# 142. min/max/avg/sum

Igual principio para:

```php
min()
max()
avg()
sum()
```

---

# 143. cursor()

```php
$query->cursor();
```

declarará:

```text
ResultMode::CURSOR
```

o equivalente.

---

# 144. stream()

Podrá declarar:

```text
ResultMode::STREAMING
```

---

# 145. Streaming semantics

Builder no abrirá el cursor.

El Executor/Result System será propietario del recurso runtime.

---

# 146. Lazy iteration

Una API lazy deberá distinguir:

```text
query construction
```

de:

```text
resource lifetime
```

---

# 147. chunk()

`chunk()` será una operación de procesamiento de datasets, no una primitive estructural SELECT.

Su arquitectura pertenece a:

```text
201_DATABASE_CHUNK_PROCESSING_SYSTEM.md
```

---

# 148. lazy()

Igualmente:

```text
202_DATABASE_LAZY_COLLECTION_SYSTEM.md
```

---

# 149. Builder reuse

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

# 150. Copy semantics

`copy()` deberá crear estado independiente.

---

# 151. Fork semantics

Podrá existir:

```php
$base->fork();
```

si se desea distinguir conceptualmente:

```text
clone implementation
```

de:

```text
semantic query branch
```

---

# 152. No accidental aliasing

Debe evitarse que una modificación de `$admins` cambie `$customers`.

---

# 153. Immutable alternative

Una versión futura podría adoptar fluent immutable builders:

```php
$new = $query->where(...);
```

pero no será requisito inicial.

---

# 154. Dynamic queries

Ejemplo:

```php
$query = DB::table('orders');

if ($status !== null) {
    $query->where('status', $status);
}

if ($customerId !== null) {
    $query->where('customer_id', $customerId);
}

if ($from !== null) {
    $query->where('created_at', '>=', $from);
}
```

será un caso de uso de primera clase.

---

# 155. when()

Podrá utilizarse:

```php
$query->when(
    $status !== null,
    fn ($query) =>
        $query->where('status', $status)
);
```

---

# 156. when() disappears

`when()` es únicamente developer-construction logic.

No existe en el Query AST.

---

# 157. unless()

Mismo principio.

---

# 158. tap()

Helpers de inspección/construcción podrán existir sin convertirse en semántica query.

---

# 159. Scopes

La capa ORM podrá definir:

```php
User::query()
    ->active()
    ->verified();
```

---

# 160. Scope expansion

Los scopes deberán expandirse antes del procesamiento semántico a:

```text
structured query operations
```

---

# 161. Example

```text
active()
    ↓
where('active', true)

verified()
    ↓
whereNotNull('verified_at')
```

---

# 162. No hidden scope SQL

Nunca:

```text
Scope
→ append raw SQL after compilation
```

---

# 163. Soft-delete integration

Un scope:

```text
deleted_at IS NULL
```

deberá aparecer como Predicate AST.

---

# 164. Tenant integration

Igualmente:

```text
tenant_id = P1
```

deberá ser estructurado y visible.

---

# 165. Authorization integration

Filtros de data access podrán producir predicates estructurados mediante adapters/policies.

---

# 166. Provenance

Predicates añadidos por framework integrations podrán registrar:

```text
origin
transformation
policy identifier
```

sin guardar User/Tenant objects.

---

# 167. ExpressionBuilder integration

`SelectQueryBuilder` reutilizará:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

---

# 168. PredicateBuilder integration

Reutilizará:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

---

# 169. Parameter integration

Reutilizará:

```text
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

---

# 170. Query Type integration

El Builder podrá proporcionar hints.

La resolución real corresponde a:

```text
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
```

---

# 171. Semantic integration

Una vez finalizado:

```text
SelectQueryNode
      │
      ▼
Semantic Analysis
```

resolverá significado.

---

# 172. Symbol resolution

Ejemplo:

```text
u.email
```

será resuelto mediante:

```text
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
```

---

# 173. Schema-aware resolution

La existencia real de:

```text
users.email
```

se determinará mediante:

```text
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
```

---

# 174. Relation resolution

Los joins serán resueltos mediante:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

---

# 175. Constraint analysis

Predicates como:

```text
users.id = orders.user_id
```

podrán derivar constraints mediante:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

---

# 176. Semantic graph

Finalmente:

```text
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

representará relaciones semánticas entre:

```text
relations
columns
predicates
parameters
projections
constraints
dependencies
lineage
```

---

# 177. Raw expressions

Ejemplo:

```php
$query->selectRaw(
    'custom_function(?) AS score',
    [$value]
);
```

podrá soportarse.

---

# 178. Raw values still parameterized

Incluso raw:

```text
runtime value
→ Parameter
```

cuando sea posible.

---

# 179. Raw portability

Una RawExpression deberá marcarse:

```text
RAW
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
UNKNOWN
```

según corresponda.

---

# 180. Raw semantic opacity

Semantic Analysis no deberá inventar significado para SQL raw desconocido.

---

# 181. Raw security

Raw API deberá estar sujeta a:

```text
RawSqlTrust
QuerySecurityPolicy
```

---

# 182. Platform-specific extensions

Podrán existir APIs avanzadas:

```text
PostgreSQL-specific query extension
MySQL-specific query extension
```

pero deberán ser explícitas.

---

# 183. Capability-first API

Siempre que sea posible, la API deberá expresar:

```text
semantic capability
```

en vez de:

```text
vendor syntax
```

---

# 184. Example: returning-like SELECT feature

Si una característica semántica puede representarse de forma portable, se modelará semánticamente y el Platform Capability System determinará soporte.

---

# 185. Extension model

Una extensión del SELECT Builder podrá añadir:

```text
projection type
predicate type
relation type
join type
ordering type
window feature
query hint
metadata declaration
```

---

# 186. Extension completeness

Una semantic extension deberá proporcionar soporte necesario en:

```text
Builder
AST
Validation
Semantic Analysis
Capability Analysis
Optimizer
Planner
Compiler
Fingerprinting
Diagnostics
```

según las fases que afecte.

---

# 187. Builder macros

Un macro simple:

```php
$query->whereActive();
```

podrá expandirse a:

```php
$query->where('active', true);
```

sin crear nuevo nodo.

---

# 188. Semantic extensions ≠ macros

Una característica nueva como:

```text
vector similarity
```

puede requerir nuevos nodos y reglas semánticas.

---

# 189. Hints

Podrá existir una API estructurada para:

```text
QueryHint
```

---

# 190. Hint ≠ SQL fragment

Un hint portable o semantic deberá modelarse como metadata/requirement.

Vendor hints deberán marcarse explícitamente platform/dialect-specific.

---

# 191. Security model

La API SELECT será segura por defecto mediante:

```text
automatic value parameterization
structured identifiers
structured operators
structured ordering
explicit raw APIs
binding redaction
capability validation
```

---

# 192. Dynamic column names

Cuando un desarrollador acepte columnas desde input externo:

```php
$query->orderBy($requestColumn);
```

el framework no deberá tratar automáticamente cualquier string arbitrario como seguro.

---

# 193. Application allowlists

Para user-controlled structural input deberá recomendarse:

```text
application-level allowlist
```

por ejemplo:

```php
$column = match ($sort) {
    'name' => 'name',
    'date' => 'created_at',
    default => 'name',
};
```

---

# 194. Parameterization cannot protect identifiers

Regla:

```text
Value
→ parameterizable

Identifier
→ structurally validated/allowlisted
```

---

# 195. Query complexity

Builder podrá llevar contadores preliminares:

```text
joins
predicates
parameters
CTEs
subqueries
projections
```

para safety budgets.

---

# 196. Formal budget enforcement

El procesamiento posterior aplicará:

```text
QueryProcessingBudget
```

según el Query Context System.

---

# 197. Persistent runtime safety

`SelectQueryBuilder` deberá ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
long-running workers
coroutines
```

---

# 198. Shared components

Podrán compartirse:

```text
SelectQueryBuilderFactory
ExpressionFactory
PredicateFactory
IdentifierFactory
frozen descriptors
frozen extension registries
```

---

# 199. Non-shared components

Nunca compartir entre operaciones:

```text
SelectBuilderState
ParameterRegistry
BindingSetBuilder
PredicateGroupBuilder
JoinConditionBuilder
MetadataBuilder
temporary source maps
```

---

# 200. No worker leakage

```text
Request A
   │
   ▼
SelectBuilderState A
   │
   ▼
destroy/finalize

Request B
   │
   ▼
SelectBuilderState B
```

---

# 201. Concurrent queries

```text
Coroutine A
└── SelectBuilder A

Coroutine B
└── SelectBuilder B
```

deberán ser completamente independientes.

---

# 202. No mutable singleton builder

Prohibido:

```php
$container->singleton(
    SelectQueryBuilder::class,
    ...
);
```

---

# 203. Builder factory may be singleton

Sí podrá ser singleton si:

```text
stateless
immutable dependencies
no current query state
```

---

# 204. Determinism

Dados:

```text
same builder operations
same structured inputs
same extension configuration
```

la finalización deberá producir la misma estructura lógica.

---

# 205. Runtime values and determinism

Los valores concretos podrán cambiar el `BindingSet`, pero no deberán alterar innecesariamente la estructura query.

---

# 206. Example

```php
DB::table('users')->where('id', 10);
DB::table('users')->where('id', 20);
```

deberán poder producir la misma:

```text
query shape
```

con diferentes:

```text
BindingSet
```

---

# 207. Query shape

```text
users.id = Parameter(P1)
```

---

# 208. Bindings

Primera ejecución:

```text
P1 = 10
```

Segunda:

```text
P1 = 20
```

---

# 209. Cache benefit

Esto permite reutilizar:

```text
normalized artifacts
semantic artifacts
query plans
compiled query templates
```

cuando sea seguro.

---

# 210. Value-sensitive specialization

Si alguna estrategia depende de:

```text
collection cardinality
binding shape
```

deberá especializarse en Binding Planner/Planner.

No mediante mutación arbitraria del Builder.

---

# 211. Diagnostics

Errores propios del SELECT Builder podrán incluir:

```text
InvalidSelectBuilderStateException
MissingFromRelationException
InvalidProjectionException
InvalidJoinDefinitionException
InvalidGroupingDefinitionException
InvalidWindowDefinitionException
InvalidOrderDefinitionException
InvalidLimitException
InvalidOffsetException
InvalidLockDefinitionException
InvalidCteDefinitionException
InvalidSetOperationDefinitionException
SelectBuilderAlreadyFinalizedException
```

---

# 212. Semantic errors

Ejemplos que NO pertenecen al Builder:

```text
unknown table users
unknown column email
ambiguous id
invalid aggregate grouping
incompatible UNION types
unsupported RIGHT JOIN
invalid correlation
```

---

# 213. Capability errors

Ejemplo:

```text
RIGHT JOIN unsupported by target
```

deberá resolverse después de conocer capabilities efectivas.

---

# 214. Source locations

Builder podrá asociar opcionalmente:

```text
SourceLocation
```

a construcciones para mejorar diagnostics.

---

# 215. Sensitive bindings

Diagnostics deberán mostrar:

```text
Parameter(P1)
```

o tipo/shape seguro.

No necesariamente:

```text
P1 = actual-secret-value
```

---

# 216. Testing sin DB

La mayor parte del SELECT Builder deberá poder probarse sin servidor de base de datos.

---

# 217. Example test

```php
$query = DB::table('users')
    ->select('id', 'name')
    ->where('active', true)
    ->orderBy('name');

$artifact = $query->toQueryArtifact();
```

Assertions conceptuales:

```text
FROM = users

PROJECTION =
    id
    name

WHERE =
    active = P1

ORDER =
    name ASC

P1 shape =
    SCALAR
```

---

# 218. No compiler assertions

Los tests del Builder no deberán comprobar principalmente:

```sql
SELECT id, name
FROM users
WHERE active = ?
ORDER BY name ASC
```

Eso pertenece a tests del SQL Compiler.

---

# 219. Test categories

```text
FROM
projection
wildcard
aliases
expressions
distinct
where
logical predicates
null predicates
IN
BETWEEN
EXISTS
subqueries
joins
grouping
having
windows
ordering
limit
offset
CTEs
set operations
locking
metadata
terminal operations
bindings
copy/fork
raw
extensions
security
persistent runtime
concurrency
architecture
```

---

# 220. Property tests

Podrán comprobar:

```text
finalization determinism
copy isolation
parameter identity stability
no accidental mutation
structural equivalence
safe value separation
```

---

# 221. Architecture tests

`SelectQueryBuilder` no podrá importar:

```text
PDO
PDOStatement
NativeConnection
ConnectionLease
ConnectionPool
Driver implementation
SQL Compiler implementation
Query Executor implementation
EntityManager
UnitOfWork
IdentityMap
HTTP Request
AuthenticatedUser
Tenant Entity
OpenTelemetry Span
```

---

# 222. Namespace recomendado

```text
VoltStack\Quantum\Database\Query\Builder\Select
```

---

# 223. Estructura propuesta

```text
Query/
└── Builder/
    └── Select/
        ├── Contract/
        │   ├── SelectQueryBuilderInterface.php
        │   └── SelectBuilderFinalizerInterface.php
        │
        ├── Core/
        │   ├── SelectQueryBuilder.php
        │   ├── SelectBuilderState.php
        │   ├── SelectBuilderFinalizer.php
        │   └── SelectBuilderSnapshot.php
        │
        ├── Projection/
        │   ├── ProjectionBuilder.php
        │   ├── ProjectionListBuilder.php
        │   └── ProjectionFactory.php
        │
        ├── From/
        │   ├── FromBuilder.php
        │   └── RelationFactory.php
        │
        ├── Join/
        │   ├── JoinBuilder.php
        │   ├── JoinConditionBuilder.php
        │   └── JoinListBuilder.php
        │
        ├── Predicate/
        │   ├── WhereBuilder.php
        │   └── HavingBuilder.php
        │
        ├── Grouping/
        │   └── GroupingBuilder.php
        │
        ├── Window/
        │   ├── WindowBuilder.php
        │   ├── WindowDefinitionBuilder.php
        │   └── WindowFrameBuilder.php
        │
        ├── Ordering/
        │   └── OrderingBuilder.php
        │
        ├── Pagination/
        │   ├── LimitBuilder.php
        │   └── OffsetBuilder.php
        │
        ├── Cte/
        │   ├── CteBuilder.php
        │   └── RecursiveCteBuilder.php
        │
        ├── SetOperation/
        │   └── SetOperationBuilder.php
        │
        ├── Lock/
        │   └── LockBuilder.php
        │
        ├── Terminal/
        │   ├── SelectTerminalOperations.php
        │   ├── AggregateTerminalOperations.php
        │   └── ExistenceTerminalOperations.php
        │
        ├── Extension/
        │   └── SelectBuilderExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── SelectBuilderDiagnostic.php
        │
        └── Exception/
            ├── SelectQueryBuilderException.php
            ├── InvalidProjectionException.php
            ├── InvalidJoinDefinitionException.php
            ├── InvalidGroupingDefinitionException.php
            ├── InvalidWindowDefinitionException.php
            ├── InvalidOrderDefinitionException.php
            ├── InvalidLimitException.php
            ├── InvalidOffsetException.php
            └── SelectBuilderAlreadyFinalizedException.php
```

---

# 224. Relación con otros componentes

```text
SelectQueryBuilder
│
├── Identifier System
├── Expression System
├── Predicate System
├── Parameter System
├── Query Metadata System
├── Query Model
└── Query AST

            │
            ▼

Normalization
Validation
Semantic Analysis
Optimizer
Planner
Compiler
Executor
```

---

# 225. Ownership matrix

| Concepto | Owner |
|---|---|
| Fluent SELECT construction | SelectQueryBuilder |
| Mutable construction state | SelectBuilderState |
| FROM declaration | SelectQueryBuilder / Relation Builder |
| Projection declaration | Projection Builder |
| WHERE construction | Predicate Builder |
| JOIN declaration | Join Builder |
| GROUP BY declaration | Grouping Builder |
| HAVING construction | Predicate Builder |
| Window declaration | Window Builder |
| ORDER declaration | Ordering Builder |
| LIMIT/OFFSET declaration | Select Builder |
| CTE declaration | CTE Builder |
| Set operations | Set Operation Builder |
| Lock declaration | Lock Builder |
| Parameter identity | Parameter System |
| Runtime values | BindingSet |
| Query metadata | Metadata System |
| Immutable SELECT structure | SelectQueryModel / AST |
| Column resolution | Semantic Engine |
| Query types | Type Inference |
| Join semantics | Relation Resolution |
| Constraints | Constraint Analysis |
| Semantic graph | Semantic Graph System |
| Query rewrites | Optimizer |
| Execution strategy | Planner |
| SQL | Compiler |
| Physical execution | Executor |
| Result resources | Result System |

---

# 226. Architectural invariants

## DB-SEL-001

`SelectQueryBuilder` nunca generará SQL.

## DB-SEL-002

Nunca concatenará cláusulas SQL.

## DB-SEL-003

Nunca realizará identifier quoting.

## DB-SEL-004

Nunca asignará native placeholders.

## DB-SEL-005

Nunca realizará native parameter binding.

## DB-SEL-006

Nunca dependerá de PDO.

## DB-SEL-007

Nunca dependerá de una Connection física.

## DB-SEL-008

Nunca ejecutará schema introspection.

## DB-SEL-009

Construir un SELECT no producirá I/O de base de datos.

## DB-SEL-010

Construir un SELECT no adquirirá conexiones.

## DB-SEL-011

El Builder será diferente del SelectQueryModel.

## DB-SEL-012

El SelectQueryModel será diferente del SelectQueryNode.

## DB-SEL-013

La estructura final será immutable.

## DB-SEL-014

Builder state será operation-scoped.

## DB-SEL-015

No existirá current global Select Builder.

## DB-SEL-016

Los valores runtime estarán separados de la estructura.

## DB-SEL-017

Los valores runtime serán parámetros por defecto.

## DB-SEL-018

Los identifiers serán estructurados.

## DB-SEL-019

Los operators serán estructurados.

## DB-SEL-020

Los aliases serán identifiers estructurados.

## DB-SEL-021

Wildcard será un nodo explícito.

## DB-SEL-022

Wildcard no se expandirá en Builder.

## DB-SEL-023

Projection order será preservado.

## DB-SEL-024

FROM relation será estructurada.

## DB-SEL-025

Subquery relations serán artifacts finalizados.

## DB-SEL-026

Child builders mutables no serán retenidos por parent builders.

## DB-SEL-027

WHERE utilizará Predicate System.

## DB-SEL-028

HAVING utilizará Predicate System.

## DB-SEL-029

WHERE y HAVING conservarán contextos diferentes.

## DB-SEL-030

Column comparisons serán distintas de value comparisons.

## DB-SEL-031

IN collections utilizarán CollectionParameter.

## DB-SEL-032

Builder no expandirá CollectionParameter a placeholders.

## DB-SEL-033

Builder no resolverá empty-IN mediante SQL syntax.

## DB-SEL-034

EXISTS almacenará una subquery estructurada.

## DB-SEL-035

Builder no resolverá correlation.

## DB-SEL-036

Pattern predicates preservarán intención semántica.

## DB-SEL-037

Case-insensitive search no se representará directamente como `ILIKE` en core.

## DB-SEL-038

JOIN será una estructura semántica.

## DB-SEL-039

Join kind será explícito.

## DB-SEL-040

Join condition será Predicate AST.

## DB-SEL-041

Builder no inferirá foreign keys.

## DB-SEL-042

Builder no inferirá relation cardinality.

## DB-SEL-043

Builder no calculará semantic join nullability.

## DB-SEL-044

Declared join order será preservado inicialmente.

## DB-SEL-045

Join reordering pertenecerá al Optimizer.

## DB-SEL-046

GROUP BY utilizará expressions estructuradas.

## DB-SEL-047

Builder no realizará aggregate semantic validation.

## DB-SEL-048

Builder no inferirá functional dependencies.

## DB-SEL-049

HAVING no se tratará como WHERE textual.

## DB-SEL-050

Aggregate functions serán expresiones semánticas.

## DB-SEL-051

Window functions serán expresiones semánticas.

## DB-SEL-052

Window specifications serán estructuradas.

## DB-SEL-053

ORDER BY utilizará expressions estructuradas.

## DB-SEL-054

Order direction será enum/value object.

## DB-SEL-055

NULL ordering será semántico.

## DB-SEL-056

Random ordering no contendrá vendor function names en core.

## DB-SEL-057

LIMIT será una especificación estructurada.

## DB-SEL-058

OFFSET será una especificación estructurada.

## DB-SEL-059

Pagination avanzada será un sistema superior.

## DB-SEL-060

CTEs serán artifacts estructurados.

## DB-SEL-061

Builder no resolverá CTE symbols.

## DB-SEL-062

Builder no validará semantic recursive CTE dependencies.

## DB-SEL-063

Set operations conservarán estructura explícita.

## DB-SEL-064

Builder no inferirá common types de set operations.

## DB-SEL-065

Builder no validará set output type compatibility.

## DB-SEL-066

Locking será una semantic specification.

## DB-SEL-067

Builder no generará vendor locking syntax.

## DB-SEL-068

Lock capabilities se resolverán posteriormente.

## DB-SEL-069

SELECT no implicará automáticamente replica eligibility.

## DB-SEL-070

Builder no realizará connection routing.

## DB-SEL-071

Builder no adquirirá replica/primary.

## DB-SEL-072

Terminal operations deberán finalizar la query antes de submission.

## DB-SEL-073

Terminal operations no ejecutarán SQL directamente.

## DB-SEL-074

`first()` no modificará inesperadamente un Builder reutilizable.

## DB-SEL-075

`value()` utilizará structured projection semantics.

## DB-SEL-076

`exists()` expresará existencia semántica.

## DB-SEL-077

Aggregate terminal operations no generarán SQL directamente.

## DB-SEL-078

`cursor()` no abrirá native cursors dentro del Builder.

## DB-SEL-079

`stream()` no será propietario de runtime streams.

## DB-SEL-080

Result resource lifetime pertenecerá al Result/Execution System.

## DB-SEL-081

Builder scopes se expandirán antes del Semantic Engine.

## DB-SEL-082

Framework filters serán visibles estructuralmente.

## DB-SEL-083

Soft-delete predicates no se añadirán directamente durante SQL compilation.

## DB-SEL-084

Tenant predicates no se añadirán directamente durante SQL compilation.

## DB-SEL-085

Authorization predicates no se añadirán directamente durante SQL compilation.

## DB-SEL-086

Policy-generated predicates podrán registrar provenance.

## DB-SEL-087

Builder core no dependerá de Multitenancy.

## DB-SEL-088

Builder core no dependerá de Authorization.

## DB-SEL-089

Builder core no dependerá del ORM.

## DB-SEL-090

ORM podrá consumir/reutilizar el Select Query infrastructure.

## DB-SEL-091

Raw SQL será explícito.

## DB-SEL-092

Raw SQL podrá aceptar safe bindings.

## DB-SEL-093

Raw constructs podrán actuar como semantic barriers.

## DB-SEL-094

Raw constructs tendrán portability classification.

## DB-SEL-095

Raw constructs estarán sujetos a security policy.

## DB-SEL-096

Builder extensions serán registradas de forma controlada.

## DB-SEL-097

Extension registries serán frozen después de bootstrap.

## DB-SEL-098

DX macros serán diferentes de semantic extensions.

## DB-SEL-099

Semantic extensions deberán declarar soporte de fases requerido.

## DB-SEL-100

User-controlled identifiers no serán protegidos mediante value parameterization.

## DB-SEL-101

Structural user input deberá validarse/allowlistarse.

## DB-SEL-102

Builder diagnostics no expondrán sensitive values.

## DB-SEL-103

Select Builder será seguro para persistent workers.

## DB-SEL-104

Select Builder será seguro para concurrent coroutines.

## DB-SEL-105

Builder states concurrentes no compartirán mutable parameter registries.

## DB-SEL-106

Builder copy/fork producirá estados independientes.

## DB-SEL-107

Query shape deberá poder permanecer estable entre diferentes runtime values.

## DB-SEL-108

Runtime binding values no formarán parte del structural query fingerprint.

## DB-SEL-109

Builder no será cache identity autoritativa.

## DB-SEL-110

Canonical identity pertenecerá a artifacts normalizados/semánticos.

## DB-SEL-111

Builder no será responsable de Query Optimization.

## DB-SEL-112

Builder no será responsable de Query Planning.

## DB-SEL-113

Builder no será responsable de SQL Compilation.

## DB-SEL-114

Builder no será responsable de Query Execution.

## DB-SEL-115

Builder no será responsable de Hydration.

## DB-SEL-116

Builder no será responsable de Entity lifecycle.

## DB-SEL-117

Builder no contendrá EntityManager.

## DB-SEL-118

Builder no contendrá UnitOfWork.

## DB-SEL-119

Builder no contendrá IdentityMap.

## DB-SEL-120

Builder no contendrá active Transaction.

## DB-SEL-121

Builder no contendrá ConnectionLease.

## DB-SEL-122

Builder no contendrá native Statement.

## DB-SEL-123

Builder no contendrá HTTP Request.

## DB-SEL-124

Builder no contendrá current User.

## DB-SEL-125

Builder no contendrá Tenant Entity.

## DB-SEL-126

Builder no contendrá active telemetry Span.

## DB-SEL-127

Select Builder deberá poder utilizarse offline.

## DB-SEL-128

Select Builder deberá poder probarse sin base de datos.

## DB-SEL-129

Select Builder deberá mantener Laravel-like DX.

## DB-SEL-130

La ergonomía pública nunca justificará romper las fronteras internas del Query Engine.

---

# 227. Anti-patterns

## 227.1 SQL-based SELECT builder

```php
$sql = 'SELECT ';
$sql .= implode(', ', $columns);
$sql .= ' FROM '.$table;
```

**Rechazado.**

---

## 227.2 Schema introspection during select()

```php
$query->select('*');

// queries database metadata immediately
```

**Rechazado.**

---

## 227.3 Dynamic identifier parameterization

```sql
ORDER BY ?
```

intentando bindear un nombre de columna.

**Rechazado.**

---

## 227.4 Vendor-aware random ordering

```php
if ($driver === 'mysql') {
    $sql .= ' RAND()';
}
```

**Rechazado.**

---

## 227.5 JOIN relationship guessing in Builder

```php
join('orders');

// Builder searches foreign keys automatically
```

como comportamiento implícito del Builder core.

**Rechazado.**

Una API ORM especializada podría solicitar relación por metadata, pero deberá convertirla explícitamente a Query Model antes del Semantic Engine.

---

## 227.6 Hidden tenant SQL

```text
Compiler
→ secretly append tenant_id = ...
```

**Rechazado.**

---

## 227.7 `first()` mutating base builder

```php
$query->first();

// permanently adds limit(1) to reusable $query
```

**Rechazado.**

---

## 227.8 SQL aggregate helpers

```php
public function count()
{
    return $this->selectRaw('COUNT(*)');
}
```

como arquitectura fundamental.

**Rechazado.**

---

## 227.9 Raw aliases

Guardar:

```text
"LOWER(email) AS normalized_email"
```

como projection normal.

**Rechazado.**

---

## 227.10 Global SELECT state

```php
SelectQueryBuilder::$current;
```

**Rechazado.**

---

# 228. Ejemplo completo

Consulta:

```php
$query = DB::table('users as u')
    ->select([
        'u.id',
        'u.name',
        'u.email',
        'order_count' => DB::expr()->count('o.id'),
    ])
    ->leftJoin('orders as o', function ($join) {
        $join
            ->on('o.user_id', '=', 'u.id')
            ->where('o.status', 'paid');
    })
    ->where('u.active', true)
    ->where(function ($where) {
        $where
            ->where('u.country', 'MX')
            ->orWhere('u.country', 'US');
    })
    ->groupBy(
        'u.id',
        'u.name',
        'u.email'
    )
    ->having(
        DB::expr()->count('o.id'),
        '>',
        0
    )
    ->orderByDesc('order_count')
    ->limit(100);
```

---

# 229. Builder representation

Conceptualmente:

```text
SelectBuilderState
│
├── FROM
│   └── users AS u
│
├── PROJECTIONS
│   ├── u.id
│   ├── u.name
│   ├── u.email
│   └── COUNT(o.id) AS order_count
│
├── JOIN
│   └── LEFT orders AS o
│       └── AND
│           ├── o.user_id = u.id
│           └── o.status = P1
│
├── WHERE
│   └── AND
│       ├── u.active = P2
│       └── OR
│           ├── u.country = P3
│           └── u.country = P4
│
├── GROUP BY
│   ├── u.id
│   ├── u.name
│   └── u.email
│
├── HAVING
│   └── COUNT(o.id) > P5
│
├── ORDER BY
│   └── order_count DESC
│
└── LIMIT
    └── 100
```

---

# 230. Bindings

```text
P1 → "paid"
P2 → true
P3 → "MX"
P4 → "US"
P5 → 0
```

---

# 231. Semantic resolution

Posteriormente el Semantic Engine podrá determinar:

```text
u
→ relation users

o
→ relation orders

u.id
→ users.id
→ UserId
→ NON_NULL

o.user_id
→ orders.user_id
→ UserId

LEFT JOIN
→ right-side output becomes nullable

P1
→ OrderStatus

P2
→ Boolean

P3/P4
→ CountryCode

COUNT(o.id)
→ integer/big-integer aggregate result

order_count
→ projection alias

GROUP BY
→ grouping semantics validated
```

---

# 232. Semantic graph

Conceptualmente:

```text
Query
│
├── Relation(users:u)
│   ├── Column(id)
│   ├── Column(name)
│   ├── Column(email)
│   ├── Column(active)
│   └── Column(country)
│
├── Relation(orders:o)
│   ├── Column(id)
│   ├── Column(user_id)
│   └── Column(status)
│
├── Join
│   ├── users:u
│   ├── orders:o
│   └── o.user_id = u.id
│
├── Predicate
│   ├── u.active = P2
│   └── country predicates
│
├── Aggregate
│   └── COUNT(o.id)
│
└── OutputRelation
    ├── id
    ├── name
    ├── email
    └── order_count
```

---

# 233. Planner

Después podrá decidir:

```text
logical operations
join planning
aggregate planning
collection strategies
capability strategies
result strategy
```

---

# 234. Compiler

Sólo entonces aparecerá una representación SQL específica.

```text
Semantic Query
      │
      ▼
Optimized Query
      │
      ▼
Physical Plan
      │
      ▼
Target SQL Compiler
      │
      ▼
CompiledQuery
```

---

# 235. Modelo de integración completo

```text
Developer
   │
   ▼
DB::table()
   │
   ▼
SelectQueryBuilder
   │
   ├── ProjectionBuilder
   ├── ExpressionBuilder
   ├── PredicateBuilder
   ├── JoinBuilder
   ├── GroupingBuilder
   ├── WindowBuilder
   ├── OrderingBuilder
   ├── CteBuilder
   ├── SetOperationBuilder
   └── MetadataBuilder
          │
          ▼
   SelectBuilderFinalizer
          │
          ├───────────────┐
          ▼               ▼
 SelectQueryModel      BindingSet
          │
          ▼
   SelectQueryNode
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
       Result
```

---

# 236. Fórmulas maestras

## Select Query Builder

```text
SelectQueryBuilder
=
FROM
+
Projections
+
Predicates
+
Joins
+
Grouping
+
Windows
+
Ordering
+
Limits
+
CTEs
+
Set Operations
+
Lock Requirements
+
Parameters
+
Metadata
```

---

## Select Query Artifact

```text
SelectQueryArtifact
=
Immutable SelectQueryModel
+
Parameter Definitions
+
QueryMetadata
```

con runtime:

```text
QueryExecutionRequest
=
SelectQueryArtifact
+
BindingSet
+
Execution Requirements
```

---

## SELECT semantics

```text
SELECT Meaning
=
Relations
+
Output Expressions
+
Predicates
+
Grouping
+
Aggregates
+
Windows
+
Ordering
+
Set Operations
+
Semantic Requirements
```

No:

```text
SELECT Meaning
=
SQL String
```

---

## Safe SELECT construction

```text
Safe SELECT
=
Structured Relations
+
Structured Identifiers
+
Structured Expressions
+
Structured Predicates
+
Automatic Parameterization
+
Explicit Raw Escape Hatches
+
Immutable Final Artifact
```

---

# 237. Diseño definitivo

El diseño de `SelectQueryBuilder` combinará:

```text
Laravel-like fluent API
          +
VoltStack Query Model
          +
Structured AST
          +
Semantic Query Engine
          +
Optimizer
          +
Planner
          +
Capability-aware Compiler
```

permitiendo al desarrollador trabajar con:

```php
DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

mientras internamente la consulta pasa por:

```text
Developer Intent
      │
      ▼
SelectQueryBuilder
      │
      ▼
SelectQueryModel
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
Semantic Query Graph
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

# 238. Conclusión

`SelectQueryBuilder` será la API especializada para expresar consultas de lectura dentro de VoltStack.

Su función termina cuando la intención del desarrollador ha sido convertida en una representación estructurada e inmutable.

La frontera definitiva será:

```text
SelectQueryBuilder
        │
        │ builds
        ▼
SelectQueryModel
        │
        ▼
SelectQueryNode
```

Nunca:

```text
SelectQueryBuilder
        │
        X
        ▼
SQL
```

El sistema mantendrá así la ergonomía esperada de un framework moderno sin sacrificar las fronteras arquitectónicas necesarias para:

```text
semantic analysis
query optimization
cross-platform compilation
safe parameter binding
persistent runtimes
offline processing
query caching
ORM reuse
advanced diagnostics
future distributed database capabilities
```

La regla final será:

> **`SelectQueryBuilder` describe qué información desea obtener la aplicación; el resto del Query Engine determina qué significa esa consulta, cómo debe planificarse, cómo debe compilarse y cómo debe ejecutarse.**

---

# 239. Siguiente documento

```text
45_DATABASE_INSERT_QUERY_BUILDER.md
```

El siguiente documento definirá la construcción estructurada de operaciones `INSERT`, incluyendo:

```text
single-row insert
multi-row insert
column mapping
parameterization
bulk values
INSERT ... SELECT
default values
generated columns
identity handling
RETURNING semantics
conflict/upsert interaction
batch construction
parameter shapes
insert result expectations
security
capabilities
persistent-runtime safety
```

manteniendo la misma separación:

```text
InsertQueryBuilder
      │
      ▼
InsertQueryModel
      │
      ▼
InsertQueryNode
      │
      ▼
Semantic Engine
      │
      ▼
Planner
      │
      ▼
Compiler
```

y nunca:

```text
InsertQueryBuilder
      │
      X
      ▼
INSERT SQL String
```