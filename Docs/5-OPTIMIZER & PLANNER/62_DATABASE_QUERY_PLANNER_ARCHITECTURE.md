# 62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md

# VoltStack Quantum Database
## Query Planner Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 62 — Query Planner Architecture  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Planner  
**Versión:** 1.0

---

# 1. Propósito

`Query Planner Architecture` define la arquitectura responsable de transformar una consulta semánticamente válida y lógicamente optimizada en una representación explícita de **cómo puede ser ejecutada**.

El Planner constituye la frontera entre:

```text
logical query reasoning
```

y:

```text
execution strategy
```

La regla fundamental será:

```text
Optimizer
    =
Which semantically equivalent logical form is preferable?

Planner
    =
How can that logical query be executed?
```

Por tanto:

```text
Optimizer ≠ Planner
```

y también:

```text
Planner ≠ Compiler
Planner ≠ Executor
```

---

# 2. Posición dentro del Query Engine

La arquitectura general será:

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
Query Planner
     │
     ├── Logical Planning
     ├── Physical Alternative Generation
     ├── Physical Property Analysis
     ├── Cost Evaluation
     ├── Resource Planning
     └── Plan Selection
     │
     ▼
PhysicalQueryPlan
     │
     ▼
Execution Plan Finalization
     │
     ▼
ExecutionPlan
     │
     ▼
SQL Compiler / Executor
```

Los documentos 63–65 formalizarán los artifacts concretos.

---

# 3. Fronteras fundamentales

VoltStack distinguirá estrictamente:

```text
Query Model
    ≠
Semantic Query
    ≠
Optimized Query
    ≠
Logical Query Plan
    ≠
Physical Query Plan
    ≠
Execution Plan
    ≠
Compiled SQL
    ≠
Executed Statement
```

---

# 4. Query Model

Describe:

```text
what the developer requested structurally
```

Ejemplo:

```text
SELECT
FROM
JOIN
WHERE
GROUP
ORDER
```

Todavía no representa una estrategia de ejecución.

---

# 5. Semantic Query

Describe:

```text
what the query means
```

Incluye:

```text
resolved symbols
types
relations
correlations
constraints
dependencies
capabilities
lineage
```

---

# 6. Optimized Query

Representa una forma lógicamente equivalente derivada por el Optimizer.

Ejemplo:

```text
Original:

A LEFT JOIN B
WHERE B.id IS NOT NULL
```

puede convertirse, si existe prueba suficiente, en:

```text
A INNER JOIN B
```

El Planner recibe la forma ya optimizada.

---

# 7. Logical Query Plan

Describe una estrategia lógica de operadores.

Ejemplo:

```text
Project
   │
 Filter
   │
 Join
 ┌─┴─┐
 A   B
```

Todavía no dice:

```text
HashJoin
MergeJoin
NestedLoop
IndexScan
SequentialScan
```

---

# 8. Physical Query Plan

Selecciona implementaciones físicas.

Ejemplo:

```text
Projection
    │
HashJoin
 ┌──┴────────┐
SeqScan(A) IndexScan(B)
```

---

# 9. Execution Plan

Es la forma final preparada para ejecución/compilación.

Puede incluir:

```text
physical operators
resolved platform strategies
parameters
temporary resources
materialization
ordering enforcement
resource requirements
execution dependencies
compiler requirements
```

---

# 10. SQL Compiler

El Compiler transforma el plan correspondiente en representación ejecutable para una plataforma.

No decide nuevamente el plan.

---

# 11. Executor

El Executor ejecuta el artifact compilado.

No realiza planificación lógica.

---

# 12. Principio arquitectónico central

```text
Semantic Analysis
    determines meaning.

Optimizer
    transforms meaning-preserving logical forms.

Planner
    determines execution strategy.

Compiler
    renders target representation.

Executor
    performs execution.
```

---

# 13. Entrada del Planner

La entrada principal será:

```php
OptimizedQueryArtifact
```

acompañada por un:

```php
PlanningContext
```

Conceptualmente:

```text
Plan(
    OptimizedQueryArtifact,
    PlanningContext
)
→
PlanningArtifact
```

---

# 14. OptimizedQueryArtifact

Conceptualmente contendrá:

```text
OptimizedQueryArtifact
├── SemanticQueryArtifact reference
├── OptimizedQueryRepresentation
├── OutputContract
├── ConstraintGraph
├── Dependencies
├── CapabilityRequirements
├── OptimizationEvidence
├── SemanticMappings
└── OptimizationFingerprint
```

---

# 15. Semantic truth remains authoritative

El Planner no reinterpretará la consulta desde cero.

Siempre:

```text
SemanticQueryArtifact
=
authoritative semantic meaning
```

mientras:

```text
OptimizedQueryArtifact
=
derived equivalent representation
```

---

# 16. PlanningContext

`PlanningContext` será operation-scoped.

Ejemplo:

```php
final readonly class PlanningContext
{
    public function __construct(
        public PlatformCapabilitySnapshot $capabilities,
        public QueryCostHintSet $costHints,
        public ?StatisticsSnapshot $statistics,
        public PlanningConfiguration $configuration,
        public PlanningBudget $budget,
        public PhysicalPropertyEnvironment $properties,
    ) {}
}
```

---

# 17. PlanningContext ≠ QueryContext

```text
QueryContext
    =
general query processing context

OptimizationContext
    =
logical transformation context

PlanningContext
    =
execution strategy planning context
```

---

# 18. PlanningContext ≠ ExecutionContext

El Planner no tendrá:

```text
live statement
current cursor
execution result
PDOStatement
active result stream
```

---

# 19. PlanningContext ≠ Connection

El Planner no deberá requerir:

```php
PDO
Connection
Statement
ResultCursor
```

para producir un plan.

---

# 20. Explicit snapshots

Toda información externa necesaria deberá llegar mediante artifacts explícitos:

```text
CapabilitySnapshot
StatisticsSnapshot
CostHintSet
Schema information
Resource profile
```

---

# 21. No hidden I/O

Durante planificación pura no deberá ocurrir:

```text
database query
network request
filesystem lookup
statistics refresh
schema introspection
```

---

# 22. QueryPlanner

Contrato principal:

```php
interface QueryPlanner
{
    public function plan(
        OptimizedQueryArtifact $query,
        PlanningContext $context,
    ): QueryPlanningResult;
}
```

---

# 23. QueryPlanningResult

Podrá contener:

```php
final readonly class QueryPlanningResult
{
    public function __construct(
        public LogicalQueryPlan $logicalPlan,
        public PhysicalQueryPlan $physicalPlan,
        public PlanningMetadata $metadata,
        public PlanningDiagnostics $diagnostics,
    ) {}
}
```

El `ExecutionPlan` final será formalizado separadamente en el documento 65.

---

# 24. Planner architecture

```text
OptimizedQueryArtifact
        │
        ▼
Planning Preparation
        │
        ▼
Logical Plan Construction
        │
        ▼
Logical Property Derivation
        │
        ▼
Physical Alternative Generation
        │
        ▼
Physical Property Analysis
        │
        ▼
Cost Estimation
        │
        ▼
Alternative Search
        │
        ▼
Physical Plan Selection
        │
        ▼
Plan Validation
        │
        ▼
PhysicalQueryPlan
```

---

# 25. Planning phases

VoltStack utilizará fases explícitas.

Propuesta:

```text
P0 Planning Preparation
P1 Logical Plan Construction
P2 Logical Property Derivation
P3 Physical Requirement Derivation
P4 Physical Alternative Generation
P5 Property Enforcement Planning
P6 Cost Evaluation
P7 Alternative Search
P8 Physical Plan Selection
P9 Resource Planning
P10 Final Validation
P11 Freeze
```

---

# 26. P0 — Planning Preparation

Responsabilidades:

```text
validate optimized artifact
load planning configuration
validate capabilities
load cost hints
initialize planning budget
initialize memo/search state
```

No genera todavía operadores físicos.

---

# 27. P1 — Logical Plan Construction

Transforma:

```text
OptimizedQueryArtifact
```

en:

```text
LogicalQueryPlan
```

Ejemplo:

```text
SELECT u.id
FROM users u
WHERE u.active = true
```

puede convertirse en:

```text
LogicalProject
      │
LogicalFilter
      │
LogicalRelationScan
```

---

# 28. Logical planning ≠ optimization

El Planner no deberá repetir innecesariamente las optimizaciones de los documentos 55–61.

---

# 29. Logical plan lowering

La transformación será principalmente:

```text
semantic query representation
        ↓
logical operator representation
```

---

# 30. Logical operators

Ejemplos:

```text
LogicalScan
LogicalFilter
LogicalProject
LogicalJoin
LogicalAggregate
LogicalWindow
LogicalSort
LogicalLimit
LogicalSetOperation
LogicalMaterialize
LogicalMutation
LogicalValues
```

---

# 31. Logical operator ≠ physical operator

```text
LogicalJoin
```

no significa:

```text
HashJoin
```

---

# 32. P2 — Logical Property Derivation

Cada nodo podrá poseer propiedades derivadas.

Ejemplos:

```text
output schema
cardinality bounds
uniqueness
functional dependencies
ordering
partitioning
nullability
rewindability requirements
correlation dependencies
volatility
side-effect properties
```

---

# 33. Logical properties

Modelo conceptual:

```php
final readonly class LogicalProperties
{
    public function __construct(
        public OutputRelation $output,
        public CardinalityProperties $cardinality,
        public UniquenessProperties $uniqueness,
        public OrderingProperties $ordering,
        public DistributionProperties $distribution,
        public DependencyProperties $dependencies,
    ) {}
}
```

---

# 34. Logical property ≠ cost estimate

Ejemplo:

```text
output cardinality <= 1
```

puede ser un fact.

Mientras:

```text
estimated output rows ≈ 0.8
```

es información de costo.

---

# 35. Required vs provided properties

Esta distinción será fundamental.

```text
Required Physical Property
        ≠
Provided Physical Property
```

---

# 36. Required property

Ejemplo:

```sql
ORDER BY created_at
```

puede generar un requerimiento:

```text
RequiredOrdering(created_at ASC)
```

---

# 37. Provided property

Un access path podría proporcionar naturalmente:

```text
ProvidedOrdering(created_at ASC)
```

---

# 38. Property satisfaction

Si:

```text
ProvidedOrdering
satisfies
RequiredOrdering
```

no se necesita un operador adicional de sort.

---

# 39. Property enforcement

Si no se satisface:

```text
RequiredOrdering
```

el Planner puede insertar:

```text
PhysicalSort
```

---

# 40. Enforcer

Un `Enforcer` es un operador físico introducido para satisfacer una propiedad requerida.

Ejemplos:

```text
Sort
Exchange
Materialize
Spool
Repartition
Gather
```

---

# 41. Enforcer ≠ semantic query operator

Un `ORDER BY` sí posee significado lógico.

Un sort interno utilizado sólo para permitir MergeJoin puede ser una decisión física.

---

# 42. Physical properties

Propiedades candidatas:

```text
ordering
distribution
partitioning
locality
rewindability
materialization
uniqueness
streamability
parallelism
```

---

# 43. Ordering property

```php
final readonly class PhysicalOrdering
{
    public function __construct(
        public array $keys,
    ) {}
}
```

---

# 44. Distribution property

Pensando en ejecución distribuida futura:

```text
SINGLE
HASH_PARTITIONED
RANGE_PARTITIONED
REPLICATED
ROUND_ROBIN
UNKNOWN
```

---

# 45. Distribution does not imply current distributed support

El core puede definir el concepto sin requerir que V1 implemente ejecución distribuida.

---

# 46. Streamability

Un operador podrá clasificarse conceptualmente como:

```text
STREAMING
BLOCKING
PARTIALLY_BLOCKING
```

---

# 47. Blocking operator

Ejemplos típicos potenciales:

```text
full sort
hash aggregate
materialization
```

dependiendo de implementación.

---

# 48. Rewindability

Algunas estrategias pueden requerir releer un input.

Esto será una propiedad física explícita.

---

# 49. P3 — Physical Requirement Derivation

El Planner derivará propiedades requeridas desde:

```text
query semantics
parent operators
selected physical algorithms
platform requirements
resource constraints
```

---

# 50. Top-down requirements

Algunas propiedades fluyen:

```text
Parent
  │
  ▼
Child
```

Ejemplo:

```text
ORDER BY
```

puede exigir ordering a su input.

---

# 51. Bottom-up properties

Los operadores hijos proporcionan propiedades:

```text
Child
  │
  ▼
Parent
```

---

# 52. Bidirectional planning

La planificación puede necesitar:

```text
top-down requirements
+
bottom-up derived properties
```

---

# 53. P4 — Physical Alternative Generation

Para cada operador lógico se generarán implementaciones físicas compatibles.

Ejemplo:

```text
LogicalJoin
    │
    ├── NestedLoopJoin
    ├── HashJoin
    ├── MergeJoin
    └── ExtensionJoinStrategy
```

---

# 54. Alternative generation ≠ selection

Generar:

```text
HashJoin
MergeJoin
NestedLoop
```

no significa elegir ninguna todavía.

---

# 55. Physical operator registry

Podrá existir:

```text
PhysicalOperatorRegistry
```

con descriptors inmutables.

---

# 56. Physical strategy descriptor

Ejemplo:

```php
interface PhysicalStrategyDescriptor
{
    public function id(): PhysicalStrategyId;

    public function supports(
        LogicalPlanNode $node,
        PlanningContext $context,
    ): bool;

    public function requiredChildProperties(
        LogicalPlanNode $node,
        PhysicalPropertySet $required,
    ): array;

    public function providedProperties(
        LogicalPlanNode $node,
        array $children,
    ): PhysicalPropertySet;
}
```

---

# 57. Stable strategy IDs

No se utilizará el nombre PHP de una clase como identidad semántica estable.

Ejemplo:

```text
core.join.hash
core.join.merge
core.join.nested_loop
```

---

# 58. Scan alternatives

Un:

```text
LogicalRelationScan
```

podría generar:

```text
SequentialScan
IndexScan
IndexOnlyScan
BitmapScan
ExtensionScan
```

dependiendo de capabilities.

---

# 59. Access path

`AccessPath` será concepto de Planner.

No del Query Builder.

---

# 60. AccessPath ≠ Index

Un índice es una estructura disponible.

Un access path es una forma concreta de acceder a una relación.

---

# 61. AccessPathDescriptor

Podrá incluir:

```text
relation
index/reference
supported predicates
provided ordering
estimated rows
covering properties
capabilities
```

---

# 62. Scan planning

El Planner podrá comparar:

```text
Full Scan
Index Access
Index-Only Access
Specialized Access
```

---

# 63. Join physical strategies

Ejemplos:

```text
NestedLoopJoin
IndexNestedLoopJoin
HashJoin
MergeJoin
```

---

# 64. Logical join type remains authoritative

Una estrategia física no podrá convertir arbitrariamente:

```text
LEFT JOIN
```

en:

```text
INNER JOIN
```

Eso corresponde al Optimizer y requiere prueba semántica.

---

# 65. Join algorithm selection

Dependerá de:

```text
join type
predicate form
input cardinalities
ordering
indexes
memory
platform capabilities
cost
```

---

# 66. Hash Join

Puede requerir:

```text
hash-compatible equality keys
sufficient capability
appropriate memory/resource policy
```

---

# 67. Merge Join

Puede requerir:

```text
compatible ordering
merge-compatible predicates
```

y quizá introducir sorts.

---

# 68. Nested Loop

Puede ser apropiado para:

```text
small outer relation
cheap inner access
correlated execution
index lookup
```

pero estas son decisiones del cost/planning model.

---

# 69. Aggregate strategies

Un:

```text
LogicalAggregate
```

podría mapear a:

```text
HashAggregate
SortAggregate
StreamingAggregate
PartialAggregate
ParallelAggregate
ExtensionAggregate
```

---

# 70. Window strategies

Un:

```text
LogicalWindow
```

podrá generar:

```text
SortWindow
StreamingWindow
PartitionedWindow
MaterializedWindow
```

dependiendo de propiedades.

---

# 71. Set operation strategies

Ejemplos:

```text
Append
HashDistinct
SortDistinct
HashIntersect
SortIntersect
HashExcept
SortExcept
MultisetCounter
```

según operación/capability.

---

# 72. Mutation planning

DML también requerirá planificación.

```text
LogicalInsert
LogicalUpdate
LogicalDelete
```

podrán tener estrategias físicas propias.

---

# 73. DML correctness

El Planner deberá preservar:

```text
mutation target
affected row semantics
RETURNING semantics
locking
ordering/limit semantics
trigger-visible behavior
```

---

# 74. Mutation ≠ SELECT planning

No se reutilizarán estrategias de SELECT sin considerar efectos observables.

---

# 75. Materialization

El Planner podrá decidir introducir materialización cuando sea legal.

---

# 76. Materialization requirements

Debe respetar:

```text
REQUIRE_MATERIALIZED
REQUIRE_INLINE
PROHIBIT
security barriers
volatility
recursive semantics
```

---

# 77. CTE planning

Un CTE podrá terminar como:

```text
InlineLogicalPlan
MaterializedPlan
ReusableSpool
PlatformNativeCTE
```

según semántica, capabilities y política.

---

# 78. Recursive CTE

Requerirá estrategia física especializada.

No se tratará como simple CTE no recursivo.

---

# 79. Correlation planning

Correlated subqueries no decorrelacionadas podrán requerir estrategias como:

```text
ParameterizedNestedLoop
CorrelatedSubplan
```

---

# 80. Correlation ≠ per-row execution

La existencia de correlación no implica automáticamente ejecución ingenua por fila.

El Planner puede encontrar estrategias físicas mejores.

---

# 81. P5 — Property Enforcement Planning

Una alternativa puede necesitar enforcers.

Ejemplo:

```text
MergeJoin
```

requiere:

```text
left ordered by key
right ordered by key
```

Si los inputs no lo proporcionan:

```text
Sort(left)
Sort(right)
```

pueden convertirse en alternativas.

---

# 82. Enforcer cost

Los enforcers tienen costo.

No se insertarán gratuitamente.

---

# 83. Enforcer search

Conceptualmente:

```text
Required Property
      │
      ▼
Already Provided?
   ┌──┴──┐
  yes    no
   │      │
   ▼      ▼
 reuse   generate enforcer alternatives
```

---

# 84. Property subsumption

Una propiedad puede satisfacer otra más débil.

Ejemplo:

```text
ORDER BY (a, b, c)
```

puede satisfacer ciertos requerimientos:

```text
ORDER BY (a, b)
```

según semántica de ordering definida.

---

# 85. PhysicalPropertySet

```php
final readonly class PhysicalPropertySet
{
    public function __construct(
        public ?PhysicalOrdering $ordering,
        public ?PhysicalDistribution $distribution,
        public ?Rewindability $rewindability,
        public ?MaterializationProperty $materialization,
        public ?ParallelismProperty $parallelism,
    ) {}
}
```

---

# 86. Property satisfaction service

```php
interface PhysicalPropertySatisfaction
{
    public function satisfies(
        PhysicalPropertySet $provided,
        PhysicalPropertySet $required,
    ): bool;
}
```

---

# 87. P6 — Cost Evaluation

Cada alternativa física podrá recibir:

```text
EstimatedPhysicalCost
```

---

# 88. Cost Estimate ≠ Semantic Fact

Siempre:

```text
EstimatedPhysicalCost
≠
Semantic Truth
```

---

# 89. PhysicalCostModel

Contrato conceptual:

```php
interface PhysicalCostModel
{
    public function estimate(
        PhysicalPlanCandidate $candidate,
        PlanningContext $context,
    ): EstimatedPhysicalCost;
}
```

---

# 90. Cost dimensions

El costo podrá considerar:

```text
CPU
I/O
memory
network
latency
startup
total work
temporary storage
parallel overhead
```

---

# 91. Startup cost vs total cost

Esto será importante para consultas como:

```text
LIMIT 1
```

donde:

```text
low startup cost
```

puede ser más importante que mínimo costo total.

---

# 92. CostVector

Podrá reutilizar/concretar el modelo del documento 61:

```text
CPU
IO
MEMORY
NETWORK
LATENCY
```

---

# 93. Physical cost

Podrá contener:

```php
final readonly class EstimatedPhysicalCost
{
    public function __construct(
        public CostVector $vector,
        public CostConfidence $confidence,
        public CostBreakdown $breakdown,
    ) {}
}
```

---

# 94. Cost breakdown

Para explainability:

```text
Scan cost
Join cost
Sort cost
Materialization cost
Network cost
Memory penalty
Parallel overhead
```

---

# 95. Cost comparison

No deberá necesariamente reducirse inmediatamente a un solo `float`.

---

# 96. Cost comparator

```php
interface PhysicalCostComparator
{
    public function compare(
        EstimatedPhysicalCost $a,
        EstimatedPhysicalCost $b,
        OptimizationObjective $objective,
    ): CostComparison;
}
```

---

# 97. Cost uncertainty

Dos planes pueden tener costos demasiado cercanos o estimaciones demasiado inciertas.

El Planner podrá representar:

```text
EQUIVALENT
A_PREFERRED
B_PREFERRED
INCOMPARABLE
UNCERTAIN
```

---

# 98. Stable tie breaking

Si dos planes son equivalentes según el modelo:

```text
deterministic tie-breaking
```

será obligatorio.

---

# 99. No random planning

Mismos inputs:

```text
query
statistics
capabilities
configuration
planner version
```

deberán producir el mismo plan, salvo modo adaptativo explícito.

---

# 100. P7 — Alternative Search

El espacio físico puede crecer combinatoriamente.

Ejemplo:

```text
3 join algorithms
×
4 access paths
×
3 orderings
×
2 materialization choices
...
```

---

# 101. Exhaustive search

No siempre será viable.

---

# 102. Search strategies

La arquitectura permitirá:

```text
Rule-Based Search
Greedy Search
Dynamic Programming
Memoized Search
Branch-and-Bound
Cascades-Style Search
Extension Search
```

---

# 103. V1 strategy

Para V1 se recomienda:

```text
bounded memoized planning
+
heuristic alternative ordering
+
cost-based selection
```

sin requerir desde el inicio un Cascades completo.

---

# 104. Future evolution

La arquitectura deberá permitir evolucionar a:

```text
Cascades-style optimizer/planner
```

sin cambiar la API pública del Query Builder.

---

# 105. Planner memo

Podrá existir:

```text
PlanningMemo
```

---

# 106. PlanningMemo purpose

Almacena:

```text
equivalent logical groups
physical alternatives
required properties
provided properties
cost estimates
best known alternatives
```

---

# 107. Memo ≠ Query Deduplication System

Documento 60 deduplica estructuras durante optimización.

`PlanningMemo` organiza alternativas durante planificación física.

---

# 108. Memo group

Conceptualmente:

```text
MemoGroup
├── LogicalExpression A
├── LogicalExpression B
├── PhysicalCandidate C
├── PhysicalCandidate D
└── BestPlansByRequiredProperty
```

---

# 109. Required properties in memo key

El mejor plan para:

```text
unordered output
```

puede no ser el mejor para:

```text
ordered output
```

Por tanto:

```text
MemoKey
=
LogicalGroup
+
RequiredPhysicalProperties
```

---

# 110. BestPlanEntry

```php
final readonly class BestPlanEntry
{
    public function __construct(
        public MemoGroupId $group,
        public PhysicalPropertySet $requiredProperties,
        public PhysicalPlanCandidate $plan,
        public EstimatedPhysicalCost $cost,
    ) {}
}
```

---

# 111. Branch and bound

Si:

```text
current candidate cost
>
best known cost
```

y existe prueba suficiente de lower bound:

```text
candidate branch
```

puede descartarse.

---

# 112. Estimate uncertainty and pruning

Con estimaciones poco confiables, el Planner deberá ser más conservador al podar.

---

# 113. Search budget

El Planner tendrá límites explícitos.

---

# 114. PlanningBudget

```php
final readonly class PlanningBudget
{
    public function __construct(
        public int $maxLogicalNodes,
        public int $maxPhysicalAlternatives,
        public int $maxMemoGroups,
        public int $maxMemoEntries,
        public int $maxRuleApplications,
        public int $maxPropertyAlternatives,
        public int $maxEnforcers,
        public int $maxSearchWorkUnits,
        public int $maxPlanDepth,
    ) {}
}
```

---

# 115. Work units

Se preferirán:

```text
deterministic work units
```

sobre depender exclusivamente de wall-clock timeout.

---

# 116. Wall clock timeout

Podrá existir como protección externa.

Pero:

```text
wall clock
≠
deterministic planning budget
```

---

# 117. Budget exhaustion

Si se agota el budget:

```text
Planner
```

podrá devolver:

```text
best fully valid plan found
```

si la política lo permite.

Nunca:

```text
partially constructed invalid plan
```

---

# 118. Minimum viable plan

El Planner deberá intentar mantener una estrategia segura de fallback.

Ejemplo:

```text
simple scan
+
nested loop
+
explicit sort
```

si capabilities lo permiten.

---

# 119. Fallback ≠ silent semantic downgrade

El fallback podrá ser menos eficiente.

Nunca semánticamente diferente.

---

# 120. P8 — Physical Plan Selection

Después de explorar alternativas:

```text
Candidate Plans
      │
      ▼
Constraint Validation
      │
      ▼
Cost Comparison
      │
      ▼
Objective Evaluation
      │
      ▼
Deterministic Tie Break
      │
      ▼
Selected Physical Plan
```

---

# 121. Hard constraints first

Antes de comparar costo se eliminan alternativas que violen:

```text
semantic requirements
capabilities
security barriers
resource hard limits
REQUIRE hints
PROHIBIT hints
```

---

# 122. Cost second

Sólo entre planes válidos:

```text
cost
```

decide preferencia.

---

# 123. Correctness before cost

Regla:

```text
Valid expensive plan
>
Invalid cheap plan
```

siempre.

---

# 124. Optimization objective

Puede ser:

```text
BALANCED
LATENCY
THROUGHPUT
MEMORY
IO
NETWORK
```

como se definió conceptualmente en documento 61.

---

# 125. Objective does not alter results

Cambiar:

```text
LATENCY
→
MEMORY
```

puede cambiar plan.

No resultados.

---

# 126. P9 — Resource Planning

El Planner podrá modelar recursos.

Ejemplos:

```text
memory
temporary storage
parallel workers
network
spill
buffering
```

---

# 127. Resource requirement

```php
final readonly class PhysicalResourceRequirement
{
    public function __construct(
        public ?MemoryRequirement $memory,
        public ?TemporaryStorageRequirement $temporaryStorage,
        public ?ParallelismRequirement $parallelism,
        public ?NetworkRequirement $network,
    ) {}
}
```

---

# 128. Resource estimate ≠ reservation

El Planner estima necesidades.

El runtime puede realizar posteriormente reservas/admission control.

---

# 129. Memory planning

Ejemplo:

```text
HashJoin:
    expected hash table memory = 200 MB
```

Si budget:

```text
64 MB
```

el Planner podrá:

```text
choose another strategy
```

o:

```text
plan spill
```

si la plataforma lo soporta.

---

# 130. Spill

Spill será propiedad/estrategia explícita.

No comportamiento accidental.

---

# 131. Temporary storage

Operadores como:

```text
sort
hash
materialize
spool
```

pueden requerir almacenamiento temporal.

---

# 132. Parallel planning

El Planner podrá considerar:

```text
serial
parallel scan
parallel aggregate
parallel join
parallel partitions
```

según capabilities.

---

# 133. Parallelism ≠ concurrency framework

La planificación paralela de una query es distinta del sistema general `Quantum/Concurrency`.

---

# 134. Distributed planning

Futuro:

```text
Logical Plan
     │
     ▼
Distribution Planning
     │
     ├── Local operators
     ├── Remote operators
     ├── Exchanges
     └── Gather
```

---

# 135. Exchange

Un `Exchange` será un operador físico para cambiar distribución.

Ejemplos:

```text
Gather
Broadcast
HashExchange
RangeExchange
```

---

# 136. Network-aware cost

Distributed planning utilizará:

```text
NetworkCostHint
LocalityHint
DistributionProperty
```

del sistema de costo.

---

# 137. Capability-driven planning

Nunca:

```php
if ($database === 'postgresql') {
    useStrategy(...);
}
```

---

# 138. Capability snapshot

Preferible:

```php
if ($capabilities->supports(
    DatabaseCapability::HASH_JOIN
)) {
    ...
}
```

conceptualmente.

---

# 139. Native vs emulated capability

Las capabilities podrán indicar:

```text
NATIVE
EMULATABLE
RESTRICTED
UNSUPPORTED
```

---

# 140. Planner capability requirement

Una physical strategy declarará explícitamente sus requisitos.

---

# 141. Platform-independent logical plan

`LogicalQueryPlan` deberá ser lo más independiente posible del motor.

---

# 142. Platform-aware physical plan

`PhysicalQueryPlan` puede depender de capabilities de la plataforma seleccionada.

---

# 143. Vendor-specific physical strategies

Podrán existir mediante extensiones.

Ejemplo conceptual:

```text
PostgreSqlBitmapHeapScan
```

pero detrás de:

```text
PhysicalStrategyDescriptor
+
Capabilities
```

---

# 144. Core does not require vendor branching

La registry decide qué estrategias están disponibles.

---

# 145. Compiler boundary

El Planner puede seleccionar:

```text
PhysicalIndexScan
```

El Compiler determina cómo representarlo para el target.

---

# 146. Planner does not emit SQL

Nunca:

```php
$planner->plan(...) === "SELECT ..."
```

---

# 147. Compiler does not re-plan

El Compiler no deberá decidir:

```text
actually use hash join instead
```

---

# 148. Compiler unsupported plan

Si un Compiler no puede representar un plan seleccionado:

```text
architecture/configuration error
```

o capability inconsistency.

No deberá improvisar una estrategia distinta.

---

# 149. Native SQL databases

Existe una consideración importante:

MySQL, PostgreSQL y SQLite poseen sus propios query planners.

Por tanto, VoltStack no debe asumir que siempre controla directamente todos los algoritmos físicos internos.

---

# 150. Planning abstraction levels

VoltStack soportará conceptualmente:

```text
Framework Logical Planning
Framework Physical Intent Planning
Native Database Planning
Framework Execution Planning
```

---

# 151. Physical intent

Para motores SQL tradicionales, algunos physical nodes podrán representar:

```text
desired execution properties
available access path
compiler guidance
framework-side execution strategy
```

sin garantizar que el DBMS use exactamente un algoritmo interno.

---

# 152. Native planner authority

Si el motor no permite controlar:

```text
HashJoin vs MergeJoin
```

VoltStack no fingirá que puede hacerlo.

---

# 153. Capability accuracy

Ejemplo:

```text
supportsExplicitHashJoinSelection = false
```

aunque el DBMS internamente soporte hash joins.

---

# 154. Framework-controlled vs engine-controlled

Cada estrategia podrá declarar:

```text
FRAMEWORK_CONTROLLED
ENGINE_GUIDED
ENGINE_CONTROLLED
HYBRID
```

---

# 155. ExecutionControlLevel

```php
enum ExecutionControlLevel
{
    case FRAMEWORK_CONTROLLED;
    case ENGINE_GUIDED;
    case ENGINE_CONTROLLED;
    case HYBRID;
}
```

---

# 156. Why this matters

Evita diseñar un Planner ficticio que afirme controlar detalles que realmente pertenecen al motor SQL.

---

# 157. Example PostgreSQL-like target

VoltStack podría decidir:

```text
query structure
CTE materialization intent
index-compatible shape
ordering
framework-side batching
```

mientras el DBMS decide internamente:

```text
actual hash join
buffer strategy
parallel worker details
```

según capabilities.

---

# 158. SQLite target

Un conjunto diferente de capabilities producirá un espacio de planificación diferente.

---

# 159. Future non-SQL drivers

Esta separación permitirá que futuros drivers controlados completamente por VoltStack implementen planes físicos más directos.

---

# 160. Planner modes

Podrán existir:

```text
NATIVE_DATABASE
FRAMEWORK_ASSISTED
FRAMEWORK_CONTROLLED
DISTRIBUTED
```

---

# 161. PlanningMode

```php
enum PlanningMode
{
    case NATIVE_DATABASE;
    case FRAMEWORK_ASSISTED;
    case FRAMEWORK_CONTROLLED;
    case DISTRIBUTED;
}
```

---

# 162. Native database mode

El Planner puede limitarse a:

```text
logical planning
query shaping
capability validation
access-path hints
materialization decisions
execution wrapper planning
```

dejando al DBMS su physical optimizer interno.

---

# 163. Framework-assisted mode

VoltStack puede influir más mediante:

```text
query rewrites
hints
CTE strategies
query decomposition
batching
temporary relations
```

---

# 164. Framework-controlled mode

VoltStack controla directamente operadores físicos.

Útil para futuros:

```text
federated engines
specialized drivers
in-memory execution
```

---

# 165. Distributed mode

El Planner puede fragmentar el plan entre múltiples execution locations.

Reservado para arquitectura futura.

---

# 166. Plan fragments

En modo distribuido:

```text
PhysicalQueryPlan
├── Fragment A
├── Fragment B
└── Exchange Edges
```

---

# 167. Plan identity

Cada nodo tendrá identidad.

```text
LogicalPlanNodeId
PhysicalPlanNodeId
ExecutionPlanNodeId
```

serán distintos.

---

# 168. Semantic ID ≠ Plan Node ID

Ejemplo:

```text
RelationId(R17)
```

puede producir:

```text
LogicalPlanNodeId(L4)
```

que luego produce:

```text
PhysicalPlanNodeId(P9)
```

---

# 169. Identity mapping

Se conservarán mappings:

```text
SemanticId
   ↓
LogicalPlanNodeId
   ↓
PhysicalPlanNodeId
```

para diagnostics y telemetry.

---

# 170. Output identity preservation

El Planner podrá cambiar operadores internos.

No deberá cambiar el contrato observable de output.

---

# 171. OutputContract

Incluye:

```text
column count
column identity
semantic type
nullability
ordering when semantically required
names/aliases
lineage mapping
```

---

# 172. Internal projection pruning

Si el Optimizer elimina columnas internas innecesarias:

```text
public output contract
```

debe mantenerse.

---

# 173. Parameter preservation

El Planner no cambiará arbitrariamente:

```text
ParameterId
```

---

# 174. Runtime values

El Planner genérico no inspeccionará `BindingSet`.

---

# 175. Parameter-sensitive planning

Requerirá una fase de especialización explícita futura.

---

# 176. Plan parameters

El Execution Plan podrá contener:

```text
ParameterId references
```

pero no necesariamente sus valores.

---

# 177. Placeholder generation

Sigue siendo responsabilidad del Compiler.

---

# 178. Planning barriers

El Planner deberá respetar barriers provenientes de fases anteriores.

Ejemplos:

```text
RawBarrier
SecurityBarrier
PolicyBarrier
VolatilityBarrier
LockingBarrier
MutationBarrier
CorrelationBarrier
MaterializationBarrier
ExtensionBarrier
```

---

# 179. Raw barrier

Un raw expression opaco puede impedir determinadas estrategias.

---

# 180. Security barrier

No podrá moverse ejecución de forma que:

```text
expose rows before mandatory policy filtering
```

si el security contract lo prohíbe.

---

# 181. Volatility barrier

Una función volátil puede impedir:

```text
duplicate evaluation
elimination
reordering
precomputation
```

---

# 182. Locking barrier

Queries con:

```text
FOR UPDATE
```

u otros locks pueden limitar access paths/ordering/distribution.

---

# 183. Mutation barrier

Un UPDATE/DELETE no podrá planificarse como si fuera un SELECT puro.

---

# 184. Correlation barrier

Dependencias laterales/correlacionadas restringen reordenamiento y ejecución.

---

# 185. Extension barrier

Una extensión sin contratos físicos suficientes deberá limitar optimizaciones/estrategias.

---

# 186. Planner extension system

El sistema del documento 54 permitirá registrar extensiones del Planner.

---

# 187. Planner extension obligations

Una physical strategy extension deberá declarar:

```text
stable ID
supported logical nodes
required capabilities
required properties
provided properties
cost estimator
resource requirements
compiler support
execution control level
fingerprint contribution
diagnostics
```

---

# 188. No compiler-only physical extension

Una extensión física ejecutable deberá integrarse correctamente al Planner.

No aparecer únicamente como SQL string especial.

---

# 189. Planner rule system

Además de physical descriptors podrán existir:

```text
PlanningRule
```

---

# 190. PlanningRule ≠ OptimizationRule

```text
OptimizationRule
    transforms equivalent logical query forms

PlanningRule
    derives/plans execution alternatives
```

---

# 191. PlanningRule contract

```php
interface PlanningRule
{
    public function id(): PlanningRuleId;

    public function matches(
        PlanningExpression $expression,
        PlanningContext $context,
    ): bool;

    public function alternatives(
        PlanningExpression $expression,
        PlanningContext $context,
    ): iterable;
}
```

---

# 192. Planning rules examples

```text
LogicalJoin → HashJoin
LogicalJoin → MergeJoin
LogicalJoin → NestedLoopJoin

LogicalAggregate → HashAggregate
LogicalAggregate → SortAggregate

LogicalScan → SequentialScan
LogicalScan → IndexScan
```

---

# 193. Rule registry

Será:

```text
immutable
frozen after bootstrap
versioned
deterministically ordered
```

---

# 194. Registration order ≠ planning order

La ejecución dependerá de:

```text
phase
dependencies
priority
strategy
```

no de quién se registró primero.

---

# 195. Planning rule failures

`NoMatch` es normal.

Un rule producing an invalid physical plan:

```text
internal invariant violation
```

---

# 196. Planning trace

VoltStack podrá generar:

```text
PlanningTrace
```

---

# 197. PlanningTrace contents

Ejemplo:

```text
LogicalJoin J7

Generated:
  HashJoin H1
  MergeJoin M1
  NestedLoop N1

Rejected:
  MergeJoin M1
  reason: required ordering too expensive

Rejected:
  HashJoin H1
  reason: memory hard limit

Selected:
  NestedLoop N1

Cost:
  startup: LOW
  total: MEDIUM
```

---

# 198. Explain plan

El sistema deberá poder responder:

```text
Why was this plan selected?
```

---

# 199. Explain alternatives

Cuando diagnostics estén habilitados:

```text
candidate
cost
requirements
provided properties
rejection reason
```

podrán exponerse.

---

# 200. Trace budget

No se almacenará un trace ilimitado.

---

# 201. Production mode

Podrá conservar únicamente:

```text
selected plan
summary
important rejected constraints
fingerprints
```

---

# 202. Development mode

Podrá conservar:

```text
detailed alternatives
cost breakdown
rule trace
property derivation
```

---

# 203. Planning diagnostics

Tipos:

```text
INFO
WARNING
ERROR
INTERNAL_ERROR
```

---

# 204. Diagnostic examples

```text
STATISTICS_MISSING
STALE_STATISTICS
NO_INDEX_ACCESS_PATH
REQUIRED_PROPERTY_ENFORCED
HIGH_MEMORY_PLAN_REJECTED
MATERIALIZATION_REQUIRED
NATIVE_PLANNER_CONTROL_LIMITED
PLANNING_BUDGET_EXHAUSTED
```

---

# 205. Plan fingerprinting

Cada artifact tendrá fingerprint distinto.

---

# 206. LogicalPlanFingerprint

Puede depender de:

```text
OptimizedQueryFingerprint
LogicalPlanStructure
PlannerLogicalVersion
ExtensionVersions
```

---

# 207. PhysicalPlanFingerprint

Puede depender de:

```text
LogicalPlanFingerprint
PhysicalStrategies
Capabilities
StatisticsVersion
CostHintFingerprint
PlanningConfiguration
PlannerVersion
ExtensionVersions
```

---

# 208. ExecutionPlanFingerprint

Documento 65 añadirá detalles de ejecución/compilación necesarios.

---

# 209. SemanticFingerprint stability

Cambiar estadísticas no deberá cambiar:

```text
SemanticFingerprint
```

---

# 210. Physical plan changes

Cambiar estadísticas sí puede cambiar:

```text
PhysicalPlanFingerprint
```

---

# 211. Plan caching

Esto permitirá posteriormente:

```text
LogicalPlanCache
PhysicalPlanCache
ExecutionPlanCache
```

con invalidez independiente.

---

# 212. Cache correctness

Una physical plan cache deberá considerar:

```text
capability version
statistics version
schema dependencies
planner version
extension versions
configuration
```

cuando sean relevantes.

---

# 213. Cached plan ≠ permanently valid plan

Siempre existirá política de invalidez.

---

# 214. Schema changes

Pueden invalidar:

```text
access paths
indexes
types
constraints
capabilities
```

y por tanto physical plans.

---

# 215. Statistics changes

Pueden hacer un plan menos óptimo sin volverlo necesariamente semánticamente inválido.

---

# 216. Capability changes

Pueden hacer un plan físicamente inválido.

---

# 217. Extension changes

Pueden invalidar:

```text
physical strategy IDs
cost models
serialization
compiler support
```

---

# 218. Planner versioning

`PlannerVersion` participará en caches persistentes cuando corresponda.

---

# 219. Plan serialization

Artifacts persistibles tendrán codecs versionados.

---

# 220. Unknown physical strategy

Al deserializar:

```text
unknown strategy ID
```

no se convertirá a una estrategia genérica silenciosamente.

---

# 221. Replanning

Resultado:

```text
plan invalid
→
replan
```

cuando sea posible.

---

# 222. Replanning ≠ retry execution

Son mecanismos diferentes.

---

# 223. Adaptive planning

Arquitectura futura podrá soportar:

```text
Plan
  │
  ▼
Execution Feedback
  │
  ▼
Replanning Decision
  │
  ▼
Alternative Plan
```

---

# 224. Adaptive planning boundaries

No deberá modificar resultados semánticos.

---

# 225. Mid-query replanning

Será una capability avanzada.

No requisito V1.

---

# 226. Runtime cardinality feedback

Podrá detectar:

```text
estimated 1,000 rows
actual 100,000,000 rows
```

y disparar telemetry o futura adaptación.

---

# 227. Planner remains deterministic by default

Adaptive behavior deberá habilitarse explícitamente.

---

# 228. Planner safety model

Antes de seleccionar un plan deberá verificarse:

```text
semantic preservation
output preservation
capability satisfaction
barrier compliance
resource hard constraints
security policy
parameter preservation
mutation correctness
```

---

# 229. PlanValidator

```php
interface PhysicalPlanValidator
{
    public function validate(
        PhysicalQueryPlan $plan,
        PlanningValidationContext $context,
    ): PlanningValidationResult;
}
```

---

# 230. Validation does not optimize

El validator no deberá arreglar silenciosamente un plan inválido.

---

# 231. Invalid plan

Debe:

```text
reject candidate
```

o:

```text
fail planning
```

según etapa.

---

# 232. Final selected plan

Debe pasar validación completa.

---

# 233. Planner failure model

Errores posibles:

```text
NoValidPlanException
PlanningBudgetExceededException
RequiredPhysicalPropertyException
UnsupportedPhysicalStrategyException
CapabilityPlanningException
PhysicalPlanValidationException
PlannerExtensionException
CostModelException
ResourcePlanningException
PlanSerializationException
```

---

# 234. No valid plan

Una query semánticamente válida puede no tener plan válido para una plataforma concreta.

Ejemplo:

```text
required feature
+
platform unsupported
+
no exact emulation
```

---

# 235. No silent semantic emulation

Si no existe emulación exacta:

```text
fail explicitly
```

---

# 236. Resource hard failure

Si todo plan requiere más recursos que un hard policy permite:

```text
ResourcePlanningException
```

puede ser correcto.

---

# 237. Planner telemetry

Métricas:

```text
planning duration
planning work units
logical nodes
physical alternatives
memo groups
memo entries
planning rules applied
candidates rejected
enforcers generated
cost evaluations
budget exhaustion
selected strategy distribution
planning cache hit/miss
replanning count
```

---

# 238. Strategy telemetry

Podrá mostrar:

```text
HashJoin selected: X
NestedLoop selected: Y
IndexScan selected: Z
```

cuando VoltStack realmente controle esas estrategias.

---

# 239. Native planner telemetry

Para estrategias engine-controlled:

```text
VoltStack physical intent
```

deberá distinguirse de:

```text
actual DBMS execution plan
```

---

# 240. Actual DB execution plan

La obtención de:

```text
EXPLAIN
EXPLAIN ANALYZE
```

pertenecerá a Telemetry/Profiler/Diagnostics.

No a planning puro.

---

# 241. Planner cannot validate itself with EXPLAIN during planning

No:

```text
plan
→ EXPLAIN
→ modify plan
```

como comportamiento core implícito.

---

# 242. Offline planning

Una meta importante será permitir:

```text
planning tests
```

sin una conexión real cuando existan snapshots suficientes.

---

# 243. Testability

Se podrán probar:

```text
logical plan
physical alternatives
cost comparison
property enforcement
capability restrictions
budget behavior
fingerprints
```

de forma determinista.

---

# 244. Conformance tests

Cada physical strategy deberá pasar tests de:

```text
supported logical nodes
required properties
provided properties
capabilities
cost integration
serialization
compiler compatibility
persistent runtime
```

---

# 245. Persistent runtime safety

Con FrankenPHP:

```text
PlanningRuleRegistry
PhysicalStrategyRegistry
CostModel descriptors
Property descriptors
```

podrán ser shared immutable.

---

# 246. Operation-local planner state

Deberá incluir:

```text
PlanningMemo
SearchQueue
PlanningTrace
BudgetCounters
CandidateSet
CostCache
PropertyCache
Diagnostics
```

---

# 247. Forbidden singleton state

Nunca:

```text
currentPlan
currentQuery
currentTenant
currentMemo
currentRequiredOrdering
currentBestCost
```

en singleton mutable.

---

# 248. Worker reset

Toda referencia operation-scoped será liberada al terminar la operación/request.

---

# 249. Concurrency

Dos queries planificadas simultáneamente:

```text
Q1
Q2
```

no compartirán mutable planning state.

---

# 250. Immutable artifacts

Una vez congelados:

```text
LogicalQueryPlan
PhysicalQueryPlan
```

serán inmutables.

---

# 251. Planner internal mutable state

Durante búsqueda sí puede existir estado mutable local.

Pero:

```text
mutable planner search state
```

nunca escapará como artifact final.

---

# 252. Thread/coroutine safety

La arquitectura no dependerá de globals ni static mutable state.

---

# 253. Security integration

El Planner consumirá security metadata/barriers ya explícitos.

---

# 254. Planner does not authorize

No preguntará:

```text
is user admin?
```

---

# 255. Authorization decisions occur before planning

Cualquier policy relevante deberá estar materializada como:

```text
query structure
semantic metadata
security barrier
planning constraint
```

antes del Planner.

---

# 256. Mandatory security predicates

No podrán desaparecer por razones de costo.

---

# 257. Security-sensitive physical planning

Algunos planes pueden estar prohibidos por:

```text
data locality
sensitive materialization
temporary storage restrictions
cross-region restrictions
```

---

# 258. Security physical constraints

Podrán expresarse mediante:

```text
PhysicalPlanningConstraint
```

---

# 259. Temporary data security

Un materialized intermediate podría requerir:

```text
encrypted temp storage
memory-only execution
restricted locality
```

en sistemas avanzados.

---

# 260. Multitenancy integration

El Planner no dependerá del paquete Multitenancy.

---

# 261. Tenant planning constraints

El paquete opcional podrá aportar:

```text
tenant-locality constraints
tenant-specific statistics
tenant resource limits
tenant physical capabilities
```

mediante interfaces.

---

# 262. Tenant plan isolation

Caches de planes deberán considerar contexto tenant cuando afecte planificación.

---

# 263. Same semantic query, different physical plan

Puede ocurrir:

```text
Tenant A:
    100 rows

Tenant B:
    100,000,000 rows
```

La misma estructura lógica podría producir planes físicos diferentes.

---

# 264. Semantic cache vs physical cache

Esto demuestra por qué:

```text
SemanticQueryCache
≠
PhysicalPlanCache
```

---

# 265. ORM integration

ORM produce consultas estructuradas.

No planes.

---

# 266. ORM does not choose access paths

Nunca:

```text
EntityManager
→ choose HashJoin
```

---

# 267. Active Record

Igualmente:

```text
Model::query()
```

termina en el mismo Query Engine y Planner.

No existe un Planner especial de Active Record.

---

# 268. Repository API

También utiliza el mismo pipeline.

---

# 269. One Planner

Regla:

```text
Active Record
Repository
EntityManager
Query Builder
    │
    └────► Same Query Planner
```

---

# 270. Planner and Execution Engine

El Planner produce artifacts.

Execution Engine los consume.

---

# 271. Planner does not execute

No:

```php
return $connection->execute($plan);
```

---

# 272. Executor does not optimize

No:

```text
Executor notices slow join
→ changes logical join order
```

salvo futuro adaptive execution formal.

---

# 273. Compiler pipeline relationship

Después del bloque actual:

```text
Physical/Execution Plan
       │
       ▼
SQL Compiler Architecture
       │
       ▼
Platform SQL
```

Los documentos 66–75 formalizarán esa etapa.

---

# 274. Planning for SQL compilation

El plan deberá proporcionar suficiente información para que el Compiler no tenga que adivinar.

---

# 275. Compiler requirements

Ejemplos:

```text
selected query structure
materialization decisions
required ordering
platform capability decisions
parameter references
selected emulation strategy
```

---

# 276. Native optimizer hints

Si una plataforma soporta hints nativos y el Planner decide usarlos:

```text
PhysicalPlan
```

deberá contener una representación estructurada.

El Compiler sólo renderiza.

---

# 277. No raw hint string

Preferible:

```text
NativePlannerDirective(
    semantic ID,
    structured arguments
)
```

sobre:

```text
"/*+ HASH_JOIN */"
```

---

# 278. Plan portability

Logical plans serán más portables que physical plans.

---

# 279. Physical portability

Un Physical Plan puede declarar:

```text
portable
platform-family-specific
platform-specific
extension-specific
```

---

# 280. PortabilityProfile

```php
enum PlanPortability
{
    case PORTABLE;
    case CAPABILITY_DEPENDENT;
    case PLATFORM_FAMILY_SPECIFIC;
    case PLATFORM_SPECIFIC;
    case EXTENSION_SPECIFIC;
}
```

---

# 281. Planning profile

Podrán existir perfiles:

```text
MINIMAL
DEFAULT
THOROUGH
```

para controlar profundidad de búsqueda.

---

# 282. Planning effort ≠ semantics

Siempre:

```text
Planning Effort
≠
Query Meaning
```

---

# 283. Minimal profile

Puede usar:

```text
small alternative set
simple heuristics
limited memo
```

---

# 284. Thorough profile

Puede:

```text
explore more alternatives
evaluate more properties
use richer statistics
perform deeper cost analysis
```

---

# 285. Profile fingerprint

El perfil participará en physical plan fingerprints/caches cuando pueda cambiar el plan.

---

# 286. Planning configuration

Ejemplo:

```php
final readonly class PlanningConfiguration
{
    public function __construct(
        public PlanningProfile $profile,
        public PlanningMode $mode,
        public OptimizationObjective $objective,
        public bool $enableMemoization,
        public bool $enablePropertyEnforcement,
        public bool $enableDetailedTrace,
    ) {}
}
```

---

# 287. Default configuration

Debe ser:

```text
safe
deterministic
bounded
portable where reasonable
```

---

# 288. Production defaults

Evitarán:

```text
unbounded search
full trace
expensive diagnostics
```

---

# 289. Development defaults

Podrán habilitar:

```text
planning explanations
candidate traces
cost breakdowns
```

---

# 290. Planner architecture overview

```text
                    OptimizedQueryArtifact
                             │
                             ▼
                    PlanningCoordinator
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
     LogicalPlanBuilder               PlanningContext
              │                             │
              ▼                             │
       LogicalQueryPlan                     │
              │                             │
              ▼                             │
   LogicalPropertyDeriver                   │
              │                             │
              └──────────────┬──────────────┘
                             ▼
                    PlanningMemo/Search
                             │
               ┌─────────────┼──────────────┐
               ▼             ▼              ▼
          Scan Planner   Join Planner   Aggregate Planner
               │             │              │
               └─────────────┼──────────────┘
                             ▼
                 Physical Alternatives
                             │
                             ▼
                   Property Enforcement
                             │
                             ▼
                       Cost Model
                             │
                             ▼
                      Plan Selection
                             │
                             ▼
                    PhysicalQueryPlan
                             │
                             ▼
                     Plan Validation
                             │
                             ▼
                    Frozen Plan Artifact
```

---

# 291. Proposed namespaces

```text
VoltStack\Quantum\Database\Query\Planner
```

---

# 292. Proposed directory structure

```text
Query/
└── Planner/
    ├── Contract/
    │   ├── QueryPlanner.php
    │   ├── LogicalPlanBuilder.php
    │   ├── PhysicalStrategyDescriptor.php
    │   ├── PlanningRule.php
    │   ├── PhysicalCostModel.php
    │   ├── PhysicalCostComparator.php
    │   ├── PhysicalPropertySatisfaction.php
    │   └── PhysicalPlanValidator.php
    │
    ├── Core/
    │   ├── PlanningCoordinator.php
    │   ├── DefaultQueryPlanner.php
    │   └── QueryPlanningResult.php
    │
    ├── Context/
    │   ├── PlanningContext.php
    │   ├── PlanningConfiguration.php
    │   ├── PlanningProfile.php
    │   └── PlanningMode.php
    │
    ├── Logical/
    │   ├── LogicalQueryPlan.php
    │   ├── LogicalPlanNode.php
    │   ├── LogicalPlanNodeId.php
    │   ├── LogicalScan.php
    │   ├── LogicalFilter.php
    │   ├── LogicalProject.php
    │   ├── LogicalJoin.php
    │   ├── LogicalAggregate.php
    │   ├── LogicalWindow.php
    │   ├── LogicalSort.php
    │   ├── LogicalLimit.php
    │   ├── LogicalSetOperation.php
    │   ├── LogicalMaterialize.php
    │   └── LogicalMutation.php
    │
    ├── Property/
    │   ├── LogicalProperties.php
    │   ├── PhysicalPropertySet.php
    │   ├── PhysicalOrdering.php
    │   ├── PhysicalDistribution.php
    │   ├── Rewindability.php
    │   ├── MaterializationProperty.php
    │   ├── ParallelismProperty.php
    │   ├── RequiredPhysicalProperties.php
    │   └── ProvidedPhysicalProperties.php
    │
    ├── Physical/
    │   ├── PhysicalQueryPlan.php
    │   ├── PhysicalPlanNode.php
    │   ├── PhysicalPlanNodeId.php
    │   ├── Scan/
    │   ├── Join/
    │   ├── Aggregate/
    │   ├── Window/
    │   ├── SetOperation/
    │   ├── Mutation/
    │   └── Materialization/
    │
    ├── AccessPath/
    │   ├── AccessPath.php
    │   ├── AccessPathDescriptor.php
    │   ├── SequentialAccessPath.php
    │   ├── IndexAccessPath.php
    │   └── ExtensionAccessPath.php
    │
    ├── Enforcer/
    │   ├── PhysicalEnforcer.php
    │   ├── SortEnforcer.php
    │   ├── MaterializeEnforcer.php
    │   ├── ExchangeEnforcer.php
    │   └── RepartitionEnforcer.php
    │
    ├── Cost/
    │   ├── EstimatedPhysicalCost.php
    │   ├── CostBreakdown.php
    │   ├── CostConfidence.php
    │   └── DefaultPhysicalCostComparator.php
    │
    ├── Search/
    │   ├── PlanningSearchStrategy.php
    │   ├── PlanningMemo.php
    │   ├── MemoGroup.php
    │   ├── MemoGroupId.php
    │   ├── MemoKey.php
    │   ├── BestPlanEntry.php
    │   ├── CandidateSet.php
    │   └── SearchQueue.php
    │
    ├── Rule/
    │   ├── PlanningRuleId.php
    │   ├── PlanningRuleRegistry.php
    │   └── PlanningRuleResult.php
    │
    ├── Resource/
    │   ├── PhysicalResourceRequirement.php
    │   ├── MemoryRequirement.php
    │   ├── TemporaryStorageRequirement.php
    │   ├── ParallelismRequirement.php
    │   └── NetworkRequirement.php
    │
    ├── Distribution/
    │   ├── DistributionPlanner.php
    │   ├── Exchange.php
    │   ├── GatherExchange.php
    │   ├── HashExchange.php
    │   └── BroadcastExchange.php
    │
    ├── Control/
    │   ├── ExecutionControlLevel.php
    │   └── PlanPortability.php
    │
    ├── Mapping/
    │   └── SemanticPlanIdentityMap.php
    │
    ├── Validation/
    │   ├── PlanningValidationContext.php
    │   └── PlanningValidationResult.php
    │
    ├── Budget/
    │   ├── PlanningBudget.php
    │   └── PlanningBudgetTracker.php
    │
    ├── Fingerprint/
    │   ├── LogicalPlanFingerprint.php
    │   ├── PhysicalPlanFingerprint.php
    │   └── PlannerVersion.php
    │
    ├── Trace/
    │   ├── PlanningTrace.php
    │   ├── PlanningTraceEntry.php
    │   └── PlanningDecision.php
    │
    ├── Diagnostic/
    │   ├── PlanningDiagnostic.php
    │   └── PlanningDiagnostics.php
    │
    ├── Registry/
    │   └── PhysicalStrategyRegistry.php
    │
    ├── Serialization/
    │   ├── LogicalPlanCodec.php
    │   └── PhysicalPlanCodec.php
    │
    └── Exception/
        ├── NoValidPlanException.php
        ├── PlanningBudgetExceededException.php
        ├── RequiredPhysicalPropertyException.php
        ├── UnsupportedPhysicalStrategyException.php
        ├── CapabilityPlanningException.php
        ├── PhysicalPlanValidationException.php
        ├── PlannerExtensionException.php
        ├── CostModelException.php
        └── ResourcePlanningException.php
```

---

# 293. Ejemplo completo

Consulta conceptual:

```sql
SELECT
    u.id,
    COUNT(o.id)
FROM users u
JOIN orders o
    ON o.user_id = u.id
WHERE u.status = ?
GROUP BY u.id
ORDER BY COUNT(o.id) DESC
LIMIT 10;
```

Después del Optimizer:

```text
OptimizedQueryArtifact
```

El Logical Planner puede producir:

```text
LogicalLimit(10)
       │
LogicalSort(count DESC)
       │
LogicalProject
       │
LogicalAggregate(group=u.id)
       │
LogicalJoin(INNER)
      /      \
Filter       Scan orders
  │
Scan users
```

---

# 294. Physical alternatives

Para `users`:

```text
SequentialScan(users)
IndexScan(users.status)
```

Para `orders`:

```text
SequentialScan(orders)
IndexScan(orders.user_id)
```

Para join:

```text
NestedLoop
HashJoin
MergeJoin
```

Para aggregate:

```text
HashAggregate
SortAggregate
```

Para ORDER BY:

```text
reuse existing ordering
explicit sort
top-N strategy
```

---

# 295. Candidate A

```text
TopN
 │
HashAggregate
 │
HashJoin
├── IndexScan(users.status)
└── SequentialScan(orders)
```

---

# 296. Candidate B

```text
Limit
 │
Sort
 │
SortAggregate
 │
MergeJoin
├── Sort(IndexScan(users.status))
└── IndexScan(orders.user_id)
```

---

# 297. Candidate C

```text
TopN
 │
HashAggregate
 │
NestedLoop
├── IndexScan(users.status)
└── IndexLookup(orders.user_id)
```

---

# 298. Cost evaluation

Supóngase:

```text
users after filter:
    ~500 rows

orders:
    ~100,000,000 rows

average orders/user:
    ~20
```

Candidate C puede ser atractivo porque:

```text
500 index lookups
×
~20 orders
```

puede evitar leer gran parte de `orders`.

Pero eso es una decisión de costo, no semántica.

---

# 299. Changed statistics

Si:

```text
users after filter:
    ~5,000,000
```

el Planner podría seleccionar otra estrategia.

Misma consulta.

Misma semántica.

Distinto physical plan.

---

# 300. Formal equations

La relación general será:

```text
LogicalQueryPlan
=
Lower(
    OptimizedQueryArtifact
)
```

---

La generación física:

```text
PhysicalAlternatives
=
Implement(
    LogicalQueryPlan,
    RequiredProperties,
    Capabilities
)
```

---

El costo:

```text
EstimatedCost(P)
=
CostModel(
    P,
    StatisticsSnapshot,
    QueryCostHintSet,
    ResourceProfile
)
```

---

La selección:

```text
SelectedPlan
=
argmin
    ValidPhysicalPlans
    CostAccordingToObjective(P)
```

sujeto a:

```text
SemanticRequirements
∧
CapabilityRequirements
∧
SecurityConstraints
∧
PhysicalPropertyRequirements
∧
ResourceHardConstraints
```

---

# 301. Safe planning formula

```text
Safe Planning
=
Semantic Preservation
+
Valid Logical Lowering
+
Capability Compliance
+
Physical Property Satisfaction
+
Barrier Compliance
+
Resource Compliance
+
Bounded Search
+
Deterministic Selection
```

---

# 302. Planning quality formula

```text
Planning Quality
≈
Logical Plan Quality
+
Statistics Quality
+
Cost Model Quality
+
Alternative Coverage
+
Physical Property Reasoning
+
Search Budget
```

---

# 303. Important consequence

```text
Planning Quality
≠
Query Correctness
```

Una planificación mediocre puede producir una query lenta.

No debe producir una query incorrecta.

---

# 304. Architectural invariants

## DB-PLAN-001

Optimizer será distinto de Planner.

## DB-PLAN-002

Planner será distinto de Compiler.

## DB-PLAN-003

Planner será distinto de Executor.

## DB-PLAN-004

LogicalQueryPlan será distinto de OptimizedQueryArtifact.

## DB-PLAN-005

LogicalQueryPlan será distinto de PhysicalQueryPlan.

## DB-PLAN-006

PhysicalQueryPlan será distinto de ExecutionPlan.

## DB-PLAN-007

ExecutionPlan será distinto de Compiled SQL.

## DB-PLAN-008

Planner no generará SQL.

## DB-PLAN-009

Planner no ejecutará statements.

## DB-PLAN-010

Planner no abrirá conexiones durante planificación pura.

## DB-PLAN-011

Planner no realizará schema introspection oculta.

## DB-PLAN-012

Planner no cargará live statistics desde reglas.

## DB-PLAN-013

Toda información externa llegará mediante artifacts/context explícitos.

## DB-PLAN-014

SemanticQueryArtifact conservará autoridad semántica.

## DB-PLAN-015

Planner no reinterpretará symbols.

## DB-PLAN-016

Planner no reinferirá query types.

## DB-PLAN-017

Planner no inventará constraints.

## DB-PLAN-018

Planner no cambiará output semantics.

## DB-PLAN-019

Logical operator será distinto de physical operator.

## DB-PLAN-020

LogicalJoin no implicará HashJoin.

## DB-PLAN-021

LogicalScan no implicará SequentialScan.

## DB-PLAN-022

LogicalAggregate no implicará HashAggregate.

## DB-PLAN-023

Required property será distinta de provided property.

## DB-PLAN-024

Enforcer será explícito.

## DB-PLAN-025

Enforcer tendrá costo.

## DB-PLAN-026

Sort semántico será distinto de physical enforcement sort.

## DB-PLAN-027

Physical properties serán estructuradas.

## DB-PLAN-028

Property satisfaction será explícita.

## DB-PLAN-029

Physical alternatives serán generadas antes de selección.

## DB-PLAN-030

Alternative generation será distinta de alternative selection.

## DB-PLAN-031

Physical strategies tendrán stable IDs.

## DB-PLAN-032

Physical strategies declararán capabilities.

## DB-PLAN-033

Physical strategies declararán required properties.

## DB-PLAN-034

Physical strategies declararán provided properties.

## DB-PLAN-035

AccessPath será distinto de Index.

## DB-PLAN-036

Index availability no obligará IndexScan.

## DB-PLAN-037

Join algorithm selection pertenecerá al Planner.

## DB-PLAN-038

Aggregate algorithm selection pertenecerá al Planner.

## DB-PLAN-039

Window physical strategy selection pertenecerá al Planner.

## DB-PLAN-040

Set physical strategy selection pertenecerá al Planner.

## DB-PLAN-041

DML tendrá planificación física explícita.

## DB-PLAN-042

DML planning preservará mutation semantics.

## DB-PLAN-043

Materialization decision será explícita.

## DB-PLAN-044

Materialization requirements serán respetados.

## DB-PLAN-045

Recursive CTE tendrá estrategia especializada.

## DB-PLAN-046

Correlation no implicará ejecución ingenua por fila.

## DB-PLAN-047

Physical property enforcement podrá generar alternativas.

## DB-PLAN-048

Property enforcement será capability-aware.

## DB-PLAN-049

Physical cost será distinto de semantic fact.

## DB-PLAN-050

Physical cost podrá ser multidimensional.

## DB-PLAN-051

Startup cost será distinto de total cost.

## DB-PLAN-052

Cost comparison respetará OptimizationObjective.

## DB-PLAN-053

OptimizationObjective no cambiará query semantics.

## DB-PLAN-054

Cost uncertainty será representable.

## DB-PLAN-055

Tie-breaking será determinista.

## DB-PLAN-056

Planner no utilizará randomness implícito.

## DB-PLAN-057

Planning search será bounded.

## DB-PLAN-058

PlanningMemo será operation-scoped.

## DB-PLAN-059

PlanningMemo será distinto de Query Deduplication.

## DB-PLAN-060

Memo keys podrán incluir required properties.

## DB-PLAN-061

El mejor plan dependerá de required properties.

## DB-PLAN-062

Branch-and-bound sólo podará cuando sea seguro según el modelo.

## DB-PLAN-063

Low-confidence estimates reducirán pruning agresivo.

## DB-PLAN-064

PlanningBudget será explícito.

## DB-PLAN-065

Budget exhaustion nunca devolverá un plan parcialmente inválido.

## DB-PLAN-066

Fallback plan deberá preservar semantics.

## DB-PLAN-067

Correctness tendrá prioridad sobre cost.

## DB-PLAN-068

Hard constraints se aplicarán antes de cost comparison.

## DB-PLAN-069

Plan inválido nunca ganará por ser barato.

## DB-PLAN-070

Resource planning será explícito.

## DB-PLAN-071

Resource estimate será distinto de runtime reservation.

## DB-PLAN-072

Spill será estrategia explícita.

## DB-PLAN-073

Parallelism planning será capability-aware.

## DB-PLAN-074

Parallel query planning será distinto de Quantum/Concurrency.

## DB-PLAN-075

Distributed planning será opcional.

## DB-PLAN-076

Exchange será physical operator.

## DB-PLAN-077

Capability checks reemplazarán vendor conditionals.

## DB-PLAN-078

Logical plan será platform-neutral cuando sea posible.

## DB-PLAN-079

Physical plan podrá ser platform-aware.

## DB-PLAN-080

Vendor-specific strategies usarán extension/capability contracts.

## DB-PLAN-081

Compiler no replanificará.

## DB-PLAN-082

Compiler no cambiará physical strategy silenciosamente.

## DB-PLAN-083

Planner reconocerá límites de control sobre native DB planners.

## DB-PLAN-084

VoltStack no fingirá controlar algoritmos internos no controlables.

## DB-PLAN-085

ExecutionControlLevel será explícito.

## DB-PLAN-086

PlanningMode será explícito.

## DB-PLAN-087

Native DB planning será distinto de framework-controlled planning.

## DB-PLAN-088

Semantic IDs serán distintos de plan node IDs.

## DB-PLAN-089

LogicalPlanNodeId será distinto de PhysicalPlanNodeId.

## DB-PLAN-090

Mappings semánticos se conservarán para diagnostics.

## DB-PLAN-091

Public output contract se preservará.

## DB-PLAN-092

ParameterIds se preservarán.

## DB-PLAN-093

Planner genérico no inspeccionará runtime bindings.

## DB-PLAN-094

Placeholder generation pertenecerá al Compiler.

## DB-PLAN-095

Planning barriers serán respetadas.

## DB-PLAN-096

Raw barriers limitarán physical planning cuando corresponda.

## DB-PLAN-097

Security barriers no podrán eliminarse por costo.

## DB-PLAN-098

Volatile evaluation semantics serán preservadas.

## DB-PLAN-099

Locking semantics serán preservadas.

## DB-PLAN-100

Mutation barriers serán preservadas.

## DB-PLAN-101

Correlation dependencies serán preservadas.

## DB-PLAN-102

Planner extensions tendrán stable IDs.

## DB-PLAN-103

Planner extensions declararán capabilities.

## DB-PLAN-104

Planner extensions declararán properties.

## DB-PLAN-105

Planner extensions declararán cost integration.

## DB-PLAN-106

Planner extensions declararán compiler support.

## DB-PLAN-107

PlanningRule será distinta de OptimizationRule.

## DB-PLAN-108

PlanningRule registration order no definirá execution order.

## DB-PLAN-109

Planning rule registry se congelará después de bootstrap.

## DB-PLAN-110

Planning traces serán bounded.

## DB-PLAN-111

Explain distinguirá generated/rejected/selected alternatives.

## DB-PLAN-112

Plan fingerprints serán distintos de semantic fingerprints.

## DB-PLAN-113

Statistics changes no cambiarán SemanticFingerprint.

## DB-PLAN-114

Statistics changes podrán cambiar PhysicalPlanFingerprint.

## DB-PLAN-115

Capability changes podrán invalidar physical plans.

## DB-PLAN-116

Schema changes podrán invalidar access paths.

## DB-PLAN-117

Plan caches serán versionadas.

## DB-PLAN-118

Unknown serialized strategies no tendrán silent fallback.

## DB-PLAN-119

Invalid cached plan requerirá replanificación.

## DB-PLAN-120

Replanning será distinto de execution retry.

## DB-PLAN-121

Adaptive planning será explícito.

## DB-PLAN-122

Adaptive planning no cambiará query semantics.

## DB-PLAN-123

Planner será deterministic por defecto.

## DB-PLAN-124

Final selected plan pasará validación.

## DB-PLAN-125

Validator no reparará silenciosamente planes inválidos.

## DB-PLAN-126

No valid plan producirá error explícito.

## DB-PLAN-127

Unsupported exact semantics no tendrá approximate fallback.

## DB-PLAN-128

Planner telemetry distinguirá intent físico de actual DB execution plan.

## DB-PLAN-129

EXPLAIN no será dependencia del planning puro.

## DB-PLAN-130

Planning podrá probarse offline mediante snapshots.

## DB-PLAN-131

Physical strategies tendrán conformance tests.

## DB-PLAN-132

Shared planner registries serán inmutables.

## DB-PLAN-133

Planning state será operation-scoped.

## DB-PLAN-134

No habrá current plan global.

## DB-PLAN-135

No habrá current memo global.

## DB-PLAN-136

Queries concurrentes no compartirán mutable planning state.

## DB-PLAN-137

Final plan artifacts serán inmutables.

## DB-PLAN-138

Planner no realizará authorization decisions.

## DB-PLAN-139

Security policies llegarán ya materializadas como constraints/barriers.

## DB-PLAN-140

Temporary data security podrá restringir physical strategies.

## DB-PLAN-141

Database core no dependerá directamente de Multitenancy.

## DB-PLAN-142

Tenant planning information se integrará mediante contratos.

## DB-PLAN-143

Tenant-specific statistics podrán cambiar physical plan.

## DB-PLAN-144

Tenant plan caches estarán correctamente aisladas.

## DB-PLAN-145

ORM no tendrá Planner separado.

## DB-PLAN-146

Active Record no tendrá Planner separado.

## DB-PLAN-147

Repository no tendrá Planner separado.

## DB-PLAN-148

EntityManager no elegirá physical strategies.

## DB-PLAN-149

Executor no realizará logical optimization implícita.

## DB-PLAN-150

Planner será persistent-runtime safe.

---

# 305. Anti-patterns

## 305.1 Planner generando SQL

```php
return 'SELECT ...';
```

**Rechazado.**

---

## 305.2 Compiler eligiendo HashJoin

**Rechazado.**

---

## 305.3 Executor reordenando joins

**Rechazado.**

---

## 305.4 Planner haciendo `SELECT COUNT(*)`

para obtener estadísticas.

**Rechazado.**

---

## 305.5 Physical strategy basada en vendor conditional

```php
if ($driver === 'pgsql') {
    ...
}
```

**Rechazado en el core.**

---

## 305.6 `LogicalJoin === HashJoin`

**Rechazado.**

---

## 305.7 Ignorar required ordering

porque un plan es más barato.

**Rechazado.**

---

## 305.8 Ignorar security barrier

porque aumenta costo.

**Rechazado.**

---

## 305.9 Elegir un plan inválido porque tiene menor score

**Rechazado.**

---

## 305.10 Memo global entre requests

**Rechazado.**

---

## 305.11 Current tenant en singleton Planner

**Rechazado.**

---

## 305.12 Planificación ilimitada

**Rechazado.**

---

## 305.13 Runtime bindings usados implícitamente para planificar

**Rechazado.**

---

## 305.14 Asumir que VoltStack controla el planner interno del DBMS

**Rechazado.**

---

## 305.15 Tratar un native optimizer hint como garantía

**Rechazado salvo capability/contrato que realmente la garantice.**

---

## 305.16 Plan cache sin capabilities/versioning

**Rechazado.**

---

## 305.17 Convertir estrategia desconocida a SQL raw

**Rechazado.**

---

## 305.18 Reutilizar un physical plan de otro tenant sin verificar aislamiento

**Rechazado.**

---

# 306. Decisión arquitectónica

VoltStack utilizará una arquitectura de Planner dividida conceptualmente en:

```text
Logical Planning
      │
      ▼
Physical Planning
      │
      ▼
Execution Planning
```

donde cada nivel posee responsabilidades distintas.

La regla será:

```text
LogicalQueryPlan
    =
execution-independent logical operator structure

PhysicalQueryPlan
    =
selected physical implementations and properties

ExecutionPlan
    =
final executable/compilable execution artifact
```

---

# 307. Frontera Optimizer → Planner

La frontera formal será:

```text
OptimizedQueryArtifact
        │
        ▼
Query Planner
```

El Optimizer entrega:

```text
a better equivalent logical query
```

El Planner determina:

```text
how that query can be executed
```

---

# 308. Frontera Planner → Compiler

```text
Physical / Execution Plan
        │
        ▼
SQL Compiler
```

El Planner decide estrategia.

El Compiler decide representación.

---

# 309. Regla consolidada

```text
Optimizer
    decides logical equivalence transformations.

Planner
    decides execution strategy.

Compiler
    decides target syntax/representation.

Executor
    performs the work.
```

---

# 310. Arquitectura final del Query Planner

```text
                  OptimizedQueryArtifact
                           │
                           ▼
                   PlanningCoordinator
                           │
                           ▼
                   LogicalQueryPlan
                           │
                           ▼
                Logical Properties
                           │
                           ▼
                Required Properties
                           │
                           ▼
                Planning Memo/Search
                           │
        ┌──────────────────┼───────────────────┐
        ▼                  ▼                   ▼
  Access Paths        Join Strategies     Other Strategies
        │                  │                   │
        └──────────────────┼───────────────────┘
                           ▼
                 Physical Alternatives
                           │
                           ▼
                 Property Enforcement
                           │
                           ▼
                     Cost Model
                           │
                           ▼
                    Plan Selection
                           │
                           ▼
                   Resource Planning
                           │
                           ▼
                   Plan Validation
                           │
                           ▼
                  PhysicalQueryPlan
                           │
                           ▼
                    Execution Plan
                           │
                           ▼
                      Compiler
                           │
                           ▼
                      Executor
```

---

# 311. Estado del bloque

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md             ← actual
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 312. Siguiente documento

```text
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
```

El siguiente documento formalizará específicamente el artifact:

```text
LogicalQueryPlan
```

incluyendo:

```text
LogicalPlanNode
LogicalScan
LogicalFilter
LogicalProject
LogicalJoin
LogicalAggregate
LogicalWindow
LogicalSort
LogicalLimit
LogicalSetOperation
LogicalCTE
LogicalRecursiveCTE
LogicalValues
LogicalInsert
LogicalUpdate
LogicalDelete
LogicalMutation
LogicalProperties
LogicalPlanGraph
Semantic-to-Logical Mapping
LogicalPlanFingerprint
```

y establecerá definitivamente la frontera:

```text
Optimized Semantic Query
        ↓
Logical Query Plan
        ↓
Physical Query Plan
```

sin permitir que el Logical Plan incorpore prematuramente decisiones como:

```text
HashJoin
IndexScan
ParallelScan
SortAlgorithm
MemoryAllocation
WorkerCount
```

que pertenecen exclusivamente a la planificación física.