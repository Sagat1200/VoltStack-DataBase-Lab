# 104_DATABASE_MIGRATION_REPOSITORY_SYSTEM.md

# VoltStack Quantum Database
## Database Migration Repository System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 104 — Database Migration Repository System  
**Bloque:** 9 — Migrations  
**Estado:** Architecture Specification  
**Versión:** 1.0

---

# 1. Propósito

`Database Migration Repository System` define la arquitectura mediante la cual VoltStack conserva, consulta, valida y protege el **historial persistente de migraciones aplicadas**.

Este sistema responde una pregunta distinta a Discovery:

> **¿Qué migraciones conoce la base de datos como aplicadas, en qué estado, con qué identidad, checksum, batch y metadata de ejecución?**

La relación principal será:

```text id="w7ap01"
Migration Catalog
=
what migrations are available
```

mientras:

```text id="2xn27m"
Migration Repository
=
what migration history the database records
```

Por tanto:

```text id="qe2c53"
Migration Discovery
≠
Migration Repository
```

---

# 2. Principio central

> **El Migration Repository es el registro persistente autoritativo del historial aceptado de migraciones para un target de base de datos; no es un catálogo de definiciones ni un ejecutor de migraciones.**

Formalmente:

```text id="nml0l4"
Migration Repository
=
Persistent Migration History
```

No:

```text id="p623vm"
Migration Repository
=
Migration Files
+
SQL Executor
```

---

# 3. Responsabilidades

El Repository System deberá:

- persistir migraciones aplicadas;
- conservar identidad calificada;
- almacenar checksum semántico;
- registrar batch;
- registrar timestamps;
- mantener metadata de aplicación;
- generar snapshots consistentes;
- detectar inconsistencias;
- soportar repository bootstrap;
- soportar repository schema versioning;
- mantener historial de intentos cuando la policy lo requiera;
- distinguir ejecución exitosa de ejecución fallida;
- soportar estados parciales/unknown cuando sea necesario;
- coordinar escrituras concurrentes;
- validar integridad del repository;
- soportar drift detection;
- soportar múltiples namespaces;
- mantenerse aislado en persistent runtimes.

---

# 4. No responsabilidades

El Migration Repository no deberá:

```text id="ko9hy8"
discover migration files
load MigrationDefinition
plan migrations
compile Schema AST
execute DDL
execute data migration
choose rollback strategy
decide migration safety
infer platform compatibility
manage arbitrary application transactions
```

---

# 5. Posición arquitectónica

```text id="rojs5f"
Migration Sources
      │
      ▼
Migration Discovery
      │
      ▼
Migration Catalog
      │
      │
      │         Database
      │            │
      │            ▼
      │    Migration Repository
      │            │
      │            ▼
      │     Repository Snapshot
      │            │
      └──────┬─────┘
             ▼
      Migration Planner
             │
             ▼
      Migration Executor
             │
             ▼
Repository Update
```

---

# 6. Distinciones fundamentales

VoltStack deberá mantener:

```text id="avwl58"
Migration Repository
≠
Migration Catalog

Repository Record
≠
Migration Definition

Applied Migration
≠
Available Migration

Repository State
≠
Execution Attempt State

Migration Batch
≠
Transaction

Repository Checksum
≠
Source Checksum

Repository Snapshot
≠
Live Repository Session
```

---

# 7. Contrato principal

Se propone:

```php id="fgza3r"
interface MigrationRepository
{
    public function snapshot(
        MigrationRepositoryContext $context
    ): MigrationRepositorySnapshot;

    public function recordApplied(
        AppliedMigrationRecord $record,
        MigrationRepositoryWriteContext $context,
    ): void;

    public function recordRolledBack(
        MigrationIdentity $migration,
        MigrationRepositoryWriteContext $context,
    ): void;
}
```

El contrato final podrá refinarse, pero deberá separar claramente:

```text id="sc1p8n"
read
write
bootstrap
validation
history
```

---

# 8. Read model vs write model

Se recomienda diferenciar:

```text id="6azvg9"
MigrationRepositoryReader
```

de:

```text id="x6491t"
MigrationRepositoryWriter
```

Ejemplo:

```php id="d2msty"
interface MigrationRepositoryReader
{
    public function snapshot(
        MigrationRepositoryReadContext $context
    ): MigrationRepositorySnapshot;
}
```

```php id="s6crpa"
interface MigrationRepositoryWriter
{
    public function appendApplied(
        AppliedMigrationRecord $record,
        MigrationRepositoryWriteContext $context
    ): void;

    public function appendExecution(
        MigrationExecutionRecord $record,
        MigrationRepositoryWriteContext $context
    ): void;
}
```

Esto reduce responsabilidades ambiguas.

---

# 9. Repository implementation

Implementación predeterminada:

```text id="9ucydd"
DatabaseMigrationRepository
```

basada en tablas internas de VoltStack.

Otras implementaciones podrán existir:

```text id="zxmap1"
InMemoryMigrationRepository
TestingMigrationRepository
ExtensionMigrationRepository
```

pero deberán respetar el mismo contrato.

---

# 10. Repository storage

El repository predeterminado vivirá en la misma base lógica de la aplicación salvo configuración distinta.

Ejemplo conceptual:

```text id="g37m64"
voltstack_migrations
```

pero el nombre final será configurable.

---

# 11. Internal schema

Debe distinguirse:

```text id="zwjwio"
Framework Internal Schema
≠
Application Schema
```

El repository es infraestructura interna del framework.

---

# 12. Namespace interno

En plataformas que soporten schema namespaces, podría utilizarse:

```text id="lbai4w"
voltstack
```

o un namespace configurable.

Pero no deberá exigirse universalmente.

---

# 13. Repository schema mínimo

Conceptualmente:

```text id="l1zqml"
migration_repository
├── namespace
├── migration_id
├── semantic_checksum
├── checksum_version
├── batch_id
├── applied_at
├── execution_id
├── repository_version
└── metadata
```

---

# 14. Tabla recomendada

Conceptualmente:

```sql id="tfvldc"
voltstack_migrations
```

con columnas semejantes a:

```text id="8j23vk"
namespace
migration_id
checksum
checksum_algorithm
checksum_version
batch_sequence
execution_id
applied_at
source_kind?
metadata?
```

La definición física exacta será generada por Schema System.

---

# 15. Qualified migration identity

La clave lógica será:

```text id="k4k86d"
MigrationIdentity
=
MigrationNamespace
+
MigrationId
```

Por ejemplo:

```text id="f1d0sm"
app:
2026_09_06_010000_create_users
```

y:

```text id="6z71ex"
package.acme.billing:
2026_09_06_020000_create_invoices
```

son identidades distintas.

---

# 16. Repository key

No utilizar únicamente:

```text id="e7r4bb"
migration filename
```

como primary identity.

---

# 17. AppliedMigrationRecord

Se propone:

```php id="5zvhtj"
final readonly class AppliedMigrationRecord
{
    public function __construct(
        public MigrationIdentity $migration,
        public MigrationSemanticChecksum $checksum,
        public MigrationBatchId $batch,
        public MigrationExecutionId $execution,
        public DateTimeImmutable $appliedAt,
        public AppliedMigrationMetadata $metadata,
    ) {}
}
```

---

# 18. Immutable record

Una vez persistido:

```text id="0x5jv6"
AppliedMigrationRecord
```

deberá considerarse immutable conceptualmente.

Correcciones deberán realizarse mediante operaciones explícitas de repository maintenance, no mutación silenciosa.

---

# 19. Applied ≠ execution attempt

Debe mantenerse:

```text id="08b7wu"
AppliedMigrationRecord
```

para historial aceptado,

y:

```text id="c0dm7a"
MigrationExecutionRecord
```

para intentos individuales.

---

# 20. Execution records

Se propone opcionalmente una segunda estructura:

```text id="lp8wmm"
voltstack_migration_executions
```

para conservar:

```text id="cfhe3e"
execution ID
migration identity
started_at
finished_at
status
outcome certainty
failure category
attempt metadata
```

---

# 21. Execution history optionality

El historial detallado podrá ser:

```text id="w08ckp"
MINIMAL
STANDARD
AUDIT
```

según configuración.

Pero el registro de migraciones aplicadas será obligatorio.

---

# 22. MigrationExecutionRecord

```php id="upk8bw"
final readonly class MigrationExecutionRecord
{
    public function __construct(
        public MigrationExecutionId $executionId,
        public MigrationIdentity $migration,
        public MigrationExecutionState $state,
        public MigrationOutcomeCertainty $certainty,
        public DateTimeImmutable $startedAt,
        public ?DateTimeImmutable $finishedAt,
        public MigrationExecutionRecordMetadata $metadata,
    ) {}
}
```

---

# 23. Success record

Una migración debe aparecer en:

```text id="pgsqy7"
Applied Migration History
```

solo si cumple el execution contract.

---

# 24. Failure record

Una ejecución fallida:

```text id="isvtdf"
FAILED
```

no deberá insertar automáticamente:

```text id="x57yus"
AppliedMigrationRecord
```

---

# 25. Partial execution

Si una migración queda:

```text id="w19o4b"
PARTIALLY_APPLIED
```

su estado deberá quedar representado en execution history / recovery state.

No debe fingirse:

```text id="1b9k7h"
APPLIED
```

ni:

```text id="xl5na2"
NOTHING HAPPENED
```

---

# 26. Unknown outcome

Si outcome es:

```text id="yfy0hi"
UNKNOWN
```

Repository deberá conservar esa incertidumbre cuando la configuración de recovery lo soporte.

---

# 27. Repository state model

Se propone:

```text id="mk3u5y"
MigrationRepositoryState
├── APPLIED
├── NOT_APPLIED
├── DRIFTED
├── MISSING_DEFINITION
├── PARTIALLY_APPLIED
├── UNKNOWN
└── CORRUPTED
```

Pero algunos de estos son **reconciliation states**, no filas físicas.

---

# 28. Stored state vs derived state

Debe distinguirse:

```text id="ijpio4"
Stored Repository Facts
```

de:

```text id="fbzj0w"
Derived Migration State
```

Ejemplo:

```text id="j9prl1"
Stored:
M1 applied with checksum A
```

más Catalog:

```text id="c656yx"
M1 current checksum B
```

produce:

```text id="zi4z8v"
Derived:
DRIFTED
```

---

# 29. Repository Snapshot

Se propone:

```php id="n1t5le"
final readonly class MigrationRepositorySnapshot
{
    public function __construct(
        public AppliedMigrationRecordSet $applied,
        public MigrationExecutionRecordSet $executions,
        public MigrationRepositoryVersion $version,
        public MigrationRepositorySnapshotMetadata $metadata,
        public MigrationRepositoryFingerprint $fingerprint,
    ) {}
}
```

---

# 30. Snapshot semantics

Un snapshot deberá representar un punto lógico consistente del repository.

```text id="x57lpu"
Snapshot
≠
live mutable cursor
```

---

# 31. Snapshot immutability

Después de crearse:

```text id="g7co19"
MigrationRepositorySnapshot
```

será immutable.

---

# 32. Snapshot consistency

Idealmente todas las filas del snapshot deberán leerse bajo una vista consistente.

Según plataforma:

```text id="q0jtkj"
transaction snapshot
stable query sequence
repository lock
```

podrán utilizarse.

---

# 33. RepositoryFingerprint

```text id="i7tiwn"
MigrationRepositoryFingerprint
=
H(
    applied migration identities,
    semantic checksums,
    batch information,
    repository version,
    relevant execution states
)
```

---

# 34. Fingerprint use

Puede utilizarse para:

```text id="f6rtdd"
planning cache
drift detection
concurrency verification
audit
diagnostics
```

---

# 35. Fingerprint ≠ locking

Un fingerprint no reemplaza mecanismos de exclusión.

---

# 36. MigrationSemanticChecksum

Debe corresponder al checksum definido en el documento 102.

```text id="64q551"
Repository Checksum
=
Semantic Migration Checksum at application time
```

---

# 37. Source checksum

Puede almacenarse opcionalmente:

```text id="zuxmmw"
source_checksum
```

para debugging.

Pero:

```text id="8nk0qa"
source checksum
≠
semantic repository checksum
```

---

# 38. Checksum algorithm

Debe persistirse:

```text id="77xwri"
algorithm
version
value
```

No únicamente el hash.

---

# 39. Checksum upgrade

Si VoltStack cambia algoritmo:

```text id="9cangq"
checksum v1
→
checksum v2
```

el Repository System debe poder reconocer ambos.

---

# 40. No silent checksum rewrite

Nunca:

```text id="fk5lcq"
new framework version
→ overwrite historical checksum
```

sin operación de mantenimiento explícita.

---

# 41. Drift detection

Al reconciliar:

```text id="7l67z9"
Repository checksum A
Current definition checksum B
```

si:

```text id="71jd7i"
A ≠ B
```

se produce:

```text id="7rew71"
MIGRATION_DEFINITION_DRIFT
```

---

# 42. Drift categories

Se propone:

```text id="rfqywl"
UNCHANGED
SEMANTICALLY_MODIFIED
SOURCE_ONLY_MODIFIED
MISSING_DEFINITION
UNKNOWN
```

---

# 43. Semantic drift

Ejemplo:

```text id="u6k165"
Applied migration:
VARCHAR(255)

Current definition:
VARCHAR(320)
```

Debe producir:

```text id="zl6etr"
SEMANTICALLY_MODIFIED
```

---

# 44. Source-only drift

Ejemplo:

```text id="ft1wzo"
only comments/whitespace changed
```

puede producir:

```text id="0s20cc"
SOURCE_ONLY_MODIFIED
```

si se almacenó source checksum.

---

# 45. Missing definition

Repository tiene:

```text id="pthp7g"
app:M42
```

pero Catalog no.

Esto produce:

```text id="zgtgyb"
MISSING_DEFINITION
```

como reconciliación.

---

# 46. Missing definition ≠ rollback

Regla crítica:

```text id="bap4uj"
Missing Definition
≠
Rolled Back Migration
```

---

# 47. Missing definition possible causes

Puede ocurrir por:

```text id="e64ysx"
package removal
artifact mismatch
bad deployment
catalog partial
manual file deletion
branch divergence
```

El Repository System no adivina la causa.

---

# 48. Repository bootstrap

Antes de registrar migraciones:

```text id="ewiwyw"
repository schema must exist
```

Por eso existirá:

```text id="xndnvp"
MigrationRepositoryBootstrapper
```

---

# 49. Bootstrap responsibilities

El bootstrapper podrá:

```text id="6sl7re"
detect repository presence
create minimal internal tables
validate repository schema version
perform framework-internal repository upgrade
```

---

# 50. Bootstrapper ≠ Migration Engine

El bootstrapper no ejecutará migraciones de aplicación.

---

# 51. Bootstrapper ≠ general Schema Builder API

Aunque internamente utilice Schema subsystem:

```text id="qp6jlw"
MigrationRepositoryBootstrapper
      ↓
Schema AST / Compiler / Execution
```

su scope será únicamente el repository interno.

---

# 52. RepositoryVersion

Se propone:

```php id="d3mq08"
final readonly class MigrationRepositoryVersion
{
    public function __construct(
        public int $major,
        public int $minor,
    ) {}
}
```

---

# 53. Repository schema version

Diferente de:

```text id="syj13r"
migration version
framework version
database version
```

---

# 54. Repository evolution

Ejemplo:

```text id="03y0ig"
Repository v1:
migration + batch

Repository v2:
namespace + checksum

Repository v3:
execution IDs + metadata
```

---

# 55. Repository upgrade

Debe existir mecanismo separado:

```text id="hx4qvr"
MigrationRepositoryUpgrader
```

para evolucionar estructuras internas de VoltStack.

---

# 56. Internal repository upgrade

No debe aparecer como migration de aplicación normal salvo decisión explícita.

---

# 57. Repository schema compatibility

Al iniciar:

```text id="9ux6vn"
Current Repository Version
+
Required Repository Version
```

produce:

```text id="bfr0qa"
COMPATIBLE
UPGRADE_REQUIRED
TOO_NEW
CORRUPTED
UNKNOWN
```

---

# 58. Too-new repository

Si runtime antiguo encuentra repository más nuevo:

```text id="c03jms"
repository version > supported version
```

deberá fallar de manera segura.

Nunca intentar interpretar campos desconocidos silenciosamente.

---

# 59. Repository state validation

Se propone:

```php id="kz7qyw"
interface MigrationRepositoryValidator
{
    public function validate(
        MigrationRepositorySnapshot $snapshot,
        MigrationRepositoryValidationContext $context
    ): MigrationRepositoryValidationReport;
}
```

---

# 60. Validation checks

Incluir:

```text id="no8p9z"
duplicate migration identity
invalid namespace
invalid checksum format
duplicate batch sequence anomalies
missing required values
unknown execution states
inconsistent timestamps
repository version mismatch
duplicate applied records
```

---

# 61. Duplicate applied record

No debe existir:

```text id="fdssj6"
same MigrationIdentity applied twice
```

en la tabla principal de applied migrations.

---

# 62. Retry history

Reintentos viven en:

```text id="6o6bnk"
execution history
```

no como múltiples applied rows.

---

# 63. Unique constraint

La tabla principal deberá imponer conceptualmente:

```text id="76alju"
UNIQUE(namespace, migration_id)
```

o equivalente.

---

# 64. Batch identity

Cada aplicación exitosa pertenece a:

```text id="c383um"
MigrationBatchId
```

o batch sequence.

---

# 65. Batch ≠ transaction

Regla:

```text id="wydi6f"
MigrationBatch
≠
DatabaseTransaction
```

Un batch puede abarcar:

```text id="kprrff"
multiple migrations
multiple transactions
non-transactional steps
```

---

# 66. Batch sequence

Se propone almacenar:

```text id="s9y18p"
batch sequence
```

monotónica dentro del repository target.

Ejemplo:

```text id="2009z2"
1
2
3
4
```

---

# 67. Batch ID

Además de sequence:

```text id="bn9vuo"
MigrationBatchId
```

puede ser un identificador estable de ejecución.

---

# 68. Batch sequence concurrency

Debe asignarse de forma segura.

Incorrecto:

```php id="3x9gfa"
$next = $repository->maxBatch() + 1;
```

sin coordinación.

---

# 69. Safe batch allocation

Puede utilizar:

```text id="alcy68"
repository transaction
row lock
database sequence
atomic insert strategy
migration coordination lock
```

según plataforma.

---

# 70. Repository writer concurrency

Aunque Migration Lock normalmente impida múltiples migradores:

```text id="cj6k2r"
Repository
```

deberá mantener integridad incluso ante carreras inesperadas.

---

# 71. Double apply protection

Si dos ejecutores intentan registrar:

```text id="u3wt2q"
app:M1
```

simultáneamente, la constraint de unicidad deberá impedir doble registro.

---

# 72. Optimistic repository verification

Un plan podrá registrar:

```text id="2qrnw1"
RepositoryFingerprint R₀
```

y antes de ejecutar verificar:

```text id="7ifj1l"
CurrentFingerprint == R₀
```

si la policy requiere detectar cambios concurrentes.

---

# 73. Repository changed during planning

Si:

```text id="roinpe"
planner snapshot = R₀
runtime repository = R₁
```

el plan puede estar stale.

Debe producir:

```text id="sk8puh"
MigrationRepositoryChangedException
```

o replanning.

---

# 74. Repository snapshot generation

Ideal:

```text id="r0e2gr"
Read Repository Metadata
Read Applied Migrations
Read Relevant Execution State
Validate
Canonicalize
Fingerprint
Seal Snapshot
```

---

# 75. Snapshot canonicalization

Records deben ordenarse determinísticamente, por ejemplo:

```text id="g5xnlh"
qualified migration identity ASC
```

para hashing y testing.

---

# 76. Repository ordering ≠ execution dependency

El orden canónico del snapshot es representacional.

No sustituye:

```text id="o7uh1n"
migration dependency graph
```

---

# 77. Applied order

Si se necesita orden histórico:

```text id="73y8r8"
batch sequence
+
migration sequence within batch
```

deberá almacenarse explícitamente.

---

# 78. Migration position within batch

Se recomienda:

```text id="fw1fyg"
batch_position
```

para reconstruir con precisión el orden histórico.

---

# 79. Applied timestamp

```text id="kbl4v9"
applied_at
```

es metadata histórica.

No debe utilizarse como identidad.

---

# 80. Timestamp source

Preferencia:

```text id="slm9n5"
database/server time strategy
```

o un Clock explícito.

No `now()` disperso.

---

# 81. Clock abstraction

Se propone:

```php id="9ruv6i"
interface MigrationClock
{
    public function now(): DateTimeImmutable;
}
```

para determinismo/testing.

---

# 82. Timestamp ≠ ordering proof

En sistemas distribuidos:

```text id="5fjiu0"
applied_at(A) < applied_at(B)
```

no es suficiente para demostrar dependencia.

---

# 83. Execution metadata

Podrá incluir:

```text id="l6q13g"
runtime
node ID
framework version
application release
duration
plan fingerprint
platform ID
capability fingerprint
```

según profile.

---

# 84. Metadata security

No persistir:

```text id="lcokvc"
password
DSN secrets
raw credentials
sensitive query payloads
```

---

# 85. Application release

Puede almacenarse opcionalmente:

```text id="j9h3r0"
release_id
```

para correlacionar migraciones con despliegues.

Pero:

```text id="u2d9sw"
Migration Repository
≠
Deployment Repository
```

---

# 86. Rollback record

Cuando una migración se revierte exitosamente, existen dos estrategias:

```text id="8yfbrg"
A. delete applied row
B. preserve historical event + active-state projection
```

---

# 87. Recommended model

Para arquitectura avanzada se recomienda:

```text id="x0kv6g"
Immutable History
+
Current Applied Projection
```

en vez de borrar toda evidencia histórica.

---

# 88. Event-oriented history

Conceptualmente:

```text id="5ti4mj"
MigrationHistoryEvent
├── APPLIED
├── ROLLED_BACK
├── FAILED
├── PARTIAL
└── RECOVERED
```

---

# 89. Current projection

A partir del historial:

```text id="82x8hg"
MigrationCurrentState
```

puede responder:

```text id="dh4u1q"
is currently applied?
```

---

# 90. Minimal v1 option

Para una V1 simplificada:

```text id="j4979m"
voltstack_migrations
```

puede contener solo migraciones actualmente aplicadas.

Pero la arquitectura deberá permitir ampliar a history sin romper contracts públicos.

---

# 91. Applied-history strategy

Se recomienda separar:

```text id="819uct"
Current Applied Table
```

y opcional:

```text id="eug0d8"
Execution/History Table
```

---

# 92. Rollback semantics

Cuando rollback completa:

```text id="1hj578"
Current Applied Projection
```

debe dejar de considerar la migration aplicada.

Pero su execution/history record puede permanecer.

---

# 93. Rollback failure

Si rollback falla parcialmente:

```text id="kj7e26"
do not simply delete applied record
```

porque el estado real puede ser incierto.

---

# 94. Recovery required

Debe existir posibilidad de:

```text id="viquf4"
REQUIRES_RECONCILIATION
```

cuando current repository state no puede representar con certeza el estado físico.

---

# 95. Repository reconciliation

Se propone:

```php id="t06dce"
interface MigrationRepositoryReconciler
{
    public function reconcile(
        MigrationCatalog $catalog,
        MigrationRepositorySnapshot $repository,
        MigrationReconciliationContext $context
    ): MigrationReconciliationReport;
}
```

---

# 96. Reconciliation outputs

```text id="o54hxp"
APPLIED_AND_MATCHING
APPLIED_BUT_DRIFTED
APPLIED_BUT_MISSING_DEFINITION
AVAILABLE_BUT_NOT_APPLIED
PARTIAL_EXECUTION
UNKNOWN_EXECUTION
REPOSITORY_CORRUPTION
```

---

# 97. Reconciliation ≠ planning

Reconciliation describe hechos/diferencias.

Planner decide acciones.

---

# 98. Repository corruption

Ejemplos:

```text id="4vv78c"
duplicate identity rows
invalid checksum
impossible state transition
unknown repository schema version
broken history reference
invalid batch reference
```

---

# 99. Corrupted repository

No deberá continuar automáticamente con migrations.

Debe producir:

```text id="d7p2j7"
MigrationRepositoryCorruptedException
```

o report equivalente.

---

# 100. Manual repository edits

Cambios manuales podrán detectarse como:

```text id="yb93u9"
corruption
drift
administrative modification
```

dependiendo del contexto.

No asumir que son válidos.

---

# 101. Administrative tooling

Futuro CLI podrá ofrecer:

```text id="fv6hrw"
migration repository inspect
migration repository repair
migration repository reconcile
```

pero acciones de repair deberán ser explícitas.

---

# 102. Repair ≠ normal migration

No mezclar:

```text id="3s7eg4"
Repository Repair
```

con:

```text id="ezyef4"
Application Migration
```

---

# 103. Repository transaction boundaries

Las escrituras al repository deberán coordinarse con ejecución cuando sea posible.

Caso ideal:

```text id="n9lzam"
BEGIN
  migration effects
  applied migration insert
COMMIT
```

---

# 104. Non-transactional DDL

En plataformas/operaciones donde no sea posible:

```text id="87mhig"
effect commit
        ↓
repository write
```

no puede ser atómico.

La arquitectura debe preservar esa realidad.

---

# 105. Write-ahead execution state

Para operaciones no transaccionales puede usarse:

```text id="c81qnc"
execution record RUNNING
       ↓
perform operation
       ↓
determine outcome
       ↓
mark SUCCEEDED
       ↓
insert applied record
```

---

# 106. Crash window

Caso:

```text id="4vzb0h"
DDL succeeded
process crashes
before repository write
```

Después:

```text id="13671j"
schema changed
repository says not applied
```

Este es un estado real que deberá poder reconciliarse.

---

# 107. Crash recovery

El Repository System deberá conservar suficiente información para que:

```text id="8ndy1d"
Migration Executor / Recovery logic
```

pueda detectar:

```text id="g2t4j2"
possibly applied but unrecorded
```

cuando sea viable.

---

# 108. Repository cannot prove physical state alone

Regla:

```text id="bfo45n"
Repository says NOT_APPLIED
⇏
Database effects absent
```

después de un fallo no atómico.

---

# 109. Applied implication

Ideal:

```text id="wtui8c"
Repository APPLIED
⇒
Migration execution contract completed
```

Esta es una garantía que el sistema debe proteger.

---

# 110. Repository write idempotency

Registrar una misma migration aplicada dos veces deberá ser:

```text id="fe38fo"
rejected
```

o reconocido como duplicate-safe cuando checksum/execution ID coincidan bajo recovery protocol.

No insertar otra fila.

---

# 111. Compare-and-record

Se podría proporcionar:

```php id="9lcw0i"
recordAppliedIfAbsent(
    AppliedMigrationRecord $record
): MigrationRepositoryWriteResult;
```

con resultados:

```text id="sdw4xr"
INSERTED
ALREADY_PRESENT_MATCHING
CONFLICT
```

---

# 112. Matching duplicate

Si una recovery operation encuentra:

```text id="8rprfg"
same migration
same semantic checksum
same execution contract
```

puede tratarse de forma especial.

---

# 113. Conflicting duplicate

Si:

```text id="u8y0d0"
same migration identity
different checksum
```

debe ser:

```text id="6y0xsg"
CONFLICT
```

---

# 114. Repository Write Context

Se propone:

```php id="86almk"
final readonly class MigrationRepositoryWriteContext
{
    public function __construct(
        public MigrationExecutionId $execution,
        public ?TransactionContext $transaction,
        public MigrationRepositoryWritePolicy $policy,
    ) {}
}
```

El repository no crea necesariamente la transaction; puede recibir un contexto válido.

---

# 115. Transaction ownership

Debe mantenerse:

```text id="8umz6q"
Migration Executor / Transaction Manager
owns transaction
```

```text id="0jsd9k"
Repository
participates in transaction
```

cuando proceda.

---

# 116. Repository connection

El repository necesita acceso a una conexión durante operaciones live.

Pero:

```text id="j6jhyo"
MigrationRepositorySnapshot
```

no contendrá esa conexión.

---

# 117. Repository session

Se propone:

```text id="4w55nx"
MigrationRepositorySession
```

operation-scoped.

---

# 118. Session responsibilities

Puede contener:

```text id="zsbimw"
connection lease
prepared statements
transaction participation
repository version
temporary write state
```

---

# 119. Session ≠ repository service

Shared service:

```text id="ytkqsz"
DatabaseMigrationRepository
```

idealmente immutable/stateless.

Live resources:

```text id="l62zl1"
MigrationRepositorySession
```

operation-scoped.

---

# 120. Persistent runtime safety

Con FrankenPHP:

```text id="a9bnk2"
shared repository service
+
fresh session per migration command
```

---

# 121. RoadRunner

Misma regla:

```text id="a9b4kj"
worker survives
repository session does not
```

---

# 122. OpenSwoole

Cada coroutine deberá tener su propia:

```text id="0ta14g"
RepositorySession
```

o un context seguro.

---

# 123. No current repository connection singleton

Prohibido:

```php id="r2hcm4"
MigrationRepository::$connection;
```

como live mutable global.

---

# 124. Connection selection

Repository target debe corresponder al migration database target.

Pero puede configurarse:

```text id="avmg3s"
application database
framework metadata database
central migration database
```

en escenarios avanzados.

---

# 125. Repository locality

Default recomendado:

```text id="3ge9uu"
Repository lives with target database
```

porque reduce inconsistencias.

---

# 126. Central repository caveat

Un repository central para muchas bases introduce:

```text id="zbj38t"
distributed consistency
network dependency
cross-database atomicity problems
```

Por tanto deberá ser configuración avanzada.

---

# 127. Multi-database identity

Si se usa repository central, la key deberá incluir:

```text id="vxb704"
DatabaseTargetIdentity
+
MigrationIdentity
```

---

# 128. RepositoryTargetIdentity

Se propone:

```php id="9jded5"
final readonly class MigrationRepositoryTargetIdentity
{
    public function __construct(
        public string $value
    ) {}
}
```

---

# 129. Multitenancy integration

Core no dependerá de Multitenancy.

Pero cuando cada tenant tenga base/schema separado, el integration package podrá proporcionar:

```text id="emw346"
TenantRepositoryTargetIdentity
```

---

# 130. Tenant isolation

No deberá ocurrir:

```text id="0w7543"
tenant A:M1 applied
```

y que tenant B sea considerado aplicado por la misma fila.

---

# 131. Tenant-aware repository

Opciones:

```text id="amcwj2"
repository per tenant database
```

o:

```text id="1s9ct2"
central repository with explicit tenant target identity
```

según integración.

---

# 132. Repository schema naming

Se recomienda una estrategia:

```text id="bnl4un"
MigrationRepositoryNamingStrategy
```

para nombres físicos.

No hardcode disperso.

---

# 133. Default table names

Conceptualmente:

```text id="3vhsin"
voltstack_migrations
voltstack_migration_executions
voltstack_migration_repository
```

La decisión exacta podrá simplificarse.

---

# 134. Minimal schema proposal

Una primera versión robusta puede usar:

```text id="en9vvy"
voltstack_migrations
├── repository_target
├── namespace
├── migration_id
├── semantic_checksum
├── checksum_algorithm
├── checksum_version
├── batch_id
├── batch_sequence
├── batch_position
├── execution_id
├── applied_at
└── metadata
```

Unique:

```text id="k2sklw"
(repository_target, namespace, migration_id)
```

---

# 135. Execution table proposal

```text id="hqt1a2"
voltstack_migration_executions
├── execution_id
├── repository_target
├── namespace
├── migration_id
├── batch_id
├── state
├── outcome_certainty
├── started_at
├── finished_at
├── plan_fingerprint
├── failure_code
└── metadata
```

---

# 136. Repository metadata table

Opcional:

```text id="5gi8l5"
voltstack_migration_repository_meta
├── repository_target
├── schema_version
├── created_at
├── upgraded_at
└── metadata
```

---

# 137. Table count tradeoff

V1 puede usar:

```text id="nfuvnv"
2 tables
```

o:

```text id="hqbphc"
3 tables
```

dependiendo del nivel de audit requerido.

La arquitectura no deberá depender de una tabla única.

---

# 138. Metadata encoding

No guardar información semánticamente crítica únicamente dentro de:

```text id="nrvaq3"
JSON metadata
```

Campos esenciales deberán ser columnas tipadas.

---

# 139. Metadata JSON

Puede utilizarse para información adicional no indexada:

```text id="tybd80"
runtime adapter
release ID
diagnostic annotations
```

---

# 140. Repository queries

Internamente debe utilizar:

```text id="n1j7bb"
low-level Query Engine
```

o una capa especializada del Database subsystem.

No ORM.

---

# 141. Repository must not depend on ORM

Regla crítica:

```text id="hbu84k"
Migration Repository
→ Query/Execution
```

No:

```text id="lwz64f"
Migration Repository
→ ORM
```

porque el ORM puede depender del schema/migrations indirectamente y producir ciclos.

---

# 142. No Active Record

No modelar la tabla como:

```php id="1a8oqw"
class MigrationRecord extends Model
```

como dependencia interna principal.

---

# 143. Repository bootstrap query path

Debe utilizar:

```text id="sf061d"
Schema subsystem
```

para crear las tablas.

Después:

```text id="kgiywr"
Query Engine / Execution Engine
```

para lecturas/escrituras.

---

# 144. No circular dependency

Evitar:

```text id="zqrewo"
Migration Repository
  ↓
ORM
  ↓
Schema
  ↓
Migration
```

---

# 145. Read operations

Ejemplos:

```text id="5jkm2e"
snapshot()
hasApplied(identity)
findApplied(identity)
latestBatch()
executionHistory(identity)
repositoryVersion()
```

Pero se recomienda que el Planner consuma snapshot, no haga cientos de reads dispersos.

---

# 146. Snapshot-first planning

Preferir:

```text id="jpvuwr"
Repository
  ↓
Snapshot
  ↓
Planner
```

en vez de:

```text id="b33e9t"
Planner
 ├── repository.has(M1)
 ├── repository.has(M2)
 ├── repository.has(M3)
 └── ...
```

---

# 147. Why snapshot-first

Mejora:

```text id="kfpukl"
consistency
determinism
performance
testing
cacheability
```

---

# 148. Write operations

Podrán incluir:

```text id="8ykgml"
beginExecution
updateExecutionState
appendApplied
removeCurrentApplied
recordRollback
recordRecovery
```

dependiendo del history profile.

---

# 149. State transitions

Execution history state machine:

```text id="j7uoqp"
CREATED
   ↓
RUNNING
   ├────► FAILED
   ├────► PARTIAL
   ├────► CANCELLED
   ├────► UNKNOWN
   └────► SUCCEEDED
```

---

# 150. Invalid transitions

Ejemplos:

```text id="zln51c"
FAILED → RUNNING
SUCCEEDED → RUNNING
CANCELLED → SUCCEEDED
```

sin recovery operation explícita deberán rechazarse.

---

# 151. State transition validator

Se propone:

```text id="ad908u"
MigrationExecutionStateMachine
```

---

# 152. Recovery transition

Podría permitirse:

```text id="0k0wpz"
UNKNOWN
   ↓
RECOVERED_AS_SUCCEEDED
```

mediante un recovery workflow explícito.

---

# 153. History immutability

Idealmente el repository history será append-oriented.

Cambios en projection podrán ocurrir, pero historial no deberá reescribirse silenciosamente.

---

# 154. Audit profile

En modo audit:

```text id="qkcu59"
all relevant transitions
```

deberán preservarse.

---

# 155. Repository read isolation

Las lecturas del snapshot deberán usar un nivel suficiente para impedir mezcla incoherente de filas.

No se requiere universalmente SERIALIZABLE.

Debe ser capability-driven.

---

# 156. Lock interaction

El Migration Lock normalmente se adquiere antes de:

```text id="whg82t"
final repository snapshot + planning/execution
```

según workflow.

Pero Repository System no deberá confundirse con Lock Manager.

---

# 157. Repository lock ≠ Migration Lock

Puede existir un lock interno para:

```text id="xikrn5"
batch allocation
repository bootstrap
repository upgrade
```

pero:

```text id="jsvohr"
Repository Internal Lock
≠
Migration Coordination Lock
```

---

# 158. Bootstrap concurrency

Dos nodos iniciando al mismo tiempo podrían intentar crear el repository.

El bootstrapper deberá ser race-safe.

---

# 159. Bootstrap race

Ejemplo:

```text id="y4rj62"
Node A: repository missing
Node B: repository missing
Node A: CREATE
Node B: CREATE
```

La segunda operación deberá manejarse mediante:

```text id="h8r3wr"
idempotent bootstrap strategy
or coordination
```

sin ocultar errores ajenos.

---

# 160. IF NOT EXISTS caveat

```text id="xexl2g"
CREATE TABLE IF NOT EXISTS
```

puede ayudar, pero después deberá validarse:

```text id="2b5nyq"
repository schema matches expected internal schema
```

---

# 161. Bootstrap safety

Un objeto con mismo nombre pero schema incompatible no deberá aceptarse como repository válido solo porque existe.

---

# 162. Repository self-validation

Después del bootstrap:

```text id="teu65x"
Introspect internal repository schema
or validate via supported metadata strategy
```

según diseño.

---

# 163. Introspection dependency

El bootstrapper podrá usar Schema Introspection explícitamente.

Esto no introduce hidden I/O en pure data objects; el bootstrapper es un runtime service.

---

# 164. Repository schema fingerprint

Podrá existir:

```text id="fsr6xg"
MigrationRepositorySchemaFingerprint
```

para detectar modificaciones manuales.

---

# 165. Internal schema drift

Debe distinguirse:

```text id="pxpvi7"
Application Schema Drift
```

de:

```text id="61ga3v"
Migration Repository Schema Drift
```

---

# 166. Repository schema drift

Es una condición de infraestructura y puede bloquear migrations.

---

# 167. Cleanup policy

Execution history puede crecer.

Se podrá definir:

```text id="0x4bcb"
MigrationRepositoryRetentionPolicy
```

para ejecuciones antiguas.

Pero:

```text id="lz7eq6"
Applied Migration History
```

no deberá eliminarse arbitrariamente.

---

# 168. Retention

Puede aplicarse a:

```text id="npl2av"
failed attempt details
verbose telemetry
large metadata
```

según policy.

---

# 169. Applied records retention

Recomendación:

```text id="k75x0w"
retain indefinitely
```

mientras la base continúe perteneciendo a ese historial.

---

# 170. Repository export

Tooling podrá exportar:

```text id="wuqhoh"
Repository Snapshot
```

para auditoría/debugging.

Debe ser versionado.

---

# 171. Repository serialization

Formato:

```text id="s540hx"
typed
versioned
deterministic
```

No usar PHP `serialize()` como contrato.

---

# 172. Snapshot serialization

Puede incluir:

```text id="5t5jtg"
repository version
target identity
applied records
checksums
batches
relevant execution states
fingerprint
```

---

# 173. Security

El Repository System es infraestructura privilegiada.

Debe proteger:

```text id="scja5w"
repository writes
history integrity
target identity
checksums
execution metadata
```

---

# 174. SQL injection

Todas las queries internas deberán usar:

```text id="ogw2oq"
typed Query Engine / parameter binding
```

No interpolación.

---

# 175. Identifier configuration

El nombre físico de tablas internas deberá resolverse mediante:

```text id="69v3bp"
structured identifiers
```

y Schema Compiler.

---

# 176. Repository tampering

Si la aplicación tiene acceso directo y modifica filas manualmente:

```text id="l5vq8i"
tampering
```

puede resultar indistinguible de una herramienta administrativa legítima sin audit adicional.

VoltStack deberá al menos detectar inconsistencias posibles.

---

# 177. Cryptographic signing

No será requisito core inicial.

Pero una extensión futura podría añadir:

```text id="36hz9c"
signed repository records
```

para entornos de alta seguridad.

---

# 178. No secrets in metadata

Nunca almacenar:

```text id="bh6p3l"
DB password
API key
private key
full secret values
```

---

# 179. Telemetry

Eventos/metrics posibles:

```text id="cfs4uu"
migration.repository.snapshot
migration.repository.read
migration.repository.write
migration.repository.applied
migration.repository.rollback
migration.repository.drift
migration.repository.corruption
migration.repository.bootstrap
migration.repository.upgrade
```

---

# 180. Telemetry metrics

Ejemplos:

```text id="9b31d6"
repository.snapshot.duration
repository.applied.count
repository.execution_history.count
repository.write.failures
repository.drift.count
repository.bootstrap.duration
```

---

# 181. Telemetry privacy

No registrar automáticamente:

```text id="fmii1m"
full migration metadata JSON
sensitive target names
credentials
raw exception payloads containing secrets
```

---

# 182. Diagnostics

Un diagnostic debe indicar:

```text id="7x0txa"
repository target
migration identity
problem kind
expected state
observed state
safe next action category
```

---

# 183. Example drift diagnostic

```text id="cobq8z"
Migration definition drift detected.

Migration:
app:2026_09_06_010000_create_users

Repository checksum:
sha256:v1:abc...

Current semantic checksum:
sha256:v1:def...

Status:
SEMANTICALLY_MODIFIED
```

---

# 184. Example missing definition

```text id="rnwib7"
Applied migration definition is missing.

Migration:
package.acme.billing:2026_08_11_create_invoice_items

Repository state:
APPLIED

Catalog state:
NOT FOUND

Repository remains unchanged.
```

---

# 185. Example corruption

```text id="evrwb6"
Migration repository corruption detected.

Identity:
app:M42

Problem:
Multiple applied records exist for the same qualified identity.

Expected:
Exactly one current applied record.
```

---

# 186. Testing strategy

Debe incluir:

```text id="py8k30"
bootstrap tests
repository version tests
snapshot tests
checksum storage
applied record writes
duplicate protection
batch allocation
rollback projection
execution history
partial/unknown states
drift detection
missing definition reconciliation
repository corruption
transaction participation
non-transactional crash windows
concurrency
persistent runtime isolation
tenant target isolation
serialization
```

---

# 187. In-memory repository tests

Debe existir:

```text id="pu2k28"
InMemoryMigrationRepository
```

para unit testing.

---

# 188. Conformance suite

Todas las implementaciones deberán pasar:

```text id="im943p"
MigrationRepositoryConformanceTests
```

---

# 189. Conformance invariants

Verificar:

```text id="ypfnqp"
same identity cannot be applied twice
snapshot immutable
records deterministic
drift comparison stable
batch sequence behavior correct
```

---

# 190. Concurrency test

Dos writers:

```text id="ljy4yq"
Writer A → apply M1
Writer B → apply M1
```

solo uno deberá registrar la migration.

---

# 191. Snapshot consistency test

Mientras ocurre write concurrente:

```text id="508c2d"
Snapshot
```

debe ser:

```text id="xguxd2"
before
```

o:

```text id="nq34gn"
after
```

según isolation contract,

no una mezcla inválida.

---

# 192. Crash simulation

Simular:

```text id="q03aeu"
DDL success
process crash
repository record missing
```

y verificar que recovery/reconciliation no marca automáticamente applied.

---

# 193. Repository schema version test

Runtime soporta:

```text id="5qu9cp"
repository v1-v3
```

repository es:

```text id="33i6yu"
v4
```

debe producir:

```text id="a78e4x"
UnsupportedMigrationRepositoryVersionException
```

---

# 194. Persistent runtime test

Dos migrations concurrentes en contexts separados no deberán compartir:

```text id="i8gzf0"
session
transaction
target identity
batch state
execution record buffer
```

---

# 195. Error hierarchy

Se propone:

```text id="wpirjk"
DatabaseMigrationRepositoryException
├── MigrationRepositoryBootstrapException
├── MigrationRepositoryUnavailableException
├── MigrationRepositoryVersionException
│   ├── UnsupportedMigrationRepositoryVersionException
│   └── MigrationRepositoryUpgradeRequiredException
│
├── MigrationRepositorySchemaException
│   ├── InvalidMigrationRepositorySchemaException
│   └── MigrationRepositorySchemaDriftException
│
├── MigrationRepositoryReadException
├── MigrationRepositoryWriteException
├── DuplicateAppliedMigrationException
├── MigrationRepositoryChecksumConflictException
├── MigrationRepositoryChangedException
├── MigrationRepositoryCorruptedException
├── MigrationRepositoryReconciliationException
├── MigrationRepositoryBatchException
├── MigrationRepositoryExecutionStateException
├── MigrationRepositoryTargetException
├── MigrationRepositorySerializationException
├── MigrationRepositoryConcurrencyException
└── MigrationRepositoryInvariantException
```

---

# 196. Namespace propuesto

```text id="q6x708"
VoltStack\Quantum\Database\Migration\Repository
```

---

# 197. Estructura propuesta

```text id="yvwfjw"
Migration/
└── Repository/
    ├── Contract/
    │   ├── MigrationRepository.php
    │   ├── MigrationRepositoryReader.php
    │   ├── MigrationRepositoryWriter.php
    │   ├── MigrationRepositoryValidator.php
    │   └── MigrationRepositoryReconciler.php
    │
    ├── Core/
    │   ├── DatabaseMigrationRepository.php
    │   ├── InMemoryMigrationRepository.php
    │   ├── MigrationRepositoryContext.php
    │   ├── MigrationRepositoryReadContext.php
    │   ├── MigrationRepositoryWriteContext.php
    │   └── MigrationRepositorySession.php
    │
    ├── Record/
    │   ├── AppliedMigrationRecord.php
    │   ├── AppliedMigrationRecordSet.php
    │   ├── MigrationExecutionRecord.php
    │   ├── MigrationExecutionRecordSet.php
    │   ├── AppliedMigrationMetadata.php
    │   └── MigrationExecutionRecordMetadata.php
    │
    ├── Snapshot/
    │   ├── MigrationRepositorySnapshot.php
    │   ├── MigrationRepositorySnapshotMetadata.php
    │   └── MigrationRepositoryFingerprint.php
    │
    ├── State/
    │   ├── MigrationRepositoryState.php
    │   ├── MigrationExecutionStateMachine.php
    │   └── MigrationReconciliationState.php
    │
    ├── Checksum/
    │   ├── MigrationRepositoryChecksum.php
    │   ├── MigrationSemanticChecksum.php
    │   └── MigrationChecksumComparison.php
    │
    ├── Batch/
    │   ├── MigrationBatchId.php
    │   ├── MigrationBatchSequence.php
    │   ├── MigrationBatchPosition.php
    │   └── MigrationBatchAllocator.php
    │
    ├── Version/
    │   ├── MigrationRepositoryVersion.php
    │   ├── MigrationRepositoryVersionResolver.php
    │   └── MigrationRepositoryUpgrader.php
    │
    ├── Bootstrap/
    │   ├── MigrationRepositoryBootstrapper.php
    │   ├── MigrationRepositoryBootstrapPlan.php
    │   └── MigrationRepositoryBootstrapResult.php
    │
    ├── Schema/
    │   ├── MigrationRepositorySchemaDefinition.php
    │   ├── MigrationRepositorySchemaValidator.php
    │   └── MigrationRepositorySchemaFingerprint.php
    │
    ├── Reconciliation/
    │   ├── MigrationRepositoryReconciler.php
    │   ├── MigrationReconciliationReport.php
    │   ├── MigrationReconciliationFinding.php
    │   └── MigrationReconciliationState.php
    │
    ├── History/
    │   ├── MigrationHistoryEvent.php
    │   ├── MigrationHistoryEventKind.php
    │   └── MigrationCurrentStateProjection.php
    │
    ├── Target/
    │   └── MigrationRepositoryTargetIdentity.php
    │
    ├── Retention/
    │   └── MigrationRepositoryRetentionPolicy.php
    │
    ├── Serialization/
    │   ├── MigrationRepositorySnapshotSerializer.php
    │   └── MigrationRepositorySnapshotDeserializer.php
    │
    ├── Diagnostic/
    │   ├── MigrationRepositoryDiagnostic.php
    │   └── MigrationRepositoryDiagnosticBag.php
    │
    ├── Telemetry/
    │   └── MigrationRepositoryTelemetry.php
    │
    └── Exception/
        ├── DatabaseMigrationRepositoryException.php
        ├── MigrationRepositoryBootstrapException.php
        ├── MigrationRepositoryUnavailableException.php
        ├── UnsupportedMigrationRepositoryVersionException.php
        ├── MigrationRepositoryUpgradeRequiredException.php
        ├── InvalidMigrationRepositorySchemaException.php
        ├── MigrationRepositorySchemaDriftException.php
        ├── MigrationRepositoryReadException.php
        ├── MigrationRepositoryWriteException.php
        ├── DuplicateAppliedMigrationException.php
        ├── MigrationRepositoryChecksumConflictException.php
        ├── MigrationRepositoryChangedException.php
        ├── MigrationRepositoryCorruptedException.php
        ├── MigrationRepositoryReconciliationException.php
        ├── MigrationRepositoryBatchException.php
        ├── MigrationRepositoryExecutionStateException.php
        ├── MigrationRepositoryTargetException.php
        ├── MigrationRepositoryConcurrencyException.php
        └── MigrationRepositoryInvariantException.php
```

---

# 198. Architectural invariants

## DB-MIGRATION-REPO-001
Migration Repository será distinto de Migration Discovery.

## DB-MIGRATION-REPO-002
Migration Repository será distinto de Migration Catalog.

## DB-MIGRATION-REPO-003
Repository Record será distinto de Migration Definition.

## DB-MIGRATION-REPO-004
Applied Migration será distinto de Available Migration.

## DB-MIGRATION-REPO-005
Repository State será distinto de Execution Attempt State.

## DB-MIGRATION-REPO-006
Migration Batch será distinto de Transaction.

## DB-MIGRATION-REPO-007
Repository Checksum será distinto de Source Checksum.

## DB-MIGRATION-REPO-008
Repository Snapshot será distinto de live Repository Session.

## DB-MIGRATION-REPO-009
Repository no descubrirá migration files.

## DB-MIGRATION-REPO-010
Repository no cargará MigrationDefinitions como responsabilidad central.

## DB-MIGRATION-REPO-011
Repository no planificará migrations.

## DB-MIGRATION-REPO-012
Repository no ejecutará DDL.

## DB-MIGRATION-REPO-013
Repository no ejecutará data migrations.

## DB-MIGRATION-REPO-014
Repository no decidirá rollback strategy.

## DB-MIGRATION-REPO-015
Repository no decidirá safety.

## DB-MIGRATION-REPO-016
Repository no decidirá platform compatibility.

## DB-MIGRATION-REPO-017
Qualified MigrationIdentity será first-class.

## DB-MIGRATION-REPO-018
Repository identity incluirá namespace cuando aplique.

## DB-MIGRATION-REPO-019
Filename no será repository identity.

## DB-MIGRATION-REPO-020
AppliedMigrationRecord será immutable conceptualmente.

## DB-MIGRATION-REPO-021
Applied records no se reescribirán silenciosamente.

## DB-MIGRATION-REPO-022
Applied record será distinto de ExecutionRecord.

## DB-MIGRATION-REPO-023
Failed execution no creará AppliedMigrationRecord.

## DB-MIGRATION-REPO-024
Partial execution no se marcará APPLIED.

## DB-MIGRATION-REPO-025
Unknown execution no se marcará APPLIED.

## DB-MIGRATION-REPO-026
Repository reconciliation states podrán derivarse.

## DB-MIGRATION-REPO-027
Stored facts serán distintos de derived states.

## DB-MIGRATION-REPO-028
RepositorySnapshot será immutable.

## DB-MIGRATION-REPO-029
RepositorySnapshot será deterministic.

## DB-MIGRATION-REPO-030
RepositorySnapshot será target-specific.

## DB-MIGRATION-REPO-031
Snapshot deberá representar vista consistente según contract.

## DB-MIGRATION-REPO-032
RepositoryFingerprint será deterministic.

## DB-MIGRATION-REPO-033
RepositoryFingerprint no reemplazará locking.

## DB-MIGRATION-REPO-034
Semantic checksum aplicado será persistido.

## DB-MIGRATION-REPO-035
Checksum algorithm será persistido.

## DB-MIGRATION-REPO-036
Checksum version será persistida.

## DB-MIGRATION-REPO-037
Checksum upgrades no reescribirán historial silenciosamente.

## DB-MIGRATION-REPO-038
Semantic checksum drift será detectable.

## DB-MIGRATION-REPO-039
Source-only drift podrá distinguirse si hay source checksum.

## DB-MIGRATION-REPO-040
Missing Definition será distinto de rollback.

## DB-MIGRATION-REPO-041
Missing Definition no implicará causa específica.

## DB-MIGRATION-REPO-042
Repository bootstrap será separado del migration execution normal.

## DB-MIGRATION-REPO-043
Bootstrapper no será segundo general-purpose Schema System.

## DB-MIGRATION-REPO-044
RepositoryVersion será first-class.

## DB-MIGRATION-REPO-045
RepositoryVersion será distinta de framework version.

## DB-MIGRATION-REPO-046
RepositoryVersion será distinta de MigrationVersion.

## DB-MIGRATION-REPO-047
Unsupported newer repository version deberá fallar.

## DB-MIGRATION-REPO-048
Repository upgrade será explícito.

## DB-MIGRATION-REPO-049
Repository schema será validable.

## DB-MIGRATION-REPO-050
Duplicate applied identity será corrupción/conflicto.

## DB-MIGRATION-REPO-051
Current applied table impondrá uniqueness por identity.

## DB-MIGRATION-REPO-052
Retry history no duplicará applied records.

## DB-MIGRATION-REPO-053
Batch sequence allocation será concurrency-safe.

## DB-MIGRATION-REPO-054
Batch será distinto de execution ID.

## DB-MIGRATION-REPO-055
Batch order histórico será explícito si se necesita.

## DB-MIGRATION-REPO-056
Applied timestamp no será identity.

## DB-MIGRATION-REPO-057
Applied timestamp no será dependency proof.

## DB-MIGRATION-REPO-058
Execution metadata no contendrá secrets.

## DB-MIGRATION-REPO-059
Deployment release metadata será opcional.

## DB-MIGRATION-REPO-060
Repository no se convertirá en deployment repository.

## DB-MIGRATION-REPO-061
Rollback no deberá borrar todo historial en audit mode.

## DB-MIGRATION-REPO-062
Current applied projection será separable de immutable history.

## DB-MIGRATION-REPO-063
Rollback failure no eliminará applied record indiscriminadamente.

## DB-MIGRATION-REPO-064
Repository reconciliation será distinto de planning.

## DB-MIGRATION-REPO-065
Corruption será first-class.

## DB-MIGRATION-REPO-066
Corrupted repository no continuará migrations automáticamente.

## DB-MIGRATION-REPO-067
Repository repair será explícito.

## DB-MIGRATION-REPO-068
Repository repair será distinto de application migration.

## DB-MIGRATION-REPO-069
Repository writes participarán en transaction when possible.

## DB-MIGRATION-REPO-070
Repository no asumirá transactional DDL universal.

## DB-MIGRATION-REPO-071
Non-transactional crash windows serán modelados.

## DB-MIGRATION-REPO-072
Repository NOT_APPLIED no probará ausencia física después de uncertain failure.

## DB-MIGRATION-REPO-073
Repository APPLIED deberá implicar completed execution contract.

## DB-MIGRATION-REPO-074
Duplicate record attempts serán protegidos.

## DB-MIGRATION-REPO-075
Same identity + different checksum será conflict.

## DB-MIGRATION-REPO-076
Transaction ownership permanecerá fuera del Repository.

## DB-MIGRATION-REPO-077
Repository podrá participar en transaction context explícito.

## DB-MIGRATION-REPO-078
RepositorySession será operation-scoped.

## DB-MIGRATION-REPO-079
RepositorySnapshot no contendrá Connection.

## DB-MIGRATION-REPO-080
Shared repository service no conservará live transaction state.

## DB-MIGRATION-REPO-081
FrankenPHP no filtrará repository session state.

## DB-MIGRATION-REPO-082
RoadRunner no filtrará repository session state.

## DB-MIGRATION-REPO-083
OpenSwoole no compartirá mutable repository sessions entre coroutines.

## DB-MIGRATION-REPO-084
No habrá global mutable current repository connection.

## DB-MIGRATION-REPO-085
Repository locality será configurable.

## DB-MIGRATION-REPO-086
Central repository será advanced configuration.

## DB-MIGRATION-REPO-087
Central repository identity incluirá target.

## DB-MIGRATION-REPO-088
Multitenancy será integración externa.

## DB-MIGRATION-REPO-089
Tenant applied state estará aislado.

## DB-MIGRATION-REPO-090
Repository naming será strategy-driven.

## DB-MIGRATION-REPO-091
Critical fields no vivirán únicamente en JSON metadata.

## DB-MIGRATION-REPO-092
Repository queries no dependerán del ORM.

## DB-MIGRATION-REPO-093
Repository no dependerá de Active Record.

## DB-MIGRATION-REPO-094
Repository bootstrap utilizará Schema subsystem explícitamente.

## DB-MIGRATION-REPO-095
Repository runtime reads/writes utilizarán Query/Execution infrastructure.

## DB-MIGRATION-REPO-096
No habrá circular dependency Repository→ORM→Schema→Migration.

## DB-MIGRATION-REPO-097
Planner deberá preferir RepositorySnapshot sobre reads dispersos.

## DB-MIGRATION-REPO-098
Execution history state transitions serán validadas.

## DB-MIGRATION-REPO-099
Invalid execution state transitions serán rechazadas.

## DB-MIGRATION-REPO-100
Recovery transitions serán explícitas.

## DB-MIGRATION-REPO-101
History podrá ser append-oriented.

## DB-MIGRATION-REPO-102
Audit profile conservará transitions relevantes.

## DB-MIGRATION-REPO-103
Repository read isolation será capability-driven.

## DB-MIGRATION-REPO-104
Repository Internal Lock será distinto de Migration Coordination Lock.

## DB-MIGRATION-REPO-105
Bootstrap será concurrency-safe.

## DB-MIGRATION-REPO-106
IF NOT EXISTS no sustituirá schema validation.

## DB-MIGRATION-REPO-107
Repository self-validation ocurrirá tras bootstrap cuando sea requerido.

## DB-MIGRATION-REPO-108
Repository Schema Drift será distinto de Application Schema Drift.

## DB-MIGRATION-REPO-109
Repository schema drift podrá bloquear migrations.

## DB-MIGRATION-REPO-110
Execution history retention será configurable.

## DB-MIGRATION-REPO-111
Applied migration history no se eliminará arbitrariamente.

## DB-MIGRATION-REPO-112
Repository snapshot serialization será versionada.

## DB-MIGRATION-REPO-113
Repository snapshot serialization será deterministic.

## DB-MIGRATION-REPO-114
PHP serialize no será public repository contract.

## DB-MIGRATION-REPO-115
Repository writes usarán safe parameter binding.

## DB-MIGRATION-REPO-116
Repository table identifiers serán structured identifiers.

## DB-MIGRATION-REPO-117
Repository tampering deberá ser detectable cuando produzca inconsistencia.

## DB-MIGRATION-REPO-118
Cryptographic signing no será requisito core inicial.

## DB-MIGRATION-REPO-119
Telemetry será structured.

## DB-MIGRATION-REPO-120
Telemetry no expondrá secrets.

## DB-MIGRATION-REPO-121
Repository diagnostics serán source/identity-aware.

## DB-MIGRATION-REPO-122
Repository deberá poder probarse in-memory.

## DB-MIGRATION-REPO-123
Implementaciones Repository tendrán conformance suite.

## DB-MIGRATION-REPO-124
Concurrency tests serán obligatorios.

## DB-MIGRATION-REPO-125
Crash window tests serán obligatorios.

## DB-MIGRATION-REPO-126
Repository version tests serán obligatorios.

## DB-MIGRATION-REPO-127
Persistent runtime isolation tests serán obligatorios.

## DB-MIGRATION-REPO-128
Repository snapshot canonical ordering será deterministic.

## DB-MIGRATION-REPO-129
Canonical snapshot order será distinto de execution dependency order.

## DB-MIGRATION-REPO-130
Repository target identity será explicit where needed.

## DB-MIGRATION-REPO-131
Target identity no se inferirá desde mutable global tenant context.

## DB-MIGRATION-REPO-132
Repository fingerprints no incluirán live object IDs.

## DB-MIGRATION-REPO-133
Repository errors serán typed.

## DB-MIGRATION-REPO-134
Read errors serán distinguibles de write errors.

## DB-MIGRATION-REPO-135
Checksum conflicts serán distinguibles de duplicate identities.

## DB-MIGRATION-REPO-136
Repository corruption será distinguible de drift.

## DB-MIGRATION-REPO-137
Repository changed during planning será detectable.

## DB-MIGRATION-REPO-138
Stale plan handling no será responsabilidad del raw repository storage layer.

## DB-MIGRATION-REPO-139
Repository facts no serán reinterpretados como physical schema truth.

## DB-MIGRATION-REPO-140
Repository no hará hidden schema introspection durante ordinary snapshot reads.

## DB-MIGRATION-REPO-141
Bootstrap/validation introspection será explícita.

## DB-MIGRATION-REPO-142
Repository current state projection será deterministic.

## DB-MIGRATION-REPO-143
Execution history retention no podrá borrar current applied truth.

## DB-MIGRATION-REPO-144
Repository maintenance será auditable cuando sea posible.

## DB-MIGRATION-REPO-145
Applied checksum siempre corresponderá al momento de aplicación aceptada.

## DB-MIGRATION-REPO-146
Repository no actualizará applied checksum para “hacer coincidir” una definición modificada.

## DB-MIGRATION-REPO-147
Manual acceptance de drift requerirá operación explícita de governance/repair.

## DB-MIGRATION-REPO-148
Repository no resolverá migration dependencies.

## DB-MIGRATION-REPO-149
Repository no seleccionará pending migrations.

## DB-MIGRATION-REPO-150
VoltStack tratará el Migration Repository como historial persistente protegido, no como una simple lista mutable de filenames ejecutados.

---

# 199. Anti-patterns

## 199.1 Repository como array de filenames

Incorrecto:

```text id="jl7pdt"
[
    'create_users.php',
    'create_posts.php',
]
```

Falta:

```text id="6lbmtf"
namespace
semantic checksum
batch
execution identity
repository version
```

---

## 199.2 Sin qualified identity

Incorrecto:

```text id="xrcggf"
PRIMARY KEY(migration_id)
```

si packages pueden reutilizar IDs.

Preferir:

```text id="8iw07o"
(repository_target, namespace, migration_id)
```

según topology.

---

## 199.3 Checksum sobrescrito

Incorrecto:

```php id="6k9hw4"
if ($currentChecksum !== $storedChecksum) {
    updateStoredChecksum($currentChecksum);
}
```

Esto elimina la capacidad de detectar drift.

---

## 199.4 Migration applied dos veces

Incorrecto:

```text id="o740fx"
M1
M1
```

como dos current applied rows.

---

## 199.5 Batch = transaction

Incorrecto:

```text id="57jg3b"
batch 4
=
transaction 4
```

---

## 199.6 Rollback = delete record immediately

Incorrecto antes de demostrar que rollback terminó correctamente.

---

## 199.7 Repository usando ORM

Incorrecto como dependencia central:

```php id="4knxse"
MigrationRecord::query()->get();
```

si el ORM forma parte de capas superiores.

---

## 199.8 Repository global connection

Incorrecto:

```php id="x3081x"
final class MigrationRepository
{
    public static Connection $connection;
}
```

---

## 199.9 Missing definition = rolled back

Incorrecto:

```text id="fg82w0"
not in source tree
⇒
not applied
```

---

## 199.10 Repository APPLIED tras fallo parcial

Incorrecto:

```text id="scwnr0"
some steps succeeded
migration failed
repository inserts applied anyway
```

---

# 200. Ejemplo de snapshot

```text id="3tdsf0"
MigrationRepositorySnapshot
│
├── repositoryVersion: 3.0
├── target: default
├── applied:
│   ├── app:M1
│   │   ├── checksum: sha256:v2:...
│   │   ├── batch: B1
│   │   └── appliedAt: ...
│   │
│   ├── app:M2
│   │   ├── checksum: sha256:v2:...
│   │   ├── batch: B1
│   │   └── appliedAt: ...
│   │
│   └── package.billing:M3
│       ├── checksum: sha256:v2:...
│       ├── batch: B2
│       └── appliedAt: ...
│
└── fingerprint: ...
```

---

# 201. Reconciliation example

Catalog:

```text id="b7vg1l"
app:M1 checksum A
app:M2 checksum B2
app:M4 checksum D
```

Repository:

```text id="n8uj23"
app:M1 checksum A
app:M2 checksum B1
app:M3 checksum C
```

Resultado:

```text id="huivqf"
M1 → APPLIED_AND_MATCHING
M2 → APPLIED_BUT_DRIFTED
M3 → APPLIED_BUT_MISSING_DEFINITION
M4 → AVAILABLE_BUT_NOT_APPLIED
```

---

# 202. Planner input

El Planner deberá recibir:

```text id="6cj7dk"
MigrationCatalog
+
MigrationRepositorySnapshot
+
ReconciliationReport
+
MigrationDefinitions
+
Policies
+
Capabilities
```

No deberá leer directamente filas arbitrarias durante todo el planning process.

---

# 203. Repository correctness formula

```text id="xogbwm"
CorrectRepository
=
UniqueQualifiedIdentity
∧
ImmutableAppliedRecords
∧
VersionedChecksums
∧
ConsistentSnapshots
∧
ProtectedWrites
∧
ExplicitExecutionStates
∧
TargetIsolation
∧
DriftPreservation
```

---

# 204. Applied-state guarantee

La garantía principal será:

```text id="605906"
Applied(M)
⇒
MigrationExecutionContractSatisfied(M)
```

VoltStack deberá diseñar todo el write protocol para maximizar esta propiedad.

---

# 205. Limitation formula

En ausencia de atomicidad:

```text id="77vzh6"
¬Applied(M)
⇏
¬Effects(M)
```

debido a:

```text id="xznx93"
crash
network failure
non-transactional DDL
unknown outcome
```

Por eso repository y physical database state son fuentes diferentes.

---

# 206. Repository vs physical schema

```text id="noe49d"
Migration Repository
=
historical claim
```

```text id="wcszyh"
Schema Introspection
=
observed structural state
```

```text id="8wxiti"
Schema Diff
=
structural comparison
```

Ninguno sustituye a los demás.

---

# 207. Arquitectura final

```text id="8omlvi"
                     Database Target
                           │
                           ▼
               Migration Repository
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
        Metadata       Applied State   Execution History
            │              │              │
            └──────────────┼──────────────┘
                           ▼
                 Repository Validator
                           │
                           ▼
                  Canonical Snapshot
                           │
                           ▼
               Repository Fingerprint
                           │
                           ▼
                  Reconciliation
                     /            \
                    /              \
                   ▼                ▼
          Migration Catalog    Definitions
                    \              /
                     \            /
                      ▼          ▼
                    Migration Planner
                           │
                           ▼
                    Migration Executor
                           │
                           ▼
                  Repository Writer
```

---

# 208. Regla maestra

> **El Migration Repository recuerda qué evolución ha sido aceptada como aplicada; no decide qué evolución existe, cuál debe ejecutarse ni cuál es el estado físico real de la base.**

En forma compacta:

```text id="wzr1xg"
Discovery finds.
Definition describes.
Repository remembers.
Reconciliation compares history.
Planner decides.
Executor changes the database.
```

---

# 209. Resultado arquitectónico

Con este sistema VoltStack obtiene un repository capaz de soportar desde una aplicación simple hasta escenarios avanzados con:

```text id="rnbo7f"
application migrations
package migrations
namespaces
semantic checksums
batches
execution history
audit
recovery
multiple database targets
persistent workers
multitenancy integration
schema drift reconciliation
```

sin reducir el historial de evolución a:

```text id="8tf3i7"
filename + integer batch
```

La versión simple de VoltStack podrá mantener una DX similar a Laravel, pero internamente dispondrá de semántica suficiente para:

```text id="dpuq3z"
deterministic planning
drift detection
zero-downtime migrations
safe retries
failure recovery
package isolation
auditable execution
```

---

# 210. Siguiente documento

```text id="zlhx22"
105_DATABASE_MIGRATION_PLANNER_SYSTEM.md
```

El siguiente documento definirá cómo VoltStack combina:

```text id="12min0"
Migration Catalog
+
Migration Definitions
+
Migration Repository Snapshot
+
Dependency Graph
+
Compatibility Reports
+
Safety Requirements
+
Platform Capabilities
+
Migration Target
+
Policies
```

para producir:

```text id="smwhr0"
MigrationPlan
```

incluyendo:

```text id="bijgy8"
pending migration selection
dependency closure
topological ordering
migration target resolution
operation graph construction
schema/data operation planning
barriers
transaction boundaries
locking requirements
preconditions
postconditions
backfill planning
compatibility strategies
retry constraints
zero-downtime phases
plan fingerprinting
deterministic planning
stale repository detection
```

y fijará la separación:

```text id="dwkpou"
Migration Planner
≠
Migration Executor

Migration Planner
≠
Schema Planner

Migration Planner
≠
Repository

Migration Planner
≠
Safety Analyzer

Planning
≠
Execution

Declared Migration Order
≠
Physical Execution Strategy
```