# 61_DATABASE_QUERY_COST_HINT_SYSTEM.md

# VoltStack Quantum Database
## Query Cost Hint System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 61 — Query Cost Hint System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Cost Intelligence  
**Versión:** 1.0

---

# 1. Propósito

`Query Cost Hint System` define la arquitectura mediante la cual VoltStack podrá proporcionar información orientativa sobre:

- cardinalidad;
- selectividad;
- tamaño de relaciones;
- distribución de datos;
- disponibilidad lógica de índices;
- ordenamiento disponible;
- locality;
- costo relativo;
- presión de memoria;
- preferencias;
- restricciones;
- confianza de estimaciones;

al:

```text
Query Optimizer
```

y posteriormente al:

```text
Query Planner
```

sin convertir esta información en decisiones físicas de ejecución.

La regla fundamental será:

```text
Cost Hint
    ≠
Physical Plan Decision
```

Un hint representa:

```text
evidence
preference
estimate
constraint
guidance
```

pero no necesariamente:

```text
final execution strategy
```

---

# 2. Problema arquitectónico

Una consulta puede poseer múltiples formas lógicamente equivalentes.

Ejemplo:

```text
A JOIN B JOIN C
```

puede evaluarse conceptualmente como:

```text
(A JOIN B) JOIN C
```

o:

```text
A JOIN (B JOIN C)
```

El documento 59 permite descubrir alternativas de join.

Sin embargo, para determinar cuál parece más prometedora se necesita información adicional.

Por ejemplo:

```text
Rows(A) ≈ 10
Rows(B) ≈ 100
Rows(C) ≈ 50,000,000
```

junto con:

```text
Selectivity(A.id = B.a_id) ≈ 0.01
```

puede ayudar a priorizar determinadas alternativas.

Esa información pertenece al:

```text
Query Cost Hint System
```

pero todavía no significa:

```text
Use HashJoin
Use NestedLoop
Use IndexScan
```

Esas decisiones pertenecen posteriormente al Planner.

---

# 3. Distinciones fundamentales

VoltStack distinguirá estrictamente:

```text
Semantic Fact
      ≠
Constraint Fact
      ≠
Cardinality Fact
      ≠
Cardinality Estimate
      ≠
Selectivity Estimate
      ≠
Statistics
      ≠
Cost Hint
      ≠
Optimization Preference
      ≠
Physical Cost
      ≠
Physical Plan
```

---

# 4. Semantic Fact

Ejemplo:

```text
users.id : UserId
```

Es una verdad semántica conocida.

No es una estimación.

---

# 5. Constraint Fact

Ejemplo:

```text
users.id is UNIQUE
```

si proviene de una constraint conocida.

Tampoco es un hint.

---

# 6. Cardinality Fact

Algunas cardinalidades pueden conocerse exactamente por estructura.

Ejemplo conceptual:

```text
SELECT COUNT(...)
```

sin `GROUP BY` ni `HAVING` puede tener:

```text
output cardinality = exactly 1
```

bajo las condiciones semánticas correspondientes.

Esto es diferente de estimar cuántas filas existen en una tabla.

---

# 7. Cardinality Estimate

Ejemplo:

```text
estimated users rows = 12,500,000
```

No es una verdad semántica.

Puede:

```text
be stale
be approximate
be sampled
be incomplete
```

---

# 8. Selectivity Estimate

Ejemplo:

```text
status = 'ACTIVE'
≈ 0.35
```

representa una estimación sobre qué proporción de filas sobrevivirá a un predicate.

---

# 9. Statistics

Statistics son datos observacionales utilizados para producir estimaciones.

Ejemplos:

```text
row count
distinct values
null fraction
histograms
frequency distributions
min/max
correlations
data size
```

---

# 10. Cost Hint

Un `CostHint` es una pieza estructurada de información que puede orientar una decisión de optimización o planificación.

Puede derivarse de:

```text
statistics
schema metadata
application knowledge
extensions
previous analysis
configuration
explicit developer intent
```

---

# 11. Cost Hint ≠ Statistics

Statistics contienen observaciones.

Hints contienen información ya expresada en una forma útil para consumidores del Query Engine.

Ejemplo:

```text
Statistic:
    relation row count = 100,000,000

Derived Hint:
    relation cardinality is VERY_LARGE
```

---

# 12. Cost Hint ≠ Cost Model

El Cost Model transforma información sobre una alternativa en una estimación de costo.

Ejemplo conceptual:

```text
CostModel(
    logical/physical alternative,
    statistics,
    environment
)
→
EstimatedCost
```

`Query Cost Hint System` no sustituye ese modelo.

---

# 13. Cost Hint ≠ Planner

Un hint:

```text
Prefer index-compatible predicate
```

no significa:

```text
Planner MUST choose index scan
```

salvo que exista una constraint explícita con semántica `REQUIRE`.

Incluso entonces, el Planner deberá validar que la exigencia sea realizable.

---

# 14. Posición arquitectónica

```text
Schema
Statistics
Application
Extensions
Configuration
     │
     ▼
Cost Hint Providers
     │
     ▼
Cost Hint Normalization
     │
     ▼
Cost Hint Validation
     │
     ▼
Cost Hint Resolution
     │
     ▼
QueryCostHintSet
     │
     ├──────────────► Query Optimizer
     │
     └──────────────► Query Planner
```

---

# 15. Relación con Semantic Query Engine

```text
SemanticQueryArtifact
        │
        ├──────────────┐
        ▼              ▼
ConstraintGraph    Dependencies
        │
        │
        ▼
Cost Hint Resolution
        │
        ▼
OptimizationContext
```

Los hints podrán utilizar semantic identities resueltas como:

```text
RelationId
SymbolId
ExpressionId
PredicateId
JoinId
```

---

# 16. No reinterpretación semántica

El Cost Hint System no podrá:

```text
resolve symbols
infer types
invent constraints
change nullability
change relation semantics
```

Consumirá los resultados del Semantic Engine.

---

# 17. Arquitectura general

```text
                   Query Cost Intelligence
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      Statistics       Static Hints    App/Extension Hints
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                    Hint Collection
                           │
                           ▼
                    Hint Validation
                           │
                           ▼
                  Hint Normalization
                           │
                           ▼
                    Hint Resolution
                           │
                           ▼
                    QueryCostHintSet
                           │
                ┌──────────┴───────────┐
                ▼                      ▼
            Optimizer               Planner
```

---

# 18. QueryCostHint

Contrato conceptual:

```php
interface QueryCostHint
{
    public function id(): CostHintId;

    public function kind(): CostHintKind;

    public function scope(): CostHintScope;

    public function source(): CostHintSource;

    public function confidence(): HintConfidence;

    public function strength(): HintStrength;
}
```

---

# 19. CostHintId

Cada hint tendrá identidad explícita:

```php
final readonly class CostHintId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 20. CostHintKind

Ejemplos:

```php
enum CostHintKind
{
    case CARDINALITY;
    case SELECTIVITY;
    case RELATION_SIZE;
    case DISTINCT_COUNT;
    case NULL_FRACTION;
    case DATA_DISTRIBUTION;
    case JOIN_PRIORITY;
    case ACCESS_PATH;
    case INDEX_AVAILABILITY;
    case ORDERING;
    case LOCALITY;
    case NETWORK;
    case MEMORY;
    case PARALLELISM;
    case MATERIALIZATION;
    case CUSTOM;
}
```

---

# 21. QueryCostHintSet

Los hints resueltos serán almacenados en un artifact inmutable.

```php
final readonly class QueryCostHintSet
{
    public function __construct(
        public array $hints,
        public CostHintSetFingerprint $fingerprint,
    ) {}
}
```

---

# 22. CostHintScope

Un hint nunca deberá aplicarse ambiguamente.

Posibles scopes:

```text
QUERY
RELATION
PREDICATE
EXPRESSION
JOIN
SUBQUERY
CTE
AGGREGATE
WINDOW
SET_OPERATION
LOGICAL_ALTERNATIVE
EXTENSION_NODE
```

---

# 23. Scope target

Ejemplo:

```php
final readonly class RelationCostHintScope
{
    public function __construct(
        public RelationId $relationId,
    ) {}
}
```

---

# 24. Semantic IDs

Después de semantic analysis se utilizarán preferentemente IDs semánticos.

No:

```text
"users"
```

sino:

```text
RelationId(R17)
```

---

# 25. Self-joins

Esto es esencial para:

```sql
FROM users u1
JOIN users u2 ...
```

`u1` y `u2` son dos relation instances distintas.

Un hint puede aplicar a una sola de ellas.

---

# 26. HintSource

Todo hint tendrá provenance.

```php
enum CostHintSourceKind
{
    case SEMANTIC_ANALYSIS;
    case SCHEMA;
    case STATISTICS;
    case CONFIGURATION;
    case APPLICATION;
    case EXTENSION;
    case OPTIMIZER;
    case PLANNER;
}
```

---

# 27. CostHintSource

```php
final readonly class CostHintSource
{
    public function __construct(
        public CostHintSourceKind $kind,
        public string $sourceId,
        public ?string $version = null,
    ) {}
}
```

---

# 28. Provenance

Ejemplo:

```text
Hint:
    RelationCardinality(users) ≈ 12,500,000

Source:
    StatisticsSnapshot #S381

Collected:
    schema statistics provider
```

---

# 29. HintConfidence

Las estimaciones no tendrán una falsa precisión.

```php
enum HintConfidenceLevel
{
    case EXACT;
    case VERY_HIGH;
    case HIGH;
    case MEDIUM;
    case LOW;
    case VERY_LOW;
    case UNKNOWN;
}
```

---

# 30. EXACT

`EXACT` sólo deberá utilizarse cuando exista una garantía real.

Ejemplo:

```text
VALUES (1), (2), (3)
```

puede tener cardinalidad exacta:

```text
3
```

---

# 31. Statistics exactness

Que un motor reporte:

```text
table rows = 10,000
```

no significa necesariamente que sea exacto.

El provider deberá conocer la semántica del dato recibido.

---

# 32. HintConfidence model

Podrá combinar:

```text
confidence level
sample quality
age
coverage
source reliability
```

---

# 33. HintStrength

Confianza y fuerza son conceptos diferentes.

```php
enum HintStrength
{
    case INFORMATIONAL;
    case PREFER;
    case AVOID;
    case REQUIRE;
    case PROHIBIT;
}
```

---

# 34. Confidence ≠ Strength

Ejemplo:

```text
confidence = LOW
strength   = PREFER
```

significa:

```text
the application prefers this,
but the underlying estimate is uncertain
```

---

# 35. INFORMATIONAL

No expresa preferencia.

Sólo aporta información.

---

# 36. PREFER

La alternativa debe favorecerse cuando sea razonable.

---

# 37. AVOID

La alternativa debe penalizarse cuando existan opciones adecuadas.

---

# 38. REQUIRE

Representa una exigencia explícita.

Si no puede satisfacerse:

```text
fail explicitly
```

o aplicar una política de fallback explícitamente configurada.

Nunca ignorarla silenciosamente.

---

# 39. PROHIBIT

Una alternativa determinada no deberá utilizarse.

---

# 40. HintStrength ≠ SQL optimizer hint syntax

VoltStack define semántica framework-level.

No está modelando directamente:

```sql
/*+ INDEX(...) */
```

La traducción a hints específicos del motor, si existe, será responsabilidad posterior del Compiler/Platform integration.

---

# 41. CardinalityHint

```php
final readonly class CardinalityHint implements QueryCostHint
{
    public function __construct(
        public CostHintId $id,
        public CostHintScope $scope,
        public CardinalityEstimate $estimate,
        public CostHintSource $source,
        public HintConfidence $confidence,
        public HintStrength $strength,
    ) {}
}
```

---

# 42. CardinalityEstimate

No se representará necesariamente sólo como un entero.

Podrá contener:

```text
lower bound
expected value
upper bound
confidence
```

---

# 43. CardinalityRange

```php
final readonly class CardinalityRange
{
    public function __construct(
        public ?int $minimum,
        public ?float $expected,
        public ?int $maximum,
    ) {}
}
```

---

# 44. Unknown cardinality

Será explícita.

No:

```text
0
```

No:

```text
-1
```

Preferible:

```text
UnknownCardinalityEstimate
```

---

# 45. Exact cardinality

Ejemplo:

```php
ExactCardinality::of(10);
```

---

# 46. Estimated cardinality

Ejemplo:

```php
EstimatedCardinality::around(
    expected: 1_200_000,
    lower: 900_000,
    upper: 1_800_000,
);
```

---

# 47. Cardinality Fact vs Hint

Si Constraint Analysis demuestra:

```text
cardinality <= 1
```

esa información no deberá degradarse a un hint probabilístico.

Podrá exponerse al sistema de costos como:

```text
Exact/Proven Cardinality Bound
```

manteniendo su provenance de semantic fact.

---

# 48. RelationSizeHint

Cardinality y tamaño físico son diferentes.

```text
1,000 rows × 2 KB
```

puede costar más memoria/I/O que:

```text
10,000 rows × 20 bytes
```

---

# 49. RelationSizeEstimate

Podrá contener:

```text
row count
average row width
logical bytes
estimated physical bytes
```

---

# 50. RowWidthEstimate

```php
final readonly class RowWidthEstimate
{
    public function __construct(
        public ?float $averageBytes,
        public HintConfidence $confidence,
    ) {}
}
```

---

# 51. SelectivityHint

Representa una estimación:

```text
0.0 <= selectivity <= 1.0
```

---

# 52. PredicateSelectivityHint

Ejemplo:

```text
Predicate P17

estimated selectivity:
    0.02
```

---

# 53. Selectivity ≠ truth probability

No representa la probabilidad de que el predicate sea lógicamente verdadero en abstracto.

Representa la fracción estimada de rows relevantes que sobreviven.

---

# 54. SQL NULL semantics

El cálculo puede necesitar distinguir:

```text
TRUE fraction
FALSE fraction
UNKNOWN fraction
```

---

# 55. PredicateTruthDistribution

Modelo avanzado:

```php
final readonly class PredicateTruthDistribution
{
    public function __construct(
        public float $trueFraction,
        public float $falseFraction,
        public float $unknownFraction,
    ) {}
}
```

con:

```text
TRUE + FALSE + UNKNOWN ≈ 1
```

---

# 56. NULL fraction

Un:

```text
NullFractionHint
```

puede ser relevante para:

```text
IS NULL
IS NOT NULL
comparison predicates
outer joins
aggregates
```

---

# 57. DistinctCountHint

Representa:

```text
NDV
=
Number of Distinct Values
```

para una expresión o columna.

---

# 58. Distinct count ≠ row count

Ejemplo:

```text
Rows(status) = 10,000,000
NDV(status)   = 4
```

---

# 59. DistributionHint

Algunas columnas tienen distribuciones altamente sesgadas.

Ejemplo:

```text
status:
    ACTIVE    95%
    SUSPENDED 3%
    DELETED   2%
```

---

# 60. HistogramHint

VoltStack podrá modelar histogramas abstractos.

No deberá exponer directamente formatos internos de MySQL/PostgreSQL al Optimizer.

---

# 61. Platform statistics adapters

Arquitectura:

```text
MySQL Statistics
PostgreSQL Statistics
SQLite Information
Extension Statistics
       │
       ▼
Platform Statistics Adapter
       │
       ▼
VoltStack Statistics Model
       │
       ▼
Cost Hints
```

---

# 62. Statistics abstraction

El Optimizer no deberá preguntar:

```php
if ($database === 'postgresql') {
    ...
}
```

---

# 63. DataDistributionHint

Podrá representar:

```text
uniform
skewed
clustered
partitioned
unknown
```

---

# 64. Distribution ≠ physical partitioning

Un distribution hint describe propiedades estimadas de los datos.

No decide cómo ejecutar la consulta.

---

# 65. Correlation statistics

Predicates no siempre son independientes.

Ejemplo:

```text
country = 'MX'
AND
state = 'Nuevo León'
```

No deberá estimarse necesariamente como:

```text
selectivity(country)
*
selectivity(state)
```

---

# 66. CorrelationHint

Podrá indicar correlación estadística entre:

```text
expressions
predicates
columns
```

---

# 67. Independence assumption

Si no existen estadísticas de correlación, podrá utilizarse una heuristic explícita.

Pero deberá quedar marcada como:

```text
assumption
```

no semantic fact.

---

# 68. JoinCardinalityHint

Podrá estimar:

```text
Rows(A ⋈ B)
```

---

# 69. Join selectivity

Modelo conceptual:

```text
JoinSelectivity
≈
OutputRows / CartesianRows
```

cuando esa formulación sea apropiada.

---

# 70. Constraint-assisted estimates

Información demostrada como:

```text
PK
FK
UNIQUE
NOT NULL
```

puede mejorar estimaciones.

---

# 71. FK example

Si:

```text
orders.user_id
→ users.id
```

es FK conocida y no nullable, puede aportar información fuerte sobre cardinalidad de ciertos joins.

Pero:

```text
FK
≠
automatic exact join cardinality
```

debido a filtros y otros factores.

---

# 72. JoinPriorityHint

Podrá sugerir:

```text
evaluate selective relation early
```

pero no seleccionar todavía un algoritmo físico.

---

# 73. JoinPriority

No será un simple:

```text
int priority
```

sin provenance.

Preferible:

```php
final readonly class JoinPriorityHint
{
    public function __construct(
        public JoinId $joinId,
        public JoinPriority $priority,
        public CostHintSource $source,
        public HintConfidence $confidence,
        public HintStrength $strength,
    ) {}
}
```

---

# 74. Relative preferences

Algunos hints podrán expresar:

```text
prefer A before B
```

en lugar de una puntuación absoluta.

---

# 75. Preference graph

Podrá existir:

```text
HintPreferenceGraph
```

para representar relaciones como:

```text
Alternative A preferred over B
B preferred over C
```

---

# 76. Cyclic preferences

Si aparecen:

```text
A > B
B > C
C > A
```

deberán detectarse.

---

# 77. Hint conflict

Un conflicto no se resolverá mediante:

```text
last registered wins
```

---

# 78. HintConflictResolver

```php
interface HintConflictResolver
{
    public function resolve(
        CostHintConflict $conflict,
        CostHintResolutionContext $context,
    ): CostHintResolution;
}
```

---

# 79. Conflict categories

```text
compatible
redundant
refinement
contradictory
incomparable
hard conflict
```

---

# 80. Hard conflict

Ejemplo:

```text
Hint A:
    REQUIRE strategy X

Hint B:
    PROHIBIT strategy X
```

debe producir diagnóstico/error.

---

# 81. Soft conflict

Ejemplo:

```text
Provider A:
    cardinality ≈ 1M

Provider B:
    cardinality ≈ 1.2M
```

puede resolverse mediante:

```text
confidence
freshness
source policy
```

---

# 82. Hint precedence

No deberá codificarse dispersamente.

Podrá existir:

```text
CostHintPrecedencePolicy
```

---

# 83. Example precedence

Una configuración posible:

```text
proven semantic fact
>
explicit REQUIRE/PROHIBIT
>
trusted application hint
>
fresh statistics
>
schema heuristic
>
generic heuristic
```

pero será una política formal, no una regla universal oculta.

---

# 84. Facts cannot be overridden

Un hint jamás podrá contradecir un semantic fact demostrado.

Ejemplo:

```text
Semantic fact:
    cardinality <= 1

Application hint:
    cardinality ≈ 1,000,000
```

El segundo será inválido.

---

# 85. Hint merging

Hints compatibles podrán fusionarse.

Ejemplo:

```text
row count
+
row width
```

pueden formar:

```text
relation size estimate
```

---

# 86. Confidence merging

No se promediarán niveles de confianza arbitrariamente.

Cada tipo de hint tendrá política explícita.

---

# 87. HintNormalizer

Responsabilidades:

```text
canonical scopes
canonical units
canonical ranges
normalized confidence representation
normalized source identity
```

No cambiará semántica.

---

# 88. Units

Los tamaños usarán unidades internas inequívocas.

Por ejemplo:

```text
bytes
```

No strings:

```text
"12 MB"
```

en artifacts internos.

---

# 89. Probabilities

Selectivity utilizará representación validada.

```php
final readonly class Selectivity
{
    private function __construct(
        public float $value,
    ) {}
}
```

con:

```text
0 <= value <= 1
```

---

# 90. NaN e Infinity

No se permitirán como estimaciones válidas salvo que exista un tipo explícito para representar unknown/unbounded.

---

# 91. StatisticsSnapshot

Las estadísticas utilizadas durante una optimización se capturarán como snapshot lógico.

```php
final readonly class StatisticsSnapshot
{
    public function __construct(
        public StatisticsSnapshotId $id,
        public StatisticsVersion $version,
        public array $statistics,
        public StatisticsFingerprint $fingerprint,
    ) {}
}
```

---

# 92. No live statistics reads inside rules

Una OptimizationRule no deberá ejecutar:

```text
SELECT COUNT(*)
```

ni consultar catálogos directamente.

---

# 93. Explicit I/O boundary

La obtención de estadísticas ocurrirá antes de la fase pura del Optimizer o mediante una capa explícita de preparación.

---

# 94. OptimizationContext

El Optimizer recibirá:

```text
SemanticQueryArtifact
ConstraintGraph
QueryCostHintSet
StatisticsSnapshot?
CapabilitySnapshot
OptimizerConfiguration
OptimizationBudget
```

---

# 95. OptimizationContext ≠ Connection

No contendrá:

```text
PDO
Connection
Statement
ResultCursor
```

---

# 96. Staleness

Statistics pueden envejecer.

---

# 97. StatisticsFreshness

```php
enum StatisticsFreshness
{
    case FRESH;
    case ACCEPTABLE;
    case STALE;
    case VERY_STALE;
    case UNKNOWN;
}
```

---

# 98. Freshness policy

La clasificación dependerá del provider/configuración.

No se hardcodeará universalmente:

```text
older than 24 hours = stale
```

---

# 99. Stale hints

Un hint stale:

```text
does not become false
```

pero su confianza puede reducirse.

---

# 100. Expired hints

La política podrá:

```text
ignore
downgrade
warn
reject
```

dependiendo de la clase de hint.

---

# 101. Application hints

VoltStack permitirá que aplicaciones proporcionen conocimiento que la base de datos quizá no conozca.

Ejemplo:

```text
tenant_id = currentTenant
```

puede tener distribuciones particulares.

---

# 102. ApplicationHintProvider

```php
interface ApplicationCostHintProvider
{
    public function provide(
        SemanticQueryArtifact $query,
        CostHintProviderContext $context,
    ): iterable;
}
```

---

# 103. Application hints are not trusted facts

Por defecto:

```text
application hint
≠
semantic truth
```

---

# 104. Trusted source policy

Podrán existir providers con diferentes niveles de confianza configurados.

---

# 105. Developer-facing hints

Una futura API podría permitir:

```php
$query
    ->costHint(
        CardinalityHint::approximately(1000)
    );
```

pero esta API deberá ser estructurada.

---

# 106. No arbitrary strings

Evitar:

```php
$query->hint('USE INDEX users_idx');
```

en el core semántico.

---

# 107. Platform-specific escape hatch

Si se necesita un hint específico del motor deberá modelarse mediante una extensión explícita y capability-gated.

---

# 108. Hint namespaces

Podrán existir:

```text
core.cardinality
core.selectivity
core.relation_size
core.join_priority
extension.vector.index_preference
```

---

# 109. Stable HintTypeId

Extensiones utilizarán identificadores semánticos estables.

---

# 110. IndexAvailabilityHint

Puede indicar que existe una estructura potencialmente útil para determinados access patterns.

---

# 111. Index availability ≠ IndexScan

La existencia de:

```text
index(users.email)
```

no obliga al Planner a usarlo.

---

# 112. Index semantic model

El hint podrá referenciar:

```text
IndexId
IndexCapability
supported expressions
ordering properties
uniqueness
covering properties
```

cuando esa información esté disponible.

---

# 113. IndexHint ≠ vendor index hint

La arquitectura permanecerá independiente de:

```text
USE INDEX
FORCE INDEX
INDEX(...)
```

---

# 114. OrderingAvailabilityHint

Puede indicar que una fuente puede proporcionar determinado ordering de forma potencialmente económica.

---

# 115. Logical ordering property

Ejemplo:

```text
users.id ASC
```

deberá expresarse mediante semantic expressions y ordering descriptors.

No SQL strings.

---

# 116. Ordering availability ≠ final ORDER BY

Una relación físicamente disponible en cierto orden no cambia el orden observable de una query salvo que el plan preserve y utilice esa propiedad.

---

# 117. MaterializationHint

Podrá expresar:

```text
PREFER_MATERIALIZATION
AVOID_MATERIALIZATION
REQUIRE_MATERIALIZATION
PROHIBIT_MATERIALIZATION
```

---

# 118. CTE integration

Los intents del documento 49:

```text
DEFAULT
PREFER_MATERIALIZED
PREFER_INLINE
REQUIRE_MATERIALIZED
REQUIRE_INLINE
```

podrán traducirse a información consumible por Planner.

No deberán perder su fuerza semántica.

---

# 119. Intent ≠ estimate

Una preferencia de materialización no es una estimación de costo.

El sistema podrá almacenarla junto a cost hints, pero como categoría tipada diferente.

---

# 120. LocalityHint

Pensando en futuras bases distribuidas:

```text
local
same node
same region
remote region
unknown
```

---

# 121. NetworkCostHint

Podrá estimar:

```text
bytes transferred
latency class
bandwidth class
cross-region penalty
```

---

# 122. No network dependency in core optimizer

El Optimizer no consultará servicios externos para medir latencia durante la optimización.

Utilizará snapshots/hints preparados.

---

# 123. MemoryPressureHint

Podrá describir restricciones aproximadas del entorno:

```text
low
normal
high
critical
```

o budgets cuantitativos.

---

# 124. Memory pressure ≠ query memory limit

El primero describe condiciones.

El segundo puede ser una constraint de ejecución.

---

# 125. ResourceBudget

Podrá integrarse posteriormente con:

```text
Database Resource Governance System
```

---

# 126. ParallelismHint

Podrá expresar:

```text
parallelism beneficial
parallelism expensive
parallelism unavailable
```

sin decidir todavía:

```text
workers = 8
```

---

# 127. Physical planner responsibility

El número final de workers pertenece al Physical Planner/Execution Plan.

---

# 128. Cost classes

Además de valores numéricos, VoltStack podrá utilizar clases ordinales:

```php
enum RelativeCostClass
{
    case TRIVIAL;
    case VERY_LOW;
    case LOW;
    case MEDIUM;
    case HIGH;
    case VERY_HIGH;
    case EXTREME;
    case UNKNOWN;
}
```

---

# 129. Relative vs absolute cost

El sistema no asumirá que:

```text
cost = 100
```

tiene significado universal entre platforms.

---

# 130. CostVector

En lugar de reducir todo inmediatamente a un único número, podrá modelarse:

```php
final readonly class CostVector
{
    public function __construct(
        public ?float $cpu,
        public ?float $io,
        public ?float $memory,
        public ?float $network,
        public ?float $latency,
    ) {}
}
```

---

# 131. CostVector ≠ final planner cost

Representa información parcial o relativa.

El Planner podrá aplicar posteriormente:

```text
weights
resource policy
target environment
```

---

# 132. Multi-objective optimization

Una query puede preferir:

```text
lower latency
```

aunque consuma más CPU.

Otra puede preferir:

```text
lower memory
```

aunque tarde más.

La arquitectura no deberá colapsar prematuramente estas dimensiones.

---

# 133. OptimizationObjective

Podrá existir:

```php
enum OptimizationObjective
{
    case BALANCED;
    case LATENCY;
    case THROUGHPUT;
    case MEMORY;
    case IO;
    case NETWORK;
}
```

---

# 134. Objective ≠ semantic behavior

Cambiar el objetivo puede cambiar el plan.

Nunca el resultado lógico de la query.

---

# 135. Optimizer integration

El Query Optimizer podrá utilizar hints para:

```text
prioritize alternatives
bound search
rank rewrite candidates
choose exploration order
avoid obviously expensive logical forms
```

---

# 136. Search order

Ejemplo:

```text
Join Alternative A:
    estimated intermediate rows = 100

Join Alternative B:
    estimated intermediate rows = 50,000,000
```

El Optimizer puede explorar A primero.

---

# 137. Search order ≠ correctness

Si B no se explora por budget, la query seguirá siendo semánticamente válida.

Sólo puede perderse una oportunidad de optimización.

---

# 138. Hint-driven pruning

Eliminar completamente una alternativa únicamente por una estimación incierta será conservador.

Preferible:

```text
deprioritize
```

antes que:

```text
discard
```

salvo proof o policy explícita.

---

# 139. Hard constraints

`REQUIRE` y `PROHIBIT` sí pueden limitar el espacio de búsqueda.

Pero deberán validarse.

---

# 140. Optimizer heuristic score

Podrá existir:

```text
LogicalAlternativeScore
```

calculado desde múltiples hints.

---

# 141. Score ≠ semantic fact

Nunca se escribirá ese score en `ConstraintGraph`.

---

# 142. ConstraintGraph boundary

```text
ConstraintGraph
    =
provable facts

CostHintSet
    =
estimates/preferences/evidence
```

Esta separación será obligatoria.

---

# 143. Join optimizer integration

Documento 59 podrá consultar:

```text
relation cardinality
predicate selectivity
join selectivity
relation size
join preference
distribution
```

---

# 144. Predicate optimizer integration

Documento 58 podrá usar selectivity como orientación para:

```text
predicate ordering candidates
pushdown priority
```

cuando la semántica permita reordenamiento.

---

# 145. Deduplication integration

Documento 60 utilizará fingerprints para evitar alternativas repetidas.

Los cost hints no deberán cambiar:

```text
SemanticFingerprint
```

pero sí pueden participar en:

```text
OptimizationStateFingerprint
```

si modifican decisiones futuras.

---

# 146. CostHintSetFingerprint

```text
CostHintSetFingerprint
=
Hint Types
+
Scopes
+
Values/Classes
+
Confidence
+
Strength
+
Source Versions
+
Statistics Snapshot Version
+
Relevant Policies
```

---

# 147. Runtime bindings excluded

Los valores de:

```text
BindingSet
```

no participarán normalmente.

---

# 148. Parameter-sensitive optimization

Una arquitectura futura podría especializar queries según runtime values.

Ejemplo:

```text
WHERE tenant_id = ?
```

donde ciertos tenants son mucho mayores que otros.

Esto requerirá:

```text
Query Specialization
```

explícita.

No se mezclará silenciosamente con el Cost Hint System general.

---

# 149. Parameter sniffing

VoltStack no dependerá implícitamente de:

```text
runtime parameter sniffing
```

en el core optimizer.

---

# 150. Specialization artifact

En el futuro:

```text
GenericOptimizedQuery
        │
        ▼
SpecializationContext
        │
        ▼
SpecializedOptimizedQuery
```

con fingerprint propio.

---

# 151. Planner integration

El Planner recibirá:

```text
OptimizedQueryArtifact
QueryCostHintSet
StatisticsSnapshot
CapabilitySnapshot
PlanningContext
```

---

# 152. Planner responsibility

El Planner podrá transformar hints en decisiones como:

```text
scan choice
join algorithm
sort strategy
aggregate algorithm
materialization
parallelism
distribution
spill strategy
```

---

# 153. Cost model boundary

La arquitectura conceptual será:

```text
Cost Hints
Statistics
Logical Properties
Physical Properties
Environment
      │
      ▼
Physical Cost Model
      │
      ▼
Estimated Plan Cost
```

---

# 154. No physical algorithm in core hints

Evitar core hints como:

```text
UseHashJoinHint
UseNestedLoopHint
```

como modelo principal.

Preferible expresar:

```text
cardinality
distribution
ordering
preference
constraint
```

y permitir que Planner derive la estrategia.

---

# 155. Explicit physical hints

Si VoltStack soporta en el futuro:

```text
REQUIRE HASH JOIN
```

será una extensión/planner directive separada.

No se confundirá con cost evidence.

---

# 156. Cost hint lifecycle

```text
Discover
   │
   ▼
Collect
   │
   ▼
Normalize
   │
   ▼
Validate
   │
   ▼
Resolve Conflicts
   │
   ▼
Freeze
   │
   ▼
Optimizer
   │
   ▼
Planner
```

---

# 157. Collection phase

Providers registrados producirán hints candidatos.

---

# 158. Validation phase

Verificará:

```text
scope exists
target identity valid
value valid
source valid
confidence valid
strength allowed
capabilities valid
```

---

# 159. Resolution phase

Resolverá:

```text
duplicates
overlaps
conflicts
refinements
precedence
```

---

# 160. Freeze phase

`QueryCostHintSet` será inmutable durante optimización.

---

# 161. No mutation by optimization rules

Una regla no podrá hacer:

```php
$hints->setCardinality(...);
```

---

# 162. Derived hints

Si una regla produce nueva información de costo, deberá crear:

```text
DerivedCostHintSet
```

o un nuevo immutable optimization state.

---

# 163. DerivedCostHint

Debe registrar:

```text
source rule
input hints
derivation
confidence
```

---

# 164. Confidence degradation

Ejemplo:

```text
Relation cardinality confidence = HIGH
Predicate selectivity confidence = MEDIUM
```

la cardinalidad derivada no deberá declararse automáticamente `HIGH`.

---

# 165. Confidence propagation

Cada estimator tendrá reglas explícitas.

---

# 166. Estimate arithmetic

Ejemplo conceptual:

```text
RowsAfterFilter
≈
RowsBefore
×
Selectivity
```

pero deberá manejar:

```text
bounds
unknown
confidence
correlation
overflow
```

---

# 167. EstimateMath

Podrá existir:

```text
CardinalityEstimator
SelectivityEstimator
JoinCardinalityEstimator
AggregateCardinalityEstimator
SetOperationCardinalityEstimator
```

---

# 168. Estimator ≠ semantic analyzer

Los estimators no demostrarán query legality.

---

# 169. Filter cardinality

```text
Rows(filter)
≈
Rows(input)
×
Selectivity(predicate)
```

---

# 170. Join cardinality

La estimación dependerá de:

```text
join type
input cardinalities
join predicate
NDV
null fraction
constraints
distribution
correlation
```

---

# 171. Outer join lower bounds

Las propiedades semánticas del join deberán limitar las estimaciones.

Por ejemplo, ciertos outer joins no pueden estimarse por debajo de determinados bounds de su preserved side.

---

# 172. Semantic bounds first

Modelo:

```text
Statistical Estimate
       │
       ▼
Apply Proven Semantic Bounds
       │
       ▼
Valid Cardinality Estimate
```

---

# 173. Aggregate cardinality

Puede estimarse usando:

```text
grouping keys
NDV
functional dependencies
grouping sets
filters
```

---

# 174. Window cardinality

Las window functions normalmente preservan row cardinality de su input window stage.

Esto es semantic information, no estimation.

---

# 175. Set operations

`UNION ALL` puede derivar cardinalidades de forma diferente a:

```text
UNION DISTINCT
INTERSECT
EXCEPT
```

---

# 176. LIMIT

Un:

```text
LIMIT 10
```

establece un upper bound semántico:

```text
<= 10
```

aunque input cardinality sea desconocida.

---

# 177. OFFSET

No deberá estimarse ingenuamente sin considerar input cardinality.

---

# 178. Empty relation proof

Si Constraint Analysis demuestra contradicción:

```text
Rows = 0
```

es un fact.

No un estimate.

---

# 179. CostHintProvider

```php
interface CostHintProvider
{
    public function provide(
        SemanticQueryArtifact $query,
        CostHintProviderContext $context,
    ): iterable;
}
```

---

# 180. Provider categories

```text
SemanticFactCostHintProvider
SchemaCostHintProvider
StatisticsCostHintProvider
ConfigurationCostHintProvider
ApplicationCostHintProvider
ExtensionCostHintProvider
```

---

# 181. SemanticFactCostHintProvider

Adapta facts demostrados para consumidores de costos sin degradar su categoría de truth.

---

# 182. SchemaCostHintProvider

Puede usar:

```text
PK
FK
UNIQUE
indexes
column widths
types
partition metadata
```

---

# 183. StatisticsCostHintProvider

Usa `StatisticsSnapshot`.

---

# 184. ConfigurationCostHintProvider

Puede aportar:

```text
optimization objective
memory class
latency preference
```

---

# 185. ExtensionCostHintProvider

Permite nuevos tipos de datos o motores especializados.

Ejemplo:

```text
vector index selectivity
geospatial index characteristics
```

---

# 186. Provider ordering

El orden de registro no determinará precedence.

---

# 187. Provider registry

```text
CostHintProviderRegistry
```

será congelado tras bootstrap.

---

# 188. Provider purity

Durante la fase pura de optimización los providers no realizarán I/O oculto.

---

# 189. Statistics loading

Podrá existir una etapa:

```text
StatisticsPreparation
```

antes del Optimizer.

---

# 190. Lazy statistics

Si en el futuro se soporta carga lazy, deberá pasar por una boundary explícita del coordinator.

Nunca desde una OptimizationRule individual.

---

# 191. Cost hint extension SPI

Extensiones podrán registrar:

```text
HintTypeDescriptor
HintValidator
HintMerger
HintFingerprintContributor
HintExplainFormatter
Estimator
```

---

# 192. Extension isolation

Una extensión no recibirá acceso irrestricto al Container.

---

# 193. Unknown hint

Un hint desconocido no se ignorará silenciosamente si tiene:

```text
REQUIRE
PROHIBIT
```

---

# 194. Unknown informational hint

Podrá conservarse como extension metadata si existe codec/descriptor válido.

---

# 195. Serialization

Hints persistibles deberán tener:

```text
type ID
version
codec
source
scope
confidence
strength
fingerprint
```

---

# 196. Statistics serialization

No todo snapshot necesita ser persistible.

Pero cualquier cache cross-request deberá tener versioning explícito.

---

# 197. Cost hint cache

Podrá existir posteriormente:

```text
StatisticsCache
CostHintCache
```

pero no como mutable state interno del Optimizer.

---

# 198. Cache key

Podrá considerar:

```text
schema dependencies
statistics version
provider versions
semantic query fingerprint
configuration profile
```

---

# 199. Cache invalidation

Cambios relevantes deberán invalidar hints.

---

# 200. Stale cache

Nunca se confundirá:

```text
cached
```

con:

```text
fresh
```

---

# 201. Persistent runtime

En FrankenPHP:

```text
ProviderRegistry
Descriptors
Policies
```

pueden ser shared immutable services.

---

# 202. Operation-scoped state

```text
StatisticsSnapshot
QueryCostHintSet
HintResolutionState
EstimatorState
Diagnostics
```

serán operation-scoped.

---

# 203. Worker reset

No deberá quedar:

```text
current query hints
current tenant statistics
current query cardinality
```

en singletons después de una request.

---

# 204. Multitenancy

Las estadísticas pueden variar radicalmente entre tenants.

---

# 205. Tenant statistics

Cuando exista el paquete Multitenancy:

```text
TenantContext
```

podrá seleccionar un snapshot apropiado.

Pero `Quantum/Database` no dependerá directamente del paquete Multitenancy.

---

# 206. Tenant isolation

Hints de:

```text
Tenant A
```

nunca deberán contaminar optimización de:

```text
Tenant B
```

---

# 207. Tenant-aware fingerprint

Cuando statistics sean tenant-specific:

```text
StatisticsFingerprint
```

deberá incorporar una identidad de aislamiento adecuada.

No necesariamente el tenant ID en texto plano.

---

# 208. Security

Statistics pueden revelar información sensible.

Ejemplo:

```text
row counts
value frequencies
distribution
```

---

# 209. Statistics visibility

No todos los hints deberán aparecer íntegramente en:

```text
debug toolbar
logs
exceptions
telemetry
```

---

# 210. SensitiveHintClassification

Podrá existir:

```php
enum HintSensitivity
{
    case PUBLIC;
    case INTERNAL;
    case SENSITIVE;
    case RESTRICTED;
}
```

---

# 211. Explain redaction

Un explain puede mostrar:

```text
estimated selectivity: LOW
```

sin mostrar:

```text
specific sensitive histogram values
```

---

# 212. Security policies

Application hints no podrán desactivar:

```text
security predicates
tenant predicates
mandatory policy barriers
```

por razones de costo.

---

# 213. Performance ≠ authorization

Ningún:

```text
cost hint
```

podrá alterar quién tiene acceso a datos.

---

# 214. Security barriers

Si una optimization barrier prohíbe un rewrite:

```text
cheap alternative
```

no puede saltarse la barrera.

---

# 215. Hint trust

Un developer-provided hint no debe poder introducir:

```text
SQL
identifier injection
raw predicates
```

---

# 216. Structured hints only

Todos los hints core serán objetos tipados.

---

# 217. Explainability

VoltStack deberá poder explicar:

```text
why an alternative was preferred
```

---

# 218. Example explain

```text
Join Alternative J17 preferred over J22

Evidence:
  R1 estimated rows:       1,200
  R2 estimated rows:       4,800
  R3 estimated rows:  12,500,000

Predicate P7 selectivity:
  0.004
  confidence: HIGH

Estimated intermediate cardinality:
  J17: 4,900
  J22: 8,700,000

Decision:
  explore J17 first

This is an optimization preference,
not a semantic requirement.
```

---

# 219. Explain confidence

Toda estimación importante deberá poder mostrar:

```text
source
confidence
freshness
```

---

# 220. Explain conflicts

Ejemplo:

```text
Cardinality hints conflict:

StatisticsProvider:
    1.2M
    HIGH
    fresh

ApplicationProvider:
    500K
    LOW

Resolution:
    statistics estimate selected
```

---

# 221. Telemetry

Métricas posibles:

```text
cost hints collected
cost hints accepted
cost hints rejected
cost hint conflicts
statistics snapshots loaded
stale statistics used
unknown cardinalities
selectivity estimates
join estimates
estimation errors
optimizer alternatives prioritized
hard hint violations
hint resolution time
estimation time
```

---

# 222. Estimate quality telemetry

Cuando sea posible comparar:

```text
estimated rows
vs
actual rows
```

podrán generarse métricas para mejorar estimators.

---

# 223. Feedback loop

Arquitectura futura:

```text
Estimated Cardinality
        │
        ▼
Execution
        │
        ▼
Actual Cardinality
        │
        ▼
Telemetry
        │
        ▼
Statistics / Estimator Improvement
```

---

# 224. No direct self-learning mutation

El Optimizer no modificará sus propios global statistics durante ejecución.

---

# 225. Statistics update pipeline

Cualquier feedback deberá pasar por un subsistema explícito.

---

# 226. CostHintBudget

El procesamiento estará limitado.

```php
final readonly class CostHintBudget
{
    public function __construct(
        public int $maxHints,
        public int $maxHintsPerScope,
        public int $maxProviders,
        public int $maxConflicts,
        public int $maxHistogramBuckets,
        public int $maxCorrelationEntries,
        public int $maxEstimatorWorkUnits,
        public int $maxDerivedHints,
    ) {}
}
```

---

# 227. Budget exhaustion

No cambiará query semantics.

---

# 228. Missing hints

Una query deberá poder ejecutarse con:

```text
no statistics
```

---

# 229. Graceful degradation

```text
Rich Statistics
      │
      ▼
High Quality Optimization

Partial Statistics
      │
      ▼
Mixed Estimate + Heuristics

No Statistics
      │
      ▼
Safe Heuristic Optimization
```

---

# 230. No-statistics correctness

La ausencia de statistics sólo afecta:

```text
optimization quality
```

nunca:

```text
query correctness
```

---

# 231. Default heuristics

Podrán existir estimaciones conservadoras como:

```text
unknown equality selectivity
unknown range selectivity
unknown join selectivity
```

pero serán configurables/versionadas.

---

# 232. HeuristicVersion

Cambios en heuristics deberán participar en fingerprints relevantes.

---

# 233. Cost model evolution

VoltStack podrá evolucionar desde:

```text
simple heuristics
```

a:

```text
statistics-aware estimation
```

y posteriormente:

```text
multi-dimensional cost models
adaptive estimation
```

sin cambiar Query Model.

---

# 234. Optimizer profile integration

Perfiles:

```text
OFF
MINIMAL
DEFAULT
AGGRESSIVE
```

podrán controlar cuánto trabajo de estimación se realiza.

---

# 235. Profile does not alter semantics

Siempre:

```text
OptimizationProfile
≠
Query Semantics
```

---

# 236. Cost intelligence tiers

Propuesta:

```text
Tier 0
    semantic bounds only

Tier 1
    static heuristics

Tier 2
    schema-aware hints

Tier 3
    statistics-aware estimates

Tier 4
    correlation/distribution aware

Tier 5
    adaptive/feedback-assisted
```

---

# 237. V1 recommendation

VoltStack V1 puede comenzar con:

```text
Tier 0
Tier 1
Tier 2
basic Tier 3
```

sin requerir un optimizer estadístico extremadamente complejo desde el inicio.

---

# 238. Planner future compatibility

La arquitectura deberá soportar posteriormente:

```text
Cost-Based Optimizer
Memo-based Planner
Cascades-style exploration
Physical Property Enforcement
Adaptive Planning
Distributed Planning
```

---

# 239. QueryCostIntelligence

Podrá existir una facade interna:

```php
interface QueryCostIntelligence
{
    public function hintsFor(
        SemanticQueryArtifact $query,
        CostIntelligenceContext $context,
    ): QueryCostHintSet;
}
```

---

# 240. Internal facade

No deberá convertirse en:

```text
God Service
```

Los providers, estimators y resolvers permanecerán separados.

---

# 241. Error model

Errores posibles:

```text
InvalidCostHintException
UnknownHintTargetException
CostHintConflictException
InvalidSelectivityException
InvalidCardinalityException
UnsupportedHintException
RequiredHintUnsatisfiedException
ProhibitedAlternativeException
StatisticsSnapshotException
StatisticsBudgetExceededException
HintProviderException
HintExtensionException
```

---

# 242. Provider failure

Una falla de provider `INFORMATIONAL` podrá, según política:

```text
degrade gracefully
+
emit diagnostic
```

---

# 243. Required provider failure

Si una política requiere estadísticas específicas:

```text
fail explicitly
```

---

# 244. Extension failure isolation

Una extensión no deberá corromper el `QueryCostHintSet`.

Los hints serán validados antes de freeze.

---

# 245. Final validation

Antes de entregar hints al Optimizer:

```text
all targets valid
all values valid
all hard conflicts resolved
all required descriptors available
all fingerprints computable
all security policies satisfied
```

---

# 246. Cost hint directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Cost/
                    ├── Contract/
                    │   ├── QueryCostHint.php
                    │   ├── CostHintProvider.php
                    │   ├── CostHintValidator.php
                    │   ├── CostHintNormalizer.php
                    │   ├── CostHintMerger.php
                    │   ├── HintConflictResolver.php
                    │   └── QueryCostIntelligence.php
                    │
                    ├── Hint/
                    │   ├── CostHintId.php
                    │   ├── CostHintKind.php
                    │   ├── QueryCostHintSet.php
                    │   ├── CardinalityHint.php
                    │   ├── RelationSizeHint.php
                    │   ├── SelectivityHint.php
                    │   ├── PredicateSelectivityHint.php
                    │   ├── DistinctCountHint.php
                    │   ├── NullFractionHint.php
                    │   ├── DataDistributionHint.php
                    │   ├── CorrelationHint.php
                    │   ├── JoinCardinalityHint.php
                    │   ├── JoinPriorityHint.php
                    │   ├── IndexAvailabilityHint.php
                    │   ├── OrderingAvailabilityHint.php
                    │   ├── MaterializationHint.php
                    │   ├── LocalityHint.php
                    │   ├── NetworkCostHint.php
                    │   ├── MemoryPressureHint.php
                    │   └── ParallelismHint.php
                    │
                    ├── Scope/
                    │   ├── CostHintScope.php
                    │   ├── QueryHintScope.php
                    │   ├── RelationHintScope.php
                    │   ├── PredicateHintScope.php
                    │   ├── ExpressionHintScope.php
                    │   ├── JoinHintScope.php
                    │   └── LogicalAlternativeHintScope.php
                    │
                    ├── Source/
                    │   ├── CostHintSource.php
                    │   ├── CostHintSourceKind.php
                    │   └── HintProvenance.php
                    │
                    ├── Confidence/
                    │   ├── HintConfidence.php
                    │   ├── HintConfidenceLevel.php
                    │   └── ConfidencePropagationPolicy.php
                    │
                    ├── Strength/
                    │   └── HintStrength.php
                    │
                    ├── Estimate/
                    │   ├── CardinalityEstimate.php
                    │   ├── ExactCardinality.php
                    │   ├── EstimatedCardinality.php
                    │   ├── UnknownCardinality.php
                    │   ├── CardinalityRange.php
                    │   ├── Selectivity.php
                    │   ├── PredicateTruthDistribution.php
                    │   ├── RowWidthEstimate.php
                    │   ├── RelationSizeEstimate.php
                    │   ├── RelativeCostClass.php
                    │   └── CostVector.php
                    │
                    ├── Statistics/
                    │   ├── StatisticsSnapshot.php
                    │   ├── StatisticsSnapshotId.php
                    │   ├── StatisticsVersion.php
                    │   ├── StatisticsFingerprint.php
                    │   ├── StatisticsFreshness.php
                    │   ├── RelationStatistics.php
                    │   ├── ColumnStatistics.php
                    │   ├── Histogram.php
                    │   ├── FrequencyDistribution.php
                    │   └── CorrelationStatistics.php
                    │
                    ├── Provider/
                    │   ├── SemanticFactCostHintProvider.php
                    │   ├── SchemaCostHintProvider.php
                    │   ├── StatisticsCostHintProvider.php
                    │   ├── ConfigurationCostHintProvider.php
                    │   ├── ApplicationCostHintProvider.php
                    │   └── ExtensionCostHintProvider.php
                    │
                    ├── Estimator/
                    │   ├── CardinalityEstimator.php
                    │   ├── SelectivityEstimator.php
                    │   ├── JoinCardinalityEstimator.php
                    │   ├── AggregateCardinalityEstimator.php
                    │   └── SetOperationCardinalityEstimator.php
                    │
                    ├── Resolution/
                    │   ├── CostHintResolver.php
                    │   ├── CostHintConflict.php
                    │   ├── CostHintConflictResolver.php
                    │   ├── CostHintResolution.php
                    │   ├── CostHintPrecedencePolicy.php
                    │   └── HintPreferenceGraph.php
                    │
                    ├── Objective/
                    │   ├── OptimizationObjective.php
                    │   └── OptimizationObjectiveProfile.php
                    │
                    ├── Fingerprint/
                    │   ├── CostHintSetFingerprint.php
                    │   ├── HintFingerprintContributor.php
                    │   └── HeuristicVersion.php
                    │
                    ├── Registry/
                    │   ├── CostHintProviderRegistry.php
                    │   └── HintTypeRegistry.php
                    │
                    ├── Security/
                    │   ├── HintSensitivity.php
                    │   └── CostHintSecurityPolicy.php
                    │
                    ├── Budget/
                    │   └── CostHintBudget.php
                    │
                    ├── Diagnostic/
                    │   └── CostHintDiagnostic.php
                    │
                    └── Exception/
                        ├── InvalidCostHintException.php
                        ├── CostHintConflictException.php
                        ├── RequiredHintUnsatisfiedException.php
                        └── StatisticsSnapshotException.php
```

---

# 247. Flujo completo

```text
SemanticQueryArtifact
        │
        ├─────────────► ConstraintGraph
        │
        ├─────────────► SchemaView
        │
        └─────────────► Dependencies
                            │
                            ▼
                    Statistics Preparation
                            │
                            ▼
                    StatisticsSnapshot
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
          Schema         Statistics    Application /
          Hints            Hints       Extension Hints
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                     Hint Collection
                            │
                            ▼
                     Normalization
                            │
                            ▼
                       Validation
                            │
                            ▼
                    Conflict Resolution
                            │
                            ▼
                    QueryCostHintSet
                            │
                ┌───────────┴────────────┐
                ▼                        ▼
        Query Optimizer              Query Planner
                │                        │
                ▼                        ▼
       Logical Alternatives       Physical Alternatives
```

---

# 248. Ejemplo — predicate selectivity

Consulta:

```sql
SELECT *
FROM users
WHERE status = ?
  AND country = ?
```

Supóngase:

```text
Rows(users)
≈ 10,000,000

Selectivity(status)
≈ 0.10

Selectivity(country)
≈ 0.05
```

Una estimación ingenua:

```text
10,000,000
× 0.10
× 0.05
=
50,000
```

Pero si existe correlación entre ambas columnas:

```text
CorrelationHint(status, country)
```

el estimator deberá utilizarla.

---

# 249. Ejemplo — semantic bound

```sql
SELECT *
FROM users
WHERE id = ?
```

Si `id` es UNIQUE:

```text
Semantic/Constraint Fact:
    result cardinality <= 1
```

Aunque statistics indiquen:

```text
users rows ≈ 50,000,000
```

el upper bound seguirá siendo:

```text
1
```

---

# 250. Ejemplo — join ordering

```text
A = 100 rows
B = 1,000 rows
C = 100,000,000 rows
```

Predicates:

```text
A ⋈ B
selectivity ≈ 0.01

B ⋈ C
selectivity ≈ 0.50
```

El Join Optimizer puede priorizar:

```text
(A ⋈ B) ⋈ C
```

sin que el Cost Hint System determine todavía:

```text
HashJoin
NestedLoop
MergeJoin
```

---

# 251. Ejemplo — stale statistics

```text
Relation:
    orders

Statistics:
    rows ≈ 1,000,000

Age:
    VERY_STALE

Application Hint:
    recent migration increased data ~10x
```

El resolver puede:

```text
downgrade statistics confidence
+
prefer application estimate
+
emit diagnostic
```

según política.

---

# 252. Ejemplo — hard conflict

```text
Application:
    REQUIRE materialization

Extension:
    PROHIBIT materialization
```

Resultado:

```text
CostHintConflictException
```

No:

```text
last hint wins
```

---

# 253. Ejemplo — missing statistics

```text
Rows(A) = UNKNOWN
Rows(B) = UNKNOWN
```

El Optimizer podrá utilizar:

```text
schema constraints
static heuristics
structural facts
```

y continuar.

---

# 254. Invariantes arquitectónicos

## DB-COST-001

Cost Hint no será Semantic Fact.

## DB-COST-002

Statistics no serán Semantic Facts.

## DB-COST-003

Cardinality Estimate será distinta de Cardinality Fact.

## DB-COST-004

Selectivity Estimate será distinta de Predicate Truth.

## DB-COST-005

Cost Hint será distinto de Physical Cost.

## DB-COST-006

Cost Hint será distinto de Physical Plan.

## DB-COST-007

Cost Hint será distinto de SQL optimizer hint syntax.

## DB-COST-008

El Cost Hint System no generará SQL.

## DB-COST-009

El Cost Hint System no ejecutará queries.

## DB-COST-010

El Cost Hint System no abrirá conexiones durante optimización pura.

## DB-COST-011

Optimization Rules no consultarán live statistics directamente.

## DB-COST-012

Statistics serán proporcionadas mediante snapshots explícitos.

## DB-COST-013

StatisticsSnapshot será inmutable durante una optimización.

## DB-COST-014

Todo hint tendrá scope explícito.

## DB-COST-015

Todo hint tendrá source/provenance.

## DB-COST-016

Todo hint estimativo tendrá confidence explícita.

## DB-COST-017

HintConfidence será distinta de HintStrength.

## DB-COST-018

INFORMATIONAL no impondrá una decisión.

## DB-COST-019

PREFER será distinto de REQUIRE.

## DB-COST-020

AVOID será distinto de PROHIBIT.

## DB-COST-021

REQUIRE no se ignorará silenciosamente.

## DB-COST-022

PROHIBIT no se ignorará silenciosamente.

## DB-COST-023

Hard hint conflicts producirán resolución explícita o error.

## DB-COST-024

No existirá last-one-wins para conflictos.

## DB-COST-025

Semantic facts no podrán ser sobrescritos por hints.

## DB-COST-026

Unknown cardinality será explícita.

## DB-COST-027

Unknown selectivity será explícita.

## DB-COST-028

Unknown no será representado como cero.

## DB-COST-029

Cardinality podrá expresarse mediante bounds.

## DB-COST-030

Exact cardinality sólo se declarará cuando sea demostrable.

## DB-COST-031

Estimated cardinality preservará su carácter estimativo.

## DB-COST-032

Relation cardinality será distinta de relation byte size.

## DB-COST-033

Row width podrá estimarse independientemente.

## DB-COST-034

Selectivity estará dentro de un dominio validado.

## DB-COST-035

NaN no será una selectivity válida.

## DB-COST-036

Infinity no será una cardinality normal válida.

## DB-COST-037

SQL UNKNOWN podrá modelarse en truth distributions.

## DB-COST-038

NDV será distinto de row count.

## DB-COST-039

Histograms serán platform-neutral en el core.

## DB-COST-040

Optimizer no consumirá directamente estructuras estadísticas vendor-specific.

## DB-COST-041

Platform adapters traducirán statistics al modelo VoltStack.

## DB-COST-042

Data skew podrá representarse.

## DB-COST-043

Predicate correlation podrá representarse.

## DB-COST-044

Independence assumptions serán explícitas.

## DB-COST-045

Join estimates utilizarán join semantics.

## DB-COST-046

Constraint facts podrán limitar estimates.

## DB-COST-047

FK no implicará automáticamente cardinalidad exacta.

## DB-COST-048

JoinPriorityHint no elegirá physical join algorithm.

## DB-COST-049

IndexAvailabilityHint no elegirá IndexScan.

## DB-COST-050

OrderingAvailabilityHint no garantizará query result ordering.

## DB-COST-051

Materialization preference será distinta de materialization decision.

## DB-COST-052

CTE REQUIRE_MATERIALIZED conservará su fuerza.

## DB-COST-053

Locality podrá modelarse sin dependencia distribuida obligatoria.

## DB-COST-054

Network cost será multidimensional cuando sea necesario.

## DB-COST-055

Memory pressure no será confundida con query memory limit.

## DB-COST-056

Parallelism hint no fijará automáticamente worker count.

## DB-COST-057

Cost podrá ser multidimensional.

## DB-COST-058

CPU cost será distinto de I/O cost.

## DB-COST-059

Memory cost será distinto de network cost.

## DB-COST-060

Latency objective será distinto de throughput objective.

## DB-COST-061

Optimization objective no cambiará query semantics.

## DB-COST-062

Optimizer podrá utilizar hints para exploration order.

## DB-COST-063

Exploration order no cambiará correctness.

## DB-COST-064

Un estimate incierto no descartará automáticamente una alternativa válida.

## DB-COST-065

Hard constraints podrán limitar search space.

## DB-COST-066

Hard constraints serán validadas.

## DB-COST-067

Optimizer scores no entrarán en ConstraintGraph.

## DB-COST-068

ConstraintGraph contendrá facts demostrables.

## DB-COST-069

CostHintSet contendrá evidence/estimates/preferences.

## DB-COST-070

Join Optimizer podrá consumir cost hints.

## DB-COST-071

Predicate Optimizer podrá consumir cost hints.

## DB-COST-072

Deduplication fingerprints semánticos no dependerán de cost hints.

## DB-COST-073

Optimization state fingerprints podrán depender de hints relevantes.

## DB-COST-074

CostHintSet tendrá fingerprint propio.

## DB-COST-075

Runtime bindings no formarán parte del CostHintSet general.

## DB-COST-076

Parameter-sensitive optimization requerirá sistema explícito.

## DB-COST-077

Core optimizer no realizará parameter sniffing implícito.

## DB-COST-078

Planner consumirá hints mediante contratos explícitos.

## DB-COST-079

Planner será responsable de physical strategy selection.

## DB-COST-080

Cost Hint System no seleccionará HashJoin.

## DB-COST-081

Cost Hint System no seleccionará MergeJoin.

## DB-COST-082

Cost Hint System no seleccionará NestedLoop.

## DB-COST-083

Cost Hint System no seleccionará physical scan strategy.

## DB-COST-084

Cost Hint System no decidirá spill strategy.

## DB-COST-085

Cost Hint System no decidirá final parallelism.

## DB-COST-086

Hint lifecycle será explícito.

## DB-COST-087

Hints serán validados antes del Optimizer.

## DB-COST-088

Hints serán normalizados antes de resolution.

## DB-COST-089

Resolved CostHintSet será inmutable.

## DB-COST-090

Optimization Rules no mutarán CostHintSet.

## DB-COST-091

Derived hints serán artifacts explícitos.

## DB-COST-092

Derived hints tendrán provenance.

## DB-COST-093

Confidence propagation será explícita.

## DB-COST-094

Estimate arithmetic manejará unknown.

## DB-COST-095

Estimate arithmetic manejará bounds.

## DB-COST-096

Estimate arithmetic manejará confidence.

## DB-COST-097

Estimate arithmetic respetará semantic bounds.

## DB-COST-098

Window cardinality preservation utilizará semantic facts.

## DB-COST-099

LIMIT producirá upper bounds cuando corresponda.

## DB-COST-100

Contradiction-proven empty relation tendrá cardinality fact 0.

## DB-COST-101

CostHintProviders estarán registrados explícitamente.

## DB-COST-102

Provider registration order no definirá precedence.

## DB-COST-103

Provider registry se congelará después de bootstrap.

## DB-COST-104

Providers usados durante optimization pura no harán I/O oculto.

## DB-COST-105

Statistics loading tendrá boundary explícita.

## DB-COST-106

Extensions podrán registrar nuevos hint types.

## DB-COST-107

Extension hints tendrán stable semantic IDs.

## DB-COST-108

Unknown hard hints no se ignorarán.

## DB-COST-109

Serializable hints serán versionados.

## DB-COST-110

Cross-request hint caches serán explícitos.

## DB-COST-111

Cached statistics conservarán freshness metadata.

## DB-COST-112

Cached no significará fresh.

## DB-COST-113

Persistent runtime compartirá sólo registries/descriptors inmutables.

## DB-COST-114

QueryCostHintSet será operation-scoped.

## DB-COST-115

StatisticsSnapshot será operation-scoped cuando corresponda.

## DB-COST-116

Tenant statistics estarán aisladas.

## DB-COST-117

Tenant A hints no contaminarán Tenant B.

## DB-COST-118

Database core no dependerá directamente de Multitenancy.

## DB-COST-119

Statistics podrán clasificarse como sensitive.

## DB-COST-120

Sensitive statistics serán redactadas en diagnostics.

## DB-COST-121

Cost hints no podrán remover security predicates.

## DB-COST-122

Cost hints no podrán saltar security barriers.

## DB-COST-123

Cost hints no modificarán authorization semantics.

## DB-COST-124

Developer hints serán estructurados.

## DB-COST-125

Developer hints no aceptarán SQL arbitrario en el core.

## DB-COST-126

Explain mostrará source/confidence cuando sea relevante.

## DB-COST-127

Explain distinguirá facts de estimates.

## DB-COST-128

Telemetry distinguirá estimated y actual cardinality.

## DB-COST-129

Feedback de ejecución no mutará optimizer global state directamente.

## DB-COST-130

Feedback utilizará pipeline explícito.

## DB-COST-131

Cost hint processing será bounded.

## DB-COST-132

Budget exhaustion no cambiará semantics.

## DB-COST-133

La ausencia de statistics no impedirá ejecutar una query válida.

## DB-COST-134

No-statistics mode utilizará heuristics seguras.

## DB-COST-135

Heuristics serán versionadas.

## DB-COST-136

Optimizer profiles podrán variar esfuerzo de estimation.

## DB-COST-137

Optimizer profile no cambiará semantics.

## DB-COST-138

Cost intelligence podrá evolucionar independientemente del Query Model.

## DB-COST-139

V1 no requerirá un cost-based optimizer completo.

## DB-COST-140

La arquitectura será compatible con un futuro Cascades-style planner.

## DB-COST-141

La arquitectura será compatible con adaptive planning.

## DB-COST-142

La arquitectura será compatible con distributed planning.

## DB-COST-143

Cost intelligence no será un God Service.

## DB-COST-144

Provider failures serán gobernados por policy.

## DB-COST-145

Invalid hints nunca entrarán al Optimizer.

## DB-COST-146

Hint fingerprints excluirán secretos cuando sea posible.

## DB-COST-147

Statistics fingerprints serán versionados.

## DB-COST-148

Cost Hint System será deterministic para mismos inputs/snapshots/policies.

## DB-COST-149

Cost Hint System será persistent-runtime safe.

## DB-COST-150

Query correctness tendrá prioridad sobre cualquier cost estimate.

---

# 255. Anti-patterns

## 255.1 `SELECT COUNT(*)` desde una OptimizationRule

**Rechazado.**

---

## 255.2 Statistics live dentro del Optimizer

```php
$connection->query(...);
```

**Rechazado.**

---

## 255.3 Tratar estimate como fact

```text
estimated rows = 0
→ relation is provably empty
```

**Rechazado.**

---

## 255.4 Tratar FK como cardinalidad exacta

**Rechazado.**

---

## 255.5 Un único número universal de costo

```text
cost = 123
```

para todos los motores y recursos.

**Rechazado como arquitectura base.**

---

## 255.6 Hint string vendor-specific

```php
$query->hint('USE INDEX(foo)');
```

como API core.

**Rechazado.**

---

## 255.7 `last hint wins`

**Rechazado.**

---

## 255.8 Ignorar REQUIRE

**Rechazado.**

---

## 255.9 Ignorar PROHIBIT

**Rechazado.**

---

## 255.10 Sobrescribir semantic facts con application hints

**Rechazado.**

---

## 255.11 Asumir independencia estadística universal

```text
P(A AND B) = P(A) × P(B)
```

sin modelar que es una heuristic.

**Rechazado.**

---

## 255.12 Usar statistics vendor-specific en reglas

**Rechazado.**

---

## 255.13 Guardar hints del tenant actual en singleton

**Rechazado.**

---

## 255.14 Mostrar histogramas sensibles sin redacción

**Rechazado.**

---

## 255.15 Desactivar security predicate porque parece costoso

**Rechazado.**

---

## 255.16 Elegir HashJoin dentro del Cost Hint System

**Rechazado.**

---

## 255.17 Confundir index availability con index selection

**Rechazado.**

---

## 255.18 Confundir materialization preference con physical decision

**Rechazado.**

---

# 256. Decisión arquitectónica final

VoltStack implementará el `Query Cost Hint System` como una capa de:

```text
Cost Intelligence
```

situada entre:

```text
Semantic Query Analysis
```

y:

```text
Optimizer / Planner
```

con una separación estricta entre:

```text
Facts
Estimates
Statistics
Hints
Preferences
Constraints
Physical Costs
Physical Decisions
```

La fórmula conceptual será:

```text
Query Cost Intelligence
=
Semantic Bounds
+
Schema Evidence
+
Statistics
+
Heuristics
+
Application Knowledge
+
Extension Knowledge
+
Confidence
+
Provenance
```

mientras:

```text
Optimization Decision
=
Semantic Equivalence
+
Optimization Rules
+
Cost Intelligence
+
Optimization Budget
```

y posteriormente:

```text
Physical Planning Decision
=
Optimized Logical Query
+
Logical Properties
+
Physical Properties
+
Cost Intelligence
+
Platform Capabilities
+
Resource Constraints
+
Physical Cost Model
```

---

# 257. Principio central

```text
Cost Estimate
≠
Semantic Truth
```

y:

```text
Cost Hint
≠
Physical Plan
```

Por tanto:

> **VoltStack utilizará información de costo para orientar la búsqueda de mejores alternativas, pero nunca permitirá que una estimación estadística modifique el significado lógico de una consulta.**

---

# 258. Resultado arquitectónico

```text
                    Semantic Query Artifact
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
       Constraint Facts   Schema Evidence   Dependencies
              │               │                │
              └───────────────┼────────────────┘
                              ▼
                    Statistics Snapshot
                              │
              ┌───────────────┼─────────────────┐
              ▼               ▼                 ▼
         Statistics       Heuristics       App/Extension
              │               │                 │
              └───────────────┼─────────────────┘
                              ▼
                      Cost Hint System
                              │
                              ▼
                      QueryCostHintSet
                              │
                  ┌───────────┴────────────┐
                  ▼                        ▼
            Query Optimizer          Query Planner
                  │                        │
                  ▼                        ▼
        Better Logical Form        Physical Plan Choice
```

---

# 259. Cierre del área de optimización previa al Planner

Con los documentos:

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
```

VoltStack dispone ahora conceptualmente de:

```text
SemanticQueryArtifact
        │
        ▼
Rewrite Engine
        │
        ▼
Optimization Rule Engine
        │
        ├── Predicate Optimization
        ├── Join Optimization
        ├── Deduplication
        └── Cost Intelligence
        │
        ▼
OptimizedQueryArtifact
```

El siguiente paso ya no consiste simplemente en transformar la consulta.

Consiste en responder:

```text
¿Cómo convertimos esta consulta lógica optimizada
en un plan explícito de ejecución?
```

Esa responsabilidad pertenece al:

```text
Query Planner
```

---

# 260. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md               ← actual
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 261. Siguiente documento

```text
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
```

Este documento deberá establecer la frontera formal:

```text
Optimizer
    =
Which equivalent logical form is preferable?

Planner
    =
How can that logical query be executed?
```

y definir la transición:

```text
OptimizedQueryArtifact
        │
        ▼
Query Planner
        │
        ├── Logical Planning
        ├── Physical Alternative Generation
        ├── Physical Property Analysis
        ├── Access Path Selection
        ├── Join Strategy Selection
        ├── Aggregate Strategy Selection
        ├── Sort / Ordering Planning
        ├── Materialization Planning
        ├── Parallelism Planning
        ├── Distribution Planning
        ├── Resource Planning
        └── Cost-Based Plan Selection
        │
        ▼
ExecutionPlan
```

manteniendo todavía separados los tres artifacts fundamentales:

```text
LogicalQueryPlan
        ≠
PhysicalQueryPlan
        ≠
ExecutionPlan
```

que serán formalizados en los documentos:

```text
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```