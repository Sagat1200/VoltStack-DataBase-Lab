# 56_DATABASE_QUERY_REWRITE_SYSTEM.md

# VoltStack Quantum Database
## Query Rewrite System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 56 — Query Rewrite System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Rewrite  
**Versión:** 1.0

---

# 1. Propósito

`Query Rewrite System` define la infraestructura responsable de transformar una representación semántica de consulta en otra representación **demostrablemente equivalente**.

Su responsabilidad principal es:

```text
Query Representation A
        │
        ▼
   Rewrite Rule
        │
        ▼
Query Representation B

with:

Semantics(A) = Semantics(B)
```

El Rewrite System constituye uno de los mecanismos fundamentales utilizados por:

```text
Query Optimizer
```

pero no constituye por sí mismo el Optimizer completo.

La regla principal será:

> **Una rewrite determina qué transformación es semánticamente válida; el Optimizer determina cuándo y por qué conviene aplicarla.**

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
┌──────────────────────────────┐
│     QUERY REWRITE SYSTEM     │
└──────────────────────────────┘
     │
     ▼
Equivalent Query Forms
     │
     ▼
Query Optimizer
     │
     ▼
OptimizedQueryArtifact
     │
     ▼
Query Planner
```

El Rewrite System se encuentra conceptualmente dentro del dominio del Optimizer, pero mantiene contratos propios.

---

# 3. Regla maestra

Toda rewrite deberá cumplir:

```text
Semantics(Q)
=
Semantics(R(Q))
```

donde:

```text
Q    = query original
R    = rewrite
R(Q) = query transformada
```

Formalmente:

```text
Q ≡ R(Q)
```

---

# 4. Rewrite ≠ Normalization

Normalization busca una representación canónica inicial.

Ejemplo:

```text
NOT NOT P
```

podría normalizarse si esa transformación forma parte de las reglas canónicas globales.

Rewrite, en cambio, trabaja después de Semantic Analysis y puede utilizar información como:

```text
resolved symbols
types
constraints
lineage
cardinality facts
correlations
capabilities
volatility
```

Por tanto:

```text
Normalization
=
structure-oriented canonicalization

Rewrite
=
semantic-aware equivalence transformation
```

---

# 5. Rewrite ≠ Semantic Analysis

Semantic Analysis responde:

```text
What does this query mean?
```

Rewrite responde:

```text
Can this query be represented
in another equivalent form?
```

---

# 6. Rewrite ≠ Optimization Rule

Una `RewriteRule` describe:

```text
A → B
```

y las condiciones bajo las cuales:

```text
A ≡ B
```

Una `OptimizationRule` puede decidir aplicar esa rewrite porque:

```text
B
```

es preferible para una determinada estrategia.

---

# 7. Rewrite ≠ Planner transformation

Una rewrite produce:

```text
equivalent logical query representation
```

No produce:

```text
HashJoin
IndexScan
SortAggregate
NestedLoop
```

Eso pertenece al Planner.

---

# 8. Rewrite ≠ SQL generation

Nunca:

```text
RewriteRule
→ SQL string
```

El Compiler conserva esa responsabilidad.

---

# 9. Rewrite ≠ Execution

El Rewrite System:

```text
does not execute queries
does not bind parameters
does not open connections
does not fetch rows
```

---

# 10. Input principal

El Rewrite System opera sobre información derivada de:

```text
SemanticQueryArtifact
```

y, durante optimización, sobre:

```text
OptimizationWorkingArtifact
```

---

# 11. Información semántica disponible

Las reglas podrán consultar:

```text
SemanticQueryGraph
SymbolTable
ScopeTable
QueryTypeTable
RelationSemanticTable
ExpressionSemanticTable
PredicateSemanticTable
ParameterSemanticTable
ConstraintGraph
DependencyGraph
LineageGraph
CorrelationDescriptors
CapabilityRequirements
Security Metadata
Volatility Metadata
```

---

# 12. No semantic guessing

Una rewrite no deberá inferir de forma ad hoc:

```text
probably unique
probably non-null
probably same column
probably same relation
```

Toda propiedad requerida deberá provenir de evidencia semántica explícita.

---

# 13. Arquitectura general

```text
QueryRewriteSystem
│
├── RewriteCoordinator
├── RewriteRegistry
├── RewriteRule
├── RewritePattern
├── RewriteMatcher
├── RewritePrecondition
├── RewriteEvidence
├── RewriteBarrierResolver
├── RewriteScheduler
├── RewriteFixpointRunner
├── RewriteBudget
├── RewriteTrace
├── RewriteStatistics
└── RewriteExtensionRegistry
```

---

# 14. QueryRewriteSystem

Contrato conceptual:

```php
interface QueryRewriteSystem
{
    public function rewrite(
        OptimizationWorkingArtifact $artifact,
        RewriteContext $context,
    ): RewriteResult;
}
```

---

# 15. RewriteRule

Contrato conceptual:

```php
interface RewriteRule
{
    public function id(): RewriteRuleId;

    public function descriptor(): RewriteRuleDescriptor;

    public function matches(
        RewriteTarget $target,
        RewriteContext $context,
    ): bool;

    public function rewrite(
        RewriteTarget $target,
        RewriteContext $context,
    ): RewriteRuleResult;
}
```

---

# 16. RewriteRuleDescriptor

Modelo conceptual:

```php
final readonly class RewriteRuleDescriptor
{
    public function __construct(
        public RewriteRuleId $id,
        public RewriteRuleSemanticVersion $version,
        public RewriteCategory $category,
        public RewritePattern $pattern,
        public RewriteSafetyClass $safety,
        public RewriteCostClass $cost,
        public array $requiredFacts,
        public array $barriers,
    ) {}
}
```

---

# 17. RewriteRuleId

Debe ser estable.

Ejemplos:

```text
predicate.remove_true_conjunct
predicate.remove_false_disjunct
predicate.double_negation
relation.outer_join_strengthening
relation.join_elimination
subquery.exists_to_semijoin
subquery.not_exists_to_antijoin
projection.prune_unused
set.flatten_union_all
```

---

# 18. Semantic version de reglas

Cada regla deberá declarar:

```text
RewriteRuleSemanticVersion
```

Ejemplo:

```text
predicate.remove_true_conjunct@1
```

Cambios que alteren:

```text
matching
preconditions
transformation
semantic assumptions
```

podrán requerir nueva versión.

---

# 19. RewritePattern

Una regla deberá declarar qué estructura reconoce.

Ejemplo conceptual:

```text
AND(
    P,
    TRUE
)
```

---

# 20. Pattern ≠ string matching

No deberá implementarse mediante:

```text
strpos($sql, 'AND TRUE')
```

ni sobre SQL generado.

El matching ocurre sobre estructuras semánticas.

---

# 21. Pattern model

Ejemplo:

```php
final readonly class RewritePattern
{
    public function __construct(
        public QueryNodeKind $rootKind,
        public array $children,
        public array $semanticRequirements = [],
    ) {}
}
```

---

# 22. Pattern variables

Podrán existir:

```text
$predicate
$relation
$expression
$subquery
```

como variables estructurales del patrón.

Ejemplo:

```text
AND($P, TRUE)
```

---

# 23. RewriteMatcher

Responsable de:

```text
pattern
+
candidate node
→
RewriteMatch
```

---

# 24. RewriteMatch

Modelo conceptual:

```php
final readonly class RewriteMatch
{
    public function __construct(
        public RewriteRuleId $rule,
        public QueryNodeId $target,
        public array $bindings,
    ) {}
}
```

Ejemplo:

```text
$P = PredicateNode#17
```

---

# 25. Matching ≠ validity

Que una estructura coincida con el patrón:

```text
matches
```

no significa todavía:

```text
safe to rewrite
```

---

# 26. Rewrite Preconditions

Después del matching deberán comprobarse:

```text
semantic preconditions
```

---

# 27. Ejemplo de precondition

Para eliminar un join:

```text
unused joined relation
```

no es suficiente.

También podría requerirse:

```text
cardinality preserving
referential guarantee
no volatile dependency
no security barrier
no locking effect
```

---

# 28. Precondition contract

```php
interface RewritePrecondition
{
    public function evaluate(
        RewriteMatch $match,
        RewriteContext $context,
    ): RewritePreconditionResult;
}
```

---

# 29. Precondition result

Debe poder representar:

```text
SATISFIED
NOT_SATISFIED
UNKNOWN
```

---

# 30. UNKNOWN

Regla fundamental:

```text
UNKNOWN
≠
SATISFIED
```

Si una rewrite requiere una propiedad y ésta no puede demostrarse:

```text
rewrite rejected
```

---

# 31. Equivalence evidence

Una rewrite aceptada deberá estar asociada conceptualmente a:

```text
RewriteEquivalenceEvidence
```

---

# 32. Evidence types

Ejemplos:

```text
STRUCTURAL_IDENTITY
BOOLEAN_IDENTITY
THREE_VALUED_LOGIC_IDENTITY
TYPE_EQUIVALENCE
CONSTRAINT_PROOF
UNIQUENESS_PROOF
NULL_REJECTION_PROOF
CARDINALITY_PROOF
DEPENDENCY_PROOF
CAPABILITY_PROOF
EXTENSION_PROOF
```

---

# 33. Evidence ≠ runtime formal theorem necessarily

VoltStack no necesita incorporar inicialmente un theorem prover completo.

Pero las reglas deberán estructurar explícitamente:

```text
why transformation is valid
```

---

# 34. RewriteSafetyClass

Podrán existir:

```text
STRUCTURAL
SEMANTIC
CONSTRAINT_DEPENDENT
CAPABILITY_DEPENDENT
VOLATILITY_SENSITIVE
SECURITY_SENSITIVE
```

---

# 35. Structural rewrite

Depende principalmente de la estructura.

Ejemplo:

```text
AND(P, TRUE)
→
P
```

cuando los contracts del Predicate System garantizan la equivalencia.

---

# 36. Constraint-dependent rewrite

Ejemplo:

```text
DISTINCT
```

puede eliminarse si Constraint Analysis demuestra que la salida ya es única.

---

# 37. Capability-dependent rewrite

Una representación puede convertirse en otra únicamente cuando el target posee las capacidades requeridas.

---

# 38. Rewrite categories

VoltStack podrá clasificar reglas en:

```text
EXPRESSION
PREDICATE
RELATIONAL
JOIN
SUBQUERY
CTE
SET_OPERATION
AGGREGATION
WINDOW
PROJECTION
ORDERING
LIMIT
DML
EXTENSION
```

---

# 39. Expression rewrites

Ejemplos potenciales:

```text
x + 0
→ x

x * 1
→ x

CAST(CAST(x AS T) AS T)
→ CAST(x AS T)
```

sólo cuando tipos y semántica lo permitan.

---

# 40. Arithmetic caution

No toda identidad matemática es una identidad válida de query.

Ejemplo:

```text
x * 0
→ 0
```

puede ser incorrecta si evaluar `x` tiene semántica observable.

---

# 41. Volatility example

Si:

```text
x = volatile_function()
```

eliminar su evaluación puede cambiar comportamiento.

---

# 42. Error semantics

Incluso una expresión aparentemente irrelevante podría:

```text
raise database error
```

dependiendo del contrato.

VoltStack deberá definir qué aspectos de error evaluation forman parte de la equivalencia observable.

---

# 43. Conservative V1

V1 deberá preferir:

```text
conservative rewrites
```

cuando exista ambigüedad respecto a evaluación o errores.

---

# 44. Predicate rewrites

Ejemplos:

```text
P AND TRUE
→ P

P OR FALSE
→ P

NOT NOT P
→ P
```

---

# 45. SQL 3VL

Las reglas deberán validarse bajo:

```text
TRUE
FALSE
UNKNOWN
```

---

# 46. Truth table testing

Cada rewrite booleana deberá poder probarse mediante tablas 3VL.

Ejemplo:

```text
P AND TRUE
```

| P | Resultado |
|---|---|
| TRUE | TRUE |
| FALSE | FALSE |
| UNKNOWN | UNKNOWN |

equivalente a:

```text
P
```

---

# 47. Unsafe classical rewrite

No deberán importarse automáticamente identidades desde álgebra booleana clásica sin comprobar SQL 3VL.

---

# 48. Predicate factoring

Ejemplo potencial:

```text
(A AND B)
OR
(A AND C)
```

a:

```text
A
AND
(B OR C)
```

requiere comprobar:

```text
3VL
volatility
evaluation multiplicity
```

---

# 49. Predicate duplication

Una transformación que duplique:

```text
P
```

puede ser inválida si `P` contiene expresiones volátiles.

---

# 50. NULL-aware rewrites

Toda regla relacionada con:

```text
IS NULL
IS NOT NULL
IN
NOT IN
DISTINCT
comparison
```

deberá usar las semánticas definidas por Predicate System y Query Type System.

---

# 51. NOT IN

Regla crítica:

```text
x NOT IN (subquery)
```

no deberá convertirse ingenuamente a:

```text
NOT EXISTS (...)
```

---

# 52. Reason

Porque:

```text
NULL
```

dentro del conjunto puede cambiar el resultado a:

```text
UNKNOWN
```

---

# 53. Relational rewrites

Incluyen:

```text
projection pruning
predicate pushdown
join simplification
join strengthening
join elimination
subquery flattening
```

---

# 54. Relational algebra is SQL-aware

VoltStack no deberá asumir álgebra relacional pura.

Debe considerar:

```text
duplicates
NULL
ordering
limits
windows
volatility
locking
security
```

---

# 55. Predicate pushdown

Transformación conceptual:

```text
Filter(P)
    │
DerivedRelation
```

a:

```text
DerivedRelation(
    Filter(P)
)
```

cuando sea válida.

---

# 56. Pushdown preconditions

Deberán comprobarse:

```text
symbol visibility
lineage
aggregation boundary
window boundary
distinct semantics
limit semantics
outer join semantics
security barriers
volatility
```

---

# 57. Projection pruning

Transformación:

```text
Project(a,b,c,d)
```

a:

```text
Project(a,b)
```

si:

```text
c,d
```

no son requeridos.

---

# 58. Dependency-aware pruning

Debe consultar:

```text
LineageGraph
DependencyGraph
```

---

# 59. Hidden dependencies

Una columna puede ser requerida por:

```text
ORDER BY
GROUP BY
HAVING
WINDOW
JOIN
locking
security
extension semantics
```

aunque no aparezca en la salida pública.

---

# 60. Join rewrites

El Rewrite System podrá representar reglas como:

```text
OuterJoinStrengthening
JoinElimination
JoinAssociativeRewrite
JoinCommutativeRewrite
CrossJoinFilterRewrite
```

---

# 61. INNER JOIN commutativity

Conceptualmente:

```text
A INNER JOIN B
```

puede intercambiarse con:

```text
B INNER JOIN A
```

pero el sistema deberá preservar correctamente:

```text
output mapping
symbol identity
predicate references
metadata
security
```

---

# 62. Join reorder ≠ textual swap

No basta intercambiar dos nodos.

Debe actualizarse la representación semántica asociada.

---

# 63. Outer join

Transformaciones algebraicas serán mucho más restringidas.

---

# 64. OuterJoinStrengthening

Ejemplo:

```text
A LEFT JOIN B ON J
WHERE NullRejectingPredicate(B)
```

puede convertirse a:

```text
A INNER JOIN B ON J
WHERE NullRejectingPredicate(B)
```

si el predicate realmente rechaza toda fila null-extended.

---

# 65. Null rejection proof

Debe provenir de:

```text
PredicateSemanticInfo
+
QueryTypeTable
+
ConstraintGraph
```

---

# 66. Join elimination

Ejemplo:

```text
A JOIN B
```

podrá reducirse a:

```text
A
```

si B:

```text
contributes no observable output
does not filter A
does not multiply A
does not affect locking
does not affect security
does not evaluate observable volatile expressions
```

---

# 67. Cardinality preservation

Será una precondition fundamental.

---

# 68. FK is evidence, not automatic proof

Un FK puede contribuir al proof.

Pero:

```text
foreign key
≠
automatic removable join
```

---

# 69. Subquery rewrites

Ejemplos:

```text
EXISTS → SEMI JOIN
NOT EXISTS → ANTI JOIN
derived table flattening
scalar subquery simplification
subquery predicate pushdown
```

---

# 70. EXISTS decorrelation

Entrada:

```text
Filter(
    Exists(
        CorrelatedSubquery
    )
)
```

Salida lógica potencial:

```text
SemiJoin
```

---

# 71. Correlation proof

Debe existir:

```text
CorrelationDescriptor
```

resuelto por Semantic Analysis.

---

# 72. No string correlation detection

Prohibido:

```text
if child SQL contains parent alias
```

---

# 73. Scalar subquery rewrite

Un scalar subquery podría sustituirse por otra forma sólo cuando se preserve:

```text
one-column contract
at-most-one-row semantics
NULL-on-no-row semantics
error-on-multiple-row semantics
```

cuando aplique.

---

# 74. CTE rewrites

Posibles reglas:

```text
CTE inline
CTE deduplication
CTE projection pruning
predicate propagation
unused CTE elimination
```

---

# 75. Unused CTE elimination

Un CTE no referenciado puede eliminarse sólo si su evaluación no posee semántica observable según el modelo soportado.

---

# 76. Materialization intent

Debe respetarse:

```text
DEFAULT
PREFER_MATERIALIZED
PREFER_INLINE
REQUIRE_MATERIALIZED
REQUIRE_INLINE
```

---

# 77. REQUIRE

Los estados:

```text
REQUIRE_MATERIALIZED
REQUIRE_INLINE
```

son barriers contractuales.

---

# 78. Recursive CTE

No deberá ser inlineado o transformado mediante reglas diseñadas para CTEs no recursivos.

---

# 79. Recursive rewrite rules

Deberán declarar explícitamente:

```text
supportsRecursiveCte = true
```

o equivalente.

---

# 80. Set operation rewrites

Podrán incluir:

```text
UNION chain flattening
UNION ALL chain flattening
operand pruning
projection pushdown
predicate distribution
redundant DISTINCT simplification
```

---

# 81. Set tree grouping

Toda rewrite deberá respetar el árbol explícito.

Ejemplo:

```text
(A UNION B) INTERSECT C
```

no es intercambiable automáticamente con:

```text
A UNION (B INTERSECT C)
```

---

# 82. EXCEPT

Nunca asumir:

```text
A EXCEPT B
=
B EXCEPT A
```

---

# 83. Bag semantics

Para `ALL` deberá preservarse multiplicidad.

---

# 84. Multiplicity formulas

Para un row value `x`:

```text
UNION ALL:

mout(x)
=
mA(x) + mB(x)
```

```text
INTERSECT ALL:

mout(x)
=
min(mA(x), mB(x))
```

```text
EXCEPT ALL:

mout(x)
=
max(mA(x) - mB(x), 0)
```

---

# 85. DISTINCT set operations

Para:

```text
UNION DISTINCT
```

la equivalencia deberá utilizar:

```text
SetRowEquivalenceSemantics
```

no `PredicateNode "="`.

---

# 86. NULL row equivalence

Duplicate elimination tiene semántica distinta de una comparación ordinaria:

```text
NULL = NULL
```

Por ello no deberá implementarse mediante predicates normales.

---

# 87. Aggregation rewrites

Posibles reglas:

```text
HAVING pushdown
aggregate simplification
group-key simplification
redundant grouping elimination
aggregate decomposition
```

---

# 88. HAVING pushdown

Ejemplo:

```text
GROUP BY country
HAVING country = 'MX'
```

podría permitir:

```text
WHERE country = 'MX'
GROUP BY country
```

si el predicate depende sólo de grouping keys y la semántica es preservada.

---

# 89. Aggregate function descriptors

Las propiedades:

```text
decomposable
idempotent
order-sensitive
distinct-sensitive
null-sensitive
```

deberán provenir del descriptor semántico de la función.

---

# 90. No function-name guessing

No:

```php
if ($function->name === 'SUM') {
    ...
}
```

como mecanismo central.

Preferir:

```text
AggregateFunctionDescriptor
```

---

# 91. Window rewrites

Podrán existir transformaciones sobre:

```text
window definitions
partition specifications
ordering
frames
shared windows
```

---

# 92. Window boundaries

Una window expression se evalúa en una fase lógica específica.

No deberá moverse arbitrariamente debajo de:

```text
WHERE
GROUP BY
HAVING
```

---

# 93. Frame semantics

Debe preservarse:

```text
ROWS
RANGE
GROUPS
frame start
frame end
exclusion
peer semantics
```

---

# 94. Projection rewrites

Incluyen:

```text
unused output elimination
expression alias simplification
internal projection merging
```

---

# 95. Public output boundary

Nunca deberá alterarse accidentalmente:

```text
column order
column name
column semantic type
column nullability
```

del output público.

---

# 96. Ordering rewrites

Podrán eliminarse orderings internos únicamente cuando:

```text
ordering is not observable
```

---

# 97. Ordering observability

Puede ser observable por:

```text
LIMIT
OFFSET
window function
ordered aggregate
outer query contract
cursor semantics
```

---

# 98. LIMIT/OFFSET

Una rewrite que atraviese:

```text
LIMIT
OFFSET
```

deberá demostrar equivalencia de selección de filas.

---

# 99. Example unsafe rewrite

No asumir:

```text
Filter(P, Limit(10, R))
=
Limit(10, Filter(P, R))
```

---

# 100. DML rewrites

UPDATE/DELETE podrán utilizar:

```text
predicate simplification
subquery rewrites
join rewrites
```

pero deberán preservar exactamente:

```text
mutation set
```

---

# 101. Mutation equivalence

Para DML:

```text
SemanticEquivalence
```

incluye:

```text
same rows affected
same values written
same returning output
same relevant locking semantics
same mandatory security semantics
```

---

# 102. INSERT...SELECT

El source query puede reescribirse como cualquier relation-producing query.

---

# 103. Conflict semantics

Las rewrites no deberán alterar:

```text
ON CONFLICT
UPSERT
duplicate handling
```

---

# 104. Rewrite barriers

Arquitectura:

```text
Rewrite Target
     │
     ▼
RewriteBarrierResolver
     │
     ├── RAW
     ├── SECURITY
     ├── VOLATILITY
     ├── ORDERING
     ├── LIMIT
     ├── OFFSET
     ├── DISTINCT
     ├── AGGREGATION
     ├── WINDOW
     ├── LOCKING
     ├── CORRELATION
     ├── RECURSION
     ├── MATERIALIZATION
     ├── EXTENSION
     └── CAPABILITY
```

---

# 105. Barrier ≠ always absolute

Una barrier puede tener:

```text
scope
strength
allowed operations
```

---

# 106. Barrier model

```php
final readonly class RewriteBarrier
{
    public function __construct(
        public RewriteBarrierKind $kind,
        public RewriteBarrierStrength $strength,
        public QueryNodeId $owner,
        public array $allowedRewriteClasses = [],
    ) {}
}
```

---

# 107. Barrier strengths

Ejemplo:

```text
ADVISORY
RESTRICTED
HARD
```

---

# 108. Hard barrier

No podrá cruzarse salvo mediante una regla específicamente autorizada por el semantic contract.

---

# 109. Security barrier

Será normalmente:

```text
HARD
```

para transformaciones no declaradas como security-preserving.

---

# 110. Raw barrier

Dependerá del contract de la Raw Expression.

---

# 111. Raw opaque

Una Raw expression totalmente opaca puede bloquear razonamiento local.

---

# 112. Raw annotated

Una Raw expression con:

```text
known type
known dependencies
known volatility
known capabilities
```

puede permitir transformaciones externas limitadas.

---

# 113. Volatility

Clasificación conceptual:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

---

# 114. Unknown volatility

V1 deberá tratar:

```text
UNKNOWN
```

conservadoramente.

---

# 115. Duplicate evaluation

Una rewrite deberá comprobar si cambia:

```text
number of evaluations
```

de una expresión.

---

# 116. Example

Transformar:

```text
P
```

a:

```text
P OR P
```

es lógicamente redundante para valores booleanos, pero puede duplicar evaluación.

No deberá realizarse sin contract que lo permita.

---

# 117. Evaluation order

SQL no garantiza siempre orden de evaluación.

Aun así, VoltStack no deberá realizar rewrites que dependan de un orden no garantizado.

---

# 118. Error-sensitive expressions

Una futura clasificación podrá incluir:

```text
MAY_ERROR
```

para expresiones como divisiones u operaciones con conversiones.

---

# 119. Conservative error preservation

V1 deberá evitar transformaciones que cambien claramente si una expresión potencialmente errónea necesita evaluarse.

---

# 120. Security provenance

Cada predicate o relation introducido por políticas deberá conservar:

```text
provenance
```

---

# 121. Provenance examples

```text
USER_DECLARED
ORM_GENERATED
TENANT_GENERATED
AUTHORIZATION_GENERATED
SECURITY_GENERATED
OPTIMIZER_GENERATED
EXTENSION_GENERATED
```

---

# 122. Optimizer-generated nodes

Una rewrite podrá crear nuevos nodes con:

```text
OPTIMIZER_GENERATED
```

más:

```text
derivedFrom
ruleId
```

---

# 123. Rewrite scheduling

Las reglas no deberán ejecutarse en orden accidental.

---

# 124. RewriteScheduler

Responsable de determinar:

```text
which rule
which target
which phase
which order
```

---

# 125. Scheduling inputs

```text
rule priorities
rule dependencies
phase
node kind
budget
previous changes
fixpoint group
```

---

# 126. Rule priority

Puede existir:

```text
RewritePriority
```

pero no deberá ser el único mecanismo de ordenamiento.

---

# 127. Dependencies

Ejemplo:

```text
SimplifyPredicate
    before
PredicatePushdown
```

---

# 128. Rewrite DAG

Las dependencias producirán:

```text
RewriteRuleDAG
```

---

# 129. Dependency cycle

Si:

```text
A before B
B before C
C before A
```

sin ser fixpoint group:

```text
bootstrap failure
```

---

# 130. Fixpoint rewrites

Algunas reglas deberán repetirse.

Ejemplo:

```text
A AND TRUE AND TRUE
```

puede simplificarse iterativamente.

---

# 131. Fixpoint group

Modelo conceptual:

```php
final readonly class RewriteFixpointGroup
{
    public function __construct(
        public RewriteFixpointGroupId $id,
        public array $rules,
        public int $maxIterations,
    ) {}
}
```

---

# 132. Convergence

La condición preferida será:

```text
structural fingerprint unchanged
```

---

# 133. Semantic fingerprint

En ciertos casos también podrá compararse:

```text
semantic fingerprint
```

---

# 134. Oscillation

Ejemplo:

```text
Rule A:
X → Y

Rule B:
Y → X
```

Debe detectarse.

---

# 135. Rewrite history

El runner podrá conservar un conjunto limitado de:

```text
recent fingerprints
```

---

# 136. Rewrite budget

No deberá existir:

```text
unbounded rewriting
```

---

# 137. Budget dimensions

```text
max rule evaluations
max rule applications
max node visits
max generated nodes
max fixpoint iterations
max nested rewrite depth
max alternative rewrites
```

---

# 138. RewriteBudget

```php
final readonly class RewriteBudget
{
    public function __construct(
        public int $maxRuleEvaluations,
        public int $maxRuleApplications,
        public int $maxGeneratedNodes,
        public int $maxFixpointIterations,
        public int $maxNestedDepth,
    ) {}
}
```

---

# 139. Deterministic budgets

Preferir:

```text
rule evaluations
node visits
generated nodes
```

sobre decisiones dependientes únicamente de tiempo real.

---

# 140. Budget exhaustion

Si el budget termina:

```text
last valid artifact
```

será preservado.

---

# 141. Atomic rewrite application

Una regla deberá comportarse conceptualmente como:

```text
match
validate
build replacement
validate replacement
commit
```

---

# 142. No partial mutation

Nunca:

```text
change half subtree
throw exception
leave working tree inconsistent
```

---

# 143. Transactional rewrite semantics

Conceptualmente:

```text
RewriteAttempt
    │
    ├── success → commit replacement
    └── failure → retain original
```

---

# 144. RewriteRuleResult

Modelo:

```php
final readonly class RewriteRuleResult
{
    public function __construct(
        public RewriteStatus $status,
        public ?QueryNode $replacement,
        public ?RewriteEquivalenceEvidence $evidence,
        public array $semanticUpdates = [],
    ) {}
}
```

---

# 145. RewriteStatus

```text
NO_MATCH
REJECTED
NO_CHANGE
APPLIED
```

---

# 146. NO_MATCH

Pattern no coincide.

---

# 147. REJECTED

Pattern coincide, pero preconditions no están demostradas.

---

# 148. NO_CHANGE

La regla es aplicable pero la representación ya está en forma objetivo.

---

# 149. APPLIED

Se creó una representación equivalente distinta.

---

# 150. Semantic maintenance

Una rewrite puede invalidar información derivada.

---

# 151. Semantic invalidation

Ejemplo:

```text
Join tree changed
```

puede afectar:

```text
relation graph
nullability
constraints
lineage
cardinality facts
```

---

# 152. SemanticUpdatePlan

Cada rewrite deberá declarar qué información:

```text
preserved
remapped
recomputed
invalidated
```

---

# 153. Update model

```php
final readonly class SemanticUpdatePlan
{
    public function __construct(
        public array $preserved,
        public array $remapped,
        public array $recompute,
        public array $invalidate,
    ) {}
}
```

---

# 154. Incremental maintenance

Será preferible cuando sea simple y verificable.

---

# 155. Selective re-analysis

Cuando no sea seguro mantener metadata incrementalmente:

```text
re-run affected semantic passes
```

---

# 156. Full semantic analysis

Deberá evitarse para cada pequeña rewrite por costo.

Pero puede ser fallback de desarrollo o para reglas complejas.

---

# 157. Stale semantic data

Está absolutamente prohibido:

```text
query tree changed
semantic facts unchanged blindly
```

---

# 158. Symbol identity

Una rewrite deberá preservar o remapear explícitamente:

```text
SymbolId
RelationId
ExpressionId
PredicateId
ParameterId
```

cuando corresponda.

---

# 159. Stable IDs

Nodos estructuralmente conservados podrán mantener IDs cuando el contract lo permita.

---

# 160. New nodes

Nodos creados deberán recibir IDs operation-scoped.

---

# 161. No global ID counter

Prohibido:

```php
static int $nextNodeId;
```

---

# 162. Rewrite identity mapping

Podrá producirse:

```text
OldNodeId → NewNodeId
```

para:

```text
debugging
trace
semantic remapping
```

---

# 163. Parameter rewrites

Una regla puede eliminar parámetros innecesarios.

---

# 164. Example

```text
FALSE
AND
column = P17
```

puede simplificarse a:

```text
FALSE
```

si la evaluación de la segunda rama puede eliminarse conforme al semantic contract.

---

# 165. ParameterProjection

El artifact deberá conocer:

```text
required parameters after rewrite
```

---

# 166. Parameter duplication

Duplicar una referencia a:

```text
P17
```

no significa crear otro runtime value.

La identidad del parámetro debe mantenerse correctamente.

---

# 167. Parameter remapping

Si composición estructural exige nuevos IDs:

```text
ParameterRemapping
```

deberá ser explícito.

---

# 168. Runtime value separation

Rewrite System nunca contendrá:

```text
P17 = actual secret value
```

---

# 169. Capability-aware rewriting

Algunas rewrites pueden reducir requisitos de plataforma.

---

# 170. Example

Una operación semántica avanzada podría transformarse a una representación equivalente basada en primitivas más portables.

---

# 171. CapabilityRequirementDelta

Una rewrite podrá declarar:

```text
removed capabilities
added capabilities
preserved capabilities
```

---

# 172. No silent capability degradation

Una rewrite nunca podrá cambiar semántica sólo para hacer una query compatible.

---

# 173. Emulation

Una emulación será válida únicamente si:

```text
semantic equivalence
```

está garantizada.

---

# 174. Portability rewrites

Podrán existir reglas específicamente clasificadas como:

```text
PORTABILITY
```

---

# 175. Extension rewrites

Paquetes podrán registrar:

```text
ExtensionRewriteRule
```

---

# 176. ExtensionRewriteRule

Deberá implementar el mismo contrato base.

---

# 177. Extension metadata

Deberá declarar:

```text
ExtensionId
ExtensionSemanticVersion
RewriteRuleId
RewriteRuleSemanticVersion
TargetNodeKinds
RequiredCapabilities
Barriers
```

---

# 178. Unknown extension node

Si el core no entiende una extension semantic node:

```text
do not rewrite through it
```

salvo contract explícito.

---

# 179. Extension barrier default

La política segura será:

```text
unknown extension semantics
=
rewrite barrier
```

---

# 180. No extension SQL inspection

Una extension no deberá entregar SQL para que el Rewrite System lo analice.

---

# 181. Rewrite trace

Cada aplicación podrá producir:

```text
RewriteTraceEntry
```

---

# 182. Trace model

```php
final readonly class RewriteTraceEntry
{
    public function __construct(
        public RewriteRuleId $rule,
        public QueryNodeId $target,
        public StructuralFingerprint $before,
        public StructuralFingerprint $after,
        public RewriteStatus $status,
        public RewriteReason $reason,
    ) {}
}
```

---

# 183. Trace example

```text
Rule:
    predicate.remove_true_conjunct

Target:
    Predicate#21

Before:
    AND(P17, TRUE)

After:
    P17

Evidence:
    SQL_3VL_IDENTITY
```

---

# 184. Rejected trace

```text
Rule:
    relation.join_elimination

Target:
    Join#8

Status:
    REJECTED

Reason:
    uniqueness not proven
```

---

# 185. Rewrite statistics

Podrán incluir:

```text
rules evaluated
patterns matched
preconditions rejected
rules applied
nodes generated
fixpoint iterations
barriers encountered
semantic reanalysis count
```

---

# 186. Rewrite fingerprint

El resultado podrá depender de:

```text
input semantic fingerprint
rewrite rule set
rule semantic versions
capability fingerprint
extension semantic versions
rewrite profile
```

---

# 187. Runtime values excluded

Los valores del BindingSet no forman parte del fingerprint canónico.

---

# 188. Determinism

Con inputs idénticos:

```text
same rewrite result
```

deberá producirse.

---

# 189. Stable traversal

La visita de nodos deberá usar:

```text
deterministic traversal order
```

---

# 190. Stable matching

Si múltiples reglas aplican:

```text
rule ordering
```

será explícito.

---

# 191. Rewrite conflicts

Dos reglas pueden competir por el mismo subtree.

---

# 192. Conflict resolution

Podrá considerar:

```text
phase
priority
dependency
specificity
stable rule id
```

---

# 193. No registration-order accident

No deberá depender de:

```text
which Composer package booted first
```

---

# 194. Rewrite profiles

Podrán existir:

```text
MINIMAL
SAFE
STANDARD
EXTENDED
DEBUG
```

---

# 195. MINIMAL

Sólo rewrites canónicas y de simplificación trivial.

---

# 196. SAFE

Incluye rewrites con proofs locales fuertes.

---

# 197. STANDARD

Perfil predeterminado.

---

# 198. EXTENDED

Permite más búsquedas y transformaciones complejas.

---

# 199. DEBUG

Activa:

```text
full traces
extra invariant validation
before/after snapshots
```

---

# 200. Extended ≠ unsafe

La corrección no cambia por profile.

Sólo cambia:

```text
amount of optimization work
```

---

# 201. Rewrite testing architecture

Cada regla deberá probarse individualmente.

---

# 202. Test dimensions

```text
positive
negative
unknown-proof
NULL
duplicates
volatility
security
ordering
locking
type boundaries
platform capabilities
budget
```

---

# 203. Truth-table tests

Requeridos para rewrites booleanas relevantes.

---

# 204. Dataset equivalence tests

Podrá ejecutarse:

```text
Original Query
```

y:

```text
Rewritten Query
```

sobre datasets generados.

---

# 205. Equality requirement

Debe compararse según:

```text
query output semantics
```

no necesariamente mediante igualdad PHP ingenua.

---

# 206. Bag comparison

Para queries sin DISTINCT:

```text
multiplicity
```

debe preservarse.

---

# 207. Ordering comparison

Si el output contract garantiza ordering:

```text
sequence
```

también debe preservarse.

---

# 208. NULL comparison

Los tests deberán manejar NULL conforme a SQL result semantics.

---

# 209. DML equivalence tests

Para UPDATE/DELETE:

```text
database state before
→ execute original

database state before
→ execute rewritten
```

deberán producir estados equivalentes.

---

# 210. Property-based testing

Podrán generarse:

```text
query trees
predicates
NULL distributions
duplicate distributions
relation cardinalities
constraints
```

---

# 211. Metamorphic testing

Una regla:

```text
R
```

deberá satisfacer:

```text
Execute(Q, D)
≈
Execute(R(Q), D)
```

para datasets `D` válidos.

---

# 212. Fuzzing

Especialmente útil para:

```text
predicate rewrites
join rewrites
set operation rewrites
subquery rewrites
```

---

# 213. Cross-platform validation

Las reglas con semántica dependiente de capability deberán probarse contra los perfiles soportados:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

---

# 214. Persistent-runtime safety

El Rewrite System deberá funcionar bajo workers persistentes.

---

# 215. Shared immutable state

Puede compartirse:

```text
RewriteRuleDescriptor
compiled RewritePattern
RewriteRuleDAG
frozen RewriteRegistry
```

---

# 216. Operation-local state

Debe ser local:

```text
RewriteContext
RewriteBudgetCounter
RewriteTrace
RewriteStatistics
RewriteMatch state
RewriteFixpoint state
temporary node mappings
```

---

# 217. No current artifact singleton

Prohibido:

```text
RewriteManager::$currentArtifact
```

---

# 218. No static matches

Prohibido:

```text
static $lastMatch
```

---

# 219. No mutable global registry after bootstrap

El registry deberá:

```text
register during bootstrap
freeze
use read-only during runtime
```

---

# 220. Concurrent rewrites

Un mismo registry deberá poder procesar:

```text
Query A
Query B
Query C
```

simultáneamente.

---

# 221. Memory release

Al finalizar deberán ser liberables:

```text
matches
temporary replacements
trace snapshots
fingerprint history
fixpoint history
```

---

# 222. FrankenPHP

Diseño obligatorio:

```text
request N
must not influence
request N+1
```

---

# 223. RoadRunner/OpenSwoole

La misma arquitectura será compatible.

---

# 224. Security

El Rewrite System es una frontera de seguridad importante porque puede mover o eliminar predicates.

---

# 225. Security invariant

Nunca:

```text
security predicate
→ accidentally removed
```

---

# 226. Tenant invariant

Nunca:

```text
tenant isolation predicate
→ moved outside its valid scope
```

---

# 227. Authorization invariant

Nunca:

```text
authorization-generated relation
→ eliminated without policy proof
```

---

# 228. Sensitive data

Rewrite traces no contendrán runtime parameter values.

---

# 229. Complexity attacks

Queries especialmente construidas pueden provocar explosión de:

```text
rewrite candidates
fixpoint iterations
tree size
```

---

# 230. Defensive limits

Se requieren:

```text
budgets
depth limits
generated-node limits
fixpoint limits
```

---

# 231. Rewrite expansion

Algunas reglas pueden aumentar temporalmente el árbol.

Ejemplo:

```text
predicate distribution
```

---

# 232. Expansion factor

Podrá declararse:

```text
RewriteExpansionClass
```

como:

```text
SHRINKING
NEUTRAL
BOUNDED_EXPANSION
HIGH_EXPANSION
```

---

# 233. High-expansion rules

Deberán tener budgets específicos.

---

# 234. Prefer shrinking rewrites early

Una estrategia razonable será:

```text
simplify
prune
reduce
before
expand/search
```

---

# 235. Canonical rewrite phase

Podrá ejecutar:

```text
simple semantic canonicalization
```

antes de rewrites relacionales complejas.

---

# 236. Proposed phases

```text
Rewrite Pipeline

1. Semantic Canonical Rewrite
2. Expression Simplification
3. Predicate Simplification
4. Projection Reduction
5. Relational Simplification
6. Subquery Rewrite
7. Join Rewrite
8. Set Operation Rewrite
9. Aggregation Rewrite
10. Window Rewrite
11. Final Simplification
```

---

# 237. Phase boundaries

No son necesariamente una sola pasada.

Podrán contener fixpoint groups.

---

# 238. Example pipeline

```text
Predicate Simplification
        │
        ▼
Projection Pruning
        │
        ▼
Subquery Flattening
        │
        ▼
Predicate Pushdown
        │
        ▼
Join Simplification
        │
        ▼
Predicate Simplification
```

---

# 239. Why repeated simplification

Una transformación compleja puede generar nuevas oportunidades simples.

---

# 240. Rewrite worklist

En lugar de recorrer siempre todo el árbol, podrá usarse:

```text
RewriteWorklist
```

---

# 241. Worklist

Contiene nodos potencialmente afectados por cambios recientes.

---

# 242. Incremental scheduling

Cuando cambia:

```text
Predicate#10
```

no necesariamente debe reexaminarse toda la query.

---

# 243. V1 implementation recommendation

V1 podrá comenzar con:

```text
deterministic tree traversal
+
bounded fixpoint
```

y evolucionar posteriormente a:

```text
incremental worklist
```

sin cambiar contratos públicos.

---

# 244. Rewrite cache

Una regla local pura puede potencialmente cachear:

```text
input subtree fingerprint
→ output subtree
```

---

# 245. Cache restrictions

Sólo cuando la rewrite no dependa de:

```text
changing scope
statistics
capabilities
security context
extension state
```

o cuando dichos inputs formen parte de la cache key.

---

# 246. Cache ≠ global mutable memo

Una cache persistente será un subsistema explícito futuro.

---

# 247. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Rewrite/
                    ├── Contract/
                    │   ├── QueryRewriteSystem.php
                    │   ├── RewriteRule.php
                    │   ├── RewritePrecondition.php
                    │   └── RewriteMatcher.php
                    │
                    ├── Core/
                    │   ├── DefaultQueryRewriteSystem.php
                    │   ├── RewriteCoordinator.php
                    │   ├── RewriteContext.php
                    │   └── RewriteWorkingState.php
                    │
                    ├── Pattern/
                    │   ├── RewritePattern.php
                    │   ├── RewritePatternNode.php
                    │   ├── RewritePatternVariable.php
                    │   ├── RewriteMatch.php
                    │   └── DefaultRewriteMatcher.php
                    │
                    ├── Rule/
                    │   ├── RewriteRuleId.php
                    │   ├── RewriteRuleDescriptor.php
                    │   ├── RewriteRuleSemanticVersion.php
                    │   ├── RewriteCategory.php
                    │   ├── RewriteSafetyClass.php
                    │   └── RewriteExpansionClass.php
                    │
                    ├── Evidence/
                    │   ├── RewriteEquivalenceEvidence.php
                    │   ├── RewriteEvidenceKind.php
                    │   └── RewriteProofReference.php
                    │
                    ├── Barrier/
                    │   ├── RewriteBarrier.php
                    │   ├── RewriteBarrierKind.php
                    │   ├── RewriteBarrierStrength.php
                    │   └── RewriteBarrierResolver.php
                    │
                    ├── Schedule/
                    │   ├── RewriteScheduler.php
                    │   ├── RewriteRuleDAG.php
                    │   └── RewritePriority.php
                    │
                    ├── Fixpoint/
                    │   ├── RewriteFixpointGroup.php
                    │   ├── RewriteFixpointRunner.php
                    │   └── RewriteConvergenceTracker.php
                    │
                    ├── Worklist/
                    │   └── RewriteWorklist.php
                    │
                    ├── Semantic/
                    │   ├── SemanticUpdatePlan.php
                    │   ├── SemanticInvalidationSet.php
                    │   └── RewriteIdentityMapping.php
                    │
                    ├── Parameter/
                    │   └── RewriteParameterProjection.php
                    │
                    ├── Budget/
                    │   ├── RewriteBudget.php
                    │   └── RewriteBudgetCounter.php
                    │
                    ├── Trace/
                    │   ├── RewriteTrace.php
                    │   ├── RewriteTraceEntry.php
                    │   └── RewriteReason.php
                    │
                    ├── Statistics/
                    │   └── RewriteStatistics.php
                    │
                    ├── Fingerprint/
                    │   └── RewriteFingerprint.php
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
                    ├── Extension/
                    │   └── ExtensionRewriteRegistry.php
                    │
                    ├── Diagnostic/
                    │   └── RewriteDiagnostic.php
                    │
                    └── Exception/
                        ├── QueryRewriteException.php
                        ├── RewriteInvariantViolation.php
                        ├── RewriteDependencyCycleException.php
                        ├── RewriteNonConvergenceException.php
                        └── RewriteBudgetExceededException.php
```

---

# 248. Ejemplo completo I — Predicate simplification

Entrada:

```text
AND(
    active = TRUE,
    TRUE
)
```

Matching:

```text
AND($P, TRUE)

$P = active = TRUE
```

Preconditions:

```text
predicate valid
SQL 3VL identity valid
no special extension semantics
```

Evidence:

```text
THREE_VALUED_LOGIC_IDENTITY
```

Resultado:

```text
active = TRUE
```

---

# 249. Ejemplo completo II — Outer join strengthening

Entrada:

```text
Filter(
    profiles.id IS NOT NULL,
    LeftJoin(
        users,
        profiles,
        profiles.user_id = users.id
    )
)
```

Pattern:

```text
Filter(
    $P,
    LeftJoin($A, $B, $J)
)
```

Preconditions:

```text
$P references nullable side $B

$P is proven null-rejecting for $B

no security barrier prevents rewrite

locking semantics preserved
```

Transformación:

```text
Filter(
    $P,
    InnerJoin(
        $A,
        $B,
        $J
    )
)
```

Evidence:

```text
NULL_REJECTION_PROOF
```

---

# 250. Ejemplo completo III — EXISTS decorrelation

Entrada conceptual:

```sql
SELECT u.id
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
);
```

Semantic representation:

```text
Filter
│
├── UsersRelation
│
└── ExistsPredicate
      │
      └── CorrelatedSubquery
            │
            └── Correlation:
                  o.user_id = outer u.id
```

Rewrite:

```text
LogicalSemiJoin(
    users,
    orders,
    orders.user_id = users.id
)
```

Preconditions:

```text
correlation is decorrelatable
subquery output is irrelevant to EXISTS
duplicate semantics preserved
NULL semantics preserved
security barriers preserved
```

---

# 251. Important SEMI JOIN property

La transformación no deberá producir:

```text
INNER JOIN
```

ingenuamente.

Porque un INNER JOIN puede multiplicar filas de `users`.

`SEMI JOIN` preserva la semántica existencial.

---

# 252. Ejemplo completo IV — NOT EXISTS

```text
NOT EXISTS correlated subquery
```

puede transformarse en:

```text
LogicalAntiJoin
```

si se cumplen las preconditions.

---

# 253. AntiJoin ≠ LEFT JOIN IS NULL blindly

Aunque un Compiler o Planner pueda implementar un AntiJoin de diversas formas, la rewrite semántica deberá producir:

```text
LogicalAntiJoin
```

no una emulación SQL prematura.

---

# 254. Ejemplo completo V — Projection pruning

Entrada:

```text
Parent needs:
    child.id
    child.name

Child outputs:
    id
    name
    email
    created_at
```

Lineage determina:

```text
email
created_at

have no downstream dependency
```

Rewrite:

```text
Child outputs:
    id
    name
```

---

# 255. Example barrier

Si:

```text
ORDER BY created_at
LIMIT 10
```

existe dentro del child:

```text
created_at
```

puede seguir siendo una dependency interna aunque no forme parte del public output.

---

# 256. Rewrite correctness hierarchy

Toda regla deberá superar:

```text
1. Pattern Match
2. Structural Preconditions
3. Semantic Preconditions
4. Barrier Checks
5. Capability Checks
6. Equivalence Evidence
7. Budget Check
8. Replacement Construction
9. Semantic Maintenance
10. Invariant Validation
```

---

# 257. Formal rewrite lifecycle

```text
Candidate Node
      │
      ▼
Pattern Match
      │
      ├── no
      │    └── NO_MATCH
      │
      ▼
Precondition Evaluation
      │
      ├── false/unknown
      │    └── REJECTED
      │
      ▼
Barrier Resolution
      │
      ├── blocked
      │    └── REJECTED
      │
      ▼
Equivalence Evidence
      │
      ├── insufficient
      │    └── REJECTED
      │
      ▼
Budget Reservation
      │
      ▼
Build Replacement
      │
      ▼
Semantic Update
      │
      ▼
Invariant Check
      │
      ├── failure
      │    └── rollback
      │
      ▼
Commit
      │
      ▼
APPLIED
```

---

# 258. Core invariants

## DB-RW-001

Toda rewrite preservará semántica observable.

## DB-RW-002

Rewrite System no será el Normalizer.

## DB-RW-003

Rewrite System no será Semantic Analysis.

## DB-RW-004

Rewrite System no será Planner.

## DB-RW-005

Rewrite System no será Compiler.

## DB-RW-006

Rewrite System no será Executor.

## DB-RW-007

Rewrite System no generará SQL.

## DB-RW-008

Rewrite System no ejecutará queries.

## DB-RW-009

Rewrite System no abrirá conexiones.

## DB-RW-010

Rewrite System operará sobre estructuras semánticas.

## DB-RW-011

No habrá SQL string pattern matching.

## DB-RW-012

Cada rule tendrá identidad estable.

## DB-RW-013

Cada rule tendrá semantic version.

## DB-RW-014

Cada rule declarará pattern.

## DB-RW-015

Pattern match no implicará validez.

## DB-RW-016

Semantic preconditions serán explícitas.

## DB-RW-017

UNKNOWN no satisfará una precondition requerida.

## DB-RW-018

Toda rewrite aplicada tendrá equivalence evidence.

## DB-RW-019

SQL 3VL será preservada.

## DB-RW-020

NULL semantics será preservada.

## DB-RW-021

Bag semantics será preservada.

## DB-RW-022

Duplicate multiplicity será preservada.

## DB-RW-023

Ordering observable será preservado.

## DB-RW-024

Locking observable será preservado.

## DB-RW-025

Volatility será considerada.

## DB-RW-026

Security semantics serán preservadas.

## DB-RW-027

Tenant semantics serán preservadas.

## DB-RW-028

Authorization semantics serán preservadas.

## DB-RW-029

Runtime bindings no serán inspeccionados.

## DB-RW-030

Sensitive values no aparecerán en traces.

## DB-RW-031

Arithmetic identities requerirán type proof.

## DB-RW-032

Potential overflow será considerado.

## DB-RW-033

Precision/scale serán considerados.

## DB-RW-034

Predicate rewrites serán 3VL-safe.

## DB-RW-035

Predicate duplication considerará volatility.

## DB-RW-036

NOT IN no será reescrito ingenuamente a NOT EXISTS.

## DB-RW-037

Predicate pushdown respetará semantic boundaries.

## DB-RW-038

Projection pruning será dependency-aware.

## DB-RW-039

Projection pruning preservará public output.

## DB-RW-040

Join rewrites preservarán relation identity mapping.

## DB-RW-041

Outer joins tendrán reglas específicas.

## DB-RW-042

Outer join strengthening requerirá null-rejection proof.

## DB-RW-043

Join elimination requerirá cardinality proof.

## DB-RW-044

Foreign key solo será evidencia.

## DB-RW-045

EXISTS podrá producir LogicalSemiJoin.

## DB-RW-046

NOT EXISTS podrá producir LogicalAntiJoin.

## DB-RW-047

SEMI JOIN no será reemplazado ingenuamente por INNER JOIN.

## DB-RW-048

Correlation utilizará CorrelationDescriptor.

## DB-RW-049

No se detectará correlation mediante strings.

## DB-RW-050

Scalar subquery rewrites preservarán cardinality contract.

## DB-RW-051

CTE rewrites respetarán materialization intent.

## DB-RW-052

REQUIRE_MATERIALIZED será obligatorio.

## DB-RW-053

REQUIRE_INLINE será obligatorio.

## DB-RW-054

Recursive CTEs usarán reglas explícitamente compatibles.

## DB-RW-055

Set rewrites preservarán tree grouping.

## DB-RW-056

Set rewrites preservarán multiplicity.

## DB-RW-057

EXCEPT preservará directionality.

## DB-RW-058

Set duplicate semantics no usarán Predicate "=".

## DB-RW-059

Aggregate rewrites preservarán grouping semantics.

## DB-RW-060

HAVING pushdown requerirá proof.

## DB-RW-061

Aggregate properties vendrán de descriptors.

## DB-RW-062

Window rewrites preservarán partition semantics.

## DB-RW-063

Window rewrites preservarán ordering semantics.

## DB-RW-064

Window rewrites preservarán frame semantics.

## DB-RW-065

Ordering sólo se eliminará si no es observable.

## DB-RW-066

LIMIT/OFFSET serán rewrite barriers cuando corresponda.

## DB-RW-067

DML rewrites preservarán mutation set.

## DB-RW-068

DML rewrites preservarán returning semantics.

## DB-RW-069

INSERT rewrites preservarán conflict semantics.

## DB-RW-070

Rewrite barriers serán explícitas.

## DB-RW-071

Barrier strength será explícita.

## DB-RW-072

Unknown extension semantics será barrier por default.

## DB-RW-073

Unknown volatility será tratada conservadoramente.

## DB-RW-074

Rules no dependerán de evaluation order no garantizado.

## DB-RW-075

Rules considerarán evaluation multiplicity.

## DB-RW-076

Security provenance será preservada.

## DB-RW-077

Optimizer-generated nodes tendrán provenance.

## DB-RW-078

Rewrite scheduling será determinista.

## DB-RW-079

Rule dependencies serán explícitas.

## DB-RW-080

Dependency cycles inválidos fallarán en bootstrap.

## DB-RW-081

Fixpoint groups serán explícitos.

## DB-RW-082

Fixpoint iterations serán bounded.

## DB-RW-083

Oscillation será detectable.

## DB-RW-084

Rewrite budgets serán obligatorios.

## DB-RW-085

Budgets preferirán deterministic work units.

## DB-RW-086

Budget exhaustion preservará último artifact válido.

## DB-RW-087

Rewrite application será atómica.

## DB-RW-088

No existirán partial mutations.

## DB-RW-089

Semantic information será actualizada o invalidada.

## DB-RW-090

Stale semantic metadata estará prohibida.

## DB-RW-091

Identity remapping será explícito.

## DB-RW-092

No existirán global node ID counters.

## DB-RW-093

Parameter identity será preservada/remapeada explícitamente.

## DB-RW-094

Parameter elimination actualizará required binding projection.

## DB-RW-095

Rewrite System no asignará SQL placeholders.

## DB-RW-096

Capability deltas serán explícitos.

## DB-RW-097

Portability no justificará semantic degradation.

## DB-RW-098

Extension rules implementarán el mismo contract.

## DB-RW-099

Extensions no tendrán un rewrite engine paralelo.

## DB-RW-100

Unknown extension nodes serán barriers por default.

## DB-RW-101

Trace será opcional.

## DB-RW-102

Trace podrá registrar rejected rewrites.

## DB-RW-103

Fingerprint excluirá runtime values.

## DB-RW-104

Rewrite output será determinista.

## DB-RW-105

Traversal order será determinista.

## DB-RW-106

Rule conflict resolution será determinista.

## DB-RW-107

Registration order no definirá semántica.

## DB-RW-108

Extended profile no significará unsafe.

## DB-RW-109

Cada rule tendrá positive tests.

## DB-RW-110

Cada rule tendrá negative tests.

## DB-RW-111

Rules booleanas tendrán 3VL tests.

## DB-RW-112

Set rewrites tendrán multiplicity tests.

## DB-RW-113

DML rewrites tendrán state-equivalence tests.

## DB-RW-114

Property-based testing será soportado.

## DB-RW-115

Persistent runtime state será operation-scoped.

## DB-RW-116

Rule registry será frozen después de bootstrap.

## DB-RW-117

No habrá current-artifact singleton.

## DB-RW-118

Concurrent rewrite operations estarán aisladas.

## DB-RW-119

Temporary rewrite state será liberable.

## DB-RW-120

FrankenPHP worker reuse será soportado.

## DB-RW-121

RoadRunner/OpenSwoole no requerirán rediseño.

## DB-RW-122

Complexity attacks serán mitigados mediante budgets.

## DB-RW-123

Expansion rules declararán expansion class.

## DB-RW-124

High-expansion rules tendrán límites específicos.

## DB-RW-125

Shrinking rules podrán priorizarse antes de expansion.

## DB-RW-126

Worklist optimization podrá añadirse sin cambiar contracts.

## DB-RW-127

Rewrite cache será explícita, nunca accidental.

## DB-RW-128

Public artifacts permanecerán immutable.

## DB-RW-129

Internal controlled mutability será operation-scoped.

## DB-RW-130

Correctness tendrá prioridad sobre rewrite opportunity.

---

# 259. Anti-patterns

## 259.1 SQL regex rewriting

```php
$sql = preg_replace(...);
```

**Rechazado.**

---

## 259.2 Rewrite based on table name

```php
if ($table === 'users') {
    ...
}
```

**Rechazado.**

---

## 259.3 Rewrite based on runtime binding

```php
if ($bindings['status'] === 'active') {
    ...
}
```

en canonical rewrite.

**Rechazado.**

---

## 259.4 Classical boolean assumptions

Ignorar `UNKNOWN`.

**Rechazado.**

---

## 259.5 Blind join elimination

```text
joined columns unused
→ remove join
```

**Rechazado.**

---

## 259.6 Blind EXISTS → INNER JOIN

**Rechazado.**

Debe usarse una representación como:

```text
LogicalSemiJoin
```

---

## 259.7 Blind NOT IN → AntiJoin

**Rechazado.**

NULL semantics debe demostrarse.

---

## 259.8 CTE inline ignoring REQUIRE_MATERIALIZED

**Rechazado.**

---

## 259.9 Flattening EXCEPT chains as commutative

**Rechazado.**

---

## 259.10 Removing ORDER BY without observability analysis

**Rechazado.**

---

## 259.11 Duplicating volatile expression

**Rechazado.**

---

## 259.12 Global rewrite state

**Rechazado.**

---

## 259.13 Unbounded fixpoint

**Rechazado.**

---

## 259.14 Stale semantic graph after rewrite

**Rechazado.**

---

# 260. Fórmula general

```text
Safe Rewrite
=
Pattern Match
+
Semantic Preconditions
+
Barrier Validation
+
Capability Validation
+
Equivalence Evidence
+
Atomic Replacement
+
Semantic Maintenance
+
Invariant Validation
```

---

# 261. Fórmula de matching

```text
Rewrite Candidate
=
Structural Pattern
+
Semantic Pattern
+
Resolved Identities
```

---

# 262. Fórmula de validez

```text
Rewrite Validity
=
Pattern Match
AND
All Required Preconditions Proven
AND
No Blocking Barrier
AND
Required Capabilities Available
```

---

# 263. Fórmula de equivalencia

```text
Query A ≡ Query B
```

requiere preservar, según corresponda:

```text
Values
+
Multiplicity
+
NULL Semantics
+
Output Contract
+
Ordering Guarantees
+
Cardinality
+
Volatility Semantics
+
Locking Semantics
+
Security Semantics
+
Mutation Semantics
```

---

# 264. Fórmula de rewrite segura en runtime persistente

```text
Persistent-Safe Rewrite System
=
Frozen Rule Registry
+
Immutable Rule Descriptors
+
Immutable Input/Output
+
Operation-Scoped Matches
+
Operation-Scoped Budget
+
Operation-Scoped Trace
+
Bounded Fixpoints
+
No Global Mutable Query State
```

---

# 265. Decisión arquitectónica final

VoltStack adoptará un Query Rewrite System:

```text
semantic-aware
rule-based
proof-oriented
deterministic
bounded
barrier-aware
capability-driven
extensible
explainable
persistent-runtime safe
```

La arquitectura seguirá:

```text
Semantic Query
      │
      ▼
Pattern Matching
      │
      ▼
Semantic Preconditions
      │
      ▼
Barrier Validation
      │
      ▼
Equivalence Evidence
      │
      ▼
Atomic Rewrite
      │
      ▼
Semantic Maintenance
      │
      ▼
Equivalent Query
```

La regla definitiva será:

> **VoltStack no considerará una transformación como rewrite válida porque produzca SQL parecido, porque sea una identidad matemática conocida o porque normalmente funcione en un motor concreto. La transformación deberá estar expresada sobre el modelo semántico de la consulta y demostrar que preserva los aspectos observables relevantes de SQL.**

---

# 266. Relación con Query Optimizer

```text
Query Rewrite System
        │
        │ provides
        ▼
Equivalent Transformations
        │
        ▼
Query Optimizer
        │
        │ chooses / schedules
        ▼
Preferred Equivalent Query
```

Por tanto:

```text
Rewrite
=
What transformations are valid?

Optimization
=
Which valid transformations
should be applied?
```

---

# 267. Relación con el siguiente documento

El siguiente documento formalizará la infraestructura que registra, clasifica, ordena y ejecuta las reglas utilizadas por el Optimizer:

```text
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
```

Deberá profundizar en:

```text
OptimizationRule
OptimizationRuleId
OptimizationRuleDescriptor
OptimizationRuleRegistry
OptimizationRuleSet
OptimizationRuleCategory

Rule Lifecycle
Rule Registration
Rule Discovery
Rule Freezing

Rule Preconditions
Rule Effects
Rule Dependencies
Rule Conflicts

Rule Priority
Rule Ordering
Rule DAG
Rule Scheduling

Rule Applicability
Rule Matching
Rule Evaluation

Rule Cost
Rule Benefit
Rule Risk Classification
Rule Expansion Classification

Rule Groups
Rule Phases
Fixpoint Groups

Core Rules
Extension Rules
Platform Capability Rules

Rule Profiles
SAFE
BALANCED
AGGRESSIVE
DEBUG

Rule Versioning
Rule Fingerprints

Rule Diagnostics
Rule Explainability
Rule Telemetry

Rule Testing
Rule Conformance
Rule Security

Persistent Runtime Safety
Extension Governance
```

---

# 268. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md               ← actual
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

# 269. Siguiente documento

```text
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
```

La progresión arquitectónica queda:

```text
SemanticQueryArtifact
        │
        ▼
Query Rewrite System
        │
        └── defines safe equivalent transformations
                    │
                    ▼
Optimization Rule System
        │
        └── governs when/how optimization rules operate
                    │
                    ▼
Predicate Optimization
                    │
                    ▼
Join Optimization
                    │
                    ▼
Query Planner
```