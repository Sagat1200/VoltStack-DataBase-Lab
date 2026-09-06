# 53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md

# VoltStack Quantum Database
## Raw Expression and Escape Hatch System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 53 — Raw Expression and Escape Hatch System  
**Estado:** Architecture Specification  
**Nivel:** Query Engine / Query Builder / Security / Extensibility  
**Versión:** 1.0

---

# 1. Propósito

`Raw Expression and Escape Hatch System` define la arquitectura mediante la cual VoltStack permite introducir expresiones, predicates, projections, orderings, groupings y otros fragmentos no modelados por el Query Engine estructurado.

El objetivo no es hacer que Raw sea una ruta paralela al Query Engine.

El objetivo es ofrecer una salida explícita, controlada y auditable para casos donde:

```text
la capacidad requerida
no existe todavía como nodo semántico
```

o donde el desarrollador necesita:

```text
vendor-specific SQL
dialect-specific syntax
database-specific functions
experimental features
legacy SQL integration
special operator syntax
```

La regla central será:

```text
Raw SQL
=
Explicit Escape Hatch
```

y nunca:

```text
Raw SQL
=
Default Query Representation
```

---

# 2. Regla maestra

```text
Structured Query Representation
is the default

Raw Representation
is exceptional
```

Además:

```text
Raw
≠
Trusted

Raw
≠
Safe

Raw
≠
Portable

Raw
≠
Semantically Understood

Raw
≠
Optimizer-Friendly
```

---

# 3. Posición arquitectónica

```text
Developer
    │
    ▼
Query Builder
    │
    ├── Structured API
    │      │
    │      ▼
    │   Query Model / AST
    │
    └── Raw Escape Hatch
           │
           ▼
      Raw Query Nodes
           │
           ▼
   Validation / Security
           │
           ▼
    Semantic Analysis
           │
           ├── understood regions
           └── raw barriers
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

# 4. Objetivos

El sistema deberá proporcionar:

- raw expressions;
- raw predicates;
- raw projections;
- raw ordering;
- raw grouping;
- raw HAVING;
- raw join conditions;
- raw source fragments;
- raw DML expressions;
- raw CTE bodies cuando sean explícitamente permitidos;
- parameter binding estructurado;
- identifier safety;
- trust classification;
- provenance;
- portability classification;
- capability declarations;
- semantic barrier metadata;
- optimizer barrier metadata;
- lineage contracts opcionales;
- type contracts opcionales;
- output contracts opcionales;
- fingerprinting;
- diagnostics;
- telemetry;
- persistent-runtime safety.

---

# 5. No responsabilidades

El sistema Raw no deberá:

```text
convertir automáticamente strings en SQL
marcar raw como seguro por default
bypassear parameter binding
bypassear identifier rules
bypassear authorization
bypassear multitenancy
bypassear diagnostics
bypassear resource budgets
bypassear query fingerprints
```

---

# 6. Relación con documentos anteriores

Este sistema depende especialmente de:

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
39_DATABASE_QUERY_TYPE_INFERENCE_SYSTEM.md
41_DATABASE_QUERY_CONSTRAINT_ANALYSIS_SYSTEM.md
42_DATABASE_QUERY_SEMANTIC_GRAPH_SYSTEM.md
43_DATABASE_QUERY_BUILDER_ARCHITECTURE.md
48_DATABASE_JOIN_QUERY_BUILDER.md
49_DATABASE_SUBQUERY_AND_CTE_SYSTEM.md
50_DATABASE_UNION_AND_SET_OPERATION_SYSTEM.md
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
```

---

# 7. Taxonomía Raw

VoltStack deberá distinguir al menos:

```text
RawExpression
RawPredicate
RawProjection
RawOrdering
RawGrouping
RawHavingPredicate
RawJoinPredicate
RawRelationSource
RawSubquery
RawCteBody
RawAssignmentExpression
RawReturningExpression
RawExtensionNode
```

No deberá existir un único:

```text
RawSql(string)
```

utilizado indiscriminadamente para todo.

---

# 8. Context-sensitive Raw

Un fragmento utilizado como:

```text
predicate
```

no es lo mismo que un fragmento utilizado como:

```text
projection
```

aunque el texto sea parecido.

Por tanto:

```text
RawExpression
≠
RawPredicate
```

y:

```text
RawOrdering
≠
RawGrouping
```

---

# 9. RawFragmentId

Cada raw node podrá tener identidad explícita:

```text
RawFragmentId
```

Ejemplo:

```text
RAW1
RAW2
RAW3
```

---

# 10. RawFragment

Contrato conceptual:

```php
interface RawFragmentNode
{
    public function rawFragmentId(): RawFragmentId;

    public function rawContract(): RawFragmentContract;
}
```

---

# 11. RawFragmentContract

Modelo conceptual:

```php
final readonly class RawFragmentContract
{
    public function __construct(
        public RawTrustLevel $trust,
        public RawDialectScope $dialect,
        public RawSemanticBarrierSet $barriers,
        public CapabilityRequirementSet $capabilities,
        public QueryMetadata $metadata,
    ) {}
}
```

---

# 12. Trust classification

VoltStack deberá distinguir:

```php
enum RawTrustLevel
{
    case UNTRUSTED;
    case APPLICATION_TRUSTED;
    case FRAMEWORK_TRUSTED;
    case EXTENSION_TRUSTED;
}
```

---

# 13. Trust ≠ safety

Incluso:

```text
FRAMEWORK_TRUSTED
```

no significa:

```text
semantically understood
portable
optimizer-safe
```

Sólo describe el origen/confianza del fragmento.

---

# 14. Default trust

Toda entrada raw construida desde aplicación deberá tratarse como:

```text
UNTRUSTED
```

o:

```text
APPLICATION_TRUSTED
```

según la API utilizada.

Nunca como framework-trusted automáticamente.

---

# 15. Raw text

El raw text deberá mantenerse separado de:

```text
bindings
identifiers
metadata
contracts
```

---

# 16. RawExpression

Ejemplo público:

```php
$query->selectRaw(
    'COALESCE(score, 0)'
);
```

Internamente:

```text
RawProjection
└── RawExpression
    ├── fragment
    ├── parameter definitions
    ├── trust
    ├── dialect scope
    └── barriers
```

---

# 17. Raw values

Incorrecto:

```php
$query->whereRaw(
    "email = '$email'"
);
```

Correcto:

```php
$query->whereRaw(
    'email = ?',
    [$email]
);
```

Pero internamente VoltStack no deberá preservar `?` como physical placeholder final si puede evitarlo.

---

# 18. Structured Raw parameters

Preferiblemente:

```text
RawFragment
├── text template
├── RawParameterReference P1
└── BindingSet
    └── P1 → value
```

---

# 19. Physical placeholder independence

El Raw System deberá evitar que el desarrollador dependa de:

```text
?
$1
:foo
```

como identidad semántica del parámetro.

---

# 20. RawTemplate

Una estrategia posible:

```php
final readonly class RawTemplate
{
    public function __construct(
        public string $template,
        public RawTemplatePlaceholderSet $placeholders,
    ) {}
}
```

---

# 21. Placeholder model

Ejemplo conceptual:

```text
COALESCE(score, {{P1}})
```

internamente.

Compiler transforma posteriormente `P1` al placeholder físico correspondiente.

---

# 22. No string replacement naïve

Nunca:

```php
str_replace(
    '?',
    $value,
    $sql
);
```

---

# 23. Parameter safety

Los valores raw deberán pasar por el mismo:

```text
Parameter System
Binding System
Type System
Sensitive Redaction System
```

que el resto del Query Engine.

---

# 24. Raw identifiers

Los identifiers no deberán entrar como runtime bind values.

Ejemplo inseguro:

```php
$order = $_GET['column'];

$query->orderByRaw($order);
```

---

# 25. Identifier allowlist

La aplicación deberá utilizar:

```text
IdentifierFactory
+
allowlist
```

para identifiers dinámicos.

---

# 26. Parameterization does not protect identifiers

Se mantiene:

```text
Value Parameterization
≠
Identifier Safety
```

---

# 27. RawPredicate

Modelo conceptual:

```php
final readonly class RawPredicate
    implements PredicateNode, RawFragmentNode
{
    public function __construct(
        public RawFragmentId $id,
        public RawTemplate $template,
        public ParameterDefinitionSet $parameters,
        public RawFragmentContract $contract,
    ) {}
}
```

---

# 28. Raw predicate barrier

Por default un `RawPredicate` podrá bloquear:

```text
truth-domain understanding
constraint derivation
null-rejection analysis
predicate implication
predicate simplification
pushdown
join classification
```

---

# 29. RawExpression barrier

Por default podrá bloquear:

```text
type inference
lineage
constantness
volatility analysis
determinism analysis
function resolution
operator resolution
```

---

# 30. RawProjection

Un raw projection puede impedir conocer:

```text
output type
output nullability
lineage
dependency set
column identity semantics
```

salvo que se proporcione contrato explícito.

---

# 31. Raw output contract

La API avanzada podrá aceptar:

```php
$query->selectRaw(
    'vendor_score(data)',
    output: RawOutputContract::scalar(
        type: QueryType::decimal(),
        nullable: false,
        alias: 'score'
    )
);
```

---

# 32. Declared ≠ proven

Toda información suministrada manualmente deberá etiquetarse como:

```text
DECLARED
```

no:

```text
PROVEN
```

---

# 33. RawTypeContract

Conceptualmente:

```php
final readonly class RawTypeContract
{
    public function __construct(
        public QueryType $type,
        public Nullability $nullability,
        public SemanticCertainty $certainty,
    ) {}
}
```

---

# 34. RawLineageContract

Podrá declarar:

```php
final readonly class RawLineageContract
{
    public function __construct(
        public LineageSet $sources,
        public SemanticCertainty $certainty,
    ) {}
}
```

---

# 35. RawDependencyContract

Igualmente:

```text
RawDependencyContract
```

podrá enumerar:

```text
tables
columns
functions
types
extensions
capabilities
```

---

# 36. Explicit contract improves analysis

Un raw fragment:

```text
vendor_hash(users.email)
```

sin contrato puede ser opaque.

Con descriptor:

```text
type = BinaryHash
lineage = users.email
determinism = deterministic
nullability = nullable if input nullable
```

puede participar mucho mejor en Semantic Analysis.

---

# 37. Prefer extension over raw

Si una expresión raw se usa frecuentemente:

```text
vendor_geo_distance(...)
```

la recomendación será convertirla en:

```text
Semantic Query Extension
```

definida en el documento 54.

---

# 38. Raw is migration path

Raw debe verse como:

```text
escape hatch
+
prototype path
+
legacy integration path
```

no como destino arquitectónico final para features importantes.

---

# 39. RawOrdering

Ejemplo:

```php
$query->orderByRaw(
    'FIELD(status, ?, ?, ?)',
    ['critical', 'high', 'normal']
);
```

Internamente deberá existir un nodo específico de ordering raw.

---

# 40. Raw ordering barrier

Puede impedir:

```text
ordering equivalence
stable ordering analysis
cursor pagination analysis
sort reuse
window compatibility
```

---

# 41. RawGrouping

Ejemplo:

```php
$query->groupByRaw(
    'vendor_grouping_expression(category)'
);
```

Debe existir como:

```text
RawGroupingExpression
```

no como string almacenada en `groupBy[]`.

---

# 42. Raw HAVING

```php
$query->havingRaw(
    'SUM(total) > ?',
    [1000]
);
```

será un:

```text
RawHavingPredicate
```

con el mismo parameter system.

---

# 43. RawJoinPredicate

Ejemplo:

```php
$join->onRaw(
    'vendor_match(a.payload, b.payload)'
);
```

deberá marcarse como:

```text
join semantic barrier
```

cuando no exista descriptor.

---

# 44. Join classification impact

Con raw ON, Relation/Join Analysis puede no poder determinar:

```text
EQUI_JOIN
RANGE_JOIN
relationship evidence
join keys
null rejection
```

---

# 45. Raw relation source

El sistema podrá soportar un:

```text
RawRelationSource
```

sólo mediante API muy explícita.

---

# 46. Example

```php
$query->fromRaw(
    'vendor_table_function(?) AS t',
    [$value]
);
```

---

# 47. Raw relation risk

Esto afecta profundamente:

```text
relation identity
output columns
scope
types
lineage
dependencies
```

por lo que deberá requerir contratos adicionales.

---

# 48. RawRelationContract

Conceptualmente:

```php
final readonly class RawRelationContract
{
    public function __construct(
        public RelationAlias $alias,
        public RawRelationOutputContract $output,
        public RawDependencyContract $dependencies,
        public RawFragmentContract $fragment,
    ) {}
}
```

---

# 49. RawRelationOutputContract

Podrá declarar:

```text
column names
types
nullability
ordinal positions
lineage when known
```

---

# 50. Unknown output

Si el output no puede conocerse:

```text
Wildcard expansion
Set Operation compatibility
Hydration
Ordering alias resolution
```

podrán quedar bloqueados.

---

# 51. Raw subquery

Un `RawSubquery` deberá ser claramente distinguible de un query artifact normal.

---

# 52. Raw subquery contract

Deberá incluir al menos:

```text
output shape
dialect scope
parameters
trust
barriers
dependencies when known
```

---

# 53. Raw CTE

Un raw CTE body deberá ser una feature avanzada.

No:

```php
$query->with('x', 'SELECT ...');
```

por simple overload ambiguo.

Preferible:

```php
$query->withRaw(
    'x',
    RawQuery::from(...)
);
```

---

# 54. Explicit raw APIs

La API deberá favorecer nombres como:

```text
selectRaw()
whereRaw()
havingRaw()
orderByRaw()
groupByRaw()
onRaw()
fromRaw()
withRaw()
```

para hacer visible el escape hatch.

---

# 55. No implicit raw strings

Prohibido:

```php
$query->select('COUNT(*)');
```

interpretándolo mágicamente como SQL.

Deberá interpretarse como identifier/structured input o fallar.

---

# 56. Strings by default are data/identifiers

VoltStack deberá mantener:

```text
ordinary string input
≠
SQL fragment
```

---

# 57. RawDialectScope

Una raw feature deberá declarar su alcance.

```php
enum RawDialectScope
{
    case PORTABLE_SQL;
    case GENERIC_SQL;
    case MYSQL;
    case MARIADB;
    case POSTGRESQL;
    case SQLITE;
    case PLATFORM_SET;
    case UNKNOWN;
}
```

---

# 58. Better generalized model

En implementación real puede preferirse:

```text
DialectRequirementSet
```

en lugar de hardcodear vendors en el enum core.

Por ejemplo:

```text
DialectRequirementSet
├── sql-standard
├── postgresql-family
└── custom-extension:x
```

---

# 59. No vendor branching in core

Aunque Raw declare target dialect:

```text
Raw System
```

no deberá contener lógica tipo:

```php
if ($database === 'postgresql') ...
```

para semántica.

---

# 60. Capability declaration

Raw fragments podrán declarar:

```text
CapabilityRequirementSet
```

por ejemplo:

```text
FUNCTION.vendor_score
OPERATOR.json_contains
FEATURE.full_text_search
```

---

# 61. Raw portability

Un fragmento podrá clasificarse:

```text
PORTABLE
PORTABLE_WITH_REQUIREMENTS
PLATFORM_SPECIFIC
DIALECT_SPECIFIC
UNKNOWN
```

---

# 62. Unknown by default

Si VoltStack no conoce la semántica del raw fragment:

```text
Portability = UNKNOWN
```

deberá ser preferible a inventar portabilidad.

---

# 63. Semantic barriers

Se definirá un conjunto tipado.

```php
enum RawSemanticBarrier
{
    case TYPE_INFERENCE;
    case NULLABILITY_ANALYSIS;
    case CONSTRAINT_ANALYSIS;
    case LINEAGE_ANALYSIS;
    case DEPENDENCY_ANALYSIS;
    case VOLATILITY_ANALYSIS;
    case DETERMINISM_ANALYSIS;
    case SYMBOL_ANALYSIS;
    case RELATION_ANALYSIS;
    case OPTIMIZATION;
    case PORTABILITY_ANALYSIS;
}
```

---

# 64. Barrier set

```php
final readonly class RawSemanticBarrierSet
{
    /**
     * @param set<RawSemanticBarrier> $barriers
     */
    public function __construct(
        public array $barriers,
    ) {}
}
```

---

# 65. Barrier ≠ total blindness

Un raw node puede bloquear sólo determinadas fases.

Ejemplo:

```text
type known
lineage known
determinism known
but optimizer rewrite not allowed
```

---

# 66. Fine-grained contracts

VoltStack deberá permitir:

```text
partial semantic understanding
```

en lugar de:

```text
RAW = everything unknown
```

---

# 67. Raw semantic profile

Conceptualmente:

```text
RawSemanticProfile
├── Type: DECLARED
├── Nullability: DECLARED
├── Lineage: UNKNOWN
├── Determinism: DECLARED
├── Volatility: UNKNOWN
├── Constraints: BARRIER
└── Optimization: BARRIER
```

---

# 68. Optimizer barrier

Un raw expression deberá ser tratado de forma conservadora.

No se deberá:

```text
duplicate
eliminate
reorder
push
fold
```

si no se conoce suficiente información.

---

# 69. Volatility importance

Ejemplo:

```text
vendor_random()
```

si se duplica podría producir resultados distintos.

Por tanto:

```text
unknown volatility
→ conservative behavior
```

---

# 70. Constant folding

Raw nodes no deberán constant-folded salvo descriptor que lo permita explícitamente.

---

# 71. Predicate pushdown

Un `RawPredicate` no deberá moverse libremente a través de:

```text
LEFT JOIN
GROUP BY
window stage
CTE barrier
security barrier
```

sin proof suficiente.

---

# 72. Raw security policy

La plataforma deberá permitir una política global:

```php
enum RawQueryPolicy
{
    case ALLOW;
    case ALLOW_WITH_WARNING;
    case REQUIRE_TRUST_MARKER;
    case FORBID_APPLICATION_RAW;
    case FORBID_ALL_RAW;
}
```

---

# 73. Production policy

Una aplicación de alta seguridad podrá configurar:

```text
FORBID_APPLICATION_RAW
```

permitiendo únicamente fragments provenientes de:

```text
framework
verified extensions
```

---

# 74. Environment-specific policy

Desarrollo:

```text
ALLOW_WITH_WARNING
```

Producción:

```text
REQUIRE_TRUST_MARKER
```

es una posible configuración.

---

# 75. Trust marker

La API avanzada podría exigir:

```php
Raw::trusted(
    fragment: '...',
    justification: 'Uses internal PostgreSQL extension'
);
```

---

# 76. Justification metadata

Podrá registrarse:

```text
reason
source location
owner/package
ticket/reference
expected dialect
review status
```

---

# 77. Auditability

Raw fragments deberán ser visibles en:

```text
debug tools
telemetry
query explain
security audit
query inspection
```

---

# 78. Telemetry event

Podrá emitirse información como:

```text
database.raw_fragment.used
```

con:

```text
fragment kind
trust level
dialect scope
barriers
package/source
```

pero nunca valores sensibles sin redacción.

---

# 79. No raw text telemetry by default

El raw SQL completo no deberá enviarse automáticamente a telemetry externa.

---

# 80. Fingerprinting

Debe evitarse incluir:

```text
credentials
secrets
sensitive literals
```

en fingerprints/logs.

---

# 81. Raw structural fingerprint

Podrá calcularse sobre:

```text
normalized raw template
parameter positions
fragment kind
trust level
dialect requirements
semantic contracts
extension version
```

---

# 82. Runtime values excluded

Los binding values no deberán participar en el structural fingerprint ordinario.

---

# 83. Sensitive literal problem

Si el usuario incrusta literalmente:

```text
secret token
```

dentro de raw text, VoltStack no podrá distinguirlo siempre.

Por eso deberá desaconsejarse fuertemente.

---

# 84. Raw literals policy

Podrá existir un linter que detecte patrones sospechosos:

```text
quoted strings
credentials
long tokens
email-like literals
UUIDs
```

pero sólo como diagnóstico heurístico.

---

# 85. Heuristic ≠ security proof

Un linter raw:

```text
cannot guarantee
absence of secrets
```

---

# 86. Raw validation pipeline

```text
Raw Construction
      │
      ▼
Template Validation
      │
      ▼
Parameter Validation
      │
      ▼
Security Policy
      │
      ▼
Structural Query Validation
      │
      ▼
Semantic Contract Validation
      │
      ▼
Capability Validation
      │
      ▼
Compilation
```

---

# 87. Template validation

Deberá detectar al menos:

```text
placeholder count mismatch
unknown placeholder IDs
duplicate malformed placeholders
forbidden delimiter patterns
invalid empty fragment
```

---

# 88. Parameter mismatch

Ejemplo:

```text
fragment expects P1, P2
bindings provide P1 only
```

deberá fallar antes de Compiler.

---

# 89. Extra binding

Igualmente:

```text
binding P3
not referenced
```

podrá tratarse como error.

---

# 90. Raw SQL parsing

VoltStack V1 no deberá depender de implementar un parser SQL completo sólo para entender Raw.

---

# 91. Raw parsing policy

La arquitectura podrá soportar:

```text
optional parsers
optional static analysis
dialect analyzers
```

pero Raw seguirá siendo un escape hatch.

---

# 92. Partial parser

Un extension package podría analizar raw PostgreSQL expressions y generar mejores contracts.

---

# 93. Parser result ≠ original AST

Incluso si se parsea un fragmento raw, deberá distinguirse si fue:

```text
fully lifted into semantic AST
```

o sólo:

```text
partially analyzed raw
```

---

# 94. Raw lifting

Una futura feature podrá intentar:

```text
Raw Fragment
    │
    ▼
Dialect Parser
    │
    ▼
Structured Query Nodes
```

---

# 95. Successful lifting

Si el lifting es completo y confiable, el resultado podrá dejar de ser Raw.

---

# 96. Partial lifting

Si es incompleto:

```text
Raw node
+
partial semantic contracts
```

deberá mantenerse.

---

# 97. Query type inference

Si no hay TypeContract:

```text
RawExpression
→ QueryType::Unknown
```

---

# 98. Unknown type propagation

Unknown no deberá convertirse automáticamente en:

```text
Any
String
Mixed
```

---

# 99. Explicit type declaration

La API podrá permitir:

```php
DB::rawExpression(
    'vendor_uuid()'
)->returns(
    QueryType::uuid(),
    nullable: false
);
```

---

# 100. Type declarations and safety

Declarar tipo incorrectamente puede causar:

```text
invalid optimizer assumptions
hydration errors
compiler coercion errors
```

por lo que deberá llevar provenance `DECLARED`.

---

# 101. Nullability

Sin contrato:

```text
Nullability = UNKNOWN
```

---

# 102. Determinism

Sin descriptor:

```text
Determinism = UNKNOWN
```

---

# 103. Volatility

Sin descriptor:

```text
Volatility = UNKNOWN
```

y el Optimizer actuará conservadoramente.

---

# 104. Constraint Analysis

Constraint Analysis no deberá derivar facts internos desde raw text sin analyzer explícito.

---

# 105. Example

```text
RawPredicate("age > 18")
```

no deberá generar automáticamente:

```text
age >= 19
age NOT NULL
```

sólo por parsing superficial.

---

# 106. Declared constraint contract

Una API avanzada podría adjuntar:

```text
ConstraintDeclarationSet
```

pero éstos seguirán siendo:

```text
DECLARED
```

---

# 107. Security-generated raw

Idealmente las políticas de seguridad nunca deberán usar Raw.

---

# 108. Security policies prefer structured AST

Para:

```text
tenant_id = P1
```

deberá utilizarse:

```text
ComparisonPredicate
```

y no raw.

---

# 109. Mandatory policy barriers

Si por excepción una security integration utiliza raw, deberá marcar:

```text
MANDATORY
SECURITY_SENSITIVE
NON_REMOVABLE
```

---

# 110. Optimizer respect

Optimizer nunca deberá eliminar ese node basándose sólo en opacidad.

---

# 111. Multitenancy

Multitenancy deberá preferir:

```text
structured predicates
structured relation sources
```

y no raw SQL injection.

---

# 112. Raw and ORM

ORM puede exponer:

```text
whereRaw()
selectRaw()
```

como DX.

Pero deberá reutilizar el mismo Raw System.

---

# 113. No ORM-specific raw engine

Prohibido tener:

```text
ORMRawExpression
```

con semántica independiente del Query Engine.

---

# 114. Active Record integration

Active Record simplemente delegará:

```text
Model Query
→ Query Builder
→ Raw System
```

---

# 115. Raw in INSERT

Ejemplo:

```php
$query->valueExpression(
    'created_at',
    DB::rawExpression('CURRENT_TIMESTAMP')
);
```

---

# 116. Better semantic alternative

Si `CURRENT_TIMESTAMP` se usa frecuentemente, deberá existir preferentemente:

```text
CurrentTimestampExpression
```

en Expression System.

---

# 117. Raw in UPDATE

Ejemplo:

```php
$query->set(
    'counter',
    DB::rawExpression('counter + 1')
);
```

es posible.

Pero la forma estructurada preferida será:

```text
BinaryExpression(
    counter,
    ADD,
    Literal/Parameter(1)
)
```

---

# 118. Raw in DELETE

Generalmente Raw aparecerá en:

```text
predicate
CTE
source
```

pero no cambia la semántica fundamental de DELETE.

---

# 119. Raw RETURNING

Un returning raw deberá exigir output contract cuando sea necesario para result mapping.

---

# 120. Raw in JOIN

Debe conservar:

```text
JoinPredicate provenance
```

y sus barriers.

---

# 121. Raw in GROUP BY

Debe afectar:

```text
grouping legality analysis
functional dependency analysis
portability
```

---

# 122. Raw in HAVING

Constraint Analysis deberá ser conservador.

---

# 123. Raw in Window

Puede afectar:

```text
partition equivalence
ordering equivalence
frame analysis
sort reuse
```

---

# 124. Raw in Set Operations

Si un raw operand tiene output desconocido:

```text
Set Operation compatibility
```

puede quedar imposibilitada.

---

# 125. Raw output contracts required for set operands

Para participar confiablemente en:

```text
UNION
INTERSECT
EXCEPT
```

un raw operand deberá proveer:

```text
arity
types
nullability
output names
```

o resultar no compilable semánticamente.

---

# 126. Raw in CTE

Igualmente el CTE output deberá conocerse si el parent query lo consume estructuradamente.

---

# 127. Raw serialization

Los raw artifacts serializables sólo contendrán datos.

Nunca:

```text
Closure
Connection
PDO
Transaction
Request
User
Tenant
Container
Telemetry Span
```

---

# 128. Raw callbacks

Si la API utiliza closures para construir contracts, dichas closures deberán desaparecer después de finalization.

---

# 129. Persistent runtime safety

El sistema deberá ser seguro bajo:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 130. Shared immutable state

Podrá compartirse:

```text
Raw policy
frozen raw analyzer registry
immutable contracts
stateless validators
```

---

# 131. Operation-local state

Deberá ser local:

```text
RawFragmentBuilderState
parameter allocation
diagnostics
contract builder
security decisions
temporary analyzer state
```

---

# 132. No global raw trust flag

Prohibido:

```php
Raw::$trusted = true;
```

---

# 133. No global current dialect override

Prohibido:

```php
Raw::$dialect = 'postgresql';
```

por request.

---

# 134. Explicit context

Dialect/capability context deberá fluir por:

```text
QueryContext
SemanticContext
CompilationContext
```

---

# 135. Raw analyzer extensions

Podrá existir:

```text
RawFragmentAnalyzerRegistry
```

---

# 136. Raw analyzer responsibilities

Un analyzer puede aportar:

```text
type
nullability
dependencies
lineage
volatility
determinism
capabilities
portability
```

---

# 137. Analyzer identity

Cada analyzer deberá tener:

```text
id
version
supported fragment kind
supported dialect family
```

---

# 138. Analyzer version and fingerprints

Si un analyzer cambia la semántica derivada:

```text
analyzer version
```

deberá contribuir al semantic fingerprint.

---

# 139. Analyzer failure

Si no puede analizar con confianza:

```text
return UNKNOWN
```

no inventar facts.

---

# 140. RawFragmentAnalysisResult

Conceptualmente:

```php
final readonly class RawFragmentAnalysisResult
{
    public function __construct(
        public QueryType $type,
        public Nullability $nullability,
        public Determinism $determinism,
        public Volatility $volatility,
        public DependencySet $dependencies,
        public LineageSet $lineage,
        public CapabilityRequirementSet $capabilities,
        public RawSemanticBarrierSet $remainingBarriers,
        public DiagnosticCollection $diagnostics,
    ) {}
}
```

---

# 141. User contract vs analyzer result

Si ambos existen y chocan:

```text
Declared Contract
vs
Analyzer Result
```

VoltStack deberá diagnosticar la discrepancia.

---

# 142. Conflict policy

La política recomendada:

```text
PROVEN/ANALYZED
overrides
DECLARED
```

para ejecución, salvo error explícito si contradicción afecta seguridad.

---

# 143. No silent conflict

Ejemplo:

```text
Declared: NOT NULL
Analyzed: Nullable
```

deberá fallar o producir diagnóstico severo.

---

# 144. Raw compiler handling

Compiler recibirá:

```text
validated raw template
+
physical parameter mapping
+
dialect requirements
```

---

# 145. Compiler must not interpolate values

Compiler generará placeholders.

Driver posteriormente realiza binding.

---

# 146. Raw template compile

Conceptualmente:

```text
RawTemplate
    │
    ├── literal segment
    ├── ParameterRef P1
    ├── literal segment
    └── ParameterRef P2
```

Compiler produce:

```text
vendor_fn(?, ?)
```

o:

```text
vendor_fn($1, $2)
```

según el target.

---

# 147. Raw identifier placeholders

Si se soportan identifiers dinámicos, deberán utilizar un token distinto:

```text
RawIdentifierRef
```

no `ParameterRef`.

---

# 148. RawIdentifierRef

Compiler podrá quote/compile un identifier estructurado.

Así:

```text
value placeholder
≠
identifier placeholder
```

---

# 149. Example template

Conceptualmente:

```text
{{IDENTIFIER:I1}} = {{PARAM:P1}}
```

Compiler:

```text
"status" = $1
```

---

# 150. This improves safety

Permite evitar:

```text
manual identifier concatenation
```

aunque el fragmento siga siendo raw.

---

# 151. Raw macro API

Podrá existir una API segura:

```php
Raw::template(
    '{{id:column}} = {{param:value}}',
    identifiers: [
        'column' => Identifier::column('status'),
    ],
    values: [
        'value' => 'active',
    ],
);
```

---

# 152. Raw template parser

Este mini template parser deberá comprender sólo tokens VoltStack.

No intentar parsear SQL completo.

---

# 153. Template token escaping

Deberá existir una política clara para escapes y caracteres literales.

---

# 154. No ambiguous placeholder syntax

La sintaxis interna deberá ser no ambigua y completamente validable.

---

# 155. Raw fingerprint normalization

Dos templates equivalentes con distintos runtime values deberán compartir query shape.

---

# 156. Example

```text
score > P1
```

con:

```text
P1 = 10
```

y:

```text
P1 = 20
```

deberán compartir structural fingerprint.

---

# 157. Raw text whitespace

La arquitectura deberá decidir si:

```text
whitespace-only changes
```

afectan fingerprints.

---

# 158. Recommended V1

No intentar canonicalización SQL profunda.

Puede utilizarse:

```text
exact normalized template representation
```

para evitar equivalencias falsas.

---

# 159. Comments

Raw comments pueden contener información sensible.

Deberán ser tratados cuidadosamente en telemetry/fingerprints.

---

# 160. Fingerprint source

El fingerprint podrá utilizar:

```text
cryptographic hash of normalized raw template
```

en vez de exponer el texto.

---

# 161. Diagnostics without raw leakage

Los diagnostics podrán mostrar:

```text
RawFragmentId
kind
source location
template hash
```

y opcionalmente el fragmento si la política local lo permite.

---

# 162. Sensitive redaction

Si el template usa correctamente ParameterRefs, los valores podrán redacted mediante el sistema normal.

---

# 163. Raw literal cannot always be redacted safely

Esto refuerza la regla:

```text
never embed runtime values directly in raw text
```

---

# 164. Query inspector

Podrá mostrar:

```text
RAW3
Kind: Predicate
Trust: ApplicationTrusted
Dialect: PostgreSQL-like
Parameters: 2
Barriers:
  - Constraint Analysis
  - Optimization
```

---

# 165. Developer tooling

Debug Toolbar podrá marcar Raw visualmente como:

```text
RAW / NON-PORTABLE / OPTIMIZER BARRIER
```

---

# 166. Production warnings

Una policy podrá emitir warning si:

```text
raw count > threshold
```

por query.

---

# 167. Raw budget

Deberán existir límites para:

```text
raw fragment count
raw template length
raw parameters per fragment
raw identifiers per fragment
raw relation output width
raw contract complexity
raw analyzer work
```

---

# 168. RawQueryBudget

Conceptualmente:

```php
final readonly class RawQueryBudget
{
    public function __construct(
        public int $maxFragments,
        public int $maxTemplateLength,
        public int $maxParametersPerFragment,
        public int $maxIdentifiersPerFragment,
        public int $maxDeclaredOutputColumns,
    ) {}
}
```

---

# 169. Budget exhaustion

Deberá generar:

```text
RawQueryBudgetExceededException
```

---

# 170. No silent truncation

Nunca:

```text
drop fragment tail
drop parameters
ignore output columns
```

---

# 171. Diagnostics

Errores típicos:

```text
RawTemplateException
RawParameterMismatchException
RawIdentifierMismatchException
RawPolicyViolationException
RawDialectMismatchException
RawContractConflictException
RawBudgetExceededException
```

---

# 172. Dialect mismatch diagnostic

Ejemplo:

```text
Raw fragment RAW5 requires dialect capability:

    vendor:postgresql

Selected target:

    sqlite

The fragment cannot be compiled for
the selected platform.
```

---

# 173. Trust diagnostic

```text
Application raw SQL is disabled by
the active RawQueryPolicy.

Fragment:
    RAW8

Source:
    ReportQuery.php:42
```

---

# 174. Parameter diagnostic

```text
Raw fragment RAW11 references parameter P3,
but no matching binding is present.
```

---

# 175. Contract diagnostic

```text
Raw projection RAW17 declares:

    type: Integer
    nullable: false

Analyzer reports:

    type: String
    nullable: unknown

The declared contract conflicts with
available semantic evidence.
```

---

# 176. Security architecture

La seguridad deberá estructurarse como:

```text
Raw Fragment
     │
     ▼
Raw Policy
     │
     ▼
Template Validation
     │
     ▼
Identifier Validation
     │
     ▼
Parameter Validation
     │
     ▼
Trust Validation
     │
     ▼
Capability Validation
     │
     ▼
Compiler
```

---

# 177. SQL injection prevention

Las garantías principales serán:

```text
runtime values
→ parameters

identifiers
→ Identifier objects / allowlists

raw syntax
→ explicit API

trust
→ policy

compiler
→ no value interpolation
```

---

# 178. Impossible guarantee

VoltStack no puede garantizar que un raw SQL arbitrary string escrito por el desarrollador sea seguro si contiene concatenación manual.

---

# 179. Therefore

Raw deberá considerarse:

```text
explicit developer responsibility
+
framework-assisted containment
```

---

# 180. Logging

Raw fragments deberán poder loggearse en forma segura:

```text
hash
kind
parameter count
dialect
barriers
trust
```

---

# 181. Query telemetry

La telemetría podrá incluir:

```text
raw_fragment_count
raw_barrier_count
raw_portability_state
raw_trust_distribution
```

---

# 182. Performance impact

Raw puede impedir optimizaciones.

Por ejemplo:

```text
Predicate Pushdown: blocked
Join Elimination: blocked
Type Narrowing: blocked
```

---

# 183. Explain output

Query Explain debería indicar:

```text
Optimization Barrier RAW4
Reason:
    Unknown volatility
    Unknown predicate semantics
```

---

# 184. Migration guidance

Developer tooling podrá sugerir:

```text
Raw expression appears frequently.
Consider creating a Query Extension.
```

---

# 185. Raw usage metrics

Telemetry local podrá identificar:

```text
most-used raw templates
most common barriers
platform-specific raw dependencies
```

para orientar futuras features del framework.

---

# 186. Query extension promotion

Flujo recomendado:

```text
Raw Feature
    │
    ▼
Repeated Usage
    │
    ▼
Semantic Contract
    │
    ▼
Query Extension
    │
    ▼
First-Class Query Node
```

---

# 187. Raw and capability discovery

Un raw node puede declarar capabilities explícitas.

No deberá intentar descubrirlas desde strings mediante heurísticas obligatorias.

---

# 188. Raw and query portability tools

Un analyzer podrá reportar:

```text
Query portability degraded because
3 raw fragments are dialect-specific.
```

---

# 189. Serialization

Raw templates y contracts deberán ser versionables.

---

# 190. Raw artifact version

Podrá existir:

```text
RawArtifactFormatVersion
```

para cache/serialization.

---

# 191. Backward compatibility

Cambios en:

```text
template token syntax
contract model
barrier semantics
```

deberán formar parte de versioning policy.

---

# 192. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Raw
```

---

# 193. Directory structure propuesta

```text
Query/
└── Raw/
    ├── Contract/
    │   ├── RawFragmentNode.php
    │   ├── RawAnalyzerInterface.php
    │   └── RawPolicyInterface.php
    │
    ├── Core/
    │   ├── RawFragmentId.php
    │   ├── RawTemplate.php
    │   ├── RawFragmentContract.php
    │   ├── RawTrustLevel.php
    │   ├── RawDialectScope.php
    │   ├── RawSemanticBarrier.php
    │   └── RawSemanticBarrierSet.php
    │
    ├── Expression/
    │   ├── RawExpression.php
    │   └── RawProjection.php
    │
    ├── Predicate/
    │   ├── RawPredicate.php
    │   ├── RawHavingPredicate.php
    │   └── RawJoinPredicate.php
    │
    ├── Ordering/
    │   └── RawOrdering.php
    │
    ├── Grouping/
    │   └── RawGroupingExpression.php
    │
    ├── Relation/
    │   ├── RawRelationSource.php
    │   ├── RawRelationContract.php
    │   └── RawRelationOutputContract.php
    │
    ├── Subquery/
    │   └── RawSubquery.php
    │
    ├── Cte/
    │   └── RawCteBody.php
    │
    ├── Dml/
    │   ├── RawAssignmentExpression.php
    │   └── RawReturningExpression.php
    │
    ├── Parameter/
    │   ├── RawParameterReference.php
    │   ├── RawIdentifierReference.php
    │   └── RawTemplatePlaceholderSet.php
    │
    ├── Semantic/
    │   ├── RawTypeContract.php
    │   ├── RawLineageContract.php
    │   ├── RawDependencyContract.php
    │   ├── RawOutputContract.php
    │   ├── RawSemanticProfile.php
    │   └── RawFragmentAnalysisResult.php
    │
    ├── Analysis/
    │   ├── RawFragmentAnalyzerRegistry.php
    │   └── RawContractValidator.php
    │
    ├── Security/
    │   ├── RawQueryPolicy.php
    │   ├── RawPolicyEvaluator.php
    │   ├── RawTrustValidator.php
    │   └── RawIdentifierValidator.php
    │
    ├── Fingerprint/
    │   └── RawFingerprintBuilder.php
    │
    ├── Budget/
    │   └── RawQueryBudget.php
    │
    ├── Diagnostic/
    │   └── RawDiagnostic.php
    │
    └── Exception/
        ├── RawQueryException.php
        ├── RawTemplateException.php
        ├── RawParameterMismatchException.php
        ├── RawIdentifierMismatchException.php
        ├── RawPolicyViolationException.php
        ├── RawDialectMismatchException.php
        ├── RawContractConflictException.php
        └── RawQueryBudgetExceededException.php
```

---

# 194. Ownership matrix

| Concern | Owner |
|---|---|
| Explicit raw construction | Raw System |
| Runtime values | Parameter/Binding System |
| Dynamic identifiers | Identifier System |
| Raw trust | Raw Security System |
| Raw policy | Raw Security System |
| Declared type contracts | Raw Semantic Contract |
| Proven type info | Semantic Analyzer |
| Raw lineage contract | Raw Semantic Contract |
| Raw barriers | Raw System |
| Optimizer restrictions | Optimizer consumes barriers |
| Capability requirements | Raw Contract / Capability System |
| Portability | Semantic/Capability Analysis |
| Physical placeholders | Compiler |
| Native binding | Driver |
| SQL execution | Executor |
| Audit/telemetry | Telemetry integration |
| Persistent state reset | Runtime lifecycle |

---

# 195. Invariantes arquitectónicos

## DB-RAW-001
Raw será un escape hatch explícito.

## DB-RAW-002
Raw no será la representación por default.

## DB-RAW-003
Strings ordinarios no se interpretarán como SQL.

## DB-RAW-004
Raw no implicará trusted.

## DB-RAW-005
Raw no implicará safe.

## DB-RAW-006
Raw no implicará portable.

## DB-RAW-007
Raw no implicará semantic understanding.

## DB-RAW-008
RawExpression será distinta de RawPredicate.

## DB-RAW-009
RawOrdering será distinta de RawGrouping.

## DB-RAW-010
Raw fragments tendrán identidad explícita.

## DB-RAW-011
Raw values usarán Parameter System.

## DB-RAW-012
Raw bindings usarán BindingSet.

## DB-RAW-013
Compiler nunca interpolará runtime values.

## DB-RAW-014
Identifiers no serán tratados como value parameters.

## DB-RAW-015
Dynamic identifiers requerirán Identifier System.

## DB-RAW-016
Parameterization no protegerá identifiers.

## DB-RAW-017
Raw predicates podrán actuar como constraint barriers.

## DB-RAW-018
Raw expressions podrán actuar como type barriers.

## DB-RAW-019
Raw expressions podrán actuar como lineage barriers.

## DB-RAW-020
Raw ordering podrá actuar como ordering-analysis barrier.

## DB-RAW-021
Raw grouping podrá actuar como grouping-analysis barrier.

## DB-RAW-022
Raw JOIN predicates podrán bloquear join classification.

## DB-RAW-023
Raw relation sources requerirán output contract cuando sea necesario.

## DB-RAW-024
Raw subqueries serán explícitas.

## DB-RAW-025
Raw CTE bodies serán explícitos.

## DB-RAW-026
No habrá overload implícito string→RawCTE.

## DB-RAW-027
Declared contracts serán distintos de proven facts.

## DB-RAW-028
Declared type será etiquetado DECLARED.

## DB-RAW-029
Declared lineage será etiquetado DECLARED.

## DB-RAW-030
Declared dependencies serán etiquetadas DECLARED.

## DB-RAW-031
Unknown semantic facts permanecerán unknown.

## DB-RAW-032
Unknown type no se convertirá automáticamente en Any.

## DB-RAW-033
Unknown nullability permanecerá unknown.

## DB-RAW-034
Unknown volatility provocará comportamiento conservador.

## DB-RAW-035
Unknown determinism provocará comportamiento conservador.

## DB-RAW-036
Raw constant folding estará deshabilitado sin evidence.

## DB-RAW-037
Raw predicate pushdown requerirá evidence.

## DB-RAW-038
Raw duplication requerirá volatility knowledge.

## DB-RAW-039
Raw elimination requerirá semantic proof.

## DB-RAW-040
Raw barriers serán fine-grained.

## DB-RAW-041
Raw no significará blindness total obligatoriamente.

## DB-RAW-042
Partial semantic contracts estarán permitidos.

## DB-RAW-043
Raw dialect scope será explícito.

## DB-RAW-044
Raw capabilities serán explícitas cuando se conozcan.

## DB-RAW-045
Portability unknown será preferible a una inferencia falsa.

## DB-RAW-046
Core Raw System no dependerá de vendor branching.

## DB-RAW-047
RawQueryPolicy será configurable.

## DB-RAW-048
Application raw podrá ser prohibido.

## DB-RAW-049
Framework raw tendrá provenance explícita.

## DB-RAW-050
Extension raw tendrá provenance explícita.

## DB-RAW-051
Trust markers no sustituirán parameterization.

## DB-RAW-052
Raw fragments serán auditables.

## DB-RAW-053
Telemetry no expondrá runtime sensitive values.

## DB-RAW-054
Raw SQL completo no deberá enviarse a telemetry por default.

## DB-RAW-055
Fingerprints excluirán binding values.

## DB-RAW-056
Sensitive literals embebidos en raw son responsabilidad explícita del developer.

## DB-RAW-057
Raw linting será heurístico.

## DB-RAW-058
Heuristics no se tratarán como security proof.

## DB-RAW-059
Raw template validation ocurrirá antes de Compiler.

## DB-RAW-060
Parameter mismatch fallará antes de execution.

## DB-RAW-061
Identifier mismatch fallará antes de execution.

## DB-RAW-062
V1 no requerirá parser SQL completo para Raw.

## DB-RAW-063
Optional raw analyzers podrán existir.

## DB-RAW-064
Raw lifting podrá existir en el futuro.

## DB-RAW-065
Partial lifting mantendrá raw identity/barriers.

## DB-RAW-066
Fully lifted semantics podrán convertirse a structured AST.

## DB-RAW-067
Constraint Analysis no parseará raw text superficialmente.

## DB-RAW-068
Security policies preferirán structured AST.

## DB-RAW-069
Multitenancy preferirá structured AST.

## DB-RAW-070
Mandatory raw security nodes serán non-removable.

## DB-RAW-071
ORM reutilizará el mismo Raw System.

## DB-RAW-072
Active Record no tendrá raw engine independiente.

## DB-RAW-073
Repository API no tendrá raw engine independiente.

## DB-RAW-074
Raw DML expressions usarán el mismo core.

## DB-RAW-075
Structured expressions serán preferidas sobre raw DML.

## DB-RAW-076
Raw RETURNING deberá declarar output cuando sea necesario.

## DB-RAW-077
Raw grouping afectará grouping legality conservadoramente.

## DB-RAW-078
Raw windows afectarán window analysis conservadoramente.

## DB-RAW-079
Raw set operands requerirán output contract para compatibility analysis.

## DB-RAW-080
Raw CTE consumers requerirán output shape conocido.

## DB-RAW-081
Raw artifacts no almacenarán runtime resources.

## DB-RAW-082
Raw builder closures serán construction-only.

## DB-RAW-083
Raw System será persistent-runtime safe.

## DB-RAW-084
Raw trust no será global mutable state.

## DB-RAW-085
Raw dialect override no será global mutable state.

## DB-RAW-086
Raw analyzers serán versionados.

## DB-RAW-087
Analyzer versions contribuirán al fingerprint cuando afecten semántica.

## DB-RAW-088
Analyzer failure devolverá unknown.

## DB-RAW-089
Analyzer no inventará facts.

## DB-RAW-090
Declared/analyzed contract conflicts serán diagnosticados.

## DB-RAW-091
Contradicciones de seguridad no serán silenciosas.

## DB-RAW-092
Compiler recibirá raw ya validado.

## DB-RAW-093
Compiler asignará physical placeholders.

## DB-RAW-094
Identifier references serán distintas de parameter references.

## DB-RAW-095
Raw template token syntax será validada.

## DB-RAW-096
Raw template syntax será no ambigua.

## DB-RAW-097
Raw fingerprints no intentarán semantic SQL equivalence profunda en V1.

## DB-RAW-098
Template hash podrá usarse en logs.

## DB-RAW-099
Query Inspector mostrará raw barriers.

## DB-RAW-100
Developer tooling podrá sugerir promotion to extension.

## DB-RAW-101
Raw usage podrá medirse mediante telemetry segura.

## DB-RAW-102
Repeated raw features deberán poder migrarse a Query Extensions.

## DB-RAW-103
Raw serialization tendrá versioning.

## DB-RAW-104
Raw budget será explícito.

## DB-RAW-105
Budget exhaustion será error.

## DB-RAW-106
No habrá silent raw truncation.

## DB-RAW-107
Raw construction será determinista.

## DB-RAW-108
Parameter mapping será determinista.

## DB-RAW-109
Raw fingerprints serán deterministas.

## DB-RAW-110
Persistent workers no compartirán mutable raw state.

## DB-RAW-111
Concurrent raw queries permanecerán aisladas.

## DB-RAW-112
Raw System permanecerá separado de Connection.

## DB-RAW-113
Raw System permanecerá separado de Executor.

## DB-RAW-114
Raw System permanecerá separado de ORM persistence.

## DB-RAW-115
Raw System no eliminará semantic phase boundaries.

## DB-RAW-116
Raw barriers serán respetados por Optimizer.

## DB-RAW-117
Planner no ignorará raw capability requirements.

## DB-RAW-118
Compiler no reinterpretará raw trust.

## DB-RAW-119
Driver sólo recibirá final bindings.

## DB-RAW-120
Raw seguirá siendo una excepción explícita dentro de VoltStack.

---

# 196. Anti-patterns

## 196.1 Concatenación de runtime value

```php
$query->whereRaw(
    "status = '$status'"
);
```

**Rechazado.**

---

## 196.2 Dynamic identifier concatenation

```php
$query->orderByRaw(
    $_GET['column']
);
```

**Rechazado.**

---

## 196.3 Raw as default implementation

```php
public function where(...)
{
    return $this->whereRaw(...);
}
```

**Rechazado.**

---

## 196.4 Treat Raw as trusted

```text
raw
→ safe
```

**Rechazado.**

---

## 196.5 Parse strings heuristically

```text
if string contains "("
    assume SQL expression
```

**Rechazado.**

---

## 196.6 Infer semantics from raw text casually

```text
"age > 18"
→ infer all constraints automatically
```

**Rechazado.**

---

## 196.7 Ignore Raw in fingerprints

**Rechazado.**

La estructura raw sí afecta query identity.

---

## 196.8 Ignore Raw in portability

**Rechazado.**

---

## 196.9 Ignore Raw in telemetry/explain

**Rechazado.**

---

## 196.10 Global raw trust switch

```php
Raw::trustAll();
```

como estado global mutable de request.

**Rechazado.**

---

# 197. Ejemplo integral

Supongamos que una aplicación utiliza una función PostgreSQL específica aún no modelada semánticamente:

```php
$query = DB::table('documents as d')
    ->select('d.id')
    ->select('d.title')
    ->selectRaw(
        'ts_rank(d.search_vector, plainto_tsquery({{param:q}}))',
        parameters: [
            'q' => $search,
        ],
        output: RawOutputContract::scalar(
            type: QueryType::decimal(),
            nullable: false,
            alias: 'rank',
        ),
        dialect: DialectRequirement::postgresql(),
    )
    ->whereRaw(
        'd.search_vector @@ plainto_tsquery({{param:q}})',
        parameters: [
            'q' => $search,
        ],
        dialect: DialectRequirement::postgresql(),
    )
    ->orderBy('rank', 'desc');
```

---

# 198. Builder representation

Conceptualmente:

```text
SelectQuery
│
├── FROM
│   └── documents AS d
│
├── PROJECTION
│   ├── d.id
│   ├── d.title
│   └── RAW1
│       ├── kind = RawProjection
│       ├── template
│       ├── P1 = search
│       ├── output type = Decimal [DECLARED]
│       ├── nullability = NOT NULL [DECLARED]
│       └── dialect = PostgreSQL
│
├── WHERE
│   └── RAW2
│       ├── kind = RawPredicate
│       ├── template
│       ├── P2 = search
│       ├── constraint barrier
│       └── dialect = PostgreSQL
│
└── ORDER BY
    └── rank DESC
```

---

# 199. Semantic analysis

VoltStack podrá conocer:

```text
RAW1
├── Output type: Decimal [Declared]
├── Nullability: NotNull [Declared]
├── Dialect: PostgreSQL
├── Lineage: Unknown
├── Volatility: Unknown
└── Optimization: Restricted
```

Mientras:

```text
RAW2
├── Predicate type: Boolean-like
├── Constraint semantics: Unknown
├── Null rejection: Unknown
├── Dialect: PostgreSQL
└── Optimization barrier: yes
```

---

# 200. Query portability result

```text
Query Portability:
    PLATFORM_SPECIFIC

Required:
    PostgreSQL-compatible full-text capabilities

Raw fragments:
    RAW1
    RAW2
```

---

# 201. Optimizer behavior

Optimizer podrá optimizar las regiones estructuradas de la query, pero deberá respetar:

```text
RAW2
→ predicate movement barrier
```

si no existe evidence suficiente.

---

# 202. Future extension promotion

Posteriormente se podría crear:

```text
PostgreSqlFullTextRankExpression
PostgreSqlTextSearchPredicate
```

como semantic extension.

Entonces:

```text
RAW1 / RAW2
```

dejarían de ser necesarios.

---

# 203. Filosofía de evolución

```text
Raw First
    │
    ▼
Usage Proven
    │
    ▼
Semantic Contract Stabilized
    │
    ▼
Query Extension
    │
    ▼
First-Class Feature
```

---

# 204. Fórmula principal

```text
Raw Fragment
=
Explicit Template
+
Structured Parameter References
+
Structured Identifier References
+
Trust Metadata
+
Dialect/Capability Requirements
+
Semantic Contracts
+
Barrier Set
+
Provenance
```

---

# 205. Fórmula de seguridad

```text
Safe Raw Usage
=
Explicit Raw API
+
No Runtime Value Interpolation
+
Parameter Binding
+
Identifier Allowlisting
+
Trust Policy
+
Dialect Validation
+
Auditability
+
Resource Budgets
```

---

# 206. Fórmula semántica

```text
Raw Semantic Knowledge
=
Declared Contracts
+
Analyzer Results
+
Known Dependencies
+
Known Capabilities
-
Remaining Semantic Barriers
```

---

# 207. Fórmula de optimización

```text
Raw Optimization Freedom
=
Known Determinism
+
Known Volatility
+
Known Types
+
Known Lineage
+
Known Constraints
-
Semantic Barriers
```

---

# 208. Fórmula de portabilidad

```text
Raw Portability
=
Declared Dialect Scope
+
Capability Requirements
+
Analyzer Evidence
-
Unknown Vendor Semantics
```

---

# 209. Fórmula de persistent-runtime safety

```text
Persistent-Safe Raw System
=
Immutable Raw Artifacts
+
Operation-Scoped Raw State
+
Frozen Analyzer Registry
+
No Global Trust Switch
+
No Global Dialect State
+
No Runtime Resource Retention
```

---

# 210. Boundary final

```text
Structured Query Engine
        │
        ▼
"Can VoltStack represent
this feature semantically?"
        │
        ├── YES
        │     │
        │     ▼
        │ Structured AST
        │
        └── NO
              │
              ▼
          Explicit Raw
              │
              ▼
        Security Policy
              │
              ▼
        Semantic Contracts
              │
              ▼
          Raw Barriers
              │
              ▼
          Optimizer
              │
              ▼
           Compiler
```

---

# 211. Decisión arquitectónica final

VoltStack adopta Raw SQL únicamente como:

```text
Explicit Escape Hatch
```

protegido por:

```text
Typed Fragment Kinds
+
Structured Parameters
+
Structured Identifiers
+
Trust Classification
+
Security Policy
+
Semantic Contracts
+
Fine-Grained Barriers
+
Capability Requirements
+
Portability Classification
+
Fingerprints
+
Diagnostics
+
Telemetry
+
Budgets
```

La regla final será:

> **Cuando VoltStack no entienda completamente una expresión, no fingirá entenderla. El framework permitirá Raw de forma explícita, conservará lo que sí conoce, marcará lo que desconoce mediante barreras semánticas y nunca sacrificará parameter binding, seguridad, trazabilidad ni separación arquitectónica sólo para aceptar un fragmento SQL.**

---

# 212. Estado del Bloque 4 — Query Builder

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
51_DATABASE_AGGREGATION_AND_GROUPING_SYSTEM.md
52_DATABASE_WINDOW_FUNCTION_SYSTEM.md
53_DATABASE_RAW_EXPRESSION_AND_ESCAPE_HATCH_SYSTEM.md
```

Sólo resta un documento para cerrar el bloque:

```text
54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md
```

---

# 213. Siguiente documento

```text
54_DATABASE_QUERY_BUILDER_EXTENSION_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
Query Builder Extensions
Builder Macros
Typed Builder Extensions
Semantic Query Extensions
Custom Query Nodes
Custom Predicates
Custom Expressions
Custom Relations
Custom Joins
Custom Aggregates
Custom Windows
Custom Set Operations

Extension Registration
Extension Descriptors
Extension Versioning
Extension Capabilities
Extension Validation
Extension Semantic Analysis
Extension Optimization
Extension Planning
Extension Compilation

Extension Fingerprinting
Extension Security
Extension Portability
Extension Diagnostics
Extension Testing
Extension Compatibility
Extension Lifecycle
Frozen Registries
Persistent Runtime Safety
```

manteniendo la distinción fundamental:

```text
DX Macro
≠
Semantic Query Extension
```

y:

```text
Extension
≠
Raw SQL Shortcut
```