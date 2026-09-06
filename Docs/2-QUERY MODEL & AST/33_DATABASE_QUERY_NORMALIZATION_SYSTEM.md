# 33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md

# VoltStack Quantum Database
## Query Normalization System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 33 — Query Normalization System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Normalization Layer  
**Versión:** 1.0

---

# 1. Propósito

Este documento define la arquitectura oficial del:

```text
VoltStack/Quantum/Database Query Normalization System
```

El sistema será responsable de transformar diferentes representaciones estructuralmente equivalentes de una consulta hacia una forma:

- canónica;
- determinista;
- estable;
- estructurada;
- portable;
- segura;
- idempotente;
- adecuada para análisis posterior.

La transformación conceptual será:

```text
Query Model / Query AST
        │
        ▼
Non-Canonical Representation
        │
        ▼
Normalization Pipeline
        │
        ▼
Canonical Query AST
        │
        ▼
Structural Validation
        │
        ▼
Semantic Analysis
```

La regla fundamental será:

```text
normalize(normalize(Q))
=
normalize(Q)
```

---

# 2. Problema arquitectónico

Dos consultas pueden expresar la misma intención utilizando estructuras diferentes.

Ejemplo conceptual:

```text
WHERE active = true
AND age > 18
```

y:

```text
WHERE
    AND(
        active = true,
        age > 18
    )
```

pueden representar exactamente la misma estructura semántica.

También pueden aparecer variaciones como:

```text
NOT(NOT(P))
```

o:

```text
AND(P)
```

o:

```text
AND(
    P,
    AND(Q, R)
)
```

Estas diferencias dificultan:

- fingerprints;
- caching;
- structural equality;
- semantic analysis;
- optimizer rules;
- planner rules;
- diagnostics;
- testing;
- compiled query caching.

---

# 3. Solución

VoltStack introducirá una fase explícita de normalización.

```text
Query AST
   │
   ▼
NormalizationContext
   │
   ▼
NormalizationPipeline
   │
   ├── Structural Rules
   ├── Expression Rules
   ├── Predicate Rules
   ├── Parameter Rules
   ├── Identifier Rules
   ├── Clause Rules
   └── Extension Rules
   │
   ▼
NormalizedQueryAst
```

---

# 4. Regla maestra

> La normalización cambiará la forma estructural de una consulta únicamente cuando pueda preservar su intención semántica.

---

# 5. Normalization ≠ Semantic Analysis

La normalización no deberá resolver todavía:

```text
column existence
table existence
relationship meaning
database type compatibility
platform support
function overload
operator overload
foreign keys
schema relations
```

Eso corresponde principalmente a Semantic Analysis.

---

# 6. Normalization ≠ Optimization

Tampoco deberá intentar elegir el plan más rápido.

```text
Normalization
=
canonicalization

Optimization
=
semantics-preserving improvement
```

---

# 7. Ejemplo

Transformar:

```text
AND(
    P,
    AND(Q, R)
)
```

en:

```text
AND(
    P,
    Q,
    R
)
```

es normalización.

Pero cambiar:

```text
JOIN A
JOIN B
```

por:

```text
JOIN B
JOIN A
```

para intentar mejorar rendimiento pertenece al Optimizer y sólo cuando sea semánticamente válido.

---

# 8. Normalization ≠ Compilation

La normalización nunca deberá producir:

```text
SELECT ...
WHERE ...
```

como SQL.

---

# 9. Normalization ≠ Dialect Transformation

No deberá convertir:

```text
semantic upsert
```

en:

```text
ON CONFLICT
```

o:

```text
ON DUPLICATE KEY UPDATE
```

---

# 10. Normalization ≠ Platform Resolution

No deberá preguntar:

```php
if ($platform === 'postgresql') {
}
```

para canonicalizar una query portable.

---

# 11. Objetivos

El sistema deberá:

1. producir representación canónica;
2. eliminar variaciones estructurales irrelevantes;
3. estabilizar fingerprints;
4. simplificar Semantic Analysis;
5. simplificar Optimizer y Planner;
6. garantizar determinismo;
7. garantizar idempotencia;
8. soportar extensiones;
9. detectar ciclos de rewrite;
10. limitar complejidad;
11. preservar metadata relevante;
12. preservar source information cuando sea posible;
13. mantener AST immutable;
14. ser seguro para persistent workers;
15. permitir ejecución offline.

---

# 12. No objetivos

No deberá:

- ejecutar SQL;
- consultar PDO;
- abrir conexiones;
- resolver current tenant;
- resolver current user;
- decidir replica;
- iniciar transacciones;
- generar SQL;
- hidratar entidades;
- elegir índices;
- calcular planes físicos;
- reordenar operaciones basándose en estadísticas;
- sustituir al optimizer del servidor.

---

# 13. Posición en el pipeline

La arquitectura conceptual será:

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
Initial Structural Validation
     │
     ▼
Policy Transformations
     │
     ▼
Normalization
     │
     ▼
Canonical AST
     │
     ▼
Structural Validation
     │
     ▼
Semantic Analysis
```

---

# 14. Validación antes y después

La validación podrá ocurrir en múltiples niveles.

Antes de normalizar:

```text
basic structural validity
```

Después:

```text
canonical structural validity
```

Posteriormente:

```text
semantic validity
```

---

# 15. Razón

El normalizador no deberá recibir estructuras arbitrariamente corruptas.

Pero algunas estructuras válidas sólo podrán alcanzar su forma canónica mediante normalización.

---

# 16. NormalizedQueryAst

Se introducirá conceptualmente:

```text
NormalizedQueryAst
```

como artifact explícito.

---

# 17. No boolean flag

Evitar:

```php
$ast->normalized = true;
```

Preferir:

```text
QueryAst
→ Normalization
→ NormalizedQueryAst
```

---

# 18. Ventaja

Esto hace imposible confundir fácilmente:

```text
raw AST
```

con:

```text
canonical AST
```

---

# 19. Type-state architecture

Conceptualmente:

```text
QueryAst
        │
        ▼
NormalizedQueryAst
        │
        ▼
ValidatedQueryAst
        │
        ▼
SemanticQueryArtifact
```

No necesariamente deberán ser clases completamente distintas si el costo resulta excesivo, pero la frontera deberá existir conceptualmente.

---

# 20. Normalization contract

Conceptualmente:

```php
interface QueryNormalizerInterface
{
    public function normalize(
        QueryAst $query,
        NormalizationContext $context,
    ): NormalizedQueryAst;
}
```

---

# 21. NormalizationContext

Definido conceptualmente en:

```text
32_DATABASE_QUERY_CONTEXT_SYSTEM.md
```

Podrá contener:

```text
NormalizationPolicy
NormalizationRuleRegistry
NormalizationBudget
DiagnosticSink
ExtensionContext
```

---

# 22. NormalizationState

El mutable progress pertenecerá a:

```text
NormalizationState
```

y no al contexto estable.

---

# 23. Estado temporal

Puede incluir:

```text
currentPass
rewriteCount
visitedNodes
changedNodes
ruleApplications
memoization
cycleDetection
budgetConsumption
```

---

# 24. Lifecycle

```text
CREATED
   │
   ▼
NORMALIZING
   │
   ▼
CONVERGED
   │
   ▼
VALIDATED
   │
   ▼
COMPLETED
```

Fallos:

```text
FAILED
BUDGET_EXCEEDED
NON_CONVERGENT
```

---

# 25. Pipeline de normalización

```text
Input AST
   │
   ▼
Root Normalization
   │
   ▼
Clause Normalization
   │
   ▼
Source Normalization
   │
   ▼
Expression Normalization
   │
   ▼
Predicate Normalization
   │
   ▼
Parameter Normalization
   │
   ▼
Metadata Normalization
   │
   ▼
Extension Normalization
   │
   ▼
Canonicalization Pass
   │
   ▼
Convergence Check
   │
   ▼
Normalized AST
```

---

# 26. Pass-based architecture

VoltStack podrá ejecutar reglas por passes.

Ejemplo:

```text
Pass 1
Structural Canonicalization

Pass 2
Expression Canonicalization

Pass 3
Predicate Canonicalization

Pass 4
Clause Canonicalization

Pass 5
Parameter Canonicalization

Pass 6
Extension Canonicalization

Pass 7
Final Canonical Validation
```

---

# 27. No requirement for seven physical traversals

La implementación podrá fusionar passes por rendimiento.

La arquitectura describe responsabilidades, no obliga a recorrer el AST siete veces.

---

# 28. NormalizationRule

La unidad básica será:

```text
NormalizationRule
```

---

# 29. Contract conceptual

```php
interface NormalizationRuleInterface
{
    public function supports(AstNode $node): bool;

    public function normalize(
        AstNode $node,
        NormalizationRuleContext $context,
    ): AstNode;
}
```

---

# 30. Prefer specialized contracts

Para hot paths podrá evitarse `supports()` dinámico mediante registries por NodeType.

Ejemplo:

```text
NodeType
→ precompiled rule list
```

---

# 31. Rule registry

```text
NormalizationRuleRegistry
```

será:

- specialized;
- deterministic;
- bootstrap-built;
- frozen at runtime.

---

# 32. No universal registry

No deberá convertirse en:

```text
DatabaseRegistry
```

para todas las extensiones.

---

# 33. Rule ordering

El orden de reglas será explícito.

---

# 34. Ordering mechanisms

Podrá soportar:

```text
priority
before
after
phase
```

---

# 35. Deterministic ordering

Dos ejecuciones con las mismas extensiones deberán producir el mismo orden.

---

# 36. Collision handling

No habrá:

```text
last registered wins
```

silencioso.

---

# 37. Rule categories

Conceptualmente:

```text
STRUCTURAL
IDENTIFIER
SOURCE
EXPRESSION
PREDICATE
PARAMETER
CLAUSE
METADATA
EXTENSION
FINALIZATION
```

---

# 38. Structural normalization

Incluye canonicalización de estructuras generales.

Ejemplos:

```text
single-element containers
nested equivalent containers
empty optional clauses
canonical child collections
canonical node variants
```

---

# 39. Empty structures

Ejemplo:

```text
ORDER BY []
```

podrá normalizarse a:

```text
no OrderByClause
```

---

# 40. Empty WHERE

Una cláusula WHERE sin predicate válido no deberá convertirse arbitrariamente en `TRUE`.

Según el origen puede:

- eliminarse si representa ausencia real;
- rechazarse si representa una estructura inválida.

---

# 41. Important distinction

```text
absence
≠
empty
≠
TRUE
≠
FALSE
```

---

# 42. Canonical optional representation

Cada optional clause deberá tener una representación canónica.

Ejemplo:

```text
null
```

o:

```text
OptionalClause::none()
```

pero no múltiples formas equivalentes mezcladas.

---

# 43. Collection normalization

Collections del AST deberán usar:

- orden semántico;
- representación estable;
- sin nulls;
- sin wrappers innecesarios.

---

# 44. Ordering caution

No toda colección puede ordenarse.

Ejemplo:

```text
SELECT a, b
```

no es necesariamente equivalente a:

```text
SELECT b, a
```

porque cambia el orden del resultado.

---

# 45. Semantic order preservation

Normalización nunca deberá ordenar elementos cuya posición tenga significado observable.

---

# 46. Identifier normalization

Los identificadores permanecerán estructurados.

---

# 47. No quoting

Nunca:

```text
users
→
`users`
```

durante normalización.

---

# 48. Identifier canonicalization

Puede incluir:

```text
split qualified identifier
normalize structural representation
remove redundant empty qualification
normalize alias object representation
validate basic identifier shape
```

---

# 49. Case normalization caution

No convertir indiscriminadamente:

```text
Users
→
users
```

porque la semántica de case folding depende del Platform/Dialect/schema.

---

# 50. Rule

> La normalización estructural de identifiers no deberá destruir información necesaria para la resolución semántica posterior.

---

# 51. Qualified identifiers

Ejemplo:

```text
"users.id" as one string
```

deberá haberse convertido antes o durante construcción a:

```text
QualifiedIdentifier
├── users
└── id
```

Nunca se deberá depender del `.` como string durante semantic analysis.

---

# 52. Alias normalization

Aliases deberán utilizar:

```text
Alias
```

como Value Object.

---

# 53. Source normalization

Sources:

```text
TableSource
SubquerySource
CteSource
FunctionSource
DerivedSource
ExtensionSource
```

deberán alcanzar una representación estructural consistente.

---

# 54. Nested queries

Cada subquery tendrá su propio subtree normalizable.

---

# 55. Recursive normalization

Normalizar una query implica normalizar sus:

```text
subqueries
CTEs
derived tables
nested predicates
nested expressions
set-operation branches
```

---

# 56. Depth protection

La recursión estará protegida por:

```text
NormalizationBudget
```

---

# 57. Expression normalization

Las expresiones serán canonicalizadas sin resolver todavía completamente sus tipos.

---

# 58. Example arithmetic structure

```text
a + (b + c)
```

no deberá reordenarse automáticamente a:

```text
(a + b) + c
```

si la equivalencia puede depender de:

- type;
- overflow;
- precision;
- operator semantics.

---

# 59. Semantic conservatism

La normalización sólo aplicará transformations cuya equivalencia pueda garantizarse sin información semántica faltante.

---

# 60. Literal normalization

Literals podrán canonicalizar su representación interna.

Ejemplo:

```text
BooleanLiteral(true)
```

en vez de múltiples representaciones:

```text
1
"true"
TRUE
```

cuando el Builder ya conoce que el valor representa un boolean literal.

---

# 61. Parameter ≠ literal

Un valor runtime normal deberá continuar como:

```text
ParameterExpression
```

no convertirse en SQL literal.

---

# 62. Null normalization

`NULL` deberá tener un node/value explícito:

```text
NullLiteral
```

cuando realmente sea literal estructural.

---

# 63. Null comparison

Una expresión de alto nivel:

```text
column = NULL
```

podrá canonicalizarse a:

```text
IS NULL
```

sólo si la API/AST define inequívocamente esa intención.

---

# 64. Caution

No debe transformarse SQL raw con `= NULL`.

Raw SQL no pasa por semantic structured normalization completa.

---

# 65. Boolean predicate normalization

Predicates son uno de los principales objetivos.

---

# 66. Flatten AND

```text
AND(
    A,
    AND(B, C),
    D
)
```

→

```text
AND(
    A,
    B,
    C,
    D
)
```

---

# 67. Flatten OR

```text
OR(
    A,
    OR(B, C),
    D
)
```

→

```text
OR(
    A,
    B,
    C,
    D
)
```

---

# 68. Single-child AND

```text
AND(A)
```

→

```text
A
```

---

# 69. Single-child OR

```text
OR(A)
```

→

```text
A
```

---

# 70. Double negation

```text
NOT(NOT(A))
```

→

```text
A
```

cuando la predicate semantics lo garantice.

---

# 71. Boolean constants

Si existen:

```text
TruePredicate
FalsePredicate
```

pueden aplicarse reglas seguras.

---

# 72. AND identity

```text
AND(A, TRUE)
```

→

```text
A
```

---

# 73. AND annihilator

```text
AND(A, FALSE)
```

→

```text
FALSE
```

si no existen side effects.

---

# 74. Predicates are side-effect free

El Query AST deberá modelar predicates como expresiones declarativas sin side effects.

Esto hace seguras ciertas simplificaciones.

---

# 75. OR identity

```text
OR(A, FALSE)
```

→

```text
A
```

---

# 76. OR annihilator

```text
OR(A, TRUE)
```

→

```text
TRUE
```

---

# 77. Empty AND/OR

No deberán aparecer normalmente.

Si aparecen mediante extensión:

```text
AND()
OR()
```

la política deberá definir si:

- se rechazan;
- se canonicalizan a identidad lógica.

Preferencia inicial:

```text
reject structurally ambiguous empty logical groups
```

salvo que el node contract defina formalmente su identidad.

---

# 78. Predicate order

No se deberá reordenar automáticamente:

```text
AND(A, B)
```

a:

```text
AND(B, A)
```

en la normalización inicial.

---

# 79. Why

Aunque la lógica pura sea conmutativa, preservar el orden ayuda a:

- diagnostics;
- parameter ordering;
- source mapping;
- predictable SQL;
- developer expectations.

El Optimizer podrá decidir cambios posteriormente cuando sean seguros.

---

# 80. Duplicate predicates

```text
AND(A, A)
```

matemáticamente podría reducirse a:

```text
A
```

pero la V1 no deberá asumir structural duplicate elimination como normalización general.

---

# 81. Reason

Puede afectar:

- parameter identity;
- diagnostics;
- source mapping;
- extension semantics;
- query shape expectations.

La deduplicación avanzada pertenece al Optimizer.

---

# 82. De Morgan transformations

```text
NOT(AND(A, B))
```

→

```text
OR(NOT(A), NOT(B))
```

es semánticamente válida en lógica clásica, pero SQL posee three-valued logic.

Por tanto no será una normalización universal sin análisis semántico apropiado.

---

# 83. SQL three-valued logic

VoltStack deberá respetar:

```text
TRUE
FALSE
UNKNOWN
```

en predicates SQL.

---

# 84. Important consequence

Reglas booleanas deberán demostrar compatibilidad con three-valued logic.

---

# 85. Comparison normalization

Comparisons deberán utilizar operadores canónicos.

Ejemplo:

```text
EQUAL
NOT_EQUAL
GREATER_THAN
GREATER_THAN_OR_EQUAL
LESS_THAN
LESS_THAN_OR_EQUAL
```

No strings:

```text
=
<>
!=
>=
```

en el AST semántico.

---

# 86. Syntax aliases

El Builder puede aceptar:

```text
!=
<>
```

pero ambos deberán producir:

```text
ComparisonOperator::NOT_EQUAL
```

---

# 87. Symmetric comparison

No reordenar automáticamente:

```text
5 < age
```

a:

```text
age > 5
```

en V1.

Aunque pueda ser equivalente, puede afectar source mapping/fingerprints y requerir type/operator semantics.

---

# 88. BETWEEN

No convertir automáticamente:

```text
x BETWEEN a AND b
```

a:

```text
x >= a AND x <= b
```

porque se perdería intención estructural y pueden existir diferencias de compilación/optimization.

---

# 89. IN predicates

`IN` mantendrá un node específico.

---

# 90. Empty IN

Una API de alto nivel puede producir:

```text
x IN []
```

El sistema deberá tener política explícita.

---

# 91. Preferred high-level semantics

Si el Builder define:

```text
whereIn('id', [])
```

como “no rows”:

```text
x IN []
→
FalsePredicate
```

puede realizarse durante Query Model construction o normalization.

---

# 92. NOT IN empty

Análogamente:

```text
x NOT IN []
→
TruePredicate
```

si ésa es la semántica oficial de la API.

---

# 93. Important

Esta decisión pertenece a la semántica de la API VoltStack, no a una suposición de SQL syntax.

---

# 94. IN value normalization

Los valores se representarán mediante:

```text
ValueList
Subquery
ParameterList
```

según corresponda.

---

# 95. Parameter list expansion

No necesariamente se expandirá a placeholders durante normalización.

Eso pertenece a binding compilation.

---

# 96. LIKE normalization

Mantendrá:

```text
LikePredicate
```

con:

```text
expression
pattern
negated
escape?
caseSensitivity?
```

sin generar syntax.

---

# 97. ILIKE

No deberá convertirse en:

```text
LOWER(column) LIKE LOWER(?)
```

durante normalización.

Eso sería estrategia/emulación y pertenece al Planner/Compiler.

---

# 98. EXISTS normalization

```text
ExistsPredicate
```

se mantendrá como intención semántica.

---

# 99. NULL predicates

Preferir nodes explícitos:

```text
IsNullPredicate
IsNotNullPredicate
```

---

# 100. Parameter normalization

El sistema definido en:

```text
29_DATABASE_QUERY_PARAMETER_AND_BINDING_SYSTEM.md
```

mantendrá separación entre:

```text
QueryParameter
CompiledParameter
DriverBinding
```

---

# 101. Parameter identity

Normalization podrá estabilizar:

```text
QueryParameterId
```

cuando la identidad dependa de estructura.

---

# 102. Parameter values

Los valores runtime no deberán afectar normalmente la forma canónica.

---

# 103. Example

```text
age > 18
```

y:

```text
age > 30
```

podrán compartir:

```text
QueryShapeFingerprint
```

si ambos valores son runtime parameters del mismo tipo/rol.

---

# 104. Value-sensitive exceptions

Algunas estrategias pueden depender del valor estructural.

Ejemplos:

```text
LIMIT
OFFSET
identifier
array arity
compile-time constant
```

La política de fingerprint deberá distinguirlos cuando sea necesario.

---

# 105. Parameter numbering

No generar:

```text
$1
$2
?
:parameter
```

durante normalization.

---

# 106. Placeholder allocation

Pertenece a:

```text
CompilationContext
```

---

# 107. Named builder parameters

Si Developer API usa:

```text
:min_age
```

podrá convertirse a identidad semántica independiente de la sintaxis final.

---

# 108. Parameter deduplication

No deduplicar valores simplemente porque sean iguales.

```text
a = 10 AND b = 10
```

puede mantener dos parameter identities.

---

# 109. Why

La deduplicación de bindings depende de:

- driver;
- placeholder strategy;
- type;
- compiler;
- diagnostics;
- execution semantics.

---

# 110. Query Type normalization

El sistema de tipos del documento 30 podrá canonicalizar aliases de tipos conocidos.

Ejemplo:

```text
int
integer
```

→

```text
QueryTypeId::INTEGER
```

si ambos representan exactamente el mismo tipo VoltStack.

---

# 111. Type hints

Los type hints declarados deberán canonicalizarse sin pretender resolver el tipo efectivo final.

---

# 112. Unknown type

Mantener:

```text
UnknownQueryType
```

cuando aún no pueda resolverse.

No adivinar.

---

# 113. Metadata normalization

El documento 31 distingue metadata declarativa.

La normalización podrá:

- canonicalizar keys;
- validar namespaces;
- ordenar metadata que sea semánticamente unordered;
- eliminar metadata redundante;
- preservar provenance.

---

# 114. Metadata caution

No toda metadata participa en:

```text
QueryShapeFingerprint
```

---

# 115. Fingerprint categories

Metadata podrá clasificarse:

```text
SEMANTIC
EXECUTION
DIAGNOSTIC
OBSERVATIONAL
EXTENSION
```

---

# 116. Semantic metadata

Puede afectar normalized fingerprint.

---

# 117. Diagnostic metadata

Normalmente no.

Ejemplo:

```text
debug label
source line
trace annotation
```

---

# 118. Clause normalization

Cada query operation tendrá reglas específicas.

---

# 119. SELECT normalization

Puede normalizar:

```text
projection representation
FROM representation
JOIN containers
WHERE predicate
GROUP BY representation
HAVING predicate
WINDOW definitions
SET operations
ORDER BY
pagination
locking
returning where applicable
```

---

# 120. Projection order

Se preservará.

---

# 121. Duplicate projection

No eliminar automáticamente:

```text
SELECT id, id
```

porque cambia result shape.

---

# 122. Wildcard

Mantener:

```text
WildcardProjection
```

hasta Semantic Analysis.

No expandir a columnas durante normalization.

---

# 123. Why

Wildcard expansion requiere schema knowledge.

---

# 124. FROM normalization

Una tabla deberá estar representada por:

```text
TableSource(
    QualifiedIdentifier(...)
)
```

no raw string.

---

# 125. Join normalization

Nested Builder representations deberán convertirse a:

```text
JoinClause
```

canónica.

---

# 126. Join order

Se preservará.

---

# 127. CROSS JOIN

No transformar automáticamente entre:

```text
CROSS JOIN
```

y:

```text
INNER JOIN ... TRUE
```

porque mantener intención facilita capabilities y compilation.

---

# 128. WHERE normalization

Todo WHERE será representado por:

```text
Predicate
```

canónico.

---

# 129. GROUP BY

Preservar orden inicialmente.

---

# 130. HAVING

Mismas reglas generales de predicate normalization.

---

# 131. ORDER BY

Canonical entry:

```text
OrderByItem
├── expression
├── direction
└── nullOrdering?
```

---

# 132. Default direction

Si Developer API omite dirección:

```text
orderBy('name')
```

podrá normalizarse a:

```text
ASC
```

si ASC es parte formal del API VoltStack.

---

# 133. Null ordering

No inferir:

```text
NULLS FIRST
NULLS LAST
```

desde un vendor durante normalization.

---

# 134. Pagination

Canonical model:

```text
PaginationClause
├── limit?
├── offset?
└── strategy metadata?
```

---

# 135. LIMIT ALL

No transformar a ausencia de limit salvo que la semántica portable VoltStack lo defina inequívocamente.

---

# 136. Locking

Canonical:

```text
LockClause
├── mode
├── targets
└── waitPolicy
```

No syntax:

```text
FOR UPDATE SKIP LOCKED
```

---

# 137. INSERT normalization

Incluye:

```text
target
columns
source
conflict strategy
returning
metadata
```

---

# 138. Insert source

Forma canónica:

```text
ValuesInsertSource
QueryInsertSource
DefaultValuesInsertSource
```

---

# 139. Row normalization

Cada row tendrá arity estructural consistente.

La validación semántica de tipos vendrá después.

---

# 140. Column ordering

No ordenar insert columns.

```text
INSERT (a, b)
```

no debe convertirse arbitrariamente en:

```text
INSERT (b, a)
```

porque los rows dependen de esa posición.

---

# 141. Conflict normalization

Mantener:

```text
ConflictStrategy
ConflictTarget
ConflictAction
```

sin vendor syntax.

---

# 142. UPDATE normalization

Forma canónica:

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

# 143. Assignment order

Se preservará inicialmente.

---

# 144. Duplicate assignments

```text
SET a = 1, a = 2
```

no deberán ser silenciosamente reducidos.

Preferencia:

```text
structural validation error
```

o semantic error según resolución del identifier.

---

# 145. DELETE normalization

Forma canónica:

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

# 146. Full-table mutations

Normalization no decidirá autorización.

Pero preservará metadata como:

```text
FullTableMutationIntent
```

si Developer API la declaró.

---

# 147. CTE normalization

Cada CTE tendrá:

```text
CteDefinition
├── name
├── columns?
├── query
├── recursive metadata
└── materialization intent?
```

---

# 148. CTE order

Se preservará.

Puede ser relevante para:

- diagnostics;
- dependency analysis;
- recursive semantics.

---

# 149. Recursive CTE

No inferir automáticamente recursive sólo porque detectemos referencia al mismo nombre durante structural normalization.

Eso pertenece a semantic resolution.

---

# 150. Set operations

Canonical node:

```text
SetOperation
├── type
├── left
├── right
└── quantifier
```

---

# 151. Quantifier

Ejemplo:

```text
UNION
```

podrá canonicalizarse a:

```text
UNION DISTINCT
```

si ésa es formalmente la semántica del Query Model.

---

# 152. Preserve parentheses semantics

Set operation grouping deberá permanecer estructuralmente explícito.

---

# 153. Subquery normalization

Toda subquery se normalizará recursivamente.

---

# 154. Correlation

No resolver correlation durante normalization.

---

# 155. Window normalization

Canonical:

```text
WindowSpecification
├── partitionBy
├── orderBy
├── frame?
└── reference?
```

---

# 156. Window frame defaults

No materializar defaults vendor-specific durante normalization.

---

# 157. Function normalization

Function calls deberán usar:

```text
FunctionId
```

o unresolved function reference estructurado.

---

# 158. Function aliases

Si VoltStack define aliases propios inequívocos:

```text
len
length
```

podrán normalizarse a una función semántica común.

Pero no se mapearán funciones de vendor arbitrariamente.

---

# 159. Function resolution

Overloads y tipos pertenecen a Semantic Analysis.

---

# 160. Operator normalization

Builder syntax:

```text
+
-
*
/
%
```

se convierte a:

```text
OperatorId
```

estructurado.

---

# 161. Vendor operators

Operadores como:

```text
@>
?| 
#>>
```

deberán entrar mediante extension nodes/operator IDs explícitos.

---

# 162. No raw operator strings

Excepto escape hatch controlado.

---

# 163. Cast normalization

Canonical:

```text
CastExpression
├── expression
└── target QueryTypeReference
```

No syntax específica.

---

# 164. CASE normalization

Canonical:

```text
CaseExpression
├── operand?
├── branches
└── else?
```

---

# 165. CASE order

Branches nunca deberán reordenarse.

---

# 166. COALESCE

Mantener como semantic function/expression.

No simplificar:

```text
COALESCE(a, a)
```

sin análisis suficiente.

---

# 167. Constant folding

No será responsabilidad general de normalization.

---

# 168. Example

```text
1 + 2
→ 3
```

será preferentemente una optimization rule.

---

# 169. Exception

Canonical literal parsing como:

```text
"001"
→ integer value 1
```

sólo si el input ya fue declarado inequívocamente como integer literal.

---

# 170. Why separate constant folding

Constant folding puede depender de:

- numeric precision;
- overflow;
- collation;
- timezone;
- function volatility;
- platform semantics.

---

# 171. Volatile functions

Normalization nunca evaluará:

```text
NOW()
RANDOM()
UUID()
CURRENT_TIMESTAMP
```

---

# 172. Function volatility

Será metadata semántica posterior.

---

# 173. Canonical boolean groups

Se preferirá un único node por operador lógico.

```text
AndPredicate(children)
OrPredicate(children)
NotPredicate(child)
```

---

# 174. Builder nested closure

Developer API:

```php
->where(function ($q) {
    $q->where('a', 1)
      ->orWhere('b', 2);
})
```

deberá convertirse antes o durante model construction a estructura explícita.

No closures en AST.

---

# 175. Normalizer never executes builder closures

Éstas ya deberán haber sido materializadas.

---

# 176. Structural equality

Después de normalization:

```text
structurally equivalent queries
```

deberán tener alta probabilidad de producir la misma estructura canónica.

---

# 177. But not forced equivalence

VoltStack no intentará resolver todas las equivalencias matemáticas posibles.

---

# 178. Canonicalization boundary

La V1 se centrará en:

```text
representation equivalence
```

más que:

```text
deep semantic equivalence
```

---

# 179. Example

Estas formas sí deberían converger:

```text
AND(A, AND(B))
```

y:

```text
AND(A, B)
```

---

# 180. Pero no necesariamente

```text
NOT(A OR B)
```

y:

```text
NOT(A) AND NOT(B)
```

---

# 181. Normalization levels

Podrán definirse:

```text
BASIC
STANDARD
STRICT
```

---

# 182. BASIC

Sólo invariantes indispensables:

- node shape;
- identifiers;
- operators;
- obvious wrappers.

---

# 183. STANDARD

Default de producción:

- structural canonicalization;
- predicate flattening;
- parameter normalization;
- clause normalization;
- deterministic metadata.

---

# 184. STRICT

Puede ejecutar:

- additional canonical validation;
- extension conformance checks;
- expensive diagnostics;

principalmente desarrollo/testing.

---

# 185. No aggressive mode initially

No introducir:

```text
AGGRESSIVE_NORMALIZATION
```

que se comporte como optimizer.

---

# 186. NormalizationPolicy

Conceptualmente:

```php
final readonly class NormalizationPolicy
{
    public function __construct(
        public NormalizationLevel $level,
        public int $maxPasses,
        public bool $verifyIdempotence,
        public bool $collectDiagnostics,
    ) {}
}
```

---

# 187. Production defaults

Preferencia:

```text
level = STANDARD
verifyIdempotence = false
```

porque la implementación deberá garantizar idempotencia mediante tests.

---

# 188. Development mode

Puede habilitar:

```text
verifyIdempotence = true
```

y ejecutar:

```text
N1 = normalize(Q)
N2 = normalize(N1)

assert N1 == N2
```

---

# 189. Convergence

Un pipeline de reglas debe alcanzar:

```text
fixed point
```

---

# 190. Fixed point

```text
normalize(Q) = Q
```

para una query ya normalizada.

---

# 191. Multi-pass convergence

Algunas reglas pueden habilitar otras.

Ejemplo:

```text
AND(A, AND(B, TRUE))
```

Pass inicial:

```text
AND(A, B, TRUE)
```

siguiente:

```text
AND(A, B)
```

---

# 192. Implementation option

El engine puede aplicar reglas bottom-up para alcanzar el mismo resultado en un único traversal.

---

# 193. Pass limit

Siempre existirá:

```text
maxNormalizationPasses
```

---

# 194. Rewrite limit

También:

```text
maxNormalizationRewrites
```

---

# 195. Non-convergence

Si se supera:

```text
NormalizationDidNotConvergeException
```

---

# 196. Cycle example

Extension Rule A:

```text
X → Y
```

Extension Rule B:

```text
Y → X
```

---

# 197. Cycle detection

Podrá usar:

```text
node fingerprint history
rule history
pass fingerprint
```

---

# 198. Cycle diagnostics

Deberán mostrar:

```text
rule A
rule B
node type
source location
```

sin datos sensibles.

---

# 199. Rule termination requirement

Cada regla deberá declarar o demostrar que:

- reduce una medida estructural;
- canonicaliza hacia una forma estable;
- o forma parte de un pipeline con convergencia demostrada.

---

# 200. Normalization measure

Conceptualmente:

```text
NormalizationMeasure
```

puede considerar:

```text
nested redundant nodes
non-canonical aliases
non-canonical operators
wrapper count
unnormalized metadata
```

---

# 201. No requirement for mathematical proof

Pero las reglas core deberán tener:

- rationale;
- idempotence tests;
- convergence tests.

---

# 202. Rule purity

Las reglas deberán ser preferentemente:

```text
pure
deterministic
side-effect free
```

---

# 203. Forbidden rule dependencies

Una regla no deberá:

- query database;
- inspect current time;
- read random;
- read HTTP request;
- inspect current user;
- mutate global state.

---

# 204. Determinism

Mismos:

```text
Input AST
NormalizationPolicy
ExtensionGraph
```

deberán producir:

```text
same Normalized AST
```

---

# 205. External state

Si una regla necesita información contextual legítima, deberá venir explícitamente en `NormalizationContext`.

---

# 206. Capability-dependent normalization

Debe minimizarse.

---

# 207. Preferred architecture

Capabilities influyen principalmente en:

```text
Semantic Validation
Planning
Compilation
```

No en canonicalización estructural.

---

# 208. Exception

Puede existir una extension normalization rule específica cuando un extension node tiene canonical forms dependientes de una capability declarada.

Debe ser explícito.

---

# 209. Query normalization fingerprint

Podrá existir:

```text
NormalizedQueryFingerprint
```

---

# 210. Fingerprint purpose

Identificar:

```text
canonical query shape
```

antes de Semantic Analysis.

---

# 211. Fingerprint pipeline

```text
Query AST
   │
   ▼
Normalize
   │
   ▼
Normalized AST
   │
   ▼
Canonical Serializer / Hasher
   │
   ▼
NormalizedQueryFingerprint
```

---

# 212. Fingerprint ≠ serialization protocol

No es necesario serializar el AST a JSON para calcular fingerprint.

---

# 213. Hash input

Deberá utilizar:

```text
node kinds
semantic structural values
child ordering where significant
parameter shape
semantic metadata
extension node identity/version
```

---

# 214. Exclude by default

No incluir:

```text
source line
debug label
trace ID
QueryProcessingId
ExecutionId
runtime parameter values
PDO identity
request ID
```

---

# 215. Parameter fingerprint

Puede incluir:

```text
parameter semantic type hint
parameter role
list/scalar shape
nullable metadata
```

según necesidad.

---

# 216. Extension fingerprint

Extension nodes deberán aportar una representación estable.

---

# 217. Versioning

Fingerprint deberá incluir una versión del algoritmo.

Ejemplo:

```text
nqf:v1:...
```

---

# 218. Why

Cambiar reglas de normalization puede cambiar la forma canónica.

---

# 219. Cache implications

Cambiar:

```text
NormalizationVersion
```

puede invalidar:

- semantic cache;
- plan cache;
- compiled query cache.

---

# 220. Separate fingerprints

No crear un único hash universal.

---

# 221. Distinction

```text
QueryModelFingerprint
NormalizedQueryFingerprint
SemanticFingerprint
LogicalPlanFingerprint
CompiledQueryFingerprint
```

son conceptos distintos.

---

# 222. Source locations

Normalización deberá preservar source mapping cuando sea posible.

---

# 223. Node replacement

Si:

```text
AND(A)
→
A
```

los diagnostics podrán conservar provenance del wrapper eliminado.

---

# 224. SourceMap

Podrá existir:

```text
NormalizationSourceMap
```

---

# 225. Provenance

Puede mapear:

```text
normalized node
→
one or more original nodes
```

---

# 226. Why

Útil para:

- error messages;
- debugging;
- developer toolbar;
- query explain;
- tests.

---

# 227. Provenance overhead

Podrá ser configurable.

---

# 228. Production

Puede conservar provenance mínima.

---

# 229. Development

Puede conservar trace detallado.

---

# 230. Rewrite trace

Opcional:

```text
NormalizationTrace
```

---

# 231. Example

```text
Pass 1
Rule: FlattenAndPredicate
Node: P-14
Before: AND(A, AND(B, C))
After: AND(A, B, C)

Pass 1
Rule: RemoveTrueFromAnd
Node: P-14
Before: AND(A, B, TRUE)
After: AND(A, B)
```

---

# 232. Trace ≠ AST mutation log in production

No mantener logs detallados por defecto.

---

# 233. Diagnostics

Podrán producirse:

```text
NORMALIZATION_RULE_APPLIED
NORMALIZATION_FALLBACK
NORMALIZATION_EXTENSION_WARNING
NORMALIZATION_NON_CANONICAL_INPUT
NORMALIZATION_BUDGET_WARNING
```

---

# 234. Security

Diagnostics nunca deberán mostrar parameter values sensibles por defecto.

---

# 235. Query normalization and raw SQL

`RawSqlQuery` no será parseado automáticamente a AST.

---

# 236. Raw SQL path

```text
RawSqlQuery
    │
    ├── metadata normalization
    ├── parameter descriptor normalization
    └── security validation
```

pero no:

```text
SQL parser
→ AST
```

en V1.

---

# 237. Reason

Implementar un parser SQL multi-dialect introduciría:

- enorme complejidad;
- ambiguities;
- dialect coupling;
- security surface.

---

# 238. Future SQL parser

Podría existir como paquete separado.

No será requisito del Query Engine base.

---

# 239. Raw query fingerprint

Puede usar:

```text
raw SQL shape
+
binding descriptors
+
declared target metadata
```

con reglas independientes.

---

# 240. Policy transformations

La normalización deberá ocurrir después de transformaciones de policy que cambien la estructura.

---

# 241. Example tenant policy

```text
SELECT users
```

puede transformarse a:

```text
SELECT users
WHERE tenant_id = :tenant
```

antes de normalización.

---

# 242. Important

Multitenancy no será dependencia del Query Normalizer.

---

# 243. Generic transformation pipeline

```text
Query AST
   │
   ▼
Policy Transformation Ports
   │
   ▼
Transformed AST
   │
   ▼
Normalization
```

---

# 244. Why normalize after policies

Las policies pueden introducir:

```text
nested AND
new parameters
new predicates
new joins
new metadata
```

que deben canonicalizarse.

---

# 245. Re-normalization

Si una transformación posterior modifica el AST, deberá producir una nueva fase de normalization explícita.

---

# 246. No normalized flag mutation

No:

```text
$ast->normalized = false;
```

---

# 247. Artifact transition

Preferir:

```text
NormalizedQueryAst
   │
   ▼
Transformation
   │
   ▼
QueryAst
   │
   ▼
Normalize again
```

---

# 248. Semantic transformations

Después de Semantic Analysis pueden existir rewrites basados en información semántica.

Éstos no pertenecen al Query Normalization System inicial.

---

# 249. Semantic rewrite layer

Conceptualmente:

```text
Normalized AST
   │
   ▼
Semantic Analysis
   │
   ▼
Semantic Graph
   │
   ▼
Semantic Rewrites / Optimizer
```

---

# 250. Normalization vs optimization matrix

| Operación | Normalization | Optimizer |
|---|---:|---:|
| Flatten nested AND | Sí | Puede asumirlo |
| Remove AND(TRUE) | Sí | Puede asumirlo |
| Canonical operator enum | Sí | No |
| Quote identifier | No | No |
| Resolve column | No | No |
| Reorder joins | No | Sí |
| Predicate pushdown | No | Sí |
| Constant folding | Limitado/No | Sí |
| Duplicate query elimination | No | Sí |
| Vendor strategy selection | No | Planner |
| SQL syntax generation | No | Compiler |

---

# 251. Normalization vs semantic matrix

| Operación | Normalization | Semantic |
|---|---:|---:|
| Flatten predicate groups | Sí | No |
| Normalize operator IDs | Sí | No |
| Resolve table alias | No | Sí |
| Resolve column type | No | Sí |
| Validate function overload | No | Sí |
| Validate comparison compatibility | No | Sí |
| Determine correlation | No | Sí |
| Determine aggregate scope | No | Sí |
| Validate GROUP BY semantics | No | Sí |

---

# 252. Normalization vs compilation matrix

| Operación | Normalization | Compiler |
|---|---:|---:|
| Canonical predicate tree | Sí | No |
| Allocate placeholders | No | Sí |
| Quote identifiers | No | Sí |
| Generate SQL keywords | No | Sí |
| Generate vendor UPSERT | No | Sí |
| Emit RETURNING syntax | No | Sí |
| Format SQL | No | Sí |

---

# 253. Extension architecture

Third-party extensions podrán registrar:

```text
NormalizationRule
```

para sus propios node types.

---

# 254. Extension rule scope

Preferencia:

> Una extensión deberá normalizar los nodes que introduce, no alterar arbitrariamente todo el core AST.

---

# 255. Core node interception

Sólo mediante extension point explícito.

---

# 256. Reason

Evita que un paquete modifique silenciosamente la semántica global.

---

# 257. Extension descriptor

Conceptualmente:

```text
NormalizationExtensionDescriptor
├── extensionId
├── version
├── supportedNodeTypes
├── rules
├── ordering
├── dependencies
└── fingerprintContribution
```

---

# 258. Extension compatibility

Debe declarar versión compatible de:

```text
Query AST API
Normalization API
```

---

# 259. Extension rule stability

Cambiar una regla de canonicalization puede requerir cambiar:

```text
extension normalization version
```

---

# 260. Extension failure

Una extensión requerida que falla al normalizar no deberá ser ignorada silenciosamente.

---

# 261. Observational extension

Un observer de diagnostics puede fallar según policy sin cambiar semantics.

---

# 262. Semantic extension

Una normalization rule sí afecta artifacts y su failure es relevante.

---

# 263. Rule conflict

Si dos reglas intentan canonicalizar el mismo node de formas incompatibles:

```text
NormalizationRuleConflictException
```

durante bootstrap cuando sea detectable.

---

# 264. Runtime conflict

Si sólo puede detectarse durante una query:

```text
NormalizationConflictException
```

---

# 265. NormalizationBudget

Definido desde Query Context.

---

# 266. Budget fields

Conceptualmente:

```text
maxNodes
maxDepth
maxPasses
maxRewrites
maxRuleApplications
maxExtensionRewrites
maxTemporaryNodes
```

---

# 267. Optional wall clock

```text
deadline
```

como defensa adicional.

---

# 268. Complexity accounting

El engine deberá contar trabajo real de forma razonablemente barata.

---

# 269. Avoid attacker-controlled explosion

Una regla no deberá expandir:

```text
N nodes
→ 2^N nodes
```

sin límites explícitos.

---

# 270. Expansion rules

Cualquier regla expansiva deberá declarar:

```text
ExpansionPolicy
```

o estar reservada para optimizer/planner.

---

# 271. Normalization should usually reduce or preserve size

Regla general:

```text
normalized node count
<=
reasonable function of input node count
```

---

# 272. Query complexity

Queries generadas dinámicamente pueden ser enormes incluso sin SQL injection.

Por tanto, normalization es una frontera útil para resource governance.

---

# 273. Persistent runtime safety

Normalizer services compartidos deberán ser:

- stateless;
- immutable;
- reentrant;
- concurrency-safe.

---

# 274. Shared services

Pueden sobrevivir:

```text
Normalizer
FrozenRuleRegistry
RuleDescriptors
CanonicalizationPolicies
```

---

# 275. Operation-local state

No puede sobrevivir:

```text
NormalizationState
visited nodes
rewrite history
memoization
trace
source map builder
```

---

# 276. FrankenPHP

```text
Worker
├── shared Normalizer
│
├── Request A
│   ├── NormalizationState A1
│   └── NormalizationState A2
│
└── Request B
    └── NormalizationState B1
```

---

# 277. OpenSwoole

Dos coroutines pueden ejecutar el mismo Normalizer simultáneamente.

---

# 278. Invariant

No habrá:

```text
Normalizer::$currentNode
Normalizer::$currentPass
Normalizer::$currentContext
```

---

# 279. Thread/coroutine safety

Toda mutable operation state se pasa explícitamente o vive en objeto local.

---

# 280. Performance model

Normalization estará en el hot path.

---

# 281. Performance priorities

1. correctness;
2. deterministic output;
3. bounded complexity;
4. low allocation;
5. minimal traversals;
6. cache friendliness.

---

# 282. Avoid premature micro-optimization

Primero:

```text
clear canonical model
```

después:

```text
specialized visitors
iterative traversal
structural sharing
node interning
compiled rule dispatch
```

si benchmarks lo justifican.

---

# 283. Structural sharing

Como AST es immutable, transformations podrán reutilizar nodes sin cambios.

---

# 284. Example

Si sólo cambia un predicate:

```text
SelectQueryNode
├── Projection      reused
├── Source          reused
├── Predicate       replaced
└── OrderBy         reused
```

---

# 285. Copy-on-write semantics

No es mutable copy-on-write tradicional.

Es:

```text
immutable structural reuse
```

---

# 286. Identity preservation

Un node no modificado podrá conservar su `AstNodeId` si la semántica de identidad del documento 26 lo permite.

---

# 287. Rewritten node identity

Un node nuevo deberá recibir identidad según política AST.

---

# 288. Fingerprint independent of incidental NodeId

`AstNodeId` normalmente no participará en structural fingerprint.

---

# 289. Memoization

Podrá memoizar:

```text
original node identity
→ normalized node
```

dentro de una operación.

---

# 290. Shared subtree

Si el AST permite structural sharing, memoization evita normalizar el mismo subtree varias veces.

---

# 291. Cycles

AST normal deberá ser acíclico.

---

# 292. Cycle detection

Si una extension produce un ciclo:

```text
AstCycleDetectedException
```

antes o durante normalization.

---

# 293. Testing architecture

Se requerirán:

```text
Rule Unit Tests
Idempotence Tests
Convergence Tests
Determinism Tests
Property-Based Tests
Golden AST Tests
Extension Tests
Budget Tests
Persistent Runtime Tests
Concurrency Tests
Fingerprint Tests
Source Mapping Tests
Architecture Tests
```

---

# 294. Rule unit test

Ejemplo:

```text
Input:
AND(A, AND(B, C))

Expected:
AND(A, B, C)
```

---

# 295. Idempotence test

Para cada fixture:

```text
N1 = normalize(Q)
N2 = normalize(N1)

assert N1 == N2
```

---

# 296. Determinism test

```text
normalize(Q, C)
```

ejecutado múltiples veces produce mismo structural fingerprint.

---

# 297. Property-based tests

Generar ASTs válidos y comprobar:

```text
normalize(normalize(Q))
=
normalize(Q)
```

---

# 298. Semantic preservation tests

Cuando exista evaluator/DB conformance:

```text
execute(Q)
```

y:

```text
execute(normalize(Q))
```

deberán producir resultados equivalentes para reglas aplicables.

---

# 299. Multi-platform tests

Especialmente para reglas que podrían interactuar con:

```text
NULL
boolean logic
numeric semantics
collation
```

---

# 300. Three-valued logic tests

Críticos para predicate simplification.

---

# 301. Golden AST tests

Fixtures legibles:

```text
input AST
expected normalized AST
```

---

# 302. Fingerprint tests

Queries equivalentes por representación deben producir el mismo fingerprint cuando la especificación diga que son canonicalmente equivalentes.

---

# 303. Negative fingerprint tests

Queries cuya diferencia sea observable deberán producir fingerprints distintos.

---

# 304. Example

```text
SELECT a, b
```

vs:

```text
SELECT b, a
```

deben permanecer distintos.

---

# 305. Extension conformance

Toda normalization extension deberá pasar:

```text
NormalizationExtensionConformanceSuite
```

---

# 306. Conformance requirements

- deterministic;
- idempotent;
- bounded;
- no I/O;
- no globals;
- stable fingerprint contribution;
- lifecycle safe;
- compatible ordering;
- no illegal core mutation.

---

# 307. Debug mode

Podrá ofrecer:

```text
database:query:normalize
```

conceptualmente.

---

# 308. Example CLI

```text
Input Query Model
        │
        ▼
Raw AST
        │
        ▼
Normalization Trace
        │
        ▼
Normalized AST
```

---

# 309. CLI safety

Parameter values sensibles deberán redactarse.

---

# 310. Explain normalization

Podrá mostrar:

```text
rules applied
nodes changed
passes
rewrite count
final fingerprint
warnings
```

---

# 311. Telemetry

Métricas posibles:

```text
database.query.normalization.duration
database.query.normalization.nodes
database.query.normalization.rewrites
database.query.normalization.passes
database.query.normalization.failures
```

---

# 312. Avoid high-cardinality labels

No utilizar query fingerprint completo como metric label por defecto.

---

# 313. Tracing

Puede existir span:

```text
database.query.normalize
```

mediante Telemetry adapter.

---

# 314. No direct Telemetry dependency

Query Normalizer dependerá de:

```text
DiagnosticSink / Observer Port
```

si necesita observabilidad.

---

# 315. Error hierarchy

Conceptualmente:

```text
QueryNormalizationException
├── InvalidNormalizationInputException
├── NormalizationRuleException
├── NormalizationRuleConflictException
├── NormalizationDidNotConvergeException
├── NormalizationCycleDetectedException
├── NormalizationBudgetExceededException
├── NormalizationExtensionException
├── InvalidNormalizedAstException
└── UnsupportedNormalizationNodeException
```

---

# 316. Error source

Cuando sea posible:

```text
node type
node id
source location
rule id
phase
```

---

# 317. No sensitive values

Errors no incluirán runtime parameter values salvo safe debug policy explícita.

---

# 318. Suggested namespace

```text
VoltStack\Quantum\Database\Query\Normalization\
```

---

# 319. Proposed structure

```text
Query/
└── Normalization/
    ├── Contract/
    │   ├── QueryNormalizerInterface.php
    │   ├── NormalizationRuleInterface.php
    │   ├── NormalizationRuleRegistryInterface.php
    │   └── NormalizationValidatorInterface.php
    │
    ├── Core/
    │   ├── QueryNormalizer.php
    │   ├── NormalizationPipeline.php
    │   ├── NormalizationPass.php
    │   ├── NormalizationResult.php
    │   └── NormalizedQueryAst.php
    │
    ├── Context/
    │   ├── NormalizationContext.php
    │   ├── NormalizationRuleContext.php
    │   └── NormalizationPolicy.php
    │
    ├── State/
    │   ├── NormalizationState.php
    │   ├── NormalizationPassState.php
    │   ├── RewriteCounter.php
    │   └── ConvergenceState.php
    │
    ├── Rule/
    │   ├── Structural/
    │   ├── Identifier/
    │   ├── Source/
    │   ├── Expression/
    │   ├── Predicate/
    │   ├── Parameter/
    │   ├── Clause/
    │   └── Metadata/
    │
    ├── Predicate/
    │   ├── FlattenAndPredicateRule.php
    │   ├── FlattenOrPredicateRule.php
    │   ├── RemoveSingleAndRule.php
    │   ├── RemoveSingleOrRule.php
    │   ├── RemoveDoubleNegationRule.php
    │   ├── SimplifyBooleanAndRule.php
    │   └── SimplifyBooleanOrRule.php
    │
    ├── Identifier/
    │   ├── IdentifierNormalizer.php
    │   ├── QualifiedIdentifierNormalizer.php
    │   └── AliasNormalizer.php
    │
    ├── Parameter/
    │   ├── QueryParameterNormalizer.php
    │   ├── ParameterShapeNormalizer.php
    │   └── ParameterIdentityNormalizer.php
    │
    ├── Clause/
    │   ├── SelectClauseNormalizer.php
    │   ├── InsertClauseNormalizer.php
    │   ├── UpdateClauseNormalizer.php
    │   ├── DeleteClauseNormalizer.php
    │   ├── JoinNormalizer.php
    │   ├── CteNormalizer.php
    │   ├── SetOperationNormalizer.php
    │   ├── OrderingNormalizer.php
    │   ├── PaginationNormalizer.php
    │   └── LockingNormalizer.php
    │
    ├── Convergence/
    │   ├── ConvergenceDetector.php
    │   ├── RewriteCycleDetector.php
    │   └── NormalizationMeasure.php
    │
    ├── Budget/
    │   ├── NormalizationBudget.php
    │   └── NormalizationBudgetTracker.php
    │
    ├── Fingerprint/
    │   ├── NormalizedQueryFingerprint.php
    │   ├── NormalizedQueryFingerprintFactory.php
    │   └── NormalizationVersion.php
    │
    ├── Provenance/
    │   ├── NormalizationSourceMap.php
    │   ├── NormalizationTrace.php
    │   └── RewriteRecord.php
    │
    ├── Extension/
    │   ├── NormalizationExtensionDescriptor.php
    │   ├── NormalizationExtensionRegistry.php
    │   └── NormalizationExtensionConformanceSuite.php
    │
    ├── Diagnostic/
    │   ├── NormalizationDiagnostic.php
    │   └── NormalizationDiagnosticRenderer.php
    │
    └── Exception/
        ├── QueryNormalizationException.php
        ├── NormalizationRuleException.php
        ├── NormalizationRuleConflictException.php
        ├── NormalizationDidNotConvergeException.php
        ├── NormalizationCycleDetectedException.php
        ├── NormalizationBudgetExceededException.php
        └── InvalidNormalizedAstException.php
```

---

# 320. Dependency direction

```text
Query AST
    │
    ▼
Normalization
    │
    ├── AST Contracts
    ├── Expression Contracts
    ├── Predicate Contracts
    ├── Parameter Contracts
    ├── Query Type IDs
    ├── Metadata Contracts
    └── Query Context
    │
    ▼
Normalized Query AST
```

---

# 321. Forbidden dependencies

Normalization no deberá depender de:

```text
PDO
NativeConnection
ConnectionLease
Driver
EntityManager
UnitOfWork
IdentityMap
Repository
Hydrator
HTTP Request
Authentication
Authorization Manager
Tenant Entity
Service Container
```

---

# 322. Dialect dependency

Core normalization deberá ser dialect-neutral.

---

# 323. Platform dependency

Core normalization deberá ser platform-neutral.

---

# 324. Capability dependency

Sólo cuando una extension rule explícitamente lo requiera y esté arquitectónicamente justificado.

---

# 325. DB-QNORM-001

Toda query estructurada deberá pasar por normalization antes de Semantic Analysis.

---

# 326. DB-QNORM-002

Normalization deberá producir una forma canónica.

---

# 327. DB-QNORM-003

Normalization será determinista.

---

# 328. DB-QNORM-004

Normalization será idempotente.

---

# 329. DB-QNORM-005

Normalization deberá converger.

---

# 330. DB-QNORM-006

El número de passes estará limitado.

---

# 331. DB-QNORM-007

El número de rewrites estará limitado.

---

# 332. DB-QNORM-008

Normalization no ejecutará SQL.

---

# 333. DB-QNORM-009

Normalization no generará SQL.

---

# 334. DB-QNORM-010

Normalization no abrirá conexiones.

---

# 335. DB-QNORM-011

Normalization no consultará el servidor DB implícitamente.

---

# 336. DB-QNORM-012

Normalization no resolverá current tenant.

---

# 337. DB-QNORM-013

Normalization no resolverá current user.

---

# 338. DB-QNORM-014

Normalization no utilizará Service Locator.

---

# 339. DB-QNORM-015

Normalization no utilizará estado global mutable.

---

# 340. DB-QNORM-016

Normalizer compartido será stateless y reentrant.

---

# 341. DB-QNORM-017

NormalizationState será operation-local.

---

# 342. DB-QNORM-018

AST input no será mutado.

---

# 343. DB-QNORM-019

Las transformations producirán nuevos nodes cuando sea necesario.

---

# 344. DB-QNORM-020

Subtrees no modificados podrán reutilizarse.

---

# 345. DB-QNORM-021

Node IDs incidentales no definirán structural fingerprint.

---

# 346. DB-QNORM-022

Normalization preservará orden cuando sea observable.

---

# 347. DB-QNORM-023

Projection order será preservado.

---

# 348. DB-QNORM-024

Insert column order será preservado.

---

# 349. DB-QNORM-025

CASE branch order será preservado.

---

# 350. DB-QNORM-026

JOIN order será preservado por normalization.

---

# 351. DB-QNORM-027

CTE order será preservado inicialmente.

---

# 352. DB-QNORM-028

Predicate order será preservado inicialmente.

---

# 353. DB-QNORM-029

Join reordering pertenecerá al Optimizer.

---

# 354. DB-QNORM-030

Predicate pushdown pertenecerá al Optimizer.

---

# 355. DB-QNORM-031

General constant folding pertenecerá al Optimizer.

---

# 356. DB-QNORM-032

Platform strategy selection pertenecerá al Planner.

---

# 357. DB-QNORM-033

Dialect syntax generation pertenecerá al Compiler.

---

# 358. DB-QNORM-034

Identifier quoting pertenecerá al Compiler/Dialect.

---

# 359. DB-QNORM-035

Placeholder allocation pertenecerá al Compiler.

---

# 360. DB-QNORM-036

Parameter values no serán interpolados.

---

# 361. DB-QNORM-037

Parameter identity permanecerá independiente del placeholder SQL.

---

# 362. DB-QNORM-038

Builder operator aliases deberán converger a operator IDs canónicos.

---

# 363. DB-QNORM-039

Raw operator strings no formarán parte del core AST.

---

# 364. DB-QNORM-040

Identifiers permanecerán estructurados.

---

# 365. DB-QNORM-041

Identifiers no serán quoted durante normalization.

---

# 366. DB-QNORM-042

Identifier case no será destruido sin conocimiento semántico suficiente.

---

# 367. DB-QNORM-043

Wildcard no será expandido durante normalization.

---

# 368. DB-QNORM-044

Function overloads no serán resueltos durante normalization.

---

# 369. DB-QNORM-045

Column references no serán resueltas durante normalization.

---

# 370. DB-QNORM-046

Table references no serán semánticamente resueltas durante normalization.

---

# 371. DB-QNORM-047

Correlation no será resuelta durante normalization.

---

# 372. DB-QNORM-048

Three-valued SQL logic deberá respetarse.

---

# 373. DB-QNORM-049

Boolean simplifications deberán demostrar compatibilidad con la semántica predicate.

---

# 374. DB-QNORM-050

De Morgan no será una normalization universal.

---

# 375. DB-QNORM-051

BETWEEN mantendrá su intención estructural.

---

# 376. DB-QNORM-052

EXISTS mantendrá su intención estructural.

---

# 377. DB-QNORM-053

LIKE mantendrá su intención estructural.

---

# 378. DB-QNORM-054

UPSERT mantendrá intención semántica vendor-neutral.

---

# 379. DB-QNORM-055

RETURNING mantendrá intención semántica.

---

# 380. DB-QNORM-056

Locking mantendrá intención semántica.

---

# 381. DB-QNORM-057

Pagination permanecerá vendor-neutral.

---

# 382. DB-QNORM-058

Set operations mantendrán grouping explícito.

---

# 383. DB-QNORM-059

Subqueries serán normalizadas recursivamente.

---

# 384. DB-QNORM-060

AST cycles serán rechazados.

---

# 385. DB-QNORM-061

Extension rules serán registradas antes de freeze.

---

# 386. DB-QNORM-062

No habrá runtime registration por defecto.

---

# 387. DB-QNORM-063

Rule ordering será determinista.

---

# 388. DB-QNORM-064

Rule conflicts no utilizarán last-wins silencioso.

---

# 389. DB-QNORM-065

Extension rules deberán ser idempotentes.

---

# 390. DB-QNORM-066

Extension rules deberán ser deterministas.

---

# 391. DB-QNORM-067

Extension rules no realizarán I/O oculto.

---

# 392. DB-QNORM-068

Extension rules deberán respetar NormalizationBudget.

---

# 393. DB-QNORM-069

NormalizedQueryFingerprint tendrá versionado.

---

# 394. DB-QNORM-070

Runtime parameter values normalmente no participarán en QueryShapeFingerprint.

---

# 395. DB-QNORM-071

Diagnostic metadata normalmente no participará en semantic fingerprint.

---

# 396. DB-QNORM-072

Source locations no participarán normalmente en structural fingerprint.

---

# 397. DB-QNORM-073

Normalization provenance podrá preservarse separadamente.

---

# 398. DB-QNORM-074

Detailed rewrite traces serán opcionales.

---

# 399. DB-QNORM-075

Production path evitará detailed traces por defecto.

---

# 400. DB-QNORM-076

Normalization errors estarán redacted.

---

# 401. DB-QNORM-077

Normalization será compatible con offline processing.

---

# 402. DB-QNORM-078

Normalization será segura para FrankenPHP.

---

# 403. DB-QNORM-079

Normalization será segura para RoadRunner.

---

# 404. DB-QNORM-080

Normalization será segura para OpenSwoole.

---

# 405. DB-QNORM-081

Dos normalization operations concurrentes no compartirán mutable state.

---

# 406. DB-QNORM-082

Normalized artifacts no contendrán request-scoped runtime resources.

---

# 407. DB-QNORM-083

Policy transformations estructurales ocurrirán antes de normalization.

---

# 408. DB-QNORM-084

Una transformación posterior que invalide canonical form requerirá renormalization explícita.

---

# 409. DB-QNORM-085

No se utilizará un mutable `normalized` flag como única garantía arquitectónica.

---

# 410. DB-QNORM-086

Normalized AST será distinguible conceptualmente del raw AST.

---

# 411. DB-QNORM-087

Structural validation deberá verificar el artifact normalizado.

---

# 412. DB-QNORM-088

Semantic Analysis podrá asumir invariantes documentados del Normalized AST.

---

# 413. DB-QNORM-089

Optimizer podrá asumir invariantes documentados del Semantic Artifact.

---

# 414. DB-QNORM-090

Compiler nunca deberá compensar silenciosamente por AST no normalizado.

---

# 415. DB-QNORM-091

Normalization deberá evitar primitive obsession donde existan Value Objects apropiados.

---

# 416. DB-QNORM-092

Normalization no convertirá AST estructurado en raw SQL.

---

# 417. DB-QNORM-093

Raw SQL no será parseado automáticamente por el normalizador base.

---

# 418. DB-QNORM-094

NormalizationBudget será configurable pero bounded por secure defaults.

---

# 419. DB-QNORM-095

Queries patológicas fallarán de manera controlada antes de agotar el proceso.

---

# 420. DB-QNORM-096

El pipeline podrá optimizar traversals internamente sin cambiar las fronteras arquitectónicas.

---

# 421. DB-QNORM-097

Canonical output no dependerá de hash-map iteration order.

---

# 422. DB-QNORM-098

Canonical output no dependerá de extension discovery order accidental.

---

# 423. DB-QNORM-099

Canonical output no dependerá de current time/random/process ID.

---

# 424. DB-QNORM-100

Correctness y semantic preservation tendrán prioridad sobre canonicalization agresiva.

---

# 425. Anti-pattern — normalization as optimizer

Incorrecto:

```text
Normalizer
├── join reordering
├── index selection
├── cost estimation
└── predicate pushdown
```

---

# 426. Anti-pattern — normalization as compiler

Incorrecto:

```text
Normalizer
→ quote identifiers
→ create placeholders
→ generate SQL
```

---

# 427. Anti-pattern — vendor checks

Incorrecto:

```php
if ($platform === 'mysql') {
    return $this->normalizeForMySql($query);
}
```

---

# 428. Anti-pattern — mutable AST

Incorrecto:

```php
$node->children = $normalizedChildren;
```

---

# 429. Anti-pattern — infinite extension rewrites

```text
A → B
B → A
```

sin detection/budget.

---

# 430. Anti-pattern — sort everything

Ordenar indiscriminadamente:

```text
projection
joins
CTEs
assignments
predicates
```

para obtener hashes iguales.

Eso puede cambiar comportamiento observable.

---

# 431. Anti-pattern — parameter-value fingerprint

```text
WHERE id = 100
```

y:

```text
WHERE id = 101
```

no deberían necesariamente crear compiled query shapes diferentes.

---

# 432. Anti-pattern — raw SQL canonicalizer

Intentar hacer:

```text
regex SQL normalization
```

para fingir que raw SQL tiene un AST fiable.

---

# 433. Anti-pattern — hidden schema lookup

Normalization no debe preguntar:

```text
Does users.name exist?
```

---

# 434. Anti-pattern — application policy lookup

Normalization no debe llamar:

```text
Auth
TenantManager
AuthorizationManager
```

---

# 435. Canonicalization formula

```text
CanonicalQuery
=
Normalize(
    StructuredQuery,
    StableNormalizationPolicy,
    FrozenRuleSet
)
```

---

# 436. Idempotence formula

```text
N(N(Q))
=
N(Q)
```

---

# 437. Determinism formula

Para:

```text
Q1 = Q2
Policy1 = Policy2
Rules1 = Rules2
```

debe cumplirse:

```text
N(Q1)
=
N(Q2)
```

---

# 438. Convergence formula

Existe un número finito `k` tal que:

```text
N^k(Q)
=
N^(k+1)(Q)
```

dentro de los límites permitidos.

---

# 439. Semantic preservation formula

```text
Meaning(N(Q))
=
Meaning(Q)
```

para toda transformation considerada normalization válida.

---

# 440. Fingerprint formula

```text
NormalizedQueryFingerprint
=
Hash(
    NormalizationVersion
    +
    CanonicalNodeStructure
    +
    SemanticStructuralMetadata
    +
    ParameterShape
    +
    RelevantExtensionVersions
)
```

---

# 441. Persistent-safe formula

```text
Persistent-Safe Normalization
=
Stateless Normalizer
+
Frozen Rules
+
Immutable AST
+
Operation-Scoped State
+
Bounded Rewrites
+
No Global State
+
No Hidden I/O
```

---

# 442. Arquitectura final

```text
                     Query AST
                         │
                         ▼
                Initial Validation
                         │
                         ▼
                Policy Transformations
                         │
                         ▼
              NormalizationContext
                         │
                         ▼
               Normalization Engine
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
 Structural Rules   Predicate Rules  Expression Rules
        │                │                │
        ├────────────────┼────────────────┤
        ▼                ▼                ▼
 Identifier Rules   Parameter Rules   Clause Rules
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                  Extension Rules
                         │
                         ▼
                Convergence Check
                         │
                         ▼
                Canonical Validation
                         │
                         ▼
               Normalized Query AST
                         │
                         ├── Fingerprint
                         ├── Source Map
                         └── Diagnostics
                         │
                         ▼
               Structural Validation
                         │
                         ▼
                 Semantic Analysis
```

---

# 443. Invariantes que Semantic Analysis podrá asumir

Después de normalization:

1. logical groups no tendrán nesting redundante equivalente;
2. operator aliases estarán canonicalizados;
3. identifiers tendrán representación estructurada;
4. optional clauses tendrán representación consistente;
5. parameters tendrán shape estructural consistente;
6. subqueries estarán normalizadas;
7. CTEs estarán estructuralmente normalizadas;
8. set operations tendrán grouping explícito;
9. query clauses usarán Value Objects/Nodes canónicos;
10. extension nodes habrán pasado sus normalizers registrados;
11. AST permanecerá immutable;
12. el artifact habrá convergido;
13. normalization budgets habrán sido respetados;
14. el fingerprint podrá calcularse determinísticamente.

---

# 444. Qué NO podrá asumir Semantic Analysis

Normalization todavía no garantiza:

```text
table exists
column exists
alias resolves
types are compatible
function exists
operator exists for effective types
join relation is valid
GROUP BY is semantically valid
capability is supported
RETURNING is available
lock mode is supported
schema object exists
```

Eso pertenece al siguiente nivel.

---

# 445. Relación con documentos anteriores

```text
23_DATABASE_QUERY_ARCHITECTURE.md
            │
            ▼
24_DATABASE_QUERY_MODEL.md
            │
            ▼
25_DATABASE_QUERY_AST_SYSTEM.md
            │
            ▼
26_DATABASE_QUERY_AST_NODE_MODEL.md
            │
            ├── 27 Expression System
            ├── 28 Predicate System
            ├── 29 Parameter & Binding
            ├── 30 Query Type System
            ├── 31 Query Metadata
            └── 32 Query Context
                    │
                    ▼
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
```

---

# 446. Relación con las siguientes fases

```text
Query AST
   │
   ▼
33 Normalization
   │
   ▼
34 Validation
   │
   ▼
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
39 Type Inference
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

---

# 447. Architectural checkpoint

Con este documento el Query Engine ya posee:

```text
Query Model
        │
        ▼
AST
        │
        ├── Nodes
        ├── Expressions
        ├── Predicates
        ├── Parameters
        ├── Types
        ├── Metadata
        └── Context
        │
        ▼
Normalization
        │
        ▼
Canonical AST
```

Esto crea la frontera necesaria para que las siguientes fases trabajen sobre una representación estable.

---

# 448. Decisión final

VoltStack adoptará:

```text
Conservative Canonical Normalization
```

en lugar de:

```text
Aggressive Semantic Rewriting
```

como comportamiento base.

La división será:

```text
Normalization
→ canonical representation

Semantic Analysis
→ meaning

Optimizer
→ equivalent improvement

Planner
→ execution strategy

Compiler
→ dialect-specific SQL

Executor
→ runtime execution
```

---

# 449. Regla maestra final

> El Query Normalization System no intentará descubrir la mejor consulta ni la sintaxis correcta para un motor concreto; su responsabilidad será convertir una consulta estructurada en una representación canónica, determinista e idempotente cuya intención permanezca intacta.

---

# 450. Resultado arquitectónico

El sistema resultante proporciona:

- AST canónico;
- fingerprints estables;
- menor complejidad para Semantic Analysis;
- reglas de optimización más simples;
- compiled-query caching más fiable;
- extensiones deterministas;
- protección contra rewrites infinitos;
- aislamiento de estado;
- compatibilidad con persistent workers;
- procesamiento offline;
- mejor debugging;
- mejor testing;
- separación estricta entre canonicalization, semantics, optimization, planning y compilation.

La propiedad central será:

```text
Different Representation
        │
        ▼
Same Semantic Intent
        │
        ▼
Normalization
        │
        ▼
Same Canonical Representation
```

siempre que la equivalencia pueda establecerse de forma segura sin introducir conocimiento que pertenezca a fases posteriores.

---

# 451. Próximo documento

El siguiente documento será:

```text
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
```

y cerrará el bloque fundamental:

```text
23–34 Query Model and AST
```

antes de comenzar:

```text
35–42 Semantic Query Engine
```

La siguiente frontera será:

```text
Normalized Query AST
        │
        ▼
Validation Pipeline
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