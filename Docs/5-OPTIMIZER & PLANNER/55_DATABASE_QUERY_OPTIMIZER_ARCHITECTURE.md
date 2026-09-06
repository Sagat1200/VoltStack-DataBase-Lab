# 55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md

# VoltStack Quantum Database
## Query Optimizer Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 55 — Query Optimizer Architecture  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer  
**Versión:** 1.0

---

# 1. Propósito

`Query Optimizer Architecture` define la arquitectura general del sistema encargado de transformar una consulta semánticamente válida en una representación equivalente más conveniente para planificación y ejecución.

El Optimizer recibe:

```text
SemanticQueryArtifact
```

y produce:

```text
OptimizedQueryArtifact
```

preservando obligatoriamente el significado observable de la consulta.

Su responsabilidad puede expresarse como:

```text
Semantic Query
      │
      ▼
Equivalent Query Transformations
      │
      ▼
Optimization
      │
      ▼
Optimized Semantic Query
```

La regla fundamental será:

> **El Optimizer puede cambiar la forma de una consulta, pero nunca su significado observable.**

---

# 2. Posición dentro del Query Engine

La arquitectura completa mantiene:

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
┌──────────────────────────┐
│     QUERY OPTIMIZER      │
└──────────────────────────┘
     │
     ▼
OptimizedQueryArtifact
     │
     ▼
Query Planner
     │
     ▼
Logical / Physical Plan
     │
     ▼
SQL Compiler
     │
     ▼
Executor
```

---

# 3. Frontera conceptual principal

VoltStack deberá distinguir estrictamente:

```text
Normalization
≠
Semantic Analysis
≠
Optimization
≠
Planning
≠
Compilation
≠
Execution
```

---

# 4. Semantic Analysis vs Optimizer

Semantic Analysis responde:

```text
What does this query mean?
```

Optimizer responde:

```text
Which semantically equivalent
representation is preferable?
```

Ejemplo:

```sql
WHERE active = true
AND true
```

Semantic Analysis determina:

```text
active = true
AND true

is a valid Boolean predicate
```

Optimizer puede transformarlo a:

```sql
WHERE active = true
```

---

# 5. Optimizer vs Planner

Optimizer responde:

```text
Which equivalent query representation
should we use?
```

Planner responde:

```text
How should this representation
be executed?
```

Ejemplo:

```text
A JOIN B
```

Optimizer podría decidir:

```text
B JOIN A
```

si demuestra equivalencia.

Pero decidir:

```text
Hash Join
Merge Join
Nested Loop Join
```

pertenece al Planner.

---

# 6. Optimizer vs Compiler

El Optimizer no genera:

```text
SELECT ...
FROM ...
WHERE ...
```

El Compiler es responsable de transformar una representación planificada a sintaxis del target.

---

# 7. Regla maestra

Toda transformación deberá cumplir:

```text
Semantics(Q)
=
Semantics(Optimize(Q))
```

dentro del contexto semántico relevante.

Formalmente:

```text
Q ≡ O(Q)
```

donde:

```text
Q
=
original semantic query

O
=
optimizer

O(Q)
=
optimized query
```

---

# 8. Equivalencia observable

La equivalencia no significa únicamente producir las mismas filas.

Debe considerar, cuando sean observables:

```text
row values
duplicate multiplicity
NULL semantics
ordering guarantees
cardinality
locking behavior
volatility
side-effect-sensitive expressions
error behavior where contractually relevant
security policies
tenant isolation
correlation semantics
recursive semantics
```

---

# 9. SQL no es álgebra booleana clásica

El Optimizer deberá respetar:

```text
SQL Three-Valued Logic
```

con:

```text
TRUE
FALSE
UNKNOWN
```

Por ello transformaciones booleanas aparentemente triviales pueden no ser válidas en todos los contextos.

---

# 10. SQL no es álgebra de conjuntos pura

Las queries operan frecuentemente sobre:

```text
bags / multisets
```

y no únicamente sobre sets.

Por tanto:

```text
duplicate multiplicity
```

forma parte de la semántica.

---

# 11. NULL

El Optimizer nunca deberá asumir:

```text
NULL = NULL
```

ni aplicar equivalencias propias de lenguajes convencionales cuando SQL tenga reglas diferentes.

---

# 12. Semantic Artifact como entrada

El Optimizer no deberá reinterpretar el AST desde cero.

Su entrada autoritativa será:

```text
SemanticQueryArtifact
```

definido por los documentos 35–42.

---

# 13. Información disponible

El artifact puede proporcionar:

```text
SemanticQueryGraph
ScopeTable
SymbolTable
RelationSemanticTable
ExpressionSemanticTable
PredicateSemanticTable
ParameterSemanticTable
QueryTypeTable
ConstraintGraph
DependencyGraph
CorrelationDescriptors
CapabilityRequirements
Lineage
Volatility
Security Metadata
Portability Information
```

---

# 14. Optimizer consumes facts

El Optimizer deberá consumir hechos ya demostrados.

Ejemplo:

```text
ConstraintGraph says:

users.id
IS NOT NULL
```

Entonces ciertas simplificaciones podrán utilizar esa información.

---

# 15. Optimizer shall not invent facts

Si:

```text
ConstraintGraph
```

no puede demostrar una propiedad:

```text
UNKNOWN
```

será la interpretación correcta.

No:

```text
probably true
```

---

# 16. OptimizedQueryArtifact

Modelo conceptual:

```php
final readonly class OptimizedQueryArtifact
{
    public function __construct(
        public QueryModel $query,
        public SemanticQueryGraph $semanticGraph,
        public QuerySemanticTables $semanticTables,
        public ConstraintGraph $constraints,
        public OptimizationTrace $trace,
        public OptimizationStatistics $statistics,
        public OptimizationFingerprint $fingerprint,
    ) {}
}
```

---

# 17. AST y optimización

El Optimizer puede producir una nueva estructura query.

No deberá mutar el artifact original.

```text
SemanticQueryArtifact
        │
        │ immutable
        ▼
Optimizer
        │
        ▼
New OptimizedQueryArtifact
```

---

# 18. Inmutabilidad

Regla:

```text
Optimize(Q)
must not mutate Q
```

Esto permite:

```text
cache reuse
parallel optimization
debugging
before/after comparison
deterministic fingerprints
persistent-runtime safety
```

---

# 19. Optimizer architecture

```text
QueryOptimizer
│
├── OptimizationCoordinator
├── OptimizationPipeline
├── OptimizationPhaseRegistry
├── OptimizationRuleRegistry
├── OptimizationScheduler
├── EquivalenceVerifier
├── OptimizationBudget
├── OptimizationTrace
├── OptimizationStatistics
├── OptimizationBarrierSystem
└── ExtensionOptimizationRegistry
```

---

# 20. QueryOptimizer

Interfaz conceptual:

```php
interface QueryOptimizer
{
    public function optimize(
        SemanticQueryArtifact $query,
        QueryOptimizationContext $context,
    ): OptimizedQueryArtifact;
}
```

---

# 21. QueryOptimizationContext

Debe contener únicamente información necesaria para optimización.

Ejemplo:

```php
final readonly class QueryOptimizationContext
{
    public function __construct(
        public OptimizationProfile $profile,
        public OptimizationBudget $budget,
        public CapabilitySnapshot $capabilities,
        public OptimizationRuleSet $rules,
        public OptimizationTelemetryContext $telemetry,
    ) {}
}
```

---

# 22. Context ≠ Service Locator

`QueryOptimizationContext` no deberá convertirse en:

```text
Container
Connection
Request
EntityManager
```

---

# 23. No database I/O

El Optimizer no deberá abrir conexiones para preguntar:

```text
Does this table exist?
How many rows does it have?
What indexes exist?
```

La información necesaria deberá entrar mediante snapshots explícitos cuando corresponda.

---

# 24. Estadísticas externas

En futuras fases cost-based, podrán existir:

```text
DatabaseStatisticsSnapshot
```

con:

```text
table cardinality
column statistics
histograms
index metadata
selectivity estimates
```

pero será entrada explícita.

---

# 25. No hidden introspection

Prohibido:

```text
Optimizer
→ Connection
→ INFORMATION_SCHEMA
```

durante una optimización normal.

---

# 26. Optimization Pipeline

La optimización deberá organizarse en fases explícitas.

Ejemplo conceptual:

```text
SemanticQueryArtifact
       │
       ▼
Canonical Optimization Preparation
       │
       ▼
Expression Optimization
       │
       ▼
Predicate Optimization
       │
       ▼
Relational Optimization
       │
       ▼
Subquery Optimization
       │
       ▼
Join Optimization
       │
       ▼
Aggregation Optimization
       │
       ▼
Set Operation Optimization
       │
       ▼
Window Optimization
       │
       ▼
Projection Optimization
       │
       ▼
Final Simplification
       │
       ▼
OptimizedQueryArtifact
```

---

# 27. Pipeline ≠ rigid implementation order

La lista anterior representa dominios.

La ejecución real podrá usar:

```text
phase DAG
```

y:

```text
bounded fixpoint groups
```

cuando varias reglas interactúen.

---

# 28. OptimizationPhase

Modelo:

```php
interface OptimizationPhase
{
    public function id(): OptimizationPhaseId;

    public function optimize(
        OptimizationWorkingArtifact $artifact,
        OptimizationPhaseContext $context,
    ): OptimizationPhaseResult;
}
```

---

# 29. OptimizationPhaseId

Ejemplos:

```text
expression.simplification
predicate.simplification
predicate.pushdown
projection.pruning
join.reordering
join.elimination
subquery.decorrelation
cte.optimization
set.optimization
aggregate.optimization
window.optimization
```

---

# 30. Rule-based architecture

Cada phase podrá ejecutar:

```text
OptimizationRule
```

---

# 31. OptimizationRule

Conceptualmente:

```php
interface OptimizationRule
{
    public function id(): OptimizationRuleId;

    public function matches(
        OptimizationNode $node,
        OptimizationRuleContext $context,
    ): bool;

    public function apply(
        OptimizationNode $node,
        OptimizationRuleContext $context,
    ): OptimizationRuleResult;
}
```

---

# 32. Rule responsibilities

Una regla deberá declarar:

```text
identity
version
target node kinds
preconditions
required semantic facts
barriers
transformation
equivalence rationale
cost class
budget impact
```

---

# 33. Rule example

```text
Rule:
    RemoveTrueConjunct

Input:
    P AND TRUE

Output:
    P

Precondition:
    valid predicate semantics

Equivalence:
    SQL 3VL compatible

Cost:
    constant
```

---

# 34. Rewrite vs Optimization

VoltStack distinguirá:

```text
Query Rewrite
```

de:

```text
Query Optimization
```

---

# 35. Rewrite

Una rewrite es una transformación semánticamente equivalente.

Ejemplo:

```text
NOT (NOT P)
→
P
```

cuando las reglas semánticas lo permiten.

---

# 36. Optimization

Optimization selecciona una transformación porque se espera que produzca una forma más conveniente.

Por tanto:

```text
Rewrite
=
equivalent transformation

Optimization
=
equivalent transformation
+
preference criterion
```

---

# 37. Document separation

El sistema detallado de rewrites será definido en:

```text
56_DATABASE_QUERY_REWRITE_SYSTEM.md
```

---

# 38. Rule system

La administración formal de reglas será definida en:

```text
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
```

---

# 39. Determinismo

Con:

```text
same semantic artifact
same optimizer version
same rules
same capabilities
same statistics snapshot
same optimization profile
```

deberá obtenerse:

```text
same optimized artifact
```

---

# 40. No random rule selection

No deberá existir:

```text
shuffle($rules)
```

ni selección aleatoria implícita.

---

# 41. Deterministic tie-breaking

Cuando existan alternativas equivalentes con igual preferencia:

```text
stable rule ordering
stable node identity
stable structural fingerprint
```

podrán resolver el empate.

---

# 42. Rule ordering

El orden no deberá depender accidentalmente de:

```text
Composer package discovery order
PHP array insertion accidents
filesystem order
```

---

# 43. Rule dependencies

Una regla podrá declarar:

```text
requires
runs-before
runs-after
conflicts-with
```

---

# 44. Optimization DAG

El coordinator construirá:

```text
Optimization Dependency DAG
```

cuando corresponda.

---

# 45. Cyclic rule dependencies

Un ciclo no declarado como fixpoint group deberá fallar durante bootstrap.

---

# 46. Fixpoint optimization

Algunas transformaciones pueden habilitar otras.

Ejemplo:

```text
Predicate Simplification
        │
        ▼
Join Simplification
        │
        ▼
Predicate Pushdown
        │
        ▼
More Predicate Simplification
```

---

# 47. Fixpoint group

Podrá definirse:

```text
RelationalSimplificationFixpoint
```

---

# 48. Fixpoint algorithm

Conceptualmente:

```text
state = initial

repeat:
    previous = fingerprint(state)

    state = applyRules(state)

until:
    fingerprint(state) == previous
    OR budget exhausted
```

---

# 49. Bounded convergence

Nunca:

```text
while (changed) {
    optimize();
}
```

sin límites.

---

# 50. OptimizationBudget

Modelo conceptual:

```php
final readonly class OptimizationBudget
{
    public function __construct(
        public int $maxRuleApplications,
        public int $maxIterations,
        public int $maxGeneratedNodes,
        public int $maxAlternatives,
        public int $maxJoinSearchStates,
        public int $maxDecorrelationAttempts,
        public int $maxOptimizationTimeUnits,
    ) {}
}
```

---

# 51. Time budget

Preferentemente el core determinista deberá usar:

```text
work units
```

y no depender exclusivamente de wall-clock time.

---

# 52. Why

Wall-clock puede variar por:

```text
CPU load
runtime scheduling
worker contention
environment
```

y perjudicar reproducibilidad.

---

# 53. Wall-clock safeguard

Puede existir además un límite defensivo externo.

Pero:

```text
deterministic work budget
```

será preferible para decisiones normales.

---

# 54. Budget exhaustion

Al agotarse el budget:

```text
Optimizer
```

deberá producir una forma válida conocida.

No deberá dejar:

```text
partially corrupted artifact
```

---

# 55. Optimization completeness

El Optimizer no necesita encontrar siempre:

```text
globally optimal query
```

Debe encontrar:

```text
valid equivalent query
within bounded resources
```

---

# 56. Safe fallback

La forma semántica original constituye una posible fallback válida.

Por tanto:

```text
Optimization failure to improve
≠
Query failure
```

salvo que exista un error interno o requirement explícito.

---

# 57. Rule failure

Una regla que no aplica deberá devolver:

```text
NO_CHANGE
```

no una excepción.

---

# 58. Internal invariant violation

Sí deberá fallar cuando detecte:

```text
invalid semantic artifact
impossible node state
broken extension contract
non-equivalent transformation evidence
```

---

# 59. Optimization barriers

No toda estructura puede cruzarse o transformarse libremente.

VoltStack tendrá:

```text
OptimizationBarrier
```

---

# 60. Barrier types

Ejemplos:

```text
RAW
SECURITY
VOLATILITY
ORDERING
LIMIT
OFFSET
DISTINCT
AGGREGATION
WINDOW
LOCKING
CORRELATION
RECURSION
MATERIALIZATION
EXTENSION
PLATFORM
SIDE_EFFECT
```

---

# 61. Raw barrier

Un Raw expression puede impedir:

```text
expression reasoning
predicate pushdown
constant folding
dependency inference
```

dependiendo de su semantic contract.

---

# 62. Raw ≠ total barrier always

Si Raw declara información estructurada fiable:

```text
type
dependencies
volatility
capabilities
```

algunas optimizaciones aún podrán ocurrir alrededor de él.

---

# 63. Security barrier

Predicates introducidos por:

```text
authorization
multitenancy
row-level security
```

pueden requerir restricciones especiales de movimiento.

---

# 64. Security provenance

El Optimizer deberá conservar:

```text
MANDATORY
SECURITY_GENERATED
TENANT_GENERATED
```

provenance.

---

# 65. No security predicate elimination

Una regla no podrá eliminar un predicate obligatorio sólo porque parezca redundante estructuralmente.

Necesitará prueba autorizada por las políticas correspondientes.

---

# 66. Volatility

Expressions podrán clasificarse conceptualmente:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

---

# 67. Volatile expressions

El Optimizer no deberá:

```text
duplicate
remove
reorder across observable boundaries
pre-evaluate
```

una expresión volatile sin garantía semántica.

---

# 68. Example

Una expresión equivalente a:

```text
random()
```

no puede duplicarse ingenuamente.

---

# 69. Constant folding

El Optimizer podrá evaluar expresiones constantes cuando:

```text
all operands are compile-time constants
function semantics are known
function is safe to fold
result is target-independent or explicitly modeled
```

---

# 70. Constant folding ≠ runtime parameter evaluation

Un:

```text
ParameterNode
```

no se convierte en constante porque el BindingSet actual tenga un valor.

---

# 71. Runtime bindings

Regla:

```text
Optimizer
must not inspect runtime parameter values
```

en el pipeline general cacheable.

---

# 72. Specialized adaptive optimization

Una futura fase runtime-adaptive podría existir separadamente.

Pero no deberá contaminar:

```text
canonical query optimization
```

---

# 73. Expression simplification

Posibles transformaciones:

```text
x + 0 → x
x * 1 → x
NOT NOT P → P
constant function folding
redundant cast removal
```

sólo cuando Query Type System y semantic rules demuestren equivalencia.

---

# 74. Overflow and numeric semantics

No se podrá asumir que:

```text
(x + 1) - 1
=
x
```

para todos los tipos.

Deben considerarse:

```text
overflow
precision
decimal scale
floating-point semantics
platform behavior
```

---

# 75. Predicate simplification

Será desarrollado detalladamente en:

```text
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
```

---

# 76. Examples

```text
P AND TRUE
→ P

P OR FALSE
→ P
```

cuando SQL 3VL y contexto lo permitan.

---

# 77. Contradiction elimination

Si Constraint Analysis demuestra:

```text
x > 10
AND
x < 5
```

el Optimizer puede reconocer:

```text
unsatisfiable predicate
```

---

# 78. Empty relation

Una query demostrablemente imposible puede transformarse a:

```text
EmptyRelation
```

como representación lógica.

---

# 79. EmptyRelation ≠ SQL string

Es un concepto lógico.

El Compiler decidirá cómo representarlo para cada plataforma.

---

# 80. Predicate pushdown

Ejemplo:

```text
SELECT *
FROM (
    SELECT *
    FROM users
) u
WHERE u.active = true
```

puede potencialmente transformarse para acercar el predicate a `users`.

---

# 81. Pushdown barriers

No será seguro siempre ante:

```text
DISTINCT
LIMIT
OFFSET
WINDOW
AGGREGATION
VOLATILE EXPRESSIONS
OUTER JOINS
SECURITY BARRIERS
```

---

# 82. Projection pruning

Si una subquery produce:

```text
id
name
email
created_at
```

pero el parent sólo consume:

```text
id
name
```

el Optimizer puede eliminar columnas innecesarias cuando no altere semántica.

---

# 83. Lineage assists pruning

El sistema de:

```text
lineage
dependency
```

será esencial para determinar qué outputs siguen siendo necesarios.

---

# 84. Join optimization

Será desarrollado en:

```text
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
```

---

# 85. Join reordering

Podrá considerarse:

```text
(A INNER JOIN B) INNER JOIN C
```

vs:

```text
A INNER JOIN (B INNER JOIN C)
```

cuando constraints y semantics lo permitan.

---

# 86. Outer joins

No deberán reordenarse utilizando reglas de INNER JOIN.

---

# 87. Join elimination

Ejemplo conceptual:

```text
users
JOIN countries
ON users.country_id = countries.id
```

podría permitir eliminar `countries` si:

```text
no country column is used
FK guarantees match
join multiplicity cannot change
join has no security/volatile semantics
```

---

# 88. FK alone may not be enough

Debe considerarse:

```text
nullable FK
constraint enforcement
join type
referential guarantees
security predicates
```

---

# 89. Outer join strengthening

Ejemplo:

```text
LEFT JOIN B
WHERE B.id IS NOT NULL
```

podría convertirse en:

```text
INNER JOIN B
```

si la equivalencia está demostrada.

---

# 90. Subquery optimization

Posibles transformaciones:

```text
EXISTS decorrelation
IN decorrelation
derived table flattening
scalar subquery simplification
redundant subquery removal
```

---

# 91. Correlation ≠ per-row execution

El Optimizer puede transformar una correlated subquery en una forma relacional distinta.

---

# 92. EXISTS to SEMI JOIN

Ejemplo lógico:

```text
WHERE EXISTS (
    SELECT ...
)
```

podría convertirse a:

```text
LogicalSemiJoin
```

---

# 93. NOT EXISTS to ANTI JOIN

Igualmente:

```text
NOT EXISTS
```

puede transformarse en:

```text
LogicalAntiJoin
```

cuando sea correcto.

---

# 94. NOT IN caution

Nunca deberá suponerse:

```text
NOT IN
=
NOT EXISTS
```

sin considerar NULL semantics.

---

# 95. CTE optimization

El Optimizer podrá decidir entre representaciones lógicas como:

```text
inline
shared reusable relation
materialization candidate
```

según:

```text
CTE materialization intent
volatility
reuse
security
capabilities
```

---

# 96. CTE requirement

Si el CTE declara:

```text
REQUIRE_MATERIALIZED
```

el Optimizer no podrá inlinearlo.

---

# 97. Preference vs requirement

```text
PREFER_INLINE
```

no equivale a:

```text
REQUIRE_INLINE
```

---

# 98. Recursive CTE

Las transformaciones sobre recursive CTEs deberán ser especialmente conservadoras.

---

# 99. Set operation optimization

Podrá considerar:

```text
UNION flattening
operand simplification
predicate distribution
projection pruning
duplicate elimination simplification
```

siempre preservando:

```text
bag semantics
duplicate semantics
NULL row equivalence
collation
type reconciliation
```

---

# 100. UNION ALL

No deberá transformarse accidentalmente en:

```text
UNION DISTINCT
```

---

# 101. EXCEPT

Debe preservarse:

```text
directionality
```

porque:

```text
A EXCEPT B
≠
B EXCEPT A
```

---

# 102. Aggregate optimization

Posibles optimizaciones:

```text
predicate pushdown before aggregation
redundant grouping elimination
aggregate simplification
partial aggregation preparation
```

---

# 103. Aggregate barriers

No deberá empujarse arbitrariamente un HAVING predicate a WHERE.

---

# 104. HAVING pushdown

Sólo podrá hacerse cuando el predicate dependa exclusivamente de:

```text
grouping keys
```

y la transformación preserve semántica.

---

# 105. Aggregate decomposition

Funciones como ciertas:

```text
SUM
COUNT
MIN
MAX
```

pueden tener propiedades útiles para planificación distribuida.

Pero:

```text
decomposability
```

deberá provenir de `AggregateFunctionDescriptor`.

---

# 106. Window optimization

Las window functions crean fuertes restricciones de:

```text
ordering
partitioning
frame semantics
peer semantics
```

---

# 107. Window expressions

No deberán evaluarse antes de que la semántica relacional permita hacerlo.

---

# 108. QUALIFY-like optimization

Si una futura extensión introduce filtros sobre resultados de window functions, el Optimizer deberá respetar su fase lógica.

---

# 109. DISTINCT optimization

Podrá eliminarse un DISTINCT sólo cuando se demuestre que el input ya es único respecto a la proyección relevante.

---

# 110. ConstraintGraph assists DISTINCT

Por ejemplo:

```text
SELECT DISTINCT users.id
```

si:

```text
users.id
```

es una key única no-null y ninguna operación anterior multiplica filas.

---

# 111. LIMIT optimization

LIMIT puede habilitar optimizaciones posteriores de Planner.

Pero el Optimizer no deberá asumir un orden inexistente.

---

# 112. LIMIT without ORDER BY

```text
LIMIT 10
```

no significa:

```text
first 10 by primary key
```

---

# 113. ORDER BY elimination

Un ORDER BY interno podría eliminarse cuando:

```text
its ordering is not semantically observable
```

pero sólo después de analizar el contexto.

---

# 114. Ordering property

El Optimizer deberá tratar:

```text
ordering
```

como propiedad semántica/física explícita, no como efecto accidental.

---

# 115. Locking semantics

Queries con:

```text
FOR UPDATE
FOR SHARE
```

o equivalentes tendrán optimization barriers adicionales.

---

# 116. Lock scope

Transformaciones que cambien qué filas son bloqueadas pueden alterar comportamiento observable.

Por ello:

```text
locking
```

forma parte del contrato semántico.

---

# 117. DML optimization

UPDATE y DELETE también podrán beneficiarse de:

```text
predicate simplification
join simplification
subquery optimization
```

pero deberán preservarse:

```text
mutation target
affected row set
returning semantics
locking
security
```

---

# 118. INSERT optimization

INSERT...SELECT podrá optimizar su source query.

No deberá alterar:

```text
insert mapping
conflict semantics
returning semantics
```

---

# 119. Query cost

VoltStack distinguirá:

```text
rule-based optimization
```

de:

```text
cost-based optimization
```

---

# 120. Rule-based optimization

Utiliza:

```text
semantic equivalence
heuristics
constraints
known structural properties
```

sin necesitar estimaciones detalladas de ejecución.

---

# 121. Cost-based optimization

Podrá utilizar:

```text
cardinality estimates
selectivity
statistics
indexes
physical capabilities
```

---

# 122. Cost-based boundary

La arquitectura deberá evitar duplicar responsabilidades con el Planner.

Una regla útil será:

```text
Optimizer:
chooses better logical alternatives

Planner:
chooses physical execution alternatives
```

---

# 123. Logical cost hints

El Optimizer puede recibir:

```text
logical preference information
```

sin seleccionar directamente:

```text
HashJoin
IndexScan
```

---

# 124. Query Planner boundary

La transición será:

```text
OptimizedQueryArtifact
        │
        ▼
Query Planner
        │
        ▼
LogicalQueryPlan
        │
        ▼
PhysicalQueryPlan
```

---

# 125. Alternative search

Algunas optimizaciones, especialmente joins, pueden producir múltiples alternativas.

---

# 126. Search explosion

Con `n` relaciones, el número de órdenes posibles puede crecer rápidamente.

Por tanto se necesitan:

```text
search budgets
pruning
memoization
heuristics
```

---

# 127. Memoization

El Optimizer podrá utilizar:

```text
OptimizationMemo
```

para evitar reevaluar expresiones equivalentes.

---

# 128. Memo ≠ global cache

`OptimizationMemo` será:

```text
operation-scoped
```

---

# 129. Future Cascades-style architecture

VoltStack podrá evolucionar hacia un modelo inspirado conceptualmente en:

```text
memo-based optimizer
```

sin hacer obligatorio ese diseño para V1.

---

# 130. V1 recommendation

Para V1:

```text
deterministic rule pipeline
+
bounded fixpoint groups
+
targeted alternative search for joins/subqueries
```

ofrece mejor equilibrio entre:

```text
complexity
correctness
performance
extensibility
```

---

# 131. Optimization profiles

Podrán existir:

```text
SAFE
BALANCED
AGGRESSIVE
DEBUG
```

---

# 132. SAFE

Prioriza:

```text
simple proven transformations
minimal search
strict barriers
```

---

# 133. BALANCED

Será recomendado como default.

---

# 134. AGGRESSIVE

Podrá habilitar:

```text
larger join search
more decorrelation attempts
additional algebraic rewrites
```

siempre sin relajar correctness.

---

# 135. Aggressive ≠ unsafe

Muy importante:

```text
AGGRESSIVE
≠
ALLOW SEMANTIC RISK
```

La diferencia será cantidad de trabajo/búsqueda, no nivel de corrección.

---

# 136. DEBUG

Puede priorizar:

```text
full optimization trace
rule-by-rule snapshots
extra invariant checks
```

---

# 137. OptimizationTrace

El Optimizer deberá poder explicar:

```text
what changed
which rule changed it
why the rule was valid
```

---

# 138. Trace entry

Modelo conceptual:

```php
final readonly class OptimizationTraceEntry
{
    public function __construct(
        public OptimizationRuleId $rule,
        public QueryNodeId $target,
        public StructuralFingerprint $before,
        public StructuralFingerprint $after,
        public OptimizationReason $reason,
    ) {}
}
```

---

# 139. Trace detail levels

Podrán existir:

```text
NONE
SUMMARY
NORMAL
VERBOSE
```

---

# 140. Production overhead

En producción podrá guardarse únicamente:

```text
summary statistics
```

mientras debugging puede conservar:

```text
full trace
```

---

# 141. Explain Optimizer

Developer tooling podrá mostrar:

```text
Optimization Summary

Rules evaluated:     38
Rules applied:       7
Fixpoint iterations: 3

Applied:
  RemoveTrueConjunct
  PredicatePushdown
  ProjectionPruning
  OuterJoinStrengthening
```

---

# 142. Explain transformation

Ejemplo:

```text
OuterJoinStrengthening

Before:
    users LEFT JOIN profiles
    WHERE profiles.id IS NOT NULL

After:
    users INNER JOIN profiles

Reason:
    post-join predicate rejects
    null-extended rows.
```

---

# 143. Rule rejection trace

En modo debug podrá mostrarse:

```text
JoinElimination rejected

Reason:
    relationship cardinality
    could not be proven.
```

Esto es muy importante para DX.

---

# 144. Optimization statistics

Podrán registrarse:

```text
rules considered
rules applied
nodes visited
nodes created
fixpoint iterations
join alternatives
decorrelation attempts
budget consumption
barriers encountered
```

---

# 145. Telemetry

Métricas futuras:

```text
database.optimizer.duration
database.optimizer.rule_evaluations
database.optimizer.rule_applications
database.optimizer.fixpoint_iterations
database.optimizer.join_alternatives
database.optimizer.budget_usage
database.optimizer.cache_hits
```

---

# 146. Sensitive data

Optimization trace nunca deberá incluir:

```text
password
token
secret parameter value
PII binding
```

---

# 147. Parameter representation

Debe mostrar:

```text
P1
P2
```

no sus bindings sensibles.

---

# 148. Optimization fingerprint

Podrá considerar:

```text
semantic query fingerprint
optimizer semantic version
enabled rule set
rule semantic versions
optimization profile
capability snapshot fingerprint
relevant statistics snapshot version
extension semantic versions
```

---

# 149. Runtime bindings excluded

```text
P1 = 10
```

no deberá normalmente cambiar el canonical optimization fingerprint.

---

# 150. Optimization cache

Una futura:

```text
OptimizedQueryCache
```

podrá reutilizar artifacts optimizados.

---

# 151. Cache safety

Sólo si coinciden:

```text
semantic fingerprint
optimizer fingerprint
capabilities
relevant schema/statistics versions
extensions
```

---

# 152. Statistics sensitivity

No toda optimización depende de estadísticas.

Por tanto será útil distinguir:

```text
statistics-independent optimization
```

de:

```text
statistics-dependent optimization
```

---

# 153. Two-stage optimization

Una futura arquitectura puede usar:

```text
Semantic/Heuristic Optimization
            │
            ▼
Cost-Aware Logical Optimization
```

antes del Planner.

---

# 154. Extension optimization rules

El documento 54 permite registrar:

```text
ExtensionOptimizationRule
```

---

# 155. Extension rule obligations

Toda extension rule deberá declarar:

```text
ExtensionId
RuleId
SemanticVersion
TargetNodeKinds
RequiredFacts
Barriers
BudgetCost
```

---

# 156. Extension rules cannot override safety

No podrán deshabilitar:

```text
security barriers
raw barriers
budget enforcement
semantic equivalence checks
```

arbitrariamente.

---

# 157. Extension rule ordering

Será integrado en el mismo:

```text
Optimization Rule DAG
```

que las reglas core.

---

# 158. No separate extension optimizer

No:

```text
Core Optimizer
+
Package Optimizer
+
ORM Optimizer
```

como motores independientes.

Debe existir:

```text
one coordinated optimization architecture
```

---

# 159. ORM integration

ORM produce queries que terminan en el mismo:

```text
SemanticQueryArtifact
```

por tanto usa exactamente el mismo Optimizer.

---

# 160. No ORM query optimization engine

Prohibido:

```text
ORM Optimizer
→ ORM SQL
```

---

# 161. ORM-specific metadata

El Optimizer puede recibir metadata de provenance útil.

Pero no deberá necesitar:

```text
EntityManager
UnitOfWork
IdentityMap
```

para optimizar queries.

---

# 162. Multitenancy

Tenant predicates deberán estar incorporados antes del Optimizer.

---

# 163. No hidden tenant injection

El Optimizer no será responsable de:

```text
adding tenant_id = ...
```

---

# 164. Tenant predicate preservation

Sí deberá garantizar que las transformaciones preserven:

```text
tenant isolation semantics
```

---

# 165. Authorization

Lo mismo aplica a predicates de authorization.

---

# 166. Security transformation policy

Puede existir:

```text
SecurityOptimizationPolicy
```

que determine qué transformaciones pueden cruzar boundaries de seguridad.

---

# 167. Policy-generated joins

Un security package puede introducir joins.

El Optimizer podrá modificarlos únicamente cuando:

```text
security semantic contract
```

lo permita.

---

# 168. Persistent runtime

El Optimizer debe ser compatible con workers persistentes.

---

# 169. Shared state

Podrán compartirse:

```text
frozen rule descriptors
frozen phase descriptors
immutable optimization profiles
compiled rule DAG
```

---

# 170. Operation-local state

Deberán ser locales:

```text
OptimizationWorkingArtifact
OptimizationMemo
OptimizationBudgetCounter
OptimizationTrace
OptimizationStatistics
candidate alternatives
fixpoint state
```

---

# 171. No current query singleton

Prohibido:

```text
Optimizer::$currentQuery
```

---

# 172. No global budget counter

Prohibido:

```text
static $remainingRules
```

---

# 173. Concurrent safety

Un mismo Optimizer shared deberá procesar:

```text
Q1
Q2
Q3
```

simultáneamente sin contaminación.

---

# 174. FrankenPHP

La arquitectura deberá asumir desde V1 que:

```text
worker lifetime
>
request lifetime
```

---

# 175. RoadRunner/OpenSwoole

Las mismas reglas permiten compatibilidad futura sin rediseñar el Optimizer.

---

# 176. Memory governance

El Optimizer deberá liberar al terminar:

```text
memo
alternatives
trace snapshots
temporary graphs
temporary fingerprints
```

---

# 177. Large queries

Queries extremadamente complejas deberán estar protegidas por:

```text
OptimizationBudget
QueryComplexityBudget
```

---

# 178. Complexity dimensions

Podrán incluir:

```text
AST nodes
semantic graph nodes
predicates
relations
joins
subqueries
CTEs
set operands
aggregate expressions
window expressions
extension nodes
```

---

# 179. Adversarial queries

Los budgets también son una defensa contra:

```text
optimization complexity attacks
```

---

# 180. Optimizer failure isolation

Una query que exceda el optimization budget no deberá degradar permanentemente al worker.

---

# 181. Diagnostics

Errores posibles:

```text
OptimizationInvariantViolation
OptimizationRuleConflict
OptimizationRuleDependencyCycle
OptimizationBudgetExceeded
OptimizationNonConvergence
InvalidOptimizationTransformation
UnsupportedOptimizationExtension
```

---

# 182. Budget exceeded behavior

Dependiendo de la fase:

```text
soft budget exhaustion
→ return best valid known form

hard safety exhaustion
→ fail query optimization
```

---

# 183. Non-convergence

Si dos reglas producen:

```text
A → B
B → A
```

indefinidamente:

```text
OptimizationNonConvergence
```

deberá detectarse.

---

# 184. Oscillation detection

Podrán almacenarse fingerprints recientes:

```text
F1
F2
F1
```

para detectar ciclos.

---

# 185. Rule semantic version

Cada regla deberá tener:

```text
OptimizationRuleSemanticVersion
```

---

# 186. Why

Cambiar la lógica de una regla puede cambiar:

```text
optimized fingerprint
cached optimized artifacts
planner inputs
```

---

# 187. Optimizer semantic version

También deberá existir:

```text
QueryOptimizerSemanticVersion
```

---

# 188. Testing strategy

El Optimizer deberá probarse mediante:

```text
unit tests
property tests
equivalence tests
metamorphic tests
integration tests
fuzz tests
budget tests
concurrency tests
persistent-worker tests
```

---

# 189. Rule unit tests

Cada regla deberá tener:

```text
positive case
negative case
barrier case
NULL case
type edge case
```

---

# 190. Equivalence testing

Para queries controladas:

```text
execute(original)
```

y:

```text
execute(optimized)
```

deberán producir resultados observacionalmente equivalentes.

---

# 191. Property-based testing

Podrán generarse:

```text
random predicates
random joins
random NULL distributions
random relation cardinalities
```

para verificar equivalencias.

---

# 192. Metamorphic testing

Si:

```text
Q1 ≡ Q2
```

el Optimizer deberá preservar:

```text
Semantics(Q1)
=
Semantics(Q2)
```

bajo datasets generados.

---

# 193. Multi-platform testing

Las reglas sensibles a SQL semantics deberán probarse contra:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

cuando la semántica pueda variar.

---

# 194. Capability-based tests

Los tests no deberán depender sólo de:

```text
if database == PostgreSQL
```

sino de capability profiles.

---

# 195. Debug invariant checker

En desarrollo podrá existir:

```text
OptimizationInvariantChecker
```

---

# 196. Invariant checker

Puede verificar:

```text
all symbols remain resolvable
all output columns remain valid
parameter identities preserved
security predicates preserved
required capabilities preserved
type contracts remain valid
```

---

# 197. Semantic revalidation

Después de ciertas transformaciones complejas, podrá ejecutarse una validación semántica incremental.

---

# 198. Incremental semantic maintenance

Idealmente las reglas deberán actualizar:

```text
semantic tables
constraints
lineage
dependencies
```

incrementalmente.

---

# 199. Full re-analysis fallback

Para V1, transformaciones complejas podrán optar por:

```text
re-run selected semantic passes
```

sobre el nuevo subtree.

---

# 200. No blind stale metadata

Nunca deberá ocurrir:

```text
AST changed
+
semantic tables left stale
```

---

# 201. OptimizationWorkingArtifact

Conceptualmente:

```php
final class OptimizationWorkingArtifact
{
    private QueryModel $query;
    private QuerySemanticTables $semantics;
    private ConstraintGraph $constraints;
    private SemanticQueryGraph $graph;

    // operation-local mutable orchestration state
}
```

---

# 202. Working artifact ≠ public artifact

Puede ser mutable internamente para eficiencia.

Pero:

```text
public input
public output
```

seguirán siendo inmutables.

---

# 203. Copy-on-write

Podrá utilizarse:

```text
persistent data structures
copy-on-write
structural sharing
```

para evitar copiar árboles completos.

---

# 204. Structural sharing

Si sólo cambia un predicate:

```text
new Query Tree
```

puede compartir nodos inmutables no modificados con el original.

---

# 205. Node identity

Debe distinguirse:

```text
Semantic Identity
```

de:

```text
Optimization Instance Identity
```

cuando una transformación reemplaza nodos.

---

# 206. Provenance chain

Un nuevo node podrá registrar:

```text
derivedFrom
```

para debugging.

---

# 207. Example

```text
Predicate P17
    │
    ├── optimized by:
    │      RemoveRedundantPredicate
    │
    ▼
Predicate P42
```

---

# 208. Optimization provenance

No forma necesariamente parte de query semantics.

Será metadata de tooling.

---

# 209. Query outputs

El Optimizer deberá preservar el:

```text
QueryOutputContract
```

---

# 210. Output contract includes

```text
column count
column order
column semantic identity
names
types
nullability
observable ordering requirements
```

según el tipo de query.

---

# 211. Internal projection changes

Podrá introducir/eliminar columnas internas para optimización.

Pero la salida pública deberá permanecer compatible.

---

# 212. Hidden optimizer columns

Si el Planner necesita columnas temporales, eso generalmente pertenece al plan.

El Optimizer no deberá contaminar innecesariamente el public query output.

---

# 213. Parameter identity

El Optimizer deberá preservar correctamente:

```text
ParameterId
```

o producir un mapping explícito cuando una transformación combine/elimine parámetros.

---

# 214. Parameter elimination

Ejemplo:

```text
WHERE FALSE AND x = P1
```

podría eliminar la rama con `P1`.

Entonces el optimized artifact puede no necesitar ese parámetro.

---

# 215. Binding projection

Deberá producirse información suficiente para que Execution conozca qué bindings siguen siendo requeridos.

---

# 216. No placeholder handling

El Optimizer nunca asignará:

```text
?
$1
:p1
```

Eso pertenece al Compiler.

---

# 217. Capability preservation

Una optimización puede:

```text
reduce
maintain
or transform
```

capability requirements.

---

# 218. Example

Si una extension semantic operation puede reescribirse de forma portable:

```text
Capability X
```

podría desaparecer del optimized artifact.

Pero sólo mediante una transformación formalmente soportada.

---

# 219. Portability optimization

Podrán existir reglas que prefieran:

```text
more portable equivalent representation
```

si el profile lo solicita.

---

# 220. OptimizationProfile dimensions

Podrá incluir:

```text
search intensity
portability preference
compile simplicity preference
logical cost preference
trace level
extension rule policy
```

---

# 221. Platform-neutral optimization

La mayor parte del Optimizer deberá operar sobre:

```text
semantic capabilities
```

no sobre nombres de motores.

---

# 222. Platform-specific optimization

Cuando sea inevitable:

```text
CapabilitySnapshot
```

deberá exponer la propiedad relevante.

---

# 223. No vendor branching

Evitar:

```php
if ($platform === 'postgresql') {
    ...
}
```

en core optimizer rules.

Preferir:

```php
if ($capabilities->supports(...)) {
    ...
}
```

---

# 224. Logical optimizer output

El `OptimizedQueryArtifact` seguirá representando:

```text
logical query meaning
```

No deberá contener:

```text
PDO statements
SQL strings
physical join algorithms
physical indexes selected
network connections
```

---

# 225. Optimizer does not execute

Regla absoluta:

```text
Optimizer
≠
Executor
```

---

# 226. Optimizer does not compile

Regla absoluta:

```text
Optimizer
≠
SQL Compiler
```

---

# 227. Optimizer does not own schema

Consume:

```text
SchemaView / semantic facts
```

pero no realiza introspección.

---

# 228. Optimizer does not own statistics

Consume snapshots.

No administra:

```text
ANALYZE
statistics collection
histogram persistence
```

---

# 229. Optimizer does not own security policy generation

Consume security semantics ya incorporada.

---

# 230. Optimizer does not own tenant resolution

Consume tenant-aware query semantics ya resueltas.

---

# 231. Arquitectura interna propuesta

```text
Query/
└── Optimizer/
    ├── Contract/
    │   ├── QueryOptimizer.php
    │   ├── OptimizationPhase.php
    │   ├── OptimizationRule.php
    │   └── EquivalenceVerifier.php
    │
    ├── Core/
    │   ├── DefaultQueryOptimizer.php
    │   ├── OptimizationCoordinator.php
    │   ├── QueryOptimizationContext.php
    │   ├── OptimizationWorkingArtifact.php
    │   ├── OptimizedQueryArtifact.php
    │   └── QueryOptimizerSemanticVersion.php
    │
    ├── Pipeline/
    │   ├── OptimizationPipeline.php
    │   ├── OptimizationPhaseRegistry.php
    │   ├── OptimizationPhaseGraph.php
    │   └── OptimizationScheduler.php
    │
    ├── Rule/
    │   ├── OptimizationRuleRegistry.php
    │   ├── OptimizationRuleId.php
    │   ├── OptimizationRuleSemanticVersion.php
    │   ├── OptimizationRuleDescriptor.php
    │   ├── OptimizationRuleResult.php
    │   └── OptimizationRuleDependency.php
    │
    ├── Rewrite/
    │   └── ...
    │
    ├── Expression/
    │   └── ...
    │
    ├── Predicate/
    │   └── ...
    │
    ├── Relation/
    │   └── ...
    │
    ├── Join/
    │   └── ...
    │
    ├── Subquery/
    │   └── ...
    │
    ├── Cte/
    │   └── ...
    │
    ├── SetOperation/
    │   └── ...
    │
    ├── Aggregate/
    │   └── ...
    │
    ├── Window/
    │   └── ...
    │
    ├── Projection/
    │   └── ...
    │
    ├── Constraint/
    │   └── OptimizationConstraintView.php
    │
    ├── Barrier/
    │   ├── OptimizationBarrier.php
    │   ├── OptimizationBarrierKind.php
    │   └── OptimizationBarrierResolver.php
    │
    ├── Equivalence/
    │   ├── SemanticEquivalence.php
    │   ├── EquivalenceEvidence.php
    │   └── EquivalenceVerifier.php
    │
    ├── Fixpoint/
    │   ├── OptimizationFixpointGroup.php
    │   ├── OptimizationFixpointRunner.php
    │   └── OptimizationConvergenceTracker.php
    │
    ├── Memo/
    │   └── OptimizationMemo.php
    │
    ├── Budget/
    │   ├── OptimizationBudget.php
    │   ├── OptimizationBudgetCounter.php
    │   └── OptimizationComplexity.php
    │
    ├── Trace/
    │   ├── OptimizationTrace.php
    │   ├── OptimizationTraceEntry.php
    │   └── OptimizationReason.php
    │
    ├── Statistics/
    │   └── OptimizationStatistics.php
    │
    ├── Fingerprint/
    │   └── OptimizationFingerprint.php
    │
    ├── Extension/
    │   └── ExtensionOptimizationRegistry.php
    │
    ├── Diagnostic/
    │   └── OptimizationDiagnostic.php
    │
    └── Exception/
        ├── QueryOptimizationException.php
        ├── OptimizationInvariantViolation.php
        ├── OptimizationRuleConflictException.php
        ├── OptimizationRuleDependencyCycleException.php
        ├── OptimizationNonConvergenceException.php
        └── OptimizationBudgetExceededException.php
```

---

# 232. Flujo completo

```text
SemanticQueryArtifact
        │
        ▼
OptimizationCoordinator
        │
        ├── validate optimizer context
        ├── resolve optimization profile
        ├── resolve rule set
        ├── initialize budget
        └── initialize working artifact
        │
        ▼
OptimizationPipeline
        │
        ├── Preparation
        ├── Expression Rules
        ├── Predicate Rules
        ├── Relational Rules
        ├── Subquery Rules
        ├── Join Rules
        ├── Set Rules
        ├── Aggregate Rules
        ├── Window Rules
        └── Final Simplification
        │
        ▼
Semantic Consistency Check
        │
        ▼
Optimization Fingerprint
        │
        ▼
OptimizedQueryArtifact
```

---

# 233. Invariantes arquitectónicos

## DB-OPT-001

Optimizer recibirá `SemanticQueryArtifact`.

## DB-OPT-002

Optimizer producirá `OptimizedQueryArtifact`.

## DB-OPT-003

Optimizer no mutará el artifact original.

## DB-OPT-004

Toda transformación preservará semántica observable.

## DB-OPT-005

Optimizer será distinto de Normalizer.

## DB-OPT-006

Optimizer será distinto de Semantic Analyzer.

## DB-OPT-007

Optimizer será distinto de Planner.

## DB-OPT-008

Optimizer será distinto de Compiler.

## DB-OPT-009

Optimizer será distinto de Executor.

## DB-OPT-010

Optimizer no generará SQL.

## DB-OPT-011

Optimizer no ejecutará queries.

## DB-OPT-012

Optimizer no abrirá conexiones.

## DB-OPT-013

Optimizer no realizará hidden schema introspection.

## DB-OPT-014

Optimizer consumirá semantic facts existentes.

## DB-OPT-015

Optimizer no inventará constraints.

## DB-OPT-016

UNKNOWN no será interpretado como TRUE.

## DB-OPT-017

SQL Three-Valued Logic será preservada.

## DB-OPT-018

Bag semantics será preservada.

## DB-OPT-019

Duplicate multiplicity será considerada.

## DB-OPT-020

NULL semantics será preservada.

## DB-OPT-021

Output contract será preservado.

## DB-OPT-022

Parameter identity será preservada o remapeada explícitamente.

## DB-OPT-023

Optimizer no asignará SQL placeholders.

## DB-OPT-024

Runtime bindings no serán inspeccionados por canonical optimization.

## DB-OPT-025

Optimization pipeline será explícito.

## DB-OPT-026

Optimization phases tendrán identidad estable.

## DB-OPT-027

Optimization rules tendrán identidad estable.

## DB-OPT-028

Rules tendrán semantic version.

## DB-OPT-029

Rule ordering será determinista.

## DB-OPT-030

Rule dependencies serán explícitas.

## DB-OPT-031

Dependency cycles no autorizados fallarán.

## DB-OPT-032

Fixpoint groups serán explícitos.

## DB-OPT-033

Fixpoint optimization será bounded.

## DB-OPT-034

Non-convergence será detectable.

## DB-OPT-035

Oscillation podrá detectarse mediante fingerprints.

## DB-OPT-036

Optimization tendrá budgets.

## DB-OPT-037

Budgets preferirán deterministic work units.

## DB-OPT-038

Budget exhaustion no corromperá artifacts.

## DB-OPT-039

Failure to improve no implicará query failure.

## DB-OPT-040

Original semantic form será fallback válida.

## DB-OPT-041

Optimization barriers serán explícitos.

## DB-OPT-042

Raw barriers serán respetados.

## DB-OPT-043

Security barriers serán respetados.

## DB-OPT-044

Volatility barriers serán respetados.

## DB-OPT-045

Ordering barriers serán respetados.

## DB-OPT-046

LIMIT/OFFSET semantics serán respetadas.

## DB-OPT-047

DISTINCT semantics serán respetadas.

## DB-OPT-048

Aggregation boundaries serán respetadas.

## DB-OPT-049

Window boundaries serán respetadas.

## DB-OPT-050

Locking semantics serán respetadas.

## DB-OPT-051

Correlation semantics serán respetadas.

## DB-OPT-052

Recursive semantics serán respetadas.

## DB-OPT-053

Materialization requirements serán respetados.

## DB-OPT-054

Volatile expressions no serán duplicadas arbitrariamente.

## DB-OPT-055

Volatile expressions no serán preevaluadas arbitrariamente.

## DB-OPT-056

Constant folding requerirá semantic proof.

## DB-OPT-057

Constant folding no inspeccionará runtime bindings.

## DB-OPT-058

Numeric rewrites respetarán overflow/precision semantics.

## DB-OPT-059

Predicate simplification respetará SQL 3VL.

## DB-OPT-060

Contradictions podrán producir logical EmptyRelation.

## DB-OPT-061

EmptyRelation será logical representation, no SQL.

## DB-OPT-062

Predicate pushdown respetará semantic barriers.

## DB-OPT-063

Projection pruning utilizará dependency/lineage facts.

## DB-OPT-064

Projection pruning preservará output contract.

## DB-OPT-065

Join reorder requerirá equivalence proof.

## DB-OPT-066

Outer joins no usarán ingenuamente inner-join algebra.

## DB-OPT-067

Join elimination requerirá cardinality proof.

## DB-OPT-068

FK presence sola no garantizará join elimination.

## DB-OPT-069

Outer join strengthening requerirá null-rejection proof.

## DB-OPT-070

Subquery decorrelation será optimizer responsibility.

## DB-OPT-071

Correlation no implicará per-row execution.

## DB-OPT-072

EXISTS podrá transformarse a logical SEMI JOIN.

## DB-OPT-073

NOT EXISTS podrá transformarse a logical ANTI JOIN.

## DB-OPT-074

NOT IN no será convertido ingenuamente a NOT EXISTS.

## DB-OPT-075

CTE optimization respetará materialization intent.

## DB-OPT-076

REQUIRE_MATERIALIZED será obligatorio.

## DB-OPT-077

PREFER_MATERIALIZED será sólo preferencia.

## DB-OPT-078

Recursive CTE transformations serán conservadoras.

## DB-OPT-079

Set optimization preservará multiplicity.

## DB-OPT-080

Set optimization preservará row-equivalence semantics.

## DB-OPT-081

UNION ALL no se convertirá accidentalmente a DISTINCT.

## DB-OPT-082

EXCEPT preservará directionality.

## DB-OPT-083

Aggregate rewrites respetarán grouping semantics.

## DB-OPT-084

HAVING no se moverá a WHERE sin proof.

## DB-OPT-085

Aggregate decomposition vendrá de descriptors.

## DB-OPT-086

Window optimization preservará partition/order/frame semantics.

## DB-OPT-087

DISTINCT sólo se eliminará con uniqueness proof.

## DB-OPT-088

LIMIT no implicará ordering.

## DB-OPT-089

ORDER BY sólo se eliminará si no es observable.

## DB-OPT-090

Locking behavior será observable cuando corresponda.

## DB-OPT-091

DML optimization preservará mutation target.

## DB-OPT-092

DML optimization preservará affected-row semantics.

## DB-OPT-093

INSERT optimization preservará mapping/conflict semantics.

## DB-OPT-094

Rule-based y cost-based optimization serán distinguibles.

## DB-OPT-095

Optimizer no elegirá physical join algorithms.

## DB-OPT-096

Optimizer no elegirá physical index scans como responsabilidad core.

## DB-OPT-097

Alternative search estará bounded.

## DB-OPT-098

OptimizationMemo será operation-scoped.

## DB-OPT-099

V1 favorecerá deterministic rule pipeline.

## DB-OPT-100

V1 permitirá bounded fixpoint groups.

## DB-OPT-101

AGGRESSIVE profile nunca significará unsafe.

## DB-OPT-102

Optimization trace será opcional.

## DB-OPT-103

Trace podrá explicar rules aplicadas.

## DB-OPT-104

Trace podrá explicar rules rechazadas.

## DB-OPT-105

Trace no expondrá sensitive bindings.

## DB-OPT-106

Optimization statistics serán operation-scoped.

## DB-OPT-107

Optimization fingerprint incluirá optimizer semantic version.

## DB-OPT-108

Optimization fingerprint incluirá rule semantic versions relevantes.

## DB-OPT-109

Runtime bindings serán excluidos del canonical fingerprint.

## DB-OPT-110

Statistics-dependent optimization identificará statistics version.

## DB-OPT-111

Extension rules usarán el mismo optimizer framework.

## DB-OPT-112

Extensions no crearán optimizer paralelo.

## DB-OPT-113

Extension rules no deshabilitarán core safety arbitrariamente.

## DB-OPT-114

ORM utilizará el mismo Optimizer.

## DB-OPT-115

Optimizer no dependerá de UnitOfWork.

## DB-OPT-116

Optimizer no dependerá de IdentityMap.

## DB-OPT-117

Tenant policies existirán antes de optimization.

## DB-OPT-118

Authorization policies existirán antes de optimization.

## DB-OPT-119

Security predicates obligatorios serán preservados.

## DB-OPT-120

Shared optimizer services serán immutable/stateless.

## DB-OPT-121

Optimization working state será operation-scoped.

## DB-OPT-122

No existirá current-query global mutable state.

## DB-OPT-123

No existirá global mutable budget.

## DB-OPT-124

Optimizer será safe para workers persistentes.

## DB-OPT-125

Optimizer será safe para concurrent requests.

## DB-OPT-126

Temporary optimization memory será liberable por operación.

## DB-OPT-127

Large-query optimization tendrá complexity budgets.

## DB-OPT-128

Optimizer deberá resistir complexity attacks.

## DB-OPT-129

Non-convergence no contaminará workers futuros.

## DB-OPT-130

Rule changes serán semantic-versioned.

## DB-OPT-131

Optimizer será probado mediante equivalence tests.

## DB-OPT-132

Optimizer será probado mediante property-based tests.

## DB-OPT-133

Optimizer será probado mediante persistent-runtime tests.

## DB-OPT-134

Semantic metadata no quedará stale después de transformations.

## DB-OPT-135

Transformations actualizarán o revalidarán semantic information.

## DB-OPT-136

Public optimized artifacts serán immutable.

## DB-OPT-137

Internal working structures podrán usar controlled mutability.

## DB-OPT-138

Structural sharing será permitido.

## DB-OPT-139

Optimization provenance será distinguible de query semantics.

## DB-OPT-140

VoltStack priorizará correctness sobre optimization opportunity.

---

# 234. Anti-patterns

## 234.1 Optimizer generating SQL

```text
Optimizer
→ "SELECT ..."
```

**Rechazado.**

---

## 234.2 Optimizer opening PDO

```php
$pdo->query(...)
```

**Rechazado.**

---

## 234.3 Reanalyzing query meaning from strings

```text
column name guessing
function name guessing
alias guessing
```

**Rechazado.**

Debe utilizar Semantic Analysis.

---

## 234.4 Inspecting parameter values

```php
if ($bindings['P1'] === null) {
    rewrite();
}
```

en canonical optimization.

**Rechazado.**

---

## 234.5 Boolean rewrites ignoring UNKNOWN

**Rechazado.**

---

## 234.6 Join reorder ignoring outer joins

**Rechazado.**

---

## 234.7 NOT IN → NOT EXISTS blindly

**Rechazado.**

---

## 234.8 Removing security predicates as duplicates

**Rechazado.**

---

## 234.9 Infinite rewrite loop

```text
A → B
B → A
```

sin convergence detection.

**Rechazado.**

---

## 234.10 Global optimization memo

**Rechazado.**

---

## 234.11 Aggressive = unsafe

**Rechazado.**

---

## 234.12 Optimizer choosing Hash Join

Como responsabilidad del logical optimizer:

**Rechazado.**

Pertenece al Planner.

---

# 235. Ejemplo integral

Consulta conceptual:

```sql
SELECT
    u.id,
    u.name
FROM users u
LEFT JOIN profiles p
    ON p.user_id = u.id
WHERE
    u.active = true
    AND true
    AND p.id IS NOT NULL;
```

---

# 236. Semantic input

Semantic Analysis determina:

```text
Relations:
    U = users
    P = profiles

Join:
    LEFT U → P

Predicates:
    P1 = u.active = true
    P2 = TRUE
    P3 = p.id IS NOT NULL

Facts:
    profiles.id is non-null
    P3 rejects null-extended rows
```

---

# 237. Optimization step 1

Regla:

```text
RemoveTrueConjunct
```

Transforma:

```text
u.active = true
AND TRUE
AND p.id IS NOT NULL
```

a:

```text
u.active = true
AND p.id IS NOT NULL
```

---

# 238. Optimization step 2

Regla:

```text
OuterJoinStrengthening
```

Observa:

```text
LEFT JOIN profiles
+
WHERE profiles.id IS NOT NULL
```

y demuestra que las filas null-extended son rechazadas.

Transforma:

```text
LEFT JOIN
```

en:

```text
INNER JOIN
```

---

# 239. Optimization step 3

Projection analysis detecta que:

```text
profiles
```

no aporta columnas al resultado.

Sin embargo, todavía no puede eliminarse automáticamente el join si no puede demostrar:

```text
exact cardinality preservation
```

---

# 240. Constraint analysis

Si además se demuestra:

```text
profiles.user_id
UNIQUE NOT NULL

and every relevant user
has exactly one matching profile
```

podría estudiarse join elimination.

Sin esa prueba:

```text
JOIN remains
```

---

# 241. Optimized representation

```text
SELECT
    u.id,
    u.name

FROM users u

INNER JOIN profiles p
    ON p.user_id = u.id

WHERE
    u.active = true
    AND p.id IS NOT NULL
```

Posteriores reglas podrían simplificar más si existen facts suficientes.

---

# 242. Optimization trace

```text
OptimizationTrace

#1 RemoveTrueConjunct
   target: Predicate P2
   result: removed

#2 OuterJoinStrengthening
   target: Join J1
   LEFT → INNER

   evidence:
      predicate P3 null-rejects
      right relation P

#3 JoinElimination
   target: Join J1
   result: rejected

   reason:
      cardinality preservation
      not proven
```

---

# 243. Resultado

El Optimizer entrega:

```text
OptimizedQueryArtifact
```

No:

```text
SQL
```

y no:

```text
physical execution plan
```

---

# 244. Fórmula del Optimizer

```text
Query Optimization
=
Semantic Query Artifact
+
Proven Equivalence Rules
+
Semantic Facts
+
Optimization Heuristics
+
Capabilities
+
Bounded Search
+
Deterministic Scheduling
→
Equivalent Optimized Query Artifact
```

---

# 245. Fórmula de seguridad

```text
Safe Optimization
=
Semantic Equivalence
+
3VL Awareness
+
Bag Awareness
+
NULL Awareness
+
Volatility Awareness
+
Security Barriers
+
Ordering Awareness
+
Locking Awareness
+
Bounded Resources
```

---

# 246. Fórmula de una regla

```text
Optimization Rule
=
Pattern
+
Preconditions
+
Required Semantic Facts
+
Barrier Checks
+
Transformation
+
Equivalence Evidence
+
Budget Cost
```

---

# 247. Fórmula de fixpoint

```text
Optimization Fixpoint
=
Repeated Safe Transformations
until
Structural/Semantic Stability
or
Budget Exhaustion
```

---

# 248. Fórmula de persistent-runtime safety

```text
Persistent-Safe Optimizer
=
Frozen Rule Graph
+
Stateless Shared Services
+
Immutable Input
+
Immutable Output
+
Operation-Scoped Working State
+
Bounded Memo
+
Bounded Trace
+
No Hidden Runtime Resources
```

---

# 249. Decisión arquitectónica final

VoltStack adoptará un Optimizer:

```text
semantic-aware
rule-driven
deterministic
bounded
extensible
capability-driven
explainable
persistent-runtime safe
```

con una arquitectura inicial basada en:

```text
Deterministic Rule Pipeline
        +
Bounded Fixpoint Groups
        +
Semantic Constraint Facts
        +
Explicit Optimization Barriers
        +
Targeted Alternative Search
```

y preparada para evolucionar posteriormente hacia:

```text
Memo-Based Logical Optimization
+
Cost-Aware Search
```

sin modificar las fronteras fundamentales del Query Engine.

La regla definitiva será:

> **VoltStack nunca optimizará una consulta basándose únicamente en que una transformación “parece más rápida”. Primero deberá demostrar que la transformación conserva el significado observable de la consulta; sólo después podrá decidir si la forma equivalente es preferible.**

---

# 250. Relación con los siguientes documentos

Este documento define la arquitectura general.

Los siguientes documentos especializarán sus componentes:

```text
56_DATABASE_QUERY_REWRITE_SYSTEM.md
    │
    └── catálogo y arquitectura de transformaciones equivalentes

57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
    │
    └── registry, lifecycle, scheduling y contratos de reglas

58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
    │
    └── simplificación, inferencia y movimiento de predicates

59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
    │
    └── reorder, elimination, strengthening y search

60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
    │
    └── detección y reutilización de estructuras equivalentes

61_DATABASE_QUERY_COST_HINT_SYSTEM.md
    │
    └── información de coste y preferencias lógicas

62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
    │
    └── frontera Optimizer → Planner

63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md

64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md

65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 251. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md        ← actual
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 252. Siguiente documento

```text
56_DATABASE_QUERY_REWRITE_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en:

```text
Rewrite Architecture

RewriteRule
RewritePattern
RewriteMatch
RewritePrecondition
RewriteResult

Local Rewrites
Relational Rewrites
Expression Rewrites
Predicate Rewrites

Canonical Rewrites
Simplification Rewrites
Structural Rewrites

Semantic Equivalence Evidence

NULL-Safe Rewrites
Three-Valued Logic Rewrites
Bag-Safe Rewrites

Subquery Rewrites
CTE Rewrites
Set Operation Rewrites
Aggregate Rewrites
Window Rewrites

Rewrite Barriers
Volatility
Security
Raw
Ordering
Locking

Rewrite Scheduling
Rewrite Dependencies
Rewrite Fixpoints
Rewrite Convergence

Rewrite Fingerprints
Rewrite Trace
Rewrite Budgets

Extension Rewrite Rules
Testing and Formal Verification
Persistent Runtime Safety
```

manteniendo la frontera:

```text
Normalization
    │
    └── canonical representation
             │
             ▼
Semantic Analysis
    │
    └── authoritative meaning
             │
             ▼
Rewrite System
    │
    └── proven equivalent forms
             │
             ▼
Optimizer
    │
    └── preferred equivalent form
             │
             ▼
Planner
```