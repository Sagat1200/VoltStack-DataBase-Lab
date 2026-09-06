# 107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Rollback System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 107 — Database Migration Rollback System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Rollback System` define la arquitectura responsable de planificar, validar, ejecutar y verificar la reversión controlada de migraciones previamente aplicadas.

Su objetivo no es proporcionar simplemente:

```php
public function down(): void
```

sino establecer un modelo formal capaz de responder:

> **¿Puede una migración aplicada ser revertida de forma correcta, segura y demostrable bajo el estado actual de la base de datos?**

VoltStack deberá distinguir entre:

- reversión estructural;
- recuperación de datos;
- compensación;
- forward recovery;
- rollback lógico;
- rollback físico;
- rollback transaccional;
- rollback de una migración ya aplicada;
- rollback parcial;
- rollback incierto.

---

# 2. Principio central

> **Rollback es una nueva transición versionada del estado de la base de datos; no es la ejecución mecánica de las operaciones originales en orden inverso.**

Por tanto:

```text
Rollback(M)
≠
Reverse(UP(M))
```

y:

```text
Rollback
≠
Transaction Rollback
```

---

# 3. Distinciones fundamentales

VoltStack deberá mantener:

```text
Migration Rollback
≠
Database Transaction Rollback

Rollback Definition
≠
Rollback Plan

Rollback Plan
≠
Rollback Execution

Rollback
≠
Compensation

Rollback
≠
Forward Recovery

Structural Reversal
≠
Data Recovery

Schema Reversal
≠
Semantic Restoration

Applied Migration
≠
Rollback-Safe Migration

Reversible
≠
Safe Right Now

Rollback Success
≠
Original State Proven Restored
```

---

# 4. Problema arquitectónico

Considérese:

```text
M1
ADD COLUMN email
```

Una reversión aparentemente simple sería:

```text
DROP COLUMN email
```

Pero si desde la aplicación de `M1`:

```text
email contains production data
```

entonces:

```text
DROP COLUMN email
```

puede restaurar parcialmente la estructura anterior, pero destruir información creada después.

Por tanto:

```text
Structural Inverse
≠
Semantic Inverse
```

---

# 5. Segundo ejemplo

Migración:

```text
users.name
        ↓
split
        ↓
users.first_name
users.last_name
```

La operación inversa no necesariamente puede reconstruir:

```text
name
```

exactamente.

Por tanto:

```text
Transformation(A → B)
```

no implica:

```text
∃ ExactInverse(B → A)
```

---

# 6. Posición arquitectónica

```text
Applied Migration State
        │
        ▼
Rollback Request
        │
        ▼
Rollback Target Resolution
        │
        ▼
Dependency Analysis
        │
        ▼
Reversibility Analysis
        │
        ▼
Current-State Validation
        │
        ▼
Rollback Planning
        │
        ├── Schema Reverse Operations
        ├── Data Recovery Operations
        ├── Compensation Operations
        ├── Preconditions
        ├── Safety Requirements
        └── Verification Operations
        │
        ▼
Rollback Safety Gate
        │
        ▼
Rollback Execution Plan
        │
        ▼
Migration Execution Infrastructure
        │
        ▼
Post-Rollback Verification
        │
        ▼
Repository State Transition
```

---

# 7. Relación con documentos anteriores

El sistema consume conceptos de:

```text
101 Migration Architecture
102 Migration System
103 Migration Discovery
104 Migration Repository
105 Migration Planner
106 Migration Execution
```

pero introduce una semántica adicional:

```text
Applied State
      ↓
Rollback Analysis
      ↓
Rollback Plan
```

---

# 8. Responsabilidades

El Rollback System deberá:

- resolver qué migraciones se solicitan revertir;
- verificar que están aplicadas;
- analizar dependencias posteriores;
- determinar reversibilidad;
- verificar estado físico actual;
- verificar data dependencies;
- detectar drift;
- construir rollback plan;
- clasificar riesgo;
- identificar pérdida de datos;
- exigir approvals cuando corresponda;
- definir transaction boundaries;
- ejecutar explicit rollback operations;
- ejecutar data restoration cuando exista;
- ejecutar compensations cuando proceda;
- verificar estado resultante;
- actualizar Migration Repository;
- preservar history;
- producir recovery information ante fallos.

---

# 9. No responsabilidades

Rollback System no deberá:

```text
inventar un down() automáticamente
suponer que DROP revierte CREATE de forma segura
suponer que ADD revierte DROP
restaurar datos inexistentes
eliminar migration history
ignorar dependencias
ignorar drift
ignorar aplicaciones desplegadas
ocultar irreversible migrations
convertir UNKNOWN en REVERSIBLE
```

---

# 10. Modelo general

```text
RollbackRequest
      ↓
RollbackResolver
      ↓
RollbackCandidateSet
      ↓
DependencyAnalyzer
      ↓
ReversibilityAnalyzer
      ↓
RollbackPlanner
      ↓
RollbackSafetyAnalyzer
      ↓
RollbackExecutionPlan
      ↓
RollbackExecutor
      ↓
RollbackResult
```

---

# 11. RollbackRequest

Se propone:

```php
final readonly class MigrationRollbackRequest
{
    public function __construct(
        public MigrationRollbackSelector $selector,
        public MigrationRollbackPolicy $policy,
        public MigrationExecutionTarget $target,
        public MigrationApprovalSet $approvals,
        public Deadline $deadline,
        public CancellationToken $cancellation,
    ) {}
}
```

---

# 12. Selección

`MigrationRollbackSelector` puede representar:

```text
LAST_MIGRATION
LAST_BATCH
STEPS
TO_MIGRATION
SPECIFIC_MIGRATION
SPECIFIC_SET
MODULE
PACKAGE
ALL
```

---

# 13. Selección ≠ plan

Solicitar:

```text
rollback last 3
```

no significa:

```text
take last 3 rows and execute down()
```

Primero debe analizarse:

```text
dependency closure
reversibility
current state
safety
```

---

# 14. Rollback candidate

```php
final readonly class MigrationRollbackCandidate
{
    public function __construct(
        public MigrationRecord $record,
        public MigrationDefinition $definition,
        public MigrationReversibilityReport $reversibility,
    ) {}
}
```

---

# 15. MigrationDefinition availability

Una migración aplicada puede existir en repository pero su código ya no estar disponible.

Estado:

```text
APPLIED
+
DEFINITION_MISSING
```

Esto deberá ser first-class.

---

# 16. Missing definition

No debe asumirse:

```text
repository record
→ enough information to rollback
```

Puede requerir:

```text
archived migration definition
serialized rollback contract
manual recovery
```

---

# 17. Definiciones aplicadas inmutables

Una migration aplicada no debería ser modificada.

Si:

```text
stored checksum ≠ current definition checksum
```

el rollback deberá detenerse por defecto.

---

# 18. Modified applied migration

Resultado:

```text
MODIFIED_APPLIED_MIGRATION
```

No utilizar silenciosamente el nuevo `down()`.

---

# 19. Rollback identity

Cada rollback tendrá:

```text
MigrationRollbackExecutionId
```

distinto de:

```text
MigrationExecutionId
MigrationId
MigrationBatchId
```

---

# 20. Reversibility model

Se define:

```text
MigrationReversibility
```

con estados:

```text
REVERSIBLE
CONDITIONALLY_REVERSIBLE
PARTIALLY_REVERSIBLE
IRREVERSIBLE
UNKNOWN
```

---

# 21. REVERSIBLE

Significa:

> Existe una transición explícita capaz de restaurar el contrato requerido bajo las condiciones declaradas.

No significa necesariamente:

```text
bit-for-bit identical database
```

---

# 22. CONDITIONALLY_REVERSIBLE

Ejemplo:

```text
DROP new_column
```

solo si:

```text
column contains no required business data
```

o:

```text
application version no longer depends on it
```

---

# 23. PARTIALLY_REVERSIBLE

Ejemplo:

```text
schema can be restored
but original transformed data cannot
```

---

# 24. IRREVERSIBLE

Ejemplos:

```text
DROP COLUMN containing data
destructive aggregation
hashing without original value
irreversible anonymization
lossy type conversion
```

---

# 25. UNKNOWN

Se utiliza cuando falta evidencia suficiente.

Regla:

```text
UNKNOWN
≠
REVERSIBLE
```

---

# 26. Reversibility dimensions

Una migración deberá analizarse al menos en:

```text
STRUCTURAL
DATA
SEMANTIC
OPERATIONAL
APPLICATION
TRANSACTIONAL
```

---

# 27. Reversibility report

```php
final readonly class MigrationReversibilityReport
{
    public function __construct(
        public MigrationReversibility $overall,
        public StructuralReversibility $structural,
        public DataReversibility $data,
        public SemanticReversibility $semantic,
        public OperationalReversibility $operational,
        public ApplicationCompatibility $application,
        public array $requirements,
        public array $risks,
        public array $evidence,
    ) {}
}
```

---

# 28. Structural reversibility

Pregunta:

> ¿Puede reconstruirse el esquema estructural requerido?

Ejemplo:

```text
ADD TABLE
→ DROP TABLE
```

estructuralmente reversible.

Pero:

```text
DROP TABLE
→ CREATE TABLE
```

no implica restaurar sus datos.

---

# 29. Data reversibility

Pregunta:

> ¿Pueden restaurarse los datos relevantes?

Puede requerir:

```text
backup
shadow column
archive table
history table
explicit reverse transform
external recovery source
```

---

# 30. Semantic reversibility

Pregunta:

> ¿El estado restaurado conserva el significado original?

Ejemplo:

```text
decimal(18,6)
→ float
→ decimal(18,6)
```

puede ser estructuralmente posible pero semánticamente los valores pueden haber perdido precisión.

---

# 31. Operational reversibility

Pregunta:

> ¿Puede realizarse la reversión bajo las capacidades y condiciones operacionales actuales?

Considera:

```text
locks
table size
DDL transaction support
maintenance window
replica lag
runtime availability
storage
timeout
```

---

# 32. Application compatibility

Una base de datos puede ser técnicamente reversible mientras la aplicación actual requiere el nuevo schema.

Ejemplo:

```text
Application v4 requires users.normalized_email
```

Rollback:

```text
DROP users.normalized_email
```

podría romper producción.

---

# 33. Database rollback ≠ deployment rollback

VoltStack Database deberá permitir integrar:

```text
DeploymentCompatibilityProvider
```

sin depender del deployment system.

---

# 34. Rollback contract

Una migration puede declarar explícitamente:

```text
UP contract
DOWN contract
```

---

# 35. Ejemplo DSL

```php
return Migration::define('2026_09_06_create_profiles')
    ->up(function (MigrationDefinitionContext $m) {
        $m->schema()->create('profiles', function (TableBlueprint $table) {
            // ...
        });
    })
    ->down(function (MigrationDefinitionContext $m) {
        $m->schema()->drop('profiles');
    });
```

Pero la presencia de `down()`:

```text
does not prove safety
```

---

# 36. Explicit down

Regla recomendada:

> VoltStack no generará automáticamente operaciones `down` destructivas a partir de `up`.

---

# 37. Generated inverse candidates

El framework podría generar:

```text
RollbackSuggestion
```

para DX.

Pero:

```text
RollbackSuggestion
≠
RollbackDefinition
```

y nunca será ejecutable automáticamente sin aceptación explícita.

---

# 38. Reverse operation taxonomy

Se proponen:

```text
SchemaRollbackOperation
DataRollbackOperation
CompensationOperation
ValidationOperation
BarrierOperation
BackupRestoreOperation
ExtensionRollbackOperation
```

---

# 39. Schema rollback operation

Ejemplos:

```text
DROP newly created index
DROP newly added constraint
rename column back
restore previous column type
drop newly added column
recreate dropped object
```

---

# 40. Data rollback operation

Ejemplos:

```text
restore values from shadow column
restore rows from archive
reverse deterministic transformation
restore from migration snapshot
```

---

# 41. Compensation

Compensation no necesariamente restaura el estado anterior.

Ejemplo:

```text
original:
delete invalid records

compensation:
recreate placeholder records
```

Esto puede reparar el sistema, pero:

```text
Compensation ≠ Exact Rollback
```

---

# 42. Forward recovery

A veces es más seguro:

```text
M42 fails partially
      ↓
M42_FIX
```

que intentar:

```text
rollback M42
```

---

# 43. Rollback strategy

Se propone:

```text
EXACT_RESTORE
STRUCTURAL_RESTORE
COMPENSATE
FORWARD_RECOVERY
MANUAL_RECOVERY
```

---

# 44. Strategy selection

Debe realizarse explícitamente por:

```text
Rollback Planner / Recovery Policy
```

No por el Executor.

---

# 45. Dependency analysis

Si:

```text
M1 → M2 → M3
```

y se solicita rollback de `M1`:

```text
M2
M3
```

deben analizarse.

---

# 46. Reverse dependency closure

Formalmente:

```text
RollbackClosure(M)
=
M
∪
AppliedDependents(M)
```

cuando esos dependents no puedan permanecer válidos sin `M`.

---

# 47. Dependency-aware rollback

Ejemplo:

```text
M1 create users
M2 add users.email
M3 create index users.email
```

No puede ejecutarse:

```text
rollback M1
```

dejando M2/M3 aplicadas.

---

# 48. Rollback order

Para DAG:

```text
UP:
A → B → C
```

rollback normalmente:

```text
C → B → A
```

pero:

> El orden debe derivarse del dependency graph, no simplemente del timestamp descendente.

---

# 49. Reverse topological order

Formalmente:

```text
RollbackOrder
=
ReverseTopologicalOrder(
    SelectedAppliedSubgraph
)
```

sujeto a rollback-specific dependencies.

---

# 50. Rollback-specific dependency

Puede existir una dependencia diferente de la de `UP`.

Ejemplo:

```text
restore archived data
```

antes de:

```text
drop archive structure
```

---

# 51. Rollback graph

Se propone:

```text
MigrationRollbackGraph
```

separado del MigrationGraph original.

---

# 52. Rollback graph nodes

```text
MigrationRollbackNode
RollbackOperationNode
RollbackBarrierNode
RollbackValidationNode
```

---

# 53. Rollback graph edge

Puede expresar:

```text
MUST_RUN_BEFORE
MUST_RUN_AFTER
REQUIRES
BLOCKS
```

---

# 54. Cycle

Un ciclo irresoluble produce:

```text
MigrationRollbackDependencyCycleException
```

---

# 55. Current-state verification

Rollback nunca debe basarse únicamente en:

```text
repository says APPLIED
```

cuando la reversión depende del estado físico.

---

# 56. Observed schema

Puede utilizar:

```text
Schema Introspection
```

para comparar:

```text
ExpectedCurrentState
vs
ObservedCurrentState
```

---

# 57. Drift before rollback

Si existe drift:

```text
Expected schema after M42
≠
Observed schema
```

el rollback plan puede quedar:

```text
STALE
UNSAFE
UNKNOWN
```

---

# 58. Drift ≠ automatic repair

Rollback no deberá reparar drift automáticamente.

---

# 59. Data preconditions

Ejemplo:

```text
DROP normalized_email
```

puede requerir:

```text
all data recoverable from email
```

---

# 60. Data validation

Puede ejecutar queries read-only:

```text
SELECT COUNT(*)
```

mediante Query Engine.

---

# 61. Runtime precondition

Ejemplo:

```text
No rows exist that depend exclusively on the new representation
```

---

# 62. Preconditions are first-class

```text
RollbackPrecondition
```

podrá ser:

```text
SchemaRollbackCondition
DataRollbackCondition
RepositoryRollbackCondition
ApplicationRollbackCondition
BackupRollbackCondition
CapabilityRollbackCondition
ApprovalRollbackCondition
```

---

# 63. Unknown precondition

Default:

```text
UNKNOWN
→
BLOCK
```

para operaciones destructivas.

---

# 64. Rollback planning

Entrada:

```text
RollbackRequest
AppliedRepositorySnapshot
MigrationDefinitions
ObservedSchema?
CapabilitySnapshot
ApplicationCompatibility?
Policy
```

Salida:

```text
MigrationRollbackPlan
```

---

# 65. MigrationRollbackPlan

```php
final readonly class MigrationRollbackPlan
{
    public function __construct(
        public MigrationRollbackPlanId $id,
        public MigrationRollbackTargetSet $targets,
        public MigrationRollbackGraph $graph,
        public MigrationRollbackSafetyReport $safety,
        public MigrationRollbackTransactionPlan $transactions,
        public MigrationRollbackVerificationPlan $verification,
        public MigrationRollbackPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 66. Rollback plan immutable

Una vez aceptado:

```text
MigrationRollbackPlan
```

será immutable.

---

# 67. Plan vs execution

```text
Rollback Planner decides.
Rollback Executor coordinates.
```

---

# 68. Rollback operation

```php
interface MigrationRollbackOperation
{
    public function id(): MigrationRollbackOperationId;

    public function kind(): MigrationRollbackOperationKind;
}
```

---

# 69. Operation kinds

```text
SCHEMA
DATA
RESTORE
COMPENSATION
VALIDATION
BARRIER
REPOSITORY
EXTENSION
```

---

# 70. Explicit irreversibility

Migration API deberá permitir:

```php
->irreversible('Original data was permanently anonymized')
```

---

# 71. Conditional reversibility

Ejemplo:

```php
->rollbackRequires(
    RollbackCondition::backupAvailable('users-pre-normalization')
)
```

---

# 72. Reversibility metadata

Puede declararse:

```text
structural = REVERSIBLE
data = REQUIRES_BACKUP
semantic = CONDITIONALLY_REVERSIBLE
```

---

# 73. Declaration ≠ proof

Una migration que declara:

```text
REVERSIBLE
```

no obliga al analyzer a aceptarlo si el estado contradice la declaración.

---

# 74. Safety model

Se propone:

```text
SAFE
SAFE_WITH_CONDITIONS
REQUIRES_APPROVAL
HIGH_RISK
UNSAFE
UNKNOWN
```

---

# 75. Reversibility ≠ safety

Ejemplo:

```text
reversible = YES
```

pero:

```text
requires 4-hour table lock
```

puede ser:

```text
HIGH_RISK
```

---

# 76. Safety dimensions

```text
DATA_LOSS
LOCKING
AVAILABILITY
APPLICATION_COMPATIBILITY
REPLICATION
TRANSACTIONALITY
RECOVERY
PERFORMANCE
UNKNOWN_STATE
```

---

# 77. Data loss risk

Se propone:

```text
NONE
POTENTIAL
LIKELY
CERTAIN
UNKNOWN
```

---

# 78. Destructive rollback

Ejemplo:

```text
DROP TABLE feature_data
```

puede requerir:

```text
explicit approval
backup evidence
maintenance policy
```

---

# 79. Approval

`MigrationApprovalSet` podrá contener:

```text
ALLOW_DATA_LOSS
ALLOW_LONG_LOCK
ALLOW_NON_ATOMIC_ROLLBACK
ALLOW_APPLICATION_INCOMPATIBILITY
ALLOW_IRREVERSIBLE_COMPENSATION
```

según policy.

---

# 80. Approval ≠ correctness

Approval puede permitir riesgo.

No puede convertir:

```text
semantically impossible rollback
```

en correcto.

---

# 81. Transaction model

Rollback puede tener:

```text
ATOMIC
MULTI_TRANSACTION
NON_ATOMIC
PLATFORM_DEPENDENT
```

---

# 82. Transaction rollback vs migration rollback

Ejemplo:

```text
BEGIN
DROP INDEX
ADD OLD INDEX
COMMIT
```

es:

```text
Migration Rollback
```

ejecutado dentro de:

```text
Database Transaction
```

---

# 83. Transaction failure

Si falla antes de commit y rollback transaccional es cierto:

```text
Migration rollback effects may be absent
```

pero eso no significa que la migration original haya sido revertida.

---

# 84. Multi-step rollback

```text
restore data
↓
remove new constraint
↓
restore old type
↓
remove new column
```

puede cruzar múltiples transactions.

---

# 85. Non-transactional DDL

El sistema deberá modelar:

```text
partial rollback risk
```

explícitamente.

---

# 86. Rollback execution

Se propone:

```php
interface MigrationRollbackExecutor
{
    public function execute(
        MigrationRollbackPlan $plan,
        MigrationRollbackExecutionRequest $request
    ): MigrationRollbackResult;
}
```

---

# 87. Reutilización del Migration Execution System

Siempre que sea posible:

```text
Rollback Executor
       ↓
Migration Execution Infrastructure
       ├── locks
       ├── deadlines
       ├── cancellation
       ├── transactions
       ├── schema execution
       ├── query execution
       ├── retry
       └── telemetry
```

---

# 88. No second execution engine

VoltStack no deberá crear un segundo motor completamente independiente para rollback.

---

# 89. Rollback execution lifecycle

```text
RECEIVED
↓
VALIDATING
↓
LOCKING
↓
PREPARING
↓
ROLLING_BACK
↓
VERIFYING
↓
RECORDING
↓
SUCCEEDED
```

Alternativas:

```text
FAILED
PARTIALLY_ROLLED_BACK
CANCELLED
UNKNOWN_OUTCOME
REQUIRES_RECOVERY
```

---

# 90. Repository state

Repository no deberá simplemente borrar el registro aplicado.

---

# 91. Preserve history

Se debe conservar:

```text
Migration Applied
Migration Rolled Back
Migration Re-Applied
```

como history.

---

# 92. Current state vs history

Se recomienda separar:

```text
CurrentMigrationState
```

de:

```text
MigrationExecutionHistory
```

---

# 93. Example history

```text
M42
├── APPLY E100 → SUCCESS
├── ROLLBACK R101 → SUCCESS
└── APPLY E102 → SUCCESS
```

---

# 94. Repository transition

Conceptualmente:

```text
APPLIED
↓
ROLLBACK_RUNNING
↓
ROLLED_BACK
```

sin destruir history.

---

# 95. Durable vs transient state

Puede ser preferible mantener:

```text
current durable state
```

y:

```text
execution run state
```

separados para evitar que un crash deje semántica ambigua.

---

# 96. Mark rolled back

Solo cuando:

```text
all required rollback operations succeeded
+
required verification succeeded
+
repository transition persisted
```

---

# 97. Repository failure after rollback

Caso:

```text
physical rollback succeeded
repository update failed
```

Resultado:

```text
REQUIRES_RECOVERY
```

No volver a ejecutar rollback ciegamente.

---

# 98. Repository failure before physical rollback

Es diferente.

Outcome puede ser:

```text
CERTAIN_FAILURE
```

sin efectos físicos.

---

# 99. Rollback result

```php
final readonly class MigrationRollbackResult
{
    public function __construct(
        public MigrationRollbackExecutionId $executionId,
        public MigrationRollbackStatus $status,
        public MigrationOutcomeCertainty $certainty,
        public MigrationRollbackSummary $summary,
        public ?MigrationRollbackFailureReport $failure,
        public MigrationRollbackRecoveryDescriptor $recovery,
    ) {}
}
```

---

# 100. Rollback statuses

```text
SUCCEEDED
FAILED
PARTIALLY_ROLLED_BACK
CANCELLED
TIMED_OUT
UNKNOWN_OUTCOME
BLOCKED
REQUIRES_RECOVERY
```

---

# 101. Partial rollback

Definición:

```text
some rollback effects committed
+
rollback contract incomplete
```

---

# 102. Partial rollback example

```text
Drop new index       ✓
Restore old column   ✓
Restore archived data ✗
```

Resultado:

```text
PARTIALLY_ROLLED_BACK
```

---

# 103. Partial rollback ≠ original migration still simply applied

El estado puede ser híbrido.

Debe modelarse.

---

# 104. Hybrid schema state

Después de partial rollback:

```text
State ≠ BeforeMigration
State ≠ AfterMigration
```

sino:

```text
Intermediate/Recovery State
```

---

# 105. Unknown rollback outcome

Ejemplo:

```text
ALTER TABLE ...
COMMIT sent
connection lost
```

Resultado:

```text
UNKNOWN_OUTCOME
```

---

# 106. No blind retry

Si outcome es UNKNOWN:

```text
verify
↓
reconcile
↓
decide
```

antes de retry.

---

# 107. Rollback retry

Debe satisfacer:

```text
failure retryable
outcome sufficiently known
operation replayable
transaction state compatible
rollback operation idempotent or reconcilable
deadline available
retry budget available
```

---

# 108. Rollback of rollback

No se modelará como:

```text
rollback(rollback(M))
```

para reaplicar.

Debe utilizarse:

```text
apply M
```

mediante el migration planning normal.

---

# 109. Re-apply

Después de rollback exitoso:

```text
M state = ROLLED_BACK
```

puede volver a ser:

```text
PENDING
```

según repository/query semantics.

---

# 110. Re-apply checksum

Debe utilizar la misma immutable migration definition esperada.

Si cambió:

```text
ModifiedAppliedMigrationException
```

o una nueva migration.

---

# 111. Data backup integration

Rollback podrá declarar:

```text
BackupRequirement
```

pero Database Migration Core no necesita implementar todo Backup System.

---

# 112. Backup evidence

Puede contener:

```text
backup ID
creation timestamp
source fingerprint
coverage
integrity status
restore capability
```

---

# 113. Backup exists ≠ restorable

Rollback deberá distinguir:

```text
BackupAvailable
≠
BackupVerified
≠
RestoreTested
```

---

# 114. Backup freshness

Una copia anterior a cambios posteriores puede no ser adecuada.

---

# 115. Shadow data strategy

Migraciones zero-downtime pueden conservar temporalmente:

```text
old_column
shadow_table
mapping_table
```

para facilitar rollback.

---

# 116. Contract phase risk

En patrón:

```text
EXPAND
↓
MIGRATE
↓
VALIDATE
↓
CONTRACT
```

antes de CONTRACT puede ser relativamente sencillo revertir.

Después:

```text
rollback may become significantly harder
```

---

# 117. Rollback window

Se propone:

```text
RollbackWindow
```

---

# 118. Rollback window example

```text
M42 reversible until:
old_email column is dropped
```

Después:

```text
data reversibility requires backup
```

---

# 119. Temporal reversibility

Por tanto:

```text
Reversibility(M, t₁)
≠
Reversibility(M, t₂)
```

---

# 120. State-dependent reversibility

Más precisamente:

```text
Reversibility
=
f(MigrationDefinition,
  CurrentSchema,
  CurrentData,
  RepositoryState,
  ApplicationState,
  Capabilities)
```

---

# 121. Rollback barrier

Puede requerir:

```text
application version <= compatible version
traffic drained
background jobs paused
replicas caught up
backup completed
```

---

# 122. Barrier semantics

```text
Barrier satisfied
```

es requisito de ejecución, no una sugerencia.

---

# 123. Deployment integration

Podrá existir:

```php
interface MigrationRollbackApplicationStateProvider
{
    public function currentState(): ApplicationDatabaseCompatibilityState;
}
```

---

# 124. Multitenancy

Core no dependerá del paquete Multitenancy.

Pero deberá soportar:

```text
RollbackScopeIdentity
```

para que integración futura pueda representar:

```text
tenant
database
schema
partition
```

---

# 125. Tenant rollback

Nunca asumir:

```text
rollback one tenant
=
rollback all tenants
```

---

# 126. Multi-target rollback

Si migration afectó:

```text
Database A
Database B
```

no asumir atomicidad distribuida.

---

# 127. Distributed rollback

Debe modelar:

```text
A rolled back
B failed
```

como:

```text
PARTIAL
```

---

# 128. No hidden 2PC

VoltStack no inventará distributed transaction atomicity si no existe.

---

# 129. Rollback locks

Rollback utilizará coordination locking igual que migration execution.

---

# 130. Lock scope

Puede ser:

```text
database
schema
migration namespace
tenant
target set
```

según adapter/policy.

---

# 131. Lock lost

```text
lock lost
→ stop at safest boundary
```

y producir recovery state si existen efectos.

---

# 132. Cancellation

Rollback será cancelable solo dentro de los límites de las operations.

---

# 133. Cancellation ≠ restore

Cancelar un rollback parcial no restaura automáticamente el estado post-migration.

---

# 134. Deadline

Rollback respetará deadline global.

---

# 135. Timeout

Timeout puede dejar:

```text
PARTIAL
UNKNOWN
```

según operation outcome.

---

# 136. Post-rollback verification

Después de ejecutar rollback:

```text
verify resulting state
```

---

# 137. Verification dimensions

```text
schema
constraints
indexes
data invariants
repository state
application compatibility
```

---

# 138. Expected rollback state

Se propone:

```text
ExpectedRollbackState
```

que puede ser:

```text
exact previous schema
semantic previous schema
explicit target state
compensated safe state
```

---

# 139. Exact restoration

Cuando exista snapshot previo:

```text
ObservedAfterRollback
≈
ExpectedBeforeMigration
```

según comparison profile.

---

# 140. Semantic restoration

No siempre se necesita exact physical equality.

Ejemplo:

```text
index generated name differs
```

pero semantics equivalentes.

---

# 141. Verification failure

Si rollback commands terminan pero verification falla:

```text
Rollback != SUCCEEDED
```

---

# 142. Drift after rollback

Debe reportarse.

No corregirse silenciosamente.

---

# 143. Forward recovery recommendation

Cuando rollback no sea seguro:

```text
MigrationRollbackDecision
=
FORWARD_RECOVERY_RECOMMENDED
```

---

# 144. Decision model

Se propone:

```text
ROLLBACK_ALLOWED
ROLLBACK_ALLOWED_WITH_CONDITIONS
ROLLBACK_REQUIRES_APPROVAL
ROLLBACK_BLOCKED
FORWARD_RECOVERY_RECOMMENDED
MANUAL_RECOVERY_REQUIRED
UNKNOWN
```

---

# 145. Rollback decision ≠ reversibility

Una migration puede ser reversible pero:

```text
ROLLBACK_BLOCKED
```

por application state.

---

# 146. Explainability

Toda decisión debe incluir:

```text
reason
evidence
risk
blocking conditions
required approvals
recommended action
```

---

# 147. Ejemplo diagnostic

```text
Rollback blocked.

Migration:
app:2026_09_06_remove_legacy_email

Reason:
The migration permanently removed legacy_email.

Data reversibility:
IRREVERSIBLE

Backup evidence:
NONE

Structural reversibility:
REVERSIBLE

Recommended action:
Use forward recovery or restore from an external verified backup.
```

---

# 148. Otro diagnostic

```text
Rollback conditionally allowed.

Migration:
app:2026_09_06_add_normalized_email

Structural:
REVERSIBLE

Data:
REVERSIBLE

Application compatibility:
UNSATISFIED

Current application:
v4.2

Required database contract:
normalized_email must exist

Action:
Deploy a compatible application version before rollback.
```

---

# 149. Security

Rollback es una operación privilegiada.

Puede ser incluso más peligrosa que `UP`.

---

# 150. Security requirements

Debe soportar:

```text
authorization context
approval context
target verification
raw operation policy
data-loss acknowledgement
audit trail
```

---

# 151. Raw rollback SQL

Debe ser:

```text
explicit
trusted
platform-scoped
auditable
redacted
```

---

# 152. No secret leakage

No registrar:

```text
credentials
backup encryption keys
raw sensitive values
```

---

# 153. Audit

Registrar:

```text
who requested
what target
which migrations
which plan fingerprint
which approvals
when
result
certainty
risks acknowledged
```

cuando Security/Audit integration esté activa.

---

# 154. Telemetry

Métricas posibles:

```text
rollback duration
lock wait
migrations rolled back
operations executed
rows restored
retry count
verification duration
partial rollback count
unknown outcome count
irreversible rollback blocks
```

---

# 155. Events

Se proponen:

```text
MigrationRollbackRequested
MigrationRollbackPlanning
MigrationRollbackPlanned
MigrationRollbackBlocked
MigrationRollbackStarting
MigrationRollbackMigrationStarting
MigrationRollbackOperationStarting
MigrationRollbackOperationCompleted
MigrationRollbackOperationFailed
MigrationRollbackVerificationStarting
MigrationRollbackVerificationCompleted
MigrationRollbackCompleted
MigrationRollbackFailed
MigrationRollbackRecoveryRequired
```

---

# 156. Event safety

Listeners no podrán:

```text
modify rollback graph
bypass safety
mark repository rolled back
forge success
```

---

# 157. Persistent runtime

Shared:

```text
immutable rollback services
frozen rule registry
configuration
```

Operation-scoped:

```text
RollbackExecutionSession
locks
connections
transactions
checkpoints
failure state
```

---

# 158. FrankenPHP

Rollback state deberá destruirse al finalizar la operación.

---

# 159. RoadRunner

No podrá filtrarse entre jobs.

---

# 160. OpenSwoole

No podrá compartirse entre coroutines.

---

# 161. No static rollback context

Prohibido:

```php
MigrationRollback::$current;
```

---

# 162. Extension model

Se propone:

```text
MigrationRollbackExtension
```

para:

```text
custom reversibility rule
custom backup evidence provider
custom application compatibility provider
custom compensation operation
custom verification strategy
```

---

# 163. Frozen extension registry

Las extensiones deberán registrarse durante bootstrap y congelarse.

---

# 164. Extension safety

Una extensión no podrá:

```text
turn UNKNOWN into SAFE without evidence
hide data loss
bypass required approvals
forge verification success
modify repository outside protocol
```

---

# 165. Budgets

Rollback puede definir:

```text
max runtime
max lock wait
max operations
max restored rows
max retries
max affected rows
max validation queries
```

---

# 166. Budget exhaustion

No significa:

```text
rollback failed cleanly
```

Puede producir:

```text
PARTIAL
```

o:

```text
UNKNOWN
```

según effects.

---

# 167. Testing architecture

Deberá probar:

- reversible migrations;
- irreversible migrations;
- conditional reversibility;
- missing definitions;
- modified applied migration;
- dependency closure;
- reverse ordering;
- dependency cycles;
- schema drift;
- data preconditions;
- application incompatibility;
- backup availability;
- partial rollback;
- unknown outcome;
- transaction rollback;
- non-transactional DDL;
- repository write failure;
- lock loss;
- cancellation;
- timeout;
- retries;
- verification failures;
- multi-target partial rollback;
- persistent-runtime isolation.

---

# 168. Fault injection

Puntos críticos:

```text
before first rollback operation
after destructive DDL
after data restoration
before commit
after commit send
before repository transition
during repository transition
during verification
during lock release
```

---

# 169. Critical crash scenario

```text
Physical rollback succeeds
↓
Process crashes
↓
Repository still says APPLIED
```

Recovery deberá detectar:

```text
repository/schema disagreement
```

antes de repetir.

---

# 170. Second crash scenario

```text
Repository run = ROLLBACK_RUNNING
↓
process dies
```

No asumir:

```text
rollback failed
```

Debe reconciliarse.

---

# 171. Conformance suite

Custom rollback providers deberán pasar:

```text
MigrationRollbackConformanceSuite
```

---

# 172. Namespace

Se propone:

```text
VoltStack\Quantum\Database\Migration\Rollback
```

---

# 173. Estructura propuesta

```text
Migration/
└── Rollback/
    ├── Contract/
    │   ├── MigrationRollbackService.php
    │   ├── MigrationRollbackPlanner.php
    │   ├── MigrationRollbackExecutor.php
    │   ├── MigrationReversibilityAnalyzer.php
    │   └── MigrationRollbackVerifier.php
    │
    ├── Request/
    │   ├── MigrationRollbackRequest.php
    │   ├── MigrationRollbackSelector.php
    │   └── MigrationRollbackPolicy.php
    │
    ├── Identity/
    │   ├── MigrationRollbackPlanId.php
    │   ├── MigrationRollbackExecutionId.php
    │   └── MigrationRollbackOperationId.php
    │
    ├── Candidate/
    │   ├── MigrationRollbackCandidate.php
    │   └── MigrationRollbackCandidateSet.php
    │
    ├── Reversibility/
    │   ├── MigrationReversibility.php
    │   ├── MigrationReversibilityReport.php
    │   ├── StructuralReversibility.php
    │   ├── DataReversibility.php
    │   ├── SemanticReversibility.php
    │   ├── OperationalReversibility.php
    │   ├── MigrationReversibilityAnalyzer.php
    │   └── MigrationReversibilityRuleRegistry.php
    │
    ├── Dependency/
    │   ├── MigrationRollbackGraph.php
    │   ├── MigrationRollbackNode.php
    │   ├── MigrationRollbackEdge.php
    │   └── MigrationRollbackDependencyAnalyzer.php
    │
    ├── Planning/
    │   ├── DefaultMigrationRollbackPlanner.php
    │   ├── MigrationRollbackPlan.php
    │   ├── MigrationRollbackTargetSet.php
    │   └── MigrationRollbackPlanningContext.php
    │
    ├── Operation/
    │   ├── MigrationRollbackOperation.php
    │   ├── SchemaRollbackOperation.php
    │   ├── DataRollbackOperation.php
    │   ├── CompensationOperation.php
    │   ├── ValidationRollbackOperation.php
    │   ├── BarrierRollbackOperation.php
    │   └── ExtensionRollbackOperation.php
    │
    ├── Condition/
    │   ├── MigrationRollbackCondition.php
    │   ├── SchemaRollbackCondition.php
    │   ├── DataRollbackCondition.php
    │   ├── ApplicationRollbackCondition.php
    │   └── BackupRollbackCondition.php
    │
    ├── Safety/
    │   ├── MigrationRollbackSafetyAnalyzer.php
    │   ├── MigrationRollbackSafetyReport.php
    │   ├── MigrationRollbackRisk.php
    │   └── MigrationRollbackDecision.php
    │
    ├── Backup/
    │   ├── MigrationBackupRequirement.php
    │   ├── MigrationBackupEvidence.php
    │   └── MigrationBackupEvidenceProvider.php
    │
    ├── Application/
    │   ├── MigrationRollbackApplicationStateProvider.php
    │   └── ApplicationDatabaseCompatibilityState.php
    │
    ├── Transaction/
    │   └── MigrationRollbackTransactionPlan.php
    │
    ├── Execution/
    │   ├── DefaultMigrationRollbackExecutor.php
    │   ├── MigrationRollbackExecutionRequest.php
    │   ├── MigrationRollbackExecutionSession.php
    │   └── MigrationRollbackStatus.php
    │
    ├── Verification/
    │   ├── MigrationRollbackVerificationPlan.php
    │   ├── MigrationRollbackVerifier.php
    │   ├── MigrationRollbackVerificationResult.php
    │   └── ExpectedRollbackState.php
    │
    ├── Result/
    │   ├── MigrationRollbackResult.php
    │   └── MigrationRollbackSummary.php
    │
    ├── Recovery/
    │   ├── MigrationRollbackRecoveryDescriptor.php
    │   └── MigrationRollbackRecoveryEvidence.php
    │
    ├── Fingerprint/
    │   └── MigrationRollbackPlanFingerprint.php
    │
    ├── Extension/
    │   ├── MigrationRollbackExtension.php
    │   └── MigrationRollbackExtensionRegistry.php
    │
    ├── Telemetry/
    │   └── MigrationRollbackTelemetry.php
    │
    └── Exception/
        └── ...
```

---

# 174. Error hierarchy

```text
DatabaseMigrationRollbackException
├── InvalidMigrationRollbackRequestException
├── MigrationRollbackTargetNotAppliedException
├── MigrationRollbackDefinitionMissingException
├── ModifiedAppliedMigrationRollbackException
├── MigrationRollbackDependencyException
│   └── MigrationRollbackDependencyCycleException
├── MigrationRollbackReversibilityException
├── IrreversibleMigrationException
├── MigrationRollbackConditionException
├── MigrationRollbackDriftException
├── MigrationRollbackPlanningException
├── MigrationRollbackSafetyException
├── UnsafeMigrationRollbackException
├── MigrationRollbackApprovalRequiredException
├── MigrationRollbackBackupException
├── MigrationRollbackApplicationCompatibilityException
├── MigrationRollbackExecutionException
├── MigrationRollbackPartialExecutionException
├── MigrationRollbackOutcomeUnknownException
├── MigrationRollbackVerificationException
├── MigrationRollbackRepositoryException
├── MigrationRollbackRecoveryException
├── MigrationRollbackBudgetExceededException
├── MigrationRollbackExtensionException
└── MigrationRollbackInvariantException
```

---

# 175. Invariantes

## DB-MIGRATION-ROLLBACK-001
Rollback será distinto de transaction rollback.

## DB-MIGRATION-ROLLBACK-002
Rollback no será definido como inversión mecánica de `UP`.

## DB-MIGRATION-ROLLBACK-003
Structural reversal será distinto de data restoration.

## DB-MIGRATION-ROLLBACK-004
Rollback será distinto de compensation.

## DB-MIGRATION-ROLLBACK-005
Rollback será distinto de forward recovery.

## DB-MIGRATION-ROLLBACK-006
Applied no implicará reversible.

## DB-MIGRATION-ROLLBACK-007
Reversible no implicará currently safe.

## DB-MIGRATION-ROLLBACK-008
Rollback definition será distinta de rollback plan.

## DB-MIGRATION-ROLLBACK-009
Rollback plan será distinto de rollback execution.

## DB-MIGRATION-ROLLBACK-010
Rollback plan será immutable.

## DB-MIGRATION-ROLLBACK-011
Rollback execution state será operation-scoped.

## DB-MIGRATION-ROLLBACK-012
Rollback ID será distinto de Migration ID.

## DB-MIGRATION-ROLLBACK-013
Rollback ID será distinto de original execution ID.

## DB-MIGRATION-ROLLBACK-014
Migration definition checksum será validado.

## DB-MIGRATION-ROLLBACK-015
Modified applied migration bloqueará rollback por default.

## DB-MIGRATION-ROLLBACK-016
Missing definition será first-class state.

## DB-MIGRATION-ROLLBACK-017
Missing definition no será reconstruida desde filename.

## DB-MIGRATION-ROLLBACK-018
Rollback target selection será distinta del final plan.

## DB-MIGRATION-ROLLBACK-019
Dependency closure será analizada.

## DB-MIGRATION-ROLLBACK-020
Dependents no serán ignorados.

## DB-MIGRATION-ROLLBACK-021
Rollback order será dependency-driven.

## DB-MIGRATION-ROLLBACK-022
Timestamp descending no será suficiente como semantic ordering.

## DB-MIGRATION-ROLLBACK-023
Rollback graph podrá diferir del UP graph.

## DB-MIGRATION-ROLLBACK-024
Rollback dependency cycle será error.

## DB-MIGRATION-ROLLBACK-025
Reversibility será multidimensional.

## DB-MIGRATION-ROLLBACK-026
UNKNOWN reversibility no equivaldrá a reversible.

## DB-MIGRATION-ROLLBACK-027
PARTIALLY_REVERSIBLE será first-class.

## DB-MIGRATION-ROLLBACK-028
IRREVERSIBLE será first-class.

## DB-MIGRATION-ROLLBACK-029
Conditional reversibility requerirá conditions explícitas.

## DB-MIGRATION-ROLLBACK-030
Structural reversibility no probará data reversibility.

## DB-MIGRATION-ROLLBACK-031
Data reversibility no probará semantic reversibility.

## DB-MIGRATION-ROLLBACK-032
Operational reversibility será analizada separadamente.

## DB-MIGRATION-ROLLBACK-033
Application compatibility será distinta de database compatibility.

## DB-MIGRATION-ROLLBACK-034
Database rollback no asumirá deployment rollback.

## DB-MIGRATION-ROLLBACK-035
Presence of down() no probará safety.

## DB-MIGRATION-ROLLBACK-036
VoltStack no generará destructivos `down` ejecutables automáticamente.

## DB-MIGRATION-ROLLBACK-037
Generated inverse será suggestion solamente.

## DB-MIGRATION-ROLLBACK-038
Explicit irreversibility será soportada.

## DB-MIGRATION-ROLLBACK-039
Reversibility declarations serán evidencia, no verdad absoluta.

## DB-MIGRATION-ROLLBACK-040
Current schema podrá verificarse antes de rollback.

## DB-MIGRATION-ROLLBACK-041
Repository APPLIED no probará physical schema state.

## DB-MIGRATION-ROLLBACK-042
Drift será first-class.

## DB-MIGRATION-ROLLBACK-043
Drift no será reparado automáticamente.

## DB-MIGRATION-ROLLBACK-044
Data preconditions serán first-class.

## DB-MIGRATION-ROLLBACK-045
Unknown destructive precondition bloqueará por default.

## DB-MIGRATION-ROLLBACK-046
Rollback Planner decidirá estrategia.

## DB-MIGRATION-ROLLBACK-047
Rollback Executor no replanificará silenciosamente.

## DB-MIGRATION-ROLLBACK-048
Compensation strategy será explícita.

## DB-MIGRATION-ROLLBACK-049
Forward recovery recommendation será explícita.

## DB-MIGRATION-ROLLBACK-050
Rollback operations serán typed.

## DB-MIGRATION-ROLLBACK-051
Schema rollback usará Schema subsystem.

## DB-MIGRATION-ROLLBACK-052
Data rollback usará Query/Execution subsystem.

## DB-MIGRATION-ROLLBACK-053
Rollback executor no generará SQL arbitrariamente.

## DB-MIGRATION-ROLLBACK-054
Raw rollback será explicit trust boundary.

## DB-MIGRATION-ROLLBACK-055
Safety será distinta de reversibility.

## DB-MIGRATION-ROLLBACK-056
Approval será distinta de correctness.

## DB-MIGRATION-ROLLBACK-057
Approval no convertirá impossible rollback en valid rollback.

## DB-MIGRATION-ROLLBACK-058
Data-loss risk será explícito.

## DB-MIGRATION-ROLLBACK-059
Unknown risk no se convertirá en safe.

## DB-MIGRATION-ROLLBACK-060
Transaction policy será explícita.

## DB-MIGRATION-ROLLBACK-061
Migration rollback será distinto de DB transaction rollback.

## DB-MIGRATION-ROLLBACK-062
Non-transactional rollback podrá ser partial.

## DB-MIGRATION-ROLLBACK-063
Rollback execution reutilizará execution infrastructure cuando sea posible.

## DB-MIGRATION-ROLLBACK-064
No existirá segundo driver execution engine para rollback.

## DB-MIGRATION-ROLLBACK-065
Rollback outcome certainty será first-class.

## DB-MIGRATION-ROLLBACK-066
Rollback failure será distinto de rollback certainty.

## DB-MIGRATION-ROLLBACK-067
Partial rollback será first-class.

## DB-MIGRATION-ROLLBACK-068
Partial rollback podrá producir hybrid state.

## DB-MIGRATION-ROLLBACK-069
Hybrid state no se clasificará simplemente como APPLIED o ROLLED_BACK.

## DB-MIGRATION-ROLLBACK-070
Unknown rollback outcome será first-class.

## DB-MIGRATION-ROLLBACK-071
Unknown outcome no será reintentado ciegamente.

## DB-MIGRATION-ROLLBACK-072
Rollback retry requerirá replay safety.

## DB-MIGRATION-ROLLBACK-073
Rollback retry considerará transaction certainty.

## DB-MIGRATION-ROLLBACK-074
Rollback retry considerará deadline.

## DB-MIGRATION-ROLLBACK-075
Rollback retry considerará budget.

## DB-MIGRATION-ROLLBACK-076
Rollback of rollback no será usado para reapply.

## DB-MIGRATION-ROLLBACK-077
Reapply usará normal migration application pipeline.

## DB-MIGRATION-ROLLBACK-078
Migration history no será eliminada por rollback.

## DB-MIGRATION-ROLLBACK-079
Current migration state será distinto de history.

## DB-MIGRATION-ROLLBACK-080
Repository row no será simplemente borrada como única semántica de rollback.

## DB-MIGRATION-ROLLBACK-081
ROLLED_BACK se persistirá solo tras completed rollback contract.

## DB-MIGRATION-ROLLBACK-082
Repository write failure después de physical rollback requerirá recovery.

## DB-MIGRATION-ROLLBACK-083
Repository inconsistency no provocará blind rollback retry.

## DB-MIGRATION-ROLLBACK-084
Backup requirement será explícito.

## DB-MIGRATION-ROLLBACK-085
Backup existence no implicará verified restore capability.

## DB-MIGRATION-ROLLBACK-086
Backup freshness será considerada.

## DB-MIGRATION-ROLLBACK-087
Shadow data podrá extender rollback window.

## DB-MIGRATION-ROLLBACK-088
Rollback window será first-class cuando aplique.

## DB-MIGRATION-ROLLBACK-089
Reversibility podrá cambiar con el tiempo.

## DB-MIGRATION-ROLLBACK-090
Reversibility dependerá del current state.

## DB-MIGRATION-ROLLBACK-091
Application barriers podrán bloquear rollback.

## DB-MIGRATION-ROLLBACK-092
Barrier no será warning cuando sea mandatory.

## DB-MIGRATION-ROLLBACK-093
Core no dependerá del deployment system.

## DB-MIGRATION-ROLLBACK-094
Core no dependerá de Multitenancy.

## DB-MIGRATION-ROLLBACK-095
Rollback scope será explícito.

## DB-MIGRATION-ROLLBACK-096
Tenant rollback no implicará global rollback.

## DB-MIGRATION-ROLLBACK-097
Multi-target rollback no asumirá distributed atomicity.

## DB-MIGRATION-ROLLBACK-098
No se inventará hidden 2PC.

## DB-MIGRATION-ROLLBACK-099
Multi-target partial rollback será first-class.

## DB-MIGRATION-ROLLBACK-100
Rollback coordination lock será explícito.

## DB-MIGRATION-ROLLBACK-101
Lock será distinto de transaction.

## DB-MIGRATION-ROLLBACK-102
Lost lock detendrá execution en safe boundary.

## DB-MIGRATION-ROLLBACK-103
Cancellation será distinta de restoration.

## DB-MIGRATION-ROLLBACK-104
Cancellation podrá dejar partial state.

## DB-MIGRATION-ROLLBACK-105
Deadline será globalmente respetado.

## DB-MIGRATION-ROLLBACK-106
Timeout podrá dejar unknown outcome.

## DB-MIGRATION-ROLLBACK-107
Post-rollback verification será first-class.

## DB-MIGRATION-ROLLBACK-108
Command success no implicará rollback success.

## DB-MIGRATION-ROLLBACK-109
Verification failure bloqueará success.

## DB-MIGRATION-ROLLBACK-110
Exact physical equality no será siempre requerida.

## DB-MIGRATION-ROLLBACK-111
Semantic restoration podrá utilizar comparison profile.

## DB-MIGRATION-ROLLBACK-112
Forward recovery podrá ser preferido sobre rollback.

## DB-MIGRATION-ROLLBACK-113
Rollback decision será explainable.

## DB-MIGRATION-ROLLBACK-114
Rollback decision será distinta de reversibility report.

## DB-MIGRATION-ROLLBACK-115
Rollback será privileged operation.

## DB-MIGRATION-ROLLBACK-116
Security context será explícito.

## DB-MIGRATION-ROLLBACK-117
Data-loss approvals serán auditables.

## DB-MIGRATION-ROLLBACK-118
Secrets no aparecerán en diagnostics.

## DB-MIGRATION-ROLLBACK-119
Rollback events no podrán bypass safety.

## DB-MIGRATION-ROLLBACK-120
Rollback extensions serán frozen after bootstrap.

## DB-MIGRATION-ROLLBACK-121
Extensions no podrán forge success.

## DB-MIGRATION-ROLLBACK-122
Extensions no podrán convertir UNKNOWN en SAFE sin evidence.

## DB-MIGRATION-ROLLBACK-123
Extensions no podrán ocultar data loss.

## DB-MIGRATION-ROLLBACK-124
Extensions no escribirán repository state fuera del protocol.

## DB-MIGRATION-ROLLBACK-125
Rollback budgets serán explícitos.

## DB-MIGRATION-ROLLBACK-126
Budget exhaustion no implicará clean failure.

## DB-MIGRATION-ROLLBACK-127
Rollback fault injection será soportado en testing.

## DB-MIGRATION-ROLLBACK-128
Crash after physical rollback será probado.

## DB-MIGRATION-ROLLBACK-129
Crash during repository transition será probado.

## DB-MIGRATION-ROLLBACK-130
Commit uncertainty será probado.

## DB-MIGRATION-ROLLBACK-131
Lock loss será probado.

## DB-MIGRATION-ROLLBACK-132
Persistent runtime isolation será probado.

## DB-MIGRATION-ROLLBACK-133
FrankenPHP rollback state no sobrevivirá request/run.

## DB-MIGRATION-ROLLBACK-134
RoadRunner rollback state no sobrevivirá job.

## DB-MIGRATION-ROLLBACK-135
OpenSwoole rollback state no se compartirá entre coroutines.

## DB-MIGRATION-ROLLBACK-136
No habrá global mutable current rollback.

## DB-MIGRATION-ROLLBACK-137
Rollback result será immutable.

## DB-MIGRATION-ROLLBACK-138
Rollback failure report preservará primary failure.

## DB-MIGRATION-ROLLBACK-139
Cleanup failure será secondary failure.

## DB-MIGRATION-ROLLBACK-140
Rollback telemetry será structured.

## DB-MIGRATION-ROLLBACK-141
Rollback telemetry no expondrá sensitive row data por default.

## DB-MIGRATION-ROLLBACK-142
Source mapping permitirá rastrear operation hasta migration definition.

## DB-MIGRATION-ROLLBACK-143
Irreversible migration producirá error explícito cuando rollback sea requerido.

## DB-MIGRATION-ROLLBACK-144
Rollback no inventará datos eliminados.

## DB-MIGRATION-ROLLBACK-145
Rollback no fingirá restauración semántica.

## DB-MIGRATION-ROLLBACK-146
Rollback no asumirá que DROP/CREATE son inversos semánticos.

## DB-MIGRATION-ROLLBACK-147
Rollback no asumirá que type conversion es reversible.

## DB-MIGRATION-ROLLBACK-148
Rollback no asumirá que constraint removal restaura datos previamente rechazados/eliminados.

## DB-MIGRATION-ROLLBACK-149
Rollback deberá preservar outcome traceability.

## DB-MIGRATION-ROLLBACK-150
VoltStack nunca declarará una migración revertida sin satisfacer y verificar su rollback contract.

---

# 176. Anti-patterns

## 176.1 Auto-reverse

Incorrecto:

```php
$down = reverse($up);
```

---

## 176.2 Drop equals rollback

Incorrecto:

```text
UP: CREATE TABLE
DOWN: DROP TABLE
therefore fully reversible
```

No considera datos.

---

## 176.3 Borrar migration row

Incorrecto:

```php
$repository->delete($migration);
```

como definición completa de rollback.

---

## 176.4 Down means safe

Incorrecto:

```php
if ($migration->hasDown()) {
    return SAFE;
}
```

---

## 176.5 Ignore dependents

Incorrecto:

```text
rollback M1
while M2 and M3 depend on M1
```

---

## 176.6 Reverse timestamp

Incorrecto:

```sql
ORDER BY migration_timestamp DESC
```

como única regla.

---

## 176.7 Assume data restoration

Incorrecto:

```text
DROP column
→ later ADD column
→ data restored
```

---

## 176.8 Approval overrides impossibility

Incorrecto:

```text
irreversible
+ --force
=
reversible
```

---

## 176.9 Retry unknown rollback

Incorrecto:

```text
connection lost
→ run down() again
```

---

## 176.10 Rollback as emergency magic

Incorrecto:

```text
production problem
→ automatically rollback database
```

sin verificar application/schema/data compatibility.

---

# 177. Ejemplo completo — reversible

Migration:

```text
M100
ADD INDEX idx_users_email
```

Rollback definition:

```text
DROP INDEX idx_users_email
```

Analysis:

```text
Structural       REVERSIBLE
Data             REVERSIBLE
Semantic         REVERSIBLE
Application      COMPATIBLE
Operational      SUPPORTED
Risk             LOW
```

Plan:

```text
Acquire lock
↓
Verify index exists
↓
Drop index
↓
Verify index absent
↓
Record rollback
↓
Release lock
```

---

# 178. Ejemplo — conditionally reversible

Migration:

```text
M101
ADD COLUMN normalized_email
BACKFILL normalized_email
Application switches reads to normalized_email
```

Rollback:

```text
Application barrier:
old application-compatible read path restored
        ↓
Validate original email remains authoritative
        ↓
Drop normalized_email
```

Resultado:

```text
CONDITIONALLY_REVERSIBLE
```

---

# 179. Ejemplo — partially reversible

Migration:

```text
M102
Convert free-text status
→ constrained enum
Delete invalid legacy values
```

Rollback:

```text
enum → text
```

puede restaurar estructura.

Pero los valores eliminados:

```text
cannot be reconstructed
```

Resultado:

```text
Structural = REVERSIBLE
Data       = IRREVERSIBLE
Overall    = PARTIALLY_REVERSIBLE
```

---

# 180. Ejemplo — irreversible

```text
M103
SHA256(email)
DROP original email
```

Sin backup:

```text
Data Reversibility = IRREVERSIBLE
```

Aunque pueda recrearse una columna `email`, no puede recuperarse su contenido original.

---

# 181. Ejemplo — forward recovery

Estado:

```text
M104 partially deployed
new schema already consumed by production writes
```

Rollback requeriría perder nuevas escrituras.

Entonces:

```text
ROLLBACK_BLOCKED
```

y:

```text
FORWARD_RECOVERY_RECOMMENDED
```

mediante:

```text
M105 corrective migration
```

---

# 182. Fórmula de reversibilidad

```text
Reversible(M, S)
=
StructuralRestorePossible(M, S)
∧
DataRestoreContractSatisfied(M, S)
∧
SemanticRestorePossible(M, S)
```

donde:

```text
S = current runtime/database/application state
```

---

# 183. Fórmula de rollback permitido

```text
RollbackAllowed(M, S)
=
Reversible(M, S)
∧
DependenciesSatisfied(M, S)
∧
SafetyPolicySatisfied(M, S)
∧
ApplicationCompatible(M, S)
∧
RequiredEvidenceAvailable(M, S)
∧
RequiredApprovalsPresent(M, S)
```

---

# 184. Fórmula de rollback exitoso

```text
RollbackSucceeded
=
AllRequiredRollbackOperationsSucceeded
∧
RequiredDataRestorationSucceeded
∧
RequiredPostconditionsSatisfied
∧
ExpectedRollbackStateVerified
∧
RepositoryTransitionPersisted
∧
NoBlockingOutcomeUncertainty
```

---

# 185. Fórmula de rollback parcial

```text
PartialRollback
=
RollbackEffectsExist
∧
¬RollbackContractCompleted
```

---

# 186. Fórmula de rollback incierto

```text
UnknownRollbackOutcome
=
InsufficientEvidence(
    AppliedRollbackEffects
)
```

---

# 187. Fórmula de seguridad

```text
SafeRollback
=
Reversible
∧
NoUnacceptedDataLoss
∧
OperationalRiskWithinPolicy
∧
ApplicationCompatibilitySatisfied
∧
DependenciesSatisfied
∧
VerificationPossible
∧
RecoveryPathKnown
```

---

# 188. Fórmula maestra

```text
Migration Rollback System
=
Target Resolution
+
Reverse Dependency Analysis
+
State-Dependent Reversibility Analysis
+
Explicit Rollback Definitions
+
Schema/Data Restoration
+
Compensation Support
+
Safety Gates
+
Application Compatibility
+
Transaction Planning
+
Controlled Execution
+
Post-Rollback Verification
+
Durable Repository History
+
Outcome Certainty
+
Recovery Semantics
```

---

# 189. Regla arquitectónica maestra

> **VoltStack tratará cada rollback como una nueva transición controlada del estado de la base de datos cuya reversibilidad, seguridad y resultado deben demostrarse explícitamente.**

Nunca asumirá:

```text
UP succeeded
+
DOWN exists
=
safe rollback
```

La relación correcta será:

```text
Applied Migration
        ↓
Current State
        ↓
Dependency Analysis
        ↓
Reversibility Analysis
        ↓
Safety Analysis
        ↓
Rollback Planning
        ↓
Controlled Execution
        ↓
Verification
        ↓
Repository Transition
```

---

# 190. Resultado arquitectónico

Con este diseño VoltStack soportará desde:

```text
php volt migration:rollback
```

hasta escenarios complejos de producción:

```text
zero-downtime migration rollback
large-data rollback
backup-assisted restoration
application-version coordination
multi-step rollback
partial rollback recovery
distributed target reconciliation
forward-recovery decisions
```

sin caer en el modelo simplista:

```php
$migration->down();
$repository->delete($migration);
```

El sistema preservará cuatro propiedades fundamentales:

```text
Reversibility
Safety
Traceability
Outcome Certainty
```

---

# 191. Relación con documentos posteriores

Este documento establece la arquitectura base que será refinada por:

```text
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

Especialmente:

- `108` definirá agrupación y lifecycle de batches;
- `109` convertirá diferencias estructurales en migration candidates/plans;
- `110` profundizará expand/backfill/validate/contract y compatibilidad entre versiones;
- `111` formalizará el motor general de seguridad, riesgo, approvals y políticas.

---

# 192. Siguiente documento

```text
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
```

El siguiente documento deberá definir formalmente:

```text
MigrationBatch
≠
Migration

Batch ID
≠
Migration ID

Discovery Order
≠
Batch Membership

Batch Membership
≠
Dependency

Batch
≠
Transaction

Batch
≠
Deployment

Batch
≠
Execution Attempt
```

y establecer la arquitectura para:

```text
batch creation
batch identity
batch numbering
batch membership
batch planning
dependency-aware execution
partial batch execution
batch failure
batch rollback
repository representation
retries
resume
concurrent migrators
multi-target batches
package/module migrations
persistent runtime isolation
telemetry
audit
```