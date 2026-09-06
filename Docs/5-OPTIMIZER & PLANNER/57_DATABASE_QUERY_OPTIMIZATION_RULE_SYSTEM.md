# 57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md

# VoltStack Quantum Database
## Query Optimization Rule System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 57 — Query Optimization Rule System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Rules  
**Versión:** 1.0

---

# 1. Propósito

`Query Optimization Rule System` define la infraestructura responsable de registrar, describir, clasificar, validar, organizar, seleccionar, programar y ejecutar las reglas utilizadas por el optimizador de consultas de VoltStack.

Su función principal es convertir un conjunto potencialmente grande de transformaciones y análisis en un sistema:

```text
determinista
extensible
versionado
explicable
seguro
acotado
capability-driven
persistent-runtime safe
```

La arquitectura general será:

```text
SemanticQueryArtifact
        │
        ▼
Query Optimizer
        │
        ▼
Optimization Rule System
        │
        ├── Rule Registry
        ├── Rule Catalog
        ├── Rule Profiles
        ├── Rule Dependencies
        ├── Rule Scheduler
        ├── Rule Preconditions
        ├── Rule Cost/Benefit
        ├── Rule Budgets
        ├── Rule Diagnostics
        └── Extension Rules
        │
        ▼
OptimizedQueryArtifact
```

---

# 2. Regla maestra

Una regla de optimización nunca deberá significar:

```text
"transform this query because it looks faster"
```

Deberá significar:

```text
Transformation is semantically valid
+
Transformation is applicable
+
Transformation is allowed
+
Transformation is expected to improve
a defined optimization objective
```

Formalmente:

```text
Apply(R, Q)
iff

Valid(R, Q)
∧ Applicable(R, Q)
∧ Allowed(R, Q)
∧ Beneficial(R, Q)
```

---

# 3. Optimization Rule ≠ Rewrite Rule

Esta separación es fundamental.

Una:

```text
RewriteRule
```

define una transformación semánticamente equivalente.

Una:

```text
OptimizationRule
```

decide cómo utilizar transformaciones, análisis o decisiones dentro del proceso de optimización.

Por tanto:

```text
RewriteRule
=
What equivalent transformation exists?

OptimizationRule
=
When should an optimization action be considered/applied?
```

---

# 4. Optimization Rule puede usar RewriteRule

Ejemplo:

```text
OptimizationRule:
    EliminateRedundantJoin
```

puede utilizar:

```text
RewriteRule:
    relation.join_elimination
```

después de determinar que:

```text
join is removable
+
rewrite is safe
+
optimization policy allows it
```

---

# 5. Optimization Rule no necesariamente transforma

Una regla también podrá:

```text
derive facts
annotate candidates
create alternatives
assign optimization hints
identify opportunities
request semantic refinement
```

Ejemplo:

```text
DetectRedundantProjectionRule
```

puede únicamente marcar:

```text
Projection#17
as removable candidate
```

para una fase posterior.

---

# 6. Optimization Rule ≠ Planner Rule

Una regla del Query Optimizer opera principalmente sobre:

```text
semantic/logical query representation
```

Una regla del Planner opera sobre:

```text
logical/physical plans
```

Por tanto:

```text
JoinReorderingRule
```

en el Optimizer puede producir:

```text
alternative logical join structures
```

mientras que:

```text
HashJoinSelection
```

pertenece al Planner.

---

# 7. Optimization Rule ≠ Compiler Rule

Nunca:

```text
OptimizationRule
→ SQL syntax
```

El Compiler conserva:

```text
semantic/logical representation
→ target SQL
```

---

# 8. Optimization Rule ≠ Platform conditional

Prohibido como arquitectura:

```php
if ($driver === 'mysql') {
    applyRule();
}
```

Debe utilizarse:

```text
PlatformCapabilities
```

---

# 9. Posición arquitectónica

```text
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
┌──────────────────────────────────┐
│         QUERY OPTIMIZER          │
│                                  │
│   Query Optimization Rule System │
│              │                   │
│              ├── Rewrite System  │
│              ├── Constraints     │
│              ├── Cost Hints      │
│              └── Capabilities    │
└──────────────────────────────────┘
      │
      ▼
OptimizedQueryArtifact
      │
      ▼
Query Planner
```

---

# 10. Objetivos

El sistema deberá proporcionar:

```text
stable rule identities
rule descriptors
rule registration
rule discovery
rule freezing
rule categorization
rule dependencies
rule conflicts
rule phases
rule profiles
rule applicability
rule preconditions
rule scheduling
rule budgets
rule statistics
rule diagnostics
rule explainability
rule versioning
rule extensions
```

---

# 11. Arquitectura general

```text
OptimizationRuleSystem
│
├── OptimizationRule
├── OptimizationRuleDescriptor
├── OptimizationRuleRegistry
├── OptimizationRuleCatalog
├── OptimizationRuleSet
├── OptimizationRuleProfile
│
├── RuleApplicabilityResolver
├── RulePreconditionEvaluator
├── RuleDependencyGraph
├── RuleConflictResolver
│
├── OptimizationRuleScheduler
├── OptimizationPhase
├── OptimizationPhasePlan
├── OptimizationFixpointGroup
│
├── RuleBenefitEstimator
├── RuleCostEstimator
├── RuleRiskClassifier
│
├── OptimizationRuleBudget
├── OptimizationRuleTrace
├── OptimizationRuleStatistics
│
└── OptimizationRuleExtensionRegistry
```

---

# 12. OptimizationRule

Contrato base:

```php
interface OptimizationRule
{
    public function id(): OptimizationRuleId;

    public function descriptor(): OptimizationRuleDescriptor;

    public function evaluate(
        OptimizationTarget $target,
        OptimizationContext $context,
    ): OptimizationRuleResult;
}
```

---

# 13. OptimizationRuleId

Cada regla tendrá identidad estable.

Ejemplos:

```text
predicate.constant_simplification
predicate.pushdown
predicate.null_rejection_analysis

projection.prune_unused

join.eliminate_redundant
join.strengthen_outer
join.reorder_inner

subquery.decorrelate_exists
subquery.decorrelate_not_exists

aggregate.pushdown_having
aggregate.reduce_grouping_keys

window.share_sort

set.flatten_union_all
```

---

# 14. IDs semánticos

El ID deberá describir:

```text
domain.action
```

o:

```text
domain.subdomain.action
```

No deberá contener nombres de clases PHP.

Incorrecto:

```text
OptimizeJoinRuleV2
```

Preferido:

```text
join.eliminate_redundant
```

---

# 15. Rule Semantic Version

Cada regla deberá declarar:

```text
OptimizationRuleSemanticVersion
```

Ejemplo:

```text
join.eliminate_redundant@2
```

---

# 16. ¿Por qué versionar reglas?

Porque modificar:

```text
applicability
preconditions
transformation strategy
required facts
capability requirements
```

puede alterar:

```text
optimized query fingerprint
compiled query cache
explain output
performance behavior
```

---

# 17. OptimizationRuleDescriptor

Modelo conceptual:

```php
final readonly class OptimizationRuleDescriptor
{
    public function __construct(
        public OptimizationRuleId $id,
        public OptimizationRuleSemanticVersion $version,
        public OptimizationRuleCategory $category,
        public OptimizationRulePhase $phase,
        public OptimizationRuleSafetyClass $safety,
        public OptimizationRuleRiskClass $risk,
        public OptimizationRuleExpansionClass $expansion,
        public array $targetKinds,
        public array $requiredFacts,
        public array $requiredCapabilities,
        public array $dependencies,
        public array $conflicts,
    ) {}
}
```

---

# 18. Descriptor immutable

Una vez registrado:

```text
OptimizationRuleDescriptor
```

será immutable.

---

# 19. Rule category

Categorías iniciales:

```text
EXPRESSION
PREDICATE
PROJECTION
RELATION
JOIN
SUBQUERY
CTE
SET_OPERATION
AGGREGATION
WINDOW
ORDERING
PAGINATION
DML
PORTABILITY
EXTENSION
```

---

# 20. Category ≠ phase

Una categoría describe:

```text
what domain the rule affects
```

Una phase describe:

```text
when it runs
```

---

# 21. Safety classification

Propuesta:

```text
LOCAL_PROVEN
SEMANTIC_PROVEN
CONSTRAINT_PROVEN
CAPABILITY_PROVEN
COST_DEPENDENT
EXTENSION_PROVEN
```

---

# 22. LOCAL_PROVEN

La regla puede validarse localmente.

Ejemplo:

```text
P AND TRUE
→ P
```

---

# 23. SEMANTIC_PROVEN

Requiere información del:

```text
SemanticQueryArtifact
```

---

# 24. CONSTRAINT_PROVEN

Requiere:

```text
ConstraintGraph
```

Ejemplo:

```text
join elimination
```

---

# 25. CAPABILITY_PROVEN

Su aplicación depende de capacidades disponibles.

---

# 26. COST_DEPENDENT

La transformación es correcta pero sólo debería aplicarse cuando existe evidencia de beneficio.

---

# 27. Rule risk

Riesgo no significa:

```text
risk of incorrect result
```

Una regla incorrecta nunca deberá ejecutarse.

Risk representa:

```text
optimization search complexity
plan instability
query growth
cost uncertainty
```

---

# 28. Risk classes

```text
LOW
MODERATE
HIGH
EXPERIMENTAL
```

---

# 29. LOW

Ejemplo:

```text
remove redundant predicate
```

---

# 30. MODERATE

Ejemplo:

```text
predicate pushdown across derived relation
```

---

# 31. HIGH

Ejemplo:

```text
large join reorder search
```

---

# 32. EXPERIMENTAL

Sólo habilitable explícitamente.

Nunca será parte del profile predeterminado.

---

# 33. Expansion classification

```text
SHRINKING
NEUTRAL
BOUNDED_EXPANSION
SEARCH_EXPANSION
```

---

# 34. SHRINKING

Reduce estructura.

Ejemplo:

```text
projection pruning
```

---

# 35. NEUTRAL

Mantiene tamaño aproximadamente estable.

---

# 36. BOUNDED_EXPANSION

Puede aumentar la representación de manera controlada.

---

# 37. SEARCH_EXPANSION

Puede generar múltiples alternativas.

Ejemplo:

```text
join ordering alternatives
```

---

# 38. Optimization Rule Registry

El:

```text
OptimizationRuleRegistry
```

será el registro canónico de reglas disponibles.

---

# 39. Registro

Ejemplo conceptual:

```php
$registry->register(
    new PredicatePushdownRule()
);
```

---

# 40. Registry responsibilities

```text
register rules
validate IDs
validate versions
detect duplicates
validate dependencies
validate conflicts
build rule catalog
freeze registry
```

---

# 41. Registry lifecycle

```text
Framework Bootstrap
       │
       ▼
Mutable Registry
       │
       ├── Core Rules
       ├── Database Package Rules
       └── Extension Rules
       │
       ▼
Validation
       │
       ▼
Dependency Graph
       │
       ▼
Freeze
       │
       ▼
Read-only Runtime Registry
```

---

# 42. Frozen registry

Después de bootstrap:

```text
register()
```

deberá fallar.

---

# 43. Duplicate RuleId

Dos reglas no podrán compartir:

```text
OptimizationRuleId
```

dentro del mismo semantic rule universe.

---

# 44. Rule replacement

Una extensión no podrá reemplazar silenciosamente:

```text
core rule
```

---

# 45. Explicit override

Si en el futuro se permite:

```text
rule override
```

deberá requerir:

```text
explicit configuration
compatibility declaration
semantic version declaration
diagnostic
```

---

# 46. OptimizationRuleCatalog

El Catalog será una vista immutable del Registry.

Podrá consultar:

```text
all()
byCategory()
byPhase()
byProfile()
byTargetKind()
byCapability()
```

---

# 47. RuleSet

Un:

```text
OptimizationRuleSet
```

representará un conjunto concreto de reglas seleccionadas para una operación.

---

# 48. Registry ≠ RuleSet

```text
Registry
=
all available rules

RuleSet
=
rules selected for current optimization
```

---

# 49. RuleSet immutable

Una vez iniciada la optimización:

```text
RuleSet
```

no cambiará.

---

# 50. Rule selection

Dependerá de:

```text
OptimizationProfile
PlatformCapabilities
QueryFeatures
Configuration
Budgets
Extensions
```

---

# 51. Optimization profiles

VoltStack definirá inicialmente:

```text
MINIMAL
SAFE
STANDARD
AGGRESSIVE
DEBUG
```

---

# 52. MINIMAL

Incluye:

```text
low-cost simplification
canonical reductions
obvious redundant structure elimination
```

---

# 53. SAFE

Añade:

```text
semantic-proven rules
constraint-proven local optimizations
bounded relational simplification
```

---

# 54. STANDARD

Será el perfil predeterminado.

Incluye:

```text
SAFE
+
predicate optimization
+
projection pruning
+
join simplification
+
subquery optimization
+
bounded join reorder
+
aggregate/window optimizations
```

---

# 55. AGGRESSIVE

Permite:

```text
larger search spaces
higher rewrite budgets
more alternatives
higher-cost analysis
```

---

# 56. DEBUG

No significa mayor optimización.

Significa:

```text
maximum diagnostics
rule traces
proof traces
before/after snapshots
invariant validation
```

---

# 57. Profile correctness

Todos los profiles deberán producir resultados semánticamente equivalentes.

La diferencia será:

```text
optimization effort
```

---

# 58. Rule profile membership

Una regla podrá declarar:

```text
minimumProfile
```

Ejemplo:

```text
predicate.constant_simplification
→ MINIMAL

join.eliminate_redundant
→ SAFE

join.reorder_inner
→ STANDARD
```

---

# 59. Profile exclusions

También podrán configurarse:

```text
disabledRules
disabledCategories
```

---

# 60. Per-query rule policy

`QueryMetadata` podrá expresar:

```text
OptimizationPolicy
```

Ejemplo:

```text
DEFAULT
MINIMAL
DISABLE_RULE
ENABLE_EXPERIMENTAL_RULE
```

---

# 61. User hints ≠ command

Por defecto:

```text
optimization hint
```

será una preferencia.

No deberá obligar al sistema a ejecutar una transformación inválida.

---

# 62. Required optimization directive

Si alguna API futura permite:

```text
REQUIRE optimization X
```

y no puede cumplirse:

```text
explicit failure
```

será preferible a degradación silenciosa.

---

# 63. Rule applicability

Antes de evaluar una regla deberá comprobarse:

```text
RuleApplicability
```

---

# 64. Applicability dimensions

```text
target node kind
query form
query features
semantic phase
required facts
required capabilities
barriers
profile
budget
extension availability
```

---

# 65. RuleApplicabilityResolver

Contrato:

```php
interface RuleApplicabilityResolver
{
    public function resolve(
        OptimizationRuleDescriptor $rule,
        OptimizationTarget $target,
        OptimizationContext $context,
    ): RuleApplicabilityResult;
}
```

---

# 66. Applicability result

```text
APPLICABLE
NOT_APPLICABLE
UNKNOWN
BLOCKED
```

---

# 67. UNKNOWN

Si la regla requiere evidencia no disponible:

```text
UNKNOWN
```

no deberá convertirse automáticamente en:

```text
APPLICABLE
```

---

# 68. BLOCKED

Significa que existe una barrera explícita.

Ejemplos:

```text
security barrier
raw barrier
locking barrier
materialization requirement
extension barrier
```

---

# 69. Preconditions

Después de applicability podrán evaluarse:

```text
rule-specific preconditions
```

---

# 70. Preconditions examples

Para join elimination:

```text
joined output unused
join cardinality preserving
no filtering effect
no locking effect
no security effect
```

---

# 71. Preconditions must be provable

Nunca:

```text
probably true
```

---

# 72. Rule effects

Toda regla deberá declarar qué puede modificar.

---

# 73. OptimizationRuleEffect

Ejemplos:

```text
EXPRESSION_STRUCTURE
PREDICATE_STRUCTURE
RELATION_STRUCTURE
JOIN_STRUCTURE
PROJECTION
ORDERING
CARDINALITY
PARAMETER_REQUIREMENTS
CAPABILITY_REQUIREMENTS
SEMANTIC_METADATA
```

---

# 74. Effect declaration

Permite al Scheduler saber qué reglas podrían necesitar re-evaluarse después.

---

# 75. Example

Si una regla modifica:

```text
PREDICATE_STRUCTURE
```

podrían reactivarse:

```text
predicate simplification
predicate pushdown
outer join strengthening
```

---

# 76. Rule dependency

Una regla puede requerir que otra haya sido ejecutada antes.

Ejemplo:

```text
predicate.simplify
        ↓
predicate.pushdown
```

---

# 77. Dependency types

```text
RUN_BEFORE
RUN_AFTER
REQUIRES_RULE
REQUIRES_PHASE
REQUIRES_FACT
```

---

# 78. RuleDependencyGraph

```text
OptimizationRule
      │
      ▼
RuleDependencyGraph
      │
      ▼
Validated execution ordering
```

---

# 79. Dependency graph validation

Durante bootstrap deberán detectarse:

```text
missing dependencies
cycles
invalid phase dependencies
conflicting requirements
```

---

# 80. Dependency cycles

Un ciclo no declarado como fixpoint:

```text
A → B → C → A
```

será error de configuración.

---

# 81. Rule conflicts

Dos reglas pueden ser individualmente válidas pero estratégicamente incompatibles.

---

# 82. Conflict example

Una regla puede:

```text
inline CTE
```

mientras otra pretende:

```text
preserve CTE for reuse/materialization
```

---

# 83. OptimizationRuleConflict

Modelo:

```php
final readonly class OptimizationRuleConflict
{
    public function __construct(
        public OptimizationRuleId $left,
        public OptimizationRuleId $right,
        public RuleConflictKind $kind,
    ) {}
}
```

---

# 84. Conflict kinds

```text
MUTUALLY_EXCLUSIVE
ORDER_SENSITIVE
SEARCH_ALTERNATIVE
PROFILE_CONFLICT
CAPABILITY_CONFLICT
```

---

# 85. MUTUALLY_EXCLUSIVE

Sólo una podrá aplicarse sobre el mismo target.

---

# 86. ORDER_SENSITIVE

Ambas pueden aplicarse, pero el orden importa.

---

# 87. SEARCH_ALTERNATIVE

Ambas producen alternativas válidas.

El Optimizer puede evaluarlas como candidatos.

---

# 88. Rule Scheduler

El:

```text
OptimizationRuleScheduler
```

determinará:

```text
which rule
on which target
in which phase
in which order
```

---

# 89. Scheduler inputs

```text
RuleSet
RuleDependencyGraph
OptimizationPhasePlan
QueryFeatures
ChangedNodes
Budgets
RuleStatistics
```

---

# 90. Scheduler determinism

Mismo:

```text
query
capabilities
configuration
rules
versions
```

deberá producir el mismo scheduling lógico.

---

# 91. No accidental class iteration order

No depender de:

```php
foreach ($container->allRules() as $rule)
```

si ese orden no está formalizado.

---

# 92. Stable tie-break

Cuando dos reglas tienen la misma prioridad:

```text
stable RuleId ordering
```

podrá utilizarse como último desempate.

---

# 93. Optimization phases

Propuesta:

```text
PHASE_01_CANONICAL
PHASE_02_EXPRESSION
PHASE_03_PREDICATE
PHASE_04_PROJECTION
PHASE_05_RELATIONAL
PHASE_06_SUBQUERY
PHASE_07_JOIN
PHASE_08_SET_OPERATION
PHASE_09_AGGREGATION
PHASE_10_WINDOW
PHASE_11_FINAL_SIMPLIFICATION
```

---

# 94. Phase plan

```php
final readonly class OptimizationPhasePlan
{
    public function __construct(
        public array $phases,
    ) {}
}
```

---

# 95. Phase ≠ single pass

Una phase puede contener:

```text
single-pass rules
fixpoint groups
worklists
alternative generation
```

---

# 96. Example phase

```text
Predicate Optimization Phase
│
├── simplify constants
├── simplify null predicates
├── remove redundant predicates
├── infer null rejection
├── push predicates
└── simplify again
```

---

# 97. Fixpoint groups

Algunas reglas se ejecutarán hasta convergencia.

---

# 98. OptimizationFixpointGroup

```php
final readonly class OptimizationFixpointGroup
{
    public function __construct(
        public OptimizationFixpointGroupId $id,
        public array $rules,
        public int $maxIterations,
    ) {}
}
```

---

# 99. Fixpoint example

```text
predicate simplification
        ↓
predicate pushdown
        ↓
join strengthening
        ↓
predicate simplification
```

puede descubrir nuevas oportunidades.

---

# 100. Convergence

Se utilizarán:

```text
structural fingerprints
semantic fingerprints
change sets
```

---

# 101. Oscillation protection

El sistema deberá detectar:

```text
A
→ B
→ A
→ B
```

---

# 102. Rule application count

Cada regla tendrá límites como:

```text
maxApplicationsPerQuery
maxApplicationsPerTarget
```

cuando sea necesario.

---

# 103. Optimization budget

El Rule System compartirá el budget global del Optimizer.

---

# 104. Budget dimensions

```text
rule evaluations
rule applications
node visits
generated nodes
alternatives
fixpoint iterations
semantic refinements
search states
```

---

# 105. Per-rule budget

Además podrá existir:

```text
OptimizationRuleBudget
```

---

# 106. Example

```text
join.reorder_inner:

maxRelations = 8
maxAlternatives = 256
```

---

# 107. Budget reservation

Antes de una operación costosa:

```text
reserve budget
```

---

# 108. Budget exhaustion

Nunca deberá producir:

```text
partially corrupted optimizer state
```

---

# 109. Valid fallback

Al agotarse el budget:

```text
best valid artifact found so far
```

podrá continuar hacia Planner.

---

# 110. Cost of rule

Una regla puede tener costo de optimización.

No confundir con costo de ejecutar la query.

---

# 111. RuleCost

Representa:

```text
CPU optimizer work
memory
search expansion
semantic analysis work
```

---

# 112. Query execution cost

Representa:

```text
estimated database execution work
```

Son conceptos distintos.

---

# 113. RuleCostEstimator

Contrato:

```php
interface RuleCostEstimator
{
    public function estimate(
        OptimizationRuleDescriptor $rule,
        OptimizationTarget $target,
        OptimizationContext $context,
    ): OptimizationRuleCost;
}
```

---

# 114. Benefit estimation

Una regla podrá declarar beneficio esperado.

---

# 115. Benefit classes

```text
CERTAIN
LIKELY
UNKNOWN
SEARCH_DEPENDENT
```

---

# 116. CERTAIN

Ejemplo:

```text
remove unused projection
```

normalmente reduce trabajo sin ampliar search.

---

# 117. LIKELY

Ejemplo:

```text
predicate pushdown
```

puede reducir cardinalidad tempranamente.

---

# 118. UNKNOWN

Ejemplo:

```text
some join reorder
```

sin statistics suficientes.

---

# 119. SEARCH_DEPENDENT

El beneficio sólo puede conocerse comparando alternativas.

---

# 120. Benefit ≠ correctness

Aunque una regla tenga:

```text
CERTAIN benefit
```

deberá demostrar primero corrección.

---

# 121. Rule scoring

Una estrategia futura podrá usar:

```text
RuleScore
=
ExpectedBenefit
-
OptimizationCost
-
SearchRisk
```

---

# 122. V1 recommendation

V1 no necesita un modelo matemático sofisticado.

Puede usar:

```text
rule class
+
query shape
+
budget
+
known semantic facts
```

---

# 123. Statistics independence

El Rule System deberá funcionar incluso sin estadísticas físicas de base de datos.

---

# 124. Statistics-enhanced optimization

Cuando existan:

```text
row counts
histograms
selectivity
index statistics
```

podrán mejorar decisiones.

Pero:

```text
statistics unavailable
```

no deberá impedir optimizaciones semánticas seguras.

---

# 125. Semantic facts vs statistics

```text
Semantic Fact:
    column is UNIQUE

Statistic:
    approximately 2.3M distinct values
```

No son equivalentes.

---

# 126. Rule targeting

Una regla deberá declarar:

```text
targetKinds
```

Ejemplo:

```text
PredicateNode
JoinNode
ProjectionNode
AggregateScope
WindowExpression
```

---

# 127. Target indexing

El Optimizer podrá mantener índices operation-scoped:

```text
nodesByKind
relationsByKind
predicatesByScope
joinsByType
```

para evitar recorrer todo el graph constantemente.

---

# 128. Worklist scheduling

Modelo futuro recomendado:

```text
Changed Node
     │
     ▼
Affected Rule Index
     │
     ▼
Optimization Worklist
```

---

# 129. Rule trigger index

Ejemplo:

```text
JoinNode changed
```

activa potencialmente:

```text
join simplification
join elimination
predicate pushdown
projection pruning
```

---

# 130. ChangeSet

Cada regla aplicada deberá producir:

```text
OptimizationChangeSet
```

---

# 131. ChangeSet model

```php
final readonly class OptimizationChangeSet
{
    public function __construct(
        public array $createdNodes,
        public array $removedNodes,
        public array $changedNodes,
        public array $invalidatedFacts,
        public array $newFacts,
    ) {}
}
```

---

# 132. ChangeSet purpose

Permite:

```text
incremental scheduling
semantic maintenance
debugging
telemetry
```

---

# 133. Alternative-producing rules

Algunas reglas no reemplazarán inmediatamente una representación.

Podrán producir:

```text
OptimizationAlternativeSet
```

---

# 134. Example

Para:

```text
A JOIN B JOIN C
```

podrían generarse:

```text
(A JOIN B) JOIN C

(A JOIN C) JOIN B

(B JOIN C) JOIN A
```

si son semánticamente equivalentes.

---

# 135. Alternative ≠ physical plan

Siguen siendo:

```text
logical query alternatives
```

---

# 136. Alternative identity

Cada alternativa tendrá:

```text
OptimizationAlternativeId
```

---

# 137. Alternative deduplication

Dos reglas pueden producir la misma estructura.

Se utilizará:

```text
semantic/structural fingerprint
```

para deduplicarlas.

---

# 138. Search explosion

Debe existir control explícito sobre:

```text
number of alternatives
```

---

# 139. Search pruning

Podrán descartarse alternativas:

```text
dominated
duplicate
over-budget
blocked
known-worse
```

---

# 140. Rule result

Modelo conceptual:

```php
final readonly class OptimizationRuleResult
{
    public function __construct(
        public OptimizationRuleStatus $status,
        public ?OptimizationChangeSet $changeSet = null,
        public ?OptimizationAlternativeSet $alternatives = null,
        public array $diagnostics = [],
    ) {}
}
```

---

# 141. Rule status

```text
NOT_APPLICABLE
BLOCKED
REJECTED
NO_CHANGE
APPLIED
ALTERNATIVES_PRODUCED
BUDGET_SKIPPED
```

---

# 142. REJECTED

La regla era potencialmente aplicable pero una precondition no fue demostrada.

---

# 143. BLOCKED

Una barrier impidió la aplicación.

---

# 144. BUDGET_SKIPPED

La regla podría ser válida, pero no existe budget suficiente para evaluarla.

---

# 145. Diagnostics

El sistema deberá distinguir:

```text
error
warning
info
trace
```

---

# 146. Optimization diagnostics examples

```text
RuleBlockedBySecurityBarrier
RuleBlockedByRawExpression
RuleMissingRequiredCapability
RulePreconditionUnknown
RuleBudgetExceeded
RuleDependencyCycle
RuleConflictDetected
RuleProducedDuplicateAlternative
RuleFixpointLimitReached
```

---

# 147. Explainability

VoltStack deberá poder responder:

```text
Why was this rule applied?

Why was this rule not applied?
```

---

# 148. Rule explanation

Ejemplo:

```text
Rule:
    join.eliminate_redundant

Status:
    REJECTED

Reason:
    joined relation uniqueness not proven
```

---

# 149. Applied explanation

```text
Rule:
    predicate.pushdown

Status:
    APPLIED

Reason:
    predicate references only child relation
    no aggregation/window/limit barrier
    predicate is non-volatile
```

---

# 150. Query EXPLAIN integration

En el futuro:

```text
VoltStack Query Explain
```

podrá mostrar:

```text
Semantic Query
Optimization Rules Applied
Logical Plan
Physical Plan
Compiled SQL
```

---

# 151. Rule trace

Ejemplo:

```text
Optimization Trace

[01] predicate.constant_simplification
     APPLIED

[02] projection.prune_unused
     APPLIED

[03] join.eliminate_redundant
     REJECTED
     reason: cardinality preservation unknown

[04] subquery.decorrelate_exists
     APPLIED
```

---

# 152. Telemetry

Podrán registrarse:

```text
rule evaluations
rule applications
rule rejections
rule blocking
rule runtime
generated alternatives
budget usage
```

---

# 153. Telemetry ≠ semantics

Telemetry nunca decidirá por sí sola la corrección de una regla.

---

# 154. Rule performance telemetry

Podrá ayudar a descubrir:

```text
expensive optimizer rules
rules with low hit rate
rules causing excessive alternatives
```

---

# 155. Adaptive optimization

No será parte de V1.

Una futura versión podría usar telemetry histórica para ajustar:

```text
rule scheduling
budgets
```

sin modificar semantic correctness.

---

# 156. Core predicate rules

Ejemplos:

```text
predicate.remove_true_conjunct
predicate.remove_false_disjunct
predicate.double_negation
predicate.constant_fold
predicate.redundancy_elimination
predicate.contradiction_detection
predicate.pushdown
predicate.null_rejection
```

---

# 157. Core projection rules

```text
projection.prune_unused
projection.merge
projection.remove_identity
projection.reduce_internal_columns
```

---

# 158. Core join rules

```text
join.strengthen_outer
join.eliminate_redundant
join.reorder_inner
join.reassociate_inner
join.cross_to_inner
```

---

# 159. Core subquery rules

```text
subquery.decorrelate_exists
subquery.decorrelate_not_exists
subquery.flatten_derived
subquery.push_predicate
subquery.prune_projection
```

---

# 160. Core CTE rules

```text
cte.remove_unused
cte.inline
cte.projection_prune
cte.predicate_pushdown
```

respetando:

```text
materialization intent
recursion
volatility
security barriers
```

---

# 161. Core set rules

```text
set.flatten_union_all
set.remove_empty_operand
set.projection_pushdown
set.predicate_pushdown
set.remove_redundant_distinct
```

---

# 162. Core aggregate rules

```text
aggregate.push_having_predicate
aggregate.reduce_grouping_keys
aggregate.deduplicate_expression
aggregate.preaggregate
aggregate.partial
```

---

# 163. Core window rules

```text
window.deduplicate_expression
window.merge_compatible_groups
window.share_ordering
window.prune_unused
```

---

# 164. Ordering rules

```text
ordering.remove_unobservable
ordering.merge_equivalent
ordering.remove_redundant_terms
```

---

# 165. Pagination rules

Extremadamente conservadoras.

```text
LIMIT
OFFSET
cursor pagination
```

son semantic boundaries importantes.

---

# 166. DML rules

```text
update.predicate_simplification
delete.predicate_simplification
insert.source_optimization
```

---

# 167. DML mutation invariant

Nunca cambiar:

```text
which rows are mutated
```

sin proof de equivalencia.

---

# 168. Extension rules

Los paquetes podrán registrar reglas adicionales.

---

# 169. Extension contract

```php
interface OptimizationRuleExtension
{
    public function rules(): iterable;
}
```

---

# 170. Extension identity

Toda extension rule deberá declarar:

```text
ExtensionId
ExtensionSemanticVersion
OptimizationRuleId
OptimizationRuleSemanticVersion
```

---

# 171. Namespacing de IDs

Recomendado:

```text
extension.vendor.package.rule_name
```

Ejemplo:

```text
extension.acme.analytics.timeseries_pushdown
```

---

# 172. Extension cannot bypass safety

Una extensión no podrá evitar:

```text
semantic validation
barrier validation
budget
rule lifecycle
```

---

# 173. Extension capability

Las extensiones podrán introducir:

```text
custom semantic capabilities
```

que sus reglas requieran explícitamente.

---

# 174. Rule registration order

No definirá ejecución.

---

# 175. Rule freezing

Después de:

```text
DatabaseBootstrap
```

el conjunto disponible será immutable.

---

# 176. Runtime rule enabling

La selección podrá cambiar por query/profile, pero el Registry no.

Es decir:

```text
Registry = frozen

RuleSet = operation-specific
```

---

# 177. Persistent runtime

Especialmente importante para FrankenPHP.

---

# 178. Shared state

Puede compartirse:

```text
OptimizationRuleDescriptor
OptimizationRuleRegistry
OptimizationRuleCatalog
RuleDependencyGraph
compiled rule metadata
```

---

# 179. Operation-local state

Debe mantenerse por query:

```text
RuleSet
Scheduler state
Worklist
Rule counters
Budgets
Rule trace
Rule statistics
AlternativeSet
ChangeSet
```

---

# 180. No global current query

Prohibido:

```php
OptimizationRuleRegistry::$currentQuery
```

---

# 181. No global disabled rules mutation

Prohibido:

```php
$registry->disable('join.reorder');
```

durante una request.

Debe existir:

```text
OptimizationContext
```

o:

```text
RuleSet
```

operation-scoped.

---

# 182. Concurrency

El mismo:

```text
frozen registry
```

deberá poder atender múltiples optimizaciones concurrentes.

---

# 183. State reset

Al terminar:

```text
worklists
alternative sets
rule counters
trace buffers
temporary mappings
```

deberán ser liberables.

---

# 184. Security rules

Algunas reglas deberán estar marcadas:

```text
SECURITY_SENSITIVE
```

---

# 185. Security-sensitive examples

```text
predicate pushdown
join elimination
subquery flattening
CTE inlining
```

---

# 186. Policy-generated structures

Estructuras provenientes de:

```text
tenant policy
authorization policy
security policy
```

deberán contener provenance explícita.

---

# 187. Security barrier

Una regla no declarada como:

```text
security-preserving
```

no podrá atravesar una security barrier.

---

# 188. Rule trust

Las reglas core tendrán:

```text
CORE_TRUSTED
```

Las extension rules podrán clasificarse:

```text
EXTENSION_TRUSTED
EXTENSION_RESTRICTED
```

---

# 189. Trust ≠ correctness bypass

Incluso:

```text
CORE_TRUSTED
```

deberá respetar contracts y invariants.

---

# 190. Rule configuration

Configuración conceptual:

```php
'database' => [
    'optimizer' => [
        'profile' => 'standard',

        'rules' => [
            'disable' => [],
            'experimental' => false,
        ],

        'budget' => [
            'max_rule_evaluations' => 10_000,
            'max_rule_applications' => 2_000,
            'max_alternatives' => 256,
        ],
    ],
];
```

Los valores reales serán definidos posteriormente mediante benchmarking.

---

# 191. Configuration ≠ mutable registry

Config selecciona:

```text
RuleSet
```

No modifica descriptors globales.

---

# 192. Fingerprinting

El resultado de optimización deberá considerar:

```text
semantic query fingerprint
rule set fingerprint
rule versions
optimization profile
capability fingerprint
relevant optimizer configuration
extension versions
```

---

# 193. RuleSet fingerprint

Conceptualmente:

```text
RuleSetFingerprint
=
Hash(
    ordered RuleIds
    +
    semantic versions
    +
    profile
)
```

---

# 194. Runtime statistics exclusion

Datos puramente telemetry no deberán entrar en fingerprints semánticos.

---

# 195. Physical statistics

Si estadísticas físicas afectan la selección lógica:

```text
statistics version/fingerprint
```

podrá formar parte del optimization cache key correspondiente.

---

# 196. Testing strategy

Cada regla tendrá tests independientes.

---

# 197. Required rule tests

```text
registration
descriptor
applicability
positive case
negative case
unknown proof
barrier case
capability case
budget case
determinism
persistent-runtime isolation
```

---

# 198. Rule dependency tests

Deberán verificar:

```text
topological ordering
cycle detection
missing dependency detection
```

---

# 199. Conflict tests

Verificar:

```text
mutually exclusive rules
order-sensitive rules
alternative rules
```

---

# 200. Profile tests

Cada profile deberá tener:

```text
expected RuleSet
```

estable.

---

# 201. Extension tests

Las reglas externas deberán pasar:

```text
OptimizationRuleConformanceSuite
```

---

# 202. Conformance suite

Podrá verificar automáticamente:

```text
stable ID
valid semantic version
descriptor completeness
dependency validity
capability declarations
budget compliance
no global mutable state
deterministic output
```

---

# 203. Semantic equivalence testing

Toda regla transformacional deberá demostrar equivalencia mediante:

```text
unit proofs
property tests
metamorphic tests
cross-platform tests
```

según corresponda.

---

# 204. Rule benchmark

Cada regla costosa deberá poder medirse en:

```text
optimizer CPU
optimizer memory
nodes visited
alternatives generated
query execution improvement
```

---

# 205. Benefit benchmark

No basta:

```text
optimizer became smarter
```

Debe medirse cuando sea posible:

```text
before plan cost
after plan cost

before execution time
after execution time
```

en suites controladas.

---

# 206. Rule governance

Una nueva regla core deberá documentar:

```text
RuleId
category
phase
semantic version
purpose
preconditions
required facts
capabilities
barriers
effects
risk
expansion
tests
benchmarks
```

---

# 207. Proposed rule declaration

Conceptualmente:

```php
#[OptimizationRuleDefinition(
    id: 'join.eliminate_redundant',
    version: 1,
    category: OptimizationRuleCategory::JOIN,
    phase: OptimizationRulePhase::RELATIONAL,
    safety: OptimizationRuleSafetyClass::CONSTRAINT_PROVEN,
    risk: OptimizationRuleRiskClass::LOW,
    expansion: OptimizationRuleExpansionClass::SHRINKING,
)]
final class EliminateRedundantJoinRule implements OptimizationRule
{
}
```

El atributo sería metadata declarativa, no el motor de ejecución.

---

# 208. Attributes optional

VoltStack no deberá requerir attributes como única forma de registrar reglas.

También podrá existir:

```text
programmatic registration
```

---

# 209. No reflection on hot path

Si se utilizan attributes:

```text
reflection
```

deberá realizarse en bootstrap/compilation.

No repetidamente por query.

---

# 210. Compiled rule catalog

En producción podrá generarse:

```text
CompiledOptimizationRuleCatalog
```

---

# 211. Compiled catalog benefits

```text
no runtime reflection
prevalidated dependencies
precomputed phase grouping
precomputed target indexes
precomputed profile membership
```

---

# 212. Production architecture

```text
Bootstrap
   │
   ▼
Rule Discovery
   │
   ▼
Rule Validation
   │
   ▼
Compile Rule Catalog
   │
   ▼
Freeze
   │
   ▼
Persistent Worker
   │
   ├── Query A → operation RuleSet
   ├── Query B → operation RuleSet
   └── Query C → operation RuleSet
```

---

# 213. Error handling

Errores de configuración:

```text
DuplicateOptimizationRuleId
InvalidOptimizationRuleVersion
MissingRuleDependency
OptimizationRuleDependencyCycle
InvalidRulePhaseDependency
InvalidRuleConflict
InvalidRuleDescriptor
```

deberán detectarse preferentemente en bootstrap.

---

# 214. Runtime rule failures

Errores internos:

```text
RuleInvariantViolation
RuleProducedInvalidArtifact
RuleSemanticMaintenanceFailure
```

serán considerados:

```text
optimizer internal failures
```

---

# 215. Production fallback

La política podrá permitir:

```text
abort optimization
→ use last known valid artifact
```

cuando sea seguro.

---

# 216. Never compile invalid optimizer state

Nunca:

```text
rule fails halfway
→ compiler receives corrupted tree
```

---

# 217. Debug validation

En DEBUG:

```text
validate after every applied rule
```

podrá habilitarse.

---

# 218. Production validation

En producción podrán utilizarse:

```text
cheaper invariants
```

más validaciones profundas en puntos estratégicos.

---

# 219. Query Optimizer integration

Flujo:

```text
SemanticQueryArtifact
        │
        ▼
OptimizationContextFactory
        │
        ▼
RuleSetResolver
        │
        ▼
OptimizationRuleScheduler
        │
        ▼
Rule Evaluation
        │
        ├── Rewrite System
        ├── Constraint Graph
        ├── Semantic Graph
        ├── Capabilities
        └── Cost Hints
        │
        ▼
OptimizationChangeSet
        │
        ▼
Semantic Maintenance
        │
        ▼
Next Rule / Fixpoint
        │
        ▼
OptimizedQueryArtifact
```

---

# 220. Rule system public visibility

La aplicación normal no deberá manipular directamente:

```text
OptimizationRuleRegistry
```

---

# 221. Developer-facing API

Podrá ofrecerse:

```php
$query->optimizationProfile(
    OptimizationProfile::STANDARD
);
```

o metadata equivalente.

---

# 222. Advanced API

Para herramientas/debugging:

```php
$query->withoutOptimizationRule(
    'join.reorder_inner'
);
```

podría existir.

---

# 223. Production policy

Deshabilitar reglas críticas arbitrariamente podrá estar restringido por configuración.

---

# 224. Optimizer extensions

Paquetes podrán integrar:

```text
DatabaseExtension
        │
        ▼
OptimizationRuleProvider
        │
        ▼
OptimizationRuleRegistry
```

durante bootstrap.

---

# 225. RuleProvider

```php
interface OptimizationRuleProvider
{
    public function registerRules(
        OptimizationRuleRegistry $registry,
    ): void;
}
```

---

# 226. Dependency direction

```text
Optimization Rules
        ↓
Semantic Query Contracts
        ↓
Rewrite Contracts
        ↓
Constraint/Capability Contracts
```

Nunca:

```text
Optimization Rule
→ PDO
```

---

# 227. No ORM dependency

El Query Optimization Rule System no dependerá directamente del ORM.

El ORM produce queries que entran al mismo Query Engine.

---

# 228. ORM provenance

Sí podrá conservar metadata indicando:

```text
ORM-generated query structure
```

si alguna optimización necesita respetar una semantic barrier explícita.

---

# 229. No authentication dependency

El core no dependerá de Authentication.

---

# 230. No authorization dependency

El core no dependerá de Authorization.

---

# 231. No multitenancy dependency

El core no dependerá de Multitenancy.

---

# 232. Integration through metadata

Esos sistemas podrán introducir:

```text
semantic metadata
provenance
security barriers
```

que el Optimizer respeta.

---

# 233. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Rule/
                    ├── Contract/
                    │   ├── OptimizationRule.php
                    │   ├── OptimizationRuleProvider.php
                    │   ├── RuleApplicabilityResolver.php
                    │   ├── RuleCostEstimator.php
                    │   └── RuleBenefitEstimator.php
                    │
                    ├── Definition/
                    │   ├── OptimizationRuleId.php
                    │   ├── OptimizationRuleDescriptor.php
                    │   ├── OptimizationRuleSemanticVersion.php
                    │   ├── OptimizationRuleCategory.php
                    │   ├── OptimizationRuleSafetyClass.php
                    │   ├── OptimizationRuleRiskClass.php
                    │   └── OptimizationRuleExpansionClass.php
                    │
                    ├── Registry/
                    │   ├── OptimizationRuleRegistry.php
                    │   ├── FrozenOptimizationRuleRegistry.php
                    │   ├── OptimizationRuleCatalog.php
                    │   └── CompiledOptimizationRuleCatalog.php
                    │
                    ├── Set/
                    │   ├── OptimizationRuleSet.php
                    │   ├── OptimizationRuleSetResolver.php
                    │   └── OptimizationRuleSetFingerprint.php
                    │
                    ├── Profile/
                    │   ├── OptimizationProfile.php
                    │   └── OptimizationProfileResolver.php
                    │
                    ├── Applicability/
                    │   ├── RuleApplicability.php
                    │   ├── RuleApplicabilityResult.php
                    │   └── DefaultRuleApplicabilityResolver.php
                    │
                    ├── Dependency/
                    │   ├── RuleDependency.php
                    │   ├── RuleDependencyKind.php
                    │   ├── RuleDependencyGraph.php
                    │   └── RuleDependencyValidator.php
                    │
                    ├── Conflict/
                    │   ├── OptimizationRuleConflict.php
                    │   ├── RuleConflictKind.php
                    │   └── RuleConflictResolver.php
                    │
                    ├── Schedule/
                    │   ├── OptimizationRuleScheduler.php
                    │   ├── OptimizationPhase.php
                    │   ├── OptimizationPhasePlan.php
                    │   └── OptimizationWorklist.php
                    │
                    ├── Fixpoint/
                    │   ├── OptimizationFixpointGroup.php
                    │   ├── OptimizationFixpointRunner.php
                    │   └── OptimizationConvergenceTracker.php
                    │
                    ├── Effect/
                    │   ├── OptimizationRuleEffect.php
                    │   └── OptimizationChangeSet.php
                    │
                    ├── Alternative/
                    │   ├── OptimizationAlternativeId.php
                    │   ├── OptimizationAlternative.php
                    │   ├── OptimizationAlternativeSet.php
                    │   └── OptimizationAlternativeDeduplicator.php
                    │
                    ├── Cost/
                    │   ├── OptimizationRuleCost.php
                    │   ├── OptimizationRuleBenefit.php
                    │   └── OptimizationRuleScore.php
                    │
                    ├── Budget/
                    │   ├── OptimizationRuleBudget.php
                    │   └── OptimizationRuleBudgetCounter.php
                    │
                    ├── Result/
                    │   ├── OptimizationRuleResult.php
                    │   └── OptimizationRuleStatus.php
                    │
                    ├── Trace/
                    │   ├── OptimizationRuleTrace.php
                    │   └── OptimizationRuleTraceEntry.php
                    │
                    ├── Statistics/
                    │   └── OptimizationRuleStatistics.php
                    │
                    ├── Diagnostic/
                    │   └── OptimizationRuleDiagnostic.php
                    │
                    ├── Extension/
                    │   └── OptimizationRuleExtensionRegistry.php
                    │
                    ├── Compilation/
                    │   └── OptimizationRuleCatalogCompiler.php
                    │
                    └── Exception/
                        ├── OptimizationRuleException.php
                        ├── DuplicateOptimizationRuleIdException.php
                        ├── MissingRuleDependencyException.php
                        ├── RuleDependencyCycleException.php
                        ├── RuleInvariantViolation.php
                        └── RuleBudgetExceededException.php
```

---

# 234. Ejemplo completo — Predicate Pushdown

Query lógica:

```text
Filter(
    orders.status = 'paid',
    DerivedRelation(
        Project(
            orders.id,
            orders.status,
            orders.total
        )
    )
)
```

Rule:

```text
predicate.pushdown
```

Applicability:

```text
predicate references only derived input symbols
```

Preconditions:

```text
no aggregation barrier
no window barrier
no LIMIT/OFFSET barrier
no DISTINCT semantic barrier
predicate non-volatile
security scope preserved
```

Rewrite:

```text
DerivedRelation(
    Project(
        Filter(
            orders.status = 'paid',
            orders
        )
    )
)
```

---

# 235. Rule result

```text
Status:
    APPLIED

Effects:
    PREDICATE_STRUCTURE
    RELATION_STRUCTURE

Changed:
    Filter#17
    DerivedRelation#4
```

---

# 236. Scheduler reaction

El ChangeSet puede reactivar:

```text
projection.prune_unused
predicate.simplify
relation.simplify
```

---

# 237. Ejemplo completo — Join Elimination

Query:

```text
SELECT orders.id
FROM orders
JOIN customers
    ON customers.id = orders.customer_id
```

Supongamos que:

```text
customers columns unused
```

Eso no basta.

---

# 238. Required facts

La regla requerirá demostrar:

```text
orders.customer_id NOT NULL

orders.customer_id
references customers.id

customers.id UNIQUE

referential integrity guaranteed

join has no security effect

join has no locking effect
```

---

# 239. Resultado

Si todo se demuestra:

```text
SELECT orders.id
FROM orders
```

es semánticamente equivalente bajo el contract establecido.

---

# 240. Unknown fact

Si:

```text
foreign key metadata unavailable
```

entonces:

```text
RuleStatus:
    REJECTED

Reason:
    cardinality preservation unknown
```

---

# 241. Ejemplo completo — Rule conflict

Reglas:

```text
cte.inline
cte.preserve_for_reuse
```

Ambas analizan:

```text
CTE#7
```

El conflict resolver puede determinar:

```text
SEARCH_ALTERNATIVE
```

---

# 242. Alternatives

```text
Alternative A:
    inline CTE

Alternative B:
    preserve CTE
```

El Optimizer podrá conservar ambas si existe budget.

---

# 243. Ejemplo completo — Profile

Query:

```text
A JOIN B JOIN C JOIN D
```

Con:

```text
SAFE
```

puede realizar:

```text
join simplification
outer join strengthening
redundant join elimination
```

pero no búsqueda amplia de join ordering.

Con:

```text
STANDARD
```

puede añadirse:

```text
bounded join reorder
```

Con:

```text
AGGRESSIVE
```

podrá ampliarse el search budget.

---

# 244. Rule lifecycle completo

```text
Rule Source
    │
    ▼
Registration
    │
    ▼
Descriptor Validation
    │
    ▼
Dependency Validation
    │
    ▼
Conflict Validation
    │
    ▼
Catalog Compilation
    │
    ▼
Registry Freeze
    │
    ▼
Query Optimization
    │
    ▼
RuleSet Resolution
    │
    ▼
Applicability
    │
    ▼
Preconditions
    │
    ▼
Scheduling
    │
    ▼
Rule Evaluation
    │
    ▼
ChangeSet / Alternatives
    │
    ▼
Semantic Maintenance
    │
    ▼
Further Scheduling
```

---

# 245. Core invariants

## DB-ORULE-001

Toda regla tendrá `OptimizationRuleId` estable.

## DB-ORULE-002

Toda regla tendrá semantic version.

## DB-ORULE-003

Toda regla tendrá descriptor immutable.

## DB-ORULE-004

Rule category y phase serán conceptos distintos.

## DB-ORULE-005

Optimization Rule no será SQL compiler rule.

## DB-ORULE-006

Optimization Rule no ejecutará queries.

## DB-ORULE-007

Optimization Rule no abrirá conexiones.

## DB-ORULE-008

Optimization Rule no dependerá de PDO.

## DB-ORULE-009

Optimization Rule no inspeccionará SQL strings.

## DB-ORULE-010

RewriteRule y OptimizationRule permanecerán separados.

## DB-ORULE-011

Las reglas transformacionales usarán rewrites semánticamente válidas.

## DB-ORULE-012

Correctness precederá benefit estimation.

## DB-ORULE-013

Una regla nunca será aplicada sólo porque parezca rápida.

## DB-ORULE-014

Applicability será explícita.

## DB-ORULE-015

UNKNOWN no significará APPLICABLE.

## DB-ORULE-016

Barriers serán respetadas.

## DB-ORULE-017

Preconditions requeridas deberán demostrarse.

## DB-ORULE-018

Rule effects serán declarados.

## DB-ORULE-019

Rule dependencies serán explícitas.

## DB-ORULE-020

Rule conflicts serán explícitos.

## DB-ORULE-021

Rule scheduling será determinista.

## DB-ORULE-022

Registration order no definirá scheduling.

## DB-ORULE-023

Stable RuleId podrá utilizarse como tie-break final.

## DB-ORULE-024

Registry será mutable sólo durante bootstrap.

## DB-ORULE-025

Registry será frozen durante runtime.

## DB-ORULE-026

RuleSet será operation-scoped.

## DB-ORULE-027

RuleSet será immutable durante una optimización.

## DB-ORULE-028

Profiles no cambiarán semantic correctness.

## DB-ORULE-029

AGGRESSIVE no significará unsafe.

## DB-ORULE-030

DEBUG no significará mayor agresividad.

## DB-ORULE-031

Experimental rules requerirán habilitación explícita.

## DB-ORULE-032

Rule dependency cycles inválidos fallarán en bootstrap.

## DB-ORULE-033

Missing rule dependencies fallarán en bootstrap.

## DB-ORULE-034

Fixpoint groups serán explícitos.

## DB-ORULE-035

Fixpoint iterations estarán acotadas.

## DB-ORULE-036

Oscillation será detectable.

## DB-ORULE-037

Budgets serán obligatorios.

## DB-ORULE-038

Budget exhaustion preservará artifact válido.

## DB-ORULE-039

Rule cost y query execution cost serán distintos.

## DB-ORULE-040

Benefit estimation nunca sustituirá equivalence proof.

## DB-ORULE-041

Statistics físicas serán opcionales para optimizaciones semánticas.

## DB-ORULE-042

Semantic facts y statistics serán conceptos distintos.

## DB-ORULE-043

Rules declararán target kinds.

## DB-ORULE-044

ChangeSet será explícito.

## DB-ORULE-045

Changed nodes podrán reactivar reglas afectadas.

## DB-ORULE-046

Alternative-producing rules estarán acotadas.

## DB-ORULE-047

Alternatives serán deduplicadas.

## DB-ORULE-048

Alternative logical query no será physical plan.

## DB-ORULE-049

Search explosion será controlada.

## DB-ORULE-050

Rule status distinguirá NOT_APPLICABLE de REJECTED.

## DB-ORULE-051

BLOCKED será distinto de REJECTED.

## DB-ORULE-052

BUDGET_SKIPPED será observable.

## DB-ORULE-053

Diagnostics serán estructurados.

## DB-ORULE-054

Applied rules serán explicables.

## DB-ORULE-055

Rejected rules podrán ser explicables en DEBUG.

## DB-ORULE-056

Telemetry no alterará correctness.

## DB-ORULE-057

Predicate rules respetarán SQL 3VL.

## DB-ORULE-058

Join rules respetarán cardinality semantics.

## DB-ORULE-059

Subquery rules respetarán correlation semantics.

## DB-ORULE-060

CTE rules respetarán materialization semantics.

## DB-ORULE-061

Set rules respetarán bag semantics.

## DB-ORULE-062

Aggregate rules respetarán grouping semantics.

## DB-ORULE-063

Window rules respetarán frame semantics.

## DB-ORULE-064

Ordering rules respetarán observability.

## DB-ORULE-065

Pagination rules serán conservadoras.

## DB-ORULE-066

DML rules preservarán mutation sets.

## DB-ORULE-067

Extension rules usarán el mismo lifecycle.

## DB-ORULE-068

Extensions no tendrán optimizer paralelo.

## DB-ORULE-069

Extensions no podrán bypass barriers.

## DB-ORULE-070

Extension rule IDs serán globalmente no ambiguos.

## DB-ORULE-071

Extension semantic versions serán registradas.

## DB-ORULE-072

Rule discovery podrá compilarse.

## DB-ORULE-073

Reflection no será necesaria en hot path.

## DB-ORULE-074

Compiled Rule Catalog será immutable.

## DB-ORULE-075

Rule configuration no mutará Registry.

## DB-ORULE-076

Per-query rule policy será metadata explícita.

## DB-ORULE-077

Hints no podrán forzar transformaciones inválidas.

## DB-ORULE-078

Required directives fallarán si no pueden satisfacerse.

## DB-ORULE-079

RuleSet fingerprint será determinista.

## DB-ORULE-080

Runtime binding values quedarán fuera del fingerprint.

## DB-ORULE-081

Rule versions afectarán optimization fingerprint.

## DB-ORULE-082

Capability fingerprint podrá afectar RuleSet.

## DB-ORULE-083

Rule testing incluirá negative cases.

## DB-ORULE-084

Rule testing incluirá UNKNOWN cases.

## DB-ORULE-085

Rule testing incluirá barrier cases.

## DB-ORULE-086

Rule testing incluirá capability cases.

## DB-ORULE-087

Rule testing incluirá budget cases.

## DB-ORULE-088

Rule conformance suite será extensible.

## DB-ORULE-089

Transformational rules tendrán equivalence tests.

## DB-ORULE-090

Costly rules tendrán benchmarks.

## DB-ORULE-091

Rule governance requerirá documentación.

## DB-ORULE-092

Attributes serán metadata opcional.

## DB-ORULE-093

Attributes no definirán semántica por sí solos.

## DB-ORULE-094

No habrá runtime reflection obligatoria.

## DB-ORULE-095

Configuration errors se detectarán preferentemente en bootstrap.

## DB-ORULE-096

Rule failures no dejarán optimizer state parcial.

## DB-ORULE-097

Compiler nunca recibirá artifact corrupto.

## DB-ORULE-098

Debug podrá validar invariants después de cada rule.

## DB-ORULE-099

Production podrá usar validación incremental.

## DB-ORULE-100

Rule Registry podrá compartirse entre requests.

## DB-ORULE-101

Rule runtime state será operation-scoped.

## DB-ORULE-102

No existirá global current query.

## DB-ORULE-103

No existirá global mutable RuleSet.

## DB-ORULE-104

Concurrent optimizations estarán aisladas.

## DB-ORULE-105

Temporary rule state será liberable.

## DB-ORULE-106

FrankenPHP será soportado nativamente.

## DB-ORULE-107

RoadRunner será compatible sin rediseño.

## DB-ORULE-108

OpenSwoole será compatible sin rediseño.

## DB-ORULE-109

Security-sensitive rules serán identificables.

## DB-ORULE-110

Security provenance será preservada.

## DB-ORULE-111

Tenant-generated structures serán preservadas correctamente.

## DB-ORULE-112

Authorization-generated structures serán preservadas correctamente.

## DB-ORULE-113

Core optimizer no dependerá de Multitenancy.

## DB-ORULE-114

Core optimizer no dependerá de Authentication.

## DB-ORULE-115

Core optimizer no dependerá de Authorization.

## DB-ORULE-116

Integraciones utilizarán metadata y barriers.

## DB-ORULE-117

Optimization rules no dependerán del ORM.

## DB-ORULE-118

ORM queries usarán el mismo Query Optimizer.

## DB-ORULE-119

Platform-specific decisions usarán capabilities.

## DB-ORULE-120

Vendor conditionals directos estarán prohibidos como diseño central.

## DB-ORULE-121

Rule profiles serán versionables.

## DB-ORULE-122

Phase plans serán deterministas.

## DB-ORULE-123

Rule priorities no serán el único mecanismo de scheduling.

## DB-ORULE-124

Rule conflicts no se resolverán accidentalmente.

## DB-ORULE-125

Search alternatives tendrán identidad.

## DB-ORULE-126

Search alternatives tendrán fingerprints.

## DB-ORULE-127

Duplicate alternatives serán eliminables.

## DB-ORULE-128

Search states estarán limitados por budget.

## DB-ORULE-129

Optimization state nunca será global.

## DB-ORULE-130

Correctness tendrá prioridad absoluta sobre performance.

---

# 246. Anti-patterns

## 246.1 Mega optimizer switch

```php
switch ($node->type) {
    case 'join':
        // 3000 lines...
}
```

**Rechazado.**

---

## 246.2 Vendor-specific rule

```php
if ($connection->driver() === 'pgsql') {
    ...
}
```

**Rechazado como arquitectura central.**

Usar:

```text
PlatformCapabilities
```

---

## 246.3 Mutable runtime registry

```php
$registry->removeRule(...);
```

durante una request.

**Rechazado.**

---

## 246.4 Rule registration order as priority

**Rechazado.**

---

## 246.5 Rule without descriptor

**Rechazado.**

---

## 246.6 Rule without semantic version

**Rechazado.**

---

## 246.7 Rule without applicability contract

**Rechazado.**

---

## 246.8 Rule without budget

para operaciones potencialmente expansivas.

**Rechazado.**

---

## 246.9 Experimental rule in default profile

**Rechazado.**

---

## 246.10 Benefit before correctness

```text
this should be faster
therefore apply it
```

**Rechazado.**

---

## 246.11 Unknown proof treated as true

**Rechazado.**

---

## 246.12 Silent security barrier bypass

**Rechazado.**

---

## 246.13 Physical planner logic inside rule

```text
choose HashJoin
```

**Rechazado.**

---

## 246.14 SQL generation inside rule

**Rechazado.**

---

## 246.15 Global optimizer state

**Rechazado.**

---

# 247. Fórmula del sistema

```text
Optimization Rule System
=
Registry
+
Catalog
+
RuleSet
+
Profiles
+
Applicability
+
Preconditions
+
Dependencies
+
Conflicts
+
Scheduling
+
Budgets
+
Effects
+
Alternatives
+
Diagnostics
+
Extensions
```

---

# 248. Fórmula de selección

```text
SelectedRules(Q)
=
Registry
∩ Profile
∩ QueryFeatures
∩ PlatformCapabilities
∩ Configuration
∩ ExtensionAvailability
```

---

# 249. Fórmula de aplicación

```text
Apply(R, Q)
=
Applicable(R,Q)
∧ PreconditionsProven(R,Q)
∧ BarriersAllow(R,Q)
∧ CapabilitiesAllow(R,Q)
∧ BudgetAllows(R,Q)
∧ StrategySelects(R,Q)
```

---

# 250. Fórmula de corrección

```text
Optimization Correctness
=
Semantic Equivalence
+
Output Contract Preservation
+
NULL Semantics Preservation
+
Multiplicity Preservation
+
Ordering Preservation
+
Security Preservation
+
Mutation Preservation
```

según las características de la query.

---

# 251. Fórmula de extensibilidad

```text
Safe Rule Extension
=
Stable Rule Contract
+
Explicit Descriptor
+
Semantic Version
+
Declared Capabilities
+
Declared Dependencies
+
Declared Effects
+
Conformance Tests
+
Frozen Registration
```

---

# 252. Fórmula persistent-runtime

```text
Persistent-Safe Rule System
=
Frozen Global Catalog
+
Immutable Rule Descriptors
+
Operation-Scoped RuleSet
+
Operation-Scoped Scheduler
+
Operation-Scoped Worklist
+
Operation-Scoped Budgets
+
Operation-Scoped Alternatives
+
No Global Query State
```

---

# 253. Decisión arquitectónica final

VoltStack adoptará un sistema de reglas:

```text
declarative
semantic
versioned
deterministic
dependency-aware
conflict-aware
profile-driven
capability-driven
budgeted
explainable
extensible
persistent-runtime safe
```

El modelo final será:

```text
                    Frozen Rule Registry
                            │
                            ▼
                    Optimization Catalog
                            │
            ┌───────────────┼────────────────┐
            │               │                │
            ▼               ▼                ▼
         Profiles      Capabilities      Extensions
            │               │                │
            └───────────────┼────────────────┘
                            ▼
                         RuleSet
                            │
                            ▼
                  Dependency / Conflict
                         Resolution
                            │
                            ▼
                        Scheduler
                            │
                            ▼
                 Applicability Check
                            │
                            ▼
                     Preconditions
                            │
                            ▼
                    Rule Evaluation
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
             ChangeSet             Alternatives
                │                       │
                └───────────┬───────────┘
                            ▼
                   Semantic Maintenance
                            │
                            ▼
                    Optimization Loop
```

La regla definitiva será:

> **Las optimizaciones de VoltStack no estarán dispersas en `if`, `switch`, compiladores, drivers o builders. Cada optimización será una unidad arquitectónica identificable, versionada, gobernada, verificable y observable dentro de un sistema formal de reglas.**

---

# 254. Relación con documentos anteriores

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE
              │
              │ defines optimizer
              ▼
56_DATABASE_QUERY_REWRITE_SYSTEM
              │
              │ defines safe transformations
              ▼
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM
              │
              │ governs optimization rules
              ▼
Specialized Optimization Systems
```

---

# 255. Especialización siguiente

La siguiente capa comienza a aplicar esta infraestructura a dominios concretos.

Primero:

```text
Predicate Optimization
```

porque los predicates afectan directamente:

```text
filtering
selectivity
join optimization
constraint propagation
subquery decorrelation
index opportunities
cardinality
```

---

# 256. Siguiente documento

```text
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
```

Este documento deberá definir específicamente:

```text
Predicate Optimization Architecture

Predicate Simplification
Predicate Canonical Optimization

Constant Predicate Elimination
Redundant Predicate Elimination
Duplicate Predicate Detection

Contradiction Detection
Tautology Detection

NULL-Aware Optimization
SQL Three-Valued Logic

Null-Rejection Analysis

Predicate Implication
Predicate Subsumption

Equality Propagation
Transitive Predicate Derivation

Range Constraint Optimization

IN Predicate Optimization
BETWEEN Optimization

LIKE / Pattern Optimization

Predicate Pushdown

Join Predicate Pushdown
Derived Table Predicate Pushdown
CTE Predicate Pushdown
Set Operation Predicate Pushdown

Aggregate / HAVING Predicate Movement

Predicate Pull-Up

Predicate Factorization

Predicate Reordering

Volatility-Sensitive Predicates

Security Predicate Barriers

Tenant Predicate Preservation

Predicate Selectivity Hints

ConstraintGraph Integration

SemanticGraph Integration

Predicate Lineage

Optimizer Rule Integration

Budgets
Diagnostics
Telemetry
Testing
Persistent Runtime Safety
```

---

# 257. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md        ← actual
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

# 258. Progresión arquitectónica

```text
Semantic Query
      │
      ▼
Query Optimizer Architecture
      │
      ▼
Rewrite System
      │
      ▼
Optimization Rule System
      │
      ├──────────────┐
      ▼              ▼
Predicate         Join
Optimization      Optimization
      │              │
      └──────┬───────┘
             ▼
     Query Deduplication
             │
             ▼
       Query Cost Hints
             │
             ▼
        Query Planner
```

---

# 259. Resultado

Con este documento VoltStack obtiene una separación clara entre:

```text
Semantic Correctness
        │
        ▼
Rewrite System

Optimization Governance
        │
        ▼
Optimization Rule System

Domain Optimization
        │
        ├── Predicate Optimizer
        ├── Join Optimizer
        ├── Aggregate Optimizer
        └── Window Optimizer

Plan Construction
        │
        ▼
Query Planner
```

Esta separación permitirá que `VoltStack/Quantum/Database` evolucione hacia un optimizador considerablemente más sofisticado sin convertir `QueryBuilder`, `Compiler`, `Connection` u ORM en puntos de acumulación de lógica de optimización.