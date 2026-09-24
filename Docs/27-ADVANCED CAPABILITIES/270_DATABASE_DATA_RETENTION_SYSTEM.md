# 270_DATABASE_DATA_RETENTION_SYSTEM.md

# VoltStack Quantum Database
## Data Retention System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 270 — Data Retention System  
**Bloque:** 27 — Advanced Database Capabilities  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `269_DATABASE_SOFT_DELETE_SYSTEM.md`  
**Siguiente documento:** `271_DATABASE_DATA_ARCHIVAL_SYSTEM.md`

---

# 1. Propósito

Este documento define la arquitectura del **Data Retention System** de VoltStack Database.

El sistema será responsable de representar, evaluar y ejecutar políticas que determinen:

```text
qué información debe conservarse
durante cuánto tiempo
desde qué evento comienza a contarse
cuándo se vuelve elegible para eliminación
cuándo debe conservarse obligatoriamente
cuándo debe archivarse antes de eliminarse
qué condiciones bloquean el purge
qué evidencia debe conservarse de la operación
```

La regla central será:

> **Data Retention gobierna el ciclo temporal de conservación y eliminación de datos; no es equivalente a Soft Delete, Archive, Backup, TTL, History ni a un cron que ejecuta DELETE periódicamente.**

---

# 2. Objetivos

El sistema deberá proporcionar:

- políticas declarativas de retención;
- ventanas mínimas y máximas;
- anchors temporales explícitos;
- cálculo determinista de elegibilidad;
- soporte para Soft Delete;
- soporte para History/Versioning;
- Legal Hold;
- políticas por clasificación de datos;
- políticas por tenant;
- purga segura;
- procesamiento incremental;
- ejecución distribuida;
- auditoría;
- telemetría;
- simulación;
- dry-run;
- explainability;
- integración con archival;
- integración con seguridad;
- integración con transacciones;
- seguridad para FrankenPHP, RoadRunner y OpenSwoole.

---

# 3. Distinciones fundamentales

VoltStack distinguirá:

```text
Retention
≠ Soft Delete
≠ Hard Delete
≠ Archive
≠ Backup
≠ History
≠ Temporal Validity
≠ Cache TTL
≠ Expiration
≠ Garbage Collection
```

---

# 4. Retention ≠ Soft Delete

Soft Delete responde:

```text
¿Esta entidad debe permanecer visible como activa?
```

Retention responde:

```text
¿Cuánto tiempo debe conservarse esta información?
```

Una entidad puede estar:

```text
SOFT_DELETED
```

pero todavía encontrarse dentro de su periodo obligatorio de retención.

---

# 5. Retention ≠ Hard Delete

Retention puede determinar:

```text
PURGE_ELIGIBLE
```

pero eso no significa que la eliminación física ya haya ocurrido.

Por tanto:

```text
Purge Eligible
≠
Purged
```

---

# 6. Retention ≠ Archive

Retention puede exigir:

```text
archive before purge
```

pero el Archive System será responsable de realizar y verificar el archivado.

---

# 7. Retention ≠ Backup

Un backup representa recuperación operacional.

No necesariamente representa conservación regulatoria.

```text
Backup Retention
≠
Application Data Retention
```

---

# 8. Retention ≠ History

History responde:

```text
¿Cómo cambió el dato?
```

Retention responde:

```text
¿Cuánto tiempo deben conservarse esas versiones?
```

Las versiones históricas pueden tener políticas de retención distintas a la entidad actual.

---

# 9. Retention ≠ Temporal Validity

Un contrato puede tener:

```text
valid_to = 2026-12-31
```

sin ser elegible para purga el 1 de enero de 2027.

Puede existir una obligación adicional:

```text
retain for 5 years after contract termination
```

---

# 10. Retention ≠ Cache TTL

```text
cache TTL = 10 minutes
```

sólo determina cuánto puede permanecer una representación en cache.

No determina cuánto debe existir el dato original.

---

# 11. Modelo conceptual

```text
                    DATA CREATED
                         │
                         ▼
                      ACTIVE
                         │
                         │ lifecycle event
                         ▼
                RETENTION CLOCK START
                         │
                         ▼
                RETENTION REQUIRED
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         LEGAL HOLD             TIME PASSES
              │                     │
              │                     ▼
              │               PERIOD EXPIRES
              │                     │
              │                     ▼
              │               PURGE ELIGIBLE
              │                     │
              │             ┌───────┴────────┐
              │             │                │
              │             ▼                ▼
              │          ARCHIVE          PURGE
              │             │                │
              │             └──────┬─────────┘
              │                    ▼
              └──────────────► RETAIN/PURGE
```

---

# 12. Retention Policy

La unidad principal será:

```text
RetentionPolicy
```

Una policy describe:

```text
scope
anchor
minimum duration
maximum duration
hold rules
archive requirements
purge behavior
classification
priority
```

---

# 13. Modelo conceptual

```php
final readonly class RetentionPolicy
{
    public function __construct(
        public RetentionPolicyId $id,
        public RetentionScope $scope,
        public RetentionAnchor $anchor,
        public ?RetentionDuration $minimum,
        public ?RetentionDuration $maximum,
        public PurgePolicy $purge,
        public ArchiveRequirement $archive,
    ) {}
}
```

---

# 14. Policy ID

Toda policy deberá tener identidad estable:

```text
customer-personal-data-v1
invoice-financial-record-v3
audit-security-log-v2
```

No deberá depender únicamente del nombre de una clase PHP.

---

# 15. Versionado

Las políticas deberán ser versionables.

Ejemplo:

```text
financial-records:v1
financial-records:v2
financial-records:v3
```

porque los requisitos pueden cambiar con el tiempo.

---

# 16. Policy version ≠ schema version

```text
RetentionPolicyVersion
≠
SchemaVersion
≠
MigrationVersion
```

---

# 17. Retention Scope

Una policy podrá aplicar a:

```text
Entity Type
Entity Field
Data Classification
Tenant
Relationship
Historical Version
Audit Record
Domain
Custom Dataset
```

---

# 18. Entity-level retention

Ejemplo:

```text
Invoice
retain 5 years
```

---

# 19. Field-level retention

Puede existir:

```text
Customer
├── accounting data → 5 years
├── marketing metadata → 2 years
└── security logs → 1 year
```

Sin embargo, field-level physical purging puede requerir:

```text
redaction
anonymization
tokenization
nullification
```

en lugar de eliminar toda la entidad.

---

# 20. Retention Action

El vencimiento de una policy no implica únicamente DELETE.

Podrán existir:

```php
enum RetentionAction
{
    case KEEP;
    case ARCHIVE;
    case PURGE;
    case REDACT;
    case ANONYMIZE;
    case CUSTOM;
}
```

---

# 21. Data Classification

Las políticas podrán asociarse a clasificación:

```text
PUBLIC
INTERNAL
CONFIDENTIAL
PERSONAL
SENSITIVE
FINANCIAL
SECURITY
AUDIT
CUSTOM
```

---

# 22. Classification ≠ authorization

Clasificar un dato como:

```text
SENSITIVE
```

no define quién puede leerlo.

Authorization continúa siendo responsabilidad del Security/Authorization System.

---

# 23. Retention Anchor

Toda duración necesita un punto de inicio.

```text
RetentionDuration
```

sin anchor es incompleta.

---

# 24. Ejemplo

```text
retain 90 days
```

es ambiguo.

¿Desde?

```text
creation?
last update?
soft deletion?
contract termination?
account closure?
invoice payment?
explicit business event?
```

---

# 25. Anchor types

VoltStack podrá definir:

```php
enum RetentionAnchorType
{
    case CREATED_AT;
    case UPDATED_AT;
    case SOFT_DELETED_AT;
    case DOMAIN_EVENT;
    case TEMPORAL_END;
    case EXPLICIT_FIELD;
    case CUSTOM;
}
```

---

# 26. Soft-delete anchor

Ejemplo:

```text
retain deleted customer for 90 days after soft deletion
```

formalmente:

```text
anchor = deleted_at
duration = P90D
```

---

# 27. Domain event anchor

Ejemplo:

```text
retain contract data for 5 years after termination
```

El anchor será:

```text
ContractTerminated
```

no necesariamente:

```text
updated_at
```

---

# 28. Explicit field anchor

Ejemplo:

```text
closed_at
completed_at
cancelled_at
expired_at
```

---

# 29. Missing anchor

Si la policy requiere un anchor que no existe:

```text
UNKNOWN
```

No:

```text
eligible
```

---

# 30. Critical invariant

> **Missing evidence never makes data purgeable.**

---

# 31. Retention Clock

El sistema modelará conceptualmente:

```text
RetentionClock
```

como cálculo temporal basado en un reloj explícito.

---

# 32. Clock injection

No deberá existir:

```php
new DateTimeImmutable();
```

disperso en el sistema.

Se utilizará:

```text
Clock
```

inyectable.

---

# 33. Why

Permite:

```text
deterministic tests
simulation
replay
dry-run
consistent evaluation
```

---

# 34. Retention Window

Una ventana podrá contener:

```text
minimum retention
maximum retention
```

---

# 35. Minimum retention

Define:

> antes de este momento el dato no puede purgarse por esta policy.

---

# 36. Maximum retention

Puede expresar:

> después de este momento el dato debería eliminarse, anonimizarse o archivarse salvo bloqueo superior.

---

# 37. Minimum ≠ Maximum

Ejemplo:

```text
minimum = 5 years
maximum = 7 years
```

Entre ambos:

```text
purge permitted
```

Después del máximo:

```text
purge required/preferred
```

según policy.

---

# 38. Retention deadlines

Conceptualmente:

```text
EarliestPurgeAt
LatestPreferredPurgeAt
```

---

# 39. Duration representation

Las duraciones deberán ser tipadas.

Ejemplo:

```text
P90D
P1Y
P5Y
```

---

# 40. Calendar duration challenge

```text
365 days
```

no siempre equivale semánticamente a:

```text
1 calendar year
```

---

# 41. Duration semantics

VoltStack distinguirá cuando sea necesario:

```text
FIXED_DURATION
CALENDAR_PERIOD
BUSINESS_PERIOD
CUSTOM
```

---

# 42. Timezone

Retention basada en:

```text
Instant
```

deberá evitar ambigüedad de timezone.

Cuando la regla sea calendárica:

```text
local legal date
```

el timezone deberá formar parte de la policy.

---

# 43. Evaluation result

El motor no retornará sólo:

```text
true / false
```

---

# 44. RetentionDecision

Conceptualmente:

```php
enum RetentionDecision
{
    case RETAIN;
    case PURGE_ELIGIBLE;
    case PURGE_REQUIRED;
    case ARCHIVE_REQUIRED;
    case BLOCKED;
    case UNKNOWN;
}
```

---

# 45. UNKNOWN

Puede aparecer cuando:

```text
anchor unavailable
classification unknown
policy resolution incomplete
archive evidence missing
legal hold state unknown
tenant policy unavailable
```

---

# 46. UNKNOWN ≠ PURGE_ELIGIBLE

Regla fundamental.

---

# 47. UNKNOWN default

Para operaciones destructivas:

```text
UNKNOWN → RETAIN / BLOCK
```

como fail-safe.

---

# 48. Retention Evaluation

Pipeline:

```text
Entity/Data Record
       ↓
Classification
       ↓
Policy Resolution
       ↓
Anchor Resolution
       ↓
Retention Window
       ↓
Hold Evaluation
       ↓
Archive Requirement
       ↓
Purge Eligibility
       ↓
Retention Decision
```

---

# 49. Retention Evaluator

```php
interface RetentionEvaluator
{
    public function evaluate(
        RetentionSubject $subject,
        RetentionContext $context,
    ): RetentionEvaluation;
}
```

---

# 50. RetentionSubject

Puede representar:

```text
Entity
Historical Version
Audit Record
Field
Relationship Data
Custom Record
```

---

# 51. RetentionEvaluation

Podrá contener:

```text
decision
matched policies
effective policy
anchor
earliest purge time
maximum retention time
holds
archive requirement
evidence
warnings
```

---

# 52. Explainability

Toda decisión destructiva deberá poder responder:

```text
¿Por qué este dato es purgeable?
```

---

# 53. Example

```text
Customer #42

Policy:
  customer-deleted-data:v3

Anchor:
  deleted_at

Anchor Value:
  2026-01-01

Minimum Retention:
  180 days

Earliest Purge:
  2026-06-30

Current Evaluation:
  PURGE_ELIGIBLE

Legal Hold:
  none

Archive:
  required

Archive State:
  verified

Final Decision:
  PURGE_ELIGIBLE
```

---

# 54. Multiple policies

Un dato puede estar sujeto a varias policies.

Ejemplo:

```text
Customer
├── company policy
├── tenant policy
├── data classification policy
└── legal hold
```

---

# 55. Policy composition

No deberá elegirse simplemente:

```text
first policy wins
```

---

# 56. Conservative composition

Como regla base:

> La policy efectiva deberá preservar todas las obligaciones de conservación aplicables.

---

# 57. Example

```text
Policy A → retain 2 years
Policy B → retain 5 years
```

Si ambas son obligatorias:

```text
effective minimum retention = 5 years
```

---

# 58. Maximum retention conflict

Más complejo:

```text
Policy A → must delete after 3 years
Policy B → must retain 5 years
```

Esto representa:

```text
POLICY_CONFLICT
```

No debe resolverse silenciosamente.

---

# 59. Retention conflict

Deberá existir:

```text
RetentionPolicyConflict
```

---

# 60. Conflict ≠ arbitrary precedence

Sólo podrá resolverse mediante reglas explícitas como:

```text
legal hierarchy
policy priority
jurisdiction
tenant contract
system policy
manual resolution
```

---

# 61. Policy precedence

Puede modelarse mediante:

```text
source
authority
priority
scope specificity
version
effective period
```

---

# 62. Effective period

Una policy puede ser válida:

```text
from T1
until T2
```

---

# 63. Historical policy question

Si una policy cambia hoy:

```text
¿debe aplicarse retroactivamente?
```

No siempre.

---

# 64. Policy application modes

```text
PROSPECTIVE
RETROACTIVE
EVENT_TIME
CUSTOM
```

---

# 65. Policy snapshot

Para ciertos dominios puede ser necesario registrar:

```text
RetentionPolicyId
RetentionPolicyVersion
```

aplicada al dato/evento.

---

# 66. Legal Hold

Un:

```text
LegalHold
```

impide acciones destructivas aunque la retención ordinaria haya vencido.

---

# 67. Legal hold semantics

```text
Purge Eligible
+
Active Legal Hold
=
BLOCKED
```

---

# 68. LegalHold ≠ RetentionPolicy

Retention dice:

```text
cuánto conservar
```

Legal Hold dice:

```text
no destruir hasta nueva orden/condición
```

---

# 69. Hold scope

Puede aplicar a:

```text
entity
customer
tenant
case
account
data classification
time range
query-defined dataset
custom domain scope
```

---

# 70. Hold identity

Todo hold deberá tener:

```text
stable ID
reason/reference
scope
start
optional end/release
authority
```

---

# 71. Sensitive reason

La razón completa de un Legal Hold puede ser sensible.

Telemetry/debugging no deberá exponerla automáticamente.

---

# 72. Hold states

```text
ACTIVE
RELEASED
EXPIRED
UNKNOWN
```

---

# 73. UNKNOWN hold state

Para purge:

```text
UNKNOWN → BLOCK
```

---

# 74. Soft Delete integration

Una policy podrá usar:

```text
SOFT_DELETED_AT
```

como anchor.

---

# 75. Example

```text
ACTIVE
   │
softDelete()
   ▼
SOFT_DELETED
   │
   │ retain 90 days
   ▼
PURGE_ELIGIBLE
```

---

# 76. But not universal

Algunos datos pueden tener retención desde:

```text
creation
business closure
last activity
contract termination
```

independientemente del soft delete.

---

# 77. Soft delete ≠ automatic retention policy

Una entidad con:

```php
#[SoftDelete]
```

no obtiene automáticamente una duración arbitraria.

---

# 78. Declarative metadata

Ejemplo conceptual:

```php
#[Retention(
    policy: 'customer-deleted-data'
)]
final class Customer
{
}
```

---

# 79. Better separation

Preferentemente:

```text
Entity Metadata
→ RetentionPolicyReference
```

y la policy completa vive en un registry/configuration layer.

Esto evita codificar reglas regulatorias cambiantes directamente en clases.

---

# 80. Retention Policy Registry

Responsable de:

```text
registration
resolution
versioning
validation
freeze
```

---

# 81. Registry mutability

En runtime estable:

```text
compile
→ validate
→ freeze
```

---

# 82. Persistent worker safety

No deberá modificarse una policy global durante un request.

---

# 83. History integration

Una entidad y sus versiones pueden tener distintas policies.

---

# 84. Example

```text
Current Customer Entity:
  retain until account deletion + 90 days

Historical Financial Versions:
  retain 5 years
```

---

# 85. Purging entity ≠ purging history

---

# 86. History policy

Cuando se purga una entidad:

```text
RETAIN_HISTORY
PURGE_HISTORY
ARCHIVE_HISTORY
INDEPENDENT_POLICY
```

podrán ser estrategias.

---

# 87. Recommended architecture

History tendrá su propia:

```text
RetentionSubject
```

y se evaluará independientemente.

---

# 88. Temporal data integration

Temporal versions podrán tener anchors como:

```text
valid_to
system_to
version_created_at
```

---

# 89. Temporal expiration ≠ purge eligibility

Una versión que ya no es temporalmente vigente puede seguir bajo retención.

---

# 90. Archive integration

Una policy puede establecer:

```text
archive required before purge
```

---

# 91. ArchiveRequirement

```php
enum ArchiveRequirement
{
    case NONE;
    case OPTIONAL;
    case REQUIRED;
    case VERIFIED_REQUIRED;
}
```

---

# 92. REQUIRED

Significa que debe existir una operación de archival antes de purge.

---

# 93. VERIFIED_REQUIRED

Además requiere evidencia verificable de que el archive fue creado correctamente.

---

# 94. Archive success ≠ purge success

Son operaciones distintas.

---

# 95. Archive failure

Debe impedir purge cuando archive sea requisito.

---

# 96. Archive UNKNOWN

También deberá bloquear purge cuando la policy requiera verificación.

---

# 97. Purge

Purge representa:

```text
destrucción física o transformación irreversible
```

según policy.

---

# 98. Purge types

```text
ROW_DELETE
FIELD_REDACTION
ANONYMIZATION
CRYPTO_ERASURE
ARCHIVE_AND_DELETE
CUSTOM
```

---

# 99. Purge ≠ forceDelete API

`forceDelete()` es una operación ORM explícita.

Purge es una acción de lifecycle gobernada por Retention.

Puede utilizar physical delete internamente, pero sus precondiciones son distintas.

---

# 100. Purge Planner

Antes de cualquier purge:

```text
Retention Evaluation
        ↓
Hold Evaluation
        ↓
Archive Evaluation
        ↓
Security
        ↓
Relationship Analysis
        ↓
History Policy
        ↓
Distribution
        ↓
Resource Governance
        ↓
Purge Plan
```

---

# 101. PurgePlan

Conceptualmente:

```php
final readonly class PurgePlan
{
    public function __construct(
        public RetentionSubject $subject,
        public RetentionDecision $decision,
        public array $operations,
        public array $preconditions,
        public array $auditRequirements,
        public PurgeSafetyLevel $safety,
    ) {}
}
```

---

# 102. Planner ≠ Executor

El planner:

```text
decides and represents
```

El executor:

```text
executes approved plan
```

---

# 103. Purge eligibility race

Entre:

```text
evaluate
```

y:

```text
delete
```

puede aparecer:

```text
new legal hold
policy update
entity update
archive invalidation
```

---

# 104. Revalidation

Operaciones destructivas deberán poder revalidar precondiciones cerca del execution boundary.

---

# 105. Evaluation token

Puede utilizarse:

```text
RetentionEvaluationToken
```

con:

```text
policy generation
subject version
hold generation
archive state generation
evaluation time
```

---

# 106. Token ≠ authorization

Sólo demuestra qué estado fue evaluado.

---

# 107. Purge transaction

Cuando sea posible:

```text
revalidate
delete/redact
audit DB state
```

podrán formar parte de una misma transaction.

---

# 108. External archive problem

External archive no puede incluirse necesariamente en la misma DB transaction.

---

# 109. Workflow

Ejemplo:

```text
identify candidate
↓
archive
↓
verify archive
↓
mark purge-ready
↓
revalidate retention
↓
purge database
↓
commit
↓
publish audit/event
```

---

# 110. Distributed transaction

VoltStack no fingirá:

```text
atomic DB + object storage + external archive
```

si no existe tal garantía.

---

# 111. Saga/outbox

Flujos externos podrán utilizar:

```text
workflow
saga
outbox
idempotency
reconciliation
```

---

# 112. Purge execution modes

```text
SINGLE
BATCH
CHUNKED
DISTRIBUTED
DRY_RUN
```

---

# 113. Dry Run

Dry-run deberá ser first-class.

Ejemplo:

```bash
db:retention:purge --policy=customer-data --dry-run
```

---

# 114. Dry-run output

Podrá mostrar:

```text
Candidates: 48,291
Eligible: 47,800
Blocked by hold: 211
Archive required: 280
Unknown: 0
Estimated batches: 48
```

---

# 115. Dry Run ≠ guarantee

Entre dry-run y ejecución real el estado puede cambiar.

Por ello la ejecución deberá reevaluar.

---

# 116. Candidate discovery

Debe distinguirse:

```text
candidate
```

de:

```text
eligible
```

---

# 117. Candidate

Un candidate es un dato que potencialmente podría haber vencido.

---

# 118. Eligibility

Un candidate sólo será purgeable después de evaluar todas las policies relevantes.

---

# 119. Query architecture

Candidate discovery utilizará:

```text
Query Engine
```

y no SQL generado manualmente por Retention System.

---

# 120. Retention predicate

Podrán existir semantic predicates como:

```text
RetentionAnchorBefore(...)
RetentionPolicyCandidate(...)
```

que posteriormente se traduzcan a Query AST.

---

# 121. Database filtering

Siempre que sea semánticamente seguro, el sistema deberá filtrar candidates en DB.

---

# 122. PHP full scan anti-pattern

No:

```php
foreach (Customer::all() as $customer) {
    if ($customer->deletedAt < $threshold) {
        // ...
    }
}
```

para millones de registros.

---

# 123. Chunk Processing integration

Para grandes datasets:

```text
Retention Candidate Query
          ↓
Chunk Processing
          ↓
Evaluate
          ↓
Purge Batch
```

---

# 124. Preferred traversal

Cuando el dataset se reduce durante purge:

```text
KEYSET
```

será generalmente preferible a OFFSET.

---

# 125. Chunk ≠ transaction

Cada chunk podrá tener:

```text
PER_CHUNK
```

transaction policy.

---

# 126. Whole traversal transaction

No será default para grandes datasets.

---

# 127. Batch purge

Podrá utilizar:

```text
Bulk Delete
Bulk Update
Bulk Redaction
```

cuando las garantías sean compatibles.

---

# 128. Bulk ≠ entity lifecycle

Al igual que otros bulk systems:

```text
bulk purge
```

no fingirá callbacks individuales si no fueron ejecutados.

---

# 129. History requirements

Si cada purge requiere historial individual, audit individual o relationship logic individual:

```text
bulk strategy
```

deberá demostrar que preserva esas garantías.

---

# 130. Resource Governance

Toda ejecución tendrá budgets.

Ejemplo:

```text
max rows
max duration
max batch size
max transaction duration
max lock duration
max memory
max retries
```

---

# 131. Default safety

Una ejecución sin límites explícitos no deberá asumir:

```text
delete everything eligible in one transaction
```

---

# 132. Retention Runner

Responsable de ejecutar un:

```text
RetentionExecutionPlan
```

de manera acotada.

---

# 133. Runner lifecycle

```text
CREATED
PLANNING
DISCOVERING
EVALUATING
EXECUTING
COMMITTING
CHECKPOINTING
COMPLETED
FAILED
CANCELLED
UNKNOWN
```

---

# 134. Checkpointing

Procesamientos grandes podrán guardar:

```text
RetentionCheckpoint
```

---

# 135. Checkpoint contents

Conceptualmente:

```text
policy ID/version
query fingerprint
last traversal boundary
tenant
shard
policy generation
processed count
purged count
blocked count
execution ID
```

---

# 136. Checkpoint ≠ exactly once

---

# 137. Retry semantics

Purge deberá diseñarse para:

```text
idempotency
re-evaluation
unknown outcomes
```

---

# 138. UNKNOWN commit

Si:

```text
DELETE sent
COMMIT sent
connection lost
```

el resultado será:

```text
UNKNOWN
```

---

# 139. UNKNOWN ≠ retry immediately

Antes de repetir deberá reconciliarse el estado cuando sea posible.

---

# 140. Delete idempotency

Un segundo:

```text
DELETE WHERE id = X
```

puede ser físicamente tolerable.

Pero:

```text
audit
archive
history
external deletion
billing
events
```

pueden no ser idempotentes.

---

# 141. RetentionExecutionId

Cada ejecución podrá tener:

```text
RetentionExecutionId
```

para correlación y deduplicación.

---

# 142. Per-subject operation ID

Podrá existir:

```text
PurgeOperationId
```

para operaciones sensibles.

---

# 143. Audit

Retention deberá integrarse profundamente con:

```text
233_DATABASE_QUERY_AUDIT_SYSTEM.md
```

y auditoría de dominio cuando aplique.

---

# 144. Audit information

Podrá registrar:

```text
policy ID/version
action
scope
evaluation
execution time
result
actor/system identity
retention reason
hold evaluation
archive verification
```

---

# 145. Do not audit purged sensitive value unnecessarily

El audit no deberá copiar el contenido sensible que precisamente se intenta eliminar.

---

# 146. Evidence minimization

Preferir:

```text
stable subject reference
policy
decision
hash/fingerprint if justified
timestamps
result
```

---

# 147. Purge evidence paradox

Debe conservarse evidencia de que la eliminación ocurrió sin conservar innecesariamente el dato eliminado.

---

# 148. Tombstone

Podrá existir un:

```text
PurgeTombstone
```

mínimo.

---

# 149. Tombstone ≠ soft-deleted entity

Un tombstone puede contener sólo:

```text
opaque subject reference
policy
purged_at
operation ID
```

sin conservar los datos originales.

---

# 150. Tombstone retention

Los tombstones tendrán su propia policy.

---

# 151. Security

Retention es un subsistema altamente sensible porque puede destruir datos.

---

# 152. Required capabilities

Podrán distinguirse:

```text
retention.view
retention.evaluate
retention.simulate
retention.execute
retention.override
retention.manage_policy
retention.manage_hold
retention.release_hold
```

---

# 153. Policy administration

Modificar una retention policy será una operación privilegiada.

---

# 154. Hold release

Liberar un Legal Hold será especialmente sensible.

---

# 155. Retention override

Un override manual deberá:

```text
require authorization
require reason
be audited
be scoped
have optional expiration
```

---

# 156. No silent override

No:

```php
$retention->ignoreEverything();
```

como escape hatch ordinaria.

---

# 157. Break-glass

Si existe un mecanismo excepcional:

```text
BREAK_GLASS
```

deberá ser:

```text
explicit
authorized
audited
time-bound
observable
```

---

# 158. Tenant Retention

En sistemas multitenant:

```text
Tenant A
```

puede tener una policy distinta de:

```text
Tenant B
```

---

# 159. Tenant policy

Ejemplo:

```text
Base platform:
  minimum 90 days

Tenant A:
  retain 1 year

Tenant B:
  retain 3 years
```

---

# 160. Effective policy

Si todas son obligaciones mínimas:

```text
effective = strongest applicable minimum
```

según composition rules.

---

# 161. Tenant policy cannot weaken mandatory platform/legal policy

A menos que el sistema de policy authority declare explícitamente que puede hacerlo.

---

# 162. Tenant context

Candidate discovery deberá estar vinculado a:

```text
TenantContext
```

---

# 163. No cross-tenant purge

Una ejecución tenant-scoped no podrá purgar otro tenant.

---

# 164. Platform-wide purge

Deberá ser un modo explícito y privilegiado.

---

# 165. Tenant deletion

Cuando se elimina un tenant:

```text
delete tenant
```

no implica necesariamente:

```text
immediately purge all tenant data
```

---

# 166. Tenant offboarding

Puede requerir:

```text
soft disable
export
archive
retention period
legal hold check
purge
```

---

# 167. Sharding

Retention deberá operar sobre la topología real.

---

# 168. Per-shard discovery

Normalmente:

```text
Shard A → candidates
Shard B → candidates
Shard C → candidates
```

podrán procesarse independientemente.

---

# 169. Global ordering

No será requerido salvo que la policy lo necesite.

---

# 170. Per-shard checkpoints

Preferidos para escalabilidad.

---

# 171. Shard map generation

Checkpoint deberá poder incluir:

```text
ShardMapGeneration
```

---

# 172. Topology change

Si la topología cambia:

```text
resume
```

deberá validar que la continuación siga siendo correcta.

---

# 173. No fake distributed transaction

VoltStack no prometerá atomicidad global entre shards.

---

# 174. Partial execution

Resultado distribuido podrá ser:

```text
COMPLETED
PARTIAL
FAILED
UNKNOWN
```

---

# 175. PARTIAL ≠ COMPLETED

---

# 176. Replica usage

Candidate discovery podría ejecutarse en replica sólo cuando la consistency policy lo permita.

---

# 177. Destructive decision authority

La decisión final antes de purge deberá usar evidencia suficientemente fresca.

---

# 178. Stale replica danger

Una replica podría no conocer:

```text
new legal hold
recent update
policy-related state
```

---

# 179. Recommended default

La validación destructiva final deberá ejecutarse contra:

```text
writer / authoritative source
```

salvo capability/consistency explícita equivalente.

---

# 180. Cache

Retention decisions destructivas no deberán depender exclusivamente de caches potencialmente stale.

---

# 181. Metadata cache

Sí podrá cachearse:

```text
compiled retention policy metadata
```

con generación/versionado.

---

# 182. Decision cache

Si existe, deberá tener:

```text
strict validity
policy generation
subject version
hold generation
expiration
```

---

# 183. Cached eligibility ≠ final authorization to purge

---

# 184. Policy change invalidation

Cambiar una policy deberá invalidar decisiones derivadas incompatibles.

---

# 185. Hold change invalidation

Crear/release Legal Hold también.

---

# 186. Event architecture

Eventos conceptuales:

```text
RetentionEvaluationStarted
RetentionEvaluated
RetentionCandidateDiscovered
RetentionPurgePlanned
RetentionPurgeStarted
RetentionSubjectPurged
RetentionPurgeCommitted
RetentionPurgeBlocked
RetentionPurgeFailed
RetentionOutcomeUnknown
LegalHoldApplied
LegalHoldReleased
```

---

# 187. Event volume

Eventos per-row pueden ser enormes.

Deberá existir política de:

```text
aggregation
sampling
batch events
audit vs telemetry separation
```

---

# 188. Event ≠ Audit

Un evento operacional no sustituye registro de auditoría requerido.

---

# 189. AfterCommit

Los eventos que afirman:

```text
PURGED
```

deberán emitirse después de conocer el commit cuando requieran certeza.

---

# 190. Outbox

Para sincronizar:

```text
search indexes
external archives
external processors
```

se recomienda Outbox.

---

# 191. Telemetry

Métricas posibles:

```text
database_retention_evaluations_total
database_retention_candidates_total
database_retention_purged_total
database_retention_blocked_total
database_retention_unknown_total
database_retention_archive_required_total
database_retention_failures_total
database_retention_execution_duration
```

---

# 192. Bounded labels

Permitidos:

```text
policy category
decision
action
result
execution mode
```

Evitar:

```text
entity ID
tenant ID
customer email
case ID
```

---

# 193. Tracing

Spans:

```text
database.retention.plan
database.retention.discover
database.retention.evaluate
database.retention.archive_check
database.retention.purge
database.retention.checkpoint
```

---

# 194. Slow retention detection

Una ejecución podrá detectar:

```text
slow candidate query
slow batch
long transaction
lock contention
archive bottleneck
```

---

# 195. Diagnostics

Ejemplo:

```text
Retention Diagnostics
────────────────────────────────

Subject:
  Customer

Policy:
  customer-deleted-data:v3

Anchor:
  SOFT_DELETED_AT

Minimum:
  P180D

Maximum:
  P365D

Archive:
  VERIFIED_REQUIRED

Legal Hold:
  none

Tenant Policy:
  inherited

Purge:
  ROW_DELETE

Decision:
  PURGE_ELIGIBLE
```

---

# 196. Explain command

Podrá existir:

```bash
volt database:retention:explain Customer 42
```

---

# 197. Explain output

```text
Policy resolution
  ✓ platform policy
  ✓ entity policy
  ✓ tenant policy

Anchor
  deleted_at
  2026-01-01T10:00:00Z

Minimum retention
  180 days

Earliest purge
  2026-06-30T10:00:00Z

Legal holds
  none

Archive
  verified

Decision
  PURGE_ELIGIBLE
```

---

# 198. Policy validation

Durante bootstrap/compilation deberán detectarse:

```text
missing policy IDs
invalid durations
impossible windows
unknown anchors
cyclic policy dependencies
conflicting actions
unsupported purge strategies
invalid archive requirements
```

---

# 199. Impossible window

Ejemplo:

```text
minimum = 7 years
maximum = 5 years
```

sin semántica especial.

Debe ser error.

---

# 200. Compilation

Retention metadata deberá integrarse con:

```text
246_DATABASE_METADATA_COMPILATION_SYSTEM.md
```

---

# 201. CompiledRetentionPolicy

Podrá contener:

```text
resolved scope
compiled anchor resolver
duration semantics
action
hold rules
archive requirements
policy fingerprint
generation
```

---

# 202. Runtime efficiency

No reevaluar configuración textual compleja para cada row si puede compilarse una vez.

---

# 203. Immutable compiled policies

Podrán compartirse entre requests si no contienen estado mutable/request-specific.

---

# 204. Persistent runtime

Mutable state como:

```text
current tenant
current execution
current checkpoint
current override
current clock snapshot
```

será scope-local.

---

# 205. Forbidden

No:

```php
static ?TenantId $tenant;
static bool $ignoreRetention;
static ?RetentionExecution $current;
```

---

# 206. FrankenPHP

Cada request/job deberá obtener su propio:

```text
RetentionContext
```

---

# 207. RoadRunner

Misma regla.

---

# 208. OpenSwoole

Context state deberá ser:

```text
coroutine-safe
```

---

# 209. RetentionContext

Conceptualmente:

```php
final readonly class RetentionContext
{
    public function __construct(
        public Instant $evaluationTime,
        public ?TenantContext $tenant,
        public RetentionExecutionMode $mode,
        public AuthorizationContext $authorization,
    ) {}
}
```

---

# 210. Stable evaluation time

Durante un batch puede ser conveniente fijar:

```text
evaluationTime = T0
```

para que todos los elementos se evalúen contra el mismo instante lógico.

---

# 211. Fixed evaluation time ≠ database snapshot

Aunque todos usen T0:

```text
database state
```

puede cambiar durante la ejecución.

---

# 212. Consistency profiles

Podrán existir:

```text
BEST_EFFORT
AUTHORITATIVE_REVALIDATION
SNAPSHOT
CUSTOM
```

---

# 213. Default destructive profile

Recomendado:

```text
AUTHORITATIVE_REVALIDATION
```

---

# 214. Relationship retention

Parent y child pueden tener policies distintas.

---

# 215. Example

```text
Customer
└── Invoices
```

Customer puede ser purgeable después de 90 días.

Invoices pueden requerir 5 años.

---

# 216. Therefore

```text
purge Customer
```

no puede asumir:

```text
cascade delete Invoices
```

---

# 217. Relationship planner

Deberá detectar:

```text
retained dependents
FK restrictions
ownership
independent retention
archive dependencies
```

---

# 218. Strategies

Podrán incluir:

```text
BLOCK_PARENT_PURGE
DETACH
ANONYMIZE_PARENT
RETAIN_TOMBSTONE
REKEY_RELATION
CUSTOM
```

según modelo de dominio.

---

# 219. DB cascade danger

```text
ON DELETE CASCADE
```

puede destruir datos que aún están bajo retention.

---

# 220. Critical rule

> **Una foreign key cascade física nunca deberá utilizarse como sustituto de evaluación de retention.**

---

# 221. Schema diagnostics

VoltStack podrá advertir:

```text
Retention-protected child
+
ON DELETE CASCADE parent
```

cuando represente riesgo.

---

# 222. Anonymization

En algunos casos no se necesita eliminar la fila completa.

---

# 223. Example

```text
Customer
├── id             retained
├── statistics     retained
├── email          anonymized
├── name           anonymized
└── address        removed
```

---

# 224. Anonymization ≠ encryption

---

# 225. Anonymization ≠ masking

Masking puede ser reversible o sólo de presentación.

Anonymization busca eliminar la capacidad razonable de reidentificación según la policy definida.

---

# 226. Database system limitation

VoltStack puede ejecutar transformaciones técnicas, pero la determinación legal de si un dataset está realmente anonimizado pertenece a políticas externas al motor.

---

# 227. Redaction plan

```text
Retention Policy
      ↓
Fields
      ↓
Transformation Rules
      ↓
Update Query Models
      ↓
Persistence
```

---

# 228. Crypto erasure

Para datos cifrados bajo claves segregadas, una policy avanzada podría utilizar:

```text
CRYPTO_ERASURE
```

---

# 229. Crypto erasure requirements

Sólo será válida si la arquitectura criptográfica demuestra que destruir la clave vuelve inaccesible el dato requerido.

---

# 230. Crypto erase ≠ DELETE

---

# 231. Backup problem

Aunque se purgue producción, copias de seguridad pueden contener versiones anteriores.

---

# 232. Retention + Backup

Deberá existir integración documental/policy con:

```text
backup retention
backup rotation
restore procedures
post-restore re-purge
```

---

# 233. Restored backup problem

Si se restaura un backup antiguo pueden reaparecer datos ya purgados.

---

# 234. Purge ledger

Un:

```text
PurgeLedger
```

mínimo puede ayudar a reaplicar eliminaciones después de restore.

---

# 235. Purge Ledger ≠ full deleted data copy

Debe minimizar contenido.

---

# 236. Restore reconciliation

Después de restaurar backup:

```text
restore
↓
load purge ledger
↓
identify data that must remain purged
↓
reapply purge/redaction
↓
open service
```

cuando la política operacional lo requiera.

---

# 237. Backups independent lifecycle

El Database Retention System no controlará directamente todos los proveedores de backup.

Podrá emitir:

```text
requirements
events
integration contracts
```

para Backup System.

---

# 238. Import

Importar datos históricos deberá preservar o reconstruir:

```text
retention anchors
policy identity
classification
```

cuando sea necesario.

---

# 239. Import timestamp danger

No usar automáticamente:

```text
imported_at
```

como retention anchor si el dato real tiene 5 años.

---

# 240. Export

Exportar datos no reinicia retention automáticamente.

---

# 241. Copy semantics

Crear una copia puede crear una nueva retention subject dependiendo del dominio.

Esto deberá ser explícito.

---

# 242. Data migration

Mover datos:

```text
Database A → Database B
```

no debe resetear retention clock accidentalmente.

---

# 243. Clock preservation

Los anchors relevantes deberán preservarse durante migraciones.

---

# 244. Testing architecture

El sistema requerirá pruebas en varias capas.

---

# 245. Unit tests

Probar:

```text
anchor resolution
duration calculation
minimum window
maximum window
policy composition
conflicts
hold evaluation
archive requirement
decision generation
```

---

# 246. Deterministic clock tests

Ejemplo:

```text
Anchor:
  2026-01-01

Duration:
  90 days

Clock:
  2026-03-01
```

Resultado:

```text
RETAIN
```

---

# 247. Boundary tests

Probar exactamente:

```text
T < deadline
T = deadline
T > deadline
```

---

# 248. Calendar tests

Incluir:

```text
leap years
month boundaries
DST
timezone transitions
end-of-month semantics
```

cuando aplique.

---

# 249. Hold tests

```text
active hold
released hold
expired hold
unknown hold
multiple holds
```

---

# 250. Policy conflict tests

Demostrar que:

```text
must delete after 3 years
must retain 5 years
```

no se resuelve silenciosamente.

---

# 251. Soft-delete integration tests

```text
soft delete
retention clock starts
before deadline → retain
after deadline → eligible
```

---

# 252. Restore integration

Si una entidad se restaura antes de purge:

```text
SOFT_DELETED
→ ACTIVE
```

la policy deberá determinar si:

```text
retention clock cancelled
retention clock preserved
new lifecycle starts
```

---

# 253. No universal restore rule

Será policy-driven.

---

# 254. Transaction tests

```text
purge commit
purge rollback
deadlock retry
unknown commit
savepoint
```

---

# 255. Archive tests

```text
archive required + missing
archive required + failed
archive required + verified
archive unknown
```

---

# 256. Relationship tests

```text
parent eligible
child retained
FK restrict
DB cascade
anonymization strategy
```

---

# 257. Multitenancy tests

```text
Tenant A policy
Tenant B policy
same entity type
different effective retention
```

sin leakage.

---

# 258. Sharding tests

```text
per-shard execution
partial shard failure
topology change
checkpoint resume
```

---

# 259. Persistent runtime tests

Request A:

```text
Tenant A
RetentionPolicy A
```

Request B:

```text
Tenant B
```

no deberá heredar contexto de A.

---

# 260. Failure tests

```text
policy unavailable
anchor unavailable
hold provider unavailable
archive provider unavailable
database unavailable
commit unknown
checkpoint failure
```

---

# 261. Fail-safe property

Cuando la evidencia necesaria para destrucción sea desconocida:

```text
do not purge
```

por default.

---

# 262. Directory proposal

```text
src/Quantum/Database/Retention/
│
├── Contract/
│   ├── RetentionEvaluator.php
│   ├── RetentionPolicyResolver.php
│   ├── RetentionAnchorResolver.php
│   ├── RetentionPlanner.php
│   ├── RetentionRunner.php
│   ├── PurgePlanner.php
│   └── LegalHoldResolver.php
│
├── Policy/
│   ├── RetentionPolicy.php
│   ├── RetentionPolicyId.php
│   ├── RetentionPolicyVersion.php
│   ├── RetentionPolicyRegistry.php
│   ├── RetentionScope.php
│   ├── RetentionDuration.php
│   ├── RetentionAction.php
│   ├── RetentionApplicationMode.php
│   └── CompiledRetentionPolicy.php
│
├── Anchor/
│   ├── RetentionAnchor.php
│   ├── RetentionAnchorType.php
│   ├── CreatedAtAnchor.php
│   ├── UpdatedAtAnchor.php
│   ├── SoftDeletedAtAnchor.php
│   ├── DomainEventAnchor.php
│   ├── FieldAnchor.php
│   └── CustomAnchor.php
│
├── Evaluation/
│   ├── RetentionSubject.php
│   ├── RetentionContext.php
│   ├── RetentionEvaluation.php
│   ├── RetentionDecision.php
│   ├── RetentionEvaluationToken.php
│   └── DefaultRetentionEvaluator.php
│
├── Hold/
│   ├── LegalHold.php
│   ├── LegalHoldId.php
│   ├── LegalHoldState.php
│   ├── LegalHoldScope.php
│   └── DefaultLegalHoldResolver.php
│
├── Archive/
│   ├── ArchiveRequirement.php
│   ├── ArchiveEvidence.php
│   └── ArchiveVerification.php
│
├── Purge/
│   ├── PurgePolicy.php
│   ├── PurgeAction.php
│   ├── PurgePlan.php
│   ├── PurgeOperationId.php
│   ├── PurgeTombstone.php
│   ├── DefaultPurgePlanner.php
│   └── PurgeExecutor.php
│
├── Execution/
│   ├── RetentionExecutionId.php
│   ├── RetentionExecutionPlan.php
│   ├── RetentionExecutionMode.php
│   ├── RetentionExecutionState.php
│   ├── RetentionCheckpoint.php
│   └── DefaultRetentionRunner.php
│
├── Classification/
│   ├── DataClassification.php
│   └── DataClassificationResolver.php
│
├── Security/
│   ├── RetentionPermission.php
│   ├── RetentionOverride.php
│   └── RetentionBreakGlassPolicy.php
│
├── Diagnostics/
│   ├── RetentionDiagnostics.php
│   ├── RetentionExplain.php
│   └── RetentionConflictReport.php
│
├── Telemetry/
│   └── RetentionTelemetry.php
│
└── Exception/
    ├── RetentionException.php
    ├── RetentionPolicyException.php
    ├── RetentionPolicyConflictException.php
    ├── RetentionAnchorException.php
    ├── RetentionUnknownException.php
    ├── RetentionBlockedException.php
    ├── LegalHoldException.php
    ├── ArchiveRequirementException.php
    ├── PurgeException.php
    ├── PurgeSafetyException.php
    └── RetentionOutcomeUnknownException.php
```

---

# 263. Exception hierarchy

```text
DatabaseException
└── RetentionException
    ├── RetentionPolicyException
    │   ├── InvalidRetentionPolicyException
    │   └── RetentionPolicyConflictException
    │
    ├── RetentionAnchorException
    ├── RetentionUnknownException
    ├── RetentionBlockedException
    ├── LegalHoldException
    ├── ArchiveRequirementException
    ├── PurgeException
    │   └── PurgeSafetyException
    │
    └── RetentionOutcomeUnknownException
```

---

# 264. Architectural invariants

## DB-RETENTION-001

Retention será distinto de Soft Delete.

## DB-RETENTION-002

Retention será distinto de Archive.

## DB-RETENTION-003

Retention será distinto de Backup.

## DB-RETENTION-004

Retention será distinto de History.

## DB-RETENTION-005

Retention será distinto de Temporal Validity.

## DB-RETENTION-006

Retention será distinto de Cache TTL.

## DB-RETENTION-007

Retention será distinto de Hard Delete.

## DB-RETENTION-008

Purge eligibility será distinto de purge execution.

## DB-RETENTION-009

Toda duración tendrá anchor explícito.

## DB-RETENTION-010

Missing anchor no significará purgeable.

## DB-RETENTION-011

UNKNOWN no significará purgeable.

## DB-RETENTION-012

UNKNOWN destructivo será fail-safe por default.

## DB-RETENTION-013

Retention policies tendrán IDs estables.

## DB-RETENTION-014

Retention policies serán versionables.

## DB-RETENTION-015

Policy version será distinta de schema version.

## DB-RETENTION-016

Policies podrán tener effective periods.

## DB-RETENTION-017

Retroactividad será explícita.

## DB-RETENTION-018

Multiple policies se compondrán explícitamente.

## DB-RETENTION-019

First policy wins no será regla implícita.

## DB-RETENTION-020

Conflicting mandatory policies producirán conflicto explícito.

## DB-RETENTION-021

Policy conflict no se resolverá arbitrariamente.

## DB-RETENTION-022

Minimum retention será distinta de maximum retention.

## DB-RETENTION-023

Calendar period será distinguible de fixed duration.

## DB-RETENTION-024

Timezone será explícito cuando afecte semántica.

## DB-RETENTION-025

Clock será inyectable.

## DB-RETENTION-026

Evaluation time podrá fijarse por ejecución.

## DB-RETENTION-027

Fixed evaluation time no implicará DB snapshot.

## DB-RETENTION-028

Legal Hold será distinto de Retention Policy.

## DB-RETENTION-029

Active Legal Hold bloqueará purge.

## DB-RETENTION-030

Unknown Legal Hold state bloqueará purge por default.

## DB-RETENTION-031

Legal Hold tendrá scope explícito.

## DB-RETENTION-032

Legal Hold release será privilegiado.

## DB-RETENTION-033

Soft Delete podrá actuar como retention anchor.

## DB-RETENTION-034

Soft Delete no creará duración arbitraria automáticamente.

## DB-RETENTION-035

Restore semantics respecto a retention serán policy-driven.

## DB-RETENTION-036

Entity retention será independiente de history retention.

## DB-RETENTION-037

Purging entity no implicará purging history.

## DB-RETENTION-038

Temporal expiration no implicará purge eligibility.

## DB-RETENTION-039

Archive requirement será explícito.

## DB-RETENTION-040

Archive failure bloqueará purge cuando archive sea obligatorio.

## DB-RETENTION-041

Archive UNKNOWN bloqueará verified-required purge.

## DB-RETENTION-042

Archive success será distinto de purge success.

## DB-RETENTION-043

Purge podrá significar delete, redact, anonymize o estrategia explícita.

## DB-RETENTION-044

Purge será distinto de forceDelete ORM.

## DB-RETENTION-045

Purge Planner no ejecutará SQL.

## DB-RETENTION-046

Retention Evaluator no ejecutará SQL destructivo.

## DB-RETENTION-047

Execution deberá revalidar precondiciones destructivas.

## DB-RETENTION-048

Evaluation token no será authorization token.

## DB-RETENTION-049

External archive no fingirá atomicidad con DB.

## DB-RETENTION-050

Distributed external workflows podrán usar outbox/saga.

## DB-RETENTION-051

Dry-run será first-class.

## DB-RETENTION-052

Dry-run no garantizará estado futuro.

## DB-RETENTION-053

Candidate será distinto de eligible.

## DB-RETENTION-054

Candidate discovery utilizará Query Engine.

## DB-RETENTION-055

Retention System no generará SQL manualmente.

## DB-RETENTION-056

Large retention execution utilizará bounded processing.

## DB-RETENTION-057

Whole-dataset transaction no será default.

## DB-RETENTION-058

Chunk será distinto de transaction.

## DB-RETENTION-059

Keyset será preferible cuando purge reduzca el dataset y sea compatible.

## DB-RETENTION-060

Bulk purge no fingirá entity lifecycle.

## DB-RETENTION-061

Bulk purge deberá demostrar compatibilidad con history/audit requirements.

## DB-RETENTION-062

Resource budgets serán explícitos.

## DB-RETENTION-063

Unlimited destructive batch no será default.

## DB-RETENTION-064

Checkpoints no prometerán exactly-once.

## DB-RETENTION-065

UNKNOWN commit permanecerá UNKNOWN.

## DB-RETENTION-066

UNKNOWN commit no se reintentará ciegamente.

## DB-RETENTION-067

Physical idempotency no implicará workflow idempotency.

## DB-RETENTION-068

RetentionExecutionId podrá correlacionar ejecución.

## DB-RETENTION-069

PurgeOperationId podrá soportar deduplicación.

## DB-RETENTION-070

Audit no copiará datos sensibles innecesariamente.

## DB-RETENTION-071

Purge evidence deberá minimizar información.

## DB-RETENTION-072

Purge tombstone será distinto de soft-deleted entity.

## DB-RETENTION-073

Tombstones tendrán su propia retention.

## DB-RETENTION-074

Retention execution será privilegiada.

## DB-RETENTION-075

Retention policy administration será privilegiada.

## DB-RETENTION-076

Retention override será explícito.

## DB-RETENTION-077

Override será auditado.

## DB-RETENTION-078

Break-glass será excepcional y observable.

## DB-RETENTION-079

Tenant policy será tenant-scoped.

## DB-RETENTION-080

Tenant override no debilitará policy superior sin autoridad explícita.

## DB-RETENTION-081

Tenant execution no purgará otro tenant.

## DB-RETENTION-082

Tenant deletion no implicará immediate purge.

## DB-RETENTION-083

Tenant offboarding podrá tener lifecycle independiente.

## DB-RETENTION-084

Sharded retention podrá operar per-shard.

## DB-RETENTION-085

Global ordering no será obligatorio por default.

## DB-RETENTION-086

Checkpoints podrán ser per-shard.

## DB-RETENTION-087

Shard map generation podrá formar parte del checkpoint.

## DB-RETENTION-088

Topology changes se validarán al resume.

## DB-RETENTION-089

No se prometerá global distributed ACID por default.

## DB-RETENTION-090

PARTIAL será distinto de COMPLETED.

## DB-RETENTION-091

Destructive final validation usará evidencia suficientemente fresca.

## DB-RETENTION-092

Stale replica no será autoridad destructiva por default.

## DB-RETENTION-093

Cached eligibility no autorizará purge por sí sola.

## DB-RETENTION-094

Policy changes invalidarán derived decisions incompatibles.

## DB-RETENTION-095

Hold changes invalidarán derived decisions incompatibles.

## DB-RETENTION-096

Retention event será distinto de audit record.

## DB-RETENTION-097

Committed purge event requerirá commit conocido.

## DB-RETENTION-098

Per-row telemetry no será default.

## DB-RETENTION-099

Telemetry labels serán bounded.

## DB-RETENTION-100

Sensitive IDs no serán telemetry labels.

## DB-RETENTION-101

Policies serán compilables.

## DB-RETENTION-102

Compiled policies serán immutable.

## DB-RETENTION-103

Compiled shared policies no contendrán request state.

## DB-RETENTION-104

Tenant context será scope-local.

## DB-RETENTION-105

Execution context será scope-local.

## DB-RETENTION-106

No habrá global `$ignoreRetention`.

## DB-RETENTION-107

FrankenPHP no filtrará retention state entre requests.

## DB-RETENTION-108

RoadRunner no filtrará retention state entre jobs/requests.

## DB-RETENTION-109

OpenSwoole no compartirá mutable retention state entre coroutines.

## DB-RETENTION-110

Parent retention será independiente de child retention.

## DB-RETENTION-111

Parent purge no implicará child purge.

## DB-RETENTION-112

DB ON DELETE CASCADE no sustituirá retention evaluation.

## DB-RETENTION-113

Retention-protected children podrán bloquear parent purge.

## DB-RETENTION-114

Anonymization será distinta de masking.

## DB-RETENTION-115

Anonymization será distinta de encryption.

## DB-RETENTION-116

Crypto erasure será capability/policy-driven.

## DB-RETENTION-117

Production purge no garantizará eliminación inmediata de backups.

## DB-RETENTION-118

Backup lifecycle será explícito.

## DB-RETENTION-119

Restore de backup deberá considerar previously purged data.

## DB-RETENTION-120

Purge ledger minimizará datos almacenados.

## DB-RETENTION-121

Import no reseteará retention anchor arbitrariamente.

## DB-RETENTION-122

Export no reseteará retention clock automáticamente.

## DB-RETENTION-123

Data migration preservará retention anchors cuando corresponda.

## DB-RETENTION-124

Retention decision será explainable.

## DB-RETENTION-125

Destructive decision conservará evidencia suficiente.

## DB-RETENTION-126

Missing evidence no será interpretada como ausencia de obligación.

## DB-RETENTION-127

Metadata compilation validará policies.

## DB-RETENTION-128

Impossible retention windows serán error.

## DB-RETENTION-129

Unsupported destructive strategy será error/unsupported, no fallback silencioso.

## DB-RETENTION-130

Platform capability será distinta de retention policy.

## DB-RETENTION-131

Compiler no decidirá retention policy.

## DB-RETENTION-132

Driver no conocerá retention semantics.

## DB-RETENTION-133

Connection no resolverá Legal Holds.

## DB-RETENTION-134

ORM no generará SQL de purge.

## DB-RETENTION-135

Query Engine representará candidate predicates.

## DB-RETENTION-136

Persistence/Execution ejecutará operaciones ya planificadas.

## DB-RETENTION-137

Authorization será independiente de retention eligibility.

## DB-RETENTION-138

Purge eligibility no otorgará permission to purge.

## DB-RETENTION-139

Permission to purge no convertirá dato no elegible en elegible.

## DB-RETENTION-140

Override será requerido para romper policy ordinaria cuando esté permitido.

## DB-RETENTION-141

Policy authority será explícita.

## DB-RETENTION-142

Data classification será distinta de authorization.

## DB-RETENTION-143

Field-level retention podrá usar redaction.

## DB-RETENTION-144

Field-level retention no exigirá eliminar toda la row.

## DB-RETENTION-145

Retention jobs serán cancelables.

## DB-RETENTION-146

Cancellation liberará recursos.

## DB-RETENTION-147

Cancellation no se reportará como successful completion.

## DB-RETENTION-148

Execution deadline será soportable.

## DB-RETENTION-149

Transaction budgets serán respetados.

## DB-RETENTION-150

Lock budgets serán respetados.

## DB-RETENTION-151

Retention queries serán observables.

## DB-RETENTION-152

Slow retention operations serán diagnosticables.

## DB-RETENTION-153

Policy resolution será observable sin exponer secretos.

## DB-RETENTION-154

Legal Hold reason no se expondrá por telemetry default.

## DB-RETENTION-155

Archive evidence podrá ser verificada.

## DB-RETENTION-156

Archive verification será distinta de archive existence claim.

## DB-RETENTION-157

Purge execution podrá ser resumible.

## DB-RETENTION-158

Resume validará policy generation.

## DB-RETENTION-159

Resume validará tenant/shard context.

## DB-RETENTION-160

Resume no asumirá que old eligibility sigue vigente.

## DB-RETENTION-161

Retention lifecycle será deterministicamente testeable.

## DB-RETENTION-162

Boundary temporal será explícitamente testeada.

## DB-RETENTION-163

Policy conflicts serán testeables.

## DB-RETENTION-164

UNKNOWN paths serán testeables.

## DB-RETENTION-165

Rollback no publicará purge como committed.

## DB-RETENTION-166

AfterCommit failure no revertirá DB commit.

## DB-RETENTION-167

External side effects usarán mecanismos resilientes cuando requieran garantía.

## DB-RETENTION-168

Retention no será implementada como cron + DELETE ad hoc.

## DB-RETENTION-169

Retention no será implementada como TTL genérico.

## DB-RETENTION-170

Retention no será implementada como simple columna `expires_at` universal.

## DB-RETENTION-171

Retention podrá usar `expires_at` como evidencia cuando la policy lo defina.

## DB-RETENTION-172

ExpiresAt será distinto de PurgedAt.

## DB-RETENTION-173

EligibleAt será distinto de ExecutedAt.

## DB-RETENTION-174

Policy evaluation será distinta de policy enforcement.

## DB-RETENTION-175

Retention System será una capa de governance, no un segundo ORM.

## DB-RETENTION-176

Retention System reutilizará Query, Transaction, Security, Telemetry y Persistence.

## DB-RETENTION-177

Retention no creará un segundo transaction engine.

## DB-RETENTION-178

Retention no creará un segundo cache engine.

## DB-RETENTION-179

Retention no creará un segundo scheduler.

## DB-RETENTION-180

Retention no creará un segundo event system.

---

# 265. Modelo formal

Sea:

```text
x
```

un RetentionSubject.

Sea:

```text
P(x)
```

el conjunto de policies aplicables.

Sea:

```text
A_p(x)
```

el anchor temporal para policy `p`.

Y:

```text
Dmin(p)
```

la duración mínima.

Entonces:

```text
EarliestPurge_p(x)
=
A_p(x) + Dmin(p)
```

---

# 266. Elegibilidad temporal

Para instante de evaluación:

```text
t
```

una policy permite temporalmente purge cuando:

```text
t >= EarliestPurge_p(x)
```

---

# 267. Composición mínima

Si múltiples policies obligatorias aplican:

```text
EarliestPurge(x)
=
max(
    EarliestPurge_p1(x),
    EarliestPurge_p2(x),
    ...
)
```

si sus semánticas son compatibles.

---

# 268. Hold

Definimos:

```text
H(x,t)
```

como existencia de hold activo.

Entonces:

```text
PurgeEligible(x,t)
=
TemporalEligible(x,t)
∧ ¬H(x,t)
∧ ArchiveSatisfied(x)
∧ SecuritySatisfied(x)
∧ RelationshipConstraintsSatisfied(x)
∧ EvidenceComplete(x)
```

---

# 269. Unknown evidence

Si cualquiera de las precondiciones críticas es:

```text
UNKNOWN
```

entonces, por default:

```text
PurgeEligible(x,t) = false
```

---

# 270. Arquitectura conceptual final

```text
                   Data / Entity / Version
                            │
                            ▼
                    Retention Subject
                            │
                            ▼
                   Classification Resolver
                            │
                            ▼
                     Policy Resolver
                            │
                ┌───────────┼────────────┐
                │           │            │
                ▼           ▼            ▼
             Platform     Tenant      Domain
              Policy      Policy      Policy
                │           │            │
                └───────────┼────────────┘
                            ▼
                     Policy Composition
                            │
                            ▼
                      Anchor Resolver
                            │
                            ▼
                     Retention Window
                            │
                            ▼
                       Hold Resolver
                            │
                            ▼
                    Archive Requirement
                            │
                            ▼
                   Retention Evaluation
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       RETAIN            BLOCKED       PURGE_ELIGIBLE
                                               │
                                               ▼
                                         Purge Planner
                                               │
                                      ┌────────┼─────────┐
                                      │        │         │
                                      ▼        ▼         ▼
                                   Delete   Redact   Anonymize
                                      │        │         │
                                      └────────┼─────────┘
                                               ▼
                                      Persistence Engine
                                               │
                                               ▼
                                         Transaction
                                               │
                                               ▼
                                            Commit
                                               │
                                               ▼
                                      Audit / Events / Cache
```

---

# 271. Relación con arquitectura general

```text
Retention System
      │
      ├── Metadata System
      ├── Temporal Data System
      ├── History System
      ├── Soft Delete System
      ├── Query Engine
      ├── Relationship System
      ├── Transaction System
      ├── Cache System
      ├── Event System
      ├── Telemetry System
      ├── Security System
      ├── Resource Governance
      ├── Multitenancy
      ├── Sharding
      ├── Backup System
      └── Archive System
```

Pero no reemplaza ninguno.

---

# 272. Regla de dependencia

```text
Retention
   ↓
Query / Persistence abstractions
   ↓
Execution
   ↓
Connection
   ↓
Driver
```

Nunca:

```text
Driver
→ Retention Policy
```

---

# 273. API conceptual

Una API administrativa podría permitir:

```php
$evaluation = $retention->evaluate($customer);

if ($evaluation->isPurgeEligible()) {
    // informational only
}
```

---

# 274. Dry-run

```php
$report = $retention
    ->forPolicy('customer-deleted-data')
    ->dryRun();
```

---

# 275. Controlled execution

```php
$retention
    ->forPolicy('customer-deleted-data')
    ->execute(
        batchSize: 500,
        mode: RetentionExecutionMode::PURGE,
    );
```

---

# 276. Query integration

```php
Customer::query()
    ->retentionCandidates('customer-deleted-data');
```

será una convenience API sobre el mismo Query Engine.

---

# 277. No direct destructive shortcut

Una API como:

```php
Customer::purgeExpired();
```

si existe, deberá seguir:

```text
Policy Resolution
→ Evaluation
→ Safety
→ Planning
→ Resource Governance
→ Execution
```

y no:

```text
DELETE WHERE deleted_at < ...
```

directamente.

---

# 278. Decisión arquitectónica final

VoltStack implementará Data Retention como una **capa explícita de gobierno del ciclo de vida de los datos**, basada en políticas declarativas, anchors temporales, evidencia verificable y decisiones conservadoras.

El sistema seguirá:

```text
Data
 ↓
Classification
 ↓
Policy Resolution
 ↓
Anchor Resolution
 ↓
Retention Evaluation
 ↓
Hold / Archive / Security Evaluation
 ↓
Purge Eligibility
 ↓
Purge Planning
 ↓
Bounded Execution
 ↓
Transaction
 ↓
Audit / Events / Telemetry
```

El principio rector será:

> **Que un dato sea antiguo, esté expirado o haya sido soft-deleted no significa que pueda destruirse. VoltStack sólo permitirá una operación de purga cuando la política aplicable, su anchor temporal, los Legal Holds, los requisitos de archivo, la seguridad, las relaciones y la evidencia disponible demuestren explícitamente que la destrucción es válida.**

Y de forma complementaria:

> **Cuando VoltStack no pueda demostrar de forma suficiente que un dato puede eliminarse, deberá conservarlo.**

---

# 279. Bloque 27 — progreso

```text
BLOCK 27 — ADVANCED DATABASE CAPABILITIES

✓ 267_DATABASE_TEMPORAL_DATA_SYSTEM.md
✓ 268_DATABASE_HISTORY_AND_VERSIONING_SYSTEM.md
✓ 269_DATABASE_SOFT_DELETE_SYSTEM.md
✓ 270_DATABASE_DATA_RETENTION_SYSTEM.md
□ 271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
□ 272_DATABASE_FULL_TEXT_SEARCH_SYSTEM.md
□ 273_DATABASE_JSON_QUERY_SYSTEM.md
□ 274_DATABASE_GEOGRAPHIC_DATA_EXTENSION_SYSTEM.md
□ 275_DATABASE_DATABASE_FEATURE_CAPABILITY_SYSTEM.md
```

---

# 280. Siguiente documento

```text
271_DATABASE_DATA_ARCHIVAL_SYSTEM.md
```

El siguiente documento definirá:

```text
Archive Policies
Archive Eligibility
Archive Targets
Hot / Warm / Cold Data
Archive Packages
Archive Manifests
Archive Integrity
Archive Verification
Archive Encryption
Archive Compression
Archive Storage Providers
Archive Retrieval
Archive Restore
Archive Rehydration
Archive Search Metadata
Retention + Archive
Soft Delete + Archive
History + Archive
Database → Object Storage Archival
Incremental Archival
Chunked Archival
Distributed Archival
Tenant Archival
Shard Archival
Archive Lifecycle
Archive Expiration
Archive Purging
Audit
Telemetry
Security
Persistent Runtime Safety
```

estableciendo como principio:

> **Archival moverá datos fuera de su almacenamiento operacional primario preservando las garantías necesarias para su conservación y recuperación; no será equivalente a backup, soft delete, purge ni simple exportación de archivos.**