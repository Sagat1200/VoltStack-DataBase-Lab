# 102_DATABASE_MIGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Migration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 102 — Database Migration System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration System` define el modelo central, contratos fundamentales, objetos de valor, estados, invariantes y API conceptual mediante los cuales VoltStack representa una **migración individual** como una unidad versionada y reproducible de evolución de base de datos.

Este documento profundiza el núcleo definido en:

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
```

y establece las piezas que utilizarán posteriormente:

```text
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

La pregunta principal es:

> **¿Qué es exactamente una migración dentro de VoltStack y qué información debe contener antes de que Discovery, Repository, Planning o Execution intervengan?**

---

# 2. Principio central

> **Una Migration Definition es una descripción inmutable, versionada y determinista de una evolución solicitada; no contiene el estado runtime de su ejecución ni ejecuta por sí misma los cambios.**

Por tanto:

```text
Migration Definition
≠
Migration Execution

Migration Definition
≠
Migration Repository Record

Migration Definition
≠
Migration Plan

Migration Definition
≠
Schema AST únicamente

Migration Definition
≠
SQL Script
```

---

# 3. Separación fundamental

VoltStack distinguirá:

```text
Migration
├── Definition
├── Descriptor
├── Identity
├── Metadata
├── Dependencies
├── Operation Factory
├── Rollback Definition
├── Policy Requirements
└── Checksum
```

de:

```text
Migration Runtime
├── Discovery State
├── Repository State
├── Planning State
├── Execution State
├── Lock State
├── Transaction State
├── Failure State
└── Telemetry State
```

---

# 4. Modelo conceptual

```text
Migration
│
├── MigrationId
├── MigrationVersion?
├── MigrationNamespace
├── MigrationMetadata
├── MigrationDependencySet
├── MigrationTargetDefinition
├── MigrationOperationFactory
├── MigrationRollbackDefinition
├── MigrationPolicyRequirements
└── MigrationChecksumDescriptor
```

La definición deberá ser:

```text
immutable
deterministic
serializable conceptually
source-traceable
platform-neutral where possible
```

---

# 5. Contrato principal

Se propone:

```php
interface Migration
{
    public function id(): MigrationId;

    public function define(
        MigrationDefinitionContext $context
    ): MigrationDefinition;
}
```

Otra opción igualmente válida:

```php
abstract class Migration
{
    abstract public function id(): MigrationId;

    abstract protected function up(MigrationContext $migration): void;

    protected function down(MigrationContext $migration): void
    {
        throw new IrreversibleMigrationException();
    }
}
```

La API final puede parecer familiar para desarrolladores Laravel, pero internamente:

```text
up()
↓
collect typed operations
↓
seal definition
```

nunca:

```text
up()
↓
execute connection
```

---

# 6. MigrationDefinition

Se propone:

```php
final readonly class MigrationDefinition
{
    public function __construct(
        public MigrationId $id,
        public MigrationNamespace $namespace,
        public MigrationMetadata $metadata,
        public MigrationDependencySet $dependencies,
        public MigrationOperationSet $operations,
        public MigrationRollbackDefinition $rollback,
        public MigrationPolicyRequirementSet $requirements,
        public MigrationDefinitionMetadata $definitionMetadata,
    ) {}
}
```

La clase será la representación canónica de una migración ya construida.

---

# 7. Migration object vs definition

Debe distinguirse:

```text
Migration class/file
=
developer-facing source
```

```text
MigrationDefinition
=
canonical immutable representation
```

Esto permite que distintas fuentes produzcan la misma abstracción:

```text
PHP class
package migration
generated migration
serialized migration definition
testing migration
```

---

# 8. Lifecycle de construcción

```text
Migration Source
     ↓
Instantiate Migration
     ↓
Create Definition Context
     ↓
Invoke Definition
     ↓
Collect Operations
     ↓
Validate
     ↓
Seal
     ↓
MigrationDefinition
```

---

# 9. MigrationDefinitionContext

Se propone:

```php
final readonly class MigrationDefinitionContext
{
    public function __construct(
        public MigrationDefinitionProfile $profile,
        public FrozenMigrationExtensionRegistry $extensions,
        public MigrationDefinitionBudget $budget,
    ) {}
}
```

No contendrá:

```text
live connection
current transaction
repository mutable state
current tenant global
request object
```

---

# 10. MigrationContext

La API para construir operaciones:

```php
interface MigrationContext
{
    public function schema(): MigrationSchemaOperations;

    public function data(): MigrationDataOperations;

    public function validate(): MigrationValidationOperations;

    public function barrier(): MigrationBarrierOperations;
}
```

---

# 11. MigrationContext ≠ Connection

Regla crítica:

```text
MigrationContext
=
Operation Construction Context
```

no:

```text
MigrationContext
=
Database Connection Wrapper
```

---

# 12. MigrationSchemaOperations

Podrá ofrecer:

```php
interface MigrationSchemaOperations
{
    public function create(
        string|TableIdentifier $table,
        Closure $definition
    ): void;

    public function table(
        string|TableIdentifier $table,
        Closure $definition
    ): void;

    public function rename(
        string|TableIdentifier $from,
        string|TableIdentifier $to
    ): void;

    public function drop(
        string|TableIdentifier $table
    ): void;
}
```

Internamente genera:

```text
SchemaMigrationOperation
```

---

# 13. MigrationDataOperations

Podrá ofrecer:

```php
interface MigrationDataOperations
{
    public function insert(...): void;

    public function update(...): void;

    public function delete(...): void;

    public function backfill(BackfillDefinition $backfill): void;
}
```

Pero cada operación deberá bajar a estructuras tipadas del Query System.

---

# 14. MigrationValidationOperations

Permitirá pre/post conditions explícitas:

```php
$migration->validate()->noNulls('users', 'email');

$migration->validate()->tableExists('users');

$migration->validate()->columnContainsOnly(
    table: 'orders',
    column: 'status',
    values: ['new', 'paid', 'cancelled'],
);
```

Estas operaciones se ejecutarán después, no durante definition building.

---

# 15. Barrier operations

API conceptual:

```php
$migration->barrier()->checkpoint('users_email_backfilled');
```

o internamente:

```text
MigrationBarrierOperation
```

para expresar dependencias temporales/operativas.

---

# 16. MigrationOperation

Contrato central:

```php
interface MigrationOperation
{
    public function id(): MigrationOperationId;

    public function kind(): MigrationOperationKind;

    public function metadata(): MigrationOperationMetadata;
}
```

---

# 17. MigrationOperationKind

Se propone:

```php
enum MigrationOperationKind
{
    case SCHEMA;
    case DATA;
    case VALIDATION;
    case BARRIER;
    case RAW;
    case EXTENSION;
}
```

---

# 18. MigrationOperationId

```php
final readonly class MigrationOperationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Debe ser único dentro de una `MigrationDefinition`.

---

# 19. Operation ID determinista

Idealmente:

```text
MigrationId
+
OperationOrdinal
+
OperationKind
```

o equivalente estable.

Ejemplo:

```text
2026_09_06_create_users#op-001
```

No utilizar UUID aleatorio si se desea checksum reproducible.

---

# 20. Operation ordering

La definición preservará orden declarado:

```text
OP1
OP2
OP3
```

Pero:

```text
Declared Operation Order
≠
Final Execution Order
```

si el planner puede legalmente insertar:

```text
validation
transaction boundary
compatibility strategy
barrier
```

Sin embargo, no podrá violar dependencias semánticas declaradas.

---

# 21. MigrationOperationSet

```php
final readonly class MigrationOperationSet
{
    /** ordered immutable operation collection */
}
```

Propiedades:

```text
ordered
immutable
duplicate-ID-safe
deterministic
```

---

# 22. Operation dependencies

Además del orden secuencial, una operación podrá declarar:

```text
dependsOn(operationId)
```

para permitir grafos explícitos.

---

# 23. Sequential default

Por ergonomía:

```text
OP1
OP2
OP3
```

podrá implicar:

```text
OP2 depends on OP1
OP3 depends on OP2
```

salvo que una API avanzada declare independencia.

---

# 24. Parallelizable operations

Arquitectónicamente podrán existir:

```text
OP2 ─┐
     ├→ OP4
OP3 ─┘
```

aunque la ejecución paralela real dependa de:

```text
capabilities
safety
locking
runtime
```

---

# 25. MigrationId

Requisitos:

```text
stable
unique
deterministic
human-readable where possible
safe for repository persistence
```

---

# 26. MigrationNamespace

Para evitar colisiones entre:

```text
application
framework
packages
plugins
```

se propone:

```php
final readonly class MigrationNamespace
{
    public function __construct(
        public string $value,
    ) {}
}
```

Ejemplos:

```text
app
voltstack.core
package.acme.billing
plugin.vendor.audit
```

---

# 27. Full identity

Conceptualmente:

```text
MigrationIdentity
=
Namespace
+
MigrationId
```

---

# 28. Filename naming convention

Se podrá recomendar:

```text
YYYY_MM_DD_HHMMSS_description.php
```

pero:

```text
filename convention
≠
identity model
```

---

# 29. MigrationVersion

No será obligatoria inicialmente.

Si existe:

```php
final readonly class MigrationVersion
{
    public function __construct(
        public string $value,
    ) {}
}
```

servirá para dominios que prefieran:

```text
1.0.0
1.1.0
2.0.0
```

sobre timestamps.

---

# 30. Migration version ≠ framework version

Debe mantenerse:

```text
MigrationVersion
≠
VoltStackVersion
≠
PackageVersion
```

aunque puedan relacionarse mediante metadata.

---

# 31. MigrationMetadata

Se propone:

```php
final readonly class MigrationMetadata
{
    public function __construct(
        public ?string $description,
        public MigrationTagSet $tags,
        public MigrationSourceMetadata $source,
        public MigrationTrustLevel $trust,
        public ExtensionMetadataSet $extensions,
    ) {}
}
```

---

# 32. Metadata no estructural

Campos como:

```text
description
author
source file
tags
```

no deberán modificar semántica de la migración salvo que el profile de checksum explícitamente los incluya.

---

# 33. Source metadata

```php
final readonly class MigrationSourceMetadata
{
    public function __construct(
        public ?string $file,
        public ?int $line,
        public MigrationSourceKind $sourceKind,
    ) {}
}
```

---

# 34. SourceKind

```text
APPLICATION
FRAMEWORK
PACKAGE
PLUGIN
GENERATED
TESTING
EXTERNAL
```

---

# 35. MigrationTrustLevel

```text
FRAMEWORK
APPLICATION
TRUSTED_PACKAGE
UNTRUSTED_EXTENSION
EXTERNAL
UNKNOWN
```

Trust nunca omitirá validación.

---

# 36. Migration tags

Ejemplos:

```text
users
billing
security
analytics
cleanup
release-2.0
```

Podrán servir para:

```text
filtering
tooling
target selection
diagnostics
```

---

# 37. Tags ≠ dependencies

No se deberá usar:

```text
tag: users
```

como dependency implícita.

---

# 38. MigrationDependencySet

```php
final readonly class MigrationDependencySet
{
    /** @var MigrationDependency[] */
}
```

---

# 39. Dependency kinds

Se propone:

```php
enum MigrationDependencyKind
{
    case REQUIRED;
    case OPTIONAL;
    case AFTER;
    case BEFORE;
}
```

Sin embargo, el core mínimo puede comenzar únicamente con:

```text
REQUIRED
```

y añadir otros tipos solo si aportan semántica clara.

---

# 40. Required dependency

```text
M2 depends on M1
```

significa:

```text
M1 must be applied before M2
```

---

# 41. BEFORE/AFTER caution

`BEFORE` y `AFTER` pueden introducir grafos difíciles de mantener.

Preferencia:

```text
explicit required dependencies
+
stable deterministic order
```

como baseline.

---

# 42. External dependencies

Una migración de package puede depender de:

```text
voltstack.core:migration_X
```

mediante identidad calificada.

---

# 43. Dependency resolution deferred

MigrationDefinition solo declara dependencias.

La resolución ocurre en:

```text
103 Discovery
105 Planner
```

según responsabilidades.

---

# 44. MigrationTargetDefinition

Una migración podrá restringir dónde puede aplicarse.

```php
final readonly class MigrationTargetDefinition
{
    public function __construct(
        public MigrationConnectionSelector $connection,
        public ?SchemaNamespaceReference $namespace,
        public MigrationTargetScope $scope,
    ) {}
}
```

---

# 45. Target scope

Ejemplos:

```text
DEFAULT_DATABASE
NAMED_CONNECTION
FRAMEWORK_INTERNAL
APPLICATION_DATABASE
EXTENSION_DEFINED
```

Multitenancy se añadirá por integración externa.

---

# 46. Target resolution

Definition expresa:

```text
logical target
```

Execution resuelve:

```text
physical connection
```

---

# 47. Logical target ≠ physical connection

Esto permite:

```text
logical: billing
```

resolverse a:

```text
MySQL production billing cluster
```

sin acoplar definición al DSN.

---

# 48. Platform requirements

Una migración puede declarar:

```text
requires capability X
```

pero se recomienda derivarlas de operaciones siempre que sea posible.

---

# 49. Explicit migration requirements

Solo deberían añadirse cuando la migración como conjunto requiere algo adicional.

Ejemplo:

```text
requires atomic DDL
requires advisory lock support
requires application dual-write phase
```

---

# 50. MigrationPolicyRequirementSet

Se propone:

```text
transaction requirement
lock requirement
safety minimum
raw SQL policy
execution mode restrictions
environment restrictions?
```

Debe mantenerse compacto para no mezclar policy runtime con definition.

---

# 51. Policy requirement ≠ runtime policy

Ejemplo:

```text
Migration requires exclusive lock
```

versus:

```text
Runtime policy refuses exclusive lock migrations in production
```

Son conceptos distintos.

---

# 52. MigrationChecksum

Debe poder calcularse sobre la definición canónica.

```php
final readonly class MigrationChecksum
{
    public function __construct(
        public string $algorithm,
        public int $version,
        public string $value,
    ) {}
}
```

---

# 53. Checksum formula

Conceptualmente:

```text
MigrationChecksum
=
H(
    CanonicalMigrationDefinition,
    ChecksumProfile,
    ChecksumVersion
)
```

---

# 54. Canonical definition

Debe excluir, según profile:

```text
absolute filesystem path
runtime object IDs
memory addresses
timestamps of loading
```

---

# 55. Semantic checksum

Se recomienda que el checksum principal represente:

```text
identity
dependencies
ordered operations
rollback definition
semantic policy requirements
```

---

# 56. Source checksum

También podrá existir:

```text
MigrationSourceChecksum
```

para detectar cambios de archivo, pero será distinto del checksum semántico.

---

# 57. Source checksum ≠ semantic checksum

Cambio de:

```text
whitespace
comments
formatting
```

puede alterar source checksum sin alterar semantic checksum.

---

# 58. Structural checksum

Para migraciones schema-only podría derivarse de:

```text
Schema AST fingerprints
```

más orchestration metadata.

---

# 59. Dynamic values prohibited

Una migración no deberá incorporar en su definición canónica:

```text
current timestamp
random number
host name
process ID
request ID
```

salvo que se modelen como runtime operation semantics explícitas.

---

# 60. Example anti-determinism

Incorrecto:

```php
$table->string('backup_' . date('Ymd'));
```

durante construcción.

---

# 61. Runtime expressions

Si se necesita:

```text
current database time
```

deberá representarse como:

```text
SchemaExpression::currentTimestamp()
```

o Query expression equivalente.

No evaluarse en PHP durante definition.

---

# 62. MigrationState

Debe distinguirse entre:

```text
definition state
repository state
execution state
```

No crear un único enum ambiguo `MigrationStatus`.

---

# 63. MigrationDefinitionState

Internamente durante construcción:

```text
CREATED
BUILDING
VALIDATING
SEALED
```

---

# 64. Repository migration state

Podrá ser:

```text
NOT_APPLIED
APPLIED
DRIFTED
MISSING_DEFINITION
UNKNOWN
```

---

# 65. MigrationExecutionState

```text
CREATED
PLANNED
WAITING_FOR_LOCK
RUNNING
SUCCEEDED
FAILED
PARTIALLY_APPLIED
CANCELLED
UNKNOWN_OUTCOME
```

---

# 66. Applied ≠ succeeded attempt

Repository state:

```text
APPLIED
```

significa que la migración forma parte del historial aceptado.

Execution state:

```text
SUCCEEDED
```

describe un intento.

---

# 67. MigrationExecutionId

Cada intento:

```php
final readonly class MigrationExecutionId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 68. Multiple attempts

Puede existir:

```text
Execution A → failed
Execution B → succeeded
```

para la misma `MigrationId`.

---

# 69. Execution state not in definition

Nunca:

```php
$migration->status = 'running';
```

sobre la definición compartida.

---

# 70. MigrationDescriptor

Discovery no necesita construir inmediatamente todo el contenido.

Se propone:

```php
final readonly class MigrationDescriptor
{
    public function __construct(
        public MigrationIdentity $identity,
        public MigrationSourceReference $source,
        public MigrationDescriptorMetadata $metadata,
    ) {}
}
```

---

# 71. Descriptor ≠ Definition

```text
Descriptor
=
lightweight discoverable metadata
```

```text
Definition
=
fully built canonical migration
```

Esto permite discovery eficiente.

---

# 72. Lazy definition loading

Discovery podrá catalogar miles de migraciones sin ejecutar su definición inmediatamente.

Luego:

```text
Descriptor
    ↓
MigrationDefinitionLoader
    ↓
MigrationDefinition
```

---

# 73. Lazy loading safety

Cargar una definición deberá seguir siendo:

```text
side-effect free with respect to DB
```

---

# 74. MigrationDefinitionLoader

```php
interface MigrationDefinitionLoader
{
    public function load(
        MigrationDescriptor $descriptor
    ): MigrationDefinition;
}
```

---

# 75. Definition loader and PHP files

Si la fuente es PHP:

```text
file
↓
isolated loader
↓
Migration object
↓
definition collection
```

Se deberán controlar:

```text
return type
duplicate identity
exceptions
unexpected side effects where possible
```

---

# 76. Migration Reversibility

Se propone:

```php
enum MigrationReversibility
{
    case REVERSIBLE;
    case PARTIALLY_REVERSIBLE;
    case IRREVERSIBLE;
    case UNKNOWN;
}
```

---

# 77. Reversibility source

Puede derivarse de:

```text
explicit rollback definition
+
operation analysis
```

pero nunca asumir que `down()` existente significa rollback seguro.

---

# 78. MigrationRollbackDefinition

```php
final readonly class MigrationRollbackDefinition
{
    public function __construct(
        public MigrationReversibility $reversibility,
        public MigrationOperationSet $operations,
        public ?string $reason,
    ) {}
}
```

---

# 79. Forward and rollback asymmetry

Una migración puede ser:

```text
Forward:
1 operation

Rollback:
5 operations
```

No se exige simetría.

---

# 80. Irreversible definition

Ejemplo:

```php
MigrationRollbackDefinition::irreversible(
    'Dropped customer history cannot be reconstructed.'
);
```

---

# 81. Unknown reversibility

Usar:

```text
UNKNOWN
```

cuando una extensión/raw operation no puede analizarse.

No clasificar automáticamente como reversible.

---

# 82. Rollback operation IDs

Rollback tendrá su propio espacio de operaciones.

Ejemplo:

```text
forward: f-op-001
rollback: r-op-001
```

No reutilizar IDs ambiguamente.

---

# 83. MigrationResult

Debe distinguirse de execution state.

```php
final readonly class MigrationResult
{
    public function __construct(
        public MigrationIdentity $migration,
        public MigrationExecutionState $state,
        public MigrationOutcomeCertainty $certainty,
        public MigrationExecutionMetadata $metadata,
        public ?MigrationFailure $failure,
    ) {}
}
```

---

# 84. Outcome certainty

Se propone:

```text
CERTAIN_SUCCESS
CERTAIN_FAILURE
PARTIAL
UNKNOWN
```

alineado con Execution Engine.

---

# 85. Failure model

```php
final readonly class MigrationFailure
{
    public function __construct(
        public MigrationFailureKind $kind,
        public ?MigrationOperationId $operation,
        public MigrationOutcomeCertainty $certainty,
        public Throwable $cause,
        public MigrationFailureMetadata $metadata,
    ) {}
}
```

---

# 86. Failure kinds

```text
DEFINITION
DEPENDENCY
COMPATIBILITY
SAFETY
LOCK
TRANSACTION
SCHEMA_EXECUTION
DATA_EXECUTION
VALIDATION
REPOSITORY
CANCELLATION
TIMEOUT
EXTENSION
UNKNOWN
```

---

# 87. Definition errors before execution

Problemas como:

```text
duplicate operation ID
invalid dependency
invalid schema operation
invalid rollback definition
```

deberán impedir llegar a planning.

---

# 88. MigrationValidation

Se propone:

```text
Definition Validation
├── Identity Validation
├── Dependency Syntax Validation
├── Operation Validation
├── Rollback Validation
├── Policy Requirement Validation
├── Extension Validation
└── Determinism Validation
```

---

# 89. Determinism validation

No siempre será posible detectar código dinámico arbitrario.

Pero el framework puede reducirlo mediante:

```text
restricted construction context
canonical fingerprints
source checksums
diagnostics
```

---

# 90. Definition budget

```php
final readonly class MigrationDefinitionBudget
{
    public function __construct(
        public int $maxOperations,
        public int $maxDependencies,
        public int $maxTags,
        public int $maxExtensionEntries,
        public int $maxSerializedBytes,
        public int $maxOperationDepth,
    ) {}
}
```

---

# 91. Budget exhaustion

Debe fallar con:

```text
MigrationDefinitionBudgetExceededException
```

Nunca truncar operaciones.

---

# 92. Operation metadata

Cada operación podrá incluir:

```text
source location
label
description
risk hint
extension metadata
```

pero risk hint no sustituirá Safety Analyzer.

---

# 93. Developer labels

Ejemplo:

```php
$migration->label(
    'backfill-normalized-email',
    fn () => ...
);
```

puede facilitar:

```text
telemetry
diagnostics
resume tooling
```

---

# 94. Label ≠ OperationId

Puede ser human-readable y no necesariamente único globalmente.

---

# 95. Migration capabilities

La definición podrá exponer requirements derivados:

```text
MigrationDefinition
    ↓
Capability Requirement Projection
```

pero preferiblemente no almacenar duplicados.

---

# 96. Derived capability set

```text
MigrationCapabilityRequirements
=
Union(
    OperationCapabilityRequirements
)
+
MigrationLevelRequirements
```

---

# 97. No stale duplicated requirements

Si cambia una operación, los requirements deben recalcularse.

No mantener dos fuentes manuales.

---

# 98. Migration impact projection

Igualmente:

```text
MigrationImpact
=
Aggregate(OperationImpacts)
```

solo como proyección.

La clasificación final la hará Safety System.

---

# 99. Migration type

Puede existir:

```php
enum MigrationType
{
    case SCHEMA_ONLY;
    case DATA_ONLY;
    case MIXED;
    case EXTENSION;
}
```

derivado automáticamente de las operaciones.

---

# 100. Type should be derived

No confiar en:

```php
public string $type = 'schema';
```

si las operaciones contradicen el valor.

---

# 101. Mixed migration example

```text
OP1 Schema
OP2 Data
OP3 Validation
OP4 Schema
```

produce:

```text
MigrationType::MIXED
```

---

# 102. Migration phases

Aunque `110` profundizará zero-downtime, el core puede permitir:

```text
EXPAND
DATA
VALIDATE
CONTRACT
CUSTOM
```

como `MigrationOperationPhase`.

---

# 103. Phase metadata

No debe alterar automáticamente el orden.

Sirve para:

```text
planning
tooling
zero-downtime analysis
```

---

# 104. Preconditions

Una migración podrá tener preconditions globales además de operaciones de validación.

Ejemplo:

```text
requires extension pgcrypto
requires schema version X
requires feature flag state?
```

Debe evitarse acoplar al deployment system salvo integración explícita.

---

# 105. MigrationPreconditionDefinition

```php
interface MigrationPreconditionDefinition
{
    public function id(): MigrationPreconditionId;
}
```

---

# 106. Postconditions

Después de execution:

```text
table exists
constraint validated
schema fingerprint matches expected
```

podrán comprobarse cuando la migration lo declare.

---

# 107. Postcondition ≠ repository write

El Repository no deberá marcar `APPLIED` hasta que las postconditions requeridas se cumplan.

---

# 108. Schema-aware postcondition

Ejemplo:

```text
ExpectedSchemaFingerprint
```

podrá utilizar introspection explícita en execution phase.

No durante definition.

---

# 109. Idempotency declaration

MigrationDefinition podrá proporcionar:

```text
MigrationIdempotencyProfile
```

pero preferiblemente derivado por operation analyzer.

---

# 110. Global migration idempotency

Se calcula conservadoramente:

```text
MigrationIdempotent
=
all operations safely repeatable
∧
same ordering semantics
∧
repository effects repeatable
```

---

# 111. IF NOT EXISTS caveat

Ejemplo:

```text
CREATE TABLE IF NOT EXISTS
```

puede ser idempotente sintácticamente pero no verificar que la tabla existente sea la esperada.

Por tanto:

```text
IF NOT EXISTS
≠
Semantic Idempotency
```

---

# 112. Resume capability

Migraciones complejas podrán ser resumibles.

Se propone:

```text
MigrationResumeCapability
├── NOT_RESUMABLE
├── RESUMABLE
├── PARTIALLY_RESUMABLE
└── UNKNOWN
```

---

# 113. Resume state

No vive en la Definition.

Se guardará en:

```text
Execution State / Repository Extension
```

---

# 114. Backfill resume

Un `BackfillOperation` podrá incluir:

```text
progress key
batch ordering
resume cursor strategy
```

sin guardar el cursor runtime en la definición.

---

# 115. Migration portability

Puede proyectarse:

```text
PORTABLE
PLATFORM_SCOPED
PORTABLE_WITH_LIMITATIONS
UNKNOWN
```

a partir de sus operaciones.

---

# 116. Platform-scoped migration

Ejemplo:

```text
uses PostgreSQL extension operation
```

podrá declararse:

```text
platform scope = PostgreSQL
```

---

# 117. Platform scope ≠ runtime detection

No hacer:

```php
if ($db === 'postgres') ...
```

durante definition building como arquitectura normal.

---

# 118. Platform-specific branch

Si se necesita:

```text
platform-specific alternative
```

deberá modelarse como:

```text
MigrationAlternativeOperation
```

o planner strategy explícita.

---

# 119. Alternative operation concept

```text
Logical Intent
├── PostgreSQL strategy candidate
├── MySQL strategy candidate
└── SQLite strategy candidate
```

Preferiblemente el intent se modela a nivel Schema y Compatibility/Planner decide, no mediante branches manuales.

---

# 120. Raw platform migration

Sólo cuando realmente no exista abstracción portable.

---

# 121. Migration serialization

`MigrationDefinition` podrá serializarse conceptualmente para:

```text
plan approval
CI
artifact review
checksum
diagnostics
```

---

# 122. Serialized migration format

Debe ser:

```text
typed
versioned
deterministic
extension-aware
```

---

# 123. PHP serialize prohibited as public contract

No:

```php
serialize($migrationDefinition);
```

como formato persistente oficial.

---

# 124. Serialized operation references

Cada operación deberá incluir:

```text
operation kind
operation ID
payload version
dependencies
metadata
```

---

# 125. Unknown operation extension

Política:

```text
FAIL
PRESERVE_OPAQUE
```

dependiendo del contexto.

Para ejecución:

```text
FAIL
```

será normalmente obligatorio si no se entiende la operación.

---

# 126. Definition fingerprint

Además del checksum, puede existir:

```text
MigrationDefinitionFingerprint
```

para caching y comparación interna.

---

# 127. Checksum vs fingerprint

```text
Checksum
=
repository drift contract
```

```text
Fingerprint
=
internal structural identity accelerator
```

---

# 128. Definition equality

Se podrán distinguir:

```text
Exact Definition Equality
Structural Definition Equality
Semantic Definition Equality
```

---

# 129. Exact equality

Puede considerar:

```text
description
tags
source metadata
```

según profile.

---

# 130. Semantic equality

Principalmente:

```text
identity
dependencies
operations
rollback semantics
requirements
```

---

# 131. Migration modification after apply

Si semantic checksum cambia:

```text
Applied migration mutated
```

Debe producir:

```text
MigrationChecksumMismatch
```

o drift report.

---

# 132. Allowed metadata edits

Una policy podría permitir modificar:

```text
description
comment
source formatting
```

sin considerarlo semantic drift.

---

# 133. Never mutate history silently

Incluso si policy tolera metadata changes, el sistema deberá poder reportarlos.

---

# 134. Migration cloning

No será necesario como concepto público.

Al ser immutable:

```text
withMetadata(...)
```

podrá producir nueva definition, pero cambiar una migration histórica crea una definición diferente.

---

# 135. Migration collection

Podrá existir:

```text
MigrationDefinitionSet
```

en Planning, pero Catalog preferirá descriptors para discovery eficiente.

---

# 136. Migration identity comparison

Debe considerar namespace.

```text
app:create_users
≠
package.foo:create_users
```

---

# 137. Namespaced repository storage

El Repository deberá persistir:

```text
namespace
migration ID
```

o una identidad calificada estable.

---

# 138. Migration display name

Puede existir:

```text
MigrationDisplayName
```

separado de identidad.

---

# 139. Human-readable display

Ejemplo:

```text
Create users table
```

no usado como key de repository.

---

# 140. Extension model

Se propone:

```text
MigrationDefinitionExtension
```

para agregar:

```text
custom operation kinds
custom metadata
custom validation
custom serialization
```

---

# 141. Extension operation IDs

Tipos de extensión deberán tener:

```text
vendor namespace
type ID
version
```

Ejemplo:

```text
acme.search:create_vector_index@1
```

---

# 142. Extension constraints

Una extensión no podrá:

- ejecutar durante definition building;
- abrir connection;
- modificar repository;
- mutar otra migration;
- bypass validation;
- crear hidden dependencies.

---

# 143. Hidden dependency prohibited

Si una custom operation requiere otra migración:

```text
dependency must be declared
```

o generada explícitamente durante canonical analysis.

---

# 144. Error hierarchy

Se propone:

```text
DatabaseMigrationDefinitionException
├── InvalidMigrationDefinitionException
├── InvalidMigrationIdException
├── InvalidMigrationNamespaceException
├── DuplicateMigrationOperationIdException
├── InvalidMigrationOperationException
├── InvalidMigrationDependencyException
├── InvalidMigrationRollbackDefinitionException
├── InvalidMigrationTargetException
├── MigrationDefinitionNonDeterministicException
├── MigrationChecksumException
├── MigrationSerializationException
├── MigrationExtensionDefinitionException
├── MigrationDefinitionBudgetExceededException
└── MigrationDefinitionInvariantException
```

---

# 145. Proposed namespace

```text
VoltStack\Quantum\Database\Migration
```

con núcleo:

```text
VoltStack\Quantum\Database\Migration\Definition
VoltStack\Quantum\Database\Migration\Operation
VoltStack\Quantum\Database\Migration\Identity
VoltStack\Quantum\Database\Migration\Metadata
VoltStack\Quantum\Database\Migration\Checksum
VoltStack\Quantum\Database\Migration\Rollback
```

---

# 146. Proposed directory structure

```text
Migration/
├── Contract/
│   ├── Migration.php
│   ├── MigrationOperation.php
│   ├── MigrationDefinitionLoader.php
│   └── MigrationPreconditionDefinition.php
│
├── Definition/
│   ├── MigrationDefinition.php
│   ├── MigrationDefinitionContext.php
│   ├── MigrationDefinitionProfile.php
│   ├── MigrationDefinitionState.php
│   ├── MigrationDefinitionMetadata.php
│   └── MigrationDefinitionBudget.php
│
├── Context/
│   ├── MigrationContext.php
│   ├── MigrationSchemaOperations.php
│   ├── MigrationDataOperations.php
│   ├── MigrationValidationOperations.php
│   └── MigrationBarrierOperations.php
│
├── Identity/
│   ├── MigrationIdentity.php
│   ├── MigrationId.php
│   ├── MigrationNamespace.php
│   ├── MigrationVersion.php
│   ├── MigrationOperationId.php
│   ├── MigrationExecutionId.php
│   └── MigrationDisplayName.php
│
├── Descriptor/
│   ├── MigrationDescriptor.php
│   ├── MigrationDescriptorMetadata.php
│   └── MigrationSourceReference.php
│
├── Metadata/
│   ├── MigrationMetadata.php
│   ├── MigrationSourceMetadata.php
│   ├── MigrationSourceKind.php
│   ├── MigrationTrustLevel.php
│   ├── MigrationTag.php
│   └── MigrationTagSet.php
│
├── Dependency/
│   ├── MigrationDependency.php
│   ├── MigrationDependencyKind.php
│   └── MigrationDependencySet.php
│
├── Operation/
│   ├── MigrationOperation.php
│   ├── MigrationOperationKind.php
│   ├── MigrationOperationSet.php
│   ├── MigrationOperationMetadata.php
│   ├── SchemaMigrationOperation.php
│   ├── DataMigrationOperation.php
│   ├── ValidationMigrationOperation.php
│   ├── BarrierMigrationOperation.php
│   ├── RawMigrationOperation.php
│   └── ExtensionMigrationOperation.php
│
├── Phase/
│   └── MigrationOperationPhase.php
│
├── Target/
│   ├── MigrationTargetDefinition.php
│   ├── MigrationTargetScope.php
│   └── MigrationConnectionSelector.php
│
├── Requirement/
│   ├── MigrationPolicyRequirementSet.php
│   └── MigrationCapabilityRequirementProjection.php
│
├── Checksum/
│   ├── MigrationChecksum.php
│   ├── MigrationSourceChecksum.php
│   ├── MigrationChecksumProfile.php
│   └── MigrationChecksumGenerator.php
│
├── Fingerprint/
│   └── MigrationDefinitionFingerprint.php
│
├── Rollback/
│   ├── MigrationRollbackDefinition.php
│   └── MigrationReversibility.php
│
├── State/
│   ├── MigrationRepositoryState.php
│   ├── MigrationExecutionState.php
│   └── MigrationOutcomeCertainty.php
│
├── Result/
│   ├── MigrationResult.php
│   └── MigrationFailure.php
│
├── Validation/
│   ├── MigrationDefinitionValidator.php
│   ├── MigrationIdentityValidator.php
│   ├── MigrationOperationValidator.php
│   └── MigrationRollbackValidator.php
│
├── Serialization/
│   ├── MigrationDefinitionSerializer.php
│   ├── MigrationDefinitionDeserializer.php
│   └── OpaqueMigrationOperation.php
│
├── Extension/
│   ├── MigrationDefinitionExtension.php
│   └── FrozenMigrationExtensionRegistry.php
│
└── Exception/
    ├── DatabaseMigrationDefinitionException.php
    ├── InvalidMigrationDefinitionException.php
    ├── InvalidMigrationIdException.php
    ├── InvalidMigrationNamespaceException.php
    ├── DuplicateMigrationOperationIdException.php
    ├── InvalidMigrationOperationException.php
    ├── InvalidMigrationDependencyException.php
    ├── InvalidMigrationRollbackDefinitionException.php
    ├── InvalidMigrationTargetException.php
    ├── MigrationDefinitionNonDeterministicException.php
    ├── MigrationChecksumException.php
    ├── MigrationSerializationException.php
    ├── MigrationExtensionDefinitionException.php
    ├── MigrationDefinitionBudgetExceededException.php
    └── MigrationDefinitionInvariantException.php
```

---

# 147. Ejemplo 1 — schema migration

```php
return new class extends Migration
{
    public function id(): MigrationId
    {
        return new MigrationId(
            '2026_09_06_010000_create_users'
        );
    }

    protected function up(MigrationContext $migration): void
    {
        $migration->schema()->create(
            'users',
            function (TableBlueprint $table) {
                $table->id();
                $table->string('email', 320)->unique();
                $table->timestamps();
            }
        );
    }

    protected function down(MigrationContext $migration): void
    {
        $migration->schema()->drop('users');
    }
};
```

Canonical representation:

```text
MigrationDefinition
├── id
│   └── 2026_09_06_010000_create_users
│
├── operations
│   └── op-001
│       └── SchemaMigrationOperation
│           └── CreateTable(users)
│
└── rollback
    └── r-op-001
        └── DropTable(users)
```

---

# 148. Ejemplo 2 — mixed migration

```php
protected function up(MigrationContext $migration): void
{
    $migration->schema()->table(
        'users',
        fn (AlterTableBlueprint $table) =>
            $table->string('normalized_email', 320)->nullable()
    );

    $migration->data()->backfill(
        Backfill::table('users')
            ->source('email')
            ->target('normalized_email')
            ->transform(Expression::lower(
                Expression::column('email')
            ))
            ->batchSize(1_000)
    );

    $migration->validate()->noNulls(
        'users',
        'normalized_email'
    );

    $migration->schema()->table(
        'users',
        fn (AlterTableBlueprint $table) =>
            $table->alterColumn(
                'normalized_email',
                fn (ColumnAlteration $column) =>
                    $column->notNullable()
            )
    );
}
```

Representation:

```text
MigrationDefinition
├── op-001 Schema.AddColumn
├── op-002 Data.Backfill
├── op-003 Validation.NoNulls
└── op-004 Schema.AlterColumn
```

---

# 149. Ejemplo 3 — dependencies

```php
public function dependencies(): MigrationDependencySet
{
    return MigrationDependencySet::of(
        MigrationDependency::required(
            MigrationIdentity::of(
                'app',
                '2026_09_06_010000_create_users'
            )
        )
    );
}
```

Entonces:

```text
create_users
     ↓
create_user_profiles
```

---

# 150. Ejemplo 4 — irreversible migration

```php
protected function down(MigrationContext $migration): void
{
    $migration->irreversible(
        'The migration permanently removes legacy audit payloads.'
    );
}
```

Canonical:

```text
rollback.reversibility
=
IRREVERSIBLE
```

---

# 151. Ejemplo 5 — raw migration operation

```php
$migration->raw(
    RawMigrationOperation::trusted(
        platform: PlatformId::POSTGRESQL,
        sql: '...',
    )
);
```

Debe quedar marcado:

```text
RAW
PLATFORM_SCOPED
TRUSTED
NONPORTABLE
ANALYSIS_LIMITED
```

---

# 152. Ejemplo 6 — operation phases

```text
OP1
phase = EXPAND

OP2
phase = DATA

OP3
phase = VALIDATE

OP4
phase = CONTRACT
```

El core preserva esta metadata.

`110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md` decidirá cómo utilizarla.

---

# 153. Testing strategy

El sistema deberá tener pruebas para:

```text
migration identity
namespace collisions
definition construction
operation collection
operation ordering
operation IDs
dependency declaration
checksum determinism
source vs semantic checksum
rollback definition
irreversible migrations
mixed migrations
raw operation boundaries
serialization
extensions
budgets
persistent runtime isolation
```

---

# 154. Determinism property

Debe cumplirse:

```text
Build(MigrationSource, SameContext)
=
Equivalent MigrationDefinition
```

---

# 155. Checksum property

```text
Checksum(Build(M))
=
Checksum(Build(M))
```

para mismo código/configuración semántica.

---

# 156. Operation order property

Declaraciones iguales deberán mantener:

```text
same operation ordering
```

---

# 157. Round-trip serialization

```text
Deserialize(
    Serialize(MigrationDefinition)
)
≈
MigrationDefinition
```

según profile.

---

# 158. Mutation test

Cambiar:

```text
VARCHAR(255)
```

a:

```text
VARCHAR(320)
```

deberá cambiar semantic checksum.

---

# 159. Formatting test

Cambiar únicamente:

```text
whitespace
comments in source
```

no deberá necesariamente cambiar semantic checksum.

---

# 160. Migration system invariants

## DB-MIGRATION-SYS-001
MigrationDefinition será immutable.

## DB-MIGRATION-SYS-002
MigrationDefinition será la representación canónica de una migración construida.

## DB-MIGRATION-SYS-003
Migration source será distinta de MigrationDefinition.

## DB-MIGRATION-SYS-004
MigrationDescriptor será distinto de MigrationDefinition.

## DB-MIGRATION-SYS-005
MigrationDefinition será distinta de execution state.

## DB-MIGRATION-SYS-006
MigrationDefinition será distinta de repository state.

## DB-MIGRATION-SYS-007
MigrationDefinition será distinta de MigrationPlan.

## DB-MIGRATION-SYS-008
MigrationDefinition no contendrá Connection.

## DB-MIGRATION-SYS-009
MigrationDefinition no contendrá Transaction.

## DB-MIGRATION-SYS-010
MigrationDefinition no contendrá PDO.

## DB-MIGRATION-SYS-011
MigrationContext recolectará operaciones.

## DB-MIGRATION-SYS-012
MigrationContext no será connection wrapper.

## DB-MIGRATION-SYS-013
`up()` no ejecutará SQL como comportamiento core.

## DB-MIGRATION-SYS-014
`down()` no ejecutará SQL como comportamiento core.

## DB-MIGRATION-SYS-015
Migration operations serán tipadas.

## DB-MIGRATION-SYS-016
MigrationOperationId será estable dentro de la definición.

## DB-MIGRATION-SYS-017
Duplicate operation IDs serán inválidos.

## DB-MIGRATION-SYS-018
MigrationOperationSet será ordered.

## DB-MIGRATION-SYS-019
MigrationOperationSet será immutable después de sealing.

## DB-MIGRATION-SYS-020
Declared operation order será preservado.

## DB-MIGRATION-SYS-021
Declared order será distinto de final execution planning.

## DB-MIGRATION-SYS-022
Operation dependencies serán explícitas.

## DB-MIGRATION-SYS-023
MigrationId será estable.

## DB-MIGRATION-SYS-024
MigrationId será distinto del filename.

## DB-MIGRATION-SYS-025
MigrationId será distinto de MigrationExecutionId.

## DB-MIGRATION-SYS-026
Migration namespace será first-class.

## DB-MIGRATION-SYS-027
Full identity incluirá namespace cuando corresponda.

## DB-MIGRATION-SYS-028
Package migrations no colisionarán con application migrations por simple filename.

## DB-MIGRATION-SYS-029
Migration metadata será distinta de structural operations.

## DB-MIGRATION-SYS-030
Source location no definirá semantic identity.

## DB-MIGRATION-SYS-031
MigrationTrustLevel no omitirá validation.

## DB-MIGRATION-SYS-032
Tags no serán dependencies.

## DB-MIGRATION-SYS-033
Dependencies usarán qualified migration identity.

## DB-MIGRATION-SYS-034
Definition declarará dependencies; no las resolverá.

## DB-MIGRATION-SYS-035
Logical migration target será distinto de physical connection.

## DB-MIGRATION-SYS-036
Credentials no estarán en target definition.

## DB-MIGRATION-SYS-037
Capability requirements serán derivables desde operations.

## DB-MIGRATION-SYS-038
Derived requirements no se duplicarán manualmente sin necesidad.

## DB-MIGRATION-SYS-039
Migration checksum será versionado.

## DB-MIGRATION-SYS-040
Migration checksum será determinista.

## DB-MIGRATION-SYS-041
Source checksum será distinto de semantic checksum.

## DB-MIGRATION-SYS-042
Whitespace no deberá afectar semantic checksum.

## DB-MIGRATION-SYS-043
Runtime timestamps no afectarán semantic checksum.

## DB-MIGRATION-SYS-044
Random values no deberán participar en definition building.

## DB-MIGRATION-SYS-045
Migration definition será reproducible.

## DB-MIGRATION-SYS-046
Definition state será distinta de execution state.

## DB-MIGRATION-SYS-047
Repository state será distinta de execution state.

## DB-MIGRATION-SYS-048
Applied será distinto de succeeded attempt.

## DB-MIGRATION-SYS-049
Execution IDs serán per-attempt.

## DB-MIGRATION-SYS-050
Multiple attempts podrán pertenecer a una MigrationId.

## DB-MIGRATION-SYS-051
MigrationDescriptor será lightweight.

## DB-MIGRATION-SYS-052
Definition loading será separable de discovery.

## DB-MIGRATION-SYS-053
Lazy definition loading no ejecutará DB operations.

## DB-MIGRATION-SYS-054
Reversibility será explícita.

## DB-MIGRATION-SYS-055
REVERSIBLE será distinto de PARTIALLY_REVERSIBLE.

## DB-MIGRATION-SYS-056
PARTIALLY_REVERSIBLE será distinto de IRREVERSIBLE.

## DB-MIGRATION-SYS-057
UNKNOWN reversibility será first-class.

## DB-MIGRATION-SYS-058
Presence de rollback definition no implicará rollback safety.

## DB-MIGRATION-SYS-059
Rollback operations tendrán IDs propios.

## DB-MIGRATION-SYS-060
MigrationResult será distinto de MigrationDefinition.

## DB-MIGRATION-SYS-061
Outcome certainty será explícita.

## DB-MIGRATION-SYS-062
Migration failure conservará failed operation ID cuando sea conocido.

## DB-MIGRATION-SYS-063
Definition validation ocurrirá antes de planning.

## DB-MIGRATION-SYS-064
Invalid identity impedirá definition sealing.

## DB-MIGRATION-SYS-065
Invalid operations impedirán definition sealing.

## DB-MIGRATION-SYS-066
Definition budgets serán explícitos.

## DB-MIGRATION-SYS-067
Budget overflow no truncará operaciones.

## DB-MIGRATION-SYS-068
Risk hints no reemplazarán Migration Safety System.

## DB-MIGRATION-SYS-069
Labels serán distintos de operation IDs.

## DB-MIGRATION-SYS-070
Migration capability projection será derivada.

## DB-MIGRATION-SYS-071
Migration type será derivado desde operations.

## DB-MIGRATION-SYS-072
Mixed migration será representada explícitamente.

## DB-MIGRATION-SYS-073
Operation phase será metadata estructurada.

## DB-MIGRATION-SYS-074
Phase no modificará orden silenciosamente.

## DB-MIGRATION-SYS-075
Preconditions serán distintas de definition branching.

## DB-MIGRATION-SYS-076
Postconditions podrán formar parte del execution contract.

## DB-MIGRATION-SYS-077
Repository no marcará applied antes de required postconditions.

## DB-MIGRATION-SYS-078
Idempotency será distinta de IF NOT EXISTS.

## DB-MIGRATION-SYS-079
Idempotency será distinta de replayability.

## DB-MIGRATION-SYS-080
Resume state no vivirá en MigrationDefinition.

## DB-MIGRATION-SYS-081
Backfill runtime cursor no vivirá en definition.

## DB-MIGRATION-SYS-082
Portability podrá derivarse.

## DB-MIGRATION-SYS-083
Platform-specific migrations serán explícitas.

## DB-MIGRATION-SYS-084
Platform branching arbitrario será evitado.

## DB-MIGRATION-SYS-085
Raw migration operations serán platform-scoped cuando aplique.

## DB-MIGRATION-SYS-086
Raw operations serán analysis-limited.

## DB-MIGRATION-SYS-087
MigrationDefinition serialization será versionada.

## DB-MIGRATION-SYS-088
MigrationDefinition serialization será deterministic.

## DB-MIGRATION-SYS-089
PHP serialize no será contrato público.

## DB-MIGRATION-SYS-090
Unknown executable operation kinds no serán ignorados.

## DB-MIGRATION-SYS-091
MigrationDefinition fingerprint será distinto de repository checksum contract.

## DB-MIGRATION-SYS-092
Exact equality será distinta de semantic equality.

## DB-MIGRATION-SYS-093
Applied migration mutation será detectable.

## DB-MIGRATION-SYS-094
Metadata-only change podrá distinguirse de semantic change.

## DB-MIGRATION-SYS-095
Migration identity incluirá namespace.

## DB-MIGRATION-SYS-096
Display name no será repository key.

## DB-MIGRATION-SYS-097
Extensions usarán namespaced operation type IDs.

## DB-MIGRATION-SYS-098
Extension operations serán versionadas.

## DB-MIGRATION-SYS-099
Extensions no ejecutarán DB operations durante definition building.

## DB-MIGRATION-SYS-100
Extensions no crearán hidden dependencies.

## DB-MIGRATION-SYS-101
Definition errors serán typed.

## DB-MIGRATION-SYS-102
Migration definitions serán shareable en persistent runtimes si son immutable.

## DB-MIGRATION-SYS-103
Mutable definition builder state será operation-scoped.

## DB-MIGRATION-SYS-104
No existirá global current migration mutable.

## DB-MIGRATION-SYS-105
No existirá global current connection dentro de definition subsystem.

## DB-MIGRATION-SYS-106
No existirá global current tenant dentro de definition subsystem.

## DB-MIGRATION-SYS-107
FrankenPHP no reutilizará mutable construction state entre operaciones.

## DB-MIGRATION-SYS-108
RoadRunner no reutilizará mutable construction state entre jobs.

## DB-MIGRATION-SYS-109
OpenSwoole no reutilizará mutable construction state entre coroutines.

## DB-MIGRATION-SYS-110
Migration definition building será database-independent.

## DB-MIGRATION-SYS-111
Migration definition building será repository-independent.

## DB-MIGRATION-SYS-112
Migration definition building será execution-independent.

## DB-MIGRATION-SYS-113
Migration definition building será transaction-independent.

## DB-MIGRATION-SYS-114
Migration definition building será lock-independent.

## DB-MIGRATION-SYS-115
Definition checksum no incluirá runtime execution state.

## DB-MIGRATION-SYS-116
Definition checksum no incluirá repository batch state.

## DB-MIGRATION-SYS-117
Schema operations conservarán typed Schema AST intent.

## DB-MIGRATION-SYS-118
Data operations conservarán typed Query intent cuando sea posible.

## DB-MIGRATION-SYS-119
Validation operations serán first-class.

## DB-MIGRATION-SYS-120
Barrier operations serán first-class.

## DB-MIGRATION-SYS-121
Migration definition no será únicamente una lista SQL.

## DB-MIGRATION-SYS-122
Migration definition no será únicamente un Schema AST.

## DB-MIGRATION-SYS-123
Migration definition podrá orquestar schema + data.

## DB-MIGRATION-SYS-124
Operation ordering será determinista.

## DB-MIGRATION-SYS-125
Dependency identity será determinista.

## DB-MIGRATION-SYS-126
MigrationDefinition canonicalization será idempotente.

## DB-MIGRATION-SYS-127
Canonicalization no ejecutará I/O.

## DB-MIGRATION-SYS-128
Definition comparison será side-effect free.

## DB-MIGRATION-SYS-129
Semantic checksum generation será side-effect free.

## DB-MIGRATION-SYS-130
Descriptor loading no cambiará repository state.

## DB-MIGRATION-SYS-131
Descriptor loading no cambiará schema.

## DB-MIGRATION-SYS-132
Definition loader errors serán distinguibles de migration execution errors.

## DB-MIGRATION-SYS-133
Unknown extension operations impedirán execution salvo handler válido.

## DB-MIGRATION-SYS-134
Opaque operation preservation no implicará ejecutabilidad.

## DB-MIGRATION-SYS-135
Migration requirements no podrán falsear platform capabilities.

## DB-MIGRATION-SYS-136
Policy requirements no podrán alterar technical facts.

## DB-MIGRATION-SYS-137
MigrationDefinition no decidirá safety final.

## DB-MIGRATION-SYS-138
MigrationDefinition no decidirá compatibility final.

## DB-MIGRATION-SYS-139
MigrationDefinition no decidirá transaction strategy final.

## DB-MIGRATION-SYS-140
MigrationDefinition no decidirá lock backend.

## DB-MIGRATION-SYS-141
MigrationDefinition no actualizará repository.

## DB-MIGRATION-SYS-142
MigrationDefinition no tendrá execution retries.

## DB-MIGRATION-SYS-143
MigrationDefinition no conocerá execution outcome.

## DB-MIGRATION-SYS-144
MigrationResult no mutará MigrationDefinition.

## DB-MIGRATION-SYS-145
Migration definition errors serán diagnosticables antes de DB I/O.

## DB-MIGRATION-SYS-146
MigrationDefinition deberá poder inspeccionarse por tooling.

## DB-MIGRATION-SYS-147
MigrationDefinition deberá poder fingerprintarse.

## DB-MIGRATION-SYS-148
MigrationDefinition deberá poder validarse sin DB real.

## DB-MIGRATION-SYS-149
MigrationDefinition deberá preservar intención de evolución.

## DB-MIGRATION-SYS-150
VoltStack tratará una migración como un artefacto estructurado y reproducible, no como código arbitrario ejecutado contra una conexión.

---

# 161. Anti-patterns

## 161.1 Migration object con PDO

Incorrecto:

```php
class CreateUsersMigration
{
    public PDO $connection;
}
```

---

## 161.2 up() que ejecuta

Incorrecto:

```php
public function up(): void
{
    $this->pdo->exec('CREATE TABLE ...');
}
```

---

## 161.3 ID derivado solo de path

Incorrecto:

```text
Migration identity
=
C:\project\database\migrations\file.php
```

---

## 161.4 Random operation IDs

Incorrecto:

```php
new MigrationOperationId(
    bin2hex(random_bytes(16))
);
```

si se espera checksum reproducible.

---

## 161.5 Checksum del source únicamente

Insuficiente para determinar cambio semántico.

---

## 161.6 Dynamic definition

Incorrecto:

```php
if (getenv('APP_ENV') === 'production') {
    $table->string('production_only');
}
```

sin input explícito/versionado.

---

## 161.7 Current time in definition

Incorrecto:

```php
$table->default(now());
```

si la intención real es "momento de inserción".

Correcto conceptualmente:

```php
$table->default(
    SchemaExpression::currentTimestamp()
);
```

---

## 161.8 Database introspection in up()

Incorrecto como branching de definición:

```php
if (Schema::hasColumn('users', 'email')) {
    ...
}
```

---

## 161.9 Platform branching

Incorrecto:

```php
if ($driver === 'mysql') {
    ...
} else {
    ...
}
```

como diseño normal de una migration portable.

---

## 161.10 `down()` implies safe rollback

Incorrecto.

La existencia de código de reversión no garantiza recuperación de datos.

---

# 162. Fórmula canónica

```text
MigrationDefinition
=
Identity
+
Metadata
+
Dependencies
+
Ordered Typed Operations
+
Rollback Semantics
+
Target Definition
+
Policy Requirements
+
Extension Metadata
```

---

# 163. Fórmula de checksum

```text
SemanticChecksum(M)
=
H(
    Normalize(
        Identity(M),
        Dependencies(M),
        Operations(M),
        Rollback(M),
        Requirements(M)
    ),
    ChecksumVersion
)
```

---

# 164. Fórmula de tipo

```text
MigrationType(M)
=
classify(
    operation kinds in M
)
```

Ejemplo:

```text
Only Schema → SCHEMA_ONLY
Only Data   → DATA_ONLY
Mixed       → MIXED
```

---

# 165. Fórmula de requirements

```text
Requirements(M)
=
Union(
    Requirements(OP₁),
    Requirements(OP₂),
    ...
    Requirements(OPₙ)
)
+
MigrationLevelRequirements
```

---

# 166. Fórmula de reversibilidad

```text
Reversibility(M)
=
Analyze(
    ExplicitRollbackDefinition,
    ForwardOperations,
    RollbackOperations
)
```

pero:

```text
ReversibleStructure
≠
SafeDataRecovery
```

---

# 167. Arquitectura final

```text
                 Migration Source
                        │
                        ▼
               Migration Descriptor
                        │
                        ▼
              Definition Loader
                        │
                        ▼
                Migration Object
                        │
                        ▼
                MigrationContext
                  /      |      \
                 /       |       \
                ▼        ▼        ▼
             Schema     Data   Validation
                \        |        /
                 \       |       /
                  ▼      ▼      ▼
               Operation Collector
                        │
                        ▼
                 Validation
                        │
                        ▼
                    Seal
                        │
                        ▼
              MigrationDefinition
               /      |       \
              /       |        \
             ▼        ▼         ▼
       Dependencies Checksum  Rollback
             │        │         │
             └────────┼─────────┘
                      ▼
                Discovery /
                 Planning
```

---

# 168. Regla arquitectónica final

> **MigrationDefinition debe capturar completamente la intención versionada de evolución sin poseer ninguna responsabilidad de descubrimiento, persistencia histórica, planificación física o ejecución runtime.**

En forma resumida:

```text
Definition says what the migration means.
Discovery says where it comes from.
Repository says whether it ran.
Planner says how it should proceed.
Executor says what happened.
```

---

# 169. Resultado

Con `Database Migration System`, VoltStack dispone de un modelo central capaz de representar de manera uniforme:

```text
Schema-only migrations
Data migrations
Mixed migrations
Validation steps
Backfills
Barriers
Raw escape hatches
Extension operations
Rollback semantics
Dependencies
Targets
Checksums
```

sin convertir el migration file en un script imperativo conectado directamente a la base.

Esto permite que los documentos posteriores trabajen sobre artefactos estables:

```text
MigrationDescriptor
MigrationDefinition
MigrationIdentity
MigrationOperationSet
MigrationChecksum
MigrationRollbackDefinition
```

en lugar de depender de ejecución arbitraria de PHP.

---

# 170. Siguiente documento

```text
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack encuentra, indexa, valida y ensambla las migraciones disponibles desde diferentes fuentes:

```text
Application
Packages
Framework
Modules
Plugins
Testing
      │
      ▼
Migration Sources
      │
      ▼
Discovery
      │
      ▼
Descriptors
      │
      ▼
Identity Validation
      │
      ▼
Migration Catalog
```

y deberá diferenciar especialmente:

```text
Discovery
≠
Definition Loading

Discovery
≠
Repository

Discovery
≠
Pending Calculation

Discovery
≠
Planning

Filesystem Order
≠
Migration Dependency Order

Source Path
≠
Migration Identity
```