# 49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md

# VoltStack Quantum Database
## Subquery and Common Table Expression System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 49 — Subquery and CTE System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Query Composition  
**Versión:** 1.0

---

# 1. Propósito

`Subquery and CTE System` define la arquitectura mediante la cual VoltStack podrá utilizar queries completas como componentes estructurados de otras queries.

El sistema deberá soportar:

```text
Scalar Subqueries
Row Subqueries
Table Subqueries
Predicate Subqueries
EXISTS / NOT EXISTS
IN / NOT IN Subqueries
Derived Tables
Correlated Subqueries
Lateral Subqueries
Common Table Expressions
Recursive CTEs
CTE Dependency Graphs
CTE Materialization Intent
```

sin convertir prematuramente estas estructuras en SQL.

La regla fundamental será:

```text
Subquery
≠
SQL fragment
```

y:

```text
CTE
≠
WITH string
```

Ambos serán objetos semánticos estructurados dentro del Query Model y del Query AST.

---

# 2. Posición arquitectónica

```text
Application
    │
    ▼
Query Builder
    │
    ├── Select Builder
    ├── Insert Builder
    ├── Update Builder
    ├── Delete Builder
    └── Subquery / CTE System
              │
              ▼
        Query Model / AST
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
       ┌──────┼───────┐
       ▼      ▼       ▼
    Scopes  Symbols  Types
       │      │       │
       └──────┼───────┘
              ▼
     Correlation Resolution
              │
              ▼
      Dependency Analysis
              │
              ▼
      Constraint Analysis
              │
              ▼
      Semantic Query Graph
              │
              ▼
          Optimizer
              │
      ┌───────┼─────────┐
      ▼       ▼         ▼
  Inline  Decorrelate Materialize
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

# 3. Regla maestra

```text
Subquery Builder
≠
Subquery Resolver
≠
Subquery Optimizer
≠
Subquery Executor
```

y:

```text
CTE Builder
≠
CTE Materializer
≠
CTE Planner
≠
WITH Compiler
```

El Builder declara estructura.

Semantic Analysis determina significado.

Optimizer determina transformaciones equivalentes.

Planner determina estrategia.

Compiler genera sintaxis.

Executor ejecuta.

---

# 4. Relación con documentos anteriores

Este documento depende especialmente de:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
26_DATABASE_QUERY_AST_NODE_MODEL.md
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
31_DATABASE_QUERY_METADATA_SYSTEM.md
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
```

---

# 5. Objetivos

El sistema deberá proporcionar:

- subqueries como objetos estructurados;
- CTEs como definiciones estructuradas;
- scopes explícitos;
- correlación explícitamente resoluble;
- aliases tipados;
- output relations;
- output columns;
- parameter sharing seguro;
- recursive CTEs;
- dependency graphs;
- cycle validation;
- capability requirements;
- materialization intent;
- portability;
- provenance;
- fingerprints;
- extensibilidad;
- budgets;
- serialización segura;
- persistent-runtime safety.

---

# 6. No responsabilidades

Este sistema no:

```text
genera SQL
abre conexiones
ejecuta subqueries
materializa CTEs
elige temporary tables
elige join algorithms
realiza schema introspection
hidrata entidades
administra UnitOfWork
elige índices
decide costos físicos
```

---

# 7. Taxonomía general

VoltStack distinguirá:

```text
Subquery
├── Scalar Subquery
├── Row Subquery
├── Table Subquery
├── Predicate Subquery
├── EXISTS Subquery
├── IN Subquery
├── Derived Table
├── Correlated Subquery
└── Lateral Subquery

CTE
├── Non-Recursive CTE
├── Recursive CTE
├── Data-producing CTE
└── Extension CTE
```

Estas categorías pueden solaparse semánticamente.

Por ejemplo:

```text
Correlated Scalar Subquery
```

es simultáneamente:

```text
Scalar
+
Correlated
```

---

# 8. Query as composable structure

Una query completa podrá convertirse en un componente de otra.

Ejemplo:

```php
$activeUsers = DB::table('users')
    ->select('id')
    ->where('active', true);

$query = DB::table('orders')
    ->whereIn('user_id', $activeUsers);
```

Conceptualmente:

```text
SelectQuery
│
├── FROM orders
│
└── WHERE
    └── InSubqueryPredicate
        ├── orders.user_id
        └── Subquery
            ├── SELECT users.id
            └── WHERE users.active = P1
```

---

# 9. Builder ownership

Una subquery construida mediante otro Builder deberá finalizarse antes de incorporarse al parent query.

```text
Child QueryBuilder
       │
       ▼
Child QueryBuildArtifact
       │
       ▼
SubqueryExpression / SubquerySource
       │
       ▼
Parent QueryBuilder
```

---

# 10. Regla de inmutabilidad

Prohibido:

```text
Parent Query Artifact
└── mutable Child QueryBuilder
```

Permitido:

```text
Parent Query Artifact
└── immutable Child Query Artifact
```

---

# 11. SubqueryId

Cada instancia de subquery tendrá:

```text
SubqueryId
```

Ejemplo:

```text
SQ1
SQ2
SQ3
```

Dos usos de la misma query estructural pueden ser dos instancias semánticas diferentes.

---

# 12. Query identity ≠ structural equality

Dos subqueries pueden tener estructura equivalente:

```text
SQ1 ≡ SQ2 structurally
```

sin compartir:

```text
SubqueryId
```

---

# 13. SubqueryKind

Modelo conceptual:

```php
enum SubqueryKind
{
    case SCALAR;
    case ROW;
    case TABLE;
    case EXISTS;
    case IN;
    case DERIVED_TABLE;
    case LATERAL;
    case EXTENSION;
}
```

La clasificación definitiva puede derivarse del contexto.

---

# 14. Scalar Subquery

Una scalar subquery produce conceptualmente:

```text
0 or 1 row
×
1 column
```

Ejemplo:

```php
$query = DB::table('users as u')
    ->select(
        'u.id',
        DB::subquery(
            DB::table('orders as o')
                ->selectRaw('MAX(o.created_at)')
                ->whereColumn('o.user_id', 'u.id')
        )->as('last_order_at')
    );
```

---

# 15. Representación

```text
ProjectionItem
└── ScalarSubqueryExpression
    └── SelectQueryArtifact
```

---

# 16. Scalar cardinality

El Builder no probará:

```text
at most one row
```

Semantic/Constraint Analysis podrá determinarlo.

---

# 17. Runtime cardinality violation

Si una scalar subquery devuelve múltiples filas y la plataforma considera eso inválido, la responsabilidad final corresponde al motor de base de datos o a validaciones demostrables anteriores.

VoltStack no fingirá conocer cardinalidad que no puede demostrar.

---

# 18. Scalar output width

Una scalar subquery deberá producir exactamente una columna semántica.

Esto será validado después de resolver el output relation.

---

# 19. Row Subquery

Una row subquery produce:

```text
single logical row
+
multiple columns
```

y puede participar en comparaciones de tuples cuando la plataforma/capabilities lo permitan.

---

# 20. Ejemplo conceptual

```text
(a, b)
=
(
    SELECT x, y
    FROM ...
)
```

---

# 21. Row output compatibility

Type Inference/Semantic Analysis deberá verificar:

```text
left arity
=
right arity
```

y compatibilidad de tipos por posición.

---

# 22. Table Subquery

Una table subquery produce una relación.

Ejemplo:

```php
$totals = DB::table('order_items')
    ->select('order_id')
    ->selectAggregate('sum', 'amount', as: 'total')
    ->groupBy('order_id');

$query = DB::query()
    ->fromSub($totals, 'totals')
    ->select('totals.order_id', 'totals.total');
```

---

# 23. Derived table

Una subquery utilizada en `FROM` será modelada como:

```text
DerivedTableSource
```

---

# 24. DerivedTableSource

Conceptualmente:

```php
final readonly class DerivedTableSource
{
    public function __construct(
        public SubqueryId $id,
        public QueryArtifact $query,
        public RelationAlias $alias,
        public ?ColumnAliasList $columnAliases,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 25. Alias

Las derived tables normalmente requerirán alias.

```text
Derived Table
+
Alias
→
Query Relation Instance
```

---

# 26. Output relation

Semantic Analysis producirá:

```text
DerivedRelation
├── RelationId
├── alias
├── output columns
├── types
├── nullability
├── lineage
├── constraints
└── dependencies
```

---

# 27. Derived output columns

Cada projection del child query podrá convertirse en:

```text
RelationOutputColumn
```

visible para el parent scope.

---

# 28. Important distinction

```text
Child QueryOutputColumn
≠
Parent RelationOutputColumn
```

aunque exista lineage entre ambos.

---

# 29. Column alias list

Podrá soportarse:

```text
derived AS d(a, b, c)
```

mediante:

```text
ColumnAliasList
```

sin generar la sintaxis directamente.

---

# 30. Alias arity validation

Si se declaran aliases:

```text
number of aliases
=
number of output columns
```

deberá validarse cuando el output sea conocido.

---

# 31. EXISTS

Ejemplo:

```php
DB::table('users as u')
    ->whereExists(function ($query) {
        $query
            ->from('orders as o')
            ->selectLiteral(1)
            ->whereColumn('o.user_id', 'u.id');
    });
```

---

# 32. Representación EXISTS

```text
ExistsPredicate
└── Subquery
    ├── FROM orders AS o
    └── WHERE o.user_id = u.id
```

---

# 33. EXISTS output projection

La semántica de `EXISTS` depende de existencia de filas, no del valor concreto de las projections.

El Optimizer podrá simplificar projections si es seguro.

El Builder no realizará dicha transformación.

---

# 34. NOT EXISTS

```php
->whereNotExists(...)
```

produce:

```text
NotExistsPredicate
```

o una representación semánticamente equivalente explícita.

---

# 35. EXISTS ≠ COUNT

VoltStack no convertirá automáticamente:

```text
EXISTS(subquery)
```

a:

```text
COUNT(*) > 0
```

en el Builder.

---

# 36. Razón

Las dos formas pueden tener:

```text
different optimization characteristics
different short-circuit opportunities
different physical plans
```

---

# 37. IN Subquery

```php
->whereIn(
    'user_id',
    DB::table('active_users')->select('id')
)
```

produce:

```text
InSubqueryPredicate
├── expression
└── subquery
```

---

# 38. IN subquery arity

Para una expresión escalar:

```text
IN subquery
```

el child output deberá tener una columna.

---

# 39. Tuple IN

Podrá soportarse:

```text
(a, b) IN (
    SELECT x, y
)
```

mediante arity-compatible row expressions.

---

# 40. NOT IN

`NOT IN` deberá preservar SQL 3VL.

---

# 41. Critical NULL rule

```text
x NOT IN (subquery)
```

no deberá transformarse ingenuamente a:

```text
NOT EXISTS(...)
```

sin probar las condiciones necesarias respecto a NULL.

---

# 42. Predicate System ownership

La semántica detallada de:

```text
IN
NOT IN
EXISTS
NOT EXISTS
```

continúa perteneciendo a:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

Este documento define su composición con child queries.

---

# 43. Correlated Subquery

Una subquery es correlated cuando referencia símbolos de un outer scope.

Ejemplo:

```text
SELECT u.id
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

---

# 44. Correlation is semantic

El Builder puede contener:

```text
ColumnReference("u.id")
```

dentro del child query.

No deberá decidir por sí mismo si es:

```text
local
outer
invalid
ambiguous
```

---

# 45. Symbol Resolution

`37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md` resolverá el símbolo.

---

# 46. CorrelationDescriptor

Una referencia externa resuelta producirá:

```php
final readonly class CorrelationDescriptor
{
    public function __construct(
        public CorrelationId $id,
        public ScopeId $innerScope,
        public ScopeId $outerScope,
        public SymbolId $outerSymbol,
        public int $depth,
        public SemanticProvenance $provenance,
    ) {}
}
```

---

# 47. Correlation depth

Ejemplo:

```text
Q0
└── Q1
    └── Q2
        └── reference Q0.symbol
```

puede tener:

```text
correlation depth = 2
```

---

# 48. Correlation ≠ execution

Regla:

```text
Correlated Subquery
≠
Execute Once Per Outer Row
```

Eso es una posible estrategia física, no la semántica.

---

# 49. Decorrelation

Optimizer podrá intentar transformar:

```text
Correlated EXISTS
```

en estructuras como:

```text
Semi Join
```

cuando sea semánticamente seguro.

---

# 50. Decorrelation ownership

```text
Builder
→ declares structure

Semantic Engine
→ resolves correlation

Optimizer
→ decorrelates

Planner
→ chooses physical strategy
```

---

# 51. Correlation graph

Semantic Analysis podrá construir:

```text
CorrelationGraph
```

Ejemplo:

```text
Scope S0
  ▲
  │ correlation
  │
Scope S1
  ▲
  │ correlation
  │
Scope S2
```

---

# 52. Illegal correlation

Una referencia a un scope no visible deberá producir:

```text
InvalidCorrelationException
```

o diagnóstico semántico equivalente.

---

# 53. Lateral subqueries

Una derived table normal no necesariamente puede referenciar relations anteriores del mismo FROM scope.

Para ello existe semánticamente:

```text
LATERAL
```

---

# 54. Example

```text
orders AS o
LEFT LATERAL JOIN
(
    latest payment referencing o.id
)
```

---

# 55. Lateral specification

```php
final readonly class LateralSpecification
{
    public function __construct(
        public LateralMode $mode,
    ) {}
}
```

---

# 56. Lateral capability

La feature deberá declarar:

```text
QUERY.SUBQUERY.LATERAL
```

o capability equivalente.

---

# 57. Subquery scope

Cada child query introduce normalmente un nuevo:

```text
ScopeId
```

---

# 58. Scope hierarchy

```text
Query Scope S0
│
├── Derived Table Scope S1
│
└── EXISTS Scope S2
    │
    └── Nested Scope S3
```

---

# 59. Visibility

No todos los símbolos de S0 son necesariamente visibles desde S1.

La visibilidad depende del contexto y correlation mode.

---

# 60. ScopeGraph

La estructura será registrada en:

```text
ScopeGraph
```

definido por Semantic Query Engine.

---

# 61. Subquery parameters

Una child query puede contener parameters.

---

# 62. Parameter identity

Al incorporar una child query, VoltStack deberá evitar colisiones entre:

```text
Parent P1
Child P1
```

---

# 63. Scoped parameter identity

Una estrategia conceptual:

```text
ParameterId
├── QueryScopeId
└── LocalParameterSequence
```

o IDs opacos globalmente únicos dentro del QueryBuildArtifact final.

---

# 64. No global allocator

Prohibido:

```php
static int $parameter = 0;
```

---

# 65. Parameter remapping

Durante composición podrá existir:

```text
ParameterRemapper
```

que transforme identities de child artifacts de manera determinista.

---

# 66. Values remain external

Aunque parameters sean remapped:

```text
ParameterDefinition
≠
Binding Value
```

---

# 67. Sensitive parameters

Sensitivity metadata deberá sobrevivir:

```text
child query
→ composition
→ semantic artifact
→ compilation
→ telemetry
```

---

# 68. Subquery metadata

Podrá existir:

```text
SubqueryMetadata
├── label
├── provenance
├── optimization intent
├── security classification
├── portability intent
└── extension metadata
```

---

# 69. Common Table Expressions

Una CTE define una query nombrada visible dentro de determinado query scope.

Ejemplo:

```php
$activeUsers = DB::table('users')
    ->select('id', 'name')
    ->where('active', true);

$query = DB::query()
    ->with('active_users', $activeUsers)
    ->from('active_users')
    ->select('*');
```

---

# 70. Structural representation

```text
SelectQuery
├── CTE Set
│   └── active_users
│       └── SelectQueryArtifact
│
└── FROM
    └── active_users
```

---

# 71. CteId

Cada CTE tendrá:

```text
CteId
```

independiente de su nombre.

---

# 72. CTE name ≠ identity

```text
CteName("active_users")
≠
CteId
```

---

# 73. CteDefinition

Modelo conceptual:

```php
final readonly class CteDefinition
{
    public function __construct(
        public CteId $id,
        public CteName $name,
        public QueryArtifact $query,
        public ?ColumnAliasList $columns,
        public CteRecursionMode $recursion,
        public CteMaterializationIntent $materialization,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 74. CteSet

La query principal contendrá:

```text
CteSet
```

ordenado estructuralmente.

---

# 75. CTE scope

Una CTE sólo será visible donde las reglas semánticas lo permitan.

---

# 76. CTE symbol

Symbol Resolution producirá:

```text
CteSymbol
```

---

# 77. CTE reference

Una referencia:

```text
FROM active_users
```

no será clasificada por string matching en Builder.

---

# 78. Resolution order

Semantic Analysis determinará si:

```text
active_users
```

corresponde a:

```text
CTE
schema table
view
extension relation
```

siguiendo las reglas de namespace y scope.

---

# 79. CTE output

Cada CTE produce una relación lógica:

```text
CteRelation
├── CteId
├── RelationId
├── OutputColumns
├── Types
├── Nullability
├── Constraints
├── Lineage
└── Dependencies
```

---

# 80. Column aliases

Ejemplo conceptual:

```text
WITH totals(order_id, total) AS (...)
```

será:

```text
CteDefinition
└── ColumnAliasList
    ├── order_id
    └── total
```

---

# 81. CTE alias arity

Cuando se conozca el output:

```text
CTE column alias count
=
CTE output width
```

---

# 82. Multiple CTEs

```php
$query
    ->with('active_users', $users)
    ->with('recent_orders', $orders)
    ->with('totals', $totals);
```

produce:

```text
CteSet
├── C1 active_users
├── C2 recent_orders
└── C3 totals
```

---

# 83. CTE dependencies

Una CTE podrá depender de otra.

Ejemplo:

```text
A
↓
B
↓
C
```

---

# 84. CteDependencyGraph

Semantic Analysis construirá:

```text
CteDependencyGraph
```

---

# 85. Example graph

```text
active_users
     │
     ▼
recent_orders
     │
     ▼
user_totals
```

---

# 86. Dependency ≠ textual order

El orden estructural declarado puede ayudar a diagnóstico, pero el significado deberá expresarse mediante dependencies explícitas.

---

# 87. Forward references

La política sobre referencias a CTEs declaradas posteriormente deberá ser explícita.

V1 deberá preferir reglas portables y predecibles.

---

# 88. Recommended V1

Por default:

```text
Non-recursive CTE
may reference
previously visible CTEs
```

y no asumir forward-reference portability.

---

# 89. Capability-specific behavior

Si una plataforma soporta semántica más amplia, deberá expresarse mediante capability descriptors, no vendor conditionals.

---

# 90. Recursive CTE

VoltStack deberá modelar recursividad como propiedad semántica.

---

# 91. Recursive CTE architecture

Una recursive CTE normalmente contiene:

```text
Anchor Member
+
Recursive Member
```

combinados mediante una operación de conjunto.

---

# 92. Conceptual example

```text
tree
=
anchor query
UNION ALL
recursive query referencing tree
```

---

# 93. No SQL string

No se almacenará:

```text
"WITH RECURSIVE tree AS (...)"
```

---

# 94. RecursiveCteDefinition

Conceptualmente:

```php
final readonly class RecursiveCteDefinition
{
    public function __construct(
        public CteId $id,
        public CteName $name,
        public QueryArtifact $anchor,
        public QueryArtifact $recursiveMember,
        public RecursiveSetOperator $operator,
        public ?ColumnAliasList $columns,
        public CteMaterializationIntent $materialization,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 95. RecursiveSetOperator

V1:

```php
enum RecursiveSetOperator
{
    case UNION;
    case UNION_ALL;
}
```

La arquitectura deberá permitir extensiones cuando tengan sentido.

---

# 96. Anchor member

El anchor member:

```text
must not depend recursively on the CTE
```

salvo semántica de extensión explícita.

---

# 97. Recursive member

El recursive member puede referenciar:

```text
its recursive CTE symbol
```

bajo reglas controladas.

---

# 98. Recursive reference

Se representará semánticamente mediante:

```text
RecursiveCteReference
```

no mediante detección textual tardía.

---

# 99. Recursive scope

La recursive CTE requiere un scope especial donde su símbolo sea visible al recursive member.

---

# 100. Recursive scope model

```text
Outer Query Scope
      │
      ▼
CTE Definition Scope
      │
      ├── Anchor Scope
      │
      └── Recursive Scope
             │
             └── Recursive CTE Symbol visible
```

---

# 101. Recursive output compatibility

Anchor y recursive member deberán producir outputs compatibles.

---

# 102. Compatibility dimensions

Se validará:

```text
column count
types
domain compatibility
nullability
coercion requirements
collation where relevant
extension semantics
```

---

# 103. Type inference

El tipo final de cada CTE output column podrá requerir resolver:

```text
Anchor Type
+
Recursive Member Type
→
Common Query Type
```

---

# 104. Fixpoint

Recursive type/semantic analysis podrá requerir:

```text
bounded fixpoint
```

---

# 105. No unbounded semantic recursion

El analyzer deberá tener:

```text
iteration budgets
cycle guards
diagnostics
```

---

# 106. Recursive dependency cycle

Un ciclo intencional de recursive CTE:

```text
tree → tree
```

es distinto de un ciclo ilegal entre CTEs no recursivas.

---

# 107. Cycle classification

```text
CteDependencyCycle
├── LEGAL_RECURSIVE
├── ILLEGAL_NON_RECURSIVE
├── MUTUAL_RECURSION
└── EXTENSION_DEFINED
```

---

# 108. Mutual recursion

CTEs como:

```text
A → B
B → A
```

no deberán asumirse válidas.

---

# 109. Recommended V1

VoltStack V1 deberá soportar:

```text
single-CTE self recursion
```

como forma base.

Mutual recursion podrá reservarse para una capability/extensión futura.

---

# 110. Recursive reference count

Algunas plataformas pueden imponer restricciones sobre número o posición de referencias recursivas.

Estas restricciones deberán modelarse como capabilities/validation rules posteriores.

---

# 111. Recursive aggregates

Restricciones sobre:

```text
aggregates
window functions
outer joins
subqueries
ordering
limits
```

dentro del recursive member dependerán de semantic/capability validation.

Builder no codificará reglas por vendor.

---

# 112. Recursion termination

VoltStack no puede demostrar en general que una recursive CTE terminará.

---

# 113. Runtime governance

Podrán existir:

```text
recursion depth limits
statement timeout
resource governance
execution budgets
```

en capas posteriores.

---

# 114. SEARCH / CYCLE extensions

Features avanzadas equivalentes a:

```text
SEARCH
CYCLE
```

podrán modelarse mediante semantic descriptors.

---

# 115. Traversal specification

Conceptualmente:

```text
RecursiveTraversalSpecification
├── strategy
├── ordering keys
├── generated depth/order column
└── cycle handling
```

---

# 116. No vendor syntax

El modelo no deberá almacenar directamente:

```text
SEARCH DEPTH FIRST BY ...
```

---

# 117. CTE materialization

Una CTE puede ser:

```text
inlined
materialized
shared
recomputed
```

dependiendo de optimizer/planner/platform.

---

# 118. Important distinction

```text
CTE
≠
Materialized Temporary Table
```

---

# 119. Materialization intent

El desarrollador podrá opcionalmente expresar:

```text
DEFAULT
PREFER_MATERIALIZED
PREFER_INLINE
REQUIRE_MATERIALIZED
REQUIRE_INLINE
```

si la arquitectura decide exponer estas garantías.

---

# 120. Recommended model

```php
enum CteMaterializationIntent
{
    case DEFAULT;
    case PREFER_MATERIALIZED;
    case PREFER_INLINE;
    case REQUIRE_MATERIALIZED;
    case REQUIRE_INLINE;
}
```

---

# 121. Preference ≠ requirement

```text
PREFER_MATERIALIZED
≠
REQUIRE_MATERIALIZED
```

---

# 122. Planner ownership

El Planner deberá decidir la estrategia real cuando el intent no sea obligatorio.

---

# 123. Capability requirement

Si:

```text
REQUIRE_MATERIALIZED
```

no puede garantizarse en el target, la query deberá fallar de manera explícita.

---

# 124. No silent semantic downgrade

VoltStack no convertirá:

```text
REQUIRE
```

en:

```text
PREFER
```

silenciosamente.

---

# 125. CTE reuse

Ejemplo:

```text
CTE A
├── referenced by query branch B
└── referenced by query branch C
```

---

# 126. Reuse count

Semantic Analysis podrá derivar:

```text
CteReferenceCount
```

como dato para Optimizer/Planner.

---

# 127. Reuse count ≠ materialization

Que una CTE sea referenciada dos veces no obliga automáticamente a materializarla.

---

# 128. Volatility

Si una CTE contiene:

```text
volatile expressions
side-effect-sensitive operations
platform-sensitive functions
```

inlining/materialization puede afectar semántica.

---

# 129. Volatility preservation

Optimizer deberá considerar:

```text
ExpressionVolatility
```

antes de duplicar, inlinear o eliminar una CTE.

---

# 130. Deterministic CTE

Una CTE determinista puede ofrecer más oportunidades de transformación.

---

# 131. CTE in SELECT

Ejemplo:

```text
WITH totals AS (...)
SELECT ...
FROM totals
```

soportado naturalmente.

---

# 132. CTE in INSERT

Podrá existir:

```text
WITH source AS (...)
INSERT ...
SELECT ...
FROM source
```

cuando el target/capabilities lo permitan.

---

# 133. CTE in UPDATE

Podrá existir:

```text
WITH ...
UPDATE ...
```

mediante integración con:

```text
46_DATABASE_UPDATE_QUERY_BUILDER.md
```

---

# 134. CTE in DELETE

Podrá existir:

```text
WITH ...
DELETE ...
```

mediante integración con:

```text
47_DATABASE_DELETE_QUERY_BUILDER.md
```

---

# 135. DML CTEs

Si una plataforma soporta CTEs que contienen DML y producen rows, VoltStack deberá modelarlas mediante una capability explícita.

---

# 136. No assumption

V1 no deberá asumir que:

```text
every CTE body is SELECT
```

a nivel arquitectónico, aunque el soporte inicial pueda limitarlo.

---

# 137. QueryKind compatibility

Cada CTE tendrá:

```text
BodyQueryKind
```

y Semantic Validation verificará compatibilidad con su contexto.

---

# 138. CTE name collision

Dos CTEs con el mismo nombre en el mismo namespace/scope deberán producir error.

---

# 139. Shadowing

Nested scopes podrán shadow CTE names si las reglas semánticas lo permiten.

---

# 140. CTE ≠ temporary table

Una CTE no aparecerá automáticamente en:

```text
schema metadata
database catalog
connection state
```

---

# 141. CTE lifetime

Su lifetime semántico será el query artifact correspondiente.

---

# 142. CTE isolation

No deberá sobrevivir como mutable runtime state entre requests.

---

# 143. Dependency types

Subqueries/CTEs podrán declarar dependencias sobre:

```text
Tables
Views
Columns
Functions
Types
Sequences
Other CTEs
Outer Symbols
Capabilities
Extensions
Schema Objects
```

---

# 144. QueryDependencySet

Estas dependencias terminarán en:

```text
QueryDependencySet
```

del `SemanticQueryArtifact`.

---

# 145. Dependency ≠ lineage

```text
Dependency
```

responde:

```text
¿Qué objetos necesita esta query?
```

`Lineage` responde:

```text
¿De dónde provienen estos datos?
```

---

# 146. Dependency ≠ correlation

Una subquery puede depender de una tabla física sin estar correlated.

---

# 147. Correlation dependency

Cuando depende de un outer symbol:

```text
CorrelationDependency
```

será explícita.

---

# 148. Constraint propagation

Constraint Analysis podrá propagar determinados hechos a través de subquery boundaries cuando sea semánticamente válido.

---

# 149. No unrestricted propagation

Un constraint dentro de:

```text
EXISTS
```

no necesariamente es un constraint global del outer query.

---

# 150. Scope-aware facts

Los derived facts deberán conservar:

```text
scope
certainty
provenance
```

---

# 151. Subquery cardinality facts

Podrán derivarse:

```text
ZERO_OR_ONE
EXACTLY_ONE
AT_MOST_ONE
ONE_OR_MORE
ZERO_OR_MORE
UNKNOWN
```

según constraints.

---

# 152. Scalar subquery proof

Por ejemplo:

```text
aggregate without GROUP BY
```

puede permitir derivar:

```text
AT_MOST_ONE / EXACTLY_ONE
```

dependiendo de semántica concreta.

---

# 153. Unique lookup

Una subquery filtrada por unique key completo puede derivar:

```text
AT_MOST_ONE
```

---

# 154. Builder does not infer cardinality

Toda esta información pertenece a Constraint Analysis.

---

# 155. Semi joins

Optimizer podrá representar:

```text
EXISTS
```

mediante un logical:

```text
SEMI_JOIN
```

---

# 156. Anti joins

`NOT EXISTS` podrá transformarse a:

```text
ANTI_JOIN
```

cuando sea seguro.

---

# 157. SEMI/ANTI are optimizer forms

No necesitan ser JoinType público V1.

---

# 158. IN transformation

`IN subquery` podrá transformarse en:

```text
SEMI_JOIN
```

si:

```text
NULL semantics
duplicates
types
collations
correlation
```

lo permiten.

---

# 159. NOT IN caution

`NOT IN` requiere especial cuidado debido a NULL.

---

# 160. Subquery flattening

Optimizer podrá convertir:

```text
FROM (SELECT ...)
```

en una estructura plana cuando sea equivalente.

---

# 161. Flattening barriers

Pueden impedirlo:

```text
DISTINCT
GROUP BY
HAVING
WINDOWS
LIMIT
OFFSET
LOCKING
VOLATILE EXPRESSIONS
SET OPERATIONS
SECURITY BARRIERS
RAW EXPRESSIONS
EXTENSIONS
```

según contexto.

---

# 162. CTE inlining

Optimizer podrá inlinear una CTE cuando:

```text
semantics are preserved
```

---

# 163. CTE materialization

Planner podrá materializarla cuando sea apropiado y soportado.

---

# 164. Security barrier

Una CTE/subquery podrá portar metadata:

```text
SecurityBarrier
```

para impedir determinadas transformaciones que puedan filtrar información o alterar políticas.

---

# 165. Policy-generated subqueries

Authorization/Multitenancy podrán introducir subqueries explícitas.

Ejemplo conceptual:

```text
WHERE EXISTS (
    permitted_resources ...
)
```

---

# 166. Provenance

Deberán identificarse como:

```text
POLICY_GENERATED
```

o equivalente.

---

# 167. No hidden SQL injection

Una política nunca deberá introducir:

```text
SQL fragment
```

después de Semantic Analysis.

---

# 168. Policy timing

Las transformaciones estructurales de políticas deberán ocurrir antes de cerrar la representación semántica relevante.

---

# 169. Capability model

Capacidades posibles:

```text
QUERY.SUBQUERY.SCALAR
QUERY.SUBQUERY.ROW
QUERY.SUBQUERY.TABLE
QUERY.SUBQUERY.CORRELATED
QUERY.SUBQUERY.LATERAL
QUERY.SUBQUERY.IN
QUERY.SUBQUERY.EXISTS

QUERY.CTE
QUERY.CTE.RECURSIVE
QUERY.CTE.MATERIALIZATION_HINT
QUERY.CTE.MATERIALIZATION_REQUIREMENT
QUERY.CTE.DML_BODY
QUERY.CTE.SEARCH
QUERY.CTE.CYCLE
QUERY.CTE.MUTUAL_RECURSION
```

---

# 170. Capability-driven design

Prohibido:

```php
if ($platform === 'postgresql') {
    // recursive behavior
}
```

en el Builder/Semantic core.

---

# 171. PlatformCapabilitySnapshot

La plataforma será representada mediante:

```text
PlatformCapabilitySnapshot
```

en fases donde sea necesario.

---

# 172. Portability profile

Subqueries/CTEs contribuirán a:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 173. Base portability

Features ampliamente soportadas como:

```text
simple scalar subquery
EXISTS
basic CTE
```

pueden ser altamente portables.

---

# 174. Advanced portability

Features como:

```text
recursive CTE
materialization requirement
lateral correlation
DML CTE
SEARCH/CYCLE
```

requerirán capability analysis más detallado.

---

# 175. Query normalization

Normalization podrá:

```text
canonicalize structural representations
normalize aliases structurally
normalize predicate groups
normalize child artifact representation
normalize metadata ordering
```

---

# 176. Normalization cannot

No podrá:

```text
decorrelate
inline CTEs
materialize CTEs
flatten derived tables for performance
convert EXISTS to JOIN
```

porque son optimizaciones.

---

# 177. Validation

Validation deberá verificar estructura básica antes de Semantic Analysis.

---

# 178. Structural subquery validation

Ejemplos:

```text
child query exists
artifact is finalized
required alias present
column alias list structurally valid
subquery context valid
nesting budget not exceeded
```

---

# 179. Semantic subquery validation

Posteriormente:

```text
scalar width = 1
row arity compatible
IN output compatible
outer reference legal
derived output resolvable
types compatible
```

---

# 180. Structural CTE validation

Ejemplos:

```text
valid CTE name
non-empty body
unique declaration identity
column aliases syntactically valid
recursion structure valid
```

---

# 181. Semantic CTE validation

Incluye:

```text
name resolution
dependency resolution
cycle classification
anchor/recursive compatibility
output type compatibility
recursive references
capability requirements
```

---

# 182. CTE dependency validation

Graph analysis deberá detectar:

```text
illegal cycles
missing references
duplicate names
invalid visibility
unsupported recursion
```

---

# 183. Strongly connected components

`CteDependencyGraph` podrá utilizar análisis de:

```text
Strongly Connected Components
```

para clasificar ciclos.

---

# 184. SCC meaning

Un SCC de tamaño:

```text
1 with self-edge
```

puede representar self recursion.

Un SCC de tamaño:

```text
> 1
```

puede representar mutual recursion.

---

# 185. Graph analysis ≠ vendor behavior

La clasificación lógica será independiente del SQL dialect.

---

# 186. CteGraphNode

Conceptualmente:

```text
CteGraphNode
├── CteId
├── name
├── body kind
├── recursion mode
└── output relation
```

---

# 187. CteGraphEdge

```text
CteGraphEdge
├── source CteId
├── target CteId
├── reference scope
├── recursive?
└── provenance
```

---

# 188. Semantic Query Graph integration

Subqueries y CTEs serán nodos explícitos del:

```text
SemanticQueryGraph
```

---

# 189. Example graph

```text
Main Query Q0
│
├── CTE C1
│   └── Query Q1
│
├── FROM C1
│
└── EXISTS SQ1
    └── Query Q2
        └── Correlation → Q0.user_id
```

---

# 190. Graph edges

Podrán existir:

```text
DEFINES
REFERENCES
CONTAINS
DEPENDS_ON
CORRELATES_WITH
DERIVES_FROM
PRODUCES
CONSUMES
REQUIRES_CAPABILITY
```

---

# 191. Authoritative ownership

La información seguirá distribuida en side tables.

```text
Scope information
→ ScopeGraph

Symbols
→ SymbolTable

Types
→ QueryTypeTable

Relations
→ RelationSemanticTable

Correlations
→ CorrelationTable

Constraints
→ Constraint system

Dependencies
→ QueryDependencySet

Output
→ QueryOutputRelation

Cross-navigation
→ SemanticQueryGraph
```

---

# 192. No giant mutable graph

`SemanticQueryGraph` no se convertirá en un:

```text
array<string, mixed>
```

con duplicación de toda la información.

---

# 193. Fingerprints

Se definirán fingerprints por fase.

---

# 194. Structural fingerprint

Puede incluir:

```text
subquery kind
child query structure
CTE names
CTE structural order
column aliases
recursion declaration
materialization intent
metadata affecting semantics
extension identities
```

---

# 195. Semantic fingerprint

Podrá incluir:

```text
resolved scopes
symbols
relations
types
correlations
dependencies
constraints
capability requirements
schema versions
extension versions
```

---

# 196. Runtime bindings excluded

Los valores runtime normales no deberán alterar el semantic fingerprint.

---

# 197. Sensitive literal handling

Si un literal estructural sí afecta la semántica/fingerprint, deberá utilizarse una representación segura y redactable.

---

# 198. Cache identity

Una query cacheable deberá incluir suficiente información para evitar reutilización entre contextos semánticamente incompatibles.

---

# 199. Tenant-sensitive schema

Si la resolución depende de un schema identity distinto por tenant/context:

```text
SchemaIdentity
```

deberá contribuir al aislamiento del cache.

---

# 200. No Tenant object

Eso no significa almacenar:

```text
Tenant object
```

en el semantic artifact.

---

# 201. Persistent runtime safety

El sistema deberá ser seguro en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 202. Shared immutable state

Podrá compartirse:

```text
Subquery descriptors
CTE descriptors
frozen extension registries
immutable capability descriptors
stateless factories
immutable cached semantic artifacts
```

---

# 203. Operation-local state

Deberá permanecer local:

```text
SubqueryBuilderState
CteBuilderState
Scope builders
Parameter remapping state
Dependency graph builders
Correlation builders
Diagnostics
Fixpoint work queues
```

---

# 204. No global CTE registry

Prohibido:

```php
static array $currentCtes;
```

---

# 205. No global scope stack

Prohibido:

```php
static array $scopeStack;
```

---

# 206. No global correlation context

Prohibido:

```php
static $currentOuterQuery;
```

---

# 207. Serialization safety

Artifacts serializables no deberán contener:

```text
PDO
Connection
Transaction
Statement
Closure
Generator
Fiber
Request
Response
User object
Tenant object
EntityManager
UnitOfWork
Container
Telemetry Span
```

---

# 208. Builder closures

APIs como:

```php
->whereExists(function ($query) {
    ...
})
```

utilizarán la closure únicamente durante construcción.

---

# 209. Closure lifecycle

```text
Closure
   │
   ▼
Child Builder
   │
   ▼
Finalized Child Artifact
   │
   ▼
Closure discarded
```

---

# 210. Extensibility

Podrán existir extensiones para:

```text
custom subquery kinds
custom CTE body kinds
custom recursion models
custom traversal semantics
custom materialization intents
custom dependency types
custom semantic graph nodes
```

---

# 211. Extension descriptor

```text
SubqueryCteExtensionDescriptor
├── id
├── version
├── structural node support
├── validation support
├── semantic support
├── optimizer support
├── planner support
├── compiler support
├── capabilities
├── portability
├── security classification
└── fingerprint contribution
```

---

# 212. Frozen extension registry

```text
SubqueryCteExtensionRegistry
```

se congelará después del bootstrap.

---

# 213. Unknown extension

Una query con una extensión cuyo semantic handler no está disponible deberá fallar explícitamente.

---

# 214. No silent fallback

No se reducirá una extensión desconocida a:

```text
raw SQL
```

automáticamente.

---

# 215. Budgets

El sistema deberá soportar límites para:

```text
subquery count
nesting depth
CTE count
recursive depth analysis
dependency edges
correlation count
parameter count
output width
recursive fixpoint iterations
extension nodes
```

---

# 216. QueryCompositionBudget

Conceptualmente:

```php
final readonly class QueryCompositionBudget
{
    public function __construct(
        public int $maxSubqueries,
        public int $maxSubqueryDepth,
        public int $maxCtes,
        public int $maxDependencyEdges,
        public int $maxCorrelations,
        public int $maxRecursiveIterations,
    ) {}
}
```

---

# 217. Budget failure

Cuando se exceda:

```text
QueryCompositionBudgetExceededException
```

No habrá truncación silenciosa.

---

# 218. Recursion execution limit

El límite de análisis semántico no debe confundirse con:

```text
database recursive execution depth
```

Son dos budgets distintos.

---

# 219. Diagnostics

Los diagnósticos deberán incluir cuando sea posible:

```text
SubqueryId
CteId
CTE name
scope
outer scope
reference
dependency path
cycle path
source location
semantic phase
capability requirement
```

---

# 220. Cycle diagnostic example

```text
Illegal CTE dependency cycle detected:

    active_users
        ↓
    recent_orders
        ↓
    active_users

The CTEs are not declared as a supported
recursive component.

Cycle:
C1 → C2 → C1
```

---

# 221. Correlation diagnostic

```text
Subquery SQ3 references symbol "u.id"
from scope S1, but S1 is not visible
from subquery scope S4.

Correlation requires an explicitly
correlatable context.
```

---

# 222. Scalar diagnostic

```text
Scalar subquery SQ2 produces 3 columns.

Expected:
    1 column

Actual:
    id
    status
    created_at
```

---

# 223. Security

Dynamic CTE/subquery identifiers shall not be treated as bind parameters.

---

# 224. Identifier safety

Un nombre dinámico de CTE deberá pasar por:

```text
IdentifierFactory
```

y las políticas de identifiers.

---

# 225. Runtime values

Los valores continuarán utilizando:

```text
ParameterExpression
+
BindingSet
```

---

# 226. Raw subqueries

No deberá existir una conversión implícita:

```text
string
→
subquery SQL
```

---

# 227. Explicit raw escape hatch

Si se soporta:

```text
RawSubquery
```

deberá ser explícito y portar:

```text
trust classification
portability classification
security metadata
capability requirements
provenance
```

---

# 228. Raw barriers

Un RawSubquery podrá impedir:

```text
symbol analysis
type inference
correlation analysis
constraint analysis
decorrelation
inlining
lineage analysis
dependency extraction
```

parcial o totalmente.

---

# 229. Raw rule

```text
Raw
≠
Semantically understood
```

---

# 230. Public DX

VoltStack deberá permitir ergonomía como:

```php
DB::table('users')
    ->whereExists(function ($query) {
        $query
            ->from('orders')
            ->whereColumn('orders.user_id', 'users.id');
    });
```

sin sacrificar la arquitectura interna.

---

# 231. Explicit reusable query

También:

```php
$subquery = DB::table('orders')
    ->select('user_id')
    ->where('status', 'paid');

DB::table('users')
    ->whereIn('id', $subquery);
```

---

# 232. CTE DX

```php
$orders = DB::table('orders')
    ->select('user_id', 'total')
    ->where('status', 'paid');

$query = DB::query()
    ->with('paid_orders', $orders)
    ->from('paid_orders')
    ->select('*');
```

---

# 233. Recursive DX

Una API posible:

```php
$query->withRecursive(
    'tree',
    anchor: $anchor,
    recursive: $recursive,
    columns: ['id', 'parent_id', 'depth'],
    union: RecursiveSetOperator::UNION_ALL,
);
```

---

# 234. No overloaded magic

Se deberá evitar una API donde:

```php
with(...)
```

acepte cualquier combinación indeterminada de:

```text
string
array
closure
SQL
builder
callable
object
```

sin contracts claros.

---

# 235. Progressive disclosure

La API puede ofrecer shortcuts simples y objetos avanzados para casos complejos.

---

# 236. Query composition interfaces

Propuesta:

```php
interface SubqueryCapableQueryBuilder
{
    public function subquery(QueryBuildArtifact $query): SubqueryReference;
}

interface CteCapableQueryBuilder
{
    public function with(
        string|Identifier $name,
        QueryBuildArtifact $query,
    ): static;
}
```

La API pública final podrá aceptar Builders mediante un adaptador que los finalice de forma segura.

---

# 237. Builder acceptance

Si una API recibe:

```text
QueryBuilder
```

internamente deberá hacer:

```text
Builder
→ snapshot/finalize child
→ immutable artifact
```

sin conservar el Builder.

---

# 238. Parent mutation isolation

Cambiar posteriormente el child builder original no deberá modificar la query parent ya compuesta.

---

# 239. Example invariant

```php
$child = DB::table('users')->select('id');

$parent = DB::table('orders')
    ->whereIn('user_id', $child);

$child->where('active', true);
```

El cambio posterior a `$child` no deberá alterar silenciosamente `$parent`.

---

# 240. Copy/fork semantics

Esto sigue la estrategia definida en:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
```

de:

```text
mutable fluent construction
+
explicit snapshot/finalization
+
immutable query artifacts
```

---

# 241. Directory structure propuesta

```text
Query/
└── Builder/
    └── Composition/
        ├── Subquery/
        │   ├── Contract/
        │   │   ├── SubqueryFactoryInterface.php
        │   │   └── SubqueryFinalizerInterface.php
        │   │
        │   ├── Core/
        │   │   ├── SubqueryId.php
        │   │   ├── SubqueryKind.php
        │   │   ├── SubqueryReference.php
        │   │   ├── SubqueryFactory.php
        │   │   └── SubqueryFinalizer.php
        │   │
        │   ├── Scalar/
        │   │   └── ScalarSubqueryExpression.php
        │   │
        │   ├── Row/
        │   │   └── RowSubqueryExpression.php
        │   │
        │   ├── Table/
        │   │   ├── DerivedTableSource.php
        │   │   └── ColumnAliasList.php
        │   │
        │   ├── Predicate/
        │   │   ├── ExistsSubquery.php
        │   │   └── InSubquery.php
        │   │
        │   ├── Correlation/
        │   │   ├── CorrelationId.php
        │   │   ├── CorrelationDescriptor.php
        │   │   └── CorrelationTable.php
        │   │
        │   └── Lateral/
        │       ├── LateralMode.php
        │       └── LateralSpecification.php
        │
        ├── Cte/
        │   ├── Contract/
        │   │   └── CteFactoryInterface.php
        │   │
        │   ├── Core/
        │   │   ├── CteId.php
        │   │   ├── CteName.php
        │   │   ├── CteDefinition.php
        │   │   ├── CteSet.php
        │   │   └── CteFactory.php
        │   │
        │   ├── Recursive/
        │   │   ├── RecursiveCteDefinition.php
        │   │   ├── RecursiveSetOperator.php
        │   │   ├── RecursiveCteReference.php
        │   │   ├── RecursiveTraversalSpecification.php
        │   │   └── CteRecursionMode.php
        │   │
        │   ├── Materialization/
        │   │   └── CteMaterializationIntent.php
        │   │
        │   ├── Dependency/
        │   │   ├── CteDependencyGraph.php
        │   │   ├── CteDependencyNode.php
        │   │   ├── CteDependencyEdge.php
        │   │   └── CteCycleClassifier.php
        │   │
        │   └── Semantic/
        │       └── CteSemanticTable.php
        │
        ├── Parameter/
        │   └── ParameterRemapper.php
        │
        ├── Budget/
        │   └── QueryCompositionBudget.php
        │
        ├── Extension/
        │   ├── SubqueryCteExtension.php
        │   ├── SubqueryCteExtensionDescriptor.php
        │   └── SubqueryCteExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── QueryCompositionDiagnostic.php
        │
        └── Exception/
            ├── QueryCompositionException.php
            ├── InvalidSubqueryException.php
            ├── InvalidCorrelationException.php
            ├── InvalidCteException.php
            ├── DuplicateCteNameException.php
            ├── InvalidRecursiveCteException.php
            ├── CteDependencyCycleException.php
            └── QueryCompositionBudgetExceededException.php
```

---

# 242. Semantic namespace

La información resuelta deberá permanecer principalmente bajo:

```text
Query/
└── Semantic/
    ├── Scope/
    ├── Symbol/
    ├── Relation/
    ├── Constraint/
    ├── Graph/
    ├── Correlation/
    └── Dependency/
```

evitando duplicar sistemas semánticos dentro del Builder.

---

# 243. Testing strategy

Se deberán cubrir:

```text
scalar subqueries
row subqueries
table subqueries
derived tables
EXISTS
NOT EXISTS
IN
NOT IN
tuple IN
correlated subqueries
nested correlation
illegal correlation
lateral subqueries
subquery aliases
column aliases
parameter remapping
sensitive bindings
CTE declarations
multiple CTEs
CTE references
CTE dependencies
duplicate names
recursive CTE
recursive type inference
recursive cycles
illegal cycles
mutual recursion rejection
materialization intent
capability requirements
raw barriers
security barriers
serialization
fingerprints
budgets
persistent workers
concurrency
```

---

# 244. Structural tests

Builder tests deberán comparar:

```text
QueryBuildArtifact
QueryModel
SubqueryReference
CteDefinition
BindingSet
```

y no SQL.

---

# 245. Semantic tests

Semantic tests deberán verificar:

```text
ScopeGraph
SymbolTable
CorrelationTable
RelationGraph
CteDependencyGraph
QueryTypeTable
ConstraintGraph
SemanticQueryGraph
```

---

# 246. Compiler tests

SQL como:

```text
WITH ...
WITH RECURSIVE ...
EXISTS (...)
LATERAL (...)
```

se probará exclusivamente en Compiler/Dialect tests.

---

# 247. Persistent runtime test

Miles de queries sucesivas dentro del mismo worker deberán demostrar:

```text
no leaked CTEs
no leaked scopes
no leaked correlations
no leaked parameters
no leaked aliases
no leaked dependency nodes
no leaked diagnostics
```

---

# 248. Concurrent runtime test

Dos queries concurrentes podrán usar localmente:

```text
CTE C1
Subquery SQ1
Parameter P1
Scope S1
```

sin conflicto porque sus identities estarán scoped al artifact correspondiente.

---

# 249. Performance

La construcción deberá permanecer aproximadamente proporcional al tamaño estructural:

```text
O(nodes + edges)
```

sin acceso a base de datos.

---

# 250. Dependency analysis

Construir el CTE dependency graph será aproximadamente:

```text
O(V + E)
```

para:

```text
V = CTE nodes
E = dependency edges
```

---

# 251. Cycle detection

Tarjan/Kosaraju o algoritmo equivalente podrá detectar SCCs en:

```text
O(V + E)
```

---

# 252. Semantic fixpoint

Recursive CTE type/constraint inference podrá requerir múltiples iteraciones, siempre bounded.

---

# 253. Resource governance

Una query adversarial no deberá poder crear recursión semántica ilimitada.

---

# 254. Architectural invariants

## DB-SUBCTE-001

Subquery no será SQL string.

## DB-SUBCTE-002

CTE no será WITH string.

## DB-SUBCTE-003

Builder no generará SQL.

## DB-SUBCTE-004

Builder no ejecutará child queries.

## DB-SUBCTE-005

Builder no abrirá conexiones.

## DB-SUBCTE-006

Builder no realizará schema I/O.

## DB-SUBCTE-007

Child artifacts serán immutable.

## DB-SUBCTE-008

Parent artifacts no almacenarán mutable child builders.

## DB-SUBCTE-009

Cada subquery tendrá identidad explícita.

## DB-SUBCTE-010

Structural equality no implicará identity equality.

## DB-SUBCTE-011

Scalar subqueries serán estructuradas.

## DB-SUBCTE-012

Scalar width se validará semánticamente.

## DB-SUBCTE-013

Builder no inventará cardinality proofs.

## DB-SUBCTE-014

Row subqueries preservarán arity.

## DB-SUBCTE-015

Table subqueries producirán relations.

## DB-SUBCTE-016

Derived tables tendrán relation identity propia.

## DB-SUBCTE-017

Derived table output será distinto del child query output identity.

## DB-SUBCTE-018

Lineage conectará ambos outputs.

## DB-SUBCTE-019

Required derived aliases serán validados.

## DB-SUBCTE-020

Column alias lists serán estructuradas.

## DB-SUBCTE-021

EXISTS será un predicate semántico.

## DB-SUBCTE-022

EXISTS no se convertirá a COUNT en Builder.

## DB-SUBCTE-023

NOT EXISTS preservará su semántica.

## DB-SUBCTE-024

IN subquery será estructurado.

## DB-SUBCTE-025

NOT IN preservará SQL 3VL.

## DB-SUBCTE-026

NOT IN no se convertirá ingenuamente a NOT EXISTS.

## DB-SUBCTE-027

Correlation será resuelta mediante scopes/symbols.

## DB-SUBCTE-028

Builder no resolverá outer symbols por strings.

## DB-SUBCTE-029

Correlation tendrá identidad explícita.

## DB-SUBCTE-030

Correlation depth será representable.

## DB-SUBCTE-031

Correlation no determinará physical execution strategy.

## DB-SUBCTE-032

Decorrelation pertenecerá al Optimizer.

## DB-SUBCTE-033

Illegal correlation será error semántico.

## DB-SUBCTE-034

LATERAL será explícito.

## DB-SUBCTE-035

LATERAL será capability-driven.

## DB-SUBCTE-036

Cada child query introducirá scope apropiado.

## DB-SUBCTE-037

Scope visibility será explícita.

## DB-SUBCTE-038

Parameter identities no colisionarán durante composición.

## DB-SUBCTE-039

No habrá global parameter allocator.

## DB-SUBCTE-040

Parameter remapping será determinista.

## DB-SUBCTE-041

Binding values permanecerán separados de parameter definitions.

## DB-SUBCTE-042

Sensitivity metadata sobrevivirá composición.

## DB-SUBCTE-043

Cada CTE tendrá CteId.

## DB-SUBCTE-044

CteName será distinto de CteId.

## DB-SUBCTE-045

CTEs serán definiciones estructuradas.

## DB-SUBCTE-046

CTE bodies serán immutable query artifacts.

## DB-SUBCTE-047

CTE scope será explícito.

## DB-SUBCTE-048

CTE symbols serán resueltos semánticamente.

## DB-SUBCTE-049

Builder no distinguirá CTE/table por simple string guessing.

## DB-SUBCTE-050

CTE output será una relation semántica.

## DB-SUBCTE-051

Duplicate CTE names serán errores de scope.

## DB-SUBCTE-052

CTE dependencies serán explícitas.

## DB-SUBCTE-053

Dependency graph será independiente del SQL dialect.

## DB-SUBCTE-054

Cycles serán clasificados explícitamente.

## DB-SUBCTE-055

Non-recursive illegal cycles fallarán.

## DB-SUBCTE-056

Recursive CTE será explícita.

## DB-SUBCTE-057

Recursive CTE distinguirá anchor y recursive member.

## DB-SUBCTE-058

Anchor no dependerá recursivamente del CTE por default.

## DB-SUBCTE-059

Recursive member podrá referenciar su recursive symbol.

## DB-SUBCTE-060

Recursive reference será semántica, no textual.

## DB-SUBCTE-061

Anchor y recursive member deberán tener outputs compatibles.

## DB-SUBCTE-062

Recursive type inference será bounded.

## DB-SUBCTE-063

Semantic recursion no será ilimitada.

## DB-SUBCTE-064

Self recursion será distinta de illegal cycle.

## DB-SUBCTE-065

Mutual recursion no se asumirá soportada.

## DB-SUBCTE-066

Recursive platform restrictions usarán capabilities.

## DB-SUBCTE-067

Builder no contendrá vendor-specific recursion rules.

## DB-SUBCTE-068

Termination no será asumida.

## DB-SUBCTE-069

Execution recursion limit será distinto del semantic analysis budget.

## DB-SUBCTE-070

SEARCH/CYCLE serán extensiones semánticas.

## DB-SUBCTE-071

CTE no implicará materialization.

## DB-SUBCTE-072

Materialization intent será distinto de physical strategy.

## DB-SUBCTE-073

PREFER será distinto de REQUIRE.

## DB-SUBCTE-074

REQUIRE no será degradado silenciosamente.

## DB-SUBCTE-075

CTE reuse no obligará materialization.

## DB-SUBCTE-076

Optimizer considerará volatility.

## DB-SUBCTE-077

Inlining pertenecerá al Optimizer.

## DB-SUBCTE-078

Materialization strategy pertenecerá al Planner.

## DB-SUBCTE-079

Subquery flattening pertenecerá al Optimizer.

## DB-SUBCTE-080

EXISTS-to-SEMI-JOIN pertenecerá al Optimizer.

## DB-SUBCTE-081

NOT-EXISTS-to-ANTI-JOIN pertenecerá al Optimizer.

## DB-SUBCTE-082

NULL semantics deberán preservarse en transformaciones.

## DB-SUBCTE-083

Security barriers serán explícitas.

## DB-SUBCTE-084

Policy-generated subqueries tendrán provenance.

## DB-SUBCTE-085

No habrá hidden SQL policy injection.

## DB-SUBCTE-086

Capabilities reemplazarán vendor checks.

## DB-SUBCTE-087

Portability será calculable.

## DB-SUBCTE-088

Dependencies serán distintas de lineage.

## DB-SUBCTE-089

Dependencies serán distintas de correlations.

## DB-SUBCTE-090

Constraint propagation será scope-aware.

## DB-SUBCTE-091

Builder no derivará semantic constraints.

## DB-SUBCTE-092

SemanticQueryGraph representará subqueries/CTEs explícitamente.

## DB-SUBCTE-093

Authoritative semantic facts permanecerán en side tables.

## DB-SUBCTE-094

Graph no duplicará indiscriminadamente toda la información.

## DB-SUBCTE-095

Structural fingerprint excluirá runtime bindings.

## DB-SUBCTE-096

Semantic fingerprint incluirá semantic dependencies relevantes.

## DB-SUBCTE-097

Tenant-sensitive schema identity deberá aislar caches.

## DB-SUBCTE-098

Semantic artifacts no almacenarán Tenant objects.

## DB-SUBCTE-099

Builder state será operation-scoped.

## DB-SUBCTE-100

No habrá global CTE registry.

## DB-SUBCTE-101

No habrá global scope stack.

## DB-SUBCTE-102

No habrá global correlation context.

## DB-SUBCTE-103

Closures no sobrevivirán finalization.

## DB-SUBCTE-104

Artifacts no almacenarán runtime resources.

## DB-SUBCTE-105

Shared extension registries serán frozen.

## DB-SUBCTE-106

Extensions serán versionadas.

## DB-SUBCTE-107

Unknown semantic extensions fallarán explícitamente.

## DB-SUBCTE-108

Unknown extensions no caerán silenciosamente a Raw SQL.

## DB-SUBCTE-109

Budgets serán explícitos.

## DB-SUBCTE-110

Budget exhaustion no truncará artifacts.

## DB-SUBCTE-111

Builder composition será deterministic.

## DB-SUBCTE-112

Changing child builder after composition no mutará parent artifact.

## DB-SUBCTE-113

Persistent workers no compartirán mutable composition state.

## DB-SUBCTE-114

Concurrent queries serán state-isolated.

## DB-SUBCTE-115

Compiler será el único responsable de WITH/RECURSIVE syntax.

## DB-SUBCTE-116

Planner será responsable de physical materialization strategy.

## DB-SUBCTE-117

Optimizer será responsable de semantic-preserving query transformations.

## DB-SUBCTE-118

Executor no reinterpretará CTE/subquery semantics.

## DB-SUBCTE-119

ORM reutilizará este mismo Query Engine.

## DB-SUBCTE-120

Subqueries y CTEs serán ciudadanos de primer nivel del Query Model.

---

# 255. Anti-patterns

## 255.1 SQL subquery strings

```php
$where = "id IN (SELECT user_id FROM orders)";
```

**Rechazado como API estructurada principal.**

---

## 255.2 CTE SQL strings

```php
$sql = "WITH users AS (...)";
```

**Rechazado en Builder.**

---

## 255.3 Store child Builder

```text
CteDefinition
└── QueryBuilder mutable
```

**Rechazado.**

---

## 255.4 Global CTE registry

```php
static $ctes = [];
```

**Rechazado.**

---

## 255.5 Global outer query

```php
static $currentOuterQuery;
```

**Rechazado.**

---

## 255.6 Guess correlation

```text
unknown identifier
→ assume outer column
```

**Rechazado.**

La resolución debe seguir scopes y namespaces.

---

## 255.7 Always materialize CTE

```text
CTE
→ temporary table
```

**Rechazado.**

---

## 255.8 Always inline CTE

```text
CTE
→ duplicate body everywhere
```

**Rechazado.**

---

## 255.9 EXISTS → COUNT

```text
EXISTS(Q)
→ COUNT(Q) > 0
```

en Builder.

**Rechazado.**

---

## 255.10 NOT IN → NOT EXISTS

Sin análisis de NULL.

**Rechazado.**

---

## 255.11 Execute correlated subquery per row by definition

**Rechazado.**

Eso sería una decisión física prematura.

---

## 255.12 Vendor checks

```php
if ($driver === 'pgsql') {
    $cte->materialized();
}
```

**Rechazado en core.**

---

# 256. Ejemplo integral

Consideremos:

```php
$paidTotals = DB::table('orders as o')
    ->select('o.user_id')
    ->selectAggregate('sum', 'o.total', as: 'paid_total')
    ->where('o.status', 'paid')
    ->groupBy('o.user_id');

$query = DB::query()
    ->with('paid_totals', $paidTotals)
    ->from('users as u')
    ->leftJoin('paid_totals as pt', function ($join) {
        $join->on('pt.user_id', '=', 'u.id');
    })
    ->whereExists(function ($query) {
        $query
            ->from('sessions as s')
            ->selectLiteral(1)
            ->whereColumn('s.user_id', 'u.id')
            ->where('s.active', true);
    })
    ->select(
        'u.id',
        'u.name',
        'pt.paid_total'
    );
```

---

# 257. Structural representation

```text
SelectQuery
│
├── CTE C1: paid_totals
│   │
│   └── SelectQuery
│       ├── FROM orders AS o
│       ├── SELECT
│       │   ├── o.user_id
│       │   └── SUM(o.total) AS paid_total
│       ├── WHERE o.status = P1
│       └── GROUP BY o.user_id
│
├── FROM users AS u
│
├── LEFT JOIN
│   └── paid_totals AS pt
│       └── ON pt.user_id = u.id
│
├── WHERE
│   └── EXISTS SQ1
│       └── SelectQuery
│           ├── FROM sessions AS s
│           ├── SELECT literal(1)
│           └── WHERE
│               └── AND
│                   ├── s.user_id = u.id
│                   └── s.active = P2
│
└── SELECT
    ├── u.id
    ├── u.name
    └── pt.paid_total
```

Bindings:

```text
P1 → "paid"
P2 → true
```

---

# 258. Scope structure

Semantic Analysis podrá producir:

```text
Scope S0
Main Query
│
├── CTE C1
│   └── Scope S1
│
└── EXISTS SQ1
    └── Scope S2
         │
         └── correlation → S0.u.id
```

---

# 259. Symbol structure

```text
S0
├── u  → users relation
└── pt → paid_totals CTE relation

S1
└── o → orders relation

S2
└── s → sessions relation
```

Con:

```text
S2.s.user_id
=
S0.u.id
```

como correlated predicate.

---

# 260. Dependency graph

```text
Main Query
├── users
├── sessions
└── CTE paid_totals
      └── orders
```

---

# 261. Lineage

```text
pt.paid_total
    │
    ▼
CTE paid_totals.paid_total
    │
    ▼
SUM(orders.total)
    │
    ▼
orders.total
```

---

# 262. Constraint knowledge

Semantic/Constraint Analysis podría derivar:

```text
paid_totals.user_id
→ unique per group

EXISTS SQ1
→ existence dependency on sessions

SQ1
→ correlated with users.id
```

sin ejecutar ninguna query.

---

# 263. Optimizer possibilities

El Optimizer podría considerar:

```text
inline paid_totals
materialize paid_totals
decorrelate EXISTS
convert EXISTS to SEMI_JOIN
push safe predicates
simplify projections
```

siempre preservando semántica.

---

# 264. Planner possibilities

Planner podría elegir:

```text
CTE inline
+
Hash Aggregate orders
+
Hash Join users/totals
+
Semi Join sessions
```

o una estrategia diferente.

---

# 265. Compiler

Sólo en esta fase se decidirá una representación como:

```text
WITH ...
SELECT ...
```

adaptada a:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

y sus capabilities reales.

---

# 266. Arquitectura definitiva

```text
                    Query Composition
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
         Subquery                       CTE
             │                           │
     ┌───────┼────────┐          ┌───────┼─────────┐
     ▼       ▼        ▼          ▼       ▼         ▼
  Scalar   Table   Predicate   Normal Recursive Dependency
     │       │        │          │       │         │
     └───────┴────────┴──────────┴───────┴─────────┘
                           │
                           ▼
                    Immutable Query AST
                           │
                           ▼
                    Semantic Analysis
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
          Scope         Correlation    Dependencies
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                  Semantic Query Graph
                           │
                           ▼
                       Optimizer
                           │
             ┌─────────────┼──────────────┐
             ▼             ▼              ▼
          Inline      Decorrelate      Rewrite
             │             │              │
             └─────────────┼──────────────┘
                           ▼
                        Planner
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
               Inline            Materialize
                 │                   │
                 └─────────┬─────────┘
                           ▼
                        Compiler
                           │
                           ▼
                         SQL
```

---

# 267. Fórmula maestra de Subquery

```text
Subquery
=
Immutable Child Query
+
Subquery Context
+
Scope Boundary
+
Output Contract
+
Parameter Mapping
+
Metadata
```

---

# 268. Fórmula de correlación

```text
Correlation
=
Inner Scope
+
Visible Outer Scope
+
Resolved Outer Symbol
+
Reference Dependency
+
Correlation Depth
+
Provenance
```

---

# 269. Fórmula de CTE

```text
CTE
=
Identity
+
Name
+
Immutable Query Body
+
Output Contract
+
Scope
+
Dependencies
+
Recursion Mode
+
Materialization Intent
+
Metadata
```

---

# 270. Fórmula Recursive CTE

```text
Recursive CTE
=
Anchor Member
+
Recursive Member
+
Recursive Symbol
+
Set Operation
+
Compatible Output Relation
+
Bounded Semantic Fixpoint
+
Capability Requirements
```

---

# 271. Fórmula del Dependency Graph

```text
CTE Dependency Graph
=
CTE Nodes
+
Reference Edges
+
Scope Information
+
Recursion Classification
+
Cycle Classification
```

---

# 272. Fórmula de optimización

```text
Subquery/CTE Optimization
=
Semantic Structure
+
Correlation Knowledge
+
Constraint Knowledge
+
Cardinality Facts
+
Volatility
+
Security Barriers
+
Capability Knowledge
+
Cost Model
```

---

# 273. Fórmula de seguridad

```text
Safe Query Composition
=
Immutable Child Artifacts
+
Structured Identifiers
+
Parameterized Values
+
Explicit Scope Boundaries
+
Explicit Correlation
+
Explicit Provenance
+
Security Barriers
+
Resource Budgets
```

---

# 274. Fórmula de persistent-runtime safety

```text
Persistent-Safe Query Composition
=
Immutable Query Artifacts
+
Frozen Shared Registries
+
Operation-Scoped Scope State
+
Operation-Scoped Correlation State
+
Operation-Scoped Dependency Builders
+
No Global CTE Registry
+
No Global Parameter Allocator
+
No Stored Closures
+
No Runtime Resource Retention
```

---

# 275. Principio arquitectónico final

La separación deberá permanecer:

```text
Subquery / CTE Builder
        │
        ▼
"What nested query structure
did the developer declare?"
        │
        ▼
Semantic Analysis
        │
        ▼
"What scopes, symbols, relations,
outputs and correlations does it mean?"
        │
        ▼
Constraint Analysis
        │
        ▼
"What can be proven about
cardinality, nullability and dependencies?"
        │
        ▼
Optimizer
        │
        ▼
"Can it be flattened, decorrelated,
inlined or transformed?"
        │
        ▼
Planner
        │
        ▼
"Should it be materialized, shared,
recomputed or executed another way?"
        │
        ▼
Compiler
        │
        ▼
"How does the target platform
represent this structure?"
```

---

# 276. Conclusión

`Subquery and CTE System` convierte las queries anidadas en ciudadanos de primer nivel dentro del Query Engine de VoltStack.

La arquitectura permitirá:

```text
Scalar Subqueries
Row Subqueries
Derived Tables
EXISTS
NOT EXISTS
IN
NOT IN
Correlated Subqueries
Lateral Subqueries
Non-Recursive CTEs
Recursive CTEs
CTE Dependencies
Materialization Intent
```

manteniendo completamente separadas las responsabilidades de:

```text
Construction
Semantic Resolution
Constraint Analysis
Optimization
Physical Planning
Compilation
Execution
```

Esto permite conservar una API sencilla:

```php
DB::table('users')
    ->whereExists(...)
    ->whereIn(...)
    ->with(...);
```

mientras internamente VoltStack opera mediante:

```text
Immutable Query Artifacts
+
Explicit Scope Graph
+
Explicit Symbol Resolution
+
Explicit Correlation Graph
+
CTE Dependency Graph
+
Semantic Query Graph
+
Constraint Knowledge
+
Capability-Driven Optimization
+
Physical Planning
+
Dialect Compilation
```

La regla final será:

> **Una subquery no es SQL dentro de SQL y una CTE no es un `WITH` textual: ambas son queries estructuradas con identidad, scope, output, dependencies, correlation y semántica propias, susceptibles de análisis y transformación antes de que exista una sola cadena SQL.**

---

# 277. Estado del bloque Query Builder

Con este documento:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
```

queda definida la composición básica de queries anidadas y relaciones temporales nombradas.

---

# 278. Siguiente documento

```text
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
Set Operation AST
Set Operand
Set Operation Tree
Set Output Relation
Column Arity Compatibility
Type Reconciliation
Domain Type Preservation
Nullability Reconciliation
Column Naming Rules
Duplicate Semantics
Ordering Scope
LIMIT/OFFSET Scope
Nested Set Operations
Parentheses / Grouping Semantics
Associativity Boundaries
Precedence
Set Operation Normalization
Set Operation Semantic Analysis
Set Operation Constraint Analysis
Set Operation Optimization
Capability Requirements
Emulation Boundaries
Portability
Recursive CTE Integration
Query Builder API
Persistent Runtime Safety
```

manteniendo:

```text
Set Operation Builder
        │
        ▼
Structured Set Query
        │
        ▼
Semantic Compatibility
        │
        ▼
Unified Output Relation
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