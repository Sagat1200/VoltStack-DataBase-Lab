# 59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md

# VoltStack Quantum Database
## Join Optimization System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 59 — Join Optimization System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Optimizer / Join  
**Versión:** 1.0

---

# 1. Propósito

`Join Optimization System` define la arquitectura responsable de transformar estructuras lógicas de joins en formas semánticamente equivalentes que reduzcan el espacio de ejecución, eliminen relaciones innecesarias, expongan mejores órdenes lógicos y proporcionen alternativas al futuro Query Planner.

Opera después de:

```text
Normalization
Validation
Semantic Analysis
Symbol Resolution
Schema-Aware Resolution
Type Inference
Relation & Join Resolution
Constraint Analysis
Semantic Graph Construction
```

y utiliza directamente la infraestructura definida por:

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
```

Flujo conceptual:

```text
SemanticQueryArtifact
        │
        ▼
RelationGraph
ConstraintGraph
Predicate Facts
        │
        ▼
Join Optimization
        │
        ├── Join Classification
        ├── Join Graph Analysis
        ├── Predicate Analysis
        ├── Join Strengthening
        ├── Join Elimination
        ├── Join Reordering
        ├── Semi/Anti Transformation
        ├── Decorrelation
        ├── Constraint Propagation
        └── Alternative Generation
        │
        ▼
Optimized Logical Join Structure
        │
        ▼
Query Planner
```

---

# 2. Regla maestra

Una transformación de join sólo será válida cuando preserve la semántica observable de la consulta.

Formalmente:

```text
OptimizeJoin(J, C) = J'

iff

Semantics(J, C)
=
Semantics(J', C)
```

La equivalencia deberá considerar como mínimo:

```text
row cardinality
row multiplicity
NULL extension
predicate truth semantics
output symbols
column lineage
ordering guarantees
correlation
security boundaries
locking semantics
volatility
side effects
```

---

# 3. Join Optimizer ≠ Join Builder

El `JoinQueryBuilder` declara:

```text
what relations are joined
logical join type
join condition
correlation/lateral intent
```

El `Join Optimizer` decide si esa estructura lógica puede transformarse.

Nunca:

```text
JoinBuilder
→ reorder joins
```

ni:

```text
JoinBuilder
→ eliminate joins
```

---

# 4. Join Optimizer ≠ Relation Resolver

`Relation & Join Resolution System` determina:

```text
what relation a reference means
what symbols are visible
what join instances exist
what correlation exists
```

El optimizador consume esos resultados.

No vuelve a resolver nombres.

---

# 5. Join Optimizer ≠ Predicate Optimizer

El Predicate Optimizer determina hechos como:

```text
predicate implication
null rejection
ranges
equality classes
predicate placement
pushdown eligibility
```

El Join Optimizer consume esos hechos para transformar joins.

Ejemplo:

```text
Predicate Optimizer
    │
    ▼
customers.active = TRUE
is null-rejecting on customers
    │
    ▼
Join Optimizer
    │
    ▼
LEFT JOIN customers
→ INNER JOIN customers
```

---

# 6. Join Optimizer ≠ Query Planner

El Join Optimizer trabaja sobre:

```text
logical joins
```

El Query Planner trabaja sobre:

```text
physical execution alternatives
```

Por tanto:

```text
Join Optimizer
```

no seleccionará:

```text
Hash Join
Merge Join
Nested Loop Join
Index Nested Loop
Parallel Hash Join
```

---

# 7. Logical Join ≠ Physical Join

Ejemplo:

```text
Logical:

Orders INNER JOIN Customers
```

puede ejecutarse físicamente mediante:

```text
HashJoin
MergeJoin
NestedLoop
```

La decisión pertenece al Planner.

---

# 8. Join Optimizer ≠ SQL Compiler

El Join Optimizer nunca producirá:

```sql
INNER JOIN customers
ON ...
```

El resultado seguirá siendo representación lógica estructurada.

---

# 9. Posición arquitectónica

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
SemanticQueryGraph
     │
     ▼
Constraint Analysis
     │
     ▼
Query Optimizer
     │
     ├── Predicate Optimizer
     │
     ├── Join Optimizer
     │
     ├── Subquery Optimizer
     │
     ├── Aggregate Optimizer
     │
     └── Set Optimizer
     │
     ▼
Optimized Logical Query
     │
     ▼
Query Planner
```

---

# 10. Entradas

El sistema podrá consumir:

```text
SemanticQueryArtifact
SemanticQueryGraph
RelationGraph
ConstraintGraph
PredicateOptimizationFacts
QueryTypeTable
SymbolTable
ScopeTable
DependencyGraph
CorrelationDescriptors
CapabilitySet
QueryMetadata
OptimizationContext
```

---

# 11. Salidas

Podrá producir:

```text
JoinOptimizationResult
```

con:

```text
OptimizationChangeSet
JoinOptimizationFacts
JoinAlternatives
JoinOptimizationProofs
JoinDiagnostics
```

---

# 12. JoinOptimizationFacts

Modelo conceptual:

```php
final readonly class JoinOptimizationFacts
{
    public function __construct(
        public JoinId $joinId,
        public LogicalJoinType $type,
        public JoinConnectivity $connectivity,
        public CardinalityConstraint $cardinality,
        public array $equalityFacts,
        public array $nullRejectionFacts,
        public array $dependencyFacts,
        public array $keyFacts,
        public array $barriers,
    ) {}
}
```

---

# 13. Join types

V1 reconocerá al menos:

```php
enum LogicalJoinType
{
    case INNER;
    case LEFT;
    case RIGHT;
    case FULL;
    case CROSS;

    case SEMI;
    case ANTI;
}
```

Los dos últimos pueden ser formas lógicas internas generadas por optimización.

---

# 14. SEMI y ANTI

El Query Builder no necesita exponerlos necesariamente como API pública V1.

Pueden aparecer mediante transformaciones como:

```text
EXISTS
→ SEMI JOIN
```

y:

```text
NOT EXISTS
→ ANTI JOIN
```

cuando sea seguro.

---

# 15. Lateralidad

`LATERAL` no se modelará simplemente como otro valor de `LogicalJoinType`.

Será una propiedad/dependencia del join source:

```text
Join
+
LateralDependency
```

porque describe principalmente visibilidad y dependencia.

---

# 16. Join relation instance

Una tabla física no equivale a una única relación lógica.

Ejemplo:

```sql
employees e
JOIN employees manager
```

produce:

```text
RelationInstance(employee:e)
RelationInstance(employee:manager)
```

---

# 17. Join identity

Cada join tendrá:

```text
JoinId
```

La identidad no dependerá de SQL textual.

---

# 18. Join Graph

Las consultas con múltiples joins se modelarán mediante un grafo lógico.

```text
RelationNode
    │
    │ JoinEdge
    ▼
RelationNode
```

---

# 19. JoinGraph

Modelo conceptual:

```php
final readonly class JoinGraph
{
    public function __construct(
        public array $relations,
        public array $edges,
        public array $dependencies,
        public array $barriers,
    ) {}
}
```

---

# 20. JoinGraph ≠ AST

El AST conserva:

```text
declared query structure
```

El JoinGraph expone:

```text
optimization relationships
```

No constituye un segundo AST mutable.

---

# 21. Join edge

```php
final readonly class JoinEdge
{
    public function __construct(
        public JoinId $joinId,
        public RelationId $left,
        public RelationId $right,
        public LogicalJoinType $type,
        public ?PredicateId $predicate,
        public JoinDependencySet $dependencies,
    ) {}
}
```

---

# 22. Hypergraph future model

Algunos predicates pueden conectar más de dos relaciones.

Ejemplo:

```text
a.x + b.y = c.z
```

Por ello la arquitectura no deberá impedir una futura representación:

```text
Join Hypergraph
```

---

# 23. V1 representation

V1 podrá combinar:

```text
binary JoinGraph edges
+
multi-relation PredicateDependencySet
```

sin exigir hypergraph completo.

---

# 24. Join connectivity

Cada join podrá clasificarse como:

```text
CONNECTED
PARTIALLY_CONNECTED
CARTESIAN
CORRELATED
LATERAL_DEPENDENT
BARRIER_CONSTRAINED
```

---

# 25. Cartesian joins

Un:

```text
CROSS JOIN
```

puede ser intencional.

También puede existir un:

```text
INNER JOIN
```

sin condición efectiva que sea semánticamente cartesiano.

---

# 26. Cartesian detection

El sistema deberá distinguir:

```text
explicit Cartesian intent
```

de:

```text
accidental disconnected join graph
```

para diagnostics.

---

# 27. Join predicate classification

Los predicates asociados a joins podrán clasificarse como:

```text
EQUI_JOIN
NON_EQUI_JOIN
RANGE_JOIN
THETA_JOIN
CROSS
MIXED
CORRELATED
EXTENSION
```

---

# 28. Equi-join

Ejemplo:

```text
orders.customer_id = customers.id
```

---

# 29. Multi-column equi-join

Ejemplo:

```text
a.tenant_id = b.tenant_id
AND
a.id = b.a_id
```

produce un conjunto de claves de join.

---

# 30. EquiJoinKeySet

```php
final readonly class EquiJoinKeySet
{
    public function __construct(
        public array $leftExpressions,
        public array $rightExpressions,
        public EqualitySemantics $semantics,
    ) {}
}
```

---

# 31. Non-equi join

Ejemplo:

```text
a.created_at < b.expires_at
```

---

# 32. Range join

Ejemplo:

```text
event.timestamp >= period.start
AND
event.timestamp < period.end
```

Puede clasificarse especialmente para el Planner.

---

# 33. Classification ≠ physical selection

Detectar:

```text
EQUI_JOIN
```

no significa seleccionar:

```text
HashJoin
```

Sólo expone una propiedad lógica.

---

# 34. Join equality facts

Un equi-join puede producir:

```text
EqualityFacts
```

pero deberá respetarse:

```text
join type
null extension
scope
```

---

# 35. INNER JOIN equality

En un inner join:

```text
A.x = B.x
```

puede generar hechos de igualdad sobre las filas resultantes.

---

# 36. OUTER JOIN equality

En un outer join, esos hechos no pueden propagarse de la misma forma hacia las filas null-extended.

---

# 37. Join constraints

El sistema consumirá:

```text
PrimaryKeyConstraint
UniqueConstraint
ForeignKeyConstraint
NotNullConstraint
FunctionalDependency
CardinalityConstraint
EqualityConstraint
```

---

# 38. FK ≠ automatic join

La existencia de:

```text
orders.customer_id
→ customers.id
```

no significa que VoltStack agregará un join automáticamente.

---

# 39. FK as evidence

Un FK puede ayudar a demostrar:

```text
referential existence
join cardinality
join elimination
uniqueness
```

---

# 40. Foreign key trust

No todos los schema facts deberán considerarse necesariamente confiables.

Podrá existir:

```text
ConstraintTrustLevel
```

---

# 41. ConstraintTrustLevel

Propuesta:

```php
enum ConstraintTrustLevel
{
    case ENFORCED;
    case DECLARED;
    case ASSUMED;
    case UNKNOWN;
}
```

---

# 42. Optimization proof

Transformaciones destructivas como join elimination deberán exigir el nivel de confianza apropiado.

---

# 43. Join cardinality

El sistema manejará constraints lógicos como:

```text
ONE_TO_ONE
ONE_TO_ZERO_OR_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
UNKNOWN
```

---

# 44. Cardinality relation

También podrá expresar:

```text
output <= left × right
output <= left
output = left
output >= left
```

cuando pueda demostrarse.

---

# 45. Cardinality fact ≠ estimate

Esto:

```text
output <= left
```

puede ser un hecho lógico.

Esto:

```text
estimated rows = 12,540
```

es estimación del Planner/cost system.

---

# 46. Join elimination

Objetivo:

eliminar relaciones cuyo join no afecta el resultado observable.

Ejemplo conceptual:

```text
Orders
JOIN Customers
```

si:

```text
no Customer columns used
join cannot filter Orders
join cannot duplicate Orders
join has no observable side effects
```

podría eliminarse.

---

# 47. Elimination conditions

Una eliminación deberá demostrar:

```text
output symbols unaffected
cardinality unaffected
multiplicity unaffected
filtering unaffected
NULL semantics unaffected
security unaffected
locking unaffected
volatility unaffected
```

---

# 48. Many-to-one elimination

Ejemplo:

```sql
SELECT orders.id
FROM orders
JOIN customers
    ON customers.id = orders.customer_id
```

Si:

```text
orders.customer_id NOT NULL
FK orders.customer_id → customers.id
customers.id UNIQUE
FK trusted/enforced
no customer expression consumed
```

el join puede ser eliminable.

---

# 49. Nullable FK

Si:

```text
orders.customer_id
```

puede ser `NULL`, un inner join puede filtrar filas.

Por tanto la eliminación ya no es automáticamente válida.

---

# 50. Missing referential guarantee

Si no existe FK confiable, el join puede filtrar orders sin customer.

No eliminar.

---

# 51. Duplicate amplification

Si el lado derecho no es unique sobre la join key:

```text
one order
→ multiple customer matches
```

el join puede multiplicar filas.

No eliminar.

---

# 52. Join elimination proof

Modelo:

```php
final readonly class JoinEliminationProof
{
    public function __construct(
        public JoinId $joinId,
        public bool $preservesCardinality,
        public bool $preservesMultiplicity,
        public bool $preservesOutputs,
        public bool $preservesFiltering,
        public bool $preservesSecurity,
        public array $supportingConstraints,
    ) {}
}
```

---

# 53. LEFT JOIN elimination

Ejemplo:

```sql
SELECT orders.id
FROM orders
LEFT JOIN customers
    ON customers.id = orders.customer_id
```

Si:

```text
customers.id UNIQUE
no customer output used
join predicate has no side effects
```

puede ser eliminable incluso sin FK existence guarantee, porque LEFT JOIN preserva las filas de orders.

Pero deberá garantizarse que el lado derecho no duplica filas.

---

# 54. LEFT JOIN duplicate risk

Si múltiples `customers` pudieran coincidir:

```text
LEFT JOIN
```

sí podría multiplicar filas.

Por tanto:

```text
uniqueness proof
```

es esencial.

---

# 55. FULL JOIN elimination

Mucho más restrictivo.

Normalmente no se eliminará sin proof completo sobre ambos lados.

---

# 56. Join strengthening

Transformación:

```text
weaker outer join
→ stronger join
```

cuando filtros posteriores eliminan las filas null-extended.

---

# 57. LEFT → INNER

Ejemplo:

```sql
FROM orders
LEFT JOIN customers
    ON customers.id = orders.customer_id
WHERE customers.active = TRUE
```

Si el predicate es null-rejecting sobre `customers`:

```text
LEFT JOIN
→ INNER JOIN
```

puede ser válido.

---

# 58. RIGHT → INNER

Simétricamente:

```text
RIGHT JOIN
→ INNER JOIN
```

cuando predicates posteriores rechazan el lado null-extended correspondiente.

---

# 59. FULL reductions

Un `FULL JOIN` puede potencialmente reducirse a:

```text
LEFT
RIGHT
INNER
```

dependiendo de null-rejection.

---

# 60. FULL → LEFT

Si las filas que existen sólo en el lado derecho son rechazadas:

```text
FULL
→ LEFT
```

puede ser posible.

---

# 61. FULL → RIGHT

Si las filas sólo del lado izquierdo son rechazadas:

```text
FULL
→ RIGHT
```

---

# 62. FULL → INNER

Si ambos conjuntos de filas null-extended son rechazados:

```text
FULL
→ INNER
```

---

# 63. Strengthening proof

Debe considerar:

```text
post-join predicates
predicate placement
3VL
null-rejection
security predicates
volatile predicates
```

---

# 64. Strengthening ≠ predicate movement

Primero puede detectarse:

```text
null rejection
```

y luego cambiar el join.

No es necesario mover el predicate.

---

# 65. Join commutativity

Inner join puede ser lógicamente conmutativo bajo condiciones apropiadas:

```text
A ⋈ B
≡
B ⋈ A
```

---

# 66. Join associativity

También puede existir:

```text
(A ⋈ B) ⋈ C
≡
A ⋈ (B ⋈ C)
```

para joins compatibles.

---

# 67. Not universal

No asumir estas propiedades para:

```text
outer joins
lateral joins
correlated joins
security barriers
volatile expressions
locking-sensitive structures
```

---

# 68. JoinPropertySet

Cada join podrá exponer:

```php
final readonly class JoinPropertySet
{
    public function __construct(
        public bool $commutative,
        public bool $associative,
        public bool $reorderable,
        public bool $eliminable,
        public bool $strengthenable,
    ) {}
}
```

---

# 69. Join reorderability

La pregunta central no será:

```text
Can SQL syntax be reordered?
```

sino:

```text
Can this logical relation graph be transformed
without changing semantics?
```

---

# 70. Reorder groups

Los joins podrán agruparse en:

```text
JoinReorderGroup
```

que define regiones donde el orden lógico puede cambiar.

---

# 71. Example

```text
A INNER JOIN B
INNER JOIN C
INNER JOIN D
```

puede formar un reorder group si no existen barriers.

---

# 72. Outer join boundary

Ejemplo:

```text
(A LEFT JOIN B)
INNER JOIN C
```

no deberá aplanarse automáticamente como un único reorder group.

---

# 73. Join dependency graph

Además del JoinGraph se necesita:

```text
JoinDependencyGraph
```

para representar:

```text
lateral dependencies
correlation
outer join preservation
security ordering constraints
extension constraints
```

---

# 74. Join dependency

Ejemplo:

```text
B depends on A
```

significa que B no puede evaluarse lógicamente antes de que A esté disponible.

---

# 75. LATERAL

Ejemplo:

```sql
FROM users u
CROSS JOIN LATERAL (
    ...
    WHERE x.user_id = u.id
) x
```

produce:

```text
x depends on u
```

---

# 76. Lateral dependency

Por tanto:

```text
u → x
```

es una arista dirigida de dependencia.

---

# 77. Join order constraints

Un orden candidato deberá respetar:

```text
JoinDependencyGraph
```

---

# 78. Correlation

Las subqueries correlacionadas también pueden introducir dependencias.

---

# 79. Decorrelation

El optimizador podrá intentar eliminar algunas dependencias mediante transformaciones formales.

---

# 80. Decorrelation ≠ join reorder

Primero:

```text
correlated expression
→ equivalent uncorrelated logical form
```

Luego podrá participar en join optimization.

---

# 81. EXISTS decorrelation

Ejemplo conceptual:

```sql
WHERE EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.order_id = orders.id
)
```

puede transformarse lógicamente a:

```text
Orders
SEMI JOIN Payments
ON payments.order_id = orders.id
```

cuando sea seguro.

---

# 82. Why SEMI JOIN

Un inner join sería incorrecto porque podría multiplicar `orders`.

SEMI JOIN expresa:

```text
keep left row if at least one right match exists
```

sin duplicar la fila izquierda.

---

# 83. SEMI JOIN semantics

Formalmente:

```text
A SEMI JOIN B
```

produce únicamente filas de `A` para las que existe al menos un match en `B`.

---

# 84. ANTI JOIN semantics

```text
A ANTI JOIN B
```

produce filas de `A` para las que no existe match.

---

# 85. NOT EXISTS decorrelation

Puede transformarse a:

```text
ANTI JOIN
```

cuando las condiciones de correlation y predicate semantics lo permitan.

---

# 86. NOT IN warning

`NOT IN` no deberá convertirse directamente en ANTI JOIN sin resolver correctamente su semántica de `NULL`.

---

# 87. Null-aware anti join

Una futura representación puede incluir:

```text
NULL_AWARE_ANTI
```

como propiedad lógica especializada cuando sea necesario.

No deberá confundirse con ANTI estándar.

---

# 88. EXISTS with volatile subquery

La decorrelation puede estar bloqueada si altera:

```text
volatile evaluation
observable side effects
```

---

# 89. Correlated aggregate

Ejemplo:

```text
WHERE (
    SELECT COUNT(*)
    FROM payments
    WHERE payments.order_id = orders.id
) > 0
```

podría admitir transformaciones, pero no será tratado como simple EXISTS sin proof.

---

# 90. Scalar correlated subquery

Puede requerir:

```text
single-row cardinality proof
```

antes de convertirse en join.

---

# 91. Decorrelation rules

Se mantendrán en coordinación con:

```text
Subquery Optimization System
```

No se implementarán dos motores de decorrelation.

---

# 92. Ownership model

```text
Subquery Optimizer
    │
    ├── identifies decorrelation candidate
    └── proves subquery transformation
            │
            ▼
Join Optimizer
    │
    └── integrates resulting logical join
```

---

# 93. Predicate pushdown

El Join Optimizer consume las decisiones del Predicate Optimizer.

Ejemplo:

```text
Filter(A.status = 1)
over
A INNER JOIN B
```

puede empujarse hacia `A`.

---

# 94. Join predicate extraction

Predicates globales también pueden exponer nuevas join edges.

Ejemplo:

```sql
FROM a, b
WHERE a.id = b.a_id
```

semánticamente puede reconocerse como:

```text
A INNER JOIN B
ON A.id = B.a_id
```

sin necesidad de que el Builder lo haya escrito con `join()`.

---

# 95. CROSS + WHERE

Una estructura:

```text
CROSS JOIN
+
cross-relation predicate
```

puede convertirse en:

```text
INNER JOIN
```

lógico.

---

# 96. Preconditions

La transformación deberá preservar:

```text
predicate placement
volatility
security boundaries
correlation
```

---

# 97. Join predicate derivation

De:

```text
A.x = B.x
AND
B.x = C.x
```

puede derivarse:

```text
A.x = C.x
```

como fact.

---

# 98. Why useful

Esto puede crear una nueva edge en el JoinGraph:

```text
A ─── B
│     │
└──── C
```

mejorando opciones de join ordering.

---

# 99. Derived join edge

Debe marcarse:

```text
OPTIMIZER_DERIVED
```

y conservar proof.

---

# 100. Derived edge ≠ added predicate requirement

La arista derivada puede ser utilizada como:

```text
optimization fact
```

sin necesariamente insertar un predicate redundante en el AST.

---

# 101. Functional dependencies

Ejemplo:

```text
customers.id
→ customers.name
customers.email
```

por PK.

Esto puede ayudar a:

```text
join elimination
cardinality reasoning
uniqueness propagation
```

---

# 102. Key propagation

En ciertos inner joins:

```text
A.foreign_key = B.primary_key
```

pueden propagarse facts sobre:

```text
uniqueness
functional dependency
```

al output.

---

# 103. Outer join key propagation

Debe considerar null-extension.

Un PK del lado nullable puede dejar de comportarse como:

```text
NOT NULL key
```

en la relación resultante.

---

# 104. Unique ≠ primary key

El optimizador distinguirá:

```text
UNIQUE
PRIMARY KEY
```

porque nullability y guarantees pueden diferir.

---

# 105. Composite keys

Todas las reglas deberán soportar:

```text
single-column keys
composite keys
```

---

# 106. Composite FK

Ejemplo:

```text
(a.tenant_id, a.customer_id)
→
(b.tenant_id, b.id)
```

deberá tratarse como una unidad lógica.

---

# 107. Partial uniqueness

Una unique constraint parcial/condicional no deberá utilizarse fuera de su predicate de validez.

---

# 108. Constraint applicability

Cada constraint podrá incluir:

```text
scope
condition
trust
```

---

# 109. Join elimination with DISTINCT

La existencia de `DISTINCT` no autoriza automáticamente eliminar joins que duplican filas.

---

# 110. Why

Aunque:

```text
DISTINCT
```

pueda eliminar duplicados posteriormente, el join también puede:

```text
filter rows
affect expressions
affect volatile evaluation
affect aggregates
```

---

# 111. DISTINCT-aware elimination

Podrán existir reglas avanzadas con proof específico, pero no será una simplificación general.

---

# 112. Aggregation

Un join previo a aggregation puede afectar:

```text
COUNT
SUM
AVG
MIN/MAX multiplicity
group existence
```

---

# 113. Join elimination before aggregate

Requerirá proof de que:

```text
group membership
multiplicity
aggregate inputs
```

no cambian.

---

# 114. COUNT(*) sensitivity

Especialmente:

```text
COUNT(*)
```

es sensible a duplicate amplification.

---

# 115. Window functions

Cambiar joins antes de una Window Stage puede modificar:

```text
partition contents
row numbers
ranks
frames
lag/lead
```

---

# 116. Window-aware join optimization

Una transformación de join sólo será válida si preserva la relación que entra a la Window Stage.

---

# 117. Set operations

Los joins dentro de cada operand podrán optimizarse independientemente cuando no existan dependencias externas.

---

# 118. CTEs

Los joins dentro de un CTE podrán optimizarse en su propio query scope.

---

# 119. Shared CTE

La optimización deberá considerar que un CTE puede tener múltiples consumidores.

---

# 120. Recursive CTE

Join reordering dentro del recursive member deberá respetar:

```text
recursive dependency
```

---

# 121. Join search space

Para `N` relaciones reorderables, el número de posibles árboles crece rápidamente.

No se enumerarán todas las alternativas sin límite.

---

# 122. Combinatorial explosion

Conceptualmente:

```text
JoinSearchSpace(N)
```

crece superlineal/exponencialmente según el modelo de árboles y permutaciones considerado.

---

# 123. Bounded search

VoltStack utilizará:

```text
JoinOptimizationBudget
```

---

# 124. JoinOptimizationBudget

```php
final readonly class JoinOptimizationBudget
{
    public function __construct(
        public int $maxRelations,
        public int $maxEdges,
        public int $maxAlternatives,
        public int $maxReorderGroupSize,
        public int $maxSearchStates,
        public int $maxDerivedEdges,
        public int $maxDecorrelationAttempts,
        public int $maxFixpointIterations,
    ) {}
}
```

---

# 125. Small join groups

Para grupos pequeños puede permitirse búsqueda más exhaustiva.

---

# 126. Large join groups

Para grupos grandes podrán utilizarse:

```text
heuristic ordering
greedy exploration
bounded dynamic programming
```

---

# 127. Important boundary

El Join Optimizer puede generar:

```text
logical join orders
```

pero no necesita asignar todavía un costo físico completo.

---

# 128. Query Cost Hint System

El documento:

```text
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
```

proporcionará hints que podrán utilizarse para priorizar alternativas.

---

# 129. Cost hint ≠ physical cost model

Ejemplo:

```text
small relation
highly selective predicate
unique lookup
```

pueden influir en prioridad lógica sin convertirse todavía en:

```text
CPU cost = 123.4
I/O cost = 88.1
```

---

# 130. JoinOrderingHint

Modelo:

```php
final readonly class JoinOrderingHint
{
    public function __construct(
        public RelationId $relation,
        public JoinPriority $priority,
        public JoinHintConfidence $confidence,
        public string $reasonCode,
    ) {}
}
```

---

# 131. Join hints

Los hints podrán provenir de:

```text
semantic cardinality facts
query metadata
extension metadata
cost hints
planner feedback
```

---

# 132. User hints

Una futura API podrá aceptar hints como:

```text
prefer join order
prefer leading relation
preserve join order
```

pero serán metadata estructurada, no SQL comments.

---

# 133. Hint ≠ command

Por defecto:

```text
PREFER
```

no significa:

```text
REQUIRE
```

---

# 134. Requirement

Si se necesita enforcement:

```text
JoinOrderingRequirement
```

deberá ser explícito.

---

# 135. JoinAlternative

```php
final readonly class JoinAlternative
{
    public function __construct(
        public JoinAlternativeId $id,
        public LogicalJoinTree $tree,
        public JoinPropertySet $properties,
        public array $requiredCapabilities,
        public array $proofs,
        public JoinAlternativeFingerprint $fingerprint,
    ) {}
}
```

---

# 136. LogicalJoinTree

Representará una posible estructura lógica.

Ejemplo:

```text
        Join
       /    \
      A     Join
           /   \
          B     C
```

---

# 137. Bushy trees

La arquitectura permitirá:

```text
      Join
     /    \
   Join   Join
   / \    / \
  A   B  C   D
```

No limitarse permanentemente a left-deep trees.

---

# 138. Left-deep trees

Pueden ser una estrategia preferida V1 para limitar búsqueda, pero no un invariante arquitectónico.

---

# 139. Right-deep trees

También deberán ser representables.

---

# 140. Alternative fingerprint

Evitará mantener alternativas lógicamente duplicadas.

---

# 141. Join alternative deduplication

Ejemplo:

dos secuencias de rewrites pueden producir el mismo:

```text
LogicalJoinTree
```

Sólo deberá mantenerse una alternativa equivalente cuando metadata/proofs no requieran diferenciarlas.

---

# 142. Query Deduplication System

El siguiente documento:

```text
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
```

formalizará esta infraestructura de deduplicación.

---

# 143. Search strategy

Propuesta inicial:

```text
1. Build reorder groups
2. Build connectivity graph
3. Apply mandatory dependencies
4. Eliminate impossible alternatives
5. Prioritize connected joins
6. Use logical cardinality facts
7. Use cost hints
8. Generate bounded alternatives
9. Deduplicate alternatives
10. Forward candidates to Planner
```

---

# 144. Connected-first heuristic

Evitar productos cartesianos intermedios cuando existen alternativas conectadas suele ser una heurística razonable.

Pero no será una regla semántica universal.

---

# 145. Cartesian necessity

Algunas consultas requieren productos cartesianos.

El optimizador deberá soportarlos.

---

# 146. Join enumeration

Podrán coexistir estrategias:

```text
GreedyJoinEnumerator
DynamicProgrammingJoinEnumerator
DependencyAwareJoinEnumerator
ExtensionJoinEnumerator
```

---

# 147. Enumeration ≠ planning

Los enumerators producen:

```text
logical join alternatives
```

no physical operators.

---

# 148. Dynamic programming

Para grupos pequeños puede utilizarse:

```text
best known logical candidates
per relation subset
```

---

# 149. DP state

Conceptualmente:

```text
State({A,B,C})
```

representa alternativas que producen ese conjunto de relaciones.

---

# 150. DP budget

El número de subsets crece exponencialmente.

Debe existir límite.

---

# 151. Greedy strategy

Para grupos grandes:

```text
start with candidate
repeatedly attach best connected relation
```

según:

```text
connectivity
constraints
selectivity hints
dependency legality
```

---

# 152. Deterministic tie breaking

Mismos inputs deberán producir mismo resultado.

No usar:

```text
hash iteration randomness
```

para elegir orden.

---

# 153. Stable identities

Los desempates podrán utilizar:

```text
RelationId canonical ordering
SemanticFingerprint
```

---

# 154. Join barriers

Un join podrá estar restringido por:

```text
OUTER_JOIN_BOUNDARY
LATERAL_DEPENDENCY
CORRELATION
SECURITY_BARRIER
VOLATILITY_BARRIER
LOCKING_BARRIER
EXTENSION_BARRIER
USER_REQUIREMENT
```

---

# 155. Barrier model

```php
final readonly class JoinBarrier
{
    public function __construct(
        public JoinBarrierType $type,
        public JoinId $joinId,
        public array $affectedRelations,
        public string $reasonCode,
    ) {}
}
```

---

# 156. Security barriers

Un join introducido o protegido por una política de seguridad no deberá moverse a través de una frontera si eso puede:

```text
expose rows
change policy evaluation
alter side-channel behavior
```

---

# 157. Tenant isolation

Predicates como:

```text
orders.tenant_id = :tenant
customers.tenant_id = :tenant
```

podrán ayudar a formar:

```text
tenant-aware equality facts
```

pero el core no dependerá directamente del paquete Multitenancy.

---

# 158. Tenant boundary metadata

La integración podrá aportar:

```text
SecurityDomainId
IsolationDomainId
```

---

# 159. Cross-domain joins

Podrán requerir:

```text
explicit capability
security approval
optimization barrier
```

---

# 160. Security ≠ performance hint

Una security barrier nunca deberá ignorarse porque otra alternativa parezca más barata.

---

# 161. Locking

Queries con:

```text
FOR UPDATE
FOR SHARE
```

pueden tener restricciones de transformación.

---

# 162. Lock target semantics

Si el DB/platform distingue qué relaciones son bloqueadas, join elimination/reordering deberá preservar esas semantics.

---

# 163. Locking capability

Se modelará mediante:

```text
QueryMetadata
+
PlatformCapabilities
```

---

# 164. Volatility

Funciones volátiles dentro de:

```text
join predicates
relation sources
lateral subqueries
```

pueden bloquear:

```text
reordering
elimination
duplication
decorrelation
```

---

# 165. Join source volatility

Ejemplo:

```text
table_function(random_source())
```

no puede tratarse como una tabla pura convencional.

---

# 166. Extension joins

Extensiones podrán definir nuevos logical join semantics.

---

# 167. Extension contract

Deberán registrar:

```text
commutativity
associativity
null-extension behavior
cardinality behavior
reorderability
elimination rules
planner capabilities
```

---

# 168. Unknown join extension

Si no existen semantic properties suficientes:

```text
optimization barrier
```

---

# 169. No guessing

VoltStack preferirá:

```text
less optimized but correct
```

sobre:

```text
aggressively rewritten but uncertain
```

---

# 170. Join capability model

Propuesta:

```text
QUERY.OPTIMIZER.JOIN.BASIC
QUERY.OPTIMIZER.JOIN.GRAPH
QUERY.OPTIMIZER.JOIN.REORDER
QUERY.OPTIMIZER.JOIN.ELIMINATION
QUERY.OPTIMIZER.JOIN.STRENGTHENING
QUERY.OPTIMIZER.JOIN.SEMI
QUERY.OPTIMIZER.JOIN.ANTI
QUERY.OPTIMIZER.JOIN.DECORRELATION
QUERY.OPTIMIZER.JOIN.FK_REASONING
QUERY.OPTIMIZER.JOIN.FD_REASONING
QUERY.OPTIMIZER.JOIN.DP_ENUMERATION
QUERY.OPTIMIZER.JOIN.GREEDY_ENUMERATION
QUERY.OPTIMIZER.JOIN.LATERAL
```

---

# 171. Platform capabilities

También pueden existir:

```text
QUERY.JOIN.RIGHT
QUERY.JOIN.FULL
QUERY.JOIN.LATERAL
QUERY.JOIN.LOCKING
```

---

# 172. Logical support vs native syntax

Una transformación lógica puede producir una forma que el target no soporte nativamente.

Ejemplo:

```text
SEMI JOIN
```

puede permanecer como operador lógico interno.

El Compiler/Planner decidirá su representación final.

---

# 173. SEMI JOIN compilation

Podría terminar representándose mediante:

```text
EXISTS
```

si el target no tiene sintaxis `SEMI JOIN`.

Esto no invalida que el optimizer utilice SEMI como forma lógica.

---

# 174. ANTI JOIN compilation

Igualmente podría compilarse mediante:

```text
NOT EXISTS
```

cuando sea semánticamente correcto.

---

# 175. Optimizer logical vocabulary

El vocabulario interno puede ser más rico que el SQL superficial de un motor concreto.

---

# 176. JoinOptimizationRule

```php
interface JoinOptimizationRule extends OptimizationRule
{
    public function supportedJoinKinds(): array;
}
```

---

# 177. Catálogo V1

```text
join.cross_to_inner
join.extract_equi_keys
join.derive_transitive_edge
join.strengthen.left_to_inner
join.strengthen.right_to_inner
join.strengthen.full_to_left
join.strengthen.full_to_right
join.strengthen.full_to_inner
join.eliminate.inner_many_to_one
join.eliminate.left_unique_unused
join.exists_to_semi
join.not_exists_to_anti
join.reorder.inner_group
join.propagate_constraints
join.propagate_keys
join.detect_cartesian
join.generate_alternatives
join.deduplicate_alternatives
```

---

# 178. Rule phases

Propuesta:

```text
J1 — Join Classification
J2 — Constraint Enrichment
J3 — Join Strengthening
J4 — Join Elimination
J5 — Decorrelation / Semi-Anti
J6 — Reorder Group Construction
J7 — Alternative Generation
J8 — Alternative Deduplication
J9 — Post-Join Simplification
```

---

# 179. J1 — Classification

Determina:

```text
join type
connectivity
equi keys
range predicates
dependencies
barriers
```

---

# 180. J2 — Constraint Enrichment

Deriva:

```text
equality classes
FK facts
unique facts
FDs
cardinality constraints
```

---

# 181. J3 — Strengthening

Ejecuta:

```text
LEFT → INNER
RIGHT → INNER
FULL reductions
```

cuando existe proof.

---

# 182. Why strengthening early

Un outer join convertido a inner join puede abrir:

```text
reordering
predicate pushdown
join elimination
```

---

# 183. J4 — Elimination

Elimina joins demostrablemente redundantes.

---

# 184. J5 — Decorrelation

Introduce:

```text
SEMI
ANTI
```

u otras formas lógicas.

---

# 185. J6 — Reorder groups

Construye regiones legales de búsqueda.

---

# 186. J7 — Alternatives

Genera estructuras lógicas candidatas.

---

# 187. J8 — Deduplication

Elimina candidatos equivalentes.

---

# 188. J9 — Simplification

Nuevas estructuras pueden permitir:

```text
predicate simplification
constraint simplification
projection pruning
```

---

# 189. Fixpoint coordination

Ejemplo:

```text
LEFT JOIN
    │
    ▼
null-rejecting predicate
    │
    ▼
INNER JOIN
    │
    ▼
new reorder group
    │
    ▼
new predicate pushdown
    │
    ▼
join elimination
```

Por ello se utilizará un fixpoint global acotado.

---

# 190. No optimizer recursion

No:

```text
JoinOptimizer
    calls PredicateOptimizer
        calls JoinOptimizer
```

---

# 191. Scheduler

Se utilizará:

```text
OptimizationRuleScheduler
```

con:

```text
ChangeSet
InvalidationSet
NewFacts
```

---

# 192. Fact invalidation

Cambiar:

```text
LEFT
→ INNER
```

puede invalidar:

```text
null-extension facts
output nullability facts
join barriers
```

y generar otros nuevos.

---

# 193. Incremental recomputation

El sistema podrá recalcular únicamente facts afectados.

No será obligatorio reconstruir todo el SemanticGraph desde cero.

---

# 194. Semantic graph immutability

El artifact original permanecerá inmutable.

Las transformaciones producirán:

```text
optimized logical representation
+
derived semantic facts
```

sin mutar el artifact de entrada.

---

# 195. Optimization provenance

Cada transformación deberá registrar:

```text
rule
input
output
proof
supporting facts
```

---

# 196. Example trace

```text
Rule:
    join.strengthen.left_to_inner

Join:
    J4

Reason:
    WHERE predicate P8 null-rejects relation R2

Transformation:
    LEFT → INNER

Supporting facts:
    NullRejectionFact(P8, R2)
```

---

# 197. Elimination trace

```text
Rule:
    join.eliminate.left_unique_unused

Join:
    J7

Removed relation:
    customers

Proof:
    right output unused
    customers.id unique
    join predicate deterministic
    left rows preserved
    right cannot duplicate left
```

---

# 198. Reorder trace

```text
Original:
    ((A JOIN B) JOIN C)

Alternative:
    (A JOIN (B JOIN C))

Reason:
    inner join reorder group
    no dependencies
    connected graph
```

---

# 199. Diagnostics

Propuesta:

```text
JoinOptimizationBudgetExceeded
CartesianJoinDetected
DisconnectedJoinGraph
JoinEliminationBlocked
JoinEliminationBlockedByMultiplicity
JoinEliminationBlockedByNullableForeignKey
JoinEliminationBlockedByUntrustedConstraint
JoinEliminationBlockedBySecurity
JoinStrengtheningBlocked
JoinReorderingBlocked
JoinReorderingBlockedByOuterJoin
JoinReorderingBlockedByLateralDependency
JoinReorderingBlockedByCorrelation
JoinReorderingBlockedByVolatility
JoinReorderingBlockedByLocking
JoinDecorrelationBlocked
JoinAlternativeBudgetExceeded
JoinExtensionBarrier
```

---

# 200. Warnings vs errors

La mayoría de fallos de optimización serán:

```text
diagnostic / trace
```

no query errors.

---

# 201. Example

Si no puede eliminarse un join:

```text
query remains valid
```

simplemente menos optimizada.

---

# 202. Budget exhaustion

Igualmente:

```text
budget exhausted
→ stop exploring
→ retain best valid alternatives found
```

No:

```text
query fails
```

---

# 203. Determinism

Mismos:

```text
query
schema
capabilities
rules
metadata
budgets
```

deberán producir la misma salida lógica.

---

# 204. Statistics

Cuando en el futuro existan estadísticas dinámicas, podrán influir en ranking de alternativas.

Pero deberán formar parte explícita del contexto/fingerprint correspondiente.

---

# 205. No hidden database I/O

El Join Optimizer no ejecutará:

```text
SHOW INDEX
EXPLAIN
SELECT COUNT(*)
```

durante optimization.

---

# 206. SchemaView

Toda información necesaria llegará mediante:

```text
SchemaView
ConstraintGraph
StatisticsSnapshot
```

cuando corresponda.

---

# 207. Persistent runtime safety

Compartible:

```text
JoinOptimizationRule descriptors
Join semantic descriptors
frozen registries
immutable capability metadata
```

Operation-scoped:

```text
JoinGraph
JoinDependencyGraph
reorder groups
search states
alternatives
proofs
budgets
diagnostics
```

---

# 208. No global join graph

Prohibido:

```php
JoinOptimizer::$graph
```

---

# 209. No global current relation

Prohibido:

```php
JoinOptimizer::$currentRelation
```

---

# 210. No global search state

Toda enumeración será operation-scoped.

---

# 211. Worker reset

Al finalizar una operación/request deberán descartarse:

```text
join search states
temporary alternatives
derived join edges
temporary cardinality hints
diagnostics buffers
```

---

# 212. FrankenPHP

La arquitectura será segura bajo workers persistentes de FrankenPHP.

---

# 213. RoadRunner / OpenSwoole

La misma separación de estado permitirá adaptarlos sin rediseñar el Join Optimizer.

---

# 214. Testing architecture

Se requerirán:

```text
unit tests
rule tests
constraint tests
property tests
metamorphic tests
cross-platform tests
integration tests
optimizer convergence tests
```

---

# 215. Join equivalence testing

Para una transformación:

```text
J → J'
```

deberá comprobarse sobre datasets generados que:

```text
Result(J)
=
Result(J')
```

considerando:

```text
NULLs
duplicates
missing FK targets
multiple matches
empty relations
```

---

# 216. Outer join datasets

Deberán incluir:

```text
matched rows
left-only rows
right-only rows
NULL join keys
duplicate join keys
```

---

# 217. Join elimination tests

Deberán cubrir:

```text
trusted FK
untrusted FK
nullable FK
NOT NULL FK
unique target
non-unique target
unused target
used target
```

---

# 218. Join strengthening tests

Cubrirán todas las combinaciones:

```text
LEFT
RIGHT
FULL
```

con predicates:

```text
null-rejecting
non-null-rejecting
UNKNOWN
volatile
security-bound
```

---

# 219. Decorrelation tests

Cubrirán:

```text
EXISTS
NOT EXISTS
correlated equality
correlated non-equality
aggregate subquery
volatile subquery
nested correlation
```

---

# 220. Search tests

Comprobarán:

```text
bounded alternatives
deterministic order
dependency preservation
no illegal Cartesian introduction
```

---

# 221. Security tests

Verificarán que:

```text
tenant boundaries
security barriers
policy provenance
```

no se pierdan por reordering/elimination.

---

# 222. Locking tests

Deberán cubrir targets de lock cuando el platform los soporte.

---

# 223. Telemetry

Métricas posibles:

```text
joins analyzed
join graphs built
Cartesian joins detected
joins strengthened
joins eliminated
semi joins introduced
anti joins introduced
decorrelations applied
reorder groups
alternatives generated
alternatives deduplicated
search states explored
search budget exhausted
optimization time
```

---

# 224. Explain integration

Una futura herramienta:

```text
voltstack database:explain
```

podrá mostrar:

```text
Declared Join Tree
Optimized Join Tree
Join Constraints
Join Transformations
Join Alternatives
Planner Selection
```

---

# 225. Developer Debug Toolbar

Podrá visualizar:

```text
Original joins: 7
Optimized joins: 5

Eliminated:
    customer_profiles
    regions

Strengthened:
    payments LEFT → INNER

Reordered:
    orders → customers → payments
```

---

# 226. Query fingerprint

El fingerprint estructural de joins incluirá:

```text
relation identities
join types
join predicates
join grouping
lateral intent
metadata
```

---

# 227. Optimized fingerprint

La alternativa optimizada incluirá:

```text
logical join tree
derived join types
relation ordering
semi/anti operators
barriers
semantic versions
```

---

# 228. Runtime bindings

No deberán formar parte del join structural fingerprint general.

---

# 229. Rule versions

Cambios en reglas relevantes deberán invalidar caches de optimized plans cuando corresponda.

---

# 230. Directory structure

```text
VoltStack/
└── Quantum/
    └── Database/
        └── Query/
            └── Optimizer/
                └── Join/
                    ├── Contract/
                    │   ├── JoinOptimizationRule.php
                    │   ├── JoinEnumerator.php
                    │   └── JoinAlternativeGenerator.php
                    │
                    ├── Model/
                    │   ├── JoinOptimizationFacts.php
                    │   ├── JoinPropertySet.php
                    │   ├── JoinConnectivity.php
                    │   ├── JoinAlternative.php
                    │   └── LogicalJoinTree.php
                    │
                    ├── Graph/
                    │   ├── JoinGraph.php
                    │   ├── JoinEdge.php
                    │   ├── JoinDependencyGraph.php
                    │   ├── JoinReorderGroup.php
                    │   └── JoinGraphBuilder.php
                    │
                    ├── Predicate/
                    │   ├── JoinPredicateClassifier.php
                    │   ├── EquiJoinKeySet.php
                    │   ├── EquiJoinDetector.php
                    │   ├── RangeJoinDetector.php
                    │   └── JoinPredicateFactExtractor.php
                    │
                    ├── Constraint/
                    │   ├── JoinConstraintAnalyzer.php
                    │   ├── JoinCardinalityAnalyzer.php
                    │   ├── JoinKeyAnalyzer.php
                    │   ├── ForeignKeyJoinAnalyzer.php
                    │   ├── FunctionalDependencyJoinAnalyzer.php
                    │   └── ConstraintTrustLevel.php
                    │
                    ├── Strengthening/
                    │   ├── JoinStrengtheningAnalyzer.php
                    │   ├── LeftToInnerJoinRule.php
                    │   ├── RightToInnerJoinRule.php
                    │   ├── FullToLeftJoinRule.php
                    │   ├── FullToRightJoinRule.php
                    │   └── FullToInnerJoinRule.php
                    │
                    ├── Elimination/
                    │   ├── JoinEliminationAnalyzer.php
                    │   ├── JoinEliminationProof.php
                    │   ├── InnerJoinEliminationRule.php
                    │   └── LeftJoinEliminationRule.php
                    │
                    ├── Decorrelation/
                    │   ├── JoinDecorrelationAnalyzer.php
                    │   ├── ExistsToSemiJoinRule.php
                    │   ├── NotExistsToAntiJoinRule.php
                    │   └── CorrelationDependencyAnalyzer.php
                    │
                    ├── Reorder/
                    │   ├── JoinReorderAnalyzer.php
                    │   ├── JoinReorderGroupBuilder.php
                    │   ├── JoinOrderingHint.php
                    │   └── JoinOrderingRequirement.php
                    │
                    ├── Enumeration/
                    │   ├── GreedyJoinEnumerator.php
                    │   ├── DynamicProgrammingJoinEnumerator.php
                    │   ├── DependencyAwareJoinEnumerator.php
                    │   └── JoinSearchState.php
                    │
                    ├── Alternative/
                    │   ├── JoinAlternativeSet.php
                    │   ├── JoinAlternativeFingerprint.php
                    │   └── JoinAlternativeDeduplicator.php
                    │
                    ├── Barrier/
                    │   ├── JoinBarrier.php
                    │   ├── JoinBarrierType.php
                    │   └── JoinBarrierAnalyzer.php
                    │
                    ├── Budget/
                    │   └── JoinOptimizationBudget.php
                    │
                    ├── Proof/
                    │   └── JoinOptimizationProof.php
                    │
                    ├── Diagnostic/
                    │   └── JoinOptimizationDiagnostic.php
                    │
                    └── Exception/
                        └── JoinOptimizationException.php
```

---

# 231. Ejemplo completo — LEFT → INNER

Consulta:

```sql
SELECT
    orders.id,
    customers.name
FROM orders
LEFT JOIN customers
    ON customers.id = orders.customer_id
WHERE customers.active = TRUE;
```

Representación inicial:

```text
Filter
│
│ customers.active = TRUE
│
└── LEFT JOIN
    ├── Orders
    └── Customers
```

Predicate Optimizer:

```text
customers.active = TRUE
    │
    ▼
NullRejecting(Customers)
```

Join Optimizer:

```text
LEFT JOIN
+
NullRejecting(right)
    │
    ▼
INNER JOIN
```

Resultado:

```text
Filter
│
└── INNER JOIN
    ├── Orders
    └── Customers
```

---

# 232. Nueva oportunidad

Una vez convertido en `INNER JOIN`:

```text
customers.active = TRUE
```

puede potencialmente empujarse hacia `Customers`.

```text
INNER JOIN
├── Orders
└── Filter(Customers.active = TRUE)
    └── Customers
```

Esto muestra por qué Predicate y Join Optimization deben coordinarse mediante scheduler.

---

# 233. Ejemplo completo — LEFT JOIN elimination

Consulta:

```sql
SELECT orders.id
FROM orders
LEFT JOIN customers
    ON customers.id = orders.customer_id;
```

Facts:

```text
customers.id UNIQUE
customer outputs unused
predicate deterministic
no security barrier
```

Entonces:

```text
LEFT JOIN
├── Orders
└── Customers
```

puede reducirse a:

```text
Orders
```

si el uniqueness proof garantiza que `Customers` no puede duplicar `Orders`.

---

# 234. Ejemplo — eliminación rechazada

Si:

```text
customers.id
```

no es unique:

```text
one order
→ multiple customer matches
```

por lo que eliminar el join cambiaría multiplicity.

Resultado:

```text
JoinEliminationBlockedByMultiplicity
```

---

# 235. Ejemplo — inner join + FK

Consulta:

```sql
SELECT orders.id
FROM orders
INNER JOIN customers
    ON customers.id = orders.customer_id;
```

Facts:

```text
orders.customer_id NOT NULL
FK orders.customer_id → customers.id
FK ENFORCED
customers.id UNIQUE
customer outputs unused
```

Proof:

```text
every order has exactly one matching customer
```

Por tanto:

```text
INNER JOIN
→ eliminated
```

---

# 236. Ejemplo — nullable FK

Si:

```text
orders.customer_id nullable
```

una order con:

```text
customer_id = NULL
```

sería eliminada por el inner join.

Por tanto:

```text
INNER JOIN elimination
BLOCKED
```

salvo que otro predicate demuestre:

```text
customer_id IS NOT NULL
```

en ese scope.

---

# 237. Constraint interaction

Esto demuestra:

```text
Schema Constraint
+
Predicate Constraint
=
Optimization Proof
```

Ejemplo:

```text
customer_id nullable in schema

WHERE customer_id IS NOT NULL
```

puede permitir una optimización que el schema aislado no permitía.

---

# 238. Ejemplo — EXISTS → SEMI

Entrada:

```text
Filter(
    EXISTS(
        Payments
        WHERE Payments.order_id = Orders.id
    ),
    Orders
)
```

Después de decorrelation:

```text
SEMI JOIN
├── Orders
└── Payments

ON Payments.order_id = Orders.id
```

Cardinalidad:

```text
each Orders row appears at most once
```

independientemente de cuántos payments existan.

---

# 239. Incorrect transformation

No usar:

```text
INNER JOIN Payments
```

porque:

```text
1 order
3 payments
→ 3 rows
```

cambiaría multiplicity.

---

# 240. Ejemplo — NOT EXISTS → ANTI

Entrada:

```text
Orders
WHERE NOT EXISTS (
    Payments
    WHERE Payments.order_id = Orders.id
)
```

Forma lógica:

```text
Orders
ANTI JOIN Payments
ON Payments.order_id = Orders.id
```

---

# 241. Ejemplo — join graph

Consulta:

```text
Orders O
Customers C
Payments P
Countries K
```

Predicates:

```text
O.customer_id = C.id
P.order_id = O.id
C.country_id = K.id
```

Graph:

```text
P
│
│ P.order_id = O.id
│
O
│
│ O.customer_id = C.id
│
C
│
│ C.country_id = K.id
│
K
```

---

# 242. Alternative orders

Podrían existir:

```text
((O ⋈ C) ⋈ P) ⋈ K
```

```text
((O ⋈ P) ⋈ C) ⋈ K
```

```text
(O ⋈ P) ⋈ (C ⋈ K)
```

si dependencies/barriers lo permiten.

---

# 243. Planner boundary

El Join Optimizer entrega alternativas como:

```text
Alternative A
Alternative B
Alternative C
```

El Planner decidirá después si una alternativa se implementa mediante:

```text
Hash Join
Merge Join
Nested Loop
...
```

---

# 244. Fórmula del sistema

```text
Join Optimization
=
Join Graph Analysis
+
Constraint Reasoning
+
Predicate Facts
+
Join Strengthening
+
Join Elimination
+
Decorrelation
+
Dependency-Aware Reordering
+
Bounded Alternative Generation
```

---

# 245. Safe join elimination

```text
SafeEliminate(J)
=
OutputsUnused(J)
∧ CardinalityPreserved(J)
∧ MultiplicityPreserved(J)
∧ FilteringPreserved(J)
∧ NullSemanticsPreserved(J)
∧ SecurityPreserved(J)
∧ LockingPreserved(J)
∧ VolatilityPreserved(J)
```

---

# 246. Safe join reorder

```text
SafeReorder(J1, J2)
=
LogicalPropertiesAllow
∧ DependenciesPreserved
∧ NullExtensionPreserved
∧ PredicateScopesPreserved
∧ CorrelationPreserved
∧ SecurityPreserved
∧ LockingPreserved
∧ VolatilityPreserved
```

---

# 247. Safe join strengthening

```text
SafeStrengthen(OuterJoin)
=
NullExtendedRowsProvablyRejected
∧ PredicatePlacementPreserved
∧ SecurityPreserved
∧ VolatilityPreserved
```

---

# 248. Join alternative legality

```text
LegalAlternative(A)
=
ConnectedOrRequiredCartesian(A)
∧ DependencyGraphSatisfied(A)
∧ BarriersSatisfied(A)
∧ ScopeValid(A)
∧ CapabilitiesSatisfied(A)
```

---

# 249. Arquitectura resumida

```text
                   Semantic Query
                        │
                        ▼
              ┌──────────────────┐
              │   Join Graph     │
              └────────┬─────────┘
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
     Constraints   Predicates   Dependencies
          │            │             │
          └────────────┼─────────────┘
                       ▼
               Join Classification
                       │
                       ▼
              Join Strengthening
                       │
                       ▼
                Join Elimination
                       │
                       ▼
                 Decorrelation
                       │
                       ▼
               Reorder Groups
                       │
                       ▼
             Alternative Search
                       │
                       ▼
               Deduplication
                       │
                       ▼
             Logical Alternatives
                       │
                       ▼
                  Query Planner
```

---

# 250. Invariantes arquitectónicos

## DB-JOINOPT-001

Join optimization será exclusivamente lógica.

## DB-JOINOPT-002

Join Optimizer no seleccionará algoritmos físicos.

## DB-JOINOPT-003

Join Optimizer no generará SQL.

## DB-JOINOPT-004

Join Optimizer no abrirá conexiones.

## DB-JOINOPT-005

Join Optimizer no resolverá nombres nuevamente.

## DB-JOINOPT-006

Join Optimizer consumirá SemanticQueryArtifact.

## DB-JOINOPT-007

JoinGraph no sustituirá al AST.

## DB-JOINOPT-008

JoinGraph será una vista de optimización.

## DB-JOINOPT-009

Relation instance será distinta de schema relation.

## DB-JOINOPT-010

Self joins conservarán identidades independientes.

## DB-JOINOPT-011

JoinId será independiente del SQL textual.

## DB-JOINOPT-012

Join predicates serán clasificados semánticamente.

## DB-JOINOPT-013

Equi-join detection será type-aware.

## DB-JOINOPT-014

Equi-join detection respetará domain semantics.

## DB-JOINOPT-015

Multi-column equi joins serán first-class.

## DB-JOINOPT-016

Non-equi joins serán soportados.

## DB-JOINOPT-017

Range joins podrán clasificarse.

## DB-JOINOPT-018

Join classification no elegirá physical operator.

## DB-JOINOPT-019

FK no creará joins automáticamente.

## DB-JOINOPT-020

FK será evidence para optimización.

## DB-JOINOPT-021

Constraint trust será explícito.

## DB-JOINOPT-022

Un constraint no confiable no justificará una eliminación insegura.

## DB-JOINOPT-023

Logical cardinality facts serán distintos de estimates.

## DB-JOINOPT-024

Join elimination requerirá proof.

## DB-JOINOPT-025

Join elimination preservará cardinalidad.

## DB-JOINOPT-026

Join elimination preservará multiplicity.

## DB-JOINOPT-027

Join elimination preservará output semantics.

## DB-JOINOPT-028

Join elimination preservará filtering.

## DB-JOINOPT-029

Join elimination preservará security.

## DB-JOINOPT-030

Join elimination preservará locking.

## DB-JOINOPT-031

Join elimination preservará volatility semantics.

## DB-JOINOPT-032

Many-to-one no se asumirá sin proof.

## DB-JOINOPT-033

Nullable FK será considerada.

## DB-JOINOPT-034

NOT NULL facts podrán provenir de predicates.

## DB-JOINOPT-035

Unique target será requerido cuando multiplicity dependa de ello.

## DB-JOINOPT-036

LEFT JOIN elimination analizará duplicate amplification.

## DB-JOINOPT-037

FULL JOIN elimination será conservadora.

## DB-JOINOPT-038

Join strengthening requerirá null-rejection proof.

## DB-JOINOPT-039

LEFT podrá convertirse a INNER sólo con proof.

## DB-JOINOPT-040

RIGHT podrá convertirse a INNER sólo con proof.

## DB-JOINOPT-041

FULL podrá reducirse sólo con proof.

## DB-JOINOPT-042

Predicate placement será preservado.

## DB-JOINOPT-043

3VL será preservado.

## DB-JOINOPT-044

Outer join null-extension será explícita.

## DB-JOINOPT-045

Join commutativity no será universal.

## DB-JOINOPT-046

Join associativity no será universal.

## DB-JOINOPT-047

Outer joins crearán reorder boundaries cuando corresponda.

## DB-JOINOPT-048

Lateral dependencies restringirán reorder.

## DB-JOINOPT-049

Correlation restringirá reorder.

## DB-JOINOPT-050

Security barriers restringirán reorder.

## DB-JOINOPT-051

Volatility restringirá reorder.

## DB-JOINOPT-052

Locking podrá restringir reorder.

## DB-JOINOPT-053

JoinDependencyGraph será explícito.

## DB-JOINOPT-054

LATERAL será modelado como dependency semantics.

## DB-JOINOPT-055

LATERAL no será tratado sólo como syntax flag.

## DB-JOINOPT-056

EXISTS podrá convertirse a SEMI JOIN sólo con proof.

## DB-JOINOPT-057

NOT EXISTS podrá convertirse a ANTI JOIN sólo con proof.

## DB-JOINOPT-058

EXISTS no se convertirá ingenuamente en INNER JOIN.

## DB-JOINOPT-059

SEMI JOIN preservará left multiplicity.

## DB-JOINOPT-060

ANTI JOIN preservará left schema.

## DB-JOINOPT-061

NOT IN no se convertirá ingenuamente en ANTI JOIN.

## DB-JOINOPT-062

NULL-aware anti semantics permanecerá distinguible.

## DB-JOINOPT-063

Decorrelation respetará volatility.

## DB-JOINOPT-064

Decorrelation respetará correlation scope.

## DB-JOINOPT-065

Scalar subquery decorrelation requerirá cardinality proof.

## DB-JOINOPT-066

Subquery Optimizer y Join Optimizer no duplicarán decorrelation engines.

## DB-JOINOPT-067

CROSS + WHERE podrá convertirse en logical inner join cuando sea seguro.

## DB-JOINOPT-068

Derived join edges tendrán provenance.

## DB-JOINOPT-069

Derived facts no necesitarán convertirse en AST predicates.

## DB-JOINOPT-070

Functional dependencies serán consumibles.

## DB-JOINOPT-071

Key propagation respetará outer joins.

## DB-JOINOPT-072

Unique y primary key permanecerán conceptos distintos.

## DB-JOINOPT-073

Composite keys serán first-class.

## DB-JOINOPT-074

Composite foreign keys serán first-class.

## DB-JOINOPT-075

Partial constraints tendrán applicability conditions.

## DB-JOINOPT-076

DISTINCT no justificará join elimination automáticamente.

## DB-JOINOPT-077

Aggregation será considerada en join elimination.

## DB-JOINOPT-078

COUNT(*) multiplicity será preservada.

## DB-JOINOPT-079

Window input cardinality será preservada.

## DB-JOINOPT-080

Recursive CTE dependencies serán respetadas.

## DB-JOINOPT-081

Join search será bounded.

## DB-JOINOPT-082

Join search no enumerará ilimitadamente.

## DB-JOINOPT-083

Reorder groups tendrán tamaño máximo configurable.

## DB-JOINOPT-084

DP enumeration será bounded.

## DB-JOINOPT-085

Greedy enumeration será determinista.

## DB-JOINOPT-086

Connected-first será heuristic, no semantic invariant.

## DB-JOINOPT-087

Cartesian joins seguirán siendo representables.

## DB-JOINOPT-088

Logical alternatives serán distintas de physical plans.

## DB-JOINOPT-089

Bushy trees serán representables.

## DB-JOINOPT-090

Left-deep trees no serán invariante arquitectónico.

## DB-JOINOPT-091

Alternative deduplication utilizará fingerprints semánticos/lógicos.

## DB-JOINOPT-092

Search tie breaking será determinista.

## DB-JOINOPT-093

Join hints serán metadata estructurada.

## DB-JOINOPT-094

PREFER será distinto de REQUIRE.

## DB-JOINOPT-095

Security requirements dominarán performance hints.

## DB-JOINOPT-096

Tenant integration no será dependencia obligatoria del core.

## DB-JOINOPT-097

SecurityDomain metadata podrá restringir joins.

## DB-JOINOPT-098

Cross-domain joins podrán ser barriers.

## DB-JOINOPT-099

Lock target semantics serán preservadas.

## DB-JOINOPT-100

Volatile join sources serán barriers cuando corresponda.

## DB-JOINOPT-101

Extension joins deberán declarar semantic properties.

## DB-JOINOPT-102

Unknown extension joins serán barriers.

## DB-JOINOPT-103

VoltStack no adivinará propiedades de joins desconocidos.

## DB-JOINOPT-104

Logical optimizer vocabulary podrá exceder SQL superficial del target.

## DB-JOINOPT-105

SEMI/ANTI podrán ser operadores internos.

## DB-JOINOPT-106

Compiler decidirá representación SQL final.

## DB-JOINOPT-107

Platform checks directos por vendor estarán prohibidos.

## DB-JOINOPT-108

Capabilities gobernarán diferencias.

## DB-JOINOPT-109

Join rules utilizarán Optimization Rule System.

## DB-JOINOPT-110

Join rewrites utilizarán Query Rewrite System.

## DB-JOINOPT-111

Join transformations producirán proofs.

## DB-JOINOPT-112

Join transformations producirán ChangeSets.

## DB-JOINOPT-113

Fact invalidation será explícita.

## DB-JOINOPT-114

Optimizer coordination usará scheduler.

## DB-JOINOPT-115

JoinOptimizer no llamará recursivamente PredicateOptimizer.

## DB-JOINOPT-116

Fixpoint global será bounded.

## DB-JOINOPT-117

Budget exhaustion conservará una query válida.

## DB-JOINOPT-118

Failure to optimize no será query failure.

## DB-JOINOPT-119

No habrá hidden database I/O.

## DB-JOINOPT-120

Schema information vendrá mediante snapshots/views.

## DB-JOINOPT-121

Search state será operation-scoped.

## DB-JOINOPT-122

JoinGraph será operation-scoped.

## DB-JOINOPT-123

Join alternatives serán operation-scoped.

## DB-JOINOPT-124

No habrá global current relation.

## DB-JOINOPT-125

No habrá global current join.

## DB-JOINOPT-126

No habrá global search state.

## DB-JOINOPT-127

Frozen rule descriptors podrán compartirse entre requests.

## DB-JOINOPT-128

Persistent workers no compartirán mutable query state.

## DB-JOINOPT-129

FrankenPHP será soportado de forma segura.

## DB-JOINOPT-130

RoadRunner podrá soportarse sin rediseño.

## DB-JOINOPT-131

OpenSwoole podrá soportarse sin rediseño.

## DB-JOINOPT-132

Optimizer traces deberán poder explicar join elimination.

## DB-JOINOPT-133

Optimizer traces deberán poder explicar join strengthening.

## DB-JOINOPT-134

Optimizer traces deberán poder explicar reorder decisions.

## DB-JOINOPT-135

Sensitive runtime bindings no aparecerán por defecto en traces.

## DB-JOINOPT-136

Testing incluirá NULL.

## DB-JOINOPT-137

Testing incluirá duplicate amplification.

## DB-JOINOPT-138

Testing incluirá missing FK targets.

## DB-JOINOPT-139

Testing incluirá outer join unmatched rows.

## DB-JOINOPT-140

Testing incluirá correlation.

## DB-JOINOPT-141

Testing incluirá LATERAL.

## DB-JOINOPT-142

Testing incluirá security barriers.

## DB-JOINOPT-143

Testing incluirá locking cuando sea soportado.

## DB-JOINOPT-144

Testing incluirá volatile expressions.

## DB-JOINOPT-145

Testing incluirá composite keys.

## DB-JOINOPT-146

Testing incluirá untrusted constraints.

## DB-JOINOPT-147

Correctness dominará join search aggressiveness.

## DB-JOINOPT-148

El optimizer podrá detener búsqueda en cualquier momento conservando alternativas válidas.

## DB-JOINOPT-149

El Planner será la autoridad sobre physical join algorithms.

## DB-JOINOPT-150

Join Optimization permanecerá desacoplado de PDO, ORM y SQL textual.

---

# 251. Anti-patterns

## 251.1 Elegir Hash Join en Join Optimizer

```text
EQUI JOIN
→ HASH JOIN
```

**Rechazado.**

---

## 251.2 Reordenar todos los joins

sin analizar:

```text
outer semantics
lateral
correlation
security
```

**Rechazado.**

---

## 251.3 Eliminar join porque sus columnas no aparecen en SELECT

Insuficiente.

Puede:

```text
filter
duplicate
lock
enforce security
```

**Rechazado.**

---

## 251.4 FK implica eliminación automática

**Rechazado.**

---

## 251.5 LEFT JOIN siempre preserva multiplicity

Falso si el lado derecho tiene múltiples matches.

**Rechazado.**

---

## 251.6 EXISTS → INNER JOIN

sin SEMI semantics.

**Rechazado.**

---

## 251.7 NOT EXISTS → LEFT JOIN IS NULL

como transformación universal.

**Rechazado sin proof.**

---

## 251.8 NOT IN → ANTI JOIN

sin null analysis.

**Rechazado.**

---

## 251.9 LATERAL como simple string

**Rechazado.**

---

## 251.10 Ignorar composite keys

**Rechazado.**

---

## 251.11 Usar statistics como constraints

**Rechazado.**

---

## 251.12 Usar constraints no confiables como guarantees

**Rechazado.**

---

## 251.13 Enumerar todas las permutaciones

sin budget.

**Rechazado.**

---

## 251.14 Limitar arquitectura permanentemente a left-deep joins

**Rechazado.**

---

## 251.15 Optimizar atravesando security barriers

**Rechazado.**

---

## 251.16 Consultar la base de datos desde Join Optimizer

**Rechazado.**

---

## 251.17 Estado global de búsqueda

**Rechazado.**

---

# 252. Decisión arquitectónica final

VoltStack implementará `Join Optimization System` como un subsistema:

```text
logical
graph-based
constraint-aware
predicate-aware
null-aware
cardinality-aware
multiplicity-aware
dependency-aware
correlation-aware
security-aware
capability-driven
rule-based
bounded
deterministic
explainable
persistent-runtime safe
```

La regla fundamental será:

> **El Join Optimizer de VoltStack nunca elegirá cómo ejecutar físicamente un join. Su responsabilidad será demostrar qué transformaciones lógicas de relaciones son equivalentes y generar un conjunto acotado de alternativas legales para que el Query Planner decida posteriormente cómo ejecutarlas.**

---

# 253. Resultado arquitectónico

```text
Declared Join Structure
        │
        ▼
Semantic Join Resolution
        │
        ▼
JoinGraph
        │
        ├── predicates
        ├── constraints
        ├── keys
        ├── dependencies
        ├── null-extension
        ├── security
        └── correlation
        │
        ▼
Logical Join Optimization
        │
        ├── Strengthening
        ├── Elimination
        ├── Decorrelation
        ├── Semi/Anti
        ├── Reordering
        └── Alternative Generation
        │
        ▼
Legal Logical Alternatives
        │
        ▼
Query Planner
        │
        ▼
Physical Planning
        │
        ├── Hash Join
        ├── Merge Join
        ├── Nested Loop
        └── Other Strategies
```

---

# 254. Bloque actual

```text
Block 5 — Optimizer and Planner

55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
56_DATABASE_QUERY_REWRITE_SYSTEM.md
57_DATABASE_QUERY_OPTIMIZATION_RULE_SYSTEM.md
58_DATABASE_PREDICATE_OPTIMIZATION_SYSTEM.md
59_DATABASE_JOIN_OPTIMIZATION_SYSTEM.md              ← actual
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
61_DATABASE_QUERY_COST_HINT_SYSTEM.md
62_DATABASE_QUERY_PLANNER_ARCHITECTURE.md
63_DATABASE_LOGICAL_QUERY_PLAN_SYSTEM.md
64_DATABASE_PHYSICAL_QUERY_PLAN_SYSTEM.md
65_DATABASE_EXECUTION_PLAN_SYSTEM.md
```

---

# 255. Siguiente documento

```text
60_DATABASE_QUERY_DEDUPLICATION_SYSTEM.md
```

El siguiente sistema deberá formalizar cómo VoltStack detectará equivalencia y duplicación en:

```text
expressions
predicates
subqueries
CTEs
logical operators
join alternatives
optimization states
query fragments
```

sin confundir:

```text
Structural Equality
Semantic Equivalence
Logical Equivalence
Runtime Equality
```

y deberá introducir conceptos como:

```text
StructuralFingerprint
SemanticFingerprint
LogicalFingerprint

Canonical Query Representation

Expression Deduplication
Predicate Deduplication
Subquery Deduplication
CTE Reuse Analysis

Common Subexpression Detection

Optimization State Deduplication

Join Alternative Deduplication

Memoization Keys
Memo Groups
Equivalence Classes

Volatility Barriers
Security Barriers
Correlation Boundaries

Parameter Identity
Binding Independence

Schema Fingerprints
Capability Fingerprints
Rule Version Fingerprints

Bounded Deduplication
Collision Safety
Deterministic Canonicalization

Persistent Runtime Safety
```