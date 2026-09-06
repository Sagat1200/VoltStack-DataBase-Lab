# 41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md

# VoltStack Quantum Database
## Query Constraint Analysis System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 41 — Query Constraint Analysis System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

El **Query Constraint Analysis System** analiza las restricciones lógicas y estructurales conocidas de una consulta para derivar hechos semánticos adicionales que puedan ser utilizados posteriormente por:

- Semantic Query Graph;
- Query Optimizer;
- Query Planner;
- Query Validator;
- Query Diagnostics;
- N+1 analysis;
- ORM;
- Persistence Engine;
- security policies;
- static analysis tooling.

El sistema recibe conocimiento proveniente de:

```text
Predicates
+
Query Types
+
Schema Constraints
+
Relation Graph
+
Primary Keys
+
Unique Constraints
+
Foreign Keys
+
NULL Constraints
+
Equality Predicates
+
Range Predicates
+
Constants
+
Domain Metadata
```

y produce:

```text
Semantic Constraint Model
+
Derived Semantic Facts
```

---

# 2. Ejemplo fundamental

Consulta:

```sql
SELECT *
FROM users u
JOIN orders o
    ON o.user_id = u.id
WHERE o.user_id = :userId;
```

El sistema conoce:

```text
o.user_id = u.id
o.user_id = :userId
```

y puede derivar:

```text
u.id = :userId
```

sin modificar todavía la consulta.

---

# 3. Regla maestra

> Constraint Analysis descubre hechos que pueden demostrarse a partir de la consulta y del schema; no decide todavía si dichos hechos deben utilizarse para reescribir u optimizar la consulta.

Por tanto:

```text
Constraint Analysis
≠
Query Optimization
```

y:

```text
Derived Fact
≠
Query Rewrite
```

---

# 4. Separaciones fundamentales

VoltStack deberá mantener:

```text
Constraint
≠
Predicate
≠
Derived Fact
≠
Optimization Rule
≠
Execution Decision
```

Asimismo:

```text
Schema Constraint
≠
Query Constraint
```

y:

```text
Logical Equality
≠
PHP Equality
```

---

# 5. Ubicación en el pipeline

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
Relation / Join Resolution
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
```

---

# 6. Entradas

El sistema podrá consumir:

```text
Normalized Query AST
SymbolResolutionTable
SchemaResolutionTable
QueryTypeTable
NullabilityTable
RelationGraph
QuerySchemaView
CapabilitySnapshot
PolicySnapshot
QueryMetadata
```

---

# 7. Salidas

La salida principal será:

```text
QueryConstraintAnalysisResult
```

conteniendo:

```text
ConstraintGraph
DerivedFactTable
EqualityClasses
NullabilityFacts
RangeFacts
UniquenessFacts
FunctionalDependencies
RelationshipFacts
Contradictions
CapabilityRequirements
Diagnostics
Fingerprint
```

---

# 8. Constraint sources

Las restricciones pueden originarse en:

```text
QUERY
SCHEMA
TYPE_SYSTEM
RELATIONSHIP
ORM_METADATA
DOMAIN_METADATA
POLICY
EXTENSION
DERIVED
```

---

# 9. Constraint provenance

Toda constraint deberá conservar:

```text
ConstraintProvenance
```

para responder:

```text
Where did this fact come from?
```

---

# 10. Ejemplo de provenance

```text
Fact:
u.id = :id

Derived from:
C1 o.user_id = u.id
C2 o.user_id = :id

Rule:
Equality Transitivity
```

---

# 11. ConstraintId

Cada constraint tendrá:

```text
ConstraintId
```

estable dentro de la operación.

---

# 12. FactId

Cada hecho derivado tendrá:

```text
FactId
```

independiente del `ConstraintId`.

---

# 13. Constraint taxonomy

V1 deberá soportar:

```text
EqualityConstraint
InequalityConstraint
NullConstraint
NonNullConstraint
RangeConstraint
ConstantConstraint
TypeConstraint
UniquenessConstraint
PrimaryKeyConstraint
ForeignKeyConstraint
FunctionalDependencyConstraint
CardinalityConstraint
RelationshipConstraint
DomainConstraint
PredicateConstraint
CapabilityConstraint
ExtensionConstraint
```

---

# 14. Equality constraints

Ejemplo:

```text
A = B
```

podrá generar una:

```text
EqualityConstraint
```

---

# 15. Equality class

Igualdades compatibles podrán agruparse en:

```text
EqualityClass
```

---

# 16. Ejemplo

Dados:

```text
A = B
B = C
C = :p
```

se obtiene:

```text
EqualityClass E1
├── A
├── B
├── C
└── :p
```

---

# 17. EqualityClassId

Cada clase tendrá:

```text
EqualityClassId
```

---

# 18. Equality transitivity

Cuando sea semánticamente válida:

```text
A = B
B = C
───────
A = C
```

---

# 19. SQL NULL caveat

La transitividad no podrá ignorar SQL three-valued logic.

Si los operands pueden ser NULL, el sistema deberá distinguir:

```text
value equality
```

de:

```text
guaranteed equality fact
```

---

# 20. Equality certainty

Una igualdad podrá clasificarse como:

```text
GUARANTEED
CONDITIONAL
NULL_SENSITIVE
UNKNOWN
```

---

# 21. Predicate truth context

Una igualdad en:

```text
WHERE
```

tiene consecuencias diferentes de una igualdad contenida en:

```text
SELECT CASE
```

porque las filas que sobreviven al `WHERE` satisfacen `TRUE`.

---

# 22. WHERE refinement

Para filas supervivientes:

```sql
WHERE x = 10
```

puede derivarse:

```text
x = 10
x IS NOT NULL
```

dentro del scope posterior al filtro.

---

# 23. ON context

Una condición en:

```text
LEFT JOIN ... ON ...
```

no permite aplicar automáticamente las mismas conclusiones al resultado completo.

---

# 24. Ejemplo crítico

```sql
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
```

No puede concluirse globalmente:

```text
o.user_id IS NOT NULL
```

porque las filas null-extended del LEFT JOIN contienen NULL.

---

# 25. Constraint scope

Toda constraint tendrá:

```text
ConstraintScope
```

---

# 26. Scope examples

```text
QUERY
WHERE_FILTERED_ROWS
JOIN_MATCH
RELATION
SUBQUERY
CTE
AGGREGATE_GROUP
HAVING_FILTERED_GROUPS
BRANCH
EXTENSION
```

---

# 27. Scoped facts

Un hecho puede ser válido únicamente dentro de un scope.

Ejemplo:

```text
WHERE x IS NOT NULL
```

produce:

```text
x NON_NULL
```

para:

```text
rows after WHERE
```

no necesariamente para el schema original.

---

# 28. Fact validity

Cada fact podrá contener:

```text
validityScope
certainty
provenance
conditions
```

---

# 29. ConstraintGraph

La representación central será:

```text
ConstraintGraph
```

---

# 30. Graph structure

```text
ConstraintGraph
├── Subjects
├── Constraints
├── EqualityClasses
├── Dependencies
├── DerivedFacts
├── ScopeEdges
└── ProvenanceEdges
```

---

# 31. ConstraintSubject

Un subject puede ser:

```text
ColumnSymbol
Expression
Parameter
Projection
Relation
Tuple
Constant
DomainValue
OutputColumn
```

---

# 32. ConstraintSubjectId

Se utilizará:

```text
ConstraintSubjectId
```

para evitar acoplar el grafo directamente a objetos mutables.

---

# 33. Constant constraints

Ejemplo:

```sql
WHERE status = 'active'
```

puede producir:

```text
ConstantFact
subject: status
value: semantic literal active
```

---

# 34. Sensitive constants

Los valores sensibles no deberán copiarse indiscriminadamente a diagnostics, telemetry o fingerprints.

---

# 35. Runtime parameter distinction

```sql
WHERE id = :id
```

no significa que el analyzer conozca el valor de `:id`.

Conoce:

```text
id = Parameter(:id)
```

---

# 36. Parameter facts

Puede derivar:

```text
:id type = UserId
:id NON_NULL
```

si la semántica lo demuestra.

Pero no:

```text
:id = 42
```

sin un literal conocido dentro del artifact analizado.

---

# 37. Null constraints

Tipos:

```text
IS_NULL
IS_NOT_NULL
MAY_BE_NULL
GUARANTEED_NON_NULL
NULL_EXTENDED
UNKNOWN
```

---

# 38. Nullability facts

Ejemplo:

```sql
WHERE email IS NOT NULL
```

produce:

```text
email → NON_NULL
```

dentro del filtered scope.

---

# 39. Schema nullability

Si schema declara:

```text
users.id NOT NULL
```

puede existir:

```text
SchemaNonNullFact(users.id)
```

---

# 40. Join nullability

Sin embargo:

```sql
LEFT JOIN users
```

puede introducir:

```text
QueryNullableFact(users.id)
```

en determinado relation output.

---

# 41. Nullability lattice

Conceptualmente:

```text
IMPOSSIBLE_NULL
      │
      ▼
CONDITIONALLY_NON_NULL
      │
      ▼
MAY_BE_NULL
      │
      ▼
UNKNOWN
```

La implementación concreta podrá usar un lattice más formal.

---

# 42. Contradictory nullability

Ejemplo:

```sql
WHERE x IS NULL
AND x IS NOT NULL
```

produce:

```text
Contradiction
```

---

# 43. Contradiction ≠ immediate rewrite

El analyzer podrá marcar:

```text
UnsatisfiableConstraintSet
```

pero no deberá reemplazar automáticamente la consulta por:

```text
WHERE FALSE
```

Eso corresponde al Optimizer.

---

# 44. Contradiction model

```text
ConstraintContradiction
├── involvedConstraints
├── scope
├── reason
├── certainty
└── provenance
```

---

# 45. Range constraints

Ejemplo:

```sql
WHERE age >= 18
AND age < 65
```

produce:

```text
RangeFact
age ∈ [18, 65)
```

---

# 46. RangeBound

Modelo:

```text
RangeBound
├── value
├── inclusive
└── type
```

---

# 47. RangeFact

```text
RangeFact
├── subject
├── lowerBound?
├── upperBound?
├── exclusions?
├── scope
└── provenance
```

---

# 48. Range intersection

Dados:

```text
x >= 10
x >= 20
```

puede derivarse:

```text
x >= 20
```

---

# 49. Range contradiction

```text
x > 20
x < 10
```

produce:

```text
UnsatisfiableRange
```

---

# 50. Range semantics

La comparación deberá respetar:

```text
Query Type
Collation
Temporal semantics
Domain semantics
Platform-dependent behavior
```

---

# 51. No PHP comparison

Nunca:

```php
$value1 < $value2
```

como regla universal de comparación semántica.

---

# 52. String ranges

El analyzer deberá ser conservador con:

```text
string collation
case sensitivity
locale
platform behavior
```

---

# 53. Numeric ranges

Podrán analizarse con mayor precisión cuando:

```text
type semantics
```

sean conocidas.

---

# 54. Temporal ranges

Ejemplo:

```sql
created_at >= :start
AND created_at < :end
```

produce constraints temporales estructurados.

---

# 55. BETWEEN

```sql
age BETWEEN 18 AND 64
```

puede convertirse semánticamente en:

```text
age >= 18
age <= 64
```

como facts derivados, sin reescribir AST.

---

# 56. NOT BETWEEN

Requiere representar:

```text
x < lower OR x > upper
```

lógicamente.

No debe simplificarse ingenuamente bajo cualquier contexto.

---

# 57. Inequality constraints

Tipos:

```text
LESS_THAN
LESS_THAN_OR_EQUAL
GREATER_THAN
GREATER_THAN_OR_EQUAL
NOT_EQUAL
DISTINCT_FROM
```

---

# 58. Distinctness

Debe mantenerse:

```text
x <> y
≠
x IS DISTINCT FROM y
```

debido a NULL.

---

# 59. Type constraints

El documento 39 produce información de tipos.

Constraint Analysis puede incorporar:

```text
TypeFact
```

pero no sustituye al Type Inference System.

---

# 60. Type ownership

```text
Type Inference
=
owner of resolved query types

Constraint Analysis
=
consumer of type facts
```

---

# 61. Type-based facts

Ejemplo:

```text
UserId
```

puede implicar:

```text
ComparableWith(UserId)
```

pero no necesariamente:

```text
NumericArithmeticAllowed
```

aunque use BIGINT físicamente.

---

# 62. Domain constraints

Los Domain Types pueden aportar:

```text
DomainConstraint
```

---

# 63. Ejemplo

```text
PositiveMoney
```

podría declarar:

```text
value >= 0
```

si esa propiedad forma parte formal del tipo.

---

# 64. Domain metadata trust

Sólo metadata declarada mediante contratos confiables podrá convertirse en constraint.

No se inferirán reglas desde nombres como:

```text
PositiveAmount
```

---

# 65. Primary key constraints

Del schema:

```text
PRIMARY KEY(users.id)
```

se derivan facts como:

```text
UNIQUE(users.id)
NON_NULL(users.id)
```

cuando la plataforma/schema semantics lo garantice.

---

# 66. Composite primary key

```text
PRIMARY KEY(tenant_id, id)
```

implica:

```text
UNIQUE(tenant_id, id)
```

No necesariamente:

```text
UNIQUE(id)
```

---

# 67. Unique constraints

Ejemplo:

```text
UNIQUE(users.email)
```

produce:

```text
UniquenessFact(users.email)
```

considerando semántica de NULL del motor.

---

# 68. NULL uniqueness caveat

Diferentes motores pueden tratar múltiples NULL en UNIQUE de forma distinta.

Por tanto:

```text
UNIQUE
```

deberá incorporar capability/platform semantics cuando sea relevante.

---

# 69. UniquenessFact

Modelo:

```text
UniquenessFact
├── subjects
├── nullSemantics
├── scope
├── provenance
└── certainty
```

---

# 70. Functional dependencies

Una dependencia funcional:

```text
A → B
```

significa:

> para un valor de A, B queda determinado dentro del scope correspondiente.

---

# 71. Primary key FD

```text
users.id
→
users.email
users.name
users.created_at
...
```

dentro de la relación `users`.

---

# 72. Composite FD

```text
(tenant_id, id)
→
remaining columns
```

---

# 73. FunctionalDependency

Modelo:

```text
FunctionalDependency
├── determinantSubjects
├── dependentSubjects
├── scope
├── provenance
└── certainty
```

---

# 74. FD ≠ foreign key

Debe mantenerse:

```text
Functional Dependency
≠
Foreign Key
```

---

# 75. Foreign key constraints

Ejemplo:

```text
orders.user_id
REFERENCES users.id
```

produce:

```text
ForeignKeyFact
```

---

# 76. ForeignKeyFact

Puede incluir:

```text
sourceColumns
targetColumns
targetUniqueness
sourceNullability
referentialActions
deferrability
provenance
```

---

# 77. FK existence semantics

La existencia de una FK puede permitir razonar sobre existencia referencial cuando:

- constraint está habilitada;
- metadata es confiable;
- semántica de NULL se considera;
- no existen estados transitorios relevantes;
- el scope es apropiado.

---

# 78. Deferred constraints

Una FK deferred no debe tratarse ingenuamente como una verdad operacional en todos los puntos de una transacción.

---

# 79. Constraint certainty

Se deberá modelar:

```text
ConstraintCertainty
```

con niveles como:

```text
GUARANTEED
DECLARED
INFERRED
CONDITIONAL
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 80. Trusted schema

Un schema snapshot confiable puede elevar:

```text
DECLARED
```

a una garantía utilizable según policy.

---

# 81. Partial schema

Metadata incompleta no significa:

```text
constraint does not exist
```

sino:

```text
constraint unknown
```

---

# 82. Closed-world vs open-world

Constraint Analysis deberá conocer la diferencia entre:

```text
Closed World Metadata
```

y:

```text
Partial/Open World Metadata
```

---

# 83. SchemaKnowledgeMode

Valores posibles:

```text
COMPLETE
PARTIAL
UNKNOWN
```

---

# 84. Relationship constraints

Del documento 40 pueden recibirse:

```text
RelationshipEvidence
```

---

# 85. Example

```text
orders.user_id = users.id
```

respaldado por FK puede producir:

```text
RelationshipFact
```

---

# 86. Relationship cardinality

Si:

```text
orders.user_id
```

no es UNIQUE:

```text
users → orders
```

puede ser:

```text
ONE_TO_MANY
```

---

# 87. Unique foreign key

Si:

```text
orders.user_id UNIQUE
```

la relación puede convertirse en:

```text
ONE_TO_ZERO_OR_ONE
```

dependiendo de nullability/existence.

---

# 88. Cardinality facts

El analyzer puede derivar cardinalidad lógica cualitativa:

```text
AT_MOST_ONE
EXACTLY_ONE
ZERO_OR_ONE
ONE_OR_MORE
ZERO_OR_MORE
UNKNOWN
```

cuando exista evidencia suficiente.

---

# 89. Cardinality ≠ cost estimate

No debe calcular aquí:

```text
estimated rows = 15342
```

Eso pertenece al Planner/Optimizer.

---

# 90. At-most-one query

Ejemplo:

```sql
SELECT *
FROM users
WHERE id = :id
```

si `id` es PK puede derivarse:

```text
ResultCardinality <= 1
```

---

# 91. Composite uniqueness

```sql
WHERE tenant_id = :tenant
AND slug = :slug
```

si:

```text
UNIQUE(tenant_id, slug)
```

entonces:

```text
ResultCardinality <= 1
```

---

# 92. Missing unique component

Si sólo:

```text
tenant_id = :tenant
```

no puede derivirse `<= 1`.

---

# 93. Constant propagation

Dados:

```text
A = B
B = 10
```

puede derivarse:

```text
A = 10
```

cuando sea semánticamente válido.

---

# 94. Parameter propagation

Dados:

```text
A = B
B = :p
```

puede derivarse:

```text
A = :p
```

sin conocer el runtime value.

---

# 95. Null propagation

Dados:

```text
A = B
A IS NOT NULL
```

no siempre debe derivarse ingenuamente:

```text
B IS NOT NULL
```

sin considerar scope y semántica de la igualdad.

---

# 96. WHERE equality refinement

En un `WHERE` donde:

```text
A = B
```

ha evaluado TRUE:

```text
A
B
```

son ambos no NULL en las filas supervivientes bajo igualdad SQL ordinaria.

---

# 97. NULL-safe equality

Para:

```text
IS NOT DISTINCT FROM
```

la inferencia de NON_NULL no es válida.

---

# 98. Predicate knowledge

Cada operador deberá proporcionar un descriptor semántico indicando qué facts puede derivar.

---

# 99. PredicateConstraintRule

Contrato conceptual:

```php
interface PredicateConstraintRuleInterface
{
    public function supports(PredicateNode $predicate): bool;

    public function analyze(
        PredicateNode $predicate,
        ConstraintAnalysisContext $context,
        ConstraintBuilder $constraints,
    ): void;
}
```

---

# 100. Rule registry

```text
PredicateConstraintRuleRegistry
```

se congelará tras bootstrap.

---

# 101. Built-in rules

V1:

```text
ComparisonConstraintRule
NullPredicateConstraintRule
BetweenConstraintRule
InConstraintRule
ExistsConstraintRule
LogicalAndConstraintRule
LogicalOrConstraintRule
LogicalNotConstraintRule
DistinctnessConstraintRule
BooleanTestConstraintRule
```

---

# 102. AND

```text
A AND B
```

permite combinar constraints cuando el contexto requiere que ambos sean TRUE.

---

# 103. OR

```text
A OR B
```

requiere mucha más cautela.

---

# 104. OR example

```sql
WHERE status = 'active'
OR status = 'pending'
```

no permite afirmar:

```text
status = 'active'
```

ni:

```text
status = 'pending'
```

globalmente.

---

# 105. Disjunctive facts

Podrá representarse:

```text
status ∈ {'active', 'pending'}
```

si existe una abstracción segura.

---

# 106. DisjunctionConstraint

Modelo opcional:

```text
DisjunctionConstraint
├── alternatives
├── commonFacts
└── scope
```

---

# 107. Common facts across OR

Ejemplo:

```text
(A = 10 AND B = 1)
OR
(A = 10 AND B = 2)
```

puede derivar:

```text
A = 10
```

---

# 108. Complexity control

El análisis de OR puede producir explosión combinatoria.

Por ello tendrá límites estrictos.

---

# 109. NOT

La negación no se resolverá mediante reglas clásicas ingenuas debido a SQL 3VL.

---

# 110. De Morgan

Transformaciones:

```text
NOT(A AND B)
```

a:

```text
NOT A OR NOT B
```

pueden ser lógicamente válidas bajo ciertas semánticas, pero Constraint Analysis no necesita reescribir AST para utilizarlas.

---

# 111. UNKNOWN preservation

Toda regla lógica deberá preservar:

```text
TRUE
FALSE
UNKNOWN
```

---

# 112. PredicateTruthModel

Podrá existir:

```text
PredicateTruthModel
```

para clasificar implicaciones.

---

# 113. IN constraints

```sql
status IN ('active', 'pending')
```

puede derivar:

```text
status ∈ {'active', 'pending'}
```

---

# 114. IN with NULL

Si la lista contiene NULL, el modelo deberá preservar semántica SQL correctamente.

---

# 115. NOT IN

Requiere cautela especial.

No debe derivarse:

```text
x != every value
```

ignorando NULL.

---

# 116. Empty IN

La normalización previa ya deberá haber resuelto la semántica definida por la API para listas vacías.

---

# 117. LIKE constraints

En V1, `LIKE` podrá aportar información limitada.

Ejemplo:

```text
name LIKE 'abc%'
```

no deberá transformarse automáticamente en un range universal debido a collation/platform semantics.

---

# 118. Regex constraints

Normalmente:

```text
opaque semantic predicate
```

salvo reglas especializadas.

---

# 119. EXISTS constraints

`EXISTS` puede aportar facts de existencia/correlation, pero no necesariamente facts sobre valores proyectados.

---

# 120. NOT EXISTS

Puede aportar ausencia relacional dentro del scope correspondiente.

No será convertido aquí en anti join.

---

# 121. Join constraint analysis

Los JOINs aportan constraints mediante:

```text
JoinType
JoinPredicate
RelationGraph
NullabilityEffects
RelationshipEvidence
```

---

# 122. INNER JOIN

Un predicate de INNER JOIN satisfecho puede aportar facts al output combinado.

---

# 123. LEFT JOIN

Los facts derivados del `ON` deben permanecer limitados al:

```text
JOIN_MATCH_SCOPE
```

salvo que condiciones posteriores permitan fortalecerlos.

---

# 124. Outer join strengthening evidence

Ejemplo:

```sql
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
WHERE o.id IS NOT NULL
```

puede aportar evidencia de que las filas supervivientes tienen match en `orders`.

---

# 125. No rewrite here

El analyzer puede producir:

```text
OuterJoinMatchRequiredFact
```

pero no cambia:

```text
LEFT JOIN
```

por:

```text
INNER JOIN
```

Eso pertenece al Optimizer.

---

# 126. Join elimination evidence

Si una relación no aporta columnas y una FK/uniqueness demuestra determinadas propiedades, el analyzer puede producir facts útiles.

Pero:

```text
remove join
```

es una decisión del Optimizer.

---

# 127. Functional dependency propagation

Las equalities pueden propagar FDs cuidadosamente.

---

# 128. Example

Si:

```text
A UNIQUE
A = B
```

puede existir evidencia para:

```text
B unique within matching scope
```

si la igualdad y nullability permiten la conclusión.

---

# 129. Scope-sensitive uniqueness

La uniqueness de un schema column puede alterarse después de operaciones relacionales.

Ejemplo:

```text
users.id
```

es UNIQUE en `users`.

Después de:

```sql
users
JOIN orders
```

`users.id` puede repetirse en el resultado.

---

# 130. Critical distinction

Debe mantenerse:

```text
Source Relation Uniqueness
≠
Query Result Uniqueness
```

---

# 131. Key preservation

Se deberá modelar:

```text
KeyPreservationFact
```

para determinar si una operación conserva una key.

---

# 132. Example

Una relación `users` unida 1:1 puede preservar `users.id`.

Una relación 1:N puede no preservarla en el resultado.

---

# 133. Grouping constraints

Para:

```sql
GROUP BY user_id
```

el output posee una key lógica:

```text
user_id
```

por grupo.

---

# 134. Group key

Puede derivarse:

```text
GroupKeyFact(user_id)
```

---

# 135. Aggregate cardinality

Una aggregate query sin GROUP BY:

```sql
SELECT COUNT(*)
FROM users
```

puede derivar:

```text
ResultCardinality = 1
```

según semántica SQL aplicable.

---

# 136. GROUP BY cardinality

Con GROUP BY:

```text
ResultCardinality <= number of distinct group keys
```

pero no se estima aquí un número físico.

---

# 137. HAVING

Los constraints de HAVING se aplican al:

```text
GROUP OUTPUT SCOPE
```

no a filas individuales previas al grouping.

---

# 138. Projection constraints

Projection aliases pueden recibir facts derivados de su expression.

Ejemplo:

```sql
SELECT price * quantity AS subtotal
```

`subtotal` puede heredar:

```text
type
nullability
domain characteristics
```

pero no todas las constraints de ambos operands.

---

# 139. Expression constraint analysis

El sistema podrá tener:

```text
ExpressionConstraintRule
```

para operaciones conocidas.

---

# 140. Arithmetic constraints

Ejemplo:

```text
ABS(x)
```

puede aportar:

```text
result >= 0
```

si la semántica del tipo/función lo garantiza.

---

# 141. Function constraints

Las funciones podrán declarar:

```text
FunctionConstraintDescriptor
```

---

# 142. Example

`COALESCE(x, y)` puede aportar nullability facts según sus argumentos.

---

# 143. Extension functions

No se asumirán propiedades de funciones desconocidas.

---

# 144. Volatility

Functions/predicates pueden ser:

```text
IMMUTABLE
STABLE
VOLATILE
UNKNOWN
```

---

# 145. Volatility impact

Una expresión volatile limita:

```text
deduplication
constant reasoning
equivalence reasoning
evaluation-count assumptions
```

---

# 146. Example

Dos ocurrencias de:

```text
random()
```

no son necesariamente iguales.

---

# 147. Expression identity

Debe mantenerse:

```text
structurally equal expression
≠
same runtime value
```

si la expresión es volatile.

---

# 148. Determinism

Constraint derivation deberá consultar:

```text
ExpressionSemanticInfo
```

para conocer determinism/volatility.

---

# 149. Equivalence classes

Sólo expresiones apropiadas podrán participar en equivalence classes.

---

# 150. Raw expressions

```text
RawExpression
RawPredicate
```

serán barriers por defecto.

---

# 151. Raw facts

Una extensión podrá aportar facts explícitos sobre raw constructs únicamente mediante contrato especializado.

---

# 152. Security constraints

Policies pueden aportar:

```text
SecurityConstraint
```

por ejemplo:

```text
tenant_id = TenantParameter
```

---

# 153. Security provenance

Estas constraints deberán conservar:

```text
origin = SECURITY_POLICY
```

---

# 154. Security constraints no eliminables

Podrán marcarse:

```text
MANDATORY
```

para que fases posteriores no las eliminen incorrectamente.

---

# 155. Multitenancy

Una integración puede aportar:

```text
tenant_id = :tenant
```

sin introducir dependencia del core a:

```text
Tenant
TenantManager
CurrentTenant
```

---

# 156. Authorization

Una política de acceso puede aportar constraints estructuradas.

El core no dependerá de:

```text
AuthenticatedUser
Session
AuthorizationManager
```

---

# 157. Constraint strength

Cada constraint podrá tener:

```text
ConstraintStrength
```

como:

```text
INFORMATIONAL
INFERRED
REQUIRED
MANDATORY
```

---

# 158. Constraint confidence

Separado de strength:

```text
EXACT
HIGH
CONDITIONAL
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 159. Strength ≠ certainty

Una constraint puede ser:

```text
MANDATORY
```

pero depender de una capability específica.

---

# 160. Constraint implication

El sistema deberá poder responder:

```text
Does constraint set C imply fact F?
```

dentro de un scope limitado.

---

# 161. ConstraintImplicationEngine

Componente:

```text
ConstraintImplicationEngine
```

---

# 162. V1 implication scope

V1 podrá soportar:

```text
equality
constant propagation
nullability
simple ranges
uniqueness
primary keys
foreign keys
simple cardinality
simple functional dependencies
```

---

# 163. No general theorem prover

El sistema no intentará resolver lógica arbitraria.

---

# 164. Implication result

```text
PROVEN
DISPROVEN
CONDITIONAL
UNKNOWN
```

---

# 165. Proof model

Cuando sea útil podrá producir:

```text
ConstraintProof
```

---

# 166. ConstraintProof

```text
ConstraintProof
├── conclusion
├── premises
├── rule
├── scope
└── confidence
```

---

# 167. Proof DAG

En lugar de duplicar cadenas completas, podrá utilizarse:

```text
ProofGraph
```

---

# 168. Diagnostic mode

Proof information detallada puede habilitarse para:

```text
debug
testing
explain
developer toolbar
```

---

# 169. Production mode

Podrá conservar sólo provenance mínima necesaria para reducir memoria.

---

# 170. Constraint normalization

Constraints equivalentes deberán tener una representación canónica.

---

# 171. Example

```text
A = B
```

y:

```text
B = A
```

pueden compartir una forma canónica cuando el operador sea simétrico.

---

# 172. Canonicalization safety

No todos los operadores pueden invertirse/reordenarse.

Las reglas deberán provenir de descriptors semánticos.

---

# 173. Constraint fingerprinting

El artifact deberá ser fingerprintable.

---

# 174. Fingerprint inputs

Podrá incluir:

```text
normalized query fingerprint
schema semantic fingerprint
type inference fingerprint
relation graph fingerprint
constraint rule registry version
policy fingerprint
relevant capability fingerprint
extension semantic versions
```

---

# 175. Runtime values excluded

No incluir:

```text
runtime parameter values
credentials
request IDs
connection IDs
telemetry span IDs
```

---

# 176. Literal handling

Los literals estructurales que forman parte del Query AST sí pueden influir en el fingerprint semántico cuando cambian la consulta.

---

# 177. Sensitive literals

Deberán fingerprintarse mediante representación segura cuando sea necesario, sin exponer el valor.

---

# 178. ConstraintAnalysisContext

Contexto propuesto:

```text
ConstraintAnalysisContext
├── SymbolResolutionTable
├── SchemaResolutionTable
├── QueryTypeTable
├── NullabilityTable
├── RelationGraph
├── QuerySchemaView
├── CapabilitySnapshot
├── PolicySnapshot
├── QueryMetadata
├── DiagnosticSink
├── ConstraintAnalysisBudget
└── ExtensionRegistry
```

---

# 179. ConstraintAnalysisState

Estado temporal:

```text
ConstraintAnalysisState
├── ConstraintGraphBuilder
├── EqualityClassBuilder
├── RangeFactBuilder
├── NullabilityFactBuilder
├── FunctionalDependencyBuilder
├── DerivedFactBuilder
├── ContradictionBuilder
├── ProofBuilder
├── WorkQueue
├── Memo
└── Counters
```

---

# 180. Context vs state

```text
Context
=
stable operation inputs

State
=
temporary mutable analysis progress
```

---

# 181. Worklist architecture

```text
Seed Constraints
      │
      ▼
Constraint Graph
      │
      ▼
Work Queue
      │
      ▼
Apply Derivation Rules
      │
      ▼
New Facts?
  │       │
 yes      no
  │       │
  └──► requeue
          │
          ▼
      Fixed Point
```

---

# 182. Fixed point

El análisis deberá converger hacia un conjunto estable de facts.

---

# 183. Idempotence

Conceptualmente:

```text
analyze(analyze(Q))
=
analyze(Q)
```

en términos de conocimiento semántico.

---

# 184. Monotonicity

Siempre que sea posible, las reglas serán monotónicas:

```text
known facts
→
known facts + derived facts
```

sin retirar hechos arbitrariamente.

---

# 185. Non-monotonic rules

Si futuras extensiones requieren reglas no monotónicas, deberán estar explícitamente aisladas y controladas.

---

# 186. Rule categories

```text
EqualityRule
NullabilityRule
RangeRule
UniquenessRule
FunctionalDependencyRule
RelationshipRule
CardinalityRule
PredicateRule
ExpressionRule
SchemaRule
DomainRule
ExtensionRule
```

---

# 187. ConstraintRuleRegistry

Todas las reglas registradas estarán en:

```text
ConstraintRuleRegistry
```

---

# 188. Registry lifecycle

```text
bootstrap
→ register
→ validate
→ freeze
```

---

# 189. Deterministic ordering

Las reglas tendrán:

```text
ruleId
priority
dependencies
version
```

---

# 190. Rule cycle

Las reglas pueden derivar facts recursivamente, pero el engine deberá detectar:

```text
non-convergent rule behavior
```

---

# 191. Analysis budget

Se definirá:

```text
ConstraintAnalysisBudget
```

---

# 192. Budget limits

Podrá controlar:

```text
maxConstraints
maxDerivedFacts
maxEqualityClasses
maxRangeFragments
maxProofNodes
maxRuleApplications
maxIterations
maxDisjunctionBranches
maxFunctionalDependencies
maxCardinalityFacts
maxExtensionSteps
```

---

# 193. Budget exceeded

Produce:

```text
ConstraintAnalysisBudgetExceededException
```

---

# 194. Partial result policy

Por defecto:

```text
budget exceeded
→ analysis failure
```

si la fase necesita garantías completas.

---

# 195. Best-effort mode

Herramientas de diagnostics podrían permitir:

```text
PARTIAL
```

pero deberá estar explícitamente marcado.

Nunca se tratará como artifact completo.

---

# 196. Complexity attacks

Los límites protegen contra queries diseñadas para provocar:

```text
exponential OR expansion
massive equality graphs
deep nested predicates
huge IN lists
recursive extension rules
proof graph explosion
```

---

# 197. Persistent runtime safety

Compartible:

```text
Frozen Rule Registry
Frozen Type Descriptors
Frozen Extension Registry
Stateless Rule Objects
```

Operation-scoped:

```text
ConstraintAnalysisContext
ConstraintAnalysisState
ConstraintGraphBuilder
WorkQueue
ProofBuilder
```

---

# 198. No global state

Nunca:

```php
ConstraintAnalyzer::$currentFacts
```

---

# 199. FrankenPHP

```text
Worker
├── Frozen Constraint Rules
│
├── Query A
│   └── ConstraintState A
│
└── Query B
    └── ConstraintState B
```

---

# 200. OpenSwoole

```text
Coroutine A → ConstraintState A
Coroutine B → ConstraintState B
```

---

# 201. Offline analysis

Constraint Analysis deberá poder ejecutarse sin una conexión física cuando disponga de:

```text
SchemaSnapshot
CapabilitySnapshot
Semantic Query Artifact
```

---

# 202. No hidden schema I/O

Nunca deberá consultar directamente:

```text
INFORMATION_SCHEMA
pg_catalog
PRAGMA
```

durante esta fase.

Eso pertenece a Schema Introspection.

---

# 203. Capability awareness

Algunas constraints dependen de:

```text
NULL uniqueness semantics
collation
comparison semantics
constraint enforcement
deferrability
```

---

# 204. CapabilitySnapshot

El analyzer consume una snapshot estable.

No pregunta dinámicamente al Driver.

---

# 205. Platform-dependent facts

Podrán marcarse:

```text
PLATFORM_DEPENDENT
```

---

# 206. Optimizer contract

El Optimizer recibirá facts como:

```text
EqualityClass
RangeFact
NonNullFact
UniquenessFact
FunctionalDependency
CardinalityFact
Contradiction
KeyPreservationFact
RelationshipFact
```

---

# 207. Optimizer decisions

Con esos facts podrá posteriormente considerar:

```text
predicate simplification
predicate propagation
join elimination
join strengthening
constant folding
redundant predicate removal
range simplification
subquery decorrelation
```

pero no pertenecen a este sistema.

---

# 208. Planner contract

El Planner podrá utilizar:

```text
join keys
uniqueness
cardinality bounds
functional dependencies
relationship facts
```

para seleccionar planes.

---

# 209. Example: primary key lookup

Consulta:

```sql
SELECT *
FROM users
WHERE id = :id;
```

Facts:

```text
users.id PRIMARY KEY
users.id = :id
```

Derived:

```text
users.id UNIQUE
users.id NON_NULL
ResultCardinality <= 1
```

El Planner puede usar posteriormente esa información.

---

# 210. Example: contradiction

```sql
SELECT *
FROM users
WHERE id = 10
AND id = 20;
```

si los literals son distintos dentro del mismo tipo:

```text
id = 10
id = 20
────────
CONTRADICTION
```

---

# 211. Analyzer output

```text
UnsatisfiableConstraintSet
```

No modifica todavía el AST.

---

# 212. Example: range contradiction

```sql
WHERE age > 50
AND age < 20
```

produce:

```text
age ∈ (50, +∞)
∩
age ∈ (-∞, 20)

=
∅
```

---

# 213. Example: equality propagation

```sql
WHERE a = b
AND b = c
AND c = :value
```

produce:

```text
EqualityClass
├── a
├── b
├── c
└── :value
```

---

# 214. Example: outer join

```sql
SELECT *
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
WHERE o.id IS NOT NULL;
```

Facts:

```text
J1 = LEFT JOIN
orders side = NULL_EXTENDED
WHERE o.id IS NOT NULL
orders.id schema NON_NULL
```

Derived:

```text
SurvivingRowsRequireRightMatch(J1)
```

Esto es evidencia para una futura:

```text
LEFT JOIN → INNER JOIN
```

pero el cambio no ocurre aquí.

---

# 215. Example: composite unique lookup

Schema:

```text
UNIQUE(tenant_id, slug)
```

Query:

```sql
WHERE tenant_id = :tenant
AND slug = :slug
```

Derived:

```text
ResultCardinality <= 1
```

---

# 216. Example: incomplete composite key

```sql
WHERE tenant_id = :tenant
```

No se deriva:

```text
ResultCardinality <= 1
```

---

# 217. Example: functional dependency

Schema:

```text
users.id PRIMARY KEY
```

Then:

```text
users.id
→
users.email
users.name
users.created_at
```

within source relation scope.

---

# 218. Example: join key preservation

Schema:

```text
users.id PK
profiles.user_id UNIQUE FK users.id
```

Query:

```sql
users
LEFT JOIN profiles
    ON profiles.user_id = users.id
```

puede derivar:

```text
profiles matches at most one row per user
```

y potencialmente:

```text
users.id key preserved
```

según join semantics.

---

# 219. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Semantic\Constraint\
```

---

# 220. Estructura propuesta

```text
Query/
└── Semantic/
    └── Constraint/
        ├── Contract/
        │   ├── ConstraintRuleInterface.php
        │   ├── PredicateConstraintRuleInterface.php
        │   ├── ExpressionConstraintRuleInterface.php
        │   └── ConstraintImplicationEngineInterface.php
        │
        ├── Core/
        │   ├── QueryConstraintAnalysisEngine.php
        │   ├── ConstraintCollector.php
        │   ├── ConstraintDerivationEngine.php
        │   └── ConstraintImplicationEngine.php
        │
        ├── Model/
        │   ├── ConstraintId.php
        │   ├── FactId.php
        │   ├── ConstraintSubjectId.php
        │   ├── ConstraintScope.php
        │   ├── ConstraintStrength.php
        │   ├── ConstraintCertainty.php
        │   └── ConstraintProvenance.php
        │
        ├── Graph/
        │   ├── ConstraintGraph.php
        │   ├── ConstraintGraphBuilder.php
        │   └── ConstraintGraphFingerprint.php
        │
        ├── Equality/
        │   ├── EqualityConstraint.php
        │   ├── EqualityClass.php
        │   ├── EqualityClassId.php
        │   └── EqualityClassBuilder.php
        │
        ├── Nullability/
        │   ├── NullConstraint.php
        │   ├── NonNullConstraint.php
        │   ├── NullabilityFact.php
        │   └── NullabilityFactBuilder.php
        │
        ├── Range/
        │   ├── RangeConstraint.php
        │   ├── RangeFact.php
        │   ├── RangeBound.php
        │   └── RangeAnalyzer.php
        │
        ├── Uniqueness/
        │   ├── UniquenessConstraint.php
        │   ├── UniquenessFact.php
        │   ├── KeyPreservationFact.php
        │   └── UniqueKeyAnalyzer.php
        │
        ├── Dependency/
        │   ├── FunctionalDependency.php
        │   ├── FunctionalDependencyBuilder.php
        │   ├── ForeignKeyFact.php
        │   └── RelationshipFact.php
        │
        ├── Cardinality/
        │   ├── CardinalityConstraint.php
        │   ├── CardinalityFact.php
        │   └── CardinalityBound.php
        │
        ├── Constant/
        │   ├── ConstantConstraint.php
        │   └── ConstantFact.php
        │
        ├── Predicate/
        │   ├── ComparisonConstraintRule.php
        │   ├── NullPredicateConstraintRule.php
        │   ├── BetweenConstraintRule.php
        │   ├── InConstraintRule.php
        │   ├── LogicalAndConstraintRule.php
        │   ├── LogicalOrConstraintRule.php
        │   ├── LogicalNotConstraintRule.php
        │   └── DistinctnessConstraintRule.php
        │
        ├── Proof/
        │   ├── ConstraintProof.php
        │   ├── ProofGraph.php
        │   └── ProofRule.php
        │
        ├── Contradiction/
        │   ├── ConstraintContradiction.php
        │   ├── UnsatisfiableConstraintSet.php
        │   └── ContradictionDetector.php
        │
        ├── Context/
        │   ├── ConstraintAnalysisContext.php
        │   ├── ConstraintAnalysisState.php
        │   └── ConstraintAnalysisBudget.php
        │
        ├── Rule/
        │   ├── ConstraintRuleRegistry.php
        │   ├── ConstraintRuleDescriptor.php
        │   └── ConstraintRuleId.php
        │
        ├── Extension/
        │   ├── ConstraintExtensionRegistry.php
        │   └── ConstraintExtensionDescriptor.php
        │
        ├── Result/
        │   ├── QueryConstraintAnalysisResult.php
        │   ├── DerivedFactTable.php
        │   └── ConstraintAnalysisFingerprint.php
        │
        ├── Diagnostic/
        │   ├── ConstraintDiagnostic.php
        │   └── ConstraintDiagnosticCode.php
        │
        └── Exception/
            ├── QueryConstraintAnalysisException.php
            ├── ConstraintConflictException.php
            ├── ConstraintContradictionException.php
            ├── ConstraintRuleConflictException.php
            ├── ConstraintAnalysisBudgetExceededException.php
            └── InvalidConstraintExtensionException.php
```

---

# 221. Invariantes arquitectónicos

## DB-CONSTRAINT-001

Constraint Analysis no generará SQL.

## DB-CONSTRAINT-002

No ejecutará queries.

## DB-CONSTRAINT-003

No abrirá conexiones.

## DB-CONSTRAINT-004

No dependerá de PDO.

## DB-CONSTRAINT-005

No dependerá del Driver.

## DB-CONSTRAINT-006

No dependerá del Executor.

## DB-CONSTRAINT-007

No seleccionará índices.

## DB-CONSTRAINT-008

No seleccionará join algorithms.

## DB-CONSTRAINT-009

No realizará query rewrites.

## DB-CONSTRAINT-010

No realizará predicate pushdown.

## DB-CONSTRAINT-011

No eliminará joins.

## DB-CONSTRAINT-012

No fortalecerá joins directamente.

## DB-CONSTRAINT-013

No reemplazará contradicciones por FALSE en el AST.

## DB-CONSTRAINT-014

Constraint será diferente de Predicate.

## DB-CONSTRAINT-015

Derived Fact será diferente de Constraint.

## DB-CONSTRAINT-016

Optimization Rule será diferente de Constraint Rule.

## DB-CONSTRAINT-017

Toda constraint tendrá scope.

## DB-CONSTRAINT-018

Toda constraint tendrá provenance.

## DB-CONSTRAINT-019

Toda derived fact tendrá provenance.

## DB-CONSTRAINT-020

SQL three-valued logic será preservada.

## DB-CONSTRAINT-021

UNKNOWN no será FALSE.

## DB-CONSTRAINT-022

WHERE facts serán scope-sensitive.

## DB-CONSTRAINT-023

JOIN ON facts serán scope-sensitive.

## DB-CONSTRAINT-024

LEFT JOIN ON facts no serán promovidos globalmente sin prueba.

## DB-CONSTRAINT-025

RIGHT JOIN ON facts no serán promovidos globalmente sin prueba.

## DB-CONSTRAINT-026

FULL JOIN requerirá análisis conservador.

## DB-CONSTRAINT-027

Equality propagation respetará NULL.

## DB-CONSTRAINT-028

Distinctness no será igualdad ordinaria.

## DB-CONSTRAINT-029

NOT IN respetará NULL semantics.

## DB-CONSTRAINT-030

OR no será tratado como AND.

## DB-CONSTRAINT-031

NOT respetará three-valued logic.

## DB-CONSTRAINT-032

Constraint Analysis no será un theorem prover general.

## DB-CONSTRAINT-033

Las inferencias estarán limitadas a reglas registradas.

## DB-CONSTRAINT-034

Rule Registry será frozen tras bootstrap.

## DB-CONSTRAINT-035

Rule ordering será determinista.

## DB-CONSTRAINT-036

Rule conflicts serán errores.

## DB-CONSTRAINT-037

Analysis deberá converger.

## DB-CONSTRAINT-038

Analysis tendrá budget.

## DB-CONSTRAINT-039

Budget exhaustion no será éxito silencioso.

## DB-CONSTRAINT-040

Constraint graph será operation-scoped.

## DB-CONSTRAINT-041

Analysis state será operation-scoped.

## DB-CONSTRAINT-042

No existirá global current constraint state.

## DB-CONSTRAINT-043

Persistent workers no compartirán mutable analysis state.

## DB-CONSTRAINT-044

Concurrent queries tendrán analysis states independientes.

## DB-CONSTRAINT-045

Schema nullability será diferente de query nullability.

## DB-CONSTRAINT-046

Source uniqueness será diferente de result uniqueness.

## DB-CONSTRAINT-047

Primary Key podrá implicar uniqueness.

## DB-CONSTRAINT-048

Primary Key podrá implicar non-null cuando schema semantics lo garanticen.

## DB-CONSTRAINT-049

Composite uniqueness no implicará individual uniqueness.

## DB-CONSTRAINT-050

Foreign Key será diferente de Functional Dependency.

## DB-CONSTRAINT-051

Foreign Key será diferente de JOIN.

## DB-CONSTRAINT-052

ORM Relationship será evidencia opcional.

## DB-CONSTRAINT-053

Query Engine funcionará sin ORM.

## DB-CONSTRAINT-054

Cardinality fact será cualitativo/lógico.

## DB-CONSTRAINT-055

Physical row estimation pertenecerá al Optimizer/Planner.

## DB-CONSTRAINT-056

Domain types conservarán identidad semántica.

## DB-CONSTRAINT-057

Storage compatibility no implicará domain compatibility.

## DB-CONSTRAINT-058

No se utilizarán PHP comparisons como semántica universal.

## DB-CONSTRAINT-059

Range analysis será type-aware.

## DB-CONSTRAINT-060

String range analysis será collation-aware/conservative.

## DB-CONSTRAINT-061

Temporal range analysis preservará temporal semantics.

## DB-CONSTRAINT-062

Volatile expressions limitarán equivalence reasoning.

## DB-CONSTRAINT-063

Raw expressions serán barriers por defecto.

## DB-CONSTRAINT-064

Unknown knowledge no significará false/absent.

## DB-CONSTRAINT-065

Partial schema no se tratará como complete schema.

## DB-CONSTRAINT-066

Schema knowledge mode será explícito.

## DB-CONSTRAINT-067

Deferred constraints serán tratadas explícitamente.

## DB-CONSTRAINT-068

Platform-dependent facts serán marcados.

## DB-CONSTRAINT-069

CapabilitySnapshot será estable durante análisis.

## DB-CONSTRAINT-070

No habrá hidden schema I/O.

## DB-CONSTRAINT-071

Analysis podrá ejecutarse offline.

## DB-CONSTRAINT-072

Type Inference seguirá siendo dueño de Query Types.

## DB-CONSTRAINT-073

Relation Resolution seguirá siendo dueño de RelationGraph.

## DB-CONSTRAINT-074

Constraint Analysis consumirá esos artifacts sin mutarlos.

## DB-CONSTRAINT-075

AST permanecerá immutable.

## DB-CONSTRAINT-076

Semantic facts se almacenarán en side tables/artifacts.

## DB-CONSTRAINT-077

Equality classes no mutarán expressions.

## DB-CONSTRAINT-078

Contradictions serán explícitas.

## DB-CONSTRAINT-079

Proofs serán opcionales.

## DB-CONSTRAINT-080

Production podrá usar compact provenance.

## DB-CONSTRAINT-081

Diagnostics no expondrán secretos.

## DB-CONSTRAINT-082

Sensitive constants tendrán redaction.

## DB-CONSTRAINT-083

Runtime parameter values no serán requeridos.

## DB-CONSTRAINT-084

Constraint fingerprint excluirá runtime parameter values.

## DB-CONSTRAINT-085

Relevant schema changes invalidarán analysis cache.

## DB-CONSTRAINT-086

Relevant rule changes invalidarán analysis cache.

## DB-CONSTRAINT-087

Relevant extension changes invalidarán analysis cache.

## DB-CONSTRAINT-088

Security constraints conservarán provenance.

## DB-CONSTRAINT-089

Mandatory security constraints no podrán debilitarse silenciosamente.

## DB-CONSTRAINT-090

Multitenancy será integración opcional.

## DB-CONSTRAINT-091

Core no dependerá de Tenant.

## DB-CONSTRAINT-092

Core no dependerá de AuthenticatedUser.

## DB-CONSTRAINT-093

Functional dependencies serán scope-aware.

## DB-CONSTRAINT-094

Key preservation será explícita.

## DB-CONSTRAINT-095

Grouping introducirá scope propio.

## DB-CONSTRAINT-096

HAVING constraints pertenecerán al group-output scope.

## DB-CONSTRAINT-097

Constraint implication devolverá UNKNOWN cuando no pueda probar.

## DB-CONSTRAINT-098

Falta de prueba no será prueba de falsedad.

## DB-CONSTRAINT-099

El resultado será determinista.

## DB-CONSTRAINT-100

Constraint Analysis sólo producirá hechos justificables mediante evidencia y reglas semánticas explícitas.

---

# 222. Anti-patrones

## Anti-pattern 1 — Optimizar dentro del analyzer

```text
contradiction
→ rewrite WHERE FALSE
```

Incorrecto.

---

## Anti-pattern 2 — Ignorar scopes

```text
LEFT JOIN ON x = y
→ globally x = y
```

Incorrecto.

---

## Anti-pattern 3 — Classical boolean logic

Ignorar `UNKNOWN`.

Incorrecto.

---

## Anti-pattern 4 — Composite key simplification

```text
UNIQUE(a,b)
→ UNIQUE(a)
```

Incorrecto.

---

## Anti-pattern 5 — FK means inner join

Incorrecto.

---

## Anti-pattern 6 — Schema uniqueness survives every join

Incorrecto.

---

## Anti-pattern 7 — Unknown means absent

Incorrecto.

---

## Anti-pattern 8 — Evaluate runtime parameters

Constraint Analysis no deberá depender de valores de ejecución.

---

## Anti-pattern 9 — General theorem prover

Intentar resolver lógica arbitraria provocaría complejidad y comportamiento impredecible.

---

## Anti-pattern 10 — Shared mutable fact table

Inseguro bajo runtimes persistentes.

---

# 223. Ejemplo integral

Schema:

```text
users
├── id       UserId PK
├── email    Email UNIQUE NOT NULL
└── active   Boolean NOT NULL

orders
├── id       OrderId PK
├── user_id  UserId NOT NULL FK → users.id
├── status   OrderStatus NOT NULL
└── total    Money NOT NULL
```

Consulta:

```sql
SELECT
    u.id,
    u.email,
    o.total
FROM users u
JOIN orders o
    ON o.user_id = u.id
WHERE
    o.user_id = :userId
    AND o.total >= :minimum
    AND o.status = 'paid';
```

Constraint seeds:

```text
C1 o.user_id = u.id
C2 o.user_id = :userId
C3 o.total >= :minimum
C4 o.status = 'paid'

C5 users.id PRIMARY KEY
C6 orders.id PRIMARY KEY
C7 orders.user_id FK → users.id
C8 users.email UNIQUE
```

Type facts:

```text
u.id       → UserId
o.user_id  → UserId
:userId    → UserId

o.total    → Money
:minimum   → Money

o.status   → OrderStatus
```

Equality class:

```text
E1
├── u.id
├── o.user_id
└── :userId
```

Derived facts:

```text
F1 u.id = :userId
F2 u.id NON_NULL
F3 o.user_id NON_NULL
F4 :userId NON_NULL within successful equality scope
F5 users.id UNIQUE in source relation
F6 o.status = 'paid'
F7 o.total >= :minimum
```

Relationship:

```text
orders.user_id
→ FK users.id
```

Join evidence:

```text
Join J1
├── EQUI_JOIN
├── FK_BACKED
└── MANY_TO_ONE orders → users
```

El analyzer termina aquí.

No realiza:

```text
predicate propagation rewrite
join elimination
index selection
join ordering
SQL generation
```

---

# 224. Arquitectura final

```text
                     Semantic Inputs
                           │
       ┌───────────────────┼────────────────────┐
       ▼                   ▼                    ▼
    Predicates          Schema               Types
       │                   │                    │
       └─────────────┐     │     ┌──────────────┘
                     ▼     ▼     ▼
                    Constraint
                     Collector
                        │
                        ▼
                 Seed Constraints
                        │
                        ▼
                 Constraint Graph
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
      Equality      Nullability      Ranges
          │             │             │
          ├─────────────┼─────────────┤
          ▼             ▼             ▼
     Uniqueness     Dependencies   Cardinality
          │             │             │
          └─────────────┼─────────────┘
                        ▼
               Derivation Engine
                        │
                        ▼
                  Fixed Point
                        │
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
    Derived Facts   Contradictions    Proofs
          │             │              │
          └─────────────┼──────────────┘
                        ▼
          QueryConstraintAnalysisResult
                        │
                        ▼
              Semantic Query Graph
```

---

# 225. Fórmulas maestras

## Constraint Analysis

```text
Constraint Knowledge
=
Query Predicates
+
Schema Constraints
+
Resolved Types
+
Relation Semantics
+
Relationship Evidence
+
Domain Rules
+
Policy Constraints
+
Derived Facts
```

---

## Derived Fact

```text
Derived Fact
=
Premises
+
Semantic Rule
+
Scope
+
Type Semantics
+
NULL Semantics
+
Provenance
```

---

## Equality class

```text
EqualityClass
=
Compatible Equality Subjects
+
Scope
+
NULL Semantics
+
Certainty
```

---

## Query cardinality bound

```text
Logical Cardinality Bound
=
Unique Key Evidence
+
Equality Constraints
+
Filter Coverage
+
Join Cardinality
+
Grouping Semantics
```

---

## Safe constraint analysis

```text
Safe Constraint Analysis
=
Explicit Constraints
+
Scoped Reasoning
+
Three-Valued Logic
+
Type Awareness
+
Schema Awareness
+
Relationship Awareness
+
Bounded Derivation
+
Deterministic Rules
+
Provenance
+
Conservative Unknown Handling
```

---

# 226. Frontera con el Optimizer

Esta frontera es crítica.

Constraint Analysis puede descubrir:

```text
A = B
B = 10
────────
A = 10
```

Optimizer puede decidir:

```text
propagate A = 10
```

Constraint Analysis puede descubrir:

```text
x > 50
x < 20
────────
UNSATISFIABLE
```

Optimizer puede decidir:

```text
replace filter with FALSE
```

Constraint Analysis puede descubrir:

```text
LEFT JOIN
+
right.id IS NOT NULL
────────
right match required
```

Optimizer puede considerar:

```text
LEFT JOIN → INNER JOIN
```

Por tanto:

```text
Constraint Analysis
=
Knowledge Discovery
```

mientras:

```text
Optimizer
=
Semantics-Preserving Transformation
```

---

# 227. Estado completo del Semantic Query Engine

Con los documentos `35`–`41`, el pipeline semántico queda:

```text
                     Query AST
                        │
                        ▼
              36 Semantic Analysis
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           │
   37 Symbol Resolution               │
          │                           │
          ▼                           │
38 Schema-Aware Resolution            │
          │                           │
          ▼                           │
   39 Type Inference                  │
          │                           │
          ▼                           │
40 Relation / Join Resolution         │
          │                           │
          ▼                           │
41 Constraint Analysis                │
          │                           │
          └─────────────┬─────────────┘
                        ▼
             Semantic Knowledge
                        │
                        ▼
            42 Semantic Query Graph
```

El sistema ya conoce:

```text
Symbols
+
Schema Objects
+
Types
+
Nullability
+
Relations
+
Joins
+
Correlation
+
Relationship Evidence
+
Equalities
+
Ranges
+
Constants
+
Keys
+
Uniqueness
+
Functional Dependencies
+
Cardinality Bounds
+
Contradictions
+
Capability Requirements
```

sin haber generado todavía SQL.

---

# 228. Próximo documento

```text
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

El siguiente documento cerrará el bloque **Semantic Query Engine** consolidando los artifacts producidos por:

```text
Symbol Resolution
Schema-Aware Resolution
Type Inference
Relation Resolution
Constraint Analysis
```

en una representación unificada:

```text
SemanticQueryGraph
│
├── Symbol Graph
├── Relation Graph
├── Expression Semantics
├── Predicate Semantics
├── Type Information
├── Nullability
├── Constraint Graph
├── Dependency Graph
├── Correlation Graph
├── Lineage
├── Capability Requirements
└── Provenance
```

que se convertirá en la **representación semántica autoritativa** que recibirán posteriormente:

```text
Query Optimizer
        │
        ▼
Query Planner
        │
        ▼
SQL Compiler
```

manteniendo la regla:

```text
Query AST
=
What the query structurally says

Semantic Query Graph
=
What the query semantically means

Optimized Query
=
Equivalent but improved semantic form

Query Plan
=
How the query should be executed

Compiled SQL
=
How the selected plan is expressed for a target database
```