# 51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md

# VoltStack Quantum Database
## Aggregation and Grouping System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 51 — Aggregation and Grouping System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Semantic Query  
**Versión:** 1.0

---

# 1. Propósito

`Aggregation and Grouping System` define la arquitectura mediante la cual VoltStack representa consultas agregadas y operaciones de agrupación.

El sistema cubre:

```text
COUNT
SUM
AVG
MIN
MAX

DISTINCT aggregates

GROUP BY
HAVING

GROUPING SETS
ROLLUP
CUBE

aggregate FILTER
ordered aggregates
custom aggregates
```

sin convertir estas operaciones prematuramente en SQL.

La regla fundamental es:

```text
Aggregation
≠
SQL aggregate syntax
```

y:

```text
Grouping
≠
textual GROUP BY clause
```

VoltStack deberá representar la intención de agregación mediante estructuras semánticas capaces de participar en:

```text
Normalization
Validation
Symbol Resolution
Type Inference
Grouping Analysis
Constraint Analysis
Semantic Graph
Optimization
Planning
Compilation
Telemetry
Security
```

---

# 2. Regla maestra

```text
Aggregation Builder
≠
Aggregation Compiler
≠
Aggregation Executor
```

El Builder declara:

> qué valores desea agrupar y calcular el desarrollador.

Semantic Analysis determina:

> qué expresiones pertenecen a cada grupo y si la consulta es semánticamente válida.

Optimizer determina:

> qué transformaciones preservan el resultado.

Planner determina:

> cómo realizar físicamente la agregación.

Compiler determina:

> cómo expresar el plan para la plataforma objetivo.

---

# 3. Posición arquitectónica

```text
Application
    │
    ▼
Select Query Builder
    │
    ▼
Aggregation / Grouping Builder
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
    ├── Scope Resolution
    ├── Symbol Resolution
    ├── Type Inference
    ├── Aggregate Resolution
    ├── Grouping Analysis
    ├── Functional Dependency Analysis
    ├── HAVING Resolution
    └── Output Resolution
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

# 4. Objetivos

El sistema deberá proporcionar:

- funciones agregadas estructuradas;
- agregados built-in y extensibles;
- `DISTINCT` dentro de agregados;
- agrupación simple;
- múltiples grouping keys;
- expresiones de grouping;
- `HAVING`;
- `GROUPING SETS`;
- `ROLLUP`;
- `CUBE`;
- aggregate filters;
- ordered aggregates;
- análisis de legality;
- detección de nested aggregates inválidos;
- análisis de functional dependencies;
- reconciliación de tipos;
- nullability semántica;
- lineage;
- cardinality facts;
- capability requirements;
- portability;
- optimizaciones seguras;
- planificación física;
- extensibilidad;
- persistent-runtime safety.

---

# 5. No responsabilidades

Este sistema no deberá:

```text
generar SQL
abrir conexiones
ejecutar consultas
leer schema mediante I/O oculto
seleccionar índices
elegir HashAggregate desde Builder
hidratar entidades
administrar UnitOfWork
manejar transacciones
convertir automáticamente agregados a window functions
```

---

# 6. Relación con documentos anteriores

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
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
```

---

# 7. Conceptos fundamentales

VoltStack deberá distinguir explícitamente:

```text
Aggregate Function
Aggregate Expression
Grouping Expression
Grouping Key
Grouping Specification
Aggregate Scope
Group
Aggregate Output
HAVING Predicate
Window Function
Scalar Function
```

Estas entidades no deberán mezclarse.

---

# 8. Aggregate Function ≠ Aggregate Expression

Una función:

```text
COUNT
```

es una definición semántica.

Una expresión:

```text
COUNT(users.id)
```

es una instancia concreta dentro de una query.

Por tanto:

```text
AggregateFunctionDescriptor
≠
AggregateExpression
```

---

# 9. Aggregate ≠ Window Function

La distinción es crítica:

```text
SUM(amount)
```

como aggregate reduce filas.

Mientras:

```text
SUM(amount) OVER (...)
```

es una window expression y normalmente conserva la cardinalidad de entrada.

Por tanto:

```text
AggregateExpression
≠
WindowExpression
```

---

# 10. Scalar Function ≠ Aggregate Function

Ejemplo:

```text
LOWER(name)
```

opera conceptualmente sobre una fila.

Mientras:

```text
MAX(created_at)
```

opera sobre un conjunto/grupo de filas.

---

# 11. Aggregate function identity

Las funciones deberán poseer identidad semántica independiente del nombre SQL.

Ejemplo:

```text
AggregateFunctionId::COUNT
AggregateFunctionId::SUM
AggregateFunctionId::AVG
AggregateFunctionId::MIN
AggregateFunctionId::MAX
```

---

# 12. No vendor function names in core semantics

El core no deberá razonar mediante:

```php
if ($database === 'postgres') {
    ...
}
```

ni almacenar sintaxis específica como significado semántico.

---

# 13. AggregateFunctionDescriptor

Conceptualmente:

```php
final readonly class AggregateFunctionDescriptor
{
    public function __construct(
        public AggregateFunctionId $id,
        public AggregateArity $arity,
        public AggregateTypeRule $typeRule,
        public AggregateNullRule $nullRule,
        public AggregateDistinctSupport $distinctSupport,
        public AggregateOrderingSupport $orderingSupport,
        public AggregateFilterSupport $filterSupport,
        public Volatility $volatility,
        public Determinism $determinism,
        public CapabilityRequirementSet $capabilities,
    ) {}
}
```

---

# 14. Built-in aggregates V1

VoltStack deberá soportar como mínimo:

```text
COUNT
SUM
AVG
MIN
MAX
```

---

# 15. Future aggregate families

La arquitectura deberá permitir posteriormente:

```text
STRING_AGG
ARRAY_AGG
JSON_AGG
BOOL_AND
BOOL_OR
BIT_AND
BIT_OR
STDDEV
VARIANCE
PERCENTILE
MEDIAN
statistical aggregates
extension aggregates
```

sin modificar la arquitectura base.

---

# 16. AggregateExpression

Modelo conceptual:

```php
final readonly class AggregateExpression implements ExpressionNode
{
    public function __construct(
        public AggregateFunctionId $function,
        public AggregateArgumentSet $arguments,
        public AggregateQuantifier $quantifier,
        public ?AggregateFilter $filter,
        public AggregateOrdering $ordering,
        public AggregateMetadata $metadata,
    ) {}
}
```

---

# 17. Aggregate quantifier

Debe distinguirse:

```text
ALL
DISTINCT
```

Ejemplo:

```text
COUNT(email)

COUNT(DISTINCT email)
```

---

# 18. AggregateQuantifier

```php
enum AggregateQuantifier
{
    case ALL;
    case DISTINCT;
}
```

---

# 19. DISTINCT aggregate ≠ query DISTINCT

Esta distinción es obligatoria:

```text
SELECT DISTINCT category, COUNT(*)
```

no significa lo mismo que:

```text
SELECT category, COUNT(DISTINCT user_id)
```

Por tanto:

```text
QueryDistinctSpecification
≠
AggregateQuantifier
```

---

# 20. COUNT(*)

`COUNT(*)` deberá tener representación estructurada.

No:

```php
RawExpression('COUNT(*)');
```

sino algo equivalente a:

```text
AggregateExpression
├── function = COUNT
└── argument = CountAllRowsArgument
```

---

# 21. CountAllRowsArgument

```php
final readonly class CountAllRowsArgument implements AggregateArgument
{
}
```

---

# 22. COUNT(expression)

Deberá distinguirse de `COUNT(*)`.

```text
COUNT(email)
```

normalmente ignora valores NULL de la expresión.

---

# 23. COUNT result type

El Type System determinará el tipo semántico del resultado.

El Builder no deberá asumir directamente:

```text
PHP int
```

como definición completa del tipo de base de datos.

---

# 24. SUM

`SUM(expression)` deberá requerir un tipo agregable compatible.

Ejemplo:

```text
SUM(order.total)
```

---

# 25. SUM type inference

El tipo resultante podrá depender de:

```text
input type
numeric widening rules
domain rules
platform capabilities
```

---

# 26. AVG

`AVG` podrá requerir promoción de tipo.

Por ejemplo:

```text
INTEGER
→
DECIMAL-like semantic result
```

dependiendo de las reglas del Query Type System.

---

# 27. MIN/MAX

`MIN` y `MAX` requieren una semántica de orden válida para el tipo.

No deberá asumirse que cualquier custom type es ordenable.

---

# 28. Aggregate type rules

Cada descriptor podrá declarar:

```text
ACCEPTS_NUMERIC
ACCEPTS_ORDERABLE
ACCEPTS_ANY
CUSTOM_TYPE_RULE
```

---

# 29. Aggregate argument validation

Builder podrá verificar estructura básica.

Semantic Analysis verificará:

```text
resolved argument type
aggregate compatibility
domain compatibility
capabilities
```

---

# 30. Aggregate parameters

Valores runtime utilizados dentro de aggregate expressions continuarán siendo parámetros estructurados.

Ejemplo:

```text
SUM(price * :tax)
```

internamente:

```text
AggregateExpression
└── BinaryExpression
    ├── price
    └── ParameterExpression P1
```

---

# 31. No interpolation

Prohibido:

```php
"SUM(price * $tax)"
```

desde APIs estructuradas.

---

# 32. Aggregate filter

VoltStack podrá modelar:

```text
COUNT(*) FILTER (WHERE status = 'paid')
```

como:

```text
AggregateExpression
├── COUNT(*)
└── AggregateFilter
    └── Predicate
```

---

# 33. Filter ≠ HAVING

Muy importante:

```text
Aggregate Filter
≠
HAVING Predicate
```

Aggregate filter limita las filas consumidas por una instancia agregada.

`HAVING` filtra grupos/resultados agregados.

---

# 34. Example

```text
COUNT(*) FILTER (WHERE status = 'paid')
```

puede convivir con:

```text
HAVING COUNT(*) > 10
```

y ambos pertenecen a scopes semánticos diferentes.

---

# 35. Aggregate filter capabilities

El Builder declarará la feature semántica.

El sistema de capabilities determinará si existe:

```text
native support
safe emulation
unsupported
```

---

# 36. No CASE rewrite in Builder

Builder no convertirá automáticamente:

```text
FILTER
```

en:

```text
CASE WHEN ...
```

La emulación pertenece a fases posteriores.

---

# 37. Ordered aggregate

Algunas funciones agregadas dependen de ordering interno.

Conceptualmente:

```text
AGGREGATE(value ORDER BY created_at)
```

---

# 38. Aggregate ordering ≠ query ordering

```text
AggregateOrdering
≠
QueryOrderBy
```

El primero afecta una aggregate expression.

El segundo ordena el resultado de la query.

---

# 39. AggregateOrdering

```php
final readonly class AggregateOrdering
{
    /**
     * @param list<OrderingTerm> $terms
     */
    public function __construct(
        public array $terms,
    ) {}
}
```

---

# 40. Grouping

`GROUP BY` será modelado mediante una:

```text
GroupingSpecification
```

y no como strings SQL.

---

# 41. GroupingSpecification

Conceptualmente:

```php
interface GroupingSpecification
{
}
```

Implementaciones:

```text
SimpleGroupingSpecification
GroupingSetsSpecification
RollupSpecification
CubeSpecification
ExtensionGroupingSpecification
```

---

# 42. Simple grouping

Ejemplo:

```php
DB::table('orders')
    ->select('customer_id')
    ->selectRawAggregate(...)
    ->groupBy('customer_id');
```

deberá producir conceptualmente:

```text
SimpleGroupingSpecification
└── GroupingKey
    └── ColumnReference(customer_id)
```

---

# 43. GroupingKey

Un grouping key podrá ser:

```text
column expression
computed expression
tuple/grouping item
extension expression
```

si las reglas semánticas y capabilities lo permiten.

---

# 44. Grouping key ≠ column name

Por ejemplo:

```text
GROUP BY DATE(created_at)
```

deberá modelarse como expresión estructurada.

---

# 45. GroupingExpression

```php
final readonly class GroupingExpression
{
    public function __construct(
        public ExpressionNode $expression,
    ) {}
}
```

---

# 46. Multiple grouping keys

```text
GROUP BY country, city
```

produce conceptualmente:

```text
GroupingKeySet
├── country
└── city
```

---

# 47. Composite grouping key

El sistema deberá distinguir, cuando sea semánticamente relevante:

```text
GROUP BY a, b
```

de estructuras avanzadas como:

```text
GROUPING SETS ((a, b), (a), ())
```

---

# 48. Empty grouping set

Una grouping set vacía:

```text
()
```

representa un grupo global.

No deberá confundirse con:

```text
no aggregation
```

---

# 49. Query without GROUP BY but with aggregate

Ejemplo:

```text
SELECT COUNT(*)
FROM users
```

representa una:

```text
GlobalAggregateQuery
```

o una semántica equivalente.

---

# 50. Implicit global group

Conceptualmente:

```text
Aggregate query
+
no explicit grouping keys
=
single global grouping domain
```

sujeto a la semántica de la consulta.

---

# 51. No GROUP BY ≠ no grouping semantics

Una query con aggregate y sin `GROUP BY` sigue teniendo semántica agregada.

---

# 52. Aggregate scope

Semantic Analysis deberá construir un:

```text
AggregateScope
```

que determine:

- input rows;
- grouping keys;
- aggregate expressions;
- HAVING predicates;
- output expressions;
- aggregate dependencies.

---

# 53. AggregateScopeId

Cada aggregate scope tendrá identidad explícita:

```text
AggregateScopeId
```

---

# 54. Nested query scopes

Cada subquery podrá poseer su propio aggregate scope.

```text
Outer Query
└── Subquery
    └── Aggregate Scope
```

---

# 55. Aggregate scope isolation

Los aggregates de una subquery no deberán contaminar el scope del parent.

---

# 56. Aggregate query phases

Conceptualmente:

```text
Input Relation
    │
    ▼
WHERE
    │
    ▼
Grouping
    │
    ▼
Aggregate Evaluation
    │
    ▼
HAVING
    │
    ▼
Projection
    │
    ▼
Query-level DISTINCT
    │
    ▼
Ordering
    │
    ▼
Pagination
```

Esta representación es semántica; no prescribe el physical plan.

---

# 57. WHERE ≠ HAVING

`WHERE` filtra filas antes de grouping.

`HAVING` filtra grupos después de agregación semántica.

---

# 58. Builder API

Ejemplo:

```php
$query = DB::table('orders')
    ->select('customer_id')
    ->selectAggregate('sum', 'total', as: 'total_spent')
    ->where('status', 'paid')
    ->groupBy('customer_id')
    ->havingAggregate('sum', 'total', '>', 1000);
```

---

# 59. Structured representation

Conceptualmente:

```text
SelectQuery
├── FROM orders
├── WHERE
│   └── status = P1
├── GROUPING
│   └── customer_id
├── PROJECTION
│   ├── customer_id
│   └── SUM(total) AS total_spent
└── HAVING
    └── SUM(total) > P2
```

---

# 60. Aggregate expression identity

Si la misma aggregate expression aparece varias veces:

```text
SUM(total)
```

VoltStack podrá identificar equivalencia estructural/semántica.

Pero no deberá depender de string matching.

---

# 61. AggregateExpressionId

Podrá existir:

```text
AggregateExpressionId
```

para registrar instancias y facts.

---

# 62. Common aggregate expressions

Semantic Analysis podrá detectar:

```text
SUM(total)
```

en projection y HAVING como expresiones equivalentes cuando corresponda.

---

# 63. Deduplication ownership

La reutilización física de una aggregate computation pertenece al:

```text
Optimizer / Planner
```

no al Builder.

---

# 64. HAVING predicate system

`HAVING` reutilizará el Predicate System.

```text
HavingPredicate
→ PredicateNode
```

con un contexto semántico específico.

---

# 65. HAVING scope

HAVING podrá referenciar, según reglas del lenguaje semántico:

```text
grouping keys
aggregate expressions
aggregate output aliases
permitted expressions
```

---

# 66. Alias visibility

La visibilidad de aliases en HAVING no deberá depender accidentalmente de diferencias entre motores.

VoltStack deberá definir una regla semántica propia.

---

# 67. Recommended V1 alias rule

La API podrá permitir referencias explícitas a:

```text
ProjectionAliasReference
```

cuando sea legal, y el Compiler adaptará la representación al target.

---

# 68. No textual alias guessing

Una string `"total"` no deberá adivinarse simultáneamente como:

```text
source column
projection alias
aggregate alias
```

sin reglas de resolución explícitas.

---

# 69. Grouping legality

Semantic Analysis deberá verificar que las expresiones proyectadas sean válidas bajo el grouping.

Ejemplo:

```text
SELECT customer_id, status, COUNT(*)
FROM orders
GROUP BY customer_id
```

`status` requiere análisis.

---

# 70. Naïve rule rejected

No se deberá implementar únicamente:

```text
every non-aggregate projected column
must appear textually in GROUP BY
```

porque existen:

```text
functional dependencies
equivalent expressions
keys
platform semantics
```

---

# 71. Functional dependency analysis

El sistema deberá integrar:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

para determinar cuándo una expresión está funcionalmente determinada por los grouping keys.

---

# 72. Example functional dependency

Si:

```text
users.id
```

es primary key:

```text
users.id → users.name
users.id → users.email
```

puede existir evidencia de que agrupar por `users.id` determina otras columnas.

---

# 73. Platform differences

No todas las bases de datos aceptan las mismas formas de functional-dependency grouping.

Por tanto deberán distinguirse:

```text
Semantic validity
Target platform expressibility
```

---

# 74. Portable strict mode

VoltStack podrá ofrecer una política:

```text
STRICT_PORTABLE_GROUPING
```

que requiera reglas compatibles entre plataformas.

---

# 75. Platform-aware mode

También podrá existir:

```text
TARGET_PLATFORM_GROUPING
```

cuando la aplicación acepte semántica específica.

---

# 76. GroupingLegalityResult

Conceptualmente:

```php
final readonly class GroupingLegalityResult
{
    public function __construct(
        public bool $valid,
        public GroupingLegalityReasonSet $reasons,
        public CapabilityRequirementSet $capabilities,
    ) {}
}
```

---

# 77. Functional dependency ≠ textual grouping

Una expresión puede ser semánticamente determinada sin aparecer literalmente como grouping key.

---

# 78. Expression equivalence

Ejemplo:

```text
GROUP BY user_id
```

y una projection equivalente a ese símbolo deberá resolverse por identidad semántica, no por comparación textual.

---

# 79. Derived grouping expressions

Para:

```text
GROUP BY YEAR(created_at)
```

una projection de `YEAR(created_at)` puede ser grouping-compatible.

---

# 80. Similar text ≠ same expression

Dos funciones con mismo texto aparente pero diferentes:

```text
collation
type
scope
function identity
volatility
```

no deberán asumirse equivalentes.

---

# 81. Volatile expressions

Una volatile expression requiere tratamiento conservador.

---

# 82. Grouping volatile expression

Si el lenguaje permite agrupar por una volatile expression, VoltStack deberá preservar su semántica y limitar transformaciones.

---

# 83. Nested aggregates

Ejemplo:

```text
SUM(COUNT(*))
```

en el mismo aggregate scope normalmente será inválido.

---

# 84. Nested aggregate rule

```text
AggregateExpression
inside
AggregateExpression
within same AggregateScope
→ invalid
```

salvo una feature explícita que defina otra semántica.

---

# 85. Aggregate over subquery aggregate

Esto puede ser válido:

```text
Outer SUM(...)
    │
    ▼
Subquery
    └── COUNT(...)
```

porque existen aggregate scopes diferentes.

---

# 86. Aggregate/window boundary

También deberá distinguirse:

```text
aggregate inside window
window inside aggregate
```

y aplicar las reglas semánticas correspondientes.

El sistema completo de windows pertenece al documento 52.

---

# 87. GROUPING SETS

La arquitectura deberá representar:

```text
GROUPING SETS
```

como first-class semantic structure.

---

# 88. Example

```text
GROUP BY GROUPING SETS (
    (country, city),
    (country),
    ()
)
```

---

# 89. GroupingSetsSpecification

```php
final readonly class GroupingSetsSpecification
    implements GroupingSpecification
{
    /**
     * @param list<GroupingSet> $sets
     */
    public function __construct(
        public array $sets,
    ) {}
}
```

---

# 90. GroupingSet

```php
final readonly class GroupingSet
{
    /**
     * @param list<GroupingExpression> $expressions
     */
    public function __construct(
        public array $expressions,
    ) {}
}
```

---

# 91. ROLLUP

`ROLLUP` deberá ser una estructura semántica.

Ejemplo:

```text
ROLLUP(country, city)
```

no deberá convertirse en raw SQL desde Builder.

---

# 92. RollupSpecification

```php
final readonly class RollupSpecification
    implements GroupingSpecification
{
    public function __construct(
        public GroupingKeySet $keys,
    ) {}
}
```

---

# 93. CUBE

Igualmente:

```text
CUBE(country, city)
```

se representará mediante:

```text
CubeSpecification
```

---

# 94. ROLLUP/CUBE expansion

VoltStack no estará obligado a expandir inmediatamente:

```text
ROLLUP(a, b)
```

a grouping sets equivalentes.

---

# 95. Semantic preservation

Mantener la intención original puede beneficiar:

```text
diagnostics
capability analysis
optimizer
compiler
telemetry
explain
```

---

# 96. Normalized grouping representation

Semantic Analysis podrá producir una representación normalizada adicional cuando sea útil.

---

# 97. Grouping functions

Funciones semánticas como:

```text
GROUPING(...)
GROUPING_ID(...)
```

podrán modelarse mediante descriptors especializados.

---

# 98. Grouping metadata

Con grouping sets, una output row puede representar distintos niveles de agregación.

El sistema deberá conservar información suficiente para analizarlo.

---

# 99. Nullable output due grouping sets

Una columna originalmente `NOT NULL` puede aparecer conceptualmente NULL en determinados subtotal groups.

Por tanto:

```text
Schema Nullability
≠
Grouping Output Nullability
```

---

# 100. Grouping-induced nullability

Deberá registrarse explícitamente:

```text
GroupingNullabilityEffect
```

o información equivalente.

---

# 101. Aggregate nullability

Los aggregates poseen reglas distintas.

Ejemplo conceptual:

```text
COUNT(...)
→ non-null numeric count

SUM(...)
→ potentially NULL on empty input

AVG(...)
→ potentially NULL on empty input

MIN(...)
→ potentially NULL on empty input

MAX(...)
→ potentially NULL on empty input
```

según la semántica definida.

---

# 102. Empty input semantics

Una global aggregate query sobre cero filas no necesariamente produce cero filas.

Ejemplo conceptual:

```text
COUNT(*)
```

puede producir una fila con cero.

---

# 103. Grouped empty input

Una grouped query sobre input vacío puede producir:

```text
zero groups
```

según su grouping specification.

---

# 104. Cardinality distinction

```text
Global Aggregate
≠
Grouped Aggregate
```

respecto a cardinalidad.

---

# 105. Global aggregate cardinality

Constraint Analysis podrá derivar:

```text
ResultCardinality = 1
```

para determinadas global aggregate queries sin HAVING que pueda eliminar el grupo.

---

# 106. HAVING impact

Con:

```text
HAVING ...
```

la cardinalidad de una global aggregate puede ser:

```text
0..1
```

---

# 107. Grouped cardinality

Para:

```text
GROUP BY key
```

la cardinalidad está relacionada con:

```text
number of distinct grouping keys
```

pero no deberá inventarse un número exacto sin estadísticas/evidencia.

---

# 108. Logical cardinality ≠ estimate

Constraint Analysis produce bounds/facts.

Planner podrá producir estimaciones estadísticas.

---

# 109. Group uniqueness

Los grouping keys normalmente determinan una fila por grupo dentro de una grouping set simple.

Esto puede producir:

```text
UniquenessFact(grouping keys)
```

sobre el output agregado.

---

# 110. Grouping sets uniqueness

Con múltiples grouping sets, esa conclusión requiere mayor cuidado.

---

# 111. Aggregate lineage

Una aggregate output deberá conservar lineage hacia sus inputs.

Ejemplo:

```text
SUM(orders.total)
```

produce:

```text
AggregateOutput.total
└── DERIVES_FROM orders.total
```

---

# 112. COUNT(*) lineage

`COUNT(*)` depende de la existencia/cardinalidad de las filas de input, aunque no dependa de una columna específica.

Podrá utilizar:

```text
RelationCardinalityLineage
```

o una representación equivalente.

---

# 113. Multi-expression lineage

Para:

```text
SUM(price * quantity)
```

lineage deberá incluir:

```text
price
quantity
```

---

# 114. Filter lineage

Para:

```text
COUNT(*) FILTER (WHERE status = 'paid')
```

deberá distinguirse:

```text
value lineage
predicate dependency
```

---

# 115. Lineage ≠ dependency

Se mantiene:

```text
Lineage
≠
Dependency
```

---

# 116. Aggregate dependency

Una aggregate expression podrá depender de:

```text
columns
relations
functions
types
collations
predicates
parameters
capabilities
extensions
```

---

# 117. Aggregate semantic info

Conceptualmente:

```php
final readonly class AggregateSemanticInfo
{
    public function __construct(
        public AggregateExpressionId $id,
        public AggregateFunctionId $function,
        public QueryType $resultType,
        public Nullability $nullability,
        public AggregateScopeId $scope,
        public DependencySet $dependencies,
        public LineageSet $lineage,
        public CapabilityRequirementSet $capabilities,
        public Portability $portability,
    ) {}
}
```

---

# 118. AggregateSemanticTable

La información resuelta deberá vivir en una side table.

No deberá mutarse el AST con datos resueltos.

---

# 119. AST immutability

```text
AggregateExpressionNode
```

permanece estructural.

```text
AggregateSemanticTable
```

contiene el significado resuelto.

---

# 120. Grouping semantic info

Podrá existir:

```text
GroupingSemanticInfo
```

con:

```text
scope
resolved grouping expressions
grouping sets
functional dependencies
nullability effects
capabilities
portability
```

---

# 121. Grouping output relation

Semantic Analysis deberá producir una relación de salida explícita.

```text
AggregateOutputRelation
```

---

# 122. AggregateOutputRelation

Ejemplo:

```text
AggregateOutputRelation
├── customer_id
│   └── grouping key
├── total_spent
│   └── SUM(total)
└── order_count
    └── COUNT(*)
```

---

# 123. Output column identity

Cada columna tendrá:

```text
OutputColumnId
```

independiente de su source symbol.

---

# 124. Grouping key output

Una grouping key proyectada no deberá reutilizar directamente el `ColumnSymbolId` original como output identity.

---

# 125. Query Semantic Graph

El graph deberá representar explícitamente:

```text
AggregateScope
Grouping
GroupingKeys
AggregateExpressions
HAVING
AggregateOutputRelation
```

---

# 126. Example semantic graph

```text
AggregateScope AG1
│
├── INPUT → orders
│
├── GROUPS_BY → customer_id
│
├── COMPUTES → SUM(total)
│
├── COMPUTES → COUNT(*)
│
├── FILTERS_GROUPS_BY → HAVING
│
└── PRODUCES → AggregateOutputRelation R2
```

---

# 127. Graph edges

Podrán incluir:

```text
GROUPS_BY
AGGREGATES
AGGREGATES_OVER
FILTERS_AGGREGATE
ORDERS_AGGREGATE_BY
PRODUCES
DERIVES_FROM
DEPENDS_ON
REQUIRES_CAPABILITY
FUNCTIONALLY_DETERMINES
```

---

# 128. Constraint Analysis integration

Constraint Analysis podrá derivar facts desde grouping.

Ejemplos:

```text
grouping key uniqueness
global aggregate cardinality
aggregate non-nullability
HAVING constraints
functional dependencies
```

---

# 129. HAVING facts

Una condición:

```text
HAVING COUNT(*) > 0
```

podrá producir facts sobre grupos sobrevivientes.

---

# 130. Scope-specific facts

Estos facts pertenecen a:

```text
HAVING_FILTERED_GROUPS
```

y no necesariamente al input relation.

---

# 131. WHERE fact propagation

Facts provenientes de WHERE pueden influir en aggregate analysis.

Ejemplo:

```text
WHERE status = 'paid'
GROUP BY status
```

permite derivar que el grouping key sobreviviente tiene valor `paid`.

---

# 132. Constant grouping key

Constraint Analysis podrá detectar grouping keys constantes.

---

# 133. Redundant grouping key evidence

Puede producir evidencia de que una grouping key es redundante.

La eliminación pertenece al Optimizer.

---

# 134. Functional dependency simplification

Si:

```text
A → B
```

y ambos están en GROUP BY, puede existir oportunidad de simplificación.

Pero sólo Optimizer podrá transformarlo.

---

# 135. Builder never removes grouping keys

Aunque una key parezca redundante:

```text
Builder
→ preserve developer intent
```

---

# 136. Optimizer responsibilities

El Optimizer podrá considerar:

```text
grouping key reduction
predicate pushdown
HAVING pushdown
aggregate deduplication
aggregate simplification
partial aggregation
aggregation pushdown
pre-aggregation
constant aggregate simplification
empty-input simplification
distinct aggregate optimization
grouping sets normalization
```

siempre preservando semántica.

---

# 137. HAVING pushdown

Ejemplo:

```text
HAVING grouping_key = constant
```

podría convertirse parcialmente en un predicate previo al grouping.

Pero sólo cuando sea seguro.

---

# 138. Aggregate predicate pushdown

No todo HAVING puede empujarse.

Ejemplo:

```text
HAVING SUM(total) > 1000
```

depende del grupo completo.

---

# 139. Partial aggregation

En determinados planes:

```text
Input Partitions
    │
    ├── Partial Aggregate
    ├── Partial Aggregate
    └── Partial Aggregate
            │
            ▼
       Final Aggregate
```

---

# 140. Partial aggregation ≠ semantic aggregate

La query sigue conteniendo un único aggregate semántico.

Las etapas parciales son physical planning.

---

# 141. Aggregate decomposition

Cada aggregate descriptor podrá indicar si soporta:

```text
partial aggregation
merge
finalization
```

---

# 142. Algebraic properties

Descriptors avanzados podrán declarar:

```text
decomposable
associative
commutative
order-sensitive
distinct-sensitive
```

cuando estas propiedades estén formalmente definidas.

---

# 143. No assumed associativity

VoltStack no deberá asumir que toda custom aggregate es asociativa.

---

# 144. No assumed commutativity

Tampoco deberá asumir que el orden de input es irrelevante.

---

# 145. Planner

Planner podrá elegir estrategias como:

```text
HashAggregate
SortAggregate
StreamingAggregate
PartialAggregate
ParallelAggregate
DistinctAggregate
GroupingSetsAggregate
```

---

# 146. Logical aggregate ≠ HashAggregate

```text
SUM(total) GROUP BY customer_id
```

es semántica.

```text
HashAggregate
```

es una estrategia física.

---

# 147. Planner considerations

Podrá considerar:

```text
estimated rows
distinct group estimate
row width
memory
existing ordering
indexes
parallelism
aggregate decomposability
spill capability
grouping set count
distinct aggregate count
```

---

# 148. SortAggregate

Puede ser favorable si input ya posee ordering compatible.

---

# 149. HashAggregate

Puede ser favorable cuando:

```text
hashable grouping keys
memory budget
cardinality estimate
```

lo permiten.

---

# 150. Streaming aggregate

Puede ser posible sobre input ordenado por grouping keys.

---

# 151. Physical strategy capabilities

No todas las estrategias deberán existir en todos los drivers/platforms.

---

# 152. Database execution vs VoltStack planning

Si VoltStack compila a SQL tradicional, parte de la estrategia física final seguirá perteneciendo al database optimizer.

Aun así, VoltStack puede mantener un logical/physical planning layer para:

```text
query rewrites
capability decisions
distributed execution
future execution engines
diagnostics
explain
```

---

# 153. Compiler

Compiler deberá traducir la representación ya resuelta.

---

# 154. Compiler responsibilities

Puede decidir:

```text
GROUP BY syntax
GROUPING SETS syntax
ROLLUP syntax
CUBE syntax
FILTER syntax
aggregate DISTINCT syntax
ordered aggregate syntax
grouping function syntax
required casts
```

---

# 155. Compiler non-responsibilities

No deberá:

```text
decidir si una projection es grouping-valid
inferir functional dependencies
adivinar aggregate types
resolver aliases semánticamente
inventar aggregate capabilities
```

---

# 156. Capability model

Capacidades propuestas:

```text
QUERY.AGGREGATE.COUNT
QUERY.AGGREGATE.SUM
QUERY.AGGREGATE.AVG
QUERY.AGGREGATE.MIN
QUERY.AGGREGATE.MAX

QUERY.AGGREGATE.DISTINCT
QUERY.AGGREGATE.FILTER
QUERY.AGGREGATE.ORDERING
QUERY.AGGREGATE.MULTI_ARGUMENT

QUERY.GROUPING.BASIC
QUERY.GROUPING.EXPRESSION
QUERY.GROUPING.SETS
QUERY.GROUPING.ROLLUP
QUERY.GROUPING.CUBE
QUERY.GROUPING.EMPTY_SET
QUERY.GROUPING.FUNCTION
QUERY.GROUPING.FUNCTIONAL_DEPENDENCY
```

---

# 157. Capability status

Cada capability podrá clasificarse:

```text
NATIVE
EMULATABLE
RESTRICTED
UNSUPPORTED
```

---

# 158. Emulation

Una feature podrá emularse únicamente si:

```text
semantic equivalence
```

está garantizada.

---

# 159. Aggregate FILTER emulation

Una plataforma podría permitir transformar conceptualmente:

```text
aggregate FILTER
```

a otra estructura.

Pero Builder no realizará la transformación.

---

# 160. GROUPING SETS emulation

Puede implicar múltiples grouped queries combinadas mediante set operations.

Por ejemplo conceptualmente:

```text
Grouping Sets
        │
        ▼
Multiple Aggregate Queries
        │
        ▼
UNION ALL
```

pero esto sólo será permitido si se preservan:

```text
duplicate semantics
grouping nullability
grouping metadata
types
ordering
side effects
volatility
```

---

# 161. Integration with Set Operation System

La posible emulación reutilizará:

```text
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
```

y no implementará un segundo sistema de `UNION ALL`.

---

# 162. ROLLUP/CUBE expansion

Igualmente podrán convertirse a grouping sets durante Optimization/Planning si existe beneficio.

---

# 163. No forced expansion in Builder

Builder conservará:

```text
ROLLUP
CUBE
GROUPING SETS
```

como intención estructurada.

---

# 164. Portability

El sistema deberá producir información de portability.

---

# 165. Portable aggregate

Funciones básicas pueden clasificarse como:

```text
PORTABLE
```

cuando sus tipos y semántica sean compatibles.

---

# 166. Platform-specific aggregate

Una custom/vendor aggregate puede clasificarse:

```text
PLATFORM_SPECIFIC
```

---

# 167. Portable with requirements

Ejemplo:

```text
GROUPING SETS
```

puede clasificarse:

```text
PORTABLE_WITH_REQUIREMENTS
```

---

# 168. Raw aggregate

Una expresión:

```php
selectRaw('vendor_agg(foo)')
```

deberá marcarse:

```text
RAW
```

salvo que exista un semantic descriptor registrado.

---

# 169. Raw aggregate barrier

Puede limitar:

```text
type inference
lineage
constraint analysis
optimizer rewrites
portability
partial aggregation
```

---

# 170. Prefer semantic extension

En lugar de:

```php
selectRaw('median(price)')
```

una extensión debería registrar:

```text
AggregateFunctionDescriptor::MEDIAN
```

---

# 171. Aggregate extension registry

Propuesta:

```text
AggregateFunctionRegistry
```

---

# 172. Registry lifecycle

```text
Bootstrap
    │
    ▼
Register built-ins
    │
    ▼
Register extensions
    │
    ▼
Validate
    │
    ▼
Freeze
```

---

# 173. No runtime mutation

Una vez iniciado el persistent worker:

```text
AggregateFunctionRegistry
```

deberá permanecer frozen.

---

# 174. Extension descriptor requirements

Una custom aggregate deberá declarar al menos:

```text
semantic identity
argument arity
argument type rules
result type rule
nullability rule
distinct support
filter support
ordering support
volatility
determinism
capabilities
portability
fingerprint version
```

---

# 175. Optional advanced extension properties

También:

```text
partial aggregation support
merge semantics
state type
finalization semantics
associativity
commutativity
order sensitivity
empty input behavior
```

---

# 176. Builder API proposal

Ejemplos:

```php
$query
    ->select('customer_id')
    ->selectCount('*', as: 'orders')
    ->selectSum('total', as: 'revenue')
    ->groupBy('customer_id');
```

---

# 177. Expression API

También:

```php
$sum = DB::expr()
    ->aggregate('sum', DB::expr()->column('total'));

$query->selectAs($sum, 'revenue');
```

---

# 178. Typed aggregate API

Preferiblemente:

```php
DB::aggregate()->sum(
    DB::expr()->column('total')
);
```

para evitar stringly-typed APIs internas.

---

# 179. DX shortcuts

Métodos como:

```text
count()
sum()
avg()
min()
max()
```

podrán existir como shortcuts.

Pero todos deberán producir las mismas semantic aggregate structures.

---

# 180. No duplicate aggregate engines

Prohibido tener:

```text
CountBuilder
SumBuilder
AvgBuilder
```

como engines semánticos independientes.

Podrán existir helpers especializados, pero delegarán al mismo sistema.

---

# 181. groupBy API

```php
$query->groupBy('customer_id');
```

deberá transformar el identifier a estructura tipada.

---

# 182. Multiple groupBy

```php
$query->groupBy(
    'country',
    'city',
);
```

---

# 183. addGroupBy

Podrá existir:

```php
$query->addGroupBy('category');
```

---

# 184. groupBy expression

```php
$query->groupByExpression(
    DB::expr()->datePart('year', 'created_at')
);
```

---

# 185. No raw by default

`groupBy('YEAR(created_at)')` no deberá interpretarse automáticamente como SQL.

---

# 186. Explicit raw grouping

Si existe:

```php
groupByRaw(...)
```

deberá ser:

```text
explicit
parameterizable when applicable
provenance-tracked
portability-aware
semantic barrier
```

---

# 187. having API

Ejemplo:

```php
$query->having('order_count', '>', 10);
```

requiere reglas explícitas para saber si `order_count` es alias/output reference.

---

# 188. Prefer explicit aggregate HAVING

También:

```php
$query->having(
    DB::aggregate()->countAll(),
    '>',
    10,
);
```

---

# 189. HAVING values

El valor:

```text
10
```

se convertirá en:

```text
ParameterExpression
```

por default.

---

# 190. HAVING raw

`havingRaw()` será escape hatch explícito.

---

# 191. Query metadata

Aggregate queries podrán declarar metadata como:

```text
query label
performance intent
expected cardinality class
consistency requirement
diagnostic flags
```

sin almacenar runtime resources.

---

# 192. Security

Valores en HAVING y aggregate expressions deberán parameterizarse.

---

# 193. Dynamic aggregate names

No deberá permitirse convertir directamente input de usuario:

```text
$_GET['aggregate']
```

en función SQL.

---

# 194. Allowlist

Las aggregate functions dinámicas deberán resolverse mediante:

```text
AggregateFunctionRegistry
+
application allowlist
```

---

# 195. Dynamic grouping identifiers

Igualmente:

```text
GROUP BY user_input
```

requiere identifier allowlisting.

Parameterization no protege identifiers.

---

# 196. Security policies

Filtros de autorización/multitenancy que afecten input rows deberán materializarse antes de aggregation semantics.

---

# 197. No hidden post-aggregate injection

Una tenant/security policy no deberá aparecer mágicamente durante SQL compilation.

---

# 198. Policy provenance

Predicates introducidos por policy deberán conservar:

```text
provenance
strength
security metadata
```

---

# 199. Aggregate data leakage

El sistema deberá considerar que agregaciones pueden revelar información incluso sin devolver filas individuales.

Ejemplos:

```text
COUNT
MIN
MAX
SUM
```

---

# 200. Authorization boundary

Las políticas que determinen qué filas puede observar una query deberán aplicarse al input relation antes de agregación, salvo que una policy explícita defina otra semántica.

---

# 201. Aggregation ≠ authorization

El core Aggregate System no dependerá directamente de:

```text
Authentication
Authorization
Tenant
```

La integración será externa.

---

# 202. Persistent runtime

El sistema deberá ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 203. Shared immutable state

Podrá compartirse:

```text
frozen aggregate descriptors
frozen aggregate registry
frozen capability descriptors
immutable semantic artifacts
stateless factories
```

---

# 204. Operation-local state

Deberá ser local:

```text
AggregationBuilderState
GroupingBuilderState
AggregateResolutionState
GroupingAnalysisState
AggregateScopeBuilder
DiagnosticCollector
work queues
temporary memoization
```

---

# 205. No global aggregate scope

Prohibido:

```php
static $currentAggregateScope;
```

---

# 206. No global grouping state

Prohibido:

```php
static array $currentGroupBy;
```

---

# 207. No cross-request aggregate cache with mutable objects

Cualquier cache compartido deberá contener únicamente artifacts seguros e immutable.

---

# 208. Concurrency

Dos queries concurrentes:

```text
Q1 → GROUP BY customer_id
Q2 → GROUP BY tenant_id
```

no deberán compartir estado mutable.

---

# 209. Builder state

Propuesta:

```php
final class AggregationBuilderState
{
    public array $aggregateExpressions = [];
    public ?GroupingSpecification $grouping = null;
    public ?PredicateNode $having = null;
}
```

La implementación real deberá utilizar tipos especializados en lugar de arrays genéricos cuando sea posible.

---

# 210. Finalization

```text
Mutable Aggregation Builder
        │
        ▼
AggregationFinalizer
        │
        ▼
Immutable Query Model
        │
        ▼
Query AST
```

---

# 211. Finalizer

Puede:

```text
freeze aggregate expressions
freeze grouping specification
freeze HAVING
validate basic builder state
```

---

# 212. Finalizer shall not

```text
resolve schema
infer aggregate result types
resolve functional dependencies
decide grouping legality from schema
optimize
plan
compile
execute
```

---

# 213. Normalization

Normalization podrá canonicalizar estructuras seguras.

Ejemplos:

```text
flatten grouping builder wrappers
canonicalize equivalent structural forms
normalize aggregate quantifier defaults
normalize grouping collection representation
```

---

# 214. Normalization shall not

```text
remove grouping keys based on statistics
rewrite aggregate algorithms
push HAVING based on cost
expand every grouping set
```

---

# 215. Validation

Structural Validation deberá detectar:

```text
malformed aggregate calls
invalid arity known structurally
empty required grouping expressions
malformed grouping sets
invalid aggregate filter structure
invalid aggregate ordering structure
```

---

# 216. Semantic Analysis

Deberá resolver:

```text
aggregate function identity
argument types
result types
nullability
aggregate scope
grouping keys
functional dependencies
HAVING references
projection legality
grouping-induced nullability
capability requirements
portability
output relation
lineage
dependencies
```

---

# 217. Semantic completeness

Una executable aggregate query deberá llegar a:

```text
COMPLETE
```

antes de Optimization.

---

# 218. No lazy grouping resolution

Optimizer, Planner y Compiler no deberán volver a descubrir:

```text
which columns are grouping keys
which expressions are aggregates
whether a projection is grouping-valid
```

---

# 219. Fingerprinting

Structural fingerprint deberá incluir:

```text
aggregate function identities
argument structure
quantifiers
filters
aggregate ordering
grouping specification
grouping keys
HAVING structure
extension versions
relevant metadata
```

---

# 220. Semantic fingerprint

Además:

```text
resolved argument types
aggregate result types
nullability
resolved grouping symbols
functional dependency decisions
grouping output
capabilities
portability
schema identity/version
semantic extension versions
```

---

# 221. Runtime values excluded

Bindings ordinarios no deberán formar parte del semantic fingerprint.

---

# 222. Cache safety

Una cached semantic aggregate artifact sólo podrá reutilizarse cuando:

```text
query structure
schema fingerprint
type rules
capabilities
policies
extension versions
```

sean compatibles.

---

# 223. Budgets

El sistema deberá limitar:

```text
aggregate expression count
grouping key count
grouping set count
grouping set width
aggregate nesting depth
HAVING complexity
aggregate filter complexity
aggregate ordering terms
lineage edges
semantic work
extension work
```

---

# 224. AggregationBudget

Conceptualmente:

```php
final readonly class AggregationBudget
{
    public function __construct(
        public int $maxAggregates,
        public int $maxGroupingKeys,
        public int $maxGroupingSets,
        public int $maxGroupingWidth,
        public int $maxAggregateDepth,
        public int $maxLineageEdges,
    ) {}
}
```

---

# 225. Combinatorial grouping sets

`CUBE` puede crecer exponencialmente.

Para `N` dimensions:

```text
GroupingSets(CUBE)
≈
2^N
```

Por tanto deberá existir protección explícita.

---

# 226. No eager exponential expansion

VoltStack no deberá expandir siempre `CUBE` a todas las grouping sets durante Builder.

---

# 227. Budget exhaustion

Deberá producir error explícito:

```text
AggregationBudgetExceededException
```

y no truncar silenciosamente.

---

# 228. Diagnostics

Los diagnostics deberán incluir cuando sea posible:

```text
AggregateExpressionId
AggregateScopeId
GroupingSpecification
GroupingKey
AST source location
resolved symbol
resolved type
capability
platform
provenance
```

---

# 229. Invalid projection diagnostic

Ejemplo:

```text
Projection "orders.status" is not valid
within aggregate scope AG1.

Grouping keys:
    orders.customer_id

The expression is neither:
    - an aggregate expression,
    - grouping-equivalent,
    - nor functionally determined
      under the active grouping policy.
```

---

# 230. Nested aggregate diagnostic

```text
Aggregate SUM contains aggregate COUNT
inside the same aggregate scope AG2.

Nested aggregates within the same
aggregate scope are not permitted.
```

---

# 231. Type diagnostic

```text
Aggregate SUM cannot consume
semantic type "EmailAddress".

Expected:
    Numeric-compatible type

Actual:
    Domain<EmailAddress>
```

---

# 232. Capability diagnostic

```text
GROUPING SETS are required by query Q17,
but the selected target platform does not
provide a native or semantics-preserving
execution strategy.
```

---

# 233. Testing

El sistema deberá incluir tests para:

```text
COUNT(*)
COUNT(expression)
COUNT(DISTINCT expression)
SUM
AVG
MIN
MAX
aggregate NULL behavior
global aggregation
basic GROUP BY
multiple grouping keys
expression grouping
HAVING
aggregate FILTER
aggregate ORDER BY
GROUPING SETS
ROLLUP
CUBE
empty grouping set
grouping functions
functional dependencies
invalid non-grouped projections
nested aggregates
subquery aggregate scopes
aggregate/window boundaries
lineage
dependencies
nullability
cardinality facts
capabilities
portability
raw aggregates
extensions
budgets
persistent workers
concurrency
```

---

# 234. Builder tests

Builder tests deberán verificar:

```text
Query Model
AST
AggregateExpression
GroupingSpecification
HAVING Predicate
ParameterDefinitionSet
BindingSet
Metadata
```

no SQL.

---

# 235. Semantic tests

Deberán comprobar:

```text
AggregateScope
resolved aggregate functions
types
nullability
grouping legality
functional dependencies
output relation
lineage
constraints
capabilities
SemanticQueryGraph
```

---

# 236. Optimizer tests

Deberán comprobar:

```text
aggregate deduplication
HAVING pushdown
grouping key reduction
partial aggregation
aggregation pushdown
grouping set normalization
empty input simplification
```

incluyendo casos donde las transformaciones deben rechazarse.

---

# 237. Compiler tests

Aquí sí deberán probarse representaciones como:

```text
COUNT(*)
COUNT(DISTINCT ...)
GROUP BY
HAVING
FILTER
GROUPING SETS
ROLLUP
CUBE
```

para cada plataforma.

---

# 238. Architecture tests

El Builder/Aggregation System no deberá importar:

```text
PDO
Driver
Connection
ConnectionManager
Statement
Executor
SQL Compiler
UnitOfWork
EntityManager
```

---

# 239. Persistent runtime tests

Después de múltiples requests/work units deberá verificarse:

```text
no leaked aggregate expressions
no leaked grouping keys
no leaked HAVING predicates
no leaked parameters
no leaked aggregate scopes
no leaked diagnostics
no leaked schema information
```

---

# 240. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Builder\Aggregate
```

para construcción.

Y:

```text
VoltStack\Quantum\Database\Query\Semantic\Aggregate
```

para significado resuelto.

---

# 241. Directory structure propuesta

```text
Query/
├── Builder/
│   └── Aggregate/
│       ├── Contract/
│       │   ├── AggregationBuilderInterface.php
│       │   └── AggregationFinalizerInterface.php
│       │
│       ├── Core/
│       │   ├── AggregateFunctionId.php
│       │   ├── AggregateExpressionId.php
│       │   ├── AggregateExpression.php
│       │   ├── AggregateQuantifier.php
│       │   ├── AggregateArgument.php
│       │   ├── AggregateArgumentSet.php
│       │   └── CountAllRowsArgument.php
│       │
│       ├── Grouping/
│       │   ├── GroupingSpecification.php
│       │   ├── SimpleGroupingSpecification.php
│       │   ├── GroupingExpression.php
│       │   ├── GroupingKeySet.php
│       │   ├── GroupingSet.php
│       │   ├── GroupingSetsSpecification.php
│       │   ├── RollupSpecification.php
│       │   └── CubeSpecification.php
│       │
│       ├── Filter/
│       │   └── AggregateFilter.php
│       │
│       ├── Ordering/
│       │   └── AggregateOrdering.php
│       │
│       ├── Builder/
│       │   ├── AggregationBuilder.php
│       │   ├── AggregationBuilderState.php
│       │   └── AggregationFinalizer.php
│       │
│       ├── Extension/
│       │   ├── AggregateFunctionDescriptor.php
│       │   ├── AggregateFunctionRegistry.php
│       │   └── AggregateExtension.php
│       │
│       ├── Budget/
│       │   └── AggregationBudget.php
│       │
│       └── Exception/
│           ├── AggregationException.php
│           ├── InvalidAggregateException.php
│           ├── InvalidGroupingException.php
│           └── AggregationBudgetExceededException.php
│
└── Semantic/
    └── Aggregate/
        ├── AggregateScopeId.php
        ├── AggregateScope.php
        ├── AggregateSemanticInfo.php
        ├── AggregateSemanticTable.php
        ├── GroupingSemanticInfo.php
        ├── GroupingLegalityAnalyzer.php
        ├── AggregateTypeResolver.php
        ├── AggregateNullabilityResolver.php
        ├── AggregateOutputRelation.php
        └── AggregateSemanticAnalyzer.php
```

---

# 242. Ownership matrix

| Concern | Owner |
|---|---|
| Fluent aggregate construction | Query Builder |
| Aggregate AST structure | Query Model / AST |
| Aggregate function registry | Aggregate System |
| Parameter values | BindingSet |
| Symbol resolution | Semantic Engine |
| Argument type inference | Query Type System |
| Aggregate result type | Aggregate Semantic Analysis |
| Grouping legality | Aggregate Semantic Analysis |
| Functional dependencies | Constraint Analysis |
| HAVING predicates | Predicate + Aggregate Semantic System |
| Lineage | Semantic Lineage System |
| Cardinality facts | Constraint Analysis |
| Algebraic rewrites | Optimizer |
| Partial aggregation | Planner |
| Hash/Sort aggregate choice | Planner / DB optimizer |
| SQL syntax | Compiler |
| Execution | Executor |
| ORM hydration | ORM/Hydration |
| Authorization filters | Integration layer |
| Runtime state reset | Runtime lifecycle |

---

# 243. Invariantes arquitectónicos

## DB-AGG-001
Aggregation no será SQL syntax.

## DB-AGG-002
Grouping no será textual `GROUP BY`.

## DB-AGG-003
Builder no generará SQL.

## DB-AGG-004
Builder no ejecutará queries.

## DB-AGG-005
Aggregate Function será distinta de Aggregate Expression.

## DB-AGG-006
Aggregate será distinto de Window Function.

## DB-AGG-007
Aggregate será distinto de Scalar Function.

## DB-AGG-008
Aggregate functions tendrán identidad semántica.

## DB-AGG-009
Core semantics no dependerá de vendor function names.

## DB-AGG-010
COUNT(*) tendrá representación estructurada.

## DB-AGG-011
COUNT(*) será distinto de COUNT(expression).

## DB-AGG-012
Aggregate DISTINCT será distinto de query DISTINCT.

## DB-AGG-013
Aggregate filters serán estructurados.

## DB-AGG-014
Aggregate Filter será distinto de HAVING.

## DB-AGG-015
Aggregate Ordering será distinto de Query Ordering.

## DB-AGG-016
GroupingSpecification será estructurada.

## DB-AGG-017
Grouping keys podrán ser expressions.

## DB-AGG-018
Grouping key no será necesariamente column name.

## DB-AGG-019
No GROUP BY con aggregate seguirá teniendo grouping semantics.

## DB-AGG-020
Global aggregate tendrá scope explícito.

## DB-AGG-021
Aggregate scopes estarán aislados entre subqueries.

## DB-AGG-022
WHERE será distinto de HAVING.

## DB-AGG-023
HAVING reutilizará Predicate System.

## DB-AGG-024
Aliases tendrán resolución explícita.

## DB-AGG-025
Grouping legality será semántica.

## DB-AGG-026
Grouping legality no dependerá sólo de textual equality.

## DB-AGG-027
Functional dependencies podrán afectar grouping legality.

## DB-AGG-028
Functional dependency será distinta de textual grouping.

## DB-AGG-029
Platform grouping semantics serán capability-aware.

## DB-AGG-030
Nested aggregates dentro del mismo scope serán rechazados salvo feature explícita.

## DB-AGG-031
Aggregates en scopes diferentes podrán componerse legalmente.

## DB-AGG-032
GROUPING SETS será first-class.

## DB-AGG-033
ROLLUP será first-class.

## DB-AGG-034
CUBE será first-class.

## DB-AGG-035
Builder no expandirá obligatoriamente ROLLUP.

## DB-AGG-036
Builder no expandirá obligatoriamente CUBE.

## DB-AGG-037
Empty grouping set será distinta de no aggregation.

## DB-AGG-038
Grouping-induced nullability será explícita.

## DB-AGG-039
Schema nullability será distinta de aggregate output nullability.

## DB-AGG-040
COUNT nullability seguirá descriptor semántico.

## DB-AGG-041
SUM nullability seguirá descriptor semántico.

## DB-AGG-042
AVG nullability seguirá descriptor semántico.

## DB-AGG-043
MIN/MAX nullability seguirá descriptor semántico.

## DB-AGG-044
Empty input semantics serán explícitas.

## DB-AGG-045
Global aggregate cardinality será distinta de grouped aggregate cardinality.

## DB-AGG-046
HAVING podrá modificar cardinality bounds.

## DB-AGG-047
Logical cardinality será distinta de estimated cardinality.

## DB-AGG-048
Grouping keys podrán producir uniqueness facts.

## DB-AGG-049
Grouping sets requerirán análisis especial de uniqueness.

## DB-AGG-050
Aggregate lineage será explícita.

## DB-AGG-051
COUNT(*) podrá depender de relation cardinality.

## DB-AGG-052
Lineage será distinta de dependency.

## DB-AGG-053
Aggregate semantic info vivirá en side tables.

## DB-AGG-054
AST permanecerá immutable.

## DB-AGG-055
SemanticQueryGraph representará aggregate scopes.

## DB-AGG-056
SemanticQueryGraph representará grouping.

## DB-AGG-057
Constraint Analysis será aggregate-aware.

## DB-AGG-058
HAVING facts serán scope-specific.

## DB-AGG-059
Builder no eliminará grouping keys.

## DB-AGG-060
Optimizer será dueño de grouping simplification.

## DB-AGG-061
Optimizer será dueño de HAVING pushdown.

## DB-AGG-062
Optimizer será dueño de aggregate deduplication.

## DB-AGG-063
Partial aggregation será physical planning.

## DB-AGG-064
Logical aggregate será distinto de physical aggregate.

## DB-AGG-065
Custom aggregates no serán asumidas associative.

## DB-AGG-066
Custom aggregates no serán asumidas commutative.

## DB-AGG-067
Order-sensitive aggregates serán explícitas.

## DB-AGG-068
Distinct-sensitive aggregates serán explícitas.

## DB-AGG-069
Compiler no decidirá grouping legality.

## DB-AGG-070
Compiler no inferirá aggregate types.

## DB-AGG-071
Capabilities reemplazarán vendor conditionals.

## DB-AGG-072
Unsafe aggregate emulation estará prohibida.

## DB-AGG-073
Grouping Sets emulation reutilizará Set Operation System.

## DB-AGG-074
No existirá segundo UNION engine para grouping emulation.

## DB-AGG-075
Portability será explícita.

## DB-AGG-076
Raw aggregates serán explícitas.

## DB-AGG-077
Raw aggregate será semantic barrier cuando corresponda.

## DB-AGG-078
Semantic aggregate extensions serán preferidas sobre raw SQL.

## DB-AGG-079
Aggregate registry será frozen.

## DB-AGG-080
Registry no tendrá runtime mutation.

## DB-AGG-081
Extensions declararán argument type rules.

## DB-AGG-082
Extensions declararán result type rules.

## DB-AGG-083
Extensions declararán nullability rules.

## DB-AGG-084
Extensions declararán capabilities.

## DB-AGG-085
Extensions declararán portability.

## DB-AGG-086
Builder shortcuts reutilizarán un mismo aggregate core.

## DB-AGG-087
No habrá engines independientes por aggregate built-in.

## DB-AGG-088
Grouping identifiers serán estructurados.

## DB-AGG-089
Raw grouping requerirá API explícita.

## DB-AGG-090
HAVING values serán parameterized por default.

## DB-AGG-091
Dynamic aggregate names requerirán allowlist/registry.

## DB-AGG-092
Dynamic grouping identifiers requerirán allowlist.

## DB-AGG-093
Parameterization no protegerá identifiers.

## DB-AGG-094
Security predicates deberán existir antes de aggregate semantics.

## DB-AGG-095
Compiler no inyectará hidden security filters.

## DB-AGG-096
Core Aggregate System no dependerá de Authentication.

## DB-AGG-097
Core Aggregate System no dependerá de Authorization.

## DB-AGG-098
Core Aggregate System no dependerá de Multitenancy.

## DB-AGG-099
Builder state será operation-scoped.

## DB-AGG-100
Aggregate resolution state será operation-scoped.

## DB-AGG-101
No existirá global aggregate scope.

## DB-AGG-102
No existirá global grouping state.

## DB-AGG-103
Persistent workers compartirán sólo immutable/frozen state.

## DB-AGG-104
Concurrent aggregate queries estarán aisladas.

## DB-AGG-105
Finalizer no realizará schema resolution.

## DB-AGG-106
Finalizer no realizará type inference.

## DB-AGG-107
Finalizer no realizará optimization.

## DB-AGG-108
Normalization no realizará cost-based transformations.

## DB-AGG-109
Validation distinguirá structural de semantic errors.

## DB-AGG-110
Semantic Analysis resolverá aggregate meaning completamente.

## DB-AGG-111
Optimizer no realizará lazy symbol resolution.

## DB-AGG-112
Planner no realizará lazy grouping legality analysis.

## DB-AGG-113
Compiler no reinterpretará aggregate semantics.

## DB-AGG-114
Fingerprints incluirán aggregate identities.

## DB-AGG-115
Fingerprints incluirán grouping structure.

## DB-AGG-116
Runtime binding values no formarán parte del semantic fingerprint.

## DB-AGG-117
CUBE estará protegido mediante budgets.

## DB-AGG-118
Grouping Sets estarán protegidas mediante budgets.

## DB-AGG-119
Budget exhaustion será explícito.

## DB-AGG-120
No habrá silent grouping truncation.

## DB-AGG-121
Aggregation construction será determinista.

## DB-AGG-122
Aggregate type resolution será determinista.

## DB-AGG-123
Grouping legality analysis será determinista.

## DB-AGG-124
ORM reutilizará el mismo Aggregate Query System.

## DB-AGG-125
Repository queries reutilizarán el mismo Aggregate Query System.

## DB-AGG-126
Active Record no implementará un aggregate engine separado.

## DB-AGG-127
Aggregate aliases no modificarán source symbol identity.

## DB-AGG-128
Aggregate output columns tendrán identidad propia.

## DB-AGG-129
Grouping output columns tendrán identidad propia.

## DB-AGG-130
Aggregation deberá permanecer independiente de SQL dialect syntax.

---

# 244. Anti-patterns

## 244.1 Raw COUNT by default

```php
selectRaw('COUNT(*)');
```

como implementación interna de `count()`.

**Rechazado.**

---

## 244.2 String GROUP BY

```php
$group = 'customer_id, status';
```

almacenado como cláusula SQL.

**Rechazado.**

---

## 244.3 HAVING as raw SQL

```php
$having = 'COUNT(*) > 10';
```

como representación principal.

**Rechazado.**

---

## 244.4 Query DISTINCT reused for aggregate DISTINCT

**Rechazado.**

---

## 244.5 Window reused as aggregate

**Rechazado.**

---

## 244.6 Textual grouping legality

```php
in_array($column, $groupByStrings);
```

como único mecanismo.

**Rechazado.**

---

## 244.7 Hidden schema lookup

Builder consultando schema para decidir grouping validity.

**Rechazado.**

---

## 244.8 Automatic arbitrary cast

Builder agregando casts para que `SUM()` acepte un tipo.

**Rechazado.**

---

## 244.9 Automatic ROLLUP expansion in Builder

**Rechazado.**

---

## 244.10 Automatic HAVING pushdown in Builder

**Rechazado.**

---

## 244.11 Global aggregate scope

**Rechazado.**

---

## 244.12 Vendor branching in semantic layer

```php
if ($platform === 'mysql') {
    // grouping semantics
}
```

**Rechazado.**

Se utilizarán capability/policy descriptors.

---

# 245. Ejemplo integral

Supongamos:

```php
$query = DB::table('orders')
    ->select('customer_id')
    ->selectSum('total', as: 'revenue')
    ->selectCount('*', as: 'orders_count')
    ->where('status', 'paid')
    ->groupBy('customer_id')
    ->having(
        DB::aggregate()->sum(
            DB::expr()->column('total')
        ),
        '>',
        1000
    )
    ->orderBy('revenue', 'desc')
    ->limit(100);
```

---

# 246. Builder representation

```text
SelectQuery
│
├── FROM
│   └── orders
│
├── WHERE
│   └── status = P1
│
├── GROUPING
│   └── customer_id
│
├── PROJECTION
│   ├── customer_id
│   ├── SUM(total) AS revenue
│   └── COUNT(*) AS orders_count
│
├── HAVING
│   └── SUM(total) > P2
│
├── ORDER
│   └── revenue DESC
│
└── LIMIT
    └── 100
```

Bindings:

```text
P1 = "paid"
P2 = 1000
```

---

# 247. Semantic resolution

Semantic Analysis resuelve:

```text
orders.customer_id
    │
    └── QueryType: CustomerId

orders.total
    │
    └── QueryType: Money
```

y posteriormente:

```text
SUM(orders.total)
    │
    ├── AggregateFunction = SUM
    ├── InputType = Money
    ├── ResultType = MoneyAggregate
    ├── Nullability = Nullable
    └── Scope = AG1
```

La denominación exacta del tipo dependerá del Type System.

---

# 248. Aggregate scope

```text
AggregateScope AG1
│
├── Input
│   └── orders
│
├── Pre-group Predicate
│   └── status = P1
│
├── Grouping Key
│   └── customer_id
│
├── Aggregate
│   ├── SUM(total)
│   └── COUNT(*)
│
├── HAVING
│   └── SUM(total) > P2
│
└── Output
    ├── customer_id
    ├── revenue
    └── orders_count
```

---

# 249. Output relation

```text
AggregateOutputRelation R2
│
├── C1 customer_id
│   ├── type = CustomerId
│   ├── groupingKey = true
│   └── lineage = orders.customer_id
│
├── C2 revenue
│   ├── type = MoneyAggregate
│   ├── aggregate = SUM
│   └── lineage = orders.total
│
└── C3 orders_count
    ├── aggregate = COUNT
    └── lineage = orders relation cardinality
```

---

# 250. Constraint facts

Constraint Analysis podrá derivar:

```text
orders.status = "paid"
```

para las filas de input sobrevivientes.

También:

```text
AggregateOutput.customer_id
is unique per simple grouping result
```

bajo las condiciones correspondientes.

Y desde:

```text
HAVING SUM(total) > 1000
```

un fact scoped al resultado:

```text
revenue > 1000
```

---

# 251. Optimizer

Podrá detectar que:

```text
SUM(total)
```

aparece tanto en:

```text
projection
HAVING
```

y producir una única aggregate computation lógica/física cuando sea seguro.

---

# 252. Planner

Un posible plan conceptual:

```text
Limit 100
    │
Sort revenue DESC
    │
Filter revenue > P2
    │
HashAggregate
├── Group Key: customer_id
├── SUM(total)
└── COUNT(*)
    │
Filter status = P1
    │
Scan orders
```

Este plan es ilustrativo.

El Planner real podrá elegir otra estrategia.

---

# 253. Separation

```text
Developer API
    │
    ▼
Aggregation Builder
    │
    ▼
Structured Aggregate Query
    │
    ▼
Semantic Aggregate Analysis
    │
    ▼
Aggregate Output Relation
    │
    ▼
Constraint Knowledge
    │
    ▼
Optimizer
    │
    ▼
Logical / Physical Planning
    │
    ▼
Compiler
    │
    ▼
SQL
```

---

# 254. Fórmula del aggregate

```text
Aggregate Expression
=
Aggregate Function
+
Arguments
+
Quantifier
+
Optional Filter
+
Optional Aggregate Ordering
+
Semantic Metadata
```

---

# 255. Fórmula del grouping

```text
Grouping
=
Grouping Specification
+
Grouping Expressions
+
Aggregate Scope
+
Grouping Semantics
```

---

# 256. Fórmula del aggregate scope

```text
Aggregate Scope
=
Input Relation
+
Pre-Grouping Predicates
+
Grouping Specification
+
Aggregate Expressions
+
HAVING
+
Output Expressions
```

---

# 257. Fórmula de semantic aggregation

```text
Semantic Aggregation
=
Resolved Aggregate Functions
+
Resolved Argument Types
+
Grouping Keys
+
Functional Dependencies
+
Aggregate Result Types
+
Nullability
+
HAVING Semantics
+
Lineage
+
Constraints
+
Capabilities
+
Portability
```

---

# 258. Fórmula de seguridad

```text
Safe Aggregation
=
Structured Aggregate Functions
+
Structured Grouping Expressions
+
Parameterized Values
+
Explicit Scope Boundaries
+
Pre-Aggregation Security Policies
+
Explicit Raw Barriers
+
Capability Validation
+
Resource Budgets
```

---

# 259. Fórmula de optimización

```text
Safe Aggregate Optimization
=
Semantic Aggregate Graph
+
Grouping Knowledge
+
Functional Dependencies
+
Constraint Knowledge
+
Cardinality Knowledge
+
Volatility
+
Aggregate Algebraic Properties
+
Capability Knowledge
+
Cost Information
```

---

# 260. Fórmula de persistent-runtime safety

```text
Persistent-Safe Aggregation
=
Immutable Query Artifacts
+
Operation-Scoped Aggregate State
+
Operation-Scoped Grouping State
+
Frozen Aggregate Registry
+
No Global Aggregate Scope
+
No Runtime Resource Retention
```

---

# 261. Boundary final

```text
Aggregation Builder
        │
        ▼
"What does the developer
want to aggregate and group?"
        │
        ▼
Semantic Analysis
        │
        ▼
"What constitutes each group,
which expressions are legal,
and what types do aggregates return?"
        │
        ▼
Constraint Analysis
        │
        ▼
"What facts are true about
groups and aggregate outputs?"
        │
        ▼
Optimizer
        │
        ▼
"Can grouping, predicates or
aggregate computations be simplified?"
        │
        ▼
Planner
        │
        ▼
"Should this use hash, sort,
streaming, partial or another strategy?"
        │
        ▼
Compiler
        │
        ▼
"How does the target platform
express the aggregate query?"
```

---

# 262. Decisión arquitectónica final

VoltStack adopta:

```text
Aggregate semantics
        │
        ├── AggregateFunctionDescriptor
        ├── AggregateExpression
        ├── AggregateScope
        ├── GroupingSpecification
        ├── HAVING Predicate
        ├── AggregateSemanticInfo
        └── AggregateOutputRelation
```

como representación independiente del SQL.

La regla final es:

> **VoltStack modelará las agregaciones como transformaciones semánticas de relaciones. `GROUP BY`, `HAVING`, `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `DISTINCT`, `FILTER`, `GROUPING SETS`, `ROLLUP` y `CUBE` serán conceptos estructurados del Query Engine y nunca simples fragmentos SQL.**

---

# 263. Estado del Bloque 4 — Query Builder

Con este documento quedan definidos:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
```

El bloque continúa con la semántica analítica de ventanas.

---

# 264. Siguiente documento

```text
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
```

Este documento deberá definir:

```text
Window Functions
Window Expressions
OVER
PARTITION BY
Window ORDER BY
Window Frames
ROWS
RANGE
GROUPS
Frame Boundaries
UNBOUNDED PRECEDING
CURRENT ROW
UNBOUNDED FOLLOWING
Named Windows
Window Inheritance
Ranking Functions
ROW_NUMBER
RANK
DENSE_RANK
NTILE
Navigation Functions
LAG
LEAD
FIRST_VALUE
LAST_VALUE
NTH_VALUE
Aggregate Window Functions
Window Scope
Window Output
Window Type Resolution
Window Nullability
Window Ordering Semantics
Peer Groups
Frame Semantics
Window Chaining
Window Validation
Window Capabilities
Window Portability
Window Optimization
Window Planning
Window Extensions
Persistent Runtime Safety
```

manteniendo la separación fundamental:

```text
Aggregate Function
≠
Window Function

Grouping
≠
Partitioning

Aggregate Scope
≠
Window Scope

GROUP BY
≠
PARTITION BY
```