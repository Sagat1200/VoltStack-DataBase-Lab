# 64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md

# VoltStack Quantum Database
## Physical Query Plan System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 64 — Physical Query Plan System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Planner / Physical Planning  
**Versión:** 1.0

---

# 1. Propósito

`Physical Query Plan System` define la representación física intermedia mediante la cual VoltStack transforma un:

```text
LogicalQueryPlan
```

en una estrategia concreta de acceso y procesamiento capaz de satisfacer exactamente sus requisitos lógicos.

Su pregunta fundamental es:

> ¿Cómo deben implementarse físicamente las operaciones definidas por el Logical Query Plan?

Formalmente:

```text
PhysicalPlanning(
    LogicalQueryPlan,
    PhysicalPlanningContext
)
→
PhysicalQueryPlan
```

La distinción central será:

```text
LogicalQueryPlan
=
what relational operations are required

PhysicalQueryPlan
=
how those operations should be implemented
```

---

# 2. Posición arquitectónica

```text
Query Builder
     │
     ▼
Query AST
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
Logical Plan System
     │
     ▼
LogicalQueryPlan
     │
     ▼
┌─────────────────────────────────────┐
│      Physical Query Plan System     │
│                                     │
│ Access Path Selection               │
│ Physical Operator Selection         │
│ Physical Property Planning          │
│ Physical Alternative Evaluation     │
│ Property Enforcement                │
│ Resource Requirement Modeling       │
│ Physical Cost Evaluation            │
└─────────────────────────────────────┘
     │
     ▼
PhysicalQueryPlan
     │
     ▼
Execution Plan System
     │
     ▼
ExecutionPlan
```

---

# 3. Frontera arquitectónica

El sistema físico sí puede decidir:

```text
SequentialScan
IndexScan
IndexOnlyScan
NestedLoopJoin
IndexNestedLoopJoin
HashJoin
MergeJoin
HashAggregate
SortAggregate
PhysicalSort
TopN
HashDistinct
SortDistinct
Materialize
Spool
```

pero todavía no debe:

```text
open Connection
prepare Statement
bind runtime values
execute SQL
consume ResultCursor
hydrate entities
```

---

# 4. Physical Plan ≠ Logical Plan

Ejemplo lógico:

```text
LogicalJoin
├── LogicalScan(users)
└── LogicalScan(orders)
```

podría producir distintas alternativas:

```text
HashJoin
├── SequentialScan(users)
└── SequentialScan(orders)
```

o:

```text
NestedLoopJoin
├── SequentialScan(users)
└── IndexScan(orders.user_id)
```

o:

```text
MergeJoin
├── IndexScan(users.id)
└── IndexScan(orders.user_id)
```

Las tres pueden implementar el mismo join lógico.

---

# 5. Physical Plan ≠ Execution Plan

Esta separación será fundamental.

```text
PhysicalQueryPlan
=
selected physical relational strategy

ExecutionPlan
=
executable operational representation
```

El Physical Plan puede expresar:

```text
HashJoin
```

mientras el Execution Plan posteriormente determina detalles como:

```text
operator initialization
resource handles
statement preparation
runtime parameter slots
execution dependencies
cleanup
cancellation
instrumentation
```

---

# 6. Physical Plan ≠ SQL

VoltStack no diseñará el Physical Plan como una cadena SQL.

Incorrecto:

```php
new PhysicalPlan(
    sql: 'SELECT ...'
);
```

Correcto conceptualmente:

```text
PhysicalProject
      │
PhysicalHashJoin
   ┌──┴──┐
Scan    Scan
```

---

# 7. Entrada principal

El input será:

```php
LogicalQueryPlan
```

acompañado de un contexto físico explícito.

---

# 8. PhysicalPlanningContext

Modelo conceptual:

```php
final readonly class PhysicalPlanningContext
{
    public function __construct(
        public PlatformCapabilitySnapshot $capabilities,
        public PhysicalCatalogSnapshot $catalog,
        public StatisticsSnapshot $statistics,
        public CostModelSnapshot $costModel,
        public QueryCostHintSet $costHints,
        public PhysicalPlanningConfiguration $configuration,
        public PhysicalPlanningBudget $budget,
        public PhysicalExtensionSet $extensions,
    ) {}
}
```

---

# 9. No service locator

`PhysicalPlanningContext` no expondrá arbitrariamente:

```text
Container
Application
EntityManager
HTTP Request
Global Tenant Resolver
Executor
```

---

# 10. Snapshot principle

Toda información utilizada por el planner deberá llegar mediante snapshots explícitos.

Ejemplo:

```text
StatisticsSnapshot
PhysicalCatalogSnapshot
PlatformCapabilitySnapshot
```

No mediante consultas ocultas al servidor durante cada regla.

---

# 11. Physical catalog

`PhysicalCatalogSnapshot` podrá describir:

```text
tables
indexes
index columns
index ordering
index uniqueness
covering capabilities
partitioning metadata
physical storage capabilities
platform-specific access capabilities
```

---

# 12. Schema semantics ≠ Physical catalog

El Schema/Semantic layer responde:

```text
What does users.email mean?
What is its type?
Is it nullable?
Is it unique?
```

El Physical Catalog puede responder:

```text
Is there an index usable for users.email?
Does it cover requested outputs?
What ordering can it provide?
```

---

# 13. Statistics

Las estadísticas podrán incluir estimaciones como:

```text
row count
selectivity
distinct value count
null fraction
histograms
frequency information
correlation estimates
relation size
index size
```

---

# 14. Statistics ≠ semantic truth

Siempre:

```text
Semantic Fact
≠
Statistical Estimate
```

Una estimación incorrecta puede producir un plan menos eficiente.

Nunca deberá producir semántica incorrecta.

---

# 15. Physical plan artifact

Modelo:

```php
final readonly class PhysicalQueryPlan
{
    public function __construct(
        public PhysicalPlanId $id,
        public PhysicalPlanNode $root,
        public PhysicalPlanGraph $graph,
        public PhysicalPropertyTable $properties,
        public LogicalPhysicalMapping $mapping,
        public PhysicalResourceProfile $resources,
        public PhysicalPlanCost $cost,
        public PhysicalPlanDependencySet $dependencies,
        public PhysicalPlanMetadata $metadata,
        public PhysicalPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 16. Inmutabilidad

Una vez seleccionado:

```text
PhysicalQueryPlan
```

será inmutable.

---

# 17. Planning workspace

Durante búsqueda podrán existir estructuras mutables como:

```text
PhysicalPlanningWorkspace
CandidateSet
Memo
PropertyRequirementStack
CostAccumulator
SearchFrontier
```

pero serán operation-scoped.

---

# 18. IDs

Se distinguirán:

```text
LogicalPlanNodeId
PhysicalPlanNodeId
ExecutionPlanNodeId
```

Siempre:

```text
LogicalPlanNodeId
≠
PhysicalPlanNodeId
≠
ExecutionPlanNodeId
```

---

# 19. PhysicalPlanNode

Contrato base:

```php
interface PhysicalPlanNode
{
    public function id(): PhysicalPlanNodeId;

    public function kind(): PhysicalPlanNodeKind;

    public function inputs(): PhysicalPlanInputList;
}
```

---

# 20. PhysicalPlanNodeKind

V1 podrá incluir:

```text
SEQUENTIAL_SCAN
INDEX_SCAN
INDEX_ONLY_SCAN

FILTER
PROJECT

NESTED_LOOP_JOIN
INDEX_NESTED_LOOP_JOIN
HASH_JOIN
MERGE_JOIN

HASH_AGGREGATE
SORT_AGGREGATE

WINDOW

SORT
TOP_N

HASH_DISTINCT
SORT_DISTINCT

SET_OPERATION

MATERIALIZE
SPOOL

VALUES

INSERT
UPDATE
DELETE
RETURNING

EMPTY_RELATION

EXTENSION
```

---

# 21. Taxonomía física

```text
PhysicalPlanNode
│
├── Access
├── Relational
├── Join
├── Aggregate
├── Window
├── Ordering
├── Distinct
├── Set
├── Materialization
├── Mutation
└── Extension
```

---

# 22. Access Path

Un concepto central será:

```text
AccessPath
```

Representa una forma física válida de acceder a una relación.

---

# 23. AccessPath ≠ LogicalScan

Un:

```text
LogicalScan(users)
```

puede tener:

```text
SequentialAccessPath
IndexAccessPath
IndexOnlyAccessPath
```

---

# 24. AccessPath contract

```php
interface AccessPath
{
    public function relation(): RelationInstanceId;

    public function providedProperties(): ProvidedPhysicalProperties;

    public function estimatedCost(): PhysicalCostEstimate;
}
```

---

# 25. SequentialScan

Representa lectura física secuencial.

```php
final readonly class SequentialScan implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public RelationInstanceId $relation,
        public PhysicalPredicateSet $residualPredicates,
        public PhysicalOutputSet $output,
    ) {}
}
```

---

# 26. SequentialScan no es fallback universal

No toda fuente necesariamente admite sequential scan.

Ejemplo:

```text
table function
remote relation
extension source
virtual source
```

puede tener estrategias diferentes.

---

# 27. IndexScan

```php
final readonly class IndexScan implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public RelationInstanceId $relation,
        public IndexId $index,
        public IndexAccessCondition $accessCondition,
        public PhysicalPredicateSet $residualPredicates,
        public PhysicalOutputSet $output,
    ) {}
}
```

---

# 28. Access predicate ≠ residual predicate

Distinción:

```text
Index Access Condition
=
used to navigate/restrict the index

Residual Predicate
=
still evaluated after candidate rows are obtained
```

---

# 29. Ejemplo

Consulta:

```sql
WHERE email = ?
AND active = true
```

Con índice únicamente sobre `email`:

```text
IndexScan(users.email)
├── access: email = ?
└── residual: active = true
```

---

# 30. IndexOnlyScan

Podrá utilizarse cuando el índice proporcione toda la información requerida y la plataforma permita semántica equivalente.

```text
IndexOnlyScan
```

no será inferido únicamente porque "las columnas están en el índice".

También dependerá de capabilities reales.

---

# 31. Covering property

Se modelará explícitamente:

```text
IndexCoverage
```

---

# 32. Index selection

El planner podrá considerar:

```text
selectivity
ordering
coverage
uniqueness
estimated I/O
estimated CPU
relation size
limit
required outputs
```

---

# 33. Index hints

Los hints del documento 61 podrán influir en preferencia.

No deberán convertir un access path semánticamente inválido en válido.

---

# 34. PhysicalFilter

```php
final readonly class PhysicalFilter implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public PhysicalPlanNodeId $input,
        public PredicateId $predicate,
    ) {}
}
```

---

# 35. Filter fusion

El planner/compiler podrá posteriormente fusionar un filtro con otro operador cuando sea seguro.

---

# 36. PhysicalProject

Representa materialización/proyección física del output.

No implica necesariamente copiar cada columna.

---

# 37. Projection elimination

Si el child ya produce exactamente el output requerido:

```text
PhysicalProject
```

puede no ser necesario.

---

# 38. Join strategies

El sistema distinguirá al menos:

```text
NestedLoopJoin
IndexNestedLoopJoin
HashJoin
MergeJoin
```

---

# 39. NestedLoopJoin

Conceptualmente:

```text
for each row in outer:
    evaluate inner
```

pero el Physical Plan describe la estrategia, no implementa el loop.

---

# 40. NestedLoopJoin model

```php
final readonly class NestedLoopJoin implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public LogicalJoinType $joinType,
        public PhysicalPlanNodeId $outer,
        public PhysicalPlanNodeId $inner,
        public ?PredicateId $condition,
    ) {}
}
```

---

# 41. Outer/inner ≠ logical left/right

Importante:

```text
physical outer input
≠
SQL LEFT operand necessarily
```

La estrategia puede reorganizar roles físicos sólo cuando preserve la semántica lógica.

---

# 42. IndexNestedLoopJoin

Utiliza un access path indexado para el lado dependiente.

Ejemplo:

```text
SequentialScan(users)
       │
       ▼
IndexNestedLoopJoin
       │
       └── IndexLookup(orders.user_id = users.id)
```

---

# 43. Correlated lookup

Este operador será especialmente útil para ciertas dependencias correlacionadas.

---

# 44. HashJoin

```php
final readonly class HashJoin implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public LogicalJoinType $joinType,
        public PhysicalPlanNodeId $build,
        public PhysicalPlanNodeId $probe,
        public HashJoinKeySet $keys,
        public ?PredicateId $residualCondition,
    ) {}
}
```

---

# 45. Build/probe ≠ logical left/right

Siempre:

```text
Build Side
≠
Logical Left Side
```

por definición.

La correspondencia será explícita.

---

# 46. Hash join eligibility

Requiere condiciones físicas compatibles.

No todo predicado de join puede convertirse en hash key.

---

# 47. Residual join predicate

Ejemplo:

```text
a.customer_id = b.customer_id
AND a.created_at < b.created_at
```

podría usar:

```text
Hash key:
a.customer_id = b.customer_id

Residual:
a.created_at < b.created_at
```

---

# 48. MergeJoin

```php
final readonly class MergeJoin implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public LogicalJoinType $joinType,
        public PhysicalPlanNodeId $left,
        public PhysicalPlanNodeId $right,
        public MergeJoinKeySet $keys,
        public ?PredicateId $residualCondition,
    ) {}
}
```

---

# 49. MergeJoin requirements

Puede requerir:

```text
ordered left input
ordered right input
compatible comparison semantics
compatible collation
```

---

# 50. Existing ordering

Si ambos inputs ya proporcionan ordering requerido:

```text
no extra PhysicalSort
```

será necesario.

---

# 51. Property enforcement

Si no lo proporcionan:

```text
PropertyEnforcer
```

puede insertar:

```text
PhysicalSort
```

---

# 52. Logical SEMI JOIN

Podrá implementarse mediante:

```text
NestedLoopSemiJoin
HashSemiJoin
MergeSemiJoin
```

si se introducen operadores especializados.

---

# 53. Logical ANTI JOIN

Igualmente:

```text
NestedLoopAntiJoin
HashAntiJoin
MergeAntiJoin
```

sólo cuando la semántica NULL correspondiente sea compatible con la forma lógica ya demostrada.

---

# 54. Physical strategy cannot reinterpret NOT IN

La etapa física recibe un `LogicalAntiJoin` sólo si etapas anteriores ya demostraron esa forma.

No vuelve a interpretar `NOT IN`.

---

# 55. Join strategy selection

Considerará:

```text
estimated input cardinalities
available indexes
join keys
required ordering
available ordering
memory budget
cost hints
capabilities
correlation
resource constraints
```

---

# 56. Join algorithm ≠ join reordering

El documento 59 optimiza estructura lógica de joins.

Aquí se decide:

```text
how to execute the chosen logical join structure
```

---

# 57. No hidden semantic join reorder

El Physical Planner no deberá alterar la semántica de outer joins, correlations, barriers o volatility bajo el pretexto de seleccionar un algoritmo.

---

# 58. PhysicalAggregate

Logical:

```text
LogicalAggregate
```

podrá implementarse mediante:

```text
HashAggregate
SortAggregate
StreamingAggregate
ExtensionAggregate
```

cuando existan.

---

# 59. HashAggregate

```php
final readonly class HashAggregate implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public PhysicalPlanNodeId $input,
        public GroupingSpecification $grouping,
        public PhysicalAggregateSet $aggregates,
    ) {}
}
```

---

# 60. HashAggregate resource requirements

Puede declarar:

```text
memory sensitivity
spill capability
partitionability
```

sin reservar memoria todavía.

---

# 61. SortAggregate

Puede requerir input ordenado por grouping keys.

```text
PhysicalSort
      │
SortAggregate
```

o reutilizar ordering existente.

---

# 62. Streaming aggregate

Si el input ya satisface el grouping order:

```text
StreamingAggregate
```

puede ser posible.

---

# 63. Global aggregate

No pierde su semántica física:

```text
empty grouping set
```

sigue siendo distinto de ausencia de agregación.

---

# 64. DISTINCT aggregates

El physical strategy debe preservar:

```text
COUNT(DISTINCT x)
```

como propiedad del aggregate.

No confundir con query-level distinct.

---

# 65. PhysicalWindow

Representa estrategia para ejecutar:

```text
LogicalWindow
```

---

# 66. Window requirements

Puede necesitar:

```text
partition ordering
peer ordering
frame support
buffering
rewindability
```

---

# 67. Window ordering ≠ result ordering

Un sort introducido para una window no garantiza automáticamente el output ordering contractual.

Sólo puede reutilizarse si sus propiedades realmente lo satisfacen.

---

# 68. PhysicalSort

```php
final readonly class PhysicalSort implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public PhysicalPlanNodeId $input,
        public PhysicalOrdering $ordering,
        public SortExecutionRequirement $requirement,
    ) {}
}
```

---

# 69. PhysicalSort as enforcer

Puede ser introducido por:

```text
PropertyEnforcer
```

aunque no exista un `LogicalSort`.

Ejemplo:

```text
MergeJoin requires ordering
```

---

# 70. LogicalSort without PhysicalSort

También es posible:

```text
LogicalSort
```

pero ningún `PhysicalSort`, si un access path ya proporciona el ordering.

---

# 71. Required vs Provided properties

Conceptos fundamentales:

```text
RequiredPhysicalProperties
ProvidedPhysicalProperties
```

---

# 72. RequiredPhysicalProperties

Describe lo que un operador necesita de sus inputs.

Ejemplo:

```text
ordering
partitioning
rewindability
materialization
uniqueness
locality
```

---

# 73. ProvidedPhysicalProperties

Describe lo que un physical node produce.

---

# 74. Ejemplo

```text
IndexScan(idx_created_at)
```

puede proporcionar:

```text
ordering = created_at ASC
```

---

# 75. Property satisfaction

Formalmente:

```text
Provided(child)
satisfies
Required(parent)
```

---

# 76. Property satisfaction ≠ equality

Un ordering más fuerte puede satisfacer uno más débil.

Ejemplo:

```text
(a ASC, b ASC)
```

puede satisfacer:

```text
(a ASC)
```

cuando las reglas de ordering lo permitan.

---

# 77. PropertyEnforcer

Contrato conceptual:

```php
interface PhysicalPropertyEnforcer
{
    public function enforce(
        PhysicalPlanCandidate $candidate,
        RequiredPhysicalProperties $required,
        PhysicalPlanningContext $context,
    ): PhysicalPlanCandidate;
}
```

---

# 78. Enforcers posibles

```text
SortEnforcer
MaterializationEnforcer
RewindEnforcer
PartitionEnforcer
ExtensionEnforcer
```

---

# 79. Enforcer ≠ optimizer rewrite

No cambia significado lógico.

Añade infraestructura física para satisfacer un requisito.

---

# 80. TopN

Una combinación:

```text
ORDER BY ...
LIMIT N
```

podrá implementarse mediante:

```text
PhysicalTopN
```

si la plataforma/engine lo soporta.

---

# 81. TopN semantics

Debe preservar exactamente:

```text
ordering
limit
offset semantics
ties behavior when applicable
```

---

# 82. LIMIT without ordering

No habilita arbitrariamente un ordered TopN.

---

# 83. PhysicalDistinct

Podrá implementarse mediante:

```text
HashDistinct
SortDistinct
StreamingDistinct
```

---

# 84. Distinct strategy

Dependerá de:

```text
estimated cardinality
row width
available ordering
memory
platform capabilities
```

---

# 85. DISTINCT semantics remain logical

El algoritmo nunca redefine la igualdad de filas.

---

# 86. Physical set operations

`UNION`, `INTERSECT` y `EXCEPT` podrán utilizar estrategias físicas distintas.

---

# 87. UNION ALL

Puede ser:

```text
PhysicalConcatenate
```

cuando corresponda.

---

# 88. UNION DISTINCT

Puede requerir:

```text
Concatenate
      │
HashDistinct
```

o estrategia equivalente.

---

# 89. INTERSECT / EXCEPT

Podrán utilizar:

```text
hash-based
sort-based
merge-based
platform-native
```

estrategias.

---

# 90. ALL semantics

Las variantes:

```text
INTERSECT ALL
EXCEPT ALL
```

deberán preservar multiplicidades.

No podrán reducirse a set-distinct algorithms incorrectamente.

---

# 91. Materialization

El physical layer podrá introducir:

```text
PhysicalMaterialize
```

por razones de estrategia.

---

# 92. Materialization reasons

Ejemplos:

```text
rewindability
repeated consumption
correlated execution
recursive processing
memory/I/O tradeoff
CTE strategy
pipeline breaking
```

---

# 93. PhysicalMaterialize ≠ LogicalMaterialize

Si existe:

```text
LogicalMaterialize
```

la materialización es requerida.

Si sólo existe:

```text
PhysicalMaterialize
```

es una decisión de implementación.

---

# 94. Spool

Podrá existir:

```text
PhysicalSpool
```

para almacenar/reutilizar resultados intermedios.

---

# 95. Spool types

Futuro:

```text
EagerSpool
LazySpool
RewindSpool
RecursiveSpool
DiskSpool
MemorySpool
```

---

# 96. Physical spool ≠ cache

No deberá confundirse con:

```text
Query Cache
Result Cache
Entity Cache
```

Es estado temporal de ejecución del plan.

---

# 97. PhysicalValues

Implementa:

```text
LogicalValues
```

---

# 98. Parameterized values

Los valores runtime seguirán referenciados mediante parámetros.

No se interpolarán en el Physical Plan.

---

# 99. PhysicalEmptyRelation

Implementa:

```text
LogicalEmptyRelation
```

sin acceder innecesariamente al almacenamiento.

---

# 100. Empty relation output

Debe preservar el output contract.

---

# 101. Mutations

El physical layer podrá seleccionar estrategias para:

```text
INSERT
UPDATE
DELETE
RETURNING
```

---

# 102. PhysicalInsert

Modelo conceptual:

```php
final readonly class PhysicalInsert implements PhysicalPlanNode
{
    public function __construct(
        public PhysicalPlanNodeId $id,
        public RelationInstanceId $target,
        public PhysicalPlanNodeId $source,
        public PhysicalInsertStrategy $strategy,
        public ?PhysicalConflictStrategy $conflict,
    ) {}
}
```

---

# 103. Insert strategies

Podrán incluir:

```text
SingleRowInsert
MultiRowInsert
InsertFromPlan
NativeBulkInsert
ExtensionInsert
```

---

# 104. Bulk ≠ multi-row

La distinción arquitectónica anterior se conserva.

```text
multi-row insert
≠
bulk ingestion subsystem
```

---

# 105. PhysicalUpdate

Puede decidir:

```text
target access path
mutation strategy
RETURNING handling
```

sin cambiar el mutation set lógico.

---

# 106. PhysicalDelete

Igualmente.

---

# 107. Mutation set invariant

Siempre:

```text
RowsMutated(PhysicalPlan)
=
RowsMutated(LogicalPlan)
```

salvo errores de ejecución.

---

# 108. RETURNING invariant

```text
ReturningResult(PhysicalPlan)
=
ReturningResult(LogicalPlan)
```

---

# 109. Locking

El Physical Plan deberá preservar requisitos de locking provenientes del Logical Plan.

---

# 110. Locking strategy

Puede elegir una implementación compatible con:

```text
platform
transaction capabilities
query semantics
```

pero no reducir garantías silenciosamente.

---

# 111. Required physical properties

Modelo:

```php
final readonly class RequiredPhysicalProperties
{
    public function __construct(
        public PhysicalOrderingRequirement $ordering,
        public PhysicalPartitioningRequirement $partitioning,
        public RewindabilityRequirement $rewindability,
        public MaterializationRequirement $materialization,
        public ResourceRequirementSet $resources,
    ) {}
}
```

---

# 112. Provided physical properties

```php
final readonly class ProvidedPhysicalProperties
{
    public function __construct(
        public PhysicalOrdering $ordering,
        public PhysicalPartitioning $partitioning,
        public RewindabilityProperty $rewindability,
        public MaterializationProperty $materialization,
    ) {}
}
```

---

# 113. Ordering property

Deberá incluir:

```text
expressions/symbols
direction
NULL ordering
collation
stability where relevant
```

---

# 114. Collation

Dos orderings textualmente parecidos pueden no ser compatibles si usan collations distintas.

---

# 115. NULL ordering

Igualmente:

```text
NULLS FIRST
NULLS LAST
```

forma parte del contrato cuando sea observable/relevante.

---

# 116. Partitioning

Aunque VoltStack V1 pueda no ejecutar queries distribuidas, el modelo debe permitir:

```text
SINGLE
HASH
RANGE
ROUND_ROBIN
REPLICATED
UNKNOWN
EXTENSION
```

para evolución futura.

---

# 117. Partitioning ≠ database table partitioning

Distinción:

```text
execution partitioning
≠
storage partitioning
```

aunque puedan interactuar.

---

# 118. Rewindability

Algunos operadores necesitan consumir un input múltiples veces.

Se modelará explícitamente.

---

# 119. Streaming

Otros pueden consumir:

```text
single-pass streaming input
```

sin materialización.

---

# 120. Pipeline behavior

Cada operador podrá declarar propiedades como:

```text
PIPELINED
BLOCKING
PARTIALLY_BLOCKING
```

---

# 121. Blocking examples

Generalmente:

```text
Sort
HashAggregate
some Distinct
some HashJoin phases
```

pueden requerir materialización parcial/completa.

---

# 122. Pipeline behavior ≠ execution implementation

El Physical Plan declara el comportamiento esperado.

El Execution Plan define la mecánica concreta.

---

# 123. Resource modeling

Cada candidato podrá declarar requisitos físicos estimados.

---

# 124. PhysicalResourceRequirement

Ejemplo:

```php
final readonly class PhysicalResourceRequirement
{
    public function __construct(
        public MemoryRequirement $memory,
        public CpuRequirement $cpu,
        public IoRequirement $io,
        public SpillRequirement $spill,
        public ParallelismRequirement $parallelism,
    ) {}
}
```

---

# 125. Resource estimate ≠ reservation

El Physical Planner no reserva memoria.

---

# 126. Memory

Podrá estimar:

```text
minimum
preferred
upper estimate
spillable
non-spillable
```

---

# 127. Spill capability

Operadores como:

```text
HashJoin
HashAggregate
Sort
Distinct
```

podrán declarar si soportan spill.

---

# 128. Resource governance

El planner deberá respetar límites proporcionados por:

```text
Database Resource Governance
```

cuando se implemente el bloque correspondiente.

---

# 129. Cost model

Cada candidato tendrá:

```text
PhysicalPlanCost
```

---

# 130. Cost ≠ wall-clock prediction exacta

Será un modelo relativo/estimado.

---

# 131. PhysicalPlanCost

Modelo conceptual:

```php
final readonly class PhysicalPlanCost
{
    public function __construct(
        public CostValue $startup,
        public CostValue $cpu,
        public CostValue $io,
        public CostValue $memory,
        public CostValue $network,
        public CostValue $total,
        public CostConfidence $confidence,
    ) {}
}
```

---

# 132. Cost dimensions

Podrán modelarse separadamente:

```text
startup cost
per-row CPU
I/O
memory pressure
network
spill risk
parallel overhead
```

---

# 133. Total cost

No deberá reducir prematuramente todo a una fórmula rígida universal.

---

# 134. CostModel

```php
interface PhysicalCostModel
{
    public function estimate(
        PhysicalPlanCandidate $candidate,
        PhysicalPlanningContext $context,
    ): PhysicalPlanCost;
}
```

---

# 135. Cost model is platform-aware

Los costos relativos pueden variar entre:

```text
MySQL
MariaDB
PostgreSQL
SQLite
future engines
```

sin introducir vendor conditionals dispersos.

---

# 136. Capability-driven cost model

Preferir:

```text
PhysicalCapability
CostProfile
```

sobre:

```php
if ($driver === 'pgsql') { ... }
```

---

# 137. Query Cost Hints

Documento 61 alimenta este layer.

---

# 138. Hint semantics

Un hint podrá expresar:

```text
prefer
avoid
require
limit search
resource preference
latency preference
throughput preference
```

según su contrato.

---

# 139. Preference hint

Ejemplo:

```text
PREFER INDEX idx_users_email
```

puede modificar ranking.

No garantiza selección si el índice no es válido.

---

# 140. Required hint

Si un hint explícitamente requiere una capacidad y no puede satisfacerse:

```text
planning failure
```

será preferible a ignorarlo silenciosamente.

---

# 141. Hint ≠ semantic permission

Nunca autoriza:

```text
dropping security filter
changing NULL semantics
changing duplicate semantics
ignoring locks
```

---

# 142. Physical alternatives

Para cada logical operator pueden existir múltiples:

```text
PhysicalPlanCandidate
```

---

# 143. Candidate model

```php
final readonly class PhysicalPlanCandidate
{
    public function __construct(
        public PhysicalPlanNode $root,
        public ProvidedPhysicalProperties $properties,
        public PhysicalPlanCost $cost,
        public PhysicalResourceProfile $resources,
        public PhysicalCandidateProvenance $provenance,
    ) {}
}
```

---

# 144. Candidate generation

Ejemplo:

```text
LogicalScan(users)
      │
      ├── SequentialScan
      ├── IndexScan(idx_email)
      ├── IndexScan(idx_status)
      └── IndexOnlyScan(idx_email_cover)
```

---

# 145. Candidate pruning

El planner deberá eliminar candidatos:

```text
semantically invalid
capability-incompatible
property-incompatible
resource-infeasible
dominated
over budget
```

---

# 146. Dominance

Candidato A puede dominar B si:

```text
A provides >= required properties
AND
A cost <= B cost
AND
A resource risk <= B
```

bajo un modelo formal.

---

# 147. Dominance ≠ simplistic total-cost comparison

Un candidato más caro puede proporcionar ordering que evite un sort posterior.

---

# 148. Property-aware planning

Por ello, candidatos deben compararse considerando:

```text
cost + provided properties
```

---

# 149. Interesting properties

El planner podrá conservar candidatos que proporcionen propiedades físicamente útiles aunque no sean inmediatamente requeridas.

Ejemplo:

```text
interesting ordering
```

---

# 150. Search explosion

Esto deberá estar fuertemente limitado.

---

# 151. Physical planning budget

```php
final class PhysicalPlanningBudget
{
    public int $candidateLimit;
    public int $nodeExpansionLimit;
    public int $propertyVariantLimit;
    public int $searchWorkUnits;
}
```

conceptualmente.

---

# 152. Budget is operation-local

Nunca global.

---

# 153. Budget exhaustion

Si se agota:

```text
return best valid candidate discovered
```

cuando exista uno seguro.

---

# 154. No invalid fallback

Nunca:

```text
budget exhausted
→ select semantically invalid cheap plan
```

---

# 155. Search strategies

V1 podrá combinar:

```text
deterministic local enumeration
bounded candidate search
property-aware dynamic programming
targeted join strategy enumeration
```

---

# 156. Future memo planner

Posteriormente podrá utilizarse:

```text
Memo
Groups
GroupExpressions
PhysicalAlternatives
PropertyContexts
```

sin cambiar la API pública del Logical Plan.

---

# 157. Planner strategy ≠ Physical Plan

La técnica interna de búsqueda no forma parte del artifact.

---

# 158. Physical mapping

Debe conservarse:

```text
LogicalPhysicalMapping
```

---

# 159. Mapping model

```php
final readonly class LogicalPhysicalMapping
{
    public function __construct(
        public LogicalToPhysicalNodeMap $forward,
        public PhysicalToLogicalOriginMap $reverse,
    ) {}
}
```

---

# 160. One logical → many physical

Ejemplo:

```text
LogicalSort
```

puede lowering a:

```text
PhysicalSort
```

o a ningún operador adicional si ordering ya existe.

---

# 161. One physical → multiple logical responsibilities

Una estrategia física especializada puede satisfacer varias necesidades.

Ejemplo:

```text
TopN
```

puede satisfacer:

```text
LogicalSort + LogicalLimit
```

---

# 162. Mapping therefore is not 1:1

Siempre:

```text
LogicalPhysicalMapping
≠
simple node-id replacement
```

---

# 163. Provenance

Cada physical node deberá poder explicar:

```text
which logical requirement produced it
which planning rule selected it
which property requirement caused it
which extension owns it
```

---

# 164. PhysicalPlanMetadata

Podrá incluir:

```text
planner version
planning profile
selected strategy tags
hint effects
capability decisions
fallback decisions
extension metadata
diagnostic metadata
```

---

# 165. Metadata ≠ runtime handles

No:

```text
PDOStatement
open file
memory buffer
worker handle
```

---

# 166. Physical dependencies

El plan depende de elementos físicos adicionales.

---

# 167. PhysicalPlanDependencySet

Podrá incluir:

```text
relation physical identity
index identity/version
statistics snapshot version
capability snapshot
cost model version
extension versions
physical catalog version
```

---

# 168. Logical vs physical invalidation

Ejemplo:

```text
DROP INDEX idx_email
```

puede dejar:

```text
LogicalQueryPlan valid
PhysicalQueryPlan invalid
```

---

# 169. Statistics invalidation policy

Cambio de estadísticas puede:

```text
invalidate for replanning
```

sin significar que el plan anterior sea semánticamente incorrecto.

---

# 170. Hard vs soft invalidation

Se distinguirá:

```text
HARD_INVALID
SOFT_STALE
VALID
```

---

# 171. Hard invalidation

Ejemplo:

```text
selected index no longer exists
```

---

# 172. Soft stale

Ejemplo:

```text
statistics changed significantly
```

---

# 173. PhysicalPlanFingerprint

Debe reflejar decisiones físicas.

---

# 174. Fingerprint inputs

```text
LogicalPlanFingerprint
PhysicalOperatorStructure
AccessPathIdentities
PhysicalProperties
RelevantCapabilities
CostModelVersion
PlanningProfile
RelevantHints
ExtensionVersions
PhysicalCatalogVersion
```

---

# 175. Runtime bindings excluded

No incluir:

```text
actual email parameter
actual tenant id
actual search string
```

por defecto.

---

# 176. Parameter-sensitive planning

Si en el futuro se implementa:

```text
parameter-sensitive planning
```

deberá utilizar una clasificación segura:

```text
ParameterSelectivityClass
```

y no necesariamente incorporar el valor sensible al fingerprint.

---

# 177. Physical plan cache

Arquitectura futura:

```text
LogicalPlanFingerprint
      +
PhysicalContextFingerprint
          │
          ▼
PhysicalPlanCache
          │
          ▼
PhysicalQueryPlan
```

---

# 178. PhysicalContextFingerprint

Podrá incorporar:

```text
catalog version
statistics generation
capabilities
cost model version
resource profile
hints
```

---

# 179. Cache reuse

Sólo si el plan sigue:

```text
valid
compatible
sufficiently fresh
```

---

# 180. Replanning

Puede activarse por:

```text
hard dependency invalidation
statistics drift
capability changes
resource profile changes
hint changes
extension version changes
```

---

# 181. Adaptive execution

No pertenece inicialmente al Physical Plan System.

---

# 182. Future adaptive layer

Podría permitir:

```text
runtime cardinality feedback
adaptive join switching
adaptive parallelism
```

pero deberá integrarse sin convertir el Physical Plan en estado mutable global.

---

# 183. Runtime feedback ≠ semantic feedback

Una mala estimación de cardinalidad no cambia el significado de la query.

---

# 184. Platform execution boundary

Hay una decisión importante para VoltStack:

```text
Framework Physical Plan
vs
Database Server Native Planner
```

---

# 185. SQL database reality

MySQL, MariaDB, PostgreSQL y SQLite ya poseen sus propios optimizadores físicos.

VoltStack no deberá asumir que puede reemplazarlos completamente mediante SQL estándar.

---

# 186. Dos niveles físicos

Por ello se distinguirán:

```text
Framework Physical Planning
Database-Native Physical Planning
```

---

# 187. Framework physical plan

VoltStack puede decidir:

```text
query decomposition
subquery strategy
framework-side materialization
batching
statement topology
routing
logical-to-SQL shape
capability emulation
framework execution operators
```

---

# 188. Native DB physical plan

El servidor puede decidir internamente:

```text
actual table scan
actual index scan
actual join algorithm
actual parallel workers
actual buffer strategy
```

dependiendo de la plataforma.

---

# 189. Physical control level

Cada physical decision deberá declarar cuánto control posee VoltStack.

```text
ExecutionControlLevel
```

---

# 190. ExecutionControlLevel

Propuesta:

```text
FRAMEWORK_CONTROLLED
COMPILER_INFLUENCED
DATABASE_DELEGATED
HYBRID
```

---

# 191. Ejemplo PostgreSQL/MySQL

VoltStack puede representar:

```text
DatabaseDelegatedScan(users)
```

cuando el SQL compiler sólo emite una relación y deja al motor decidir el access path real.

---

# 192. No fictional guarantees

VoltStack no deberá afirmar:

```text
IndexScan
```

si el SQL generado no puede garantizar que el servidor realmente lo utilizará.

---

# 193. Important architectural refinement

Por ello:

```text
Physical Strategy
```

debe poder expresar tanto:

```text
framework-enforced strategy
```

como:

```text
database-delegated strategy
```

---

# 194. PhysicalScanStrategy

Podría ser:

```text
DelegatedScan
RequestedIndexAccess
FrameworkMaterializedAccess
ExtensionAccess
```

dependiendo de capabilities.

---

# 195. Requested ≠ guaranteed

Un index hint de plataforma podría representar:

```text
CompilerInfluencedIndexAccess
```

si el servidor aún conserva decisión final.

---

# 196. Native explain feedback

Futuro:

```text
VoltStack Physical Plan
        │
        ▼
Compiled SQL
        │
        ▼
Database EXPLAIN
        │
        ▼
NativeExecutionPlanObservation
```

---

# 197. Native plan observation ≠ VoltStack Physical Plan

Son artifacts distintos.

---

# 198. Benefit

Esto evita construir una arquitectura ficticia que pretenda controlar internamente PostgreSQL/MySQL como si VoltStack fuera su storage engine.

---

# 199. Framework-executed operators

Algunas operaciones sí pueden ser controladas completamente por VoltStack.

Ejemplos futuros:

```text
cross-database federation
distributed merge
application-side materialization
multi-source union
shard merge
client-side batch orchestration
```

---

# 200. Database-delegated subtree

Podrá existir:

```text
PhysicalDatabaseSubplan
```

---

# 201. PhysicalDatabaseSubplan

Representa una región del plan delegable a un servidor.

```text
PhysicalDatabaseSubplan
├── relation
├── predicates
├── joins
├── grouping
└── projection
```

que posteriormente el SQL Compiler convierte en SQL.

---

# 202. Delegation boundary

Esto permitirá decidir:

```text
which operations execute in database
vs
which operations execute in VoltStack runtime
```

---

# 203. V1 strategy

Para bases SQL tradicionales, V1 deberá favorecer:

```text
maximum safe pushdown to database
```

porque los motores nativos suelen poseer optimizadores físicos maduros.

---

# 204. Physical planning therefore has two responsibilities

```text
1. Select framework-level physical topology.
2. Express/delegate database-level physical intent.
```

---

# 205. Native optimizer coexistence

Principio:

```text
VoltStack Planner
cooperates with
Database Native Optimizer
```

no:

```text
VoltStack Planner blindly replaces it
```

---

# 206. Platform capabilities

Cada plataforma declarará qué decisiones pueden:

```text
guarantee
request
influence
observe
not control
```

---

# 207. Capability example

```text
supportsIndexHints()
supportsForcedIndex()
supportsExplainPlan()
supportsMaterializedCteHint()
supportsJoinHints()
supportsStatementTimeout()
```

según modelo futuro.

---

# 208. No vendor conditionals

Incorrecto:

```php
if ($database === 'mysql') {
    // FORCE INDEX
}
```

Correcto:

```php
if ($capabilities->supports(
    PhysicalCapability::FORCED_INDEX
)) {
    ...
}
```

---

# 209. Physical extension system

Extensiones podrán registrar:

```text
physical operators
access paths
property providers
property enforcers
cost estimators
planning rules
delegation strategies
```

---

# 210. Extension physical node

```php
interface PhysicalExtensionNode extends PhysicalPlanNode
{
    public function extensionId(): ExtensionId;

    public function physicalNodeType(): PhysicalNodeTypeId;
}
```

---

# 211. Extension obligations

Debe declarar:

```text
logical operator compatibility
required properties
provided properties
cost model
resources
capabilities
execution control level
compiler/execution integration
fingerprinting
serialization
diagnostics
```

---

# 212. No compiler-only physical extension

Si introduce una estrategia física nueva deberá participar en el planner.

---

# 213. Unknown extension

Debe actuar como:

```text
unsupported physical strategy
```

no como raw SQL.

---

# 214. Registry lifecycle

```text
discover
   ↓
validate
   ↓
resolve dependencies
   ↓
compile physical rule set
   ↓
freeze
```

---

# 215. Persistent runtime

Registries/descriptors compartidos serán inmutables.

---

# 216. Operation-local planning

Serán locales:

```text
candidate sets
memo
cost accumulators
property requirements
planning budget
search frontier
diagnostic trace
```

---

# 217. No current-plan singleton

Prohibido:

```php
PhysicalPlanner::$currentPlan
```

---

# 218. Determinism

Con los mismos:

```text
LogicalQueryPlan
PhysicalCatalogSnapshot
StatisticsSnapshot
Capabilities
CostModel
Hints
Configuration
Extensions
```

el planner deberá producir la misma selección física.

---

# 219. Tie-breaking

Empates se resolverán determinísticamente.

---

# 220. No random planner

No se utilizará randomización no controlada para seleccionar planes en producción.

---

# 221. Physical plan validation

Antes de congelar:

```text
PhysicalPlanValidator
```

deberá verificar el artifact.

---

# 222. Validation checks

```text
all logical requirements satisfied
all required properties satisfied
valid access paths
valid capabilities
valid graph
valid resources
valid dependencies
valid mutation semantics
valid security barriers
valid extensions
```

---

# 223. Required property validation

Cada edge deberá satisfacer:

```text
Provided(child) ⊇ Required(parent,input)
```

o tener enforcer explícito.

---

# 224. Logical output validation

El root físico deberá producir el mismo:

```text
LogicalOutputContract
```

observable.

---

# 225. Physical equivalence

Formalmente:

```text
Semantics(PhysicalQueryPlan)
=
Semantics(LogicalQueryPlan)
```

---

# 226. Cost cannot override equivalence

Siempre:

```text
Correctness
>
Cost
```

---

# 227. Security cannot be costed away

Siempre:

```text
Mandatory Security Semantics
>
Optimization Benefit
```

---

# 228. Locks cannot be costed away

Igualmente.

---

# 229. Volatility

Un plan físico no podrá:

```text
duplicate
eliminate
reorder across forbidden boundaries
```

evaluaciones volátiles de manera observable.

---

# 230. Evaluation multiplicity

Debe conservarse cuando forme parte de la semántica.

---

# 231. Error semantics

Una estrategia física no deberá alterar de manera observable errores requeridos.

Ejemplo:

```text
scalar subquery cardinality violation
```

---

# 232. Short-circuit assumptions

No se asumirá comportamiento PHP-like para expresiones SQL.

---

# 233. Transaction semantics

El plan deberá conservar requisitos de:

```text
locking
isolation-sensitive operations
mutation atomicity
RETURNING
```

---

# 234. Physical Plan and transactions

No abre la transacción.

Sólo expresa requerimientos que Execution/Transaction systems deberán satisfacer.

---

# 235. Physical plan graph

```php
final readonly class PhysicalPlanGraph
{
    public function __construct(
        public PhysicalPlanNodeMap $nodes,
        public PhysicalPlanEdgeSet $edges,
        public PhysicalPlanNodeId $root,
    ) {}
}
```

---

# 236. Edge kinds

Podrán incluir:

```text
DATA
DEPENDENCY
BUILD
PROBE
CORRELATION
MATERIALIZATION
RECURSION
MUTATION
EXTENSION
```

---

# 237. Build/probe edges

Permiten representar explícitamente hash joins sin confundir roles físicos con roles lógicos.

---

# 238. Graph remains structured

No se utilizarán referencias arbitrarias difíciles de validar.

---

# 239. Physical subplans

Podrán existir:

```text
PhysicalSubplanId
```

para:

```text
correlated subqueries
CTEs
recursive members
database-delegated subplans
```

---

# 240. Execution multiplicity

El physical layer podrá declarar:

```text
ONCE
PER_OUTER_ROW
REUSABLE
MATERIALIZED_ONCE
RECURSIVE
DELEGATED
```

cuando sea parte de la estrategia física.

---

# 241. Multiplicity ≠ semantic cardinality

No confundir:

```text
how often a subplan executes
```

con:

```text
how many rows it returns
```

---

# 242. Physical plan phases

Propuesta:

```text
P0 Context Validation
P1 Required Property Derivation
P2 Access Path Enumeration
P3 Physical Operator Enumeration
P4 Property Propagation
P5 Property Enforcement
P6 Cost Estimation
P7 Candidate Pruning
P8 Candidate Selection
P9 Resource Validation
P10 Logical Contract Validation
P11 Dependency Assembly
P12 Fingerprinting
P13 Freeze
```

---

# 243. P0 — Context validation

Valida:

```text
catalog compatibility
statistics version
capabilities
hints
extensions
planning configuration
```

---

# 244. P1 — Property derivation

Parte del:

```text
LogicalOutputContract
```

y propaga requirements hacia abajo.

---

# 245. Top-down requirements

Ejemplo:

```text
LogicalSort(a)
```

produce:

```text
RequiredOrdering(a)
```

---

# 246. Bottom-up properties

Los candidatos reportan:

```text
ProvidedPhysicalProperties
```

---

# 247. Bidirectional planning

El planner combina:

```text
top-down requirements
+
bottom-up capabilities/properties
```

---

# 248. P2 — Access paths

Enumera alternativas para fuentes.

---

# 249. P3 — Physical operators

Enumera implementaciones para logical operators.

---

# 250. P4 — Property propagation

Calcula requirements y provided properties.

---

# 251. P5 — Enforcement

Inserta operadores físicos cuando sea necesario.

---

# 252. P6 — Cost

Evalúa candidatos válidos.

---

# 253. P7 — Pruning

Reduce search space.

---

# 254. P8 — Selection

Selecciona el mejor candidato válido bajo profile/hints/budget.

---

# 255. P9 — Resource validation

Evita seleccionar planes incompatibles con límites conocidos.

---

# 256. P10 — Contract validation

Comprueba equivalencia con Logical Plan.

---

# 257. P11 — Dependencies

Congela dependencias físicas.

---

# 258. P12 — Fingerprint

Genera identidad estructural.

---

# 259. P13 — Freeze

Produce artifact final inmutable.

---

# 260. Planning profiles

Podrán existir:

```text
FAST
BALANCED
THOROUGH
DEBUG
```

para el Physical Planner.

---

# 261. Profile ≠ correctness

`FAST` reduce búsqueda.

No reduce garantías.

---

# 262. FAST

Puede limitar:

```text
candidate count
interesting properties
join strategies
search depth
```

---

# 263. BALANCED

Será candidato a default.

---

# 264. THOROUGH

Permitirá búsqueda más extensa dentro de límites.

---

# 265. DEBUG

Podrá conservar más candidatos y razones de rechazo.

---

# 266. Explain physical

Ejemplo:

```text
PhysicalProject
└── PhysicalTopN [10, order=created_at DESC]
    └── HashJoin [INNER]
        ├── SequentialScan [orders]
        └── IndexScan [users.idx_users_id]
```

---

# 267. Explain properties

Podrá mostrar:

```text
required ordering
provided ordering
estimated rows
cost
memory
access path
control level
```

---

# 268. Explain decisions

Ejemplo:

```text
IndexScan(idx_email)
selected because:
- predicate usable as index equality
- covers projected id/email
- estimated selectivity 0.0008
- provides no required ordering
- estimated cost 2.8

SequentialScan rejected:
- estimated cost 94.2
```

---

# 269. Rejected candidates

DEBUG podrá conservar razones:

```text
capability missing
property mismatch
higher dominated cost
resource limit
hint conflict
planning budget
```

---

# 270. Sensitive data

Explain nunca deberá mostrar runtime values sensibles sin redacción explícita.

---

# 271. Telemetry

Podrá registrar:

```text
planning duration
candidate count
pruned candidates
access paths evaluated
property enforcers inserted
cost evaluations
budget consumption
selected operator kinds
replan reason
```

---

# 272. High cardinality

No usar fingerprints completos como metric labels.

---

# 273. Testing

Se requerirán:

```text
access path tests
join strategy tests
aggregate strategy tests
ordering property tests
property enforcement tests
cost model tests
candidate dominance tests
budget tests
hint tests
resource tests
capability tests
extension tests
persistent worker tests
concurrency tests
```

---

# 274. Equivalence tests

Para cada physical implementation:

```text
PhysicalOperator
```

deberá probarse contra su logical contract.

---

# 275. Join tests

Cubrir:

```text
INNER
LEFT
RIGHT
FULL
SEMI
ANTI
NULL join keys
residual predicates
duplicate rows
```

---

# 276. Aggregate tests

Cubrir:

```text
global aggregate
grouped aggregate
NULL
DISTINCT aggregate
empty input
grouping sets
```

---

# 277. Sort tests

Cubrir:

```text
ASC
DESC
NULLS FIRST/LAST
collations
multi-key
already ordered inputs
```

---

# 278. Distinct tests

Especialmente:

```text
NULL rows
duplicate multiplicities
collations
compound rows
```

---

# 279. Set tests

Cubrir:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
```

---

# 280. Cost model tests

No deben exigir estimaciones perfectas.

Deben verificar:

```text
consistency
monotonicity where expected
determinism
reasonable preference behavior
```

---

# 281. Budget tests

Una query adversarial no deberá provocar búsqueda combinatoria ilimitada.

---

# 282. Persistent worker tests

Verificar:

```text
no candidate leakage
no statistics leakage
no hint leakage
no tenant leakage
no previous-plan state
```

---

# 283. Concurrency tests

Múltiples planners deberán operar simultáneamente sobre registries frozen.

---

# 284. Error model

Posibles excepciones:

```text
PhysicalPlanningException
NoValidPhysicalPlanException
PhysicalPropertyUnsatisfiedException
PhysicalCapabilityException
PhysicalAccessPathException
PhysicalCostModelException
PhysicalResourceException
PhysicalHintException
PhysicalPlanInvariantException
PhysicalPlanExtensionException
PhysicalPlanCompatibilityException
```

---

# 285. No valid plan

Si ninguna estrategia satisface el Logical Plan:

```text
fail explicitly
```

---

# 286. No SQL fallback

Nunca:

```text
no physical strategy
→ raw SQL
```

---

# 287. Database delegation fallback

Delegar al optimizador nativo sí puede ser una estrategia física válida **si está modelada explícitamente**.

Ejemplo:

```text
DatabaseDelegatedSubplan
```

No es raw fallback.

---

# 288. Delegation contract

Debe declarar:

```text
which logical operators are delegated
required database capabilities
compiler contract
observable semantics
control level
```

---

# 289. Security boundary

Una región delegada debe contener explícitamente todos los filtros/policies que deban ejecutarse en la base.

---

# 290. No security post-filter assumption

No se deberá enviar información prohibida a VoltStack para filtrarla después si la política exige enforcement en database boundary.

---

# 291. Tenant isolation

Igualmente, el planner no podrá mover tenant predicates fuera de una frontera que rompa aislamiento.

---

# 292. ORM boundary

Physical Planner no conoce:

```text
Entity
Model
UnitOfWork
IdentityMap
Repository
EntityManager
```

---

# 293. ORM pipeline

```text
ORM Query
   │
   ▼
Query Model
   │
   ▼
Semantic Engine
   │
   ▼
Optimizer
   │
   ▼
Logical Plan
   │
   ▼
Physical Plan
```

El mismo motor se reutiliza.

---

# 294. Active Record boundary

Active Record tampoco introduce otro Physical Planner.

---

# 295. Runtime boundary

FrankenPHP, RoadRunner y OpenSwoole no cambian la semántica del Physical Plan.

---

# 296. Runtime may influence resource profile

Sí pueden afectar:

```text
memory limits
concurrency limits
persistent worker configuration
```

mediante contexto explícito.

---

# 297. Runtime-specific conditionals

No deberán dispersarse dentro de physical nodes.

---

# 298. Proposed namespace

```text
VoltStack/Quantum/Database/Query/Planner/Physical
├── Contract
│   ├── PhysicalPlanNode.php
│   ├── PhysicalPlanner.php
│   └── PhysicalCostModel.php
│
├── Plan
│   ├── PhysicalQueryPlan.php
│   ├── PhysicalPlanId.php
│   ├── PhysicalPlanGraph.php
│   ├── PhysicalPlanNodeId.php
│   └── PhysicalPlanFingerprint.php
│
├── Access
│   ├── AccessPath.php
│   ├── SequentialScan.php
│   ├── IndexScan.php
│   ├── IndexOnlyScan.php
│   └── AccessPathEnumerator.php
│
├── Operator
│   ├── PhysicalFilter.php
│   ├── PhysicalProject.php
│   ├── PhysicalValues.php
│   └── PhysicalEmptyRelation.php
│
├── Join
│   ├── NestedLoopJoin.php
│   ├── IndexNestedLoopJoin.php
│   ├── HashJoin.php
│   ├── MergeJoin.php
│   └── JoinStrategyEnumerator.php
│
├── Aggregate
│   ├── HashAggregate.php
│   ├── SortAggregate.php
│   └── StreamingAggregate.php
│
├── Window
│   └── PhysicalWindow.php
│
├── Ordering
│   ├── PhysicalSort.php
│   ├── PhysicalTopN.php
│   └── PhysicalOrdering.php
│
├── Distinct
│   ├── HashDistinct.php
│   └── SortDistinct.php
│
├── SetOperation
│   └── PhysicalSetOperation.php
│
├── Materialization
│   ├── PhysicalMaterialize.php
│   └── PhysicalSpool.php
│
├── Mutation
│   ├── PhysicalInsert.php
│   ├── PhysicalUpdate.php
│   ├── PhysicalDelete.php
│   └── PhysicalReturning.php
│
├── Property
│   ├── RequiredPhysicalProperties.php
│   ├── ProvidedPhysicalProperties.php
│   ├── PhysicalPropertyEnforcer.php
│   └── PhysicalPropertyTable.php
│
├── Candidate
│   ├── PhysicalPlanCandidate.php
│   ├── CandidateSet.php
│   ├── CandidatePruner.php
│   └── CandidateSelector.php
│
├── Cost
│   ├── PhysicalPlanCost.php
│   ├── PhysicalCostEstimate.php
│   └── CostModelSnapshot.php
│
├── Resource
│   ├── PhysicalResourceProfile.php
│   └── PhysicalResourceRequirement.php
│
├── Delegation
│   ├── ExecutionControlLevel.php
│   ├── DatabaseDelegatedSubplan.php
│   └── DelegationStrategy.php
│
├── Mapping
│   └── LogicalPhysicalMapping.php
│
├── Dependency
│   └── PhysicalPlanDependencySet.php
│
├── Extension
│   ├── PhysicalExtensionNode.php
│   └── PhysicalExtensionRegistry.php
│
├── Diagnostic
│   ├── PhysicalPlanPrinter.php
│   └── PhysicalPlanningTrace.php
│
├── Validation
│   └── PhysicalPlanValidator.php
│
└── Exception
```

---

# 299. Dependencias permitidas

```text
Physical Planner
      │
      ├── Logical Plan
      ├── Platform Capabilities
      ├── Physical Catalog Snapshot
      ├── Statistics Snapshot
      ├── Cost Hints
      ├── Cost Model
      └── Extension Contracts
```

---

# 300. Dependencias prohibidas

```text
Physical Planner
      ✗ ORM
      ✗ EntityManager
      ✗ UnitOfWork
      ✗ HTTP
      ✗ Controller
      ✗ Livewire-like runtime
      ✗ direct PDO execution
```

---

# 301. Invariantes

## DB-PPLAN-001

PhysicalQueryPlan será distinto de LogicalQueryPlan.

## DB-PPLAN-002

PhysicalQueryPlan será distinto de ExecutionPlan.

## DB-PPLAN-003

PhysicalQueryPlan será distinto de SQL.

## DB-PPLAN-004

Physical Planner no ejecutará queries.

## DB-PPLAN-005

Physical Planner no abrirá conexiones.

## DB-PPLAN-006

Physical Planner no hidratará entidades.

## DB-PPLAN-007

Physical Planner consumirá LogicalQueryPlan como autoridad lógica inmediata.

## DB-PPLAN-008

Physical planning preservará la semántica del Logical Plan.

## DB-PPLAN-009

Correctness tendrá prioridad sobre cost.

## DB-PPLAN-010

Security tendrá prioridad sobre cost.

## DB-PPLAN-011

Locking semantics tendrán prioridad sobre cost.

## DB-PPLAN-012

LogicalPlanNodeId será distinto de PhysicalPlanNodeId.

## DB-PPLAN-013

PhysicalPlanNodeId será distinto de ExecutionPlanNodeId.

## DB-PPLAN-014

Physical IDs serán operation-scoped.

## DB-PPLAN-015

Physical Plan final será inmutable.

## DB-PPLAN-016

Planning workspace será operation-scoped.

## DB-PPLAN-017

No existirán static current plans.

## DB-PPLAN-018

Physical catalog será explícito.

## DB-PPLAN-019

Statistics serán explícitas.

## DB-PPLAN-020

Statistics no serán semantic truth.

## DB-PPLAN-021

AccessPath será distinto de LogicalScan.

## DB-PPLAN-022

SequentialScan será una estrategia física.

## DB-PPLAN-023

IndexScan será una estrategia física.

## DB-PPLAN-024

IndexOnlyScan requerirá capabilities válidas.

## DB-PPLAN-025

Index access condition será distinta de residual predicate.

## DB-PPLAN-026

Index hint no hará válido un access path inválido.

## DB-PPLAN-027

PhysicalFilter preservará predicate semantics.

## DB-PPLAN-028

PhysicalProject podrá eliminarse sólo si el output contract sigue satisfecho.

## DB-PPLAN-029

LogicalJoin no determinará automáticamente HashJoin.

## DB-PPLAN-030

LogicalJoin no determinará automáticamente MergeJoin.

## DB-PPLAN-031

LogicalJoin no determinará automáticamente NestedLoopJoin.

## DB-PPLAN-032

Physical outer/build/probe roles serán distintos de logical left/right identities.

## DB-PPLAN-033

Hash join keys serán explícitos.

## DB-PPLAN-034

Residual join predicates serán preservados.

## DB-PPLAN-035

MergeJoin requerirá properties compatibles.

## DB-PPLAN-036

Physical Planner no reinterpretará NOT IN.

## DB-PPLAN-037

Logical SEMI semantics serán preservadas.

## DB-PPLAN-038

Logical ANTI semantics serán preservadas.

## DB-PPLAN-039

Physical join selection será distinta de logical join reordering.

## DB-PPLAN-040

Outer join semantics no serán alteradas por strategy selection.

## DB-PPLAN-041

LogicalAggregate tendrá múltiples posibles implementations.

## DB-PPLAN-042

HashAggregate será distinto de SortAggregate.

## DB-PPLAN-043

Global aggregate preservará empty grouping semantics.

## DB-PPLAN-044

Aggregate DISTINCT será distinto de query DISTINCT.

## DB-PPLAN-045

PhysicalWindow preservará window frame semantics.

## DB-PPLAN-046

Window ordering será distinto de result ordering.

## DB-PPLAN-047

PhysicalSort podrá ser un property enforcer.

## DB-PPLAN-048

LogicalSort no requerirá siempre PhysicalSort.

## DB-PPLAN-049

PhysicalSort podrá existir sin LogicalSort.

## DB-PPLAN-050

RequiredPhysicalProperties serán distintas de ProvidedPhysicalProperties.

## DB-PPLAN-051

Property satisfaction no requerirá igualdad estructural exacta.

## DB-PPLAN-052

PropertyEnforcer no cambiará logical semantics.

## DB-PPLAN-053

TopN sólo será seleccionado si preserva sort/limit semantics.

## DB-PPLAN-054

LIMIT sin ORDER BY no inventará ordering.

## DB-PPLAN-055

PhysicalDistinct preservará SQL row distinctness.

## DB-PPLAN-056

HashDistinct y SortDistinct serán estrategias distintas.

## DB-PPLAN-057

Set ALL preservará multiplicidades.

## DB-PPLAN-058

UNION ALL no será convertido a UNION DISTINCT.

## DB-PPLAN-059

INTERSECT ALL preservará bag semantics.

## DB-PPLAN-060

EXCEPT ALL preservará bag semantics.

## DB-PPLAN-061

PhysicalMaterialize será distinto de LogicalMaterialize.

## DB-PPLAN-062

PhysicalSpool será distinto de Query Cache.

## DB-PPLAN-063

PhysicalSpool será execution-temporary.

## DB-PPLAN-064

PhysicalValues no interpolará runtime bindings.

## DB-PPLAN-065

PhysicalEmptyRelation preservará output contract.

## DB-PPLAN-066

Physical mutation preservará mutation set.

## DB-PPLAN-067

Physical mutation preservará RETURNING.

## DB-PPLAN-068

Physical mutation no sustituirá ORM persistence.

## DB-PPLAN-069

Physical locking strategy no reducirá garantías silenciosamente.

## DB-PPLAN-070

Resource estimate será distinto de resource reservation.

## DB-PPLAN-071

Memory requirement podrá declarar spillability.

## DB-PPLAN-072

Cost será una estimación.

## DB-PPLAN-073

Cost no será semantic fact.

## DB-PPLAN-074

CostModel será explícito.

## DB-PPLAN-075

Cost model podrá ser platform-aware mediante capabilities/profiles.

## DB-PPLAN-076

Vendor conditionals dispersos estarán prohibidos.

## DB-PPLAN-077

Hints influirán preferencias sin cambiar semántica.

## DB-PPLAN-078

Required hints imposibles fallarán explícitamente.

## DB-PPLAN-079

Candidates deberán ser semánticamente válidos antes de cost evaluation final.

## DB-PPLAN-080

Candidate pruning será property-aware.

## DB-PPLAN-081

Candidate dominance no será únicamente comparación de total cost.

## DB-PPLAN-082

Interesting properties estarán bounded.

## DB-PPLAN-083

Planning search tendrá budget.

## DB-PPLAN-084

Budget será operation-scoped.

## DB-PPLAN-085

Budget exhaustion nunca seleccionará plan inválido.

## DB-PPLAN-086

Planner podrá retornar mejor candidato válido al agotar budget.

## DB-PPLAN-087

LogicalPhysicalMapping será explícito.

## DB-PPLAN-088

LogicalPhysicalMapping no será necesariamente 1:1.

## DB-PPLAN-089

TopN podrá mapear múltiples logical requirements.

## DB-PPLAN-090

Physical provenance será preservable.

## DB-PPLAN-091

Physical metadata no contendrá runtime handles.

## DB-PPLAN-092

Physical dependencies serán explícitas.

## DB-PPLAN-093

Index removal podrá hard-invalidar Physical Plan.

## DB-PPLAN-094

Statistics drift podrá soft-invalidar Physical Plan.

## DB-PPLAN-095

Hard invalidation será distinta de stale planning information.

## DB-PPLAN-096

PhysicalPlanFingerprint incluirá decisiones físicas relevantes.

## DB-PPLAN-097

Runtime binding values no estarán en fingerprint por defecto.

## DB-PPLAN-098

Parameter-sensitive planning será explícito.

## DB-PPLAN-099

Physical plan cache validará dependencies.

## DB-PPLAN-100

Adaptive execution será distinto de Physical Plan V1.

## DB-PPLAN-101

VoltStack distinguirá framework planning de database-native planning.

## DB-PPLAN-102

VoltStack no afirmará control físico que la plataforma no garantice.

## DB-PPLAN-103

ExecutionControlLevel será explícito.

## DB-PPLAN-104

FRAMEWORK_CONTROLLED será distinto de DATABASE_DELEGATED.

## DB-PPLAN-105

COMPILER_INFLUENCED será distinto de guaranteed execution.

## DB-PPLAN-106

DatabaseDelegatedSubplan será una estrategia válida explícita.

## DB-PPLAN-107

Database delegation no será raw fallback.

## DB-PPLAN-108

Delegated subplan tendrá compiler contract.

## DB-PPLAN-109

V1 favorecerá safe database pushdown para motores SQL maduros.

## DB-PPLAN-110

VoltStack cooperará con el native optimizer.

## DB-PPLAN-111

Platform capabilities declararán control real disponible.

## DB-PPLAN-112

Native EXPLAIN será distinto de VoltStack Physical Plan.

## DB-PPLAN-113

Framework-executed operators podrán coexistir con database-delegated subplans.

## DB-PPLAN-114

Physical extensions tendrán contracts completos.

## DB-PPLAN-115

Unknown physical extension no tendrá raw fallback.

## DB-PPLAN-116

Physical extension registry será frozen después de bootstrap.

## DB-PPLAN-117

Extension conflicts no serán last-one-wins.

## DB-PPLAN-118

Planning será determinista con inputs equivalentes.

## DB-PPLAN-119

Tie-breaking será determinista.

## DB-PPLAN-120

PhysicalPlanValidator verificará logical contract.

## DB-PPLAN-121

Every required property deberá satisfacerse o tener enforcer.

## DB-PPLAN-122

Physical root preservará LogicalOutputContract.

## DB-PPLAN-123

Cost nunca autorizará remover security filters.

## DB-PPLAN-124

Cost nunca autorizará remover locks requeridos.

## DB-PPLAN-125

Volatile evaluation multiplicity será preservada cuando sea observable.

## DB-PPLAN-126

Scalar subquery error semantics serán preservadas.

## DB-PPLAN-127

Physical Plan no abrirá transacciones.

## DB-PPLAN-128

Transaction requirements serán expresados para etapas posteriores.

## DB-PPLAN-129

Physical Plan será conceptualmente un graph.

## DB-PPLAN-130

Build/probe edges podrán ser distintos de data edges genéricos.

## DB-PPLAN-131

Physical subplan execution multiplicity será distinta de row cardinality.

## DB-PPLAN-132

Planning combinará top-down requirements y bottom-up properties.

## DB-PPLAN-133

Access paths serán enumerados antes de selección.

## DB-PPLAN-134

Property enforcement ocurrirá explícitamente.

## DB-PPLAN-135

Resource validation ocurrirá antes de freeze.

## DB-PPLAN-136

Planning profile no reducirá correctness.

## DB-PPLAN-137

Explain Physical será distinto de SQL Explain nativo.

## DB-PPLAN-138

Explain no expondrá sensitive runtime values.

## DB-PPLAN-139

Telemetry evitará high-cardinality fingerprints como labels.

## DB-PPLAN-140

Physical implementations tendrán equivalence tests.

## DB-PPLAN-141

Cost models tendrán determinism tests.

## DB-PPLAN-142

Planning budgets tendrán adversarial tests.

## DB-PPLAN-143

Persistent workers no compartirán mutable candidate state.

## DB-PPLAN-144

Concurrent planning será aislado.

## DB-PPLAN-145

NoValidPhysicalPlan fallará explícitamente.

## DB-PPLAN-146

No habrá raw SQL fallback por failure de planning.

## DB-PPLAN-147

Security boundaries serán preservadas durante delegation.

## DB-PPLAN-148

Tenant isolation boundaries serán preservadas.

## DB-PPLAN-149

Physical Planner no dependerá de ORM.

## DB-PPLAN-150

Active Record no tendrá physical planner separado.

## DB-PPLAN-151

Runtime server no alterará physical semantics.

## DB-PPLAN-152

Runtime resource constraints llegarán mediante contexto explícito.

## DB-PPLAN-153

Physical Plan final será serializable sólo mediante contratos versionados.

## DB-PPLAN-154

Physical operator compatibility será capability-driven.

## DB-PPLAN-155

No physical strategy podrá producir un observable result distinto del Logical Plan.

## DB-PPLAN-156

`Semantics(PhysicalQueryPlan) = Semantics(LogicalQueryPlan)` será el invariante maestro.

---

# 302. Ejemplo completo

Consulta conceptual:

```sql
SELECT
    u.id,
    SUM(o.total) AS spent
FROM users u
JOIN orders o
    ON o.user_id = u.id
WHERE
    u.active = true
GROUP BY
    u.id
ORDER BY
    spent DESC
LIMIT 10;
```

Logical Plan:

```text
LogicalLimit [10]
└── LogicalSort [spent DESC]
    └── LogicalProject [u.id, spent]
        └── LogicalAggregate
            group = [u.id]
            SUM(o.total)
            └── LogicalJoin [INNER]
                ├── LogicalFilter [u.active = true]
                │   └── LogicalScan [users]
                └── LogicalScan [orders]
```

Posible Physical Plan controlado:

```text
PhysicalTopN [10, spent DESC]
└── PhysicalProject
    └── HashAggregate [group=u.id]
        └── HashJoin
            build:
            │   IndexScan [users.idx_active]
            │
            probe:
                SequentialScan [orders]
```

Pero en un servidor SQL tradicional VoltStack podría determinar que la mejor frontera es:

```text
DatabaseDelegatedSubplan
│
├── Scan(users)
├── Scan(orders)
├── Join
├── Filter
├── Aggregate
├── Order
└── Limit
```

y dejar al optimizador nativo decidir:

```text
actual indexes
actual join algorithm
actual aggregation algorithm
actual sort strategy
```

---

# 303. Principio de realismo arquitectónico

Este punto será especialmente importante para VoltStack:

```text
A framework query planner must distinguish
what it can model
from
what it can actually control.
```

Por tanto:

```text
Physical Intent
≠
Guaranteed Native Database Execution
```

salvo cuando la plataforma exponga una garantía concreta.

---

# 304. Arquitectura consolidada

```text
                     LogicalQueryPlan
                            │
                            ▼
                  PhysicalPlanner
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
   Access Paths      Operator Strategies   Properties
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                    Candidate Generator
                            │
                            ▼
                     Candidate Set
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
          Cost          Resources       Capabilities
            │               │               │
            └───────────────┼───────────────┘
                            ▼
                     Candidate Pruner
                            │
                            ▼
                     Candidate Selector
                            │
                            ▼
                    Property Enforcers
                            │
                            ▼
                    Contract Validator
                            │
                            ▼
                  PhysicalQueryPlan
                            │
          ┌─────────────────┴─────────────────┐
          ▼                                   ▼
Framework-Controlled                   DB-Delegated
Operators                              Subplans
          │                                   │
          └─────────────────┬─────────────────┘
                            ▼
                    Execution Plan
```

---

# 305. Relación entre propiedades

```text
Logical Requirement
        │
        ▼
RequiredPhysicalProperties
        │
        ▼
Candidate Enumeration
        │
        ▼
ProvidedPhysicalProperties
        │
        ├── satisfies ────────────────┐
        │                             │
        └── does not satisfy          │
                 │                    │
                 ▼                    │
          PropertyEnforcer            │
                 │                    │
                 └────────────────────┘
                         │
                         ▼
                  Valid Candidate
```

---

# 306. Fórmula de selección

Una forma conceptual:

```text
BestPhysicalPlan
=
argmin Cost(P)
```

sujeto a:

```text
Semantics(P) = Semantics(L)
∧
RequiredProperties(L) satisfied
∧
Capabilities(P) available
∧
Security(P) preserved
∧
TransactionRequirements(P) preserved
∧
Resources(P) acceptable
∧
Hints(P) satisfied according to their strength
```

donde:

```text
L = LogicalQueryPlan
P = PhysicalPlanCandidate
```

---

# 307. La fórmula completa

```text
ValidPhysicalCandidate(P)
=
SemanticEquivalence(P)
∧
PropertySatisfaction(P)
∧
CapabilityCompatibility(P)
∧
BarrierPreservation(P)
∧
SecurityPreservation(P)
∧
MutationPreservation(P)
∧
ResourceFeasibility(P)
```

Sólo después:

```text
PreferredCandidate
=
CostAndPolicySelection(
    ValidPhysicalCandidates
)
```

---

# 308. Principio maestro

```text
Validity before Cost.
```

Nunca:

```text
Cheapest first, correctness later.
```

---

# 309. Integración con el siguiente nivel

El Physical Plan será consumido por:

```text
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

La transición será:

```text
PhysicalQueryPlan
        │
        ▼
Execution Plan Construction
        │
        ▼
ExecutionPlan
```

---

# 310. Qué todavía falta

El Physical Plan sabe:

```text
which physical operators
which access/delegation strategies
which physical properties
which resources are expected
which dependencies exist
which strategy was selected
```

pero aún no contiene necesariamente:

```text
runtime operator instances
parameter slots
statement handles
resource acquisition sequence
cancellation hooks
cleanup graph
execution state machines
cursor ownership
runtime scheduling
failure propagation
```

Eso pertenece al Execution Plan.

---

# 311. Pipeline final actualizado

```text
Developer Intent
      │
      ▼
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
Logical Plan System
      │
      ▼
LogicalQueryPlan
      │
      ▼
Physical Plan System
      │
      ▼
PhysicalQueryPlan
      │
      ▼
Execution Plan System
      │
      ▼
ExecutionPlan
      │
      ▼
SQL Compiler / Runtime Executor
      │
      ▼
Database
```

---

# 312. Distinción definitiva

```text
SemanticQueryArtifact
=
What the query means.

OptimizedQueryArtifact
=
Which equivalent logical form is preferred.

LogicalQueryPlan
=
Which relational operations are required.

PhysicalQueryPlan
=
Which physical strategies should implement them.

ExecutionPlan
=
How those strategies become executable work.

CompiledQuery
=
How database-delegated work is represented to the target platform.

Executor
=
How the work actually runs.
```

---

# 313. Fórmula arquitectónica final

```text
Physical Query Planning
=
Logical Requirements
+
Access Path Enumeration
+
Physical Operator Selection
+
Required/Provided Property Reasoning
+
Property Enforcement
+
Capability Resolution
+
Resource Modeling
+
Cost Estimation
+
Bounded Candidate Search
+
Deterministic Selection
+
Execution-Control Awareness
```

sujeto siempre a:

```text
Semantic Preservation
+
Security Preservation
+
Transaction Preservation
+
Barrier Preservation
+
Observable Output Preservation
```

---

# 314. Estado del bloque

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
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md             ← actual
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 315. Siguiente documento

```text
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

El siguiente documento deberá formalizar la transición:

```text
PhysicalQueryPlan
       │
       ▼
Execution Plan Construction
       │
       ▼
ExecutionPlan
```

y definir en detalle:

```text
ExecutionPlan
ExecutionPlanNode
ExecutionPlanGraph
ExecutionStage
ExecutionPipeline
ExecutionDependency
ExecutionOperator
ExecutionState
RuntimeParameterSlot
StatementExecutionUnit
DatabaseExecutionUnit
FrameworkExecutionUnit
ResourceAcquisitionPlan
ResourceReleasePlan
CursorOwnership
BufferOwnership
MaterializationRuntime
ExecutionScheduling
ExecutionOrdering
ExecutionCancellation
ExecutionTimeout
ExecutionFailurePropagation
ExecutionCleanup
ExecutionInstrumentation
ExecutionControlBoundary
DatabaseDelegatedExecution
FrameworkControlledExecution
ExecutionPlanValidator
ExecutionPlanFingerprint
```

manteniendo como invariante:

```text
PhysicalQueryPlan
=
selected physical strategy

ExecutionPlan
=
executable orchestration of that strategy
```