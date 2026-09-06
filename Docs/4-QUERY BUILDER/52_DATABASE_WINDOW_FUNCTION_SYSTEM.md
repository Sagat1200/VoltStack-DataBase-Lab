# 52_DATABASE_WINDOW_FUNCTION_SYSTEM.md

# VoltStack Quantum Database
## Window Function System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 52 — Window Function System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Semantic Query  
**Versión:** 1.0

---

# 1. Propósito

`Window Function System` define la arquitectura mediante la cual VoltStack representa, valida, analiza, optimiza, planifica y posteriormente compila operaciones analíticas basadas en ventanas.

El sistema cubre:

```text
Window Functions
OVER
PARTITION BY
Window ORDER BY
Window Frames
ROWS
RANGE
GROUPS

UNBOUNDED PRECEDING
PRECEDING
CURRENT ROW
FOLLOWING
UNBOUNDED FOLLOWING

Named Windows
Window References

ROW_NUMBER
RANK
DENSE_RANK
NTILE

LAG
LEAD
FIRST_VALUE
LAST_VALUE
NTH_VALUE

Aggregate Window Functions
Custom Window Functions
```

sin convertir estas operaciones prematuramente en SQL.

La regla fundamental es:

```text
Window Semantics
≠
SQL Window Syntax
```

---

# 2. Regla maestra

```text
Window Function
≠
Aggregate Function

Window Partition
≠
GROUP BY

Window ORDER BY
≠
Query ORDER BY

Window Frame
≠
Query Pagination
```

Una función de ventana normalmente calcula información sobre un conjunto relacionado de filas sin reducir necesariamente la cardinalidad de entrada.

Ejemplo:

```text
Input rows: 100
       │
       ▼
SUM(amount) OVER (PARTITION BY customer_id)
       │
       ▼
Output rows: 100
```

Mientras una agregación tradicional:

```text
Input rows: 100
       │
       ▼
GROUP BY customer_id
       │
       ▼
Output rows: N groups
```

---

# 3. Posición arquitectónica

```text
Application
    │
    ▼
Select Query Builder
    │
    ▼
Window Builder
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
    ├── Symbol Resolution
    ├── Type Inference
    ├── Aggregate Analysis
    ├── Window Resolution
    ├── Partition Resolution
    ├── Ordering Resolution
    ├── Frame Analysis
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
SQL Compiler
    │
    ▼
Executor
```

---

# 4. Objetivos

El sistema deberá proporcionar:

- representación estructurada de window functions;
- `OVER`;
- particiones;
- ordenamiento interno de ventanas;
- frames;
- named windows;
- ranking functions;
- navigation functions;
- value functions;
- aggregate window functions;
- funciones extensibles;
- análisis de peer groups;
- análisis de frame semantics;
- resolución de tipos;
- resolución de nullability;
- capability requirements;
- portability;
- lineage;
- dependency tracking;
- semantic fingerprints;
- optimización;
- planificación;
- resource budgets;
- persistent-runtime safety.

---

# 5. No responsabilidades

Window Function System no deberá:

```text
generar SQL
abrir conexiones
ejecutar statements
leer schema mediante I/O oculto
seleccionar índices
elegir physical sort algorithms desde Builder
hidratar entidades
manejar UnitOfWork
administrar transacciones
convertir ventanas en GROUP BY arbitrariamente
```

---

# 6. Dependencias arquitectónicas

Este documento se integra especialmente con:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
```

---

# 7. Modelo conceptual

La representación fundamental será:

```text
WindowExpression
│
├── WindowFunction
│
└── WindowSpecification
    ├── PartitionSpecification
    ├── WindowOrdering
    └── WindowFrame
```

Conceptualmente:

```text
WindowExpression
=
Function
+
OVER(...)
```

---

# 8. WindowExpression

Modelo conceptual:

```php
final readonly class WindowExpression implements ExpressionNode
{
    public function __construct(
        public WindowFunctionExpression $function,
        public WindowReference $window,
        public WindowMetadata $metadata,
    ) {}
}
```

---

# 9. Window function identity

Cada función deberá tener identidad semántica.

Ejemplos:

```text
ROW_NUMBER
RANK
DENSE_RANK
NTILE

LAG
LEAD

FIRST_VALUE
LAST_VALUE
NTH_VALUE
```

No deberá dependerse exclusivamente de nombres SQL.

---

# 10. WindowFunctionId

Conceptualmente:

```php
final readonly class WindowFunctionId
{
    public function __construct(
        public string $namespace,
        public string $name,
    ) {}
}
```

Ejemplos:

```text
core:row_number
core:rank
core:dense_rank
core:ntile

core:lag
core:lead

core:first_value
core:last_value
core:nth_value
```

---

# 11. Window function categories

VoltStack distinguirá:

```text
RankingWindowFunction
NavigationWindowFunction
ValueWindowFunction
AggregateWindowFunction
ExtensionWindowFunction
```

---

# 12. Ranking functions

V1 deberá contemplar:

```text
ROW_NUMBER
RANK
DENSE_RANK
NTILE
```

---

# 13. Navigation functions

V1 deberá contemplar:

```text
LAG
LEAD
```

---

# 14. Value functions

V1 deberá contemplar:

```text
FIRST_VALUE
LAST_VALUE
NTH_VALUE
```

---

# 15. Aggregate window functions

Una función agregada podrá ser utilizada en contexto window cuando su descriptor lo permita.

Ejemplo:

```text
SUM(amount) OVER (...)
```

---

# 16. Aggregate Window ≠ Group Aggregate

Estas expresiones:

```text
SUM(amount)
```

y:

```text
SUM(amount) OVER (...)
```

comparten una función agregada conceptual, pero poseen semántica operacional distinta.

Por tanto:

```text
AggregateExpression
≠
AggregateWindowExpression
```

---

# 17. Cardinalidad

Una propiedad fundamental será:

```text
Window Evaluation
normally preserves input row cardinality
```

mientras:

```text
Grouping
may reduce cardinality
```

---

# 18. WindowSpecification

La ventana será una estructura first-class.

```php
final readonly class WindowSpecification
{
    public function __construct(
        public ?WindowReferenceName $baseWindow,
        public PartitionSpecification $partition,
        public WindowOrdering $ordering,
        public ?WindowFrame $frame,
    ) {}
}
```

---

# 19. OVER

`OVER` no deberá representarse como texto.

Incorrecto:

```php
new RawExpression(
    'SUM(amount) OVER (PARTITION BY customer_id)'
);
```

Representación esperada:

```text
WindowExpression
├── SUM(amount)
└── WindowSpecification
    └── PARTITION customer_id
```

---

# 20. Empty OVER

Debe poder representarse:

```text
COUNT(*) OVER ()
```

mediante una especificación vacía válida.

---

# 21. Empty window ≠ absent window

```text
SUM(amount)
```

es aggregate normal.

```text
SUM(amount) OVER ()
```

es window expression.

La presencia de `OVER`, incluso vacío, cambia la semántica.

---

# 22. PARTITION BY

Se representará mediante:

```text
PartitionSpecification
```

---

# 23. PartitionSpecification

```php
final readonly class PartitionSpecification
{
    /**
     * @param list<ExpressionNode> $expressions
     */
    public function __construct(
        public array $expressions,
    ) {}
}
```

---

# 24. Partition expressions

Podrán ser:

```text
column references
scalar expressions
function expressions
typed expressions
extension expressions
```

si son semánticamente válidas.

---

# 25. Partition ≠ grouping

Ejemplo:

```text
PARTITION BY customer_id
```

crea grupos lógicos para una window function, pero no colapsa las filas.

---

# 26. Example

Input:

```text
customer_id | amount
------------+-------
1           | 10
1           | 20
2           | 30
```

Con:

```text
SUM(amount) OVER (PARTITION BY customer_id)
```

resultado conceptual:

```text
customer_id | amount | total
------------+--------+------
1           | 10     | 30
1           | 20     | 30
2           | 30     | 30
```

---

# 27. Window ordering

`ORDER BY` dentro de `OVER` será:

```text
WindowOrdering
```

y será distinto de:

```text
QueryOrdering
AggregateOrdering
```

---

# 28. Tres órdenes distintos

VoltStack deberá distinguir:

```text
Query ORDER BY
Aggregate ORDER BY
Window ORDER BY
```

Ejemplo conceptual:

```text
STRING_AGG(name ORDER BY name)
OVER (
    PARTITION BY department
    ORDER BY created_at
)

ORDER BY department
```

contiene potencialmente tres órdenes semánticamente distintos.

---

# 29. WindowOrdering

```php
final readonly class WindowOrdering
{
    /**
     * @param list<WindowOrderingTerm> $terms
     */
    public function __construct(
        public array $terms,
    ) {}
}
```

---

# 30. WindowOrderingTerm

Podrá contener:

```text
expression
direction
null ordering
collation semantics
metadata
```

---

# 31. Ordering direction

```php
enum OrderingDirection
{
    case ASC;
    case DESC;
}
```

El mismo enum podrá reutilizarse cuando su semántica sea realmente común.

---

# 32. NULL ordering

Deberá modelarse semánticamente:

```text
DEFAULT
FIRST
LAST
```

sin codificar sintaxis vendor-specific.

---

# 33. Deterministic ordering

Una ventana ordenada no implica necesariamente un orden total.

Ejemplo:

```text
ORDER BY created_at
```

puede tener múltiples filas con el mismo valor.

---

# 34. Peer groups

Las filas equivalentes bajo el ordering de la ventana forman conceptualmente:

```text
PeerGroup
```

---

# 35. PeerGroup

Dos filas son peers cuando sus window ordering keys son equivalentes según las reglas semánticas aplicables.

---

# 36. Peer groups matter

Son relevantes especialmente para:

```text
RANK
DENSE_RANK
RANGE
GROUPS
```

---

# 37. ROW_NUMBER

`ROW_NUMBER()` asigna una posición por fila dentro de la partición según el ordering aplicable.

---

# 38. ROW_NUMBER without ordering

Si se permite una ventana sin ordering:

```text
ROW_NUMBER() OVER ()
```

el sistema deberá registrar que el resultado puede no poseer orden determinista entre filas.

---

# 39. Determinism metadata

Podrá existir:

```text
WindowDeterminism
```

con estados como:

```text
DETERMINISTIC
ORDER_DEPENDENT
NON_TOTAL_ORDER
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 40. RANK

`RANK()` deberá respetar peer groups y permitir gaps.

Ejemplo conceptual:

```text
value | rank
------+-----
100   | 1
100   | 1
90    | 3
```

---

# 41. DENSE_RANK

```text
value | dense_rank
------+-----------
100   | 1
100   | 1
90    | 2
```

---

# 42. ROW_NUMBER ≠ RANK ≠ DENSE_RANK

Estas funciones tendrán descriptors independientes.

No deberán modelarse mediante un único flag ambiguo.

---

# 43. NTILE

`NTILE(n)` divide conceptualmente una partición ordenada en buckets.

El argumento deberá ser estructurado y tipado.

---

# 44. NTILE argument

Deberá validarse semánticamente como un valor compatible con entero positivo según las reglas aplicables.

---

# 45. Runtime NTILE value

Si:

```text
NTILE(:buckets)
```

utiliza un parámetro runtime, Builder no conocerá necesariamente su valor.

Podrá derivar:

```text
Parameter type = integer-like
Required runtime constraint = > 0
```

sin inspeccionar el valor durante semantic analysis.

---

# 46. LAG

Modelo conceptual:

```text
LAG(
    expression,
    offset?,
    default?
)
OVER (...)
```

---

# 47. LEAD

Análogo:

```text
LEAD(
    expression,
    offset?,
    default?
)
OVER (...)
```

---

# 48. Offset

El offset será una expresión/argumento estructurado.

No un fragmento SQL.

---

# 49. Default argument

El valor default deberá participar en type inference.

Ejemplo:

```text
LAG(price, 1, 0)
```

requiere reconciliar:

```text
price type
default type
```

---

# 50. LAG/LEAD result type

Conceptualmente:

```text
ResultType
=
CommonCompatibleType(
    value expression,
    default expression
)
```

sujeto a reglas específicas.

---

# 51. LAG/LEAD nullability

Sin default:

```text
LAG(value)
```

puede producir NULL en boundaries aunque `value` sea NOT NULL.

Por tanto:

```text
Input Nullability
≠
Window Output Nullability
```

---

# 52. Boundary-induced nullability

Deberá existir información equivalente a:

```text
WindowBoundaryNullabilityEffect
```

---

# 53. FIRST_VALUE

`FIRST_VALUE(expression)` obtiene un valor determinado por el frame/window semantics.

---

# 54. LAST_VALUE

`LAST_VALUE(expression)` requiere especial atención porque su resultado depende fuertemente del frame.

---

# 55. NTH_VALUE

`NTH_VALUE(expression, n)` también depende del frame.

---

# 56. Window frame

El frame será representado mediante:

```text
WindowFrame
```

---

# 57. Frame units

V1 deberá contemplar:

```text
ROWS
RANGE
GROUPS
```

mediante:

```php
enum WindowFrameUnit
{
    case ROWS;
    case RANGE;
    case GROUPS;
}
```

---

# 58. WindowFrame

Conceptualmente:

```php
final readonly class WindowFrame
{
    public function __construct(
        public WindowFrameUnit $unit,
        public WindowFrameBoundary $start,
        public WindowFrameBoundary $end,
        public WindowFrameExclusion $exclusion,
    ) {}
}
```

---

# 59. Frame boundaries

Tipos fundamentales:

```text
UNBOUNDED PRECEDING
OFFSET PRECEDING
CURRENT ROW
OFFSET FOLLOWING
UNBOUNDED FOLLOWING
```

---

# 60. Boundary model

```php
interface WindowFrameBoundary
{
}
```

Implementaciones:

```text
UnboundedPrecedingBoundary
OffsetPrecedingBoundary
CurrentRowBoundary
OffsetFollowingBoundary
UnboundedFollowingBoundary
```

---

# 61. Explicit boundary objects

No deberá utilizarse:

```php
$frameStart = '5 PRECEDING';
```

como modelo interno.

---

# 62. Frame start/end legality

No toda combinación será válida.

Por ejemplo, el sistema deberá detectar límites lógicamente invertidos o incompatibles.

---

# 63. Frame boundary ordering

Conceptualmente puede establecerse:

```text
UNBOUNDED PRECEDING
        <
N PRECEDING
        <
CURRENT ROW
        <
N FOLLOWING
        <
UNBOUNDED FOLLOWING
```

aunque la validación exacta dependerá de la unidad y reglas aplicables.

---

# 64. Offset boundary

```php
final readonly class OffsetPrecedingBoundary
    implements WindowFrameBoundary
{
    public function __construct(
        public ExpressionNode $offset,
    ) {}
}
```

---

# 65. Offset validation

Semantic Analysis deberá determinar:

```text
offset type
offset constraints
platform requirements
```

---

# 66. No runtime value inspection

Si el offset es parámetro:

```text
ROWS BETWEEN :n PRECEDING AND CURRENT ROW
```

el Semantic Engine podrá exigir:

```text
:n integer-compatible
:n >= 0
```

sin depender del valor concreto.

---

# 67. ROWS semantics

`ROWS` opera conceptualmente sobre posiciones físicas/lógicas relativas dentro del ordering de la partición.

---

# 68. RANGE semantics

`RANGE` opera respecto a valores del ordering y peer/value ranges.

No deberá implementarse como alias de `ROWS`.

---

# 69. GROUPS semantics

`GROUPS` opera sobre peer groups.

No deberá implementarse como alias de `ROWS` ni `RANGE`.

---

# 70. Frame unit distinctions

```text
ROWS
≠
RANGE
≠
GROUPS
```

será un invariante central.

---

# 71. CURRENT ROW semantics

El significado de:

```text
CURRENT ROW
```

puede depender de la unidad del frame.

Especialmente:

```text
ROWS CURRENT ROW
```

y:

```text
RANGE CURRENT ROW
```

no deberán asumirse equivalentes.

---

# 72. Default frame

Si el desarrollador no declara frame, podrá existir un:

```text
ImplicitWindowFrame
```

determinado por semantic policy/capabilities.

---

# 73. Explicit ≠ implicit frame

VoltStack deberá conservar si el frame fue:

```text
EXPLICIT
IMPLICIT
```

porque puede ser importante para:

```text
diagnostics
portability
normalization
explain
```

---

# 74. Default frame resolution

La resolución de defaults deberá ocurrir mediante reglas semánticas conocidas.

No mediante SQL strings.

---

# 75. Frame exclusion

La arquitectura deberá ser extensible a:

```text
EXCLUDE CURRENT ROW
EXCLUDE GROUP
EXCLUDE TIES
EXCLUDE NO OTHERS
```

cuando la plataforma/capabilities lo permitan.

---

# 76. WindowFrameExclusion

```php
enum WindowFrameExclusion
{
    case DEFAULT;
    case CURRENT_ROW;
    case GROUP;
    case TIES;
    case NO_OTHERS;
}
```

---

# 77. Frame exclusion capability

Deberá ser capability-driven.

---

# 78. Frame semantics descriptor

Podrá existir:

```text
WindowFrameSemanticInfo
```

con:

```text
unit
resolved start
resolved end
exclusion
ordering requirements
peer semantics
capabilities
portability
```

---

# 79. Named windows

VoltStack deberá soportar ventanas nombradas.

Conceptualmente:

```text
WINDOW w AS (
    PARTITION BY customer_id
    ORDER BY created_at
)
```

---

# 80. WindowDefinition

```php
final readonly class WindowDefinition
{
    public function __construct(
        public WindowName $name,
        public WindowSpecification $specification,
    ) {}
}
```

---

# 81. Window reference

Una expression podrá referenciar:

```text
WindowReference(name)
```

en lugar de duplicar la especificación.

---

# 82. Named window identity

```text
WindowName
≠
WindowDefinitionId
```

El nombre pertenece al namespace visible.

La identidad semántica deberá ser independiente.

---

# 83. WindowDefinitionId

Cada definición resuelta tendrá:

```text
WindowDefinitionId
```

---

# 84. Window namespace

Semantic Analysis deberá resolver:

```text
window definitions
window references
scope
duplicate names
unknown names
shadowing rules
```

---

# 85. Window scope

Las named windows pertenecerán a un:

```text
WindowScope
```

---

# 86. WindowScopeId

Cada scope tendrá identidad explícita.

---

# 87. Window inheritance

La arquitectura podrá soportar:

```text
WINDOW w2 AS (
    w1
    ORDER BY ...
)
```

si la semántica/capabilities lo permiten.

---

# 88. Base window reference

`WindowSpecification` podrá contener:

```text
baseWindow
```

---

# 89. Window inheritance resolution

Semantic Analysis deberá combinar:

```text
base specification
local partition
local ordering
local frame
```

de acuerdo con reglas explícitas.

---

# 90. No arbitrary override

No deberá permitirse sobreescribir partes de una base window cuando la semántica del lenguaje lo prohíba.

---

# 91. ResolvedWindowSpecification

Después de Semantic Analysis deberá existir una representación completa:

```php
final readonly class ResolvedWindowSpecification
{
    public function __construct(
        public WindowDefinitionId $id,
        public ResolvedPartition $partition,
        public ResolvedWindowOrdering $ordering,
        public ResolvedWindowFrame $frame,
        public WindowSemanticProperties $properties,
    ) {}
}
```

---

# 92. Window chaining

Varias window expressions podrán compartir:

```text
partition
ordering
frame
```

total o parcialmente.

---

# 93. Example

```text
SUM(amount) OVER (
    PARTITION BY customer_id
    ORDER BY created_at
)

AVG(amount) OVER (
    PARTITION BY customer_id
    ORDER BY created_at
)
```

poseen potencialmente la misma window specification.

---

# 94. Builder behavior

Builder deberá preservar ambas expressions.

No deberá implementar physical deduplication.

---

# 95. Semantic equivalence

Semantic Analysis podrá determinar que ambas specifications son equivalentes.

---

# 96. Optimizer behavior

Optimizer podrá consolidar window groups cuando sea seguro.

---

# 97. WindowGroup

Conceptualmente:

```text
WindowGroup
=
Window Expressions
sharing compatible
partition/order/frame requirements
```

---

# 98. WindowGroup ≠ SQL WINDOW clause

Es un concepto semántico/planning.

---

# 99. Window evaluation dependencies

Algunas window groups deben ejecutarse antes que otras.

Por tanto podrá existir:

```text
WindowDependencyGraph
```

---

# 100. WindowDependencyGraph

```text
WG1
 │
 ▼
WG2
 │
 ▼
WG3
```

cuando outputs de una fase sean inputs permitidos de otra fase semántica.

---

# 101. Same-level nested windows

Una expresión como:

```text
LAG(
    ROW_NUMBER() OVER (...)
) OVER (...)
```

puede ser inválida dentro del mismo query evaluation level.

---

# 102. Window nesting rule

VoltStack deberá distinguir:

```text
same query level
different subquery level
different semantic window stage
```

antes de decidir legalidad.

---

# 103. Window over subquery output

Esto sí puede ser válido:

```text
Outer Window
    │
    ▼
Derived Relation
    │
    └── Inner Window
```

porque existe una nueva query relation.

---

# 104. Aggregate and window interaction

Ejemplo:

```text
SUM(COUNT(*)) OVER (...)
```

puede tener una semántica válida cuando:

```text
COUNT(*)
```

es aggregate output de grouping y:

```text
SUM(...) OVER (...)
```

opera posteriormente sobre esos grupos.

---

# 105. Evaluation layers

Conceptualmente:

```text
Input Rows
    │
    ▼
WHERE
    │
    ▼
Grouping / Aggregate
    │
    ▼
HAVING
    │
    ▼
Window Evaluation
    │
    ▼
Projection
    │
    ▼
Query DISTINCT
    │
    ▼
Query Ordering
    │
    ▼
Pagination
```

La posición exacta de alias/projection resolution será definida por el Semantic Engine, no por textual SQL interpretation.

---

# 106. Window cannot generally feed WHERE

Una window expression no deberá estar disponible en un pre-window `WHERE` del mismo query level.

---

# 107. Window cannot generally feed GROUP BY

Tampoco deberá utilizarse arbitrariamente como grouping key del mismo evaluation level.

---

# 108. Window cannot generally feed HAVING

La legalidad deberá definirse explícitamente.

No se deberá asumir soporte sólo porque alguna plataforma acepte extensiones específicas.

---

# 109. Post-window filtering

Si VoltStack desea soportar una semántica tipo:

```text
QUALIFY
```

deberá modelarse como fase semántica explícita.

No como `havingRaw()`.

---

# 110. Future QUALIFY system

La arquitectura deberá permitir:

```text
WindowResultPredicate
```

sin obligar a que la plataforma tenga sintaxis `QUALIFY`.

---

# 111. QUALIFY ≠ WHERE

```text
PreWindowPredicate
≠
PostWindowPredicate
```

---

# 112. Projection aliases

Window expressions podrán producir aliases:

```text
ROW_NUMBER() OVER (...) AS row_number
```

pero:

```text
ProjectionAlias
≠
WindowExpressionId
```

---

# 113. WindowExpressionId

Cada instancia podrá poseer:

```text
WindowExpressionId
```

---

# 114. Semantic identity

Dos expresiones estructuralmente equivalentes pueden poseer IDs distintos pero ser declaradas:

```text
SemanticallyEquivalent
```

---

# 115. Window function descriptors

Conceptualmente:

```php
final readonly class WindowFunctionDescriptor
{
    public function __construct(
        public WindowFunctionId $id,
        public WindowFunctionCategory $category,
        public FunctionArity $arity,
        public WindowArgumentTypeRule $argumentRule,
        public WindowResultTypeRule $resultRule,
        public WindowNullabilityRule $nullabilityRule,
        public WindowRequirementSet $requirements,
        public Volatility $volatility,
        public Determinism $determinism,
        public CapabilityRequirementSet $capabilities,
        public Portability $portability,
    ) {}
}
```

---

# 116. Function requirements

Un descriptor podrá declarar:

```text
ORDER_REQUIRED
ORDER_OPTIONAL
FRAME_ALLOWED
FRAME_REQUIRED
FRAME_IGNORED
PARTITION_ALLOWED
DISTINCT_ALLOWED
NULL_TREATMENT_SUPPORTED
```

---

# 117. ROW_NUMBER requirements

Podrá declarar:

```text
partition = optional
ordering = semantic-policy dependent
frame = ignored/not applicable
```

---

# 118. RANK requirements

Peer semantics dependerán del ordering.

---

# 119. LAG/LEAD requirements

Podrán declarar que el frame no determina el offset lookup de la misma forma que value frame functions.

Estas diferencias deberán vivir en descriptors/rules.

---

# 120. FIRST_VALUE/LAST_VALUE

Deberán declarar dependencia explícita del frame.

---

# 121. Function-specific semantics

No deberá existir una regla genérica:

```text
all window functions use frame identically
```

porque es falsa.

---

# 122. Type inference

Window Semantic Analysis deberá integrarse con:

```text
Query Type System
```

---

# 123. ROW_NUMBER result type

Será un tipo integral semántico apropiado.

No deberá codificarse simplemente como:

```text
PHP int
```

---

# 124. RANK result type

Igualmente será integer-like semántico.

---

# 125. NTILE result type

Será integer-like.

---

# 126. LAG/LEAD result type

Derivará principalmente de:

```text
value argument
+
default argument
```

---

# 127. FIRST_VALUE result type

Derivará de la expresión de valor.

---

# 128. Aggregate window type

Deberá reutilizar las reglas semánticas del Aggregate Function Descriptor cuando corresponda.

---

# 129. No duplicate SUM type rules

No deberá existir:

```text
SumAggregateTypeResolver
```

y otro independiente:

```text
SumWindowTypeResolver
```

si ambos pueden compartir la misma regla base.

---

# 130. Window context may modify nullability

Aunque el tipo base se comparta, la nullability podrá cambiar por:

```text
frame emptiness
partition boundaries
navigation boundaries
default arguments
```

---

# 131. Frame emptiness

Determinados frames pueden ser vacíos para ciertas filas.

Por tanto:

```text
AggregateWindow SUM
```

puede producir NULL aun si la input expression es NOT NULL.

---

# 132. WindowNullabilityInfo

Conceptualmente:

```php
final readonly class WindowNullabilityInfo
{
    public function __construct(
        public Nullability $result,
        public WindowNullabilityReasonSet $reasons,
    ) {}
}
```

---

# 133. Null treatment

La arquitectura deberá permitir funciones/features como:

```text
RESPECT NULLS
IGNORE NULLS
```

cuando existan capabilities.

---

# 134. NullTreatment

```php
enum NullTreatment
{
    case DEFAULT;
    case RESPECT_NULLS;
    case IGNORE_NULLS;
}
```

---

# 135. No vendor assumption

El soporte de `IGNORE NULLS` no deberá asumirse universal.

---

# 136. FROM LAST/FROM FIRST

Features avanzadas de ciertas value functions podrán modelarse mediante descriptors/capabilities y no strings raw.

---

# 137. WindowSemanticInfo

Conceptualmente:

```php
final readonly class WindowSemanticInfo
{
    public function __construct(
        public WindowExpressionId $id,
        public WindowFunctionId $function,
        public WindowScopeId $scope,
        public ResolvedWindowSpecification $window,
        public QueryType $resultType,
        public Nullability $nullability,
        public WindowDeterminism $determinism,
        public DependencySet $dependencies,
        public LineageSet $lineage,
        public CapabilityRequirementSet $capabilities,
        public Portability $portability,
    ) {}
}
```

---

# 138. WindowSemanticTable

Los resultados semánticos vivirán en:

```text
WindowSemanticTable
```

y no serán escritos dentro del AST.

---

# 139. AST remains structural

```text
WindowExpressionNode
```

describe estructura.

```text
WindowSemanticInfo
```

describe significado resuelto.

---

# 140. Window scope table

Podrá existir:

```text
WindowScopeTable
```

con información de:

```text
visible windows
definitions
references
inheritance
dependencies
evaluation stage
```

---

# 141. WindowDefinitionTable

Mantendrá:

```text
WindowDefinitionId
WindowName
ResolvedWindowSpecification
ScopeId
Provenance
```

---

# 142. Semantic graph integration

`SemanticQueryGraph` deberá representar:

```text
WindowExpression
WindowFunction
WindowScope
WindowDefinition
PartitionExpression
WindowOrdering
WindowFrame
PeerSemantics
WindowOutput
```

---

# 143. Graph example

```text
WindowExpression W1
│
├── USES_FUNCTION → SUM
│
├── PARTITIONS_BY → customer_id
│
├── ORDERS_BY → created_at
│
├── USES_FRAME → F1
│
├── DEPENDS_ON → amount
│
└── PRODUCES → OutputColumn C4
```

---

# 144. Suggested graph edges

```text
USES_WINDOW
REFERENCES_WINDOW
INHERITS_WINDOW
PARTITIONS_BY
ORDERS_BY
USES_FRAME
HAS_FRAME_START
HAS_FRAME_END
USES_FUNCTION
WINDOWS_OVER
DEPENDS_ON
DERIVES_FROM
PRODUCES
REQUIRES_CAPABILITY
```

---

# 145. Lineage

Ejemplo:

```text
LAG(price)
```

produce lineage hacia:

```text
price
```

---

# 146. Ranking lineage

`ROW_NUMBER()` no deriva de una value column concreta.

Pero depende de:

```text
partition membership
ordering
input row identity/cardinality
```

---

# 147. Ranking dependency

Por tanto deberá distinguirse:

```text
Value Lineage
≠
Ordering Dependency
≠
Partition Dependency
```

---

# 148. RANK lineage

`RANK()` depende especialmente de:

```text
ordering expressions
peer semantics
partition
```

---

# 149. Aggregate window lineage

```text
SUM(amount) OVER (...)
```

tendrá:

```text
value lineage → amount
partition dependencies
ordering dependencies
frame dependencies
```

---

# 150. Dependency precision

El sistema deberá evitar tratar todas estas relaciones como una sola lista genérica.

---

# 151. Constraint Analysis integration

Constraint Analysis podrá derivar facts limitados sobre window outputs.

---

# 152. ROW_NUMBER facts

Dentro de cada partition, puede existir evidencia como:

```text
ROW_NUMBER >= 1
```

---

# 153. RANK facts

Igualmente:

```text
RANK >= 1
DENSE_RANK >= 1
```

---

# 154. NTILE facts

Con un bucket count válido:

```text
NTILE >= 1
```

y potencialmente:

```text
NTILE <= bucket_count
```

como fact condicional.

---

# 155. No false uniqueness

`ROW_NUMBER` puede ser único dentro de una partición, pero no necesariamente globalmente.

---

# 156. Scope-specific uniqueness

Podrá expresarse:

```text
UniqueWithinPartitionFact
```

---

# 157. LAG/LEAD facts

No deberá inferirse que:

```text
LAG(x) = x
```

ni que preserva non-nullability.

---

# 158. Frame facts

Los frames podrán producir facts estructurales, pero Constraint Analysis no deberá intentar enumerar filas runtime.

---

# 159. Optimizer responsibilities

Optimizer podrá considerar:

```text
window specification deduplication
named-window normalization
window group consolidation
compatible sort reuse
partition expression simplification
ordering simplification
constant partition elimination
redundant window elimination
safe frame normalization
predicate interactions
window stage consolidation
```

---

# 160. Optimizer shall not

```text
change ROW_NUMBER ordering semantics
remove ordering required by RANK
change frame boundaries unsafely
merge incompatible window groups
replace RANGE with ROWS
replace GROUPS with ROWS
```

---

# 161. Window specification deduplication

Dos specifications semánticamente iguales podrán compartir información/planning.

---

# 162. Structural equality ≠ semantic equality

Ejemplo:

```text
ORDER BY users.id
```

y una expresión distinta con mismo texto superficial no deberán considerarse iguales por strings.

---

# 163. Sort reuse

Si varias window groups requieren:

```text
PARTITION BY customer_id
ORDER BY created_at
```

Planner podrá intentar reutilizar ordering.

---

# 164. Prefix-compatible ordering

Ejemplo:

```text
W1:
PARTITION BY customer_id
ORDER BY created_at

W2:
PARTITION BY customer_id
ORDER BY created_at, id
```

puede existir una relación de compatibilidad física.

La decisión pertenece al Planner.

---

# 165. Window planning

Planner podrá producir operaciones conceptuales:

```text
Partition
Sort
WindowEvaluate
WindowGroupEvaluate
FrameEvaluate
```

---

# 166. Example logical plan

```text
Projection
    │
WindowEvaluate WG2
    │
WindowEvaluate WG1
    │
Sort
    │
Input Relation
```

---

# 167. Example optimized planning

Varias expressions compatibles:

```text
ROW_NUMBER()
RANK()
SUM(amount)
```

podrían evaluarse dentro de un mismo compatible window group.

---

# 168. Physical window operator

Conceptualmente:

```text
PhysicalWindowOperator
```

podrá contener:

```text
partition keys
ordering keys
frame requirements
function evaluations
memory requirements
spill policy
```

---

# 169. SQL database target

En la mayoría de drivers SQL, VoltStack compilará la semántica de ventana y dejará al database optimizer parte de la ejecución física.

---

# 170. Future execution engines

La separación permitirá futuros:

```text
distributed query engines
in-memory execution
analytical adapters
remote query engines
```

sin rediseñar Query Model.

---

# 171. Compiler responsibilities

Compiler deberá convertir:

```text
WindowFunctionId
ResolvedWindowSpecification
WindowFrame
WindowOrdering
PartitionSpecification
NamedWindow
```

a sintaxis del target.

---

# 172. Compiler shall not

Compiler no deberá resolver:

```text
unknown named windows
window inheritance
frame legality
result types
nullability
semantic ordering
window nesting legality
```

---

# 173. Capability model

Capabilities propuestas:

```text
QUERY.WINDOW.BASIC

QUERY.WINDOW.PARTITION
QUERY.WINDOW.ORDERING
QUERY.WINDOW.NAMED
QUERY.WINDOW.INHERITANCE

QUERY.WINDOW.FRAME.ROWS
QUERY.WINDOW.FRAME.RANGE
QUERY.WINDOW.FRAME.GROUPS
QUERY.WINDOW.FRAME.EXCLUSION
QUERY.WINDOW.FRAME.OFFSET

QUERY.WINDOW.ROW_NUMBER
QUERY.WINDOW.RANK
QUERY.WINDOW.DENSE_RANK
QUERY.WINDOW.NTILE

QUERY.WINDOW.LAG
QUERY.WINDOW.LEAD
QUERY.WINDOW.FIRST_VALUE
QUERY.WINDOW.LAST_VALUE
QUERY.WINDOW.NTH_VALUE

QUERY.WINDOW.AGGREGATE
QUERY.WINDOW.NULL_TREATMENT
QUERY.WINDOW.POST_FILTER
```

---

# 174. Capability status

Cada capability podrá ser:

```text
NATIVE
EMULATABLE
RESTRICTED
UNSUPPORTED
```

---

# 175. Capability restrictions

Una plataforma podría soportar:

```text
RANGE
```

pero sólo bajo determinados tipos de ordering/frame offsets.

Por tanto una capability podrá incluir restricciones estructuradas.

---

# 176. CapabilityDescriptor

Ejemplo conceptual:

```text
WindowRangeCapability
├── supported
├── allowedOrderingArity
├── allowedOrderingTypes
├── offsetSupport
├── dynamicOffsetSupport
└── exclusions
```

---

# 177. No boolean-only capabilities when insufficient

Un simple:

```php
supportsRange(): bool
```

puede ser insuficiente cuando existen restricciones internas.

---

# 178. Portability

Cada window expression deberá poder clasificarse:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
UNKNOWN
```

---

# 179. Raw window expression

Ejemplo:

```php
selectRaw(
    'vendor_window_function(...) OVER (...)'
);
```

será un raw barrier salvo descriptor especializado.

---

# 180. Raw barrier effects

Puede limitar:

```text
type inference
nullability analysis
lineage
window grouping
sort reuse
frame validation
portability
optimization
```

---

# 181. Extension model

VoltStack deberá permitir custom window functions.

---

# 182. WindowFunctionRegistry

Propuesta:

```text
WindowFunctionRegistry
```

---

# 183. Registry lifecycle

```text
Bootstrap
    │
    ▼
Register Core Functions
    │
    ▼
Register Extensions
    │
    ▼
Validate Descriptors
    │
    ▼
Freeze Registry
```

---

# 184. Registry persistent safety

Después de bootstrap:

```text
WindowFunctionRegistry
```

deberá ser immutable/frozen.

---

# 185. Extension requirements

Una extensión deberá declarar:

```text
function identity
category
arity
argument rules
result type rule
nullability rule
partition requirements
ordering requirements
frame behavior
peer behavior
determinism
volatility
capabilities
portability
fingerprint version
```

---

# 186. No compiler-only window extension

Registrar únicamente:

```text
function name → SQL string
```

no será suficiente para una semantic extension completa.

---

# 187. Builder API

Ejemplo deseado:

```php
$query = DB::table('orders')
    ->select('id')
    ->select('customer_id')
    ->select('total')
    ->selectWindow(
        DB::windowFunction()
            ->rowNumber()
            ->over(
                DB::window()
                    ->partitionBy('customer_id')
                    ->orderBy('created_at')
            ),
        as: 'sequence'
    );
```

---

# 188. Fluent shortcut

Podrán existir shortcuts:

```php
$query->selectRowNumber(
    partitionBy: ['customer_id'],
    orderBy: ['created_at'],
    as: 'sequence',
);
```

pero deberán producir exactamente el mismo Query Model.

---

# 189. Aggregate window API

Ejemplo:

```php
$query->selectWindow(
    DB::aggregate()
        ->sum('total')
        ->over(
            DB::window()
                ->partitionBy('customer_id')
        ),
    as: 'customer_total',
);
```

---

# 190. Named window API

Ejemplo:

```php
$query
    ->window(
        'customer_orders',
        DB::window()
            ->partitionBy('customer_id')
            ->orderBy('created_at')
    )
    ->selectWindow(
        DB::windowFunction()
            ->rowNumber()
            ->overWindow('customer_orders'),
        as: 'sequence'
    );
```

---

# 191. Frame API

Ejemplo:

```php
DB::window()
    ->partitionBy('customer_id')
    ->orderBy('created_at')
    ->rowsBetween(
        WindowBoundary::unboundedPreceding(),
        WindowBoundary::currentRow(),
    );
```

---

# 192. RANGE API

```php
DB::window()
    ->orderBy('score')
    ->rangeBetween(
        WindowBoundary::preceding(10),
        WindowBoundary::currentRow(),
    );
```

---

# 193. GROUPS API

```php
DB::window()
    ->orderBy('score')
    ->groupsBetween(
        WindowBoundary::preceding(1),
        WindowBoundary::following(1),
    );
```

---

# 194. No SQL boundary strings

Evitar:

```php
->frame('ROWS BETWEEN 5 PRECEDING AND CURRENT ROW')
```

como API estructurada principal.

---

# 195. Dynamic construction

Las closures podrán utilizarse para DX:

```php
->over(function (WindowBuilder $window) {
    $window
        ->partitionBy('customer_id')
        ->orderBy('created_at');
})
```

---

# 196. Closure lifetime

La closure deberá desaparecer al finalizar construcción.

Nunca deberá entrar en:

```text
Query Model
AST
Semantic Artifact
cache
fingerprint
serialized plan
```

---

# 197. Builder state

Propuesta:

```php
final class WindowBuilderState
{
    public array $partitionExpressions = [];
    public array $orderingTerms = [];
    public ?WindowFrame $frame = null;
    public ?WindowReferenceName $baseWindow = null;
}
```

La implementación final deberá preferir colecciones tipadas.

---

# 198. Builder finalization

```text
Mutable WindowBuilder
        │
        ▼
WindowBuilderFinalizer
        │
        ▼
Immutable WindowSpecification
```

---

# 199. Finalizer responsibilities

Puede:

```text
freeze partition expressions
freeze ordering
freeze frame
validate basic builder lifecycle
normalize builder-only wrappers
```

---

# 200. Finalizer non-responsibilities

No deberá:

```text
resolve columns
resolve types
resolve named windows
validate platform capabilities
infer default frames semantically
optimize sorts
compile SQL
execute
```

---

# 201. Validation

Structural Validation deberá detectar:

```text
malformed window function
invalid argument count known structurally
malformed frame
invalid boundary structure
duplicate named window declarations
empty invalid constructs
invalid builder lifecycle
```

---

# 202. Semantic validation

Semantic Analysis deberá detectar:

```text
unknown window reference
invalid inheritance
invalid function arguments
invalid partition expressions
invalid ordering expressions
invalid frame unit
invalid boundary ordering
missing required ordering
unsupported frame/function combination
illegal nested windows
illegal evaluation-stage references
```

---

# 203. Frame validation example

```text
ROWS BETWEEN
    5 FOLLOWING
AND
    2 PRECEDING
```

deberá rechazarse.

---

# 204. Ranking validation

Si una function descriptor requiere ordering para una semántica portable estricta, deberá diagnosticarse su ausencia.

---

# 205. Navigation validation

Offsets deberán cumplir sus type/runtime constraints.

---

# 206. Named window cycle

Si la arquitectura permite inheritance, esto deberá rechazarse:

```text
w1 → w2
w2 → w1
```

---

# 207. Window dependency cycle

Los cycles ilegales deberán detectarse antes de Optimization.

---

# 208. Semantic result

Podrá existir:

```php
final readonly class WindowAnalysisResult
{
    public function __construct(
        public WindowSemanticTable $windows,
        public WindowDefinitionTable $definitions,
        public WindowScopeTable $scopes,
        public WindowDependencyGraph $dependencies,
        public CapabilityRequirementSet $capabilities,
        public DiagnosticCollection $diagnostics,
        public SemanticFingerprint $fingerprint,
    ) {}
}
```

---

# 209. Completeness

Para ejecución normal:

```text
WindowAnalysisResult
→ COMPLETE
```

será obligatorio.

---

# 210. No late semantic discovery

Optimizer, Planner y Compiler no deberán descubrir tardíamente:

```text
what window name means
whether frame is legal
what function is being called
what type it returns
whether ordering is required
```

---

# 211. Normalization

Normalization podrá:

```text
canonicalize window builder structures
normalize default markers
canonicalize ordering terms
flatten equivalent wrapper nodes
normalize boundary representations
```

---

# 212. Normalization shall not

No deberá:

```text
change frame meaning
remove ordering based on cost
merge window groups
choose named-window expansion strategy
select physical sorts
```

---

# 213. Default frame normalization

La arquitectura deberá distinguir:

```text
syntactic normalization
semantic default resolution
```

La segunda requiere conocimiento de función/order/capabilities y pertenece a Semantic Analysis.

---

# 214. Fingerprinting

Structural fingerprint deberá incluir:

```text
window function identity
argument structure
partition structure
ordering structure
frame structure
named window references
null treatment
extensions
metadata
```

---

# 215. Semantic fingerprint

Además incluirá:

```text
resolved symbols
resolved types
resolved window definitions
resolved inheritance
resolved frames
nullability
determinism
capability requirements
portability
schema identity/version
extension versions
```

---

# 216. Runtime values

Valores ordinarios de parámetros no deberán formar parte del semantic fingerprint.

---

# 217. Dynamic frame parameters

La estructura:

```text
ROWS BETWEEN :n PRECEDING AND CURRENT ROW
```

sí forma parte del query shape.

El valor concreto de `:n` no.

---

# 218. Cache isolation

Un semantic window artifact sólo podrá reutilizarse cuando:

```text
query shape
schema fingerprint
type system version
window registry version
capability snapshot
policy fingerprint
extension versions
```

sean compatibles.

---

# 219. Persistent runtime safety

El sistema deberá ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 220. Shared state

Podrá compartirse:

```text
frozen WindowFunctionRegistry
immutable descriptors
immutable WindowSpecifications
immutable semantic artifacts
stateless factories
```

---

# 221. Operation-local state

Deberá mantenerse local:

```text
WindowBuilderState
WindowResolutionState
WindowScopeBuilder
WindowDependencyGraphBuilder
FrameAnalysisState
diagnostics
work queues
temporary indexes
memoization
```

---

# 222. No current window singleton

Prohibido:

```php
WindowContext::$current;
```

---

# 223. No global named-window registry per request

Named windows pertenecen al query/scope actual, no a un registry global mutable.

---

# 224. Concurrent query isolation

Dos queries concurrentes con una named window:

```text
w
```

deberán tener identidades completamente independientes.

---

# 225. Resource budgets

Deberán existir límites para:

```text
window expression count
named window count
inheritance depth
partition expression count
ordering term count
frame expression complexity
window dependency edges
window groups
extension nodes
semantic analysis work
```

---

# 226. WindowBudget

Conceptualmente:

```php
final readonly class WindowBudget
{
    public function __construct(
        public int $maxWindowExpressions,
        public int $maxNamedWindows,
        public int $maxInheritanceDepth,
        public int $maxPartitionExpressions,
        public int $maxOrderingTerms,
        public int $maxDependencyEdges,
    ) {}
}
```

---

# 227. Budget exhaustion

Deberá generar un error explícito.

Nunca:

```text
silently ignore remaining windows
```

---

# 228. Diagnostics

Los diagnostics deberán incluir cuando sea posible:

```text
WindowExpressionId
WindowDefinitionId
WindowScopeId
function identity
partition
ordering
frame
AST source
resolved symbols
resolved types
capability
platform
provenance
```

---

# 229. Invalid frame diagnostic

Ejemplo:

```text
Window frame W7 is invalid.

Start:
    4 FOLLOWING

End:
    2 PRECEDING

The frame start cannot occur logically
after the frame end.
```

---

# 230. Missing ordering diagnostic

```text
Window function RANK requires an ordering
under the active semantic policy.

Window:
    sales_rank

Partition:
    region_id
```

---

# 231. Unknown named window

```text
Window expression W12 references
unknown window "customer_window".

Visible named windows:
    monthly_orders
    regional_orders
```

---

# 232. Type diagnostic

```text
NTILE argument must be compatible
with a positive integer domain.

Resolved type:
    String
```

---

# 233. Capability diagnostic

```text
The query requires GROUPS window frames,
but the selected platform does not provide
a native or semantics-preserving strategy.
```

---

# 234. Testing strategy

Tests deberán cubrir:

```text
empty OVER
PARTITION BY
window ORDER BY
ROWS
RANGE
GROUPS
frame boundaries
frame exclusions
implicit frames
explicit frames

ROW_NUMBER
RANK
DENSE_RANK
NTILE

LAG
LEAD
FIRST_VALUE
LAST_VALUE
NTH_VALUE

aggregate windows

named windows
window inheritance
window cycles

peer groups
nullability
type inference
lineage
dependencies
capabilities
portability

optimizer grouping
sort reuse
persistent runtime
concurrency
budgets
extensions
```

---

# 235. Builder tests

Deberán verificar estructuras:

```text
WindowExpression
WindowSpecification
PartitionSpecification
WindowOrdering
WindowFrame
WindowDefinition
WindowReference
ParameterDefinitionSet
BindingSet
```

y no SQL.

---

# 236. Semantic tests

Deberán verificar:

```text
function resolution
window scope
named window resolution
type resolution
nullability
frame semantics
peer semantics
determinism
dependency graph
lineage
capabilities
portability
```

---

# 237. Compiler tests

Aquí sí deberán comprobarse representaciones SQL específicas para:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

según sus capabilities reales.

---

# 238. Architecture tests

El Window Builder no deberá importar:

```text
PDO
Driver
Connection
ConnectionManager
Statement
Executor
SQLCompiler
EntityManager
UnitOfWork
```

---

# 239. Runtime leakage tests

Entre work units deberá comprobarse ausencia de:

```text
named window leakage
window expression leakage
frame leakage
parameter leakage
diagnostic leakage
schema leakage
scope leakage
```

---

# 240. Namespace propuesto

Construcción:

```text
VoltStack\Quantum\Database\Query\Builder\Window
```

Semántica:

```text
VoltStack\Quantum\Database\Query\Semantic\Window
```

---

# 241. Directory structure propuesta

```text
Query/
├── Builder/
│   └── Window/
│       ├── Contract/
│       │   ├── WindowBuilderInterface.php
│       │   └── WindowFinalizerInterface.php
│       │
│       ├── Core/
│       │   ├── WindowExpression.php
│       │   ├── WindowExpressionId.php
│       │   ├── WindowSpecification.php
│       │   ├── WindowReference.php
│       │   ├── WindowName.php
│       │   ├── WindowDefinition.php
│       │   └── WindowDefinitionId.php
│       │
│       ├── Function/
│       │   ├── WindowFunctionId.php
│       │   ├── WindowFunctionDescriptor.php
│       │   ├── WindowFunctionCategory.php
│       │   └── WindowFunctionRegistry.php
│       │
│       ├── Partition/
│       │   └── PartitionSpecification.php
│       │
│       ├── Ordering/
│       │   ├── WindowOrdering.php
│       │   └── WindowOrderingTerm.php
│       │
│       ├── Frame/
│       │   ├── WindowFrame.php
│       │   ├── WindowFrameUnit.php
│       │   ├── WindowFrameBoundary.php
│       │   ├── UnboundedPrecedingBoundary.php
│       │   ├── OffsetPrecedingBoundary.php
│       │   ├── CurrentRowBoundary.php
│       │   ├── OffsetFollowingBoundary.php
│       │   ├── UnboundedFollowingBoundary.php
│       │   └── WindowFrameExclusion.php
│       │
│       ├── Builder/
│       │   ├── WindowBuilder.php
│       │   ├── WindowBuilderState.php
│       │   └── WindowFinalizer.php
│       │
│       ├── Budget/
│       │   └── WindowBudget.php
│       │
│       ├── Extension/
│       │   └── WindowExtension.php
│       │
│       └── Exception/
│           ├── WindowException.php
│           ├── InvalidWindowException.php
│           ├── InvalidWindowFrameException.php
│           ├── UnknownWindowException.php
│           └── WindowBudgetExceededException.php
│
└── Semantic/
    └── Window/
        ├── WindowScopeId.php
        ├── WindowScope.php
        ├── WindowScopeTable.php
        ├── WindowSemanticInfo.php
        ├── WindowSemanticTable.php
        ├── WindowDefinitionTable.php
        ├── ResolvedWindowSpecification.php
        ├── ResolvedWindowFrame.php
        ├── WindowFrameSemanticInfo.php
        ├── WindowNullabilityInfo.php
        ├── WindowDependencyGraph.php
        ├── WindowGroup.php
        ├── WindowSemanticAnalyzer.php
        ├── WindowFrameAnalyzer.php
        └── WindowLegalityAnalyzer.php
```

---

# 242. Ownership matrix

| Concern | Owner |
|---|---|
| Fluent window construction | Window Builder |
| Window AST structure | Query Model / AST |
| Function descriptors | Window Function System |
| Runtime values | BindingSet |
| Symbols | Symbol Resolution |
| Argument types | Query Type System |
| Result types | Window Semantic Analysis |
| Named window resolution | Window Semantic Analysis |
| Frame legality | Window Semantic Analysis |
| Peer semantics | Window Semantic Analysis |
| Window nullability | Window Semantic Analysis |
| Constraints/facts | Constraint Analysis |
| Window grouping | Optimizer |
| Sort reuse | Optimizer / Planner |
| Physical window strategy | Planner |
| SQL syntax | Compiler |
| Statement execution | Executor |
| ORM hydration | ORM/Hydration |
| Runtime reset | Runtime lifecycle |

---

# 243. Invariantes arquitectónicos

## DB-WIN-001
Window Function será distinta de Aggregate Function.

## DB-WIN-002
Window Partition será distinta de GROUP BY.

## DB-WIN-003
Window ORDER BY será distinto de Query ORDER BY.

## DB-WIN-004
Window ORDER BY será distinto de Aggregate ORDER BY.

## DB-WIN-005
Window Frame será distinto de pagination.

## DB-WIN-006
Window Builder no generará SQL.

## DB-WIN-007
Window Builder no ejecutará queries.

## DB-WIN-008
Window expressions serán estructuradas.

## DB-WIN-009
OVER será estructurado.

## DB-WIN-010
Empty OVER será distinto de ausencia de OVER.

## DB-WIN-011
Window functions tendrán identidad semántica.

## DB-WIN-012
Core no dependerá de nombres SQL vendor-specific.

## DB-WIN-013
ROW_NUMBER será distinto de RANK.

## DB-WIN-014
RANK será distinto de DENSE_RANK.

## DB-WIN-015
NTILE tendrá reglas propias.

## DB-WIN-016
LAG tendrá descriptor semántico.

## DB-WIN-017
LEAD tendrá descriptor semántico.

## DB-WIN-018
FIRST_VALUE tendrá descriptor semántico.

## DB-WIN-019
LAST_VALUE tendrá descriptor semántico.

## DB-WIN-020
NTH_VALUE tendrá descriptor semántico.

## DB-WIN-021
Aggregate window será distinta de grouped aggregate.

## DB-WIN-022
Window evaluation preservará cardinalidad salvo semántica explícita distinta.

## DB-WIN-023
Partition expressions serán estructuradas.

## DB-WIN-024
Window ordering terms serán estructurados.

## DB-WIN-025
NULL ordering será semántico.

## DB-WIN-026
Peer groups serán explícitamente analizables.

## DB-WIN-027
Non-total ordering afectará determinism.

## DB-WIN-028
Frame será first-class.

## DB-WIN-029
ROWS será distinto de RANGE.

## DB-WIN-030
RANGE será distinto de GROUPS.

## DB-WIN-031
GROUPS será distinto de ROWS.

## DB-WIN-032
Frame boundaries serán objetos estructurados.

## DB-WIN-033
Boundary offsets serán expressions.

## DB-WIN-034
Runtime offsets no serán inspeccionados durante Semantic Analysis.

## DB-WIN-035
CURRENT ROW dependerá de frame semantics.

## DB-WIN-036
Explicit frame será distinto de implicit frame.

## DB-WIN-037
Frame exclusions serán capability-driven.

## DB-WIN-038
Named windows serán first-class.

## DB-WIN-039
WindowName será distinto de WindowDefinitionId.

## DB-WIN-040
Named windows tendrán scope.

## DB-WIN-041
Window inheritance será resuelta semánticamente.

## DB-WIN-042
Window inheritance cycles serán rechazados.

## DB-WIN-043
Builder no realizará physical window deduplication.

## DB-WIN-044
Semantic Analysis podrá detectar equivalent windows.

## DB-WIN-045
Optimizer será dueño de window consolidation.

## DB-WIN-046
Window dependencies serán explícitas.

## DB-WIN-047
Same-level illegal nested windows serán rechazadas.

## DB-WIN-048
Subquery window scopes estarán aislados.

## DB-WIN-049
Aggregate/window evaluation levels serán explícitos.

## DB-WIN-050
Window outputs no estarán disponibles arbitrariamente en WHERE.

## DB-WIN-051
Window outputs no estarán disponibles arbitrariamente en GROUP BY.

## DB-WIN-052
Post-window filtering será una fase distinta.

## DB-WIN-053
QUALIFY-like semantics no se implementará como HAVING raw.

## DB-WIN-054
Projection alias será distinto de WindowExpressionId.

## DB-WIN-055
Window function descriptors declararán arity.

## DB-WIN-056
Descriptors declararán argument rules.

## DB-WIN-057
Descriptors declararán result type rules.

## DB-WIN-058
Descriptors declararán nullability rules.

## DB-WIN-059
Descriptors declararán ordering requirements.

## DB-WIN-060
Descriptors declararán frame behavior.

## DB-WIN-061
No todas las window functions utilizarán frame de igual manera.

## DB-WIN-062
Window types reutilizarán Query Type System.

## DB-WIN-063
Aggregate window types reutilizarán aggregate rules cuando corresponda.

## DB-WIN-064
No existirán reglas duplicadas de SUM sin necesidad.

## DB-WIN-065
Window context podrá modificar nullability.

## DB-WIN-066
LAG/LEAD podrán introducir nullability.

## DB-WIN-067
Empty frames podrán introducir nullability.

## DB-WIN-068
Null treatment será explícito.

## DB-WIN-069
Null treatment será capability-aware.

## DB-WIN-070
Window semantic info vivirá en side tables.

## DB-WIN-071
AST permanecerá immutable.

## DB-WIN-072
SemanticQueryGraph representará windows.

## DB-WIN-073
Value lineage será distinta de ordering dependency.

## DB-WIN-074
Partition dependency será explícita.

## DB-WIN-075
Frame dependency será explícita.

## DB-WIN-076
Ranking functions no inventarán value lineage.

## DB-WIN-077
Constraint facts de windows serán scope-aware.

## DB-WIN-078
ROW_NUMBER uniqueness será partition-scoped cuando corresponda.

## DB-WIN-079
Optimizer no cambiará frame semantics.

## DB-WIN-080
Optimizer no sustituirá RANGE por ROWS arbitrariamente.

## DB-WIN-081
Optimizer no eliminará required ordering.

## DB-WIN-082
Sort reuse pertenecerá a Optimizer/Planner.

## DB-WIN-083
Physical window strategy pertenecerá al Planner.

## DB-WIN-084
Compiler no resolverá named windows semánticamente.

## DB-WIN-085
Compiler no validará frame legality.

## DB-WIN-086
Compiler no inferirá window result types.

## DB-WIN-087
Capabilities reemplazarán vendor conditionals.

## DB-WIN-088
Capability descriptors podrán contener restricciones.

## DB-WIN-089
Boolean capabilities no se usarán cuando sean insuficientes.

## DB-WIN-090
Unsafe emulation estará prohibida.

## DB-WIN-091
Portability será explícita.

## DB-WIN-092
Raw window expressions serán explícitas.

## DB-WIN-093
Raw windows podrán actuar como semantic barriers.

## DB-WIN-094
Semantic extensions serán preferidas sobre raw SQL.

## DB-WIN-095
WindowFunctionRegistry será frozen.

## DB-WIN-096
Registry no tendrá request-local mutation.

## DB-WIN-097
Named windows no vivirán en registry global mutable.

## DB-WIN-098
Builder closures serán construction-only.

## DB-WIN-099
Closures no entrarán al Query Model.

## DB-WIN-100
Closures no entrarán a semantic fingerprints.

## DB-WIN-101
WindowBuilderState será operation-scoped.

## DB-WIN-102
WindowResolutionState será operation-scoped.

## DB-WIN-103
No existirá current-window global.

## DB-WIN-104
Concurrent window queries estarán aisladas.

## DB-WIN-105
Finalizer no resolverá schema.

## DB-WIN-106
Finalizer no inferirá tipos.

## DB-WIN-107
Finalizer no seleccionará capabilities.

## DB-WIN-108
Normalization no realizará cost-based window rewrites.

## DB-WIN-109
Semantic Analysis resolverá window meaning antes de Optimization.

## DB-WIN-110
No habrá lazy window resolution en Planner.

## DB-WIN-111
No habrá lazy window resolution en Compiler.

## DB-WIN-112
Fingerprints incluirán window function identity.

## DB-WIN-113
Fingerprints incluirán partition structure.

## DB-WIN-114
Fingerprints incluirán window ordering.

## DB-WIN-115
Fingerprints incluirán frame structure.

## DB-WIN-116
Semantic fingerprints incluirán resolved window identity.

## DB-WIN-117
Runtime binding values estarán fuera del semantic fingerprint.

## DB-WIN-118
Window analysis tendrá resource budgets.

## DB-WIN-119
Budget exhaustion será explícito.

## DB-WIN-120
No habrá silent truncation.

## DB-WIN-121
Window construction será determinista.

## DB-WIN-122
Window resolution será determinista.

## DB-WIN-123
Window inheritance resolution será determinista.

## DB-WIN-124
Frame resolution será determinista.

## DB-WIN-125
ORM reutilizará el mismo Window Query System.

## DB-WIN-126
Active Record no implementará otro window engine.

## DB-WIN-127
Repository queries no implementarán otro window engine.

## DB-WIN-128
Window System permanecerá independiente de ORM.

## DB-WIN-129
Window System permanecerá independiente de Connection.

## DB-WIN-130
Window System permanecerá independiente del SQL dialect syntax.

---

# 244. Anti-patterns

## 244.1 Raw window as default

```php
selectRaw(
    'ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at)'
);
```

como implementación interna principal.

**Rechazado.**

---

## 244.2 Window stored as string

```php
$window = 'PARTITION BY user_id ORDER BY created_at';
```

**Rechazado.**

---

## 244.3 Frame stored as string

```php
$frame = 'ROWS BETWEEN 5 PRECEDING AND CURRENT ROW';
```

**Rechazado.**

---

## 244.4 Reusing GROUP BY model for PARTITION BY

**Rechazado.**

Aunque ambos contienen expressions, poseen semánticas distintas.

---

## 244.5 Reusing Query ORDER BY as Window ORDER BY without context

**Rechazado.**

Los primitives comunes pueden reutilizarse, pero el contexto semántico debe permanecer explícito.

---

## 244.6 RANGE = ROWS

**Rechazado.**

---

## 244.7 Ignoring frame semantics

Especialmente para:

```text
FIRST_VALUE
LAST_VALUE
NTH_VALUE
```

**Rechazado.**

---

## 244.8 Compiler resolves named windows

**Rechazado.**

---

## 244.9 Global named window map

**Rechazado.**

---

## 244.10 Vendor-specific semantic branches

```php
if ($driver === 'postgres') {
    // semantic resolution
}
```

**Rechazado.**

---

# 245. Ejemplo integral

Supongamos:

```php
$query = DB::table('orders')
    ->select('id')
    ->select('customer_id')
    ->select('created_at')
    ->select('total')
    ->window(
        'customer_orders',
        DB::window()
            ->partitionBy('customer_id')
            ->orderBy('created_at')
            ->orderBy('id')
    )
    ->selectWindow(
        DB::windowFunction()
            ->rowNumber()
            ->overWindow('customer_orders'),
        as: 'order_number'
    )
    ->selectWindow(
        DB::aggregate()
            ->sum('total')
            ->over(
                DB::window()
                    ->base('customer_orders')
                    ->rowsBetween(
                        WindowBoundary::unboundedPreceding(),
                        WindowBoundary::currentRow(),
                    )
            ),
        as: 'running_total'
    )
    ->selectWindow(
        DB::windowFunction()
            ->lag('total')
            ->overWindow('customer_orders'),
        as: 'previous_total'
    );
```

---

# 246. Builder representation

Conceptualmente:

```text
SelectQuery
│
├── FROM
│   └── orders
│
├── WINDOW DEFINITIONS
│   └── customer_orders
│       ├── PARTITION BY
│       │   └── customer_id
│       └── ORDER BY
│           ├── created_at
│           └── id
│
└── PROJECTION
    ├── id
    ├── customer_id
    ├── created_at
    ├── total
    │
    ├── ROW_NUMBER()
    │   └── OVER customer_orders
    │
    ├── SUM(total)
    │   └── OVER
    │       ├── BASE customer_orders
    │       └── ROWS
    │           ├── UNBOUNDED PRECEDING
    │           └── CURRENT ROW
    │
    └── LAG(total)
        └── OVER customer_orders
```

---

# 247. Semantic resolution

Named window:

```text
WindowDefinition WD1
│
├── Name
│   └── customer_orders
│
├── Partition
│   └── orders.customer_id
│
└── Ordering
    ├── orders.created_at ASC
    └── orders.id ASC
```

---

# 248. ROW_NUMBER semantic info

```text
WindowExpression W1
│
├── Function
│   └── ROW_NUMBER
│
├── Window
│   └── WD1
│
├── Result Type
│   └── IntegerLike
│
├── Nullability
│   └── NOT NULL
│
└── Determinism
    └── depends on total ordering proof
```

---

# 249. Running SUM semantic info

```text
WindowExpression W2
│
├── Function
│   └── SUM
│
├── Input
│   └── orders.total
│
├── Partition
│   └── customer_id
│
├── Ordering
│   ├── created_at
│   └── id
│
├── Frame
│   └── ROWS
│       ├── UNBOUNDED PRECEDING
│       └── CURRENT ROW
│
├── Lineage
│   └── orders.total
│
└── Output
    └── running_total
```

---

# 250. LAG semantic info

```text
WindowExpression W3
│
├── Function
│   └── LAG
│
├── Value
│   └── orders.total
│
├── Offset
│   └── default 1
│
├── Window
│   └── WD1
│
├── Result Type
│   └── type(total)
│
└── Nullability
    └── nullable because partition boundary
```

---

# 251. Semantic graph

```text
Query
│
├── Relation orders
│
├── WindowDefinition WD1
│   ├── PARTITIONS_BY → customer_id
│   └── ORDERS_BY
│       ├── created_at
│       └── id
│
├── WindowExpression W1
│   ├── USES_FUNCTION → ROW_NUMBER
│   └── USES_WINDOW → WD1
│
├── WindowExpression W2
│   ├── USES_FUNCTION → SUM
│   ├── INHERITS_WINDOW → WD1
│   ├── USES_FRAME → F1
│   └── DERIVES_FROM → total
│
└── WindowExpression W3
    ├── USES_FUNCTION → LAG
    ├── USES_WINDOW → WD1
    └── DERIVES_FROM → total
```

---

# 252. Optimizer view

Optimizer podrá detectar:

```text
W1
W2
W3
```

comparten:

```text
PARTITION BY customer_id
ORDER BY created_at, id
```

y generar una oportunidad de:

```text
CompatibleWindowGroup
```

---

# 253. Logical planning

Conceptualmente:

```text
Projection
    │
WindowGroup
├── ROW_NUMBER
├── running SUM
└── LAG
    │
Ordered Partition
├── customer_id
├── created_at
└── id
    │
Scan orders
```

---

# 254. Physical considerations

Planner podrá considerar:

```text
existing ordering
index ordering
partition cardinality
memory
spill support
row width
frame type
number of functions
parallelism
```

sin modificar la semántica declarada.

---

# 255. Fórmula principal

```text
Window Expression
=
Window Function
+
Arguments
+
Window Specification
+
Semantic Metadata
```

---

# 256. Fórmula de Window Specification

```text
Window Specification
=
Optional Base Window
+
Partition Specification
+
Window Ordering
+
Optional Frame
```

---

# 257. Fórmula de frame

```text
Window Frame
=
Frame Unit
+
Start Boundary
+
End Boundary
+
Exclusion
```

---

# 258. Fórmula de semantic window

```text
Semantic Window
=
Resolved Function
+
Resolved Arguments
+
Resolved Partition
+
Resolved Ordering
+
Resolved Frame
+
Peer Semantics
+
Result Type
+
Nullability
+
Determinism
+
Lineage
+
Dependencies
+
Capabilities
+
Portability
```

---

# 259. Fórmula de seguridad

```text
Safe Window Query
=
Structured Functions
+
Structured Partitions
+
Structured Ordering
+
Structured Frames
+
Parameterized Runtime Values
+
Explicit Raw Barriers
+
Capability Validation
+
Resource Budgets
```

---

# 260. Fórmula de optimización

```text
Safe Window Optimization
=
Semantic Window Graph
+
Partition Equivalence
+
Ordering Equivalence
+
Frame Compatibility
+
Dependency Knowledge
+
Determinism
+
Volatility
+
Cost Information
```

---

# 261. Fórmula de persistent-runtime safety

```text
Persistent-Safe Window System
=
Immutable Window Artifacts
+
Operation-Scoped Builder State
+
Operation-Scoped Semantic State
+
Frozen Function Registry
+
No Global Window Scope
+
No Runtime Resource Retention
```

---

# 262. Boundary final

```text
Window Builder
      │
      ▼
"What analytical operation
does the developer want?"
      │
      ▼
Semantic Analysis
      │
      ▼
"What function, partition,
ordering and frame does it mean?"
      │
      ▼
Constraint Analysis
      │
      ▼
"What facts can be proven
about its output?"
      │
      ▼
Optimizer
      │
      ▼
"Which windows can share
work safely?"
      │
      ▼
Planner
      │
      ▼
"What ordering/window stages
should be used?"
      │
      ▼
Compiler
      │
      ▼
"How does this platform
express the window query?"
```

---

# 263. Decisión arquitectónica final

VoltStack adopta las window functions como un sistema semántico first-class compuesto por:

```text
Window Function
+
Window Expression
+
Window Specification
+
Partition Specification
+
Window Ordering
+
Window Frame
+
Named Window
+
Window Scope
+
Window Semantic Information
+
Window Dependency Graph
```

La regla final será:

> **VoltStack no tratará `OVER`, `PARTITION BY`, window `ORDER BY`, `ROWS`, `RANGE`, `GROUPS`, ranking, navigation ni aggregate windows como fragmentos SQL. Serán conceptos estructurados del Query Engine, resueltos semánticamente antes de Optimization, Planning y Compilation.**

Y se mantiene como invariante:

```text
GROUP BY
≠
PARTITION BY

Aggregate Scope
≠
Window Scope

Aggregate Function
≠
Window Expression

Query ORDER BY
≠
Window ORDER BY

ROWS
≠
RANGE
≠
GROUPS
```

---

# 264. Estado del Bloque 4 — Query Builder

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
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
```

Quedan dos documentos para cerrar el bloque:

```text
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md
```

---

# 265. Siguiente documento

```text
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
```

El siguiente documento deberá definir la frontera controlada entre el Query Engine estructurado de VoltStack y expresiones SQL/raw específicas del desarrollador o plataforma, incluyendo:

```text
RawExpression
RawPredicate
RawProjection
RawOrdering
RawGrouping
RawHaving
RawJoinPredicate
RawCTE
RawDML Expression

Trusted vs Untrusted Raw Input
Raw Parameter Binding
Raw Identifier Handling
Raw Fragment Contracts
Raw Provenance
Raw Capabilities
Raw Portability

Semantic Barrier
Optimizer Barrier
Type Inference Barrier
Constraint Analysis Barrier
Lineage Barrier

Raw Security Policy
SQL Injection Prevention
Sensitive Parameter Redaction

Platform-specific Raw Expressions
Dialect-specific Raw Expressions

Raw Extension Integration
Raw Fingerprinting
Raw Serialization
Raw Diagnostics
Raw Telemetry

Persistent Runtime Safety
```

manteniendo el principio:

```text
Raw SQL
=
Explicit Escape Hatch

Raw SQL
≠
Default Query Representation
```