# 58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md

# VoltStack Quantum Database
## Predicate Optimization System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 58 — Predicate Optimization System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Predicate  
**Versión:** 1.0

---

# 1. Propósito

`Predicate Optimization System` define el subsistema encargado de analizar y transformar predicados de una consulta para reducir trabajo innecesario y exponer mejores oportunidades de optimización, sin alterar su semántica observable.

Opera sobre predicados ya:

```text
normalizados
validados
resueltos semánticamente
tipados
vinculados a símbolos
analizados mediante constraints
```

y utiliza la infraestructura definida por:

```text
56_DATABASE_QUERY_REWRITE_SYSTEM
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM
```

Su objetivo puede resumirse como:

```text
Predicate
    │
    ▼
Semantic Facts
    │
    ▼
Constraint Analysis
    │
    ▼
Predicate Optimization
    │
    ├── Simplification
    ├── Deduplication
    ├── Contradiction Analysis
    ├── Implication
    ├── Equality Propagation
    ├── Range Analysis
    ├── Null-Rejection
    ├── Pushdown
    ├── Pull-Up
    ├── Factorization
    └── Reordering
    │
    ▼
Optimized Predicate Structure
```

---

# 2. Regla maestra

Una optimización de predicado será válida únicamente cuando preserve la semántica SQL observable en el contexto concreto donde el predicado se evalúa.

Formalmente:

```text
OptimizePredicate(P, C) = P'

iff

ObservableTruth(P, C)
=
ObservableTruth(P', C)
```

Pero `ObservableTruth` no significa simplemente:

```text
TRUE / FALSE
```

SQL utiliza:

```text
TRUE
FALSE
UNKNOWN
```

Por tanto:

```text
Predicate Optimization
must preserve
SQL Three-Valued Logic
```

---

# 3. Predicate Optimizer ≠ Boolean Simplifier

Ésta es una de las reglas arquitectónicas más importantes.

No debe modelarse SQL como álgebra booleana PHP.

Incorrecto:

```text
SQL Predicate
→ bool
```

Correcto:

```text
SQL Predicate
→ TruthValue
```

donde:

```php
enum SqlTruthValue
{
    case TRUE;
    case FALSE;
    case UNKNOWN;
}
```

---

# 4. SQL 3VL

VoltStack reconocerá explícitamente:

| A | B | A AND B | A OR B |
|---|---|---|---|
| TRUE | TRUE | TRUE | TRUE |
| TRUE | FALSE | FALSE | TRUE |
| TRUE | UNKNOWN | UNKNOWN | TRUE |
| FALSE | TRUE | FALSE | TRUE |
| FALSE | FALSE | FALSE | FALSE |
| FALSE | UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | TRUE | UNKNOWN | TRUE |
| UNKNOWN | FALSE | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |

Y:

| A | NOT A |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

---

# 5. Truth semantics ≠ filter semantics

Existe además una distinción crítica:

```text
Predicate Truth Semantics
```

frente a:

```text
Predicate Filtering Semantics
```

En un `WHERE`:

```text
TRUE     → row survives
FALSE    → row rejected
UNKNOWN  → row rejected
```

Por tanto, ciertas transformaciones pueden ser equivalentes como:

```text
filter predicates
```

sin ser equivalentes como expresiones de verdad generales.

---

# 6. PredicateEvaluationContext

VoltStack modelará el contexto donde se evalúa un predicado.

Propuesta:

```php
enum PredicateEvaluationContext
{
    case WHERE;
    case JOIN_ON;
    case HAVING;
    case FILTER;
    case CASE_CONDITION;
    case CHECK_CONSTRAINT;
    case BOOLEAN_EXPRESSION;
    case SECURITY_POLICY;
    case EXTENSION;
}
```

---

# 7. Context-sensitive optimization

Una regla deberá declarar los contextos donde es válida.

Nunca asumir:

```text
valid in WHERE
=
valid everywhere
```

---

# 8. Predicate Optimizer ≠ Predicate Normalizer

La normalización ocurre anteriormente.

```text
Normalization
=
canonical structural representation
```

Mientras:

```text
Optimization
=
semantics-preserving transformation
intended to improve execution/planning
```

Ejemplo:

```text
Normalization:
    canonicalize operator structure

Optimization:
    remove predicate proven redundant
```

---

# 9. Predicate Optimizer ≠ Constraint Analyzer

`Query Constraint Analysis System` descubre hechos.

El Predicate Optimizer los consume.

```text
Predicate
    │
    ▼
Constraint Analysis
    │
    ▼
Facts
    │
    ▼
Predicate Optimizer
```

El optimizador puede derivar hechos adicionales como consecuencia de transformaciones, pero no deberá crear un segundo `ConstraintGraph` incompatible.

---

# 10. Predicate Optimizer ≠ Query Planner

El Predicate Optimizer puede producir:

```text
more selective logical predicates
better predicate placement
simplified logical structure
```

pero no selecciona:

```text
index scan
bitmap scan
table scan
join algorithm
```

Eso pertenece al Planner.

---

# 11. Predicate Optimizer ≠ SQL Compiler

Nunca:

```text
PredicateOptimizer
→ SQL string
```

La salida continúa siendo representación semántica/lógica estructurada.

---

# 12. Posición arquitectónica

```text
SemanticQueryArtifact
        │
        ▼
Query Optimizer
        │
        ▼
Optimization Rule System
        │
        ▼
┌───────────────────────────────────┐
│ Predicate Optimization System     │
│                                   │
│ ├── Simplification                │
│ ├── Deduplication                 │
│ ├── Contradiction                 │
│ ├── Implication                   │
│ ├── Equality Propagation          │
│ ├── Range Analysis                │
│ ├── Null-Rejection                │
│ ├── Pushdown                      │
│ ├── Pull-Up                       │
│ ├── Factorization                 │
│ └── Reordering                    │
└───────────────────────────────────┘
        │
        ▼
OptimizedQueryArtifact
```

---

# 13. Entradas

El subsistema podrá consumir:

```text
PredicateNode
PredicateSemanticInfo
ExpressionSemanticInfo
QueryTypeTable
SymbolTable
ScopeTable
RelationGraph
ConstraintGraph
DependencyGraph
CapabilitySet
QueryMetadata
OptimizationContext
```

---

# 14. Salida

Las reglas producirán:

```text
OptimizationRuleResult
```

conteniendo, cuando corresponda:

```text
OptimizationChangeSet
PredicateOptimizationProof
PredicateOptimizationDiagnostic
```

---

# 15. Arquitectura interna

```text
PredicateOptimizationSystem
│
├── PredicateSimplifier
├── PredicateDeduplicator
├── PredicateContradictionAnalyzer
├── PredicateImplicationAnalyzer
├── PredicateSubsumptionAnalyzer
│
├── PredicateEqualityPropagator
├── PredicateRangeAnalyzer
├── PredicateNullRejectionAnalyzer
│
├── PredicatePushdownAnalyzer
├── PredicatePullUpAnalyzer
├── PredicateFactorizationAnalyzer
│
├── PredicateReorderAnalyzer
│
├── PredicateVolatilityAnalyzer
├── PredicateBarrierAnalyzer
│
├── PredicateOptimizationProof
├── PredicateOptimizationFacts
└── PredicateOptimizationRules
```

---

# 16. PredicateOptimizationFacts

Modelo conceptual:

```php
final readonly class PredicateOptimizationFacts
{
    public function __construct(
        public PredicateId $predicateId,
        public SqlTruthDomain $possibleTruthValues,
        public bool $isDeterministic,
        public bool $isVolatile,
        public bool $isNullRejecting,
        public bool $isTautology,
        public bool $isContradiction,
        public array $referencedSymbols,
        public array $referencedRelations,
        public array $impliedConstraints,
    ) {}
}
```

---

# 17. Possible truth domain

No siempre conocemos un único valor.

Puede conocerse:

```text
{TRUE}
{FALSE}
{UNKNOWN}
{TRUE, FALSE}
{TRUE, UNKNOWN}
{FALSE, UNKNOWN}
{TRUE, FALSE, UNKNOWN}
```

Esto permite optimizaciones más precisas.

---

# 18. Predicate identity

Cada predicado continuará utilizando:

```text
PredicateId
```

La identidad no se inferirá del texto SQL.

---

# 19. Structural equality ≠ semantic equivalence

Dos predicados pueden tener distinta estructura pero ser equivalentes.

Ejemplo:

```text
a = 5 AND a > 3
```

puede simplificarse a:

```text
a = 5
```

si el sistema de tipos y constraints demuestra que la comparación es compatible.

---

# 20. Semantic equivalence ≠ textual equality

Nunca usar:

```text
(string) $predicateA === (string) $predicateB
```

como base semántica.

---

# 21. Predicate fingerprints

Se utilizarán:

```text
StructuralPredicateFingerprint
SemanticPredicateFingerprint
```

---

# 22. Structural fingerprint

Considerará:

```text
predicate kind
operator
children
expression structure
parameter identities
metadata affecting semantics
```

---

# 23. Semantic fingerprint

Podrá considerar:

```text
resolved symbols
semantic types
collations
domain identities
nullability
operator semantics
capabilities
```

---

# 24. Runtime values

Los valores concretos de runtime no forman parte normalmente del fingerprint estructural.

---

# 25. Constant predicate simplification

Ejemplos seguros:

```text
P AND TRUE
→ P
```

```text
P OR FALSE
→ P
```

```text
NOT TRUE
→ FALSE
```

```text
NOT FALSE
→ TRUE
```

```text
NOT UNKNOWN
→ UNKNOWN
```

---

# 26. UNKNOWN-aware simplification

También:

```text
FALSE AND UNKNOWN
→ FALSE
```

```text
TRUE OR UNKNOWN
→ TRUE
```

Pero:

```text
TRUE AND UNKNOWN
→ UNKNOWN
```

y:

```text
FALSE OR UNKNOWN
→ UNKNOWN
```

---

# 27. Constant ≠ parameter

Esto:

```text
column = :value
```

no es constant predicate simplemente porque el runtime tenga actualmente:

```text
:value = 5
```

---

# 28. Runtime-value-independent optimization

Por defecto:

```text
optimizer
```

no inspeccionará valores concretos de bindings para reescribir estructura.

---

# 29. Specialized optimization

Una futura infraestructura podría permitir:

```text
parameter-specialized plans
```

pero deberá ser explícita y separada del modelo general.

---

# 30. Predicate deduplication

Ejemplo:

```text
P AND P
→ P
```

y:

```text
P OR P
→ P
```

sólo cuando repetir `P` sea semánticamente eliminable.

---

# 31. Volatility caveat

Si `P` contiene una función:

```text
volatile()
```

la duplicación/eliminación puede alterar efectos observables o resultados.

Por tanto:

```text
P AND P
→ P
```

no se aplicará ciegamente.

---

# 32. Function volatility

Las funciones deberán clasificarse mediante metadata semántica como:

```text
IMMUTABLE
STABLE
VOLATILE
SIDE_EFFECT_SENSITIVE
```

según el modelo de VoltStack.

---

# 33. Predicate determinism

Un predicado puede ser:

```text
DETERMINISTIC
STABLE_WITHIN_STATEMENT
NON_DETERMINISTIC
UNKNOWN
```

---

# 34. Predicate reorder

No debe asumirse que:

```text
A AND B
```

puede siempre ejecutarse físicamente en ese orden.

Tampoco deberá prometerse short-circuit SQL.

---

# 35. Logical ordering ≠ physical evaluation order

El optimizador representa:

```text
logical predicate structure
```

El motor de base de datos puede elegir otro orden de evaluación.

---

# 36. Side-effect-sensitive expressions

VoltStack evitará transformaciones que dependan de efectos laterales de expresiones SQL.

Las funciones con side effects deberán actuar como barreras fuertes.

---

# 37. Redundant predicate elimination

Ejemplo:

```text
age > 18
AND age > 10
```

puede reducirse a:

```text
age > 18
```

si:

```text
same resolved expression
same comparison semantics
compatible types
compatible collation/domain
```

---

# 38. Stronger predicate

Definiremos:

```text
P ⇒ Q
```

como:

```text
P implies Q
```

Entonces:

```text
P AND Q
→ P
```

cuando la equivalencia sea válida en el contexto correspondiente.

---

# 39. Predicate implication

El sistema podrá demostrar:

```text
x = 10
⇒ x >= 5
```

---

# 40. Predicate implication engine

No será un theorem prover general.

Será:

```text
bounded
typed
domain-aware
SQL-null-aware
scope-aware
```

---

# 41. Predicate subsumption

Ejemplo:

```text
x > 10
OR x > 5
```

puede convertirse en:

```text
x > 5
```

bajo condiciones semánticas compatibles.

---

# 42. Nullability matters

Si las expresiones pueden ser `NULL`, el análisis deberá comprobar la tabla de verdad completa y el contexto.

---

# 43. Contradiction detection

Ejemplo:

```text
x = 5
AND x = 6
```

puede ser contradicción cuando:

```text
5 ≠ 6
```

bajo el tipo y comparación resueltos.

---

# 44. Contradiction result

Una contradicción puede producir:

```text
FALSE
```

en ciertos contextos de filtrado.

Pero el sistema deberá distinguirla de:

```text
UNKNOWN
```

cuando `NULL` interviene.

---

# 45. Example

```text
x = NULL
```

no deberá interpretarse como:

```text
FALSE
```

Es una construcción cuya semántica depende del operador/modelo y normalmente deberá haberse validado/normalizado adecuadamente.

---

# 46. IS NULL

```text
x IS NULL
```

es distinto de:

```text
x = NULL
```

---

# 47. Tautology detection

Ejemplo:

```text
x IS NULL
OR x IS NOT NULL
```

puede ser tautología bajo la semántica estándar de `IS NULL`.

---

# 48. Dangerous tautology assumption

Esto:

```text
x = 5
OR x <> 5
```

no es necesariamente `TRUE`.

Para:

```text
x = NULL
```

el resultado puede ser:

```text
UNKNOWN
```

---

# 49. Therefore

Nunca aplicar ingenuamente:

```text
P OR NOT P
→ TRUE
```

en SQL 3VL.

---

# 50. Similarly

Nunca asumir:

```text
P AND NOT P
→ FALSE
```

sin considerar `UNKNOWN`.

Puede resultar:

```text
UNKNOWN
```

---

# 51. Null-Rejection Analysis

Un predicado es null-rejecting respecto a un conjunto de símbolos cuando sustituir esos símbolos por `NULL` impide que el predicado sea `TRUE`.

Formalmente:

```text
NullRejecting(P, S)
iff
P[S := NULL] cannot evaluate TRUE
```

---

# 52. Example

```text
customer.id = 10
```

es normalmente null-rejecting respecto a:

```text
customer.id
```

---

# 53. Another example

```text
customer.id IS NULL
```

no es null-rejecting respecto a:

```text
customer.id
```

---

# 54. Null-rejection use cases

Es esencial para:

```text
outer join strengthening
predicate pushdown
join elimination analysis
constraint propagation
```

---

# 55. LEFT JOIN strengthening

Query:

```sql
SELECT ...
FROM orders
LEFT JOIN customers
    ON customers.id = orders.customer_id
WHERE customers.active = TRUE
```

El `WHERE` rechaza filas donde los símbolos de `customers` sean null-extended.

Puede existir oportunidad de:

```text
LEFT JOIN
→ INNER JOIN
```

---

# 56. But proof required

La transformación deberá considerar:

```text
predicate semantics
null extension
volatile expressions
security barriers
join metadata
query context
```

---

# 57. Ownership

La regla concreta de:

```text
LEFT JOIN → INNER JOIN
```

pertenece principalmente a:

```text
Join Optimization System
```

El Predicate Optimization System proporciona:

```text
NullRejectionFacts
```

---

# 58. Equality propagation

Ejemplo:

```text
a = b
AND b = 5
```

permite derivar:

```text
a = 5
```

bajo condiciones válidas.

---

# 59. Equality classes

Podrá utilizarse:

```text
EqualityClass
```

Ejemplo:

```text
{ a, b, c }
```

cuando se haya demostrado:

```text
a = b
b = c
```

con semántica compatible.

---

# 60. NULL caveat for equality classes

SQL equality no equivale automáticamente a una relación matemática total sobre valores nullable.

Por tanto las equality classes deberán registrar:

```text
nullability conditions
comparison semantics
domain compatibility
```

---

# 61. EqualityPropagationFact

Modelo conceptual:

```php
final readonly class EqualityPropagationFact
{
    public function __construct(
        public ExpressionId $left,
        public ExpressionId $right,
        public EqualitySemantics $semantics,
        public NullabilityCondition $nullability,
    ) {}
}
```

---

# 62. Transitive derivation

Podrá derivarse:

```text
a = b
b = c
→
a = c
```

sólo cuando el operador y tipos tengan semántica transitiva aplicable.

---

# 63. Domain types

No eliminar identidad de dominio sólo porque la representación física coincida.

Ejemplo:

```text
CustomerId
OrderId
```

podrían ambos almacenarse como `BIGINT`, pero no ser semánticamente intercambiables.

---

# 64. Range analysis

El sistema podrá construir:

```text
RangeConstraint
```

para expresiones comparables.

Ejemplo:

```text
x >= 10
AND x < 20
```

produce conceptualmente:

```text
[10, 20)
```

---

# 65. Range model

```php
final readonly class RangeConstraint
{
    public function __construct(
        public ExpressionId $expression,
        public ?RangeBound $lower,
        public ?RangeBound $upper,
        public NullAllowance $nullAllowance,
    ) {}
}
```

---

# 66. Range bound

```text
value
inclusive/exclusive
semantic type
comparison semantics
```

---

# 67. Range intersection

```text
x > 5
AND x >= 10
```

puede reducirse a:

```text
x >= 10
```

---

# 68. Empty range

```text
x > 10
AND x < 5
```

produce un rango imposible para valores no-null.

El resultado completo debe seguir considerando semántica de `NULL`.

---

# 69. Range union

En OR:

```text
x < 5
OR x >= 5
```

no deberá simplificarse automáticamente a `TRUE` si `x` puede ser `NULL`.

---

# 70. BETWEEN

```text
x BETWEEN a AND b
```

permanecerá como predicado semántico estructurado.

El optimizador podrá razonar sobre su rango sin necesidad de descomponerlo permanentemente.

---

# 71. BETWEEN ≠ blind rewrite

No será obligatorio transformar siempre:

```text
x BETWEEN a AND b
```

en:

```text
x >= a AND x <= b
```

La representación semántica puede conservar `BETWEEN` y exponer constraints equivalentes.

---

# 72. NOT BETWEEN

Requiere especial cuidado con:

```text
NULL
3VL
```

No se aplicarán reglas de álgebra booleana clásica sin proof.

---

# 73. IN predicate optimization

Ejemplo:

```text
x IN (1, 2, 2, 3)
```

puede deduplicar valores cuando:

```text
comparison semantics
determinism
NULL semantics
```

lo permitan.

---

# 74. IN list deduplication

Podrá transformar:

```text
IN (1, 2, 2, 3)
```

a:

```text
IN (1, 2, 3)
```

cuando los elementos sean semánticamente equivalentes y seguros de deduplicar.

---

# 75. IN with NULL

```text
x IN (1, NULL)
```

tiene semántica 3VL que debe preservarse.

---

# 76. NOT IN warning

`NOT IN` es particularmente sensible a `NULL`.

Nunca aplicar ingenuamente:

```text
NOT IN (subquery)
→ NOT EXISTS (...)
```

---

# 77. NOT IN vs NOT EXISTS

No son universalmente equivalentes.

La transformación sólo podrá realizarse con proof suficiente sobre:

```text
nullability
correlation
comparison semantics
subquery output
```

---

# 78. Empty IN

El comportamiento de:

```text
IN ()
```

debe resolverse antes o mediante representación semántica especial.

No deberá generarse SQL inválido accidentalmente.

---

# 79. IN strategy ≠ predicate optimizer

Decidir si un `IN` grande se implementa mediante:

```text
parameter expansion
array binding
VALUES
temporary relation
```

pertenece a fases posteriores de planning/compilation.

---

# 80. LIKE optimization

El sistema podrá analizar:

```text
LIKE
```

como predicado semántico.

---

# 81. Pattern facts

Podrá derivar información como:

```text
exact pattern
prefix pattern
suffix pattern
contains pattern
general pattern
```

sin decidir todavía el acceso físico.

---

# 82. Example

```text
name LIKE 'John%'
```

podrá producir:

```text
PrefixPatternFact('John')
```

---

# 83. Index selection

El Predicate Optimizer no decidirá:

```text
use index
```

El Planner podrá consumir:

```text
PrefixPatternFact
```

junto con capabilities/statistics/index metadata.

---

# 84. Collation

LIKE/pattern optimization deberá considerar:

```text
collation
case sensitivity
locale
escape semantics
platform capability
```

---

# 85. Regex

Las expresiones regex serán:

```text
capability-driven
```

y generalmente actuarán como transformaciones conservadoras.

---

# 86. Predicate Pushdown

Una de las optimizaciones centrales.

Objetivo:

```text
evaluate filtering as close as safely possible
to its required input
```

---

# 87. Pushdown rule

Formalmente:

```text
Filter(P, Operator(X))
→
Operator(Filter(P, X))
```

sólo si:

```text
SafeToPush(P, Operator)
```

---

# 88. Pushdown prerequisites

Se analizarán:

```text
referenced symbols
relation ownership
scope
operator semantics
cardinality
null extension
volatility
security barriers
ordering barriers
aggregation
windows
set operations
pagination
distinctness
```

---

# 89. Relation-local predicate

Si:

```text
ReferencedRelations(P) = {R}
```

puede existir oportunidad de pushdown hacia `R`.

Pero esto por sí solo no demuestra seguridad.

---

# 90. Projection pushdown

Ejemplo:

```text
Filter(P, Project(X))
```

podrá convertirse en:

```text
Project(Filter(P', X))
```

si las referencias pueden mapearse correctamente a través de la proyección.

---

# 91. Projection lineage

La transformación requerirá:

```text
output → input lineage
```

---

# 92. Derived table pushdown

Ejemplo:

```sql
SELECT *
FROM (
    SELECT id, status
    FROM orders
) o
WHERE o.status = 'paid'
```

puede convertirse lógicamente en:

```text
Derived(
    Filter(
        orders.status = 'paid'
    )
)
```

si no existe barrera.

---

# 93. Derived table barriers

Ejemplos:

```text
DISTINCT
GROUP BY
window functions
LIMIT
OFFSET
locking
volatile expressions
security barriers
set operations
```

dependiendo del predicado.

---

# 94. DISTINCT pushdown

No todo predicado está bloqueado por `DISTINCT`.

Algunas condiciones pueden empujarse de forma segura.

La regla deberá demostrarlo.

---

# 95. LIMIT pushdown

Extremadamente peligroso.

```text
Filter(P, Limit(X))
```

generalmente no equivale a:

```text
Limit(Filter(P, X))
```

---

# 96. Example

```text
take first 10 rows
then filter
```

no es igual a:

```text
filter
then take first 10 rows
```

---

# 97. OFFSET

También constituye una fuerte barrera de cardinalidad/posición.

---

# 98. Join predicate pushdown

Para:

```text
A JOIN B
```

un predicado que sólo referencia `A` puede ser candidato a empujarse hacia `A`.

---

# 99. INNER JOIN

El pushdown suele ser más permisivo, pero sigue requiriendo análisis de:

```text
volatility
security
correlation
extensions
```

---

# 100. OUTER JOIN

Debe considerar:

```text
preserved side
null-supplying side
ON vs WHERE
null-rejection
```

---

# 101. ON ≠ WHERE

Nunca mover ciegamente:

```text
JOIN ON predicate
```

a:

```text
WHERE predicate
```

especialmente con outer joins.

---

# 102. LEFT JOIN example

```sql
A
LEFT JOIN B
    ON A.id = B.a_id
   AND B.active = TRUE
```

no es generalmente equivalente a:

```sql
A
LEFT JOIN B
    ON A.id = B.a_id
WHERE B.active = TRUE
```

---

# 103. Predicate placement is semantic

Por tanto:

```text
PredicatePlacement
```

deberá modelarse explícitamente.

---

# 104. PredicatePlacement

Propuesta:

```php
enum PredicatePlacement
{
    case WHERE;
    case JOIN_ON;
    case HAVING;
    case AGGREGATE_FILTER;
    case SECURITY_FILTER;
    case EXTENSION;
}
```

---

# 105. Set-operation pushdown

Ejemplo:

```text
Filter(P, UNION ALL(A, B))
```

puede convertirse en:

```text
UNION ALL(
    Filter(PA, A),
    Filter(PB, B)
)
```

si:

```text
output mapping is valid
predicate semantics preserved
```

---

# 106. UNION ALL

Es generalmente más sencillo que operaciones con duplicate elimination.

---

# 107. UNION DISTINCT

El pushdown puede seguir siendo válido para ciertos filtros, pero deberá demostrarse respecto a:

```text
duplicate semantics
output coercions
collations
```

---

# 108. INTERSECT / EXCEPT

Requieren reglas específicas.

No asumir simetría con UNION.

---

# 109. CTE pushdown

Un predicado sobre una referencia CTE puede ser candidato a entrar al cuerpo del CTE.

---

# 110. CTE barriers

Se considerarán:

```text
multiple consumers
materialization intent
recursive CTE
volatility
security
reuse
capabilities
```

---

# 111. Recursive CTE

El pushdown dentro de:

```text
anchor
recursive member
```

requiere análisis especializado.

No será una regla genérica V1.

---

# 112. Aggregate pushdown

Un predicado posterior a aggregation no puede moverse automáticamente antes de `GROUP BY`.

---

# 113. HAVING

Ejemplo:

```text
HAVING customer_id = 10
```

podría moverse a `WHERE` si:

```text
customer_id
```

es una grouping key y la transformación preserva semántica.

---

# 114. Aggregate-dependent HAVING

Esto:

```text
HAVING COUNT(*) > 5
```

no puede convertirse simplemente en `WHERE`.

---

# 115. HAVING classification

El sistema podrá clasificar:

```text
GROUP_KEY_ONLY
AGGREGATE_DEPENDENT
MIXED
VOLATILE
UNKNOWN
```

---

# 116. Grouping sets

Con:

```text
GROUPING SETS
ROLLUP
CUBE
```

el pushdown de predicates sobre grouping keys puede ser más delicado debido a:

```text
synthetic nullability
multiple grouping levels
```

---

# 117. Window barrier

Window functions evalúan después del filtrado de su input lógico.

Filtrar antes puede cambiar:

```text
partition contents
rank
row_number
lag/lead
frame contents
```

---

# 118. Example

```text
Filter(
    row_number <= 10,
    Window(...)
)
```

no puede convertirse en:

```text
Window(
    Filter(...)
)
```

sin cambiar semántica.

---

# 119. QUALIFY

Si una futura capability soporta:

```text
QUALIFY
```

deberá modelarse como fase/predicate placement específica.

No será tratado como simple `WHERE`.

---

# 120. Predicate Pull-Up

En algunos casos puede ser útil mover un predicado hacia arriba.

---

# 121. Pull-up use cases

```text
factor common predicates
expose join simplification
avoid duplicated evaluation
construct shared predicate
```

---

# 122. Pull-up safety

Al igual que pushdown:

```text
proof required
```

---

# 123. Predicate factorization

Ejemplo clásico:

```text
(A AND B)
OR
(A AND C)
```

puede factorizarse:

```text
A AND (B OR C)
```

pero SQL 3VL y volatility deberán preservarse.

---

# 124. Factorization goals

Puede:

```text
reduce duplicate expression work
expose pushdown
reduce AST size
```

---

# 125. Expansion vs factorization

El optimizador no deberá expandir indiscriminadamente:

```text
A AND (B OR C)
```

a:

```text
(A AND B) OR (A AND C)
```

porque puede producir explosión exponencial.

---

# 126. CNF/DNF

VoltStack no convertirá por defecto predicados completos a:

```text
CNF
DNF
```

---

# 127. Why

Porque:

```text
predicate size
```

puede crecer exponencialmente.

---

# 128. Bounded local forms

Sí podrán existir:

```text
bounded local normalization/optimization
```

para oportunidades específicas.

---

# 129. Predicate reorder

Reordenar hijos lógicos puede ser útil para:

```text
canonical representation
deduplication
optimizer matching
```

pero no deberá confundirse con orden físico de ejecución.

---

# 130. Canonical ordering

Predicados:

```text
A AND B
```

y:

```text
B AND A
```

pueden compartir una forma canónica cuando:

```text
commutativity is semantically safe
volatility does not forbid it
```

---

# 131. Cost-based predicate ordering

La decisión física de qué filtro ejecutar primero podrá utilizar:

```text
selectivity
cost
```

más adelante.

---

# 132. Predicate selectivity hint

El sistema podrá producir:

```text
PredicateSelectivityHint
```

---

# 133. Hint ≠ statistic

Ejemplo semántico:

```text
primary_key = constant
```

puede sugerir:

```text
high selectivity
```

pero el Planner podrá combinarlo con statistics reales.

---

# 134. Selectivity classes

V1 puede utilizar:

```text
VERY_HIGH
HIGH
MEDIUM
LOW
UNKNOWN
```

como hints cualitativos.

---

# 135. Equality to unique key

Si:

```text
x UNIQUE
AND x = value
```

el Constraint System puede derivar:

```text
cardinality <= 1
```

más fuerte que un simple selectivity hint.

---

# 136. Constraint ≠ heuristic

```text
cardinality <= 1
```

es hecho lógico.

```text
probably selective
```

es heuristic.

Nunca mezclarlos.

---

# 137. Predicate implication graph

Para queries complejas podrá construirse:

```text
PredicateImplicationGraph
```

---

# 138. Example

```text
P1: x = 10
P2: x > 5
P3: x IS NOT NULL
```

Edges:

```text
P1 → P2
P1 → P3
```

---

# 139. Uses

Permite:

```text
redundancy elimination
subsumption
constraint derivation
pushdown opportunity discovery
```

---

# 140. Graph boundedness

No deberá crecer sin límites.

---

# 141. ConstraintGraph integration

La arquitectura será:

```text
Semantic Query
      │
      ▼
ConstraintGraph
      │
      ├── equality
      ├── nullability
      ├── ranges
      ├── uniqueness
      ├── FKs
      └── implications
      │
      ▼
Predicate Optimizer
```

---

# 142. No duplicate truth source

El Predicate Optimizer no mantendrá un segundo sistema independiente de constraints.

---

# 143. Incremental constraint maintenance

Después de una transformación:

```text
OptimizationChangeSet
```

podrá invalidar o añadir facts.

---

# 144. Example

Eliminar:

```text
x > 5
```

porque:

```text
x = 10
```

lo implica no significa eliminar el fact:

```text
x > 5
```

El constraint puede seguir siendo derivable.

---

# 145. SemanticGraph integration

Los predicates estarán conectados con:

```text
symbols
relations
expressions
scopes
constraints
dependencies
lineage
```

---

# 146. Predicate dependency

Un predicado puede depender de:

```text
columns
parameters
functions
subqueries
outer symbols
collations
extensions
```

---

# 147. Dependency ≠ lineage

Predicate dependency indica:

```text
what is needed to evaluate predicate
```

Lineage describe:

```text
where semantic value/information originates
```

---

# 148. Correlated predicates

Un predicate dentro de subquery puede referenciar outer symbols.

No deberá empujarse fuera de su correlation boundary sin una transformación formal de decorrelation.

---

# 149. Correlation barrier

```text
CorrelationDescriptor
```

deberá ser respetado.

---

# 150. EXISTS predicates

El sistema puede simplificar ciertos `EXISTS` cuando hechos de cardinalidad lo permiten.

---

# 151. Example

Si una subquery se demuestra:

```text
always empty
```

entonces:

```text
EXISTS(subquery)
→ FALSE
```

---

# 152. Always non-empty

Si se demuestra que la subquery siempre produce al menos una fila:

```text
EXISTS(subquery)
→ TRUE
```

si no existen efectos observables que lo impidan.

---

# 153. EXISTS projection

Las columnas proyectadas dentro de `EXISTS` generalmente no afectan existencia.

Projection pruning puede reducirlas.

Esto deberá coordinarse con:

```text
Projection Optimization
Subquery Optimization
```

---

# 154. Security predicates

Predicados introducidos por:

```text
authorization
tenant isolation
row-level policy
```

deberán identificarse mediante provenance.

---

# 155. Predicate provenance

Propuesta:

```text
USER_QUERY
ORM_SCOPE
TENANT_POLICY
AUTHORIZATION_POLICY
SECURITY_POLICY
FRAMEWORK_POLICY
OPTIMIZER_DERIVED
EXTENSION
```

---

# 156. Security predicate elimination

Un security predicate no deberá eliminarse sólo porque parezca redundante localmente.

---

# 157. Example

```text
tenant_id = 10
AND tenant_id = 10
```

puede parecer duplicado.

Pero si uno representa una:

```text
security enforcement boundary
```

la metadata deberá preservarse aunque la expresión física pueda eventualmente deduplicarse.

---

# 158. Semantic predicate ≠ policy evidence

La estructura lógica y la evidencia de que una política fue aplicada son conceptos distintos.

---

# 159. Policy provenance preservation

Si se deduplican predicados:

```text
P_user
P_security
```

semánticamente equivalentes, el predicado resultante deberá conservar provenance combinada cuando corresponda.

---

# 160. Security barriers

Un:

```text
SecurityBarrier
```

puede prohibir:

```text
pushdown
pull-up
factorization
deduplication across boundary
```

---

# 161. Multitenancy

El core no dependerá directamente de Multitenancy.

Pero podrá reconocer:

```text
TENANT_POLICY provenance
```

mediante metadata genérica.

---

# 162. Raw predicates

Un:

```text
RawPredicateNode
```

será una barrera de optimización por defecto.

---

# 163. Raw opacity

El optimizador no intentará interpretar texto raw como:

```text
SQL semantics
```

---

# 164. Example

```php
$query->whereRaw('price > cost * 1.2');
```

no deberá ser parseado improvisadamente por el Predicate Optimizer.

---

# 165. Raw metadata

Una extensión avanzada podría adjuntar:

```text
semantic facts
dependencies
volatility
nullability
```

al raw node.

Aun así seguirá siendo una frontera explícita.

---

# 166. Extension predicates

Predicados custom deberán registrar:

```text
semantic descriptor
truth behavior
type requirements
null behavior
volatility
capabilities
optimizer hooks
```

---

# 167. Unknown extension

Si el optimizador desconoce la semántica:

```text
treat as optimization barrier
```

No:

```text
guess
```

---

# 168. Capability model

Propuesta:

```text
QUERY.OPTIMIZER.PREDICATE.BASIC
QUERY.OPTIMIZER.PREDICATE.IMPLICATION
QUERY.OPTIMIZER.PREDICATE.RANGE
QUERY.OPTIMIZER.PREDICATE.NULL_REJECTION
QUERY.OPTIMIZER.PREDICATE.PUSHDOWN
QUERY.OPTIMIZER.PREDICATE.JOIN_PUSHDOWN
QUERY.OPTIMIZER.PREDICATE.SUBQUERY_PUSHDOWN
QUERY.OPTIMIZER.PREDICATE.CTE_PUSHDOWN
QUERY.OPTIMIZER.PREDICATE.SET_PUSHDOWN
QUERY.OPTIMIZER.PREDICATE.AGGREGATE_MOVEMENT
QUERY.OPTIMIZER.PREDICATE.FACTORIZATION
```

---

# 169. Internal vs platform capabilities

Muchas optimizaciones son capabilities de VoltStack:

```text
optimizer capability
```

no de la base de datos.

Otras dependen de capacidades del target.

Ambas deberán distinguirse.

---

# 170. Platform behavior

Comparaciones pueden depender de:

```text
collation
type semantics
NULL behavior extensions
operator semantics
```

Por tanto el optimizador consume:

```text
PlatformCapabilitySystem
```

sin condicionales por vendor.

---

# 171. PredicateOptimizationRule

Base conceptual:

```php
interface PredicateOptimizationRule extends OptimizationRule
{
    public function predicateKinds(): array;
}
```

---

# 172. Core rule catalog

V1 puede incluir:

```text
predicate.constant_simplification
predicate.logical_identity
predicate.deduplicate
predicate.remove_redundant
predicate.detect_contradiction
predicate.detect_tautology
predicate.implication
predicate.subsumption
predicate.equality_propagation
predicate.range_intersection
predicate.range_subsumption
predicate.in_deduplication
predicate.null_rejection
predicate.pushdown.projection
predicate.pushdown.join
predicate.pushdown.derived
predicate.pushdown.set
predicate.pushdown.cte
predicate.having_to_where
predicate.factor_common
```

---

# 173. Rule ordering

Ejemplo:

```text
Constant Simplification
        │
        ▼
Deduplication
        │
        ▼
Constraint Extraction
        │
        ▼
Implication
        │
        ▼
Range Simplification
        │
        ▼
Null-Rejection
        │
        ▼
Pushdown
        │
        ▼
Simplification
```

---

# 174. Fixpoint

Predicate optimization probablemente requerirá:

```text
bounded fixpoint
```

porque una transformación puede descubrir otra.

---

# 175. Example

```text
pushdown
    ↓
new local constraints
    ↓
redundancy elimination
    ↓
join strengthening
    ↓
new pushdown opportunity
```

---

# 176. Cross-optimizer coordination

El Predicate Optimizer deberá coordinarse con:

```text
Join Optimizer
Subquery Optimizer
Aggregate Optimizer
Window Optimizer
Set Optimizer
Projection Optimizer
```

mediante:

```text
OptimizationChangeSet
Rule Scheduler
```

No mediante llamadas circulares ad hoc.

---

# 177. Predicate Optimization Phase

Propuesta:

```text
PHASE P1
Local Simplification

PHASE P2
Constraint-Based Simplification

PHASE P3
Null-Rejection Analysis

PHASE P4
Predicate Movement

PHASE P5
Post-Movement Simplification
```

---

# 178. P1 — Local Simplification

Incluye:

```text
constants
logical identities
safe deduplication
local NOT simplification
```

---

# 179. P2 — Constraint-Based Simplification

Incluye:

```text
implication
subsumption
equality propagation
ranges
contradictions
```

---

# 180. P3 — Null-Rejection

Produce facts para:

```text
outer joins
pushdown
join strengthening
```

---

# 181. P4 — Predicate Movement

Incluye:

```text
pushdown
pull-up
HAVING movement
set distribution
```

---

# 182. P5 — Final Simplification

Después del movimiento pueden aparecer:

```text
duplicates
new contradictions
new range intersections
```

---

# 183. Budget model

El subsistema deberá tener límites sobre:

```text
predicates visited
logical terms
implication edges
range facts
equality classes
derived predicates
pushdown attempts
factorization expansions
fixpoint iterations
```

---

# 184. PredicateOptimizationBudget

Modelo conceptual:

```php
final readonly class PredicateOptimizationBudget
{
    public function __construct(
        public int $maxPredicates,
        public int $maxLogicalTerms,
        public int $maxDerivedPredicates,
        public int $maxImplicationEdges,
        public int $maxRangeFacts,
        public int $maxPushdownAttempts,
        public int $maxFixpointIterations,
    ) {}
}
```

---

# 185. Derived predicate budget

Especialmente importante para:

```text
transitive derivation
```

---

# 186. Example explosion

Si existen:

```text
a = b
b = c
c = d
...
```

no es necesario materializar todas las comparaciones transitivas posibles.

---

# 187. Compact facts

Preferir:

```text
EqualityClass
```

sobre generar:

```text
a=b
a=c
a=d
b=c
b=d
...
```

---

# 188. Factorization budget

No generar árboles exponenciales.

---

# 189. Budget exhaustion

Debe producir:

```text
valid partially optimized query
```

no error semántico.

---

# 190. Diagnostics

Propuesta:

```text
PredicateOptimizationBudgetExceeded
PredicatePushdownBlocked
PredicatePushdownBlockedByLimit
PredicatePushdownBlockedByWindow
PredicatePushdownBlockedBySecurity
PredicatePushdownBlockedByVolatility
PredicateImplicationUnknown
PredicateNullSemanticsPreventRewrite
PredicateRangeComparisonUnsupported
PredicateCollationPreventsRewrite
PredicateExtensionBarrier
PredicateCorrelationBarrier
```

---

# 191. Explainability

Ejemplo:

```text
Rule:
    predicate.remove_redundant

Removed:
    age > 10

Proof:
    age > 18 implies age > 10
```

---

# 192. Pushdown explanation

```text
Rule:
    predicate.pushdown.derived

Predicate:
    orders.status = 'paid'

Moved:
    outer WHERE
    →
    derived relation input

Proof:
    references only derived input symbol
    no DISTINCT barrier
    no LIMIT/OFFSET
    no window
    non-volatile
```

---

# 193. Blocked explanation

```text
Rule:
    predicate.pushdown.derived

Status:
    BLOCKED

Reason:
    derived relation contains LIMIT
```

---

# 194. Null rejection explanation

```text
Predicate:
    customers.active = TRUE

Relation:
    customers

Null-Rejecting:
    YES

Reason:
    null-extension cannot make predicate TRUE
```

---

# 195. Telemetry

Podrán recopilarse:

```text
predicates analyzed
predicates removed
predicates derived
contradictions found
pushdowns applied
pushdowns blocked
null-rejection facts
implication proofs
range reductions
optimization iterations
```

---

# 196. Telemetry privacy

No registrar por defecto:

```text
sensitive parameter values
```

---

# 197. Parameter redaction

Los traces deberán mostrar:

```text
email = :p1
```

no necesariamente:

```text
email = 'user@example.com'
```

---

# 198. Persistent runtime safety

Compartible:

```text
rule descriptors
operator semantic descriptors
immutable capability metadata
```

Operation-scoped:

```text
predicate facts
implication graph
range analysis
equality classes
pushdown worklist
diagnostics
budgets
```

---

# 199. No global PredicateContext

Prohibido:

```php
PredicateOptimizer::$currentScope
```

---

# 200. No global EqualityClass

Las equality classes pertenecen a una operación/query concreta.

---

# 201. No cross-request facts

Nunca reutilizar:

```text
predicate facts
```

de una query anterior sólo porque la estructura se parezca.

---

# 202. Cacheable analysis

Si en el futuro se cachea:

```text
predicate semantic analysis
```

deberá estar correctamente keyed por:

```text
semantic fingerprint
schema fingerprint
capabilities
rule versions
```

---

# 203. Thread/concurrency safety

El subsistema deberá funcionar bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

sin estado mutable compartido por request.

---

# 204. Testing

La suite deberá cubrir:

```text
2VL traps
3VL truth tables
NULL predicates
NOT IN
outer joins
HAVING
window barriers
LIMIT/OFFSET
DISTINCT
set operations
CTEs
correlation
volatile functions
security predicates
raw predicates
```

---

# 205. Property-based testing

Especialmente útil para:

```text
logical rewrites
```

Generar combinaciones de:

```text
TRUE
FALSE
UNKNOWN
```

y comprobar equivalencia.

---

# 206. Metamorphic testing

Para una transformación:

```text
P → P'
```

verificar:

```text
Eval(P, row)
=
Eval(P', row)
```

sobre datasets generados con:

```text
NULL
boundary values
duplicate values
collation-sensitive values
```

---

# 207. Cross-platform testing

MySQL/MariaDB/PostgreSQL/SQLite podrán diferir en ciertas capabilities.

Las reglas deberán probarse contra:

```text
semantic profile
+
platform capability profile
```

---

# 208. Golden tests

Para explainability podrán existir snapshots de:

```text
before predicate
after predicate
proof
diagnostic
```

---

# 209. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Predicate/
                    ├── Contract/
                    │   ├── PredicateOptimizationRule.php
                    │   ├── PredicateAnalyzer.php
                    │   └── PredicateMovementRule.php
                    │
                    ├── Model/
                    │   ├── PredicateOptimizationFacts.php
                    │   ├── PredicateEvaluationContext.php
                    │   ├── PredicatePlacement.php
                    │   ├── SqlTruthValue.php
                    │   └── SqlTruthDomain.php
                    │
                    ├── Simplification/
                    │   ├── PredicateSimplifier.php
                    │   ├── ConstantPredicateSimplifier.php
                    │   ├── PredicateDeduplicator.php
                    │   └── PredicateRedundancyAnalyzer.php
                    │
                    ├── Implication/
                    │   ├── PredicateImplicationAnalyzer.php
                    │   ├── PredicateImplicationGraph.php
                    │   └── PredicateSubsumptionAnalyzer.php
                    │
                    ├── Equality/
                    │   ├── EqualityClass.php
                    │   ├── EqualityPropagationFact.php
                    │   └── PredicateEqualityPropagator.php
                    │
                    ├── Range/
                    │   ├── RangeConstraint.php
                    │   ├── RangeBound.php
                    │   ├── PredicateRangeAnalyzer.php
                    │   └── RangeIntersectionResolver.php
                    │
                    ├── Null/
                    │   ├── PredicateNullRejectionAnalyzer.php
                    │   └── NullRejectionFact.php
                    │
                    ├── Movement/
                    │   ├── PredicatePushdownAnalyzer.php
                    │   ├── PredicatePullUpAnalyzer.php
                    │   ├── PredicateMovementDecision.php
                    │   ├── PredicateBarrierAnalyzer.php
                    │   └── PredicatePlacementResolver.php
                    │
                    ├── Factorization/
                    │   └── PredicateFactorizationAnalyzer.php
                    │
                    ├── Selectivity/
                    │   ├── PredicateSelectivityHint.php
                    │   └── PredicateSelectivityAnalyzer.php
                    │
                    ├── Rule/
                    │   ├── ConstantSimplificationRule.php
                    │   ├── PredicateDeduplicationRule.php
                    │   ├── RedundantPredicateRule.php
                    │   ├── PredicateImplicationRule.php
                    │   ├── EqualityPropagationRule.php
                    │   ├── RangeSimplificationRule.php
                    │   ├── NullRejectionRule.php
                    │   ├── ProjectionPredicatePushdownRule.php
                    │   ├── JoinPredicatePushdownRule.php
                    │   ├── DerivedPredicatePushdownRule.php
                    │   ├── SetPredicatePushdownRule.php
                    │   ├── CtePredicatePushdownRule.php
                    │   └── HavingPredicateMovementRule.php
                    │
                    ├── Proof/
                    │   └── PredicateOptimizationProof.php
                    │
                    ├── Budget/
                    │   └── PredicateOptimizationBudget.php
                    │
                    ├── Diagnostic/
                    │   └── PredicateOptimizationDiagnostic.php
                    │
                    └── Exception/
                        └── PredicateOptimizationException.php
```

---

# 210. Ejemplo — simplificación por rango

Entrada:

```text
WHERE
    age >= 18
AND age >= 21
AND age < 100
```

Análisis:

```text
age range:

[21, 100)
```

Resultado lógico:

```text
WHERE
    age >= 21
AND age < 100
```

---

# 211. Proof

```text
age >= 21
⇒
age >= 18
```

Por tanto:

```text
age >= 18
```

es redundante.

---

# 212. Ejemplo — NULL evita simplificación clásica

Entrada:

```text
WHERE
    status = 'active'
OR NOT(status = 'active')
```

No transformar automáticamente en:

```text
TRUE
```

porque:

```text
status = NULL
```

produce:

```text
UNKNOWN OR UNKNOWN
=
UNKNOWN
```

---

# 213. Ejemplo — contradiction

```text
WHERE
    id = 10
AND id = 20
```

Si:

```text
id
```

es un tipo escalar compatible y los valores son semánticamente distintos:

```text
predicate cannot be TRUE
```

En `WHERE` puede reducirse a un filtro imposible.

---

# 214. Context caveat

La representación final podría ser:

```text
FALSE
```

o un:

```text
UnsatisfiableFilter
```

dependiendo de la arquitectura lógica posterior.

---

# 215. Ejemplo — pushdown

Entrada:

```text
Filter(
    o.status = 'paid',

    DerivedTable(
        Project(
            orders.id,
            orders.status,
            orders.total
        )
    )
)
```

Resultado:

```text
DerivedTable(
    Project(
        Filter(
            orders.status = 'paid',
            orders
        )
    )
)
```

---

# 216. Preconditions

```text
status lineage resolved
predicate only depends on orders
no window
no aggregate barrier
no LIMIT
no OFFSET
no volatile expression
no security barrier
```

---

# 217. Ejemplo — pushdown bloqueado por LIMIT

Entrada:

```text
Filter(
    status = 'paid',

    Limit(
        Orders,
        10
    )
)
```

No transformar en:

```text
Limit(
    Filter(
        status = 'paid',
        Orders
    ),
    10
)
```

---

# 218. Razón

La primera consulta significa:

```text
take 10
then filter
```

La segunda:

```text
filter
then take 10
```

Cardinalidad y filas observables pueden cambiar.

---

# 219. Ejemplo — HAVING

```sql
SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id
HAVING customer_id = 10
```

Puede existir transformación hacia:

```sql
WHERE customer_id = 10
GROUP BY customer_id
```

si la semántica de grouping lo permite.

---

# 220. Aggregate-dependent HAVING

Esto:

```sql
HAVING COUNT(*) > 10
```

permanece posterior a aggregation.

---

# 221. Ejemplo — window barrier

```text
SELECT *
FROM (
    SELECT
        id,
        ROW_NUMBER() OVER (
            ORDER BY created_at
        ) AS rn
    FROM orders
) q
WHERE rn <= 10
```

El filtro:

```text
rn <= 10
```

no puede moverse debajo de la Window Stage.

---

# 222. Ejemplo — NOT IN

```text
user_id NOT IN (
    SELECT banned_user_id
    FROM bans
)
```

No convertir automáticamente a:

```text
NOT EXISTS(...)
```

si:

```text
banned_user_id
```

puede ser `NULL`.

---

# 223. Ejemplo — security predicate

Entrada:

```text
tenant_id = :tenant
AND status = 'active'
```

Si:

```text
tenant_id = :tenant
```

proviene de una tenant policy:

```text
PredicateProvenance:
    TENANT_POLICY
```

deberá preservarse durante toda optimización.

---

# 224. Fórmula del sistema

```text
Predicate Optimization
=
3VL-Aware Simplification
+
Constraint Reasoning
+
Implication
+
Range Reasoning
+
Null-Rejection
+
Safe Predicate Movement
+
Barrier Preservation
+
Bounded Search
```

---

# 225. Fórmula de pushdown

```text
SafePushdown(P, O)
=
ReferencesResolvableBelow(P,O)
∧ SemanticPhaseAllows(P,O)
∧ CardinalitySemanticsPreserved(P,O)
∧ NullSemanticsPreserved(P,O)
∧ VolatilityAllows(P,O)
∧ SecurityAllows(P,O)
∧ CorrelationAllows(P,O)
∧ CapabilitiesAllow(P,O)
```

---

# 226. Fórmula de redundancia

```text
Redundant(Q | P)
iff

P ⇒ Q
```

sujeto al contexto SQL correspondiente.

---

# 227. Fórmula de null-rejection

```text
NullRejecting(P, S)
iff

TRUE ∉ Evaluate(
    P,
    S := NULL
)
```

---

# 228. Fórmula de rango

```text
Range(P1 AND P2)
=
Intersection(
    Range(P1),
    Range(P2)
)
```

cuando ambos predicates son representables bajo el mismo dominio de orden.

---

# 229. Fórmula de seguridad

```text
SafePredicateOptimization
=
SemanticEquivalence
+
3VLPreservation
+
NullabilityPreservation
+
CardinalityPreservation
+
ScopePreservation
+
SecurityPreservation
+
VolatilityPreservation
```

---

# 230. Invariantes arquitectónicos

## DB-PREDOPT-001

El Predicate Optimizer preservará SQL 3VL.

## DB-PREDOPT-002

SQL predicates no serán tratados como PHP booleans.

## DB-PREDOPT-003

TRUE, FALSE y UNKNOWN serán estados semánticos distintos.

## DB-PREDOPT-004

Filter semantics y truth semantics serán distinguibles.

## DB-PREDOPT-005

Las reglas podrán ser context-sensitive.

## DB-PREDOPT-006

WHERE y JOIN ON no serán intercambiables automáticamente.

## DB-PREDOPT-007

HAVING y WHERE no serán intercambiables automáticamente.

## DB-PREDOPT-008

Predicate optimization ocurrirá después de semantic analysis.

## DB-PREDOPT-009

Predicate optimizer no generará SQL.

## DB-PREDOPT-010

Predicate optimizer no abrirá conexiones.

## DB-PREDOPT-011

Predicate optimizer no elegirá índices.

## DB-PREDOPT-012

Predicate optimizer no elegirá physical scans.

## DB-PREDOPT-013

ConstraintGraph será la fuente compartida de hechos.

## DB-PREDOPT-014

No existirá un segundo constraint engine incompatible.

## DB-PREDOPT-015

Structural equality no implicará semantic equivalence.

## DB-PREDOPT-016

Semantic equivalence no dependerá de SQL strings.

## DB-PREDOPT-017

Runtime bindings no serán inspeccionados por defecto.

## DB-PREDOPT-018

Parameter specialization será una capability separada.

## DB-PREDOPT-019

Constant simplification respetará UNKNOWN.

## DB-PREDOPT-020

`P OR NOT P → TRUE` no será regla general.

## DB-PREDOPT-021

`P AND NOT P → FALSE` no será regla general.

## DB-PREDOPT-022

Predicate deduplication respetará volatility.

## DB-PREDOPT-023

Predicate factorization respetará volatility.

## DB-PREDOPT-024

Predicate reordering no prometerá physical evaluation order.

## DB-PREDOPT-025

SQL short-circuit no será supuesto.

## DB-PREDOPT-026

Side-effect-sensitive expressions serán barriers.

## DB-PREDOPT-027

Implication será typed.

## DB-PREDOPT-028

Implication será null-aware.

## DB-PREDOPT-029

Implication será bounded.

## DB-PREDOPT-030

Subsumption requerirá proof.

## DB-PREDOPT-031

Contradiction detection distinguirá FALSE de UNKNOWN.

## DB-PREDOPT-032

Tautology detection distinguirá TRUE de UNKNOWN.

## DB-PREDOPT-033

IS NULL será semánticamente distinto de equality-to-null.

## DB-PREDOPT-034

Null-rejection será explícita.

## DB-PREDOPT-035

Null-rejection será relation/symbol aware.

## DB-PREDOPT-036

Outer join optimization consumirá null-rejection facts.

## DB-PREDOPT-037

Equality propagation respetará domain identity.

## DB-PREDOPT-038

Equality propagation respetará nullability.

## DB-PREDOPT-039

Equality classes no asumirán mathematical equality universal.

## DB-PREDOPT-040

Range analysis será type-aware.

## DB-PREDOPT-041

Range analysis será collation/operator aware cuando corresponda.

## DB-PREDOPT-042

Range intersections imposibles serán detectables.

## DB-PREDOPT-043

BETWEEN podrá permanecer estructurado.

## DB-PREDOPT-044

NOT BETWEEN preservará 3VL.

## DB-PREDOPT-045

IN deduplication preservará NULL semantics.

## DB-PREDOPT-046

NOT IN nunca se convertirá ingenuamente en NOT EXISTS.

## DB-PREDOPT-047

Empty IN será manejado estructuralmente.

## DB-PREDOPT-048

IN physical strategy no pertenecerá al Predicate Optimizer.

## DB-PREDOPT-049

LIKE optimization respetará collation.

## DB-PREDOPT-050

LIKE optimization no seleccionará índices.

## DB-PREDOPT-051

Regex será capability-driven.

## DB-PREDOPT-052

Pushdown requerirá proof.

## DB-PREDOPT-053

ReferencedRelations por sí solo no demostrará pushdown safety.

## DB-PREDOPT-054

Projection pushdown usará lineage.

## DB-PREDOPT-055

Derived-table pushdown respetará semantic barriers.

## DB-PREDOPT-056

LIMIT será una barrier fuerte para predicate movement.

## DB-PREDOPT-057

OFFSET será una barrier fuerte para predicate movement.

## DB-PREDOPT-058

Window stages serán barriers para predicates dependientes de resultados window.

## DB-PREDOPT-059

Aggregate stages serán barriers salvo reglas demostradas.

## DB-PREDOPT-060

GROUPING SETS recibirán tratamiento especializado.

## DB-PREDOPT-061

OUTER JOIN pushdown será null-extension aware.

## DB-PREDOPT-062

ON predicates no se moverán ciegamente a WHERE.

## DB-PREDOPT-063

WHERE predicates no se moverán ciegamente a ON.

## DB-PREDOPT-064

Set-operation pushdown respetará multiplicity.

## DB-PREDOPT-065

Set-operation pushdown respetará output coercions.

## DB-PREDOPT-066

CTE pushdown respetará materialization intent.

## DB-PREDOPT-067

Recursive CTE pushdown no será una regla genérica.

## DB-PREDOPT-068

HAVING movement distinguirá grouping-key predicates.

## DB-PREDOPT-069

Aggregate-dependent HAVING permanecerá post-aggregation salvo transformación formal.

## DB-PREDOPT-070

QUALIFY será una placement distinta si se soporta.

## DB-PREDOPT-071

Pull-up requerirá proof.

## DB-PREDOPT-072

Factorization será bounded.

## DB-PREDOPT-073

VoltStack no requerirá CNF global.

## DB-PREDOPT-074

VoltStack no requerirá DNF global.

## DB-PREDOPT-075

Predicate expansion estará limitada por budget.

## DB-PREDOPT-076

Logical predicate order y physical evaluation order serán distintos.

## DB-PREDOPT-077

Selectivity hints no serán constraints.

## DB-PREDOPT-078

Logical cardinality facts no serán heuristics.

## DB-PREDOPT-079

ImplicationGraph será bounded.

## DB-PREDOPT-080

Equality facts podrán almacenarse compactamente.

## DB-PREDOPT-081

Derived predicates estarán limitados.

## DB-PREDOPT-082

Predicate dependencies serán explícitas.

## DB-PREDOPT-083

Dependency y lineage serán distintos.

## DB-PREDOPT-084

Correlation boundaries serán respetadas.

## DB-PREDOPT-085

EXISTS simplification requerirá cardinality proof.

## DB-PREDOPT-086

Security predicate provenance será preservada.

## DB-PREDOPT-087

Tenant predicate provenance será preservada.

## DB-PREDOPT-088

Policy evidence no se perderá por deduplication.

## DB-PREDOPT-089

Security barriers restringirán predicate movement.

## DB-PREDOPT-090

Raw predicates serán barriers por defecto.

## DB-PREDOPT-091

Raw SQL no será reinterpretado ad hoc.

## DB-PREDOPT-092

Extension predicates requerirán semantic descriptors para optimización avanzada.

## DB-PREDOPT-093

Unknown extension semantics producirán conservación, no guessing.

## DB-PREDOPT-094

Vendor checks directos estarán prohibidos en reglas centrales.

## DB-PREDOPT-095

Capabilities describirán diferencias de plataforma.

## DB-PREDOPT-096

Predicate rules usarán Optimization Rule System.

## DB-PREDOPT-097

Predicate rules serán versionadas.

## DB-PREDOPT-098

Predicate rules serán explicables.

## DB-PREDOPT-099

Predicate rules serán deterministas bajo mismos inputs.

## DB-PREDOPT-100

Predicate fixpoints estarán acotados.

## DB-PREDOPT-101

Predicate facts serán operation-scoped.

## DB-PREDOPT-102

Equality classes serán operation-scoped.

## DB-PREDOPT-103

Implication graphs serán operation-scoped.

## DB-PREDOPT-104

Range state será operation-scoped.

## DB-PREDOPT-105

No habrá current predicate global.

## DB-PREDOPT-106

No habrá current scope global.

## DB-PREDOPT-107

No habrá cross-request predicate facts.

## DB-PREDOPT-108

Frozen rule descriptors podrán compartirse.

## DB-PREDOPT-109

FrankenPHP persistent workers serán seguros.

## DB-PREDOPT-110

RoadRunner será soportable sin rediseño.

## DB-PREDOPT-111

OpenSwoole será soportable sin rediseño.

## DB-PREDOPT-112

Predicate optimization budgets serán explícitos.

## DB-PREDOPT-113

Budget exhaustion conservará query válida.

## DB-PREDOPT-114

Optimization nunca dependerá de completar todas las oportunidades.

## DB-PREDOPT-115

Diagnostics no expondrán sensitive bindings por defecto.

## DB-PREDOPT-116

Telemetry no cambiará semantic correctness.

## DB-PREDOPT-117

Property tests cubrirán 3VL.

## DB-PREDOPT-118

Metamorphic tests cubrirán semantic equivalence.

## DB-PREDOPT-119

NULL datasets serán obligatorios en pruebas de rewrites.

## DB-PREDOPT-120

Outer join tests serán obligatorios para predicate movement.

## DB-PREDOPT-121

Window barrier tests serán obligatorios.

## DB-PREDOPT-122

Aggregate barrier tests serán obligatorios.

## DB-PREDOPT-123

Set-operation tests serán obligatorios.

## DB-PREDOPT-124

Security barrier tests serán obligatorios.

## DB-PREDOPT-125

Volatility tests serán obligatorios.

## DB-PREDOPT-126

NOT IN + NULL será un caso de conformidad obligatorio.

## DB-PREDOPT-127

Optimizer no inventará facts ante UNKNOWN.

## DB-PREDOPT-128

Optimizer preferirá conservar estructura antes que aplicar rewrite dudosa.

## DB-PREDOPT-129

Toda transformación producirá ChangeSet cuando cambie estructura.

## DB-PREDOPT-130

Correctness tendrá prioridad absoluta sobre selectivity/performance.

---

# 231. Anti-patterns

## 231.1 Tratar predicates como booleanos PHP

```php
if ($predicate) {
    ...
}
```

**Rechazado.**

---

## 231.2 Ley del tercero excluido

```text
P OR NOT P
→ TRUE
```

**Rechazado como regla general SQL.**

---

## 231.3 Contradicción clásica ciega

```text
P AND NOT P
→ FALSE
```

**Rechazado como regla general SQL.**

---

## 231.4 NOT IN → NOT EXISTS automático

**Rechazado.**

---

## 231.5 ON → WHERE automático

**Rechazado.**

---

## 231.6 WHERE → ON automático

**Rechazado.**

---

## 231.7 Pushdown a través de LIMIT

sin proof.

**Rechazado.**

---

## 231.8 Pushdown a través de Window

sin proof.

**Rechazado.**

---

## 231.9 CNF/DNF global

**Rechazado.**

---

## 231.10 Inspeccionar runtime bindings

para optimización general.

**Rechazado.**

---

## 231.11 Parsear Raw SQL dentro del optimizer

**Rechazado.**

---

## 231.12 Eliminar security predicates perdiendo provenance

**Rechazado.**

---

## 231.13 Elegir índices desde Predicate Optimizer

**Rechazado.**

---

## 231.14 Elegir physical scans

**Rechazado.**

---

## 231.15 Mantener equality classes globales

**Rechazado.**

---

# 232. Relación con Query Rewrite System

```text
Predicate Optimizer
       │
       ▼
Optimization Rule
       │
       ▼
Proof
       │
       ▼
Query Rewrite System
       │
       ▼
Transformed Query
```

El Predicate Optimizer decide:

```text
whether
where
why
```

La infraestructura de rewrite ejecuta:

```text
safe structural transformation
```

---

# 233. Relación con Optimization Rule System

Cada optimización será una regla formal:

```text
PredicateOptimizationRule
        │
        ├── descriptor
        ├── applicability
        ├── preconditions
        ├── capabilities
        ├── barriers
        ├── effects
        ├── budget
        └── diagnostics
```

---

# 234. Relación con Join Optimizer

La relación será especialmente estrecha:

```text
Predicate Optimizer
       │
       ├── Null-Rejection Facts
       ├── Predicate Placement
       ├── Equality Facts
       └── Range Facts
       │
       ▼
Join Optimizer
```

Y:

```text
Join Optimizer
       │
       ├── Join Strengthening
       ├── Join Elimination
       └── Join Reordering
       │
       ▼
new predicate opportunities
       │
       ▼
Predicate Optimizer
```

---

# 235. Coordinación mediante Scheduler

No habrá:

```text
PredicateOptimizer
→ directly calls JoinOptimizer
→ directly calls PredicateOptimizer
```

Se utilizará:

```text
OptimizationRuleScheduler
+
OptimizationChangeSet
```

para evitar dependencias circulares.

---

# 236. Arquitectura resultante

```text
                 SemanticQueryArtifact
                         │
                         ▼
                  ConstraintGraph
                         │
                         ▼
              Predicate Optimization
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
     Simplification   Reasoning      Movement
          │              │              │
          │       ┌──────┴──────┐       │
          │       ▼             ▼       │
          │   Equality        Range      │
          │       │             │       │
          │       └──────┬──────┘       │
          │              ▼              │
          │        Null-Rejection       │
          │              │              │
          └──────────────┼──────────────┘
                         ▼
               OptimizationChangeSet
                         │
                         ▼
               Optimization Scheduler
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
      Predicate Rules           Join Rules
```

---

# 237. Decisión arquitectónica final

VoltStack implementará Predicate Optimization como un sistema:

```text
3VL-aware
typed
scope-aware
constraint-driven
null-aware
lineage-aware
barrier-aware
security-aware
capability-driven
rule-based
bounded
explainable
persistent-runtime safe
```

La regla fundamental será:

> **VoltStack nunca optimizará un predicado aplicando álgebra booleana clásica de manera ciega. Toda transformación deberá respetar SQL Three-Valued Logic, nullability, scope, cardinality, volatility, correlation, security provenance y las barreras semánticas del operador donde el predicado se encuentre.**

---

# 238. Resultado arquitectónico

Con este sistema:

```text
PredicateNode
      │
      ▼
Semantic Meaning
      │
      ▼
Constraint Facts
      │
      ▼
3VL Reasoning
      │
      ├── implication
      ├── ranges
      ├── equality
      └── null-rejection
      │
      ▼
Safe Predicate Transformations
      │
      ▼
Better Logical Query
```

sin introducir dependencias hacia:

```text
SQL Compiler
PDO
ORM
physical planner
specific database vendor
```

---

# 239. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md          ← actual
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 240. Siguiente documento

```text
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md
```

El siguiente documento deberá definir la capa que utiliza directamente buena parte de los hechos producidos aquí:

```text
Join Optimization Architecture

Logical Join Semantics
Join Graph Optimization

Join Classification
Inner Join Optimization
Outer Join Optimization
Cross Join Optimization

Join Predicate Analysis
Equi-Join Detection
Non-Equi Join Detection

Join Reordering
Join Associativity
Join Commutativity

Join Search Space
Join Enumeration

Join Elimination
Redundant Join Detection

Outer Join Strengthening
LEFT → INNER
RIGHT → INNER
FULL reductions

Semi Join Transformation
Anti Join Transformation

EXISTS Decorrelating
NOT EXISTS Decorrelating

Join Predicate Pushdown

Join Constraint Propagation

Foreign-Key-Aware Optimization
Unique-Key-Aware Optimization

Cardinality Preservation Proof

Functional Dependency Integration

Join Connectivity Graph
Join Dependency Graph

Lateral / Correlated Join Constraints

Security Barriers
Tenant Boundaries

Volatile Predicate Constraints

Join Ordering Hints
Cost Hints

Greedy Join Ordering
Dynamic Programming Search
Bounded Search

Join Alternative Generation
Join Alternative Deduplication

Optimizer Rule Integration
ConstraintGraph Integration
SemanticGraph Integration

Budgets
Diagnostics
Telemetry
Testing

Persistent Runtime Safety
```

La separación deberá mantenerse estrictamente:

```text
Join Optimizer
=
logical join optimization

Query Planner
=
physical join planning

Physical Planner
=
Hash Join / Merge Join / Nested Loop / etc.

SQL Compiler
=
target SQL syntax
```