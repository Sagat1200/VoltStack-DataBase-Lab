# 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md

# VoltStack Quantum Database
## Database Transaction Event Pipeline

**Proyecto:** VoltStack Framework  
**Subsistema:** `VoltStack/Quantum/Database`  
**Documento:** 212 — Database Transaction Event Pipeline  
**Bloque:** 20 — Events  
**Estado:** Architecture Specification  
**Versión:** 1.0  
**Documento anterior:** `211_DATABASE_CONNECTION_EVENT_SYSTEM.md`  
**Siguiente documento:** `213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md`

---

# 1. Propósito

`Database Transaction Event Pipeline` define cómo VoltStack representará, correlacionará y distribuirá eventos relacionados con el ciclo de vida de una transacción.

El pipeline cubrirá:

```text
Transaction Request
↓
Transaction Context
↓
Begin Attempt
↓
ACTIVE
↓
Queries / Flushes
↓
Nested Scopes / Savepoints
↓
Commit Attempt
   ├── COMMITTED
   ├── FAILED
   └── UNKNOWN
or
Rollback Attempt
   ├── ROLLED_BACK
   ├── FAILED
   └── UNKNOWN
↓
Deferred Event Processing
↓
AfterCommit / AfterRollback / AfterCompletion
```

La regla central será:

> **Un Transaction Event describe un hecho, intento o outcome observado dentro del ciclo transaccional; nunca redefine el estado real de la transacción ni convierte un intento de commit, un `flush()`, un savepoint o el éxito de una query en un commit confirmado.**

---

# 2. Relación con la arquitectura existente

Este documento especializa:

```text
164_DATABASE_TRANSACTION_ARCHITECTURE.md
165_DATABASE_TRANSACTION_MANAGER_SYSTEM.md
166_DATABASE_TRANSACTION_CONTEXT_SYSTEM.md
167_DATABASE_TRANSACTION_ISOLATION_SYSTEM.md
168_DATABASE_NESTED_TRANSACTION_SYSTEM.md
169_DATABASE_SAVEPOINT_SYSTEM.md
170_DATABASE_TRANSACTION_RETRY_SYSTEM.md
171_DATABASE_DEADLOCK_HANDLING_SYSTEM.md
172_DATABASE_OPTIMISTIC_LOCKING_SYSTEM.md
173_DATABASE_PESSIMISTIC_LOCKING_SYSTEM.md
174_DATABASE_CONCURRENCY_CONTROL_SYSTEM.md
175_DATABASE_TRANSACTION_EVENT_SYSTEM.md
209_DATABASE_EVENT_ARCHITECTURE.md
210_DATABASE_QUERY_EVENT_SYSTEM.md
211_DATABASE_CONNECTION_EVENT_SYSTEM.md
```

`175_DATABASE_TRANSACTION_EVENT_SYSTEM.md` define las capacidades transaccionales generales.

El presente documento define específicamente:

```text
event ordering
event phases
event correlation
attempt boundaries
deferred delivery
afterCommit
afterRollback
afterCompletion
nested transaction behavior
savepoint event behavior
retry correlation
UNKNOWN outcomes
listener failure semantics
```

---

# 3. Objetivos

El sistema deberá permitir observar:

```text
transaction start
begin success/failure
transaction attempts
nested scopes
savepoints
query correlation
flush correlation
commit attempts
confirmed commits
rollback attempts
confirmed rollbacks
deadlocks
retries
retry exhaustion
unknown outcomes
transaction taint
connection correlation
deferred events
afterCommit delivery
afterRollback delivery
afterCompletion delivery
```

sin romper los límites entre:

```text
Transaction Manager
Connection Manager
Query Engine
ORM
Persistence Engine
Event System
Telemetry
```

---

# 4. Distinciones fundamentales

VoltStack deberá preservar:

```text
Transaction
≠
Transaction Event
≠
Transaction Context
≠
Transaction Attempt
≠
Connection
≠
Query
≠
UnitOfWork
≠
Flush
≠
Savepoint
≠
Nested Scope
≠
Event Transaction
```

---

# 5. Flush ≠ Commit

Esta será una de las invariantes más importantes.

```text
EntityManager::flush()
```

puede producir:

```text
INSERT
UPDATE
DELETE
```

pero:

```text
flush()
≠
commit()
```

Ejemplo:

```text
BEGIN

flush()
↓
INSERT succeeds

flush()
↓
UPDATE succeeds

ROLLBACK
```

Ningún `flush()` implicó commit.

---

# 6. QueryExecuted ≠ TransactionCommitted

Ejemplo:

```text
BEGIN
↓
UPDATE
↓
QueryExecuted
↓
DELETE
↓
QueryExecuted
↓
ROLLBACK
```

Ambas queries fueron ejecutadas exitosamente.

La transacción no fue committed.

---

# 7. Commit requested ≠ Commit confirmed

```text
commit()
```

es una solicitud.

El resultado puede ser:

```text
COMMITTED
FAILED
UNKNOWN
```

---

# 8. TransactionCommitStarted ≠ TransactionCommitted

Nunca deberán considerarse equivalentes.

```text
TransactionCommitStarted
↓
connection lost
↓
TransactionOutcomeUnknown
```

es perfectamente válido.

---

# 9. Rollback requested ≠ Rollback confirmed

También:

```text
TransactionRollbackStarted
≠
TransactionRolledBack
```

---

# 10. Savepoint ≠ Transaction

Un savepoint representa un boundary interno.

```text
SAVEPOINT s1
```

no crea una nueva transacción física.

---

# 11. Savepoint release ≠ Commit

```text
RELEASE SAVEPOINT s1
```

no implica:

```text
COMMIT
```

---

# 12. Nested scope ≠ Independent physical transaction

Un scope lógico anidado puede usar:

```text
JOIN
SAVEPOINT
REJECT
REQUIRES_NEW
```

según política.

Por tanto:

```text
NestedScopeCompleted
≠
TransactionCommitted
```

---

# 13. Transaction retry ≠ Query retry

Si una transacción falla por deadlock:

```text
BEGIN
Q1
Q2
Q3 → DEADLOCK
```

el boundary seguro puede ser:

```text
rollback entire transaction
↓
replay entire transaction
```

no:

```text
retry Q3 only
```

---

# 14. TransactionEvent contract

```php
interface TransactionEvent extends DatabaseEvent
{
    public function transactionId(): TransactionId;

    public function transactionContext(): TransactionEventContext;
}
```

---

# 15. TransactionId

Representará la identidad de la operación transaccional lógica.

Una transacción lógica con retries conserva:

```text
TransactionId
```

pero puede tener múltiples:

```text
TransactionAttemptId
```

---

# 16. TransactionId ≠ TransactionAttemptId

Ejemplo:

```text
Transaction T1
│
├── Attempt A1
│   └── DEADLOCK
│
└── Attempt A2
    └── COMMITTED
```

---

# 17. TransactionAttemptId

```php
final readonly class TransactionAttemptId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 18. TransactionScopeId

Nested scopes necesitarán identidad independiente.

```text
TransactionId
    ↓
TransactionAttemptId
    ↓
TransactionScopeId
```

---

# 19. SavepointId

Los savepoints deberán usar identidad lógica separada de su nombre físico.

```php
final readonly class SavepointId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 20. SavepointId ≠ SQL savepoint name

El nombre compilado podría ser:

```text
vs_sp_0007
```

pero las integraciones deberían usar:

```text
SavepointId
```

estable dentro del contexto.

---

# 21. Correlation hierarchy

```text
RequestId
↓
OperationId
↓
TransactionId
↓
TransactionAttemptId
↓
TransactionScopeId
↓
QueryId
↓
QueryAttemptId
↓
EventId
```

---

# 22. Transaction event context

```php
final readonly class TransactionEventContext
{
    public function __construct(
        public TransactionId $transactionId,
        public TransactionAttemptId $attemptId,
        public TransactionScopeId $scopeId,
        public TransactionDepth $depth,
        public TransactionIsolation $requestedIsolation,
        public TransactionIsolation $effectiveIsolation,
        public TransactionEventPhase $phase,
        public TransactionEventFinality $finality,
        public ?ConnectionIdentity $connection,
        public PersistenceDomain $domain,
        public ?TenantContextReference $tenant,
        public ?ShardId $shard,
        public TransactionEventMetadata $metadata,
    ) {}
}
```

---

# 23. Transaction phases

```php
enum TransactionEventPhase
{
    case BEGIN;
    case ACTIVE;
    case SAVEPOINT;
    case COMMIT;
    case ROLLBACK;
    case RETRY;
    case COMPLETION;
    case DEFERRED_DELIVERY;
}
```

---

# 24. Transaction finality

```php
enum TransactionEventFinality
{
    case NONE;
    case SCOPE;
    case ATTEMPT;
    case TRANSACTION;
}
```

Esto permite distinguir:

```text
nested scope completion
attempt completion
logical transaction completion
```

---

# 25. Transaction states

El pipeline deberá reflejar los estados definidos por Transaction System.

Conceptualmente:

```text
NEW
↓
BEGINNING
↓
ACTIVE
├── COMMITTING
│      ├── COMMITTED
│      ├── FAILED
│      └── UNKNOWN
│
└── ROLLING_BACK
       ├── ROLLED_BACK
       ├── FAILED
       └── UNKNOWN
```

Podrán existir además:

```text
RETRY_PENDING
TAINTED
CLOSED
```

según contexto.

---

# 26. Event pipeline principal

```text
TransactionRequested
↓
TransactionBeginStarted
↓
BEGIN
↓
TransactionStarted
↓
ACTIVE
↓
queries / flush / savepoints
↓
TransactionCommitStarted
↓
COMMIT
├── confirmed
│      ↓
│   TransactionCommitted
│      ↓
│   AfterCommit
│
├── known failure
│      ↓
│   TransactionCommitFailed
│
└── uncertain
       ↓
    TransactionOutcomeUnknown
```

---

# 27. Begin events

Eventos principales:

```text
TransactionBeginStarted
TransactionStarted
TransactionBeginFailed
```

---

# 28. TransactionBeginStarted

Representa que el Transaction Manager está a punto de iniciar la transacción física/lógica correspondiente.

---

# 29. TransactionStarted

Solo deberá emitirse cuando el sistema tenga evidencia suficiente de que la transacción está activa.

---

# 30. Begin failure

Si:

```text
BEGIN
```

falla:

```text
TransactionBeginFailed
```

y no:

```text
TransactionStarted
```

---

# 31. Requested isolation

El evento podrá indicar:

```text
requestedIsolation
```

y:

```text
effectiveIsolation
```

por separado.

---

# 32. Requested isolation ≠ Effective isolation

Nunca asumir:

```text
requested = effective
```

sin confirmación/capability.

---

# 33. TransactionStarted payload

Podrá incluir:

```text
transaction ID
attempt ID
connection identity
requested isolation
effective isolation
read-only intent
retry policy ID
tenant
shard
start timestamp
```

con seguridad adecuada.

---

# 34. Active transaction events

Durante `ACTIVE` podrán correlacionarse:

```text
Query Events
Persistence Events
Entity Lifecycle Events
Connection Events
```

mediante `TransactionId`.

---

# 35. Transaction event pipeline ≠ event bus transaction

No deberá confundirse con sistemas que intentan hacer:

```text
EventBus.beginTransaction()
```

Este documento trata de eventos acerca de Database Transactions.

---

# 36. Nested transaction scopes

Ejemplo:

```php
$db->transaction(function () {
    // outer

    $db->transaction(function () {
        // nested
    });
});
```

El comportamiento depende de:

```text
NestedTransactionPolicy
```

---

# 37. Nested scope events

Podrán existir:

```text
TransactionNestedScopeStarted
TransactionNestedScopeCompleted
TransactionNestedScopeFailed
```

---

# 38. Nested scope depth

Ejemplo:

```text
depth 0 → root
depth 1 → nested
depth 2 → nested nested
```

---

# 39. Nested strategy

```php
enum NestedTransactionStrategy
{
    case JOIN;
    case SAVEPOINT;
    case REJECT;
    case REQUIRES_NEW;
}
```

---

# 40. JOIN strategy

```text
Outer Transaction
↓
Nested Scope
↓
same physical transaction
```

Por tanto:

```text
NestedScopeCompleted
```

solo significa que el callback/scope lógico terminó correctamente.

No significa commit.

---

# 41. SAVEPOINT strategy

```text
Outer Transaction
↓
SAVEPOINT
↓
Nested Scope
↓
RELEASE SAVEPOINT
```

sigue siendo una única transacción física.

---

# 42. REJECT strategy

Si nested transactions no están permitidas:

```text
NestedScopeRequested
↓
NestedTransactionRejected
```

sin crear savepoint.

---

# 43. REQUIRES_NEW

Si la plataforma/arquitectura lo soporta, podrá requerir:

```text
suspend outer context
↓
acquire independent transaction/connection
↓
execute inner transaction
↓
resume outer
```

---

# 44. REQUIRES_NEW ≠ savepoint

Debe conservarse esta distinción.

---

# 45. Savepoint events

Familia:

```text
SavepointCreationStarted
SavepointCreated

SavepointReleaseStarted
SavepointReleased

SavepointRollbackStarted
SavepointRolledBack

SavepointOperationFailed
```

---

# 46. SavepointCreated

Solo deberá emitirse después de creación confirmada.

---

# 47. SavepointReleased

No representa commit.

```text
SavepointReleased
≠
TransactionCommitted
```

---

# 48. SavepointRolledBack

No representa rollback completo.

```text
SavepointRolledBack
≠
TransactionRolledBack
```

---

# 49. Savepoint rollback and ORM

La base puede volver a un savepoint mientras:

```text
PHP object graph
UnitOfWork
IdentityMap
```

no retroceden automáticamente.

---

# 50. Database rollback ≠ object graph rewind

La regla existente continúa:

> **DatabaseRollback ≠ AutomaticObjectGraphRewind**

---

# 51. Commit pipeline

```text
ACTIVE
↓
TransactionCommitStarted
↓
Driver/Connection COMMIT
↓
Outcome Classification
├── COMMITTED
├── FAILED
└── UNKNOWN
```

---

# 52. TransactionCommitStarted

Representa un intento.

Podrá incluir:

```text
attempt ID
connection
transaction duration so far
query count
write count
flush count
```

de forma bounded.

---

# 53. Commit confirmation

Solo se emitirá:

```text
TransactionCommitted
```

si existe evidencia suficiente de commit exitoso.

---

# 54. Commit failure

Si existe evidencia conocida de que commit no ocurrió:

```text
TransactionCommitFailed
```

---

# 55. Commit UNKNOWN

Escenario clásico:

```text
COMMIT sent
↓
connection lost
↓
response unavailable
```

Resultado:

```text
TransactionOutcomeUnknown
```

---

# 56. UNKNOWN ≠ rollback

No deberá suponerse:

```text
connection lost
=
rollback
```

---

# 57. UNKNOWN ≠ commit

Tampoco:

```text
COMMIT sent
=
commit confirmed
```

---

# 58. Transaction outcome

```php
enum TransactionOutcome
{
    case COMMITTED;
    case ROLLED_BACK;
    case FAILED;
    case CANCELLED;
    case UNKNOWN;
}
```

---

# 59. FAILED semantics

`FAILED` deberá usarse cuando el sistema conozca que la operación transaccional no alcanzó el outcome solicitado.

---

# 60. UNKNOWN semantics

`UNKNOWN` significa:

> VoltStack carece de evidencia suficiente para afirmar si el efecto transaccional ocurrió.

---

# 61. Connection taint after UNKNOWN

Default recomendado:

```text
TransactionOutcomeUnknown
↓
ConnectionTainted
↓
ConnectionDiscarded
```

---

# 62. Why discard

No debe reutilizarse una conexión cuyo:

```text
transaction state
protocol state
session state
```

sea incierto.

---

# 63. Rollback pipeline

```text
ACTIVE / FAILED
↓
TransactionRollbackStarted
↓
ROLLBACK
├── confirmed
│      ↓
│   TransactionRolledBack
│
├── known failure
│      ↓
│   TransactionRollbackFailed
│
└── uncertain
       ↓
    TransactionOutcomeUnknown
```

---

# 64. TransactionRollbackStarted

Es intento, no outcome.

---

# 65. TransactionRolledBack

Solo se emite cuando rollback está confirmado.

---

# 66. TransactionRollbackFailed

Una falla de rollback es crítica.

Puede producir:

```text
TransactionContext TAINTED
Connection TAINTED
```

---

# 67. Rollback failure ≠ original exception

Si la transacción se estaba revirtiendo debido a:

```text
DomainException
```

y rollback falla:

```text
DomainException
+
RollbackFailure
```

ambos deben preservarse.

---

# 68. Primary exception preservation

El error original no deberá desaparecer.

---

# 69. Exception composition

Podrá existir:

```php
final class TransactionRollbackException extends TransactionException
{
    public function primaryCause(): ?Throwable;
    public function rollbackCause(): Throwable;
}
```

o equivalente.

---

# 70. Automatic rollback

Si el callback falla:

```text
Transaction Callback
↓
Exception
↓
RollbackStarted
↓
RolledBack
↓
original exception rethrown
```

---

# 71. Rollback listener failure

No deberá sustituir la excepción original de negocio/transacción.

---

# 72. Deferred event phases

El pipeline deberá soportar:

```text
IMMEDIATE
AFTER_COMMIT
AFTER_ROLLBACK
AFTER_COMPLETION
```

---

# 73. Delivery phase

```php
enum TransactionEventDeliveryPhase
{
    case IMMEDIATE;
    case AFTER_COMMIT;
    case AFTER_ROLLBACK;
    case AFTER_COMPLETION;
}
```

---

# 74. IMMEDIATE

Se utiliza para observar el hecho en el momento en que ocurre.

Ejemplo:

```text
TransactionStarted
```

---

# 75. AFTER_COMMIT

Solo se entrega si:

```text
TransactionOutcome = COMMITTED
```

confirmado.

---

# 76. AFTER_ROLLBACK

Solo se entrega si:

```text
TransactionOutcome = ROLLED_BACK
```

confirmado.

---

# 77. AFTER_COMPLETION

Puede ejecutarse para outcomes terminales definidos:

```text
COMMITTED
ROLLED_BACK
FAILED
UNKNOWN
```

según contrato.

---

# 78. AfterCommit ≠ Commit

Crítico:

```text
TransactionCommitted
↓
AfterCommit listener
```

El listener ocurre después de que commit ya fue confirmado.

---

# 79. AfterCommit failure

Si:

```text
COMMIT succeeds
↓
TransactionCommitted
↓
AfterCommit listener fails
```

la transacción sigue:

```text
COMMITTED
```

---

# 80. Listener failure cannot undo commit

Nunca:

```text
AfterCommit failure
↓
TransactionCommitFailed
```

---

# 81. AfterCommit delivery outcome

Deberá registrarse separadamente:

```text
Transaction outcome = COMMITTED
Event delivery outcome = FAILED
```

---

# 82. External side effects

Ejemplo:

```text
AfterCommit
↓
send email
```

puede fallar.

Esto no revierte la base de datos.

---

# 83. Durable side effects

Para garantías robustas deberá usarse:

```text
Transactional Outbox
```

u otro patrón durable.

---

# 84. AfterCommit callback ≠ Outbox

Un callback in-memory:

```text
afterCommit(fn() => ...)
```

no proporciona durabilidad ante:

```text
process crash
worker kill
machine failure
```

---

# 85. Crash window

Ejemplo:

```text
COMMIT confirmed
↓
process crashes
↓
before afterCommit callback
```

La DB está committed.

El callback nunca se ejecutó.

---

# 86. Durable delivery

Si el efecto es crítico:

```text
transaction
├── domain write
└── outbox record
↓
COMMIT
↓
outbox worker
↓
external side effect
```

---

# 87. Deferred event queue

Cada TransactionContext podrá mantener:

```text
DeferredTransactionEventQueue
```

scope-local.

---

# 88. Queue ≠ message broker

Será una estructura interna temporal.

---

# 89. Deferred event record

```php
final readonly class DeferredTransactionEvent
{
    public function __construct(
        public DatabaseEvent $event,
        public TransactionEventDeliveryPhase $phase,
        public EventSequence $sequence,
    ) {}
}
```

---

# 90. Event ordering

Los eventos deferred deberán conservar un orden determinista.

---

# 91. Example

Dentro de una transaction:

```text
E1
E2
E3
```

registrados `AFTER_COMMIT`.

Tras commit:

```text
E1
E2
E3
```

deberán entregarse en ese orden salvo contrato explícito diferente.

---

# 92. Nested deferred events

Este caso requiere especial cuidado.

```text
Outer Transaction
├── Event A
├── Nested Scope
│   └── Event B
└── Event C
```

Si nested scope usa JOIN/SAVEPOINT:

```text
B
```

no deberá dispararse `AFTER_COMMIT` al terminar el nested scope.

---

# 93. Root commit controls AfterCommit

Para JOIN/SAVEPOINT:

```text
Outer COMMIT
↓
A
B
C
```

pueden liberarse.

---

# 94. Nested rollback to savepoint

Si:

```text
Nested Scope
↓
Event B queued
↓
ROLLBACK TO SAVEPOINT
```

los eventos deferred pertenecientes a los efectos revertidos deberán descartarse o marcarse según contrato.

---

# 95. Deferred queue scopes

Podrá modelarse:

```text
Root Deferred Queue
├── Scope S0
├── Scope S1
└── Scope S2
```

---

# 96. Savepoint checkpoint

Al crear savepoint:

```text
DeferredEventCheckpoint
```

podrá registrar la posición actual.

---

# 97. Rollback to savepoint

Podrá eliminar:

```text
events queued after checkpoint
```

cuando sus efectos DB hayan sido revertidos.

---

# 98. Release savepoint

Los eventos del nested scope permanecen pendientes del root commit.

---

# 99. Nested scope failure with JOIN

Si no existe savepoint y el nested scope falla, la transacción raíz puede quedar:

```text
rollback-only
```

según policy.

---

# 100. Rollback-only state

Podrá existir:

```text
MARKED_ROLLBACK_ONLY
```

en TransactionContext.

---

# 101. Rollback-only event

Evento diagnóstico:

```text
TransactionMarkedRollbackOnly
```

podrá indicar:

```text
reason
scope
cause category
```

---

# 102. Commit on rollback-only

Debe rechazarse o transformarse según Transaction Manager contract.

No deberá emitirse `TransactionCommitted`.

---

# 103. Retry pipeline

```text
Attempt A1
↓
TransactionStarted
↓
Queries
↓
Deadlock
↓
Rollback
↓
TransactionRetryScheduled
↓
Backoff
↓
TransactionRetryStarted
↓
Attempt A2
↓
TransactionStarted
↓
Queries replayed
↓
Commit
↓
TransactionCommitted
```

---

# 104. Retry policy ownership

La decisión pertenece a:

```text
TransactionRetrySystem
```

no al Event Pipeline.

---

# 105. TransactionRetryScheduled

Describe una decisión ya tomada.

---

# 106. Retry payload

Podrá contener:

```text
current attempt
next attempt
reason
failure category
backoff
maximum attempts
retry policy ID
```

---

# 107. RetryStarted

Se emitirá al comenzar el nuevo attempt.

---

# 108. Logical TransactionId preserved

```text
TransactionId = same
```

mientras:

```text
TransactionAttemptId = new
```

---

# 109. Query IDs on retry

Las queries reejecutadas deberán recibir nuevos QueryIds/attempt identities según el Query Event contract.

No deberán aparentar ser la ejecución física original.

---

# 110. Retry and PHP state

Un retry de transaction puede volver a ejecutar:

```php
$db->transaction(function () {
    // callback
});
```

Por tanto el callback debe considerarse replayable bajo la policy.

---

# 111. Side effects during retry

Peligroso:

```php
$db->transaction(function () {
    chargeExternalApi();
    updateDatabase();
});
```

Si ocurre retry:

```text
external API
```

podría ejecutarse múltiples veces.

---

# 112. Retry safety

El Event Pipeline podrá exponer:

```text
retry safety classification
```

pero no garantizar idempotencia.

---

# 113. Retry safety classification

```php
enum TransactionRetrySafety
{
    case SAFE;
    case REQUIRES_IDEMPOTENCY;
    case UNSAFE;
    case UNKNOWN;
}
```

---

# 114. UNKNOWN commit and retry

Regla absoluta:

> **Un commit con outcome UNKNOWN no deberá ser reintentado ciegamente como si hubiese fallado.**

---

# 115. Why

Podría ocurrir:

```text
Attempt A1 actually COMMITTED
↓
client sees UNKNOWN
↓
retry A2
↓
duplicate effects
```

---

# 116. Retry eligibility

Solo failures con outcome suficientemente conocido podrán pasar al retry policy.

---

# 117. Deadlock

Normalmente:

```text
deadlock detected
↓
transaction aborted/rollback required
↓
retry whole transaction
```

según plataforma y policy.

---

# 118. Deadlock event correlation

Podrá correlacionarse:

```text
QueryFailed(DEADLOCK)
↓
TransactionRollbackStarted
↓
TransactionRolledBack
↓
TransactionRetryScheduled
```

---

# 119. Serialization failure

Similarmente:

```text
SERIALIZATION_FAILURE
```

puede requerir replay completo.

---

# 120. Optimistic lock conflict

No necesariamente deberá auto-retry.

Puede requerir:

```text
application conflict resolution
```

---

# 121. Pessimistic locking

Lock acquisition queries siguen siendo Query Events.

La transaction sigue siendo el scope de ownership del lock.

---

# 122. Lock release

Normalmente ocurre por:

```text
commit
rollback
connection termination
```

según DB.

No debe inferirse únicamente desde un listener.

---

# 123. Transaction cancellation

Podrá existir:

```text
TransactionCancellationRequested
```

---

# 124. Cancellation requested ≠ rollback confirmed

```text
CancellationRequested
↓
RollbackStarted
↓
RolledBack
```

es el flujo normal.

Pero solo `RolledBack` confirma rollback.

---

# 125. Transaction timeout

Podrá originarse en:

```text
application deadline
transaction timeout
database timeout
resource governor
```

---

# 126. Timeout ≠ outcome

Un timeout puede terminar en:

```text
ROLLED_BACK
FAILED
UNKNOWN
```

dependiendo del estado real.

---

# 127. TransactionOutcomeUnknown event

```php
final readonly class TransactionOutcomeUnknown implements TransactionEvent
{
    public function __construct(
        public TransactionEventContext $context,
        public TransactionUnknownOutcomeReason $reason,
    ) {}
}
```

---

# 128. Unknown reasons

```php
enum TransactionUnknownOutcomeReason
{
    case CONNECTION_LOST_DURING_COMMIT;
    case CONNECTION_LOST_DURING_ROLLBACK;
    case DRIVER_STATE_UNKNOWN;
    case TIMEOUT_DURING_FINALIZATION;
    case PROTOCOL_FAILURE;
    case OTHER;
}
```

---

# 129. UNKNOWN handling

Deberá provocar estrategias conservadoras:

```text
mark transaction closed/unknown
taint connection
discard connection
invalidate uncertain cache domains where necessary
surface error
avoid blind retry
```

---

# 130. Cache interaction

Writes dentro de una transacción no deberán publicar cache state como committed antes de commit.

---

# 131. Cache invalidation

Podrá registrarse:

```text
Deferred Cache Invalidation
```

para:

```text
AFTER_COMMIT
```

---

# 132. Rollback

Si rollback ocurre:

```text
pending afterCommit cache actions
=
discarded
```

---

# 133. UNKNOWN commit

Debe usarse política conservadora.

Ejemplo:

```text
invalidate potentially affected cache
```

puede ser más seguro que conservar datos posiblemente stale.

---

# 134. Entity cache

Misma regla:

```text
uncommitted entity state
```

no deberá publicarse como committed second-level cache state.

---

# 135. Result cache

Queries cacheadas relacionadas con writes podrán invalidarse after commit.

---

# 136. ORM integration

```text
EntityManager
↓
UnitOfWork
↓
flush
↓
Persistence Engine
↓
Queries
↓
Transaction
```

---

# 137. flush() event relationship

Podrá existir:

```text
PersistenceFlushStarted
PersistenceFlushCompleted
```

pero:

```text
PersistenceFlushCompleted
≠
TransactionCommitted
```

---

# 138. Implicit transaction

ORM puede utilizar una transacción interna para flush.

En ese caso:

```text
FlushStarted
↓
TransactionStarted
↓
Persistence Queries
↓
Commit
↓
TransactionCommitted
↓
FlushCompleted
```

según ownership exacto.

---

# 139. Existing transaction

Si `flush()` ocurre dentro de transaction externa:

```text
TransactionStarted
↓
FlushStarted
↓
Queries
↓
FlushCompleted
↓
more work
↓
Commit
```

---

# 140. Flush ownership

Persistence Engine no deberá hacer commit de una transacción caller-owned.

---

# 141. Object graph after rollback

Una vez más:

```text
Database rollback
≠
PHP object graph rewind
```

---

# 142. ORM consistency event

Tras rollback podría ser necesario:

```text
EntityManager clear
refresh
mark stale
mark uncertain
```

según Persistence Consistency System.

Pero no corresponde al Event Pipeline decidirlo.

---

# 143. Entity lifecycle deferred events

Eventos como:

```text
EntityCreated
EntityUpdated
EntityDeleted
```

pueden requerir fases distintas:

```text
immediate persistence observation
after commit domain visibility
```

---

# 144. Entity lifecycle event ≠ transaction event

La correlación ocurre mediante:

```text
TransactionId
```

---

# 145. Persistence events

El documento:

```text
214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
```

definirá esta integración en detalle.

---

# 146. Event listener phases

Podrá existir:

```php
#[Listen(
    to: TransactionCommitted::class,
    phase: TransactionEventDeliveryPhase::IMMEDIATE
)]
```

y mecanismos de deferred registration para eventos de otros subsistemas.

---

# 147. TransactionSynchronization

Conviene definir un contrato interno inspirado en synchronization callbacks:

```php
interface TransactionSynchronization
{
    public function afterCommit(TransactionCompletion $completion): void;

    public function afterRollback(TransactionCompletion $completion): void;

    public function afterCompletion(TransactionCompletion $completion): void;
}
```

---

# 148. Synchronization ≠ generic event listener

`TransactionSynchronization` puede utilizarse para correctness-sensitive integrations internas.

Generic listeners son observacionales por default.

---

# 149. Why separate

Componentes como:

```text
Cache Invalidation Coordinator
Outbox Coordinator
Persistence Consistency Coordinator
```

pueden requerir semánticas más estrictas que un listener genérico.

---

# 150. Correctness hooks

No deberán depender de:

```text
best-effort generic listeners
```

cuando la integridad dependa de ellos.

---

# 151. Event listener failure policies

```php
enum TransactionListenerFailurePolicy
{
    case PROPAGATE_BEFORE_OUTCOME;
    case RECORD_AND_CONTINUE;
    case FAIL_DEFERRED_DELIVERY;
    case CUSTOM;
}
```

---

# 152. Before commit listener danger

Permitir listeners arbitrarios que ejecuten lógica antes de commit puede cambiar:

```text
latency
deadlock probability
failure surface
transaction duration
```

---

# 153. Pre-commit mutation

No deberá usarse el Event System genérico para modificar la transaction justo antes del commit.

Para eso deberán existir contratos explícitos.

---

# 154. BeforeCommit synchronization

Si se requiere:

```text
beforeCommit
```

deberá ser una API especializada con semántica fuerte.

---

# 155. beforeCommit failure

Puede impedir que se intente commit si ocurre antes del physical commit boundary.

---

# 156. afterCommit failure

No puede deshacer el commit.

---

# 157. afterRollback failure

No puede convertir un rollback confirmado en rollback fallido.

---

# 158. afterCompletion failure

No cambia el outcome transaccional.

---

# 159. Canonical precedence

Siempre:

```text
Database Transaction Outcome
>
Event Delivery Outcome
```

---

# 160. Event delivery result

```php
final readonly class TransactionEventDeliveryResult
{
    public function __construct(
        public TransactionOutcome $transactionOutcome,
        public EventDeliveryOutcome $deliveryOutcome,
        public array $listenerFailures,
    ) {}
}
```

---

# 161. EventDeliveryOutcome

```php
enum EventDeliveryOutcome
{
    case NOT_REQUIRED;
    case COMPLETED;
    case PARTIAL;
    case FAILED;
}
```

---

# 162. COMMITTED + FAILED delivery

Es válido:

```text
transactionOutcome = COMMITTED
deliveryOutcome = FAILED
```

---

# 163. ROLLED_BACK + FAILED delivery

También válido.

---

# 164. UNKNOWN + deferred events

`AFTER_COMMIT` no deberá ejecutarse.

`AFTER_ROLLBACK` tampoco.

Porque ninguno está confirmado.

---

# 165. AFTER_COMPLETION with UNKNOWN

Podrá ejecutarse si el listener declara soportar:

```text
UNKNOWN
```

---

# 166. Completion context

```php
final readonly class TransactionCompletion
{
    public function __construct(
        public TransactionId $transactionId,
        public TransactionAttemptId $attemptId,
        public TransactionOutcome $outcome,
        public TransactionCompletionEvidence $evidence,
    ) {}
}
```

---

# 167. Evidence

```php
enum TransactionCompletionEvidence
{
    case DRIVER_CONFIRMED;
    case CONNECTION_CONFIRMED;
    case DATABASE_CONFIRMED;
    case UNKNOWN;
}
```

---

# 168. Evidence ≠ absolute physical truth

Representa el nivel de evidencia disponible para VoltStack.

---

# 169. Connection ownership

Transaction Manager podrá adquirir/pin una conexión.

---

# 170. Connection events

Ejemplo:

```text
ConnectionAcquired
↓
TransactionBeginStarted
↓
TransactionStarted
↓
...
TransactionCommitted
↓
ConnectionReset
↓
ConnectionReleased
```

---

# 171. Connection released before transaction completion

No deberá ocurrir para una transaction que requiere connection affinity.

---

# 172. Connection loss

Puede producir:

```text
TransactionOutcomeUnknown
```

dependiendo del stage.

---

# 173. Loss before commit

Si connection falla mientras transaction está activa pero antes de commit:

```text
transaction failed
```

podría conocerse suficientemente según driver.

---

# 174. Loss during commit

Es el caso más peligroso:

```text
UNKNOWN
```

---

# 175. Loss after confirmed commit

Si confirmation ya fue recibida:

```text
COMMITTED
```

aunque la conexión falle inmediatamente después.

---

# 176. Event timestamps

Eventos podrán contener:

```text
wall-clock timestamp
monotonic offset
```

---

# 177. Transaction duration

Deberá distinguir:

```text
active transaction duration
commit duration
rollback duration
retry backoff
afterCommit delivery duration
```

---

# 178. Listener duration ≠ transaction duration

Si commit ya terminó:

```text
afterCommit listener time
```

no deberá sumarse a:

```text
physical transaction duration
```

---

# 179. Long transaction detection

Telemetry podrá consumir:

```text
TransactionStarted
TransactionCommitted/RolledBack
```

para detectar transacciones largas.

---

# 180. Transaction telemetry

La especialización será:

```text
219_DATABASE_TRANSACTION_TELEMETRY_SYSTEM.md
```

---

# 181. Telemetry ≠ Event Pipeline

El pipeline genera hechos.

Telemetry los transforma en:

```text
metrics
traces
logs
profiles
```

---

# 182. Security

Transaction Events pueden revelar:

```text
tenant
shard
connection
transaction duration
query counts
rollback causes
```

---

# 183. Security policy

No deberán incluir:

```text
credentials
raw DSN
full SQL
raw query parameters
managed entities
full object graph
```

---

# 184. TransactionEventExposureLevel

```php
enum TransactionEventExposureLevel
{
    case MINIMAL;
    case SAFE_DIAGNOSTIC;
    case DEVELOPMENT;
}
```

---

# 185. MINIMAL

Ejemplo:

```text
transaction fingerprint
attempt
outcome
duration
```

---

# 186. SAFE_DIAGNOSTIC

Puede añadir:

```text
isolation
connection role
shard
retry reason category
query count
```

---

# 187. DEVELOPMENT

Puede incluir más diagnostics, pero nunca secrets por default.

---

# 188. Transaction fingerprint

Podrá existir:

```text
TransactionFingerprint
```

para telemetry agregada.

No deberá basarse en IDs únicos.

---

# 189. High cardinality

No usar automáticamente como metric labels:

```text
TransactionId
RequestId
TenantId
TraceId
```

---

# 190. Event externalization

Transaction events serán:

```text
IN_PROCESS
```

por default.

---

# 191. External transaction events

Algunos eventos pueden externalizarse:

```text
TransactionCommitted
TransactionRolledBack
TransactionOutcomeUnknown
```

solo bajo contrato explícito.

---

# 192. Externalization ≠ durable domain event

Publicar `TransactionCommitted` externamente no sustituye:

```text
Outbox
```

---

# 193. Crash safety

Si la aplicación necesita garantizar:

```text
DB commit
+
message eventually published
```

deberá utilizar una estrategia durable.

---

# 194. Testing

Deberá ser posible comprobar:

```php
DatabaseEvents::assertSequence([
    TransactionBeginStarted::class,
    TransactionStarted::class,
    TransactionCommitStarted::class,
    TransactionCommitted::class,
]);
```

---

# 195. Rollback testing

```php
DatabaseEvents::assertSequence([
    TransactionStarted::class,
    TransactionRollbackStarted::class,
    TransactionRolledBack::class,
]);
```

---

# 196. UNKNOWN testing

```text
TransactionStarted
↓
CommitStarted
↓
connection failure
↓
TransactionOutcomeUnknown
```

---

# 197. Savepoint testing

```text
TransactionStarted
↓
SavepointCreated
↓
SavepointRolledBack
↓
TransactionCommitted
```

---

# 198. Nested afterCommit testing

```text
Outer begin
↓
Nested scope
↓
queue Event B
↓
Nested scope complete
↓
assert B NOT delivered
↓
Outer commit
↓
assert B delivered
```

---

# 199. Savepoint rollback deferred-event test

```text
savepoint
↓
queue B
↓
rollback to savepoint
↓
commit outer
↓
B MUST NOT be delivered
```

si B correspondía exclusivamente a efectos revertidos.

---

# 200. Retry testing

```text
Attempt A1
↓
deadlock
↓
rollback
↓
retry scheduled
↓
Attempt A2
↓
commit
```

---

# 201. AfterCommit failure testing

```text
TransactionCommitted
↓
listener throws
```

assert:

```text
TransactionOutcome = COMMITTED
EventDeliveryOutcome = FAILED
```

---

# 202. Rollback failure testing

Debe preservar:

```text
original exception
+
rollback failure
```

---

# 203. Persistent runtime safety

Aplicable a:

```text
FrankenPHP
RoadRunner
OpenSwoole
```

---

# 204. Scope-local state

Deberá permanecer scope-local:

```text
TransactionId
TransactionAttemptId
TransactionScopeId
savepoint stack
deferred event queue
retry attempt
rollback-only state
transaction synchronization registry
connection affinity
tenant context
```

---

# 205. No static transaction state

Prohibido:

```php
TransactionEvents::$currentTransaction;
```

---

# 206. No static deferred queue

Prohibido:

```php
TransactionEvents::$afterCommitCallbacks;
```

---

# 207. FrankenPHP

Al terminar request:

```text
no active transaction
no deferred callbacks
no savepoint stack
no transaction-scoped event state
```

deberá sobrevivir accidentalmente.

---

# 208. RoadRunner

Cada job/request deberá obtener un transaction event scope limpio.

---

# 209. OpenSwoole

Transaction context deberá ser:

```text
coroutine-local
```

o explícitamente propagado.

---

# 210. Worker termination

Si un worker termina con una transaction activa:

```text
best-effort rollback
connection taint/discard
diagnostic event
```

según capability.

---

# 211. Worker termination ≠ confirmed rollback

Si no puede confirmarse:

```text
UNKNOWN
```

deberá preservarse.

---

# 212. Event pipeline performance

Las transacciones son hot path.

Debe existir:

```text
fast path
```

cuando no haya listeners/synchronizations.

---

# 213. Deferred queue allocation

No será necesario crear una queue si no se registra ningún deferred event.

---

# 214. Lazy event payload

Información costosa como:

```text
query summaries
stack traces
entity summaries
```

solo deberá generarse cuando se solicite.

---

# 215. Event sequence

Dentro de una transaction podrá utilizarse:

```php
final readonly class TransactionEventSequence
{
    public function __construct(
        public int $value,
    ) {}
}
```

---

# 216. Deterministic ordering

Cada evento registrado podrá recibir:

```text
sequence++
```

dentro del scope.

---

# 217. Sequence ≠ global order

La secuencia es local a la transaction/attempt.

No representa orden global distribuido.

---

# 218. Distributed transactions

VoltStack no deberá fingir:

```text
single ACID transaction
```

sobre múltiples shards si no existe un protocolo explícito.

---

# 219. Cross-shard operation

Podrá tener:

```text
Transaction T-Shard-A
Transaction T-Shard-B
```

pero no deberán presentarse automáticamente como una sola transaction física.

---

# 220. Distributed transaction events

Si en el futuro se implementa coordinación distribuida deberá tener:

```text
DistributedTransactionId
ParticipantTransactionId
```

separados.

---

# 221. No implicit 2PC

Este Event Pipeline no introduce:

```text
two-phase commit
```

---

# 222. Tenant context

Transaction deberá conservar el mismo tenant context cuando la política lo requiera.

---

# 223. Tenant drift

No deberá ocurrir:

```text
BEGIN tenant A
↓
switch tenant B
↓
COMMIT
```

sin una arquitectura explícita que lo permita.

---

# 224. Tenant context event

Podrá incluir:

```text
tenant reference/fingerprint
```

de manera segura.

---

# 225. Shard affinity

Similarmente:

```text
transaction shard
```

deberá permanecer estable para single-shard transactions.

---

# 226. Retry and topology change

Entre attempts podría cambiar:

```text
writer endpoint
authority epoch
```

si la policy lo permite.

---

# 227. Retry transaction identity

La transacción lógica permanece igual, pero el attempt y connection pueden cambiar.

---

# 228. Event diagnostics

El sistema deberá poder generar:

```text
Transaction Event Explain
```

---

# 229. Ejemplo explain

```text
Transaction: tx_01J...
Attempt: 2
Scope: root
Depth: 0

Isolation:
  Requested: READ_COMMITTED
  Effective: READ_COMMITTED

Connection:
  Role: WRITER
  Shard: shard_01

Lifecycle:
  BEGIN_STARTED
  STARTED
  QUERY x 4
  COMMIT_STARTED
  COMMITTED

Retry:
  Previous Attempt: DEADLOCK
  Backoff: 25ms

Duration:
  Active: 41ms
  Commit: 3ms
  Total Attempt: 44ms

Deferred:
  AfterCommit queued: 3
  Delivered: 3
  Failed: 0

Outcome:
  Transaction: COMMITTED
  Event Delivery: COMPLETED
```

---

# 230. Error hierarchy

```text
TransactionEventPipelineException
├── TransactionEventLifecycleException
├── TransactionEventCorrelationException
├── TransactionEventDeliveryException
├── TransactionDeferredEventException
├── TransactionSynchronizationException
├── TransactionUnknownOutcomeException
├── TransactionRetryEventException
├── TransactionSavepointEventException
├── TransactionEventSecurityException
└── TransactionEventCompatibilityException
```

---

# 231. TransactionEventLifecycleException

Ejemplos:

```text
Committed before CommitStarted
RolledBack after Committed
SavepointReleased after root completion
AfterCommit delivered before commit
```

---

# 232. Directory structure

```text
src/Quantum/Database/Event/Transaction/
│
├── Contract/
│   ├── TransactionEvent.php
│   ├── TransactionAttemptEvent.php
│   ├── TransactionScopeEvent.php
│   ├── SavepointEvent.php
│   └── TransactionSynchronization.php
│
├── Event/
│   ├── TransactionBeginStarted.php
│   ├── TransactionStarted.php
│   ├── TransactionBeginFailed.php
│   ├── TransactionNestedScopeStarted.php
│   ├── TransactionNestedScopeCompleted.php
│   ├── TransactionNestedScopeFailed.php
│   ├── TransactionMarkedRollbackOnly.php
│   ├── SavepointCreationStarted.php
│   ├── SavepointCreated.php
│   ├── SavepointReleaseStarted.php
│   ├── SavepointReleased.php
│   ├── SavepointRollbackStarted.php
│   ├── SavepointRolledBack.php
│   ├── SavepointOperationFailed.php
│   ├── TransactionCommitStarted.php
│   ├── TransactionCommitted.php
│   ├── TransactionCommitFailed.php
│   ├── TransactionRollbackStarted.php
│   ├── TransactionRolledBack.php
│   ├── TransactionRollbackFailed.php
│   ├── TransactionCancellationRequested.php
│   ├── TransactionRetryScheduled.php
│   ├── TransactionRetryStarted.php
│   └── TransactionOutcomeUnknown.php
│
├── Context/
│   ├── TransactionEventContext.php
│   ├── TransactionEventMetadata.php
│   ├── TransactionEventCorrelationContext.php
│   └── TransactionEventScope.php
│
├── Model/
│   ├── TransactionAttemptId.php
│   ├── TransactionScopeId.php
│   ├── SavepointId.php
│   ├── TransactionEventPhase.php
│   ├── TransactionEventFinality.php
│   ├── TransactionOutcome.php
│   ├── TransactionRetrySafety.php
│   ├── TransactionUnknownOutcomeReason.php
│   ├── TransactionCompletion.php
│   └── TransactionCompletionEvidence.php
│
├── Deferred/
│   ├── DeferredTransactionEvent.php
│   ├── DeferredTransactionEventQueue.php
│   ├── DeferredEventCheckpoint.php
│   ├── TransactionEventDeliveryPhase.php
│   ├── TransactionEventDeliveryResult.php
│   └── EventDeliveryOutcome.php
│
├── Synchronization/
│   ├── TransactionSynchronizationRegistry.php
│   ├── TransactionSynchronizationCoordinator.php
│   └── TransactionSynchronizationResult.php
│
├── Security/
│   ├── TransactionEventSanitizer.php
│   ├── TransactionEventExposureLevel.php
│   └── TransactionEventExposurePolicy.php
│
├── Diagnostics/
│   ├── TransactionEventInspector.php
│   ├── TransactionEventLifecycleValidator.php
│   └── TransactionEventExplain.php
│
├── Testing/
│   ├── TransactionEventRecorder.php
│   ├── TransactionEventAssertions.php
│   └── FakeTransactionSynchronization.php
│
└── Exception/
    ├── TransactionEventPipelineException.php
    ├── TransactionEventLifecycleException.php
    ├── TransactionEventCorrelationException.php
    ├── TransactionEventDeliveryException.php
    ├── TransactionDeferredEventException.php
    ├── TransactionSynchronizationException.php
    ├── TransactionUnknownOutcomeException.php
    ├── TransactionRetryEventException.php
    ├── TransactionSavepointEventException.php
    ├── TransactionEventSecurityException.php
    └── TransactionEventCompatibilityException.php
```

---

# 233. Integración general

```text
                    Transaction Manager
                            │
                            ▼
                   Transaction Context
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
           Begin         Active         Retry
              │             │             │
              ▼             ▼             ▼
        Begin Events    Query Events   Retry Events
                            │
                     Nested / Savepoint
                            │
                            ▼
                    Savepoint Events
                            │
                    ┌───────┴────────┐
                    ▼                ▼
                  Commit           Rollback
                    │                │
                    ▼                ▼
              Commit Events    Rollback Events
                    │                │
                    └───────┬────────┘
                            ▼
                         Outcome
                ┌───────────┼───────────┐
                ▼           ▼           ▼
            COMMITTED   ROLLED_BACK   UNKNOWN
                │           │           │
                ▼           ▼           ▼
          AfterCommit AfterRollback AfterCompletion
```

---

# 234. Architectural invariants

## DB-TEVENT-001
Transaction Event será distinto de Transaction.

## DB-TEVENT-002
Transaction Event será distinto de TransactionContext.

## DB-TEVENT-003
TransactionId será distinto de TransactionAttemptId.

## DB-TEVENT-004
TransactionAttemptId será distinto de TransactionScopeId.

## DB-TEVENT-005
TransactionScopeId será distinto de SavepointId.

## DB-TEVENT-006
Transaction será distinta de Connection.

## DB-TEVENT-007
Transaction será distinta de Query.

## DB-TEVENT-008
Transaction será distinta de UnitOfWork.

## DB-TEVENT-009
Transaction será distinta de Flush.

## DB-TEVENT-010
Transaction será distinta de Savepoint.

## DB-TEVENT-011
Transaction será distinta de Nested Scope.

## DB-TEVENT-012
Flush será distinto de Commit.

## DB-TEVENT-013
QueryExecuted será distinto de TransactionCommitted.

## DB-TEVENT-014
Commit requested será distinto de Commit confirmed.

## DB-TEVENT-015
CommitStarted será distinto de Committed.

## DB-TEVENT-016
Rollback requested será distinto de Rollback confirmed.

## DB-TEVENT-017
RollbackStarted será distinto de RolledBack.

## DB-TEVENT-018
Savepoint será distinto de Transaction.

## DB-TEVENT-019
Savepoint release será distinto de Commit.

## DB-TEVENT-020
Savepoint rollback será distinto de root rollback.

## DB-TEVENT-021
Nested scope completion será distinto de Commit.

## DB-TEVENT-022
JOIN nested strategy no creará transaction física nueva.

## DB-TEVENT-023
SAVEPOINT strategy no creará transaction física independiente.

## DB-TEVENT-024
REQUIRES_NEW será distinto de SAVEPOINT.

## DB-TEVENT-025
Savepoint identity será distinta del nombre SQL.

## DB-TEVENT-026
TransactionStarted requerirá evidencia de active transaction.

## DB-TEVENT-027
Begin failure no emitirá TransactionStarted.

## DB-TEVENT-028
Requested isolation será distinta de effective isolation.

## DB-TEVENT-029
Transaction state authority pertenecerá al Transaction System.

## DB-TEVENT-030
Events no redefinirán canonical transaction state.

## DB-TEVENT-031
Commit confirmation requerirá evidencia suficiente.

## DB-TEVENT-032
Known commit failure será distinto de UNKNOWN.

## DB-TEVENT-033
UNKNOWN será first-class.

## DB-TEVENT-034
UNKNOWN no será convertido en rollback.

## DB-TEVENT-035
UNKNOWN no será convertido en commit.

## DB-TEVENT-036
Connection loss during commit podrá producir UNKNOWN.

## DB-TEVENT-037
UNKNOWN podrá taint connection.

## DB-TEVENT-038
UNKNOWN connection state no será reusable por default.

## DB-TEVENT-039
Rollback confirmation requerirá evidencia suficiente.

## DB-TEVENT-040
Rollback failure podrá taint transaction context.

## DB-TEVENT-041
Rollback failure podrá taint connection.

## DB-TEVENT-042
Original exception se preservará ante rollback failure.

## DB-TEVENT-043
Database rollback será distinto de object graph rewind.

## DB-TEVENT-044
Immediate event será distinto de deferred event.

## DB-TEVENT-045
AFTER_COMMIT solo se ejecutará tras COMMITTED confirmado.

## DB-TEVENT-046
AFTER_ROLLBACK solo se ejecutará tras ROLLED_BACK confirmado.

## DB-TEVENT-047
AFTER_COMPLETION tendrá outcome explícito.

## DB-TEVENT-048
AfterCommit será distinto de Commit.

## DB-TEVENT-049
AfterCommit failure no cambiará COMMITTED.

## DB-TEVENT-050
AfterRollback failure no cambiará ROLLED_BACK.

## DB-TEVENT-051
AfterCompletion failure no cambiará transaction outcome.

## DB-TEVENT-052
Event delivery outcome será distinto de transaction outcome.

## DB-TEVENT-053
COMMITTED + failed event delivery será representable.

## DB-TEVENT-054
ROLLED_BACK + failed event delivery será representable.

## DB-TEVENT-055
Generic afterCommit callback no será durable outbox.

## DB-TEVENT-056
Process crash podrá ocurrir después de commit y antes de callback.

## DB-TEVENT-057
Critical external side effects usarán durable mechanism cuando sea necesario.

## DB-TEVENT-058
Deferred queue será scope-local.

## DB-TEVENT-059
Deferred queue no será global static state.

## DB-TEVENT-060
Deferred event order será deterministic.

## DB-TEVENT-061
Nested JOIN afterCommit esperará root commit.

## DB-TEVENT-062
Nested SAVEPOINT afterCommit esperará root commit.

## DB-TEVENT-063
Savepoint release no disparará root afterCommit.

## DB-TEVENT-064
Rollback to savepoint podrá descartar deferred events revertidos.

## DB-TEVENT-065
Deferred savepoint checkpoint será distinto de DB savepoint.

## DB-TEVENT-066
Rollback-only state será explícito.

## DB-TEVENT-067
Rollback-only transaction no será reported committed.

## DB-TEVENT-068
Transaction retry será distinto de Query retry.

## DB-TEVENT-069
Retry decision pertenecerá a TransactionRetrySystem.

## DB-TEVENT-070
Retry event describirá una decisión ya tomada.

## DB-TEVENT-071
Logical TransactionId permanecerá estable entre retries.

## DB-TEVENT-072
TransactionAttemptId cambiará en cada attempt.

## DB-TEVENT-073
Replayed queries serán nuevas ejecuciones físicas.

## DB-TEVENT-074
Transaction callback podrá ejecutarse múltiples veces durante retry.

## DB-TEVENT-075
External side effects dentro de retry requerirán idempotency.

## DB-TEVENT-076
Retry safety podrá ser UNKNOWN.

## DB-TEVENT-077
UNKNOWN commit no será blindly retried.

## DB-TEVENT-078
Deadlock podrá requerir whole-transaction retry.

## DB-TEVENT-079
Serialization failure podrá requerir whole-transaction retry.

## DB-TEVENT-080
Optimistic conflict no implicará auto-retry.

## DB-TEVENT-081
Cancellation requested será distinta de rollback confirmed.

## DB-TEVENT-082
Timeout será distinto de transaction outcome.

## DB-TEVENT-083
UNKNOWN reason será explícita cuando sea conocida.

## DB-TEVENT-084
Cache no publicará uncommitted state como committed.

## DB-TEVENT-085
AfterCommit cache invalidation esperará commit confirmado.

## DB-TEVENT-086
Rollback descartará pending afterCommit actions.

## DB-TEVENT-087
UNKNOWN commit utilizará conservative cache policy.

## DB-TEVENT-088
Entity cache no publicará uncommitted state.

## DB-TEVENT-089
Result cache invalidation preservará transaction semantics.

## DB-TEVENT-090
PersistenceFlushCompleted será distinto de TransactionCommitted.

## DB-TEVENT-091
Persistence Engine no cometerá caller-owned transaction.

## DB-TEVENT-092
ORM state reconciliation será distinta del Event Pipeline.

## DB-TEVENT-093
Entity Lifecycle Events serán distintos de Transaction Events.

## DB-TEVENT-094
Persistence Events serán distintos de Transaction Events.

## DB-TEVENT-095
TransactionSynchronization será distinto de generic listener.

## DB-TEVENT-096
Correctness-critical synchronization no dependerá de best-effort listeners.

## DB-TEVENT-097
Pre-commit mutation usará contrato explícito.

## DB-TEVENT-098
Generic events no serán hidden transaction mutation mechanism.

## DB-TEVENT-099
Database transaction outcome tendrá precedencia sobre listener outcome.

## DB-TEVENT-100
Listener failure no reescribirá transaction history.

## DB-TEVENT-101
Transaction event payload no expondrá credentials.

## DB-TEVENT-102
Transaction event payload no expondrá raw DSN.

## DB-TEVENT-103
Transaction event payload no incluirá full SQL por default.

## DB-TEVENT-104
Transaction event payload no incluirá query parameters por default.

## DB-TEVENT-105
Managed entities no serán event payload por default.

## DB-TEVENT-106
High-cardinality IDs no serán metric labels automáticos.

## DB-TEVENT-107
Externalization será opt-in.

## DB-TEVENT-108
External transaction event será distinto de durable domain event.

## DB-TEVENT-109
Transaction lifecycle será testeable.

## DB-TEVENT-110
Commit UNKNOWN será testeable.

## DB-TEVENT-111
Rollback failure será testeable.

## DB-TEVENT-112
Nested afterCommit behavior será testeable.

## DB-TEVENT-113
Savepoint deferred-event rollback será testeable.

## DB-TEVENT-114
Retry lifecycle será testeable.

## DB-TEVENT-115
AfterCommit failure será testeable.

## DB-TEVENT-116
Transaction event state será request/operation scoped.

## DB-TEVENT-117
TransactionId no será static global state.

## DB-TEVENT-118
Deferred callbacks no serán static global state.

## DB-TEVENT-119
Savepoint stack será scope-local.

## DB-TEVENT-120
Retry state será scope-local.

## DB-TEVENT-121
FrankenPHP no heredará transaction state entre requests.

## DB-TEVENT-122
RoadRunner no heredará transaction state entre jobs.

## DB-TEVENT-123
OpenSwoole transaction state será coroutine-safe.

## DB-TEVENT-124
Worker shutdown no fabricará rollback confirmation.

## DB-TEVENT-125
Fast path existirá cuando no haya observers.

## DB-TEVENT-126
Deferred queue podrá crearse lazy.

## DB-TEVENT-127
Expensive event payload será lazy cuando sea posible.

## DB-TEVENT-128
Event sequence será local al transaction attempt.

## DB-TEVENT-129
Local event sequence no será global distributed ordering.

## DB-TEVENT-130
Cross-shard operation no fingirá single ACID transaction.

## DB-TEVENT-131
Event Pipeline no introducirá implicit 2PC.

## DB-TEVENT-132
Distributed transaction requerirá arquitectura explícita.

## DB-TEVENT-133
Tenant context será estable dentro de transaction cuando corresponda.

## DB-TEVENT-134
Shard affinity será estable en single-shard transaction.

## DB-TEVENT-135
Retry podrá cambiar physical connection sin cambiar logical TransactionId.

## DB-TEVENT-136
Retry podrá cambiar writer endpoint bajo policy explícita.

## DB-TEVENT-137
Transaction duration será distinta de afterCommit duration.

## DB-TEVENT-138
Listener duration será distinta de physical transaction duration.

## DB-TEVENT-139
Telemetry podrá consumir Transaction Events.

## DB-TEVENT-140
Telemetry no redefinirá transaction outcome.

## DB-TEVENT-141
Event System no iniciará transactions.

## DB-TEVENT-142
Event System no hará commit.

## DB-TEVENT-143
Event System no hará rollback.

## DB-TEVENT-144
Event System no creará savepoints.

## DB-TEVENT-145
Event System no decidirá isolation.

## DB-TEVENT-146
Event System no decidirá retries.

## DB-TEVENT-147
Event System no administrará connection affinity.

## DB-TEVENT-148
Event System no administrará UnitOfWork.

## DB-TEVENT-149
Event System no realizará flush.

## DB-TEVENT-150
Event System no será cache consistency engine.

## DB-TEVENT-151
Event System no será outbox.

## DB-TEVENT-152
Transaction state nunca se inferirá únicamente desde events.

## DB-TEVENT-153
Statement success nunca equivaldrá a transaction success.

## DB-TEVENT-154
All statement success nunca garantizará commit.

## DB-TEVENT-155
Savepoint success nunca garantizará root commit.

## DB-TEVENT-156
Nested scope success nunca garantizará root commit.

## DB-TEVENT-157
CommitStarted nunca garantizará commit.

## DB-TEVENT-158
RollbackStarted nunca garantizará rollback.

## DB-TEVENT-159
UNKNOWN nunca se ocultará para simplificar API.

## DB-TEVENT-160
Event Pipeline preservará evidence-aware outcomes.

## DB-TEVENT-161
Commit confirmed tendrá precedencia sobre later listener failure.

## DB-TEVENT-162
Rollback confirmed tendrá precedencia sobre later listener failure.

## DB-TEVENT-163
Transaction retry conservará attempt history.

## DB-TEVENT-164
Retry exhaustion será observable.

## DB-TEVENT-165
Deferred events revertidos por savepoint no deberán escapar al root commit.

## DB-TEVENT-166
Transaction completion cerrará deferred queue.

## DB-TEVENT-167
No podrán registrarse nuevos afterCommit events sobre transaction ya cerrada salvo contrato explícito de immediate fallback.

## DB-TEVENT-168
Default será rechazar late deferred registration.

## DB-TEVENT-169
Connection taint después de UNKNOWN será observable.

## DB-TEVENT-170
Transaction Event Pipeline preservará los boundaries del Transaction Manager.

---

# 235. Flujo exitoso

```text
TransactionBeginStarted
↓
BEGIN
↓
TransactionStarted
↓
Query Q1
↓
QueryExecuted
↓
Query Q2
↓
QueryExecuted
↓
TransactionCommitStarted
↓
COMMIT
↓
TransactionCommitted
↓
AfterCommit Queue
↓
AfterCompletion
```

---

# 236. Flujo rollback por excepción

```text
TransactionStarted
↓
Q1 success
↓
Application Exception
↓
TransactionRollbackStarted
↓
ROLLBACK
↓
TransactionRolledBack
↓
AfterRollback
↓
AfterCompletion
↓
Original Exception Propagated
```

---

# 237. Flujo commit UNKNOWN

```text
TransactionStarted
↓
Writes
↓
TransactionCommitStarted
↓
COMMIT sent
↓
Connection Lost
↓
TransactionOutcomeUnknown
↓
ConnectionTainted
↓
ConnectionDiscarded
↓
AfterCompletion(UNKNOWN)
```

No:

```text
AfterCommit
```

ni:

```text
AfterRollback
```

---

# 238. Flujo nested SAVEPOINT

```text
Root TransactionStarted
↓
Event A queued
↓
NestedScopeStarted
↓
SavepointCreated
↓
Event B queued
↓
SavepointReleased
↓
NestedScopeCompleted
↓
Event C queued
↓
Root CommitStarted
↓
TransactionCommitted
↓
deliver A
deliver B
deliver C
```

---

# 239. Nested rollback

```text
Root TransactionStarted
↓
Event A queued
↓
SavepointCreated
↓
Event B queued
↓
Nested failure
↓
SavepointRollbackStarted
↓
SavepointRolledBack
↓
discard B
↓
continue root
↓
Event C queued
↓
root commit
↓
deliver A
deliver C
```

---

# 240. Retry flow

```text
Transaction T1

Attempt A1
├── BEGIN
├── Q1
├── Q2 → DEADLOCK
├── ROLLBACK
└── RetryScheduled

backoff

Attempt A2
├── BEGIN
├── Q1'
├── Q2'
├── COMMIT
└── COMMITTED

Logical Result:
Transaction T1 = COMMITTED
Attempts = 2
```

---

# 241. Modelo formal de commit

Sea una transacción:

```text
T
```

y un intento:

```text
A_i
```

El evento:

```text
CommitStarted(A_i)
```

solo indica:

```text
CommitAttempted(A_i) = true
```

No implica:

```text
Committed(A_i) = true
```

Formalmente:

```text
CommitStarted(A_i) ↛ Committed(A_i)
```

---

# 242. Modelo formal de query

Para cualquier query `Q` dentro de `T`:

```text
Executed(Q) ↛ Committed(T)
```

Incluso:

```text
∀Q ∈ T, Executed(Q)
```

no implica:

```text
Committed(T)
```

---

# 243. Modelo formal de nested transaction

Para nested scope `S` dentro de root transaction `T`:

```text
Completed(S) ↛ Committed(T)
```

Y para savepoint `P`:

```text
Released(P) ↛ Committed(T)
```

---

# 244. Modelo formal de deferred events

Sea:

```text
D_commit(T)
```

el conjunto ordenado de eventos `AFTER_COMMIT`.

Entonces:

```text
Outcome(T) = COMMITTED
⇒
Deliver(D_commit(T))
```

Pero:

```text
Outcome(T) ∈ {ROLLED_BACK, FAILED, UNKNOWN}
⇒
DoNotDeliver(D_commit(T))
```

---

# 245. Modelo formal de rollback events

```text
Outcome(T) = ROLLED_BACK
⇒
Deliver(D_rollback(T))
```

Mientras:

```text
Outcome(T) ≠ ROLLED_BACK
⇒
DoNotDeliver(D_rollback(T))
```

---

# 246. Savepoint deferred-event model

Sea:

```text
checkpoint(P) = n
```

la posición de la deferred queue al crear savepoint `P`.

Si:

```text
RollbackTo(P)
```

entonces, bajo semantics de efectos revertidos:

```text
Discard events where sequence > n
```

para el scope correspondiente.

---

# 247. Retry model

Para transaction lógica:

```text
T
```

con attempts:

```text
A(T) = {A1, A2, ..., An}
```

solo un attempt deberá producir el outcome lógico final exitoso.

Ejemplo:

```text
A1 = ROLLED_BACK
A2 = ROLLED_BACK
A3 = COMMITTED
```

Entonces:

```text
Outcome(T) = COMMITTED
```

---

# 248. UNKNOWN retry rule

Si:

```text
Outcome(Ai) = UNKNOWN
```

entonces:

```text
BlindRetry(Ai+1) = FORBIDDEN
```

por default.

---

# 249. Regla maestra final

VoltStack deberá preservar:

```text
Transaction Event
≠
Transaction
```

```text
Transaction
≠
UnitOfWork
```

```text
Transaction
≠
Flush
```

```text
Transaction
≠
Savepoint
```

```text
Flush
≠
Commit
```

```text
QueryExecuted
≠
TransactionCommitted
```

```text
Commit Requested
≠
Commit Confirmed
```

```text
CommitStarted
≠
Committed
```

```text
RollbackStarted
≠
RolledBack
```

```text
SavepointReleased
≠
TransactionCommitted
```

```text
SavepointRolledBack
≠
TransactionRolledBack
```

```text
NestedScopeCompleted
≠
TransactionCommitted
```

```text
DatabaseRollback
≠
ObjectGraphRewind
```

```text
Transaction Retry
≠
Query Retry
```

```text
AfterCommit
≠
Commit
```

```text
AfterCommit Callback
≠
Transactional Outbox
```

```text
Event Delivery Failure
≠
Transaction Failure
```

```text
Connection Lost During Commit
≠
Rollback
```

```text
UNKNOWN
≠
FAILED
```

```text
UNKNOWN
≠
ROLLED_BACK
```

```text
UNKNOWN
≠
COMMITTED
```

y especialmente:

```text
Database Transaction Reality
≠
Event Delivery Reality
```

---

# 250. Resultado arquitectónico

Con `Database Transaction Event Pipeline`, VoltStack obtiene un modelo transaccional capaz de representar correctamente:

```text
begin
nested scopes
savepoints
queries
flushes
commit attempts
rollback attempts
retries
deadlocks
cancellation
UNKNOWN outcomes
afterCommit
afterRollback
afterCompletion
```

sin introducir la peligrosa simplificación:

```text
no exception
=
committed
```

La arquitectura final queda:

```text
Transaction Manager
=
authority over transaction lifecycle

Transaction Context
=
authority over scoped transaction state

Connection
=
physical execution resource

Query Engine
=
statement/query execution

Persistence Engine
=
ORM synchronization

Transaction Event Pipeline
=
typed observation + controlled deferred delivery

Telemetry
=
measurement

Outbox
=
durable external-event delivery when required
```

---

# 251. Bloque 20 — Estado

```text
BLOCK 20 — EVENTS

✓ 209_DATABASE_EVENT_ARCHITECTURE.md
✓ 210_DATABASE_QUERY_EVENT_SYSTEM.md
✓ 211_DATABASE_CONNECTION_EVENT_SYSTEM.md
✓ 212_DATABASE_TRANSACTION_EVENT_PIPELINE.md
○ 213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
○ 214_DATABASE_PERSISTENCE_EVENT_SYSTEM.md
○ 215_DATABASE_EVENT_EXTENSION_SYSTEM.md
```

---

# 252. Siguiente documento

```text
213_DATABASE_ENTITY_LIFECYCLE_EVENT_SYSTEM.md
```

El siguiente documento definirá eventos relacionados con el ciclo de vida ORM de las entidades:

```text
Entity Instantiation
↓
Hydration
↓
Managed Registration
↓
State Transition
↓
Persist Scheduling
↓
Change Detection
↓
Update/Delete Scheduling
↓
Flush
↓
Post-Persistence State
↓
Transaction Completion
↓
Detach / Clear
```

incluyendo la separación crítica entre:

```text
Entity Lifecycle Event
≠
Persistence Event
≠
Database Query Event
≠
Transaction Event
≠
Domain Event
```

y entre:

```text
EntityPersisted
≠
EntityInserted
≠
TransactionCommitted
```

bajo la regla:

> **El estado de una entidad dentro del ORM y el estado durable de sus datos en la base son realidades relacionadas pero diferentes; los eventos de lifecycle nunca deberán afirmar durabilidad antes de que el Transaction System pueda demostrar un commit.**