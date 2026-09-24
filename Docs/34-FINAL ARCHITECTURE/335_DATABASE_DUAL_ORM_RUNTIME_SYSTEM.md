# 335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md

## 1. Propósito

Este documento define la arquitectura oficial del **Database Dual ORM Runtime System** de `VoltStack/Quantum/Database`.

El componente se denomina conceptualmente:

```text
DatabaseDualOrmRuntimeSystem
```

y permite la coexistencia **temporal, controlada, observable y reversible** entre:

```text
Legacy Persistence Runtime
```

y:

```text
VoltStack/Quantum/Database
```

durante una migración progresiva.

El objetivo no es mantener permanentemente dos ORMs.

El objetivo es proporcionar una fase de transición segura en la que módulos, casos de uso o rutas puedan migrarse gradualmente sin requerir un cambio completo tipo:

```text
big bang migration
```

---

## 2. Dependencias documentales

Este documento continúa:

```text
325_DATABASE_MIGRATION_FROM_EXTERNAL_ORM_SYSTEM.md
326_DATABASE_ELOQUENT_MIGRATION_ADAPTER.md
327_DATABASE_DOCTRINE_MIGRATION_ADAPTER.md
328_DATABASE_LEGACY_DATABASE_MIGRATION_SYSTEM.md
329_DATABASE_MIGRATION_ANALYSIS_ENGINE.md
330_DATABASE_MIGRATION_INTERMEDIATE_MODEL.md
331_DATABASE_MIGRATION_RULE_ENGINE.md
332_DATABASE_MIGRATION_CODE_TRANSFORMER.md
333_DATABASE_MIGRATION_SCHEMA_COMPATIBILITY_SYSTEM.md
334_DATABASE_MIGRATION_BEHAVIOR_VERIFICATION_SYSTEM.md
```

y alimentará:

```text
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
337_DATABASE_MIGRATION_TESTING_AND_VALIDATION_SYSTEM.md
338_DATABASE_MIGRATION_CLI_AND_DEVELOPER_EXPERIENCE.md
339_DATABASE_MIGRATION_REPORTING_AND_DIAGNOSTICS.md
340_DATABASE_MIGRATION_ROLLBACK_AND_RECOVERY_SYSTEM.md
```

---

## 3. Principio fundamental

```text
Dual runtime is a migration mechanism,
not a permanent architecture.
```

---

## 4. Problema

Una aplicación grande puede contener:

```text
500 models
200 repositories
thousands of queries
background jobs
CLI commands
scheduled tasks
integrations
```

Migrar todo simultáneamente incrementa:

```text
blast radius
deployment risk
rollback complexity
debugging difficulty
downtime risk
```

---

## 5. Estrategia

VoltStack permitirá:

```text
Legacy Runtime
      +
VoltStack Runtime
```

durante una ventana de transición.

---

## 6. Objetivo

Permitir migraciones:

```text
module-by-module
repository-by-repository
use-case-by-use-case
route-by-route
read-path-first
```

manteniendo una única estrategia de control.

---

# Parte I — Definición

## 7. Dual ORM Runtime

Se define como:

```text
la coexistencia temporal de dos sistemas de persistencia
sobre una aplicación y, potencialmente, sobre el mismo
database físico.
```

---

## 8. Sistemas soportables

El concepto deberá permitir:

```text
Eloquent + VoltStack
Doctrine ORM + VoltStack
Doctrine DBAL + VoltStack
Legacy PDO + VoltStack
Custom ORM + VoltStack
```

---

## 9. No requisito de ORM

El término:

```text
Dual ORM
```

es conceptual.

Uno de los lados puede ser:

```text
raw SQL
DAO
Table Gateway
custom persistence layer
```

---

# Parte II — Arquitectura general

## 10. Arquitectura

```text
Application
    │
    ▼
Persistence Migration Gateway
    │
    ├──────────────► Legacy Runtime
    │
    └──────────────► VoltStack Runtime
                           │
                           ▼
                     Shared Database
```

---

## 11. Migration Gateway

El componente central será:

```text
MigrationPersistenceGateway
```

---

## 12. Responsabilidad

El Gateway decide:

```text
which runtime handles an operation
```

según un plan previamente aprobado.

---

## 13. No business decision

El Gateway no deberá decidir dinámicamente basándose en intuiciones o resultados de queries.

Utilizará:

```text
MigrationRoutingPolicy
```

---

# Parte III — Routing Policy

## 14. Migration Routing Policy

Define:

```text
module
entity
repository
operation
runtime
migration phase
fallback policy
shadow policy
```

---

## 15. Runtime Targets

```text
LEGACY
VOLTSTACK
SHADOW_LEGACY_PRIMARY
SHADOW_VOLTSTACK_PRIMARY
DUAL_READ_COMPARE
CONTROLLED_DUAL_WRITE
BLOCKED
```

---

## 16. Default

Para una unidad aún no migrada:

```text
LEGACY
```

---

## 17. Migrated

Después de validación:

```text
VOLTSTACK
```

---

# Parte IV — Migration States

## 18. State Machine

Cada unidad podrá recorrer:

```text
LEGACY
  ↓
PREPARED
  ↓
SHADOW_READ
  ↓
VOLTSTACK_READ
  ↓
VOLTSTACK_WRITE
  ↓
VOLTSTACK_PRIMARY
  ↓
LEGACY_DISABLED
  ↓
LEGACY_REMOVED
```

---

## 19. LEGACY

Todo continúa en el runtime origen.

---

## 20. PREPARED

El código VoltStack existe, pero no atiende tráfico normal.

---

## 21. SHADOW_READ

El legacy responde al usuario.

VoltStack ejecuta lectura secundaria para comparación.

---

## 22. VOLTSTACK_READ

Lecturas seleccionadas utilizan VoltStack.

Writes permanecen legacy.

---

## 23. VOLTSTACK_WRITE

Writes seleccionados utilizan VoltStack bajo gates estrictos.

---

## 24. VOLTSTACK_PRIMARY

VoltStack controla la unidad.

Legacy permanece temporalmente disponible para rollback compatible.

---

## 25. LEGACY_DISABLED

El runtime legacy ya no atiende esa unidad.

---

## 26. LEGACY_REMOVED

Código y dependencias legacy han sido retirados.

---

# Parte V — Migration Unit

## 27. Unidad

El routing no deberá operar únicamente por ORM completo.

Usará:

```text
MigrationRuntimeUnit
```

---

## 28. Ejemplos

```text
module: Billing
repository: UserRepository
entity: Invoice
operation: invoice.read
use case: CreateOrder
route: admin.users.index
```

---

## 29. Granularidad

La granularidad deberá ser explícita y estable.

---

## 30. Avoid Mixed Ownership

No deberá permitirse que una operación individual cambie aleatoriamente de runtime entre llamadas.

---

# Parte VI — Shared Database

## 31. Shared Physical Schema

La estrategia más común será:

```text
Legacy Runtime ─┐
                ├──► Same Physical Database
VoltStack ──────┘
```

---

## 32. Precondition

Antes de compartir tablas:

```text
333 Schema Compatibility
```

deberá haber validado el contrato físico.

---

## 33. Shared Schema Contract

Durante dual runtime:

```text
physical schema
```

actúa como contrato compartido.

---

## 34. Schema Ownership

Debe existir un único responsable de cambios de schema.

---

## 35. Rule

```text
Two runtimes may use the schema.

Two independent migration systems must not
change it without coordination.
```

---

# Parte VII — Schema Freeze

## 36. Schema Freeze Window

Durante ciertas fases podrá establecerse:

```text
MigrationSchemaFreeze
```

---

## 37. Purpose

Evitar:

```text
legacy migration changes table
while VoltStack assumes previous schema
```

---

## 38. Controlled Changes

Si el schema debe cambiar:

```text
plan
compatibility check
migration
re-introspection
behavior verification
```

---

# Parte VIII — Connection Architecture

## 39. Connections

Legacy y VoltStack podrán utilizar:

```text
same database credentials
different pools
different connection abstractions
```

---

## 40. Connection Independence

No deberá intentarse compartir directamente:

```text
PDO instance
Doctrine Connection object
Eloquent Connection object
```

entre runtimes salvo adapter explícito y probado.

---

## 41. Preferred

```text
independent runtime connections
to the same physical database
```

---

## 42. Consequence

Esto implica que:

```text
cross-runtime transaction atomicity
```

no aparece automáticamente.

---

# Parte IX — Transaction Boundary

## 43. Critical Rule

```text
One logical transaction should have one transaction owner.
```

---

## 44. Unsafe Example

```text
Eloquent begins transaction
VoltStack opens another connection
VoltStack writes
Eloquent rolls back
```

Resultado posible:

```text
VoltStack write committed
Legacy write rolled back
```

---

## 45. Transaction Owner

Se definirá:

```text
MigrationTransactionOwner
```

---

## 46. Values

```text
LEGACY
VOLTSTACK
EXTERNAL_COORDINATOR
NONE
```

---

## 47. V1 Policy

Para V1:

```text
avoid distributed cross-runtime write transactions
```

---

## 48. Cross-runtime Write

Si una operación necesita writes en ambos runtimes dentro de una misma unidad atómica:

```text
BLOCK
```

salvo estrategia explícita certificada.

---

# Parte X — Read Migration

## 49. Read-first

La estrategia preferida será:

```text
migrate reads before writes
```

cuando el dominio lo permita.

---

## 50. Benefits

Las lecturas:

```text
do not mutate state
are easier to shadow
are easier to compare
are easier to rollback
```

---

## 51. Flow

```text
LEGACY READ
   ↓
SHADOW VOLTSTACK READ
   ↓
COMPARE
   ↓
VOLTSTACK READ
```

---

# Parte XI — Shadow Reads

## 52. Shadow Read

Durante:

```text
SHADOW_READ
```

se ejecutan:

```text
Primary read
+
Secondary read
```

---

## 53. Response

Solo el primary produce la respuesta de aplicación.

---

## 54. Secondary Failure

Un fallo shadow:

```text
must not fail user request
```

por defecto.

Se registra para diagnóstico.

---

## 55. Detailed System

La arquitectura especializada se define en:

```text
336_DATABASE_SHADOW_QUERY_AND_RESULT_COMPARISON_SYSTEM.md
```

---

# Parte XII — Write Migration

## 56. Writes

La migración de writes requiere más precaución.

---

## 57. Preferred Pattern

```text
single writer
```

por unidad.

---

## 58. Legacy Writer Phase

```text
Reads:
Legacy / Shadow VoltStack

Writes:
Legacy only
```

---

## 59. VoltStack Writer Phase

Después de verificación:

```text
Reads:
VoltStack

Writes:
VoltStack only
```

---

## 60. Dual Write

```text
Legacy write
+
VoltStack write
```

sobre el mismo dato no será el modo predeterminado.

---

# Parte XIII — Controlled Dual Write

## 61. Controlled Dual Write

Existirá únicamente para casos explícitos:

```text
CONTROLLED_DUAL_WRITE
```

---

## 62. Preconditions

Requerirá:

```text
idempotent operation
known transaction semantics
known side effects
isolated test evidence
conflict strategy
rollback strategy
explicit approval
```

---

## 63. Same-table Dual Write

Si ambos runtimes escriben la misma fila:

```text
normally prohibited
```

porque puede duplicar:

```text
events
timestamps
version increments
triggers
audit records
```

---

## 64. Safer Uses

Dual write puede ser más apropiado cuando:

```text
writing separate migration store
writing audit comparison data
writing disposable shadow DB
```

---

# Parte XIV — Write Ownership

## 65. Ownership Registry

Existirá:

```text
MigrationWriteOwnershipRegistry
```

---

## 66. Example

```text
users:
    owner = LEGACY

invoices:
    owner = VOLTSTACK
```

---

## 67. Granular Ownership

También podrá ser:

```text
entity
aggregate
module
operation
```

---

## 68. Conflict

Dos owners activos para el mismo write scope:

```text
WRITE_OWNERSHIP_CONFLICT
```

---

# Parte XV — Identity

## 69. Identity Problem

Eloquent, Doctrine y VoltStack pueden representar la misma fila con objetos distintos.

---

## 70. Rule

```text
Do not assume cross-runtime object identity.
```

---

## 71. Boundary Data

Entre runtimes deberán preferirse:

```text
IDs
scalar values
DTOs
immutable messages
```

---

## 72. No Entity Passing

Evitar:

```text
Doctrine Entity
→ VoltStack Repository
```

o:

```text
Eloquent Model
→ VoltStack UnitOfWork
```

---

# Parte XVI — Unit of Work

## 73. Separate UoW

Cada ORM mantendrá su propio:

```text
Unit of Work
Identity Map
dirty tracking
```

---

## 74. Stale Object Risk

Si:

```text
VoltStack updates row
Doctrine entity already loaded
```

Doctrine puede conservar estado obsoleto.

---

## 75. Rule

No deberán mezclarse dos persistence contexts sobre el mismo aggregate dentro de una request sin estrategia explícita.

---

## 76. Refresh

Cuando sea inevitable podrá requerirse:

```text
refresh
clear
reload
```

según runtime.

---

# Parte XVII — Cache Isolation

## 77. Cache Risk

Cada ORM puede mantener:

```text
query cache
entity cache
metadata cache
application cache
```

---

## 78. Shared Data, Separate Cache

Un write en VoltStack puede dejar cache legacy obsoleta.

---

## 79. Cache Policy

Durante dual runtime se requerirá:

```text
shared invalidation
cache disablement
short TTL
or isolated ownership
```

según caso.

---

## 80. Second-level Cache

Doctrine second-level cache deberá tratarse explícitamente.

---

# Parte XVIII — Events

## 81. Event Duplication

Si ambos runtimes procesan el mismo write pueden disparar:

```text
two audit events
two emails
two jobs
two domain events
```

---

## 82. Rule

```text
One business write must have one side-effect authority.
```

---

## 83. Event Ownership

Se podrá definir:

```text
MigrationSideEffectOwnership
```

---

## 84. Values

```text
LEGACY
VOLTSTACK
EXTERNAL
SUPPRESSED
```

---

# Parte XIX — Lifecycle

## 85. Lifecycle Hooks

Durante coexistencia deberán inventariarse:

```text
Eloquent observers
Doctrine listeners/subscribers
VoltStack events
DB triggers
```

---

## 86. Duplicate Behavior

Si dos mecanismos realizan la misma acción:

```text
DUPLICATE_LIFECYCLE_BEHAVIOR
```

---

# Parte XX — Database Triggers

## 87. Triggers

Un trigger compartido se ejecutará independientemente del runtime que escriba.

---

## 88. Consequence

La migración no deberá duplicar en VoltStack un comportamiento que ya pertenece al DB.

---

# Parte XXI — Soft Delete

## 89. Shared Soft Delete

Ambos runtimes deberán acordar:

```text
column
null semantics
visibility
restore behavior
```

---

## 90. Filter Difference

Si Legacy oculta soft-deleted rows y VoltStack no:

```text
DUAL_RUNTIME_INCOMPATIBLE
```

para ese read path.

---

# Parte XXII — Timestamps

## 91. Timestamp Ownership

Debe saberse quién genera:

```text
created_at
updated_at
```

---

## 92. Preferred

Durante dual runtime, mantener una única semántica.

---

## 93. DB-generated

Database-generated timestamps pueden simplificar coexistencia si ya son el contrato actual, pero no deberán introducirse únicamente por conveniencia sin plan.

---

# Parte XXIII — Optimistic Locking

## 94. Version Column

Ambos runtimes deberán respetar la misma:

```text
version column
```

si existe.

---

## 95. Unsafe Runtime

Un runtime que ignore optimistic locking no podrá ser writer de ese aggregate.

---

# Parte XXIV — Pessimistic Locking

## 96. Locks

Si ambos runtimes usan la misma DB, locks físicos pueden interoperar.

Pero deberán verificar:

```text
same rows
same transaction duration
same isolation
same connection semantics
```

---

# Parte XXV — Error Translation

## 97. Error Boundary

La aplicación no deberá recibir errores arbitrariamente distintos solo por cambiar runtime.

---

## 98. Normalization

El Migration Gateway podrá apoyarse en contratos comunes de error.

---

# Parte XXVI — Repository Boundary

## 99. Preferred Seam

La frontera ideal para migración es:

```text
Repository Interface
```

---

## 100. Example

```php
interface UserRepository
{
    public function find(UserId $id): ?User;
}
```

Implementaciones:

```text
LegacyUserRepository
VoltStackUserRepository
```

---

## 101. Routing

El Gateway selecciona implementación.

---

## 102. Benefit

Los consumidores no conocen el runtime.

---

# Parte XXVII — Query Service Boundary

## 103. Read-heavy Systems

Para reporting puede utilizarse:

```text
QueryService Interface
```

como seam.

---

## 104. Example

```text
SalesReportQuery
├── LegacySalesReportQuery
└── VoltStackSalesReportQuery
```

---

# Parte XXVIII — Direct ORM Usage

## 105. Problem

Código como:

```php
User::where(...)->get();
```

está fuertemente acoplado al runtime.

---

## 106. Migration

El Code Transformer podrá introducir temporalmente:

```text
repository/query boundary
```

antes de activar dual runtime.

---

# Parte XXIX — Runtime Router

## 107. Component

```text
MigrationPersistenceRuntimeRouter
```

---

## 108. Contract

Conceptualmente:

```php
interface MigrationPersistenceRuntimeRouterInterface
{
    public function resolve(
        MigrationRuntimeOperation $operation
    ): MigrationRuntimeTarget;
}
```

---

## 109. Determinism

Misma:

```text
operation
migration state
configuration
```

deberá resolver el mismo target.

---

# Parte XXX — Feature Flags

## 110. Feature Flags

La selección podrá integrarse con feature flags.

---

## 111. Allowed Uses

```text
enable VoltStack reads
enable shadow reads
switch module primary
emergency fallback
```

---

## 112. Not Random Experimentation

No deberá utilizarse A/B aleatorio para writes sobre datos persistentes.

---

## 113. Percentage Rollout

Para reads stateless puede existir rollout gradual si el contrato lo permite.

---

## 114. Stable Cohort

Cuando sea necesario, el rollout deberá ser estable por:

```text
tenant
account
user
request key
```

para evitar comportamiento inconsistente.

---

# Parte XXXI — Emergency Fallback

## 115. Fallback

Durante una ventana controlada podrá existir:

```text
VoltStack Primary
→ Legacy Fallback
```

---

## 116. Read Fallback

Es relativamente seguro si ambos runtimes son read-compatible.

---

## 117. Write Fallback

Es mucho más complejo.

---

## 118. Write Fallback Rule

No deberá ocurrir automáticamente después de un error ambiguo.

Ejemplo:

```text
VoltStack write timed out
```

no permite saber inmediatamente si:

```text
write committed
or
write did not commit
```

Reintentar legacy podría duplicar efectos.

---

## 119. Ambiguous Commit

Estado:

```text
WRITE_OUTCOME_UNKNOWN
```

deberá bloquear fallback automático.

---

# Parte XXXII — Idempotency

## 120. Idempotency Keys

Operaciones críticas podrán utilizar:

```text
idempotency key
```

si el dominio ya lo soporta o el plan lo introduce explícitamente.

---

## 121. Purpose

Reducir riesgo durante:

```text
retry
fallback
network ambiguity
```

---

# Parte XXXIII — Session Consistency

## 122. Read-after-write

Si un write se realiza con un runtime y el read inmediato con otro, deberá verificarse:

```text
visibility
replica lag
cache
transaction state
```

---

## 123. Same Request

Dentro de una request, el router podrá mantener:

```text
runtime affinity
```

para ciertos aggregates.

---

# Parte XXXIV — Runtime Affinity

## 124. Affinity

```text
MigrationRuntimeAffinity
```

podrá garantizar que operaciones relacionadas usen el mismo runtime.

---

## 125. Scope

```text
request
transaction
aggregate
use case
```

---

## 126. Persistent Worker Safety

Affinity nunca deberá almacenarse en static global sin reset.

---

# Parte XXXV — FrankenPHP

## 127. Default Runtime

FrankenPHP es el runtime de aplicación predeterminado de VoltStack.

---

## 128. Dual Runtime Risks

Se deberá evitar fuga de:

```text
selected runtime
current tenant
legacy EntityManager
VoltStack PersistenceContext
transaction owner
shadow state
feature flag result
```

entre requests.

---

## 129. Request Context

Toda decisión request-local deberá almacenarse en:

```text
request-scoped migration context
```

---

## 130. Reset

Al finalizar request:

```text
clear legacy UoW where required
clear VoltStack persistence context
rollback unfinished transactions
reset tenant
reset filters
clear migration routing context
release temporary resources
```

---

## 131. Worker Test

Debe existir prueba:

```text
Request A → Legacy
Request B → VoltStack
Request C → Legacy
```

sin contaminación.

---

## 132. Long-running Worker

Se medirán:

```text
memory growth
identity map growth
connection leakage
listener duplication
```

---

# Parte XXXVI — RoadRunner/OpenSwoole

## 133. Runtime Profiles

Los mismos contratos se aplicarán mediante:

```text
MigrationWorkerRuntimeProfile
```

---

## 134. No FrankenPHP Coupling

El core Dual ORM no dependerá directamente de FrankenPHP.

---

# Parte XXXVII — CLI

## 135. Status

Conceptualmente:

```bash
php volt database:migrate:runtime --status
```

---

## 136. Output

```text
Unit                 Read      Write      State
Users                Legacy    Legacy     LEGACY
Catalog              VoltStack Legacy     VOLTSTACK_READ
Billing              VoltStack VoltStack  VOLTSTACK_PRIMARY
Reports              Shadow    Legacy     SHADOW_READ
```

---

## 137. Change State

```bash
php volt database:migrate:runtime \
    --unit=Catalog \
    --state=VOLTSTACK_READ
```

---

## 138. Safety Gate

La transición deberá comprobar:

```text
schema verification
behavior verification
required tests
rollback readiness
```

antes de aceptar el cambio.

---

## 139. Force

Un posible:

```text
--force
```

no deberá saltarse invariantes críticas de integridad.

---

# Parte XXXVIII — Runtime Manifest

## 140. Manifest

Estado persistente:

```text
MigrationRuntimeManifest
```

---

## 141. Contents

```text
unit
state
read owner
write owner
transaction owner
side-effect owner
shadow mode
fallback
verification references
updated version
```

---

## 142. Versioning

El manifest deberá ser:

```text
versioned
auditable
```

---

## 143. Deployment

Idealmente formará parte de configuración desplegable/versionada.

---

# Parte XXXIX — State Transition Guard

## 144. Component

```text
MigrationRuntimeStateTransitionGuard
```

---

## 145. Example

Para:

```text
SHADOW_READ
→ VOLTSTACK_READ
```

puede exigir:

```text
shadow comparison threshold met
no critical differences
schema compatible
behavior contracts passed
```

---

## 146. Write Gate

Para:

```text
VOLTSTACK_READ
→ VOLTSTACK_WRITE
```

requerirá más evidencia.

---

# Parte XL — State Rollback

## 147. Runtime Rollback

Un estado podrá volver:

```text
VOLTSTACK_READ
→ LEGACY
```

si todavía existe compatibilidad.

---

## 148. Rollback Window

La capacidad de rollback puede expirar después de:

```text
schema contraction
legacy dependency removal
behavioral divergence
data format change
```

---

## 149. Explicit Window

Se registrará:

```text
rollback_available
rollback_conditions
rollback_expiry_phase
```

---

# Parte XLI — Schema Contract During Rollback

## 150. Compatibility

Mientras se prometa fallback legacy, el schema deberá seguir siendo legible por legacy.

---

## 151. Expand/Contract

Por ello:

```text
expand
→ dual compatible phase
→ cutover
→ contract
```

es una estrategia preferible para cambios incompatibles.

---

# Parte XLII — Data Format Compatibility

## 152. Stored Representation

No basta que ambos runtimes compartan columnas.

Deben comprender:

```text
enum representation
serialized data
encrypted envelopes
UUID encoding
JSON shape
discriminators
```

---

## 153. One-way Transformation

Si VoltStack empieza a escribir un formato que legacy no comprende:

```text
legacy rollback may become impossible
```

---

# Parte XLIII — Migration Barrier

## 154. Barrier

Antes de una transformación irreversible podrá existir:

```text
MigrationRuntimeBarrier
```

---

## 155. Purpose

Declarar:

```text
legacy fallback ends here
```

antes de ejecutar el cambio.

---

# Parte XLIV — Background Jobs

## 156. Jobs

Los workers de jobs pueden ejecutar código de versiones diferentes durante deployment.

---

## 157. Risk

Un job encolado antes del cutover puede consumir:

```text
legacy serialized entity
```

después de retirar el ORM.

---

## 158. Queue Compatibility Window

Se deberá considerar:

```text
oldest outstanding job
retry window
dead-letter queue
scheduled jobs
```

---

## 159. Safe Payloads

Preferir:

```text
IDs
DTOs
versioned messages
```

---

# Parte XLV — Scheduled Tasks and CLI

## 160. Non-HTTP Workloads

El routing deberá funcionar también para:

```text
cron
scheduler
CLI
queue worker
batch
```

---

## 161. Runtime Context

No deberá asumir existencia de HTTP request.

---

# Parte XLVI — Multiple Connections

## 162. Multiple DBs

Una aplicación puede migrar:

```text
connection A first
connection B later
```

---

## 163. Runtime Unit

El manifest deberá incluir conexión cuando sea relevante.

---

# Parte XLVII — Read Replicas

## 164. Replicas

El routing deberá distinguir:

```text
ORM migration
```

de:

```text
primary/replica routing
```

---

## 165. Comparison

Shadow queries sobre replicas deberán considerar:

```text
replication lag
```

para evitar falsos positivos.

---

# Parte XLVIII — Multitenancy

## 166. Optional Integration

El paquete Multitenancy podrá definir rollout:

```text
tenant-by-tenant
```

---

## 167. Example

```text
Tenant A → VoltStack
Tenant B → Legacy
Tenant C → Shadow
```

---

## 168. Isolation

La decisión deberá ser request-local y no filtrarse entre tenants.

---

## 169. Schema-per-tenant

Cada tenant deberá cumplir schema compatibility antes del cutover si sus schemas pueden divergir.

---

# Parte XLIX — SaaS

## 170. Separation

El paquete SaaS permanecerá independiente.

Podrá consumir hooks del Dual Runtime si está instalado.

---

# Parte L — Security

## 171. Security Principle

La coexistencia aumenta la superficie de ataque.

---

## 172. Legacy Dependencies

Mantener temporalmente un ORM antiguo puede conservar vulnerabilidades.

El periodo dual deberá minimizarse.

---

## 173. Credentials

No deberán duplicarse secretos innecesariamente.

---

## 174. Authorization

El runtime router no deberá ser controlable directamente por parámetros de usuario.

---

## 175. Internal Control

La selección provendrá de:

```text
trusted configuration
migration manifest
server-side feature flags
approved deployment state
```

---

## 176. Debug Headers

No se deberá permitir cambiar runtime mediante headers públicos en producción.

---

# Parte LI — Observability

## 177. Metrics

```text
database.migration.runtime.legacy_requests
database.migration.runtime.voltstack_requests
database.migration.runtime.shadow_reads
database.migration.runtime.fallbacks
database.migration.runtime.write_owner_conflicts
database.migration.runtime.ambiguous_writes
database.migration.runtime.state_transitions
database.migration.runtime.context_leaks
```

---

## 178. Labels

Labels deberán evitar cardinalidad excesiva.

---

## 179. Tracing

Spans podrán incluir:

```text
migration.runtime
migration.unit
migration.state
shadow.enabled
```

sin información sensible.

---

## 180. Logging

Canal:

```text
database.migration.runtime
```

---

# Parte LII — Health

## 181. Health Check

Podrá comprobar:

```text
legacy runtime available
VoltStack runtime available
schema fingerprint expected
manifest valid
connections healthy
```

---

## 182. Degraded State

Si shadow runtime falla pero primary funciona:

```text
DEGRADED_MIGRATION_OBSERVABILITY
```

sin necesariamente interrumpir tráfico.

---

# Parte LIII — Circuit Breaker

## 183. Shadow Circuit Breaker

Errores masivos en shadow podrán suspender temporalmente shadow execution para proteger capacidad.

---

## 184. Primary Circuit Breaker

No deberá cambiar automáticamente de write runtime ante fallos ambiguos.

---

# Parte LIV — Performance

## 185. Dual Cost

Shadow execution puede aproximadamente incrementar:

```text
query load
connection usage
CPU
memory
latency if synchronous
```

---

## 186. Sampling

El sistema podrá ejecutar shadow solo sobre:

```text
sampled reads
```

---

## 187. Async Comparison

Cuando sea seguro, la comparación podrá desacoplarse del response path.

---

## 188. No Unbounded Queue

Los resultados shadow deberán tener:

```text
backpressure
sampling
retention policy
```

---

# Parte LV — Capacity Guard

## 189. Capacity

Antes de habilitar shadow masivo deberá evaluarse:

```text
DB capacity
connection pool
worker memory
CPU
```

---

## 190. Auto Protection

Podrá reducir shadow sampling si supera límites operacionales, sin cambiar el primary runtime.

---

# Parte LVI — Dependency Removal

## 191. Legacy Removal

No se eliminará el paquete legacy hasta:

```text
0 active legacy runtime units
0 fallback dependencies
0 legacy queue payload requirements
0 source-only migration tools required at runtime
0 active compatibility adapters requiring it
```

---

## 192. Migration-only Package

Los adapters de análisis pueden permanecer en `require-dev` si aún son necesarios para auditoría.

---

# Parte LVII — Architecture Tests

## 193. Tests

Podrán impedir nuevo código:

```text
new Eloquent model usage
new EntityManager injection
new direct PDO creation
```

en módulos ya migrados.

---

## 194. Regression Gate

Una unidad:

```text
LEGACY_REMOVED
```

no deberá volver a introducir dependencias legacy sin decisión arquitectónica explícita.

---

# Parte LVIII — Testing

## 195. Unit Tests

Para:

```text
routing policy
state machine
ownership
transition guards
manifest
fallback decisions
```

---

## 196. Integration Tests

Para:

```text
Legacy → DB
VoltStack → DB
shared schema
read-after-write
cache invalidation
```

---

## 197. Transaction Tests

Especialmente:

```text
rollback
ambiguous write
nested transaction
cross-runtime boundary
```

---

## 198. Worker Tests

```text
FrankenPHP
RoadRunner
OpenSwoole
```

según paquetes instalados.

---

## 199. Failure Injection

Simular:

```text
legacy unavailable
VoltStack unavailable
DB timeout
shadow timeout
partial deployment
stale manifest
```

---

## 200. Load Tests

Evaluar costo de:

```text
shadow reads
two connection pools
dual metadata
runtime routing
```

---

# Parte LIX — Error Model

## 201. Exceptions

Conceptualmente:

```text
MigrationDualRuntimeException
MigrationRuntimeRoutingException
MigrationRuntimeStateException
MigrationRuntimeOwnershipException
MigrationRuntimeTransactionConflictException
MigrationRuntimeFallbackException
MigrationRuntimeManifestException
MigrationRuntimeCompatibilityException
```

---

# Parte LX — Componentes principales

## 202. Architecture

```text
DatabaseDualOrmRuntimeSystem
│
├── MigrationPersistenceGateway
├── MigrationPersistenceRuntimeRouter
├── MigrationRoutingPolicy
├── MigrationRuntimeManifest
├── MigrationRuntimeUnitRegistry
├── MigrationRuntimeStateMachine
├── MigrationRuntimeStateTransitionGuard
├── MigrationWriteOwnershipRegistry
├── MigrationTransactionOwnershipResolver
├── MigrationSideEffectOwnershipResolver
├── MigrationRuntimeAffinityManager
├── MigrationLegacyRuntimeAdapter
├── MigrationVoltStackRuntimeAdapter
├── MigrationShadowExecutionCoordinator
├── MigrationFallbackCoordinator
├── MigrationRuntimeContext
├── MigrationRuntimeResetManager
├── MigrationRuntimeHealthMonitor
├── MigrationRuntimeTelemetry
└── MigrationRuntimeReporter
```

---

# Parte LXI — Pipeline

## 203. Request/Operation Pipeline

```text
Application Operation
        │
        ▼
Identify Migration Unit
        │
        ▼
Load Runtime Manifest
        │
        ▼
Resolve Runtime State
        │
        ▼
Check Ownership / Affinity
        │
        ▼
Select Primary Runtime
        │
        ├───────────────► Optional Shadow Runtime
        │
        ▼
Execute Primary
        │
        ▼
Capture Telemetry
        │
        ▼
Return Result
        │
        ▼
Reset Request Runtime State
```

---

# Parte LXII — State Transition Pipeline

## 204. Transition

```text
Requested State Change
        │
        ▼
Load Verification Evidence
        │
        ├── Schema Compatibility
        ├── Behavior Verification
        ├── Test Results
        └── Rollback Readiness
        │
        ▼
Transition Guard
        │
    ┌───┴────┐
    ▼        ▼
 APPROVE    BLOCK
    │
    ▼
Update Versioned Manifest
    │
    ▼
Deploy / Activate
    │
    ▼
Observe
```

---

# Parte LXIII — Ejemplo Eloquent → VoltStack

## 205. Initial

```text
Users:
read  = Eloquent
write = Eloquent
state = LEGACY
```

---

## 206. Shadow

```text
Users:
primary read = Eloquent
shadow read  = VoltStack
write        = Eloquent
state        = SHADOW_READ
```

---

## 207. Read Cutover

```text
Users:
read  = VoltStack
write = Eloquent
state = VOLTSTACK_READ
```

---

## 208. Write Cutover

Después de transaction/behavior verification:

```text
Users:
read  = VoltStack
write = VoltStack
state = VOLTSTACK_PRIMARY
```

---

## 209. Removal

Finalmente:

```text
Users:
runtime = VoltStack
legacy dependency = removed
state = LEGACY_REMOVED
```

---

# Parte LXIV — Ejemplo Doctrine

## 210. Doctrine Risk

Una Entity puede permanecer en Identity Map mientras VoltStack actualiza la misma fila.

---

## 211. Strategy

Evitar acceso mixto al mismo aggregate dentro de la misma unidad de trabajo.

Si ocurre transición:

```text
flush/complete legacy work
clear appropriate persistence context
switch boundary
reload through target
```

según plan.

---

# Parte LXV — Ejemplo Raw PDO

## 212. Legacy

```text
ReportService
→ PDO
```

---

## 213. Dual

```text
ReportQueryInterface
├── LegacyPdoReportQuery
└── VoltStackReportQuery
```

---

## 214. Shadow

Ambos pueden ejecutar SELECT y comparar resultados sin cambiar writes.

---

# Parte LXVI — Ejemplo Unsafe Write Fallback

## 215. Operation

```text
Charge invoice
```

VoltStack envía INSERT.

DB responde timeout.

---

## 216. Incorrect

```text
catch timeout
→ run legacy INSERT
```

Puede crear duplicado.

---

## 217. Correct

Clasificar:

```text
WRITE_OUTCOME_UNKNOWN
```

y aplicar estrategia de reconciliación/idempotencia.

---

# Parte LXVII — Deployment

## 218. Rolling Deployment

Durante rolling deploy pueden coexistir:

```text
old application workers
new application workers
```

---

## 219. Manifest Compatibility

El runtime manifest deberá ser compatible con la ventana de despliegue.

---

## 220. State Transition Timing

No deberá activarse un estado que workers antiguos no comprendan.

---

## 221. Deployment Barrier

Podrá requerirse:

```text
all compatible binaries deployed
→ activate new migration state
```

---

# Parte LXVIII — Configuration

## 222. Example

```php
return [
    'migration_runtime' => [
        'enabled' => true,

        'default' => 'legacy',

        'units' => [
            'users' => [
                'state' => 'shadow_read',
                'read' => 'legacy',
                'write' => 'legacy',
                'shadow' => 'voltstack',
            ],
        ],
    ],
];
```

---

## 223. Production Source

En producción, el estado podrá provenir de configuración desplegable o store de control confiable.

---

## 224. Cache

La configuración podrá cachearse, pero deberá invalidarse de forma segura al cambiar estado.

---

# Parte LXIX — V1 Scope Boundary

## 225. Included in V1

```text
migration runtime routing
state machine
single-writer ownership
read shadowing integration
controlled fallback
runtime manifest
transition gates
persistent-worker isolation
telemetry
rollback-aware states
```

---

## 226. Not a V1 Goal

No se pretende construir aquí:

```text
distributed database replication platform
generic distributed transaction coordinator
cross-cloud database fabric
automatic multi-master conflict resolution
autonomous migration AI
```

Estos temas pertenecen a evoluciones posteriores.

---

# Parte LXX — Decisiones arquitectónicas

## 227. Decisión 1

Dual ORM será una capacidad temporal de migración.

## 228. Decisión 2

La migración preferirá unidades pequeñas y explícitas.

## 229. Decisión 3

El schema físico será un contrato compartido durante coexistencia.

## 230. Decisión 4

Habrá un único owner de cambios de schema.

## 231. Decisión 5

Cada write scope tendrá un único write owner por defecto.

## 232. Decisión 6

Se evitarán transacciones distribuidas entre runtimes en V1.

## 233. Decisión 7

Las lecturas se migrarán antes que writes cuando sea viable.

## 234. Decisión 8

Shadow reads serán de primera clase.

## 235. Decisión 9

Dual writes estarán deshabilitados por defecto.

## 236. Decisión 10

No se compartirán entidades ORM entre runtimes.

## 237. Decisión 11

Cada runtime conservará su propio UoW/Identity Map.

## 238. Decisión 12

El Gateway seleccionará runtime mediante policy determinista.

## 239. Decisión 13

Fallback de write no ocurrirá automáticamente ante resultados ambiguos.

## 240. Decisión 14

El state machine estará protegido por evidencia de schema, behavior y testing.

## 241. Decisión 15

La capacidad de rollback será explícita y podrá expirar.

## 242. Decisión 16

FrankenPHP requerirá aislamiento/reset estricto del contexto dual entre requests.

## 243. Decisión 17

El routing también funcionará para CLI, jobs, scheduler y batch.

## 244. Decisión 18

El legacy runtime se eliminará una vez que no exista dependencia operacional real.

---

# Parte LXXI — Criterios de transición

## 245. LEGACY → PREPARED

Requiere:

```text
target implementation exists
source analysis complete
MIM available
rules approved
```

---

## 246. PREPARED → SHADOW_READ

Requiere:

```text
schema compatible
target read executable
shadow safe
telemetry enabled
```

---

## 247. SHADOW_READ → VOLTSTACK_READ

Requiere:

```text
required comparisons pass
no unresolved critical differences
behavior contracts pass
operational capacity acceptable
```

---

## 248. VOLTSTACK_READ → VOLTSTACK_WRITE

Requiere:

```text
write behavior verified
transaction semantics verified
side effects verified
rollback strategy ready
```

---

## 249. VOLTSTACK_WRITE → VOLTSTACK_PRIMARY

Requiere:

```text
target stable
legacy fallback conditions known
critical tests pass
```

---

## 250. VOLTSTACK_PRIMARY → LEGACY_DISABLED

Requiere:

```text
rollback policy allows
no required legacy traffic
queue compatibility resolved
```

---

## 251. LEGACY_DISABLED → LEGACY_REMOVED

Requiere:

```text
0 runtime references
0 fallback requirement
0 serialized legacy payload dependency
0 active compatibility adapter requiring runtime
architecture tests pass
```

---

# Parte LXXII — Completion Criteria

## 252. Dual Runtime Completion

El Dual Runtime System habrá cumplido su propósito cuando:

```text
all migration units = LEGACY_REMOVED
```

o cuando las unidades excluidas estén documentadas explícitamente.

---

## 253. Success Is Removal

La métrica final no será:

```text
how long both ORMs coexist
```

sino:

```text
how safely the legacy runtime can be removed
```

---

# Parte LXXIII — Resultado esperado

## 254. Antes

```text
Legacy ORM
    │
    ▼
Entire Application
```

Migración riesgosa:

```text
switch everything at once
```

---

## 255. Durante

```text
Application
    │
    ▼
Migration Persistence Gateway
    │
    ├── Legacy units
    ├── Shadow units
    ├── VoltStack read units
    └── VoltStack primary units
```

---

## 256. Después

```text
Application
    │
    ▼
VoltStack/Quantum/Database
```

sin runtime legacy.

---

# Parte LXXIV — Principio final

## 257. Regla

```text
Coexist only to migrate.

Route explicitly.

Write from one authority.

Compare before cutover.

Keep rollback possible until the migration proves it no longer needs it.

Then remove the legacy runtime.
```

---

# Parte LXXV — Conclusión

## 258. Arquitectura final

`DATABASE_DUAL_ORM_RUNTIME_SYSTEM` convierte una migración de persistencia de alto riesgo en una transición progresiva gobernada por estados.

La arquitectura completa queda:

```text
Legacy Persistence
       │
       │
       ├───────────────┐
       │               │
       ▼               ▼
Migration Persistence Gateway
       │
       ├── Runtime Routing Policy
       ├── Write Ownership
       ├── Transaction Ownership
       ├── Side-effect Ownership
       ├── Runtime Affinity
       ├── Shadow Coordination
       ├── Fallback Control
       └── Transition Guards
       │
       ├───────────────┬───────────────┐
       ▼               ▼               ▼
    Legacy          Shadow          VoltStack
    Runtime         Runtime         Runtime
       │               │               │
       └───────────────┴───────────────┘
                       │
                       ▼
               Compatible Schema
```

Su integración con los documentos anteriores establece:

```text
333 proves the shared schema can support both sides.

334 proves required behavior is preserved.

335 controls which side is active.

336 will compare shadow reads at scale.

337 will coordinate complete validation.

340 will define recovery when a transition must be reversed.
```

La regla arquitectónica definitiva será:

```text
Dual runtime is a bridge.

It must have an entrance,
a controlled crossing,
a rollback path,
and an exit.
```

---

**Documento:** `335_DATABASE_DUAL_ORM_RUNTIME_SYSTEM.md`  
**Proyecto:** VoltStack Framework  
**Módulo:** `VoltStack/Quantum/Database`  
**Versión objetivo:** Database V1  
**MIM objetivo:** `1.x`  
**Estado:** Architectural Specification
