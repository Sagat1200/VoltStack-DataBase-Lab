# 299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Custom Query Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 299 — Custom Query Extension System  
**Bloque:** 30 — Extensibility  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `298_DATABASE_CUSTOM_COMPILER_SYSTEM.md`  
**Siguiente documento:** `300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura oficial mediante la cual VoltStack Database permitirá ampliar su lenguaje de consultas con nuevas construcciones semánticas sin modificar directamente el núcleo de Query Engine y sin utilizar `RawExpression` como mecanismo principal de extensibilidad.

El sistema permitirá incorporar:

```text
custom expressions
custom predicates
custom operators
custom functions
custom aggregates
custom window expressions
custom clauses
custom ordering semantics
custom projections
custom search operations
custom JSON operations
custom spatial operations
custom vector operations
future database-specific query constructs
```

manteniendo el pipeline:

```text
Developer API
    ↓
Query Builder
    ↓
Query Model / AST
    ↓
Semantic Analysis
    ↓
Optimizer
    ↓
Planner
    ↓
Compiler
    ↓
Dialect
    ↓
Execution Engine
```

La regla central será:

> **Una Custom Query Extension deberá ampliar el modelo semántico de Query Engine mediante estructuras tipadas, validables, analizables, optimizables, planificables y compilables. Raw SQL continuará existiendo como escape hatch explícito, pero no constituirá el mecanismo principal para incorporar nuevas capacidades al lenguaje de consultas de VoltStack.**

Formalmente:

```text
Custom Query Extension
=
Stable Identity
+
Semantic Model
+
AST Integration
+
Builder Integration
+
Validation Rules
+
Type Semantics
+
Capability Requirements
+
Optimizer Contracts
+
Planner Contracts
+
Compiler Contracts
+
Dialect Representation
+
Security Policy
+
Diagnostics
+
Testing
```

---

# 2. Objetivos

El sistema deberá permitir:

1. extender Query Engine sin modificar su núcleo;
2. crear nodos AST personalizados;
3. crear expresiones personalizadas;
4. crear predicates personalizados;
5. registrar operadores;
6. registrar funciones;
7. registrar agregados;
8. registrar expresiones window;
9. incorporar cláusulas especializadas;
10. integrar extensiones con Query Builder;
11. mantener tipado;
12. mantener semantic analysis;
13. mantener optimización;
14. mantener planificación;
15. mantener compilation;
16. declarar capabilities requeridas;
17. soportar múltiples dialectos;
18. declarar portabilidad;
19. soportar emulación explícita;
20. evitar SQL injection;
21. detectar conflictos entre extensiones;
22. producir diagnostics;
23. soportar plugins;
24. funcionar en runtimes persistentes;
25. permitir testing/conformance;
26. mantener determinismo;
27. preservar compatibilidad futura.

---

# 3. Problema que resuelve

Sin un sistema formal de extensión, una aplicación podría terminar implementando características específicas mediante:

```php
$query->whereRaw(
    'embedding <-> ? < ?',
    [$vector, $distance]
);
```

Esto introduce varios problemas:

```text
semantic meaning lost
type information lost
capability information lost
optimizer visibility lost
planner visibility lost
portability lost
security becomes harder
diagnostics become weaker
compiler cannot reason about operation
```

VoltStack deberá permitir expresar lo mismo como una operación semántica.

Ejemplo conceptual:

```php
$query->where(
    VectorDistance::between(
        column: 'embedding',
        vector: $vector,
        operator: DistanceOperator::LESS_THAN,
        distance: $distance,
    )
);
```

produciendo un AST como:

```text
VectorDistancePredicate
├── vectorExpression
├── targetExpression
├── metric
├── comparison
└── threshold
```

El Compiler/Dialect decidirá posteriormente cómo representarlo.

---

# 4. Regla fundamental

```text
Custom Query Extension
≠
Raw SQL Macro
```

Una extensión deberá preservar información semántica durante todo el pipeline.

---

# 5. Posición arquitectónica

```text
Application
    │
    ▼
Query Builder Extension
    │
    ▼
Custom Semantic Node
    │
    ▼
Query AST
    │
    ▼
Semantic Extension Layer
    │
    ├── Validation
    ├── Symbol Resolution
    ├── Type Inference
    └── Capability Requirements
    │
    ▼
Optimizer
    │
    ▼
Planner
    │
    ▼
Custom Compiler Extension
    │
    ▼
Dialect
    │
    ▼
Compiled Command
```

---

# 6. Extension ≠ Builder Method

Agregar:

```php
$query->vectorDistance(...)
```

no constituye por sí mismo una Query Extension.

El método del Builder es únicamente una API de construcción.

La extensión real requiere:

```text
Builder API
+
Semantic Node
+
Validation
+
Type Semantics
+
Planning
+
Compilation
```

---

# 7. Extension ≠ AST Node

Igualmente:

```text
Custom AST Node
```

es sólo una parte.

Un nodo sin semantic rules ni compiler support es incompleto.

---

# 8. Extension ≠ Compiler Extension

Regla:

```text
Query Extension
≠
Compiler Extension
```

La primera define:

```text
what the operation means
```

La segunda:

```text
how that operation is represented
```

---

# 9. Extension ≠ Dialect Extension

Dialect Extension conoce sintaxis.

Query Extension conoce semántica.

---

# 10. Extension ≠ Capability

Registrar una extensión no significa que todos los DBMS puedan ejecutarla.

---

# 11. Extension ≠ Plugin

Un Plugin es un mecanismo de distribución/composición.

Una Query Extension es una capacidad semántica.

---

# 12. Extension ≠ RawExpression

`RawExpression` seguirá existiendo para casos no modelados.

Pero:

```text
reusable domain/database feature
→ Query Extension

one-off unsupported SQL escape hatch
→ RawExpression
```

---

# 13. Arquitectura general

```text
Database Plugin
      │
      ▼
Query Extension Provider
      │
      ▼
Query Extension Descriptor
      │
      ▼
Query Extension Registry
      │
      ├───────────────┐
      ▼               ▼
Semantic Registry   Builder Registry
      │               │
      ├───────────────┘
      ▼
Custom AST Nodes
      │
      ▼
Semantic Engine
      │
 ┌────┼─────────────┐
 ▼    ▼             ▼
Types Validation Capabilities
 │    │             │
 └────┼─────────────┘
      ▼
   Optimizer
      ↓
    Planner
      ↓
Compiler Extension
      ↓
    Dialect
      ↓
Compiled Command
```

---

# 14. Extension Identity

Toda extensión deberá tener:

```text
QueryExtensionId
```

estable.

Ejemplos:

```text
voltstack.query.json
voltstack.query.full_text
voltstack.query.spatial

acme.vector
acme.graph
acme.timeseries
```

---

# 15. QueryExtensionId

Conceptualmente:

```php
final readonly class QueryExtensionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 16. QueryExtensionId ≠ Package Name

Un package puede proporcionar múltiples extensiones.

---

# 17. QueryExtensionId ≠ FQCN

La identidad pública no dependerá de nombres internos de clases.

---

# 18. Extension Version

Podrá existir:

```text
QueryExtensionVersion
```

para versionar su contrato.

---

# 19. ExtensionVersion ≠ PackageVersion

Aunque puedan relacionarse:

```text
ExtensionVersion
≠
ComposerPackageVersion
```

---

# 20. Extension Descriptor

Conceptualmente:

```php
final readonly class QueryExtensionDescriptor
{
    public function __construct(
        public QueryExtensionId $id,
        public QueryExtensionVersion $version,
        public QueryExtensionKindSet $kinds,
        public PortabilityLevel $portability,
        public array $capabilityRequirements,
    ) {}
}
```

---

# 21. Extension Kinds

Podrán incluir:

```text
EXPRESSION
PREDICATE
OPERATOR
FUNCTION
AGGREGATE
WINDOW
CLAUSE
PROJECTION
ORDERING
SEARCH
```

---

# 22. Descriptor ≠ Runtime State

El descriptor será metadata inmutable.

---

# 23. Query Extension Registry

Resolverá:

```text
QueryExtensionId
→
QueryExtensionDefinition
```

---

# 24. Registry Lifecycle

```text
BUILDING
   ↓
VALIDATING
   ↓
COMPILING
   ↓
FROZEN
```

---

# 25. Runtime Registration

No estará permitida por defecto.

Especialmente en:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 26. Duplicate Extension ID

Resultado:

```text
DuplicateQueryExtensionException
```

---

# 27. Last Registration Wins

No será política predeterminada.

---

# 28. Query Extension Definition

Podrá agrupar:

```text
Descriptor
Semantic Nodes
Validators
Type Resolvers
Optimizer Rules
Planner Handlers
Compiler Handlers
Capability Requirements
Diagnostics
```

---

# 29. Semantic-first Design

VoltStack seguirá:

```text
Semantics
   ↓
Representation
```

y no:

```text
SQL syntax
   ↓
invent semantics afterwards
```

---

# 30. Custom AST Nodes

Una extensión podrá registrar nuevos tipos de nodo.

Ejemplo:

```php
final readonly class VectorDistanceExpression
    implements ExpressionNode
{
    public function __construct(
        public ExpressionNode $left,
        public ExpressionNode $right,
        public VectorMetric $metric,
    ) {}
}
```

---

# 31. AST Nodes Shall Be Immutable

Los nodos personalizados deberán ser inmutables.

---

# 32. AST Nodes Shall Be Structural

No deberán contener:

```text
Connection
Driver
EntityManager
PDO
Service Container
Request
Current User
```

---

# 33. AST Node ≠ Service

Un nodo representa información semántica.

No comportamiento operacional arbitrario.

---

# 34. AST Node Identity

El tipo de nodo deberá ser inequívoco.

Podrá existir:

```text
QueryNodeTypeId
```

estable para serialización, diagnostics y tooling.

---

# 35. NodeTypeId ≠ FQCN

Esto permitirá evolucionar implementaciones internas.

---

# 36. Custom Expressions

Una extensión podrá definir expresiones que produzcan valores.

Ejemplos:

```text
VectorDistanceExpression
JsonPathExpression
GeoDistanceExpression
FullTextRankExpression
TemporalOverlapExpression
```

---

# 37. Expression Contract

Conceptualmente:

```php
interface ExpressionNode extends QueryAstNode
{
}
```

Su tipo de resultado será resuelto por Semantic/Type System.

---

# 38. Expression ≠ Predicate

Una expresión produce un valor.

Un predicate representa una condición lógica.

---

# 39. Custom Predicates

Ejemplos:

```text
VectorWithinDistancePredicate
JsonContainsPredicate
GeoIntersectsPredicate
FullTextMatchesPredicate
```

---

# 40. Predicate Result

Semánticamente deberá producir:

```text
BOOLEAN / PREDICATE
```

según el modelo interno.

---

# 41. Predicate NULL Semantics

Cada predicate deberá declarar comportamiento respecto a:

```text
NULL
UNKNOWN
three-valued SQL logic
```

cuando sea relevante.

---

# 42. Custom Operators

Podrán definirse operadores semánticos.

Ejemplo:

```text
VECTOR_DISTANCE
JSON_CONTAINS
ARRAY_OVERLAPS
GEO_INTERSECTS
```

---

# 43. Semantic Operator ≠ SQL Token

Un operador semántico:

```text
ARRAY_OVERLAPS
```

podrá representarse como:

```sql
&&
```

en un dialecto y como una función en otro.

---

# 44. Operator Definition

Conceptualmente:

```php
final readonly class QueryOperatorDefinition
{
    public function __construct(
        public QueryOperatorId $id,
        public OperandSignature $operands,
        public TypeRule $resultType,
    ) {}
}
```

---

# 45. Operator Arity

Deberá declararse:

```text
UNARY
BINARY
NARY
CUSTOM
```

---

# 46. Operator Type Constraints

Ejemplo:

```text
VECTOR_DISTANCE(
    VECTOR,
    VECTOR
)
→ DECIMAL
```

---

# 47. Custom Functions

Podrán registrarse funciones semánticas.

Ejemplo:

```text
JSON_VALUE
VECTOR_NORM
GEO_DISTANCE
TEMPORAL_BUCKET
```

---

# 48. Function Identity

Toda función tendrá:

```text
QueryFunctionId
```

estable.

---

# 49. Function Name ≠ SQL Function Name

Ejemplo:

```text
QueryFunctionId:
voltstack.string.length
```

puede convertirse en:

```text
LENGTH()
CHAR_LENGTH()
```

según dialecto.

---

# 50. Function Signature

Una función deberá poder declarar:

```text
arguments
argument types
optional arguments
variadic rules
return type
nullability
determinism
```

cuando sea necesario.

---

# 51. Function Determinism

Podrá clasificarse:

```text
DETERMINISTIC
STABLE_WITHIN_STATEMENT
VOLATILE
UNKNOWN
```

---

# 52. Why Determinism Matters

El Optimizer podrá usarlo para decidir si una expresión puede:

```text
be deduplicated
be reordered
be constant-folded
be cached
```

---

# 53. Compiler Must Not Guess Determinism

Será metadata semántica.

---

# 54. Custom Aggregates

Ejemplos:

```text
VECTOR_CENTROID
PERCENTILE
JSON_AGGREGATE
STATISTICAL_MEDIAN
```

---

# 55. Aggregate ≠ Scalar Function

El Semantic Engine deberá conocer que:

```text
aggregate
```

depende de un conjunto de filas.

---

# 56. Aggregate Rules

Podrán declarar:

```text
DISTINCT support
FILTER support
ORDER BY support
null behavior
result type
window compatibility
```

---

# 57. Custom Window Expressions

Podrán definirse extensiones que operen sobre:

```text
WindowSpecification
```

---

# 58. Window Capability

Deberán declarar requisitos granulares.

Ejemplo:

```text
WINDOW_FUNCTIONS
WINDOW_FRAME_GROUPS
WINDOW_EXCLUDE
```

---

# 59. Custom Clauses

En casos avanzados una extensión podrá añadir una cláusula estructural.

Ejemplos potenciales:

```text
vendor-specific search clause
sampling clause
qualify-like clause
specialized temporal clause
```

---

# 60. Clause Extension Risk

Las cláusulas son más invasivas que expresiones.

Por ello deberán declarar:

```text
allowed statement types
position
multiplicity
conflicts
required capabilities
planner implications
```

---

# 61. Clause Position

No deberá expresarse simplemente como:

```text
"append SQL here"
```

Preferido:

```text
ClauseSlot
```

Ejemplo:

```text
AFTER_FROM
AFTER_WHERE
AFTER_GROUP
BEFORE_ORDER
AFTER_ORDER
BEFORE_LIMIT
```

sólo si el modelo sintáctico requiere slots.

---

# 62. Semantic Clause First

Aun con slots:

```text
ClauseNode
```

deberá preservar semántica.

---

# 63. Custom Projection

Podrán existir nodos especializados para:

```text
search rank
vector similarity
JSON projection
spatial representation
```

---

# 64. Projection ≠ Hydration

La extensión define qué valor solicita la query.

Hydration define cómo el resultado se transforma a una forma de aplicación.

---

# 65. Query Builder Integration

Las extensiones podrán proporcionar APIs ergonómicas.

Ejemplo:

```php
$query
    ->whereVectorDistance(
        'embedding',
        $vector,
        '<',
        0.20
    );
```

---

# 66. Builder Extension ≠ Core Mutation

No será necesario modificar la clase central del Query Builder para cada feature.

---

# 67. Builder Extension Strategies

VoltStack podrá soportar:

```text
typed extension object
registered fluent extension
specialized builder facade
extension namespace
```

---

# 68. Preferred API

Deberá favorecer:

```text
IDE discoverability
static analysis
type safety
```

sobre métodos mágicos ilimitados.

---

# 69. Magic Methods

Podrán existir como capa ergonómica opcional, pero no serán el único contrato.

---

# 70. Example Typed Extension

```php
$vector = $query->extension(VectorQuery::class);

$vector->whereWithinDistance(
    column: 'embedding',
    target: $embedding,
    distance: 0.20,
);
```

---

# 71. Query Builder Still Produces AST

Incluso mediante extensiones:

```text
Builder Extension
→ AST
```

Nunca:

```text
Builder Extension
→ execute SQL
```

---

# 72. ORM Integration

Una ORM extension podrá usar Query Extensions.

Ejemplo:

```php
Product::query()
    ->whereSimilarTo($embedding);
```

pero deberá converger a:

```text
Query AST
```

---

# 73. ORM Extension ≠ Query Extension

Se mantendrán separados.

El documento `300` definirá la extensión ORM.

---

# 74. Semantic Validation

Todo nodo personalizado deberá poder validarse.

---

# 75. Validation Rules

Podrán verificar:

```text
arity
operand types
context
statement placement
aggregate scope
window scope
symbol validity
capability requirements
```

---

# 76. Validation ≠ Compilation

Un compiler no será responsable de descubrir que:

```text
VECTOR_DISTANCE(integer, date)
```

es semánticamente inválido.

---

# 77. Semantic Validator

Contrato conceptual:

```php
interface QueryExtensionValidator
{
    public function validate(
        QueryAstNode $node,
        SemanticContext $context
    ): SemanticValidationResult;
}
```

---

# 78. Validation Result

Podrá contener:

```text
valid
errors
warnings
required capabilities
semantic annotations
```

---

# 79. Symbol Resolution

Las extensiones deberán integrarse con:

```text
SymbolResolutionSystem
```

para columnas, aliases, CTEs, scopes y relaciones.

---

# 80. Custom Node Shall Not Resolve Database Directly

Incorrecto:

```php
$connection->describeTable(...);
```

desde el nodo.

La resolución se realizará mediante el Semantic Context.

---

# 81. Type Inference Integration

Las extensiones deberán participar en:

```text
DATABASE_QUERY_TYPE_INFERENCE_SYSTEM
```

---

# 82. Type Resolver

Conceptualmente:

```php
interface QueryExtensionTypeResolver
{
    public function infer(
        ExpressionNode $expression,
        TypeInferenceContext $context
    ): QueryType;
}
```

---

# 83. Known Type

Ejemplo:

```text
VectorDistance
→ DecimalType
```

---

# 84. Dependent Type

Una función podría devolver un tipo dependiente de sus argumentos.

Ejemplo conceptual:

```text
JSON_VALUE(json, path, requestedType)
→ requestedType
```

---

# 85. Unknown Type

Si no puede inferirse:

```text
UNKNOWN
```

deberá preservarse.

No inventarse.

---

# 86. Type UNKNOWN ≠ ANY

Una distinción importante:

```text
UNKNOWN
≠
accepts everything
```

---

# 87. Nullability

El type resolver podrá declarar:

```text
NON_NULL
NULLABLE
UNKNOWN
```

---

# 88. Capability Requirements

Toda extensión podrá declarar requisitos.

Ejemplo:

```text
vector extension requires:
DATABASE_VECTOR_TYPE
VECTOR_DISTANCE_OPERATOR
```

---

# 89. Capability Requirement Composition

Podrá usar:

```text
ALL_OF
ANY_OF
NONE_OF
```

según `DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM`.

---

# 90. Extension Installed ≠ Capability Available

Regla:

```text
QueryExtensionRegistered
≠
DatabaseFeatureAvailable
```

---

# 91. Endpoint-specific Capabilities

Una extensión podrá funcionar en:

```text
endpoint A
```

y no en:

```text
endpoint B
```

dentro del mismo sistema distribuido.

---

# 92. Capability Check Phase

Los requisitos podrán comprobarse durante:

```text
semantic analysis
planning
compilation
```

dependiendo de cuándo exista suficiente información.

---

# 93. Early Rejection

Cuando sea posible:

```text
unsupported feature
```

deberá detectarse antes de execution.

---

# 94. Optimizer Integration

Las extensiones podrán registrar reglas de optimización.

---

# 95. Optimizer Extension ≠ Mandatory

Una extensión puede funcionar correctamente sin optimización especializada.

---

# 96. Correctness Before Optimization

Regla:

```text
No optimizer rule
→ potentially slower

Wrong optimizer rule
→ potentially incorrect
```

Por tanto, la ausencia de optimización es preferible a una transformación semánticamente insegura.

---

# 97. Custom Optimization Rule

Conceptualmente:

```php
interface QueryExtensionOptimizationRule
{
    public function supports(
        QueryAstNode $node
    ): bool;

    public function optimize(
        QueryAstNode $node,
        OptimizationContext $context
    ): QueryAstNode;
}
```

---

# 98. Optimization Must Preserve Semantics

Formalmente:

```text
Semantics(NodeBefore)
=
Semantics(NodeAfter)
```

dentro de las condiciones declaradas.

---

# 99. Function Volatility and Optimization

Funciones `VOLATILE` no deberán deduplicarse/reordenarse como si fueran deterministas.

---

# 100. Predicate Pushdown

Una extensión podrá declarar si un predicate es:

```text
pushdown-safe
pushdown-conditional
not-pushdown-safe
unknown
```

---

# 101. Optimizer Must Not Guess

Un nodo desconocido deberá tratarse conservadoramente.

---

# 102. Unknown Extension Optimization

Preferido:

```text
opaque semantic node
```

conservando su posición y semántica.

---

# 103. Planner Integration

El Planner decidirá cómo implementar una operación semántica.

---

# 104. Planner Handler

Conceptualmente:

```php
interface QueryExtensionPlanner
{
    public function plan(
        QueryAstNode $node,
        PlanningContext $context
    ): PlannedQueryNode;
}
```

---

# 105. Native Strategy

Ejemplo:

```text
VectorSearch
→ NativeVectorIndexPlan
```

---

# 106. Emulated Strategy

Otro endpoint podría utilizar:

```text
VectorSearch
→ FunctionEvaluationPlan
```

si la equivalencia y política lo permiten.

---

# 107. Unsupported Strategy

Si no existe representación válida:

```text
UnsupportedQueryExtensionException
```

---

# 108. Planner Chooses Strategy

No el Query Builder.

No el AST node.

No el Compiler.

---

# 109. Cost Hints

Una extensión podrá proporcionar:

```text
selectivity hint
cost category
index-awareness metadata
cardinality behavior
```

cuando exista evidencia suficiente.

---

# 110. Hint ≠ Truth

Los hints no deberán presentarse como garantías.

---

# 111. Compiler Integration

Después de planificación:

```text
Planned Custom Node
      ↓
Custom Compiler Extension
      ↓
Compiled Fragment
```

---

# 112. Compiler Extension Required

Toda extensión que alcance compilation deberá tener una representación compatible para el target.

---

# 113. Missing Compiler

Resultado:

```text
UnsupportedCompilationNodeException
```

---

# 114. Dialect Integration

La extensión podrá proporcionar renderers por dialecto.

Ejemplo:

```text
VectorDistance
├── PostgreSQLRenderer
├── MySQLRenderer
└── CustomDbRenderer
```

---

# 115. Renderer ≠ Semantics

Los tres renderers deberán implementar la misma semántica declarada o declarar diferencias explícitas.

---

# 116. PostgreSQL Example

Un backend con una extensión vectorial podría representar una distancia mediante un operador especializado.

La semántica seguirá siendo:

```text
VectorDistance
```

no el token SQL concreto.

---

# 117. Portable Extension

Una extensión será portable si puede preservar la semántica requerida en varios targets.

---

# 118. Portability Model

Podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_LIMITATIONS
PLATFORM_FAMILY
VENDOR_SPECIFIC
EXTENSION_SPECIFIC
UNKNOWN
```

---

# 119. PORTABLE ≠ Identical Performance

La misma semántica puede tener costos muy diferentes.

---

# 120. PORTABLE_WITH_LIMITATIONS

Deberá documentar:

```text
precision differences
collation differences
null differences
ordering differences
feature subset
performance implications
```

---

# 121. Vendor-specific Extension

Será válida.

VoltStack no deberá fingir portabilidad inexistente.

---

# 122. Explicit Vendor Feature

Ejemplo conceptual:

```text
PostgreSqlDistinctOn
```

puede ser mejor que esconderlo como una operación supuestamente universal.

---

# 123. Extension-specific Database Feature

Una capability podría depender de una extensión instalada en el servidor.

Ejemplo conceptual:

```text
VECTOR_EXTENSION_AVAILABLE
```

---

# 124. Server Extension ≠ VoltStack Plugin

No deberán confundirse.

```text
DBMS server extension
≠
VoltStack PHP plugin
```

---

# 125. Security Architecture

Las Query Extensions forman parte del boundary de seguridad de consultas.

---

# 126. Values

Los valores dinámicos deberán representarse como:

```text
ParameterNode
```

siempre que corresponda.

---

# 127. Identifiers

Los identifiers deberán usar:

```text
typed identifier model
```

---

# 128. Operators

No deberán aceptarse tokens SQL arbitrarios provenientes del usuario.

Incorrecto:

```php
vectorDistance(
    operator: $_GET['operator']
);
```

si se concatena directamente.

Preferido:

```text
DistanceComparison enum
```

---

# 129. Functions

Nombres de funciones deberán provenir del registry.

---

# 130. Custom Clause Input

No deberá permitir:

```text
arbitrary trailing SQL
```

como mecanismo ordinario.

---

# 131. Raw SQL Boundary

Si realmente se requiere:

```text
RawExpression
RawPredicate
RawClause
```

deberán estar marcados explícitamente como unsafe/escape-hatch según política.

---

# 132. Raw Query Extension Anti-pattern

Incorrecto:

```php
final class Extension
{
    public function sql(string $input): string
    {
        return $input;
    }
}
```

---

# 133. Parameterization Must Survive Extension Pipeline

Desde:

```text
Builder
```

hasta:

```text
Driver binding
```

la extensión no deberá perder la distinción:

```text
SQL structure
≠
runtime value
```

---

# 134. Authorization

Una Query Extension no deberá decidir por sí sola:

```text
who may query which data
```

---

# 135. Data Access Policies

Podrán transformar/restringir el AST antes de compilation.

La extensión deberá respetar esas restricciones.

---

# 136. Multitenancy

Una extensión no deberá omitir accidentalmente tenant predicates.

---

# 137. Custom Query ≠ Tenant Escape

Las mismas reglas de:

```text
Tenant Query Context
```

seguirán aplicando.

---

# 138. Sharding

Una extensión que utilice columnas relevantes para shard routing deberá exponer suficiente semántica al:

```text
Partition Routing System
```

---

# 139. Opaque Raw SQL Problem

Esta es otra razón para preferir AST:

```text
typed predicate
→ shard router can inspect

raw SQL
→ shard router may not understand
```

---

# 140. Read/Write Routing

Una extensión podrá declarar si la operación:

```text
READ_ONLY
WRITE
LOCKING_READ
UNKNOWN
```

cuando introduzca nuevos statement types.

---

# 141. UNKNOWN Access Intent

No deberá clasificarse automáticamente como replica-safe.

---

# 142. Transaction Requirements

Una extensión podrá declarar:

```text
requires transaction
requires writer
requires lock
requires same connection
```

como requirements semánticos/planificados.

---

# 143. Query Extension Does Not Begin Transaction

Sólo declara requisitos.

---

# 144. Cache Integration

Query Cache/Compiled Query Cache necesitarán entender la identidad semántica de extensiones.

---

# 145. Semantic Fingerprint

Un custom node deberá participar en:

```text
SemanticQueryFingerprint
```

---

# 146. Fingerprint Contract

Conceptualmente:

```php
interface QueryFingerprintContributor
{
    public function contribute(
        QueryAstNode $node,
        FingerprintBuilder $builder
    ): void;
}
```

---

# 147. Fingerprint Must Be Deterministic

Mismo nodo semántico:

```text
→ same fingerprint contribution
```

---

# 148. Values in Fingerprints

La política dependerá del cache.

Para compiled query cache:

```text
parameter value
```

normalmente no deberá alterar la estructura.

---

# 149. Structural Options

Sí deberán participar.

Ejemplo:

```text
distance metric = COSINE
```

si cambia el SQL/plan.

---

# 150. Extension Generation

Cambiar definición semántica/compiler support deberá invalidar caches relevantes.

---

# 151. QueryExtensionFingerprint

Podrá definirse:

```text
H(
    ExtensionId
    + ExtensionVersion
    + SemanticGeneration
    + CompilerGeneration
)
```

según contexto.

---

# 152. Result Cache

Si una extensión cambia semántica del resultado:

```text
Result Cache Key
```

deberá incluir su fingerprint semántico.

---

# 153. Metadata Cache

Podrá almacenar definiciones compiladas de extensiones, no state de queries.

---

# 154. Extension Lifecycle

```text
DISCOVERED
   ↓
REGISTERED
   ↓
VALIDATED
   ↓
COMPILED
   ↓
FROZEN
   ↓
ACTIVE
```

---

# 155. Runtime Mutation

Después de `FROZEN`:

```text
register/remove extension
```

estará prohibido por defecto.

---

# 156. Why Freeze

Para garantizar:

```text
determinism
cache safety
persistent runtime safety
performance
debuggability
```

---

# 157. Extension Dependencies

Una extensión podrá declarar:

```text
requires
optional
conflicts
before
after
```

cuando exista una relación real.

---

# 158. Example Dependency

```text
acme.vector.search
requires
acme.vector.core
```

---

# 159. Dependency Graph

```text
Extension Definitions
       ↓
Dependency Graph
       ↓
Cycle Detection
       ↓
Compatibility Validation
       ↓
Topological Resolution
       ↓
Frozen Registry
```

---

# 160. Dependency Cycle

Resultado:

```text
QueryExtensionDependencyCycleException
```

---

# 161. Conflicts

Ejemplo:

Dos extensiones registran:

```text
same QueryFunctionId
```

con semánticas diferentes.

Resultado:

```text
QueryExtensionConflictException
```

---

# 162. Priority Is Not a Conflict Solver by Default

No se ocultará el conflicto con:

```text
priority = 100
```

salvo que el extension point permita override explícito.

---

# 163. Override Policy

Si en el futuro se permite override:

```text
override target
expected version
compatibility declaration
explicit configuration
```

serán requeridos.

---

# 164. Core Semantic Namespace

VoltStack reservará namespaces como:

```text
voltstack.*
```

---

# 165. Third-party Namespace

Se recomendará:

```text
vendor.extension
```

Ejemplo:

```text
acme.vector
```

---

# 166. Query Function Namespace

Ejemplo:

```text
acme.vector.distance
```

---

# 167. Naming Stability

Los IDs persistentes no deberán cambiar simplemente por refactor PHP.

---

# 168. Diagnostics

El sistema deberá poder explicar:

```text
which extension introduced this node?
which validator processed it?
what type was inferred?
which capabilities are required?
which optimizer rule transformed it?
which planner strategy was chosen?
which compiler rendered it?
which dialect representation was used?
```

---

# 169. Query Extension Inspector

Conceptualmente:

```php
interface QueryExtensionInspector
{
    public function inspect(
        QueryExtensionId $id
    ): QueryExtensionDiagnosticReport;
}
```

---

# 170. Node Diagnostics

Ejemplo:

```text
Node:
VectorDistanceExpression

Extension:
acme.vector

Input Types:
VECTOR(1536), VECTOR(1536)

Result Type:
DECIMAL

Metric:
COSINE

Capability:
VECTOR_COSINE_DISTANCE

Planner Strategy:
NativeVectorDistance

Compiler:
PostgreSQLVectorDistanceCompiler

Portability:
EXTENSION_SPECIFIC
```

---

# 171. Diagnostics ≠ Execution

Inspection no deberá disparar queries ocultas.

---

# 172. Telemetry

Podrán observarse:

```text
extension usage count
semantic validation failures
unsupported capability failures
planner strategy usage
compiler strategy usage
emulation usage
```

---

# 173. Telemetry Cardinality

No se utilizarán:

```text
raw parameter values
arbitrary SQL
tenant IDs
user IDs
```

como metric labels.

---

# 174. Query Profiler

Podrá mostrar:

```text
custom semantic nodes
```

de forma sanitizada.

---

# 175. Debug Toolbar

Podrá indicar:

```text
Extensions used:
- acme.vector
- voltstack.json
```

sin exponer secretos.

---

# 176. Error Taxonomy

Propuesta:

```text
QueryExtensionException
├── QueryExtensionNotFoundException
├── DuplicateQueryExtensionException
├── QueryExtensionConflictException
├── QueryExtensionDependencyException
├── QueryExtensionDependencyCycleException
├── InvalidQueryExtensionNodeException
├── QueryExtensionTypeException
├── QueryExtensionValidationException
├── UnsupportedQueryExtensionException
├── QueryExtensionCapabilityException
├── QueryExtensionCompilerMissingException
├── QueryExtensionDialectUnsupportedException
└── QueryExtensionInvariantViolationException
```

---

# 177. Error Context

Un error podrá incluir:

```text
extension id
node type
semantic location
required capability
platform
dialect
```

pero no valores sensibles.

---

# 178. Testing Architecture

Toda Query Extension deberá probarse en múltiples niveles.

---

# 179. Unit Tests

Deberán validar:

```text
node construction
immutability
validation
type inference
fingerprinting
optimizer rules
planner decisions
```

sin DB real cuando no sea necesaria.

---

# 180. Compiler Tests

Deberán validar:

```text
semantic node
→ compiled representation
```

por dialecto.

---

# 181. Integration Tests

Cuando la semántica dependa del DBMS:

```text
Custom Query
   ↓
Compile
   ↓
Execute on real DBMS
   ↓
Verify result
```

---

# 182. Semantic Conformance

La prueba importante no será sólo:

```text
expected SQL string
```

sino:

```text
expected semantic result
```

---

# 183. Cross-platform Tests

Si la extensión declara:

```text
PORTABLE
```

deberá probarse en todos los targets soportados.

---

# 184. Vendor-specific Tests

Si declara:

```text
VENDOR_SPECIFIC
```

no se exigirá falsa portabilidad.

---

# 185. Capability-conditioned Tests

Para cada feature:

```text
SUPPORTED
→ execute native path

REQUIRES_EMULATION
→ verify approved emulation

UNSUPPORTED
→ verify explicit rejection

UNKNOWN
→ verify no unsupported assumption
```

---

# 186. Security Tests

Deberán incluir:

```text
malicious values
malicious identifiers
operator injection attempts
function-name injection
raw clause attempts
```

---

# 187. Parameter Injection Test

Un valor como:

```text
' OR 1=1 --
```

deberá continuar siendo un parameter.

---

# 188. Persistent Runtime Tests

Deberán probar:

```text
Request A uses extension X
Request B does not
```

sin contaminación de:

```text
AST
parameters
diagnostics
tenant context
extension state
```

---

# 189. FrankenPHP

Será el runtime persistente principal de referencia.

---

# 190. OpenSwoole

Los registries podrán compartirse si son inmutables.

Los query states deberán ser coroutine-scoped.

---

# 191. Performance Tests

Deberán medir:

```text
extension dispatch
semantic validation
type inference
optimizer overhead
planner overhead
compiler overhead
fingerprint overhead
```

cuando la extensión se encuentre en hot paths.

---

# 192. Extension Dispatch

No deberá realizar:

```text
foreach all extensions
```

por nodo.

---

# 193. Precompiled Dispatch Tables

Preferido:

```text
NodeTypeId
→ Validator
→ TypeResolver
→ Optimizer Metadata
→ Planner Handler
→ Compiler Handler
```

---

# 194. Registry Compilation

Durante bootstrap:

```text
Mutable Definitions
       ↓
Validation
       ↓
Conflict Resolution
       ↓
Dispatch Table Compilation
       ↓
Frozen Registry
```

---

# 195. Reflection

Podrá utilizarse durante bootstrap cuando aporte valor.

Deberá evitarse en hot paths cuando pueda precompilarse.

---

# 196. Container Resolution

No deberá ocurrir por cada expresión.

---

# 197. Extension Services

Los services deberán resolverse durante bootstrap o mediante factories explícitas.

---

# 198. Stateless Components

Preferidos para:

```text
validators
type resolvers
compilers
fingerprinters
```

---

# 199. Query-scoped State

Si una extensión necesita state temporal:

```text
QueryExtensionContext
```

deberá ser operation-scoped.

---

# 200. Extension Context ≠ Global State

Nunca:

```php
static $currentVectorMetric;
```

---

# 201. Example: Vector Extension

API:

```php
$query
    ->select('id')
    ->selectExtension(
        Vector::distance(
            'embedding',
            $targetVector,
            VectorMetric::COSINE
        ),
        'distance'
    )
    ->orderBy('distance')
    ->limit(10);
```

---

# 202. Resulting Semantic AST

```text
SelectQuery
├── Projection
│   ├── Column(id)
│   └── Alias(distance)
│       └── VectorDistanceExpression
│           ├── Column(embedding)
│           ├── Parameter(targetVector)
│           └── Metric(COSINE)
├── OrderBy
│   └── distance ASC
└── Limit
    └── 10
```

---

# 203. Semantic Analysis

```text
Column embedding
→ VECTOR(1536)

targetVector
→ VECTOR(1536)

VectorDistance
→ valid

Result
→ DECIMAL
```

---

# 204. Capability Resolution

```text
VECTOR_TYPE
+
COSINE_DISTANCE
```

deberán estar disponibles o existir una estrategia aprobada.

---

# 205. Planning

El Planner podría seleccionar:

```text
NativeVectorIndexPlan
```

si existe soporte apropiado.

De lo contrario:

```text
SequentialVectorEvaluationPlan
```

sólo si la política y capabilities lo permiten.

---

# 206. Compilation

Un dialecto podría representar:

```text
VectorDistance
```

mediante un operador.

Otro mediante una función.

El Query AST permanece igual.

---

# 207. Benefit

La aplicación conserva:

```text
semantic portability
type safety
security
optimizer visibility
planner visibility
diagnostics
testability
```

---

# 208. Example: JSON Extension

API conceptual:

```php
$query->where(
    Json::contains(
        column: 'metadata',
        path: '$.roles',
        value: 'admin'
    )
);
```

---

# 209. JSON Semantics

Deberá preservar:

```text
SQL NULL
JSON null
missing path
empty collection
```

como conceptos distintos cuando corresponda.

---

# 210. JSON Compiler

Podrá producir distintas representaciones por dialecto sin cambiar el modelo semántico.

---

# 211. Example: Full-text Search

```php
$query->where(
    FullText::matches(
        columns: ['title', 'content'],
        query: $search,
    )
);
```

---

# 212. Full-text Semantics

Podrá incluir:

```text
language
tokenization mode
ranking
matching mode
normalization
```

cuando puedan modelarse portablemente.

---

# 213. Platform Differences

Si las diferencias entre motores impiden equivalencia real:

```text
PORTABLE_WITH_LIMITATIONS
```

o:

```text
VENDOR_SPECIFIC
```

será preferible a fingir equivalencia.

---

# 214. Example: Spatial Extension

```php
Geo::withinDistance(
    point: column('location'),
    target: parameter($point),
    distance: 1000,
    unit: DistanceUnit::METER,
);
```

---

# 215. Coordinate Reference System

Deberá formar parte de la semántica cuando sea necesario.

No dejarse implícito en SQL raw.

---

# 216. Units

También deberán ser tipadas.

```text
meter
kilometer
mile
```

no strings concatenadas.

---

# 217. Extension Interoperability

Dos extensiones podrán coexistir.

Ejemplo:

```text
JSON expression
inside
custom aggregate
inside
window expression
```

si sus contratos son compatibles.

---

# 218. Composition

El sistema deberá favorecer composición de nodos.

---

# 219. Extension Isolation

Una extensión no deberá asumir que controla todo el Query AST.

---

# 220. Cross-extension Dependencies

Si necesita otro nodo:

```text
requires extension
```

deberá declararlo.

---

# 221. Cross-extension Compiler Compatibility

No basta con que ambas extensiones estén registradas.

El target compiler deberá soportar la composición resultante.

---

# 222. Compatibility Matrix

Podrá modelarse:

```text
Extension
×
Compiler
×
Dialect
×
Capability
```

---

# 223. Extension Support Decision

Formalmente:

```text
Support(E, Target)
=
SemanticSupport(E)
∧
CompilerSupport(E, Target)
∧
DialectRepresentability(E, Target)
∧
CapabilityRequirements(E, Target)
```

---

# 224. Registered ≠ Usable

Por tanto:

```text
Registered(E)
≠
Usable(E, Endpoint)
```

---

# 225. Query Extension Public API

La API pública deberá priorizar:

```text
typed objects
enums
value objects
builder extensions
```

sobre:

```text
SQL strings
magic arrays
unvalidated tokens
```

---

# 226. Extension Configuration

Podrá existir configuración estructural.

Ejemplo:

```php
VectorExtensionConfig(
    defaultMetric: VectorMetric::COSINE,
);
```

---

# 227. Configuration ≠ Query State

La configuración global no almacenará:

```text
current vector
current query
current tenant
```

---

# 228. Default Semantics

Los defaults deberán ser explícitos y estables.

---

# 229. Hidden Vendor Defaults

Deberán evitarse cuando cambien la semántica.

---

# 230. Serialization

Los nodos podrán necesitar serialización para:

```text
query cache
debugging
distributed jobs
precompiled manifests
```

en el futuro.

---

# 231. Stable Semantic Representation

Por ello se evitará almacenar closures arbitrarios dentro de nodos AST.

---

# 232. Closures in AST

No serán recomendadas como contrato público durable.

---

# 233. Query Extension Metadata

Podrá incluir:

```text
extension id
node type
semantic version
capabilities
portability
determinism
side-effect classification
```

---

# 234. Side Effects

Las expresiones normales deberán ser consideradas libres de side effects en VoltStack, salvo metadata explícita.

---

# 235. Database Volatile Functions

Una función DB volátil no implica side effects PHP, pero sí afecta reglas de optimización.

---

# 236. Statement Extensions

Una extensión que introduzca nuevos tipos de statement requerirá contratos más estrictos.

Ejemplos futuros:

```text
vendor-specific maintenance query
analytical command
specialized search command
```

---

# 237. Statement Extension Requirements

Deberá declarar:

```text
read/write intent
transaction requirements
result shape
compiler support
driver requirements
security classification
```

---

# 238. DDL

Las operaciones estructurales seguirán perteneciendo principalmente a:

```text
Schema AST
Schema Compiler
Migration System
```

No deberán introducirse como Query Extension ordinaria.

---

# 239. Query Extension ≠ Schema Extension

Regla:

```text
Query semantics
≠
Schema evolution
```

---

# 240. Administration

Tampoco:

```text
Query Extension
≠
Administration command
```

---

# 241. Extension Deprecation

Una extensión podrá marcarse:

```text
DEPRECATED
```

con:

```text
replacement
deprecation version
removal target
migration guidance
```

---

# 242. Node Deprecation

También podrán deprecarse nodos específicos sin eliminar toda la extensión.

---

# 243. Backward Compatibility

Los cambios en semántica serán tratados como cambios significativos.

---

# 244. Renaming PHP Class

No deberá romper fingerprints/identidades si el contrato semántico permanece.

---

# 245. Changing Semantic Meaning

Sí deberá alterar:

```text
semantic version
generation
fingerprint
```

cuando corresponda.

---

# 246. Query Extension Conformance Kit

VoltStack podrá proporcionar:

```text
QueryExtensionConformanceKit
```

para terceros.

---

# 247. Conformance Levels

Podrán definirse:

```text
SEMANTIC
COMPILER
DIALECT
INTEGRATION
PORTABILITY
SECURITY
RUNTIME
```

---

# 248. Semantic Conformance

Comprueba:

```text
AST validity
type inference
validation
fingerprint
optimizer safety
planner contracts
```

---

# 249. Compiler Conformance

Comprueba:

```text
node compilation
parameterization
identifier handling
determinism
```

---

# 250. Integration Conformance

Comprueba comportamiento real en DBMS.

---

# 251. Portability Conformance

Sólo aplica si la extensión declara portabilidad.

---

# 252. Security Conformance

Comprueba que la extensión no cree nuevas vías de injection.

---

# 253. Runtime Conformance

Comprueba ausencia de leaks/state contamination.

---

# 254. Proposed Directory Structure

```text
src/Quantum/Database/Query/Extension/
├── Contract/
│   ├── QueryExtension.php
│   ├── QueryExtensionProvider.php
│   ├── QueryExtensionValidator.php
│   ├── QueryExtensionTypeResolver.php
│   ├── QueryExtensionPlanner.php
│   └── QueryFingerprintContributor.php
│
├── Identity/
│   ├── QueryExtensionId.php
│   ├── QueryExtensionVersion.php
│   ├── QueryNodeTypeId.php
│   ├── QueryFunctionId.php
│   └── QueryOperatorId.php
│
├── Descriptor/
│   ├── QueryExtensionDescriptor.php
│   ├── QueryExtensionKind.php
│   ├── QueryExtensionMetadata.php
│   └── PortabilityLevel.php
│
├── Registry/
│   ├── QueryExtensionRegistry.php
│   ├── MutableQueryExtensionRegistry.php
│   ├── FrozenQueryExtensionRegistry.php
│   └── QueryExtensionDispatchTable.php
│
├── Ast/
│   ├── CustomExpressionNode.php
│   ├── CustomPredicateNode.php
│   ├── CustomClauseNode.php
│   └── CustomProjectionNode.php
│
├── Function/
│   ├── QueryFunctionDefinition.php
│   ├── QueryFunctionSignature.php
│   └── QueryFunctionRegistry.php
│
├── Operator/
│   ├── QueryOperatorDefinition.php
│   ├── OperandSignature.php
│   └── QueryOperatorRegistry.php
│
├── Aggregate/
│   ├── QueryAggregateDefinition.php
│   └── QueryAggregateRegistry.php
│
├── Window/
│   ├── QueryWindowFunctionDefinition.php
│   └── QueryWindowFunctionRegistry.php
│
├── Semantic/
│   ├── ExtensionSemanticAnalyzer.php
│   ├── ExtensionValidationResult.php
│   └── ExtensionTypeInference.php
│
├── Optimization/
│   ├── QueryExtensionOptimizationRule.php
│   └── ExtensionOptimizationRegistry.php
│
├── Planning/
│   ├── QueryExtensionPlanner.php
│   ├── ExtensionPlanStrategy.php
│   └── ExtensionPlanningRegistry.php
│
├── Compilation/
│   ├── QueryExtensionCompilerBridge.php
│   └── QueryExtensionCompilationRequirement.php
│
├── Capability/
│   ├── QueryExtensionCapabilityRequirement.php
│   └── QueryExtensionSupportResolver.php
│
├── Fingerprint/
│   ├── QueryExtensionFingerprint.php
│   └── ExtensionFingerprintContributor.php
│
├── Diagnostics/
│   ├── QueryExtensionInspector.php
│   └── QueryExtensionDiagnosticReport.php
│
└── Exception/
    ├── QueryExtensionException.php
    ├── QueryExtensionNotFoundException.php
    ├── DuplicateQueryExtensionException.php
    ├── QueryExtensionConflictException.php
    ├── QueryExtensionDependencyException.php
    ├── QueryExtensionValidationException.php
    ├── QueryExtensionTypeException.php
    ├── UnsupportedQueryExtensionException.php
    └── QueryExtensionInvariantViolationException.php
```

---

# 255. Third-party Package Example

```text
acme/voltstack-vector/
├── composer.json
├── src/
│   ├── Plugin/
│   │   └── VectorDatabasePlugin.php
│   ├── Query/
│   │   ├── VectorQueryExtension.php
│   │   ├── VectorDistanceExpression.php
│   │   ├── VectorWithinDistancePredicate.php
│   │   └── VectorMetric.php
│   ├── Semantic/
│   │   ├── VectorValidator.php
│   │   └── VectorTypeResolver.php
│   ├── Optimization/
│   ├── Planning/
│   ├── Compiler/
│   │   ├── PostgreSQL/
│   │   └── Custom/
│   └── Capability/
└── tests/
    ├── Unit/
    ├── Conformance/
    ├── Security/
    └── Integration/
```

---

# 256. Architectural Invariants

## DB-CUSTOM-QUERY-001

Custom Query Extension ≠ Raw SQL Macro.

## DB-CUSTOM-QUERY-002

Extension ≠ Builder Method.

## DB-CUSTOM-QUERY-003

Extension ≠ AST Node.

## DB-CUSTOM-QUERY-004

Query Extension ≠ Compiler Extension.

## DB-CUSTOM-QUERY-005

Query Extension ≠ Dialect Extension.

## DB-CUSTOM-QUERY-006

Query Extension ≠ Capability.

## DB-CUSTOM-QUERY-007

Query Extension ≠ Plugin.

## DB-CUSTOM-QUERY-008

Query Extension ≠ RawExpression.

## DB-CUSTOM-QUERY-009

Semantics precederá representation.

## DB-CUSTOM-QUERY-010

QueryExtensionId será estable.

## DB-CUSTOM-QUERY-011

QueryExtensionId ≠ FQCN.

## DB-CUSTOM-QUERY-012

ExtensionVersion ≠ PackageVersion.

## DB-CUSTOM-QUERY-013

Registry será frozen después de bootstrap.

## DB-CUSTOM-QUERY-014

Duplicate Extension ID será error.

## DB-CUSTOM-QUERY-015

Last registration wins estará prohibido por defecto.

## DB-CUSTOM-QUERY-016

Custom AST nodes serán inmutables.

## DB-CUSTOM-QUERY-017

AST nodes no contendrán Connection.

## DB-CUSTOM-QUERY-018

AST nodes no contendrán Driver.

## DB-CUSTOM-QUERY-019

AST nodes no contendrán EntityManager.

## DB-CUSTOM-QUERY-020

AST nodes no contendrán Service Container.

## DB-CUSTOM-QUERY-021

AST node ≠ Service.

## DB-CUSTOM-QUERY-022

Expression ≠ Predicate.

## DB-CUSTOM-QUERY-023

Predicate NULL semantics deberán ser explícitas cuando sean relevantes.

## DB-CUSTOM-QUERY-024

Semantic Operator ≠ SQL Token.

## DB-CUSTOM-QUERY-025

Function Identity ≠ SQL Function Name.

## DB-CUSTOM-QUERY-026

Function determinism será metadata semántica.

## DB-CUSTOM-QUERY-027

Compiler no adivinará function volatility.

## DB-CUSTOM-QUERY-028

Aggregate ≠ Scalar Function.

## DB-CUSTOM-QUERY-029

Window capabilities podrán ser granulares.

## DB-CUSTOM-QUERY-030

Custom clauses declararán posición y restricciones.

## DB-CUSTOM-QUERY-031

Custom clause ≠ arbitrary trailing SQL.

## DB-CUSTOM-QUERY-032

Projection ≠ Hydration.

## DB-CUSTOM-QUERY-033

Builder extensions producirán AST.

## DB-CUSTOM-QUERY-034

Builder extensions no ejecutarán SQL.

## DB-CUSTOM-QUERY-035

ORM Extension ≠ Query Extension.

## DB-CUSTOM-QUERY-036

Validation ≠ Compilation.

## DB-CUSTOM-QUERY-037

Custom nodes no consultarán DB directamente durante semantic analysis.

## DB-CUSTOM-QUERY-038

Type inference será extensible.

## DB-CUSTOM-QUERY-039

UNKNOWN Type ≠ ANY.

## DB-CUSTOM-QUERY-040

QueryExtensionRegistered ≠ DatabaseFeatureAvailable.

## DB-CUSTOM-QUERY-041

Capabilities podrán ser endpoint-specific.

## DB-CUSTOM-QUERY-042

Unsupported features deberán fallar antes de execution cuando sea posible.

## DB-CUSTOM-QUERY-043

Optimizer integration será opcional.

## DB-CUSTOM-QUERY-044

Correctness tendrá prioridad sobre optimization.

## DB-CUSTOM-QUERY-045

Optimizer rules preservarán semántica.

## DB-CUSTOM-QUERY-046

VOLATILE functions no serán tratadas como deterministic.

## DB-CUSTOM-QUERY-047

Optimizer no adivinará pushdown safety.

## DB-CUSTOM-QUERY-048

Unknown extension nodes serán tratados conservadoramente.

## DB-CUSTOM-QUERY-049

Planner decidirá implementation strategy.

## DB-CUSTOM-QUERY-050

Query Builder no decidirá native/emulated strategy.

## DB-CUSTOM-QUERY-051

Compiler no decidirá native/emulated strategy unilateralmente.

## DB-CUSTOM-QUERY-052

Cost hint ≠ Truth.

## DB-CUSTOM-QUERY-053

Missing compiler support será error.

## DB-CUSTOM-QUERY-054

Renderer ≠ Semantics.

## DB-CUSTOM-QUERY-055

Portability ≠ Identical SQL.

## DB-CUSTOM-QUERY-056

Portability ≠ Identical Performance.

## DB-CUSTOM-QUERY-057

VoltStack no fingirá portabilidad inexistente.

## DB-CUSTOM-QUERY-058

DBMS Extension ≠ VoltStack Plugin.

## DB-CUSTOM-QUERY-059

Dynamic values serán parameters.

## DB-CUSTOM-QUERY-060

Identifiers serán tipados.

## DB-CUSTOM-QUERY-061

Operators no aceptarán tokens arbitrarios.

## DB-CUSTOM-QUERY-062

Function names provendrán de registry.

## DB-CUSTOM-QUERY-063

Raw SQL será escape hatch explícito.

## DB-CUSTOM-QUERY-064

Parameterization sobrevivirá todo el pipeline.

## DB-CUSTOM-QUERY-065

Query Extension no decidirá autorización.

## DB-CUSTOM-QUERY-066

Query Extension no omitirá tenant constraints.

## DB-CUSTOM-QUERY-067

Custom Query ≠ Tenant Escape.

## DB-CUSTOM-QUERY-068

Sharding deberá poder inspeccionar semántica relevante.

## DB-CUSTOM-QUERY-069

UNKNOWN access intent ≠ replica-safe.

## DB-CUSTOM-QUERY-070

Query Extension no iniciará transaction.

## DB-CUSTOM-QUERY-071

Custom nodes participarán en semantic fingerprints.

## DB-CUSTOM-QUERY-072

Fingerprints serán deterministas.

## DB-CUSTOM-QUERY-073

Structural options participarán en fingerprints cuando afecten output.

## DB-CUSTOM-QUERY-074

Extension generations invalidarán caches relevantes.

## DB-CUSTOM-QUERY-075

Result Cache reflejará semántica de extensions.

## DB-CUSTOM-QUERY-076

Metadata Cache no almacenará query state.

## DB-CUSTOM-QUERY-077

Registries no mutarán después de freeze.

## DB-CUSTOM-QUERY-078

Extension dependency cycles serán error.

## DB-CUSTOM-QUERY-079

Conflicts serán explícitos.

## DB-CUSTOM-QUERY-080

Priority no ocultará conflicts por defecto.

## DB-CUSTOM-QUERY-081

Core namespace será reservado.

## DB-CUSTOM-QUERY-082

IDs no cambiarán por refactor interno.

## DB-CUSTOM-QUERY-083

Diagnostics no ejecutarán queries ocultas.

## DB-CUSTOM-QUERY-084

Telemetry no utilizará sensitive values como labels.

## DB-CUSTOM-QUERY-085

Conformance ≠ SQL string equality.

## DB-CUSTOM-QUERY-086

Portable extensions se probarán en todos sus targets declarados.

## DB-CUSTOM-QUERY-087

Vendor-specific extensions no requerirán falsa portabilidad.

## DB-CUSTOM-QUERY-088

Security tests serán obligatorios para input dinámico.

## DB-CUSTOM-QUERY-089

Persistent runtime state será aislado.

## DB-CUSTOM-QUERY-090

Extension dispatch no escaneará todos los plugins por nodo.

## DB-CUSTOM-QUERY-091

Dispatch tables podrán precompilarse.

## DB-CUSTOM-QUERY-092

Container resolution no ocurrirá por expresión.

## DB-CUSTOM-QUERY-093

Query-scoped state no será static/global.

## DB-CUSTOM-QUERY-094

JSON SQL NULL ≠ JSON null ≠ missing path.

## DB-CUSTOM-QUERY-095

Spatial units serán tipadas cuando sean semánticamente relevantes.

## DB-CUSTOM-QUERY-096

Extensions podrán componerse sólo bajo contratos compatibles.

## DB-CUSTOM-QUERY-097

Registered ≠ Usable.

## DB-CUSTOM-QUERY-098

Query Extension ≠ Schema Extension.

## DB-CUSTOM-QUERY-099

Query Extension ≠ Administration Command.

## DB-CUSTOM-QUERY-100

Custom Query Extensions preservarán las invariantes globales de Query Engine.

---

# 257. Anti-patrones

## 257.1 Extensión como string SQL

```php
$extension->render($userSql);
```

Incorrecto.

---

## 257.2 Builder method ejecutando SQL

Incorrecto.

---

## 257.3 Nodo AST con PDO

Incorrecto.

---

## 257.4 Nodo AST resolviendo services desde Container

Incorrecto.

---

## 257.5 Nodo AST leyendo request global

Incorrecto.

---

## 257.6 Query function definida únicamente por nombre SQL

Insuficiente.

---

## 257.7 Operadores provenientes directamente del usuario

Incorrecto.

---

## 257.8 Optimizer transformando nodo desconocido agresivamente

Incorrecto.

---

## 257.9 Compiler inventando semántica

Incorrecto.

---

## 257.10 Asumir capability porque plugin está instalado

Incorrecto.

---

## 257.11 Usar SQLite para probar portabilidad universal

Incorrecto.

---

## 257.12 Tratar vendor-specific feature como portable sin evidencia

Incorrecto.

---

## 257.13 Extension registry mutable por request

Incorrecto.

---

## 257.14 Last registration wins

Incorrecto.

---

## 257.15 Magic method como único API contract

Debe evitarse.

---

## 257.16 Closures arbitrarios dentro del AST

Debe evitarse como contrato durable.

---

## 257.17 Raw SQL para toda feature nueva

Incorrecto.

---

## 257.18 Ignorar semantic fingerprint

Incorrecto.

---

## 257.19 Query Extension iniciando transacción

Incorrecto.

---

## 257.20 Query Extension seleccionando replica/shard

Incorrecto.

---

# 258. Modelo formal

Sea:

```text
E
```

una Query Extension y:

```text
N
```

uno de sus nodos.

Para que `N` sea utilizable deberá cumplirse:

```text
Registered(E)
∧
Valid(N)
∧
TypesResolved(N)
∧
CapabilitiesSatisfied(N)
∧
PlanAvailable(N)
∧
CompilerAvailable(N)
∧
DialectRepresentable(N)
```

---

# 259. Semantic Preservation

Para una transformación de optimizer:

```text
N → N'
```

deberá cumplirse:

```text
Semantics(N)
=
Semantics(N')
```

bajo las precondiciones declaradas.

---

# 260. Compilation

Para un nodo planificado:

```text
Compile(
    PlannedNode,
    Compiler,
    Dialect,
    Capabilities
)
→
CompiledFragment
```

deberá preservar:

```text
Semantics(CompiledFragment)
≡
Semantics(PlannedNode)
```

dentro del contrato del target.

---

# 261. Usability

Formalmente:

```text
Usable(E, Endpoint)
=
Registered(E)
∧
SemanticSupport(E)
∧
CapabilitiesSatisfied(E, Endpoint)
∧
PlannerSupport(E)
∧
CompilerSupport(E)
∧
DialectSupport(E)
```

Por tanto:

```text
Registered(E)
≠
Usable(E, Endpoint)
```

---

# 262. Portability

Para una extensión `E` y plataformas:

```text
P1 ... Pn
```

podrá declararse portable sólo cuando:

```text
∀ Pi ∈ SupportedPlatforms(E):
Semantics(E, Pi)
≈
DeclaredSemantics(E)
```

donde cualquier desviación relevante deberá formar parte del contrato de portabilidad.

---

# 263. Fingerprint

```text
QueryExtensionFingerprint
=
H(
    ExtensionId
    + SemanticVersion
    + StructuralConfiguration
    + SemanticGeneration
)
```

y el fingerprint del query deberá incorporar las contribuciones estructurales de sus nodos.

---

# 264. Security

Para todo input dinámico `V`:

```text
V ∈ Values
→
Parameterize(V)
```

Para todo identifier `I`:

```text
I ∈ Identifiers
→
Validate(I)
→
TypedIdentifier(I)
→
DialectQuote(I)
```

Nunca:

```text
UserInput
→
SQL Concatenation
```

como camino ordinario.

---

# 265. Arquitectura consolidada

```text
                         Application
                              │
                              ▼
                     Query Builder API
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
          Core Query Nodes         Extension Builder
                                           │
                                           ▼
                                  Custom Semantic Nodes
                 │                         │
                 └────────────┬────────────┘
                              ▼
                         Query AST
                              │
                              ▼
                    Semantic Analysis
                 ┌────────────┼─────────────┐
                 ▼            ▼             ▼
             Symbols        Types       Validation
                 │            │             │
                 └────────────┼─────────────┘
                              ▼
                       Capabilities
                              │
                              ▼
                         Optimizer
                              │
                              ▼
                          Planner
                              │
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
           Native Plan    Emulated Plan   Unsupported
               │              │
               └──────────────┘
                              ▼
                         Compiler
                              │
                              ▼
                    Compiler Extension
                              │
                              ▼
                           Dialect
                              │
                              ▼
                     Compiled Command
                              │
                              ▼
                     Execution Engine
                              │
                              ▼
                            Driver
                              │
                              ▼
                             DBMS
```

---

# 266. Estrategia V1

La V1 deberá estabilizar primero:

```text
QueryExtensionId
QueryExtensionDescriptor
QueryExtensionRegistry
QueryExtensionProvider

Custom Expression Nodes
Custom Predicate Nodes
Custom Operators
Custom Functions

Semantic Validation
Type Inference
Capability Requirements

Optimizer Extension Contract
Planner Extension Contract
Compiler Bridge

Semantic Fingerprinting
Portability Metadata
Conflict Detection
Dependency Resolution
Diagnostics
Conformance Testing
```

Las cláusulas y statement extensions altamente invasivas podrán mantenerse como APIs avanzadas hasta estabilizar los contratos fundamentales.

---

# 267. Evolución posterior

La arquitectura permitirá incorporar posteriormente:

```text
vector search DSL
advanced JSON query DSL
spatial DSL
full-text search DSL
temporal query DSL
graph-query extensions
time-series functions
statistical functions
analytical functions
machine-learning database functions
custom index-aware predicates
database-native AI operations
specialized distributed query nodes
custom statement types
```

sin convertir Query Builder en un conjunto de SQL strings específicos de cada proveedor.

---

# 268. Relación con documentos anteriores

El sistema deberá respetar:

```text
294_DATABASE_EXTENSION_ARCHITECTURE
        ↓
295_DATABASE_PLUGIN_SYSTEM
        ↓
296_DATABASE_CUSTOM_DRIVER_SYSTEM
        ↓
297_DATABASE_CUSTOM_DIALECT_SYSTEM
        ↓
298_DATABASE_CUSTOM_COMPILER_SYSTEM
        ↓
299_DATABASE_CUSTOM_QUERY_EXTENSION_SYSTEM
```

Cada capa responde a una pregunta diferente:

```text
Extension Architecture
→ ¿cómo puede extenderse Database de forma segura?

Plugin System
→ ¿cómo se distribuyen y componen extensiones?

Custom Driver
→ ¿cómo se comunica VoltStack con otro backend?

Custom Dialect
→ ¿cómo se representa su variante SQL?

Custom Compiler
→ ¿cómo se transforma el modelo semántico en comandos?

Custom Query Extension
→ ¿cómo ampliamos el lenguaje semántico de consultas?
```

---

# 269. Regla final

> **VoltStack no tratará una nueva capacidad de consulta como un fragmento SQL hasta que sea imposible o injustificado modelarla semánticamente. Siempre que una operación tenga identidad, tipos, reglas, capacidades, planificación o comportamiento reutilizable, deberá representarse como una Query Extension estructurada.**

Por tanto:

```text
Query Extension
≠
Raw SQL Macro
```

```text
Query Extension
≠
Builder Method
```

```text
Query Extension
≠
AST Node únicamente
```

```text
Query Extension
≠
Compiler Extension
```

```text
Query Extension
≠
Dialect Extension
```

```text
Query Extension
≠
Capability
```

```text
Query Extension
≠
Plugin
```

```text
Query Extension
≠
ORM Extension
```

```text
Query Extension
≠
Schema Extension
```

```text
Expression
≠
Predicate
```

```text
Aggregate
≠
Scalar Function
```

```text
Semantic Operator
≠
SQL Token
```

```text
Semantic Function
≠
Vendor Function Name
```

```text
Registered Extension
≠
Available Database Feature
```

```text
Registered Extension
≠
Usable Extension
```

```text
Portability
≠
Identical SQL
```

```text
Portability
≠
Identical Performance
```

```text
RawExpression
≠
Primary Extension Mechanism
```

y finalmente:

```text
Safe Query Extension
=
Stable Identity
+
Typed Semantic Nodes
+
Immutable AST
+
Builder Integration
+
Semantic Validation
+
Type Inference
+
Capability Requirements
+
Optimizer Safety
+
Explicit Planning
+
Compiler Integration
+
Dialect Representation
+
Parameterization
+
Security Boundaries
+
Semantic Fingerprinting
+
Portability Contract
+
Deterministic Registration
+
Persistent Runtime Isolation
+
Conformance Testing
```

---

# 270. Siguiente documento

```text
300_DATABASE_CUSTOM_ORM_EXTENSION_SYSTEM.md
```

El siguiente documento deberá definir cómo VoltStack permitirá ampliar el ORM sin crear motores de persistencia paralelos ni romper las garantías de:

```text
EntityManager
UnitOfWork
IdentityMap
Metadata
Hydration
Persistence Engine
Relationship System
Query Engine
Transaction System
```

La arquitectura deberá cubrir:

```text
Custom ORM Extension System
│
├── ORM Extension Identity
├── Extension Descriptor
├── ORM Extension Registry
├── Metadata Extensions
├── Mapping Extensions
├── Entity Lifecycle Extensions
├── Repository Extensions
├── Model API Extensions
├── Query Integration
├── UnitOfWork Extensions
├── Change Tracking Extensions
├── Persistence Extensions
├── Hydration Extensions
├── Relationship Extensions
├── Type/Value Object Integration
├── Lifecycle Event Integration
├── Security Boundaries
├── Transaction Boundaries
├── Cache Integration
├── Persistent Runtime Isolation
├── Conflict Resolution
├── Diagnostics
├── Testing
└── Plugin Integration
```

manteniendo como principio central:

> **Una ORM Extension podrá ampliar metadata, mapping, lifecycle, query ergonomics y puntos de extensión explícitamente soportados, pero no deberá crear un segundo UnitOfWork, una segunda IdentityMap o un motor de persistencia paralelo que compita con el ORM central de VoltStack.**