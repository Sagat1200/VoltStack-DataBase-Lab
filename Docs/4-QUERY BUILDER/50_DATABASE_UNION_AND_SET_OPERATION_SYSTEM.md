# 50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md

# VoltStack Quantum Database
## Union and Set Operation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 50 — Union and Set Operation System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Query Composition  
**Versión:** 1.0

---

# 1. Propósito

`Union and Set Operation System` define la arquitectura mediante la cual VoltStack representa, valida, analiza, optimiza, planifica y posteriormente compila operaciones entre resultados de queries.

El sistema deberá soportar conceptualmente:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
```

además de permitir extensiones futuras.

La regla fundamental será:

```text
Set Operation
≠
SQL concatenation
```

Por tanto:

```text
Query A
UNION
Query B
```

no se representará internamente como:

```php
$sqlA . ' UNION ' . $sqlB;
```

sino mediante un modelo estructurado:

```text
SetOperationQuery
├── Left Operand
├── Operator
└── Right Operand
```

capaz de participar completamente en:

```text
Normalization
Validation
Semantic Analysis
Type Inference
Constraint Analysis
Optimization
Planning
Compilation
Telemetry
Security
Caching
```

---

# 2. Objetivos

El sistema deberá proporcionar:

- representación estructurada de operaciones de conjuntos;
- composición arbitraria de queries;
- soporte para árboles de operaciones;
- preservación explícita del operador;
- preservación de semántica `ALL`/distinct;
- validación de aridad;
- reconciliación de tipos;
- reconciliación de nullability;
- resolución del output relation;
- reglas de nombres de columnas;
- manejo explícito de precedencia;
- grouping estructural;
- scopes independientes por operand;
- parámetros independientes;
- dependency aggregation;
- lineage por columna;
- capability requirements;
- portability analysis;
- optimizaciones seguras;
- integración con CTEs;
- integración con subqueries;
- persistent-runtime safety.

---

# 3. No responsabilidades

Este sistema no deberá:

```text
generar SQL
concatenar SQL
abrir conexiones
ejecutar queries
hidratar entidades
resolver PDO statements
seleccionar índices
realizar schema introspection oculta
decidir algoritmos físicos
realizar ORM persistence
```

---

# 4. Posición arquitectónica

```text
Application
    │
    ▼
Query Builder
    │
    ▼
Set Operation Builder
    │
    ▼
Set Query Model
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
    ├── Scope Resolution
    ├── Symbol Resolution
    ├── Type Inference
    ├── Output Reconciliation
    ├── Constraint Analysis
    └── Capability Analysis
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
Compiler
    │
    ▼
Executor
```

---

# 5. Regla maestra

```text
Set Operation Builder
≠
Set Operation Compiler
```

y:

```text
Set Operation
≠
Physical execution strategy
```

El Builder declara:

> qué resultados desea combinar el desarrollador.

Semantic Analysis determina:

> si esos resultados son semánticamente compatibles.

Optimizer determina:

> qué transformaciones equivalentes son posibles.

Planner determina:

> cómo organizar físicamente la operación.

Compiler determina:

> cómo expresarla en el dialecto objetivo.

---

# 6. Relación con documentos anteriores

Este sistema se apoya principalmente en:

```text
23_DATABASE_QUERY_ARCHITECTURE.md
24_DATABASE_QUERY_MODEL.md
25_DATABASE_QUERY_AST_SYSTEM.md
26_DATABASE_QUERY_AST_NODE_MODEL.md
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
30_DATABASE_QUERY_TYPE_SYSTEM.md
31_DATABASE_QUERY_METADATA_SYSTEM.md
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
```

---

# 7. Concepto de operación de conjunto

Una operación de conjunto combina dos relaciones compatibles:

```text
Relation A
    │
    ├── UNION
    ├── INTERSECT
    └── EXCEPT
    │
Relation B
    │
    ▼
Result Relation
```

Formalmente:

```text
R = A OP B
```

donde:

```text
OP ∈ {
    UNION,
    INTERSECT,
    EXCEPT
}
```

con una cuantificación:

```text
DISTINCT
ALL
```

---

# 8. Operator ≠ quantifier

VoltStack no deberá modelar:

```text
UNION
UNION ALL
```

necesariamente como operadores totalmente desconectados.

Puede representarlos como:

```text
SetOperator = UNION
SetQuantifier = DISTINCT
```

y:

```text
SetOperator = UNION
SetQuantifier = ALL
```

---

# 9. Modelo recomendado

```php
enum SetOperator
{
    case UNION;
    case INTERSECT;
    case EXCEPT;
}
```

y:

```php
enum SetQuantifier
{
    case DISTINCT;
    case ALL;
}
```

---

# 10. SetOperationSpecification

```php
final readonly class SetOperationSpecification
{
    public function __construct(
        public SetOperator $operator,
        public SetQuantifier $quantifier,
    ) {}
}
```

---

# 11. Ventaja

Esto evita proliferación innecesaria:

```text
UNION
UNION_ALL
INTERSECT
INTERSECT_ALL
EXCEPT
EXCEPT_ALL
```

y permite razonar independientemente sobre:

```text
operation semantics
+
duplicate semantics
```

---

# 12. UNION

`UNION` combina filas compatibles eliminando duplicados según la semántica de igualdad del motor.

Conceptualmente:

```text
A = {1, 2}
B = {2, 3}

A UNION B
=
{1, 2, 3}
```

---

# 13. UNION ALL

`UNION ALL` conserva multiplicidades.

```text
A = [1, 2]
B = [2, 3]

A UNION ALL B
=
[1, 2, 2, 3]
```

---

# 14. INTERSECT

`INTERSECT` conserva filas presentes en ambos operands según la semántica definida por la plataforma.

---

# 15. INTERSECT ALL

Cuando sea soportado:

```text
INTERSECT ALL
```

preservará multiplicidades según semántica multiset.

---

# 16. EXCEPT

`EXCEPT` representa diferencia de conjuntos.

```text
A EXCEPT B
```

devuelve filas de A que no permanecen después de aplicar la semántica correspondiente respecto a B.

---

# 17. EXCEPT ALL

`EXCEPT ALL` aplica diferencia sobre multiplicidades.

---

# 18. Set semantics ≠ PHP array semantics

VoltStack nunca deberá evaluar la semántica mediante:

```php
array_unique()
array_intersect()
array_diff()
```

para razonar sobre SQL.

Las bases de datos pueden aplicar:

```text
database type semantics
collation
NULL rules
numeric coercion
temporal semantics
domain semantics
```

---

# 19. Query operands

Cada operand deberá ser un query artifact estructurado.

```text
SetOperation
├── Operand A
└── Operand B
```

---

# 20. Operand types

Inicialmente podrán participar:

```text
SelectQuery
SetOperationQuery
CTE Query
compatible extension query
```

y otras categorías cuando la arquitectura lo permita.

---

# 21. SetOperand

```php
interface SetOperand
{
    public function queryArtifact(): QueryBuildArtifact;
}
```

La representación interna final podrá ser más especializada.

---

# 22. Operand immutability

Cada operand deberá estar finalizado antes de entrar en el parent set operation.

Prohibido:

```text
SetOperation
├── mutable Builder A
└── mutable Builder B
```

Permitido:

```text
SetOperation
├── immutable QueryArtifact A
└── immutable QueryArtifact B
```

---

# 23. SetOperationId

Cada operación tendrá identidad:

```text
SetOperationId
```

Ejemplo:

```text
SET1
SET2
SET3
```

---

# 24. Identity ≠ structure

Dos operaciones estructuralmente equivalentes pueden poseer identidades diferentes.

---

# 25. Binary representation

La forma base deberá ser binaria:

```text
SetOperationNode
├── left
├── operator
├── quantifier
└── right
```

---

# 26. Árbol de operaciones

Una composición:

```text
A UNION B INTERSECT C
```

deberá convertirse en un árbol explícito.

Por ejemplo:

```text
        INTERSECT
        /       \
     UNION       C
    /     \
   A       B
```

o:

```text
        UNION
        /   \
       A   INTERSECT
           /       \
          B         C
```

dependiendo de la agrupación semántica declarada.

---

# 27. Precedence must be explicit

VoltStack no deberá depender accidentalmente de:

```text
SQL textual precedence
```

para determinar el significado interno.

---

# 28. Grouping

La agrupación deberá existir en el Query Model/AST.

```text
SetGroup
└── SetOperation
```

---

# 29. Parentheses ≠ semantics

Los paréntesis SQL son una representación del grouping.

Internamente:

```text
Grouping
→ semantic structure

Compiler
→ parentheses if required
```

---

# 30. Builder example

```php
$a = DB::table('users')
    ->select('email');

$b = DB::table('customers')
    ->select('email');

$query = $a->union($b);
```

---

# 31. Structural result

```text
SetOperationQuery
├── Left
│   └── Select users.email
├── Operator
│   └── UNION
├── Quantifier
│   └── DISTINCT
└── Right
    └── Select customers.email
```

---

# 32. UNION ALL API

```php
$query = $a->unionAll($b);
```

produce:

```text
operator = UNION
quantifier = ALL
```

---

# 33. INTERSECT API

```php
$query = $a->intersect($b);
```

---

# 34. INTERSECT ALL API

```php
$query = $a->intersectAll($b);
```

si la capability correspondiente existe.

---

# 35. EXCEPT API

```php
$query = $a->except($b);
```

---

# 36. EXCEPT ALL API

```php
$query = $a->exceptAll($b);
```

cuando esté soportado.

---

# 37. Fluent API ≠ mutation of finalized operand

Si:

```php
$a->union($b);
```

produce una set query, no deberá modificar silenciosamente un artifact finalizado de `$a`.

---

# 38. Recommended composition model

```text
Builder A
   │
   ▼
Snapshot A
   │
   ├──────────┐
   │          │
   ▼          ▼
Set Builder  Snapshot B
   │          │
   └────┬─────┘
        ▼
SetOperationQueryModel
```

---

# 39. Arity compatibility

La regla estructural-semántica principal será:

```text
width(A)
=
width(B)
```

---

# 40. Ejemplo válido

```text
A:
id | name

B:
id | name
```

Aridad:

```text
2 = 2
```

---

# 41. Ejemplo inválido

```text
A:
id | name

B:
id | name | email
```

produce:

```text
2 ≠ 3
```

y deberá fallar antes de compilación.

---

# 42. Builder cannot always know arity

Si existen:

```text
*
table.*
raw projections
extension projections
```

el Builder puede no conocer el número real de columnas.

Por tanto:

```text
Builder Validation
→ structural checks

Semantic Analysis
→ authoritative output width
```

---

# 43. Wildcards

Ejemplo:

```text
SELECT *
UNION
SELECT *
```

no se asumirá compatible hasta resolver ambos outputs.

---

# 44. Schema-aware expansion

La expansión semántica de wildcards dependerá de:

```text
QuerySchemaView
```

y no de schema introspection oculta desde Builder.

---

# 45. Positional compatibility

Set operations comparan outputs por posición.

```text
A.column[1] ↔ B.column[1]
A.column[2] ↔ B.column[2]
...
```

No por nombre.

---

# 46. Critical distinction

```text
Column Name Compatibility
≠
Column Position Compatibility
```

Dos queries:

```text
A:
id, name

B:
customer_id, full_name
```

pueden ser estructuralmente compatibles si sus tipos por posición son compatibles.

---

# 47. Output naming

El nombre del output normalmente deberá derivarse de un operand definido por la semántica de la operación/plataforma.

VoltStack deberá normalizar esto a una regla semántica estable.

---

# 48. Recommended VoltStack rule

El `SetOperationOutputRelation` utilizará como base los nombres lógicos del **leftmost output operand**, salvo que una feature semántica explícita defina otra cosa.

---

# 49. Leftmost output

Para:

```text
A UNION B UNION C
```

los nombres públicos del resultado se derivarán de A.

---

# 50. Identity remains new

Aunque los nombres procedan de A:

```text
A.OutputColumn1
≠
SetResult.OutputColumn1
```

---

# 51. Output lineage

La nueva columna tendrá lineage hacia todos los operands relevantes.

```text
SetOutputColumn #1
├── derives from A.column #1
├── derives from B.column #1
└── derives from C.column #1
```

---

# 52. Multi-source lineage

Por tanto, una set operation introduce naturalmente:

```text
many-to-one lineage
```

---

# 53. SetOperationOutputRelation

Conceptualmente:

```php
final readonly class SetOperationOutputRelation
{
    /**
     * @param list<SetOperationOutputColumn> $columns
     */
    public function __construct(
        public RelationId $relationId,
        public array $columns,
    ) {}
}
```

---

# 54. Output column

```php
final readonly class SetOperationOutputColumn
{
    public function __construct(
        public OutputColumnId $id,
        public string $name,
        public QueryType $type,
        public Nullability $nullability,
        public LineageSet $lineage,
    ) {}
}
```

---

# 55. Type compatibility

Para cada posición:

```text
Type(Ai)
+
Type(Bi)
→
Common Set Type
```

---

# 56. Common type

El resultado deberá usar el sistema definido en:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
```

---

# 57. No PHP coercion

Prohibido resolver:

```text
integer + string
```

usando las reglas de PHP.

---

# 58. Domain types

Si:

```text
A = UserId
B = UserId
```

el sistema podrá preservar:

```text
UserId
```

cuando las reglas de dominio lo permitan.

---

# 59. Different domain types

Si:

```text
A = UserId
B = OrderId
```

aunque ambos físicamente sean integers, no deberán considerarse automáticamente semánticamente equivalentes.

---

# 60. Domain preservation

La reconciliación deberá considerar:

```text
semantic domain identity
```

antes de representación física.

---

# 61. Compatible base types

Una política explícita podrá permitir:

```text
Domain<UserId>
+
Integer
→
Integer
```

o requerir cast explícito.

Esto deberá definirse mediante Type System policy.

---

# 62. Numeric reconciliation

Ejemplo conceptual:

```text
INTEGER
+
BIGINT
→
BIGINT
```

si la política de tipos lo determina.

---

# 63. Decimal reconciliation

La precisión y escala deberán preservarse o reconciliarse explícitamente.

---

# 64. String reconciliation

Se deberán considerar:

```text
length
collation
character set semantics
domain identity
```

cuando sean relevantes.

---

# 65. Temporal reconciliation

```text
DATE
TIMESTAMP
TIMESTAMP WITH TIME ZONE
```

no deberán mezclarse mediante heurísticas arbitrarias.

---

# 66. Unknown type

Si uno de los operands tiene:

```text
Unknown
```

Type Inference podrá obtener restricciones del otro operand.

---

# 67. Any ≠ Unknown

Se conserva la distinción establecida anteriormente:

```text
Unknown
≠
Any
```

---

# 68. NULL literal

Ejemplo:

```text
SELECT NULL
UNION ALL
SELECT user_id
```

puede permitir inferir el tipo de la primera columna a partir de `user_id`.

---

# 69. Nullability reconciliation

En general:

```text
Nullable(A)
OR
Nullable(B)
→
Nullable(Result)
```

pero la implementación deberá usar el lattice formal del Query Type System.

---

# 70. Schema nullability ≠ result nullability

El resultado depende de:

```text
expression semantics
set operation semantics
type coercions
```

no sólo del schema físico.

---

# 71. Explicit casts

El desarrollador podrá utilizar:

```text
CastExpression
```

para resolver incompatibilidades.

---

# 72. Builder does not insert arbitrary casts

El Builder no deberá inventar casts para hacer que cualquier set operation compile.

---

# 73. Safe coercion

Type Inference podrá declarar una:

```text
RequiredCoercion
```

cuando sea semánticamente segura.

---

# 74. Physical cast

La representación física de esa coerción pertenecerá al Compiler.

---

# 75. Duplicate semantics

La cuantificación determina si la operación utiliza semántica:

```text
DISTINCT
```

o:

```text
ALL
```

---

# 76. DISTINCT ≠ Builder deduplication

VoltStack no deduplicará filas en memoria desde Query Builder.

---

# 77. Equality semantics

La eliminación de duplicados depende de la semántica de comparación del target database.

---

# 78. NULL duplicate semantics

La semántica de NULL dentro de operaciones de conjunto no deberá inferirse mediante igualdad PHP.

---

# 79. Collation impact

Strings bajo distintas collations pueden requerir:

```text
compatibility analysis
coercion
capability requirement
explicit failure
```

---

# 80. UNION ALL optimization importance

`UNION ALL` generalmente permite más libertad porque no requiere deduplicación.

Sin embargo, el Optimizer seguirá siendo responsable de las transformaciones.

---

# 81. Set tree

Una query compleja podrá ser:

```text
                 UNION ALL
                /         \
           INTERSECT       D
           /       \
          A       EXCEPT
                 /      \
                B        C
```

---

# 82. SetOperationTree

El sistema deberá poder representar:

```text
SetOperationTree
```

como estructura lógica.

---

# 83. Set operation tree ≠ SQL parentheses

El árbol representa significado.

Los paréntesis son sólo una estrategia del Compiler para preservarlo.

---

# 84. Associativity

No se deberá asumir que todas las transformaciones:

```text
(A OP B) OP C
↔
A OP (B OP C)
```

son válidas.

---

# 85. UNION ALL associativity

Puede existir una regla de normalización/optimización para flattening de `UNION ALL` cuando sea demostrablemente equivalente.

---

# 86. UNION DISTINCT

También puede ser asociativa matemáticamente bajo determinadas semánticas, pero VoltStack deberá considerar:

```text
types
coercions
collations
extensions
ordering boundaries
limits
volatility
```

antes de reestructurar.

---

# 87. EXCEPT

`EXCEPT` no deberá tratarse como conmutativo.

```text
A EXCEPT B
≠
B EXCEPT A
```

---

# 88. INTERSECT

Aunque matemáticamente pueda ser conmutativo, el Optimizer deberá considerar efectos semánticos adicionales antes de cambiar operands.

---

# 89. Builder preserves declared structure

El Builder no:

```text
reorders
flattens
commutes
associates
```

operands por razones de optimización.

---

# 90. Normalization

Normalization sólo podrá realizar transformaciones canónicas claramente semantics-preserving.

---

# 91. Optimizer ownership

Transformaciones algebraicas pertenecen a:

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
```

y documentos relacionados.

---

# 92. Ordering

`ORDER BY` requiere un scope claramente definido.

---

# 93. Operand ordering

Un operand puede contener ordering interno cuando semánticamente permitido.

Pero:

```text
Operand ORDER BY
≠
Set Result ORDER BY
```

---

# 94. Set result ordering

Ejemplo:

```php
$query = $a
    ->union($b)
    ->orderBy('email');
```

El `orderBy` deberá aplicarse al:

```text
SetOperationOutputRelation
```

y no sólo a `$b`.

---

# 95. Output alias resolution

El ORDER BY externo podrá resolver:

```text
OutputAliasReference
```

contra el output relation del set.

---

# 96. Order ordinal

Si VoltStack soporta:

```text
ORDER BY 1
```

deberá utilizar:

```text
OrderOrdinalExpression
```

y no interpretar arbitrariamente cualquier integer como ordinal.

---

# 97. LIMIT

Un limit aplicado al resultado:

```text
(A UNION B)
LIMIT 10
```

es distinto de:

```text
(A LIMIT 10)
UNION
B
```

---

# 98. Structural boundary

Ambos casos deberán tener ASTs diferentes.

---

# 99. SetResultModifiers

Podrá existir:

```php
final readonly class SetResultModifiers
{
    public function __construct(
        public OrderByList $ordering,
        public PaginationSpecification $pagination,
    ) {}
}
```

---

# 100. Operand modifiers

Cada operand conservará sus propios modifiers.

---

# 101. No modifier leakage

```text
ORDER BY
LIMIT
OFFSET
```

de un child operand no deberán filtrarse al parent set result.

---

# 102. Locking

La interacción entre set operations y:

```text
FOR UPDATE
FOR SHARE
SKIP LOCKED
NOWAIT
```

puede variar por plataforma.

---

# 103. Lock capability

El sistema deberá declarar capabilities como:

```text
QUERY.SET_OPERATION.LOCKING
```

cuando sean necesarias.

---

# 104. Builder no transaction assumption

El Builder no abrirá ni exigirá físicamente una transacción.

Sólo podrá declarar lock intent.

---

# 105. CTE integration

Una CTE podrá contener una set operation.

Ejemplo:

```text
WITH source AS (
    Query A
    UNION ALL
    Query B
)
SELECT ...
FROM source
```

---

# 106. Structural representation

```text
CteDefinition
└── SetOperationQuery
    ├── A
    ├── UNION ALL
    └── B
```

---

# 107. Recursive CTE integration

Las recursive CTEs normalmente dependen fuertemente de operaciones de conjunto.

Conceptualmente:

```text
Anchor
UNION ALL
Recursive Member
```

---

# 108. Recursive set role

El sistema de CTEs deberá reutilizar:

```text
SetOperationSpecification
```

en vez de implementar un segundo sistema de `UNION`.

---

# 109. One semantic set engine

Regla:

```text
Recursive CTE UNION
=
same Set Operation semantics
used elsewhere
```

con restricciones adicionales de recursion context.

---

# 110. Subquery integration

Una set operation podrá utilizarse como:

```text
scalar subquery
table subquery
derived table
CTE body
EXISTS body
IN subquery
```

si su output/context es compatible.

---

# 111. Example derived set

```php
$people = $users
    ->unionAll($customers);

$query = DB::query()
    ->fromSub($people, 'people')
    ->select('people.email');
```

---

# 112. Derived output

Semantic Analysis producirá:

```text
people
└── email
    ├── users.email
    └── customers.email
```

como lineage.

---

# 113. Scope model

Cada operand mantiene su propio query scope.

```text
Set Scope S0
├── Operand A Scope S1
└── Operand B Scope S2
```

---

# 114. No sibling visibility

Por default:

```text
S1 symbols
```

no son visibles desde:

```text
S2
```

simplemente porque ambos participan en la misma operación.

---

# 115. Set result scope

Después de reconciliar outputs se crea:

```text
Set Result Scope
```

o representación semántica equivalente.

---

# 116. Outer consumers

Un query externo deberá ver únicamente:

```text
SetOperationOutputRelation
```

y no los símbolos internos de A/B.

---

# 117. Encapsulation

```text
Outer Query
    │
    ▼
Set Result Relation
```

no:

```text
Outer Query
├── Operand A internals
└── Operand B internals
```

---

# 118. Parameter composition

Cada operand puede tener parameters.

Ejemplo:

```text
A → P1
B → P1
```

Estos IDs locales no deberán colisionar al combinar artifacts.

---

# 119. Parameter remapping

Se reutilizará el mecanismo establecido en:

```text
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
```

---

# 120. Example

Antes:

```text
A.P1 = "active"
B.P1 = "verified"
```

Después de composición:

```text
SET.P1 = "active"
SET.P2 = "verified"
```

o identities opacas equivalentes.

---

# 121. Physical placeholders

Todavía no existirán:

```text
?
$1
:foo
```

como decisión física.

---

# 122. Compiler ownership

Los placeholders finales serán asignados por el Compiler.

---

# 123. BindingSet

Los valores seguirán externos al AST:

```text
SetOperationQueryArtifact
+
BindingSet
```

---

# 124. Sensitive values

Sensitivity metadata deberá preservarse a través de todos los operands.

---

# 125. Dependencies

Las dependencias del set result serán la unión semántica de dependencias de sus operands, más requirements propios.

Conceptualmente:

```text
Deps(Set)
=
Deps(A)
∪
Deps(B)
∪
SetRequirements
```

---

# 126. Dependency identity

No se deberán duplicar dependencies sólo porque aparecen en ambos operands.

Podrá utilizarse un `QueryDependencySet` canónico.

---

# 127. Dependency ≠ lineage

Aunque:

```text
users.email
```

sea dependency y lineage source, ambos conceptos siguen separados.

---

# 128. Constraint analysis

Constraint Analysis podrá derivar facts sobre el output.

---

# 129. UNION ALL constraints

Un constraint sólo será globalmente válido si permanece cierto para todos los operands relevantes.

---

# 130. Example

Si:

```text
A.status = "active"
B.status = "active"
```

podrá derivarse:

```text
Set.status = "active"
```

si lineage/type semantics lo permiten.

---

# 131. Non-common fact

Si:

```text
A.status = "active"
B.status = "pending"
```

no deberá derivarse:

```text
Set.status = "active"
```

---

# 132. Common facts

La intersección lógica de facts puede contribuir a:

```text
SetOutputFacts
```

---

# 133. Uniqueness

No deberá asumirse que una unique key de cada operand permanece unique tras:

```text
UNION ALL
```

---

# 134. Example

A:

```text
id = 1
```

B:

```text
id = 1
```

individualmente pueden ser unique.

El resultado de `UNION ALL` contiene dos filas con `id = 1`.

---

# 135. UNION DISTINCT

La fila completa puede ser distinct, pero eso no implica:

```text
single column uniqueness
```

---

# 136. Key preservation

Set operations requieren análisis explícito de:

```text
KeyPreservationFact
```

---

# 137. Cardinality

Constraint Analysis podrá derivar bounds cualitativos.

Ejemplo:

```text
cardinality(A UNION ALL B)
=
cardinality(A) + cardinality(B)
```

conceptualmente, cuando se conocen bounds adecuados.

---

# 138. Exact cardinality

No deberá inferirse sin pruebas suficientes.

---

# 139. UNION DISTINCT cardinality

Podrá derivarse:

```text
max cardinality
≤
cardinality(A) + cardinality(B)
```

sin conocer necesariamente el valor exacto.

---

# 140. INTERSECT cardinality

Podrá derivarse un upper bound relacionado con ambos operands.

---

# 141. EXCEPT cardinality

El resultado no podrá superar la cardinalidad de su left operand.

---

# 142. Semantic facts ≠ cost estimates

Estos son:

```text
logical cardinality constraints
```

no:

```text
estimated row counts
```

El segundo pertenece al Planner/Cost Model.

---

# 143. Semantic Query Graph

Cada set operation deberá aparecer explícitamente.

---

# 144. Example graph

```text
SetNode SET1
├── LEFT_OPERAND → Q1
├── RIGHT_OPERAND → Q2
├── OPERATOR → UNION
├── OUTPUT → R3
└── REQUIRES_CAPABILITY → C1
```

---

# 145. Graph nodes

Podrán existir:

```text
SET_OPERATION
SET_OPERAND
SET_OUTPUT_RELATION
SET_OUTPUT_COLUMN
SET_COERCION
```

o equivalentes.

---

# 146. Graph edges

Ejemplos:

```text
LEFT_OPERAND
RIGHT_OPERAND
PRODUCES
DERIVES_FROM
REQUIRES_COERCION
REQUIRES_CAPABILITY
DEPENDS_ON
```

---

# 147. Lineage example

```text
SetOutput.email
├── DERIVES_FROM users.email
└── DERIVES_FROM customers.email
```

---

# 148. Semantic artifact

El resultado deberá formar parte del:

```text
SemanticQueryArtifact
```

y no requerir reinterpretación posterior.

---

# 149. No compiler re-resolution

Compiler no deberá volver a decidir:

```text
column compatibility
type compatibility
output naming
scope visibility
semantic operator meaning
```

---

# 150. Capability model

Capacidades propuestas:

```text
QUERY.SET.UNION
QUERY.SET.UNION_ALL
QUERY.SET.INTERSECT
QUERY.SET.INTERSECT_ALL
QUERY.SET.EXCEPT
QUERY.SET.EXCEPT_ALL
QUERY.SET.NESTED
QUERY.SET.OPERAND_ORDERING
QUERY.SET.OPERAND_LIMIT
QUERY.SET.RESULT_ORDERING
QUERY.SET.RESULT_LIMIT
QUERY.SET.LOCKING
QUERY.SET.RECURSIVE_CTE
```

---

# 151. Capability descriptors

Una capability podrá indicar:

```text
NATIVE
EMULATABLE
UNSUPPORTED
RESTRICTED
```

---

# 152. Native support

Si una plataforma soporta nativamente la operación:

```text
Compiler
→ native syntax
```

---

# 153. Emulation

Si no existe soporte nativo, VoltStack podrá considerar emulación sólo cuando:

```text
semantic equivalence can be guaranteed
```

---

# 154. No automatic unsafe emulation

Prohibido:

```text
unsupported INTERSECT
→ arbitrary JOIN
```

sin probar:

```text
duplicate semantics
NULL semantics
type semantics
collation semantics
```

---

# 155. Emulation ownership

La decisión deberá estar en:

```text
Optimizer / Planner / Compiler capability strategy
```

no en Builder.

---

# 156. INTERSECT emulation

Una posible transformación puede involucrar:

```text
semi-join
deduplication
```

pero sólo cuando preserve exactamente la semántica requerida.

---

# 157. EXCEPT emulation

Puede involucrar:

```text
anti-join
deduplication
```

con las mismas precauciones.

---

# 158. ALL emulation complexity

Las variantes:

```text
INTERSECT ALL
EXCEPT ALL
```

requieren preservar multiplicidades y pueden necesitar estrategias significativamente más complejas.

---

# 159. No fake ALL

VoltStack no degradará:

```text
INTERSECT ALL
```

a:

```text
INTERSECT
```

silenciosamente.

---

# 160. Requirement failure

Si una feature requerida no puede ejecutarse ni emularse exactamente:

```text
UnsupportedQueryCapabilityException
```

---

# 161. Portability

El sistema deberá contribuir al:

```text
QueryPortabilityProfile
```

---

# 162. Portable baseline

Normalmente:

```text
UNION
UNION ALL
```

tendrán alta portabilidad.

---

# 163. Feature-specific portability

Variantes avanzadas pueden producir:

```text
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
```

---

# 164. Raw operands

Un raw query operand deberá ser explícito.

---

# 165. Raw barrier

Un raw operand puede impedir:

```text
output resolution
type inference
constraint analysis
lineage analysis
optimization
portability analysis
```

parcialmente.

---

# 166. Raw does not grant compatibility

VoltStack no asumirá:

```text
raw operand
=
compatible operand
```

---

# 167. Explicit output contract

Para raw queries avanzadas podría requerirse:

```text
DeclaredOutputContract
```

---

# 168. Declared output

Ejemplo conceptual:

```php
RawQuery::from($sql)
    ->output([
        'id' => QueryType::integer(),
        'name' => QueryType::string(),
    ]);
```

Esto no convierte el raw SQL en semánticamente analizado; sólo aporta un contrato declarado.

---

# 169. Trust

El contrato raw deberá tener:

```text
DECLARED
```

como provenance/certainty, no `PROVEN`.

---

# 170. Security

Valores de operands continuarán parameterizados.

---

# 171. Dynamic operator safety

El desarrollador no deberá pasar directamente:

```php
$builder->setOperation($_GET['operator'], $query);
```

con strings arbitrarios.

---

# 172. Typed operators

Se utilizarán:

```text
SetOperator
SetQuantifier
```

o métodos explícitos.

---

# 173. Identifier safety

Los identifiers internos de cada operand continuarán sujetos al Identifier System.

---

# 174. Security policy operands

Una policy puede introducir operands estructurados, pero deberá hacerlo antes de completar Semantic Analysis.

---

# 175. Provenance

Cada operand podrá registrar:

```text
APPLICATION
ORM
POLICY
TENANT_POLICY
SECURITY_POLICY
EXTENSION
FRAMEWORK
```

como provenance.

---

# 176. Security barrier

Determinados operands podrán impedir transformations peligrosas mediante:

```text
SecurityBarrier
```

---

# 177. Optimization architecture

Optimizer podrá aplicar reglas como:

```text
flatten compatible UNION ALL
remove provably empty operand
deduplicate identical safe branches
push predicates
push projections
simplify nested set operations
rewrite supported set operations
```

siempre que exista prueba de equivalencia.

---

# 178. Empty operand

Si Constraint Analysis demuestra:

```text
Operand B = ∅
```

podrán existir reglas como:

```text
A UNION ALL ∅ → A
```

---

# 179. EXCEPT empty

```text
A EXCEPT ∅ → A
```

cuando sea semánticamente válido.

---

# 180. INTERSECT empty

```text
A INTERSECT ∅ → ∅
```

si la emptiness está probada.

---

# 181. Proof required

El Optimizer no deberá basar estas transformaciones en heurísticas.

---

# 182. Predicate pushdown

Un predicate aplicado al set result puede potencialmente empujarse a operands.

Ejemplo conceptual:

```text
FILTER(status = active)
        │
        ▼
    A UNION ALL B
```

puede convertirse en:

```text
FILTER(A)
UNION ALL
FILTER(B)
```

cuando:

```text
output mapping
types
collations
volatility
security barriers
```

lo permitan.

---

# 183. Projection pushdown

También podrá reducir columnas de operands si el output consumidor no las necesita.

---

# 184. Deduplication placement

Optimizer/Planner podrán razonar sobre dónde realizar deduplicación.

Builder no.

---

# 185. Physical plan possibilities

Planner podría representar:

```text
Append
HashDistinct
SortDistinct
HashIntersect
MergeIntersect
HashExcept
MergeExcept
MultisetCounter
```

u otras operaciones físicas.

---

# 186. Logical operator ≠ physical operator

```text
UNION DISTINCT
≠
HashDistinct
```

El primero es semántica.

El segundo es estrategia física.

---

# 187. Planner cost model

Podrá considerar:

```text
estimated rows
row width
available memory
ordering
indexes
parallelism
spill cost
duplicate ratio
platform capabilities
```

---

# 188. Compiler

Compiler recibirá una representación ya validada y planificada.

---

# 189. Compiler responsibilities

Compiler decidirá:

```text
target syntax
required parentheses
physical cast syntax
operator spelling
ALL syntax
ordering syntax
limit syntax
emulation syntax if planned
```

---

# 190. Compiler non-responsibilities

No deberá:

```text
guess output types
guess column count
resolve schema
decide semantic compatibility
invent coercions
choose optimization rules
```

---

# 191. Query Builder API

Una API base podría ofrecer:

```php
union()
unionAll()
intersect()
intersectAll()
except()
exceptAll()
```

---

# 192. Generic advanced API

También:

```php
setOperation(
    SetOperator $operator,
    QueryBuilder|QueryBuildArtifact $query,
    SetQuantifier $quantifier = SetQuantifier::DISTINCT,
)
```

---

# 193. Prefer typed API

Los shortcuts deberán delegar al mismo core.

```text
union()
        ┐
unionAll()
        │
intersect()
        ├── SetOperationBuilder
except()
        │
...
        ┘
```

---

# 194. No duplicate engines

No deberán existir implementaciones separadas para:

```text
UNION Builder
INTERSECT Builder
EXCEPT Builder
```

que dupliquen la arquitectura.

---

# 195. SetOperationBuilder

Propuesta:

```php
final class SetOperationBuilder
{
    public function combine(
        QueryBuildArtifact $left,
        SetOperationSpecification $operation,
        QueryBuildArtifact $right,
    ): SetOperationQueryBuilder;
}
```

---

# 196. SetOperationBuilderState

Podrá contener:

```text
left operand
right operand
operator
quantifier
result ordering
result pagination
metadata
parameter composition state
extension state
```

---

# 197. Finalization

```text
SetOperationBuilderState
        │
        ▼
SetOperationFinalizer
        │
        ▼
Immutable SetOperationQueryModel
        │
        ▼
QueryBuildArtifact
```

---

# 198. Finalizer responsibilities

Deberá:

```text
freeze operands
validate construction state
compose parameter definitions
compose bindings
freeze metadata
construct immutable model
```

---

# 199. Finalizer shall not

```text
resolve schema
infer final types
perform optimizer rewrites
compile SQL
execute query
```

---

# 200. Call order

Una vez construida la set operation, modifiers externos podrán aplicarse al resultado:

```php
$a
    ->unionAll($b)
    ->orderBy('email')
    ->limit(100);
```

---

# 201. Derived terminal helpers

Operaciones como:

```text
first()
exists()
count()
paginate()
cursor()
```

deberán operar sobre el set result completo, salvo que el desarrollador las aplique explícitamente a un operand antes de combinarlo.

---

# 202. first()

Conceptualmente:

```text
SetQuery
+
TerminalOverlay(limit = 1)
```

sin mutar destructivamente el builder original.

---

# 203. count()

No deberá asumir:

```text
count(A) + count(B)
```

porque esto es incorrecto para operaciones con distinct/intersection/difference.

---

# 204. count result

La operación deberá contar el:

```text
SetOperationResult
```

mediante el Query Engine correspondiente.

---

# 205. exists()

Igualmente:

```text
exists(set query)
```

representará una pregunta sobre el resultado total.

Optimizer podrá elegir una estrategia eficiente.

---

# 206. Pagination

Pagination se aplicará sobre:

```text
SetOperationOutputRelation
```

y deberá integrarse posteriormente con:

```text
199_DATABASE_PAGINATION_SYSTEM.md
200_DATABASE_CURSOR_PAGINATION_SYSTEM.md
```

---

# 207. Cursor pagination

Cursor pagination sobre set operations puede requerir:

```text
stable ordering
unique ordering key
output lineage
comparable output types
```

y no deberá asumirse siempre posible.

---

# 208. Persistent runtime

El sistema deberá funcionar correctamente en workers persistentes.

---

# 209. Shared state

Podrá compartirse:

```text
immutable operator descriptors
frozen capability descriptors
frozen extension registries
stateless factories
immutable cached artifacts
```

---

# 210. Operation-local state

Deberá ser local:

```text
SetOperationBuilderState
ParameterRemapper
DiagnosticCollector
Type reconciliation work state
Constraint work state
Optimization work state
```

---

# 211. No global current set

Prohibido:

```php
static $currentSetOperation;
```

---

# 212. No global operand stack

Prohibido:

```php
static array $operandStack;
```

---

# 213. No global parameter counter

Se mantiene la regla:

```text
parameter identity
must be query-scoped
```

---

# 214. Serialization

Artifacts serializables no deberán contener:

```text
PDO
Connection
Statement
Transaction
Closure
Generator
Fiber
Request
User
Tenant
EntityManager
UnitOfWork
Telemetry Span
```

---

# 215. Extension system

Podrán existir:

```text
custom set operators
custom quantifiers
custom output reconciliation rules
custom optimization rules
custom capability descriptors
```

---

# 216. Extension contract

Una extensión que introduzca un nuevo operador deberá declarar soporte para las fases que corresponda.

---

# 217. Example descriptor

```text
SetOperationExtensionDescriptor
├── id
├── version
├── semantic operator
├── quantifier support
├── validation handler
├── type reconciliation handler
├── constraint handler
├── optimizer support
├── planner support
├── compiler support
├── capability requirements
├── portability
└── fingerprint contribution
```

---

# 218. No partial invisible support

Una extensión no deberá registrar sólo:

```text
compiler syntax
```

si su operación requiere semántica desconocida por el Query Engine.

---

# 219. Frozen registry

```text
SetOperationExtensionRegistry
```

deberá congelarse después del bootstrap.

---

# 220. Fingerprint

El structural fingerprint deberá incluir:

```text
operand fingerprints
operator
quantifier
grouping
result modifiers
metadata affecting semantics
extension identity/version
```

---

# 221. Semantic fingerprint

Añadirá:

```text
resolved output types
nullability
coercions
lineage
constraints
capabilities
schema identities
semantic extension versions
```

---

# 222. Bindings excluded

Los valores runtime ordinarios no deberán formar parte del semantic fingerprint.

---

# 223. Query cache

Dos set queries sólo podrán compartir artifact/cache cuando sus fingerprints y contextos relevantes sean compatibles.

---

# 224. Budgets

Deberán existir límites para:

```text
set operation count
set tree depth
operand count
total projection width
parameter count
dependency count
lineage edges
type reconciliation work
optimizer rewrite count
extension nodes
```

---

# 225. SetOperationBudget

Conceptualmente:

```php
final readonly class SetOperationBudget
{
    public function __construct(
        public int $maxOperations,
        public int $maxTreeDepth,
        public int $maxOperands,
        public int $maxOutputWidth,
        public int $maxLineageEdges,
    ) {}
}
```

---

# 226. No silent truncation

Si se excede el budget:

```text
SetOperationBudgetExceededException
```

---

# 227. Diagnostics

Los errores deberán proporcionar:

```text
SetOperationId
operator
operand index
output position
expected type
actual type
scope
source location
capability
provenance
```

cuando sea posible.

---

# 228. Arity diagnostic

Ejemplo:

```text
Set operation SET3 has incompatible operand widths.

Left operand:
    2 columns

Right operand:
    3 columns

Operator:
    UNION ALL
```

---

# 229. Type diagnostic

```text
Set output column #2 cannot reconcile operand types.

Left:
    UserId

Right:
    OrderId

No implicit semantic coercion is available.
```

---

# 230. Capability diagnostic

```text
INTERSECT ALL is required by SET5,
but the target platform does not provide
a native or semantics-preserving emulation
strategy.
```

---

# 231. Testing strategy

Deberán existir tests para:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
nested set operations
grouping
precedence
arity compatibility
wildcard outputs
type reconciliation
domain types
NULL inference
nullability
collations
output naming
lineage
ordering
limit
offset
operand modifiers
result modifiers
CTE integration
recursive CTE integration
subquery integration
parameter remapping
constraints
capabilities
portability
raw operands
security barriers
extensions
budgets
serialization
persistent workers
concurrency
```

---

# 232. Architecture tests

Deberán impedir imports desde Set Operation Builder hacia:

```text
PDO
Driver
Connection
ConnectionManager
Statement
Executor
SQL Compiler
ORM UnitOfWork
```

---

# 233. Builder tests

Builder tests deberán comparar:

```text
SetOperationQueryModel
QueryBuildArtifact
ParameterDefinitionSet
BindingSet
QueryMetadata
```

no SQL.

---

# 234. Semantic tests

Deberán comprobar:

```text
operand scopes
output relation
column mappings
types
nullability
lineage
constraints
dependencies
capabilities
SemanticQueryGraph
```

---

# 235. Optimizer tests

Deberán verificar transformaciones como:

```text
UNION ALL flattening
empty operand elimination
predicate pushdown
projection pushdown
set operation rewrites
```

con pruebas negativas para casos no seguros.

---

# 236. Compiler tests

Sólo aquí deberán comprobarse strings como:

```text
UNION
UNION ALL
INTERSECT
EXCEPT
(...)
```

---

# 237. Persistent worker tests

Después de miles de operaciones deberá demostrarse:

```text
no leaked operands
no leaked ordering
no leaked parameters
no leaked scopes
no leaked output mappings
no leaked diagnostics
```

---

# 238. Complexity

La construcción estructural básica deberá aproximarse a:

```text
O(number of set nodes)
```

---

# 239. Output reconciliation

Para:

```text
N operands
×
M output columns
```

la reconciliación básica deberá aproximarse a:

```text
O(N × M)
```

sin contar complejidad adicional del Type System.

---

# 240. Lineage

Lineage puede crecer proporcionalmente a:

```text
operands × output width
```

por lo que deberá respetar budgets.

---

# 241. Directory structure propuesta

```text
Query/
└── Builder/
    └── SetOperation/
        ├── Contract/
        │   ├── SetOperationBuilderInterface.php
        │   └── SetOperationFinalizerInterface.php
        │
        ├── Core/
        │   ├── SetOperationId.php
        │   ├── SetOperator.php
        │   ├── SetQuantifier.php
        │   ├── SetOperationSpecification.php
        │   ├── SetOperand.php
        │   ├── SetOperationQueryModel.php
        │   ├── SetOperationBuilder.php
        │   ├── SetOperationBuilderState.php
        │   └── SetOperationFinalizer.php
        │
        ├── Grouping/
        │   ├── SetGroup.php
        │   └── SetOperationTree.php
        │
        ├── Output/
        │   ├── SetOperationOutputRelation.php
        │   ├── SetOperationOutputColumn.php
        │   └── SetOutputReconciler.php
        │
        ├── Type/
        │   ├── SetTypeReconciler.php
        │   └── SetCoercionRequirement.php
        │
        ├── Modifier/
        │   └── SetResultModifiers.php
        │
        ├── Parameter/
        │   └── SetParameterComposer.php
        │
        ├── Capability/
        │   └── SetOperationCapabilityDescriptor.php
        │
        ├── Extension/
        │   ├── SetOperationExtension.php
        │   ├── SetOperationExtensionDescriptor.php
        │   └── SetOperationExtensionRegistry.php
        │
        ├── Budget/
        │   └── SetOperationBudget.php
        │
        ├── Diagnostic/
        │   └── SetOperationDiagnostic.php
        │
        └── Exception/
            ├── SetOperationException.php
            ├── InvalidSetOperandException.php
            ├── SetArityMismatchException.php
            ├── SetTypeMismatchException.php
            ├── UnsupportedSetOperationException.php
            └── SetOperationBudgetExceededException.php
```

---

# 242. Semantic components

La información semántica especializada podrá residir en:

```text
Query/
└── Semantic/
    └── SetOperation/
        ├── SetOperationSemanticInfo.php
        ├── SetOperationSemanticTable.php
        ├── SetOutputMapping.php
        ├── SetOutputMappingTable.php
        └── SetOperationSemanticAnalyzer.php
```

sin duplicar los sistemas globales de:

```text
Type
Constraint
Relation
Lineage
Dependency
Graph
```

---

# 243. Architectural invariants

## DB-SET-001

Set Operation no será concatenación SQL.

## DB-SET-002

Builder no generará SQL.

## DB-SET-003

Builder no ejecutará operands.

## DB-SET-004

Operands serán immutable query artifacts.

## DB-SET-005

Mutable child builders no sobrevivirán composition.

## DB-SET-006

Cada set operation tendrá identidad explícita.

## DB-SET-007

Set operator será estructurado.

## DB-SET-008

Quantifier será estructurado.

## DB-SET-009

UNION DISTINCT y UNION ALL conservarán semánticas diferentes.

## DB-SET-010

INTERSECT y INTERSECT ALL conservarán semánticas diferentes.

## DB-SET-011

EXCEPT y EXCEPT ALL conservarán semánticas diferentes.

## DB-SET-012

ALL nunca será degradado silenciosamente a DISTINCT.

## DB-SET-013

DISTINCT no será implementado por Builder.

## DB-SET-014

Set operations se representarán como árboles estructurados.

## DB-SET-015

Grouping será explícito.

## DB-SET-016

Semantic grouping no dependerá del SQL textual.

## DB-SET-017

Builder preservará estructura declarada.

## DB-SET-018

Builder no reordenará operands.

## DB-SET-019

Builder no aplicará associativity optimizations.

## DB-SET-020

Builder no aplicará commutativity optimizations.

## DB-SET-021

Output compatibility será posicional.

## DB-SET-022

Output compatibility no dependerá de igualdad de nombres.

## DB-SET-023

Operand widths deberán reconciliarse.

## DB-SET-024

Wildcard width no será adivinada por Builder.

## DB-SET-025

Schema-aware output resolution no realizará hidden I/O.

## DB-SET-026

Set result tendrá RelationId propio.

## DB-SET-027

Set output columns tendrán identities propias.

## DB-SET-028

Operand output identity no será reutilizada como set output identity.

## DB-SET-029

Output lineage será explícita.

## DB-SET-030

Una output column podrá tener múltiples lineage sources.

## DB-SET-031

Output naming será determinista.

## DB-SET-032

Leftmost output naming será la política V1 por default.

## DB-SET-033

Type reconciliation utilizará Query Type System.

## DB-SET-034

PHP coercion no determinará set compatibility.

## DB-SET-035

Domain identity será preservada cuando corresponda.

## DB-SET-036

Tipos físicos iguales no implicarán domain types iguales.

## DB-SET-037

Unknown será distinto de Any.

## DB-SET-038

NULL literals podrán recibir type constraints.

## DB-SET-039

Nullability será reconciliada formalmente.

## DB-SET-040

Builder no insertará arbitrary casts.

## DB-SET-041

Required coercions serán explícitas.

## DB-SET-042

Compiler será responsable de physical cast syntax.

## DB-SET-043

Duplicate semantics pertenecerán al operador/quantifier.

## DB-SET-044

Database equality no será sustituida por PHP equality.

## DB-SET-045

Collation será considerada cuando sea relevante.

## DB-SET-046

Operand ORDER BY será distinto de result ORDER BY.

## DB-SET-047

Operand LIMIT será distinto de result LIMIT.

## DB-SET-048

Operand OFFSET será distinto de result OFFSET.

## DB-SET-049

Result modifiers tendrán scope explícito.

## DB-SET-050

Modifiers no se filtrarán entre scopes.

## DB-SET-051

Locking será capability-driven.

## DB-SET-052

Builder no abrirá transacciones.

## DB-SET-053

CTEs reutilizarán el mismo Set Operation System.

## DB-SET-054

Recursive CTE no implementará un segundo UNION engine.

## DB-SET-055

Set operations podrán ser subqueries.

## DB-SET-056

Set operations podrán ser derived tables.

## DB-SET-057

Set operations podrán ser CTE bodies.

## DB-SET-058

Cada operand tendrá scope propio.

## DB-SET-059

Sibling operands no compartirán símbolos implícitamente.

## DB-SET-060

Outer consumers sólo verán el set output relation.

## DB-SET-061

Operand internals estarán encapsulados.

## DB-SET-062

Parameter identities no colisionarán.

## DB-SET-063

No habrá global parameter allocator.

## DB-SET-064

Physical placeholders pertenecerán al Compiler.

## DB-SET-065

Binding values permanecerán fuera del AST.

## DB-SET-066

Sensitive metadata será preservada.

## DB-SET-067

Set dependencies serán explícitas.

## DB-SET-068

Dependency será distinta de lineage.

## DB-SET-069

Constraint Analysis será scope-aware.

## DB-SET-070

UNION ALL no preservará uniqueness automáticamente.

## DB-SET-071

UNION DISTINCT no implicará column uniqueness.

## DB-SET-072

Key preservation requerirá prueba.

## DB-SET-073

Logical cardinality será distinta de estimated cardinality.

## DB-SET-074

SemanticQueryGraph representará set operations explícitamente.

## DB-SET-075

Compiler no reinterpretará semantic compatibility.

## DB-SET-076

Capabilities reemplazarán vendor conditionals.

## DB-SET-077

Unsupported required operations fallarán explícitamente.

## DB-SET-078

Emulation deberá preservar semántica exactamente.

## DB-SET-079

Unsafe emulation estará prohibida.

## DB-SET-080

INTERSECT ALL no será degradado.

## DB-SET-081

EXCEPT ALL no será degradado.

## DB-SET-082

Portability será explícita.

## DB-SET-083

Raw operands serán explícitos.

## DB-SET-084

Raw no implicará semantic compatibility.

## DB-SET-085

Raw output contracts serán declarados, no probados.

## DB-SET-086

Operators públicos serán typed.

## DB-SET-087

Strings arbitrarios no se convertirán en set operators.

## DB-SET-088

Policy operands tendrán provenance.

## DB-SET-089

Security barriers serán respetadas.

## DB-SET-090

Optimizer será responsable de algebraic transformations.

## DB-SET-091

Empty-operand elimination requerirá prueba.

## DB-SET-092

Predicate pushdown requerirá equivalencia demostrable.

## DB-SET-093

Projection pushdown requerirá equivalencia demostrable.

## DB-SET-094

Logical set operator será distinto de physical operator.

## DB-SET-095

Planner elegirá physical algorithms.

## DB-SET-096

Compiler decidirá target syntax.

## DB-SET-097

Builder shortcuts reutilizarán un único core.

## DB-SET-098

No existirán engines duplicados por operador.

## DB-SET-099

Finalizer no realizará Semantic Analysis.

## DB-SET-100

Terminal helpers no mutarán destructivamente el builder.

## DB-SET-101

count() contará el resultado completo.

## DB-SET-102

exists() consultará el resultado completo.

## DB-SET-103

Pagination se aplicará al scope correspondiente.

## DB-SET-104

Cursor pagination requerirá ordering adecuado.

## DB-SET-105

Builder state será operation-scoped.

## DB-SET-106

No habrá global current set operation.

## DB-SET-107

No habrá global operand stack.

## DB-SET-108

Artifacts no retendrán runtime resources.

## DB-SET-109

Extension registries serán frozen.

## DB-SET-110

Extensions deberán declarar semantic support.

## DB-SET-111

Compiler-only unknown extensions estarán prohibidas cuando requieran semántica.

## DB-SET-112

Structural fingerprints incluirán operator y grouping.

## DB-SET-113

Runtime binding values no formarán parte del semantic fingerprint.

## DB-SET-114

Budgets serán explícitos.

## DB-SET-115

Budget exhaustion no truncará la query.

## DB-SET-116

Construction será determinista.

## DB-SET-117

Semantic reconciliation será determinista.

## DB-SET-118

Persistent workers no compartirán mutable set state.

## DB-SET-119

Concurrent set queries permanecerán aisladas.

## DB-SET-120

ORM reutilizará este mismo Set Operation System.

---

# 244. Anti-patterns

## 244.1 SQL concatenation

```php
$sql = $queryA . ' UNION ' . $queryB;
```

**Rechazado.**

---

## 244.2 Mutable operands

```text
SetOperation
├── Builder A
└── Builder B
```

**Rechazado.**

---

## 244.3 Compare columns by name

```text
A.id must match B.id
```

**Incorrecto.**

La compatibilidad es posicional.

---

## 244.4 Guess wildcard width

```text
SELECT *
→ assume N columns
```

**Rechazado.**

---

## 244.5 PHP type coercion

```php
(string) $value;
```

como mecanismo de reconciliación semántica.

**Rechazado.**

---

## 244.6 Flatten every UNION

```text
(A UNION B) UNION C
→ A UNION B UNION C
```

sin analizar boundaries.

**Rechazado.**

---

## 244.7 Reorder EXCEPT

```text
A EXCEPT B
→ B EXCEPT A
```

**Rechazado.**

---

## 244.8 Ignore ALL

```text
INTERSECT ALL
→ INTERSECT
```

**Rechazado.**

---

## 244.9 Fake unsupported operator

```text
INTERSECT
→ INNER JOIN
```

sin prueba semántica.

**Rechazado.**

---

## 244.10 Result ORDER BY attached to last operand

```text
A UNION B ORDER BY x
```

modelado como:

```text
A UNION (B ORDER BY x)
```

**Rechazado.**

---

## 244.11 Global parameter allocator

**Rechazado.**

---

## 244.12 Compiler performs semantic reconciliation

**Rechazado.**

---

# 245. Ejemplo integral

Supongamos:

```php
$customers = DB::table('customers')
    ->select(
        'id',
        'email',
        'created_at'
    )
    ->where('active', true);

$leads = DB::table('leads')
    ->select(
        'id',
        'email',
        'created_at'
    )
    ->where('converted', false);

$archived = DB::table('archived_contacts')
    ->select(
        'contact_id',
        'email_address',
        'created_at'
    );

$query = $customers
    ->unionAll($leads)
    ->union($archived)
    ->orderBy('created_at', 'desc')
    ->limit(100);
```

---

# 246. Builder structure

Primero:

```text
SET1
├── customers
├── UNION
│   └── ALL
└── leads
```

Después:

```text
SET2
├── SET1
├── UNION
│   └── DISTINCT
└── archived
```

Finalmente:

```text
SET2
├── Result ORDER BY created_at DESC
└── Result LIMIT 100
```

---

# 247. Structural tree

```text
                 SET2
              UNION DISTINCT
              /            \
           SET1          archived
        UNION ALL
        /       \
 customers      leads
```

---

# 248. Parameter composition

Antes:

```text
customers.P1 = true
leads.P1     = false
```

Después:

```text
SET2.P1 = true
SET2.P2 = false
```

---

# 249. Semantic output reconciliation

Position 1:

```text
customers.id
leads.id
archived.contact_id
        │
        ▼
SetOutputColumn #1
name = id
```

Position 2:

```text
customers.email
leads.email
archived.email_address
        │
        ▼
SetOutputColumn #2
name = email
```

Position 3:

```text
customers.created_at
leads.created_at
archived.created_at
        │
        ▼
SetOutputColumn #3
name = created_at
```

---

# 250. Output names

La política leftmost produce:

```text
id
email
created_at
```

aunque el tercer operand utilice:

```text
contact_id
email_address
created_at
```

---

# 251. Output lineage

```text
Result.id
├── customers.id
├── leads.id
└── archived_contacts.contact_id

Result.email
├── customers.email
├── leads.email
└── archived_contacts.email_address

Result.created_at
├── customers.created_at
├── leads.created_at
└── archived_contacts.created_at
```

---

# 252. Semantic scopes

```text
Set Result Scope S0
│
├── Operand Scope S1
│   └── customers
│
├── Operand Scope S2
│   └── leads
│
└── Operand Scope S3
    └── archived_contacts
```

---

# 253. Ordering resolution

```text
ORDER BY created_at
```

se resuelve contra:

```text
SetOutputColumn #3
```

no directamente contra:

```text
customers.created_at
```

---

# 254. Optimizer opportunities

Optimizer podrá analizar:

```text
SET2
├── SET1 UNION ALL
└── archived
```

y decidir si existen oportunidades de:

```text
predicate pushdown
projection pushdown
branch pruning
empty branch elimination
deduplication optimization
set flattening
```

---

# 255. Planner

Planner podrá producir conceptualmente:

```text
Limit(100)
    │
Sort(created_at DESC)
    │
Distinct
    │
Append
├── customers
├── leads
└── archived_contacts
```

si dicha estrategia preserva exactamente la semántica.

---

# 256. Important note

Ese physical plan:

```text
Distinct(Append(...))
```

no forma parte del Query Builder.

---

# 257. Arquitectura completa

```text
Developer
    │
    ▼
Select Builders
    │
    ▼
Immutable Operand Queries
    │
    ▼
Set Operation Builder
    │
    ▼
Set Operation Tree
    │
    ▼
Normalization
    │
    ▼
Structural Validation
    │
    ▼
Semantic Analysis
    │
    ├── Operand Scopes
    ├── Output Width
    ├── Output Mapping
    ├── Type Reconciliation
    ├── Nullability
    ├── Lineage
    ├── Dependencies
    └── Capabilities
    │
    ▼
Set Output Relation
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
    ├── Flatten
    ├── Pushdown
    ├── Prune
    ├── Rewrite
    └── Simplify
    │
    ▼
Planner
    │
    ├── Append
    ├── Deduplicate
    ├── Intersect
    └── Difference
    │
    ▼
Compiler
    │
    ▼
Target SQL
```

---

# 258. Fórmula maestra

```text
Set Operation
=
Left Query
+
Set Operator
+
Set Quantifier
+
Right Query
+
Explicit Grouping
```

---

# 259. Fórmula del output

```text
Set Output Relation
=
Positional Operand Mapping
+
Reconciled Types
+
Reconciled Nullability
+
Deterministic Names
+
Multi-Source Lineage
+
Derived Constraints
```

---

# 260. Fórmula de compatibilidad

```text
Set Compatibility
=
Equal Arity
+
Position-Compatible Types
+
Compatible Domain Semantics
+
Compatible Collation Semantics
+
Valid Capability Requirements
```

---

# 261. Fórmula de composición

```text
Set Query Artifact
=
Immutable Set Query Model
+
Composed Parameter Definitions
+
Composed BindingSet
+
QueryMetadata
```

---

# 262. Fórmula de seguridad

```text
Safe Set Composition
=
Immutable Operands
+
Typed Operators
+
Structured Identifiers
+
Parameterized Values
+
Explicit Scope Boundaries
+
Explicit Raw Barriers
+
Security Provenance
+
Resource Budgets
```

---

# 263. Fórmula de optimización

```text
Safe Set Optimization
=
Semantic Set Graph
+
Type Knowledge
+
Constraint Knowledge
+
Lineage
+
Cardinality Facts
+
Volatility
+
Security Barriers
+
Capability Knowledge
+
Cost Information
```

---

# 264. Fórmula de persistent-runtime safety

```text
Persistent-Safe Set Operations
=
Immutable Query Artifacts
+
Operation-Scoped Builder State
+
Operation-Scoped Parameter Mapping
+
Frozen Shared Registries
+
No Global Operand Stack
+
No Global Parameter Counter
+
No Runtime Resource Retention
```

---

# 265. Boundary final

```text
Set Operation Builder
        │
        ▼
"What query results does
the developer want to combine?"
        │
        ▼
Semantic Analysis
        │
        ▼
"Are their outputs compatible,
and what does the resulting
relation mean?"
        │
        ▼
Constraint Analysis
        │
        ▼
"What facts remain true across
the resulting relation?"
        │
        ▼
Optimizer
        │
        ▼
"Can the operation be flattened,
pushed down, pruned or rewritten?"
        │
        ▼
Planner
        │
        ▼
"Which physical strategy should
implement the set semantics?"
        │
        ▼
Compiler
        │
        ▼
"How does the target platform
express that strategy?"
```

---

# 266. Conclusión

`Union and Set Operation System` establece que las operaciones:

```text
UNION
UNION ALL
INTERSECT
INTERSECT ALL
EXCEPT
EXCEPT ALL
```

son operaciones semánticas sobre relaciones y no mecanismos de concatenación SQL.

VoltStack podrá ofrecer una experiencia sencilla:

```php
$query = $users
    ->unionAll($customers)
    ->except($blocked)
    ->orderBy('email');
```

mientras internamente mantiene:

```text
Immutable Query Operands
+
Set Operation Tree
+
Explicit Operator Semantics
+
Explicit Duplicate Semantics
+
Positional Output Mapping
+
Type Reconciliation
+
Domain Preservation
+
Nullability Reconciliation
+
Multi-Source Lineage
+
Constraint Analysis
+
Capability Requirements
+
Optimizer Transformations
+
Physical Planning
+
Dialect Compilation
```

La regla arquitectónica final será:

> **Una operación de conjunto en VoltStack combina relaciones semánticas, no cadenas SQL. Sus columnas se reconcilian por posición, sus tipos y dominios se analizan explícitamente, su output adquiere identidad propia y cualquier reordenamiento, flattening, deduplicación, emulación o estrategia física pertenece a las fases posteriores del Query Engine.**

---

# 267. Estado del bloque Query Builder

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
```

El Query Builder ya dispone de una arquitectura coherente para:

```text
SELECT
INSERT
UPDATE
DELETE
JOIN
Subqueries
CTEs
Recursive CTEs
Set Operations
```

---

# 268. Siguiente documento

```text
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
```

El siguiente documento deberá definir:

```text
Aggregate Expressions
Aggregate Functions
GROUP BY
Grouping Keys
Grouping Semantics
Grouping Sets
ROLLUP
CUBE
HAVING
Aggregate Scope
Aggregate Type Resolution
Aggregate Nullability
DISTINCT Aggregates
Aggregate FILTER
Ordered Aggregates
Aggregate Lineage
Aggregate Constraints
Functional Dependencies
Grouping Legality
Non-Grouped Column Validation
Aggregate Query Output
Aggregate Cardinality
Nested Aggregate Rules
Aggregate Window Boundary
Aggregate Optimization
Partial Aggregation
Aggregation Pushdown
Capability Requirements
Portability
Extensions
Persistent Runtime Safety
```

manteniendo la separación:

```text
Aggregation Builder
        │
        ▼
Structured Aggregate Query
        │
        ▼
Semantic Grouping Analysis
        │
        ▼
Aggregate Output Relation
        │
        ▼
Constraint Analysis
        │
        ▼
Optimizer
        │
        ▼
Planner
        │
        ▼
Compiler
```