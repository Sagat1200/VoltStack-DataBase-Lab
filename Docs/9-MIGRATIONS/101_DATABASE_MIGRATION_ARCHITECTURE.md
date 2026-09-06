# 101_DATABASE_MIGRATION_ARCHITECTURE.md

# VoltStack Quantum Database
## Database Migration Architecture

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 101 — Database Migration Architecture  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Architecture` define la arquitectura general mediante la cual VoltStack administra la **evolución versionada, reproducible, observable y segura del estado estructural de una base de datos**.

El sistema de migraciones conecta los subsistemas ya definidos:

```text
Schema Model
Schema AST
Schema Builder
Schema Introspection
Schema Metadata
Schema Diff
Schema Compiler
Platform Compatibility
Execution Engine
Transaction Manager
Telemetry
```

para resolver una pregunta diferente:

> **¿Cómo evoluciona una base de datos desde un estado conocido hacia otro estado mediante una secuencia controlada de cambios?**

Formalmente:

```text
Database State S₀
        │
        ▼
Migration M₁
        │
        ▼
Database State S₁
        │
        ▼
Migration M₂
        │
        ▼
Database State S₂
        │
       ...
        │
        ▼
Database State Sₙ
```

---

# 2. Principio central

> **Una migración representa una unidad versionada de evolución de base de datos; no representa SQL, una transacción, un Schema AST ni una simple función `up()`/`down()`.**

Por tanto:

```text
Migration
≠
SQL Script

Migration
≠
Schema AST

Migration
≠
Schema Diff

Migration
≠
Schema Builder

Migration
≠
Transaction

Migration
≠
Execution Plan

Migration
≠
Database State
```

Una migración **orquesta operaciones** que eventualmente serán analizadas, planificadas, compiladas y ejecutadas por los subsistemas correspondientes.

---

# 3. Objetivos

El sistema deberá proporcionar:

- migraciones deterministas;
- identificación estable;
- descubrimiento;
- ordenamiento;
- dependencias;
- historial persistente;
- batches;
- planificación;
- ejecución;
- rollback;
- migraciones basadas en Schema Diff;
- compatibilidad multiplataforma;
- análisis de seguridad;
- soporte zero-downtime;
- operaciones de datos;
- transacciones;
- reintentos controlados;
- recuperación ante fallos;
- locking;
- concurrencia;
- checksums;
- detección de drift;
- observabilidad;
- extensibilidad;
- soporte para runtimes persistentes;
- integración futura con Multitenancy.

---

# 4. No objetivos

El Migration System no deberá convertirse en:

```text
ORM
Query Builder
Schema Compiler
SQL Parser
Backup System
Deployment System
Distributed Transaction Manager
Business Workflow Engine
```

Podrá integrarse con esos sistemas, pero mantendrá límites arquitectónicos claros.

---

# 5. Posición arquitectónica

```text
                  Application
                      │
                      ▼
               Migration Files
                      │
                      ▼
             Migration Discovery
                      │
                      ▼
              Migration Catalog
                      │
                      ▼
            Migration Repository
                      │
                      ▼
             Migration Resolver
                      │
                      ▼
              Migration Planner
                      │
          ┌───────────┴───────────┐
          │                       │
          ▼                       ▼
   Schema Operations          Data Operations
          │                       │
          ▼                       ▼
      Schema AST              Query Engine
          │                       │
          ▼                       │
Compatibility Analysis            │
          │                       │
          └───────────┬───────────┘
                      ▼
              Migration Plan
                      │
                      ▼
            Migration Safety
                      │
                      ▼
            Migration Executor
                      │
             ┌────────┴────────┐
             ▼                 ▼
       Schema Compiler     Query Executor
             │                 │
             └────────┬────────┘
                      ▼
              Database Driver
                      │
                      ▼
                  Database
                      │
                      ▼
            Migration Repository
```

---

# 6. Modelo conceptual

La arquitectura se divide en:

```text
Migration Architecture
├── Definition
├── Identity
├── Discovery
├── Catalog
├── Repository
├── Dependency Resolution
├── Planning
├── Operations
├── Safety
├── Execution
├── Transactions
├── Locking
├── Failure Handling
├── Rollback
├── Batching
├── Diff Integration
├── Zero-Downtime
├── Telemetry
└── Extensions
```

---

# 7. Migration Definition

Una migración deberá ser una definición declarativa/orquestadora.

Contrato conceptual:

```php
interface Migration
{
    public function id(): MigrationId;

    public function definition(): MigrationDefinition;
}
```

La definición podría contener:

```php
final readonly class MigrationDefinition
{
    public function __construct(
        public MigrationId $id,
        public MigrationMetadata $metadata,
        public MigrationDependencySet $dependencies,
        public MigrationOperationFactory $operations,
        public ?MigrationRollbackDefinition $rollback,
    ) {}
}
```

La implementación concreta podrá ofrecer una API más ergonómica.

---

# 8. API para desarrolladores

VoltStack podrá soportar una experiencia similar a:

```php
return new class extends Migration
{
    public function up(MigrationContext $migration): void
    {
        $migration->schema()->create('users', function (TableBlueprint $table) {
            $table->id();
            $table->string('email', 320)->unique();
            $table->timestamps();
        });
    }

    public function down(MigrationContext $migration): void
    {
        $migration->schema()->drop('users');
    }
};
```

Pero arquitectónicamente:

```text
up()
↓
collect operations
```

no:

```text
up()
↓
execute SQL immediately
```

---

# 9. Imperative-looking API, declarative internals

La API puede parecer imperativa:

```php
$schema->create(...);
$schema->table(...);
```

pero internamente deberá producir:

```text
MigrationOperationCollector
        │
        ▼
MigrationOperationSet
        │
        ▼
Migration Planner
```

Esto permite:

```text
preview
dry-run
validation
compatibility analysis
safety analysis
dependency ordering
zero-downtime planning
telemetry
```

antes de ejecutar.

---

# 10. Migration ≠ Schema Migration únicamente

VoltStack deberá reconocer al menos:

```text
Migration
├── SchemaMigration
├── DataMigration
├── MixedMigration
└── ExtensionMigration
```

---

# 11. Schema Migration

Contiene operaciones estructurales:

```text
CreateTable
AlterTable
DropTable
AddColumn
DropColumn
CreateIndex
AddForeignKey
AddConstraint
...
```

Estas operaciones terminan en:

```text
Schema AST
```

---

# 12. Data Migration

Contiene transformaciones de datos:

```text
UPDATE
INSERT
DELETE
backfill
data normalization
data conversion
```

y deberá utilizar:

```text
Query Engine
```

no SQL interpolado arbitrariamente como mecanismo principal.

---

# 13. Mixed Migration

Ejemplo:

```text
1. Add nullable column
2. Backfill data
3. Validate data
4. Make column NOT NULL
```

Esto no puede modelarse correctamente como un único Schema AST.

Por tanto:

```text
Migration
>
Schema AST
```

en nivel de orquestación.

---

# 14. Migration Operation

Se propone:

```php
interface MigrationOperation
{
    public function id(): MigrationOperationId;

    public function kind(): MigrationOperationKind;
}
```

Tipos:

```text
MigrationOperation
├── SchemaMigrationOperation
├── DataMigrationOperation
├── ValidationMigrationOperation
├── BarrierMigrationOperation
├── ExtensionMigrationOperation
└── RawMigrationOperation
```

---

# 15. SchemaMigrationOperation

Podrá encapsular:

```php
final readonly class SchemaMigrationOperation implements MigrationOperation
{
    public function __construct(
        public MigrationOperationId $id,
        public SchemaAst $schemaAst,
        public MigrationOperationMetadata $metadata,
    ) {}
}
```

---

# 16. DataMigrationOperation

No deberá almacenar necesariamente SQL.

Puede encapsular:

```text
Query AST
Command Definition
Batch Data Operation
Backfill Definition
```

---

# 17. Barrier Operation

Algunas migraciones necesitan puntos explícitos:

```text
Schema change
      ↓
Barrier
      ↓
Backfill
      ↓
Barrier
      ↓
Constraint validation
```

El barrier indica:

> Las operaciones posteriores dependen de que las anteriores hayan alcanzado un estado definido.

---

# 18. Migration Identity

Toda migración tendrá:

```text
MigrationId
```

estable.

Ejemplo:

```text
2026_09_06_000001_create_users_table
```

Pero el timestamp será convención, no identidad arquitectónica obligatoria.

---

# 19. MigrationId

```php
final readonly class MigrationId
{
    public function __construct(
        public string $value,
    ) {}
}
```

Requisitos:

```text
stable
unique within migration namespace
serializable
comparable
safe for repository storage
```

---

# 20. MigrationId ≠ filename

Aunque pueda derivarse inicialmente:

```text
MigrationId
≠
FilesystemPath
```

Mover un archivo no debería necesariamente crear una nueva migración.

---

# 21. MigrationId ≠ batch

```text
MigrationId
≠
MigrationBatchId
```

---

# 22. MigrationId ≠ execution

```text
MigrationId
≠
MigrationExecutionId
```

Una misma migración puede tener diferentes intentos de ejecución.

---

# 23. Identidades

Se mantendrán separadas:

```text
MigrationId
MigrationOperationId
MigrationBatchId
MigrationExecutionId
MigrationPlanId
MigrationLockId
```

---

# 24. Migration Metadata

Podrá contener:

```text
description
author
createdAt
tags
source
dependencies
risk annotations
platform requirements
extension metadata
```

Pero:

```text
metadata
≠
execution state
```

---

# 25. Migration Discovery

Responsable de encontrar definiciones disponibles.

Fuentes:

```text
Application migrations
Framework migrations
Package migrations
Module migrations
Plugin migrations
Testing migrations
```

---

# 26. Discovery ≠ execution

Discovery jamás:

```text
executes migration
writes repository
opens transaction
changes schema
```

---

# 27. Migration Source

Se propone:

```text
MigrationSource
├── ApplicationMigrationSource
├── PackageMigrationSource
├── ModuleMigrationSource
├── ExtensionMigrationSource
└── TestingMigrationSource
```

---

# 28. Migration Catalog

Después del discovery:

```text
Migration Sources
       │
       ▼
Migration Discovery
       │
       ▼
Migration Catalog
```

El catálogo contiene todas las migraciones conocidas.

---

# 29. MigrationCatalog

```php
final readonly class MigrationCatalog
{
    /** @var MigrationId => MigrationDescriptor */
    private array $migrations;
}
```

Debe ser:

```text
immutable after construction
deterministic
indexed
validated
```

---

# 30. Duplicate IDs

Dos migraciones con el mismo `MigrationId` deberán producir:

```text
DuplicateMigrationIdException
```

Nunca:

```text
last migration wins
```

---

# 31. Migration Repository

El Repository representa:

```text
persistent knowledge about migration execution
```

No las definiciones.

---

# 32. Repository responsibilities

Debe almacenar:

```text
applied migration
batch
execution timestamp
checksum
execution metadata
status
possibly duration
```

---

# 33. Repository ≠ Catalog

```text
MigrationCatalog
=
available migrations
```

```text
MigrationRepository
=
known execution history
```

---

# 34. Repository entry

Conceptualmente:

```php
final readonly class AppliedMigration
{
    public function __construct(
        public MigrationId $migrationId,
        public MigrationBatchId $batchId,
        public MigrationChecksum $checksum,
        public DateTimeImmutable $appliedAt,
        public MigrationExecutionMetadata $metadata,
    ) {}
}
```

---

# 35. Migration checksum

VoltStack deberá poder calcular:

```text
MigrationChecksum
```

para detectar modificación posterior.

---

# 36. Applied migration mutation

Si:

```text
Migration M
```

fue aplicada con:

```text
checksum A
```

pero el archivo actual produce:

```text
checksum B
```

VoltStack deberá detectar:

```text
Migration Drift
```

---

# 37. Migration drift

No deberá ignorarse silenciosamente.

Estados posibles:

```text
UNCHANGED
MODIFIED
MISSING
UNKNOWN
```

---

# 38. Missing applied migration

Caso:

```text
Repository says M applied
Catalog does not contain M
```

Esto no significa automáticamente error fatal, porque puede ocurrir por:

```text
package removal
archive
deployment topology
```

pero debe diagnosticarse.

---

# 39. Pending migration

Formalmente:

```text
Pending
=
Catalog
-
Applied
```

considerando identidad y estado.

---

# 40. Applied migration

```text
Applied(M)
=
Repository contains successful application of M
```

---

# 41. Migration dependency

Una migración podrá declarar:

```text
dependsOn(MigrationId)
```

Esto permite superar el modelo simplista:

```text
filename order = dependency order
```

---

# 42. Migration dependency graph

```text
M1 ─────► M3
 │
 └──────► M4

M2 ─────► M4
```

Representado por:

```text
MigrationDependencyGraph
```

---

# 43. Topological ordering

La ejecución deberá respetar:

```text
TopologicalSort(MigrationDependencyGraph)
```

---

# 44. Filename ordering

Podrá utilizarse como:

```text
deterministic tie-breaker
```

pero no como sustituto universal de dependencias.

---

# 45. Cyclic dependencies

```text
M1 → M2 → M3 → M1
```

deberá producir:

```text
MigrationDependencyCycleException
```

---

# 46. Missing dependency

Si:

```text
M2 depends on M1
```

pero `M1` no está en catálogo ni repository:

```text
MissingMigrationDependencyException
```

---

# 47. Migration Planner

Responsable de transformar:

```text
Migration Catalog
+
Repository State
+
Dependencies
+
Target
+
Capabilities
+
Policies
```

en:

```text
MigrationPlan
```

---

# 48. Planner ≠ Executor

Planner:

```text
decides what should happen
```

Executor:

```text
performs the planned work
```

---

# 49. MigrationPlan

```php
final readonly class MigrationPlan
{
    public function __construct(
        public MigrationPlanId $id,
        public MigrationPlanStepSet $steps,
        public MigrationPlanMetadata $metadata,
    ) {}
}
```

---

# 50. Plan steps

```text
MigrationPlan
├── AcquireLock
├── ValidateRepository
├── ExecuteMigration M1
│   ├── SchemaOperation
│   ├── DataOperation
│   └── RepositoryWrite
├── ExecuteMigration M2
├── ...
└── ReleaseLock
```

---

# 51. Migration Plan ≠ Execution Plan

Se mantendrá:

```text
MigrationPlan
≠
Database ExecutionPlan
```

Migration Plan es de nivel superior.

Puede contener múltiples:

```text
Schema execution plans
Query execution plans
Transaction scopes
Repository operations
```

---

# 52. Plan immutability

Una vez construido:

```text
MigrationPlan
```

deberá ser immutable.

---

# 53. Migration execution instance

Estado mutable separado:

```text
MigrationPlan
      │
      ▼
MigrationExecution
```

---

# 54. MigrationExecution

Contiene:

```text
current step
completed steps
active transaction
lock lease
failure
timings
cancellation
```

No deberá almacenarse dentro del plan.

---

# 55. Preflight phase

Antes de modificar la base:

```text
Discovery
    ↓
Repository Read
    ↓
Dependency Validation
    ↓
Checksum Validation
    ↓
Compatibility Analysis
    ↓
Safety Analysis
    ↓
Plan
    ↓
Preflight Report
```

---

# 56. Preflight

Deberá poder ejecutarse sin aplicar cambios.

Ejemplo futuro:

```bash
php volt migrate --plan
```

---

# 57. Dry run

Modo:

```text
DRY_RUN
```

deberá producir:

```text
migration ordering
operations
compatibility findings
risk findings
compiled commands when available
transaction boundaries
estimated lock implications
```

sin ejecutar.

---

# 58. Validate-only

Otro modo:

```text
VALIDATE_ONLY
```

podrá comprobar:

```text
migration graph
checksums
platform compatibility
schema references
extensions
repository consistency
```

---

# 59. Execution modes

Se propone:

```php
enum MigrationExecutionMode
{
    case APPLY;
    case ROLLBACK;
    case PLAN;
    case DRY_RUN;
    case VALIDATE_ONLY;
}
```

---

# 60. Migration target

El usuario podrá indicar:

```text
latest
specific migration
batch
step count
tag
version
```

según APIs futuras.

---

# 61. Target state

Debe modelarse explícitamente:

```text
MigrationTarget
```

No mediante lógica dispersa en CLI.

---

# 62. Schema operations

El Migration Planner delegará:

```text
SchemaMigrationOperation
        ↓
Schema Planner
        ↓
Schema Compiler
```

---

# 63. Data operations

Delegará:

```text
DataMigrationOperation
        ↓
Query Engine
        ↓
Query Executor
```

---

# 64. Raw operations

VoltStack podrá ofrecer:

```php
$migration->raw(...);
```

como escape hatch.

Pero deberá clasificarse:

```text
trusted
nonportable
analysis-limited
security-sensitive
```

---

# 65. Raw migration ≠ default

La ruta recomendada será:

```text
Schema Builder
Query Engine
Typed Migration Operations
```

---

# 66. Data migration safety

Operaciones como:

```text
UPDATE users SET ...
DELETE ...
```

deberán poder recibir metadata:

```text
estimated scope
batch strategy
idempotency
retryability
locking implications
```

---

# 67. Backfill

Debe ser first-class.

```text
BackfillOperation
```

podrá definir:

```text
source
target
batch size
ordering key
resume token
idempotency
concurrency policy
```

---

# 68. Backfill ≠ single UPDATE

Para datasets grandes:

```text
Backfill
≠
UPDATE entire table
```

VoltStack deberá permitir planificación incremental.

---

# 69. Migration Safety System

El documento `111` profundizará este subsistema.

Arquitectónicamente:

```text
MigrationPlan
      │
      ▼
Safety Analyzer
      │
      ▼
MigrationSafetyReport
```

---

# 70. Risk categories

Se anticipan:

```text
SAFE
LOW
MODERATE
HIGH
DESTRUCTIVE
UNKNOWN
```

pero riesgo no equivale a autorización.

---

# 71. Safety ≠ compatibility

Ejemplo:

```text
DROP TABLE
```

puede ser:

```text
Platform Compatible = YES
```

pero:

```text
Migration Safety = DESTRUCTIVE
```

---

# 72. Safety ≠ permission

El sistema podrá decir:

```text
operation is destructive
```

pero Authorization decide si está permitida cuando dicha integración aplique.

---

# 73. Transaction architecture

Migration System no implementará transacciones directamente.

Delegará a:

```text
Transaction Manager
```

---

# 74. Transaction policy

Una migración podrá declarar:

```text
REQUIRED
PREFERRED
FORBIDDEN
PLATFORM_DEFAULT
```

---

# 75. Transaction capability

No todas las plataformas soportan:

```text
transactional DDL
```

de la misma manera.

Por tanto:

```text
MigrationTransactionStrategy
=
f(
    operations,
    capabilities,
    migration policy
)
```

---

# 76. Migration ≠ transaction

Una migración puede requerir:

```text
multiple transaction scopes
```

especialmente en:

```text
zero-downtime migrations
concurrent indexes
large backfills
```

---

# 77. Transaction boundary

Debe aparecer explícitamente en `MigrationPlan`.

Ejemplo:

```text
Begin Transaction
    ↓
Add Column
    ↓
Commit
    ↓
Backfill
    ↓
Begin Transaction
    ↓
Add Constraint
    ↓
Commit
```

---

# 78. Automatic transaction

Nunca deberá asumirse universalmente:

```text
one migration
=
one transaction
```

---

# 79. Locking

Para impedir migraciones concurrentes incompatibles:

```text
Migration Lock
```

será first-class.

---

# 80. Migration lock

Contrato conceptual:

```php
interface MigrationLockManager
{
    public function acquire(
        MigrationLockRequest $request
    ): MigrationLockLease;
}
```

---

# 81. Lock lease

```text
MigrationLockLease
```

debe tener:

```text
identity
owner
acquiredAt
expiry/heartbeat semantics when applicable
release
```

---

# 82. Distributed environments

El lock deberá funcionar cuando:

```text
Node A ──┐
Node B ──┼── deploy simultaneously
Node C ──┘
```

para impedir ejecución duplicada.

---

# 83. Lock backend

Podrá implementarse mediante:

```text
database advisory lock
migration repository lock
distributed lock integration
```

según capability.

---

# 84. Lock ≠ database transaction lock

Son conceptos distintos:

```text
Migration Coordination Lock
≠
Row/Table Lock
≠
Transaction Lock
```

---

# 85. Repository consistency

Una migración no deberá considerarse aplicada hasta que:

```text
required operations completed
+
repository state committed according to execution contract
```

---

# 86. Atomic repository update

Cuando la plataforma lo permita:

```text
migration operations
+
migration repository record
```

podrán compartir transaction scope.

Cuando no:

```text
failure semantics
```

deberán ser explícitas.

---

# 87. Failure states

Se propone:

```text
NOT_STARTED
RUNNING
SUCCEEDED
FAILED
PARTIALLY_APPLIED
CANCELLED
UNKNOWN
```

---

# 88. PARTIALLY_APPLIED

Es crítico para DDL no transaccional.

Ejemplo:

```text
Operation 1 ✓
Operation 2 ✓
Operation 3 ✗
```

No deberá registrarse simplemente:

```text
migration failed
```

sin reconocer cambios ya aplicados.

---

# 89. UNKNOWN

Puede ocurrir por:

```text
connection loss
process crash
network partition
database failover
```

cuando no pueda demostrarse el outcome.

---

# 90. Migration outcome certainty

Debe integrarse con el modelo definido en Execution Error System:

```text
CERTAIN_SUCCESS
CERTAIN_FAILURE
PARTIAL
UNKNOWN
```

---

# 91. Retry

Regla heredada del documento 86:

```text
Retryable Failure
≠
Safe Retry
```

---

# 92. Migration retry

Antes de reintentar deberá conocerse:

```text
operation idempotency
outcome certainty
transaction state
repository state
replayability
migration policy
```

---

# 93. Blind migration retry

Prohibido:

```text
catch Exception
sleep(1)
run migration again
```

---

# 94. Idempotency

Una migración no deberá considerarse idempotente simplemente porque usa:

```text
IF EXISTS
IF NOT EXISTS
```

Idempotencia es propiedad del efecto completo.

---

# 95. Operation idempotency

Puede modelarse:

```php
enum MigrationIdempotency
{
    case IDEMPOTENT;
    case CONDITIONALLY_IDEMPOTENT;
    case NON_IDEMPOTENT;
    case UNKNOWN;
}
```

---

# 96. Replayability

Separada de idempotencia:

```text
Replayable
≠
Idempotent
```

Una operación puede reconstruirse pero no ser segura al repetir.

---

# 97. Rollback

Rollback será un subsistema explícito.

```text
Migration
       │
       ├── Forward Definition
       │
       └── Rollback Definition
```

---

# 98. Rollback ≠ automatic inverse

Ejemplo:

```text
DROP COLUMN
```

no puede invertirse automáticamente recuperando los datos eliminados.

---

# 99. Reversible migration

Se propone:

```text
REVERSIBLE
PARTIALLY_REVERSIBLE
IRREVERSIBLE
UNKNOWN
```

---

# 100. Explicit irreversible migrations

VoltStack deberá permitir declarar:

```php
public function rollback(): MigrationRollbackDefinition
{
    return MigrationRollbackDefinition::irreversible(
        'Original data cannot be reconstructed.'
    );
}
```

---

# 101. Automatic inversion

Solo podrá utilizarse cuando exista prueba suficiente.

Ejemplo:

```text
Create empty table
        ↕
Drop table
```

puede parecer invertible estructuralmente.

Pero después de uso real:

```text
Drop table
```

destruye datos.

Por ello:

```text
Structural Inverse
≠
Operational Rollback Safety
```

---

# 102. Batch System

Cada ejecución agrupada podrá crear:

```text
MigrationBatch
```

---

# 103. MigrationBatch

```php
final readonly class MigrationBatch
{
    public function __construct(
        public MigrationBatchId $id,
        public int $sequence,
        public MigrationIdSet $migrations,
        public MigrationBatchMetadata $metadata,
    ) {}
}
```

---

# 104. Batch ≠ transaction

```text
Migration Batch
≠
Database Transaction
```

Un batch puede contener múltiples transacciones.

---

# 105. Batch rollback

Podrá significar:

```text
rollback migrations belonging to latest batch
```

pero siempre sujeto a reversibility/safety.

---

# 106. Schema Diff migration

VoltStack soportará:

```text
Current Schema
      │
      ▼
Schema Diff
      │
      ▼
Schema ChangeSet
      │
      ▼
Migration Generation
```

---

# 107. Generated migration

Una migración generada deberá seguir siendo:

```text
reviewable
versioned
explicit
deterministic
```

No deberá aplicarse automáticamente solo porque Diff la produjo.

---

# 108. Rename ambiguity

Schema Diff puede detectar:

```text
drop old_column
add new_column
```

pero no demostrar que fue rename.

Por tanto:

```text
possible rename
≠
confirmed rename
```

Migration generation deberá preservar esta incertidumbre.

---

# 109. Zero-downtime migrations

VoltStack deberá considerar first-class estrategias como:

```text
Expand
Migrate
Contract
```

---

# 110. Expand/Contract

Ejemplo:

```text
Phase 1 — Expand
Add new column

Phase 2 — Migrate
Backfill new column

Phase 3 — Compatibility
Application reads/writes both forms

Phase 4 — Contract
Remove old column
```

---

# 111. Migration ≠ deployment

Aunque zero-downtime pueda requerir coordinación con código:

```text
Migration System
≠
Deployment Orchestrator
```

Podrá expresar prerequisites/barriers para integrarse con uno.

---

# 112. Migration phase

Se propone:

```text
PRE_DEPLOY
DEPLOY_COMPATIBLE
POST_DEPLOY
CLEANUP
```

como metadata/orchestration capability futura.

---

# 113. Application compatibility window

Una migración podrá declarar:

```text
old application compatible?
new application compatible?
dual-version compatible?
```

para zero-downtime tooling.

No deberá inferirse automáticamente.

---

# 114. Schema drift

Debe diferenciarse:

```text
Migration Drift
≠
Schema Drift
```

Migration Drift:

```text
migration definition changed after application
```

Schema Drift:

```text
actual database schema differs from expected schema
```

---

# 115. Schema drift detection

Podrá integrar:

```text
Migration History
+
Expected Schema
+
Schema Introspection
+
Schema Diff
```

---

# 116. Drift state

```text
CLEAN
EXPECTED_DIFFERENCE
DRIFTED
UNKNOWN
```

---

# 117. Migration expected schema

Opcionalmente podrá calcularse:

```text
Sₙ
=
Apply(
    S₀,
    M₁ ... Mₙ
)
```

mediante interpretación estructural sin ejecutar DB.

---

# 118. Schema AST Interpreter integration

Para operaciones puramente estructurales:

```text
SchemaAstInterpreter
```

podrá simular evolución.

---

# 119. Mixed migrations

Si existen data-dependent schema decisions:

```text
pure static expected schema
```

puede no ser calculable completamente.

Esto deberá reflejarse como incertidumbre, no inventarse.

---

# 120. Migration state machine

```text
DISCOVERED
    │
    ▼
VALIDATED
    │
    ▼
PLANNED
    │
    ▼
PENDING
    │
    ▼
RUNNING
    │
    ├────────► FAILED
    │
    ├────────► PARTIALLY_APPLIED
    │
    ├────────► CANCELLED
    │
    ├────────► UNKNOWN
    │
    ▼
APPLIED
```

---

# 121. Definition state ≠ execution state

La migración en catálogo es immutable.

Su ejecución posee estado separado.

---

# 122. MigrationContext

Durante construcción:

```php
interface MigrationContext
{
    public function schema(): MigrationSchemaBuilder;

    public function data(): MigrationDataBuilder;
}
```

Este contexto:

```text
collects operations
```

No es un wrapper de connection.

---

# 123. No Connection in migration API by default

Evitar:

```php
$migration->connection()->exec(...);
```

como API principal.

---

# 124. Connection selection

La selección de connection será responsabilidad de:

```text
Migration Target
Connection Resolver
Execution Context
```

---

# 125. Multi-database migrations

Una migración avanzada puede apuntar a:

```text
database A
database B
```

pero eso deberá ser explícito.

---

# 126. Cross-database atomicity

Nunca asumir:

```text
Migration touching DB A + DB B
=
atomic
```

sin distributed transaction capability explícita.

---

# 127. Connection target

```php
final readonly class MigrationDatabaseTarget
{
    public function __construct(
        public ConnectionName $connection,
        public ?SchemaNamespace $namespace,
    ) {}
}
```

---

# 128. Multitenancy

Core Database no dependerá de Multitenancy.

Pero Multitenancy podrá proporcionar:

```text
TenantMigrationTargetResolver
```

---

# 129. Tenant migration

Conceptualmente:

```text
Migration Definition
        │
        ▼
Tenant Migration Integration
        │
        ├── Tenant A
        ├── Tenant B
        ├── Tenant C
        └── ...
```

---

# 130. Tenant execution state

Cada tenant deberá tener:

```text
independent execution state
```

cuando posea schema/database independiente.

---

# 131. Tenant migration failure

Fallos en tenant A no deberán contaminar estado runtime de tenant B.

---

# 132. Persistent runtimes

Con FrankenPHP:

```text
Worker
├── request 1
├── request 2
├── CLI/task execution
└── ...
```

ningún estado mutable de migración deberá permanecer accidentalmente en servicios compartidos.

---

# 133. Shared immutable state

Puede compartirse:

```text
Migration definitions
Frozen catalogs
Frozen registries
Configuration
Platform definitions
```

---

# 134. Operation-scoped state

Debe permanecer aislado:

```text
MigrationExecution
MigrationPlanSession
Transaction
Connection lease
Lock lease
Diagnostics
Current migration
Current operation
```

---

# 135. RoadRunner/OpenSwoole

La misma regla deberá mantenerse para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 136. Cancellation

Migration execution deberá aceptar:

```text
CancellationToken
Deadline
```

cuando la operación subyacente pueda responder.

---

# 137. Cancellation ≠ rollback

Si se cancela:

```text
migration execution
```

no significa que todos los efectos anteriores hayan desaparecido.

---

# 138. Deadline

Debe propagarse:

```text
Migration
    ↓
Plan Step
    ↓
Schema/Query Execution
    ↓
Statement
```

sin reiniciarse arbitrariamente.

---

# 139. Timeout

Timeout de una operación no implica outcome conocido.

Debe conservarse el modelo de certeza del Execution Engine.

---

# 140. Telemetry

El Migration System deberá emitir telemetry estructurada.

Ejemplos:

```text
migration.discovered
migration.planned
migration.started
migration.operation.started
migration.operation.completed
migration.failed
migration.completed
migration.rollback.started
migration.rollback.completed
migration.lock.wait
```

---

# 141. Metrics

Ejemplos:

```text
migration.duration
migration.operation.duration
migration.pending.count
migration.failed.count
migration.lock.wait.duration
migration.backfill.rows
```

---

# 142. Tracing

Un trace podrá verse:

```text
migration batch
└── migration M1
    ├── schema operation
    ├── backfill
    │   ├── batch 1
    │   ├── batch 2
    │   └── batch N
    └── constraint validation
```

---

# 143. Sensitive telemetry

No deberá registrar automáticamente:

```text
credentials
full raw SQL
sensitive data values
secrets
```

---

# 144. Events

Migration System podrá publicar eventos:

```text
MigrationPlanning
MigrationPlanned
MigrationStarting
MigrationStarted
MigrationCompleted
MigrationFailed
MigrationRollingBack
MigrationRolledBack
```

---

# 145. Events ≠ control flow oculto

Listeners no deberán poder alterar arbitrariamente:

```text
migration order
safety result
transaction boundaries
repository state
```

sin contratos explícitos.

---

# 146. Extension System

Se permitirá:

```text
custom migration source
custom operation
custom safety rule
custom planner strategy
custom repository
custom lock backend
```

mediante contratos.

---

# 147. Frozen extension registry

Antes de ejecutar:

```text
MigrationExtensionRegistry
        ↓
freeze
        ↓
FrozenMigrationExtensionRegistry
```

---

# 148. No last-wins

Dos extensiones con mismo identificador:

```text
duplicate extension
```

deberán producir error.

---

# 149. Extension safety

Una extensión no deberá poder saltarse silenciosamente:

```text
compatibility
safety
repository consistency
locking
transaction contracts
```

---

# 150. Migration configuration

Configuración conceptual:

```php
return [
    'table' => 'voltstack_migrations',

    'paths' => [
        database_path('migrations'),
    ],

    'locking' => true,

    'checksum' => true,

    'strict_drift_detection' => true,

    'transaction_policy' => 'platform',

    'safety' => [
        'destructive' => 'confirm',
    ],
];
```

La configuración real será definida posteriormente.

---

# 151. Repository bootstrap

Existe un problema inicial:

```text
Migration Repository
must exist
before migrations can be tracked.
```

Esto requiere un:

```text
MigrationRepositoryBootstrapper
```

mínimo y separado del flujo normal.

---

# 152. Bootstrapper

Podrá crear únicamente las estructuras internas necesarias para Migration Repository.

No deberá convertirse en un segundo Schema System.

---

# 153. Repository schema version

La propia tabla de migraciones deberá poder evolucionar.

Por tanto:

```text
Migration Repository
```

tendrá:

```text
repository schema version
```

independiente de las migraciones de aplicación.

---

# 154. Internal metadata tables

Deben distinguirse:

```text
Framework Internal Schema
≠
Application Schema
```

---

# 155. CLI integration

Futuras operaciones:

```bash
php volt migrate
php volt migrate:status
php volt migrate:plan
php volt migrate:rollback
php volt migrate:reset
php volt migrate:fresh
php volt migrate:validate
```

Pero CLI será solo adapter.

---

# 156. CLI ≠ Migration Engine

Arquitectura:

```text
CLI
 ↓
Migration Application Service
 ↓
Migration Engine
```

---

# 157. Programmatic API

También deberá ser posible:

```php
$result = $migrationManager->migrate(
    MigrationRequest::latest()
);
```

sin CLI.

---

# 158. MigrationManager

Fachada de alto nivel:

```php
interface MigrationManager
{
    public function plan(
        MigrationRequest $request
    ): MigrationPlan;

    public function execute(
        MigrationPlan $plan
    ): MigrationExecutionResult;
}
```

---

# 159. Manager ≠ god object

Internamente delegará:

```text
Discovery
Repository
Resolver
Planner
Safety
Locking
Executor
Telemetry
```

---

# 160. Migration request

```php
final readonly class MigrationRequest
{
    public function __construct(
        public MigrationExecutionMode $mode,
        public MigrationTarget $target,
        public MigrationDatabaseTarget $database,
        public MigrationPolicy $policy,
    ) {}
}
```

---

# 161. Migration policy

Podrá agrupar:

```text
safety policy
transaction policy
retry policy
drift policy
locking policy
compatibility policy
```

---

# 162. Policy ≠ capability

Policy dice:

```text
what VoltStack/user allows
```

Capability dice:

```text
what platform can do
```

---

# 163. Policy cannot override correctness

Una policy no podrá decir:

```text
ignore semantic incompatibility
```

y convertirla en `SUPPORTED`.

Puede permitir una operación riesgosa, pero no cambiar hechos técnicos.

---

# 164. Security

Migration System es una superficie privilegiada.

Puede:

```text
create tables
drop tables
modify data
change constraints
execute DDL
```

Por tanto requiere límites fuertes.

---

# 165. Trusted migration sources

Se distinguirá:

```text
FRAMEWORK
APPLICATION
PACKAGE
EXTENSION
EXTERNAL
```

---

# 166. Trust ≠ skip validation

Incluso:

```text
FRAMEWORK
```

deberá cumplir invariantes.

---

# 167. Raw migration security

Raw SQL deberá ser:

```text
explicit
trusted
platform-scoped
diagnostic-visible
nonportable by default
```

---

# 168. Identifier safety

Operaciones tipadas utilizarán:

```text
Structured Identifiers
```

y quoting será responsabilidad de Schema/SQL Compiler.

---

# 169. Secrets

Migration definitions no deberán contener credenciales.

Connection resolution ocurre externamente.

---

# 170. Destructive operations

Ejemplos:

```text
DROP TABLE
DROP COLUMN
TRUNCATE
destructive type narrowing
```

deberán ser clasificables.

---

# 171. Production safety

VoltStack podrá aplicar policies diferentes:

```text
development
testing
staging
production
```

pero el entorno no deberá cambiar semántica estructural.

---

# 172. Confirmation

Confirmaciones interactivas pertenecen a:

```text
CLI/UI adapter
```

no al core Migration Engine.

---

# 173. Non-interactive execution

CI/CD debe poder usar:

```text
explicit policy
```

sin prompts ocultos.

---

# 174. Determinism

Para mismo:

```text
Catalog
Repository State
Capabilities
Configuration
Policies
Target
```

deberá obtenerse el mismo:

```text
MigrationPlan
```

salvo metadata explícitamente no semántica.

---

# 175. Plan fingerprint

Se propone:

```text
MigrationPlanFingerprint
```

---

# 176. Plan fingerprint inputs

```text
Migration Catalog Fingerprint
Repository State Fingerprint
Capability Fingerprint
Policy Fingerprint
Planner Version
Target
```

---

# 177. Plan fingerprint use

Puede servir para:

```text
deployment approval
audit
dry-run comparison
reproducibility
telemetry correlation
```

---

# 178. Fingerprint ≠ authorization

Tener el mismo hash no autoriza ejecución.

---

# 179. Migration checksum profile

Debe distinguirse:

```text
SourceChecksum
StructuralChecksum
SemanticChecksum
```

si la arquitectura futura lo requiere.

---

# 180. Formatting changes

Cambiar:

```text
whitespace
comments
```

no debería necesariamente producir migration drift semántico.

Por ello se recomienda que el checksum principal sea sobre una representación canónica de la definición cuando sea posible.

---

# 181. Dynamic migration definitions

Debe evitarse:

```php
if (date('N') === 1) {
    $table->string('foo');
}
```

porque rompe determinismo.

---

# 182. Environment-dependent migration

También peligroso:

```php
if (env('SOMETHING')) {
    ...
}
```

Si se permite, deberá formar parte explícita del input/fingerprint.

---

# 183. Hidden database-dependent migration

Prohibido como ruta normal:

```php
if ($connection->tableExists(...)) {
    ...
}
```

durante construcción de definición.

Eso mezcla:

```text
definition
+
introspection
```

y rompe reproducibilidad.

---

# 184. Conditional migration

Si se requiere:

```text
condition
```

deberá modelarse como operación/planning condition explícita.

---

# 185. Schema existence checks

`ifExists` / `ifNotExists` son comportamientos estructurales explícitos.

No equivalen a lógica arbitraria basada en introspection.

---

# 186. Migration dependency on data

Casos como:

```text
Add NOT NULL only if no nulls exist
```

deberán modelarse mediante:

```text
ValidationOperation
```

no mediante branching oculto.

---

# 187. Validation operation

Ejemplo:

```text
AssertNoNullValues(users.email)
```

podrá ejecutarse antes de:

```text
SetNotNull(users.email)
```

---

# 188. Preconditions

Una operación podrá declarar:

```text
Preconditions
```

---

# 189. Postconditions

También:

```text
Postconditions
```

para verificar que el efecto esperado se logró.

---

# 190. Preconditions ≠ hidden branching

Si falla una precondition:

```text
migration fails/blocks
```

según policy.

No cambia silenciosamente el plan.

---

# 191. Migration invariants

A continuación se establecen los invariantes base.

## DB-MIGRATION-001
Migration será distinta de SQL.

## DB-MIGRATION-002
Migration será distinta de Schema AST.

## DB-MIGRATION-003
Migration será distinta de Schema Diff.

## DB-MIGRATION-004
Migration será distinta de Schema Builder.

## DB-MIGRATION-005
Migration será distinta de Transaction.

## DB-MIGRATION-006
Migration será distinta de Execution Plan.

## DB-MIGRATION-007
Migration será una unidad versionada de evolución.

## DB-MIGRATION-008
Migration podrá contener operaciones estructurales y de datos.

## DB-MIGRATION-009
Schema Migration utilizará Schema subsystem.

## DB-MIGRATION-010
Data Migration utilizará Query subsystem cuando sea posible.

## DB-MIGRATION-011
API imperativa no implicará ejecución inmediata.

## DB-MIGRATION-012
Migration operations serán recolectables antes de ejecutar.

## DB-MIGRATION-013
Migration definitions serán deterministas.

## DB-MIGRATION-014
Migration definitions no abrirán conexiones por defecto.

## DB-MIGRATION-015
Migration definitions no ejecutarán SQL por defecto.

## DB-MIGRATION-016
MigrationId será estable.

## DB-MIGRATION-017
MigrationId será distinto del filename.

## DB-MIGRATION-018
MigrationId será distinto de BatchId.

## DB-MIGRATION-019
MigrationId será distinto de ExecutionId.

## DB-MIGRATION-020
MigrationOperationId será distinto de MigrationId.

## DB-MIGRATION-021
Duplicate MigrationId será error.

## DB-MIGRATION-022
Discovery será distinto de execution.

## DB-MIGRATION-023
Catalog representará migraciones disponibles.

## DB-MIGRATION-024
Repository representará historial persistente.

## DB-MIGRATION-025
Catalog será distinto de Repository.

## DB-MIGRATION-026
Applied state será persistido.

## DB-MIGRATION-027
Migration checksum será verificable.

## DB-MIGRATION-028
Applied migration mutation será detectable.

## DB-MIGRATION-029
Migration drift no se ignorará silenciosamente.

## DB-MIGRATION-030
Missing migration no equivaldrá automáticamente a unapplied.

## DB-MIGRATION-031
Pending será calculado desde Catalog + Repository.

## DB-MIGRATION-032
Dependencies serán explícitas cuando existan.

## DB-MIGRATION-033
Filename order no sustituirá dependency graph.

## DB-MIGRATION-034
Dependency ordering será topológico.

## DB-MIGRATION-035
Dependency cycles serán error.

## DB-MIGRATION-036
Missing dependencies serán error/diagnóstico explícito.

## DB-MIGRATION-037
Planner será distinto de Executor.

## DB-MIGRATION-038
MigrationPlan será immutable.

## DB-MIGRATION-039
MigrationPlan será distinto de DB ExecutionPlan.

## DB-MIGRATION-040
Mutable execution state estará fuera del plan.

## DB-MIGRATION-041
Preflight ocurrirá antes de modificación cuando sea requerido.

## DB-MIGRATION-042
Dry-run no modificará la base.

## DB-MIGRATION-043
Validate-only no modificará la base.

## DB-MIGRATION-044
Migration target será explícito.

## DB-MIGRATION-045
Schema operations delegarán al Schema System.

## DB-MIGRATION-046
Data operations delegarán al Query/Execution System.

## DB-MIGRATION-047
Raw operations serán escape hatch.

## DB-MIGRATION-048
Raw operations serán nonportable por defecto.

## DB-MIGRATION-049
Backfill será first-class.

## DB-MIGRATION-050
Backfill no implicará single massive UPDATE.

## DB-MIGRATION-051
Safety será distinta de compatibility.

## DB-MIGRATION-052
Safety será distinta de authorization.

## DB-MIGRATION-053
Destructive operation puede ser platform-compatible.

## DB-MIGRATION-054
Transactions serán delegadas al Transaction Manager.

## DB-MIGRATION-055
One migration no implicará one transaction.

## DB-MIGRATION-056
Transaction policy será explícita.

## DB-MIGRATION-057
DDL transaction capability será respetada.

## DB-MIGRATION-058
Transaction boundaries serán visibles en plan.

## DB-MIGRATION-059
Migration coordination lock será first-class.

## DB-MIGRATION-060
Migration lock será distinto de transaction lock.

## DB-MIGRATION-061
Concurrent migrators deberán coordinarse.

## DB-MIGRATION-062
Repository consistency será protegida.

## DB-MIGRATION-063
Migration no será APPLIED antes de cumplir execution contract.

## DB-MIGRATION-064
Partial application será estado explícito.

## DB-MIGRATION-065
Unknown outcome será estado explícito.

## DB-MIGRATION-066
Connection loss no implicará safe retry.

## DB-MIGRATION-067
Retryable failure será distinto de safe retry.

## DB-MIGRATION-068
Blind retry estará prohibido.

## DB-MIGRATION-069
Idempotency será explícita.

## DB-MIGRATION-070
Replayability será distinta de idempotency.

## DB-MIGRATION-071
Rollback será distinto de automatic inverse.

## DB-MIGRATION-072
Rollback safety será distinta de structural inverse.

## DB-MIGRATION-073
Irreversible migrations serán representables.

## DB-MIGRATION-074
Partial reversibility será representable.

## DB-MIGRATION-075
Batch será distinto de transaction.

## DB-MIGRATION-076
Batch rollback respetará reversibility.

## DB-MIGRATION-077
Schema Diff podrá generar candidatos de migración.

## DB-MIGRATION-078
Generated migration deberá ser revisable.

## DB-MIGRATION-079
Schema Diff no confirmará rename sin evidencia suficiente.

## DB-MIGRATION-080
Zero-downtime será first-class.

## DB-MIGRATION-081
Expand/Migrate/Contract será soportable.

## DB-MIGRATION-082
Migration será distinta de deployment orchestration.

## DB-MIGRATION-083
Deployment prerequisites podrán modelarse explícitamente.

## DB-MIGRATION-084
Migration Drift será distinto de Schema Drift.

## DB-MIGRATION-085
Schema Drift utilizará introspection/diff.

## DB-MIGRATION-086
Expected schema podrá calcularse cuando sea posible.

## DB-MIGRATION-087
Schema AST Interpreter podrá participar en simulación.

## DB-MIGRATION-088
Uncertainty no se reemplazará por assumptions.

## DB-MIGRATION-089
Definition state será distinto de execution state.

## DB-MIGRATION-090
MigrationContext no será Connection wrapper.

## DB-MIGRATION-091
Connection resolution será externa a definition.

## DB-MIGRATION-092
Multi-database targets serán explícitos.

## DB-MIGRATION-093
Cross-database atomicity no será asumida.

## DB-MIGRATION-094
Multitenancy será integración opcional.

## DB-MIGRATION-095
Tenant execution state será aislado.

## DB-MIGRATION-096
Persistent workers no compartirán mutable execution state.

## DB-MIGRATION-097
FrankenPHP será soportado sin state leakage.

## DB-MIGRATION-098
RoadRunner seguirá las mismas reglas de aislamiento.

## DB-MIGRATION-099
OpenSwoole seguirá las mismas reglas de aislamiento.

## DB-MIGRATION-100
Cancellation será distinta de rollback.

## DB-MIGRATION-101
Deadline será propagado.

## DB-MIGRATION-102
Timeout no implicará known outcome.

## DB-MIGRATION-103
Telemetry será estructurada.

## DB-MIGRATION-104
Telemetry no expondrá secretos por defecto.

## DB-MIGRATION-105
Events no serán control flow oculto.

## DB-MIGRATION-106
Extensions usarán contratos explícitos.

## DB-MIGRATION-107
Extension registries serán frozen.

## DB-MIGRATION-108
Extension collision no usará last-wins.

## DB-MIGRATION-109
Extensions no saltarán safety silenciosamente.

## DB-MIGRATION-110
Extensions no saltarán compatibility silenciosamente.

## DB-MIGRATION-111
Repository bootstrap será separado.

## DB-MIGRATION-112
Repository bootstrap no será segundo Schema System.

## DB-MIGRATION-113
Internal schema será distinto de application schema.

## DB-MIGRATION-114
CLI será adapter.

## DB-MIGRATION-115
Migration Engine será usable programáticamente.

## DB-MIGRATION-116
MigrationManager no implementará todas las responsabilidades internamente.

## DB-MIGRATION-117
Policies serán explícitas.

## DB-MIGRATION-118
Policy será distinta de capability.

## DB-MIGRATION-119
Policy no podrá cambiar hechos de compatibilidad.

## DB-MIGRATION-120
Migration System será superficie privilegiada.

## DB-MIGRATION-121
Migration source trust será explícito.

## DB-MIGRATION-122
Trust no omitirá validation.

## DB-MIGRATION-123
Raw SQL será explícito.

## DB-MIGRATION-124
Credentials no vivirán en migration definitions.

## DB-MIGRATION-125
Destructive operations serán clasificables.

## DB-MIGRATION-126
Environment policy no cambiará semántica estructural.

## DB-MIGRATION-127
Interactive confirmation pertenecerá al adapter.

## DB-MIGRATION-128
CI/CD podrá operar non-interactively mediante policy.

## DB-MIGRATION-129
Planning será determinista.

## DB-MIGRATION-130
MigrationPlan podrá fingerprintarse.

## DB-MIGRATION-131
Fingerprint no será authorization.

## DB-MIGRATION-132
Migration checksum será deterministic.

## DB-MIGRATION-133
Canonical checksum será preferible a source formatting checksum.

## DB-MIGRATION-134
Dynamic hidden migration definitions serán evitadas.

## DB-MIGRATION-135
Environment-dependent behavior deberá ser input explícito.

## DB-MIGRATION-136
Hidden introspection during definition será evitada.

## DB-MIGRATION-137
Conditional behavior será estructurado.

## DB-MIGRATION-138
Validation operations serán first-class.

## DB-MIGRATION-139
Preconditions serán explícitas.

## DB-MIGRATION-140
Postconditions podrán ser explícitas.

## DB-MIGRATION-141
Failed precondition no cambiará silenciosamente el plan.

## DB-MIGRATION-142
Compatibility será evaluable antes de execution.

## DB-MIGRATION-143
Safety será evaluable antes de execution.

## DB-MIGRATION-144
Schema Compiler no administrará Migration Repository.

## DB-MIGRATION-145
Query Executor no administrará Migration Repository.

## DB-MIGRATION-146
Migration Repository no compilará SQL.

## DB-MIGRATION-147
Migration Planner no ejecutará SQL.

## DB-MIGRATION-148
Migration Executor no reinterpretará schema semantics.

## DB-MIGRATION-149
Outcome certainty será preservada.

## DB-MIGRATION-150
VoltStack priorizará reproducibilidad y seguridad sobre conveniencia implícita.

---

# 192. Anti-patterns

## 192.1 Migración como script SQL

```php
public function up()
{
    DB::statement('ALTER TABLE ...');
}
```

como arquitectura principal.

Problemas:

```text
no AST
no portability
limited safety analysis
limited diff integration
limited capability analysis
```

---

## 192.2 `up()` ejecuta inmediatamente

Incorrecto:

```text
up()
 ↓
PDO::exec()
```

Correcto:

```text
up()
 ↓
Operation Collector
 ↓
Planner
 ↓
Safety
 ↓
Executor
```

---

## 192.3 Orden únicamente por filename

Incorrecto:

```text
sort filenames
execute
```

Preferir:

```text
Dependency Graph
+
Deterministic Tie Breaking
```

---

## 192.4 Una migración = una transacción

No es universalmente válido.

---

## 192.5 Rollback automático de todo

Incorrecto:

```text
DROP COLUMN
    ↓ inverse
ADD COLUMN
```

Eso recupera estructura, no datos.

---

## 192.6 Reintentar toda migración

Incorrecto ante outcome incierto.

---

## 192.7 Ocultar emulación

Incorrecto:

```text
ALTER unsupported
      ↓
rebuild table secretly
```

La estrategia debe aparecer en el plan.

---

## 192.8 Ignorar checksum

Permitir modificar migraciones aplicadas sin detectar drift destruye reproducibilidad.

---

## 192.9 Consultar DB durante definición

Incorrecto:

```php
if (Schema::hasColumn(...)) {
    ...
}
```

como branching oculto de la definición.

---

## 192.10 Migration Manager god object

Incorrecto:

```text
MigrationManager
├── scans files
├── parses
├── plans
├── compiles SQL
├── opens PDO
├── handles transactions
├── writes telemetry
└── performs rollback
```

Debe delegar.

---

# 193. Ejemplo completo

Migración:

```php
return new class extends Migration
{
    public function up(MigrationContext $migration): void
    {
        $migration->schema()->table('users', function (AlterTableBlueprint $table) {
            $table->string('normalized_email', 320)->nullable();
        });

        $migration->data()->backfill(
            Backfill::table('users')
                ->target('normalized_email')
                ->expression(
                    Expression::lower(
                        Expression::column('email')
                    )
                )
                ->batchSize(1000)
        );

        $migration->validate(
            Validation::noNulls('users', 'normalized_email')
        );

        $migration->schema()->table('users', function (AlterTableBlueprint $table) {
            $table->alterColumn(
                'normalized_email',
                fn (ColumnAlteration $column) => $column->notNullable()
            );

            $table->unique('normalized_email');
        });
    }
};
```

---

# 194. Representación interna

```text
Migration M42
│
├── OP1 AddColumn
│
├── OP2 Backfill
│
├── OP3 ValidateNoNulls
│
├── OP4 SetNotNull
│
└── OP5 AddUniqueConstraint
```

Dependencias:

```text
OP1
 ↓
OP2
 ↓
OP3
 ↓
OP4
 ↓
OP5
```

---

# 195. Planning

```text
M42
 │
 ▼
Compatibility Analysis
 │
 ▼
Safety Analysis
 │
 ▼
Migration Planner
 │
 ├── Transaction A
 │   └── AddColumn
 │
 ├── Backfill batches
 │   ├── 1
 │   ├── 2
 │   └── N
 │
 ├── Validation
 │
 └── Transaction B
     ├── SetNotNull
     └── AddUniqueConstraint
```

---

# 196. Execution

```text
Acquire Migration Lock
          │
          ▼
Validate Repository
          │
          ▼
Execute Transaction A
          │
          ▼
Execute Backfill
          │
          ▼
Validate
          │
          ▼
Execute Transaction B
          │
          ▼
Write AppliedMigration
          │
          ▼
Release Migration Lock
```

---

# 197. Arquitectura de componentes

```text
Migration/
├── Contract
├── Definition
├── Identity
├── Discovery
├── Catalog
├── Repository
├── Dependency
├── Planning
├── Operation
├── Execution
├── Rollback
├── Batch
├── Lock
├── Safety
├── Backfill
├── Validation
├── Drift
├── Checksum
├── Policy
├── Target
├── Telemetry
├── Event
├── Extension
├── Runtime
└── Exception
```

---

# 198. Namespace propuesto

```text
VoltStack\Quantum\Database\Migration
```

---

# 199. Estructura propuesta

```text
src/Quantum/Database/Migration/
│
├── Contract/
│   ├── Migration.php
│   ├── MigrationRepository.php
│   ├── MigrationPlanner.php
│   ├── MigrationExecutor.php
│   ├── MigrationLockManager.php
│   └── MigrationSource.php
│
├── Definition/
│   ├── MigrationDefinition.php
│   ├── MigrationRollbackDefinition.php
│   └── MigrationMetadata.php
│
├── Identity/
│   ├── MigrationId.php
│   ├── MigrationOperationId.php
│   ├── MigrationBatchId.php
│   ├── MigrationExecutionId.php
│   └── MigrationPlanId.php
│
├── Discovery/
│   ├── MigrationDiscovery.php
│   ├── MigrationDescriptor.php
│   └── MigrationSourceRegistry.php
│
├── Catalog/
│   ├── MigrationCatalog.php
│   └── MigrationCatalogBuilder.php
│
├── Repository/
│   ├── DatabaseMigrationRepository.php
│   ├── AppliedMigration.php
│   ├── MigrationRepositoryState.php
│   └── MigrationRepositoryBootstrapper.php
│
├── Dependency/
│   ├── MigrationDependency.php
│   ├── MigrationDependencySet.php
│   ├── MigrationDependencyGraph.php
│   └── MigrationDependencyResolver.php
│
├── Planning/
│   ├── MigrationPlan.php
│   ├── MigrationPlanStep.php
│   ├── MigrationPlanStepSet.php
│   ├── MigrationPlanningContext.php
│   └── DefaultMigrationPlanner.php
│
├── Operation/
│   ├── MigrationOperation.php
│   ├── SchemaMigrationOperation.php
│   ├── DataMigrationOperation.php
│   ├── ValidationMigrationOperation.php
│   ├── BarrierMigrationOperation.php
│   ├── ExtensionMigrationOperation.php
│   └── RawMigrationOperation.php
│
├── Execution/
│   ├── MigrationExecution.php
│   ├── MigrationExecutionContext.php
│   ├── MigrationExecutionResult.php
│   ├── MigrationExecutionStatus.php
│   └── DefaultMigrationExecutor.php
│
├── Rollback/
│   ├── MigrationRollbackPlan.php
│   ├── MigrationReversibility.php
│   └── MigrationRollbackPlanner.php
│
├── Batch/
│   ├── MigrationBatch.php
│   └── MigrationBatchRepository.php
│
├── Lock/
│   ├── MigrationLockRequest.php
│   ├── MigrationLockLease.php
│   └── DatabaseMigrationLockManager.php
│
├── Safety/
│   ├── MigrationSafetyAnalyzer.php
│   ├── MigrationSafetyReport.php
│   └── MigrationRisk.php
│
├── Backfill/
│   ├── BackfillOperation.php
│   ├── BackfillPlan.php
│   └── BackfillProgress.php
│
├── Validation/
│   ├── MigrationPrecondition.php
│   ├── MigrationPostcondition.php
│   └── MigrationValidationOperation.php
│
├── Drift/
│   ├── MigrationDriftDetector.php
│   ├── MigrationDriftReport.php
│   └── SchemaDriftIntegration.php
│
├── Checksum/
│   ├── MigrationChecksum.php
│   └── MigrationChecksumGenerator.php
│
├── Policy/
│   ├── MigrationPolicy.php
│   ├── MigrationTransactionPolicy.php
│   ├── MigrationRetryPolicy.php
│   └── MigrationDriftPolicy.php
│
├── Target/
│   ├── MigrationTarget.php
│   └── MigrationDatabaseTarget.php
│
├── Telemetry/
│   └── MigrationTelemetry.php
│
├── Event/
│   └── MigrationEvent.php
│
├── Extension/
│   ├── MigrationExtensionRegistry.php
│   └── FrozenMigrationExtensionRegistry.php
│
├── Runtime/
│   └── MigrationRuntimeContext.php
│
└── Exception/
    ├── DatabaseMigrationException.php
    ├── DuplicateMigrationIdException.php
    ├── MissingMigrationDependencyException.php
    ├── MigrationDependencyCycleException.php
    ├── MigrationChecksumMismatchException.php
    ├── MigrationDriftException.php
    ├── MigrationPlanningException.php
    ├── MigrationSafetyException.php
    ├── MigrationExecutionException.php
    ├── MigrationPartiallyAppliedException.php
    ├── MigrationUnknownOutcomeException.php
    ├── MigrationRollbackException.php
    ├── IrreversibleMigrationException.php
    ├── MigrationLockException.php
    └── MigrationInvariantException.php
```

---

# 200. Relaciones con subsistemas existentes

```text
Migration
   │
   ├────► Schema
   │       ├── Builder
   │       ├── AST
   │       ├── Diff
   │       ├── Compatibility
   │       └── Compiler
   │
   ├────► Query Engine
   │       ├── AST
   │       ├── Planner
   │       └── Executor
   │
   ├────► Transaction Manager
   │
   ├────► Connection Manager
   │
   ├────► Execution Engine
   │
   ├────► Telemetry
   │
   ├────► Events
   │
   └────► Runtime
```

Dependencia inversa prohibida:

```text
Schema Compiler
      ↓
Migration System
```

No deberá existir.

---

# 201. Pipeline completo

```text
Migration Sources
        │
        ▼
Discovery
        │
        ▼
Catalog
        │
        ├───────────────┐
        │               │
        ▼               ▼
Repository State    Checksums
        │               │
        └───────┬───────┘
                ▼
       Dependency Resolver
                │
                ▼
          Pending Set
                │
                ▼
        Operation Collection
                │
                ▼
       Compatibility Analysis
                │
                ▼
          Migration Planner
                │
                ▼
          Safety Analysis
                │
                ▼
          Migration Plan
                │
                ▼
           Acquire Lock
                │
                ▼
        Migration Executor
                │
       ┌────────┴────────┐
       ▼                 ▼
Schema Operations    Data Operations
       │                 │
       ▼                 ▼
Schema Compiler      Query Engine
       │                 │
       └────────┬────────┘
                ▼
         Execution Engine
                │
                ▼
             Database
                │
                ▼
       Repository Update
                │
                ▼
          Release Lock
```

---

# 202. Fórmula conceptual

Para migraciones pendientes:

```text
P = {M₁, M₂, ..., Mₙ}
```

el objetivo es construir:

```text
Plan(P)
```

tal que:

```text
∀ Mi ∈ P:
    DependenciesSatisfied(Mi)
    ∧ Compatible(Mi)
    ∧ SafetyPolicySatisfied(Mi)
```

y posteriormente ejecutar:

```text
Sₙ = Execute(Plan(P), S₀)
```

---

# 203. Correctness condition

Una ejecución correcta deberá satisfacer:

```text
MigrationCorrectness
=
DependencyCorrectness
∧
OperationCorrectness
∧
PlatformCompatibility
∧
RepositoryConsistency
∧
TransactionCorrectness
∧
OutcomeCertainty
```

cuando la certeza pueda demostrarse.

---

# 204. Repository invariant

Idealmente:

```text
RepositoryApplied(M)
⇒
EffectsRequiredBy(M) are committed
```

El inverso no siempre puede demostrarse tras fallos no transaccionales:

```text
EffectsPresent(M)
⇏
RepositoryApplied(M)
```

Por eso existen:

```text
PARTIALLY_APPLIED
UNKNOWN
```

---

# 205. Migration evolution equation

```text
Sₙ
=
Mₙ(
    Mₙ₋₁(
        ...
        M₂(
            M₁(S₀)
        )
    )
)
```

pero cada `M` deberá interpretarse como:

```text
planned controlled transformation
```

no como función PHP arbitraria.

---

# 206. Separación fundamental

```text
Migration Definition
        ↓
WHAT evolution is requested

Migration Planner
        ↓
HOW operations are ordered/coordinated

Schema/Query Planner
        ↓
HOW each database operation is represented

Compiler
        ↓
HOW target commands are produced

Executor
        ↓
HOW commands are performed

Repository
        ↓
WHAT migration history is known
```

---

# 207. Principio de no duplicación

Migration System no recreará:

```text
Schema AST
Query AST
SQL Compiler
Connection Pool
Transaction Engine
Telemetry Engine
```

Los compondrá.

---

# 208. Relación con Laravel-like DX

VoltStack podrá mantener ergonomía:

```php
Schema::create(...);
```

dentro de migraciones.

Pero la arquitectura interna será más cercana a:

```text
Migration DSL
     ↓
Operation Collection
     ↓
AST
     ↓
Planning
     ↓
Compatibility
     ↓
Safety
     ↓
Execution
```

Esto permite conservar:

```text
Laravel-like productivity
+
Doctrine-like explicit architecture
+
VoltStack planning/semantic infrastructure
```

---

# 209. Principio de migraciones reproducibles

Una migración deberá poder responder:

```text
What was requested?
Why is it pending?
What will be executed?
Why this order?
What capabilities are required?
What is destructive?
What can be rolled back?
What actually completed?
```

sin depender de estado global oculto.

---

# 210. Master formula

```text
Database Migration Architecture
=
Versioned Migration Definitions
+
Stable Identity
+
Discovery
+
Immutable Catalog
+
Persistent Repository
+
Checksums
+
Drift Detection
+
Dependency Graph
+
Typed Operations
+
Schema Integration
+
Data Operations
+
Planning
+
Compatibility Analysis
+
Safety Analysis
+
Explicit Transaction Boundaries
+
Coordination Locking
+
Execution
+
Outcome Certainty
+
Rollback Semantics
+
Batching
+
Backfills
+
Validation
+
Zero-Downtime Strategies
+
Telemetry
+
Extension Control
+
Persistent Runtime Isolation
```

---

# 211. Regla arquitectónica maestra

> **Una migración en VoltStack describe una evolución versionada de la base de datos; el Migration System determina qué migraciones corresponden, el Planner organiza su transformación, los subsistemas Schema y Query resuelven las operaciones concretas, y el Execution Engine realiza los efectos.**

En forma resumida:

```text
Migration describes evolution.
Catalog knows definitions.
Repository knows history.
Resolver knows dependencies.
Planner knows order and strategy.
Safety knows risk.
Schema/Query know database operations.
Executor performs effects.
Repository records the result.
```

Nunca:

```text
Migration File
    ↓
SQL
    ↓
PDO
```

como arquitectura central.

---

# 212. Bloque 9 — Migrations

Este documento abre:

```text
101_DATABASE_MIGRATION_ARCHITECTURE.md
                │
                ▼
102_DATABASE_MIGRATION_SYSTEM.md
                │
                ▼
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md
                │
                ▼
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
                │
                ▼
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
                │
                ▼
106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md
                │
                ▼
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md
                │
                ▼
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
                │
                ▼
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
                │
                ▼
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
                │
                ▼
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

La distribución conceptual será:

```text
101 Architecture
      ↓
102 Core Migration Model
      ↓
103 Discovery
      ↓
104 Persistent History
      ↓
105 Planning
      ↓
106 Execution
      ↓
107 Rollback
      ↓
108 Batches
      ↓
109 Automatic/Assisted Diff
      ↓
110 Zero-Downtime Evolution
      ↓
111 Safety
```

---

# 213. Siguiente documento

```text
102_DATABASE_MIGRATION_SYSTEM.md
```

El siguiente documento deberá profundizar específicamente en el **modelo central de una migración**, incluyendo:

```text
Migration
MigrationDefinition
MigrationId
MigrationDescriptor
MigrationMetadata
MigrationContext
MigrationOperation
MigrationOperationSet
MigrationDependency
MigrationChecksum
MigrationState
MigrationExecutionState
MigrationReversibility
MigrationPolicy
MigrationTarget
MigrationResult
```

y deberá fijar definitivamente la separación:

```text
Migration Definition
≠
Migration Discovery
≠
Migration Repository
≠
Migration Plan
≠
Migration Execution
≠
Migration Rollback
```

sobre la cual se construirán los documentos `103–111`.