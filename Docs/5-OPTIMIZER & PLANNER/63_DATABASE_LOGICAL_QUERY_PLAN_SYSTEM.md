# 63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md

# VoltStack Quantum Database
## Logical Query Plan System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 63 — Logical Query Plan System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Planner / Logical Planning  
**Versión:** 1.0

---

# 1. Propósito

`Logical Query Plan System` define la representación lógica intermedia utilizada por VoltStack entre:

```text
OptimizedQueryArtifact
```

y:

```text
PhysicalQueryPlan
```

Su responsabilidad principal es convertir una consulta semánticamente resuelta y lógicamente optimizada en un grafo de **operadores relacionales lógicos explícitos**, independiente de los detalles concretos de ejecución.

La definición central será:

```text
LogicalQueryPlan
=
Logical representation of what relational operations
must occur without deciding how they are physically executed.
```

Por tanto:

```text
Logical Plan
≠
Query AST
≠
Semantic Graph
≠
Physical Plan
≠
Execution Plan
≠
SQL
```

---

# 2. Posición arquitectónica

```text
Query Builder
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
     ▼
SemanticQueryArtifact
     │
     ▼
Query Optimizer
     │
     ▼
OptimizedQueryArtifact
     │
     ▼
┌───────────────────────────────┐
│     Logical Plan System       │
│                               │
│ Semantic → Logical Lowering   │
│ Logical Operators             │
│ Logical Properties            │
│ Dependencies                  │
│ Output Contracts              │
│ Logical Plan Graph            │
└───────────────────────────────┘
     │
     ▼
LogicalQueryPlan
     │
     ▼
Physical Query Planner
     │
     ▼
PhysicalQueryPlan
```

---

# 3. Objetivo fundamental

El sistema deberá responder:

> ¿Qué operaciones lógicas deben realizarse para obtener el resultado?

No deberá responder:

> ¿Qué algoritmo físico utilizará el motor?

Ejemplo:

```text
LogicalJoin
```

expresa:

```text
combine these two logical relations
according to these join semantics
```

pero no:

```text
HashJoin
MergeJoin
NestedLoopJoin
```

---

# 4. Frontera fundamental

```text
LogicalQueryPlan
=
logical relational execution structure

PhysicalQueryPlan
=
physical implementation strategy
```

---

# 5. Ejemplo simple

Consulta:

```sql
SELECT id, name
FROM users
WHERE active = true;
```

Logical Plan:

```text
LogicalProject
    id
    name
      │
      ▼
LogicalFilter
    active = true
      │
      ▼
LogicalScan
    users
```

No aparecerá todavía:

```text
IndexScan
SequentialScan
BitmapScan
```

---

# 6. Logical Plan ≠ Query AST

El Query AST conserva la estructura sintáctico-semántica de la consulta.

Ejemplo:

```text
SelectQuery
├── SelectList
├── From
├── Where
├── GroupBy
├── Having
└── OrderBy
```

El Logical Plan representa operadores relacionales:

```text
LogicalProject
      │
LogicalFilter
      │
LogicalScan
```

---

# 7. AST y Plan tienen propósitos diferentes

```text
AST
=
query language representation

Logical Plan
=
relational operation representation
```

---

# 8. Semantic Graph ≠ Logical Plan

El Semantic Graph contiene:

```text
symbols
scopes
relations
types
constraints
dependencies
correlations
lineage
capabilities
```

El Logical Plan consume esta información.

No la sustituye.

---

# 9. Semantic truth

Siempre:

```text
SemanticQueryArtifact
=
authoritative semantic meaning
```

El Logical Plan será una proyección/lowering de esa semántica hacia operadores ejecutables.

---

# 10. OptimizedQueryArtifact

El input inmediato será:

```php
OptimizedQueryArtifact
```

El sistema no deberá regresar al AST original para reinterpretar la consulta.

---

# 11. Lowering

La operación central se denominará:

```text
Logical Lowering
```

Formalmente:

```text
Lower(
    OptimizedQueryArtifact
)
→
LogicalQueryPlan
```

---

# 12. Lowering ≠ Optimization

Lowering transforma representación.

No decide por sí mismo una forma semánticamente diferente.

---

# 13. Lowering ≠ Compilation

No genera:

```text
SQL
placeholders
quoted identifiers
dialect syntax
```

---

# 14. Lowering ≠ Physical Planning

No selecciona:

```text
indexes
hash joins
merge joins
scan algorithms
parallel workers
memory algorithms
```

---

# 15. Logical Plan artifact

Modelo conceptual:

```php
final readonly class LogicalQueryPlan
{
    public function __construct(
        public LogicalPlanId $id,
        public LogicalPlanNode $root,
        public LogicalPlanGraph $graph,
        public LogicalOutputContract $output,
        public LogicalPlanPropertyTable $properties,
        public SemanticLogicalMapping $semanticMapping,
        public LogicalPlanDependencySet $dependencies,
        public LogicalPlanMetadata $metadata,
        public LogicalPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 16. Inmutabilidad

Una vez construido:

```text
LogicalQueryPlan
```

será inmutable.

---

# 17. Construction state

Durante lowering podrá existir:

```text
MutableLogicalPlanBuilderState
```

pero nunca formará parte del artifact final.

---

# 18. LogicalPlanId

Cada plan tendrá:

```php
LogicalPlanId
```

generado dentro del scope de la operación.

No será global.

---

# 19. LogicalPlanNodeId

Cada nodo tendrá:

```php
LogicalPlanNodeId
```

---

# 20. IDs distintos

Siempre:

```text
SemanticNodeId
≠
LogicalPlanNodeId
≠
PhysicalPlanNodeId
```

---

# 21. Razón

Una entidad semántica puede producir múltiples nodos lógicos.

Ejemplo:

```text
Semantic Relation R17
```

puede participar en:

```text
LogicalScan L4
LogicalFilter L5
LogicalProject L6
```

---

# 22. Logical operator model

Todos los operadores implementarán:

```php
interface LogicalPlanNode
{
    public function id(): LogicalPlanNodeId;

    public function kind(): LogicalPlanNodeKind;

    public function inputs(): LogicalPlanInputList;
}
```

---

# 23. LogicalPlanNodeKind

V1 podrá incluir:

```text
SCAN
VALUES
FILTER
PROJECT
JOIN
AGGREGATE
WINDOW
SORT
LIMIT
OFFSET
DISTINCT
SET_OPERATION
MATERIALIZE
CTE_REFERENCE
RECURSIVE_CTE
SEMI_JOIN
ANTI_JOIN
INSERT
UPDATE
DELETE
MUTATION
RETURNING
EMPTY_RELATION
EXTENSION
```

---

# 24. Operators ≠ SQL clauses

No se diseñarán simplemente como una copia de:

```text
SELECT
FROM
WHERE
GROUP BY
```

El Logical Plan utilizará operadores relacionales.

---

# 25. LogicalScan

Representa una fuente lógica.

```php
final readonly class LogicalScan implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public RelationInstanceId $relation,
        public LogicalOutputSchema $output,
    ) {}
}
```

---

# 26. LogicalScan ≠ SequentialScan

Fundamental:

```text
LogicalScan
≠
SequentialScan
```

---

# 27. LogicalScan ≠ Table name

El scan apunta a:

```text
RelationInstanceId
```

resuelto semánticamente.

No vuelve a resolver strings.

---

# 28. Scan sources

Puede representar:

```text
table
view
derived relation
CTE
table function
VALUES relation
extension relation
```

cuando corresponda.

---

# 29. LogicalValues

Representa una relación construida desde valores estructurados.

Ejemplo:

```sql
VALUES
    (?, ?),
    (?, ?)
```

Logical:

```text
LogicalValues
```

---

# 30. LogicalFilter

Representa filtrado lógico.

```php
final readonly class LogicalFilter implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public PredicateId $predicate,
    ) {}
}
```

---

# 31. Predicate identity

`LogicalFilter` referencia la representación semántica/optimizada del predicado.

No necesita reconstruirlo desde SQL.

---

# 32. Filter preserves 3VL

Toda evaluación lógica seguirá la semántica SQL de:

```text
TRUE
FALSE
UNKNOWN
```

---

# 33. Filter ≠ Index condition

Un filtro lógico:

```text
age > 18
```

no significa todavía:

```text
IndexRangeScan(age)
```

---

# 34. LogicalProject

Representa transformación del output.

```php
final readonly class LogicalProject implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public array $projections,
        public LogicalOutputSchema $output,
    ) {}
}
```

---

# 35. Projection expression

Cada proyección conservará:

```text
ExpressionId
OutputSymbolId
Alias
Type
Nullability
Lineage
```

mediante referencias semánticas apropiadas.

---

# 36. LogicalProject ≠ SELECT string

No:

```php
new LogicalProject('u.id, u.name');
```

---

# 37. LogicalJoin

Representa join lógico.

```php
final readonly class LogicalJoin implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalJoinType $type,
        public LogicalPlanNodeId $left,
        public LogicalPlanNodeId $right,
        public ?PredicateId $condition,
        public LogicalJoinProperties $properties,
    ) {}
}
```

---

# 38. LogicalJoinType

Podrá contener:

```text
INNER
LEFT
RIGHT
FULL
CROSS
SEMI
ANTI
```

---

# 39. SEMI y ANTI

Aunque el lenguaje original no los exponga directamente, el Optimizer puede haber producido formas lógicas:

```text
LogicalSemiJoin
LogicalAntiJoin
```

por ejemplo mediante decorrelación de:

```text
EXISTS
NOT EXISTS
```

---

# 40. Logical join ≠ physical join

Nunca:

```text
LogicalJoinType::INNER
→
HashJoin automatically
```

---

# 41. Join input order

El Logical Plan conserva el orden lógico final producido por el Optimizer.

---

# 42. Planner physical may use different execution properties

El Physical Planner puede seleccionar una estrategia cuya implementación interna procese inputs de forma distinta, siempre que preserve la semántica.

---

# 43. Outer joins

El Logical Plan deberá preservar explícitamente:

```text
LEFT
RIGHT
FULL
```

y sus propiedades de null-extension.

---

# 44. Null extension

Las propiedades de output reflejarán la nullabilidad derivada.

Ejemplo:

```text
A LEFT JOIN B
```

puede hacer nullable columnas provenientes de `B`.

---

# 45. LogicalAggregate

Representa agregación lógica.

```php
final readonly class LogicalAggregate implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public GroupingSpecification $grouping,
        public array $aggregates,
        public LogicalOutputSchema $output,
    ) {}
}
```

---

# 46. Aggregate ≠ HashAggregate

Nunca:

```text
LogicalAggregate
=
HashAggregate
```

---

# 47. Grouping semantics

Conservará:

```text
simple grouping
empty grouping
grouping sets
rollup
cube
```

según capabilities semánticas del query.

---

# 48. Global aggregate

Consulta:

```sql
SELECT COUNT(*)
FROM users;
```

produce conceptualmente:

```text
LogicalAggregate
  grouping = EMPTY
      │
LogicalScan(users)
```

---

# 49. Empty grouping ≠ no aggregate

Fundamental:

```text
Global Aggregate
≠
Non-Aggregated Query
```

---

# 50. LogicalWindow

Representa evaluación lógica de funciones window.

```php
final readonly class LogicalWindow implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public array $windowExpressions,
        public LogicalOutputSchema $output,
    ) {}
}
```

---

# 51. Window semantics

Conservará:

```text
PARTITION BY
window ordering
frame
frame boundaries
exclusion
named window resolution
```

en forma semántica estructurada.

---

# 52. LogicalWindow ≠ Sort

Aunque una window function pueda requerir ordering físico:

```text
LogicalWindow
```

no insertará automáticamente:

```text
PhysicalSort
```

---

# 53. LogicalSort

Representa ordering observable requerido por la query.

```php
final readonly class LogicalSort implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public OrderingSpecification $ordering,
    ) {}
}
```

---

# 54. LogicalSort ≠ PhysicalSort

Un LogicalSort expresa:

```text
observable ordering requirement
```

El Physical Planner puede satisfacerlo mediante:

```text
existing index ordering
merge-preserved ordering
physical sort
other strategy
```

---

# 55. Ordering requirement

Por ello:

```text
LogicalSort
```

puede desaparecer como operador físico explícito si el input ya satisface el ordering requerido.

---

# 56. LogicalLimit

Representa limitación semántica de cardinalidad.

```php
final readonly class LogicalLimit implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public LimitExpression $limit,
    ) {}
}
```

---

# 57. LogicalOffset

Representa:

```text
OFFSET
```

cuando se mantenga separado.

---

# 58. Limit ≠ Top-N algorithm

```text
LogicalLimit
≠
PhysicalTopN
```

---

# 59. LIMIT without ORDER BY

No se inventará ordering.

```text
LIMIT 10
```

sin:

```text
ORDER BY
```

no significa:

```text
first 10 by primary key
```

---

# 60. LogicalDistinct

`DISTINCT` podrá representarse como operador lógico explícito.

```php
final readonly class LogicalDistinct implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public LogicalPlanNodeId $input,
        public DistinctSpecification $specification,
    ) {}
}
```

---

# 61. DISTINCT ≠ HashDistinct

El Physical Planner decidirá implementación.

---

# 62. DISTINCT row semantics

La deduplicación seguirá equivalencia de filas SQL.

No:

```text
PHP ==
```

ni:

```text
predicate equality
```

---

# 63. LogicalSetOperation

Representa:

```text
UNION
INTERSECT
EXCEPT
```

con:

```text
DISTINCT
ALL
```

explícitos.

---

# 64. Modelo

```php
final readonly class LogicalSetOperation implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public SetOperationKind $operation,
        public SetQuantifier $quantifier,
        public array $inputs,
        public LogicalOutputSchema $output,
    ) {}
}
```

---

# 65. Bag semantics

`ALL` preservará multiplicidades.

---

# 66. Logical set tree

El agrupamiento será explícito.

```text
(A UNION B) EXCEPT C
```

será distinto estructuralmente de:

```text
A UNION (B EXCEPT C)
```

---

# 67. LogicalMaterialize

Puede representar materialización **semánticamente requerida**.

---

# 68. Logical materialization vs physical materialization

Distinción:

```text
LogicalMaterialize
=
materialization is part of required logical contract

PhysicalMaterialize
=
planner-selected implementation strategy
```

---

# 69. CTEs

Un CTE no necesariamente se convertirá en `LogicalMaterialize`.

---

# 70. LogicalCTEReference

Podrá representar una referencia a una definición CTE cuando la estructura lógica necesite conservarla.

---

# 71. CTE definition graph

Las definiciones podrán mantenerse en:

```text
LogicalCteDefinitionSet
```

separadas del árbol principal.

---

# 72. CTE identity

Siempre:

```text
CteId
≠
CTE name string
```

---

# 73. Recursive CTE

Se representará explícitamente.

Conceptualmente:

```text
LogicalRecursiveCte
├── Anchor Plan
├── Recursive Member Plan
├── Set Operation
└── Recursive Binding
```

---

# 74. Recursive semantics

No se reducirá a un loop genérico.

---

# 75. Logical recursion ≠ physical recursion strategy

El Physical Planner decidirá:

```text
working table
recursive spool
native recursive CTE
extension strategy
```

según capabilities.

---

# 76. EmptyRelation

El Optimizer puede demostrar que una rama no produce filas.

Logical Plan:

```text
LogicalEmptyRelation
```

---

# 77. EmptyRelation semantics

Debe contener un output schema válido.

Ejemplo:

```text
LogicalEmptyRelation
output:
    id:int
    name:string
```

---

# 78. Empty relation ≠ missing plan

Es un operador válido.

---

# 79. LogicalInsert

Representa inserción lógica.

```php
final readonly class LogicalInsert implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public RelationInstanceId $target,
        public InsertColumnSet $columns,
        public LogicalPlanNodeId $source,
        public ?ConflictAction $conflictAction,
        public ?ReturningSpecification $returning,
    ) {}
}
```

---

# 80. Insert source

Puede provenir de:

```text
LogicalValues
LogicalProject
LogicalScan
LogicalSetOperation
other relation-producing logical plan
```

---

# 81. LogicalInsert ≠ SQL INSERT

No contiene sintaxis vendor-specific.

---

# 82. LogicalUpdate

Representa mutation lógica.

```php
final readonly class LogicalUpdate implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public RelationInstanceId $target,
        public LogicalPlanNodeId $source,
        public AssignmentSet $assignments,
        public ?ReturningSpecification $returning,
    ) {}
}
```

---

# 83. LogicalDelete

```php
final readonly class LogicalDelete implements LogicalPlanNode
{
    public function __construct(
        public LogicalPlanNodeId $id,
        public RelationInstanceId $target,
        public LogicalPlanNodeId $source,
        public ?ReturningSpecification $returning,
    ) {}
}
```

---

# 84. Mutation source

El source logical plan determina el conjunto de filas candidatas.

---

# 85. Mutation semantics

El Logical Plan deberá conservar:

```text
target identity
row qualification
assignments
RETURNING
ordering when meaningful
limit semantics
locking requirements
security predicates
```

---

# 86. Direct mutation ≠ ORM persistence

Estos operadores representan DML del Query Engine.

No:

```text
EntityManager flush
UnitOfWork
entity lifecycle callbacks
```

---

# 87. LogicalReturning

`RETURNING` podrá representarse:

1. dentro del mutation operator; o
2. mediante un operador lógico dedicado.

VoltStack favorecerá una representación explícita compartida:

```text
LogicalReturning
```

cuando simplifique el output pipeline.

---

# 88. LogicalExtensionNode

Extensiones semánticas podrán aportar operadores lógicos.

```php
interface LogicalExtensionNode extends LogicalPlanNode
{
    public function extensionId(): ExtensionId;

    public function extensionNodeType(): ExtensionLogicalNodeType;
}
```

---

# 89. Unknown extension

No se transformará en:

```text
RawLogicalNode
```

silenciosamente.

---

# 90. Extension contract

Debe declarar:

```text
logical semantics
inputs
output schema
logical properties
dependencies
capabilities
barriers
fingerprinting
physical planning integration
```

---

# 91. Logical Plan tree vs graph

Aunque muchas consultas se representen como árbol:

```text
LogicalQueryPlan
```

será conceptualmente un grafo.

---

# 92. Por qué grafo

Por estructuras como:

```text
CTE reuse
recursive CTE
shared logical subplans
dependency edges
correlation edges
```

---

# 93. LogicalPlanGraph

```php
final readonly class LogicalPlanGraph
{
    public function __construct(
        public LogicalPlanNodeMap $nodes,
        public LogicalPlanEdgeSet $edges,
        public LogicalPlanNodeId $root,
    ) {}
}
```

---

# 94. Edge kinds

```text
DATA
DEPENDENCY
CORRELATION
CTE_REFERENCE
RECURSION
MUTATION_DEPENDENCY
EXTENSION
```

---

# 95. Data edge

Representa flujo relacional.

```text
Scan
  │ DATA
  ▼
Filter
```

---

# 96. Dependency edge

No necesariamente transporta filas.

Ejemplo:

```text
Subplan A
depends on
CTE Definition B
```

---

# 97. Correlation edge

Representa dependencia de símbolos externos.

---

# 98. Dependency ≠ data flow

Fundamental:

```text
Dependency Edge
≠
Data Edge
```

---

# 99. Correlation ≠ lineage

También:

```text
Correlation
≠
Lineage
```

---

# 100. Lineage

Lineage describe:

```text
where output values originate
```

No:

```text
which plan node must execute first
```

---

# 101. Logical output schema

Cada operador relation-producing tendrá:

```text
LogicalOutputSchema
```

---

# 102. LogicalOutputSchema

Ejemplo conceptual:

```php
final readonly class LogicalOutputSchema
{
    /**
     * @param list<LogicalOutputColumn> $columns
     */
    public function __construct(
        public array $columns,
    ) {}
}
```

---

# 103. LogicalOutputColumn

Podrá contener:

```text
OutputSymbolId
name
QueryType
nullability
lineage
visibility
ordinal
```

---

# 104. Output identity

La identidad será más importante que el nombre textual.

---

# 105. Duplicate names

Esto deberá ser válido internamente:

```text
id
id
```

si provienen de símbolos diferentes.

---

# 106. Alias

Un alias no reemplaza identidad.

```text
OutputSymbolId
≠
Alias
```

---

# 107. OutputContract

El root deberá satisfacer el contrato observable de la query.

---

# 108. LogicalOutputContract

```php
final readonly class LogicalOutputContract
{
    public function __construct(
        public LogicalOutputSchema $schema,
        public ObservableOrderingContract $ordering,
        public CardinalityContract $cardinality,
        public ResultShapeContract $shape,
    ) {}
}
```

---

# 109. Result shape

Puede distinguir:

```text
ROWS
SCALAR
ROW
MUTATION_COUNT
RETURNING_ROWS
EXISTS_RESULT
```

según el pipeline correspondiente.

---

# 110. Logical properties

El Logical Plan derivará propiedades útiles para planificación física.

---

# 111. Logical property examples

```text
output schema
cardinality bounds
uniqueness
functional dependencies
nullability
ordering requirement
distinctness
correlation
volatility
side effects
rewind requirements
required capabilities
```

---

# 112. Property table

No todas las propiedades necesitan incrustarse en cada nodo.

VoltStack utilizará side tables.

```php
final readonly class LogicalPlanPropertyTable
{
    public function properties(
        LogicalPlanNodeId $node
    ): LogicalProperties;
}
```

---

# 113. Why side tables

Evita convertir los nodos en objetos gigantes con metadata mutable.

---

# 114. LogicalProperties

```php
final readonly class LogicalProperties
{
    public function __construct(
        public LogicalOutputSchema $output,
        public CardinalityProperties $cardinality,
        public UniquenessProperties $uniqueness,
        public FunctionalDependencySet $functionalDependencies,
        public NullabilityProperties $nullability,
        public LogicalOrderingProperties $ordering,
        public CorrelationProperties $correlation,
        public VolatilityProperties $volatility,
        public LogicalEffectProperties $effects,
    ) {}
}
```

---

# 115. Logical properties ≠ physical properties

Ejemplo:

```text
query requires ordering by created_at
```

es logical requirement.

Mientras:

```text
IndexScan provides ordering by created_at
```

es physical property.

---

# 116. Cardinality properties

Se distinguirán:

```text
provable bounds
estimated cardinality
```

---

# 117. Proven cardinality

Ejemplo:

```text
scalar aggregate without GROUP BY
```

puede tener:

```text
exactly 1 row
```

como propiedad lógica demostrable.

---

# 118. Estimated cardinality

Ejemplo:

```text
~12,500 rows
```

es una estimación de costo/planning.

No deberá almacenarse como verdad semántica.

---

# 119. CardinalityBound

Podrá representar:

```text
ZERO
ZERO_OR_ONE
EXACTLY_ONE
AT_MOST_N
AT_LEAST_ONE
UNBOUNDED
UNKNOWN
```

o un modelo equivalente.

---

# 120. Uniqueness

Puede derivarse desde:

```text
primary keys
unique constraints
DISTINCT
grouping keys
proven join properties
```

---

# 121. Uniqueness is semantic evidence

No deberá inferirse únicamente de estadísticas.

---

# 122. Functional dependencies

Se conservarán cuando sean válidas a través de operadores.

---

# 123. Filter properties

Generalmente un filter:

```text
does not introduce new columns
does not increase cardinality
```

---

# 124. Projection properties

Projection puede:

```text
remove columns
rename outputs
compute expressions
destroy some uniqueness facts
preserve others
```

---

# 125. Join properties

Join puede afectar:

```text
cardinality
nullability
uniqueness
functional dependencies
correlation
```

---

# 126. Aggregate properties

Aggregate puede introducir:

```text
group-key uniqueness
new aggregate outputs
bounded cardinality
```

---

# 127. DISTINCT properties

Después de:

```text
LogicalDistinct
```

el output completo posee una propiedad de uniqueness por fila.

---

# 128. Limit properties

`LIMIT N` puede derivar:

```text
maxRows <= N
```

---

# 129. Offset

`OFFSET` no establece por sí mismo una cota superior.

---

# 130. EmptyRelation

Tiene:

```text
cardinality = exactly 0
```

---

# 131. Logical property derivation

Servicio:

```php
interface LogicalPropertyDeriver
{
    public function derive(
        LogicalPlanNode $node,
        LogicalPropertyDerivationContext $context,
    ): LogicalProperties;
}
```

---

# 132. Bottom-up derivation

Muchas propiedades se derivarán:

```text
children
   │
   ▼
parent
```

---

# 133. Example

```text
LogicalScan
  unique: user.id

      ↓

LogicalFilter
  unique: user.id still preserved

      ↓

LogicalProject(user.id)
  unique: output user.id
```

---

# 134. Property invalidation

Una transformación deberá eliminar propiedades que ya no sean demostrables.

---

# 135. Unknown ≠ false

Si uniqueness deja de poder demostrarse:

```text
UNKNOWN
```

no:

```text
NOT UNIQUE
```

---

# 136. Semantic-to-logical mapping

El lowering conservará mappings explícitos.

---

# 137. SemanticLogicalMapping

```php
final readonly class SemanticLogicalMapping
{
    public function __construct(
        public SemanticToLogicalNodeMap $nodes,
        public LogicalToSemanticOriginMap $origins,
        public OutputSymbolMapping $outputs,
    ) {}
}
```

---

# 138. One-to-many mapping

Un nodo semántico puede producir varios logical nodes.

---

# 139. Many-to-one mapping

Algunas construcciones optimizadas pueden converger en un único logical operator.

---

# 140. Mapping is not lineage

El mapping describe correspondencia entre artifacts.

Lineage describe procedencia de valores.

---

# 141. Provenance

Cada logical node podrá conservar:

```text
source semantic node
optimizer-generated
policy-generated
extension-generated
planner-generated
```

según corresponda.

---

# 142. Provenance usefulness

Para:

```text
debugging
EXPLAIN
telemetry
security auditing
optimizer diagnostics
```

---

# 143. Logical metadata

`LogicalPlanMetadata` podrá contener:

```text
query intent
read/write classification
portability requirements
security markers
policy provenance
diagnostic tags
extension metadata
```

---

# 144. Metadata ≠ runtime state

No contendrá:

```text
Connection
PDO
current tenant object
current user object
cursor
statement
```

---

# 145. Logical dependencies

El artifact mantendrá:

```text
LogicalPlanDependencySet
```

---

# 146. Dependency kinds

```text
SCHEMA_OBJECT
RELATION
COLUMN
FUNCTION
TYPE
COLLATION
CTE
CAPABILITY
EXTENSION
POLICY
```

---

# 147. Dependency usage

Servirá para:

```text
cache invalidation
plan validation
diagnostics
replanning
```

---

# 148. Capability requirements

Logical Plan podrá declarar capabilities lógicas.

Ejemplo:

```text
RECURSIVE_CTE
WINDOW_FRAME_GROUPS
FULL_OUTER_JOIN
RETURNING
```

---

# 149. Logical capability ≠ physical capability

Ejemplo:

```text
logical query requires FULL OUTER JOIN semantics
```

pero una plataforma podría satisfacerlo mediante:

```text
native support
exact emulation
framework strategy
```

---

# 150. Capability resolution

El Physical Planner decide cómo satisfacer el requerimiento.

---

# 151. Logical plan normalization

El Logical Plan podrá tener una forma canónica propia.

---

# 152. Logical canonicalization ≠ Query Normalization

Documento 33 normaliza Query AST.

Aquí sólo se estandariza la representación del plan.

---

# 153. Example

Dos formas semánticas ya equivalentes pueden lowering a:

```text
LogicalFilter
    │
LogicalScan
```

con una estructura estándar.

---

# 154. No optimizer resurrection

Logical canonicalization no deberá volver a implementar:

```text
join reordering
predicate pushdown
subquery decorrelation
join elimination
```

---

# 155. Lowering phases

Propuesta:

```text
L0 Input Verification
L1 Root Intent Lowering
L2 Relation Lowering
L3 Predicate Lowering
L4 Projection Lowering
L5 Aggregate Lowering
L6 Window Lowering
L7 Ordering/Limit Lowering
L8 CTE/Subquery Lowering
L9 Mutation Lowering
L10 Graph Assembly
L11 Property Derivation
L12 Output Contract Validation
L13 Fingerprinting
L14 Freeze
```

---

# 156. L0 — Input verification

Verifica:

```text
OptimizedQueryArtifact valid
semantic mappings available
required tables available
no stale semantic state
```

---

# 157. L1 — Root intent

Clasifica:

```text
SELECT
INSERT
UPDATE
DELETE
```

sin depender de SQL strings.

---

# 158. L2 — Relations

Construye fuentes lógicas.

---

# 159. L3 — Predicates

Construye filtros y join conditions desde referencias semánticas.

---

# 160. L4 — Projection

Construye el output lógico.

---

# 161. L5 — Aggregation

Introduce operadores aggregate cuando corresponda.

---

# 162. L6 — Window

Introduce la fase window.

---

# 163. L7 — Ordering and limits

Representa:

```text
ORDER BY
OFFSET
LIMIT
```

en su fase lógica correcta.

---

# 164. L8 — CTE/subquery

Integra:

```text
subplans
CTEs
correlation
recursion
```

---

# 165. L9 — Mutations

Construye DML logical operators.

---

# 166. L10 — Graph assembly

Ensambla:

```text
nodes
data edges
dependency edges
correlation edges
```

---

# 167. L11 — Property derivation

Calcula propiedades lógicas demostrables.

---

# 168. L12 — Output validation

Verifica que el root preserve:

```text
expected output shape
types
nullability
aliases
observable ordering
mutation contract
```

---

# 169. L13 — Fingerprinting

Genera:

```text
LogicalPlanFingerprint
```

---

# 170. L14 — Freeze

El artifact se vuelve completamente inmutable.

---

# 171. SELECT logical ordering

La estructura conceptual típica será:

```text
FROM
 ↓
JOIN
 ↓
WHERE
 ↓
AGGREGATE
 ↓
HAVING
 ↓
WINDOW
 ↓
PROJECT
 ↓
DISTINCT
 ↓
ORDER
 ↓
OFFSET/LIMIT
```

pero el lowering seguirá la representación optimizada real y las reglas semánticas definidas.

---

# 172. HAVING

Puede representarse como:

```text
LogicalFilter
```

sobre:

```text
LogicalAggregate
```

---

# 173. WHERE vs HAVING

Aunque ambos puedan ser `LogicalFilter`, su posición en el graph preserva su significado.

---

# 174. Example

```text
LogicalFilter(HAVING)
       │
LogicalAggregate
       │
LogicalFilter(WHERE)
       │
LogicalScan
```

---

# 175. Operator type alone is insufficient

La semántica depende también de:

```text
graph position
scope
predicate identity
metadata
```

---

# 176. Window phase

Ejemplo:

```text
LogicalProject
      │
LogicalWindow
      │
LogicalAggregate
      │
LogicalFilter
      │
LogicalScan
```

---

# 177. QUALIFY future

Si se soporta:

```text
QUALIFY
```

podrá lowering como filter posterior a window:

```text
LogicalFilter(QUALIFY)
      │
LogicalWindow
```

sin romper la arquitectura.

---

# 178. Subquery logical representation

Un subquery relation-producing tendrá su propio subplan.

---

# 179. Scalar subquery

Podrá representarse mediante:

```text
LogicalScalarSubquery
```

o una expresión lógica con referencia a un subplan.

Debe conservar:

```text
one-column requirement
cardinality semantics
correlation
error semantics
```

---

# 180. EXISTS

Si sigue como subquery:

```text
LogicalExistsSubquery
```

Si fue decorrelacionado por Optimizer:

```text
LogicalSemiJoin
```

---

# 181. NOT EXISTS

Puede convertirse en:

```text
LogicalAntiJoin
```

cuando ya exista prueba de equivalencia.

---

# 182. NOT IN

No se lowering ingenuamente a `LogicalAntiJoin` salvo que el Optimizer haya demostrado la equivalencia considerando NULL/3VL.

---

# 183. Scalar cardinality

Un scalar subquery conserva el requisito:

```text
0 rows → NULL
1 row → scalar
>1 rows → cardinality error
```

según la semántica definida.

---

# 184. Logical cardinality assertion

Puede existir:

```text
LogicalCardinalityCheck
```

si una estrategia posterior necesita hacer explícita esta condición.

---

# 185. Cardinality check ≠ estimate

Es una restricción semántica.

---

# 186. Lateral/correlated relation

El Logical Plan conservará:

```text
CorrelationDependencySet
```

---

# 187. Correlated node

Podrá declarar:

```text
required outer symbols
```

---

# 188. Outer symbol

No se resolverá nuevamente por nombre.

Se referencia mediante identidad semántica.

---

# 189. Correlation graph

Ejemplo:

```text
Outer Scan
    │
    ├──────────────┐
    │              │
    ▼              ▼
Outer Plan     Correlated Subplan
                    │
                    └── requires SymbolId(U.id)
```

---

# 190. Logical plan cycles

El data-flow normal será acíclico.

---

# 191. Recursive CTE exception

La recursión se representará mediante estructura explícita controlada.

No mediante ciclos arbitrarios de objetos.

---

# 192. Structured recursion

Preferible:

```text
LogicalRecursiveCte
├── anchor
└── recursiveMember
```

con metadata de recursión.

---

# 193. Graph validation

El validator comprobará:

```text
valid root
reachable nodes
valid inputs
no illegal data cycles
valid recursion structures
valid output schemas
valid correlations
valid dependencies
```

---

# 194. LogicalPlanValidator

```php
interface LogicalPlanValidator
{
    public function validate(
        LogicalQueryPlan $plan,
        LogicalPlanValidationContext $context,
    ): LogicalPlanValidationResult;
}
```

---

# 195. Validation ≠ repair

No modificará silenciosamente el plan.

---

# 196. Invalid lowering

Si lowering produce un plan inválido:

```text
internal architecture error
```

no una oportunidad para improvisar.

---

# 197. Determinism

Mismo:

```text
OptimizedQueryArtifact
LogicalPlannerVersion
ExtensionSet
Configuration
```

deberá producir el mismo Logical Plan.

---

# 198. Deterministic node allocation

Los IDs podrán ser operation-local pero su asignación seguirá traversal determinista.

---

# 199. Fingerprint does not depend on random IDs

`LogicalPlanFingerprint` no deberá depender de identificadores efímeros arbitrarios.

---

# 200. Structural fingerprint

Se calculará desde:

```text
operator kinds
semantic identities
expression/predicate fingerprints
child structure
logical metadata
extension versions
```

---

# 201. Runtime values excluded

No se incluirán valores concretos de:

```text
BindingSet
```

en el fingerprint genérico.

---

# 202. LogicalPlanFingerprint

Conceptualmente:

```text
Fingerprint(
    OptimizedQueryFingerprint,
    LogicalOperatorStructure,
    SemanticReferences,
    OutputContract,
    LogicalPlannerVersion,
    ExtensionVersions
)
```

---

# 203. Statistics excluded

Las estadísticas físicas no deberán afectar:

```text
LogicalPlanFingerprint
```

si el Logical Plan no depende de ellas.

---

# 204. Cost hints

Igualmente, hints puramente físicos no cambian el Logical Plan.

---

# 205. Semantic hints

Si un hint forma parte del contrato lógico, deberá haber sido materializado antes.

---

# 206. Logical Plan cache

Podrá existir posteriormente:

```text
OptimizedQueryFingerprint
        ↓
LogicalPlanCache
        ↓
LogicalQueryPlan
```

---

# 207. Cache invalidation

Dependerá de:

```text
semantic dependencies
logical planner version
extension versions
relevant capability contract versions
```

---

# 208. Schema changes

Si cambian elementos semánticamente relevantes:

```text
types
columns
constraints
relation identities
```

el plan puede invalidarse.

---

# 209. Index changes

Un cambio exclusivamente de índice normalmente:

```text
does not invalidate LogicalQueryPlan
```

pero sí puede invalidar PhysicalQueryPlan.

---

# 210. Important separation

```text
Add index
    │
    ├── Logical Plan: often reusable
    │
    └── Physical Plan: potentially replan
```

---

# 211. Statistics change

Normalmente:

```text
Logical Plan remains valid
Physical Plan may change
```

---

# 212. Platform change

Si el logical semantics son portables:

```text
Logical Plan may remain reusable
```

mientras:

```text
Physical Plan must be regenerated
```

---

# 213. Capability requirements

El Logical Plan declarará requerimientos.

No decidirá necesariamente la estrategia de emulación.

---

# 214. Portability

Cada logical node podrá contribuir a:

```text
LogicalPortabilityProfile
```

---

# 215. Logical portability levels

Conceptualmente:

```text
PORTABLE
CAPABILITY_DEPENDENT
EXTENSION_DEPENDENT
PLATFORM_RESTRICTED
```

---

# 216. Raw expressions

Un raw expression puede producir:

```text
LogicalRawExpressionReference
```

dentro de un operador.

---

# 217. Raw barrier preservation

El Logical Plan conservará:

```text
RawBarrier
```

para evitar que Physical Planner suponga propiedades desconocidas.

---

# 218. Raw ≠ logical extension

Si una característica tiene semántica conocida y recurrente:

```text
Semantic Extension
```

es preferible a raw.

---

# 219. Security provenance

Logical nodes derivados de políticas podrán conservar:

```text
PolicyProvenance
```

---

# 220. Security filters

Ejemplo:

```text
LogicalFilter
predicate = tenant_id = :tenant
provenance = MandatoryTenantPolicy
```

---

# 221. Physical Planner cannot discard it

El costo nunca será razón suficiente para removerlo.

---

# 222. Security barrier nodes

En algunos casos podrá existir:

```text
LogicalSecurityBarrier
```

si es necesario expresar una frontera de evaluación.

---

# 223. Security barrier ≠ ordinary filter

Puede imponer restricciones adicionales de reorder/pushdown/materialization.

---

# 224. Volatility

Logical expressions conservarán información de:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

o modelo equivalente.

---

# 225. Volatility influences physical planning

Ejemplo:

```text
do not duplicate evaluation
```

pero no convierte al Logical Plan en Physical Plan.

---

# 226. Side effects

Funciones/extensiones con efectos observables deberán declararlos explícitamente.

---

# 227. LogicalEffectProperties

Podrá representar:

```text
PURE
READS_EXTERNAL_STATE
VOLATILE
SIDE_EFFECTING
UNKNOWN
```

según contratos futuros.

---

# 228. Ordering observability

No todo ordering interno es observable.

---

# 229. Observable ordering

Ejemplo:

```sql
ORDER BY created_at
```

sí pertenece al contrato.

---

# 230. Internal ordering

Ordering requerido por un algoritmo físico:

```text
does not belong in LogicalQueryPlan
```

---

# 231. Aggregate ordering

Un aggregate ordenado puede tener ordering como parte de su propia semántica.

Ejemplo conceptual:

```text
aggregate(... ORDER BY ...)
```

Eso no implica ordering global del result set.

---

# 232. Window ordering

Igualmente:

```text
WindowOrdering
≠
QueryOrdering
```

---

# 233. Logical Plan must preserve distinction

Nunca se fusionarán accidentalmente.

---

# 234. Distinctness

También se distinguirán:

```text
Query DISTINCT
Aggregate DISTINCT
Set DISTINCT
```

---

# 235. Logical operators reflect scope

Cada forma de distinctness permanecerá asociada a su operador correspondiente.

---

# 236. Mutation plan example

Consulta:

```sql
UPDATE users
SET status = ?
WHERE last_login < ?
RETURNING id;
```

Logical:

```text
LogicalReturning(id)
        │
LogicalUpdate
  target = users
  assignments = status
        │
LogicalFilter(last_login < ?)
        │
LogicalScan(users)
```

---

# 237. Physical decisions absent

No aparece:

```text
IndexRangeScan(last_login)
BitmapScan
RowIdMutation
BatchUpdate
```

---

# 238. Delete example

```sql
DELETE FROM sessions
WHERE expires_at < ?;
```

Logical:

```text
LogicalDelete
      │
LogicalFilter
      │
LogicalScan(sessions)
```

---

# 239. Insert-select example

```sql
INSERT INTO archive_users (...)
SELECT ...
FROM users
WHERE ...;
```

Logical:

```text
LogicalInsert(archive_users)
        │
LogicalProject
        │
LogicalFilter
        │
LogicalScan(users)
```

---

# 240. INSERT VALUES example

```text
LogicalInsert(users)
       │
LogicalValues
```

---

# 241. Conflict action

`ON CONFLICT` / upsert semantics serán representadas estructuralmente.

---

# 242. Vendor-independent conflict model

No:

```text
ON DUPLICATE KEY UPDATE ...
```

como string dentro del logical plan.

---

# 243. Logical mutation constraints

El plan podrá conservar:

```text
UnboundedMutationMarker
SafetyApprovalMetadata
MutationTargetContract
```

provenientes de etapas anteriores.

---

# 244. Planner cannot bypass mutation safety

Un physical strategy no podrá convertir una mutation restringida en unrestricted mutation.

---

# 245. Plan visitors

Para tooling:

```php
interface LogicalPlanVisitor
{
    public function visit(
        LogicalPlanNode $node,
        LogicalPlanVisitContext $context,
    ): void;
}
```

---

# 246. Visitor ≠ mutation API

El visitor por defecto será read-only.

---

# 247. Plan transformation

Cualquier transformación deberá producir:

```text
new LogicalQueryPlan
```

no mutar el artifact congelado.

---

# 248. Post-lowering logical transformations

Idealmente serán mínimas.

Si se requieren optimizaciones lógicas adicionales:

```text
they belong to Optimizer
```

---

# 249. Physical Planner may construct alternatives

Eso no deberá modificar el Logical Plan original.

---

# 250. Shared logical subplans

Podrán representarse mediante:

```text
LogicalPlanNodeId references
```

evitando duplicación estructural cuando sea seguro.

---

# 251. Shared subplan ≠ automatic materialization

Compartir representación lógica no significa ejecutar una sola vez.

---

# 252. Execution multiplicity

Es una decisión posterior.

---

# 253. Plan graph ownership

Todos los nodos pertenecerán explícitamente a un:

```text
LogicalQueryPlan
```

---

# 254. Cross-plan node references

No estarán permitidas salvo artifacts explícitos de composición.

---

# 255. Subplans

Los subplans podrán poseer:

```text
SubplanId
```

dentro del mismo Logical Plan artifact.

---

# 256. Subplan registry

```text
LogicalSubplanSet
```

podrá mantener:

```text
scalar subqueries
CTEs
recursive members
extension subplans
```

---

# 257. Subplan scope

Cada subplan conservará su scope/correlation contract.

---

# 258. Scope identities

No se reconstruirán desde aliases.

---

# 259. LogicalPlanContext

Durante lowering:

```php
final readonly class LogicalPlanContext
{
    public function __construct(
        public SemanticQueryArtifact $semantic,
        public OptimizedQueryArtifact $optimized,
        public LogicalPlanConfiguration $configuration,
        public LogicalPlanExtensionSet $extensions,
    ) {}
}
```

---

# 260. No service locator

`LogicalPlanContext` no expondrá:

```text
Container
ConnectionManager
EntityManager
Executor
HTTP Request
```

---

# 261. LogicalPlanBuilder

Contrato:

```php
interface LogicalPlanBuilder
{
    public function build(
        OptimizedQueryArtifact $query,
        LogicalPlanContext $context,
    ): LogicalQueryPlan;
}
```

---

# 262. Lowering handlers

Se favorecerá composición:

```text
RelationLowerer
PredicateLowerer
ProjectionLowerer
AggregateLowerer
WindowLowerer
SetOperationLowerer
SubqueryLowerer
CteLowerer
MutationLowerer
```

---

# 263. No MegaLowerer

Evitar una clase:

```text
LogicalPlanBuilder.php
```

con miles de líneas y conocimiento de todo el sistema.

---

# 264. Lowerer registry

Extensiones podrán registrar handlers mediante:

```text
LogicalLoweringRegistry
```

---

# 265. Registry lifecycle

```text
bootstrap
   ↓
discover
   ↓
validate
   ↓
resolve
   ↓
freeze
```

---

# 266. No runtime registration

Con workers persistentes:

```text
registry mutation after bootstrap
```

estará prohibida.

---

# 267. Extension conflicts

No:

```text
last registered wins
```

---

# 268. Stable extension ownership

Cada extension logical node/handler tendrá:

```text
ExtensionId
Version
LogicalNodeTypeId
```

---

# 269. Fingerprint contribution

Toda extensión que afecte estructura lógica deberá contribuir al fingerprint.

---

# 270. Serialization

Logical Plan podrá ser serializable si todos sus componentes lo son.

---

# 271. No closures

Artifact final no contendrá closures.

---

# 272. No service instances

Artifact final no contendrá servicios runtime.

---

# 273. No connections

Artifact final no contendrá conexiones.

---

# 274. No live schema objects

Utilizará snapshots/identities/contracts serializables.

---

# 275. Codec versioning

```text
LogicalPlanCodecVersion
```

será explícita.

---

# 276. Deserialization validation

Un plan deserializado deberá validar:

```text
codec version
planner version compatibility
dependencies
extensions
node kinds
fingerprint
```

---

# 277. Unknown node

Nunca:

```text
unknown node
→ RawNode
```

---

# 278. Persistent runtime safety

Shared:

```text
LoweringRegistry
LogicalNodeDescriptors
LogicalPropertyRules
CodecDescriptors
```

serán frozen/immutable.

---

# 279. Operation-local

```text
NodeAllocator
GraphBuilder
PropertyWorklist
MappingBuilder
DiagnosticCollector
FingerprintBuilder
```

---

# 280. Request reset

Todo estado mutable de lowering deberá descartarse al terminar.

---

# 281. No static counters

No:

```php
static $nextLogicalNodeId;
```

---

# 282. Deterministic allocator

Se utilizará:

```text
operation-local deterministic allocator
```

---

# 283. Concurrency

Dos operaciones podrán construir planes simultáneamente sin interferencia.

---

# 284. Telemetry

Métricas:

```text
logical plan build duration
logical nodes
logical edges
subplans
CTEs
correlated subplans
recursive plans
logical depth
property derivations
validation failures
cache hits/misses
```

---

# 285. Cardinality-safe telemetry

No usar:

```text
LogicalPlanFingerprint
```

como etiqueta de alta cardinalidad en métricas agregadas.

---

# 286. Sensitive values

Nunca se incluirán runtime parameter values en telemetry del plan.

---

# 287. Debug representation

Podrá existir:

```text
LogicalPlanPrinter
```

---

# 288. Example output

```text
LogicalLimit [10]
└── LogicalSort [orders.total DESC]
    └── LogicalProject [users.id, orders.total]
        └── LogicalJoin [INNER]
            ├── LogicalFilter [users.active = :p1]
            │   └── LogicalScan [users]
            └── LogicalScan [orders]
```

---

# 289. Printer ≠ SQL generator

El printer es diagnóstico.

No produce SQL ejecutable.

---

# 290. Explain logical

API conceptual:

```php
$database->explain($query)->logical();
```

podría mostrar el Logical Plan.

---

# 291. Explain stages

Futuro:

```text
Semantic
Optimized
Logical
Physical
Execution
SQL
```

como niveles separados.

---

# 292. Testing strategy

Se requerirán:

```text
unit tests
lowering tests
property derivation tests
graph validation tests
fingerprint tests
serialization tests
extension conformance tests
persistent runtime tests
concurrency tests
```

---

# 293. Golden plan tests

Consultas conocidas podrán compararse contra una representación lógica estable.

---

# 294. Golden tests must avoid ephemeral IDs

Las snapshots deberán usar:

```text
canonical structural representation
```

---

# 295. Property tests

Ejemplo:

```text
Filter never increases proven maximum cardinality.
```

---

# 296. Join property tests

Deberán cubrir:

```text
INNER
LEFT
RIGHT
FULL
SEMI
ANTI
```

---

# 297. NULL tests

Especialmente:

```text
outer joins
NOT IN
aggregates
set operations
```

---

# 298. Bag semantics tests

Para:

```text
UNION ALL
INTERSECT ALL
EXCEPT ALL
DISTINCT
```

---

# 299. Correlation tests

Cubrirán:

```text
scalar correlated subquery
EXISTS
NOT EXISTS
lateral relation
nested correlations
```

---

# 300. Recursive tests

Cubrirán:

```text
anchor
recursive member
UNION
UNION ALL
dependency validation
illegal cycles
```

---

# 301. Mutation tests

Cubrirán:

```text
INSERT VALUES
INSERT SELECT
UPDATE
DELETE
RETURNING
conflict actions
unbounded mutation markers
```

---

# 302. Extension tests

Toda extensión deberá demostrar:

```text
deterministic lowering
valid output
valid properties
stable fingerprint
serialization compatibility
physical planning compatibility
```

---

# 303. Error model

Excepciones posibles:

```text
LogicalPlanLoweringException
UnsupportedLogicalConstructException
LogicalPlanInvariantException
LogicalPlanGraphException
LogicalPlanPropertyException
LogicalPlanOutputContractException
LogicalPlanExtensionException
LogicalPlanSerializationException
LogicalPlanCompatibilityException
```

---

# 304. Unsupported construct

Si una construcción optimizada no tiene lowering válido:

```text
fail explicitly
```

---

# 305. No raw fallback

Nunca:

```text
unsupported semantic node
→ SQL raw
```

---

# 306. No physical fallback

Tampoco:

```text
unsupported logical node
→ arbitrary physical operator
```

---

# 307. Architectural formula

```text
LogicalQueryPlan
=
Lower(
    OptimizedQueryArtifact,
    SemanticQueryArtifact,
    LogicalLoweringRules
)
```

sujeto a:

```text
Semantic Preservation
∧
Output Contract Preservation
∧
Dependency Preservation
∧
Barrier Preservation
∧
Deterministic Structure
```

---

# 308. Logical property formula

Para nodo `N`:

```text
LogicalProperties(N)
=
Derive(
    Semantics(N),
    LogicalProperties(children(N)),
    SemanticFacts(N)
)
```

---

# 309. No cost in logical truth

Nunca:

```text
LogicalProperties(N)
=
statistics-based guesses
```

Las estimaciones pertenecen al Planning/Cost layer.

---

# 310. Lowering correctness

Formalmente, si:

```text
Q = OptimizedQueryArtifact
L = LogicalQueryPlan
```

entonces:

```text
Semantics(Q)
=
Semantics(L)
```

---

# 311. Logical Plan invariants

## DB-LPLAN-001

LogicalQueryPlan será distinto de Query AST.

## DB-LPLAN-002

LogicalQueryPlan será distinto de SemanticQueryArtifact.

## DB-LPLAN-003

LogicalQueryPlan será distinto de OptimizedQueryArtifact.

## DB-LPLAN-004

LogicalQueryPlan será distinto de PhysicalQueryPlan.

## DB-LPLAN-005

LogicalQueryPlan será distinto de ExecutionPlan.

## DB-LPLAN-006

LogicalQueryPlan no contendrá SQL generado.

## DB-LPLAN-007

LogicalQueryPlan no contendrá placeholders de plataforma.

## DB-LPLAN-008

LogicalQueryPlan no contendrá PDO.

## DB-LPLAN-009

LogicalQueryPlan no contendrá Connection.

## DB-LPLAN-010

LogicalQueryPlan no contendrá Statement.

## DB-LPLAN-011

Logical lowering no ejecutará queries.

## DB-LPLAN-012

Logical lowering no hará schema introspection oculta.

## DB-LPLAN-013

Logical lowering no realizará physical planning.

## DB-LPLAN-014

Logical lowering no repetirá optimizaciones globales.

## DB-LPLAN-015

SemanticQueryArtifact seguirá siendo autoridad semántica.

## DB-LPLAN-016

Logical Plan no resolverá símbolos por string.

## DB-LPLAN-017

Logical Plan utilizará semantic identities.

## DB-LPLAN-018

SemanticNodeId será distinto de LogicalPlanNodeId.

## DB-LPLAN-019

LogicalPlanNodeId será distinto de PhysicalPlanNodeId.

## DB-LPLAN-020

Logical node IDs serán operation-scoped.

## DB-LPLAN-021

Fingerprint no dependerá de IDs efímeros.

## DB-LPLAN-022

LogicalScan será distinto de SequentialScan.

## DB-LPLAN-023

LogicalScan será distinto de IndexScan.

## DB-LPLAN-024

LogicalFilter será distinto de index predicate.

## DB-LPLAN-025

LogicalProject será distinto de SELECT SQL.

## DB-LPLAN-026

LogicalJoin será distinto de HashJoin.

## DB-LPLAN-027

LogicalJoin será distinto de MergeJoin.

## DB-LPLAN-028

LogicalJoin será distinto de NestedLoopJoin.

## DB-LPLAN-029

LogicalAggregate será distinto de HashAggregate.

## DB-LPLAN-030

LogicalWindow será distinto de physical window strategy.

## DB-LPLAN-031

LogicalSort será distinto de PhysicalSort.

## DB-LPLAN-032

LogicalLimit será distinto de TopN.

## DB-LPLAN-033

LogicalDistinct será distinto de HashDistinct.

## DB-LPLAN-034

Set operations preservarán quantifier ALL/DISTINCT.

## DB-LPLAN-035

Set operation tree grouping será preservado.

## DB-LPLAN-036

Bag semantics serán preservadas.

## DB-LPLAN-037

Outer join null-extension será preservada.

## DB-LPLAN-038

SEMI JOIN conservará su semántica.

## DB-LPLAN-039

ANTI JOIN conservará su semántica.

## DB-LPLAN-040

NOT IN no se convertirá ingenuamente en ANTI JOIN.

## DB-LPLAN-041

Global aggregate será distinto de query no agregada.

## DB-LPLAN-042

Empty grouping será explícito.

## DB-LPLAN-043

Window ordering será distinto de query ordering.

## DB-LPLAN-044

Aggregate ordering será distinto de query ordering.

## DB-LPLAN-045

Query DISTINCT será distinto de Aggregate DISTINCT.

## DB-LPLAN-046

Set DISTINCT será distinto de Query DISTINCT.

## DB-LPLAN-047

LIMIT sin ORDER BY no inventará ordering.

## DB-LPLAN-048

LogicalMaterialize será distinto de PhysicalMaterialize.

## DB-LPLAN-049

CTE reference será identity-based.

## DB-LPLAN-050

CteId será distinto de CTE name.

## DB-LPLAN-051

Recursive CTE será estructurado explícitamente.

## DB-LPLAN-052

Recursion no utilizará ciclos arbitrarios de objetos.

## DB-LPLAN-053

EmptyRelation será un operador válido.

## DB-LPLAN-054

EmptyRelation conservará output schema.

## DB-LPLAN-055

LogicalInsert será vendor-independent.

## DB-LPLAN-056

LogicalUpdate será vendor-independent.

## DB-LPLAN-057

LogicalDelete será vendor-independent.

## DB-LPLAN-058

Mutation semantics serán preservadas.

## DB-LPLAN-059

Direct DML será distinto de ORM persistence.

## DB-LPLAN-060

RETURNING será representado estructuralmente.

## DB-LPLAN-061

Conflict actions serán estructuradas.

## DB-LPLAN-062

Logical Plan será conceptualmente un graph.

## DB-LPLAN-063

Data edge será distinto de dependency edge.

## DB-LPLAN-064

Dependency edge será distinto de correlation edge.

## DB-LPLAN-065

Correlation será distinta de lineage.

## DB-LPLAN-066

Lineage será distinta de provenance.

## DB-LPLAN-067

Cada relation-producing node tendrá output schema.

## DB-LPLAN-068

Output identity será distinta de output name.

## DB-LPLAN-069

Duplicate output names serán soportados.

## DB-LPLAN-070

Root satisfará LogicalOutputContract.

## DB-LPLAN-071

Logical properties serán distintas de physical properties.

## DB-LPLAN-072

Proven cardinality será distinta de estimated cardinality.

## DB-LPLAN-073

Statistics no serán semantic facts.

## DB-LPLAN-074

Unknown uniqueness será distinta de non-unique.

## DB-LPLAN-075

Functional dependencies sólo se conservarán si siguen demostrables.

## DB-LPLAN-076

Filter no aumentará cardinalidad lógica demostrable.

## DB-LPLAN-077

Projection invalidará facts que deje de preservar.

## DB-LPLAN-078

Join derivará nullability correctamente.

## DB-LPLAN-079

Aggregate derivará group uniqueness correctamente.

## DB-LPLAN-080

DISTINCT derivará row uniqueness.

## DB-LPLAN-081

LIMIT podrá derivar upper cardinality bound.

## DB-LPLAN-082

Properties podrán almacenarse en side tables.

## DB-LPLAN-083

Logical nodes no tendrán metadata mutable gigante.

## DB-LPLAN-084

Semantic-to-logical mapping será explícito.

## DB-LPLAN-085

Semantic-to-logical mapping será distinto de lineage.

## DB-LPLAN-086

Logical provenance será preservable.

## DB-LPLAN-087

Logical metadata no contendrá runtime services.

## DB-LPLAN-088

Logical dependencies serán explícitas.

## DB-LPLAN-089

Dependencies participarán en invalidación.

## DB-LPLAN-090

Logical capabilities serán distintas de physical strategies.

## DB-LPLAN-091

Logical canonicalization será distinta de Query Normalization.

## DB-LPLAN-092

Logical canonicalization no repetirá join optimization.

## DB-LPLAN-093

Lowering tendrá fases explícitas.

## DB-LPLAN-094

Lowering validará input.

## DB-LPLAN-095

Lowering validará output contract.

## DB-LPLAN-096

Lowering producirá graph válido.

## DB-LPLAN-097

WHERE y HAVING podrán usar mismo operator kind sin perder phase semantics.

## DB-LPLAN-098

Graph position participará en semántica.

## DB-LPLAN-099

Window phase será preservada.

## DB-LPLAN-100

Scalar subquery preservará cardinality semantics.

## DB-LPLAN-101

EXISTS podrá permanecer como subplan.

## DB-LPLAN-102

Decorrelated EXISTS podrá aparecer como SEMI JOIN.

## DB-LPLAN-103

Decorrelated NOT EXISTS podrá aparecer como ANTI JOIN.

## DB-LPLAN-104

Correlated subplans declararán required outer symbols.

## DB-LPLAN-105

Outer symbols serán identity-based.

## DB-LPLAN-106

Illegal graph cycles serán rechazados.

## DB-LPLAN-107

Recursive structures serán validadas.

## DB-LPLAN-108

LogicalPlanValidator no reparará silenciosamente artifacts.

## DB-LPLAN-109

Logical Plan será determinista.

## DB-LPLAN-110

Fingerprint será estructural.

## DB-LPLAN-111

Runtime binding values no participarán en fingerprint genérico.

## DB-LPLAN-112

Statistics no participarán en LogicalPlanFingerprint salvo dependencia lógica explícita.

## DB-LPLAN-113

Physical cost hints no cambiarán Logical Plan.

## DB-LPLAN-114

Index changes normalmente no invalidarán Logical Plan.

## DB-LPLAN-115

Index changes podrán invalidar Physical Plan.

## DB-LPLAN-116

Statistics changes normalmente no invalidarán Logical Plan.

## DB-LPLAN-117

Statistics changes podrán cambiar Physical Plan.

## DB-LPLAN-118

Logical capability requirements serán explícitos.

## DB-LPLAN-119

Raw barriers serán preservadas.

## DB-LPLAN-120

Unknown raw semantics no serán inventadas.

## DB-LPLAN-121

Security provenance será preservada.

## DB-LPLAN-122

Mandatory security filters no serán removibles por Physical Planner.

## DB-LPLAN-123

Security barriers serán distintas de ordinary filters cuando sea necesario.

## DB-LPLAN-124

Volatility será preservada.

## DB-LPLAN-125

Side-effect properties serán preservadas.

## DB-LPLAN-126

Observable ordering será distinto de internal physical ordering.

## DB-LPLAN-127

Logical mutation safety metadata será preservada.

## DB-LPLAN-128

Visitors serán read-only por defecto.

## DB-LPLAN-129

Frozen Logical Plan no será mutado.

## DB-LPLAN-130

Post-lowering optimization global pertenecerá al Optimizer.

## DB-LPLAN-131

Shared logical subplan no implicará single execution.

## DB-LPLAN-132

Subplans conservarán scopes.

## DB-LPLAN-133

LogicalPlanContext no será service locator.

## DB-LPLAN-134

Lowering se dividirá en handlers especializados.

## DB-LPLAN-135

No habrá MegaLowerer obligatorio.

## DB-LPLAN-136

Extension registry será frozen después de bootstrap.

## DB-LPLAN-137

Extension conflicts no usarán last-one-wins.

## DB-LPLAN-138

Extensions contribuirán al fingerprint.

## DB-LPLAN-139

Logical Plan final no contendrá closures.

## DB-LPLAN-140

Logical Plan final no contendrá service instances.

## DB-LPLAN-141

Logical Plan serialization será versionada.

## DB-LPLAN-142

Unknown serialized node no tendrá raw fallback.

## DB-LPLAN-143

Shared registries serán immutable.

## DB-LPLAN-144

Construction state será operation-scoped.

## DB-LPLAN-145

No habrá static node counters.

## DB-LPLAN-146

Concurrent plan construction estará aislada.

## DB-LPLAN-147

Telemetry no expondrá parameter values.

## DB-LPLAN-148

LogicalPlanPrinter no será SQL generator.

## DB-LPLAN-149

Extensions tendrán conformance tests.

## DB-LPLAN-150

Unsupported lowering fallará explícitamente.

## DB-LPLAN-151

No habrá SQL raw fallback para semantic nodes desconocidos.

## DB-LPLAN-152

Logical Plan preservará semántica del OptimizedQueryArtifact.

---

# 312. Anti-patterns

## 312.1 Logical node con SQL

```php
new LogicalScan('SELECT * FROM users');
```

**Rechazado.**

---

## 312.2 LogicalScan eligiendo índice

```php
new LogicalScan(
    table: 'users',
    index: 'idx_users_email',
);
```

**Rechazado.**

El índice pertenece a physical planning.

---

## 312.3 LogicalJoin con algoritmo físico

```php
new LogicalJoin(
    algorithm: 'hash'
);
```

**Rechazado.**

---

## 312.4 LogicalAggregate con memory budget

**Rechazado.**

---

## 312.5 LogicalSort con algoritmo quicksort

**Rechazado.**

---

## 312.6 LogicalLimit inventando ordering

**Rechazado.**

---

## 312.7 Resolver columnas otra vez

```php
$schema->findColumn('users', 'id');
```

durante lowering.

**Rechazado.**

La identidad ya fue resuelta semánticamente.

---

## 312.8 Statistics como semantic property

```text
rowCount = 1,234
```

tratado como verdad.

**Rechazado.**

---

## 312.9 CTE name como identidad

**Rechazado.**

---

## 312.10 SQL raw como fallback de logical extension

**Rechazado.**

---

## 312.11 Mutar el Logical Plan desde Physical Planner

**Rechazado.**

---

## 312.12 Guardar current plan en singleton

**Rechazado.**

---

## 312.13 Logical Plan con PDO

**Rechazado.**

---

## 312.14 Logical Plan con EntityManager

**Rechazado.**

---

## 312.15 Logical Plan con runtime tenant object

**Rechazado.**

---

## 312.16 Reordenar joins nuevamente durante lowering

**Rechazado.**

---

## 312.17 Convertir NOT IN automáticamente en AntiJoin

**Rechazado.**

---

## 312.18 Tratar WindowOrdering como QueryOrdering

**Rechazado.**

---

## 312.19 Tratar shared subplan como materialización obligatoria

**Rechazado.**

---

## 312.20 Ignorar security provenance durante lowering

**Rechazado.**

---

# 313. Arquitectura interna consolidada

```text
                 OptimizedQueryArtifact
                          │
                          ▼
               LogicalPlanCoordinator
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
 RelationLowerer    ExpressionLowerer    MutationLowerer
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ▼
                 Logical Nodes
                          │
                          ▼
                 Graph Assembler
                          │
                          ▼
                 LogicalPlanGraph
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
       Properties      Mapping      Dependencies
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                Output Validation
                          │
                          ▼
                  Fingerprinting
                          │
                          ▼
                       Freeze
                          │
                          ▼
                 LogicalQueryPlan
```

---

# 314. Logical operator hierarchy

```text
LogicalPlanNode
│
├── Source
│   ├── LogicalScan
│   ├── LogicalValues
│   ├── LogicalEmptyRelation
│   └── LogicalCteReference
│
├── Unary Relational
│   ├── LogicalFilter
│   ├── LogicalProject
│   ├── LogicalAggregate
│   ├── LogicalWindow
│   ├── LogicalSort
│   ├── LogicalDistinct
│   ├── LogicalLimit
│   ├── LogicalOffset
│   └── LogicalMaterialize
│
├── Multi-Input Relational
│   ├── LogicalJoin
│   └── LogicalSetOperation
│
├── Subplan
│   ├── LogicalScalarSubquery
│   ├── LogicalExistsSubquery
│   └── LogicalRecursiveCte
│
├── Mutation
│   ├── LogicalInsert
│   ├── LogicalUpdate
│   ├── LogicalDelete
│   └── LogicalReturning
│
└── Extension
    └── LogicalExtensionNode
```

---

# 315. Query pipeline final

```text
Developer Query
      │
      ▼
Query Builder
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
OptimizedQueryArtifact
      │
      ▼
Logical Lowering
      │
      ▼
LogicalQueryPlan
      │
      ▼
Physical Planning
      │
      ▼
PhysicalQueryPlan
      │
      ▼
Execution Planning
      │
      ▼
ExecutionPlan
      │
      ▼
Compiler
      │
      ▼
Compiled Query
      │
      ▼
Executor
```

---

# 316. Principio de diseño final

La separación deberá ser siempre:

```text
AST
    describes query structure.

Semantic Graph
    describes query meaning.

Optimized Query
    describes an equivalent preferred logical form.

Logical Query Plan
    describes required relational operations.

Physical Query Plan
    describes implementation strategies.

Execution Plan
    describes executable work.

Compiler
    describes target representation.

Executor
    performs that work.
```

---

# 317. Fórmula arquitectónica definitiva

```text
Logical Query Planning
=
Semantic-to-Relational Lowering
+
Logical Operator Modeling
+
Logical Dependency Modeling
+
Logical Property Derivation
+
Output Contract Preservation
+
Barrier Preservation
+
Deterministic Graph Construction
```

Nunca:

```text
Logical Query Planning
=
SQL Generation
+
Index Selection
+
Join Algorithm Selection
```

---

# 318. Estado del bloque

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md              ← actual
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 319. Siguiente documento

```text
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
```

El siguiente documento formalizará la transición:

```text
LogicalQueryPlan
       │
       ▼
Physical Planning
       │
       ▼
PhysicalQueryPlan
```

y definirá en detalle:

```text
PhysicalPlanNode
PhysicalScan
SequentialScan
IndexScan
IndexOnlyScan
PhysicalFilter
PhysicalProjection
NestedLoopJoin
IndexNestedLoopJoin
HashJoin
MergeJoin
HashAggregate
SortAggregate
PhysicalWindow
PhysicalSort
TopN
PhysicalDistinct
PhysicalSetOperation
PhysicalMaterialize
PhysicalSpool
PhysicalMutation
PhysicalReturning
AccessPath
RequiredPhysicalProperties
ProvidedPhysicalProperties
PropertyEnforcer
PhysicalResourceRequirement
ExecutionControlLevel
PhysicalPlanCandidate
PhysicalPlanCost
PhysicalPlanFingerprint
```

manteniendo como invariante fundamental:

```text
LogicalQueryPlan
    says what relational operations are required.

PhysicalQueryPlan
    says how those operations are intended to be performed.
```