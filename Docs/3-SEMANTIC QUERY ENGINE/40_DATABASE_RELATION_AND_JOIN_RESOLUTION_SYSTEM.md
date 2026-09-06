# 40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md

# VoltStack Quantum Database
## Relation and Join Resolution System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 40 — Relation and Join Resolution System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

El **Relation and Join Resolution System** es responsable de transformar las fuentes relacionales presentes en una consulta en una representación semántica explícita que describa:

- qué relaciones participan;
- qué columnas expone cada relación;
- cómo se conectan;
- qué tipo de JOIN existe;
- qué predicates pertenecen al JOIN;
- qué símbolos son visibles;
- qué relaciones dependen de otras;
- qué subqueries están correlacionadas;
- qué efectos de nullability introduce cada JOIN;
- qué relaciones provienen del schema;
- qué relaciones son derivadas;
- qué dependencias laterales existen;
- qué restricciones semánticas afectan futuras optimizaciones.

Este sistema convierte:

```sql
SELECT u.id, o.total
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
WHERE u.active = true
```

en una representación conceptual:

```text
RelationGraph
│
├── R1 users AS u
│   ├── id
│   ├── email
│   └── active
│
├── R2 orders AS o
│   ├── id
│   ├── user_id
│   └── total
│
└── J1 LEFT_JOIN
    ├── left: R1
    ├── right: R2
    ├── predicate:
    │   └── R2.user_id = R1.id
    └── semantic effects:
        └── R2 output nullable
```

sin seleccionar todavía un algoritmo físico.

---

# 2. Regla maestra

> Relation Resolution describe qué relaciones existen y cómo se relacionan semánticamente. El Query Planner decidirá posteriormente cómo ejecutarlas físicamente.

Por tanto:

```text
Semantic Join
≠
Physical Join Algorithm
```

y:

```text
LEFT JOIN
≠
Nested Loop Join
≠
Hash Join
≠
Merge Join
```

---

# 3. Separaciones fundamentales

VoltStack deberá mantener:

```text
Schema Table
≠
Query Relation
≠
Relation Symbol
≠
Relation Instance
≠
Join
≠
Relationship
≠
Physical Join
```

Asimismo:

```text
Foreign Key
≠
JOIN
```

y:

```text
ORM Relationship
≠
SQL JOIN
```

aunque ambos puedan aportar evidencia para resolver una consulta.

---

# 4. Ubicación dentro del pipeline

```text
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
    │
    ├── Schema-Aware Resolution
    │
    ├── Type Inference
    │
    ├── Relation / Join Resolution
    │
    ├── Constraint Analysis
    │
    ▼
Semantic Query Graph
    │
    ▼
Optimizer
    │
    ▼
Planner
```

---

# 5. Responsabilidades

El sistema deberá resolver:

```text
FROM sources
JOIN sources
derived tables
subqueries
CTEs
VALUES relations
table functions
aliases
relation visibility
column outputs
join predicates
join dependencies
correlation
lateral references
outer join nullability
relationship evidence
relation lineage
```

---

# 6. No responsabilidades

No deberá:

- generar SQL;
- abrir conexiones;
- ejecutar consultas;
- consultar estadísticas físicas;
- seleccionar índices;
- elegir Hash Join;
- elegir Merge Join;
- elegir Nested Loop;
- reordenar JOINs por costo;
- hacer predicate pushdown;
- materializar CTEs;
- decidir distribución/sharding;
- hidratar entidades;
- ejecutar lazy loading.

---

# 7. Concepto de Query Relation

Una **Query Relation** representa una fuente tabular visible dentro de una consulta.

Ejemplos:

```text
Physical Table
View
CTE
Derived Table
Subquery
VALUES
Table Function
Join Result
Extension Relation
```

---

# 8. RelationId

Cada instancia relacional tendrá:

```text
RelationId
```

Ejemplo:

```text
R1
R2
R3
```

---

# 9. RelationId ≠ SchemaObjectId

La tabla:

```text
users
```

puede aparecer dos veces:

```sql
FROM users manager
JOIN users employee
```

Por tanto:

```text
SchemaObject(users)
        │
        ├── RelationInstance R1 manager
        └── RelationInstance R2 employee
```

---

# 10. RelationInstance

Modelo conceptual:

```text
RelationInstance
├── relationId
├── source
├── alias
├── output
├── scope
├── lineage
├── dependencies
├── nullabilityEffect
└── metadata
```

---

# 11. RelationSource

Tipos iniciales:

```text
TABLE
VIEW
CTE
DERIVED_TABLE
SUBQUERY
VALUES
TABLE_FUNCTION
JOIN_RESULT
EXTENSION
```

---

# 12. Physical table relation

Ejemplo:

```sql
FROM users u
```

resuelve:

```text
TableRelation
├── RelationId R1
├── SchemaObject users
├── Alias u
└── Output
    ├── id
    ├── email
    ├── active
    └── created_at
```

---

# 13. View relation

Una vista puede resolverse como:

```text
ViewRelation
├── schema object
├── declared output
└── capabilities / metadata
```

sin expandir obligatoriamente su definición interna.

---

# 14. CTE relation

Ejemplo:

```sql
WITH active_users AS (
    SELECT id, email
    FROM users
    WHERE active = true
)
SELECT *
FROM active_users;
```

produce:

```text
CTE C1
    │
    ▼
Relation R1
├── id
└── email
```

---

# 15. Derived table

```sql
FROM (
    SELECT user_id, SUM(total) AS total
    FROM orders
    GROUP BY user_id
) stats
```

produce una nueva relación:

```text
DerivedRelation
├── alias: stats
└── output
    ├── user_id
    └── total
```

---

# 16. Derived relation output

El output proviene de:

```text
Projection Resolution
+
Type Inference
+
Alias Resolution
```

No directamente del schema físico.

---

# 17. VALUES relation

Ejemplo:

```sql
FROM (
    VALUES
        (1, 'A'),
        (2, 'B')
) AS x(id, name)
```

puede producir:

```text
ValuesRelation
├── id   : Integer
└── name : String
```

---

# 18. Table function

Motores/extensiones podrán exponer:

```text
TableFunctionRelation
```

para funciones que retornan conjuntos tabulares.

---

# 19. Extension relation

Un paquete podrá introducir:

```text
ExtensionRelation
```

pero deberá participar formalmente en:

- symbol resolution;
- type inference;
- output resolution;
- dependency analysis;
- capability validation;
- semantic graph;
- fingerprinting.

---

# 20. RelationOutput

Cada relación expone:

```text
RelationOutput
```

---

# 21. RelationOutputColumn

Modelo conceptual:

```text
RelationOutputColumn
├── outputColumnId
├── relationId
├── name
├── sourceSymbol?
├── type
├── nullability
├── lineage
├── visibility
└── metadata
```

---

# 22. OutputColumnId

Debe mantenerse:

```text
ColumnSymbolId
≠
OutputColumnId
```

porque una projection puede transformar una columna.

---

# 23. Ejemplo

```sql
SELECT price * quantity AS subtotal
```

`subtotal` no es una columna física.

Es:

```text
Derived Output Column
```

---

# 24. Relation lineage

VoltStack deberá poder rastrear:

```text
Derived output
    │
    ▼
Expression
    │
    ▼
Source columns
    │
    ▼
Source relations
```

---

# 25. Ejemplo lineage

```text
subtotal
   │
   └── Multiply
       ├── orders.price
       └── orders.quantity
```

---

# 26. Join Model

Un JOIN semántico será representado mediante:

```text
SemanticJoin
```

---

# 27. SemanticJoin

Modelo conceptual:

```text
SemanticJoin
├── joinId
├── type
├── left
├── right
├── predicate
├── dependencies
├── visibility
├── nullabilityEffects
├── capabilities
├── provenance
└── metadata
```

---

# 28. JoinId

Cada JOIN tendrá:

```text
JoinId
```

independiente de sus relaciones.

---

# 29. Join types

V1 deberá contemplar:

```text
INNER
LEFT
RIGHT
FULL
CROSS
```

y cuando el Query Model lo requiera:

```text
LATERAL
CROSS_LATERAL
LEFT_LATERAL
```

Los conceptos:

```text
SEMI
ANTI
```

podrán aparecer posteriormente como representación lógica del Optimizer/Planner.

---

# 30. INNER JOIN

```sql
A INNER JOIN B ON predicate
```

semánticamente:

```text
Rows where predicate evaluates TRUE
```

---

# 31. SQL three-valued logic

Para JOIN predicates:

```text
TRUE
→ match

FALSE
→ no match

UNKNOWN
→ no match
```

pero `UNKNOWN` sigue siendo semánticamente distinto de `FALSE`.

---

# 32. LEFT JOIN

```sql
A LEFT JOIN B ON predicate
```

preserva:

```text
A
```

y convierte el output de B en potencialmente nullable.

---

# 33. Nullability effect

Si:

```text
orders.id
```

es:

```text
NOT NULL
```

en schema, entonces:

```sql
users
LEFT JOIN orders
```

produce:

```text
orders.id
→ QUERY NULLABLE
```

---

# 34. RIGHT JOIN

El efecto equivalente se aplica sobre la relación izquierda.

---

# 35. FULL JOIN

Ambos lados se vuelven potencialmente nullable.

---

# 36. CROSS JOIN

No posee predicate de matching.

```text
CROSS JOIN
=
Cartesian relational combination
```

---

# 37. CROSS JOIN no es INNER JOIN TRUE

Aunque puedan ser equivalentes en determinados contextos relacionales, VoltStack preservará la intención estructural.

---

# 38. Join predicate

El predicate debe ser un:

```text
PredicateNode
```

ya resuelto semánticamente.

---

# 39. Join predicate dependencies

Ejemplo:

```sql
ON o.user_id = u.id
```

produce:

```text
PredicateDependency
├── R1 users
└── R2 orders
```

---

# 40. Invalid join predicate

Ejemplo:

```sql
FROM users u
JOIN orders o
    ON p.user_id = u.id
```

si `p` no es visible:

```text
UnresolvedRelationReferenceException
```

---

# 41. Join predicate classification

El sistema podrá clasificar predicates como:

```text
EQUI_JOIN
RANGE_JOIN
INEQUALITY_JOIN
COMPLEX_JOIN
CONSTANT_JOIN
CORRELATED_JOIN
EXTENSION
```

sin elegir algoritmo físico.

---

# 42. Equi-join

Ejemplo:

```text
orders.user_id = users.id
```

produce evidencia:

```text
Equality Relation Constraint
```

---

# 43. Join key candidate

Podrá derivarse:

```text
JoinKeyCandidate
├── leftExpression
├── rightExpression
├── compatibility
└── provenance
```

---

# 44. Join key ≠ physical hash key

El Planner podrá usarlo posteriormente para Hash Join, pero el Semantic Layer no toma esa decisión.

---

# 45. Range join

Ejemplo:

```sql
ON event.timestamp >= period.start
AND event.timestamp < period.end
```

se clasifica semánticamente sin determinar implementación.

---

# 46. Complex join

Ejemplo:

```sql
ON LOWER(a.email) = LOWER(b.email)
```

continúa siendo una relación semántica válida.

---

# 47. RelationGraph

El principal artifact será:

```text
RelationGraph
```

---

# 48. Graph model

```text
RelationGraph
├── Relations
├── Joins
├── Dependencies
├── Correlations
├── Visibility
├── NullabilityEffects
├── RelationshipEvidence
└── Lineage
```

---

# 49. Ejemplo simple

```text
R1 users
 |
 | J1 INNER
 |
R2 orders
```

---

# 50. Ejemplo múltiple

```sql
FROM users u
JOIN orders o
    ON o.user_id = u.id
LEFT JOIN payments p
    ON p.order_id = o.id
```

produce:

```text
R1 users
   │
   └── J1 INNER
       │
       R2 orders
          │
          └── J2 LEFT
              │
              R3 payments
```

---

# 51. Join tree ≠ join graph

El AST puede representar inicialmente:

```text
Join Tree
```

mientras que Semantic Analysis puede construir:

```text
Relation Graph
```

con información adicional de dependencias.

---

# 52. AST join order

El orden estructural original deberá preservarse en esta fase.

---

# 53. No join reordering

Relation Resolution no realizará:

```text
A JOIN B JOIN C
```

→

```text
C JOIN A JOIN B
```

por costo.

Eso pertenece al Optimizer/Planner.

---

# 54. Join associativity

No se asumirá universalmente:

```text
(A JOIN B) JOIN C
=
A JOIN (B JOIN C)
```

especialmente con:

```text
OUTER JOIN
LATERAL
correlation
volatile predicates
```

---

# 55. Join commutativity

Tampoco se aplicará automáticamente:

```text
A JOIN B
=
B JOIN A
```

aunque INNER JOIN pueda permitir ciertas transformaciones bajo condiciones posteriores.

---

# 56. Relation dependency

Se modelará:

```text
RelationDependency
```

---

# 57. Independent relation

Una tabla normal:

```text
users
```

puede no depender de otra relación.

---

# 58. Dependent relation

Una subquery lateral:

```sql
FROM users u
JOIN LATERAL (
    SELECT *
    FROM orders o
    WHERE o.user_id = u.id
) x ON true
```

depende de:

```text
x → u
```

---

# 59. Dependency direction

```text
Dependent Relation
        │
        ▼
Required Outer Relation
```

---

# 60. DependencyGraph

Podrá existir:

```text
RelationDependencyGraph
```

como parte del `RelationGraph`.

---

# 61. Dependency cycle

Un ciclo inválido:

```text
R1 depends R2
R2 depends R1
```

deberá detectarse cuando la semántica no permita recursividad explícita.

---

# 62. LATERAL

LATERAL modifica las reglas de visibilidad.

---

# 63. Non-lateral derived relation

Normalmente:

```sql
FROM users u,
(
    SELECT *
    FROM orders
    WHERE orders.user_id = u.id
) x
```

no puede referenciar `u` si el dialecto/forma no permite lateral correlation.

---

# 64. Lateral relation

Con:

```sql
LATERAL (...)
```

la referencia puede ser válida.

---

# 65. Lateral capability

La consulta puede producir:

```text
CapabilityRequirement:
query.join.lateral
```

---

# 66. Capability ≠ dialect check

Nunca:

```php
if ($platform === 'postgresql')
```

en Relation Resolution.

---

# 67. Correlation

Una subquery está correlacionada cuando referencia símbolos de un scope exterior.

---

# 68. Ejemplo

```sql
SELECT u.id
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

---

# 69. Correlation edge

Se representa:

```text
Subquery S1
    │
    └── Correlation
        └── Outer Symbol u.id
```

---

# 70. CorrelationDescriptor

Modelo:

```text
CorrelationDescriptor
├── subqueryId
├── outerScopeId
├── outerSymbols
├── innerReferences
├── depth
└── provenance
```

---

# 71. Correlation depth

Debe conocerse si una subquery referencia:

```text
parent
grandparent
higher scope
```

---

# 72. Correlation no implica ejecución por fila

El Semantic Layer sólo declara:

```text
logical correlation
```

No decide:

```text
execute subquery once per row
```

---

# 73. Decorrelation

Transformaciones como:

```text
correlated EXISTS
→ semi join
```

pertenecen al Optimizer.

---

# 74. NOT EXISTS

Puede ser candidato futuro a:

```text
ANTI JOIN
```

pero no se reescribe aquí.

---

# 75. NOT IN

Nunca deberá equipararse ingenuamente con:

```text
ANTI JOIN
```

por semántica de NULL.

---

# 76. Relation visibility

Cada relation scope tendrá reglas explícitas.

---

# 77. Visibility model

Conceptualmente:

```text
RelationVisibility
├── localRelations
├── outerVisibleRelations
├── lateralVisibleRelations
├── cteRelations
└── hiddenRelations
```

---

# 78. Alias visibility

Cuando existe:

```sql
FROM users AS u
```

el nombre válido normalmente será:

```text
u
```

y no necesariamente:

```text
users
```

según las reglas del Query Model.

---

# 79. Duplicate alias

```sql
FROM users u
JOIN orders u
```

deberá producir:

```text
DuplicateRelationAliasException
```

---

# 80. Ambiguous unqualified column

```sql
SELECT id
FROM users
JOIN orders
```

si ambos exponen `id`:

```text
AmbiguousColumnReferenceException
```

---

# 81. Qualified column

```text
users.id
```

o alias:

```text
u.id
```

elimina la ambigüedad cuando la relación es visible.

---

# 82. Relation namespace

Los aliases de relación pertenecen a un namespace semántico específico.

---

# 83. Relation alias ≠ projection alias

Debe mantenerse:

```text
RelationAlias
≠
ProjectionAlias
```

---

# 84. CTE namespace

Los CTEs deberán manejarse mediante namespace/scope explícito.

---

# 85. Nested CTE scopes

Una consulta interna puede introducir CTEs que oculten nombres exteriores cuando la semántica lo permita.

---

# 86. Shadowing

El shadowing deberá ser:

```text
explicit
deterministic
scope-aware
diagnosable
```

---

# 87. Relationship evidence

El sistema podrá utilizar evidencia de:

```text
Foreign Keys
Unique Constraints
ORM Relationships
Entity Metadata
Explicit Join Metadata
```

---

# 88. Foreign key evidence

Ejemplo:

```text
orders.user_id
→ FK users.id
```

y:

```sql
ON orders.user_id = users.id
```

puede reconocerse como:

```text
ForeignKeyBackedJoin
```

---

# 89. Foreign key ≠ required join

La existencia de una FK no significa que VoltStack deba insertar automáticamente el JOIN.

---

# 90. ORM relationship evidence

Una relación ORM:

```text
User
hasMany
Orders
```

puede enriquecer el semantic join.

---

# 91. ORM independence

El Query Engine deberá funcionar sin ORM.

Por tanto:

```text
ORM Relationship Metadata
=
optional semantic evidence
```

---

# 92. RelationshipDescriptor

Modelo conceptual:

```text
RelationshipDescriptor
├── relationshipId
├── sourceRelation
├── targetRelation
├── keyPairs
├── cardinality
├── optionality
├── uniqueness
├── provenance
└── confidence
```

---

# 93. Cardinality evidence

Puede describir:

```text
ONE_TO_ONE
ONE_TO_MANY
MANY_TO_ONE
MANY_TO_MANY
UNKNOWN
```

---

# 94. Cardinality ≠ runtime row count

`ONE_TO_MANY` describe una propiedad relacional.

No significa:

```text
exactly N rows
```

---

# 95. Join cardinality estimation

La estimación cuantitativa pertenece al Planner/Optimizer.

---

# 96. Uniqueness evidence

Si:

```text
users.id
```

es UNIQUE/PK, el sistema puede registrar esa propiedad.

---

# 97. Constraint analysis integration

El documento 41 utilizará esta evidencia para razonar sobre:

```text
functional dependencies
uniqueness
equalities
nullability
cardinality constraints
```

---

# 98. Join predicate decomposition

Un:

```text
AND
```

puede analizarse en conjuncts.

Ejemplo:

```sql
ON a.user_id = b.user_id
AND a.region = b.region
```

produce dos dependencias semánticas.

---

# 99. Composite join key

Podrá reconocerse:

```text
CompositeJoinKey
├── a.user_id = b.user_id
└── a.region  = b.region
```

---

# 100. No arbitrary AND reordering

La descomposición no implica reordenar evaluación.

Especialmente con predicates:

```text
volatile
extension-defined
raw
```

---

# 101. RawPredicate

Un RawPredicate será tratado como:

```text
semantic barrier
```

salvo metadata explícita de extensión.

---

# 102. Raw join dependencies

Si las dependencias no pueden determinarse:

```text
DependencyKnowledge = UNKNOWN
```

---

# 103. Unknown ≠ none

Nunca:

```text
cannot determine dependencies
→ no dependencies
```

---

# 104. Join nullability model

Se requiere:

```text
JoinNullabilityEffect
```

---

# 105. INNER JOIN nullability

Por defecto no null-extends ninguno de los lados.

---

# 106. LEFT JOIN nullability

```text
Right relation
→ null-extended
```

---

# 107. RIGHT JOIN nullability

```text
Left relation
→ null-extended
```

---

# 108. FULL JOIN nullability

```text
Left + Right
→ null-extended
```

---

# 109. Nested outer joins

Ejemplo:

```sql
A
LEFT JOIN B ON ...
LEFT JOIN C ON B.id = C.b_id
```

requiere propagación cuidadosa.

---

# 110. Nullability propagation

```text
Schema Nullability
        │
        ▼
Relation Nullability
        │
        ▼
Join Nullability Effects
        │
        ▼
Query Output Nullability
```

---

# 111. Query nullability ≠ schema nullability

Esta distinción será obligatoria.

---

# 112. Join predicate type validation

Un `ON` deberá producir semántica de predicate.

No será válido:

```sql
ON orders.total
```

salvo que el sistema de tipos/plataforma tenga una regla booleana explícita compatible.

---

# 113. Join operand compatibility

```text
UserId = UserId
```

válido.

```text
UserId = OrderId
```

inválido por defecto.

---

# 114. Type inference integration

Relation Resolution consume:

```text
QueryTypeTable
```

producida por documento 39.

---

# 115. Feedback semantic loop

Algunas relaciones pueden aportar constraints adicionales.

Por tanto el Semantic Analyzer podrá coordinar fases hasta alcanzar un artifact consistente.

Pero:

```text
Relation Resolver
```

no deberá invocar recursivamente al Type Inference Engine de manera arbitraria.

---

# 116. Semantic orchestration

La coordinación pertenece a:

```text
SemanticAnalysisEngine
```

---

# 117. Join scope

Cada JOIN puede introducir un scope específico para resolver su lado derecho y su predicate.

---

# 118. Left-to-right visibility

La visibilidad dependerá de:

```text
join form
lateral semantics
subquery boundaries
```

y no de variables globales.

---

# 119. JoinScope

Modelo:

```text
JoinScope
├── leftVisibleRelations
├── rightVisibleRelations
├── outerRelations
├── lateralPermissions
└── parentScope
```

---

# 120. Join result relation

Conceptualmente un JOIN produce:

```text
JoinResultRelation
```

con el output combinado.

---

# 121. Join output

```text
JOIN(A, B)
→ Output(A) + Output(B)
```

aplicando:

```text
visibility
nullability
aliasing
```

---

# 122. USING

Para SQL/API que modele:

```sql
JOIN orders USING (user_id)
```

debe existir una representación semántica estructurada.

---

# 123. USING ≠ raw ON

No convertir inmediatamente a string SQL.

---

# 124. UsingJoinSpecification

Modelo:

```text
UsingJoinSpecification
├── columnNames
├── resolvedPairs
└── outputMergeSemantics
```

---

# 125. USING output semantics

`USING` puede afectar cómo aparecen columnas duplicadas en el output.

Esto deberá modelarse explícitamente.

---

# 126. NATURAL JOIN

Si VoltStack decide soportarlo:

```text
NATURAL JOIN
```

deberá resolverse mediante columnas comunes conocidas.

---

# 127. Recomendación V1

No promover `NATURAL JOIN` como API principal debido a su dependencia implícita del schema.

Puede existir como feature avanzada.

---

# 128. Natural join schema sensitivity

Un cambio de schema puede modificar silenciosamente la semántica de NATURAL JOIN.

Por tanto deberá:

```text
participar en schema fingerprint
```

y generar diagnóstico apropiado.

---

# 129. Join metadata

Puede incluir:

```text
origin
relationship evidence
semantic classification
capability requirements
portability
provenance
diagnostic source
```

pero nunca:

```text
physical algorithm
connection
statement
cursor
```

---

# 130. Join provenance

Ejemplo:

```text
Join J1
origin: ORM relationship User.orders
source: Repository UserRepository
relationship: users.id → orders.user_id
```

---

# 131. Explicit join vs generated join

Debe distinguirse:

```text
EXPLICIT
ORM_GENERATED
POLICY_GENERATED
EXTENSION_GENERATED
OPTIMIZER_GENERATED
```

---

# 132. Optimizer-generated join

No aparece todavía durante esta fase inicial, pero el modelo semántico debe poder representar joins derivados posteriormente.

---

# 133. Security policy joins

Una integración podría agregar joins para políticas.

Ejemplo conceptual:

```text
resource
JOIN access_control
```

pero deberá ser una transformación explícita y auditable.

---

# 134. Multitenancy

Una integración multitenant podría aportar:

```text
tenant predicate
tenant relation
partition requirement
```

pero el core no dependerá de:

```text
Tenant
TenantManager
CurrentTenant
```

---

# 135. RelationPolicyAdapter

Las integraciones deberán producir estructuras semánticas normales:

```text
RelationNode
PredicateNode
QueryMetadata
```

no side channels.

---

# 136. Query security

Los joins generados por seguridad deberán ser visibles en:

```text
provenance
diagnostics
semantic graph
```

aunque puedan ocultarse en APIs de alto nivel.

---

# 137. Relation requirements

Una relación puede generar:

```text
CapabilityRequirementSet
```

---

# 138. Ejemplos

```text
LATERAL
table function
JSON table
platform-specific relation
recursive CTE
```

---

# 139. Portability

Cada relation/join feature podrá clasificarse:

```text
PORTABLE
CAPABILITY_DEPENDENT
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
RAW
```

---

# 140. No fake portability

Si un motor no soporta determinada semántica:

```text
UnsupportedRelationCapabilityException
```

o el Planner podrá seleccionar una emulación formal cuando exista.

---

# 141. Semantic emulation

Relation Resolution sólo podrá declarar:

```text
emulation candidate / requirement
```

La estrategia concreta pertenece a fases posteriores.

---

# 142. Relation fingerprint

Cada relation artifact deberá ser fingerprintable.

---

# 143. Fingerprint inputs

Podrá incluir:

```text
source kind
resolved schema object
alias semantics
output structure
dependencies
join type
join predicate fingerprint
correlation
capability requirements
relationship evidence when semantic
schema snapshot fingerprint
extension versions
```

---

# 144. Fingerprint exclusions

No incluir:

```text
runtime parameter values
connection IDs
server process IDs
request IDs
telemetry span IDs
```

---

# 145. RelationGraph fingerprint

Conceptualmente:

```text
RelationGraphFingerprint
=
Relations
+
Joins
+
Dependencies
+
Correlations
+
Semantic Join Predicates
+
Nullability Effects
+
Relevant Schema Fingerprint
+
Relevant Extension Versions
```

---

# 146. Determinism

Misma:

```text
Query AST
Schema Snapshot
Symbol Table
Type Table
Capability Snapshot
Policy Snapshot
```

debe producir el mismo:

```text
RelationGraph
```

---

# 147. RelationResolutionContext

Contexto específico:

```text
RelationResolutionContext
├── SymbolResolutionTable
├── SchemaResolutionTable
├── QueryTypeTable
├── QuerySchemaView
├── CapabilitySnapshot
├── PolicySnapshot
├── DiagnosticSink
├── ResolutionBudget
└── ExtensionRegistry
```

---

# 148. RelationResolutionState

Estado temporal:

```text
RelationResolutionState
├── relationBuilder
├── joinBuilder
├── dependencyBuilder
├── correlationBuilder
├── visibilityStack
├── nullabilityBuilder
├── lineageBuilder
├── workQueue
└── counters
```

---

# 149. Context vs state

```text
Context
=
stable operation inputs

State
=
temporary resolution progress
```

---

# 150. No runtime resources

El contexto no contendrá:

```text
PDO
Connection
ConnectionLease
Transaction
EntityManager
UnitOfWork
Request
Session
AuthenticatedUser
Tenant Entity
```

---

# 151. ResolutionBudget

Debe limitar:

```text
maxRelations
maxJoins
maxNestedSubqueries
maxCorrelationDepth
maxDependencies
maxOutputColumns
maxJoinPredicates
maxLineageEdges
maxExtensionSteps
```

---

# 152. Budget exceeded

Produce:

```text
RelationResolutionBudgetExceededException
```

---

# 153. Resource governance

Una consulta no deberá poder crear accidentalmente:

```text
millions of relation graph edges
```

durante compilación/análisis.

---

# 154. RelationResolutionResult

Artifact:

```text
RelationResolutionResult
├── RelationGraph
├── RelationOutputTable
├── JoinResolutionTable
├── CorrelationTable
├── DependencyGraph
├── NullabilityEffects
├── RelationshipEvidenceTable
├── Diagnostics
└── Fingerprint
```

---

# 155. RelationOutputTable

Mapea:

```text
RelationId
→ RelationOutput
```

---

# 156. JoinResolutionTable

Mapea:

```text
JoinNodeId
→ SemanticJoin
```

---

# 157. CorrelationTable

Mapea:

```text
SubqueryId
→ CorrelationDescriptor
```

---

# 158. RelationshipEvidenceTable

Mapea:

```text
JoinId
→ RelationshipEvidence[]
```

---

# 159. Semantic annotations

Los resultados no deberán almacenarse mutando:

```text
JoinNode
RelationNode
ColumnNode
```

---

# 160. Side tables

Preferencia:

```text
Immutable AST
+
Immutable Semantic Side Tables
```

---

# 161. Persistent runtime safety

Shared:

```text
Frozen Relation Rule Registry
Frozen Extension Registry
Stateless Resolvers
```

Operation-scoped:

```text
RelationResolutionContext
RelationResolutionState
RelationGraphBuilder
VisibilityStack
DependencyBuilder
```

---

# 162. Concurrent queries

```text
Fiber A
└── RelationGraph A

Fiber B
└── RelationGraph B
```

sin estado compartido mutable.

---

# 163. No current relation global

Nunca:

```php
RelationResolver::$currentRelation
```

---

# 164. Extension architecture

Extensiones podrán registrar:

```text
RelationSourceResolver
JoinSemanticResolver
RelationOutputResolver
RelationDependencyResolver
RelationshipEvidenceProvider
```

---

# 165. Extension descriptor

Cada extensión declarará:

```text
supportedNodeKinds
priority
capabilityRequirements
portability
fingerprintVersion
semanticVersion
```

---

# 166. Frozen extension registry

Después del bootstrap:

```text
RelationResolutionExtensionRegistry
```

será immutable.

---

# 167. Extension conflict

Dos extensiones incompatibles producen:

```text
RelationExtensionConflictException
```

---

# 168. No arbitrary object metadata

Las extensiones no podrán guardar servicios/runtime resources dentro del RelationGraph.

---

# 169. Error taxonomy

```text
RelationResolutionException
UnresolvedRelationException
DuplicateRelationAliasException
AmbiguousRelationReferenceException
AmbiguousColumnReferenceException
InvalidJoinPredicateException
InvalidJoinScopeException
InvalidLateralReferenceException
InvalidCorrelationException
CircularRelationDependencyException
IncompatibleJoinOperandException
UnsupportedJoinTypeException
UnsupportedRelationCapabilityException
InvalidDerivedRelationException
InvalidCteRelationException
InvalidValuesRelationException
RelationOutputConflictException
RelationExtensionConflictException
RelationResolutionBudgetExceededException
```

---

# 170. Diagnostic codes

```text
DB-REL-001 UNRESOLVED_RELATION
DB-REL-002 DUPLICATE_RELATION_ALIAS
DB-REL-003 AMBIGUOUS_RELATION
DB-REL-004 AMBIGUOUS_COLUMN
DB-REL-005 INVALID_JOIN_PREDICATE
DB-REL-006 INVALID_JOIN_SCOPE
DB-REL-007 INVALID_LATERAL_REFERENCE
DB-REL-008 INVALID_CORRELATION
DB-REL-009 CIRCULAR_DEPENDENCY
DB-REL-010 INCOMPATIBLE_JOIN_OPERANDS
DB-REL-011 UNSUPPORTED_JOIN_TYPE
DB-REL-012 UNSUPPORTED_RELATION_CAPABILITY
DB-REL-013 INVALID_DERIVED_RELATION
DB-REL-014 INVALID_CTE_RELATION
DB-REL-015 INVALID_VALUES_RELATION
DB-REL-016 RELATION_OUTPUT_CONFLICT
DB-REL-017 EXTENSION_CONFLICT
DB-REL-018 RESOLUTION_BUDGET_EXCEEDED
```

---

# 171. Ejemplo diagnóstico

```text
DB-REL-007 INVALID_LATERAL_REFERENCE

Derived relation "recent_orders" references outer relation "u",
but the relation is not declared as lateral.

Reference:
    u.id

Referenced from:
    recent_orders

Suggestion:
    declare the derived relation as LATERAL
    or remove the outer relation reference.
```

---

# 172. Testing

El sistema deberá cubrir:

```text
single relation
multiple relations
aliases
self joins
inner joins
outer joins
cross joins
derived relations
CTEs
recursive CTEs
VALUES
table functions
correlated subqueries
lateral joins
ambiguous columns
duplicate aliases
nullability propagation
relationship evidence
foreign keys
domain type compatibility
extension relations
persistent runtime
concurrency
budgets
fingerprints
```

---

# 173. Self join test

```sql
SELECT manager.id, employee.id
FROM users manager
JOIN users employee
    ON employee.manager_id = manager.id
```

debe producir:

```text
SchemaObject users
├── R1 manager
└── R2 employee
```

---

# 174. LEFT JOIN nullability test

```text
orders.id schema:
NON_NULL

after LEFT JOIN:
orders.id query:
NULLABLE
```

---

# 175. Correlation test

```sql
SELECT *
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
```

debe registrar:

```text
Correlation:
S1 → R1.users
```

---

# 176. Lateral test

Validar que:

```text
LATERAL
```

permita referencias que una derived relation ordinaria no permite.

---

# 177. Domain mismatch test

```text
UserId = OrderId
```

debe fallar cuando la política domain sea estricta.

---

# 178. Persistent worker test

```text
Request A
users + orders
→ RelationGraph A

cleanup

Request B
products + categories
→ RelationGraph B
```

Ninguna relación de A puede sobrevivir en B.

---

# 179. Concurrency test

```text
Coroutine A
alias u → users

Coroutine B
alias u → units
```

sin colisión.

---

# 180. Architecture test

El namespace no podrá importar:

```text
PDO
NativeConnection
ConnectionLease
QueryExecutor
EntityManager
UnitOfWork
IdentityMap
Http\Request
Session
ServiceContainer
```

---

# 181. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Semantic\Relation\
```

---

# 182. Estructura propuesta

```text
Query/
└── Semantic/
    └── Relation/
        ├── Contract/
        │   ├── RelationResolverInterface.php
        │   ├── RelationSourceResolverInterface.php
        │   ├── JoinResolverInterface.php
        │   └── RelationshipEvidenceProviderInterface.php
        │
        ├── Core/
        │   ├── RelationResolutionEngine.php
        │   ├── RelationResolver.php
        │   ├── JoinResolver.php
        │   └── RelationOutputResolver.php
        │
        ├── Model/
        │   ├── RelationId.php
        │   ├── RelationInstance.php
        │   ├── RelationSource.php
        │   ├── RelationOutput.php
        │   ├── RelationOutputColumn.php
        │   └── OutputColumnId.php
        │
        ├── Source/
        │   ├── TableRelation.php
        │   ├── ViewRelation.php
        │   ├── CteRelation.php
        │   ├── DerivedRelation.php
        │   ├── SubqueryRelation.php
        │   ├── ValuesRelation.php
        │   ├── TableFunctionRelation.php
        │   └── ExtensionRelation.php
        │
        ├── Join/
        │   ├── JoinId.php
        │   ├── SemanticJoin.php
        │   ├── SemanticJoinType.php
        │   ├── JoinKeyCandidate.php
        │   ├── CompositeJoinKey.php
        │   ├── JoinPredicateClassification.php
        │   ├── JoinNullabilityEffect.php
        │   └── UsingJoinSpecification.php
        │
        ├── Graph/
        │   ├── RelationGraph.php
        │   ├── RelationGraphBuilder.php
        │   ├── RelationDependencyGraph.php
        │   └── RelationGraphFingerprint.php
        │
        ├── Scope/
        │   ├── RelationScope.php
        │   ├── JoinScope.php
        │   ├── RelationVisibility.php
        │   └── RelationVisibilityStack.php
        │
        ├── Dependency/
        │   ├── RelationDependency.php
        │   ├── RelationDependencyBuilder.php
        │   └── CircularDependencyDetector.php
        │
        ├── Correlation/
        │   ├── CorrelationDescriptor.php
        │   ├── CorrelationTable.php
        │   └── CorrelationResolver.php
        │
        ├── Relationship/
        │   ├── RelationshipDescriptor.php
        │   ├── RelationshipEvidence.php
        │   ├── RelationshipEvidenceTable.php
        │   └── RelationshipCardinality.php
        │
        ├── Nullability/
        │   ├── JoinNullabilityResolver.php
        │   └── RelationNullabilityTable.php
        │
        ├── Lineage/
        │   ├── RelationLineage.php
        │   ├── OutputLineage.php
        │   └── RelationLineageBuilder.php
        │
        ├── Context/
        │   ├── RelationResolutionContext.php
        │   ├── RelationResolutionState.php
        │   └── RelationResolutionBudget.php
        │
        ├── Result/
        │   ├── RelationResolutionResult.php
        │   ├── RelationOutputTable.php
        │   └── JoinResolutionTable.php
        │
        ├── Extension/
        │   ├── RelationResolutionExtensionRegistry.php
        │   └── RelationResolutionExtensionDescriptor.php
        │
        ├── Diagnostic/
        │   ├── RelationDiagnostic.php
        │   └── RelationDiagnosticCode.php
        │
        └── Exception/
            ├── RelationResolutionException.php
            ├── UnresolvedRelationException.php
            ├── DuplicateRelationAliasException.php
            ├── AmbiguousColumnReferenceException.php
            ├── InvalidJoinPredicateException.php
            ├── InvalidLateralReferenceException.php
            ├── CircularRelationDependencyException.php
            └── RelationResolutionBudgetExceededException.php
```

---

# 183. Invariantes arquitectónicos

## DB-REL-001

Relation Resolution no generará SQL.

## DB-REL-002

No ejecutará consultas.

## DB-REL-003

No abrirá conexiones.

## DB-REL-004

No dependerá de PDO.

## DB-REL-005

No dependerá del Driver.

## DB-REL-006

No dependerá del Executor.

## DB-REL-007

No seleccionará algoritmos físicos de JOIN.

## DB-REL-008

No seleccionará índices.

## DB-REL-009

No realizará join ordering por costo.

## DB-REL-010

No realizará predicate pushdown.

## DB-REL-011

Schema Table y Query Relation serán conceptos distintos.

## DB-REL-012

Cada instancia relacional tendrá RelationId propio.

## DB-REL-013

Self joins producirán RelationIds diferentes.

## DB-REL-014

Relation aliases serán scope-aware.

## DB-REL-015

RelationAlias será distinto de ProjectionAlias.

## DB-REL-016

Duplicate aliases serán rechazados.

## DB-REL-017

Ambiguous column references serán rechazadas.

## DB-REL-018

Unknown dependency no significará ausencia de dependency.

## DB-REL-019

Join type será semántico.

## DB-REL-020

Physical join algorithm pertenecerá al Planner.

## DB-REL-021

INNER JOIN no será automáticamente reordenado.

## DB-REL-022

OUTER JOIN semantics serán preservadas.

## DB-REL-023

LEFT JOIN null-extend el lado derecho.

## DB-REL-024

RIGHT JOIN null-extend el lado izquierdo.

## DB-REL-025

FULL JOIN null-extend ambos lados.

## DB-REL-026

Query nullability será distinta de schema nullability.

## DB-REL-027

Join predicates usarán PredicateNode.

## DB-REL-028

Join predicate type compatibility será validada.

## DB-REL-029

SQL three-valued logic será preservada.

## DB-REL-030

NOT IN no será convertido ingenuamente a anti join.

## DB-REL-031

Correlation será explícita.

## DB-REL-032

Correlation no implicará ejecución por fila.

## DB-REL-033

Decorrelation pertenecerá al Optimizer.

## DB-REL-034

Lateral dependencies serán explícitas.

## DB-REL-035

Non-lateral relations no podrán acceder arbitrariamente a outer relations.

## DB-REL-036

Capability checks sustituirán vendor checks.

## DB-REL-037

Foreign Key no será JOIN.

## DB-REL-038

ORM Relationship no será JOIN.

## DB-REL-039

Foreign Keys podrán aportar relationship evidence.

## DB-REL-040

ORM relationships podrán aportar evidencia opcional.

## DB-REL-041

Query Engine funcionará sin ORM.

## DB-REL-042

Cardinality metadata no será row-count estimation.

## DB-REL-043

Physical cardinality estimation pertenecerá al Planner.

## DB-REL-044

Relation outputs tendrán identidad propia.

## DB-REL-045

Derived outputs tendrán lineage.

## DB-REL-046

Lineage no mutará AST.

## DB-REL-047

RelationGraph será artifact separado del AST.

## DB-REL-048

RelationGraph publicado será immutable.

## DB-REL-049

Semantic annotations usarán side tables.

## DB-REL-050

Relation resolution será determinista.

## DB-REL-051

RelationGraph será fingerprintable.

## DB-REL-052

Fingerprints excluirán runtime values.

## DB-REL-053

Schema-sensitive semantics afectarán fingerprint.

## DB-REL-054

NATURAL JOIN, si existe, será schema-sensitive.

## DB-REL-055

USING tendrá representación estructurada.

## DB-REL-056

USING no será raw SQL.

## DB-REL-057

CROSS JOIN preservará identidad estructural.

## DB-REL-058

Raw predicates serán semantic barriers.

## DB-REL-059

Raw dependency knowledge podrá ser UNKNOWN.

## DB-REL-060

No se inventarán dependencies.

## DB-REL-061

Subquery outputs serán relaciones estructuradas.

## DB-REL-062

CTEs tendrán scope explícito.

## DB-REL-063

Recursive CTE dependencies serán bounded.

## DB-REL-064

VALUES podrá actuar como relation source.

## DB-REL-065

Table functions serán capability-aware.

## DB-REL-066

Extension relations usarán contratos formales.

## DB-REL-067

Extension registry será frozen tras bootstrap.

## DB-REL-068

Extension conflicts serán errores explícitos.

## DB-REL-069

RelationResolutionContext será operation-scoped.

## DB-REL-070

RelationResolutionState será operation-scoped.

## DB-REL-071

No existirá global current relation.

## DB-REL-072

Persistent workers no compartirán mutable resolution state.

## DB-REL-073

Concurrent queries tendrán estados independientes.

## DB-REL-074

Relation resolution tendrá resource budget.

## DB-REL-075

Budget exhaustion no producirá artifact válido parcial.

## DB-REL-076

Relation graph cycles inválidos serán detectados.

## DB-REL-077

Relationship evidence tendrá provenance.

## DB-REL-078

Security-generated relations tendrán provenance.

## DB-REL-079

Tenant integrations no introducirán dependencia core a Tenant.

## DB-REL-080

Authorization integrations no introducirán dependencia core a User/Session.

## DB-REL-081

Semantic join classification no determinará implementación física.

## DB-REL-082

Equi-join podrá generar JoinKeyCandidate.

## DB-REL-083

JoinKeyCandidate no será physical hash key.

## DB-REL-084

Composite joins conservarán todas sus key pairs.

## DB-REL-085

Volatile predicates limitarán transformaciones posteriores.

## DB-REL-086

Relation Resolution preservará AST join order.

## DB-REL-087

Join associativity no se asumirá universalmente.

## DB-REL-088

Join commutativity no se asumirá universalmente.

## DB-REL-089

Outer join semantics tendrán precedence sobre optimizaciones futuras.

## DB-REL-090

Correlation depth será explícita.

## DB-REL-091

Relation visibility será scope-driven.

## DB-REL-092

Shadowing será determinista.

## DB-REL-093

No se realizará hidden schema I/O.

## DB-REL-094

QuerySchemaView será read-only.

## DB-REL-095

Relation resolution podrá ejecutarse offline.

## DB-REL-096

CapabilitySnapshot será estable durante la operación.

## DB-REL-097

Type information será consumida mediante QueryTypeTable.

## DB-REL-098

Relation Resolution no será dueño del Type Inference System.

## DB-REL-099

SemanticAnalysisEngine coordinará dependencias entre fases.

## DB-REL-100

El resultado deberá describir completamente la topología relacional semántica antes de entrar al Constraint Analysis System.

---

# 184. Anti-patrones

### 1. Resolver JOIN como SQL

```php
$join->sql = 'LEFT JOIN orders ...';
```

Incorrecto.

### 2. Guardar algoritmo físico

```php
$join->algorithm = 'hash';
```

Incorrecto en Semantic Layer.

### 3. Usar tabla como identidad relacional

```text
users == relation
```

Incorrecto para self joins.

### 4. Ignorar outer join nullability

Incorrecto.

### 5. Resolver LATERAL mediante variables globales

Incorrecto.

### 6. Convertir automáticamente FK en JOIN

Incorrecto.

### 7. Depender del ORM

Incorrecto.

### 8. Considerar correlation como nested-loop obligatorio

Incorrecto.

### 9. Reordenar joins durante resolución

Incorrecto.

### 10. Usar vendor names

```php
if ($driver === 'pgsql')
```

Incorrecto.

---

# 185. Ejemplo completo

Consulta:

```sql
SELECT
    u.id,
    u.email,
    o.id AS order_id,
    o.total
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
WHERE u.active = true;
```

Schema:

```text
users
├── id       UserId NOT NULL PK
├── email    EmailAddress NOT NULL
└── active   Boolean NOT NULL

orders
├── id       OrderId NOT NULL PK
├── user_id  UserId NOT NULL FK → users.id
└── total    Money NOT NULL
```

Resolución:

```text
Relation R1
├── source: users
├── alias: u
└── output
    ├── id     UserId NON_NULL
    ├── email  EmailAddress NON_NULL
    └── active Boolean NON_NULL

Relation R2
├── source: orders
├── alias: o
└── output
    ├── id      OrderId NON_NULL
    ├── user_id UserId NON_NULL
    └── total   Money NON_NULL
```

JOIN:

```text
Join J1
├── type: LEFT
├── left: R1
├── right: R2
├── predicate:
│   └── R2.user_id = R1.id
├── classification:
│   └── EQUI_JOIN
├── relationship evidence:
│   └── FK orders.user_id → users.id
└── nullability effect:
    └── R2 → NULL_EXTENDED
```

Output contextual:

```text
R1.id      UserId       NON_NULL
R1.email   EmailAddress NON_NULL
R1.active  Boolean      NON_NULL

R2.id      OrderId      NULLABLE
R2.user_id UserId       NULLABLE
R2.total   Money        NULLABLE
```

Nótese:

```text
orders.total schema = NON_NULL
orders.total query  = NULLABLE
```

debido al LEFT JOIN.

---

# 186. Arquitectura final

```text
                 Query AST
                    │
                    ▼
             Symbol Resolution
                    │
                    ▼
           Schema-Aware Resolution
                    │
                    ▼
              Type Inference
                    │
                    ▼
        Relation Resolution Engine
                    │
       ┌────────────┼───────────────┐
       ▼            ▼               ▼
   Relations      Joins        Correlations
       │            │               │
       ├────────────┼───────────────┤
       ▼            ▼               ▼
    Outputs    Dependencies     Visibility
       │            │               │
       ├────────────┼───────────────┤
       ▼            ▼               ▼
   Lineage     Relationships    Nullability
       │            │               │
       └────────────┼───────────────┘
                    ▼
              RelationGraph
                    │
                    ▼
         Constraint Analysis
                    │
                    ▼
          Semantic Query Graph
```

---

# 187. Fórmulas maestras

## Relation Resolution

```text
Resolved Relation
=
Relation Source
+
Symbol Resolution
+
Schema Resolution
+
Type Information
+
Scope
+
Output Model
+
Lineage
+
Dependencies
```

## Semantic Join

```text
Semantic Join
=
Left Relation
+
Right Relation
+
Join Type
+
Join Predicate
+
Visibility Rules
+
Dependency Rules
+
Nullability Effects
+
Relationship Evidence
```

## Relation Graph

```text
RelationGraph
=
Relations
+
Joins
+
Dependencies
+
Correlations
+
Visibility
+
Outputs
+
Lineage
+
Nullability
+
Relationship Evidence
+
Capability Requirements
```

## Outer join output

```text
LEFT JOIN(A, B)
=
Output(A)
+
Nullable(Output(B))
```

```text
RIGHT JOIN(A, B)
=
Nullable(Output(A))
+
Output(B)
```

```text
FULL JOIN(A, B)
=
Nullable(Output(A))
+
Nullable(Output(B))
```

## Correlation

```text
Correlation
=
Inner Query Reference
+
Outer Scope Symbol
+
Scope Distance
+
Visibility Permission
+
Dependency Edge
```

---

# 188. Invariante central

> El Relation and Join Resolution System debe describir completamente la topología relacional lógica de una consulta sin decidir todavía cómo será ejecutada.

Por tanto:

```text
Relation Resolution
=
What relations participate
+
How they are semantically connected
```

mientras que:

```text
Query Planner
=
How those relations will be physically accessed and combined
```

Esta frontera permite que VoltStack mantenga:

```text
Semantic Correctness
        │
        ▼
Logical Optimization
        │
        ▼
Physical Planning
        │
        ▼
SQL Compilation
```

como responsabilidades independientes.

---

# 189. Estado del Semantic Query Engine

```text
35 Semantic Query Architecture
          │
          ▼
36 Semantic Analysis
          │
          ▼
37 Symbol Resolution
          │
          ▼
38 Schema-Aware Resolution
          │
          ▼
39 Query Type Inference
          │
          ▼
40 Relation / Join Resolution
          │
          ▼
41 Query Constraint Analysis
          │
          ▼
42 Semantic Query Graph
```

Con el documento 40, VoltStack ya puede conocer no sólo:

```text
qué significa cada símbolo
```

y:

```text
qué tipo tiene cada expresión
```

sino también:

```text
qué relaciones existen
cómo se conectan
qué dependencias poseen
qué joins son outer
qué outputs se vuelven nullable
qué subqueries están correlacionadas
qué relationships respaldan los joins
```

antes de realizar cualquier optimización.

---

# 190. Próximo documento

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

El siguiente documento formalizará el sistema que deriva conocimiento lógico adicional a partir de:

```text
Predicates
+
Types
+
Schema Constraints
+
Relation Graph
+
Equality Conditions
+
NULL Conditions
+
Unique Keys
+
Foreign Keys
+
Ranges
+
Constants
```

para construir conocimiento como:

```text
u.id = o.user_id
o.user_id = :id
─────────────────
u.id = :id
```

o:

```text
users.id PRIMARY KEY
────────────────────
users.id is UNIQUE
users.id is NON_NULL
```

manteniendo siempre la frontera:

```text
Constraint Analysis
≠
Query Optimization
```

El sistema descubrirá **hechos semánticos demostrables**; el Optimizer decidirá posteriormente cómo aprovecharlos.