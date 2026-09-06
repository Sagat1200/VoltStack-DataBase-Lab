# 73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md

# VoltStack Quantum Database
## SQL Compiler Extension System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 73 — SQL Compiler Extension System  
**Bloque:** 6 — SQL Compiler  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`SQL Compiler Extension System` define la arquitectura mediante la cual VoltStack podrá ampliar el SQL Compiler sin modificar directamente su núcleo.

El sistema permitirá incorporar:

- nuevos statements;
- nuevas expressions;
- nuevos predicates;
- nuevas funciones;
- nuevos operadores;
- nuevos tipos;
- nuevos constructs SQL;
- nuevas estrategias de representación;
- nuevas capacidades de plataforma;
- extensiones específicas de MySQL;
- extensiones específicas de MariaDB;
- extensiones específicas de PostgreSQL;
- extensiones específicas de SQLite;
- extensiones oficiales de VoltStack;
- extensiones de paquetes;
- extensiones de terceros.

Principio central:

```text
Compiler Extensibility
≠
Arbitrary SQL Mutation
```

La arquitectura deberá garantizar:

```text
Compiler Extensibility
=
Typed Extension Contracts
+
Declared Semantics
+
Explicit Capabilities
+
Deterministic Registration
+
Validated Composition
+
Security Boundaries
+
Versioned Dependencies
+
Persistent Runtime Safety
```

---

# 2. Objetivo arquitectónico

El sistema deberá permitir:

```text
Core SQL Compiler
        +
Official Extensions
        +
Platform Extensions
        +
Package Extensions
        +
Application Extensions
```

sin convertir el compiler en:

```text
if extension A
else if extension B
else if extension C
...
```

ni permitir:

```text
$sql = extension($sql);
```

como mecanismo general de extensibilidad.

---

# 3. Principio fundamental

Toda extensión deberá declarar:

```text
Identity
Scope
Kind
Capabilities
Dependencies
Conflicts
Ordering Constraints
Supported Platforms
Supported Compiler Versions
Semantic Contract
Compilation Contract
Security Contract
Determinism Contract
Version
Fingerprint Contribution
```

antes de participar en compilation.

---

# 4. Posición arquitectónica

```text
CompilableDatabaseOperation
        │
        ▼
SQL Compiler Pipeline
        │
        ├── Core Compilation Components
        │
        └── SQL Compiler Extension System
                  │
                  ├── Extension Registry
                  ├── Dependency Resolver
                  ├── Conflict Resolver
                  ├── Ordering Resolver
                  ├── Capability Resolver
                  ├── Platform Resolver
                  ├── Extension Validators
                  └── Frozen Extension Graph
        │
        ▼
SQL Generation
        │
        ▼
CompiledDatabaseCommand
```

---

# 5. Extensión ≠ plugin arbitrario

Una compiler extension no será simplemente:

```php
interface Extension
{
    public function handle(string $sql): string;
}
```

Ese diseño permitiría:

- eliminar predicates;
- modificar placeholders;
- romper bindings;
- alterar aliases;
- introducir SQL injection;
- invalidar source maps;
- modificar result contracts;
- alterar semántica;
- romper fingerprints;
- producir SQL no determinista.

Por tanto:

```text
SQL String Postprocessor
≠
Compiler Extension
```

---

# 6. Extension architecture

```text
Extension Package
      │
      ▼
Extension Descriptor
      │
      ▼
Discovery
      │
      ▼
Structural Validation
      │
      ▼
Compatibility Validation
      │
      ▼
Dependency Resolution
      │
      ▼
Conflict Resolution
      │
      ▼
Ordering Resolution
      │
      ▼
Capability Resolution
      │
      ▼
Extension Graph
      │
      ▼
Freeze
      │
      ▼
Immutable Compiler Extension Registry
```

---

# 7. Lifecycle

El lifecycle formal será:

```text
DISCOVERED
    ↓
DESCRIBED
    ↓
VALIDATED
    ↓
COMPATIBLE
    ↓
DEPENDENCIES_RESOLVED
    ↓
CONFLICTS_RESOLVED
    ↓
ORDER_RESOLVED
    ↓
CAPABILITIES_RESOLVED
    ↓
REGISTERED
    ↓
FROZEN
```

Después de `FROZEN`:

```text
runtime mutation forbidden
```

---

# 8. Extension descriptor

```php
final readonly class SqlCompilerExtensionDescriptor
{
    public function __construct(
        public SqlCompilerExtensionId $id,
        public SqlCompilerExtensionVersion $version,
        public SqlCompilerExtensionKindSet $kinds,
        public SqlCompilerExtensionScope $scope,
        public PlatformConstraintSet $platforms,
        public CompilerVersionConstraint $compilerVersions,
        public ExtensionDependencySet $dependencies,
        public ExtensionConflictSet $conflicts,
        public ExtensionOrderingConstraints $ordering,
        public ExtensionCapabilityDeclaration $capabilities,
        public ExtensionSecurityDescriptor $security,
        public ExtensionDeterminismDescriptor $determinism,
        public ExtensionFingerprintDescriptor $fingerprint,
    ) {}
}
```

---

# 9. Extension identity

```php
final readonly class SqlCompilerExtensionId
{
    public function __construct(
        public string $vendor,
        public string $package,
        public string $name,
    ) {}
}
```

Ejemplo conceptual:

```text
voltstack/database-json/sqlite-json
```

o:

```text
acme/database/postgresql-vector
```

---

# 10. Identity invariants

La identidad deberá ser:

```text
stable
unique
version-independent
case-normalized
deterministic
```

---

# 11. Extension version

```php
final readonly class SqlCompilerExtensionVersion
{
    public function __construct(
        public int $major,
        public int $minor,
        public int $patch,
    ) {}
}
```

---

# 12. Version ≠ capability

```text
Extension Version
≠
Extension Capability Set
```

Una capability deberá declararse explícitamente.

---

# 13. Extension kinds

```php
enum SqlCompilerExtensionKind
{
    case STATEMENT;
    case EXPRESSION;
    case PREDICATE;
    case FUNCTION;
    case OPERATOR;
    case TYPE;
    case RELATION;
    case JOIN;
    case AGGREGATE;
    case WINDOW;
    case CTE;
    case SET_OPERATION;
    case IDENTIFIER;
    case PARAMETER;
    case RESULT;
    case EMISSION_NODE;
    case DIALECT;
    case PLATFORM;
    case COMPILATION_PASS;
}
```

Una extensión podrá declarar varios kinds cuando sean coherentes.

---

# 14. Extension scope

```php
enum SqlCompilerExtensionScope
{
    case PORTABLE;
    case PLATFORM_FAMILY;
    case PLATFORM_SPECIFIC;
    case DIALECT_SPECIFIC;
    case APPLICATION;
}
```

---

# 15. Portable extension

Una extensión portable deberá proporcionar una representación válida para cada target declarado.

Ejemplo:

```text
Semantic Function
        │
        ├── MySQL Adapter
        ├── MariaDB Adapter
        ├── PostgreSQL Adapter
        └── SQLite Adapter
```

---

# 16. Platform-specific extension

Ejemplos:

```text
PostgreSQL vector operators
SQLite FTS5
MySQL optimizer hint construct
MariaDB-specific function
```

No deberán fingirse portables.

---

# 17. Platform constraints

```php
final readonly class PlatformConstraintSet
{
    /** @param list<PlatformConstraint> $constraints */
    public function __construct(
        public array $constraints,
    ) {}
}
```

Un constraint podrá considerar:

```text
platform family
platform identity
capabilities
version range
dialect profile
driver compilation profile
extension capability profile
```

---

# 18. No vendor checks dispersos

Incorrecto:

```php
if ($platform->name() === 'postgresql') {
    // extension behavior
}
```

disperso por toda la extensión.

Preferir:

```text
PlatformConstraint
+
CapabilityRequirement
+
Target Adapter
```

---

# 19. Extension contract

Contrato raíz:

```php
interface SqlCompilerExtension
{
    public function descriptor(): SqlCompilerExtensionDescriptor;
}
```

Este contrato no otorga automáticamente acceso a todos los puntos del compiler.

---

# 20. Capability-specific contracts

Una extensión deberá implementar únicamente los contracts que necesita.

Ejemplo:

```php
interface SqlExpressionCompilerExtension
    extends SqlCompilerExtension
{
    public function expressionCompiler():
        SqlExtensionExpressionCompiler;
}
```

---

# 21. Statement extension

```php
interface SqlStatementCompilerExtension
    extends SqlCompilerExtension
{
    public function supportedStatementKinds():
        StatementKindSet;

    public function compiler():
        SqlExtensionStatementCompiler;
}
```

---

# 22. Statement compiler contract

```php
interface SqlExtensionStatementCompiler
{
    public function compile(
        ExtensionStatementNode $statement,
        SqlExtensionCompilationContext $context,
    ): SqlEmissionNode;
}
```

---

# 23. Statement extension boundary

La extensión podrá representar:

```text
known structured statement
        ↓
SQL emission structure
```

pero no:

```text
unknown semantic intent
        ↓
invent execution semantics
```

---

# 24. Expression extension

```php
interface SqlExpressionCompilerExtension
    extends SqlCompilerExtension
{
    public function expressionKinds():
        ExpressionKindSet;

    public function compiler():
        SqlExtensionExpressionCompiler;
}
```

---

# 25. Expression compilation

```text
ExtensionExpressionNode
        ↓
Extension Expression Compiler
        ↓
SqlEmissionExpression
```

---

# 26. Expression extension invariant

La extensión no deberá:

```text
resolve symbols
infer semantic types
change query cardinality
introduce hidden subqueries
execute SQL
```

salvo que el semantic/physical/execution model ya represente explícitamente esos efectos.

---

# 27. Predicate extension

```php
interface SqlPredicateCompilerExtension
    extends SqlCompilerExtension
{
    public function predicateKinds():
        PredicateKindSet;

    public function compiler():
        SqlExtensionPredicateCompiler;
}
```

---

# 28. Predicate semantics

Todo predicate extension deberá declarar su relación con SQL three-valued logic.

```text
TRUE
FALSE
UNKNOWN
```

no podrá reemplazarse por boolean semantics de PHP.

---

# 29. Predicate descriptor

```php
final readonly class ExtensionPredicateDescriptor
{
    public function __construct(
        public ExtensionPredicateId $id,
        public QueryType $resultType,
        public NullSemanticsDescriptor $nullSemantics,
        public VolatilityDescriptor $volatility,
        public SecurityBehaviorDescriptor $security,
    ) {}
}
```

---

# 30. Function extension

```php
interface SqlFunctionCompilerExtension
    extends SqlCompilerExtension
{
    public function functions():
        iterable;
}
```

Cada función deberá tener descriptor propio.

---

# 31. Function descriptor

```php
final readonly class SqlExtensionFunctionDescriptor
{
    public function __construct(
        public SemanticFunctionId $id,
        public FunctionSignatureSet $signatures,
        public FunctionVolatility $volatility,
        public NullBehaviorDescriptor $nullBehavior,
        public PlatformRepresentationSet $representations,
    ) {}
}
```

---

# 32. Function identity ≠ SQL name

```text
SemanticFunctionId
≠
RenderedFunctionName
```

Por ejemplo:

```text
Semantic:
JsonArrayLength

SQLite:
json_array_length(...)

PostgreSQL:
platform-specific representation

MySQL:
platform-specific representation
```

---

# 33. Function overloading

La resolución de overload semántico deberá ocurrir antes del SQL rendering.

El compiler recibe una función ya resuelta.

---

# 34. Operator extension

```php
interface SqlOperatorCompilerExtension
    extends SqlCompilerExtension
{
    public function operators():
        iterable;
}
```

---

# 35. Operator descriptor

```php
final readonly class SqlExtensionOperatorDescriptor
{
    public function __construct(
        public SemanticOperatorId $id,
        public OperatorArity $arity,
        public OperatorPrecedence $precedence,
        public OperatorAssociativity $associativity,
        public NullSemanticsDescriptor $nullSemantics,
        public PlatformRepresentationSet $representations,
    ) {}
}
```

---

# 36. Operator precedence

Una extensión no podrá confiar únicamente en:

```text
"just concatenate the operator"
```

Deberá declarar:

```text
precedence
associativity
arity
parenthesization requirements
```

---

# 37. Type extension

```php
interface SqlTypeCompilerExtension
    extends SqlCompilerExtension
{
    public function types():
        iterable;
}
```

---

# 38. Type extension responsibility

Podrá proporcionar:

```text
semantic type → target SQL type representation
binding conversion descriptor
result conversion descriptor
cast representation
literal representation where explicitly safe
```

---

# 39. Type extension boundary

No deberá convertir:

```text
Compiler
```

en:

```text
ORM Value Object Hydrator
```

---

# 40. Type mapping ≠ hydration

```text
SQL Type Mapping
≠
Entity Hydration
```

---

# 41. Relation extension

Permitirá representar constructs como:

```text
table-valued functions
virtual tables
special relation sources
extension relations
```

mediante typed relation nodes.

---

# 42. Relation contract

```php
interface SqlRelationCompilerExtension
    extends SqlCompilerExtension
{
    public function relationKinds(): RelationKindSet;

    public function compiler():
        SqlExtensionRelationCompiler;
}
```

---

# 43. Join extension

Sólo deberá utilizarse cuando una plataforma introduzca una representación genuinamente especial.

No deberá utilizarse para que una extensión cambie arbitrariamente el join order.

---

# 44. Join invariant

```text
Compiler Extension
cannot reorder joins
```

La decisión pertenece al Optimizer/Planner.

---

# 45. Aggregate extension

Permitirá:

```text
custom aggregate functions
platform aggregate syntax
ordered aggregates
aggregate-specific modifiers
```

cuando hayan sido modelados previamente.

---

# 46. Window extension

Permitirá ampliar:

```text
window functions
frame syntax
window modifiers
platform-specific window constructs
```

sin redefinir la semántica del Window System.

---

# 47. CTE extension

Podrá representar:

```text
platform-specific CTE modifiers
materialization hints
search/cycle-like constructs
```

cuando existan capabilities y semantic descriptors adecuados.

---

# 48. Set-operation extension

Podrá representar operaciones adicionales sólo si existen como operaciones semánticas explícitas.

No podrá convertir arbitrariamente:

```text
UNION
```

en otra operación.

---

# 49. Dialect extension

```php
interface SqlDialectCompilerExtension
    extends SqlCompilerExtension
{
    public function dialectContribution():
        SqlDialectContribution;
}
```

---

# 50. Dialect contribution

Podrá añadir:

```text
keywords
function spellings
operator spellings
identifier rules
syntactic constructs
emission renderers
```

pero no alterar libremente SQL ya renderizado.

---

# 51. Platform extension

```php
interface SqlPlatformCompilerExtension
    extends SqlCompilerExtension
{
    public function platformContribution():
        SqlPlatformCompilerContribution;
}
```

---

# 52. Platform contribution

Podrá aportar:

```text
capability descriptors
target-specific representations
type mappings
function mappings
operator mappings
statement renderers
validation rules
```

---

# 53. Emission-node extension

La extensión podrá introducir un nuevo nodo de emission.

```php
interface SqlEmissionNodeExtension
    extends SqlCompilerExtension
{
    public function nodeKind(): SqlEmissionNodeKind;

    public function renderer():
        SqlEmissionNodeRenderer;
}
```

---

# 54. Emission node requirements

Todo custom emission node deberá declarar:

```text
identity
children
rendering contract
source provenance
parameter occurrences
identifier dependencies
capability requirements
fingerprint representation
```

---

# 55. Unknown emission nodes

Nunca:

```text
unknown node
→ cast to string
```

Resultado correcto:

```text
unknown node
→ compilation failure
```

---

# 56. Extension compilation pass

VoltStack podrá permitir passes controlados.

```php
interface SqlCompilationExtensionPass
{
    public function descriptor():
        SqlCompilationPassDescriptor;

    public function process(
        SqlCompilationArtifact $artifact,
        SqlCompilationPassContext $context,
    ): SqlCompilationArtifact;
}
```

---

# 57. Compilation pass ≠ arbitrary middleware

Un pass no podrá interceptar cualquier fase sin restricciones.

Cada pass deberá declarar:

```text
allowed phase
input artifact type
output artifact type
ordering
capabilities
mutation scope
determinism
```

---

# 58. Protected core phases

Determinadas fases serán protegidas.

Ejemplo:

```text
Input Validation
Capability Preflight
Security Validation
Binding Validation
Post Compilation Validation
Freeze
```

No podrán eliminarse ni sustituirse mediante extensiones ordinarias.

---

# 59. Extension hook model

Posibles hooks:

```text
BEFORE_SQL_LOWERING
SQL_LOWERING
AFTER_SQL_LOWERING
DIALECT_ADAPTATION
BEFORE_IDENTIFIER_PLANNING
BEFORE_PARAMETER_PLANNING
BEFORE_RENDERING
EMISSION_RENDERING
AFTER_RENDERING_INSPECTION
RESULT_CONTRACT_COMPILATION
DEPENDENCY_COLLECTION
FINGERPRINT_CONTRIBUTION
DIAGNOSTIC_ENRICHMENT
```

---

# 60. Mutation vs inspection hooks

Los hooks deberán clasificarse como:

```text
TRANSFORM
CONTRIBUTE
VALIDATE
INSPECT
```

---

# 61. Post-render hook

Por defecto:

```text
AFTER_RENDERING
=
INSPECT ONLY
```

No:

```text
AFTER_RENDERING
=
rewrite arbitrary SQL string
```

---

# 62. Why post-render mutation is forbidden

Modificar SQL después del rendering podría invalidar:

```text
placeholder plan
binding layout
source map
result contract
fingerprint
security provenance
dependency set
```

---

# 63. Extension registry

```php
interface SqlCompilerExtensionRegistry
{
    public function get(
        SqlCompilerExtensionId $id,
    ): SqlCompilerExtension;

    public function has(
        SqlCompilerExtensionId $id,
    ): bool;

    public function all():
        iterable;

    public function byKind(
        SqlCompilerExtensionKind $kind,
    ): iterable;
}
```

---

# 64. Mutable builder vs frozen registry

Durante bootstrap:

```text
MutableSqlCompilerExtensionRegistryBuilder
```

Después:

```text
FrozenSqlCompilerExtensionRegistry
```

---

# 65. Runtime registration forbidden

Incorrecto:

```php
$compiler->extensions()->register($extension);
```

durante una request.

---

# 66. Bootstrap registration

Correcto:

```text
Application Bootstrap
        ↓
Extension Discovery
        ↓
Extension Registration
        ↓
Validation
        ↓
Freeze
        ↓
Worker Requests
```

---

# 67. Discovery

Las extensiones podrán descubrirse desde:

```text
official package manifests
Composer package metadata
application configuration
service providers/bootstrap descriptors
explicit registration
```

pero discovery y activation deberán ser conceptos distintos.

---

# 68. Discovery ≠ activation

```text
Installed
≠
Enabled
≠
Compatible
≠
Active
```

---

# 69. Extension manifest

Ejemplo conceptual:

```php
final readonly class SqlCompilerExtensionManifest
{
    public function __construct(
        public SqlCompilerExtensionId $id,
        public string $implementation,
        public bool $enabled,
        public ExtensionConfiguration $configuration,
    ) {}
}
```

---

# 70. No arbitrary instantiation

La discovery no deberá ejecutar código arbitrario sólo para descubrir metadata cuando pueda evitarse.

---

# 71. Descriptor-first design

Preferencia:

```text
Manifest
    ↓
Descriptor
    ↓
Compatibility Resolution
    ↓
Instantiate active components
```

---

# 72. Dependency system

Una extensión podrá declarar:

```text
required extensions
optional extensions
required capabilities
required platform features
required compiler version
```

---

# 73. Required dependency

```php
final readonly class RequiredExtensionDependency
{
    public function __construct(
        public SqlCompilerExtensionId $id,
        public ExtensionVersionConstraint $version,
    ) {}
}
```

---

# 74. Optional dependency

Una optional dependency podrá habilitar integración adicional.

Su ausencia no deberá hacer fallar la extensión base.

---

# 75. Dependency graph

```text
Extension A
    │
    ├── requires B
    │       │
    │       └── requires D
    │
    └── optionally integrates C
```

El resolver deberá construir un DAG válido.

---

# 76. Cyclic dependencies

```text
A requires B
B requires A
```

deberá producir:

```text
SqlCompilerExtensionDependencyCycleException
```

---

# 77. Missing dependency

```text
Extension A requires B
B unavailable
```

resultado:

```text
extension activation failure
```

No:

```text
silently ignore dependency
```

---

# 78. Version compatibility

Ejemplo:

```text
Extension A
requires
Extension B >=2.0 <3.0
```

La resolución deberá ser determinista.

---

# 79. Compiler compatibility

Una extensión deberá declarar qué versiones del extension API soporta.

---

# 80. Extension API version

```php
final readonly class SqlCompilerExtensionApiVersion
{
    public function __construct(
        public int $major,
        public int $minor,
    ) {}
}
```

---

# 81. Package version ≠ API version

```text
Package Version
≠
Extension API Version
```

---

# 82. Conflict system

Una extensión podrá declarar conflictos explícitos.

```php
final readonly class ExtensionConflict
{
    public function __construct(
        public SqlCompilerExtensionId $with,
        public ExtensionConflictReason $reason,
    ) {}
}
```

---

# 83. Structural conflicts

También deberán detectarse conflictos no declarados.

Ejemplos:

```text
two exclusive compilers claim same statement kind
two renderers claim same emission node
two incompatible type mappings claim same semantic type
two operators claim same semantic identity
```

---

# 84. No last-wins

Nunca:

```text
register A
register B
B silently replaces A
```

---

# 85. Conflict resolution

Los conflictos deberán:

```text
be rejected
```

o resolverse mediante una política explícita declarada por el framework.

Nunca por orden incidental de carga.

---

# 86. Override model

Si VoltStack permite override explícito:

```php
final readonly class ExtensionOverrideDeclaration
{
    public function __construct(
        public ExtensionComponentId $target,
        public SqlCompilerExtensionId $replacement,
        public OverrideReason $reason,
    ) {}
}
```

---

# 87. Override constraints

Un override deberá ser:

```text
explicit
validated
version-aware
fingerprinted
diagnosable
```

---

# 88. Ordering system

Algunos extension passes necesitarán orden relativo.

No deberá usarse exclusivamente:

```text
priority = 100
priority = 200
```

---

# 89. Ordering constraints

Preferir:

```text
runsAfter(A)
runsBefore(B)
requires(C)
conflictsWith(D)
```

---

# 90. Ordering descriptor

```php
final readonly class ExtensionOrderingConstraints
{
    public function __construct(
        public ExtensionIdSet $after,
        public ExtensionIdSet $before,
    ) {}
}
```

---

# 91. Ordering graph

```text
A → C
B → C
C → D
```

deberá producir un topological ordering determinista.

---

# 92. Ordering cycle

```text
A before B
B before C
C before A
```

deberá fallar durante bootstrap.

---

# 93. Tie breaking

Cuando dos extensiones sean independientes, el orden deberá definirse mediante una regla estable.

Por ejemplo:

```text
canonical extension identity
```

Nunca:

```text
filesystem discovery order
Composer package enumeration accident
object creation order
```

---

# 94. Capability contribution

Una extensión podrá proporcionar nuevas capabilities.

```php
interface SqlCompilerCapabilityContributor
{
    public function capabilities():
        ExtensionCapabilitySet;
}
```

---

# 95. Capability ownership

Cada capability deberá conocer:

```text
provider
version
scope
platform
requirements
```

---

# 96. Capability contribution ≠ capability fabrication

Una extensión no podrá declarar:

```text
supportsReturning = true
```

si no existe una implementación válida para el target.

---

# 97. Capability proof

El capability resolver deberá poder relacionar:

```text
Capability
        ↓
Provider
        ↓
Representation
        ↓
Validation Contract
```

---

# 98. Native capability

Ejemplo:

```text
PostgreSQL native RETURNING
```

---

# 99. Extension capability

Ejemplo conceptual:

```text
custom PostgreSQL vector operator
```

aportado por una extensión.

---

# 100. Capability composition

```text
Base Platform Capabilities
        +
Driver Compilation Capabilities
        +
Installed Target Extensions
        ↓
Effective Compilation Capability Snapshot
```

---

# 101. Effective capability snapshot

Deberá ser immutable y fingerprinted.

---

# 102. Extension configuration

Cada extensión podrá tener configuración.

```php
final readonly class SqlCompilerExtensionConfiguration
{
    public function __construct(
        public SqlCompilerExtensionId $extension,
        public array $values,
        public ConfigurationFingerprint $fingerprint,
    ) {}
}
```

---

# 103. Configuration constraints

Sólo configuración que pueda afectar compilation deberá formar parte del compiler extension context.

---

# 104. Secrets

Credenciales y secretos no deberán introducirse en extension configuration del SQL Compiler salvo que exista una razón arquitectónica extraordinaria.

---

# 105. Configuration fingerprint

Si una opción cambia el SQL generado:

```text
option
→ fingerprint contribution
```

---

# 106. Runtime configuration

Opciones puramente runtime no deberán invalidar compiled SQL innecesariamente.

---

# 107. Extension context

```php
final readonly class SqlExtensionCompilationContext
{
    public function __construct(
        public SqlCompilationContext $compilation,
        public SqlCompilerExtensionDescriptor $extension,
        public ExtensionConfigurationView $configuration,
        public ExtensionCapabilityView $capabilities,
    ) {}
}
```

---

# 108. Restricted context

El extension context no deberá exponer automáticamente:

```text
ServiceContainer
PDO
Connection
Transaction
EntityManager
HTTP Request
Session
global tenant resolver
filesystem
network
```

---

# 109. No service locator

Incorrecto:

```php
$container = $context->container();
```

---

# 110. Functional extension model

Preferir:

```text
immutable input
+
immutable context
        ↓
deterministic transformation
        ↓
immutable output
```

---

# 111. External I/O forbidden

Durante compilation, una extensión no deberá:

```text
query database
call network
read mutable remote config
write files
run subprocess
```

---

# 112. Local static resources

Si una extensión requiere metadata estática empaquetada, deberá cargarse durante bootstrap y convertirse en immutable descriptors.

---

# 113. Determinism contract

```php
final readonly class ExtensionDeterminismDescriptor
{
    public function __construct(
        public bool $deterministic,
        public DeterminismRequirementSet $requirements,
    ) {}
}
```

---

# 114. Production requirement

Las compiler extensions normales deberán ser deterministas.

---

# 115. Forbidden nondeterminism

No deberán influir:

```text
current time
random values
process ID
memory address
thread ID
request ID
global counters
filesystem ordering
network state
```

---

# 116. Deterministic output invariant

```text
Same Extension Input
+
Same Extension Configuration
+
Same Capability Snapshot
+
Same Extension Version
=
Same Extension Compilation Output
```

---

# 117. Fingerprint contribution

Cada extensión que pueda afectar output deberá aportar una identidad fingerprintable.

---

# 118. Extension fingerprint

Conceptualmente:

```text
ExtensionFingerprint
=
ExtensionId
+
ExtensionVersion
+
ExtensionApiVersion
+
ConfigurationFingerprint
+
CapabilityFingerprint
+
ImplementationFingerprint
```

---

# 119. Aggregate fingerprint

```text
CompilerExtensionSetFingerprint
=
CanonicalHash(
    sorted active extension fingerprints
)
```

---

# 120. Ordering in fingerprint

Si el orden de passes afecta el resultado:

```text
resolved pass graph/order
```

deberá formar parte del fingerprint.

---

# 121. Extension dependency contribution

Los compiled commands deberán registrar las extensiones de las que dependen realmente.

---

# 122. Dependency precision

Una query ordinaria que no utilice una extensión opcional no deberá necesariamente depender de ella sólo porque esté instalada.

---

# 123. Used-extension tracking

```text
Compilation Session
        ↓
UsedExtensionTracker
        ↓
CompilationDependencySet
```

---

# 124. Extension usage

Se considera usada cuando participa materialmente en:

```text
representation
validation
type mapping
function rendering
operator rendering
emission rendering
result contract
```

---

# 125. Source provenance

Toda emission producida por una extensión deberá mantener:

```text
extension identity
source semantic node
source query node
source physical/execution node where applicable
```

---

# 126. Source map integration

```text
Extension Source
        ↓
Emission Node
        ↓
Rendered SQL Range
        ↓
SqlSourceMap
```

---

# 127. No anonymous generated SQL

Una extensión no deberá producir SQL significativo sin provenance.

---

# 128. Diagnostics

Los diagnostics de extensión deberán incluir:

```text
extension id
extension version
component kind
phase
source node
platform
capability
error code
```

---

# 129. Diagnostic codes

Ejemplos:

```text
SQL_EXT_DEPENDENCY_MISSING
SQL_EXT_DEPENDENCY_CYCLE
SQL_EXT_CONFLICT
SQL_EXT_ORDERING_CYCLE
SQL_EXT_UNSUPPORTED_PLATFORM
SQL_EXT_UNSUPPORTED_COMPILER_VERSION
SQL_EXT_CAPABILITY_MISSING
SQL_EXT_COMPONENT_COLLISION
SQL_EXT_SECURITY_VIOLATION
SQL_EXT_NON_DETERMINISTIC
SQL_EXT_INVALID_EMISSION
SQL_EXT_BUDGET_EXCEEDED
```

---

# 130. Extension exception hierarchy

```text
SqlCompilerExtensionException
├── SqlCompilerExtensionDiscoveryException
├── SqlCompilerExtensionDescriptorException
├── SqlCompilerExtensionCompatibilityException
├── SqlCompilerExtensionDependencyException
├── SqlCompilerExtensionDependencyCycleException
├── SqlCompilerExtensionConflictException
├── SqlCompilerExtensionOrderingException
├── SqlCompilerExtensionOrderingCycleException
├── SqlCompilerExtensionCapabilityException
├── SqlCompilerExtensionSecurityException
├── SqlCompilerExtensionDeterminismException
├── SqlCompilerExtensionEmissionException
├── SqlCompilerExtensionBudgetException
└── SqlCompilerExtensionInvariantException
```

---

# 131. Security architecture

El Extension System deberá considerarse una frontera de seguridad.

Una extensión participa en la generación de SQL que llegará al database.

---

# 132. Security rule

```text
Extension Code
does not imply
Unrestricted SQL Authority
```

---

# 133. Mandatory security semantics

Una extensión no podrá eliminar:

```text
tenant predicates
authorization predicates
mandatory filters
locking requirements
mutation restrictions
security barriers
```

---

# 134. Security provenance

Los nodos security-sensitive deberán portar metadata que permita verificar su conservación.

---

# 135. Security validator

```php
interface SqlCompilerExtensionSecurityValidator
{
    public function validate(
        SqlCompilationArtifact $before,
        SqlCompilationArtifact $after,
        SqlCompilerExtensionDescriptor $extension,
    ): void;
}
```

---

# 136. Protected nodes

El sistema podrá marcar:

```text
SECURITY_PROTECTED
SEMANTIC_PROTECTED
MUTATION_PROTECTED
LOCKING_PROTECTED
CARDINALITY_PROTECTED
```

---

# 137. Protected-node transformation

Una extensión sólo podrá transformar un protected node mediante un contract explícitamente autorizado que demuestre equivalencia representacional.

---

# 138. Security predicate removal

```text
before:
tenant_id = P1

after:
<missing>
```

deberá producir:

```text
SqlCompilerExtensionSecurityException
```

---

# 139. Raw SQL extensions

Las extensiones podrán usar raw SQL únicamente mediante el raw-expression architecture formal.

---

# 140. Raw extension fragment

```php
final readonly class ExtensionRawSqlFragment
{
    public function __construct(
        public SqlCompilerExtensionId $owner,
        public TrustedRawSql $sql,
        public ParameterOccurrenceSet $parameters,
        public RawSqlSecurityDescriptor $security,
    ) {}
}
```

---

# 141. Raw SQL cannot bypass binding model

Runtime values deberán continuar utilizando bindings.

---

# 142. Raw identifiers

No deberán concatenarse identifiers no validados dentro de raw fragments.

---

# 143. Trust levels

```text
CORE_TRUSTED
OFFICIAL_EXTENSION_TRUSTED
PACKAGE_EXTENSION_RESTRICTED
APPLICATION_EXPLICIT_RAW
```

podrán utilizarse para diagnostics/policies.

No deberán cambiar por sí solos la semántica de seguridad.

---

# 144. Trust ≠ correctness

Incluso una extensión oficial deberá pasar:

```text
validation
capability checks
binding checks
source-map checks
fingerprint checks
```

---

# 145. Extension budget

Las extensiones compartirán el global compilation budget.

---

# 146. Per-extension budget

También podrá existir:

```text
max generated nodes
max recursion depth
max pass invocations
max generated parameters
max source map entries
```

por extensión.

---

# 147. Budget accounting

```text
Global Compilation Budget
        ├── Core Consumption
        └── Extension Consumption
                ├── Extension A
                ├── Extension B
                └── Extension C
```

---

# 148. Budget bypass forbidden

Una extensión no podrá crear un segundo allocator sin accounting.

---

# 149. Recursive extension expansion

Debe existir protección contra:

```text
Extension A node
→ expands to A node
→ expands to A node
→ ...
```

---

# 150. Expansion depth

Se aplicará un límite explícito.

---

# 151. Pass fixpoint

Si algún extension pass opera hasta fixpoint:

```text
bounded fixpoint
```

será obligatorio.

---

# 152. Unbounded loops forbidden

Nunca:

```php
while ($changed) {
    // no iteration budget
}
```

---

# 153. Extension composition

Varias extensiones podrán cooperar.

Ejemplo:

```text
Vector Semantic Type Extension
        ↓
PostgreSQL Vector Operator Extension
        ↓
PostgreSQL Vector Function Extension
        ↓
SQL Compiler
```

---

# 154. Composition contract

La cooperación deberá ser declarada mediante:

```text
dependencies
optional integrations
capabilities
ordering
```

no mediante búsqueda global improvisada.

---

# 155. Optional integration

Ejemplo:

```text
Extension A
works alone

if Extension B exists:
    A+B integration component becomes active
```

---

# 156. Integration component

```php
interface SqlCompilerExtensionIntegration
{
    public function descriptor():
        ExtensionIntegrationDescriptor;
}
```

---

# 157. Integration identity

Una integración A+B deberá tener identidad/fingerprint propio si altera compilation.

---

# 158. Platform adapters

Una extensión portable podrá tener:

```text
Extension Core
├── MySQL Adapter
├── MariaDB Adapter
├── PostgreSQL Adapter
└── SQLite Adapter
```

---

# 159. Adapter selection

```text
Semantic Extension Node
        ↓
Extension Registry
        ↓
Target Capability Resolution
        ↓
Platform Adapter
        ↓
Emission
```

---

# 160. Missing platform adapter

Si una extensión se utiliza en un target sin representación:

```text
→ UnsupportedExtensionPlatformException
```

---

# 161. No silent raw fallback

Nunca:

```text
platform adapter missing
→ use raw SQL string from another platform
```

---

# 162. MySQL extension scope

Ejemplos:

```text
MySQL-specific functions
optimizer hints
JSON representations
locking syntax
special DML constructs
```

---

# 163. MariaDB extension scope

MariaDB deberá tener adapters independientes cuando diverja de MySQL.

```text
MariaDB
≠
MySQL alias
```

---

# 164. PostgreSQL extension scope

Ejemplos:

```text
custom operators
extensions
range types
array operators
vector types
full-text constructs
special functions
```

---

# 165. SQLite extension scope

Ejemplos:

```text
FTS5
RTree
virtual tables
custom functions
custom collations
JSON target constructs
```

---

# 166. Extension-provided functions

Una extension podrá registrar una semantic function sólo si existe una definición semántica suficiente.

---

# 167. Compiler-only function aliases

Una simple diferencia de spelling podrá registrarse como representation mapping, sin crear una nueva semantic function.

---

# 168. Semantic extension boundary

Si una extensión introduce una operación con nueva semántica, también deberá extender las capas anteriores necesarias:

```text
Query AST
Semantic Engine
Type System
Optimizer rules if needed
Planner
Execution Plan
SQL Compiler
```

---

# 169. Compiler extension cannot create semantics retroactively

```text
Unknown semantics
        ↓
SQL Compiler Extension
```

es inválido.

---

# 170. Cross-layer extension manifest

Para features complejas podrá existir:

```php
final readonly class DatabaseFeatureExtensionDescriptor
{
    public function __construct(
        public FeatureExtensionId $id,
        public QueryExtensionDescriptor $query,
        public SemanticExtensionDescriptor $semantic,
        public PlannerExtensionDescriptor $planner,
        public CompilerExtensionDescriptor $compiler,
    ) {}
}
```

---

# 171. Compiler-only extension use cases

Son apropiadas cuando sólo cambia:

```text
SQL spelling
target representation
target function mapping
target operator mapping
target type mapping
target-specific syntax
```

---

# 172. Not appropriate for compiler-only extension

No es suficiente cuando cambia:

```text
query meaning
cardinality
side effects
transaction behavior
execution multiplicity
distributed behavior
security semantics
```

---

# 173. Extension registry partitioning

Internamente podrán existir:

```text
StatementExtensionRegistry
ExpressionExtensionRegistry
PredicateExtensionRegistry
FunctionExtensionRegistry
OperatorExtensionRegistry
TypeExtensionRegistry
RelationExtensionRegistry
EmissionExtensionRegistry
PassExtensionRegistry
```

coordinados por un root registry.

---

# 174. Specialized registry principle

Preferir:

```text
small typed registries
```

sobre:

```text
one giant map<string, mixed>
```

---

# 175. Component resolution

```text
Component Request
        ↓
Typed Registry
        ↓
Candidate Set
        ↓
Compatibility Filter
        ↓
Capability Filter
        ↓
Conflict Rules
        ↓
Deterministic Selection
```

---

# 176. Resolution result

```php
final readonly class ResolvedExtensionComponent
{
    public function __construct(
        public ExtensionComponentId $id,
        public SqlCompilerExtensionId $owner,
        public object $component,
        public ResolutionEvidence $evidence,
    ) {}
}
```

---

# 177. Resolution evidence

Podrá registrar:

```text
why selected
platform match
capability match
version match
override rule
dependency path
```

---

# 178. Explain integration

VoltStack podrá mostrar:

```text
SQL Compilation
├── Core compiler
├── PostgreSQL compiler
├── Extension: acme/vector
│   ├── vector type mapping
│   ├── distance operator
│   └── cosine_distance function
└── Renderer
```

---

# 179. Extension trace

Debug mode podrá mostrar:

```text
extension discovered
extension activated
dependency resolved
component selected
pass executed
capability consumed
emission produced
```

sin exponer valores sensibles.

---

# 180. Production trace

Deberá poder reducirse para minimizar overhead.

---

# 181. Instrumentation

El extension system podrá emitir instrumentation events conceptuales:

```text
extension.resolve.start
extension.resolve.end
extension.compile.start
extension.compile.end
extension.validation.failure
extension.budget.exhausted
```

sin depender directamente de OpenTelemetry.

---

# 182. Telemetry boundary

```text
Compiler Extension System
        ↓
Instrumentation Contract
        ↓
Database Telemetry Integration
```

---

# 183. Performance requirements

El lookup de una extensión activa no deberá requerir reconstruir el dependency graph por query.

---

# 184. Bootstrap compilation

Durante bootstrap:

```text
extension graph
        ↓
resolve
        ↓
validate
        ↓
compile registries
        ↓
freeze
```

---

# 185. Runtime lookup

Durante compilation:

```text
typed key
→ frozen lookup
→ resolved component
```

---

# 186. No runtime graph solving

No:

```text
each query
→ Composer scan
→ dependency resolution
→ conflict resolution
```

---

# 187. Compiled extension registry

Podrá existir:

```php
final readonly class CompiledSqlCompilerExtensionRegistry
{
    public function __construct(
        public StatementExtensionTable $statements,
        public ExpressionExtensionTable $expressions,
        public PredicateExtensionTable $predicates,
        public FunctionExtensionTable $functions,
        public OperatorExtensionTable $operators,
        public TypeExtensionTable $types,
        public RelationExtensionTable $relations,
        public EmissionExtensionTable $emissions,
        public CompilationPassTable $passes,
        public CompilerExtensionSetFingerprint $fingerprint,
    ) {}
}
```

---

# 188. Memory model

Compartible entre workers:

```text
immutable descriptors
compiled lookup tables
dependency graph
resolved ordering
fingerprints
```

Operation-local:

```text
used-extension tracker
extension diagnostics
extension pass state
budget counters
temporary emission nodes
```

---

# 189. FrankenPHP safety

El registry congelado podrá reutilizarse entre requests.

Pero nunca deberá almacenar:

```text
current query
current tenant
current parameters
current connection
current transaction
current source map builder
```

---

# 190. RoadRunner/OpenSwoole compatibility

La misma regla de aislamiento aplicará para futuros runtime adapters.

---

# 191. Tenant isolation

Una extensión no deberá conservar tenant state dentro de instancias compartidas.

---

# 192. Tenant-specific compiler capability

Si excepcionalmente una capability depende del tenant:

```text
Tenant Context
        ↓
explicit immutable compilation capability snapshot
```

No:

```text
extension reads global current tenant
```

---

# 193. Security isolation

```text
Tenant A compilation
≠
Tenant B extension mutable state
```

---

# 194. Cache interaction

Compiled-query caching deberá considerar las extensiones realmente relevantes.

---

# 195. Cache key contribution

Conceptualmente:

```text
CompiledQueryCacheKey
=
Operation Fingerprint
+
Platform Fingerprint
+
Compiler Fingerprint
+
Used Extension Fingerprint
+
Representation Configuration
```

---

# 196. Installed extension ≠ cache dependency

Si una extensión instalada no participa ni cambia effective compiler semantics para una query, no deberá necesariamente invalidar su compiled command.

---

# 197. Registry fingerprint vs used fingerprint

Distinguir:

```text
FullExtensionRegistryFingerprint
```

de:

```text
UsedExtensionSetFingerprint
```

---

# 198. Full registry fingerprint

Útil para:

```text
bootstrap diagnostics
environment identity
registry cache
```

---

# 199. Used extension fingerprint

Útil para:

```text
compiled query dependency
compiled query cache
```

---

# 200. Prepared statement integration

El próximo Prepared Statement Compilation System deberá consumir el resultado de extensiones sin necesitar conocer sus implementaciones internas.

---

# 201. Extension parameter descriptors

Una extensión podrá generar parameter occurrences estructurados.

Nunca valores runtime.

---

# 202. Parameter invariant

Todo parameter generado deberá entrar al mismo:

```text
Placeholder Plan
+
Binding Layout
```

del compiler principal.

---

# 203. No private placeholder allocator

Incorrecto:

```php
$placeholder = ':ext_' . ++$this->counter;
```

---

# 204. Shared operation allocator

Correcto:

```text
Extension
        ↓
ParameterOccurrence Request
        ↓
Operation Placeholder Planner
```

---

# 205. Identifier invariant

Lo mismo aplica a aliases.

Una extensión deberá solicitar:

```text
AliasId
```

al allocator operation-scoped.

---

# 206. No extension global alias counter

Nunca:

```php
static $aliasCounter;
```

---

# 207. Result contract extension

Una extensión que produzca output deberá declarar su shape.

---

# 208. Result descriptor

```php
interface SqlResultContractExtension
    extends SqlCompilerExtension
{
    public function contribute(
        ExtensionResultDescriptor $result,
        ResultContractBuilder $builder,
    ): void;
}
```

---

# 209. Result extension cannot hydrate entities

Sólo podrá describir:

```text
columns
types
ordinals
driver labels
semantic mappings
conversion descriptors
```

---

# 210. Extension result ≠ ORM entity

```text
Compiled Result Contract
≠
Entity Hydration Plan
```

---

# 211. Compilation dependency extension

```php
interface SqlCompilationDependencyContributor
{
    public function dependencies(
        SqlExtensionCompilationContext $context,
    ): CompilationDependencySet;
}
```

---

# 212. Dependency types

Podrán incluir:

```text
extension version
platform capability
custom type
custom function
custom operator
custom collation
virtual table module
compiler API version
extension configuration
```

---

# 213. Hard dependency

Si cambia, compiled SQL deja de ser válido.

---

# 214. Soft diagnostic dependency

Podrán existir dependencies informativas que no invaliden el artifact.

Pero deberán distinguirse explícitamente.

---

# 215. No dependency guessing

El cache system no deberá inferir dependencies parseando SQL.

---

# 216. Serialization

Los extension descriptors y compiled extension references deberán poder serializarse cuando el cache architecture lo requiera.

---

# 217. No closures in cached descriptors

Evitar:

```php
'compiler' => fn (...) => ...
```

en artifacts serializables.

---

# 218. Component identity serialization

Preferir:

```text
ExtensionComponentId
+
ExtensionVersion
+
ImplementationDescriptor
```

---

# 219. Unknown cached extension

Si se carga un compiled artifact cuya required extension ya no existe:

```text
→ cache miss / invalidation
```

No ejecución insegura.

---

# 220. Version mismatch

Igualmente:

```text
cached extension version
≠
compatible active extension version
```

deberá provocar invalidación cuando corresponda.

---

# 221. Extension deprecation

Una extensión podrá declarar:

```text
deprecated component
replacement component
removal version
migration note
```

---

# 222. Deprecation does not change semantics

La existencia de warning no deberá modificar el SQL.

---

# 223. Backward compatibility

El Extension API deberá seguir la política general de compatibilidad de VoltStack.

---

# 224. Extension API stability

Contracts públicos deberán minimizar exposición de clases internas del compiler.

---

# 225. Public extension SPI

Se recomienda separar:

```text
Compiler Public API
```

de:

```text
Compiler Extension SPI
```

---

# 226. SPI

`Service Provider Interface` incluirá únicamente los contracts diseñados para extensiones.

---

# 227. Internal compiler classes

No deberán considerarse automáticamente estables para terceros.

---

# 228. Extension compatibility levels

```php
enum ExtensionApiStability
{
    case STABLE;
    case EXPERIMENTAL;
    case INTERNAL;
}
```

---

# 229. Official extensions

Las extensiones oficiales también deberán usar el SPI cuando sea razonable.

Esto permite validar que el sistema de extensión es realmente suficiente.

---

# 230. Core privilege

Sólo componentes estrictamente internos podrán utilizar APIs privilegiadas.

---

# 231. Privileged extension

Si alguna extensión oficial requiere privilegios especiales, deberá declararse explícitamente.

---

# 232. Privileged ≠ unrestricted

Incluso una extensión privilegiada deberá conservar invariantes semánticos y de seguridad.

---

# 233. Sandboxing

PHP no ofrece aislamiento de seguridad fuerte simplemente por usar interfaces.

Por tanto:

```text
Extension Contract Safety
≠
Malicious Code Sandbox
```

---

# 234. Threat model

El Extension System protege principalmente contra:

```text
architectural misuse
accidental semantic corruption
non-deterministic composition
unsafe SQL generation patterns
state leakage
component conflicts
```

No pretende ejecutar código PHP malicioso de forma segura.

---

# 235. Package trust

La instalación de una extensión PHP implica confianza de ejecución a nivel aplicación.

Aun así, el compiler impondrá contracts estructurales.

---

# 236. Testing contract for extensions

Toda extensión deberá poder ejecutar un conformance suite.

---

# 237. Conformance suite

```text
descriptor validation
identity stability
dependency validation
platform compatibility
capability correctness
determinism
binding safety
identifier safety
source-map provenance
fingerprint stability
persistent-worker isolation
concurrency
budget compliance
security preservation
```

---

# 238. Extension test kit

VoltStack podrá proporcionar:

```php
abstract class SqlCompilerExtensionTestCase
{
    // reusable extension conformance assertions
}
```

---

# 239. Determinism test

```text
compile N times
        ↓
compare canonical artifacts
        ↓
must be identical
```

---

# 240. Concurrency test

```text
compile A
compile B
compile C
in concurrent execution
```

sin cross-operation state leakage.

---

# 241. Security preservation test

Un protected predicate deberá permanecer presente después de cada transform pass autorizado.

---

# 242. Binding test

Toda placeholder occurrence deberá tener binding descriptor válido.

---

# 243. Source map test

Todo SQL significativo producido por la extensión deberá poder relacionarse con provenance suficiente.

---

# 244. Fingerprint test

Cambiar una opción que modifica SQL deberá cambiar el fingerprint.

---

# 245. Non-semantic configuration test

Cambiar configuración irrelevante para compilation no deberá cambiarlo innecesariamente.

---

# 246. Platform matrix

Las extensiones portables deberán probar todos sus adapters soportados.

---

# 247. Unsupported target test

Un target no soportado deberá fallar explícitamente.

---

# 248. Failure injection

La suite deberá poder simular fallos durante:

```text
discovery
validation
dependency resolution
ordering
compilation
rendering
dependency collection
fingerprinting
```

---

# 249. Atomicity test

Un fallo de extensión nunca deberá publicar un:

```text
partially valid CompiledDatabaseCommand
```

---

# 250. Performance tests

Medir:

```text
registry lookup
component resolution
extension pass overhead
emission expansion
fingerprint contribution
diagnostic overhead
```

---

# 251. Bootstrap performance

Dependency/conflict/order solving se mide separadamente del per-query compilation.

---

# 252. Runtime target

El runtime path deberá aproximarse a:

```text
typed lookup
→ component invocation
```

no a graph resolution.

---

# 253. Anti-pattern — string postprocessor

Incorrecto:

```php
public function afterCompile(string $sql): string
{
    return str_replace(...);
}
```

como mecanismo general.

---

# 254. Anti-pattern — regex SQL rewriting

Incorrecto:

```php
preg_replace('/SELECT/', 'SELECT SQL_NO_CACHE', $sql);
```

---

# 255. Anti-pattern — parse generated SQL

Incorrecto:

```text
render
→ extension reparses SQL
→ modifies it
```

---

# 256. Anti-pattern — hidden query

Incorrecto:

```php
$connection->query('SELECT version()');
```

desde la extensión.

---

# 257. Anti-pattern — service locator

Incorrecto:

```php
app(Connection::class)
```

durante compilation.

---

# 258. Anti-pattern — global mutable registry

Incorrecto:

```php
GlobalCompilerExtensions::$extensions[] = ...
```

---

# 259. Anti-pattern — last registered wins

Incorrecto:

```text
A claims function
B claims function
→ B wins
```

---

# 260. Anti-pattern — numeric priority only

Incorrecto:

```text
priority 100
priority 101
priority 102
```

sin dependency semantics.

---

# 261. Anti-pattern — arbitrary raw SQL

Incorrecto:

```php
return new RawSql($userProvidedString);
```

---

# 262. Anti-pattern — hidden semantic rewrite

Incorrecto:

```text
Extension receives unsupported operation
→ changes meaning to something executable
```

---

# 263. Anti-pattern — extension-owned parameter numbering

Incorrecto:

```text
Extension A → $1
Extension B → $1
```

---

# 264. Anti-pattern — runtime tenant state

Incorrecto:

```php
$this->tenantId = $currentTenant;
```

en shared compiler extension.

---

# 265. Anti-pattern — connection-specific leakage

Incorrecto:

```text
Connection A supports custom function
→ extension caches globally
→ Connection B compiles same function
```

---

# 266. Anti-pattern — installed means supported

Incorrecto:

```text
package installed
→ feature assumed available
```

---

# 267. Anti-pattern — package version as capability

Incorrecto:

```text
version >= X
→ capability automatically true
```

sin resolver target conditions.

---

# 268. Anti-pattern — arbitrary compiler override

Incorrecto:

```text
Extension replaces entire compiler
```

sin explicit override contract.

---

# 269. Anti-pattern — extension changes result shape secretly

Incorrecto:

```text
original output: 2 columns
extension output: 3 columns
```

sin actualizar el ResultContract y sin que la semántica lo permita.

---

# 270. Anti-pattern — extension removes security

Siempre prohibido.

---

# 271. Ejemplo — PostgreSQL vector extension

Supongamos un paquete:

```text
voltstack/database-postgresql-vector
```

que introduce:

```text
VectorType
CosineDistance
L2Distance
InnerProduct
```

---

# 272. Descriptor

```php
new SqlCompilerExtensionDescriptor(
    id: new SqlCompilerExtensionId(
        'voltstack',
        'database-postgresql-vector',
        'compiler',
    ),
    version: new SqlCompilerExtensionVersion(1, 0, 0),
    kinds: SqlCompilerExtensionKindSet::of(
        SqlCompilerExtensionKind::TYPE,
        SqlCompilerExtensionKind::FUNCTION,
        SqlCompilerExtensionKind::OPERATOR,
    ),
    scope: SqlCompilerExtensionScope::PLATFORM_SPECIFIC,
    platforms: PlatformConstraintSet::postgresql(),
    // ...
);
```

---

# 273. Semantic input

```text
CosineDistance(
    products.embedding,
    P1<Vector>
)
```

---

# 274. Compiler resolution

```text
SemanticFunctionId::CosineDistance
        ↓
Function Extension Registry
        ↓
PostgreSQL Vector Extension
        ↓
PostgreSQL representation
```

---

# 275. Output

Conceptualmente:

```sql
"products"."embedding" <=> $1
```

si ésa es la representación declarada por la extensión/target.

---

# 276. What extension does not do

No deberá:

```text
discover extension installation via query
infer Vector semantic type
choose nearest-neighbor index
choose query ordering
execute query
```

---

# 277. Physical planning integration

La elección de un vector index pertenecería al Physical Planner.

---

# 278. Ejemplo — SQLite FTS extension

```text
SQLiteFts5CompilerExtension
```

podrá registrar:

```text
FTS relation
MATCH-like semantic predicate
FTS-specific functions
```

si existe la integración semántica correspondiente.

---

# 279. Capability requirement

```text
requires:
SQLiteFeature::FTS5
```

---

# 280. Missing capability

Si el SQLite target no posee FTS5:

```text
→ SQLite extension capability failure
```

---

# 281. Ejemplo — portable UUID type

Una extensión portable podría modelar:

```text
Uuid Semantic Type
```

con adapters:

```text
PostgreSQL → native uuid
MySQL     → configured binary/text representation
MariaDB   → configured representation
SQLite    → text/blob representation
```

---

# 282. Type representation does not redefine UUID

```text
UUID Semantic Type
```

permanece estable aunque cambie el storage representation.

---

# 283. Ejemplo — custom function

Aplicación define:

```text
SemanticFunctionId:
NormalizePhoneNumber
```

y registra una función equivalente en una conexión SQLite.

La compiler extension sólo podrá utilizarla si:

```text
Effective Capability Snapshot
```

declara su disponibilidad.

---

# 284. No assumption from PHP registration

El hecho de que algún bootstrap haya registrado la función en una conexión no autoriza automáticamente a todos los targets.

---

# 285. Example dependency graph

```text
DatabaseJsonExtension
        │
        ├── requires CoreJsonSemanticExtension
        │
        ├── optional PostgreSQLJsonPathExtension
        │
        └── optional SQLiteJsonOperatorExtension
```

---

# 286. Example ordering

```text
Core SQL Lowering
        ↓
JSON Semantic Emission
        ↓
Platform JSON Adaptation
        ↓
Identifier Planning
        ↓
Placeholder Planning
        ↓
Rendering
```

Una extensión no podrá mover `Platform JSON Adaptation` después del final binding validation.

---

# 287. Extension graph model

```php
final readonly class SqlCompilerExtensionGraph
{
    public function __construct(
        public ExtensionNodeSet $nodes,
        public ExtensionDependencyEdgeSet $dependencies,
        public ExtensionOrderingEdgeSet $ordering,
        public ExtensionConflictSet $conflicts,
    ) {}
}
```

---

# 288. Graph edge kinds

```text
REQUIRES
OPTIONAL_INTEGRATION
RUNS_BEFORE
RUNS_AFTER
OVERRIDES
CONFLICTS
PROVIDES_CAPABILITY
CONSUMES_CAPABILITY
```

---

# 289. Conflict edges

`CONFLICTS` no forma parte del execution ordering DAG.

Se utiliza para validation.

---

# 290. Optional integration edge

No deberá convertirse accidentalmente en required dependency.

---

# 291. Extension graph validation

Deberá comprobar:

```text
unique identities
valid versions
no required dependency cycles
no ordering cycles
no unresolved conflicts
valid overrides
valid platform scopes
valid capabilities
valid SPI versions
```

---

# 292. Registry compilation

Después de validar el graph:

```text
Extension Graph
        ↓
Registry Compiler
        ↓
Specialized Lookup Tables
        ↓
Frozen Registry
```

---

# 293. Registry compilation is not SQL compilation

Se ejecutará durante bootstrap/configuration compilation.

---

# 294. Development hot reload

Si VoltStack permite hot reload en desarrollo:

```text
old frozen registry
        ↓
build entirely new registry
        ↓
atomic registry replacement
```

No mutación parcial.

---

# 295. Production registry

En producción deberá permanecer estable durante el worker lifecycle salvo mecanismo explícito de reload.

---

# 296. Reload and compiled cache

Un cambio de registry deberá invalidar los artifacts afectados mediante fingerprints/dependencies.

---

# 297. Extension enable/disable

Cambiar:

```text
enabled → disabled
```

deberá producir un nuevo registry generation/fingerprint.

---

# 298. Registry generation

```php
final readonly class SqlCompilerExtensionRegistryGeneration
{
    public function __construct(
        public string $id,
        public CompilerExtensionSetFingerprint $fingerprint,
    ) {}
}
```

---

# 299. Generation ID

No deberá depender necesariamente de un random UUID si se requiere determinismo.

Puede derivarse del fingerprint.

---

# 300. Diagnostics command

VoltStack CLI podrá ofrecer conceptualmente:

```text
php voltstack database:compiler-extensions
```

---

# 301. CLI output

Podrá mostrar:

```text
Extension
Version
Status
Kinds
Platforms
Capabilities
Dependencies
Conflicts
Ordering
API Compatibility
Fingerprint
```

---

# 302. Explain extension

Otro comando conceptual:

```text
php voltstack database:compiler-extension acme/vector
```

---

# 303. Compiler diagnostics integration

Un debug report podrá mostrar:

```text
Active SQL Compiler Extensions
────────────────────────────────────────
voltstack/core-json        1.0.0
acme/postgresql-vector     2.1.0
...
```

---

# 304. Namespace propuesto

```text
VoltStack\Quantum\Database\Query\Compiler\Extension
```

---

# 305. Estructura propuesta

```text
Extension/
├── Contract/
│   ├── SqlCompilerExtension.php
│   ├── SqlStatementCompilerExtension.php
│   ├── SqlExpressionCompilerExtension.php
│   ├── SqlPredicateCompilerExtension.php
│   ├── SqlFunctionCompilerExtension.php
│   ├── SqlOperatorCompilerExtension.php
│   ├── SqlTypeCompilerExtension.php
│   ├── SqlRelationCompilerExtension.php
│   ├── SqlDialectCompilerExtension.php
│   ├── SqlPlatformCompilerExtension.php
│   ├── SqlEmissionNodeExtension.php
│   └── SqlCompilationExtensionPass.php
│
├── Descriptor/
│   ├── SqlCompilerExtensionDescriptor.php
│   ├── SqlCompilerExtensionId.php
│   ├── SqlCompilerExtensionVersion.php
│   ├── SqlCompilerExtensionKind.php
│   ├── SqlCompilerExtensionScope.php
│   └── SqlCompilerExtensionApiVersion.php
│
├── Discovery/
│   ├── SqlCompilerExtensionDiscovery.php
│   ├── SqlCompilerExtensionManifest.php
│   └── ExtensionDiscoveryResult.php
│
├── Registry/
│   ├── SqlCompilerExtensionRegistry.php
│   ├── MutableSqlCompilerExtensionRegistryBuilder.php
│   ├── FrozenSqlCompilerExtensionRegistry.php
│   ├── CompiledSqlCompilerExtensionRegistry.php
│   ├── StatementExtensionRegistry.php
│   ├── ExpressionExtensionRegistry.php
│   ├── PredicateExtensionRegistry.php
│   ├── FunctionExtensionRegistry.php
│   ├── OperatorExtensionRegistry.php
│   ├── TypeExtensionRegistry.php
│   ├── RelationExtensionRegistry.php
│   ├── EmissionExtensionRegistry.php
│   └── CompilationPassRegistry.php
│
├── Graph/
│   ├── SqlCompilerExtensionGraph.php
│   ├── ExtensionGraphBuilder.php
│   ├── ExtensionGraphValidator.php
│   ├── ExtensionDependencyEdge.php
│   ├── ExtensionOrderingEdge.php
│   └── ExtensionGraphResolver.php
│
├── Dependency/
│   ├── ExtensionDependencySet.php
│   ├── RequiredExtensionDependency.php
│   ├── OptionalExtensionDependency.php
│   ├── ExtensionDependencyResolver.php
│   └── ExtensionVersionConstraint.php
│
├── Conflict/
│   ├── ExtensionConflict.php
│   ├── ExtensionConflictSet.php
│   ├── ExtensionConflictResolver.php
│   └── ExtensionOverrideDeclaration.php
│
├── Ordering/
│   ├── ExtensionOrderingConstraints.php
│   ├── ExtensionOrderingResolver.php
│   └── ResolvedExtensionOrdering.php
│
├── Capability/
│   ├── ExtensionCapabilityDeclaration.php
│   ├── ExtensionCapabilitySet.php
│   ├── ExtensionCapabilityResolver.php
│   ├── EffectiveCompilationCapabilitySnapshot.php
│   └── SqlCompilerCapabilityContributor.php
│
├── Configuration/
│   ├── SqlCompilerExtensionConfiguration.php
│   ├── ExtensionConfigurationView.php
│   └── ConfigurationFingerprint.php
│
├── Resolution/
│   ├── ExtensionComponentResolver.php
│   ├── ResolvedExtensionComponent.php
│   └── ResolutionEvidence.php
│
├── Pass/
│   ├── SqlCompilationPassDescriptor.php
│   ├── SqlCompilationPassPhase.php
│   ├── SqlCompilationPassContext.php
│   └── ExtensionPassExecutor.php
│
├── Security/
│   ├── ExtensionSecurityDescriptor.php
│   ├── SqlCompilerExtensionSecurityValidator.php
│   ├── ProtectedCompilationNode.php
│   └── RawSqlSecurityDescriptor.php
│
├── Determinism/
│   ├── ExtensionDeterminismDescriptor.php
│   └── ExtensionDeterminismValidator.php
│
├── Fingerprint/
│   ├── ExtensionFingerprint.php
│   ├── CompilerExtensionSetFingerprint.php
│   ├── UsedExtensionSetFingerprint.php
│   └── ExtensionFingerprintBuilder.php
│
├── Tracking/
│   ├── UsedExtensionTracker.php
│   └── ExtensionUsageRecord.php
│
├── Diagnostic/
│   ├── SqlCompilerExtensionDiagnostic.php
│   ├── ExtensionDiagnosticCode.php
│   └── ExtensionCompilationTrace.php
│
├── Testing/
│   ├── SqlCompilerExtensionTestCase.php
│   └── SqlCompilerExtensionConformanceSuite.php
│
└── Exception/
    ├── SqlCompilerExtensionException.php
    ├── SqlCompilerExtensionDiscoveryException.php
    ├── SqlCompilerExtensionDescriptorException.php
    ├── SqlCompilerExtensionCompatibilityException.php
    ├── SqlCompilerExtensionDependencyException.php
    ├── SqlCompilerExtensionDependencyCycleException.php
    ├── SqlCompilerExtensionConflictException.php
    ├── SqlCompilerExtensionOrderingException.php
    ├── SqlCompilerExtensionOrderingCycleException.php
    ├── SqlCompilerExtensionCapabilityException.php
    ├── SqlCompilerExtensionSecurityException.php
    ├── SqlCompilerExtensionDeterminismException.php
    ├── SqlCompilerExtensionEmissionException.php
    ├── SqlCompilerExtensionBudgetException.php
    └── SqlCompilerExtensionInvariantException.php
```

---

# 306. Invariantes arquitectónicos

## DB-SQLEXT-001

Toda compiler extension tendrá identidad estable.

## DB-SQLEXT-002

Toda compiler extension tendrá versión explícita.

## DB-SQLEXT-003

Package version será distinta de Extension API version.

## DB-SQLEXT-004

Extension version será distinta de capability set.

## DB-SQLEXT-005

Toda extensión declarará su scope.

## DB-SQLEXT-006

Toda extensión declarará sus kinds.

## DB-SQLEXT-007

Toda extensión declarará plataformas compatibles.

## DB-SQLEXT-008

Toda extensión declarará compatibilidad con el Compiler Extension API.

## DB-SQLEXT-009

Discovery será distinta de activation.

## DB-SQLEXT-010

Installed será distinto de enabled.

## DB-SQLEXT-011

Enabled será distinto de compatible.

## DB-SQLEXT-012

Compatible será distinto de active.

## DB-SQLEXT-013

Extension registration ocurrirá antes de freeze.

## DB-SQLEXT-014

Frozen registry no será mutable durante requests.

## DB-SQLEXT-015

No habrá runtime extension registration ordinario.

## DB-SQLEXT-016

No habrá last-wins conflict resolution.

## DB-SQLEXT-017

Required dependencies deberán existir.

## DB-SQLEXT-018

Required dependency cycles estarán prohibidos.

## DB-SQLEXT-019

Optional dependencies no se convertirán implícitamente en required.

## DB-SQLEXT-020

Ordering constraints serán explícitos.

## DB-SQLEXT-021

Ordering cycles estarán prohibidos.

## DB-SQLEXT-022

Ordering no dependerá del filesystem.

## DB-SQLEXT-023

Ordering no dependerá del registration accident.

## DB-SQLEXT-024

Independent extension ordering tendrá deterministic tie-break.

## DB-SQLEXT-025

Compiler extension no será arbitrary SQL postprocessor.

## DB-SQLEXT-026

Compiler extension no podrá reparsear SQL para modificarlo.

## DB-SQLEXT-027

Post-render hooks serán inspect-only por defecto.

## DB-SQLEXT-028

Protected compiler phases no podrán eliminarse.

## DB-SQLEXT-029

Extension passes declararán input y output types.

## DB-SQLEXT-030

Extension passes declararán su phase.

## DB-SQLEXT-031

Extension passes declararán ordering constraints.

## DB-SQLEXT-032

Extension passes serán bounded.

## DB-SQLEXT-033

Fixpoint extension passes serán bounded.

## DB-SQLEXT-034

Toda extensión respetará global compilation budget.

## DB-SQLEXT-035

Una extensión no podrá bypass budget accounting.

## DB-SQLEXT-036

Statement extension recibirá structured statements.

## DB-SQLEXT-037

Expression extension recibirá structured expressions.

## DB-SQLEXT-038

Predicate extension preservará SQL 3VL.

## DB-SQLEXT-039

Function identity será distinta del SQL function name.

## DB-SQLEXT-040

Operator identity será distinta del SQL operator token.

## DB-SQLEXT-041

Operator precedence será explícita.

## DB-SQLEXT-042

Type mapping será distinta de ORM hydration.

## DB-SQLEXT-043

Relation extension no elegirá physical join order.

## DB-SQLEXT-044

Join extension no realizará optimizer duties.

## DB-SQLEXT-045

Aggregate extension no redefinirá aggregate semantics.

## DB-SQLEXT-046

Window extension no redefinirá window semantics.

## DB-SQLEXT-047

CTE extension no redefinirá recursive semantics.

## DB-SQLEXT-048

Set-operation extension preservará multiplicity semantics.

## DB-SQLEXT-049

Dialect extension no tendrá unrestricted post-render access.

## DB-SQLEXT-050

Platform extension será capability-aware.

## DB-SQLEXT-051

Unknown emission nodes fallarán explícitamente.

## DB-SQLEXT-052

Emission nodes tendrán provenance.

## DB-SQLEXT-053

Emission nodes declararán parameter occurrences.

## DB-SQLEXT-054

Emission nodes declararán identifier dependencies.

## DB-SQLEXT-055

Capabilities tendrán providers identificables.

## DB-SQLEXT-056

Capabilities no podrán fabricarse sin implementación válida.

## DB-SQLEXT-057

Effective capability snapshot será immutable.

## DB-SQLEXT-058

Effective capability snapshot será fingerprinted.

## DB-SQLEXT-059

Extension configuration que cambie SQL cambiará fingerprint.

## DB-SQLEXT-060

Runtime-only configuration no deberá invalidar compiled SQL innecesariamente.

## DB-SQLEXT-061

Compiler extension context no expondrá ServiceContainer.

## DB-SQLEXT-062

Compiler extension context no expondrá live Connection.

## DB-SQLEXT-063

Compiler extension context no expondrá Transaction.

## DB-SQLEXT-064

Compiler extension context no expondrá EntityManager.

## DB-SQLEXT-065

Compiler extension no realizará database I/O.

## DB-SQLEXT-066

Compiler extension no realizará network I/O.

## DB-SQLEXT-067

Compiler extension no escribirá files durante compilation.

## DB-SQLEXT-068

Compiler extension deberá ser determinista.

## DB-SQLEXT-069

Current time no influirá en compiler extension output.

## DB-SQLEXT-070

Randomness no influirá en compiler extension output.

## DB-SQLEXT-071

Global mutable counters estarán prohibidos.

## DB-SQLEXT-072

Extension output fingerprint incluirá extension identity.

## DB-SQLEXT-073

Extension output fingerprint incluirá relevante extension version.

## DB-SQLEXT-074

Extension configuration relevante será fingerprinted.

## DB-SQLEXT-075

Used extensions serán tracked.

## DB-SQLEXT-076

Installed extensions serán distintas de used extensions.

## DB-SQLEXT-077

Compiled command dependerá de extensiones realmente relevantes.

## DB-SQLEXT-078

Source map conservará extension provenance.

## DB-SQLEXT-079

Diagnostics identificarán extension owner.

## DB-SQLEXT-080

Diagnostics no expondrán runtime parameter values por defecto.

## DB-SQLEXT-081

Mandatory security predicates estarán protegidos.

## DB-SQLEXT-082

Tenant predicates no podrán eliminarse.

## DB-SQLEXT-083

Authorization predicates no podrán eliminarse.

## DB-SQLEXT-084

Locking requirements no podrán eliminarse.

## DB-SQLEXT-085

Mutation restrictions no podrán eliminarse.

## DB-SQLEXT-086

Protected-node transformations requerirán explicit contract.

## DB-SQLEXT-087

Raw SQL no bypassará parameter binding.

## DB-SQLEXT-088

Raw SQL no autorizará arbitrary identifier interpolation.

## DB-SQLEXT-089

Official extension trust no eliminará validation.

## DB-SQLEXT-090

Extension budget exhaustion fallará explícitamente.

## DB-SQLEXT-091

Extension failure no producirá partial compiled command.

## DB-SQLEXT-092

Compilation seguirá siendo atomic.

## DB-SQLEXT-093

Extension recursion será bounded.

## DB-SQLEXT-094

Extension composition será declarativa.

## DB-SQLEXT-095

Optional integration será explícita.

## DB-SQLEXT-096

Platform adapter selection será determinista.

## DB-SQLEXT-097

Missing platform adapter fallará explícitamente.

## DB-SQLEXT-098

No habrá cross-platform raw fallback.

## DB-SQLEXT-099

MariaDB adapters podrán divergir de MySQL.

## DB-SQLEXT-100

SQLite-specific capabilities no se fingirán portables.

## DB-SQLEXT-101

PostgreSQL-specific capabilities no se fingirán portables.

## DB-SQLEXT-102

Compiler-only extensions no podrán introducir nueva semántica sin soporte upstream.

## DB-SQLEXT-103

Nueva semántica requerirá cross-layer extension cuando corresponda.

## DB-SQLEXT-104

Specialized registries serán typed.

## DB-SQLEXT-105

No se utilizará un giant mixed registry como modelo principal.

## DB-SQLEXT-106

Component resolution conservará evidence.

## DB-SQLEXT-107

Registry dependency graph se resolverá durante bootstrap.

## DB-SQLEXT-108

Dependency graph no se reconstruirá por query.

## DB-SQLEXT-109

Runtime lookup será bounded.

## DB-SQLEXT-110

Frozen registry podrá compartirse en persistent workers.

## DB-SQLEXT-111

Operation-specific extension state no podrá compartirse entre requests.

## DB-SQLEXT-112

Tenant state no podrá almacenarse en shared extension objects.

## DB-SQLEXT-113

Connection-specific capabilities no podrán filtrarse entre connections.

## DB-SQLEXT-114

UsedExtensionTracker será operation-scoped.

## DB-SQLEXT-115

Extension diagnostics serán operation-scoped.

## DB-SQLEXT-116

Extension budget counters serán operation-scoped.

## DB-SQLEXT-117

Extension-generated parameters usarán central Placeholder Planner.

## DB-SQLEXT-118

Extensions no tendrán private placeholder numbering.

## DB-SQLEXT-119

Extension-generated aliases usarán operation-scoped allocator.

## DB-SQLEXT-120

Extensions no tendrán global alias counters.

## DB-SQLEXT-121

Extension output columns deberán entrar al ResultContract.

## DB-SQLEXT-122

ResultContract extensions no hidratarán entities.

## DB-SQLEXT-123

Extension dependencies serán estructuradas.

## DB-SQLEXT-124

Dependency tracking no se inferirá parseando rendered SQL.

## DB-SQLEXT-125

Cached extension references serán version-aware.

## DB-SQLEXT-126

Missing cached extension invalidará el artifact.

## DB-SQLEXT-127

Incompatible extension version invalidará el artifact cuando corresponda.

## DB-SQLEXT-128

Public Extension SPI estará separado de internal compiler implementation.

## DB-SQLEXT-129

Internal compiler classes no serán automáticamente public SPI.

## DB-SQLEXT-130

Official extensions deberán respetar invariants.

## DB-SQLEXT-131

Privileged extension no significará unrestricted extension.

## DB-SQLEXT-132

Extension contracts no se considerarán malicious-code sandbox.

## DB-SQLEXT-133

Extension conformance tests serán reutilizables.

## DB-SQLEXT-134

Extension determinism será verificable.

## DB-SQLEXT-135

Extension concurrency isolation será verificable.

## DB-SQLEXT-136

Security preservation será verificable.

## DB-SQLEXT-137

Binding safety será verificable.

## DB-SQLEXT-138

Source-map provenance será verificable.

## DB-SQLEXT-139

Fingerprint stability será verificable.

## DB-SQLEXT-140

Unsupported targets fallarán explícitamente.

## DB-SQLEXT-141

Failure injection no publicará artifacts parciales.

## DB-SQLEXT-142

Extension runtime overhead deberá ser medible.

## DB-SQLEXT-143

Bootstrap overhead será distinto de per-query overhead.

## DB-SQLEXT-144

Extension registry reload será atomic.

## DB-SQLEXT-145

Registry changes tendrán nueva generation/fingerprint.

## DB-SQLEXT-146

Compiled cache podrá detectar registry/extension incompatibilities.

## DB-SQLEXT-147

Extension enable/disable será explícito.

## DB-SQLEXT-148

Extension diagnostics podrán exponerse mediante CLI.

## DB-SQLEXT-149

Compiler Explain podrá mostrar extensiones participantes.

## DB-SQLEXT-150

Telemetry integration será mediante contracts.

## DB-SQLEXT-151

Compiler extension no dependerá directamente de OpenTelemetry.

## DB-SQLEXT-152

Compiler extension no podrá ejecutar PreparedStatement.

## DB-SQLEXT-153

Compiler extension no podrá bindear runtime values.

## DB-SQLEXT-154

Compiler extension no podrá fetch results.

## DB-SQLEXT-155

Compiler extension no podrá iniciar transactions.

## DB-SQLEXT-156

Compiler extension no podrá seleccionar replicas.

## DB-SQLEXT-157

Compiler extension no podrá cambiar execution multiplicity.

## DB-SQLEXT-158

Compiler extension no podrá cambiar observable cardinality.

## DB-SQLEXT-159

Compiler extension no podrá degradar semantic guarantees.

## DB-SQLEXT-160

Toda extensión deberá preservar la semántica del artifact que transforma.

---

# 307. Invariante maestro

Para cualquier extension pass válido:

```text
Semantics(
    ExtensionTransform(Artifact)
)
=
Semantics(
    Artifact
)
```

cuando el pass sea exclusivamente representacional.

---

# 308. Invariante de seguridad

```text
SecurityRequirements(
    Output
)
>=
SecurityRequirements(
    Input
)
```

en el sentido de que ninguna restricción obligatoria puede desaparecer durante compilation.

---

# 309. Invariante de ejecución

```text
Compiler Extension
cannot introduce
hidden execution steps
```

Por tanto:

```text
one operation
→ extension
→ hidden SELECT + UPDATE
```

queda prohibido.

---

# 310. Invariante de parámetros

```text
All Runtime Values
        ↓
Central Parameter Model
        ↓
Placeholder Plan
        ↓
Binding Layout
```

incluidos los generados por extensiones.

---

# 311. Invariante de identidad

```text
Semantic Identity
≠
Rendered SQL Spelling
```

aplicable a:

```text
functions
operators
types
relations
predicates
extensions
```

---

# 312. Invariante de lifecycle

```text
Mutable During Bootstrap
        ↓
Validated
        ↓
Resolved
        ↓
Frozen
        ↓
Immutable During Compilation
```

---

# 313. Invariante de persistent runtime

```text
Shared
=
Frozen + Immutable + Request-Independent
```

y:

```text
Operation Local
=
Bindings
+
Allocators
+
Diagnostics
+
Budgets
+
Usage Tracking
+
Temporary Compilation State
```

---

# 314. Fórmula de resolución

Para un component candidate `C`:

```text
Valid(C)
=
IdentityValid(C)
∧
ApiCompatible(C)
∧
PlatformCompatible(C)
∧
DependenciesSatisfied(C)
∧
CapabilitiesSatisfied(C)
∧
NoUnresolvedConflict(C)
∧
SecurityValid(C)
```

Sólo después:

```text
Resolve(Candidates)
```

---

# 315. Fórmula de activación

```text
ActiveExtension
=
Installed
∧
Enabled
∧
Compatible
∧
DependenciesResolved
∧
ConflictsResolved
∧
OrderingResolved
∧
CapabilitiesResolved
∧
Validated
```

---

# 316. Fórmula de fingerprint

```text
UsedExtensionFingerprint
=
Hash(
    ExtensionIdentity
    +
    ExtensionVersion
    +
    ExtensionApiVersion
    +
    RelevantConfiguration
    +
    RelevantCapabilities
    +
    RelevantIntegrationComponents
    +
    RelevantOrdering
)
```

---

# 317. Fórmula arquitectónica

```text
SQL Compiler Extension System
=
Typed SPI
+
Descriptor Model
+
Discovery
+
Compatibility Resolution
+
Dependency Graph
+
Conflict Detection
+
Deterministic Ordering
+
Capability Composition
+
Typed Registries
+
Controlled Compilation Passes
+
Security Validation
+
Budget Governance
+
Source Provenance
+
Dependency Tracking
+
Fingerprinting
+
Persistent Runtime Isolation
```

---

# 318. Arquitectura completa

```text
                       Extension Packages
                              │
            ┌─────────────────┼──────────────────┐
            │                 │                  │
            ▼                 ▼                  ▼
        Official          Third Party       Application
       Extensions         Extensions        Extensions
            │                 │                  │
            └─────────────────┼──────────────────┘
                              │
                              ▼
                    Extension Discovery
                              │
                              ▼
                         Descriptors
                              │
                              ▼
                 Compatibility Validation
                              │
                              ▼
                     Dependency Graph
                              │
                              ▼
                     Conflict Analysis
                              │
                              ▼
                    Ordering Resolution
                              │
                              ▼
                   Capability Resolution
                              │
                              ▼
                      Registry Compiler
                              │
                              ▼
                 Frozen Extension Registry
                              │
                              ▼
                   SQL Compiler Pipeline
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
          Statements      Expressions       Functions
              │               │                │
              ├───────────────┼────────────────┤
              │               │                │
              ▼               ▼                ▼
          Predicates       Operators          Types
              │               │                │
              ├───────────────┼────────────────┤
                              │
                              ▼
                       SQL Emission Tree
                              │
                              ▼
                    Platform Adaptation
                              │
                              ▼
                 Identifier/Parameter Plan
                              │
                              ▼
                         Rendering
                              │
                              ▼
                  CompiledDatabaseCommand
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
        BindingLayout    ResultContract    SourceMap
              │               │                │
              └───────────────┼────────────────┘
                              │
                              ▼
                     Dependency Set
                              │
                              ▼
                         Fingerprint
```

---

# 319. Architectural principle

El sistema deberá favorecer:

```text
Extension adds capability
```

sobre:

```text
Extension overrides framework behavior
```

---

# 320. Composition principle

Preferir:

```text
small specialized extension components
```

sobre:

```text
one extension object controlling the entire compiler
```

---

# 321. Safety principle

```text
Extensibility stops
where semantic integrity begins.
```

---

# 322. Determinism principle

```text
Extension flexibility
must not produce
compiler unpredictability.
```

---

# 323. Portability principle

Una extensión portable deberá modelar:

```text
One Semantic Capability
        ↓
Multiple Explicit Platform Representations
```

No:

```text
One Platform SQL String
        ↓
hope every database accepts it
```

---

# 324. Failure principle

Ante una extensión incompatible:

```text
Fail During Bootstrap
```

cuando sea posible.

Ante una feature no representable para una query:

```text
Fail During Compilation
```

Nunca:

```text
Fail unpredictably after SQL reaches database
```

cuando el problema pueda detectarse antes.

---

# 325. No fallback principle

```text
Unknown Extension
Unsupported Extension
Missing Capability
Invalid Extension
```

nunca implicará:

```text
Raw SQL Fallback
```

---

# 326. VoltStack package ecosystem

Este sistema permitirá que paquetes oficiales futuros puedan extender Database de forma desacoplada.

Ejemplos conceptuales:

```text
VoltStack/Quantum/Database/PostgreSQL/Vector
VoltStack/Quantum/Database/PostgreSQL/FullText
VoltStack/Quantum/Database/SQLite/FTS
VoltStack/Quantum/Database/MySQL/Spatial
VoltStack/Quantum/Database/Json
VoltStack/Quantum/Database/Geospatial
```

sin aumentar continuamente el núcleo del SQL Compiler.

---

# 327. Relationship with Database Extension System

Este documento define específicamente:

```text
SQL Compiler Extension SPI
```

El sistema general posterior:

```text
294_DATABASE_EXTENSION_ARCHITECTURE.md
295_DATABASE_PLUGIN_SYSTEM.md
```

coordinará extensiones de todo Database.

Por tanto:

```text
Database Plugin
        ↓
may contain
        ↓
SQL Compiler Extension
```

pero:

```text
SQL Compiler Extension
≠
Entire Database Plugin
```

---

# 328. Relationship with custom compilers

El futuro:

```text
298_DATABASE_CUSTOM_COMPILER_SYSTEM.md
```

definirá cómo sustituir o incorporar compiladores completos.

Este documento favorece extensiones parciales y composables.

```text
Compiler Extension
≠
Custom Compiler
```

---

# 329. Cuándo usar una extensión

Usar `SQL Compiler Extension` cuando se necesite:

```text
new representation
new target syntax
new function mapping
new operator mapping
new type mapping
new emission construct
new platform compilation capability
```

---

# 330. Cuándo no usar una extensión

No usarla como única capa cuando se necesite:

```text
new query semantics
new persistence semantics
new transaction semantics
new distributed execution
new ORM behavior
new security model
new physical planning algorithm
```

Estas características deberán integrarse en las capas correspondientes.

---

# 331. Arquitectura final del SPI

```text
                   SQL Compiler Extension SPI
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
     Metadata              Components             Passes
        │                     │                     │
        ▼                     ▼                     ▼
   Descriptor            Typed Contracts      Phase Contracts
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
                         Validation
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
        Dependencies       Conflicts        Security
              │               │               │
              └───────────────┼───────────────┘
                              │
                              ▼
                           Ordering
                              │
                              ▼
                         Capabilities
                              │
                              ▼
                     Compiled Registry
                              │
                              ▼
                           Freeze
                              │
                              ▼
                    Compilation Runtime
                              │
              ┌───────────────┼────────────────┐
              │               │                │
              ▼               ▼                ▼
          Typed Lookup      Budget         Usage Tracker
              │               │                │
              └───────────────┼────────────────┘
                              │
                              ▼
                         SQL Emission
                              │
                              ▼
                         Validation
                              │
                              ▼
                  CompiledDatabaseCommand
```

---

# 332. Resultado final

VoltStack obtiene un SQL Compiler capaz de crecer mediante un ecosistema de paquetes sin sacrificar:

```text
semantic correctness
security
portability
determinism
cacheability
diagnostics
performance
persistent runtime safety
```

La extensibilidad queda gobernada por:

```text
Contracts before Hooks
Descriptors before Runtime Discovery
Capabilities before Vendor Conditionals
Dependency Graphs before Load Order
Validation before Activation
Typed Emission before SQL Strings
Security before Convenience
Determinism before Flexibility
```

---

# 333. Regla maestra

```text
An extension may extend
what the compiler can represent.

It may not silently redefine
what the query means.
```

---

# 334. Fórmula final

```text
Safe SQL Compiler Extensibility
=
Explicit Semantics
+
Typed Extension SPI
+
Frozen Registries
+
Deterministic Resolution
+
Capability Awareness
+
Security Preservation
+
Bounded Compilation
+
Dependency Tracking
+
Stable Fingerprinting
```

---

# 335. Estado del bloque SQL Compiler

Con este documento quedan definidos:

```text
66_DATABASE_SQL_COMPILER_ARCHITECTURE.md
67_DATABASE_SQL_COMPILER_PIPELINE.md
68_DATABASE_SQL_GENERATION_SYSTEM.md
69_DATABASE_MYSQL_SQL_COMPILER.md
70_DATABASE_MARIADB_SQL_COMPILER.md
71_DATABASE_POSTGRESQL_SQL_COMPILER.md
72_DATABASE_SQLITE_SQL_COMPILER.md
73_DATABASE_SQL_COMPILER_EXTENSION_SYSTEM.md
```

La arquitectura del bloque queda ahora preparada para pasar de:

```text
SQL representation
```

a:

```text
statement preparation
+
runtime binding layout
```

sin mezclar compilation con execution.

---

# 336. Siguiente documento

```text
74_DATABASE_PREPARED_STATEMENT_COMPILATION_SYSTEM.md
```

El siguiente documento deberá formalizar:

```text
CompiledDatabaseCommand
        ↓
Prepared Statement Compilation
        ↓
PreparedStatementBlueprint
        ↓
Runtime Statement Preparation
        ↓
Parameter Binding
        ↓
Execution
```

incluyendo:

```text
prepared statement architecture
statement blueprint
parameter slot model
placeholder-to-binding mapping
driver preparation contracts
binding conversion plans
statement identity
statement cacheability
statement reuse
connection affinity
transaction affinity
schema/session dependencies
driver-specific preparation profiles
prepared statement lifecycle boundaries
persistent worker safety
statement cache interaction
error translation
security invariants
```

manteniendo:

```text
Prepared Statement Compilation
≠
Live Statement Preparation
≠
Parameter Binding
≠
Statement Execution
```