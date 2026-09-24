# 306_DATABASE_QUERY_DEVELOPER_EXPERIENCE.md

# VoltStack Quantum Database
## Database Query Developer Experience

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 306 — Database Query Developer Experience  
**Bloque:** 31 — Developer Experience / Public API  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `305_DATABASE_MODEL_DEVELOPER_EXPERIENCE.md`  
**Siguiente documento:** `307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md`

---

# 1. Propósito

Este documento define la experiencia oficial de desarrollo para construir, inspeccionar, ejecutar y extender consultas dentro de `VoltStack/Quantum/Database`.

La experiencia deberá cubrir:

- Query Builder;
- Entity Query Builder;
- SELECT;
- INSERT;
- UPDATE;
- DELETE;
- predicates;
- expressions;
- joins;
- subqueries;
- Common Table Expressions;
- recursive CTE;
- UNION y set operations;
- grouping;
- aggregations;
- window functions;
- JSON;
- Full Text Search;
- geographic extensions;
- pagination;
- cursor pagination;
- streaming;
- lazy iteration;
- chunk processing;
- parameter binding;
- tipos;
- raw expressions;
- locking;
- timeouts;
- query hints;
- capabilities;
- debugging;
- explainability;
- testing;
- telemetry;
- seguridad;
- extensiones.

La regla fundamental será:

> **El Query Builder de VoltStack será una API para construir una representación semántica tipada de una consulta; nunca será un sistema de concatenación de SQL.**

Por tanto:

```text
Developer API
     ↓
Query Builder
     ↓
Query Model / AST
     ↓
Normalization
     ↓
Semantic Analysis
     ↓
Optimization
     ↓
Planning
     ↓
SQL Compiler
     ↓
Execution Engine
     ↓
Connection
     ↓
Driver
     ↓
DBMS
```

---

# 2. Objetivo

VoltStack deberá permitir que consultas comunes sean sencillas:

```php
$users = DB::table('users')
    ->where('active', true)
    ->get();
```

sin perder la capacidad de expresar consultas complejas:

```php
$report = DB::table('orders', 'o')
    ->join('customers as c', 'c.id', '=', 'o.customer_id')
    ->where('o.status', OrderStatus::PAID)
    ->whereBetween('o.created_at', [$from, $to])
    ->groupBy('c.country')
    ->select([
        'c.country',
        DB::count('o.id')->as('orders'),
        DB::sum('o.total')->as('revenue'),
    ])
    ->having(DB::sum('o.total'), '>', $minimumRevenue)
    ->orderByDesc('revenue')
    ->limit(100)
    ->get();
```

Ambas consultas deberán atravesar la misma arquitectura interna.

---

# 3. Principios fundamentales

Debe mantenerse:

```text
Query Builder
≠
SQL String Builder
```

```text
Query Builder
≠
SQL Compiler
```

```text
Query Builder
≠
Executor
```

```text
Query Builder
≠
Driver
```

```text
Query AST
≠
SQL
```

```text
Query Plan
≠
SQL
```

```text
Compiled Query
≠
Executed Query
```

```text
Parameter
≠
SQL Literal
```

```text
Raw Expression
≠
Normal Query Input
```

---

# 4. Developer Experience Goals

La Query DX deberá priorizar:

1. sintaxis fluida;
2. tipado;
3. autocompletado;
4. seguridad por defecto;
5. parameter binding automático;
6. composabilidad;
7. inmutabilidad cuando resulte conveniente;
8. semántica independiente del vendor;
9. capabilities explícitas;
10. errores descriptivos;
11. debugging;
12. explainability;
13. testabilidad;
14. extensibilidad;
15. rendimiento;
16. streaming;
17. operaciones sobre datasets grandes;
18. compatibilidad con ORM;
19. compatibilidad con persistent runtimes;
20. escape hatches controladas.

---

# 5. Progressive Query API

VoltStack deberá permitir varios niveles.

```text
Level 1
Model Query

User::query()

Level 2
Fluent Query Builder

DB::table('users')

Level 3
Expression API

DB::expr(...)

Level 4
Query AST

QueryNode / ExpressionNode

Level 5
Planner / Compiler APIs

advanced infrastructure/extensions
```

El desarrollador podrá descender de nivel sin abandonar la infraestructura Database.

---

# 6. Public Query Entry Points

Las superficies principales podrán ser:

```php
DB::table('users');
```

```php
db()->table('users');
```

```php
$queryBuilder->table('users');
```

y para ORM:

```php
User::query();
```

Todas deberán converger en el Query Engine.

---

# 7. Facade

`DB` podrá ofrecer:

```php
DB::table('users');
```

pero, conforme a `303_DATABASE_FACADE_SYSTEM.md`:

```text
DB facade
≠
static Database state
```

La facade resolverá servicios desde el scope vigente.

---

# 8. Helper

También podrá existir:

```php
db()->table('users');
```

conforme a:

```text
304_DATABASE_HELPER_SYSTEM.md
```

---

# 9. Same Query Engine Rule

Estas APIs:

```text
DB::table()
db()->table()
User::query()
Repository::query()
```

deberán converger conceptualmente en:

```text
Query Model
    ↓
Query AST
    ↓
Semantic Engine
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Executor
```

---

# 10. Basic SELECT

Ejemplo:

```php
$users = DB::table('users')
    ->select(['id', 'name', 'email'])
    ->get();
```

Representación conceptual:

```text
SelectQuery
├── Source(users)
├── Projection
│   ├── Column(id)
│   ├── Column(name)
│   └── Column(email)
└── ResultShape(rows)
```

No:

```text
"SELECT id, name, email FROM users"
```

en la etapa Builder.

---

# 11. Default Projection

Podrá permitirse:

```php
DB::table('users')->get();
```

como equivalente semántico a:

```text
all visible columns
```

pero la representación interna deberá ser explícita.

---

# 12. `select()`

Ejemplo:

```php
DB::table('users')
    ->select('id', 'name');
```

o:

```php
DB::table('users')
    ->select([
        'id',
        'name',
    ]);
```

---

# 13. `addSelect()`

Podrá existir:

```php
$query
    ->select('id')
    ->addSelect('email');
```

sin necesidad de reconstruir manualmente la proyección.

---

# 14. Alias

Ejemplo:

```php
DB::table('users')
    ->select([
        DB::column('name')->as('user_name'),
    ]);
```

Podrá existir una sintaxis abreviada:

```php
->select('name as user_name')
```

pero deberá ser parseada/validada y no convertirse en una escape hatch arbitraria.

---

# 15. Structured API Preferred

Para infraestructura avanzada deberá preferirse:

```php
DB::column('name')->as('user_name');
```

sobre interpretación extensa de strings.

---

# 16. Table Source

```php
DB::table('users');
```

creará conceptualmente:

```text
TableSource(
    identifier: users
)
```

---

# 17. Table Identifier ≠ Raw SQL

El nombre:

```text
users
```

será tratado como:

```text
Identifier
```

no como valor SQL arbitrario.

---

# 18. Identifier Security

Debe mantenerse:

```text
Identifier
≠
Bound Value
```

Los identifiers deberán validarse mediante el sistema correspondiente.

---

# 19. Dynamic Table Names

Una entrada externa no deberá utilizarse directamente:

```php
DB::table($request->input('table'));
```

sin validación/allow-list apropiada.

---

# 20. Aliases

Podrá utilizarse:

```php
DB::table('users', 'u');
```

o una sintaxis equivalente.

Internamente:

```text
TableSource
├── table: users
└── alias: u
```

---

# 21. WHERE

Ejemplo:

```php
DB::table('users')
    ->where('active', true);
```

deberá crear un predicate semántico:

```text
EqualsPredicate
├── Column(active)
└── Parameter(true)
```

---

# 22. Automatic Parameterization

La API:

```php
->where('email', $email)
```

deberá producir:

```text
Column(email)
=
Parameter(email)
```

no:

```text
email = '$email'
```

---

# 23. Binding

El valor deberá mantenerse separado del SQL compilado.

Conceptualmente:

```text
Query AST
+
Parameter Set
```

---

# 24. WHERE operators

Podrá soportarse:

```php
->where('age', '>=', 18);
```

La operación deberá convertirse a un operator node validado.

---

# 25. Operator ≠ arbitrary SQL

Un operador proveniente de input externo no deberá concatenarse directamente.

---

# 26. Predicate Groups

Ejemplo:

```php
DB::table('users')
    ->where(function ($query) {
        $query
            ->where('status', 'active')
            ->orWhere('status', 'pending');
    })
    ->where('verified', true);
```

AST:

```text
AND
├── OR
│   ├── status = :p1
│   └── status = :p2
└── verified = :p3
```

---

# 27. Logical precedence

La estructura del AST deberá preservar precedencia.

No dependerá únicamente del orden de concatenación textual.

---

# 28. Null predicates

API:

```php
->whereNull('deleted_at');
```

y:

```php
->whereNotNull('email_verified_at');
```

deberá utilizar nodos específicos.

---

# 29. NULL semantics

No deberá compilarse:

```text
column = NULL
```

cuando la semántica requerida sea:

```text
IS NULL
```

---

# 30. `whereIn()`

```php
->whereIn('id', [1, 2, 3]);
```

deberá producir:

```text
InPredicate
├── Column(id)
└── ParameterCollection
```

---

# 31. Empty IN

La semántica de:

```php
->whereIn('id', []);
```

deberá definirse explícitamente.

Una estrategia segura podrá normalizarlo a:

```text
FALSE
```

en lugar de generar SQL inválido.

---

# 32. Empty NOT IN

De manera equivalente:

```php
->whereNotIn('id', []);
```

podrá normalizarse semánticamente a:

```text
TRUE
```

cuando corresponda.

---

# 33. `whereBetween()`

```php
->whereBetween('created_at', [$from, $to]);
```

creará un `BetweenPredicate`.

---

# 34. `whereExists()`

```php
DB::table('users', 'u')
    ->whereExists(function ($query) {
        $query
            ->from('orders', 'o')
            ->select(DB::literal(1))
            ->whereColumn('o.user_id', 'u.id');
    });
```

---

# 35. Column Comparison

`whereColumn()` deberá distinguir:

```text
Column
vs
Column
```

de:

```text
Column
vs
Value
```

---

# 36. Example

```php
->whereColumn('orders.user_id', 'users.id');
```

no deberá parameterizar el segundo identifier como valor.

---

# 37. Expression System

Expresiones avanzadas deberán utilizar:

```text
Database Query Expression System
```

definido previamente.

Ejemplo:

```php
DB::lower('email');
```

```php
DB::count('id');
```

```php
DB::sum('total');
```

---

# 38. Functions

Una función deberá representarse como:

```text
FunctionExpression
├── FunctionIdentity
└── Arguments
```

---

# 39. Function ≠ arbitrary string

No deberá utilizarse:

```php
DB::function($request->input('function'));
```

sin una extensión/registry validada.

---

# 40. ORDER BY

```php
->orderBy('name');
```

```php
->orderBy('name', 'desc');
```

o:

```php
->orderByDesc('created_at');
```

---

# 41. Direction validation

Sólo deberán aceptarse direcciones reconocidas:

```text
ASC
DESC
```

más capacidades futuras explícitas.

---

# 42. User-controlled sort

Un API HTTP:

```text
?sort=name
```

deberá mapearse mediante allow-list.

Ejemplo:

```php
$sorts = [
    'name' => 'users.name',
    'created' => 'users.created_at',
];
```

---

# 43. No identifier parameter binding

Los nombres de columnas no pueden protegerse usando placeholders de valores.

Por tanto:

```text
Dynamic Identifier Security
=
validation / allow-list
```

---

# 44. LIMIT

```php
->limit(100);
```

se representará como metadata/nodo de limit.

---

# 45. OFFSET

```php
->offset(200);
```

deberá ser validado como valor numérico apropiado.

---

# 46. Offset pagination warning

Para datasets grandes:

```text
OFFSET pagination
```

puede degradarse.

VoltStack deberá ofrecer Cursor Pagination como alternativa.

---

# 47. JOIN

Ejemplo:

```php
DB::table('orders', 'o')
    ->join(
        'users as u',
        'u.id',
        '=',
        'o.user_id'
    );
```

deberá producir un `JoinNode`.

---

# 48. JOIN ≠ string concatenation

Representación:

```text
Join
├── type: INNER
├── source: users u
└── predicate:
    u.id = o.user_id
```

---

# 49. Join Builder

Para condiciones complejas:

```php
->join('users as u', function ($join) {
    $join
        ->on('u.id', '=', 'orders.user_id')
        ->where('u.active', true);
});
```

---

# 50. `on()` vs `where()`

Dentro de JOIN:

```text
on()
→ column/expression comparison

where()
→ value predicate
```

deberán mantenerse diferenciados.

---

# 51. Join Types

Podrán soportarse:

```text
INNER
LEFT
RIGHT
FULL
CROSS
LATERAL
```

según capabilities.

---

# 52. Capability-aware joins

Una operación no soportada deberá producir:

```text
UnsupportedCapabilityException
```

o estrategia de emulación explícita.

Nunca SQL accidentalmente inválido.

---

# 53. Subqueries

Ejemplo:

```php
$activeUsers = DB::table('users')
    ->select('id')
    ->where('active', true);

$orders = DB::table('orders')
    ->whereIn('user_id', $activeUsers)
    ->get();
```

---

# 54. Query as expression

Un Query Model podrá ser utilizado como subquery sin compilarlo anticipadamente a string.

---

# 55. No string embedding

Prohibido conceptualmente:

```php
->whereRaw(
    'user_id IN (' . $subquery->toSql() . ')'
);
```

cuando el AST puede representarlo directamente.

---

# 56. CTE

Ejemplo:

```php
$activeUsers = DB::table('users')
    ->select(['id', 'name'])
    ->where('active', true);

$query = DB::table('active_users')
    ->with('active_users', $activeUsers)
    ->get();
```

---

# 57. CTE representation

```text
SelectQuery
├── CTEs
│   └── active_users
│       └── SelectQuery(...)
└── Main Query
```

---

# 58. Recursive CTE

Podrá ofrecerse una API explícita:

```php
->withRecursive(...)
```

cuando la plataforma lo soporte.

---

# 59. Capability Requirement

El Query Model podrá declarar:

```text
Requires:
RECURSIVE_CTE
```

---

# 60. Set Operations

Podrán soportarse:

```php
$queryA->union($queryB);
```

```php
$queryA->unionAll($queryB);
```

```text
INTERSECT
EXCEPT
```

según capabilities.

---

# 61. Set compatibility

El Semantic Engine deberá validar:

```text
projection arity
compatible types
result shape
```

antes de compilación cuando sea posible.

---

# 62. GROUP BY

```php
DB::table('orders')
    ->select([
        'customer_id',
        DB::sum('total')->as('total'),
    ])
    ->groupBy('customer_id')
    ->get();
```

---

# 63. HAVING

```php
->having(
    DB::sum('total'),
    '>',
    1000
);
```

---

# 64. Aggregate functions

Podrán existir helpers:

```php
DB::count()
DB::sum()
DB::avg()
DB::min()
DB::max()
```

que construyan expressions.

---

# 65. Convenience aggregates

También:

```php
DB::table('orders')->count();
```

```php
DB::table('orders')->sum('total');
```

---

# 66. Aggregation ≠ entity hydration

Las agregaciones deberán utilizar scalar/projection hydration apropiada.

---

# 67. Window Functions

Ejemplo conceptual:

```php
DB::rowNumber()
    ->over(
        partitionBy: ['customer_id'],
        orderBy: ['created_at']
    );
```

---

# 68. Window API

Deberá ser estructurada y capability-aware.

---

# 69. Window ≠ raw SQL requirement

La API no deberá obligar al usuario a utilizar `raw()` para funciones comunes.

---

# 70. JSON Query

Ejemplo conceptual:

```php
DB::table('users')
    ->whereJson('preferences', '$.theme', 'dark');
```

La representación deberá utilizar el JSON Query System.

---

# 71. JSON path

JSON paths deberán ser:

```text
validated structured input
```

cuando sea posible.

---

# 72. Vendor independence

El usuario no deberá escribir necesariamente:

```text
JSON_EXTRACT(...)
```

o:

```text
->>
```

para operaciones comunes.

El compiler específico de plataforma resolverá la representación física.

---

# 73. Full Text Search

Ejemplo conceptual:

```php
DB::table('articles')
    ->whereFullText(
        ['title', 'body'],
        $search
    );
```

---

# 74. Full Text capability

Debe consultarse:

```text
Capability System
```

porque las semánticas varían entre plataformas.

---

# 75. Geographic queries

Una extensión podrá ofrecer:

```php
->whereDistance(...)
```

pero deberá construir nodos semánticos del Geographic Data Extension System.

---

# 76. Platform capability model

Debe mantenerse:

```text
Vendor + Version
≠
Capability
```

La query preguntará:

```text
Can this semantic operation be represented?
```

no:

```text
Is this PostgreSQL?
```

---

# 77. Capability Outcomes

Podrán existir:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
REQUIRES_EXTENSION
REQUIRES_EMULATION
UNSUPPORTED
UNKNOWN
```

---

# 78. UNKNOWN

Debe mantenerse:

```text
UNKNOWN
≠
UNSUPPORTED
```

---

# 79. Emulation

Una operación podrá ser:

```text
NATIVE
EMULATED
DEGRADED
NONE
```

si la arquitectura correspondiente lo permite.

---

# 80. Emulation transparency

El desarrollador deberá poder inspeccionar si una operación será emulada.

---

# 81. INSERT

Query API:

```php
DB::table('users')->insert([
    'name' => 'Alice',
    'email' => 'alice@example.com',
]);
```

---

# 82. Multi-row insert

```php
DB::table('users')->insert([
    [
        'name' => 'Alice',
        'email' => 'alice@example.com',
    ],
    [
        'name' => 'Bob',
        'email' => 'bob@example.com',
    ],
]);
```

podrá utilizar Bulk Insert según volumen/policy.

---

# 83. Insert ≠ ORM persist

Debe mantenerse:

```text
Query Insert
≠
EntityManager::persist()
```

---

# 84. ORM coherence

Direct Query mutations podrán dejar entities managed desactualizadas.

La documentación/API deberá hacerlo visible.

---

# 85. RETURNING

Podrá existir:

```php
DB::table('users')
    ->insertReturning(
        [...],
        ['id']
    );
```

o una API composable equivalente.

---

# 86. Returning capability

No deberá asumirse sólo por vendor/version.

Deberá resolverse por capabilities.

---

# 87. UPDATE

```php
DB::table('users')
    ->where('id', 42)
    ->update([
        'active' => false,
    ]);
```

---

# 88. Update assignment

Los valores deberán convertirse en parameters.

---

# 89. Update without WHERE

Una actualización:

```php
DB::table('users')->update([...]);
```

es potencialmente destructiva.

---

# 90. Mutation Safety

VoltStack podrá soportar una policy:

```text
ALLOW_UNBOUNDED
WARN_UNBOUNDED
FORBID_UNBOUNDED
```

---

# 91. Recommended development default

Para operaciones destructivas no acotadas:

```text
WARN
```

o una política configurable.

En contextos críticos podrá utilizarse:

```text
FORBID
```

---

# 92. DELETE

```php
DB::table('users')
    ->where('id', 42)
    ->delete();
```

---

# 93. Unbounded DELETE

```php
DB::table('users')->delete();
```

deberá ser detectable como:

```text
UnboundedMutation
```

---

# 94. Explicit delete-all

Podrá requerirse una API más explícita:

```php
DB::table('users')->deleteAll();
```

para hacer visible la intención.

---

# 95. Truncate

`truncate()` deberá pertenecer a una operación administrativa/schema apropiada y no fingir ser un DELETE normal.

---

# 96. Mutation result

Un UPDATE/DELETE podrá retornar un resultado tipado:

```text
MutationResult
├── affectedRows
├── warnings
└── execution metadata
```

según API.

---

# 97. Affected Rows

Debe tener semántica suficientemente estable para:

```text
optimistic locking
bulk operations
diagnostics
```

---

# 98. Query Execution Terminal Methods

Podrán incluir:

```text
get()
first()
firstOrFail()
value()
values()
exists()
count()
cursor()
stream()
lazy()
paginate()
cursorPaginate()
```

según tipo de query.

---

# 99. Builder vs Result

Antes de un terminal:

```php
$query = DB::table('users')
    ->where('active', true);
```

no deberá ejecutarse I/O.

---

# 100. Lazy construction

Debe cumplirse:

```text
Building Query
≠
Executing Query
```

---

# 101. `get()`

`get()` ejecutará y materializará el resultado según result shape.

---

# 102. `first()`

Podrá introducir semánticamente:

```text
LIMIT 1
```

cuando sea apropiado.

---

# 103. `firstOrFail()`

El Query API genérico deberá utilizar una excepción de resultado/query apropiada, distinta de la excepción ORM cuando no exista entidad.

---

# 104. `value()`

```php
$email = DB::table('users')
    ->where('id', 42)
    ->value('email');
```

deberá utilizar scalar hydration.

---

# 105. `values()`

Podrá retornar una secuencia/lista de escalares.

---

# 106. Result Object

`get()` no deberá necesariamente retornar un array PHP sin semántica.

Podrá retornar:

```text
ResultCollection
RowCollection
Collection<Row>
```

según diseño final.

---

# 107. Row

Un row podrá representarse mediante:

```text
Row
```

tipado/immutable.

---

# 108. Row ≠ Entity

Critical:

```text
Database Row
≠
ORM Entity
```

---

# 109. Associative results

Podrá existir:

```php
$query->get()->toArray();
```

para conveniencia.

---

# 110. Streaming

Para grandes datasets:

```php
foreach (
    DB::table('events')->stream() as $row
) {
    // ...
}
```

---

# 111. Streaming ≠ buffering

La API deberá preservar:

```text
stream()
≠
get()->iterate()
```

---

# 112. Resource Ownership

Streaming deberá gestionar explícitamente:

```text
statement
cursor
connection lease
```

---

# 113. Early termination

Si el desarrollador hace:

```php
foreach ($query->stream() as $row) {
    break;
}
```

los recursos deberán liberarse correctamente.

---

# 114. Lazy Collection

Podrá utilizarse:

```php
$query
    ->lazy()
    ->map(...)
    ->filter(...);
```

según el Lazy Collection System.

---

# 115. Lazy Collection ≠ Query AST

Las operaciones de colección posteriores a la ejecución no deberán fingir que son SQL si ya pertenecen a procesamiento PHP.

---

# 116. Pushdown

VoltStack podrá optimizar operaciones antes de ejecución sólo cuando la semántica lo permita y esté explícitamente representada.

---

# 117. Chunk Processing

```php
DB::table('users')
    ->orderBy('id')
    ->chunk(1000, function ($rows) {
        // ...
    });
```

---

# 118. Chunk ≠ Pagination UI

Chunk es una estrategia de procesamiento bounded.

No una interfaz de navegación para usuario.

---

# 119. Keyset chunking

Para datasets mutables/grandes deberá favorecerse keyset traversal cuando sea apropiado.

---

# 120. Pagination

```php
$page = DB::table('users')
    ->orderBy('id')
    ->paginate(50);
```

---

# 121. Pagination result

Podrá contener:

```text
items
page
perPage
total?
hasNext
hasPrevious
```

según strategy.

---

# 122. Total ≠ always known

El sistema deberá soportar:

```text
EXACT
ESTIMATED
NONE
DEFERRED
CUSTOM
```

para count total.

---

# 123. Cursor Pagination

```php
$page = DB::table('events')
    ->orderBy('created_at')
    ->orderBy('id')
    ->cursorPaginate(100);
```

---

# 124. Stable Ordering

Cursor Pagination deberá validar un orden suficientemente estable.

---

# 125. Tie-breaker

Si:

```text
created_at
```

no es único, deberá añadirse o requerirse:

```text
id
```

u otro tie-breaker estable.

---

# 126. Cursor

Debe mantenerse:

```text
Cursor
≠
Encoded OFFSET
```

---

# 127. Cursor Security

Los cursors externos podrán estar firmados.

---

# 128. Cursor ≠ Snapshot

Un cursor no deberá prometer automáticamente snapshot isolation.

---

# 129. Prepared Statements

La API deberá favorecer:

```text
prepared execution
```

mediante parámetros.

---

# 130. Prepared Reuse

El sistema podrá reutilizar statements cuando:

```text
connection
query shape
parameter metadata
driver
platform
capabilities
```

sean compatibles.

---

# 131. Query fingerprint

Queries semánticamente equivalentes podrán obtener un:

```text
QueryFingerprint
```

para:

```text
compiled query cache
telemetry
profiling
slow query detection
prepared reuse
```

---

# 132. Fingerprint ≠ raw SQL

El fingerprint deberá derivarse de una representación normalizada apropiada.

---

# 133. Query Immutability

VoltStack deberá evaluar builders persistent/immutable para mejorar:

```text
composability
reuse
concurrency safety
predictability
```

---

# 134. Example immutable semantics

```php
$base = DB::table('users')
    ->where('active', true);

$admins = $base->where('role', 'admin');
$customers = $base->where('role', 'customer');
```

Idealmente:

```text
$base
```

no deberá mutarse accidentalmente.

---

# 135. Recommended architecture

El Query Model final deberá ser:

```text
immutable
```

aunque la superficie Builder pueda utilizar una implementación ergonómica mutable durante construcción si su ownership está claramente definido.

---

# 136. Builder cloning

Si el Builder es mutable, deberá ofrecer:

```text
clone semantics
```

claras.

---

# 137. Query Reuse

Un query template podrá reutilizarse con distintos parámetros sin compartir estado mutable inseguro.

---

# 138. Persistent Runtime

Esto será especialmente importante bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 139. No request query state global

Prohibido:

```text
static $currentQuery
static $bindings
static $tenantPredicate
```

---

# 140. Query Context

Cada operación deberá utilizar un:

```text
QueryContext
```

scope-local.

---

# 141. Query Context contents

Podrá incluir:

```text
logical database
tenant
shard intent
read/write intent
consistency requirement
transaction
timeout
cancellation
security policy
telemetry context
```

---

# 142. Query Context ≠ Query AST

El AST describe la consulta.

El QueryContext describe el entorno/intención de ejecución.

---

# 143. Tenant Query Context

Multitenancy podrá contribuir filtros/routing.

Pero:

```text
Query Engine
```

no dependerá obligatoriamente del paquete Multitenancy.

---

# 144. Security Query Policies

Authorization/Data Access podrá contribuir constraints.

Deberán ser inspeccionables.

---

# 145. Hidden security predicate

Si se añade:

```text
tenant_id = :currentTenant
```

deberá existir evidencia diagnóstica de su origen.

---

# 146. Query Explain

Podrá ofrecerse:

```php
$query->explain();
```

pero deberá distinguir al menos:

```text
VoltStack Plan Explain
DBMS Explain
```

---

# 147. Framework Explain

Podrá mostrar:

```text
Query AST
Semantic Graph
Applied Rewrites
Logical Plan
Physical Plan
Capabilities
Routing
Compiler
```

---

# 148. DBMS EXPLAIN

Requiere ejecución/consulta al DBMS.

Por tanto:

```text
Framework Explain
≠
DBMS EXPLAIN
```

---

# 149. Example framework explain

```text
Query
  users
    where active = true
    order by created_at desc
    limit 50

Routing
  logical database: primary
  read intent: READ
  endpoint class: replica eligible

Capabilities
  LIMIT: native
  NULL ordering: platform normalized

Plan
  scan users
  predicate active
  sort created_at desc
  limit 50
```

---

# 150. SQL Inspection

Para debugging podrá existir:

```php
$query->toSql();
```

pero deberá quedar claro:

```text
toSql()
=
compile for a specific platform/context
```

no:

```text
return SQL already stored inside builder
```

---

# 151. Platform required

Si no existe plataforma efectiva, una API genérica podrá retornar:

```text
Query AST
```

pero no necesariamente SQL definitivo.

---

# 152. `toSql()` does not execute

Debe mantenerse:

```text
Compile
≠
Execute
```

---

# 153. Bindings Inspection

Podrá existir:

```php
$query->bindings();
```

en tooling controlado.

---

# 154. Sensitive Bindings

Valores sensibles deberán ser:

```text
redacted
classified
hashed
omitted
```

según política.

---

# 155. Debug SQL

Nunca deberá producir por defecto SQL interpolado con secretos.

---

# 156. Query Telemetry

Cada query podrá emitir:

```text
query fingerprint
operation
duration
rows
database
endpoint role
cache status
plan metadata
```

con cardinalidad controlada.

---

# 157. Raw SQL Telemetry

Raw SQL completo no deberá ser requerido para observabilidad normal.

---

# 158. Slow Query

El Slow Query Detection System podrá correlacionar:

```text
Query Fingerprint
+
duration
+
context
+
result size
+
plan
```

---

# 159. N+1

Entity queries deberán integrarse con N+1 detection.

Query Builder genérico también podrá contribuir correlation metadata.

---

# 160. Raw Expressions

VoltStack deberá ofrecer una escape hatch.

Ejemplo:

```php
DB::raw('some_platform_expression');
```

pero deberá considerarse:

```text
unsafe / trusted expression
```

según contrato.

---

# 161. Raw Expression Rule

Debe mantenerse:

```text
Raw Expression
≠
Parameter Binding
```

---

# 162. User Input

Nunca:

```php
DB::raw($request->input('expression'));
```

---

# 163. Trusted Raw

Una API más explícita podrá ser preferible:

```php
DB::trustedRaw(...)
```

para comunicar riesgo.

---

# 164. Raw Bindings

Si la escape hatch admite parámetros:

```php
DB::raw(
    'some_function(?)',
    [$value]
);
```

los valores deberán continuar separados.

---

# 165. Raw Identifier

No deberá utilizarse raw SQL cuando existe:

```text
Identifier API
```

---

# 166. Raw Expression Auditability

Debug tooling deberá poder indicar:

```text
Query contains raw expression
```

---

# 167. Raw portability

La query deberá marcarse como potencialmente:

```text
platform-specific
```

---

# 168. Raw capability

El sistema no deberá asumir que puede analizar completamente semántica/capabilities de raw SQL.

---

# 169. Query Extensions

Conforme a:

```text
299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM.md
```

podrán registrarse nuevas:

```text
expressions
predicates
operators
functions
AST nodes
planner rules
compiler handlers
```

---

# 170. Typed Extension

Ejemplo conceptual:

```php
$query->whereVectorSimilarity(
    'embedding',
    $vector,
    threshold: 0.85
);
```

deberá producir un nodo registrado.

---

# 171. Extension ≠ macro string

No deberá implementarse simplemente como:

```text
macro
→ concatenate SQL
```

---

# 172. Extension lifecycle

Los registros deberán:

```text
register during bootstrap
validate
freeze
```

---

# 173. Query Hints

Podrán existir hints.

Pero deberán diferenciarse:

```text
semantic requirement
optimization preference
vendor hint
```

---

# 174. Requirement vs Hint

Ejemplo:

```text
must use writer
```

es requirement.

```text
prefer index X
```

es hint.

No deberán modelarse igual.

---

# 175. Vendor-specific hints

Deberán aislarse mediante:

```text
platform extension
```

y no contaminar la API portable principal.

---

# 176. Read/Write Intent

SELECT normalmente:

```text
READ
```

INSERT/UPDATE/DELETE:

```text
WRITE
```

pero locks o funciones con side effects pueden alterar la clasificación.

---

# 177. Routing

La Query API no seleccionará físicamente:

```text
replica-2
```

El Read/Write Routing System lo hará.

---

# 178. Sticky Connections

Después de write, una lectura podrá dirigirse al writer según consistency policy.

La Query API no deberá implementar sticky logic.

---

# 179. Transaction affinity

Dentro de una transacción:

```text
queries
```

deberán permanecer en la conexión/contexto transaccional correspondiente.

---

# 180. Query `useWriteConnection()`

Si existe una convenience API similar deberá expresar:

```text
routing requirement
```

no seleccionar directamente un endpoint.

---

# 181. Consistency Requirement

Una API avanzada podrá permitir:

```php
$query->consistency(ReadConsistency::STRONG);
```

---

# 182. Consistency ≠ endpoint name

El router decidirá cómo satisfacer la requirement.

---

# 183. Sharding

Una query podrá contener:

```text
Shard Routing Evidence
```

derivada de predicates.

Ejemplo:

```php
DB::table('orders')
    ->where('customer_id', $customerId);
```

podría permitir inferir shard.

---

# 184. Query Builder ≠ Shard Router

La resolución corresponde al Partition Routing System.

---

# 185. UNKNOWN shard

No deberá significar automáticamente:

```text
broadcast all shards
```

---

# 186. Cross-shard query

Deberá ser explícita y soportada por una arquitectura específica.

---

# 187. Query Timeout

Podrá expresarse:

```php
$query->timeout(Duration::seconds(2));
```

---

# 188. Timeout requirement

El Execution Engine/Driver resolverá el mecanismo efectivo.

---

# 189. Timeout semantics

Debe distinguirse:

```text
client timeout
statement timeout
network timeout
transaction timeout
```

cuando corresponda.

---

# 190. Cancellation

Una query podrá recibir:

```text
CancellationToken
```

desde QueryContext/ExecutionContext.

---

# 191. Cancellation ≠ known rollback

Cancelar una query no significa necesariamente que toda transacción fue revertida.

---

# 192. Retry

Query Builder no deberá decidir retries.

---

# 193. Query Retry Risk

Una operación aparentemente read puede incluir:

```text
volatile function
lock
side effect
```

por lo que retryability deberá resolverse semánticamente.

---

# 194. Locks

Podrán existir:

```php
->forUpdate();
```

```php
->forShare();
```

---

# 195. Lock Node

Se representará como:

```text
LockRequirement
```

---

# 196. Lock ≠ raw suffix

No:

```text
$query . ' FOR UPDATE'
```

en Builder.

---

# 197. Lock Capability

El Platform Capability System decidirá soporte y variantes.

---

# 198. Transaction Requirement

Una lock query podrá requerir transacción activa según semántica/plataforma.

El sistema deberá detectarlo.

---

# 199. Query Cache

Una query podrá ser candidata al Query/Result Cache.

Pero:

```text
Query Builder
≠
Cache Engine
```

---

# 200. Cache API

Podría existir:

```php
$query->cacheFor(Duration::minutes(5));
```

como policy/hint.

---

# 201. TTL ≠ correctness

Debe mantenerse:

```text
TTL
≠
Cache Consistency
```

---

# 202. Transaction Cache Rule

No deberá publicarse como shared cache un resultado dependiente de datos no confirmados.

---

# 203. Compiled Query Cache

Será interno al pipeline.

El desarrollador normalmente no necesitará controlarlo.

---

# 204. Query Clone/Reusability

Ejemplo:

```php
$base = DB::table('orders')
    ->where('status', 'paid');

$today = $base->whereBetween('created_at', [$start, $end]);
```

La semántica de reuse deberá ser predecible.

---

# 205. Query Template

VoltStack podrá evolucionar hacia:

```text
Compiled Query Template
+
runtime parameters
```

para workloads repetitivos.

---

# 206. Prepared Query API

Podrá existir una API avanzada:

```php
$query = DB::prepare(
    DB::table('users')
        ->where('id', DB::parameter('id'))
);
```

y:

```php
$user = $query->execute([
    'id' => 42,
]);
```

si demuestra beneficios reales.

---

# 207. Named Parameters

Internamente los parámetros deberán poseer identidad estable.

---

# 208. Duplicate parameters

El compiler decidirá si:

```text
same logical parameter
```

puede reutilizar placeholder o necesita múltiples placeholders según driver.

---

# 209. Type inference

El Query Type System podrá inferir:

```text
parameter type
expression type
result type
```

cuando exista suficiente metadata.

---

# 210. Explicit type

Podrá permitirse:

```php
->where(
    'id',
    '=',
    DB::param($id, type: Types::UUID)
);
```

para casos ambiguos.

---

# 211. Type inference uncertainty

Debe mantenerse:

```text
UNKNOWN TYPE
≠
STRING
```

---

# 212. Schema-aware Query

Cuando exista Schema Metadata:

```text
users.id
```

podrá resolverse a su tipo lógico.

---

# 213. ORM-aware Query

En `User::query()`:

```text
User::$id
```

podrá resolverse mediante Entity Metadata.

---

# 214. Semantic Validation

Antes de compilación deberán detectarse, cuando sea posible:

```text
unknown table
unknown column
ambiguous column
invalid aggregate
invalid grouping
invalid set operation
type mismatch
unsupported capability
invalid relation
invalid expression
```

---

# 215. Schema Metadata Unavailable

Si no existe metadata suficiente:

```text
unknown
```

no deberá fingirse como:

```text
valid
```

---

# 216. Validation Confidence

Podrá existir:

```text
COMPLETE
PARTIAL
UNKNOWN
```

respecto a validación schema-aware.

---

# 217. Query Normalization

Dos expresiones equivalentes podrán normalizarse.

Ejemplo:

```text
where active = true
```

podrá adoptar una representación canónica.

---

# 218. Normalization ≠ optimization

Normalizar estructura no implica cambiar estrategia de ejecución.

---

# 219. Query Rewrite

Optimizer podrá realizar rewrites semánticamente equivalentes.

---

# 220. Developer visibility

En debug:

```text
Original Query Model
Normalized Query
Optimized Query
```

podrán inspeccionarse.

---

# 221. Optimizer transparency

Un rewrite deberá poder explicar:

```text
Rule:
RedundantPredicateElimination

Before:
A AND TRUE

After:
A
```

---

# 222. Query Planner

Planner decidirá:

```text
logical plan
physical plan
execution requirements
```

---

# 223. Builder does not choose physical plan

Por ejemplo:

```php
->with(...)
```

en ORM no obliga a JOIN físico.

---

# 224. SQL Compiler

Sólo después de planificación deberá generarse SQL específico.

---

# 225. MySQL

```text
Query AST
→ MySQL Compiler
```

---

# 226. MariaDB

```text
Query AST
→ MariaDB Compiler
```

independientemente de MySQL.

---

# 227. PostgreSQL

```text
Query AST
→ PostgreSQL Compiler
```

---

# 228. SQLite

```text
Query AST
→ SQLite Compiler
```

---

# 229. MySQL ≠ MariaDB

No deberá asumirse equivalencia completa.

---

# 230. SQL Dialect ≠ Driver

Debe mantenerse:

```text
Dialect
≠
Driver
```

---

# 231. Query Execution Result

La ejecución deberá producir un resultado tipado que preserve:

```text
rows
affected rows
generated values
cursor
warnings
execution metadata
```

según operación.

---

# 232. Error Taxonomy

La Query DX deberá presentar errores como:

```text
QueryConstructionException
QueryValidationException
UnknownColumnException
AmbiguousColumnException
UnsupportedQueryCapabilityException
QueryCompilationException
QueryExecutionException
QueryTimeoutException
QueryCancelledException
ConstraintViolationException
```

según la jerarquía definitiva.

---

# 233. Error Layer

El error deberá indicar la fase:

```text
BUILD
NORMALIZE
SEMANTIC_ANALYSIS
OPTIMIZE
PLAN
COMPILE
EXECUTE
FETCH
```

---

# 234. Example error

```text
Cannot compile query.

Operation:
FULL OUTER JOIN

Platform:
MySQL

Capability:
FULL_OUTER_JOIN = UNSUPPORTED

Query:
UserActivityReport

Suggested actions:
- rewrite the query using supported operations;
- provide an explicit emulation strategy;
- use a platform extension if available.
```

---

# 235. Unknown Column

Ejemplo:

```text
Unknown column `emial` in query source `users`.

Did you mean:
- email
```

sólo cuando schema metadata proporcione evidencia suficiente.

---

# 236. Ambiguous Column

Ejemplo:

```text
Column `id` is ambiguous.

Available sources:
- users.id
- orders.id

Use a qualified column reference.
```

---

# 237. Parameter Error

Ejemplo:

```text
Parameter `customer_id` expects UUID.

Received:
integer
```

---

# 238. Query Testing

El Query Assertion System deberá permitir:

```php
assertQuery($query)
    ->selects('id', 'email')
    ->from('users')
    ->whereEquals('active', true);
```

---

# 239. Semantic Assertions

Preferible:

```text
assert AST semantics
```

sobre:

```text
assert exact SQL string
```

para tests portables.

---

# 240. Compiler Testing

SQL exacto sí deberá comprobarse en:

```text
platform compiler tests
```

---

# 241. Execution Testing

Comportamiento real del DBMS deberá comprobarse con:

```text
integration tests
driver conformance tests
```

---

# 242. Mock limitation

Debe mantenerse:

```text
Mock Executor passing
≠
DBMS query works
```

---

# 243. Performance Testing

Benchmarks podrán medir por separado:

```text
builder
normalization
semantic analysis
optimizer
planner
compiler
execution
fetch
hydration
```

---

# 244. Query construction benchmark

No deberá presentarse como:

```text
database query performance
```

si sólo mide el Builder.

---

# 245. Debug Toolbar

Podrá mostrar:

```text
Query count
Total query time
Slowest query
Query fingerprints
Database
Endpoint role
Cache
N+1
Transaction
```

---

# 246. Query Timeline

Ejemplo:

```text
Request
│
├── Q1 User lookup       1.8 ms
├── Q2 Orders           12.4 ms
├── Q3 Preferences       0.7 ms
└── Q4 Audit             3.1 ms
```

---

# 247. Sensitive data

Bindings deberán estar redacted según clasificación.

---

# 248. Query Source Location

En desarrollo podrá capturarse:

```text
file
line
framework operation
```

de manera opcional.

---

# 249. Stack capture cost

No deberá habilitarse indiscriminadamente en producción debido a overhead.

---

# 250. Query naming

El desarrollador podrá opcionalmente nombrar queries:

```php
$query->name('dashboard.active-users');
```

para telemetry/debugging.

---

# 251. Query name ≠ cache key

El nombre humano no deberá utilizarse como única identidad semántica.

---

# 252. Query Tags

Podrán existir tags controlados:

```text
feature=dashboard
operation=user-search
```

con cardinalidad gobernada.

---

# 253. SQL comments

Si se utilizan comentarios SQL para tracing, deberán ser generados por infraestructura segura.

No desde input arbitrario.

---

# 254. Search APIs

Un filtro dinámico típico:

```php
$query = User::query();

if ($filters->name !== null) {
    $query = $query->where('name', 'like', $filters->name);
}

if ($filters->active !== null) {
    $query = $query->where('active', $filters->active);
}
```

deberá ser fácil de construir.

---

# 255. Conditional Builder

Podrá ofrecerse:

```php
$query->when(
    $filters->active !== null,
    fn ($query) =>
        $query->where('active', $filters->active)
);
```

como convenience.

---

# 256. `when()` ≠ query semantics

Es sólo una ayuda de construcción.

No necesita convertirse en AST.

---

# 257. Conditional security

No deberá utilizarse `when()` para ocultar requisitos de seguridad obligatorios.

---

# 258. Dynamic Filter System

Filtros HTTP dinámicos deberán mapearse explícitamente a:

```text
allowed fields
allowed operators
expected types
```

---

# 259. Example filter map

```php
$filters = QueryFilter::for(User::class)
    ->allow('name', Operators::LIKE)
    ->allow('status', Operators::EQUALS)
    ->allow('createdAt', Operators::BETWEEN);
```

---

# 260. HTTP Query Params ≠ Query AST

No deberá convertirse automáticamente cualquier:

```text
?filter[field][operator]=...
```

en Query AST sin policy.

---

# 261. Injection prevention

Debe existir defensa en profundidad:

```text
Values
→ Parameters

Identifiers
→ Validation

Operators
→ Registry/Enum

Raw SQL
→ Explicit trusted escape hatch
```

---

# 262. Query Security Formula

```text
SafeQueryConstruction
=
ParameterizedValues
∧
ValidatedIdentifiers
∧
ValidatedOperators
∧
ControlledRawExpressions
```

---

# 263. Query Developer Experience Formula

Conceptualmente:

```text
QueryDX
=
Ergonomics
+
TypeSafety
+
Composability
+
Security
+
Explainability
+
Portability
+
PerformanceAwareness
```

---

# 264. Query Correctness Formula

```text
ValidQuery
=
StructurallyValid
∧
SemanticallyValid
∧
CapabilityCompatible
∧
ContextCompatible
```

cuando existe evidencia suficiente.

---

# 265. Execution Eligibility

```text
Executable
=
ValidQuery
∧
ValidContext
∧
ResolvedPlatform
∧
ResolvedCapabilities
∧
ResolvedRouting
```

---

# 266. No hidden compilation

Construir:

```php
$query = DB::table('users');
```

no deberá compilar inmediatamente.

---

# 267. No hidden execution

Llamar:

```php
$query->where(...)
```

no deberá abrir una conexión.

---

# 268. Connection Acquisition

La conexión deberá adquirirse lo más tarde posible dentro del execution pipeline.

---

# 269. Why late acquisition

Reduce:

```text
connection hold time
pool pressure
resource contention
```

---

# 270. Query Construction without Database

Debe poder realizarse:

```php
$query = DBQuery::table('users')
    ->where('active', true);
```

en tests puros cuando se use una factory/builder independiente del runtime.

---

# 271. Builder service vs Facade

La API directa deberá estar disponible para:

```text
libraries
unit tests
infrastructure
```

sin requerir facade.

---

# 272. Query API for packages

Paquetes VoltStack deberán preferir inyección explícita cuando corresponda.

---

# 273. Facade is application convenience

No deberá convertirse en dependencia obligatoria de los componentes internos.

---

# 274. Recommended V1 API

Consulta básica:

```php
$users = DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

---

# 275. Typed expression

```php
$users = DB::table('users')
    ->select([
        'id',
        DB::lower('email')->as('normalized_email'),
    ])
    ->get();
```

---

# 276. Join

```php
$orders = DB::table('orders', 'o')
    ->join('users as u', function ($join) {
        $join->on('u.id', '=', 'o.user_id');
    })
    ->select([
        'o.id',
        'u.name',
        'o.total',
    ])
    ->get();
```

---

# 277. Aggregation

```php
$totals = DB::table('orders')
    ->select([
        'customer_id',
        DB::sum('total')->as('total'),
    ])
    ->groupBy('customer_id')
    ->get();
```

---

# 278. Transaction

```php
DB::transaction(function () {
    DB::table('accounts')
        ->where('id', 1)
        ->update([
            'balance' => DB::column('balance')->minus(100),
        ]);

    DB::table('accounts')
        ->where('id', 2)
        ->update([
            'balance' => DB::column('balance')->plus(100),
        ]);
});
```

El Query Builder no será propietario de la transacción.

---

# 279. Streaming

```php
foreach (
    DB::table('events')
        ->orderBy('id')
        ->stream() as $event
) {
    // ...
}
```

---

# 280. Cursor Pagination

```php
$page = DB::table('events')
    ->orderBy('created_at')
    ->orderBy('id')
    ->cursorPaginate(100);
```

---

# 281. Entity Query

```php
$users = User::query()
    ->where('active', true)
    ->with('company')
    ->get();
```

deberá usar la misma infraestructura Query.

---

# 282. Query Architecture

```text
Application
    │
    ├──────────────┬───────────────┐
    ▼              ▼               ▼
DB Facade       db() Helper    Model/Repository
    │              │               │
    └──────────────┼───────────────┘
                   ▼
             Query Builder
                   │
                   ▼
              Query Model
                   │
                   ▼
                AST
                   │
                   ▼
          Query Normalizer
                   │
                   ▼
          Semantic Analyzer
                   │
                   ▼
            Semantic Graph
                   │
                   ▼
              Optimizer
                   │
                   ▼
               Planner
                   │
                   ▼
          Execution Plan
                   │
                   ▼
              Compiler
                   │
                   ▼
           Compiled Query
                   │
                   ▼
              Executor
                   │
                   ▼
          Connection Manager
                   │
                   ▼
               Driver
                   │
                   ▼
                 DBMS
```

---

# 283. Query Construction Invariants

## DB-QDX-001

Query Builder ≠ SQL String Builder.

## DB-QDX-002

Query Builder ≠ Compiler.

## DB-QDX-003

Query Builder ≠ Executor.

## DB-QDX-004

Query Builder ≠ Driver.

## DB-QDX-005

Building Query ≠ Executing Query.

## DB-QDX-006

Building Query no requerirá conexión.

## DB-QDX-007

Query Model será independiente del SQL físico.

## DB-QDX-008

Query AST ≠ SQL.

## DB-QDX-009

Query Plan ≠ SQL.

## DB-QDX-010

Compiled Query ≠ Executed Query.

---

# 284. Parameter Invariants

## DB-QDX-011

Value ≠ Identifier.

## DB-QDX-012

Values serán parameterized por defecto.

## DB-QDX-013

Identifiers serán validados.

## DB-QDX-014

Operators serán validados.

## DB-QDX-015

Parameter ≠ SQL Literal.

## DB-QDX-016

NULL tendrá semántica explícita.

## DB-QDX-017

Parameter type podrá inferirse o declararse.

## DB-QDX-018

UNKNOWN type ≠ string.

## DB-QDX-019

Sensitive bindings serán redactables.

## DB-QDX-020

Raw expressions no recibirán input no confiable.

---

# 285. Semantic Invariants

## DB-QDX-021

Semantic Analysis ocurrirá antes de compilación cuando sea posible.

## DB-QDX-022

Unknown schema evidence ≠ valid schema evidence.

## DB-QDX-023

Unknown column podrá detectarse con metadata suficiente.

## DB-QDX-024

Ambiguous column deberá rechazarse.

## DB-QDX-025

Set operations validarán shape compatible.

## DB-QDX-026

Aggregate semantics serán validadas.

## DB-QDX-027

JOIN será un nodo semántico.

## DB-QDX-028

Subquery será Query Model, no SQL string.

## DB-QDX-029

CTE será parte del Query Model.

## DB-QDX-030

Window function será expresión semántica.

---

# 286. Capability Invariants

## DB-QDX-031

Vendor + Version ≠ Capability.

## DB-QDX-032

UNKNOWN ≠ UNSUPPORTED.

## DB-QDX-033

Planner consultará capabilities.

## DB-QDX-034

Builder no hará branching por vendor.

## DB-QDX-035

Compiler manejará representación específica.

## DB-QDX-036

Emulation será explícita.

## DB-QDX-037

Unsupported operation fallará antes de SQL cuando sea posible.

## DB-QDX-038

MySQL ≠ MariaDB.

## DB-QDX-039

Dialect ≠ Driver.

## DB-QDX-040

Capability snapshot tendrá scope apropiado.

---

# 287. Execution Invariants

## DB-QDX-041

Execution Engine será responsable de ejecución.

## DB-QDX-042

Compiler no ejecutará queries.

## DB-QDX-043

Connection no interpretará AST.

## DB-QDX-044

Driver no interpretará ORM entities.

## DB-QDX-045

Connection acquisition será tardía cuando sea posible.

## DB-QDX-046

Transaction affinity será preservada.

## DB-QDX-047

Query timeout ≠ retry.

## DB-QDX-048

Cancellation ≠ rollback guarantee.

## DB-QDX-049

Execution failure conservará outcome uncertainty.

## DB-QDX-050

Statement success ≠ transaction commit.

---

# 288. Result Invariants

## DB-QDX-051

Row ≠ Entity.

## DB-QDX-052

Result ≠ array necesariamente.

## DB-QDX-053

Cursor ≠ Collection.

## DB-QDX-054

Streaming ≠ Buffering.

## DB-QDX-055

Early stream termination liberará recursos.

## DB-QDX-056

Scalar query no hidratará entities.

## DB-QDX-057

Aggregation no requerirá entity hydration.

## DB-QDX-058

Affected rows conservará semántica definida.

## DB-QDX-059

Generated value no será asumido integer.

## DB-QDX-060

Result shape será explícito.

---

# 289. Pagination Invariants

## DB-QDX-061

Pagination ≠ Cursor Pagination.

## DB-QDX-062

Cursor ≠ encoded offset.

## DB-QDX-063

Cursor ≠ snapshot.

## DB-QDX-064

Cursor Pagination requerirá orden estable.

## DB-QDX-065

Tie-breaker será requerido cuando sea necesario.

## DB-QDX-066

Total count podrá ser desconocido.

## DB-QDX-067

Chunk ≠ Pagination UI.

## DB-QDX-068

Chunk podrá usar keyset traversal.

## DB-QDX-069

Large dataset APIs deberán ser bounded.

## DB-QDX-070

`get()` no será recomendado para datasets ilimitados.

---

# 290. Security Invariants

## DB-QDX-071

Query API usará parameterization por defecto.

## DB-QDX-072

Identifier input externo requerirá validación.

## DB-QDX-073

Operator input externo requerirá validación.

## DB-QDX-074

Raw SQL será escape hatch explícita.

## DB-QDX-075

Raw SQL no bypassará automáticamente bindings.

## DB-QDX-076

Tenant security policies no serán convenience filters.

## DB-QDX-077

Security policies no podrán eliminarse mediante bypass genérico.

## DB-QDX-078

Query debugging no expondrá secretos.

## DB-QDX-079

SQL comments no contendrán input arbitrario.

## DB-QDX-080

Dynamic filtering utilizará allow-lists/policies.

---

# 291. Runtime Invariants

## DB-QDX-081

QueryContext será scope-local.

## DB-QDX-082

Bindings no serán process-global.

## DB-QDX-083

Tenant query state no será process-global.

## DB-QDX-084

Transaction query state no será process-global.

## DB-QDX-085

FrankenPHP será soportado desde V1.

## DB-QDX-086

Request A query state no aparecerá en Request B.

## DB-QDX-087

RoadRunner podrá usar la misma semántica.

## DB-QDX-088

OpenSwoole requerirá coroutine isolation.

## DB-QDX-089

Immutable compiled artifacts podrán compartirse.

## DB-QDX-090

Mutable execution state no será global.

---

# 292. Debugging Invariants

## DB-QDX-091

Query podrá inspeccionarse antes de ejecución.

## DB-QDX-092

AST podrá inspeccionarse.

## DB-QDX-093

Plan podrá inspeccionarse.

## DB-QDX-094

Compiled SQL podrá inspeccionarse.

## DB-QDX-095

Compilation ≠ execution.

## DB-QDX-096

Framework Explain ≠ DBMS EXPLAIN.

## DB-QDX-097

Applied rewrites deberán poder diagnosticarse.

## DB-QDX-098

Applied security policies deberán poder identificarse de forma segura.

## DB-QDX-099

Query fingerprint ≠ raw SQL.

## DB-QDX-100

Debug data estará sujeto a redaction.

---

# 293. ORM Integration Invariants

## DB-QDX-101

Entity Query utilizará Query Engine.

## DB-QDX-102

ORM no generará SQL.

## DB-QDX-103

Model Query ≠ independent Query Engine.

## DB-QDX-104

Repository Query ≠ independent Query Engine.

## DB-QDX-105

Entity property podrá mapear a physical column.

## DB-QDX-106

Entity property ≠ physical column.

## DB-QDX-107

Projection será preferible a partial entity cuando corresponda.

## DB-QDX-108

Eager loading ≠ JOIN.

## DB-QDX-109

N+1 detection podrá usar query semantic metadata.

## DB-QDX-110

Bulk Query mutation podrá invalidar ORM state.

---

# 294. Extensibility Invariants

## DB-QDX-111

Query extension será registrada explícitamente.

## DB-QDX-112

Extension registry se congelará tras bootstrap.

## DB-QDX-113

Custom query extension no concatenará SQL como arquitectura principal.

## DB-QDX-114

Custom AST nodes tendrán semantic contracts.

## DB-QDX-115

Custom compiler handlers serán platform-aware.

## DB-QDX-116

Extension podrá declarar capabilities.

## DB-QDX-117

Extension failure será diagnosticable.

## DB-QDX-118

Extension no podrá mutar core registries arbitrariamente en request runtime.

## DB-QDX-119

Plugin query semantics serán versionables.

## DB-QDX-120

Unknown custom node no será ignorado silenciosamente.

---

# 295. Testing Invariants

## DB-QDX-121

Query semantics podrán probarse sin DB.

## DB-QDX-122

SQL compiler se probará por plataforma.

## DB-QDX-123

DBMS behavior requerirá integración real.

## DB-QDX-124

Mock passing ≠ real DBMS compatibility.

## DB-QDX-125

Query assertions favorecerán semantic assertions.

## DB-QDX-126

Driver conformance probará execution boundary.

## DB-QDX-127

Performance tests separarán pipeline phases.

## DB-QDX-128

Query security tendrá tests específicos.

## DB-QDX-129

Raw expression behavior tendrá tests.

## DB-QDX-130

Persistent runtime leakage tendrá tests.

---

# 296. Performance Invariants

## DB-QDX-131

Query construction no abrirá conexión innecesariamente.

## DB-QDX-132

Compilation podrá cachearse cuando sea seguro.

## DB-QDX-133

Metadata podrá compilarse.

## DB-QDX-134

Query fingerprint permitirá reuse apropiado.

## DB-QDX-135

Streaming evitará buffering completo.

## DB-QDX-136

Chunk processing será bounded.

## DB-QDX-137

Prepared statement reuse será capability/context-aware.

## DB-QDX-138

Debug instrumentation podrá desactivarse/reducirse en producción.

## DB-QDX-139

Optimization preservará semántica.

## DB-QDX-140

Performance optimization no bypassará security.

---

# 297. Architectural Invariants

## DB-QDX-141

Builder → AST.

## DB-QDX-142

AST → Semantic Engine.

## DB-QDX-143

Semantic Engine → Optimizer.

## DB-QDX-144

Optimizer → Planner.

## DB-QDX-145

Planner → Compiler.

## DB-QDX-146

Compiler → Executor.

## DB-QDX-147

Executor → Connection.

## DB-QDX-148

Connection → Driver.

## DB-QDX-149

Las dependencias no se invertirán arbitrariamente.

## DB-QDX-150

Query DX preservará todas las invariantes Database.

---

# 298. Anti-patrones

## 298.1 SQL concatenation Builder

```php
$sql .= ' WHERE ' . $column . ' = ' . $value;
```

Prohibido.

---

## 298.2 Interpolated values

```php
DB::raw("email = '$email'");
```

Prohibido con input dinámico.

---

## 298.3 User-controlled identifiers

```php
DB::table($_GET['table']);
```

sin allow-list.

Prohibido.

---

## 298.4 User-controlled operators

```php
->where('price', $_GET['operator'], $value);
```

sin validación.

Prohibido.

---

## 298.5 Query Builder talking directly to PDO

Prohibido.

---

## 298.6 Query Builder compiling SQL

Prohibido.

---

## 298.7 Compiler executing query

Prohibido.

---

## 298.8 Driver understanding Query AST

Prohibido.

---

## 298.9 Vendor branching everywhere

```php
if ($db === 'mysql') {
    // ...
}
```

Evitar.

---

## 298.10 MySQL == MariaDB assumption

Prohibido como regla arquitectónica.

---

## 298.11 Raw SQL for every advanced feature

Evitar.

---

## 298.12 Subquery via `toSql()` concatenation

Evitar.

---

## 298.13 Offset for every large dataset

Evitar.

---

## 298.14 Buffering millions of rows

Evitar.

---

## 298.15 Hidden DB I/O during Builder construction

Prohibido.

---

## 298.16 Hidden query execution in debug string conversion

Prohibido.

---

## 298.17 Query retries inside Builder

Prohibido.

---

## 298.18 Static bindings

Prohibido.

---

## 298.19 Static current query

Prohibido.

---

## 298.20 Automatic cross-shard broadcast

Prohibido por defecto.

---

## 298.21 UNKNOWN capability treated as unsupported

Prohibido.

---

## 298.22 Security policy as removable cosmetic scope

Prohibido.

---

## 298.23 Exact SQL assertions everywhere

Evitar.

---

## 298.24 SQLite as universal proof

Prohibido.

---

## 298.25 Average-only performance analysis

Evitar.

---

## 298.26 Raw SQL logging with credentials

Prohibido.

---

## 298.27 N+1 as acceptable default

Evitar.

---

## 298.28 Query cache treated as database truth

Prohibido.

---

## 298.29 `toSql()` treated as stored query

Prohibido.

---

## 298.30 Extension through arbitrary runtime macros

Evitar.

---

# 299. Recommended Namespace

Una estructura conceptual podrá ser:

```text
src/Quantum/Database/
├── Query/
│   ├── Builder/
│   │   ├── QueryBuilder.php
│   │   ├── SelectQueryBuilder.php
│   │   ├── InsertQueryBuilder.php
│   │   ├── UpdateQueryBuilder.php
│   │   ├── DeleteQueryBuilder.php
│   │   └── JoinBuilder.php
│   │
│   ├── Model/
│   ├── AST/
│   ├── Expression/
│   ├── Predicate/
│   ├── Parameter/
│   ├── Type/
│   ├── Metadata/
│   ├── Context/
│   ├── Normalization/
│   ├── Semantic/
│   ├── Optimization/
│   ├── Planning/
│   ├── Compilation/
│   ├── Execution/
│   ├── Result/
│   ├── Pagination/
│   ├── Streaming/
│   ├── Extension/
│   └── Diagnostics/
```

---

# 300. Public API Surface

El objetivo será mantener una superficie pequeña:

```text
DB
db()
QueryBuilder
Expression
Result
Paginator
CursorPaginator
```

mientras la complejidad interna permanezca encapsulada.

---

# 301. Developer Mental Model

Para el desarrollador:

```text
Build
→ Execute
→ Receive Result
```

Para VoltStack:

```text
Build
→ AST
→ Normalize
→ Analyze
→ Optimize
→ Plan
→ Compile
→ Bind
→ Route
→ Execute
→ Fetch
→ Convert
→ Return
```

Esta diferencia es deliberada.

---

# 302. Architectural Philosophy

VoltStack no deberá exigir que un desarrollador entienda todo el pipeline para escribir:

```php
DB::table('users')->get();
```

Pero el framework deberá conservar ese pipeline para poder proporcionar:

```text
portability
security
optimization
testing
telemetry
capabilities
extensions
persistent runtime safety
```

---

# 303. Acceptance Criteria

El Query Developer Experience estará preparado para V1 cuando pueda demostrarse que:

1. existe una API Query fluida;
2. existe una API Query inyectable sin Facade;
3. `DB::table()` no genera SQL directamente;
4. Builder produce Query Model/AST;
5. valores se parameterizan por defecto;
6. identifiers se validan;
7. operators se validan;
8. raw expressions son explícitas;
9. SELECT está soportado;
10. INSERT está soportado;
11. UPDATE está soportado;
12. DELETE está soportado;
13. JOIN está soportado;
14. subqueries están soportadas;
15. CTE tiene representación semántica;
16. aggregations tienen representación semántica;
17. window functions son capability-aware;
18. JSON query es capability-aware;
19. Full Text es capability-aware;
20. Model Query usa el mismo Query Engine;
21. Semantic Analysis precede compilación cuando sea posible;
22. Optimizer puede reescribir sin cambiar semántica;
23. Planner es independiente del Builder;
24. Compiler es platform-specific;
25. MySQL y MariaDB son plataformas distintas;
26. Compiler no ejecuta queries;
27. Executor no interpreta entidades;
28. streaming no bufferiza todo;
29. cursor pagination no es offset codificado;
30. large datasets tienen APIs bounded;
31. QueryContext es scope-local;
32. FrankenPHP no filtra estado entre requests;
33. queries pueden inspeccionarse;
34. SQL puede inspeccionarse sin ejecución;
35. bindings sensibles pueden redactarse;
36. Framework Explain y DBMS EXPLAIN están separados;
37. query testing semántico es posible;
38. DBMS behavior se valida mediante integración;
39. extensions pueden añadir nodos tipados;
40. security policies no son bypasses cosméticos;
41. routing ocurre fuera del Builder;
42. sharding ocurre fuera del Builder;
43. retry ocurre fuera del Builder;
44. cache no redefine verdad Database;
45. query performance puede medirse por etapa;
46. persistent runtimes están soportados;
47. errors identifican la fase;
48. capability UNKNOWN no se trata como UNSUPPORTED;
49. raw SQL no es necesario para la mayoría de operaciones comunes;
50. toda la API converge en el mismo Query Engine.

---

# 304. Principio final

La Query API de VoltStack deberá ofrecer la ergonomía de un Query Builder moderno sin adoptar la arquitectura de un generador de strings SQL.

La regla definitiva será:

> **El desarrollador expresará qué consulta desea realizar; el Query Engine determinará cómo representar, validar, optimizar, planificar, compilar y ejecutar esa intención de forma segura para la plataforma efectiva.**

Por tanto:

```text
Developer Intent
      ↓
Semantic Query
      ↓
Validated Meaning
      ↓
Execution Strategy
      ↓
Platform Representation
      ↓
Execution
```

y nunca:

```text
Developer Calls
      ↓
String Concatenation
      ↓
PDO
```

La experiencia deberá sentirse sencilla:

```php
$users = DB::table('users')
    ->where('active', true)
    ->orderBy('name')
    ->get();
```

mientras internamente VoltStack conserva:

```text
Query Builder
+
AST
+
Semantic Graph
+
Optimizer
+
Planner
+
Capability System
+
Compiler
+
Execution Engine
+
Connection Manager
+
Driver
```

Esto permitirá que la misma API pueda evolucionar hacia consultas más avanzadas, nuevas plataformas, nuevos runtimes y extensiones futuras sin romper el modelo arquitectónico de `VoltStack/Quantum/Database`.

---

# 305. Siguiente documento

```text
307_DATABASE_SCHEMA_DEVELOPER_EXPERIENCE.md
```

El siguiente documento definirá la experiencia de desarrollo del Schema System, incluyendo:

```text
Schema::create()
Schema::table()
table blueprints
columns
indexes
constraints
foreign keys
generated columns
platform capabilities
schema introspection
schema diff
schema compilation
safe destructive operations
schema inspection
IDE experience
testing
migration integration
```

manteniendo como regla:

> **Schema Builder expresará estructura e intención de transformación; nunca será un generador directo de SQL ni ejecutará cambios estructurales por sí mismo.**

Arquitectura esperada:

```text
Developer Schema API
        ↓
Schema Builder
        ↓
Schema Definitions
        ↓
Schema AST
        ↓
Validation
        ↓
Schema Planner
        ↓
Platform Capability Analysis
        ↓
Schema Compiler
        ↓
Schema Commands
        ↓
Execution Engine
        ↓
Driver
        ↓
DBMS
```