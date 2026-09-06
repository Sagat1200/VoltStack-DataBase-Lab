# 24_DATABASE_QUERY_MODEL.md

# VoltStack Quantum Database
## Query Model

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 24 — Query Model  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Representation  
**Versión:** 1.0

---

# 1. Propósito

Este documento define el **Query Model** de `VoltStack/Quantum/Database`.

El Query Model será la representación estructurada, portable y semánticamente explícita de una operación de consulta antes de que ésta sea transformada en el AST canónico utilizado por las etapas internas del Query Engine.

Su función principal será convertir APIs de construcción de alto nivel como:

```php
DB::table('users')
    ->select('id', 'name')
    ->where('active', true)
    ->orderBy('name')
    ->limit(100);
```

en una representación estructurada equivalente a:

```text
SelectQuery
├── Source
│   └── TableSource(users)
├── Projection
│   ├── Column(id)
│   └── Column(name)
├── Predicate
│   └── active = Parameter(true)
├── Ordering
│   └── name ASC
└── Limit
    └── 100
```

sin generar SQL.

---

# 2. Posición arquitectónica

El Query Model se encuentra entre:

```text
Public Query API
      │
      ▼
Query Builder
      │
      ▼
QUERY MODEL
      │
      ▼
Query AST
      │
      ▼
Semantic Engine
```

---

# 3. Regla fundamental

```text
Query Builder
≠
Query Model
≠
Query AST
≠
SQL
```

---

# 4. Responsabilidad

El Query Model responde:

> **¿Qué operación de datos quiere representar la aplicación?**

No responde:

> **¿Cómo se escribe esa operación en SQL para PostgreSQL, MySQL, MariaDB o SQLite?**

---

# 5. Objetivos

El Query Model deberá ser:

- estructurado;
- portable;
- tipado;
- explícito;
- independiente del Driver;
- independiente de PDO;
- independiente del Dialect;
- independiente del ORM;
- independiente de la conexión física;
- compatible con extensiones;
- sencillo de transformar a AST;
- adecuado para validación estructural;
- seguro por defecto.

---

# 6. No objetivos

El Query Model no deberá:

- generar SQL;
- ejecutar consultas;
- abrir conexiones;
- seleccionar replicas;
- resolver PDO;
- iniciar transacciones;
- hidratar Entities;
- realizar persistence tracking;
- implementar UnitOfWork;
- consultar directamente Schema;
- conocer detalles sintácticos del Dialect;
- contener estado global.

---

# 7. Arquitectura

```text
                Query Builder
                     │
                     ▼
              QueryModelFactory
                     │
                     ▼
                  Query
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   SelectQuery   InsertQuery   UpdateQuery
                                   │
                                   ▼
                              DeleteQuery

                     │
                     ▼
              Query Components
                     │
       ┌─────────────┼───────────────────┐
       ▼             ▼                   ▼
    Sources      Expressions         Metadata
       │             │                   │
       ├── Join      ├── Predicate       ├── Intent
       ├── CTE       ├── Projection      ├── Hints
       └── Subquery  └── Ordering        └── Origin
                     │
                     ▼
              Query AST Builder
                     │
                     ▼
                  Query AST
```

---

# 8. Query como Aggregate

Cada consulta deberá representarse como un aggregate estructural.

Ejemplos:

```text
SelectQuery
InsertQuery
UpdateQuery
DeleteQuery
```

Cada uno tendrá reglas propias.

---

# 9. Query contract

Conceptualmente:

```php
interface QueryInterface
{
    public function type(): QueryType;

    public function metadata(): QueryMetadata;
}
```

La interfaz pública deberá mantenerse pequeña.

---

# 10. QueryType

```php
enum QueryType: string
{
    case Select = 'select';
    case Insert = 'insert';
    case Update = 'update';
    case Delete = 'delete';
}
```

Podrán agregarse otros tipos mediante extensiones controladas.

---

# 11. No Query God Object

No utilizar:

```php
class Query
{
    public bool $select;
    public bool $insert;
    public bool $update;
    public bool $delete;

    // cientos de propiedades opcionales
}
```

Preferir:

```text
QueryInterface
├── SelectQuery
├── InsertQuery
├── UpdateQuery
└── DeleteQuery
```

---

# 12. Query Model hierarchy

```text
QueryInterface
│
├── ReadQueryInterface
│   └── SelectQuery
│
└── MutationQueryInterface
    ├── InsertQuery
    ├── UpdateQuery
    └── DeleteQuery
```

La jerarquía deberá mantenerse pequeña y basada en comportamiento real.

---

# 13. Composición sobre herencia

La mayor parte del modelo deberá construirse mediante composición.

Ejemplo:

```text
SelectQuery
├── WithClause
├── ProjectionList
├── SourceCollection
├── Predicate
├── Grouping
├── Having
├── Ordering
├── Pagination
├── Locking
└── QueryMetadata
```

---

# 14. Modelo canónico

La representación conceptual principal será:

```text
Query
├── Operation
├── Sources
├── Expressions
├── Parameters
├── Clauses
├── Requirements
└── Metadata
```

---

# 15. SelectQuery

`SelectQuery` representará una operación de lectura.

Conceptualmente:

```php
final readonly class SelectQuery implements QueryInterface
{
    public function __construct(
        public ProjectionList $projection,
        public SourceCollection $sources,
        public ?Predicate $where,
        public JoinCollection $joins,
        public Grouping $grouping,
        public ?Predicate $having,
        public Ordering $ordering,
        public ?Pagination $pagination,
        public ?LockingClause $locking,
        public WithClause $with,
        public SetOperationCollection $setOperations,
        public QueryMetadata $metadata,
    ) {}
}
```

La forma concreta podrá variar.

---

# 16. SelectQuery example

```text
SelectQuery
│
├── Projection
│   ├── users.id
│   ├── users.name
│   └── users.email
│
├── Source
│   └── users
│
├── Where
│   └── users.active = :p1
│
├── OrderBy
│   └── users.name ASC
│
└── Pagination
    └── LIMIT 100
```

La representación anterior es conceptual.

`:p1` todavía no implica syntax SQL concreta.

---

# 17. Projection

Projection representa los valores que debe producir un `SELECT`.

Ejemplos:

```text
ColumnProjection
ExpressionProjection
AggregateProjection
WildcardProjection
SubqueryProjection
```

---

# 18. Projection item

Conceptualmente:

```php
final readonly class ProjectionItem
{
    public function __construct(
        public Expression $expression,
        public ?AliasIdentifier $alias = null,
    ) {}
}
```

---

# 19. Projection list

```text
ProjectionList
├── ProjectionItem(users.id)
├── ProjectionItem(users.name)
└── ProjectionItem(COUNT(orders.id), order_count)
```

---

# 20. Wildcard projection

Representar:

```sql
SELECT *
```

como:

```text
WildcardProjection
```

y no como:

```text
Column("*")
```

si esa distinción facilita semántica y expansión.

---

# 21. Qualified wildcard

```text
users.*
```

podrá representarse como:

```text
QualifiedWildcardProjection(users)
```

---

# 22. Source Model

Las fuentes de datos deberán ser estructuradas.

Base conceptual:

```text
QuerySource
├── TableSource
├── SubquerySource
├── CteSource
├── DerivedSource
├── FunctionSource
└── ExtensionSource
```

---

# 23. TableSource

Conceptualmente:

```php
final readonly class TableSource implements QuerySource
{
    public function __construct(
        public QualifiedIdentifier $table,
        public ?AliasIdentifier $alias = null,
    ) {}
}
```

---

# 24. Table names are identifiers

No strings SQL.

Preferir:

```text
QualifiedIdentifier
├── namespace/schema
└── name
```

cuando aplique.

---

# 25. Qualified identifiers

Deberán permitir representar:

```text
users

public.users

analytics.events
```

sin incluir quoting específico.

---

# 26. Identifier quoting

No pertenece al Query Model.

```text
Query Model
     │
     ▼
Identifier
     │
     ▼
Compiler + Dialect
     │
     ▼
"users" / `users`
```

---

# 27. Aliases

Alias deberá ser un objeto estructurado:

```text
AliasIdentifier("u")
```

No:

```text
"users AS u"
```

---

# 28. SubquerySource

Una subconsulta deberá contener otro Query Model.

```text
SubquerySource
└── SelectQuery
```

---

# 29. No SQL subqueries

Incorrecto:

```php
new SubquerySource(
    'SELECT id FROM users'
);
```

como camino estructurado principal.

---

# 30. Correcto

```text
SubquerySource
└── SelectQuery
    ├── Projection(id)
    └── Source(users)
```

---

# 31. CTE Source

Un CTE deberá representarse mediante objetos propios.

```text
CommonTableExpression
├── Name
├── ColumnList
├── Query
├── RecursiveMetadata
└── MaterializationHint
```

cuando las capabilities aplicables lo permitan.

---

# 32. WithClause

```text
WithClause
├── CTE(active_users)
└── CTE(recent_orders)
```

---

# 33. Recursive CTE

La recursividad será intención semántica:

```text
RecursiveCommonTableExpression
```

No syntax vendor-specific.

---

# 34. Join Model

Los joins serán objetos estructurados.

```text
Join
├── Type
├── Source
├── Condition
└── Metadata
```

---

# 35. JoinType

Core conceptual:

```php
enum JoinType: string
{
    case Inner = 'inner';
    case Left = 'left';
    case Right = 'right';
    case Full = 'full';
    case Cross = 'cross';
}
```

La disponibilidad efectiva dependerá de capabilities.

---

# 36. Join condition

Una condición:

```text
users.id = orders.user_id
```

se representará mediante un Predicate.

---

# 37. Example Join

```text
Join
├── type: LEFT
├── source: orders
└── predicate
    └── Equality
        ├── users.id
        └── orders.user_id
```

---

# 38. CROSS JOIN

Podrá no requerir predicate.

El modelo deberá validar esa diferencia.

---

# 39. Join semantics

Query Model registra intención.

Semantic Engine resolverá:

- aliases;
- columnas;
- scopes;
- ambigüedades;
- tipos;
- relaciones.

---

# 40. Predicate Model

Las condiciones deberán utilizar el sistema definido posteriormente en:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Conceptualmente:

```text
Predicate
├── Comparison
├── Logical
├── Null
├── Between
├── In
├── Exists
├── Like
└── Extension
```

---

# 41. WHERE

`WHERE` deberá contener un Predicate.

```text
WhereClause
└── Predicate
```

o directamente un `?Predicate` en el aggregate, según implementación final.

---

# 42. HAVING

También utilizará Predicate.

Pero:

```text
WHERE
≠
HAVING
```

semánticamente.

El contexto deberá conservarse.

---

# 43. Expression Model

Expressions serán definidas formalmente en:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

El Query Model únicamente las compondrá.

---

# 44. Expression examples

```text
ColumnReference
ParameterExpression
LiteralExpression
FunctionExpression
AggregateExpression
ArithmeticExpression
CaseExpression
CastExpression
SubqueryExpression
```

---

# 45. ColumnReference

Ejemplo:

```text
ColumnReference
├── source: users
└── column: name
```

o inicialmente:

```text
UnresolvedColumnReference("users", "name")
```

antes del Semantic Engine.

---

# 46. Unresolved references

El Query Model podrá contener referencias todavía no resueltas.

Ejemplo:

```text
ColumnReference("name")
```

Semantic Analysis decidirá a qué source pertenece.

---

# 47. Resolved references

No deberían ser responsabilidad del Builder.

---

# 48. InsertQuery

`InsertQuery` representará inserciones.

Conceptualmente:

```text
InsertQuery
├── Target
├── Columns
├── Values
├── SourceQuery
├── ConflictStrategy
├── Returning
└── Metadata
```

---

# 49. Insert target

```text
MutationTarget
└── TableIdentifier
```

No deberá contener SQL.

---

# 50. Insert values

Ejemplo:

```php
DB::table('users')->insert([
    'name' => 'Ana',
    'email' => 'ana@example.com',
]);
```

podrá convertirse en:

```text
InsertQuery
├── Target(users)
├── Columns
│   ├── name
│   └── email
└── Rows
    └── Row
        ├── Parameter(Ana)
        └── Parameter(ana@example.com)
```

---

# 51. InsertRow

Conceptualmente:

```php
final readonly class InsertRow
{
    public function __construct(
        public ExpressionCollection $values,
    ) {}
}
```

---

# 52. Multi-row insert

```text
InsertQuery
├── Columns(name, email)
└── Rows
    ├── Row(:p1, :p2)
    ├── Row(:p3, :p4)
    └── Row(:p5, :p6)
```

---

# 53. Row shape validation

Todas las rows deberán ser compatibles con el target column shape.

---

# 54. Insert from query

Debe soportarse conceptualmente:

```text
INSERT INTO archive_users (...)
SELECT ...
```

como:

```text
InsertQuery
├── Target
├── TargetColumns
└── SourceQuery
    └── SelectQuery
```

---

# 55. Values vs SourceQuery

El modelo deberá impedir estados ambiguos como:

```text
VALUES
+
SELECT source
```

simultáneamente cuando no tengan semántica válida.

---

# 56. Insert source abstraction

Podría modelarse como:

```text
InsertSource
├── ValuesInsertSource
├── QueryInsertSource
└── DefaultValuesInsertSource
```

evitando propiedades mutuamente excluyentes.

---

# 57. Default values

Una inserción basada en defaults deberá ser una operación explícita.

```text
DefaultValuesInsertSource
```

sujeta a capabilities.

---

# 58. Returning

INSERT podrá declarar:

```text
ReturningClause
```

de forma semántica.

---

# 59. Example

```php
DB::table('users')
    ->insert(...)
    ->returning('id');
```

representará:

```text
ReturningClause
└── Column(id)
```

No:

```text
"RETURNING id"
```

---

# 60. Returning capability

Posteriormente:

```text
Query Model
    │
    ▼
Capability Requirement
    │
    ▼
Planner
    │
    ├── Native
    ├── Safe alternative
    └── Unsupported
```

---

# 61. UpdateQuery

Conceptualmente:

```text
UpdateQuery
├── Target
├── Assignments
├── Sources/Joins
├── Predicate
├── Ordering
├── Limit
├── Returning
└── Metadata
```

Las features permitidas dependerán de Platform capabilities.

---

# 62. Assignment

Una asignación deberá representarse como:

```text
Assignment
├── TargetColumn
└── Expression
```

---

# 63. Example update

```php
DB::table('users')
    ->where('id', 42)
    ->update([
        'active' => false,
    ]);
```

se convierte conceptualmente en:

```text
UpdateQuery
├── Target(users)
├── Assignment
│   └── active = :p1
└── Predicate
    └── id = :p2
```

---

# 64. Expression assignments

Debe ser posible representar:

```text
counter = counter + 1
```

sin Raw SQL.

```text
Assignment
├── counter
└── AddExpression
    ├── Column(counter)
    └── Literal(1)
```

---

# 65. Mutation target

El target de UPDATE deberá distinguirse de sources auxiliares.

---

# 66. Update joins

Algunas plataformas permiten distintos modelos de UPDATE con joins/from.

El Query Model deberá expresar una semántica neutral cuando sea posible.

Planner determinará la estrategia soportada.

---

# 67. Update returning

Será una feature semántica capability-driven.

---

# 68. DeleteQuery

Conceptualmente:

```text
DeleteQuery
├── Target
├── Sources/Using
├── Predicate
├── Ordering
├── Limit
├── Returning
└── Metadata
```

---

# 69. Example delete

```php
DB::table('sessions')
    ->where('expired_at', '<', $now)
    ->delete();
```

representará:

```text
DeleteQuery
├── Target(sessions)
└── Predicate
    └── expired_at < :p1
```

---

# 70. Delete target

El target deberá ser explícito.

---

# 71. Delete safety metadata

El Query Model podrá representar información utilizada por políticas de seguridad.

Ejemplo:

```text
MutationSafetyMetadata
├── hasPredicate
├── explicitAllowFullTableMutation
└── origin
```

Pero la política que prohíbe full-table mutations pertenecerá a otra capa.

---

# 72. No implicit safety bypass

Una query sin `WHERE` no deberá adquirir automáticamente:

```text
allowFullTableMutation = true
```

---

# 73. Full table mutation

Podrá requerir una API explícita:

```php
$query->allowFullTableMutation();
```

si VoltStack decide implementar esta protección.

---

# 74. Upsert Model

Upsert no deberá modelarse como SQL raw.

---

# 75. Semantic Upsert

Conceptualmente:

```text
UpsertQuery
```

o:

```text
InsertQuery
└── ConflictAction
```

---

# 76. Preferred design

Inicialmente se recomienda:

```text
InsertQuery
└── ConflictStrategy
```

para mantener la operación dentro de la semántica INSERT.

---

# 77. ConflictStrategy

Podrá modelar:

```text
NONE
IGNORE
UPDATE
```

con detalles estructurados.

---

# 78. ConflictTarget

Conceptualmente:

```text
ConflictTarget
├── Columns
├── ConstraintReference
└── Predicate
```

cuando las capacidades del target lo permitan.

---

# 79. ConflictAction

```text
ConflictAction
├── Ignore
└── Update
```

---

# 80. Vendor syntax exclusion

No colocar en Query Model:

```text
ON DUPLICATE KEY UPDATE
ON CONFLICT
INSERT OR REPLACE
```

---

# 81. Planner responsibility

El Planner mapeará la operación semántica al mecanismo apropiado.

---

# 82. Set Operations

SELECT deberá soportar:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
```

como objetos semánticos.

---

# 83. SetOperation

```text
SetOperation
├── Type
├── LeftQuery
└── RightQuery
```

---

# 84. SetOperationType

Conceptualmente:

```php
enum SetOperationType: string
{
    case Union = 'union';
    case UnionAll = 'union_all';
    case Intersect = 'intersect';
    case Except = 'except';
}
```

---

# 85. Set operation validation

Semantic Engine deberá validar compatibilidad de proyecciones.

---

# 86. Column cardinality

Ejemplo:

```text
SELECT id, name
UNION
SELECT id
```

deberá detectarse como incompatible antes de ejecución cuando exista información suficiente.

---

# 87. Type compatibility

También podrán comprobarse tipos compatibles.

---

# 88. Grouping Model

```text
Grouping
└── GroupExpressionCollection
```

---

# 89. GroupBy

Cada elemento será Expression.

No strings SQL.

---

# 90. Having

`Having` utiliza Predicate pero con semantic context de agrupación.

---

# 91. Aggregate semantics

Semantic Engine deberá validar:

- columnas no agrupadas;
- aggregates;
- nested aggregates;
- HAVING;
- platform-specific rules.

---

# 92. Ordering Model

```text
Ordering
└── OrderItemCollection
```

---

# 93. OrderItem

Conceptualmente:

```php
final readonly class OrderItem
{
    public function __construct(
        public Expression $expression,
        public SortDirection $direction,
        public ?NullOrdering $nullOrdering = null,
    ) {}
}
```

---

# 94. SortDirection

```text
ASC
DESC
```

---

# 95. NullOrdering

Conceptualmente:

```text
FIRST
LAST
PLATFORM_DEFAULT
```

---

# 96. Null ordering capability

Si la Platform no soporta syntax nativa, Planner podrá:

- emular;
- usar default;
- fallar;

dependiendo de la semántica solicitada.

---

# 97. Pagination Model

```text
Pagination
├── Limit
└── Offset
```

para offset pagination.

---

# 98. Limit

Deberá ser un value object validado.

Ejemplo:

```text
Limit(100)
```

---

# 99. Invalid limit

No permitir:

```text
Limit(-1)
```

salvo que exista semántica explícita diferente.

---

# 100. Offset

Igualmente:

```text
Offset(0..N)
```

---

# 101. Pagination syntax

No pertenece al Query Model.

---

# 102. Cursor Pagination

Cursor pagination es principalmente una estrategia de construcción.

Podrá transformar:

```text
CursorPaginationRequest
```

en:

```text
Predicate
+
Ordering
+
Limit
```

antes del AST.

---

# 103. Deterministic cursor ordering

La estrategia deberá garantizar un ordering suficientemente estable.

---

# 104. Locking Model

```text
LockingClause
├── LockMode
├── WaitPolicy
└── Targets
```

---

# 105. LockMode

Conceptualmente:

```text
UPDATE
SHARE
```

y otros modos sólo cuando exista una semántica portable o extensión explícita.

---

# 106. WaitPolicy

```text
WAIT
NO_WAIT
SKIP_LOCKED
```

sujeto a capabilities.

---

# 107. Lock targets

Algunas plataformas permiten seleccionar targets del lock.

El modelo podrá representarlos opcionalmente.

---

# 108. Locking semantics

Locking podrá generar:

```text
TransactionRequirement
ConnectionRoleRequirement
CapabilityRequirement
```

durante análisis/planning.

---

# 109. Distinct

`DISTINCT` será una propiedad semántica.

---

# 110. Distinct model

Podrá representarse como:

```text
DistinctMode
├── NONE
├── ALL_PROJECTION
└── EXPRESSION_SET
```

si se soportan extensiones tipo `DISTINCT ON`.

---

# 111. Portable core

El core deberá soportar:

```text
SELECT DISTINCT ...
```

sin asumir `DISTINCT ON`.

---

# 112. Platform-specific distinct

Features adicionales serán capability/extension-driven.

---

# 113. Query Parameters

Los parámetros serán definidos en:

```text
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

pero Query Model deberá referenciarlos estructuralmente.

---

# 114. Parameter identity

Cada parámetro deberá tener identidad independiente del placeholder SQL.

```text
QueryParameterId
```

---

# 115. Example

```text
ParameterExpression
└── QueryParameterId(7)
```

Posteriormente podría compilarse como:

```text
?
```

o:

```text
$1
```

sin cambiar Query Model.

---

# 116. Parameter values

Se deberá distinguir:

```text
Parameter Definition
Parameter Value
```

cuando sea útil para query reuse/cache.

---

# 117. Query shape

Ejemplo:

```text
users.email = Parameter<string>
```

es shape.

---

# 118. Runtime binding

```text
Parameter#1 = "ana@example.com"
```

es runtime value.

---

# 119. Query Model and cache

Separar shape y values permitirá reutilización futura.

---

# 120. QueryMetadata

Cada Query podrá contener metadata.

Conceptualmente:

```text
QueryMetadata
├── Origin
├── Label
├── ConnectionIntent
├── ConsistencyRequirement
├── CacheHints
├── ExecutionHints
├── SecurityMetadata
├── DiagnosticsMetadata
└── ExtensionMetadata
```

---

# 121. Metadata categories

Debe distinguirse:

```text
Semantic Metadata
Execution Metadata
Diagnostic Metadata
Extension Metadata
```

---

# 122. Semantic metadata

Puede cambiar significado o requisitos.

Ejemplos:

```text
locking
consistency
mutation safety
```

---

# 123. Execution metadata

Puede afectar cómo ejecutar sin cambiar resultado lógico.

Ejemplos:

```text
timeout
streaming preference
connection preference
```

---

# 124. Diagnostic metadata

Ejemplos:

```text
origin
source location
debug label
```

---

# 125. Extension metadata

Deberá estar namespaced.

Ejemplo conceptual:

```text
vendor.package.feature
```

---

# 126. Metadata must be typed

Evitar:

```php
$metadata['whatever'] = mixed;
```

como modelo principal.

---

# 127. Metadata bags

Si se requiere una bolsa extensible deberá utilizar:

- claves namespaced;
- schemas;
- typed values;
- ownership claro.

---

# 128. Query Origin

Podrá representar:

```text
APPLICATION
ORM
REPOSITORY
SCHEMA
MIGRATION
FRAMEWORK
EXTENSION
```

---

# 129. Query label

Labels serán opcionales y sanitizados.

---

# 130. Source location

En development podrá almacenarse:

```text
file
line
component
```

para debugging.

---

# 131. Production metadata

La captura costosa de stack traces no deberá activarse por defecto.

---

# 132. Connection intent

Query Model podrá contener un intent explícito o dejarlo para inferencia.

---

# 133. Intent inference

Ejemplos:

```text
SelectQuery
→ READ

InsertQuery
→ WRITE

UpdateQuery
→ WRITE

DeleteQuery
→ WRITE
```

---

# 134. Locking override

```text
SelectQuery + FOR UPDATE
```

puede inferir:

```text
PRIMARY_REQUIRED
```

---

# 135. Explicit vs inferred metadata

El sistema deberá distinguir:

```text
Explicit Requirement
Inferred Requirement
```

para detectar conflictos.

---

# 136. Example conflict

```text
Query = UPDATE
Explicit connection = READ_ONLY_REPLICA
```

deberá fallar.

---

# 137. Query Requirements

Podrá existir una estructura derivada:

```text
QueryRequirements
├── Connection
├── Transaction
├── Capability
├── Consistency
├── Result
└── Execution
```

---

# 138. Requirements are not necessarily Builder state

Muchos requisitos serán derivados por Semantic/Planner.

---

# 139. Query Model should remain descriptive

Evitar cargar el Query Model con decisiones de ejecución prematuras.

---

# 140. Query hints

Hints deberán distinguirse de requirements.

```text
Requirement
=
must

Hint
=
prefer
```

---

# 141. Example

```text
Requirement:
transaction required

Hint:
prefer streaming
```

---

# 142. Hint failure

No cumplir un hint no necesariamente debe fallar.

---

# 143. Requirement failure

No cumplir un requirement sí deberá fallar.

---

# 144. QueryOptions

Opciones puramente de construcción podrán mantenerse fuera del modelo semántico si dejan de ser relevantes tras construirlo.

---

# 145. No Builder leakage

El Query Model no deberá contener:

```text
builder callbacks
builder macros
mutable fluent state
```

---

# 146. Query immutability

El Query Model deberá ser preferentemente inmutable.

---

# 147. Example

```php
final readonly class SelectQuery
{
    // immutable state
}
```

---

# 148. Builder conversion

Builder mutable:

```text
Mutable Builder
      │
      ▼
Immutable Query Model
```

es una estrategia válida.

---

# 149. Immutable Builder alternative

También podrá utilizarse:

```text
Immutable Builder
      │
      ▼
Immutable Query Model
```

---

# 150. Final choice

La mutabilidad del Builder se decidirá en el documento 43.

El Query Model deberá permanecer independiente de esa decisión.

---

# 151. Query Model cloning

Al ser inmutable, una variación produce un nuevo modelo.

```text
Query A
  │
  ├── withPredicate(...)
  ▼
Query B
```

---

# 152. Structural sharing

Podrá emplearse internamente para evitar copias costosas.

---

# 153. Collections

Las colecciones importantes deberán ser tipos propios.

Ejemplos:

```text
ProjectionList
SourceCollection
JoinCollection
AssignmentCollection
OrderItemCollection
CteCollection
ParameterCollection
```

---

# 154. No arbitrary arrays internally

Evitar:

```php
array $joins;
array $orders;
array $columns;
```

en APIs internas críticas si se pierde invariantes.

---

# 155. Typed collection benefits

Permiten:

- invariantes;
- validación;
- mejor análisis estático;
- mejor IDE support;
- fingerprints deterministas;
- extensibilidad controlada.

---

# 156. Empty collections

Preferir:

```text
JoinCollection::empty()
```

sobre `null` cuando conceptualmente existe una colección vacía.

---

# 157. Null semantics

Utilizar `null` sólo cuando:

```text
absence
```

sea semánticamente distinta de:

```text
empty collection
```

---

# 158. Invalid states

El diseño deberá hacer difíciles los estados inválidos.

---

# 159. Example

En lugar de:

```php
new InsertQuery(
    values: $values,
    select: $select,
);
```

preferir:

```php
new InsertQuery(
    source: new ValuesInsertSource(...)
);
```

o:

```php
new InsertQuery(
    source: new QueryInsertSource(...)
);
```

---

# 160. Make invalid states unrepresentable

Principio:

> Siempre que sea razonable, una estructura inválida no deberá poder construirse.

---

# 161. Construction validation

Value objects deberán validar invariantes locales.

---

# 162. Structural validation

`QueryValidator` validará invariantes entre componentes.

---

# 163. Semantic validation

Semantic Engine validará significado.

---

# 164. Capability validation

Capability System validará disponibilidad.

---

# 165. Separation

```text
Local invariant
→ Value Object

Structural invariant
→ Query Validator

Semantic invariant
→ Semantic Engine

Platform availability
→ Capability Resolution
```

---

# 166. Query Model normalization

El Query Model podrá aceptar distintas formas de entrada desde Builder.

Antes del AST se producirá una representación canónica.

---

# 167. Example

Builder:

```php
->where([
    'active' => true,
    'verified' => true,
])
```

puede convertirse a:

```text
AndPredicate
├── active = :p1
└── verified = :p2
```

---

# 168. Builder sugar disappears

El Query Model no necesita conservar que la aplicación utilizó:

```text
where(array)
```

si esa información no tiene significado posterior.

---

# 169. Semantic preservation

Debe conservar únicamente aquello que pueda afectar:

- significado;
- ejecución;
- seguridad;
- diagnostics;
- extensiones.

---

# 170. Query Model to AST

La transformación conceptual será:

```text
Query Model
     │
     ▼
QueryAstFactory
     │
     ▼
Query AST
```

---

# 171. QueryAstFactory

Podrá ser:

```php
interface QueryAstFactoryInterface
{
    public function create(QueryInterface $query): QueryNode;
}
```

como concepto.

---

# 172. AST conversion must be deterministic

Mismo Query Model:

```text
+
same model configuration
```

deberá producir el mismo AST estructural.

---

# 173. Query Model ≠ AST

Aunque algunos value objects puedan reutilizarse, no deberán confundirse los conceptos.

---

# 174. Query Model purpose

Está optimizado para:

```text
construction
developer-facing semantics
framework integration
```

---

# 175. AST purpose

Está optimizado para:

```text
analysis
rewriting
semantic processing
optimization
planning
compilation
```

---

# 176. Why keep both

Permite que el Builder evolucione sin contaminar el AST.

---

# 177. Example

Builder puede ofrecer:

```php
->whereBetween(...)
```

Query Model puede representar:

```text
BetweenPredicate
```

AST podrá normalizarlo posteriormente a una forma canónica si conviene.

---

# 178. Query Model evolution

La API pública podrá agregar convenience methods sin agregar necesariamente nuevos AST nodes.

---

# 179. AST evolution

El AST podrá cambiar internamente sin romper la API pública cuando esté marcado `Internal`.

---

# 180. Public stability

Los contratos de Query Model expuestos a extensions deberán tener una política de estabilidad explícita.

---

# 181. API categories

```text
Public
Extension
Internal
Implementation
```

---

# 182. Public Query Model

Sólo los tipos que realmente deban ser construidos por usuarios/extensions deberán ser públicos.

---

# 183. Internal Query Model

Detalles de normalización podrán mantenerse internos.

---

# 184. Query model factories

Se podrán proporcionar factories para evitar acoplar usuarios a constructores internos.

---

# 185. Example

```php
$queryFactory->select(...)
```

para integraciones avanzadas.

---

# 186. Extension query types

No toda extensión necesitará crear un nuevo `QueryType`.

---

# 187. Prefer composition

Ejemplo:

```text
SelectQuery
+
CustomExpression
```

es preferible a:

```text
CustomSelectQuery
```

cuando sea suficiente.

---

# 188. New QueryType

Sólo cuando la operación tenga lifecycle y semántica realmente distintos.

---

# 189. Query feature extension

Una feature nueva podrá aportar:

```text
Expression
Predicate
Clause
Metadata
Query Type
```

dependiendo del caso.

---

# 190. Extension descriptor

Deberá declarar qué componentes agrega.

---

# 191. Extension validation

Una extensión incompleta deberá fallar antes del runtime cuando sea posible.

---

# 192. Query Model portability

Cada elemento podrá tener una clasificación conceptual:

```text
PORTABLE
CAPABILITY_DEPENDENT
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

---

# 193. Portable example

```text
EqualityPredicate
```

---

# 194. Capability-dependent example

```text
ReturningClause
```

---

# 195. Platform-specific example

Una feature semántica exclusiva de un engine.

---

# 196. Dialect-specific example

Un constructo de sintaxis específico.

---

# 197. Raw example

```text
TrustedRawExpression
```

---

# 198. Portability metadata

El sistema de diagnostics podrá explicar:

```text
Query portability:
PORTABLE_WITH_CAPABILITY_REQUIREMENTS
```

---

# 199. Query model and Raw SQL

Raw SQL deberá utilizar un modelo separado.

---

# 200. RawSqlQuery

Conceptualmente:

```php
final readonly class RawSqlQuery
{
    public function __construct(
        public TrustedSql $sql,
        public ParameterCollection $parameters,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 201. RawSqlQuery does not pretend to be structured

No parsear automáticamente Raw SQL para fingir que es un `SelectQuery` salvo que exista un parser explícito separado.

---

# 202. Raw operation classification

Raw SQL deberá declarar o inferir conservadoramente:

```text
READ
WRITE
DDL
UNKNOWN
```

---

# 203. Conservative default

Si no se conoce:

```text
UNKNOWN
```

---

# 204. Unknown raw SQL

No deberá enviarse automáticamente a una replica.

---

# 205. Raw parameter safety

Bindings siguen siendo obligatorios para valores dinámicos.

---

# 206. Raw identifiers

No se resolverán mediante bindings.

Por tanto deberán considerarse trusted input.

---

# 207. Query Model and ORM

ORM producirá Query Models.

```text
Entity Query
     │
     ▼
Query Model
```

---

# 208. ORM select

Ejemplo:

```text
UserRepository
      │
      ▼
Entity Query
      │
      ▼
SelectQuery
```

---

# 209. ORM persistence

```text
PersistencePlan
      │
      ▼
InsertQuery / UpdateQuery / DeleteQuery
```

---

# 210. ORM metadata not embedded in core query

No colocar:

```text
EntityManager
UnitOfWork
Entity object
```

dentro del Query Model.

---

# 211. ORM-specific metadata

Si es necesaria para hydration/provenance deberá ser una estructura separada y controlada.

---

# 212. Query Model must remain reusable

Debe poder ser utilizado sin ORM instalado.

---

# 213. Query Model and Schema

Schema DDL utilizará su propio Schema Model.

---

# 214. Query Model for introspection

Las consultas internas de introspection podrán utilizar Query Model cuando sean DML/SELECT normales.

---

# 215. Schema operations

No representar:

```text
CREATE TABLE
ALTER TABLE
DROP INDEX
```

como `SelectQuery` genérico.

---

# 216. Query Model and Transactions

El Query Model puede expresar requirements.

No posee la transacción.

---

# 217. Transaction object forbidden

No:

```text
SelectQuery
└── TransactionObject
```

---

# 218. Correct

```text
SelectQuery
└── TransactionRequirement
```

---

# 219. Query Model and Connection

No almacenar conexión física.

---

# 220. Connection name

Un query podrá tener un:

```text
ConnectionPreference
```

o explicit target lógico cuando la API lo solicite.

---

# 221. Logical only

Ejemplo:

```text
ConnectionName("analytics")
```

No:

```text
PDO instance
```

---

# 222. Query Model and Driver

Driver IDs tampoco deberán ser necesarios para una consulta normal.

---

# 223. Platform-independent model

Una misma `SelectQuery` deberá poder compilarse para múltiples targets cuando sus capabilities lo permitan.

---

# 224. Query Model and Dialect

No almacenar:

```text
quote character
placeholder style
SQL keywords
```

---

# 225. Query Model and Platform

No almacenar un `DatabasePlatform` mutable.

Capability requirements podrán derivarse después.

---

# 226. Query Model and types

Expressions podrán contener tipos declarados o hints.

---

# 227. Type resolution

Semantic Engine determinará tipos efectivos.

---

# 228. Parameter type hint

Ejemplo:

```text
Parameter
├── value
└── declared type: UUID
```

---

# 229. Column type

Normalmente se resolverá mediante Schema Metadata.

---

# 230. Query Model and Value Objects

Se favorecerán:

```text
QueryName
QueryParameterId
QualifiedIdentifier
AliasIdentifier
Limit
Offset
SortDirection
JoinType
LockMode
```

sobre primitives sin semántica.

---

# 231. Primitive obsession

Evitar:

```php
new Query(
    type: 1,
    limit: -42,
    direction: 'whatever',
);
```

---

# 232. Value object validation

Construcciones inválidas deberán fallar temprano.

---

# 233. Query Model serialization

El Query Model no deberá asumir que siempre será serializable.

---

# 234. Why

Puede contener:

- extension objects;
- complex expressions;
- metadata;
- type objects.

---

# 235. Serialization support

Si se implementa deberá ser:

```text
explicit
versioned
safe
```

---

# 236. No PHP serialize as protocol

No utilizar `serialize()` como contrato estable del Query Model.

---

# 237. Query Model fingerprint

Deberá existir una representación determinista para fingerprinting sin requerir serialización pública.

---

# 238. Fingerprint input

Podrá considerar:

```text
query type
sources
projection
expressions
predicates
clauses
semantic metadata
extension identities
```

---

# 239. Runtime values

Normalmente deberán excluirse del shape fingerprint.

---

# 240. QueryModelFingerprint

Conceptualmente:

```text
QueryModelFingerprint
```

será diferente de:

```text
SemanticQueryFingerprint
CompiledQueryFingerprint
```

---

# 241. Query Model normalization fingerprint

El fingerprint útil para compilation cache probablemente se calculará después de normalización/semantic analysis.

---

# 242. Do not over-cache early

No asumir que Query Model fingerprint es suficiente para CompiledQuery cache.

---

# 243. Debug representation

Cada Query Model podrá tener una representación segura para diagnostics.

---

# 244. Example

```text
SELECT QUERY
Source:
  users AS u

Projection:
  u.id
  u.name

Predicate:
  u.active = Parameter<boolean>

Ordering:
  u.name ASC
```

---

# 245. Debug representation is not SQL

Nunca utilizarla para ejecución.

---

# 246. Sensitive values

No mostrar valores sensibles por defecto.

---

# 247. Query Model visitor

Podrá utilizarse un Visitor cuando sea apropiado.

---

# 248. Visitor use cases

```text
AST conversion
fingerprinting
diagnostics
validation
metadata collection
```

---

# 249. Avoid visitor abuse

No centralizar toda nueva feature en un único visitor gigantesco.

---

# 250. Pattern alternatives

Dependiendo de cada dominio podrán utilizarse:

```text
Visitor
Handler Registry
Pattern Matching
Double Dispatch
Strategy
```

---

# 251. Query model handlers

Una arquitectura extensible puede utilizar:

```text
QueryModelHandlerRegistry
```

especializado.

---

# 252. No universal handler registry

No mezclar:

```text
query handlers
AST handlers
compiler handlers
type handlers
driver handlers
```

en un registro único.

---

# 253. Query Model factories

Propuesta:

```text
QueryModelFactory
├── select()
├── insert()
├── update()
└── delete()
```

pero el Builder será normalmente el principal consumidor.

---

# 254. Factory statelessness

Factories podrán ser singleton si son stateless.

---

# 255. Query model configuration

No deberá consultar config global.

---

# 256. Configuration injection

Si una factory necesita políticas:

```text
QueryModelConfiguration
```

inmutable.

---

# 257. Query Model lifetime

Normalmente:

```text
operation
```

aunque un modelo inmutable podrá mantenerse más tiempo por el código de aplicación.

---

# 258. No resource ownership

Query Model no posee:

```text
connection
statement
cursor
result
transaction
lease
```

---

# 259. Persistent runtime safety

Un Query Model inmutable no deberá contener state del request salvo metadata explícita que no escape del scope.

---

# 260. Cache safety

No cachear Query Models que contengan referencias request-scoped.

---

# 261. Preferred shared state

Sólo:

```text
immutable value objects
immutable metadata
stable identifiers
```

---

# 262. Query Model and closures

Evitar closures dentro del Query Model.

---

# 263. Builder closures

Una API como:

```php
->where(function ($query) {
    ...
})
```

deberá ejecutar la closure durante construcción.

El Query Model recibirá el resultado estructurado.

---

# 264. Correct flow

```text
Builder Closure
     │
     ▼
Nested Builder
     │
     ▼
Predicate Model
```

---

# 265. Closure does not survive

No:

```text
QueryModel
└── Closure
```

---

# 266. Query Model and callbacks

Mismo principio para:

- joins;
- subqueries;
- CTEs;
- nested predicates.

---

# 267. Example nested where

```php
$query->where(function ($q) {
    $q->where('role', 'admin')
      ->orWhere('role', 'manager');
});
```

produce:

```text
GroupedPredicate
└── OrPredicate
    ├── role = :p1
    └── role = :p2
```

---

# 268. Query Model and macros

Macros son Builder/DX concerns.

---

# 269. Macro output

Una macro deberá producir componentes estándar o extension components del Query Model.

---

# 270. Macro not persisted

El nombre de la macro no necesita llegar al Query Model salvo diagnostics.

---

# 271. Query Model and global scopes

Los scopes podrán transformar Query Models.

---

# 272. Example

```text
Original Query
      │
      ▼
SoftDeleteScope
      │
      ▼
Query + deleted_at IS NULL
```

---

# 273. Scope provenance

Podrá mantenerse metadata de transformación para debugging.

---

# 274. Query transformations must be immutable

Preferir:

```text
Query A
→ Transformer
→ Query B
```

---

# 275. QueryPolicyPipeline

Las transformaciones de política ocurrirán antes de optimización.

---

# 276. Query model canonicalization

Después de policies y normalización deberá obtenerse una forma adecuada para AST.

---

# 277. Duplicate predicates

Podrán permanecer hasta normalization/optimizer.

El Builder no necesita deduplicarlos.

---

# 278. Contradictory predicates

Ejemplo:

```text
id = 1
AND
id = 2
```

podrá ser detectado posteriormente por Constraint Analysis/Optimizer.

---

# 279. Query Model remains declarative

No intentará ejecutar lógica compleja de optimización.

---

# 280. Query model errors

Jerarquía conceptual:

```text
QueryModelException
├── InvalidQueryModelException
├── InvalidQuerySourceException
├── InvalidProjectionException
├── InvalidAssignmentException
├── InvalidPaginationException
├── InvalidLockingException
├── InvalidSetOperationException
└── InvalidMutationTargetException
```

---

# 281. Construction errors

Deberán contener información estructural segura.

---

# 282. No SQL in model errors

Un error del Query Model no debería depender del SQL final.

---

# 283. Testing strategy

El Query Model deberá probarse independientemente del servidor.

---

# 284. Unit tests

No deberán necesitar:

```text
MySQL
MariaDB
PostgreSQL
SQLite
PDO
FrankenPHP
```

---

# 285. Select tests

Cubrir:

- projections;
- sources;
- aliases;
- predicates;
- joins;
- grouping;
- ordering;
- pagination;
- locking;
- CTEs;
- set operations.

---

# 286. Insert tests

Cubrir:

- single row;
- multiple rows;
- insert from query;
- default values;
- returning;
- conflict strategy;
- invalid row shape.

---

# 287. Update tests

Cubrir:

- assignments;
- expression assignments;
- predicates;
- returning;
- target;
- invalid assignments.

---

# 288. Delete tests

Cubrir:

- target;
- predicates;
- returning;
- safety metadata;
- full table operation.

---

# 289. Immutability tests

Comprobar que transformar Query A no altera Query A.

---

# 290. Fingerprint tests

Mismo shape:

```text
same fingerprint
```

cuando corresponda.

---

# 291. Parameter value tests

Valores diferentes podrán mantener el mismo structural fingerprint.

---

# 292. Determinism tests

La misma entrada deberá producir el mismo Query Model normalizado.

---

# 293. Property tests

Útiles para:

- nested predicates;
- parameter ordering;
- immutable transformations;
- collection invariants.

---

# 294. Architecture tests

Deberán impedir imports desde Query Model hacia:

```text
PDO
Driver implementations
Connection physical resources
ORM EntityManager
UnitOfWork
HTTP Request
Container
```

---

# 295. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Model\
```

---

# 296. Proposed structure

```text
Query/
└── Model/
    ├── Contract/
    │   ├── QueryInterface.php
    │   ├── ReadQueryInterface.php
    │   ├── MutationQueryInterface.php
    │   └── QuerySource.php
    │
    ├── Select/
    │   ├── SelectQuery.php
    │   ├── ProjectionList.php
    │   ├── ProjectionItem.php
    │   └── DistinctMode.php
    │
    ├── Insert/
    │   ├── InsertQuery.php
    │   ├── InsertSource.php
    │   ├── ValuesInsertSource.php
    │   ├── QueryInsertSource.php
    │   ├── InsertRow.php
    │   ├── ConflictStrategy.php
    │   └── ConflictTarget.php
    │
    ├── Update/
    │   ├── UpdateQuery.php
    │   ├── Assignment.php
    │   └── AssignmentCollection.php
    │
    ├── Delete/
    │   └── DeleteQuery.php
    │
    ├── Source/
    │   ├── TableSource.php
    │   ├── SubquerySource.php
    │   ├── CteSource.php
    │   └── DerivedSource.php
    │
    ├── Join/
    │   ├── Join.php
    │   ├── JoinType.php
    │   └── JoinCollection.php
    │
    ├── Cte/
    │   ├── WithClause.php
    │   └── CommonTableExpression.php
    │
    ├── SetOperation/
    │   ├── SetOperation.php
    │   └── SetOperationType.php
    │
    ├── Grouping/
    ├── Ordering/
    ├── Pagination/
    ├── Locking/
    ├── Returning/
    ├── Metadata/
    ├── Requirement/
    ├── Raw/
    ├── Factory/
    ├── Fingerprint/
    ├── Diagnostics/
    └── Exception/
```

---

# 297. Boundary with Expression System

Expression implementations podrán vivir en:

```text
Query\Expression\
```

en lugar de dentro de `Query\Model`.

El Query Model sólo dependerá de sus contratos estables.

---

# 298. Boundary with Predicate System

Igualmente:

```text
Query\Predicate\
```

será un dominio especializado.

---

# 299. Boundary with Parameter System

```text
Query\Parameter\
```

será responsable de parameter identity/type/value representation.

---

# 300. Boundary with AST

```text
Query\Model
```

no dependerá de implementaciones AST si se adopta una conversión unidireccional.

Preferencia:

```text
AST layer
→ reads Query Model
```

o un mapper neutral.

---

# 301. Dependency model

```text
Builder
   │
   ▼
Query Model
   │
   ├── Expression Contracts
   ├── Predicate Contracts
   ├── Parameter Contracts
   └── Identifier Value Objects
   │
   ▼
AST Conversion
```

---

# 302. DB-QM-001

Query Model nunca generará SQL.

---

# 303. DB-QM-002

Query Model no dependerá del Driver.

---

# 304. DB-QM-003

Query Model no dependerá de PDO.

---

# 305. DB-QM-004

Query Model no abrirá conexiones.

---

# 306. DB-QM-005

Query Model no ejecutará consultas.

---

# 307. DB-QM-006

Query Model no poseerá transactions.

---

# 308. DB-QM-007

Query Model no poseerá cursors/results/statements.

---

# 309. DB-QM-008

Query Model no dependerá del ORM.

---

# 310. DB-QM-009

ORM podrá depender del Query Model.

---

# 311. DB-QM-010

Select, Insert, Update y Delete tendrán modelos especializados.

---

# 312. DB-QM-011

No se utilizará un único God Query object.

---

# 313. DB-QM-012

Los componentes se construirán principalmente mediante composición.

---

# 314. DB-QM-013

Sources serán objetos estructurados.

---

# 315. DB-QM-014

Subqueries serán Query Models, no strings SQL.

---

# 316. DB-QM-015

CTEs serán objetos estructurados.

---

# 317. DB-QM-016

Joins serán objetos estructurados.

---

# 318. DB-QM-017

Expressions serán estructuradas.

---

# 319. DB-QM-018

Predicates serán estructurados.

---

# 320. DB-QM-019

Dynamic values utilizarán Query Parameters.

---

# 321. DB-QM-020

Placeholder syntax no aparecerá en Query Model.

---

# 322. DB-QM-021

Identifiers no estarán pre-quoted.

---

# 323. DB-QM-022

Dialect syntax no aparecerá en Query Model portable.

---

# 324. DB-QM-023

Upsert se representará semánticamente.

---

# 325. DB-QM-024

Returning se representará semánticamente.

---

# 326. DB-QM-025

Locking se representará semánticamente.

---

# 327. DB-QM-026

Set operations serán estructuradas.

---

# 328. DB-QM-027

Invalid states deberán ser difíciles de representar.

---

# 329. DB-QM-028

Value objects validarán invariantes locales.

---

# 330. DB-QM-029

Structural validation pertenecerá al Query Validator.

---

# 331. DB-QM-030

Semantic validation pertenecerá al Semantic Engine.

---

# 332. DB-QM-031

Capability validation no se resolverá mediante vendor checks.

---

# 333. DB-QM-032

Query Model será preferentemente inmutable.

---

# 334. DB-QM-033

Collections críticas serán tipadas.

---

# 335. DB-QM-034

Builder closures no sobrevivirán dentro del Query Model.

---

# 336. DB-QM-035

Builder macros no formarán parte del modelo semántico.

---

# 337. DB-QM-036

Query Model no contendrá Service Container.

---

# 338. DB-QM-037

Query Model no contendrá HTTP Request.

---

# 339. DB-QM-038

Query Model no contendrá CurrentUser.

---

# 340. DB-QM-039

Query Model no contendrá EntityManager.

---

# 341. DB-QM-040

Query Model no contendrá UnitOfWork.

---

# 342. DB-QM-041

Query Model no contendrá PhysicalConnection.

---

# 343. DB-QM-042

Query Model no contendrá NativeStatement.

---

# 344. DB-QM-043

Query Model transformations serán deterministas.

---

# 345. DB-QM-044

Query Model transformations no mutarán modelos compartidos.

---

# 346. DB-QM-045

Diagnostic representation nunca será utilizada para ejecución.

---

# 347. DB-QM-046

Sensitive values serán ocultados en diagnostics.

---

# 348. DB-QM-047

RawSqlQuery será explícitamente diferente del Query Model estructurado.

---

# 349. DB-QM-048

Raw SQL desconocido será clasificado conservadoramente.

---

# 350. DB-QM-049

Raw SQL seguirá utilizando safe bindings.

---

# 351. DB-QM-050

Query Model será independiente del runtime HTTP.

---

# 352. DB-QM-051

Query Model deberá funcionar igual en:

```text
HTTP
CLI
Queue
Scheduler
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 353. DB-QM-052

Query Model no almacenará mutable request-scoped state en caches globales.

---

# 354. DB-QM-053

Extension metadata estará namespaced.

---

# 355. DB-QM-054

Extension query components deberán utilizar extension contracts oficiales.

---

# 356. DB-QM-055

No habrá un universal query component registry.

---

# 357. DB-QM-056

Requirements y hints serán conceptos distintos.

---

# 358. DB-QM-057

Connection requirements serán lógicos, no recursos físicos.

---

# 359. DB-QM-058

Transaction requirements no contendrán Transaction objects.

---

# 360. DB-QM-059

Query origin será metadata, no dependencia arquitectónica.

---

# 361. DB-QM-060

El Query Model deberá poder convertirse determinísticamente en AST.

---

# 362. Anti-pattern — SQL fragments

Incorrecto:

```php
new SelectQuery(
    where: 'active = 1'
);
```

Correcto:

```text
SelectQuery
└── Predicate
    └── Equality
        ├── Column(active)
        └── Parameter(true)
```

---

# 363. Anti-pattern — quoted identifiers

Incorrecto:

```text
ColumnReference("`users`.`name`")
```

Correcto:

```text
ColumnReference
├── source: users
└── column: name
```

---

# 364. Anti-pattern — placeholder leakage

Incorrecto:

```text
Parameter("$1")
```

Correcto:

```text
Parameter(QueryParameterId)
```

---

# 365. Anti-pattern — vendor Upsert

Incorrecto:

```text
InsertQuery
└── sqlSuffix = "ON DUPLICATE KEY UPDATE..."
```

Correcto:

```text
InsertQuery
└── ConflictStrategy
    └── Update
```

---

# 366. Anti-pattern — Entity in Query

Incorrecto:

```text
SelectQuery
└── Entity(User::class)
```

como dependencia del Query core.

Correcto:

```text
ORM Metadata
    │
    ▼
Entity Query Translator
    │
    ▼
SelectQuery
└── TableSource(users)
```

---

# 367. Anti-pattern — current connection

Incorrecto:

```php
$query->connection = DB::connection();
```

Correcto:

```text
Query
└── ConnectionRequirement/Preference
```

---

# 368. Anti-pattern — mutable shared query

Incorrecto:

```text
Singleton Query
→ modified by Request A
→ reused by Request B
```

---

# 369. Correct persistent model

```text
Application/Worker
├── Immutable factories/registries
│
├── Request A
│   └── Query Model A
│
└── Request B
    └── Query Model B
```

---

# 370. Anti-pattern — arrays everywhere

Incorrecto:

```php
[
    'type' => 'select',
    'from' => 'users',
    'where' => [...],
    'joins' => [...],
]
```

como arquitectura interna principal.

---

# 371. Correct typed model

```text
SelectQuery
├── TableSource
├── Predicate
└── JoinCollection
```

---

# 372. Anti-pattern — closure inside model

Incorrecto:

```text
Predicate
└── Closure
```

Correcto:

```text
Closure
→ Builder
→ Structured Predicate
```

---

# 373. Anti-pattern — query model optimizer

No convertir `SelectQuery` en un objeto que también:

- resuelve aliases;
- infiere tipos;
- optimiza;
- compila;
- ejecuta.

---

# 374. Correct pipeline

```text
Query Model
    │
    ▼
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
Compiler
```

---

# 375. Modelo completo de SELECT

```text
SelectQuery
│
├── WithClause
│   ├── CTE A
│   └── CTE B
│
├── Distinct
│
├── ProjectionList
│   ├── Projection A
│   ├── Projection B
│   └── Projection C
│
├── SourceCollection
│   └── TableSource
│
├── JoinCollection
│   ├── Join A
│   └── Join B
│
├── WherePredicate
│
├── Grouping
│
├── HavingPredicate
│
├── SetOperations
│
├── Ordering
│
├── Pagination
│
├── Locking
│
└── QueryMetadata
```

---

# 376. Modelo completo de INSERT

```text
InsertQuery
│
├── WithClause
│
├── MutationTarget
│
├── TargetColumns
│
├── InsertSource
│   ├── ValuesInsertSource
│   │   └── Rows
│   │
│   ├── QueryInsertSource
│   │   └── SelectQuery
│   │
│   └── DefaultValuesInsertSource
│
├── ConflictStrategy
│
├── ReturningClause
│
└── QueryMetadata
```

---

# 377. Modelo completo de UPDATE

```text
UpdateQuery
│
├── WithClause
│
├── MutationTarget
│
├── AssignmentCollection
│
├── AuxiliarySources
│
├── JoinCollection
│
├── Predicate
│
├── Ordering
│
├── Pagination/Limit
│
├── ReturningClause
│
└── QueryMetadata
```

Las cláusulas concretamente disponibles serán capability-driven.

---

# 378. Modelo completo de DELETE

```text
DeleteQuery
│
├── WithClause
│
├── MutationTarget
│
├── AuxiliarySources
│
├── Predicate
│
├── Ordering
│
├── Pagination/Limit
│
├── ReturningClause
│
└── QueryMetadata
```

---

# 379. Modelo de subquery

```text
SubqueryExpression
└── SelectQuery
    ├── Projection
    ├── Source
    └── Predicate
```

---

# 380. Modelo de CTE

```text
WithClause
└── CommonTableExpression
    ├── Identifier
    ├── Optional Column List
    ├── Query
    └── Semantic Options
```

---

# 381. Modelo de Query Requirements

```text
Query
 │
 ▼
Semantic Analysis
 │
 ▼
QueryRequirements
├── CapabilityRequirements
├── ConnectionRequirement
├── TransactionRequirement
├── ConsistencyRequirement
├── ResultRequirement
└── ExecutionRequirement
```

---

# 382. Important distinction

El Query Model puede contener información explícita proporcionada por la aplicación.

Pero el conjunto efectivo de requirements deberá calcularse posteriormente.

---

# 383. Example

```text
SelectQuery
├── Locking(FOR_UPDATE)
└── Metadata
    └── explicit timeout = 5s
```

produce después:

```text
Effective Requirements
├── PRIMARY connection
├── transaction compatible
├── row locking capability
└── timeout = 5s
```

---

# 384. Query Model canonical flow

```text
Developer API
     │
     ▼
Query Builder
     │
     ▼
Raw Builder State
     │
     ▼
Query Model Factory
     │
     ▼
Immutable Query Model
     │
     ▼
Policy Transformations
     │
     ▼
Normalized Query Model
     │
     ▼
AST
```

---

# 385. Query Model role in architecture

El Query Model será la frontera entre:

```text
Developer Ergonomics
```

y:

```text
Query Engine Internals
```

---

# 386. Developer side

```text
DB::table()
where()
join()
orderBy()
limit()
```

---

# 387. Engine side

```text
SelectQuery
Predicate
Join
Ordering
Pagination
```

---

# 388. This separation is critical

Permite cambiar:

```text
Builder API
```

sin rediseñar necesariamente:

```text
Semantic Engine
Optimizer
Planner
Compiler
```

---

# 389. Query Model minimalism

El modelo deberá contener únicamente información necesaria para representar la operación.

---

# 390. No premature planning

No almacenar en Query Model:

```text
chosen compiler
chosen physical connection
chosen replica
prepared statement
execution retry count
```

---

# 391. No premature compilation

No almacenar:

```text
SQL
quoted identifiers
native placeholders
```

---

# 392. No premature semantic resolution

El Builder no deberá necesitar Schema Metadata para construir una consulta básica.

---

# 393. Deferred knowledge

Ejemplo:

```text
ColumnReference("email")
```

puede permanecer unresolved hasta Semantic Analysis.

---

# 394. Architecture consequence

VoltStack podrá construir consultas incluso cuando:

```text
database server is offline
```

si la operación sólo requiere construcción.

---

# 395. Compile offline

También podrá compilar offline cuando disponga de:

```text
Platform target
Dialect
Capabilities
Required metadata
```

---

# 396. Query Model quality criteria

Un buen Query Model deberá permitir responder:

1. ¿Qué operación representa?
2. ¿Sobre qué fuentes?
3. ¿Qué valores produce o modifica?
4. ¿Qué condiciones aplica?
5. ¿Qué parámetros contiene?
6. ¿Qué semántica explícita solicita?
7. ¿Qué metadata relevante transporta?
8. ¿Puede transformarse a AST sin reconstruir SQL?
9. ¿Puede inspeccionarse sin conexión?
10. ¿Puede validarse estructuralmente?

---

# 397. Architectural formula

```text
Query Model
=
Operation
+
Structured Sources
+
Structured Expressions
+
Structured Predicates
+
Parameters
+
Clauses
+
Explicit Metadata
```

---

# 398. What Query Model excludes

```text
Query Model
≠
SQL
+
Driver
+
Connection
+
Execution
+
ORM State
+
Runtime State
```

---

# 399. Final architecture

```text
                  Query Builder
                       │
                       ▼
                 QUERY MODEL
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    SelectQuery     InsertQuery    UpdateQuery
                                       │
                                       ▼
                                  DeleteQuery

        │
        ├── Sources
        ├── Expressions
        ├── Predicates
        ├── Parameters
        ├── Joins
        ├── CTEs
        ├── Grouping
        ├── Ordering
        ├── Pagination
        ├── Locking
        ├── Returning
        ├── Set Operations
        └── Metadata
                       │
                       ▼
                   Query AST
                       │
                       ▼
                Semantic Engine
```

---

# 400. Regla maestra

> **El Query Model es la representación estructurada de la intención de consulta de VoltStack: suficientemente expresiva para describir la operación completa, suficientemente neutral para no conocer SQL específico de plataforma y suficientemente simple para convertirse determinísticamente en el AST canónico del Query Engine.**

En forma compacta:

```text
Builder
    │
    ▼
Query Model
    │
    ▼
AST
```

Nunca:

```text
Builder
    │
    ▼
Query Model
    │
    ▼
SQL
```

---

# 401. Relación con documentos siguientes

```text
23_DATABASE_QUERY_ARCHITECTURE.md
        │
        ▼
defines complete Query Engine pipeline

24_DATABASE_QUERY_MODEL.md
        │
        ▼
defines structured query domain
        │
        ▼
25_DATABASE_QUERY_AST_SYSTEM.md
        │
        ▼
defines canonical internal representation
        │
        ▼
26_DATABASE_QUERY_AST_NODE_MODEL.md
        │
        ▼
defines AST node taxonomy
        │
        ├───────────────┐
        ▼               ▼
27 Expressions      28 Predicates
        │               │
        └───────┬───────┘
                ▼
29 Parameters
                │
                ▼
30 Types
```

---

# 402. Próximo documento

El siguiente documento será:

```text
25_DATABASE_QUERY_AST_SYSTEM.md
```

Su responsabilidad será definir la representación interna canónica sobre la cual trabajarán:

```text
Normalization
Semantic Analysis
Query Rewrite
Optimizer
Planner
Compiler
Fingerprinting
Diagnostics
```

y deberá establecer con precisión:

```text
AST Root
Node Contracts
Node Identity
Immutability
Structural Equality
Visitors
Transformers
Traversal
Extension Nodes
Source Locations
Canonicalization
AST Validation
AST Fingerprinting
```

sin permitir que el AST se convierta en SQL prematuramente.