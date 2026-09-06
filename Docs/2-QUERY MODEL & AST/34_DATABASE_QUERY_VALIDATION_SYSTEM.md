# 34_DATABASE_QUERY_VALIDATION_SYSTEM.md

# VoltStack Quantum Database
## Query Validation System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 34 — Query Validation System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Validation Layer  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del:

```text
VoltStack/Quantum/Database Query Validation System
```

El sistema será responsable de verificar que una consulta estructurada pueda continuar de forma segura y coherente a través del Query Engine.

Su función principal será transformar:

```text
Normalized Query AST
```

en:

```text
Validated Query Artifact
```

siempre que la consulta satisfaga las invariantes exigibles en la fase actual.

La arquitectura conceptual será:

```text
Normalized Query AST
        │
        ▼
Query Validation System
        │
        ├── Structural Validation
        ├── Expression Validation
        ├── Predicate Validation
        ├── Parameter Validation
        ├── Clause Validation
        ├── Metadata Validation
        ├── Complexity Validation
        └── Extension Validation
        │
        ▼
Validated Query Artifact
        │
        ▼
Semantic Query Engine
```

---

# 2. Regla fundamental

> Query Validation comprobará aquello que puede demostrarse con la información disponible en la fase actual, sin apropiarse de responsabilidades pertenecientes a Semantic Analysis, Planning, Compilation o Execution.

---

# 3. Problema arquitectónico

Una consulta puede ser:

```text
syntactically constructible
```

pero aun así contener una estructura inválida.

Ejemplo:

```text
SELECT
```

sin una representación válida de su projection.

Otro ejemplo:

```text
UPDATE users
SET []
```

Otro:

```text
AND()
```

Otro:

```text
ORDER BY
    expression
    direction = INVALID
```

Otro:

```text
LIMIT -100
```

Otro:

```text
INSERT users (name, email)
VALUES ('John')
```

Algunas de estas condiciones pueden detectarse sin conocer todavía:

- el schema real;
- la existencia de las tablas;
- los tipos efectivos;
- el Platform;
- las capabilities;
- la conexión;
- el servidor.

Deben rechazarse antes de Semantic Analysis.

---

# 4. Validation ≠ Construction

Idealmente, muchas estructuras inválidas deberán ser imposibles de construir.

Por ejemplo:

```php
final readonly class Limit
{
    public function __construct(
        public int $value,
    ) {
        if ($value < 0) {
            throw new InvalidArgumentException();
        }
    }
}
```

Sin embargo, no toda invariant puede residir en un constructor individual.

Ejemplo:

```text
InsertQuery
├── columns: 3
└── row values: 2
```

Cada objeto puede ser localmente válido, pero la combinación no lo es.

Por ello existirán varios niveles de validación.

---

# 5. Validation ≠ Normalization

El documento:

```text
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
```

establece que Normalization convierte representaciones equivalentes en una forma canónica.

Validation responde una pregunta diferente:

```text
¿Es válida esta representación para continuar?
```

Por tanto:

```text
Normalization
=
canonicalization

Validation
=
invariant verification
```

---

# 6. Ejemplo

Entrada:

```text
AND(
    A,
    AND(B, C)
)
```

Normalization:

```text
AND(
    A,
    B,
    C
)
```

Validation:

```text
¿AND tiene el número válido de operands?
¿Todos los children son PredicateNode?
¿El árbol es acíclico?
¿La profundidad está permitida?
```

---

# 7. Validation ≠ Semantic Analysis

Validation no deberá comprobar todavía:

```text
users table exists
users.email exists
email is VARCHAR
LOWER(email) is valid
users.id = orders.user_id has compatible types
function exists for target platform
```

Eso pertenece a:

```text
Semantic Query Engine
```

---

# 8. Validation ≠ Capability Validation

La validación estructural no deberá decidir:

```text
Does SQLite support this RETURNING form?
Does MySQL support this window feature?
Does PostgreSQL support this lock strategy?
```

Eso requiere:

```text
EffectiveCapabilitySet
```

y pertenece a fases posteriores.

---

# 9. Validation ≠ Authorization

El Query Validation System no decidirá si:

```text
current user may update users
```

Eso corresponde a:

```text
VoltStack/Quantum/Authorization
```

o a políticas de aplicación.

---

# 10. Validation ≠ Security Policy

Debe distinguirse entre:

```text
query structural safety
```

y:

```text
application authorization/security policy
```

El validador podrá detectar estructuras peligrosas o inválidas, pero no reemplazará Authorization.

---

# 11. Validation ≠ Mutation Safety Policy

Ejemplo:

```text
DELETE FROM users
```

puede ser estructuralmente válido.

Pero una política de seguridad de desarrollo puede exigir:

```text
explicit full-table mutation intent
```

Estas reglas deberán implementarse como una categoría explícita de validation/policy y no confundirse con la validez estructural básica.

---

# 12. Validation ≠ Optimization

Validation nunca deberá:

- reordenar joins;
- eliminar predicates;
- hacer constant folding;
- seleccionar índices;
- empujar predicates;
- deduplicar subqueries;
- cambiar planes.

---

# 13. Validation ≠ Compilation

Validation no deberá:

- quote identifiers;
- asignar placeholders;
- generar SQL;
- resolver sintaxis de dialecto.

---

# 14. Validation ≠ Execution

Validation no deberá:

- adquirir conexiones;
- abrir PDO;
- preparar statements;
- ejecutar queries;
- iniciar transacciones.

---

# 15. Objetivos

El sistema deberá proporcionar:

1. validación estructural;
2. validación por node;
3. validación de expresiones;
4. validación de predicates;
5. validación de parámetros;
6. validación de clauses;
7. validación de metadata;
8. validación de query shape;
9. validación de complejidad;
10. validación de extensiones;
11. errores estructurados;
12. diagnósticos precisos;
13. source mapping;
14. acumulación opcional de múltiples errores;
15. fail-fast configurable;
16. determinismo;
17. procesamiento offline;
18. persistent-runtime safety;
19. extensibilidad controlada;
20. preparación formal para Semantic Analysis.

---

# 16. No objetivos

El sistema no deberá:

- resolver tablas;
- resolver columnas;
- inferir tipos efectivos;
- consultar schema;
- descubrir server version;
- resolver Platform;
- seleccionar Dialect;
- elegir Connection;
- consultar capabilities dinámicas;
- generar SQL;
- ejecutar SQL;
- hidratar entidades;
- consultar estadísticas del DB;
- aplicar reglas de negocio.

---

# 17. Pipeline general

```text
Query Builder
     │
     ▼
Query Model
     │
     ▼
Query AST
     │
     ▼
Initial Structural Checks
     │
     ▼
Policy Transformations
     │
     ▼
Normalization
     │
     ▼
Normalized Query AST
     │
     ▼
Query Validation
     │
     ▼
Validated Query Artifact
     │
     ▼
Semantic Analysis
```

---

# 18. Validación por etapas

VoltStack utilizará validación incremental.

```text
Construction Validation
        │
        ▼
AST Structural Validation
        │
        ▼
Normalized AST Validation
        │
        ▼
Semantic Validation
        │
        ▼
Capability Validation
        │
        ▼
Plan Validation
        │
        ▼
Compilation Validation
        │
        ▼
Execution Preconditions
```

---

# 19. Principio

> Ninguna fase deberá esperar hasta Execution para detectar un error que podía demostrarse correctamente en una fase anterior.

---

# 20. Pero tampoco demasiado pronto

El principio inverso también aplica:

> Ninguna fase deberá rechazar una consulta por información que todavía no posee.

---

# 21. Ejemplo

Esto puede validarse temprano:

```text
LIMIT -1
```

Esto no:

```text
column foo does not exist
```

hasta disponer del schema/symbol context apropiado.

---

# 22. Validation layers

El sistema distinguirá al menos:

```text
LOCAL
STRUCTURAL
NORMALIZED
CONTEXTUAL
SEMANTIC
CAPABILITY
PLANNING
COMPILATION
EXECUTION
```

Este documento se concentra principalmente en:

```text
LOCAL
STRUCTURAL
NORMALIZED
CONTEXTUAL
```

dentro del Query AST.

---

# 23. Local validation

Se ejecuta en Value Objects o factories.

Ejemplo:

```text
Limit(-1)
```

debe rechazarse inmediatamente.

---

# 24. Structural validation

Comprueba relaciones entre nodes.

Ejemplo:

```text
InsertQuery
columns = [a, b]
row = [1]
```

---

# 25. Normalized validation

Comprueba invariantes prometidas por:

```text
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
```

Ejemplo:

```text
AND(
    A,
    AND(B, C)
)
```

no debería existir en el artifact final si la normalización garantiza flattening.

---

# 26. Contextual validation

Utiliza:

```text
QueryValidationContext
```

pero no el servidor DB.

Ejemplo:

```text
maximum AST depth
allowed raw expressions
mutation safety policy
extension policy
```

---

# 27. Semantic validation

Se realizará en documentos posteriores.

Ejemplo:

```text
ColumnReference(users.foo)
```

¿Existe realmente?

---

# 28. Capability validation

Ejemplo:

```text
RIGHT JOIN
```

puede ser estructuralmente correcto aunque el target final no lo soporte.

---

# 29. ValidatedQueryArtifact

El resultado exitoso será conceptualmente:

```text
ValidatedQueryArtifact
```

---

# 30. Artifact structure

Podrá contener:

```text
ValidatedQueryArtifact
├── normalizedAst
├── validationSummary
├── structuralFingerprint
├── validationProfile
└── diagnostics metadata
```

---

# 31. No mutation flag

Evitar:

```php
$query->validated = true;
```

Preferir transición explícita:

```text
NormalizedQueryAst
        │
        ▼
QueryValidator
        │
        ▼
ValidatedQueryArtifact
```

---

# 32. Type-state model

Conceptualmente:

```text
QueryAst
   │
   ▼
NormalizedQueryAst
   │
   ▼
ValidatedQueryArtifact
   │
   ▼
SemanticQueryArtifact
   │
   ▼
OptimizedQueryArtifact
   │
   ▼
QueryPlan
```

---

# 33. Validation contract

Conceptualmente:

```php
interface QueryValidatorInterface
{
    public function validate(
        NormalizedQueryAst $query,
        QueryValidationContext $context,
    ): ValidatedQueryArtifact;
}
```

---

# 34. Alternative reporting API

Podrá existir internamente:

```php
interface QueryValidationEngineInterface
{
    public function inspect(
        NormalizedQueryAst $query,
        QueryValidationContext $context,
    ): QueryValidationReport;
}
```

---

# 35. Why two concepts

Permite:

```text
validate()
→ success artifact or exception
```

y:

```text
inspect()
→ complete diagnostic report
```

sin mezclar ambos workflows.

---

# 36. ValidationResult

Conceptualmente:

```text
QueryValidationReport
├── status
├── issues
├── warnings
├── statistics
└── diagnostics
```

---

# 37. ValidationStatus

```text
VALID
VALID_WITH_WARNINGS
INVALID
ABORTED
BUDGET_EXCEEDED
```

---

# 38. QueryValidationIssue

Cada problema será estructurado.

```text
QueryValidationIssue
├── code
├── severity
├── category
├── message
├── nodeId
├── nodeType
├── sourceLocation
├── path
├── ruleId
└── safeContext
```

---

# 39. Severity

```text
INFO
WARNING
ERROR
FATAL
```

---

# 40. Categories

```text
STRUCTURAL
EXPRESSION
PREDICATE
PARAMETER
TYPE_HINT
SOURCE
JOIN
CLAUSE
PROJECTION
GROUPING
ORDERING
PAGINATION
LOCKING
CTE
SET_OPERATION
INSERT
UPDATE
DELETE
METADATA
RAW
COMPLEXITY
EXTENSION
INVARIANT
```

---

# 41. Stable error codes

No depender únicamente de mensajes humanos.

Ejemplos:

```text
DB_QUERY_EMPTY_PROJECTION
DB_QUERY_INVALID_LIMIT
DB_QUERY_EMPTY_AND
DB_QUERY_INSERT_ARITY_MISMATCH
DB_QUERY_DUPLICATE_ASSIGNMENT
DB_QUERY_AST_CYCLE
```

---

# 42. Why stable codes

Permiten:

- testing;
- IDE tooling;
- CLI;
- debug toolbar;
- translations;
- API consumers;
- automated diagnostics.

---

# 43. Validation rules

La unidad básica será:

```text
QueryValidationRule
```

---

# 44. Contract conceptual

```php
interface QueryValidationRuleInterface
{
    public function validate(
        QueryAstNode $node,
        QueryValidationRuleContext $context,
    ): void;
}
```

---

# 45. Diagnostic sink

En vez de devolver arrays en cada rule:

```text
Rule
   │
   ▼
ValidationDiagnosticSink
```

---

# 46. Alternative immutable result

También podrá usarse:

```php
ValidationRuleResult
```

si benchmarks y claridad lo favorecen.

---

# 47. Specialized rule dispatch

Evitar recorrer todas las reglas preguntando:

```php
$supports = $rule->supports($node);
```

en cada node si el costo resulta significativo.

Preferir registry compilado:

```text
NodeType
   │
   ▼
Precomputed Validation Rules
```

---

# 48. ValidationRuleRegistry

Será:

- specialized;
- deterministic;
- frozen;
- bootstrap-built;
- extension-aware.

---

# 49. No universal registry

No deberá convertirse en:

```text
DatabaseEverythingRegistry
```

---

# 50. Rule phases

Conceptualmente:

```text
ROOT
STRUCTURE
NODE
EXPRESSION
PREDICATE
PARAMETER
CLAUSE
METADATA
COMPLEXITY
EXTENSION
FINAL
```

---

# 51. Rule ordering

Se podrá declarar:

```text
phase
priority
before
after
```

---

# 52. Deterministic ordering

El mismo:

```text
Normalized AST
+
Validation Profile
+
Frozen Rule Graph
```

deberá producir el mismo resultado.

---

# 53. ValidationProfile

No todas las operaciones requieren exactamente la misma política.

Conceptualmente:

```text
ValidationProfile
```

---

# 54. Profiles iniciales

```text
STANDARD
STRICT
DEVELOPMENT
MIGRATION
INTERNAL
```

---

# 55. STANDARD

Default para aplicación.

Incluye:

- structural invariants;
- safe query shape;
- complexity limits;
- extension validation;
- mutation policy configurable.

---

# 56. STRICT

Añade:

- expensive invariant checks;
- normalized-form verification;
- extension conformance;
- additional diagnostics.

---

# 57. DEVELOPMENT

Puede:

- acumular más errores;
- generar source paths;
- validar idempotence assumptions;
- producir suggestions;
- habilitar expensive diagnostics.

---

# 58. MIGRATION

No significa que Query AST sea el sistema principal de DDL.

Se usaría sólo para queries auxiliares de Migration/Schema que pasen por Query Engine.

---

# 59. INTERNAL

Para queries generadas por subsistemas oficiales.

No deberá significar:

```text
skip security
```

---

# 60. Important invariant

> Ningún perfil interno deberá permitir producir un AST corrupto.

---

# 61. Fail-fast mode

Producción podrá utilizar:

```text
FAIL_FAST
```

para detenerse ante el primer error fatal.

---

# 62. Collect mode

Desarrollo podrá utilizar:

```text
COLLECT
```

para reportar múltiples problemas.

---

# 63. But bounded

Nunca acumular errores ilimitadamente.

```text
maxValidationIssues
```

---

# 64. Root validation

Cada Query AST deberá tener un root válido.

Tipos iniciales:

```text
SelectQueryNode
InsertQueryNode
UpdateQueryNode
DeleteQueryNode
RawQueryNode
ExtensionQueryNode
```

---

# 65. Unknown root

Un node desconocido sin extension handler:

```text
UnsupportedQueryRootException
```

---

# 66. AST acyclicity

El Query AST deberá ser acíclico.

---

# 67. Cycle example

Inválido:

```text
Node A
  └── Node B
        └── Node A
```

---

# 68. Cycle detection

Podrá realizarse:

- durante AST construction;
- durante normalization;
- durante validation como defensa adicional.

---

# 69. AST depth

Se validará:

```text
maxAstDepth
```

---

# 70. AST node count

Se validará:

```text
maxAstNodes
```

---

# 71. Query complexity limits

Podrán existir límites separados:

```text
maxExpressions
maxPredicates
maxParameters
maxJoins
maxSubqueries
maxCtes
maxSetOperations
maxProjectionItems
maxOrderItems
maxGroupItems
maxInsertRows
maxAssignments
maxFunctionArguments
maxCaseBranches
maxTupleSize
```

---

# 72. Complexity ≠ cost

Estos límites representan:

```text
framework processing complexity
```

no costo real del servidor DB.

---

# 73. No query optimizer estimation

El Validation System no estimará:

```text
rows scanned
index cost
join cardinality
server execution cost
```

---

# 74. Query root consistency

Cada root deberá contener únicamente clauses permitidas para su operación.

---

# 75. Example

Inválido:

```text
InsertQueryNode
└── LockClause
```

si el Query Model no define locking para INSERT.

---

# 76. Expression validation

Se integrará con:

```text
27_DATABASE_QUERY_EXPRESSION_SYSTEM.md
```

---

# 77. Expression structural checks

Ejemplos:

- node kind válido;
- número correcto de children;
- argumentos presentes;
- no null children;
- operator válido;
- function reference estructurada;
- CASE válido;
- CAST target presente;
- WindowSpecification estructuralmente válida.

---

# 78. Arithmetic expression

Ejemplo:

```text
ArithmeticExpression
├── operator = ADD
├── left = Expression
└── right = Expression
```

No:

```text
left = null
```

---

# 79. Type compatibility not yet

Esto:

```text
DATE + JSON
```

puede ser estructuralmente válido.

La incompatibilidad pertenece a Query Type/Semantic Analysis.

---

# 80. Unary expression

Debe tener exactamente:

```text
one operand
```

---

# 81. Function expression

Podrá validar:

```text
function identifier shape
argument collection structure
modifier structure
```

pero no necesariamente:

```text
function exists
overload matches
return type
```

---

# 82. CASE validation

Para searched CASE:

```text
WHEN
```

debe tener:

```text
predicate + result expression
```

---

# 83. CASE branch count

Debe ser:

```text
>= 1
```

salvo que el Query Model defina otra cosa.

---

# 84. Simple CASE

Debe contener:

```text
operand
+
comparison values
+
result expressions
```

---

# 85. Mixing CASE forms

No permitir representación ambigua donde un mismo node sea simultáneamente:

```text
simple CASE
```

y:

```text
searched CASE
```

---

# 86. Cast validation

Debe contener:

```text
expression
targetTypeReference
```

---

# 87. Window expression validation

Podrá validar:

- WindowSpecification presente;
- frame structure;
- frame bounds structurally valid;
- partition expressions valid;
- ordering nodes valid.

No deberá validar todavía si la función concreta puede ser window function.

---

# 88. Subquery expression

Debe contener un query root válido.

---

# 89. Tuple expression

Podrá exigir:

```text
arity >= 1
```

o la invariant definida por el Query Model.

---

# 90. Predicate validation

Se integra con:

```text
28_DATABASE_QUERY_PREDICATE_SYSTEM.md
```

---

# 91. Predicate categories

```text
Comparison
Logical AND
Logical OR
Logical NOT
NULL
BETWEEN
IN
EXISTS
LIKE
Pattern
JSON
Extension
```

---

# 92. Comparison validation

Debe contener:

```text
left Expression
operator ComparisonOperator
right Expression
```

---

# 93. Comparison type compatibility

No pertenece todavía a structural validation.

---

# 94. AND validation

Después de normalization, un:

```text
AndPredicate
```

deberá cumplir la arity canónica definida.

Preferencia:

```text
>= 2 children
```

porque:

```text
AND(A)
```

deberá haberse normalizado a:

```text
A
```

---

# 95. OR validation

Igualmente:

```text
>= 2 children
```

---

# 96. NOT validation

Exactamente:

```text
1 predicate child
```

---

# 97. No nested canonical AND

Si Normalization garantiza flattening:

```text
AND(
    A,
    AND(B, C)
)
```

debe ser rechazado por normalized validation como invariant violation.

---

# 98. No nested canonical OR

Misma regla.

---

# 99. IS NULL

Debe contener una expresión.

---

# 100. BETWEEN

Debe contener:

```text
subject
lowerBound
upperBound
negated flag
```

---

# 101. BETWEEN bound type compatibility

Posterior.

---

# 102. IN validation

Debe contener:

```text
subject
+
valid IN source
```

---

# 103. IN source

Puede ser:

```text
ValueList
ParameterList
Subquery
ExtensionInSource
```

---

# 104. Empty IN

Si Normalization definió:

```text
x IN []
→ FalsePredicate
```

entonces un `InPredicate` con lista vacía después de normalization será invariant violation.

---

# 105. EXISTS

Debe contener:

```text
subquery
```

---

# 106. LIKE

Debe contener:

```text
subject expression
pattern expression
optional escape expression
semantic options
```

---

# 107. LIKE pattern type

Que el pattern sea string-compatible pertenece a Semantic Analysis.

---

# 108. JSON predicate

Validará estructura del path/arguments/options, no soporte del target Platform.

---

# 109. Parameter validation

Se integra con:

```text
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

---

# 110. Parameter structural rules

Cada parameter deberá poseer:

```text
QueryParameterId
ParameterShape
optional TypeHint
metadata
```

según su clase.

---

# 111. Parameter ID uniqueness

Dentro del scope apropiado, IDs no deberán colisionar accidentalmente.

---

# 112. Shared parameter identity

La misma identidad puede aparecer múltiples veces si representa intencionalmente el mismo semantic parameter.

---

# 113. Collision

Inválido:

```text
ParameterId P1
→ two incompatible descriptors
```

---

# 114. Parameter descriptor consistency

Si `P1` aparece como:

```text
SCALAR INTEGER
```

y posteriormente como:

```text
LIST UUID
```

deberá rechazarse.

---

# 115. Runtime values

No deberán ser inspeccionados innecesariamente durante structural validation.

---

# 116. Binding presence

La existencia de un runtime binding puede validarse en una fase contextual posterior antes de compilation/execution.

---

# 117. Missing binding

Debe distinguirse:

```text
structurally valid parameter
```

de:

```text
missing runtime value
```

---

# 118. Parameter values and security

Los valores no deberán aparecer en errores por defecto.

---

# 119. Parameter list size

Podrá validarse contra:

```text
maxParameterListSize
```

como resource-governance limit.

---

# 120. Driver parameter limit

No pertenece todavía al structural validator.

Ejemplo:

```text
SQLite max variables
```

es target capability/runtime information.

---

# 121. Query Type validation

Se integra con:

```text
30_DATABASE_QUERY_TYPE_SYSTEM.md
```

---

# 122. Type reference structural validation

Puede verificar:

```text
valid QueryTypeId
valid type parameters
valid nullability declaration
valid generic structure
```

---

# 123. Type inference

No se realizará aquí.

---

# 124. Unknown type

Un:

```text
UnknownQueryType
```

puede ser perfectamente válido en esta fase.

---

# 125. Invalid type reference

Ejemplo:

```text
DecimalType(
    precision = -10,
    scale = 50
)
```

puede rechazarse localmente.

---

# 126. Metadata validation

Se integra con:

```text
31_DATABASE_QUERY_METADATA_SYSTEM.md
```

---

# 127. Metadata keys

Deberán utilizar:

```text
typed metadata key
```

o namespace válido.

---

# 128. Metadata collisions

Dos providers no deberán registrar el mismo metadata key incompatible.

Esto deberá detectarse preferentemente durante bootstrap.

---

# 129. Metadata value validation

Cada descriptor podrá aportar:

```text
MetadataValueValidator
```

---

# 130. Metadata categories

Podrán existir:

```text
SEMANTIC
EXECUTION
DIAGNOSTIC
OBSERVATIONAL
EXTENSION
```

---

# 131. Semantic metadata

Errores pueden invalidar la query.

---

# 132. Diagnostic metadata

Errores no siempre deberán invalidar la query, dependiendo del descriptor.

---

# 133. Unknown metadata

Policy configurable:

```text
REJECT
WARN
ALLOW_NAMESPACED_EXTENSION
```

---

# 134. Query Context validation

Se integra con:

```text
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
```

---

# 135. Context availability

El Validator deberá recibir sólo el contexto necesario.

No deberá resolverlo globalmente.

---

# 136. QueryValidationContext

Conceptualmente:

```php
final readonly class QueryValidationContext
{
    public function __construct(
        public ValidationProfile $profile,
        public QueryValidationPolicy $policy,
        public QueryValidationBudget $budget,
        public ExtensionValidationContext $extensions,
        public ?DiagnosticSink $diagnostics = null,
    ) {}
}
```

---

# 137. No Connection in context

No deberá incluir por defecto:

```text
PDO
NativeConnection
ConnectionLease
```

---

# 138. No EntityManager

Tampoco:

```text
EntityManager
UnitOfWork
IdentityMap
```

---

# 139. QueryValidationPolicy

Separará reglas configurables de invariantes absolutas.

---

# 140. Absolute invariant

Ejemplo:

```text
AST must be acyclic
```

no puede desactivarse.

---

# 141. Policy rule

Ejemplo:

```text
allowRawExpressions
```

puede ser configurable.

---

# 142. Critical distinction

```text
Invariant
≠
Policy
```

---

# 143. Invariants cannot be disabled

Ni siquiera mediante:

```text
strict = false
```

---

# 144. Policy examples

```text
allowRawQueries
allowRawExpressions
requireExplicitFullTableUpdate
requireExplicitFullTableDelete
allowVendorSpecificExtensions
maxJoins
maxSubqueries
maxParameters
```

---

# 145. SELECT validation

Un `SelectQueryNode` podrá contener:

```text
WITH
projection
FROM
joins
predicate
grouping
having
windows
set operations
ordering
pagination
locking
metadata
```

según el Query Model.

---

# 146. Projection

La projection deberá cumplir la invariant definida.

Si VoltStack permite:

```text
select()
```

como shorthand para `*`, esa conversión deberá ocurrir antes.

Después de normalization deberá existir:

```text
WildcardProjection
```

o projection explícita.

---

# 147. Empty projection

Por tanto:

```text
Projection([])
```

será inválida.

---

# 148. Projection item

Debe ser:

```text
ColumnProjection
ExpressionProjection
AggregateProjection
WildcardProjection
SubqueryProjection
ExtensionProjection
```

---

# 149. Projection aliases

Podrán validarse estructuralmente:

- alias no vacío;
- Value Object válido;
- no raw quoting.

Duplicidad puede requerir policy/semantic analysis según contexto.

---

# 150. FROM

No toda SELECT necesita necesariamente `FROM`.

Ejemplo conceptual:

```text
SELECT 1
```

Por tanto, ausencia de FROM no será universalmente inválida.

---

# 151. JOIN without source

Inválido.

---

# 152. JOIN condition

Dependiendo del tipo:

```text
INNER
LEFT
RIGHT
FULL
```

normalmente requerirán condition o estructura equivalente.

---

# 153. CROSS JOIN

No deberá requerir predicate si su node model no lo define.

---

# 154. JOIN structural matrix

| Join | Source | Predicate |
|---|---:|---:|
| INNER | Required | Required |
| LEFT | Required | Required |
| RIGHT | Required | Required |
| FULL | Required | Required |
| CROSS | Required | Forbidden/Absent |

Esto describe el core portable inicial; extensiones podrán definir otros tipos.

---

# 155. JOIN capability support

No se valida aquí.

---

# 156. WHERE

Si existe, deberá contener exactamente un root PredicateNode.

---

# 157. GROUP BY

Cada item debe ser una expresión válida.

---

# 158. Empty GROUP BY

Deberá haberse eliminado durante normalization o rechazarse.

---

# 159. HAVING

Debe ser PredicateNode.

---

# 160. HAVING without GROUP BY

No será necesariamente invalidado estructuralmente.

Su semántica puede depender de aggregate query rules.

Eso pertenece a Semantic Analysis.

---

# 161. ORDER BY

Cada item:

```text
OrderByItem
├── expression
├── direction
└── nullOrdering?
```

---

# 162. Direction

Sólo valores del enum:

```text
ASC
DESC
```

---

# 163. Null ordering

Sólo valores semánticos:

```text
FIRST
LAST
UNSPECIFIED
```

No syntax strings.

---

# 164. Pagination

Validará:

```text
limit >= 0
offset >= 0
```

según el modelo elegido.

---

# 165. LIMIT zero

```text
LIMIT 0
```

deberá ser válido.

---

# 166. Offset without limit

Puede ser estructuralmente válido.

El target Platform decidirá soporte/strategy.

---

# 167. Lock clause

Validará estructura:

```text
mode
targets
waitPolicy
```

---

# 168. Lock capability

Posterior.

---

# 169. Lock target resolution

Posterior.

---

# 170. SELECT set operations

Cada branch deberá ser query válida.

---

# 171. Projection compatibility

Esto:

```text
SELECT a
UNION
SELECT b, c
```

requiere semantic validation de arity.

Podría detectarse estructuralmente si las projections son explícitas.

---

# 172. Conservative rule

Si la arity puede conocerse sin schema resolution, podrá rechazarse temprano.

Si interviene:

```text
*
```

deberá diferirse.

---

# 173. Principle

> Validar temprano sólo cuando la conclusión sea inequívoca.

---

# 174. INSERT validation

Canonical structure:

```text
InsertQuery
├── target
├── columns
├── source
├── conflict
├── returning
└── metadata
```

---

# 175. Target

Debe existir estructuralmente como TableTarget/valid target node.

No significa que la tabla exista realmente.

---

# 176. Insert source exclusivity

Exactamente una fuente:

```text
ValuesInsertSource
QueryInsertSource
DefaultValuesInsertSource
```

---

# 177. Multiple insert source forms

Inválido:

```text
VALUES
+
SELECT source
```

simultáneamente.

---

# 178. Insert columns

Si existen explícitamente:

```text
count(columns) > 0
```

salvo que el modelo soporte `DEFAULT VALUES` sin columns.

---

# 179. Duplicate insert columns

Si los identifiers son textualmente/canónicamente idénticos:

```text
INSERT users (name, name)
```

deberá rechazarse temprano.

---

# 180. Case-sensitive ambiguity

Si la igualdad depende del Platform:

```text
Name
name
```

la resolución definitiva puede diferirse.

---

# 181. Values rows

Debe existir al menos una row para `ValuesInsertSource`.

---

# 182. Row arity

Todas las rows deberán tener la misma arity.

---

# 183. Column/row arity

Si columns explícitas:

```text
count(columns)
=
count(values in every row)
```

---

# 184. Insert-from-query

La compatibilidad entre target columns y source projection pertenece a Semantic Analysis cuando no pueda determinarse estructuralmente.

---

# 185. Default values

`DefaultValuesInsertSource` no deberá contener rows.

---

# 186. Conflict strategy

Debe poseer una estructura válida.

---

# 187. Conflict target

Puede ser:

```text
columns
constraint reference
expression target
none
```

según la semántica soportada por Query Model.

---

# 188. Conflict target existence

Se resuelve posteriormente.

---

# 189. Conflict action

Ejemplos:

```text
DO_NOTHING
UPDATE(assignments)
```

---

# 190. Empty conflict update

No deberá representarse como:

```text
UPDATE([])
```

salvo que la especificación defina una semántica explícita.

---

# 191. RETURNING

Projection-like structure validable.

Capability posterior.

---

# 192. UPDATE validation

Canonical:

```text
UpdateQuery
├── target
├── assignments
├── sources
├── joins
├── predicate
├── ordering
├── pagination
├── returning
└── metadata
```

---

# 193. Assignment count

Debe existir:

```text
>= 1
```

---

# 194. Assignment structure

```text
Assignment
├── target
└── valueExpression
```

---

# 195. Duplicate assignment

Si se puede determinar inequívocamente:

```text
SET name = ?, name = ?
```

deberá rechazarse.

---

# 196. Assignment target

Debe ser un assignable reference node.

No cualquier expression.

---

# 197. Example invalid

```text
SET (price + tax) = 100
```

si el Query Model sólo permite column targets.

---

# 198. Full-table update

```text
UPDATE users
SET active = false
```

es estructuralmente válido.

---

# 199. Mutation safety policy

Puede exigir:

```text
FullTableMutationIntent::EXPLICIT
```

si no existe predicate.

---

# 200. This is policy

No structural invariant universal.

---

# 201. UPDATE ordering/limit

Puede representarse semánticamente aunque no todos los Platforms lo soporten.

---

# 202. DELETE validation

Canonical:

```text
DeleteQuery
├── target
├── sources
├── joins
├── predicate
├── ordering
├── pagination
├── returning
└── metadata
```

---

# 203. Full-table delete

Misma separación:

```text
structurally valid
```

pero posiblemente rechazado por:

```text
MutationSafetyPolicy
```

---

# 204. DELETE target

Debe ser un target estructural válido.

---

# 205. CTE validation

Cada CTE:

```text
CteDefinition
├── name
├── columns?
├── query
├── recursive metadata
└── materialization intent?
```

---

# 206. CTE name

Debe ser un Identifier válido.

---

# 207. Duplicate CTE names

Si son inequívocamente iguales en canonical representation, deberán rechazarse.

Case folding dependiente de Platform podrá diferirse.

---

# 208. CTE query

Debe ser un query root permitido.

---

# 209. CTE declared columns

No deberán contener duplicados inequívocos.

---

# 210. CTE arity

Si source projection arity es conocida estructuralmente, podrá compararse con declared columns.

Si requiere wildcard/schema resolution, se difiere.

---

# 211. Recursive semantics

No se validarán completamente hasta Symbol Resolution/Semantic Analysis.

---

# 212. CTE dependency cycles

Un ciclo de nombres puede ser válido en recursive CTEs bajo ciertas reglas.

Por tanto:

```text
AST object cycle
```

es siempre inválido.

Pero:

```text
semantic CTE reference cycle
```

no es automáticamente inválido.

---

# 213. Important distinction

```text
Object Graph Cycle
≠
Semantic Recursive Reference
```

---

# 214. Set operation validation

Cada operation tendrá:

```text
type
quantifier
left
right
```

---

# 215. Types

```text
UNION
INTERSECT
EXCEPT
```

---

# 216. Quantifiers

```text
DISTINCT
ALL
```

cuando sean aplicables.

---

# 217. Empty branch

Inválido.

---

# 218. Nested set grouping

Debe permanecer explícito.

---

# 219. Set operation arity

Validar temprano sólo cuando sea inequívoca.

---

# 220. Ordering after set operation

La estructura deberá respetar el Query Model canónico.

No se permitirá que ambiguity del Builder llegue al AST.

---

# 221. Window validation

`WindowSpecification` podrá incluir:

```text
reference
partitionBy
orderBy
frame
```

---

# 222. Window reference

Identifier estructuralmente válido.

Existencia/resolution posterior.

---

# 223. Window frame

Conceptualmente:

```text
WindowFrame
├── unit
├── start
├── end?
└── exclusion?
```

---

# 224. Frame units

```text
ROWS
RANGE
GROUPS
```

como semantic enums.

---

# 225. Frame bounds

```text
UNBOUNDED_PRECEDING
PRECEDING
CURRENT_ROW
FOLLOWING
UNBOUNDED_FOLLOWING
```

---

# 226. Impossible frame

Algunas combinaciones pueden rechazarse estructuralmente.

Ejemplo conceptual:

```text
BETWEEN
UNBOUNDED_FOLLOWING
AND
CURRENT_ROW
```

si la semántica abstracta lo hace siempre inválido.

---

# 227. Platform-specific frame restrictions

Posteriores.

---

# 228. Raw query validation

`RawSqlQuery` será escape hatch explícito.

---

# 229. Raw query requirements

Debe contener:

```text
non-empty SQL text
parameter descriptors
query classification
trust metadata
optional target metadata
```

---

# 230. No SQL parsing

El validator base no intentará validar gramática SQL raw.

---

# 231. Raw SQL classification

```text
READ
WRITE
DDL
ADMIN
UNKNOWN
```

---

# 232. UNKNOWN

Deberá tratarse conservadoramente.

Por ejemplo:

```text
no replica routing assumption
no automatic retry assumption
```

---

# 233. Raw query policy

Podrá:

```text
ALLOW
WARN
REJECT
```

según contexto.

---

# 234. Raw expression validation

Igualmente podrá exigir:

```text
trusted origin
bindings separated
portability metadata
optional type hint
```

---

# 235. Raw identifiers

No deberán aceptar user input interpolado silenciosamente.

---

# 236. Validation cannot prove raw SQL safety

Por ello:

> Raw SQL es explícitamente una reducción de las garantías estructurales del Query Engine.

---

# 237. Identifier validation

Podrá comprobar:

- no vacío;
- segment structure;
- no illegal null bytes;
- no pre-quoted representation cuando no corresponde;
- max framework-side structural length si existe;
- valid alias structure.

---

# 238. Identifier server length

Ejemplo:

```text
PostgreSQL identifier length
```

pertenece al Platform.

---

# 239. Identifier quoting

No se realizará.

---

# 240. Identifier semantic equality

Puede diferir por Platform.

No asumir globalmente case-insensitive.

---

# 241. Alias validation

Aliases deben ser Value Objects estructurados.

---

# 242. Alias collision

Algunas colisiones pueden detectarse estructuralmente.

Otras requieren Symbol Resolution.

---

# 243. Source validation

Cada source tendrá un node válido.

---

# 244. TableSource

Debe contener:

```text
QualifiedIdentifier
optional Alias
```

---

# 245. SubquerySource

Debe contener:

```text
QueryRoot
Alias
```

si el Query Model exige alias para derived tables.

---

# 246. CteSource

Debe contener CTE reference estructurada.

Existencia posterior.

---

# 247. FunctionSource

Debe contener function reference y arguments estructurados.

---

# 248. ExtensionSource

Requiere extension validator registrado.

---

# 249. Duplicate sources

No se rechazará simplemente porque la misma tabla aparezca dos veces.

Ejemplo:

```text
users u1
JOIN users u2
```

es válido.

---

# 250. Alias ambiguity

Será responsabilidad de Symbol Resolution cuando no pueda demostrarse localmente.

---

# 251. Validation budget

El sistema tendrá:

```text
QueryValidationBudget
```

---

# 252. Budget fields

Conceptualmente:

```text
maxNodesVisited
maxDepth
maxIssues
maxRulesApplied
maxExtensionChecks
maxParameters
maxSubqueries
maxJoins
maxCtes
maxSetOperations
maxCollectionItems
```

---

# 253. Deadline

Podrá incluir deadline opcional.

---

# 254. Budget exhaustion

No deberá causar:

- OOM;
- stack overflow;
- worker corruption;
- endless traversal.

---

# 255. Error

```text
QueryValidationBudgetExceededException
```

o report status:

```text
BUDGET_EXCEEDED
```

---

# 256. Iterative traversal

Para árboles muy profundos, podrá preferirse traversal iterativo sobre recursion PHP.

---

# 257. Why

Protege persistent workers de:

```text
deeply nested generated queries
```

---

# 258. Resource governance

Validation constituye una frontera temprana contra queries estructuralmente patológicas.

---

# 259. But not rate limiting

No reemplaza:

```text
HTTP rate limiting
DoS protection
authorization
request size limits
```

---

# 260. Extension validation

Las extensiones podrán introducir:

```text
ExtensionQueryNode
ExtensionExpressionNode
ExtensionPredicateNode
ExtensionSourceNode
ExtensionClauseNode
ExtensionMetadata
```

---

# 261. Extension validator

Cada extension node deberá registrar:

```text
ExtensionValidationHandler
```

cuando no pueda validarse con reglas genéricas.

---

# 262. Completeness requirement

Una extensión que introduce un node no podrá considerarse completa sólo porque el AST pueda construirlo.

Puede requerir:

```text
AST descriptor
Normalizer
Validator
Semantic handler
Type rules
Capability requirements
Planner strategy
Compiler
Fingerprint handler
Diagnostics
```

según su función.

---

# 263. Missing validator

Si el node requiere validación específica y no existe handler:

```text
IncompleteQueryExtensionException
```

---

# 264. Extension validation isolation

Una extensión deberá validar principalmente sus propios nodes.

---

# 265. Core interception

Modificar reglas core requerirá un extension point explícito.

---

# 266. Extension registration

Ocurrirá durante bootstrap.

---

# 267. Registry freeze

Antes de runtime:

```text
ValidationRuleRegistry
ExtensionValidationRegistry
```

deberán quedar frozen.

---

# 268. No runtime registration

No se permitirá por defecto:

```text
register validator during request
```

---

# 269. Extension rule conflict

No habrá:

```text
last registered wins
```

---

# 270. Extension determinism

Mismo extension graph deberá producir mismo validation behavior.

---

# 271. Validation and fingerprints

La validación no deberá cambiar el AST.

---

# 272. Therefore

Normalmente:

```text
NormalizedQueryFingerprint
```

deberá permanecer igual antes y después de Validation.

---

# 273. Validation fingerprint

Puede existir:

```text
ValidationProfileFingerprint
```

para caches que dependan de policy/profile.

---

# 274. Example

```text
ValidationArtifactKey
=
NormalizedQueryFingerprint
+
ValidationProfileFingerprint
+
ValidationRuleSetFingerprint
```

---

# 275. Caching validation results

Puede ser posible para queries completamente estructurales.

Pero deberá hacerse con cautela.

---

# 276. Do not cache request-specific policy

Si ValidationContext contiene policy scoped:

```text
cache key
```

debe reflejarla o el resultado no debe cachearse.

---

# 277. Prefer immutable reusable validation rules

Rules/descriptors pueden ser app-scoped.

Results normalmente operation/query-scoped.

---

# 278. ValidationRuleSetFingerprint

Podrá incluir:

```text
core validation version
extension validation versions
profile version
```

---

# 279. Source mapping

Validation deberá aprovechar:

```text
NormalizationSourceMap
```

del documento 33 cuando esté disponible.

---

# 280. Query path

Cada issue podrá incluir un path como:

```text
query.where.and[1].comparison.right
```

---

# 281. Why path

Permite localizar errores incluso cuando no existe source line.

---

# 282. Builder source information

Si Developer API conserva:

```text
file
line
builder call origin
```

podrá asociarse al AST mediante provenance metadata.

---

# 283. No source info in fingerprint

Normalmente no participará en query shape.

---

# 284. Error messages

Deben responder:

1. qué está mal;
2. dónde;
3. qué invariant se violó;
4. cómo corregirlo cuando sea razonable.

---

# 285. Example

En vez de:

```text
Invalid query
```

preferir:

```text
Insert row 3 contains 2 values, but the INSERT target declares 3 columns.
```

---

# 286. Error suggestions

Podrán existir:

```text
QueryValidationSuggestion
```

pero no deberán modificar automáticamente la query.

---

# 287. Validation is observational

El validator deberá:

```text
inspect
```

no:

```text
repair
```

---

# 288. Repair belongs elsewhere

Si una representación puede canonicalizarse de forma segura, debe hacerlo Normalization antes de Validation.

---

# 289. Important rule

> El Validator no será un segundo Normalizer.

---

# 290. Mutation safety

Se introducirá conceptualmente:

```text
QueryMutationSafetyPolicy
```

---

# 291. Full-table UPDATE policy

Opciones:

```text
ALLOW
WARN
REQUIRE_EXPLICIT_INTENT
REJECT
```

---

# 292. Full-table DELETE policy

Igualmente.

---

# 293. Default proposal

Para API pública:

```text
UPDATE without predicate
→ REQUIRE_EXPLICIT_INTENT

DELETE without predicate
→ REQUIRE_EXPLICIT_INTENT
```

en perfiles seguros.

---

# 294. Explicit intent

Ejemplo conceptual:

```php
DB::table('users')
    ->allowFullTableMutation()
    ->update(['active' => false]);
```

La API final podrá variar.

---

# 295. Why explicit intent

Reduce errores accidentales sin declarar estructuralmente inválida una operación SQL legítima.

---

# 296. ORM bulk mutation

ORM/Repository deberán respetar la misma policy si utilizan Query Engine.

---

# 297. Internal framework query

No deberá saltarse la policy silenciosamente.

Podrá usar:

```text
explicit trusted intent metadata
```

cuando corresponda.

---

# 298. Raw mutation

Deberá clasificarse conservadoramente.

---

# 299. Read/write classification

El Validator puede verificar que una classification declarada sea estructuralmente compatible con QueryRoot.

Ejemplo:

```text
SelectQuery
declared WRITE
```

puede ser inconsistente.

---

# 300. But locking SELECT

Un SELECT con lock puede requerir primary/transaction sin convertirse en mutation.

Por tanto:

```text
QueryOperationKind
```

y:

```text
ConnectionIntent
```

son conceptos distintos.

---

# 301. Transaction requirements

Validation podrá verificar consistencia estructural de:

```text
TransactionRequirement
```

metadata.

No deberá iniciar transaction.

---

# 302. Example conflict

```text
TransactionRequirement = FORBIDDEN
+
LockClause requiring transaction semantics
```

puede detectarse en una fase posterior cuando esas semantics sean conocidas.

No asumir demasiado pronto.

---

# 303. Consistency requirements

Igualmente:

```text
ConsistencyRequirement
```

deberá tener estructura válida.

La topología real se resuelve después.

---

# 304. Result requirements

Podrá validar estructura:

```text
BUFFERED
STREAMING
SINGLE_ROW
SCALAR
AFFECTED_ROWS
GENERATED_VALUES
```

según Query API.

---

# 305. Compatibility

La compatibilidad final entre query/result strategy puede validarse en Planner/Execution.

---

# 306. Validation and Query Metadata

Metadata puede declarar:

```text
query origin
connection preference
consistency
cache intent
execution hints
diagnostic labels
security annotations
```

El Validator verificará su estructura y combinaciones universalmente inválidas.

---

# 307. Hints

Hints no deberán utilizar strings arbitrarios core.

Preferir:

```text
QueryHintId
```

y extension namespaces.

---

# 308. Vendor hints

Deben ser explícitamente:

```text
PLATFORM_SPECIFIC
```

o:

```text
DIALECT_SPECIFIC
```

---

# 309. Validation can enforce portability policy

Ejemplo:

```text
PortableOnlyPolicy
```

puede rechazar nodes marcados:

```text
DIALECT_SPECIFIC
RAW
```

sin conocer todavía el target DB.

---

# 310. Portability policy

Opciones:

```text
ALLOW_ALL
PORTABLE_PREFERRED
PORTABLE_ONLY
```

---

# 311. This differs from capability support

Una query puede ser:

```text
platform-specific
```

y perfectamente soportada.

Portability es policy; capability es technical support.

---

# 312. Security validation

El sistema deberá reforzar:

```text
values → parameters
identifiers → structured identifiers
raw SQL → explicit escape hatch
```

---

# 313. User values

No deberán aparecer como raw SQL fragments a través de la API estructurada.

---

# 314. Dynamic identifiers

Deben pasar por Identifier abstraction.

---

# 315. Raw fragment trust

Podrá existir:

```text
RawTrustLevel
```

---

# 316. Example levels

```text
FRAMEWORK_INTERNAL
APPLICATION_TRUSTED
EXTENSION_TRUSTED
UNTRUSTED
```

---

# 317. UNTRUSTED raw SQL

Default:

```text
REJECT
```

---

# 318. Important

Esto no convierte al validator en SQL injection scanner.

La seguridad principal proviene de no permitir que datos no confiables entren como SQL syntax.

---

# 319. Sensitive metadata

El validator deberá evitar copiar secrets a diagnostics.

---

# 320. Persistent runtime model

El Query Validation System debe ser seguro para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 321. Shared services

Podrán vivir application/worker scope:

```text
QueryValidator
FrozenValidationRuleRegistry
ValidationRuleDescriptors
ValidationProfileDescriptors
```

si son:

```text
immutable
stateless
reentrant
concurrency-safe
```

---

# 322. Operation-local state

Debe permanecer scoped:

```text
ValidationState
IssueCollector
TraversalStack
VisitedNodes
ValidationPath
DiagnosticContext
BudgetTracker
```

---

# 323. No global current query

Prohibido:

```php
QueryValidator::$currentQuery;
```

---

# 324. No global current node

Prohibido:

```php
QueryValidator::$currentNode;
```

---

# 325. No static issue collector

Prohibido.

---

# 326. FrankenPHP example

```text
Worker
│
├── Frozen QueryValidator
│
├── Request A
│   └── ValidationState A
│
├── Request B
│   └── ValidationState B
│
└── Request C
    └── ValidationState C
```

---

# 327. OpenSwoole example

```text
Coroutine A ── ValidationState A
Coroutine B ── ValidationState B
Coroutine C ── ValidationState C
```

Todos pueden compartir:

```text
Frozen QueryValidator
```

pero nunca mutable operation state.

---

# 328. ValidationState

Conceptualmente:

```text
ValidationState
├── visitedNodes
├── traversalStack
├── issueCount
├── ruleApplications
├── budgetUsage
├── currentPath
└── diagnostics
```

---

# 329. Lifecycle

```text
CREATED
   │
   ▼
VALIDATING
   │
   ├── VALID
   ├── INVALID
   ├── BUDGET_EXCEEDED
   └── FAILED
   │
   ▼
CLOSED
```

---

# 330. State not reusable

Una instancia de `ValidationState` no deberá reutilizarse para otra query.

---

# 331. Concurrency

Rules compartidas deberán ser stateless.

Si una rule necesita mutable state:

```text
operation-local helper
```

deberá crearse explícitamente.

---

# 332. Performance

Validation estará en el hot path.

---

# 333. Performance priorities

```text
Correctness
    >
Bounded Complexity
    >
Determinism
    >
Low Allocation
    >
Minimal Traversal
```

---

# 334. Single traversal opportunities

Muchas reglas podrán ejecutarse durante un único traversal.

---

# 335. Specialized validators

Puede existir:

```text
SelectQueryValidator
InsertQueryValidator
UpdateQueryValidator
DeleteQueryValidator
ExpressionValidator
PredicateValidator
ParameterValidator
MetadataValidator
```

coordinados por un engine.

---

# 336. Avoid God Validator

No:

```text
QueryValidator.php
30,000 lines
```

---

# 337. Coordinator

`QueryValidator` deberá orquestar componentes especializados.

---

# 338. Precompiled dispatch

En producción podrá compilarse:

```text
NodeKind
→ validator chain
```

---

# 339. No reflection hot path

Extension discovery/reflection deberá ocurrir durante bootstrap.

---

# 340. Structural sharing

Como Validation no modifica AST, no necesita copiar nodes.

---

# 341. No defensive clone

No clonar AST completo sólo para validarlo.

---

# 342. Validation result memory

En producción podrá almacenar únicamente:

```text
status
profile fingerprint
minimal summary
```

cuando no se necesiten diagnostics.

---

# 343. Development report

Puede incluir:

```text
issues
warnings
paths
source locations
rule IDs
statistics
```

---

# 344. Validation statistics

Conceptualmente:

```text
nodesVisited
rulesApplied
maxDepthObserved
parameterCount
joinCount
subqueryCount
cteCount
issueCount
```

---

# 345. Statistics not semantics

No confundir con query execution statistics.

---

# 346. Telemetry

Métricas posibles:

```text
database.query.validation.duration
database.query.validation.nodes
database.query.validation.failures
database.query.validation.budget_exceeded
database.query.validation.issues
```

---

# 347. Low cardinality

No utilizar:

```text
table name
query text
parameter value
tenant ID
query fingerprint
```

como metric labels por defecto.

---

# 348. Tracing

Span opcional:

```text
database.query.validate
```

---

# 349. No hard Telemetry dependency

Usar:

```text
ValidationObserverPort
```

o `DiagnosticSink`.

---

# 350. Event integration

Si se emiten eventos:

```text
QueryValidationStarted
QueryValidationCompleted
QueryValidationFailed
```

serán observacionales.

---

# 351. Observer failure

No deberá cambiar semántica de la query salvo policy explícita para infraestructura crítica.

---

# 352. Validation exceptions

Jerarquía conceptual:

```text
QueryValidationException
├── InvalidQueryStructureException
├── QueryInvariantViolationException
├── InvalidExpressionStructureException
├── InvalidPredicateStructureException
├── InvalidParameterStructureException
├── InvalidClauseStructureException
├── InvalidMetadataException
├── InvalidInsertQueryException
├── InvalidUpdateQueryException
├── InvalidDeleteQueryException
├── InvalidSelectQueryException
├── QueryComplexityLimitException
├── QueryValidationBudgetExceededException
├── QueryValidationExtensionException
├── QueryValidationRuleConflictException
└── UnsupportedQueryNodeException
```

---

# 353. Aggregate exception

Para collect mode:

```text
QueryValidationFailedException
└── QueryValidationReport
```

---

# 354. Domain exception

No exponer:

```text
TypeError
Undefined index
PDOException
```

como representación normal de validation failure.

---

# 355. Internal bug

Una invariant interna imposible puede producir:

```text
DatabaseInvariantViolationException
```

diferenciada de query inválida del usuario.

---

# 356. Error ownership

Debe distinguirse:

```text
Invalid User Query
Invalid Extension Node
Framework Bug
Budget Violation
Policy Violation
```

---

# 357. Testing architecture

Se requerirán:

```text
Unit Validation Tests
Query Root Tests
Expression Tests
Predicate Tests
Parameter Tests
Metadata Tests
Clause Tests
Mutation Safety Tests
Complexity Tests
Extension Tests
Property-Based Tests
Persistent Runtime Tests
Concurrency Tests
Architecture Tests
```

---

# 358. Valid query fixtures

Debe existir un catálogo de:

```text
valid normalized AST fixtures
```

que siempre pasen.

---

# 359. Invalid query fixtures

Cada invariant deberá poseer al menos un caso negativo.

---

# 360. Normalization/validation contract tests

Toda salida del Normalizer core deberá satisfacer las invariantes de normalized validation.

---

# 361. Property

```text
validate(normalize(validQuery))
=
VALID
```

para queries válidas generadas.

---

# 362. Invalid construction property

Queries deliberadamente corruptas deberán:

```text
fail deterministically
```

sin:

- infinite loops;
- stack overflow;
- OOM;
- worker state leak.

---

# 363. Persistent worker test

Conceptualmente:

```text
for request in 1..10000:
    build query
    normalize
    validate
    discard request state
```

Verificar:

- memory stability;
- no stale issues;
- no current query leakage;
- no registry mutation.

---

# 364. Concurrency tests

Varias queries simultáneas deberán compartir validators sin compartir `ValidationState`.

---

# 365. Extension conformance suite

Toda Query extension deberá poder ejecutar:

```text
QueryValidationExtensionConformanceSuite
```

---

# 366. Architecture tests

Deberán detectar imports prohibidos desde Validation hacia:

```text
PDO
Driver
Connection
Pool
ORM
EntityManager
UnitOfWork
HTTP
Authentication
Authorization
FrankenPHP concrete APIs
RoadRunner concrete APIs
OpenSwoole concrete APIs
```

---

# 367. Platform checks forbidden

No:

```php
if ($platform === 'postgresql') {
}
```

en core structural validation.

---

# 368. Vendor checks forbidden

No:

```php
if ($driver === 'pdo.mysql') {
}
```

---

# 369. Service Locator forbidden

No:

```php
app(DatabaseManager::class);
```

dentro de validation rules.

---

# 370. Environment access forbidden

No:

```php
getenv()
$_ENV
config()
```

dentro de validation rules.

---

# 371. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Validation\
```

---

# 372. Proposed directory structure

```text
Query/
└── Validation/
    ├── Contract/
    │   ├── QueryValidatorInterface.php
    │   ├── QueryValidationEngineInterface.php
    │   ├── QueryValidationRuleInterface.php
    │   ├── ValidationDiagnosticSinkInterface.php
    │   └── ExtensionValidationHandlerInterface.php
    │
    ├── Core/
    │   ├── QueryValidator.php
    │   ├── QueryValidationEngine.php
    │   ├── ValidatedQueryArtifact.php
    │   ├── QueryValidationReport.php
    │   ├── QueryValidationIssue.php
    │   └── QueryValidationSummary.php
    │
    ├── Context/
    │   ├── QueryValidationContext.php
    │   ├── QueryValidationRuleContext.php
    │   ├── QueryValidationPolicy.php
    │   ├── ValidationProfile.php
    │   └── ValidationMode.php
    │
    ├── State/
    │   ├── ValidationState.php
    │   ├── ValidationPath.php
    │   ├── ValidationTraversalState.php
    │   └── ValidationIssueCollector.php
    │
    ├── Budget/
    │   ├── QueryValidationBudget.php
    │   ├── QueryValidationBudgetTracker.php
    │   └── QueryComplexityCounter.php
    │
    ├── Rule/
    │   ├── Root/
    │   ├── Structural/
    │   ├── Expression/
    │   ├── Predicate/
    │   ├── Parameter/
    │   ├── Metadata/
    │   ├── Source/
    │   ├── Join/
    │   ├── Projection/
    │   ├── Grouping/
    │   ├── Ordering/
    │   ├── Pagination/
    │   ├── Locking/
    │   ├── Cte/
    │   ├── SetOperation/
    │   ├── Insert/
    │   ├── Update/
    │   ├── Delete/
    │   ├── Raw/
    │   ├── Complexity/
    │   └── Final/
    │
    ├── Registry/
    │   ├── ValidationRuleRegistry.php
    │   ├── ValidationRuleDescriptor.php
    │   └── ValidationRuleSetFingerprint.php
    │
    ├── Mutation/
    │   ├── QueryMutationSafetyPolicy.php
    │   ├── FullTableMutationValidator.php
    │   └── FullTableMutationIntent.php
    │
    ├── Portability/
    │   ├── QueryPortabilityPolicy.php
    │   └── PortabilityValidator.php
    │
    ├── Security/
    │   ├── RawQueryValidator.php
    │   ├── RawExpressionValidator.php
    │   └── IdentifierSafetyValidator.php
    │
    ├── Extension/
    │   ├── ExtensionValidationRegistry.php
    │   ├── ExtensionValidationDescriptor.php
    │   └── QueryValidationExtensionConformanceSuite.php
    │
    ├── Diagnostic/
    │   ├── QueryValidationDiagnostic.php
    │   ├── QueryValidationSuggestion.php
    │   └── QueryValidationDiagnosticRenderer.php
    │
    └── Exception/
        ├── QueryValidationException.php
        ├── QueryValidationFailedException.php
        ├── QueryInvariantViolationException.php
        ├── QueryComplexityLimitException.php
        ├── QueryValidationBudgetExceededException.php
        ├── QueryValidationExtensionException.php
        └── UnsupportedQueryNodeException.php
```

---

# 373. Dependency direction

```text
Query AST
    │
    ▼
Normalization
    │
    ▼
Validation
    │
    ├── Query AST Contracts
    ├── Expression Contracts
    ├── Predicate Contracts
    ├── Parameter Contracts
    ├── Query Type References
    ├── Query Metadata
    └── Query Context
    │
    ▼
Validated Query Artifact
    │
    ▼
Semantic Query Engine
```

---

# 374. Forbidden dependency direction

Nunca:

```text
Validation
   │
   ├──► SQL Compiler
   ├──► Executor
   ├──► Connection
   ├──► Driver
   ├──► ORM
   └──► Application Services
```

---

# 375. DB-QVAL-001

Toda Query AST estructurada deberá validarse antes de Semantic Analysis.

---

# 376. DB-QVAL-002

El Validator recibirá preferentemente un `NormalizedQueryAst`.

---

# 377. DB-QVAL-003

Validation no modificará el AST.

---

# 378. DB-QVAL-004

Validation no actuará como Normalizer.

---

# 379. DB-QVAL-005

Validation no actuará como Optimizer.

---

# 380. DB-QVAL-006

Validation no actuará como Planner.

---

# 381. DB-QVAL-007

Validation no actuará como Compiler.

---

# 382. DB-QVAL-008

Validation no ejecutará queries.

---

# 383. DB-QVAL-009

Validation no abrirá conexiones.

---

# 384. DB-QVAL-010

Validation no resolverá PDO/native resources.

---

# 385. DB-QVAL-011

Validation no consultará el servidor DB implícitamente.

---

# 386. DB-QVAL-012

Validation no resolverá tablas contra schema durante structural validation.

---

# 387. DB-QVAL-013

Validation no resolverá columnas contra schema durante structural validation.

---

# 388. DB-QVAL-014

Validation no inferirá tipos efectivos durante structural validation.

---

# 389. DB-QVAL-015

Validation no comprobará soporte del Platform durante structural validation.

---

# 390. DB-QVAL-016

Validation no contendrá vendor checks en core rules.

---

# 391. DB-QVAL-017

Validation no contendrá Driver checks en core rules.

---

# 392. DB-QVAL-018

Validation será determinista.

---

# 393. DB-QVAL-019

Validation estará bounded por resource limits.

---

# 394. DB-QVAL-020

AST cycles serán inválidos.

---

# 395. DB-QVAL-021

AST depth será limitada.

---

# 396. DB-QVAL-022

AST node count será limitada.

---

# 397. DB-QVAL-023

Issue count será limitado.

---

# 398. DB-QVAL-024

Invariants absolutas no podrán deshabilitarse por configuration.

---

# 399. DB-QVAL-025

Policies configurables estarán separadas de invariants.

---

# 400. DB-QVAL-026

El Query Validation System no utilizará estado global mutable.

---

# 401. DB-QVAL-027

ValidationState será operation-scoped.

---

# 402. DB-QVAL-028

Validation rules compartidas serán stateless o immutable.

---

# 403. DB-QVAL-029

Validation será segura para persistent workers.

---

# 404. DB-QVAL-030

Validation será segura para ejecución concurrente.

---

# 405. DB-QVAL-031

Validation será runtime-neutral.

---

# 406. DB-QVAL-032

Validation será ejecutable offline.

---

# 407. DB-QVAL-033

Validation no dependerá de HTTP.

---

# 408. DB-QVAL-034

Validation no dependerá de Authentication.

---

# 409. DB-QVAL-035

Validation no dependerá de Authorization.

---

# 410. DB-QVAL-036

Validation no dependerá de Multitenancy.

---

# 411. DB-QVAL-037

Multitenancy podrá contribuir policies mediante integration ports.

---

# 412. DB-QVAL-038

Raw SQL será explícito.

---

# 413. DB-QVAL-039

Raw SQL no será parseado automáticamente por el core validator.

---

# 414. DB-QVAL-040

Raw user input no podrá considerarse trusted SQL implícitamente.

---

# 415. DB-QVAL-041

Runtime parameter values serán redactados en diagnostics por defecto.

---

# 416. DB-QVAL-042

Identifiers serán estructurados.

---

# 417. DB-QVAL-043

Identifiers no serán quoted durante Validation.

---

# 418. DB-QVAL-044

Identifier equality dependiente de Platform se diferirá.

---

# 419. DB-QVAL-045

Projection order será preservado.

---

# 420. DB-QVAL-046

JOIN order será preservado.

---

# 421. DB-QVAL-047

Assignment order será preservado.

---

# 422. DB-QVAL-048

CTE order será preservado.

---

# 423. DB-QVAL-049

CASE branch order será preservado.

---

# 424. DB-QVAL-050

Parameter IDs deberán ser consistentes dentro de su scope.

---

# 425. DB-QVAL-051

Parameter descriptors incompatibles para la misma identidad serán inválidos.

---

# 426. DB-QVAL-052

Placeholder syntax no formará parte de Validation.

---

# 427. DB-QVAL-053

Driver parameter limits se validarán cuando exista target capability information.

---

# 428. DB-QVAL-054

AND/OR normalized nodes deberán satisfacer su arity canónica.

---

# 429. DB-QVAL-055

Nested redundant logical groups después de normalization serán invariant violations.

---

# 430. DB-QVAL-056

Empty IN después de normalization será inválido si la política canónica exige su reducción.

---

# 431. DB-QVAL-057

Insert row arity deberá ser estructuralmente consistente.

---

# 432. DB-QVAL-058

Insert source forms serán mutuamente exclusivas.

---

# 433. DB-QVAL-059

Update deberá contener al menos una assignment.

---

# 434. DB-QVAL-060

Duplicate assignments inequívocos serán inválidos.

---

# 435. DB-QVAL-061

Full-table UPDATE no será universalmente estructuralmente inválido.

---

# 436. DB-QVAL-062

Full-table DELETE no será universalmente estructuralmente inválido.

---

# 437. DB-QVAL-063

Mutation safety será una policy explícita.

---

# 438. DB-QVAL-064

Mutation safety podrá requerir explicit intent.

---

# 439. DB-QVAL-065

SELECT sin FROM podrá ser válido.

---

# 440. DB-QVAL-066

SELECT con projection vacía después de normalization será inválido.

---

# 441. DB-QVAL-067

Wildcard no será expandido por Validation.

---

# 442. DB-QVAL-068

HAVING sin GROUP BY no será rechazado estructuralmente sin fundamento semántico.

---

# 443. DB-QVAL-069

Set operation arity se validará temprano sólo cuando sea inequívoca.

---

# 444. DB-QVAL-070

CTE semantic recursion se diferenciará de AST object cycles.

---

# 445. DB-QVAL-071

Extension nodes deberán poseer validation support cuando sea requerido.

---

# 446. DB-QVAL-072

Extension validators se registrarán durante bootstrap.

---

# 447. DB-QVAL-073

Validation registries quedarán frozen antes de runtime.

---

# 448. DB-QVAL-074

No habrá last-registered-wins silencioso.

---

# 449. DB-QVAL-075

Extension validation deberá ser determinista.

---

# 450. DB-QVAL-076

Extension validation deberá respetar QueryValidationBudget.

---

# 451. DB-QVAL-077

Extension validation no realizará hidden I/O.

---

# 452. DB-QVAL-078

Validation error codes serán estables.

---

# 453. DB-QVAL-079

Validation errors podrán contener source mapping seguro.

---

# 454. DB-QVAL-080

Validation diagnostics no deberán filtrar secrets.

---

# 455. DB-QVAL-081

Fail-fast y collect mode deberán producir las mismas conclusiones de validez.

---

# 456. DB-QVAL-082

Collect mode no acumulará issues ilimitados.

---

# 457. DB-QVAL-083

NormalizedQueryFingerprint no cambiará por el acto de validar.

---

# 458. DB-QVAL-084

ValidationProfile tendrá identidad/fingerprint cuando afecte caches.

---

# 459. DB-QVAL-085

Validation results cacheados deberán considerar policy y rule-set relevantes.

---

# 460. DB-QVAL-086

Validation no dependerá del orden accidental de hash maps.

---

# 461. DB-QVAL-087

Validation no dependerá del orden accidental de extension discovery.

---

# 462. DB-QVAL-088

Validation no dependerá del tiempo actual.

---

# 463. DB-QVAL-089

Validation no dependerá de random values.

---

# 464. DB-QVAL-090

Validation no dependerá de process IDs.

---

# 465. DB-QVAL-091

Validation no resolverá current user mediante globals.

---

# 466. DB-QVAL-092

Validation no resolverá current tenant mediante globals.

---

# 467. DB-QVAL-093

Validation no utilizará Service Locator.

---

# 468. DB-QVAL-094

Validation no accederá directamente a environment/config globals.

---

# 469. DB-QVAL-095

Validation rules deberán ser unit-testable sin servidor DB.

---

# 470. DB-QVAL-096

Core validation deberá poder ejecutarse sin ORM instalado.

---

# 471. DB-QVAL-097

Core validation deberá poder ejecutarse sin Telemetry instalado.

---

# 472. DB-QVAL-098

Core validation deberá poder ejecutarse sin Cache instalado.

---

# 473. DB-QVAL-099

Core validation deberá poder ejecutarse sin Multitenancy instalado.

---

# 474. DB-QVAL-100

La corrección tendrá prioridad sobre aceptar una query ambigua.

---

# 475. DB-QVAL-101

Pero una query no será rechazada prematuramente cuando su validez dependa de información semántica posterior.

---

# 476. DB-QVAL-102

El sistema deberá distinguir:

```text
INVALID
```

de:

```text
NOT YET RESOLVABLE
```

---

# 477. DB-QVAL-103

Información desconocida no será tratada automáticamente como inválida.

---

# 478. DB-QVAL-104

Una estructura demostrablemente inválida no será diferida innecesariamente.

---

# 479. DB-QVAL-105

Query Validation será una frontera explícita del Query Engine.

---

# 480. Anti-pattern — validator that fixes queries

Incorrecto:

```text
Validator
→ detects empty IN
→ rewrites to FALSE
```

Eso pertenece a Normalization.

---

# 481. Anti-pattern — validator that queries schema

Incorrecto:

```php
if (!$schemaManager->columnExists('users', 'email')) {
}
```

en structural validation.

---

# 482. Anti-pattern — validator that checks database vendor

Incorrecto:

```php
if ($database === 'sqlite' && $query->hasRightJoin()) {
}
```

El soporte efectivo se evaluará posteriormente mediante capabilities.

---

# 483. Anti-pattern — validator as authorization system

Incorrecto:

```php
if (!$currentUser->can('delete-users')) {
    throw ...
}
```

---

# 484. Anti-pattern — validator as SQL parser

Incorrecto:

```text
RawSqlQuery
→ regex validation
→ pretend structured safety
```

---

# 485. Anti-pattern — validator as optimizer

Incorrecto:

```text
Validation
→ remove duplicate predicates
→ reorder joins
→ constant fold
```

---

# 486. Anti-pattern — skip validation for ORM

Incorrecto:

```text
ORM generated it,
therefore it must be valid.
```

---

# 487. Correct rule

ORM-generated queries pasan por el mismo:

```text
Normalize
→ Validate
→ Semantic Analyze
→ Optimize
→ Plan
→ Compile
```

---

# 488. Anti-pattern — trusted extension bypass

Incorrecto:

```text
Official extension
→ skip validation
```

Las extensiones oficiales deberán cumplir las mismas invariantes.

---

# 489. Anti-pattern — validation singleton with mutable state

Incorrecto:

```php
final class QueryValidator
{
    private array $errors = [];
    private ?QueryAst $currentQuery = null;
}
```

si la instancia vive application scope.

---

# 490. Correct alternative

```text
Shared QueryValidator
        │
        ▼
Operation-local ValidationState
```

---

# 491. Anti-pattern — one giant validate() switch

Evitar:

```php
switch ($node::class) {
    // hundreds of cases
}
```

en un único archivo.

---

# 492. Preferred architecture

```text
QueryValidationEngine
        │
        ├── Root Validators
        ├── Expression Validators
        ├── Predicate Validators
        ├── Clause Validators
        ├── Parameter Validators
        ├── Metadata Validators
        ├── Complexity Validators
        └── Extension Validators
```

---

# 493. Validation formula

```text
Validated Query
=
Validate(
    Normalized Query,
    Structural Invariants,
    Validation Policy,
    Validation Budget,
    Frozen Rule Set
)
```

---

# 494. Validity formula

```text
VALID(Q)
=
StructuralValidity(Q)
∧
NormalizedInvariants(Q)
∧
ContextualPolicyValidity(Q)
∧
ResourceGovernanceValidity(Q)
∧
ExtensionValidity(Q)
```

en el alcance de esta fase.

---

# 495. Semantic validity excluded

Todavía no:

```text
SemanticValidity(Q)
```

---

# 496. Capability validity excluded

Todavía no:

```text
CapabilityValidity(Q, Target)
```

---

# 497. Execution validity excluded

Todavía no:

```text
Executable(Q, RuntimeState)
```

---

# 498. Safe validation formula

```text
Safe Query Validation
=
Early Definite Checks
+
Deferred Uncertain Checks
+
Immutable Input
+
Explicit Context
+
Bounded Traversal
+
Structured Diagnostics
+
No Hidden I/O
```

---

# 499. Persistent-safe formula

```text
Persistent-Safe Validation
=
Stateless Validator
+
Frozen Rule Registries
+
Operation-Scoped ValidationState
+
Bounded Resources
+
No Global Current Query
+
No Request Leakage
```

---

# 500. Validation boundary

```text
                    Normalized Query AST
                             │
                             ▼
                    QueryValidationEngine
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
 Structural             Expression             Predicate
 Validation             Validation             Validation
       │                     │                     │
       ├─────────────────────┼─────────────────────┤
       │                     │                     │
       ▼                     ▼                     ▼
 Parameter               Clause                Metadata
 Validation             Validation             Validation
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                             ▼
                   Complexity Validation
                             │
                             ▼
                    Extension Validation
                             │
                             ▼
                      Policy Validation
                             │
                             ▼
                     Final Invariants
                             │
               ┌─────────────┴─────────────┐
               ▼                           ▼
            INVALID                       VALID
               │                           │
               ▼                           ▼
      Validation Report          ValidatedQueryArtifact
                                           │
                                           ▼
                                  Semantic Query Engine
```

---

# 501. Qué garantiza ValidatedQueryArtifact

Una vez producido, Semantic Analysis podrá asumir:

1. el AST es acíclico;
2. el AST respeta límites estructurales;
3. el root es válido;
4. los node kinds están soportados por core o extensión;
5. la forma está normalizada;
6. logical predicates cumplen su arity canónica;
7. expression nodes tienen children estructuralmente válidos;
8. parameters tienen identidades/descriptors consistentes;
9. clauses pertenecen al query root correcto;
10. INSERT rows poseen arity estructural consistente;
11. UPDATE posee assignments;
12. set operations tienen branches válidos;
13. CTEs poseen estructura válida;
14. metadata estructural es válida;
15. raw constructs cumplen policy estructural;
16. extension nodes pasaron sus validators;
17. resource budgets fueron respetados;
18. el artifact no contiene recursos runtime prohibidos.

---

# 502. Qué NO garantiza ValidatedQueryArtifact

Todavía no garantiza:

```text
table exists
column exists
alias resolves
CTE reference resolves
function resolves
operator resolves
types are compatible
projection types are known
GROUP BY is semantically valid
aggregate scope is valid
join relation is semantically valid
correlated reference is valid
platform supports feature
driver supports execution strategy
transaction mode is supported
SQL can be compiled for target
query can execute successfully
```

---

# 503. Frontera con Semantic Query Engine

Esta separación es crítica.

```text
Validation asks:
"Is this query structurally coherent?"

Semantic Analysis asks:
"What does this query mean?"
```

---

# 504. Ejemplo completo

Query:

```php
DB::table('users')
    ->where('active', true)
    ->where('age', '>', 18)
    ->orderBy('name')
    ->get();
```

Pipeline:

```text
Developer API
    │
    ▼
Query Builder
    │
    ▼
Query Model
    │
    ▼
Query AST
    │
    ▼
Normalization
    │
    ▼
SelectQueryNode
├── projection
├── source users
├── predicate
│   └── AND
│       ├── active = P1
│       └── age > P2
├── ordering
│   └── name ASC
└── parameters
    ├── P1
    └── P2
    │
    ▼
Validation
    │
    ├── valid root
    ├── valid source structure
    ├── valid AND arity
    ├── valid comparisons
    ├── valid parameter identities
    ├── valid ordering
    └── complexity within limits
    │
    ▼
ValidatedQueryArtifact
```

Pero todavía no sabemos:

```text
Does users exist?
Does active exist?
Is active boolean?
Does age exist?
Is age numeric?
Does name exist?
```

Eso comienza en Semantic Analysis.

---

# 505. Arquitectura del bloque 23–34

Con este documento queda formalizado:

```text
23 Query Architecture
        │
        ▼
24 Query Model
        │
        ▼
25 Query AST System
        │
        ▼
26 Query AST Node Model
        │
        ├── 27 Expression System
        ├── 28 Predicate System
        ├── 29 Parameter & Binding
        ├── 30 Query Type System
        ├── 31 Query Metadata
        └── 32 Query Context
                │
                ▼
33 Query Normalization
                │
                ▼
34 Query Validation
                │
                ▼
       Validated Query Artifact
```

---

# 506. Resultado del bloque

VoltStack ya dispone conceptualmente de una representación de consultas:

```text
Structured
Typed-capable
Immutable
Portable-first
Parameterized
Metadata-aware
Context-aware
Canonical
Validated
Extensible
Persistent-runtime safe
```

---

# 507. Siguiente frontera arquitectónica

Ahora podrá comenzar:

```text
Semantic Query Engine
```

---

# 508. Pipeline siguiente

```text
ValidatedQueryArtifact
        │
        ▼
Semantic Query Architecture
        │
        ▼
Semantic Analysis
        │
        ├── Symbol Resolution
        ├── Schema Resolution
        ├── Type Inference
        ├── Function Resolution
        ├── Operator Resolution
        ├── Relation Resolution
        ├── Join Resolution
        ├── Aggregate Analysis
        ├── Scope Analysis
        ├── Constraint Analysis
        └── Capability Requirements
        │
        ▼
Semantic Query Graph
```

---

# 509. Regla maestra final

> Query Validation deberá rechazar toda estructura que pueda demostrarse inválida con la información disponible, pero deberá diferir cualquier conclusión que requiera conocimiento semántico, de plataforma, de capabilities o de runtime que todavía no exista.

---

# 510. Decisión arquitectónica final

VoltStack adoptará:

```text
Early Structural Validation
+
Deferred Semantic Validation
+
Deferred Capability Validation
+
Deferred Execution Preconditions
```

en lugar de un único:

```text
validateEverything()
```

monolítico.

La separación definitiva será:

```text
Construction
    │
    ▼
"Can this object exist?"

Normalization
    │
    ▼
"What is its canonical representation?"

Query Validation
    │
    ▼
"Is this canonical structure coherent?"

Semantic Analysis
    │
    ▼
"What does it mean?"

Capability Resolution
    │
    ▼
"Can the selected target support it?"

Optimizer
    │
    ▼
"Can it be improved?"

Planner
    │
    ▼
"How should VoltStack execute it?"

Compiler
    │
    ▼
"How is the plan represented in target SQL?"

Executor
    │
    ▼
"Execute it safely."
```

---

# 511. Cierre del bloque Query Model and AST

Con:

```text
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
```

queda cerrado oficialmente el bloque:

```text
23–34
QUERY MODEL AND AST
```

compuesto por:

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
```

La salida formal del bloque será:

```text
ValidatedQueryArtifact
```

y constituirá el contrato de entrada hacia el Semantic Query Engine.

---

# 512. Próximo documento

El siguiente documento será:

```text
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
```

que iniciará el bloque:

```text
35–42
SEMANTIC QUERY ENGINE
```

con la secuencia:

```text
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
37_DATABASE_SYMBOL_RESOLUTION_SYSTEM.md
38_DATABASE_SCHEMA_AWARE_QUERY_RESOLUTION.md
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
40_DATABASE_RELATION_AND_JOIN_RESOLUTION_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
```

La transformación fundamental del siguiente bloque será:

```text
Validated Query AST
        │
        ▼
Semantic Analysis
        │
        ├── Symbols
        ├── Scopes
        ├── Schema
        ├── Types
        ├── Functions
        ├── Operators
        ├── Relations
        ├── Joins
        ├── Aggregates
        ├── Constraints
        └── Capability Requirements
        │
        ▼
Semantic Query Graph
```

A partir de ese momento VoltStack dejará de trabajar únicamente con la **estructura** de la consulta y comenzará a comprender formalmente su **significado**.