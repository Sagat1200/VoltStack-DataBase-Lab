# 46_DATABASE_UPDATE_QUERY_BUILDER.md

# VoltStack Quantum Database
## Update Query Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 46 — Update Query Builder  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / UPDATE  
**Versión:** 1.0

---

# 1. Propósito

`UpdateQueryBuilder` será la API especializada de VoltStack para construir operaciones de actualización.

Ejemplo:

```php
DB::table('users')
    ->where('id', $userId)
    ->update([
        'name' => 'Ana',
        'active' => true,
    ]);
```

La API tendrá ergonomía Laravel-like, pero internamente no generará directamente:

```sql
UPDATE users
SET ...
WHERE ...
```

El flujo será:

```text
Developer
   │
   ▼
UpdateQueryBuilder
   │
   ▼
UpdateQueryModel
   │
   ▼
UpdateQueryNode
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
UpdateQueryBuilder
≠
UPDATE SQL Generator
```

El Builder expresa:

```text
target
assignments
predicates
joins
source relations
ordering intent
limit intent
returning intent
metadata
parameters
```

pero nunca decide la representación SQL final.

> `UpdateQueryBuilder` describe qué relación y qué valores deben modificarse bajo determinadas condiciones; no describe cómo debe escribirse físicamente el SQL.

---

# 3. Objetivos

El sistema deberá proporcionar:

- API Laravel-like.
- Target estructurado.
- Assignment model estructurado.
- Parameterización automática.
- Expresiones en assignments.
- `NULL` explícito.
- `DEFAULT` explícito.
- Operaciones aritméticas.
- Operaciones JSON.
- Subqueries.
- Predicados reutilizando Predicate System.
- JOIN-aware UPDATE.
- `UPDATE ... FROM` semántico.
- `RETURNING`.
- Optimistic locking.
- Pessimistic/concurrency integration.
- Safety policies contra updates accidentales.
- Bulk updates.
- Schema-aware validation.
- Type inference.
- Capability-driven portability.
- Persistent-runtime safety.

---

# 4. No responsabilidades

`UpdateQueryBuilder` no será responsable de:

```text
SQL generation
identifier quoting
placeholder generation
native parameter binding
schema introspection
physical connection selection
transaction management
lock acquisition
constraint execution
query optimization
join strategy selection
index selection
bulk execution strategy
ORM dirty checking
UnitOfWork
IdentityMap
entity lifecycle events
```

---

# 5. Arquitectura general

```text
                     UpdateQueryBuilder
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
    TargetBuilder    AssignmentBuilder   PredicateBuilder
          │                 │                 │
          │                 │                 │
          ├─────────────────┼─────────────────┤
          │                 │                 │
          ▼                 ▼                 ▼
     JoinBuilder       ReturningBuilder   MetadataBuilder
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    UpdateBuilderState
                            │
                            ▼
                   UpdateBuilderFinalizer
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
          UpdateQueryModel         BindingSet
                 │
                 ▼
           UpdateQueryNode
```

---

# 6. API pública básica

```php
$affected = DB::table('users')
    ->where('id', $userId)
    ->update([
        'name' => 'Ana',
        'active' => true,
    ]);
```

Conceptualmente:

```text
UpdateQueryModel
├── target
│   └── users
├── assignments
│   ├── name   = P1
│   └── active = P2
└── predicate
    └── id = P3
```

Bindings:

```text
P1 → "Ana"
P2 → true
P3 → $userId
```

---

# 7. Builder explícito

También podrá existir:

```php
DB::update()
    ->table('users')
    ->set('name', 'Ana')
    ->set('active', true)
    ->where('id', $userId)
    ->execute();
```

Ambas APIs deberán producir el mismo modelo canónico.

---

# 8. Canonical Update Model

Conceptualmente:

```php
final readonly class UpdateQueryModel
{
    public function __construct(
        public UpdateTarget $target,
        public AssignmentSet $assignments,
        public ?PredicateNode $predicate,
        public RelationSourceSet $sources,
        public JoinSet $joins,
        public ?OrderingSpecification $ordering,
        public ?LimitSpecification $limit,
        public ?ReturningSpecification $returning,
        public ParameterDefinitionSet $parameters,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 9. UpdateBuilderState

Durante construcción:

```text
UpdateBuilderState
├── target
├── assignments
├── predicate
├── joins
├── sources
├── ordering
├── limit
├── returning
├── parameterRegistry
├── bindingBuilder
├── metadataBuilder
└── sourceMap
```

Será:

```text
operation-scoped
mutable
temporary
non-cacheable
non-shared
```

---

# 10. Finalización

```text
UpdateBuilderState
       │
       ▼
UpdateBuilderFinalizer
       │
       ├───────────────┐
       ▼               ▼
UpdateQueryModel    BindingSet
       │
       ▼
UpdateQueryNode
```

El modelo final será immutable.

---

# 11. Target

```php
DB::table('users')->update(...);
```

producirá:

```text
UpdateTarget
└── RelationIdentifier(users)
```

El string recibido en el boundary será convertido inmediatamente a un identifier estructurado.

---

# 12. Qualified targets

```php
DB::table('public.users')->update(...);
```

podrá representar:

```text
QualifiedRelationIdentifier
├── namespace: public
└── relation: users
```

sin realizar quoting ni asumir sintaxis SQL.

---

# 13. Target alias

Cuando la semántica lo permita:

```php
DB::table('users as u')
```

podrá producir:

```text
AliasedUpdateTarget
├── relation: users
└── alias: u
```

La validez concreta será resuelta posteriormente.

---

# 14. Assignment System

El corazón específico del UPDATE será:

```text
Assignment
=
Target Column
+
Value Expression
```

Ejemplo:

```php
->set('name', 'Ana')
```

produce:

```text
Assignment
├── target
│   └── ColumnIdentifier(name)
└── value
    └── ParameterExpression(P1)
```

---

# 15. AssignmentSet

```text
AssignmentSet
├── Assignment(name, P1)
├── Assignment(active, P2)
└── Assignment(updated_at, CurrentTimestamp)
```

El orden podrá conservarse para:

- diagnostics;
- deterministic fingerprints;
- source mapping;

sin asumir que el orden tenga significado SQL cuando semánticamente no lo tenga.

---

# 16. Duplicate assignments

Esto:

```php
->set('name', 'Ana')
->set('name', 'Carlos')
```

deberá producir por defecto:

```text
DuplicateUpdateAssignmentException
```

No se aplicará silenciosamente "last write wins".

---

# 17. Array assignment API

```php
->update([
    'name' => 'Ana',
    'active' => true,
]);
```

se normalizará al mismo `AssignmentSet`.

---

# 18. Runtime values

Los valores normales estarán fuera del AST.

```php
[
    'name' => 'Ana',
]
```

produce:

```text
AST
└── ParameterExpression(P1)

BindingSet
└── P1 → "Ana"
```

---

# 19. NULL explícito

```php
->set('deleted_at', null)
```

significa:

```text
Assignment
├── deleted_at
└── Parameter(P1)

P1 → NULL
```

---

# 20. DEFAULT explícito

```php
->set('status', DB::default())
```

produce:

```text
Assignment
├── status
└── DefaultValueExpression
```

Regla:

```text
DEFAULT
≠
NULL
≠
missing assignment
```

---

# 21. Missing assignment

Una columna que no aparece en el `AssignmentSet`:

```text
is not modified
```

No significa:

```text
set NULL
```

ni:

```text
set DEFAULT
```

---

# 22. Expression assignments

Ejemplo:

```php
DB::table('users')
    ->where('id', $id)
    ->update([
        'updated_at' => DB::expr()->currentTimestamp(),
    ]);
```

produce una expresión semántica.

---

# 23. Arithmetic assignments

VoltStack deberá soportar:

```php
DB::table('products')
    ->where('id', $id)
    ->set(
        'stock',
        DB::expr()->column('stock')->subtract($quantity)
    )
    ->execute();
```

Conceptualmente:

```text
Assignment
├── target: products.stock
└── value
    └── SubtractExpression
        ├── ColumnReference(stock)
        └── Parameter(P1)
```

---

# 24. Increment/decrement DX

Convenience APIs:

```php
DB::table('products')
    ->where('id', $id)
    ->increment('stock', 5);

DB::table('products')
    ->where('id', $id)
    ->decrement('stock', 2);
```

deberán convertirse al mismo Expression/Assignment System.

---

# 25. No special increment engine

```text
increment()
decrement()
update()
set()
```

convergerán a:

```text
UpdateQueryModel
```

---

# 26. Multiple increments

Podrá expresarse:

```php
DB::table('statistics')
    ->where('id', $id)
    ->update([
        'views' => DB::expr()->column('views')->add(1),
        'score' => DB::expr()->column('score')->add($delta),
    ]);
```

---

# 27. Type inference

Para:

```text
stock Integer
-
P1 Unknown
```

Semantic Analysis podrá inferir:

```text
P1 → Integer-compatible
```

---

# 28. Invalid arithmetic

Si:

```text
users.email : EmailAddress
```

se utiliza como:

```text
email + 5
```

Semantic Analysis deberá rechazarlo.

---

# 29. String expressions

Podrán existir:

```text
ConcatExpression
LowerExpression
UpperExpression
SubstringExpression
```

sin SQL vendor-specific en el Builder.

---

# 30. Date/time expressions

Podrán expresarse:

```text
CurrentTimestamp
DateAdd
DateSubtract
DateTruncate
```

como funciones/operaciones semánticas capability-aware.

---

# 31. JSON assignments

Ejemplo conceptual:

```php
->set(
    'preferences',
    DB::expr()->jsonSet(
        DB::expr()->column('preferences'),
        '$.theme',
        'dark',
    )
)
```

---

# 32. JSON semantics

El Builder expresará:

```text
JSON_SET semantic operation
```

no:

```text
JSON_SET(...)
jsonb_set(...)
json_set(...)
```

---

# 33. JSON path

Las rutas JSON deberán representarse mediante estructuras seguras:

```text
JsonPath
├── Property(theme)
└── ...
```

cuando sea viable.

No deberán interpolarse ingenuamente.

---

# 34. Subquery assignment

Ejemplo:

```php
DB::table('users')
    ->where('id', $id)
    ->set(
        'last_order_at',
        DB::table('orders')
            ->select('created_at')
            ->whereColumn('orders.user_id', 'users.id')
            ->latest()
            ->limit(1)
    );
```

---

# 35. Subquery representation

```text
Assignment
├── target: users.last_order_at
└── value
    └── SubqueryExpression
        └── SelectQueryArtifact
```

---

# 36. Child builder finalization

El Select Builder deberá finalizarse antes de incorporarse.

```text
SelectQueryBuilder
       │
       ▼
SelectQueryArtifact
       │
       ▼
SubqueryExpression
```

Nunca se conservará un Builder hijo mutable.

---

# 37. Correlated subqueries

Las referencias correlacionadas serán resueltas por Semantic Analysis.

El Builder sólo preservará las referencias estructuradas.

---

# 38. Predicate System

`WHERE` reutilizará completamente:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Ejemplo:

```php
DB::table('users')
    ->where('status', 'pending')
    ->where('created_at', '<', $date)
    ->update([
        'status' => 'expired',
    ]);
```

---

# 39. Predicate AST

Conceptualmente:

```text
AND
├── status = P2
└── created_at < P3
```

Assignment:

```text
status = P1
```

---

# 40. No UPDATE-specific predicate engine

SELECT, UPDATE y DELETE reutilizarán el mismo Predicate System.

---

# 41. whereNull()

```php
->whereNull('deleted_at')
```

produce:

```text
NullPredicateNode
```

No:

```text
deleted_at = NULL
```

---

# 42. whereIn()

```php
->whereIn('id', $ids)
```

utilizará:

```text
CollectionParameter
```

cuando corresponda.

No expandirá inmediatamente el array en SQL placeholders.

---

# 43. Empty collections

La semántica de:

```php
->whereIn('id', [])
```

será resuelta antes de SQL.

Para una política donde empty `IN` significa false:

```text
UPDATE ...
WHERE FALSE
```

semánticamente.

---

# 44. UPDATE sin WHERE

VoltStack deberá prestar atención especial a:

```php
DB::table('users')->update([
    'active' => false,
]);
```

porque modifica toda la relación.

---

# 45. Unbounded Update

Se definirá:

```text
UnboundedUpdate
```

como un UPDATE sin restricción semántica suficiente.

---

# 46. Safety Policy

Podrá existir:

```text
UpdateSafetyPolicy
```

con modos como:

```text
ALLOW
WARN
REQUIRE_EXPLICIT_CONFIRMATION
FORBID
```

---

# 47. Default recomendado

Para el Query Builder genérico, VoltStack podrá mantener compatibilidad Laravel-like permitiendo el UPDATE, pero deberá hacerlo visible mediante diagnostics/telemetry.

Para APIs administrativas o configuraciones estrictas podrá requerirse:

```php
->allowUnboundedUpdate()
```

---

# 48. Explicit unbounded intent

Ejemplo:

```php
DB::table('sessions')
    ->allowUnboundedUpdate()
    ->update([
        'expired' => true,
    ]);
```

Esto crea metadata explícita.

No desactiva:

```text
authorization
security
resource governance
transaction requirements
```

---

# 49. Safety metadata

Podrá registrar:

```text
UnboundedMutationIntent::EXPLICIT
```

para:

- audit;
- diagnostics;
- security policy;
- testing;
- administration tooling.

---

# 50. Predicate presence ≠ bounded safety

Esto:

```text
WHERE TRUE
```

no deberá considerarse automáticamente seguro.

---

# 51. Semantic boundedness

Constraint Analysis podrá determinar si el predicate realmente restringe el target.

Ejemplo:

```text
id = P1
```

sobre PK implica una cardinalidad potencialmente limitada.

---

# 52. Cardinality classification

Podrá derivarse:

```text
SINGLE_ROW_EXPECTED
BOUNDED_SET
UNBOUNDED_SET
UNKNOWN
```

como metadata semántica/planning hint.

---

# 53. No false guarantees

`id = P1` puede implicar como máximo una fila si `id` es UNIQUE/PK.

Pero:

```text
status = 'active'
```

no garantiza un conjunto pequeño.

---

# 54. Constraint Analysis integration

El documento:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

podrá derivar estas propiedades usando:

```text
schema keys
unique constraints
predicate equivalences
constant constraints
null constraints
join constraints
```

---

# 55. Update joins

VoltStack deberá soportar operaciones de UPDATE que dependan de otras relaciones.

Pero no mediante sintaxis vendor-specific en el Builder.

---

# 56. Semantic model

```text
UpdateQuery
├── target: users
├── sourceRelations
│   └── accounts
├── relation predicates
└── assignments
```

---

# 57. Join-aware API

Podrá existir:

```php
DB::table('users as u')
    ->join('accounts as a', 'a.user_id', '=', 'u.id')
    ->where('a.suspended', true)
    ->update([
        'u.active' => false,
    ]);
```

---

# 58. Builder semantics

Esto representa:

```text
Update target:
users AS u

Additional relation:
accounts AS a

Join predicate:
a.user_id = u.id

Filter:
a.suspended = TRUE

Assignment:
u.active = FALSE
```

No representa todavía una sintaxis SQL específica.

---

# 59. Platform differences

Un motor podría compilar usando:

```text
UPDATE ... JOIN ...
```

otro:

```text
UPDATE ... FROM ...
```

otro requerir:

```text
correlated subquery
```

o no soportar una equivalencia segura.

---

# 60. Planner responsibility

El Planner decidirá la estrategia sólo si preserva exactamente la semántica.

---

# 61. Join update cardinality hazards

Una relación source puede producir varias filas para una misma fila target.

Ejemplo:

```text
users 1
  │
  ├── order A
  ├── order B
  └── order C
```

Un UPDATE join puede volverse ambiguo.

---

# 62. Semantic detection

Relation/Constraint Analysis deberá poder clasificar:

```text
ONE_TO_ONE
MANY_TO_ONE
ONE_TO_MANY
MANY_TO_MANY
UNKNOWN
```

respecto al target cuando haya evidencia suficiente.

---

# 63. Unsafe multi-match

Si una assignment depende de una source row y existen múltiples matches sin semántica determinista, VoltStack no deberá elegir una arbitrariamente.

---

# 64. UPDATE FROM model

Podrá existir una API explícita:

```php
DB::update()
    ->table('inventory as i')
    ->from('warehouse_stock as w')
    ->setColumn('i.quantity', 'w.quantity')
    ->whereColumn('w.product_id', 'i.product_id');
```

---

# 65. `from()` is semantic

`from()` representará una source relation adicional.

No significa necesariamente que el SQL final utilice literalmente:

```sql
UPDATE ... FROM
```

---

# 66. Relation source set

```text
RelationSourceSet
├── UpdateTarget
└── AdditionalSourceRelation*
```

será analizado por:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

---

# 67. Assignment source references

Un assignment podrá referirse a:

```text
target relation columns
source relation columns
parameters
functions
subqueries
constants
extension expressions
```

si la semántica correspondiente lo permite.

---

# 68. Assignment target restrictions

El lado izquierdo de un assignment será más restrictivo.

Normalmente deberá resolver a:

```text
writable target column
```

No a cualquier expresión.

---

# 69. Invalid target

Esto será inválido:

```text
LOWER(users.name) = ...
```

como assignment target.

---

# 70. Writable column semantics

Schema-aware resolution determinará:

```text
exists
belongs to target
writable
not generated
not forbidden identity
type
nullability
```

---

# 71. Generated columns

Si:

```text
full_name
→ generated
```

entonces:

```php
->set('full_name', '...')
```

deberá producir un error semántico salvo capability/policy explícita que lo permita.

---

# 72. Identity columns

Modificar:

```text
primary identity
```

puede ser:

```text
allowed
restricted
platform-specific
forbidden
```

según schema/platform/policy.

El Builder no decidirá.

---

# 73. Primary key updates

VoltStack no prohibirá universalmente:

```php
->set('id', $newId)
```

pero podrá clasificarlo como operación sensible.

---

# 74. Security metadata

Modificar:

```text
primary key
tenant key
ownership key
authorization key
credential field
```

podrá elevar:

```text
audit requirements
security classification
diagnostics
```

mediante integraciones superiores.

---

# 75. Multitenancy integration

El Database core no dependerá de Multitenancy.

Una integración podrá transformar:

```php
DB::table('orders')->where('id', $id)->update(...);
```

a un query semántico con:

```text
tenant_id = :tenant
```

de forma explícita y visible.

---

# 76. No Tenant::current()

Prohibido dentro del Builder core:

```php
Tenant::current();
```

---

# 77. Authorization integration

Una policy superior podrá:

- añadir predicates;
- rechazar columnas modificables;
- añadir audit metadata;
- restringir unbounded updates.

Pero `UpdateQueryBuilder` no dependerá del Authorization System.

---

# 78. Policy transformations

Las transformaciones deberán quedar registradas en Query Provenance.

Ejemplo:

```text
UpdateQuery
   │
   ▼
Authorization Query Policy
   │
   ├── adds owner_id predicate
   └── restricts assignment fields
```

---

# 79. Optimistic locking

VoltStack deberá soportar integración con optimistic locking.

Ejemplo:

```text
users.version = 7
```

update:

```text
SET
version = version + 1

WHERE
id = P1
AND version = P2
```

---

# 80. ORM path

El ORM podrá producir:

```text
OptimisticLockRequirement
├── versionColumn: version
├── expectedVersion: 7
└── nextVersionStrategy: INCREMENT
```

---

# 81. Query representation

Esto deberá bajar al mismo Update Query Model:

```text
Assignments
├── name = P1
└── version = version + 1

Predicate
├── id = P2
└── version = P3
```

más metadata de concurrency.

---

# 82. Builder convenience

Podrá existir una API avanzada:

```php
->expectVersion('version', $version)
```

si resulta útil fuera del ORM.

---

# 83. Optimistic lock success

El Executor/Persistence Engine podrá interpretar:

```text
affectedRows = 0
```

bajo un `OptimisticLockRequirement` como:

```text
OptimisticLockConflict
```

---

# 84. Builder does not interpret affected rows

`UpdateQueryBuilder` no decidirá que cero filas significa lock conflict.

---

# 85. Pessimistic locking

El UPDATE normalmente obtiene locks físicos mediante el motor, pero políticas avanzadas de locking/transacciones pertenecen al Transaction/Concurrency System.

---

# 86. RETURNING

VoltStack deberá soportar:

```php
$rows = DB::table('users')
    ->where('id', $id)
    ->returning('id', 'name', 'updated_at')
    ->update([
        'name' => 'Ana',
    ]);
```

---

# 87. ReturningSpecification

```text
ReturningSpecification
├── id
├── name
└── updated_at
```

---

# 88. Returning semantics

La semántica será:

```text
return values resulting from mutation
```

No:

```text
append SQL keyword RETURNING
```

---

# 89. Capability requirements

Semantic Analysis podrá derivar:

```text
DML.UPDATE.RETURNING
```

---

# 90. Returning expressions

Podrá considerarse:

```php
->returning([
    'id',
    'normalized_name' => DB::expr()->lower('name'),
]);
```

si la plataforma/capabilities lo permiten.

---

# 91. Result expectations

Podrán existir:

```text
AFFECTED_ROWS
RETURNING_ROWS
RETURNING_SCALAR
NONE
```

---

# 92. Internal result

Internamente:

```text
UpdateExecutionResult
├── affectedRows
├── returnedRows?
├── warnings?
└── executionMetadata
```

---

# 93. Public DX

La facade pública podrá devolver:

```text
affected row count
```

en el caso común.

Pero el core mantendrá un resultado tipado.

---

# 94. ORDER BY

Algunos motores permiten ordenamiento en determinados UPDATEs.

El Builder no deberá asumir portabilidad.

---

# 95. Semantic ordering intent

Si VoltStack expone:

```php
->orderBy('created_at')
```

sobre UPDATE, se representará como:

```text
OrderingSpecification
```

y generará capability requirements.

---

# 96. LIMIT

Igualmente:

```php
->limit(100)
```

representará:

```text
MutationLimitSpecification
```

---

# 97. UPDATE LIMIT portability

Semantic Analysis/Planner podrá derivar:

```text
DML.UPDATE.LIMIT
```

y decidir:

```text
native
safe emulation
unsupported
```

---

# 98. No naive limit emulation

No se transformará automáticamente:

```text
UPDATE ... LIMIT 100
```

en múltiples operaciones si cambia:

```text
ordering
locking
atomicity
concurrency
affected row selection
```

---

# 99. Deterministic limited update

Una política estricta podrá exigir `ORDER BY` o una selección determinista cuando `LIMIT` afecte qué filas se modifican.

---

# 100. CTE integration

Un UPDATE podrá depender de CTEs.

Conceptualmente:

```text
WITH candidate_users AS (...)
UPDATE users ...
```

pero el Builder representará:

```text
CTE definitions
+
UpdateQueryModel
```

sin generar SQL.

---

# 101. Recursive CTE

Su disponibilidad dependerá de capabilities.

El Update Builder reutilizará el CTE System general.

---

# 102. Window-derived update

Un source query/CTE podrá usar window functions para seleccionar filas a modificar.

El UPDATE no implementará un Window System separado.

---

# 103. Update using subquery predicate

Ejemplo:

```php
DB::table('users')
    ->whereExists(
        DB::table('orders')
            ->selectRaw('1')
            ->whereColumn('orders.user_id', 'users.id')
    )
    ->update([
        'has_orders' => true,
    ]);
```

deberá usar:

```text
ExistsPredicateNode
```

---

# 104. Query Type System

El sistema reutilizará:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
```

---

# 105. Assignment type compatibility

Para cada assignment:

```text
Target Column Type
        │
        ▼
Compatibility
        ▲
        │
Value Expression Type
```

---

# 106. Example

Schema:

```text
users.age
→ Integer
```

Assignment:

```php
'age' => 'hello'
```

No deberá convertirse silenciosamente mediante coerción PHP.

---

# 107. Explicit coercion

Si el desarrollador desea cast:

```php
->set(
    'age',
    DB::expr()->cast($value, QueryType::integer())
)
```

podrá expresarlo explícitamente.

---

# 108. Domain type compatibility

Si:

```text
users.owner_id
→ Domain<UserId, UUID>
```

y se asigna:

```text
Domain<OrderId, UUID>
```

la política estricta deberá detectar incompatibilidad aunque ambos utilicen UUID físicamente.

---

# 109. Nullability

Si:

```text
email
→ NOT NULL
```

entonces:

```php
->set('email', null)
```

podrá fallar semánticamente antes de ejecución.

---

# 110. Database remains authority

VoltStack no reemplaza los constraints del servidor.

El análisis anticipado mejora:

```text
developer experience
safety
diagnostics
static/offline validation
```

---

# 111. Check constraints

Constraint Analysis podrá inferir algunos conflictos obvios.

Ejemplo conceptual:

```text
CHECK stock >= 0
```

con:

```text
SET stock = -1
```

si la expresión es constante y evaluable.

---

# 112. Conservative constraint reasoning

No deberá intentarse demostrar arbitrariamente constraints complejos.

Cuando no pueda probarse:

```text
UNKNOWN
```

y la DB será autoridad final.

---

# 113. Foreign key assignments

Actualizar una FK:

```text
order.user_id = P1
```

no hará una consulta previa para verificar existencia.

---

# 114. Constraint metadata

Sí podrá conocerse:

```text
user_id
→ references users.id
```

para:

```text
lineage
type inference
dependency analysis
diagnostics
```

---

# 115. Query Semantic Graph

Un UPDATE producirá nodos como:

```text
UpdateQuery
│
├── TargetRelation
├── TargetColumn
├── Assignment
├── ValueExpression
├── Predicate
├── SourceRelation
├── Parameter
├── Constraint
└── Output
```

---

# 116. Assignment graph

Ejemplo:

```text
Parameter(P1)
      │
      ▼
Assignment
      │
      ▼
users.name
```

---

# 117. Self-referential assignment

```text
users.version
      │
      ▼
AddExpression
      │
      ├── users.version
      └── Literal(1)
      │
      ▼
Assignment
      │
      ▼
users.version
```

---

# 118. Source-derived assignment

```text
warehouse_stock.quantity
          │
          ▼
      Assignment
          │
          ▼
 inventory.quantity
```

---

# 119. Data lineage

Semantic Graph deberá preservar:

```text
source column
    │
    ▼
expression
    │
    ▼
target column
```

---

# 120. Dependency Set

Un UPDATE podrá depender de:

```text
target table
target columns
source tables
source columns
constraints
types
functions
operators
CTEs
capabilities
extensions
```

---

# 121. Mutation Set

Podrá derivarse:

```text
QueryMutationSet
├── relation: users
└── columns
    ├── name
    └── active
```

---

# 122. Mutation metadata uses

Será útil para:

```text
cache invalidation
audit
authorization
CDC integrations
telemetry
entity cache invalidation
result cache invalidation
```

---

# 123. Query dependencies vs mutations

Regla:

```text
QueryDependencySet
≠
QueryMutationSet
```

Un UPDATE puede:

```text
read orders
write users
```

---

# 124. Example

```text
Dependencies
├── users.id
├── orders.user_id
└── orders.total

Mutations
└── users.vip
```

---

# 125. Cache integration

Database Query Builder no invalidará caches directamente.

El semantic/execution artifact podrá exponer mutation information a:

```text
Quantum/Cache integration
```

---

# 126. Events integration

El Builder no disparará:

```text
QueryStarted
QueryCompleted
EntityUpdated
```

durante construcción.

---

# 127. Query events

Query execution events pertenecen al Execution/Event integration.

---

# 128. Entity events

Un UPDATE directo:

```php
DB::table('users')->update(...)
```

no deberá fingir que actualizó entidades ORM individuales.

---

# 129. Important ORM distinction

```text
Direct Query UPDATE
≠
Entity-by-Entity ORM Update
```

---

# 130. Bulk direct update

Un UPDATE directo puede modificar 100,000 filas sin cargar 100,000 entidades.

Por tanto no puede emitir automáticamente:

```text
100,000 EntityUpdated events
```

---

# 131. ORM UnitOfWork

Cuando ORM actualice entidades:

```text
UnitOfWork
    │
    ▼
Persistence Planner
    │
    ▼
UpdateQueryModel(s)
```

Los lifecycle events pertenecen al ORM/Persistence layer.

---

# 132. Dirty checking

`UpdateQueryBuilder` no compara snapshots de entidades.

Eso pertenece a:

```text
125_DATABASE_CHANGE_TRACKING_SYSTEM.md
126_DATABASE_ENTITY_SNAPSHOT_SYSTEM.md
```

---

# 133. Persistence planner

El Persistence Engine podrá agrupar cambios compatibles en updates eficientes.

Pero el Query Builder sólo construye queries.

---

# 134. Bulk update

El sistema deberá distinguir dos familias.

```text
Single-statement set-based UPDATE
```

y:

```text
bulk heterogeneous updates
```

---

# 135. Set-based update

Ejemplo:

```php
DB::table('users')
    ->where('last_login_at', '<', $date)
    ->update([
        'status' => 'inactive',
    ]);
```

una sola estructura semántica.

---

# 136. Heterogeneous bulk update

Ejemplo conceptual:

```text
user 1 → status A
user 2 → status B
user 3 → status C
```

requiere estrategias diferentes.

---

# 137. Bulk Update System

La estrategia avanzada pertenecerá principalmente a:

```text
204_DATABASE_BULK_UPDATE_SYSTEM.md
```

---

# 138. Possible bulk strategies

Podrán incluir:

```text
multiple prepared UPDATEs
CASE-based update
VALUES relation join
temporary relation
native bulk mechanism
staging table
platform-specific mechanism
```

---

# 139. Builder does not choose bulk strategy

El Query Builder podrá producir un descriptor semántico adecuado, pero no decidir la estrategia física.

---

# 140. CASE update

Si se utiliza CASE como estrategia, será generado por Planner/Compiler cuando sea semánticamente correcto.

No deberá ser construido manualmente por el API de alto nivel salvo que el desarrollador solicite explícitamente CASE.

---

# 141. Mutation cardinality

Planner podrá considerar:

```text
expected affected rows
boundedness
unique predicates
bulk cardinality
parameter count
assignment complexity
returning requirements
```

---

# 142. Resource governance

Podrán existir límites:

```text
max assignments
max predicates
max joins
max source relations
max parameters
max subquery depth
max expression complexity
max bulk rows
```

---

# 143. Builder budget

Durante construcción:

```text
UpdateBuilderBudget
```

podrá impedir estructuras abusivas antes de fases posteriores.

---

# 144. Parameter explosion

El Builder no deberá expandir indiscriminadamente grandes colecciones en `WHERE IN`.

---

# 145. Collection parameters

Usará el Parameter System:

```text
CollectionParameter
```

y el Planner decidirá:

```text
expanded placeholders
native arrays
VALUES relation
temporary relation
other safe strategy
```

---

# 146. Retry semantics

UPDATE no será automáticamente retry-safe.

---

# 147. Non-idempotent update

Ejemplo:

```text
SET views = views + 1
```

Repetirlo cambia el resultado.

Por tanto:

```text
RetrySafety::UNSAFE
```

normalmente.

---

# 148. Potentially idempotent update

```text
SET active = false
WHERE id = P1
```

puede ser idempotente respecto al estado final.

Pero aún deben considerarse:

```text
triggers
audit side effects
generated values
returning behavior
transaction outcome uncertainty
```

---

# 149. Retry classification

Será una propiedad semántica/policy-aware:

```text
SAFE
CONDITIONALLY_SAFE
UNSAFE
UNKNOWN
```

No un simple atributo basado en `QueryIntent::WRITE`.

---

# 150. Volatility

Assignments con:

```text
random()
sequence()
current timestamp
volatile extension function
```

afectarán retry/replayability analysis.

---

# 151. Transaction requirements

El UPDATE podrá declarar:

```text
TransactionRequirement
```

pero no iniciar una transacción.

---

# 152. Active transaction

Pertenece a:

```text
TransactionContext
```

---

# 153. Transaction affinity

Si existe una transacción activa:

```text
Update Execution
      │
      ▼
Transaction-affined Connection
```

La selección ocurre en Execution/Connection layers.

---

# 154. QueryIntent

UPDATE derivará normalmente:

```text
QueryIntent::WRITE
```

---

# 155. ConnectionIntent

Normalmente:

```text
ConnectionIntent::WRITE
```

---

# 156. Read dependencies

Aunque el UPDATE use source relations/subqueries:

```text
QueryIntent
```

seguirá siendo mutation/write.

---

# 157. Connection routing

El Builder no decidirá:

```text
primary hostname
replica hostname
pool
physical socket
```

---

# 158. Metadata

Ejemplo:

```php
DB::table('users')
    ->label('users.disable')
    ->timeout(seconds: 5)
    ->where('id', $id)
    ->update([
        'active' => false,
    ]);
```

---

# 159. Query metadata

Podrá contener:

```text
QueryLabel
QueryOrigin
QueryIntent
ConnectionIntent
ConsistencyRequirement
TransactionRequirement
StatementTimeout
RetrySafety
SecurityMetadata
AuditRequirement
TelemetryPolicy
SafetyPolicy
```

---

# 160. Metadata ≠ runtime context

Nunca contendrá:

```text
ConnectionLease
active Transaction
PDO
current User object
Tenant object
Span
ResultCursor
```

---

# 161. Sensitive assignments

Campos como:

```text
password_hash
api_token
secret
private_key
```

deberán poder clasificarse como sensibles.

---

# 162. Sensitive predicates

También:

```text
WHERE reset_token = P1
```

podrá marcar `P1` como sensible.

---

# 163. Redaction

Diagnostics:

```text
UPDATE users
SET password_hash = [REDACTED]
WHERE id = [PARAMETER]
```

en vez de valores reales.

---

# 164. Telemetry

Podrá registrar:

```text
query label
logical target
assignment count
predicate complexity
affected-row category
duration
success/failure
boundedness
```

sin exponer valores.

---

# 165. Audit

Mutation information podrá alimentar un sistema de auditoría.

Pero el Builder no será el Audit System.

---

# 166. Query provenance

Podrá registrar:

```text
origin: APPLICATION
builder: UpdateQueryBuilder
policy transformations
security transformations
tenant transformations
```

---

# 167. Raw assignments

Podrá existir escape hatch:

```php
->setRaw(...)
```

pero deberá producir un nodo explícito.

---

# 168. RawAssignmentExpression

Debe estar clasificado como:

```text
RAW
platform-sensitive
optimizer barrier
security-sensitive
```

según corresponda.

---

# 169. Raw predicates

Seguirán las reglas de:

```text
RawPredicate
```

del Predicate System.

---

# 170. Raw SQL does not become trusted

`DB::raw()` no significa:

```text
safe
portable
validated
```

Sólo significa:

```text
explicit escape hatch
```

---

# 171. Identifier injection protection

Esto:

```php
->set($userInputColumn, $value)
```

deberá pasar por identifier validation/policy.

No se interpolará directamente.

---

# 172. Dynamic columns

VoltStack podrá permitir dynamic identifiers cuando sean explícitamente validados.

---

# 173. SQL injection prevention

Los valores:

```text
never become SQL fragments by default
```

Los identifiers:

```text
never become value parameters
```

---

# 174. Schema-aware resolution

Semantic Analysis resolverá:

```text
target relation
target columns
source columns
assignment types
generated state
identity state
nullability
constraints
relations
```

mediante `SchemaView`.

---

# 175. No hidden I/O

Prohibido:

```text
UpdateQueryBuilder
   │
   ▼
information_schema
```

---

# 176. Offline processing

Debe ser posible:

```text
UpdateQueryArtifact
+
CompiledSchemaMetadata
+
OfflineTarget
      │
      ▼
Semantic Analysis
      │
      ▼
Planning
      │
      ▼
Compilation
```

sin conexión.

---

# 177. Capability requirements

Ejemplos:

```text
DML.UPDATE
DML.UPDATE.RETURNING
DML.UPDATE.JOIN
DML.UPDATE.FROM
DML.UPDATE.ORDER_BY
DML.UPDATE.LIMIT
DML.UPDATE.DEFAULT_ASSIGNMENT
DML.UPDATE.CTE
DML.UPDATE.IDENTITY_OVERRIDE
```

---

# 178. Capabilities describe requirements

El Builder no pregunta:

```php
if ($driver === 'mysql')
```

---

# 179. Planner capability resolution

```text
Semantic Update
      │
      ├── requirements
      ▼
Planner
      │
      ├── Target Capability Snapshot
      └── Planning Policy
      │
      ▼
Physical Strategy
```

---

# 180. Portability profile

Un UPDATE podrá ser:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 181. MySQL/MariaDB isolation

Builder core no conocerá:

```text
UPDATE ... JOIN
ORDER BY/LIMIT vendor behavior
```

---

# 182. PostgreSQL isolation

Builder core no conocerá:

```text
UPDATE ... FROM
RETURNING
```

como strings SQL.

---

# 183. SQLite isolation

Builder core no contendrá ramas específicas para sus capacidades/versiones.

---

# 184. Dialect responsibility

```text
Semantic Update
      │
      ▼
Physical Plan
      │
      ▼
Compiler
      │
      ▼
Dialect
      │
      ▼
SQL
```

---

# 185. Validation stages

```text
Builder Construction Validation
        │
        ▼
Structural Validation
        │
        ▼
Semantic Validation
        │
        ▼
Capability Validation
        │
        ▼
Runtime Validation
```

---

# 186. Construction validation

Puede detectar:

```text
missing target
empty assignment set
duplicate assignments
malformed identifier
invalid builder state
conflicting APIs
invalid limit value
invalid ordering structure
```

---

# 187. Structural validation

Puede detectar:

```text
assignment target is not structurally writable
invalid source composition
malformed joins
invalid returning shape
invalid CTE structure
invalid predicate tree
```

---

# 188. Semantic validation

Puede detectar:

```text
unknown target
unknown target column
ambiguous source column
type mismatch
NULL into NOT NULL
generated column update
invalid identity update
invalid correlation
unsafe multi-match source
invalid returning reference
```

---

# 189. Capability validation

Puede detectar:

```text
UPDATE FROM unsupported
UPDATE JOIN unsupported
RETURNING unsupported
LIMIT unsupported
ORDER BY unsupported
required CTE capability unavailable
```

---

# 190. Runtime validation

Puede detectar:

```text
missing bindings
conversion failure
transaction requirement failure
connection failure
deadlock
timeout
constraint violation
optimistic lock conflict
```

---

# 191. Error hierarchy

```text
UpdateQueryBuilderException
├── MissingUpdateTargetException
├── EmptyUpdateAssignmentSetException
├── InvalidUpdateAssignmentException
├── DuplicateUpdateAssignmentException
├── InvalidUpdateTargetException
├── InvalidUpdatePredicateException
├── InvalidUpdateSourceException
├── InvalidUpdateJoinException
├── InvalidUpdateLimitException
├── InvalidUpdateOrderingException
├── InvalidUpdateReturningException
├── UnboundedUpdateRejectedException
├── UpdateBuilderAlreadyFinalizedException
└── UpdateBuilderBudgetExceededException
```

---

# 192. Semantic errors

```text
UnknownUpdateTargetException
UnknownUpdateColumnException
AmbiguousUpdateReferenceException
UpdateTypeMismatchException
GeneratedColumnUpdateException
IdentityColumnUpdateException
InvalidUpdateSourceCardinalityException
UnsupportedUpdateCapabilityException
InvalidUpdateCorrelationException
InvalidUpdateReturningReferenceException
```

---

# 193. Lifecycle

```text
CREATED
   │
   ▼
TARGET_DEFINED
   │
   ▼
ASSIGNMENTS_DEFINED
   │
   ▼
CONDITIONS_DEFINED
   │
   ▼
CONFIGURED
   │
   ▼
FINALIZED
```

o:

```text
FAILED
```

---

# 194. Builder finalization

Una vez finalizado:

```text
UpdateQueryModel
```

será immutable.

---

# 195. Copy semantics

Si existe:

```php
$base = DB::table('users')
    ->where('tenant_id', $tenantId);

$a = $base->copy()->where('id', 1);
$b = $base->copy()->where('id', 2);
```

los estados deberán estar aislados.

---

# 196. No shared mutable bindings

Copies no compartirán:

```text
ParameterRegistry
BindingSetBuilder
PredicateBuilderState
AssignmentBuilderState
JoinBuilderState
ReturningBuilderState
```

---

# 197. Persistent runtime safety

Compatible con:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
long-running CLI
coroutines
```

---

# 198. Shared safe services

Podrán compartirse:

```text
UpdateQueryBuilderFactory
ExpressionFactory
PredicateFactory
IdentifierFactory
frozen extension registries
immutable descriptors
stateless finalizers
```

---

# 199. Operation-local state

Siempre local:

```text
UpdateBuilderState
ParameterRegistry
BindingSetBuilder
temporary predicates
temporary assignments
source map
diagnostics
```

---

# 200. No global current builder

Prohibido:

```php
UpdateQueryBuilder::current();
```

---

# 201. No static parameter counter

Prohibido:

```php
static $parameterId = 0;
```

compartido por worker.

---

# 202. Concurrent queries

```text
Coroutine A
└── UpdateBuilderState A

Coroutine B
└── UpdateBuilderState B
```

deberán ser completamente independientes.

---

# 203. Fingerprints

Se distinguirán:

```text
builder fingerprint
normalized query fingerprint
semantic fingerprint
planning fingerprint
compiled fingerprint
binding-shape fingerprint
execution fingerprint
```

---

# 204. Runtime values

No participarán en:

```text
normalized structural fingerprint
semantic fingerprint
```

salvo especialización explícita de shape cuando corresponda.

---

# 205. Mutation shape fingerprint

Podrá incluir:

```text
target relation
assignment targets
expression structures
predicate shape
source relation shape
returning shape
semantic requirements
```

---

# 206. Optimizer

El Optimizer recibirá un Semantic Query Artifact ya resuelto.

No resolverá columnas ni tipos desde cero.

---

# 207. Possible optimizations

Podrá:

```text
simplify predicates
eliminate redundant expressions
fold safe constants
propagate safe constraints
simplify assignments
optimize subqueries
```

sin cambiar semántica.

---

# 208. Assignment elimination

Ejemplo:

```text
SET active = active
```

podría parecer redundante.

Pero no deberá eliminarse automáticamente si pudiera afectar:

```text
triggers
updated timestamps
row-level effects
affected-row semantics
auditing
```

---

# 209. Mutation optimization conservatism

Regla:

> Las optimizaciones de DML deberán ser más conservadoras que las optimizaciones puramente relacionales de lectura cuando existan efectos observables.

---

# 210. Planner

El Planner decidirá:

```text
physical mutation strategy
join/update strategy
source materialization
bulk strategy
returning strategy
parameter layout
capability emulation
```

---

# 211. Compiler

El Compiler recibirá un plan.

Será responsable de:

```text
SQL syntax
dialect-specific UPDATE form
placeholder allocation
assignment SQL
join/from syntax
returning syntax
limit/order syntax
```

---

# 212. Executor

El Executor será responsable de:

```text
connection acquisition
statement preparation
binding
execution
result extraction
affected rows
returned rows
error translation
cleanup
```

---

# 213. Builder terminal API

Podrá existir:

```php
$query->execute();
```

pero internamente:

```text
execute()
   │
   ▼
Finalize
   │
   ▼
QueryExecutionRequest
   │
   ▼
Query Engine
```

---

# 214. Builder no contiene Executor físico

Preferiblemente utilizará:

```text
QuerySubmissionGateway
```

o una abstracción equivalente.

---

# 215. Build without execution

Debe soportarse:

```php
$artifact = DB::update()
    ->table('users')
    ->set('active', false)
    ->where('id', 10)
    ->toQueryArtifact();
```

---

# 216. Use cases

Esto permitirá:

```text
testing
query inspection
static analysis
semantic validation
plan preview
SQL preview
security review
debug toolbar
administration tools
```

---

# 217. SQL preview

Si existe:

```php
$query->toSql();
```

será una convenience API que invoque explícitamente el Compiler.

No una función propia del Builder.

---

# 218. Extensions

Podrán añadirse:

```text
custom assignment expressions
custom mutation sources
custom mutation hints
custom returning modes
custom safety policies
custom semantic update features
```

---

# 219. Extension descriptor

Deberá declarar:

```text
extension id
version
node kinds
validation support
semantic support
capability requirements
optimizer support
planner support
compiler support
fingerprint impact
portability
security classification
```

---

# 220. Frozen registry

El registry será congelado después de bootstrap.

No se permitirán registros arbitrarios por request.

---

# 221. Extension completeness

Un nuevo nodo UPDATE no podrá llegar a Compiler sin:

```text
validation support
semantic support
planning/compilation support
```

cuando sean requeridos.

---

# 222. Namespace recomendado

```text
VoltStack\Quantum\Database\Query\Builder\Update
```

---

# 223. Estructura propuesta

```text
Query/
└── Builder/
    └── Update/
        ├── Contract/
        │   ├── UpdateQueryBuilderInterface.php
        │   └── UpdateBuilderFinalizerInterface.php
        │
        ├── Core/
        │   ├── UpdateQueryBuilder.php
        │   ├── UpdateBuilderState.php
        │   ├── UpdateBuilderFinalizer.php
        │   └── UpdateBuilderSnapshot.php
        │
        ├── Target/
        │   ├── UpdateTarget.php
        │   ├── UpdateTargetBuilder.php
        │   └── UpdateTargetFactory.php
        │
        ├── Assignment/
        │   ├── Assignment.php
        │   ├── AssignmentSet.php
        │   ├── AssignmentBuilder.php
        │   ├── AssignmentTarget.php
        │   └── DefaultValueExpression.php
        │
        ├── Source/
        │   ├── UpdateSource.php
        │   ├── UpdateSourceSet.php
        │   └── UpdateSourceBuilder.php
        │
        ├── Join/
        │   ├── UpdateJoinBuilder.php
        │   └── UpdateJoinSet.php
        │
        ├── Predicate/
        │   └── UpdatePredicateBuilder.php
        │
        ├── Returning/
        │   ├── ReturningSpecification.php
        │   ├── ReturningBuilder.php
        │   └── UpdateResultExpectation.php
        │
        ├── Ordering/
        │   └── UpdateOrderingSpecification.php
        │
        ├── Limit/
        │   └── UpdateLimitSpecification.php
        │
        ├── Safety/
        │   ├── UpdateSafetyPolicy.php
        │   ├── UnboundedUpdateIntent.php
        │   └── MutationCardinalityClassification.php
        │
        ├── Concurrency/
        │   └── OptimisticLockRequirement.php
        │
        ├── Metadata/
        │   └── UpdateMetadataBuilder.php
        │
        ├── Extension/
        │   ├── UpdateBuilderExtension.php
        │   ├── UpdateExtensionDescriptor.php
        │   └── UpdateBuilderExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── UpdateBuilderDiagnostic.php
        │
        └── Exception/
            ├── UpdateQueryBuilderException.php
            ├── MissingUpdateTargetException.php
            ├── EmptyUpdateAssignmentSetException.php
            ├── InvalidUpdateAssignmentException.php
            ├── DuplicateUpdateAssignmentException.php
            ├── UnboundedUpdateRejectedException.php
            └── UpdateBuilderAlreadyFinalizedException.php
```

---

# 224. Ownership Matrix

| Concepto | Owner |
|---|---|
| Fluent UPDATE API | UpdateQueryBuilder |
| Temporary construction state | UpdateBuilderState |
| Target declaration | Update Target System |
| Assignment construction | Assignment System |
| Expressions | Query Expression System |
| Predicates | Query Predicate System |
| Runtime values | BindingSet |
| Parameter identity | Parameter System |
| Source relations | Relation System |
| JOIN semantics | Relation/Join Resolution |
| Column existence | Schema-Aware Resolution |
| Assignment typing | Query Type Inference |
| Mutation constraints | Constraint Analysis |
| Data lineage | Semantic Graph |
| Unbounded mutation policy | Safety Policy |
| Optimistic lock declaration | Concurrency metadata |
| Physical lock handling | Transaction/Concurrency System |
| Bulk strategy | Planner/Bulk Update System |
| SQL syntax | Compiler/Dialect |
| Native binding | Driver |
| Active transaction | TransactionManager |
| Connection | Connection System |
| Entity dirty checking | ORM/UnitOfWork |
| Cache invalidation | Cache Integration |
| Execution telemetry | Telemetry Integration |

---

# 225. Architectural Invariants

## DB-UPD-001

`UpdateQueryBuilder` nunca generará SQL.

## DB-UPD-002

Nunca concatenará `UPDATE`.

## DB-UPD-003

Nunca concatenará `SET`.

## DB-UPD-004

Nunca realizará identifier quoting.

## DB-UPD-005

Nunca generará placeholders físicos.

## DB-UPD-006

Nunca realizará native parameter binding.

## DB-UPD-007

Nunca dependerá de PDO.

## DB-UPD-008

Construir un UPDATE nunca realizará I/O.

## DB-UPD-009

El target será estructurado.

## DB-UPD-010

Los assignment targets serán estructurados.

## DB-UPD-011

Los valores normales serán parámetros.

## DB-UPD-012

Runtime values permanecerán fuera del AST.

## DB-UPD-013

`NULL ≠ DEFAULT ≠ missing assignment`.

## DB-UPD-014

Assignments serán explícitos.

## DB-UPD-015

Duplicate assignments no serán aceptados silenciosamente.

## DB-UPD-016

Expression assignments reutilizarán Query Expression System.

## DB-UPD-017

Predicates reutilizarán Query Predicate System.

## DB-UPD-018

Subqueries serán Query Artifacts finalizados.

## DB-UPD-019

No se retendrán child builders mutables.

## DB-UPD-020

Builder no resolverá columnas contra schema.

## DB-UPD-021

Builder no inferirá tipos físicos.

## DB-UPD-022

Builder no resolverá nullability.

## DB-UPD-023

Builder no descubrirá generated columns.

## DB-UPD-024

Builder no descubrirá identity semantics.

## DB-UPD-025

Builder no resolverá constraints.

## DB-UPD-026

Builder no elegirá join update strategy.

## DB-UPD-027

`from()` será semántico, no sintáctico.

## DB-UPD-028

JOIN UPDATE será capability-driven.

## DB-UPD-029

Multiple source matches no serán resueltos arbitrariamente.

## DB-UPD-030

Assignment targets deberán resolver a writable target columns.

## DB-UPD-031

UPDATE sin WHERE será clasificado explícitamente.

## DB-UPD-032

Predicate presence no implicará boundedness.

## DB-UPD-033

Mutation boundedness podrá usar semantic constraints.

## DB-UPD-034

Safety policy será explícita.

## DB-UPD-035

Safety override no deshabilitará security.

## DB-UPD-036

RETURNING será semántico.

## DB-UPD-037

Builder no generará `RETURNING`.

## DB-UPD-038

ORDER BY de mutation será capability-driven.

## DB-UPD-039

LIMIT de mutation será capability-driven.

## DB-UPD-040

No habrá emulación de LIMIT que altere semántica.

## DB-UPD-041

CTEs reutilizarán el Query CTE System.

## DB-UPD-042

Type compatibility será responsabilidad semántica.

## DB-UPD-043

No se usarán coerciones PHP como reglas SQL.

## DB-UPD-044

Domain types conservarán identidad.

## DB-UPD-045

Database constraints seguirán siendo autoridad final.

## DB-UPD-046

Constraint analysis será conservador.

## DB-UPD-047

Foreign key existence no se verificará mediante query previa.

## DB-UPD-048

Semantic Graph registrará mutation lineage.

## DB-UPD-049

Dependency Set será distinto de Mutation Set.

## DB-UPD-050

Mutation Set podrá alimentar cache invalidation.

## DB-UPD-051

Builder no invalidará caches directamente.

## DB-UPD-052

Builder no disparará execution events.

## DB-UPD-053

Direct UPDATE no fingirá entity lifecycle events.

## DB-UPD-054

ORM reutilizará el mismo Update Query Engine.

## DB-UPD-055

UnitOfWork será externo al Builder.

## DB-UPD-056

Dirty checking será externo al Builder.

## DB-UPD-057

Bulk strategy será externa al Builder.

## DB-UPD-058

Builder no codificará vendor parameter limits.

## DB-UPD-059

Collection parameters no serán expandidos prematuramente.

## DB-UPD-060

UPDATE no será automáticamente retry-safe.

## DB-UPD-061

Idempotence será analizada semánticamente.

## DB-UPD-062

Volatility afectará retry classification.

## DB-UPD-063

Builder no iniciará transacciones.

## DB-UPD-064

Builder no almacenará active Transaction.

## DB-UPD-065

Builder no almacenará ConnectionLease.

## DB-UPD-066

Builder no seleccionará primary físico.

## DB-UPD-067

WRITE intent será declarativo.

## DB-UPD-068

Metadata no contendrá runtime resources.

## DB-UPD-069

Sensitive bindings serán redactados.

## DB-UPD-070

Sensitive assignments serán clasificables.

## DB-UPD-071

Raw expressions serán explícitas.

## DB-UPD-072

Raw expressions no serán trusted automáticamente.

## DB-UPD-073

Dynamic identifiers pasarán por identifier policy.

## DB-UPD-074

No habrá vendor-name conditionals en Builder core.

## DB-UPD-075

Platform capabilities decidirán disponibilidad semántica.

## DB-UPD-076

Dialect decidirá sintaxis.

## DB-UPD-077

Driver decidirá native binding.

## DB-UPD-078

Validation estará separada por fases.

## DB-UPD-079

Builder validation no sustituirá Semantic Validation.

## DB-UPD-080

Semantic Validation no sustituirá server constraints.

## DB-UPD-081

Finalized UpdateQueryModel será immutable.

## DB-UPD-082

UpdateBuilderState será operation-scoped.

## DB-UPD-083

No existirá global current Update Builder.

## DB-UPD-084

No existirán static parameter counters compartidos.

## DB-UPD-085

Builder copies tendrán estado aislado.

## DB-UPD-086

Bindings no se compartirán entre copies.

## DB-UPD-087

Builder será persistent-runtime safe.

## DB-UPD-088

Builder será coroutine-safe.

## DB-UPD-089

No habrá state leakage entre requests.

## DB-UPD-090

No habrá state leakage entre queue jobs.

## DB-UPD-091

No habrá state leakage entre coroutines.

## DB-UPD-092

Structural fingerprints excluirán runtime values.

## DB-UPD-093

Mutation fingerprints incluirán estructura relevante.

## DB-UPD-094

Builder no contendrá EntityManager.

## DB-UPD-095

Builder no contendrá UnitOfWork.

## DB-UPD-096

Builder no contendrá IdentityMap.

## DB-UPD-097

Builder no contendrá native Statement.

## DB-UPD-098

Builder no contendrá HTTP Request.

## DB-UPD-099

Builder no contendrá current User.

## DB-UPD-100

Builder no contendrá Tenant Entity.

## DB-UPD-101

Builder no contendrá active telemetry Span.

## DB-UPD-102

Builder no usará Service Container como locator.

## DB-UPD-103

Builder no será Optimizer.

## DB-UPD-104

Builder no será Planner.

## DB-UPD-105

Builder no será Compiler.

## DB-UPD-106

Builder no será Executor.

## DB-UPD-107

Builder no será TransactionManager.

## DB-UPD-108

Builder no será ORM Persistence Engine.

## DB-UPD-109

Optimistic locking se expresará mediante estructura/metadata explícita.

## DB-UPD-110

Builder no interpretará affected rows como lock conflict.

## DB-UPD-111

Authorization será integración opcional.

## DB-UPD-112

Multitenancy será integración opcional.

## DB-UPD-113

Tenant predicates deberán ser visibles estructuralmente.

## DB-UPD-114

Policy transformations tendrán provenance.

## DB-UPD-115

Convenience APIs convergerán a UpdateQueryModel.

## DB-UPD-116

`update()` no será un engine separado.

## DB-UPD-117

`increment()` no será un engine separado.

## DB-UPD-118

`decrement()` no será un engine separado.

## DB-UPD-119

Builder podrá finalizarse sin ejecución.

## DB-UPD-120

La ergonomía Laravel-like nunca romperá las fronteras internas.

---

# 226. Anti-patterns

## 226.1 SQL generation

```php
$sql = "UPDATE {$table} SET ...";
```

**Rechazado.**

---

## 226.2 PDO inside Builder

```php
$this->pdo->prepare($sql);
```

**Rechazado.**

---

## 226.3 Vendor branches

```php
if ($driver === 'pgsql') {
    // UPDATE FROM
}
```

**Rechazado.**

---

## 226.4 Hidden schema introspection

```php
$columns = $connection->describe($table);
```

durante construcción.

**Rechazado.**

---

## 226.5 EntityManager dependency

```php
$this->entityManager->...
```

dentro de `UpdateQueryBuilder`.

**Rechazado.**

---

## 226.6 Silent duplicate assignment

```text
SET name = A
SET name = B

→ silently choose B
```

**Rechazado.**

---

## 226.7 PHP coercion

```php
(int) $value
```

como sistema universal de type compatibility.

**Rechazado.**

---

## 226.8 Naive unbounded safety

```text
has WHERE
→ safe
```

**Rechazado.**

---

## 226.9 Naive retry

```text
UPDATE
→ retry on network error
```

**Rechazado.**

---

## 226.10 Direct entity events

```text
UPDATE users WHERE active = true
→ emit UserUpdated for every row
```

**Rechazado.**

---

# 227. Ejemplo completo — UPDATE simple

```php
$result = DB::table('users')
    ->label('users.rename')
    ->where('id', $userId)
    ->update([
        'name' => 'Ana',
        'updated_at' => DB::expr()->currentTimestamp(),
    ]);
```

Builder:

```text
UpdateBuilderState
│
├── TARGET
│   └── users
│
├── ASSIGNMENTS
│   ├── name
│   │   └── P1
│   └── updated_at
│       └── CurrentTimestamp
│
├── PREDICATE
│   └── id = P2
│
└── METADATA
    ├── QueryLabel: users.rename
    ├── QueryIntent: WRITE
    └── ConnectionIntent: WRITE
```

Bindings:

```text
P1 → Ana
P2 → $userId
```

---

# 228. Semantic resolution

Schema:

```text
users
├── id
│   ├── UUID
│   ├── PRIMARY KEY
│   └── NOT NULL
│
├── name
│   ├── String
│   └── NOT NULL
│
└── updated_at
    ├── DateTime
    └── NOT NULL
```

Semantic Analysis:

```text
P1
→ String

P2
→ UUID

CurrentTimestamp
→ DateTime-compatible

id = P2
→ unique constraint
→ cardinality upper bound: 1
```

---

# 229. Semantic Update Artifact

Conceptualmente:

```text
SemanticUpdate
├── target
│   └── RelationSymbol(users)
│
├── assignments
│   ├── ColumnSymbol(name)
│   │   └── Parameter(P1:String)
│   │
│   └── ColumnSymbol(updated_at)
│       └── CurrentTimestamp:DateTime
│
├── predicate
│   └── ColumnSymbol(id:UUID) = P2:UUID
│
├── constraints
│   └── id unique
│
├── mutationSet
│   ├── users.name
│   └── users.updated_at
│
├── dependencies
│   ├── users
│   ├── users.id
│   ├── users.name
│   └── users.updated_at
│
└── cardinality
    └── SINGLE_ROW_EXPECTED
```

---

# 230. Ejemplo — Optimistic locking

```text
Original entity
├── id = U1
├── name = Alice
└── version = 7
```

Cambio:

```text
name = Ana
```

Persistence Engine construye:

```text
UPDATE users

Assignments
├── name = P1
└── version = version + 1

Predicate
├── id = P2
└── version = P3

Concurrency Requirement
└── optimistic version check
```

Bindings:

```text
P1 → Ana
P2 → U1
P3 → 7
```

---

# 231. Result interpretation

```text
affectedRows = 1
        │
        ▼
update succeeded

affectedRows = 0
        │
        ▼
Persistence/Concurrency Layer
        │
        ▼
OptimisticLockConflict
```

No será interpretado por el Builder.

---

# 232. Ejemplo — UPDATE con source relation

```php
DB::update()
    ->table('inventory as i')
    ->from('warehouse_stock as w')
    ->setColumn('i.quantity', 'w.quantity')
    ->whereColumn('w.product_id', 'i.product_id')
    ->execute();
```

Semantic representation:

```text
UpdateQuery
│
├── TargetRelation
│   └── inventory AS i
│
├── SourceRelation
│   └── warehouse_stock AS w
│
├── Predicate
│   └── w.product_id = i.product_id
│
└── Assignment
    └── i.quantity = w.quantity
```

---

# 233. Semantic relation analysis

```text
inventory AS i
      │
      │ product_id
      │
      ◄──────────────►
                     │
            warehouse_stock AS w
```

Relation Resolution determinará:

```text
symbol identity
column identity
join relationship
possible multiplicity
nullability
lineage
constraints
```

---

# 234. Planner possibilities

Dependiendo del target:

```text
Semantic Update
      │
      ▼
Planner
      │
      ├── native UPDATE FROM
      ├── native UPDATE JOIN
      ├── safe correlated strategy
      └── unsupported
```

---

# 235. Pipeline completo

```text
Developer
   │
   ▼
UpdateQueryBuilder
   │
   ├── TargetBuilder
   ├── AssignmentBuilder
   ├── ExpressionBuilder
   ├── PredicateBuilder
   ├── Join/Source Builder
   ├── ReturningBuilder
   ├── ParameterRegistry
   └── MetadataBuilder
          │
          ▼
UpdateBuilderFinalizer
          │
          ├────────────────┐
          ▼                ▼
 UpdateQueryModel       BindingSet
          │
          ▼
   UpdateQueryNode
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
          ├── target resolution
          ├── assignment resolution
          ├── source resolution
          ├── predicate resolution
          ├── type inference
          ├── nullability
          ├── writable-column analysis
          ├── relation/join analysis
          ├── constraint analysis
          ├── mutation cardinality
          ├── capability requirements
          └── semantic graph
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
          ├── mutation strategy
          ├── join/from strategy
          ├── bulk strategy
          ├── returning strategy
          ├── limit/order strategy
          └── binding layout
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
UpdateExecutionResult
```

---

# 236. Fórmula maestra

```text
UpdateQueryBuilder
=
Target
+
AssignmentSet
+
Predicate
+
SourceRelations
+
JoinSemantics
+
OrderingIntent
+
LimitIntent
+
ReturningIntent
+
ConcurrencyRequirements
+
SafetyMetadata
+
ParameterDefinitions
+
QueryMetadata
```

---

# 237. Fórmula del artifact

```text
UpdateQueryArtifact
=
Immutable UpdateQueryModel
+
ParameterDefinitionSet
+
QueryMetadata
```

---

# 238. Fórmula semántica

```text
Update Meaning
=
Resolved Target
+
Writable Target Columns
+
Resolved Assignment Expressions
+
Resolved Predicate
+
Resolved Source Relations
+
Type Compatibility
+
Nullability
+
Relation Multiplicity
+
Constraints
+
Mutation Set
+
Dependency Set
+
Cardinality Classification
+
Capability Requirements
+
Data Lineage
```

---

# 239. Fórmula de seguridad

```text
Safe Update
=
Structured Target
+
Structured Assignments
+
Parameterized Values
+
Typed Expressions
+
Explicit Predicates
+
Unbounded Mutation Policy
+
Schema-Aware Resolution
+
Sensitive-Value Redaction
+
Explicit Retry Semantics
+
Resource Governance
```

---

# 240. Fórmula de portabilidad

```text
Portable Update
=
Semantic Mutation Model
-
Vendor SQL Assumptions
+
Capability Requirements
+
Relation Semantics
+
Planner Strategies
+
Dialect Compilation
```

---

# 241. Fórmula de persistent-runtime safety

```text
Persistent-Safe Update Builder
=
Immutable Shared Services
+
Frozen Registries
+
Operation-Scoped Builder State
+
Operation-Scoped Parameter Registry
+
Operation-Scoped Binding State
+
No Global Current Builder
+
No Runtime Resource Retention
+
Deterministic Cleanup
```

---

# 242. Diseño definitivo

VoltStack proporcionará:

```php
DB::table('users')
    ->where('id', $id)
    ->update([
        'name' => 'Ana',
    ]);
```

con una experiencia sencilla, mientras internamente mantendrá:

```text
UpdateQueryBuilder
       │
       ▼
UpdateQueryModel
       │
       ▼
UpdateQueryNode
       │
       ▼
Semantic Query Engine
       │
       ▼
Semantic Update
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

# 243. Conclusión

`UpdateQueryBuilder` será una API de construcción semántica de mutaciones y no un generador de SQL.

Permitirá expresar:

```text
simple updates
expression assignments
increment/decrement
NULL
DEFAULT
subquery assignments
source-relation updates
join-aware updates
CTEs
RETURNING
limited updates
optimistic locking
bulk mutation intent
security/safety metadata
```

sin introducir conocimiento de:

```text
PDO
physical connections
SQL vendor syntax
EntityManager
UnitOfWork
active transactions
runtime users
runtime tenants
```

La separación definitiva será:

```text
Builder
   │
   ▼
"What should change?"
   │
   ▼
Semantic Engine
   │
   ▼
"What does that mutation mean?"
   │
   ▼
Optimizer
   │
   ▼
"What can be simplified safely?"
   │
   ▼
Planner
   │
   ▼
"How should it be performed?"
   │
   ▼
Compiler
   │
   ▼
"How is that plan represented for this dialect?"
   │
   ▼
Executor
   │
   ▼
"Perform it."
```

La regla central será:

> **`UpdateQueryBuilder` define la intención de modificación; Semantic Analysis determina el significado de target, assignments, predicates y relaciones; Constraint Analysis determina propiedades relevantes; Planner selecciona una estrategia física; Compiler produce SQL y Executor realiza la mutación.**

Esto permitirá mantener simultáneamente:

```text
Laravel-like DX
+
Doctrine-like architectural boundaries
+
VoltStack semantic query engine
+
persistent-runtime safety
+
multi-platform portability
+
advanced mutation capabilities
```

sin convertir el Builder en un subsistema monolítico.

---

# 244. Siguiente documento

```text
47_DATABASE_DELETE_QUERY_BUILDER.md
```

El siguiente documento deberá definir:

```text
DeleteQueryBuilder
target resolution
predicate integration
DELETE without WHERE safety
bounded/unbounded deletion
join-aware deletes
USING/source relations
subqueries
CTEs
RETURNING
ORDER BY/LIMIT portability
soft-delete boundary
cascade implications
constraint dependencies
mutation/dependency sets
bulk deletion
retry semantics
transaction integration
ORM integration
security
audit
persistent-runtime safety
```

manteniendo:

```text
DeleteQueryBuilder
      │
      ▼
DeleteQueryModel
      │
      ▼
DeleteQueryNode
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
Compiler
      │
      ▼
Executor
```