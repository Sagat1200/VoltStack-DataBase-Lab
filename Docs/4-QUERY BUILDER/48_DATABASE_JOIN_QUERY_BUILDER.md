# 48_DATABASE_JOIN_QUERY_BUILDER.md

# VoltStack Quantum Database
## Join Query Builder System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 48 — Join Query Builder  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Relation Composition  
**Versión:** 1.0

---

# 1. Propósito

`JoinQueryBuilder` define la infraestructura común utilizada para construir relaciones compuestas mediante JOINs.

Será reutilizable principalmente por:

```text
SELECT
UPDATE
DELETE
```

y potencialmente por extensiones futuras.

La experiencia pública podrá ser:

```php
$query = DB::table('orders as o')
    ->join('users as u', 'u.id', '=', 'o.user_id')
    ->leftJoin('payments as p', 'p.order_id', '=', 'o.id')
    ->where('o.status', 'pending')
    ->select(
        'o.id',
        'u.name',
        'p.status'
    );
```

Pero internamente:

```text
join()
≠
concatenar "JOIN"
```

El flujo será:

```text
Developer API
     │
     ▼
JoinQueryBuilder
     │
     ▼
JoinSpecification
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
Relation & Join Resolution
     │
     ▼
Semantic Join
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
Compiler
```

---

# 2. Regla fundamental

```text
JoinQueryBuilder
≠
Join Resolver
≠
Join Optimizer
≠
Physical Join Planner
≠
SQL Join Compiler
```

El Builder sólo expresa:

```text
which relation participates
+
which logical join type is requested
+
how relations are qualified
+
which predicates constrain matching
+
which dependencies/correlations are declared structurally
```

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
JOIN declaration
≠
semantic relation resolution
≠
relationship metadata
≠
foreign key
≠
physical join algorithm
≠
SQL syntax
```

Por ejemplo:

```text
LEFT JOIN
```

es una propiedad semántica.

No significa:

```text
Nested Loop
Hash Join
Merge Join
```

---

# 4. Relación con documentos anteriores

Este sistema se apoya directamente en:

```text
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
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
```

---

# 5. Objetivos

El sistema deberá proporcionar:

- API Laravel-like.
- representación estructurada de JOINs;
- aliases;
- qualified identifiers;
- INNER JOIN;
- LEFT JOIN;
- RIGHT JOIN;
- FULL JOIN;
- CROSS JOIN;
- LATERAL JOIN;
- subquery joins;
- CTE joins;
- table-function joins;
- nested join structures;
- múltiples predicates ON;
- grouped predicates;
- `USING`;
- correlation declarations;
- parameterización;
- metadata;
- provenance;
- capability requirements;
- portability analysis;
- extensibilidad;
- budgets;
- persistent-runtime safety.

---

# 6. No responsabilidades

`JoinQueryBuilder` no:

```text
abre conexiones
consulta information_schema
resuelve columnas
resuelve foreign keys
descubre relaciones ORM
genera SQL
elige JOIN syntax
elige Hash Join
elige Merge Join
elige Nested Loop
reordena joins
elimina joins
convierte LEFT JOIN en INNER JOIN
ejecuta queries
```

---

# 7. Arquitectura general

```text
                     QueryBuilder
                          │
                          ▼
                  JoinQueryBuilder
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
 JoinTargetBuilder   JoinPredicateBuilder  JoinMetadataBuilder
        │                 │                 │
        ├─────────────────┼─────────────────┤
        │                 │                 │
        ▼                 ▼                 ▼
 JoinSourceBuilder   JoinUsingBuilder   JoinCorrelationBuilder
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
                  JoinBuilderState
                          │
                          ▼
                 JoinFinalizer
                          │
                          ▼
                JoinSpecification
                          │
                          ▼
                    Query AST
```

---

# 8. API básica

```php
DB::table('orders as o')
    ->join(
        'users as u',
        'u.id',
        '=',
        'o.user_id'
    );
```

Representación conceptual:

```text
JoinSpecification
├── type: INNER
├── target
│   └── users AS u
└── predicate
    └── u.id = o.user_id
```

---

# 9. No string SQL

No se almacenará:

```text
"JOIN users u ON u.id = o.user_id"
```

Se almacenará estructura.

---

# 10. JoinSpecification

Modelo conceptual:

```php
final readonly class JoinSpecification
{
    public function __construct(
        public JoinId $id,
        public JoinType $type,
        public JoinSource $source,
        public ?PredicateNode $predicate,
        public ?JoinUsingSpecification $using,
        public JoinCorrelationMode $correlation,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 11. JoinId

Cada JOIN tendrá identidad propia:

```text
J1
J2
J3
...
```

Dos JOINs sobre la misma tabla no serán el mismo JOIN.

---

# 12. Self join

```php
DB::table('employees as e')
    ->leftJoin(
        'employees as manager',
        'manager.id',
        '=',
        'e.manager_id'
    );
```

produce dos relation instances:

```text
employees AS e
employees AS manager
```

---

# 13. Regla

```text
Schema Relation
≠
Query Relation Instance
```

---

# 14. JoinType

V1:

```php
enum JoinType
{
    case INNER;
    case LEFT;
    case RIGHT;
    case FULL;
    case CROSS;
}
```

---

# 15. Join type ≠ syntax

`JoinType::LEFT` significa:

```text
preserve unmatched rows from left input
+
null-extend right output
```

No significa literalmente:

```text
"LEFT JOIN"
```

---

# 16. INNER JOIN

```php
->join('users as u', 'u.id', '=', 'o.user_id')
```

equivale semánticamente a:

```text
JoinType::INNER
```

---

# 17. Explicit innerJoin()

También podrá existir:

```php
->innerJoin(
    'users as u',
    'u.id',
    '=',
    'o.user_id'
)
```

---

# 18. LEFT JOIN

```php
->leftJoin(
    'payments as p',
    'p.order_id',
    '=',
    'orders.id'
)
```

---

# 19. Semántica LEFT

```text
LEFT
├── matched rows
└── unmatched left rows
       │
       ▼
right side becomes null-extended
```

---

# 20. RIGHT JOIN

```php
->rightJoin(...)
```

será representado explícitamente.

---

# 21. No automatic rewrite

El Builder no convertirá:

```text
RIGHT JOIN
```

en:

```text
LEFT JOIN with swapped inputs
```

aunque posteriormente pudiera existir una transformación equivalente.

---

# 22. FULL JOIN

```php
->fullJoin(...)
```

será capability-driven.

---

# 23. FULL JOIN semantics

```text
preserve unmatched left
+
preserve unmatched right
```

---

# 24. CROSS JOIN

```php
->crossJoin('currencies as c')
```

produce:

```text
JoinType::CROSS
```

sin predicate de matching.

---

# 25. CROSS JOIN validation

Por default:

```text
CROSS JOIN + ON
```

será inválido.

---

# 26. JoinConditionMode

Podrá existir:

```php
enum JoinConditionMode
{
    case ON;
    case USING;
    case NONE;
}
```

---

# 27. INNER/OUTER condition

Normalmente:

```text
INNER
LEFT
RIGHT
FULL
```

requerirán:

```text
ON
or
USING
```

salvo extensiones explícitas.

---

# 28. JoinSource

Un JOIN podrá tener diferentes fuentes:

```text
Table
View
CTE
Derived Table
Subquery
Values Relation
Table Function
Extension Relation
```

---

# 29. JoinSourceKind

```php
enum JoinSourceKind
{
    case RELATION;
    case CTE;
    case SUBQUERY;
    case DERIVED_TABLE;
    case VALUES;
    case TABLE_FUNCTION;
    case EXTENSION;
}
```

---

# 30. Physical table join

```php
->join('users as u', ...)
```

produce:

```text
RelationJoinSource
```

---

# 31. View join

Una view sigue siendo una relación resoluble.

El Builder no necesita saber anticipadamente si:

```text
users_view
```

es tabla o view.

---

# 32. Schema Resolution

`38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md` determinará la identidad real.

---

# 33. Subquery join

Ejemplo:

```php
$totals = DB::table('order_items')
    ->select('order_id')
    ->selectRaw('SUM(total) AS total')
    ->groupBy('order_id');

$query = DB::table('orders as o')
    ->leftJoinSub(
        $totals,
        'totals',
        fn ($join) => $join->on(
            'totals.order_id',
            '=',
            'o.id'
        )
    );
```

---

# 34. Internal representation

```text
JoinSpecification
├── type: LEFT
├── source
│   └── DerivedQuerySource
│       ├── query: SelectQueryArtifact
│       └── alias: totals
└── predicate
    └── totals.order_id = o.id
```

---

# 35. Child query finalization

El subquery será finalizado antes de incorporarse:

```text
SelectQueryBuilder
       │
       ▼
SelectQueryArtifact
       │
       ▼
DerivedQuerySource
```

---

# 36. No child mutable builder

Prohibido en artifacts finales:

```text
JoinSpecification
└── SelectQueryBuilder mutable
```

---

# 37. Alias requirement

Derived tables/subqueries podrán requerir alias.

El Builder podrá exigirlo estructuralmente:

```php
->joinSub($query, 'totals', ...)
```

---

# 38. CTE join

```php
->join('active_users as u', ...)
```

puede resolverse posteriormente como:

```text
CTE Symbol
```

si `active_users` existe en el scope correspondiente.

---

# 39. CTE name resolution

El Builder no decidirá:

```text
CTE
vs
physical table
```

a partir del nombre.

Eso pertenece a Symbol Resolution.

---

# 40. VALUES join

Podrá soportarse:

```php
->joinValues(
    rows: [
        [1, 'A'],
        [2, 'B'],
    ],
    alias: 'input',
    columns: ['id', 'code'],
    callback: ...
)
```

---

# 41. VALUES representation

```text
ValuesRelationSource
├── columns
│   ├── id
│   └── code
└── rows
    ├── [P1, P2]
    └── [P3, P4]
```

---

# 42. Values parameterization

Runtime values serán parameters.

No se concatenarán como SQL literals.

---

# 43. Table function join

Podrá existir:

```php
->joinTableFunction(
    function: ...,
    alias: 'items',
    callback: ...
)
```

---

# 44. TableFunctionSource

Se representará semánticamente mediante:

```text
FunctionIdentifier
Arguments
Alias
Output Metadata?
Capability Requirements
```

---

# 45. LATERAL

LATERAL deberá ser una propiedad estructurada.

---

# 46. LateralJoinMode

```php
enum JoinLateralMode
{
    case NONE;
    case LATERAL;
}
```

---

# 47. Ejemplo

```php
->leftJoinLateral(
    $latestPaymentQuery,
    'latest_payment'
)
```

---

# 48. Correlation

Un lateral subquery podrá referenciar columnas de relaciones anteriores.

Ejemplo conceptual:

```text
orders AS o
   │
   ▼
LATERAL subquery
   │
   └── payments.order_id = o.id
```

---

# 49. Correlation ≠ execution strategy

La existencia de correlation no significa automáticamente:

```text
execute subquery once per row
```

El Optimizer podría decorrelacionarla.

---

# 50. Correlation resolution

`Relation & Join Resolution` construirá:

```text
CorrelationDescriptor
├── outerScope
├── outerSymbol
├── innerScope
├── depth
└── provenance
```

---

# 51. Non-lateral correlation

Una referencia ilegal a una relación externa deberá generar error semántico.

---

# 52. Join predicate API

Forma básica:

```php
->join('users as u', function ($join) {
    $join->on('u.id', '=', 'orders.user_id');
});
```

---

# 53. JoinClauseBuilder

Se definirá:

```text
JoinClauseBuilder
```

para construir el predicate de matching.

---

# 54. Ejemplo compuesto

```php
->join('inventory as i', function ($join) {
    $join
        ->on('i.product_id', '=', 'items.product_id')
        ->where('i.warehouse_id', 10)
        ->whereNull('i.deleted_at');
})
```

---

# 55. Important distinction

Dentro de JoinClause:

```text
on()
```

normalmente compara:

```text
expression ↔ expression
```

mientras:

```text
where()
```

normalmente compara:

```text
expression ↔ runtime value
```

---

# 56. on()

```php
$join->on(
    'users.id',
    '=',
    'orders.user_id'
);
```

produce:

```text
ComparisonPredicate
├── ColumnReference(users.id)
├── =
└── ColumnReference(orders.user_id)
```

---

# 57. where()

```php
$join->where(
    'users.status',
    '=',
    'active'
);
```

produce:

```text
ComparisonPredicate
├── ColumnReference(users.status)
├── =
└── ParameterExpression(P1)
```

---

# 58. No ambiguity value/identifier

VoltStack distinguirá explícitamente:

```text
on()
where()
whereColumn()
whereExpr()
```

---

# 59. Parameter bindings

```php
$join->where('users.status', 'active');
```

creará:

```text
ParameterId P1
```

y:

```text
BindingSet
P1 → "active"
```

---

# 60. Parameter scope

Los parameters del JOIN pertenecen a la query completa.

---

# 61. No join-local physical placeholders

No existirá:

```text
join placeholder #1
```

independiente del query parameter model.

---

# 62. Multiple ON predicates

```php
$join
    ->on('a.id', '=', 'b.a_id')
    ->on('a.version', '=', 'b.version');
```

produce:

```text
AND
├── a.id = b.a_id
└── a.version = b.version
```

---

# 63. orOn()

```php
$join
    ->on(...)
    ->orOn(...);
```

deberá preservar agrupación lógica explícita.

---

# 64. Precedence

El Builder no dependerá de precedencia implícita accidental.

---

# 65. Grouped ON

```php
$join->onGroup(function ($on) {
    $on
        ->on('a.id', '=', 'b.a_id')
        ->orOn('a.legacy_id', '=', 'b.a_id');
});
```

produce:

```text
GROUP
└── OR
    ├── a.id = b.a_id
    └── a.legacy_id = b.a_id
```

---

# 66. Complex predicate

JOIN podrá reutilizar:

```text
Comparison
NULL
BETWEEN
IN
EXISTS
LIKE
Boolean tests
RawPredicate
ExtensionPredicate
```

cuando tengan sentido semántico.

---

# 67. ON TRUE

Una API explícita podría representar:

```text
INNER JOIN relation ON TRUE
```

sin convertirlo accidentalmente en CROSS JOIN.

---

# 68. Important distinction

```text
INNER JOIN ... ON TRUE
≠
CROSS JOIN
```

aunque puedan ser equivalentes en algunos contextos.

El Builder preservará la intención declarada.

---

# 69. USING

Ejemplo:

```php
->joinUsing(
    'profiles',
    ['user_id']
)
```

---

# 70. JoinUsingSpecification

```php
final readonly class JoinUsingSpecification
{
    public function __construct(
        public IdentifierList $columns,
    ) {}
}
```

---

# 71. USING is structured

No se almacenará:

```text
"USING (user_id)"
```

---

# 72. USING semantics

`USING(user_id)` implica reglas particulares sobre:

```text
column matching
output naming
column visibility
duplicate output columns
nullability
```

que deberán resolverse semánticamente.

---

# 73. USING ≠ ON shortcut

No deberá reducirse inmediatamente:

```text
USING(id)
```

a:

```text
left.id = right.id
```

porque pueden existir diferencias de output-column semantics.

---

# 74. Multiple USING columns

```php
->joinUsing(
    'prices',
    ['product_id', 'currency']
)
```

---

# 75. NATURAL JOIN

V1 no deberá favorecer `NATURAL JOIN`.

---

# 76. Razón

`NATURAL JOIN` depende implícitamente del schema:

```text
same-named columns
```

por lo que cambios de schema pueden modificar silenciosamente la semántica.

---

# 77. Política recomendada

```text
NATURAL JOIN
→ unsupported by default
```

---

# 78. Extension possibility

Si se añade:

```text
NaturalJoinSpecification
```

deberá ser schema-aware y explícitamente capability-sensitive.

---

# 79. Join aliases

```php
->join('users as author', ...)
->join('users as editor', ...)
```

crea dos relation instances distintas.

---

# 80. Alias uniqueness

Dentro del mismo scope:

```text
duplicate alias
```

deberá generar error semántico.

---

# 81. Alias shadowing

Subqueries podrán crear scopes donde determinados nombres sean shadowed.

Eso será responsabilidad de Symbol Resolution.

---

# 82. Qualified names

```text
author.id
editor.id
```

serán estructurados como:

```text
QualifiedColumnReference
```

---

# 83. Join ordering

El Builder preservará el orden estructural declarado:

```text
FROM A
JOIN B
JOIN C
JOIN D
```

---

# 84. Structural order ≠ physical order

El Planner podrá terminar ejecutando:

```text
C → A → B → D
```

si la semántica lo permite.

---

# 85. Outer join restrictions

No todos los outer joins pueden reordenarse libremente.

---

# 86. Optimizer responsibility

El Optimizer deberá probar equivalencia antes de:

```text
reorder
associate
commute
eliminate
strengthen
```

JOINs.

---

# 87. Builder does not reorder

Aunque:

```text
INNER JOIN
```

pueda ser conmutativo bajo determinadas condiciones, el Builder no lo transformará.

---

# 88. Nested joins

El modelo deberá poder representar:

```text
A
LEFT JOIN
(
    B
    INNER JOIN C
)
```

---

# 89. JoinGroup

Podrá existir:

```php
final readonly class JoinGroup
{
    public function __construct(
        public RelationSource $base,
        public JoinSpecificationList $joins,
    ) {}
}
```

---

# 90. Why JoinGroup matters

Estas estructuras:

```text
(A LEFT JOIN B) INNER JOIN C
```

y:

```text
A LEFT JOIN (B INNER JOIN C)
```

no son universalmente equivalentes.

---

# 91. No flattening

Normalization no deberá aplanar JOIN groups si cambia semántica.

---

# 92. Relation tree

La representación estructural puede visualizarse como:

```text
           LEFT
          /    \
         A     INNER
              /     \
             B       C
```

---

# 93. Relation Graph

Después de Semantic Analysis:

```text
Relations
├── RA
├── RB
└── RC

Joins
├── J1
└── J2
```

más:

```text
dependencies
visibility
nullability effects
correlations
lineage
relationship evidence
```

---

# 94. Nullability propagation

Schema:

```text
users.name : string NOT NULL
```

Query:

```text
orders
LEFT JOIN users
```

Resultado:

```text
users.name
→ nullable in joined output
```

---

# 95. Important rule

```text
Schema Nullability
≠
Query Result Nullability
```

---

# 96. LEFT null extension

```text
LEFT JOIN
→ right output may become NULL
```

---

# 97. RIGHT null extension

```text
RIGHT JOIN
→ left output may become NULL
```

---

# 98. FULL null extension

```text
FULL JOIN
→ both sides may become NULL
```

---

# 99. Builder responsibility

El Builder sólo declara `JoinType`.

La nullability efectiva se deriva posteriormente.

---

# 100. SQL three-valued logic

Los predicates `ON` obedecen SQL 3VL:

```text
TRUE
FALSE
UNKNOWN
```

---

# 101. Match semantics

Para un JOIN condicionado:

```text
ON predicate
```

sólo `TRUE` produce match.

---

# 102. UNKNOWN

```text
UNKNOWN
```

no se tratará como `TRUE`.

---

# 103. Constraint analysis

El sistema de constraints podrá analizar:

```text
equality
nullability
unique keys
foreign keys
functional dependencies
constant constraints
range constraints
relationship evidence
```

---

# 104. Equi-join

```text
users.id = orders.user_id
```

podrá clasificarse como:

```text
EQUI_JOIN
```

---

# 105. Range join

```text
event.timestamp
BETWEEN period.start
AND period.end
```

podrá clasificarse:

```text
RANGE_JOIN
```

---

# 106. Inequality join

```text
a.score > b.threshold
```

podrá clasificarse:

```text
INEQUALITY_JOIN
```

---

# 107. Complex join

Un predicate compuesto podrá clasificarse:

```text
COMPLEX_JOIN
```

---

# 108. Correlated join

Un lateral source podrá derivar:

```text
CORRELATED_JOIN
```

---

# 109. Join classification ≠ algorithm

```text
EQUI_JOIN
≠
HASH_JOIN
```

---

# 110. JoinKeyCandidate

Constraint/Semantic Analysis podrá derivar:

```text
JoinKeyCandidate
├── left: users.id
└── right: orders.user_id
```

---

# 111. JoinKeyCandidate ≠ hash key

Es conocimiento lógico.

Planner decide su uso físico.

---

# 112. Foreign key evidence

Schema:

```text
orders.user_id
→ users.id
```

y predicate:

```text
orders.user_id = users.id
```

permiten adjuntar:

```text
ForeignKeyRelationshipEvidence
```

---

# 113. FK ≠ JOIN

La existencia de FK no crea automáticamente un JOIN.

---

# 114. No auto joins

Query Builder core nunca hará:

```text
orders.user_id FK users.id
→ automatically JOIN users
```

---

# 115. ORM relationships

Una relación ORM:

```text
Order::user
```

podrá servir a una API ORM superior para construir un JOIN.

---

# 116. Core independence

`JoinQueryBuilder` no dependerá de:

```text
EntityManager
ORM metadata
UnitOfWork
Model
ActiveRecord
```

---

# 117. Relationship evidence

Podrá provenir de:

```text
Foreign Key
Unique Constraint
ORM Metadata
Domain Metadata
Explicit Metadata
Extension Metadata
```

---

# 118. Evidence ≠ semantics override

La evidencia puede ayudar al análisis.

No reemplaza el predicate explícito.

---

# 119. Cardinality evidence

Podrá derivarse:

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
UNKNOWN
```

como información cualitativa.

---

# 120. Cardinality ≠ row estimate

```text
MANY_TO_ONE
```

no significa:

```text
estimated 500 rows
```

---

# 121. Key preservation

Semantic Analysis podrá determinar:

```text
KeyPreservationFact
```

para una relación.

---

# 122. Example

Si:

```text
orders.user_id
→ users.id UNIQUE
```

un JOIN puede preservar determinada identidad de `orders`.

---

# 123. But not automatically

La presencia de un JOIN puede duplicar filas dependiendo de constraints reales.

---

# 124. Join elimination

Optimizer podrá eliminar un JOIN únicamente si demuestra que:

```text
output
predicates
cardinality
nullability
existence semantics
security semantics
```

se preservan.

---

# 125. Builder never eliminates

El Builder conserva todos los JOINs declarados.

---

# 126. LEFT → INNER strengthening

Ejemplo:

```text
A
LEFT JOIN B
WHERE B.id IS NOT NULL
```

Constraint Analysis puede derivar:

```text
OuterJoinMatchRequiredFact
```

---

# 127. Transformation ownership

Convertirlo a:

```text
INNER JOIN
```

pertenece al Optimizer.

---

# 128. Security predicates in JOIN

Una integración podrá añadir:

```text
tenant_id
organization_id
visibility constraints
policy predicates
```

al JOIN.

---

# 129. Provenance

Dichos predicates deberán registrar:

```text
source
policy
mandatory status
security classification
```

---

# 130. Mandatory ON predicates

Un predicate de aislamiento podrá marcarse:

```text
MANDATORY
```

---

# 131. Optimizer restriction

No podrá eliminarse como "redundante" sin considerar su significado de seguridad.

---

# 132. Multitenancy example

```text
orders AS o
JOIN users AS u
  ON u.id = o.user_id
 AND u.tenant_id = Ptenant
```

---

# 133. No Tenant dependency

Join Builder core no conocerá:

```text
Tenant::current()
```

---

# 134. Security scope

La integración superior proporcionará el predicate explícitamente.

---

# 135. JoinMetadata

Podrá incluir:

```text
label
provenance
source location
optimization hints
security metadata
extension metadata
developer intent
```

---

# 136. Hints

Un hint como:

```text
prefer hash join
```

no deberá confundirse con una decisión física obligatoria.

---

# 137. Hint ownership

Podrá representarse:

```text
QueryPlanningHint
```

para que Planner decida si puede respetarlo.

---

# 138. Builder doesn't plan

Nunca:

```php
$join->useHashJoin();
```

como decisión física directa del core.

---

# 139. Optional ergonomic hints

Si se permite una API equivalente:

```php
->hint(JoinHint::preferHash())
```

seguirá siendo metadata, no physical plan.

---

# 140. Parameterization

Todos los runtime values dentro de predicates JOIN serán parameterizados por default.

---

# 141. Example

```php
$join->where('inventory.region', $region);
```

produce:

```text
inventory.region = P1
```

Bindings:

```text
P1 → $region
```

---

# 142. Sensitive parameter

Podrá marcarse:

```text
P1
└── Sensitive
```

---

# 143. Raw ON

Podrá existir:

```php
$join->onRaw(...);
```

como escape hatch explícito.

---

# 144. Raw barrier

`RawPredicate` puede bloquear:

```text
join classification
constraint inference
relationship analysis
optimizer transformations
portability analysis
security inspection
```

---

# 145. Raw ≠ trusted

Regla:

```text
RAW
≠
SAFE
≠
TRUSTED
≠
PORTABLE
```

---

# 146. Capability model

Ejemplos:

```text
QUERY.JOIN.INNER
QUERY.JOIN.LEFT
QUERY.JOIN.RIGHT
QUERY.JOIN.FULL
QUERY.JOIN.CROSS
QUERY.JOIN.LATERAL
QUERY.JOIN.USING
QUERY.JOIN.SUBQUERY
QUERY.JOIN.VALUES
QUERY.JOIN.TABLE_FUNCTION
QUERY.JOIN.NESTED
```

---

# 147. Capability-driven

Nunca:

```php
if ($driver === 'mysql') {
}
```

dentro del Join Builder.

---

# 148. Builder construction vs capability support

El Builder podrá construir una feature incluso antes de conocer la plataforma objetivo.

---

# 149. Semantic requirement

Posteriormente se podrá derivar:

```text
CapabilityRequirement
└── QUERY.JOIN.FULL
```

---

# 150. Planner/compiler resolution

Si la plataforma no soporta la forma nativa:

```text
can safely emulate?
```

será decisión posterior.

---

# 151. No unsafe emulation

Una feature no será emulada si cambia:

```text
nullability
duplicates
cardinality
ordering
locking
correlation
side effects
```

---

# 152. Portability

Cada JOIN podrá contribuir al:

```text
PortabilityProfile
```

---

# 153. Portable join

Ejemplo:

```text
INNER JOIN table ON equality
```

normalmente será altamente portable.

---

# 154. Less portable join

```text
FULL JOIN
LATERAL
TABLE FUNCTION
extension relation
raw predicate
```

podrán imponer requirements adicionales.

---

# 155. Validation pipeline

```text
Construction Validation
        │
        ▼
Structural Validation
        │
        ▼
Symbol Resolution
        │
        ▼
Schema Resolution
        │
        ▼
Type Inference
        │
        ▼
Relation Resolution
        │
        ▼
Constraint Analysis
        │
        ▼
Capability Validation
```

---

# 156. Construction validation

Detectará:

```text
missing source
invalid join type
invalid alias
ON + USING conflict
CROSS + ON
empty USING
invalid builder state
invalid nesting
```

---

# 157. Structural validation

Detectará:

```text
malformed predicate
malformed source
invalid subquery structure
invalid JoinGroup
invalid correlation declaration
```

---

# 158. Symbol validation

Detectará:

```text
unknown alias
ambiguous column
duplicate alias
illegal scope reference
```

---

# 159. Schema validation

Detectará:

```text
unknown relation
unknown column
invalid table function
invalid schema object
```

cuando exista schema knowledge suficiente.

---

# 160. Type validation

Podrá detectar:

```text
incompatible comparison types
invalid operator
invalid function arguments
invalid tuple comparison
```

---

# 161. Relation validation

Detectará:

```text
illegal lateral dependency
invalid join visibility
invalid relation dependency
invalid correlation
```

---

# 162. Partial schema

Si:

```text
SchemaKnowledgeMode::PARTIAL
```

la ausencia de metadata no significará automáticamente error.

---

# 163. Unknown ≠ invalid

Regla:

```text
UNKNOWN
≠
FALSE
≠
UNSUPPORTED
```

---

# 164. JoinBuilderState

Será temporal:

```text
JoinBuilderState
├── joinId
├── type
├── source
├── predicateState
├── usingState
├── correlationMode
├── metadataBuilder
├── parameterAllocator
├── sourceMap
└── diagnostics
```

---

# 165. Finalization

```text
JoinBuilderState
      │
      ▼
JoinFinalizer
      │
      ▼
JoinSpecification
```

---

# 166. Final artifact

`JoinSpecification` será immutable.

---

# 167. Builder closure

Una API:

```php
->leftJoin('users as u', function ($join) {
    ...
})
```

ejecutará el callback durante construcción.

---

# 168. Closure not stored

La closure no formará parte de:

```text
Query AST
Semantic Artifact
Compiled Query
Cache
Serialization
```

---

# 169. Persistent runtime

Esto es especialmente importante para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 170. No request capture

No deberá persistirse accidentalmente:

```text
Request
User
Tenant
Container
EntityManager
Transaction
PDO
Telemetry Span
```

dentro del JoinSpecification.

---

# 171. Shared services

Podrán compartirse:

```text
JoinBuilderFactory
IdentifierFactory
ExpressionFactory
PredicateFactory
JoinFinalizer
frozen extension registries
immutable descriptors
```

---

# 172. Operation-scoped state

Siempre local:

```text
JoinBuilderState
PredicateBuilderState
Parameter allocation
Diagnostics
Source map
Temporary metadata builders
```

---

# 173. Concurrency

Dos queries simultáneas:

```text
Coroutine A
└── JoinBuilderState A

Coroutine B
└── JoinBuilderState B
```

no compartirán estado mutable.

---

# 174. Join IDs

`JoinId` deberá asignarse de forma determinista dentro de la query.

---

# 175. No global counter

Prohibido:

```text
static $nextJoinId
```

compartido por workers.

---

# 176. Determinism

La misma construcción estructural deberá producir artifacts equivalentes.

---

# 177. Fingerprints

Podrán existir:

```text
JoinStructuralFingerprint
JoinSemanticFingerprint
JoinPlanningFingerprint
JoinCompiledFingerprint
```

---

# 178. Structural fingerprint

Podrá incluir:

```text
join type
source kind
source structural identity
alias
predicate structure
USING structure
correlation mode
semantic metadata
extension identity
```

---

# 179. Runtime values excluded

No incluirá valores concretos de bindings normales.

---

# 180. Semantic fingerprint

Podrá incorporar:

```text
resolved relation IDs
resolved symbols
types
nullability
schema identity
constraints
correlations
relationship evidence
capability requirements
extension versions
```

---

# 181. Relation dependencies

Semantic Analysis producirá:

```text
RelationDependencyGraph
```

---

# 182. Example

```text
A
JOIN B
LATERAL JOIN C
```

donde `C` referencia `A` y `B`:

```text
C → A
C → B
```

---

# 183. Dependency ≠ order only

El orden textual puede no capturar toda la semántica de dependencia.

---

# 184. Lateral constraints

Un lateral relation sólo podrá depender de símbolos visibles.

---

# 185. Join output

Un JOIN produce una relación lógica combinada.

---

# 186. JoinResultRelation

Conceptualmente:

```text
JoinResultRelation
├── left output
├── right output
├── join type
├── output nullability
├── output visibility
└── lineage
```

---

# 187. Output columns

Cada output column tendrá identidad semántica.

---

# 188. Important distinction

```text
SchemaColumnId
≠
ColumnSymbolId
≠
RelationOutputColumnId
≠
QueryOutputColumnId
```

---

# 189. USING output

USING puede crear reglas particulares de coalescencia/visibilidad.

Estas reglas pertenecerán a Semantic Resolution.

---

# 190. Lineage

Ejemplo:

```text
Query output: u.name
```

podrá tener:

```text
Lineage
→ users.name
```

---

# 191. Join lineage

El JOIN no destruye la procedencia de las columnas.

---

# 192. Data lineage ≠ provenance

```text
Lineage
```

responde:

```text
¿de qué datos proviene?
```

Mientras:

```text
Provenance
```

responde:

```text
¿qué parte del sistema introdujo esta estructura?
```

---

# 193. Query Semantic Graph

Ejemplo:

```text
orders ───────────────┐
   │                  │
   │ user_id          │
   ▼                  │
Predicate             │
   ▲                  │
   │ id               │
users ────────────────┘
   │
   ▼
INNER JOIN
   │
   ▼
Joined Relation
```

---

# 194. Outer join graph

```text
orders
   │
   ├──────── LEFT JOIN ─────── users
   │                             │
   │                             ▼
   │                      Null Extension
   │                             │
   └──────────────┬──────────────┘
                  ▼
            Joined Relation
```

---

# 195. SemanticJoin

Después de resolución:

```php
final readonly class SemanticJoin
{
    public function __construct(
        public JoinId $id,
        public JoinType $type,
        public RelationId $left,
        public RelationId $right,
        public ?PredicateSemanticId $predicate,
        public RelationDependencySet $dependencies,
        public VisibilityDescriptor $visibility,
        public NullabilityEffectSet $nullabilityEffects,
        public RelationshipEvidenceSet $relationshipEvidence,
        public CapabilityRequirementSet $capabilities,
        public SemanticProvenance $provenance,
    ) {}
}
```

---

# 196. Builder vs SemanticJoin

Builder produce:

```text
JoinSpecification
```

Semantic Engine produce:

```text
SemanticJoin
```

---

# 197. Important boundary

```text
JoinSpecification
→ declared structure

SemanticJoin
→ resolved meaning
```

---

# 198. Optimizer input

Optimizer consumirá:

```text
SemanticJoin
RelationGraph
ConstraintGraph
TypeTable
PredicateSemanticTable
RelationshipEvidence
```

---

# 199. Join optimizer

Podrá realizar:

```text
join reordering
join elimination
outer join strengthening
predicate pushdown
predicate propagation
semi-join transformation
anti-join transformation
decorrelation
redundancy elimination
```

---

# 200. Predicate pushdown

Ejemplo:

```text
A JOIN B
WHERE B.status = 'active'
```

podría permitir mover ciertas condiciones.

Pero sólo si preserva semántica.

---

# 201. Outer join caution

Mover predicates alrededor de LEFT/RIGHT/FULL JOIN puede cambiar resultados.

---

# 202. Example

```text
A LEFT JOIN B
ON A.id = B.a_id
WHERE B.active = TRUE
```

no es equivalentemente transformable de manera ingenua a:

```text
A LEFT JOIN B
ON A.id = B.a_id
AND B.active = TRUE
```

---

# 203. Rule

Builder nunca realizará predicate pushdown.

---

# 204. Planner

Planner decidirá:

```text
join ordering
physical algorithm
build/probe side
index strategy
materialization
lateral execution strategy
parallel strategy
```

---

# 205. Physical algorithms

Ejemplos:

```text
NestedLoopJoin
HashJoin
MergeJoin
IndexNestedLoopJoin
MaterializedJoin
ExtensionJoin
```

---

# 206. Logical type remains

Aunque Planner elija:

```text
HashJoin
```

el logical join puede seguir siendo:

```text
INNER
LEFT
...
```

---

# 207. Compiler

Compiler traduce el plan a capacidades reales del dialecto.

---

# 208. Compiler decisions

Puede decidir:

```text
JOIN keyword
ON syntax
USING syntax
LATERAL syntax
parentheses
derived table syntax
alias syntax
table function syntax
```

---

# 209. Executor

Executor no reinterpretará JOIN semantics.

Recibirá una query ya compilada.

---

# 210. Explainability

Podrá existir:

```php
$query->inspectJoins();
```

en tooling superior.

---

# 211. Builder inspection

Podrá mostrar:

```text
J1 INNER users AS u
   ON u.id = orders.user_id

J2 LEFT payments AS p
   ON p.order_id = orders.id
```

sin SQL compilation.

---

# 212. Semantic inspection

Después de Semantic Analysis:

```text
J1
Type: INNER
Left: orders#R1
Right: users#R2
Predicate: PRED#14
Classification: EQUI_JOIN
Relationship: MANY_TO_ONE
Key preservation: orders
Capabilities: portable
```

---

# 213. Planner inspection

Posteriormente:

```text
J1
Physical strategy: HashJoin
Build side: users
Probe side: orders
```

Estas son capas distintas.

---

# 214. Serialization

`JoinSpecification` podrá serializarse estructuralmente.

---

# 215. Never serialize

No deberá contener:

```text
PDO
Connection
Transaction
Closure
Generator
Fiber
Request
User
Tenant object
EntityManager
UnitOfWork
Telemetry Span
Container
```

---

# 216. Extension architecture

Podrán existir extensiones para:

```text
custom relation sources
custom join types
custom predicates
custom correlation modes
custom metadata
custom capabilities
```

---

# 217. Extension contract

Ejemplo conceptual:

```php
interface JoinBuilderExtension
{
    public function descriptor(): JoinExtensionDescriptor;
}
```

---

# 218. Descriptor

```text
JoinExtensionDescriptor
├── id
├── version
├── node types
├── validation support
├── semantic support
├── optimizer support
├── planner support
├── compiler support
├── capability requirements
├── portability
├── security classification
└── fingerprint contribution
```

---

# 219. Frozen registry

```text
JoinBuilderExtensionRegistry
```

será congelado después del bootstrap.

---

# 220. No runtime macros with captured state

Se evitarán macros globales que retengan closures/request state en workers persistentes.

---

# 221. Dynamic ergonomics

Si VoltStack permite métodos dinámicos, deberán resolverse contra descriptors registrados y seguros.

---

# 222. Budgets

Podrán configurarse:

```text
max joins
max nested join depth
max ON predicates
max USING columns
max lateral dependencies
max derived tables
max table functions
max join parameters
max extension nodes
```

---

# 223. JoinBuilderBudget

Ejemplo:

```text
JoinBuilderBudget
├── maxJoins: 64
├── maxDepth: 16
├── maxPredicatesPerJoin: 128
└── maxUsingColumns: 64
```

Los valores concretos serán configurables.

---

# 224. Budget exceeded

Generará:

```text
JoinBuilderBudgetExceededException
```

No truncará silenciosamente la query.

---

# 225. Error hierarchy

```text
JoinQueryBuilderException
├── InvalidJoinTypeException
├── InvalidJoinSourceException
├── MissingJoinConditionException
├── ConflictingJoinConditionException
├── InvalidJoinAliasException
├── InvalidJoinPredicateException
├── InvalidJoinUsingException
├── InvalidJoinCorrelationException
├── InvalidJoinGroupException
├── JoinBuilderBudgetExceededException
├── JoinExtensionConflictException
└── JoinBuilderFinalizationException
```

---

# 226. Semantic errors

Podrán existir:

```text
UnknownJoinRelationException
UnknownJoinColumnException
AmbiguousJoinColumnException
DuplicateJoinAliasException
InvalidJoinScopeException
InvalidLateralDependencyException
IncompatibleJoinTypeException
UnsupportedJoinCapabilityException
```

---

# 227. Diagnostics

Un error deberá mostrar:

```text
query source
join id
relation alias
predicate location
problematic identifier
semantic phase
suggested correction
```

cuando sea seguro.

---

# 228. Example diagnostic

```text
Join J2 references alias "customer"
but no visible relation with that alias exists.

Did you mean:
  customers AS c

Location:
  orders.php:42
```

---

# 229. Sensitive diagnostics

Runtime values serán redactados.

---

# 230. Testing strategy

El sistema deberá probar:

```text
join construction
join types
aliases
self joins
ON predicates
OR predicates
grouped predicates
USING
subqueries
CTEs
VALUES
table functions
lateral joins
correlations
nested joins
parameterization
immutability
serialization
extensions
budgets
persistent workers
concurrency
```

---

# 231. AST tests

Ejemplo:

```php
$query = Query::table('orders as o')
    ->join(
        'users as u',
        'u.id',
        '=',
        'o.user_id'
    );
```

deberá compararse contra:

```text
JoinSpecification
├── INNER
├── users AS u
└── ComparisonPredicate
    ├── u.id
    ├── =
    └── o.user_id
```

No contra SQL.

---

# 232. No SQL snapshot as Builder truth

Este test:

```text
assertEquals(
    'JOIN users ...',
    $builder->toSql()
);
```

no deberá ser la prueba principal del Builder.

---

# 233. Compiler tests

La generación SQL tendrá sus propios tests en:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
...
```

---

# 234. Cross-platform conformance

Deberán existir tests donde el mismo:

```text
SemanticJoin
```

se compile para diferentes plataformas.

---

# 235. Persistent-runtime test

Ejecutar miles de builders consecutivos en el mismo worker deberá demostrar:

```text
no leaked aliases
no leaked predicates
no leaked parameters
no leaked joins
no leaked metadata
no leaked extensions
```

---

# 236. Concurrent test

Dos builders simultáneos deberán poder producir:

```text
Query A
J1, J2
P1, P2

Query B
J1
P1
```

sin interferencia.

---

# 237. Performance objectives

El Builder deberá ser ligero.

La construcción de un JOIN no deberá requerir:

```text
database round-trip
schema introspection
optimizer execution
SQL compilation
connection acquisition
```

---

# 238. Complexity

Construcción normal deberá aproximarse a:

```text
O(number of declared join structures)
```

sin análisis global costoso.

---

# 239. Semantic complexity

Los análisis más costosos se delegan a:

```text
Semantic Analysis
Constraint Analysis
Optimizer
Planner
```

---

# 240. Directory structure propuesta

```text
Query/
└── Builder/
    └── Join/
        ├── Contract/
        │   ├── JoinQueryBuilderInterface.php
        │   ├── JoinClauseBuilderInterface.php
        │   └── JoinFinalizerInterface.php
        │
        ├── Core/
        │   ├── JoinQueryBuilder.php
        │   ├── JoinClauseBuilder.php
        │   ├── JoinBuilderState.php
        │   ├── JoinFinalizer.php
        │   └── JoinSpecification.php
        │
        ├── Type/
        │   ├── JoinType.php
        │   ├── JoinConditionMode.php
        │   └── JoinLateralMode.php
        │
        ├── Source/
        │   ├── JoinSource.php
        │   ├── JoinSourceKind.php
        │   ├── RelationJoinSource.php
        │   ├── DerivedQuerySource.php
        │   ├── ValuesJoinSource.php
        │   ├── TableFunctionJoinSource.php
        │   └── ExtensionJoinSource.php
        │
        ├── Predicate/
        │   ├── JoinPredicateBuilder.php
        │   ├── JoinPredicateGroup.php
        │   └── JoinConditionFactory.php
        │
        ├── Using/
        │   ├── JoinUsingSpecification.php
        │   └── JoinUsingBuilder.php
        │
        ├── Correlation/
        │   ├── JoinCorrelationMode.php
        │   └── JoinCorrelationDescriptor.php
        │
        ├── Group/
        │   ├── JoinGroup.php
        │   └── JoinGroupBuilder.php
        │
        ├── Metadata/
        │   └── JoinMetadataBuilder.php
        │
        ├── Budget/
        │   └── JoinBuilderBudget.php
        │
        ├── Extension/
        │   ├── JoinBuilderExtension.php
        │   ├── JoinExtensionDescriptor.php
        │   └── JoinBuilderExtensionRegistry.php
        │
        ├── Diagnostic/
        │   └── JoinBuilderDiagnostic.php
        │
        └── Exception/
            ├── JoinQueryBuilderException.php
            ├── InvalidJoinTypeException.php
            ├── InvalidJoinSourceException.php
            ├── MissingJoinConditionException.php
            ├── InvalidJoinCorrelationException.php
            └── JoinBuilderBudgetExceededException.php
```

---

# 241. Integration Matrix

| Sistema | Join Builder |
|---|---|
| Query AST | Produce estructura |
| Expression System | Consume expressions |
| Predicate System | Consume predicates |
| Parameter System | Registra parameters |
| Binding System | Produce bindings indirectamente |
| Metadata System | Adjunta metadata |
| Symbol Resolution | Entrega nombres para resolver |
| Schema Resolution | Entrega relation/column refs |
| Type Inference | Entrega expressions |
| Relation Resolution | Entrega JoinSpecification |
| Constraint Analysis | Consume SemanticJoin |
| Semantic Graph | Representa significado |
| Optimizer | Consume semantic joins |
| Planner | Decide estrategia |
| Compiler | Genera dialect SQL |
| Executor | Ninguna dependencia directa |
| ORM | Integración superior opcional |
| Multitenancy | Integración superior opcional |
| Security | Integración superior opcional |
| Telemetry | Observación posterior |

---

# 242. Architectural Invariants

## DB-JOIN-001

Join Builder nunca generará SQL.

## DB-JOIN-002

Nunca concatenará `JOIN`.

## DB-JOIN-003

Nunca concatenará `ON`.

## DB-JOIN-004

Nunca concatenará `USING`.

## DB-JOIN-005

Nunca realizará identifier quoting.

## DB-JOIN-006

Nunca abrirá conexiones.

## DB-JOIN-007

Nunca realizará schema introspection mediante I/O.

## DB-JOIN-008

Nunca elegirá physical join algorithms.

## DB-JOIN-009

Logical Join Type será distinto de Physical Join Strategy.

## DB-JOIN-010

Schema Relation será distinta de Relation Instance.

## DB-JOIN-011

Cada JOIN tendrá JoinId.

## DB-JOIN-012

Cada relation instance tendrá identidad propia.

## DB-JOIN-013

Self joins crearán relation instances distintas.

## DB-JOIN-014

Aliases serán estructurados.

## DB-JOIN-015

Duplicate aliases serán errores de scope.

## DB-JOIN-016

INNER será explícito semánticamente.

## DB-JOIN-017

LEFT será explícito semánticamente.

## DB-JOIN-018

RIGHT será explícito semánticamente.

## DB-JOIN-019

FULL será explícito semánticamente.

## DB-JOIN-020

CROSS será explícito semánticamente.

## DB-JOIN-021

RIGHT no se reescribirá automáticamente en Builder.

## DB-JOIN-022

FULL será capability-driven.

## DB-JOIN-023

CROSS no requerirá matching predicate.

## DB-JOIN-024

CROSS + ON será inválido por default.

## DB-JOIN-025

ON y USING no coexistirán salvo extensión explícita.

## DB-JOIN-026

USING será estructurado.

## DB-JOIN-027

USING no será reducido prematuramente a ON.

## DB-JOIN-028

NATURAL JOIN no será default.

## DB-JOIN-029

NATURAL JOIN será schema-sensitive si se soporta.

## DB-JOIN-030

Join source será tipado.

## DB-JOIN-031

Subquery join usará finalized query artifacts.

## DB-JOIN-032

Artifacts no retendrán mutable child builders.

## DB-JOIN-033

Derived relations podrán requerir alias.

## DB-JOIN-034

CTE resolution pertenecerá a Symbol Resolution.

## DB-JOIN-035

VALUES será estructurado.

## DB-JOIN-036

VALUES runtime values serán parameterized.

## DB-JOIN-037

Table functions serán semánticas, no SQL strings.

## DB-JOIN-038

LATERAL será explícito.

## DB-JOIN-039

Correlation será explícita semánticamente.

## DB-JOIN-040

Correlation no implicará physical per-row execution.

## DB-JOIN-041

Illegal correlation será error semántico.

## DB-JOIN-042

ON reutilizará Predicate System.

## DB-JOIN-043

on() distinguirá expressions de runtime values.

## DB-JOIN-044

where() dentro del JOIN parameterizará values.

## DB-JOIN-045

Parameters pertenecerán a la query.

## DB-JOIN-046

No existirán physical placeholders en Builder.

## DB-JOIN-047

Multiple ON predicates conservarán grouping.

## DB-JOIN-048

Boolean precedence será explícita.

## DB-JOIN-049

SQL 3VL será preservado.

## DB-JOIN-050

UNKNOWN no será tratado como TRUE.

## DB-JOIN-051

Builder preservará declared join order.

## DB-JOIN-052

Declared order no será physical order.

## DB-JOIN-053

Builder nunca reordenará JOINs.

## DB-JOIN-054

Builder nunca eliminará JOINs.

## DB-JOIN-055

Builder nunca fortalecerá outer joins.

## DB-JOIN-056

Nested join grouping será preservado.

## DB-JOIN-057

Normalization no aplanará joins de forma insegura.

## DB-JOIN-058

LEFT JOIN podrá null-extend right output.

## DB-JOIN-059

RIGHT JOIN podrá null-extend left output.

## DB-JOIN-060

FULL JOIN podrá null-extend ambos outputs.

## DB-JOIN-061

Schema nullability será distinta de result nullability.

## DB-JOIN-062

Nullability effects serán derivados semánticamente.

## DB-JOIN-063

Join classification será distinta de physical algorithm.

## DB-JOIN-064

EQUI_JOIN no significará HASH_JOIN.

## DB-JOIN-065

JoinKeyCandidate será conocimiento lógico.

## DB-JOIN-066

Foreign key no creará JOIN automáticamente.

## DB-JOIN-067

ORM relationship no será dependencia core.

## DB-JOIN-068

Relationship evidence no reemplazará predicates.

## DB-JOIN-069

Cardinality evidence será cualitativa.

## DB-JOIN-070

Cardinality evidence no será cost estimate.

## DB-JOIN-071

Key preservation deberá demostrarse.

## DB-JOIN-072

Join elimination pertenecerá al Optimizer.

## DB-JOIN-073

Outer join strengthening pertenecerá al Optimizer.

## DB-JOIN-074

Predicate pushdown pertenecerá al Optimizer.

## DB-JOIN-075

Security predicates serán explícitos.

## DB-JOIN-076

Security predicates conservarán provenance.

## DB-JOIN-077

Mandatory predicates no serán eliminados silenciosamente.

## DB-JOIN-078

Multitenancy será integración opcional.

## DB-JOIN-079

Join Builder no consultará global current tenant.

## DB-JOIN-080

Join metadata será tipada.

## DB-JOIN-081

Planning hints no serán physical decisions.

## DB-JOIN-082

Runtime values serán parameterized by default.

## DB-JOIN-083

Sensitive parameters serán redactables.

## DB-JOIN-084

Raw predicates serán explícitos.

## DB-JOIN-085

Raw no significará trusted.

## DB-JOIN-086

Raw podrá actuar como semantic barrier.

## DB-JOIN-087

Capabilities reemplazarán vendor checks.

## DB-JOIN-088

No habrá `if mysql` en Builder.

## DB-JOIN-089

No habrá `if postgres` en Builder.

## DB-JOIN-090

No habrá `if sqlite` en Builder.

## DB-JOIN-091

Unsupported native syntax no implicará inmediatamente unsupported semantics.

## DB-JOIN-092

Emulation deberá preservar semántica.

## DB-JOIN-093

Portability será calculable.

## DB-JOIN-094

Unknown schema metadata no significará ausencia.

## DB-JOIN-095

UNKNOWN no significará INVALID.

## DB-JOIN-096

JoinBuilderState será operation-scoped.

## DB-JOIN-097

JoinSpecification será immutable.

## DB-JOIN-098

Closures serán evaluadas durante construcción.

## DB-JOIN-099

Closures no se almacenarán en artifacts.

## DB-JOIN-100

Artifacts no almacenarán runtime resources.

## DB-JOIN-101

No habrá global current Join Builder.

## DB-JOIN-102

No habrá global JoinId counter.

## DB-JOIN-103

Builder será persistent-runtime safe.

## DB-JOIN-104

Builder será coroutine-safe.

## DB-JOIN-105

Shared registries serán frozen.

## DB-JOIN-106

Extensions serán versionadas.

## DB-JOIN-107

Extensions declararán semantic support.

## DB-JOIN-108

Extensions declararán compiler support cuando sea necesario.

## DB-JOIN-109

Extensions contribuirán al fingerprint cuando afecten semántica.

## DB-JOIN-110

Budgets no truncarán queries silenciosamente.

## DB-JOIN-111

SemanticJoin será distinto de JoinSpecification.

## DB-JOIN-112

RelationGraph será authoritative para relation topology.

## DB-JOIN-113

ConstraintGraph será authoritative para derived constraints.

## DB-JOIN-114

Semantic Graph consolidará relaciones cross-semantic.

## DB-JOIN-115

Optimizer consumirá semantic results.

## DB-JOIN-116

Optimizer no re-resolverá aliases desde strings.

## DB-JOIN-117

Planner elegirá physical strategy.

## DB-JOIN-118

Compiler elegirá dialect syntax.

## DB-JOIN-119

Executor no interpretará JOIN semantics.

## DB-JOIN-120

La ergonomía pública no romperá las fronteras internas.

---

# 243. Anti-patterns

## 243.1 Concatenar JOIN

```php
$sql .= " JOIN {$table} ON {$condition}";
```

**Rechazado.**

---

## 243.2 Driver logic

```php
if ($driver === 'pgsql') {
    ...
}
```

**Rechazado.**

---

## 243.3 Auto FK join

```text
Foreign Key detected
→ automatically join relation
```

**Rechazado en core.**

---

## 243.4 Physical strategy in Builder

```php
$join->hashJoin();
```

como decisión obligatoria.

**Rechazado.**

---

## 243.5 Store closures

```text
JoinSpecification
└── Closure
```

**Rechazado.**

---

## 243.6 Mutable child query

```text
JoinSpecification
└── SelectQueryBuilder
```

**Rechazado.**

---

## 243.7 Flatten all joins

```text
nested joins
→ flat list always
```

**Rechazado.**

---

## 243.8 LEFT to INNER in Builder

```text
LEFT JOIN + WHERE right.id NOT NULL
→ INNER JOIN
```

**Rechazado en Builder.**

---

## 243.9 USING to ON immediately

```text
USING(id)
→ left.id = right.id
```

sin conservar USING semantics.

**Rechazado.**

---

## 243.10 Global relation aliases

```php
static $aliases = [];
```

**Rechazado.**

---

# 244. Ejemplo completo

```php
$query = DB::table('orders as o')
    ->join('users as u', function ($join) {
        $join
            ->on('u.id', '=', 'o.user_id')
            ->where('u.active', true);
    })
    ->leftJoin('payments as p', function ($join) {
        $join
            ->on('p.order_id', '=', 'o.id')
            ->whereNull('p.deleted_at');
    })
    ->where('o.status', 'pending')
    ->select(
        'o.id',
        'u.name',
        'p.status'
    );
```

---

# 245. Builder representation

```text
SelectQueryModel
│
├── FROM
│   └── orders AS o
│
├── JOIN J1
│   ├── type: INNER
│   ├── source: users AS u
│   └── predicate
│       └── AND
│           ├── u.id = o.user_id
│           └── u.active = P1
│
├── JOIN J2
│   ├── type: LEFT
│   ├── source: payments AS p
│   └── predicate
│       └── p.deleted_at IS NULL
│
├── WHERE
│   └── o.status = P2
│
└── SELECT
    ├── o.id
    ├── u.name
    └── p.status
```

Bindings:

```text
P1 → true
P2 → "pending"
```

---

# 246. Semantic resolution

Después de resolución:

```text
Relations
├── R1 orders AS o
├── R2 users AS u
└── R3 payments AS p

Joins
├── J1
│   ├── INNER
│   ├── R1 ↔ R2
│   ├── EQUI_JOIN
│   └── users.id ↔ orders.user_id
│
└── J2
    ├── LEFT
    ├── JoinResult(J1) ↔ R3
    ├── EQUI_JOIN
    └── payments.order_id ↔ orders.id
```

---

# 247. Constraint knowledge

Si schema declara:

```text
orders.user_id
→ FK users.id

users.id
→ PK

payments.order_id
→ FK orders.id
```

Constraint Analysis podrá derivar:

```text
J1 relationship
→ MANY_TO_ONE

J1 users side
→ AT_MOST_ONE match per order

J2 relationship
→ possibly ONE_TO_MANY
```

dependiendo de constraints reales.

---

# 248. Nullability

Debido a J2:

```text
p.status
→ query-nullable
```

aunque:

```text
payments.status
→ schema NOT NULL
```

---

# 249. Semantic graph

```text
                       Query
                         │
                         ▼
                    orders R1
                         │
                         │ J1 INNER
                         ▼
                     users R2
                         │
                         ▼
                  JoinedRelation
                         │
                         │ J2 LEFT
                         ▼
                    payments R3
                         │
                         ▼
                 FinalRelationOutput
```

con:

```text
predicates
types
constraints
lineage
nullability
dependencies
relationship evidence
```

como side tables asociadas.

---

# 250. Optimizer

Podrá analizar:

```text
J1/J2 ordering
predicate pushdown
key preservation
join elimination
outer join strengthening
relationship constraints
```

sin modificar la intención arbitrariamente.

---

# 251. Planner

Podría producir conceptualmente:

```text
Scan users
   │
   ▼
Hash Build
   │
   ├─────────────┐
   │             │
Scan orders      │
   │             │
   ▼             │
Hash Join ◄──────┘
   │
   ▼
Index/Hash Join payments
   │
   ▼
Projection
```

El Builder no conoce nada de este plan.

---

# 252. Pipeline definitivo

```text
Developer
   │
   ▼
QueryBuilder
   │
   ▼
JoinQueryBuilder
   │
   ├── JoinSource
   ├── JoinType
   ├── ON / USING
   ├── Parameters
   ├── Correlation
   ├── Groups
   └── Metadata
          │
          ▼
   JoinSpecification
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
    Symbol Resolution
          │
          ▼
    Schema Resolution
          │
          ▼
      Type Inference
          │
          ▼
Relation & Join Resolution
          │
          ▼
      SemanticJoin
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
       Compiler
          │
          ▼
       Executor
```

---

# 253. Fórmula maestra

```text
JoinSpecification
=
Logical Join Type
+
Relation Source
+
Matching Predicate / USING
+
Correlation Declaration
+
Structural Grouping
+
Metadata
```

---

# 254. Fórmula semántica

```text
Semantic Join
=
Resolved Left Relation
+
Resolved Right Relation
+
Join Type
+
Resolved Predicate
+
Visibility
+
Dependencies
+
Correlation
+
Nullability Effects
+
Relationship Evidence
+
Constraint Knowledge
+
Capability Requirements
+
Provenance
```

---

# 255. Fórmula del Relation Graph

```text
RelationGraph
=
Relation Instances
+
Semantic Joins
+
Relation Dependencies
+
Correlations
+
Visibility
+
Outputs
+
Lineage
+
Nullability Effects
+
Relationship Evidence
```

---

# 256. Fórmula de optimización

```text
Join Optimization
=
Semantic Join Graph
+
Constraint Knowledge
+
Type Knowledge
+
Relationship Evidence
+
Cardinality Knowledge
+
Capability Knowledge
+
Cost Model
```

No:

```text
SQL string parsing
```

---

# 257. Fórmula de portabilidad

```text
Portable Join Construction
=
Semantic Join Declaration
-
Vendor Syntax Assumptions
+
Capability Requirements
+
Planner Strategies
+
Dialect Compilation
```

---

# 258. Fórmula de seguridad

```text
Safe Join Construction
=
Structured Identifiers
+
Typed Expressions
+
Parameterized Values
+
Explicit Predicate Groups
+
Explicit Correlations
+
Provenance
+
Mandatory Policy Predicates
+
Resource Budgets
```

---

# 259. Fórmula de persistent-runtime safety

```text
Persistent-Safe Join Builder
=
Immutable Final Specifications
+
Frozen Shared Registries
+
Operation-Scoped Builder State
+
Operation-Scoped Parameter State
+
No Global Alias Registry
+
No Global Join Counter
+
No Stored Closures
+
No Runtime Resource Retention
```

---

# 260. Principio final

La frontera conceptual deberá permanecer:

```text
Join Builder
     │
     ▼
"What relation composition did
the developer declare?"
     │
     ▼
Relation & Join Resolution
     │
     ▼
"What relations and symbols
does that declaration mean?"
     │
     ▼
Constraint Analysis
     │
     ▼
"What can be proven about
this relationship?"
     │
     ▼
Optimizer
     │
     ▼
"Which equivalent logical
form is better?"
     │
     ▼
Planner
     │
     ▼
"Which physical join strategy
should be used?"
     │
     ▼
Compiler
     │
     ▼
"How is that strategy represented
for this database platform?"
```

---

# 261. Conclusión

`JoinQueryBuilder` será la capa común para expresar composición relacional en VoltStack.

Permitirá construir:

```text
INNER JOIN
LEFT JOIN
RIGHT JOIN
FULL JOIN
CROSS JOIN
ON
USING
subquery joins
CTE joins
VALUES joins
table-function joins
LATERAL joins
correlated joins
nested joins
parameterized join predicates
security-aware predicates
extension joins
```

sin acoplarse a:

```text
PDO
physical connections
database introspection
ORM
UnitOfWork
IdentityMap
SQL strings
vendor syntax
physical join algorithms
```

La arquitectura completa mantendrá:

```text
Developer
    │
    ▼
JoinQueryBuilder
    │
    ▼
JoinSpecification
    │
    ▼
Relation Resolution
    │
    ▼
SemanticJoin
    │
    ▼
Constraint Analysis
    │
    ▼
SemanticQueryGraph
    │
    ▼
Optimizer
    │
    ▼
Logical Join Graph
    │
    ▼
Planner
    │
    ▼
Physical Join Plan
    │
    ▼
Compiler
    │
    ▼
SQL
```

De esta manera VoltStack puede ofrecer una API cercana a Laravel:

```php
DB::table('orders')
    ->join('users', ...)
    ->leftJoin('payments', ...)
```

mientras internamente conserva una arquitectura más estricta:

```text
Laravel-like DX
+
typed Query AST
+
explicit relation identities
+
semantic join resolution
+
constraint-aware reasoning
+
join optimization
+
capability-driven portability
+
persistent-runtime safety
```

> **`JoinQueryBuilder` declara la topología relacional solicitada por el desarrollador; `Relation & Join Resolution` determina su significado, Constraint Analysis descubre sus propiedades, Optimizer transforma únicamente equivalencias demostrables y Planner decide cómo ejecutar físicamente esa topología.**

---

# 262. Estado del bloque Query Builder

Con este documento:

```text
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
```

queda definida la infraestructura fundamental de construcción de queries y relaciones.

Los siguientes documentos desarrollarán capacidades composables superiores.

---

# 263. Siguiente documento

```text
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
Subquery Architecture
Scalar Subquery
Row Subquery
Table Subquery
Predicate Subquery
EXISTS / NOT EXISTS
IN Subquery
Correlated Subquery
Derived Tables
Subquery Scope
Outer References
Correlation Descriptors
Decorrelation Boundaries
CTE Architecture
Non-Recursive CTE
Recursive CTE
CTE Scope
CTE Dependency Graph
CTE Materialization Intent
CTE Column Lists
CTE Recursion Validation
Mutual Dependencies
Cycle Detection
Recursive Anchor
Recursive Member
UNION semantics
Search/Traversal Extensions
CTE capability requirements
CTE semantic fingerprints
CTE optimization boundaries
CTE in SELECT
CTE in INSERT
CTE in UPDATE
CTE in DELETE
persistent-runtime safety
```

manteniendo estrictamente:

```text
Subquery/CTE Builder
        │
        ▼
Structural Query Artifact
        │
        ▼
Scope & Symbol Resolution
        │
        ▼
Correlation Resolution
        │
        ▼
Semantic Query Graph
        │
        ▼
Optimizer
        │
        ├── decorrelation
        ├── inlining
        ├── materialization decisions
        └── predicate transformations
        │
        ▼
Planner
        │
        ▼
Compiler
```