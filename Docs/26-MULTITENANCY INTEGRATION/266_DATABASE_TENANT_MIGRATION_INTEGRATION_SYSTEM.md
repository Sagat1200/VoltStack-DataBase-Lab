# 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Tenant Migration Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Integración:** `VoltStack/Quantum/Multitenancy`  
**Documento:** 266 — Tenant Migration Integration System  
**Bloque:** 26 — Multitenancy Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md`  
**Siguiente documento:** `267_DATABASE_TEMPORAL_DATA_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura mediante la cual el sistema canónico de migraciones de VoltStack podrá operar sobre arquitecturas multitenant sin crear un segundo Migration Engine.

El sistema deberá soportar tenants aislados mediante:

```text
shared database + shared schema
shared database + separate schemas
separate databases
shards
hybrid placement
```

y permitir que una evolución estructural pueda aplicarse de forma:

```text
deterministic
bounded
observable
resumable
recoverable
version-aware
tenant-aware
placement-aware
```

La regla central será:

> **Una migración multitenant de VoltStack no será una única migración repetida ciegamente N veces. Será una operación de flota compuesta por unidades tenant-scoped independientes, versionadas, observables y recuperables, cuya ejecución deberá respetar isolation strategy, compatibility windows, placement generations y resultados parciales sin fingir atomicidad global.**

Por tanto:

```text
Tenant Migration
≠
foreach ($tenants as $tenant) {
    migrate();
}
```

---

# 2. Relación con el Migration Engine existente

La integración utilizará:

```text
101_DATABASE_MIGRATION_ARCHITECTURE
102_DATABASE_MIGRATION_SYSTEM
103_DATABASE_MIGRATION_DISCOVERY_SYSTEM
104_DATABASE_MIGRATION_REPOSITORY_SYSTEM
105_DATABASE_MIGRATION_PLANNER_SYSTEM
106_DATABASE_MIGRATION_EXECUTION_SYSTEM
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM
108_DATABASE_MIGRATION_BATCH_SYSTEM
109_DATABASE_SCHEMA_DIFF_MIGRATION_SYSTEM
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM
111_DATABASE_MIGRATION_SAFETY_SYSTEM
```

No se implementará:

```text
TenantMigrationEngine
```

como motor independiente.

La integración será:

```text
Canonical Migration Engine
          +
Multitenancy Context
          +
Fleet Orchestration
          =
Tenant Migration System
```

---

# 3. Dependencia arquitectónica

```text
Quantum/Database/Migration
           ↑
           │ integration
           │
Quantum/Multitenancy/Database/Migration
```

El núcleo Database no deberá depender obligatoriamente del paquete Multitenancy.

---

# 4. Separación fundamental

Deben distinguirse:

```text
Migration Definition
Migration Plan
Migration Execution
Tenant Migration Unit
Tenant Migration Fleet
Tenant Migration Rollout
```

No son equivalentes.

---

# 5. Migration Definition

Describe:

```text
qué cambio estructural debe realizarse
```

Ejemplo:

```php
Schema::table('orders', function (TableBlueprint $table) {
    $table->string('external_reference')->nullable();
});
```

La definición no debe conocer miles de tenants.

---

# 6. Tenant Migration Unit

Representa la aplicación de una migración a un domain tenant concreto.

Conceptualmente:

```text
Migration M
+
Tenant Persistence Domain D
=
TenantMigrationUnit U
```

---

# 7. Fleet Migration

Una misma migración puede producir:

```text
Migration M
│
├── Unit Tenant A
├── Unit Tenant B
├── Unit Tenant C
├── Unit Tenant D
└── ...
```

---

# 8. Fleet ≠ distributed transaction

Regla crítica:

```text
Fleet Migration
≠
Global ACID Transaction
```

Si 10,000 tenants son migrados, VoltStack no fingirá que todos forman una sola transaction.

---

# 9. Migration scopes

Se definen inicialmente:

```php
enum MigrationScope
{
    case GLOBAL;
    case TENANT;
    case SYSTEM;
}
```

---

# 10. GLOBAL migration

Modifica estructuras compartidas globalmente.

Ejemplo:

```text
global currencies
system tables
shared control-plane metadata
```

---

# 11. TENANT migration

Modifica estructuras pertenecientes a tenants.

---

# 12. SYSTEM migration

Reservada para infraestructura interna de VoltStack.

---

# 13. GLOBAL ≠ ALL TENANTS

Muy importante:

```text
GLOBAL
≠
execute once per tenant
```

---

# 14. Tenant migration declaration

Podrá expresarse mediante metadata:

```php
#[MigrationScope(MigrationScope::TENANT)]
final class AddExternalReferenceToOrders
{
}
```

---

# 15. Default scope

El default deberá ser explícito por módulo/contexto y no inferirse peligrosamente sólo del nombre de tabla.

---

# 16. Migration metadata

Conceptualmente:

```php
final readonly class TenantMigrationMetadata
{
    public function __construct(
        public MigrationId $migration,
        public MigrationScope $scope,
        public TenantMigrationCompatibility $compatibility,
        public TenantMigrationSafetyProfile $safety,
        public TenantMigrationExecutionHints $hints,
    ) {}
}
```

---

# 17. TenantMigrationContext

Cada unidad tendrá un contexto inmutable.

```php
final readonly class TenantMigrationContext
{
    public function __construct(
        public TenantId $tenant,
        public TenantIsolationDomain $isolationDomain,
        public TenantPersistenceDomain $persistenceDomain,
        public TenantPlacementGeneration $placementGeneration,
        public TenantMigrationTarget $target,
        public TenantMigrationRunId $run,
    ) {}
}
```

---

# 18. Context ≠ currentTenant global

Nunca:

```php
Tenant::current()
```

dentro del Migration Engine para descubrir qué tenant está siendo migrado.

---

# 19. Context propagation

```text
Fleet Planner
     ↓
Tenant Migration Unit
     ↓
Migration Planner
     ↓
Migration Executor
     ↓
Connection Resolver
     ↓
Database
```

El contexto deberá permanecer estable.

---

# 20. Isolation strategy

El comportamiento físico dependerá de:

```text
TenantIsolationStrategy
```

---

# 21. Shared database + shared schema

Supongamos:

```text
orders
├── tenant_id
├── id
└── status
```

Todos los tenants comparten físicamente la misma tabla.

Una migration estructural:

```text
ALTER TABLE orders ADD COLUMN ...
```

debe ejecutarse una sola vez físicamente.

---

# 22. Critical distinction

Aunque existan:

```text
10,000 tenants
```

si comparten la misma tabla:

```text
Physical Migration Targets = 1
```

no:

```text
10,000
```

---

# 23. Logical tenants ≠ physical migration targets

Regla:

```text
Tenant Count
≠
Migration Target Count
```

---

# 24. Migration target abstraction

```php
interface TenantMigrationTarget
{
    public function identity(): MigrationTargetId;
}
```

---

# 25. Target kinds

```text
SHARED_SCHEMA
TENANT_SCHEMA
TENANT_DATABASE
SHARD
CLUSTER
CUSTOM
```

---

# 26. Shared schema target

Múltiples tenants pueden mapear al mismo:

```text
MigrationTargetId
```

---

# 27. Target deduplication

El Fleet Planner deberá deduplicar targets físicos.

---

# 28. Example

```text
Tenant A ─┐
Tenant B ─┼── Shared Database / public schema
Tenant C ─┘
```

produce:

```text
1 migration unit
```

para un cambio puramente estructural compartido.

---

# 29. Data migrations

Una data migration puede ser diferente.

Ejemplo:

```text
backfill tenant-specific records
```

puede requerir:

```text
one logical work unit per tenant
```

aunque la estructura física sea compartida.

---

# 30. Structural target ≠ data target

Debe distinguirse:

```text
SchemaMigrationTarget
DataMigrationTarget
```

---

# 31. Schema-per-tenant

Ejemplo:

```text
database
├── tenant_a.orders
├── tenant_b.orders
└── tenant_c.orders
```

Aquí:

```text
3 tenant schemas
=
3 physical structural targets
```

---

# 32. Database-per-tenant

```text
Tenant A → Database A
Tenant B → Database B
Tenant C → Database C
```

produce igualmente targets independientes.

---

# 33. Sharded tenants

El target puede depender de:

```text
tenant
+
shard
```

---

# 34. Hybrid model

VoltStack deberá soportar simultáneamente:

```text
Tenant A → shared schema
Tenant B → dedicated schema
Tenant C → dedicated database
Tenant D → shard 17
```

sin cambiar la definición de migration.

---

# 35. Fleet discovery

Pipeline:

```text
Migration Definition
       ↓
Migration Scope
       ↓
Tenant Selection
       ↓
Tenant Placement Resolution
       ↓
Physical Target Resolution
       ↓
Target Deduplication
       ↓
Fleet Plan
```

---

# 36. Tenant selection

No siempre deben migrarse todos los tenants.

Podrán seleccionarse:

```text
all
specific tenant
tenant group
region
placement
schema version
application version
rollout wave
canary cohort
custom selector
```

---

# 37. Tenant selector

```php
interface TenantMigrationSelector
{
    public function select(
        TenantMigrationSelectionContext $context
    ): iterable;
}
```

---

# 38. Selector output

No debe retornar simplemente strings.

Preferir:

```text
TenantMigrationCandidate
```

con identidad y placement evidence.

---

# 39. Selection snapshot

Una fleet migration puede capturar:

```text
SelectionGeneration
```

para detectar cambios significativos durante el rollout.

---

# 40. Dynamic tenants

Mientras se ejecuta una migration pueden crearse nuevos tenants.

La política deberá definir si:

```text
new tenants are excluded
new tenants are appended
new tenants start at target version
```

---

# 41. Recommended provisioning rule

Un tenant creado durante un rollout debería ser provisionado directamente en una versión compatible actual cuando sea posible.

---

# 42. Migration Fleet Plan

```php
final readonly class TenantMigrationFleetPlan
{
    public function __construct(
        public TenantMigrationRunId $run,
        public MigrationSet $migrations,
        public TenantMigrationSelection $selection,
        public array $waves,
        public TenantMigrationExecutionPolicy $policy,
    ) {}
}
```

---

# 43. Immutable plan

Una vez aprobado:

```text
FleetPlan
```

será inmutable.

Cambios requerirán:

```text
new plan
```

o una transición controlada.

---

# 44. Fleet planning ≠ execution

El planner:

```text
discovers
groups
orders
validates
estimates
classifies
```

pero no ejecuta DDL.

---

# 45. Migration DAG

Las migrations conservarán dependencias.

```text
M1
 ↓
M2
 ↓
M3
```

---

# 46. Fleet DAG

Ahora existe una segunda dimensión:

```text
Migration dependency
×
Tenant target dependency
```

---

# 47. Ordering

No siempre será:

```text
M1 all tenants
M2 all tenants
M3 all tenants
```

Puede ser:

```text
Tenant A: M1 → M2 → M3
Tenant B: M1 → M2 → M3
```

si compatibility policy lo permite.

---

# 48. Ordering strategy

Podrá configurarse:

```text
MIGRATION_MAJOR
TENANT_MAJOR
WAVE_MAJOR
CUSTOM
```

---

# 49. MIGRATION_MAJOR

```text
M1 → all selected targets
M2 → all selected targets
M3 → all selected targets
```

---

# 50. TENANT_MAJOR

```text
Tenant A → M1 M2 M3
Tenant B → M1 M2 M3
```

---

# 51. WAVE_MAJOR

```text
Wave 1 → full compatible migration set
Wave 2 → full compatible migration set
...
```

---

# 52. Recommended default

Para grandes flotas:

```text
WAVE_MAJOR
```

con compatibility windows explícitas.

---

# 53. Canary tenants

Primer wave:

```text
small representative cohort
```

---

# 54. Canary purpose

Detectar:

```text
migration failures
unexpected lock duration
performance regression
application incompatibility
schema drift
replica problems
```

antes del rollout amplio.

---

# 55. Canary ≠ proof

Un canary exitoso no garantiza éxito de toda la flota.

---

# 56. Rollout waves

Ejemplo:

```text
Wave 0  Canary      5 tenants
Wave 1  Early       100 tenants
Wave 2  Regional    1,000 tenants
Wave 3  General     remaining tenants
```

---

# 57. Wave definition

```php
final readonly class TenantMigrationWave
{
    public function __construct(
        public TenantMigrationWaveId $id,
        public TenantMigrationTargetSet $targets,
        public ConcurrencyLimit $concurrency,
        public WaveGatePolicy $gate,
    ) {}
}
```

---

# 58. Wave gates

Antes de continuar:

```text
success ratio
failure threshold
latency threshold
lock threshold
drift threshold
health checks
manual approval
```

podrán ser evaluados.

---

# 59. No automatic success inference

```text
No reported error
≠
Migration verified
```

---

# 60. Unit lifecycle

Cada TenantMigrationUnit podrá estar:

```text
PLANNED
QUEUED
ACQUIRING_LEASE
VALIDATING
RUNNING
VERIFYING
SUCCEEDED
FAILED
BLOCKED
PAUSED
CANCELLED
UNKNOWN
```

---

# 61. Fleet lifecycle

```text
PLANNING
READY
RUNNING
PAUSED
PARTIALLY_SUCCEEDED
SUCCEEDED
FAILED
CANCELLED
UNKNOWN
```

---

# 62. PARTIALLY_SUCCEEDED

Es un estado de primera clase.

---

# 63. Example

```text
10,000 targets

9,996 SUCCEEDED
3 FAILED
1 UNKNOWN
```

El resultado es:

```text
PARTIALLY_SUCCEEDED
```

no:

```text
SUCCEEDED
```

---

# 64. UNKNOWN dominates safety decisions

Un target con outcome desconocido no deberá tratarse como failed-before-execution ni successful.

---

# 65. Migration repository

El sistema canónico:

```text
DATABASE_MIGRATION_REPOSITORY_SYSTEM
```

seguirá siendo autoridad sobre historial.

---

# 66. Tenant-aware repository key

Podrá usar:

```text
MigrationId
+
MigrationTargetId
```

---

# 67. Per-tenant repository

No siempre debe existir una tabla de migration por tenant.

---

# 68. Repository strategies

Podrán existir:

```text
CENTRALIZED
PER_DATABASE
PER_SCHEMA
HYBRID
```

---

# 69. Centralized repository

Ejemplo:

```text
tenant_migration_history

target_id
migration_id
batch
status
started_at
finished_at
checksum
```

---

# 70. Per-target repository

Puede vivir dentro del tenant database/schema.

---

# 71. Trade-off

Centralized:

```text
easier fleet visibility
```

Per-target:

```text
stronger target-local provenance
```

---

# 72. Hybrid repository

Puede mantener:

```text
local authoritative record
+
central fleet index
```

---

# 73. Split authority risk

Si existen ambos, deberá definirse cuál es autoridad para cada dato.

---

# 74. Fleet index ≠ migration truth

Un control-plane index puede estar stale.

---

# 75. Migration version

Cada target podrá tener:

```text
SchemaVersion
MigrationSequence
MigrationChecksumSet
```

---

# 76. Version skew

Ejemplo:

```text
A → v120
B → v120
C → v119
D → v117
```

es representable.

---

# 77. Version skew ≠ failure

Puede ser parte normal del rollout.

---

# 78. Application compatibility

La aplicación deberá declarar:

```text
MinSchemaVersion
MaxSchemaVersion
```

o una política más rica.

---

# 79. Compatibility window

Ejemplo:

```text
Application 7.2
supports schema 118..120
```

---

# 80. Too old

```text
schema 117
```

puede requerir:

```text
MIGRATION_REQUIRED
```

---

# 81. Too new

Un runtime antiguo frente a schema nuevo también puede ser incompatible.

---

# 82. Bidirectional compatibility

Debe comprobarse:

```text
Application → Schema
Schema → Application expectations
```

---

# 83. Expand/contract

Modelo recomendado:

```text
EXPAND
   ↓
COMPATIBILITY WINDOW
   ↓
DATA BACKFILL
   ↓
APPLICATION CUTOVER
   ↓
CONTRACT
```

---

# 84. Expand phase

Añadir estructuras compatibles.

Ejemplo:

```text
add nullable column
add new table
add compatible index
```

---

# 85. Backfill

Mover/popular datos gradualmente.

---

# 86. Dual read/write

Puede existir temporalmente:

```text
write old + new
read old/new
```

pero pertenece a una estrategia explícita de aplicación.

---

# 87. Contract

Eliminar estructuras antiguas sólo después de probar que ningún runtime compatible las necesita.

---

# 88. Contract safety

Una migration destructiva deberá verificar:

```text
old application versions drained
jobs compatible
workers restarted/upgraded
replicas compatible
```

cuando sea relevante.

---

# 89. Persistent runtimes

Con:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

pueden existir workers con código anterior aún activos.

---

# 90. Schema deployment ≠ application deployment

Por tanto:

```text
Migration completed
≠
All workers run new code
```

---

# 91. Worker generation

VoltStack podrá registrar:

```text
ApplicationGeneration
```

para gates de contract migrations.

---

# 92. Zero-downtime migration

Se integrará con:

```text
110_DATABASE_ZERO_DOWNTIME_MIGRATION_SYSTEM
```

---

# 93. Tenant ZDT

Añade:

```text
fleet rollout
tenant placement
tenant traffic
version skew
per-target cutover
```

---

# 94. Online capability

El sistema deberá consultar capabilities como:

```text
online index creation
concurrent index creation
non-blocking DDL
transactional DDL
lock behavior
```

---

# 95. Version ≠ capability

Nunca inferir seguridad únicamente por:

```text
database version number
```

---

# 96. Migration safety

Cada unit pasará por:

```text
MigrationSafetyAnalyzer
```

del sistema canónico.

---

# 97. Tenant-specific safety

Podrá añadir:

```text
tenant size
traffic
placement
replica state
maintenance window
schema drift
```

---

# 98. Same migration ≠ same operational risk

Ejemplo:

```text
Tenant A → 10 MB table
Tenant B → 4 TB table
```

La misma migration puede tener riesgos muy distintos.

---

# 99. Risk classes

Conceptualmente:

```text
LOW
MODERATE
HIGH
CRITICAL
UNKNOWN
```

---

# 100. UNKNOWN ≠ LOW

---

# 101. Table statistics

Podrán utilizarse como evidencia.

Pero:

```text
EstimatedRows
≠
ExactRows
```

---

# 102. Planning evidence

El plan deberá registrar qué evidencia utilizó.

---

# 103. Preflight validation

Antes de una unit:

```text
target exists
placement generation current
schema version expected
migration dependencies satisfied
connection available
capabilities valid
no conflicting migration lease
safety policy satisfied
```

---

# 104. Generation validation

Supongamos:

```text
Plan:
Tenant A → Database X
Generation 42
```

pero al ejecutar:

```text
Current:
Tenant A → Database Y
Generation 43
```

La unit deberá:

```text
REPLAN
```

o:

```text
FAIL
```

según política.

Nunca migrar X ciegamente.

---

# 105. Migration lease

Para evitar ejecuciones concurrentes conflictivas:

```php
interface TenantMigrationLeaseManager
{
    public function acquire(
        MigrationTargetId $target,
        TenantMigrationRunId $run
    ): TenantMigrationLease;
}
```

---

# 106. Lease ≠ database transaction

```text
Migration Lease
≠
Transaction
```

---

# 107. Lease purpose

Coordina:

```text
who may migrate target
```

no:

```text
atomic database changes
```

---

# 108. Lease properties

Podrá incluir:

```text
owner
target
run
fencing token
acquired_at
expires_at
heartbeat
```

---

# 109. Fencing token

Útil para impedir que un worker con lease expirado continúe escribiendo estado de control.

---

# 110. Distributed lock caveat

Un lock distribuido por sí solo no prueba que un worker viejo haya dejado de ejecutar.

---

# 111. Fencing

Preferir:

```text
lease
+
monotonic fencing token
```

cuando el backend lo permita.

---

# 112. Target lock

Además del lease lógico, algunas migrations pueden requerir locks de DB.

---

# 113. Lease ≠ DDL lock

Ambos conceptos son distintos.

---

# 114. Migration concurrency

Para miles de targets se necesitará concurrencia.

---

# 115. Unlimited concurrency

No:

```text
10,000 targets
→
10,000 simultaneous migrations
```

---

# 116. Resource governance

Se definirán límites por:

```text
global fleet
database server
cluster
region
shard
migration type
tenant class
```

---

# 117. Concurrency hierarchy

Ejemplo:

```text
Fleet max            100
Cluster max           20
Database host max      5
Heavy migration max    2
```

---

# 118. Adaptive concurrency

Opcionalmente podrá reducirse según:

```text
CPU
I/O
lock waits
replica lag
error rate
connection pressure
```

---

# 119. Resource governance authority

La integración usará:

```text
249_DATABASE_RESOURCE_GOVERNANCE_SYSTEM
```

---

# 120. Connection usage

Cada migration unit adquirirá conexiones mediante:

```text
Tenant Connection Resolution
```

---

# 121. No bypass

Migration no construirá DSNs directamente desde TenantId.

---

# 122. Schema-per-tenant

Resolverá:

```text
TenantSchemaDescriptor
```

---

# 123. Database-per-tenant

Resolverá:

```text
TenantConnectionResolution
```

---

# 124. Query context

Data migrations tenant-aware usarán:

```text
TenantQueryContext
```

cuando interactúen con el Query Engine.

---

# 125. Structural DDL

Schema Compiler seguirá siendo responsable de producir DDL.

---

# 126. Migration system ≠ SQL generator

La integración tenant no generará SQL manualmente.

---

# 127. Execution ownership

El Migration Executor canónico ejecutará los comandos.

---

# 128. Transactional DDL

Si la plataforma soporta DDL transaccional, puede utilizarse.

Pero:

```text
Transactional DDL
≠
Global Fleet Transaction
```

---

# 129. Per-unit transaction

Puede existir:

```text
BEGIN
migration target A
COMMIT
```

sin implicar atomicidad con target B.

---

# 130. Non-transactional DDL

Debe modelarse explícitamente.

---

# 131. Partial unit outcome

Una migration individual puede quedar parcialmente aplicada si la plataforma no ofrece atomicidad suficiente.

---

# 132. Unit outcome states

```text
NOT_STARTED
APPLIED
ROLLED_BACK
PARTIALLY_APPLIED
FAILED_BEFORE_EFFECT
FAILED_AFTER_EFFECT
UNKNOWN
```

---

# 133. Statement success ≠ migration success

---

# 134. Migration success ≠ fleet success

---

# 135. Retry

No todas las migrations son retry-safe.

---

# 136. Retry classification

```text
SAFE
SAFE_AFTER_VERIFICATION
IDEMPOTENT
REQUIRES_REPLAN
UNSAFE
UNKNOWN
```

---

# 137. UNKNOWN ≠ retryable

---

# 138. Retry after connection loss

Primero deberá determinarse:

```text
did DDL commit?
did schema change?
did repository update?
```

---

# 139. Introspection-assisted recovery

Puede utilizarse:

```text
Schema Introspection
Schema Diff
Migration Repository
```

para determinar estado.

---

# 140. Repository record missing

No demuestra:

```text
migration not applied
```

si la conexión falló después del DDL y antes de registrar historial.

---

# 141. Physical reality > assumption

VoltStack deberá reconciliar:

```text
Migration Repository Knowledge
```

con:

```text
Observed Schema Reality
```

---

# 142. Recovery state

Podrá clasificarse:

```text
CONFIRMED_APPLIED
CONFIRMED_NOT_APPLIED
PARTIALLY_APPLIED
INCONSISTENT
UNKNOWN
```

---

# 143. Forward recovery

En producción, muchas migrations serán más seguras mediante:

```text
forward fix
```

que rollback.

---

# 144. Rollback

Se integrará con:

```text
107_DATABASE_MIGRATION_ROLLBACK_SYSTEM
```

---

# 145. Rollback ≠ inverse guaranteed

No toda migration tiene inversa segura.

---

# 146. Tenant fleet rollback

Podrá ser:

```text
specific target
failed wave
completed wave
selected tenants
entire compatible fleet
```

---

# 147. Fleet rollback is another fleet operation

No será una reversión mágica global.

---

# 148. Destructive rollback

Deberá pasar por safety analysis nuevamente.

---

# 149. Data loss

Si rollback implica pérdida de datos:

```text
REJECT
```

por default salvo override privilegiado.

---

# 150. Pause

Un rollout podrá pausarse.

---

# 151. Pause semantics

```text
PAUSE
```

significa:

```text
do not start new units
```

por default.

No significa necesariamente cancelar DDL ya ejecutándose.

---

# 152. Cancel

Cancelación deberá distinguir:

```text
cancel queued work
request cancellation of active work
force termination
```

---

# 153. Force termination

Puede dejar outcome:

```text
UNKNOWN
```

---

# 154. Resume

Una fleet pausada podrá reanudarse desde estado persistido.

---

# 155. Checkpoint

El sistema podrá registrar:

```text
completed waves
completed units
failed units
unknown units
current migration
selection generation
fleet plan fingerprint
```

---

# 156. Checkpoint ≠ transaction log

---

# 157. Resume validation

Antes de resume:

```text
plan fingerprint valid
migration definitions unchanged or compatible
placements current
schema states reconcilable
leases expired/recovered
```

---

# 158. Migration checksum

Cada definition podrá tener:

```text
MigrationChecksum
```

---

# 159. Modified migration

Si una migration ya ejecutada cambia de checksum:

```text
DRIFT
```

o error según política.

---

# 160. Migration definition immutability

Una migration publicada/ejecutada no debería modificarse.

Preferir:

```text
new migration
```

---

# 161. Schema drift

Un tenant puede divergir del estado esperado.

---

# 162. Sources of drift

```text
manual DDL
failed migration
old software
partial restore
operator intervention
platform-specific behavior
```

---

# 163. Pre-migration drift detection

Para migrations sensibles podrá requerirse:

```text
ExpectedSchemaFingerprint
=
ObservedSchemaFingerprint
```

o una comprobación estructural equivalente.

---

# 164. Drift response

```text
WARN
SKIP
BLOCK
QUARANTINE
REPAIR
CUSTOM
```

---

# 165. Automatic repair

No deberá ser universal.

---

# 166. Drift ≠ migration pending

Son estados diferentes.

---

# 167. Tenant provisioning

Crear un tenant nuevo puede usar el mismo Migration Engine.

---

# 168. Provisioning strategy A

```text
Create empty target
      ↓
Run all migrations
```

---

# 169. Provisioning strategy B

```text
Create from baseline snapshot/template
      ↓
Verify baseline
      ↓
Run incremental migrations
```

---

# 170. Baseline template

Puede mejorar provisioning de grandes sistemas.

---

# 171. Template safety

Deberá estar versionado.

```text
BaselineVersion
SchemaFingerprint
Platform
CapabilityProfile
```

---

# 172. Template ≠ trusted forever

Debe verificarse.

---

# 173. Provisioning activation

Tenant sólo se marcará:

```text
ACTIVE
```

después de:

```text
schema/database created
migration target compatible
required migrations applied
verification passed
```

---

# 174. Provisioning race

Queries no deberán llegar a un target:

```text
PROVISIONING
```

salvo operaciones administrativas autorizadas.

---

# 175. Tenant mobility

Mover tenant:

```text
Database A
→
Database B
```

puede implicar versiones diferentes.

---

# 176. Target preparation

Antes de copiar:

```text
target schema/database
```

deberá estar en una versión compatible.

---

# 177. Migration during mobility

Deberá existir política:

```text
BLOCK_MIGRATIONS
ALLOW_COMPATIBLE
MIGRATE_SOURCE_AND_TARGET
CUSTOM
```

---

# 178. Recommended default

Durante cutover crítico:

```text
BLOCK_CONFLICTING_MIGRATIONS
```

---

# 179. Placement generation

Toda migration unit deberá validar:

```text
TenantPlacementGeneration
```

antes de side effects.

---

# 180. Cutover race

Evitar:

```text
Migration Worker → old database
Mobility Worker  → new database
```

ambos creyéndose autoridad.

---

# 181. Coordination

Migration y Mobility deberán compartir un:

```text
TenantPlacementOperationGuard
```

o mecanismo equivalente.

---

# 182. Backup integration

Antes de migrations de alto riesgo podrá requerirse:

```text
backup checkpoint
```

---

# 183. Backup ≠ rollback

Tener backup no convierte una migration destructiva en reversible instantáneamente.

---

# 184. Restore impact

Restore puede:

```text
take time
lose post-backup writes
require traffic stop
```

---

# 185. Backup evidence

Safety analyzer podrá registrar:

```text
backup exists
backup age
backup verification status
restore test status
```

---

# 186. Replica considerations

DDL puede afectar:

```text
replication lag
replica compatibility
read availability
```

---

# 187. Replica lag gates

Una wave podrá pausarse si:

```text
lag > threshold
```

---

# 188. Lag UNKNOWN

No se interpretará como healthy.

---

# 189. Failover

Una migration no deberá continuar ciegamente si ocurre failover.

---

# 190. Authority epoch

El plan podrá bindear:

```text
DatabaseAuthorityEpoch
```

cuando corresponda.

---

# 191. Epoch change

Puede requerir:

```text
pause
revalidate
replan
```

---

# 192. Cache integration

Después de migration pueden invalidarse:

```text
Schema Metadata Cache
ORM Metadata Cache
Compiled Query Cache
Query Plan Cache
Result Cache
Entity Cache
```

según el cambio.

---

# 193. No FLUSHALL

Nunca usar invalidación global indiscriminada como default.

---

# 194. Generation-based invalidation

Preferir:

```text
SchemaGeneration
MetadataGeneration
```

para volver obsoletas entradas anteriores.

---

# 195. Metadata compilation

Se integrará con:

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM
```

---

# 196. Post-migration metadata

Podrá recompilarse:

```text
eagerly
lazily
per wave
per target
```

---

# 197. Prepared statements

DDL puede invalidar prepared statements.

Connection/statement caches deberán reaccionar según platform capabilities.

---

# 198. Persistent workers

Después de schema change, workers pueden conservar:

```text
old metadata
old query plans
old prepared statements
```

---

# 199. Runtime notification

VoltStack podrá emitir:

```text
DatabaseSchemaGenerationChanged
```

---

# 200. Worker response

Puede:

```text
invalidate caches
refresh metadata
drain worker
restart worker
```

según severidad.

---

# 201. FrankenPHP

Como runtime predeterminado, la integración deberá garantizar que una migration no dependa de reiniciar PHP después de cada cambio si puede invalidarse estado de forma segura.

---

# 202. RoadRunner

Workers persistentes deberán observar generation changes.

---

# 203. OpenSwoole

Igualmente, sin depender de globals mutables compartidos entre coroutines.

---

# 204. Runtime compatibility gate

Una contract migration podrá requerir:

```text
all active workers >= generation X
```

---

# 205. Jobs

Jobs antiguos pueden seguir esperando schema anterior.

---

# 206. Job compatibility

El deployment planner deberá considerar:

```text
queued jobs
scheduled jobs
retry jobs
dead-letter jobs
```

cuando una migration sea incompatible hacia atrás.

---

# 207. Migration jobs

La propia fleet migration puede ejecutarse mediante Job System.

---

# 208. Migration orchestration ≠ queue semantics

El Migration System seguirá siendo autoridad de estado.

La queue sólo transportará trabajo.

---

# 209. Duplicate delivery

Si un migration job se entrega dos veces:

```text
lease
repository state
fencing token
```

deberán evitar ejecución insegura.

---

# 210. Exactly-once

No se prometerá:

```text
exactly-once migration execution
```

sólo porque se use una queue.

---

# 211. Idempotency

Las operations deberán diseñarse para:

```text
detect previous completion
verify physical state
avoid duplicate unsafe effects
```

cuando sea posible.

---

# 212. Event system

Eventos conceptuales:

```text
TenantMigrationFleetPlanned
TenantMigrationFleetStarted
TenantMigrationWaveStarted
TenantMigrationUnitStarted
TenantMigrationUnitSucceeded
TenantMigrationUnitFailed
TenantMigrationUnitUnknown
TenantMigrationWaveCompleted
TenantMigrationFleetPaused
TenantMigrationFleetCompleted
TenantMigrationDriftDetected
TenantMigrationCompatibilityViolation
```

---

# 213. Event timing

Debe distinguirse:

```text
before execution
after physical effect
after repository recording
after verification
```

---

# 214. Event failure

Un listener fallando después de DDL exitoso:

```text
does not undo DDL
```

---

# 215. Events ≠ migration outcome

---

# 216. Telemetry

Métricas:

```text
database_tenant_migration_units_total
database_tenant_migration_failures_total
database_tenant_migration_unknown_total
database_tenant_migration_duration
database_tenant_migration_wave_duration
database_tenant_migration_active_units
database_tenant_migration_drift_total
```

---

# 217. Metric labels

Permitidos:

```text
platform
migration_class
wave
outcome
risk_class
target_kind
```

con cardinalidad controlada.

---

# 218. TenantId labels

No por default.

---

# 219. Migration ID cardinality

También deberá evaluarse antes de usarlo como metric label.

---

# 220. Tracing

Una fleet podrá tener:

```text
Fleet Span
 ├── Wave Span
 │    ├── Unit Span
 │    └── Unit Span
 └── Wave Span
```

---

# 221. Trace context

Puede incluir:

```text
run id
wave id
target kind
migration count
risk classification
```

---

# 222. Logging

Logs estructurados deberán usar IDs/fingerprints seguros.

---

# 223. Audit

Toda operación de fleet migration será auditable.

---

# 224. Audit fields

```text
actor
run
migration set
selection
approval
safety override
start
pause
resume
cancel
rollback
outcome
```

---

# 225. Security

Sólo actores autorizados podrán:

```text
start fleet
pause fleet
resume fleet
cancel fleet
override safety
rollback
select production tenants
run destructive migrations
```

---

# 226. Safety override

Requerirá:

```text
explicit capability
reason
audit record
```

---

# 227. Migration code trust

Las migration definitions son código privilegiado.

No deben aceptar SQL arbitrario proveniente de usuarios.

---

# 228. Secrets

Migration telemetry/logs no expondrán:

```text
database passwords
DSNs
tenant secrets
credentials
```

---

# 229. Tenant selection authorization

Un operador autorizado para Tenant A no obtiene automáticamente permiso sobre todos los tenants.

---

# 230. Cross-tenant fleet capability

Una fleet general requerirá autorización correspondiente.

---

# 231. CLI

Ejemplos conceptuales:

```text
php volt migrate --tenants
```

---

# 232. Single tenant

```text
php volt migrate --tenant=acme
```

---

# 233. Canary

```text
php volt migrate --tenants --wave=canary
```

---

# 234. Plan only

```text
php volt migrate --tenants --plan
```

---

# 235. Explain

```text
php volt migrate --tenants --explain
```

---

# 236. Dry run

```text
php volt migrate --tenants --dry-run
```

---

# 237. Dry run limitation

```text
Dry Run
≠
Proof of production success
```

---

# 238. Status

```text
php volt migrate:tenants:status
```

---

# 239. Resume

```text
php volt migrate:tenants:resume <run>
```

---

# 240. Pause

```text
php volt migrate:tenants:pause <run>
```

---

# 241. Failed targets

```text
php volt migrate:tenants:failed <run>
```

---

# 242. Retry

```text
php volt migrate:tenants:retry <run> --failed
```

pero sólo para units clasificadas retry-safe.

---

# 243. Diagnostics example

```text
Tenant Migration Fleet
────────────────────────────────────────

Run
  Status:              RUNNING
  Strategy:            WAVE_MAJOR

Selection
  Tenants:             10,000
  Physical Targets:    8,421

Migrations
  Pending:             3
  Compatibility:       118..121

Current Wave
  Name:                general-2
  Targets:             2,000
  Completed:           1,641
  Running:             20
  Failed:              3
  Unknown:             0

Concurrency
  Fleet Limit:         100
  Host Limit:          5

Safety
  Destructive:         no
  Drift Check:         enabled
  Placement Validation: enabled

Overall
  Status:              RUNNING
```

---

# 244. Explain output

```text
Migration Fleet Plan
────────────────────────────────────────

1. Migration scope resolved as TENANT.
2. 10,000 tenant candidates selected.
3. Current placements resolved.
4. Candidates mapped to 8,421 physical targets.
5. Shared targets deduplicated.
6. Schema versions inspected.
7. 41 incompatible targets excluded.
8. Safety analysis completed.
9. Canary cohort selected.
10. Remaining targets divided into bounded waves.
11. Per-host concurrency limits applied.
12. Placement generation checks required before execution.
13. Schema drift checks enabled.
14. Retry classification calculated.
15. Contract phase blocked until old workers drain.
```

---

# 245. Failure example

Supongamos:

```text
Wave 4
Targets: 1,000

997 success
2 failed
1 unknown
```

VoltStack deberá reportar:

```text
Wave Status:
PARTIALLY_SUCCEEDED
```

y detener/continuar según gate policy.

Nunca:

```text
Success: 100%
```

---

# 246. UNKNOWN example

```text
ALTER TABLE sent
connection lost
```

No deberá asumirse:

```text
FAILED
```

ni:

```text
SUCCEEDED
```

Se deberá ejecutar:

```text
reconciliation
```

---

# 247. Reconciliation

Pipeline:

```text
UNKNOWN
   ↓
Migration Repository Inspection
   ↓
Schema Introspection
   ↓
Schema Diff
   ↓
Physical State Classification
   ↓
CONFIRMED_APPLIED
or
CONFIRMED_NOT_APPLIED
or
PARTIAL
or
UNKNOWN
```

---

# 248. Fleet recovery

Una fleet deberá poder continuar aunque algunas units requieran intervención.

---

# 249. Quarantine

Un target problemático podrá pasar a:

```text
QUARANTINED
```

fuera del rollout general.

---

# 250. Quarantine semantics

Significa:

```text
do not automatically migrate further
```

hasta resolución.

---

# 251. Application routing

Si schema incompatible:

```text
Tenant Availability Policy
```

puede decidir:

```text
normal
read-only
maintenance
reject
```

---

# 252. Migration system ≠ traffic router

Sólo publica estado/evidencia.

Otro sistema decide routing de aplicación.

---

# 253. Testing architecture

Se requerirán:

```text
Migration Scope Tests
Target Resolution Tests
Target Deduplication Tests
Fleet Planning Tests
Wave Planning Tests
Canary Tests
Compatibility Tests
Version Skew Tests
Lease Tests
Fencing Tests
Concurrency Tests
Safety Tests
Schema Drift Tests
Retry Tests
Unknown Outcome Tests
Rollback Tests
Resume Tests
Provisioning Tests
Mobility Tests
Persistent Runtime Tests
Cache Invalidation Tests
Telemetry Tests
Security Tests
Fault Injection Tests
Performance Tests
```

---

# 254. Shared-schema test

```text
10 tenants
1 shared table
```

debe producir:

```text
1 structural migration target
```

---

# 255. Schema-per-tenant test

```text
10 tenants
10 schemas
```

debe producir:

```text
10 targets
```

---

# 256. Dedicated DB test

```text
10 tenants
10 databases
```

produce:

```text
10 targets
```

---

# 257. Hybrid test

Mezclar:

```text
shared
schema
database
shard
```

y comprobar target resolution.

---

# 258. Deduplication test

Dos tenants compartiendo el mismo target no deberán ejecutar el mismo DDL dos veces.

---

# 259. Data migration test

Aun compartiendo schema, un backfill tenant-scoped podrá producir unidades lógicas separadas.

---

# 260. Placement race test

Mover tenant entre planning y execution.

Debe detectarse generation mismatch.

---

# 261. Lease expiry test

Worker A pierde lease.

Worker B obtiene nuevo fencing token.

A no deberá publicar estado posterior como autoridad.

---

# 262. Concurrent fleet test

Dos fleets incompatibles sobre mismo target deberán coordinarse/rechazarse.

---

# 263. Failure injection

Fallar en:

```text
before DDL
during DDL
after DDL
before repository write
after repository write
during verification
```

---

# 264. Unknown commit test

Simular pérdida de conexión en punto ambiguo.

---

# 265. Drift test

Modificar manualmente schema antes de migration.

---

# 266. Resume test

Detener proceso después de varias waves y reiniciar en otro worker.

---

# 267. Persistent runtime test

Mantener workers antiguos durante migration y comprobar compatibility gates.

---

# 268. Contract migration test

No deberá ejecutarse mientras workers incompatibles permanezcan activos.

---

# 269. Queue duplicate test

Entregar dos veces la misma unit.

---

# 270. Backup gate test

Migration marcada como backup-required deberá bloquearse si backup evidence no satisface policy.

---

# 271. Replica lag test

Lag excesivo deberá activar wave gate cuando esté configurado.

---

# 272. Failover test

Cambiar authority epoch durante migration.

---

# 273. Cache generation test

Después de migration, metadata/query caches antiguas deberán quedar obsoletas.

---

# 274. Large fleet benchmark

Probar:

```text
100
1,000
10,000
100,000
```

targets lógicos, incluso si no todos son físicos.

Medir:

```text
planning time
memory
repository throughput
lease throughput
scheduler overhead
telemetry overhead
```

---

# 275. Proposed directory

```text
src/Quantum/Multitenancy/Database/Migration/
│
├── Contract/
│   ├── TenantMigrationTargetResolver.php
│   ├── TenantMigrationSelector.php
│   ├── TenantMigrationFleetPlanner.php
│   ├── TenantMigrationFleetRunner.php
│   ├── TenantMigrationLeaseManager.php
│   └── TenantMigrationRepository.php
│
├── Context/
│   ├── TenantMigrationContext.php
│   ├── TenantMigrationRunId.php
│   ├── TenantMigrationScope.php
│   └── TenantMigrationSelectionContext.php
│
├── Target/
│   ├── TenantMigrationTarget.php
│   ├── MigrationTargetId.php
│   ├── SharedSchemaMigrationTarget.php
│   ├── TenantSchemaMigrationTarget.php
│   ├── TenantDatabaseMigrationTarget.php
│   ├── ShardMigrationTarget.php
│   └── TenantMigrationTargetResolver.php
│
├── Fleet/
│   ├── TenantMigrationFleetPlan.php
│   ├── TenantMigrationFleetState.php
│   ├── TenantMigrationUnit.php
│   ├── TenantMigrationUnitState.php
│   ├── TenantMigrationWave.php
│   └── TenantMigrationWaveState.php
│
├── Selection/
│   ├── AllTenantMigrationSelector.php
│   ├── ExplicitTenantMigrationSelector.php
│   ├── VersionTenantMigrationSelector.php
│   ├── RegionTenantMigrationSelector.php
│   └── CanaryTenantMigrationSelector.php
│
├── Planning/
│   ├── DefaultTenantMigrationFleetPlanner.php
│   ├── TenantMigrationTargetDeduplicator.php
│   ├── TenantMigrationWavePlanner.php
│   ├── TenantMigrationDependencyPlanner.php
│   └── TenantMigrationCompatibilityPlanner.php
│
├── Execution/
│   ├── TenantMigrationFleetRunner.php
│   ├── TenantMigrationUnitRunner.php
│   ├── TenantMigrationWaveRunner.php
│   ├── TenantMigrationPreflightValidator.php
│   └── TenantMigrationVerifier.php
│
├── Lease/
│   ├── TenantMigrationLease.php
│   ├── TenantMigrationLeaseManager.php
│   ├── TenantMigrationFencingToken.php
│   └── TenantMigrationLeaseException.php
│
├── Compatibility/
│   ├── TenantSchemaCompatibilityWindow.php
│   ├── TenantMigrationCompatibilityResult.php
│   ├── TenantMigrationVersionAnalyzer.php
│   └── TenantMigrationWorkerCompatibilityGuard.php
│
├── Safety/
│   ├── TenantMigrationSafetyPolicy.php
│   ├── TenantMigrationRiskAnalyzer.php
│   ├── TenantMigrationBackupGuard.php
│   └── TenantMigrationDriftGuard.php
│
├── Recovery/
│   ├── TenantMigrationRecoveryManager.php
│   ├── TenantMigrationReconciler.php
│   ├── TenantMigrationRetryClassifier.php
│   └── TenantMigrationUnknownOutcomeResolver.php
│
├── Runtime/
│   ├── TenantMigrationRuntimeCoordinator.php
│   ├── TenantMigrationWorkerGenerationGuard.php
│   └── TenantMigrationCacheInvalidator.php
│
├── Telemetry/
│   └── TenantMigrationTelemetry.php
│
├── Diagnostics/
│   └── TenantMigrationDiagnostics.php
│
└── Exception/
    ├── TenantMigrationException.php
    ├── TenantMigrationPlanningException.php
    ├── TenantMigrationCompatibilityException.php
    ├── TenantMigrationPlacementChangedException.php
    ├── TenantMigrationLeaseException.php
    ├── TenantMigrationDriftException.php
    ├── TenantMigrationSafetyException.php
    ├── TenantMigrationRecoveryException.php
    └── TenantMigrationUnknownOutcomeException.php
```

---

# 276. Dependency direction

```text
Multitenancy Migration Integration
            ↓
Canonical Migration Engine
            ↓
Schema System
            ↓
Compiler
            ↓
Execution Engine
            ↓
Connection Manager
            ↓
Driver
```

Nunca:

```text
Driver
→
TenantMigrationFleetPlanner
```

---

# 277. Integraciones

```text
Tenant Migration
│
├── Tenant Placement
├── Tenant Connection Resolution
├── Tenant Database Isolation
├── Tenant Schema Isolation
├── Tenant Query Context
├── Migration Engine
├── Schema Engine
├── Transactions
├── Cache
├── Events
├── Telemetry
├── Security
├── Resource Governance
├── Jobs
└── Runtime
```

---

# 278. Invariantes

## DB-TMI-001

Multitenancy no creará un segundo Migration Engine.

## DB-TMI-002

Tenant migration utilizará el Migration Engine canónico.

## DB-TMI-003

Migration Definition será distinta de Tenant Migration Unit.

## DB-TMI-004

Tenant Migration Unit será distinta de Fleet Migration.

## DB-TMI-005

Fleet Migration no será una global ACID transaction.

## DB-TMI-006

Migration scope será explícito.

## DB-TMI-007

GLOBAL no significará all-tenants.

## DB-TMI-008

TENANT no significará necesariamente un physical target por tenant.

## DB-TMI-009

Tenant count será distinto de physical target count.

## DB-TMI-010

Shared-schema structural migrations serán deduplicadas.

## DB-TMI-011

Data migration targeting podrá diferir de structural targeting.

## DB-TMI-012

Schema-per-tenant producirá target por schema cuando corresponda.

## DB-TMI-013

Database-per-tenant producirá target por database cuando corresponda.

## DB-TMI-014

Sharding será target-aware.

## DB-TMI-015

Hybrid placement será soportable.

## DB-TMI-016

Migration definitions no deberán conocer physical tenant placement.

## DB-TMI-017

Fleet planner resolverá placements.

## DB-TMI-018

Fleet planner deduplicará physical targets.

## DB-TMI-019

Fleet planning será distinto de execution.

## DB-TMI-020

Tenant selection será explícita.

## DB-TMI-021

Tenant selection podrá ser parcial.

## DB-TMI-022

Dynamic tenant creation tendrá policy explícita.

## DB-TMI-023

New tenants podrán provisionarse directamente en compatible version.

## DB-TMI-024

Fleet plans serán inmutables.

## DB-TMI-025

Migration dependencies serán preservadas.

## DB-TMI-026

Fleet ordering será explícito.

## DB-TMI-027

Waves serán soportadas.

## DB-TMI-028

Canaries serán soportados.

## DB-TMI-029

Canary success no probará fleet success.

## DB-TMI-030

Wave gates serán configurables.

## DB-TMI-031

No-error no equivaldrá a verified success.

## DB-TMI-032

Unit state será persistible.

## DB-TMI-033

Fleet state será persistible.

## DB-TMI-034

PARTIALLY_SUCCEEDED será estado válido.

## DB-TMI-035

UNKNOWN será estado válido.

## DB-TMI-036

UNKNOWN no será convertido a FAILED arbitrariamente.

## DB-TMI-037

UNKNOWN no será convertido a SUCCEEDED arbitrariamente.

## DB-TMI-038

Migration Repository seguirá siendo autoridad canónica de migration history.

## DB-TMI-039

Repository será target-aware.

## DB-TMI-040

Per-tenant migration table no será requisito universal.

## DB-TMI-041

Centralized repository será soportable.

## DB-TMI-042

Per-target repository será soportable.

## DB-TMI-043

Hybrid repository tendrá autoridad claramente definida.

## DB-TMI-044

Control-plane index no equivaldrá automáticamente a physical truth.

## DB-TMI-045

Schema version skew será representable.

## DB-TMI-046

Version skew no equivaldrá automáticamente a failure.

## DB-TMI-047

Application/schema compatibility será explícita.

## DB-TMI-048

Too-old schemas serán detectables.

## DB-TMI-049

Too-new schemas serán detectables.

## DB-TMI-050

Expand/contract será soportado.

## DB-TMI-051

Expand ocurrirá antes de contract.

## DB-TMI-052

Contract requerirá compatibility evidence.

## DB-TMI-053

Migration completed no implicará all workers updated.

## DB-TMI-054

Persistent worker generations podrán participar en safety gates.

## DB-TMI-055

Tenant ZDT utilizará el ZDT system canónico.

## DB-TMI-056

Online DDL será capability-driven.

## DB-TMI-057

Database version no equivaldrá a capability.

## DB-TMI-058

Migration safety será evaluada por target.

## DB-TMI-059

Same migration podrá tener diferente risk por tenant.

## DB-TMI-060

UNKNOWN risk no equivaldrá a LOW.

## DB-TMI-061

Statistics serán evidence, no certainty.

## DB-TMI-062

Preflight validation ocurrirá antes de side effects.

## DB-TMI-063

Placement generation será validada antes de execution.

## DB-TMI-064

Stale placement no será migrado ciegamente.

## DB-TMI-065

Migration lease será distinto de transaction.

## DB-TMI-066

Migration lease será distinto de DDL lock.

## DB-TMI-067

Lease podrá usar fencing.

## DB-TMI-068

Expired worker no deberá seguir publicando authoritative state.

## DB-TMI-069

Concurrency será bounded.

## DB-TMI-070

Concurrency podrá ser jerárquica.

## DB-TMI-071

Resource pressure podrá reducir concurrency.

## DB-TMI-072

Migration usará Tenant Connection Resolution.

## DB-TMI-073

Migration no construirá DSNs desde TenantId.

## DB-TMI-074

Schema-per-tenant usará TenantSchemaResolution.

## DB-TMI-075

Data migrations podrán usar TenantQueryContext.

## DB-TMI-076

Tenant integration no generará SQL directamente.

## DB-TMI-077

Schema Compiler seguirá generando DDL.

## DB-TMI-078

Migration Executor seguirá ejecutando DDL.

## DB-TMI-079

Transactional DDL no implicará fleet transaction.

## DB-TMI-080

Per-target transactions no implicarán global atomicity.

## DB-TMI-081

Non-transactional DDL será representado explícitamente.

## DB-TMI-082

Partial application será representable.

## DB-TMI-083

Statement success no equivaldrá a migration success.

## DB-TMI-084

Migration success no equivaldrá a fleet success.

## DB-TMI-085

Retry safety será clasificada.

## DB-TMI-086

UNKNOWN retry safety no será retryable.

## DB-TMI-087

Connection loss requerirá reconciliation cuando outcome sea ambiguo.

## DB-TMI-088

Missing repository record no probará absence of physical effect.

## DB-TMI-089

Physical schema reality será verificable.

## DB-TMI-090

Recovery podrá usar introspection.

## DB-TMI-091

Recovery podrá usar schema diff.

## DB-TMI-092

Forward recovery será first-class.

## DB-TMI-093

Rollback no será considerado siempre seguro.

## DB-TMI-094

Fleet rollback será otra fleet operation.

## DB-TMI-095

Destructive rollback pasará safety analysis.

## DB-TMI-096

Data-loss rollback será rechazado por default.

## DB-TMI-097

Pause no significará force-cancel active DDL.

## DB-TMI-098

Cancel semantics serán explícitas.

## DB-TMI-099

Forced cancellation podrá producir UNKNOWN.

## DB-TMI-100

Fleet execution será resumable.

## DB-TMI-101

Resume validará plan state.

## DB-TMI-102

Checkpoint no será transaction log.

## DB-TMI-103

Migration definitions tendrán checksum.

## DB-TMI-104

Executed migration mutation será detectable.

## DB-TMI-105

Published migrations deberán considerarse immutable.

## DB-TMI-106

Schema drift será distinto de pending migration.

## DB-TMI-107

Drift podrá bloquear migrations.

## DB-TMI-108

Automatic drift repair no será universal.

## DB-TMI-109

Provisioning podrá reutilizar Migration Engine.

## DB-TMI-110

Provisioning podrá usar baseline templates.

## DB-TMI-111

Baseline templates serán versionados.

## DB-TMI-112

Baseline template será verificable.

## DB-TMI-113

Tenant no será ACTIVE antes de migration verification.

## DB-TMI-114

PROVISIONING tenant no recibirá traffic normal.

## DB-TMI-115

Mobility y migration deberán coordinarse.

## DB-TMI-116

Migration no continuará sobre stale source después de cutover.

## DB-TMI-117

Migration unit será placement-generation aware.

## DB-TMI-118

Backup podrá ser safety evidence.

## DB-TMI-119

Backup no equivaldrá a instant rollback.

## DB-TMI-120

Replica lag podrá formar parte de wave gates.

## DB-TMI-121

UNKNOWN replica lag no equivaldrá a healthy.

## DB-TMI-122

Failover durante migration requerirá revalidation.

## DB-TMI-123

Authority epoch podrá bindear execution.

## DB-TMI-124

Schema migrations invalidarán caches semánticamente.

## DB-TMI-125

No se utilizará FLUSHALL como default.

## DB-TMI-126

Generation-based invalidation será preferida.

## DB-TMI-127

Metadata compilation será migration-aware.

## DB-TMI-128

DDL podrá invalidar prepared statements.

## DB-TMI-129

Persistent workers podrán contener stale metadata.

## DB-TMI-130

Schema generation changes serán propagables.

## DB-TMI-131

FrankenPHP integration será persistent-runtime safe.

## DB-TMI-132

RoadRunner integration será persistent-runtime safe.

## DB-TMI-133

OpenSwoole integration será coroutine-safe.

## DB-TMI-134

Contract migrations podrán esperar worker drainage.

## DB-TMI-135

Queued jobs serán considerados en incompatible contract changes.

## DB-TMI-136

Queue delivery no definirá migration truth.

## DB-TMI-137

Duplicate migration job delivery será tolerable.

## DB-TMI-138

Queue no garantizará exactly-once migration.

## DB-TMI-139

Migration events no redefinirán migration outcome.

## DB-TMI-140

Listener failure después de DDL no deshará DDL.

## DB-TMI-141

Telemetry será bounded.

## DB-TMI-142

TenantId no será metric label por default.

## DB-TMI-143

Migration fleet será traceable.

## DB-TMI-144

Migration operations serán auditables.

## DB-TMI-145

Safety overrides serán auditables.

## DB-TMI-146

Destructive migrations requerirán authorization apropiada.

## DB-TMI-147

Migration definitions serán privileged code.

## DB-TMI-148

Migration logs no expondrán credentials.

## DB-TMI-149

Tenant selection respetará authorization.

## DB-TMI-150

Cross-tenant fleet requerirá explicit capability.

## DB-TMI-151

CLI permitirá planning sin execution.

## DB-TMI-152

Dry run no equivaldrá a production proof.

## DB-TMI-153

Fleet diagnostics expondrán partial outcomes.

## DB-TMI-154

Unknown units serán visibles.

## DB-TMI-155

Failed units serán individualmente recuperables.

## DB-TMI-156

Targets podrán ser quarantined.

## DB-TMI-157

Quarantined target no continuará automáticamente.

## DB-TMI-158

Migration system no decidirá application traffic routing.

## DB-TMI-159

Migration system publicará compatibility state.

## DB-TMI-160

Shared-schema target deduplication tendrá tests.

## DB-TMI-161

Schema-per-tenant targeting tendrá tests.

## DB-TMI-162

Database-per-tenant targeting tendrá tests.

## DB-TMI-163

Hybrid targeting tendrá tests.

## DB-TMI-164

Placement races tendrán tests.

## DB-TMI-165

Lease/fencing tendrán tests.

## DB-TMI-166

Unknown outcomes tendrán fault tests.

## DB-TMI-167

Resume tendrá tests.

## DB-TMI-168

Version skew tendrá tests.

## DB-TMI-169

Persistent worker compatibility tendrá tests.

## DB-TMI-170

Large fleets tendrán benchmarks.

## DB-TMI-171

Fleet planner no ejecutará DDL.

## DB-TMI-172

Fleet runner no compilará SQL directamente.

## DB-TMI-173

Tenant integration no duplicará Schema Engine.

## DB-TMI-174

Tenant integration no duplicará Connection Manager.

## DB-TMI-175

Tenant integration no duplicará Transaction Manager.

## DB-TMI-176

Tenant integration no duplicará Query Engine.

## DB-TMI-177

Tenant migration state será scoped, no static global.

## DB-TMI-178

Migration workers podrán reiniciarse sin perder authoritative fleet state.

## DB-TMI-179

No se fingirá rollback de units ya committed.

## DB-TMI-180

Toda fleet migration deberá poder explicar qué targets fueron modificados, cuáles no, cuáles fallaron y cuáles permanecen UNKNOWN.

---

# 279. Modelo formal

Sea:

```text
M = conjunto de migrations
T = conjunto de tenants seleccionados
P(t) = placement del tenant t
R(t,m) = physical migration target
U = conjunto de migration units
```

La planificación deberá calcular:

```text
U =
Unique {
    R(t,m)
    |
    t ∈ T,
    m ∈ M
}
```

cuando la operación sea estructural y los targets puedan deduplicarse.

---

# 280. Data migration formal

Para data migrations tenant-scoped:

```text
Udata =
{
    (t, R(t,m), m)
    |
    t ∈ T,
    m ∈ M
}
```

porque dos tenants que comparten tabla física pueden seguir necesitando trabajo lógico independiente.

---

# 281. Fleet outcome

Sea:

```text
S = successful units
F = failed units
K = unknown units
P = pending units
```

Fleet sólo será:

```text
SUCCEEDED
```

si:

```text
|F| = 0
∧
|K| = 0
∧
|P| = 0
```

y todas las verificaciones obligatorias fueron satisfechas.

---

# 282. Partial success

Si:

```text
|S| > 0
```

y:

```text
|F| + |K| > 0
```

el sistema deberá poder representar:

```text
PARTIALLY_SUCCEEDED
```

---

# 283. Execution safety

Para ejecutar una unit `u`:

```text
Executable(u)
=
LeaseValid(u)
∧ PlacementCurrent(u)
∧ DependenciesSatisfied(u)
∧ CompatibilityValid(u)
∧ SafetyAccepted(u)
∧ TargetReachable(u)
∧ NoConflictingOperation(u)
```

---

# 284. Retry safety

```text
Retryable(u)
=
OutcomeKnownSafeForRetry(u)
∧ MigrationRetryPolicyAllows(u)
∧ PlacementStillCompatible(u)
```

Por tanto:

```text
Outcome(u) = UNKNOWN
```

no implica:

```text
Retryable(u) = true
```

---

# 285. Arquitectura general

```text
                       MIGRATION DEFINITIONS
                               │
                               ▼
                       Migration Discovery
                               │
                               ▼
                       Migration Metadata
                               │
                               ▼
                         Tenant Selector
                               │
                               ▼
                         Tenant Candidates
                               │
                               ▼
                    Tenant Placement Resolver
                               │
                               ▼
                    Physical Target Resolver
                               │
                               ▼
                      Target Deduplication
                               │
                               ▼
                      Fleet Migration Planner
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        Compatibility        Safety          Resources
              │                │                │
              └────────────────┼────────────────┘
                               ▼
                         Rollout Waves
                               │
                               ▼
                         Fleet Runner
                               │
                               ▼
                            Wave
                               │
                               ▼
                       Migration Units
                               │
                      ┌────────┴────────┐
                      ▼                 ▼
                 Acquire Lease       BLOCKED
                      │
                      ▼
               Validate Placement
                      │
                      ▼
                  Preflight
                      │
                      ▼
              Canonical Migration
                  Planner
                      │
                      ▼
               Schema Compiler
                      │
                      ▼
              Migration Executor
                      │
                      ▼
                   Database
                      │
                      ▼
                 Verification
                      │
          ┌───────────┼────────────┐
          ▼           ▼            ▼
       SUCCESS      FAILURE      UNKNOWN
          │           │            │
          ▼           ▼            ▼
      Repository   Recovery    Reconciliation
          │           │            │
          └───────────┼────────────┘
                      ▼
                 Fleet State
                      │
                      ▼
                  Wave Gate
                      │
             ┌────────┴─────────┐
             ▼                  ▼
          CONTINUE            PAUSE
```

---

# 286. Ejemplo de migración híbrida

Supongamos:

```text
1,000 tenants
```

distribuidos así:

```text
600 → shared schema
300 → individual schemas
90  → dedicated databases
10  → dedicated shards
```

Una migration estructural no necesariamente produce:

```text
1,000 DDL executions
```

El Fleet Planner podría obtener:

```text
1 shared target
+
300 schema targets
+
90 database targets
+
10 shard targets
=
401 physical targets
```

---

# 287. Data backfill del mismo ejemplo

Si cada tenant requiere un backfill independiente:

```text
1,000 logical data units
```

aunque sólo existan:

```text
401 physical structural targets
```

Esto demuestra la separación:

```text
Tenant
≠
Physical Migration Target
≠
Logical Data Work Unit
```

---

# 288. Estrategia recomendada

Para VoltStack se recomienda que una migration tenant-aware atraviese:

```text
Migration Definition
      ↓
Scope Classification
      ↓
Tenant Selection
      ↓
Placement Resolution
      ↓
Physical Target Resolution
      ↓
Deduplication
      ↓
Compatibility Analysis
      ↓
Safety Analysis
      ↓
Wave Planning
      ↓
Bounded Execution
      ↓
Verification
      ↓
Repository Update
      ↓
Generation/Cache Invalidation
      ↓
Fleet State
```

---

# 289. Regla arquitectónica definitiva

> **VoltStack tratará una migración multitenant como una operación de flota sobre dominios de persistencia y targets físicos explícitos. El número de tenants no determinará directamente el número de ejecuciones DDL; el TenantPlacement y la estrategia de aislamiento determinarán los targets reales. Cada unidad tendrá estado, lease, generation, compatibility y outcome propios. Las waves podrán avanzar gradualmente y la plataforma conservará explícitamente los estados parciales y UNKNOWN sin fingir una transacción distribuida global.**

En consecuencia:

```text
Tenant Count
≠
DDL Count
```

```text
Migration Success
≠
Fleet Success
```

```text
Fleet Migration
≠
Global Transaction
```

```text
Rollback
≠
Guaranteed Inverse
```

```text
Repository Record
≠
Physical Reality
```

```text
Migration Complete
≠
All Workers Compatible
```

y:

```text
UNKNOWN
≠
FAILED
≠
SUCCEEDED
```

---

# 290. Resultado del bloque de Multitenancy

Con los documentos `261–266`, la arquitectura Database dispone ahora de:

```text
Multitenancy Integration
│
├── Tenant Isolation Architecture
│
├── Tenant Connection Resolution
│
├── Tenant Database Isolation
│
├── Tenant Schema Isolation
│
├── Tenant Query Context
│
└── Tenant Migration Integration
```

permitiendo que:

```text
Business Query
Business Entity
Repository
Migration Definition
```

permanezcan separados de:

```text
Physical Tenant Placement
```

---

# 291. Bloque 26 completado

```text
BLOCK 26 — MULTITENANCY INTEGRATION

✓ 261_DATABASE_MULTITENANCY_INTEGRATION_ARCHITECTURE.md
✓ 262_DATABASE_TENANT_CONNECTION_RESOLUTION_SYSTEM.md
✓ 263_DATABASE_TENANT_DATABASE_ISOLATION_SYSTEM.md
✓ 264_DATABASE_TENANT_SCHEMA_ISOLATION_SYSTEM.md
✓ 265_DATABASE_TENANT_QUERY_CONTEXT_SYSTEM.md
✓ 266_DATABASE_TENANT_MIGRATION_INTEGRATION_SYSTEM.md
```

---

# 292. Siguiente bloque

A partir del siguiente documento comienza:

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES
```

con:

```text
267_DATABASE_TEMPORAL_DATA_SYSTEM.md
268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
269_DATABASE_SOFT_DELETE_SYSTEM.md
270_DATABASE_DATA_RETENTION_SYSTEM.md
271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
273_DATABASE_JSON_QUERY_SYSTEM.md
274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 293. Siguiente documento

```text
267_DATABASE_TEMPORAL_DATA_SYSTEM.md
```

El siguiente documento definirá la arquitectura temporal de VoltStack Database para distinguir y modelar correctamente:

```text
Instant
LocalDate
LocalTime
LocalDateTime
ZonedDateTime
Business Time
Valid Time
Transaction Time
System Time
Temporal Interval
Temporal Range
Temporal Precision
Temporal Ordering
Temporal Query
Temporal Predicate
As-Of Query
Temporal Snapshot
Temporal Relationship
Temporal Consistency
Temporal Indexing
Temporal Partitioning
Temporal Retention
```

y establecerá la separación crítica:

```text
Temporal Data
≠
DateTime Casting
≠
Entity History
≠
Audit Log
≠
Soft Delete
≠
Database Transaction Timestamp
```

bajo la regla:

> **El sistema temporal de VoltStack deberá representar explícitamente qué significa el tiempo dentro del dominio y de la persistencia; un timestamp almacenado en una columna no convierte por sí solo a una entidad, query o tabla en un modelo temporal.**