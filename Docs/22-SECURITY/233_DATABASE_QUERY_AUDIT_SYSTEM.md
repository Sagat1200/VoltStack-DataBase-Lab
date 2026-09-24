# 233_DATABASE_QUERY_AUDIT_SYSTEM.md

# VoltStack Quantum Database
## Query Audit System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 233 — Query Audit System  
**Bloque:** 22 — Security  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md`  
**Siguiente documento:** `234_DATABASE_DATABASE_PERMISSION_MODEL.md`

---

# 1. Propósito

Este documento define la arquitectura del sistema de auditoría de operaciones de base de datos de VoltStack.

El sistema deberá generar evidencia estructurada y durable sobre operaciones relevantes ejecutadas a través de:

```text
Query Builder
ORM
Repository
Model API
Persistence Engine
Bulk Operations
Import / Export
Schema
Migrations
Administrative Operations
Raw Query APIs
```

La regla central será:

> **Una auditoría de Database registra evidencia estructurada sobre quién realizó una operación relevante, sobre qué recurso, bajo qué contexto, mediante qué intención semántica y con qué resultado conocido, sin convertir el registro de auditoría en una copia insegura de los datos procesados.**

---

# 2. Audit ≠ Logging

Un application log responde principalmente:

```text
¿Qué ocurrió operacionalmente?
```

Un audit record responde:

```text
¿Quién realizó qué operación,
sobre qué recurso,
bajo qué autoridad,
cuándo,
desde qué contexto,
y cuál fue el resultado conocido?
```

Por tanto:

```text
Audit
≠
Application Log
```

---

# 3. Audit ≠ Telemetry

Telemetry está orientada a:

```text
performance
latency
throughput
errors
health
traces
metrics
profiling
```

Audit está orientado a:

```text
accountability
security evidence
compliance
forensics
privileged activity
data access history
```

Por tanto:

```text
Audit
≠
Telemetry
```

---

# 4. Audit ≠ Debug Information

Debug Information puede desaparecer al terminar una request.

Audit puede necesitar:

```text
durability
retention
integrity
searchability
controlled access
```

---

# 5. Audit ≠ Event System

Un evento:

```text
EntityUpdated
```

puede utilizarse para integración.

Un Audit Record:

```text
Principal P
updated Customer #42
under Tenant T
through operation O
result COMMITTED
```

es evidencia.

Por tanto:

```text
Domain/Event Pipeline
≠
Audit Trail
```

---

# 6. Audit ≠ Transaction Log

El transaction log del motor DB registra información necesaria para recuperación/replicación.

El Audit System registra semántica de aplicación/framework.

```text
Database WAL/Binlog
≠
VoltStack Audit Trail
```

---

# 7. Audit ≠ Query Profiler

Profiler:

```text
query took 325 ms
```

Audit:

```text
principal X exported restricted customer records
```

---

# 8. Audit ≠ Query History

Guardar cada SQL ejecutado no constituye automáticamente auditoría.

---

# 9. Objetivos

El sistema deberá proporcionar:

1. Audit Events estructurados.
2. Audit Records persistibles.
3. identificación de actor.
4. identificación de recurso.
5. operación semántica.
6. tenant context.
7. shard/database context.
8. transaction correlation.
9. request/job correlation.
10. policy-driven auditing.
11. privileged access auditing.
12. sensitive data auditing.
13. mutation auditing.
14. query fingerprinting.
15. redaction.
16. integrity protection.
17. append-oriented storage.
18. failure policies.
19. retention.
20. archival.
21. búsqueda.
22. export control.
23. extensibilidad.
24. runtime isolation.

---

# 10. Arquitectura conceptual

```text
Application Operation
        │
        ▼
Database API
        │
        ▼
Semantic Operation
        │
        ├──────────────► Security Decision
        │
        ▼
Audit Policy Engine
        │
        ▼
Audit Decision
        │
        ├── NONE
        ├── BASIC
        ├── SECURITY
        ├── DETAILED
        └── REQUIRED
        │
        ▼
Audit Event
        │
        ▼
Audit Sanitizer
        │
        ▼
Audit Record
        │
        ▼
Audit Integrity
        │
        ▼
Audit Buffer / Writer
        │
        ▼
Audit Store
```

---

# 11. Audit architecture layers

```text
Audit
├── Metadata
├── Context
├── Policy
├── Capture
├── Sanitization
├── Correlation
├── Integrity
├── Persistence
├── Query
├── Retention
├── Export
└── Telemetry
```

---

# 12. Semantic auditing

VoltStack deberá preferir auditar:

```text
UPDATE Customer
EXPORT Payroll
DELETE Invoice
READ RestrictedRecord
ROTATE SensitiveKey
RUN Migration
```

en lugar de depender exclusivamente de:

```text
UPDATE customers SET ...
SELECT ...
DELETE ...
```

---

# 13. Why semantic audit?

Una sola operación ORM puede generar:

```text
5 INSERT
3 UPDATE
2 DELETE
```

pero semánticamente representar:

```text
ApprovePurchaseOrder
```

La auditoría podrá correlacionar ambos niveles.

---

# 14. Audit granularity

VoltStack deberá soportar:

```text
OPERATION
QUERY
ENTITY
RESOURCE
FIELD
TRANSACTION
BULK_OPERATION
ADMINISTRATIVE
```

---

# 15. AuditLevel

```php
enum AuditLevel
{
    case NONE;
    case BASIC;
    case SECURITY;
    case DETAILED;
    case REQUIRED;
}
```

---

# 16. NONE

No genera audit record.

Solo válido cuando la policy lo permite.

---

# 17. BASIC

Registra información esencial:

```text
actor
operation
resource type
timestamp
outcome
correlation
```

---

# 18. SECURITY

Añade:

```text
authorization decision
security context
tenant
sensitivity
privilege level
```

---

# 19. DETAILED

Puede añadir metadata estructural adicional.

No implica guardar valores sensibles.

---

# 20. REQUIRED

La operación no deberá considerarse aceptable si no puede satisfacerse el contrato de auditoría requerido.

---

# 21. Audit level ≠ data verbosity

`REQUIRED` no significa:

```text
dump everything
```

Puede ser obligatorio registrar un conjunto pequeño pero seguro de evidencia.

---

# 22. AuditPolicy

```php
interface AuditPolicy
{
    public function evaluate(
        AuditCandidate $candidate,
        AuditContext $context,
    ): AuditDecision;
}
```

---

# 23. AuditCandidate

Representa una operación candidata a auditoría.

```php
final readonly class AuditCandidate
{
    public function __construct(
        public AuditOperation $operation,
        public AuditSubject $subject,
        public AuditSensitivity $sensitivity,
        public AuditOperationMetadata $metadata,
    ) {}
}
```

---

# 24. AuditDecision

```php
final readonly class AuditDecision
{
    public function __construct(
        public AuditLevel $level,
        public bool $required,
        public AuditCapturePolicy $capture,
        public AuditFailurePolicy $failure,
        public AuditRetentionPolicyId $retention,
    ) {}
}
```

---

# 25. Policy sources

Podrán provenir de:

```text
framework defaults
entity metadata
field metadata
security policies
application configuration
tenant policy
operation metadata
compliance extension
```

---

# 26. Policy precedence

La combinación deberá ser determinista.

Una policy menos estricta no podrá reducir silenciosamente una requirement superior.

Conceptualmente:

```text
EffectiveAuditRequirement
=
Merge(
    Framework,
    Resource,
    Security,
    Tenant,
    Application
)
```

---

# 27. AuditContext

```php
final readonly class AuditContext
{
    public function __construct(
        public AuditActor $actor,
        public ExecutionContextId $execution,
        public ?RequestId $request,
        public ?JobId $job,
        public ?TransactionId $transaction,
        public ?TenantId $tenant,
        public DatabaseDomainId $database,
        public ?ShardId $shard,
        public AuditContextMetadata $metadata,
    ) {}
}
```

---

# 28. AuditActor

El actor no deberá reducirse siempre a `User`.

Puede ser:

```text
USER
SERVICE
WORKER
JOB
SYSTEM
ADMINISTRATOR
API_CLIENT
AUTOMATION
UNKNOWN
```

---

# 29. AuditActorType

```php
enum AuditActorType
{
    case USER;
    case SERVICE;
    case WORKER;
    case JOB;
    case SYSTEM;
    case ADMINISTRATOR;
    case API_CLIENT;
    case UNKNOWN;
}
```

---

# 30. Unknown actor

```text
UNKNOWN
```

deberá representarse explícitamente.

Nunca fabricarse:

```text
SYSTEM
```

cuando realmente se desconoce.

---

# 31. Actor identity

```php
final readonly class AuditActor
{
    public function __construct(
        public AuditActorType $type,
        public ?AuditActorId $id,
        public ?string $displayReference,
        public AuditAuthorityContext $authority,
    ) {}
}
```

---

# 32. Actor display reference

No deberá contener información sensible innecesaria.

Preferir:

```text
usr_9F82
```

sobre:

```text
john.smith@example.com
```

cuando la identidad interna sea suficiente.

---

# 33. Authentication ≠ Audit identity

Authentication determina identidad/autenticidad.

Audit captura la evidencia relevante de esa identidad.

---

# 34. Authorization ≠ Audit

Authorization decide:

```text
ALLOW / DENY
```

Audit puede registrar esa decisión.

---

# 35. Denied operations

También pueden requerir auditoría.

Ejemplo:

```text
Admin attempted unauthorized payroll export
→ DENIED
→ AUDITED
```

---

# 36. AuditOutcome

```php
enum AuditOutcome
{
    case ATTEMPTED;
    case ALLOWED;
    case DENIED;
    case EXECUTED;
    case COMMITTED;
    case ROLLED_BACK;
    case FAILED;
    case CANCELLED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 37. ATTEMPTED ≠ EXECUTED

Una operación puede ser solicitada y fallar antes de ejecución.

---

# 38. EXECUTED ≠ COMMITTED

Una sentencia ejecutada dentro de una transacción puede posteriormente hacer rollback.

---

# 39. UNKNOWN

Crítico:

```text
commit sent
+
connection lost
```

puede producir:

```text
AuditOutcome::UNKNOWN
```

---

# 40. UNKNOWN ≠ FAILED

VoltStack nunca inventará:

```text
FAILED
```

si el resultado físico es incierto.

---

# 41. AuditSubject

Representa el objeto lógico afectado.

```php
interface AuditSubject
{
    public function type(): AuditSubjectType;

    public function reference(): AuditSubjectReference;
}
```

---

# 42. Subject types

```text
DATABASE
SCHEMA
TABLE
ENTITY
ENTITY_COLLECTION
FIELD
QUERY
TRANSACTION
MIGRATION
IMPORT
EXPORT
BACKUP
ADMIN_OPERATION
CUSTOM
```

---

# 43. Resource identity

Ejemplo:

```text
type: ENTITY
entity: Customer
identifier: 42
```

---

# 44. Sensitive identifiers

Los identificadores también pueden ser sensibles.

Por tanto, la representación de `AuditSubjectReference` será policy-aware.

---

# 45. Subject fingerprint

Podrá almacenarse:

```text
opaque/fingerprinted subject identity
```

cuando el ID real no deba aparecer.

---

# 46. AuditOperation

```php
enum AuditOperation
{
    case READ;
    case CREATE;
    case UPDATE;
    case DELETE;
    case UPSERT;

    case BULK_INSERT;
    case BULK_UPDATE;
    case BULK_DELETE;

    case IMPORT;
    case EXPORT;

    case SCHEMA_CHANGE;
    case MIGRATION;

    case TRANSACTION_BEGIN;
    case TRANSACTION_COMMIT;
    case TRANSACTION_ROLLBACK;

    case PRIVILEGED_READ;
    case PRIVILEGED_MUTATION;

    case CUSTOM;
}
```

---

# 47. Query operation ≠ SQL verb

Un `SELECT` puede semánticamente representar:

```text
READ
EXPORT
REPORT
SECURITY_INSPECTION
```

Por ello:

```text
SQL verb
≠
AuditOperation
```

---

# 48. QueryAuditDescriptor

```php
final readonly class QueryAuditDescriptor
{
    public function __construct(
        public QueryFingerprint $fingerprint,
        public QueryOperationType $type,
        public array $resources,
        public array $sensitivity,
        public QueryAuditShape $shape,
    ) {}
}
```

---

# 49. Query fingerprint

VoltStack reutilizará el concepto de semantic query fingerprint.

---

# 50. Semantic fingerprint ≠ SQL hash

Preferir:

```text
Query Model
→ normalize
→ semantic fingerprint
```

en lugar de:

```text
hash(rendered SQL string)
```

---

# 51. Why?

SQL puede variar por:

```text
platform
whitespace
aliases
compiler
placeholder syntax
```

sin cambiar intención semántica.

---

# 52. Fingerprint privacy

El fingerprint no deberá permitir reconstruir valores sensibles.

---

# 53. Query parameters

Por default, Audit no almacenará plaintext de query parameters.

---

# 54. Parameter audit representation

Ejemplo:

```text
parameter:
    name: customer_email
    type: string
    classification: CONFIDENTIAL
    value: [REDACTED]
```

---

# 55. Query shape

Podrá registrar:

```text
SELECT
resources = [Customer]
predicates = 3
joins = 2
limit = 100
```

sin registrar valores.

---

# 56. Query metadata

Información segura potencial:

```text
query fingerprint
operation type
resource types
row estimate
affected row count
execution result
database domain
shard
```

---

# 57. SQL text

Guardar SQL completo será policy-dependent.

---

# 58. Default SQL audit

Preferir:

```text
normalized structural SQL
+
placeholders
```

sobre SQL interpolado.

---

# 59. Raw SQL auditing

Raw SQL deberá auditarse con controles adicionales.

---

# 60. Raw SQL fingerprint

Podrá producirse mediante:

```text
normalized raw statement
+
safe fingerprint
```

sin interpolar bindings.

---

# 61. SensitiveData integration

El sistema deberá integrar:

```text
232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM
```

---

# 62. Audit records are sensitive

El propio audit trail puede contener:

```text
who accessed what
when
tenant
resource identity
security decisions
```

y por tanto será información sensible.

---

# 63. Audit classification

Los Audit Records tendrán clasificación propia.

---

# 64. Audit data ≠ source data

Aunque los valores fuente estén redacted, metadata de acceso puede ser sensible.

---

# 65. AuditSanitizer

```php
interface AuditSanitizer
{
    public function sanitize(
        AuditEvent $event,
        AuditSanitizationContext $context,
    ): SanitizedAuditEvent;
}
```

---

# 66. Sanitization

Podrá aplicar:

```text
REDACT
MASK
FINGERPRINT
DROP
GENERALIZE
```

---

# 67. Generalization

Ejemplo:

En vez de:

```text
Customer #918273
```

puede registrarse:

```text
Customer
```

si policy no requiere identidad individual.

---

# 68. Field audit

Para updates podrá registrarse:

```text
fields_changed:
- email
- status
```

sin valores.

---

# 69. Before/after values

No serán default.

---

# 70. ChangeSet audit

Ejemplo seguro:

```text
Entity: Customer
Operation: UPDATE
Fields:
    email:
        changed: true
        classification: CONFIDENTIAL
        values: REDACTED

    status:
        old: pending
        new: active
```

si `status` está autorizado por policy para captura.

---

# 71. CapturePolicy

```php
enum AuditValueCapturePolicy
{
    case NONE;
    case METADATA_ONLY;
    case SAFE_VALUES;
    case MASKED_VALUES;
    case PROTECTED_VALUES;
}
```

---

# 72. Full plaintext capture

No deberá existir como default general.

---

# 73. Protected audit values

En casos específicos podrá almacenarse valor cifrado bajo una policy de audit separada.

---

# 74. Audit encryption key

Puede ser distinta de la key usada para los datos principales.

---

# 75. Why separate keys?

Permite:

```text
separate access
separate rotation
separate retention
reduced privilege coupling
```

---

# 76. AuditEvent

Representa el evento interno previo a persistencia.

```php
final readonly class AuditEvent
{
    public function __construct(
        public AuditEventId $id,
        public AuditTimestamp $timestamp,
        public AuditActor $actor,
        public AuditOperation $operation,
        public AuditSubject $subject,
        public AuditOutcome $outcome,
        public AuditContext $context,
        public AuditPayload $payload,
    ) {}
}
```

---

# 77. AuditEvent ≠ AuditRecord

```text
AuditEvent
→ internal transient representation

AuditRecord
→ sanitized/persistable evidence
```

---

# 78. AuditRecord

```php
final readonly class AuditRecord
{
    public function __construct(
        public AuditRecordId $id,
        public AuditTimestamp $timestamp,
        public AuditActorReference $actor,
        public AuditOperation $operation,
        public AuditSubjectReference $subject,
        public AuditOutcome $outcome,
        public AuditCorrelation $correlation,
        public SanitizedAuditPayload $payload,
        public AuditIntegrityMetadata $integrity,
    ) {}
}
```

---

# 79. Audit IDs

Deberán ser:

```text
unique
stable
opaque
non-semantic
```

---

# 80. Timestamp

Audit deberá distinguir cuando sea útil:

```text
attempted_at
executed_at
completed_at
committed_at
recorded_at
```

---

# 81. One timestamp ≠ full lifecycle

Una operación transaccional puede durar segundos o minutos.

---

# 82. AuditCorrelation

```php
final readonly class AuditCorrelation
{
    public function __construct(
        public ExecutionContextId $execution,
        public ?RequestId $request,
        public ?JobId $job,
        public ?TransactionId $transaction,
        public ?TraceId $trace,
        public ?BulkOperationId $bulkOperation,
    ) {}
}
```

---

# 83. Correlation ≠ causation

Compartir TraceId no demuestra por sí solo causalidad.

---

# 84. Causation ID

Podrá existir:

```text
AuditEventId causedBy
```

para relaciones explícitas.

---

# 85. Parent operation

Ejemplo:

```text
ExportOperation #A
    ├── Query #B
    ├── Query #C
    └── Query #D
```

---

# 86. Hierarchical auditing

Permitirá registrar:

```text
high-level export
+
low-level database activity
```

sin perder relación.

---

# 87. Avoid audit explosion

No deberá registrarse automáticamente un record por cada fila de un bulk operation.

---

# 88. Bulk audit

Preferir:

```text
BulkUpdate
resource: Customer
affected: 1,248
query fingerprint: ...
result: COMMITTED
```

---

# 89. Per-record audit

Solo cuando policy lo requiera.

---

# 90. Bulk thresholds

Policy podrá establecer:

```text
aggregate audit
per-record audit
sampled details
hybrid
```

---

# 91. Sampling

Sampling no será permitido cuando la policy requiera evidencia completa.

---

# 92. Audit completeness

Deberá expresarse:

```php
enum AuditCompleteness
{
    case COMPLETE;
    case AGGREGATED;
    case SAMPLED;
    case PARTIAL;
    case UNKNOWN;
}
```

---

# 93. UNKNOWN ≠ COMPLETE

Invariante crítica.

---

# 94. ORM audit

Integración:

```text
EntityManager
    ↓
UnitOfWork
    ↓
ChangeSet
    ↓
Persistence Planner
    ↓
Audit Capture
```

---

# 95. ORM semantic advantage

El ORM conoce:

```text
entity type
identifier
changed fields
relationship changes
operation intent
```

que SQL aislado puede no conocer.

---

# 96. `persist()` auditing

`persist()` no implica INSERT inmediato.

Por tanto:

```text
persist()
≠
successful mutation audit
```

---

# 97. `flush()` auditing

`flush()` ejecuta sincronización.

Pero:

```text
flush()
≠
commit()
```

---

# 98. Transaction-aware audit

Audit deberá seguir el resultado real de la transacción.

---

# 99. Audit lifecycle

```text
ATTEMPT
   ↓
AUTHORIZED
   ↓
EXECUTION
   ↓
STATEMENT SUCCESS
   ↓
TRANSACTION OUTCOME
   ├── COMMITTED
   ├── ROLLED_BACK
   └── UNKNOWN
```

---

# 100. Mutation audit stages

Podrán existir:

```text
AuditIntent
AuditExecutionEvidence
AuditTransactionEvidence
AuditFinalRecord
```

---

# 101. Before execution audit

Para operaciones de alto riesgo puede ser necesario registrar el intento antes de ejecutar.

---

# 102. After execution audit

Permite registrar:

```text
affected rows
execution outcome
```

---

# 103. After commit audit

Permite afirmar:

```text
COMMITTED
```

cuando existe evidencia.

---

# 104. Rollback

Una operación ejecutada pero revertida podrá conservar evidencia:

```text
operation = UPDATE
execution = EXECUTED
transaction = ROLLED_BACK
```

---

# 105. Do not erase rollback audit

Rollback de negocio no deberá borrar necesariamente evidencia de que la operación fue intentada.

---

# 106. Transaction-owned audit storage

Si el Audit Record se escribe dentro de la misma transacción:

```text
business rollback
→ audit rollback
```

Esto puede ser indeseable para security audit.

---

# 107. Independent audit durability

Para ciertos eventos se necesita almacenamiento independiente.

---

# 108. Same-transaction audit

Puede ser útil para:

```text
business-consistent change history
```

pero no para toda auditoría de seguridad.

---

# 109. Two audit channels

VoltStack deberá poder distinguir:

```text
TRANSACTIONAL_AUDIT
SECURITY_AUDIT
```

---

# 110. Transactional audit

Comparte destino transaccional cuando se desea atomicidad con business mutation.

---

# 111. Security audit

Busca preservar evidencia incluso ante:

```text
rollback
denied operation
failed attempt
```

---

# 112. AuditDurabilityMode

```php
enum AuditDurabilityMode
{
    case TRANSACTIONAL;
    case INDEPENDENT;
    case DEFERRED_DURABLE;
    case EXTERNAL;
}
```

---

# 113. Independent writer

Debe evitar contaminar la transacción de negocio.

---

# 114. Independent writer ≠ arbitrary second connection

La arquitectura deberá resolver explícitamente:

```text
audit destination
connection
credentials
failure policy
tenant policy
```

---

# 115. Audit destination

Puede ser:

```text
same database
separate database
append-only store
external audit service
security event system
```

---

# 116. AuditStore

```php
interface AuditStore
{
    public function append(AuditRecord $record): AuditAppendResult;

    public function appendBatch(array $records): AuditBatchAppendResult;
}
```

---

# 117. Append semantics

El modelo será append-oriented.

---

# 118. Append-oriented ≠ physically immutable

Una DB normal puede permitir UPDATE/DELETE.

Por eso deben existir controles adicionales.

---

# 119. Audit immutability

Objetivo:

```text
normal application role
→ INSERT audit
→ no UPDATE
→ no DELETE
```

cuando el backend lo permita.

---

# 120. Privilege separation

Audit writer deberá usar privilegios mínimos.

---

# 121. Audit reader

Puede tener credenciales separadas.

---

# 122. Audit administrator

También.

---

# 123. AuditIntegrity

VoltStack deberá soportar mecanismos para detectar modificación.

---

# 124. Record hash

Puede existir:

```text
H(record canonical representation)
```

pero un hash simple no evita reescritura maliciosa completa.

---

# 125. Keyed integrity

Podrá utilizarse:

```text
MAC
digital signature
external integrity provider
```

según policy.

---

# 126. Hash chaining

Opcionalmente:

```text
R1 → H1
R2 → H(R2 || H1)
R3 → H(R3 || H2)
```

---

# 127. Hash chain benefit

Puede detectar:

```text
modification
reordering
removal
```

dentro de ciertos modelos.

---

# 128. Hash chain limitations

No es sustituto automático de:

```text
external anchoring
access control
backup
independent verification
```

---

# 129. AuditIntegrityMetadata

```php
final readonly class AuditIntegrityMetadata
{
    public function __construct(
        public AuditIntegritySchemeId $scheme,
        public AuditIntegrityVersion $version,
        public ?string $previousDigest,
        public ?string $digest,
        public ?KeyReference $key,
    ) {}
}
```

---

# 130. Integrity key ≠ encryption key

Separación recomendada.

---

# 131. Canonical serialization

Integrity requiere representación determinista.

---

# 132. Canonical audit record

Campos deberán serializarse en orden/formato definido.

---

# 133. Schema evolution

Audit records tendrán:

```text
AuditSchemaVersion
```

---

# 134. AuditSchemaVersion ≠ FrameworkVersion

---

# 135. Audit schema compatibility

Readers deberán soportar versiones históricas bajo política de compatibilidad.

---

# 136. Record mutation

Si se requiere corregir metadata:

```text
original record
+
correction record
```

preferible a sobrescribir.

---

# 137. Correction event

```text
AuditCorrection
references original AuditRecordId
```

---

# 138. Deletion requests

Retención y privacidad pueden exigir eliminación.

Por tanto:

```text
Audit Immutability
≠
Infinite Retention
```

---

# 139. Retention

Cada record podrá asociarse a:

```text
AuditRetentionPolicyId
```

---

# 140. RetentionPolicy

Puede determinar:

```text
retention duration
archive destination
legal hold
deletion eligibility
encryption requirements
```

---

# 141. Legal hold integration

Será extensible.

Database core no hardcodeará legislación específica.

---

# 142. Archive

Records antiguos podrán migrarse a storage de menor costo.

---

# 143. Archive integrity

La integridad deberá preservarse durante archival.

---

# 144. Archive encryption

También.

---

# 145. Audit search

Debe soportar búsquedas por metadata segura.

Ejemplos:

```text
actor
operation
resource type
date range
tenant
outcome
transaction
request
query fingerprint
```

---

# 146. Search access control

No todo usuario autorizado a ejecutar queries deberá poder consultar audit logs.

---

# 147. Audit of audit access

Acceder a registros de auditoría puede requerir auditoría.

---

# 148. Recursive auditing

Debe evitar loops infinitos.

---

# 149. Meta-audit

Operaciones sobre Audit Store podrán usar un canal administrativo separado.

---

# 150. QueryAuditRecorder

```php
interface QueryAuditRecorder
{
    public function recordAttempt(QueryAuditAttempt $attempt): void;

    public function recordExecution(QueryAuditExecution $execution): void;

    public function recordOutcome(QueryAuditOutcome $outcome): void;
}
```

---

# 151. Recorder ≠ Store

Recorder coordina.

Store persiste.

---

# 152. AuditAssembler

Convierte evidencia parcial en records.

---

# 153. Audit state machine

```text
PLANNED
   ↓
ATTEMPTED
   ↓
AUTHORIZED / DENIED
   ↓
EXECUTING
   ↓
EXECUTED
   ↓
AWAITING_TRANSACTION
   ↓
COMMITTED
   or
ROLLED_BACK
   or
UNKNOWN
```

---

# 154. Non-transactional query

Puede terminar:

```text
EXECUTED
```

sin etapa explícita de transaction commit.

---

# 155. Autocommit

No deberá confundirse con ausencia total de transacción física del motor.

El audit utilizará la evidencia expuesta por Execution/Connection layer.

---

# 156. Read auditing

No todos los SELECT deberán auditarse por default.

Eso sería costoso.

---

# 157. Read audit policies

Ejemplos:

```text
ordinary public read
→ NONE

confidential customer read
→ BASIC

restricted payroll read
→ SECURITY

secret material reveal
→ REQUIRED
```

---

# 158. Read volume

Para reads masivos podrá registrarse:

```text
query
resource
count
scope
```

en vez de cada fila.

---

# 159. Read result count

Podrá ser:

```text
exact
estimated
unknown
```

---

# 160. Unknown count ≠ zero

---

# 161. Sensitive field read

Puede requerir auditoría incluso cuando la Entity completa no sea clasificada como altamente sensible.

---

# 162. Projection audit

Query:

```text
SELECT name, salary
```

deberá reconocer que incluye `salary`.

---

# 163. Sensitivity lineage

Reutilizará metadata del documento 232.

---

# 164. Derived expressions

Una expresión derivada de información sensible puede elevar audit requirement.

---

# 165. Export audit

Todo export sensible deberá poder generar:

```text
actor
export type
resource scope
record count
destination class
protection mode
outcome
```

---

# 166. Export destination

No registrar credenciales ni signed URLs completas.

---

# 167. Import audit

Podrá registrar:

```text
source type
rows attempted
rows accepted
rows rejected
resource type
outcome
```

---

# 168. File path sensitivity

Paths pueden contener datos sensibles.

Sanitización requerida.

---

# 169. Bulk mutation audit

```text
BULK_UPDATE
query fingerprint
resource
affected rows
transaction outcome
```

---

# 170. Bulk delete

Deberá ser auditado bajo policy más estricta cuando corresponda.

---

# 171. Destructive operations

Ejemplos:

```text
TRUNCATE
DROP
bulk delete
schema destructive migration
```

podrán ser `REQUIRED`.

---

# 172. Schema audit

Registrar:

```text
migration
schema operation
actor
database
outcome
```

---

# 173. Migration audit

Debe integrarse con Migration Execution System.

---

# 174. Migration SQL

No necesariamente debe almacenarse completo si contiene sensitive literals.

---

# 175. Backup audit

Podrá registrar:

```text
backup started
backup completed
destination class
encryption policy
actor
```

---

# 176. Restore audit

Especialmente importante.

---

# 177. Administrative operations

Ejemplos:

```text
replica promotion
failover
shard move
maintenance
credential rotation
key rotation
```

podrán producir Audit Records.

---

# 178. Query denial audit

Data Access Security puede generar:

```text
DENIED
```

antes de Execution Engine.

---

# 179. SQL injection prevention audit

Intentos rechazados por APIs inseguras podrán producir security audit cuando policy lo requiera.

---

# 180. Input security audit

No deberá copiar input malicioso completo si puede contener secretos o payloads enormes.

---

# 181. Bounded payload

Todos los Audit Records tendrán límites.

---

# 182. Limits

Ejemplos:

```text
max metadata fields
max string length
max nesting depth
max resource references
max changed fields
```

---

# 183. Oversized payload

Deberá truncarse/summarize según policy.

---

# 184. Truncation marker

Nunca ocultar que ocurrió truncation.

```text
payload_truncated = true
```

---

# 185. AuditCompleteness

Puede pasar a:

```text
PARTIAL
```

si evidencia requerida no pudo capturarse completamente y policy permite continuar.

---

# 186. Required evidence failure

Si policy exige COMPLETE:

```text
capture failure
→ operation denied/failed
```

según etapa.

---

# 187. AuditFailurePolicy

```php
enum AuditFailurePolicy
{
    case FAIL_OPERATION;
    case FAIL_BEFORE_EXECUTION;
    case BUFFER_AND_CONTINUE;
    case FALLBACK_STORE;
    case CONTINUE_WITH_ALERT;
}
```

---

# 188. Dangerous default

Para operaciones críticas:

```text
audit unavailable
→ silently continue
```

no será default.

---

# 189. Failure timing

No siempre es posible revertir una operación ya committed porque el Audit Store falló después.

---

# 190. After-commit audit failure

Si:

```text
business commit = SUCCESS
audit append = FAILURE
```

VoltStack no deberá afirmar que el negocio hizo rollback.

---

# 191. Outcome representation

Debe registrar/emitir:

```text
business outcome = COMMITTED
audit outcome = FAILED
```

como dos hechos distintos.

---

# 192. AuditFailure ≠ BusinessRollback

Invariante crítica.

---

# 193. Preflight audit

Para `REQUIRED` podrá verificarse antes:

```text
Audit Store available?
Policy resolvable?
Integrity provider available?
```

---

# 194. Preflight ≠ guaranteed append

El store puede fallar después.

---

# 195. Durable buffering

Podrá existir:

```text
Audit Outbox / Durable Buffer
```

---

# 196. In-memory buffer

No es durable.

---

# 197. Persistent runtime buffer

Un buffer worker-local no deberá confundirse con garantía de auditoría.

---

# 198. Deferred audit

Puede utilizar:

```text
business transaction
→ durable audit intent
→ asynchronous audit delivery
```

---

# 199. Audit outbox

Cuando se requiere atomicidad entre business change y audit intent:

```text
Business Mutation
+
Audit Intent
↓
same transaction
```

Después:

```text
Audit Intent
→ Audit Store
```

---

# 200. Outbox tradeoff

Si business transaction hace rollback, el intent también desaparece.

Por eso denied/failed attempts necesitan canal distinto.

---

# 201. Hybrid model

VoltStack podrá usar:

```text
Security Attempt Audit
→ independent

Committed Mutation Audit
→ transactional outbox
```

---

# 202. Transaction correlation

Todos los records asociados deberán compartir:

```text
TransactionId
```

cuando exista.

---

# 203. Nested transactions

Logical nested scopes podrán tener:

```text
TransactionScopeId
```

sin fingir transacciones físicas independientes.

---

# 204. Savepoints

Rollback a savepoint podrá reflejarse.

---

# 205. Savepoint audit

No es obligatorio para todas las apps.

Policy-driven.

---

# 206. Retry

Transaction retry puede ejecutar varias veces la misma operación lógica.

---

# 207. Retry correlation

Debe distinguir:

```text
LogicalOperationId
AttemptNumber
TransactionAttemptId
```

---

# 208. Example

```text
Operation: TransferFunds
LogicalOperationId: op_A

Attempt 1:
deadlock
rolled back

Attempt 2:
committed
```

---

# 209. Do not report duplicate business success

El audit deberá permitir reconstruir que hubo dos attempts pero un solo resultado lógico exitoso.

---

# 210. UNKNOWN commit

No deberá ser reetiquetado como:

```text
ROLLED_BACK
```

---

# 211. Distributed operations

Una operación puede tocar varios shards.

---

# 212. Distributed audit

Deberá poder representar:

```text
operation
├── shard A → committed
├── shard B → committed
└── shard C → unknown
```

---

# 213. Partial distributed outcome

Resultado:

```text
PARTIAL
```

o:

```text
UNKNOWN
```

según evidencia.

---

# 214. No fake global transaction

Audit no deberá afirmar atomicidad distribuida inexistente.

---

# 215. Shard identity

Será metadata estructurada.

---

# 216. Topology generation

Cuando sea relevante podrá capturarse:

```text
ShardMapGeneration
AuthorityEpoch
```

---

# 217. Replica reads

Audit podrá registrar:

```text
read role = replica
endpoint class
consistency policy
```

sin necesidad de guardar secretos de conexión.

---

# 218. Endpoint identity

Debe utilizar identificador seguro.

No DSN completo.

---

# 219. Credentials

Nunca deberán aparecer en Audit Records.

---

# 220. Connection strings

Nunca completos si contienen secretos.

---

# 221. Tenant audit

TenantId formará parte del AuditContext cuando exista.

---

# 222. Cross-tenant operations

Deberán tener auditoría reforzada.

---

# 223. Tenant impersonation

Si un administrador actúa:

```text
Admin
on behalf of
Tenant
```

ambas identidades deberán preservarse.

---

# 224. Effective actor

Podrán distinguirse:

```text
authenticated_actor
effective_actor
delegated_tenant
```

---

# 225. Impersonation ≠ identity replacement

No sobrescribir actor original.

---

# 226. Delegation chain

Podrá modelarse:

```text
Admin
→ Support Session
→ Tenant
→ Operation
```

---

# 227. Authorization decision evidence

Audit podrá capturar:

```text
policy ID
decision
reason code
policy generation
```

---

# 228. Avoid policy internals

No guardar secrets ni estructuras excesivas del authorization engine.

---

# 229. Permission evidence

El documento 234 definirá el modelo completo.

---

# 230. Audit and caches

Audit records no deberán depender de Result Cache para existir.

---

# 231. Cached reads

Un read servido desde cache puede seguir siendo auditable.

---

# 232. Database query absent

Si:

```text
application read
→ entity cache hit
```

no existe SQL.

Pero puede existir:

```text
Data Access Audit
```

---

# 233. Query Audit scope

Por tanto el sistema deberá distinguir:

```text
QUERY_AUDIT
DATA_ACCESS_AUDIT
```

---

# 234. Query Audit

Registra actividad del Query Engine/Execution.

---

# 235. Data Access Audit

Registra acceso lógico a recursos incluso cuando no se ejecuta SQL.

---

# 236. Cache source metadata

Podrá registrarse:

```text
source = DATABASE
source = RESULT_CACHE
source = ENTITY_CACHE
```

si policy lo requiere.

---

# 237. Lazy Collection

La creación de:

```php
$query->lazy();
```

no implica acceso ejecutado.

---

# 238. Lazy iteration

Audit debe ocurrir al abrir/consumir la fuente según policy.

---

# 239. Chunk processing

No generar millones de audit records por default.

---

# 240. Chunk audit

Puede registrar:

```text
processing operation
chunks processed
rows processed
checkpoint
outcome
```

---

# 241. Pagination

Una página puede auditarse como read operation.

---

# 242. Cursor pagination

No guardar cursor completo si contiene sensitive boundaries.

---

# 243. Cursor fingerprint

Podrá registrarse una identidad derivada segura.

---

# 244. Streaming

Auditoría podrá registrar:

```text
stream opened
records consumed
stream closed
outcome
```

---

# 245. Early termination

Debe distinguirse de failure.

---

# 246. Cancellation

Resultado:

```text
CANCELLED
```

---

# 247. Query timeout

Resultado:

```text
FAILED
```

con reason code seguro:

```text
QUERY_TIMEOUT
```

---

# 248. Error audit

No guardar exception message crudo sin sanitización.

---

# 249. AuditReasonCode

Preferir códigos estables:

```text
ACCESS_DENIED
QUERY_TIMEOUT
DEADLOCK
CONNECTION_FAILURE
AUDIT_STORE_FAILURE
VALIDATION_REJECTED
SECURITY_POLICY_REJECTED
```

---

# 250. Reason code ≠ raw exception

---

# 251. Telemetry integration

Audit System emitirá telemetry sobre sí mismo.

---

# 252. Metrics

Ejemplos:

```text
db.audit.records.created
db.audit.records.failed
db.audit.records.buffered
db.audit.records.dropped
db.audit.integrity.failures
db.audit.store.latency
db.audit.required.failures
```

---

# 253. Cardinality

Labels seguras:

```text
operation
outcome
audit_level
store
failure_type
```

---

# 254. Forbidden metric labels

Evitar:

```text
user ID
tenant ID
record ID
email
query parameters
SQL
```

como labels de alta cardinalidad.

---

# 255. Audit telemetry ≠ audit trail

Metrics sobre Audit System no reemplazan Audit Records.

---

# 256. Debugging

Debug Toolbar podrá indicar:

```text
Audit:
required = yes
recorded = yes
record_id = aud_xxx
```

si policy permite mostrar ID.

---

# 257. Debug toolbar security

No mostrará contenido sensible del record por default.

---

# 258. Audit query API

Podrá existir:

```php
$audit->query()
    ->actor($actor)
    ->operation(AuditOperation::EXPORT)
    ->between($from, $to)
    ->outcome(AuditOutcome::COMMITTED)
    ->get();
```

---

# 259. Audit query API ≠ Query Builder

Será una API de dominio especializada.

Internamente podrá utilizar Query Engine.

---

# 260. Audit authorization

Consultar Audit Store requerirá policy específica.

---

# 261. Audit export

Exportar audit records es una operación sensible.

---

# 262. Audit export audit

Puede requerir su propio audit record.

---

# 263. Recursion guard

El sistema deberá impedir:

```text
audit export
→ audit
→ audit
→ audit
→ infinite recursion
```

---

# 264. AuditExecutionGuard

Podrá marcar operaciones internas:

```text
AUDIT_INTERNAL
```

sin desactivar meta-auditoría requerida.

---

# 265. Retention jobs

Deberán auditar operaciones destructivas de records cuando policy lo requiera.

---

# 266. Archive jobs

Igualmente.

---

# 267. Persistent runtimes

Crítico para:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 268. Scoped state

Debe ser request/job/operation scoped:

```text
AuditContext
AuditActor
TransactionCorrelation
PendingAuditEvents
SecurityDecisionEvidence
TenantContext
```

---

# 269. No static actor

Prohibido:

```php
Audit::$currentUser
```

---

# 270. No static tenant

Prohibido.

---

# 271. Immutable shared components

Podrán compartirse:

```text
compiled policies
audit schema definitions
sanitization rules
integrity algorithms
```

---

# 272. Pending records

Nunca deberán filtrarse entre requests.

---

# 273. Worker reset

Al terminar operación:

```text
clear pending audit context
clear actor
clear transaction correlation
clear tenant correlation
clear temporary buffers
```

---

# 274. Coroutine isolation

OpenSwoole deberá aislar AuditContext por coroutine/request.

---

# 275. Audit buffer isolation

Un buffer compartido puede ser válido únicamente si sus records ya están completamente materializados, sanitizados y desligados del mutable request context.

---

# 276. Backpressure

Si Audit Store se ralentiza:

```text
block
buffer
fallback
reject
```

dependerá de policy.

---

# 277. No unbounded audit queue

Memoria del worker deberá estar bounded.

---

# 278. Batch writing

Permitido cuando preserve:

```text
record identity
ordering requirements
integrity
failure evidence
```

---

# 279. Batch failure

No asumir que todos los records fallaron o tuvieron éxito si provider devuelve resultado parcial.

---

# 280. AuditAppendResult

```php
final readonly class AuditAppendResult
{
    public function __construct(
        public AuditAppendStatus $status,
        public ?AuditRecordId $record,
        public ?AuditFailureEvidence $failure,
    ) {}
}
```

---

# 281. Append statuses

```text
APPENDED
BUFFERED
DUPLICATE
FAILED
UNKNOWN
```

---

# 282. UNKNOWN append

Debe preservarse.

---

# 283. Idempotency

Delivery retries requieren:

```text
AuditRecordId
+
idempotency semantics
```

---

# 284. Duplicate audit

Store deberá poder detectar record IDs repetidos cuando sea posible.

---

# 285. Duplicate delivery ≠ duplicate business operation

---

# 286. Ordering

Audit no deberá asumir orden global perfecto en sistemas distribuidos.

---

# 287. Per-stream ordering

Puede garantizarse por:

```text
transaction
actor
tenant
audit partition
```

según backend.

---

# 288. Global sequence

Solo si provider realmente la ofrece.

---

# 289. Timestamp ≠ total ordering

Dos records con timestamps similares no establecen necesariamente causalidad.

---

# 290. Sequence metadata

Podrá existir:

```text
AuditSequence
```

cuando backend lo soporte.

---

# 291. Clock

Usará Clock abstraction.

---

# 292. No direct system time scattered

Evitar:

```php
new DateTimeImmutable();
```

en cada componente.

---

# 293. Testing clock

Debe poder fijarse.

---

# 294. Testing architecture

Se requerirán pruebas para:

```text
policy resolution
actor resolution
tenant isolation
query fingerprint
sanitization
sensitive fields
transaction correlation
rollback
unknown commit
retry
bulk operations
distributed outcomes
integrity
retention
append failure
buffering
idempotency
persistent runtimes
```

---

# 295. Audit assertions

Ejemplo:

```php
$this->assertDatabaseAudited(
    operation: AuditOperation::UPDATE,
    subject: Customer::class,
    outcome: AuditOutcome::COMMITTED,
);
```

---

# 296. Negative assertion

```php
$this->assertAuditDoesNotContain(
    'john@example.com'
);
```

---

# 297. FakeAuditStore

Para tests:

```php
final class FakeAuditStore implements AuditStore
{
    // deterministic in-memory records
}
```

---

# 298. Failure injection

Debe permitir simular:

```text
store unavailable
partial batch failure
integrity provider failure
buffer full
unknown append outcome
```

---

# 299. Security tests

Deberán verificar que:

```text
password
credential
secret token
sensitive parameter
connection secret
encryption key
```

nunca aparezcan en records.

---

# 300. Property tests

Ejemplo:

Para cualquier sensitive value `S`:

```text
Sanitize(AuditEvent(S))
```

deberá satisfacer:

```text
S ∉ SerializedAuditRecord
```

cuando policy indique redaction.

---

# 301. Directory structure

```text
src/Quantum/Database/Security/Audit/
│
├── Contract/
│   ├── AuditPolicy.php
│   ├── AuditStore.php
│   ├── AuditSanitizer.php
│   ├── QueryAuditRecorder.php
│   ├── AuditIntegrityProvider.php
│   └── AuditRetentionProvider.php
│
├── Model/
│   ├── AuditEvent.php
│   ├── AuditRecord.php
│   ├── AuditEventId.php
│   ├── AuditRecordId.php
│   ├── AuditOperation.php
│   ├── AuditOutcome.php
│   ├── AuditLevel.php
│   ├── AuditCompleteness.php
│   └── AuditTimestamp.php
│
├── Actor/
│   ├── AuditActor.php
│   ├── AuditActorId.php
│   ├── AuditActorType.php
│   ├── AuditActorResolver.php
│   └── AuditAuthorityContext.php
│
├── Subject/
│   ├── AuditSubject.php
│   ├── AuditSubjectType.php
│   ├── AuditSubjectReference.php
│   ├── AuditSubjectResolver.php
│   └── AuditSubjectFingerprint.php
│
├── Context/
│   ├── AuditContext.php
│   ├── AuditContextResolver.php
│   ├── AuditCorrelation.php
│   ├── AuditCausation.php
│   └── AuditExecutionGuard.php
│
├── Policy/
│   ├── AuditPolicyEngine.php
│   ├── AuditDecision.php
│   ├── AuditCapturePolicy.php
│   ├── AuditValueCapturePolicy.php
│   ├── AuditFailurePolicy.php
│   ├── AuditDurabilityMode.php
│   └── AuditPolicyGeneration.php
│
├── Query/
│   ├── QueryAuditDescriptor.php
│   ├── QueryAuditAttempt.php
│   ├── QueryAuditExecution.php
│   ├── QueryAuditOutcome.php
│   ├── QueryAuditShape.php
│   └── QueryAuditFingerprintResolver.php
│
├── ORM/
│   ├── OrmAuditCapture.php
│   ├── ChangeSetAuditMapper.php
│   ├── EntityAuditSubjectResolver.php
│   └── RelationshipAuditMapper.php
│
├── Transaction/
│   ├── TransactionAuditCoordinator.php
│   ├── TransactionAuditState.php
│   ├── TransactionAuditEvidence.php
│   ├── AuditTransactionAttempt.php
│   └── AuditCommitResolver.php
│
├── Bulk/
│   ├── BulkAuditCoordinator.php
│   ├── BulkAuditSummary.php
│   └── BulkAuditThresholdPolicy.php
│
├── Sanitization/
│   ├── DefaultAuditSanitizer.php
│   ├── AuditRedactor.php
│   ├── AuditMasker.php
│   ├── AuditFingerprinter.php
│   └── AuditPayloadLimiter.php
│
├── Integrity/
│   ├── AuditIntegrityMetadata.php
│   ├── AuditIntegritySchemeId.php
│   ├── AuditIntegrityVersion.php
│   ├── AuditRecordCanonicalizer.php
│   ├── AuditHashChain.php
│   └── AuditIntegrityVerifier.php
│
├── Persistence/
│   ├── AuditWriter.php
│   ├── AuditBatchWriter.php
│   ├── AuditAppendResult.php
│   ├── AuditAppendStatus.php
│   ├── AuditBuffer.php
│   ├── DurableAuditBuffer.php
│   └── AuditOutbox.php
│
├── Search/
│   ├── AuditQuery.php
│   ├── AuditQueryBuilder.php
│   ├── AuditSearchCriteria.php
│   └── AuditSearchResult.php
│
├── Retention/
│   ├── AuditRetentionPolicy.php
│   ├── AuditRetentionPolicyId.php
│   ├── AuditRetentionPlanner.php
│   ├── AuditArchivePlanner.php
│   └── AuditLegalHold.php
│
├── Export/
│   ├── AuditExportPolicy.php
│   ├── AuditExporter.php
│   └── AuditExportSanitizer.php
│
├── Telemetry/
│   └── AuditTelemetry.php
│
├── Testing/
│   ├── FakeAuditStore.php
│   ├── AuditAssertions.php
│   ├── AuditRecordFactory.php
│   └── AuditConformanceSuite.php
│
└── Exception/
    ├── AuditException.php
    ├── AuditPolicyException.php
    ├── AuditCaptureException.php
    ├── AuditSanitizationException.php
    ├── AuditIntegrityException.php
    ├── AuditStoreException.php
    ├── AuditRequiredException.php
    ├── AuditBufferException.php
    └── AuditQueryException.php
```

---

# 302. Architectural invariants

## DB-AUDIT-001
Audit será distinto de Logging.

## DB-AUDIT-002
Audit será distinto de Telemetry.

## DB-AUDIT-003
Audit será distinto de Debug Information.

## DB-AUDIT-004
Audit será distinto de Event System.

## DB-AUDIT-005
Audit será distinto del transaction log del motor.

## DB-AUDIT-006
Audit será distinto de Query Profiler.

## DB-AUDIT-007
Guardar SQL no constituirá por sí solo auditoría.

## DB-AUDIT-008
Audit preferirá semántica de operación.

## DB-AUDIT-009
Query-level auditing seguirá disponible.

## DB-AUDIT-010
AuditPolicy determinará cuándo auditar.

## DB-AUDIT-011
REQUIRED no significará maximum verbosity.

## DB-AUDIT-012
Policies se combinarán determinísticamente.

## DB-AUDIT-013
Una policy débil no reducirá silenciosamente una requirement superior.

## DB-AUDIT-014
AuditActor no será equivalente a User.

## DB-AUDIT-015
UNKNOWN actor será explícito.

## DB-AUDIT-016
UNKNOWN actor no será convertido en SYSTEM.

## DB-AUDIT-017
Authentication será distinta de Audit identity.

## DB-AUDIT-018
Authorization será distinta de Audit.

## DB-AUDIT-019
Denied operations podrán ser auditadas.

## DB-AUDIT-020
ATTEMPTED será distinto de EXECUTED.

## DB-AUDIT-021
EXECUTED será distinto de COMMITTED.

## DB-AUDIT-022
UNKNOWN será distinto de FAILED.

## DB-AUDIT-023
Unknown transaction outcome permanecerá UNKNOWN.

## DB-AUDIT-024
AuditSubject será lógico/semántico.

## DB-AUDIT-025
Subject identifiers podrán ser protegidos.

## DB-AUDIT-026
SQL verb será distinto de AuditOperation.

## DB-AUDIT-027
Query fingerprint será semántico cuando sea posible.

## DB-AUDIT-028
Semantic fingerprint será distinto de SQL hash.

## DB-AUDIT-029
Fingerprint no deberá revelar query parameters.

## DB-AUDIT-030
Query parameters sensibles estarán redacted por default.

## DB-AUDIT-031
SQL interpolado no será audit default.

## DB-AUDIT-032
Raw SQL conservará bindings separados.

## DB-AUDIT-033
Audit records serán tratados como información sensible.

## DB-AUDIT-034
Audit data será distinta de source data.

## DB-AUDIT-035
AuditSanitizer ejecutará antes de persistencia externa.

## DB-AUDIT-036
Before/after values no serán default.

## DB-AUDIT-037
Sensitive values respetarán documento 232.

## DB-AUDIT-038
Full plaintext capture no será general default.

## DB-AUDIT-039
Protected audit values podrán usar keys independientes.

## DB-AUDIT-040
AuditEvent será distinto de AuditRecord.

## DB-AUDIT-041
AuditRecord será sanitized.

## DB-AUDIT-042
AuditRecord IDs serán opacos.

## DB-AUDIT-043
Audit lifecycle podrá tener múltiples timestamps.

## DB-AUDIT-044
Correlation será distinta de causation.

## DB-AUDIT-045
Causation será explícita.

## DB-AUDIT-046
Hierarchical auditing será soportado.

## DB-AUDIT-047
Bulk operations no producirán per-row audit por default.

## DB-AUDIT-048
Sampling no satisfará COMPLETE.

## DB-AUDIT-049
UNKNOWN completeness no equivaldrá a COMPLETE.

## DB-AUDIT-050
ORM audit reutilizará semantic metadata.

## DB-AUDIT-051
persist() no equivaldrá a successful mutation.

## DB-AUDIT-052
flush() no equivaldrá a commit().

## DB-AUDIT-053
Audit será transaction-aware.

## DB-AUDIT-054
Rollback no borrará necesariamente security evidence.

## DB-AUDIT-055
Transactional audit será distinto de security audit.

## DB-AUDIT-056
Security audit podrá requerir independent durability.

## DB-AUDIT-057
AuditStore será append-oriented.

## DB-AUDIT-058
Append-oriented no prometerá physical immutability.

## DB-AUDIT-059
Audit writer utilizará least privilege.

## DB-AUDIT-060
Audit read permissions serán separables.

## DB-AUDIT-061
Audit administration permissions serán separables.

## DB-AUDIT-062
Integrity protection será extensible.

## DB-AUDIT-063
Hash simple no será considerado tamper-proof.

## DB-AUDIT-064
Hash chaining será opcional.

## DB-AUDIT-065
Hash chain no sustituirá access control.

## DB-AUDIT-066
Integrity key será separable de encryption key.

## DB-AUDIT-067
Integrity serialization será canonical.

## DB-AUDIT-068
AuditSchemaVersion será explícita.

## DB-AUDIT-069
AuditSchemaVersion será distinta de FrameworkVersion.

## DB-AUDIT-070
Correcciones preferirán append de correction record.

## DB-AUDIT-071
Immutability será distinta de infinite retention.

## DB-AUDIT-072
Retention será policy-driven.

## DB-AUDIT-073
Legal hold será extensible.

## DB-AUDIT-074
Archive preservará integrity metadata.

## DB-AUDIT-075
Archive preservará protection requirements.

## DB-AUDIT-076
Audit search tendrá authorization propia.

## DB-AUDIT-077
Access to audit data podrá ser auditado.

## DB-AUDIT-078
Meta-audit evitará recursion infinita.

## DB-AUDIT-079
Recorder será distinto de Store.

## DB-AUDIT-080
Audit state preservará resultado real.

## DB-AUDIT-081
Read auditing será policy-driven.

## DB-AUDIT-082
No todos los SELECT serán auditados por default.

## DB-AUDIT-083
Unknown result count no equivaldrá a zero.

## DB-AUDIT-084
Field sensitivity podrá elevar audit requirement.

## DB-AUDIT-085
Projection sensitivity será preservada.

## DB-AUDIT-086
Derived sensitivity será preservada.

## DB-AUDIT-087
Exports sensibles serán auditables.

## DB-AUDIT-088
Imports sensibles serán auditables.

## DB-AUDIT-089
Bulk mutations serán auditables.

## DB-AUDIT-090
Destructive operations podrán exigir REQUIRED.

## DB-AUDIT-091
Schema changes serán auditables.

## DB-AUDIT-092
Migrations serán auditables.

## DB-AUDIT-093
Backups serán auditables.

## DB-AUDIT-094
Restores serán auditables.

## DB-AUDIT-095
Administrative database operations serán auditables.

## DB-AUDIT-096
Denied security operations serán auditables.

## DB-AUDIT-097
Malicious input no será copiado sin límites.

## DB-AUDIT-098
Audit payload será bounded.

## DB-AUDIT-099
Truncation será explícita.

## DB-AUDIT-100
Required evidence failure respetará failure policy.

## DB-AUDIT-101
Audit failure no será business rollback.

## DB-AUDIT-102
Committed business operation seguirá COMMITTED aunque after-commit audit falle.

## DB-AUDIT-103
Preflight no garantizará future append.

## DB-AUDIT-104
In-memory buffer no será considerado durable.

## DB-AUDIT-105
Durable buffer será explícito.

## DB-AUDIT-106
Audit Outbox podrá utilizarse.

## DB-AUDIT-107
Outbox no preservará denied attempts que nunca entraron en transaction.

## DB-AUDIT-108
Hybrid audit durability será soportada.

## DB-AUDIT-109
TransactionId correlacionará records transaccionales.

## DB-AUDIT-110
Logical nested transaction no fingirá physical nested transaction.

## DB-AUDIT-111
Savepoint audit será policy-driven.

## DB-AUDIT-112
Retries tendrán LogicalOperationId.

## DB-AUDIT-113
Retry attempts serán distinguibles.

## DB-AUDIT-114
Retries no producirán falsos duplicate successes.

## DB-AUDIT-115
Unknown commit no será reetiquetado.

## DB-AUDIT-116
Distributed outcomes serán representables por shard.

## DB-AUDIT-117
Audit no fingirá global ACID.

## DB-AUDIT-118
Partial distributed result será explícito.

## DB-AUDIT-119
Shard identity será estructurada.

## DB-AUDIT-120
Replica identity no expondrá credentials.

## DB-AUDIT-121
DSN secrets nunca aparecerán en Audit Records.

## DB-AUDIT-122
Tenant context será preservado.

## DB-AUDIT-123
Cross-tenant operations podrán elevar audit requirement.

## DB-AUDIT-124
Impersonation preservará actor original.

## DB-AUDIT-125
Effective actor será distinto del authenticated actor cuando corresponda.

## DB-AUDIT-126
Delegation chain será representable.

## DB-AUDIT-127
Authorization decision evidence podrá registrarse.

## DB-AUDIT-128
Authorization internals sensibles no serán volcados.

## DB-AUDIT-129
Audit no dependerá de Result Cache.

## DB-AUDIT-130
Cached reads podrán auditarse.

## DB-AUDIT-131
Data Access Audit podrá existir sin SQL.

## DB-AUDIT-132
Query Audit será distinto de Data Access Audit.

## DB-AUDIT-133
Lazy Collection creation no será access execution.

## DB-AUDIT-134
Lazy iteration podrá activar audit.

## DB-AUDIT-135
Chunk processing usará aggregate audit por default.

## DB-AUDIT-136
Cursor payload sensible no será almacenado directamente por default.

## DB-AUDIT-137
Streaming podrá registrar lifecycle.

## DB-AUDIT-138
Early termination será distinta de failure.

## DB-AUDIT-139
Cancellation será explícita.

## DB-AUDIT-140
Exception cruda no será audit payload default.

## DB-AUDIT-141
Reason codes serán estables.

## DB-AUDIT-142
Audit telemetry será distinta del audit trail.

## DB-AUDIT-143
Metric labels serán bounded.

## DB-AUDIT-144
Sensitive identifiers no serán metric labels.

## DB-AUDIT-145
Debug Toolbar no expondrá Audit Record sensible.

## DB-AUDIT-146
Audit Query API tendrá authorization propia.

## DB-AUDIT-147
Audit export será operación sensible.

## DB-AUDIT-148
Audit export podrá ser auditado.

## DB-AUDIT-149
Recursion guard será obligatorio.

## DB-AUDIT-150
Retention operations podrán ser auditadas.

## DB-AUDIT-151
Archive operations podrán ser auditadas.

## DB-AUDIT-152
AuditContext será scoped.

## DB-AUDIT-153
No habrá static current actor.

## DB-AUDIT-154
No habrá static current tenant.

## DB-AUDIT-155
Pending events no cruzarán requests.

## DB-AUDIT-156
Worker reset limpiará mutable audit state.

## DB-AUDIT-157
OpenSwoole preservará coroutine isolation.

## DB-AUDIT-158
FrankenPHP preservará request isolation.

## DB-AUDIT-159
RoadRunner preservará worker/request isolation.

## DB-AUDIT-160
Audit buffers serán bounded.

## DB-AUDIT-161
Batch writing preservará record identity.

## DB-AUDIT-162
Partial batch outcome será preservado.

## DB-AUDIT-163
UNKNOWN append será preservado.

## DB-AUDIT-164
Delivery retry será idempotency-aware.

## DB-AUDIT-165
Duplicate delivery será distinta de duplicate operation.

## DB-AUDIT-166
No se asumirá global ordering en sistemas distribuidos.

## DB-AUDIT-167
Timestamp será distinto de total ordering.

## DB-AUDIT-168
Clock será abstracto.

## DB-AUDIT-169
Testing podrá controlar Clock.

## DB-AUDIT-170
Audit testing verificará ausencia de secrets.

## DB-AUDIT-171
Credential values nunca serán auditados.

## DB-AUDIT-172
Encryption keys nunca serán auditadas.

## DB-AUDIT-173
Sensitive query bindings serán redacted.

## DB-AUDIT-174
Sensitive ChangeSets serán redacted.

## DB-AUDIT-175
Audit store credentials nunca serán auditadas.

## DB-AUDIT-176
Audit policy generation será identificable.

## DB-AUDIT-177
Audit architecture será vendor-independent.

## DB-AUDIT-178
Audit persistence backend será reemplazable.

## DB-AUDIT-179
Audit integrity provider será reemplazable.

## DB-AUDIT-180
Audit security tendrá precedencia sobre debugging convenience.

---

# 303. Modelo formal

Sea una operación:

```text
O
```

ejecutada por actor:

```text
A
```

sobre recurso:

```text
R
```

bajo contexto:

```text
C
```

con resultado conocido:

```text
X
```

Entonces el candidato de auditoría será:

```text
Candidate = (A, O, R, C, X)
```

---

# 304. Policy evaluation

```text
Decision
=
AuditPolicy(
    A,
    O,
    R,
    C,
    Sensitivity(R)
)
```

---

# 305. Record construction

Si:

```text
Decision.audit = true
```

entonces:

```text
AuditRecord
=
Sanitize(
    Capture(
        Candidate,
        Decision
    )
)
```

---

# 306. Security condition

Para cualquier valor sensible `S` que la policy prohíba registrar:

```text
S ∉ AuditRecord
```

---

# 307. Transaction model

Para una mutación `M` dentro de transacción `T`:

```text
Execute(M) = SUCCESS
```

no implica:

```text
AuditOutcome(M) = COMMITTED
```

hasta conocer:

```text
Commit(T) = SUCCESS
```

---

# 308. Rollback model

Si:

```text
Execute(M) = SUCCESS
Commit(T) = ROLLBACK
```

entonces:

```text
ExecutionOutcome(M) = EXECUTED
TransactionOutcome(M) = ROLLED_BACK
```

Ambos hechos pueden conservarse.

---

# 309. Unknown transaction model

Si:

```text
CommitRequest(T) = SENT
Connection = LOST
```

entonces:

```text
TransactionOutcome(T) = UNKNOWN
```

y:

```text
AuditOutcome(M)
```

no podrá fabricarse como `COMMITTED` o `ROLLED_BACK`.

---

# 310. Audit failure model

Sea:

```text
B = business operation
A = audit append
```

Es posible:

```text
B = COMMITTED
A = FAILED
```

Por tanto:

```text
AuditFailure
≠
BusinessFailure
```

---

# 311. Distributed model

Para operación `O` sobre shards:

```text
S = {s1, s2, ..., sn}
```

su evidencia será:

```text
Outcome(O)
=
Aggregate(
    Outcome(O,s1),
    Outcome(O,s2),
    ...,
    Outcome(O,sn)
)
```

Si existe incertidumbre material:

```text
Outcome(O)
=
PARTIAL or UNKNOWN
```

según evidencia.

---

# 312. Arquitectura integrada

```text
                         Application
                              │
                              ▼
                     Database Public API
                              │
                 ┌────────────┴─────────────┐
                 ▼                          ▼
          Security Engine             Query / ORM
                 │                          │
                 ▼                          ▼
       Authorization Decision        Execution Intent
                 │                          │
                 └────────────┬─────────────┘
                              ▼
                       Audit Candidate
                              │
                              ▼
                       Audit Policy
                              │
                              ▼
                        Audit Event
                              │
                              ▼
                    Sensitive Sanitizer
                              │
                              ▼
                        Audit Record
                              │
                 ┌────────────┼─────────────┐
                 ▼            ▼             ▼
             Integrity      Buffer       Outbox
                 │            │             │
                 └────────────┼─────────────┘
                              ▼
                         Audit Store
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                  Search   Retention  Archive
```

---

# 313. Integración con Transaction System

```text
Transaction
    │
    ├── Attempt
    │      ↓
    │   Security Audit
    │
    ├── Execute
    │      ↓
    │   Execution Evidence
    │
    ├── Commit
    │      ├── SUCCESS
    │      ├── FAILURE
    │      └── UNKNOWN
    │
    ▼
Audit Outcome Resolution
```

---

# 314. Integración con ORM

```text
Entity
   ↓
UnitOfWork
   ↓
ChangeSet
   ↓
Persistence Plan
   ↓
Audit Mapper
   ↓
Sanitized Change Evidence
   ↓
Transaction Correlation
   ↓
Audit Record
```

---

# 315. Integración con Sensitive Data Protection

```text
Query / Entity / ChangeSet
          │
          ▼
Sensitivity Metadata
          │
          ▼
Audit Capture Policy
          │
          ▼
Sanitizer
    ┌─────┼─────┐
    ▼     ▼     ▼
 Redact  Mask  Fingerprint
    │     │     │
    └─────┼─────┘
          ▼
    Safe Audit Payload
```

---

# 316. Security defaults

VoltStack deberá favorecer:

```text
Sensitive values
→ REDACT

Credentials
→ NEVER CAPTURE

Encryption keys
→ NEVER CAPTURE

Unknown sensitivity
→ CONSERVATIVE

Unknown actor
→ UNKNOWN

Unknown transaction result
→ UNKNOWN

Audit append uncertainty
→ UNKNOWN

Required audit unavailable
→ FAIL according to strict policy

Bulk operation
→ AGGREGATED by default

SQL parameters
→ REDACTED

Audit store
→ APPEND-oriented

Audit access
→ RESTRICTED
```

---

# 317. Anti-patterns

## Anti-pattern 1

```php
logger()->info($sql, $bindings);
```

y llamarlo auditoría.

Problemas:

```text
secrets
PII leakage
no actor semantics
no transaction outcome
no integrity
```

---

## Anti-pattern 2

Auditar solamente queries exitosas.

Esto elimina evidencia de:

```text
denied attempts
security violations
failed privileged operations
```

---

## Anti-pattern 3

Registrar `flush()` como `COMMITTED`.

Incorrecto:

```text
flush()
≠
commit()
```

---

## Anti-pattern 4

Guardar todo el ChangeSet.

Puede convertir el Audit Store en una segunda base de datos de información sensible.

---

## Anti-pattern 5

Usar un único usuario:

```text
actor = system
```

para todos los background jobs.

Se pierde accountability.

---

## Anti-pattern 6

Auditar un bulk delete con un record por cada fila sin necesidad.

Puede provocar:

```text
audit amplification
storage exhaustion
latency
DoS
```

---

## Anti-pattern 7

Ignorar audit failure después del commit.

Debe existir evidencia operacional de:

```text
business committed
audit failed
```

---

## Anti-pattern 8

Usar timestamps como orden global absoluto.

---

## Anti-pattern 9

Permitir que cualquier aplicación modifique records históricos.

---

## Anti-pattern 10

Guardar passwords, credentials, tokens o encryption keys para "tener auditoría completa".

Eso reduce la seguridad.

---

# 318. Resultado arquitectónico

Con este sistema, VoltStack Database podrá responder preguntas como:

```text
¿Quién realizó esta operación?

¿Qué tipo de operación fue?

¿Qué recurso afectó?

¿A qué tenant pertenecía?

¿En qué transacción ocurrió?

¿Fue autorizada?

¿Fue ejecutada?

¿Fue committed?

¿Fue rolled back?

¿El resultado quedó unknown?

¿Fue parte de un bulk operation?

¿En qué shard ocurrió?

¿Fue un acceso privilegiado?

¿La operación accedió a información sensible?

¿Existe evidencia íntegra del evento?
```

sin necesitar registrar indiscriminadamente:

```text
passwords
tokens
credentials
PII
query bindings
plaintext sensitive fields
encryption keys
```

---

# 319. Regla maestra final

> **El Query Audit System de VoltStack deberá registrar evidencia, no duplicar datos.**

Formalmente:

```text
AuditEvidence
=
Identity
+
Operation
+
Subject
+
Context
+
Authority
+
Outcome
+
Correlation
+
Integrity
```

pero:

```text
AuditEvidence
≠
RawApplicationState
```

y:

```text
AuditEvidence
≠
SensitiveDataDump
```

La propiedad principal será:

```text
SufficientEvidence
∧
MinimumNecessaryExposure
```

Es decir:

> **la auditoría deberá contener suficiente información para accountability, investigación, seguridad y cumplimiento, pero únicamente la mínima cantidad de información sensible necesaria para cumplir ese propósito.**

---

# 320. Estado del Bloque 22

```text
BLOCK 22 — SECURITY

✓ 226_DATABASE_SECURITY_ARCHITECTURE.md
✓ 227_DATABASE_SQL_INJECTION_PREVENTION_SYSTEM.md
✓ 228_DATABASE_QUERY_INPUT_SECURITY_SYSTEM.md
✓ 229_DATABASE_CREDENTIAL_SECURITY_SYSTEM.md
✓ 230_DATABASE_CONNECTION_SECURITY_SYSTEM.md
✓ 231_DATABASE_DATA_ACCESS_SECURITY_SYSTEM.md
✓ 232_DATABASE_SENSITIVE_DATA_PROTECTION_SYSTEM.md
✓ 233_DATABASE_QUERY_AUDIT_SYSTEM.md
○ 234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

---

# 321. Siguiente documento

```text
234_DATABASE_DATABASE_PERMISSION_MODEL.md
```

Este documento cerrará el **Bloque 22 — Security** definiendo el modelo de permisos específico del subsistema Database:

```text
DatabasePrincipal
Permission
PermissionSet
DatabaseAction
DatabaseResource
ResourceScope
FieldScope
RowScope
TenantScope
DatabaseScope
SchemaScope
TableScope
EntityScope
QueryScope
OperationScope
PermissionDecision
PermissionResolver
PermissionEvaluator
PermissionContext
PermissionInheritance
PermissionComposition
Explicit Allow
Explicit Deny
Default Deny
Administrative Permissions
Privileged Operations
Delegation
Impersonation
Temporary Permissions
Capability-bound Permissions
Cross-tenant Permissions
Shard-aware Permissions
Raw SQL Permissions
Schema Permissions
Migration Permissions
Import/Export Permissions
Backup/Restore Permissions
Audit Permissions
Sensitive Data Permissions
Permission Cache
Permission Invalidations
Permission Telemetry
Permission Audit
Persistent Runtime Isolation
```

bajo la separación:

```text
Application Authorization
        │
        ▼
Database Permission Model
        │
        ▼
Data Access Security
        │
        ▼
Query / ORM / Schema / Operations
```

y la regla central:

> **Ninguna operación Database deberá obtener autoridad implícita por el simple hecho de alcanzar una API capaz de ejecutarla; las operaciones protegidas deberán evaluarse contra permisos semánticos, recursos, scopes y contexto antes de llegar al punto irreversible de ejecución.**