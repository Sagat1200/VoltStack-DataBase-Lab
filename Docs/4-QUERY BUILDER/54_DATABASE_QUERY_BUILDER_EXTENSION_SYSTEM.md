# 54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## Query Builder Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 54 — Query Builder Extension System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Extensibility  
**Versión:** 1.0

---

# 1. Propósito

`Query Builder Extension System` define la arquitectura mediante la cual VoltStack podrá extender el Query Builder sin romper:

```text
type safety
semantic analysis
query validation
optimizer behavior
planner behavior
compiler boundaries
portability
security
persistent-runtime safety
```

El sistema permitirá añadir:

```text
Builder Macros
Typed Builder Extensions

Custom Expressions
Custom Predicates
Custom Relations
Custom Joins
Custom Aggregates
Custom Window Functions
Custom Set Operations
Custom Query Modifiers

Semantic Query Extensions
Compiler Extensions
Optimizer Extensions
Planner Extensions
```

manteniendo una distinción fundamental:

```text
DX Extension
≠
Semantic Query Extension
```

y:

```text
Semantic Query Extension
≠
Raw SQL Shortcut
```

---

# 2. Regla maestra

Toda extensión deberá pertenecer a una de dos familias principales:

```text
Builder-Level Extension
        │
        └── expands into existing structured query constructs

Semantic Query Extension
        │
        └── introduces new semantic query meaning
```

La primera no crea nueva semántica.

La segunda sí.

---

# 3. Ejemplo de Builder Extension

Supongamos:

```php
$query->whereActive();
```

La implementación puede expandirse a:

```php
$query->where('active', true);
```

Internamente:

```text
whereActive()
      │
      ▼
ComparisonPredicate
├── active
├── =
└── P1
```

No se creó nueva semántica.

Por tanto:

```text
whereActive()
=
DX Macro
```

---

# 4. Ejemplo de Semantic Extension

Supongamos que una base de datos soporta un operador vectorial:

```text
vector_distance(embedding, query_vector)
```

y VoltStack desea entenderlo semánticamente.

La extensión podría registrar:

```text
VectorDistanceExpression
```

con:

```text
input types
result type
nullability
volatility
lineage
capabilities
compiler mapping
optimizer behavior
```

Esto sí introduce nueva semántica.

---

# 5. No responsabilidades

El Extension System no deberá permitir:

```text
arbitrary mutation of core registries at runtime
hidden SQL injection
bypassing validation
bypassing semantic analysis
bypassing capability checks
bypassing fingerprints
global mutable macro state
silent compiler-only semantic extensions
```

---

# 6. Posición arquitectónica

```text
Application / Package
        │
        ▼
Extension Registration
        │
        ▼
Extension Registry
        │
        ▼
Bootstrap Validation
        │
        ▼
Registry Freeze
        │
        ├───────────────────────────┐
        │                           │
        ▼                           ▼
Builder Extensions         Semantic Extensions
        │                           │
        ▼                           ▼
Existing Query Nodes       Custom Query Nodes
        │                           │
        └──────────┬────────────────┘
                   ▼
             Query Model / AST
                   │
                   ▼
             Semantic Engine
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

# 7. Dependencias

Este documento se integra directamente con:

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
33_DATABASE_QUERY_NORMALIZATION_SYSTEM.md
34_DATABASE_QUERY_VALIDATION_SYSTEM.md
35_DATABASE_SEMANTIC_QUERY_ARCHITECTURE.md
36_DATABASE_SEMANTIC_ANALYSIS_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
```

---

# 8. Objetivos

El sistema deberá proporcionar:

- registro de extensiones;
- typed builder extensions;
- macros controladas;
- semantic descriptors;
- custom AST nodes;
- validation handlers;
- semantic analyzers;
- constraint contributors;
- capability descriptors;
- portability metadata;
- optimizer rules;
- planner strategies;
- compiler handlers;
- extension fingerprints;
- extension versioning;
- security policies;
- dependency declarations;
- diagnostics;
- compatibility checks;
- deterministic bootstrap;
- frozen registries;
- persistent-runtime safety.

---

# 9. Taxonomía

VoltStack distinguirá al menos:

```text
QueryBuilderExtension
├── BuilderMacroExtension
├── TypedBuilderExtension
├── SemanticExpressionExtension
├── SemanticPredicateExtension
├── SemanticRelationExtension
├── SemanticJoinExtension
├── SemanticAggregateExtension
├── SemanticWindowExtension
├── SemanticSetOperationExtension
├── QueryModifierExtension
└── CompositeExtension
```

---

# 10. Builder Macro

Un macro es una conveniencia de DX.

Ejemplo:

```php
$query->wherePublished();
```

puede expandirse a:

```php
$query
    ->where('published', true)
    ->whereNull('deleted_at');
```

Después de expansión:

```text
macro disappears
```

---

# 11. Macro lifecycle

```text
Macro Call
   │
   ▼
Macro Handler
   │
   ▼
Standard Builder API
   │
   ▼
Standard Query Model
```

---

# 12. Macro must disappear

Una macro no deberá permanecer como:

```text
MacroNode
```

en el AST, salvo que realmente represente nueva semántica.

---

# 13. Builder Macro ≠ Semantic Node

```text
wherePublished()
```

es DX.

```text
GeoDistanceExpression
```

es semántica.

No deberán mezclarse.

---

# 14. Typed Builder Extension

Preferentemente, las extensiones públicas deberán ofrecer métodos tipados.

Ejemplo:

```php
final class FullTextBuilderExtension
{
    public function whereMatches(
        SelectQueryBuilder $query,
        ColumnReference $column,
        string $term
    ): SelectQueryBuilder {
        // ...
    }
}
```

---

# 15. Avoid excessive __call()

VoltStack no deberá depender principalmente de:

```php
$builder->__call(...)
```

para todas las extensiones.

---

# 16. Reason

El abuso de magic methods reduce:

```text
IDE discoverability
static analysis
type safety
refactorability
documentation quality
error quality
```

---

# 17. Typed integration preferred

VoltStack deberá favorecer:

```text
interfaces
traits
extension facades
generated stubs
typed adapters
```

sobre magia indiscriminada.

---

# 18. Optional macro layer

Macros dinámicas pueden existir para DX, pero serán:

```text
optional
controlled
registry-backed
diagnosable
```

---

# 19. QueryBuilderExtensionId

Toda extensión tendrá identidad estable:

```text
QueryBuilderExtensionId
```

Ejemplo:

```text
voltstack:core:json
vendor:package:vector
acme:database:geo
```

---

# 20. Extension version

Cada extensión deberá declarar:

```text
ExtensionVersion
```

Ejemplo:

```text
1.0.0
2.1.0
```

---

# 21. Identity ≠ class name

La identidad pública de extensión no deberá depender exclusivamente de:

```text
PHP FQCN
```

porque:

```text
classes may move
packages may refactor
serialization needs stable IDs
```

---

# 22. QueryBuilderExtensionDescriptor

Modelo conceptual:

```php
final readonly class QueryBuilderExtensionDescriptor
{
    public function __construct(
        public QueryBuilderExtensionId $id,
        public ExtensionVersion $version,
        public ExtensionKind $kind,
        public ExtensionCapabilitySet $capabilities,
        public ExtensionDependencySet $dependencies,
        public ExtensionCompatibility $compatibility,
        public ExtensionSecurityProfile $security,
        public ExtensionPortability $portability,
        public ExtensionFingerprintContribution $fingerprint,
    ) {}
}
```

---

# 23. ExtensionKind

```php
enum ExtensionKind
{
    case BUILDER_MACRO;
    case TYPED_BUILDER;
    case SEMANTIC_EXPRESSION;
    case SEMANTIC_PREDICATE;
    case SEMANTIC_RELATION;
    case SEMANTIC_JOIN;
    case SEMANTIC_AGGREGATE;
    case SEMANTIC_WINDOW;
    case SEMANTIC_SET_OPERATION;
    case QUERY_MODIFIER;
    case COMPOSITE;
}
```

---

# 24. Extension registration lifecycle

```text
Framework Bootstrap
        │
        ▼
Discover Packages
        │
        ▼
Collect Extension Descriptors
        │
        ▼
Validate Dependencies
        │
        ▼
Validate Conflicts
        │
        ▼
Register
        │
        ▼
Freeze Registries
        │
        ▼
Runtime Query Construction
```

---

# 25. No registration per request

Extensions no deberán registrarse durante cada HTTP request.

---

# 26. Persistent runtime rule

Especialmente bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

el registry deberá quedar:

```text
immutable after bootstrap
```

---

# 27. Extension registry

Podrá existir:

```text
QueryBuilderExtensionRegistry
```

como registry raíz.

---

# 28. Specialized registries

Internamente podrán existir:

```text
ExpressionExtensionRegistry
PredicateExtensionRegistry
RelationExtensionRegistry
AggregateExtensionRegistry
WindowExtensionRegistry
SetOperationExtensionRegistry
CompilerExtensionRegistry
OptimizerExtensionRegistry
```

---

# 29. Root registry ≠ God registry

El registry raíz podrá coordinar, pero no deberá convertirse en:

```text
array<string, mixed>
```

con toda la lógica mezclada.

---

# 30. Registry immutability

Después de:

```text
freeze()
```

toda operación:

```text
register()
replace()
remove()
```

deberá fallar.

---

# 31. Registration conflict

Dos extensiones que intenten registrar:

```text
same semantic ID
+
incompatible version
```

deberán producir error durante bootstrap.

---

# 32. No last-one-wins

Prohibido:

```text
Extension B silently replaces Extension A
```

---

# 33. Extension priorities

Si existen hooks donde varias extensiones participan, puede existir:

```text
priority
```

pero deberá ser:

```text
deterministic
explicit
diagnosable
```

---

# 34. Priority ≠ override

Tener mayor prioridad no autoriza a una extensión a reemplazar semántica core arbitrariamente.

---

# 35. Core semantic invariants protected

Las extensiones no podrán violar:

```text
ORM never generates SQL
Builder never executes SQL
Compiler never executes queries
Connection never knows ORM
AST remains structured
runtime values remain separate
```

---

# 36. Builder extension scope

Una extensión deberá declarar qué builders soporta:

```text
SELECT
INSERT
UPDATE
DELETE
JOIN
SUBQUERY
CTE
SET_OPERATION
AGGREGATE
WINDOW
```

---

# 37. Example scope

```text
FullTextSearchExtension
├── Select Builder
├── Predicate Builder
└── Expression Builder
```

---

# 38. Unsupported builder use

Si una extensión sólo soporta SELECT:

```text
use on UPDATE
```

deberá fallar explícitamente.

---

# 39. Macro registration

Conceptualmente:

```php
$registry->registerMacro(
    id: 'acme:published',
    target: SelectQueryBuilder::class,
    handler: PublishedMacro::class,
);
```

---

# 40. Macro handler

Un handler deberá ser preferentemente:

```text
stateless
dependency-injected
deterministic
```

---

# 41. No captured request closure

Evitar registrar:

```php
$builder->macro('foo', function () use ($request) {
    ...
});
```

en shared bootstrap state.

---

# 42. Macro runtime inputs

Request-local data deberá pasarse:

```text
as method arguments
```

o por integration context explícito.

---

# 43. Macro expansion

Una macro sólo podrá producir:

```text
valid builder operations
valid structured query nodes
```

---

# 44. Macro cannot inject SQL secretly

Prohibido:

```text
wherePublished()
→ hidden whereRaw(...)
```

si la macro se presenta como semantic structured helper.

Si usa Raw:

```text
raw provenance
```

deberá permanecer visible.

---

# 45. Semantic extension

Una semantic extension introduce uno o más nuevos:

```text
ExpressionNode
PredicateNode
RelationSource
JoinKind
AggregateDescriptor
WindowFunctionDescriptor
SetOperationDescriptor
QueryModifier
```

---

# 46. Semantic extension obligations

Una extensión semántica deberá proporcionar soporte para todas las fases necesarias.

Mínimamente:

```text
AST / Query Model
Validation
Semantic Analysis
Capabilities
Fingerprinting
Diagnostics
```

Y, cuando sea ejecutable:

```text
Planner support
Compiler support
```

---

# 47. Optional phase support

Dependiendo del tipo, podrá añadir:

```text
Normalization
Constraint Analysis
Optimizer Rules
Lineage
Dependency Analysis
Portability
Telemetry
```

---

# 48. No compiler-only semantic extension

Esto será inválido:

```text
new semantic node
+
only SQL renderer
```

si no existe forma de validar o interpretar su significado.

---

# 49. Extension phase matrix

Cada descriptor podrá declarar:

| Fase | Estado |
|---|---|
| Construction | Required |
| Validation | Required |
| Semantic Analysis | Required |
| Type Analysis | Optional/Required |
| Constraint Analysis | Optional |
| Optimization | Optional |
| Planning | Required if physical strategy needed |
| Compilation | Required for SQL target |
| Telemetry | Optional |
| Serialization | Required if artifact serializable |

---

# 50. Expression extension

Ejemplo:

```text
VectorDistanceExpression
```

podrá declarar:

```text
arg1: Vector<T,N>
arg2: Vector<T,N>
result: Float
nullable: depends on args
deterministic: yes
volatility: immutable
```

---

# 51. Expression extension model

Conceptualmente:

```php
interface SemanticExpressionExtension
{
    public function descriptor(): SemanticExpressionDescriptor;

    public function validate(
        ExpressionNode $node,
        SemanticValidationContext $context,
    ): void;

    public function analyze(
        ExpressionNode $node,
        SemanticAnalysisContext $context,
    ): ExpressionSemanticInfo;
}
```

---

# 52. Predicate extension

Ejemplo:

```text
JsonContainsPredicate
GeoWithinPredicate
FullTextMatchPredicate
```

---

# 53. Predicate extension obligations

Deberá poder declarar:

```text
truth semantics
null behavior
argument types
dependencies
constraint knowledge
volatility
capabilities
portability
```

---

# 54. Constraint contribution

Un predicate extension podrá opcionalmente producir:

```text
ConstraintFacts
```

---

# 55. No false facts

Si la extensión no puede demostrar una propiedad:

```text
UNKNOWN
```

será preferible a inventar constraints.

---

# 56. Relation extension

Ejemplos:

```text
JSON_TABLE-like relation
remote relation
table-valued function
graph traversal relation
vector search relation
```

---

# 57. Relation extension obligations

Deberá definir:

```text
relation identity
output contract
scope semantics
lineage
dependencies
capabilities
correlation behavior
```

---

# 58. Query relation identity

Una custom relation seguirá produciendo:

```text
RelationId
```

como cualquier otra relation instance.

---

# 59. Join extension

Un extension join podría introducir:

```text
special semantic join kind
```

pero no deberá utilizarse para modelar simplemente un physical algorithm.

---

# 60. Semantic Join ≠ Physical Join

Nunca registrar:

```text
HashJoinType
```

como builder semantic join.

Hash join pertenece al Planner.

---

# 61. Valid custom semantic join

Podría existir, por ejemplo:

```text
TemporalAsOfJoin
```

si representa una semántica relacional distinta y bien definida.

---

# 62. Aggregate extension

Deberá reutilizar la arquitectura definida en:

```text
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
```

registrando:

```text
AggregateFunctionDescriptor
```

---

# 63. Aggregate extension properties

Podrá declarar:

```text
argument rules
result type
nullability
distinct support
filter support
ordering support
decomposability
merge semantics
volatility
```

---

# 64. Window extension

Deberá registrar:

```text
WindowFunctionDescriptor
```

y declarar:

```text
partition requirements
ordering requirements
frame behavior
peer semantics
result type
nullability
```

---

# 65. Set Operation extension

Un custom set operator deberá definir:

```text
operand compatibility
output reconciliation
duplicate semantics
type rules
constraints
optimizer properties
compiler support
```

---

# 66. Query modifier extension

Podrá existir para nuevas propiedades de query como:

```text
special consistency semantics
locking semantics
execution requirements
vendor-independent query hints
```

---

# 67. Modifier ≠ arbitrary metadata

Si un modifier altera semántica:

```text
it must be a typed query construct
```

no simplemente:

```text
metadata['foo'] = ...
```

---

# 68. Extension metadata

Metadata seguirá reservada para información declarativa que no necesite convertirse en un nuevo semantic node.

---

# 69. Normalization extensions

Una semantic extension podrá registrar:

```text
NormalizationRule
```

pero deberá cumplir:

```text
deterministic
idempotent
semantics-preserving
bounded
```

---

# 70. No optimizer logic in normalization

Una extensión no deberá realizar:

```text
cost-based rewrites
join reordering
predicate pushdown
```

desde normalization.

---

# 71. Validation extensions

Cada semantic node deberá tener:

```text
structural validation
semantic validation
```

cuando corresponda.

---

# 72. Structural validation example

Para:

```text
VectorDistance(A, B)
```

puede validar:

```text
exactly 2 operands
```

sin conocer aún sus tipos.

---

# 73. Semantic validation example

Posteriormente:

```text
Vector dimension(A)
=
Vector dimension(B)
```

puede validarse.

---

# 74. Semantic analyzer extension

El analyzer deberá producir información tipada.

Nunca:

```text
array<string,mixed>
```

como contrato principal.

---

# 75. Semantic extension output

Ejemplo:

```php
final readonly class ExtensionExpressionSemanticInfo
{
    public function __construct(
        public QueryType $type,
        public Nullability $nullability,
        public DependencySet $dependencies,
        public LineageSet $lineage,
        public CapabilityRequirementSet $capabilities,
        public Portability $portability,
    ) {}
}
```

---

# 76. Semantic side tables

Custom nodes deberán integrarse con las mismas:

```text
ExpressionSemanticTable
PredicateSemanticTable
RelationSemanticTable
QueryTypeTable
ConstraintGraph
SemanticQueryGraph
```

No deberán crear un universo paralelo.

---

# 77. Extension graph nodes

`SemanticQueryGraph` podrá aceptar:

```text
EXTENSION
```

como node kind con descriptor identity/version.

---

# 78. Extension graph edges

Podrán añadirse edges tipados si el registry los declara.

---

# 79. No untyped graph payload

Evitar:

```text
graphNode['extensionData']
```

sin schema/versioning.

---

# 80. Type System extension

Una query extension podrá depender de un custom type registrado en:

```text
Database Type System
```

---

# 81. Query extension ≠ type extension

Aunque puedan integrarse:

```text
Query Extension
≠
Database Type Extension
```

---

# 82. Capability extension

Un package podrá registrar capabilities nuevas:

```text
QUERY.VECTOR.DISTANCE
QUERY.JSON.CONTAINS
QUERY.GEO.DISTANCE
```

---

# 83. Capability identifiers

Deberán ser globalmente estables.

Ejemplo:

```text
vendor.package.query.vector.distance
```

---

# 84. Capability checks

El Semantic Artifact declarará requirements.

No deberá preguntar directamente:

```text
if PostgreSQL
```

---

# 85. Platform adapters

Cada platform adapter podrá declarar:

```text
supports capability X
with restrictions Y
```

---

# 86. Capability restrictions

Una capability extension podrá requerir un descriptor más rico que bool.

Ejemplo:

```text
VectorDistanceCapability
├── maxDimensions
├── supportedElementTypes
├── operators
└── indexSupport
```

---

# 87. Portability

Cada semantic extension deberá declarar su perfil:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
EXTENSION_SPECIFIC
UNKNOWN
```

---

# 88. Extension portability ≠ query portability

Una query podrá contener múltiples extensiones.

El Query Portability Profile será una combinación de todas.

---

# 89. Optimizer extensions

Un package podrá registrar:

```text
OptimizationRule
```

para sus semantic nodes.

---

# 90. Optimization rule contract

Deberá declarar:

```text
rule ID
version
applicable node types
preconditions
required facts
transformation
semantic proof strategy
budget cost
```

---

# 91. Rule determinism

Misma entrada + mismo semantic context deberá producir:

```text
same transformed result
```

---

# 92. No arbitrary AST mutation

Optimizer extensions deberán usar el mismo transformation framework del Optimizer.

---

# 93. Optimization proof

Cuando una rule cambia estructura, deberá poder justificar:

```text
semantic equivalence
```

de acuerdo con el sistema que se formalizará en documentos 55–57.

---

# 94. Planner extensions

Una semantic extension puede necesitar:

```text
logical planning support
physical planning support
```

---

# 95. Logical plan node

Por ejemplo:

```text
VectorSimilaritySearchLogicalNode
```

podría ser creado por Planner.

---

# 96. Physical plan options

Posteriormente:

```text
NativeVectorIndexScan
BruteForceVectorScan
RemoteVectorSearch
```

podrían ser estrategias físicas distintas.

---

# 97. Semantic feature ≠ physical implementation

Se mantiene:

```text
Vector Similarity Semantics
≠
Vector Index Scan
```

---

# 98. Compiler extensions

Cada target SQL podrá tener:

```text
ExtensionCompilerHandler
```

---

# 99. Compiler handler responsibilities

Sólo deberá:

```text
render target syntax
render target identifiers
render target operators/functions
compile planned strategy
```

---

# 100. Compiler handler shall not

No deberá:

```text
infer types
resolve symbols
re-check relation meaning
invent semantic defaults
perform cost optimization
```

---

# 101. Missing compiler support

Si una semantic extension llega al Compiler sin handler compatible:

```text
UnsupportedExtensionCompilationException
```

---

# 102. No Raw fallback automatically

No deberá ocurrir:

```text
unknown semantic extension
→ dump node to raw SQL
```

---

# 103. Explicit fallback descriptor

Si una extensión soporta una degradación a raw de manera controlada:

```text
it must declare that strategy explicitly
```

y la pérdida semántica deberá ser conocida.

---

# 104. Recommended policy

Para execution normal:

```text
semantic extension without compatible compiler
→ fail
```

---

# 105. Extension dependencies

Una extensión podrá depender de:

```text
other query extensions
custom types
platform capabilities
framework version
database subsystem version
```

---

# 106. ExtensionDependencySet

Conceptualmente:

```php
final readonly class ExtensionDependencySet
{
    /** @var list<ExtensionDependency> */
    public array $dependencies;
}
```

---

# 107. Dependency ranges

Ejemplo:

```text
requires:
    voltstack/database >= 1.2 < 2.0
    acme/vector-type >= 3.0
```

---

# 108. Bootstrap compatibility

Las incompatibilidades deberán descubrirse al bootstrap cuando sea posible.

---

# 109. Runtime target compatibility

Algunas incompatibilidades sólo podrán descubrirse cuando se conozca:

```text
target platform capability snapshot
```

---

# 110. Extension compatibility model

Podrá considerar:

```text
framework version
query model version
semantic graph version
optimizer API version
planner API version
compiler API version
serialization format
```

---

# 111. Stable extension SPI

VoltStack deberá diferenciar:

```text
Public Query API
```

de:

```text
Extension SPI
```

---

# 112. SPI versioning

El Service Provider Interface deberá poseer versioning explícito.

---

# 113. ExtensionSpiVersion

Ejemplo:

```text
query-extension-spi:1
```

---

# 114. No accidental internal API dependency

Los packages no deberán depender de:

```text
private optimizer implementation details
private compiler internals
mutable builder internals
```

si existe un SPI formal.

---

# 115. Public contracts

El SPI deberá exponer:

```text
interfaces
value objects
descriptors
contexts
immutable artifacts
```

---

# 116. Extension security

Toda extensión deberá poder declarar:

```text
SecurityProfile
```

---

# 117. ExtensionSecurityProfile

Puede incluir:

```text
accepts raw SQL?
accepts dynamic identifiers?
can access runtime values?
can introduce relations?
can introduce predicates?
can introduce DML?
sensitive output?
audit requirement?
```

---

# 118. Least privilege

Una Builder Macro que sólo crea predicates no debería recibir:

```text
Connection
Container
Transaction
Request
```

---

# 119. Extension context

Deberán existir context objects limitados.

Ejemplo:

```text
ExpressionExtensionContext
```

en vez de pasar:

```text
entire framework container
```

---

# 120. No service locator

Una extension no deberá depender de:

```php
Container::get(...)
```

de forma arbitraria dentro de semantic analysis.

---

# 121. Deterministic semantic analysis

Semantic extensions deberán evitar:

```text
network I/O
database I/O
clock dependence
randomness
request-global state
```

---

# 122. Explicit external metadata

Si una extensión necesita metadata externa:

```text
VectorIndexMetadataSnapshot
```

deberá ser proporcionada explícitamente por context.

---

# 123. No hidden I/O

Regla:

```text
Semantic Extension
must not silently query
the database or network
```

---

# 124. Extension provenance

Cada node introducido por extensión deberá conservar:

```text
ExtensionId
ExtensionVersion
SourceLocation
Origin
```

---

# 125. Provenance helps

Esto permite:

```text
diagnostics
telemetry
fingerprints
audit
debugging
compatibility
```

---

# 126. Fingerprinting

Toda extensión que afecte query semantics deberá contribuir al fingerprint.

---

# 127. Structural fingerprint contribution

Podrá incluir:

```text
ExtensionId
ExtensionVersion
NodeKind
NodeStructure
Semantic Configuration
```

---

# 128. Semantic fingerprint contribution

Además:

```text
resolved capabilities
resolved types
resolved semantic modes
schema-dependent extension metadata
```

---

# 129. No runtime values in semantic fingerprint

Se mantiene:

```text
binding values
≠
semantic fingerprint
```

---

# 130. Extension version significance

Cambiar:

```text
1.0 → 1.1
```

no necesariamente invalidará caches si no cambia semántica.

Por ello puede existir:

```text
SemanticVersionKey
```

separado de package version.

---

# 131. SemanticVersionKey

Ejemplo:

```text
extension package: 2.4.1
semantic model: 3
```

El segundo será el que contribuya a semantic fingerprints cuando corresponda.

---

# 132. Extension serialization

Custom nodes deberán ser serializables mediante formatos versionados.

---

# 133. No arbitrary PHP serialization required

VoltStack no deberá depender obligatoriamente de:

```text
serialize(object graph)
```

---

# 134. Extension node serializer

Una extensión podrá registrar:

```text
ExtensionNodeCodec
```

---

# 135. Codec responsibilities

```text
encode immutable node
decode validated node
version format
validate extension identity
```

---

# 136. Unknown serialized extension

Si se carga un artifact que contiene:

```text
extension X
```

y X no está instalada:

```text
artifact cannot be executable
```

---

# 137. Safe diagnostic

Podrá emitirse:

```text
Missing Query Extension:
    acme/vector-query
```

---

# 138. No silent Raw conversion

Un serialized custom node desconocido nunca se convertirá automáticamente a Raw.

---

# 139. Extension diagnostics

Los errors deberán incluir:

```text
ExtensionId
ExtensionVersion
NodeId
SemanticPhase
SourceLocation
Capability
TargetPlatform
```

cuando sea posible.

---

# 140. Example diagnostic

```text
Query extension "acme/vector"
cannot analyze expression VX12.

Expected:
    Vector<Float32, 1536>

Actual:
    Vector<Float32, 768>

Expression:
    cosine_distance(document.embedding, P1)
```

---

# 141. Extension boot diagnostic

```text
Extension conflict detected.

Semantic ID:
    query.geo.distance

Provider A:
    acme/geo 1.4

Provider B:
    example/spatial 2.1

VoltStack does not allow silent replacement
of semantic query extensions.
```

---

# 142. Extension compiler diagnostic

```text
Query requires extension:

    acme.vector.cosine-distance

Target platform:
    SQLite

No compiler or semantics-preserving
emulation strategy is registered.
```

---

# 143. Extension testing requirements

Una semantic extension deberá proporcionar tests de:

```text
builder construction
AST node construction
validation
semantic analysis
type rules
nullability
lineage
dependencies
capabilities
portability
fingerprints
serialization
optimizer integration
planner integration
compiler integration
persistent runtime
concurrency
```

---

# 144. Conformance suite

VoltStack deberá proporcionar un:

```text
QueryExtensionConformanceTestSuite
```

---

# 145. Conformance tests

Podrán verificar automáticamente:

```text
stable extension ID
registry freeze compatibility
deterministic analysis
immutable output
fingerprint stability
no runtime resource retention
codec roundtrip
capability declarations
```

---

# 146. Architecture test

El framework deberá detectar si una extension semantic analyzer importa directamente:

```text
PDO
Connection
Driver
Executor
Request
```

cuando no debería.

---

# 147. Builder macro test

Un macro deberá demostrar:

```text
same output as equivalent structured builder calls
```

---

# 148. Extension performance

El sistema deberá evitar que una extensión pueda introducir trabajo ilimitado.

---

# 149. Extension budget

Podrá existir:

```text
QueryExtensionBudget
```

---

# 150. Budget dimensions

Ejemplos:

```text
max extension nodes
max extension nesting
max extension normalization rules
max extension optimizer rule applications
max extension diagnostics
max extension analysis work
```

---

# 151. Budget ownership

El Query Engine global deberá imponer el budget.

Una extensión no deberá poder deshabilitarlo.

---

# 152. No self-granted unlimited budget

Prohibido:

```text
extension.setUnlimitedAnalysis()
```

---

# 153. Extension cost accounting

Cada phase hook podrá contribuir a:

```text
AnalysisBudgetCounter
```

---

# 154. Persistent runtime safety

Compartible:

```text
frozen descriptors
frozen registries
stateless extension services
immutable codecs
immutable semantic artifacts
```

Operation-local:

```text
builder extension state
semantic work state
diagnostics
optimizer rule state
planner candidate state
compiler invocation state
```

---

# 155. No mutable extension singleton

Prohibido:

```php
final class MyExtension
{
    public array $currentQuery = [];
}
```

si la instancia es shared.

---

# 156. Stateless extensions preferred

La mayoría de extensiones deberán ser:

```text
stateless services
```

---

# 157. Scoped state

Si una extensión requiere estado temporal:

```text
ExtensionAnalysisState
```

deberá ser operation-scoped.

---

# 158. Concurrent extension use

Una misma extension shared podrá participar simultáneamente en:

```text
Query A
Query B
Query C
```

sin contaminación cruzada.

---

# 159. Extension discovery

VoltStack podrá descubrir extensiones vía:

```text
package manifests
service providers
compiled extension manifest
manual registration
```

---

# 160. Recommended production model

Producción deberá favorecer:

```text
compiled extension manifest
```

para reducir reflection/discovery overhead.

---

# 161. Extension manifest

Puede contener:

```text
Extension IDs
Versions
Handlers
Capabilities
Dependencies
SPI Version
Fingerprint Keys
```

---

# 162. Manifest cache

Podrá cachearse entre workers siempre que sea:

```text
immutable
versioned
deployment-specific
```

---

# 163. No request-local package discovery

No deberá escanearse Composer/packages en cada query.

---

# 164. Framework integration

El Container podrá registrar servicios de extensión durante bootstrap.

Pero Query Builder no deberá pedir el Container durante cada method call.

---

# 165. Resolved extension services

Los factories podrán recibir registries ya resueltos.

---

# 166. Extension facades

Puede existir una facade de DX:

```php
DB::extensions()
```

para inspección/configuración.

No deberá ser el mecanismo principal de semantic state.

---

# 167. Public inspection

Developer tooling podrá mostrar:

```text
Installed Query Extensions

acme/vector
  version: 2.1
  semantic-version: 3
  nodes:
    VectorDistanceExpression
    VectorSimilarityPredicate

supports:
    PostgreSQL + pgvector
```

---

# 168. Query explain integration

Una query explain podrá indicar:

```text
Extension Node VX4
Provider:
    acme/vector

Capability:
    QUERY.VECTOR.COSINE_DISTANCE

Semantic Type:
    Float64
```

---

# 169. Telemetry

Podrán registrarse métricas como:

```text
database.query.extension.count
database.query.extension.analysis_time
database.query.extension.compile_time
database.query.extension.failures
```

---

# 170. Telemetry attribution

Cada measurement deberá incluir:

```text
ExtensionId
SemanticVersion
Phase
```

sin exponer data sensible.

---

# 171. Security audit

Tooling podrá generar:

```text
Query Extension Security Report
```

con:

```text
extensions using raw
extensions introducing DML
extensions platform-specific
extensions requiring dynamic identifiers
extensions marked security-sensitive
```

---

# 172. Raw integration

Una extension puede utilizar Raw internamente sólo si:

```text
its descriptor declares it
```

y los Raw barriers correspondientes se conservan cuando aplique.

---

# 173. Semantic extension with compiler Raw output

Que el Compiler finalmente genere SQL textual no convierte la extension en Raw.

La diferencia está en que antes del Compiler VoltStack entiende completamente:

```text
meaning
types
constraints
capabilities
```

---

# 174. Key distinction

```text
Semantic Extension
→ understood until compilation

Raw Fragment
→ partially or fully opaque before compilation
```

---

# 175. Macro using semantic extension

Una macro podrá expandirse a un custom semantic node.

Ejemplo:

```text
nearestTo($vector)
```

puede ser DX sobre:

```text
VectorDistanceExpression
```

---

# 176. Layering

```text
Public Macro
      │
      ▼
Typed Extension API
      │
      ▼
Semantic Extension Node
      │
      ▼
Semantic Engine
```

---

# 177. ORM integration

ORM podrá ofrecer domain-specific extensions.

Ejemplo:

```text
whereRelated(...)
whereHas(...)
```

pero deberá bajar a:

```text
core query structures
or semantic query extensions
```

antes de Semantic Analysis.

---

# 178. No ORM-only compiler extension

ORM no deberá crear una ruta:

```text
ORM Custom Query
→ ORM SQL Compiler
```

independiente del Query Engine.

---

# 179. Multitenancy integration

Un package Multitenancy podrá registrar builder helpers.

Ejemplo:

```text
forTenant(...)
```

pero normalmente deberá expandirse a:

```text
structured query context
structured predicates
```

---

# 180. Authorization integration

Una extension de authorization puede insertar predicates con:

```text
MANDATORY provenance
```

pero no deberá bypassear Semantic Engine.

---

# 181. Security-sensitive extension

Podrá declararse:

```text
SecurityCriticalExtension
```

lo que permite políticas como:

```text
must be installed
cannot be disabled at runtime
must match expected semantic version
```

---

# 182. Extension replacement

Hot-swapping de semantic extensions dentro de un worker activo deberá estar prohibido.

---

# 183. Deployment update

Cambiar extensiones requiere:

```text
new bootstrap lifecycle
```

y normalmente nuevos workers.

---

# 184. Extension deprecation

VoltStack deberá permitir marcar:

```text
extension APIs
semantic nodes
builder methods
```

como deprecated.

---

# 185. Deprecation metadata

Podrá incluir:

```text
since
replacement
removal version
migration instructions
```

---

# 186. Query artifact compatibility

Un artifact cacheado con una semantic extension deprecated puede seguir siendo válido mientras:

```text
semantic fingerprint compatibility
```

se mantenga.

---

# 187. Extension removal

Si una extension desaparece:

```text
cached artifacts requiring it
```

deberán invalidarse.

---

# 188. Fingerprint makes this safe

La presencia de:

```text
ExtensionId
SemanticVersion
```

en el artifact evita reuse incorrecto.

---

# 189. Extension ordering

Si varias builder macros/processors actúan en la misma phase:

```text
order must be deterministic
```

---

# 190. Recommended ordering

```text
Core
→ Framework official extensions
→ Application extensions
```

pero sólo donde exista una pipeline extensible explícita.

---

# 191. No arbitrary interception

Una extensión no deberá poder interceptar cualquier query builder method globalmente sin declarar:

```text
hook
scope
priority
effect
```

---

# 192. Extension hooks

Hooks posibles:

```text
builder method registration
normalization rule registration
semantic pass registration
constraint contributor
optimizer rule
planner strategy
compiler handler
diagnostic formatter
```

---

# 193. Hook registry

Cada hook family deberá tener un registry especializado.

---

# 194. Extension semantic pass

Una extensión podrá añadir un semantic pass.

Pero deberá declarar:

```text
inputs
outputs
dependencies
ordering constraints
fixpoint membership
budget class
```

---

# 195. Pass DAG integration

Los semantic passes de extensiones deberán integrarse al:

```text
Semantic Pass DAG
```

definido en el documento 36.

---

# 196. No implicit pass ordering

No:

```text
registration order = semantic execution order
```

salvo que se defina explícitamente.

---

# 197. Dependency declaration

Ejemplo:

```text
VectorTypeResolutionPass
depends on:
    SymbolResolution
    SchemaResolution
```

---

# 198. Fixpoint extensions

Si un extension pass participa en un fixpoint:

```text
must declare monotonicity expectations
iteration budget
convergence key
```

---

# 199. No infinite extension loop

El coordinator deberá cortar:

```text
non-converging extension analysis
```

con diagnóstico.

---

# 200. Query Builder Extension API example

```php
final class JsonQueryExtensionProvider
{
    public function register(
        QueryExtensionRegistry $registry
    ): void {
        $registry->register(
            JsonContainsExtension::descriptor()
        );

        $registry->registerBuilderMethod(
            JsonWhereContainsMethod::class
        );

        $registry->registerCompiler(
            PostgreSqlJsonCompiler::class
        );

        $registry->registerCompiler(
            MySqlJsonCompiler::class
        );
    }
}
```

---

# 201. Builder usage

```php
$query = DB::table('products')
    ->whereJsonContains(
        'attributes',
        ['color' => 'red']
    );
```

---

# 202. Internal structured representation

```text
JsonContainsPredicate
├── left
│   └── products.attributes
├── right
│   └── P1
└── semantic operator
    └── json:contains
```

---

# 203. Semantic Analysis

Podrá resolver:

```text
left type:
    JsonDocument

right type:
    JsonValue

result:
    BooleanPredicate

capability:
    QUERY.JSON.CONTAINS
```

---

# 204. Platform compilation

PostgreSQL compiler extension puede producir una estrategia.

MySQL compiler extension puede producir otra.

SQLite puede:

```text
reject
or
use an explicitly registered emulation
```

---

# 205. Developer API remains stable

La aplicación escribe:

```text
whereJsonContains(...)
```

sin conocer SQL vendor-specific.

---

# 206. This is superior to raw

Comparado con:

```php
whereRaw('JSON_CONTAINS(...)')
```

la semantic extension permite:

```text
type checking
portability analysis
optimizer understanding
capability checks
better diagnostics
lineage
security analysis
```

---

# 207. Extension directory structure propuesta

```text
Query/
└── Extension/
    ├── Contract/
    │   ├── QueryBuilderExtension.php
    │   ├── SemanticQueryExtension.php
    │   ├── ExtensionProvider.php
    │   ├── ExtensionCodec.php
    │   └── ExtensionConformanceContract.php
    │
    ├── Core/
    │   ├── QueryBuilderExtensionId.php
    │   ├── ExtensionVersion.php
    │   ├── ExtensionKind.php
    │   ├── QueryBuilderExtensionDescriptor.php
    │   ├── ExtensionCompatibility.php
    │   ├── ExtensionSecurityProfile.php
    │   └── ExtensionPortability.php
    │
    ├── Registry/
    │   ├── QueryBuilderExtensionRegistry.php
    │   ├── BuilderMacroRegistry.php
    │   ├── SemanticExtensionRegistry.php
    │   ├── ExtensionHookRegistry.php
    │   └── ExtensionRegistryFreezer.php
    │
    ├── Builder/
    │   ├── BuilderMacroExtension.php
    │   ├── TypedBuilderExtension.php
    │   ├── BuilderExtensionMethod.php
    │   └── BuilderExtensionResolver.php
    │
    ├── Semantic/
    │   ├── SemanticExpressionExtension.php
    │   ├── SemanticPredicateExtension.php
    │   ├── SemanticRelationExtension.php
    │   ├── SemanticJoinExtension.php
    │   ├── SemanticAggregateExtension.php
    │   ├── SemanticWindowExtension.php
    │   ├── SemanticSetOperationExtension.php
    │   └── QueryModifierExtension.php
    │
    ├── Validation/
    │   ├── ExtensionValidator.php
    │   ├── ExtensionDependencyValidator.php
    │   ├── ExtensionConflictValidator.php
    │   └── ExtensionCompatibilityValidator.php
    │
    ├── SemanticPass/
    │   ├── ExtensionSemanticPass.php
    │   └── ExtensionSemanticPassRegistry.php
    │
    ├── Constraint/
    │   └── ExtensionConstraintContributor.php
    │
    ├── Optimizer/
    │   ├── ExtensionOptimizationRule.php
    │   └── ExtensionOptimizationRuleRegistry.php
    │
    ├── Planner/
    │   ├── ExtensionLogicalPlanner.php
    │   ├── ExtensionPhysicalPlanner.php
    │   └── ExtensionPlanningStrategyRegistry.php
    │
    ├── Compiler/
    │   ├── ExtensionCompiler.php
    │   └── ExtensionCompilerRegistry.php
    │
    ├── Capability/
    │   ├── ExtensionCapabilityDescriptor.php
    │   └── ExtensionCapabilityRegistry.php
    │
    ├── Serialization/
    │   ├── ExtensionNodeCodec.php
    │   └── ExtensionCodecRegistry.php
    │
    ├── Fingerprint/
    │   └── ExtensionFingerprintContributor.php
    │
    ├── Manifest/
    │   ├── QueryExtensionManifest.php
    │   ├── QueryExtensionManifestCompiler.php
    │   └── QueryExtensionManifestLoader.php
    │
    ├── Budget/
    │   └── QueryExtensionBudget.php
    │
    ├── Diagnostic/
    │   └── ExtensionDiagnostic.php
    │
    └── Exception/
        ├── QueryExtensionException.php
        ├── ExtensionConflictException.php
        ├── ExtensionDependencyException.php
        ├── ExtensionCompatibilityException.php
        ├── MissingQueryExtensionException.php
        ├── UnsupportedExtensionCompilationException.php
        └── QueryExtensionBudgetExceededException.php
```

---

# 208. Ownership matrix

| Concern | Owner |
|---|---|
| Builder macro registration | Builder Extension Registry |
| Typed Builder methods | Typed Extension Layer |
| Custom semantic node | Semantic Extension |
| Structural validation | Extension Validator |
| Semantic resolution | Extension Semantic Analyzer |
| Type rules | Query Type System + extension |
| Constraint facts | Constraint contributor |
| Capability requirements | Extension descriptor |
| Portability | Extension semantic profile |
| Optimizer rewrites | Extension Optimization Rule |
| Logical planning | Planner extension |
| Physical planning | Planner extension |
| SQL generation | Compiler extension |
| Serialization | Extension codec |
| Fingerprinting | Extension fingerprint contributor |
| Security | Extension security profile |
| Runtime registry lifecycle | Bootstrap/Registry freezer |
| Runtime state isolation | Query/operation scope |

---

# 209. Invariantes arquitectónicos

## DB-QB-EXT-001
DX Macro será distinto de Semantic Query Extension.

## DB-QB-EXT-002
Semantic Query Extension será distinta de Raw SQL.

## DB-QB-EXT-003
Macros deberán expandirse a query constructs válidos.

## DB-QB-EXT-004
Macros no permanecerán como semantic nodes salvo nueva semántica real.

## DB-QB-EXT-005
Typed extensions serán preferidas sobre magic `__call`.

## DB-QB-EXT-006
Dynamic macros serán registry-backed.

## DB-QB-EXT-007
Toda extensión tendrá identity estable.

## DB-QB-EXT-008
Extension identity será distinta de PHP class name.

## DB-QB-EXT-009
Toda extensión tendrá version.

## DB-QB-EXT-010
Semantic-changing versions afectarán fingerprints.

## DB-QB-EXT-011
Registries se congelarán después del bootstrap.

## DB-QB-EXT-012
No habrá extension registration por request.

## DB-QB-EXT-013
No habrá last-one-wins conflict resolution.

## DB-QB-EXT-014
Conflicts fallarán explícitamente.

## DB-QB-EXT-015
Extension priorities serán deterministas.

## DB-QB-EXT-016
Priority no permitirá violar core invariants.

## DB-QB-EXT-017
Extensions declararán builder scopes.

## DB-QB-EXT-018
Unsupported builder usage fallará.

## DB-QB-EXT-019
Macro handlers no capturarán request state en shared registry.

## DB-QB-EXT-020
Macro runtime data será argumento/context explícito.

## DB-QB-EXT-021
Macros no ocultarán Raw sin provenance.

## DB-QB-EXT-022
Semantic extensions introducirán nodes tipados.

## DB-QB-EXT-023
Semantic extensions declararán structural support.

## DB-QB-EXT-024
Semantic extensions declararán validation support.

## DB-QB-EXT-025
Semantic extensions declararán semantic analysis support.

## DB-QB-EXT-026
Executable semantic extensions declararán compiler support.

## DB-QB-EXT-027
Planner support será obligatorio cuando se requiera estrategia física especial.

## DB-QB-EXT-028
No habrá compiler-only semantic nodes.

## DB-QB-EXT-029
Expression extensions integrarán Query Type System.

## DB-QB-EXT-030
Predicate extensions declararán truth/null semantics.

## DB-QB-EXT-031
Predicate extensions no inventarán constraints.

## DB-QB-EXT-032
Relation extensions producirán RelationId.

## DB-QB-EXT-033
Relation extensions declararán output contracts.

## DB-QB-EXT-034
Relation extensions declararán scope semantics.

## DB-QB-EXT-035
Semantic joins serán distintos de physical joins.

## DB-QB-EXT-036
Hash/Merge/NestedLoop no serán builder join extensions.

## DB-QB-EXT-037
Aggregate extensions reutilizarán Aggregate System.

## DB-QB-EXT-038
Window extensions reutilizarán Window System.

## DB-QB-EXT-039
Set extensions reutilizarán Set Operation System.

## DB-QB-EXT-040
Semantic modifiers serán typed constructs.

## DB-QB-EXT-041
Normalization extensions serán deterministic.

## DB-QB-EXT-042
Normalization extensions serán idempotent.

## DB-QB-EXT-043
Normalization extensions no serán cost-based.

## DB-QB-EXT-044
Semantic extension info vivirá en side tables apropiadas.

## DB-QB-EXT-045
Extensions no crearán semantic engine paralelo.

## DB-QB-EXT-046
SemanticQueryGraph podrá representar extension nodes.

## DB-QB-EXT-047
Extension graph payload será versionado/tipado.

## DB-QB-EXT-048
Query extensions serán distintas de type extensions.

## DB-QB-EXT-049
Capabilities serán stable IDs.

## DB-QB-EXT-050
Capabilities reemplazarán vendor branching.

## DB-QB-EXT-051
Capability descriptors podrán incluir restricciones.

## DB-QB-EXT-052
Portability será explícita.

## DB-QB-EXT-053
Optimizer extensions usarán Optimizer framework.

## DB-QB-EXT-054
Optimizer extensions no mutarán AST arbitrariamente.

## DB-QB-EXT-055
Optimization transformations requerirán equivalencia semántica.

## DB-QB-EXT-056
Planner extensions distinguirán logical de physical nodes.

## DB-QB-EXT-057
Compiler extensions sólo renderizarán target representation.

## DB-QB-EXT-058
Compiler extensions no resolverán tipos.

## DB-QB-EXT-059
Compiler extensions no resolverán símbolos.

## DB-QB-EXT-060
Missing compiler support fallará explícitamente.

## DB-QB-EXT-061
Semantic extensions no caerán automáticamente a Raw.

## DB-QB-EXT-062
Explicit Raw fallback deberá declararse.

## DB-QB-EXT-063
Extension dependencies serán explícitas.

## DB-QB-EXT-064
Extension compatibility será validada.

## DB-QB-EXT-065
SPI tendrá versioning.

## DB-QB-EXT-066
Extensions preferirán public SPI sobre internals.

## DB-QB-EXT-067
Security profiles serán explícitos.

## DB-QB-EXT-068
Extension contexts seguirán least privilege.

## DB-QB-EXT-069
Semantic extensions no recibirán full Container por default.

## DB-QB-EXT-070
Semantic extensions no harán hidden DB I/O.

## DB-QB-EXT-071
Semantic extensions no harán hidden network I/O.

## DB-QB-EXT-072
External metadata será snapshot explícito.

## DB-QB-EXT-073
Extension provenance será explícita.

## DB-QB-EXT-074
Semantic extensions contribuirán al fingerprint.

## DB-QB-EXT-075
Runtime bindings estarán fuera de semantic fingerprints.

## DB-QB-EXT-076
Semantic version podrá diferir de package version.

## DB-QB-EXT-077
Extension serialization será versionada.

## DB-QB-EXT-078
No se requerirá PHP object serialization arbitraria.

## DB-QB-EXT-079
Missing serialized extension impedirá execution.

## DB-QB-EXT-080
Unknown serialized nodes no se convertirán en Raw.

## DB-QB-EXT-081
Extension diagnostics identificarán extension/provider.

## DB-QB-EXT-082
Semantic extensions tendrán conformance tests.

## DB-QB-EXT-083
Builder macros tendrán equivalence tests.

## DB-QB-EXT-084
Extensions estarán sujetas a budgets globales.

## DB-QB-EXT-085
Extensions no podrán autootorgarse budget ilimitado.

## DB-QB-EXT-086
Shared extension services serán stateless o immutable.

## DB-QB-EXT-087
Mutable extension state será operation-scoped.

## DB-QB-EXT-088
Concurrent queries no compartirán mutable extension state.

## DB-QB-EXT-089
Extension discovery ocurrirá en bootstrap.

## DB-QB-EXT-090
Production podrá usar compiled extension manifest.

## DB-QB-EXT-091
No habrá package discovery por query.

## DB-QB-EXT-092
Builder no consultará Container para cada extension call.

## DB-QB-EXT-093
Extension tooling podrá inspeccionar installed extensions.

## DB-QB-EXT-094
Telemetry attribution incluirá ExtensionId.

## DB-QB-EXT-095
Telemetry no expondrá sensitive values.

## DB-QB-EXT-096
Raw usage interno de una extension será declarable/auditable.

## DB-QB-EXT-097
Semantic extension compilada a SQL no será Raw por definición.

## DB-QB-EXT-098
ORM reutilizará Query Extension System.

## DB-QB-EXT-099
ORM no creará compiler paralelo.

## DB-QB-EXT-100
Multitenancy extensions bajarán a structured semantics.

## DB-QB-EXT-101
Authorization extensions no bypassearán Semantic Engine.

## DB-QB-EXT-102
Security-critical extensions podrán marcarse obligatorias.

## DB-QB-EXT-103
Hot-swapping semantic extensions en workers activos estará prohibido.

## DB-QB-EXT-104
Cambios de extensions requerirán nuevo bootstrap lifecycle.

## DB-QB-EXT-105
Deprecation será explícita.

## DB-QB-EXT-106
Removal invalidará artifacts dependientes.

## DB-QB-EXT-107
Extension ordering será determinista.

## DB-QB-EXT-108
Extensions no podrán interceptar cualquier query arbitrariamente.

## DB-QB-EXT-109
Hooks serán explícitos y tipados.

## DB-QB-EXT-110
Semantic extension passes integrarán el Pass DAG.

## DB-QB-EXT-111
Pass ordering no dependerá accidentalmente del registration order.

## DB-QB-EXT-112
Fixpoint extension passes serán bounded.

## DB-QB-EXT-113
Non-converging extensions fallarán explícitamente.

## DB-QB-EXT-114
Query Builder extensions permanecerán independientes de Connection.

## DB-QB-EXT-115
Query Builder extensions permanecerán independientes de Executor.

## DB-QB-EXT-116
Query Builder extensions permanecerán independientes de ORM persistence.

## DB-QB-EXT-117
Query Extension System preservará immutable final artifacts.

## DB-QB-EXT-118
Query Extension System será persistent-runtime safe.

## DB-QB-EXT-119
Query Extension System será coroutine-safe.

## DB-QB-EXT-120
Extensibilidad nunca romperá las fronteras fundamentales del Query Engine.

---

# 210. Anti-patterns

## 210.1 Macro as hidden SQL

```php
$query->wherePublished()
```

implementado como:

```php
return $query->whereRaw('...');
```

sin provenance.

**Rechazado.**

---

## 210.2 Everything through __call()

```php
public function __call($name, $arguments)
{
    // all query extension behavior
}
```

como arquitectura principal.

**Rechazado.**

---

## 210.3 Global mutable macro registry

```php
static array $macros;
```

modificado durante requests.

**Rechazado.**

---

## 210.4 Compiler-only semantic feature

```text
CustomNode
+
SQL renderer only
```

**Rechazado.**

---

## 210.5 Semantic analyzer opening connection

```php
$connection->query(...)
```

durante semantic analysis.

**Rechazado.**

---

## 210.6 Full container service locator

```php
Container::get(...)
```

desde cualquier extension pass.

**Rechazado.**

---

## 210.7 Silent extension override

```text
last provider wins
```

**Rechazado.**

---

## 210.8 Physical strategy exposed as builder semantics

```php
$query->hashJoin(...)
```

como semantic join.

**Rechazado.**

---

## 210.9 Unknown extension to Raw fallback

**Rechazado.**

---

## 210.10 Package version as sole semantic cache version

**Rechazado.**

Se deberá poder versionar la semántica de forma independiente.

---

# 211. Ejemplo integral — Vector Search Extension

Supongamos un paquete:

```text
voltstack/vector-query
```

que añade búsqueda por similitud vectorial.

---

# 212. Extension descriptor

```text
ExtensionId:
    voltstack:vector-query

Package Version:
    1.4.0

Semantic Version:
    2

Features:
    VectorDistanceExpression
    VectorSimilarityPredicate
    nearestTo() builder helper
```

---

# 213. Builder API

```php
$query = DB::table('documents as d')
    ->select('d.id')
    ->select('d.title')
    ->selectAs(
        Vector::cosineDistance(
            'd.embedding',
            $queryVector
        ),
        'distance'
    )
    ->where(
        Vector::cosineDistance(
            'd.embedding',
            $queryVector
        ),
        '<',
        0.25
    )
    ->orderBy('distance')
    ->limit(10);
```

También podría ofrecer DX:

```php
$query->nearestTo(
    column: 'd.embedding',
    vector: $queryVector,
    limit: 10,
);
```

---

# 214. Macro expansion

`nearestTo()` podría expandirse a:

```text
VectorDistanceExpression
+
Ordering
+
Limit
```

No a Raw.

---

# 215. Query Model

```text
SelectQuery
│
├── FROM documents AS d
│
├── PROJECTION
│   ├── d.id
│   ├── d.title
│   └── VectorDistanceExpression VX1 AS distance
│
├── WHERE
│   └── VX2 < P2
│
├── ORDER
│   └── distance ASC
│
└── LIMIT
    └── 10
```

---

# 216. Parameter definitions

```text
P1
type:
    Vector<Float32, 1536>

P2
type:
    Float
```

Bindings:

```text
P1 → query vector
P2 → 0.25
```

---

# 217. Semantic info

```text
VX1
├── function
│   └── vector.cosine_distance
├── left
│   └── documents.embedding
├── right
│   └── P1
├── result type
│   └── Float64
├── nullability
│   └── depends on arguments
├── deterministic
│   └── yes
├── capability
│   └── QUERY.VECTOR.COSINE_DISTANCE
└── provenance
    └── voltstack:vector-query
```

---

# 218. Type validation

Si:

```text
documents.embedding
=
Vector<Float32, 1536>
```

y:

```text
P1
=
Vector<Float32, 768>
```

Semantic Analysis deberá rechazar la query.

---

# 219. Capability resolution

PostgreSQL + compatible vector extension:

```text
QUERY.VECTOR.COSINE_DISTANCE
→ NATIVE
```

SQLite sin extension:

```text
QUERY.VECTOR.COSINE_DISTANCE
→ UNSUPPORTED
```

---

# 220. Optimizer knowledge

El Optimizer puede entender que:

```text
ORDER BY cosine_distance(...)
LIMIT 10
```

es candidato para una:

```text
NearestNeighborOptimization
```

si la extension registra una rule.

---

# 221. Planner strategy

Puede elegir:

```text
NativeVectorIndexScan
```

si existe índice/capability apropiado.

O:

```text
RegularScan + Distance Evaluation + Sort + Limit
```

si es semánticamente válido.

---

# 222. Compiler

PostgreSQL compiler extension podrá generar la sintaxis concreta para su plataforma.

El Builder y Semantic Engine nunca necesitan conocerla.

---

# 223. Benefit

La aplicación obtiene:

```text
Laravel-like DX
+
type safety
+
semantic understanding
+
platform capabilities
+
optimizer integration
+
planner integration
+
portable extension architecture
```

en vez de:

```text
whereRaw(...)
orderByRaw(...)
```

---

# 224. Fórmula de Builder Macro

```text
Builder Macro
=
Developer Convenience
+
Deterministic Expansion
→ Existing Structured Query Constructs
```

---

# 225. Fórmula de Semantic Extension

```text
Semantic Query Extension
=
Stable Semantic Identity
+
Typed Query Node
+
Validation Rules
+
Semantic Analysis
+
Type Rules
+
Capability Requirements
+
Portability
+
Fingerprinting
+
Optional Constraints
+
Optional Optimizer Rules
+
Planning Support
+
Compilation Support
```

---

# 226. Fórmula de seguridad

```text
Safe Query Extension
=
Explicit Extension Identity
+
Least-Privilege Context
+
No Hidden I/O
+
Typed Inputs
+
Structured Parameters
+
Capability Validation
+
Security Profile
+
Deterministic Analysis
+
Resource Budgets
```

---

# 227. Fórmula de compatibility

```text
Extension Compatibility
=
SPI Compatibility
+
Semantic Version Compatibility
+
Dependency Compatibility
+
Capability Compatibility
+
Target Compiler Compatibility
```

---

# 228. Fórmula de persistent-runtime safety

```text
Persistent-Safe Extension System
=
Frozen Registries
+
Immutable Descriptors
+
Stateless Shared Services
+
Operation-Scoped Mutable State
+
No Runtime Registration
+
No Request-Captured Closures
+
No Runtime Resource Retention
```

---

# 229. Frontera final

```text
Developer Convenience
        │
        ▼
    DX Macro
        │
        ▼
Existing Structured Query Nodes
```

o:

```text
New Query Meaning
        │
        ▼
Semantic Query Extension
        │
        ▼
Custom Structured Node
        │
        ▼
Validation
        │
        ▼
Semantic Analysis
        │
        ▼
Constraint / Graph Integration
        │
        ▼
Optimizer
        │
        ▼
Planner
        │
        ▼
Compiler Extension
```

Nunca:

```text
Extension
   │
   ▼
Hidden Raw SQL
   │
   ▼
Bypass Query Engine
```

---

# 230. Decisión arquitectónica final

VoltStack adopta un modelo de extensibilidad en dos niveles:

```text
Level 1
DX / Builder Extension

Level 2
Semantic Query Extension
```

El primero:

```text
does not introduce new semantics
```

El segundo:

```text
must participate in the semantic pipeline
```

Esto permite ofrecer APIs de alto nivel extremadamente productivas sin sacrificar la arquitectura interna.

La regla final será:

> **Una extensión de Query Builder en VoltStack nunca deberá ser una puerta trasera para saltarse el Query Engine. Si sólo mejora la ergonomía, deberá expandirse a construcciones existentes; si introduce nuevo significado, deberá convertirse en una extensión semántica completa capaz de participar en Validation, Semantic Analysis, Capabilities, Optimization, Planning, Compilation y Fingerprinting.**

---

# 231. Cierre formal del Bloque 4 — Query Builder

Con este documento queda completado:

```text
Block 4 — Query Builder

43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
44_DATABASE_SELECT_QUERY_BUILDER.md
45_DATABASE_INSERT_QUERY_BUILDER.md
46_DATABASE_UPDATE_QUERY_BUILDER.md
47_DATABASE_DELETE_QUERY_BUILDER.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md
```

El bloque ya cubre:

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

Aggregations
Grouping
HAVING
Grouping Sets
ROLLUP
CUBE

Window Functions
Frames
Named Windows

Raw Escape Hatches

Builder Macros
Semantic Query Extensions
```

---

# 232. Arquitectura consolidada del Query Builder

```text
Developer API
     │
     ▼
Query Builder
     │
     ├── Select
     ├── Insert
     ├── Update
     ├── Delete
     ├── Join
     ├── Subquery / CTE
     ├── Set Operations
     ├── Aggregation
     ├── Window
     ├── Raw
     └── Extensions
             │
             ▼
       Immutable Query Model
             │
             ▼
            AST
             │
             ▼
       Semantic Query Engine
```

---

# 233. Regla consolidada del bloque

```text
Query Builder
=
Developer Intent
→
Structured Query Representation
```

Nunca:

```text
Query Builder
=
Developer Intent
→
SQL String
```

---

# 234. Siguiente bloque

El siguiente documento inicia:

```text
Block 5 — Optimizer and Planner
```

con:

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
```

---

# 235. Siguiente documento

```text
55_DATABASE_QUERY_OPTIMIZER_ARCHITECTURE.md
```

Este documento deberá formalizar:

```text
Query Optimizer
Optimization Pipeline
Optimization Phases
Optimization Context

Semantic Artifact Input
Optimized Query Artifact Output

Rewrite vs Optimization
Rule System
Rule Scheduling
Rule Dependencies
Rule Priorities

Fixpoint Optimization
Bounded Iteration
Convergence

Semantic Equivalence
Proof Requirements
Safety Barriers

Predicate Optimization
Join Optimization
Subquery Decorrelation
CTE Optimization
Set Operation Optimization
Aggregate Optimization
Window Optimization

Constant Folding
Predicate Simplification
Expression Simplification
Dead Projection Elimination
Redundant Predicate Removal
Join Elimination
Outer Join Strengthening
Predicate Pushdown
Projection Pushdown

Volatility
Determinism
Raw Barriers
Security Barriers

Optimization Budgets
Cost-Aware Optimization
Rule Telemetry
Explainability

Extension Optimization Rules
Persistent Runtime Safety
```

manteniendo la frontera:

```text
Semantic Analysis
=
What does this query mean?

Optimizer
=
Which equivalent query form
is better?

Planner
=
How should that optimized query
be executed?
```