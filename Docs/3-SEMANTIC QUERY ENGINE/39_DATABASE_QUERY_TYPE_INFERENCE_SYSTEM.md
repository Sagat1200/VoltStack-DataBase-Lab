# 39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md

# VoltStack Quantum Database
## Query Type Inference System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 39 — Query Type Inference System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Semantic Layer  
**Versión:** 1.0

---

# 1. Propósito

El **Query Type Inference System** determina los tipos semánticos de:

- expresiones;
- parámetros;
- columnas;
- funciones;
- operadores;
- predicates;
- projections;
- aggregates;
- CASE expressions;
- subqueries;
- CTE outputs;
- set operations;
- tuple expressions;
- collection parameters;
- JSON expressions;
- domain types.

El sistema parte de evidencia parcial distribuida entre:

```text
Query AST
Schema Metadata
Symbol Resolution
Parameter Definitions
Function Signatures
Operator Signatures
ORM Metadata
Domain Type Metadata
Explicit Casts
Literal Types
```

y produce:

```text
Resolved Query Types
```

sin depender de:

```text
PDO
Connection
Driver
SQL execution
Database server type inference
EntityManager
UnitOfWork
```

---

# 2. Objetivo principal

VoltStack debe poder transformar:

```sql
SELECT u.email
FROM users u
WHERE u.id = :id
```

desde:

```text
u.id      → unresolved type
:id       → unknown type
u.email   → unresolved type
```

hacia:

```text
u.id      → app.user_id
:id       → app.user_id
u.email   → app.email_address
```

utilizando evidencia semántica.

---

# 3. Regla maestra

> Los tipos de una consulta se resolverán mediante constraints semánticos y propagación de evidencia, nunca mediante reglas improvisadas de "primer tipo encontrado".

---

# 4. Separación fundamental

Debe mantenerse:

```text
PHP Runtime Type
≠
Domain Type
≠
Query Semantic Type
≠
Schema Physical Type
≠
Platform Database Type
≠
Database Representation
≠
Driver Binding Type
```

---

# 5. Pipeline general

```text
Normalized + Validated Query AST
              │
              ▼
       Symbol Resolution
              │
              ▼
   Schema-Aware Resolution
              │
              ▼
      Evidence Collection
              │
              ▼
   Type Constraint Generation
              │
              ▼
     Type Constraint Graph
              │
              ▼
        Constraint Solver
              │
              ▼
       Type Propagation
              │
              ▼
    Compatibility Validation
              │
              ▼
      QueryTypeTable
```

---

# 6. Relación con documento 30

El documento:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

define:

```text
qué es un Query Type
```

Este documento define:

```text
cómo se descubre ese Query Type dentro de una consulta
```

Por tanto:

```text
Query Type System
        │
        ├── type identities
        ├── descriptors
        ├── compatibility
        ├── coercion
        ├── registry
        └── platform mappings

Query Type Inference System
        │
        ├── evidence
        ├── constraints
        ├── propagation
        ├── solving
        └── resolved type table
```

---

# 7. Principio de inferencia

La inferencia debe ser:

```text
Evidence-driven
Constraint-based
Bidirectional
Deterministic
Conservative
Explainable
Extensible
Platform-neutral
Persistent-runtime-safe
```

---

# 8. No "first type wins"

Nunca:

```php
$type = $firstKnownType;
```

para resolver conflictos.

Ejemplo:

```text
Parameter :value

Occurrence A → INTEGER
Occurrence B → UUID
```

no produce:

```text
INTEGER
```

porque apareció primero.

Debe producir:

```text
ConflictingTypeConstraints
```

---

# 9. Evidencia de tipos

Las principales fuentes serán:

```text
Explicit Type
Schema Metadata
ORM Mapping Metadata
Domain Metadata
Expression Structure
Operator Signature
Function Signature
Predicate Context
Parameter Definition
Literal Value
Cast Expression
Set Operation
CASE branches
Aggregate semantics
Window function semantics
Subquery output
Extension metadata
```

---

# 10. Prioridad de evidencia

La precedencia conceptual será:

```text
Explicit Semantic Type
        │
        ▼
Schema / ORM Domain Metadata
        │
        ▼
Expression / Operator / Function Constraints
        │
        ▼
Parameter Context
        │
        ▼
Literal Inference
        │
        ▼
Safe Runtime Value Inference
        │
        ▼
Unknown
```

Esto no significa que una evidencia superior pueda ignorar conflictos.

---

# 11. ExplicitType

Una declaración explícita:

```php
->bind('id', $id, UserIdType::class)
```

aporta:

```text
ExplicitTypeConstraint(:id, app.user_id)
```

---

# 12. Explicit type no elimina validación

Si posteriormente:

```text
:id
```

se compara con:

```text
orders.uuid
```

de tipo:

```text
core.uuid
```

el sistema debe verificar compatibilidad.

---

# 13. Schema evidence

Del documento 38:

```text
SchemaColumnDescriptor
```

puede aportar:

```text
type
nullability
domain mapping
```

---

# 14. Ejemplo

```text
users.id
→ Schema BIGINT
→ ORM UserId
→ Query Type app.user_id
```

---

# 15. ORM evidence

El ORM puede enriquecer:

```text
BIGINT
```

con:

```text
app.user_id
```

pero no sustituye arbitrariamente la realidad física del schema.

---

# 16. Domain type evidence

Los domain types permiten distinguir:

```text
UserId
OrderId
InvoiceId
```

aunque todos estén almacenados como:

```text
BIGINT
```

---

# 17. Regla

```text
app.user_id
≠
app.order_id
```

aunque ambos puedan mapearse físicamente a:

```text
BIGINT
```

---

# 18. Literal inference

Los literals podrán aportar tipos seguros.

Ejemplos:

```text
10       → core.integer
10.25    → core.decimal
true     → core.boolean
"hello"  → core.string
NULL     → core.null
```

---

# 19. Literal string limitation

Una cadena:

```text
"550e8400-e29b-41d4-a716-446655440000"
```

no deberá inferirse automáticamente como:

```text
core.uuid
```

Sólo:

```text
core.string
```

salvo contexto adicional.

---

# 20. Runtime value inference

La inferencia desde valores runtime será el fallback más débil.

Ejemplo:

```php
42
```

puede aportar:

```text
integer-compatible
```

pero no:

```text
app.user_id
```

sin contexto.

---

# 21. QueryTypeVariable

Cada posición todavía no resuelta podrá representarse mediante:

```text
QueryTypeVariable
```

Ejemplo:

```text
T1 = type(users.id)
T2 = type(:id)
T3 = type(users.email)
```

---

# 22. TypeVariableId

Cada variable tendrá:

```text
TypeVariableId
```

estable dentro de la operación semántica.

---

# 23. Type variables no son AST nodes

Debe mantenerse:

```text
AstNodeId
≠
TypeVariableId
```

---

# 24. Mapping

Podrá existir:

```text
AstNodeId N17
→ TypeVariable T4
```

en una tabla de análisis.

---

# 25. TypeConstraint

La unidad fundamental será:

```text
TypeConstraint
```

---

# 26. Ejemplos

```text
T1 = app.user_id
T2 compatible-with T1
T3 must-be numeric
T4 common-type(T5, T6)
T7 nullable(T8)
T9 result-of function(length, T10)
```

---

# 27. Constraint categories

Inicialmente:

```text
EXACT_TYPE
COMPATIBILITY
EQUALITY
FAMILY
CATEGORY
PROMOTION
COERCION
NULLABILITY
FUNCTION_ARGUMENT
FUNCTION_RETURN
OPERATOR_OPERAND
OPERATOR_RESULT
COMMON_TYPE
COLLECTION_ELEMENT
TUPLE_ELEMENT
CAST_TARGET
DOMAIN_COMPATIBILITY
PLATFORM_CAPABILITY
EXTENSION
```

---

# 28. ExactTypeConstraint

Ejemplo:

```text
T1 = app.user_id
```

---

# 29. CompatibilityConstraint

Ejemplo:

```text
T_parameter compatible-with T_column
```

No necesariamente significa:

```text
T_parameter == T_column
```

---

# 30. EqualityConstraint

Cuando dos posiciones deben poseer exactamente el mismo tipo semántico:

```text
T1 == T2
```

---

# 31. FamilyConstraint

Ejemplo:

```text
T1 ∈ NumericFamily
```

---

# 32. CategoryConstraint

Ejemplo:

```text
T1 ∈ Comparable
```

o:

```text
T1 ∈ Orderable
```

---

# 33. CommonTypeConstraint

Necesario para:

```text
CASE
COALESCE
UNION
VALUES
tuple combinations
```

---

# 34. TypeConstraintGraph

Las constraints se almacenarán en:

```text
TypeConstraintGraph
```

---

# 35. Ejemplo

Consulta:

```sql
WHERE users.id = :id
```

genera:

```text
Schema:
users.id → app.user_id

Variables:
T1 = users.id
T2 = :id

Constraints:

T1 EXACT app.user_id
T1 COMPATIBLE T2
```

---

# 36. Propagación

El solver podrá concluir:

```text
T1 = app.user_id
T2 = app.user_id
```

cuando las reglas de compatibilidad lo permitan.

---

# 37. Bidirectional inference

La inferencia deberá funcionar en ambas direcciones.

Ejemplo:

```text
column → parameter
```

pero también:

```text
explicit parameter type → unknown expression
```

cuando semánticamente corresponda.

---

# 38. Ejemplo bidireccional

```sql
CAST(:value AS DECIMAL(10,2)) + price
```

puede propagar información entre:

```text
:value
cast
arithmetic expression
price
```

---

# 39. Constraint solver

El componente central será:

```text
QueryTypeConstraintSolver
```

---

# 40. Responsabilidades del solver

Debe:

- consumir constraints;
- propagar evidencia;
- detectar conflictos;
- resolver tipos comunes;
- aplicar promotion rules;
- aplicar coercion rules permitidas;
- mantener nullability;
- producir provenance;
- detectar unresolved variables;
- generar diagnostics.

---

# 41. No responsabilidades

No debe:

- ejecutar SQL;
- consultar DB;
- abrir Connection;
- generar SQL;
- seleccionar índices;
- elegir join algorithms;
- convertir valores para Driver;
- hidratar entities.

---

# 42. Constraint generation

Antes del solver existirá:

```text
QueryTypeConstraintCollector
```

---

# 43. Collector

Recorrerá:

```text
Expressions
Predicates
Parameters
Functions
Operators
Subqueries
Projections
Aggregates
Set operations
```

generando constraints.

---

# 44. Expression typing

Cada Expression kind tendrá una regla especializada.

Ejemplo:

```text
LiteralExpression
ColumnExpression
ParameterExpression
ArithmeticExpression
FunctionExpression
CaseExpression
CastExpression
SubqueryExpression
TupleExpression
```

---

# 45. ExpressionTypeRule

Contrato conceptual:

```php
interface ExpressionTypeRuleInterface
{
    public function supports(ExpressionNode $expression): bool;

    public function collectConstraints(
        ExpressionNode $expression,
        TypeInferenceContext $context,
        TypeConstraintBuilder $constraints,
    ): void;
}
```

---

# 46. Frozen rule registry

Las reglas se registrarán durante bootstrap y posteriormente:

```text
ExpressionTypeRuleRegistry
```

será immutable.

---

# 47. ColumnExpression typing

Para:

```text
u.email
```

se consulta:

```text
ColumnSymbol
        │
        ▼
SchemaResolutionTable
        │
        ▼
SchemaColumnDescriptor
        │
        ▼
Query Type Evidence
```

---

# 48. ParameterExpression typing

Para:

```text
:id
```

se combina:

```text
ParameterDefinition
+
Explicit Binding Type
+
Occurrence Constraints
+
Expression Context
```

---

# 49. Parameter value separation

El AST no contiene:

```text
runtime parameter value
```

La inferencia semántica puede realizarse sin él.

---

# 50. Runtime specialization

Si alguna estrategia futura utiliza runtime shape/type:

```text
QueryTemplate
+
BindingShape
```

deberá producir una especialización separada.

Nunca mutará el AST original.

---

# 51. Arithmetic typing

Para:

```text
price * quantity
```

se generan:

```text
price    ∈ Numeric
quantity ∈ Numeric

result = NumericPromotion(price, quantity)
```

---

# 52. Numeric promotion

Ejemplo:

```text
INTEGER + INTEGER
→ INTEGER or wider semantic integer

INTEGER + DECIMAL
→ DECIMAL

DECIMAL + FLOAT
→ policy-dependent / potentially lossy
```

---

# 53. Precision preservation

No debe ocurrir:

```text
DECIMAL + DECIMAL
→ FLOAT
```

por conveniencia.

---

# 54. Decimal inference

Podrá considerar:

```text
precision
scale
operation
```

cuando esa metadata esté disponible.

---

# 55. Division

La división requiere reglas explícitas.

No asumir:

```text
INTEGER / INTEGER → INTEGER
```

universalmente.

---

# 56. Platform semantics

Cuando el resultado dependa del motor:

```text
PLATFORM_DEPENDENT
```

deberá quedar explícito.

---

# 57. Operator typing

Cada operador semántico tendrá:

```text
OperatorSignature
```

---

# 58. Example

```text
ADD
(Numeric, Numeric) → Numeric
```

---

# 59. Comparison operators

Para:

```text
=
<
>
<=
>=
```

los operands deberán satisfacer reglas de compatibilidad.

---

# 60. Predicate result

Una comparación produce conceptualmente:

```text
core.boolean
```

con:

```text
TruthDomain = TRUE | FALSE | UNKNOWN
```

según nullability.

---

# 61. Predicate ≠ expression

Aunque exista un tipo lógico asociado:

```text
PredicateNode
≠
ExpressionNode
```

La información de verdad pertenece al sistema semántico del predicate.

---

# 62. Equality

```sql
users.id = :id
```

genera:

```text
Comparable(users.id, :id)
```

y constraints compatibles.

---

# 63. Domain equality

```text
UserId = UserId
```

válido.

---

# 64. Domain mismatch

```text
UserId = OrderId
```

deberá fallar por defecto.

---

# 65. Explicit compatibility policy

Una aplicación/extensión podría definir conversiones explícitas, pero no serán implícitas por defecto.

---

# 66. NULL comparison

La normalización ya deberá haber convertido:

```text
column = NULL
```

cuando provenga de API semántica apropiada, a:

```text
NullPredicate
```

No se utilizará Type Inference para cambiar la estructura tardíamente.

---

# 67. BETWEEN

Para:

```sql
age BETWEEN :min AND :max
```

se generan:

```text
age compatible :min
age compatible :max
age orderable
```

---

# 68. IN

```sql
id IN (:ids)
```

genera:

```text
type(id) compatible element-type(:ids)
```

---

# 69. Collection type

`:ids` podrá tener:

```text
Collection<app.user_id>
```

---

# 70. Collection ≠ native DB array

Debe mantenerse:

```text
Query Collection Type
≠
PostgreSQL ARRAY
```

El planner decide posteriormente la estrategia física.

---

# 71. Tuple IN

```sql
(a, b) IN ((:a1, :b1), (:a2, :b2))
```

requiere:

```text
Tuple<Ta, Tb>
```

y validación de aridad.

---

# 72. Function typing

Las funciones utilizan:

```text
FunctionDescriptor
+
FunctionSignature
```

---

# 73. Example LENGTH

```text
length(string) → integer
```

---

# 74. Example LOWER

```text
lower(string) → string
```

preservando metadata relevante cuando sea posible.

---

# 75. Example ABS

```text
abs(T Numeric) → T
```

o promotion definida por signature.

---

# 76. Function overloads

Una función puede tener múltiples signatures.

Ejemplo conceptual:

```text
round(decimal) → decimal
round(decimal, integer) → decimal
round(float) → float
```

---

# 77. Overload resolution

El sistema deberá:

1. obtener candidates;
2. comparar operand types;
3. calcular compatibility;
4. aplicar promotion/coercion rules;
5. seleccionar candidate inequívoco;
6. reportar ambiguity si existe.

---

# 78. No arbitrary overload selection

Nunca:

```text
first registered signature wins
```

---

# 79. FunctionSignatureResolver

Componente:

```text
FunctionSignatureResolver
```

---

# 80. Generic signatures

Podrán existir:

```text
identity<T>(T) → T
```

o:

```text
coalesce<T>(T...) → T
```

---

# 81. Aggregate typing

Ejemplos:

```text
COUNT(*) → core.big_integer
SUM(Integer) → implementation-defined wider numeric
SUM(Decimal) → Decimal
AVG(Integer) → Decimal/Platform-dependent semantic numeric
MIN(T orderable) → T
MAX(T orderable) → T
```

---

# 82. Aggregate result policy

No deberá codificarse basándose sólo en SQL vendor strings.

Usará:

```text
AggregateFunctionDescriptor
```

y capabilities.

---

# 83. COUNT nullability

`COUNT` normalmente produce:

```text
NON_NULL
```

aunque la entrada sea nullable.

---

# 84. SUM nullability

`SUM` sobre conjunto vacío puede producir:

```text
NULL
```

según semántica SQL.

Por tanto:

```text
input nullability
≠
aggregate output nullability
```

---

# 85. CASE expression

Ejemplo:

```sql
CASE
    WHEN active THEN 'yes'
    ELSE 'no'
END
```

requiere un common type entre branches.

---

# 86. CASE result

```text
CommonType(
    type('yes'),
    type('no')
)
→ core.string
```

---

# 87. Mixed CASE

```text
INTEGER
+
DECIMAL
```

puede producir:

```text
DECIMAL
```

si la promoción es lossless.

---

# 88. Incompatible CASE

```text
UUID
+
JSON
```

debe fallar salvo conversión explícita válida.

---

# 89. CASE without ELSE

Debe considerar:

```text
implicit NULL
```

por tanto el resultado puede volverse nullable.

---

# 90. COALESCE

```text
COALESCE(T1, T2, ...)
```

requiere:

```text
CommonType(T1, T2, ...)
```

y análisis específico de nullability.

---

# 91. NULLIF

`NULLIF(a,b)`:

```text
result type ≈ type(a)
```

pero:

```text
nullable = true
```

---

# 92. CAST

```sql
CAST(value AS target_type)
```

produce:

```text
result EXACT target_type
```

pero debe validar:

```text
source → target coercion
```

---

# 93. Explicit lossy cast

Puede permitirse cuando el usuario lo solicita explícitamente y la Platform lo soporta.

---

# 94. Implicit lossy cast

Por defecto:

```text
forbidden
```

---

# 95. Cast target

El target del cast es:

```text
Query Semantic Type
```

que posteriormente Compiler/Platform mapearán a sintaxis física.

---

# 96. JSON typing

Debe mantenerse:

```text
JSON
≠
String
```

---

# 97. JSON extraction

La semántica puede producir diferentes tipos:

```text
JSON value
JSON scalar
String
Number
Boolean
Unknown JSON scalar
```

según operación.

---

# 98. JSON null

Debe mantenerse:

```text
SQL NULL
≠
JSON null
≠
JSON missing
```

---

# 99. Date/time typing

Tipos separados:

```text
Date
Time
DateTime
DateTimeTz
Interval
```

---

# 100. Temporal arithmetic

Ejemplos:

```text
Date + Interval → Date/DateTime
DateTime + Interval → DateTime
DateTime - DateTime → Interval
```

según reglas semánticas/capabilities.

---

# 101. UUID typing

UUID conservará identidad semántica independientemente de si físicamente se almacena como:

```text
UUID
CHAR(36)
BINARY(16)
TEXT
```

---

# 102. Enum typing

Un enum:

```text
app.user_status
```

no será reducido automáticamente a:

```text
core.string
```

aunque tenga backing string.

---

# 103. Enum literal comparison

```text
user.status = :status
```

debe inferir:

```text
:status → app.user_status
```

---

# 104. Domain type propagation

Ejemplo:

```text
users.id → app.user_id
```

Entonces:

```sql
WHERE users.id = :id
```

propaga:

```text
:id → app.user_id
```

---

# 105. Domain arithmetic

Por defecto:

```text
UserId + 1
```

puede ser inválido aunque el storage sea integer.

---

# 106. Domain characteristics

Un DomainType deberá declarar explícitamente qué operaciones soporta.

---

# 107. Domain unwrap

No ocurrirá implícitamente para todas las operaciones.

---

# 108. Domain conversion

Podrá existir:

```text
DomainTypeDescriptor
├── underlyingType
├── supportedOperations
├── coercionRules
└── conversionRules
```

---

# 109. Subquery expression

Ejemplo:

```sql
WHERE user_id = (
    SELECT owner_id
    FROM projects
    WHERE ...
)
```

la subquery escalar deberá producir exactamente una output column semántica.

---

# 110. Scalar subquery type

```text
type(subquery)
=
type(single projection)
```

---

# 111. Multi-column scalar subquery

Debe producir error.

---

# 112. Tuple subquery

Cuando el contexto permita tuple:

```text
Tuple<T1, T2, ...>
```

podrá ser válido.

---

# 113. EXISTS

`EXISTS(subquery)` no necesita inferir un scalar output type.

Produce predicate truth semantics.

---

# 114. CTE type inference

El output de un CTE deriva de:

```text
projection expressions
```

---

# 115. CTE explicit column names

Los nombres pueden cambiar, pero los tipos provienen de las expressions correspondientes.

---

# 116. Recursive CTE

Requiere resolución especial.

Ejemplo:

```text
Anchor Query
+
Recursive Query
```

deben producir outputs compatibles.

---

# 117. Recursive CTE fixed point

Podrá requerirse:

```text
provisional output types
→ recursive constraints
→ convergence
```

---

# 118. No infinite inference

El solver tendrá límites de iteración.

---

# 119. UNION

Para:

```sql
SELECT id FROM users
UNION
SELECT user_id FROM orders
```

debe resolver common type por posición.

---

# 120. Set operation arity

Primero:

```text
same number of output columns
```

Luego:

```text
compatible type per ordinal
```

---

# 121. Set operation example

```text
UserId
UNION
UserId
→ UserId
```

---

# 122. Domain mismatch set operation

```text
UserId
UNION
OrderId
```

debe fallar por defecto.

---

# 123. VALUES

```sql
VALUES
(1, 'A'),
(2, 'B')
```

produce:

```text
column1 → integer
column2 → string
```

mediante common type vertical.

---

# 124. VALUES with NULL

```text
(1)
(NULL)
```

produce:

```text
integer nullable
```

---

# 125. Projection types

Cada projection deberá obtener:

```text
ProjectionTypeInfo
```

---

# 126. Structure

```text
ProjectionTypeInfo
├── projectionId
├── resolvedType
├── nullability
├── provenance
└── certainty
```

---

# 127. QueryTypeTable

El resultado principal será:

```text
QueryTypeTable
```

---

# 128. Entries

Podrá mapear:

```text
AstNodeId
ParameterId
ProjectionId
SymbolId
OutputColumnId
```

hacia:

```text
ResolvedQueryType
```

---

# 129. Example

```text
AstNode N12 → app.user_id
Parameter id → app.user_id
Symbol C7 → app.email_address
Projection P1 → app.email_address
```

---

# 130. ResolvedQueryType

Conceptualmente:

```text
ResolvedQueryType
├── type
├── nullability
├── certainty
├── provenance
├── coercions
├── capabilityRequirements
└── diagnostics
```

---

# 131. Type certainty

Estados sugeridos:

```text
EXACT
RESOLVED
INFERRED
PROMOTED
COERCED
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 132. Provenance

Debe explicar por qué se resolvió el tipo.

Ejemplo:

```text
Parameter :id
→ compared with users.id
→ users.id schema column
→ ORM mapping UserId
→ app.user_id
```

---

# 133. TypeInferenceTrace

En modo diagnóstico podrá construirse:

```text
TypeInferenceTrace
```

---

# 134. Example trace

```text
T17 (:id)

Evidence:
  E1 ParameterDefinition = unknown
  E2 Comparison operand with T4
  E3 T4 schema type = app.user_id

Constraints:
  C1 T17 compatible T4
  C2 T4 exact app.user_id

Result:
  T17 = app.user_id
```

---

# 135. Trace production

Debe ser opcional para evitar overhead en producción.

---

# 136. Diagnostics without values

Nunca incluir secretos/runtime values innecesariamente.

---

# 137. Conflict detection

Ejemplo:

```sql
WHERE users.id = :value
  AND users.uuid = :value
```

si:

```text
users.id   → UserId
users.uuid → UUID
```

el mismo parameter produce:

```text
T_parameter compatible UserId
T_parameter compatible UUID
```

---

# 138. Result

```text
ConflictingParameterTypeException
```

---

# 139. No silent string fallback

Nunca resolver el conflicto como:

```text
VARCHAR
```

simplemente porque ambos puedan representarse como texto.

---

# 140. Type mismatch diagnostic

Ejemplo:

```text
DB-QTYPE-INF-001

Parameter ":value" has incompatible type requirements.

Occurrence 1:
  users.id
  expected: app.user_id

Occurrence 2:
  users.uuid
  expected: core.uuid
```

---

# 141. Unknown type policy

Al finalizar puede haber variables:

```text
UNKNOWN
```

---

# 142. UnknownTypePolicy

Modos:

```text
STRICT
ALLOW_UNKNOWN
ALLOW_OPAQUE
```

---

# 143. Strict mode

En `STRICT`, si el tipo es necesario para:

```text
semantic correctness
binding
compilation
```

deberá producir error.

---

# 144. Allow unknown

Puede utilizarse para:

```text
raw SQL
legacy integration
dynamic schemas
extension nodes
```

con garantías reducidas.

---

# 145. Unknown propagation

`Unknown` no deberá contaminar automáticamente todo.

Ejemplo:

```text
known + unknown
```

puede generar constraints que posteriormente resuelvan `unknown`.

---

# 146. Unknown ≠ Any

Reiteración crítica:

```text
Unknown
≠
compatible with everything
```

---

# 147. NullType

`NULL` literal tiene:

```text
NullType
```

pero:

```text
NullType
≠
Nullable<T>
```

---

# 148. Common type with NULL

```text
CommonType(Integer, Null)
→ Integer + NULLABLE
```

---

# 149. Nullability solver

Aunque relacionado con tipos, puede existir:

```text
QueryNullabilitySolver
```

separado o coordinado.

---

# 150. Nullability evidence

Proviene de:

```text
Schema
Literal NULL
Outer joins
CASE
COALESCE
Aggregates
Functions
Operators
Predicates
Extensions
```

---

# 151. Type vs nullability

Mantener:

```text
Resolved Type
+
Resolved Nullability
```

como dimensiones separadas.

---

# 152. Nullability lattice

Conceptualmente:

```text
NON_NULL
NULLABLE
UNKNOWN
```

---

# 153. Outer join

Como se definió en doc 38:

```text
orders.id schema NOT NULL
```

puede convertirse en:

```text
orders.id query NULLABLE
```

dentro de un LEFT JOIN.

---

# 154. Predicate refinement

Una condición:

```sql
WHERE email IS NOT NULL
```

puede permitir razonamiento contextual de nullability.

---

# 155. Flow-sensitive typing

Este tipo de refinamiento deberá tratarse con cuidado.

---

# 156. V1 policy

V1 podrá soportar refinamientos simples:

```text
IS NULL
IS NOT NULL
```

sin construir todavía un sistema completo de flow-sensitive types.

---

# 157. Branch-sensitive typing

CASE puede necesitar información por branch.

Ejemplo:

```sql
CASE
WHEN value IS NOT NULL THEN value
ELSE 'default'
END
```

---

# 158. Constraint analyzer integration

Parte del razonamiento avanzado podrá delegarse posteriormente a:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

---

# 159. Type inference boundary

Este sistema no debe convertirse en:

```text
full theorem prover
```

---

# 160. Compatibility engine

Componente:

```text
QueryTypeCompatibilityResolver
```

---

# 161. Compatibility result

```text
EXACT
COMPATIBLE
PROMOTABLE
COERCIBLE
PLATFORM_DEPENDENT
INCOMPATIBLE
UNKNOWN
```

---

# 162. Promotion

Promotion debe ser:

```text
safe
lossless
semantically valid
```

por defecto.

---

# 163. Coercion

Coercion podrá clasificarse:

```text
IMPLICIT_SAFE
IMPLICIT_LOSSLESS
EXPLICIT_SAFE
EXPLICIT_LOSSY
PLATFORM_SPECIFIC
UNSUPPORTED
```

---

# 164. Implicit coercion policy

Por defecto sólo:

```text
IMPLICIT_SAFE
IMPLICIT_LOSSLESS
```

---

# 165. Explicit cast policy

Puede habilitar:

```text
EXPLICIT_SAFE
EXPLICIT_LOSSY
```

cuando la Platform lo soporte.

---

# 166. CommonTypeResolver

Componente central:

```text
CommonTypeResolver
```

---

# 167. Inputs

```text
types
operation context
coercion policy
capabilities
domain policy
```

---

# 168. Output

```text
CommonTypeResolution
├── resolvedType
├── promotions
├── coercions
├── nullability
├── certainty
└── diagnostics
```

---

# 169. No universal common string

Nunca:

```text
if incompatible → string
```

---

# 170. Operator overloads

Operadores extension pueden registrar signatures.

---

# 171. OperatorDescriptorRegistry

Será:

```text
immutable after bootstrap
```

---

# 172. Function registry

Igualmente:

```text
FunctionDescriptorRegistry
```

se congela después del bootstrap.

---

# 173. Extension types

Un paquete podrá registrar:

```text
Money
Vector
Geometry
IPAddress
Range
FullTextVector
```

---

# 174. Extension requirements

Cada type extension deberá declarar:

```text
TypeDescriptor
CompatibilityRules
CoercionRules
OperationCapabilities
FunctionSignatures
OperatorSignatures
PlatformMappings
ConversionRules
FingerprintVersion
```

según aplique.

---

# 175. No partial unsafe extension

Si un extension type participa en una operación sin regla conocida:

```text
UnsupportedTypeOperationException
```

---

# 176. Extension inference rule

Podrá registrar:

```text
TypeInferenceRule
```

para AST nodes propios.

---

# 177. Extension registry

```text
TypeInferenceExtensionRegistry
```

frozen después de bootstrap.

---

# 178. Deterministic precedence

Dos reglas que reclaman el mismo node kind deberán tener:

```text
explicit priority
```

o producir configuración inválida.

---

# 179. No service locator

Las inference rules no obtendrán servicios mediante:

```text
Container::get()
```

---

# 180. TypeInferenceContext

El contexto específico podrá contener:

```text
QueryTypeRegistry
SchemaResolutionTable
SymbolResolutionTable
FunctionRegistry
OperatorRegistry
CapabilitySnapshot
PolicySnapshot
QueryMetadata
DiagnosticSink
InferenceBudget
```

---

# 181. No runtime resources

No contendrá:

```text
PDO
ConnectionLease
Transaction
EntityManager
UnitOfWork
Request
Session
```

---

# 182. TypeInferenceState

El estado temporal podrá contener:

```text
TypeConstraintGraph
TypeVariableTable
WorkQueue
ResolvedTypeBuilder
NullabilityState
ConflictSet
IterationCounter
InferenceMemo
```

---

# 183. Context vs state

```text
TypeInferenceContext
=
stable operation inputs

TypeInferenceState
=
temporary mutable solving progress
```

---

# 184. Output artifact

```text
TypeInferenceResult
```

---

# 185. Structure

```text
TypeInferenceResult
├── QueryTypeTable
├── NullabilityTable
├── AppliedCoercions
├── TypeRequirements
├── Diagnostics
├── Provenance
└── InferenceFingerprint
```

---

# 186. AppliedCoercion

Cuando se autorice una coercion:

```text
AppliedCoercion
├── nodeId
├── sourceType
├── targetType
├── coercionKind
├── reason
└── platformRequirement?
```

---

# 187. AST mutation

El Type Inference System no deberá insertar casts físicos mutando AST arbitrariamente.

---

# 188. Semantic coercion annotation

Puede producir:

```text
SemanticCoercionRequirement
```

para fases posteriores.

---

# 189. Explicit semantic rewrite

Si la arquitectura requiere insertar un semantic cast node, deberá hacerse mediante una transformación formal que produzca un nuevo AST.

Nunca mediante mutation.

---

# 190. Platform-dependent types

Algunas operaciones pueden requerir target platform.

---

# 191. Offline analysis

Si no existe Platform concreta:

```text
PLATFORM_DEPENDENT
```

podrá mantenerse como resultado cuando policy lo permita.

---

# 192. Strict portable mode

En:

```text
STRICT_PORTABLE
```

una operación cuyo tipo no pueda resolverse portablemente puede fallar.

---

# 193. Targeted compilation

Si existe:

```text
PostgreSQL Capability Snapshot
```

pueden resolverse reglas adicionales sin conexión física.

---

# 194. Capability requirements

Type inference puede producir:

```text
CapabilityRequirementSet
```

---

# 195. Example

Una operación JSON avanzada puede requerir:

```text
query.json.path
```

---

# 196. Requirement vs decision

Type inference declara:

```text
required capability
```

No decide:

```text
how SQL will implement it
```

---

# 197. Planner boundary

El Planner decidirá estrategias físicas.

---

# 198. Compiler boundary

El Compiler seleccionará sintaxis según:

```text
Dialect
Platform
Resolved Types
Plan
```

---

# 199. Driver boundary

El Driver recibe:

```text
database representations
```

no Query Type inference rules.

---

# 200. Parameter binding integration

El resultado:

```text
ParameterId :id
→ app.user_id
```

alimenta posteriormente:

```text
BindingPlanner
```

---

# 201. Conversion pipeline

```text
Runtime Value
      │
      ▼
Resolved Query Type
      │
      ▼
Platform Type Mapping
      │
      ▼
Value Conversion
      │
      ▼
Database Representation
      │
      ▼
Driver Binding
```

---

# 202. No conversion during inference

Type inference no convierte:

```text
UserId object
```

en:

```text
123
```

Eso pertenece al Value Conversion System.

---

# 203. Fingerprinting

El resultado deberá ser fingerprintable.

---

# 204. Inference fingerprint

Conceptualmente:

```text
TypeInferenceFingerprint
=
Normalized Query Shape
+
Schema Semantic Fingerprint
+
Type Registry Version
+
Function Registry Version
+
Operator Registry Version
+
Type Policy Fingerprint
+
Relevant Capability Fingerprint
+
Extension Versions
```

---

# 205. Runtime values excluded

No incluir:

```text
parameter values
passwords
tokens
request IDs
connection IDs
```

---

# 206. Cache

Podrá existir:

```text
TypeInferenceCache
```

como optimización futura.

---

# 207. Cache key

Debe depender de toda información que pueda cambiar el resultado semántico.

---

# 208. Schema invalidation

Si:

```text
users.id
```

cambia de:

```text
INTEGER
```

a:

```text
UUID
```

el artifact de inferencia anterior no puede reutilizarse.

---

# 209. Registry invalidation

Si cambia una:

```text
FunctionSignature
```

también puede invalidarse.

---

# 210. Extension versioning

Cada extensión de tipos deberá aportar:

```text
semantic version/fingerprint
```

para cache correctness.

---

# 211. Inference convergence

El solver deberá alcanzar:

```text
fixed point
```

---

# 212. Formula

```text
solve(solve(C)) = solve(C)
```

en términos de resultado semántico estable.

---

# 213. Worklist algorithm

Una estrategia apropiada:

```text
Initialize variables
      │
      ▼
Collect constraints
      │
      ▼
Queue affected constraints
      │
      ▼
Propagate evidence
      │
      ▼
Update variables
      │
      ▼
Requeue dependents
      │
      ▼
Fixed point
```

---

# 214. No blind full rescans

No recorrer todas las constraints en cada cambio si puede utilizarse:

```text
dependency-indexed work queue
```

---

# 215. InferenceBudget

Debe existir:

```text
InferenceBudget
```

---

# 216. Limits

Puede controlar:

```text
maxTypeVariables
maxConstraints
maxIterations
maxWorkItems
maxOverloadCandidates
maxCommonTypeCandidates
maxRecursiveCteIterations
maxExtensionSteps
```

---

# 217. Budget exceeded

Produce:

```text
TypeInferenceBudgetExceededException
```

---

# 218. No partial success

Un resultado incompleto por budget no será tratado como inferencia válida.

---

# 219. Recursive cycle detection

Constraints cíclicos son normales:

```text
T1 compatible T2
T2 compatible T1
```

pero ciclos sin evidencia concreta pueden quedar:

```text
UNKNOWN
```

---

# 220. Conflict cycle

Si posteriormente:

```text
T1 exact Integer
T2 exact UUID
```

debe detectarse incompatibilidad.

---

# 221. Strongly connected components

El solver podrá utilizar SCCs en versiones avanzadas para optimizar grupos equivalentes.

---

# 222. Determinism

Misma entrada:

```text
AST
Schema Snapshot
Registries
Policies
Capabilities
```

debe producir el mismo resultado.

---

# 223. No environment reads

Nunca depender de:

```text
current time
random
environment variables
current request
current user
current tenant global
```

---

# 224. Persistent runtime safety

Los registries pueden ser:

```text
application-scoped + immutable
```

El inference state será:

```text
operation-scoped
```

---

# 225. FrankenPHP

```text
Worker
├── Frozen Type Registry
├── Frozen Function Registry
├── Frozen Operator Registry
│
├── Query A
│   └── TypeInferenceState A
│
└── Query B
    └── TypeInferenceState B
```

---

# 226. Coroutine safety

OpenSwoole:

```text
Coroutine A → State A
Coroutine B → State B
```

sin globals.

---

# 227. No static current solver state

Nunca:

```php
TypeInference::$currentParameterTypes
```

---

# 228. Error taxonomy

```text
QueryTypeInferenceException
UnresolvedQueryTypeException
ConflictingTypeConstraintException
ConflictingParameterTypeException
IncompatibleOperandTypeException
IncompatibleDomainTypeException
AmbiguousFunctionOverloadException
NoMatchingFunctionSignatureException
AmbiguousOperatorOverloadException
NoMatchingOperatorSignatureException
NoCommonTypeException
UnsupportedImplicitCoercionException
LossyImplicitCoercionException
InvalidCastException
TupleTypeArityException
CollectionElementTypeException
RecursiveTypeInferenceException
TypeInferenceBudgetExceededException
TypeExtensionConflictException
```

---

# 229. Diagnostic codes

Sugeridos:

```text
DB-QTYPE-INF-001 CONFLICTING_TYPE_CONSTRAINT
DB-QTYPE-INF-002 UNRESOLVED_TYPE
DB-QTYPE-INF-003 INCOMPATIBLE_OPERANDS
DB-QTYPE-INF-004 INCOMPATIBLE_DOMAIN_TYPES
DB-QTYPE-INF-005 NO_COMMON_TYPE
DB-QTYPE-INF-006 AMBIGUOUS_FUNCTION_OVERLOAD
DB-QTYPE-INF-007 NO_FUNCTION_SIGNATURE
DB-QTYPE-INF-008 AMBIGUOUS_OPERATOR_OVERLOAD
DB-QTYPE-INF-009 NO_OPERATOR_SIGNATURE
DB-QTYPE-INF-010 UNSAFE_IMPLICIT_COERCION
DB-QTYPE-INF-011 LOSSY_IMPLICIT_COERCION
DB-QTYPE-INF-012 INVALID_CAST
DB-QTYPE-INF-013 TUPLE_ARITY_MISMATCH
DB-QTYPE-INF-014 COLLECTION_ELEMENT_MISMATCH
DB-QTYPE-INF-015 RECURSIVE_INFERENCE_FAILED
DB-QTYPE-INF-016 BUDGET_EXCEEDED
DB-QTYPE-INF-017 EXTENSION_CONFLICT
DB-QTYPE-INF-018 PLATFORM_DEPENDENT_TYPE
```

---

# 230. Diagnostic quality

Un error deberá responder:

```text
What failed?
Where?
Which types were involved?
Why were those types inferred?
Which rule failed?
Was coercion possible?
What explicit action could resolve it?
```

---

# 231. Example diagnostic

```text
DB-QTYPE-INF-004

Cannot compare values of domain types:

Left:
  app.user_id
  source: users.id

Right:
  app.order_id
  source: orders.id

Both types use an integer storage representation,
but they represent different semantic domains.
```

---

# 232. Testing architecture

El sistema requerirá:

```text
Unit tests
Constraint solver tests
Expression typing tests
Predicate typing tests
Function overload tests
Operator overload tests
Domain type tests
Nullability tests
Set operation tests
Recursive CTE tests
Extension tests
Property-based tests
Persistent runtime tests
Concurrency tests
Architecture tests
```

---

# 233. Property test: determinism

```text
infer(Q, S, P)
=
infer(Q, S, P)
```

---

# 234. Property test: order independence

Cuando constraints sean semánticamente equivalentes:

```text
solve(C1, C2, C3)
=
solve(C3, C1, C2)
```

---

# 235. Property test: idempotent result

```text
resolve(resolve(C))
=
resolve(C)
```

---

# 236. Property test: no silent loss

Ninguna coercion implícita podrá perder información sin producir error/policy explícita.

---

# 237. Parameter tests

Probar:

```text
column → parameter inference
explicit parameter type
same parameter multiple occurrences
compatible occurrences
conflicting occurrences
nullable parameter
collection parameter
tuple parameter
unknown parameter
```

---

# 238. Function tests

```text
single signature
multiple overloads
generic signature
ambiguous overload
no matching overload
platform-dependent signature
extension function
```

---

# 239. Arithmetic tests

```text
integer + integer
integer + decimal
decimal + decimal
decimal + float
string + integer invalid
domain + integer invalid
```

---

# 240. CASE tests

```text
same type branches
numeric promotion
NULL branch
no ELSE
incompatible domains
no common type
```

---

# 241. Set operation tests

```text
same type
numeric promotion
NULL
domain mismatch
arity mismatch
recursive union
```

---

# 242. Schema tests

```text
schema known type
unknown type
partial schema
ORM enrichment
schema/ORM conflict
schema version invalidation
```

---

# 243. Persistent worker tests

```text
Query A infers UserId
reset
Query B infers OrderId
```

No state de A puede aparecer en B.

---

# 244. Concurrency test

```text
Fiber A
:id → UserId

Fiber B
:id → UUID
```

simultáneamente.

---

# 245. Architecture tests

El namespace de inferencia no podrá importar:

```text
PDO
NativeConnection
ConnectionLease
ConnectionManager
QueryExecutor
EntityManager
UnitOfWork
IdentityMap
HTTP Request
Session
Service Container
```

---

# 246. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Semantic\TypeInference\
```

---

# 247. Estructura propuesta

```text
Query/
└── Semantic/
    └── TypeInference/
        ├── Contract/
        │   ├── TypeInferenceRuleInterface.php
        │   ├── ExpressionTypeRuleInterface.php
        │   ├── PredicateTypeRuleInterface.php
        │   └── TypeConstraintSolverInterface.php
        │
        ├── Core/
        │   ├── QueryTypeInferenceEngine.php
        │   ├── QueryTypeConstraintCollector.php
        │   ├── QueryTypeConstraintSolver.php
        │   ├── QueryTypeCompatibilityResolver.php
        │   └── CommonTypeResolver.php
        │
        ├── Variable/
        │   ├── TypeVariable.php
        │   ├── TypeVariableId.php
        │   └── TypeVariableTable.php
        │
        ├── Constraint/
        │   ├── TypeConstraint.php
        │   ├── TypeConstraintGraph.php
        │   ├── ExactTypeConstraint.php
        │   ├── CompatibilityConstraint.php
        │   ├── EqualityConstraint.php
        │   ├── FamilyConstraint.php
        │   ├── CommonTypeConstraint.php
        │   ├── FunctionArgumentConstraint.php
        │   ├── OperatorOperandConstraint.php
        │   ├── CollectionElementConstraint.php
        │   └── TupleElementConstraint.php
        │
        ├── Rule/
        │   ├── LiteralTypeRule.php
        │   ├── ColumnTypeRule.php
        │   ├── ParameterTypeRule.php
        │   ├── ArithmeticTypeRule.php
        │   ├── FunctionTypeRule.php
        │   ├── AggregateTypeRule.php
        │   ├── CaseTypeRule.php
        │   ├── CastTypeRule.php
        │   ├── SubqueryTypeRule.php
        │   ├── TupleTypeRule.php
        │   └── CollectionTypeRule.php
        │
        ├── Function/
        │   ├── FunctionSignatureResolver.php
        │   ├── FunctionOverloadCandidate.php
        │   └── FunctionOverloadResolution.php
        │
        ├── Operator/
        │   ├── OperatorSignatureResolver.php
        │   ├── OperatorOverloadCandidate.php
        │   └── OperatorOverloadResolution.php
        │
        ├── Common/
        │   ├── CommonTypeCandidate.php
        │   └── CommonTypeResolution.php
        │
        ├── Nullability/
        │   ├── QueryNullabilitySolver.php
        │   ├── NullabilityConstraint.php
        │   └── NullabilityTable.php
        │
        ├── Coercion/
        │   ├── CoercionResolver.php
        │   ├── AppliedCoercion.php
        │   └── SemanticCoercionRequirement.php
        │
        ├── Context/
        │   ├── TypeInferenceContext.php
        │   ├── TypeInferenceState.php
        │   └── InferenceBudget.php
        │
        ├── Result/
        │   ├── TypeInferenceResult.php
        │   ├── QueryTypeTable.php
        │   ├── ProjectionTypeInfo.php
        │   └── ResolvedQueryType.php
        │
        ├── Provenance/
        │   ├── TypeEvidence.php
        │   ├── TypeEvidenceSource.php
        │   ├── TypeInferenceTrace.php
        │   └── TypeInferenceProvenance.php
        │
        ├── Fingerprint/
        │   ├── TypeInferenceFingerprint.php
        │   └── TypeInferenceFingerprintBuilder.php
        │
        ├── Extension/
        │   ├── TypeInferenceExtensionRegistry.php
        │   ├── TypeInferenceExtensionDescriptor.php
        │   └── ExtensionTypeInferenceRule.php
        │
        ├── Diagnostic/
        │   ├── TypeInferenceDiagnostic.php
        │   └── TypeInferenceDiagnosticCode.php
        │
        └── Exception/
            ├── QueryTypeInferenceException.php
            ├── UnresolvedQueryTypeException.php
            ├── ConflictingTypeConstraintException.php
            ├── ConflictingParameterTypeException.php
            ├── IncompatibleOperandTypeException.php
            ├── IncompatibleDomainTypeException.php
            ├── NoCommonTypeException.php
            ├── AmbiguousFunctionOverloadException.php
            ├── NoMatchingFunctionSignatureException.php
            ├── InvalidCastException.php
            └── TypeInferenceBudgetExceededException.php
```

---

# 248. Invariantes arquitectónicos

## DB-QTYPE-INF-001

Query Type Inference será independiente de SQL físico.

## DB-QTYPE-INF-002

No dependerá de PDO.

## DB-QTYPE-INF-003

No abrirá conexiones.

## DB-QTYPE-INF-004

No ejecutará queries.

## DB-QTYPE-INF-005

No dependerá del Driver.

## DB-QTYPE-INF-006

No dependerá del Executor.

## DB-QTYPE-INF-007

No dependerá de EntityManager.

## DB-QTYPE-INF-008

No dependerá de UnitOfWork.

## DB-QTYPE-INF-009

No dependerá de IdentityMap.

## DB-QTYPE-INF-010

No utilizará Service Locator.

## DB-QTYPE-INF-011

Los tipos serán semánticos.

## DB-QTYPE-INF-012

PHP type no será Query Type.

## DB-QTYPE-INF-013

Physical DB type no será Query Type.

## DB-QTYPE-INF-014

Driver binding type no será Query Type.

## DB-QTYPE-INF-015

Unknown no será Any.

## DB-QTYPE-INF-016

NullType no será nullable type.

## DB-QTYPE-INF-017

Nullability será una dimensión separada.

## DB-QTYPE-INF-018

Type inference será constraint-based.

## DB-QTYPE-INF-019

No existirá first-type-wins.

## DB-QTYPE-INF-020

La inferencia será bidireccional cuando sea semánticamente válido.

## DB-QTYPE-INF-021

Toda evidencia tendrá provenance.

## DB-QTYPE-INF-022

Los conflictos serán explícitos.

## DB-QTYPE-INF-023

No habrá silent string fallback.

## DB-QTYPE-INF-024

No habrá silent lossy coercion.

## DB-QTYPE-INF-025

Implicit coercion será conservadora.

## DB-QTYPE-INF-026

Explicit casts serán validados.

## DB-QTYPE-INF-027

Domain types conservarán identidad.

## DB-QTYPE-INF-028

UserId no será OrderId por compartir storage type.

## DB-QTYPE-INF-029

Enum no será reducido automáticamente a backing type.

## DB-QTYPE-INF-030

UUID no será reducido automáticamente a string.

## DB-QTYPE-INF-031

JSON no será string.

## DB-QTYPE-INF-032

SQL NULL no será JSON null.

## DB-QTYPE-INF-033

Collection Type no será native array.

## DB-QTYPE-INF-034

Tuple arity será validada.

## DB-QTYPE-INF-035

Literal strings no se interpretarán arbitrariamente como UUID/JSON/date.

## DB-QTYPE-INF-036

Runtime inference será evidencia débil.

## DB-QTYPE-INF-037

Schema evidence será explícita.

## DB-QTYPE-INF-038

ORM metadata podrá enriquecer pero no falsificar schema.

## DB-QTYPE-INF-039

Schema/ORM conflicts serán diagnosticados.

## DB-QTYPE-INF-040

Functions usarán signatures.

## DB-QTYPE-INF-041

Operators usarán signatures.

## DB-QTYPE-INF-042

Overloads no usarán first-match.

## DB-QTYPE-INF-043

Overload ambiguity será error.

## DB-QTYPE-INF-044

Common type resolution será explícita.

## DB-QTYPE-INF-045

Incompatible CASE branches producirán error.

## DB-QTYPE-INF-046

Set operations validarán tipos por ordinal.

## DB-QTYPE-INF-047

Set operations validarán aridad.

## DB-QTYPE-INF-048

VALUES inferirá tipos por columna.

## DB-QTYPE-INF-049

Scalar subquery requerirá una columna.

## DB-QTYPE-INF-050

Recursive CTE inference será bounded.

## DB-QTYPE-INF-051

Aggregate typing será descriptor-driven.

## DB-QTYPE-INF-052

Aggregate nullability tendrá reglas propias.

## DB-QTYPE-INF-053

Arithmetic promotion preservará precisión.

## DB-QTYPE-INF-054

Decimal no será convertido silenciosamente a float.

## DB-QTYPE-INF-055

Platform-dependent semantics serán explícitas.

## DB-QTYPE-INF-056

Capabilities serán requirements, no SQL decisions.

## DB-QTYPE-INF-057

Planner elegirá estrategias físicas.

## DB-QTYPE-INF-058

Compiler elegirá sintaxis física.

## DB-QTYPE-INF-059

Driver sólo manejará representation/binding.

## DB-QTYPE-INF-060

Type inference no convertirá runtime values.

## DB-QTYPE-INF-061

AST permanecerá immutable.

## DB-QTYPE-INF-062

Semantic coercions no mutarán AST.

## DB-QTYPE-INF-063

Type variables serán operation-scoped.

## DB-QTYPE-INF-064

Constraint graph será operation-scoped.

## DB-QTYPE-INF-065

Registries compartidos serán immutable.

## DB-QTYPE-INF-066

Registry mutation después de bootstrap estará prohibida.

## DB-QTYPE-INF-067

Extension precedence será determinista.

## DB-QTYPE-INF-068

Extension conflicts serán explícitos.

## DB-QTYPE-INF-069

Inference tendrá budget.

## DB-QTYPE-INF-070

Budget exhaustion no producirá partial success.

## DB-QTYPE-INF-071

Inference deberá converger.

## DB-QTYPE-INF-072

El resultado será determinista.

## DB-QTYPE-INF-073

Constraint insertion order no deberá alterar semántica.

## DB-QTYPE-INF-074

QueryTypeTable será immutable después de publicación.

## DB-QTYPE-INF-075

QueryTypeTable permanecerá separada del AST.

## DB-QTYPE-INF-076

Resolved types no serán almacenados mediante mutable node properties.

## DB-QTYPE-INF-077

Type inference result será fingerprintable.

## DB-QTYPE-INF-078

Fingerprints excluirán runtime values.

## DB-QTYPE-INF-079

Schema version relevante afectará fingerprints.

## DB-QTYPE-INF-080

Registry versions relevantes afectarán fingerprints.

## DB-QTYPE-INF-081

Extension semantic versions afectarán fingerprints.

## DB-QTYPE-INF-082

TypeInferenceContext no contendrá Connection.

## DB-QTYPE-INF-083

TypeInferenceContext no contendrá Transaction.

## DB-QTYPE-INF-084

TypeInferenceContext no contendrá Request.

## DB-QTYPE-INF-085

TypeInferenceContext no contendrá Session.

## DB-QTYPE-INF-086

TypeInferenceContext no contendrá Tenant Entity.

## DB-QTYPE-INF-087

No existirá global current type inference state.

## DB-QTYPE-INF-088

Queries concurrentes tendrán states independientes.

## DB-QTYPE-INF-089

Persistent workers no compartirán inference state mutable.

## DB-QTYPE-INF-090

Diagnostics no expondrán valores sensibles.

## DB-QTYPE-INF-091

Inference traces serán opcionales.

## DB-QTYPE-INF-092

Unknown types no serán ocultados.

## DB-QTYPE-INF-093

Strict mode requerirá resolver tipos necesarios para binding/compilation.

## DB-QTYPE-INF-094

Permissive mode reducirá garantías explícitamente.

## DB-QTYPE-INF-095

Schema partial knowledge será respetado.

## DB-QTYPE-INF-096

No se inferirá ausencia a partir de metadata desconocida.

## DB-QTYPE-INF-097

Type inference podrá ejecutarse offline.

## DB-QTYPE-INF-098

Targeted offline compilation podrá utilizar CapabilitySnapshot.

## DB-QTYPE-INF-099

Type inference alimentará Semantic Graph sin depender de él circularmente.

## DB-QTYPE-INF-100

Toda resolución final deberá ser explicable mediante evidencia y constraints.

---

# 249. Anti-patrones

## Anti-pattern 1 — PHP type as query type

```php
is_int($value)
→ BIGINT
```

Incorrecto.

---

## Anti-pattern 2 — Physical type as domain type

```text
BIGINT
→ UserId
```

sin metadata adicional.

Incorrecto.

---

## Anti-pattern 3 — First type wins

```text
:first occurrence determines parameter type
```

Incorrecto.

---

## Anti-pattern 4 — String fallback

```text
incompatible types
→ VARCHAR
```

Incorrecto.

---

## Anti-pattern 5 — Cast until valid

```text
type mismatch
→ insert CAST automatically
```

Incorrecto.

---

## Anti-pattern 6 — Server as type checker

```text
send invalid SQL
and see if DB accepts it
```

Incorrecto.

---

## Anti-pattern 7 — Mutable AST typing

```php
$node->resolvedType = $type;
```

Incorrecto.

---

## Anti-pattern 8 — ORM owns query types

Esto impediría usar Query Engine sin ORM.

Incorrecto.

---

## Anti-pattern 9 — PDO constants in semantic types

```php
PDO::PARAM_INT
```

no pertenece al Query Type System.

---

## Anti-pattern 10 — Global inference state

Especialmente peligroso con FrankenPHP/OpenSwoole.

---

# 250. Ejemplo completo — UserId

Consulta:

```php
DB::table('users')
    ->where('id', $userId)
    ->first();
```

Builder produce:

```text
ComparisonPredicate
├── ColumnReference(users.id)
└── ParameterExpression(p1)
```

Schema resolution:

```text
users.id
→ app.user_id
```

Constraint generation:

```text
T_column exact app.user_id
T_parameter compatible T_column
```

Solver:

```text
T_column = app.user_id
T_parameter = app.user_id
```

Binding metadata:

```text
p1
→ app.user_id
```

Posteriormente:

```text
UserId object
→ Value Converter
→ integer DB representation
→ Driver Binding
```

---

# 251. Ejemplo completo — conflicto de dominios

Consulta:

```sql
SELECT *
FROM users u
JOIN orders o
    ON u.id = o.id
```

Schema/ORM:

```text
u.id → app.user_id
o.id → app.order_id
```

Constraints:

```text
Comparable(UserId, OrderId)
```

Resultado:

```text
INCOMPATIBLE_DOMAIN_TYPES
```

aunque ambos utilicen:

```text
BIGINT
```

físicamente.

---

# 252. Ejemplo completo — CASE

```sql
CASE
    WHEN amount > 100
    THEN amount
    ELSE 0
END
```

Evidence:

```text
amount → Decimal(12,2)
0 → Integer
```

Common type:

```text
Decimal(12,2)
```

Resultado:

```text
CASE → Decimal(12,2)
```

sin convertir Decimal a Float.

---

# 253. Ejemplo completo — collection parameter

```php
DB::table('users')
    ->whereIn('id', $ids)
    ->get();
```

AST:

```text
InPredicate
├── Column(users.id)
└── CollectionParameter(p1)
```

Schema:

```text
users.id → app.user_id
```

Constraint:

```text
elementType(p1) compatible app.user_id
```

Inference:

```text
p1 → Collection<app.user_id>
```

Planner posteriormente decide:

```text
expanded placeholders
native array
VALUES relation
temporary relation
```

según capabilities.

---

# 254. Ejemplo completo — UNION

```sql
SELECT user_id AS id
FROM sessions

UNION

SELECT id
FROM users
```

Evidence:

```text
sessions.user_id → app.user_id
users.id         → app.user_id
```

Common type:

```text
app.user_id
```

Output:

```text
id → app.user_id
```

---

# 255. Arquitectura final

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
                   Type Evidence Layer
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
       Schema           Functions          Operators
          │                 │                  │
          ├─────────────┐   │   ┌──────────────┤
          │             ▼   ▼   ▼              │
          │        Type Constraint             │
          │           Collector                │
          │             │                      │
          └─────────────┼──────────────────────┘
                        ▼
                Type Constraint Graph
                        │
                        ▼
                 Constraint Solver
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Types     Nullability  Coercions
             │          │          │
             └──────────┼──────────┘
                        ▼
                  QueryTypeTable
                        │
                        ▼
                Semantic Query Artifact
```

---

# 256. Relación con los siguientes sistemas

El resultado alimentará:

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

para comprender:

```text
relation outputs
join compatibility
join predicates
outer join nullability
correlation
relation dependencies
```

Después:

```text
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
```

podrá razonar sobre:

```text
equality constraints
nullability refinements
uniqueness
constant values
range constraints
relationship constraints
```

Finalmente:

```text
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

consolidará:

```text
Symbols
+
Schema Resolution
+
Types
+
Relations
+
Joins
+
Constraints
+
Dependencies
+
Capabilities
```

en el modelo semántico consumido por Optimizer y Planner.

---

# 257. Fórmulas maestras

## Type inference

```text
Resolved Query Type
=
Explicit Type Evidence
+
Schema Evidence
+
ORM / Domain Evidence
+
Expression Constraints
+
Operator Signatures
+
Function Signatures
+
Parameter Constraints
+
Common Type Rules
+
Coercion Policy
+
Capability Constraints
```

---

## Parameter inference

```text
Resolved Parameter Type
=
Declared Parameter Type
+
Occurrence Constraints
+
Compared Column Types
+
Function Argument Requirements
+
Operator Requirements
+
Collection / Tuple Shape
+
Safe Runtime Evidence
```

---

## Expression inference

```text
Expression Type
=
Child Types
+
Expression Semantics
+
Operator / Function Signature
+
Contextual Constraints
+
Promotion Rules
+
Coercion Rules
```

---

## Common type

```text
Common Type
=
Input Types
+
Domain Compatibility
+
Promotion Rules
+
Coercion Policy
+
Nullability
+
Platform Capabilities
```

---

## Safe inference

```text
Safe Query Type Inference
=
Structured Evidence
+
Explicit Constraints
+
Bidirectional Propagation
+
Deterministic Solver
+
Conservative Coercion
+
Domain Preservation
+
Nullability Awareness
+
Capability Awareness
+
Bounded Convergence
+
Explicit Diagnostics
```

---

# 258. Invariante central

> VoltStack nunca inferirá un tipo porque "parece correcto" según el valor PHP o porque un motor de base de datos probablemente pueda convertirlo.

El tipo deberá poder justificarse mediante:

```text
Evidence
      +
Constraints
      +
Semantic Rules
      +
Compatibility
      +
Explicit Policies
```

Por tanto:

```text
Type Inference
≠
Guessing
```

y:

```text
Storage Compatibility
≠
Semantic Compatibility
```

Esta distinción permitirá detectar errores que normalmente sólo aparecerían durante ejecución o, peor aún, serían aceptados silenciosamente por el motor de base de datos.

---

# 259. Estado del Semantic Query Engine

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
41 Constraint Analysis
          │
          ▼
42 Semantic Graph
```

Con este documento, VoltStack dispone ya de las bases para transformar una consulta desde una estructura sintáctica:

```text
Query AST
```

hasta una estructura donde:

```text
symbols are resolved
schema objects are known
parameters have semantic constraints
expressions have semantic types
domain identities are preserved
nullability can be reasoned about
functions/operators are validated
```

sin generar todavía SQL ni tocar una conexión física.

---

# 260. Próximo documento

```text
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
```

El siguiente sistema formalizará:

```text
Relation Sources
+
Relation Outputs
+
Join Graph
+
Join Predicates
+
Join Types
+
Outer Join Semantics
+
Correlation
+
Lateral Dependencies
+
Relation Visibility
+
Nullability Propagation
+
Relationship Evidence
```

permitiendo pasar de:

```text
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
```

a una representación semántica como:

```text
RelationGraph
├── R1 users
├── R2 orders
└── J1 LEFT_JOIN
    ├── left = R1
    ├── right = R2
    ├── predicate
    │   └── orders.user_id = users.id
    └── effects
        └── R2 output becomes nullable
```

sin elegir todavía algoritmos físicos como:

```text
Nested Loop
Hash Join
Merge Join
```

ya que esas decisiones pertenecerán al **Query Planner**.