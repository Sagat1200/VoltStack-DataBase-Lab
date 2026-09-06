# 110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Zero-Downtime Migration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 110 — Database Zero-Downtime Migration System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Zero-Downtime Migration System` define la arquitectura mediante la cual VoltStack podrá evolucionar esquemas y datos de bases de datos en producción mientras aplicaciones antiguas y nuevas continúan atendiendo tráfico durante una **ventana explícita de compatibilidad**.

El sistema deberá coordinar:

```text
Schema Evolution
+
Data Evolution
+
Application Compatibility
+
Traffic Continuity
+
Backfills
+
Validation
+
Cutover
+
Rollback Windows
+
Operational Safety
```

sin asumir que una operación DDL soportada por el motor sea automáticamente segura para producción.

La pregunta central es:

> **¿Cómo transforma VoltStack un cambio incompatible en una secuencia explícita de estados compatibles que permita desplegar software y evolucionar datos sin requerir una interrupción coordinada del servicio?**

---

# 2. Principio central

> **Zero downtime no es una propiedad de una sentencia SQL; es una propiedad de una transición completa entre estados de aplicación, esquema y datos.**

Formalmente:

```text
ZeroDowntimeMigration
≠
OnlineDDL
```

y:

```text
ZeroDowntimeMigration
≠
FastMigration
```

---

# 3. Separaciones fundamentales

VoltStack deberá preservar:

```text
Zero Downtime
≠
Zero Locks

Zero Downtime
≠
Zero Latency Impact

Zero Downtime
≠
Zero Risk

Zero Downtime
≠
Instant Migration

Zero Downtime
≠
Online DDL

Zero Downtime
≠
One Migration

Zero Downtime
≠
One Transaction

Zero Downtime
≠
Deployment

Migration
≠
Deployment

Schema Compatibility
≠
Application Compatibility

Backward Compatibility
≠
Forward Compatibility

Read Compatibility
≠
Write Compatibility

Dual Read
≠
Dual Write

Backfill
≠
Replication

Validation
≠
Migration Completion

Cutover
≠
Contract

Rollback
≠
Forward Recovery

Migration Phase
≠
Migration Batch
```

---

# 4. Problema fundamental

Supóngase:

```text
Application V1
```

utiliza:

```text
users.name
```

y `Application V2` desea utilizar:

```text
users.full_name
```

La migración ingenua:

```text
RENAME users.name TO users.full_name
```

puede romper inmediatamente instancias V1 todavía atendiendo tráfico.

El problema real no es:

```text
Can the database rename the column?
```

sino:

```text
Can V1 and V2 coexist safely
during the transition?
```

---

# 5. Modelo de estado conjunto

VoltStack deberá considerar:

```text
SystemState
=
ApplicationState
×
SchemaState
×
DataState
×
TrafficState
```

Una transición zero-downtime solo será válida si los estados intermedios necesarios mantienen el contrato operacional requerido.

---

# 6. Arquitectura general

```text
Desired Structural Change
          │
          ▼
Schema Diff Migration System
          │
          ▼
Migration Candidate
          │
          ▼
┌──────────────────────────────────┐
│ Zero-Downtime Strategy Analyzer  │
│                                  │
│ Compatibility Analysis           │
│ Phase Decomposition              │
│ Backfill Planning                │
│ Read/Write Transition            │
│ Validation                       │
│ Cutover                          │
│ Contract Safety                  │
└─────────────────┬────────────────┘
                  ▼
       ZeroDowntimeMigrationPlan
                  │
     ┌────────────┼─────────────┐
     ▼            ▼             ▼
   EXPAND       MIGRATE       CONTRACT
     │            │             │
     ▼            ▼             ▼
 Migration    Backfill /     Cleanup /
 Execution    Validation     Destruction
     │            │             │
     └────────────┼─────────────┘
                  ▼
         Final Target State
```

---

# 7. Patrón maestro

La estrategia principal será:

```text
EXPAND
   ↓
MIGRATE
   ↓
VALIDATE
   ↓
CUTOVER
   ↓
STABILIZE
   ↓
CONTRACT
```

En escenarios complejos:

```text
PREPARE
   ↓
EXPAND
   ↓
DUAL COMPATIBILITY
   ↓
BACKFILL
   ↓
VALIDATE
   ↓
CUTOVER
   ↓
OBSERVE
   ↓
CONTRACT
```

---

# 8. Expand

`EXPAND` agrega estructuras nuevas sin eliminar inmediatamente las utilizadas por aplicaciones existentes.

Ejemplo:

```text
users.name
```

se mantiene y se agrega:

```text
users.full_name
```

Resultado:

```text
users
├── name
└── full_name
```

---

# 9. Propiedad del Expand

Idealmente:

```text
OldApplicationCompatible(ExpandedSchema)
=
true
```

---

# 10. New application compatibility

También debe prepararse para:

```text
NewApplicationCompatible(ExpandedSchema)
=
true
```

cuando corresponda.

---

# 11. Expand ≠ safe automáticamente

Agregar una columna puede todavía causar:

- table rewrite;
- metadata lock;
- replication lag;
- storage pressure;
- index build;
- long transaction;
- blocking DDL.

Por tanto:

```text
AdditiveChange
≠
OperationallySafeChange
```

---

# 12. Migrate

La fase `MIGRATE` transforma los datos existentes hacia la nueva representación.

Ejemplo:

```text
name
↓
full_name
```

---

# 13. Backfill

```text
UPDATE users
SET full_name = name
WHERE full_name IS NULL
```

es conceptualmente un backfill.

Pero VoltStack deberá modelarlo como operación tipada y gobernada, no simplemente como SQL arbitrario.

---

# 14. Validate

Después del backfill:

```text
∀ row:
full_name satisfies target invariant
```

deberá verificarse antes del cutover/contract.

---

# 15. Cutover

`CUTOVER` cambia la autoridad lógica desde la representación antigua hacia la nueva.

Ejemplo:

```text
Before:
authoritative = name

After:
authoritative = full_name
```

---

# 16. Stabilize

VoltStack debería permitir una fase:

```text
STABILIZE / OBSERVE
```

antes del contract.

Objetivo:

```text
verify new representation
observe production
preserve rollback window
```

---

# 17. Contract

Solo después de verificar que ninguna aplicación activa necesita la estructura antigua:

```text
DROP users.name
```

podrá considerarse.

---

# 18. Regla crítica

> **Contract es una operación destructiva posterior a la eliminación demostrable de dependencias antiguas.**

---

# 19. Migration phases

Se propone:

```php
enum ZeroDowntimeMigrationPhase
{
    case PREPARE;
    case EXPAND;
    case DUAL_COMPATIBILITY;
    case BACKFILL;
    case VALIDATE;
    case CUTOVER;
    case STABILIZE;
    case CONTRACT;
    case COMPLETE;
}
```

---

# 20. Phase ≠ migration

Una fase puede contener:

```text
multiple migrations
multiple batches
multiple deployments
multiple validation operations
```

---

# 21. Phase ≠ batch

Por ejemplo:

```text
EXPAND
├── Batch B10
└── Batch B11
```

es válido.

---

# 22. Migration ≠ deployment

El sistema de base de datos deberá producir:

```text
DeploymentBarrier
```

cuando requiera coordinación externa.

No deberá convertirse en Deployment Orchestrator.

---

# 23. Deployment timeline

Ejemplo:

```text
T0  Deploy DB Expand
T1  Deploy App V2 compatibility code
T2  Start dual write
T3  Backfill
T4  Validate
T5  Switch reads
T6  Observe
T7  Remove old writes
T8  Deploy cleanup app
T9  Contract schema
```

---

# 24. Compatibility window

El intervalo:

```text
T1 ... T8
```

puede representar una:

```text
CompatibilityWindow
```

---

# 25. CompatibilityWindow

Se propone:

```php
final readonly class MigrationCompatibilityWindow
{
    public function __construct(
        public ApplicationCompatibilityContract $oldApplication,
        public ApplicationCompatibilityContract $newApplication,
        public SchemaCompatibilityContract $schema,
        public DataCompatibilityContract $data,
    ) {}
}
```

---

# 26. Backward compatibility

El nuevo schema puede soportar la aplicación anterior:

```text
OldApp → NewSchema
```

---

# 27. Forward compatibility

La aplicación nueva puede funcionar antes de completar totalmente el nuevo estado:

```text
NewApp → TransitionalSchema
```

---

# 28. Compatibility matrix

Debe poder representarse:

| Application | Old Schema | Expanded Schema | Final Schema |
|---|---:|---:|---:|
| V1 | ✓ | ✓ | ✗ |
| V2-transition | ✗/partial | ✓ | ✓ |
| V2-final | ✗ | ✓ | ✓ |

---

# 29. Compatibility contract

No deberá ser un simple boolean.

Se propone:

```text
SUPPORTED
SUPPORTED_WITH_LIMITATIONS
UNSUPPORTED
UNKNOWN
```

---

# 30. Compatibility dimensions

Al menos:

```text
READ
WRITE
CONSTRAINT
DATA_REPRESENTATION
TRANSACTIONAL
PERFORMANCE
```

---

# 31. Read compatibility

Una aplicación puede leer ambas representaciones.

Ejemplo:

```php
$value = $row->full_name ?? $row->name;
```

---

# 32. Dual read

Durante transición:

```text
read new
fallback old
```

o:

```text
read both
compare
```

---

# 33. Dual read strategies

```text
NEW_THEN_OLD
OLD_THEN_NEW
READ_BOTH_COMPARE
NEW_ONLY
OLD_ONLY
```

---

# 34. Dual write

Durante la ventana:

```text
write old
+
write new
```

---

# 35. Dual write problem

Dual write puede fallar:

```text
write old ✓
write new ✗
```

Por tanto:

```text
DualWrite
≠
AtomicWrite
```

---

# 36. Dual-write strategies

Podrán incluir:

```text
APPLICATION_DUAL_WRITE
DATABASE_TRIGGER
CHANGE_CAPTURE
SINGLE_SOURCE_DERIVATION
TRANSACTIONAL_DUAL_WRITE
```

---

# 37. Default recommendation

Preferir:

```text
single authoritative representation
+
derived transitional representation
```

cuando sea posible.

---

# 38. Dual write ownership

Debe declararse:

```text
OldAuthoritative
NewAuthoritative
Bidirectional
```

---

# 39. Bidirectional synchronization

Será altamente restringida porque puede producir:

```text
write conflicts
cycles
ordering problems
semantic divergence
```

---

# 40. Authority state

Se propone:

```php
enum MigrationDataAuthority
{
    case OLD;
    case NEW;
    case TRANSITIONAL;
}
```

---

# 41. Authority transition

```text
OLD
 ↓
TRANSITIONAL
 ↓
NEW
```

deberá ser explícita.

---

# 42. Shadow column

Una técnica común:

```text
original column
+
shadow column
```

---

# 43. Ejemplo

```text
price VARCHAR
```

debe convertirse a:

```text
price_decimal DECIMAL
```

Estrategia:

```text
ADD price_decimal nullable
↓
dual-write
↓
backfill
↓
validate conversion
↓
read price_decimal
↓
stop old writes
↓
drop price
```

---

# 44. Shadow table

Cambios complejos pueden usar:

```text
old_table
new_table
```

con:

```text
copy
synchronization
validation
cutover
cleanup
```

---

# 45. Table replacement

Pipeline conceptual:

```text
Create Shadow Table
        ↓
Initial Copy
        ↓
Catch-up Synchronization
        ↓
Validation
        ↓
Cutover
        ↓
Observation Window
        ↓
Retire Old Table
```

---

# 46. Shadow strategy ≠ generic default

Tiene costos altos:

```text
storage
write amplification
sync complexity
operational risk
```

---

# 47. Backfill architecture

Zero-downtime backfills deberán utilizar el backfill model introducido en Migration Architecture.

Se propone:

```text
ZeroDowntimeBackfillPlan
```

---

# 48. Backfill requirements

Deberá declarar:

```text
source
target
ordering key
batch size
resume token
throttling
concurrency
idempotency
validation
replica impact policy
```

---

# 49. Chunking

Para grandes datasets:

```text
Backfill
→
Chunk 1
→
Checkpoint
→
Chunk 2
→
Checkpoint
→ ...
```

---

# 50. Offset pagination

No deberá ser default para grandes tablas mutables.

Preferir:

```text
keyset / cursor progression
```

sobre una clave estable.

---

# 51. Resume token

Ejemplo:

```text
last_processed_id = 5,000,000
```

pero conceptualmente deberá ser:

```text
BackfillResumeToken
```

tipado.

---

# 52. Idempotency

Un backfill reanudable deberá diseñarse para que:

```text
ApplyChunk(C)
ApplyChunk(C)
```

no produzca corrupción.

---

# 53. Replayability

Aun así:

```text
Idempotency
≠
Replayability
```

porque inputs externos o estado mutable pueden cambiar.

---

# 54. Backfill checkpoint

Se propone:

```php
final readonly class MigrationBackfillCheckpoint
{
    public function __construct(
        public BackfillId $backfill,
        public BackfillResumeToken $token,
        public BackfillProgress $progress,
        public Instant $recordedAt,
    ) {}
}
```

---

# 55. Checkpoint durability

No deberá adelantarse al efecto durable.

Incorrecto:

```text
checkpoint chunk complete
↓
commit data
```

Correcto:

```text
durable data effect
↓
durable checkpoint
```

con manejo explícito de incertidumbre entre ambos.

---

# 56. Backfill throttling

Debe soportar:

```text
rows per batch
sleep interval
max QPS
max transaction duration
replica lag threshold
database load threshold
```

mediante policy/capabilities.

---

# 57. Adaptive throttling

Puede existir:

```text
load increases
↓
reduce backfill rate
```

pero deberá ser:

```text
bounded
observable
policy-driven
```

---

# 58. Replica awareness

En sistemas read-replica:

```text
large backfill
→
replication lag
```

---

# 59. Replica lag gate

Puede modelarse:

```text
if lag > threshold:
    pause backfill
```

como policy operacional.

---

# 60. Replica lag ≠ database correctness

Es una condición operacional, no una propiedad del Schema Model.

---

# 61. Validation architecture

Antes de avanzar de fase deberán existir:

```text
MigrationPhaseValidation
```

---

# 62. Validation types

```text
SCHEMA_VALIDATION
DATA_VALIDATION
CONSISTENCY_VALIDATION
APPLICATION_COMPATIBILITY_VALIDATION
REPLICA_VALIDATION
PERFORMANCE_VALIDATION
OPERATIONAL_VALIDATION
```

---

# 63. Validation gate

```text
Phase N
   ↓
Validation Gate
   ↓
Phase N+1
```

---

# 64. Validation result

```text
PASS
FAIL
UNKNOWN
PARTIAL
```

---

# 65. UNKNOWN

Nunca deberá tratarse automáticamente como:

```text
PASS
```

---

# 66. Dual-read comparison

Durante migración:

```text
old_value
new_value
```

pueden compararse.

Resultado:

```text
MATCH
MISMATCH
UNCOMPARABLE
UNKNOWN
```

---

# 67. Sampling

Para tablas enormes puede utilizarse:

```text
sampling
```

pero:

```text
SampleValid
≠
GloballyValid
```

---

# 68. Validation strength

Se propone:

```text
EXHAUSTIVE
BOUNDED
SAMPLED
INFERRED
UNKNOWN
```

---

# 69. Constraint validation phases

Algunas plataformas permiten:

```text
add constraint
without validating all existing rows
↓
validate later
```

cuando capability lo soporte.

---

# 70. Capability-driven

No asumir esta estrategia universalmente.

---

# 71. Online index creation

Cuando plataforma soporte:

```text
concurrent/online index build
```

podrá utilizarse.

---

# 72. Online index ≠ no impact

Todavía puede consumir:

```text
CPU
I/O
memory
temporary disk
replication bandwidth
```

---

# 73. Index strategy

Se propone:

```text
BLOCKING
ONLINE
CONCURRENT
DEFERRED
PLATFORM_SPECIFIC
UNKNOWN
```

---

# 74. Constraint tightening

Ejemplo:

```text
nullable
→
NOT NULL
```

zero-downtime podría ser:

```text
1 application stops creating NULL
2 backfill existing NULL
3 validate no NULL
4 enforce NOT NULL
```

---

# 75. Unique constraint

```text
1 stop creating duplicates
2 detect duplicates
3 repair duplicates
4 build unique structure
5 validate
```

---

# 76. Foreign key

```text
1 application stops creating orphans
2 repair existing orphans
3 add FK using least-blocking strategy
4 validate FK
```

---

# 77. Type change

Direct:

```text
ALTER TYPE
```

puede bloquear/rewrite.

Alternative:

```text
shadow column
↓
dual-write
↓
backfill conversion
↓
validate
↓
cutover
↓
contract
```

---

# 78. Primary key change

Debe considerarse especialmente riesgoso por afectar:

```text
foreign keys
indexes
ORM identity
replication
partitioning
application contracts
```

---

# 79. Large-table classification

Se propone:

```text
MigrationTableScaleProfile
```

con métricas/hints:

```text
row count estimate
table bytes
index bytes
write rate
read rate
replica count
criticality
```

---

# 80. Estimates ≠ facts

Deberán conservar:

```text
source
timestamp
certainty
```

---

# 81. Scale profile no en Schema Model

Es metadata operacional.

---

# 82. Strategy selection

Se propone:

```php
interface ZeroDowntimeMigrationStrategySelector
{
    public function select(
        MigrationCandidate $candidate,
        ZeroDowntimeMigrationContext $context
    ): ZeroDowntimeStrategySelection;
}
```

---

# 83. Strategy inputs

```text
schema change
platform capabilities
table scale
application compatibility
deployment constraints
SLO
risk policy
replica topology
```

---

# 84. Strategy result

No solo enum.

Debe contener:

```text
selected strategy
alternatives
requirements
limitations
evidence
risks
unknowns
```

---

# 85. Strategy examples

```text
DIRECT_ONLINE_DDL
EXPAND_CONTRACT
SHADOW_COLUMN
SHADOW_TABLE
ONLINE_INDEX
PHASED_CONSTRAINT
APPLICATION_DUAL_WRITE
TRIGGER_SYNCHRONIZATION
MANUAL_STRATEGY
UNSUPPORTED
```

---

# 86. Direct migration

Una migración directa puede calificarse zero-downtime solo si:

```text
operational characteristics
+
application compatibility
+
platform behavior
```

lo justifican.

---

# 87. Lock analysis

Se propone:

```text
MigrationLockImpact
```

---

# 88. Lock impact states

```text
NONE_EXPECTED
METADATA_ONLY
SHORT_BLOCKING
POTENTIALLY_LONG
LONG_BLOCKING
PLATFORM_DEPENDENT
UNKNOWN
```

---

# 89. Lock estimate ≠ guarantee

Debe conservar uncertainty.

---

# 90. Lock budget

Policy puede declarar:

```text
max acceptable blocking duration
```

---

# 91. Lock timeout

Puede utilizarse para evitar bloqueo prolongado.

Pero:

```text
LockTimeout
≠
MigrationTimeout
```

---

# 92. Application compatibility declaration

Idealmente la aplicación podrá declarar:

```text
DatabaseContractVersion
```

---

# 93. Database contract

Ejemplo:

```text
App V1 requires DB contract >= 10 < 13
App V2 requires DB contract >= 11 < 15
```

---

# 94. Compatibility overlap

Zero downtime requiere, en determinados puntos:

```text
SupportedDB(V1)
∩
SupportedDB(V2)
≠
∅
```

---

# 95. Contract version ≠ migration number

Muy importante:

```text
DatabaseContractVersion
≠
MigrationBatchSequence
```

---

# 96. Contract capability

También podría declararse por capabilities:

```text
requires column users.full_name
accepts legacy users.name
```

en vez de un número global.

---

# 97. Application dependency manifest

Se propone:

```text
DatabaseApplicationContractManifest
```

---

# 98. Manifest example

```php
DatabaseContract::define('users-v2')
    ->requiresColumn('users', 'full_name')
    ->acceptsColumn('users', 'name')
    ->writes('users.full_name')
    ->readsFallback('users.name');
```

---

# 99. Contract System boundary

Este manifest pertenece a integración Framework/Database.

No convierte Database en Deployment Manager.

---

# 100. Deployment barrier

Se propone:

```text
MigrationDeploymentBarrier
```

---

# 101. Barrier examples

```text
WAIT_FOR_APPLICATION_V2_DEPLOYED
WAIT_FOR_OLD_WORKERS_DRAINED
WAIT_FOR_DUAL_WRITE_ENABLED
WAIT_FOR_NEW_READ_PATH_ENABLED
WAIT_FOR_ROLLBACK_WINDOW_EXPIRED
```

---

# 102. Barrier execution

Migration Engine puede:

```text
pause
report
require external acknowledgement
```

No necesariamente desplegar software.

---

# 103. Manual acknowledgement

Podrá requerirse:

```text
operator approval
CI/CD signal
deployment integration
```

---

# 104. Barrier evidence

Debe conservar:

```text
who/what satisfied it
timestamp
evidence
source
```

---

# 105. Old worker draining

Especialmente importante con:

```text
FrankenPHP
RoadRunner
OpenSwoole
queue workers
```

porque procesos persistentes pueden conservar código antiguo.

---

# 106. Deployment complete ≠ old code gone

Puede haber:

```text
old HTTP workers
old queue jobs
long-running jobs
scheduled workers
```

todavía activos.

---

# 107. Contract safety

Antes de eliminar estructura legacy:

```text
NoActiveConsumer(old schema)
```

deberá estar suficientemente demostrado.

---

# 108. Consumer registry

Podría integrarse con:

```text
Runtime
Deployment
Telemetry
Job/Queue
Application Contract manifests
```

sin ser dependencia obligatoria del core Migration System.

---

# 109. Cutover

Se propone:

```text
MigrationCutover
```

como transición explícita.

---

# 110. Cutover types

```text
READ_CUTOVER
WRITE_CUTOVER
AUTHORITY_CUTOVER
TABLE_CUTOVER
INDEX_CUTOVER
CONSTRAINT_CUTOVER
```

---

# 111. Read cutover

```text
old reads
↓
new reads with fallback
↓
new-only reads
```

---

# 112. Write cutover

```text
old authoritative write
↓
dual write
↓
new authoritative write
↓
new-only write
```

---

# 113. Cutover preconditions

Ejemplo:

```text
backfill complete
validation pass
new code deployed
dual writes healthy
replica lag acceptable
```

---

# 114. Cutover postconditions

```text
new writes observed
new reads healthy
old/new divergence below policy threshold
```

---

# 115. Cutover rollback

Antes del contract puede ser posible:

```text
switch back to old reads
```

si la vieja representación sigue sincronizada.

---

# 116. Rollback window

Se propone:

```text
MigrationRollbackWindow
```

---

# 117. Rollback window state

```text
OPEN
DEGRADED
CLOSED
UNKNOWN
```

---

# 118. Rollback capability

No es solo tiempo.

Depende de:

```text
old schema still exists
old data still synchronized
old app deployable
no irreversible new writes
```

---

# 119. Fórmula conceptual

```text
RollbackPossible
=
OldStructureAvailable
∧
OldRepresentationConsistent
∧
OldApplicationCompatible
∧
NoIrreversibleTransition
```

---

# 120. Contract closes rollback

Con frecuencia:

```text
DROP legacy column
```

cierra o degrada severamente la rollback window.

---

# 121. Contract gate

Antes de contract:

```text
RollbackWindowMayClose
```

debe ser explícitamente visible.

---

# 122. Forward recovery

En producción puede ser preferible:

```text
ForwardRecovery
```

sobre rollback.

---

# 123. Forward recovery ≠ rollback

Ejemplo:

```text
new schema active
minor data mismatch
```

puede resolverse corrigiendo/backfilling hacia adelante.

---

# 124. Contract phase

Solo deberá ejecutarse si:

```text
new state validated
old consumers absent
rollback policy satisfied
safety policy satisfied
```

---

# 125. Contract operation examples

```text
drop legacy column
drop compatibility trigger
drop shadow table
remove old index
enforce stricter constraint
remove transitional default
```

---

# 126. Contract may be delayed

Es válido mantener estructura legacy durante:

```text
hours
days
releases
```

si policy lo permite.

---

# 127. Zero-downtime plan

Se propone:

```php
final readonly class ZeroDowntimeMigrationPlan
{
    public function __construct(
        public ZeroDowntimeMigrationPlanId $id,
        public MigrationTargetIdentity $target,
        public ZeroDowntimeMigrationPhaseSet $phases,
        public MigrationCompatibilityWindowSet $compatibilityWindows,
        public MigrationDeploymentBarrierSet $barriers,
        public MigrationRollbackWindowSet $rollbackWindows,
        public ZeroDowntimeRiskReport $risk,
        public ZeroDowntimeMigrationPlanFingerprint $fingerprint,
    ) {}
}
```

---

# 128. Plan ≠ execution

```text
ZeroDowntimeMigrationPlan
≠
ZeroDowntimeMigrationExecution
```

---

# 129. Phase plan

```php
final readonly class ZeroDowntimeMigrationPhasePlan
{
    public function __construct(
        public ZeroDowntimeMigrationPhaseId $id,
        public ZeroDowntimeMigrationPhase $phase,
        public MigrationDefinitionSet $migrations,
        public MigrationValidationSet $validations,
        public MigrationDeploymentBarrierSet $barriers,
        public MigrationPhasePreconditionSet $preconditions,
        public MigrationPhasePostconditionSet $postconditions,
    ) {}
}
```

---

# 130. Phase dependencies

Ejemplo:

```text
EXPAND
   ↓
BACKFILL
   ↓
VALIDATE
   ↓
CUTOVER
```

deberán formar DAG explícito.

---

# 131. Phase checkpoint

Cada fase completada tendrá:

```text
MigrationPhaseCheckpoint
```

durable.

---

# 132. Phase checkpoint ≠ batch record

Batch System registra ejecución agrupada de migrations.

Phase checkpoint registra progreso de estrategia zero-downtime.

---

# 133. Resume

Después de crash:

```text
load ZeroDowntimePlan
↓
load phase checkpoints
↓
load batch/repository state
↓
reconcile active operation
↓
validate compatibility assumptions
↓
resume
```

---

# 134. Resume preconditions

```text
plan fingerprint unchanged
schema state compatible
application contract state compatible
completed phase postconditions valid
unknown operations reconciled
```

---

# 135. Stale plan

Si aplicación/esquema/topología cambió:

```text
STALE_ZERO_DOWNTIME_PLAN
```

---

# 136. No blind resume

Especialmente tras:

```text
cutover
contract
unknown DDL outcome
```

---

# 137. State machine

```text
PLANNED
   ↓
PREPARING
   ↓
EXPANDING
   ↓
MIGRATING
   ↓
VALIDATING
   ↓
CUTTING_OVER
   ↓
STABILIZING
   ↓
CONTRACTING
   ↓
COMPLETED
```

Alternativas:

```text
PAUSED
BLOCKED
FAILED
PARTIAL
UNKNOWN
REQUIRES_RECOVERY
```

---

# 138. Pause

Zero-downtime plans deberán poder:

```text
pause between safe boundaries
```

---

# 139. Safe pause boundary

Ejemplos:

```text
after expand
between backfill chunks
after validation
before cutover
before contract
```

---

# 140. Unsafe pause

Puede existir dentro de:

```text
atomic cutover sequence
```

y deberá ser explícita.

---

# 141. Cancellation

Cancellation no significa:

```text
restore original schema
```

---

# 142. Cancellation result

Debe reportar:

```text
current phase
completed operations
active compatibility window
data authority
rollback availability
recommended recovery
```

---

# 143. Timeouts

Cada nivel puede tener:

```text
operation timeout
migration timeout
phase timeout
plan deadline
lock timeout
```

---

# 144. Effective deadline

Debe propagarse hacia abajo.

---

# 145. Transaction model

No existe una transacción que abarque:

```text
hours/days
multiple deployments
backfills
cutover
contract
```

---

# 146. Transactions are local

Se utilizarán alrededor de operaciones concretas cuando sean soportadas y convenientes.

---

# 147. Saga-like nature

Zero-downtime migration se parece operacionalmente a:

```text
long-running stateful workflow
```

pero el Migration System no deberá fingir atomicidad global.

---

# 148. Failure model

Cada fase puede producir:

```text
CERTAIN_SUCCESS
CERTAIN_FAILURE
PARTIAL
UNKNOWN
```

---

# 149. Failure during expand

Normalmente:

```text
old application still functional
```

si expand fue correctamente diseñado.

---

# 150. Failure during backfill

Idealmente:

```text
resume
```

desde checkpoint.

---

# 151. Failure during cutover

Es especialmente sensible.

Debe determinar:

```text
which representation is authoritative?
```

---

# 152. Failure during contract

Puede ser irreversible.

Requiere tratamiento conservador.

---

# 153. Outcome certainty

Reglas del documento 85/86 siguen aplicando:

```text
Retryable Failure
≠
Safe Retry
```

---

# 154. Safety system integration

El documento 111 deberá evaluar:

```text
phase risks
DDL risks
data risks
locking risks
contract risks
rollback-window risks
resource risks
```

---

# 155. Safety decision ≠ zero-downtime classification

Una estrategia puede ser zero-downtime pero:

```text
too risky
```

para policy actual.

---

# 156. Resource governance

El plan deberá poder declarar budgets:

```text
max DB load
max replica lag
max lock wait
max chunk duration
max write amplification
max temporary storage
max total runtime
```

---

# 157. Resource pressure

Si excede thresholds:

```text
PAUSE
THROTTLE
ABORT_BEFORE_NEXT_BOUNDARY
```

según policy.

---

# 158. Disk-space safety

Shadow tables/indexes pueden requerir gran espacio temporal.

Debe existir:

```text
StorageCapacityRequirement
```

---

# 159. Capacity unknown

No asumir suficiente espacio.

Resultado:

```text
UNKNOWN / REQUIRE VALIDATION
```

según policy.

---

# 160. Observability

Zero-downtime migration requiere telemetría first-class.

---

# 161. Metrics

Ejemplos:

```text
migration.phase
backfill.rows_processed
backfill.rows_remaining
backfill.rate
backfill.errors
backfill.divergence
replica.lag
database.load
lock.wait
cutover.duration
validation.mismatch_count
```

---

# 162. Tracing

Correlación:

```text
ZeroDowntimePlanId
PhaseId
MigrationId
BatchId
MigrationExecutionId
BackfillId
TargetId
DeploymentCorrelationId
```

---

# 163. Events

Se proponen:

```text
ZeroDowntimeMigrationPlanned
MigrationPhaseStarting
MigrationPhaseCompleted
MigrationPhaseBlocked
CompatibilityWindowOpened
CompatibilityWindowClosing
BackfillStarted
BackfillCheckpointed
BackfillCompleted
ValidationPassed
ValidationFailed
CutoverStarting
CutoverCompleted
RollbackWindowOpened
RollbackWindowClosed
ContractStarting
ContractCompleted
ZeroDowntimeMigrationCompleted
ZeroDowntimeMigrationRecoveryRequired
```

---

# 164. Event listeners

No podrán:

- saltarse required validation;
- cerrar rollback window;
- declarar compatibility;
- alterar authority;
- marcar phase complete;
- ejecutar contract destructivo;

sin pasar por contratos autorizados.

---

# 165. Security

Las operaciones zero-downtime podrán requerir privilegios adicionales.

Ejemplos:

```text
CREATE TRIGGER
CREATE INDEX
ALTER TABLE
CREATE TABLE
DROP COLUMN
```

---

# 166. Least privilege

No exigir superuser universal si puede evitarse.

---

# 167. Trigger strategy

Si se usan triggers para sincronización:

```text
trigger definition
ownership
cleanup
failure semantics
recursive behavior
performance impact
```

deberán modelarse.

---

# 168. Trigger ≠ invisible implementation detail

Debe aparecer explícitamente en el plan.

---

# 169. Application dual-write

Igualmente deberá aparecer como:

```text
ExternalApplicationRequirement
```

---

# 170. External requirement

Ejemplo:

```text
Enable application dual write before backfill catch-up
```

---

# 171. External state verification

No asumir que un deployment ocurrió solo porque el operador lo dijo anteriormente.

El barrier deberá registrar evidencia apropiada.

---

# 172. Queue jobs

Antes de contract deberá considerarse que jobs antiguos pueden contener lógica que todavía usa schema legacy.

---

# 173. Scheduled tasks

Misma consideración.

---

# 174. Persistent runtime integration

FrankenPHP:

```text
old worker process
```

puede sobrevivir más allá del deployment.

---

# 175. Worker generation

RuntimeManagerServer podría exponer:

```text
RuntimeGenerationId
```

para demostrar que workers legacy fueron drenados.

---

# 176. Optional integration

Database Migration System no dependerá obligatoriamente de RuntimeManagerServer.

---

# 177. Multitenancy

Multitenancy sigue siendo paquete opcional.

---

# 178. Tenant zero-downtime

Cuando esté instalado:

```text
Tenant A
Tenant B
Tenant C
```

pueden estar en diferentes fases.

---

# 179. No global mutable phase

Nunca:

```php
ZeroDowntimeMigration::$currentPhase;
```

---

# 180. Tenant state

Debe estar:

```text
target/tenant scoped
```

y persistido cuando sea durable.

---

# 181. Rolling tenant migration

Puede utilizar:

```text
canary tenants
↓
small cohort
↓
larger cohort
↓
all tenants
```

---

# 182. Cohort ≠ batch

`TenantMigrationCohort` será conceptualmente distinto de `MigrationBatch`.

---

# 183. Canary

Puede reducir riesgo operacional.

Pero:

```text
CanarySuccess
≠
GlobalSafetyProof
```

---

# 184. Sharding

En sistemas shard-aware:

```text
Shard 1
Shard 2
Shard 3
```

pueden migrarse progresivamente.

---

# 185. Cross-shard atomicity

Nunca asumida.

---

# 186. Shard compatibility window

Aplicación deberá tolerar temporalmente:

```text
some shards old schema
some shards new schema
```

si estrategia rolling lo requiere.

---

# 187. Schema capability routing

Esto puede requerir que aplicación conozca:

```text
DatabaseCapability/Contract
```

por target.

---

# 188. Platform adapters

Se podrán implementar:

```text
MySqlZeroDowntimeAdvisor
MariaDbZeroDowntimeAdvisor
PostgreSqlZeroDowntimeAdvisor
SqliteZeroDowntimeAdvisor
```

---

# 189. Advisor ≠ planner

Advisor aporta:

```text
platform facts
limitations
strategy evidence
```

Planner conserva decisión.

---

# 190. SQLite

Zero-downtime en SQLite tendrá restricciones diferentes por su modelo de concurrencia y operaciones estructurales.

No deberá fingirse equivalencia con PostgreSQL/MySQL.

---

# 191. PostgreSQL

Capacidades específicas podrán habilitar estrategias como creación concurrente de ciertos índices y validaciones diferidas, siempre mediante capability checks.

---

# 192. MySQL/MariaDB

Deberán analizarse por separado respecto a:

```text
online/in-place DDL
algorithm
locking
instant operations
version/capability differences
```

---

# 193. MariaDB first-class

Nunca tratar:

```text
MariaDB = MySQL
```

como regla arquitectónica.

---

# 194. Capability snapshot

Toda decisión deberá usar:

```text
immutable capability snapshot
```

---

# 195. Runtime revalidation

Antes de una fase sensible puede requerirse:

```text
refresh/revalidate capabilities
```

mediante orchestrator explícito.

No desde un objeto de plan puro.

---

# 196. Plan determinism

Mismos:

```text
candidate
capabilities
policies
application contracts
operational metadata snapshot
planner version
```

deberán producir plan canónico equivalente.

---

# 197. Fingerprint

Conceptualmente:

```text
Hash(
    MigrationCandidateFingerprint
    + CapabilityFingerprint
    + ApplicationContractFingerprint
    + PolicyFingerprint
    + OperationalMetadataFingerprint
    + PlannerVersion
    + ExtensionFingerprint
)
```

---

# 198. Operational metadata freshness

El fingerprint puede distinguir:

```text
structural plan fingerprint
```

de:

```text
operational assessment fingerprint
```

porque métricas como table size cambian frecuentemente.

---

# 199. Plan identity ≠ assessment identity

Esto evita invalidar toda identidad estructural porque cambió el row count.

---

# 200. ZeroDowntimeAssessment

Se propone:

```text
ZeroDowntimeMigrationAssessment
```

separado del plan.

---

# 201. Assessment

Contendrá:

```text
current scale
load
replica health
estimated impact
compatibility status
current risks
freshness
```

---

# 202. Reassessment

Antes de operaciones costosas:

```text
reassess
```

---

# 203. Testing

El sistema deberá probar al menos:

```text
simple additive change
column rename
type migration
nullable → not null
unique constraint
foreign key
large backfill
backfill resume
backfill idempotency
dual read
dual write
cutover
cutover failure
contract
contract blocked by old workers
rollback window
replica lag
lock timeout
online index
shadow table
persistent workers
tenant rolling migration
shard rolling migration
platform capability drift
```

---

# 204. Rename test

```text
name
→
full_name
```

debe producir estrategia expand/contract cuando direct rename rompa compatibility.

---

# 205. Backfill crash test

```text
Chunk 1 ✓
Chunk 2 ✓
Crash
```

Resume deberá continuar desde checkpoint reconciliado.

---

# 206. Dual-write divergence test

Simular:

```text
old write ✓
new write ✗
```

y comprobar detección/recovery.

---

# 207. Old worker test

Contract deberá bloquearse mientras exista evidencia de consumer legacy.

---

# 208. Replica test

Backfill deberá throttlearse/pausarse al exceder lag policy.

---

# 209. Cutover failure test

Debe preservar autoridad conocida o marcar:

```text
UNKNOWN
```

si no puede determinarse.

---

# 210. Contract failure test

Debe reportar partial/unknown effects sin afirmar rollback automático.

---

# 211. Error hierarchy

```text
DatabaseZeroDowntimeMigrationException
├── ZeroDowntimePlanningException
├── ZeroDowntimeStrategyNotFoundException
├── ZeroDowntimeCompatibilityException
├── ZeroDowntimeApplicationCompatibilityException
├── ZeroDowntimeSchemaCompatibilityException
├── ZeroDowntimeDataCompatibilityException
├── ZeroDowntimePhaseException
├── ZeroDowntimePhaseDependencyException
├── ZeroDowntimePhaseValidationException
├── ZeroDowntimeBarrierException
├── ZeroDowntimeBarrierNotSatisfiedException
├── ZeroDowntimeBackfillException
├── ZeroDowntimeBackfillCheckpointException
├── ZeroDowntimeBackfillDivergenceException
├── ZeroDowntimeCutoverException
├── ZeroDowntimeCutoverUnknownOutcomeException
├── ZeroDowntimeContractException
├── ZeroDowntimeContractBlockedException
├── ZeroDowntimeRollbackWindowException
├── ZeroDowntimeResourceBudgetException
├── ZeroDowntimeReplicaLagException
├── ZeroDowntimePlanStaleException
├── ZeroDowntimeRecoveryException
├── ZeroDowntimeExtensionException
└── ZeroDowntimeInvariantException
```

---

# 212. Namespace

Se propone:

```text
VoltStack\Quantum\Database\Migration\ZeroDowntime
```

---

# 213. Estructura propuesta

```text
Migration/
└── ZeroDowntime/
    ├── Contract/
    │   ├── ZeroDowntimeMigrationPlanner.php
    │   ├── ZeroDowntimeMigrationStrategySelector.php
    │   ├── ZeroDowntimeMigrationExecutor.php
    │   └── ZeroDowntimeMigrationAdvisor.php
    │
    ├── Planning/
    │   ├── DefaultZeroDowntimeMigrationPlanner.php
    │   ├── ZeroDowntimeMigrationPlan.php
    │   ├── ZeroDowntimeMigrationPlanId.php
    │   ├── ZeroDowntimeMigrationPlanFingerprint.php
    │   └── ZeroDowntimeMigrationContext.php
    │
    ├── Phase/
    │   ├── ZeroDowntimeMigrationPhase.php
    │   ├── ZeroDowntimeMigrationPhaseId.php
    │   ├── ZeroDowntimeMigrationPhasePlan.php
    │   ├── ZeroDowntimeMigrationPhaseSet.php
    │   ├── MigrationPhaseCheckpoint.php
    │   └── MigrationPhaseStateMachine.php
    │
    ├── Compatibility/
    │   ├── MigrationCompatibilityWindow.php
    │   ├── ApplicationCompatibilityContract.php
    │   ├── SchemaCompatibilityContract.php
    │   ├── DataCompatibilityContract.php
    │   ├── DatabaseApplicationContractManifest.php
    │   └── DatabaseContractVersion.php
    │
    ├── Strategy/
    │   ├── ZeroDowntimeStrategy.php
    │   ├── ZeroDowntimeStrategySelection.php
    │   ├── ExpandContractStrategy.php
    │   ├── ShadowColumnStrategy.php
    │   ├── ShadowTableStrategy.php
    │   ├── OnlineIndexStrategy.php
    │   └── PhasedConstraintStrategy.php
    │
    ├── Data/
    │   ├── MigrationDataAuthority.php
    │   ├── MigrationReadStrategy.php
    │   ├── MigrationWriteStrategy.php
    │   ├── MigrationDualReadContract.php
    │   └── MigrationDualWriteContract.php
    │
    ├── Backfill/
    │   ├── ZeroDowntimeBackfillPlan.php
    │   ├── BackfillId.php
    │   ├── BackfillResumeToken.php
    │   ├── MigrationBackfillCheckpoint.php
    │   ├── BackfillThrottlePolicy.php
    │   └── BackfillValidation.php
    │
    ├── Validation/
    │   ├── MigrationPhaseValidation.php
    │   ├── MigrationValidationResult.php
    │   ├── MigrationValidationStrength.php
    │   ├── MigrationPhasePrecondition.php
    │   └── MigrationPhasePostcondition.php
    │
    ├── Cutover/
    │   ├── MigrationCutover.php
    │   ├── MigrationCutoverType.php
    │   ├── MigrationCutoverPlan.php
    │   └── MigrationCutoverResult.php
    │
    ├── Barrier/
    │   ├── MigrationDeploymentBarrier.php
    │   ├── MigrationDeploymentBarrierSet.php
    │   ├── MigrationBarrierEvidence.php
    │   └── MigrationBarrierResolver.php
    │
    ├── Rollback/
    │   ├── MigrationRollbackWindow.php
    │   ├── MigrationRollbackWindowState.php
    │   └── MigrationForwardRecoveryPlan.php
    │
    ├── Impact/
    │   ├── MigrationLockImpact.php
    │   ├── MigrationTableScaleProfile.php
    │   ├── MigrationOperationalImpact.php
    │   └── MigrationStorageCapacityRequirement.php
    │
    ├── Assessment/
    │   ├── ZeroDowntimeMigrationAssessment.php
    │   └── ZeroDowntimeMigrationAssessor.php
    │
    ├── Platform/
    │   ├── MySqlZeroDowntimeAdvisor.php
    │   ├── MariaDbZeroDowntimeAdvisor.php
    │   ├── PostgreSqlZeroDowntimeAdvisor.php
    │   └── SqliteZeroDowntimeAdvisor.php
    │
    ├── Runtime/
    │   ├── ZeroDowntimeMigrationExecution.php
    │   ├── ZeroDowntimeMigrationExecutionSession.php
    │   └── ZeroDowntimeMigrationRecoveryDescriptor.php
    │
    ├── Telemetry/
    │   └── ZeroDowntimeMigrationTelemetry.php
    │
    ├── Extension/
    │   ├── ZeroDowntimeMigrationExtension.php
    │   └── ZeroDowntimeMigrationExtensionRegistry.php
    │
    └── Exception/
        └── ...
```

---

# 214. Invariantes

## DB-MIGRATION-ZDT-001
Zero downtime será distinto de online DDL.

## DB-MIGRATION-ZDT-002
Zero downtime será distinto de zero locks.

## DB-MIGRATION-ZDT-003
Zero downtime será distinto de zero latency impact.

## DB-MIGRATION-ZDT-004
Zero downtime será distinto de zero risk.

## DB-MIGRATION-ZDT-005
Zero downtime será distinto de instant migration.

## DB-MIGRATION-ZDT-006
Zero downtime será distinto de one transaction.

## DB-MIGRATION-ZDT-007
Migration será distinta de deployment.

## DB-MIGRATION-ZDT-008
Schema compatibility será distinta de application compatibility.

## DB-MIGRATION-ZDT-009
Backward compatibility será distinta de forward compatibility.

## DB-MIGRATION-ZDT-010
Read compatibility será distinta de write compatibility.

## DB-MIGRATION-ZDT-011
Dual read será distinto de dual write.

## DB-MIGRATION-ZDT-012
Backfill será distinto de replication.

## DB-MIGRATION-ZDT-013
Validation será distinta de completion.

## DB-MIGRATION-ZDT-014
Cutover será distinto de contract.

## DB-MIGRATION-ZDT-015
Rollback será distinto de forward recovery.

## DB-MIGRATION-ZDT-016
Migration phase será distinta de batch.

## DB-MIGRATION-ZDT-017
Zero-downtime se evaluará sobre transición completa.

## DB-MIGRATION-ZDT-018
System state incluirá application/schema/data/traffic state.

## DB-MIGRATION-ZDT-019
Expand preservará old application compatibility cuando el strategy contract lo requiera.

## DB-MIGRATION-ZDT-020
Additive change no será automáticamente safe.

## DB-MIGRATION-ZDT-021
Backfill será first-class.

## DB-MIGRATION-ZDT-022
Validation precederá cutover cuando sea requerida.

## DB-MIGRATION-ZDT-023
Cutover cambiará authority explícitamente.

## DB-MIGRATION-ZDT-024
Stabilization podrá preceder contract.

## DB-MIGRATION-ZDT-025
Contract será tratado como potentially destructive.

## DB-MIGRATION-ZDT-026
Contract requerirá evidence de ausencia de legacy consumers cuando aplique.

## DB-MIGRATION-ZDT-027
Una phase podrá contener múltiples migrations.

## DB-MIGRATION-ZDT-028
Una phase podrá contener múltiples batches.

## DB-MIGRATION-ZDT-029
Una phase podrá cruzar deployments.

## DB-MIGRATION-ZDT-030
Database Migration no desplegará aplicaciones por sí mismo.

## DB-MIGRATION-ZDT-031
Compatibility window será explícita.

## DB-MIGRATION-ZDT-032
Compatibility result no será boolean simplista.

## DB-MIGRATION-ZDT-033
Read/write compatibility serán dimensiones separadas.

## DB-MIGRATION-ZDT-034
Dual write no implicará atomicity.

## DB-MIGRATION-ZDT-035
Dual-write authority será explícita.

## DB-MIGRATION-ZDT-036
Bidirectional synchronization será restringida.

## DB-MIGRATION-ZDT-037
Data authority transition será explícita.

## DB-MIGRATION-ZDT-038
Shadow column será strategy, no default universal.

## DB-MIGRATION-ZDT-039
Shadow table será strategy explícita.

## DB-MIGRATION-ZDT-040
Shadow strategy expondrá resource cost.

## DB-MIGRATION-ZDT-041
Backfill tendrá stable ordering strategy.

## DB-MIGRATION-ZDT-042
Offset pagination no será default para mutable large datasets.

## DB-MIGRATION-ZDT-043
Backfill resume token será typed.

## DB-MIGRATION-ZDT-044
Resumable backfill considerará idempotency.

## DB-MIGRATION-ZDT-045
Idempotency será distinta de replayability.

## DB-MIGRATION-ZDT-046
Checkpoint no precederá durable effect.

## DB-MIGRATION-ZDT-047
Backfill throttling será policy-driven.

## DB-MIGRATION-ZDT-048
Adaptive throttling será bounded.

## DB-MIGRATION-ZDT-049
Replica lag podrá pausar backfill.

## DB-MIGRATION-ZDT-050
Replica lag será operational metadata.

## DB-MIGRATION-ZDT-051
Phase validation será first-class.

## DB-MIGRATION-ZDT-052
UNKNOWN validation no será PASS.

## DB-MIGRATION-ZDT-053
Sample validation no implicará exhaustive validation.

## DB-MIGRATION-ZDT-054
Validation strength será explícita.

## DB-MIGRATION-ZDT-055
Constraint validation strategy será capability-driven.

## DB-MIGRATION-ZDT-056
Online index no implicará zero impact.

## DB-MIGRATION-ZDT-057
Index strategy será explícita.

## DB-MIGRATION-ZDT-058
Nullability tightening considerará existing data.

## DB-MIGRATION-ZDT-059
Unique constraint considerará duplicates.

## DB-MIGRATION-ZDT-060
Foreign key considerará orphans.

## DB-MIGRATION-ZDT-061
Type change podrá requerir shadow strategy.

## DB-MIGRATION-ZDT-062
Primary-key changes serán high-risk.

## DB-MIGRATION-ZDT-063
Table scale será operational metadata.

## DB-MIGRATION-ZDT-064
Estimates conservarán provenance/certainty.

## DB-MIGRATION-ZDT-065
Strategy selection será evidence-based.

## DB-MIGRATION-ZDT-066
Strategy result conservará alternatives.

## DB-MIGRATION-ZDT-067
Direct DDL solo será zero-downtime con evidencia suficiente.

## DB-MIGRATION-ZDT-068
Lock impact será first-class.

## DB-MIGRATION-ZDT-069
Lock estimate no será guarantee.

## DB-MIGRATION-ZDT-070
Lock timeout será distinto de migration timeout.

## DB-MIGRATION-ZDT-071
Application DB contract será distinto de batch sequence.

## DB-MIGRATION-ZDT-072
Compatibility overlap será verificable.

## DB-MIGRATION-ZDT-073
Application manifest no convertirá Database en Deployment Manager.

## DB-MIGRATION-ZDT-074
Deployment barriers serán first-class.

## DB-MIGRATION-ZDT-075
Barrier satisfaction tendrá evidence.

## DB-MIGRATION-ZDT-076
Deployment completion no implicará old worker drainage.

## DB-MIGRATION-ZDT-077
Persistent workers serán considerados antes de contract.

## DB-MIGRATION-ZDT-078
Cutover será first-class.

## DB-MIGRATION-ZDT-079
Read cutover será distinto de write cutover.

## DB-MIGRATION-ZDT-080
Cutover tendrá preconditions.

## DB-MIGRATION-ZDT-081
Cutover tendrá postconditions.

## DB-MIGRATION-ZDT-082
Rollback window será first-class.

## DB-MIGRATION-ZDT-083
Rollback window no dependerá solo del tiempo.

## DB-MIGRATION-ZDT-084
Contract podrá cerrar rollback window.

## DB-MIGRATION-ZDT-085
Rollback-window closure será visible.

## DB-MIGRATION-ZDT-086
Forward recovery será first-class.

## DB-MIGRATION-ZDT-087
Contract podrá retrasarse entre releases.

## DB-MIGRATION-ZDT-088
ZeroDowntimePlan será immutable.

## DB-MIGRATION-ZDT-089
Plan será distinto de execution.

## DB-MIGRATION-ZDT-090
Phase dependencies serán explícitas.

## DB-MIGRATION-ZDT-091
Phase checkpoints serán durable.

## DB-MIGRATION-ZDT-092
Phase checkpoint será distinto de batch record.

## DB-MIGRATION-ZDT-093
Resume reconciliará repository y phase state.

## DB-MIGRATION-ZDT-094
Resume verificará plan fingerprint.

## DB-MIGRATION-ZDT-095
Resume verificará compatibility assumptions.

## DB-MIGRATION-ZDT-096
Stale plan no será resumed ciegamente.

## DB-MIGRATION-ZDT-097
Safe pause boundaries serán explícitas.

## DB-MIGRATION-ZDT-098
Cancellation no implicará original-state restoration.

## DB-MIGRATION-ZDT-099
Cancellation reportará current authority.

## DB-MIGRATION-ZDT-100
Deadlines serán jerárquicos.

## DB-MIGRATION-ZDT-101
No existirá global transaction para toda estrategia.

## DB-MIGRATION-ZDT-102
Transactions serán local boundaries.

## DB-MIGRATION-ZDT-103
Failure certainty será preservada.

## DB-MIGRATION-ZDT-104
Backfill failure deberá ser resumable cuando contract lo permita.

## DB-MIGRATION-ZDT-105
Cutover failure deberá preservar/identificar authority.

## DB-MIGRATION-ZDT-106
Contract failure podrá ser irreversible.

## DB-MIGRATION-ZDT-107
Retryable Failure será distinto de Safe Retry.

## DB-MIGRATION-ZDT-108
Safety decision pertenecerá al Safety System.

## DB-MIGRATION-ZDT-109
Zero-downtime classification no implicará safety approval.

## DB-MIGRATION-ZDT-110
Resource budgets serán first-class.

## DB-MIGRATION-ZDT-111
Resource pressure podrá pausar/throttlear.

## DB-MIGRATION-ZDT-112
Temporary storage requirement será visible.

## DB-MIGRATION-ZDT-113
Unknown capacity no será assumed sufficient.

## DB-MIGRATION-ZDT-114
Telemetry será first-class.

## DB-MIGRATION-ZDT-115
Tracing correlacionará plan/phase/migration/batch/backfill.

## DB-MIGRATION-ZDT-116
Events no saltarán validation gates.

## DB-MIGRATION-ZDT-117
Events no cambiarán data authority.

## DB-MIGRATION-ZDT-118
Events no ejecutarán contract destructivo sin autorización.

## DB-MIGRATION-ZDT-119
Least privilege será preferido.

## DB-MIGRATION-ZDT-120
Trigger strategy será visible.

## DB-MIGRATION-ZDT-121
Application dual write será external requirement explícito.

## DB-MIGRATION-ZDT-122
Old queue jobs serán considerados antes de contract.

## DB-MIGRATION-ZDT-123
Old scheduled workers serán considerados antes de contract.

## DB-MIGRATION-ZDT-124
Persistent runtime generation podrá integrarse como evidence.

## DB-MIGRATION-ZDT-125
RuntimeManagerServer será integración opcional.

## DB-MIGRATION-ZDT-126
Multitenancy será integración opcional.

## DB-MIGRATION-ZDT-127
Tenant phases serán aisladas.

## DB-MIGRATION-ZDT-128
No habrá global mutable current tenant phase.

## DB-MIGRATION-ZDT-129
Canary success no probará global safety.

## DB-MIGRATION-ZDT-130
Shard rolling migration no asumirá cross-shard atomicity.

## DB-MIGRATION-ZDT-131
Application deberá tolerar mixed shard states cuando strategy lo requiera.

## DB-MIGRATION-ZDT-132
Platform advisors serán distintos de planner.

## DB-MIGRATION-ZDT-133
SQLite no fingirá equivalencia operacional con otros motores.

## DB-MIGRATION-ZDT-134
MySQL y MariaDB serán evaluados separadamente.

## DB-MIGRATION-ZDT-135
MariaDB será first-class.

## DB-MIGRATION-ZDT-136
Capability snapshot será immutable.

## DB-MIGRATION-ZDT-137
Version será distinta de capability.

## DB-MIGRATION-ZDT-138
Sensitive phase podrá revalidar capabilities explícitamente.

## DB-MIGRATION-ZDT-139
Plan generation será deterministic para mismos inputs.

## DB-MIGRATION-ZDT-140
Operational assessment será distinto de structural plan.

## DB-MIGRATION-ZDT-141
Operational metadata tendrá freshness.

## DB-MIGRATION-ZDT-142
Reassessment podrá ocurrir antes de expensive operations.

## DB-MIGRATION-ZDT-143
No habrá hidden table rebuild.

## DB-MIGRATION-ZDT-144
No habrá hidden trigger synchronization.

## DB-MIGRATION-ZDT-145
No habrá hidden application requirement.

## DB-MIGRATION-ZDT-146
No habrá hidden destructive contract.

## DB-MIGRATION-ZDT-147
No habrá blind resume después de unknown outcome.

## DB-MIGRATION-ZDT-148
No habrá blind contract mientras legacy consumers puedan existir.

## DB-MIGRATION-ZDT-149
Toda pérdida de compatibility será explícita.

## DB-MIGRATION-ZDT-150
VoltStack nunca calificará una migración como zero-downtime únicamente porque el motor pueda ejecutar su DDL en línea.

---

# 215. Anti-patterns

## 215.1 Online DDL = zero downtime

Incorrecto:

```text
supports online ALTER
⇒
zero downtime
```

---

## 215.2 Rename directo durante rolling deployment

Incorrecto:

```text
V1 reads name
V2 reads full_name

RENAME name → full_name
```

sin compatibility window.

---

## 215.3 Add NOT NULL inmediatamente

Incorrecto:

```text
ADD column NOT NULL
```

sobre tabla grande con datos existentes sin analizar rewrite/backfill/locking.

---

## 215.4 Giant transaction

Incorrecto:

```text
BEGIN
expand
backfill 500M rows
cutover
contract
COMMIT
```

---

## 215.5 Dual write sin authority

Incorrecto:

```text
write old
write new
```

sin definir cuál representación gana ante divergencia.

---

## 215.6 Backfill sin checkpoint

Incorrecto para grandes datasets:

```text
process 500M rows
```

como una operación indivisible.

---

## 215.7 Contract después del deployment

Incorrecto:

```text
new deployment complete
⇒
DROP old column
```

sin verificar workers/jobs legacy.

---

## 215.8 Sampling = proof

Incorrecto:

```text
1000 rows matched
⇒
all rows match
```

---

## 215.9 Hidden trigger

Incorrecto:

```text
planner silently creates synchronization trigger
```

---

## 215.10 Zero downtime = zero impact

Incorrecto:

```text
no outage
⇒
no production impact
```

---

# 216. Ejemplo completo — rename de columna

Objetivo:

```text
users.name
→
users.full_name
```

## Fase 1 — EXPAND

```text
ADD users.full_name NULL
```

Schema:

```text
users
├── name
└── full_name
```

V1 continúa usando:

```text
name
```

---

## Fase 2 — DUAL COMPATIBILITY

V2 escribe:

```text
name
+
full_name
```

o mantiene una autoridad definida.

---

## Fase 3 — BACKFILL

```text
full_name = name
```

por chunks.

---

## Fase 4 — VALIDATE

```text
count(name != full_name) = 0
```

bajo equivalencia declarada.

---

## Fase 5 — READ CUTOVER

V2:

```text
read full_name
fallback name
```

Después:

```text
read full_name only
```

---

## Fase 6 — WRITE CUTOVER

```text
authoritative = full_name
```

---

## Fase 7 — STABILIZE

Observar:

```text
errors
mismatches
legacy reads
legacy writes
```

---

## Fase 8 — CONTRACT

Cuando:

```text
NoLegacyConsumer(name)
```

entonces:

```text
DROP name
```

---

# 217. Ejemplo — nullable a NOT NULL

Current:

```text
users.phone NULL
```

Target:

```text
users.phone NOT NULL
```

Plan:

```text
Application stops producing NULL
        ↓
Backfill existing NULL
        ↓
Validate NULL count = 0
        ↓
Enforce NOT NULL
```

No:

```text
ALTER NOT NULL
```

directamente.

---

# 218. Ejemplo — cambio de tipo

Current:

```text
orders.amount VARCHAR
```

Target:

```text
orders.amount_decimal DECIMAL
```

Plan:

```text
ADD amount_decimal
        ↓
Dual-compatible writes
        ↓
Backfill conversion
        ↓
Validate conversion
        ↓
Read cutover
        ↓
Write authority cutover
        ↓
Observe
        ↓
DROP amount
```

---

# 219. Ejemplo — índice grande

Objetivo:

```text
INDEX orders(created_at)
```

Plan puede seleccionar:

```text
online/concurrent build
```

si capability lo permite.

Aun deberá evaluar:

```text
I/O
CPU
temporary disk
replica lag
lock behavior
```

---

# 220. Ejemplo — shadow table

Current:

```text
events_v1
```

Target requiere reorganización incompatible.

Plan:

```text
CREATE events_v2
        ↓
COPY historical data
        ↓
Synchronize new writes
        ↓
Catch up
        ↓
Validate
        ↓
Cutover reads/writes
        ↓
Observe
        ↓
Retire events_v1
```

---

# 221. Fórmula de compatibilidad transicional

Para una fase `P`:

```text
TransitionCompatible(P)
=
OldApplicationCompatible(P)
∧
NewApplicationCompatible(P)
∧
DataContractPreserved(P)
```

según los contracts requeridos para esa fase.

---

# 222. Fórmula zero-downtime

Conceptualmente:

```text
ZeroDowntime(plan)
=
∀ transition t ∈ plan:
    RequiredTrafficContinuity(t)
    ∧ RequiredApplicationCompatibility(t)
    ∧ RequiredDataIntegrity(t)
    ∧ NoUnboundedBlocking(t)
```

No implica impacto cero.

---

# 223. Fórmula de contract safety

```text
ContractAllowed
=
TargetStateValidated
∧
NoRequiredLegacyConsumer
∧
RollbackPolicySatisfied
∧
SafetyPolicySatisfied
∧
OperationalPreconditionsSatisfied
```

---

# 224. Fórmula de backfill resume

```text
SafeBackfillResume
=
CheckpointValid
∧
CompletedEffectsKnown
∧
RemainingOperationReplayable
∧
SourceSemanticsStillValid
∧
TargetSemanticsStillValid
```

---

# 225. Fórmula de rollback window

```text
RollbackWindowOpen
=
OldStructureAvailable
∧
OldRepresentationSufficientlySynchronized
∧
OldApplicationCompatible
∧
NoIrreversibleContractExecuted
```

---

# 226. Fórmula de cutover

```text
CutoverAllowed
=
BackfillSatisfied
∧
ValidationSatisfied
∧
ApplicationBarrierSatisfied
∧
OperationalHealthAcceptable
∧
AuthorityTransitionDefined
```

---

# 227. Fórmula maestra

```text
Zero-Downtime Migration System
=
Expand/Contract Architecture
+
Application Compatibility Windows
+
Schema Compatibility
+
Data Authority Transitions
+
Dual Read/Write Contracts
+
Resumable Backfills
+
Validation Gates
+
Platform-Aware Online Strategies
+
Deployment Barriers
+
Explicit Cutovers
+
Rollback Windows
+
Contract Safety
+
Resource Governance
+
Replica Awareness
+
Persistent Runtime Awareness
+
Phase Checkpoints
+
Recovery
+
Telemetry
+
Migration Safety Integration
```

---

# 228. Regla arquitectónica maestra

> **VoltStack considerará una migración zero-downtime únicamente cuando pueda demostrar una secuencia explícita de estados intermedios en la que las aplicaciones, el esquema y los datos mantengan los contratos requeridos mientras el tráfico continúa siendo atendido.**

La secuencia conceptual será:

```text
Current State
      ↓
PREPARE
      ↓
EXPAND
      ↓
Compatibility Window
      ↓
BACKFILL
      ↓
VALIDATE
      ↓
CUTOVER
      ↓
STABILIZE
      ↓
CONTRACT
      ↓
Final State
```

y no:

```text
Generate clever ALTER TABLE
↓
call it zero downtime
```

---

# 229. Resultado arquitectónico

Con este diseño VoltStack podrá soportar:

```text
rolling application deployments
expand/contract migrations
large-table backfills
resumable data transformations
shadow-column migrations
shadow-table migrations
online index strategies
phased constraints
dual-read transitions
dual-write transitions
application cutovers
rollback windows
forward recovery
persistent-worker awareness
replica-aware throttling
tenant/shard rolling migrations
platform-specific online strategies
```

sin introducir una falsa garantía de:

```text
zero locks
zero impact
global atomicity
automatic rollback
```

---

# 230. Relación con documentos anteriores

Este sistema integra especialmente:

```text
84_DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM.md
85_DATABASE_EXECUTION_ERROR_SYSTEM.md
86_DATABASE_EXECUTION_RETRY_SYSTEM.md

87_DATABASE_SCHEMA_ARCHITECTURE.md
98_DATABASE_SCHEMA_DIFF_SYSTEM.md
99_DATABASE_SCHEMA_COMPILER_SYSTEM.md
100_DATABASE_SCHEMA_PLATFORM_COMPATIBILITY_SYSTEM.md

101_DATABASE_MIGRATION_ARCHITECTURE.md
102_DATABASE_MIGRATION_SYSTEM.md
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM.md
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
106_DATABASE_MIGRATION_EXECUTION_SYSTEM.md
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM.md
108_DATABASE_MIGRATION_BATCH_SYSTEM.md
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM.md
```

---

# 231. Siguiente documento

```text
111_DATABASE_MIGRATION_SAFETY_SYSTEM.md
```

El siguiente documento cerrará el **Bloque 9 — Migrations** formalizando la capa que deberá decidir si una migración puede ejecutarse bajo las políticas y condiciones actuales.

Deberá distinguir:

```text
Migration Safety
≠
Schema Compatibility

Migration Safety
≠
Authorization

Migration Safety
≠
Zero Downtime

Migration Safety
≠
Transaction Safety

Risk
≠
Failure

Hazard
≠
Risk

Destructive
≠
Forbidden

Safe
≠
Risk Free

Policy Approval
≠
Technical Compatibility

Warning
≠
Safety Gate

Force
≠
Ignore Correctness
```

y deberá cubrir:

```text
risk classification
hazard model
destructive operations
data-loss detection
table-lock risk
rewrite risk
large-table risk
transaction risk
rollback/reversibility risk
unknown-outcome risk
replication risk
resource exhaustion
backup requirements
preflight checks
safety policies
approval gates
environment policies
production protections
migration budgets
force/override semantics
safety reports
safety fingerprints
runtime revalidation
zero-downtime integration
telemetry
audit
extension governance
```

Con `111_DATABASE_MIGRATION_SAFETY_SYSTEM.md` quedará cerrado el bloque:

```text
101–111
DATABASE MIGRATIONS
```

y el siguiente bloque comenzará con:

```text
112_DATABASE_ORM_ARCHITECTURE.md
```