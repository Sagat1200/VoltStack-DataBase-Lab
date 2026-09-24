# 320_DATABASE_JOB_AND_QUEUE_INTEGRATION_SYSTEM.md

# VoltStack Quantum Database
## Database Job and Queue Integration System

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 320 — Database Job and Queue Integration System  
**Bloque:** 32 — VoltStack Integration  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `319_DATABASE_AUTHORIZATION_INTEGRATION_SYSTEM.md`  
**Siguiente documento:** `321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md`

---

# 1. Propósito

Este documento define la arquitectura de integración entre:

```text
VoltStack/Quantum/Database
```

y:

```text
VoltStack Jobs / Queue
```

incluyendo:

- dispatch de jobs;
- workers;
- queues;
- jobs diferidos;
- jobs programados;
- jobs batch;
- retries;
- idempotencia;
- deduplicación;
- transactional dispatch;
- `afterCommit`;
- Transactional Outbox;
- Database-backed queues;
- serialización de payloads;
- referencias a entidades;
- propagación de tenant;
- propagación de actor;
- reconstrucción de `DatabaseContext`;
- conexiones;
- transacciones;
- leasing;
- visibility timeout;
- heartbeats;
- failed jobs;
- dead-letter queues;
- sharding;
- read/write routing;
- persistent runtimes;
- telemetry;
- auditoría;
- seguridad;
- testing.

La regla fundamental será:

> **Database y Queue constituyen fronteras transaccionales diferentes salvo que exista un mecanismo explícito que demuestre lo contrario; VoltStack nunca deberá interpretar la escritura exitosa en Database y la publicación exitosa en una Queue externa como una única transacción atómica.**

Por tanto:

```text
Database Commit
≠
Queue Publish
```

y:

```text
Job Dispatched
≠
Job Executed
≠
Job Completed
```

---

# 2. Problema central: Dual Write

Considérese:

```php
$order = new Order(...);

$entityManager->persist($order);
$entityManager->flush();

$queue->dispatch(
    new SendOrderConfirmation($order->id)
);
```

Aparentemente:

```text
Save Order
+
Dispatch Job
```

forman una sola operación.

Arquitectónicamente no necesariamente.

---

# 3. Dos sistemas

Podrían existir:

```text
Database
   │
   └── COMMIT

Queue Broker
   │
   └── PUBLISH
```

con dos resultados independientes.

---

# 4. Failure window A

```text
Database COMMIT
      ↓
SUCCESS
      ↓
Queue publish
      ↓
FAIL
```

Resultado:

```text
Order exists
Job does not exist
```

---

# 5. Failure window B

Si se publica antes:

```text
Queue publish
      ↓
SUCCESS
      ↓
Database COMMIT
      ↓
FAIL
```

resultado:

```text
Job exists
Order does not exist
```

---

# 6. Regla de dual-write

> **VoltStack no deberá intentar ocultar un dual-write detrás de una API aparentemente atómica si no existe una garantía real que una ambas fronteras.**

---

# 7. Separaciones fundamentales

Deberá mantenerse:

```text
Database Transaction
≠
Queue Transaction
```

```text
Database Commit
≠
Queue Publish
```

```text
Queue ACK
≠
Database Commit
```

```text
Job Payload
≠
Entity
```

```text
Job Retry
≠
Database Retry
```

```text
Queue Delivery
≠
Exactly-Once Execution
```

```text
Job Lease
≠
Job Ownership Forever
```

```text
Job Completed
≠
External Side Effect Completed
```

```text
afterCommit
≠
Distributed Transaction
```

```text
Outbox
≠
Queue
```

---

# 8. Arquitectura general

```text
Application
    │
    ├───────────────┐
    │               │
    ▼               ▼
Database API      Job Dispatcher
    │               │
    ▼               │
Transaction         │
Manager             │
    │               │
    ├── afterCommit ┤
    │               │
    ▼               ▼
Database          Queue System
    │               │
    │               ▼
    │            Broker
    │               │
    │               ▼
    │             Worker
    │               │
    └───────────────┤
                    ▼
             DatabaseContext
                    │
                    ▼
                Database
```

Para operaciones críticas:

```text
Application
    │
    ▼
Database Transaction
    │
    ├── Domain Changes
    │
    └── Outbox Record
            │
            ▼
         COMMIT
            │
            ▼
      Outbox Relay
            │
            ▼
        Queue Broker
            │
            ▼
          Worker
```

---

# 9. Responsabilidades de Database

Database será responsable de:

- persistencia;
- transacciones;
- atomicidad Database;
- locks;
- concurrencia;
- outbox persistence;
- idempotency records cuando se almacenen en DB;
- job state si se utiliza Database Queue;
- leasing DB-backed;
- tenant/shard routing;
- connection lifecycle;
- consistency evidence.

---

# 10. Responsabilidades de Queue

Queue será responsable de:

- encolar mensajes;
- transportar jobs;
- scheduling;
- delivery;
- leasing/visibility;
- retries de delivery;
- acknowledgements;
- dead-lettering;
- distribución entre workers.

---

# 11. Responsabilidades de Jobs

Jobs será responsable de:

- representar unidades de trabajo;
- definir handlers;
- reconstruir contexto;
- ejecutar lógica;
- gestionar retries;
- idempotencia;
- timeouts;
- cancellation;
- failure policy;
- lifecycle.

---

# 12. Integration layer

Se propone:

```text
VoltStack\Quantum\Database\Integration\Queue
```

como frontera.

---

# 13. Dependency direction

Preferentemente:

```text
Jobs / Queue
      ↓
Database Integration Contracts
      ↓
Database
```

Database Core no deberá conocer handlers concretos.

---

# 14. Database Core independence

No deberá existir:

```text
QueryBuilder
→ JobDispatcher
```

ni:

```text
Driver
→ QueueBroker
```

ni:

```text
SQLCompiler
→ Job
```

ni:

```text
Connection
→ Worker
```

---

# 15. Dispatch modes

VoltStack podrá soportar:

```text
IMMEDIATE
AFTER_COMMIT
OUTBOX
DATABASE_QUEUE
SCHEDULED
```

---

# 16. Immediate dispatch

```text
dispatch(job)
      ↓
Queue Publish
```

No espera una transacción Database.

---

# 17. Riesgo del immediate dispatch

Dentro de:

```text
BEGIN
...
dispatch(job)
...
COMMIT
```

el worker podría ejecutar antes del commit.

---

# 18. Ejemplo

```text
Transaction A
─────────────
INSERT Order

dispatch Job
      │
      └───────────────► Worker
                         │
                         ▼
                     SELECT Order
                         │
                         ▼
                     NOT FOUND

COMMIT
```

---

# 19. AFTER_COMMIT

`AFTER_COMMIT` registra la intención durante la transacción y publica después de un commit confirmado.

```text
BEGIN
  DB changes
  register pending job
COMMIT
  ↓
confirmed
  ↓
publish job
```

---

# 20. afterCommit rule

Un job registrado como:

```text
AFTER_COMMIT
```

no deberá publicarse si ocurre:

```text
ROLLBACK
```

---

# 21. Nested transactions

El sistema deberá distinguir:

```text
Logical Nested Transaction
```

de:

```text
Physical Transaction
```

---

# 22. afterCommit and savepoints

Liberar un:

```text
SAVEPOINT
```

no significa:

```text
database transaction committed
```

Por tanto:

```text
afterCommit
```

se dispara únicamente cuando la frontera física correspondiente se confirma.

---

# 23. afterCommit limitation

Considérese:

```text
DB COMMIT
   ↓
SUCCESS
   ↓
process crashes
   ↓
before queue publish
```

El job puede perderse.

---

# 24. Regla crítica

> **`afterCommit` resuelve el problema de publicar antes de confirmar Database, pero por sí solo no garantiza que una publicación ocurra después de un commit exitoso.**

---

# 25. afterCommit ≠ reliable delivery

Formalmente:

```text
afterCommit
≠
Guaranteed Publish
```

---

# 26. Transactional Outbox

Para garantías más fuertes se utilizará:

```text
Transactional Outbox
```

---

# 27. Outbox concept

Dentro de la misma transacción:

```text
BEGIN

INSERT/UPDATE business data

INSERT outbox_message

COMMIT
```

Ambos pertenecen a:

```text
same Database transaction
```

---

# 28. Atomic persistence

Entonces:

```text
Business Change
+
Outbox Intent
```

son atómicos respecto al DB.

---

# 29. Outbox pipeline

```text
Application
      │
      ▼
BEGIN
      │
      ├── Business Mutation
      │
      └── Outbox Message
      │
      ▼
COMMIT
      │
      ▼
Outbox Relay
      │
      ▼
Queue Broker
      │
      ▼
Consumer
```

---

# 30. Outbox ≠ exactly once

El relay puede:

```text
publish
      ↓
broker accepts
      ↓
relay crashes
      ↓
before marking published
```

y publicar nuevamente.

---

# 31. Therefore

```text
Transactional Outbox
+
Idempotent Consumer
```

es el modelo preferido.

---

# 32. Outbox state

Se propone:

```text
PENDING
LEASED
PUBLISHED
RETRYABLE_FAILURE
FAILED
```

---

# 33. UNKNOWN publish outcome

Debe existir conceptualmente:

```text
UNKNOWN
```

cuando no pueda demostrarse si el broker recibió el mensaje.

---

# 34. UNKNOWN ≠ failed

Nunca:

```text
timeout
→
definitely not published
```

---

# 35. Duplicate possibility

Si el publish outcome es desconocido:

```text
retry publish
```

puede producir duplicados.

Esto debe ser asumido por diseño.

---

# 36. Message identity

Todo mensaje durable deberá tener:

```text
MessageId
```

estable.

---

# 37. Retry preserves MessageId

Un retry de publicación del mismo mensaje deberá conservar:

```text
MessageId
```

---

# 38. Execution identity

Un delivery podrá tener:

```text
DeliveryId
```

diferente.

---

# 39. MessageId ≠ DeliveryId

Ejemplo:

```text
MessageId = M100

Delivery 1 = D1
Delivery 2 = D2
Delivery 3 = D3
```

---

# 40. Job identity

También podrá existir:

```text
JobId
```

---

# 41. Identifiers

Por tanto:

```text
MessageId
≠
DeliveryId
≠
JobExecutionId
≠
CorrelationId
```

---

# 42. Job envelope

Se propone:

```text
JobEnvelope
├── JobId
├── MessageId
├── JobType
├── SchemaVersion
├── Payload
├── CreatedAt
├── AvailableAt
├── Attempt
├── MaxAttempts
├── CorrelationId?
├── CausationId?
├── TenantReference?
├── ShardHint?
├── ActorReference?
├── TraceContext?
├── IdempotencyKey?
└── Metadata
```

---

# 43. Payload rule

> **Un Job Payload deberá contener los datos mínimos necesarios para reconstruir el trabajo, no una copia arbitraria del estado interno del ORM.**

---

# 44. Entity ≠ payload

Evitar:

```php
new ProcessOrderJob($order);
```

si `$order` es una entidad administrada.

Preferir:

```php
new ProcessOrderJob(
    orderId: $order->id,
);
```

---

# 45. Why not serialize entities

Una entidad puede contener:

- dirty state;
- proxies;
- lazy relations;
- EntityManager references;
- IdentityMap assumptions;
- snapshots;
- stale data;
- tenant context;
- non-serializable resources.

---

# 46. Entity serialization prohibited by default

VoltStack deberá impedir o advertir sobre serialización automática de entidades ORM completas en jobs.

---

# 47. Entity reference

Se propone:

```text
EntityReference
├── EntityType
├── Identifier
├── TenantReference?
├── ShardReference?
└── Version?
```

---

# 48. Reference ≠ snapshot

```text
EntityReference
```

significa:

```text
load current state when executing
```

salvo semántica explícita.

---

# 49. Snapshot payload

Si el job requiere el estado exacto del momento de dispatch, deberá utilizar:

```text
Explicit Snapshot DTO
```

---

# 50. Snapshot ≠ entity

El DTO será:

```text
immutable
serializable
versioned
minimal
```

---

# 51. Stale references

Entre dispatch y ejecución:

```text
resource
```

puede:

- cambiar;
- desaparecer;
- cambiar de tenant;
- ser archivado;
- ser soft-deleted.

El handler deberá definir la política correspondiente.

---

# 52. Job handler flow

```text
Delivery
   ↓
Decode Envelope
   ↓
Validate Version
   ↓
Resolve Execution Context
   ↓
Resolve Tenant
   ↓
Resolve Shard
   ↓
Resolve Actor / Delegation
   ↓
Create DatabaseContext
   ↓
Open Operation Scope
   ↓
Execute Handler
   ↓
Commit / Rollback
   ↓
Cleanup Database State
   ↓
ACK / Retry / Fail
```

---

# 53. Worker scope

Cada ejecución deberá tener un:

```text
JobExecutionScope
```

independiente.

---

# 54. Job execution context

Se propone:

```text
JobExecutionContext
├── JobId
├── MessageId
├── DeliveryId
├── Attempt
├── Queue
├── WorkerId
├── TenantContext?
├── ShardContext?
├── ActorContext?
├── TraceContext?
├── CancellationContext
├── Deadline
└── DatabaseContext
```

---

# 55. DatabaseContext reconstruction

Nunca deberá reutilizarse ciegamente un:

```text
DatabaseContext
```

del proceso que creó el job.

---

# 56. New scope

Cada job crea:

```text
fresh DatabaseContext
```

a partir de metadata segura.

---

# 57. Tenant propagation

Si el job pertenece a un tenant:

```text
TenantReference
```

podrá transportarse en el envelope.

---

# 58. Tenant reference ≠ trust

El worker deberá validar:

```text
TenantReference
```

antes de utilizarla.

---

# 59. Tenant resolution

```text
TenantReference
      ↓
Tenant Resolver
      ↓
Tenant Context
      ↓
Database Routing
```

---

# 60. Cross-tenant jobs

Deberán ser:

```text
explicit
privileged
auditable
```

---

# 61. Shard propagation

Puede transportarse:

```text
ShardHint
```

pero:

```text
ShardHint
≠
Shard Authority
```

---

# 62. Shard resolution

El sistema podrá recalcular el shard a partir de:

```text
Shard Key
Tenant
Resource Identity
Partition Map
```

---

# 63. Stale shard hint

Si el topology generation cambió:

```text
old shard hint
```

podrá ser inválido.

---

# 64. Actor propagation

Un job podrá ejecutar como:

```text
SystemActor
ServiceActor
OriginalActor
DelegatedActor
```

---

# 65. Authentication object prohibited

No serializar:

```text
AuthenticationContext
Session
AccessToken
User ORM Entity
```

completos por defecto.

---

# 66. ActorReference

Preferir:

```text
ActorReference
├── ActorType
├── ActorId
├── DelegationId?
└── SecurityVersion?
```

---

# 67. Reauthorization

Si la operación depende de autorización actual:

```text
worker
→
re-authorize
```

durante ejecución.

---

# 68. Authorized at dispatch ≠ authorized at execution

Regla:

```text
ALLOW(t1)
≠
ALLOW(t2)
```

---

# 69. Delegated jobs

Si se requiere conservar autoridad concedida previamente, utilizar una:

```text
DelegationCapability
```

explícita.

---

# 70. Delegation properties

Podrá contener:

```text
issuer
subject
actions
resource scope
tenant
issued at
expiration
nonce
policy generation
signature
```

---

# 71. Delegation security

Nunca deberá convertirse:

```text
user was admin at dispatch
```

en:

```text
job has permanent admin privileges
```

implícitamente.

---

# 72. Job transaction boundary

Un handler podrá ejecutar:

```php
$database->transaction(function () use ($job) {
    // work
});
```

---

# 73. Job ≠ transaction

Un job puede contener:

```text
zero
one
multiple
```

transacciones.

---

# 74. Transaction-per-job

Puede ofrecerse como política:

```text
TRANSACTION_PER_JOB
```

pero no será universal.

---

# 75. Why not universal

Algunos jobs:

- llaman APIs externas;
- procesan streams;
- duran minutos;
- ejecutan múltiples fases;
- requieren commits parciales.

---

# 76. Long transaction warning

Nunca deberá mantenerse una transacción Database abierta durante:

```text
slow external HTTP call
```

sin razón explícita.

---

# 77. External side effects

Ejemplo:

```text
BEGIN
UPDATE invoice
send email
COMMIT
```

es peligroso.

---

# 78. Rollback cannot unsend email

```text
Database Rollback
≠
External Side Effect Rollback
```

---

# 79. Side-effect orchestration

Preferir:

```text
Database Commit
      ↓
Outbox
      ↓
Job
      ↓
External Side Effect
```

cuando sea apropiado.

---

# 80. Idempotency

Los handlers deberán asumir potencialmente:

```text
at-least-once delivery
```

---

# 81. Idempotent execution

Sea:

```text
F(job)
```

idealmente:

```text
F(F(job))
≈
F(job)
```

respecto al efecto observable esperado.

---

# 82. Natural idempotency

Ejemplo:

```text
SET status = processed
```

puede ser naturalmente más idempotente que:

```text
balance = balance + 100
```

---

# 83. Idempotency key

Se propone:

```text
IdempotencyKey
```

para operaciones que necesiten deduplicación persistente.

---

# 84. Idempotency record

Ejemplo:

```text
job_id
operation
idempotency_key
status
created_at
completed_at
result_reference
```

---

# 85. Atomic idempotency

Idealmente:

```text
Idempotency Claim
+
Business Mutation
```

se ejecutarán dentro de la misma transacción Database cuando ambos estén en el mismo DB.

---

# 86. Naive idempotency bug

Incorrecto:

```text
SELECT idempotency_key

if not exists:
    perform mutation
    INSERT idempotency_key
```

dos workers pueden competir.

---

# 87. Correct claim

Utilizar:

```text
UNIQUE constraint
```

y/o:

```text
atomic insert
```

como parte de la estrategia.

---

# 88. Idempotency state

Podrá modelarse:

```text
CLAIMED
PROCESSING
COMPLETED
FAILED_RETRYABLE
FAILED_FINAL
UNKNOWN
```

---

# 89. UNKNOWN execution

Si la conexión se pierde durante commit:

```text
COMMIT sent
connection lost
```

el resultado podrá ser:

```text
UNKNOWN
```

---

# 90. Critical retry rule

> **Un worker no deberá repetir ciegamente una operación cuyo resultado Database sea UNKNOWN.**

---

# 91. Reconciliation

Deberá:

```text
inspect durable state
```

antes de decidir si reintentar.

---

# 92. Job retry

Un retry de job debe comenzar desde una frontera segura.

---

# 93. Job retry ≠ statement retry

Nunca:

```text
failed SQL statement
→
re-run arbitrary statement
```

como sustituto de reejecutar correctamente el handler/transacción.

---

# 94. Retry hierarchy

```text
Statement Retry
Transaction Retry
Job Retry
Message Redelivery
Outbox Publish Retry
```

son mecanismos diferentes.

---

# 95. Retry ownership

Cada capa deberá conocer:

```text
who owns retry?
```

para evitar retries multiplicativos.

---

# 96. Retry explosion

Ejemplo peligroso:

```text
Driver retry:       3
Transaction retry:  3
Job retry:          5
Queue redelivery:   5
```

potencialmente:

```text
3 × 3 × 5 × 5 = 225
```

intentos parciales.

---

# 97. Retry budget

Se deberá modelar:

```text
RetryBudget
```

a nivel de operación.

---

# 98. Retry metadata

Podrá incluir:

```text
attempt
max attempts
first attempted at
last attempted at
retry reason
backoff
deadline
```

---

# 99. Backoff

Se soportará:

```text
fixed
linear
exponential
exponential + jitter
custom
```

---

# 100. Deadlock

Un deadlock dentro de una transacción puede permitir:

```text
transaction retry
```

sin necesariamente consumir un retry completo de queue.

---

# 101. Handler policy

La política deberá decidir qué capa absorbe el fallo.

---

# 102. Serialization failures

Un payload inválido deberá fallar antes de ejecutar lógica Database.

---

# 103. Job schema version

Todo job durable deberá ser versionable.

Ejemplo:

```text
JobType = SendInvoice
SchemaVersion = 3
```

---

# 104. Worker compatibility

Workers deberán conocer:

```text
supported job versions
```

---

# 105. Unknown job version

Nunca ejecutar parcialmente.

Debe:

```text
reject
quarantine
dead-letter
```

según política.

---

# 106. Payload migrations

Podrán existir:

```text
JobPayloadUpcaster
```

para convertir versiones antiguas.

---

# 107. Upcaster restrictions

Un upcaster:

```text
old payload
→
new payload
```

no deberá realizar queries Database arbitrarias por defecto.

---

# 108. Job persistence

Queue metadata podrá persistirse en:

```text
external broker
database
hybrid
```

---

# 109. Database-backed queue

VoltStack podrá proporcionar un adapter:

```text
DatabaseQueueDriver
```

---

# 110. Database queue architecture

```text
Producer
   │
   ▼
INSERT job
   │
   ▼
jobs table
   │
   ▼
Worker
   │
   ▼
Claim / Lease
   │
   ▼
Execute
   │
   ▼
ACK / Retry / Fail
```

---

# 111. Database queue advantages

Puede ofrecer:

- infraestructura simple;
- transactional enqueue;
- familiaridad operacional;
- atomicidad con datos de negocio si comparten DB/transacción;
- buena opción para cargas moderadas.

---

# 112. Database queue limitations

Puede introducir:

- contention;
- polling;
- table growth;
- index pressure;
- lock contention;
- cleanup requirements;
- DB workload competition.

---

# 113. Database Queue ≠ Outbox

Aunque ambos usen tablas:

```text
DatabaseQueue
≠
TransactionalOutbox
```

---

# 114. Difference

Database Queue representa:

```text
work waiting for workers
```

Outbox representa:

```text
durable intent to publish/integrate
```

---

# 115. Combined model

Puede existir:

```text
Business DB
+
Outbox
+
Database Queue
```

pero no deberán confundirse sus responsabilidades.

---

# 116. Database queue schema

Conceptualmente:

```text
jobs
├── id
├── queue
├── type
├── version
├── payload
├── status
├── priority
├── available_at
├── leased_until
├── lease_owner
├── attempts
├── max_attempts
├── created_at
├── completed_at
└── metadata
```

---

# 117. Claiming jobs

Un worker deberá reclamar trabajo atómicamente.

---

# 118. Incorrect claim

```text
SELECT first pending job
↓
UPDATE status = running
```

sin protección puede permitir doble claim.

---

# 119. Claim strategies

Dependiendo de capabilities:

```text
SELECT ... FOR UPDATE SKIP LOCKED
atomic UPDATE ... RETURNING
lease token
compare-and-swap
platform-specific safe strategy
```

---

# 120. Capability-driven strategy

Nunca asumir:

```text
SKIP LOCKED
```

universalmente.

Consultar:

```text
DatabaseCapabilitySystem
```

---

# 121. Lease

Un job reclamado obtiene:

```text
JobLease
```

---

# 122. Lease structure

```text
JobLease
├── JobId
├── LeaseId
├── WorkerId
├── AcquiredAt
├── ExpiresAt
└── Generation
```

---

# 123. Lease ≠ ownership

El worker no posee permanentemente el job.

---

# 124. Visibility timeout

Mientras el lease esté vigente:

```text
other workers
```

no deberán reclamar normalmente el job.

---

# 125. Worker crash

Si el worker desaparece:

```text
lease expires
```

y el job puede ser reclamado nuevamente.

---

# 126. Duplicate execution window

El worker original podría seguir vivo pero haber perdido conectividad.

Entonces:

```text
Worker A still executing
Lease expires
Worker B claims
```

produciendo ejecución concurrente.

---

# 127. Therefore

```text
Lease
≠
Exactly Once
```

---

# 128. Fencing token

Para operaciones sensibles podrá utilizarse:

```text
FencingToken
```

monotónico.

---

# 129. Fencing model

```text
Lease generation 41
Worker A

Lease expires

Lease generation 42
Worker B
```

La persistencia puede rechazar acciones con:

```text
generation < 42
```

cuando el modelo lo permita.

---

# 130. Lease renewal

Jobs largos podrán renovar:

```text
leased_until
```

mediante heartbeat.

---

# 131. Heartbeat ≠ business progress

Un heartbeat sólo demuestra:

```text
worker lease activity
```

no que el job haya completado trabajo correctamente.

---

# 132. ACK

Un ACK indica al queue system:

```text
delivery accepted as completed
```

según su protocolo.

---

# 133. ACK timing

Para jobs Database:

```text
COMMIT
must be confirmed
before ACK
```

cuando el commit representa el efecto requerido.

---

# 134. Incorrect ACK

Nunca:

```text
ACK
↓
COMMIT
```

si un crash entre ambos perdería el trabajo.

---

# 135. Correct basic sequence

```text
Execute
   ↓
COMMIT confirmed
   ↓
ACK
```

---

# 136. ACK failure after commit

Puede ocurrir:

```text
DB COMMIT
   ↓
SUCCESS
   ↓
ACK
   ↓
FAIL
```

El broker podrá redeliver.

---

# 137. Therefore idempotency

Este escenario exige nuevamente:

```text
idempotent handler
```

---

# 138. Queue outcome states

Se deberá distinguir:

```text
PUBLISHED
DELIVERED
LEASED
EXECUTING
COMPLETED
ACKNOWLEDGED
RETRY_SCHEDULED
FAILED
DEAD_LETTERED
UNKNOWN
```

---

# 139. State ≠ truth across systems

El estado Queue no prueba automáticamente el estado Database.

---

# 140. Failed jobs

Un job podrá alcanzar:

```text
FAILED_FINAL
```

cuando su retry policy se agote.

---

# 141. Failed job record

Podrá contener:

```text
JobId
JobType
Attempt
FailureClass
FailureCode
FailedAt
SafeDiagnostic
EnvironmentFingerprint
TraceId
```

---

# 142. No secrets

No almacenar:

- passwords;
- tokens;
- raw credentials;
- unnecessary PII;
- raw SQL parameters sensibles.

---

# 143. Dead-letter queue

Una DLQ deberá preservar suficiente evidencia para:

```text
diagnose
replay
quarantine
```

---

# 144. Replay

Replay no significa:

```text
reset attempt count and hope
```

Deberá comprobar:

- payload compatibility;
- tenant validity;
- authorization;
- idempotency;
- resource state;
- handler version;
- environment.

---

# 145. Manual replay

Deberá ser:

```text
authorized
audited
explicit
```

---

# 146. Job cancellation

Un job podrá tener:

```text
CancellationToken
```

---

# 147. Cancellation ≠ rollback

Si ya se realizaron commits:

```text
cancel
```

no revierte automáticamente esos commits.

---

# 148. Cooperative cancellation

Handlers largos deberán consultar cancellation en fronteras seguras.

---

# 149. Job timeout

Deberán distinguirse:

```text
Queue Lease Timeout
Job Execution Timeout
Database Query Timeout
Transaction Timeout
External Service Timeout
```

---

# 150. Timeout ≠ cancellation confirmation

Si vence un timeout:

```text
operation outcome
```

puede ser desconocido.

---

# 151. Database timeout

Un query timeout deberá utilizar la semántica definida en:

```text
DATABASE_QUERY_TIMEOUT_AND_CANCELLATION_SYSTEM
```

---

# 152. Connection lifecycle

Cada job deberá adquirir conexiones mediante:

```text
ConnectionManager
```

---

# 153. No connection serialization

Nunca:

```text
Job Payload
→
PDO / Connection object
```

---

# 154. Connection release

Al terminar el job:

```text
commit/rollback
close cursors
reset session state
reset tenant state
reset roles
reset isolation
release/discard connection
```

---

# 155. Dirty connection

Si no puede demostrarse limpia:

```text
DISCARD
```

---

# 156. Worker lifecycle

Workers son típicamente procesos persistentes.

Por tanto:

```text
Job 1 state
```

no deberá sobrevivir hacia:

```text
Job 2
```

---

# 157. State requiring reset

Incluye:

```text
EntityManager
UnitOfWork
IdentityMap
TransactionContext
DatabaseContext
TenantContext
AuthorizationContext
QueryContext
Connection session state
Temporary query state
Telemetry scope
```

---

# 158. Worker isolation

```text
Worker Process
   │
   ├── Job Scope A
   │    └── Database Scope A
   │
   ├── RESET
   │
   └── Job Scope B
        └── Database Scope B
```

---

# 159. Reset failure

Si el reset falla:

```text
Worker
→
QUARANTINE / RESTART
```

según severidad.

---

# 160. FrankenPHP

FrankenPHP es el runtime principal de VoltStack.

Sin embargo:

```text
HTTP worker lifecycle
```

y:

```text
Queue worker lifecycle
```

no deberán asumirse idénticos.

---

# 161. Queue workers under FrankenPHP ecosystem

La integración podrá reutilizar:

```text
Runtime Scope Manager
State Reset System
Connection Reuse System
Telemetry
Container
```

sin mezclar scopes HTTP y Queue.

---

# 162. RoadRunner

Un adapter futuro deberá garantizar:

```text
job isolation
connection reset
context reset
```

en workers persistentes.

---

# 163. OpenSwoole

Debe además considerar:

```text
coroutine isolation
```

---

# 164. Coroutine rule

Nunca almacenar:

```text
current job
current tenant
current entity manager
current transaction
```

en estado global compartido entre coroutines.

---

# 165. Concurrency

Un worker pool podrá procesar:

```text
N jobs
```

concurrentemente.

---

# 166. Database resource governance

La concurrencia Queue no deberá exceder ciegamente:

```text
DB connection capacity
```

---

# 167. Resource budget

Se propone:

```text
QueueDatabaseBudget
├── MaxConcurrentJobs
├── MaxConnections
├── MaxTransactions
├── QueryBudget
├── MemoryBudget
└── ExecutionDeadline
```

---

# 168. Backpressure

Si Database está saturado:

```text
Queue workers
```

deberán reducir/adaptar consumo según política.

---

# 169. Queue depth ≠ DB capacity

Miles de jobs pendientes no significan que deban ejecutarse simultáneamente.

---

# 170. Admission control

Puede existir:

```text
DatabaseAwareJobAdmissionController
```

---

# 171. Read/write routing

Jobs read-only podrán declarar:

```text
ReadIntent
```

---

# 172. Mutation jobs

Jobs con mutaciones deberán utilizar:

```text
Writer
```

---

# 173. Transaction pinning

Dentro de una transacción:

```text
connection
```

deberá permanecer correctamente fijada.

---

# 174. Replica lag

Un job que inmediatamente procesa datos recién creados puede requerir:

```text
writer
```

o:

```text
read-your-writes consistency
```

---

# 175. Sticky HTTP state ≠ job state

No deberá transportarse ciegamente:

```text
HTTP sticky connection state
```

a un worker.

---

# 176. Consistency requirement

El job podrá declarar:

```text
STRONG
READ_YOUR_WRITES
EVENTUAL
STALE_ALLOWED
```

según arquitectura.

---

# 177. Job query intent

Se propone:

```text
JobDatabaseIntent
├── READ_ONLY
├── MUTATING
├── TRANSACTIONAL
├── BULK
└── ADMINISTRATIVE
```

---

# 178. Sharding

Jobs deberán llevar suficiente información para:

```text
route deterministically
```

cuando sea posible.

---

# 179. Cross-shard jobs

Deberán ser explícitos.

---

# 180. No hidden distributed transaction

VoltStack no deberá fingir:

```text
Shard A transaction
+
Shard B transaction
=
atomic distributed transaction
```

sin infraestructura real.

---

# 181. Fan-out jobs

Para múltiples shards:

```text
Parent Job
   │
   ├── Child Shard A
   ├── Child Shard B
   └── Child Shard C
```

puede ser preferible.

---

# 182. Fan-in

Un coordinator podrá esperar resultados.

Pero:

```text
all child jobs completed
```

no implica una transacción distribuida.

---

# 183. Job batches

Se podrá modelar:

```text
JobBatch
├── BatchId
├── Jobs
├── State
├── Progress
├── FailurePolicy
└── CompletionPolicy
```

---

# 184. Batch states

```text
PENDING
RUNNING
PARTIALLY_COMPLETED
COMPLETED
FAILED
CANCELLED
```

---

# 185. Batch ≠ transaction

Un batch de 10,000 jobs no deberá ejecutarse dentro de una única transacción.

---

# 186. Batch progress

El progreso deberá persistirse de manera:

```text
idempotent
concurrency-safe
```

---

# 187. Scheduled jobs

Un job puede tener:

```text
available_at
```

---

# 188. Scheduling ≠ execution

```text
Scheduled for 10:00
```

no garantiza:

```text
executed exactly at 10:00
```

---

# 189. Scheduler time

Utilizar:

```text
Clock abstraction
```

para lógica.

Pero la medición real de tiempo operacional podrá requerir reloj monotónico.

---

# 190. Time zones

Persistir instantes programados en representación temporal canónica.

---

# 191. Priority

Queue priority no deberá romper:

```text
tenant fairness
resource budgets
```

sin política explícita.

---

# 192. Multi-tenant queue fairness

Un tenant con millones de jobs no debería necesariamente monopolizar todos los workers.

---

# 193. Fair scheduling

Podrán existir:

```text
per-tenant concurrency limits
weighted queues
fair leasing
rate limits
```

---

# 194. Queue name security

El nombre de queue deberá ser:

```text
typed / validated
```

no concatenado directamente desde input no confiable.

---

# 195. Payload security

Payloads deberán:

- validarse;
- versionarse;
- limitar tamaño;
- proteger datos sensibles;
- evitar deserialización insegura.

---

# 196. No arbitrary object deserialization

Nunca utilizar payloads que permitan instanciar clases arbitrarias proporcionadas por el mensaje.

---

# 197. Job type registry

Se propone:

```text
JobTypeRegistry
```

con mapping:

```text
stable job alias
→
handler
→
payload schema
```

---

# 198. Stable alias

Persistir:

```text
invoice.send
```

preferiblemente sobre:

```text
App\Jobs\Invoice\SendInvoiceJob
```

como identidad durable.

---

# 199. FQCN ≠ durable protocol

Renombrar una clase PHP no debería invalidar automáticamente millones de mensajes persistidos.

---

# 200. Registry freeze

El registry deberá congelarse después del bootstrap.

---

# 201. Outbox message model

Se propone:

```text
OutboxMessage
├── MessageId
├── Topic
├── MessageType
├── SchemaVersion
├── Payload
├── Headers
├── CreatedAt
├── AvailableAt
├── Attempts
├── Status
├── Lease?
└── CorrelationMetadata
```

---

# 202. Outbox payload

Deberá ser independiente de:

```text
EntityManager
UnitOfWork
Connection
Transaction object
```

---

# 203. Outbox insertion

El Persistence Engine podrá colaborar con:

```text
OutboxWriter
```

pero no deberá publicar directamente al broker.

---

# 204. Domain events

Un patrón posible:

```text
Entity
   ↓
Domain Event
   ↓
UoW
   ↓
Outbox Message
   ↓
COMMIT
```

---

# 205. Domain Event ≠ Queue Message

Puede existir transformación:

```text
Domain Event
→
Integration Message
→
Job
```

---

# 206. Event publication timing

Un evento que representa:

```text
committed state
```

no deberá publicarse antes del commit.

---

# 207. Outbox relay

El relay será responsable de:

```text
claim
publish
record outcome
retry
release
```

---

# 208. Relay concurrency

Múltiples relays deberán poder operar sin publicar el mismo registro innecesariamente, aunque duplicados seguirán siendo posibles en failure windows.

---

# 209. Relay leasing

Puede reutilizar conceptos:

```text
lease
fencing
SKIP LOCKED
atomic claim
```

capability-aware.

---

# 210. Relay cleanup

Mensajes publicados podrán:

```text
retain
archive
delete
```

según política.

---

# 211. Retention

La retención del outbox deberá evitar crecimiento ilimitado.

---

# 212. Cleanup safety

Nunca borrar:

```text
UNKNOWN
PENDING
LEASED
```

sólo por antigüedad sin política explícita.

---

# 213. Outbox partitioning

Para alto volumen podrán existir particiones por:

```text
time
tenant
topic
shard
```

según plataforma.

---

# 214. Outbox sharding

Si business data está shardado:

```text
outbox intent
```

deberá normalmente residir en la misma frontera transaccional que el cambio correspondiente.

---

# 215. Central outbox problem

Guardar business data en:

```text
Shard A
```

y outbox en:

```text
Central DB
```

reintroduce dual-write.

---

# 216. Therefore

Preferir:

```text
Shard-local Outbox
```

cuando se necesite atomicidad local.

---

# 217. Outbox relay routing

El relay deberá descubrir:

```text
which outboxes exist
```

sin asumir un único DB.

---

# 218. Database-backed queue transactional enqueue

Si:

```text
business tables
```

y:

```text
jobs table
```

están en la misma DB/transacción:

```text
Business Mutation
+
Job Insert
```

pueden ser atómicos localmente.

---

# 219. Advantage

En ese caso:

```text
Database Queue
```

puede evitar el dual-write de enqueue.

---

# 220. But

La ejecución del job sigue siendo:

```text
asynchronous
at-least-once capable
```

---

# 221. Queue driver abstraction

Se propone:

```php
interface QueueDriver
{
    public function publish(JobEnvelope $job): PublishOutcome;

    public function receive(ReceiveRequest $request): DeliveryBatch;

    public function acknowledge(Delivery $delivery): AckOutcome;

    public function release(
        Delivery $delivery,
        RetrySchedule $schedule
    ): ReleaseOutcome;
}
```

---

# 222. Database Queue adapter

```text
DatabaseQueueDriver
```

utilizará exclusivamente:

```text
Database public contracts
```

sin saltarse Query Engine/Execution boundaries salvo adapter interno justificado.

---

# 223. Queue Driver ≠ Database Driver

Debe mantenerse:

```text
QueueDriver
≠
DatabaseDriver
```

---

# 224. Job repository

Un:

```text
JobRepository
```

podrá administrar estado durable de jobs.

No será un ORM Repository de entidades de negocio necesariamente.

---

# 225. Job metadata transaction

Cambios de estado del job y business data podrán requerir coordinación.

Pero no deberán combinarse automáticamente si tienen semánticas diferentes.

---

# 226. Inbox pattern

Para consumidores de mensajes externos podrá utilizarse:

```text
Transactional Inbox
```

---

# 227. Inbox purpose

Registrar:

```text
MessageId already processed
```

dentro de la misma transacción que el efecto Database.

---

# 228. Inbox flow

```text
Receive M1
   ↓
BEGIN
   ↓
INSERT inbox(M1)
   ↓
Business Mutation
   ↓
COMMIT
   ↓
ACK
```

---

# 229. Duplicate

Si M1 llega otra vez:

```text
UNIQUE(MessageId)
```

permite detectar procesamiento previo.

---

# 230. Inbox ≠ Idempotency universal

No todos los efectos externos pueden protegerse únicamente con una tabla Inbox.

---

# 231. Inbox + Outbox

Patrón robusto:

```text
Incoming Message
      ↓
Inbox
      ↓
Business Mutation
      ↓
Outgoing Outbox
      ↓
same DB transaction
```

---

# 232. Exactly-once terminology

VoltStack deberá evitar prometer:

```text
exactly-once
```

salvo que se defina rigurosamente la frontera.

---

# 233. Preferred terminology

Utilizar:

```text
at-most-once delivery
at-least-once delivery
effectively-once effect
idempotent processing
deduplicated processing
```

con definiciones explícitas.

---

# 234. Effectively-once

Puede aproximarse:

```text
At-Least-Once Delivery
+
Durable Deduplication
+
Idempotent Effect
```

dentro de una frontera definida.

---

# 235. Exactly-once illusion

Nunca:

```text
broker says exactly once
→
all DB + external effects exactly once
```

---

# 236. Database failure classification

Jobs deberán distinguir:

```text
ConnectionFailure
Deadlock
SerializationFailure
LockTimeout
ConstraintViolation
QueryTimeout
ResourceExhaustion
UnknownTransactionOutcome
```

---

# 237. Failure ≠ retryable

Por ejemplo:

```text
UniqueConstraintViolation
```

puede ser:

```text
business conflict
```

no retry automático.

---

# 238. Retry classification

Se propone:

```text
RETRYABLE
NON_RETRYABLE
RECONCILIATION_REQUIRED
UNKNOWN
```

---

# 239. Poison jobs

Un job que falla determinísticamente no deberá ejecutarse indefinidamente.

---

# 240. Circuit breaker

Si Database está indisponible:

```text
workers
```

pueden activar:

```text
circuit breaker
```

o backpressure.

---

# 241. Retry storm

Evitar:

```text
10,000 jobs
×
immediate retry
```

contra un DB caído.

---

# 242. Global backoff

Podrá aplicarse:

```text
queue-level backoff
```

cuando el fallo sea sistémico.

---

# 243. Resource exhaustion

Si el pool de conexiones está saturado:

```text
wait bounded
backpressure
release job
```

según política.

---

# 244. No infinite connection wait

Un job deberá respetar:

```text
deadline
```

---

# 245. Telemetry architecture

Se medirán:

```text
queue.dispatch
queue.publish
queue.receive
queue.lease
job.execute
job.retry
job.fail
job.ack
outbox.persist
outbox.publish
inbox.claim
database.acquire
database.transaction
```

---

# 246. Correlation

Propagar:

```text
TraceId
CorrelationId
CausationId
MessageId
JobId
JobExecutionId
```

---

# 247. Trace propagation

El job podrá iniciar un nuevo span conectado causalmente al dispatch.

---

# 248. Trace ≠ execution context

No utilizar trace headers como autoridad de:

```text
tenant
actor
authorization
```

---

# 249. Metrics

Ejemplos:

```text
queue_depth
job_execution_duration
job_retry_count
job_failure_count
outbox_pending
outbox_publish_latency
database_connection_wait
job_database_transaction_duration
```

---

# 250. Metric cardinality

No utilizar:

```text
JobId
MessageId
TenantId
UserId
```

como labels no acotados.

---

# 251. Slow jobs

Un job lento deberá poder correlacionarse con:

```text
slow queries
N+1
connection wait
lock wait
external I/O
```

---

# 252. Audit

Operaciones sensibles deberán registrar:

```text
who/what initiated job
what job type
tenant
resource reference
authorization/delegation context
final outcome
```

sin datos innecesarios.

---

# 253. Job payload encryption

Payloads sensibles podrán requerir:

```text
encryption at rest
```

según queue/provider.

---

# 254. Encryption ≠ authorization

Cifrar un payload no concede acceso al handler.

---

# 255. Secret handling

No colocar credentials duraderas en payloads.

Preferir:

```text
SecretReference
```

resuelta durante ejecución.

---

# 256. Secret rotation

Jobs diferidos deberán utilizar la credencial vigente cuando sea apropiado.

---

# 257. Database credentials

El worker podrá tener un rol DB diferente del proceso HTTP.

---

# 258. Least privilege

Ejemplo:

```text
HTTP Runtime Role
Queue Worker Role
Migration Role
Admin Role
Backup Role
```

podrán ser distintos.

---

# 259. Worker privilege

Un worker no deberá recibir privilegios administrativos simplemente porque ejecuta background jobs.

---

# 260. Testing architecture

La integración requerirá:

```text
Unit Tests
Integration Tests
Queue Contract Tests
Database Integration Tests
Failure Tests
Concurrency Tests
Persistent Worker Tests
Security Tests
Performance Tests
```

---

# 261. Unit tests

Cubrir:

- envelope;
- payload validation;
- retry policy;
- backoff;
- job registry;
- idempotency decisions;
- outbox planning;
- context reconstruction;
- failure classification.

---

# 262. Integration tests

Utilizar:

```text
real database
```

para garantías de:

- transactions;
- leasing;
- locks;
- claims;
- idempotency;
- outbox;
- inbox.

---

# 263. Broker integration

Cuando la propiedad dependa del broker:

```text
real broker
```

será necesario.

Mocks no demuestran delivery real.

---

# 264. Database Queue matrix

Probar independientemente:

```text
MySQL
MariaDB
PostgreSQL
SQLite
```

según capabilities.

---

# 265. SQLite limitation

SQLite no deberá utilizarse como prueba universal del comportamiento concurrente de:

```text
MySQL/PostgreSQL/MariaDB
```

---

# 266. Outbox crash test A

```text
COMMIT business + outbox
↓
crash
↓
before publish
```

Al reiniciar:

```text
outbox remains publishable
```

---

# 267. Outbox crash test B

```text
publish
↓
crash
↓
before mark published
```

Debe tolerarse:

```text
duplicate publish
```

---

# 268. Consumer crash test

```text
business COMMIT
↓
crash
↓
before ACK
```

Redelivery deberá ser segura mediante idempotencia/deduplicación.

---

# 269. Unknown commit test

Provocar:

```text
COMMIT sent
connection lost
```

y verificar:

```text
UNKNOWN
```

sin retry ciego.

---

# 270. Lease expiry test

```text
Worker A claims
↓
lease expires
↓
Worker B claims
```

y comprobar comportamiento de fencing/idempotencia.

---

# 271. Concurrency tests

Utilizar:

```text
barriers
latches
explicit synchronization
```

No depender de `sleep()` para correctness.

---

# 272. Multi-worker test

Ejecutar:

```text
N workers
```

intentando reclamar los mismos jobs.

Cada lease deberá cumplir la semántica definida.

---

# 273. Tenant isolation test

```text
Job Tenant A
```

nunca deberá ejecutar accidentalmente con:

```text
DatabaseContext Tenant B
```

---

# 274. Persistent worker test

```text
Job A
Tenant A
Actor Alice

RESET

Job B
Tenant B
Actor Bob
```

Verificar ausencia total de contaminación.

---

# 275. Authorization test

Revocar permiso entre:

```text
dispatch
```

y:

```text
execution
```

y comprobar reauthorization cuando la política lo exija.

---

# 276. Retry test

Un deadlock deberá respetar:

```text
Transaction Retry Policy
```

sin provocar innecesariamente un:

```text
Job Retry
```

si la transacción se recupera.

---

# 277. Retry budget test

Verificar que retries de capas inferiores no provoquen multiplicación no acotada.

---

# 278. Performance tests

Medir:

- enqueue throughput;
- claim latency;
- lease contention;
- outbox relay throughput;
- DB connection pressure;
- worker throughput;
- batch scaling;
- payload serialization;
- cleanup;
- persistent worker memory.

---

# 279. Soak tests

Workers deberán ejecutar miles/millones de scopes para detectar:

```text
memory growth
state leakage
connection leakage
cursor leakage
transaction leakage
```

---

# 280. Proposed namespace

```text
VoltStack\Quantum\Database\Integration\Queue
```

---

# 281. Proposed directory structure

```text
src/Quantum/Database/Integration/Queue/
├── Contract/
│   ├── DatabaseJobDispatcher.php
│   ├── TransactionalJobDispatcher.php
│   ├── OutboxWriter.php
│   ├── InboxWriter.php
│   └── JobDatabaseContextResolver.php
│
├── Context/
│   ├── JobExecutionContext.php
│   ├── JobDatabaseContext.php
│   ├── JobDatabaseIntent.php
│   └── JobScope.php
│
├── Dispatch/
│   ├── DispatchMode.php
│   ├── ImmediateDispatcher.php
│   ├── AfterCommitDispatcher.php
│   ├── OutboxDispatcher.php
│   └── TransactionalDispatchCoordinator.php
│
├── Outbox/
│   ├── OutboxMessage.php
│   ├── OutboxMessageId.php
│   ├── OutboxStatus.php
│   ├── OutboxRepository.php
│   ├── OutboxRelay.php
│   ├── OutboxClaim.php
│   ├── OutboxLease.php
│   ├── OutboxPublisher.php
│   ├── OutboxRetryPolicy.php
│   └── OutboxCleanupPolicy.php
│
├── Inbox/
│   ├── InboxMessage.php
│   ├── InboxRepository.php
│   ├── InboxClaim.php
│   └── InboxDeduplicator.php
│
├── Idempotency/
│   ├── IdempotencyKey.php
│   ├── IdempotencyRecord.php
│   ├── IdempotencyRepository.php
│   ├── IdempotencyClaim.php
│   └── IdempotencyCoordinator.php
│
├── DatabaseQueue/
│   ├── DatabaseQueueDriver.php
│   ├── DatabaseQueueRepository.php
│   ├── DatabaseJobRecord.php
│   ├── DatabaseJobClaim.php
│   ├── DatabaseJobLease.php
│   ├── DatabaseJobLeaseManager.php
│   ├── DatabaseQueueCleaner.php
│   └── DatabaseQueueCapabilityResolver.php
│
├── Lease/
│   ├── JobLease.php
│   ├── LeaseId.php
│   ├── FencingToken.php
│   └── LeaseRenewalPolicy.php
│
├── Transaction/
│   ├── JobTransactionCoordinator.php
│   ├── AfterCommitJobRegistry.php
│   └── TransactionalJobPolicy.php
│
├── Retry/
│   ├── JobRetryPolicy.php
│   ├── RetryBudget.php
│   ├── RetryClassification.php
│   └── JobBackoffPolicy.php
│
├── Routing/
│   ├── JobTenantResolver.php
│   ├── JobShardResolver.php
│   └── JobReadWriteIntentResolver.php
│
├── Runtime/
│   ├── JobDatabaseScopeManager.php
│   ├── JobDatabaseStateResetter.php
│   └── PersistentWorkerDatabaseGuard.php
│
├── Telemetry/
│   └── QueueDatabaseTelemetry.php
│
├── Audit/
│   └── QueueDatabaseAuditBridge.php
│
└── Exception/
    ├── QueueDatabaseIntegrationException.php
    ├── JobContextResolutionException.php
    ├── JobLeaseException.php
    ├── JobIdempotencyException.php
    ├── OutboxException.php
    ├── InboxException.php
    └── UnknownJobDatabaseOutcomeException.php
```

---

# 282. Configuration example

```php
return [
    'queue' => [
        'default' => 'redis',

        'dispatch' => [
            'default_mode' => 'after_commit',
        ],

        'outbox' => [
            'enabled' => true,
            'batch_size' => 100,
            'max_attempts' => 20,
        ],

        'database_queue' => [
            'connection' => 'default',
            'table' => 'voltstack_jobs',
            'lease_seconds' => 60,
        ],

        'workers' => [
            'max_database_connections' => 20,
        ],
    ],
];
```

La configuración será normalizada mediante el sistema definido en:

```text
313_DATABASE_CONFIG_INTEGRATION_SYSTEM.md
```

---

# 283. Developer API — afterCommit

```php
Queue::afterCommit(
    new SendInvoiceJob(
        invoiceId: $invoice->id,
    )
);
```

---

# 284. Outbox API

```php
Outbox::dispatch(
    new InvoiceCreatedMessage(
        invoiceId: $invoice->id,
    )
);
```

Dentro de una transacción:

```php
Database::transaction(function () use ($invoice) {
    $invoice->save();

    Outbox::dispatch(
        new InvoiceCreatedMessage($invoice->id)
    );
});
```

---

# 285. Explicit dispatch mode

```php
Queue::dispatch(
    job: new ProcessOrder($orderId),
    mode: DispatchMode::AFTER_COMMIT,
);
```

---

# 286. Handler example

```php
final class ProcessOrderHandler
{
    public function __invoke(ProcessOrder $job): void
    {
        Database::transaction(function () use ($job) {
            $order = Order::query()
                ->whereKey($job->orderId)
                ->lockForUpdate()
                ->firstOrFail();

            if ($order->isProcessed()) {
                return;
            }

            $order->markProcessed();
            $order->save();
        });
    }
}
```

---

# 287. Idempotency example

```php
Idempotency::execute(
    key: "process-order:{$job->orderId}",
    callback: function () use ($job) {
        // durable business operation
    },
);
```

---

# 288. Context-aware handler

```php
final class GenerateTenantReportHandler
{
    public function __invoke(
        GenerateTenantReport $job,
        JobExecutionContext $context,
    ): void {
        $database = $context->database();

        // DatabaseContext already resolved for the job tenant.
    }
}
```

---

# 289. CLI

Podrán existir:

```text
php voltstack queue:work
php voltstack queue:status
php voltstack queue:failed
php voltstack queue:retry
php voltstack queue:replay
php voltstack queue:prune
php voltstack database:outbox:relay
php voltstack database:outbox:status
php voltstack database:outbox:prune
php voltstack database:queue:diagnose
```

---

# 290. Safety of replay commands

Comandos:

```text
retry
replay
```

deberán mostrar:

- job type;
- attempts;
- tenant;
- failure classification;
- idempotency status;
- payload version;

antes de operaciones sensibles cuando corresponda.

---

# 291. Core invariants

## DB-QUEUE-INT-001

Database Transaction ≠ Queue Transaction.

## DB-QUEUE-INT-002

Database Commit ≠ Queue Publish.

## DB-QUEUE-INT-003

afterCommit ≠ Distributed Transaction.

## DB-QUEUE-INT-004

Outbox ≠ Queue.

## DB-QUEUE-INT-005

Database Queue ≠ Outbox.

## DB-QUEUE-INT-006

Job ≠ Transaction.

## DB-QUEUE-INT-007

Job Payload ≠ Entity.

## DB-QUEUE-INT-008

Queue Driver ≠ Database Driver.

## DB-QUEUE-INT-009

Job Dispatched ≠ Job Executed.

## DB-QUEUE-INT-010

Job Executed ≠ Job Completed.

---

# 292. Delivery invariants

## DB-QUEUE-INT-011

Delivery ≠ exactly-once execution.

## DB-QUEUE-INT-012

Lease ≠ permanent ownership.

## DB-QUEUE-INT-013

ACK ≠ Database Commit.

## DB-QUEUE-INT-014

ACK ocurrirá después del commit requerido.

## DB-QUEUE-INT-015

ACK failure podrá producir redelivery.

## DB-QUEUE-INT-016

Redelivery deberá asumirse posible.

## DB-QUEUE-INT-017

MessageId será estable entre redeliveries.

## DB-QUEUE-INT-018

DeliveryId podrá cambiar.

## DB-QUEUE-INT-019

UNKNOWN publish ≠ failed publish.

## DB-QUEUE-INT-020

Timeout ≠ proof of non-delivery.

---

# 293. Transaction invariants

## DB-QUEUE-INT-021

afterCommit sólo seguirá a commit confirmado.

## DB-QUEUE-INT-022

Rollback cancelará pending afterCommit dispatches.

## DB-QUEUE-INT-023

Savepoint release ≠ outer commit.

## DB-QUEUE-INT-024

Outbox business state e intent compartirán transacción cuando se prometa atomicidad.

## DB-QUEUE-INT-025

External broker publish no formará parte del DB commit por defecto.

## DB-QUEUE-INT-026

Unknown DB commit permanecerá UNKNOWN.

## DB-QUEUE-INT-027

UNKNOWN no será retry ciego.

## DB-QUEUE-INT-028

Job retry ≠ transaction retry.

## DB-QUEUE-INT-029

Transaction retry ≠ statement retry.

## DB-QUEUE-INT-030

Rollback ≠ external side-effect rollback.

---

# 294. Idempotency invariants

## DB-QUEUE-INT-031

At-least-once delivery asumirá duplicados posibles.

## DB-QUEUE-INT-032

Idempotency será durable cuando la garantía lo requiera.

## DB-QUEUE-INT-033

Check-then-insert ingenuo no será mecanismo suficiente.

## DB-QUEUE-INT-034

Unique constraints podrán reforzar claims.

## DB-QUEUE-INT-035

Idempotency Key tendrá scope explícito.

## DB-QUEUE-INT-036

Retry del mismo efecto preservará su idempotency identity.

## DB-QUEUE-INT-037

Duplicate Delivery ≠ Duplicate Effect necesariamente.

## DB-QUEUE-INT-038

UNKNOWN effect requerirá reconciliation.

## DB-QUEUE-INT-039

Inbox podrá deduplicar incoming messages.

## DB-QUEUE-INT-040

Exactly-once no será prometido sin frontera rigurosa.

---

# 295. Payload invariants

## DB-QUEUE-INT-041

Managed Entity no será payload durable por defecto.

## DB-QUEUE-INT-042

Connection no será serializable en job.

## DB-QUEUE-INT-043

EntityManager no será serializable en job.

## DB-QUEUE-INT-044

UnitOfWork no será serializable en job.

## DB-QUEUE-INT-045

Payload será versionado.

## DB-QUEUE-INT-046

Job types utilizarán alias estable.

## DB-QUEUE-INT-047

FQCN ≠ durable job protocol.

## DB-QUEUE-INT-048

Payload size será bounded.

## DB-QUEUE-INT-049

Arbitrary object deserialization estará prohibida.

## DB-QUEUE-INT-050

Secrets duraderos no se incluirán en payloads.

---

# 296. Context invariants

## DB-QUEUE-INT-051

Cada job tendrá fresh DatabaseContext.

## DB-QUEUE-INT-052

TenantReference ≠ trusted TenantContext.

## DB-QUEUE-INT-053

ShardHint ≠ routing authority.

## DB-QUEUE-INT-054

ActorReference ≠ authenticated session.

## DB-QUEUE-INT-055

Authorized at dispatch ≠ authorized at execution.

## DB-QUEUE-INT-056

Cross-tenant jobs serán explícitos.

## DB-QUEUE-INT-057

Cross-shard jobs serán explícitos.

## DB-QUEUE-INT-058

Delegated authority será explícita.

## DB-QUEUE-INT-059

Delegated authority tendrá scope.

## DB-QUEUE-INT-060

Delegated authority tendrá lifetime definido.

---

# 297. Worker invariants

## DB-QUEUE-INT-061

Job scopes estarán aislados.

## DB-QUEUE-INT-062

IdentityMap no sobrevivirá entre jobs.

## DB-QUEUE-INT-063

UnitOfWork no sobrevivirá entre jobs.

## DB-QUEUE-INT-064

TransactionContext no sobrevivirá entre jobs.

## DB-QUEUE-INT-065

TenantContext no sobrevivirá entre jobs.

## DB-QUEUE-INT-066

AuthorizationContext no sobrevivirá entre jobs.

## DB-QUEUE-INT-067

Dirty connections no volverán al pool.

## DB-QUEUE-INT-068

Failed reset podrá descartar connection.

## DB-QUEUE-INT-069

Severe reset failure podrá reiniciar/quarantine worker.

## DB-QUEUE-INT-070

OpenSwoole coroutine state estará aislado.

---

# 298. Database queue invariants

## DB-QUEUE-INT-071

Job claim será atómico.

## DB-QUEUE-INT-072

SELECT then UPDATE sin protección no será claim válido.

## DB-QUEUE-INT-073

Claim strategy será capability-aware.

## DB-QUEUE-INT-074

Lease expiration permitirá redelivery.

## DB-QUEUE-INT-075

Lease expiration podrá coexistir con ejecución anterior.

## DB-QUEUE-INT-076

Lease ≠ exactly-once.

## DB-QUEUE-INT-077

Fencing podrá utilizarse para operaciones sensibles.

## DB-QUEUE-INT-078

Heartbeat ≠ business completion.

## DB-QUEUE-INT-079

Database Queue cleanup será ownership-safe.

## DB-QUEUE-INT-080

Queue tables no crecerán sin política de retención.

---

# 299. Outbox invariants

## DB-QUEUE-INT-081

Outbox intent será durable antes de relay.

## DB-QUEUE-INT-082

Outbox relay asumirá duplicate publish possible.

## DB-QUEUE-INT-083

Outbox MessageId será estable.

## DB-QUEUE-INT-084

PUBLISHED sólo se marcará con evidencia suficiente.

## DB-QUEUE-INT-085

UNKNOWN ≠ PUBLISHED.

## DB-QUEUE-INT-086

UNKNOWN ≠ FAILED.

## DB-QUEUE-INT-087

Outbox cleanup no eliminará estado incierto arbitrariamente.

## DB-QUEUE-INT-088

Shard-local business mutation usará shard-local outbox cuando se requiera atomicidad local.

## DB-QUEUE-INT-089

Central outbox separado puede reintroducir dual-write.

## DB-QUEUE-INT-090

Outbox relay será retryable e idempotency-aware.

---

# 300. Retry invariants

## DB-QUEUE-INT-091

Retry ownership será explícito.

## DB-QUEUE-INT-092

Retries de capas diferentes no se multiplicarán sin control.

## DB-QUEUE-INT-093

RetryBudget será bounded.

## DB-QUEUE-INT-094

Non-retryable failures no serán reintentados automáticamente.

## DB-QUEUE-INT-095

Systemic failures usarán backoff.

## DB-QUEUE-INT-096

Retry storm deberá prevenirse.

## DB-QUEUE-INT-097

Deadlock podrá resolverse dentro de Transaction Retry.

## DB-QUEUE-INT-098

Unknown transaction outcome requerirá reconciliation.

## DB-QUEUE-INT-099

Poison jobs llegarán a final failure/DLQ.

## DB-QUEUE-INT-100

Replay será operación explícita.

---

# 301. Resource invariants

## DB-QUEUE-INT-101

Queue concurrency respetará DB capacity.

## DB-QUEUE-INT-102

Connection acquisition será bounded.

## DB-QUEUE-INT-103

Job execution tendrá deadline cuando corresponda.

## DB-QUEUE-INT-104

Backpressure será soportado.

## DB-QUEUE-INT-105

Queue depth ≠ permitted concurrency.

## DB-QUEUE-INT-106

Database saturation no provocará unlimited retries.

## DB-QUEUE-INT-107

Long jobs no mantendrán transacciones abiertas innecesariamente.

## DB-QUEUE-INT-108

Streams/cursors serán cerrados.

## DB-QUEUE-INT-109

Connection leaks serán detectables.

## DB-QUEUE-INT-110

Memory growth en persistent workers será medible.

---

# 302. Security invariants

## DB-QUEUE-INT-111

Payload input será validado.

## DB-QUEUE-INT-112

Job type será resuelto desde registry.

## DB-QUEUE-INT-113

Unknown job type no será instanciado arbitrariamente.

## DB-QUEUE-INT-114

Unknown payload version no será ejecutada parcialmente.

## DB-QUEUE-INT-115

Secrets serán resueltos en ejecución cuando corresponda.

## DB-QUEUE-INT-116

Worker DB role seguirá least privilege.

## DB-QUEUE-INT-117

Failed-job diagnostics serán redactados.

## DB-QUEUE-INT-118

DLQ replay será autorizado.

## DB-QUEUE-INT-119

Tenant metadata será validada.

## DB-QUEUE-INT-120

Trace metadata no será security authority.

---

# 303. Observability invariants

## DB-QUEUE-INT-121

JobId ≠ TraceId.

## DB-QUEUE-INT-122

MessageId ≠ CorrelationId.

## DB-QUEUE-INT-123

Telemetry no cambiará job semantics.

## DB-QUEUE-INT-124

Queue metrics evitarán cardinalidad no acotada.

## DB-QUEUE-INT-125

Slow query podrá correlacionarse con slow job.

## DB-QUEUE-INT-126

Retry reasons serán observables.

## DB-QUEUE-INT-127

Outbox backlog será observable.

## DB-QUEUE-INT-128

Connection pressure será observable.

## DB-QUEUE-INT-129

Lease contention será observable.

## DB-QUEUE-INT-130

Unknown outcomes serán visibles y no normalizados como failures ordinarios.

---

# 304. Testing invariants

## DB-QUEUE-INT-131

Mocks no demostrarán broker delivery real.

## DB-QUEUE-INT-132

Fakes no demostrarán DB locking real.

## DB-QUEUE-INT-133

SQLite no sustituirá conformance de MySQL/PostgreSQL/MariaDB.

## DB-QUEUE-INT-134

Crash windows serán probados explícitamente.

## DB-QUEUE-INT-135

Concurrency tests utilizarán sincronización determinista.

## DB-QUEUE-INT-136

Persistent workers serán sometidos a leak tests.

## DB-QUEUE-INT-137

Duplicate delivery será probado.

## DB-QUEUE-INT-138

Unknown commit será probado.

## DB-QUEUE-INT-139

Lease expiration race será probada.

## DB-QUEUE-INT-140

Tenant contamination será probada.

---

# 305. Additional invariants

## DB-QUEUE-INT-141

Batch ≠ transaction.

## DB-QUEUE-INT-142

Schedule time ≠ exact execution time.

## DB-QUEUE-INT-143

Cancellation ≠ rollback.

## DB-QUEUE-INT-144

Timeout ≠ cancellation confirmation.

## DB-QUEUE-INT-145

External side effect ≠ DB state.

## DB-QUEUE-INT-146

Domain Event ≠ Queue Message.

## DB-QUEUE-INT-147

Inbox ≠ Outbox.

## DB-QUEUE-INT-148

Inbox ≠ universal idempotency solution.

## DB-QUEUE-INT-149

Database Queue transactional enqueue sólo será atómico dentro de su misma DB transaction.

## DB-QUEUE-INT-150

No se fabricará atomicidad entre fronteras independientes.

---

# 306. Anti-pattern: dispatch before commit

```php
Database::transaction(function () use ($order) {
    $order->save();

    Queue::dispatch(
        new ProcessOrder($order->id)
    );
});
```

si `dispatch()` publica inmediatamente.

El worker podría ejecutar antes del commit.

---

# 307. Anti-pattern: afterCommit as guaranteed delivery

Incorrecto:

```text
afterCommit
=
guaranteed durable publication
```

No lo es.

---

# 308. Anti-pattern: serialize entity

```php
Queue::dispatch(
    new Job($managedEntity)
);
```

puede transportar estado obsoleto y dependencias ORM.

---

# 309. Anti-pattern: retry unknown commit

Nunca:

```text
COMMIT
connection lost
↓
retry whole mutation blindly
```

---

# 310. Anti-pattern: ACK before commit

```text
ACK
↓
DB COMMIT
```

puede perder trabajo.

---

# 311. Anti-pattern: assume exactly once

```text
broker configuration
→
business effect exactly once
```

es una conclusión inválida.

---

# 312. Anti-pattern: naive lease

```text
status = running
```

sin lease expiration/generation no es suficiente para muchos escenarios distribuidos.

---

# 313. Anti-pattern: queue concurrency equals CPU count

El límite real puede estar determinado por:

```text
DB connections
locks
I/O
memory
external APIs
```

---

# 314. Anti-pattern: one giant job transaction

```text
BEGIN
process 1,000,000 records
COMMIT
```

puede producir:

- long locks;
- huge rollback;
- memory growth;
- replication pressure;
- timeout;
- operational risk.

---

# 315. Anti-pattern: one row per transaction blindly

El extremo contrario también puede destruir throughput.

La granularidad será workload-specific.

---

# 316. Anti-pattern: sleep-based worker coordination

Tests o workers no deberán usar `sleep()` como mecanismo de exclusión/correctness.

---

# 317. Anti-pattern: global tenant

Nunca:

```php
CurrentTenant::$id = $job->tenantId;
```

en persistent workers.

---

# 318. Anti-pattern: infinite retries

```text
while failure:
    retry
```

está prohibido.

---

# 319. Anti-pattern: DLQ as garbage bin

La Dead-Letter Queue deberá ser operable, observable y recuperable.

---

# 320. Anti-pattern: retry every exception

No todo fallo es transitorio.

---

# 321. Anti-pattern: same credentials everywhere

HTTP, workers, migrations y administración no deberán compartir obligatoriamente un superuser DB.

---

# 322. Formal dual-write model

Sean:

```text
D = durable database commit
Q = durable queue publish
```

Una operación distribuida deseada sería:

```text
D ∧ Q
```

Pero sin coordinador transaccional:

```text
Commit(D)
```

y:

```text
Publish(Q)
```

son eventos independientes.

Por tanto pueden existir:

```text
D ∧ ¬Q
```

o:

```text
¬D ∧ Q
```

---

# 323. afterCommit model

`afterCommit` reduce:

```text
¬D ∧ Q
```

pero todavía permite:

```text
D ∧ ¬Q
```

por crash/fallo posterior.

---

# 324. Outbox model

Sea:

```text
B = Business Change
O = Outbox Intent
```

si ambos comparten transacción:

```text
Commit(B ∧ O)
```

entonces:

```text
B
```

y:

```text
O
```

tienen atomicidad local Database.

La publicación:

```text
O → Q
```

sigue siendo asincrónica y potencialmente duplicable.

---

# 325. Consumer model

Sea:

```text
M = Message
I = Inbox Claim
E = Business Effect
```

Una estrategia robusta puede ejecutar:

```text
Commit(I ∧ E)
```

y posteriormente:

```text
ACK(M)
```

---

# 326. Effective-once model

Dentro de una frontera bien definida:

```text
EffectivelyOnce
≈
AtLeastOnceDelivery
∧
StableMessageIdentity
∧
DurableDeduplication
∧
IdempotentEffect
∧
AtomicClaimAndEffect
```

cuando sea aplicable.

---

# 327. Job execution correctness

```text
JobExecutionCorrectness
=
ValidPayload
∧
ValidJobVersion
∧
CorrectTenant
∧
CorrectShard
∧
CorrectActorContext
∧
CorrectDatabaseContext
∧
SafeTransactionBoundary
∧
Idempotency
∧
OutcomeEvidence
∧
CleanRuntimeReset
```

---

# 328. Retry correctness

```text
SafeRetry
=
RetryableFailure
∧
KnownSafeBoundary
∧
RemainingRetryBudget
∧
NoUnreconciledUnknownOutcome
∧
IdempotentOrCompensatedEffect
```

---

# 329. Outbox correctness

```text
OutboxCorrectness
=
AtomicIntentPersistence
∧
StableMessageIdentity
∧
SafeClaiming
∧
RetryablePublication
∧
DuplicateTolerance
∧
OutcomeTracking
∧
SafeRetention
```

---

# 330. Database queue correctness

```text
DatabaseQueueCorrectness
=
AtomicClaim
∧
LeaseSemantics
∧
DuplicateTolerance
∧
SafeACK
∧
RetryPolicy
∧
FailureIsolation
∧
ResourceGovernance
```

---

# 331. Recommended default architecture

Para VoltStack V1 se recomienda:

```text
Application
    │
    ├── Non-critical async work
    │       ↓
    │   AFTER_COMMIT
    │
    └── Reliable integration work
            ↓
    TRANSACTIONAL OUTBOX
            ↓
       Outbox Relay
            ↓
       Queue Provider
            ↓
          Worker
            ↓
       Fresh Job Scope
            ↓
       DatabaseContext
            ↓
      Idempotent Handler
```

---

# 332. Why two modes

`AFTER_COMMIT` es adecuado cuando:

```text
occasional lost dispatch
```

puede ser tolerado o reconciliado fácilmente.

`OUTBOX` es preferible cuando:

```text
committed business change
```

debe producir de forma durable trabajo/evento posterior.

---

# 333. Default safety policy

VoltStack no deberá activar automáticamente Outbox para todos los jobs.

El desarrollador podrá elegir:

```text
IMMEDIATE
AFTER_COMMIT
OUTBOX
```

con defaults documentados.

---

# 334. Suggested default

Dentro de una transacción activa:

```text
Queue::dispatch()
```

podrá:

1. advertir;
2. requerir modo explícito; o
3. utilizar `AFTER_COMMIT` por configuración.

La opción elegida deberá ser visible y no mágica.

---

# 335. Strong recommendation

Para evitar ambigüedad, la API pública deberá favorecer:

```php
Queue::dispatch(...)
Queue::afterCommit(...)
Outbox::dispatch(...)
```

como intenciones claramente diferentes.

---

# 336. V1 scope

VoltStack V1 deberá incluir:

```text
JobEnvelope
stable JobType registry
payload versioning
EntityReference
JobExecutionContext
fresh DatabaseContext per job
tenant propagation
actor reference propagation
IMMEDIATE dispatch integration
AFTER_COMMIT dispatch
transaction-aware pending dispatch registry
Transactional Outbox
Outbox Relay
stable MessageId
idempotency contracts
Database-backed idempotency store
Inbox contracts
Database Queue adapter
atomic job claim
leases
visibility timeout
retry policies
retry budgets
failure classification
failed jobs
DLQ integration contracts
connection cleanup
persistent worker reset
FrankenPHP worker integration
telemetry
audit
unit tests
real DB integration tests
concurrency tests
crash-window tests
```

---

# 337. V2

Podrá incorporar:

```text
fencing tokens
advanced lease renewal
adaptive DB-aware worker concurrency
multi-tenant fair scheduling
advanced batch jobs
shard-local outbox relays
outbox partitioning
payload encryption policies
advanced inbox/outbox orchestration
reconciliation engine
operational dashboard
RoadRunner queue integration
OpenSwoole coroutine workers
```

---

# 338. V3

Podrá incorporar:

```text
distributed queue orchestration
multi-region outbox relays
global deduplication strategies
distributed scheduling
advanced workflow orchestration
sagas
compensation plans
cross-shard workflow coordination
adaptive retry control
capacity-aware scheduling
```

sin fingir:

```text
global ACID transaction
```

donde no exista.

---

# 339. Final architecture

```text
                    VoltStack Application
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
      IMMEDIATE        AFTER_COMMIT         OUTBOX
          │                 │                 │
          │          Transaction Manager      │
          │                 │                 │
          │                 ▼                 ▼
          │             DB COMMIT      Business Data
          │                 │           + Outbox Intent
          │                 │                 │
          │                 │                 ▼
          │                 │             DB COMMIT
          │                 │                 │
          │                 │                 ▼
          │                 │            Outbox Relay
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                       Queue Driver
                            │
                            ▼
                       Queue Broker
                            │
                            ▼
                         Delivery
                            │
                            ▼
                      Worker Runtime
                            │
                            ▼
                    JobExecutionScope
                            │
           ┌────────────────┼────────────────┐
           ▼                ▼                ▼
       TenantContext   ActorContext    DatabaseContext
           │                │                │
           └────────────────┼────────────────┘
                            ▼
                        Job Handler
                            │
                            ▼
                   Transaction Manager
                            │
                            ▼
                         Database
                            │
                    ┌───────┴────────┐
                    ▼                ▼
               COMMIT known     Outcome UNKNOWN
                    │                │
                    ▼                ▼
              Idempotency       Reconciliation
                    │
                    ▼
                   ACK
```

---

# 340. Regla definitiva de dispatch

> **El acto de solicitar la ejecución de un Job y el acto de confirmar una transacción Database serán tratados como operaciones independientes salvo que una estrategia explícita —como una Database Queue transaccional o Transactional Outbox— establezca una frontera durable común para la intención correspondiente.**

---

# 341. Regla definitiva de ejecución

> **Todo Job deberá ejecutarse dentro de un scope nuevo y aislado, reconstruyendo explícitamente su `DatabaseContext`, tenant, shard, actor y demás contexto permitido, sin reutilizar estado mutable procedente del request, job o coroutine anterior.**

---

# 342. Regla definitiva de entrega

> **VoltStack diseñará sus workers suponiendo que un mensaje puede ser entregado más de una vez, que un ACK puede perderse, que un lease puede expirar y que un worker puede morir en cualquier punto de la ejecución.**

Por tanto:

```text
Reliable Jobs
=
Durable Intent
+
Stable Identity
+
Idempotency
+
Safe Transaction Boundaries
+
Retry Discipline
+
Outcome Evidence
+
Runtime Isolation
```

---

# 343. Regla definitiva de incertidumbre

> **Cuando VoltStack no pueda demostrar si una transacción, publicación o efecto fue completado, conservará el resultado como `UNKNOWN` y utilizará reconciliación o evidencia durable antes de repetir una operación potencialmente no idempotente.**

Nunca:

```text
UNKNOWN
→
assume failure
→
blind retry
```

---

# 344. Regla definitiva de entidades

> **Las entidades administradas por el ORM pertenecen al scope del EntityManager que las administra y no constituyen un protocolo durable de comunicación entre procesos. Los Jobs deberán transportar referencias o DTOs explícitos y reconstruir el estado requerido dentro de su propio scope.**

Por tanto:

```text
Managed Entity
≠
Job Payload
```

---

# 345. Regla definitiva de Outbox

> **Transactional Outbox será el mecanismo oficial de VoltStack para convertir una intención producida dentro de una transacción Database en trabajo o integración asincrónica durable cuando `afterCommit` no proporcione garantías suficientes.**

Pero:

```text
Outbox
≠
Exactly Once
```

por lo que deberá combinarse con:

```text
Stable Message Identity
+
Idempotent Consumer
+
Deduplication
+
Reconciliation
```

según el nivel de garantía requerido.

---

# 346. Invariante final

La arquitectura completa deberá preservar:

```text
Database Commit
≠
Queue Publish
```

```text
Queue Publish
≠
Queue Delivery
```

```text
Queue Delivery
≠
Job Execution
```

```text
Job Execution
≠
Database Commit
```

```text
Database Commit
≠
Queue ACK
```

```text
Queue ACK
≠
External Side Effect
```

y:

```text
Retry
≠
Proof Previous Attempt Failed
```

---

# 347. Siguiente documento

```text
321_DATABASE_HTTP_REQUEST_LIFECYCLE_INTEGRATION.md
```

Este documento cerrará el bloque de integración del framework definiendo la relación:

```text
HTTP Request
      ↓
VoltStack HttpKernel
      ↓
Request Scope
      ↓
DatabaseContext
      ↓
Connection / Transaction / ORM
      ↓
Response
      ↓
Database State Reset
```

y deberá formalizar especialmente:

```text
HTTP Request
≠
Database Transaction
```

```text
Request Scope
≠
Worker Scope
```

```text
Response Sent
≠
Database Commit
```

```text
Connection Reuse
≠
State Reuse
```

junto con:

- bootstrap Database por request;
- lazy connection acquisition;
- `DatabaseContext`;
- tenant resolution;
- authentication/authorization propagation;
- EntityManager scope;
- UnitOfWork;
- IdentityMap;
- transaction ownership;
- streaming responses;
- exception handling;
- disconnects;
- cancellation;
- `afterCommit`;
- response lifecycle;
- state reset;
- FrankenPHP;
- persistent workers;
- RoadRunner;
- OpenSwoole;
- concurrent requests;
- telemetry;
- security;
- testing;
- leak detection.